---
title: UART不定长数据接收
aliases:
  - UART IDLE DMA 接收
  - 串口 DMA 循环缓冲区
tags:
  - STM32
  - UART
  - DMA
  - 串口
  - 中断
published: 2026-06-04
updated: 2026-06-04
draft: false
description: UART不定长数据接收，使用IDLE中断和DMA
category: 嵌入式
---

# UART 不定长数据接收（DMA + IDLE 中断）

## 架构

```
                    DMA1_Channel5 (Circular)
USART1 RX ──► USART1->DR ────────────────► dma_rx_buffer[256]
                                              │
                                    IDLE IRQ  │
                                    ──────►  frame_ready = 1
                                              │
                                    uart_read_frame() 读取
```

- DMA 负责把数据从 USART DR 搬到环形缓冲区，CPU 零参与
- IDLE 中断只通知 CPU "一帧结束"，不搬运数据
- `uart_read_frame` 在任务中轮询 `frame_ready` 标志，提取数据

---

## 头文件 `Inc/uart.h`

```c
#ifndef __UART_H
#define __UART_H

#include "main.h"
#include "stm32f10x_rcc.h"
#include "stm32f10x_usart.h"
#include <stdint.h>

typedef enum {
  SEND_OK = 0,
  SEND_ERR = -1,
  SEND_TIMEOUT = -2
} uart_status;

RES uart_init(void);

uart_status uart_send(USART_TypeDef* USARTx, uint8_t* data, uint16_t len, uint32_t timeout);

uint16_t uart_read_frame(USART_TypeDef* USARTx, uint8_t* buf, uint16_t max_len);

#endif
```

---

## 实现 `Src/uart.c`

### 全局变量

```c
#define RX_BUFFER_SIZE 256

static uint8_t  dma_rx_buffer[RX_BUFFER_SIZE];   // DMA 环形缓冲区
static volatile uint16_t last_read_index;          // 上次读取位置
static volatile uint8_t  frame_ready;              // IDLE 中断置位，表示一帧完成
```

### uart_init — 初始化

```c
RES uart_init() {
  // === 1. 时钟 ===
  RCC_APB2PeriphClockCmd(RCC_APB2Periph_USART1 | RCC_APB2Periph_GPIOA, ENABLE);
  RCC_AHBPeriphClockCmd(RCC_AHBPeriph_DMA1, ENABLE);

  // === 2. GPIO ===
  GPIO_InitTypeDef gpio;
  gpio.GPIO_Mode  = GPIO_Mode_AF_PP;
  gpio.GPIO_Speed = GPIO_Speed_50MHz;
  gpio.GPIO_Pin   = GPIO_Pin_9;          // TX
  GPIO_Init(GPIOA, &gpio);

  gpio.GPIO_Pin   = GPIO_Pin_10;         // RX
  gpio.GPIO_Mode  = GPIO_Mode_IN_FLOATING;
  GPIO_Init(GPIOA, &gpio);

  // === 3. DMA 循环模式 (USART1_RX → DMA1_Channel5) ===
  DMA_DeInit(DMA1_Channel5);

  DMA_InitTypeDef dma;
  dma.DMA_PeripheralBaseAddr = (uint32_t)&USART1->DR;  // 源: USART 数据寄存器
  dma.DMA_MemoryBaseAddr     = (uint32_t)dma_rx_buffer; // 目的: 内存缓冲区
  dma.DMA_DIR                = DMA_DIR_PeripheralSRC;   // 外设 → 内存
  dma.DMA_BufferSize         = RX_BUFFER_SIZE;          // 传输次数
  dma.DMA_PeripheralInc      = DMA_PeripheralInc_Disable;  // 外设地址不递增
  dma.DMA_MemoryInc          = DMA_MemoryInc_Enable;       // 内存地址递增
  dma.DMA_PeripheralDataSize = DMA_PeripheralDataSize_Byte;
  dma.DMA_MemoryDataSize     = DMA_MemoryDataSize_Byte;
  dma.DMA_Mode               = DMA_Mode_Circular;          // 循环模式
  dma.DMA_Priority           = DMA_Priority_High;
  dma.DMA_M2M                = DMA_M2M_Disable;
  DMA_Init(DMA1_Channel5, &dma);
  DMA_Cmd(DMA1_Channel5, ENABLE);        // 开启 DMA

  // === 4. USART 参数 ===
  USART_Cmd(USART1, DISABLE);            // 配置前先关闭

  USART_InitTypeDef uart;
  uart.USART_BaudRate            = 115200;
  uart.USART_HardwareFlowControl = USART_HardwareFlowControl_None;
  uart.USART_Mode                = USART_Mode_Rx | USART_Mode_Tx;
  uart.USART_StopBits            = USART_StopBits_1;
  uart.USART_WordLength          = USART_WordLength_8b;
  uart.USART_Parity              = USART_Parity_No;
  USART_Init(USART1, &uart);

  USART_DMACmd(USART1, USART_DMAReq_Rx, ENABLE);  // 使能 USART 的 DMA 发送请求
  USART_ITConfig(USART1, USART_IT_IDLE, ENABLE);   // 使能空闲中断
  USART_Cmd(USART1, ENABLE);                       // 开启 USART

  // === 5. NVIC ===
  NVIC_InitTypeDef nvic;
  nvic.NVIC_IRQChannel                   = USART1_IRQn;
  nvic.NVIC_IRQChannelPreemptionPriority = 0;
  nvic.NVIC_IRQChannelSubPriority        = 1;
  nvic.NVIC_IRQChannelCmd                = ENABLE;
  NVIC_Init(&nvic);

  return OK;
}
```

### uart_send — 阻塞发送

```c
uart_status uart_send(USART_TypeDef *USARTx, uint8_t *data, uint16_t len, uint32_t timeout) {
  uint32_t start = get_tick();

  for (uint16_t i = 0; i < len; i++) {
    while (USART_GetFlagStatus(USARTx, USART_FLAG_TXE) == RESET) {
      if (get_tick() - start > timeout) return SEND_TIMEOUT;
    }
    USART_SendData(USARTx, data[i]);
  }

  while (USART_GetFlagStatus(USARTx, USART_FLAG_TC) == RESET) {
    if (get_tick() - start > timeout) return SEND_TIMEOUT;
  }

  return SEND_OK;
}
```

### uart_read_frame — 从环形缓冲区读取一帧

```c
uint16_t uart_read_frame(USART_TypeDef *USARTx, uint8_t *buf, uint16_t max_len) {
  if (USART1 != USARTx || !frame_ready) return 0;

  // 通过 CNDTR 计算 DMA 当前写入位置
  uint16_t cndtr       = DMA1_Channel5->CNDTR;
  uint16_t write_index = (cndtr == RX_BUFFER_SIZE) ? 0 : (RX_BUFFER_SIZE - cndtr);
  uint16_t read        = last_read_index;

  // 从上次读取位置到当前写入位置，取出数据
  uint16_t count = 0;
  while (read != write_index && count < max_len - 1) {
    buf[count++] = dma_rx_buffer[read];
    read = (read + 1) % RX_BUFFER_SIZE;
  }

  // 去除末尾 \r\n
  while (count > 0 && (buf[count - 1] == '\r' || buf[count - 1] == '\n')) {
    count--;
  }
  buf[count] = '\0';

  last_read_index = read;
  frame_ready = 0;
  return count;
}
```

### USART1_IRQHandler — IDLE 中断

```c
void USART1_IRQHandler(void) {
  if (USART_GetITStatus(USART1, USART_IT_IDLE) != RESET) {
    USART_ReceiveData(USART1);  // 读 DR 清除 IDLE 标志（先读 SR 在 GetITStatus 中已完成）
    frame_ready = 1;
  }
}
```

---

## 关键寄存器详解

### DMA1_Channel5

| 寄存器 | 全称 | 说明 |
|--------|------|------|
| `CPAR`  | Peripheral Address Register | 外设地址，固定 `&USART1->DR` |
| `CMAR`  | Memory Address Register    | 内存地址，**配置后不变**，传输期间不会更新 |
| `CNDTR` | Number of Data Register    | 剩余传输次数，每传 1 字节减 1；循环模式到 0 自动重载为 `BufferSize` |
| `CCR`   | Configuration Register     | EN / CIRC / DIR / MINC / PSIZE / MSIZE 等 |

> [!tip] CNDTR 推算当前写入位置
> ```
> 写入索引 = (CNDTR == BUFFER_SIZE) ? 0 : (BUFFER_SIZE - CNDTR)
> ```
> 例：`RX_BUFFER_SIZE=256`
> - `CNDTR=256` → 刚重载，写入位置 0
> - `CNDTR=200` → 已收 56 字节，写入位置 56
> - `CNDTR=1`   → 已收 255 字节，写入位置 255

### USART

| 寄存器 | 说明 |
|--------|------|
| `SR`  | 状态寄存器：IDLE / TC / TXE / RXNE / ORE / NE / FE |
| `DR`  | 数据寄存器，读 = 接收，写 = 发送 |
| `CR1` | 控制寄存器：UE / IDLEIE / TE / RE / RXNEIE |

### IDLE 中断

- **触发条件**：RX 线空闲超过 1 帧传输时间
- **清除方式**：先读 `SR`，再读 `DR`
  - `USART_GetITStatus` 读 SR
  - `USART_ReceiveData` 读 DR
- **注意**：IDLE 标志**不会**自动重触发，清除后下次空闲才会再次置位

---

## 环形缓冲区工作机制

```
初始化:  read = 0, write = 0
         ┌───┬───┬───┬───┬───┬───┬───┬───┐
         │   │   │   │   │   │   │   │   │
         └───┴───┴───┴───┴───┴───┴───┴───┘
         ↑
       read/write

收到数据: DMA 向 write 位置写入，write 递增
         ┌───┬───┬───┬───┬───┬───┬───┬───┐
         │ A │ B │ C │   │   │   │   │   │
         └───┴───┴───┴───┴───┴───┴───┴───┘
         ↑           ↑
       read        write

IDLE 中断: frame_ready = 1, 任务调用 uart_read_frame
         ┌───┬───┬───┬───┬───┬───┬───┬───┐
         │ A │ B │ C │   │   │   │   │   │
         └───┴───┴───┴───┴───┴───┴───┴───┘
         ↑           ↑
       read=0      write=3

读取后:  read 更新到 write 位置
         ┌───┬───┬───┬───┬───┬───┬───┬───┐
         │ A │ B │ C │   │   │   │   │   │
         └───┴───┴───┴───┴───┴───┴───┴───┘
                     ↑
                   read/write
```

---

## 使用示例

```c
// main.c
int main(void) {
  HAL_Init();
  uart_init();

  uint8_t rx_buf[128];

  while (1) {
    uint16_t len = uart_read_frame(USART1, rx_buf, sizeof(rx_buf));
    if (len > 0) {
      // 处理接收到的数据帧
      uart_send(USART1, rx_buf, len, 1000);
    }
  }
}
```

---

## 注意事项

1. **`CMAR` 不会更新**：STM32F1 的 `CMAR` 在 DMA 使能后保持初值，必须用 `CNDTR` 计算写入位置
2. **IDLE 中断清除**：必须读 SR+DR，只读 DR 不会清除标志
3. **环形缓冲区溢出**：如果 DMA 写入速度超过读取速度，旧数据会被覆盖，可根据需要添加溢出检测
4. **`frame_ready` 是 volatile**：在中断和主循环间共享，必须加 volatile
5. **DMA 通道映射**：USART1_RX 固定映射到 DMA1_Channel5，不可更改