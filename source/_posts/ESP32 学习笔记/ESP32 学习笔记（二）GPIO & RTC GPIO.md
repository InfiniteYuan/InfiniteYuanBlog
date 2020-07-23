---
title: ESP32 学习笔记（二）GPIO & RTC GPIO
date: 2018-08-08 11:02:31
categories:
- ESP32 学习笔记
tags:
- ESP32
---

# GPIO & RTC GPIO

## 概述

ESP32 芯片具有 40 个物理 GPIO pads (焊盘).某些 GPIO pad 既不能使用,在芯片封装上也没有相应的引脚(请参阅技术参考手册).所以只有 34 个物理 GPIO pads 可供使用.每个 pad 都可用作一个通用 IO,或连接一个内部的外设信号.IO_MUX、RTC IO_MUX 和 GPIO 交换矩阵用于将信号从外设传输至 GPIO pad.这些模块共同组成了芯片的 IO 控制.

注意:

 - 管脚 SCK/CLK,SDO/SD0,SDI/SD1,SHD/SD2,SWP/SD3,和 SCS/CMD,即 GPIO6 至 GPIO11 用于连接模组上集成的 SPI flash,不建议用于其他功能.
 - ESP32-D2WD 的管脚 GPIO16,GPIO17,SD_CMD,SD_CLK,SD_DATA_0 和 SD_DATA_1 用于连接嵌入式 Flash,不建议用于其他功能.
 - GPIO 34-39 只能设置为输入模式,没有软件上拉或下拉功能.
 - 这 34 个物理 GPIO pad 的序列号为:0-19, 21-23, 25-27, 32-39.其中 GPIO 34-39 仅用作输入管脚,其他的既可以作为输入又可以作为输出管脚.

<!--more-->

当 GPIO 被连接到 “RTC” 低功耗和模拟子系统时,还有独立的 “RTC GPIO” 支持. 这些引脚功能可在深度睡眠,[超低功耗协处理器](https://esp-idf.readthedocs.io/zh_CN/latest/api-guides/ulp.html)运行时或使用 ADC/DAC/等 模拟功能时使用.

下图描述了数字 pad(控制信号:FUNC_SEL、IE、OE、WPU、WDU 等)和 162 个外设输入以及 176 个外设输出信号(控制信号:SIG_IN_SEL、SIG_OUT_SEL、IE、OE 等)和快速外设输入/输出信号(控制信号:IE、OE 等)以及 RTC IO_MUX 之间的信号选择和连接关系.
![图1: IO_MUX、RTC IO_MUX 和GPIO 交换矩阵结构框图](https://img-blog.csdn.net/20180811133142567?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
<center>图1: IO_MUX、RTC IO_MUX 和GPIO 交换矩阵结构框图</center>

 1. IO_MUX 中每个 GPIO pad 有一组寄存器.每个 pad 可以配置成 GPIO 功能(连接 GPIO 交换矩阵)或者直连功能(旁路 GPIO 交换矩阵,快速信号如以太网、SDIO、SPI、JTAG、UART 等会旁路 GPIO 交换矩阵以实现更好的高频数字特性.所以高速信号会直接通过 IO_MUX 输入和输出.)
 2. GPIO 交换矩阵是外设输入和输出信号和 pad 之间的全交换矩阵.
	 - 芯片输入方向:162 个外设输入信号都可以选择任意一个 GPIO pad 的输入信号.
	 - 芯片输出方向:每个 GPIO pad 的输出信号可来自 176 个外设输出信号中的任意一个.
 3. RTC IO_MUX 用于控制 GPIO pad 的低功耗和模拟功能.只有部分 GPIO pad 具有这些功能.

## 通过 GPIO 交换矩阵的外设输入

为实现通过 GPIO 交换矩阵接收外设输入信号,需要配置 GPIO 交换矩阵从 34 个 GPIO(0-19,21-23,25-27,32-39)中获取外设输入信号的索引号(0-18,23-36,39-58,61-90,95-124,140-155,164-181,190-195,198-206).

输入信号通过 IO_MUX 从 GPIO pad 中读取.IO_MUX 必须设置相应 pad 为 GPIO 功能.这样 GPIO pad 的输入信号就可进入 GPIO 交换矩阵然后通过 GPIO 交换矩阵进入选择的外设输入.
![图2: 通过IO_MUX、GPIO 交换矩阵的外设输入](https://img-blog.csdn.net/20180811135015590?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
<center>图2: 通过IO_MUX、GPIO 交换矩阵的外设输入</center>

## 通过 GPIO 交换矩阵的外设输出

为实现通过 GPIO 交换矩阵输出外设信号,需要配置 GPIO 交换矩阵将输出索引号为 0-18,23-37,61-121,140-215,224-228 的外设信号输出到 28 个 GPIO (0-19, 21-23, 25-27, 32-33).

输出信号从外设输出到 GPIO 交换矩阵,然后到达 IO_MUX.IO_MUX 必须设置相应 pad 为 GPIO 功能.这样输出 GPIO 信号就能连接到相应 pad.

图3 所示为 176 个输出信号中的某一个信号通过 GPIO 交换矩阵到达 IO_MUX 然后连接到某个 pad.
![图3: 通过GPIO 交换矩阵输出信号](https://img-blog.csdn.net/20180811135319400?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
<center>图3: 通过GPIO 交换矩阵输出信号</center>

## IO_MUX 的直接 I/O 功能

快速信号如以太网、SDIO、SPI、JTAG、UART 等会使用直连功能(旁路 GPIO 交换矩阵)以实现更好的高频数字特性.所以高速信号会直接通过 IO_MUX 输入和输出.

这样比使用 GPIO 交换矩阵的灵活度要低,即每个 GPIO pad 的 IO_MUX 寄存器只有较少的功能选择,但可以实现更好的高频数字特性.

为实现外设 I/O 旁路 GPIO 交换矩阵必须配置两个寄存器:

- GPIO pad 的 IO_MUX 必须设置为相应的 pad 功能,表1列出了 pad 功能.
- 对于输入信号,必须置位 SIG_IN_SEL 寄存器,直接将输入信号输出到外设.

![IO_MUX Pad 列表](https://img-blog.csdn.net/20180811140112726?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
<center>表1: IO_MUX Pad 列表</center>

## RTC IO_MUX 的低功耗和模拟 I/O 功能

18 个 GPIO 管脚具有低功耗(低功耗 RTC)性能和模拟功能,由 ESP32 的 RTC 子系统控制.这些功能不使用 IO_MUX 和 GPIO 交换矩阵,而是使用 RTC_MUX 将 I/O 指向 RTC 子系统.

当这些管脚被配置为 RTC GPIO 管脚,作为输出管脚时仍然能够在芯片处于 Deep-sleep 睡眠模式下保持输出电平值或者作为输入管脚使用时可以将芯片从 Deep-sleep 中唤醒.

每个 pad 的模拟和 RTC 功能是由 RTC_GPIO_PINx 寄存器中的 RTC_IO_TOUCH_PADx_TO_GPIO 位控制的.此位默认置为 1,通过 IO_MUX 子系统输入输出信号,如前文所述.

如果清零 RTC_IO_TOUCH_PADx_TO_GPIO 位,则输入输出信号会经过 RTC 子系统.在这种模式下,RTC_GPIO_PINx 寄存器用于数字 I/O,pad 的模拟功能也可以实现.

## Light-sleep 模式管脚功能

当 ESP32 处于 Light-sleep 模式时管脚可以有不同的功能.如果某一 GPIO pad 的 IO_MUX 寄存器中 GPIOxx_SLP_SEL 位置为 1,芯片处于 Light-sleep 模式下将由另一组不同的寄存器控制 pad.

如果 GPIOxx_SLP_SEL 置为 0,则芯片在正常工作和 Light-sleep 模式下,管脚的功能一样.

## Pad Hold 特性

每个 IO pad(包括 RTC pad)都有单独的 hold 功能,由 RTC 寄存器控制.pad 的 hold 功能被置上后,pad 在置上 hold 那一刻的状态被强制保持,无论内部信号如何变化,修改 IO_MUX 配置或者 GPIO 配置,都不会改变 pad 的状态.应用如果希望在看门狗超时触发内核复位和系统复位时或者 Deep-sleep 时 pad 的状态不被改变,就需要提前把 hold 置上.

 - 对于数字 pad 而言, 若要在深度睡眠掉电之后保持 pad 输入输出的状态值,需要在掉电之前把寄存器 REG_DG_PAD_FORCE_UNHOLD 设置成 0.对于 RTC pad 而言,pad 的输入输出值,由寄存器 RTC_CNTL_HOLD_FORCE_REG 中相应的位来控制 Hold 和 Unhold pad 的值.
 - 在芯片被唤醒之后,若要关闭 Hold 功能,将寄存器 REG_DG_PAD_FORCE_UNHOLD 设置成 1.若想继续保持 pad 的值,可把 RTC_CNTL_HOLD_FORCE_REG 寄存器中相应的位设置成1.

示例：

RTC GPIO （deep sleep）：
```
rtc_gpio_init(27);
rtc_gpio_set_direction(27, RTC_GPIO_MODE_OUTPUT_ONLY);
rtc_gpio_set_level(27, 1);
rtc_gpio_pullup_en(27);
rtc_gpio_hold_en(27);
```

Digital GPIO：
```
/**
  * The state of digital gpio cannot be held during Deep-sleep, and it will resume the hold function
  * when the chip wakes up from Deep-sleep. If the digital gpio also needs to be held during Deep-sleep,
  * `gpio_deep_sleep_hold_en` should also be called.
  *
  * Power down or call gpio_hold_dis will disable this function.
  */
gpio_pad_select_gpio(26);
gpio_set_direction(26, GPIO_MODE_OUTPUT);
gpio_set_level(26, 1);

gpio_hold_en(26);
gpio_deep_sleep_hold_en();
```

## 应用示例

GPIO 输出/输入中断示例:[peripherals/gpio](https://github.com/espressif/esp-idf/tree/f9a4496/examples/peripherals/gpio).

## API Reference - Normal GPIO

### Header File

 - [driver/include/driver/gpio.h](https://github.com/espressif/esp-idf/blob/f9a4496/components/driver/include/driver/gpio.h)

## API Reference - RTC GPIO

### Header File

 - [driver/include/driver/rtc_io.h](https://github.com/espressif/esp-idf/blob/7abed5f/components/driver/include/driver/rtc_io.h)

## 参考资料

 - [GPIO & RTC GPIO](https://docs.espressif.com/projects/esp-idf/en/v3.2/api-reference/peripherals/gpio.html)
