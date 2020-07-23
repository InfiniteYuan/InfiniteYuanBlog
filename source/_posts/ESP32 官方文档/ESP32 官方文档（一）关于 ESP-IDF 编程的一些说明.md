---
title: ESP32 官方文档（一）关于 ESP-IDF 编程的一些说明
date: 2012-07-22 09:43:43
categories:
- ESP32 官方文档
tags:
- ESP32
---

# 关于 ESP-IDF 编程的一些说明

## 应用启动流程

本文档说明了在调用 ESP-IDF 应用程序的 `app_main` 函数之前发生的一些步骤.

启动过程如下：
 **1. 位于 ROM 中的第一阶段引导程序将第二阶段引导程序映像从 flash 0x1000 地址加载到 RAM( IRAM 和 DRAM ).
 2. 第二阶段引导程序从 flash 中加载分区表和主应用程序映像.主应用程序包含 RAM 段和通过 flash cache 映射的只读段.
 3. 主应用程序映像执行.此时,可以启动第二个 CPU 和 RTOS 调度器.**
以下各节将详细介绍此过程.

<!--more-->

## 第一阶段引导程序

**SoC 复位后,PRO CPU 会立即开始运行,执行复位向量代码,而 APP CPU 将保持复位状态.** 在启动过程中, PRO CPU 会进行所有有关的初始化.APP CPU 复位状态在应用程序启动代码的 `call_start_cpu0` 函数中被取消.复位向量代码位于 ESP32 芯片掩模 ROM 中的 0x40000400 地址,并且无法修改.
复位向量调用的启动代码通过检查 `GPIO_STRAP_REG` 寄存器的引导引脚状态来确定引导模式.根据复位原因,有以下情况：
 1. 从深度睡眠(`deep sleep`)模式复位：如果 `RTC_CNTL_STORE6_REG` 中的值非零,并且 `RTC_CNTL_STORE7_REG` 中的 RTC 存储器的 CRC 值有效,则使用 `RTC_CNTL_STORE6_REG` 作为入口地址并立即跳转到该地址.如果 `RTC_CNTL_STORE6_REG` 为零,或者 `RTC_CNTL_STORE7_REG` 包含无效的  CRC 值,或者通过 `RTC_CNTL_STORE6_REG` 调用的代码执行完毕,则继续启动,就像上电复位一样.注意：此时若要运行自定义的代码,需要提供深度睡眠存根机制.请参阅[深度睡眠](https://esp-idf.readthedocs.io/en/latest/api-guides/deep-sleep-stub.html)文档.
2.  对于上电复位,软件 SOC 复位和看门狗 SOC 复位：检查 `GPIO_STRAP_REG` 寄存器,是否要求 UART 或 SDIO 下载模式.如果是这种情况,配置 UART 或 SDIO,并等待下载代码.否则,继续启动,就好像是由于软件 CPU 复位.
3.  对于软件 CPU 复位和看门狗 CPU 复位：根据 EFUSE 值配置 SPI flash,并尝试从 flash 中加载代码.在下面的段落中会更详细地描述了该步骤.如果从 flash 中加载代码失败,解压 BASIC 解释器到 RAM 中并启动它.请注意,发生这种情况时 RTC 看门狗仍然是开启的,因此除非解释器接收到任何输入,否则看门狗将在几百毫秒内复位 SOC,重复整个过程.如果解释器收到来自 UART 的任何输入,它将禁用看门狗.

**应用程序二进制映像从 flash 的 0x1000 地址开始加载.** 闪存的第一个 4kB 扇区用于存储安全引导IV和应用程序映像的签名.有关详细信息,请查看安全启动文档.

## 第二阶段引导程序

在 ESP-IDF 中,位于 flash 0x1000 地址的二进制映像是第二阶段引导程序. 第二阶段引导程序源代码可在 ESP-IDF 的 `components/bootloader` 目录中找到.请注意,第二阶段引导程序这样的安排并不是 ESP32 芯片唯一可行的安排.可以编写一个功能齐全的应用程序,当 flash 偏移到 0x1000 地址时可以工作,但这超出了本文档的范围.ESP-IDF 中使用第二阶段引导程序来增加 flash 布局的灵活性(使用分区表),并允许与闪存加密,安全引导和无线更新(OTA)相关的操作执行.

**当第一阶段引导程序完成校验并加载第二阶段引导程序时,它会跳转到在二进制映像头中找到的第二阶段引导程序的入口地址.**

**第二阶段引导程序读取在 0x8000 地址的分区表.** 更多有关信息,请参阅[分区表](https://esp-idf.readthedocs.io/en/latest/api-guides/partition-tables.html)文档.**引导程序找到 factory 和 OTA 分区,并根据 OTA 信息分区中的数据决定引导哪个分区.**

对于所选分区,第二阶段引导程序将映射到 IRAM 和 DRAM 的数据和代码段拷贝到其加载地址.对于在 DROM 和 IROM 区域中具有加载地址的节,flash MMU 将提供正确的映射.请注意,第二阶段引导程序为 PRO 和 APP CPU 配置了 flash MMU,但它仅为 PRO CPU 启用 flash MMU.原因是第二阶段引导程序代码被加载到 APP CPU 缓存所使用的内存区域.为 APP CPU 启用缓存的任务将交给应用程序.加载代码并配置 flash MMU 后,第二阶段引导程序将跳转到二进制映像头中的应用程序入口地址.

目前,无法将应用程序定义的挂钩添加到引导程序以自定义应用程序分区的选择逻辑.例如,这可能需要根据 GPIO 的状态加载不同的应用程序映像.此类自定义功能将在未来添加到 ESP-IDF 中.目前,可以通过将 bootloader 组件拷贝到应用程序目录并在那里进行必要的更改来自定义引导程序.在这种情况下,ESP-IDF 构建系统将编译应用程序目录中的组件而不是 ESP-IDF 自身目录中的组件.

## 应用启动

ESP-IDF 应用程序入口地址是在 `components/esp32/cpu_start.c`中的`call_start_cpu0`函数.这个函数的两个主要功能是启用堆分配器并使 APP CPU 跳转到其入口地址`call_start_cpu1`.PRO CPU 上的代码设置 APP CPU 的入口地址、取消 APP CPU 复位,并等待全局标志被 APP CPU 上运行的代码设置,以表示 APP CPU 已经启动.这个函数执行后,PRO CPU 跳转到`start_cpu0`函数,APP CPU 跳转到`start_cpu1`函数.

`start_cpu0`和`start_cpu1`都是弱函数,这意味着如果需要对初始化序列进行一些特定于应用程序的更改,则可以在应用程序中覆盖它们.`start_cpu0`默认启用或初始化 `menuconfig` 中选择的组件.请参阅`components/esp32/cpu_start.c`中此函数的源代码,以获取最新的执行步骤.请注意,在此阶段将调用应用程序中存在的任何 C ++ 全局构造函数.初始化完所有必要组件后,将创建主任务并启动 FreeRTOS 调度器.

当 PRO CPU 在 `start_cpu0`函数中进行初始化时,APP CPU 将在`start_cpu1`函数中等待调度器在 PRO CPU 上启动.在 PRO CPU 上启动调器器后,APP CPU 上的代码也会开启调度器.

主任务是运行 `app_main`函数. 可以在 `menuconfig` 中配置主任务堆栈大小和优先级. 应用程序可以将此任务用于特定应用程序的初始化,例如启动其他任务. 应用程序还可以将主任务用于事件循环和其他通用活动. 如果 `app_main` 函数返回,主任务会被删除.

## 应用内存布局

ESP32 芯片具有灵活的内存映射功能.本节将介绍  ESP-IDF 在默认情况下是如何使用这些功能的.
ESP-IDF 中的应用程序代码可以放入以下内存区域之一.

### IRAM(指令RAM)

ESP-IDF 为指令 RAM 分配内部 SRAM0 区域的一部分(在技术参考手册中定义).除了第一个 64 kB 块用于 PRO 和 APP CPU 高速缓存之外,该存储器范围的其余部分(即从 `0x40080000` 到 `0x400A0000` )用于存储需要从 RAM 中运行的应用程序部分.

使用链接描述文件将 ESP-IDF 的一些组件和 WiFi 协议栈的一些部分放入该区域.

如果需要将一些应用程序代码放入 IRAM,可以使用 `IRAM_ATTR` 定义：
```
#include "esp_attr.h"
void IRAM_ATTR gpio_isr_handler(void* arg)
{
        // ...
}
```
以下是部分应用可能或应该放在 IRAM 中的情况.

 - 如果在注册中断处理程序时使用`ESP_INTR_FLAG_IRAM`,则必须将中断处理程序放在 IRAM 中.在这种情况下,ISR 可能只调用放在 IRAM 中的函数或存在于 ROM 中的函数.注意：所有 FreeRTOS API 目前都放在 IRAM 中,因此可以安全地从中断处理程序中调用.如果将 ISR 放在 IRAM 中,则必须使用 `DRAM_ATTR` 将 ISR 使用的所有常量数据和从 ISR 调用的函数(包括但不限于 `const char` 数组)放在 DRAM 中.
 - 可以将一些关键性的时序代码放在 IRAM 中以减少从闪存加载代码相关的损耗.ESP32 通过 32 kB 高速缓存从闪存中读取代码和数据.在某些情况下,将函数放入到 IRAM 中可以减少由高速缓存未命中引起的延迟.

### IROM(从Flash执行的代码)

如果函数未明确放入到 IRAM 或 RTC 内存中,则将其放置到闪存中.Flash 技术参考手册中介绍了 Flash MMU 用于允许代码从闪存执行的机制. ESP-IDF 从 0x400D0000 - 0x40400000 区域开始放置应该从闪存执行的代码.启动时,第二阶段引导加载程序初始化 Flash MMU 以将代码所在的 flash 中的位置映射到该区域的开头.使用 0x40070000 - 0x40080000 范围内的两个 32kB 块透明地缓存对该区域的访问.
请注意,使用 Window ABI CALLx 指令可能无法访问 0x40000000 - 0x40400000 区域外的代码,因此如果应用程序使用 0x40400000 - 0x40800000 或 0x40800000 - 0x40C00000 区域,需要特别小心. ESP-IDF 默认不使用这些区域.

### RTC 高速存储器

从深度睡眠模式唤醒后,将要运行的代码必须放入 RTC 存储器.请查看[深度睡眠](https://esp-idf.readthedocs.io/en/latest/api-guides/deep-sleep-stub.html)文档中的详细说明.

### DRAM (数据 RAM)

链接器将非常量静态数据和零初始化数据放入 256 kB 的 0x3FFB0000 - 0x3FFF0000 区域中.请注意,如果使用蓝牙堆栈,此区域将减少 64kB(通过将开始地址移至 0x3FFC0000).如果使用跟踪存储器,该区域的长度也会减少 16 kB 或 32kB.放置静态数据后,在此区域中的所有剩下的空间都将用于运行时堆.

常量数据也可以放入 DRAM 中,例如,如果它用在 ISR 中(参见上面 IRAM 部分的注释).为此,可以使用 DRAM_ATTR 定义：
```
DRAM_ATTR const char[] format_string = "%p %x";
char buffer[64];
sprintf(buffer, format_string, ptr, val);
```
不必说,不建议在 ISR 中使用 printf 和其他输出功能.为了进行调试,从中断服务程序打印log时使用 `ESP_EARLY_LOGx` 宏.在这种情况下,必须确保将 TAG 和格式字符串都放入到 DRAM 中.

`__NOINIT_ATTR`宏可以用作将数据放入`.noinit`部分的属性.放入此部分的值不会在启动时初始化,并在软件重新启动后保留其值.
例如：
```
__NOINIT_ATTR uint32_t noinit_data;
```

### DROM(存储在 Flash 中的数据)

默认情况下,链接器将常量数据放入到一个 4 MB 的区域中(0x3F400000 - 0x3F800000),该区域被用于通过 Flash MMU 和缓存访问外部闪存.但是对于文字常量不同,它们由编译器嵌入到应用程序代码中.

### RTC 低速存储器

从 RTC 存储器中运行的代码使用的全局和静态变量(即深度睡眠存根代码)必须放入到 RTC 慢速存储器中. 请查看[深度睡眠](https://esp-idf.readthedocs.io/en/latest/api-guides/deep-sleep-stub.html)文档中的详细说明.

`RTC_NOINIT_ATTR`宏可用于将数据放入到这种类型的内存中.放在此部分中的数据,在从深度睡眠中醒来后也会保持其值.
例如：

```
RTC_NOINIT_ATTR uint32_t rtc_noinit_data;
```

### DMA 能力要求

大多数 DMA 控制器(例如 SPI,sdmmc 等)都要求发送/接收缓冲区应放在 DRAM 中并进行字对齐.我们建议将 DMA 缓冲区放在静态变量中而不是堆栈中.使用`DMA_ATTR`宏声明全局/本地静态变量,如：

```
DMA_ATTR uint8_t buffer[]="I want to send something";

void app_main()
{
    // initialization code...
    spi_transaction_t temp = {
        .tx_buffer = buffer,
        .length = 8*sizeof(buffer),
    };
    spi_device_transmit( spi, &temp );
    // other stuff
}
```

或者：

```
void app_main()
{
    DMA_ATTR static uint8_t buffer[]="I want to send something";
    // initialization code...
    spi_transaction_t temp = {
        .tx_buffer = buffer,
        .length = 8*sizeof(buffer),
    };
    spi_device_transmit( spi, &temp );
    // other stuff
}
```

在堆栈中放置DMA缓冲区仍然是允许的,但必须记住：
 1. 如果堆栈在 pSRAM 中,切勿尝试这样做.如果任务的堆栈放在 pSRAM 中,则必须执行[支持外部 RAM ](https://esp-idf.readthedocs.io/en/latest/api-guides/external-ram.html)文档(至少在 menuconfig 中启用 `SPIRAM_ALLOW_STACK_EXTERNAL_MEMORY` 选项)中所述的几个步骤.确保你的任务不在 pSRAM 中.
 2. 在将变量放在适当的位置之前的函数中使用 `WORD_ALIGNED_ATTR` 宏,如：

```
void app_main()
{
    uint8_t stuff;
    WORD_ALIGNED_ATTR uint8_t buffer[]="I want to send something";   //or the buffer will be placed right after stuff.
    // initialization code...
    spi_transaction_t temp = {
        .tx_buffer = buffer,
        .length = 8*sizeof(buffer),
    };
    spi_device_transmit( spi, &temp );
    // other stuff
}
```

## 参考资料

 - [原文链接](https://esp-idf.readthedocs.io/en/latest/api-guides/general-notes.html)
