---
title: ESP32 学习笔记（二十）RTC 和低功耗管理
date: 2019-03-04 10:10:42
categories:
- ESP32 学习笔记
tags:
- ESP32
---

# RTC 和低功耗管理

ESP32 采用了先进的电源管理技术，可以在不同的功耗模式之间切换。

## 1 功耗模式

- Active 模式:芯片射频处于工作状态。芯片可以接收、发射和侦听信号。
- Modem-sleep 模式:CPU 可运行，时钟可被配置。Wi-Fi/蓝牙基带和射频关闭。
- Light-sleep 模式:CPU 暂停运行。RTC 存储器和外设以及 ULP 协处理器运行。任何唤醒事件(MAC、主机、RTC 定时器或外部中断)都会唤醒芯片。
- Deep-sleep 模式:CPU 和大部分外设都会掉电，只有 RTC 存储器和 RTC 外设处于工作状态。Wi-Fi 和蓝牙连接数据存储在 RTC 中。ULP 协处理器可以工作。
- Hibernation 模式:内置的 8 MHz 振荡器和 ULP 协处理器均被禁用。RTC 内存恢复电源被切断。只有1 个位于低速时钟上的 RTC 时钟定时器和某些 RTC GPIO 在工作。RTC 时钟定时器或 RTC GPIO 可以将芯片从 Hibernation 模式中唤醒。

<!--more-->

## 2 低功耗模式功耗

设备在不同的功耗模式下有不同的电流消耗，详情请见下表。

![不同功耗模式下的功耗](https://img-blog.csdnimg.cn/20190304100817958.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3,size_16,color_FFFFFF,t_70)![射频功耗参数](https://img-blog.csdnimg.cn/201903041011254.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3,size_16,color_FFFFFF,t_70)

## 3 说明

 - ESP32 系列芯片中，ESP32-D0WDQ6 与 ESP32-D0WD 的 CPU 最大频率为 240 MHz;ESP32-D2WD 与 ESP32-S0WD 的 CPU 最大频率为 160 MHz。
 - 在 Wi-Fi 开启的场景中，芯片会在 Active 和 Modem-sleep 模式之间切换，功耗也会在两种模式间变化。
 - Modem-sleep 模式下，CPU 频率自动变化，频率取决于 CPU 负载和使用的外设。
 - Deep-sleep 模式下，仅 ULP 协处理器处于工作状态时，可以操作 GPIO 及低功耗 I2C。
 - 当系统处于超低功耗传感器监测模式时，ULP 协处理器和传感器周期性工作,ADC 以 1% 占空比工作，系统功耗典型值为 100 μA。

## 4 参考资料

 - [ESP32 技术参考手册](https://www.espressif.com/sites/default/files/documentation/esp32_technical_reference_manual_en.pdf)
 - [ESP32 学习笔记（二十九） ESP32 低功耗模式](https://infiniteyuan.blog.csdn.net/article/details/106451678)
