title: CubeMX配置H743使用7寸RGB屏
author: Roxy
tags:
  - Stm32
  - CubeIDE
categories:
  - CubeMX
date: 2022-09-23 22:59:00
cover: 'https://cdn-1311941597.cos.ap-shanghai.myqcloud.com/blog/铁驭.jpg'
---
# CubeMX配置H743使用7寸RGB屏

​		咕咕咕咕咕咕咕

​		臭鸽子自从暑假重装系统以后一直在摆烂，博客也一直没写，git也好久没更新了，甚至都在给大二培训培训到stm32使用LCD了，只能说还好有个学弟正好来问7寸LCD如何配置，否则真一直摆烂下去了，还是要好好写博客来记录学习的

​		摆烂摆到甚至有些忘了Typora的快捷键怎么用，咕。



​		还是回归正题，这次教程记录如何配置使用7寸RGB屏，使用CubeMX生成代码框架，学习使用安富莱对应的LCD驱动，使用的是正点原子H743和正点厂的7寸1024*600RGB屏幕，每次声讨一遍正点的代码可读性是真的烂，以及正点的HAL库例程写的是他自己生产出来的HAL库吧？

## CubeMx配置

### 	Rcc配置

首先是很重要的Rcc配置，妖精记得将`Power Regulator Voltage Scale `  **电源稳压器电压刻度** 这个从默认的模式3改成模式1，目的是为了最终使得H743的主频能跑到400M。如果保持默认的模式3直接去设置时钟树，会出现如下报错：***”Frequency searched for is out of range“***

![报错](/assets/images/1008/image-20220923224554706.png)

> **解决方法：**
>
> + 这里提示的是MCU时钟配置的【电压域】等级不对
> + 使用其他的STM32配置时钟时，会配置【电压域】，最高主频需要配置【电压域】等级为0
> + STM32CubeMX 【RCC】配置部分，有MCU内核【电压域】的配置，这里默认等级为3，改为1
> + 【Power Regulator Voltage Scale 3】改为【Power Regulator Voltage Scale 1】

**改为验证：**

> + 发现可以正常配置系统主频了，这里我设置为主频【400MHz】

![验证结果](/assets/images/1008/image-20220923225226006.png)

### LTDC设置使用

​		这里配置使用的是RGB565(16 bits)模式，如图配置各类参数，这些参数是根据RGB的数据手册查找配置

**参数配置**

![各类参数](/assets/images/1008/image-20220923231739884.png)



**图层设置**

![图层设置](/assets/images/1008/image-20220923231828800.png)



**中断设置**

![中断设置](/assets/images/1008/image-20220923231957099.png)



**引脚配置**

​		然后是最重要的，LTDC的默认引脚不一定是对应硬件设备上的对应引脚，是故根据硬件设备调整引脚位置，如图所示。

![引脚配置](/assets/images/1008/image-20220923232323110.png)



​		届此，LTDC的配置完成

## 	DMA2D设置

​		使能DMA2D设置，如图配置

![DMA2D设置](/assets/images/1008/image-20220923232532088.png)

## 	SDRAM设置

​		选择FMC，在FMC选项中SDRAM 1中设置，如图设置

​	**FMC设置**

![SDRAM设置](/assets/images/1008/image-20220923233731350.png)



**GPIO配置**

![GPIO配置](/assets/images/1008/image-20220923233813392.png)

## 导入驱动代码

​		LCD代码使用的是以安富莱电子为基础的驱动，安富莱的代码是真的精细思路清晰，各层封装得也很到位，对着手打临摹一遍以后学习到很多

本次工程使用的驱动代码放在github上面了，这里为地址

> https://github.com/RoxyKko/CubeIDE/tree/master/H7_train/H7_Demo1_LTDC/Drivers/H7_LCD
>
> 无法翻墙使用国内端的也可使用以下这个链接去下载整个工程
>
> https://www.armbbs.cn/forum.php?mod=viewthread&tid=114858&extra=
>
> 

使用的IDE为ST推出的CubeIDE