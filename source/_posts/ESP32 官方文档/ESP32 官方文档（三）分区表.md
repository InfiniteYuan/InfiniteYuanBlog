---
title: ESP32 官方文档（三）分区表
date: 2018-08-29 01:42:40
categories:
- ESP32 官方文档
tags:
- ESP32
---

# 分区表

## 概述

单个 ESP32 的 flash 可以包含多个应用程序,以及许多不同类型的数据(校准数据,文件系统,参数存储等). 因此,分区表被下载到 flash 中的 0x8000 地址(默认偏移量).

分区表长度为 0xC00 字节(最多 95 个分区表条目). 在表数据之后附加 MD5 校验和. 如果分区表由于安全引导而签名,则签名将附加在分区表之后.

分区表中的每个条目都有一个 `name` (label),`type`(app,data 或其他),`subtype`以及加载分区的 flash 中的 `offset` (偏移量).

使用分区表的最简单方法是 `make menuconfig` 并选择一个简单的预定义分区表:

 - “Single factory app, no OTA”
 - “Factory app, two OTA definitions”

在这两种情况下,`factory` 应用程序下载到 0x10000 地址. 如果您 `make partition_table`,那么它将打印分区表的摘要.

<!--more-->

## 内置分区表

以下是 `Single factory app, no OTA` 的分区表配置信息:
```
# Espressif ESP32 Partition Table
# Name,   Type, SubType, Offset,  Size, Flags
nvs,      data, nvs,     0x9000,  0x6000,
phy_init, data, phy,     0xf000,  0x1000,
factory,  app,  factory, 0x10000, 1M,
```
 - flash 中的 0x10000(64KB) 偏移量被标记为 `factory` 应用程序. 默认情况下,引导加载程序将运行此应用程序.
 - 在分区表中还定义了两个用于存储 NVS 库分区和 PHY 初始化数据的数据区域.

以下是 `Factory app, two OTA definitions` 的分区表配置信息:
```
# Espressif ESP32 Partition Table
# Name,   Type, SubType, Offset,  Size, Flags
nvs,      data, nvs,     0x9000,  0x4000,
otadata,  data, ota,     0xd000,  0x2000,
phy_init, data, phy,     0xf000,  0x1000,
factory,  0,    0,       0x10000, 1M,
ota_0,    0,    ota_0,  0x110000, 1M,
ota_1,    0,    ota_1,  0x210000, 1M,
```
 - 现在有三个应用程序分区定义.
 - 这三种 `Type` 都设置为 `app`,但在 0x10000 位置的 `factory` 应用程序和后面的两个 `OTA` 应用程序的子类型有所不同.
 - 还有一个新的 `ota data` 区域,用于保存 OTA 更新的数据. 引导加载程序会查询此数据,以便了解要执行的应用程序. 如果 `ota data` 为空,它将执行 `factory` 应用程序.

## 创建自定义分区表

如果在 menuconfig 中选择 “Custom partition table CSV”,还应该输入要用于分区表的 CSV 文件的名称(在项目目录中). CSV 文件中有将要用于分区表配置的任意数量的定义.

CSV 格式与上面摘要中打印的格式相同. 但是,CSV 中并非所有字段都是必需的. 例如,以下是 OTA 分区表的“输入” CSV:
```
# Custom partition table
# Name,   Type, SubType, Offset, Size, Flags
nvs,      data, nvs,     ,       0x4000,
otadata,  data, ota,     ,       0x2000,
phy_init, data, phy,     ,       0x1000,
factory,  app,  factory, ,       1M,
ota_0,    app,  ota_0,   ,       1M,
ota_1,    app,  ota_1,   ,       1M,
```
 - 字段之间的空格被忽略,任何以＃(注释)开头的行也是如此.
 - CSV 文件中的每个非注释行都是分区定义.
 - 每个分区的 “Offset” 字段为空. `gen_esp32part.py` 工具填充每个空白偏移量,从分区表开始并确保每个分区正确对齐.

### 名字字段

名称字段可以是任何有意义的名称. 这对 ESP32 来说并不重要. 超过 16 个字符的名称将被截取.

### 类型字段

分区类型字段可以指定为 app(0) 或 data(1). 或者它可以是数字 0-254(或十六进制 0x00-0xFE). 类型 0x00-0x3F 保留用于 esp-idf 核心功能.

如果您的应用程序需要存储数据,请在 0x40-0xFE 范围内添加自定义分区类型.

引导加载程序忽略 app(0) 和 data(1) 以外的任何分区类型.

### 子类型

8 位子类型字段特定于给定的分区类型.

esp-idf 当前仅指定 “app” 和 “data” 分区类型的子类型字段的含义.

### App 子类型

当 type 为 “app” 时,子类型字段可以指定为 factory(0),ota_0(0x10)... ota_15(0x1F) 或 test(0x20).

- factory(0) 是默认的应用程序分区. 引导加载程序将执行工厂应用程序,除非它看到类型为 data/ota 的分区,在这种情况下,它会读取此分区以确定要引导的 OTA 映像.
	- OTA 永远不会更新 `factory` 分区.
	- 如果要保留 OTA 项目中的闪存使用率,可以删除 `factory` 分区并改为使用 ota_0.
- ota_0(0x10)... ota_15(0x1F) 是 OTA app 区域. 有关更多详细信息,请参阅[OTA文档](https://esp-idf.readthedocs.io/en/latest/api-reference/system/ota.html),然后使用 OTA 数据分区配置引导加载程序应引导的应用程序. 如果使用 OTA,则应用程序应至少具有两个 OTA 应用程序槽(`ota_0`＆`ota_1`). 有关更多详细信息,请参阅[OTA文档](https://esp-idf.readthedocs.io/en/latest/api-reference/system/ota.html).
- test(0x2) 是 `factory` 测试程序的保留子类型. esp-idf 引导程序当前不支持它.

### 数据子类型

当 type 为 “data” 时,子类型字段可以指定为 ota(0),phy(1),nvs(2).

 - ota(0) 是[OTA数据分区](https://esp-idf.readthedocs.io/en/latest/api-reference/system/ota.html#ota-data-partition),它存储有关当前所选 OTA 应用程序的信息.此分区的大小应为 0x2000 字节.有关更多详细信息,请参阅[OTA文档](https://esp-idf.readthedocs.io/en/latest/api-reference/system/ota.html#ota-data-partition).
 - phy(1) 用于存储 PHY 初始化数据.这允许 PHY 到每个设备被配置,而不是在固件.
	- 在默认配置中,不使用 phy 分区,并且 PHY 初始化数据被编译到 app 本身.因此,可以从分区表中删除此分区以节省空间.
	- 要从此分区加载 PHY 数据,请运行 `make menuconfig` 并启用 `CONFIG_ESP32_PHY_INIT_DATA_IN_PARTITION` 选项.您还需要使用 `phy init` 数据刷新(flash)设备,因为 esp-idf 构建系统不会自动执行此操作.
 - nvs(2) 用于[非易失性存储(NVS)API](https://esp-idf.readthedocs.io/en/latest/api-reference/storage/nvs_flash.html).
	- NVS 用于存储每个设备 PHY 校准数据(与初始化数据不同).
	- 如果使用[esp_wifi_set_storage(WIFI_STORAGE_FLASH)](https://esp-idf.readthedocs.io/en/latest/api-reference/wifi/esp_wifi.html)初始化功能,则 NVS 用于存储 WiFi 数据.
	- NVS API 还可用于其他应用程序数据.
	- 强烈建议您在项目中包含至少 0x3000 字节的 NVS 分区.
	- 如果使用 NVS API 存储大量数据,请将 NVS 分区默认的 0x6000 字节大小增加.

其他数据子类型保留用于将来的 esp-idf 用途.

### 偏移量 & 大小

具有空白偏移的分区将在前一个分区之后开始,或者第一个分区是在分区表之后开始.

应用程序分区必须处于与 0x10000(64K) 对齐的偏移量. 如果将偏移字段留空,工具将自动对齐分区. 如果为应用程序分区指定未对齐的偏移量,该工具将返回错误.

大小和偏移量可以指定为十进制数,带前缀 0x 的十六进制数,或大小乘数 K 或 M(1024 和 1024 * 1024 字节).

如果希望分区表中的分区与表本身的任何起始偏移量(`CONFIG_PARTITION_TABLE_OFFSET`)一起使用,请将所有分区的偏移字段(在 CSV 文件中)留空. 类似地,如果更改分区表偏移,则要注意所有空白分区偏移可能会更改为匹配,并且任何固定偏移现在可能与分区表冲突(导致错误).

### 标志

目前仅支持一个加密的标志. 如果此字段设置为加密,则在启用 Flash 加密时将对此分区进行加密.

(请注意,无论是否设置此标志,应用程序类型分区都将始终加密.)

## 生成二进制分区表

下载到 ESP32 的分区表是二进制格式,而不是 CSV 格式. 工具 `partition_table/gen_esp32part.py` 用于在 CSV 和二进制格式之间进行转换.

如果在 `make menuconfig` 中配置分区表 CSV 名称,然后生成 `partition_table`,则此转换将在构建过程中完成.

要手动将 CSV 转换为二进制:
```
python gen_esp32part.py input_partitions.csv binary_partitions.bin
```
要将二进制格式转换回 CSV:
```
python gen_esp32part.py binary_partitions.bin input_partitions.csv
```
在 stdout 上显示二进制分区表的内容(这是生成 `make partition_table` 时显示的摘要的方式:
```
python gen_esp32part.py binary_partitions.bin
```

### MD5 校验和

分区表的二进制格式包含基于分区表计算的 MD5 校验和. 此校验和用于在引导期间检查分区表的完整性.

可以通过 `gen_esp32part.py` 的 `--disable-md5sum` 选项或 `CONFIG_PARTITION_TABLE_MD5` 选项禁用 MD5 校验和生成. 例如,当使用传统引导加载程序无法处理 MD5 校验和且引导失败并且错误消息无效幻数 0xebeb 时,这很有用.

## 烧录分区表

 - `make partition_table-flash`:将使用 `esptool.py 下载分区表.
 - `make flash`:会下载所有固件,包括分区表.

手动下载命令也会打印在 `make partition_table` 中.

请注意,更新分区表不会擦除已根据旧分区表存储的数据. 您可以使用 `make erase_flash` (或 `esptool.py erase_flash`)来擦除整个闪存内容.

## 参考资料

 - [原文链接](https://docs.espressif.com/projects/esp-idf/en/latest/api-guides/partition-tables.html)
