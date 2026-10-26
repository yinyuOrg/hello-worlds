# 3 GPIO通用输入输出口
> 笔记主要参考B站江科大自化协教学视频“[STM32入门教程-2023版](https://www.bilibili.com/video/BV1th411z7sn/)”。

## 3.1 GPIO输入输出原理
GPIO（General Purpose Input Output）**通用输入输出口**可配置为8种输入输出模式。引脚电平范围位0V~3.3V，部分引脚可容忍5V。**输出模式**下可控制端口输出高低电平，用以驱动LED、控制蜂鸣器、模拟通信协议输出时序等，当然若驱动大功率设备还需要添加驱动电路。**输入模式**下可读取端口的高低电平或电压，用于读取按键输入、外接模块电平信号输入、ADC电压采集、模拟通信协议接收数据等。

![GPIO基本结构](../images/3.1%20GPIO基本结构.png)

上图给出了GPIO的基本结构图。在STM32中，所有的GPIO都挂载在APB2外设总线上。命名方式采用GPIOA、GPIOB、GPIOC...的方式命名。每个GPIO模块内，主要包括寄存器、驱动器等。
- 寄存器就是一段特殊的存储器，内核可以通过APB2总线对寄存器进行读写，从而完成输出电平和读取电平的功能。该寄存器的每一位都对应一个引脚，由于STM32是32位的单片机，所以所有的寄存器都是32位的，也就是说只有寄存器的低16位对应上了响应的GPIO口。
- 驱动器就是增加信号的驱动能力的。

> STM32F103C8T6 芯片上48个引脚，除了基本的电源和晶振等维持系统正常运行的引脚外，分别包括PA0~PA15、PB0~PB15、PC13~PC15。

![GPIO位结构](../images/3.2%20GPIO位结构.png)

上图就是将“GPIO的基本结构”进行放大，得到的实际的位结构。

**输入部分**：
- 整体框架从左到右依次是寄存器、驱动器、IO引脚，从上到下分为“输入”、“输出”。
- 最右侧的IO引脚上两个保护二极管，其作用是对IO引脚的输出电压进行限幅在0~3.3V之间，进而可以避免过高的IO引脚输入电压对电路内部造成伤害。
- 输入驱动器的上、下拉电阻：相应的两个开关可以通过程序进行配置，分别有上拉输入模式（上开关导通&下开关断开）、下拉输入模式（下开关导通&上开关断开）、浮空输入模式（两个开关都断开）。上下拉电阻的作用就是给引脚输入提供一个默认的输入电平，进而避免引脚悬空导致的不确定。都属于弱上拉、弱下拉。
- 输入驱动器的触发器：这里是用肖特基管构成的施密特触发器。只有高于上限、低于下限电压才进行变化，作用是对输入电压进行整形，可以消除电压波纹、使电压的上升沿/下降沿更加陡峭。也就是说，**STM32的GPIO断开会自动对输入的数字电压进行整形**。
- “模拟输入”、“复用功能输入”：都是连接到片上外设的一些端口，前者用于ADC等需要模拟输入的外设，后者用于串口输入引脚等需要数字量的外设。

**输出部分**：
- 输出数据：可以由输出数据寄存器（平台的IO口输出）、片上外设来指定，数据选择控制数据来源。
- 位设置/清除寄存器：单独操作输出数据的某一位，而不是影响其他位。
- 驱动器中的MOS管：MOS管相当于一种开关，输出信号来控制这两个MOS管的开启状态，进而输出信号。可以选择推挽、开漏、关闭三种输出模式。
    1. 推挽输出模式：两个MOS管均有效，STM32对IO口有绝对的控制权，也称为强输出模式。
    2. 开漏输出模式：P-MOS无效。只有低电平有驱动能力，高电平输出高阻。
    3. 关闭模式：两个MOS管均无效，端口电平由外部信号控制。

补充：STM32如何将数据写入寄存器？
1. 通过软件的方式。由于STM32的寄存器智能进行整体读写，所以可以先将数据全部读出，然后代码中用`&=`清零、`|=`置位的方式改变单独某一位的数据，再将改写后的数据写回寄存器。此方法比较麻烦，效率不高，对于IO口进行操作不合适。
2. 通过**位设置/清除寄存器**。若对某一位 置1，只需对位设置寄存器的相应位 置1；若对某一位清零，则对清除寄存器相应位清零。这种方式通过内置电路完成操作，一步到位。
3. 通过读写STM32中的“位带”区域。在STM32中，专门分配一段地址区域，该区域影射了RAM和外设寄存器所有的位。读写这段地址中数据，就相当于读写所映射位置的某一位。

| 模式名称 | 性质 | 特征 |
|:---:|:---:|:---|
| 浮空输入 | 数字输入 | 可读取引脚电平，若引脚悬空则电平不确定，需要连续驱动源 | 
| 上拉输入 | 数字输入 | 可读取引脚电平，内部连接上拉电阻，悬空时默认高电平 | 
| 下拉输入 | 数字输入 | 可读取引脚电平，内部连接下拉电阻，悬空时默认高电平 | 
| 模拟输入 | 模拟输入 | GPIO无效，引脚直接接入内部ADC（ADC专属配置） | 
| 开漏输出 | 数字输出 | 可输出引脚电平，高电平为高阻态，低电平接VSS | 
| 推挽输出 | 数字输出 | 可输出引脚电平，高电平接VDD，低电平接VSS | 
| 复用开漏输出 | 数字输出 | 由片上外设控制，高电平为高阻态，低电平接VSS | 
| 复用推挽输出 | 数字输出 | 由片上外设控制，高电平接VDD，低电平接VSS | 

上表给出了GPIO的8种模式，通过配置GPIO的端口配置寄存器即可选择对应的模式。
1. 每一个端口的模式由4位进行控制，16个端口就需要64位，也就是两个32位寄存器，即端口配置低寄存器、端口配置高寄存器。
2. 输入输出下，输出无效；而输出模式下，输入有效。这是因为一个IO口只能有一个输出，但只有一个输入，所以直接将输出信号输入回去也没问题。

## 3.2 硬件介绍-LED、蜂鸣器、面包板
首先，简单介绍下stm32芯片外围的电路。
- LED：发光二极管，正向通电点亮，反向通电不亮。
- 有源蜂鸣器（本实验）：内部自带震荡源，将正负极接上直流电压即可持续发声，频率固定。上图所示的蜂鸣器使用三极管作为开关。
- 无源蜂鸣器：内部不带震荡源，需要控制器提供震荡脉冲才可发声，调整提供震荡脉冲的频率，可发出不同频率的声音。

实物图：
![LED和有源蜂鸣器实物图](../images/3.3%20LED和有源蜂鸣器实物图.png)
> LED长脚为正极、灯内部小头为正极。本实验的蜂鸣器低电平驱动。

![LED和蜂鸣器驱动电路设计](../images/3.4%20LED和蜂鸣器驱动电路设计.png)
上图给出了LED和蜂鸣器的驱动电路图。注意，**三极管的发射极一定要直接接正电源/地**，这是因为三极管的开启需要发射极和基极之间有一定的电压，如果接在负载侧则有可能会导致三极管无法正常开启。

![面包板实物图](../images/3.5%20面包板实物图.png)
上图给出了面包板的示意图。可以看出，面包板中间的金属爪是竖着排列的，用于插各种元器件；上下四排金属爪是横着排列的，一般用于供电。注意，**在使用面包板之前，一定要观察孔位的连接情况**。

## 3.3 实验：LED闪烁、LED流水灯、蜂鸣器提示
**需求1**：面包板上的LED以1s为周期进行闪烁。亮0.5s、灭0.5s......
- LED低电平驱动
- 需要用到延时函数`Delay.h`、`Delay.c`，在UP提供的程序源码中为了方便管理，应在工程内创建System文件夹，专门存放这些可以复用的代码。

![接线图-LED闪烁](../images/3.6%20接线图-LED闪烁.png)
> 实际上，应该在LED和驱动电源之间接上保护电阻，但是由于本电路过于简单，于是之间省略保护电阻。后面“LED流水灯”、“蜂鸣器提示”实验同样省略保护电阻。

![代码调用-LED闪烁](../images/3.7%20代码调用-LED闪烁.png)
代码展示：
- main.c
    ``` C
        #include "stm32f10x.h"
        #include "Delay.h"

        int main(void){
            // 开启APB2-GPIOA的外设时钟RCC
            RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
            // 初始化PA0端口：定义结构体及参数
            GPIO_InitTypeDef GPIO_InitStructure;
            GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0;
            GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
            GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
            GPIO_Init(GPIOA, &GPIO_InitStructure);

            // 下面是对GPIO端口赋值的常用的四种方式
            //    GPIO_ResetBits(GPIOA, GPIO_Pin_0);//复位PA0
            //    GPIO_SetBits(GPIOA, GPIO_Pin_0);//将PA0置1
            //    GPIO_WriteBit(GPIOA, GPIO_Pin_0, Bit_RESET);//将PA0清零
            //    GPIO_Write(GPIO_TypeDef* GPIOx, uint16_t PortVal);//此函数可以对16位端口
            while(1){
                // 正常思路
                GPIO_ResetBits(GPIOA, GPIO_Pin_0);//复位PA0
                Delay_ms(500);
                GPIO_SetBits(GPIOA, GPIO_Pin_0);//将PA0置1
                Delay_ms(500);

                // 使用GPIO_WriteBit函数，且强制类型转换
                GPIO_WriteBit(GPIOA, GPIO_Pin_0, (BitAction)0); //把0类型转换成BitAction
                Delay_ms(500);
                GPIO_WriteBit(GPIOA, GPIO_Pin_0, (BitAction)1);
                Delay_ms(500);
            }
        }
    ```
- Delay.h
    ``` C
        #ifndef __DELAY_H
        #define __DELAY_H

        void Delay_us(uint32_t us);
        void Delay_ms(uint32_t ms);
        void Delay_s(uint32_t s);

        #endif
    ```
- Delay.c
    ``` C
        #include "stm32f10x.h"
        
        /**
        * @brief  微秒级延时
        * @param  xus 延时时长，范围：0~233015
        * @retval 无
        */
        void Delay_us(unit32_y xus)
        {
            SysTick->LOAD = 72 * xus; // 设置定时器重装值
            SysTick->VAL = 0x00; //清空当前计数值
            SysTick->CTRL = 0x00000005;          //设置时钟源为HCLK，启动定时器
            while(!(SysTick->CTRL & 0x00010000));//等待计数到0
            SysTick->CTRL = 0x00000004;          //关闭定时器
        }

        /**
        * @brief  毫秒级延时
        * @param  xms 延时时长，范围：0~4294967295
        * @retval 无
        */
        void Delay_ms(uint32_t xms)
        {
            while(xms--)
            {
                Delay_us(1000);
            }
        }

        /**
        * @brief  秒级延时
        * @param  xs 延时时长，范围：0~4294967295
        * @retval 无
        */
        void Delay_s(uint32_t xs)
        {
            while(xs--)
            {
                Delay_ms(1000);
            }
        } 
    ```
> 此后`Delay.h`、`Delay.c`将作为常用函数长期存放于`System文件夹`中，后续如果使用到将直接调用，不会在笔记中展示源代码。

编程感想：
1. Keil编译后，整个工程会比较大，不利于分享给别人。可以使用UP提供的批处理程序，删掉工程中的中间文件后再分享给别人，其他人使用的时候只需要重新编译一下就行。
2. 本教程用到了RCC和GPIO两个外设，这些外设的库函数在Library中，一般存放在相应的`.h`文件的最后。
3. 将LED的短脚接负极，长脚接PA0口，就是高电平驱动方式，但是现象和低电平相同。
4. 将GPIO设置成开漏输出模式，可以发现高电平（高阻态）无驱动能力，低电平有驱动能力。

**需求2**：面包板上的8个LED以0.5s切换一个的速度，实现流水灯。低电平驱动。
![接线图-LED流水灯](../images/3.8%20接线图-LED流水灯.png)

代码调用关系与“LED闪烁”实验相同，代码如下：
- main.c
    ``` C
        #include "stm32f10x.h"                  // Device header
        #include "Delay.h"

        int main(void){
            // 开启APB2-GPIOA的外设时钟RCC
            RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
            // 初始化PA的8个端口：定义结构体及参数
            GPIO_InitTypeDef GPIO_InitStructure;
            //同时定义某几个端口
            GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0 | GPIO_Pin_1 | GPIO_Pin_2 | GPIO_Pin_3 | 
                                        GPIO_Pin_4 | GPIO_Pin_5 | GPIO_Pin_6 | GPIO_Pin_7;
            GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
            GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
            GPIO_Init(GPIOA, &GPIO_InitStructure);

            while(1){
                //使用GPIO_SetBits、GPIO_ResetBits进行赋值，这里仅用于演示“或操作”同时赋值
                GPIO_SetBits(GPIOA, GPIO_Pin_0 | GPIO_Pin_1 | GPIO_Pin_2 | GPIO_Pin_3 | 
                                    GPIO_Pin_4 | GPIO_Pin_5 | GPIO_Pin_6 | GPIO_Pin_7);
                GPIO_ResetBits(GPIOA, GPIO_Pin_0);
            //    //对指定的端口同时赋值
            //    GPIO_Write(GPIOA, ~0x01);
                Delay_ms(500);
                GPIO_Write(GPIOA, ~0x02);
                Delay_ms(500);
                GPIO_Write(GPIOA, ~0x04);
                Delay_ms(500);
                GPIO_Write(GPIOA, ~0x08);
                Delay_ms(500);
                GPIO_Write(GPIOA, ~0x10);
                Delay_ms(500);
                GPIO_Write(GPIOA, ~0x20);
                Delay_ms(500);
                GPIO_Write(GPIOA, ~0x40);
                Delay_ms(500);
                GPIO_Write(GPIOA, ~0x80);
                Delay_ms(500);
            };
        }
    ```
**需求3**：蜂鸣器不断地发出滴滴、滴滴......的提示音。蜂鸣器低电平触发。注：蜂鸣器执行四个动作为1个周期，分别是响0.1s、静0.1s、响0.1s、静0.7s。
![接线图-蜂鸣器提示](../images/3.9%20接线图-蜂鸣器提示.png)

代码调用关系与“LED闪烁”实验相同，下面是代码展示： 
- main.c
    ``` C
        #include "stm32f10x.h"                  // Device header
        #include "Delay.h"

        int main(void){
            // 开启APB2-GPIOB的外设时钟RCC
            RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE);
            // 初始化PB12端口：定义结构体及参数
            GPIO_InitTypeDef GPIO_InitStructure;
            GPIO_InitStructure.GPIO_Pin = GPIO_Pin_12;
            GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
            GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
            GPIO_Init(GPIOB, &GPIO_InitStructure);

            while(1){
                GPIO_ResetBits(GPIOB, GPIO_Pin_12);
                Delay_ms(100);
                GPIO_SetBits(GPIOB, GPIO_Pin_12);
                Delay_ms(100);
                GPIO_ResetBits(GPIOB, GPIO_Pin_12);
                Delay_ms(100);
                GPIO_SetBits(GPIOB, GPIO_Pin_12);
                Delay_ms(100);
                Delay_ms(600);
            };
        }

    ```
编程感想：
1. 控制蜂鸣器的IO端口可以随便选，但是不要选择三个JTAG调试端口：PA15、PB3、PB4。本实验选择PB12端口进行输出。
2. 关于调用库函数，有以下几种方法：
    - 直接查看每一个外设的`.h`函数，拖到最后就可以看到本外设的所有库函数，然后在对应的`.c`文件中查看函数定义和调用方式即可。
    - 查看库函数的用户手册——“STM32F103xx固件函数库用户手册.pdf”，这个中文版比较老；新版本的用户手册可以在ST公司的帮助文档中查看，但只有英文版。

## 3.4 硬件介绍-按键开关、光敏电阻
![原理图-按键开关](../images/3.10%20原理图-按键开关.png)
按键是最常见的输入设备，按下导通，松手断开。由于按键内部使用的是机械式弹簧片来进行通断的，所以在按下和松手的瞬间会伴随有一连串的抖动。虽然前面已经说过，GPIO端口有专门的肖特基触发器对输入信号进行整形，但按键开关的抖动幅度大、时间长，所以还是**需要“软件消抖”**。基本思路就是延迟5~10ms，跳过抖动时间范围即可。

![电路设计-按键开关](../images/3.11%20电路设计-按键开关.png)
上图给出了按键开关的硬件电路设计图。对于按键开关来说，常见以上四种设计方法，而行业规范中，单片机端口一般都有上拉输入模式（弱上拉），所有**基本上就选择内部上拉/外部上拉的设计电路**。
- 如果电路同时存在内部上拉和外部上拉，那么其高电平的驱动能力更强，但是低电平会更加耗电。两个下拉电路则可以使低电平驱动能力更强，而不会明显增加损耗。
- 浮空输入模式下，每部没有上下拉，此时必须在外部有上下拉电路。
- 注意**内部和外部的上下拉模式必须一致！**内部有上下拉时，就可以不用配置外部上下拉。

![实物图-传感器模块](../images/3.12%20实物图-传感器模块.png)

上面给出了传感器模块的实物图，从左到右依次是光敏电阻传感器、热敏电阻传感器、对射式红外传感器、反射式红外传感器。

传感器元件的电阻会随外界模拟量的变化而变化，通过与定值电阻分压即可得到模拟电压输出，再通过电压比较器进行二值化即可得到数字电压输出。

![原理图-传感器模块](../images/3.13%20原理图-传感器模块.png)

上面所给出的原理图则是一个比较通用的传感器模块格式：
- 左起第三个模块：下面的可变电阻就是各种传感器模块所对应的阻值，与上面的分压电阻R1进行分压，进而输出模拟电压值AO。电容C2是滤波电容。
- 左起第一个模块：使用LM393模块，通过运算放大器实现“电压比较器”的功能。IN- 是一个可以调节的阈值，IN+ 则直接连接传感器的模拟分压AO，当AO > IN- 时，数字输出DO拉高；当AO < IN- 时，数字输出DO拉低。
- 左起第二个模块：通过一个滑动变阻器实现比较电压IN- 的调整。
- 左起第四个模块：电源指示灯。
- 左起第五个模块：传感器模块的端口。LED2用于指示数字输出DO的值。注意R5上拉电阻保证DO的默认值为高。

而对于传感器模块的电路设计来说，由于采用模块的方案，所以直接给传感器接上VCC和GND，然后将模拟信号AO和数字信号DO接在STM32的对应端口上即可。本次实验采用数字信号DO接入，关于模拟信号接入的使用方法在后面AD/DA的实验中继续讲解。

## 3.5 实验：按键控制LED、光敏传感器控制蜂鸣器

需求1：一个按键开关控制一个LED，每次按下按键，LED就改变自己的亮灭状态；两套系统互不影响。
- LED低电平驱动。
- 按键B11控制LEDA2，按键B1控制LEDA1。
- LED的状态改变是“松开触发”。

接线图-按键控制LED:
![接线图-按键控制LED](../images/3.14%20接线图-按键控制LED.png)

![代码调用-按键控制LED](../images/3.15%20代码调用-按键控制LED.png)

代码展示：
- main.c
    ``` C
        #include "stm32f10x.h"                  // Device header
        #include "Delay.h"
        #include "LED.h"
        #include "Key.h"

        int main(void){    
            LED_Init();
            Key_Init();
            
            while(1){
                if(Key_GetNum()==1){LED1_TURN();}
                else if(Key_GetNum()==2){LED2_TURN();}        
            };
        }
    ```
- LED.h
    ``` C
        #ifndef __LED_H
        #define __LED_H

        void LED_Init(void);
        void LED1_TURN(void);
        void LED2_TURN(void);

        #endif
    ```
- LED.c
    ``` C
        #include "stm32f10x.h"                  // Device header

        /**
        * @brief  初始化PA2、PA1作为两个LED的输出端口
        * @param  无
        * @retvl  无
        */
        void LED_Init(void){
            // 开启APB2-GPIOA的外设时钟RCC
            RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
            // 初始化PA的输出端口：定义结构体及参数
            GPIO_InitTypeDef GPIO_InitStructure;
            GPIO_InitStructure.GPIO_Pin = GPIO_Pin_2 | GPIO_Pin_1;
            GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
            GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;//推挽输出
            GPIO_Init(GPIOA, &GPIO_InitStructure);
            //默认输出为低电平-LED初始不亮
            GPIO_SetBits(GPIOA, GPIO_Pin_2 | GPIO_Pin_1);
        };

        /**
        * @brief  LED1状态翻转
        * @param  无
        * @retvl  无
        */
        void LED1_TURN(void){
            if(GPIO_ReadOutputDataBit(GPIOA, GPIO_Pin_1)==1){
                GPIO_ResetBits(GPIOA, GPIO_Pin_1);
            }else{
                GPIO_SetBits(GPIOA, GPIO_Pin_1);
            }
        }

        /**
        * @brief  LED2状态翻转
        * @param  无
        * @retvl  无
        */
        void LED2_TURN(void){
            if(GPIO_ReadOutputDataBit(GPIOA, GPIO_Pin_2)==1){
                GPIO_ResetBits(GPIOA, GPIO_Pin_2);
            }else{
                GPIO_SetBits(GPIOA, GPIO_Pin_2);
            }
        }
    ```
- Key.h
    ``` C
        #ifndef __KEY_H
        #define __KEY_H

        void Key_Init(void);
        uint8_t Key_GetNum(void);

        #endif
    ```
- Key.c
    ``` C
        #include "stm32f10x.h"                  // Device header
        #include "Delay.h"

        /**
        * @brief  初始化B11、B1作为按键2、按键1
        * @param  无
        * @retvl  无
        */
        void Key_Init(void){
            RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE);
            GPIO_InitTypeDef GPIO_InitStructure;
            GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;//上拉输入
            GPIO_InitStructure.GPIO_Pin = GPIO_Pin_11 | GPIO_Pin_1;
            GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;//只有输出才有速度等级，但是这里也可以定义
            GPIO_Init(GPIOB, &GPIO_InitStructure);
        };

        /**
        * @brief  检测哪个按键已经按下-松开触发
        * @param  无
        * @retvl  返回按下的按键编号
        *     @arg  0,1,2
        */
        uint8_t Key_GetNum(void){
            uint8_t keynum = 0;
            if(GPIO_ReadInputDataBit(GPIOB, GPIO_Pin_11)==0){
                Delay_ms(20);
                while(GPIO_ReadInputDataBit(GPIOB, GPIO_Pin_11)==0);
                Delay_ms(20);
                keynum = 2;
            }
            if(GPIO_ReadInputDataBit(GPIOB, GPIO_Pin_1)==0){
                Delay_ms(20);
                while(GPIO_ReadInputDataBit(GPIOB, GPIO_Pin_1)==0);
                Delay_ms(20);
                keynum = 1;
            }
            return keynum;
        };
    ```

**需求2**：光敏电阻被遮挡，蜂鸣器长鸣，光敏电阻不被遮挡，蜂鸣器不响。
- 蜂鸣器低电平驱动。
- 光敏传感器，光强越强阻值越小，分压越小；DO的LED指示灯低电平驱动。

![接线图-光敏传感器控制蜂鸣器](../images/3.16%20接线图-光敏传感器控制蜂鸣器.png)

![代码调用-光敏传感器控制蜂鸣器](../images/3.17%20代码调用-光敏传感器控制蜂鸣器.png)

代码展示：
- main.c
    ``` C
        #include "stm32f10x.h"                  // Device header
        #include "Buzzer.h"
        #include "LightSensor.h"

        int main(void){    
            Buzzer_Init();
            LightSensor_Init();
            while(1){
                if(LightSensor_Get()==1){Buzzer_ON();}
                else                    {Buzzer_OFF();}
            };
        }
    ```
- Buzzer.h
    ``` C
        #ifndef __BUZZER_H
        #define __BUZZER_H

        void Buzzer_Init(void);
        void Buzzer_ON(void);
        void Buzzer_OFF(void);

        #endif
    ```
- Buzzer.c
    ``` C
        #include "stm32f10x.h"                  // Device header

        /**
        * @brief  蜂鸣器初始化-PB12推挽输出
        */
        void Buzzer_Init(void){
            RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE);
            GPIO_InitTypeDef GPIO_InitStructure;
            GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;//推挽输出
            GPIO_InitStructure.GPIO_Pin = GPIO_Pin_12;
            GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
            GPIO_Init(GPIOB, &GPIO_InitStructure);
            
            GPIO_SetBits(GPIOB, GPIO_Pin_12);
        }

        /**
        * @brief  蜂鸣器开启-低电平驱动
        */
        void Buzzer_ON(void){
            GPIO_ResetBits(GPIOB, GPIO_Pin_12);
        }

        /**
        * @brief  蜂鸣器开启-低电平驱动
        */
        void Buzzer_OFF(void){
            GPIO_SetBits(GPIOB, GPIO_Pin_12);
        }
    ```
- LightSensor.h
    ``` C
        #ifndef __LIGHTSENSOR_H
        #define __LIGHTSENSOR_H

        void LightSensor_Init(void);
        uint8_t LightSensor_Get(void);

        #endif
    ```
- LightSensor.c
    ``` C
        #include "stm32f10x.h"                  // Device header

        /**
        * @brief  光敏传感器初始化-PB13上拉输入
        */
        void LightSensor_Init(void){
            RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE);
            GPIO_InitTypeDef GPIO_InitStructure;
            GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;//上拉输入
            GPIO_InitStructure.GPIO_Pin = GPIO_Pin_13;
            GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
            GPIO_Init(GPIOB, &GPIO_InitStructure);
        }

        /**
        * @brief  读取光敏传感器的数字输入信号-光强越强分压越低
        * @retvl  读取到的数字输入信号0/1
        */
        uint8_t LightSensor_Get(void){
            return GPIO_ReadInputDataBit(GPIOB, GPIO_Pin_13);
        }
    ```

## 3.6 C语言的语法
1. 数据类型

| 关键字 | 位数 | 表示范围 | stdint关键字 | ST关键字 |
|---|---|---|---|---|
| char | 8 | -128 ~ 127 | int8_t | s8 |
| unsigned char | 8 | 0 ~ 255 | uint8_t | u8 |
| short | 16 | -32768 ~ 32767 | int16_t | s16 |
| unsigned short | 16 | 0 ~ 65535 | uint16_t | u16 |
| int | 32 | -2147483648 ~ 2147483647 | int32_t | s32 |
| unsigned int | 32 | 0 ~ 4294967295 | uint32_t | u32 |
| long | 32 | -2147483648 ~ 2147483647 |  |  |
| unsigned long | 32 | 0 ~ 4294967295 |  |  |
| long long | 64 | -(2^64)/2 ~ (2^64)/2-1 | int64_t |  |
| unsigned long long | 64 | 0 ~ (2^64)-1 | uint64_t |  |
| float | 32 | -3.4e38 ~ 3.4e38 |  |  |
| double | 64 | -1.7e308 ~ 1.7e308 |  |  |

2. 宏定义`#define`关键字`#define`，主要用于：用一个字符串代替一个数字，便于理解，防止出错；或者提前程序中经常出现的参数，便于快速修改。
    ``` C
        //定义宏定义：
        #define ABC 12345
        //引用宏定义：
        int a = ABC;  //等效于int a = 12345;
    ```
3. 关键字typedef关键字typedef，常用于将一个比较长的变量类型名换个名字，便于使用。相比于`#define`来说，`typedef`在进行改名时会进行变量类型检查，所以更加安全。
    ``` C
        //定义typedef：
        typedef unsigned char uint8_t;
        //引用typedef：
        uint8_t num1; //等效于unsigned char num1;
    ```
4. 结构体关键字 struct，用途：数据打包，将不同类型变量组成一个集合。
     ``` C
        //在main函数中定义结构体变量：
        struct{
            char x;
            int y;
            float z;
        } StructName;
        //因为结构体变量类型较长，所以通常在main函数外用typedef更改变量类型名
        typedef struct{
            char x;
            int y;
            float z;
        } StructName;

        //引用结构体成员：方法一
        StructName struct1;
        struct1.x = 'A';
        struct1.y = 66;
        struct1.z = 1.23;
        //引用结构体成员：方法二
        pStructName->x = 'A';   //pStructName为结构体的地址
        pStructName->y = 66;
        pStructName->z = 1.23;
    ```
5. 枚举类型 关键字 enum，用途：定义一个取值受限制的整型变量，用于限制变量取值范围；枚举也相当于一个宏定义的集合，可以直接把里面的枚举变量拿出来用。注意枚举变量用逗号隔开，且最后一个枚举变量不加逗号。
    ``` C
        //函数内定义枚举变量：
        enum{
            FALSE = 0,
            TRUE  = 1
        } EnumName;
        //因为枚举变量类型较长，所以通常在函数外用typedef更改变量类型名
        typedef enum{
            FALSE = 0,
            TRUE  = 1
        } EnumName;

        //引用枚举成员：
        EnumName emu1;
        emu1 = FALSE;
        emu1 = TRUE;
    ```