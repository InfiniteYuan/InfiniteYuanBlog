---
title: ESP32 学习笔记（十）Timer
date: 2018-08-13 20:36:36
categories:
- ESP32 学习笔记
tags:
- ESP32
---

# Timer

## 介绍

ESP32 芯片包含两个硬件定时器组。每组有两个通用硬件定时器。它们都是基于 16 位预分频器和 64 位自动重载功能的向上/向下计数器的 64 位通用定时器。

## 功能概述

以下各节介绍了配置操作定时器的典型步骤：
 
 * [定时器初始化](#定时器初始化) - 应设置哪些参数以使定时器工作以及根据设置提供的具体功能。
 * [定时器控制](#定时器控制) - 如何读取定时器的值，暂停/启动定时器，以及如何操作。
 * [警报](#警报) - 设置和使用警报。
 * [中断](#中断) - 如何启用和使用中断。

<!--more-->

### 定时器初始化

使用 `timer_group_t` 标识 ESP32 的两个定时器组。使用 `timer_idx_t` 标识组中的各个定时器。每个定时器组都有两个定时器，总共提供四个定时器。

在启动定时器之前，应该通过调用 `timer_init()` 来初始化定时器。应该为此函数提供结构 `timer_config_t`，以定义定时器应如何工作。特别是以下定时器参数的设置：

 * **分频器**：定时器的计数器“滴答作响”的速度有多快。这取决于分频器的设置，它将用作输入的 80 MHz APB_CLK 时钟的除数。
 * **模式**：`counter_dir` 决定了计数器递增或递减，通过从 `timer_count_dir_t` 中选择一个值来设置 `counter_dir` 。
 * **计数器启用**：如果计数器已启用，则在调用 `timer_init()` 后它将立即开始递增/递减。通过从 `timer_start_t` 中选择一个值，使用 `counter_en` 设置此操作。
 * **报警启用**：由 `alarm_en` 的设置决定。
 * **自动重新加载**：计数器是否应在定时器的警报上自动重新加载特定的初始值，或者继续递增或递减。
 * **中断类型**：定时器报警是否触发中断。设置 `timer_intr_mode_t` 中定义的值。

要获取定时器设置的当前值，请使用函数 `timer_get_config()`。

### 定时器控制

一旦配置并启用了定时器，它就已经“滴答作响”了。要检查它的当前值，请调用 `timer_get_counter_value()` 或 `timer_get_counter_time_sec()`。要将定时器设置为特定的起始值，请调用 `timer_set_counter_value()`。

可以通过调用 `timer_pause()` 随时暂停定时器。要再次启动它，请调用 `timer_start()`。

要更改定时器的运行方式，可以再次调用 `Timer_init()`，如 [Timer Initialization](#定时器初始化) 部分所述。另一种选择是使用专用功能来更改个别设置：

 * **分频器值** - `timer_set_divider()`。注意：更改分频器时应暂停定时器以避免不可预测的结果。如果定时器已在运行，`timer_set_divider()` 将首先暂停定时器，更改分频器，最后再次启动定时器。
 * **模式**(计数器是递增还是递减) - `timer_set_counter_mode()`
 * **报警时自动重载计数器** - `timer_set_auto_reload()`

### 警报

要设置警报，请调用函数 `timer_set_alarm_value()`，然后使用 `timer_set_alarm()` 启用它。当调用 `timer_init()` 时，也可以在定时器初始化阶段启用警报。

启用警报并且定时器达到警报值后，根据配置，可能会发生以下两种操作：

 * 如果先前已配置，将触发中断。请参见中断如何配置中断一节。
 * 启用 `auto_reload` 后，将重新加载定时器的计数器以从特定的初始值开始计数。应使用 `timer_set_counter_value()` 预先设置要启动的值。

> * 如果设置了警报值并且定时器已经超过该值，则将立即触发警报。
> * 触发后，警报将自动禁用，需要重新启动以再次触发。

要检查已设置的警报值，请调用 `timer_get_alarm_value()`。

### 中断

为特定定时器组和定时器注册中断处理程序是通过调用 `timer_isr_register()` 来完成的。

要为定时器组启用中断，请调用 `timer_group_intr_enable()`。要为特定定时器执行此操作，请调用 `timer_enable_intr()`。使用相应的函数 `timer_group_intr_disable()` 和 `timer_disable_intr()` 来禁用中断。

在 ISR 中处理中断时，需要明确清除中断。为此，请设置定义在 [soc/esp32/include/soc/timer_group_struct.h](https://github.com/espressif/esp-idf/blob/30545f4/components/soc/esp32/include/soc/timer_group_struct.h)里的 `TIMERGN.int_clr_timers.tM` 结构，其中 N 是定时器组号 [0,1]，M 是定时器号 [0,1]。例如，要清除定时器组 0 中定时器 1 的中断，请调用以下命令：

```c
TIMERG0.int_clr_timers.t1 = 1
```
请参阅下面的应用示例如何使用中断。

## 应用示例

64 位硬件定时器示例：[peripherals/timer_group](https://github.com/espressif/esp-idf/tree/30545f4/examples/peripherals/timer_group).

## API Reference

### Header File

 - [driver/include/driver/timer.h](https://github.com/espressif/esp-idf/blob/30545f4/components/driver/include/driver/timer.h)

## 参考资料

 - [Timer](https://docs.espressif.com/projects/esp-idf/en/v3.2/api-reference/peripherals/timer.html)
