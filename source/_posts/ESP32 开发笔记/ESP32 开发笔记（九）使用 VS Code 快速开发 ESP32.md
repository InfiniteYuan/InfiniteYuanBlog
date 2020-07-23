---
title: ESP32 开发笔记（九）使用 VS Code 快速开发 ESP32
date: 2018-12-17 18:28:16
categories:
- ESP32 开发笔记
tags:
- ESP32
---

# 使用 VS Code 快速开发 ESP32

## 搭建开发环境

- 根据[官方文档](https://blog.csdn.net/qq_27114397/article/details/79078449)进行 `esp-idf` 开发环境搭建
- 安装 [VS Code](https://code.visualstudio.com/Download) 

## 在 VS Code 中进行开发

- 将 `esp-idf` 中的模板工程 [hello_world](https://github.com/espressif/esp-idf/tree/master/examples/get-started/hello_world) 在 VS Code 中打开
- 在 VS Code 中开发项目

<!--more-->

## VS Code 任务、快捷键配置

### 任务配置

- 按下 `Ctrl+Shift+P`
- 输入、选择 `Tasks: Configure Task`(任务：配置任务)
- 使用模板创建 `tasks.json` 文件
- 选择 `others`
- 可使用下面的的任务配置模板（实现：快捷编译、下载、擦除 flash、清除编译、打开 monitor、menuconfig）

```c
{
    // See https://go.microsoft.com/fwlink/?LinkId=733558
    // for the documentation about the tasks.json format
    "version": "2.0.0",
    "tasks": [
        {
            "label": "build app", // f5
            "type": "shell",
            "command": "cd ${fileDirname} && cd ../ && make -j8",
            "group": {
                "kind": "build",
                "isDefault": true
            }
        },
        {
            "label": "flash app", // f6
            "type": "shell",
            "command": "cd ${fileDirname} && cd ../ && make -j8 flash"
        },
        {
            "label": "monitor", // f7
            "type": "shell",
            "command": "cd ${fileDirname} && cd ../ && make monitor"
        },
        {
            "label": "clean app", // f8
            "type": "shell",
            "command": "cd ${fileDirname} && cd ../ && make clean",
        },
        {
            "label": "erase flash", // f9
            "type": "shell",
            "command": "cd ${fileDirname} && cd ../ && make erase_flash",
        },
        {
            "label": "menuconfig", // f10
            "type": "shell",
            "command": "cd ${fileDirname} && cd ../ && make menuconfig"
        },
    ]
}
```

详细配置过程：![详细配置过程](https://img-blog.csdnimg.cn/20181217181209238.gif)

### 快捷键配置

接下来我们给这些编译命令增加快捷键。

- 按下：`Ctrl+Shift+P`
- 输入、选择 `Preferences: Open Keyboard Shortcuts(JSON)` (首选项:打开键盘快捷方式)
- 高级自定义请打开和编辑 `keybindings.json`
- 填充参数

```c
// Override key bindings by placing them into your key bindings file.
[
    {
        "key": "f5",
        "command": "workbench.action.tasks.runTask",
        "args": "build app"
    },
    {
        "key": "f6",
        "command": "workbench.action.tasks.runTask",
        "args": "flash app"
    },
    {
        "key": "f7",
        "command": "workbench.action.tasks.runTask",
        "args": "monitor"
    },
    {
        "key": "f8",
        "command": "workbench.action.tasks.runTask",
        "args": "clean app"
    },
    {
        "key": "f9",
        "command": "workbench.action.tasks.runTask",
        "args": "erase flash"
    },
    {
        "key": "f10",
        "command": "workbench.action.tasks.runTask",
        "args": "menuconfig"
    }
]
```

这样我们就可通过快捷键进行编译、下载等

快捷键 | 执行的命令 | 功能
-------- | -------- | --------
F5 | `make -j8` | 编译
F6 | `make -j8 flash` | 编译、下载
F7 | `make monitor` | 监视器
F8 | `make clean` | 清除编译
F9 | `make erase_flash` | 擦除 flash
F10 | `make menuconfig` | 打开 menuconfig

> NOTE: 这些命令都应该在工程的 `main` 目录下的文件中执行，例如： 在 VS Code 中打开了 hello_world 工程中 main 目录下的 `hello_world_main.c` 文件，可以按快捷键 `F6` 进行编译、下载。暂不支持在其他目录下进行。


详细配置过程：
![详细配置过程](https://img-blog.csdnimg.cn/20181217182915118.gif)
