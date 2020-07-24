---
title: ESP32 学习笔记（十九）High Resolution Timer
date: 2018-12-24 11:22:42
categories:
- ESP32 学习笔记
tags:
- ESP32
---

# 高分辨率定时器

## 概述

虽然 FreeRTOS 提供软件定时器，但这些定时器有一些限制：

- 最大分辨率等于 RTOS 滴答周期
- 低优先级任务调度定时器回调

硬件定时器没有这两个限制，但通常使用起来不方便。例如，应用程序组件可能需要在将来的某些时间触发定时器事件，但硬件定时器仅包含一个用于中断生成的“比较”值。这意味着需要在硬件计时器之上构建一些设施来管理待处理事件列表，可以在发生相应的硬件中断时为这些事件调度回调。

`esp_timer` API 集提供了这样的功能。在内部，`esp_timer` 使用 32 位硬件定时器（FRC1，“传统”定时器）。`esp_timer` 提供单次和周期定时器，微秒时间分辨率和 64 位范围。

从高优先级的 `esp_timer` 任务调度定时器回调。由于所有回调都是从同一任务调度的，因此建议仅从回调本身执行尽可能少的工作，而是使用队列将事件发布到优先级较低的任务。

如果优先级高于 `esp_timer` 的其他任务正在运行，则回调调度将延迟，直到 `esp_timer` 任务有机会运行。例如，如果正在进行 `SPI` 闪存操作，则会发生这种情况。

创建和启动计时器，并调度回调需要一些时间。因此，单触发 `esp_timer` 的超时值有一个下限。如果调用 `esp_timer_start_once()` 时超时值小于20us，则仅在大约 20us 后调度回调。

周期性 `esp_timer` 还对最小定时器周期施加 50us 限制。周期小于 50us 的定期软件定时器不实用，因为它们会占用大部分 CPU 时间。如果发现需要小周期的定时器，请考虑使用专用硬件外设或 DMA 功能。

<!--more-->

## 使用 `esp_timer` API

单个计时器由 `esp_timer_handle_t` 类型表示。定时器有与之关联的回调函数。每次计时器过去时，都会从 `esp_timer` 任务调用此回调函数。

- 要创建计时器，请调用 `esp_timer_create()`。
- 要在不再需要时删除计时器，请调用 `esp_timer_delete()`。

定时器可以以单次模式或定期模式启动。

- 要以单次模式启动计时器，请调用 `esp_timer_start_once()`，传递应该调用回调的时间间隔。调用回调时，会认为计时器已停止。
- 要以周期模式启动定时器，请调用 `esp_timer_start_periodic()`，传递应调用回调的周期。计时器一直运行，直到调用 `esp_timer_stop()`。

请注意，调用 `esp_timer_start_once()` 或 `esp_timer_start_periodic()` 时，计时器不能运行。要重新启动正在运行的计时器，请先调用 `esp_timer_stop()`，然后调用其中一个启动函数。

## 获得当前时间

`esp_timer` 还提供了一个便捷函数来获取自启动以来经过的时间，精度为微秒：`esp_timer_get_time()`。此函数返回自 `esp_timer` 初始化以来的微秒数，这通常在调用 `app_main` 函数之前不久发生。

与 `gettimeofday` 函数不同，`esp_timer_get_time()` 返回的值：

- 芯片从深度睡眠中唤醒后，从零开始
- 没有应用时区或 DST 调整

## 应用示例

 - [system/esp_timer](https://github.com/espressif/esp-idf/tree/7b13308/examples/system/esp_timer)

## API 参考

 - [esp32/include/esp_timer.h](https://github.com/espressif/esp-idf/blob/7b13308/components/esp32/include/esp_timer.h)

## 参考资料

 - [High Resolution Timer](https://docs.espressif.com/projects/esp-idf/en/v3.2/api-reference/system/esp_timer.html)
