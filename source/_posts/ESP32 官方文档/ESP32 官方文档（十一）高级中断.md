---
title: ESP32 官方文档（十一）高级中断
date: 2012-07-22 09:43:43
categories:
- ESP32 官方文档
tags:
- ESP32
---

# 高级中断

Xtensa 架构支持 32 个中断,分为 8 个级别,以及各种异常. 在 ESP32 上,中断复用器允许使用[中断分配器](https://docs.espressif.com/projects/esp-idf/en/latest/api-reference/system/intr_alloc.html)将大多数中断源路由到这些中断. 通常,中断将以 C 语言写入,但 ESP-IDF 也允许在汇编中写入高级中断,从而允许非常低的中断延迟.

## 中断级别

|级别|标识|备注|
|:---:|:--:|:--:|
|1|N/A|异常和0级中断. 由 ESP-IDF 处理|
|2-3|N/A|中级中断. 由 ESP-IDF 处理|
|4|xt_highint4|通常由 ESP-IDF 调试逻辑使用|
|5|xt_highint4|随意使用|
|NMI|xt_nmi|随意使用|
|dbg|xt_debugexception|调试异常. 被称为例如 BREAK 指令.|

使用这些标识是通过创建一个程序集文件(后缀 .S)并定义命名符号来完成的,如下所示：

```
    .section .iram1,"ax"
    .global     xt_highint5
    .type       xt_highint5,@function
    .align      4
xt_highint5:
    ... your code here
    rsr     a0, EXCSAVE_5
    rfi     5
```

有关实际示例,请参阅 `components/esp32/panic_highint_hdl.S` 文件; Panic 处理程序中断在那里实现.

## 注意

 - 不要从高级别中断调用 C 代码; 因为这些中断仍然在关键部分运行,这可能会导致崩溃. (恐慌处理程序中断会调用正常的C代码,但这没关系,因为之后无意返回正常的代码流.)
 - 确保汇编代码被链接.如果中断处理程序符号是代码中其余代码使用的唯一符号,则链接器将采用默认的ISR,而不是将程序集文件链接到最终项目. 要解决此问题,请在汇编文件中定义符号,如下所示：
	
	```
	     .global ld_include_my_isr_file
	ld_include_my_isr_file:
	```

(此符号在此处称为 `ld_include_my_isr_file`,但可以具有未在其他任何位置定义的任意名称.)然后,在 `component.mk` 中,将此文件作为未解析的符号添加到 ld 命令行参数：

```
COMPONENT_ADD_LDFLAGS := -u ld_include_my_isr_file
```

这应该导致链接器始终包含定义 `ld_include_my_isr_file` 的文件,从而导致始终链接 ISR.

 - 可以使用 `esp_intr_alloc` 和相关函数路由和处理高级中断. 但是,`esp_intr_alloc` 的处理程序和处理程序参数必须为 NULL.
 - 理论上,中等优先级中断也可以这种方式处理. 目前, ESP-IDF 不支持这一点.

[原文链接](https://docs.espressif.com/projects/esp-idf/en/latest/api-guides/hlinterrupts.html)
