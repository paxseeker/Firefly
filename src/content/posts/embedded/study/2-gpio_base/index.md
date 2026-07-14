---
tags:
  - 嵌入式
title: GPIO基础
description: GPIO寄存器操作基础
category: 嵌入式
published: 2026-05-22
draft: false
image: https://imgbed.paxseeker.xyz/file/1784009580948_image.png
---
# GPIO基础
## GPIO结构图
![](https://imgbed.paxseeker.xyz/file/1784009580948_image.png)
## GPIO寄存器
- 两个32位配置寄存器(GPIOx_CRL，GPIOx_CRH)
- 两个32位数据寄存器 (GPIOx_IDR和GPIOx_ODR)
- 一个32位置位/复位寄存器(GPIOx_BSRR)
- 一个16位复位寄存 器(GPIOx_BRR)
- 一个32位锁定寄存器(GPIOx_LCKR)
### 地址图
![](https://imgbed.paxseeker.xyz/file/1783996958064_image.png)

```c
#define GPIOC_BASE      0x40011000      // GPIOC寄存器基地址
#define GPIOC_CRL       (*(volatile unsigned int*)(GPIOC_BASE))         // 配置低32位寄存器
#define GPIOC_CRH       (*(volatile unsigned int*)(GPIOC_BASE + 0x04))  // 配置高32位寄存器
#define GPIOC_IDR       (*(volatile unsigned int*)(GPIOC_BASE + 0x08))  // 输入数据寄存器
#define GPIOC_ODR       (*(volatile unsigned int*)(GPIOC_BASE + 0x0C))  // 输出数据寄存器
#define GPIOC_BSRR      (*(volatile unsigned int*)(GPIOC_BASE + 0x10))  // 置位/复位寄存器
#define GPIOC_BRR       (*(volatile unsigned int*)(GPIOC_BASE + 0x14))  // 复位寄存器
#define GPIOC_LCKR      (*(volatile unsigned int*)(GPIOC_BASE + 0x18))  // 锁定寄存器

```

当然，可以直接用C结构体定义
```c
#define __IO volatile // 告诉编译器不要优化，每次去实际地址获取值
#define GPIOC_BASE      0x40011000  
typedef struct {
  __IO uint32_t CRL;
  __IO uint32_t CRH;
  __IO uint32_t IDR;
  __IO uint32_t ODR;
  __IO uint32_t BSRR;
  __IO uint32_t BRR;
  __IO uint32_t LCKR;
} GPIO_TypeDef;

#define GPIOC ((GPIO_TypeDef*)(GPIOC_BASE))
```
## 配置寄存器

![](https://imgbed.paxseeker.xyz/file/1783999208167_image.png)
> [!tip] c8t6每个GPIO一共有16个引脚，对应0-15
> - 配置寄存器包括高位和低位两个32位寄存器，一共64位，每个引脚对应4位配置
> - 低两位为Mode，用于设置选择输入还是输出
> - 高两位为CNF，用于选择输入输出模式

由此我们可以写出初始化GPIO的方法
```c
typedef struct {
    uint32_t Pin;
    uint32_t Mode;
    uint32_t Speed;
    uint32_t Pull;
} GPIO_InitTypeDef;

#define GPIO_Pin_0                 ((uint16_t)0x0001)  /*!< Pin 0 selected */
#define GPIO_Pin_1                 ((uint16_t)0x0002)  /*!< Pin 1 selected */
#define GPIO_Pin_2                 ((uint16_t)0x0004)  /*!< Pin 2 selected */
#define GPIO_Pin_3                 ((uint16_t)0x0008)  /*!< Pin 3 selected */
#define GPIO_Pin_4                 ((uint16_t)0x0010)  /*!< Pin 4 selected */
#define GPIO_Pin_5                 ((uint16_t)0x0020)  /*!< Pin 5 selected */
#define GPIO_Pin_6                 ((uint16_t)0x0040)  /*!< Pin 6 selected */
#define GPIO_Pin_7                 ((uint16_t)0x0080)  /*!< Pin 7 selected */
#define GPIO_Pin_8                 ((uint16_t)0x0100)  /*!< Pin 8 selected */
#define GPIO_Pin_9                 ((uint16_t)0x0200)  /*!< Pin 9 selected */
#define GPIO_Pin_10                ((uint16_t)0x0400)  /*!< Pin 10 selected */
#define GPIO_Pin_11                ((uint16_t)0x0800)  /*!< Pin 11 selected */
#define GPIO_Pin_12                ((uint16_t)0x1000)  /*!< Pin 12 selected */
#define GPIO_Pin_13                ((uint16_t)0x2000)  /*!< Pin 13 selected */
#define GPIO_Pin_14                ((uint16_t)0x4000)  /*!< Pin 14 selected */
#define GPIO_Pin_15                ((uint16_t)0x8000)  /*!< Pin 15 selected */
#define GPIO_Pin_All               ((uint16_t)0xFFFF)  /*!< All pins selected */

#define GPIO_Speed_10MHz    0x01
#define GPIO_Speed_2MHz     0x02
#define GPIO_Speed_50MHz    0x03

#define GPIO_MODE_INPUT           0x00000000u  /*!< 输入模式 (具体浮空/上拉/下拉由Pull决定) */
#define GPIO_MODE_OUTPUT_PP       0x00000001u  /*!< 推挽输出 */
#define GPIO_MODE_AF_PP           0x00000002u  /*!< 复用推挽 */
#define GPIO_MODE_ANALOG          0x00000003u  /*!< 模拟模式 */
#define GPIO_MODE_OUTPUT_OD       0x00000011u  /*!< 开漏输出 */
#define GPIO_MODE_AF_OD           0x00000012u  /*!< 复用开漏 */

#define GPIO_NOPULL               0x00000000u  /*!< 无上下拉（浮空） */
#define GPIO_PULLUP               0x00000001u  /*!< 上拉 */
#define GPIO_PULLDOWN             0x00000002u  /*!< 下拉 */ 


void GPIO_Init(GPIO_TypeDef* GPIOx, GPIO_InitTypeDef* GPIO_InitStruct)
{
    uint8_t pinpos;
    uint32_t pin_mask = GPIO_InitStruct->Pin;

    // 从 Mode 中提取低 2 位（0=输入, 1=输出, 2=复用, 3=模拟）
    uint32_t moder = GPIO_InitStruct->Mode & 0x03;

    // 提取 bit4 判断是否为开漏模式（1=开漏，0=推挽）
    uint32_t is_open_drain = (GPIO_InitStruct->Mode & 0x10) ? 1 : 0;

    // 提取速度（1=10MHz, 2=2MHz, 3=50MHz）
    uint32_t speed = GPIO_InitStruct->Speed & 0x03;

    // 提取上下拉（0=无, 1=上拉, 2=下拉）
    uint32_t pull = GPIO_InitStruct->Pull & 0x03;

    for (pinpos = 0; pinpos < 16; pinpos++)
    {
        // 检查当前引脚是否被选中
        if ((pin_mask >> pinpos) & 0x01)
        {
            uint32_t cnf = 0;      // CNF[1:0] 字段
            uint32_t mode = 0;     // MODE[1:0] 字段
            uint32_t reg_value;    // 最终写入 CRL/CRH 的 4 位值

            // ----- 根据 moder 和开漏标志决定 CNF -----
            switch (moder)
            {
                case 0: // 输入模式
                    // 输入模式下，CNF 由 Pull 决定
                    if (pull == GPIO_PULLUP || pull == GPIO_PULLDOWN)
                    {
                        cnf = 2;   // 10 = 上拉/下拉输入
                        // 上下拉的控制通过 ODR 实现
                        if (pull == GPIO_PULLUP)
                            GPIOx->ODR |= (1 << pinpos);   // 上拉：ODR=1
                        else
                            GPIOx->ODR &= ~(1 << pinpos);  // 下拉：ODR=0
                    }
                    else
                    {
                        cnf = 1;   // 01 = 浮空输入
                    }
                    mode = 0;      // 输入模式 MODE=00
                    break;

                case 1: // 通用输出模式
                    cnf = is_open_drain ? 1 : 0;  // 开漏=01, 推挽=00
                    mode = speed;  // MODE 由速度决定
                    break;

                case 2: // 复用功能模式
                    cnf = is_open_drain ? 3 : 2;  // 复用开漏=11, 复用推挽=10
                    mode = speed;  // MODE 由速度决定
                    break;

                case 3: // 模拟模式
                default:
                    cnf = 0;
                    mode = 0;
                    break;
            }

            // 合成最终写入寄存器的 4 位值: (CNF << 2) | MODE
            reg_value = (cnf << 2) | mode;

            // ----- 写入 CRL 或 CRH（读-改-写） -----
            if (pinpos < 8)
            {
                GPIOx->CRL &= ~(0x0F << (pinpos * 4));
                GPIOx->CRL |= (reg_value << (pinpos * 4));
            }
            else
            {
                uint8_t offset = pinpos - 8;
                GPIOx->CRH &= ~(0x0F << (offset * 4));
                GPIOx->CRH |= (reg_value << (offset * 4));
            }
        }
    }
}


```

## 端口位设置/清除寄存器(GPIOx_BSRR)
![](https://imgbed.paxseeker.xyz/file/1784004390081_image.png)

> [!todo] 为啥能直接操作ODR,还要有BSRR？BSRR包括BRR功能，为啥还有BRR？

读取和修改GPIO的方法
```c
void GPIO_WirtePin(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin, uint8_t level) {
    if (level) {
        GPIOx->BSRR = GPIO_Pin; 
    } else {
        GPIOx->BRR = GPIO_Pin;
        //GPIOx->BSRR = GPIO_Pin << 16;
    }
}

uint8_t GPIO_ReadPin(GPIO_TypeDef *GPIOx, uint16_t GPIO_Pin) {
    return (GPIOx->IDR & GPIO_Pin) ? 1 : 0;
}

void GPIO_SetPin(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin) {
    GPIOx->BSRR = GPIO_Pin; 
}
void GPIO_ResetPin(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin) {
    GPIOx->BRR = GPIO_Pin;
    //GPIOx->BSRR = GPIO_Pin << 16;
}
void GPIO_TogglePin(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin) {
    //GPIOx->ODR ^= GPIO_Pin;
    uint16_t toggle_mask = GPIOx->ODR & GPIO_Pin;
    GPIOx->BRR = toggle_mask;
    GPIOx->BSRR = (GPIO_Pin & ~toggle_mask);
}
``` 

