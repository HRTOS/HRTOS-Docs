# HRTOS 内核概述

## 模块介绍

HRTOS（硬实时操作系统）是专为 8051 系列微控制器设计的轻量级实时操作系统。内核提供基本的 RTOS 服务，包括任务管理、进程间通信（IPC）、定时服务和中断处理，同时保持最小的内存占用，适合资源受限的嵌入式系统。

## 主要职责

HRTOS 内核负责：

- **任务管理**：创建、调度和管理多个并发任务
- **调度**：基于优先级的抢占式调度，可选时间片模式
- **IPC 服务**：事件标志、消息队列、互斥锁、信号量和邮箱
- **定时服务**：系统时钟生成、延时函数和运行时间跟踪
- **中断管理**：ISR 安全 API 和临界区保护
- **内存管理**：静态内存分配，使用位图跟踪栈空间
- **上下文切换**：调度期间保存和恢复任务上下文

## 主要文件

### 头文件（Inc/）

- `hrtos.h`：包含所有公共 API 的主头文件
- `config.h`：系统配置、类型定义和数据结构
- `hrtos_internal.h`：内核内部声明和内存布局
- `kernel.h`：内核初始化和核心 API
- `task.h`：任务管理 API
- `event.h`：事件同步 API
- `interrupt.h`：中断处理和临界区 API
- `mutex.h`：互斥锁（互斥）API
- `semaphore.h`：信号量 API
- `time.h`：定时和延时 API
- `wait.h`：统一等待机制
- `mailbox.h`：邮箱 API
- `msgq.h`：消息队列 API

### 源文件（Src/）

#### 内核模块（Src/kernel/）
- `os_core.c`：核心内核数据结构和内存布局
- `scheduler.c`：调度器选择和模式切换
- `scheduler_dispatch.c`：任务分发机制
- `scheduler_mode.c`：调度模式配置
- `task_create.c`：任务创建和注册
- `task_cleanup.c`：任务清理和资源释放
- `idle_task.c`：空闲任务实现
- `idle_hook_register.c`：空闲钩子注册
- `os_data.c`：DATA 段初始化
- `os_xdata.c`：XDATA 段初始化

#### 任务模块（Src/task/）
- `task_delete.c`：任务删除
- `task_exit.c`：任务退出
- `task_is_valid.c`：任务有效性检查
- `task_resume.c`：任务恢复
- `task_scheduler.c`：任务调度器
- `task_self.c`：获取当前任务 ID
- `task_state.c`：任务状态查询
- `task_suspend.c`：任务挂起
- `task_suspend_self.c`：自挂起
- `task_yield.c`：任务让出

#### 事件模块（Src/event/）
- `event_delete.c`：事件删除
- `event_init.c`：事件初始化
- `event_query.c`：事件查询
- `event_wait.c`：事件等待
- `event_write.c`：事件写入

#### 中断模块（Src/interrupt/）
- `enter_critical.c`：进入临界区
- `event_set_from_isr.c`：ISR 安全的事件设置
- `exit_critical.c`：退出临界区
- `in_isr.c`：ISR 上下文检查
- `interrupt_id.c`：中断 ID 查询
- `msgq_recv_from_isr.c`：ISR 安全的消息接收
- `msgq_send_from_isr.c`：ISR 安全的消息发送
- `queue_send_from_isr.c`：ISR 安全的队列发送
- `schedule_request.c`：调度请求
- `semaphore_post_from_isr.c`：ISR 安全的信号量释放

#### 互斥锁模块（Src/mutex/）
- `mutex_init.c`：互斥锁初始化
- `mutex_lock.c`：具有优先级继承的互斥锁加锁
- `mutex_unlock.c`：互斥锁解锁

#### 信号量模块（Src/semaphore/）
- `sem_init.c`：信号量初始化
- `sem_post.c`：信号量释放（V 操作）
- `sem_wait.c`：信号量等待（P 操作）

#### 定时模块（Src/time/）
- `delay.c`：任务延时
- `delay_ms.c`：毫秒延时（忙等待）
- `ms_to_tick.c`：毫秒到时钟周期转换
- `sec_to_tick.c`：秒到时钟周期转换
- `tick_config.c`：时钟配置
- `tick_get.c`：获取时钟周期计数
- `uptime_ms.c`：获取运行时间（毫秒）

#### 等待模块（Src/wait/）
- `os_wait.c`：统一等待核心
- `res_init.c`：资源初始化
- `wake_task.c`：任务唤醒

#### 邮箱模块（Src/mailbox/）
- `mailbox_clear.c`：邮箱清除
- `mailbox_init.c`：邮箱初始化
- `mailbox_query.c`：邮箱查询
- `mailbox_recv.c`：邮箱接收
- `mailbox_send.c`：邮箱发送

#### 消息队列模块（Src/msgq/）
- `msgq_clear.c`：消息队列清除
- `msgq_init.c`：消息队列初始化
- `msgq_recv.c`：消息队列接收
- `msgq_send.c`：消息队列发送

#### 任务优先级模块（Src/task_priority/）
- `lock.c`：优先级锁定
- `lock_query.c`：优先级锁定查询
- `task_get_priority.c`：获取任务优先级
- `task_set_priority.c`：设置任务优先级
- `unlock.c`：优先级解锁

#### 移植模块（Src/port/）
- `cpu_nop.c`：CPU 空操作

## 数据结构

### OS_TCB（任务控制块）

位于 `config.h`，任务控制块包含：

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

### OS_RESOURCE（统一 IPC 资源）

位于 `config.h`，统一资源对象：

```c
typedef struct {
    u8 value;           /* 资源值 */
    u8 owner;           /* 当前所有者（仅互斥锁） */
    u8 wait_cnt;        /* 等待任务计数 */
    u16 wait_mask;      /* 位图等待队列 */
    u8 pending_signal;  /* ISR 信号计数 */
} OS_RESOURCE;
```

### os_msgq_t（消息队列）

位于 `config.h`：

```c
typedef struct {
    u8 *buf;            /* 缓冲区指针 */
    u8 _size;           /* 缓冲区大小 */
    u8 head;            /* 写指针（生产者） */
    u8 tail;            /* 读指针（消费者） */
    u8 count;           /* 当前计数 */
} os_msgq_t;
```

## 核心函数

### 内核初始化

- `os_interrupt_init()`：初始化中断系统
- `os_idle_task()`：系统空闲任务
- `hrtos_main()`：用户定义的主函数

### 任务管理

- `os_task_create()`：创建并注册任务
- `os_task_cleanup()`：清理任务资源
- `os_task_exit()`：退出当前任务
- `os_task_delete()`：删除任务

### 调度

- `os_set_scheduler()`：设置调度器模式（HRTOS/MYOS）
- `os_scheduler_mode_switch()`：切换调度模式
- `os_schedule_request()`：请求重新调度

### 等待机制

- `os_wait()`：统一等待核心
- `wake_task()`：唤醒任务

### 临界区

- `os_enter_critical()`：进入临界区
- `os_exit_critical()`：退出临界区

## 调用关系

### 初始化流程

```
系统复位
    ↓
os_idle_task() [入口点]
    ↓
初始化 TCB 数组
初始化资源数组
初始化消息队列
    ↓
hrtos_main() [用户初始化]
    ↓
os_task_create() 调用
    ↓
OS_CFG_LOCK = 1（锁定注册）
    ↓
启用中断（EA = 1）
    ↓
带有任务清理的空闲循环
```

### 任务创建流程

```
os_task_create(addr, id, pr, sd)
    ↓
参数验证
    ↓
栈分配（位图搜索）
    ↓
栈指针计算
    ↓
上下文初始化
    ↓
任务状态设置
    ↓
返回成功
```

### 等待流程

```
os_wait(type, obj, tick)
    ↓
进入临界区
    ↓
设置 TCB 等待字段
    ↓
添加到资源等待队列（如适用）
    ↓
设置任务状态为 WAIT
    ↓
触发调度
    ↓
退出临界区
    ↓
任务阻塞直到调用 wake_task()
```

## 生命周期

### 系统生命周期

1. **启动**：系统进入 `os_idle_task()`
2. **初始化**：内核结构初始化
3. **用户初始化**：调用 `hrtos_main()` 创建任务
4. **注册锁定**：任务注册锁定
5. **调度**：启用中断，调度器激活
6. **运行时**：任务基于优先级执行
7. **空闲**：空闲任务在其他任务未就绪时运行

### 任务生命周期

1. **创建**：`os_task_create()` 分配栈并初始化 TCB
2. **就绪**：任务标记为可调度
3. **运行**：任务被调度器选中时执行
4. **等待**：任务在 IPC 或延时上阻塞
5. **唤醒**：`wake_task()` 将任务返回到就绪状态
6. **退出**：`os_task_exit()` 或 `os_task_delete()` 移除任务

## 设计原则

### 内存效率

- 仅静态分配（无动态内存）
- 基于位图的栈管理
- 紧凑的数据结构
- XDATA 段用于大型数组

### 实时性能

- 基于优先级的抢占式调度
- 快速上下文切换
- 最小中断延迟
- ISR 安全 API

### 8051 优化

- DATA 段用于频繁访问的变量
- IDATA 用于内核状态
- XDATA 用于大型数据结构
- 关键路径的汇编优化

### 统一 IPC 模型

- 所有 IPC 类型使用单一 `OS_RESOURCE` 结构
- 通过 `os_wait()` 统一等待机制
- 跨 IPC 类型的一致 API 模式

## 约束

- 最多 16 个标准任务
- 最多 2 个快速任务
- 最多 8 个 IPC 资源
- 最多 4 个消息队列
- 优先级级别 0-10
- 无动态内存分配
- 无递归互斥锁支持
- 栈溢出保护有限
