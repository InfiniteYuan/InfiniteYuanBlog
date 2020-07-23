---
title: ESP32 开发笔记（十五）使用 LittlevGL 实现 2048 小游戏
date: 2020-05-17 14:43:38
categories:
- ESP32 开发笔记
tags:
- ESP32
---

# 使用 LittlevGL 实现 2048 小游戏

2048 这款益智小游戏，游戏的规则十分简单，简单易上手的数字小游戏，但又十分虐心。曾经也是风靡一时。

现在我们在 ESP32 上自己动手实现 2048 这款小游戏吧。

## 1 使用 LittlevGL

esp32-lvgl-gui 仓库已经适配了 LittlevGL V5.3，并适配了几款屏幕(ILI9341、ST7789、SSD1306、NT35510) 和 触摸驱动(FT5X06、XPT2046)。

使用时，在 `menuconfig` 中选择你所使用的 屏幕 和 触摸驱动。

编译代码：

1. 添加头文件：

```c
/* lvgl includes */
#include "iot_lvgl.h"

/* game include */
#include "game.h"
```

2. 初始化 LittlevGL：

```c
extern "C" void app_main()
{
    /* Initialize LittlevGL */
    lv_init();

    /* Tick interface， Initialize a Timer for 1 ms period and in its interrupt call*/
    // esp_register_freertos_tick_hook(lv_tick_task_callback);
    lvgl_tick_timer = xTimerCreate(
        "lv_tickinc_task",
        1 / portTICK_PERIOD_MS,            //period time
        pdTRUE,                            //auto load
        (void *)NULL,                      //timer parameter
        lv_tick_task_callback);            //timer callback
    xTimerStart(lvgl_tick_timer, 0);

    /* Display interface */
    lvgl_lcd_display_init();	           /*Initialize your display*/

    /* Input device interface */
    input_device = lvgl_indev_init();                     /*Initialize your indev*/

    lvgl_timer = xTimerCreate(
        "lv_task",
        10 / portTICK_PERIOD_MS,           //period time
        pdTRUE,                            //auto load
        (void *)NULL,                      //timer parameter
        lvgl_task_time_callback);          //timer callback
    xTimerStart(lvgl_timer, 0);

	// 2048 game init
    game_init(480);

	// game logic handle task
    xTaskCreate(
        user_task,   //Task Function
        "user_task", //Task Name
        1024*4,      //Stack Depth
        NULL,        //Parameters
        1,           //Priority
        NULL);       //Task Handler
}
```

## 2 2048 游戏逻辑

### 2.1 界面初始化

将游戏相关界面初始化分为以下几步：

1. Step 1: 画游戏背景
2. Step 2: 画游戏分数显示框
3. Step 3: 画游戏网格
4. Step 4: 画游戏网格中每个单元格的内容

### 2.2 滑动处理

#### 2.2.1 判断滑动方向

通过调用 LittlevGL 的 API 读取触摸状态(抬起/按下、坐标点)，计算水平/竖直方向滑动的差值，判断为哪个方向上的滑动(上/下/左/右)，执行相应的操作。

```c
static lv_indev_drv_t input_device;

static void user_task(void *pvParameter)
{
    bool pressing = false;
    uint8_t value = 0;
    lv_indev_data_t touchpad_data;
    lv_point_t last_data;
    int16_t x_diff, y_diff;
    while (1)
    {
        vTaskDelay(50 / portTICK_PERIOD_MS);
        input_device.read(&touchpad_data); // 读取触摸驱动的值
        if (touchpad_data.state == LV_INDEV_STATE_REL) { // 当前为 `抬起` 状态
            pressing = false; // 计算坐标偏移量
            x_diff = touchpad_data.point.x - last_data.x;
            y_diff = touchpad_data.point.y - last_data.y;
            if(fabs(x_diff) > SENSITIVE || fabs(y_diff) > SENSITIVE) { // 判断滑动距离是否超过判断阈值
                if (fabs(x_diff) > fabs(y_diff)) { // 判断是否为水平滑动
                    if (x_diff > 0) { // 判单是否为向右滑动
                        move_right();
                    } else { // 向左滑动
                        move_left();
                    }
                } else { // 竖直方向滑动
                    if (y_diff > 0) { // 判单是否为向下滑动
                        move_down();
                    } else { // 向上滑动
                        move_up();
                    }
                }
            }
            last_data.x = touchpad_data.point.x;
            last_data.y = touchpad_data.point.y;
        } else if (touchpad_data.state == LV_INDEV_STATE_PR) { // 当前为 `按下` 状态
            if (!pressing) { // 按下状态，记录初次按下的坐标点
                last_data.x = touchpad_data.point.x;
                last_data.y = touchpad_data.point.y;
                pressing = true;
            }
        }
    }
}
```

#### 2.2.2 滑动逻辑处理

1. 判断同一行/列滑动方向上是否存在相等的数值
2. 相同的单元格，数值相加为相邻单元格中的后一个(滑动方向上)的数值
3. 画出总的得分
4. 判断游戏是否结束
5. 刷新界面上的单元格

```c
static uint16_t num_matrix[5][5] = {0}; // 保存所有的单元格中的数值
static uint32_t score_num = 0; // 总得分

void move_up()
{
    int i, j, k, t, nx = -1, ny = 0, nn = 0;
    for (i = 1; i <= 4; ++i)
        tmp[i] = 0;
    for (j = 1; j <= 4; ++j)
    {
        k = 0;
        for (i = 1; i <= 4; ++i)
        {
            if (num_matrix[i][j])
            {
                tmp[++k] = num_matrix[i][j];
                num_matrix[i][j] = 0;
            }
        }
        t = 1;
        while (t <= k - 1)
        {
            if (tmp[t] == tmp[t + 1])
            {
                tmp[t] *= 2;
                tmp[t + 1] = 0;
                score_num += tmp[t];
                if (nx == -1)
                {
                    nx = t;
                    ny = j;
                    nn = tmp[t];
                }
                t += 2;
            }
            else
                t++;
        }
        t = 1;
        for (i = 1; i <= k; ++i)
            if (tmp[i])
                num_matrix[t++][j] = tmp[i];
    }
    // 画总分
    draw_score_num(score_num);
    // 刷新网格中单元格的内容
    gen_num();
}
```

## 参考链接

GitHub 源码：[esp32-lvgl-gui](https://github.com/InfiniteYuan/esp32-lvgl-gui/tree/master/lvgl_2048)
Twitter 视频：[Twitter](https://twitter.com/InfiniteYuan/status/1048106496649641985?s=20)
