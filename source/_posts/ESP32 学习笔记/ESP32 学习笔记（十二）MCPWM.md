---
title: ESP32 学习笔记（十二）MCPWM
date: 2018-08-13 20:57:00
categories:
- ESP32 学习笔记
tags:
- ESP32
---

# MCPWM

## 概述

ESP32 有两个 MCPWM 单元，可用于控制不同的电机。每个单元有三对 PWM 输出。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20181228185337698.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3,size_16,color_FFFFFF,t_70)
此外，在文档中，单个单元的输出标记为 PWMxA/PWMxB。

<!--more-->

MCPWM 单元的更详细框图如下所示。每个 A/B 对可以由定时器 0,1 和 2 的三个定时器中的任何一个定时。相同的定时器可以用于为一对以上的 PWM 输出提供时钟。每个单元还能够收集诸如 `SYNC SIGNALS` 之类的输入，检测诸如电动机过电流或过电压之类的 `FAULT SIGNALS`，以及使用 `CAPTURE SIGNALS` 获得反馈信号例如：转子位置。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20181228185639802.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3,size_16,color_FFFFFF,t_70)
此 API 的说明首先配置 MCPWM 的定时器和操作子模块，以提供基本的电机控制功能。然后讨论了故障处理程序，信号捕获，载波和中断等更高级的子模块和功能。

## 目录

- 配置输出的基本功能
- 操作输出以驱动电机
- 调整电机的驱动方式
- 捕获外部信号以提供额外的控制
- 使用 `Fault Handler` 检测和管理故障
- 如果输出信号通过隔离变压器，则添加更高频率的载波
- 中断的配置和处理。

## 配置

配置范围取决于电机类型，特别是需要多少输出和输入，以及驱动电机的信号序列。

在这种情况下，我们将描述一种简单的配置来控制仅使用一些可用 MCPWM 资源的有刷直流电机。示例电路如下所示。它包括一个 H 桥，用于切换施加在电机（M）上的电压的极化，并提供足够的电流来驱动它。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20181228190041789.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3,size_16,color_FFFFFF,t_70)
配置包括以下步骤：

- 选择将用于驱动电机的 MPWM 单元。ESP32 上有两个可用的单元，并在 `mcpwm_unit_t` 中枚举。
- 通过调用 `mcpwm_gpio_init()` 将两个 GPIO 初始化为选定单元内的输出信号。两个输出信号通常用于命令电机向右或向左旋转。所有可用的信号选项都列在 `mcpwm_io_signals_t` 中。要一次设置多个引脚，请将函数 `mcpwm_set_pin()` 与 `mcpwm_pin_config_t` 一起使用。
- 选择计时器。该装置内有三个定时器。计时器列在 `mcpwm_timer_t` 中。
- 在 `mcpwm_config_t` 结构中设置定时器频率和初始占空比。
- 使用上述参数调用 `mcpwm_init()` 以使配置生效。

## 应用示例

使用 MCPWM 进行电机控制的示例：[peripherals/mcpwm](https://github.com/espressif/esp-idf/tree/30545f4/examples/peripherals/mcpwm).

## API Reference

### Header File

 * [driver/include/driver/mcpwm.h](https://github.com/espressif/esp-idf/blob/30545f4/components/driver/include/driver/mcpwm.h)

## 参考资料

 - [MCPWM](https://docs.espressif.com/projects/esp-idf/en/v3.2/api-reference/peripherals/mcpwm.html)
