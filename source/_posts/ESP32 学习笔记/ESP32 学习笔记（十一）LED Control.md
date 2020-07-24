---
title: ESP32 学习笔记（十一）LED Control
date: 2018-08-13 20:53:55
categories:
- ESP32 学习笔记
tags:
- ESP32
---

# LED Control

## 介绍

LED 控制(LEDC)模块主要用于控制 LED 的强度，尽管它也可用于生成 PWM 信号以用于其他目的。它有16个通道可以产生独立的波形，可以用来驱动例如: RGB LED设备。

所有 LEDC 通道中有一半提供高速操作模式。该模式提供硬件实现，自动和无干扰的 PWM 占空比改变。另一半通道在低速模式下运行，其中变化的时刻取决于应用软件。每组通道也能够使用不同的时钟源，但 API 中未实现此功能。

PWM 控制器还能够逐渐自动增加或减少占空比，允许在没有任何处理器干扰的情况下衰减。

<!--more-->

## 功能概述

让 LEDC 在高速或低速模式下在特定通道上工作分三步完成：

 * 配置定时器以确定 PWM 信号的频率和数字(占空比范围分辨率)。
 * 通过将通道与定时器和 GPIO 相关联来配置通道，以输出 PWM 信号。
 * 更改 PWM 信号，驱动输出以改变 LED 的强度。这可以在软件的完全控制下或在硬件衰落功能的帮助下完成。

在可选步骤中，还可以在淡入淡出端设置中断。
![这里写图片描述](https://img-blog.csdn.net/20180813204054898?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

### 配置定时器

通过调用函数 `ledc_timer_config()` 来完成定时器的设置。应为此函数提供包含以下配置设置的数据结构 `ledc_timer_config_t`：

 * 定时器号 `ledc_timer_t` 和速度模式 `ledc_mode_t`。
 * PWM 信号的频率和 PWM 占空比值的分辨率会发生变化。

频率和占空比分辨率是相互依赖的。PWM 频率越高，可提供更低的占空比分辨率，反之亦然。如果您计划将此 API 用于其他改变 LED 强度的目的，这种关系可能会变得很重要。有关更多详细信息，请查看[支持的频率范围和占空比分辨率](#支持的频率范围和占空比分辨率)。

### 配置通道

设置定时器后，下一步是配置所选通道( `ledc_channel_t` 中的一个)。这是通过调用函数 `ledc_channel_config()` 来完成的。

以类似的方式，与定时器配置类似，应为通道设置功能提供特定结构 `ledc_channel_config_t`，其中包含通道的配置参数。

此时，通道应开始工作，并开始生成由定时器设置和所选 GPIO 上的占空比确定的频率 PWM 信号，如 `ledc_channel_config_t` 中所配置。可以通过调用函数 `ledc_stop()` 随时暂停通道操作/信号生成。

### 改变 PWM 信号

一旦通道工作并产生恒定占空比和频率的PWM信号，有几种方法可以改变这个信号。在驱动LED时，我们主要改变了改变光强度的占空比。请参阅下面的两部分，了解如何通过软件或硬件衰减来更改占空比。如果需要，我们也可以更改信号的频率，这将在[更改PWM频率](#更改PWM频率)部分中介绍。

#### 通过软件更改PWM占空比

通过首先调用专用函数 `ledc_set_duty()` 然后调用 `ledc_update_duty()` 来使更改生效来完成占空比的设置。要检查当前设置的值，有一个相应的 `_get_` 函数 `ledc_get_duty()`。

设置占空比和其他一些通道参数的另一种方法是调用上一节中讨论的 `ledc_channel_config()`。

输入函数的占空比值的范围取决于所选的 `duty_resolution`，并且应该从 0 到 (2 ** duty_resolution) - 1。例如，如果选择的占空比分辨率为 10，则占空比范围为 0 到 1023。这提供了分辨率为~0.1％。

#### 通过硬件衰落改变PWM占空比

LEDC 硬件提供了从一个占空值逐渐淡入另一个值的方法。要使用此功能，首先使用 `ledc_fade_func_install()` 启用淡入淡出。然后通过调用一个可用的淡入淡出函数来配置它：

 * `ledc_set_fade_with_time()`
 * `ledc_set_fade_with_step()`
 * `ledc_set_fade()`

最后用 `ledc_fade_start()` 开始淡出。

如果不再需要，可以使用 `ledc_fade_func_uninstall()` 禁用衰落和相关中断。

#### 更改PWM频率

LEDC API 提供了几种“动态”改变 PWM 频率的方法。

 * 其中一个选项是调用 `ledc_set_freq()`。有一个相应的函数 `ledc_get_freq()` 来检查当前设置的频率。
 * 另一种改变频率和占空比分辨率的方法是调用 `ledc_bind_channel_timer()` 将其他定时器绑定到通道。
 * 最后，可以通过调用 `ledc_channel_config()` 来更改通道的计时器。

#### 更多控制PWM

有几个低级定时器特定功能，可用于提供更改 PWM 设置的其他方法：

 * `ledc_timer_set()`
 * `ledc_timer_rst()`
 * `ledc_timer_pause()`
 * `ledc_timer_resume()`

前两个函数被 `ledc_channel_config()` 称为“幕后”，以便在配置后提供“干净”的计时器启动。

### 使用中断

配置 LEDC 通道时，在 `ledc_channel_config_t` 中选择的参数之一是 `ledc_intr_type_t`，并允许在淡入淡出完成时启用中断。

通过调用 `ledc_isr_register()` 来注册处理此中断的处理程序。


## LEDC高低速模式

LED PWM 控制器中共有 8 个定时器和 16 个通道，其中一半专用于高速模式，另一半专用于低速模式。选择低速或高速“有能力”定时器或通道是通过适用的函数调用中存在的参数 `ledc_mode_t` 来完成的。

高速模式的优点是支持 h/w，定时器设置无故障切换。这意味着如果修改了定时器设置，则在定时器的下一个溢出中断后将自动应用更改。相反，在更新低速定时器时，应特别由软件触发设置更改。LEDC API 正在“幕后”进行，例如，当调用 `ledc_timer_config()` 或 `ledc_timer_set()` 时。

有关速度模式的更多详细信息，请参阅 ESP32 技术参考手册(PDF)。请注意，本手册中提到的对 `SLOW_CLOCK` 的支持未在LEDC API 中实现。

## 支持的频率和占空比分辨率范围

LED PWM 控制器主要用于驱动 LED，并提供宽泛的 PWM 占空比设置。例如，对于 5 kHz 的 PWM 频率，最大占空比分辨率为 13 位。这意味着占空比可以设置在 0 到 100％ 之间，分辨率为~0.012％(13 ** 2 = 8192 LED 强度的离散电平)。

LEDC 可以用于以更高的频率提供信号以对其他设备进行计时，例如，数码相机模块。在这种情况下，最大可用频率为 40 MHz，占空比分辨率为 1 位。这意味着频率固定在 50％，无法调整。

API 用于在尝试设置超出 LEDC 硬件范围的频率和占空比分辨率时报告错误。例如，尝试将频率设置为 20 MHz 且占空比分辨率为 3 位将导致串行监视器上报告以下错误：

```
E (196) ledc: requested frequency and duty resolution can not be achieved, try reducing freq_hz or duty_resolution. div_param=128
```

在这种情况下，应降低占空比分辨率或频率。例如，将占空比分辨率设置为 2 将解决该问题，并提供以 25％ 步长设置占空比的可能性，即 25％，50％或75％。

LEDC API 还将捕获并报告尝试配置低于支持的最小值的频率/任务分辨率组合，例如：

```
E (196) ledc: requested frequency and duty resolution can not be achieved, try increasing freq_hz or duty_resolution. div_param=128000000
```

通常使用 `ledc_timer_bit_t` 来设置占空比分辨率。该枚举涵盖 10 到 15 位的范围。如果需要较小的占空比分辨率(低于 10 至 1)，请直接输入等效数值。

## 应用示例

LEDC 改变占空比和衰落控制示例：[peripherals/ledc](https://github.com/espressif/esp-idf/tree/30545f4/examples/peripherals/ledc).

## API Reference

### Header File

 * [driver/include/driver/ledc.h](https://github.com/espressif/esp-idf/blob/30545f4/components/driver/include/driver/ledc.h)

## 参考资料

 - [LED Control](https://docs.espressif.com/projects/esp-idf/en/v3.2/api-reference/peripherals/ledc.html)
