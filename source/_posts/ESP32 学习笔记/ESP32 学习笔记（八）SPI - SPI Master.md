---
title: ESP32 学习笔记（八）SPI - SPI Master
date: 2018-08-12 23:25:08
categories:
- ESP32 学习笔记
tags:
- ESP32
---

# SPI Master

## 概述

ESP32 有四个 SPI 外设,称为 SPI0,SPI1,HSPI 和 VSPI. SPI0 是专用于 flash 缓存, ESP32 将连接的 SPI flash 设备映射到存储器. SPI1 和 SPI0 使用相同的硬件线,SPI1 用于写入 flash 芯片. HSPI 和 VSPI 可以任意使用. SPI1,HSPI 和 VSPI 都有三条片选线,作为 SPI 主机允许它们最多驱动三个 SPI 设备.

### spi_master 驱动

即使在多线程环境中,spi_master 驱动程序也可以轻松与 SPI 从设备进行通信. 它完全透明地处理 DMA 传输以读取和写入数据,并自动处理同一主机上不同 SPI 从站之间的多路复用.

<!--more-->

### 术语

spi_master 驱动程序使用以下术语:

 - 主机:ESP32 内部的 SPI 外设启动 SPI 传输. SPI,HSPI 或 VSPI 之一. (目前,驱动程序实际上只支持 HSPI 或 VSPI;它将在未来的某个地方支持所有 3个外设.)
 - 总线:SPI总线,与连接到一个主机的所有 SPI 设备共用.通常,总线由 miso,mosi,sclk 和可选的 quadwp 和 quadhd 信号组成. SPI 从设备并联连接到这些信号.
	 - miso - 也称为 q,这是将串行流输入到 ESP32中
	 - mosi - 也称为 d,这是来自 ESP32 的串行流的输出
	 - sclk - 时钟信号.每个数据位在该信号的正或负边沿输出或输出
	 - quadwp - 写保护信号.仅用于 4 位(qio/qout)事务.
	 - quadhd - 保持信号.仅用于 4 位(qio/qout)事务.
 - 设备:SPI 从设备.每个 SPI 从设备都有自己的片选(CS)线,当发生到/从 SPI 从设备的传输时,该线被激活.
 - 事务:一个 CS 活动的实例,来自和/或发生的设备的数据传输,以及 CS 再次变为非活动状态.事务是原子的,因为它们永远不会被另一个事务中断.

## SPI transactions

SPI 总线上的事务由五个阶段组成,其中任何阶段都可以跳过:

 - 命令阶段. 在此阶段,命令(0-16 位)被输出.
 - 地址阶段. 在此阶段,地址(0-64 位)被输出.
 - 写阶段. 主设备将数据发送到从设备.
 - 虚拟阶段. 该阶段是可配置的,用于满足时序要求.
 - 阅读阶段. 从站将数据发送给主站.

在全双工模式下,读写阶段被组合,SPI 主机同时读写数据.**总事务长度**由 `command_bits + address_bits + trans_conf.length` 决定,而 `trans_conf.rx_length` **仅确定**接收到缓冲区的数据长度.

在半双工模式下,主机具有独立的写和读阶段.写入阶段和读取阶段的**长度分别**由 `trans_conf.length` 和 `trans_conf.rx_length` 决定.

命令和地址阶段是可选的,因为不是每个 SPI 设备都需要发送命令和/或地址.这反映在设备配置中:当 `command_bits` 或 `address_bits` 字段设置为零时,不执行命令或地址阶段.

读写阶段的情况类似:并非每个事务都需要写入数据以及要读取的数据.当 `rx_buffer` 为 NULL(并且未设置 `SPI_USE_RXDATA`)时,将跳过读取阶段.当 `tx_buffer` 为 NULL(并且未设置 `SPI_USE_TXDATA`)时,将跳过写入阶段.

### Interrupt transactions

中断事务将阻塞事务例程，直到事务完成为止，从而使CPU可以运行其他任务。

当事务在传输中时，中断事务使用中断驱动的逻辑。程序将被阻塞，允许 CPU 在等待事务完成时运行其他任务。

中断事务可以排队到设备中，驱动程序会自动在 ISR 中逐个发送它们。任务可以将多个事务加入到队列中，并且能在事务完成之前执行其他操作。

### Polling transactions

轮询事务不依赖于中断，程序将持续轮询 SPI 外设的状态位，直到事务完成。

执行中断事务的所有任务可能会被队列阻塞，此时他们需要等待 ISR 在事务完成之前运行两次。轮询事务节省了在队列处理和上下文切换上花费的时间，从而导致较小的事务间隔。缺点是当这些事务在传输中时 CPU 正忙不能运行其他任务。

当事务完成时，`spi_device_polling_end` 例程至少花费 1us 开销来解除阻塞其他任务。强烈建议在 `spi_device_acquire_bus` 和 `spi_device_release_bus` 中包含一系列轮询事务以避免开销。

## Command and address phases

在命令和地址阶段，`spi_transaction_t` 结构中的 `cmd` 和 `addr` 字段将发送到总线，而不会同时读取任何内容。命令和地址阶段的默认长度在 `spi_device_interface_config_t` 和 `spi_bus_add_device` 中设置。当 `spi_transaction_t` 中未设置标志 `SPI_TRANS_VARIABLE_CMD` 和 `SPI_TRANS_VARIABLE_ADDR` 时，驱动程序会自动将这些阶段的长度分别设置为初始化设备时设置的默认值。

如果命令和地址阶段的长度需要更改，则声明一个 `spi_transaction_ext_t` 描述符，在 `base` 成员的 `flag` 中设置标志 `SPI_TRANS_VARIABLE_CMD` 或/和 `SPI_TRANS_VARIABLE_ADDR`，并像往常一样配置 `base` 的其余部分。然后每个阶段的长度将是 `spi_transaction_ext_t` 中设置的 `command_bits` 和 `address_bits`。

## Write and read phases

通常，要写入到设备或从设备读取的数据将从事务结构的 `rx_buffer` 和 `tx_buffer` 成员指向的一块存储器中读取或写入。为传输启用 DMA 时，强烈建议使用这些缓冲区以满足以下要求：

- 使用 `pvPortMallocCaps(size，MALLOC_CAP_DMA)` 在支持 `DMA` 的内存中分配;
- 32 位对齐（从边界开始，长度为 4 个字节的倍数）。

如果不满足这些要求，由于临时缓冲区的分配和 `memcpy`，事务效率将受到影响。

>使用 DMA 时，不支持有读和写两个阶段的半双工事务。有关详细信息和解决方法，请参阅已知问题。

## Bus acquiring

有时您可能希望不断地连续发送spi事务，以使其尽可能快。您可以使用 `spi_device_acquire_bus` 和 `spi_device_release_bus` 来实现这一点。获取总线后，与其他设备（无论是轮询还是中断）的事务处于待处理状态，直到总线被释放。

## 使用 spi_master 驱动

 - 通过调用 `spi_bus_initialize` 初始化 SPI 总线. 确保在 `bus_config` 结构中设置正确的 IO 引脚. 注意将不需要的信号设置为 -1.
 - 通过调用 `spi_bus_add_device` 告诉驱动程序连接到总线的 SPI 从设备. 确保在 `dev_config` 结构中配置设备具有的任何时序要求. 您现在应该拥有该设备的句柄,以便在发送事务时使用.
 - 要与设备交互,请使用您需要的任何事务参数填充一个或多个 `spi_transaction_t` 结构. 然后以轮询方式或中断方式发送它们：
	 - 中断方式：通过调用 `spi_device_queue_trans` 将事务添加到队列中,之后使用 `spi_device_get_trans_result` 查询结果,或者通过将它们提供给 `spi_device_transmit` 来处理所有请求.
	 - 轮询方式：调用 `spi_device_polling_transmit` 发送轮询事务。或者，如果要在它们之间插入内容，可以通过 `spi_device_polling_start` 和 `spi_device_polling_end` 发送轮询事务。
 - 可选：要对设备执行事务，请在事务之前调用 `spi_device_acquire_bus`，并在事务之后调用 `spi_device_release_bus`。
 - 可选：要卸载设备的驱动程序,请以设备句柄作为参数调用 `spi_bus_remove_device`
 - 可选：要删除总线的驱动程序,请确保没有连接更多驱动程序并调用 `spi_bus_free`.

### 提示

1. 少量数据的事务：

	有时，数据量非常小，因此不足以为其分配单独的缓冲区。如果要传输的数据是 32 位或更少，则可以将其存储在事务结构本身中。对于传输的数据，请使用 `tx_data` 成员并在传输上设置 `SPI_USE_TXDATA` 标志。对于接收的数据，使用 `rx_data` 并设置 `SPI_USE_RXDATA`。在这两种情况下，请勿触摸 `tx_buffer` 或 `rx_buffer` 成员，因为它们使用与 `tx_data` 和 `rx_data` 相同的内存位置。

2. 除了 `uint8_t` 之外的整数事务

	SPI 外设逐字节地读写存储器。默认情况下，SPI 工作在 MSB 优先模式，每个字节从 MSB 发送或接收到 LSB。但是，如果要发送长度不是 8 位倍数的数据，则会发送未使用的位。

	例如：你写 `uint8_t data = 0x15（00010101B）`，并设置长度只有 5 位，发送的数据是 00010B 而不是预期的 10101B。

	此外，ESP32 是一个小端芯片，其最低字节存储在 `uint16_t` 和 `uint32_t` 变量的起始地址。因此，如果 `uint16_t` 存储在存储器中，则首先发送第 7 位，然后将第 6 位发送到 0，然后将第 15 位发送到第 8 位。

	要发送 `uint8_t` 数组以外的数据，提供宏 `SPI_SWAP_DATA_TX` 以将数据转移到 `MSB` 并将 `MSB` 交换到最低地址;而 `SPI_SWAP_DATA_RX` 可用于将接收到的数据从 MSB 交换到正确的位置。

## GPIO matrix and IOMUX

ESP32 中的大多数外设信号可以直接连接到特定的 GPIO,称为 IOMUX 引脚. 当外设信号路由到 IOMUX 引脚以外的引脚时,ESP32 使用较不直接的 GPIO matrix 进行连接.

如果驱动器配置了所有 SPI 信号设置为其特定的 IOMUX 引脚(或未连接),它将绕过 GPIO matrix. 如果任何 SPI 信号配置到 IOMUx 引脚以外的引脚,驱动器将自动通过 GPIO matrix 路由所有信号. GPIO matrix对 80MHz 的所有信号进行采样,并在 GPIO 和外设之间发送.

当使用 GPIO matrix 时,由于 MISO 信号的输入延迟增加,因此超过 40MHz 的信号不能传播并且 MISO 的建立时间更容易被违反. GPIO Matrix 的最大时钟频率为 40MHz 或更低,而使用所有 IOMUX 引脚允许 80MHz.

>有关输入延迟对最大时钟频率影响的更多详细信息,请参阅下面的[时序注意事项](https://esp-idf.readthedocs.io/en/latest/api-reference/peripherals/spi_master.html#timing-considerations).

SPI 控制器的 IOMUX 引脚如下:
|Pin Name|HSPI(GPIO Number)|VSPI(GPIO Number)|
|-------------|:-------------:|:-------------:|
|CS0*|15|5|
|SCLK|14|18|
|MISO|12|19|
|MOSI|13|23|
|QUADWP|2|22|
|QUADHD|4|21|

>只有连接到总线的第一个设备才能使用 CS0 引脚.

## 已知的问题

1. 当存在写入和读取阶段时，半双工模式与 DMA 不兼容。

	如果需要此类交易，则必须使用其中一种替代解决方案：

	1. 改为使用全双工模式。

	2. 通过在总线初始化函数中将最后一个参数设置为 0 来禁用 DMA，如下所示：`ret = spi_bus_initialize(VSPI_HOST，＆buscfg，0);` 这可能会禁止您发送和接收超过 64 个字节的数据。

	3. 尝试使用命令和地址字段来替换写入阶段。

2. 全双工模式与虚拟位解决方案不兼容，因此频率有限。

3. `cs_ena_pretrans` 与全双工模式下的命令，地址阶段不兼容。

## 应用示例

## API Reference - SPI Common

在 WROVER-Kits 的 320x240 LCD 上显示图形:[peripherals/spi_master](https://github.com/espressif/esp-idf/tree/30545f4/examples/peripherals/spi_master).

### Header File

 - [driver/include/driver/spi_common.h](https://github.com/espressif/esp-idf/blob/30545f4/components/driver/include/driver/spi_common.h)

## API Reference - SPI Master

### Header File

 - [driver/include/driver/spi_master.h](https://github.com/espressif/esp-idf/blob/30545f4/components/driver/include/driver/spi_master.h)

## 参考资料
 - [SPI Master driver](https://docs.espressif.com/projects/esp-idf/en/v3.2/api-reference/peripherals/spi_master.html)
