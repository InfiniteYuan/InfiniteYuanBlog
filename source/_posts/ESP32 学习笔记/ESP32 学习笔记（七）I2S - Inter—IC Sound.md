---
title: ESP32 学习笔记（七）I2S - Inter—IC Sound
date: 2018-08-12 22:18:02
categories:
- ESP32 学习笔记
tags:
- ESP32
---

# I2S

## 概述

ESP32 包含两个 I2S 外设. 这些外设可配置为通过 I2S 驱动程序输入和输出样本数据.

I2S 标准总线定义了三种信号:时钟信号 BCK、声道选择信号 WS 和串行数据信号 SD.一个基本的 I2S 数据总线有一个主机和一个从机.主机和从机的角色在通信过程中保持不变.ESP32 的 I2S 模块包含独立的发送和接收声道,能够保证优良的通信性能.

I2S 外设支持 DMA,这意味着它可以传输使用样本数据流,而无需 CPU 读取或写入每个样本.

I2S 输出也可以直接路由到数字/模拟转换器输出通道(GPIO 25 和 GPIO 26),直接产生模拟输出,而不是通过外部 I2S 编解码器.

<!--more-->

>对于高精度时钟应用,APLL 时钟源可与.use_apll = true 一起使用,ESP32 将自动计算 APLL 参数.

<clear>
>如果 use_apll = true 且 fixed_mclk> 0,则 I2S 的主时钟输出固定且等于 fixed_mclk 值. 音频时钟速率(LRCK)始终为 MCLK 除数,`0<MCLK / LRCK / channels / bits_per_sample <64`

![这里写图片描述](https://img-blog.csdn.net/20180812222632469?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
<center>图 1 I2S 系统框图</center>

图 1 是 ESP32 I2S 模块的结构框图,图中 "n"  对应为 0 或 1,即 I2S0 或 I2S1.每个 I2S 模块包含一个独立的发送单元(Tx) 和一个独立的接收单元(Rx).发送和接收单元各自有一组三线接口,分别为时钟线 BCK,声道选择线 WS 和串行数据线 SD.其中,发送单元的串行数据线固定为输出,接收单元的串行数据线固定为接收.发送单元和接收单元的时钟线和声道选择线均可配置为主机发送和从机接收.在 LCD 模式下,串行数据线扩展为并行数据总线.I2S 模块发送和接收单元各有一块宽 32 bit、深 64 bit 的 FIFO.此外,只有 I2S0 支持接收/发送 PDM 信号并且支持片上 DAC/ADC 模块.

图 1 右侧为 I2S 模块的信号总线.Rx 和 Tx 模块的信号命名规则为: I2SnA_B_C.其中 "n" 为模块名,表示 I2S0 或 I2S1；"A" 表示 I2S 模块的数据总线信号的方向,"I" 表示输入,"O" 表示输出；"B" 表示信号功能；"C" 表示该信号的方向,"in" 表示该信号输入I2S 模块,"out" 表示该信号自 I2S 模块输出.各信号总线的具体描述见表 1.除 I2Sn_CLK 信号外,其他信号均需要经过 GPIO 交换矩阵和 IO_MUX 映射到芯片的管脚.I2Sn_CLK 信号需要经过 IO_MUX 映射到芯片管脚.

![I2S 信号总线描述](https://img-blog.csdn.net/20180812223349366?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
<center>表 1 I2S 信号总线描述</center>

## 主要特性

I2S 模式

 - 可配置高精度输出时钟；
 - 支持全双工和半双工收发数据；
 - 支持多种音频标准；
 - 内嵌 A 律压缩/解压缩模块；
 - 可配置时钟；
 - 支持 PDM 信号输入输出；
 - 收发数据模式可配置.

LCD 模式

 - 支持外接 LCD；
 - 支持外接 Camera；
 - 支持多种 LCD 模式；
 - 支持连接片上 DAC/ADC 模式.

I2S 中断

 - I2S 接口中断；
 - I2S DMA 接口中断.

## 应用示例

esp-idf 中提供了完整的 I2S 示例:[peripherals/i2s](https://github.com/espressif/esp-idf/tree/30545f4/examples/peripherals/i2s).

I2S 配置的简短示例:

```c
#include "driver/i2s.h"
#include "freertos/queue.h"

static const int i2s_num = 0; // i2s port number

static const i2s_config_t i2s_config = {
     .mode = I2S_MODE_MASTER | I2S_MODE_TX,
     .sample_rate = 44100,
     .bits_per_sample = 16,
     .channel_format = I2S_CHANNEL_FMT_RIGHT_LEFT,
     .communication_format = I2S_COMM_FORMAT_I2S | I2S_COMM_FORMAT_I2S_MSB,
     .intr_alloc_flags = 0, // default interrupt priority
     .dma_buf_count = 8,
     .dma_buf_len = 64,
     .use_apll = false
};

static const i2s_pin_config_t pin_config = {
    .bck_io_num = 26,
    .ws_io_num = 25,
    .data_out_num = 22,
    .data_in_num = I2S_PIN_NO_CHANGE
};

...

    i2s_driver_install(i2s_num, &i2s_config, 0, NULL);   //install and start i2s driver

    i2s_set_pin(i2s_num, &pin_config);

    i2s_set_sample_rates(i2s_num, 22050); //set sample rates

    i2s_driver_uninstall(i2s_num); //stop & destroy i2s driver
```

配置 I2S 以使用内部 DAC 进行模拟输出的简短示例:

```c
#include "driver/i2s.h"
#include "freertos/queue.h"

static const int i2s_num = 0; // i2s port number

static const i2s_config_t i2s_config = {
     .mode = I2S_MODE_MASTER | I2S_MODE_TX | I2S_MODE_DAC_BUILT_IN,
     .sample_rate = 44100,
     .bits_per_sample = 16, /* the DAC module will only take the 8bits from MSB */
     .channel_format = I2S_CHANNEL_FMT_RIGHT_LEFT,
     .communication_format = I2S_COMM_FORMAT_I2S_MSB,
     .intr_alloc_flags = 0, // default interrupt priority
     .dma_buf_count = 8,
     .dma_buf_len = 64,
     .use_apll = false
};

...

    i2s_driver_install(i2s_num, &i2s_config, 0, NULL);   //install and start i2s driver

    i2s_set_pin(i2s_num, NULL); //for internal DAC, this will enable both of the internal channels

    //You can call i2s_set_dac_mode to set built-in DAC output mode.
    //i2s_set_dac_mode(I2S_DAC_CHANNEL_BOTH_EN);

    i2s_set_sample_rates(i2s_num, 22050); //set sample rates

    i2s_driver_uninstall(i2s_num); //stop & destroy i2s driver
```
## API Reference

### Header File

 - [driver/include/driver/i2s.h](https://github.com/espressif/esp-idf/blob/30545f4/components/driver/include/driver/i2s.h)

## 参考资料

 - [I2S](https://docs.espressif.com/projects/esp-idf/en/v3.2/api-reference/peripherals/i2s.html)
