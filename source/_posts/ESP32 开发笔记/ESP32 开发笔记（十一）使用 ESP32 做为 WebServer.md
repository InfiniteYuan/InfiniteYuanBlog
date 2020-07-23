---
title: ESP32 开发笔记（十一）使用 ESP32 做为 WebServer
date: 2019-04-28 19:34:34
categories:
- ESP32 开发笔记
tags:
- ESP32
---

# 使用 ESP32 做为 WebServer

在某些场景，我们可能需要在手机上或者其他移动终端访问 ESP32 的数据，这个时候我们需要在手机上展示 ESP32 设备的相关信息，这个时候可以用 APP 在手机上展示数据，或者在手机浏览器中打开存储在 ESP32 上的网页。或者其他的方式。

这篇文章我们将介绍第二种方式。在 ESP32 上存储网页文件，将 ESP32 做为一个简单的 WebServer。

工作流程：(第一种方式)

 1. 首先通过 gzip 将 HTML 文件压缩为 `.gz` 文件
 2. 使用 filetoarray 工具将 `.gz` 文件转为头文件
 3. 在 ESP32 程序中将头文件中的数组发送出去

工作流程：(第二种方式)

 1. 首先通过 gzip 将 HTML 文件压缩为 `.gz` 文件
 2. 使用 ESP32 构建系统中的[嵌入二进制数据](https://blog.csdn.net/qq_27114397/article/details/81152448#_449)的方式，将其添加到 Flash 中的 `.rodata` 部分
 3. 在 ESP32 程序中将 Flash 中的数组发送出去

<!--more-->

## filetoarray 工具

使用这个工具将 `.gz` 文件转换为包含十六进制数组和其长度的头文件。

源码如下：
```c
#include <stdio.h>
#include <stdarg.h>
#include <stdlib.h>
#include <string.h>

int main(int argc, char *argv[])
{
    FILE *fp;
    char *buffer;
    long flen;
    char *fname;
    char pname[1024];

    if ( argc == 2 ) {
        fname = argv[1];
        strcpy(pname, fname);
        char *dot = strchr(pname, '.');
        while (dot != NULL) {
            *dot = '_';
            dot = strchr(pname, '.');
        }
    } else {
        printf("Filename not supplied\n");
        return 1;
    }

    fp = fopen(fname, "rb");
    fseek(fp, 0, SEEK_END);
    flen = ftell(fp);
    rewind(fp);

    buffer = (char *)malloc((flen + 1) * sizeof(char));
    fread(buffer, flen, 1, fp);
    fclose(fp);

    printf("\n//File: %s, Size: %lu\n", fname, flen);
    printf("#define %s_len %lu\n", pname, flen);
    printf("const uint8_t %s[] PROGMEM = {\n", pname);
    long i;
    for (i = 0; i < flen; i++) {
        printf(" 0x%02X", (unsigned char)(buffer[i]));
        if (i < (flen - 1)) {
            printf(",");
        }
        if ((i % 16) == 15) {
            printf("\n");
        }
    }
    printf("\n};\n\n");
    free(buffer);
    return 0;
}
```

## HTML 文件到头文件

使用方式：

 1. 使用 gzip 将 HTML 文件转换为 `.gz` 文件

```c
gzip index.html
```

 2. 编译 `filetoarray.c` 源文件，生成可执行文件

```c
gcc filetoarray.c -o filetoarray
```

 3. 使用 filetoarray 将  `.gz` 文件转为头文件

```c
./filetoarray index.html.gz > index.h
```

## HTML 文件到 Flash 

使用方式：

 1. 使用 gzip 将 HTML 文件转换为 `.gz` 文件

```c
gzip index.html
```

 2. 在工程 `main` 目录下的 `component.mk` 中添加

```c
COMPONENT_EMBED_FILES := www/index.html.gz
```

 3. 在工程源码中这样使用

```c
extern const unsigned char index_html_gz_start[] asm("_binary_index_html_gz_start");
extern const unsigned char index_html_gz_end[]   asm("_binary_index_html_gz_end");
size_t index_html_gz_len = index_html_gz_end - index_html_gz_start;

httpd_resp_set_type(req, "text/html");
httpd_resp_set_hdr(req, "Content-Encoding", "gzip");
httpd_resp_send(req, (const char *)index_html_gz_start, index_html_gz_len);
```

## 在 ESP32 中启动 HTTP Server

第一种方式：
```c
#include "index.h"

static esp_err_t index_handler(httpd_req_t *req){
    httpd_resp_set_type(req, "text/html");
    httpd_resp_set_hdr(req, "Content-Encoding", "gzip");
    return httpd_resp_send(req, (const char *)index_html_gz, index_html_gz_len);
}
```

第二种方式：
```c
static esp_err_t index_handler(httpd_req_t *req){
    extern const unsigned char index_html_gz_start[] asm("_binary_index_html_gz_start");
	extern const unsigned char index_html_gz_end[]   asm("_binary_index_html_gz_end");
	size_t index_html_gz_len = index_html_gz_end - index_html_gz_start;


    httpd_resp_set_type(req, "text/html");
    httpd_resp_set_hdr(req, "Content-Encoding", "gzip");
    return httpd_resp_send(req, (const char *)index_html_gz_start, index_html_gz_len);
}
```

```c
#include "app_httpd.h"
#include "esp_http_server.h"

void app_httpd_main() {
	httpd_handle_t camera_httpd = NULL;
    httpd_config_t config = HTTPD_DEFAULT_CONFIG();

    httpd_uri_t index_uri = {
        .uri       = "/",
        .method    = HTTP_GET,
        .handler   = index_handler,
        .user_ctx  = NULL
    };

    ESP_LOGI(TAG, "Starting web server on port: '%d'", config.server_port);
    if (httpd_start(&camera_httpd, &config) == ESP_OK) {
        httpd_register_uri_handler(camera_httpd, &index_uri);
    }
}
```

## Example

 - [web server](https://github.com/espressif/esp-who/blob/master/examples/single_chip/camera_web_server/main/app_httpd.c)
