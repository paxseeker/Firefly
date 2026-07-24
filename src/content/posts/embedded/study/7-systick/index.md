---
title: Systick--状态机控制两个LED闪烁 + OLED + 串口通信
description: STM32 系统嘀嗒定时器，状态机控制LED，OLED显示状态，串口通信
tags:
  - 嵌入式
  - 定时器
  - 状态机
category: 嵌入式
draft: false
published: 2026-05-26
image: https://imgbed.paxseeker.xyz/file/1784869288852_【哲风壁纸】云层倒影-天空之境.png
---

# Systick -- 状态机控制两个LED闪烁 + OLED + 串口通信

## HAL 库 SysTick 初始化流程

```
1. HAL_Init();                     // 内部调 HAL_SYSTICK_Config(72)
2. HAL_RCC_OscConfig(...)          // 配置振荡器
3. HAL_RCC_ClockConfig(...)        // 配置总线时钟（72MHz）
4. HAL_InitTick(...)               // 重新配置 SysTick（按新频率）
```

> [!tip] 经过了两次 SysTick 配置
>
> 第一次（HAL_Init 内）：系统时钟还没配置，用默认 HSI（8MHz）
>
> 第二次（HAL_RCC_ClockConfig 后）：系统时钟已配置为 72MHz，重新设置 SysTick

### HAL_InitTick
```c
__weak HAL_StatusTypeDef HAL_InitTick(uint32_t TickPriority)
{
  if (HAL_SYSTICK_Config(SystemCoreClock / (1000U / uwTickFreq)) > 0U)
    return HAL_ERROR;

  if (TickPriority < (1UL << __NVIC_PRIO_BITS)) {
    HAL_NVIC_SetPriority(SysTick_IRQn, TickPriority, 0U);
    uwTickPrio = TickPriority;
  } else {
    return HAL_ERROR;
  }
  return HAL_OK;
}
```

### SysTick_Config
```c
__STATIC_INLINE uint32_t SysTick_Config(uint32_t ticks)
{
  if ((ticks - 1UL) > SysTick_LOAD_RELOAD_Msk)
    return (1UL);

  SysTick->LOAD  = (uint32_t)(ticks - 1UL);
  NVIC_SetPriority(SysTick_IRQn, (1UL << __NVIC_PRIO_BITS) - 1UL);
  SysTick->VAL   = 0UL;
  SysTick->CTRL  = SysTick_CTRL_CLKSOURCE_Msk |
                   SysTick_CTRL_TICKINT_Msk   |
                   SysTick_CTRL_ENABLE_Msk;
  return (0UL);
}
```

---

## LED 状态机设计

### led.h
```c
#ifndef LED_H
#define LED_H

#include "main.h"
#include <stdint.h>

typedef enum {
    LED_STATE_OFF = 0,
    LED_STATE_ON  = 1
} led_state_t;

typedef struct {
    GPIO_TypeDef* gpiox;
    uint16_t pin;
} led_hw_t;

typedef struct {
    led_state_t state;
    uint32_t start_tick;
    uint32_t period_on;
    uint32_t period_off;
    led_hw_t* hw;
} led_fsm_t;

void led_fsm_init(led_fsm_t* led, led_hw_t* hw, led_state_t init_state,
                  uint32_t period_on, uint32_t period_off);
void led_fsm_tick(led_fsm_t* led);
led_state_t led_get_state(const led_fsm_t* led);
void led_set_force(led_fsm_t* led, led_state_t state);

#endif
```

### led.c
```c
#include "led.h"

static void led_set_pin(led_hw_t* hw, uint8_t level) {
    HAL_GPIO_WritePin(hw->gpiox, hw->pin, level);
}

void led_fsm_init(led_fsm_t* led, led_hw_t* hw, led_state_t init_state,
                  uint32_t period_on, uint32_t period_off) {
    led->state = init_state;
    led->start_tick = HAL_GetTick();
    led->period_on = period_on;
    led->period_off = period_off;
    led->hw = hw;
    led_set_pin(hw, init_state == LED_STATE_ON ? GPIO_PIN_SET : GPIO_PIN_RESET);
}

void led_fsm_tick(led_fsm_t* led) {
    if (!led || !led->hw) return;

    uint32_t now = HAL_GetTick();
    uint32_t period = (led->state == LED_STATE_ON) ? led->period_on : led->period_off;
    uint32_t elapsed = now - led->start_tick;

    if (elapsed >= period) {
        led->start_tick = now;
        if (led->state == LED_STATE_ON) {
            led_set_pin(led->hw, GPIO_PIN_RESET);
            led->state = LED_STATE_OFF;
        } else {
            led_set_pin(led->hw, GPIO_PIN_SET);
            led->state = LED_STATE_ON;
        }
    }
}

led_state_t led_get_state(const led_fsm_t* led) {
    return led ? led->state : LED_STATE_OFF;
}

void led_set_force(led_fsm_t* led, led_state_t state) {
    if (!led || !led->hw) return;
    led_set_pin(led->hw, state == LED_STATE_ON ? GPIO_PIN_SET : GPIO_PIN_RESET);
    led->state = state;
    led->start_tick = HAL_GetTick();
}
```

### 状态机设计要点

1. **状态显式定义**：用枚举 `LED_STATE_OFF/LED_STATE_ON`，不用 bool
2. **周期检测用差值**：`elapsed = now - start_tick`，自然处理 uint32_t 溢出
3. **状态机不依赖调用频率**：tick 只判断时间差，不管多久调一次
4. **接口封装**：内部状态通过 `led_get_state()` 暴露，不直接访问 `.state`
5. **强制接口**：`led_set_force()` 允许中断/外部代码强制改变状态

### main.c
```c
int main(void) {
    HAL_Init();
    SystemClock_Config();

    MX_GPIO_Init();
    MX_USART1_UART_Init();
    MX_I2C1_Init();
    oled.hi2c = &hi2c1;
    OLED_Init(&oled);
    OLED_ShowString(&oled, 0, 0, "Systick Demo");
    OLED_ShowString(&oled, 0, 10, "Blue:");
    OLED_ShowString(&oled, 0, 20, "Green:");
    OLED_Update(&oled);

    HAL_UART_Receive_IT(&huart1, (uint8_t *)&receive_data, 1);
    led_fsm_init(&led_blue_fsm, &led_blue_hw, LED_STATE_OFF, 400, 400);
    led_fsm_init(&led_green_fsm, &led_green_hw, LED_STATE_OFF, 2000, 2000);

    while (1) {
        led_fsm_tick(&led_blue_fsm);
        led_fsm_tick(&led_green_fsm);
        // OLED 状态刷新
    }
}
```

