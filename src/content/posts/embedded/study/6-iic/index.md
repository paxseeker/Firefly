---
tags:
  - I2C
  - 嵌入式
  - HAL库
title: I2C通信
published: 2026-05-26
draft: false
category: 嵌入式
image: https://imgbed.paxseeker.xyz/file/1784634254542_night-sky.jpg
description: I2C通信--0.96寸oled屏幕显示
---

# I²C 通信

>[!tip] **I²C 与通信原理简介**
>
> **I²C (Inter-Integrated Circuit)** 是一种同步串行通信协议，由 Philips 在 1980 年代提出。仅通过**两根信号线**即可实现通信：
>
> - **SCL** (Serial Clock) — 由主机产生的时钟信号，驱动数据同步传输
> - **SDA** (Serial Data) — 双向数据总线，主机和从机共享
>
> **核心原理**：
> 1. **半双工**：SDA 为双向共用线，同一时刻只能有一个方向传输
> 2. **开漏输出**：SCL 和 SDA 均为开漏结构，需外部上拉电阻，总线空闲时为高电平
> 3. **地址寻址**：每个从机有唯一 7 位/10 位地址，主机通过地址字节选中目标从机
> 4. **时序约定**：
>    - **起始条件**：SCL 高电工作时 SDA 由高→低（S）
>    - **停止条件**：SCL 高电工作时 SDA 由低→高（P）
>    - **应答位 (ACK)**：接收方在第 9 个时钟周期拉低 SDA 表示确认
> 5. **三种速率模式**：标准 (100 kHz)、快速 (400 kHz)、高速 (3.4 MHz)
>
> **拓扑结构**：
> - **一主多从**（最常用）：一个主机 + 多个从机，主机控制 SCL，通过地址轮询每个从机
> - **多主多从**（支持但需仲裁）：多个主机挂载在同一总线上，通过"线与"仲裁机制决定谁获得总线控制权；任意时刻只有一个主机在发送
>
> **优势**：仅需 2 根线、地址寻址无需额外 CS 线、成本低
>
> **劣势**：半双工、速度低于 SPI、总线仲裁机制增加复杂度、开漏结构限制了速度上限

>[!warning] **为什么 I²C 必须使用开漏输出？**
>
> **1. 避免总线冲突（线与逻辑）**
> SDA 是双向共用线，主机和从机都可能驱动数据线。如果使用推挽输出，当一个设备输出高电平（驱动到 VCC）而另一个设备输出低电平（驱动到 GND）时，会造成**电源到地的直接短路**，可能烧毁器件。开漏输出只能主动拉低，无法主动拉高，因此多个设备同时拉低时不会出现冲突，实现自然的"线与"（Wire-AND）逻辑。
>
> **2. 多主设备总线仲裁**
> I²C 支持多主机共享同一总线。仲裁时，每个主机在发送"1"（释放总线）后回读 SDA 状态：如果读回"0"，说明有其他设备正在拉低总线，当前设备立即退出竞争。开漏结构使得任一设备拉低都能被所有设备检测到，这是仲裁机制的基础。
>
> **3. 电平兼容**
> 开漏输出通过上拉电阻决定高电平电压，使不同电压域的设备可以共享同一条总线（例如 3.3V 主机和 5V 从机通过 3.3V 上拉电阻通信），无需电平转换电路。

## 功能框图
![image.png](https://imgbed.paxseeker.xyz/file/1784529908451_image.png)

## I²C 通信详细时序

### 1. 主机写从机（Host Write）

**时序流程**：`START → 从机地址+写(0) → ACK → 数据字节1 → ACK → 数据字节2 → ACK → ... → STOP`

1. 主机发送 **START** 条件
2. 发送 **7位从机地址 + 写位(0)**，共 8 位
3. 从机响应 **ACK**（第 9 位拉低 SDA）
4. 主机逐字节发送数据，每字节后从机回 ACK
5. 数据传输完毕后，主机发送 **STOP** 条件

```
Master                    Slave
  |                         |
  |--[START]--------------->|
  |--[ADDR+W(0)]----------->|
  |<------[ACK]------------|
  |--[DATA1]--------------->|
  |<------[ACK]------------|
  |--[DATA2]--------------->|
  |<------[ACK]------------|
  |--[STOP]---------------->|
```

### 2. 主机读从机（Host Read）

**时序流程**：`START → 从机地址+写(0) → ACK → 内部地址(可选) → ACK → RESTART → 从机地址+读(1) → ACK → 数据 → NACK/ACK → STOP`

1. 主机发送 **START** + **从机地址+写位(0)**，指定从机
2. 可选：发送内部寄存器地址（让从机知道要读哪个位置）
3. 主机发送 **RESTART**（不释放总线，重新发 START）
4. 发送 **从机地址+读位(1)**，切换为读模式
5. 从机逐字节发送数据，主机每字节回 ACK
6. 最后一个字节主机回 **NACK**，然后发 **STOP**

```
Master                    Slave
  |                         |
  |--[START]--------------->|
  |--[ADDR+W(0)]----------->|
  |<------[ACK]------------|
  |--[REG ADDR]------------>|  (可选)
  |<------[ACK]------------|
  |--[RESTART]------------->|
  |--[ADDR+R(1)]----------->|
  |<------[ACK]------------|
  |<------[DATA1]----------|
  |--[ACK]----------------->|
  |<------[DATA2]----------|
  |--[NACK]---------------->|
  |--[STOP]---------------->|
```


### 3. 关键时序参数（标准模式 100kHz）

| 参数 | 描述 | 最小值 |
|------|------|--------|
| **t_SDA** | SDA 稳定建立时间（SCL 下降沿前） | 250 ns |
| **t_SCLH** | SCL 高电平持续时间 | 4.0 μs |
| **t_SCLL** | SCL 低电平持续时间 | 4.7 μs |
| **t_HOLD** | 保持时间（SCL 下降沿后 SDA 变化） | 0 ns |
| **t_RISE** | SDA 上升时间 | 1000 ns |
| **t_FALL** | SDA 下降时间 | 300 ns |
| **t_BUS_FREE** | 释放到下一次起始的时间 | 4.7 μs |

---

## STM32 I²C 相关寄存器

STM32 的 I²C 外设主要有以下几个关键寄存器（以 I²C1 为例）：

### I²C_SR1 — 状态寄存器 1

| 位 | 名称 | 含义 |
|----|------|------|
| 0 | **SB** | 起始条件已发送（主机模式，只读） |
| 1 | **ADDR** | 地址已匹配（主机发出地址 / 从机收到地址） |
| 2 | **BTF** | 字节传输完成（数据已发送且 ACK 已结束） |
| 3 | **ADD10** | 10 位地址匹配（只读） |
| 4 | **STOPF** | 停止条件检测到（从机模式） |
| 5 | **RXNE** | 数据寄存器非空（收到数据可读取） |
| 6 | **TXE** | 数据寄存器为空（可写入新数据） |
| 7 | **BERR** | 总线错误（起始/停止条件异常） |
| 8 | **AREF** | 仲裁丢失（多主模式） |
| 9 | **AF** | 应答失败（发送 ACK 时 NACK） |
| 10 | **OVER** | 过载（RXNE 未清零前又收到数据） |
| 11 | **PECERR** | PEC 校验错误 |
| 12 | **TIMEOUT** | 超时错误 |
| 13 | **SMBALERT** | SMBus 警报 |

### I²C_SR2 — 状态寄存器 2

| 位 | 名称 | 含义 |
|----|------|------|
| 0 | **BUSY** | 总线忙（高电平表示总线被占用） |
| 1 | **MSL** | 主/从模式（1=主机，0=从机） |
| 2 | **ADDRH** | 地址高位（10 位地址的高 2 位） |
| 3 | **ADDRH:0** | 地址位 |
| 13 | **PEC** | PEC 字节（16 位 CRC） |

### I²C_DR — 数据寄存器

- 双向寄存器，主机/从机均可读写
- 写入 DR：数据进入发送移位寄存器，TXE 置位
- 读取 DR：从接收移位寄存器取数据，RXNE 清除

### I²C_CR1 — 控制寄存器 1

| 位 | 名称 | 含义 |
|----|------|------|
| 0 | **PE** | I²C 外设使能 |
| 2 | **SMBUS** | SMBus 模式使能 |
| 4 | **SMBTYPE** | SMBus 设备类型（1= slave） |
| 5 | **ENPEC** | PEC 计算使能 |
| 6 | **ENARP** | 自动 PEC 接收使能 |
| 7 | **SMBDEF** | SMBus 设备故障标志 |
| 8 | **SMBALT** | SMBus 警报输入 |
| 9 | **START** | 生成起始条件（软件置位，硬件自动清零） |
| 10 | **STOP** | 生成停止条件（软件置位，硬件自动清零） |
| 11 | **ACK** | 应答使能（1=启用 ACK） |
| 12 | **PEC** | PEC 字节发送/接收使能 |
| 13 | **POS** | 应答位位置选择（用于 FM+ 模式） |
| 14 | **SCLL** | 时钟脉冲低电平控制 |

### I²C_CR2 — 控制寄存器 2

| 位 | 名称 | 含义 |
|----|------|------|
| 0 | **FREQUENCIES[3:0]** | 外设时钟频率配置（根据 APB1 时钟设定） |
| 4 | **C10STRETCH** | 10 位主机从扩展时钟 |
| 5 | **LOAD** | 预分频器加载 |
| 6 | **SLOW** | 慢模式（SCL 频率减半） |
| 7 | **ERRIE** | 错误中断使能 |
| 8 | **ITEVTEN** | 事件中断使能 |
| 9 | **ITBUFEN** | 缓冲中断使能 |
| 10 | **DMAEN** | DMA 请求使能 |
| 11 | **SOFTEND** | 软件端点使能 |

### I²C_OAR1 / I²C_OAR2 — 自身地址寄存器

- **OAR1**：配置自身 7 位或 10 位地址，从机模式时被匹配
- **OAR2**：配置第二个从机地址（可选，用于兼容多个地址）

### I²C_CCR — 时钟控制寄存器

| 位 | 名称 | 含义 |
|----|------|------|
| 0 | **FSMBUS** | 快速模式使能（1=快速模式，0=标准模式） |
| 11:0 | **DUTY** / **CCR[11:0]** | 标准模式为 SCL 时钟占空比；快速模式为时钟频率配置值 |

---

## 0.96 寸 OLED 显示屏（SSD1306）原理

### 概述

0.96 寸 OLED（有源矩阵有机发光二极管）屏幕是一种自发光显示器件，无需背光，对比度极高、视角大、功耗低。本项目使用 **SSD1306** 驱动芯片，通过 **I²C** 接口与 STM32 通信。

> **项目路径**：`spi/`（项目目录名虽为 spi，但 OLED 实际使用 I²C 接口驱动）

### 硬件规格

| 参数 | 值 |
|------|------|
| 分辨率 | **128 × 64** 像素 |
| 驱动芯片 | **SSD1306** |
| 通信接口 | **I²C**（7 位地址 0x78） |
| 显存映射 | **8 页 × 128 列**，每字节对应 1 列的 8 个垂直像素 |
| 显示颜色 | 单色（白/蓝），1 = 点亮，0 = 熄灭 |

### 显示原理

**1. 自发光机制**
OLED 的每个像素由有机发光材料构成，通电即发光，无需背光模组。像素可独立控制开/关，因此黑色完全不发光，对比度极高。

**2. SSD1306 驱动原理**
SSD1306 内置 **GRAM（Graphics RAM）**，分为 **8 页（Page 0–7）**，每页 **128 列**。每列存储 1 个字节，字节的 8 个 bit 分别控制该列从上到下的 8 个像素：

```
        Page 0 (bit7...bit0)
   ┌───┬───┬───┬───┬───┬───┬───┬───┐
   │ 1 │ 0 │ 1 │ 0 │ 1 │ 0 │ 1 │ 0 │  ← 列 x
   └───┴───┴───┴───┴───┴───┴───┴───┘

 每个字节 = 1 列 × 8 行像素（bit7=最上, bit0=最下）
```

- **地址模式**：项目使用 **页地址模式**（0x20 0x02），写入时只需指定起始页和列，数据按页顺序逐列填充
- **显示流程**：STM32 将数据写入 SSD1306 的 GRAM，驱动芯片自动将 GRAM 内容扫描到屏幕

**3. I²C 命令/数据帧结构**
SSD1306 通过 I²C 的 D/C 控制字节区分命令和数据：
- **D/C = 0x00**：后面字节为**命令**（寄存器配置）
- **D/C = 0x40**：后面字节为**数据**（写入 GRAM 的像素数据）

每次传输格式：`[D/C 控制字节][命令/数据字节]`

### 关键初始化命令解析

```c
// OLED_Init() 中发送的初始化序列：
OLED_Cmd(0xAE);        // 关显示（初始化前先关闭）
OLED_Cmd(0xD5);        // 设置时钟分频/振荡器频率
OLED_Cmd(0x80);        // 默认分频值
OLED_Cmd(0xA8);        // 设置多路复用率
OLED_Cmd(0x3F);        // 64 行 (64-1 = 0x3F)
OLED_Cmd(0xD3);        // 设置显示偏移
OLED_Cmd(0x00);        // 无偏移
OLED_Cmd(0x40);        // 设置段重映射起始行 (0x40)
OLED_Cmd(0x8D);        // 充电泵控制
OLED_Cmd(0x14);        // 开启充电泵
OLED_Cmd(0x20);        // 内存地址模式设置
OLED_Cmd(0x02);        // 页地址模式
OLED_Cmd(0xA1);        // 段重映射（左右翻转）
OLED_Cmd(0xC8);        // 行重映射（上下翻转）
OLED_Cmd(0xDA);        // 设置 COM 引脚配置
OLED_Cmd(0x12);        // 默认配置
OLED_Cmd(0x81);        // 设置对比度
OLED_Cmd(0xCF);        // 对比度值 0xCF
OLED_Cmd(0xD9);        // 设置预充电周期
OLED_Cmd(0xF1);        // 默认值
OLED_Cmd(0xDB);        // 设置 VCOMH 消隐电压
OLED_Cmd(0x40);        // 默认值
OLED_Cmd(0xA4);        // 整个显示遵循 RAM 内容（非全部点亮）
OLED_Cmd(0xA6);        // 正常显示（非反色）
OLED_Cmd(0xAF);        // 开显示
```

### 显存更新流程

1. 在 STM32 本地维护 **128×8 = 1024 字节**的帧缓冲区
2. 调用绘图函数（`OLED_DrawPixel`、`OLED_ShowChar` 等）修改缓冲区
3. 调用 `OLED_Update()` 将缓冲区逐页发送到 SSD1306 的 GRAM
4. SSD1306 自动扫描 GRAM 刷新屏幕

### 项目内 OLED 相关代码

#### 头文件 — oled.h

```c
#ifndef __OLED_H
#define __OLED_H

#include "main.h"
#include "stm32f1xx_hal_i2c.h"

#define OLED_I2C_ADDR        0x78
#define OLED_WIDTH           128
#define OLED_HEIGHT          64
#define OLED_BUFFER_SIZE     (OLED_WIDTH * 8)

typedef struct {
    I2C_HandleTypeDef *hi2c;
    uint8_t buffer[OLED_BUFFER_SIZE];
} OLED_HandleTypeDef;

HAL_StatusTypeDef OLED_Init(OLED_HandleTypeDef *oled);
void OLED_Clear(OLED_HandleTypeDef *oled);
void OLED_Update(OLED_HandleTypeDef *oled);
void OLED_DrawPixel(OLED_HandleTypeDef *oled, uint8_t x, uint8_t y, uint8_t color);
void OLED_ShowChar(OLED_HandleTypeDef *oled, uint8_t x, uint8_t y, char ch);
void OLED_ShowString(OLED_HandleTypeDef *oled, uint8_t x, uint8_t y, char *str);

#endif
```

#### 实现文件 — oled.c

```c
#include "oled.h"
#include <string.h>

// 6x8 点阵字模表（ASCII ' ' 到 '~'，共 95 个字符）
const uint8_t g_ucCode6x8[][6] = {
    {0x00,0x00,0x00,0x00,0x00,0x00},{0x00,0x00,0x5F,0x00,0x00,0x00},{0x00,0x07,0x00,0x07,0x00,0x00},
    // ... (共 95 个字符的字模数据)
    {0x10,0x08,0x08,0x10,0x08,0x00}
};

// 通过 I²C 写入字节（D/C + data）
static void OLED_WriteByte(OLED_HandleTypeDef *oled, uint8_t mode, uint8_t data) {
    uint8_t buf[2] = {mode, data};
    HAL_I2C_Master_Transmit(oled->hi2c, OLED_I2C_ADDR, buf, 2, 100);
}

// 发送命令（D/C = 0x00）
static void OLED_Cmd(OLED_HandleTypeDef *oled, uint8_t cmd) {
    OLED_WriteByte(oled, 0x00, cmd);
}

// 发送数据（D/C = 0x40）
static void OLED_Data(OLED_HandleTypeDef *oled, uint8_t dat) {
    OLED_WriteByte(oled, 0x40, dat);
}

// 设置坐标（页地址模式下）
static void OLED_SetPos(OLED_HandleTypeDef *oled, uint8_t x, uint8_t page) {
    OLED_Cmd(oled, 0xB0 | page);                           // 页地址 (0xB0~0xB7)
    OLED_Cmd(oled, ((x & 0xF0) >> 4) | 0x10);              // 列地址高 4 位 (0x10~0x1F)
    OLED_Cmd(oled, (x & 0x0F) | 0x01);                     // 列地址低 4 位 (0x01~0x0F)
}

// 初始化 OLED
HAL_StatusTypeDef OLED_Init(OLED_HandleTypeDef *oled) {
    HAL_Delay(100);
    OLED_Cmd(oled, 0xAE);       // 关显示
    OLED_Cmd(oled, 0xD5); OLED_Cmd(oled, 0x80);  // 时钟分频
    OLED_Cmd(oled, 0xA8); OLED_Cmd(oled, 0x3F);  // 复用率 64
    OLED_Cmd(oled, 0xD3); OLED_Cmd(oled, 0x00);  // 无偏移
    OLED_Cmd(oled, 0x40);                   // 起始行 0x40
    OLED_Cmd(oled, 0x8D); OLED_Cmd(oled, 0x14);  // 充电泵开
    OLED_Cmd(oled, 0x20); OLED_Cmd(oled, 0x02);  // 页地址模式
    OLED_Cmd(oled, 0xA1);                   // 段重映射
    OLED_Cmd(oled, 0xC8);                   // 行重映射
    OLED_Cmd(oled, 0xDA); OLED_Cmd(oled, 0x12);  // COM 配置
    OLED_Cmd(oled, 0x81); OLED_Cmd(oled, 0xCF);  // 对比度
    OLED_Cmd(oled, 0xD9); OLED_Cmd(oled, 0xF1);  // 预充电
    OLED_Cmd(oled, 0xDB); OLED_Cmd(oled, 0x40);  // VCOMH
    OLED_Cmd(oled, 0xA4);                   // 正常显示
    OLED_Cmd(oled, 0xA6);                   // 非反色
    OLED_Cmd(oled, 0xAF);                   // 开显示
    OLED_Clear(oled);
    OLED_Update(oled);
    return HAL_OK;
}

// 清屏
void OLED_Clear(OLED_HandleTypeDef *oled) {
    memset(oled->buffer, 0x00, OLED_BUFFER_SIZE);
}

// 将帧缓冲区内容更新到屏幕
void OLED_Update(OLED_HandleTypeDef *oled) {
    uint8_t page, col;
    for (page = 0; page < 8; page++) {
        OLED_SetPos(oled, 0, page);
        for (col = 0; col < OLED_WIDTH; col++) {
            OLED_Data(oled, oled->buffer[page * OLED_WIDTH + col]);
        }
    }
}

// 画像素
void OLED_DrawPixel(OLED_HandleTypeDef *oled, uint8_t x, uint8_t y, uint8_t color) {
    if (x >= OLED_WIDTH || y >= OLED_HEIGHT) return;
    uint8_t page = y / 8;
    uint8_t bit  = y % 8;
    uint16_t idx = page * OLED_WIDTH + x;
    if (color)
        oled->buffer[idx] |= (0x01 << bit);
    else
        oled->buffer[idx] &= ~(0x01 << bit);
}

// 显示单个字符（6x8 字模）
void OLED_ShowChar(OLED_HandleTypeDef *oled, uint8_t x, uint8_t y, char ch) {
    if (ch < ' ' || ch > '~') return;
    if (x >= OLED_WIDTH - 6 || y >= OLED_HEIGHT - 8) return;
    uint8_t index = ch - ' ';
    uint8_t page = y / 8;
    for (uint8_t i = 0; i < 6; i++) {
        if (x + i < OLED_WIDTH)
            oled->buffer[page * OLED_WIDTH + x + i] = g_ucCode6x8[index][i];
    }
}

// 显示字符串（自动换行）
void OLED_ShowString(OLED_HandleTypeDef *oled, uint8_t x, uint8_t y, char *str) {
    if (!str) return;
    while (*str) {
        OLED_ShowChar(oled, x, y, *str);
        x += 6;
        if (x >= OLED_WIDTH - 6) {
            x = 0;
            y += 8;
            if (y >= OLED_HEIGHT) return;
        }
        str++;
    }
}
```

#### 使用示例 — main.c（OLED 部分）

```c
OLED_HandleTypeDef oled;

// 在 main() 中初始化
oled.hi2c = &hi2c1;
OLED_Init(&oled);

// 每 1 秒刷新显示
if (HAL_GetTick() - last_oled_update >= 1000) {
    last_oled_update = HAL_GetTick();

    OLED_Clear(&oled);
    OLED_ShowString(&oled, 0, 0, "I2C OLED Demo");
    OLED_ShowString(&oled, 0, 10, "Count:");
    OLED_ShowString(&oled, 0, 20, "Temp:");
    OLED_ShowChar(&oled, 32, 10, counter % 10 + '0');
    OLED_ShowChar(&oled, 40, 10, (counter / 10) % 10 + '0');
    OLED_ShowChar(&oled, 48, 10, (counter / 100) % 10 + '0');
    OLED_ShowString(&oled, 32, 20, "25.6");

    OLED_Update(&oled);
}
```

#### I²C 配置 — i2c.c

```c
/* USER CODE BEGIN Header */
/**
  ******************************************************************************
  * @file    i2c.c
  * @brief   I2C HAL module initialization for OLED (SSD1306)
  ******************************************************************************
  */
/* USER CODE END Header */
/* Includes ------------------------------------------------------------------*/
#include "i2c.h"

/* USER CODE BEGIN 0 */

/* USER CODE END 0 */

I2C_HandleTypeDef hi2c1;

/* I2C1 init function */
void MX_I2C1_Init(void)
{

  /* USER CODE BEGIN I2C1_Init 0 */

  /* USER CODE END I2C1_Init 0 */

  /* USER CODE BEGIN I2C1_Init 1 */

  /* USER CODE END I2C1_Init 1 */
  hi2c1.Instance = I2C1;
  hi2c1.Init.ClockSpeed = 100000;
  hi2c1.Init.DutyCycle = I2C_DUTYCYCLE_2;
  hi2c1.Init.OwnAddress1 = 0;
  hi2c1.Init.AddressingMode = I2C_ADDRESSINGMODE_7BIT;
  hi2c1.Init.DualAddressMode = I2C_DUALADDRESS_DISABLE;
  hi2c1.Init.OwnAddress2 = 0;
  hi2c1.Init.GeneralCallMode = I2C_GENERALCALL_DISABLE;
  hi2c1.Init.NoStretchMode = I2C_NOSTRETCH_DISABLE;
  if (HAL_I2C_Init(&hi2c1) != HAL_OK)
  {
    Error_Handler();
  }
  /* USER CODE BEGIN I2C1_Init 2 */

  /* USER CODE END I2C1_Init 2 */

}

void HAL_I2C_MspInit(I2C_HandleTypeDef* i2cHandle)
{

  GPIO_InitTypeDef GPIO_InitStruct = {0};
  if(i2cHandle->Instance==I2C1)
  {
  /* USER CODE BEGIN I2C1_MspInit 0 */

  /* USER CODE END I2C1_MspInit 0 */

    __HAL_RCC_GPIOB_CLK_ENABLE();
    /**I2C1 GPIO Configuration
    PB8     ------> I2C1_SCL
    PB9     ------> I2C1_SDA
    */
    GPIO_InitStruct.Pin = GPIO_PIN_8|GPIO_PIN_9;
    GPIO_InitStruct.Mode = GPIO_MODE_AF_OD;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
    HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);

    __HAL_AFIO_REMAP_I2C1_ENABLE();

    /* I2C1 clock enable */
    __HAL_RCC_I2C1_CLK_ENABLE();
  /* USER CODE BEGIN I2C1_MspInit 1 */

  /* USER CODE END I2C1_MspInit 1 */
  }
}

void HAL_I2C_MspDeInit(I2C_HandleTypeDef* i2cHandle)
{

  if(i2cHandle->Instance==I2C1)
  {
  /* USER CODE BEGIN I2C1_MspDeInit 0 */

  /* USER CODE END I2C1_MspDeInit 0 */
    /* Peripheral clock disable */
    __HAL_RCC_I2C1_CLK_DISABLE();

    /**I2C1 GPIO Configuration
    PB8     ------> I2C1_SCL
    PB9     ------> I2C1_SDA
    */
    HAL_GPIO_DeInit(GPIOB, GPIO_PIN_8);

    HAL_GPIO_DeInit(GPIOB, GPIO_PIN_9);

  /* USER CODE BEGIN I2C1_MspDeInit 1 */

  /* USER CODE END I2C1_MspDeInit 1 */
  }
}

/* USER CODE BEGIN 1 */

/* USER CODE END 1 */


```

