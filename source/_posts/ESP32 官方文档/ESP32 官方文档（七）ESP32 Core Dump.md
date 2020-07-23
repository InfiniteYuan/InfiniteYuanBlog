---
title: ESP32 官方文档（七）ESP32 Core Dump
date: 2018-09-01 21:55:33
categories:
- ESP32 官方文档
tags:
- ESP32
---

# ESP32 Core Dump

## 概述

ESP-IDF 支持在不可恢复的软件错误上生成 Core Dump。这个技术可以对软件发生故障时的软件状态进行事后分析。在系统崩溃进入 Panic 状态时，根据配置打印一些信息并停止或重新启动。用户可以选择生成 Core Dump，以便稍后在 PC 上分析故障原因。Core Dump 包含软件发生故障时系统中所有任务的快照。快照包括任务控制块(TCB)和堆栈。因此。有可能找出什么任务，在什么指令(代码行)和该任务的什么调用堆栈导致崩溃。ESP-IDF 提供特殊脚本 `espcoredump.py`，以帮助用户检索和分析 Core Dump。此工具提供两个用于 Core Dump 分析的命令:

 - `info_corefile` - 打印崩溃的任务的寄存器，调用堆栈，系统中可用任务的列表，内存区域和存储在 Core Dump (TCB 和堆栈)中的内存内容。
 - `dbg_corefile` - 创建 Core Dump ELF 文件并使用此文件运行 GDB 调试会话。用户可以手动检查内存，变量和任务状态。请注意，由于并非所有内存都保存在 Core Dump 中，因此只有堆栈上分配的变量值才有意义。

<!--more-->

## 配置

有许多与 Core Dump 相关的配置选项，用户可以在应用程序的配置菜单中选择 (`make menuconfig`)。

 1. Core Dump 数据目标(Components -> ESP32-specific config -> Core dump destination):
	- 禁用 Core Dump 生成
	- 将 Core Dump 保存到 Flash
	- 将 Core Dump 打印到 UART
 2. 核心转储中的最大任务快照数(Components -> ESP32-specific config -> Core dump -> Maximum number of tasks)。
 3. Core Dump 打印到 UART 之前的延迟时间(Components -> ESP32-specific config -> Core dump print to UART delay).。值以 ms 为单位。

## 将 Core Dump 保存到 Flash

选择此选项后，Core Dump 将保存到 Flash 上的特殊分区。当使用随 ESP-IDF 提供的默认分区表文件时，它会自动在 Flash 上分配必要的空间，但如果用户想要将自己的布局文件与 Core Dump 功能一起使用，则应为 Core Dump 定义单独的分区，如下所示:

```
# Name,   Type, SubType, Offset,  Size
# Note: if you change the phy_init or app partition offset, make sure to change the offset in Kconfig.projbuild
nvs,      data, nvs,     0x9000,  0x6000
phy_init, data, phy,     0xf000,  0x1000
factory,  app,  factory, 0x10000, 1M
coredump, data, coredump,,        64K
```

分区名称没有特殊要求。可以根据用户应用需求选择，但分区类型应为“数据”，子类型应为“coredump”。此外，在选择分区大小时请注意，Core Dump 数据结构会引入 20 字节的常量开销和 12 字节的每任务开销。此开销不包括每个任务的 TCB 和堆栈的大小，因此，partirion 大小应至少为 20 + max task stack number x(12 + TCB size + max task stack size) 字节。

从 Flash 分析 Core Dump 的通用命令示例是: 
`espcoredump.py -p </path/to/serial/port> info_corefile </path/to/program/elf/file>` 
或
 `espcoredump.py -p </path/to/serial/port> dbg_corefile </path/to/program/elf/file>`

## 将 Core Dump 打印到UART

选择此选项时，在系统崩溃进入 Panic 状态时，将在 UART 上打印 **base64 编码的 Core Dump**。在这种情况下，用户应手动将 Core Dump 文本正文保存到某个文件，然后运行以下命令: 
`espcoredump.py info_corefile -t b64 -c </path/to/saved/base64/text> </path/to/program/elf/file>` 
或
 `espcoredump.py dbg_corefile -t b64 -c </path/to/saved/base64/text> </path/to/program/elf/file>`

Base64 编码的 Core Dump 体将位于以下页眉和页脚之间:

```
================= CORE DUMP START =================
<body of base64-encoded core dump, save it to file on disk>
================= CORE DUMP END ===================
```

CORE DUMP START 和 CORE DUMP END 行不得包含在 Core Dump 文本文件中。

## Backtraces 中的 ROM 函数

可能的情况是，在崩溃时，一些任务或/和崩溃的任务本身在其调用堆栈中具有一个或多个 ROM 功能。由于 ROM 不是程序 ELF 的一部分，GDB 不可能解析这样的调用堆栈，因为它试图分析函数的序言来实现它。在这种情况下，调用堆栈打印将在第一个 ROM 函数中被错误消息打破。要解决此问题，您可以使用 Espressif 提供的 [ROM ELF](https://dl.espressif.com/dl/esp32_rom.elf) 并将其传递给 'espcoredump.py'。

## 运行 ‘espcoredump.py’

通用命令语法:

`espcoredump.py [options] command [args]`

`Script Options:`	

 - –chip,-c {auto,esp32}. Target chip type. Supported values are auto and esp32.
 - –port,-p PORT. Serial port device.
 - –baud,-b BAUD. Serial port baud rate used when flashing/reading.

`Commands:`	

 - info_corefile. Retrieve core dump and print useful info.
 - dbg_corefile. Retrieve core dump and start GDB session with it.

`Command Arguments:`
 	
 - –gdb,-g GDB. Path to gdb to use for data retrieval.
 - –core,-c CORE. Path to core dump file to use (if skipped core dump will be read from flash).
 - –core-format,-t CORE_FORMAT. Specifies that file passed with “-c” is an ELF (“elf”), dumped raw binary (“raw”) or base64-encoded (“b64”) format.
 - –off,-o OFF. Ofsset of coredump partition in flash (type “make partition_table” to see it).
 - –save-core,-s SAVE_CORE. Save core to file. Othwerwise temporary core file will be deleted. Ignored with “-c”.
 - –rom-elf,-r ROM_ELF. Path to ROM ELF file to use (if skipped “esp32_rom.elf” is used).
 - –print-mem,-m Print memory dump. Used only with “info_corefile”.

## 参考资料

 - [原文链接](https://docs.espressif.com/projects/esp-idf/en/latest/api-guides/core_dump.html)
