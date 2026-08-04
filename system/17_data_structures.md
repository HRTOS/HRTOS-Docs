# HRTOS 数据结构

## 模块介绍

HRTOS 使用一组精心设计的数据结构来管理任务、IPC 资源、定时和系统状态。本文档提供了内核中使用的所有主要数据结构的综合参考。

## 主要职责

数据结构处理：

- 任务控制和状态
- IPC 资源管理
- 消息队列缓冲
- 内存分配跟踪
- 定时和时钟周期管理
- 中断上下文保存

## 数据结构参考

### OS_TCB（任务控制块）

**位置**：`Inc/config.h`

**目的**：存储有关任务的所有信息

**定义**：
```c
typedef struct {
    u8 id;              /* 任务 ID */
    u8 base_prio;       /* 静态优先级 */
    u8 cur_prio;        /* 动态优先级 */
    u8 state;           /* 任务状态 */
    u8 wait_type;       /* 等待类型 */
    u8 wait_flag;       /* 等待结果标志 */
    u8 wait_obj;        /* 等待对象 ID */
    u16 wait_tick;      /* 超时时钟周期 */
} OS_TCB;
```

**字段描述**：

- `id`：唯一任务标识符（0-17）
- `base_prio`：创建时分配的原始优先级
- `cur_prio`：当前优先级（可能被优先级继承提升）
- `state`：当前任务状态（READY、WAIT、RUNNING、SUSPEND、DEAD）
- `wait_type`：等待类型（WAIT_DELAY、WAIT_SEM 等）
- `wait_flag`：唤醒原因（WAIT_TIMEOUT、WAIT_SIGNAL）
- `wait_obj`：正在等待的资源 ID
- `wait_tick`：超时前的剩余时钟周期

**数组**：XDATA 中的 `OS_TASK[OS_PROCESS_MAX]`

**大小**：每个 TCB 9 字节

**总大小**：144 字节（16 个任务 × 9 字节）

### OS_RESOURCE（统一 IPC 资源）

**位置**：`Inc/config.h`

**目的**：所有 IPC 类型（信号量、互斥锁、事件、邮箱）的统一结构

**定义**：
```c
typedef struct {
    u8 value;          /* 资源值 */
    u8 owner;          /* 当前所有者（仅互斥锁） */
    u8 wait_cnt;       /* 等待任务计数 */
    u16 wait_mask;     /* 位图等待队列 */
    u8 pending_signal; /* ISR 信号计数 */
} OS_RESOURCE;
```

**字段描述**：

- `value`：资源特定值
  - 信号量：可用资源计数
  - 事件：事件标志（0/1）
  - 互斥锁：未使用
  - 邮箱：邮箱值
- `owner`：持有互斥锁的任务 ID（如果空闲则为 OS_INVALID_ID）
- `wait_cnt`：等待此资源的任务数
- `wait_mask`：等待任务的位图（位 0 = 任务 0，位 1 = 任务 1 等）
- `pending_signal`：ISR 信号计数（用于 ISR 安全操作）

**数组**：XDATA 中的 `OS_RES[OS_RESOURCE_MAX]`

**大小**：每个资源 6 字节

**总大小**：48 字节（8 个资源 × 6 字节）

### os_msgq_t（消息队列）

**位置**：`Inc/config.h`

**目的**：消息队列的循环缓冲区

**定义**：
```c
typedef struct {
    u8 *buf;            /* 缓冲区指针 */
    u8 _size;           /* 缓冲区大小 */
    u8 head;            /* 写指针（生产者） */
    u8 tail;            /* 读指针（消费者） */
    u8 count;           /* 当前消息计数 */
} os_msgq_t;
```

**字段描述**：

- `buf`：指向用户提供的缓冲区的指针
- `_size`：缓冲区容量
- `head`：下一条消息将写入的索引
- `tail`：下一条消息将读取的索引
- `count`：队列中当前的消息数

**数组**：XDATA 中的 `OS_MSGQ[OS_MSGQ_MAX]`

**大小**：每个队列 5 字节

**总大小**：20 字节（4 个队列 × 5 字节）

**注意**：实际消息缓冲区由用户提供，不计入内核内存

### OS_PROCESS_OK（任务状态数组）

**位置**：`Src/kernel/os_core.c`

**目的**：编码任务状态、优先级和栈大小

**定义**：
```c
volatile unsigned char xdata OS_PROCESS_OK[OS_PROCESS_MAX];
```

**编码**：
- 位 0：就绪标志（1 = 准备运行，0 = 未就绪）
- 位 1-3：优先级级别（0-7）
- 位 4-7：栈大小（层）

**示例**：
- `0x01`：就绪，优先级 0，栈大小 0
- `0x1F`：就绪，优先级 7，栈大小 1
- `0x00`：未就绪（已删除或挂起）

**大小**：每个任务 1 字节

**总大小**：16 字节

### 栈指针

**位置**：`Src/kernel/os_core.c`

**目的**：跟踪每个任务的栈指针

#### OS_JINCHENG_SP（标准任务栈指针）

```c
volatile unsigned char xdata OS_JINCHENG_SP[OS_PROCESS_MAX];
```

**目的**：每个标准任务的当前栈指针

**大小**：每个任务 1 字节

**总大小**：16 字节

#### OS_JINCHENG_JILU_SP（初始栈指针）

```c
volatile unsigned char xdata OS_JINCHENG_JILU_SP[OS_PROCESS_MAX];
```

**目的**：用于清理的初始栈指针

**大小**：每个任务 1 字节

**总大小**：16 字节

#### OS_KUAI_SP（快速任务栈指针）

```c
volatile unsigned char xdata OS_KUAI_SP[OS_FAST_TASK_MAX];
```

**目的**：快速任务的当前栈指针

**大小**：每个快速任务 1 字节

**总大小**：2 字节

### 上下文保存区域

**位置**：`Src/kernel/os_core.c`

**目的**：上下文切换期间保存任务上下文

#### OS_TASK_CONTEXT（标准任务上下文）

```c
volatile unsigned char xdata OS_TASK_CONTEXT[OS_PROCESS_MAX][OS_JINCHENG_SHENDU];
```

**大小**：每个任务 13 字节

**内容**：PWS、ACC、B、DPTR、寄存器组

**总大小**：208 字节（16 个任务 × 13 字节）

#### OS_FAST_CONTEXT（快速任务上下文）

```c
volatile unsigned char xdata OS_FAST_CONTEXT[OS_FAST_TASK_MAX][OS_KUAI_SHENDU];
```

**大小**：每个快速任务 5 字节

**内容**：PWS、ACC、B、DPTR

**总大小**：10 字节（2 个任务 × 5 字节）

### 内存位图

**位置**：`Src/kernel/os_core.c`

**目的**：栈分配的位图

**定义**：
```c
volatile unsigned char xdata OS_MEMORY[OS_MEMORY_ADMINISTRATION];
```

**大小**：4 字节（32 位）

**用途**：
- 每位代表一个 2 字节栈块
- 位 1 = 块已分配
- 位 0 = 块空闲
- 支持最多 31 个块（总共 62 字节）

### 定时变量

**位置**：`Src/kernel/os_core.c` 和 `Inc/hrtos_internal.h`

#### OS_TIME_XY（时间片计数器）

```c
volatile unsigned char idata OS_TIME_XY;
```

**目的**：当前时间片计数器

**大小**：1 字节

#### OS_TIME_ONCE（时间片计数）

```c
volatile unsigned char xdata OS_TIME_ONCE;
```

**目的**：每个调度周期的时间片数

**大小**：1 字节

#### OS_TICK_COUNT_L/H（时钟周期计数器）

```c
volatile unsigned int xdata OS_TICK_COUNT_L;
volatile unsigned int xdata OS_TICK_COUNT_H;
```

**目的**：32 位系统时钟周期计数器

**大小**：总共 4 字节（2 × 2 字节）

#### OS_WAIT_DI2（快速任务延时）

```c
volatile unsigned int xdata OS_WAIT_DI2[OS_FAST_TASK_MAX];
```

**目的**：快速任务的延时计数器

**大小**：总共 4 字节（2 × 2 字节）

### 中断变量

**位置**：`Src/kernel/os_core.c`

#### OS_INTERRUPT_ADDR（中断处理程序地址）

```c
volatile unsigned char xdata OS_INTERRUPT_ADDR[2];
```

**目的**：中断处理程序地址（优先级 9）

**大小**：2 字节

#### OS_INTERRUPT_ADDR2（嵌套中断处理程序地址）

```c
volatile unsigned char xdata OS_INTERRUPT_ADDR2[2];
```

**目的**：嵌套中断处理程序地址（优先级 10）

**大小**：2 字节

#### OS_INT_BAO（中断上下文保存）

```c
volatile unsigned char idata OS_INT_BAO[6];
```

**目的**：中断期间保存 ACC、B、DPH、DPL、PSW、EA

**大小**：6 字节

### 临界区变量

**位置**：`Inc/hrtos_internal.h`

#### os_critical_nesting（嵌套计数器）

```c
volatile unsigned char idata os_critical_nesting;
```

**目的**：嵌套临界区的计数

**大小**：1 字节

#### os_int_status（保存的中断状态）

```c
volatile bit os_int_status;
```

**目的**：临界区入口时保存的 EA 状态

**大小**：1 位

### 调度器变量

**位置**：`Src/kernel/os_core.c`

#### OS_CURRENT_TASK（当前任务 ID）

```c
volatile unsigned char idata OS_CURRENT_TASK;
```

**目的**：当前运行任务的 ID

**大小**：1 字节

#### OS_PREV_TASK（前一个任务 ID）

```c
volatile unsigned char idata OS_PREV_TASK;
```

**目的**：前一个运行任务的 ID

**大小**：1 字节

#### OS_DISPATCH_ID（分发触发）

```c
volatile unsigned char idata OS_DISPATCH_ID;
```

**目的**：分发触发的来源

**大小**：1 字节

#### OS_SCHED_REASON（调度触发）

```c
volatile bit OS_SCHED_REASON;
```

**目的**：指示需要调度的标志

**大小**：1 位

### 快速任务控制

**位置**：`Src/kernel/os_core.c`

#### OS_KUAI_PROCESS_A/B（快速任务标志）

```c
volatile bit OS_KUAI_PROCESS_A;
volatile bit OS_KUAI_PROCESS_B;
```

**目的**：快速任务的激活标志

**大小**：2 位

#### OS_SP_KUAI_BEI（快速任务 SP 起始）

```c
volatile unsigned char xdata OS_SP_KUAI_BEI[OS_FAST_TASK_MAX];
```

**目的**：快速任务的初始栈指针

**大小**：2 字节

#### OS_SP_KUAI_BEI2（快速任务栈大小）

```c
volatile unsigned char xdata OS_SP_KUAI_BEI2[OS_FAST_TASK_MAX];
```

**目的**：快速任务的栈大小

**大小**：2 字节

### 事件变量

**位置**：`Inc/hrtos_internal.h`

#### OS_EVENT_BIT（事件位图）

```c
volatile unsigned int xdata OS_EVENT_BIT;
```

**目的**：事件位操作

**大小**：2 字节

#### OS_TASK_EVENT（任务事件标志）

```c
volatile unsigned char xdata OS_TASK_EVENT[OS_PROCESS_MAX + OS_FAST_TASK_MAX];
```

**目的**：用于任务删除的事件标志

**大小**：18 字节

### 栈区域

**位置**：`Src/kernel/os_core.c`

#### OS_HRTOS_DUIZHAN_OS（系统栈）

```c
unsigned char data OS_HRTOS_DUIZHAN_OS[OS_SP_KER] _at_ 0x35;
```

**目的**：内核操作的系统栈

**大小**：5 字节

**地址**：0x35-0x39

#### OS_HRTOS_DUIZHAN_INT（中断栈）

```c
unsigned char data OS_HRTOS_DUIZHAN_INT[8] _at_ 0x3a;
```

**目的**：中断处理程序的栈

**大小**：8 字节

**地址**：0x3A-0x41

#### OS_HRTOS_DUIZHAN_APP（用户栈）

```c
unsigned char data OS_HRTOS_DUIZHAN_APP[62] _at_ 0x42;
```

**目的**：用户任务的栈区域

**大小**：62 字节

**地址**：0x42-0x7F

### 内核临时变量

**位置**：`Src/kernel/os_core.c`

#### DATA 段变量

```c
volatile unsigned char data OS_INTERRUPT_PROTECT _at_ 0x30;
volatile unsigned char data OS_R0_PROTECT _at_ 0x31;
volatile unsigned char data os_sp _at_ 0x32;
volatile unsigned char data OS_HRTOS _at_ 0x33;
volatile unsigned char data OS_HRTOS_2 _at_ 0x34;
```

**目的**：内核操作的临时变量

**大小**：5 字节

**地址**：0x30-0x34

### 系统标志

**位置**：`Src/kernel/os_core.c`

```c
volatile bit OS_KUAI_PROCESS_A;
volatile bit OS_KUAI_PROCESS_B;
volatile bit OS_JINCHENG_CUNCHU;
volatile bit OS_SCHED_REASON;
volatile bit OS_ISR_FLAG;
volatile bit OS_INTERRUPT_T0;
volatile bit OS_RTOS_YES;
volatile bit OS_a;
volatile bit OS_b;
volatile bit OS_INTERRUPT_GAO;
volatile bit OS_INTERRUPT_GAO_BIAO;
volatile bit os_int_status;
volatile bit OS_CFG_LOCK;
```

**目的**：各种系统标志和状态指示器

**大小**：12 位

## 内存使用摘要

### DATA 段（0x20-0x7F）

| 结构 | 大小 | 地址 |
|-----------|------|---------|
| 内核临时变量 | 5 字节 | 0x30-0x34 |
| 系统栈 | 5 字节 | 0x35-0x39 |
| 中断栈 | 8 字节 | 0x3A-0x41 |
| 用户栈 | 62 字节 | 0x42-0x7F |
| **总计** | **80 字节** | |

### IDATA 段（0xE0-0xE7）

| 结构 | 大小 | 地址 |
|-----------|------|---------|
| OS_CURRENT_TASK | 1 字节 | 0xE3 |
| OS_PREV_TASK | 1 字节 | 0xE4 |
| OS_DISPATCH_ID | 1 字节 | 0xE5 |
| OS_TIME_XY | 1 字节 | 0xE6 |
| os_critical_nesting | 1 字节 | 0xE7 |
| OS_T0_TH0 | 1 字节 | 0xE2 |
| OS_T0_TL0 | 1 字节 | 0xE1 |
| OS_INT_BAO | 6 字节 | 0xDB-0xE0 |
| **总计** | **13 字节** | |

### XDATA 段（0x0200-0x03FF）

| 结构 | 大小 | 偏移 |
|-----------|------|--------|
| OS_PROCESS_OK | 16 字节 | 0x0000 |
| OS_JINCHENG_SP | 16 字节 | 0x0010 |
| OS_KUAI_SP | 2 字节 | 0x0020 |
| OS_FAST_CONTEXT | 10 字节 | 0x0030 |
| OS_TASK_CONTEXT | 208 字节 | 0x0040 |
| OS_MEMORY | 4 字节 | 0x0110 |
| OS_JINCHENG_JILU_SP | 16 字节 | 0x0120 |
| OS_TIME_ONCE | 1 字节 | 0x0130 |
| OS_WAIT_DI2 | 4 字节 | 0x0131 |
| OS_INTERRUPT_ADDR | 2 字节 | 0x0135 |
| OS_INTERRUPT_ADDR2 | 2 字节 | 0x0137 |
| OS_TICK_COUNT_L | 2 字节 | 0x0139 |
| OS_TICK_COUNT_H | 2 字节 | 0x013B |
| OS_EVENT_BIT | 2 字节 | 0x013D |
| OS_INSIDE_STACK_USE_MAXIMUM | 1 字节 | 0x013F |
| OS_SP_KUAI_BEI | 2 字节 | 0x0140 |
| OS_SP_KUAI_BEI2 | 2 字节 | 0x0142 |
| OS_TASK_EVENT | 18 字节 | 0x0144 |
| OS_TIME_ONCE_BACKUP | 1 字节 | 0x0156 |
| OS_TASK（TCB 数组） | 144 字节 | 动态 |
| OS_RES（资源数组） | 48 字节 | 动态 |
| OS_MSGQ（队列数组） | 20 字节 | 动态 |
| **总计** | **~520 字节** | |

## 数据结构设计原则

### 紧凑设计

- 最小化内存使用
- 适当时使用位字段
- 高效打包数据
- 避免填充

### 静态分配

- 所有结构静态分配
- 无动态内存
- 可预测的内存使用
- 无碎片

### 统一模型

- 多种 IPC 类型的单一结构
- 一致的 API 模式
- 简化实现
- 减少代码大小

### 访问优化

- 频繁访问的变量在 DATA/IDATA
- 大数组在 XDATA
- 关键状态在快速内存
- 最小化 XDATA 访问

## 数据结构约束

- 最多 16 个标准任务
- 最多 2 个快速任务
- 最多 8 个 IPC 资源
- 最多 4 个消息队列
- 栈限制为 62 字节
- 无动态分配
- 运行时无数据结构扩展
