---
title: 使用VSCode插件EmbeddedIDE原生开发STM32
description: 本文为STM32的开发环境搭建教程，使用VSCode与EmbeddedIDE插件，面向从零搭建框架的原生开发方式
date: 2025-09-04 01:18:00 +0800
categories: [嵌入式]
tags: [STM32, VSCode, EmbeddedIDE]
---

## 下载STM32软件包

首先在ST官网依次点击**工具与软件**，**嵌入式软件**，**STM32嵌入式软件**，**STM32Cube MCU和MPU包**，选择你所用的MCU产品系列，进入详情页面下载。这里需要注意的是小版本更新是单独作为补丁包发布的，以G0系列为例，目前最新版本是1.6.2，那么你需要下载1.6.0的大包和1.6.2的补丁包。

![download_site](/assets/img/2025/download_site.png)

下载后，先解压大版本的包，再解压小版本的补丁包，将大版本的文件覆盖。

## STM32软件包内容概略

解压完成后可以看到里面有很多内容，根目录下是一些说明文件及发布信息，还有若干文件夹。

- **Documentation**文件夹里有上手指南，可以作为参考。
- **Drivers**就是我们需要的软件包内容，包含HAL库及CMSIS库。
- **Middlewares**是中间件，如USB、RTOS等，
- **Projects**是针对ST官方开发板适配的示例工程文件，也有各个IDE项目模板可直接复用。
- **Utilities**是一些实用工具，目前ST似乎会把IDE的一些开发包也帮你打包好了，也可以在Keil里面的包管理页面直接下载。

**Drivers**文件夹里面有**BSP**，**CMSIS**，**HAL库**。

- **BSP**是适配ST官方开发板的板级支持包，一般不需要。
- **CMSIS**是Arm内核相关的软件包。
- **xxx_HAL_Driver**就是我们需要的HAL库了。

## 新建项目文件夹

首先建立项目内的文件夹层级，以下为我使用的项目结构：

- **项目名/Driver**：此处存放驱动包代码，包括CMSIS库，HAL库，以及自己开发的驱动隔离层HDL（Hardware Driver Layer）与基础代码库EFL（Embedded Foundation Library）
- **项目名/Middleware**：此处存放中间件代码，包括自建的和第三方的，自建的如板级模块支持、一些底层无关的功能模块，第三方文件夹可以放功能模块、RTOS等内容。分别命名为**Device**、**Module**和**ThirdParty**。
- **项目名/User**：此处存放算法层以及应用层代码。
- **项目名/Project**：此处存放不同IDE的工程文件夹，比如我会使用EIDE进行开发，需要调试时使用Keil调试（EIDE可以生成Keil工程文件），EIDE和Keil放在独立文件夹中。
- **项目名/**：根目录下存放说明文档等文件。

## 复制软件包文件

将**CMSIS**和**STM32xxxx_HAL_Driver**复制到**项目名/Driver**下，并进行裁剪。

**CMSIS**文件夹里有多个文件夹：

- **docs**是帮助文档，打开**index.html**即可查看。
- **Core**和**Core_A**分别是Cortex-M和Cortex-A系列的Arm内核头文件和相关模板，我们不需要这两个文件夹，而是直接使用**Include**文件夹即可，如果了解得深的话可以对**Include**文件夹中进行精简。此文件夹**必选**。
- **Device**是ST添加的适配不同STM32系列的头文件，**Device/ST/STM32xxxx/**下的**Include**和**Source**必须复制，**Source**里面从**Template**文件夹中选择你所用编译器的启动汇编文件，以及**system_stm32xxxx.c**，复制到**Template**外面，然后将**Template**文件夹删除。此文件夹**必选**。
- **DSP**是用于数字信号处理的软件包，提供了一系列数字处理相关的函数，针对不同的内核会有不同的实现，如在Cortex-M0+中会使用朴素实现，而在支持浮点运算指令及MAC指令的Cortex-M4中会使用支持的指令实现。我们需要复制**Include**和**Lib**文件夹，部分软件包可能将**Lib**文件夹放在外面，其他的文件夹为帮助说明文件、模板文件、示例工程、实现源文件。其中**Lib**文件夹中需要根据你所用的编译器和字节序选择相应的静态链接库，或者也可复制**Source**文件夹自己编译。此文件夹**可选**。
- **NN**是用于神经网络训练的支持软件包，此文件夹**可选**。
- **RTOS**和**RTOS2**是支持实时操作系统的相关模板文件，具体区别参考帮助文档，此文件夹**可选**。

**STM32xxxx_HAL_Driver**下也有多个文件夹，复制**Inc**和**Src**文件夹，以及**xxx_User_Manual**帮助文件。

- **Inc**文件夹中有2个名字带Template的文件，**stm32_assert_template.h**和**stm32g0xx_hal_conf_template.h**，用于断言和库配置，将template后缀删掉（文件中的template注释也可删除），然后复制到**项目名/User**下。
- **Src**文件夹中有4个名字带Template的文件，都可以删除。

最后还有一个内核中断的实现文件**stm32xxxx_it.c**，可以自己写，也可以从前面说的**Projects**文件夹中的模板工程里面复制。

## 新建EIDE工程

点击**新建项目**-**空项目**-**Cortex-M项目**，输入项目名，选择保存位置为**项目名/Project**，完成后将VSCode关闭，将生成的文件夹改名为**EIDE**。文件夹中的.code-workspace文件就是EIDE的工程文件。

这里推荐先阅读下EIDE的[官方文档](https://em-ide.com/docs/intro)。

### 项目资源

将**Driver**、**Middleware**和**User**添加为普通文件夹，如果复制了DSP的静态链接库的话，添加一个虚拟文件夹**Lib**，并添加对应的静态链接库文件。

### 构建配置

点击EIDE面板中的构建配置右边的箭头，选择你使用的编译器，编译器路径需要在VSCode插件设置中指定，这里选择AC6。

构建配置菜单下的**CPU类型**选择相应的内核，我使用的是STM32G0系列，这里选择Cortex-M0+。

**RAM/FLASH布局**中根据你所用的具体型号配置，RAM和ROM的起始地址在数据手册/参考手册中可以查到，一般RAM是`0x20000000`，ROM是`0x08000000`。然后RAM和ROM的大小也是根据你的具体型号配置。配置后点击Save保存。

**构建器选项**用于配置编译器选项，**C/C++编译器**界面，勾选**One ELF Section per Function**、**Plain Char is Signed**和**enum/wchar is short type**。全局编译选项可以增加-fno-exceptions，用于禁止C++异常。其他选项不动，点击保存。

### 烧录配置

根据你所用的下载器选择即可。

### C/C++属性

包含目录增加：

- ../../User
- ../../Driver
- ../../Driver/CMSIS/DSP/Include（可选）
- ../../Driver/CMSIS/Device/ST/STM32xxxx/Include
- ../../Driver/CMSIS/Include
- ../../Driver/STM32xxxx_HAL_Driver/Inc
- ../../Middleware

预处理器定义增加：

- USE_FULL_ASSERT
- USE_HAL_DRIVER
- DEBUG
- STM32G070xx（根据你所用型号，这个定义可以在CMSIS/Deivce/ST/STM32xxxx/Include/stm32xxxx.h中找到）

### 项目目标

项目目标（Target）用于存放不同的项目配置，默认为Debug，我们再新建一个Release的配置。右键EIDE面板中的项目名称，点切换目标，然后点+号，输入Release回车。

在Release配置下，将**C/C++属性-预处理器定义**中的DEBUG和USE_FULL_ASSERT删除，将**构建配置-构建器选项**中，**C/C++编译器**界面下，勾选**Link Time OPtimization(LTO)**，代码优化级别选择level-3或level-balanced。

## 完成

现在就可以使用VSCOde+EIDE开发STM32了，编译的快捷键是F7，烧录的快捷键是Ctrl+Alt+D。对EIDE面板项目名右键可以导出Keil项目，将其放到**项目名/Project/Keil**下，第一次打开需要设置相应的项目配置，调试时可以使用Keil，或者使用VSCode也可以调试，具体参考EIDE官方文档。

## 如何切换项目至其他型号

此项目可以作为模板使用，如果是不同系列的芯片，需要按照上述步骤导入一遍，如果是同系列的不同型号芯片，按照下列步骤执行：

1. 更改工程名称
2. 更改预处理器定义中的型号宏定义（如STM32G070xx），该宏定义可在CMSIS/Deivce/ST/STM32xxxx/Include/stm32xxxx.h中找到
3. 把CMSIS/Deivce/ST/STM32xxxx/Source/下的启动文件(.s)更改为对应型号文件
4. 根据芯片规格配置正确的RAM和ROM大小
5. 修改代码中关于时钟树的配置
