title: Xilinx学习笔记-01
author: Roxy
tags:
  - Zynq
categories:
  - Xilinx
date: 2022-10-08 13:34:00
cover: 'https://cdn-1311941597.cos.ap-shanghai.myqcloud.com/blog/Noah.png'
---
# Xilinx学习笔记

> 
>
> **学习使用材料：正点源子领航者V2 Xilinx Zynq 7010**
>
> 

# GPIO的MIO控制LED

​		GPIO全程为**The general purpose I/O**，意为通用I/O引脚。相比于其他的I/O外设，例如UART、I2C、SPI、USB、CAN、SD SDIO、GigE等，GPIO能够模拟上述这些I/O外设，使GPIO当作模拟的那一个外设来使用，例如我们常用的stm32使用模块就常常使用GPIO模拟I2C、SPI进行通讯，但UART应该是模拟不了，因为还需要进行一个串并转换的过程。GPIO还可以与一些简单的通用外设进行联系，比如按键、LED、蜂鸣器这些并没有使用什么协议的简单的通用设备，就使用通用I/O接口GPIO进行通讯。

## MIO

> + **GPIO是一个外设，用来对器件的引脚做观测（input）以及控制（output）**

​		MIO（Multiuse I/O)，多用I/O口，Zynq中I/O外设和静态存储器接口链接到MIO处

![Zynq-7000 片上系统框图](/assets/images/1009/image-20221008112053600.png)

​		**MIO的用途是：将来自PS外设和静态存储器接口的访问多路复用到PS的引脚上。**--《DS190-Zynq-7000-Overview》P.8

​		**MIO的引脚数量有限，仅有54个引脚，所以Zynq中一般0-53的I/O口为MIO接口**

​		当然，有些时候MIO的54个引脚是不够用的，或者是电路设计方面，使用其他的引脚引出更为方便，这就要谈到EMIO了，EMIO中的E是扩展的意思，这些I/O扩展也可以通过Zynq板子上的64个input引脚、128个output引脚访问到Programmable Logic（PL）也就是FPGA部分，在上图左侧也有标识出。

> 
>
> ​		**简单来总结，MIO是连接到Zynq的PS端上，也就是链接到ARM核上，而EMIO是链接到Zynq的PL端，即FPGA核上，而MIO只有54个引脚，对应Zynq上0~53的I/O口，其他的I/O口都属于Zynq板的PL端**
>
> ​		**所以，图中的基本所有I/O外设，都可以通过MIO来使用PS引脚，或者是链接到EMIO来使用PL的引脚**
>
> ​																																															--《UG585-Zynq-7000-TRM》P.386



## I/O Banks

![7010 I/O Banks图](/assets/images/1009/image-20221008114145016.png)

​		不同的颜色代表着不同的Banks区，500、501、502代表这Zynq中PS端的Banks，500、501中除去链接到静态存储器的区，可以数出来54个区，即对应PS MIO的54个引脚，34和35区链接到PL端，相当于是FPGA的引脚。

​		其中502区域是专门用来链接DDR存储器的，链接到动态存储器。

正点原子从其他文档找到的解释为：

​		**In Zynq-7000 AP Soc devices， banks 500 and 501 contain the PS MIO pins and bank 502 contains the PS DDR pins。** 

我们也可以得知500、501、502三个区域的主要作用。

​		图片为XC7Z010 CL400/CLG400的封装的简单示意图，图片出自Zynq Pin封装文档《UG865-Zynq-7000-Pkg-Pinout》P.33

## GPIO

​		GPIO是可独立、可动态编程的，作为output输出、input输入、interrupt sensing中断模式。软件可以通过读取寄存器的方式读取GPIO的状态，或是通过写入寄存器改变GPIO的模式。GPIO寄存器的基地址是0XE000_A000。

​		GPIO被分成了4个Bank，Bank分的是寄存器组，两Bank为一组，分成了MIO组和EMIO组。**Bank0/Bank1通过MIO连接到PS，Bank2/Bank3通过EMIO来连接到PL。**

![Bank组](/assets/images/1009/image-20221008135451602.png)

​																																							图源	--《UG585-Zynq-7000-TRM》P.387

​		GPIO的软件控制是通过***一组** ***存储映射**（memory-mapped）的寄存器

**存储映射**的概念：

​		存储指操作的对象位于操作空间中，映射是指读写操作的时候对应的地址

​		Session 会话：对地址内数据读写操作

## GPIO寄存器组

​		**接下来的暂以Bank0/Bank1为解释对象，Bank2/Bank3将在EMIO部分理解**

![GPIO Channel](/assets/images/1009/image-20221008173325772.png)

​																						图源	--《UG585-Zynq-7000-TRM》P.389

​		框图分为两部分，上半部分为中断部分，在使用中断时才会引用到，下半部分如前面解释的，以Banks 0 & 1为示例，左侧下半部分的六个框图共同形成了GPIO的寄存器组，在这个大寄存器组中又如图细分为三个小组

​		如前面所记录的，GPIO可以用来对器件的引脚做观测（input）以及控制（output）

> + ***Input模式***连接到***DATA_RO***寄存器，若我们要观测器件外部引脚的高低电平状态，只需要读取DATA_RO寄存器的数值即可。
> + ***Output模式***由三个寄存器***DATA、MASK_DATA_LSW、MASK_DATA_MSW***形成的一组寄存器控制，通过DATA可以控制器件引脚上的数值。
> + ***Output Enable模式***，即输出使能模式，由两个寄存器***DIRM、OEN***寄存器组，通过这两寄存器与门控制output模式。

​		在实际编程中，我们更多的还是使用Xilinx已经封装好的库函数，这些库函数本质最终还是调用对应的寄存器，使用库函数并不需要我们一个一个去背每个寄存器的功能，不会独立的去操作一个个寄存器，而是调用封装好的函数，这也更方便我们编程。但我们还是需要对各个模块对应的寄存器有一定的了解，了解这些寄存器之间的关系有助于我们大致理解这些功能模块底层驱动之间的联系，其次是了解了底层的寄存器以后，在编写上层的代码能更有逻辑性，方便我们配置和理解封装好的函数的使用方法。
​	在实际编程中，我们更多的还是使用Xilinx已经封装好的库函数，这些库函数本质最终还是调用对应的寄存器，使用库函数并不需要我们一个一个去背每个寄存器的功能，不会独立的去操作一个个寄存器，而是调用封装好的函数，这也更方便我们编程。但我们还是需要对各个模块对应的寄存器有一定的了解，了解这些寄存器之间的关系有助于我们大致理解这些功能模块底层驱动之间的联系，其次是了解了底层的寄存器以后，在编写上层的代码能更有逻辑性，方便我们配置和理解封装好的函数的使用方法。

> + **DATA_RO**寄存器无论是将GPIO设置为input模式还是output模式，都会**一直将GPIO的状态映射出来**。以使用过的stm32为例子，无论设置GPIO为output模式还是input模式，都可以通过**HAL_GPIO_ReadPin**函数检查引脚状态。**故该寄存器可以通过软件来观测硬件电平状态。**当配置为output模式是，DATA_RO反应的即为当前output输出的电平。**此寄存器是只读寄存器**。
>
>   ​	注意：如果MIO没有被配置为GPIO引脚，那么该GPIO对应的此寄存器是不可预测的状态，呈现出随机性。
>
> + **DATA**寄存器在GPIO被配置为output模式时，该寄存器可以控制输出的数值。DATA寄存器是一个32位的寄存器，一个GPIO的Bank区最多只能控制32个GPIO，这联想到了什么？是的，DATA寄存器中32位每一位对应控制的32个不同的引脚的高低电平状态 （1/0），这点和STM32是一致的。**而32位数据要求且只能以一次性输入至DATA寄存器**。以32位为单位，不能以单一的一位作为输入，但当我们只需要改变单一GPIO的状态，即改变这寄存器的其中一位数据，我们也只能一次性输入32位的数据来更改单一位的数据。
>
>   ​	示例：0x1000_0000对应32‘b0001_0000_0000_0000_0000_0000_0000_0000，即PIN28输出高电平，其他输出低电平，若我们想使PIN0输出高电平，我们不能向DATA寄存器写入1’b1，而是写入8’h1000_0001即32‘b0001_0000_0000_0000_0000_0000_0000_0001的32位数据到DATA寄存器来使PIN28和PIN0都输出高电平，这就是一次性写入32位数据的意思，STM32同上理解。
>
>   ​	注意：当使用读取该寄存器时，**它只会返回之前写入该寄存器的值，而不是当前器件的寄存器的值**，故想读取当前引脚的状态值只能读取DATA_RO寄存器。
>
> + **MASK_DATA_LSW**寄存器，该寄存器的使用是为了改变DATA寄存器中特定的位，这里的LSW是指DATA寄存器的低16位
>
> + **MASK_DATA_MSW**寄存器，同上，MSW指高16位数据，两个寄存器的使用可以使输入DATA的32位数据的未更改位发生变化造成写入的数据出错。两个寄存器会将未进行更改的位屏蔽掉，只更改指定的值。
>
> + **DIRM**，用于控制I/O是作为输入还是输出。0：关闭输出驱动；1：使能输出驱动。
>
> + **OEN**，当I/O被配置成输出是，该寄存器用于打开/关闭输出使能。0：关闭输出驱动；1：使能输出驱动。
>
> ![LSW和MSW的写入](/assets/images/1009/image-20221008192242440.png)





## MIO按键点亮LED实验

​		这个实验我早就做过了，也出了教程，[教程地址](https://blog.roxyblog.cn/2022/08/15/FPGA%E5%AD%A6%E4%B9%A0-02%E6%8C%89%E9%94%AE%E6%8E%A7%E5%88%B6LED/)，使用方法是基本一样的，只是我之前用的是微相的mini板子，现在换了正点的，板子型号和DDR型号需要更改为对应的型号以及IO口引脚位置，其他基本是完全一致的。

附上这次更改的main.c代码

```c
/*
 * main.c
 *
 *  Created on: 2022年10月8日
 *      Author: NJY_r
 */

#include "stdio.h"
#include "xparameters.h"
#include "xgpiops.h"
#include "sleep.h"

#define GPIO_DEVICE_ID  	XPAR_XGPIOPS_0_DEVICE_ID

//核心板上PS端LED
#define MIO0_LED 0
#define MIO7_LED 7
#define MIO8_LED 8
#define MIO11_KEY 11
#define MIO12_KEY 12

XGpioPs_Config *ConfigPtr;

XGpioPs Gpio;

uint8_t LEDmode = 0;

//跑马灯
void runLED()
{
	XGpioPs_WritePin(&Gpio, MIO0_LED, 1);
	XGpioPs_WritePin(&Gpio, MIO7_LED, 0);
	XGpioPs_WritePin(&Gpio, MIO8_LED, 0);

	//延时
	usleep(200000);

	XGpioPs_WritePin(&Gpio, MIO0_LED, 0);
	XGpioPs_WritePin(&Gpio, MIO7_LED, 1);
	XGpioPs_WritePin(&Gpio, MIO8_LED, 0);

	//延时
	usleep(200000);

	XGpioPs_WritePin(&Gpio, MIO0_LED, 0);
	XGpioPs_WritePin(&Gpio, MIO7_LED, 0);
	XGpioPs_WritePin(&Gpio, MIO8_LED, 1);

	//延时
	usleep(200000);
}

//呼吸灯
void BreatLED()
{
	int a;
	int b = 401;
	for(a=1; a<b; a++)
	{
		XGpioPs_WritePin(&Gpio, MIO7_LED, 0);
		XGpioPs_WritePin(&Gpio, MIO8_LED, 1);
		usleep(a);
		XGpioPs_WritePin(&Gpio, MIO7_LED, 1);
		XGpioPs_WritePin(&Gpio, MIO8_LED, 0);
		usleep(b-a);
	}
	for(a=b; a>0; a--)
	{
		XGpioPs_WritePin(&Gpio, MIO7_LED, 0);
		XGpioPs_WritePin(&Gpio, MIO8_LED, 1);
		usleep(a);
		XGpioPs_WritePin(&Gpio, MIO7_LED, 1);
		XGpioPs_WritePin(&Gpio, MIO8_LED, 0);
		usleep(b-a+1);
	}
}



int main()
{

	printf("GPIO MIO TEST!\n\r");

	/*初始化GPIO驱动*/

	//根据器件ID查找器件配置信息
	ConfigPtr = XGpioPs_LookupConfig(GPIO_DEVICE_ID);

	//初始化GPIO驱动
	XGpioPs_CfgInitialize(&Gpio, ConfigPtr, ConfigPtr->BaseAddr);

	/*设置GPIO方向为输出 0：输入 1：输出*/
	XGpioPs_SetDirectionPin(&Gpio, MIO0_LED, 1);
	XGpioPs_SetDirectionPin(&Gpio, MIO7_LED, 1);
	XGpioPs_SetDirectionPin(&Gpio, MIO8_LED, 1);
	XGpioPs_SetDirectionPin(&Gpio, MIO11_KEY, 0);
	XGpioPs_SetDirectionPin(&Gpio, MIO12_KEY, 0);
	/*设置输出使能  0：关闭 1：打开*/
	XGpioPs_SetOutputEnablePin(&Gpio, MIO0_LED, 1);
	XGpioPs_SetOutputEnablePin(&Gpio, MIO7_LED, 1);
	XGpioPs_SetOutputEnablePin(&Gpio, MIO8_LED, 1);

	/*写数据到GPIO引脚 0：低电平 1：高电平 */
	XGpioPs_WritePin(&Gpio, MIO0_LED, 1);
	XGpioPs_WritePin(&Gpio, MIO7_LED, 1);
	XGpioPs_WritePin(&Gpio, MIO8_LED, 1);

	while(1)
	{
		if (XGpioPs_ReadPin(&Gpio, MIO11_KEY) == 0)
		{
			LEDmode++;
			LEDmode%=2;
		}
		if (XGpioPs_ReadPin(&Gpio, MIO12_KEY) == 0)
		{
			LEDmode++;
			LEDmode%=2;
		}



		if (LEDmode == 0)
		{
			printf("now is runLED\r\n");
			//跑马灯
			runLED();
		}
		else
		{
			printf("now is BreatLED\r\n");
			//呼吸灯
			BreatLED();
		}

	}

	return 0;
}

```

​		这里做了个跑马灯呼吸灯。是根据实验室培训的第一题的一部分，至于为什么没有PWM波部分，，，，，因为MIO部分这个正点板子根本没有引出外部引脚到外面，只能作罢，只有到EMIO部分才能做出来，至于按键消抖都知道怎么做，这里姑且犯个懒

![第一题培训内容](/assets/images/1009/image-20221009121756457.png)

ps：笑死了，去年我进实验室培训的时候才第一题就把LCD搞出来了，去年是真的卷

![PWM代码](/assets/images/1009/image-20221009122013055.png)

想要整个工程的可以去我GitHub的Zynq工程页的MIO_LED部分，[地址](https://github.com/RoxyKko/Zynq/tree/master/SDK/MIO_LED)



# EMIO控制LED

​	前面说过，MIO是占了0~53，而xilinx zynq7000系列，一般是有118个引脚可用的，为什么是118个IO口呢？前面也说过，GPIO被分成了4个Bank区，前两个区是控制MIO的，有54个可用。而还剩下两个bank区，一个bank区是32-bit，故可控64个IO口，四个bank区合计为32 + 22 + 32 + 32 = 118，依次编号为GPIO0~117。

​	换句话说，后两个bank区是用了PL端的资源，控制的IO称为EMIO，EMIO可用引脚为64个。MIO + EMIO一共有54 + 64 = 118个IO口可用。EMIO前的E的意思是扩展的意思，即扩展MIO的意思，其使用方法和MIO基本一致，与MIO最显著的区别为MIO是由PS端即ARM核控制，EMIO使用的是PL端即FPGA部分的资源。