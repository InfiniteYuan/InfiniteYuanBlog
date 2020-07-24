---
title: ESP32 学习笔记（二十七） ESP32 的启动过程 [转]
date: 2020-03-12 15:22:04
categories:
- ESP32 学习笔记
tags:
- ESP32
---

# ESP32 的启动过程

原文来自：https://www.esp32.com/viewtopic.php?f=25&t=8030&p=33812#p33812

## [关于 ROM]

在 esp32 上电运行后，芯片运行的第一个程序。这段程序是芯片设计与生产的时候，固化在硬件电路中的。所以它是不可修改的(Read Only Memory)。
esp32 的 ROM 负责检测芯片的strapping配置，来决定芯片应该处于什么状态。比如，esp32 上电后，ROM 程序会检查 [GPIO0, GPIO2, GPIO4, MTDO, GPIO5]的状态。
如果 GPIO0 / GPIO2 同时为低电平，则会进入下载模式，等待串口通信信息。
如果GPIO0为高电平，则会进入Flash 运行模式，启动SPI 驱动，并加载Flash中的程序段。

<!--more-->

BOOT_MODE[5:0]:
(pull-up, pull-down, pull-down, pull-up, pull-up, SW4 /5/4/3/2/1/ )
[GPIO0, GPIO2, GPIO4, MTDO, GPIO5]
1 x x x x --> SPI Boot
0 0 x x x --> Download Boot (Jonit-Detection of UART0+UART1+SDIO_Slave)

下载模式的串口输出如下(115200), ROM默认会输出当前所处的模式。
```c
rst:0x1 (POWERON_RESET),boot:0x3 (DOWNLOAD_BOOT(UART0/UART1/SDIO_REI_REO_V2))
waiting for download
```

其中，boot:0x3 表示的是芯片strapping pin脚的状态，
0x03对应 [GPIO0, GPIO2, GPIO4, MTDO, GPIO5] 的值为 [ 0, 0, 0, 1, 1]
所以处于 Download Boot 模式。

## [关于下载模式]

当 esp32 处于下载模式时，会等待串口通信同步，并按照通信协议等待接收指令(协议可参考该文档：[Serial-Protocol](https://github.com/espressif/esptool/wiki/Serial-Protocol)
通过esptool脚本，可以进行寄存器的读写，固件下载，程序运行等操作。

## [关于STUB]

在 ROM 模式，由于芯片处于低频工作的状态，通信速率受限。
在 esptool 中，会将一段小程序加载到 esp32 的 RAM 中，并跳转执行 RAM 中的小程序。这段小程序包含了 ROM 中相同的串口通信协议，并对其进行了扩充。感兴趣的开发者，(可以参考这里 [flasher_stub](https://github.com/espressif/esptool/tree/master/flasher_stub))

## [关于 Flash Boot 模式]

如果芯片启动时，GPIO0 为高电平，芯片会进入 Flash 运行模式。 此时，启动 SPI 驱动，并加载 Flash 中的程序段。ROM 会读取外置 Flash 的 0x1000 地址，加载并运行二级 bootloader。

## [关于 Bootloader]

bootloader 可以认为是一个独立的小程序，bootloader 会对芯片频率进行初始化，并且读取系统 SPI 的配置信息，对 Flash 运行模式以及频率进行配置，然后根据分区表的定义，从对应的地址加载应用程序，并且运行应用程序固件。
