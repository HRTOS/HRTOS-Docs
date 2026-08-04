# 系统配置模块

## 模块简介

系统配置模块定义 HRTOS 的所有系统配置参数、数据类型和核心数据结构。该模块是整个系统的基础，决定了系统的资源分配、任务数量、调度策略等关键参数。

### 设计目的

- 定义系统基本数据类型
- 配置系统资源参数
- 定义任务状态和等待类型
- 定义核心数据结构
- 提供系统配置接口

### 使用场景

- 系统定制和裁剪
- 资源分配配置
- 任务数量调整
- 调度策略配置
- 应用程序开发参考

## 数据类型

### 基本类型

| 类型定义 | 原类型 | 说明 |
|---|---|---|
| u8 | unsigned char | 8位无符号整数 |
| u16 | unsigned int | 16位无符号整数 |
| u32 | unsigned long | 32位无符号整数 |
| f32 | float | 32位浮点数 |
| f64 | double | 64位浮点数 |
| NULL | 0 | 空指针 |

### OS_TCB

任务控制块结构：

```c
typedef struct {
    u8 id;              /* 任务 ID */
    u8 base_prio;       /* 静态优先级 */
    u8 cur_prio;        /* 动态优先级 */
    u8 state;           /* 任务状态 */
    u8 wait_type;       /* 等待类型 */
    u8 wait_flag;       /* 等待结果 */
    u8 wait_obj;        /* 等待对象 ID */
    u16 wait_tick;      /* 超时等待 tick */
} OS_TCB;
```

| 成员 | 类型 | 说明 |
|---|---|---|
| id | u8 | 任务 ID |
| base_prio | u8 | 静态优先级 |
| cur_prio | u8 | 动态优先级 |
| state | u8 | 任务状态 |
| wait_type | u8 | 等待类型 |
| wait_flag | u8 | 等待结果 |
| wait_obj | u8 | 等待对象 ID |
| wait_tick | u16 | 超时等待 tick |

### OS_RESOURCE

统一资源对象结构：

```c
typedef struct {
    u8 value;           /* 资源值 */
    u8 owner;           /* 当前拥有者 */
    u8 wait_cnt;        /* 当前等待任务数量 */
    u16 wait_mask;      /* 位图等待队列 */
    u8 pending_signal;  /* ISR 中断计数 */
} OS_RESOURCE;
```

| 成员 | 类型 | 说明 |
|---|---|---|
| value | u8 | 资源值 |
| owner | u8 | 当前拥有者（互斥锁专用） |
| wait_cnt | u8 | 当前等待任务数量 |
| wait_mask | u16 | 位图等待队列 |
| pending_signal | u8 | ISR 中断计数 |

### os_msgq_t

消息队列控制块结构：

```c
typedef struct {
    u8 *buf;            /* 环形消息缓冲区指针 */
    u8 _size;           /* 队列最大容量 */
    u8 head;            /* 写指针（生产者） */
    u8 tail;            /* 读指针（消费者） */
    u8 count;           /* 当前消息数量 */
} os_msgq_t;
```

| 成员 | 类型 | 说明 |
|---|---|---|
| buf | u8* | 环形消息缓冲区指针 |
| _size | u8 | 队列最大容量 |
| head | u8 | 写指针（生产者） |
| tail | u8 | 读指针（消费者） |
| count | u8 | 当前消息数量 |

## 宏定义

### 系统配置宏

| 宏定义 | 默认值 | 功能说明 |
|---|---|---|
| OS_TIME_ONCE_DEFAULT | 5 | 时间片数量 |
| OS_TIME_T0 | 10000 | 单个时间片长度 |

### 任务配置宏

| 宏定义 | 默认值 | 功能说明 |
|---|---|---|
| OS_PROCESS_MAX | 16 | 标准任务数量最大值 |
| OS_FAST_TASK_MAX | 2 | 高速任务数量最大值 |
| OS_SCHED_LEVEL_MAX | 10 | 调度级别最大值 |
| OS_MAX_TASK | 16 | 最大任务数（等于 OS_PROCESS_MAX） |

### 资源配置宏

| 宏定义 | 默认值 | 功能说明 |
|---|---|---|
| OS_RESOURCE_MAX | 8 | 统一资源对象最大数量 |
| OS_MSGQ_MAX | 4 | 消息队列对象最大数量 |

### 任务状态宏

| 宏定义 | 值 | 说明 |
|---|---|---|
| READY | 0 | 就绪态 |
| WAIT | 1 | 等待态 |
| RUNNING | 2 | 运行态 |
| SUSPEND | 3 | 挂起态 |
| DEAD | 4 | 删除态 |

### 等待类型宏

| 宏定义 | 值 | 说明 |
|---|---|---|
| WAIT_NONE | 0 | 无等待 |
| WAIT_DELAY | 1 | 延时等待 |
| WAIT_SEM | 2 | 信号量等待 |
| WAIT_MUTEX | 3 | 互斥锁等待 |
| WAIT_MSG_SEND | 4 | 消息队列发送等待 |
| WAIT_MSG_RECV | 5 | 消息队列接收等待 |
| WAIT_EVENT | 6 | 事件等待 |
| WAIT_MAILBOX | 7 | 邮箱等待 |

### 唤醒标志宏

| 宏定义 | 值 | 说明 |
|---|---|---|
| WAIT_TIMEOUT | 1 | 超时唤醒 |
| WAIT_SIGNAL | 2 | 信号唤醒 |

### 其他宏

| 宏定义 | 值 | 说明 |
|---|---|---|
| OS_INVALID_ID | 0xFF | 无效 ID |
| OS_ENTER_CRITICAL() | do{ os_enter_critical(); }while(0) | 进入临界区宏 |
| OS_EXIT_CRITICAL() | do{ os_exit_critical(); }while(0) | 退出临界区宏 |

## 全局变量

### 任务控制块数组

```c
extern OS_TCB xdata OS_TASK[OS_PROCESS_MAX];
```

任务控制块数组，存储所有任务的状态信息。

### 统一资源对象数组

```c
extern xdata OS_RESOURCE OS_RES[OS_RESOURCE_MAX];
```

统一资源对象数组，用于信号量、互斥锁、事件、邮箱等。

### 消息队列对象数组

```c
extern xdata os_msgq_t OS_MSGQ[OS_MSGQ_MAX];
```

消息队列对象数组，存储所有消息队列的控制信息。

### 时间片备份

```c
extern unsigned char xdata OS_TIME_ONCE_BACKUP;
```

配合 HRTOS 与 MYOS 调度策略切换的时间片备份。

## 调度级别定义

HRTOS 采用 10 级优先级设计：

| 优先级范围 | 类型 | 说明 |
|---|---|---|
| 0-7 | 普通任务 | 标准任务优先级，数值越大优先级越高 |
| 8 | 高优先级任务 | 高速任务，响应更快 |
| 9 | 中断级 | 中断处理函数优先级 |
| 10 | 中断嵌套级 | 高优先级中断嵌套处理 |

## 配置说明

### 任务数量配置

- **OS_PROCESS_MAX**：标准任务最大数量，默认为 16
- **OS_FAST_TASK_MAX**：高速任务最大数量，默认为 2
- 根据实际应用需求调整，影响内存占用

### 时间配置

- **OS_TIME_ONCE_DEFAULT**：时间片数量，默认为 5
- **OS_TIME_T0**：单个时间片长度，默认为 10000
- 影响调度精度和系统响应时间

### 资源配置

- **OS_RESOURCE_MAX**：统一资源对象最大数量，默认为 8
- **OS_MSGQ_MAX**：消息队列对象最大数量，默认为 4
- 根据应用需求调整，影响内存占用

### 内存布局

系统采用精心设计的内存布局：

- **DATA 区**：内核临时寄存器
- **IDATA 高区**：中断现场保护、调度核心变量
- **XDATA 区**：任务控制块、资源对象、消息队列

## 配置建议

### 任务数量配置

- 根据实际任务需求设置 OS_PROCESS_MAX
- 高速任务数量通常较少（1-2个）
- 过多任务会增加内存开销和调度开销

### 时间片配置

- 实时性要求高的应用：减少时间片数量
- 公平性要求高的应用：增加时间片数量
- 时间片长度影响系统响应精度

### 资源配置

- 根据同步和通信需求设置 OS_RESOURCE_MAX
- 根据消息传递需求设置 OS_MSGQ_MAX
- 合理配置可优化内存使用

## 注意事项

- 修改配置后需要重新编译系统
- 配置参数影响系统性能和内存占用
- 建议根据实际应用需求进行裁剪
- 修改任务数量需要相应调整内存布局
