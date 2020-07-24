---
title: ESP32 学习笔记（九）SPI - SPI Slave
date: 2018-08-13 12:27:32
categories:
- ESP32 学习笔记
tags:
- ESP32
---

# SPI Slave

## 概述

ESP32 有四个 SPI 外设,称为 SPI0,SPI1,HSPI 和 VSPI. SPI0 是专用于 flash 缓存, ESP32 将连接的 SPI flash 设备映射到存储器. SPI1 和 SPI0 使用相同的硬件线,SPI1 用于写入 flash 芯片. HSPI 和 VSPI 可以任意使用，并使用 spi_slave 驱动程序用作 SPI 从设备，由连接的 SPI 主设备驱动。

### spi_slave 驱动

`spi_slave` 驱动程序允许将 HSPI 和/或 VSPI 外设用作全双工 SPI 从设备。它可以使用 DMA 来发送/接收任意长度的事务。

<!--more-->

### 术语

spi_slave 驱动程序使用以下术语：

 - 主机：ESP32 内部的 SPI 外设启动 SPI 传输。HSPI 或 VSPI 之一。
 - 总线：SPI 总线，与连接到一个主机的所有 SPI 设备共用。通常，总线由 miso，mosi，sclk 和可选的 quadwp 和 quadhd 信号组成。SPI 从设备并联连接到这些信号。每个 SPI 从设备也连接到一个 CS 信号。
	 - miso - 也称为 q，这是从 ESP32 到 SPI 主设备的串行流输出
	 - mosi - 也称为 d，这是从 SPI 主设备到 ESP32 的串行流输出
	 - sclk - 时钟信号。每个数据位在该信号的正或负边沿输出
	 - cs - 芯片选择。有效芯片选择描述与从属设备之间的单个事务。
 - 事务：一个 CS 活动的实例，来自和/或发送到主设备的一次数据传输，以及 CS 再次变为非活动状态。事务具有原子性，因为它们永远不会被另一个事务中断。

## SPI transactions

全双工 SPI 事务从主机拉低 CS 开始。发生这种情况后，主机开始在 CLK 线上发出时钟脉冲：每个时钟脉冲使一个数据位在 MOSI 线上从主机移位到从机，反之亦然在 MISO 线上。在传输结束时，主机再次使 CS 成为高电平。

## 使用 spi_slave 驱动

- 通过调用 `spi_slave_initialize` 将 SPI 外设初始化为从设备。确保在 `bus_config` 结构中设置正确的 IO 引脚。注意将不需要的信号设置为 -1。如果事务将大于 32 字节，则必须给出 DMA 通道（1或2），否则 `dma_chan` 参数可能为 0。
- 要设置事务，请使用您需要的任何事务参数填充一个或多个 `spi_transaction_t` 结构。通过调用 `spi_slave_queue_trans` 将所有事务添加到队列中，之后使用 `spi_slave_get_trans_result` 查询结果，或者通过将它们交给 `spi_slave_transmit` 来处理所有请求。后两个函数将阻塞，直到主设备启动并完成一个事务，使队列中的数据被发送和接收。
- 可选：要卸载 SPI 从设备驱动程序，请调用 `spi_slave_free`。

## 传输数据和主/从机长度不匹配

通常，要发送到设备或从设备接收的数据将从事务结构的 `rx_buffer` 和 `tx_buffer` 成员指向的一块存储器中读取或写入。SPI 驱动程序可能决定使用 DMA 进行传输，因此应使用 `pvPortMallocCaps(size，MALLOC_CAP_DMA)` 在具有 DMA 功能的内存中分配这些缓冲区。

写入缓冲区的数据量受事务结构的 `length` 成员的限制：驱动程序永远不会读取/写入比指定的长度更多的数据。长度不能定义 SPI 事务的实际长度;这是由主机驱动时钟和 CS 线路决定的。在传输事务结束后，可以从 `spi_slave_transaction_t` 结构的 `trans_len` 成员读取传输的实际长度。在传输长度大于缓冲区长度的情况下，仅发送和接收传输的开始，并且将 `trans_len` 设置为长度而不是实际长度。如果需要 `trans_len`，建议设置长度超过预期的最大长度。如果传输长度短于缓冲区长度，则只交换缓冲区长度的数据。

>警告：由于 ESP32 的设计特性，如果主机发送的字节数或从设备驱动中传输队列的长度（以字节为单位）不大于8且可分为4，则 SPI 硬件可以无法将最后一到七个字节写到接收缓冲区。

## 限制、已知问题

- 如果 DMA 被使能，则接收缓冲区应该是字对齐的，即从 32 位的边界开始，并且长度为 4 个字节的倍数。否则 DMA 可能会错误地写入或跳出边界。驱动程序将检查此情况。

此外，master 应写入长度为 4 个字节的倍数。超过该数据的数据将被丢弃。

## 应用示例

主/从机通信：[peripherals/spi_slave](https://github.com/espressif/esp-idf/tree/30545f4/examples/peripherals/spi_slave).

## API Reference

### Header File

 - [driver/include/driver/spi_slave.h](https://github.com/espressif/esp-idf/blob/30545f4/components/driver/include/driver/spi_slave.h)

## 参考资料

 - [SPI Slave driver](https://docs.espressif.com/projects/esp-idf/en/v3.2/api-reference/peripherals/spi_slave.html)
