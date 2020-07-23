---
title: ESP32 开发笔记（十四）ESP32 I2C Slave 实现
date: 2019-11-22 01:24:44
categories:
- ESP32 开发笔记
tags:
- ESP32
---

# 概述

这篇文章将介绍使用 ESP32 作为 I2C 实现 Random Read/Write 和  Sequential Read/Write 时序。

首先通过下面的图了解下 Random Read 时序，I2C Master 通过这个时序读取任意数据地址开始的数据。
![Random Read](https://img-blog.csdnimg.cn/20191123145556881.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3,size_16,color_FFFFFF,t_70)

<!--more-->

```c
___________________________________________________________________________________________________________________________________________________
| start | slave_addr + write_bit + ack | data address |start | slave_addr + read_bit + ack |  read n-1 bytes + ack | read 1 byte + nack | stop |
```

> **RANDOM READ**: A random read requires a “dummy” byte write sequence to load in the data word address. Once the device address word and data word address are clocked in and acknowledged by the EEPROM, the microcontroller must generate another start condition.
> The microcontroller now initiates a current address read by sending a device address with the read/write select bit high. The EEPROM acknowledges the device address and serially clocks out the data word. The microcontroller does not respond with a zero but does generate a following stop condition 

 - 现有的 I2C Slave 无法实现类似 Random Read 时序
	 - 原因分析：esp-idf 提供的 `i2c_slave_write_buffer` `i2c_slave_read_buffer` API 都是操作 RingBuffer 实现，而 Random Read 需要 Slave 在 接收到 `slave_addr + read_bit + data address` 前将数据放入 I2C 的硬件 FIFO 中。若是通过 API 进行判断当前 Master 想要操作的 数据地址，会因为这个 API 都是操作 RingBuffer 而有所延迟，导致 Master 接收到错误的数据（因为此时硬件 FIFO 还没有数据）。
	 - 解决办法：
		 - 在接收到 `slave_addr + write_bit + data address` 时将可能需要发送到主机的数据放入 TX FIFO 中，当主机继续发送 `slave_addr + read_bit + data address` 并 提供读数据时钟 时，I2C Slave 硬件会将 TX FIFO 中的数据发送到 Master。若主机不再发送 `slave_addr + read_bit + data address`，就将 FIFO 清空避免在之后的操作中造成错误。
		 - 通过自定义中断处理程序，在相应的中断中进行处理

# I2C 中断介绍

 1. 想要针对某个 I2C 或者 某种模式，使用自定义的中断处理程序，需要修改 `esp-idf/components/driver/i2c.c` 中的代码，这里仅仅针对 master 使用驱动中提供的中断处理程序，当模式为 Slave 时，使用自定义的中断处理程序。

> 在 release/3.2 中，无法直接使用 `i2c_isr_register` 覆盖之前的中断处理程序。通过简单修改驱动源文件，并在应用程序中调用 `i2c_isr_register` 实现使用自定义的中断处理程序。

修改 `esp-idf/components/driver/i2c.c` 文件中 `i2c_driver_install`  函数中的这段代码：
```c
if (mode == I2C_MODE_MASTER) {
    //hook isr handler
    i2c_isr_register(i2c_num, i2c_isr_handler_default, p_i2c_obj[i2c_num], intr_alloc_flags, &p_i2c_obj[i2c_num]->intr_handle);
}
```

在应用程序中使用自定义的中断处理程序：

```c
static void IRAM_ATTR i2c_slave_isr_handler_default(void* arg)
{
	…
	…
	…
}

/**
 * @brief i2c slave initialization
 */
static esp_err_t i2c_slave_init()
{
    int i2c_slave_port = I2C_SLAVE_NUM;
    i2c_config_t conf_slave;
    conf_slave.sda_io_num = I2C_SLAVE_SDA_IO;
    conf_slave.sda_pullup_en = GPIO_PULLUP_ENABLE;
    conf_slave.scl_io_num = I2C_SLAVE_SCL_IO;
    conf_slave.scl_pullup_en = GPIO_PULLUP_ENABLE;
    conf_slave.mode = I2C_MODE_SLAVE;
    conf_slave.slave.addr_10bit_en = 0;
    conf_slave.slave.slave_addr = ESP_SLAVE_ADDR;
    i2c_param_config(i2c_slave_port, &conf_slave);
    i2c_driver_install(i2c_slave_port, conf_slave.mode, I2C_SLAVE_RX_BUF_LEN, I2C_SLAVE_TX_BUF_LEN, 0);
    
    if (i2c_isr_register(I2C_SLAVE_NUM, i2c_slave_isr_handler_default, NULL, 0, NULL) != ESP_OK) {
        ESP_LOGE(TAG, "i2c_isr_register error");
        return ESP_FAIL;
    }
    
    return ESP_OK;
}
```

# Slave 中断处理程序

需要使用的几个中断：

 - **I2C_SLAVE_TRAN_COMP_INT**：从机收到设备地址和数据地址时将触发（device address + W/R bit + data address），在这里需要判断 Write/Read。若是 Write，需要清空 TX FIFO，并提前将需要操作的数据放入 TX FIFO 中（根据数据地址），尽可能放满 TX FIFO
 - **I2C_TRANS_COMPLETE_INT**：从机检测到 STOP 时将触发，这个中断中需要将 RX FIFO 中的数据取到数据缓冲区中
 - **I2C_TXFIFO_EMPTY_INT**：硬件 TX FIFO 为空时将触发，（PS：实际测试触发时，FIFO 大小为 27 byte），将可能操作的数据继续放入 TX FIFO 中
 - **I2C_RXFIFO_FULL_INT**：硬件 RX FIFO 满时将触发，将 RX FIFO 中的数据取到数据缓冲区中

```c
static void IRAM_ATTR i2c_slave_isr_handler_default(void* arg)
{

    int i2c_num = I2C_SLAVE_NUM;
    uint32_t status = I2C_INSTANCE[i2c_num]->int_status.val;
    int idx = 0;

    portBASE_TYPE HPTaskAwoken = pdFALSE;
    while (status != 0) {
        status = I2C_INSTANCE[i2c_num]->int_status.val;
        if (status & I2C_ACK_ERR_INT_ST_M) {
            ets_printf("ae\n");
            I2C_INSTANCE[i2c_num]->int_ena.ack_err = 0;
            I2C_INSTANCE[i2c_num]->int_clr.ack_err = 1;
        } else if (status & I2C_TRANS_COMPLETE_INT_ST_M) { // receive data after receive device address + W/R bit and data address
            // ets_printf("tc, ");
            I2C_INSTANCE[i2c_num]->int_clr.trans_complete = 1;

            int rx_fifo_cnt = I2C_INSTANCE[i2c_num]->status_reg.rx_fifo_cnt;
            if (I2C_INSTANCE[i2c_num]->status_reg.slave_rw) { // read, slave should to send
                // ets_printf("R\n");
            } else { // write, slave should to recv
                // ets_printf("W ");
                ets_printf("Slave Recv");
                
                for (idx = 0; idx < rx_fifo_cnt; idx++) {
                    slave_data[w_r_index++] = I2C_INSTANCE[i2c_num]->fifo_data.data;
                }
                ets_printf("\n");
            }
            I2C_INSTANCE[i2c_num]->int_clr.rx_fifo_full = 1;
            slave_event = SLAVE_IDLE;
        } else if (status & I2C_SLAVE_TRAN_COMP_INT_ST_M) { // slave receive device address + W/R bit + data address
            if (I2C_INSTANCE[i2c_num]->status_reg.slave_rw) { // read, slave should to send
                ets_printf("sc, Slave Send\n");
            } else { // write, slave should to recv
                // ets_printf("sc W\n");
                w_r_index = I2C_INSTANCE[i2c_num]->fifo_data.data;
                switch (slave_event) {
                    case SLAVE_IDLE:
                        ets_printf("sc, I2W\n");
                        // reset tx fifo to avoid send last byte when master send read command next.
                        i2c_reset_tx_fifo(i2c_num);

                        slave_event = SLAVE_WRITE;
                        slave_send_index = w_r_index;

                        int tx_fifo_rem = I2C_FIFO_LEN - I2C_INSTANCE[i2c_num]->status_reg.tx_fifo_cnt;
                        for (size_t i = 0; i < tx_fifo_rem; i++) {
                            WRITE_PERI_REG(I2C_DATA_APB_REG(i2c_num), slave_data[slave_send_index++]);
                        }
                        I2C_INSTANCE[i2c_num]->int_ena.tx_fifo_empty = 1;
                        I2C_INSTANCE[i2c_num]->int_clr.tx_fifo_empty = 1;
                        break;
                }
            }

            I2C_INSTANCE[i2c_num]->int_clr.slave_tran_comp = 1;
        } else if (status & I2C_TXFIFO_EMPTY_INT_ST_M) {
            ets_printf("tfe, ");
            int tx_fifo_rem = I2C_FIFO_LEN - I2C_INSTANCE[i2c_num]->status_reg.tx_fifo_cnt;

            if (I2C_INSTANCE[i2c_num]->status_reg.slave_rw) { // read, slave should to send
                ets_printf("R\r\n");
                for (size_t i = 0; i < tx_fifo_rem; i++) {
                    WRITE_PERI_REG(I2C_DATA_APB_REG(i2c_num), slave_data[slave_send_index++]);
                }
                I2C_INSTANCE[i2c_num]->int_ena.tx_fifo_empty = 1;
                I2C_INSTANCE[i2c_num]->int_clr.tx_fifo_empty = 1;
            } else { // write, slave should to recv
                ets_printf("W\r\n");
            }
        } else if (status & I2C_RXFIFO_OVF_INT_ST_M) {
            ets_printf("rfo\n");
            I2C_INSTANCE[i2c_num]->int_clr.rx_fifo_ovf = 1;
        } else if (status & I2C_RXFIFO_FULL_INT_ST_M) {
            ets_printf("rff\n");
            int rx_fifo_cnt = I2C_INSTANCE[i2c_num]->status_reg.rx_fifo_cnt;
            for (idx = 0; idx < rx_fifo_cnt; idx++) {
                slave_data[w_r_index++] = I2C_INSTANCE[i2c_num]->fifo_data.data;
            }
            I2C_INSTANCE[i2c_num]->int_clr.rx_fifo_full = 1;
        } else {
            // ets_printf("%x\n", status);
            I2C_INSTANCE[i2c_num]->int_clr.val = status;
        }
    }
    //We only need to check here if there is a high-priority task needs to be switched.
    if(HPTaskAwoken == pdTRUE) {
        portYIELD_FROM_ISR();
    }
}
```

# 完整工程

 - [ESP32-I2C-Slave](https://github.com/InfiniteYuan/ESP32-I2C-Slave/tree/master)

# 参考资料

 - [I2C Specification](https://www.nxp.com/docs/en/user-guide/UM10204.pdf)
