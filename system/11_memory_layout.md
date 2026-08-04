# HRTOS 内存布局

## 模块介绍

HRTOS 专为内存资源有限的 8051 微控制器设计。内存布局经过精心组织，在满足 8051 架构约束的同时最大化效率，8051 架构有三个主要内存段：DATA、IDATA 和 XDATA。

## 主要职责

内存布局处理：

- 频繁访问变量的 DATA 段分配
- 内核状态和关键变量的 IDATA 段
- 大型数据结构的 XDATA 段
- 任务栈管理
- 上下文保存区域
- 栈分配的内存位图
- 内存溢出保护

## 主要文件

### 源文件

- `Src/kernel/os_core.c`：内存布局定义和变量放置
- `Src/kernel/os_data.c`：DATA 段初始化
- `Src/kernel/os_xdata.c`：XDATA 段初始化

### 头文件

- `Inc/config.h`：栈配置和内存常量
- `Inc/hrtos_internal.h`：内部内存布局定义

## 数据结构

### 内存配置常量

位于 `Inc/hrtos_internal.h`：

```c
/* 栈配置 */
#define OS_SP_KER                   5           /* 系统栈大小 */
#define OS_SP_INIT                  53          /* 系统初始 SP 值 */
#define OS_APP_STACK_MAX            0x80        /* 栈溢出阈值 */
#define OS_USER_RAM_INIT            66          /* 用户栈起始 */
#define OS_USER_RAM_EXIT            127         /* 用户栈结束 */
#define OS_SP_INTERRUPT_RETURN      58          /* 中断返回 SP */
#define OS_KUAI_SHENDU              5           /* 快速任务上下文深度 */
#define OS_JINCHENG_SHENDU          13          /* 标准任务上下文深度 */

/* 内存管理 */
#define OS_MEMORY_ADMINISTRATION   4           /* 内存管理大小 */
```

### DATA 段布局（0x20-0x34）

位于 `Src/kernel/os_core.c`：

```c
/* 0x30 - 中断 ID 保护 */
volatile unsigned char data OS_INTERRUPT_PROTECT _at_ 0x30;

/* 0x31 - R0 寄存器保护 */
volatile unsigned char data OS_R0_PROTECT _at_ 0x31;

/* 0x32 - 通用工作变量 */
volatile unsigned char data os_sp _at_ 0x32;

/* 0x33 - 内核临时变量 */
volatile unsigned char data OS_HRTOS _at_ 0x33;

/* 0x34 - 中断保护寄存器 */
volatile unsigned char data OS_HRTOS_2 _at_ 0x34;
```

### 栈区域（DATA 段）

```c
/* 0x35-0x39 - 系统栈（5 字节） */
unsigned char data OS_HRTOS_DUIZHAN_OS[OS_SP_KER] _at_ 0x35;

/* 0x3A-0x41 - 中断栈（8 字节） */
unsigned char data OS_HRTOS_DUIZHAN_INT[8] _at_ 0x3a;

/* 0x42-0x7F - 用户任务栈（62 字节） */
unsigned char data OS_HRTOS_DUIZHAN_APP[62] _at_ 0x42;
```

### IDATA 段布局（0xE0-0xE7）

```c
/* 0xE7 - 临界区嵌套 */
volatile unsigned char idata os_critical_nesting _at_ 0xE7;

/* 0xE6 - 时间片控制 */
volatile unsigned char idata OS_TIME_XY _at_ 0xE6;

/* 0xE5 - 分发触发源 */
volatile unsigned char idata OS_DISPATCH_ID _at_ 0xE5;

/* 0xE4 - 前一个任务 */
volatile unsigned char idata OS_PREV_TASK _at_ 0xE4;

/* 0xE3 - 当前任务 */
volatile unsigned char idata OS_CURRENT_TASK _at_ 0xE3;

/* 0xE2 - Timer0 TH0 */
volatile unsigned char idata OS_T0_TH0 _at_ 0xE2;

/* 0xE1 - Timer0 TL0 */
volatile unsigned char idata OS_T0_TL0 _at_ 0xE1;

/* 0xDB-0xE0 - 中断上下文保存（6 字节） */
volatile unsigned char idata OS_INT_BAO[6] _at_ 0xDB;
```

### XDATA 段布局（0x0200-0x03FF）

基地址：0x0200，总大小：512 字节

#### 偏移映射

```c
#define OS_SYS_GLOBAL_BASE   0x0200
#define OS_SYS_GLOBAL_SIZE    512

/* 任务状态 / 控制 */
#define OS_OFF_PROCESS_OK            0x0000  /* 任务状态数组（16B） */
#define OS_OFF_JINCHENG_SP           0x0010  /* 标准任务 SP（16B） */
#define OS_OFF_KUAI_SP               0x0020  /* 快速任务 SP（2B） */

/* 上下文保存区域 */
#define OS_OFF_FAST_CONTEXT          0x0030  /* 快速任务上下文（10B） */
#define OS_OFF_TASK_CONTEXT          0x0040  /* 标准任务上下文（208B） */

/* 内存 / 栈管理 */
#define OS_OFF_MEMORY                0x0110  /* 栈位图（4B） */
#define OS_OFF_JINCHENG_JILU_SP      0x0120  /* SP 备份（16B） */

/* 时间 / 调度器 */
#define OS_OFF_TIME_ONCE             0x0130  /* 时间片（1B） */
#define OS_OFF_WAIT_DI2              0x0131  /* 快速任务延时（4B） */

/* 中断控制 */
#define OS_OFF_INTERRUPT_ADDR        0x0135  /* 中断地址（2B） */
#define OS_OFF_INTERRUPT_ADDR2       0x0137  /* 中断地址 2（2B） */

/* 事件 / 时钟系统 */
#define OS_OFF_TICK_L                0x0139  /* 时钟低 16 位（2B） */
#define OS_OFF_TICK_H                0x013B  /* 时钟高 16 位（2B） */
#define OS_OFF_EVENT_BIT             0x013D  /* 事件位图（2B） */

/* 内存保护 / 调试 */
#define OS_OFF_STACK_WARN            0x013F  /* 栈警告（1B） */
#define OS_OFF_KUAI_BEI              0x0140  /* 快速任务 SP 起始（2B） */
#define OS_OFF_KUAI_BEI2             0x0142  /* 快速任务 SP 大小（2B） */
#define OS_OFF_TASK_EVENT            0x0144  /* 任务事件标志（18B） */

/* 时间备份 */
#define OS_OFF_TIME_BACKUP           0x0156  /* 时间片备份（1B） */
```

#### XDATA 变量放置

```c
/* 任务状态 / 控制 */
volatile unsigned char xdata OS_PROCESS_OK[OS_PROCESS_MAX]
    _at_ (OS_SYS_GLOBAL_BASE + OS_OFF_PROCESS_OK);

volatile unsigned char xdata OS_JINCHENG_SP[OS_PROCESS_MAX]
    _at_ (OS_SYS_GLOBAL_BASE + OS_OFF_JINCHENG_SP);

volatile unsigned char xdata OS_KUAI_SP[OS_FAST_TASK_MAX]
    _at_ (OS_SYS_GLOBAL_BASE + OS_OFF_KUAI_SP);

/* 上下文保存区域 */
volatile unsigned char xdata OS_FAST_CONTEXT[OS_FAST_TASK_MAX][OS_KUAI_SHENDU]
    _at_ (OS_SYS_GLOBAL_BASE + OS_OFF_FAST_CONTEXT);

volatile unsigned char xdata OS_TASK_CONTEXT[OS_PROCESS_MAX][OS_JINCHENG_SHENDU]
    _at_ (OS_SYS_GLOBAL_BASE + OS_OFF_TASK_CONTEXT);

/* 内存 / 栈管理 */
volatile unsigned char xdata OS_MEMORY[OS_MEMORY_ADMINISTRATION]
    _at_ (OS_SYS_GLOBAL_BASE + OS_OFF_MEMORY);

volatile unsigned char xdata OS_JINCHENG_JILU_SP[OS_PROCESS_MAX]
    _at_ (OS_SYS_GLOBAL_BASE + OS_OFF_JINCHENG_JILU_SP);

/* 时间 / 调度器 */
volatile unsigned char xdata OS_TIME_ONCE
    _at_ (OS_SYS_GLOBAL_BASE + OS_OFF_TIME_ONCE);

volatile unsigned int xdata OS_WAIT_DI2[OS_FAST_TASK_MAX]
    _at_ (OS_SYS_GLOBAL_BASE + OS_OFF_WAIT_DI2);

/* 中断控制 */
volatile unsigned char xdata OS_INTERRUPT_ADDR[2]
    _at_ (OS_SYS_GLOBAL_BASE + OS_OFF_INTERRUPT_ADDR);

volatile unsigned char xdata OS_INTERRUPT_ADDR2[2]
    _at_ (OS_SYS_GLOBAL_BASE + OS_OFF_INTERRUPT_ADDR2);

/* 事件 / 时钟系统 */
volatile unsigned int xdata OS_TICK_COUNT_L
    _at_ (OS_SYS_GLOBAL_BASE + OS_OFF_TICK_L);

volatile unsigned int xdata OS_TICK_COUNT_H
    _at_ (OS_SYS_GLOBAL_BASE + OS_OFF_TICK_H);

volatile unsigned int xdata OS_EVENT_BIT
    _at_ (OS_SYS_GLOBAL_BASE + OS_OFF_EVENT_BIT);

/* 内存保护 / 调试 */
volatile unsigned char xdata OS_INSIDE_STACK_USE_MAXIMUM
    _at_ (OS_SYS_GLOBAL_BASE + OS_OFF_STACK_WARN);

volatile unsigned char xdata OS_SP_KUAI_BEI[OS_FAST_TASK_MAX]
    _at_ (OS_SYS_GLOBAL_BASE + OS_OFF_KUAI_BEI);

volatile unsigned char xdata OS_SP_KUAI_BEI2[OS_FAST_TASK_MAX]
    _at_ (OS_SYS_GLOBAL_BASE + OS_OFF_KUAI_BEI2);

volatile unsigned char xdata OS_TASK_EVENT[OS_PROCESS_MAX + OS_FAST_TASK_MAX]
    _at_ (OS_SYS_GLOBAL_BASE + OS_OFF_TASK_EVENT);

/* 时间备份 */
volatile unsigned char xdata OS_TIME_ONCE_BACKUP
    _at_ (OS_SYS_GLOBAL_BASE + OS_OFF_TIME_BACKUP);
```

## 核心函数

### 栈分配（位图）

位于 `Src/kernel/task_create.c`：

```c
// 基于位图的栈分配
for(i=0; i<(OS_USER_RAM_EXIT-OS_USER_RAM_INIT)/2; i++)
{
    x=i/8;                    // 位图字节索引
    OS_R0_PROTECT=i%8;        // 位位置
    x=OS_MEMORY[x]&(0x80>>OS_R0_PROTECT);
    if(x==0)                  // 空闲块
    {
        k++;                  // 计数连续空闲
        if(k>=sd)             // 足够空间
        {
            // 标记为已使用
            for(j=i-k+1; j<=i; j++)
            {
                x=j/8;
                OS_R0_PROTECT=j%8;
                OS_MEMORY[x]|=(0x80>>OS_R0_PROTECT);
            }
        }
    }
    else
    {
        k=0;                  // 重置计数器
    }
}
```

### 栈释放

位于 `Src/kernel/task_cleanup.c`：

```c
// 释放栈块
for(; OS_HRTOS<m; OS_HRTOS++)
{
    os_sp=OS_HRTOS>>3;
    OS_R0_PROTECT=OS_HRTOS&7;
    j=(~(0x80>>OS_R0_PROTECT));
    OS_MEMORY[os_sp]&=j;
}
```

## 内存映射摘要

### DATA 段（0x00-0x7F）

| 地址范围 | 大小 | 用途 |
|---------------|------|---------|
| 0x00-0x1F | 32 字节 | 寄存器组（R0-R7） |
| 0x20-0x2F | 16 字节 | 可位寻址区域 |
| 0x30-0x34 | 5 字节 | 内核临时变量 |
| 0x35-0x39 | 5 字节 | 系统栈 |
| 0x3A-0x41 | 8 字节 | 中断栈 |
| 0x42-0x7F | 62 字节 | 用户任务栈区域 |

### IDATA 段（0x80-0xFF）

| 地址范围 | 大小 | 用途 |
|---------------|------|---------|
| 0x80-0xFF | 128 字节 | 上 128 字节（内部 RAM） |
| 0xE0-0xE7 | 8 字节 | 内核状态变量 |
| 0xE8-0xFF | 24 字节 | 保留 |

### XDATA 段（0x0000-0xFFFF）

| 地址范围 | 大小 | 用途 |
|---------------|------|---------|
| 0x0000-0x01FF | 512 字节 | 应用程序使用 |
| 0x0200-0x03FF | 512 字节 | HRTOS 全局池 |
| 0x0400+ | - | 应用程序使用 |

## 设计原则

### 8051 内存架构

- **DATA**：直接寻址，最快访问
- **IDATA**：间接寻址，上 128 字节
- **XDATA**：外部内存，更大但更慢

### 访问频率优化

- 频繁访问的变量在 DATA/IDATA
- 大型数组在 XDATA
- 关键状态在 IDATA（0xE0-0xE7）

### 栈管理

- 基于位图的分配
- 2 字节块
- 用户栈：0x42-0x7F（62 字节 = 31 块）
- 系统栈和中断栈分离

### 上下文保存优化

- 快速任务：5 个寄存器（最小）
- 标准任务：13 个寄存器（完整）
- 位于 XDATA 以提高空间效率

## 约束

- 总用户栈：62 字节
- 最大任务栈：受可用块限制
- XDATA 池：总共 512 字节
- 无动态内存分配
- 栈溢出检测有限
- 不处理内存碎片（静态分配）

## 内存使用摘要

### DATA 段使用

- 内核变量：5 字节（0x30-0x34）
- 系统栈：5 字节（0x35-0x39）
- 中断栈：8 字节（0x3A-0x41）
- 用户栈：62 字节（0x42-0x7F）
- 总计：80 字节

### IDATA 段使用

- 内核状态：13 字节（0xDB-0xE7）
- 总计：13 字节

### XDATA 段使用

- 任务状态：16 字节
- 栈指针：18 字节
- 上下文保存：218 字节
- 内存位图：4 字节
- 定时：11 字节
- 中断：4 字节
- 事件/时钟：6 字节
- 调试：24 字节
- 总计：~301 字节（512 字节中）

## 移植考虑

### 对于不同的 8051 变体

- 根据可用 RAM 调整栈大小
- 如需要修改 XDATA 基地址
- 为不同内存大小更新偏移映射
- 为不同架构调整上下文保存深度

### 内存配置

```c
// 为不同目标调整这些
#define OS_USER_RAM_INIT            66
#define OS_USER_RAM_EXIT            127
#define OS_SYS_GLOBAL_BASE          0x0200
#define OS_SYS_GLOBAL_SIZE          512
```
