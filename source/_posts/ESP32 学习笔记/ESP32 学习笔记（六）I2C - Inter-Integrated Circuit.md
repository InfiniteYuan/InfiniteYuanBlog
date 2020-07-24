---
title: ESP32 学习笔记（六）I2C - Inter-Integrated Circuit
date: 2018-08-12 20:07:28
categories:
- ESP32 学习笔记
tags:
- ESP32
---

# I2C

I2C (内部集成电路)总线可用于与连接到与 ESP32 相同的总线的多个外部设备进行通信。ESP32 板上有两个 I2C 控制器，每个控制器可以设置为主模式或从模式。

## 概述

以下部分将指导您完成配置和操作 I2C 驱动程序的基本步骤：

 1. [配置驱动程序](#配置驱动程序) - 选择驱动程序的参数，如主模式或从模式，设置特定的 GPIO 引脚作为 SDA 和 SCL，设置时钟速度等。
 2. [安装驱动程序](#安装驱动程序) - 在主站或从站模式下激活驱动程序，以便在 ESP32 上运行两个 I2C 控制器其中一个。
 3. [进行通讯](#进行通讯)：
	 - 主机模式 - 作为主机运行通信
	 - 从机模式 - 从机响应来自主机的消息
 4. [中断处理](#中断处理) - 配置和 I2C 中断服务。
 5. [超出默认值](#超出默认值) - 调整 I2C 通信的时序，引脚配置和其他参数。
 6. [错误处理](#错误处理) - 如何识别和处理驱动程序配置和通信错误。
 7. [删除驱动程序](#删除驱动程序) - 在通信结束时释放 I2C 驱动程序所使用的资源。

I2C 驱动程序标识是从 `i2c_port_t` 中选择的两个端口号之一。在驱动程序配置期间通过 `i2c_mode_t` 选择 “master” 或 “slave” 指定端口的操作模式。

<!--more-->

## 配置驱动程序

建立 I2C 通信的第一步是配置驱动程序。这是通过设置 `i2c_config_t` 结构中包含的几个参数来完成的：

 - I2C 操作模式 - 从 `i2c_opmode_t` 中选择 slave 或 master
 - 通讯引脚配置：
	 - 分配给 SDA 和 SCL 信号的 GPIO 引脚编号
	 - 是否为各个引脚启用 ESP32 的内部上拉
 - I2C 时钟速度，如果此配置涉及主模
 - 如果此配置涉及从属模式：
	 - 是否应启用 10 位地址模式
	 - 从机地址

然后，要初始化给定 I2C 端口的配置，请使用端口号和 `i2c_config_t` 结构作为函数调用参数调用函数 `i2c_param_config()`。

在此阶段，`i2c_param_config()` 还将“其他 I2C 配置参数”设置为常用的默认值。要检查值是什么以及如何更改它们，请参阅[超出默认值](#超出默认值)。

## 安装驱动程序

初始化配置后，下一步是通过调用 `i2c_driver_install()` 来安装 I2C 驱动程序。此函数调用需要以下参数：

 - 端口号，可用的两个端口之一，从 `i2c_port_t` 中选择
 - 从 `i2c_opmode_t` 中选择的操作模式，从机或主机
 - 将分配用于在从机模式下发送和接收数据的缓冲区的大小
 - 用于分配中断的标志

## 进行通讯

安装 I2C 驱动程序后，ESP32 即可与其他 I2C 设备通信。通信编程取决于所选 I2C 端口是以主机模式还是从机模式工作。

### 主机模式

ESP32 工作在主机模式的 I2C 端口负责与从 I2C 设备建立通信，并发送命令以触发从机设备工作，如进行测量和发回结果。

为了组织这个过程，驱动程序提供了一个称为 “command link” 的容器，该容器应填充一系列命令，然后传递给 I2C 控制器执行。

#### 主机 Write

构建 I2C 主设备向从设备发送 n 个字节的命令链接的示例如下所示：
![I2C command link - master write example](https://img-blog.csdn.net/20180812211151469?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

下面介绍如何设置“主写入”的命令链接以及内部的内容：

 1. 第一步是使用 `i2c_cmd_link_create()` 创建命令链接。然后命令链接填充一系列要发送给从站的数据：
	 - 起始位 -  `i2c_master_start()`
	 - 单字节从机地址 - `i2c_master_write_byte()`。该地址作为此函数调用的参数提供。
	 - 一个或多个字节的数据作为 `i2c_master_write()` 的参数。
	 - 停止位 -  `i2c_master_stop()`

`i2c_master_write_byte()` 和 `i2c_master_write()` 命令都有另外的参数来定义 slave 是否应该确认接收的数据。
 2. 通过调用 `i2c_master_cmd_begin()` 来触发 I2C 控制器执行命令链接。
 3. 最后一步，完成命令发送后，通过调用 `i2c_cmd_link_delete()` 释放命令链接使用的资源。

#### 主机 Read

和主机类似的步骤序列，用于从从机读取数据。
![I2C command link - master read example](https://img-blog.csdn.net/20180812214852217?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

在读取数据时，不是 “i2c_master_read ...”，而是使用 `i2c_master_read_byte()` 和/或 `i2c_master_read()` 填充命令链接。此外，最后一次读取配置为不提供主机的确认。

发送从机地址后，请参见上图中的步骤 3，主机可以写入或从从机读取数据。主设备实际执行的操作信息隐藏在从设备地址的最低位。

因此，命令链接指示从机主机将写入数据包含地址，如 `(ESP_SLAVE_ADDR << 1)| I2C_MASTER_WRITE`，如下所示：

### 从机模式

API 提供了从属设备读取和写入数据的功能 - * `i2c_slave_read_buffer()` 和 `i2c_slave_write_buffer()`。[peripherals/i2c](https://github.com/espressif/esp-idf/tree/30545f4/examples/peripherals/i2c)中提供了使用这些功能的示例。

## 中断处理

调用函数 `i2c_isr_register()` 注册中断处理程序，调用 `i2c_isr_free()` 删除处理程序。[ESP32技术参考手册(PDF)](https://espressif.com/sites/default/files/documentation/esp32_technical_reference_manual_en.pdf)中提供了 I2C 控制器触发的中断描述。

## 超出默认值

在驱动程序配置期间(调用 `i2c_param_config()` 时，请参阅[配置驱动程序](https://esp-idf.readthedocs.io/en/latest/api-reference/peripherals/i2c.html#i2c-api-configure-driver))，有一些 I2C 通信参数设置为某些默认的常用值。某些参数也已在 I2C 控制器的寄存器中配置。通过调用专用函数，可以将这些参数更改为用户定义的值：

 - SCL 脉冲周期为高和低 - `i2c_set_period()`
 - 在产生启动/停止信号期间使用的 SCL 和 SDA 信号时序 - `i2c_set_start_timing()`/ `i2c_set_stop_timing()`
 - 从器件采样时以及通过主器件发送时 SCL 和 SDA 信号之间的时序关系 - `i2c_set_data_timing()`
 - I2C 超时 - `i2c_set_timeout()`
>定时值在 APB 时钟周期中定义。APB 的频率在 `I2C_APB_CLK_FREQ` 中指定。

 - 首先发送/接收 LSB 或 MSB -`i2c_set_data_mode()` 可在 `i2c_trans_mode_t` 中定义的模式中选择
 
上述每个函数都有一个`_get_`对应项来检查当前设置的值。

要在驱动程序配置期间查看参数设置的默认值，请参阅文件[driver / i2c.c](https://github.com/espressif/esp-idf/blob/30545f4/components/driver/i2c.c)查找定义了`_DEFAULT`后缀。

通过功能`i2c_set_pin()`，还可以选择不同的 SDA 和 SCL 引脚并改变上拉配置，改变已经输入的 `i2c_param_config()`。

>ESP32 的内部上拉电阻范围为几十 kOhm，因此在大多数情况下，它们本身不足以用作 I2C 上拉电阻。我们建议添加外部上拉电阻，其值如 I2C 标准中所述。

## 错误处理

大多数驱动程序的函数在成功完成时返回 `ESP_OK`，或者在失败时返回特定的错误代码。始终检查返回的值并实现错误处理是一种好习惯。例如，驱动程序也打印出志消息。检查输入配置的正确性，其中包含错误说明。有关详细信息，请参阅文件[driver / i2c.c](https://github.com/espressif/esp-idf/blob/30545f4/components/driver/i2c.c)查找定义 `_ERR_STR后` 缀。

使用专用中断来捕获通信故障。例如，当 I2C 花费太长时间来接收数据时，会触发 `I2C_TIME_OUT_INT` 中断。有关相关信息，请参阅[中断处理](https://esp-idf.readthedocs.io/en/latest/api-reference/peripherals/i2c.html#i2c-api-interrupt-handling)。

要在通信失败时重置内部硬件缓冲区，可以使用`i2c_reset_tx_fifo()`和`i2c_reset_rx_fifo()`。

## 删除驱动程序

如果使用 `i2c_driver_install()` 建立 I2C 通信一段时间之后不再需要 I2C 通信，则可以通过调用 `i2c_driver_delete()` 来移除驱动程序以释放分配的资源。

## 应用示例

I2C 主机和从机示例：[peripherals/i2c](https://github.com/espressif/esp-idf/tree/30545f4/examples/peripherals/i2c).

## API Reference

### Header File

 - [driver/include/driver/i2c.h](https://github.com/espressif/esp-idf/blob/30545f4/components/driver/include/driver/i2c.h)

## 参考资料
 - [I2C](https://docs.espressif.com/projects/esp-idf/en/v3.2/api-reference/peripherals/i2c.html)
