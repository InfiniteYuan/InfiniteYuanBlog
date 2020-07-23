---
title: ESP32 官方文档（十二）Bootloader (引导加载程序)
date: 2012-07-22 09:43:43
categories:
- ESP32 官方文档
tags:
- ESP32
---

# Bootloader

引导加载程序执行以下功能：

1. 内部模块的最初初始配置;
2. 根据分区表和 ota_data(如果有),选择要引导的应用程序分区;
3. 将此映像加载到  RAM(IRAM 和 DRAM) 并将管理传输到它.

引导加载程序位于 Flash 中的地址 0x1000.

<!--more-->

## 恢复出厂设置

用户可以编写基本工作固件并将其加载到 factory 分区.接下来,通过 OTA(无线)更新固件.更新的固件将加载到 OTA 应用程序分区位置,并更新 OTA 数据分区以从此分区引导.如果您希望能够回滚到出厂固件并清除设置,则需要设置 `CONFIG_BOOTLOADER_FACTORY_RESET`.恢复出厂设置机制允许将设备重置为 factory 设置：

 - 清除一个或多个数据分区.
 - 从 “factory” 分区启动.

`CONFIG_BOOTLOADER_DATA_FACTORY_RESET` 允许客户选择在执行恢复出厂设置时将擦除哪些数据分区.可以通过逗号分隔的可选空格指定分区的名称以便于阅读. (像这样：“nvs,phy_init,nvs_custom,......”).确保分区表中指定的名称和此处的名称相同.此处无法指定 “app” 类型的分区.

`CONFIG_BOOTLOADER_OTA_DATA_ERASE` - 设备将在恢复出厂设置后从 “factory” 分区启动. OTA 数据分区将被清除.

`CONFIG_BOOTLOADER_NUM_PIN_FACTORY_RESET` - 用于恢复出厂设置的 GPIO 输入数用于触发恢复出厂设置,此 GPIO 必须在复位时拉低以触发此操作.

`CONFIG_BOOTLOADER_HOLD_TIME_GPIO` - 这是 GPIO 保持复位/测试模式的时间(默认为 5 秒).复位后,GPIO 必须在此段时间内保持低电平,然后才能执行恢复出厂设置或测试分区引导(如果适用).

分区表：

```
# Name,   Type, SubType, Offset,   Size, Flags
# Note: if you change the phy_init or app partition offset, make sure to change the offset in Kconfig.projbuild
nvs,      data, nvs,     0x9000,   0x4000
otadata,  data, ota,     0xd000,   0x2000
phy_init, data, phy,     0xf000,   0x1000
factory,  0,    0,       0x10000,  1M
test,     0,    test,    ,         512K
ota_0,    0,    ota_0,   ,         512K
ota_1,    0,    ota_1,   ,         512K
```

## 从 TEST 固件启动

用户可以编写一个特殊的固件用于生产中的测试,并根据需要运行它. 分区表还需要一个专用分区用于此测试固件(请参阅分区表). 要触发测试应用,您需要设置 `CONFIG_BOOTLOADER_APP_TEST`.

`CONFIG_BOOTLOADER_NUM_PIN_APP_TEST` - 引导TEST 分区的 GPIO 输入的编号. 选定的 GPIO 将配置为启用内部上拉的输入. 要触发测试应用程序,必须在复位时将此 GPIO 拉低. 停用 GPIO 输入并重启设备后,旧应用程序将启动(工厂或任何 OTA 位置的应用程序).

`CONFIG_BOOTLOADER_HOLD_TIME_GPIO` - 这是 GPIO 的复位/测试模式保持时间(默认为 5 秒). 复位后,GPIO 必须在此段时间内保持低电平,然后才能执行恢复出厂设置或测试分区引导(如果适用).

## 自定义引导加载程序

当前的引导加载程序实现允许客户覆盖它. 为此,您必须复制文件夹 `/esp-idf/components/bootloader`,然后编辑 `/your_project/components/bootloader/subproject/main/bootloader_main.c`. 在引导加载程序空间中,您无法使用其他组件的驱动程序和函数. 如有必要,则应将所需功能放在文件夹引导程序中(请注意,这会增加其大小). 有必要监视其大小,因为内存中可能存在覆盖层,导致损坏. 目前,引导加载程序仅限于地址 0x8000 的分区表.

## 参考资料

 - [原文链接](https://docs.espressif.com/projects/esp-idf/en/latest/api-guides/bootloader.html)
