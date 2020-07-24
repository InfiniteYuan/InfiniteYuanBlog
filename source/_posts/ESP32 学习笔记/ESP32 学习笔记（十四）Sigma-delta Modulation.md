---
title: ESP32 学习笔记（十四）Sigma-delta Modulation
date: 2018-08-13 21:09:18
categories:
- ESP32 学习笔记
tags:
- ESP32
---

# Sigma-delta Modulation

## 介绍

ESP32 有一个二阶 sigma-delta 调制模块。此驱动程序可配置 sigma-delta 模块的通道。

## 功能概述

八个独立的 sigma-delta 调制信道用 `sigmadelta_channel_t` 进行标识。每个通道都能够输出 sigma-delta 调制模块生成的二进制硬件信号。

应通过在 `sigmadelta_config_t` 中提供配置参数然后将此配置应用于 `sigmadelta_config()` 来设置所选通道。

另一种选择是调用各个函数，逐个配置所有必需参数：

 * sigma-delta 调制模块的预分频器 - `sigmadelta_set_prescale()`
 * 输出信号的占空比 - `sigmadelta_set_duty()`
 * 输出调制信号的 GPIO 引脚 - `sigmadelta_set_pin()`

`sigmadelta_set_duty()` 的 `duty` 参数的范围是 -128 到 127（八位有符号整数）。如果设置了零值，那么输出信号的占空比将为 50％，参见 `sigmadelta_set_duty()` 的描述。

<!--more-->

## 应用示例

Σ-Δ调制示例：[peripherals/sigmadelta](https://github.com/espressif/esp-idf/tree/30545f4/examples/peripherals/sigmadelta).

## API Reference

### Header File

 * [driver/include/driver/sigmadelta.h](https://github.com/espressif/esp-idf/blob/30545f4/components/driver/include/driver/sigmadelta.h)

## 参考资料

 - [Sigma-delta Modulation](https://docs.espressif.com/projects/esp-idf/en/v3.2/api-reference/peripherals/sigmadelta.html)
