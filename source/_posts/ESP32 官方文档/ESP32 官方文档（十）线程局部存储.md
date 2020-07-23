---
title: ESP32 官方文档（十）线程局部存储
date: 2018-09-02 15:59:35
categories:
- ESP32 官方文档
tags:
- ESP32
---

# 线程局部存储

## 概述

线程局部存储 (TLS) 是一种机制,通过该机制分配变量,使每个现存线程有一个变量实例. ESP-IDF 提供了三种利用这些变量的方法:

[FreeRTOS 原生 API](#freertos-%E5%8E%9F%E7%94%9F-api): ESP-IDF FreeRTOS 原生 API.
[多线程 API](#%E5%A4%9A%E7%BA%BF%E7%A8%8B-api):ESP-IDF 的多线程 API.
[C11 标准](#c11-%E6%A0%87%E5%87%86):C11 标准引入了特殊关键字来将变量声明为线程局部.

<!--more-->

## FreeRTOS 原生 API

ESP-IDF FreeRTOS 提供以下 AP I来管理线程局部变量:

 - vTaskSetThreadLocalStoragePointer()
 - pvTaskGetThreadLocalStoragePointer()
 - vTaskSetThreadLocalStoragePointerAndDelCallback()

在这种情况下,可分配的最大变量数受 `configNUM_THREAD_LOCAL_STORAGE_POINTERS` 宏的限制. 变量保存在任务控制块 (TCB) 中,并通过索引访问. 请注意,索引 0 保留用于 ESP-IDF 内部使用. 使用该 API,用户可以分配任意大小的线程局部变量,并将它们分配给任意数量的任务. 不同的任务可以有不同的 TLS 变量集. 如果变量的大小超过 4 个字节,则用户负责为其分配/释放内存. 变量的释放由 FreeRTOS 在删除任务时启动,但用户必须提供函数(回调)才能进行适当的清理.

## 多线程 API

ESP-IDF提供以下 多线程 API 来管理线程局部变量:

 - pthread_key_create()
 - pthread_key_delete()
 - pthread_getspecific()
 - pthread_setspecific()

此 API 具有上述 API 的所有优点,但消除了一些限制. 变量的数量仅受堆上可用内存大小的限制. 由于动态特性,与原生 API 相比,此 API 引入了额外的性能开销.

## C11 标准

ESP-IDF FreeRTOS 支持根据 C11 标准的线程局部变量(使用 `__thread` 关键字指定的变量). 有关此 GCC 功能的详细信息,请参阅https://gcc.gnu.org/onlinedocs/gcc-5.5.0/gcc/Thread-Local.html#Thread-Local. 这种变量的存储在任务堆栈上分配. 请注意,程序中所有此类变量的区域将分配到系统中每个任务的堆栈上,即使该任务根本不使用此类变量也是如此. 例如, ESP-IDF 系统任务(如 `ipc`, `timer` 任务等)也将分配额外的堆栈空间. 因此应谨慎使用此功能. 有一个权衡: C11 线程局部变量在编程中非常方便,只需使用几个 Xtensa 指令即可访问,但这个好处与系统中所有任务的额外堆栈使用成本有关. 由于变量的静态性质,系统中的所有任务都具有相同的 C11 线程局部变量集.

## 参考资料

 - [原文链接](https://docs.espressif.com/projects/esp-idf/en/latest/api-guides/thread-local-storage.html)
