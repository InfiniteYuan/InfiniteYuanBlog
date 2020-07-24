---
title: ESP32 学习笔记（二十三）看门狗
date: 2019-03-12 10:52:35
categories:
- ESP32 学习笔记
tags:
- ESP32
---

# 看门狗

## 概述

ESP-IDF 支持两种类型的看门狗：中断看门狗定时器和任务看门狗定时器（TWDT）。中断看门狗定时器和 TWDT 都可以使用 make menuconfig 启用，但 TWDT 也可以在运行时启用。中断看门狗负责检测 FreeRTOS 任务切换长时间被阻止的情况。TWDT 负责检测运行的任务在长时间没有让出 CPU 的情况。

## 中断看门狗

中断看门狗可确保 FreeRTOS 任务切换中断长时间不被阻止。这种情况很不好，因为没有其他任务能获得 CPU 运行时间，包括可能重要的任务，如 WiFi 任务和空闲任务。阻塞的任务切换中断可能发生当一个程序运行到无限循环并且中断被禁用或挂起中断。

中断监视程序的默认操作是调用 panic 处理程序。将会把寄存器转储以便于使用 OpenOCD 或 gdbstub找到，在禁用中断的情况下会阻塞哪些代码。根据 panic 处理程序的配置，它还可以盲目地重置 CPU，这在生产环境中可能是首选。

中断看门狗是围绕定时器组 1 中的硬件看门狗构建的。如果由于某种原因这个看门狗无法执行调用 panic 处理程序的 NMI 处理程序（例如，因为 IRAM 被垃圾覆盖），它将硬重置 SOC。

<!--more-->

## 任务看门狗定时器

负责检测运行的任务在长时间没有让出 CPU 的情况。这是 CPU "饥饿"的症状，通常是由一个高优先级任务不让出 CPU 资源的循环引起，从而使较低优先级任务无法获得 CPU 资源。这可能是外围设备上的代码写得不好，也可能是陷入无限循环的任务。

默认情况下，TWDT 将监视每个 CPU 的空闲任务，但任何任务都可以选择由 TWDT 监视。每个观察任务必须定期“重置” TWDT 以指示它们已被分配 CPU 时间。如果任务未在 TWDT 超时期限内重置，则将打印一条警告，其中包含有关哪些任务未能及时重置 TWDT 以及哪些任务当前正在 ESP32 CPU 上运行的信息。并且还有可能在用户代码中重新定义函数 `esp_task_wdt_isr_user_handler` 以接收此事件。

TWDT 围绕定时器组 0 中的硬件看门狗定时器构建。可以通过调用 `esp_task_wdt_init（）` 来初始化 TWDT，这将配置硬件定时器。然后，任务可以使用 `esp_task_wdt_add（）` 订阅 TWDT 监视。每个订阅的任务必须定期调用 `esp_task_wdt_reset（）` 来重置 TWDT。任何订阅任务无法定期调用 `esp_task_wdt_reset（）` 表示一个或多个任务已经缺乏 CPU 资源或者陷入某个循环。

可以使用 `esp_task_wdt_delete（）` 从 TWDT 取消订阅监视的任务。已取消订阅的任务不应再调用 `esp_task_wdt_reset（）`。一旦所有任务都从 TWDT 取消订阅，可以通过调用 `esp_task_wdt_deinit（）` 来取消初始化 TWDT。

默认情况下，make menuconfig 中的 CONFIG_TASK_WDT 将被启用，导致 TWDT 在启动期间自动初始化。同样，CONFIG_TASK_WDT_CHECK_IDLE_TASK_CPU0 和 CONFIG_TASK_WDT_CHECK_IDLE_TASK_CPU1 也会默认启用，导致两个空闲任务在启动期间订阅 TWDT。

## JTAG 和 看门狗

在使用 OpenOCD 进行调试时，每次到达断点时都会暂停 CPU。但是，如果看门狗定时器在遇到断点时继续运行，它们最终会触发复位，从而很难调试代码。 因此，OpenOCD 将在每个断点处禁用中断和任务看门狗的硬件定时器。 此外，OpenOCD 在离开断点时不会重新启用它们。 这意味着中断看门狗和任务看门狗功能将基本上被禁用。 当 ESP32 通过 JTAG 连接到 OpenOCD 时，不会产生任何看门狗警告或 panic。

## 参考资料

 - [Watchdogs](https://docs.espressif.com/projects/esp-idf/en/v3.2/api-reference/system/wdts.html)
