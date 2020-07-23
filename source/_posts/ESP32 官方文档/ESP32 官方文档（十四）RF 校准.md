---
title: ESP32 官方文档（十四）RF 校准
date: 2012-07-22 09:43:43
categories:
- ESP32 官方文档
tags:
- ESP32
---

# RF 校准

ESP32 在 RF 初始化期间支持三种 RF 校准方法：

 1. 部分校准
 2. 完全校准
 3. 没有校准

## 部分校准

在 RF 初始化期间，默认情况下使用部分校准方法进行 RF 校准。 它基于存储在 NVS 中的完整校准数据完成。 要使用此方法，请到 `menuconfig` 并启用 `CONFIG_ESP32_PHY_CALIBRATION_AND_DATA_STORAGE`。

## 完全校准

在以下条件下触发完全校准：

1. NVS  不存在。
2. 用于存储校准数据的  NVS  分区被擦除。
3. 硬件 MAC  地址已更改。
4. PHY 库版本已更改。
5. 从 NVS 分区加载的 RF 校准数据被破坏。

需要大约  100 ms, 比部分校准用的时间多。如果启动持续时间不重要，建议使用完整的校准方法。要切换到完整校准方法，请转到 `menuconfig` 并禁用 `CONFIG_ESP32_PHY_CALIBRATION_AND_DATA_STORAGE`。如果使用 RF 校准的默认方法，有两种方法可以添加触发完全校准功能作为最后的补救措施。

1. 如果您不介意删除存储在  NVS  分区中的所有数据，请擦除 NVS 分区。
2. 在基于某些条件（例如，在某些诊断模式中提供的选项）初始化 WiFi  和 BT/BLE 之前调用 `esp_phy_erase_cal_data_in_nvs（）`。在这种情况下，仅擦除 NVS 分区的 phy 命名空间。

## 没有校准

ESP32 从深度睡眠中醒来时，不会使用校准方法。

## PHY 初始化数据

PHY 初始化数据用于 RF 校准。 有两种方法可以获得 PHY 初始化数据。

1.  一个是默认的初始化数据，它位于头文件 `components/esp32/phy_init_data.h` 中。 它在编译后嵌入到应用程序二进制文件中，然后存储到只读存储器（DROM）中。 要使用默认初始化数据，请到 `menuconfig` 并禁用 `CONFIG_ESP32_PHY_INIT_DATA_IN_PARTITION`。
2. 另一种是存储在分区中的初始化数据。 使用自定义分区表时，请确保包含 PHY 数据分区（类型：数据，子类型：phy）。 使用默认分区表，这是自动完成的。 如果初始化数据存储在分区中，则必须在那里闪存，否则将发生运行时错误。 要切换到存储在分区中的初始化数据，请到 `menuconfig` 并启用 `CONFIG_ESP32_PHY_INIT_DATA_IN_PARTITION`。

## 参考资料

 - [原文链接](https://docs.espressif.com/projects/esp-idf/en/latest/api-guides/RF_calibration.html)
