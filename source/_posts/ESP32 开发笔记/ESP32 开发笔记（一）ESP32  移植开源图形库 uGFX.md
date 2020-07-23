---
title: ESP32 开发笔记（一）ESP32  移植开源图形库 uGFX
date: 2018-01-10 18:40:02
categories:
- ESP32 开发笔记
tags:
- ESP32
---

# ESP32 移植开源图形库 ugfx

ESP32 移植 uGFX 的源码：[GitHub 源码地址](https://github.com/InfiniteYuan/esp32-ugfx-gui)
欢迎 Star ～

## 源码工程分析

 1. /3rdparty 这里面包含第三方相关的功能代码
 2. /boards 一些公用开发板的使用资料
 3. /demos 例子应用
 4. /docs 帮助文档
 5. /drivers 底层驱动代码
 6. /src 公共源码
 7. /tools 相关工具

-------------------

## 移植过程

在移植之前，你需要配置 esp32-iot-solution 的环境，可参考： [ ESP-IDF Programming Guide ](https://esp-idf.readthedocs.io/en/latest/get-started/index.html)

 - 从 ugfx 官网下载源码：[官网下载链接](https://community.ugfx.io/index.php?/files/)
 - 在工程下单独建立一个 ugfx 文件夹，并在下面新建一个 include 文件夹（其中放入你需要包含的头文件）
 - 拷贝配置头文件 gfxconf.h 和 gfx.h 到 include 文件夹
 - 拷贝源码文件 /src 到 ugfx 文件夹下
 - 添加驱动芯片的驱动文件到 ugfx 文件夹下（例如，你使用的 LCD 屏驱动和 touch 驱动，在 drives 中可以找到一些模板）
 - 增加自己的板级文件 board_SSD1306.c
 - 实现相关缺少函数，相关函数可以在 drivers 的驱动模板中看到。你需要选择性的实现一些函数，另一些函数使用默认的即可。

这是我的工程结构，其中实现了：LCD(ST7789)、touch(XPT2046)
![这是我的工程结构](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTgwMTEwMTgxNzM5NjM3?x-oss-process=image/format,png)
在下面的两节，我将介绍我在移植 ugfx 中的部分过程，可供参考

## 移植 LCD 驱动

 - 由于 ugfx 官方未提供 st7789 的驱动模板，在这里我们使用 drivers/gdisp/ST7735 中的驱动模板，将其拷贝到我们的 ugfx 文件夹下，并从 ST7565 中拷贝一个头文件 board_ST7565_template.h，共需添加五个文件。
 - 接着我们实现一些函数，例如 init_board、post_init_board、setpin_reset、acquire_bus、release_bus、write_cmd、write_data;这些都在 board_ST7565_template.h 中有默认定义，我们需要根据我们的工程情况修改实现方法
 - 我们还需要对文件中的相关宏定义作修改

## 移植 touch 驱动

 - 在移植完上面的 lcd 驱动，相信你已经找到感觉，我们可以直接中 drivers/ginput/ 中拷贝 ads7843 下的文件到 ugfx 文件夹下，接着和移植 lcd 是类似的操作
 - 实现函数，例如：getpin_pressed、read_value、init_board，在实现 aquire_bus、release_bus 的时候，可能编译器会报函数重定义的错，这个时候，我们需要查看是否头文件引用错误等

## 编译、运行

还需要根据 iot-solution 的规则拷贝 component.mk  到我们的工程文件夹下（参考上图），接着我们可以编译运行，其中会遇到很多文件包含错误，我们需要耐心的一个个去解决；在完成之后我们就可以使用了。
