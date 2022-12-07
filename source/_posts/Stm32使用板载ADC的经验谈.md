title: Stm32使用板载ADC的经验谈
author: Roxy
tags:
  - Stm32
  - ADC
categories:
  - Stm32
  - 经验谈
cover: 'https://s2.loli.net/2022/08/09/DvtKXnuBc4w2ykZ.jpg'
date: 2022-08-09 03:44:00
---
# Stm32F407使用板载AD经验



## 总述

​	Stm32板载的ADC是我们学习使用stm32的重要功能之一，参加电赛使用stm32所必须掌握的技能，曾使用过的ADS8688模块，号称500Ksps的采样率，在我们实际使用中哪怕是启用了硬件spi也只用得到80Ksps的采样率。有幸从敬重的学姐哪里继承了一块AD7606，并行的200Ksps16位ADC模块，目前尚未去驱动，有时间去试试。

​	言归正传，开此帖的目的是为了记录下使用板载adc进行的一些使用方法，双重、三重的交叉采样、ADC过采样、多通道规则采样等等自己使用过的方法，利弊也会一一列出，由于本人学艺不精的因素，也许也有错误存在，或是没有发挥出理想的驱动，作为抛砖引玉，希望看到我博客的能提出更好解决方法进行斧正。

> 很难言说我现在的精神状态，现在属于是醒了打代码，困了才睡，周而复始，希望不会过于胡言乱语不知所云。



# 使用设备及MCU

1、STlink

2、STM32F407ZGT6

3、实验室内的信号发生器、示波器等

4、CubeMX+CubeIDE开发

> 顺带一提，大家都知道奈奎斯特采样定理，采样率Fs应大于2倍的采集信号频率F，但是在实际使用中，如果说要对波形进行还原、FFT运算，以最简单最容易理解的也经常使用的实时采样，实时采样要采得完整的波形数字信号量，至少是Fs>=5*F，也就是说，对于大多数情况下我们使用ADC，至少需要到5倍的信号频率才能对信号完整采集。当然还有一些其他的采样算法，理论上是可以得到无限高的采样率，不过嘛，，，，都说了，理论上。



# 双重ADC快速交叉采样

​	单个ADC采样其实收到了很多限制，一般来说我们都是使用的定时器触发来设置采样频率，以ADC+DMA的方式快速得到采集数据，但我其实并不满意其速度，顶破了天其实也就200Ksps左右，再往上其实只是采正弦波都很勉强了。

​	于是在网上找到方法，ADC双重、甚至是三重交叉采样，在上一个ADC尚未转换完毕时直接将信号注入下一个ADC进行转换，理论上来说就可以成倍数的增长采样率，当时我看他们计算采样率的公式那叫一个心动啊，M级的采样率，我不是想采啥采啥。

​	来看看他们的计算公式：

```
In this example, the system clock is 144MHz, APB2 =72MHz and ADC clock = APB2 /2.
Since ADCCLK= 36MHz and Conversion rate = 6 cycle
==> Conversion Time = 36M/6cyc = 6Msps.
```

​	足足6Msps，想想都香，于是便有了这一次实验。直接开始创建工程吧。

## 建立工程

​	选择好使用的单片机，RCC选项中HSE（高速时钟）选中外部晶振，SYS选项中记得改为Serial Wire



## 配置时钟树

​	这里配置时钟树是有讲究的，这次并非直接拉满168MHz，我是照着这个算式来设置的时钟树

![](/assets/images/1003/1.png)

144/4=36；周期我们设置的6个adc周期；理论采样率为36/6=6M

## ADC和DMA设置

​	先将ADC1、ADC2打开，选择一样的IN口，我这里选的是IN12，其实无所谓，可以随便选的，这里DMA Access Mode要先把ADC1、ADC2的Resolution采样精度从默认的12bit改成8bit才能选中DMA access mode 3，之后再试试其他设置可不可行

![](/assets/images/1003/2.png)

​	接下来是ADC2的设置

![](/assets/images/1003/3.png)

​	DMA只需要打开ADC1的即可，同时要把ADC1的DMA continuous Requests DMA连续请求打开，ADC2的DMA连续请求是无法更改的，只能一会代码生成后去更改。这里有三个注意的地方，ADC1、2的设置照上面两张图对照配。

![](/assets/images/1003/4.png)

​	还有一个重要的一点，点开设置区，更改这里的顺序，定时器不用管，这个是我用来计算采样率用的，要求顺序一定是ADC1>DMA>ADC2，否则会导致采样只采样一次便终止。

![](/assets/images/1003/5.png)

​	接下来生成代码，而代码区还有很多需要更改才能使用

## 生成代码的修改

​	点开与main.c同级目录的adc.c，直接跳到**HAL_ADC2_Init(void)**函数，找到以下代码，将**DISABLE**改为**ENABLE**，使能ADC2的DMA连续请求，大概在代码的113行。

```c
  hadc2.Init.DataAlign = ADC_DATAALIGN_RIGHT;
  hadc2.Init.NbrOfConversion = 1;
  hadc2.Init.DMAContinuousRequests = ENABLE; //这一句
  hadc2.Init.EOCSelection = ADC_EOC_SINGLE_CONV;
```

​	再往下翻到**HAL_ADC_MspInit**函数，对这个函数我们将在前插入**__HAL_RCC_ADC2_CLK_ENABLE();**使能ADC2时钟，再将判断的语句删除，即删去if判断语句和else内的全部内容，更改后代码如下

> 注意，这里的更改每次生成代码都需要重新更改

```c
void HAL_ADC_MspInit(ADC_HandleTypeDef* adcHandle)
{

  GPIO_InitTypeDef GPIO_InitStruct = {0};
	//这里删去if判断和{
  /* USER CODE BEGIN ADC1_MspInit 0 */
  __HAL_RCC_ADC2_CLK_ENABLE();	//这里添加ADC2时钟使能

  /* USER CODE END ADC1_MspInit 0 */
    /* ADC1 clock enable */
    __HAL_RCC_ADC1_CLK_ENABLE();

    __HAL_RCC_GPIOC_CLK_ENABLE();
    /**ADC1 GPIO Configuration
    PC2     ------> ADC1_IN12
    */
    GPIO_InitStruct.Pin = GPIO_PIN_2;
    GPIO_InitStruct.Mode = GPIO_MODE_ANALOG;
    GPIO_InitStruct.Pull = GPIO_NOPULL;
    HAL_GPIO_Init(GPIOC, &GPIO_InitStruct);

    /* ADC1 DMA Init */
    /* ADC1 Init */
    hdma_adc1.Instance = DMA2_Stream0;
    hdma_adc1.Init.Channel = DMA_CHANNEL_0;
    hdma_adc1.Init.Direction = DMA_PERIPH_TO_MEMORY;
    hdma_adc1.Init.PeriphInc = DMA_PINC_DISABLE;
    hdma_adc1.Init.MemInc = DMA_MINC_ENABLE;
    hdma_adc1.Init.PeriphDataAlignment = DMA_PDATAALIGN_WORD;
    hdma_adc1.Init.MemDataAlignment = DMA_MDATAALIGN_WORD;
    hdma_adc1.Init.Mode = DMA_CIRCULAR;
    hdma_adc1.Init.Priority = DMA_PRIORITY_HIGH;
    hdma_adc1.Init.FIFOMode = DMA_FIFOMODE_DISABLE;
    if (HAL_DMA_Init(&hdma_adc1) != HAL_OK)
    {
      Error_Handler();
    }

    __HAL_LINKDMA(adcHandle,DMA_Handle,hdma_adc1);

  /* USER CODE BEGIN ADC1_MspInit 1 */

  /* USER CODE END ADC1_MspInit 1 */
//这里删去}和else的所有内容

}
```

## 主函数的使用

​	回到main.c首先先创建变量

```c
//DMA传输值
__IO uint16_t uhADCDualConvertedValue = 0;	
//ADC1电压值
float uwADC1ConvertedVoltage;
//ADC2电压值
float uwADC2ConvertedVoltage;
//ADC1AD值
uint8_t uwADC1ConvertedValue;
//ADC2AD值
uint8_t uwADC2ConvertedValue;
```

一个重要的关系是DMA传输值和ADC1、ADC2的AD值的关系，在前面DMA的设置中，我们设置的DMA数据传输单位是Word，即一个字，一个字由四个字节组成，而一个值正好为Half Word，即一个半字，由两个字节组成，其实答案就很明显了，高位半字用来存放ADC2的数据，而低位半字用来存放ADC1的数据。官方的文档上写的是

```
A DMA request is generated each time 2 data items are available
1st request: ADC_CDR[15:0] = (ADC2_DR[7:0] << 8) | ADC1_DR[7:0] 
2nd request: ADC_CDR[15:0] = (ADC2_DR[7:0] << 8) | ADC1_DR[7:0]
```

意思是

```
每发送一个DMA请求（两个数据可用），就好以字的形式传输表示两个ADC转换数据项的两个半字。
高位半字用来存放ADC2的数据，而低位半字用来存放ADC1的数据，以此类推。
```

创建好变量后，来到主函数。

> 先等初始化完毕后，先打开ADC2，再打开ADC1的多重ADC模式DMA，顺序一定不能错！！！

```c
/* Initialize all configured peripherals */
  MX_GPIO_Init();
  MX_ADC1_Init();
  MX_DMA_Init();
  MX_ADC2_Init();

  /* USER CODE BEGIN 2 */
//是的就这两句，顺序不对会导致只能采一次，折磨我一天从早八到现在凌晨三点才解决
  HAL_ADC_Start(&hadc2);
  HAL_ADCEx_MultiModeStart_DMA(&hadc1, (uint32_t*)&uhADCDualConvertedValue, 1);
  /* USER CODE END 2 */
```

## 采值数据转换

我们已经采值到uhADCDualConvertedValue了，接下来将其转换为电压值，不要放到while(1)循环里，会不起作用，只有使用ADC中断回调才能变换，但是我怀疑中断回调对采样率造成了影响，这个我们后面再说。

```c
/* USER CODE BEGIN 4 */
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef* hadc)
{
	//取低半字，0xff就是4095的16进制写法而已,单纯我喜欢这样写而已
	uwADC1ConvertedValue = (uhADCDualConvertedValue & 0x00FF);
	uwADC1ConvertedVoltage = uwADC1ConvertedValue * 3.3 / 0xFF;
	//取高半字
	uwADC2ConvertedValue = (uhADCDualConvertedValue >> 8);
//	uwADC2ConvertedVoltage = uwADC2ConvertedValue * 3.3 / 0xFF;

}
```

## 实际测量和优化实验

​	实际测量下，照上面的写法，我新增了一个1s的定时器，并使能中断，再在adc中断回调函数里面设置了每次执行自加2的计数值，并在定时器中断里取得值并且归零，以此来计算1s内采样到的值，即粗略判断采样频率。

​	然而，其实不过仅仅是68640，仅仅68ksps！而且还只是8bit的采样，采样精度十分令人不满意，速度也没有达到理想的6M。

​	但理论和实际差的实在太多，我也希望能找出原因来，然后我认为可能是浮点数的运算影响了采样速率，我将ADC中断回调函数更改至以下样式

```c
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef* hadc)
{
	uwADC1ConvertedValue = (uhADCDualConvertedValue & 0x00FF);
//	uwADC1ConvertedVoltage = uwADC1ConvertedValue * 3.3 / 0xFF;
	count++;
	uwADC2ConvertedValue = (uhADCDualConvertedValue >> 8);
//	uwADC2ConvertedVoltage = uwADC2ConvertedValue * 3.3 / 0xFF;
	count++;
}
```

​	注释掉浮点数运算后，再进行测量，结果是达到了392856,也就是392Ksps，虽然只是8bit的精度，但其实速率很好的上去了，可以进行间断采样的方式，先将值采到足够大的数组去，再暂停采样，进行数据处理、分析，结束后再次采样，以此循环。这样就是一个很好的高速AD方案了。我没有打开FPU，打开FPU后前面在ADC中断回调函数里面进行浮点数运算的速度应该能再进一步提升。

​	之后，我又将中断回调函数内仅仅留下计数值自加的式子

```c
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef* hadc)
{
//	uwADC1ConvertedValue = (uhADCDualConvertedValue & 0x00FF);
//	uwADC1ConvertedVoltage = uwADC1ConvertedValue * 3.3 / 0xFF;
	count++;
//	uwADC2ConvertedValue = (uhADCDualConvertedValue >> 8);
//	uwADC2ConvertedVoltage = uwADC2ConvertedValue * 3.3 / 0xFF;
	count++;
}
```

​	结果是采样速率进一步得到了提升，达到了418600,418K的sps，去掉了移位操作，采样率直接提升了6.5%。

## 总结

​	其实早已有理论和实际差别很大的心理准备，因为如果真的能那么理想化实现，ST公司肯定会铺天盖地的宣传，也不至于大家都基本用的ADC+DMA+定时器中断。

​	总的来说，双重ADC交叉采样确实是能满足一定的高速需求，但8bit的精度我认为实在有点差，之后可以采取定时器触发的方式来设置采样频率，这里使用软件触发的方式也是试图测出其能达到的上限。		

​	当然，也许真的有办法实现更加贴近理论计算的配置方法，但我现在也尚未知晓，等我先去尝试一下其他的方法，有时间再对配置优化处理。

> 注意，我这里使用的计算频率的方法不一定是正确的，也有很多漏洞，但我一时想不出来较好的计算实际采样频率的方法，故以此粗略简陋办法进行计算，也许也有错误存在，若对此对读者进行误导则先行道歉。

# 三重ADC快速交替采样

​	试试三重ADC快速交替采样，首先先看看英文文档描述

```
How to use the ADC peripheral to convert a regular channel in Triple 
interleaved mode.

The Triple interleaved delay is configured 5 ADC clk cycles.

In Triple ADC mode, three DMA requests are generated: 
- On the first request, both ADC2 and ADC1 data are transferred (ADC2 data take 
  the upper half-word and ADC1 data take the lower half-word). 
- On the second request, both ADC1 and ADC3 data are transferred (ADC1 data take
  the upper half-word and ADC3 data take the lower half-word).
- On the third request, both ADC3 and ADC2 data are transferred (ADC3 data take 
  the upper half-word and ADC2 data take the lower half-word) and so on.

On each DMA request (two data items are available) two half-words representing 
two ADC-converted data items are transferred as a word.
```

机翻翻译是

```
如何使用ADC外设转换常规通道在三重
交错模式。

“三重交错延时”配置为5个ADC时钟周期。

在Triple ADC模式下，生成3个DMA请求:
—第一次请求时，同时传输ADC2和ADC1数据(ADC2数据取
上半字和ADC1数据取下半字)。
—第二个请求同时传输ADC1和ADC3数据(ADC1数据取
上半字和ADC3数据取下半字)。
—第三次请求时，同时传输ADC3和ADC2数据(ADC3数据取
上半字和ADC2数据取下半字)等等。

对于每个DMA请求(两个数据项可用)，两个半字表示
两个adc转换的数据项作为一个字传输。
```

有点意思，以三次请求为一个循环，创建数组依照顺序就是

> ADC1[第一次DMA传回数据下半字]>>
>
> ADC2[第一次DMA传回数据上半字]>>
>
> ADC3[第二次DMA传回数据下半字]>>
>
> ADC1[第二次DMA传回数据上半字]>>
>
> ADC2[第三次DMA传回数据下半字]>>
>
> ADC3[第三次DMA传回数据上半字]；
>
> 以此循环

但我看英文文档里面的示例是直接求均值，顺便给他均值滤波了，也是个不错的方案。

描述是ADC123都使用通道12进行交叉采样，通过这种方法能达到12bit精度的6M采样率，周期间隔设置为5个ADC周期

```
The ADC1, ADC2 and ADC3 are configured to convert ADC Channel 12.
By this way, the ADC can reach 6Msps, in fact the same channel is converted
each 5 cycles
144/4=36；周期我们设置的5个adc周期；理论采样率为36/5=7.2M
```

## 创建工程

​	RCC、SYS、时钟树配置与做二重交叉采样相同，直接来看ADC的选项设置

​	ADC三个都选择通道12，回到ADC1，选择三重交叉模式，通道连续转换使能，去开启DMA后使能DMA连续转换模式，ADC2、ADC3保持默认设置

![](/assets/images/1003/6.png)

​	DMA设置还是只打开ADC1的DMA，配置与双重交叉采样时相同，这里直接用相同的图片

![](/assets/images/1003/7.png)

​	生成代码

## 生成代码修改

​	直接进到adc.c里的HAL_ADC_MspInit函数，删去if判断语句和后续else所有内容，加入启用ADC2、ADC3时钟代码，其他无需修改

```c
void HAL_ADC_MspInit(ADC_HandleTypeDef* adcHandle)
{

  GPIO_InitTypeDef GPIO_InitStruct = {0};
	//删去if判断语句
  /* USER CODE BEGIN ADC1_MspInit 0 */
  //ADC2、ADC3时钟使能
	  __HAL_RCC_ADC2_CLK_ENABLE();
	  __HAL_RCC_ADC3_CLK_ENABLE();
  /* USER CODE END ADC1_MspInit 0 */
    /* ADC1 clock enable */
    __HAL_RCC_ADC1_CLK_ENABLE();

    __HAL_RCC_GPIOC_CLK_ENABLE();
    /**ADC1 GPIO Configuration
    PC2     ------> ADC1_IN12
    */
    GPIO_InitStruct.Pin = GPIO_PIN_2;
    GPIO_InitStruct.Mode = GPIO_MODE_ANALOG;
    GPIO_InitStruct.Pull = GPIO_NOPULL;
    HAL_GPIO_Init(GPIOC, &GPIO_InitStruct);

    /* ADC1 DMA Init */
    /* ADC1 Init */
    hdma_adc1.Instance = DMA2_Stream0;
    hdma_adc1.Init.Channel = DMA_CHANNEL_0;
    hdma_adc1.Init.Direction = DMA_PERIPH_TO_MEMORY;
    hdma_adc1.Init.PeriphInc = DMA_PINC_DISABLE;
    hdma_adc1.Init.MemInc = DMA_MINC_ENABLE;
    hdma_adc1.Init.PeriphDataAlignment = DMA_PDATAALIGN_WORD;
    hdma_adc1.Init.MemDataAlignment = DMA_MDATAALIGN_WORD;
    hdma_adc1.Init.Mode = DMA_CIRCULAR;
    hdma_adc1.Init.Priority = DMA_PRIORITY_HIGH;
    hdma_adc1.Init.FIFOMode = DMA_FIFOMODE_DISABLE;
    if (HAL_DMA_Init(&hdma_adc1) != HAL_OK)
    {
      Error_Handler();
    }

    __HAL_LINKDMA(adcHandle,DMA_Handle,hdma_adc1);

  /* USER CODE BEGIN ADC1_MspInit 1 */

  /* USER CODE END ADC1_MspInit 1 */
	//后续else语句全部删除
}
```

## 主运行代码

​	回到main.c，首先创建变量

```c
__IO uint32_t aADCTripleConvertedValue[3];
```

​	在初始化以后，添加启用adc函数，注意初始化的顺序，如果照着不能实现，试试更改初始化代码的顺序至和现在示例一致试试看

```c
 /* Initialize all configured peripherals */
  MX_GPIO_Init();
  MX_ADC1_Init();
  MX_DMA_Init();
  MX_ADC3_Init();
  MX_ADC2_Init();
  /* USER CODE BEGIN 2 */
  HAL_ADC_Start(&hadc3);
  HAL_ADC_Start(&hadc2);
  HAL_ADCEx_MultiModeStart_DMA(&hadc1, (uint32_t*)aADCTripleConvertedValue, 3);
```

这样我们就可以存储采样值了，再看看均值滤波转换的方法

更改变量为

```c
__IO uint32_t aADCTripleConvertedValue[128];
uint32_t ADC3ConvertedVoltage;
uint8_t i;
float res;
```

主运行代码为

```c
  HAL_ADC_Start(&hadc3);
  HAL_ADC_Start(&hadc2);
  HAL_ADCEx_MultiModeStart_DMA(&hadc1, (uint32_t*)aADCTripleConvertedValue, 128);
  /* USER CODE END 2 */

  /* Infinite loop */
  /* USER CODE BEGIN WHILE */
  while (1)
  {
		ADC3ConvertedVoltage = 0;
		for (i = 0; i < 128; i++) {
			ADC3ConvertedVoltage += (aADCTripleConvertedValue[i] << 16) >> 16;
			ADC3ConvertedVoltage += aADCTripleConvertedValue[i] >> 16;
		}
		ADC3ConvertedVoltage = ADC3ConvertedVoltage / 256;//128数组分成高低半字那肯定*2啊
		res = ADC3ConvertedVoltage * 3.29 / 4096;
    /* USER CODE END WHILE */

    /* USER CODE BEGIN 3 */
  }
```

​	这个转换方法是我从毅子哥推荐的博主eric2013硬汉的帖子里找到的，来自安富莱电子的硬汉嵌入式论坛，我从其中学到了很多，推荐大家多看看学习。大佬eric2013又高又硬，强。

## 实际测量和优化实验

​	我将实验前面我使用的方法来粗略的测量采样频率，设置定时器2中断，周期1s

```c
//变量区
/* Private variables ---------------------------------------------------------*/

/* USER CODE BEGIN PV */
__IO uint32_t aADCTripleConvertedValue[128];
uint32_t ADC3ConvertedVoltage;
uint8_t i;
float res;

uint32_t count=0;
uint32_t Hz=0;
/* USER CODE END PV */
```

主函数区

```c
/* Initialize all configured peripherals */
  MX_GPIO_Init();
  MX_ADC1_Init();
  MX_DMA_Init();
  MX_ADC3_Init();
  MX_ADC2_Init();
  MX_TIM2_Init();
  /* USER CODE BEGIN 2 */
  HAL_TIM_Base_Start_IT(&htim2);
  HAL_ADC_Start(&hadc3);
  HAL_ADC_Start(&hadc2);
  HAL_ADCEx_MultiModeStart_DMA(&hadc1, (uint32_t*)aADCTripleConvertedValue, 128);
  /* USER CODE END 2 */
  //while(1)中已经注释
```

中断回调区

```c
/* USER CODE BEGIN 4 */
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef* hadc)
{
	ADC3ConvertedVoltage = 0;
	for (i = 0; i < 128; i++) {
		ADC3ConvertedVoltage += (aADCTripleConvertedValue[i] << 16) >> 16;
		ADC3ConvertedVoltage += aADCTripleConvertedValue[i] >> 16;
	}
	ADC3ConvertedVoltage = ADC3ConvertedVoltage / 256;
	res = ADC3ConvertedVoltage * 3.29 / 4096;
	count+=256;
}
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
	if(htim->Instance == htim2.Instance)
	{
		Hz=count;
		count=0;
	}
}
/* USER CODE END 4 */
```

​	经实验测试，HZ为4517632，也就是4.5M采样频率！！！！！！！！！且这是没有启用FPU浮点数运算的，启用FPU后说不定还能有提升，我们首先将浮点数运算注释。再进行一次。

​	再一次检验得到，Hz=4702208，即4.7M，也许是我飘了，0.2M已经是200Ksps了我居然觉得就提升这么一点啊，乐。

## 总结

​	也许实际采样频率并不是我这种粗略计算得到的数值，但得到4.5M我是没有想到的，这一个结果出乎意料，令我十分惊喜，接下来还需要在实际使用中试试看是否真的有4.5M的12bitADC转换。

​	虽然说将板载的3个ADC变得只能有一个能用，但很多时候也仅仅只需要一个高速集来即可，总的来说若真能发挥出这样的表现，那将有很多种用处。