---
title: 传感器笔记
description: STM32F103ZET6 传感器笔记：超声波 HC-SR04、旋转编码器、DHT11 温湿度
tags:
  - STM32
  - 嵌入式
  - HAL库
  - 定时器
  - 输入捕获
  - 编码器
  - 超声波
draft: false
published: 2026-05-28
category: 嵌入式
---

# STM32F103ZET6 传感器

基于 **STM32F103ZET6** 的综合演示板，核心传感器如下：

| 传感器 | 功能 | 占用资源 |
|--------|------|----------|
| **HC-SR04 超声波** | 测距（2cm~400cm） | PG14(TRIG), PA1(ECHO→TIM2_CH2) |
| **旋转编码器** | 角度/位置检测 | PE9(TIM1_CH1), PE11(TIM1_CH2) |
| **DHT11 温湿度** | 温湿度测量 | PG13(DATA) |

系统时钟：**HSE 8MHz × PLL ×9 = 72MHz**，APB1 = 36MHz

---

## 一、超声波传感器 HC-SR04

### 1.1 工作原理

HC-SR04 通过**声波往返时间**测量距离：

```
测距公式：distance(cm) = echo_time(μs) / 58
```

**工作流程：**
1. **TRIG** 引脚输入 ≥10μs 高电平触发脉冲
2. 模块自动发射 8 个 40kHz 超声波脉冲
3. **ECHO** 引脚输出高电平，持续时间 = 声波往返时间
4. 声速 ≈ 340m/s，因此 1cm 对应往返时间 ≈ 58.8μs，取整 **58**

![image.png](https://imgbed.paxseeker.xyz/file/1785211977025_image.png)
### 1.2 接线

| HC-SR04 | STM32F103ZET6 | 说明 |
|---------|---------------|------|
| **VCC** | 5V | 模块供电 |
| **GND** | GND | 共地 |
| **TRIG** | **PG14** | GPIO 输出，发送触发脉冲 |
| **ECHO** | **PA1 (TIM2_CH2)** | 输入捕获，测量脉宽 |

### 1.3 定时器配置

使用 **TIM2 输入捕获 (Input Capture)** 精确测量 ECHO 高电平脉宽：

- **TIM2 预分频：** 71 → 72MHz / 72 = **1MHz**，1 tick = **1μs**
- **TIM2 自动重装载：** 999
- **输入捕获通道：** TIM2_CH2 (PA1)
- **捕获极性：** 上升沿 → 下降沿（双沿捕获）

**测距计算：**
```
total_ticks = overflow × 1000 + (end - start)  // μs
distance    = total_ticks / 58                   // cm
```

当脉冲宽度 > 1000μs（距离 > 17cm）时计数器溢出，`overflow` 由 `HAL_TIM_PeriodElapsedCallback` 每次溢出递增。

### 1.4 代码实现

**触发流程（ultra.c）：**
```c
void ultra_trig(void) {
    // 状态守卫：上一次测量未完成则跳过
    if (ultra_state != 0) return;

    // 1. 发送 ≥10μs 触发脉冲
    HAL_GPIO_WritePin(ULTRA_TRIG_PORT, ULTRA_TRIG_PIN, GPIO_PIN_SET);
    delay_us(10);
    HAL_GPIO_WritePin(ULTRA_TRIG_PORT, ULTRA_TRIG_PIN, GPIO_PIN_RESET);

    // 2. 清零状态
    ultra_overflow = 0;
    ultra_ready = 0;
    ultra_state = 1;

    // 3. 使能更新中断（用于溢出计数）
    __HAL_TIM_ENABLE_IT(&htim2, TIM_IT_UPDATE);
    // 4. 设置为上升沿捕获
    TIM2->CCER &= ~TIM_CCER_CC2P;
    // 5. 启动输入捕获中断
    HAL_TIM_IC_Start_IT(&htim2, TIM_CHANNEL_2);
}
```

**中断回调（tim.c）：**
```c
void HAL_TIM_IC_CaptureCallback(TIM_HandleTypeDef *htim) {
    if (htim->Instance != TIM2) return;

    if (ultra_state == 1) {
        // 上升沿：记录起始值
        ultra_start = HAL_TIM_ReadCapturedValue(&htim2, TIM_CHANNEL_2);
        ultra_overflow = 0;
        TIM2->CCER |= TIM_CCER_CC2P;   // 切换为下降沿捕获
        ultra_state = 2;
    } else if (ultra_state == 2) {
        // 下降沿：记录结束值，计算完成
        ultra_end = HAL_TIM_ReadCapturedValue(&htim2, TIM_CHANNEL_2);
        TIM2->CCER &= ~TIM_CCER_CC2P;  // 恢复上升沿
        HAL_TIM_IC_Stop_IT(&htim2, TIM_CHANNEL_2);
        __HAL_TIM_DISABLE_IT(&htim2, TIM_IT_UPDATE);
        ultra_ready = 1;
        ultra_state = 0;
    }
}

void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim) {
    if (htim->Instance == TIM2 && ultra_state == 2) {
        ultra_overflow++;   // 计数器溢出，脉冲很宽
    }
}
```

**读取距离（ultra.c）：**
```c
uint32_t ultra_dist_cm(void) {
    if (!ultra_ready) return 0;
    int32_t ticks = ultra_overflow * 1000 + (int32_t)(ultra_end - ultra_start);
    if (ticks < 0) return 0;
    return ticks / 58;   // cm
}
```

**主循环调用（main.c）：**
```c
uint32_t last_ultra_tick = 0;
while (1) {
    // 每 100ms 触发一次测距
    if (HAL_GetTick() - last_ultra_tick >= 100) {
        ultra_trig();
        last_ultra_tick = HAL_GetTick();
    }
    // ... 读取距离 ultra_dist_cm()
}
```

> **注意：** 每次触发前必须确保上一次测量已完成（`ultra_state == 0`），否则会导致捕获状态混乱。

---

## 二、旋转编码器

### 2.1 工作原理

旋转编码器输出两路**正交脉冲信号**（A/B 相），相位差 90°：

![image.png](https://imgbed.paxseeker.xyz/file/1785212138927_image.png)
- **正向旋转：** A 相超前 B 相 90°
- **反向旋转：** B 相超前 A 相 90°

STM32 的**硬件编码器模式**自动处理方向判断，计数器根据旋转方向自动加减。

**分辨率：** 每个脉冲 = 1 计数。典型的 12ppr 编码器在 TI1 模式下每圈产生 4×12 = **48 个计数**。

### 2.2 接线

| 编码器 | STM32F103ZET6 | 说明 |
|--------|---------------|------|
| **VCC** | 3.3V 或 5V | 供电 |
| **GND** | GND | 共地 |
| **A 相** | **PE9 (TIM1_CH1)** | 编码器输入 |
| **B 相** | **PE11 (TIM1_CH2)** | 编码器输入 |

> ZET6 的 TIM1 经 AFIO 重映射后，CH1→PE9, CH2→PE11

### 2.3 定时器配置

```c
// TIM1 编码器模式
htim1.Instance = TIM1;
htim1.Init.Prescaler = 0;
htim1.Init.CounterMode = TIM_COUNTERMODE_UP;
htim1.Init.Period = 65535;          // 16-bit 计数器
sConfig.EncoderMode = TIM_ENCODERMODE_TI1;
sConfig.IC1Polarity = TIM_ICPOLARITY_RISING;
sConfig.IC2Polarity = TIM_ICPOLARITY_RISING;
```

### 2.4 代码使用

```c
// 启动编码器
HAL_TIM_Encoder_Start(&htim1, TIM_CHANNEL_ALL);

// 读取当前计数值（有符号，可正可负）
int32_t position = __HAL_TIM_GET_COUNTER(&htim1);

// 清零
__HAL_TIM_SET_COUNTER(&htim1, 0);
```

---

## 三、DHT11 温湿度传感器

### 3.1 工作原理

DHT11 使用**单总线协议**通信，数据线为开漏结构，需上拉电阻：

```
主机 → DHT11：拉低 ≥18ms → 拉高 20-40μs → 释放总线
DHT11 → 主机：拉低 ~80μs → 拉高 ~80μs（响应）→ 发送 40bit 数据
```

**数据格式（40bit）：**
```
8bit 湿度整数 + 8bit 湿度小数 + 8bit 温度整数 + 8bit 温度小数 + 8bit 校验和
校验和 = (湿度整数+湿度小数+温度整数+温度小数) & 0xFF
```

### 3.2 接线

| DHT11 | STM32F103ZET6 | 说明 |
|-------|---------------|------|
| **VCC** | 3.3V | 供电 |
| **GND** | GND | 共地 |
| **DATA** | **PG13** | GPIO 双向数据 |

> 模块自带 4.7kΩ~10kΩ 上拉电阻，无需外加。若用裸 DHT11 传感器需外接上拉。

### 3.3 代码实现（dht11.c）

**GPIO 配置：**

```c
static void dht11_pin_output(void)
{
    GPIO_InitTypeDef init = {0};
    init.Pin = DHT11_PIN;
    init.Mode = GPIO_MODE_OUTPUT_PP;    // 推挽输出
    init.Pull = GPIO_PULLUP;
    init.Speed = GPIO_SPEED_FREQ_HIGH;
    HAL_GPIO_Init(DHT11_PORT, &init);
}

static void dht11_pin_input(void)
{
    GPIO_InitTypeDef init = {0};
    init.Pin = DHT11_PIN;
    init.Mode = GPIO_MODE_INPUT;        // 输入模式
    init.Pull = GPIO_PULLUP;            // 内部上拉（兼做总线空闲上拉）
    HAL_GPIO_Init(DHT11_PORT, &init);
}
```

**微秒级等待（基于 TIM7，1 tick = 1μs）：**

```c
static int dht11_wait_level(uint8_t level, uint32_t timeout_us)
{
    uint32_t start = __HAL_TIM_GET_COUNTER(&htim7);
    while (HAL_GPIO_ReadPin(DHT11_PORT, DHT11_PIN) != level) {
        uint16_t elapsed = (uint16_t)(__HAL_TIM_GET_COUNTER(&htim7) - start);
        if (elapsed >= timeout_us) return -1;
    }
    return 0;
}
```

**读取一位数据：**

```c
static int dht11_read_bit(void)
{
    // 等待数据线拉高（位起始）
    if (dht11_wait_level(GPIO_PIN_SET, 100) < 0) return -1;
    // 延时 30μs 后采样（0 的高电平约 26μs，1 约 70μs）
    delay_us(30);
    return HAL_GPIO_ReadPin(DHT11_PORT, DHT11_PIN);
}
```

**完整读取流程：**

```c
int dht11_read(void)
{
    memset((void *)dht11_data, 0, sizeof(dht11_data));

    // 1. 主机发起开始信号：拉低 ≥18ms → 拉高 20-40μs
    dht11_pin_output();
    HAL_GPIO_WritePin(DHT11_PORT, DHT11_PIN, GPIO_PIN_RESET);
    HAL_Delay(20);                      // 20ms 低电平
    HAL_GPIO_WritePin(DHT11_PORT, DHT11_PIN, GPIO_PIN_SET);
    delay_us(40);                       // 40μs 高电平

    // 2. 切换输入，等待 DHT11 响应
    dht11_pin_input();

    if (dht11_wait_level(GPIO_PIN_RESET, 500) < 0) return DHT11_ERR_INIT;
    if (dht11_wait_level(GPIO_PIN_SET, 500) < 0) return DHT11_ERR_INIT;

    // 3. 等待第一个数据位拉低
    if (dht11_wait_level(GPIO_PIN_RESET, 500) < 0) return DHT11_ERR_INIT;

    // 4. 读取 40 位（5 字节）
    for (uint8_t i = 0; i < 5; i++) {
        uint8_t byte = 0;
        for (uint8_t j = 0; j < 8; j++) {
            byte <<= 1;
            int bit = dht11_read_bit();
            if (bit < 0) return DHT11_ERR_INIT;
            byte |= bit;
        }
        dht11_data[i] = byte;
    }

    // 5. 校验和
    uint8_t sum = dht11_data[0] + dht11_data[1] + dht11_data[2] + dht11_data[3];
    if (sum != dht11_data[4]) return DHT11_ERR_CRC;

    dht11_humi = dht11_data[0];
    dht11_temp = dht11_data[2];
    return DHT11_OK;
}
```

### 3.4 主循环调用

```c
uint32_t last_dht_tick = 0;
while (1) {
    // 每 2 秒读取一次
    if (HAL_GetTick() - last_dht_tick >= 2000) {
        dht11_read();
        last_dht_tick = HAL_GetTick();
    }
    // 显示数据
    display_set_line(&display, 2, "T:%02d H:%02d", dht11_temp, dht11_humi);
}
```
