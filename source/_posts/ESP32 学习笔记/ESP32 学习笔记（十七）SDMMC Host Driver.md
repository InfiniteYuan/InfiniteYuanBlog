---
title: ESP32 学习笔记（十七）SDMMC Host Driver
date: 2018-11-15 17:07:42
categories:
- ESP32 学习笔记
tags:
- ESP32
---

# 概述

在 ESP32 上，SDMMC 主机外设有两个插槽：
- 插槽 0(`SDMMC_HOST_SLOT_0`)是一个 8 位插槽。它使用 `PIN MUX` 中的 `HS1_ *` 信号。
- 插槽 1(`SDMMC_HOST_SLOT_1`)是一个 4 位插槽。它使用 `PIN MUX` 中的 `HS2_ *` 信号。

这些插槽的引脚映射如下表所示：
|Signal|Slot 0|Slot 1|
|:---:|:---:|:---:|
|CMD |	GPIO11 	|GPIO15|
CLK 	|GPIO6| 	GPIO14|
|D0 |	GPIO7 	|GPIO2|
|D1 	|GPIO8 |	GPIO4|
|D2 	|GPIO9 	|GPIO12|
|D3 	|GPIO10 	|GPIO13|
|D4 	|GPIO16 	 ||
|D5 	|GPIO17| 	| 
|D6 	|GPIO5| 	 |
|D7 	|GPIO18||
|CD 	|any input via GPIO matrix|any input via GPIO matrix|
|WP 	|any input via GPIO matrix|any input via GPIO matrix|

可以使用 GPIO 矩阵将卡检测和写保护信号路由到任意引脚。要使用这些引脚，请在调用 `sdmmc_host_init_slot()` 之前设置 `sdmmc_slot_config_t` 结构的 `gpio_cd` 和 `gpio_wp` 成员。请注意，在使用 SDIO 卡时，建议不要指定卡检测引脚，因为在 ESP32 卡检测信号中也可以触发 SDIO 从机中断。

> 插槽0(HS1_ *)使用的引脚也用于连接 ESP-WROOM32 和 ESP32-WROVER 模块中的 SPI 闪存芯片。这些引脚不能在 SD 卡和 SPI 闪存之间共享。如果需要使用 Slot 0，请将 SPI flash连接到不同的引脚并相应地设置 Efuses。

<!--more-->

## 支持的速度模式

SDMMC 主机驱动程序支持以下速度模式：

 - 默认速度(20MHz)，4 线/1 线(带 SD 卡)和8 线(带 3.3V eMMC)。
 - 高速(40MHz)，4 线/1 线(带 SD 卡)和8 线(带 3.3V eMMC)
 - 高速DDR(40MHz)，4 线(带 3.3V eMMC)

目前不支持的是：

 - 高速 DDR 模式，8 线 eMMC
 - UHS-I 1.8V 模式，4 线 SD 卡

## 使用SDMMC主机驱动程序

在下面列出的所有功能中，大多数应用程序将直接使用 `sdmmc_host_init()`，`sdmmc_host_init_slot()` 和 `sdmmc_host_deinit()`。

其他函数，例如 `sdmmc_host_set_bus_width()`，`sdmmc_host_set_card_clk()` 和 `sdmmc_host_do_transaction()` 将由 SD/MMC 协议层通过 `sdmmc_host_t` 结构中的函数指针调用。

## 配置总线宽度和频率

使用 `sdmmc_host_t` 和 `sdmmc_slot_config_t`(`SDMMC_HOST_DEFAULT` 和 `SDMMC_SLOT_CONFIG_DEFAULT`)的默认初始化程序，SDMMC 主机驱动程序将尝试使用该卡支持的最宽总线(SD 为 4 行，eMMC 为 8 行)和 20MHz 频率。

在可以实现 40MHz 频率通信的设计中，可以通过更改 `sdmmc_host_t` 的 `max_freq_khz` 字段来增加总线频率：

```c
sdmmc_host_t host = SDMMC_HOST_DEFAULT();
host.max_freq_khz = SDMMC_FREQ_HIGHSPEED;
```

要配置总线宽度，请设置 `sdmmc_slot_config_t` 的宽度字段。例如，要设置 1 位模式：

```c
sdmmc_slot_config_t slot = SDMMC_SLOT_CONFIG_DEFAULT();
slot.width = 1;
```

## 更多

有关实现协议层的更高级别驱动程序，请参阅 [SD/SDIO/MMC驱动程序](https://docs.espressif.com/projects/esp-idf/en/latest/api-reference/storage/sdmmc.html)。

有关使用 SPI 控制器的类似驱动程序，请参阅 [SD SPI主机驱动程序](https://docs.espressif.com/projects/esp-idf/en/latest/api-reference/peripherals/sdspi_host.html)，并且仅限于 SD 协议的 SPI 模式。

有关上拉支持以及有关模块和设备的兼容性，请参阅 [SD上拉要求](https://docs.espressif.com/projects/esp-idf/en/latest/api-reference/peripherals/sd_pullup_requirements.html)。

## 参考资料

 - [SDMMC Host Driver](https://docs.espressif.com/projects/esp-idf/en/v3.2/api-reference/peripherals/sdmmc_host.html)
