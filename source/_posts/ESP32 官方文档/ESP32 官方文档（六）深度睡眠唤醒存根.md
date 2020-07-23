---
title: ESP32 官方文档（六）深度睡眠唤醒存根
date: 2012-07-22 09:43:43
categories:
- ESP32 官方文档
tags:
- ESP32
---

# 深度睡眠唤醒存根

ESP32 支持在深度睡眠时运行“深度睡眠唤醒存根”。芯片唤醒后立即运行此功能 - 在任何正常初始化，引导加载程序或 ESP-IDF 代码运行之前。唤醒存根运行后，SoC 可以返回休眠状态或继续正常启动 ESP-IDF。

深度睡眠唤醒存根代码被加载到“RTC 快速存储器”中，它使用的任何数据也必须加载到 RTC 存储器中。RTC 存储区域在深度睡眠期间保持其内容。

<!--more-->

## 唤醒存根规则

必须仔细编写唤醒存根代码:

 - 由于 SoC 刚刚从睡眠状态中醒来，大多数外设都处于复位状态。SPI Flash 未映射。
 - 唤醒存根代码只能调用 ROM 中或加载到 RTC 快速存储器中实现的功能(见下文)。
 - 唤醒存根代码只能访问 RTC 存储器中加载的数据。所有其他 RAM 将无法使用并具有随机内容。唤醒存根可以使用其他 RAM 进行临时存储，但是当 SoC 重新进入休眠状态或启动 ESP-IDF 时,内容将被覆盖。
 - RTC 内存必须包含存根使用的任何只读数据(.rodata)。
 - 每当 SoC 重新启动时，RTC 存储器中的数据都会被初始化，除非从深度睡眠中唤醒。从深度睡眠中醒来时，保持睡眠前存在的值。
 - 唤醒存根代码是 esp-idf 应用程序的一部分。在 esp-idf 的正常运行期间，函数可以调用唤醒存根函数或访问 RTC 存储器。就好像这些是应用程序的常规部分。

## 实现存根

esp-idf 中的唤醒存根的函数 `esp_wake_deep_sleep()`。只要 SoC 从深度睡眠中唤醒，该函数就会运行。esp-idf 中提供了此函数的默认版本，但默认函数是弱链接的，因此如果您的应用程序包含名为 `esp_wake_deep_sleep()` 的函数，那么这将覆盖这个默认版本。

如果提供自定义唤醒存根，它首先要做的就是调用 `esp_default_wake_deep_sleep()`。

没有必要在您的应用程序中实现 `esp_wake_deep_sleep()` 以使用深度睡眠。只有你想在唤醒时立即有特殊行为才有必要。

如果要在运行时在不同的深度睡眠存根之间进行交换，也可以通过调用 `esp_set_deep_sleep_wake_stub()` 函数来执行此操作。如果仅使用默认的 `esp_wake_deep_sleep()` 函数，则不需要这样做。

所有这些函数都在 components/esp32 下的 `esp_deepsleep.h` 头文件中声明。

## 将代码加载到 RTC 内存中

唤醒存根代码必须驻留在 RTC 快速存储器中。这可以通过两种方式之一完成。

第一种方法是使用 `RTC_IRAM_ATTR` 属性将函数放入 RTC 内存:

```
void RTC_IRAM_ATTR esp_wake_deep_sleep(void) {
    esp_default_wake_deep_sleep();
    // Add additional functionality here
}
```

第二种方法是将函数放入名称以 `rtc_wake_stub` 开头的任何源文件中。文件名 `rtc_wake_stub *` 中其内容将通过链接器自动放入 `RTC` 存储器中。

对于非常简短的代码，或者对于要混合“普通”和 “RTC” 代码的源文件，第一种方法更简单。当你想为 `RTC` 内存编写更长的代码时，第二种方法更简单。

## 将数据加载到 RTC 内存中

存根代码使用的数据必须驻留在 `RTC` 慢速存储器中。ULP 也使用该存储器。

可以通过以下两种方式之一来指定此数据:

第一种方法是使用 `RTC_DATA_ATTR` 和 `RTC_RODATA_ATTR` 来指定应加载到 RTC 慢速内存中的任何数据(可写或只读，相应):

```
RTC_DATA_ATTR int wake_count;

void RTC_IRAM_ATTR esp_wake_deep_sleep(void) {
    esp_default_wake_deep_sleep();
    static RTC_RODATA_ATTR const char fmt_str[] = "Wake count %d\n";
    ets_printf(fmt_str, wake_count++);
}
```

可以通过名为 `CONFIG_ESP32_RTCDATA_IN_FAST_MEM` 的 menuconfig 选项配置放置此数据的 RTC 存储区。此选项允许为 ULP 程序保留较慢的存储区域，一旦启用，标记为 `RTC_DATA_ATTR` 和 `RTC_RODATA_ATTR` 的数据将被放置在 RTC 快速存储器段中，否则将转至 RTC 慢速存储器（默认选项）。此选项取决于 `CONFIG_FREERTOS_UNICORE`，因为 RTC 快速存储器只能由 PRO_CPU 访问。

类似的属性 `RTC_FAST_ATTR` 和 `RTC_SLOW_ATTR` 用于指定将数据分别强制放入 `RTC_FAST` 和 `RTC_SLOW` 存储器。`PRO_CPU` 仅允许对标记为 `RTC_FAST_ATTR` 的数据进行访问，并且用户有责任确保它。

不幸的是，以这种方式使用的任何字符串常量必须声明为数组并用 `RTC_RODATA_ATTR` 标记，如上例所示。

第二种方法是将数据放入名称以 `rtc_wake_stub` 开头的任何源文件中。

例如，`rtc_wake_stub_counter.c` 中的等效示例:

```
int wake_count;

void RTC_IRAM_ATTR esp_wake_deep_sleep(void) {
    esp_default_wake_deep_sleep();
    ets_printf("Wake count %d\n", wake_count++);
}
```

如果您需要使用字符串或编写其他更复杂的代码，第二种方法是更好的选择。

## 参考资料

 - [原文链接](https://docs.espressif.com/projects/esp-idf/en/latest/api-guides/deep-sleep-stub.html)
