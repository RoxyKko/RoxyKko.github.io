title: 学习使用FreeRTOS系统
author: Roxy
tags:
  - STM32
  - FreeRTOS
cover: 'https://s2.loli.net/2022/08/08/YIxrAof6KdzJiPs.png'
categories:
  - Stm32
  - FreeRTOS
date: 2022-08-08 17:21:00
---
# 学习FreeRTOS实时操作系统



## 总述

>其实如果只是参加电赛的话，其实用不到FreeRTOS，仅仅是裸机代码即可，只是FreeRTOS作为一种嵌入式开发技术，我认为有学习的必要，故开此连载记录自己的学习。



​	在学习stm32之前，已经接触过windows系统、Andriod系统的我已经对多线程多任务操作习以为常，并认为这本来就是理所应当的，但当对stm32有了一点点自己的了解后，才知道微机系统单核CPU同时只能进行一个任务。要想对多功能的实现，需要细心去安排各功能的时序和逻辑顺序，总的来说设计程序的运行顺序也是一种乐趣，但经常会有苦恼，要是能像window、linux、andriod等等那样能设置多进程任务就好了，能方便很多。这个时候便开始寻找解决方法了，找到的一个方法便是RTOS系统。

>   在学习MCU编程时，一般都是裸机编程，一般的程序设计，如果是功能要求不太复杂或者程序设计比较精良，裸机系统也能很好的实现功能，但如果功能要求较多，就需要分解成多个任务来实现，或者是对实时性要求较高的时候，这就是RTOS该出现的时候了。
>
>   ​	同时，使用RTOS并且将功能分解成多个任务，使得程序功能模块化，程序结构变得简单，就能更加便于维护和扩展，以及我的团队是由两位软件组队的，这个的学习也便于我的团队协作开发。
>
>   ​	还有一个目的便是，考虑到未来出去工作，对市面上各类嵌入式操作系统和GUI框架总要进行学习的，能越早学会越好

​	接下来将会使用CubeMX + CubeIDE相配合，进行对STM32的FreeRTOS系统的学习使用。

# 创建第一个FreeRTOS工程

使用CubeMX生成FreeRTOS框架

### RCC和SYS设置

![sys](/assets/images/1002/1.png)

SYS中更改Debug选项，并同时设置一个独立的定时器来作为HAL基础时钟源，原因是在不使用FreeRTOS时，Timebase Source默认是SysTick。在使用FreeRTOS时，FreeRTOS会将SysTick作为基础时钟，所以需要在设置一个定时器作为HAL库的基础时钟。

> 若不设置，在使能FreeRTOS后生成代码时会跳出警告。在设置之后TIM6和SysTick两个定时器的周期都会自动设置为1ms。
>
> 这样配置后HAL库基础定时器是TIM6，FreeRTOS的基础时钟为SysTick定时器。

时钟树配置按以往常规设置时配置相同，拉满至168MHz

## 使能FreeRTOS

​	在CubeMX的Middleware栏，第二个即为全大写的FreeRTOS，选择最新的版本CMSIS_V2，同时设置两个LED引脚为推挽输出OUUPUT_PP，上拉，默认为高电平，直接生成代码

> 可以去NVIC优先级设置看看，在启用FreeRTOS后，NVIC将会自动设置，这里就不放图了。

![](/assets/images/1002/2.png)

## 代码分析

​	在生成代码后，项目中新增了与FreeRTOS相关的程序文件，包括可修改的和不看修改的FreeRTOS源程序文件。用户可修改的文件分布在\Inc和\Src文件夹下。

+ \Inc目录下的文件FreeRTOSConfig.h，是FreeRTOS的配置文件，包括很多宏定义，这些宏定义与CubeMX中的FreeRTOS可视化设置参数一一对应。
+ \Src目录下的stm32f4xx_hal_timebase_tim.c，是设置HAL基础时钟的文件，这就是我们之前设置的TIM6的一些设置，使得基础时钟的中断周期为1ms。**HAL库基础定时器是TIM6，FreeRTOS的基础时钟为SysTick定时器。**
+ \Src文件夹下的文件freertos.c，是由CubeMX生成的FreeRTOS初始化文件，主要在这个文件夹里创建任务，编写自己的功能代码。

### main.c分析

main.c文件分析，删去了没用到的沙箱代码，增加中文注释，之后基本都会这么做

```c
/* Includes 头文件------------------------------------------------------------------*/
#include "main.h"
#include "cmsis_os.h"
#include "gpio.h"

/* Private function prototypes -----------------------------------------------*/
void SystemClock_Config(void);		//系统时钟配置，在main.c内实现
void MX_FREERTOS_Init(void);		//FreeRTOS对象初始化函数，在freertos.c内实现

int main(void)
{
 
  HAL_Init();		//HAL初始化，复位所有外设，初始化Flash和HAL基础时钟

  SystemClock_Config();		//系统时钟配置

  /* Initialize all configured peripherals */
  MX_GPIO_Init();		//引脚GPIO初始化
  
  /* USER CODE BEGIN 2 */
  HAL_GPIO_WritePin(GPIOG, GPIO_PIN_13, GPIO_PIN_RESET);		//PG13=LED0 点亮
  /* USER CODE END 2 */

  /* Init scheduler */
  //CMSIS-RTOS函数，初始化FreeRTOS的任务调度器
  osKernelInitialize();  /* Call init function for freertos objects (in freertos.c) */
  
  //FreeRTOS对象初始化函数，在freertos.c内实现
  MX_FREERTOS_Init();

  /* Start scheduler */
  osKernelStart();		//CMSIS-RTOS函数，启动FreeRTOS的任务调度器

	//程序不会运行到这里，因为RTOS的任务调度器接管了系统的控制
  /* We should never get here as control is now taken by the scheduler */
  /* Infinite loop */
  /* USER CODE BEGIN WHILE */
  while (1)
  {
	  HAL_GPIO_TogglePin(GPIOG, GPIO_PIN_14);	//PG14=LED1 翻转
	  HAL_Delay(200);
    /* USER CODE END WHILE */

    /* USER CODE BEGIN 3 */
  }
  /* USER CODE END 3 */
}
```

将程序写入，发现只有LED0是点亮的，我们在while循环里写的控制LED延时翻转的代码并没有运行到，是因为，**启动任务调度器后，RTOS的任务调度器接管了系统的控制。**使用osKernelStart();函数后，其内部调动了FreeRTOS的函数vTaskStartScheduler() **/*启用任务调度器*/**，RTOS接管系统控制权，处理器循环执行FreeRTOS中各个任务的代码，FreeRTOS执行任务调度和其他功能管理。**处理器不会再执行 osKernelStart();之后的代码段**。

### freertos.c分析

​	文件freertos.c没有对应的头文件freertos.h。应用程序中需要用户编写的设计FreeRTOS操作的代码主要写在这个文件内。

```c
/* Includes 头文件------------------------------------------------------------------*/
#include "FreeRTOS.h"
#include "task.h"
#include "main.h"
#include "cmsis_os.h"

/* Private variables 私有变量---------------------------------------------------------*/

/* Definitions for defaultTask 任务defaultTask的定义*/
osThreadId_t defaultTaskHandle;	//任务defaultTask的句柄变量

//任务defaultTask的属性
const osThreadAttr_t defaultTask_attributes = {
  .name = "defaultTask",	//任务名
  .stack_size = 128 * 4,	//栈存储空间大小，cube里面设置是128，单位是字，这里变为字节单位，即128 * 4
  .priority = (osPriority_t) osPriorityNormal,	//任务优先级
};

/* Private function prototypes 私有函数原型声明-----------------------------------------------*/

void StartDefaultTask(void *argument);	//启动默认任务，任务名字在cubemx自己设置的

void MX_FREERTOS_Init(void); /* (MISRA C 2004 rule 8.1) 指使用这种代码规则*/

void MX_FREERTOS_Init(void) {
  /* USER CODE BEGIN Init */

  /* USER CODE END Init */

  /* USER CODE BEGIN RTOS_MUTEX */
  /* add mutexes, ... */	//添加互斥量
  /* USER CODE END RTOS_MUTEX */

  /* USER CODE BEGIN RTOS_SEMAPHORES */
  /* add semaphores, ... */	//添加信号量
  /* USER CODE END RTOS_SEMAPHORES */

  /* USER CODE BEGIN RTOS_TIMERS */
  /* start timers, add new ones, ... */
  /* USER CODE END RTOS_TIMERS */

  /* USER CODE BEGIN RTOS_QUEUES */
  /* add queues, ... */		//添加队列
  /* USER CODE END RTOS_QUEUES */

  /* Create the thread(s) 创建Cubemx里设计的任务*/
  /* creation of defaultTask 创建任务defaultTask*/
  
  defaultTaskHandle = osThreadNew(StartDefaultTask, NULL, &defaultTask_attributes);
//任务句柄 = osThreadNew【创建新任务函数】(任务函数名称，传递给任务的参数，任务的属性定义指针)

  /* USER CODE BEGIN RTOS_THREADS */
  /* add threads, ... */	//添加任务
  /* USER CODE END RTOS_THREADS */

  /* USER CODE BEGIN RTOS_EVENTS */
  /* add events, ... */
  /* USER CODE END RTOS_EVENTS */

}

//任务defaultTask的任务函数
void StartDefaultTask(void *argument)
{
  /* USER CODE BEGIN StartDefaultTask */
  /* Infinite loop */
  for(;;)
  {
    osDelay(1);		//RTOS的任务函数
  }
  /* USER CODE END StartDefaultTask */
}

```

详细分析：

***变量详解***

+ 开头创建的osThreadId_t类型的变量defaultTaskHandle，我们称其为任务defaultTask的句柄变量。
+ 还创建了一个osThreadAttr_t类型的结构体defaultTask_attributes，且为其成员变量赋值，这个结构体就是任务defaultTask的属性结构体。
+ 句柄变量defaultTaskHandle和属性结构体defaultTask_attributes都会在后面创建任务的函数osThreadNew()中使用到。

***函数详解***

+ MX_FREERTOS_Init()函数用于创建FreeRTOS中的任务、信号量、互斥量等对象，是FreeRTOS的初始化函数，其中的多个沙箱段可以用来创建互斥量、信号量任务等对象，一些在CubeMX里无法可视化设计的，也可以在这里添加代码创建

+ 创建任务指令：示例为

  > **defaultTaskHandle = osThreadNew(StartDefaultTask, NULL, &defaultTask_attributes);**
  >
  > **任务句柄 = osThreadNew【创建新任务函数】(任务函数名称，传递给任务的参数，任务的属性定义指针)**

## CubeMX任务创建分析

FreeRTOS的Task and Queues是任务与队列的管理页面，上半部Tasks就是任务的管理，选择Add新建一个任务，窗口图如下

![](/assets/images/1002/3.png)

这些设置对应的是osThreadAttr_t类型的任务属性结构体，结构体内部每个成员变量的意义及注释为

```c
/// Attributes structure for thread.
typedef struct {
  const char                   *name;   ///< name of the thread 任务的名称，只是用于备注
  uint32_t                 attr_bits;   ///< attribute bits	属性位
  void                      *cb_mem;    ///< memory for control block	用于控制块的存储空间
  uint32_t                   cb_size;   ///< size of provided memory for control block	控制卡存储空间的大小 单位：字
  void                   *stack_mem;    ///< memory for stack	栈存储空间
  uint32_t                stack_size;   ///< size of stack	栈空间的大小，单位：字
  osPriority_t              priority;   ///< initial thread priority (default: osPriorityNormal)	任务优先级，默认为osPriorityNormal
  TZ_ModuleId_t            tz_module;   ///< TrustZone module identifier	TrustZone模块标识符
  uint32_t                  reserved;   ///< reserved (must be 0)保留变量，必须是0
} osThreadAttr_t;
```

而CubeMX对话框内几个参数的作用解释为

> + Task name ：设置任务名称，仅用于备注
> + Priority：优先级，每个任务都需要有一个优先级，优先级与任务调度有关。类型osPriority_t已经枚举了56种优先级，代表着0~55，**任务优先级的数字越小，优先级越低。**
> + Stack Size：栈空间大小。栈空间大小的单位是字，一个字等于四个字节，进行任务切换时，要进行场景的保存与恢复，栈空间的存在就是为了保存任务切换时CPU的寄存器状态，待到下一次任务切换时恢复上一次结束时的状态。
> + Entry Function：入口函数，也就是任务函数，这里就可以设置任务函数名称。
> + Code Generation Option：生成代码选项。有Default和As weak两个选项。选择Default则生成正常函数，否则生成前带__weak的弱函数。
> + Parameter：参数。为任务函数传递的参数，如不需要传参则设为NULL。
> + Allocation：内存分配方式。是指任务的栈和控制块内存分配方式，选项有Dynamic和Static两种。**Dynamic即动态分配内存，任务动态创建osThreadNew栈存储空间stack_mem和任务控制块存储空间cb_mem。Static则需要在使用任务创建osThreadNew之前为栈存储空间stack_mem和任务控制块存储空间cb_mem赋值，即静态分配内存。**
> + Buffer Name：设置Static时可用，设置栈存储空间stack_mem的缓存数组名称
> + Control Block Name：设置Static时可用，设置控制块空间cb_mem的缓存数组名称

## 编写任务功能实现代码

​	回到项目中来，这次示例只有一个任务，任务函数是默认生成的StartDefaultTask。CubeMX已经生成好了其的基本框架

```c
void StartDefaultTask(void *argument)
{
  /* USER CODE BEGIN StartDefaultTask */
  /* Infinite loop */
  for(;;)
  {
	HAL_GPIO_TogglePin(GPIOG, GPIO_PIN_13);	//PG13=LED0 翻转电平
    osDelay(300);	//延时300个tick（时钟节拍)
  }
  /* USER CODE END StartDefaultTask */
}
```

​	在for循环中，调用的osDelay()函数是CMSIS-RTOS标准接口函数，内部调用FreeRTOS的延时函数vTaskDelay()，延时单位是时钟节拍（tick），我们知道FreeRTOS使用的时钟是SysTick滴答定时器，周期为1ms，是故周期也为1ms。

​	在FreeRTOS中，延时函数有两种vTaskDelay()和vTaskDelayUntil()，这两个函数都十分重要，是HAL_Delay和其他自准delay函数所不能替代的，之后我们会讲到在任务调度时三者的区别。

> ​	将代码写入单片机中，我们发现LED0（PG13）周期性闪烁，而LED1（PG14）依然没亮，说明单片机一直在循环执行任务函数StartDefaultTask内的代码，不会运行主函数在osKernelStart();之后的代码，我们写在while(1)循环里面的LED0翻转电平根本不会运行到。
>
> ​	这就是我们使用FreeRTOS进行的第一个例程，从点灯开始。

