# HRTOS 目录分析

## 模块介绍

本文档全面分析了 HRTOS 项目目录结构，解释了源文件、头文件、文档和示例的组织方式。

## 主要职责

目录结构用于：

- 分离公共 API 和内部实现
- 按功能组织内核模块
- 提供面向用户和内部代码之间的清晰边界
- 通过硬件抽象层保持可移植性
- 支持教学和开发工作流程

### Inc/ 目录（公共头文件）

**目的**：包含所有公共 API 声明和配置定义。此目录对用户是只读的，定义了应用程序和内核之间的接口。

**文件**：

- `hrtos.h`：包含所有其他头文件的主包含文件
- `config.h`：系统配置、类型定义和数据结构
- `kernel.h`：内核初始化和核心 API
- `task.h`：任务管理 API
- `event.h`：事件同步 API
- `interrupt.h`：中断处理和临界区 API
- `mutex.h`：互斥锁 API
- `semaphore.h`：信号量 API
- `time.h`：定时和延时 API
- `wait.h`：统一等待机制
- `mailbox.h`：邮箱 API
- `msgq.h`：消息队列 API

**关键特性**：

- 定义公共 API 签名
- 包含配置宏
- 暴露数据结构定义
- 无实现细节
- 应用程序代码包含安全

### Src/ 目录（源实现）

**目的**：包含按功能模块组织的所有内核实现。此目录对用户是只读的，包含实际的内核代码。

**子目录**：

#### Src/kernel/ - 核心内核

**文件**：
- `os_core.c`：核心数据结构、内存布局定义
- `scheduler.c`：HRTOS 和 MYOS 模式之间的调度器选择
- `scheduler_dispatch.c`：任务分发实现
- `scheduler_mode.c`：调度模式切换
- `task_create.c`：任务创建和注册逻辑
- `task_cleanup.c`：任务清理和资源释放
- `idle_task.c`：空闲任务实现
- `idle_hook_register.c`：空闲钩子注册
- `os_data.c`：DATA 数据搬运
- `os_xdata.c`：XDATA 数据搬运

**职责**：
- 系统初始化
- 任务生命周期管理
- 调度核心逻辑
- 内存布局定义
- 空闲任务管理

#### Src/task/ - 任务管理

**文件**：
- `task_delete.c`：任务删除实现
- `task_exit.c`：当前任务退出
- `task_is_valid.c`：任务有效性验证
- `task_resume.c`：恢复挂起的任务
- `task_scheduler.c`：任务调度逻辑
- `task_self.c`：获取当前任务 ID
- `task_state.c`：查询任务状态
- `task_suspend.c`：挂起任务
- `task_suspend_self.c`：自挂起
- `task_yield.c`：让出 CPU 给其他任务

**职责**：
- 任务状态转换
- 任务控制操作
- 优先级管理
- 状态查询

#### Src/event/ - 事件同步

**文件**：
- `event_delete.c`：删除事件对象
- `event_init.c`：初始化事件
- `event_query.c`：查询事件状态
- `event_wait.c`：等待事件
- `event_write.c`：设置/写入事件

**职责**：
- 事件标志管理
- 事件同步
- ISR 安全事件操作

#### Src/interrupt/ - 中断处理

**文件**：
- `enter_critical.c`：进入临界区（禁用中断）
- `event_set_from_isr.c`：ISR 安全的事件设置
- `exit_critical.c`：退出临界区（恢复中断）
- `in_isr.c`：检查是否在 ISR 上下文中
- `interrupt_id.c`：获取当前中断 ID
- `msgq_recv_from_isr.c`：ISR 安全的消息接收
- `msgq_send_from_isr.c`：ISR 安全的消息发送
- `queue_send_from_isr.c`：ISR 安全的队列发送
- `schedule_request.c`：请求任务重新调度
- `semaphore_post_from_isr.c`：ISR 安全的信号量释放

**职责**：
- 临界区管理
- ISR 上下文检测
- ISR 安全 IPC 操作
- 中断驱动的调度

#### Src/mutex/ - 互斥

**文件**：
- `mutex_init.c`：初始化互斥锁
- `mutex_lock.c`：具有优先级继承的互斥锁获取
- `mutex_unlock.c`：释放互斥锁

**职责**：
- 互斥锁状态管理
- 优先级继承实现
- 死锁预防

#### Src/semaphore/ - 计数信号量

**文件**：
- `sem_init.c`：初始化信号量
- `sem_post.c`：信号量 V 操作（释放）
- `sem_wait.c`：信号量 P 操作（获取）

**职责**：
- 信号量计数/资源管理
- 资源获取/释放
- 等待队列管理

#### Src/time/ - 定时服务

**文件**：
- `delay.c`：基于时钟周期的任务延时
- `delay_ms.c`：忙等待毫秒延时
- `ms_to_tick.c`：将毫秒转换为时钟周期
- `sec_to_tick.c`：将秒转换为时钟周期
- `tick_config.c`：配置系统时钟
- `tick_get.c`：获取当前时钟周期计数
- `uptime_ms.c`：获取系统运行时间（毫秒）

**职责**：
- 系统时钟管理
- 时间转换工具
- 延时实现
- 运行时间跟踪

#### Src/wait/ - 统一等待

**文件**：
- `os_wait.c`：统一等待核心实现
- `res_init.c`：资源初始化
- `wake_task.c`：任务唤醒实现

**职责**：
- 统一等待机制
- 任务唤醒逻辑
- 资源初始化

#### Src/mailbox/ - 邮箱 IPC

**文件**：
- `mailbox_clear.c`：清除邮箱
- `mailbox_init.c`：初始化邮箱
- `mailbox_query.c`：查询邮箱状态
- `mailbox_recv.c`：从邮箱接收
- `mailbox_send.c`：发送到邮箱

**职责**：
- 邮箱状态管理
- 单字节消息传递
- 邮箱同步

#### Src/msgq/ - 消息队列

**文件**：
- `msgq_clear.c`：清除消息队列
- `msgq_init.c`：初始化消息队列
- `msgq_recv.c`：从消息队列接收
- `msgq_send.c`：发送到消息队列

**职责**：
- 消息队列管理
- 循环缓冲区实现
- 生产者-消费者同步

#### Src/task_priority/ - 优先级管理

**文件**：
- `lock.c`：锁定任务注册
- `lock_query.c`：查询优先级锁定状态
- `task_get_priority.c`：获取任务优先级
- `task_set_priority.c`：设置任务优先级
- `unlock.c`：解锁任务注册

**职责**：
- 任务注册锁定机制
- 优先级查询和修改
- 优先级更改保护

#### Src/port/ - 硬件抽象

**文件**：
- `cpu_nop.c`：CPU 空操作指令

**职责**：
- 硬件特定操作
- 不同 MCU 的移植层

## 核心函数

### 文件包含层次

1. **应用程序层**：包含 `Inc/hrtos.h`
2. **公共头文件层**：`hrtos.h` 包含所有其他公共头文件
3. **实现层**：源文件包含 `hrtos.h`

### 模块初始化顺序

基于目录结构，初始化遵循：

1. 核心内核（Src/kernel/）
2. 任务管理（Src/task/）
3. IPC 模块（Src/event/、Src/mutex/、Src/semaphore/ 等）
4. 定时（Src/time/）
5. 中断处理（Src/interrupt/）

## 调用关系

### 模块间依赖

```
Src/kernel/
    ├── 使用 → Src/wait/
    ├── 使用 → Src/task/
    └── 使用 → Src/interrupt/

Src/task/
    ├── 使用 → Src/wait/
    └── 使用 → Src/task_priority/

Src/event/
    └── 使用 → Src/wait/

Src/mutex/
    └── 使用 → Src/wait/

Src/semaphore/
    └── 使用 → Src/wait/

Src/mailbox/
    └── 使用 → Src/wait/

Src/msgq/
    └── 使用 → Src/wait/

Src/interrupt/
    ├── 使用 → Src/event/
    ├── 使用 → Src/msgq/
    └── 使用 → Src/semaphore/

Src/time/
    └── 使用 → Src/wait/
```

### 头文件依赖图

```
hrtos.h（主入口）
    ├── config.h
    ├── wait.h
    ├── event.h
    ├── interrupt.h
    ├── kernel.h
    ├── mailbox.h
    ├── mutex.h
    ├── task.h
    ├── semaphore.h
    ├── msgq.h
    └── time.h
```
