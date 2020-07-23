---
title: ESP32 开发笔记（十二）LittlevGL 添加自定义字体和物理按键
date: 2019-05-12 20:05:34
categories:
- ESP32 开发笔记
tags:
- ESP32
---

# LittlevGL 添加自定义字体

1. 获取字库 `ttf` 文件
可以从一些网站上获取字库文件，比如
> 请注意字体许可证
2. 生成源文件
使用 LittlevGL 提供的字库文件[转换工具](https://littlevgl.com/ttf-font-to-c-array)，将 `ttf` 字库文件转换为源文件。

<!--more-->
3. 将生成的源文件添加到 LittlevGL 工程中，添加以下代码声明字体：
```
extern lv_font_t my_font_name; 
```
或者
```
LV_FONT_DECLARE(my_font_name);
```
源代码中使用这个字体可以：
```
style.text.font = &my_font_name;
```
或者将这个字体添加到当前使用的字体中：
```
lv_font_add(&my_font_name, &current_use);
```

例如：
```
LV_FONT_DECLARE(my_font_name);

void lv_chinese_fonts1(void)
{
    /*Concatenate the fonts into one*/
//    lv_font_add(&arial_cyrillic_20, &arial_ascii_20); 相同高度才可以添加到一起
//    lv_font_add(&arial_math_20, &arial_ascii_20);
 
    /* 创建一个新的样式，并且修改新样式的文本字体 */
    static lv_style_t style1; 
    lv_style_copy(&style1, &lv_style_plain);
    style1.text.font = &my_font_name; /* 设置自定义字体 */
 
    /*Create a label and set new text*/
    lv_obj_t * label = lv_label_create(lv_scr_act(), NULL); /* 创建标签 */
    lv_obj_set_pos(label, 10, 10);       /* 设置相对位置 */
    lv_label_set_style(label, &style1);  /* 设置样式 */
    lv_label_set_text(label, "Hello World!\n 世界你好,我是littleVGL!");      /* 显示汉字 */
}
```

# LittlevGL 添加自定义符号

流程：

 - 寻找合适的 `ttf` 文件（包含目标符号，可在 linux 下通过 FontForge 软件打开 `ttf` 文件，查看包含那些字符）
 - 使用 LittlevGL 提供的字库文件[转换工具](https://littlevgl.com/ttf-font-to-c-array)，将选中的符号转换为 `c` 源文件。在转换的页面 `Range` 中输入目标符号的 Unicode 编码，怎么查询可以通过百度
 - 将源文件添加到工程中，和上面添加字体类似，首先声明字体，然后添加到字体中或者直接使用。
 - 定义一个宏指向这个目标符号，宏的内容需要为 `UTF-8编码`，通过 [Unicode和UTF编码转换](https://www.qqxiuzi.cn/bianma/Unicode-UTF.php) 可以进行转换

例如：
![摄氏度符号](https://img-blog.csdnimg.cn/20190515163138800.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3,size_16,color_FFFFFF,t_70)

这样就需要在程序中：

```
/* MACROS */
#define SYMBOL_TEMP "\xE2\x84\x83" // E28483

/* STATIC VARIABLES */
LV_FONT_DECLARE(tempreture_symbol_40);

/* Add font to current font */
lv_font_add(&tempreture_symbol_40, &lv_font_dejavu_40);

/* Use this symbol */
lv_label_set_text(temp_l, "21.5 "SYMBOL_TEMP);
```

# LittlevGL 添加物理按键

注册物理按键驱动：
```
uint8_t my_btn_read()
{
    if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == 0)
    {
        return 1;
    }
    else
    {
        return 0;
    }
}
bool my_input_read(lv_indev_data_t *data)
{
    static int8_t last_btn = 0;    /* Store the last pressed button */
    int8_t btn_pr = my_btn_read(); /* Get the ID (0,1,2...) of the pressed button */

    if (btn_pr > 0)
    {                                    /* Is there a button press? */
        last_btn = btn_pr;               /* Save the ID of the pressed button */
        data->state = LV_INDEV_STATE_PR; /* Set the pressed state */
    }
    else
    {
        data->state = LV_INDEV_STATE_REL; /* Set the released state */
    }

    data->btn = last_btn; /* Set the last button */

    return false; /* No buffering so no more data read */
}
void my_button_init(void)
{
    static lv_indev_t *indev;
    lv_indev_drv_t indev_drv;

    lv_indev_drv_init(&indev_drv);

    indev_drv.read = my_input_read;
    indev_drv.type = LV_INDEV_TYPE_BUTTON;
    indev = lv_indev_drv_register(&indev_drv);

    /*points_array: these points will be assigned to the buttons to press a specific point on the screen.*/
    static lv_point_t points_array[] = {{20, 20}};
    lv_indev_set_button_points(indev, points_array);
}
```

创建一个按键，保证上面的点在按钮的区域内：

```
static lv_obj_t *btn = lv_btn_create(lv_scr_act(), NULL);  
lv_obj_set_size(btn, 40, 40);
lv_obj_set_pos(btn, 0, 0);
lv_btn_set_action(btn, LV_BTN_ACTION_CLICK, btn_click_action);
```
