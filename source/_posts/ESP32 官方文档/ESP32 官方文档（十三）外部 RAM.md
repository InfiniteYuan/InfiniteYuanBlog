---
title: ESP32 官方文档（十三）外部 RAM
date: 2012-07-22 09:43:43
categories:
- ESP32 官方文档
tags:
- ESP32
---

# 外部 RAM

## 介绍

ESP32 有几百 KiB 的内部 RAM，与 ESP32 的其余部分位于同一个芯片上。 出于某些目的，这是不够的，因此ESP32还能够使用高达 4MB 的外部 SPI RAM 存储器作为存储器。 外部存储器包含在存储器映射中，并且在某些限制内，可以与内部数据 RAM 相同的方式使用。

## 硬件

ESP32 支持与 SPI Flash 芯片并联的SPI（P）SRAM。 虽然 ESP32 能够支持多种类型的 RAM 芯片，但 ESP32 SDK 目前仅支持 ESP-PSRAM32 芯片。

ESP-PSRAM32 芯片是 1.8V 器件，只能与 1.8V 闪存器件并联使用。 确保在启动时将 MTDI 引脚设置为高信号电平，或者将 ESP32 中的保险丝编程为始终使用 1.8V 的 VDD_SIO 电平。 不这样做有损坏 PSRAM 和/或 Flash 芯片的风险。

**要将 ESP-PSRAM 芯片连接到 ESP32D0W *，请连接以下信号：**

 - PSRAM /CE (pin 1) - ESP32 GPIO 16
 - PSRAM SO (pin 2) - flash DO
 - PSRAM SIO[2] (pin 3) - flash WP
 - PSRAM SI (pin 5) - flash DI
 - PSRAM SCLK (pin 6) - ESP32 GPIO 17
 - PSRAM SIO[3] (pin 7) - flash HOLD
 - PSRAM Vcc (pin 8) - ESP32 VCC_SDIO

ESP32D2W * 芯片的连接是 TBD。

> Espressif 销售 ESP-WROVER 模块，该模块包含 ESP32,1.8V Flash 和集成在模块中的 ESP-PSRAM32，可以包含在最终产品 PCB 中。

## 软件

**ESP-IDF 完全支持将外部存储器集成到您的应用程序中。 ESP-IDF 可以配置为以多种方式处理外部 RAM：**

 - 只初始化 RAM。 这允许应用程序通过解除指向外部 RAM 存储器区域（0x3F800000 及以上）的指针来手动放置数据。
 - 初始化 RAM 并将其添加到功能分配器。 这允许程序使用 `heap_caps_malloc（size，MALLOC_CAP_SPIRAM）` 专门分配一块外部 RAM。 可以使用此内存，然后使用正常的 `free（）` 调用释放。
 - 初始化 RAM，将其添加到功能分配器，并将内存添加到可由 `malloc（）` 返回的 RAM 池中。 这允许任何应用程序使用外部 RAM 而无需重写代码以使用 `heap_caps_malloc`。

可以从 menuconfig 菜单中选择所有这些选项。

### 限制

**使用外部 RAM 有一些限制：**

 - 禁用闪存缓存时（例如，因为正在写闪存），外部 RAM 也变得无法访问;对它的任何读取或写入都将导致非法的缓存访问异常。这也是 ESP-IDF 永远不会在外部 RAM 中分配任务堆栈的原因。
 - 外部 RAM 不能用作存储 DMA 事务描述符的位置，也不能用作 DMA 传输的读取或写入缓冲区。必须使用 `heap_caps_malloc（size，MALLOC_CAP_DMA）` 分配将与 DMA 结合使用的任何缓冲区（并且可以使用标准的 `free（）` 调用释放。）
 - 外部 RAM 使用与外部闪存相同的缓存区域。这意味着外部 RAM 中经常访问的变量几乎可以像内部 RAM 一样快速地读取和修改。但是，当访问大块数据（>32K）时，缓存可能不足，速度将回落到外部 RAM 的访问速度。此外，访问大块数据可以“推出”缓存的闪存，可能会使代码执行速度变慢。
 - 外部 RAM 不能用作任务堆栈内存;因此，`xTaskCreate` 和类似函数将始终为堆栈和任务 TCB 分配内部存储器，而 `xTaskCreateStatic` 类型函数将检查传递的缓冲区是否是内部的。但是，对于不以任何方式直接或间接调用 ROM 中的代码的任务， menuconfig 选项 `CONFIG_SPIRAM_ALLOW_STACK_EXTERNAL_MEMORY` 将消除 `xTaskCreateStatic` 中的检查，从而允许外部 RAM 中的任务堆栈。但是，不建议使用此方法。

因为有一些对内部存储器有特定需求的情况，但也可以使用 `malloc（）` 来耗尽内部存储器，因此有一个专门为无法从外部存储器中解析的请求而保留的池;分配任务堆栈，DMA 缓冲区和在禁用缓存时仍可访问的内存从此池中提取。此池的大小可在 menuconfig 中配置。

## 芯片版本

ESP32 的某些修订存在一些问题，这些问题会对外部 RAM 的使用产生影响。 这些内容记录在 ESP32 ECO 文档中。 特别是， ESP-IDF 以下列方式处理提到的错误：

### ESP32 rev v0

ESP-IDF 没有针对此版本硅片中的错误的解决方法，它不能用于将外部 PSRAM 映射到 ESP32s 主存储器映射中。

### ESP32 rev v1

当某些机器指令序列在外部存储器位置 （ESP32 ECO 3.2） 上运行时，此芯片版本中的错误会带来危险。 为了解决这个问题，编译 ESP-IDF 的 gcc 编译器已经扩展了一个标志： -mfix-esp32-psram-cache-issue。 将此标志传递给命令行上的 gcc，编译器可以解决这些序列，并只输出可以安全执行的代码。

在 ESP-IDF 中，当您选择 `CONFIG_SPIRAM_CACHE_WORKAROUND` 时，将启用此标志。 ESP-IDF 还采取其他措施确保不使用 PSRAM 访问和违规指令集的组合：它链接到使用 gcc 标志重新编译的 Newlib 版本，不使用某些 ROM 函数并为 WiFi 分配静态内存叠加。

## 参考资料

 - [原文链接](https://docs.espressif.com/projects/esp-idf/en/latest/api-guides/external-ram.html)
