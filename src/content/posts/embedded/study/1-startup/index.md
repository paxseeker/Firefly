---
tags:
  - 嵌入式
title: 启动文件（stm32c8t6点灯）
category: 嵌入式
draft: false
published: 2026-05-21
description: 简单启动文件，点灯
---
# 启动文件（stm32c8t6点灯）

- startup.s
```asm
.syntax unified
.cpu cortex-m3
.thumb

.global _estack

/* ========= 中断向量表（8字节对齐） ========= */
.section .isr_vector, "a"
.align 3

.long _estack               /* 0. 栈顶指针（由链接脚本提供） */
.long Reset_Handler         /* 1. 复位中断入口 */
.long Default_Handler       /* 2. NMI */
.long Default_Handler       /* 3. 硬错误(HardFault) */
.long Default_Handler       /* 4. 内存管理错误 */
.long Default_Handler       /* 5. 总线错误 */
.long Default_Handler       /* 6. 使用错误 */
.long 0                     /* 7. 保留 */
.long 0                     /* 8. 保留 */
.long 0                     /* 9. 保留 */
.long 0                     /* 10. 保留 */
.long Default_Handler       /* 11. SVCall */
.long Default_Handler       /* 12. 调试监控 */
.long 0                     /* 13. 保留 */
.long Default_Handler       /* 14. PendSV */
.long Default_Handler       /* 15. SysTick */

/* ========= 默认中断处理（死循环） ========= */
.section .text.Default_Handler, "ax"
.thumb_func
Default_Handler:
    b .

/* ========= 复位中断处理 ========= */
.section .text.Reset_Handler, "ax"
.thumb_func
.global Reset_Handler

Reset_Handler:
    /* 1. 从Flash复制 .data 段到RAM（初始化带初值的全局变量） */
    LDR     R0, =_sdata
    LDR     R1, =_edata
    LDR     R2, =_sidata
    CMP     R0, R1
    BEQ     skip_data_copy
loop_data:
    LDR     R3, [R2]
    STR     R3, [R0]
    ADDS    R0, R0, #4
    ADDS    R2, R2, #4
    CMP     R0, R1
    BCC     loop_data
skip_data_copy:

    /* 2. 清空 .bss 段（未初始化的全局变量置零） */
    LDR     R0, =_sbss
    LDR     R1, =_ebss
    MOVS    R2, #0
    CMP     R0, R1
    BEQ     skip_bss_zero
loop_bss:
    STR     R2, [R0]
    ADDS    R0, R0, #4
    CMP     R0, R1
    BCC     loop_bss
skip_bss_zero:

    /* 3. 跳转到用户 main() */
    bl      main

    /* 4. 如果 main 意外返回，则死循环 */
    b .
```

- STM32F103C8T6_FLASH.ld
```asm
/* STM32F103C8T6 内存布局: Flash 64KB, RAM 20KB */
ENTRY(Reset_Handler)

MEMORY
{
    FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 64K
    RAM   (rwx) : ORIGIN = 0x20000000, LENGTH = 20K
}

/* 栈顶指向RAM末尾（此处唯一管理栈地址） */
_estack = ORIGIN(RAM) + LENGTH(RAM);

SECTIONS
{
    /* 中断向量表必须放在Flash最开头，强制8字节对齐 */
    .isr_vector : {
        . = ALIGN(8);
        KEEP(*(.isr_vector))
    } > FLASH

    /* 代码和只读数据 */
    .text : {
        *(.text*)
        *(.rodata*)
    } > FLASH

    /* .data 段：带初值的全局变量，存于Flash，运行时复制到RAM */
    _sidata = LOADADDR(.data);
    .data : {
        . = ALIGN(4);
        _sdata = .;
        *(.data*)
        _edata = .;
    } > RAM AT> FLASH

    /* .bss 段：未初始化的全局变量，在RAM中清零 */
    .bss : {
        . = ALIGN(4);
        _sbss = .;
        *(.bss*)
        *(COMMON)
        _ebss = .;
    } > RAM
}
```
## 启动文件详解

### 启动文件（startup.s）的作用

启动文件是嵌入式系统启动过程中的第一个被执行的代码，负责：
1. **初始化硬件**：设置堆栈指针、配置中断向量表
2. **内存初始化**：将Flash中的.data段复制到RAM，清空.bss段
3. **跳转到主程序**：最终调用用户的main()函数

### 启动文件关键组件

#### 1. 汇编语法设置
```asm
.syntax unified      ; 使用统一语法（推荐）
.cpu cortex-m3       ; 指定目标CPU为Cortex-M3
.thumb              ; 使用Thumb指令集（16位指令，更高效）
```

#### 2. 符号声明
```asm
.global _estack      ; 声明_estack为全局符号，可被其他文件引用
```

#### 3. 中断向量表（.isr_vector段）
- **.section .isr_vector, "a"**: 定义.isr_vector段，属性为"a"（可分配）
- **.align 3**: 8字节对齐（2^3 = 8），满足ARM中断向量表要求
- **.long address**: 32位地址（4字节），每个条目对应一个中断或异常

#### 4. 默认中断处理（Default_Handler）
```asm
.section .text.Default_Handler, "ax"  ; 定义.text.Default_Handler段
.thumb_func                          ; 声明为Thumb函数
Default_Handler:
    b .                              ; 无限循环，防止程序继续执行
```

#### 5. 复位处理程序（Reset_Handler）
```asm
.section .text.Reset_Handler, "ax"     ; 定义.text.Reset_Handler段
.thumb_func                          ; 声明为Thumb函数
.global Reset_Handler                ; 声明为全局符号

Reset_Handler:
    /* 1. 从Flash复制.data段到RAM */
    LDR     R0, =_sdata              ; 加载.data段RAM起始地址到R0
    LDR     R1, =_edata              ; 加载.data段RAM结束地址到R1
    LDR     R2, =_sidata             ; 加载.data段Flash起始地址到R2
    CMP     R0, R1                   ; 比较R0和R1
    BEQ     skip_data_copy           ; 如果相等，跳过复制
    
loop_data:
    LDR     R3, [R2]                ; 从Flash加载4字节到R3
    STR     R3, [R0]                ; 存储到RAM
    ADDS    R0, R0, #4              ; RAM地址递增
    ADDS    R2, R2, #4              ; Flash地址递增
    CMP     R0, R1                   ; 比较当前地址和结束地址
    BCC     loop_data               ; 如果R0 < R1，继续循环
    
skip_data_copy:

    /* 2. 清空.bss段（未初始化的全局变量置零） */
    LDR     R0, =_sbss               ; 加载.bss段RAM起始地址到R0
    LDR     R1, =_ebss               ; 加载.bss段RAM结束地址到R1
    MOVS    R2, #0                   ; R2 = 0（要存储的值）
    CMP     R0, R1                   ; 比较起始和结束地址
    BEQ     skip_bss_zero            ; 如果相等，跳过清零
    
loop_bss:
    STR     R2, [R0]                ; 存储R2（0）到RAM地址R0
    ADDS    R0, R0, #4              ; 地址递增
    CMP     R0, R1                   ; 检查是否到达结束地址
    BCC     loop_bss                ; 如果R0 < R1，继续循环
    
skip_bss_zero:

    /* 3. 跳转到用户main() */
    bl      main                    ; 调用main函数（带链接返回地址）

    /* 4. 如果main()意外返回，进入死循环 */
    b .                            ; 无限循环
```

### 关键指令语法解释

#### 1. 段定义指令
- **.section name, "attributes"**: 定义新的段
- **.align n**: 2^n字节对齐
- **.global symbol**: 声明全局符号

#### 2. 数据加载/存储指令
- **LDR Rd, =symbol**: 加载符号地址到寄存器
- **LDR Rd, [Rn]**: 从内存加载到寄存器
- **STR Rd, [Rn]**: 存储寄存器到内存

#### 3. 算术指令
- **ADDS Rd, Rn, #imm**: 加立即数并更新状态标志
- **MOVS Rd, #imm**: 移动立即数到寄存器

#### 4. 比较和分支指令
- **CMP Rd, Rn**: 比较寄存器，设置条件标志
- **BEQ label**: 相等时分支
- **BCC label**: 小于时分支（基于CMP设置的条件标志）
- **BL label**: 分支并链接（函数调用，保存返回地址）
- **B label**: 无条件分支

#### 5. 函数声明
- **.thumb_func**: 声明为Thumb函数

#### 6. 符号属性指令
- **.weak symbol**: 声明弱符号，允许被覆盖
- **.long address**: 定义32位地址值

### .long 和 .weak 的配合使用

#### 1. .long 的作用
```asm
.long Reset_Handler  ; 在中断向量表中放置Reset_Handler的地址
```
- **用途**：在中断向量表中放置实际的地址值

#### 2. .weak 的作用
```asm
.weak NMI_Handler   ; 声明NMI_Handler为弱符号
```
- **用途**：声明符号为弱符号，允许用户重写

#### 3. 配合使用的工作流程
```asm
.weak NMI_Handler   ; 声明为弱符号
.long NMI_Handler   ; 在向量表中放置地址
```

**链接器决策**：
- 如果用户定义了同名强符号，使用用户的实现
- 如果没有定义，使用默认实现（如Default_Handler）

#### 4. 实际效果
**用户可以重写**：
```c
void NMI_Handler(void) {
    // 自定义NMI处理逻辑
}
```

**如果不重写**：
- 使用启动文件中定义的默认实现
- 默认实现跳转到Default_Handler
- 防止程序崩溃

### 中断向量表结构示例
```asm
.long _estack               /* 0. 栈顶指针 */
.long Reset_Handler         /* 1. 复位中断入口 */
.weak NMI_Handler          /* 2. NMI - 声明为弱符号 */
.long NMI_Handler          /* 实际地址值 */
.weak HardFault_Handler    /* 3. 硬错误 - 声明为弱符号 */
.long HardFault_Handler    /* 实际地址值 */
```

### 总结
- **.long**：在中断向量表中放置地址
- **.weak**：声明符号属性，提供灵活性
- **配合使用**：实现可扩展的中断处理机制

### 启动流程总结

## GPIO配置详解（main.c）

### 完整源码
```c
#define RCC_BASE        0x40021000
#define RCC_APB2ENR     (*(volatile unsigned int*)(RCC_BASE + 0x18))

#define GPIOC_BASE      0x40011000
#define GPIOC_CRH       (*(volatile unsigned int*)(GPIOC_BASE + 0x04))
#define GPIOC_ODR       (*(volatile unsigned int*)(GPIOC_BASE + 0x0C))


int main(void) {
    // 使能GPIOC时钟
    RCC_APB2ENR |= (1 << 4);
    
    // 配置PC13为推挽输出模式
    // 清除PC13的MODER位 (位20-23)
    GPIOC_CRH &= ~(0xF << 20);
    // 设置为输出模式（01）- 位20-21 = 01
    GPIOC_CRH |= (0x1 << 20);  // 0x1 = 01（二进制）
    // 设置为推挽输出（00）- 位22-23 = 00
    GPIOC_CRH &= ~(0x3 << 22);
    
    // 设置PC13为低电平，点亮LED
    GPIOC_ODR &= ~(1 << 13);
    
    while (1) {
        // 保持LED常亮
    }
}
```

### GPIO寄存器配置

#### 1. 使能GPIOC时钟
```c
RCC_APB2ENR |= (1 << 4);
```
- **RCC_APB2ENR**: APB2外设时钟使能寄存器
- **位4**: IOPCEN - GPIOC时钟使能
- **|= (1 << 4)**: 设置第4位为1，使能GPIOC时钟

#### 2. 配置PC13为推挽输出模式
```c
// 清除PC13的MODER位 (位20-23)
GPIOC_CRH &= ~(0xF << 20);

// 设置为输出模式（01）- 位20-21 = 01
GPIOC_CRH |= (0x1 << 20);  // 0x1 = 01（二进制）

// 设置为推挽输出（00）- 位22-23 = 00
GPIOC_CRH &= ~(0x3 << 22);
```

#### 3. 设置PC13为低电平，点亮LED
```c
GPIOC_ODR &= ~(1 << 13);
```
- **GPIOC_ODR**: 输出数据寄存器
- **位13**: 对应PC13引脚
- **&= ~(1 << 13)**: 清除第13位，设置为低电平

### 关键指令语法解释

#### 1. 位操作指令
- **&= ~mask**: 清除指定位（与掩码取反后与操作）
- **|= value**: 设置指定位（或操作）
- **<< n**: 左移n位，创建位掩码

#### 2. 寄存器访问
- **GPIOC_BASE**: GPIOC寄存器基地址 (0x40011000)
- **GPIOC_CRH**: 配置高8位寄存器 (0x40011004)
- **GPIOC_ODR**: 输出数据寄存器 (0x4001100C)

### GPIO工作模式
- **00**: 输入模式
- **01**: 通用输出模式（推挽）
- **10**: 复用功能
- **11**: 模拟模式

### LED控制原理
- **低电平点亮**: PC13连接LED到地，低电平时LED导通
- **推挽输出**: 提供强驱动能力，适合LED控制

## 启动流程总结

1. **硬件复位**：CPU从0x08000000开始执行
2. **执行中断向量表**：跳转到Reset_Handler
3. **内存初始化**：
   - 复制.data段：Flash → RAM
   - 清零.bss段：RAM置零
4. **跳转到main**：执行用户应用程序
5. **异常处理**：通过中断向量表处理各种异常

## 链接脚本（ld文件）详解

### 链接脚本的作用

链接脚本定义了：
1. **内存布局**：Flash和RAM的地址空间分配
2. **段分配**：如何将不同的代码段（.text, .data, .bss）映射到内存
3. **符号管理**：定义关键符号如栈顶地址、段边界等

### 关键概念

#### 1. MEMORY定义
```ld
MEMORY
{
    FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 64K
    RAM   (rwx) : ORIGIN = 0x20000000, LENGTH = 20K
}
```
- 定义了目标设备的内存映射
- Flash用于存储代码和只读数据
- RAM用于存储可变数据

#### 2. 段分配策略
- **.isr_vector**：强制8字节对齐，位于Flash最开头
- **.text**：代码和只读数据段
- **.data**：带初值的全局变量，Flash中存储，运行时复制到RAM
- **.bss**：未初始化的全局变量，在RAM中清零

#### 3. 符号管理
- `_estack`：栈顶地址，指向RAM末尾
- `_sidata`, `_sdata`, `_edata`：.data段的Flash和RAM地址
- `_sbss`, `_ebss`：.bss段的RAM地址边界

## 启动流程总结

1. **硬件复位**：CPU从0x08000000开始执行
2. **执行中断向量表**：跳转到Reset_Handler
3. **内存初始化**：
   - 复制.data段：Flash → RAM
   - 清零.bss段：RAM置零
4. **跳转到main**：执行用户应用程序
5. **异常处理**：通过中断向量表处理各种异常

## 链接脚本（ld文件）详解

### 链接脚本的作用

链接脚本定义了：
1. **内存布局**：Flash和RAM的地址空间分配
2. **段分配**：如何将不同的代码段（.text, .data, .bss）映射到内存
3. **符号定义**：定义关键符号如栈顶地址、段边界等

### 关键概念

#### 1. MEMORY定义
```ld
MEMORY
{
    FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 64K
    RAM   (rwx) : ORIGIN = 0x20000000, LENGTH = 20K
}
```
- 定义了目标设备的内存映射
- Flash用于存储代码和只读数据
- RAM用于存储可变数据

#### 2. 段分配策略
- **.isr_vector**：强制8字节对齐，位于Flash最开头
- **.text**：代码和只读数据段
- **.data**：带初值的全局变量，Flash中存储，运行时复制到RAM
- **.bss**：未初始化的全局变量，在RAM中清零

#### 3. 符号管理
- `_estack`：栈顶地址，指向RAM末尾
- `_sidata`, `_sdata`, `_edata`：.data段的Flash和RAM地址
- `_sbss`, `_ebss`：.bss段的RAM地址边界

## 启动流程总结

1. **硬件复位**：CPU从0x08000000开始执行
2. **执行中断向量表**：跳转到Reset_Handler
3. **内存初始化**：
   - 复制.data段：Flash → RAM
   - 清零.bss段：RAM置零
4. **跳转到main**：执行用户应用程序
5. **异常处理**：通过中断向量表处理各种异常

## 烧录

> [!info]
> - 使用stm32f103c8t6最小系统板
> - 使用stlink SWD方式烧录
> - 连接方式GND-GND, VCC-3.3V, SWD-SWDIO, SWDCLK-SWDCLK
> - 使用st-flash命令烧录

## Makefile
```makefile
# ===== 项目名称 =====
TARGET = stm32f103c8t6_min

# ===== 工具链 =====
CC = arm-none-eabi-gcc
OBJCOPY = arm-none-eabi-objcopy
SIZE = arm-none-eabi-size

# ===== 编译选项 =====
CPUFLAGS = -mcpu=cortex-m3 -mthumb
OPTIMIZATION = -O0
DEBUG = -g
WARNINGS = -Wall -Wextra
CFLAGS = $(CPUFLAGS) $(OPTIMIZATION) $(DEBUG) $(WARNINGS) -ffreestanding -nostdlib
LDFLAGS = $(CPUFLAGS) -T STM32F103C8T6_FLASH.ld -nostdlib -ffreestanding -Wl,--gc-sections

# ===== 源文件 =====
SRCS = main.c startup.s

# ===== 目标文件 =====
OBJS = $(SRCS:.c=.o)
OBJS := $(OBJS:.s=.o)

# ===== 默认目标 =====
all: $(TARGET).elf $(TARGET).hex $(TARGET).bin size

# ===== 编译规则 =====
%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

%.o: %.s
	$(CC) $(CFLAGS) -c $< -o $@

$(TARGET).elf: $(OBJS)
	$(CC) $(LDFLAGS) $^ -o $@

$(TARGET).hex: $(TARGET).elf
	$(OBJCOPY) -O ihex $^ $@

$(TARGET).bin: $(TARGET).elf
	$(OBJCOPY) -O binary $^ $@

# ===== 查看大小 =====
size: $(TARGET).elf
	$(SIZE) $^

# ===== 烧录（使用 st-flash） =====
flash: all
	st-flash --format ihex write $(TARGET).hex

# ===== 清理 =====
clean:
	rm -f $(OBJS) $(TARGET).elf $(TARGET).hex $(TARGET).bin

.PHONY: all size flash clean
```


