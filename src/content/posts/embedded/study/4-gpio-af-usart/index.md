---
title: GPIO复用--USART串口通信
category: 嵌入式
tags:
  - 嵌入式
draft: false
description: GPIO复用--USART串口通信
published: 2026-05-23
image: https://imgbed.paxseeker.xyz/file/1784268182329_cloud-sky.png
---
# GPIO复用--USART串口通信

>[!tip]
> **复用（Alternate Function）解决"是什么"的问题**
> 
> 引脚内部并联了 GPIO、USART、TIM 等多个模块。默认由 GPIO 控制，当需要发串口数据或输出 PWM 时，必须通过 `MODE = 10` 把控制权**切换给外设模块**。没有复用，USART 数据到不了引脚，只能在芯片内部打转。**只要用外设功能就必须复用**，这是硬性要求。
> 
> ---
> 
> **重映射（Remap）解决"在哪里"的问题**
> 
> 同一个 USART1 模块，TX/RX 信号可以走 PA9/PA10（默认），也可以走 PB6/PB7（重映射）。当默认引脚被占用，或 PCB 布线更方便时，通过设置 `AFIO_MAPR` 把信号**搬到另一组引脚**。重映射是**可选的**，默认引脚够用就不需要。
> 

## USART结构图
![image.png](https://imgbed.paxseeker.xyz/file/1784265708575_image.png)
## USART寄存器地址映象(USART1  - 0x40013BFF)
![image.png](https://imgbed.paxseeker.xyz/file/1784114839983_image.png)

>[!tip]
> **USART 完整配置流程**
> 
> 1. **使能时钟**：RCC->APB2ENR |= USART1EN + IOPAEN
> 2. **配置 GPIO**：PA9=AF_PP(复用推挽), PA10=INPUT(浮空输入)
> 3. **配置波特率**：USART1->BRR = 分频值
> 4. **配置帧格式**：CR1(字长/校验/收发使能) + CR2(停止位)
> 5. **使能 USART**：CR1->UE = 1
> 6. **可选使能中断**：CR1->RXNEIE = 1, NVIC_ISER 置位
> 
> ---
> 
> **数据帧格式：哪里开始，哪里停止**
> 
> 空闲时 TX 保持高电平。发送时：
> 
> ```
> 空闲    起始位   D0   D1   D2   D3   D4   D5   D6   D7   停止位   空闲
>  ─┐    ┌──┐  ┌┐  ┌┐  ┌┐  ┌┐  ┌┐  ┌┐  ┌┐  ┌┐  ┌──┐    ┌─
>   └────┘  └──┘└──┘└──┘└──┘└──┘└──┘└──┘└──┘└──┘    └────┘
>    1→0   低位先发                    高位后发    回到高电平
> ```
> 
> | 位 | 说明 |
> |---|------|
> | **起始位** | 1 位，高→低跳变，通知接收方"数据来了" |
> | **数据位** | 8 或 9 位，低位先发（LSB first）|
> | **校验位** | 可选，紧跟数据位之后 |
> | **停止位** | 1/1.5/2 位，高电平，标志帧结束 |
> 
> ---
> 
> **接收方怎么识别第一个数据**
> 
> 硬件自动完成，无需软件干预：
> 
> 1. **检测下降沿**：RX 引脚从高变低，检测到起始位
> 2. **重新同步**：内部采样时钟复位，以起始位下降沿为基准
> 3. **中间采样**：每个位周期的中间点采样（避开边沿抖动）
> 4. **存入移位寄存器**：逐位右移，先来的位进最低位
> 5. **帧结束判断**：收到停止位高电平，移位寄存器内容转存到 DR
> 6. **置 RXNE 标志**：SR->RXNE = 1，如果 RXNEIE=1 则触发中断
> 
> 软件只需读 DR，硬件已把完整字节准备好。读 DR 会自动清除 RXNE，准备接收下一帧。
> 
> ---
> 
> **波特率的作用：约定采样时刻**
> 
> 双方波特率必须一致，否则采样点错位，数据错误。
> 
> ```
> 发送方                    接收方
> 位周期 = 1/115200s       位周期 = 1/115200s
>      │                        │
>      ▼                        ▼
>   起始位下降沿 ─────────→  检测下降沿，以此为 T0
>      │                        │
>   T0 + 1.5位周期         T0 + 1.5位周期 ← 采样 D0
>   T0 + 2.5位周期         T0 + 2.5位周期 ← 采样 D1
>      ...                      ...
> ```
> 
> 过采样：STM32 每个位采样 16 次，取中间 3 次多数表决，抗干扰。

> [!warning] 双方的配置必须一样才能正常通信
> 一般串口工具默认配置都是8N1--8位数据为，没有校验位，以及1个停止位

```c
#define USART1_BASE     0x40013800
#define USART2_BASE     0x40004400
#define USART3_BASE     0x40004800

typedef struct {
    __IO uint32_t SR;       // 0x00 状态寄存器
    __IO uint32_t DR;       // 0x04 数据寄存器
    __IO uint32_t BRR;      // 0x08 波特率寄存器
    __IO uint32_t CR1;      // 0x0C 控制寄存器1
    __IO uint32_t CR2;      // 0x10 控制寄存器2
    __IO uint32_t CR3;      // 0x14 控制寄存器3
    __IO uint32_t GTPR;     // 0x18 保护时间和预分频寄存器
} USART_TypeDef;

#define USART1  ((USART_TypeDef*)USART1_BASE)
#define USART2  ((USART_TypeDef*)USART2_BASE)
#define USART3  ((USART_TypeDef*)USART3_BASE)


#define USART_BAUDRATE_9600     9600
#define USART_BAUDRATE_115200   115200

#define USART_WORDLENGTH_8B     0x00000000u  /*!< 8 数据位 */
#define USART_WORDLENGTH_9B     0x00001000u  /*!< 9 数据位 (CR1 bit12) */

#define USART_STOPBITS_1        0x00000000u  /*!< 1 停止位 */
#define USART_STOPBITS_2        0x00002000u  /*!< 2 停止位 (CR2 bit13:12 = 10) */

#define USART_PARITY_NONE       0x00000000u  /*!< 无校验 */
#define USART_PARITY_EVEN       0x00000400u  /*!< 偶校验 (CR1 bit10 PCE=1, PS=0) */
#define USART_PARITY_ODD        0x00000600u  /*!< 奇校验 (CR1 bit10 PCE=1, PS=1) */

#define USART_MODE_RX           0x00000004u  /*!< 接收使能 (RE=1) CR1 bit2 */
#define USART_MODE_TX           0x00000008u  /*!< 发送使能 (TE=1) CR1 bit3 */
#define USART_MODE_TX_RX        (USART_MODE_TX | USART_MODE_RX)

#define USART_HWCONTROL_NONE    0x00000000u  /*!< 无硬件流控 */
#define USART_HWCONTROL_RTS     0x00000100u  /*!< RTS 使能 (CR3 bit8) */
#define USART_HWCONTROL_CTS     0x00000200u  /*!< CTS 使能 (CR3 bit9) */
#define USART_HWCONTROL_RTS_CTS 0x00000300u  /*!< RTS 和 CTS 都使能 */

#define USART_FLAG_TXE          0x00000080u  /*!< 发送数据寄存器空 (SR bit7) */
#define USART_FLAG_RXNE         0x00000020u  /*!< 接收数据寄存器非空 (SR bit5) */
#define USART_FLAG_TC           0x00000040u  /*!< 发送完成 (SR bit6) */
#define USART_FLAG_IDLE         0x00000010u  /*!< 空闲总线 (SR bit4) */
#define USART_FLAG_ORE          0x00000008u  /*!< 溢出错误 (SR bit3) */
#define USART_FLAG_NE           0x00000004u  /*!< 噪声错误 (SR bit2) */
#define USART_FLAG_FE           0x00000002u  /*!< 帧错误 (SR bit1) */
#define USART_FLAG_PE           0x00000001u  /*!< 校验错误 (SR bit0) */


typedef struct {
    uint32_t BaudRate;          /*!< 波特率 */
    uint32_t WordLength;        /*!< 字长 */
    uint32_t StopBits;          /*!< 停止位 */
    uint32_t Parity;            /*!< 校验 */
    uint32_t Mode;              /*!< 收发模式 */
    uint32_t HwFlowCtl;         /*!< 硬件流控 */
} USART_InitTypeDef;


/* ==================== USART 配置函数 ==================== */

void USART_Init(USART_TypeDef* USARTx, USART_InitTypeDef* USART_InitStruct)
{
    uint32_t apbclock;
    uint32_t mantissa;
    uint32_t fraction;
    uint32_t tmpreg;

    // 判断 USART 所在总线，获取时钟频率
    if (USARTx == USART1)
    {
        apbclock = 72000000;  // USART1 在 APB2，最高 72MHz
    }
    else
    {
        apbclock = 36000000;  // USART2/3 在 APB1，最高 36MHz
    }

    // 禁用 USART（配置前必须先关）
    USARTx->CR1 &= ~(0x200C);  // 清除 UE/TE/RE 位

    // ----- 计算波特率 (BRR = USARTDIV) -----
    // BRR = PCLK / (16 * BaudRate)
    // 整数部分 = mantissa, 小数部分 = fraction (4bit)
    tmpreg = ((25 * apbclock) / (4 * USART_InitStruct->BaudRate));
    mantissa = (tmpreg / 100) << 4;
    fraction = (((tmpreg - (mantissa >> 4) * 100) * 16 + 50) / 100) & 0x0F;
    USARTx->BRR = mantissa | fraction;

    // ----- 配置 CR2 (停止位) -----
    USARTx->CR2 &= ~0x3000;  // 清除 STOP[1:0]
    USARTx->CR2 |= (USART_InitStruct->StopBits & 0x3000);

    // ----- 配置 CR3 (硬件流控) -----
    USARTx->CR3 &= ~0x0300;  // 清除 RTSE/CTSE
    USARTx->CR3 |= (USART_InitStruct->HwFlowCtl & 0x0300);

    // ----- 配置 CR1 (字长/校验/模式) -----
    tmpreg = USARTx->CR1;
    tmpreg &= ~0x3F0C;  // 清除 M/PCE/PS/TE/RE/UE 位
    
    tmpreg |= (USART_InitStruct->WordLength & 0x1000);   // M
    tmpreg |= (USART_InitStruct->Parity & 0x0600);       // PCE/PS
    tmpreg |= (USART_InitStruct->Mode & 0x000C);          // TE/RE
    
    USARTx->CR1 = tmpreg;

    // 使能 USART
    USARTx->CR1 |= 0x2000;  // UE = 1
}


void USART_SendByte(USART_TypeDef* USARTx, uint8_t data)
{
    // 等待发送数据寄存器空 (TXE=1)
    while (!(USARTx->SR & USART_FLAG_TXE));
    
    USARTx->DR = data;  // 写入数据，硬件自动清除 TXE
}


void USART_SendString(USART_TypeDef* USARTx, const char* str)
{
    while (*str)
    {
        USART_SendByte(USARTx, *str++);
    }
}


void USART_SendData(USART_TypeDef* USARTx, uint8_t* buf, uint16_t len)
{
    while (len--)
    {
        USART_SendByte(USARTx, *buf++);
    }
}


uint8_t USART_ReceiveByte(USART_TypeDef* USARTx)
{
    // 等待接收数据寄存器非空 (RXNE=1)
    while (!(USARTx->SR & USART_FLAG_RXNE));
    
    return (uint8_t)(USARTx->DR & 0xFF);
}


uint8_t USART_DataAvailable(USART_TypeDef* USARTx)
{
    return (USARTx->SR & USART_FLAG_RXNE) ? 1 : 0;
}


uint8_t USART_ReadByte(USART_TypeDef* USARTx, uint8_t* data)
{
    if (USARTx->SR & USART_FLAG_RXNE)
    {
        *data = (uint8_t)(USARTx->DR & 0xFF);
        return 0;
    }
    return 1;
}


void USART_WaitTxComplete(USART_TypeDef* USARTx)
{
    // 等待发送完成标志 TC=1
    while (!(USARTx->SR & USART_FLAG_TC));
}

```

```c
int main(void)
{
    /*
     * 使能 GPIOC 时钟
     * RCC_APB2ENR: APB2 外设时钟使能寄存器
     * 位4: IOPCEN - GPIOC 时钟使能
     * |= (1 << 4): 设置第4位为1，使能 GPIOC 时钟
     * 语法: 位或操作，只修改指定位，不影响其他位
     */

    SystemClk_Init();

    // 使能 GPIOC 时钟
    RCC->APB2ENR |= (1 << 4);

    // 使能 GPIOA 和 USART1 时钟
    RCC->APB2ENR |= (1 << 2);    // IOPAEN
    RCC->APB2ENR |= (1 << 14);   // USART1EN

    // 配置 PC13 为推挽输出
    GPIO_InitTypeDef Init_Struct;
    Init_Struct.Mode = GPIO_MODE_OUTPUT_PP;
    Init_Struct.Pin = GPIO_Pin_13;
    Init_Struct.Pull = GPIO_NOPULL;
    Init_Struct.Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOC, &Init_Struct);

    // 配置 PA9(TX) 为复用推挽输出
    Init_Struct.Pin = GPIO_Pin_9;
    Init_Struct.Mode = GPIO_MODE_AF_PP;
    Init_Struct.Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOA, &Init_Struct);

    // 配置 PA10(RX) 为浮空输入
    Init_Struct.Pin = GPIO_Pin_10;
    Init_Struct.Mode = GPIO_MODE_INPUT;
    Init_Struct.Pull = GPIO_NOPULL;
    GPIO_Init(GPIOA, &Init_Struct);

    // 初始化 USART1
    USART_InitTypeDef USART_InitStruct;
    USART_InitStruct.BaudRate = USART_BAUDRATE_115200;
    USART_InitStruct.WordLength = USART_WORDLENGTH_8B;
    USART_InitStruct.StopBits = USART_STOPBITS_1;
    USART_InitStruct.Parity = USART_PARITY_NONE;
    USART_InitStruct.Mode = USART_MODE_TX_RX;
    USART_InitStruct.HwFlowCtl = USART_HWCONTROL_NONE;
    USART_Init(USART1, &USART_InitStruct);

    // 发送启动消息
    USART_SendString(USART1, "System Start!\r\n");

    uint8_t rx_data;
    uint32_t led_toggle_count = 0;

    while (1)
    {
        // 回显接收到的数据
        if (USART_ReadByte(USART1, &rx_data) == 0)
        {
            USART_SendByte(USART1, rx_data);  // 回发

            // 收到 '1' 点亮 LED，收到 '0' 熄灭 LED
            if (rx_data == '1')
            {
                GPIO_ResetPin(GPIOC, GPIO_Pin_13);
                USART_SendString(USART1, "LED ON\r\n");
            }
            else if (rx_data == '0')
            {
                GPIO_SetPin(GPIOC, GPIO_Pin_13);
                USART_SendString(USART1, "LED OFF\r\n");
            }
        }

        // LED 闪烁
        //GPIO_TogglePin(GPIOC, GPIO_Pin_13);
        sleep(500);

        // 每隔一段时间发送状态
        led_toggle_count++;
        if (led_toggle_count >= 4)
        {
            led_toggle_count = 0;
            USART_SendString(USART1, "LED toggled\r\n");
        }
    }
}
```

>[!tip]
> **问题：接收方处理慢，数据溢出丢失**
> 
> 发送方只管发，接收方 DR 只有 1 字节深度，不及时读就会 ORE（溢出错误）。
> 
> ---
> 
> **解决方法一：增大软件缓冲区（最常用）**
> 
> 中断里把数据存入环形缓冲区，主循环慢慢处理。
> 
> ```c
> #define RX_BUF_SIZE  256   // 比 1 字节 DR 大得多
> volatile uint8_t rx_buf[RX_BUF_SIZE];
> volatile uint16_t rx_head = 0;
> volatile uint16_t rx_tail = 0;
> 
> void USART1_IRQHandler(void)
> {
>     if (USART1->SR & USART_FLAG_RXNE)
>     {
>         uint16_t next = (rx_head + 1) % RX_BUF_SIZE;
>         if (next != rx_tail)           // 满则丢弃新数据
>             rx_buf[rx_head] = USART1->DR;
>         else
>             (void)USART1->DR;          // 满了也要读，清 RXNE
>         rx_head = next;
>     }
> }
> ```
> 
> 主循环用 `USART_GetChar()` 非阻塞读取，不丢中断级数据。
> 
> ---
> 
> **解决方法二：硬件流控 RTS/CTS**
> 
> 接收方缓冲区满时，RTS 输出高电平，发送方自动暂停。
> 
> 
> 需要额外两根线，纯硬件自动完成，无软件开销。
> 
> ---
> 
> **解决方法三：软件流控 XON/XOFF**
> 
> 没额外引脚时，发特殊字符控制：
> 
> | 字符 | ASCII | 作用 |
> |------|-------|------|
> | XOFF | 0x13 (DC3) | 告诉对方暂停发送 |
> | XON  | 0x11 (DC2) | 告诉对方继续发送 |
> 
> 接收方软件判断缓冲区水位，主动发 XON/XOFF。
> 缺点：增加带宽开销，且数据流中不能出现 0x11/0x13。
> 
> ---
> 
> **解决方法四：降低发送速率**
> 
> 降低波特率，或在每帧之间加延时：
> 
> ```c
> USART_SendByte(USART1, data);
> for (volatile int i = 0; i < 1000; i++);  // 软件延时
> ```
> 
> 简单粗暴，但效率低，浪费 CPU。
> 
> ---
> 

