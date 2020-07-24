---
title: ESP32 学习笔记（三）UART - Universal Asynchronous Receiver/Transmitter
date: 2018-08-11 15:41:29
categories:
- ESP32 学习笔记
tags:
- ESP32
---

# UART - Universal Asynchronous Receiver/Transmitter

## 概述

嵌入式应用通常要求一个简单的并且占用系统资源少的方法来传输数据.通用异步收发传输器(UART) 即可以满足这些要求,它能够灵活地与外部设备进行全双工数据交换.ESP32 芯片中有 3 个UART 控制器可供使用,并且兼容不同的 UART 设备.另外,UART 还可以用作红外数据交换(IrDA) 或 RS-485 调制解调器.

3 个 UART 控制器有一组功能相同的寄存器.本文以 UARTn 指代 3 个 UART 控制器,n 为0、1、2.

## 主要特性

 - 可编程收发波特率
 - 3 个 UART 的发送 FIFO 以及接收 FIFO 共享1024 x 8-bit RAM
 - 全双工异步通信
 - 支持输入信号波特率自检功能
 - 支持 5/6/7/8 位数据长度
 - 支持 1/1.5/2/3/4 个停止位
 - 支持奇偶校验位
 - 支持 RS485 协议
 - 支持 IrDA 协议
 - 支持 DMA 高速数据通信
 - 支持 UART 唤醒模式

<!--more-->

## 功能描述

UART 是一种以字符为导向的通用数据链,可以实现设备间的通信.异步传输的意思是不需要在发送数据上添加时钟信息.这也要求发送端和接收端的速率、停止位、奇偶校验位等都要相同,通信才能成功.

一个典型的 UART 帧开始于一个起始位,紧接着是有效数据,然后是奇偶校验位(可有可无),最后是停止位.ESP32 上的 UART 控制器支持多种字符长度和停止位.另外,控制器还支持软硬件流控和 DMA,可以实现无缝高速的数据传输.开发者可以使用多个 UART 端口,同时又能保证很少的软件开销.
![UART 基本架构图](https://img-blog.csdn.net/2018081119230686?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
<center>图1 UART 基本架构图</center>

UART 有两个时钟源:80-MHz APB_CLK 以及参考时钟 REF_TICK (详情请参考章节复位和时钟).可以通过配置 UART_TICK_REF_ALWAYS_ON 来选择时钟源.时钟中的分频器用于对时钟源进行分频,然后产生时钟信号来驱动 UART 模块.ART_CLKDIV_REG 将分频系数分成两个部分:UART_CLKDIV 用于配置整数部分,UART_CLKDIV_FRAG 用于配置小数部分.

UART 控制器可以分为两个功能块:发送块和接收块.

发送块包含一个发送 FIFO 用于缓存待发送的数据.软件可以通过 APB 总线写 Tx_FIFO,也可以通过 DMA 将数据搬入 Tx_FIFO.Tx_FIFO_Ctrl 用于控制 Tx_FIFO 的读写过程,当 Tx_FIFO 非空时,Tx_FSM 通过 Tx_FIFO_Ctrl读取数据,并将数据按照配置的帧格式转化成比特流.比特流输出信号 txd_out 可以通过配置 UART_TXD_INV 寄存器实现取反功能.

接收块包含一个接收 FIFO 用于缓存待处理的数据.输入比特流 rxd_in 可以输入到 UART 控制器.可以通过 UART_RXD_INV 寄存器实现取反.Baudrate_Detect 通过检测最小比特流输入信号的脉宽来测量输入信号的波特率.Start_Detect 用于检测数据的 START 位,当检测到 START 位之后,RX_FSM 通过 Rx_FIFO_Ctrl 将帧解析后的数据存入 Rx_FIFO 中.

软件可以通过 APB 总线读取 Rx_FIFO 中的数据.为了提高数据传输效率,可以使用 DMA 方式进行数据发送或接收.

HW_Flow_Ctrl 通过标准 UART RTS 和 CTS(rtsn_out 和 ctsn_in)流控信号来控制 rxd_in 和 txd_out 的数据流.SW_Flow_Ctrl 通过在发送数据流中插入特殊字符以及在接收数据流中检测特殊字符来进行数据流的控制.当 UART 处于 Light-sleep(详情请参考章节低功耗管理)状态时,Wakeup_Ctrl 开始计算 rxd_in 的脉冲个数,当脉冲个数大于 UART_ACTIVE_THRESHOLD 时产生 wake_up 信号给 RTC 模块,由 RTC 来唤醒 UART 控制器.
![UART 共享RAM 图](https://img-blog.csdn.net/20180811193508599?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
<center>UART 共享 RAM 图</center>

## 应用示例

配置 UART 设置并使用 UART1 接口安装 UART 驱动程序进行读/写:[peripherals/uart_echo](https://github.com/espressif/esp-idf/tree/f9a4496/examples/peripherals/uart_echo).

演示如何报告各种通信事件以及如何使用模式检测中断:[peripherals/uart_events](https://github.com/espressif/esp-idf/tree/f9a4496/examples/peripherals/uart_events).

在两个独立的 FreeRTOS 任务中使用相同的 UART 进行发送和接收:[peripherals/uart_async_rxtxtasks](https://github.com/espressif/esp-idf/tree/f9a4496/examples/peripherals/uart_async_rxtxtasks).

对 UART 文件描述符使用同步 I/O 多路复用:[peripherals/uart_select](https://github.com/espressif/esp-idf/tree/f9a4496/examples/peripherals/uart_select).

设置 UART 驱动程序以半双工模式通过 RS485 接口进行通信:[peripherals/uart_echo_rs485](https://github.com/espressif/esp-idf/tree/f9a4496/examples/peripherals/uart_echo_rs485). 此示例类似于 uart_echo,但通过连接到 ESP32 引脚的 RS485 接口芯片提供通信.

## API Reference

### Header File

 - [driver/include/driver/uart.h](https://github.com/espressif/esp-idf/blob/f9a4496/components/driver/include/driver/uart.h)

## API 使用

以下概述描述了用于在 ESP32 和某些其他 UART 设备之间建立通信的功能和数据类型. 概述反映了编程 ESP32 的UART驱动程序时的典型工作流程,并分为以下几个部分:

 1. 设置通信参数 - 波特率,数据位,停止位等
 2. 设置通信引脚 - 连接另一个 UART 的引脚
 3. 驱动程序安装 - 为 UART 驱动程序分配 ESP32 的资源
 4. 运行 UART 通信 - 发送/接收数据
 5. 使用中断 - 触发特定通信事件的中断
 6. 删除驱动程序 - 如果不再需要 UART 通信,则释放 ESP32 的资源
 
使 UART 能开始工作是完成前面的四个步骤,后两个步骤是可选的.

驱动程序由`uart_port_t`标识,它对应于一个组 UART 控制器. 这种识别存在于以下所有函数调用中.

### 设置通信参数

有两种方法可以设置 UART 的通信参数. 一种是通过调用`uart_param_config()`在`uart_config_t`结构中提供配置参数来一次性完成.

另一种方法是通过调用函数单独配置特定参数:

 - 波特率 - `uart_set_baudrate()`
 - 传输的位数 - `uart_set_word_length()`指定`uart_word_length_t`的值
 - 奇偶校验控制 - `uart_set_parity()`指定`uart_parity_t` 的值
 - 停止位数目 - `uart_set_stop_bits()`指定`uart_stop_bits_t`的值
 - 硬件流控制模式 - `uart_set_hw_flow_ctrl()`指定`uart_hw_flowcontrol_t` 的值
 - 通信模式 - `uart_set_mode()`指定`uart_mode_t`的值

```c
const int uart_num = UART_NUM_2;
uart_config_t uart_config = {
    .baud_rate = 115200,
    .data_bits = UART_DATA_8_BITS,
    .parity = UART_PARITY_DISABLE,
    .stop_bits = UART_STOP_BITS_1,
    .flow_ctrl = UART_HW_FLOWCTRL_CTS_RTS,
    .rx_flow_ctrl_thresh = 122,
};
// Configure UART parameters
ESP_ERROR_CHECK(uart_param_config(uart_num, &uart_config));
```

### 设置通信引脚

通过调用函数 `uart_set_pin()`我们可以输入宏`UART_PIN_NO_CHANGE`而不是 GPIO 引脚编号,并且不会更改当前分配的引脚. 如果不使用某个引脚,则应输入相同的宏.

```c
// Set UART pins(TX: IO16 (UART2 default), RX: IO17 (UART2 default), RTS: IO18, CTS: IO19)
ESP_ERROR_CHECK(uart_set_pin(UART_NUM_2, UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE, 18, 19));
```

### 驱动程序安装

完成驱动程序配置后,我们可以通过调用 `uart_driver_install()`来安装 UART 驱动. 并分配给 UART 所需的若干资源. 资源的类型/大小被指定为函数参数并需要考虑:

 - 发送缓冲区的大小
 - 接收缓冲区的大小
 - 事件队列句柄和大小
 - 用于分配中断的标志

```c
// Setup UART buffered IO with event queue
const int uart_buffer_size = (1024 * 2);
QueueHandle_t uart_queue;
// Install UART driver using an event queue here
ESP_ERROR_CHECK(uart_driver_install(UART_NUM_2, uart_buffer_size, \
                                        uart_buffer_size, 10, &uart_queue, 0));
```

如果上述所有步骤均已完成,我们就可以连接其他 UART 设备并开始通信.

### 运行 UART 通信

串行通信的过程受 UART 硬件 FSM 的控制. 要发送的数据应放入 Tx FIFO 缓冲区, FSM 将序列化并发送出去. 完成类似的过程,但是以相反的顺序,接收数据. 传入的串行流由 FSM 处理并移至 Rx FIFO 缓冲区. 因此, API 的通信功能的任务仅限于向/从相应缓冲区写入和读取数据. 这反映在一些函数名称中,例如:`uart_write_bytes()`用于传输数据,或`uart_read_bytes()`用于读取传入数据.

### 使用中断

报告了 UART 的特定状态或检测到的错误有 19 个中断. ESP32 技术参考手册(PDF)中介绍了可用中断的完整列表. 要启用特定中断,请调用`uart_enable_intr_mask()`,以禁用调用`uart_disable_intr_mask()`. 所有中断的掩码都以`UART_INTR_MASK`的形式提供. 使用`uart_isr_register()`完成向服务中断注册处理程序,使用`uart_isr_free()`释放处理程序. 要在调用处理程序后清除中断状态位,请使用`uart_clear_intr_status()`.

### 删除驱动程序

如果与`uart_driver_install()`建立通信一段特定时间,然后不需要,则可以通过调用`uart_driver_delete()`来删除驱动程序以释放分配的资源.

## 参考资料

 - [UART](https://docs.espressif.com/projects/esp-idf/en/v3.2/api-reference/peripherals/uart.html)
