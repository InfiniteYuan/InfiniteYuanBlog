---
title: ESP32 学习笔记（二十九） ESP32 低功耗模式
date: 2020-05-31 09:59:38
categories:
- ESP32 学习笔记
tags:
- ESP32
---

# ESP32 低功耗模式

## 1 概要

ESP32 系列芯片提供三种可配置的睡眠模式，针对这些睡眠模式,我们提供了了多种低功耗解决方案，用户可以结合具体需求选择睡眠模式并进行配置。三种睡眠模式如下:

 - Modem-sleep 模式：CPU 可运行，时钟可被配置。Wi-Fi/蓝牙基带和射频关闭。
 - Light-sleep 模式：CPU 暂停运行，Wi-Fi/蓝牙基带和射频关闭。**RTC 存储器和外设以及 ULP 协处理器运行**。任何唤醒事件(MAC、主机、RTC 定时器或外部中断)都会唤醒芯片。
 - Deep-sleep 模式：CPU 和大部分外设都会掉电，Wi-Fi/蓝牙基带和射频关闭，只有 **RTC 存储器和 RTC 外设以及 ULP 协处理器**可以工作。Wi-Fi 和蓝牙连接数据存储在 RTC 中。

三种模式的区别如下：

<!--more-->

![三种模式的区别][1]

---

## 2 Modem-sleep 模式

目前 ESP32 的 Modem-sleep 仅工作在 Station 模式下,连接路由器后生效。Station 会周期性在工作状态和睡眠状态两者之间切换。
ESP32 通过 Wi-Fi 的 DTIM Beacon 机制与路由器保持连接。在 Modem-sleep 模式下，系统可以自动被唤醒，无需配置唤醒源。

> 一般路由器的 DTIM Beacon 间隔为 100 ms ~ 1,000 ms。

> DTIM (Delivery Traffic Indication Message): 使用无线路由器时无线发送数据包的频率。

在 Modem-sleep 模式下，ESP32 会在两次 DTIM Beacon 间隔时间内，关闭 Wi-Fi 模块电路，达到省电效果，在下次 Beacon 到来前自动唤醒。睡眠时间由路由器的 DTIM Beacon 时间决定。Modem-sleep 模式可以保持与路由器的 Wi-Fi 连接，并通过路由器接收来自手机或者服务器的交互信息。

### 2.1 API 说明

通过以下接口配置 Modem-sleep 模式，`type` 可选参数：

 - `WIFI_PS_NONE`: 不使用 Modem-sleep 模式
 - `WIFI_PS_MIN_MODEM`: ESP32 接收 Beacon 的间隔与路由器的 DTIM 间隔相同，即 1 个路由器间隔
 - `WIFI_PS_MAX_MODEM`: ESP32 接收 Beacon 的间隔可由程序进行配置，间隔周期 `wifi_sta_config_t` 结构体中 `listen_interval` 值决定，单位为 路由器的 Beacon 间隔，默认值为 3（即 3 个路由器 Beacon 间隔）

```c
typedef enum {
    WIFI_PS_NONE,        /**< No power save */
    WIFI_PS_MIN_MODEM,   /**< Minimum modem power saving. In this mode, station wakes up to receive beacon every DTIM period */
    WIFI_PS_MAX_MODEM,   /**< Maximum modem power saving. In this mode, interval to receive beacons is determined by the listen_interval parameter in wifi_sta_config_t */
} wifi_ps_type_t;
esp_err_t esp_wifi_set_ps(wifi_ps_type_t type);
```

1. `type` 参数为 `WIFI_PS_MAX_MODEM` ，ESP32 接收 Beacon 的间隔 `listen_interval` 配置方法：

```c
#define LISTEN_INTERVAL 3
wifi_config_t wifi_config = {
    .sta = {
        .ssid = "SSID",
        .password = "Password",
        .listen_interval = LISTEN_INTERVAL,
    },
};
ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
ESP_ERROR_CHECK(esp_wifi_set_config(ESP_IF_WIFI_STA, &wifi_config));
ESP_ERROR_CHECK(esp_wifi_start());
```

### 2.2 应用

Modem-sleep 一般用于 CPU 持续处于工作状态并需要保持 Wi-Fi 连接的应用场景，例如，使用 ESP32 本地语音唤醒功能，CPU 需要持续采集和处理音频数据。

---

## 3 Light-sleep 模式

Light-sleep 的工作模式与 Modem-sleep 相似，不同的是，除了关闭 Wi-Fi 模块电路以外，在 Light-sleep 模式下，还会关闭时钟并暂停内部 CPU，比 Modem-sleep 功耗更低。有两种方式使 CPU 进入 Light-sleep 模式：

 - 强制 Light-sleep： 通过调用 API 强制 CPU 进入 Light-sleep 模式，强制进入 Light-sleep 模式后，**不能通过路由器接收**来自手机或者服务器的交互信息
 - 自动 Light-sleep： 配置为自动休眠方式后，会在 CPU 处于空闲的状态下自动进入 Light-sleep 模式，**能通过路由器接收**来自手机或者服务器的交互信息

### 3.1 强制 Light-sleep

参考示例: [light_sleep](https://github.com/espressif/esp-idf/tree/release/v3.2/examples/system/light_sleep)

开发者可以使系统强制进入 Light-sleep 模式，即调用强制 Light-sleep 接口，强制关闭 Wi-Fi 模块电路并暂停内部 CPU。强制休眠后，能通过定时器、 GPIO（RTC IO 和 Digital IO）和 UART 唤醒。从 Light-sleep 唤醒后，会从进入休眠的位置继续执行程序。强制进入 Light-sleep 模式后，可以保持与路由器的 Wi-Fi 连接，但不能通过路由器接收来自手机或者服务器的交互信息。

> 注意：

> 1. 强制 Light-sleep 接口调用后，并不会立即休眠，而是等到系统空闲后才进入休眠，

> 2. 强制进入 Light-sleep 模式后，不能接收 IP 网络发送的网络数据。

#### 3.1.1 API 说明

1. 配置唤醒源后，可使用 `esp_light_sleep_start()`  函数进入 Light-sleep 模式。在没有配置唤醒源的情况下也可以进入轻度睡眠状态，在这种情况下，芯片将无限期地处于 Light-sleep 模式，直到外部复位。

```c
/* Enter sleep mode */
esp_light_sleep_start();
/* Execution continues here after wakeup */
```

#### 3.1.2 应用

强制 Light-sleep 模式可用于需要保持与路由器的连接，不需要实时响应路由器发来的数据的场景。

### 3.2 自动 Light-sleep

参考示例: [power_save](https://github.com/espressif/esp-idf/tree/release/v3.2/examples/wifi/power_save)

开发者可配置系统在空闲时自动进入 Light-sleep 模式，并能在需要 CPU 工作时自动唤醒，无需配置唤醒源。在配置为自动 Light-sleep 后，可以保持与路由器的 Wi-Fi 连接，并通过路由器接收来自手机或者服务器的交互信息，对用户体验没有影响。
通常自动 Light-sleep 会与 Modem-sleep 模式 以及电源管理功能共同使用，电源管理功能允许系统根据 CPU 负载动态调节 CPU 频率以降低功耗。

> 若系统应用中有小于 DTIM Beacon 间隔时间的循环定时,系统将不能进入 Light-sleep 模式。

![此处输入图片的描述][2]

#### 3.2.1 API 说明

1. 通过 `esp_err_t esp_pm_configure(const void* vconfig)` 接口配置电源管理功能，参数 `light_sleep_enable` 为 true 时使能自动休眠功能。

> 使能 自动 Light-sleep 功能，需要使能 [CONFIG_FREERTOS_USE_TICKLESS_IDLE](https://docs.espressif.com/projects/esp-idf/en/v3.2/api-reference/kconfig.html#config-freertos-use-tickless-idle) 和 [CONFIG_PM_ENABLE](https://docs.espressif.com/projects/esp-idf/en/v3.2/api-reference/kconfig.html#config-pm-enable)

```c
#if CONFIG_PM_ENABLE
    // Configure dynamic frequency scaling:
    // maximum and minimum frequencies are set in sdkconfig,
    // automatic light sleep is enabled if tickless idle support is enabled.
    esp_pm_config_esp32_t pm_config = {
            .max_freq_mhz = CONFIG_EXAMPLE_MAX_CPU_FREQ_MHZ,
            .min_freq_mhz = CONFIG_EXAMPLE_MIN_CPU_FREQ_MHZ,
#if CONFIG_FREERTOS_USE_TICKLESS_IDLE
            .light_sleep_enable = true
#endif
    };
    ESP_ERROR_CHECK( esp_pm_configure(&pm_config) );
#endif // CONFIG_PM_ENABLE
```

#### 3.2.2 应用

自动 Light-sleep 模式可用于需要保持与路由器的连接，可以实时响应路由器发来的数据的场景。并且在未接收到命令时，CPU 可以处于空闲状态。比如 Wi-Fi 开关的应用，大部分时间 CPU 都是空闲的，直到收到控制命令，CPU 才需要进行 GPIO 的操作。

---

## 3.3 Deep-sleep 模式

参考示例: [deep_sleep](https://github.com/espressif/esp-idf/tree/release/v3.2/examples/system/deep_sleep)

相对于其他两种模式，系统无法自动进入 Deep-sleep，需要由用户调用接口函数 `esp_deep_sleep_start()` 进入 Deep-sleep 模式。在该模式下，芯片会断开所有 Wi-Fi 连接与数据连接，进入 Deep-sleep 模式，只有 **RTC 存储器和 RTC 外设以及 ULP 协处理器**可以工作。从 Deep-sleep 唤醒后，CPU 将软件复位重启。

### 3.3.1 API 说明

1. 配置唤醒源后，可使用 `esp_deep_sleep_start()` 函数进入 Deep-sleep 模式。在没有配置唤醒源的情况下也可以进入 Deep-sleep 状态，在这种情况下，芯片将无限期地处于 Deep-sleep 模式，直到外部复位。

```c
/* Enter deep sleep */
esp_deep_sleep_start();
```

### 3.3.2 应用

Deep-sleep 可以用于低功耗的传感器应用,或者大部分时间都不需要进行数据传输的情况。设备可以每隔一段时间从 Deep-sleep 状态醒来测量数据并上传,之后继续进入 Deep-sleep。也可以将多个数据存储于 RTC memory(RTC memory 在 Deep-sleep 模式下仍然可以保存数据),然后一次发送出去。

---

## 4 唤醒方式

针对 **强制 Lighe-sleep** 和 **Deep-sleep** 需要配置唤醒源的情况，有以下唤醒方式可以选择：

> 部分唤醒方式仅支持 Light-sleep

### 4.1 GPIO

 1. External wakeup (ext0)
    
    RTC IO 模块包含当其中 **一个 RTC IO** 的电平为唤醒电平（可配置为逻辑高或低）触发唤醒的逻辑。

    RTC IO 是 RTC 外设电源域的一部分，因此如果使能该唤醒源，RTC 外设将在睡眠期间保持上电状态。在 ESP32 的修订版 0 和 1 中，此唤醒源与 ULP 和触摸唤醒源不兼容。
    
    由于在此模式下启用了 RTC IO 模块，因此也可以使用内部上拉或下拉电阻。
    
    `esp_sleep_enable_ext0_wakeup()` 函数可用于启用此唤醒源。

 2. External wakeup (ext1)
    
    RTC 控制器包含使用 **多个 RTC IO** 触发唤醒的逻辑。两个逻辑功能之一可用于触发唤醒：
    - 如果任何一个所选 IO 为高电平，则唤醒（`ESP_EXT1_WAKEUP_ANY_HIGH`）
    - 如果所有选定的 IO 都为低电平，则唤醒（`ESP_EXT1_WAKEUP_ALL_LOW`）
    
    该唤醒源由 RTC 控制器实现。因此，RTC 外设和 RTC 存储器可以在此模式下断电。

    但是，如果 RTC 外设断电，内部上拉和下拉电阻将被禁用。要使用内部上拉或下拉电阻，请在睡眠期间请求 RTC 外设电源域保持上电。
    
    `esp_sleep_enable_ext1_wakeup()` 函数可用于启用此唤醒源。

 3. Touch pad wakeup
    
    RTC IO 模块包含触摸传感器中断时触发唤醒的逻辑。您需要在芯片开始睡眠之前配置触摸传感器中断。
    
    当 RTC 外设未被强制上电时，ESP32 的修订版 0 和 1 仅支持此唤醒模式。

    `esp_sleep_enable_touchpad_wakeup()` 函数可用于启用此唤醒源。

 4. GPIO wakeup **(light sleep only)**
    
    除了上面描述的 EXT0 和 EXT1 唤醒源之外，在 Light-sleep 模式下还有一种从外部输入唤醒的方法。通过该唤醒源，每个引脚可以使用 `gpio_wakeup_enable()` 函数单独配置为高电平或低电平唤醒。与 EXT0 和 EXT1 唤醒源（只能与 RTC IO 一起使用）不同，此唤醒源可用于任何 IO（RTC 或数字）。
    
    `esp_sleep_enable_gpio_wakeup()` 函数可用于启用此唤醒源。

### 4.2 ULP 协处理器唤醒

参考示例: [system/ulp](https://github.com/espressif/esp-idf/tree/release/v3.2/examples/system/ulp)

ULP 协处理器可以在芯片处于睡眠模式时运行，并且可以用于轮询传感器，监视 ADC 或触摸传感器值，并在检测到特定事件时唤醒芯片。ULP 协处理器是 RTC 外设电源域的一部分，它运行存储在 RTC 慢速存储器中的程序。如果请求此唤醒模式，RTC 慢速存储器将在睡眠期间启动。在 ULP 协处理器开始运行程序之前，RTC 外设将自动上电; 程序停止运行后，RTC 外设将再次自动关闭。

当 RTC 外设未被强制上电时，ESP32 的修订版 0 和 1 仅支持此唤醒模式。

`esp_sleep_enable_ulp_wakeup()` 函数可用于启用此唤醒源。

### 4.3 Timer

RTC 控制器具有内置定时器，可用于在预定义的时间后唤醒芯片。时间以微秒精度指定，但实际分辨率取决于为 RTC SLOW_CLK 选择的时钟源。

此唤醒模式不需要在睡眠期间打开 RTC 外设或 RTC 存储器。

`esp_sleep_enable_timer_wakeup()` 函数可用于启用此唤醒源。

### 4.4 UART wakeup (light sleep only)

仅支持  Light-sleep 模式。当 ESP32 从外部设备接收 UART 输入时，通常需要在输入数据可用时唤醒芯片。UART 外设包含一项功能，当看到 RX 引脚上的一定数量的上升沿时，可以将芯片从 Light-sleep 状态唤醒。可以使用 `uart_set_wakeup_threshold()` 函数设置此上升沿数。请注意，唤醒后 UART 不会接收触发唤醒的字符（及其前面的任何字符）。这意味着外部设备通常需要在发送数据之前向 ESP32 发送额外字符以触发唤醒。

`esp_sleep_enable_uart_wakeup()` 函数可用于启用此唤醒源。

## 5 GPIO Hold 功能

每个 IO pad(包括 RTC pad)都有单独的 hold 功能，由 RTC 寄存器控制。pad 的 hold 功能被置上后，pad 在置上 hold 那一刻的状态被强制保持，无论内部信号如何变化，修改 IO_MUX 配置或者 GPIO 配置，都不会改变 pad 的状态。应用如果希望在看门狗超时触发内核复位和系统复位时或者 Deep-sleep 时 pad 的状态不被改变，就需要提前把 hold 置上。

> 在 Deep-sleep 模式下使用 Hold 功能，可能会增加功耗。并且如果使用内部上拉或下拉电阻，需要在睡眠期间使 RTC 外设电源域保持上电。

> 在睡眠模式下，部分引脚的电源域可能关掉了，所以无法 Hold 功能无法生效，这时需要确认能否打开电源域。例如：GPIO16、17 会在 light sleep 下关闭电源域 VDD_SDIO，所以无法 Hold，这时需要在 esp_light_sleep_start() 中修改代码打开电源域。

针对 RTC IO，可对**单个 RTC Pad 使能 Hold 功能**:

```c
rtc_gpio_init(27);
rtc_gpio_set_direction(27, RTC_GPIO_MODE_OUTPUT_ONLY);
rtc_gpio_set_level(27, 1);
rtc_gpio_pullup_en(27);
rtc_gpio_hold_en(27);
```

针对 数字 IO，会对**所有 数字 IO 使能 Hold 功能**:

```c
gpio_pad_select_gpio(26);
gpio_set_direction(26, GPIO_MODE_OUTPUT);
gpio_set_level(26, 1);

gpio_hold_en(26);
/*When the chip is in Deep-sleep mode, all digital gpio will hold the state before sleep, and when the chip is woken up, the status of digital gpio will not be held.*/
gpio_deep_sleep_hold_en(); 
```

## 6 常见问题

1. 进入 Deep-sleep 后，功耗较高
   
    有这些原因可能会导致功耗较高：
 - `esp_sleep_enable_ext0_wakeup()` 需要打开 RTC 外设，产生额外的 100mA 左右的电流消耗，调用 `esp_sleep_enable_ext1_wakeup()` 则不需要打开 RTC 外设，即不会产生额外的电流。
 - 在 IDF 3.1.1 前的版本可能存在使用 wifi 并关闭后进入睡眠模式，功耗比未使用 wifi 后进入睡眠模式高 1mA 左右。这种情况下，需要在进入睡眠模式前调用 `adc_power_off()` 关闭 ADC 以解决这个问题。
 - 使用 Hold 功能保持一个固定电平，可能会增加功耗 1mA 左右，这种情况也是由于打开 RTC 外设导致。
 - 某些 GPIO 漏电，一些 ESP32 IO 具有内部上拉或下拉，默认情况下启用。如果外部电路在 Deep-sleep 模式下驱动此引脚，则由于流过这些上拉和下拉的电流，电流消耗可能会增加。要隔离引脚，防止额外的电流消耗，请调用 `rtc_gpio_isolate()` 函数。例如：ESP32-WROVER 模组上，外部上拉 GPIO12。GPIO12 在 ESP32 芯片中也有内部下拉。这意味着在 Deep-sleep 中，一些电流将流过这些外部和内部电阻，从而增加电流消耗。
 - 某些 IO 芯片内外电压不匹配，可能会增加电流消耗。如果存在电压不匹配的情况，需要在进入睡眠模式前配置 IO 为 Disable 模式，并使能 Hold 功能。

2. 使能 自动 Light-sleep 后，功耗较高

    - 存在任务持续运行占用 CPU 资源，导致 CPU 负载过大无法休眠。若是这种情况，请检测是否存在 任务 持续运行，并优化当前应用程序流程使 CPU 能周期性空闲进入 Light-sleep 模式。
    - 若系统应用中有小于 DTIM Beacon 间隔时间的循环定时，系统将不能进入 Light-sleep 模式。
    - 若未使用最新 IDF 3.2 版本，可能会因为某些外设使用（例如：I2C、I2S），导致无法自动进入 Light-sleep 模式。若是这个问题，请更新到最新的 IDF 3.2 版本。
    - 若系统不需要 FreeRTOS 时钟配置为 1000 Hz，可将 FreeRTOS 时钟降低为 100 Hz，可显著降低功耗。
    
3. 自动 Light-sleep 调试方法

    - 在 menuconfig 中使能 [CONFIG_PM_PROFILING](https://docs.espressif.com/projects/esp-idf/en/v3.2/api-reference/kconfig.html#config-pm-profiling) ，如果使能，`esp_pm_*` 函数将跟踪每个电源管理锁的保留时间，`esp_pm_dump_locks` 函数将打印此信息。 此功能可用于分析哪些锁阻止芯片进入低功耗状态，并查看芯片在每种省电模式下花费的时间，并应在应用程序中保持禁用状态。
    - 在 menuconfig 中使能 [CONFIG_PM_TRACE](https://docs.espressif.com/projects/esp-idf/en/v3.2/api-reference/kconfig.html#config-pm-trace)，如果使能，某些 GPIO 将用于发出 RTOS 滴答，频率切换，进入/退出空闲状态等事件的信号。有关 GPIO 列表，请参阅 `pm_trace.c` 文件。 此功能旨在用于分析/调试电源管理实现的行为时使用，并应在应用程序中保持禁用状态。

## 7 不同功耗模式下的功耗

![不同功耗模式下的功耗][3]


[1]: https://img-blog.csdnimg.cn/2020061714483194.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3,size_16,color_FFFFFF,t_70#pic_center
[2]: https://img-blog.csdnimg.cn/20200617144640674.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3,size_16,color_FFFFFF,t_70#pic_center
[3]: https://img-blog.csdnimg.cn/20200617144756531.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3,size_16,color_FFFFFF,t_70#pic_center
