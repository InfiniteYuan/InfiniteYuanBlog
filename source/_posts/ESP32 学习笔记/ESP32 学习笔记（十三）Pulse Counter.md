---
title: ESP32 学习笔记（十三）Pulse Counter
date: 2018-08-13 21:05:12
categories:
- ESP32 学习笔记
tags:
- ESP32
---

# Pulse Counter

## 介绍

PCNT（脉冲计数器）模块用于计算输入信号的上升沿和/或下降沿的数量。每个脉冲计数器单元都有一个 16 位有符号计数器寄存器和两个通道，可配置为递增或递减计数器。每个通道都有一个接收待检测信号边沿的信号输入，以及一个可用于启用或禁用信号输入的控制输入。输入具有可选滤波器，可用于丢弃信号中不需要的毛刺。

## 功能概述

此API的功能描述分为四个部分：

 * 配置 - 描述计数器的配置参数以及如何设置计数器。
 * 操作计数器 - 提供有关暂停，测量和清除计数器的控制功能的信息。
 * 滤波脉冲 - 描述滤波脉冲和计数器控制信号的选项。
 * 使用中断 - 介绍如何在计数器的特定状态上触发中断。

<!--more-->

### 配置

PCNT 模块有 8 个独立的计数“单位”，编号从 0 到 7。在 API 中，它们使用 `pcnt_unit_t` 引用。每个单元有两个独立的通道，编号为 0 和 1，并用 `pcnt_channel_t` 指定。

使用 `pcnt_config_t` 为每个单元的通道单独提供配置，并涵盖：

 * 此配置所指的单元和通道编号。
 * 脉冲输入和脉冲门输入的 GPIO 编号。
 * 两对参数：`pcnt_ctrl_mode_t` 和 `pcnt_count_mode_t`，用于定义计数器如何响应，具体取决于控制信号的状态以及如何计算脉冲的正/负边沿。
 * 当脉冲计数满足特定限制时，用于建立观察点和触发中断的两个极限值（最小值/最大值）。

然后通过调用上面的 `pcnt_config_t` 作为输入参数的函数 `pcnt_unit_config()` 来完成特定通道的设置。

要在配置中禁用脉冲或控制输入引脚，请提供 `PCNT_PIN_NOT_USED` 而不是 GPIO 编号。

### 操作计数器

在使用 `pcnt_unit_config()` 进行设置后，计数器立即开始运行。可以通过调用 `pcnt_get_counter_value()` 来检查累积的脉冲计数。

有几个函数可以控制计数器的操作：`pcnt_counter_pause()`，`pcnt_counter_resume()` 和 `pcnt_counter_clear()`

也可以通过调用 `pcnt_set_mode()` 使用 `pcnt_unit_config()` 动态更改先前设置的计数器模式。

如果需要，可以使用 `pcnt_set_pin()` “动态”更改脉冲输入引脚和控制输入引脚。要禁用特定输入，请提供功能参数 `PCNT_PIN_NOT_USED` 而不是GPIO编号。

>为了使计数器不会错过任何脉冲，脉冲持续时间应该长于一个APB_CLK周期（12.5 ns）。脉冲在APB_CLK时钟的边沿上采样，如果在边缘之间落下，则可能会丢失。这适用于有或没有文件管理器的计数器操作。

### 滤波脉冲

PCNT 单元在每个脉冲和控制输入上都有滤波器，增加了忽略信号中短毛刺的选项。

通过调用 `pcnt_set_filter_value()` 在 APB_CLK 时钟周期中提供忽略脉冲的长度。可以使用 `pcnt_get_filter_value()` 检查当前过滤器设置。APB_CLK 时钟以 80 MHz 运行。

通过调用 `pcnt_filter_enable()`/`pcnt_filter_disable()` 将过滤器置于操作/暂停状态。

### 使用中断

在 `pcnt_evt_type_t` 中定义的五个计数器状态监视事件能够触发中断。事件发生在脉冲计数器达到特定值：

 * 最小或最大计数值：在配置中讨论的 `pcnt_config_t` 中提供的 `counter_l_lim` 或 `counter_h_lim`
 * 使用函数 `pcnt_set_event_value()` 设置阈值 0 或阈值 1 值。
 * 脉冲计数 = 0

要注册，启用或禁用中断以服务上述事件，请调用 `pcnt_isr_register()`，`pcnt_intr_enable()` 和 `pcnt_intr_disable()`。要在达到阈值时启用或禁用事件，您还需要调用函数 `pcnt_event_enable()` 和 `pcnt_event_disable()`。

要检查当前设置的阈值，请使用函数 `pcnt_get_event_value()`。

## 应用示例

带控制信号和事件中断的脉冲计数器示例：[peripherals/pcnt](https://github.com/espressif/esp-idf/tree/30545f4/examples/peripherals/pcnt).

## API Reference

### Header File
 * [driver/include/driver/pcnt.h](https://github.com/espressif/esp-idf/blob/30545f4/components/driver/include/driver/pcnt.h)

## 参考资料

 - [Pulse Counter](https://docs.espressif.com/projects/esp-idf/en/v3.2/api-reference/peripherals/pcnt.html)
