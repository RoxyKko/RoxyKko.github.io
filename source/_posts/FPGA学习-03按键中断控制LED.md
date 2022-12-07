title: FPGA学习--03按键中断控制LED
author: Roxy
tags:
  - FPGA
categories:
  - FPGA
  - 嵌入式
cover: >-
  https://cdn-1311941597.cos.ap-shanghai.myqcloud.com/blog/100412674_p0_master1200.webp
date: 2022-08-15 20:05:00
---
# 学习FPGA part.3

​		在之前的实验中，我们用轮询的方式，不停的扫描按键 I/O 口的状态来控制 LED 的亮灭，这种方式对于简单小型的项目还可以接受，但对于一些比较复杂的项目而言不仅效率低下而且浪费了大量的系统资源，严重影响系统的性能。这就需要一种更加高效的方式来处理一些突发事件。中断作为系统最重要的组成部分之一就承担了这样的工作，它在节省了大量的系统资源的同时监控一些突发事件的产生，及时向系统申请暂停主程序转而处理突发事件，大大提高了系统运行的效率。中断通常由硬件处理单元、外部设备或者软件本身产生。

# 中断控制LED

## 简介

​		本实验我们通过实现按键中断控制 LED 亮灭来了解 ZYNQ 的中断系统，学习如何使用外部输入中断，了解中断配置的流程、相关函数的作用以及各参数的含义

​		本实验将按键按下作为中断触发信号，检测到中断之后改变 LED 的状态作为检测到中断信号的指示。

## 硬件设计

​		打开开发板的原理图可以看到按键部分的电路设计如下图所示，按键未按下时 I/O 口为高电平，按下后变为低电平，过程中会产生一个下降沿信号。芯片的外部中断可以分为边沿触发和电平触发，我们本实验选取按键的下降沿信号作为触发信号。

![按键电路](/assets/images/1007/1.png)

​		我们按照之前的步骤建立一个新的 Vivado 工程，工程名为“mio_interrupt”，

![建立“mio_interrupt”工程](/assets/images/1007/2.png)

​		建立完成后创建“Block Dseign”，命名为“mio_interrupt”，在其中添加 ZYNQ7 Processing System IP 核，IP 核的配置与之前的实验完全相同，在此不再赘述，请参照之前的实验进行配置

​		配置完成后保存并验证设计，没有错误就可以导出硬件描述，生成顶层 HDL，

![导出硬件描述](/assets/images/1007/3.png)

![生成顶层 HDL](/assets/images/1007/5.png)

![生成顶层 HDL](/assets/images/1007/4.png)

## 软件设计

> ZYNQ 系统中断的配置流程：
>
> 类似于之前配置 GPIO 的操作，使用该功能之前需要先通过查找 ID 号加载该功能的驱动器并初始化该驱动器，之后配置相关设置并使能。在本实验中我们用到的有 GPIO 功能，通用中断控制功能。

实验设计流程：

​	1、通过查找 GPIO 驱动器 ID 加载驱动器并初始化 GPIO；

​	2、设置 KEY 所连接 MIO 的方向为输入，LED 所连接的 MIO 的方向为输出并使能输出；

​	3、查找并初始化中断控制器；

​	4、设置并使能中断；

​	5、设置中断处理函数；

​	6、设置中断类型为 GPIO中断，触发方式为下降沿中断，使能中断源；

​	7、处理中断事件，反转 LED 电平。

​		**在打开的 SDK软件内新建一个空白工程命名为“mio_interrupt”**

![新建空白工程](/assets/images/1007/6.png)

​		因为我们创建的是一个空白的工程，所以工程中只包含板级支持包、库函数核驱动文件等内容而没有主程序，所以我们要手动在程序中添加主程序文件。右 键 点 击 mio_interrupt 工 程 下面 的 src 文 件 夹， 在 弹 出 的选 项 栏 中 选 择New->Source File

主代码：

```
/******************************************************************************
*
* Copyright (C) 2009 - 2014 Xilinx, Inc.  All rights reserved.
*
* Permission is hereby granted, free of charge, to any person obtaining a copy
* of this software and associated documentation files (the "Software"), to deal
* in the Software without restriction, including without limitation the rights
* to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
* copies of the Software, and to permit persons to whom the Software is
* furnished to do so, subject to the following conditions:
*
* The above copyright notice and this permission notice shall be included in
* all copies or substantial portions of the Software.
*
* Use of the Software is limited solely to applications:
* (a) running on a Xilinx device, or
* (b) that interact with a Xilinx device through a bus or interconnect.
*
* THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
* IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
* FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL
* XILINX  BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY,
* WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF
* OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
* SOFTWARE.
*
* Except as contained in this notice, the name of the Xilinx shall not be used
* in advertising or otherwise to promote the sale, use or other dealings in
* this Software without prior written authorization from Xilinx.
*
******************************************************************************/

/*
 * helloworld.c: simple test application
 *
 * This application configures UART 16550 to baud rate 9600.
 * PS7 UART (Zynq) is not initialized by this application, since
 * bootrom/bsp configures it to baud rate 115200
 *
 * ------------------------------------------------
 * | UART TYPE   BAUD RATE                        |
 * ------------------------------------------------
 *   uartns550   9600
 *   uartlite    Configurable only in HW design
 *   ps7_uart    115200 (configured by bootrom/bsp)
 */

#include <stdio.h>
#include "platform.h"
#include "xil_printf.h"
#include "xparameters.h"
#include "xgpiops.h"
#include "xscugic.h"
#include "xil_exception.h"
#include "xplatform_info.h"
#include "sleep.h"

#define GPIO_DEVICE_ID		XPAR_XGPIOPS_0_DEVICE_ID		//PS端GPIO器件ID
#define INIC_DEVICE_ID		XPAR_SCUGIC_SINGLE_DEVICE_ID	//通用中断控制器ID
#define GPIO_INTERRUPT_ID	XPAR_XGPIOPS_0_INTR				//PS端GPIO中断ID

#define KEY1 9		//KEY1 连接至 MIO9
#define LED1 0		//LED1 连接至 MIO0

static void intr_handler(void *callback_ref);
int init_interrupt(XScuGic *gic_ins_ptr ,XGpioPs *gpio ,u16 GpioIntrId);

XGpioPs Gpiops;
XScuGic intc;

u32 key_flag;	//按键中断标志
u32 key_val;	//控制LED亮灭

int main()
{
    int status;
    XGpioPs_Config *ConfigPtr;

    print("mio interrupt Test\n\r");

    ConfigPtr = XGpioPs_LookupConfig(GPIO_DEVICE_ID);
    if (ConfigPtr == NULL){
    	return XST_FAILURE;
    }
    //初始化GPIO
    XGpioPs_CfgInitialize(&Gpiops ,ConfigPtr ,ConfigPtr->BaseAddr);

    //设置KEY引脚方向为输入
    XGpioPs_SetDirectionPin(&Gpiops ,KEY1 ,0);

    //设置LED引脚方向为输出并使能输出
    XGpioPs_SetDirectionPin(&Gpiops ,LED1 ,1);
    XGpioPs_SetOutputEnablePin(&Gpiops ,LED1 ,1);
    XGpioPs_Write(&Gpiops ,LED1 , 1);

    //建立中断
    status = init_interrupt(&intc ,&Gpiops ,GPIO_INTERRUPT_ID);
    if (status != XST_SUCCESS){
    	print("Setup interrupt system failed\r\n");
    	return status;
    }
    while(1)
    {
    	if(key_flag){					//触发中断
    		usleep(20000);				//延时消抖
    		if(XGpioPs_ReadPin(&Gpiops,KEY1) == 0){	//低电平按下
    			key_val = ~key_val;
    			XGpioPs_WritePin(&Gpiops,LED1,key_val);		//反转LED电平
    		}
    		key_flag = FALSE;
    		XGpioPs_IntrClearPin(&Gpiops,KEY1);		//清除中断标志
    		XGpioPs_IntrEnablePin(&Gpiops,KEY1);	//再次使能中断
    	}
    }


    return XST_SUCCESS;
}

//中断处理函数
static void intr_handler(void *callback_ref)
{
	XGpioPs *gpio = (XGpioPs *) callback_ref;

	//读取中断标志
	if (XGpioPs_IntrGetStatusPin(gpio ,KEY1)){
		key_flag = TRUE;
		XGpioPs_IntrDisablePin(gpio ,KEY1);	//屏蔽按键KEY中断
	}
}

//初始化中断系统，使能KEY1按键的下降沿中断
int init_interrupt(XScuGic *gic_ins_ptr ,XGpioPs *gpio ,u16 GpioIntrId)
{
	int status;
	XScuGic_Config *IntcConfig;

	//初始化中断控制器
	IntcConfig = XScuGic_LookupConfig(INIC_DEVICE_ID);
	if (IntcConfig == NULL){
		return XST_FAILURE;
	}

	status = XScuGic_CfgInitialize(gic_ins_ptr,IntcConfig,IntcConfig->CpuBaseAddress);
	if (status != XST_SUCCESS){
		return XST_FAILURE;
	}

	//设置并使能中断
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
			(Xil_ExceptionHandler) XScuGic_InterruptHandler,gic_ins_ptr);
	Xil_ExceptionEnable();

	//设置中断处理函数
	status = XScuGic_Connect(gic_ins_ptr,GpioIntrId,
			(Xil_ExceptionHandler) intr_handler,(void *) gpio);
	if (status != XST_SUCCESS) {
		return status;
	}

	//使能中断类型为来自GPIO的中断
	XScuGic_Enable(gic_ins_ptr , GpioIntrId);

	//设置中断触发方式为下降沿触发
	XGpioPs_SetIntrTypePin(gpio,KEY1,XGPIOPS_IRQ_TYPE_EDGE_FALLING);

	//使能中断源
	XGpioPs_IntrEnablePin(gpio,KEY1);

	return XST_SUCCESS;
}

```

​		输入完成后按快捷键 Ctrl + S 保存 main.c 文件，软件会自动进行编译，编译过程可以编辑框下面的控制台查看，同时如果软件有错误的话也会在里面显示出来。

## 下载

​		编译完成后我们就可以将程序烧录到开发板上进行验证，按照之前的方式连 接 好 JTAG 下 载 接 口 和 串 口， 在 Terminal 窗 口 连 接 串 口 ， 在 应 用 工 程gpio_mio_interrupt 上右击，选择“Run As”，然后选择第一项“1 Launch on Hardware (System Debugger)”烧录程序。烧录完成后可以在 Terminal 窗口看到打印的信息“mio interrupt”，说明程序已经运行。