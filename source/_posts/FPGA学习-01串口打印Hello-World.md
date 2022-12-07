title: FPGA学习--01串口打印Hello World
author: Roxy
tags:
  - FPGA
categories:
  - FPGA
  - 嵌入式
date: 2022-08-13 22:01:00
cover: 'https://s2.loli.net/2022/08/13/J9uZHKXNDkCTnM8.png'
---
# 学习FPGA

> ​	感谢wiki学长传承的FPGA，FreeRTOS的学习暂时告一段落，已经学到信号量了，但是FreeRTOS东西太多了，~~懒狗又不想干活了~~。中间穿插着学习FPGA作为一点调剂。

​	哦，这板子功能实在太强了，甚至可以外接显示器，让我先买根miniHDMI线，这个板子甚至可以跑linux来做opencv的图像处理。学习FPGA也能增强自己的硬件知识、数电知识，之后甚至可以与stm32双机处理，stm32作为任务调度，功能强大的FPGA用来进行数据处理。甚至使用FPGA搭建硬件门电路节省我们组硬件开发的时间，单纯将FPGA当做硬件电路来使用。

​	同时，还有学习使用ZYNQ 嵌入式系统的开发的目的。

## 开发环境

​			硬件开发环境：Vivado 2018.3

​			软件开发环境：SDK 2018.3

​			硬件系统：Z7-Lite 7010 版

## 开发流程

​	首先我们来了解一下一个完整的 ZYNQ 嵌入式系统的组成部分：

​		1、系统内核：ARM Cortex-A9；

​		2、存储器：DDR3 Memory Controller+存储芯片；

​		3、其他外设；

​		4、外部电路组件；

​		5、应用程序。

> ​	一个正确的ZYNQ嵌入式系统开发，应当是结合我们要完成的系统目标，逐步进行：
>
> + 第一步：创建Vivada工程
> + 第二步：使用IP核创建处理器系统
> + 第三步：生成顶层文件并导出硬件描述文件到SDK
> + 第四步：在SDK中创建应用程序
> + 第五步：在硬件电路板上验证功能

  下面，我们将按照上面介绍的内容，一步步搭建实现项目目标的嵌入式系统。

# Demo1：串口打印“Hello，World"

		“Hello World!”是各类编程教程中最经典的实验，无数软件工程师都是从“Hello World”开始自己的编程之路。我们也沿袭这一传统，从串口打印“Hello World”开始我们 ZYNQ 的学习之路。



## 硬件设计

### 创建Vivado工程

​	打开 Vivado，在 Vivado 主界面点击“Quick Start”栏的“Creat Project”，点击Next，然后命名和设置文件路径，勾选“Creat project subdirectiry”,在文件夹下创建工程子目录，点击“Next”

![](/assets/images/1005/1.png)

​	进入图示界面保持默认选项”RTL Project”即可，对于不需要添加源文件和约束文件的项目，可以勾选“Don not specify source at this time”。

![](/assets/images/1005/2.png)

​	在器件选型界面选择 ZYNQ 芯片的型号，该型号一定要和开发板上芯片的型号一致。

> **在选型界面 Family 栏里选择“Zynq_7000”系列，Speed 栏选择“-1”，在 Package 栏选择“clg400”。然后根据 ZYNQ 芯片的型号，在下面的器件列表里面选择“xc7z010clg400-1”**

​	最后选择Finish完成工程创建进入工程信息界面。

![](/assets/images/1005/3.png)

​	之后进入工程信息界面，显示该工程的一些信息，在此界面可以查看之前设置的工程名、器件型号等信息，检查是否有设置错误的地方。如果设置有误，可以点击信息栏错误的地方返回之前的设置界面重新设置。

![](/assets/images/1005/4.png)

​	工程创建完成后我们就开始系统硬件部分的设计，Vivado 提供了一个面向对象的设计工具——IP 集成器（IP intergator），在 IP 集成器中可以非常简单的添加各种官方提供或自己编写的各种功能模块（IP）,并配置相关参数。而且它支持大部分官方 IP 接口的智能自动连接、一键式 IP 子系统生成和实时 DRC 等功能，极大的提高了开发效率，降低出错的风险，即使面对复杂庞大的系统也能完成快速搭建。

​	点击左侧导航栏（Flow Navigator）中的 IP Integrator 选项下的 Creat Block Design。然后在弹出的对话框中输入所创建的 Block Design 的名称，在 Design name 栏中输入“hello_world”,点击“OK”。

![](/assets/images/1005/5.png)

### 创建IP	

​	点击界面中间的“+”或者按快捷键 Ctrl+ I，或者在空白区域点击鼠标右键，选择“ADD IP”。之后会弹出可添加的 IP 目录，在搜索栏中输入“ZYNQ”,在搜索结果中找到“ZYNQ Processing System”并双击，将该处理器系统 IP 添加到设计中。

![](/assets/images/1005/6.png)

![](/assets/images/1005/7.png)

​	添加完成后，模块 IP 出现在 Diagram 中

![](/assets/images/1005/8.png)

​	双击添加的 ZYNQ7 Processing System 模块，进入 ZYNQ 处理系统配置界面。页面左侧显示导航面板，右侧为配置信息面板。

![](/assets/images/1005/9.png)

​	点击左侧“Clock Configuration”，显示 ZYNQ PS 端的时钟频率配置页面，此处的各个时钟频率保持默认设置，因为本实验不需要用到 PL 部分，所以点开“PL Fabric Clock”，取消勾选“FCLK_CLK0”

​	除此之外，我们还要取消一些与 PL 相关设置的勾选，点击“PS-PLConfiguration”，在右侧展开 General 下的 Enable Clock Resets， 取消勾选其中的 FCLK_RESET0_N。

​	在当前界面中展开 AXI Non Secure Enablement 下的 GP Master AXIInterface，取消勾选其中的 MAXI GP0 interface。

![](/assets/images/1005/10.png)

![](/assets/images/1005/11.png)

![](/assets/images/1005/12.png)

​	“Peripheral I/O Pins”，右侧显示出引脚配置界面，在这里我们可以看到哪些引脚可以被配置成那些相应的功能。根据板子的原理图，选择MIO13、MIO14的串口，如图示，**下面图还有一个问题，右上角Bank 1里3.3V应改为LVCMOS 1.8V，原因是单纯使用串口用不到3.3V，1.8V足够了，能够降低功耗，包括前面取消三个时钟，全因为都是用不到，所以取消。**

> 这些取消还有一个原因，是为了培养好的系统设计习惯，多余的部分毫不犹豫的去掉，只保留应有的功能需要，尽可能的降低功耗。

![](/assets/images/1005/13.png)

![](/assets/images/1005/14.png)

​	点击左侧的“MIO Configuration”，在右侧展开 I/O Peripherals->UART1，因为我们使用的引脚为 MIO14 和 MIO15，属于 Bank0。在此处可以看到具体的引脚配置信息。其中 MIO15 为 TX 引脚方向为输出，MIO14 为 RX 引脚方向为输入，引脚电压为 3.3V，速度为 slow，使能上拉

![](/assets/images/1005/15.png)

​	回到PS-PL Configuration，这里用于设置串口的波特率，这里默认为115200

![](/assets/images/1005/16.png)

​	点击左侧“DDR Cfinguration”，选择 DDR 器件的型号，点开 DDR Controller Configuration 下拉按钮，在 Memory Part 一栏选择“MT41J256M16 RE-125”，总线宽度选择 16bit。

![](/assets/images/1005/17.png)

​	这样我们的配置就完成了，点击“OK”，保存配置并返回。

​	回到主界面我们会看到 ZYNQ7 模块发生了变化，对比之前发现少了部分接口，这是因为我们取消了关于 PL 的部分设置。

​	之后点击上图中的“Run Block Automation”，弹窗全选，点击“OK”，软件会自动为 IP 添加输入输出端口以及各个 IP 之间的连接，因为本实验只用到一个 IP，所以系统只引出了两组信号。

​	配置完成的 IP 如下图所示，点击接口旁边的“+”可以看到展开两组接口，查看其中具体的信号有哪些。

![](/assets/images/1005/18.png)

![](/assets/images/1005/19.png)

​	当前实验不需要添加其他 IP，快捷键 Ctrl + S 保存当前设计，点击工具栏的Validate Design 按钮（红框圈选按钮）验证当前的设计。验证完成后弹出对话框提示没有错误或者关键警告，点击“OK”

![](/assets/images/1005/20.png)

​	如果出现错误或严重警告说明之前的配置有问题，需要返回重新检查配置是否有遗漏或错误。

### 生产顶层HDL并导出

​	设计完成后需要生成顶层 HDL 模块，在 Sources 窗口，选中 Design Source下的 hello_world.bd，右键点击 hello_world.bd，在弹出的菜单中选择“Generate Output Products”

![](/assets/images/1005/21.png)

![](/assets/images/1005/22.png)

![](/assets/images/1005/23.png)

​	在对话框中 Synthesis Options 选择 Global； Run Setings 用于设置生成过程中要使用的处理器的线程数，进行多线程处理，保持默认或设置为个人电脑最大可使用线程数的一半。然后点击“Generate”来生成设计的综合、实现和仿真文件。

​	Generate 完成后，在弹出的对话框中点击“OK”。

​	在 Sources 窗 口 ， 选 中 Design Source 下 的 hello_world.bd ， 右 键 点 击hello_world.bd，在弹出的菜单中选择“Create HDL Wrapper”

​	在弹出的对话框中确认勾选“Let Vivado manage wrapper and auto-update”，然后点击“OK”。

![](/assets/images/1005/24.png)

![](/assets/images/1005/25.png)

这样我们的硬件设计就创建完成了，完成的 Design Source 结构

![](/assets/images/1005/26.png)

​	对于用到 PL 部分资源的项目需要添加引脚约束并进行综合、实现并生成Bitstream 文件。由于本实验未用到 PL 部分，所以无需上述步骤，可直接导出硬件到 SDK 即可。

​	在菜单栏选择 File > Export > Export hardware。

![](/assets/images/1005/27.png)

​	在弹出的对话框中可以选则是否包含 Bitstream 文件，因为我们没有生成Bitstream 文件，所以不勾选，点击“OK”。

![](/assets/images/1005/28.png)

​	硬件导出完成后，在菜单栏中选择 File > Launch SDK，启动 SDK 开发环境。对与弹出的对话框，直接点击OK。

![](/assets/images/1005/29.png)

​	到这里，我们就完成了系统硬件部分的设计，接下来需要到 SDK 软件中进行应用程序开发，也就是软件设计部分。

## 软件设计

​	完成硬件设计的时候我们将硬件设计导出并打开了 SDK 软件，软件界面如下

![](/assets/images/1005/30.png)

​	在编辑框中可以看到打开的 system.hdf 文件，这就是我们导出的硬件描述文件，上面显示了整个系统的地址映射信息

​	现在我们开始软件部分的设计。首先在菜单栏选择 File > New > Application Project, 新建一个 SDK 应用工程。在弹出的对话框中，输入工程名“hello_world”，其他选项保持默认，点击“Next”。

![](/assets/images/1005/31.png)

​	在弹出的对话框中选择希望使用的工程模板，这里我们选择“Hello World”，然后点击“Finish”。

![](/assets/images/1005/32.png)

​	之后 SDK 创建了一个 hello_world 应用工程和 hello_world_bsp 板级支持包（BSP）工程，同时 SDK 会自动对工程进行编译，并生成可执行 ELF 文件“hello_world.elf”

​	双击打开 hello_world/src 工程目录下 helloworld.c 文件，可以看到源代码。

![](/assets/images/1005/33.png)

​	在 SDK 中修改并保存源文件后，软件会自动对文件进行重新编译。可以在下面的控制台面板中查看编译进度，编译完成后会显示“Finished building”。另外也可以点击工具栏中的“Build All”或者快捷键 Ctrl + B 来编译工程，到此我们就完成了软件设计的工作，接下来验证功能能否实现。

## 下载验证

​	跳线帽选中 JTAG 启动，连接两根Type-C线至电脑主机，打开串口软件连接

![](/assets/images/1005/34.png)

​	连接成功后，下载程序。右键点击 hello_world 工程，在弹出的菜单栏中选择 Run as > 1 Launch on Hardware(sysntem Debugger)

![](/assets/images/1005/35.png)

​	烧录完成后程序会自动运行，可以在 Terminal 窗口看到芯片通过串口发送的字符“Hello World”

![](/assets/images/1005/36.png)

​	窗口成功打印出了“Hello World”字符串，说明我们的硬件和软件设计都是没有问题的。

# 后记

​	通过一个简单的工程，我们初步了解了一些 ZYNQ SDK 项目的开发流程和一些软件的操作方法，在此基础上，可以通过多次重复本实验熟悉各个步骤的操作流程和各选项的功能，熟悉开发环境，为以后的学习工作打下坚实的基础。

