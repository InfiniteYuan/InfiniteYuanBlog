---
title: ESP32 学习笔记（十五）Touch Sensor
date: 2018-08-13 21:23:49
categories:
- ESP32 学习笔记
tags:
- ESP32
---

# Touch Sensor

## 介绍

触摸传感器系统构建在基板上，该基板在保护性平坦表面下承载电极和相关连接。当用户触摸表面时，触发电容变化并产生二进制信号以指示触摸是否有效。

ESP32 可提供多达 10 个电容式触摸板/GPIO。传感垫可以以不同的组合（例如矩阵，滑块）布置，从而可以检测更大面积或更多点。触摸板感测过程在硬件实现的有限状态机（FSM）的控制下，该有限状态机由软件或专用硬件定时器启动。

[ESP32 技术参考手册（PDF）](https://espressif.com/sites/default/files/documentation/esp32_technical_reference_manual_en.pdf)中讨论了触摸传感器的设计，操作和控制寄存器。有关此子系统如何工作的更多详细信息，请参阅它。

有关ESP32的触摸传感器设计和固件开发指南的详细信息，请参阅[触摸传感器应用说明](https://github.com/espressif/esp-iot-solution/blob/master/documents/touch_pad_solution/touch_sensor_design_en.md)。如果您想在各种配置下测试触摸传感器而无需自行构建，请查看[ESP32-Sense 开发套件指南](https://github.com/espressif/esp-iot-solution/blob/master/documents/evaluation_boards/esp32_sense_kit_guide_en.md)。

## 功能概述

API 的描述分为几组功能，以提供以下功能的快速概述：

 * 触摸板驱动器的初始化
 * 触摸板 GPIO 引脚的配置
 * 进行测量
 * 调整测量参数
 * 过滤测量值
 * 触摸检测方法
 * 设置中断以报告触摸检测
 * 在中断时从睡眠模式唤醒

有关特定功能的详细说明，请转到 [API 参考](#API-Reference)部分。[应用示例](#%E5%BA%94%E7%94%A8%E7%A4%BA%E4%BE%8B)部分介绍了此 API 的实际实现。

<!--more-->

### 初始化

触摸板驱动程序应在使用前通过调用函数 `touch_pad_init()` 进行初始化。此函数在“宏”下的 [API 参考](#API-Reference)中设置了几个 `.._ DEFAULT` 驱动程序参数。它还清除之前触摸过的焊盘信息（如果有）并禁用中断。

如果不再需要，可以通过调用 `touch_pad_deinit()` 来禁用驱动程序。

### 配置

使用 `touch_pad_config()` 可以为特定 GPIO 启用触摸传感器功能。

函数 `touch_pad_set_fsm_mode()` 用于选择是否由硬件定时器或软件自动启动触摸板测量（由 FSM 操作）。如果选择了软件模式，则使用 `touch_pad_sw_start()` 启动 FSM。

### 触摸状态测量

以下两个功能可以方便地从传感器读取原始或过滤的测量值：

 * `touch_pad_read()`
 * `touch_pad_read_filtered()`

通过在触摸或释放垫时检查传感器读数的范围，它们可用于表征特定的触摸板设计。然后可以使用该信息来建立触摸阈值。

>通过调用下面描述的特定过滤器函数，在使用`touch_pad_read_filtered()`之前启动并配置过滤器。

要了解如何使用这两种读取功能，请检查 [peripherals/touch_pad_read](https://github.com/espressif/esp-idf/tree/30545f4/examples/peripherals/touch_pad_read)应用示例。

### 优化测量

触摸传感器具有多个可配置参数，以匹配特定触摸板设计的特性。例如，为了感测较小的容量变化，可以缩小触摸板充电/放电的参考电压范围。使用 `touch_pad_set_voltage()` 函数设置高和低参考电压。除了识别较小容量变化的能力之外，积极的副作用还将是降低低功率应用的功耗。可能的负面影响是测量噪声的增加。如果获得的读数的动态范围仍然令人满意，则可以通过用 `touch_pad_set_meas_time()` 降低测量时间来进一步降低功耗。

以下总结了可用的测量参数和相应的“设置”功能：

 * 触摸板充电/放电参数：
	 * 电压范围：`touch_pad_set_voltage()`
	 * 速度（斜率）：`touch_pad_set_cnt_mode()`
 * 测量时间：`touch_pad_set_meas_time()`

电压范围（高/低参考电压），速度（斜率）和测量时间之间的关系如下图所示。

![这里写图片描述](https://img-blog.csdn.net/2018081321171024?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

最后一个图表“输出”表示触摸传感器读数，即在测量时间内收集的脉冲计数。

所有函数成对提供以“设置”特定参数并“获取”当前参数的值，例如，` touch_pad_set_voltage()` 和 `touch_pad_get_voltage()`。

### 过滤测量

如果测量结果有噪音，您可以使用提供的 API 过滤它们。首次使用前应通过调用 `touch_pad_filter_start()` 启动过滤器。

滤波器类型是 IIR（无限脉冲响应），它具有可配置的周期，可以通过功能 `touch_pad_set_filter_period()` 进行设置。

您可以使用 `touch_pad_filter_stop()` 停止过滤器。如果不再需要，可以通过调用 `touch_pad_filter_delete()` 删除过滤器。

### 触摸检测

触摸检测在 ESP32 的硬件中实现，基于用户配置的阈值和 FSM 执行的原始测量。使用 `touch_pad_get_status()` 函数检查触摸的触摸板和`touch_pad_clear_status()`以清除触摸状态信息。

硬件触摸检测也可以连接到中断，这将在下一节中介绍。

如果测量结果有噪声且容量变化很小，则硬件触摸检测可能不可靠。要解决此问题，请在您自己的应用程序中实施测量过滤并执行触摸检测，而不是使用硬件检测/提供的中断。有关两种触摸检测方法的示例实现，请参阅[peripherals / touch_pad_interrupt](https://github.com/espressif/esp-idf/tree/30545f4/examples/peripherals/touch_pad_interrupt)。

### 触摸触发中断

在触摸检测中启用中断之前，用户应建立触摸检测阈值。当触摸和释放打击垫时，使用上述功能读取和显示传感器测量值。当测量结果有噪声并且相对变化很小时，应用滤波器。根据您的应用和环境条件，测试温度和电源电压变化对测量值的影响。

一旦建立了检测阈值，就可以在初始化时使用 `touch_pad_config()` 或在运行时使用 `touch_pad_set_thresh()` 进行设置。

在下一步中，配置如何触发中断。可以在低于或高于阈值的情况下触发它们，并使用函数 `touch_pad_set_trigger_mode()` 进行设置。

最后使用以下函数配置和管理中断调用：

 * `touch_pad_isr_register()`/ `touch_pad_isr_deregister()`
 * `touch_pad_intr_enable()`/ `touch_pad_intr_disable()`

当中断运行时，您可以通过调用 `touch_pad_get_status()` 并使用 `touch_pad_clear_status()` 清除焊盘状态来获取特定焊盘触发中断的信息。

>触摸检测中断根据用户建立的阈值检查原始/未过滤测量，并在硬件中实现。启用软件过滤API（请参阅过滤度量）不会影响此过程。

### 从睡眠模式唤醒

如果使用触摸板中断将芯片从休眠模式唤醒，则用户可以选择应触摸的某些焊盘配置（SET1 或 SET1 和 SET2），以触发中断并导致后续唤醒。为此，请使用 `touch_pad_set_trigger_source()` 函数。

可以通过以下方式为每个 'SET' 管理所需的焊盘位模式的配置：

 * `touch_pad_set_group_mask()`/` touch_pad_get_group_mask()`
 * `touch_pad_clear_group_mask()`

## 应用示例

触摸传感器读取示例：[peripherals/touch_pad_read](https://github.com/espressif/esp-idf/tree/30545f4/examples/peripherals/touch_pad_read).
触摸传感器中断示例：[peripherals/touch_pad_interrupt](https://github.com/espressif/esp-idf/tree/30545f4/examples/peripherals/touch_pad_interrupt).

## API Reference

### Header File

 * [driver/include/driver/touch_pad.h](https://github.com/espressif/esp-idf/blob/30545f4/components/driver/include/driver/touch_pad.h)

## 参考资料

 - [Touch Sensor](https://docs.espressif.com/projects/esp-idf/en/v3.2/api-reference/peripherals/touch_pad.html)
