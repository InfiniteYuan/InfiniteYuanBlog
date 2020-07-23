---
title: ESP32 官方文档（四）错误处理
date: 2012-07-22 09:43:43
categories:
- ESP32 官方文档
tags:
- ESP32
---

# 错误处理

## 概述

识别和处理运行时错误对于开发健壮的应用程序非常重要. ESP-IDF 中可能存在多种运行时错误:

 -  可恢复的错误:
	- 函数通过返回值表示的错误(错误代码)
	- 使用 throw 关键字抛出的 C++ 异常
 - 不可恢复(严重)错误:
	- 断言失败(使用断言宏和等效方法)和 abort() 调用.
	- CPU 异常:access to protected regions of memory, illegal instruction(访问受保护的内存区域,非法指令)等.
	- 系统级别检查:watchdog timeout, cache access error, stack overflow, stack smashing, heap corruption(监视程序超时,缓存访问错误,堆栈溢出,堆栈粉碎,堆损坏)等.

本指南介绍了与可恢复错误相关的 ESP-IDF 错误处理机制,并提供了一些常见的错误处理模式.

有关诊断不可恢复错误的说明,请参阅[错误](https://docs.espressif.com/projects/esp-idf/en/latest/api-guides/fatal-errors.html).

<!--more-->

## 错误代码

大多数特定的 ESP-IDF 函数使用 `esp_err_t` 类型来返回错误代码. `esp_err_t` 是带符号的整数类型. `ESP_OK` 代码表示成功(无错误),定义为零.

各种 ESP-IDF 头文件使用预处理器定义来定义可能的错误代码. 通常这些定义以 `ESP_ERR_` 前缀开头. 通用错误的常见错误代码(内存不足,超时,无效参数等)在 `esp_err.h` 文件中定义. ESP-IDF 中的各种组件可以为特定情况定义附加的错误代码.

有关错误代码的完整列表,请参阅[错误代码参考](https://docs.espressif.com/projects/esp-idf/en/latest/api-reference/error-codes.html).

## 将错误代码转换为错误消息

对于 ESP-IDF 组件中定义的每个错误代码,可以使用 `esp_err_to_name()` 或 `esp_err_to_name_r()` 函数将 `esp_err_t` 值转换为错误代码名称. 例如,将 `0x101` 传递给 `esp_err_to_name()` 将返回 “ESP_ERR_NO_MEM” 字符串. 可以在日志输出中使用此类字符串,以便更容易理解发生了哪个错误.

此外,如果未找到匹配的 `ESP_ERR_` 值,`esp_err_to_name_r()` 函数将尝试将错误代码解释为[标准 POSIX 错误代码](http://pubs.opengroup.org/onlinepubs/9699919799/basedefs/errno.h.html). 这是使用 `strerror_r` 函数完成的. POSIX 错误代码(例如 `ENOENT`,`ENOMEM`)在 `errno.h` 中定义,通常从 `errno` 变量获得. 在 ESP-IDF 中,这个变量是线程本地的:多个 FreeRTOS 任务都有自己的 `errno` 副本. 设置 `errno` 的函数仅修改它们运行的任务的值.

默认情况下启用此功能,但可以禁用此功能以减少应用程序二进制文件大小. 请参阅 `CONFIG_ESP_ERR_TO_NAME_LOOKUP`. 禁用此功能后,仍会定义 `esp_err_to_name()` 和 `esp_err_to_name_r()` ,并且可以调用它. 在这种情况下,`esp_err_to_name()` 将返回`UNKNOWN ERROR`,并且 `esp_err_to_name_r()` 将返回 `Unknown error 0xXXXX(YYYYY)` ,其中` 0xXXXX` 和 `YYYYY` 分别是错误代码的十六进制和十进制表示.

## `ESP_ERROR_CHECK` 宏

`ESP_ERROR_CHECK()` 宏与 `assert` 的用途相似,只是它检查 `esp_err_t` 值而不是 `bool` 条件. 如果 `ESP_ERROR_CHECK()` 的参数不等于 `ESP_OK` ,则在控制台上打印错误消息,并调用 `abort()`.

错误消息通常如下所示:
```
ESP_ERROR_CHECK failed: esp_err_t 0x107 (ESP_ERR_TIMEOUT) at 0x400d1fdf

file: "/Users/user/esp/example/main/main.c" line 20
func: app_main
expression: sdmmc_card_init(host, &card)

Backtrace: 0x40086e7c:0x3ffb4ff0 0x40087328:0x3ffb5010 0x400d1fdf:0x3ffb5030 0x400d0816:0x3ffb5050
```

> 如果使用 [IDF monitor](https://docs.espressif.com/projects/esp-idf/en/latest/get-started/idf-monitor.html),则回溯中的地址将转换为文件名和行号.

 - 第一行提到错误代码为十六进制值,以及源代码中用于此错误的标识符. 后者取决于设置的 `CONFIG_ESP_ERR_TO_NAME_LOOKUP` 选项. 打印出错误的程序中的地址也被打印出来.
 - 后续行显示程序中调用 `ESP_ERROR_CHECK()` 宏的位置,以及作为参数传递给宏的表达式.
 - 最后,打印回溯. 这是所有错误 `panic` 处理程序输出的公共部分. 有关回溯的更多信息,请参阅[错误 Fatal Errors](https://docs.espressif.com/projects/esp-idf/en/latest/api-guides/fatal-errors.html).

## 错误处理模式

 1. 尝试恢复. 根据具体情况,这可能意味着在一段时间后重试呼叫,或尝试取消初始化驱动程序并重新初始化,或使用带外机制修复错误条件(例如重置外部外围设备) 这没有回应).

	例:
	```
	esp_err_t err;
	do {
	    err = sdio_slave_send_queue(addr, len, arg, timeout);
	    // keep retrying while the sending queue is full
	} while (err == ESP_ERR_TIMEOUT);
	if (err != ESP_OK) {
	    // handle other errors
	}
	```

 2. 将错误传播给调用者. 在某些中间件组件中,这意味着函数必须以相同的错误代码退出,从而确保回滚任何资源分配.

	例:
	```
	sdmmc_card_t* card = calloc(1, sizeof(sdmmc_card_t));
if (card == NULL) {
    return ESP_ERR_NO_MEM;
}
esp_err_t err = sdmmc_card_init(host, &card);
if (err != ESP_OK) {
    // Clean up
    free(card);
    // Propagate the error to the upper layer (e.g. to notify the user).
    // Alternatively, application can define and return custom error code.
    return err;
}
	```
	
 3. 转换为不可恢复的错误,例如使用 `ESP_ERROR_CHECK`. 有关详细信息,请参阅 [`ESP_ERROR_CHECK` 宏](https://docs.espressif.com/projects/esp-idf/en/latest/api-guides/error-handling.html#esp-error-check-macro)部分.

	在出现错误的情况下终止应用程序通常是中间件组件的不良行为,但有时在应用程序级别可以接受.

	许多 ESP-IDF 示例使用 `ESP_ERROR_CHECK` 来处理来自各种API的错误. 这不是应用程序的最佳实践,并且可以使示例代码更简洁.

	例:
	```
	ESP_ERROR_CHECK(spi_bus_initialize(host, bus_config, dma_chan));
	```

## C++ 异常

默认情况下禁用对 ESP-IDF 中的 C++ 异常的支持,但可以使用 `CONFIG_CXX_EXCEPTIONS` 选项启用.

启用异常处理通常会将应用程序二进制大小增加几KB. 此外,可能需要为异常紧急池预留一定量的 RAM. 如果无法从堆中分配异常对象,则将使用此池中的内存. 可以使用 `CONFIG_CXX_EXCEPTIONS_EMG_POOL_SIZE` 变量设置应急池中的内存量.

如果抛出异常,但没有 `catch` 块,程序将被 `abort` 函数终止,并且将打印 `backtrace`. 有关回溯的更多信息,请参阅[错误](https://docs.espressif.com/projects/esp-idf/en/latest/api-guides/fatal-errors.html).

## 参考资料

 - [原文链接](https://docs.espressif.com/projects/esp-idf/en/latest/api-guides/error-handling.html)
