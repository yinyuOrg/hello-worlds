# 6 TIM定时器
> 笔记主要参考B站江科大自化协教学视频“[STM32入门教程-2023版](https://www.bilibili.com/video/BV1th411z7sn/)”。

定时器是STM32中功能最强大、结构最复杂的一个外设。定时器将包括四部分8小节：
1. 第一部分主要讲定时器基本定时的功能，也就是指定一个时间，让定时器每个这段时间就产生一个中断，如实现时钟、秒表等。
2. 第二部分主要讲定时器输出比较的功能。输出比较模块最常见的用途就是产生PWM波，用于驱动电机、舵机等设备。
3. 第三部分主要讲定时器输入捕获的功能。将会学习使用输入捕获模块测量方波频率。
4. 第四部分主要讲定时器的编码器接口。使用编码器接口，可以方便的读取正交编码器的输出波形，广泛应用在编码电机测速中。

## 6.1 TIM定时中断原理
**TIM（Timer）定时器**最基本的功能是对输入的时钟进行计数，并在计数值达到设定值时触发中断。若这个输入是一个可靠的基准时钟，那么对这个基准时钟技术就实现了计时的功能。
> 如STM32中主频一般是72MHz，那么计数值定为72，就是每1us触发一次中断；计数值设定为72000，就是每1ms触发一次中断。

STM32定时器拥有由16位**计数器、预分频器、自动重装寄存器**组成的时基单元，在72MHz计数时钟下可以实现最大59.65s的定时。TIM不仅具备基本的定时中断功能，而且还包含内外时钟源选择、输入捕获、主从触发模式、输出比较、编码器接口等多种功能，并且根据复杂度和应用场景分为了高级定时器、通用定时器、基本定时器三种类型。

定时器类型:
| 类型 | 编号 | 总线 | 功能 |
|---|---|---|---|
| 高级定时器【最复杂】 | TIM1、TIM8 | APB2 | 拥有通用定时器全部功能，并额外具有重复计数器、死区生成、互补输出、刹车输入等功能 |
| 通用定时器【最常用】 | TIM2、TIM3、TIM4、TIM5 | APB1 | 拥有基本定时器全部功能，并额外具有内外时钟源选择、输入捕获、输出比较、编码器接口、主从触发模式等功能 |
| 基本定时器【最简单】 | TIM6、TIM7 | APB1 | 拥有定时中断、主模式触发DAC的功能 |

- 只有高级定时器连接的是性能最高的APB2总线，剩下的通用定时器和基本定时器都是APB1总线。
- 高级定时器额外多出的功能主要是为了三相无刷电机的驱动设计的，本课程不会涉及到。
- 不同型号芯片的定时器数量不同，stm32f103c8t6定时器资源：TIM1、TIM2、TIM3、TIM4。没有基本定时器。

下面依次介绍上面三种类型的定时器。
![基本定时器框图](../images/6.1%20基本定时器框图.png)
- 内部时钟（CK_CNT）：一般就是系统的主频72MHz，通向时基单元的输入。
- 时基单元：16位预分频器+16位计数器+16位自动重装载寄存器。
    1. 预分频器：对输入的72MHz时钟进行预分频，寄存器内存储的值是实际的分频系数减一。写0就是不分频，写1就是2分频，写2就是3分频......
    2. 计数器：对预分频后的技术时钟进行计数，每遇到上升沿就加一。
    3. 自动重装载寄存器：存储计数的最大值，到达此值后触发中断并清零计数器。
- 折线UI：向上的折线箭头表示该位置会产生中断信号——“更新中断”（由计数值等于自动重装值产生的中断），这个中断信号会通向NVIC。
- 折线U：向下的折线箭头表示该位置会产生时间++事件——“更新事件”，这个更新事件不会触发中断，但可以触发内部其他电路的工作。

主模式触发DAC：
- STM32的一大特色就是主从触发模式，可以让内部的硬件在不受程序的控制下自动运行，可以极大地减轻CPU的负担。
- 驱动DAC正常思路及其问题：每隔一段时间就产生一个定时中断，手动更新DAC的值。但这样子会频繁的产生中断，会影响主程序的运行和其他中断的响应。
- 解决方法：定时器设计了一个主模式，使用主模式可以将定时器的“更新事件”映射到“触发输出TRGO”，然后TRGO直接接到DAC的触发转换引脚上，于是定时器的更新就不需要中断来实现了。整个工程无需软件参与，实现了硬件的自动化。
> 除了主模式外，还有更多的硬件自动化设计。

![通用定时器框图](../images/6.2%20通用定时器框图.png)
- 时基单元：中间的PSC预分频器、自动重装载寄存器、CNGT计数器。通用定时器和高级定时器新增两个功能——“向下计数模式”、“中央计数模式”。
    - 向上计数模式【常用】：从0开始累加，到自动重装载值触发中断。
    - 向下计数模式：从自动重装载值递减，到0触发中断。
    - 中央对其模式：从0开始累加，到自动重装载值触发中断，然后递减，到0再次触发中断。常用于电机控制的SVPWM算法中。
- 内外时钟源选择和主从触发模式结构：上面的一大块。下面介绍各种各样的内外时钟源：
    1. 内部时钟CK_INT【常用】：通常为72MHz，基本定时器只能选择CK_INT，通用定时器和高级定时器则新增了下面的时钟源。
    2. 外部TIMx_ETR【常用】：引脚的位置可以参考引脚定义表，如stm32f103c8t6的PA0引脚复用了TIM2_CH1_ETR等。外部输入了方波时钟，然后通过极性选择、滤波等电路进行整形，然后兵分两路，一路ETRF进入触发控制器，紧跟着就可以选择成为时基单元的时钟（外部时钟模式2）；一路进入选择器等待成为TRGI。TRGI主要用作触发输入来使用，可以触发定时器的从模式，本小节仅用做外部时钟（外部时钟模式1），其他功能后续再讲。
    3. 外部ITR信号：包括ITR0~ITR4（引脚定义见参考手册“表78 TIMx内部触发连接”），来自其他定时器，实现了定时器的级联。这个ITR信号从上一级定时器的主模式TRGO引脚来（图片右上方）。
    4. 外部TI1F-ED：来自于输入捕获单元的TIMx_CH1引脚，后缀“ED”意为边沿，也就是说该路时钟的上升沿和下降沿均有效，也就是CH1引脚的边沿。
    5. 外部TI1FP1：来自CH1引脚时钟。
    6. 外部TI1FP2：来自CH2引脚时钟。
        - 编码器接口：可以读取正交编码器的输出波形，后续介绍。
        - TRGO引脚：定时器的主模式输出，可以将内部的一些事情映射到TRGO上，相比基本定时器这些事情的范围显然更广。
    > 最后三种外部时钟用于输入捕获和测频率，后续介绍。
- 输出比较电路：下面右侧的一大堆，总共4个通道，可以用于输出PWM波形驱动电机。
- 输入捕获电路：下面左侧的一大堆，也是有4个通道，可以用于测量输入方波的频率。
> 输入捕获电路和输出比较电路不能同时使用，所以共用中间的“捕获/比较寄存器”以及输入/输出的引脚。后续再介绍。

![高级定时器框图](../images/6.3%20高级定时器框图1.png)
相比于通用定时器，高级定时器主要增加了以下功能：对输出比较模块的升级
- 重复次数计数器：可以实现每个几个计数周期，才发生一次更新事件和更新中断，相当于自带一级的定时器级联。
- DTG（Dead Time Generate）：死区生成电路。将输出引脚由原来的一个变为两个互补的输出，可以输出一对互补的有死区的PWM波（防止出现短暂的直通现象），可以驱动三相无刷电机（如四轴飞行器、电动车后轮、电钻等）。
- BRK刹车输入：为了给电机驱动提供安全保障，如果外部引脚BKIN（Break IN）产生了刹车信号或者内部时钟失效，这个电路就会自动切断电机的输出，防止意外的发生。

![定时中断基本结构图](../images/6.4%20定时中断基本结构图.png)
上图是UP画的定时中断基本结构图，去掉了一些无关的东西并加了一些定时器框图中没有体现的模块，后续在配置TIM时可以直接参考本图。
- 时基单元：中间的粉色部分。
- 运行控制：控制寄存器的一些位，如启动停止、向上或向下计数等，操作这些寄存器就可以控制时基单元的运行了。
- 内部时钟模式、外部时钟模式2、外部时钟模式1：外部时钟源选择。这个选择器的输出就是为时基单元提供时钟。
- 编码器模式：编码器独有的模式，一般用不到。
- 中断申请控制：由于定时器内部有很多地方要申请中断，“中断申请控制”就用来使能控制这些中断是否使能。比如，中断信号会先在状态寄存器里置一个中断标志位，这个标志位会通过中断输出控制，到NVIC申请中断。

**上面三个框图中，有阴影的部分都是包含缓冲寄存器（影子寄存器）的**，具体作用见下面时基单元运行时的一些细节：
![预分频器时序](../images/6.5%20预分频器时序.png)
- CK_PSC：预分频器的输入时钟，选择内部时钟源就是72MHz。
- CNT_EN：计数器使能。高电平计数器正常运行，低电平计数器停止。
- CK_CNT：计数器时钟，既是预分频器的时钟输出，也是计数器的时钟输入。
- 计数器寄存器：对CK_CNT进行自增计数，到达自动重装载值清零。
- 更新事件：计数器寄存器到达自动重装载值时产生一个脉冲。

下面三行时序体现了预分频计数器的一种缓冲机制：
- 预分频控制寄存器：供用户读写使用，实时响应用户控制，但并不直接决定分频系数。
- 预分频缓冲器：也称为影子寄存器，真正起分频作用的寄存器。只有在更新事件到达时，才从“预分频控制寄存器”更新预分频参数。以确保更新事件的稳定性。
- 预分频计数器：按照预分频参数进行计数，以产生对应的CK_CNT脉冲。

计数器无预装时序：
![计数器无预装时序](../images/6.6%20计数器无预装时序.png)

计数器有预装时序：
![计数器有预装时序](../images/6.7%20计数器有预装时序.png)

正常的计数器时序没啥好说的，就是根据CK_CNT计数，到达自动重装载值产生中断，所以只需看一下更新自动重装载值的过程：
- 无预装时序（禁用缓冲寄存器）：有溢出问题。若将自动重装载值变小，且此时计数器寄存器已经超过这个新的重装载值，那么计数器寄存器就会一直计数到FFFF才清零。这可能会造成一些问题。
- 有预装时序（启用缓冲寄存器）：比较稳定。只有更新事情来临时才更新自动重装载值。

RCC时钟树:
![时钟树](../images/6.8%20时钟树.png)
时钟是所有外设的基础，所以是需要最先配置的。ST公司写好了`SystemInit`函数来配置时钟树，下面具体介绍：
- 左侧是时钟产生电路，右侧时钟分配电路，中间的SYSCLK就是72MHz的系统时钟。
- 时钟产生电路：
    - 四个震荡源：
        1. 内部的8MHz高速RC振荡器。
        2. 外部的4-16MHz高速石英晶体振荡器：一般8MH在，相比于内部的RC高速振荡器更加稳定。
        3. 外部的32.768kHz低速晶振：一般给RTC提供时钟。
        4. 内部的40kHz低速RC振荡器：给看门狗提供时钟。
        
        上面的两个高速晶振用于给系统提供时钟，如AGB、APB1、APB2的时钟。
    - SystemInit函数配置时钟的过程：首先启动内部8MHz时钟为系统时钟，然后配置外部8MHz时钟到PLLMUL模块进行9倍频到72MHz，等到这个72MHz时钟稳定后，再将其作为系统时钟。于是就实现了系统时钟从8MHz切换到72MHz。
    - CSS：监测外部时钟的运行状态，一旦外部时钟失效，就会自动把外部时钟切换成内部时钟，保障系统时钟的运行，防止程序卡死造成事故。如果外部晶振出问题，那么就会导致程序的时钟变为8MHz，也就是比预期的时钟慢了9倍。
- 时钟分配电路
    - AHB总线：有预分频器，SysytemInit配置分频系数为1，于是AHB时钟输出就是72MHz。
    - APB1总线：SysytemInit配置分频系数为2，于是APB1时钟输出就是36MHz。
    - APB2总线：SysytemInit配置分频系数为1，于是APB2时钟输出就是72MHz。
    - 外设时钟使能：就是库函数`RCC_APB2PeriphClockCmd`开启的地方，可以控制相应的外设时钟开启。
    - 定时器的时钟：从图中可以看出，按照SystemInit的默认配置，所有的定时器时钟都是72MHz。

## 6.2 TIM定时中断相关实验

### 6.2.1 实验：定时器定时中断-内部时钟
需求：在OLED显示屏上显示数字，每秒自动加一。

定时器定时中断-内部时钟-接线图:
![接线图-定时器中断-内部时钟](../images/6.9%20接线图-定时器中断-内部时钟.png)
定时器定时中断-内部时钟-代码调用（非库函数）:
![代码调用-定时器中断-内部时钟](../images/6.10%20代码调用-定时器中断-内部时钟.png)

下面是代码展示： 
- main.c
    ``` C
        #include "stm32f10x.h"                  // Device header
        #include "OLED.h"
        #include "Timer.h"

        uint16_t TimerCount = 0;

        int main(void){
            //配置中断的优先级分组，每个工程只能出现一次！！
            NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);
            
            //OLED显示屏初始化
            OLED_Init();
            OLED_ShowString(1,1,"TIM_Interrupt:");
            OLED_ShowNum(2,1,0,5);
            
            //定时器初始化
            Timer_Init();
            
            while(1){
                OLED_ShowNum(2,1,TimerCount,5);
            };
        }

        //TIM2定时中断后的中断函数
        void TIM2_IRQHandler(void){
            if(TIM_GetITStatus(TIM2,TIM_IT_Update)==SET){
                TimerCount++;
                TIM_ClearITPendingBit(TIM2,TIM_IT_Update);
            }
        }
    ```
- Timer.h
    ``` C
        #ifndef __TIMER_H
        #define __TIMER_H

        void Timer_Init(void);

        #endif
    ```
- Timer.c
    ``` C
       #include "stm32f10x.h"                  // Device header

        // 定时器初始化-TIM2
        void Timer_Init(void){
            //1.初始化RCC内部时钟
            RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2, ENABLE);
            //2.选择时基单元的时钟
            TIM_InternalClockConfig(TIM2);//默认使用内部时钟，也可以不写
            //3.配置时基单元
            TIM_TimeBaseInitTypeDef TIM_TimeBaseInitStructure;
            TIM_TimeBaseInitStructure.TIM_ClockDivision = TIM_CKD_DIV1;//外部时钟源的输入滤波器采样频率，内部时钟无所谓
            TIM_TimeBaseInitStructure.TIM_CounterMode = TIM_CounterMode_Up;//加计数
            TIM_TimeBaseInitStructure.TIM_Period = 10000-1;//ARR自动重装器的值10000
            TIM_TimeBaseInitStructure.TIM_Prescaler = 7200-1;//PSC预分频的值7200
            TIM_TimeBaseInitStructure.TIM_RepetitionCounter = 0;//重复计数器的值（高级定时器才有）
            TIM_TimeBaseInit(TIM2, &TIM_TimeBaseInitStructure);
            TIM_ClearFlag(TIM2,TIM_FLAG_Update);//消除上一行TIM_TimeBaseInit立刻产生更新事件影响
            //4.配置中断输出控制
            TIM_ITConfig(TIM2, TIM_IT_Update, ENABLE);
            //5.配置NVIC
            NVIC_InitTypeDef NVIC_InitStructure;
            NVIC_InitStructure.NVIC_IRQChannel = TIM2_IRQn;
            NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
            NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 2 ;
            NVIC_InitStructure.NVIC_IRQChannelSubPriority = 1;
            NVIC_Init(&NVIC_InitStructure);
            //6.配置运行控制
            TIM_Cmd(TIM2, ENABLE);    
        }

        /*
        //TIM2定时中断后的中断函数
        void TIM2_IRQHandler(void){
            if(TIM_GetITStatus(TIM2,TIM_IT_Update)==SET){
                ??    
                TIM_ClearITPendingBit(TIM2,TIM_IT_Update);
            }
        }
        */
    ```

### 6.2.2 实验：定时器定时中断-外部时钟
需求：对外部输入的方波（对射式红外传感器）进行计次，每出现9个方波就自动加一。

![接线图-定时器中断-外部时钟](../images/6.11%20接线图-定时器中断-外部时钟.png)

“定时器定时中断-外部时钟”与“定时器定时中断-内部时钟”的 代码调用关系相同。下面是代码展示：
- main.c
    ``` C
        #include "stm32f10x.h"                  // Device header
        #include "OLED.h"
        #include "Timer.h"

        uint16_t TimerCount = 0;

        int main(void){
            //配置中断的优先级分组，每个工程只能出现一次！！
            NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);
            
            //OLED显示屏初始化
            OLED_Init();
            OLED_ShowString(1,1,"TimerCount:");
            OLED_ShowNum(2,1,0,5);
            OLED_ShowString(3,1,"CNT:");
            OLED_ShowNum(3,5,0,5);
            
            //定时器初始化
            Timer_Init();
            
            while(1){
                OLED_ShowNum(2,1,TimerCount,5);
                OLED_ShowNum(3,5,TIM_GetCounter(TIM2),5);
            };
        }

        //TIM2定时中断后的中断函数
        void TIM2_IRQHandler(void){
            if(TIM_GetITStatus(TIM2,TIM_IT_Update)==SET){
                TimerCount++;
                TIM_ClearITPendingBit(TIM2,TIM_IT_Update);
            }
        }
    ```
- Timer.h
    ``` C
        #ifndef __TIMER_H
        #define __TIMER_H

        void Timer_Init(void);

        #endif
    ```
- Timer.c
    ``` C
        #include "stm32f10x.h"                  // Device header

        // 定时器初始化-TIM2
        void Timer_Init(void){
            //1.初始化RCC内部时钟
            RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2, ENABLE);
            RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
            //2.配置GPIO-PA0
            GPIO_InitTypeDef GPIO_InitStructure;
            GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;//虽然器件手册推荐浮空输入，但上拉输入
            GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0;
            GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
            GPIO_Init(GPIOA, &GPIO_InitStructure);
            //3.选择时基单元的时钟-ETR外部时钟模式2
            TIM_ETRClockMode2Config(TIM2, TIM_ExtTRGPSC_OFF, TIM_ExtTRGPolarity_Inverted, 0x0F);
            //4.配置时基单元
            TIM_TimeBaseInitTypeDef TIM_TimeBaseInitStructure;
            TIM_TimeBaseInitStructure.TIM_ClockDivision = TIM_CKD_DIV1;//外部时钟源的输入滤波器采样频率，内部时钟无所谓
            TIM_TimeBaseInitStructure.TIM_CounterMode = TIM_CounterMode_Up;//加计数
            TIM_TimeBaseInitStructure.TIM_Period = 10-1;//ARR自动重装器的值
            TIM_TimeBaseInitStructure.TIM_Prescaler = 1-1;//PSC预分频的值
            TIM_TimeBaseInitStructure.TIM_RepetitionCounter = 0;//重复计数器的值（高级定时器才有）
            TIM_TimeBaseInit(TIM2, &TIM_TimeBaseInitStructure);
            TIM_ClearFlag(TIM2,TIM_FLAG_Update);//消除上一行TIM_TimeBaseInit立刻产生更新事件影响
            //5.配置中断输出控制
            TIM_ITConfig(TIM2, TIM_IT_Update, ENABLE);
            //6.配置NVIC
            NVIC_InitTypeDef NVIC_InitStructure;
            NVIC_InitStructure.NVIC_IRQChannel = TIM2_IRQn;
            NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
            NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 2 ;
            NVIC_InitStructure.NVIC_IRQChannelSubPriority = 1;
            NVIC_Init(&NVIC_InitStructure);
            //7.配置运行控制
            TIM_Cmd(TIM2, ENABLE);    
        }

        /*
        //TIM2定时中断后的中断函数
        void TIM2_IRQHandler(void){
            if(TIM_GetITStatus(TIM2,TIM_IT_Update)==SET){
                ??    
                TIM_ClearITPendingBit(TIM2,TIM_IT_Update);
            }
        }
        */
    ```

## 6.3 TIM 输出比较原理
TIM的OC（Output Compare）**输出比较**主要用于输出PWM波形，PWM又是驱动电机的必要条件（智能车、机器人等），所以应用广泛。**输出比较功能**可以通过比较**CNT计数器**与**CCR捕获/比较寄存器**（见图“通用定时器框图”）的大小，来对输出电平进行置1、置0或翻转的操作，用于输出一定频率和占空比的PWM波形。
- 每个高级定时器和通用定时器都拥有4个输出比较通道。
- 高级定时器的前3个通道额外拥有死区生成和互补输出的功能。

PWM原理示意图：
![PWM示意图](../images/6.12%20PWM示意图.png)

简单介绍一下PWM，即**PWM（Pulse Width Modulation）脉冲宽度调制**。在**具有惯性的系统**中，可以通过对一系列脉冲的宽度进行调制，来等效地输出的模拟参量，常应用于电机控速等领域。PWM参数：
- 频率 = 1 / TS，一般在几kHz~几十kHz。
- 占空比 = TON / TS。
- 分辨率 = 占空比变化步距，也就是占空比变化的精细程度。一般1%足够使用。
> 定时中断的频率就是PWM波的频率，只不过占空比的变化范围由自动重装载值决定。

有关“通用定时器框图”已经在6.1节“TIM定时中断原理”介绍过，下面来具体介绍其中的输出比较通道的电路结构。
![通用定时器-输出比较电路](../images/6.13%20通用定时器-输出比较电路.png)
- 左侧输入：CNT和CCR比较的结果。
- ETRF：定时器的小功能，一般不用。
- 输出模式控制器：CNT>=CCR1时，输出模式控制器收到信号并输出oc1ref。输出比较的8种模式如下：
    ![输出比较的8种模式](../images/6.14%20输出比较的8种模式.png)
- 主模式控制器：可以将oc1ref映射到主模式的TRGO输出上去。
- CC1P：极性选择。选择是否需要将oc1ref的高低电平翻转一下。
- 输出使能电路：由CC1E选择要不要输出。
- OC1：后续通过TIMx_CH1输出到GPIO引脚上。

![高级定时器-输出比较电路](../images/6.15%20高级定时器-输出比较电路.png)
相比于“通用定时器”，“高级定时器”的输出比较电路增加了“死区生成器”和“互补输出电路”。
- 死区生成器：消除上下两路可能同时导通的短暂状态，以防止电路发热、功率损耗。
- 互补输出OC1N：与OC1反相，用于驱动三相无刷电机。

![PWM基本结构](../images/6.15%20PWM基本结构.png)
上图与“定时器中断”的区别在于最后输出的时候不需要“更新事件”的终端申请，而是走输出比较电路。下面是一些参数计算：
- OWM频率：Freq = CK_PSC / (PSC + 1) / (ARR + 1)
- PWM占空比： Duty = CCR / (ARR + 1)
- PWM分辨率： Reso = 1 / (ARR + 1)

## 6.4 TIM输出比较相关实验
### 6.4.1 舵机简介
![舵机实物图](../images/6.17%20舵机实物图.png)
舵机是一种根据输入PWM信号 占空比来控制输出角度 的装置。
- 型号：SG90。
- 三种输入线：棕色是电源负、红色是电源正、橙色是PWM信号线。注意这个颜色因型号不同可能不同，需要查看说明手册。
- 输入PWM信号要求：周期为20ms（50Hz），高电平宽度为0.5ms~2.5ms（占空比2.5%~12.5%），脉冲控制精度为2us（0.18°，占空比精度0.01%）。
- 内部电板的基本思路：根据输入占空比得到期望角度，然后检测当前角度，若当前角度较小则顺时针转；反之则逆时针转，直到与期望角度相同。
> 这里实际上是将PWM当作一种通信协议，而不是实际输出。

![舵机电路图](../images/6.18%20舵机电路图.png)
- GND：STM32地。
- 5V电源线：舵机属于大功率设备，驱动电源也期望是大功率的输出设备，驱动电源要注意和stm32开发板共地。对于本实验来说，可以使用STLINK的5V输出引脚进行供电，属于USB供电符合功率要求，另外要求不严格也可以不共地。
- PWM：作为通信线无需大功率，可以连接到stm32的某个引脚，如PA0。

### 6.4.2 直流电机简介
![实物图-直流电机及驱动芯片](../images/6.19%20实物图-直流电机及驱动芯片.png)
直流电机是一种将电能转换为机械能的装置。
- 型号：130直流电机。
- 直流电机两个引脚：当电极正接时，电机正转；当电极反接时，电机反转。

直流电机属于大功率器件，GPIO口无法直接驱动，需要配合**电机驱动电路**来操作，如TB6612、DRV8833、L9110、L298N等。另外还有一些用分离元件（如MOS）做的驱动电路，支持更大的驱动功率。本实验采用的**TB6612**是一款双路H桥型的直流电机驱动芯片，可以驱动两个直流电机并且控制其转速和方向。而有些芯片（如ULN2003）一路就只有一个开关管，所以只能控制电机在一个方向转，选择电机的时候注意区分。

![电路图-TB6612](../images/6.20%20电路图-TB6612.png)
- VM：电机电源的正极，范围4.5V~10V。要接一个可以输出大电流的电源，且电压一般与电机的额定电压保持一致。
- VCC：逻辑电平输入端，范围2.7V~5.5V。这个要与控制控制器的电源保持一致，所以采用stm32就是3.3V、采用51单片机就是5V。
- 三个GND：都是电源负极，随便选一个接地即可。
- STBY：Stand by，待机控制引脚。接地，芯片处于待机状态不工作；接逻辑电源VCC，芯片正常工作。
- PWMA、AIN1、AIN2：接在单片机GPIO引脚上，用于控制电机，控制逻辑如上图。PWMA接PWM波，AIN1、AIN2可以任意接普通的GPIO口。
- AO1、AO2：按照控制逻辑，从VM汲取电流来驱动电机。
- PWMB、BIN1、BIN2与BO1、BO2：同上述，控制另一个电机的转动。
> 根据逻辑控制真值表，这里的PWM就是用来等效成一个模拟量。

## 6.4.3 实验：PWM驱动呼吸灯-引脚重映射
需求：实现0.5s逐渐亮、0.5s逐渐灭的呼吸灯，PA0高电平驱动。

PWM驱动呼吸灯-接线图：
![接线图-PWM呼吸灯](../images/6.21%20接线图-PWM呼吸灯.png)
PWM驱动呼吸灯-代码调用（非库函数）：
![代码调用-PWM呼吸灯](../images/6.22%20代码调用-PWM呼吸灯.png)

代码展示：OLED和Delay相关代码见前面，本节略。 
- main.c
    ``` C
        #include "stm32f10x.h"                  // Device header
        #include "OLED.h"
        #include "PWM.h"
        #include "Delay.h"

        int main(void){
            uint16_t pwm_duty = 0;//PWM波的占空比
            uint8_t  pwm_flag = 1;//占空比变化控制信号，1升0降
            
            //OLED显示屏初始化
            OLED_Init();
            OLED_ShowString(1,1,"BreathLED:");
            OLED_ShowString(2,1,"Init");
            //PWM初始化
            PWM_Init();
            
            while(1){
                
                Delay_ms(5);//0.5s完成100个占空比变化
                //调整占空比
                if(pwm_flag==1){
                    if(pwm_duty<100){
                        pwm_duty++;
                        OLED_ShowString(2,1,"Inhale");
                    }else if(pwm_duty==100){
                        pwm_flag = 0;
                    }else{
                        pwm_duty = 0;
                        pwm_flag = 1;
                    }
                }else if(pwm_flag == 0){
                    if(pwm_duty >0 && pwm_duty<=100){
                        pwm_duty--;
                        OLED_ShowString(2,1,"Exhale");
                    }else if(pwm_duty==0){
                        pwm_flag = 1;
                    }else{
                        pwm_duty = 0;
                        pwm_flag = 1;
                    }
                }else{
                    pwm_duty = 0;
                    pwm_flag = 1;
                }
                //改变占空比
                PWM_SetDuty(pwm_duty);
            };
        }
    ```
- PWM.h
    ``` C
        #ifndef __PWM_H
        #define __PWM_H

        void PWM_Init(void);
        void PWM_SetDuty(uint16_t pwm_duty);

        #endif
    ```
- PWM.c
    ``` C
        #include "stm32f10x.h"                  // Device header

        //TIM输出比较模式-PWM初始化
        void PWM_Init(void){
            //1.配置RCC
            RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2, ENABLE);
            RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
        //    //隐：引脚重映射，将PA0映射到PA15
        //    RCC_APB2PeriphClockCmd(RCC_APB2Periph_AFIO, ENABLE);//引脚重映射会使用AFIO
        //    GPIO_PinRemapConfig(GPIO_PartialRemap1_TIM2, ENABLE);//参考手册“8.3.7定时器复用功能重映射”
        //    GPIO_PinRemapConfig(GPIO_Remap_SWJ_JTAGDisable, ENABLE);//PA15、PB3、PB4变成普通IO口
            //2.选择时基单元时钟
            TIM_InternalClockConfig(TIM2);
            //3.配置时基单元-10kHz
            TIM_TimeBaseInitTypeDef TIM_TimeBaseInitStructure;
            TIM_TimeBaseInitStructure.TIM_ClockDivision = TIM_CKD_DIV1;
            TIM_TimeBaseInitStructure.TIM_CounterMode = TIM_CounterMode_Up;//向上计数
            TIM_TimeBaseInitStructure.TIM_Period = 100-1;//自动重装载值
            TIM_TimeBaseInitStructure.TIM_Prescaler = 72-1;//预分频
            TIM_TimeBaseInitStructure.TIM_RepetitionCounter = 0x0000;
            TIM_TimeBaseInit(TIM2, &TIM_TimeBaseInitStructure);
            //4.配置运行控制
            TIM_Cmd(TIM2, ENABLE);
            //5.配置输出捕获电路
            TIM_OCInitTypeDef TIM_OCInitStructure;
            TIM_OCStructInit(&TIM_OCInitStructure);//后续即使用到高级定时器初始化，也不会出错
            TIM_OCInitStructure.TIM_OCMode      = TIM_OCMode_PWM1;      //PWM模式1
            TIM_OCInitStructure.TIM_OCPolarity  = TIM_OCPolarity_High;
            TIM_OCInitStructure.TIM_OutputState = TIM_OutputState_Enable;
            TIM_OCInitStructure.TIM_Pulse       = 0x0000;                 //占空比
            TIM_OC1Init(TIM2, &TIM_OCInitStructure);
            //7.配置GPIO-PA0
            GPIO_InitTypeDef GPIO_InitStructure;
            GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;//复用推挽输出
            GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0;      //GPIO_Pin_15-引脚重映射
            GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
            GPIO_Init(GPIOA, &GPIO_InitStructure);
        }

        //设置PWM波的占空比
        //范围是0~100
        void PWM_SetDuty(uint16_t pwm_duty){
            TIM_SetCompare1(TIM2, pwm_duty);
        }
    ```

## 6.4.4 实验：PWM驱动舵机
需求：按下按键开关，舵机角度就变化一次，并在OLED显示屏上显示舵机当前的角度。
> 注1：按键开关PB1，舵机PWM接PA1（输出比较电路通道2）、电源接STLINK上的5V引脚、GND接面包板GND。 
> 注2：输入PWM信号要求周期为20ms(50Hz)，高电平宽度为0.5ms~2.5ms（占空比2.5%~12.5%），脉冲控制精度为2us(0.18°，占空比精度0.01%)。 
> 注3：舵机的三根输入线，棕色是电源负、红色是电源正、橙色是PWM信号线。

PWM驱动舵机-接线图:
![接线图-PWM驱动舵机](../images/6.23%20接线图-PWM驱动舵机.png)

PWM驱动舵机-代码调用（非库函数）