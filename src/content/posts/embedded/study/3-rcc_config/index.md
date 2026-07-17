---
title: STM32时钟树
tags:
  - 嵌入式
draft: false
published: 2026-05-23
description: 初始化系统时钟
image: https://imgbed.paxseeker.xyz/file/1784096129975_image.png
category: 嵌入式
---
# 时钟树

![image.png](https://imgbed.paxseeker.xyz/file/1784096129975_image.png)

> [!info] STM32F10x 有四个时钟源
> - HSE
> - HSI （由内部8MHz的RC振荡器产生， 当没有配置时钟树时，默认使用HSI作为SYSCLK）
> - LSE
> - LSI

## RCC寄存器地址映射（0x4002 1000 - 0x4002 13FF）
![image.png](https://imgbed.paxseeker.xyz/file/1784097736810_image.png)

```c
#define RCC_BASE        0x40021000      // RCC寄存器基地址

typedef struct {
    __IO uint32_t CR;           // 0x00 时钟控制寄存器
    __IO uint32_t CFGR;         // 0x04 时钟配置寄存器
    __IO uint32_t CIR;          // 0x08 时钟中断寄存器
    __IO uint32_t APB2RSTR;     // 0x0C APB2 外设复位寄存器
    __IO uint32_t APB1RSTR;     // 0x10 APB1 外设复位寄存器
    __IO uint32_t AHBENR;       // 0x14 AHB 外设时钟使能寄存器
    __IO uint32_t APB2ENR;      // 0x18 APB2 外设时钟使能寄存器
    __IO uint32_t APB1ENR;      // 0x1C APB1 外设时钟使能寄存器
    __IO uint32_t BDCR;         // 0x20 备份域控制寄存器
    __IO uint32_t CSR;          // 0x24 控制/状态寄存器
} RCC_TypeDef;
```
## RCC寄存器
### 时钟控制寄存器(RCC_CR)

![image.png](https://imgbed.paxseeker.xyz/file/1784096965288_image.png)
![image.png](https://imgbed.paxseeker.xyz/file/1784097058342_image.png)
### 时钟配置寄存器(RCC_CFGR)
![image.png](https://imgbed.paxseeker.xyz/file/1784099737827_image.png)
![image.png](https://imgbed.paxseeker.xyz/file/1784099829049_image.png)

### 初始化时钟，APB2到72MHz
```c
#define FLASH_ACR (*(volatile uint32_t*)0x40022000)

void SystemClk_Init() {
    // 步骤1：开启 HSE，等待就绪
    RCC->CR |= (1 << 16);           // HSEON = 1
    while (!(RCC->CR & (1 << 17))); // 等待 HSERDY = 1

    // 步骤2：配置 Flash 等待周期（必须在升频前！）
    FLASH_ACR &= ~0x07;             // 清除原有 LATENCY
    FLASH_ACR |= 0x02;              // 2 等待周期 (48~72MHz)
    FLASH_ACR |= (1 << 4);          // 可选：开启预取缓冲区 (PRFTBE)

    // 步骤3：配置 PLL 为 HSE×9
    RCC->CFGR &= ~0x003C0000;       // 清除 PLLMUL
    RCC->CFGR |= (0x07 << 18);      // PLLMUL = 0111 = ×9
    RCC->CFGR |= (1 << 16);         // PLLSRC = HSE

    // 步骤4：配置总线分频
    RCC->CFGR &= ~0x00000FF0;       // 清除 HPRE/PPRE1/PPRE2
    RCC->CFGR |= 0x00000000;        // AHB 不分频 (HPRE=0) → HCLK=72MHz
    RCC->CFGR |= 0x00000400;        // APB1 二分频 (PPRE1=100) → PCLK1=36MHz
    RCC->CFGR |= 0x00000000;        // APB2 不分频 (PPRE2=0) → PCLK2=72MHz

    // 步骤5：开启 PLL，等待就绪
    RCC->CR |= (1 << 24);           // PLLON = 1
    while (!(RCC->CR & (1 << 25))); // 等待 PLLRDY = 1

    // 步骤6：切换系统时钟源到 PLL
    RCC->CFGR &= ~0x03;             // 清除 SW[1:0]
    RCC->CFGR |= 0x02;              // SW = 10 (PLL 作为系统时钟)
    while ((RCC->CFGR & 0x0C) != (0x02 << 2)); // 等待 SWS == SW}

}
```

>[!warning]
> **Flash 等待周期必须与时钟频率相匹配**
> 
> STM32F103 的 Flash 存储器物理读取速度有限，无法跟上 72MHz CPU 的取指速度。
> 若不设置等待周期（72MHz 需 2 WS），CPU 会读到错误/无效指令，导致程序跑飞、HardFault 或行为异常。
> ![image.png](https://imgbed.paxseeker.xyz/file/1784100707874_image.png)
> ![image.png](https://imgbed.paxseeker.xyz/file/1784100871452_image.png)
> **必须先配置 Flash 等待周期，再切换系统时钟到高频！**

