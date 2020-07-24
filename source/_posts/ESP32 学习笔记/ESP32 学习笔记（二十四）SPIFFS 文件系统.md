---
title: ESP32 学习笔记（二十四）SPIFFS 文件系统
date: 2019-03-23 20:21:27
categories:
- ESP32 学习笔记
tags:
- ESP32
---

# SPIFFS 文件系统

## 概述

SPIFFS 是一个文件系统，用于嵌入式目标上的 SPI NOR 闪存设备。它支持磨损均衡，文件系统一致性检查等。

## 注意

 - 目前，spiffs 不支持目录。它产生扁平结构。如果 SPIFFS 安装在 /spiffs 下创建一个带有路径 /spiffs/tmp/myfile.txt 的文件，则会在 SPIFFS 中创建一个名为 /tmp/myfile.txt 的文件，而不是在目录 /spiffs/tmp 下创建一个名为的 myfile.txt。
 - 它不是实时堆栈。一次写入操作可能持续时间比另一次更长。
 - 目前，它没法检测或处理块被损坏的情况。

<!--more-->

## 工具

主机端工具用于创建 SPIFS 分区镜像文件(.bin)，其中一个工具是 [mkspiffs](https://github.com/igrr/mkspiffs)。您可以使用它从给定文件夹创建镜像文件，然后使用 `esptool.py` 下载该镜像文件到设备中

为此，您需要获取一些参数：

 - 块大小：4096（SPI Flash的标准）
 - 页面大小：256（SPI Flash标准）
 - 图像大小：分区大小（以字节为单位）（可以从分区表中获取）
 - 分区偏移：分区的起始地址（可以从分区表中获取）

例如：
将目标文件夹打包为 1 兆字节镜像文件：

```
mkspiffs -c [src_folder] -b 4096 -p 256 -s 0x100000 spiffs.bin
```

要在偏移量 0x110000 处将镜像文件烧录到 ESP32：

```
python esptool.py --chip esp32 --port [port] --baud [baud] write_flash -z 0x110000 spiffs.bin
```

## 解决问题

1. 无法从 SPIFFS 中读取到文件内容

这种情况是由于通过工具制作的文件系统格式与程序中的文件系统格式不同导致，可通过运行 `./mkspiffs --version` 在你的 mkspiffs 目录和 `grep SPIFFS sdkconfig` 在你的工程目录，之后对比两者配置的不同之处。

例如：

运行 `./mkspiffs --version` 在 mkspiffs 目录中：

```c
mkspiffs ver. 0.2.3-6-g983970e
Build configuration name: generic
SPIFFS ver. 0.3.7-5-gf5e26c4
Extra build flags: (none)
SPIFFS configuration:
  SPIFFS_OBJ_NAME_LEN: 32
  SPIFFS_OBJ_META_LEN: 0
  SPIFFS_USE_MAGIC: 1
  SPIFFS_USE_MAGIC_LENGTH: 1
  SPIFFS_ALIGNED_OBJECT_INDEX_TABLES: 0
```

运行 `grep SPIFFS sdkconfig` 在工程目录中：

```c
# SPIFFS Configuration
CONFIG_SPIFFS_MAX_PARTITIONS=3
# SPIFFS Cache Configuration
CONFIG_SPIFFS_CACHE=y
CONFIG_SPIFFS_CACHE_WR=y
CONFIG_SPIFFS_CACHE_STATS=
CONFIG_SPIFFS_PAGE_CHECK=y
CONFIG_SPIFFS_GC_MAX_RUNS=10
CONFIG_SPIFFS_GC_STATS=
CONFIG_SPIFFS_PAGE_SIZE=256
CONFIG_SPIFFS_OBJ_NAME_LEN=32
CONFIG_SPIFFS_USE_MAGIC=y
CONFIG_SPIFFS_USE_MAGIC_LENGTH=y
CONFIG_SPIFFS_META_LENGTH=4
CONFIG_SPIFFS_USE_MTIME=y
CONFIG_SPIFFS_DBG=
CONFIG_SPIFFS_API_DBG=
CONFIG_SPIFFS_GC_DBG=
CONFIG_SPIFFS_CACHE_DBG=
CONFIG_SPIFFS_CHECK_DBG=
CONFIG_SPIFFS_TEST_VISUALISATION=
```

对比之后我们发现有 `SPIFFS_OBJ_META_LEN:0`（mkspiffs）和 `CONFIG_SPIFFS_META_LENGTH=4`（sdkconfig） 不同，这个可能就是导致无法从文件系统中读取的原因。那我们重新构建一下 `mkspiffs`，通过在 mkspiffs 目录中运行以下命令：

```c
make clean
make dist BUILD_CONFIG_NAME="-esp-idf" CPPFLAGS="-DSPIFFS_OBJ_META_LEN=4"
```

这时，我们重新比较下两者的差别:

```c
mkspiffs ver. 0.2.3-6-g983970e
Build configuration name: esp-idf
SPIFFS ver. 0.3.7-5-gf5e26c4
Extra build flags: -DSPIFFS_OBJ_META_LEN=4
SPIFFS configuration:
  SPIFFS_OBJ_NAME_LEN: 32
  SPIFFS_OBJ_META_LEN: 4			// 这里已经改变为和工程中的 SPIFFS 格式配置相同
  SPIFFS_USE_MAGIC: 1
  SPIFFS_USE_MAGIC_LENGTH: 1
  SPIFFS_ALIGNED_OBJECT_INDEX_TABLES: 0
```

```c
# SPIFFS Configuration
CONFIG_SPIFFS_MAX_PARTITIONS=3
# SPIFFS Cache Configuration
CONFIG_SPIFFS_CACHE=y
CONFIG_SPIFFS_CACHE_WR=y
CONFIG_SPIFFS_CACHE_STATS=
CONFIG_SPIFFS_PAGE_CHECK=y
CONFIG_SPIFFS_GC_MAX_RUNS=10
CONFIG_SPIFFS_GC_STATS=
CONFIG_SPIFFS_PAGE_SIZE=256
CONFIG_SPIFFS_OBJ_NAME_LEN=32
CONFIG_SPIFFS_USE_MAGIC=y
CONFIG_SPIFFS_USE_MAGIC_LENGTH=y
CONFIG_SPIFFS_META_LENGTH=4
CONFIG_SPIFFS_USE_MTIME=y
CONFIG_SPIFFS_DBG=
CONFIG_SPIFFS_API_DBG=
CONFIG_SPIFFS_GC_DBG=
CONFIG_SPIFFS_CACHE_DBG=
CONFIG_SPIFFS_CHECK_DBG=
CONFIG_SPIFFS_TEST_VISUALISATION=
```

这样应该就可以解决不能从文件系统中读取文件内容了。

## mkspiffs 工具构建

构建 mkspiffs 的过程：

```c
git clone https://github.com/igrr/mkspiffs.git
git submodule update --init
make dist
# make dist CPPFLAGS="-DSPIFFS_OBJ_META_LEN=4" BUILD_CONFIG_NAME=-custom //可选
```

### SPIFFS 配置
在 mkspiffs 构建时设置的某些 SPIFFS 选项会影响生成的文件系统映像的格式。在构建 mkspiffs 和构建使用 SPIFFS 的应用程序时，请确保将此类选项设置为相同的值。

这些选项包括：

```c
SPIFFS_OBJ_NAME_LEN
SPIFFS_OBJ_META_LEN
SPIFFS_USE_MAGIC
SPIFFS_USE_MAGIC_LENGTH
SPIFFS_ALIGNED_OBJECT_INDEX_TABLES
```

可能是其他人要查看这些选项的默认值，请检查此存储库中的 `include/spiffs_config.h` 文件。

要在构建时覆盖某些选项，请传递额外的 CPPFLAGS。您还可以设置 BUILD_CONFIG_NAME 变量以区分构建的二进制文件：

```c
make clean
make dist CPPFLAGS="-DSPIFFS_OBJ_META_LEN=4" BUILD_CONFIG_NAME=-custom
```

要检查构建mkspiff时设置的选项，请使用 `--version` 命令：

```c
$ mkspiffs --version
mkspiffs ver. 0.2.2
Build configuration name: custom
SPIFFS ver. 0.3.7-5-gf5e26c4
Extra build flags: -DSPIFFS_OBJ_META_LEN=4
SPIFFS configuration:
  SPIFFS_OBJ_NAME_LEN: 32
  SPIFFS_OBJ_META_LEN: 4
  SPIFFS_USE_MAGIC: 1
  SPIFFS_USE_MAGIC_LENGTH: 1
  SPIFFS_ALIGNED_OBJECT_INDEX_TABLES: 0
```

## 参考资料

 - [SPIFFS Filesystem](https://docs.espressif.com/projects/esp-idf/en/v3.2/api-reference/storage/spiffs.html)
