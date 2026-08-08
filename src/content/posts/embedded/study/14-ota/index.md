---
title: OTA Bootloader (简易版，未校验)
category: 嵌入式
description: 简易串口OTA bootloader升级，无校验
draft: false
published: 2026-06-10
tags:
  - 嵌入式
  - STM32
  - UART
---
# OTA Bootloader (简易版，未校验)

基于 UART 的串口 OTA（Over-The-Air）升级 bootloader
通过串口接收固件、写入 Flash，然后跳转到应用执行。

## 文件结构

```
bootloader/
├── startup.c                      # bootloader 主程序（含向量表、启动、OTA 协议）
├── stm32f10x_md_bootloader.ld     # bootloader 链接脚本 (0x08000000, 16KB)
└── stm32f10x_md_app.ld            # 应用链接脚本 (0x08004000, 48KB)

tools/
└── ota_upgrade.py                 # 主机端升级工具（Python + pyserial）
```

---

## Flash 分区

```
地址区间                    大小    内容
0x08000000 - 0x08003FFF    16KB    Bootloader（第 0~15 页）
0x08004000 - 0x0800FFFF    48KB    Application（第 16~63 页）
```

- STM32F103（Medium Density）每页 1KB，共 64 页（`FLASH_PAGE_SIZE = 1024`）。
- Bootloader 分区 16 页 = `BOOT_PAGE_COUNT`，应用分区 48 页 = `APP_PAGE_COUNT`。
- 应用区起始地址 `APP_BASE = 0x08004000`。

---

## 启动流程

```
上电/复位
    │
    ▼
clock_init()       ── HSE 8MHz → PLL ×9 → 72MHz
    │
    ▼
systick_init()     ── SysTick 1ms 中断
    │
    ▼
uart_init()        ── USART1: PA9(TX)/PA10(RX), 115200-8N1
    │
    ▼
delay_ms(2000)     ── 固定等待 2s，为主机按复位提供时间
    │
    ▼
等待握手窗口 500ms（BOOT_WAIT_MS）
    │
    ├── 收到 0xAA (SYNC_SEND) ─→ 进入 OTA 模式
    │         │
    │         ▼
    │    flash_erase_app()   ← 先擦除 48KB 应用区
    │         │
    │         ▼
    │    发送 0x55 (SYNC_RESP)
    │         │
    │         ▼
    │    ota_receive()       ← 接收并写入固件
    │         │
    │         ▼
    │    jump_to_app()
    │
    └── 超时（未收到 0xAA）─→ jump_to_app()  ← 正常启动
```

### 关键点

- **先擦除再应答**：bootloader 收到 `0xAA` 后立即调用 `flash_erase_app()` 擦除 48KB，之后才发送 `0x55`。擦除耗时约 48ms（1ms/页）。
- 主机侧握手循环会持续发送 `0xAA` 直到读到 `0x55`，因此擦除期间发送的 `0xAA` 会被丢弃，不影响后续。
- **排空残留 `0xAA` 的安全时序**：发送 `0x55` 后 bootloader 先 `delay_ms(100)`，再排空接收缓冲。此时 host 还在 `time.sleep(0.15)` 中，尚未重置缓冲，更未发送 `CMD_DATA`。排空的只有残留的 `0xAA`，不会误吞后续命令。`delay_ms(100)` 结束后排空到 host 开始发数据之间还有约 50ms 的余量。
- **无应用有效性校验**：bootloader 不会检查应用区首两个字是否合法，OTA 完成后无条件 `jump_to_app()`。

---

## 跳转逻辑（`jump_to_app`）

```c
1. 关闭中断    NVIC_DisableIRQ() / NVIC_ClearPendingIRQ()
2. 关全局中断  __asm("cpsid i")
3. 设置向量表  SCB->VTOR = APP_BASE (0x08004000)
4. 数据同步    __DSB() / __ISB()
5. 取 MSP      vectors[0]，写入 msp
6. 跳转 ResetHandler  (*(void(*)())(vectors[1]))()
```

---

## 通信协议

### 物理层

- USART1: PA9(TX), PA10(RX)
- 波特率: 115200
- 格式: 8N1
- 流控: **无硬件流控**，采用停等协议（每包等 ACK）

### 常量定义

| 常量                 | 值      | 含义                |
| ------------------ | ------ | ----------------- |
| `SYNC_SEND`        | `0xAA` | 握手请求（主机→BL）       |
| `SYNC_RESP`        | `0x55` | 握手应答（BL→主机）       |
| `CMD_DATA`         | `0x01` | 数据包               |
| `CMD_END`          | `0x02` | 传输结束              |
| `RSP_ACK`          | `0x06` | 成功                |
| `RSP_NAK`          | `0x15` | 失败                |
| `PACKET_DATA_SIZE` | `254`  | 单包最大数据长度          |

### 握手流程

```
主机                         Bootloader
  │                              │
  │──── 0xAA ────────────────→│  (BL 擦除 48KB)
  │                              │  擦除完成后
  │←─── 0x55 ───────────────────│  就绪
```

主机在 100s 内持续发送 `0xAA`，直到读到 `0x55` 为止。

### 数据包格式

实际实现（无序号、无 CRC）：

```
主机发送：
  [CMD_DATA=0x01] [len] [data(len 字节)]

  len  — 数据长度，单字节，取值 1~254（PACKET_DATA_SIZE）
  data — 固件数据，最多 254 字节

Bootloader 回复：
  ACK (0x06) → 写入成功，可以发下一包
  NAK (0x15) → 超时/非法命令，传输中止
```

### 结束包

```
主机发送：
  [CMD_END=0x02]

Bootloader 回复：
  ACK (0x06) → 接收结束，跳转应用
```

### 超时与控制流

- 每条命令/每字节接收超时：1000ms
- 收到 `0xAA` 前的握手窗口：500ms（`BOOT_WAIT_MS`）
- 启动固定延时：2000ms
- 握手超时（未进 OTA）：直接 `jump_to_app()`
- 协议中途任何超时：发送 `NAK` 并终止

---

## Flash 操作

### 擦除（`flash_erase_app`）

```c
FLASH_Unlock();
逐页擦除 APP_BASE ~ APP_BASE + 48KB（每页 1KB，共 48 页）
FLASH_Lock();
```

### 编程（`flash_program_chunk`）

```c
FLASH_Unlock();
// 每 2 字节组成一个半字写入
for (i = 0; i + 1 < len; i += 2)
    FLASH_ProgramHalfWord(addr + i, buf[i] | (buf[i+1] << 8));
// 若剩余 1 字节（奇数长度），单独写一个半字
if (i < len)
    FLASH_ProgramHalfWord(addr + i, buf[i]);
FLASH_Lock();
```

- STM32F103 Flash 只能按 **半字（16-bit）** 编程。
- 单包最大 254 字节（偶数），实际不会出现奇数长度补位问题。

### OTA 接收循环（`ota_receive`）

```
addr = APP_BASE
死循环：
  读 cmd (1B)
    超时        → 发 NAK，返回 -1
    cmd==0x02   → 发 ACK，返回 0（成功）
    cmd!=0x01   → 发 NAK，返回 -1（非法命令）
    否则读 len (1B)，再读 len 字节 data
    写入 flash_program_chunk(addr, data, len)
    addr += len
    发 ACK
```

---

## 主机端工具（`tools/ota_upgrade.py`）

依赖 `pyserial`，用法：

```bash
python tools/ota_upgrade.py firmware.bin -p /dev/ttyUSB0 -b 115200
```

流程：

1. **握手**：打开串口、清空收发缓冲，提示"按复位键"；在 100s 内循环发送 `0xAA`，读到 `0x55` 则握手成功。
2. **发送固件**：将固件按 254 字节切块，逐包发送 `[CMD_DATA][len][data]`，每包等待 `ACK/NAK/BUSY` 应答。
3. **结束**：发送 `[CMD_END]`，等待 `ACK`。

## bootloader/startup.c

```c
#include "misc.h"
#include "stm32f10x.h"
#include "stm32f10x_flash.h"
#include "stm32f10x_gpio.h"
#include "stm32f10x_rcc.h"
#include "stm32f10x_usart.h"
#include <stdint.h>

#define APP_BASE 0x08004000
#define BOOT_PAGE_COUNT 16
#define APP_PAGE_COUNT 48
#define FLASH_PAGE_SIZE 1024

#define SYNC_SEND 0xAA
#define SYNC_RESP 0x55
#define CMD_DATA 0x01
#define CMD_END 0x02
#define CMD_EXEC 0x03
#define RSP_ACK 0x06
#define RSP_NAK 0x15
#define RSP_BUSY 0x18

#define PACKET_DATA_SIZE 254
#define BOOT_WAIT_MS 500

extern uint32_t _sdata, _edata, _sidata, _sbss, _ebss;

void Default_Handler(void) {
  while (1)
    ;
}
void Reset_Handler(void);
void NMI_Handler(void) __attribute__((weak, alias("Default_Handler")));
void HardFault_Handler(void) __attribute__((weak, alias("Default_Handler")));
void MemManage_Handler(void) __attribute__((weak, alias("Default_Handler")));
void BusFault_Handler(void) __attribute__((weak, alias("Default_Handler")));
void UsageFault_Handler(void) __attribute__((weak, alias("Default_Handler")));
void SVC_Handler(void) __attribute__((weak, alias("Default_Handler")));
void DebugMon_Handler(void) __attribute__((weak, alias("Default_Handler")));
void PendSV_Handler(void) __attribute__((weak, alias("Default_Handler")));
void SysTick_Handler(void);
int main(void);
static void flash_erase_app(void);
static int ota_receive(void);

#define IRQ_HANDLER(x)                                                         \
  void x(void) __attribute__((weak, alias("Default_Handler")))
IRQ_HANDLER(WWDG_IRQHandler);
IRQ_HANDLER(PVD_IRQHandler);
IRQ_HANDLER(TAMPER_IRQHandler);
IRQ_HANDLER(RTC_IRQHandler);
IRQ_HANDLER(FLASH_IRQHandler);
IRQ_HANDLER(RCC_IRQHandler);
IRQ_HANDLER(EXTI0_IRQHandler);
IRQ_HANDLER(EXTI1_IRQHandler);
IRQ_HANDLER(EXTI2_IRQHandler);
IRQ_HANDLER(EXTI3_IRQHandler);
IRQ_HANDLER(EXTI4_IRQHandler);
IRQ_HANDLER(DMA1_Channel1_IRQHandler);
IRQ_HANDLER(DMA1_Channel2_IRQHandler);
IRQ_HANDLER(DMA1_Channel3_IRQHandler);
IRQ_HANDLER(DMA1_Channel4_IRQHandler);
IRQ_HANDLER(DMA1_Channel5_IRQHandler);
IRQ_HANDLER(DMA1_Channel6_IRQHandler);
IRQ_HANDLER(DMA1_Channel7_IRQHandler);
IRQ_HANDLER(ADC1_2_IRQHandler);
IRQ_HANDLER(USB_HP_CAN1_TX_IRQHandler);
IRQ_HANDLER(USB_LP_CAN1_RX0_IRQHandler);
IRQ_HANDLER(CAN1_RX1_IRQHandler);
IRQ_HANDLER(CAN1_SCE_IRQHandler);
IRQ_HANDLER(EXTI9_5_IRQHandler);
IRQ_HANDLER(TIM1_BRK_IRQHandler);
IRQ_HANDLER(TIM1_UP_IRQHandler);
IRQ_HANDLER(TIM1_TRG_COM_IRQHandler);
IRQ_HANDLER(TIM1_CC_IRQHandler);
IRQ_HANDLER(TIM2_IRQHandler);
IRQ_HANDLER(TIM3_IRQHandler);
IRQ_HANDLER(TIM4_IRQHandler);
IRQ_HANDLER(I2C1_EV_IRQHandler);
IRQ_HANDLER(I2C1_ER_IRQHandler);
IRQ_HANDLER(I2C2_EV_IRQHandler);
IRQ_HANDLER(I2C2_ER_IRQHandler);
IRQ_HANDLER(SPI1_IRQHandler);
IRQ_HANDLER(SPI2_IRQHandler);
IRQ_HANDLER(USART1_IRQHandler);
IRQ_HANDLER(USART2_IRQHandler);
IRQ_HANDLER(USART3_IRQHandler);
IRQ_HANDLER(EXTI15_10_IRQHandler);
IRQ_HANDLER(RTCAlarm_IRQHandler);
IRQ_HANDLER(USBWakeUp_IRQHandler);

typedef void (*vector_t)(void);

__attribute__((section(".isr_vector"), used)) vector_t g_pfnVectors[68] = {
    (vector_t)0x20005000, /* 初始 MSP = RAM 顶部 */
    Reset_Handler,
    NMI_Handler,
    HardFault_Handler,
    MemManage_Handler,
    BusFault_Handler,
    UsageFault_Handler,
    (vector_t)0,
    (vector_t)0,
    (vector_t)0,
    (vector_t)0,
    SVC_Handler,
    DebugMon_Handler,
    (vector_t)0,
    PendSV_Handler,
    SysTick_Handler,
    WWDG_IRQHandler,
    PVD_IRQHandler,
    TAMPER_IRQHandler,
    RTC_IRQHandler,
    FLASH_IRQHandler,
    RCC_IRQHandler,
    EXTI0_IRQHandler,
    EXTI1_IRQHandler,
    EXTI2_IRQHandler,
    EXTI3_IRQHandler,
    EXTI4_IRQHandler,
    DMA1_Channel1_IRQHandler,
    DMA1_Channel2_IRQHandler,
    DMA1_Channel3_IRQHandler,
    DMA1_Channel4_IRQHandler,
    DMA1_Channel5_IRQHandler,
    DMA1_Channel6_IRQHandler,
    DMA1_Channel7_IRQHandler,
    ADC1_2_IRQHandler,
    USB_HP_CAN1_TX_IRQHandler,
    USB_LP_CAN1_RX0_IRQHandler,
    CAN1_RX1_IRQHandler,
    CAN1_SCE_IRQHandler,
    EXTI9_5_IRQHandler,
    TIM1_BRK_IRQHandler,
    TIM1_UP_IRQHandler,
    TIM1_TRG_COM_IRQHandler,
    TIM1_CC_IRQHandler,
    TIM2_IRQHandler,
    TIM3_IRQHandler,
    TIM4_IRQHandler,
    I2C1_EV_IRQHandler,
    I2C1_ER_IRQHandler,
    I2C2_EV_IRQHandler,
    I2C2_ER_IRQHandler,
    SPI1_IRQHandler,
    SPI2_IRQHandler,
    USART1_IRQHandler,
    USART2_IRQHandler,
    USART3_IRQHandler,
    EXTI15_10_IRQHandler,
    RTCAlarm_IRQHandler,
    USBWakeUp_IRQHandler,
    (vector_t)0,
    (vector_t)0,
    (vector_t)0,
    (vector_t)0,
    (vector_t)0,
    (vector_t)0,
    (vector_t)0,
    (vector_t)0,
    (vector_t)0,
};

void Reset_Handler(void) {
  uint32_t *src = &_sidata;
  uint32_t *dst = &_sdata;
  while (dst < &_edata)
    *dst++ = *src++;
  dst = &_sbss;
  while (dst < &_ebss)
    *dst++ = 0;
  main();
  while (1)
    ;
}

static volatile uint32_t tick;

static void clock_init(void) {
  RCC_DeInit();
  RCC_HSEConfig(RCC_HSE_ON);
  while (!RCC_WaitForHSEStartUp())
    ;
  FLASH_SetLatency(FLASH_Latency_2);
  RCC_HCLKConfig(RCC_SYSCLK_Div1);
  RCC_PCLK2Config(RCC_HCLK_Div1);
  RCC_PCLK1Config(RCC_HCLK_Div2);
  RCC_PLLConfig(RCC_PLLSource_HSE_Div1, RCC_PLLMul_9);
  RCC_PLLCmd(ENABLE);
  while (!RCC_GetFlagStatus(RCC_FLAG_PLLRDY))
    ;
  RCC_SYSCLKConfig(RCC_SYSCLKSource_PLLCLK);
  while (RCC_GetSYSCLKSource() != 0x08)
    ;
  SystemCoreClock = 72000000;
}

void SysTick_Handler(void) { tick++; }

void delay_ms(uint32_t ms) {
  uint32_t target = tick + ms;
  while (tick < target)
    ;
}

static void systick_init(void) { SysTick_Config(SystemCoreClock / 1000); }

static void uart_init(void) {
  RCC_APB2PeriphClockCmd(RCC_APB2Periph_USART1 | RCC_APB2Periph_GPIOA, ENABLE);

  GPIO_InitTypeDef gpio;
  gpio.GPIO_Mode = GPIO_Mode_AF_PP;
  gpio.GPIO_Speed = GPIO_Speed_50MHz;
  gpio.GPIO_Pin = GPIO_Pin_9;
  GPIO_Init(GPIOA, &gpio);

  gpio.GPIO_Pin = GPIO_Pin_10;
  gpio.GPIO_Mode = GPIO_Mode_IN_FLOATING;
  GPIO_Init(GPIOA, &gpio);

  USART_InitTypeDef uart;
  uart.USART_BaudRate = 115200;
  uart.USART_HardwareFlowControl = USART_HardwareFlowControl_None;
  uart.USART_Mode = USART_Mode_Rx | USART_Mode_Tx;
  uart.USART_StopBits = USART_StopBits_1;
  uart.USART_WordLength = USART_WordLength_8b;
  uart.USART_Parity = USART_Parity_No;
  USART_Init(USART1, &uart);

  USART_Cmd(USART1, ENABLE);
}

static void jump_to_app(void) {
  uint32_t *vectors = (uint32_t *)APP_BASE;

  for (uint32_t i = 0; i < 8; i++)
    NVIC_DisableIRQ((IRQn_Type)i);
  for (uint32_t i = 0; i < 60; i++)
    NVIC_ClearPendingIRQ((IRQn_Type)i);

  __asm volatile("cpsid i");
  SCB->VTOR = APP_BASE;
  __asm volatile("dsb");
  __asm volatile("isb");

  uint32_t msp = vectors[0];
  uint32_t reset = vectors[1];
  void (*app_entry)(void) = (void (*)(void))reset;

  __asm volatile("msr msp, %0" : : "r"(msp));
  app_entry();
}

static uint8_t usart_send(USART_TypeDef *USARTx, const uint8_t data,
                          uint32_t timeout) {
  uint32_t start = tick;
  while (USART_GetFlagStatus(USARTx, USART_FLAG_TXE) == RESET &&
         tick - start < timeout)
    ;
  if (tick - start >= timeout)
    return -1;
  USART_SendData(USARTx, data);
  while (USART_GetFlagStatus(USARTx, USART_FLAG_TC) == RESET &&
         tick - start < timeout)
    ;
  if (tick - start >= timeout)
    return -1;
  return 0;
}

static int usart_recv(USART_TypeDef* USARTx, uint32_t timeout) {
  uint32_t start = tick;
  while(USART_GetFlagStatus(USARTx, USART_FLAG_RXNE) == RESET && tick - start < timeout);
  if (tick - start >= timeout) {
    return -1;
  }
  return USART_ReceiveData(USARTx);
}

int main(void) {
  clock_init();
  systick_init();
  uart_init();

  //wait for upgrade program
  delay_ms(2000);

  uint32_t start = tick;
  int ota = 0;
  while (tick - start < BOOT_WAIT_MS) {
    uint8_t c = usart_recv(USART1, BOOT_WAIT_MS - (tick - start));
    if (c == SYNC_SEND) {
      ota = 1;
      flash_erase_app();
      break;
    }
  }

  if (!ota) {
    jump_to_app();
    while (1)
      ;
  }

  while (USART_GetFlagStatus(USART1, USART_FLAG_RXNE) == SET)
    USART_ReceiveData(USART1);

  usart_send(USART1, SYNC_RESP, 100);

  delay_ms(100);
  while (USART_GetFlagStatus(USART1, USART_FLAG_RXNE) == SET)
    USART_ReceiveData(USART1);

  if (ota_receive() == 0) {
    jump_to_app();
  }
  while (1)
    ;
}

static void flash_erase_app(void) {
  FLASH_Unlock();
  for (uint32_t addr = APP_BASE; addr < APP_BASE + APP_PAGE_COUNT * FLASH_PAGE_SIZE;
       addr += FLASH_PAGE_SIZE)
    FLASH_ErasePage(addr);
  FLASH_Lock();
}

static void flash_program_chunk(uint32_t addr, const uint8_t *buf, uint32_t len) {
  FLASH_Unlock();
  uint32_t i = 0;
  for (; i + 1 < len; i += 2) {
    uint16_t w = (uint16_t)buf[i] | ((uint16_t)buf[i + 1] << 8);
    FLASH_ProgramHalfWord(addr + i, w);
  }
  if (i < len) {
    FLASH_ProgramHalfWord(addr + i, buf[i]);
  }
  FLASH_Lock();
}

static int ota_receive(void) {
  static uint8_t flash_buf[256];

  uint32_t addr = APP_BASE;
  while (1) {
    int cmd = usart_recv(USART1, 1000);
    if (cmd < 0) {
      usart_send(USART1, RSP_NAK, 100);
      return -1;
    }
    if (cmd == CMD_END) {
      usart_send(USART1, RSP_ACK, 100);
      return 0;
    }
    if (cmd != CMD_DATA) {
      usart_send(USART1, RSP_NAK, 100);
      return -1;
    }

    int len = usart_recv(USART1, 1000);
    if (len < 0) {
      usart_send(USART1, RSP_NAK, 100);
      return -1;
    }

    for (uint32_t i = 0; i < (uint32_t)len; i++) {
      int b = usart_recv(USART1, 1000);
      if (b < 0) {
        usart_send(USART1, RSP_NAK, 100);
        return -1;
      }
      flash_buf[i] = (uint8_t)b;
    }

    flash_program_chunk(addr, flash_buf, len);
    addr += len;
    usart_send(USART1, RSP_ACK, 100);
  }
}
```

## tools/ota_upgrade.py

```python
#!/usr/bin/env python3
import serial
import sys
import argparse
import time


SYNC_SEND = 0xAA
SYNC_RESP = 0x55
CMD_DATA = 0x01
CMD_END = 0x02
CMD_EXEC = 0x03
RSP_ACK = 0x06
RSP_NAK = 0x15
RSP_BUSY = 0x18

PACKET_DATA_SIZE = 254


def main():
    parser = argparse.ArgumentParser(description="STM32 OTA Upgrade Tool")
    parser.add_argument("firmware", help="path to firmware .bin file")
    parser.add_argument("-p", "--port", default="/dev/ttyUSB0")
    parser.add_argument("-b", "--baud", type=int, default=115200)
    args = parser.parse_args()

    with serial.Serial(args.port, args.baud, timeout=100) as ser:
        ser.reset_input_buffer()
        ser.reset_output_buffer()

        print("[..] press reset on the board now...")
        deadline = time.monotonic() + 100
        while time.monotonic() < deadline:
            ser.write(bytes([SYNC_SEND]))
            resp = ser.read(1)
            if resp :
                print(resp[0])
            if resp and resp[0] == SYNC_RESP:
                print("[OK] handshake")
                break
        else:
            print("[FAIL] handshake (timeout)")
            sys.exit(1)

        time.sleep(0.15)
        ser.reset_input_buffer()
        ser.reset_output_buffer()

        with open(args.firmware, "rb") as f:
            data = f.read()
        print(f"[..] firmware {len(data)} bytes")

        def read_ack():
            while True:
                resp = ser.read(1)
                if not resp:
                    return None
                b = resp[0]
                if 0xE0 <= b <= 0xE5 or 0xF0 <= b <= 0xF5:
                    kind = "erase" if (b & 0x10) == 0 else "program"
                    print(f"[ERR] {kind} failed, status={b & 0x0F}")
                    sys.exit(1)
                if b == 0xCE:
                    dump = ser.read(4)
                    print(f"[DBG] mid flash: {dump.hex()}")
                    continue
                if b in (RSP_ACK, RSP_NAK, RSP_BUSY):
                    return b

        for i in range(0, len(data), PACKET_DATA_SIZE):
            chunk = data[i:i + PACKET_DATA_SIZE]
            packet = bytes([CMD_DATA, len(chunk)]) + chunk
            ser.write(packet)
            ack = read_ack()
            if ack == RSP_ACK:
                print(f"[OK] packet {i // PACKET_DATA_SIZE} ({len(chunk)}B)")
            else:
                print(f"[FAIL] packet {i // PACKET_DATA_SIZE}: got {ack if ack is not None else 'timeout'}")
                sys.exit(1)

        ser.write(bytes([CMD_END]))
        ack = read_ack()
        if ack == RSP_ACK:
            print("[OK] end")
        else:
            print(f"[FAIL] end: got {ack if ack is not None else 'timeout'}")
            sys.exit(1)


if __name__ == "__main__":
    main()
```
