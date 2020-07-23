---
title: ESP32 官方文档（五）严重错误
date: 2018-09-01 17:12:54
categories:
- ESP32 官方文档
tags:
- ESP32
---

# 严重错误

## 概述

在某些情况下,程序的执行,没有按照定义的方式持续执行.在 ESP-IDF 中,这些情况包括:

 - CPU 异常:Illegal Instruction, Load/Store Alignment Error, Load/Store Prohibited error, Double Exception.(非法指令,加载/存储对齐错误,加载/存储禁止错误,双重异常)
 - 系统级别检查和安全措施:
	 - Interrupt watchdog timeout 中断看门狗超时
	 - Task watchdog timeout  任务监视程序超时(如果设置了 `CONFIG_TASK_WDT_PANIC`,则仅 fatal)
	 - Cache access error 缓存访问错误
	 - Brownout detection event 掉电检测事件
	 - Stack overflow 堆栈溢出
	 - Stack smashing protection check 堆栈粉碎保护检查
	 - Heap integrity check 堆完整性检查
 - Failed assertions 断言失败,通过 `assert` ,`configASSERT` 和类似的宏.

本指南介绍了 ESP-IDF 中用于处理这些错误的过程,并提供了有关错误故障排除的建议.

<!--more-->

## Panic 处理

概述中列出的每个错误原因都将由 Panic 处理程序处理.

Panic 处理程序将首先将错误原因打印到控制台. 对于 CPU 异常,消息类似于:

```
Guru Meditation Error: Core 0 panic'ed (IllegalInstruction). Exception was unhandled.
```

对于某些系统级别检查(中断监视程序,缓存访问错误),该消息将类似于:

```
Guru Meditation Error: Core 0 panic'ed (Cache disabled but cached memory region accessed)
```

在所有情况下,错误原因将打印在括号中. 有关可能的错误原因列表,请参阅 [Guru Meditation Errors](https://docs.espressif.com/projects/esp-idf/en/latest/api-guides/fatal-errors.html#guru-meditation-errors).

可以使用 `CONFIG_ESP32_PANIC` 配置选项设置 Panic 处理程序的后续行为. 可用选项包括:

 - 打印寄存器并重新启动(`CONFIG_ESP32_PANIC_PRINT_REBOOT`) - 默认选项.
	这将在异常点打印寄存器值,打印回溯,然后重新启动芯片.
 - 打印寄存器并暂停(`CONFIG_ESP32_PANIC_PRINT_HALT`)
	与上述选项类似,但暂停而不是重新启动. 重启程序需要外部重置.

 - 无提示重启(`CONFIG_ESP32_PANIC_SILENT_REBOOT`)
	不要打印寄存器或回溯,立即重启芯片.

 - 调用 GDB 存根(`CONFIG_ESP32_PANIC_GDBSTUB`)
	 启动 GDB 服务器,它可以通过控制台 UART 端口与 GDB 通信. 有关详细信息,请参阅 [GDB 存根](https://docs.espressif.com/projects/esp-idf/en/latest/api-guides/fatal-errors.html#gdb-stub).

Panic 处理程序的行为受另外两个配置选项的影响.
 
 - 如果启用了 `CONFIG_ESP32_DEBUG_OCDAWARE` (这是默认设置),则 Panic 处理程序将检测 JTAG 调试器是否已连接. 如果是,则执行将暂停,控制权将传递给调试器. 在这种情况下,寄存器和回溯不会转储到控制台,并且不使用 GDBStub/Core Dump 功能.
 - 如果启用了核心转储功能(`CONFIG_ESP32_ENABLE_COREDUMP_TO_FLASH` 或 `CONFIG_ESP32_ENABLE_COREDUMP_TO_UART` 选项),则系统状态(任务堆栈和寄存器)将被转储到 Flash 或 UART,以供以后分析.

下图说明了 Panic 处理程序的行为:

![Panic 处理程序流程图](https://img-blog.csdn.net/20180901140838695?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

## 寄存器转储和回溯

除非启用了 `CONFIG_ESP32_PANIC_SILENT_REBOOT` 选项,否则 Panic 处理程序会将一些 CPU 寄存器和回溯打印到控制台:

```
Core 0 register dump:
PC      : 0x400e14ed  PS      : 0x00060030  A0      : 0x800d0805  A1      : 0x3ffb5030
A2      : 0x00000000  A3      : 0x00000001  A4      : 0x00000001  A5      : 0x3ffb50dc
A6      : 0x00000000  A7      : 0x00000001  A8      : 0x00000000  A9      : 0x3ffb5000
A10     : 0x00000000  A11     : 0x3ffb2bac  A12     : 0x40082d1c  A13     : 0x06ff1ff8
A14     : 0x3ffb7078  A15     : 0x00000000  SAR     : 0x00000014  EXCCAUSE: 0x0000001d
EXCVADDR: 0x00000000  LBEG    : 0x4000c46c  LEND    : 0x4000c477  LCOUNT  : 0xffffffff

Backtrace: 0x400e14ed:0x3ffb5030 0x400d0802:0x3ffb5050
```

打印的寄存器值是异常帧中的寄存器值,即 CPU 异常或其他严重错误发生时的值.

如果由于 `abort()` 调用而执行了 Panic 处理程序,则不会打印寄存器转储.

在某些情况下,例如中断看门狗超时, Panic 处理程序可能会打印额外的 CPU 寄存器 (EPC1-EPC4) 以及在另一个 CPU 上运行的代码的寄存器/回溯.

Backtrace 行包含 PC:SP 对,其中 PC 是程序计数器,SP 是堆栈指针,用于当前任务的每个堆栈帧. 如果在 ISR 内发生严重错误,则回溯可能包括来自被中断的任务和来自 ISR 的 PC:SP 对.

如果使用 IDF Monitor,程序计数器值将转换为代码位置(函数名称,文件名和行号),输出将与其他行进行注释:

```
Core 0 register dump:
PC      : 0x400e14ed  PS      : 0x00060030  A0      : 0x800d0805  A1      : 0x3ffb5030
0x400e14ed: app_main at /Users/user/esp/example/main/main.cpp:36

A2      : 0x00000000  A3      : 0x00000001  A4      : 0x00000001  A5      : 0x3ffb50dc
A6      : 0x00000000  A7      : 0x00000001  A8      : 0x00000000  A9      : 0x3ffb5000
A10     : 0x00000000  A11     : 0x3ffb2bac  A12     : 0x40082d1c  A13     : 0x06ff1ff8
0x40082d1c: _calloc_r at /Users/user/esp/esp-idf/components/newlib/syscalls.c:51

A14     : 0x3ffb7078  A15     : 0x00000000  SAR     : 0x00000014  EXCCAUSE: 0x0000001d
EXCVADDR: 0x00000000  LBEG    : 0x4000c46c  LEND    : 0x4000c477  LCOUNT  : 0xffffffff

Backtrace: 0x400e14ed:0x3ffb5030 0x400d0802:0x3ffb5050
0x400e14ed: app_main at /Users/user/esp/example/main/main.cpp:36

0x400d0802: main_task at /Users/user/esp/esp-idf/components/esp32/cpu_start.c:470
```

要查找发生严重错误的位置,请查看“Backtrace”的下一行. 严重错误位置是顶行,后续行显示调用堆栈.

## GDB 存根

如果启用了 `CONFIG_ESP32_PANIC_GDBSTUB` 选项,则发生严重错误时, Panic 处理程序不会重置芯片. 相反,它将启动 GDB 远程协议服务器,通常称为 GDB Stub. 发生这种情况时,可以指示主机上运行的 GDB 实例连接到 ESP32 UART 端口.

如果使用 [IDF Monitor](https://docs.espressif.com/projects/esp-idf/en/latest/get-started/idf-monitor.html),则在 UART 上检测到 GDB Stub 提示时会自动启动 GDB. 输出看起来像这样:

```
Entering gdb stub now.
$T0b#e6GNU gdb (crosstool-NG crosstool-ng-1.22.0-80-gff1f415) 7.10
Copyright (C) 2015 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "--host=x86_64-build_apple-darwin16.3.0 --target=xtensa-esp32-elf".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from /Users/user/esp/example/build/example.elf...done.
Remote debugging using /dev/cu.usbserial-31301
0x400e1b41 in app_main ()
    at /Users/user/esp/example/main/main.cpp:36
36      *((int*) 0) = 0;
(gdb)
```

GDB 提示可用于检查 CPU 寄存器,本地和静态变量以及内存中的任意位置. 无法设置断点,更改 PC 或继续执行. 要重置程序,请退出 GDB 并执行外部重置:IDF Monitor 中的 `Ctrl-T Ctrl-R`,或使用开发板上的外部重置按钮.

## Guru Meditation 错误

本节解释了不同错误原因的含义,打印在 `Guru Meditation Error:Core panic'ed message` 之后的括号中.

> 有关“Guru Meditation”的历史渊源,请参阅 [Wikipedia 文章](https://en.wikipedia.org/wiki/Guru_Meditation).

### IllegalInstruction (非法指令)

该 CPU 异常表示执行的指令不是有效指令. 此错误的最常见原因有:

 - FreeRTOS 任务功能已经返回. 在 FreeRTOS 中,如果任务函数需要终止,它应该调用 vTaskDelete() 函数并删除它自己,而不是返回.
 - 无法从 SPI Flash 加载下一条指令. 这种情况通常发生在:
	 - 应用程序已将 SPI Flash 引脚重新配置为其他功能(GPIO,UART 等). 有关 SPI Flash 引脚的详细信息,请参阅硬件设计指南和芯片或模块的数据表.
	 - 某些外部设备意外连接到 SPI Flash 引脚,干扰了 ESP32 和 SPI Flash 之间的通信.

### InstrFetchProhibited (禁止指令加载)

此 CPU 异常表示 CPU 无法加载指令,因为指令的地址不属于指令 RAM 或 ROM 中的有效区域.

通常这意味着尝试调用函数指针,该指针不指向有效代码. PC (程序计数器)寄存器可用作指示器:它将为零或将包含垃圾值(不是 0x4xxxxxxx).

### LoadProhibited,StoreProhibited(禁止加载，禁止存储)

当应用程序尝试读取或写入无效的内存位置时,会发生此 CPU 异常. 写入/读取的地址可在寄存器转储中的 `EXCVADDR` 寄存器中找到. 如果此地址为零,则通常表示应用程序尝试取消引用 NULL 指针. 如果此地址接近于零,则通常意味着应用程序尝试访问结构的成员,但指向该结构的指针为 NULL. 如果该地址是别的(垃圾值,不在 `0x3fxxxxxx` - `0x6xxxxxxx` 范围内),则可能意味着用于访问数据的指针未初始化或已损坏.

### IntegerDivideByZero(除以 0)

应用程序尝试将整数除以零.

### LoadStoreAlignment(对齐方式不对)

应用程序尝试读取或写入内存位置,并且地址对齐与加载/存储大小不匹配. 例如,32 位加载只能从 4 字节对齐的地址完成,而 16 位加载只能从 2 字节的对齐地址完成.

### LoadStoreError(加载/存储错误)

应用程序尝试从仅支持 32 位加载/存储的内存区域进行 8 位或 16 位加载/存储. 例如,取消引用指向内存存储器的 `char *` 指针将导致这样的错误.

### Unhandled debug exception(堆栈错误)

通常会出现以下消息:

```
Debug exception reason: Stack canary watchpoint triggered (task_name)
```

此错误表示应用程序已写入 `task_name` 任务堆栈的末尾. 请注意,并非每个堆栈溢出都可以保证触发此错误. 任务可能会在堆栈 `canary` 位置之外写入堆栈,在这种情况下,不会触发观察点.

### Interrupt wdt timeout on CPU0 / CPU1(看门狗超时)

表示发生了中断看门狗超时. 有关详细信息,请参阅[看门狗](https://docs.espressif.com/projects/esp-idf/en/latest/api-reference/system/wdts.html).

### Cache disabled but cached memory region accessed(Cache 禁止)

在某些情况下,ESP-IDF 将暂时禁止通过高速缓存访问外部 SPI Flash 和 SPI RAM. 例如,spi_flash API 用于读取/写入/擦除/mmap SPI Flash 区域. 在这些情况下,任务被挂起,并且未注册 `ESP_INTR_FLAG_IRAM` 的中断处理程序被禁用. 确保使用此标志注册的任何中断处理程序都具有 IRAM/DRAM 中的所有代码和数据. 有关更多详细信息,请参阅 [SPI Flash API 文档](https://docs.espressif.com/projects/esp-idf/en/latest/api-reference/storage/spi_flash.html#iram-safe-interrupt-handlers).

## 其他严重错误

### Brownout(欠压)

ESP32 有一个内置的掉电检测器,默认启用. 如果电源电压低于安全水平,掉电检测器可以触发系统复位. 可以使用 `CONFIG_BROWNOUT_DET` 和 `CONFIG_BROWNOUT_DET_LVL_SEL` 选项配置掉电检测器. 当掉电检测器触发时,将打印以下消息:

```
Brownout detector was triggered
```

打印消息后,芯片将复位.

请注意,如果电源电压快速下降,则控制台上只能看到部分消息.

### Corrupt Heap

ESP-IDF 堆实现包含许多堆结构的运行时检查. 可以在 menuconfig 中启用其他检查(“Heap Stisoning”). 如果其中一项检查失败,将打印类似于以下内容的消息:

```
CORRUPT HEAP: Bad tail at 0x3ffe270a. Expected 0xbaad5678 got 0xbaac5678
assertion "head != NULL" failed: file "/Users/user/esp/esp-idf/components/heap/multi_heap_poisoning.c", line 201, function: multi_heap_free
abort() was called at PC 0x400dca43 on core 0
```

有关详细信息,请参阅[堆内存调试文档](https://docs.espressif.com/projects/esp-idf/en/latest/api-reference/system/heap_debug.html).

### Stack Smashing

可以使用 `CONFIG_STACK_CHECK_MODE` 选项在 ESP-IDF 中启用 Stack Smashing 保护(基于 GCC  `-fstack-protector *` 标志). 如果检测到 Stack Smashing,将打印类似于以下内容的消息:

```
Stack smashing protect failure!

abort() was called at PC 0x400d2138 on core 0

Backtrace: 0x4008e6c0:0x3ffc1780 0x4008e8b7:0x3ffc17a0 0x400d2138:0x3ffc17c0 0x400e79d5:0x3ffc17e0 0x400e79a7:0x3ffc1840 0x400e79df:0x3ffc18a0 0x400e2235:0x3ffc18c0 0x400e1916:0x3ffc18f0 0x400e19cd:0x3ffc1910 0x400e1a11:0x3ffc1930 0x400e1bb2:0x3ffc1950 0x400d2c44:0x3ffc1a80
0
```

回溯应该指向 Stack Smashing 发生的函数. 检查功能代码以获得对本地阵列的无限制访问.

## 参考资料

 - [原文链接](https://docs.espressif.com/projects/esp-idf/en/latest/api-guides/fatal-errors.html)
