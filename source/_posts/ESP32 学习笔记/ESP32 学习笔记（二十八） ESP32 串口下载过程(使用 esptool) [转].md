---
title: ESP32 学习笔记（二十八） ESP32 串口下载过程(使用 esptool) [转]
date: 2020-03-12 15:38:11
categories:
- ESP32 学习笔记
tags:
- ESP32
---

# ESP32 串口下载过程(使用 esptool)

原文链接：https://www.esp32.com/viewtopic.php?f=25&t=8161&p=34283#p34283

通常，我们会通过 python 脚本，从 PC 端，通过串口给 esp32 芯片或者模块下载固件程序。
通过 esptool (https://github.com/espressif/esptool) 脚本，我们可以很方便的实现。

但有时，我们也希望能通过其他单片机的串口，给 esp32/esp8266 进行固件更新。（前提是外部单片机能够控制esp32 进入串口下载模式)。
完整的串口通信协议可以参考这里(https://github.com/espressif/esptool/w ... -Protocol)
为了方便调试，我们可以通过运行 esptool，来查看上位机与esp32之间的通信过程。

<!--more-->

首先，esp32 的串口通信协议基于 SLIP (Serial Line Internet Protocol)。任何一个数据包以0xC0开始，以0xC0结尾。数据包内容中的0xC0替换为0xDB 0xDC。数据包内容中的0xDB替换为0xDB 0xDD.

在运行esptool的同时，在命令中添加 --trace 参数，即可查看通信过程中的串口数据。

以读取 flash id 的过程为例：

```c
python esptool.py -b 115200 -p /dev/cu.SLAB_USBtoUART --trace --no-stub flash_id
```

从输出的 log 中，我们可以看到串口通信的具体内容。

```c
TRACE +0.000 command op=0x09 data len=16 wait_response=1 timeout=3.000 data=1c20006040000080ffffffff00000000
TRACE +0.000 Write 26 bytes: 
    c000091000000000 001c200060400000 | .......... .`@..
    80ffffffff000000 00c0             | ..........
TRACE +0.004 Read 1 bytes: c0
TRACE +0.000 Read 13 bytes: 01090400c840160000000000c0
TRACE +0.000 Received full packet: 01090400c840160000000000
```

上面的 log 中，串口向 esp32 发送 26 字节十六进制序列

```
c000091000000000001c20006040000080ffffffff00000000c0
```

esp32 向串口返回13字节十六进制序列

```
c001090400c840160000000000c0
```