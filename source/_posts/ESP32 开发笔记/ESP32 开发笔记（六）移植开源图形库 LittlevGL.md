---
title: ESP32 开发笔记（六）移植开源图形库 LittlevGL
date: 2018-08-08 10:38:39
categories:
- ESP32 开发笔记
tags:
- ESP32
---

# ESP32 移植开源图形库 LittlevGL

ESP32 移植 LittlevGL 的源码：[GitHub 源码地址](https://github.com/InfiniteYuan/esp32-lvgl-gui)
欢迎 Star ～

## LittlevGL 介绍

>littlevGL is a free and open-source graphics library providing everything you need to create embedded GUI with easy-to-use graphical elements, beautiful visual effects and low memory footprint.
>[LittlevGL 官网链接](https://littlevgl.com/)
[LittlevGL 移植教程](https://littlevgl.com/porting)
[LittlevGL 官方文档](https://littlevgl.com/basics)
[LittlevGL GitHub链接](https://github.com/littlevgl/lvgl)

<!--more-->

<h4>关键特性：</h4>

 - Powerful building blocks buttons, charts, lists, sliders, images etc
 - Advanced graphics with animations, anti-aliasing, opacity, smooth scrolling
 - Various input devices touch pad, mouse, keyboard, encoder, buttons etc
 - Multi language support with UTF-8 decoding
 - Fully customizable graphical elements
 - Hardware independent to use with any microcontroller or display
 - Scalable to operate with few memory (50 kB Flash, 10 kB RAM)
 - OS, External memory and GPU supported but not required
 - Single frame buffer operation even with advances graphical effects
 - Written in C for maximal compatibility
 - Simulator to develop on PC without embedded hardware
 - Tutorials, examples, themes for rapid development
 - Documentation and API references online

## 移植过程

 1. 在工程目录下新建一个 components 目录，放入 LittlevGL 源码
 2. 在 components 目录下新建一个 include 目录，放入 lv_conf.h 文件，由 LittlevGL 源码目录中的 lv_conf_templ.h文件而来
 3. 在 components 目录下新建一个  component.mk  文件，将 LittlevGL 相关源码添加到组件的 `COMPONENT_SRCDIRS`、`COMPONENT_ADD_INCLUDEDIRS`和`COMPONENT_PRIV_INCLUDEDIRS`目录变量中
 4. 之后在程序中使用了～，可根据 [LittlevGL Tutorial](https://github.com/littlevgl/lv_examples/blob/master/lv_tutorial/0_porting/lv_tutorial_porting.c) 在自己的主程序中实现

<h4>大致初始化流程：</h4>

```c
   /***********************
    * Initialize LittlevGL
    ***********************/
   lv_init();
   
   /***********************
    * Tick interface
    ***********************/
   /* Initialize a Timer for 1 ms period and
    * in its interrupt call
    * lv_tick_inc(1); */

   /***********************
    * Display interface
    ***********************/
   lv_disp_drv_t disp_drv;                         /*Descriptor of a display driver*/
   lv_disp_drv_init(&disp_drv);                    /*Basic initialization*/

   /*Set up the functions to access to your display*/
   disp_drv.disp_flush = ex_disp_flush;            /*Used in buffered mode (LV_VDB_SIZE != 0  in lv_conf.h)*/

   disp_drv.disp_fill = ex_disp_fill;              /*Used in unbuffered mode (LV_VDB_SIZE == 0  in lv_conf.h)*/
   disp_drv.disp_map = ex_disp_map;                /*Used in unbuffered mode (LV_VDB_SIZE == 0  in lv_conf.h)*/
   /*Finally register the driver*/
   lv_disp_drv_register(&disp_drv);

   /*************************
    * Input device interface
    *************************/
   /*Add a touchpad in the example*/
   /*touchpad_init();*/                            /*Initialize your touchpad*/
   lv_indev_drv_t indev_drv;                       /*Descriptor of an input device driver*/
   lv_indev_drv_init(&indev_drv);                  /*Basic initialization*/
   indev_drv.type = LV_INDEV_TYPE_POINTER;         /*The touchpad is pointer type device*/
   indev_drv.read = ex_tp_read;                    /*Library ready your touchpad via this function*/
   lv_indev_drv_register(&indev_drv);              /*Finally register the driver*/
   
   /*************************************
    * Run the task handler of LittlevGL
    *************************************/
   /* Periodically call this function.
    * The timing is not critical but should be between 1..10 ms */
   lv_task_handler();
   /*delay_ms(5)*/
```

<h4>需要实现的函数：</h4>

```c
   /***********************
    * Display interface
    ***********************/
    /*Write the internal buffer (VDB) to the display. 'lv_flush_ready()' has to be called when finished*/
    void (*disp_flush)(int32_t x1, int32_t y1, int32_t x2, int32_t y2, const lv_color_t * color_p);
    /*Fill an area with a color on the display*/
    void (*disp_fill)(int32_t x1, int32_t y1, int32_t x2, int32_t y2, lv_color_t color);
    /*Write pixel map (e.g. image) to the display*/
    void (*disp_map)(int32_t x1, int32_t y1, int32_t x2, int32_t y2, const lv_color_t * color_p);
    
   /*************************
    * Input device interface
    *************************/
    bool (*read)(lv_indev_data_t *data);        /*Function pointer to read data. Return 'true' if there is still data to be read (buffered)*/
```

>**注意：**在实现 `disp_flush` 函数时，在函数末尾需要调用 LittlevGL 提供的 `lv_flush_ready` 函数
>上述的这些函数可以自行实现，函数体保持一致即可

欢迎关注本人 [GitHub](https://github.com/InfiniteYuan) ，更新 ESP32 相关开源库
