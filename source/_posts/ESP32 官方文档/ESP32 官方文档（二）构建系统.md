---
title: ESP32 官方文档（二）构建系统
date: 2012-07-22 09:43:43
categories:
- ESP32 官方文档
tags:
- ESP32
---

# 构建系统

本文档解释了 Espressif 物联网开发框架构建系统和“组件”的概念.

如果您想知道如何组织新的 `ESP-IDF` 项目,请阅读本文档.

我们建议使用 [esp-idf-template](https://github.com/espressif/esp-idf-template) 项目作为项目的起点.

<!--more-->

## 使用构建系统

esp-idf 中的 `README` 文件有如何使用构建系统构建项目的说明.

## 概要

一个 `ESP-IDF` 项目可以看作是多个组件的组合.例如,对于显示当前湿度的网络服务器,可能有:

 - ESP32 基础库( libc,rom bindings 等)
 - WiFi 驱动
 - TCP / IP 协议堆栈
 - FreeRTOS 操作系统
 - Web 服务器
 - 湿度传感器驱动
 - 主程序

`ESP-IDF` 使这些组件结构更清晰并具有可配置性.为此,在编译项目时,构建环境将查找 `ESP-IDF` 目录、项目目录和(可选)其他自定义组件目录中的所有组件.之后,它允许用户使用基于文本的菜单系统去自定义每个组件来配置 `ESP-IDF` 项目.在配置完项目中的组件之后,构建程序将编译项目.

### 概念

 - “项目”是一个目录,其中包含构建单个 “app” (可执行文件)所需的文件和配置,以及其他附加文件,如:分区表,数据/文件系统分区和引导程序.
 - “项目配置”保存在项目根目录中的 `sdkconfig`  文件中.通过 `make menuconfig` 修改此文件以自定义项目配置.单个项目只包含一个项目配置.
 - “app” 是由 esp-idf 构建的可执行文件.单个项目通常会构建两个应用程序 - 一个“项目应用程序”(主要可执行文件,即您的自定义固件)和一个“引导程序”(启动项目应用程序的初始引导程序).
 - “组件”是独立代码的模块化部分,它们被编译成静态库(.a文件)并链接到应用程序.有些是由 esp-idf 本身提供的,有些则可能来自其他地方.

有些东西不是项目的一部分:

 - “ESP-IDF” 不是该项目的一部分.相反,它是独立的,并通过 `IDF_PATH` 环境变量链接到项目,该变量保存 `esp-idf` 目录的路径.这允许 IDF 框架与您的项目分离.
 - 用于编译的工具链不是项目的一部分.工具链应安装在系统命令行 `PATH` 中,或者工具链的路径设置为项目配置中编译器前缀的一部分.
 
### 示例项目

一个示例项目目录结构可能如下所示:
```
 - myProject/
             - Makefile
             - sdkconfig
             - components/ - component1/ - component.mk
                                         - Kconfig
                                         - src1.c
                           - component2/ - component.mk
                                         - Kconfig
                                         - src1.c
                                         - include/ - component2.h
             - main/       - src1.c
                           - src2.c
                           - component.mk

             - build/
```
“myProject” 示例包含以下元素:

 - 顶层项目 Makefile .此 Makefile 设置 `PROJECT_NAME` 变量,并(可选)定义项目范围的 make 变量.它包括核心的 `$(IDF_PATH)/make/project.mk` Makefile 文件,它实现了 `ESP-IDF` 构建系统的其余部分.
 - “sdkconfig” 项目配置文件.当 “make menuconfig” 运行时,将创建/更新此文件,并保存项目中所有组件的配置(包括 esp-idf 本身).“sdkconfig” 文件可能会也可能不会添加到项目的源代码管理系统中.
 - 可选的 “components” 目录包含属于项目一部分的组件.项目不必包含此类自定义组件,但它可用于构造可重用代码或包括不属于 ESP-IDF 的第三方组件.
 - “main” 目录是一个特殊的“伪组件(pseudo-component)”,它包含项目本身的源代码.“main” 是默认名称,Makefile 变量`COMPONENT_DIRS`包含此组件,但您可以修改此变量(或设置 `EXTRA_COMPONENT_DIRS`)以查找其他位置的组件.
 - “build” 目录是项目编译时创建的,包含项目编译时产生的文件.运行 make 后,该目录被创建,并包含临时目标文件和库以及最终的二进制输出文件 `bin`.此目录通常不会添加到源代码管理中,也不会随项目源代码一起发布.

组件目录包含一个组件 makefile 文件 - `component.mk`.这可能包含变量定义,以控制组件的构建过程,以及它与整个项目的集成.有关更多详细信息,请参阅[组件Makefile](#组件 Makefile).

每个组件还可以包括一个 `Kconfig` 文件,用于定义通过项目配置设置的组件配置选项.某些组件还可能包含 `Kconfig.projbuild` 和 `Makefile.projbuild` 文件,这些文件是用于覆盖项目部分的特殊文件.

### 项目 Makefile

每个项目都有一个 Makefile,其中包含整个项目的构建配置.默认情况下,项目 Makefile 可以非常小.

#### 最小示例 Makefile

```
PROJECT_NAME := myProject

include $(IDF_PATH)/make/project.mk
```

#### 强制项目变量

 - PROJECT_NAME:项目名称.二进制输出文件将使用此名称 - 即 myProject.bin, myProject.elf.
 
#### 可选项目变量

这些变量都有默认值,并可以被自定义操作覆盖.查看 `make/project.mk` 以获取所有实现细节.

 - `PROJECT_PATH`:顶级项目目录.默认为包含 Makefile 的目录.许多其他项目变量都基于此变量.项目路径不能包含空格.
 - `BUILD_DIR_BASE`:所有 objects/libraries/binaries 文件的构建输出目录.默认为`$(PROJECT_PATH)/build`.
 - `COMPONENT_DIRS`:搜索组件的目录.默认为`$(IDF_PATH)/components`( idf 组件),`$(PROJECT_PATH)/components`(项目组件),`$(PROJECT_PATH)/main` 和 `EXTRA_COMPONENT_DIRS` (其他组件).如果您不想在这些位置搜索组件,请覆盖此变量.
 - `EXTRA_COMPONENT_DIRS`:用于搜索组件的其他目录的可选列表.
 - `COMPONENTS`:要构建到项目中的组件名称列表.默认为`COMPONENT_DIRS`目录中的所有组件.
 - `EXCLUDE_COMPONENTS`:在构建过程中要排除的组件名称的可选列表.请注意,这会减少构建时间,但不会减少二进制大小.
 - `TEST_EXCLUDE_COMPONENTS`:在单元测试的构建过程中要排除的可选组件名称列表.

这些 Makefile 变量中的任何路径都应该是绝对路径.您可以使用`$(PROJECT_PATH)/ xxx`,`$(IDF_PATH)/ xxx`转换相对路径,或使用 Make 函数`$(abspath xxx)`.

这些都变量应该在 Makefile 中的 `include $(IDF_PATH)/make/project.mk` 行之前设置.

### 组件 Makefile

每个项目都包含一个或多个组件,这些组件可以是 esp-idf 的一部分,也可以从其他组件目录添加.

组件是包含 `component.mk` 文件的任何目录.

### 搜索组件

在`COMPONENT_DIRS`中的目录列表中搜索项目的组件.此列表中的目录可以是组件本身(即它们包含 `component.mk` 文件),也可以是子目录为组件的顶级目录(包含组件的目录).

运行 `make list-components` 后,会输出这些变量,这可以帮助调试组件目录是否被找到.

#### 具有相同名称的多个组件

当 esp-idf 找到所有要编译的组件时,它将按照 `COMPONENT_DIRS` 指定的顺序执行此操作; 默认情况下,首先是 idf 组件,第二个是项目组件,最后是 `EXTRA_COMPONENT_DIRS` 中的组件.如果这些目录中的两个或多个包含具有相同名称的组件子目录,则使用搜索的最后一个位置中的组件.例如,这允许通过简单地将组件从 esp-idf 组件目录复制到项目组件树然后在那里修改它来覆盖具有修改版本的 esp-idf 组件.如果以这种方式使用,esp-idf 目录本身可以保持不变.

#### 最小组件 Makefile

最小的 `component.mk` 文件是一个空文件.如果文件为空,则设置默认组件行为:

 - 与 makefile 在相同的目录中的所有源文件(`*.c`,`*.cpp`,`*.cc`,`*.S`)将被编译到组件库中
 - 子目录 “include” 将被添加到所有其他组件的全局 include 搜索路径中.
 - 组件库将链接到项目应用程序中.

有关更完整的示例组件 makefile,请参阅[示例组件 makefile](#示例组件 Makefile).

请注意,空的 `component.mk` 文件(调用默认组件构建行为)和没有 `component.mk` 文件(这意味着不会发生默认组件构建行为)之间存在差异.组件可能没有 `component.mk` 文件,如果它只包含影响项目配置或构建过程的其他文件.

#### 预设组件变量

以下特定组件的变量可在`component.mk`中使用,但不应修改:

 - `COMPONENT_PATH`:组件目录.计算包含 `component.mk` 的目录的绝对路径.组件路径不能包含空格.
 - `COMPONENT_NAME`:组件的名称.默认为组件目录的名称.
 - `COMPONENT_BUILD_DIR`:组件构建目录.计算 `$(BUILD_DIR_BASE)` 中要构建此组件源文件的目录的绝对路径.每次构建组件时,这也是当前工作目录,因此 make 等目标中的相对路径都是相对于此目录.
 - `COMPONENT_LIBRARY`:将为此组件构建的静态库文件的名称(相对于组件构建目录).默认为 `$(COMPONENT_NAME).a`.

以下变量在项目级别设置,但会导出在组件构建中使用:

 - `PROJECT_NAME`:项目名称,在项目 Makefile 中设置
 - `PROJECT_PATH`:包含项目 Makefile 的项目目录的绝对路径.
 - `COMPONENTS`:此构建中包含的所有组件的名称.
 - `CONFIG_ *`:项目配置中的每个值都有一个 make 中可用的对应变量.所有名称都以 `CONFIG_` 开头.
 - `CC`,`LD`,`AR`,`OBJCOPY`:gcc xtensa 交叉工具链中每个工具的完整路径.
 - `HOSTCC`,`HOSTLD`,`HOSTAR`:来自主机本机工具链的每个工具的全名.
 - `IDF_VER`:ESP-IDF 版本,使用 git 命令 `git describe` 从 `$(IDF_PATH)/version.txt` 文件(如果存在)中检索.这里推荐的格式是单独的一行指定主要 IDF 发布版本,例如标记版本的 `v2.0` 或任意提交的 `v2.0-275-g0efaa4f`.应用程序可以通过调用 `esp_get_idf_version()` 来使用它.
 - `PROJECT_VER`: 项目版本
	 - 如果 `PROJECT_VER` 变量在项目 Makefile 文件中设置，则将使用其值。
	 - 否则，如果 `$PROJECT_PATH/version.txt` 存在，其内容将用作 `PROJECT_VER`。
	 - 否则，如果项目位于 Git 存储库中，则将使用git describe的输出。
	 - 否则，`PROJECT_VER` 将为“1”。

如果您修改 `component.mk` 中的任何这些变量,那么这不会阻止构建其他组件,但它可能使您的组件难以构建或者调试.

#### 可选项目范围的组件变量

可以在 `component.mk` 中设置以下变量来控制整个项目中的构建设置:

 - `COMPONENT_ADD_INCLUDEDIRS`:相对于组件目录的路径,将添加到项目中所有组件的 “include” 搜索路径.如果未被覆盖,则默认`include`.如果仅需要编译此特定组件的 “include” 目录,请将其添加到 `COMPONENT_PRIV_INCLUDEDIRS`
 - `COMPONENT_ADD_LDFLAGS`:为 LDFLAGS 添加链接器参数以用于应用程序可执行文件.默认为 `-l$(COMPONENT_NAME)`.如果将预编译库添加到此目录,请将它们添加为绝对路径 `-e$(COMPONENT_PATH)/libwhatever.a`
 - `COMPONENT_DEPENDS`:应在此组件之前编译的组件名称的可选列表.对于链接时依赖性,这不是必需的,因为所有组件"include"目录始终可用.如果一个组件生成一个"include"文件,然后您想要包含在另一个组件中,则这是必要的.大多数组件不需要设置此变量.
 - `COMPONENT_ADD_LINKER_DEPS`:相对组件路径的文件的可选列表，如果它们发生更改，应触发 ELF 文件的重新链接。 通常用于链接描述文件和二进制库。大多数组件不需要设置此变量。

以下变量仅适用于属于 esp-idf 本身的组件:

 - `COMPONENT_SUBMODULES`:组件使用的 git 子模块路径(相对于 COMPONENT_PATH)的可选列表.这些将由构建过程检查(并在必要时初始化).如果组件位于 IDF_PATH 目录之外,则忽略此变量.

#### 可选特定的组件变量

可以在`component.mk`中设置以下变量来控制该组件的构建:

 - `COMPONENT_PRIV_INCLUDEDIRS`:目录路径,必须相对于组件目录,该组件目录将仅添加到此组件源文件的"include"搜索路径中.
 - `COMPONENT_EXTRA_INCLUDES`:编译组件源文件时使用的任何额外包含路径.这些将以'-I'为前缀,并按原样传递给编译器.与`COMPONENT_PRIV_INCLUDEDIRS`变量类似,但这些路径不会相对于组件目录进行扩展.
 - `COMPONENT_SRCDIRS`:目录路径,必须相对于组件目录,将用以搜索源文件(`* .cpp`,`* .c`,`* .S`).默认为'.',即组件目录本身.覆盖它以指定包含源文件的不同目录列表.
 - `COMPONENT_OBJS`:要编译的对象文件.默认值是`COMPONENT_SRCDIRS`中找到的每个源文件的 a.o 文件.覆盖此列表允许您排除`COMPONENT_SRCDIRS`中的源文件,否则将被编译.请参阅指定源文件
 - `COMPONENT_EXTRA_CLEAN`:相对于组件构建目录的路径,使用`component.mk`文件中的自定义make规则生成的任何文件,以及作为make clean的一部分需要删除的文件.有关示例,请参阅[源代码生成](#_428).
 - `COMPONENT_OWNBUILDTARGET`＆`COMPONENT_OWNCLEANTARGET`:这些目标允许您完全覆盖组件的默认构建行为.有关详细信息,请参阅[完全覆盖组件 Makefile](#完全覆盖组件 Makefile).
 - `COMPONENT_CONFIG_ONLY`:如果设置,则此标志指示组件根本不生成任何内置输出(即未构建 `COMPONENT_LIBRARY`),并忽略大多数其他组件变量.此标志用于 IDF 内部组件,其中仅包含 KConfig.projbuild 和/或 Makefile.projbuild 文件以配置项目,但没有源文件.
 - `CFLAGS`:传递给 C 编译器的标志.根据项目设置定义一组默认 `CFLAGS`.可以通过 `CFLAGS +=` 进行组件特定的添加.也可以(尽管不推荐)完全覆盖该组件的变量.
 - `CPPFLAGS`:传递给 C 预处理器的标志(用于. c , .cpp 和 .S 文件).根据项目设置定义一组默认的 `CPPFLAGS`.可以通过 `CPPFLAGS +=` 进行组件特定的添加.也可以(尽管不推荐)完全覆盖该组件的变量.
 - `CXXFLAGS`:传递给 C++ 编译器的标志.根据项目设置定义一组默认的 `CXXFLAGS`.可以通过 `CXXFLAGS +=` 进行组件特定的添加.也可以(尽管不推荐)完全覆盖该组件的变量.

要将编译标志应用于单个源文件,可以将变量覆盖添加为目标,即:

```
apps/dhcpserver.o: CFLAGS += -Wno-unused-variable
```

### 组件配置

每个组件还可以有一个 Kconfig 文件,与 `component.mk` 在同一目录下.Kconfig 中包含要添加到此组件的 “make menuconfig” 的配置设置.

运行 menuconfig 时,可在 “Component Settings” 菜单下找到这些设置.

要创建组件 KConfig 文件,最简单的方法是使用 esp-idf 中的 KConfig 文件做修改.

有关示例,请参阅[添加条件配置](#添加条件配置).

### 示例：添加二进制库、组件配置文件 Kconfig

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190131111141734.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3,size_16,color_FFFFFF,t_70)

```
#
# Component Makefile
#
ifdef CONFIG_BT_ENABLED

COMPONENT_SRCDIRS := .

COMPONENT_ADD_INCLUDEDIRS := include

# add pre-compiled libraries
LIBS := btdm_app

COMPONENT_ADD_LDFLAGS     := -lbt -L $(COMPONENT_PATH)/lib \
                           $(addprefix -l,$(LIBS))

# re-link program if BT binary libs change
COMPONENT_ADD_LINKER_DEPS := $(patsubst %,$(COMPONENT_PATH)/lib/lib%.a,$(LIBS))

COMPONENT_SUBMODULES += lib

ifeq ($(GCC_NOT_5_2_0), 1)
# TODO: annotate fallthroughs in Bluedroid code with comments
CFLAGS += -Wno-implicit-fallthrough
endif

endif

ifdef CONFIG_BLUEDROID_ENABLED

COMPONENT_PRIV_INCLUDEDIRS +=   bluedroid/bta/include                   \
                                bluedroid/bta/ar/include

COMPONENT_ADD_INCLUDEDIRS +=    bluedroid/api/include/api

COMPONENT_SRCDIRS +=    bluedroid/bta/dm                      \
                        bluedroid/bta/gatt 

ifeq ($(GCC_NOT_5_2_0), 1)
bluedroid/bta/sdp/bta_sdp_act.o: CFLAGS += -Wno-unused-const-variable
bluedroid/btc/core/btc_config.o: CFLAGS += -Wno-unused-const-variable
bluedroid/stack/btm/btm_sec.o: CFLAGS += -Wno-unused-const-variable
bluedroid/stack/smp/smp_keys.o: CFLAGS += -Wno-unused-const-variable
endif

endif

```

### 预处理器定义

ESP-IDF 构建系统在命令行上添加以下 C 预处理器定义:

 - `ESP_PLATFORM` - 可用于检测在 ESP-IDF 内发生的构建.
 - `IDF_VER` - ESP-IDF 版本,有关详细信息,请参阅[预设组件变量](https://esp-idf.readthedocs.io/en/latest/api-guides/build-system.html#preset-component-variables).
 - `PROJECT_VER`：项目版本，有关详细信息,请参阅[预设组件变量](https://esp-idf.readthedocs.io/en/latest/api-guides/build-system.html#preset-component-variables).
 - `PROJECT_NAME`：项目名称，在项目 Makefile 中设置.

### 构建过程内部

#### 顶级:Project Makefile

 - “make” 总是从项目目录和项目 makefile 运行,通常名为 Makefile.
 - 项目 makefile 设置 `PROJECT_NAME`,并可选择自定义其他可选项目变量
 - 项目 makefile 包含 `$(IDF_PATH)/make/project.mk`,其中包含项目级的 Make 逻辑.
 - `project.mk` 填写默认的项目级 make 变量,并包含项目配置中的 make 变量.如果生成的包含项目配置的 makefile 已过期,则会重新生成(通过 `project_config.mk` 中的 targets),然后 make 进程从顶部重新开始.
 - `project.mk` 根据默认组件目录或可选项目变量中设置的自定义组件列表构建需要要构建的组件列表.
 - 每个组件都可以设置一些[可选项目范围的组件变量](#可选项目范围的组件变量).这些包含在 `component_project_vars.mk` 生成的 makefile 中 - 每个组件有一个.这些生成的 makefile 包含在 `project.mk`中.如果有任何缺失或过时,它们将被重新生成(通过对组件 makefile 的递归调用),然后 make 进程从顶部重新开始.
 - 组件中的 Makefile.projbuild 文件包含在 make 进程中,以添加额外的目标或配置.
 - 默认情况下,项目 makefile 还为每个组件生成顶级构建和清理目标,并设置 app 和 clean 目标以调用这些子目标.
 - 为了编译每个组件,对组件 makefile 执行递归 make.

为了更好地理解项目构建过程,请通读 `project.mk` 文件本身.

#### 第二级:组件Makefile

 - 每次调用组件 makefile 都是通过 `$(IDF_PATH)/make/component_wrapper.mk` 包装器 makefile 进行的.
 - 此组件包装器包含所有组件 `Makefile.componentbuild` 文件,使这些文件中的任何配方,变量等可用于每个组件.
 - 调用`component_wrapper.mk`时将当前目录设置为组件构建目录,并将`COMPONENT_MAKEFILE`变量设置为`component.mk`的绝对路径.
 - `component_wrapper.mk`为所有组件变量设置默认值,然后包括可以覆盖或修改这些变量的component.mk文件.
 - 如果未定义 `COMPONENT_OWNBUILDTARGET` 和 `COMPONENT_OWNCLEANTARGET`,则会为组件的源文件和必备组件 `COMPONENT_LIBRARY` 静态库文件创建缺省构建和清除目标.
 -  `component_project_vars.mk` 文件在 `component_wrapper.mk` 中有自己的目标,如果由于组件 makefile 或项目配置的更改而需要重建此文件,则从 `project.mk` 进行评估.

为了更好地理解组件制作过程,请通读 `component_wrapper.mk` 文件和 esp-idf 中包含的一些 `component.mk` 文件.

### 以非交互方式运行

在不希望交互式提示的情况下运行 `make` 时(例如:在 IDE 或自动构建系统中)将 `BATCH_BUILD = 1` 附加到 make 参数(或将其设置为环境变量).

设置 `BATCH_BUILD` 意味着以下内容:

 - 详细输出(与 `V = 1` 相同,见下文).如果您不想要详细输出,设置 `V = 0`.
 - 如果项目配置缺少新配置项(来自新组件或 esp-idf 更新),则项目使用默认值,而不是提示用户输入每个项目.
 - 如果构建系统需要调用`menuconfig`,则会打印错误并且构建失败.

### 高级 Make 用法

- `make app`，`make bootloader`，`make partition table` 可用于仅根据需要从项目中构建 `app`，`bootloader` 或 `partition table`。
- `make erase_flash` 和 `make erase_ota` 将分别使用 `esptool.py` 从 Flash 中擦除整个 Flash 和 OTA 分区选择配置。
- `make size` 打印有关应用程序的一些大小信息。`make size-components` 和 `make size-files` 是类似的目标，分别打印更详细的每个组件或每个源文件信息。

### 调试 Make Process

调试 esp-idf 构建系统的一些技巧:

 - 将 `V = 1` 附加到 make 参数(或将其设置为环境变量)将使 make 回显所有已执行的命令,以及为 sub-make 输入的每个目录.
 - 运行 `make -w` 将导致 make 在为 sub-make 输入时回显每个目录 - 与 `V = 1` 相同但不回显所有命令.
 - 运行 `make --trace` (可能除了上述参数之一)将打印出构建时的每个目标,以及导致它构建的依赖项.
 - 运行 `make -p` 会打印每个 makefile 中每个生成的目标的(非常详细的)摘要.

有关更多调试技巧和一般制作信息,请参阅 GNU制作手册.

#### 警告未定义的变量

默认情况下,如果引用了未定义的变量(如`$(DOES_NOT_EXIST)`),构建过程将打印警告.这对于查找变量名称中的错误非常有用.

如果您不想要此行为,可以在 SDK 工具配置下的 menuconfig 顶级菜单中禁用它.

请注意,如果在 Makefile 中使用`ifdef`或`ifndef`,则此选项不会触发警告.

### 覆盖项目的部分内容

#### Makefile.projbuild

对于具有必须在顶级项目 make pass 中进行求值的构建要求的组件,可以在组件目录中创建名为 `Makefile.projbuild` 的文件.在计算 `project.mk` 时会包含此 makefile.

例如,如果您的组件需要为整个项目添加 CFLAGS (不仅仅是为了自己的源文件),那么您可以在 Makefile.projbuild 中设置 `CFLAGS +=`.

`Makefile.projbuild` 文件在 esp-idf 中大量使用,用于定义项目范围的构建功能,例如 `esptool.py` 命令行参数和 `bootloader` “特殊应用程序”.

请注意,`Makefile.projbuild` 对于最常见的组件使用不是必需的 - 例如向项目添加 include 目录,或者将 LDFLAGS 添加到最终链接步骤.可以通过 `component.mk` 文件本身自定义这些值.有关详细信息,请参阅[可选项目范围的组件变量](#可选项目范围的组件变量).

在此文件中设置变量或目标时要小心.由于这些值包含在顶级项目 makefile 中,因此它们可以影响或破坏所有组件的功能！

#### KConfig.projbuild

这相当于 `Makefile.projbuild` 的组件配置 KConfig 文件.如果要在 menuconfig 的顶层包含配置选项,而不是在 “Component Configuration” 子菜单中,则可以在 `component.mk` 文件旁边的 KConfig.projbuild 文件中定义这些选项.

在此文件中添加配置值时要小心,因为它们将包含在整个项目配置中.在可能的情况下,通常最好为组件配置创建 KConfig 文件.

#### Makefile.componentbuild

对于组件例如,包括从其他文件生成源文件的工具,必须能够将配置,宏或变量定义添加到每个组件的组件构建过程中.这是通过在组件目录中包含 Makefile.componentbuild 来完成的.在包含组件的 component.mk 之前,此文件会包含在 component_wrapper.mk 中.与 Makefile.projbuild 类似,请注意这些文件:因为它们包含在每个组件构建中,所以只有在编译完全不同的组件时才会出现 Makefile.componentbuild 错误.

#### 仅配置组件

一些不包含源文件的特殊组件,只有 `Kconfig.projbuild` 和 `Makefile.projbuild`,可以在 component.mk 文件中设置标志 `COMPONENT_CONFIG_ONLY`.如果设置了此标志,则忽略大多数其他组件变量,并且不会为组件运行构建步骤.

### 示例组件 Makefile

因为构建环境试图设置大多数时间都能工作的合理默认值,所以 component.mk 可能非常小甚至是空的(请参阅[最小组件 Makefile](#最小组件Makefile)).但是,某些功能通常需要覆盖组件变量.

以下是`component.mk` makefile 的一些更高级的示例:

#### 添加源文件目录

默认情况下,将忽略子目录.如果您的项目在子目录而不是组件的根目录中有源文件,那么您可以通过设置` COMPONENT_SRCDIRS` 告诉构建系统:
```
COMPONENT_SRCDIRS:= src1 src2
```
这将编译 src1/ 和 src2/ 子目录中的所有源文件.

#### 指定源文件

标准 component.mk 逻辑将源目录中的所有 .S 和 .c 文件添加为无条件编译的源.通过将 `COMPONENT_OBJS` 变量手动设置为需要生成的对象的名称,可以绕过该逻辑并对要编译的对象进行硬编码:
```
COMPONENT_OBJS := file1.o file2.o thing/filea.o thing/fileb.o anotherthing/main.o
COMPONENT_SRCDIRS := . thing anotherthing
```
请注意,还必须设置 `COMPONENT_SRCDIRS`.

#### 添加条件配置

配置系统可有条件地编译某些文件,具体取决于 `make menuconfig` 中选择的选项.为此, ESP-IDF 具有 `compile_only_if` 和 `compile_only_if_not` 宏:

`Kconfig`:
```
config FOO_ENABLE_BAR
    bool "Enable the BAR feature."
    help
        This enables the BAR feature of the FOO component.
```
`component.mk`:
```
$(call compile_only_if,$(CONFIG_FOO_ENABLE_BAR),bar.o)
```
从示例中可以看出,`compile_only_if` 宏将条件和目标文件列表作为参数.如果条件为真(在这种情况下:如果在 menuconfig 中启用了 BAR 功能),将始终编译目标文件(在本例中为 bar.o).相反的情况也是如此:如果条件不成立, bar.o 将永远不会被编译.`compile_only_if_not` 执行相反的操作:如果条件为false则编译,如果条件为 true 则不编译.

这也可用于选择或删除一种实现,如下所示:

`Kconfig`:
```
config ENABLE_LCD_OUTPUT
    bool "Enable LCD output."
    help
        Select this if your board has a LCD.

config ENABLE_LCD_CONSOLE
    bool "Output console text to LCD"
    depends on ENABLE_LCD_OUTPUT
    help
        Select this to output debugging output to the lcd

config ENABLE_LCD_PLOT
    bool "Output temperature plots to LCD"
    depends on ENABLE_LCD_OUTPUT
    help
        Select this to output temperature plots
```

component.mk:

```
# If LCD is enabled, compile interface to it, otherwise compile dummy interface
$(call compile_only_if,$(CONFIG_ENABLE_LCD_OUTPUT),lcd-real.o lcd-spi.o)
$(call compile_only_if_not,$(CONFIG_ENABLE_LCD_OUTPUT),lcd-dummy.o)

#We need font if either console or plot is enabled
$(call compile_only_if,$(or $(CONFIG_ENABLE_LCD_CONSOLE),$(CONFIG_ENABLE_LCD_PLOT)), font.o)
```
请注意使用 Make 'or' 功能来包含字体文件.其他替换函数,如 'and' 以及 'if' 也适用于此处.也可以使用不来自 menuconfig 的变量: ESP-IDF 使用默认的构建配置来判断一个空的变量或只包含空格为false,而其中包含任何非空格的变量为true.

(注意:本文档的旧版本建议有条件地将目标文件名添加到 `COMPONENT_OBJS`.虽然这仍然可行,但只有当组件的所有目标文件都明确命名时才会起作用,并且不会通过 `make clear` 中的取消选择的目标文件通过.)

#### 源代码生成

某些组件将出现源文件未随组件本身提供但必须从另一个文件生成的情况.假设我们的组件有一个头文件,该文件由 BMP 文件的转换后的二进制数据组成,使用名为 bmp2h 的假设工具进行转换.然后将头文件包含在名为 graphics_lib.c 的 C 源文件中:
```
COMPONENT_EXTRA_CLEAN := logo.h

graphics_lib.o: logo.h

logo.h: $(COMPONENT_PATH)/logo.bmp
    bmp2h -i $^ -o $@
```
在此示例中,将在当前目录(构建目录)中生成 `graphics_lib.o` 和 `logo.h`,而 logo.bmp 随组件一起提供并位于组件路径下.因为 logo.h 是一个生成的文件,所以当调用 make clean 时需要清理它,这就是为什么它被添加到 `COMPONENT_EXTRA_CLEAN` 变量中.

#### Cosmetic Improvements

因为 logo.h 是一个生成的文件,所以当调用 make clean 时需要清理它,这就是为什么它被添加到 `COMPONENT_EXTRA_CLEAN` 变量中.

将 logo.h 添加到 `graphics_lib.o` 依赖项会导致在编译 `graphics_lib.c` 之前生成它.

如果另一个组件中的源文件包含 `logo.h`,则必须将此组件的名称添加到另一个组件的 `COMPONENT_DEPENDS` 列表中,以确保组件按顺序构建.

#### 嵌入二进制数据

有时您有一个文件,包含组件需要使用的二进制数据或文本数据 - 但您不希望将文件重新格式化为 C 文件.

您可以在 component.mk 中设置变量 `COMPONENT_EMBED_FILES`,以这种方式给出要嵌入的文件的名称:
```
COMPONENT_EMBED_FILES:= server_root_cert.der
```
或者,如果文件是字符串,则可以使用变量 `COMPONENT_EMBED_TXTFILES`.这将把文本文件的内容嵌入为以 null 结尾的字符串:
```
COMPONENT_EMBED_TXTFILES:= server_root_cert.pem
```
文件的内容将被添加到 flash 中的 .rodata 部分,并通过符号名称提供,如下所示:
```
extern const uint8_t server_root_cert_pem_start [] asm(“_ binary_server_root_cert_pem_start”);
extern const uint8_t server_root_cert_pem_end [] asm(“_ binary_server_root_cert_pem_end”);
```
名称是根据文件的全名生成的,如 `COMPONENT_EMBED_FILES` 中所示.字符 `/`,`.`等用下划线代替.符号名称中的 `_binary` 前缀由 `objcopy` 添加,对于文本和二进制文件都是相同的.

有关使用此技术的示例,请参阅[protocols/https_request](https://github.com/espressif/esp-idf/tree/be81d2c/examples/protocols/https_request)-证书文件内容在编译时从文本 .pem 文件加载.

### 完全覆盖组件 Makefile

显然,在某些情况下,所有这些不足以满足某个组件,例如,当组件基本上是另一个第三方组件的包装器时,该第三方组件最初不打算在此构建系统下编译.在这种情况下,可以通过设置 `COMPONENT_OWNBUILDTARGET` 和可能的 `COMPONENT_OWNCLEANTARGET` 并在 `component.mk` 目标中定义名为 `build` 和 `clean` 的自己的目标来完全放弃 esp-idf 构建系统.构建目标可以执行任何操作,只要它为项目生成过程创建 $(COMPONENT_LIBRARY) 以链接到应用程序二进制文件.

(实际上,即使这不是必需的-如果重写 `COMPONENT_ADD_LDFLAGS` 变量,则组件可以指示链接器链接其他二进制文件.)

### 自定义 sdkconfig 默认值

例如,您不希望指定完整 sdkconfig 配置的项目或其他项目,但您确实希望覆盖 esp-idf 默认值中的某些键值,则可以在项目目录中创建文件 `sdkconfig.defaults`.运行 `make defconfig` 或从头创建新配置时将使用此文件.

要覆盖此文件的名称,请设置 `SDKCONFIG_DEFAULTS` 环境变量.

### 保存 flash 参数

在某些情况下,我们希望在没有 IDF 的情况下烧写目标板.对于这种情况,我们希望保存构建的二进制文件, `esptool.py` 和 `esptool write_flash` 参数.编写脚本以保存二进制文件和 `esptool.py` 很简单.我们可以使用命令 `make print_flash_cmd`,它会打印 flash 参数:
```
--flash_mode dio --flash_freq 40m --flash_size detect 0x1000 bootloader / bootloader.bin 0x10000 example_app.bin 0x8000 partition_table_unit_test_app.bin
```
然后使用 flash 参数作为 `esptool write_flash` 参数的 arguemnts:
```
python esptool.py --chip esp32 --port / dev / ttyUSB0 --baud 921600 - before default_reset - after hard_reset write_flash -z --flash_mode dio --flash_freq 40m --flash_size detect 0x1000 bootloader / bootloader.bin 0x10000 example_app .bin 0x8000 partition_table_unit_test_app.bin
```

## 构建 Bootloader

引导程序默认构建为 “make all” 的一部分,或者可以通过 “make bootloader-clean” 独立构建.还有 “make bootloader-list-components” 来查看引导加载程序构建中包含的组件.

IDF `components/bootloader` 中的组件是特殊的,因为第二阶段引导加载程序是主项目的单独.ELF和.BIN文件.但是,它与主项目共享其配置和构建目录.

这是通过在 `components/bootloader/subproject` 下添加子项目来完成的.这个子项目有自己的 Makefile,但它希望通过 `components/bootloader/Makefile.projectbuild` 文件中的一些粘合剂从项目自己的 Makefile 中调用.有关详细信息,请参阅这些文件

[原文链接](https://docs.espressif.com/projects/esp-idf/en/latest/api-guides/build-system.html)
