# 中断管理模块

## 模块简介

中断管理模块提供中断相关的控制功能，包括临界区保护、中断环境判断、中断安全 API 等。该模块确保中断与任务间的安全通信和操作。

### 设计目的

- 提供临界区保护机制
- 支持中断环境判断
- 提供中断安全 API
- 实现中断与任务间安全通信
- 支持中断嵌套

### 使用场景

- 临界区保护
- 中断服务程序
- 中断与任务通信
- 中断安全资源操作
- 中断环境判断

## 数据类型

本模块无特殊数据类型定义。

## 宏定义

本模块使用 config.h 中定义的临界区宏：

| 宏定义 | 功能说明 |
|---|---|
| OS_ENTER_CRITICAL() | 进入临界区（关中断） |
| OS_EXIT_CRITICAL() | 退出临界区（恢复中断） |

## API 函数列表

| 函数名称 | 功能说明 |
|---|---|
| os_interrupt_id | 获取中断编号 |
| os_enter_critical | 进入临界区（关中断） |
| os_exit_critical | 退出临界区（恢复中断） |
| os_in_isr | 判断当前运行环境 |
| os_queue_send_from_isr | 中断安全邮箱发送 |
| os_schedule_request | 请求系统调度 |
| os_event_set_from_isr | 中断安全事件设置 |
| os_semaphore_post_from_isr | 中断安全信号量释放 |
| os_msgq_send_from_isr | 中断安全消息队列发送 |

## API 详细说明

### os_interrupt_id

函数原型：

```c
char os_interrupt_id();
```

参数说明：

无

返回值：

- 中断编号（0, 2-31 根据自然入口顺序排列）

功能说明：

获取当前中断编号。工作在寄存器3，返回中断编号。中断编号根据自然入口顺序排列。

调用示例：

```c
char irq_id = os_interrupt_id();
```

注意事项：

- 必须在中断上下文中调用
- 返回值对应 51 单片机中断向量
- 用于中断服务程序识别

### os_enter_critical

函数原型：

```c
void os_enter_critical();
```

参数说明：

无

返回值：

无

功能说明：

关中断，保护运行不被打断。支持中断嵌套，使用嵌套计数器管理：

1. 检查嵌套计数（os_critical_nesting）
2. 如果是第一层（os_critical_nesting == 0）：
   - 保存当前中断状态（os_int_status = EA）
   - 关中断（EA = 0）
3. 嵌套计数加1（os_critical_nesting++）

调用示例：

```c
os_enter_critical();
// 临界区代码
os_exit_critical();
```

注意事项：

- 必须与 os_exit_critical 配对使用
- 支持嵌套调用
- 嵌套层数有限制
- 临界区代码应尽可能简短
- 可以使用宏 OS_ENTER_CRITICAL()

### os_exit_critical

函数原型：

```c
void os_exit_critical();
```

参数说明：

无

返回值：

无

功能说明：

恢复中断，退出临界区。支持中断嵌套，使用嵌套计数器管理：

1. 嵌套计数减1（os_critical_nesting--）
2. 如果是最后一层（os_critical_nesting == 0）：
   - 恢复中断状态（EA = os_int_status）

调用示例：

```c
os_enter_critical();
// 临界区代码
os_exit_critical();
```

注意事项：

- 必须与 os_enter_critical 配对使用
- 支持嵌套调用
- 不能单独调用（未配对）
- 可以使用宏 OS_EXIT_CRITICAL()

### os_in_isr

函数原型：

```c
char os_in_isr(void);
```

参数说明：

无

返回值：

- 1：当前在中断中运行
- 0：当前在任务中运行

功能说明：

判断当前代码是在任务里运行，还是在中断里运行。通过检查 OS_ISR_FLAG 标志判断。

调用示例：

```c
if (os_in_isr()) {
    // 当前在中断中
} else {
    // 当前在任务中
}
```

注意事项：

- 用于区分运行环境
- 中断安全 API 内部使用
- 返回值基于系统标志

### os_queue_send_from_isr

函数原型：

```c
char os_queue_send_from_isr(u8 id, u8 _data);
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| id | u8 | 邮箱编号 |
| _data | u8 | 写入数据 |

返回值：

- 发送结果

功能说明：

中断安全邮箱发送。在中断中向邮箱发送数据，确保中断环境下的安全操作。

调用示例：

```c
char result = os_queue_send_from_isr(0, 0x55);
```

注意事项：

- 必须在中断上下文中调用
- 采用延迟唤醒机制
- 触发调度请求
- 与 os_mailbox_send 对应

### os_schedule_request

函数原型：

```c
char os_schedule_request(char id);
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| id | char | 调度参数 |

返回值：

- 调度请求结果

功能说明：

通知系统重新调度。在中断中请求系统进行任务调度。

调用示例：

```c
os_schedule_request(1);
```

注意事项：

- 通常在中断服务程序结束时调用
- 触发调度器检查
- 可能导致任务切换

### os_event_set_from_isr

函数原型：

```c
char os_event_set_from_isr(u8 id);
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| id | u8 | 事件 ID |

返回值：

- 设置结果

功能说明：

中断里设置事件标志，通知任务某个条件成立了。中断安全的事件触发操作。

调用示例：

```c
char result = os_event_set_from_isr(0);
```

注意事项：

- 必须在中断上下文中调用
- 采用延迟唤醒机制
- 与 os_event_write 对应
- 会触发调度请求

### os_semaphore_post_from_isr

函数原型：

```c
char os_semaphore_post_from_isr(u8 obj);
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| obj | u8 | 资源对象 ID |

返回值：

- 1：成功
- -1：参数错误

功能说明：

中断里释放/发送一个信号量，通知某个等待信号量的任务继续运行。采用延迟唤醒机制：

**有等待任务时：**
- 仅记录事件（pending_signal++）
- 触发调度（OS_SCHED_REASON = 1）
- 实际唤醒由内核统一处理

**无等待任务时：**
- 计数+1（value++）

调用示例：

```c
char result = os_semaphore_post_from_isr(0);
```

注意事项：

- 必须在中断上下文中调用
- 采用延迟唤醒机制，避免中断中复杂操作
- 与 os_sem_post 对应
- 会触发调度请求

### os_msgq_send_from_isr

函数原型：

```c
char os_msgq_send_from_isr(os_msgq_t *q, u8 obj, u8 _data);
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| q | os_msgq_t* | 消息队列控制块 |
| obj | u8 | 资源对象 ID |
| _data | u8 | 待发送数据 |

返回值：

- 发送结果

功能说明：

中断安全消息队列发送。在中断中向消息队列发送数据，确保中断环境下的安全操作。

调用示例：

```c
char result = os_msgq_send_from_isr(&my_queue, 0, 0x55);
```

注意事项：

- 必须在中断上下文中调用
- 采用延迟唤醒机制
- 与 os_msgq_send 对应
- 会触发调度请求
