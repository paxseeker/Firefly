---
title: 中断--串口通信
tags:
  - 嵌入式
  - HAL库
  - 串口通信
category: 嵌入式
published: 2026-05-24
draft: false
description: 中断，USART中断处理数据
image: https://imgbed.paxseeker.xyz/file/1784281452500_morning-moutaine.jpg
---
# 中断--串口通信

>[!tip]
> **中断四大模块寄存器全景图**
> 
> ```
> 外部信号（按键/传感器）
>        │
>        ▼
> ┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
> │    GPIO     │────→│    AFIO     │────→│    EXTI     │────→│    NVIC     │
> │  引脚输入    │     │  EXTI线选择  │     │  边沿检测    │     │  中断仲裁    │
> │             │     │             │     │  中断/事件    │     │  优先级管理  │
> └─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
>        ↑                                    │                    │
>        └────────────────────────────────────┘                    │
>              软件触发（SWIER）                                      │
>                                                                    ▼
>                                                             ┌─────────────┐
>                                                             │    CPU      │
>                                                             │  执行ISR     │
>                                                             └─────────────┘
> 
> 内部外设（USART/TIM/ADC/DMA...）
>        │
>        └────────────────────────────────────────────────────────→  NVIC
>              直接连 NVIC，不经过 GPIO/AFIO/EXTI
> ```
> 
> ---
> 
> **模块1：GPIO（通用输入输出）**
> 
> | 寄存器 | 偏移 | 作用 | 中断相关配置 |
> |--------|------|------|-------------|
> | CRL | 0x00 | 配置低8位引脚（PA0~PA7） | MODE=00输入，CNF=01浮空/10上拉下拉 |
> | CRH | 0x04 | 配置高8位引脚（PA8~PA15） | 同上 |
> | IDR | 0x08 | 输入数据寄存器，读取引脚电平 | 中断源状态 |
> | ODR | 0x0C | 输出数据寄存器 | 上拉时写1，下拉时写0 |
> | BSRR | 0x10 | 置位/复位寄存器 | 快速翻转引脚 |
> 
> GPIO 是**信号入口**，决定引脚是输入还是输出。中断模式下必须配成输入。
> 
> ---
> 
> **模块2：AFIO（复用功能I/O）**
> 
> | 寄存器 | 偏移 | 作用 | 中断相关 |
> |--------|------|------|---------|
> | EVCR | 0x00 | 事件输出控制 | 几乎不用 |
> | MAPR | 0x04 | **重映射寄存器** | USART/TIM 等换引脚，与中断无关 |
> | EXTICR[0] | 0x08 | EXTI0~3 线选择 | bit3:0=0000选PA0，0001选PB0... |
> | EXTICR[1] | 0x0C | EXTI4~7 线选择 | 同上 |
> | EXTICR[2] | 0x10 | EXTI8~11 线选择 | 同上 |
> | EXTICR[3] | 0x14 | EXTI12~15 线选择 | 同上 |
> 
> AFIO 是**多路选择器**，把某个端口的引脚连到对应的 EXTI 线。
> 同一数字编号的引脚（PA0/PB0/PC0）共享 EXTI0，AFIO 决定当前用哪个。
> 
> > 注意：AFIO 不参与 USART/TIM/ADC 等内部外设中断，只负责外部引脚中断的路由。
> 
> ---
> 
> **模块3：EXTI（外部中断/事件控制器）**
> 
> | 寄存器 | 偏移 | 位宽 | 作用 |
> |--------|------|------|------|
> | IMR | 0x00 | 32bit | **中断屏蔽寄存器**，bit0~19 对应 EXTI0~19，1=中断使能 |
> | EMR | 0x04 | 32bit | **事件屏蔽寄存器**，1=事件使能（唤醒CPU，不进ISR）|
> | RTSR | 0x08 | 32bit | **上升沿触发选择**，bitN=1 表示EXTIN上升沿触发 |
> | FTSR | 0x0C | 32bit | **下降沿触发选择**，bitN=1 表示EXTIN下降沿触发 |
> | SWIER | 0x10 | 32bit | **软件中断事件寄存器**，写1软件触发中断 |
> | PR | 0x14 | 32bit | **挂起寄存器**，触发后硬件置1，**软件写1清除** |
> 
> EXTI 是**边沿检测器**，检测引脚电平跳变并产生中断/事件请求。
> 
> > 关键区别：
> > - 中断：触发后执行 ISR（要配 NVIC）
> > - 事件：触发后唤醒 CPU/产生脉冲（不执行 ISR，不配 NVIC）
> 
> ---
> 
> **模块4：NVIC（嵌套向量中断控制器）**
> 
> | 寄存器 | 偏移 | 作用 |
> |--------|------|------|
> | ISER[0] | 0x100 | 中断使能（0~31），写1使能，写0无效 |
> | ISER[1] | 0x104 | 中断使能（32~63） |
> | ICER[0] | 0x180 | 中断禁用（0~31），写1禁用 |
> | ICER[1] | 0x184 | 中断禁用（32~63） |
> | ISPR[0] | 0x200 | 软件置挂起（手动触发中断）|
> | ICPR[0] | 0x280 | 清除挂起 |
> | IABR[0] | 0x300 | 读取中断是否正在执行 |
> | IP[0]~IP[59] | 0x400~0x43B | **优先级寄存器**，每个中断1字节，STM32用高4位 |
> | STIR | 0xF00 | 软件触发中断寄存器 |
> 
> NVIC 是**中断调度中心**，管理所有中断的开关、优先级、响应顺序。
> 
> ---
> 
> **不同中断类型的寄存器使用对比**
> 
> | 中断类型 | GPIO | AFIO | EXTI | NVIC | 清标志方式 |
> |---------|------|------|------|------|-----------|
> | **EXTI0 引脚中断** | ✅ CRL/CRH 配输入 | ✅ EXTICR[0] 选端口 | ✅ IMR/RTSR/FTSR 配触发 | ✅ ISER/IP 使能 | 软件写 PR=1 |
> | **EXTI9_5 共用中断** | ✅ 同上 | ✅ 同上 | ✅ 同上 | ✅ ISER[0] bit23 | 软件写 PR=1，按位区分 |
> | **USART1 RXNE** | ❌ 不用 | ❌ 不用 | ❌ 不用 | ✅ ISER[1] bit5 | 读 DR 自动清 |
> | **TIM2 更新中断** | ❌ 不用 | ❌ 不用 | ❌ 不用 | ✅ ISER[0] bit28 | 软件写 TIM2->SR=0 |
> | **DMA1 传输完成** | ❌ 不用 | ❌ 不用 | ❌ 不用 | ✅ ISER[0] bit11 | 软件写 DMA->IFCR |
> 
> ---
> 
> **外部中断完整寄存器操作链（EXTI0 为例）**
> 
> ```
> 1. GPIOA->CRL:  配置 PA0 为浮空/上拉/下拉输入
>        ↓
> 2. AFIO->EXTICR[0]:  bit3:0=0000，选择 PA0 连到 EXTI0
>        ↓
> 3. EXTI->FTSR:  bit0=1，下降沿触发
>    EXTI->IMR:    bit0=1，中断模式使能
>        ↓
> 4. NVIC->IP[6]:  设置优先级（EXTI0_IRQn=6）
>    NVIC->ISER[0]: bit6=1，使能中断通道
>        ↓
> 5. 按键按下，PA0 变低
>        ↓
> 6. EXTI 检测下降沿，EXTI->PR bit0=1
>        ↓
> 7. NVIC 收到请求，仲裁后响应
>        ↓
> 8. CPU 执行 EXTI0_IRQHandler()
>        ↓
> 9. 软件写 EXTI->PR = (1<<0)，清除挂起
> ```
> 
> ---
> 
> **内部外设中断寄存器操作链（USART1 为例）**
> 
> ```
> 10. GPIOA->CRH:  配置 PA9=AF_PP, PA10=INPUT（引脚功能）
>        ↓
> 11. USART1->BRR/CR1/CR2:  配置波特率、字长、停止位、使能收发
>        ↓
> 12. USART1->CR1:  bit5=1（RXNEIE=1），使能接收中断
>        ↓
> 13. NVIC->IP[37]:  设置优先级
>    NVIC->ISER[1]:  bit5=1（37-32=5），使能中断通道
>        ↓
> 14. 数据到达 PA10
>        ↓
> 15. USART1->SR bit5=1（RXNE=1），同时触发 NVIC
>        ↓
> 16. CPU 执行 USART1_IRQHandler()
>        ↓
> 17. 读 USART1->DR，自动清除 RXNE
> ```
> 
> > 注意：USART 中断**不经过 AFIO 和 EXTI**，直接由 NVIC 管理。
> 
> ---
> 
> **关键区别总结**
> 
> | | 外部中断（EXTI） | 内部外设中断（USART/TIM/DMA）|
> |--|-----------------|---------------------------|
> | 信号来源 | 芯片外部引脚 | 芯片内部模块 |
> | 需要 GPIO | ✅ 配成输入 | ✅ 配成复用（部分需要）|
> | 需要 AFIO | ✅ 选择引脚映射 | ❌ 不需要 |
> | 需要 EXTI | ✅ 边沿检测 + 中断/事件 | ❌ 不需要 |
> | 需要 NVIC | ✅ 必须 | ✅ 必须 |
> | 清标志 | 软件写 PR=1 | 各外设不同（读 DR / 写 SR 等）|
> 

## NVIC寄存器结构
![image.png](https://imgbed.paxseeker.xyz/file/1784280217083_image.png)
![image.png](https://imgbed.paxseeker.xyz/file/1784280217920_image.png)

## NVIC寄存器
```c
typedef struct
{
  __IOM uint32_t ISER[8U];               /*!< Offset: 0x000 (R/W)  Interrupt Set Enable Register */
        uint32_t RESERVED0[24U];
  __IOM uint32_t ICER[8U];               /*!< Offset: 0x080 (R/W)  Interrupt Clear Enable Register */
        uint32_t RSERVED1[24U];
  __IOM uint32_t ISPR[8U];               /*!< Offset: 0x100 (R/W)  Interrupt Set Pending Register */
        uint32_t RESERVED2[24U];
  __IOM uint32_t ICPR[8U];               /*!< Offset: 0x180 (R/W)  Interrupt Clear Pending Register */
        uint32_t RESERVED3[24U];
  __IOM uint32_t IABR[8U];               /*!< Offset: 0x200 (R/W)  Interrupt Active bit Register */
        uint32_t RESERVED4[56U];
  __IOM uint8_t  IP[240U];               /*!< Offset: 0x300 (R/W)  Interrupt Priority Register (8Bit wide) */
        uint32_t RESERVED5[644U];
  __OM  uint32_t STIR;                   /*!< Offset: 0xE00 ( /W)  Software Trigger Interrupt Register */
}  NVIC_Type;


```

>[!tip]
> **HAL 库 UART 中断方式初始化完整流程**
> 
> 涉及三层：用户代码 → HAL 封装层 → 寄存器操作。
> 
> ---
> 
> **第一步：用户调用 HAL_UART_Init**
> 
> ```c
> huart1.Instance = USART1;
> huart1.Init.BaudRate = 115200;
> // ... 其他参数 ...
> HAL_UART_Init(&huart1);
> ```
> 
> 这一步只填充了句柄结构体，还没碰寄存器。
> 
> ---
> 
> **第二步：HAL_UART_Init 内部流程**
> 
> ```
> HAL_UART_Init()
>     │
>     ├── 检查参数合法性
>     │
>     ├── 如果是第一次初始化，调用 HAL_UART_MspInit()
>     │       │
>     │       └── 用户或 CubeMX 生成的 MSP 函数：
>     │           │
>     │           ├── 使能 GPIOA 时钟 (RCC->APB2ENR bit2)
>     │           ├── 配置 PA9=AF_PP, PA10=INPUT (GPIOA->CRH)
>     │           ├── 使能 USART1 时钟 (RCC->APB2ENR bit14)
>     │           ├── 配置 NVIC：
>     │           │   NVIC->ISER[1] |= (1 << 5)   // USART1_IRQn=37
>     │           │   NVIC->IP[37] = 优先级值
>     │           └── 可选：配置 DMA
>     │
>     ├── 禁用 USART (CR1->UE = 0)
>     │
>     ├── 计算并写入 BRR
>     │   BRR = 72000000 / (16 × 115200) = 0x0271
>     │
>     ├── 配置 CR2 (停止位)
>     │   CR2[13:12] = 00 (1 停止位)
>     │
>     ├── 配置 CR3 (流控)
>     │   CR3[9:8] = 00 (无流控)
>     │
>     ├── 配置 CR1 (字长/校验/模式)
>     │   CR1[12] = 0 (8 数据位)
>     │   CR1[10] = 0 (无校验)
>     │   CR1[3] = 1 (TE 发送使能)
>     │   CR1[2] = 1 (RE 接收使能)
>     │
>     └── 使能 USART (CR1[13] = UE = 1)
> ```
> 
> > 注意：此时 CR1[5] RXNEIE = 0，**接收中断还没开**！
> 
> ---
> 
> **第三步：用户调用 HAL_UART_Receive_IT**
> 
> ```c
> HAL_UART_Receive_IT(&huart1, buf, 1);
> ```
> 
> 这一步才**真正使能中断**：
> 
> ```
> HAL_UART_Receive_IT()
>     │
>     ├── 保存接收参数到句柄
>     │   huart->pRxBuffPtr = buf
>     │   huart->RxXferSize = 1
>     │   huart->RxXferCount = 1
>     │
>     ├── 设置状态为 BUSY_RX
>     │
>     └── 设置 CR1[5] = RXNEIE = 1  ← 关键！
> ```
> 
> 此时硬件准备就绪，RXNE=1 时会触发 NVIC 中断。
> 
> ---
> 
> **第四步：数据到达，中断触发**
> 
> ```
> 外部数据 → RX 引脚 → 移位寄存器 → DR
>                              ↓
>                         RXNE = 1
>                              ↓
>                         RXNEIE = 1 ? ──→ 触发 NVIC 中断
>                              ↓
>                         NVIC 仲裁 ──→ 执行 USART1_IRQHandler
> ```
> 
> ---
> 
> **第五步：中断服务函数执行链**
> 
> ```
> USART1_IRQHandler()          // 向量表入口，用户编写
>     │
>     └── HAL_UART_IRQHandler(&huart1)   // HAL 统一处理
>             │
>             ├── 读 SR 寄存器
>             │
>             ├── 判断中断源：
>             │   │
>             │   ├── RXNE=1 ──→ 读 DR，数据存入 pRxBuffPtr
>             │   │              RxXferCount--，指针++
>             │   │              如果 Count=0：
>             │   │                  CR1[5]=0 (关闭 RXNEIE)
>             │   │                  状态 = READY
>             │   │                  调用 HAL_UART_RxCpltCallback()
>             │   │
>             │   ├── TXE=1 ──→ 写 DR 发送数据
>             │   │
>             │   └── 错误标志 ──→ 调用 ErrorCallback
>             │
>             └── 清除中断标志（读 SR 再读/写 DR）
> ```
> 
> ---
> 
> **第六步：回调函数**
> 
> ```
> HAL_UART_RxCpltCallback()    // 用户重写
>     │
>     └── 用户处理数据
>         └── 必须再次调用 HAL_UART_Receive_IT() 才能继续接收！
> ```
> 
> > 单次触发设计：HAL 收完指定长度后自动关闭 RXNEIE，回调里必须重新订阅。
> 
> ---
> 
> **涉及的关键寄存器汇总**
> 
> | 寄存器 | 位 | 作用 | 谁操作 |
> |--------|---|------|--------|
> | RCC->APB2ENR | bit14 | USART1 时钟 | MSP |
> | RCC->APB2ENR | bit2 | GPIOA 时钟 | MSP |
> | GPIOA->CRH | bit11:4 | PA9/PA10 模式 | MSP |
> | NVIC->ISER[1] | bit5 | USART1 中断使能 | MSP |
> | NVIC->IP[37] | bit7:4 | 优先级 | MSP |
> | USART1->BRR | bit15:0 | 波特率分频 | HAL_UART_Init |
> | USART1->CR2 | bit13:12 | 停止位 | HAL_UART_Init |
> | USART1->CR3 | bit9:8 | 流控 | HAL_UART_Init |
> | USART1->CR1 | bit13 | UE 使能 | HAL_UART_Init |
> | USART1->CR1 | bit3 | TE 发送使能 | HAL_UART_Init |
> | USART1->CR1 | bit2 | RE 接收使能 | HAL_UART_Init |
> | **USART1->CR1** | **bit5** | **RXNEIE 接收中断** | **HAL_UART_Receive_IT** |
> | USART1->SR | bit5 | RXNE 接收标志 | 硬件置位 |
> | USART1->DR | bit8:0 | 数据寄存器 | 硬件/软件读写 |
> 
> ---
> 
> **容易遗漏的点**
> 
> | 遗漏项 | 后果 |
> |--------|------|
> | 没写 MSP 回调或 CubeMX 没生成 | NVIC 没使能，中断不来 |
> | 只调 HAL_UART_Init 没调 Receive_IT | RXNEIE=0，中断不触发 |
> | 回调里没重新 Receive_IT | 只收一次，之后没中断 |
> | 中断服务函数名写错 | 进默认死循环，不执行 |
> | 波特率计算错 | 收到乱码或收不到 |
> 

## CUBEMX生成代码(UART中断方式接收信息)
- hal-msp.c
```c
void HAL_UART_MspInit(UART_HandleTypeDef* huart)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};
  if(huart->Instance==USART1)
  {
    /* USER CODE BEGIN USART1_MspInit 0 */

    /* USER CODE END USART1_MspInit 0 */
    /* Peripheral clock enable */
    __HAL_RCC_USART1_CLK_ENABLE();

    __HAL_RCC_GPIOA_CLK_ENABLE();
    /**USART1 GPIO Configuration
    PA9     ------> USART1_TX
    PA10     ------> USART1_RX
    */
    GPIO_InitStruct.Pin = GPIO_PIN_9;
    GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

    GPIO_InitStruct.Pin = GPIO_PIN_10;
    GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
    GPIO_InitStruct.Pull = GPIO_NOPULL;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

    /* USART1 interrupt Init */
    HAL_NVIC_SetPriority(USART1_IRQn, 0, 0);
    HAL_NVIC_EnableIRQ(USART1_IRQn);
    /* USER CODE BEGIN USART1_MspInit 1 */

    /* USER CODE END USART1_MspInit 1 */

  }

}

/**
  * @brief UART MSP De-Initialization
  * This function freeze the hardware resources used in this example
  * @param huart: UART handle pointer
  * @retval None
  */
void HAL_UART_MspDeInit(UART_HandleTypeDef* huart)
{
  if(huart->Instance==USART1)
  {
    /* USER CODE BEGIN USART1_MspDeInit 0 */

    /* USER CODE END USART1_MspDeInit 0 */
    /* Peripheral clock disable */
    __HAL_RCC_USART1_CLK_DISABLE();

    /**USART1 GPIO Configuration
    PA9     ------> USART1_TX
    PA10     ------> USART1_RX
    */
    HAL_GPIO_DeInit(GPIOA, GPIO_PIN_9|GPIO_PIN_10);

    /* USART1 interrupt DeInit */
    HAL_NVIC_DisableIRQ(USART1_IRQn);
    /* USER CODE BEGIN USART1_MspDeInit 1 */

    /* USER CODE END USART1_MspDeInit 1 */
  }

}
```

- main.c
```c
/* USER CODE BEGIN Header */
/**
 ******************************************************************************
 * @file           : main.c
 * @brief          : Main program body
 ******************************************************************************
 * @attention
 *
 * Copyright (c) 2026 STMicroelectronics.
 * All rights reserved.
 *
 * This software is licensed under terms that can be found in the LICENSE file
 * in the root directory of this software component.
 * If no LICENSE file comes with this software, it is provided AS-IS.
 *
 ******************************************************************************
 */
/* USER CODE END Header */
/* Includes ------------------------------------------------------------------*/
#include "main.h"
#include "stm32f103xb.h"
#include "stm32f1xx_hal_gpio.h"

/* Private includes ----------------------------------------------------------*/
/* USER CODE BEGIN Includes */

/* USER CODE END Includes */

/* Private typedef -----------------------------------------------------------*/
/* USER CODE BEGIN PTD */

/* USER CODE END PTD */

/* Private define ------------------------------------------------------------*/
/* USER CODE BEGIN PD */

/* USER CODE END PD */

/* Private macro -------------------------------------------------------------*/
/* USER CODE BEGIN PM */

/* USER CODE END PM */

/* Private variables ---------------------------------------------------------*/
UART_HandleTypeDef huart1;

/* USER CODE BEGIN PV */

/* USER CODE END PV */

/* Private function prototypes -----------------------------------------------*/
void SystemClock_Config(void);
static void MX_GPIO_Init(void);
static void MX_USART1_UART_Init(void);
/* USER CODE BEGIN PFP */

/* USER CODE END PFP */

/* Private user code ---------------------------------------------------------*/
/* USER CODE BEGIN 0 */
// 中断服务程序
char received_data[1];

// 接收完成回调函数
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart) {
    if (huart == &huart1) {
        if (received_data[0] == '0') {
            HAL_UART_Transmit(&huart1, "Hello,0\r\n", 9, 50);
        } else {
            HAL_UART_Transmit(&huart1, "Hello,1\r\n", 9, 50);
        }
        HAL_UART_Receive_IT(&huart1, (uint8_t *)&received_data, 1);
    }
}

/* USER CODE END 0 */

/**
 * @brief  The application entry point.
 * @retval int
 */
int main(void) {

    /* USER CODE BEGIN 1 */

    /* USER CODE END 1 */

    /* MCU
     * Configuration--------------------------------------------------------*/

    /* Reset of all peripherals, Initializes the Flash interface and the
     * Systick. */
    HAL_Init();

    /* USER CODE BEGIN Init */

    /* USER CODE END Init */

    /* Configure the system clock */
    SystemClock_Config();

    /* USER CODE BEGIN SysInit */

    /* USER CODE END SysInit */

    /* Initialize all configured peripherals */
    MX_GPIO_Init();
    MX_USART1_UART_Init();
    /* USER CODE BEGIN 2 */
    HAL_UART_Receive_IT(&huart1, (uint8_t *)&received_data, 1);
    /* USER CODE END 2 */

    /* Infinite loop */
    /* USER CODE BEGIN WHILE */
    while (1) {
        /* USER CODE END WHILE */

        /* USER CODE BEGIN 3 */
        HAL_Delay(500);
        // HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_13);
        HAL_UART_Transmit(&huart1, "Hello,2\r\n", 9, 50);
        if (received_data[0] == '0') {
            HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_SET);
        } else {
            HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_RESET);
        }
    }
    /* USER CODE END 3 */
}

/**
 * @brief System Clock Configuration
 * @retval None
 */
void SystemClock_Config(void) {
    RCC_OscInitTypeDef RCC_OscInitStruct = {0};
    RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};

    /** Initializes the RCC Oscillators according to the specified parameters
     * in the RCC_OscInitTypeDef structure.
     */
    RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSE;
    RCC_OscInitStruct.HSEState = RCC_HSE_ON;
    RCC_OscInitStruct.HSEPredivValue = RCC_HSE_PREDIV_DIV1;
    RCC_OscInitStruct.HSIState = RCC_HSI_ON;
    RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
    RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSE;
    RCC_OscInitStruct.PLL.PLLMUL = RCC_PLL_MUL9;
    if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK) {
        Error_Handler();
    }

    /** Initializes the CPU, AHB and APB buses clocks
     */
    RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK | RCC_CLOCKTYPE_SYSCLK |
                                  RCC_CLOCKTYPE_PCLK1 | RCC_CLOCKTYPE_PCLK2;
    RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
    RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;
    RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV2;
    RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV1;

    if (HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_2) != HAL_OK) {
        Error_Handler();
    }
}

/**
 * @brief USART1 Initialization Function
 * @param None
 * @retval None
 */
static void MX_USART1_UART_Init(void) {

    /* USER CODE BEGIN USART1_Init 0 */

    /* USER CODE END USART1_Init 0 */

    /* USER CODE BEGIN USART1_Init 1 */

    /* USER CODE END USART1_Init 1 */
    huart1.Instance = USART1;
    huart1.Init.BaudRate = 115200;
    huart1.Init.WordLength = UART_WORDLENGTH_8B;
    huart1.Init.StopBits = UART_STOPBITS_1;
    huart1.Init.Parity = UART_PARITY_NONE;
    huart1.Init.Mode = UART_MODE_TX_RX;
    huart1.Init.HwFlowCtl = UART_HWCONTROL_NONE;
    huart1.Init.OverSampling = UART_OVERSAMPLING_16;
    if (HAL_UART_Init(&huart1) != HAL_OK) {
        Error_Handler();
    }
    /* USER CODE BEGIN USART1_Init 2 */

    /* USER CODE END USART1_Init 2 */
}

/**
 * @brief GPIO Initialization Function
 * @param None
 * @retval None
 */
static void MX_GPIO_Init(void) {
    GPIO_InitTypeDef GPIO_InitStruct = {0};
    /* USER CODE BEGIN MX_GPIO_Init_1 */

    /* USER CODE END MX_GPIO_Init_1 */

    /* GPIO Ports Clock Enable */
    __HAL_RCC_GPIOC_CLK_ENABLE();
    __HAL_RCC_GPIOD_CLK_ENABLE();
    __HAL_RCC_GPIOB_CLK_ENABLE();
    __HAL_RCC_GPIOA_CLK_ENABLE();

    /*Configure GPIO pin Output Level */
    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_RESET);

    /*Configure GPIO pin : PC13 */
    GPIO_InitStruct.Pin = GPIO_PIN_13;
    GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
    GPIO_InitStruct.Pull = GPIO_NOPULL;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
    HAL_GPIO_Init(GPIOC, &GPIO_InitStruct);

    /*Configure GPIO pin : PB12 */
    GPIO_InitStruct.Pin = GPIO_PIN_12;
    GPIO_InitStruct.Mode = GPIO_MODE_IT_RISING;
    GPIO_InitStruct.Pull = GPIO_NOPULL;
    HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);

    /* EXTI interrupt init*/
    HAL_NVIC_SetPriority(EXTI15_10_IRQn, 0, 0);
    HAL_NVIC_EnableIRQ(EXTI15_10_IRQn);

    /* USER CODE BEGIN MX_GPIO_Init_2 */

    /* USER CODE END MX_GPIO_Init_2 */
}

/* USER CODE BEGIN 4 */

/* USER CODE END 4 */

/**
 * @brief  This function is executed in case of error occurrence.
 * @retval None
 */
void Error_Handler(void) {
    /* USER CODE BEGIN Error_Handler_Debug */
    /* User can add his own implementation to report the HAL error return state
     */
    __disable_irq();
    while (1) {
    }
    /* USER CODE END Error_Handler_Debug */
}
#ifdef USE_FULL_ASSERT
/**
 * @brief  Reports the name of the source file and the source line number
 *         where the assert_param error has occurred.
 * @param  file: pointer to the source file name
 * @param  line: assert_param error line source number
 * @retval None
 */
void assert_failed(uint8_t *file, uint32_t line) {
    /* USER CODE BEGIN 6 */
    /* User can add his own implementation to report the file name and line
       number, ex: printf("Wrong parameters value: file %s on line %d\r\n",
       file, line) */
    /* USER CODE END 6 */
}
#endif /* USE_FULL_ASSERT */

```