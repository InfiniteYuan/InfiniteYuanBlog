---
title: ESP32 学习笔记（四）ADC - Analog to Digital Converter
date: 2018-08-11 22:26:14
categories:
- ESP32 学习笔记
tags:
- ESP32
---

# ADC - Analog to Digital Converter

## 概述

ESP32 集成了 2 个 12 位逐次逼近模数转换器 (SARADC),由 5 个专用转换器控制器管理.支持 18 个通道(模拟使能引脚)的测量. ADC 还可测量 vdd33 等内部信号.其中一些引脚可用于设计 1 个可编程增益放大器,用于测量微弱模拟信号.SAR ADC 使用的 5 个控制器均为专用控制器,其中 2 个支持高性能多通道扫描、2 个经过优化可支持 Deep-sleep 模式下的低功耗运行,另外 1 个专门用于 PWDET/ PKDET (功率检测和峰值监测).

<!--more-->

![图 1: SAR ADC 的概况](https://img-blog.csdn.net/20180824172548878?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
<center>图 1: SAR ADC 的概况</center>

ADC 驱动程序 API 支持 ADC1(8 个通道,连接到 GPIO 32-39)和 ADC2(10 个通道,连接到 GPIO 0,2,4,12-15和 25-27). 但是,使用 ADC2 的应用程序存在一些限制:

 1. 仅当 Wi-Fi 驱动程序未启动时,应用程序才能使用 ADC2,因为具有更高优先级的 Wi-Fi 驱动程序也使用 ADC.
 2. 某些 ADC2 引脚用作捆扎引脚(GPIO 0,2,15),因此无法自由使用. 例如,官方开发套件:
 * [ESP32 Core Board V2 / ESP32 DevKitC](https://esp-idf.readthedocs.io/en/latest/hw-reference/modules-and-boards.html#esp-modules-and-boards-esp32-devkitc):由于外部自动编程电路,无法使用 GPIO 0.
 * [ESP-WROVER-KIT V3](https://esp-idf.readthedocs.io/en/latest/hw-reference/modules-and-boards.html#esp-modules-and-boards-esp-wrover-kit-v3):由于外部连接用于不同目的,因此无法使用 GPIO 0,2,4 和 15.

## 主要特性

 - 采用 2 个 SAR ADC,可支持同时采样与转换
 - 采用 5 个专用 ADC 控制器,可支持不同应用场景(比如,高性能、低功耗,或功率检测和峰值检测)
 - 支持 18 个模拟输入管脚
 - 1 个内部电压 vdd33 通道、2 个 pa_pkdet 通道(部分控制器支持)
 - 可配置 12 位、11 位、10 位、9 位多种分辨率
 - 支持 DMA(1 个控制器支持)
 - 支持多通道扫描模式(2 个控制器支持)
 - 支持 Deep-sleep 模式运行(1 个控制器支持)
 - 支持 ULP 协处理器控制(2 个控制器支持)

## 功能概述

![图 2: SAR ADC 的功能概况](https://img-blog.csdn.net/20180824181445719?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
<center>图 2: SAR ADC 的功能概况</center>

![表 1: SAR ADC 的信号输入](https://img-blog.csdn.net/20180824181614793?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
<center>表 1: SAR ADC 的信号输入</center>
ESP32 内置了 5 个专用 ADC 控制器: RTC ADC1 CTRL、 RTC ADC2 CTRL、 DIG ADC1 CTRL、 DIG ADC2 CTRL,
及 PWDET CTRL。各控制器的场景支持情况见表 2

![表 2: ESP32 的 SAR ADC 控制器](https://img-blog.csdn.net/20180824184037450?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
<center>表 2: ESP32 的 SAR ADC 控制器</center>

## 配置和读取 ADC

在读取之前应配置 ADC.

 - 对于 ADC1,通过调用函数 `adc1_config_width()` 和 `adc1_config_channel_atten()` 来配置所需的精度和衰减.
 - 对于 ADC2,通过 `adc2_config_channel_atten()` 配置衰减. 每次读取时都会配置 ADC2 的读取宽度.

每个通道完成衰减配置,请参见 `adc1_channel_t` 和 `adc2_channel_t`,设置为上述功能的参数.

然后可以使用 `adc1_get_raw()` 和 `adc2_get_raw()` 读取 ADC 转换结果. 应将 ADC2 的读取宽度设置为 `adc2_get_raw()` 的参数,而不是配置函数中的参数.

>由于 ADC2 与具有更高优先级的 WIFI 模块共享,因此在 `esp_wifi_start()` 和 `esp_wifi_stop()` 之间,`adc2_get_raw()` 的读取操作将失败. 使用返回代码查看读数是否成功.

也可以通过调用专用功能 `hall_sensor_read()`通过 ADC1 读取内部霍尔效应传感器. 请注意,即使霍尔传感器在 ESP32 内部,读取它也使用 ADC1 的通道 0 和 3(GPIO 36 和 39). 请勿将其他任何东西连接到这些引脚,也不要更改其配置. 否则可能会影响来自 sesnor 的低值信号的测量.

此 API 提供了配置 ADC1 以便从 ULP 读取的便捷方法. 为此,请调用函数 `adc1_ulp_enable()`,然后如上所述设置精度和衰减.

还有另一个特定功能 `adc2_vref_to_gpio()` 用于将内部参考电压路由到 GPIO 引脚. 它可以方便地校准 ADC 读数,这将在最小化噪声部分中讨论.

## 应用示例

读取 ADC1 通道 0(GPIO 36)上的电压:
```c
#include <driver/adc.h>
...
    adc1_config_width(ADC_WIDTH_BIT_12);
    adc1_config_channel_atten(ADC1_CHANNEL_0,ADC_ATTEN_DB_0);
    int val = adc1_get_raw(ADC1_CHANNEL_0);
```
上例中的输入电压为 0 至 1.1V (衰减为 0 dB). 可以通过设置更高的衰减来扩展输入范围,请查看 `adc_atten_t`. esp-idf 中提供了使用 ADC 驱动程序的示例,包括校准(如下所述):[peripherals/adc](https://github.com/espressif/esp-idf/tree/f9a4496/examples/peripherals/adc)

读取 ADC2 通道 7 (GPIO 27)的电压:
```c
#include <driver/adc.h>
...
    int read_raw;
    adc2_config_channel_atten( ADC2_CHANNEL_7, ADC_ATTEN_0db );

    esp_err_t r = adc2_get_raw( ADC2_CHANNEL_7, ADC_WIDTH_12Bit, &read_raw);
    if ( r == ESP_OK ) {
        printf("%d\n", read_raw );
    } else if ( r == ESP_ERR_TIMEOUT ) {
        printf("ADC2 used by Wi-Fi.\n");
    }
```
由于与 Wi-Fi 冲突,读数可能会失败,应该检查. esp-idf 中提供了使用 ADC2 驱动程序读取 DAC 输出的示例:[peripherals/adc2](https://github.com/espressif/esp-idf/tree/30545f4/examples/peripherals/adc2)

读取内部霍尔效应传感器:
```c
#include <driver/adc.h>
...
    adc1_config_width(ADC_WIDTH_BIT_12);
    int val = hall_sensor_read();
```
在这两个示例中读取的值是 12 位宽(范围 0-4095).

## API Reference

该参考包括三个组成部分:

 - [ADC driver](https://esp-idf.readthedocs.io/en/latest/api-reference/peripherals/adc.html#adc-api-reference-adc-driver)
 - [ADC Calibration](https://esp-idf.readthedocs.io/en/latest/api-reference/peripherals/adc.html#adc-api-reference-adc-calibration)
 - [GPIO Lookup Macros](https://esp-idf.readthedocs.io/en/latest/api-reference/peripherals/adc.html#adc-api-reference-gpio-lookup-macros)

### Header File

 * [driver/include/driver/adc.h](https://github.com/espressif/esp-idf/blob/f9a4496/components/driver/include/driver/adc.h)
 * [esp_adc_cal/include/esp_adc_cal.h](esp_adc_cal/include/esp_adc_cal.h)
 * [soc/esp32/include/soc/adc_channel.h](https://github.com/espressif/esp-idf/blob/f9a4496/components/soc/esp32/include/soc/adc_channel.h)

## 参考资料

 - [ADC](https://docs.espressif.com/projects/esp-idf/en/v3.2/api-reference/peripherals/adc.html)
