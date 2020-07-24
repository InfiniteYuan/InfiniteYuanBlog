---
title: ESP32 学习笔记（十六）Over The Air Updates (OTA)
date: 2018-08-29 00:59:26
categories:
- ESP32 学习笔记
tags:
- ESP32
---

# Over The Air Updates (OTA)

## OTA 流程概述

OTA 更新机制允许设备根据正常固件运行时收到的数据进行更新(例如,通过WiFi或蓝牙).

OTA 要求至少使用两个 “OTA app slot” 分区(即 `ota_0` 和 `ota_1`)和 “OTA Data Partition” 来配置设备的[分区表](https://esp-idf.readthedocs.io/en/latest/api-guides/partition-tables.html).

OTA 操作功能将新的应用程序固件映像写入当前未用于启动的 OTA 应用程序位置. 验证映像后,将更新 OTA 数据分区以指定此映像应用于下次启动.

<!--more-->

## OTA 数据分区

OTA 数据分区 (type `data`, subtype `ota`) 必须包含在使用 OTA 功能的任何项目的[分区表](https://esp-idf.readthedocs.io/en/latest/api-guides/partition-tables.html)中.

对于出厂启动设置,OTA 数据分区不应包含任何数据(所有字节都擦除为 0xFF). 在这种情况下,esp-idf 软件引导加载程序将启动 factory 应用程序(如果它存在于分区表中). 如果分区表中不包含 factory 应用程序,则会启动第一个可用的 OTA 程序(通常为 `ota_0`).

在第一次 OTA 更新之后,更新 OTA 数据分区以指定下一个应该引导哪个 OTA 应用程序分区.

OTA 数据分区的大小为两个闪存扇区(0x2000 字节),以防止在写入时出现电源故障时出现问题. 扇区被独立擦除并用匹配数据写入,如果它们不同意,则使用计数器字段来确定最近写入哪个扇区.

## APP 回滚

应用程序回滚的主要目的是在更新后使设备保持工作。此功能允许您回滚到以前的工作应用程序，以防新应用程序出现严重错误。启用回滚过程并且 OTA 更新提供应用程序的新版本时，可能会发生以下三种情况之一：

- 应用程序正常，`esp_ota_mark_app_valid_cancel_rollback()` 使用状态 `ESP_OTA_IMG_VALID` 标记正在运行的应用程序。启动此应用程序没有任何限制。
- 应用程序存在严重错误，无法进行进一步的工作，需要回滚到先前的应用程序，`esp_ota_mark_app_invalid_rollback_and_reboot()` 将正在运行的应用程序标记为 `ESP_OTA_IMG_INVALID` 并重置。引导加载程序不会选择此应用程序进行引导，并将引导以前工作的应用程序。
- 如果设置了 `CONFIG_APP_ROLLBACK_ENABLE` 选项，并且在不调用任何函数的情况下发生重置，则会发生并回滚。

>注意：状态不会写入应用程序的二进制映像，而是将其写入数据分区。该分区包含一个 `ota_seq` 计数器，该计数器是指向应用程序将从中进行引导的插槽（ota_0，ota_1，...）的指针。

## APP OTA 状态

状态控制选择启动应用程序的过程：

|States|Restriction of selecting a boot app in bootloader|
|:-------:|:-------:|
|ESP_OTA_IMG_VALID |	None restriction. Will be selected.|
|ESP_OTA_IMG_UNDEFINED |	None restriction. Will be selected.|
|ESP_OTA_IMG_INVALID |	Will not be selected.|
|ESP_OTA_IMG_ABORTED |	Will not be selected.|
|ESP_OTA_IMG_NEW |	If CONFIG_APP_ROLLBACK_ENABLE option is set it will be selected only once. In bootloader the state immediately changes to `ESP_OTA_IMG_PENDING_VERIFY`.
|ESP_OTA_IMG_PENDING_VERIFY |	If CONFIG_APP_ROLLBACK_ENABLE option is set it will not be selected and the state will change to `ESP_OTA_IMG_ABORTED`.|

如果未启用 `CONFIG_APP_ROLLBACK_ENABLE` 选项（默认情况下），则使用以下函数 `esp_ota_mark_app_valid_cancel_rollback()` 和 `esp_ota_mark_app_invalid_rollback_and_reboot()` 是可选的，并且不使用 `ESP_OTA_IMG_NEW` 和 `ESP_OTA_IMG_PENDING_VERIFY` 状态。

Kconfig 中的 `CONFIG_APP_ROLLBACK_ENABLE` 选项允许您跟踪新应用程序的首次引导。在这种情况下，应用程序必须通过调用 `esp_ota_mark_app_valid_cancel_rollback()` 函数来确认其可操作性，否则应用程序将在重新启动时回滚。它允许您在引导阶段控制应用程序的可操作性。因此，新应用程序只有一次尝试成功启动。

## 回滚过程

启用 `CONFIG_APP_ROLLBACK_ENABLE` 选项时回滚过程的说明：

- 成功下载的新应用程序和 `esp_ota_set_boot_partition()` 函数使此分区可引导并设置状态 `ESP_OTA_IMG_NEW`。此状态表示应用程序是新的，应监视其首次引导。
- 重启 `esp_restart()`。
- 引导加载程序检查 `ESP_OTA_IMG_PENDING_VERIFY` 状态（如果已设置），然后将写入 `ESP_OTA_IMG_ABORTED`。
- 引导加载程序选择要引导的新应用程序，以便状态不会设置为 `ESP_OTA_IMG_INVALID` 或 `ESP_OTA_IMG_ABORTED`。
- 引导加载程序检查所选应用程序的 `ESP_OTA_IMG_NEW` 状态（如果已设置），然后它将写入 `ESP_OTA_IMG_PENDING_VERIFY`。此状态意味着应用程序需要确认其可操作性，如果没有发生并且发生重新启动，此状态将被覆盖到 `ESP_OTA_IMG_ABORTED` （参见上文），此应用程序将无法再启动，即将返回到以前的工作申请。
- 一个新的应用程序已经开始，应该进行自我测试。
- 如果自检已成功完成，则必须调用函数 `esp_ota_mark_app_valid_cancel_rollback()`，因为应用程序正在等待可操作性的确认（`ESP_OTA_IMG_PENDING_VERIFY` 状态）。
- 如果自检失败，则调用 `esp_ota_mark_app_invalid_rollback_and_reboot()` 函数回滚到上一个工作应用程序，同时将无效应用程序设置为 `ESP_OTA_IMG_INVALID` 状态。
- 如果尚未确认应用程序，则状态仍为 `ESP_OTA_IMG_PENDING_VERIFY`，下次启动时将更改为 `ESP_OTA_IMG_ABORTED`。这将阻止重新启动此应用程序。将回滚到以前的工作应用程序。

## 意外重启

如果在首次启动新应用程序时发生断电或意外崩溃，它将回滚应用程序。

建议：尽快执行自检程序，以防止因断电而导致的回滚。

只能回滚 `OTA` 分区。工厂分区未回滚。

## 启动无效/中止的应用程序

可以引导先前设置为 `ESP_OTA_IMG_INVALID` 或 `ESP_OTA_IMG_ABORTED` 的应用程序：

- 获取最后一个无效的应用程序分区 `esp_ota_get_last_invalid_partition()`。
- 将收到的分区传递给 `esp_ota_set_boot_partition()`，这将更新 `otadata`。
- 重启 `esp_restart()`。引导加载程序将引导指定的应用程序。

要确定在启动应用程序期间是否应运行自检，请调用 `esp_ota_get_state_partition()` 函数。如果结果为 `ESP_OTA_IMG_PENDING_VERIFY`，则需要进行自检并随后确认可操作性。

## 设置状态

关于状态设置的简要说明：

- `ESP_OTA_IMG_VALID` 状态由 `esp_ota_mark_app_valid_cancel_rollback()` 函数设置。
- 如果未启用 `CONFIG_APP_ROLLBACK_ENABLE` 选项，则 `esp_ota_set_boot_partition()` 函数将设置 `ESP_OTA_IMG_UNDEFINED` 状态。
- 如果启用了 `CONFIG_APP_ROLLBACK_ENABLE` 选项，则 `esp_ota_set_boot_partition()` 函数将设置 `ESP_OTA_IMG_NEW` 状态。
- `ESP_OTA_IMG_INVALID` 状态由 `esp_ota_mark_app_invalid_rollback_and_reboot()` 函数设置。
- 如果没有确认应用程序可操作性并且重新启动（如果启用了 `CONFIG_APP_ROLLBACK_ENABLE` 选项），则设置 `ESP_OTA_IMG_ABORTED` 状态。
- 如果启用了 `CONFIG_APP_ROLLBACK_ENABLE` 选项并且所选应用具有 `ESP_OTA_IMG_NEW` 状态，则在引导加载程序中设置 `ESP_OTA_IMG_PENDING_VERIFY` 状态。

## 没有安全启动的安全 OTA 更新

即使不启用硬件安全引导，也可以执行已签名的 OTA 更新的验证。为此，请参阅[没有硬件安全启动的签名应用验证](https://docs.espressif.com/projects/esp-idf/en/latest/security/secure-boot.html#signed-app-verify)

## 相关资料

 - [Partition Table documentation](https://esp-idf.readthedocs.io/en/latest/api-guides/partition-tables.html)
 - [Lower-Level SPI Flash/Partition API](https://esp-idf.readthedocs.io/en/latest/api-reference/storage/spi_flash.html)
 - [ESP HTTPS OTA](https://esp-idf.readthedocs.io/en/latest/api-reference/system/esp_https_ota.html)

## 应用示例

OTA 固件更新工作流程的端到端示例: [system/ota](https://github.com/espressif/esp-idf/tree/51a4b4b/examples/system/ota).

## API Reference

### Header File

 - [app_update/include/esp_ota_ops.h](https://github.com/espressif/esp-idf/blob/51a4b4b/components/app_update/include/esp_ota_ops.h)

## 参考资料

 - [Over The Air Updates (OTA)](https://docs.espressif.com/projects/esp-idf/en/v3.2/api-reference/system/ota.html)
