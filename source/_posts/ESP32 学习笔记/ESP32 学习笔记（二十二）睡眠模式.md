---
title: ESP32 学习笔记（二十二）睡眠模式
date: 2019-03-12 00:28:34
categories:
- ESP32 学习笔记
tags:
- ESP32
---

# 睡眠模式

## 概述

ESP32 有轻度睡眠(light sleep)和深度睡眠(deep sleep)两种省电模式。

在轻度睡眠模式下，数字外设，大多数 RAM 和 CPU 都是时钟门控的，电源电压会降低。从轻度睡眠退出后，外围设备和 CPU 恢复运行，其内部状态将得以保留。

在深度睡眠模式下，CPU，大多数 RAM 以及从 APB_CLK 提供时钟的所有数字外设都将断电。芯片中仍然可以通电的唯一部分是：RTC 控制器，RTC 外设（包括 ULP 协处理器）和 RTC 存储器（慢速和快速）。

可以使用多种唤醒源从深度和轻度睡眠模式唤醒。可以组合这些唤醒源，在这种情况下，当触发任何一个源时芯片将被唤醒。可以使用 `esp_sleep_enable_X_wakeup` API 启用唤醒源，并可以使用 `esp_sleep_disable_wakeup_source()` API 禁用唤醒源。下一节将详细介绍这些 API。在进入浅色或深度睡眠模式之前，可以随时配置唤醒源。

此外，应用程序可以使用 `esp_sleep_pd_config()` API 强制 RTC 外设和 RTC 存储器的特定掉电模式。

配置唤醒源后，应用程序可以使用 `esp_light_sleep_start()` 或 `esp_deep_sleep_start()` API 进入睡眠模式。此时，将根据请求的唤醒源配置硬件，RTC 控制器将关闭 CPU 或数字外设的电源或关闭电源。

<!--more-->

## WiFi/BT 和睡眠模式

在深度睡眠和轻度睡眠模式下，无线外围设备断电。 在进入深度睡眠或轻度睡眠模式之前，应用程序必须调用适当的函数（`esp_bluedroid_disable()`，`esp_bt_controller_disable()`，`esp_wifi_stop()`）禁用 WiFi 和 BT。 即使不调用这些功能，也不会在深度睡眠或轻度睡眠中保持 WiFi 和 BT 连接。

如果需要维护 WiFi 连接，请启用 WiFi 调制解调器睡眠(modem sleep)，并启用自动轻度睡眠功能（请参阅[电源管理 API](https://blog.csdn.net/qq_27114397/article/details/88411347)）。这将允许系统在需要 WiFi 驱动程序时自动从睡眠中唤醒，从而保持与 AP 的连接。

## 唤醒源

### 定时器

RTC 控制器具有内置定时器，可用于在预定义的时间后唤醒芯片。时间以微秒精度指定，但实际分辨率取决于为 RTC SLOW_CLK 选择的时钟源。有关 RTC 时钟选项的详细信息，请参见“ESP32 技术参考手册”的“复位和时钟”一章。

此唤醒模式不需要在睡眠期间打开 RTC 外围设备或 RTC 存储器。

`esp_sleep_enable_timer_wakeup()` 函数可用于使用定时器启用深度睡眠唤醒。

### Touch pad

RTC IO 模块包含触摸传感器中断时触发唤醒的逻辑。您需要在芯片开始深度睡眠之前配置触摸传感器。

当 RTC 外设未被强制上电时（即 `ESP_PD_DOMAIN_RTC_PERIPH` 应设置为 `ESP_PD_OPTION_AUTO`），ESP32 的修订版 0 和 1 仅支持此唤醒模式。

`esp_sleep_enable_touchpad_wakeup()` 函数可用于启用此唤醒源。

### External 唤醒(ext0)

RTC IO 模块包含当其中 **一个 RTC GPIO** 的电平为预定义的逻辑电平时触发唤醒的逻辑。RTC IO 是 RTC 外设电源域的一部分，因此如果请求唤醒源，RTC 外设将在深度睡眠期间保持通电状态。

由于在此模式下启用了 RTC IO 模块，因此也可以使用内部上拉或下拉电阻。在调用 `esp_sleep_start()` 之前，需要使用 `rtc_gpio_pullup_en()` 和 `rtc_gpio_pulldown_en()` 函数由应用程序配置它们。

在 ESP32 的修订版 0 和 1 中，此唤醒源与 ULP 和触摸唤醒源不兼容。

`esp_sleep_enable_ext0_wakeup()` 函数可用于启用此唤醒源。

>从睡眠状态唤醒后，用于唤醒的 IO pad 将被配置为 RTC IO。在将此 pad 用作数字 GPIO 之前，请使用 `rtc_gpio_deinit(gpio_num)` 函数重新配置它。

### External 唤醒(ext1)

RTC 控制器包含使用 **多个 RTC GPIO** 触发唤醒的逻辑。两个逻辑功能之一可用于触发唤醒：

 - 如果任何所选引脚为高电平，则唤醒（`ESP_EXT1_WAKEUP_ANY_HIGH`）
 - 如果所有选定的引脚都为低电平，则唤醒（`ESP_EXT1_WAKEUP_ALL_LOW`）

该唤醒源由 RTC 控制器实现。因此，RTC 外设和 RTC 存储器可以在此模式下断电。但是，如果 RTC 外设断电，内部上拉和下拉电阻将被禁用。要使用内部上拉或下拉电阻，请在睡眠期间请求 RTC 外设电源域保持开启，并在进入睡眠模式之前使用 rtc_gpio_ 函数配置上拉/下拉电阻：

```
esp_sleep_pd_config(ESP_PD_DOMAIN_RTC_PERIPH, ESP_PD_OPTION_ON);
gpio_pullup_dis(gpio_num);
gpio_pulldown_en(gpio_num);
```

`esp_sleep_enable_ext1_wakeup()`  函数可用于启用此唤醒源。

>从睡眠状态唤醒后，用于唤醒的 IO pad 将被配置为 RTC IO。在将此 pad 用作数字 GPIO 之前，请使用 `rtc_gpio_deinit(gpio_num)` 函数重新配置它。

### ULP 协处理器唤醒

ULP 协处理器可以在芯片处于睡眠模式时运行，并且可以用于轮询传感器，监视 ADC 或触摸传感器值，并在检测到特定事件时唤醒芯片。ULP 协处理器是 RTC 外设电源域的一部分，它运行存储在 RTC 慢速存储器中的程序。如果请求此唤醒模式，RTC 慢速内存将在睡眠期间启动。在 ULP 协处理器开始运行程序之前，RTC 外设将自动上电; 程序停止运行后，RTC 外设将再次自动关闭。

当 RTC 外设未被强制上电时（即 `ESP_PD_DOMAIN_RTC_PERIPH` 应设置为 `ESP_PD_OPTION_AUTO`），ESP32 的修订版 0 和 1 仅支持此唤醒模式。

`esp_sleep_enable_ulp_wakeup()`  函数可用于启用此唤醒源。

### GPIO 唤醒(仅 light sleep)

除了上面描述的 EXT0 和 EXT1 唤醒源之外，在轻度睡眠模式下还有一种从外部输入唤醒的方法。通过该唤醒源，每个引脚可以单独使用 `gpio_wakeup_enable()` 函数配置为高电平或低电平唤醒。与 EXT0 和 EXT1 唤醒源（只能与 RTC IO 一起使用）不同，此唤醒源可用于任何 IO（RTC 或数字）。

`esp_sleep_enable_gpio_wakeup()` 函数可用于启用此唤醒源。

### UART 唤醒(仅 light sleep)

当 ESP32 从外部设备接收 UART 输入时，通常需要在输入数据可用时唤醒芯片。UART 外设包含一项功能，当看到 RX 引脚上的一定数量的上升沿时，可以将芯片从轻度睡眠状态唤醒。可以使用 `uart_set_wakeup_threshold()` 函数设置此上升沿数。请注意，唤醒后 UART 不会接收触发唤醒的字符（及其前面的任何字符）。 这意味着外部设备通常需要在发送数据之前向 ESP32 发送额外字符以触发唤醒。

`esp_sleep_enable_uart_wakeup()` 函数可用于启用此唤醒源。

## RTC外设和存储器掉电

默认情况下，`esp_deep_sleep_start()` 和 `esp_light_sleep_start()` 函数将关闭所有启用的唤醒源不再需要的 RTC 电源域。要覆盖此行为，请提供 `esp_sleep_pd_config()` 函数。

注意：在 ESP32 的版本 0 中，RTC 快速存储器将始终在深度睡眠中保持启用状态，以便深度睡眠存根可以在复位后运行。 如果应用程序在深度睡眠后不需要干净的重置行为，则可以覆盖此项。

如果程序中的某些变量放入RTC慢速存储器（例如，使用 `RTC_DATA_ATTR` 属性），RTC 慢速存储器将默认保持通电状态。 如果需要，可以使用 `esp_sleep_pd_config()` 函数覆盖它。

## 进入轻度睡眠

配置唤醒源后，可使用 `esp_light_sleep_start()` 函数进入轻度睡眠模式。在没有配置唤醒源的情况下也可以进入轻度睡眠状态，在这种情况下，芯片将无限期地处于轻度睡眠模式，直到外部复位。

## 进入深度睡眠

配置唤醒源后，可使用 `esp_deep_sleep_start()` 函数进入深度睡眠模式。在没有配置唤醒源的情况下也可以进入深度睡眠状态，在这种情况下，芯片将无限期地处于深度睡眠模式，直到外部复位。

## IO 配置

一些 ESP32 IO 具有内部上拉或下拉，默认情况下启用。如果外部电路在深度睡眠模式下驱动此引脚，则由于流过这些上拉和下拉的电流，电流消耗可能会增加。

要隔离引脚，防止额外的电流消耗，请调用 `rtc_gpio_isolate()` 函数。

例如，在 ESP32-WROVER 模块上，外部上拉 GPIO12。GPIO12 在 ESP32 芯片中也有内部下拉。这意味着在深度睡眠中，一些电流将流过这些外部和内部电阻，从而将深度睡眠电流增加到最小可能值以上。在 `esp_deep_sleep_start()` 之前添加以下代码以删除此额外电流：

```
rtc_gpio_isolate(GPIO_NUM_12);
```

## UART 输出处理

在进入睡眠模式之前，`esp_deep_sleep_start()` 将刷新 UART FIFO 的内容。

使用 `esp_light_sleep_start()` 进入轻度睡眠模式时，UART FIFO 不会被刷新。相反，UART 输出将被暂停，FIFO 中的剩余字符将在从轻度睡眠唤醒后发送出去。

## 检查睡眠唤醒原因

`esp_sleep_get_wakeup_cause()` 函数可用于检查哪个唤醒源触发了从睡眠模式中唤醒。

对于 TouchPad 和 ext1 唤醒源，可以使用 `esp_sleep_get_touchpad_wakeup_status()` 和 `esp_sleep_get_ext1_wakeup_status()` 函数识别引起唤醒的引脚或 TouchPad。

## 禁用睡眠唤醒源

先前配置的唤醒源可以在之后使用 `esp_sleep_disable_wakeup_source()` API 禁用。此功能将停用触发对给定唤醒源。此外，如果参数为 ESP_SLEEP_WAKEUP_ALL，它可以禁用所有触发器。

## 应用示例

深度睡眠的基本功能的实现在 protocols/sntp 示例中给出，其中 ESP 模块被周期性地唤醒以从 NTP 服务器检索时间。

system/deep_sleep 示例中更广泛的说明了各种深度睡眠唤醒触发器和 ULP 协处理器编程的使用。

## 参考资料

 - [Sleep Modes](https://docs.espressif.com/projects/esp-idf/en/v3.2/api-reference/system/sleep_modes.html)
