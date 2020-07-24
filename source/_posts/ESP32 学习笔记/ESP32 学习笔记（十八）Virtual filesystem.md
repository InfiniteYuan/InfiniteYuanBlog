---
title: ESP32 学习笔记（十八）Virtual filesystem
date: 2018-11-15 17:24:06
categories:
- ESP32 学习笔记
tags:
- ESP32
---

# Virtual filesystem component

## 概述

虚拟文件系统(VFS)组件为可以对类文件对象执行操作的驱动程序提供统一的接口。这可以是真实的文件系统(FAT，SPIFFS 等)，也可以是有文件类接口的设备驱动程序。

该组件允许 C 库函数(如 `fopen` 和` fprintf` )与 FS 驱动程序一起使用。在高层次，每个 FS 驱动程序与某些路径前缀相关联。当其中一个 C 库函数需要打开文件时，VFS 组件将搜索与该文件路径关联的 FS 驱动程序，并将该调用转发给该驱动程序。VFS 还将给定文件的读取，写入和其他调用转发到同一 FS 驱动程序。

例如，可以使用 `/fat` 前缀注册 FAT 文件系统驱动程序，并调用 `fopen(“/fat/file.txt”，“w”)`。然后，VFS 组件将调用 FAT 驱动程序的 `open` 函数并将 `/file.txt` 参数传递给它(以及适当的模式标志)。对返回的 `FILE *` 流的所有后续 C 库函数调用也将被转发到 FAT 驱动程序。

<!--more-->

## FS注册

要注册 FS 驱动程序，应用程序需要在 `esp_vfs_t` 结构的实例中定义，并使用 `FS API` 的函数指针填充它：
```
esp_vfs_t myfs = {
    .flags = ESP_VFS_FLAG_DEFAULT,
    .write = &myfs_write,
    .open = &myfs_open,
    .fstat = &myfs_fstat,
    .close = &myfs_close,
    .read = &myfs_read,
};

ESP_ERROR_CHECK(esp_vfs_register("/data", &myfs, NULL));
```
根据 FS 驱动程序声明其 API 的方式，应使用 `read`，`write` 等，或 `read_p`，`write_p` 等。

情况1：声明 API 函数时没有额外的上下文指针(FS 驱动程序是单例)：
```
ssize_t myfs_write(int fd, const void * data, size_t size);

// In definition of esp_vfs_t:
    .flags = ESP_VFS_FLAG_DEFAULT,
    .write = &myfs_write,
// ... other members initialized

// When registering FS, context pointer (third argument) is NULL:
ESP_ERROR_CHECK(esp_vfs_register("/data", &myfs, NULL));
```
情况2：使用额外的上下文指针声明 API 函数(FS 驱动程序支持多个实例)：
```
ssize_t myfs_write(myfs_t* fs, int fd, const void * data, size_t size);

// In definition of esp_vfs_t:
    .flags = ESP_VFS_FLAG_CONTEXT_PTR,
    .write_p = &myfs_write,
// ... other members initialized

// When registering FS, pass the FS context pointer into the third argument
// (hypothetical myfs_mount function is used for illustrative purposes)
myfs_t* myfs_inst1 = myfs_mount(partition1->offset, partition1->size);
ESP_ERROR_CHECK(esp_vfs_register("/data1", &myfs, myfs_inst1));

// Can register another instance:
myfs_t* myfs_inst2 = myfs_mount(partition2->offset, partition2->size);
ESP_ERROR_CHECK(esp_vfs_register("/data2", &myfs, myfs_inst2));
```

## 同步输入/输出多路复用

如果要通过 `select()` 使用同步输入/输出多路复用，则需要使用 `start_select()` 和 `end_select()` 函数注册 VFS，类似于以下示例：
```
// In definition of esp_vfs_t:
    .start_select = &uart_start_select,
    .end_select = &uart_end_select,
// ... other members initialized
```
调用 `start_select()` 以设置用于检测属于给定 VFS 的文件描述符的读/写/错误条件的环境。调用 `end_select()` 来停止/取消初始化/释放由 `start_select()` 设置的环境。请参阅 [`vfs/vfs_uart.c`](https://github.com/espressif/esp-idf/blob/fb7ba1b/components/vfs/vfs_uart.c) 中 UART 外设的参考实现，尤其是函数 `esp_vfs_dev_uart_register()`，`uart_start_select()` 和 `uart_end_select()`。

演示使用带有 VFS 文件描述符的 `select()` 的示例是 [`peripherals/uart_select`](https://github.com/espressif/esp-idf/tree/fb7ba1b/examples/peripherals/uart_select) 和 [](https://github.com/espressif/esp-idf/tree/fb7ba1b/examples/system/select) 示例。

如果 `select()` 仅用于套接字文件描述符，那么可以启用 `CONFIG_USE_ONLY_LWIP_SELECT` 选项，这可以减少代码大小并提高性能。

## 路径

每个注册的 FS 都有一个与之关联的路径前缀。该前缀可以被认为是该分区的 `“mount point”`(挂载点)。

如果嵌套安装点，则在打开文件时将使用具有最长匹配路径前缀的安装点。例如，假设以下文件系统在 VFS 中注册：
 - FS 1 on /data
 - FS 2 on /data/static
然后：
 - 打开名为 `/data/log.txt` 的文件时将使用 FS 1
 - 打开名为 `/data/static/index.html` 的文件时将使用 FS 2
 - 即使 FS 2 中不存在 `/index.html`，也不会搜索 FS 1 `/static/index.html`。

作为一般规则，挂载点名称必须以路径分隔符(`/`)开头，并且必须在路径分隔符后包含至少一个字符。但是，也支持空挂载点名称，并且可以在应用程序需要提供“回退”文件系统或完全覆盖VFS功能的情况下使用。如果没有前缀匹配给定的路径，则将使用此类文件系统。

VFS 不以任何特殊方式处理路径名中的点(.)。VFS 不会将 `...` 视为对父目录的引用。即在上面的例子中，使用路径 `/data/static/../log.txt` 不会导致调用 FS 1 来打开 `/log.txt`。特定 FS 驱动程序(如 FATFS)可能以不同方式处理文件名中的点。

打开文件时，FS 驱动程序只会获得文件的相对路径。例如：
 - `myfs` 驱动程序使用 `/data` 注册为路径前缀
 - 和应用程序调用 `fopen(“/data/config.json”，...)`
 - 然后 VFS 组件将调用 `myfs_open(“/config.json”，...)`。
 - `myfs` 驱动程序将打开 `/config.json` 文件

VFS 不会对总文件路径长度施加限制，但会将 FS 路径前缀限制为 `ESP_VFS_PATH_MAX` 字符。各个 FS 驱动程序可能有自己的文件名长度限制。

## 文件描述符

文件描述符是从 `0` 到 `FD_SETSIZE -1` 其中 `FD_SETSIZE` 在newlib的 `sys/types.h` 中定义。最大的文件描述符(由 `CONFIG_LWIP_MAX_SOCKETS` 配置)保留给套接字。VFS 组件包含一个名为 `s_fd_table` 的查找表，用于将全局文件描述符映射到 `s_vfs` 数组中注册的 VFS 驱动程序索引。

## 标准 IO 流(stdin，stdout，stderr)

如果“UART for console output”menuconfig选项未设置为“None”，则 `stdin`，`stdout` 和 `stderr` 配置为读取和写入 UART。可以将 UART0 或 UART1 用于标准 IO。默认情况下，使用 UART0，波特率为 115200，TX 引脚为 GPIO1，RX 引脚为 GPIO3。可以在 menuconfig 中更改这些参数。

写入 `stdout` 或 `stderr` 会将字符发送到 UART 发送 FIFO。从 `stdin` 读取将从 UART 接收 FIFO 中检索字符。

默认情况下，VFS 使用简单的函数来读取和写入 UART。写入忙 - 等待直到所有数据都被放入 UART FIFO，并且读取是非阻塞的，只返回 FIFO 中的数据。由于这种非阻塞读取行为，更高级别的 C 库调用，例如 `fscanf(“％d \ n”，＆var);` 可能没有预期的结果。

使用 UART 驱动程序的应用程序可能会指示 VFS 使用驱动程序的中断驱动，阻塞读写功能。这可以通过调用 `esp_vfs_dev_uart_use_driver` 函数来完成。也可以使用 `esp_vfs_dev_uart_use_nonblocking` 调用恢复到基本的非阻塞函数。

VFS 还为输入和输出提供可选的换行转换功能。在内部，大多数应用程序发送和接收由 LF('n'')字符终止的行。不同的终端程序可能需要不同的线路终端，例如 CR 或 CRLF。应用程序可以通过 menuconfig 或通过调用 `esp_vfs_dev_uart_set_rx_line_endings` 和 `esp_vfs_dev_uart_set_tx_line_endings` 函数单独为输入和输出配置它。

## 标准流和 FreeRTOS 任务

`stdin`，`stdout` 和 `stderr` 的 `FILE` 对象在所有 FreeRTOS 任务之间共享，但指向这些对象的指针存储在每个任务的 `struct _reent` 中。以下代码：
```
fprintf(stderr，“42\n”);
```
实际上被翻译成这个(由预处理器)：
```
fprintf(__ getreent() -> _stderr，“42\n”);
```
其中 ```__getreent()```函数返回一个指向 ```struct _reent```的每个任务指针 `(newlib/include/sys/reent.h＃L370-L417)`。此结构在每个任务的 TCB 上分配。初始化任务时，`struct _reent` 的 `_stdin`，`_stdout` 和 `_stderr` 成员被设置为 `_GLOBAL_REENT` 的 `_stdin`，`_stdout` 和 `_stderr` 的值(即在 FreeRTOS 启动之前使用的结构)。

这样的设计会产生以下后果：
 - 可以为任何给定任务设置 `stdin`，`stdout` 和 `stderr`，而不会影响其他任务，例如通过做 `stdin = fopen(“/dev/uart/1”，“r”)`。
 - 使用 fclose 关闭默认 `stdin`，`stdout` 或 `stderr` 将关闭 `FILE` 流对象 - 这将影响所有其他任务。
 - 要更改新任务的默认 `stdin`，`stdout`，`stderr` 流，请在创建任务之前修改 `_GLOBAL_REENT - > _ stdin(_stdout，_stderr)`。

## 参考资料

 - [Virtual filesystem](https://docs.espressif.com/projects/esp-idf/en/v3.2/api-reference/storage/vfs.html)
