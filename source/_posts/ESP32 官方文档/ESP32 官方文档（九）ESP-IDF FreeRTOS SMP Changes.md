---
title: ESP32 官方文档（九）ESP-IDF FreeRTOS SMP Changes
date: 2012-07-22 09:43:43
categories:
- ESP32 官方文档
tags:
- ESP32
---

# ESP-IDF FreeRTOS SMP Changes

## 概述 

vanilla FreeRTOS 是设计运行在单核上. 但 ESP32 是双核的,包含 Protocol CPU (称为 **CPU 0** 或**PRO_CPU**)和 Application CPU (称为 **CPU 1** 或 **APP_CPU**). 这两个核实际上是相同的,并且共享相同的内存. 这允许任务在两个核之间交替运行.

ESP-IDF FreeRTOS 是 vanilla FreeRTOS 的修改版本,支持对称多处理 (SMP). ESP-IDF FreeRTOS 基于 FreeRTOS v8.2.0 的 Xtensa 端口. 本指南概述了 vanilla FreeRTOS 和 ESP-IDF FreeRTOS 之间的主要区别. 可以通过 http://www.freertos.org/a00106.html 找到 vanilla FreeRTOS 的 API 参考.

有关 ESP-IDF FreeRTOS 独有功能的信息,请参阅 [ESP-IDF FreeRTOS 附加功能](https://docs.espressif.com/projects/esp-idf/en/latest/api-reference/system/freertos_additions.html).

[反向移植特性](#%E5%8F%8D%E5%90%91%E7%A7%BB%E6%A4%8D%E7%89%B9%E6%80%A7):虽然 ESP-IDF FreeRTOS 基于 FreeRTOS v8.2.0 的 Xtensa 端口,但许多 FreeRTOS v9.0.0 功能已被移植到 ESP-IDF.

[任务和任务创建](#%E4%BB%BB%E5%8A%A1%E5%92%8C%E4%BB%BB%E5%8A%A1%E5%88%9B%E5%BB%BA):使用 `xTaskCreatePinnedToCore()` 或 `xTaskCreateStaticPinnedToCore()` 在 ESP-IDF FreeRTOS 中创建任务.这两个函数的最后一个参数是 xCoreID.此参数指定任务运行在那个核上.  **PRO_CPU** 为 0, **APP_CPU** 为 1,或者 tskNO_AFFINITY 允许任务在两者上运行.

循环调度:ESP-IDF FreeRTOS 调度器将在 Ready 状态下具有相同优先级的多个任务之间实施循环调度时跳过任务.要避免此行为,请确保这些任务进入阻塞状态,或者分布在更广泛的优先级中.

挂起调度器:在 ESP-IDF 中挂起调度器 FreeRTOS 只会影响调用核上的调度器.换句话说,在 **PRO_CPU** 上调用 `vTaskSuspendAll()` 不会阻止 **APP_CPU** 进行调度,反之亦然.使用临界区或信号量代替同时访问保护.

[滴答中断同步](#%E6%BB%B4%E7%AD%94%E4%B8%AD%E6%96%AD%E5%90%8C%E6%AD%A5):**PRO_CPU** 和 **APP_CPU** 的滴答中断不同步. 不要期望使用 `vTaskDelay()` 或 `vTaskDelayUntil()` 作为在两个核之间同步任务执行的准确方法. 使用计数信号量,因为它们的上下文切换不会因抢占而与滴答中断相关联.

[临界区和禁用中断](#%E4%B8%B4%E7%95%8C%E5%8C%BA%E5%92%8C%E7%A6%81%E7%94%A8%E4%B8%AD%E6%96%AD):在 ESP-IDF FreeRTOS 中,临界区是使用互斥锁实现的.进入临界区涉及获取互斥锁,然后禁用调度器和调用核的中断.然而,另一个核不受影响.如果另一个核尝试使用相同的互斥锁,它将自旋直到调用核通过退出临界区释放互斥锁.

[浮点运算](#%E6%B5%AE%E7%82%B9%E8%BF%90%E7%AE%97):ESP32 支持单精度浮点运算 (`float`) 的硬件加速.然而,硬件加速的使用导致 ESP-IDF FreeRTOS 中的一些行为限制.因此,如果没有这样做,使用 float 的任务将自动固定到核.此外, float 不能用于中断服务程序.

[任务删除](#%E4%BB%BB%E5%8A%A1%E5%88%A0%E9%99%A4):任务删除行为已从 FreeRTOS v9.0.0 反向移植并修改为 SMP 兼容.调用 `vTaskDelete()` 时,将立即释放任务内存,以删除当前未运行且未固定到其他核的任务.否则,释放任务存储器仍将被委托给空闲任务.

[线程本地存储指针和删除回调](#%E7%BA%BF%E7%A8%8B%E6%9C%AC%E5%9C%B0%E5%AD%98%E5%82%A8%E6%8C%87%E9%92%88%E5%92%8C%E5%88%A0%E9%99%A4%E5%9B%9E%E8%B0%83):ESP-IDF FreeRTOS 已经反向移植了线程本地存储指针 (TLSP) 功能.但是,添加了删除回调的额外功能.在删除任务期间会自动调用删除回调,并用于释放 TLSP 指向的内存.调用 `vTaskSetThreadLocalStoragePointerAndDelCallback()` 来设置 TLSP 和删除回调.

[配置 ESP-IDF FreeRTOS](%E9%85%8D%E7%BD%AE-esp-idf-freertos):可以使用 `make meunconfig` 配置 ESP-IDF FreeRTOS 的几个方面,例如在 Unicore 模式下运行 ESP-IDF,或配置每个任务将具有的线程本地存储指针的数量.

## 反向移植特性

以下功能已从 FreeRTOS v9.0.0 反向移植到 ESP-IDF.

### 静态分配

此特性已从 FreeRTOS v9.0.0 反向移植到 ESP-IDF. 必须在 menuconfig 中启用 `CONFIG_SUPPORT_STATIC_ALLOCATION` 选项才能使静态分配功能可用. 启用后,可以调用以下函数...

 - `xTaskCreateStatic()` (查看下面的[反向移植记录](#%E5%8F%8D%E5%90%91%E7%A7%BB%E6%A4%8D%E8%AE%B0%E5%BD%95))
 - `xQueueCreateStatic`
 - `xSemaphoreCreateBinaryStatic`
 - `xSemaphoreCreateCountingStatic`
 - `xSemaphoreCreateMutexStatic`
 - `xSemaphoreCreateRecursiveMutexStatic`
 - `xTimerCreateStatic()` (查看下面的[反向移植记录](#%E5%8F%8D%E5%90%91%E7%A7%BB%E6%A4%8D%E8%AE%B0%E5%BD%95))
 - `xEventGroupCreateStatic()`

### 其他特性

 - `vTaskSetThreadLocalStoragePointer()`  (查看下面的[反向移植记录](#%E5%8F%8D%E5%90%91%E7%A7%BB%E6%A4%8D%E8%AE%B0%E5%BD%95))
 - `pvTaskGetThreadLocalStoragePointer()`  (查看下面的[反向移植记录](#%E5%8F%8D%E5%90%91%E7%A7%BB%E6%A4%8D%E8%AE%B0%E5%BD%95))
 - `vTimerSetTimerID()`
 - `xTimerGetPeriod()`
 - `xTimerGetExpiryTime()`
 - `pcQueueGetName()`
 - `uxSemaphoreGetCount`

### 反向移植记录

1. `xTaskCreateStatic()` 以与 `xTaskCreate()` 类似的方式与 SMP 兼容(请参阅[任务和任务创建](#%E4%BB%BB%E5%8A%A1%E5%92%8C%E4%BB%BB%E5%8A%A1%E5%88%9B%E5%BB%BA)). 因此也可以调用 `xTaskCreateStaticPinnedToCore()`.
2. 尽管 vanilla FreeRTOS 允许静态分配 Timer 功能的守护进程任务,但守护进程任务总是在 ESP-IDF 中动态分配. 因此,在 ESP-IDF FreeRTOS 中使用静态分配的计时器时,不需要定义 `vApplicationGetTimerTaskMemory`.
3. 在 ESP-IDF FreeRTOS 中修改了线程本地存储指针功能,以包括删除回调(请参阅[线程本地存储指针和删除回调](#%E7%BA%BF%E7%A8%8B%E6%9C%AC%E5%9C%B0%E5%AD%98%E5%82%A8%E6%8C%87%E9%92%88%E5%92%8C%E5%88%A0%E9%99%A4%E5%9B%9E%E8%B0%83)). 因此,也可以调用函数 `vTaskSetThreadLocalStoragePointerAndDelCallback()`.

## 任务和任务创建

ESP-IDF FreeRTOS 中的任务设计为在特定核上运行,因此通过将 `PinnedToCore` 附加到 vanilla FreeRTOS 中的任务创建功能的名称,ESP-IDF FreeRTOS 中添加了两个新的任务创建功能. `xTaskCreate()` 和 `xTaskCreateStatic()` 的 vanilla FreeRTOS 函数导致在 ESP-IDF FreeRTOS 中添加了 `xTaskCreatePinnedToCore()` 和 `xTaskCreateStaticPinnedToCore()` (参见[反向移植特性](#%E5%8F%8D%E5%90%91%E7%A7%BB%E6%A4%8D%E7%89%B9%E6%80%A7)).

有关更多详细信息,请参阅 [freertos/task.c](https://github.com/espressif/esp-idf/blob/a557e8c/components/freertos/task.c)

除了称为 `xCoreID` 的额外参数外,ESP-IDF FreeRTOS 任务创建功能几乎与它们的vanilla对应物相同. 此参数指定应在其上运行任务的核,并且可以是以下值之一.

 - `0` 将任务固定在 **PRO_CPU** 上运行
 - `1` 将任务固定在 **APP_CPU** 上运行
 - `tskNO_AFFINITY`允许在两个 CPU 上运行任务

例如,`xTaskCreatePinnedToCore(tsk_callback,“APP_CPU Task”,1000,NULL,10,NULL,1)` 创建优先级为 10 的任务,该堆栈大小为 1000 字节运行在 **APP_CPU** 上. 应该注意的是,vanilla FreeRTOS 中的 `uxStackDepth` 参数根据字数指定任务的堆栈深度,而 ESP-IDF FreeRTOS 以字节的形式指定堆栈深度.

请注意,vanilla FreeRTOS函数 `xTaskCreate()` 和 `xTaskCreateStatic()` 已在 ESP-IDF FreeRTOS 中定义为内联函数,它们分别使用 tskNO_AFFINITY 作为 xCoreID 值调用 `xTaskCreatePinnedToCore()` 和 `xTaskCreateStaticPinnedToCore()`.

ESP-IDF 中的每个任务控制块 (TCB) 将 `xCoreID` 存储为成员. 因此,当每个核调用调度器选择要运行的任务时,`xCoreID` 成员将允许调度器确定是否允许让任务在调用调度器的核上运行.

## 调度方式

vanilla FreeRTOS 在 `vTaskSwitchContext()` 函数中实现调度. 此函数负责从处于就绪状态的任务列表中选择要执行的最高优先级任务,称为就绪任务列表(将在下一节中介绍). 在 ESP-IDF FreeRTOS 中,每个核将独立调用 `vTaskSwitchContext()`以从两个核之间共享的就绪任务列表中选择要运行的任务. vanilla 和 ESP-IDF FreeRTOS 之间的调度行为存在一些差异,例如 循环调度调度,调度程序暂停和滴答中断同步的差异.

### 循环调度

鉴于 Ready 状态和优先级相同的多个任务,vanilla FreeRTOS 在每个任务之间实现循环调度. 这将导致每次调用调度程序时轮流运行这些任务(例如每个滴答中断). 另一方面,当循环调度具有相同优先级的多个 Ready 状态任务时,ESP-IDF FreeRTOS 调度器可以跳过任务.

循环调度期间跳过任务的问题源于在 FreeRTOS 中实现就绪任务列表的方式. 在 vanilla FreeRTOS 中,`pxReadyTasksList` 用于存储处于 Ready 状态的任务列表. 该列表实现为长度为 `configMAX_PRIORITIES` 的数组,其中数组的每个元素都是链表. 每个链表都是 `List_t` 类型,并包含处于 Ready 状态的相同优先级任务的 TCB. 下图说明了 `pxReadyTasksList` 结构.

![pxReadyTasksList](https://img-blog.csdn.net/20180902140458206?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
 <center> Illustration of FreeRTOS Ready Task List Data Structure</center>

每个链表还包含一个 `pxIndex`,它指向查询列表时返回的最后一个 TCB. 该索引允许 `vTaskSwitchContext()` 在 `pxIndex` 之后立即开始遍历 TCB 上的列表,从而在相同优先级的任务之间实现循环调度.

在 ESP-IDF FreeRTOS 中,就绪任务列表在核之间共享,因此 `pxReadyTasksList` 将包含固定到不同核的任务. 当核调用调度程序时,它能够查看列表中每个 TCB 的 `xCoreID` 成员,以确定是否允许在调用调度程序的核上运行该任务. ESP-IDF FreeRTOS `pxReadyTasksList` 如下图所示.

![Illustration of FreeRTOS Ready Task List Data Structure in ESP-IDF](https://img-blog.csdn.net/20180902140927284?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
 <center> Illustration of FreeRTOS Ready Task List Data Structure in ESP-IDF </center> 

因此,当 **PRO_CPU** 调用调度程序时,它只会将任务视为蓝色或紫色. 而当 **APP_CPU** 调用调度程序时,它只会考虑橙色或紫色的任务.

虽然每个 TCB 在 ESP-IDF FreeRTOS 中都有一个 `xCoreID`,但每个优先级的链表只有一个 `pxIndex`. 因此,当从特定核调用调度程序并遍历链接列表时,它将跳过固定到另一个核的所有 TCB,并将 `pxIndex` 指向所选任务. 如果另一个核接着调用调度程序,它将在 `pxIndex` 之后立即遍历从 TCB 开始的链表. 因此,**在当前调度程序调用中不会考虑从由其他核先前调用调度程序跳过的 TCB**. 下图说明了此问题.

![Illustration of pxIndex behavior in ESP-IDF FreeRTOS](https://img-blog.csdn.net/20180902141125129?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
 <center>Illustration of pxIndex behavior in ESP-IDF FreeRTOS </center>

参考上面的图示,假设优先级 9 是最高优先级,并且优先级9中的任何任务都不会被阻塞,因此将始终处于运行或就绪状态.

1. **PRO_CPU** 调用调度程序并选择要运行的任务 A,因此将 `pxIndex` 移动到指向任务 A.
2. **APP_CPU** 调用调度程序并在作为任务 B 的 `pxIndex` 之后开始遍历任务.但是没有选择任务 B 运行,因为它没有固定到 **APP_CPU** 因此它被跳过并且选择了任务 C.  `pxIndex` 现在指向任务 C.
3. **PRO_CPU** 调用调度程序并开始从任务 D 开始遍历.它跳过任务 D 并选择任务 E 运行并将 `pxIndex` 指向任务 E.请注意,任务 B 未被遍历,因为最后一次 **APP_CPU** 调用调度程序时它被跳过遍历清单.
4. 如果 **APP_CPU** 再次调用调度程序,则会发生与任务 D 相同的情况,因为 `pxIndex` 现在指向任务E

任务跳过问题的一个解决方案是确保每个任务都进入阻塞状态,以便从就绪任务列表中删除它们.另一种解决方案是跨多个优先级分配任务,以便不给予给定优先级多个固定到不同核的任务.

### 挂起调度器

在 vanilla FreeRTOS 中,通过 `vTaskSuspendAll()` 挂起调度程序将阻止 `vTaskSwitchContext` 从上下文切换调用,直到调度程序已使用 `xTaskResumeAll()` 恢复.但是仍然允许为 ISR 提供服务.因此,在恢复调度程序之前,将不会执行由当前正在运行的任务或 ISRS 导致的任务状态的任何更改. vanilla FreeRTOS 中的调度程序暂停是一种常见的保护方法,可以同时访问任务之间共享的数据,同时仍允许对 ISR 进行服务.

在 ESP-IDF FreeRTOS 中,`xTaskResumeAll()` 只会阻止调用 `vTaskSwitchContext()` 来切换调用挂起的核上下文.因此,如果 **PRO_CPU** 调用 `vTaskSuspendAll()`, **APP_CPU** 仍然可以切换上下文.如果数据在固定到不同核的任务之间共享,则调度程序暂停不是防止同时访问的有效方法.在保护 ESP-IDF FreeRTOS 中的共享资源时,请考虑使用关键部分(禁用中断)或信号量(不禁用中断).

通常,最好使用其他 RTOS 原语(如互斥信号量)来防止任务之间共享的数据,而不是 `vTaskSuspendAll()`.

### 滴答中断同步

在 ESP-IDF FreeRTOS 中,由于来自每个核的调度程序调用是独立的,因此在相同的滴答计数上取消阻塞的不同核上的任务可能不会在完全相同的时间运行,并且每个核的滴答中断是不同步的.

在 vanilla FreeRTOS 中,滴答中断触发对 `xTaskIncrementTick()` 的调用,该调用负责增加滴答计数器,检查调用 `vTaskDelay()` 的任务是否已经完成延迟时间,并将这些任务从 Delayed Task List 移动​​到 Ready Task 名单.如果需要上下文切换,则滴答中断将调用调度程序.

在 ESP-IDF FreeRTOS 中,由于 **PRO_CPU** 负责增加共享滴答计数,因此参考 **PRO_CPU** 上的滴答中断来解除延迟任务的阻塞.但是,每个核的滴答中断可能不会同步(频率相同但异相),因此当 **PRO_CPU** 收到滴答中断时, **APP_CPU** 可能尚未收到它.因此,如果相同优先级的多个任务在相同的滴答计数上被解除阻塞,则固定到 **PRO_CPU** 的任务将立即运行,而固定到 **APP_CPU** 的任务必须等到 **APP_CPU** 收到其不同步滴答中断.收到滴答中断后, **APP_CPU** 将调用上下文切换,最后将上下文切换到新解锁的任务.

因此,不应将任务延迟用作 ESP-IDF FreeRTOS 中任务之间的同步方法.相反,请考虑使用计数信号量同时取消阻止多个任务.

## 临界区和禁用中断

Vanilla FreeRTOS 在 `vTaskEnterCritical` 中实现了关键部分,它们禁用调度程序并调用 `portDISABLE_INTERRUPTS`.这可以防止在关键部分中进行上下文切换和ISR服务.因此,关键部分被用作防止 vanilla FreeRTOS 同时访问的有效保护方法.

另一方面, ESP32 没有内核的硬件方法来禁用彼此的中断.调用 `portDISABLE_INTERRUPTS()` 对其他内核的中断没有影响.因此,禁用中断不是防止同时访问共享数据的有效保护方法,因为即使当前内核已禁用其自身的中断,它也会使其他内核可以自由访问数据.

因此, ESP-IDF FreeRTOS 使用互斥锁实现关键部分,进入或退出关键部分的调用必须提供与需要访问保护的共享资源相关联的互斥锁.当进入 ESP-IDF FreeRTOS 中的关键部分时,调用内核将禁用其调度程序和中断,类似于 vanilla FreeRTOS 实现.但是,调用核也将使用互斥锁,而另一个核在关键部分不受影响.如果另一个核尝试使用相同的互斥锁,它将旋转直到释放互斥锁.因此,关键部分的 ESP-IDF FreeRTOS 实现允许核具有对共享资源的受保护访问,而不会禁用其他核.另一个核只有在尝试同时访问同一资源时才会受到影响.

ESP-IDF FreeRTOS 关键部分功能已经修改如下......

  - `taskENTER_CRITICAL(mux)`,`taskENTER_CRITICAL_ISR(mux)`,`portENTER_CRITICAL(mux)`,`portENTER_CRITICAL_ISR(mux)` 都是宏定义来调用 `vTaskEnterCritical()`
  - `taskEXIT_CRITICAL(mux)`,`taskEXIT_CRITICAL_ISR(mux)`,`portEXIT_CRITICAL(mux)`,`portEXIT_CRITICAL_ISR(mux)` 都是宏定义来调用 `vTaskExitCritical()`

有关更多详细信息,请参阅 [freertos/include/freertos/portmacro.h](https://github.com/espressif/esp-idf/blob/a557e8c/components/freertos/include/freertos/portmacro.h) 和 [freertos/task.c](https://github.com/espressif/esp-idf/blob/a557e8c/components/freertos/task.c)

应该注意的是,当修改 vanilla FreeRTOS 代码与 ESP-IDF FreeRTOS 兼容时,修改关键部分的类型是微不足道的,因为它们都被定义为调用相同的函数.只要在进入和退出时提供相同的互斥锁,呼叫类型就无关紧要了.

## 浮点运算

ESP32 通过连接到每个内核的浮点单元 (FPU,也称为协处理器)支持单精度浮点运算 (`float`) 的硬件加速.使用 FPU 对 ESP-IDF FreeRTOS 施加了一些行为限制.

ESP-IDF FreeRTOS 为 FPU 实现了延迟上下文切换.换句话说,当发生上下文切换时,不会立即保存核心 FPU 寄存器的状态.因此,利用 `float` 的任务必须在创建时固定到特定的核心.如果没有, ESP-IDF FreeRTOS 会自动将有问题的任务固定到任务首次使用 `float` 任务的任何核心上.同样由于惰性上下文切换,中断服务例程也必须不使用 `float`.

ESP32 不支持双精度浮点运算 (`double`) 的硬件加速.相反, `double` 是通过软件实现的,因此关于 `float` 的行为限制不适用于 `double`.请注意,由于缺少硬件加速,与 `float` 相比,双重操作可能会消耗更多的 CPU 时间.

## 任务删除

在 v9.0.0 之前删除 FreeRTOS 任务将任务内存的释放完全委托给空闲任务. 目前,如果正在删除的任务当前没有运行或没有固定到另一个核心(相对于核心 `vTaskDelete()` 被调用),任务内存的释放将立即发生(在 `vTaskDelete()` 内). 如果满足相同的条件,TLSP删除回调也将立即运行.

但是,调用 `vTaskDelete()` 来删除当前正在运行或固定到另一个核心的任务仍将导致释放被委派给空闲任务的内存.

## 线程本地存储指针和删除回调

线程本地存储指针 (TLSP) 是直接存储在 TCB 中的指针. TLSP 允许每个任务拥有自己唯一的数据结构指针集.但是, vanilla FreeRTOS 中的任务删除行为不会自动释放 TLSP 指向的内存.因此,如果在删除任务之前用户未明确释放 TLSP 指向的内存,则会发生内存泄漏.

ESP-IDF FreeRTOS 提供了 Deletion Callbacks 的附加功能.删除任务删除期间自动调用回调以释放 TLSP 指向的内存.每个 TLSP 都可以拥有自己的 Deletion Callback.请注意,由于 [Task Deletion](https://docs.espressif.com/projects/esp-idf/en/latest/api-guides/freertos-smp.html#task-deletion) 行为,可能存在在空闲任务的上下文中调用 Deletion Callbacks 的实例.因此,删除回调不应该试图阻止,并且关键部分应该尽可能短,以最小化优先级倒置.

删除回调的类型为 `void(* TlsDeleteCallbackFunction_t)(int,void *)`,其中第一个参数是关联 TLSP 的索引号,第二个参数是 TLSP 本身.

通过调用 `vTaskSetThreadLocalStoragePointerAndDelCallback()` 将删除回调与 TLSP 一起设置.调用 vanilla FreeRTOS 函数 `vTaskSetThreadLocalStoragePointer()` 只会将 TLSP 关联的 Deletion Callback 设置为 NULL,这意味着在删除任务期间不会为该 TLSP 调用回调.如果删除回调为 NULL,则用户应在删除任务之前手动释放相关 TLSP 指向的内存,以避免内存泄漏.

menuconfig 中的 `CONFIG_FREERTOS_THREAD_LOCAL_STORAGE_POINTERS` 可用于配置TCB将具有的 TLSP 和删除回调数.

有关更多详细信息,请参阅 [FreeRTOS API 参考](https://docs.espressif.com/projects/esp-idf/en/latest/api-reference/system/freertos.html).

## 配置 ESP-IDF FreeRTOS

可以使用 `Component_Config/FreeRTOS` 下的 `make menuconfig` 配置 ESP-IDF FreeRTOS. 以下部分重点介绍了一些 ESP-IDF FreeRTOS 配置选项.有关 ESP-IDF FreeRTOS 配置的完整列表,请参阅 [FreeRTOS](https://docs.espressif.com/projects/esp-idf/en/latest/api-reference/kconfig.html)

`CONFIG_FREERTOS_UNICORE` 将仅在 **PRO_CPU** 上运行 ESP-IDF FreeRTOS.请注意,这不等于运行 vanilla FreeRTOS.将修改 ESP-IDF 中多个组件的行为,例如 [esp32/cpu_start.c](https://github.com/espressif/esp-idf/blob/a557e8c/components/esp32/cpu_start.c).有关在单核上运行 ESP-IDF FreeRTOS 的效果的更多详细信息,请在 ESP-IDF 组件中搜索 `CONFIG_FREERTOS_UNICORE` 的出现.

`CONFIG_FREERTOS_THREAD_LOCAL_STORAGE_POINTERS` 将定义每个任务在 ESP-IDF FreeRTOS 中将具有的线程本地存储指针的数量.

`CONFIG_SUPPORT_STATIC_ALLOCATION` 将在 ESP-IDF FreeRTOS 中启用 `xTaskCreateStaticPinnedToCore()` 的反向移植功能

`CONFIG_FREERTOS_ASSERT_ON_UNTESTED_FUNCTION` 将触发 ESP-IDF FreeRTOS 中特定功能的暂停,这些功能尚未在 SMP 上下文中进行全面测试.

[原文链接](https://docs.espressif.com/projects/esp-idf/en/latest/api-guides/freertos-smp.html)
