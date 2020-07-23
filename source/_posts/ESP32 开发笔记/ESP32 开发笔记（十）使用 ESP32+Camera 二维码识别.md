---
title: ESP32 开发笔记（十）使用 ESP32+Camera 二维码识别
date: 2019-03-14 15:16:37
categories:
- ESP32 开发笔记
tags:
- ESP32
---

# 使用 ESP32 Camera 进行二维码识别

 - GitHub: [esp32-camera-qr-recoginize](https://github.com/InfiniteYuan/esp32-camera-qr-recoginize/tree/master/examples/single_chip/qrcode_recoginize)

## 环境搭建

 - [ESP-WHO](https://github.com/espressif/esp-who)
 - [ESP-IDF](https://github.com/espressif/esp-idf)

<!--more-->

## 使用 quirc 二维码识别库

 - [quirc](https://github.com/dlbeer/quirc)

## 摄像头初始化

```c
#define CAMERA_PIXEL_FORM PIXFORMAT_GRAYSCALE
#define CAMERA_FRAME_SIZE FRAMESIZE_VGA

void app_main()
{
#if CONFIG_CAMERA_MODEL_CUSTOM
    /* IO13, IO14 is designed for JTAG by default,
     * to use it as generalized input,
     * firstly declair it as pullup input */
    gpio_config_t conf;
    conf.mode = GPIO_MODE_INPUT;
    conf.pull_up_en = GPIO_PULLUP_ENABLE;
    conf.pull_down_en = GPIO_PULLDOWN_DISABLE;
    conf.intr_type = GPIO_INTR_DISABLE;
    conf.pin_bit_mask = 1LL << 13;
    gpio_config(&conf);
    conf.pin_bit_mask = 1LL << 14;
    gpio_config(&conf);
#endif

    config.ledc_channel = LEDC_CHANNEL_0;
    config.ledc_timer = LEDC_TIMER_0;
    config.pin_d0 = Y2_GPIO_NUM;
    config.pin_d1 = Y3_GPIO_NUM;
    config.pin_d2 = Y4_GPIO_NUM;
    config.pin_d3 = Y5_GPIO_NUM;
    config.pin_d4 = Y6_GPIO_NUM;
    config.pin_d5 = Y7_GPIO_NUM;
    config.pin_d6 = Y8_GPIO_NUM;
    config.pin_d7 = Y9_GPIO_NUM;
    config.pin_xclk = XCLK_GPIO_NUM;
    config.pin_pclk = PCLK_GPIO_NUM;
    config.pin_vsync = VSYNC_GPIO_NUM;
    config.pin_href = HREF_GPIO_NUM;
    config.pin_sscb_sda = SIOD_GPIO_NUM;
    config.pin_sscb_scl = SIOC_GPIO_NUM;
    config.pin_pwdn = PWDN_GPIO_NUM;
    config.pin_reset = RESET_GPIO_NUM;
    // Only support 10 MHz current. Camera will output bad image when XCLK is 20 MHz.
    config.xclk_freq_hz = 10000000;
    config.pixel_format = CAMERA_PIXEL_FORM;
    config.frame_size = CAMERA_FRAME_SIZE;
    config.jpeg_quality = 10;
    config.fb_count = 1;

    // camera init
    esp_err_t err = esp_camera_init(&config);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "Camera init failed with error 0x%x", err);
        return;
    }

    /**
     * Create QR-code recognize task
     */
    app_qr_recognize(&config);

    ESP_LOGI(TAG, "Free heap: %u", xPortGetFreeHeapSize());
}
```

## 二维码识别

```c
void qr_recoginze(void *parameter)
{
    camera_config_t *camera_config = (camera_config_t *)parameter;
    // Use VGA Size currently, but quirc can support other frame size.(eg: FRAMESIZE_SVGA,FRAMESIZE_VGA，
    // FRAMESIZE_CIF,FRAMESIZE_QVGA,FRAMESIZE_HQVGA,FRAMESIZE_QCIF,FRAMESIZE_QQVGA2,FRAMESIZE_QQVGA,etc)
    if (camera_config->frame_size > FRAMESIZE_VGA) {
        ESP_LOGE(TAG, "Camera Frame Size err %d", (camera_config->frame_size));
        vTaskDelete(NULL);
    }

    // Save image width and height, avoid allocate memory repeatly.
    uint16_t old_width = 0;
    uint16_t old_height = 0;

    // Construct a new QR-code recognizer.
    ESP_LOGI(TAG, "Construct a new QR-code recognizer(quirc).");
    struct quirc *qr_recognizer = quirc_new();
    if (!qr_recognizer) {
        ESP_LOGE(TAG, "Can't create quirc object");
    }
    camera_fb_t *fb = NULL;
    uint8_t *image = NULL;
    int id_count = 0;
    UBaseType_t uxHighWaterMark;

    /* 入口处检测一次 */
    ESP_LOGI(TAG, "uxHighWaterMark = %d", uxTaskGetStackHighWaterMark( NULL ));

    while (1) {
        ESP_LOGI(TAG, "uxHighWaterMark = %d", uxTaskGetStackHighWaterMark( NULL ));
        // Capture a frame
        fb = esp_camera_fb_get();
        if (!fb) {
            ESP_LOGE(TAG, "Camera capture failed");
            continue;
        }

        if (old_width != fb->width || old_height != fb->height) {
            ESP_LOGD(TAG, "Recognizer size change w h len: %d, %d, %d", fb->width, fb->height, fb->len);
            ESP_LOGI(TAG, "Resize the QR-code recognizer.");
            // Resize the QR-code recognizer.
            if (quirc_resize(qr_recognizer, fb->width, fb->height) < 0) {
                ESP_LOGE(TAG, "Resize the QR-code recognizer err.");
                continue;
            } else {
                old_width = fb->width;
                old_height = fb->height;
            }
        }

        /** These functions are used to process images for QR-code recognition.
         * quirc_begin() must first be called to obtain access to a buffer into
         * which the input image should be placed. Optionally, the current
         * width and height may be returned.
         *
         * After filling the buffer, quirc_end() should be called to process
         * the image for QR-code recognition. The locations and content of each
         * code may be obtained using accessor functions described below.
         */
        image = quirc_begin(qr_recognizer, NULL, NULL);
        memcpy(image, fb->buf, fb->len);
        quirc_end(qr_recognizer);

        // Return the number of QR-codes identified in the last processed image.
        id_count = quirc_count(qr_recognizer);
        if (id_count == 0) {
            ESP_LOGE(TAG, "Error: not a valid qrcode");
            esp_camera_fb_return(fb);
            continue;
        }

        // Print information of QR-code
        dump_info(qr_recognizer, id_count);
        esp_camera_fb_return(fb);
    }
    // Destroy QR-Code recognizer (quirc)
    quirc_destroy(qr_recognizer);
    ESP_LOGI(TAG, "Deconstruct QR-Code recognizer(quirc)");
    vTaskDelete(NULL);
}
```

## 示例结果

![示例结果](https://img-blog.csdnimg.cn/20190315190639946.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3,size_16,color_FFFFFF,t_70)
