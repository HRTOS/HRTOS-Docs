# 统一等待模块

## 模块简介

统一等待模块是 HRTOS 的核心组件，提供统一的任务阻塞与唤醒机制。所有阻塞操作（延时、信号量、互斥锁、消息队列、事件、邮箱）都通过该模块实现。

### 设计目的

- 提供统一的任务阻塞接口
- 简化同步机制实现
- 统一管理等待队列
- 支持超时机制
- 实现任务唤醒逻辑

### 使用场景

- 任务延时等待
- 资源等待（信号量、互斥锁）
- 消息等待（消息队列、邮箱）
- 事件等待
- 所有需要任务阻塞的场景

## 数据类型

### OS_RESOURCE

统一资源对象结构（在 config.h 中定义）：

```c
typedef struct {
    u8 value;          /* 资源值 */
    u8 owner;          /* 当前拥有者（互斥锁专用） */
    u8 wait_cnt;       /* 当前等待任务数量 */
    u16 wait_mask;     /* 位图等待队列 */
    u8 pending_signal; /* ISR 中断计数 */
} OS_RESOURCE;
```

| 成员 | 类型 | 说明 |
|---|---|---|
| value | u8 | 资源值 |
| owner | u8 | 当前拥有者（互斥锁专用） |
| wait_cnt | u8 | 当前等待任务数量 |
| wait_mask | u16 | 位图等待队列 |
| pending_signal | u8 | ISR 中断计数 |

## 宏定义

本模块使用 config.h 中定义的等待类型和唤醒标志宏：

### 等待类型

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

### 唤醒标志

| 宏定义 | 值 | 说明 |
|---|---|---|
| WAIT_TIMEOUT | 1 | 超时唤醒 |
| WAIT_SIGNAL | 2 | 信号唤醒 |

## API 函数列表

| 函数名称 | 功能说明 |
|---|---|
| os_wait | 统一等待核心 |
| os_res_init | 资源对象初始化 |
| wake_task | 唤醒任务 |

## API 详细说明

### os_wait

函数原型：

```c
u8 os_wait(u8 type, u8 obj, u16 tick);
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| type | u8 | 等待类型（WAIT_*） |
| obj | u8 | 等待对象 ID（资源编号） |
| tick | u16 | 超时时间（tick 数） |

返回值：

- WAIT_TIMEOUT：超时唤醒
- WAIT_SIGNAL：资源/事件唤醒
- 0：等待失败

功能说明：

统一等待核心，所有阻塞操作统一进入该函数。支持的等待类型：

1. 延时等待（WAIT_DELAY）
2. 信号量等待（WAIT_SEM）
3. 互斥锁等待（WAIT_MUTEX）
4. 消息发送等待（WAIT_MSG_SEND）
5. 消息接收等待（WAIT_MSG_RECV）
6. 事件等待（WAIT_EVENT）
7. 邮箱等待（WAIT_MAILBOX）

**延时等待处理：**
- tick = 0 时直接返回 WAIT_TIMEOUT
- 高速任务独立处理，不进入统一等待队列
- 标准任务设置等待信息并触发调度

**资源等待处理：**
- 检查任务是否已处于等待状态（不允许重复等待）
- 检查资源 ID 是否合法
- 设置等待信息（wait_type、wait_obj、wait_tick、wait_flag）
- 加入资源等待队列（wait_mask）
- 阻塞当前任务（state = WAIT）
- 触发任务切换

**等待队列管理：**
- 使用位图管理等待任务（wait_mask）
- bit0 对应 task0，bit1 对应 task1，...
- WAIT_DELAY 属于纯时间等待，不进入资源等待队列

**任务切换：**
- 设置 OS_SCHED_REASON = 1 触发调度
- 设置 TF0 = 1 触发 T0 中断
- 任务被唤醒后返回等待结果

调用示例：

```c
// 延时等待
u8 result = os_wait(WAIT_DELAY, OS_INVALID_ID, 100);

// 信号量等待
result = os_wait(WAIT_SEM, 0, 0);

// 事件等待（带超时）
result = os_wait(WAIT_EVENT, 0, 100);
```

注意事项：

- 不允许重复进入等待
- 资源 ID 必须合法（非延时等待）
- 会阻塞当前任务
- 被唤醒后返回等待结果
- 高速任务延时独立处理

### os_res_init

函数原型：

```c
void os_res_init(OS_RESOURCE *r);
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| r | OS_RESOURCE* | 资源对象指针 |

返回值：

无

功能说明：

初始化资源对象，清空所有字段：

1. 清空资源值（r->value = 0）
2. 清空拥有者（r->owner = OS_INVALID_ID）
3. 清空等待计数（r->wait_cnt = 0）
4. 清空等待位图（r->wait_mask = 0）

调用示例：

```c
OS_RESOURCE my_res;
os_res_init(&my_res);
```

注意事项：

- 用于初始化统一资源对象
- 信号量、互斥锁、事件、邮箱都使用该函数
- 初始化后资源为空闲状态

### wake_task

函数原型：

```c
void wake_task(u8 tid, u8 flag);
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| tid | u8 | 任务编号 |
| flag | u8 | 唤醒标志（WAIT_TIMEOUT/WAIT_SIGNAL） |

返回值：

无

功能说明：

唤醒指定任务，执行以下操作：

1. 保存等待对象（obj = OS_TASK[tid].wait_obj）
2. 清理等待位图（OS_RES[obj].wait_mask &= ~((u16)1 << tid)）
3. 减少等待计数（OS_RES[obj].wait_cnt--）
4. 设置唤醒结果（OS_TASK[tid].wait_flag = flag）
5. 清理等待上下文（wait_type、wait_obj、wait_tick）
6. 设置就绪态（OS_TASK[tid].state = READY）
7. 允许调度（OS_PROCESS_OK[tid] |= 1）
8. 触发调度（OS_SCHED_REASON = 1, TF0 = 1）

调用示例：

```c
wake_task(0, WAIT_SIGNAL);  // 唤醒任务 0，信号唤醒
```

注意事项：

- 任务必须处于等待状态
- 会清理任务的所有等待信息
- 会触发任务调度
- flag 必须是合法的唤醒标志
