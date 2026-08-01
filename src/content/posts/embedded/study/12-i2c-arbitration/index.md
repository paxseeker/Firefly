---
title: I2C 总线：多设备共享总线问题与解决方案
tags:
  - 嵌入式
  - I2C
  - 调度
category: 嵌入式
published: 2026-06-01
draft: false
description: I2C 总线：多设备共享总线问题与解决方案
---

# I2C 总线：多设备共享总线问题与解决方案

## 目录

1. [饥饿问题（Starvation）](#1-饥饿问题)
2. [数据生命周期问题（Data Lifecycle）](#2-数据生命周期问题)
3. [硬实时场景下的策略](#3-硬实时场景下的策略)
4. [带宽计算与吞吐量分析](#4-带宽计算与吞吐量分析)
5. [裸机 vs RTOS 方案对比](#5-裸机-vs-rtos-方案对比)
6. [防死锁机制](#6-防死锁机制)
7. [总结](#7-总结)

---

## 1. 饥饿问题

### 场景

多个 I2C 设备共享同一条总线，当某个设备频繁使用 DMA 传输长数据帧时，其他设备可能永远无法获得总线访问权，导致**饥饿（Starvation）**。

### 根因

I2C 总线是**串行半双工**的，同一时刻只能有一个传输在运行。当设备 A 启动 DMA 传输后，总线立即进入 BUSY 状态。若调度逻辑按顺序检查各设备，设备 A 的检查排在前面，每次它都有待发数据，则后续设备 B/C 永久无法获得总线使用权。

```c
/* 错误写法：顺序轮询 */
while (1) {
    if (state == IDLE) {
        if (need_send_dev_a) {
            HAL_I2C_Master_Transmit_DMA(...);  /* state → BUSY */
        }
        /* 以下代码永远不会被执行到 */
        if (need_send_dev_b) { ... }
        if (need_send_dev_c) { ... }
    }
}
```

### 解决方案：请求队列 + 循环调度（Round-Robin）

#### 架构

```
┌──────────────┐     ┌──────────────────────┐     ┌──────────────┐
│  设备驱动 A   │ ──→ │                      │     │  调度器      │
│              │     │   请求队列 + 循环指针  │ ←── │  main loop   │
├──────────────┤     │   (Round-Robin)       │     │  或 ISR      │
│  设备驱动 B   │ ──→ │                      │     │              │
└──────────────┘     └──────────────────────┘     └──────────────┘
```

#### 核心组件

**1. 请求入队（设备侧）**

各设备在有数据要发送时，不直接调用 I2C 外设，而是向调度器提交请求：

```c
void device_a_tick(void) {
    /* 准备数据 ... */
    request_enqueue(BUS_1, DEVICE_A_ID);
}

void device_b_tick(void) {
    request_enqueue(BUS_1, DEVICE_B_ID);
}
```

**2. 循环指针出队（调度器侧）**

`dequeue_idx` 记录上次服务的位置，每次从该位置开始扫描，找到第一个待处理设备后推进指针：

```c
uint8_t request_dequeue(BusId_t bus) {
    RequestQueue_t *q = &queues[bus];
    for (int i = 0; i < QUEUE_SLOTS; i++) {
        uint8_t idx = (q->dequeue_idx + i) % QUEUE_SLOTS;
        if (q->slots[idx].pending) {
            q->slots[idx].pending = 0;
            q->dequeue_idx = (idx + 1) % QUEUE_SLOTS;  /* 循环推进 */
            return q->slots[idx].owner_id;
        }
    }
    return OWNER_NONE;
}
```

**关键保障**：`dequeue_idx` 每次递增，同一设备不会连续被选中两次（除非它是唯一有请求的设备）。

**3. 中央调度器**

```c
void i2c_dispatcher(void) {
    if (bus_is_busy(BUS_1)) return;

    uint8_t dev = request_dequeue(BUS_1);
    if (dev == DEVICE_A) {
        device_a_transmit();
    } else if (dev == DEVICE_B) {
        device_b_transmit();
    }
}
```

#### 调度位置

| 调度方式 | 特点 | 适用场景 |
|----------|------|----------|
| 主循环轮询 | 每次循环调用一次 `dispatcher` | 实时性要求不高，实现简单 |
| DMA 中断链 | 在 DMA 完成 ISR 中调度下一个 | 需要背靠背传输，消除轮询延迟 |

---

## 2. 数据生命周期问题

### 问题描述

请求入队到调度器实际处理之间存在时间差，这段时间内数据可能已被修改：

```
t=0   dev_a: 入队请求（数据 = 旧值）
t=1   dev_a: 修改数据（数据 = 新值）
t=2   调度器: 取出 dev_a 请求，启动 DMA → 发送的是旧值还是新值？
```

### 方案对比

| 方案 | 数据安全 | RAM 开销 | CPU 开销 | 实现复杂度 | 适用场景 |
|------|----------|----------|----------|-----------|----------|
| 深拷贝入队 | 绝对安全 | 高 | 入队时 memcpy | 低 | 通用场景，首选 |
| 所有权转移 | 需约定 | 低（零拷贝） | 仅指针赋值 | 中 | RAM 紧张 |
| 乒乓缓冲 | 物理隔离 | 中（2x buffer） | 零额外开销 | 中 | 周期性数据流 |
| 阻塞等待 | 安全 | 无 | CPU 空转 | 低 | ❌ 裸机不推荐 |

### 2.1 深拷贝入队

数据入队时立即拷贝到队列专用缓冲区，DMA 只读队列缓冲区，上层数据可随意修改：

```c
typedef struct {
    uint8_t data[MAX_DATA_LEN];
    uint16_t len;
    uint16_t dev_addr;
    uint8_t pending;
} RequestSlot_t;

void request_enqueue_data(BusId_t bus, uint8_t owner,
                           uint8_t *data, uint16_t len) {
    RequestSlot_t *slot = find_idle_slot(bus, owner);
    if (!slot) return;

    memcpy(slot->data, data, len);  /* 关键：立即深拷贝，隔离上层数据 */
    slot->len = len;
    slot->pending = 1;
}
```

**优点**：调用方发送后可以立即释放或修改源数据，驱动层绝对安全，心智负担最低。

**缺点**：占用额外 RAM，入队时多一次 memcpy 开销。对于大块数据（如显示屏帧缓冲区）需评估 RAM 预算。

### 2.2 所有权转移（零拷贝）

调用方传入指针后放弃修改权，DMA 完成后通过回调归还：

```c
static uint8_t tx_buffer[64];

void send_data(void) {
    tx_buffer[0] = 0x01;
    request_enqueue_ptr(BUS_1, DEVICE_ID, tx_buffer, 64);
    /* 此时不能再修改 tx_buffer — 所有权已转移给驱动 */
}

/* DMA 完成中断中回调 */
void transfer_complete(void *buffer) {
    /* 归还所有权：调用方可以再次使用 buffer */
}
```

**优点**：零拷贝，节省 RAM 和 CPU 时间。

**缺点**：需协议约定，调用方必须遵守"传输期间不修改"的契约，违反时会导致数据错乱且难以调试。

### 2.3 乒乓缓冲（Ping-Pong Buffer）

两块缓冲区物理隔离，DMA 读一块时 CPU 写另一块，交替翻转：

```c
typedef struct {
    uint8_t front[POOL_SIZE];  /* DMA 正在读 */
    uint8_t back[POOL_SIZE];   /* CPU 正在写 */
    volatile uint8_t flipping;  /* 0: DMA 读 front; 1: DMA 读 back */
} PingPongPool_t;

void write_frame(PingPongPool_t *pool, uint8_t *data, uint16_t len) {
    uint8_t *idle = pool->flipping ? pool->front : pool->back;
    memcpy(idle, data, len);
}

HAL_StatusTypeDef start_dma(PingPongPool_t *pool) {
    uint8_t *active = pool->flipping ? pool->front : pool->back;
    pool->flipping = !pool->flipping;  /* 翻转，下次写入另一块 */
    return HAL_I2C_Master_Transmit_DMA(..., active, ...);
}
```

**优点**：数据安全（物理隔离），拷贝延迟隐藏在 DMA 后台传输期间，不增加关键路径耗时。

**缺点**：需要 2 倍缓冲区 RAM。对于 N 个设备，每个设备需要独立的缓冲区对。

```
时序示意：

     DMA 读 front          DMA 读 back
CPU 写 back ←─── 间隔 ──→ CPU 写 front
             ↑                   ↑
         翻转指针              翻转指针
```

### 2.4 阻塞等待（不推荐）

```c
while (bus_is_busy(BUS_1)) ;  /* CPU 空转 */
```

**裸机中不可取**：完全浪费 DMA 解放 CPU 的意义，会导致主循环停止、其他设备无法运行、定时器中断响应延迟增加。

---

## 3. 硬实时场景下的策略

当实时性要求极高（抖动 < 5μs）时，上述通用方案需要进一步优化。

### 3.1 短消息场景：放弃 DMA

对于 ≤ 8 字节的短消息，DMA 的配置开销可能超过直接传输本身：

```
DMA 方式: 配置寄存器 + 启动通道 ≈ 20μs + 传输 8 字节 ≈ 160μs = 180μs
轮询方式: 直接传输 8 字节 ≈ 160μs
```

**结论**：DMA 适合大块数据（如显示屏帧缓冲区），对于短控制消息，CPU 直接轮询的延迟更低且更确定。

### 3.2 优先级抢占 + 最新值覆盖

公平轮询本身是实时性的天敌——高优先级设备不能因为排在队尾而等待数十毫秒。

**覆盖机制**：对于实时控制数据（如电机占空比、舵机角度），如果新指令到来时旧请求还在排队，直接覆盖旧数据：

```c
void request_enqueue_urgent(BusId_t bus, uint8_t owner,
                             uint8_t *data, uint16_t len) {
    /* 查找该设备已有槽位，直接覆盖，不追加 */
    for (int i = 0; i < QUEUE_SLOTS; i++) {
        if (queues[bus].slots[i].owner_id == owner) {
            memcpy(queues[bus].slots[i].data, data, len);
            queues[bus].slots[i].pending = 1;
            return;
        }
    }
}
```

**设计原则**：过时的数据等同于错误数据，丢弃比发送更有价值。

### 3.3 三缓冲（Triple Buffering）隐藏拷贝延迟

利用 DMA 半完成中断预填充下一帧数据，将拷贝延迟隐藏在硬件传输间隙中：

```
Buffer A: DMA 正在发送
Buffer B: DMA 半完成 → 预填充下一帧（~200μs 窗口期）
Buffer C: 应用层写入新数据
```

DMA 半完成中断触发时，还有半个报文的时间窗口。在这段时间内 CPU 将 Buffer C 的数据拷贝到 Buffer B，配置好 DMA 指针。等完成中断触发时，直接切换指针到 Buffer B，立即启动下一帧，实现零等待背靠背发送。

### 3.4 中断链调度

将调度逻辑放在 DMA 完成 ISR 中，消除主循环轮询延迟：

```c
void HAL_I2C_MasterTxCpltCallback(I2C_HandleTypeDef *hi2c) {
    bus_tx_complete(hi2c);       /* 释放总线锁 */

    uint8_t next = request_dequeue(BUS_1);
    if (next != OWNER_NONE) {
        dispatch_transfer(next);  /* 立即启动下一帧 */
    }
}
```

当前帧刚结束，下一帧在中断返回前就启动了，背靠背传输，零额外延迟。

---

## 4. 带宽计算与吞吐量分析

### 4.1 乒乓缓冲的适用范围

乒乓缓冲解决的是**单设备**的数据安全问题——DMA 读前缓冲区时，CPU 写后缓冲区。对于 N 个设备共享总线，每个设备需要独立的缓冲区对，乒乓缓冲不解决总线带宽分配问题。

### 4.2 I2C 总线带宽计算

| 速率模式 | SCL 频率 | 每字节时间 | 理论吞吐 |
|----------|----------|------------|----------|
| 标准模式 | 100 kHz  | ~90-110μs  | ~9-11 KB/s |
| 快速模式 | 400 kHz  | ~22.5-27μs | ~37-44 KB/s |
| 快速+模式 | 1 MHz   | ~9-11μs    | ~90-110 KB/s |

每字节开销明细（以 100kHz 为例）：

```
每个 SCL 周期 = 10μs
每个字节      = 8 数据位 + 1 ACK 位 = 9 SCL = 90μs
起始条件 + 地址(7bit+W/R) + ACK ≈ 11 SCL ≈ 110μs
停止条件 ≈ 2 SCL ≈ 20μs

传输 N 字节的总时间:
T = 起始 + 地址 + N × 字节 + 停止
  = 110μs + N × 90μs + 20μs
```

### 4.3 多设备排队总时间

M 个设备，每个传输 L_i 字节：

```
T_total = Σ (130μs + L_i × 90μs)
```

### 4.4 带宽规划：速率 vs 数据量

**核心问题不是"能否在单次循环内传完"，而是"数据生成速率是否超过总线传输速率"：**

```
R_gen = Σ (各设备数据量 / 产生周期)
R_bus = I2C 有效吞吐量

若 R_gen ≤ R_bus → 系统稳定，队列不会无限增长
若 R_gen > R_bus → 队列不断增长，最终溢出丢数据
```

**工程判断步骤：**

1. 计算各设备单次传输时间：`T_i = 130μs + L_i × 90μs`
2. 计算所有设备轮转一次总时间：`T_total = Σ T_i`
3. 找到最短的设备更新周期：`T_min = min(T_period_i)`
4. 判断：`T_total ≤ T_min` 则系统稳定，否则需优化

**示例**：100kHz, 1 个设备每 500ms 发 1025 字节, 2 个设备每 100ms 发 2 字节

```
T_dev1 = 92.38ms  < 500ms  ✓
T_dev2 = 0.31ms   < 100ms  ✓
T_dev3 = 0.31ms   < 100ms  ✓
T_total = 93.0ms  < 100ms  ✓ (最紧约束)
```

### 4.5 中断链调度 vs 主循环调度

**主循环调度：** 每次迭代只处理一个请求，DMA 传输期间剩余循环空转。

```
0ms        2ms        4ms             94ms        96ms
[调度A→DMA]─[跳过]─[跳过]─...─[完成]─[调度B→DMA]─[完成]
            ↑ 41 次循环空转           ↑ 额外 ~2ms 轮询延迟
总耗时: 93ms + ~4ms 额外延迟 = 97ms
```

**中断链调度：** DMA 完成 ISR 中立即启动下一个，背靠背传输。

```
0ms        92ms       92.3ms     93ms
[调度A→DMA]─[完成→调度B]─[完成→调度C]─[完成]
总耗时: 93ms，零额外延迟
```

---

## 5. 裸机 vs RTOS 方案对比

### 5.1 RTOS 提供了什么

| 问题 | RTOS 方案 | 是否自动解决 |
|------|-----------|-------------|
| 队列数据结构 | `xQueueCreate` / `xQueueSend` | ✅ 内置 |
| 数据深拷贝 | `xQueueSend` 默认拷贝数据到队列内部 | ✅ 默认安全 |
| 总线互斥 | `xSemaphoreCreateMutex` | ✅ 提供互斥量，需手动加锁 |
| ISR 通知任务 | `xSemaphoreGiveFromISR` / `xTaskNotifyFromISR` | ✅ 提供机制，需自行设计 |
| 任务阻塞等待 | `xQueueReceive` 阻塞直到有数据 | ✅ 无数据时不占 CPU |

### 5.2 RTOS 没解决什么

**饥饿问题仍然存在**——只是从"主循环轮询"变成了"任务优先级调度"：

```c
/* 高优先级任务持续往队列塞数据 */
void vTaskHigh(void *pv) {
    while (1) {
        xQueueSend(i2c_q, &data, 0);
        vTaskDelay(10);
    }
}

/* 低优先级任务也能入队，但 I2C 总线一次只能处理一个传输 */
void vTaskLow(void *pv) {
    while (1) {
        xQueueSend(i2c_q, &data, 0);  /* 入队成功，但何时被处理？ */
        vTaskDelay(1000);
    }
}
```

`xQueueSend` 只是入队，不阻塞。**I2C 总线是串行物理瓶颈，RTOS 改变不了这一点**。无论用什么队列，同一时刻总线上只能有一个传输。

### 5.3 正确做法：I2C 管理任务

所有设备通过队列向一个专门的 I2C 管理任务提交请求：

```c
void vI2CManager(void *pv) {
    I2CRequest_t req;
    while (1) {
        /* 阻塞等待请求 — 无请求时不占 CPU */
        xQueueReceive(i2c_req_q, &req, portMAX_DELAY);

        /* 锁总线 */
        xSemaphoreTake(i2c_mutex, portMAX_DELAY);

        /* 启动 DMA */
        HAL_I2C_Master_Transmit_DMA(&hi2c1, req.addr, req.data, req.len);

        /* 阻塞等待 DMA 完成通知 */
        ulTaskNotifyTake(pdTRUE, portMAX_DELAY);

        /* 释放总线 */
        xSemaphoreGive(i2c_mutex);
    }
}
```

这里 `xQueueReceive` 和 `ulTaskNotifyTake` 都是**阻塞等待**，没有请求时任务挂起，零 CPU 占用——这是与裸机轮询的本质区别。

### 5.4 裸机 vs FreeRTOS 对比

| 方面 | 裸机 + 轮询 | FreeRTOS + I2C 管理任务 |
|------|-------------|------------------------|
| 无请求时 CPU 占用 | 每次循环空转一次函数调用 | 任务挂起，零 CPU 占用 |
| 队列实现 | 手写循环数组 | `xQueue`，自带阻塞/超时/ISR 安全 |
| 数据保护 | 需手动深拷贝或乒乓缓冲 | `xQueueSend` 默认深拷贝 |
| 总线互斥 | 手写 spinlock | `xSemaphore`，支持阻塞等待 |
| 优先级处理 | 循环指针，公平但无法紧急插队 | 管理任务可检查紧急队列或消息优先级 |
| 中断通知 | 直接在 ISR 中调度 | 通过 `FromISR` 函数通知任务 |
| 确定性 | 取决于主循环周期 | 取决于任务优先级和中断延迟 |

### 5.5 架构选择建议

| 项目复杂度 | 推荐方案 | 原因 |
|-----------|----------|------|
| 1-2 个设备，简单逻辑 | 裸机 + 轮询 | 无额外学习成本，代码量小 |
| 3+ 个设备，中等复杂度 | 裸机 + 请求队列 + 中断链 | 在裸机框架内解决问题 |
| 多任务，高复杂度 | FreeRTOS + I2C 管理任务 | 模块化，阻塞等待省 CPU |
| 硬实时控制 | 裸机 + 中断驱动 | RTOS 的任务切换抖动不可控 |

---

## 6. 防死锁机制

当 DMA 超时或从机拉低 SCL 导致总线锁死时，需要超时保护：

```c
void HAL_I2C_ErrorCallback(I2C_HandleTypeDef *hi2c) {
    bus_error_recovery(hi2c);      /* 释放总线锁 */
    request_clear(hi2c->bus_id);   /* 清空待处理请求 */

    /* 可选：I2C 外设软件复位 */
    /* HAL_I2C_DeInit(hi2c); */
    /* HAL_I2C_Init(hi2c); */
}
```

**必要措施**：
- 所有阻塞等待操作必须带超时（`timeout_ms > 0`）
- DMA 传输完成/错误回调中必须释放总线锁
- 错误回调中应清空队列，避免死锁请求被反复调度

---

## 7. 总结

| 场景 | 推荐方案 | 核心原则 |
|------|----------|----------|
| 通用多设备共享总线 | 请求队列 + 循环调度 | 防饥饿，公平轮转 |
| 数据在排队期间可能变化 | 乒乓缓冲 或 深拷贝入队 | 物理隔离，数据安全 |
| RAM 极度紧张 | 所有权转移 + 回调通知 | 零拷贝，契约保证 |
| 硬实时、短消息 | 放弃 DMA，轮询发送 | 确定性优于吞吐量 |
| 硬实时、长消息 | 优先级抢占 + 最新值覆盖 | 丢旧帧，不阻塞新帧 |
| 追求极致吞吐 | 三缓冲 + 中断链调度 | 隐藏拷贝延迟，背靠背 |
| 多设备背靠背传输 | 中断链调度 | 消除主循环轮询延迟 |
| 复杂系统 | RTOS + I2C 管理任务 | 阻塞等待，模块化设计 |

