# HRTOS 内存布局

## 模块介绍

HRTOS 采用固定地址布局管理 8051 内部 RAM 和 XDATA 全局内存池。内部 RAM 主要用于内核临时变量、系统栈、中断栈、用户任务栈以及调度状态；XDATA 全局内存池用于保存任务状态、任务上下文、栈指针记录、时间数据、任务控制块、资源对象和消息队列对象。

本文以当前源码中的绝对地址声明、偏移宏、数组长度及结构体定义为依据，说明实际内存分配范围。仅存在偏移宏或注释、没有实际存储对象声明的区域，不计入已分配内存。

固定地址布局的正确性取决于以下条件：

- 对象起始地址与偏移宏一致。
- 数组长度与后续对象的起始地址相匹配。
- 结构体实际大小与固定地址间距一致。
- 内存区域不发生地址重叠。
- 目标芯片具有相应的 RAM 容量和寻址能力。

## 主要文件

| 文件 | 主要职责 |
|---|---|
| `Src/kernel/os_core.c` | 内核固定地址变量及 XDATA 对象的定义 |
| `Src/kernel/os_data.c` | DATA 区域相关初始化 |
| `Src/kernel/os_xdata.c` | XDATA 区域相关初始化 |
| `Src/kernel/task_create.c` | 任务创建及用户栈分配 |
| `Src/kernel/task_cleanup.c` | 任务清理及用户栈释放 |
| `Inc/config.h` | 任务数量、栈深度及内核容量配置 |
| `Inc/hrtos_internal.h` | 内核变量、任务控制块和内核对象结构定义 |

## 内存空间概览

HRTOS 当前源码涉及以下存储区域。

| 存储区域 | 地址范围 | 用途 |
|---|---|---|
| 内部 RAM 低地址区 | `0x00–0x7F` | 寄存器组、位寻址 RAM、内核变量及栈 |
| 内部 RAM 高地址区 | `0x80–0xFF` | 通过 IDATA 间接寻址访问的内部 RAM，具体取决于芯片 |
| XDATA 全局内存池 | `0x0200–0x03FF` | 任务上下文、控制块及内核对象 |

DATA 和 IDATA 是 8051 的不同寻址方式，并不意味着它们必然对应两块独立的物理 RAM。高地址 IDATA 是否可用，取决于具体芯片的内部 RAM 实现。

XDATA 全局池的规划容量为 512 字节。实际已分配对象的大小、地址间隙和剩余空间，应分别统计，不能仅凭全局池的结束地址推断已使用容量。

## 内部 RAM 布局

### 固定地址内核变量

| 起始地址 | 结束地址 | 大小 | 变量 |
|---|---|---:|---|
| `0x15` | `0x15` | 1 字节 | `OS_IRQ_NEST_TEMP` |
| `0x30` | `0x30` | 1 字节 | `OS_INTERRUPT_PROTECT` |
| `0x31` | `0x31` | 1 字节 | `OS_R0_PROTECT` |
| `0x32` | `0x32` | 1 字节 | `os_sp` |
| `0x33` | `0x33` | 1 字节 | `OS_HRTOS` |
| `0x34` | `0x34` | 1 字节 | `OS_HRTOS_2` |

以上地址由源码中的 `_at_` 声明直接指定。

`OS_INTERRUPT_PROTECT` 保存中断编号相关信息；`OS_R0_PROTECT` 用于 R0 保护；`os_sp`、`OS_HRTOS` 和 `OS_HRTOS_2` 用于各自实现路径中的内核临时状态。

`OS_IRQ_NEST_TEMP` 位于 `0x15`，不属于 `0x30–0x34` 连续变量区。

### 系统栈、中断栈与用户任务栈

当前源码定义如下：

```c
unsigned char data OS_HRTOS_DUIZHAN_OS[OS_SP_KER] _at_ 0x35;

unsigned char data OS_HRTOS_DUIZHAN_INT[8] _at_ 0x3A;

unsigned char data OS_HRTOS_DUIZHAN_APP[62] _at_ 0x42;
```

在 `OS_SP_KER = 5` 的配置下，内存范围如下。

| 栈区域 | 起始地址 | 结束地址 | 容量 |
|---|---|---|---:|
| 系统栈 | `0x35` | `0x39` | 5 字节 |
| 中断栈 | `0x3A` | `0x41` | 8 字节 |
| 用户任务栈 | `0x42` | `0x7F` | 62 字节 |

用户任务栈覆盖十进制地址 `66–127`，共 62 字节。`OS_USER_RAM_EXIT = 128` 是不包含在有效范围内的结束边界。

按照每个栈块 2 字节计算，用户任务栈可划分为 31 个栈块。实际可分配给单个任务的栈空间由任务创建参数和栈分配算法决定。

注意：源码中的注释与数组实际长度存在不一致的地方。本文以 `_at_` 地址和数组长度为准，而不是以旧注释中的起止地址为准。

### IDATA 中断保护及调度状态

当前源码将以下变量固定在 IDATA 高地址区域。

| 起始地址 | 结束地址 | 大小 | 变量 |
|---|---|---:|---|
| `0xDB` | `0xE0` | 6 字节 | `OS_INT_BAO` |
| `0xE1` | `0xE1` | 1 字节 | `OS_T0_TL0` |
| `0xE2` | `0xE2` | 1 字节 | `OS_T0_TH0` |
| `0xE3` | `0xE3` | 1 字节 | `OS_CURRENT_TASK` |
| `0xE4` | `0xE4` | 1 字节 | `OS_PREV_TASK` |
| `0xE5` | `0xE5` | 1 字节 | `OS_DISPATCH_ID` |
| `0xE6` | `0xE6` | 1 字节 | `OS_TIME_XY` |
| `0xE7` | `0xE7` | 1 字节 | `os_critical_nesting` |

`OS_INT_BAO` 占用 `0xDB–0xE0` 六个连续字节，后续变量从 `0xE1` 开始，整个固定地址区域共占用 13 字节。

各变量用途如下：

- `OS_INT_BAO`：中断现场保护数据。
- `OS_T0_TL0`、`OS_T0_TH0`：Timer0 相关保存数据。
- `OS_CURRENT_TASK`：当前任务 ID。
- `OS_PREV_TASK`：上一个任务 ID。
- `OS_DISPATCH_ID`：调度触发源。
- `OS_TIME_XY`：时间片控制状态。
- `os_critical_nesting`：临界区嵌套计数。

寄存器保存顺序及恢复行为由中断入口、汇编上下文保存代码和恢复流程决定。

### BIT 类型内核标志

源码声明了以下 `volatile bit` 变量：

```c
volatile bit OS_KUAI_PROCESS_A;
volatile bit OS_KUAI_PROCESS_B;
volatile bit OS_KUAI_PROCESS_A_DEL;
volatile bit OS_KUAI_PROCESS_B_DEL;

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

这些变量用于高速任务生命周期、调度请求、中断状态、内核模式及配置保护等功能。

它们没有在当前代码中通过独立的 `_at_` 地址逐一指定位置。因此，不能根据声明顺序为它们编造具体 RAM 地址，也不能直接把变量数量换算成某个连续 DATA 地址范围。

实际位存储分配应以 Keil C51 编译结果和链接映射为准。

## XDATA 全局内存池

### 基本配置

```c
#define OS_SYS_GLOBAL_BASE  0x0200
#define OS_SYS_GLOBAL_SIZE  512
#define OS_SYS_GLOBAL_END   (OS_SYS_GLOBAL_BASE + OS_SYS_GLOBAL_SIZE - 1)
```

| 项目 | 数值 |
|---|---|
| 基地址 | `0x0200` |
| 总容量 | 512 字节 |
| 最后一个有效地址 | `0x03FF` |
| 完整地址范围 | `0x0200–0x03FF` |

每个 XDATA 对象的起始地址按下式计算：

```text
对象起始地址 = OS_SYS_GLOBAL_BASE + 对象偏移
```

全局池的容量为 512 字节，并不意味着其中每个地址都已经被对象占用。

### 实际固定地址对象映射

以下表格按当前源码中的偏移宏、固定地址声明和对象长度整理。表内结束地址均包含在对象范围内。

| 起始地址 | 结束地址 | 大小 | 对象 |
|---|---|---:|---|
| `0x0200` | `0x020F` | 16 字节 | `OS_PROCESS_OK` |
| `0x0210` | `0x021F` | 16 字节 | `OS_JINCHENG_SP` |
| `0x0220` | `0x0221` | 2 字节 | `OS_KUAI_SP` |
| `0x0222` | `0x022B` | 10 字节 | `OS_FAST_CONTEXT` |
| `0x022C` | `0x02FB` | 208 字节 | `OS_TASK_CONTEXT` |
| `0x02FC` | `0x02FF` | 4 字节 | `OS_MEMORY` |
| `0x0300` | `0x030F` | 16 字节 | `OS_JINCHENG_JILU_SP` |
| `0x0310` | `0x0310` | 1 字节 | `OS_TIME_ONCE` |
| `0x0311` | `0x0314` | 4 字节 | `OS_WAIT_DI2` |
| `0x0315` | `0x0316` | 2 字节 | `OS_INTERRUPT_ADDR` |
| `0x0317` | `0x0318` | 2 字节 | `OS_INTERRUPT_ADDR2` |
| `0x0319` | `0x031A` | 2 字节 | `OS_TICK_COUNT_L` |
| `0x031B` | `0x031C` | 2 字节 | `OS_TICK_COUNT_H` |
| `0x031F` | `0x0320` | 2 字节 | `OS_SP_KUAI_BEI` |
| `0x0321` | `0x0322` | 2 字节 | `OS_SP_KUAI_BEI2` |
| `0x0324` | `0x0324` | 1 字节 | `OS_TIME_ONCE_BACKUP` |
| `0x0325` | `0x03A4` | 128 字节 | `OS_TASK` |
| `0x03A5` | `0x03C8` | 36 字节 | `OS_RES` |
| `0x03C9` | `0x03DD` | 21 字节 | `OS_MSGQ` |

这张表仅包含源码中实际声明的固定地址对象。`OS_EVENT_BIT`、`OS_STACK_WARN` 和 `OS_TASK_EVENT` 没有列为已分配对象，原因见第 6 节。

### 实际地址间隙

当前对象映射中存在以下未分配地址：

| 起始地址 | 结束地址 | 容量 | 说明 |
|---|---|---:|---|
| `0x031D` | `0x031E` | 2 字节 | 没有对应的固定地址对象声明 |
| `0x0323` | `0x0323` | 1 字节 | 没有对应的固定地址对象声明 |

因此，实际对象从 `0x0200` 连续布局至 `0x031C`，随后在 `0x031D–0x031E` 留有间隙；`0x031F–0x0322` 为高速任务栈指针备份；`0x0323` 为单字节间隙；`0x0324` 为时间备份，之后是任务控制块、资源对象及消息队列对象。

## 未实际分配的事件及栈警告区域

当前源码定义了以下偏移宏：

```c
#define OS_OFF_EVENT_BIT   0x011D
#define OS_OFF_STACK_WARN  0x011E
#define OS_OFF_TASK_EVENT  0x0123
```

这些宏本身只定义地址偏移，不会自动分配内存。

### `OS_EVENT_BIT`

其规划地址为：

```text
基地址：0x0200
偏移量：0x011D
计算地址：0x031D
```

当前所提供的源码没有出现对应的固定地址对象声明。因此，`0x031D` 不能直接标记为已经分配给 `OS_EVENT_BIT` 的地址。

### `OS_STACK_WARN`

其规划地址为：

```text
基地址：0x0200
偏移量：0x011E
计算地址：0x031E
```

当前所提供的源码没有出现对应的固定地址对象声明。因此，`0x031E` 当前属于未分配地址。

### `OS_TASK_EVENT`

其规划起始地址为：

```text
基地址：0x0200
偏移量：0x0123
计算地址：0x0323
```

源码没有提供对应的固定地址数组声明，但后续对象已经占用：

| 对象 | 实际地址范围 |
|---|---|
| `OS_TIME_ONCE_BACKUP` | `0x0324–0x0324` |
| `OS_TASK` | `0x0325–0x03A4` |

如果在 `0x0323` 声明 18 字节数组，其地址范围将是 `0x0323–0x0334`，会覆盖 `OS_TIME_ONCE_BACKUP`，并与 `OS_TASK` 的 `0x0325–0x0334` 范围重叠。

因此，**当前布局不能将 `OS_TASK_EVENT` 作为从 `0x0323` 开始的 18 字节独立对象使用。** 在重新确定地址并同步修改所有引用之前，应将该区域视为未分配的规划区域。

## 任务状态与上下文

### 任务状态数组

```c
volatile unsigned char xdata OS_PROCESS_OK[OS_PROCESS_MAX]
    _at_ (OS_SYS_GLOBAL_BASE + OS_OFF_PROCESS_OK);

volatile unsigned char xdata OS_JINCHENG_SP[OS_PROCESS_MAX]
    _at_ (OS_SYS_GLOBAL_BASE + OS_OFF_JINCHENG_SP);

volatile unsigned char xdata OS_KUAI_SP[OS_FAST_TASK_MAX]
    _at_ (OS_SYS_GLOBAL_BASE + OS_OFF_KUAI_SP);
```

当前配置：

```c
#define OS_PROCESS_MAX   16
#define OS_FAST_TASK_MAX 2
```

| 对象 | 容量 | 用途 |
|---|---:|---|
| `OS_PROCESS_OK` | 16 字节 | 普通任务启动状态及相关编码 |
| `OS_JINCHENG_SP` | 16 字节 | 普通任务栈指针记录 |
| `OS_KUAI_SP` | 2 字节 | 高速任务栈指针记录 |

`OS_PROCESS_OK` 不应简单理解为布尔数组。其位字段及编码方式必须与任务创建、等待、唤醒和调度实现保持一致。

### 任务上下文保存区域

```c
#define OS_OFF_FAST_CONTEXT 0x0022
#define OS_OFF_TASK_CONTEXT 0x002C

volatile unsigned char xdata
    OS_FAST_CONTEXT[OS_FAST_TASK_MAX][OS_KUAI_SHENDU]
    _at_ (OS_SYS_GLOBAL_BASE + OS_OFF_FAST_CONTEXT);

volatile unsigned char xdata
    OS_TASK_CONTEXT[OS_PROCESS_MAX][OS_JINCHENG_SHENDU]
    _at_ (OS_SYS_GLOBAL_BASE + OS_OFF_TASK_CONTEXT);
```

当前配置：

```c
#define OS_KUAI_SHENDU      5
#define OS_JINCHENG_SHENDU 13
```

高速任务上下文：

```text
2 × 5 = 10 字节
```

普通任务上下文：

```text
16 × 13 = 208 字节
```

两个上下文数组合计 218 字节。

上下文数组保存任务切换所需的运行现场数据；用户任务栈则提供任务执行期间的栈空间。两者用途不同，不能将上下文容量当作任务栈容量。

### 栈位图与栈指针备份

```c
#define OS_OFF_MEMORY           0x00FC
#define OS_OFF_JINCHENG_JILU_SP 0x0100
```

| 对象 | 地址范围 | 容量 | 用途 |
|---|---|---:|---|
| `OS_MEMORY` | `0x02FC–0x02FF` | 4 字节 | 用户栈分配位图 |
| `OS_JINCHENG_JILU_SP` | `0x0300–0x030F` | 16 字节 | 普通任务 SP 备份 |
| `OS_SP_KUAI_BEI` | `0x031F–0x0320` | 2 字节 | 高速任务 SP 起始地址记录 |
| `OS_SP_KUAI_BEI2` | `0x0321–0x0322` | 2 字节 | 高速任务 SP 大小记录 |

`OS_MEMORY` 共 32 位。按照每个栈块 2 字节计算，用户任务栈需要 31 个栈块，对应 31 个有效位；其余位的具体使用应以分配实现为准。

栈分配和释放必须采用一致的栈块索引规则，确保位图记录与实际栈地址对应。

## 时间、中断及调度数据

### 时间和高速任务延时

| 对象 | 地址范围 | 容量 | 用途 |
|---|---|---:|---|
| `OS_TIME_ONCE` | `0x0310–0x0310` | 1 字节 | 时间片相关数据 |
| `OS_WAIT_DI2` | `0x0311–0x0314` | 4 字节 | 两个高速任务的延时数据 |
| `OS_TIME_ONCE_BACKUP` | `0x0324–0x0324` | 1 字节 | 配合 MyOS 保存时间片相关状态 |

`OS_WAIT_DI2` 的类型为 `unsigned int[2]`。在每个 `unsigned int` 占用 2 字节的当前布局下，总计 4 字节。

### 中断地址记录

| 对象 | 地址范围 | 容量 |
|---|---|---:|
| `OS_INTERRUPT_ADDR` | `0x0315–0x0316` | 2 字节 |
| `OS_INTERRUPT_ADDR2` | `0x0317–0x0318` | 2 字节 |

两个对象分别提供中断处理相关的地址记录空间。具体编码方式及其在中断入口和恢复流程中的用途，应以实际中断处理代码为准。

### 系统时钟计数

| 对象 | 地址范围 | 容量 |
|---|---|---:|
| `OS_TICK_COUNT_L` | `0x0319–0x031A` | 2 字节 |
| `OS_TICK_COUNT_H` | `0x031B–0x031C` | 2 字节 |

每个计数变量为 `unsigned int`，两个变量合计 4 字节。变量名称表示高、低计数部分，但其组合顺序、进位处理和读取一致性应由计时实现确定。

8051 为 8 位处理器，多字节计数器的并发读写需要考虑中断更新造成的不一致风险。

## 任务控制块

### 固定地址与结构定义

```c
volatile OS_TCB xdata OS_TASK[OS_PROCESS_MAX]
    _at_ (OS_SYS_GLOBAL_BASE + 0x0125);
```

对应结构体：

```c
typedef struct
{
    u8 base_prio;
    u8 cur_prio;
    u8 state;

    u8 wait_type;
    u8 wait_flag;
    u8 wait_obj;

    u16 wait_tick;
} OS_TCB;
```

当前布局要求每个 `OS_TCB` 占用 8 字节。16 个任务控制块共占用 128 字节。

| 项目 | 数值 |
|---|---|
| 起始地址 | `0x0325` |
| 结束地址 | `0x03A4` |
| 对象数量 | 16 |
| 单个对象大小 | 8 字节 |
| 总容量 | 128 字节 |

任务控制块保存基本优先级、当前优先级、任务状态及统一等待信息。

等待字段包括等待类型、等待结果、等待对象 ID 和等待 tick。任务上下文、栈指针及用户栈空间记录保存在其他区域。

## 统一资源对象

### 固定地址与结构定义

```c
volatile OS_RESOURCE xdata OS_RES[OS_RESOURCE_MAX]
    _at_ (OS_SYS_GLOBAL_BASE + 0x01A5);
```

当前配置：

```c
#define OS_RESOURCE_MAX 6
```

结构体：

```c
typedef struct
{
    u8 value;
    u8 owner;

    u8 wait_cnt;
    u16 wait_mask;

    u8 pending_signal;
} OS_RESOURCE;
```

在当前 C51 编译布局中，每个资源对象按 6 字节计算，6 个对象合计 36 字节。

| 项目 | 数值 |
|---|---|
| 起始地址 | `0x03A5` |
| 结束地址 | `0x03C8` |
| 对象数量 | 6 |
| 单个对象大小 | 6 字节 |
| 总容量 | 36 字节 |

字段用途：

- `value`：资源状态或计数。
- `owner`：需要所有者信息时记录所有者。
- `wait_cnt`：等待任务数量。
- `wait_mask`：等待任务位图。
- `pending_signal`：待处理信号计数。

`wait_mask` 为 16 位，可以对应普通任务 ID `0–15`。等待数量与等待位图必须在任务等待、唤醒及清理过程中保持一致。

## 消息队列对象

### 固定地址与结构定义

```c
volatile os_msgq_t xdata OS_MSGQ[OS_MSGQ_MAX]
    _at_ (OS_SYS_GLOBAL_BASE + 0x01C9);
```

当前配置：

```c
#define OS_MSGQ_MAX 3
```

结构体：

```c
typedef struct
{
    u8 *buf;
    u8 _size;
    u8 head;
    u8 tail;
    u8 count;
} os_msgq_t;
```

在 XDATA 指针占用 3 字节、其余字段各占 1 字节且没有额外填充的当前布局下，每个对象占用 7 字节，3 个对象合计 21 字节。

| 项目 | 数值 |
|---|---|
| 起始地址 | `0x03C9` |
| 结束地址 | `0x03DD` |
| 对象数量 | 3 |
| 单个对象大小 | 7 字节 |
| 总容量 | 21 字节 |

消息队列对象保存缓冲区指针、容量、读写位置和当前消息数量。实际消息缓冲区由 `buf` 指向，其内存不包含在这 21 字节的控制对象容量中。

## XDATA 内存容量统计

### 实际对象容量

按当前声明的数组长度和结构体大小统计：

| 对象组 | 占用字节 |
|---|---:|
| 任务状态及栈指针记录 | 34 |
| 高速任务与普通任务上下文 | 218 |
| 用户栈位图 | 4 |
| 时间、延时、中断及 tick 数据 | 11 |
| 高速任务栈指针备份 | 4 |
| 时间备份 | 1 |
| 普通任务控制块 | 128 |
| 统一资源对象 | 36 |
| 消息队列控制对象 | 21 |
| **实际对象总计** | **457** |

上述分组中的 11 字节由 `OS_TIME_ONCE` 1 字节、`OS_WAIT_DI2` 4 字节、两个中断地址数组 4 字节及两个 tick 计数变量 4 字节构成，合计应为 13 字节；因此，正式统计应采用逐对象求和结果，而不能直接沿用未经复核的分组数字。

按全部实际对象逐项求和，正确的已分配容量为 **475 字节**。

### 地址范围与剩余空间

| 项目 | 数值 |
|---|---|
| 全局池总容量 | 512 字节 |
| 实际对象容量 | 475 字节 |
| 未分配容量 | 37 字节 |
| 最后一个已分配对象结束地址 | `0x03DD` |
| 末尾连续空闲区 | `0x03DE–0x03FF`，34 字节 |
| 内部地址间隙 | 3 字节 |

内部地址间隙由 `0x031D–0x031E` 的 2 字节和 `0x0323` 的 1 字节构成。

因此，当前全局池中共有 475 字节被实际对象占用，37 字节未分配。其中 34 字节位于池末尾，另有 3 字节位于内部地址间隙。

以上统计不包含应用程序独立提供的消息缓冲区、其他模块自行分配的 XDATA 数据，以及编译器未按固定地址放置的变量。

## 设计约束

### 固定地址约束

- 每个固定地址对象必须与其偏移宏及数组长度一致。
- 不得将只有偏移宏、没有实际声明的对象计入已分配容量。
- 不得让数组范围与后续固定地址对象重叠。
- 修改结构体字段、数组长度或配置参数后，必须重新计算全部后续地址。
- 最终布局应通过 Keil C51 编译结果和链接映射验证。

### 栈管理约束

- 系统栈、中断栈和用户任务栈必须保持独立。
- 用户栈有效范围为 `0x42–0x7F`。
- 用户栈按 2 字节一块计算，共有 31 个栈块。
- 栈分配与释放必须采用一致的位图索引规则。
- 实际栈使用情况还应结合任务创建参数和运行时栈深度分析。

### 任务及内核对象约束

- 普通任务状态、任务控制块、任务上下文和栈记录承担不同职责。
- 高速任务与普通任务使用不同的上下文和栈管理数据。
- 资源等待位图、等待计数和任务控制块等待字段必须保持一致。
- 消息队列控制对象与消息数据缓冲区应分别统计。
- 任务删除时必须同步清理任务状态、栈分配记录及等待对象关联信息。

### 芯片与编译器约束

移植到其他 8051 器件时，应确认：

- 内部 RAM 容量及高地址 IDATA 的实际可用性。
- XDATA 容量及固定地址的可访问性。
- C51 存储类型、指针宽度和结构体实际大小。
- 中断入口、寄存器保护及上下文恢复代码。
- 固定地址对象是否与最终链接映射一致。

## 模块总结

HRTOS 当前内存布局采用固定地址管理内核变量、系统栈、中断栈、用户任务栈、任务上下文及主要内核对象。

内部 RAM 中，系统栈占用 `0x35–0x39`，中断栈占用 `0x3A–0x41`，用户任务栈占用 `0x42–0x7F`。IDATA 中断保护与调度状态占用 `0xDB–0xE7`。

XDATA 全局池的规划范围为 `0x0200–0x03FF`，总容量 512 字节。按当前源码实际声明的固定地址对象统计，已分配容量为 475 字节，未分配容量为 37 字节。

本布局以当前源码为依据。任何涉及对象地址、对象大小或内存容量的修改，都必须同步检查固定地址声明、偏移宏、结构体定义及最终链接映射，确保实际内存分配不存在冲突。