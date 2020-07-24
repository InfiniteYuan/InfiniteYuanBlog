---
title: ESP32 学习笔记（二十一）电源管理
date: 2019-03-11 22:20:34
categories:
- ESP32 学习笔记
tags:
- ESP32
---

# 电源管理

## 概述

ESP-IDF 中包含的电源管理算法可以根据应用组件的要求调整 APB 频率，CPU 频率，并使芯片进入 light sleep 模式，以尽可能低的功耗运行应用程序。

应用程序组件可以通过创建和获取电源管理锁来表达其要求。

例如，由 APB 提供时钟的外围设备的驱动器可以在使用外围设备的时间内请求将 APB 频率设置为80MHz。另一个例子是，当有任务准备好运行时，RTOS 将请求 CPU 以最高配置频率运行。又一个例子是需要启用中断的外围驱动器。这样的驱动程序可以请求禁用 light sleep。

当然，要求更高的 APB 或 CPU 频率或禁用 light sleep 会导致更高的电流消耗。组件应尽可能在最短的时间内限制电源管理锁的使用。

<!--more-->

## 配置

可以使用 `CONFIG_PM_ENABLE` 选项在编译时启用电源管理。

启用电源管理功能的代价是增加了中断延迟。额外延迟取决于许多因素，其中包括 CPU 频率，单/双核模式，是否需要执行频率切换。最小额外延迟为 0.2us（当 CPU 频率为 240MHz，并且未启用频率调整时），最大额外延迟为 40us（启用频率调整时，在中断输入时执行从 40MHz 到 80MHz 的切换）。

通过调用 `esp_pm_configure()` 函数，可以在应用程序中启用动态频率调整（DFS）和自动光睡眠。它的参数是定义频率缩放设置的结构，`esp_pm_config_esp32_t`。在此结构中，需要初始化 3 个字段：

 - `max_freq_mhz`  - 最大 CPU 频率，以 MHZ 为单位（即采用 `ESP_PM_CPU_FREQ_MAX` 时使用的频率）。 这通常设置为 `CONFIG_ESP32_DEFAULT_CPU_FREQ_MHZ`。
 - `min_freq_mhz`  - 最小 CPU 频率，以 MHz 为单位（即仅采用 `ESP_PM_APB_FREQ_MAX` 锁定时使用的频率）。 这可以设置为 XTAL 频率，或 XTAL 频率除以整数。 请注意，10MHz 是可以生成 1MHz 的默认 REF_TICK 时钟的最低频率。
 - `light_sleep_enable`  - 当没有锁定时，系统是否应自动进入 light sleep 状态（true/false）。

> 自动 light sleep 模式基于 FreeRTOS Tickless Idle 功能。如果在 menuconfig 中未启用 `CONFIG_FREERTOS_USE_TICKLESS_IDLE`选项，则 `esp_pm_configure()` 将返回 `ESP_ERR_NOT_SUPPORTED` 错误，但请求自动 light sleep。

> 在 light sleep 模式下，外设是时钟门控的，不会产生中断（来自 GPIO 和内部外设）。睡眠模式文档中描述的唤醒源可用于从 light sleep 状态唤醒。例如，EXT0 和 EXT1 唤醒源可用于从 GPIO 唤醒。

或者，可以在 menuconfig 中启用 `CONFIG_PM_DFS_INIT_AUTO` 选项。如果启用，最大 CPU 频率由 `CONFIG_ESP32_DEFAULT_CPU_FREQ_MHZ` 设置决定，最小 CPU 频率设置为 XTAL 频率。

## 电源管理锁

如概述中所述，应用程序可以获取/释放锁以控制电源管理算法。当应用程序锁定时，对于每个锁定，电源管理算法操作以下面描述的方式受到限制。释放锁定时，将删除此类限制。

应用程序的不同部分可以采用相同的锁定。 在这种情况下，锁定必须被释放的次数与获取的次数相同，以便恢复功率管理算法。

在 ESP32 中，支持三种类型的锁：

 - `ESP_PM_CPU_FREQ_MAX`
请求 CPU 频率为 `esp_pm_configure()` 设置的最大值。对于 ESP32，此值可以设置为 80,160 或 240MHz。
 - `ESP_PM_APB_FREQ_MAX`
请求 APB 频率处于最大支持值。对于 ESP32，这是 80MHz。
 - `ESP_PM_NO_LIGHT_SLEEP`
防止使用自动 light sleep 模式。

## ESP32 的电源管理算法

启用动态频率调整后，CPU 频率将按如下方式切换：

 - 如果最大 CPU 频率（使用 `esp_pm_configure()` 或 `CONFIG_ESP32_DEFAULT_CPU_FREQ_MHZ` 设置）为 240 MHz：

	1. 获取 `ESP_PM_CPU_FREQ_MAX` 或 `ESP_PM_APB_FREQ_MAX` 时，CPU 频率为 240 MHz，APB 频率为 80 MHz。
	2. 否则，频率将切换到使用 `esp_pm_configure()`设置的最小值。

 - 如果最大 CPU 频率为 160 MHz：

	1. 获取 `ESP_PM_CPU_FREQ_MAX` 时，CPU 频率设置为 160 MHz，APB 频率设置为 80 MHz。
	2. 当未获取 `ESP_PM_CPU_FREQ_MAX` 但 `ESP_PM_APB_FREQ_MAX` 为时，CPU 和 APB 频率设置为 80 MHz。
	3. 否则，频率将切换到使用 `esp_pm_configure()` 设置的最小值。

 - 如果最大 CPU 频率为 80 MHz：

	1. 获取 `ESP_PM_CPU_FREQ_MAX` 或 `ESP_PM_APB_FREQ_MAX` 锁时，CPU 和 APB 频率将为 80 MHz。
	2. 否则，频率将切换到使用 `esp_pm_configure()` 设置的最小值。

 - 如果未获取任何锁定，并且在调用 `esp_pm_configure()` 时启用了轻度睡眠，则系统将进入轻度睡眠模式(light sleep)。轻度睡眠的持续时间将由以下因素确定：

	 - FreeRTOS 任务因有限超时而被阻止
	- 定时器使用高分辨率计时器 API 注册

轻度睡眠持续时间将被选择为在最近的事件之前唤醒（任务被解锁或计时器过去）。

## 动态频率调整和外设驱动程序

启用 DFS 后，APB 频率可在单个 RTOS 滴答内多次更改。即使 APB 频率发生变化，一些外围设备也能正常工作; 有些不能。

即使 APB 频率发生变化，以下外设也能正常工作：

 - UART：如果 REF_TICK 用作时钟源（请参阅 uart_config_t 的 use_ref_tick 成员）。
 - LEDC：如果 REF_TICK 用作时钟源（参见 `ledc_timer_config()` 函数）。
 - RMT：如果 REF_TICK 用作时钟源。 目前，驱动程序不支持 REF_TICK，但可以通过清除相应通道的 `RMT_REF_ALWAYS_ON_CHx` 位来启用它。

目前，以下外围驱动程序知道 DFS，并将在事务持续时间内使用 `ESP_PM_APB_FREQ_MAX` 锁定：

 - SPI master
 - SDMMC

启用驱动程序时，以下驱动程序将保持 `ESP_PM_APB_FREQ_MAX` 锁定：

 - SPI slave  - 在调用 `spi_slave_initialize()` 和 `spi_slave_free()` 之间。
 - 以太网 - 在调用 `esp_eth_enable()` 和 `esp_eth_disable()` 之间。
 - WiFi  - 在调用 `esp_wifi_start()` 和 `esp_wifi_stop()` 之间。 如果启用了调制解调器睡眠(modem sleep)，则在禁用无线电时的一段时间内将释放锁定。
 - 蓝牙 - 在调用 `esp_bt_controller_enable()` 和 `esp_bt_controller_disable()` 之间。
 - CAN  - 调用 `can_driver_install()` 和 `can_driver_uninstall()` 之间

以下外围驱动程序尚未处理 DFS。应用程序需要在必要时获取/释放锁定：

 - I2C
 - I2S
 - MCPWM
 - PCNT
 - Sigma-delta
 - Timer group

## 参考资料

 - [Power Management](https://docs.espressif.com/projects/esp-idf/en/v3.2/api-reference/system/power_management.html)
