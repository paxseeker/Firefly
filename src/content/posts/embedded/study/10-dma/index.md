---
title: " DMA"
tags:
  - 嵌入式
  - STM32
  - DMA
category: 嵌入式
published: 2026-05-29
draft: false
image: https://imgbed.paxseeker.xyz/file/1785240220953_image.png
---

# DMA

> [!info] 考虑一个问题，使用I2C与OLED通信，轮询阻塞方式，如果OLED没连接，就会等待超时时间才继续，如果初始化和超时时间过长，就会卡在初始化，后续如果有数据写入OLED,也会卡住，一般情况下如何解决
> 
> **解法1：超时 + 重试（最基础）**
> 获取返回状态，添加重试次数上限（如 10 次），每次超时后修改状态为为连接，直接跳过，超过10次后重新尝试。
> 
> **解法2：非阻塞方式（中断 / DMA）**
> 使用 `HAL_I2C_Master_Transmit_DMA` 或 `HAL_I2C_Master_Transmit_IT`，调用后立即返回，主循环继续运行。传输完成在回调中处理。这样即使 OLED 没连接，也不会阻塞主循环。
> 
> **解法3：先检测设备是否存在**
> 发送起始条件 + 从机地址，检测是否有 ACK 应答。无应答说明设备不在线，直接跳过初始化，不进入超时等待。
> 
> **解法4：状态机方式**
> 将 I2C 通信拆分为多个状态（IDLE → SEND_ADDR → SEND_DATA → WAIT_ACK → ...），在主循环中轮询执行，每帧只执行一个状态步骤。即使某步卡住，也不会阻塞整个系统。
> 
> **解法5：独立看门狗（IWDG）**
> 开启 IWDG，正常流程中定期喂狗。如果 I2C 卡死在超时等待中，看门狗超时后复位系统。这是最后一道防线。

> [!tip] 另一个问题：用 I2C 刷 OLED 屏幕（1024 字节帧缓冲），CPU 逐字节搬运效率低，如何加速？
> - DMA 就是答案——I2C 数据直接通过 DMA 从内存搬运到外设，CPU 只需在传输完成时处理一次。

## DMA 简介

**DMA（Direct Memory Access，直接存储器访问）** 是一种允许外设与内存之间直接传输数据的硬件机制，无需 CPU 逐字节搬运。

**没有 DMA 时（轮询/中断）**：
```
CPU 读 USART->DR → 存到 buf[0] → CPU 读 USART->DR → 存到 buf[1] → ...
每个字节都占用 CPU，大量数据时 CPU 被完全占满
```

**有 DMA 时**：
```
CPU 配置好 DMA 后就可以去干别的
DMA 自动把 USART->DR 的数据搬运到 buf
搬运完成后 DMA 触发中断通知 CPU 处理
```

**DMA 的适用场景**：
| 场景 | 为什么用 DMA |
|------|-------------|
| 串口收发大量数据 | 避免每字节中断，CPU 只在完成时处理 |
| ADC 多通道连续采集 | DMA 自动搬运结果到数组，CPU 定期取用 |
| 定时器自动更新 DAC | 定时器触发 DMA 搬运波形数据到 DAC |
| SPI 刷屏/读写 SD 卡 | 高速数据流，CPU 无需逐字节等待 |

---

## STM32F103C8T6 DMA 控制器架构

C8T6 只有 **DMA1**（7 个通道），每个通道对应一个中断向量，可处理一个独立的 DMA 传输请求。

### DMA1 通道与外设映射

| 通道 | 外设请求 1 | 外设请求 2 | 外设请求 3 | 外设请求 4 |
|------|-----------|-----------|-----------|-----------|
| **CH1** | ADC1 | TIM2_CH3 | TIM4_CH1 | — |
| **CH2** | USART1_TX | SPI1_RX | TIM1_CH1 | TIM2_CH2 |
| **CH3** | USART1_RX | SPI1_TX | TIM1_CH2 | TIM2_CH3 |
| **CH4** | USART2_TX | SPI2_RX | TIM1_CH3 | TIM2_CH4 |
| **CH5** | USART2_RX | SPI2_TX | TIM1_CH4 | TIM1_TRIG |
| **CH6** | USART3_TX | SPI3_RX | I2C1_TX | TIM1_CH1N |
| **CH7** | USART3_RX | SPI3_TX | I2C1_RX | TIM1_CH2N |

> 每个通道同时只能服务一个外设请求，通过 `DMA_CCRx` 的 `PL[1:0]` 和 `MSIZE/PSIZE` 等配置决定行为。

---

## DMA 关键寄存器一览

| 寄存器 | 全称 | 作用 |
|--------|------|------|
| **DMA_CCRx** | 通道配置寄存器 | 传输方向、优先级、数据宽度、循环模式、增量模式等 |
| **DMA_CNDTRx** | 通道传输数量寄存器 | 待传输的数据量（字节/半字/字），传输完成后自动减为 0 |
| **DMA_CPARx** | 通道外设地址寄存器 | 外设数据寄存器的地址（如 `&USART1->DR`） |
| **DMA_CMARx** | 通道内存地址寄存器 | 内存缓冲区地址 |
| **DMA_ISR** | 中断状态寄存器 | 记录各通道的传输完成/半完成/错误标志 |
| **DMA_IFCR** | 中断标志清除寄存器 | 写 1 清除对应通道的中断标志 |

**DMA_CCRx 关键位**：

| 位段 | 名称 | 说明 |
|------|------|------|
| `DIR` | 传输方向 | 0=外设→内存（读），1=内存→外设（写） |
| `CIRC` | 循环模式 | 1=循环传输，CNDTR 自动重载 |
| `PINC` | 外设地址增量 | 1=每次传输后外设地址递增 |
| `MINC` | 内存地址增量 | 1=每次传输后内存地址递增 |
| `PSIZE[1:0]` | 外设数据宽度 | 00=8bit, 01=16bit, 10=32bit |
| `MSIZE[1:0]` | 内存数据宽度 | 同上，通常与外设一致 |
| `PL[1:0]` | 通道优先级 | 00=低, 01=中, 10=高, 11=最高 |
| `TEIE` | 传输错误中断使能 | 1=传输错误时触发中断 |
| `TCIE` | 传输完成中断使能 | 1=传输完成时触发中断 |
| `EN` | 通道使能 | 1=启动 DMA 传输 |

---

## HAL DMA API

### 传输函数

```c
// 阻塞方式（轮询等待传输完成）
HAL_StatusTypeDef HAL_DMA_Start(DMA_HandleTypeDef *hdma,
    uint32_t SrcAddress, uint32_t DstAddress, uint32_t DataLength);

// 中断方式（传输完成后自动调用回调函数）
HAL_StatusTypeDef HAL_DMA_Start_IT(DMA_HandleTypeDef *hdma,
    uint32_t SrcAddress, uint32_t DstAddress, uint32_t DataLength);

// 停止 DMA 传输
HAL_StatusTypeDef HAL_DMA_Abort(DMA_HandleTypeDef *hdma);

// 轮询等待传输完成
HAL_StatusTypeDef HAL_DMA_PollForTransfer(DMA_HandleTypeDef *hdma,
    HAL_DMA_LevelCompleteTypeDef CompleteLevel, uint32_t Timeout);
```

>[!warning] `HAL_DMA_Start_IT` 的源地址和目标地址是**按字节**填写的，DMA 内部会根据 `MSIZE/PSIZE` 自动对齐。

### 回调函数

使用中断方式时，需要重写以下回调函数：

```c
void HAL_DMA_XferCpltCallback(DMA_HandleTypeDef *hdma);     // 传输完成
void HAL_DMA_XferHalfCpltCallback(DMA_HandleTypeDef *hdma); // 半传输完成
void HAL_DMA_XferErrorCallback(DMA_HandleTypeDef *hdma);    // 传输错误
void HAL_DMA_XferAbortCallback(DMA_HandleTypeDef *hdma);    // 传输中止
```

### 与外设 HAL 的结合（简化调用）

HAL 库为常用外设封装了 DMA 接口，内部自动调用 `HAL_DMA_Start_IT`，无需手动配置 DMA：

```c
// I2C DMA 收发（用于 OLED / 传感器等）
HAL_StatusTypeDef HAL_I2C_Master_Transmit_DMA(I2C_HandleTypeDef *hi2c,
    uint16_t DevAddress, const uint8_t *pData, uint16_t Size);
HAL_StatusTypeDef HAL_I2C_Master_Receive_DMA(I2C_HandleTypeDef *hi2c,
    uint16_t DevAddress, uint8_t *pData, uint16_t Size);
HAL_StatusTypeDef HAL_I2C_Mem_Write_DMA(I2C_HandleTypeDef *hi2c,
    uint16_t DevAddress, uint16_t MemAddress, uint16_t MemAddSize,
    const uint8_t *pData, uint16_t Size);
HAL_StatusTypeDef HAL_I2C_Mem_Read_DMA(I2C_HandleTypeDef *hi2c,
    uint16_t DevAddress, uint16_t MemAddress, uint16_t MemAddSize,
    uint8_t *pData, uint16_t Size);

// USART DMA 收发
HAL_StatusTypeDef HAL_UART_Transmit_DMA(UART_HandleTypeDef *huart,
    const uint8_t *pData, uint16_t Size);
HAL_StatusTypeDef HAL_UART_Receive_DMA(UART_HandleTypeDef *huart,
    uint8_t *pData, uint16_t Size);

// SPI DMA 收发
HAL_StatusTypeDef HAL_SPI_Transmit_DMA(SPI_HandleTypeDef *hspi,
    const uint8_t *pData, uint16_t Size);
HAL_StatusTypeDef HAL_SPI_Receive_DMA(SPI_HandleTypeDef *hspi,
    uint8_t *pData, uint16_t Size);

// ADC DMA 采集（连续模式）
HAL_StatusTypeDef HAL_ADC_Start_DMA(ADC_HandleTypeDef *hadc,
    uint32_t *pData, uint32_t Length);

// 定时器 PWM 脉冲 DMA（更新 CCR）
HAL_StatusTypeDef HAL_TIM_PWM_Start_DMA(TIM_HandleTypeDef *htim,
    uint32_t Channel, const uint32_t *pData, uint16_t Length);
```

> 这些外设级的 DMA 函数的回调函数名不同，注意区分：
> - I2C 完成回调：`HAL_I2C_MasterTxCpltCallback` / `HAL_I2C_MasterRxCpltCallback`
> - USART 完成回调：`HAL_UART_TxCpltCallback` / `HAL_UART_RxCpltCallback`
> - ADC 完成回调：`HAL_ADC_ConvCpltCallback`
> - DMA 通用回调：`HAL_DMA_XferCpltCallback`

---

## 实战：DMA + I2C 刷 OLED 屏幕

### 场景

0.96 寸 OLED（SSD1306）帧缓冲 1024 字节，每次 `OLED_Update()` 需逐字节通过 I2C 发送。轮询方式下 CPU 被完全占用，改用 DMA 后在后台传输，CPU 可同时处理其他任务。

### 对比

```
轮询方式（无 DMA）：
CPU 逐字节发送 → 每字节等待 I2C 完成 → CPU 空转
1024 字节 × 每个字节约 10μs ≈ 10ms 阻塞

DMA 方式：
CPU 启动 DMA 传输 → 立刻返回继续执行其他任务
DMA 在后台上传 1024 字节 → 完成时中断通知 CPU
```

### CubeMX 配置

```
I2C1：
  - I2C Speed Mode: Standard (100kHz) 或 Fast (400kHz)
  - 使用 PB8(SCL), PB9(SDA)  —— 与 IIC 笔记一致

DMA1 配置：
  - I2C1_TX → DMA1_CH6（I2C 写 OLED 只需 TX）
  - Mode: Normal（非循环）
  - Data Width: Byte
  - Memory Increment: Enable
  - Priority: Medium

NVIC：
  - 使能 DMA1_Channel6_IRQn
  - 使能 I2C1 事件中断（EV）和错误中断（ER）
```

### 生成的 MSP 初始化代码 + 中断处理函数

```c
// stm32f1xx_hal_msp.c
void HAL_I2C_MspInit(I2C_HandleTypeDef* hi2c)
{
    if (hi2c->Instance == I2C1) {
        // 时钟使能
        __HAL_RCC_I2C1_CLK_ENABLE();
        __HAL_RCC_GPIOB_CLK_ENABLE();
        __HAL_RCC_DMA1_CLK_ENABLE();

        // GPIO 配置 PB8(SCL), PB9(SDA) 为复用开漏
        GPIO_InitStruct.Pin = GPIO_PIN_8 | GPIO_PIN_9;
        GPIO_InitStruct.Mode = GPIO_MODE_AF_OD;
        GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
        HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);

        // DMA 配置（I2C1_TX → CH6）
        static DMA_HandleTypeDef hdma_i2c1_tx;
        hdma_i2c1_tx.Instance = DMA1_Channel6;
        hdma_i2c1_tx.Init.Direction = DMA_MEMORY_TO_PERIPH;
        hdma_i2c1_tx.Init.PeriphInc = DMA_PINC_DISABLE;
        hdma_i2c1_tx.Init.MemInc = DMA_MINC_ENABLE;
        hdma_i2c1_tx.Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE;
        hdma_i2c1_tx.Init.MemDataAlignment = DMA_MDATAALIGN_BYTE;
        hdma_i2c1_tx.Init.Mode = DMA_NORMAL;
        hdma_i2c1_tx.Init.Priority = DMA_PRIORITY_MEDIUM;
        HAL_DMA_Init(&hdma_i2c1_tx);
        __HAL_LINKDMA(hi2c, hdmatx, hdma_i2c1_tx);

        // NVIC 中断（DMA + I2C 都必须使能）
        HAL_NVIC_SetPriority(DMA1_Channel6_IRQn, 0, 0);
        HAL_NVIC_EnableIRQ(DMA1_Channel6_IRQn);
        HAL_NVIC_SetPriority(I2C1_EV_IRQn, 0, 0);
        HAL_NVIC_EnableIRQ(I2C1_EV_IRQn);
        HAL_NVIC_SetPriority(I2C1_ER_IRQn, 0, 0);
        HAL_NVIC_EnableIRQ(I2C1_ER_IRQn);
    }
}
```

```c
// stm32f1xx_it.c — 必须添加 I2C 中断处理函数！
// 否则 I2C 状态机卡死，DMA 传输永不完成
void I2C1_EV_IRQHandler(void) {
    HAL_I2C_EV_IRQHandler(&hi2c1);
}
void I2C1_ER_IRQHandler(void) {
    HAL_I2C_ER_IRQHandler(&hi2c1);
}
void DMA1_Channel6_IRQHandler(void) {
    HAL_DMA_IRQHandler(&hdma_i2c1_tx);
}
```

> [!warning] **关键**：`HAL_I2C_Master_Transmit_DMA` 内部依赖 I2C 事件中断管理 START/地址/停止阶段。**只使能 DMA 中断是不够的**，I2C1_EV 和 I2C1_ER 的中断处理函数和 NVIC 使能缺一不可。

### 改造 OLED 驱动支持 DMA

在 `oled.c` 中新增 DMA 刷屏函数，与原来的轮询方式并存：

```c
// ========== oled.h 新增 ==========
#define OLED_DMA_BUF_SIZE  (OLED_BUFFER_SIZE + 1)  // 1 字节控制头 + 1024 字节数据

HAL_StatusTypeDef OLED_Update_DMA(OLED_HandleTypeDef *oled);
extern volatile uint8_t oled_dma_busy;  // 用户可查这个标志判断是否忙

// ========== oled.c 新增 ==========
volatile uint8_t oled_dma_busy = 0;
static uint8_t dma_tx_buf[OLED_DMA_BUF_SIZE];

// 用 DMA 方式将整帧缓冲区发送到 SSD1306（一次性 1025 字节）
HAL_StatusTypeDef OLED_Update_DMA(OLED_HandleTypeDef *oled) {
    if (oled_dma_busy) return HAL_BUSY;

    dma_tx_buf[0] = 0x40;                          // 控制字节：D/C=数据模式，Co=0
    memcpy(dma_tx_buf + 1, oled->buffer, OLED_BUFFER_SIZE);  // 帧数据

    oled_dma_busy = 1;
    return HAL_I2C_Master_Transmit_DMA(oled->hi2c,
        OLED_I2C_ADDR, dma_tx_buf, OLED_DMA_BUF_SIZE);
}

// 在 main.c 或 oled.c 中实现回调
void HAL_I2C_MasterTxCpltCallback(I2C_HandleTypeDef *hi2c) {
    if (hi2c == &hi2c1) {
        oled_dma_busy = 0;  // 传输完成，释放锁
    }
}
void HAL_I2C_ErrorCallback(I2C_HandleTypeDef *hi2c) {
    if (hi2c == &hi2c1) {
        oled_dma_busy = 0;  // 出错也要释放锁，否则下次刷屏永远 HAL_BUSY
    }
}
```

> ⚠️ **SSD1306 控制字节说明**：I2C 协议中，从机地址后的第一个字节是控制字节。`0x40` 表示 Co=0（无后续控制字节）、D/C#=1（数据模式）。**Co=0 后所有后续字节都视为数据**，不需要也不应该在每字节前重复插入 `0x40`。交织插入 `0x40` 会导致数据错乱（0x40 被当作数据写入 GDDRAM）。

### 修改 SSD1306 寻址模式

DMA 一次性发送整帧的前提是 SSD1306 处于**水平寻址模式**（Horizontal Addressing Mode），否则写完一页（128 字节）后列地址回到 0、页不递增，后续数据全部覆盖同一页：

```c
// OLED_Init 中：
OLED_Cmd(oled, 0x20);
OLED_Cmd(oled, 0x00);  // 0x00 = 水平寻址模式（原 0x02 = 页寻址模式 ❌）
```

| 模式 | 命令值 | DMA 整帧传输 |
|------|--------|-------------|
| 水平寻址（Horizontal） | `0x00` | ✅ 列→127 后自动进下一页 |
| 页寻址（Page） | `0x02` | ❌ 列循环但页不变，仅第 0 页有内容 |
| 垂直寻址（Vertical） | `0x01` | ✅ 行→63 后自动进下一列 |

### 安全初始化：先检测设备是否存在

`OLED_Init` 发送约 20 条命令，每条通过轮询 `HAL_I2C_Master_Transmit`（默认超时 100ms）。若 OLED 未连接，20 × 100ms = **2 秒阻塞**。

解法：在 `OLED_Init` 前用 `HAL_I2C_IsDeviceReady` 快速检测，无应答则跳过，下次再试：

```c
void display_tick(display_t* d) {
    if (!d->dirty && d->ready) return;
    if (!fsm_timer_expired(&d->timer)) return;

    if (!d->ready) {
        // 先检测设备是否存在，避免 OLED_Init 阻塞
        if (HAL_I2C_IsDeviceReady(d->oled->hi2c, OLED_I2C_ADDR, 2, 10) == HAL_OK) {
            if (OLED_Init(d->oled) == HAL_OK) {
                d->ready = 1;
                d->dirty = 1;
            }
        }
        // 无论是否检测到，都等 500ms 再试
        fsm_timer_init(&d->timer, 500);
        return;
    }
    // ... 正常刷屏 ...
}
```

`HAL_I2C_IsDeviceReady` 发送 START + 从机地址，等待 ACK。无 ACK 则在 2 × 10ms = 20ms 内返回 `HAL_TIMEOUT`，**不会阻塞系统**。

### 使用示例

```c
/* USER CODE BEGIN PV */
OLED_HandleTypeDef oled;
uint32_t last_update = 0;
uint32_t counter = 0;
/* USER CODE END PV */

int main(void) {
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_DMA_Init();
    MX_I2C1_Init();

    /* USER CODE BEGIN 2 */
    oled.hi2c = &hi2c1;
    // 安全初始化：先检测设备是否存在
    if (HAL_I2C_IsDeviceReady(oled.hi2c, OLED_I2C_ADDR, 2, 10) == HAL_OK) {
        OLED_Init(&oled);
    }
    OLED_ShowString(&oled, 0, 0, "DMA OLED Demo");
    OLED_Update_DMA(&oled);  // 第一次刷屏用 DMA
    /* USER CODE END 2 */

    while (1) {
        // CPU 可以在 DMA 后台刷屏时干其他事
        if (HAL_GetTick() - last_update >= 100) {  // 每 100ms 更新一次
            last_update = HAL_GetTick();

            // 修改缓冲区内容
            OLED_Clear(&oled);
            OLED_ShowString(&oled, 0, 0, "DMA OLED Demo");
            counter++;
            OLED_ShowChar(&oled, 0, 20, (counter / 100) % 10 + '0');
            OLED_ShowChar(&oled, 8, 20, (counter / 10) % 10 + '0');
            OLED_ShowChar(&oled, 16, 20, counter % 10 + '0');

            // 如果上次 DMA 传输已完成，启动新一轮
            if (!oled_dma_busy) {
                OLED_Update_DMA(&oled);
            }
            // 如果还忙（比如 100ms 内 1025 字节没传完），跳过本次更新
        }

        // 其他任务可以继续执行，不会被 I2C 刷屏阻塞
        HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_13);  // LED 闪烁不受影响
        HAL_Delay(500);
    }
}
```

### DMA 传输流程时序

```
OLED_Update_DMA(&oled)
    │
    ├── 将 buffer 数据复制到 dma_tx_buf（第 0 字节 = 0x40 控制字节）
    ├── 调用 HAL_I2C_Master_Transmit_DMA
    │       │
    │       ├── 配置 DMA1_CH6：源=dma_tx_buf，目标=I2C1->DR，大小=1025
    │       ├── 使能 DMA 通道（CCR6.EN = 1）
    │       └── 使能 I2C1 的 DMA 发送请求（CR2[10] = DMAEN = 1）
    │
    ├── HAL_I2C_Master_Transmit_DMA 内部：
    │       │
    │       ├── 生成起始条件
    │       ├── 发送从机地址 + 写位
    │       ├── 等待 ACK
    │       ├── 之后每个字节自动由 DMA 搬运到 DR
    │       └── DMA 传输完成后自动生成停止条件
    │
    ├── CPU 立即返回，继续执行主循环
    │
    ├── DMA 后台传输：
    │       │
    │       ├── DMA 自动搬运 dma_tx_buf[0] = 0x40 控制字节 → I2C1->DR
    │       │   SSD1306 解析：Co=0, D/C#=1 → 后续字节均为数据
    │       ├── DMA 自动搬运 dma_tx_buf[1..1024] = 1024 字节帧数据 → I2C1->DR
    │       │   水平寻址模式：列 0→127 自动递增，页 0→7 自动递增
    │       └── CNDTR = 0 → 触发 DMA 传输完成中断
    │
    └── 回调 HAL_I2C_MasterTxCpltCallback
            │
            └── oled_dma_busy = 0  // 释放锁，主循环可发起下一次传输
```

---

## 常见问题与注意事项

### 1. 数据宽度不一致导致数据错位

```c
// 错误：ADC 12 位（半字）但配置为字节
hdma.Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE;  // ❌
hdma.Init.MemDataAlignment = DMA_MDATAALIGN_BYTE;     // ❌

// 正确：ADC 12 位 → 半字
hdma.Init.PeriphDataAlignment = DMA_PDATAALIGN_HALFWORD;  // ✅
hdma.Init.MemDataAlignment = DMA_MDATAALIGN_HALFWORD;     // ✅
```

### 2. `__HAL_LINKDMA` 的作用

```c
#define __HAL_LINKDMA(__HANDLE__, __PPP_DMA_FIELD__, __DMA_HANDLE__) \
    do{                                                              \
        (__HANDLE__)->__PPP_DMA_FIELD__ = &(__DMA_HANDLE__);         \
        (__DMA_HANDLE__).Parent = (__HANDLE__);                      \
    } while(0U)
```

这个宏做了两件事，建立**双向关联**：

**① 外设 → DMA**：`hi2c1.hdmatx = &hdma_i2c1_tx`
外设句柄的 DMA 字段指向 DMA 句柄，这样 `HAL_I2C_Master_Transmit_DMA` 内部才能找到 DMA 通道并启动传输。

**② DMA → 外设**：`hdma_i2c1_tx.Parent = &hi2c1`
DMA 句柄的 `Parent` 指回外设句柄。当 DMA 传输完成触发中断时，完整的回调链是：

```
DMA1_Channel6_IRQHandler
  → HAL_DMA_IRQHandler(&hdma_i2c1_tx)                     // ① DMA 中断
    → hdma_i2c1_tx.XferCpltCallback(hdma_i2c1_tx)         // ② 调用注册的回调
      = I2C_DMAXferCplt                                    //    HAL 内部函数
        → hdma->Parent（即 &hi2c1）                         //    通过 Parent 找到 I2C
        → SET_BIT(CR1, I2C_CR1_STOP)                      //    ③ 生成 STOP 条件
        → hi2c->State = HAL_I2C_STATE_READY
        → HAL_I2C_MasterTxCpltCallback(hi2c)              // ④ 直接调用用户回调！
```

**`XferCpltCallback` 什么时候赋值的？** — 在 `HAL_I2C_Master_Transmit_DMA` 内部：

```c
// stm32f1xx_hal_i2c.c 第 2080 行
hi2c->hdmatx->XferCpltCallback = I2C_DMAXferCplt;  // 注册 HAL 内部函数
hi2c->hdmatx->XferErrorCallback = I2C_DMAError;
HAL_DMA_Start_IT(hi2c->hdmatx, ...);               // 启动 DMA
```

赋值的是 HAL 内部静态函数 `I2C_DMAXferCplt`，不是用户函数。用户只需要实现外设层回调 `HAL_I2C_MasterTxCpltCallback`。

**I2C EV 中断的作用是什么？** — 负责 DMA 传输**之前**的 START + 地址阶段。I2C 外设需要通过 EV 中断来驱动状态机完成起始条件、发送从机地址、等待 ACK 等步骤，完成后才启动 DMA 搬运数据。没有 EV 中断，I2C 连地址都发不出去，DMA 传输永远不会开始。

所以两个中断的分工不同：
- **I2C EV 中断**：START + 地址阶段（DMA 启动前，必须）
- **DMA 中断**：数据搬运完成 → 内部 `I2C_DMAXferCplt` → 生成 STOP + 直接调用用户回调

**没有 `__HAL_LINKDMA` 会怎样？**
- `HAL_I2C_Master_Transmit_DMA` 找不到 DMA 通道，返回 `HAL_ERROR`
- 即使 DMA 中断能触发，`HAL_DMA_IRQHandler` 也不知道通知哪个外设，用户层回调永不被调用

### 3. Normal 模式传输完成后需要重新启动

```c
// 每次 DMA 传输完成后必须再次调用才能接收下一帧
HAL_I2C_Master_Transmit_DMA(&hi2c1, addr, buf, len);
```

如果需要持续传输，回调中重新启动，或使用 **Circular 模式**。

### 4. I2C DMA 需要同时使能 DMA 和 I2C 中断

使用 `HAL_I2C_Master_Transmit_DMA` 时，**只使能 DMA 中断是不够的**。HAL 库的 I2C 驱动依赖 I2C 事件中断来管理 START/地址/停止阶段，依赖错误中断处理异常：

```c
// 以下三个 NVIC 和中断处理函数缺一不可
HAL_NVIC_SetPriority(DMA1_Channel6_IRQn, 0, 0);
HAL_NVIC_EnableIRQ(DMA1_Channel6_IRQn);
HAL_NVIC_SetPriority(I2C1_EV_IRQn, 0, 0);
HAL_NVIC_EnableIRQ(I2C1_EV_IRQn);
HAL_NVIC_SetPriority(I2C1_ER_IRQn, 0, 0);
HAL_NVIC_EnableIRQ(I2C1_ER_IRQn);

// stm32f1xx_it.c 中必须实现：
void I2C1_EV_IRQHandler(void) { HAL_I2C_EV_IRQHandler(&hi2c1); }
void I2C1_ER_IRQHandler(void) { HAL_I2C_ER_IRQHandler(&hi2c1); }
void DMA1_Channel6_IRQHandler(void) { HAL_DMA_IRQHandler(&hdma_i2c1_tx); }
```

缺少 I2C 中断处理函数时，I2C 状态机卡死在某一步，DMA 传输永不完成，`oled_dma_busy` 永远为 1。

### 5. SSD1306 寻址模式与 DMA 的兼容性

SSD1306 支持三种寻址模式，通过 `0x20` 命令设置：

| 模式 | 命令值 | 列递增行为 | 页递增行为 | 适用于 DMA 整帧 |
|------|--------|-----------|-----------|:--------------:|
| 水平寻址（Horizontal） | `0x00` | 0→127 自动递增 | 列到 127 后自动+1 | ✅ |
| 垂直寻址（Vertical） | `0x01` | 行到 63 后自动+1 | 0→7 自动递增 | ✅ |
| 页寻址（Page） | `0x02` | 0→127 循环 | **不自动递增** | ❌ |

**页寻址模式**下 DMA 发送 1024 字节，所有数据都写入第 0 页（覆盖写 8 次），屏幕上只有第一行有内容，其余 7 行空白。必须改为**水平寻址模式**才能让 DMA 一次性填满全屏。

### 6. SSD1306 控制字节

SSD1306 的 I2C 协议中，从机地址后的第一个字节是**控制字节**：

- `0x00` = Co=0, D/C#=0 → 后续字节均为命令
- `0x40` = Co=0, D/C#=1 → 后续字节均为数据

**Co=0 意味着"没有后续控制字节"**，后续所有字节都按 D/C# 指定的类型处理。**不需要**在每字节前重复插入 `0x40`。错误做法：

```c
// ❌ 错误：交织插入 0x40
dma_tx_buf = [0x40, d0, 0x40, d1, 0x40, d2, ...]
// SSD1306 实际写入 GDDRAM：d0, 0x40, d1, 0x40, d2, ...（0x40 被当作数据！）

// ✅ 正确：单次控制字节 + 全部数据
dma_tx_buf = [0x40, d0, d1, d2, ..., d1023]
// SSD1306 实际写入 GDDRAM：d0, d1, d2, ..., d1023（全部正确）
```

### 7. 缓冲区对齐

某些 DMA 控制器要求缓冲区地址对齐到数据宽度（如半字对齐到 2 字节，字对齐到 4 字节）。STM32F103 的 DMA 没有强制对齐要求，但建议对齐以避免潜在问题。

### 8. 多通道共享同一 DMA 的优先级

### 9. 数据传输方向

| 方向 | SrcAddress | DstAddress | 典型场景 |
|------|-----------|-----------|----------|
| 外设→内存 | 外设地址 | 内存地址 | ADC 采集、I2C 接收 |
| 内存→外设 | 内存地址 | 外设地址 | I2C 刷屏、USART 发送、DAC 输出 |
| 内存→内存 | 内存地址 | 内存地址 | 快速拷贝（需 MEM2MEM 位） |

---

## 总结：DMA 使用三步曲

```
1. 初始化 DMA 通道（配置方向、宽度、增量、优先级）
   └── __HAL_LINKDMA 链接到外设句柄

2. 启动传输
   ├── 手动调用 HAL_DMA_Start / HAL_DMA_Start_IT
   └── 或调用外设封装函数（HAL_UART_Transmit_DMA 等）

3. 处理完成
   ├── Normal 模式：需要重新启动
   └── Circular 模式：回调中取数据，DMA 自动循环
```

> [!tip] 对比轮询、中断、DMA 三种方式
>
> | 方式 | CPU 占用 | 实时性 | 适合场景 |
> |------|---------|--------|---------|
> | 轮询 | 100% 阻塞 | 差 | 简单调试、低频操作 |
> | 中断 | 仅每字节中断 | 好 | 少量数据、要求及时响应 |
> | DMA | 仅完成时中断 | 最好 | 大量数据、持续采集 |