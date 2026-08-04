# 任务管理模块

## 模块简介

任务管理模块提供完整的任务生命周期管理功能，包括任务创建、删除、挂起、恢复、优先级调整等操作。该模块是 HRTOS 的核心组件之一，负责管理多任务并发执行。

### 设计目的

- 提供任务状态管理功能
- 支持任务优先级动态调整
- 实现任务挂起和恢复机制
- 提供任务查询和控制接口
- 支持任务锁机制

### 使用场景

- 动态创建和删除任务
- 任务优先级调整
- 任务暂停和恢复
- 任务状态查询
- 任务间同步控制

## 数据类型

本模块无特殊数据类型定义，使用 config.h 中定义的 OS_TCB 结构。

## 宏定义

本模块无公开宏定义。

## API 函数列表

| 函数名称 | 功能说明 |
|---|---|
| os_task_suspend | 挂起任务 |
| os_task_get_state | 获取任务状态 |
| os_task_set_next | 指定下一个运行任务 |
| os_task_resume | 恢复挂起任务 |
| os_task_exit | 退出当前任务 |
| os_task_delete | 删除任务 |
| os_task_set_priority | 设置任务优先级 |
| os_task_get_priority | 获取任务优先级 |
| os_task_is_valid | 检查任务是否有效 |
| os_task_self | 获取当前任务 ID |
| os_task_yield | 让出 CPU |
| os_task_suspend_self | 主动挂起当前任务 |
| os_lock | 任务加锁 |
| os_unlock | 任务解锁 |
| os_lock_query | 查询锁状态 |

## API 详细说明

### os_task_suspend

函数原型：

```c
char os_task_suspend(unsigned char id);
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| id | unsigned char | 任务编号 |

返回值：

- 1：挂起成功
- -1：参数错误

功能说明：

将指定任务的状态变为挂起状态。挂起的任务暂停运行，不参与调度，直到被恢复。

调用示例：

```c
char result = os_task_suspend(0);
if (result == 1) {
    // 任务 0 已挂起
}
```

注意事项：

- 任务编号必须在合法范围内
- 挂起当前任务需要使用 os_task_suspend_self
- 挂起的任务不会占用 CPU 资源

### os_task_get_state

函数原型：

```c
char os_task_get_state(unsigned char id);
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| id | unsigned char | 任务编号 |

返回值：

- 0：删除态（DEAD）
- 1：运行态（RUNNING）
- 2：就绪态（READY）
- 3：挂起态（SUSPEND）
- 4：超时态（上一个结束的任务）
- -1：参数错误

功能说明：

查询指定任务的当前状态。根据任务控制块和系统状态寄存器判断任务状态。

调用示例：

```c
char state = os_task_get_state(0);
if (state == 2) {
    // 任务处于就绪态
}
```

注意事项：

- 任务编号必须在合法范围内
- 返回值与 config.h 中定义的状态宏对应

### os_task_set_next

函数原型：

```c
char os_task_set_next(unsigned char id);
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| id | unsigned char | 任务编号 |

返回值：

- 1：设置成功
- -1：参数错误

功能说明：

指定下一个运行的任务，终止当前调度并强行切换到指定任务。该函数会强制触发任务切换。

调用示例：

```c
os_task_set_next(1);  // 强制切换到任务 1
```

注意事项：

- 会立即触发任务切换
- 指定的任务必须处于就绪态
- 谨慎使用，可能影响实时性

### os_task_resume

函数原型：

```c
char os_task_resume(unsigned char id);
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| id | unsigned char | 任务编号 |

返回值：

- 1：恢复成功
- -1：参数错误

功能说明：

将指定任务从挂起状态恢复为就绪状态，使其重新参与调度。

调用示例：

```c
char result = os_task_resume(0);
if (result == 1) {
    // 任务 0 已恢复
}
```

注意事项：

- 任务必须处于挂起态才能恢复
- 恢复后的任务会进入就绪队列
- 可能立即触发调度

### os_task_exit

函数原型：

```c
void os_task_exit();
```

参数说明：

无

返回值：

无

功能说明：

立即退出当前任务。任务会从系统中删除，释放相关资源。

调用示例：

```c
void task_func() {
    // 任务逻辑
    if (error_condition) {
        os_task_exit();  // 退出任务
    }
}
```

注意事项：

- 调用后任务立即终止
- 会释放任务占用的堆栈资源
- 不能在任务外部调用

### os_task_delete

函数原型：

```c
char os_task_delete(unsigned char id);
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| id | unsigned char | 任务编号 |

返回值：

- 1：删除成功
- -1：参数错误

功能说明：

彻底删除指定任务，从任务队列中移除并释放内存资源。

调用示例：

```c
char result = os_task_delete(0);
if (result == 1) {
    // 任务 0 已删除
}
```

注意事项：

- 不能删除当前运行的任务
- 删除后任务编号可以重新使用
- 会释放任务堆栈资源

### os_task_set_priority

函数原型：

```c
char os_task_set_priority(unsigned char id, unsigned char pr);
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| id | unsigned char | 任务编号 |
| pr | unsigned char | 新优先级（0-7 自然优先级，8 高优先级） |

返回值：

- 1：设置成功
- -1：参数错误

功能说明：

修改指定任务的优先级。优先级范围：
- 0-7：普通任务优先级（数值越大优先级越高）
- 8：高优先级任务

调用示例：

```c
char result = os_task_set_priority(0, 5);  // 将任务 0 优先级设为 5
```

注意事项：

- 优先级必须在合法范围内
- 修改优先级可能立即触发调度
- 高优先级任务会抢占低优先级任务

### os_task_get_priority

函数原型：

```c
char os_task_get_priority(unsigned char id);
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| id | unsigned char | 任务编号 |

返回值：

- 任务优先级（0-7 自然优先级，8 高优先级）
- -1：参数错误

功能说明：

查询指定任务的当前优先级。

调用示例：

```c
char prio = os_task_get_priority(0);
```

注意事项：

- 任务编号必须在合法范围内
- 返回值反映任务的静态优先级

### os_task_is_valid

函数原型：

```c
signed char os_task_is_valid(u8 id);
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| id | u8 | 任务编号 |

返回值：

- -1：参数异常
- 0：任务未注册
- 1：任务已注册

功能说明：

检查指定任务是否已注册/存在。用于验证任务编号的有效性。

调用示例：

```c
signed char valid = os_task_is_valid(0);
if (valid == 1) {
    // 任务 0 已注册
}
```

注意事项：

- 常用于参数检查
- 可以避免对无效任务进行操作

### os_task_self

函数原型：

```c
u8 os_task_self(void);
```

参数说明：

无

返回值：

- 当前任务的 ID

功能说明：

获取当前运行任务的编号。

调用示例：

```c
u8 my_id = os_task_self();
```

注意事项：

- 只能在任务上下文中调用
- 返回值可用于任务识别

### os_task_yield

函数原型：

```c
void os_task_yield();
```

参数说明：

无

返回值：

无

功能说明：

当前任务从运行态转为就绪态，主动让出 CPU 并触发调度。适用于需要公平共享 CPU 的场景。

调用示例：

```c
void task_func() {
    while (1) {
        // 任务逻辑
        os_task_yield();  // 让出 CPU
    }
}
```

注意事项：

- 任务会重新进入就绪队列
- 可能立即被重新调度
- 不会改变任务优先级

### os_task_suspend_self

函数原型：

```c
void os_task_suspend_self();
```

参数说明：

无

返回值：

无

功能说明：

当前任务主动挂起自己，让出 CPU 并触发调度。挂起后需要其他任务调用 os_task_resume 才能恢复。

调用示例：

```c
void task_func() {
    // 任务逻辑
    os_task_suspend_self();  // 主动挂起
}
```

注意事项：

- 只能在任务上下文中调用
- 挂起后需要其他任务恢复
- 谨慎使用，可能导致任务无法恢复

### os_lock

函数原型：

```c
void os_lock(void);
```

参数说明：

无

返回值：

无

功能说明：

对注册任务进行加锁，防止任务被修改或删除。

调用示例：

```c
os_lock();
// 临界区代码
os_unlock();
```

注意事项：

- 必须与 os_unlock 配对使用
- 避免长时间持有锁
- 可能影响系统调度

### os_unlock

函数原型：

```c
void os_unlock(void);
```

参数说明：

无

返回值：

无

功能说明：

解除注册任务的锁状态，允许任务被修改或删除。

调用示例：

```c
os_lock();
// 临界区代码
os_unlock();
```

注意事项：

- 必须与 os_lock 配对使用
- 解锁未加锁的任务可能导致错误

### os_lock_query

函数原型：

```c
char os_lock_query();
```

参数说明：

无

返回值：

- 1：锁定状态
- 0：未锁定

功能说明：

查询当前任务的锁状态。

调用示例：

```c
char locked = os_lock_query();
if (locked == 1) {
    // 任务处于锁定状态
}
```

注意事项：

- 用于检查锁状态
- 可以避免重复加锁
