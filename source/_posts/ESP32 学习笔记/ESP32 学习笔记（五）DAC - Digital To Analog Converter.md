---
title: ESP32 学习笔记（五）DAC - Digital To Analog Converter
date: 2018-08-12 19:44:50
categories:
- ESP32 学习笔记
tags:
- ESP32
---

# DAC - Digital To Analog Converter

## 概述

ESP32 有两个 8 位 DAC (数模转换器)通道，连接到 GPIO25 (通道 1)和 GPIO26(通道 2)。

DAC 驱动器允许将这些通道设置为任意电压。

当使用“内置 DAC模式”时，DAC 通道也可以通过 I2S 驱动器使用 DMA 写入采样数据进行驱动。

有关其他模拟输出选项，请参阅 [Sigma-Delta 调制模块](https://esp-idf.readthedocs.io/en/latest/api-reference/peripherals/sigmadelta.html)和 [LED 控制模块](https://esp-idf.readthedocs.io/en/latest/api-reference/peripherals/ledc.html)。这两个模块都产生高频 PWM 输出，可以进行硬件低通滤波，以产生较低频率的模拟输出。

<!--more-->

## 应用示例

将 DAC 通道 1(GPIO 25)电压设置为 VDD_A 电压(VDD * 200/255)的约 0.78。对于 VDD_A 3.3V，这是 2.59V：

```c
#include <driver/dac.h>
...
    dac_output_enable(DAC_CHANNEL_1);
    dac_output_voltage(DAC_CHANNEL_1, 200);
```

## API Reference

### Header File

 - [driver/include/driver/dac.h](https://github.com/espressif/esp-idf/blob/30545f4/components/driver/include/driver/dac.h)

## 参考资料

 - [DAC](https://docs.espressif.com/projects/esp-idf/en/v3.2/api-reference/peripherals/dac.html)
