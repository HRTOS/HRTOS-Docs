# HRTOS 调用流程

## 模块介绍

调用流程描述了 HRTOS 中的执行路径，展示了函数在各种操作期间如何相互调用。理解调用流程对于调试、性能分析和系统理解至关重要。

## 主要职责

调用流程文档涵盖：

- 任务创建和删除流程
- 调度和上下文切换流程
- IPC 操作流程
- 中断处理流程
- 定时器和延时流程

## 主要文件

### 关键流程文件

- `Src/kernel/task_create.c`：任务创建流程
- `Src/kernel/task_cleanup.c`：任务清理流程
- `Src/wait/os_wait.c`：等待流程
- `Src/wait/wake_task.c`：唤醒流程
- `Src/interrupt/schedule_request.c`：调度触发流程
- `Src/kernel/scheduler.c`：调度器选择流程

## 任务创建流程

### 完整流程图

```mermaid
sequenceDiagram
    participant App as 应用程序
    participant TC as task_create
    participant BM as 位图管理器
    participant SP as 栈指针
    participant TCB as TCB 设置
    participant Sched as 调度器

    App->>TC: os_task_create(addr, id, pr, sd)
    TC->>TC: 验证参数
    TC->>TC: 检查优先级范围
    TC->>TC: 检查重复
    TC->>BM: 搜索位图中的空闲块
    BM->>BM: 查找连续空闲块
    BM->>TC: 返回块起始
    TC->>TC: 计算初始 SP
    TC->>TC: 检查栈溢出
    TC->>SP: 将任务地址压入栈
    TC->>TCB: 设置 OS_PROCESS_OK
    TC->>TCB: 设置 OS_JINCHENG_SP
    TC->>TCB: 设置 OS_JINCHENG_JILU_SP
    TC->>Sched: 设置就绪标志
    TC-->>App: 返回成功
```

### 详细步骤

1. **参数验证**
   - 检查优先级范围（0-10）
   - 检查任务 ID 有效性
   - 检查地址不为 NULL
   - 检查重复注册

2. **栈分配**
   - 搜索位图中的空闲块
   - 查找匹配大小的连续块
   - 在位图中标记块为已使用
   - 计算初始栈指针

3. **栈初始化**
   - 保存当前 SP
   - 将 SP 设置为任务栈
   - 压入任务地址（低字节）
   - 压入任务地址（高字节）
   - 恢复 SP

4. **TCB 设置**
   - 在 `OS_PROCESS_OK` 中设置优先级
   - 设置就绪标志
   - 记录栈指针
   - 记录初始栈指针
   - 对于快速任务：设置上下文和标志

5. **完成**
   - 返回成功（1）
   - 任务现在准备好调度

## 任务删除流程

### 完整流程图

```mermaid
sequenceDiagram
    participant App as 应用程序
    participant TD as task_delete
    participant Idle as 空闲任务
    participant TC as task_cleanup
    participant BM as 位图管理器
    participant Res as 资源管理器
    participant Sched as 调度器

    App->>TD: os_task_delete(id)
    TD->>TD: 设置删除请求标志
    TD->>Sched: 触发调度
    Sched->>Idle: 上下文切换到空闲
    Idle->>Idle: 检查删除请求
    Idle->>TC: os_task_cleanup(id)
    TC->>TC: 验证任务 ID
    TC->>TC: 计算栈块
    TC->>BM: 释放位图块
    TC->>TC: 清除上下文保存区域
    TC->>TC: 清除 OS_PROCESS_OK
    TC->>Res: 从等待队列中移除
    TC->>Res: 释放互斥锁所有权
    TC->>TC: 清除 TCB 等待字段
    TC->>TC: 设置状态为 DEAD
    TC-->>Idle: 返回成功
    Idle->>Idle: 继续空闲循环
```

### 详细步骤

1. **删除请求**
   - 设置 `OS_EVENT_DELETE_REQ` 标志
   - 触发调度
   - 任务继续直到被抢占

2. **空闲任务检测**
   - 空闲任务扫描删除请求
   - 为每个请求调用 `os_task_cleanup()`

3. **栈释放**
   - 计算栈块位置
   - 清除位图中的位
   - 释放栈空间

4. **上下文清理**
   - 清除上下文保存区域
   - 清除栈指针
   - 清除任务状态

5. **资源清理**
   - 从所有等待队列中移除
   - 释放互斥锁所有权
   - 清除挂起操作

6. **TCB 清理**
   - 清除 wait_type
   - 清除 wait_flag
   - 清除 wait_obj
   - 清除 wait_tick
   - 设置状态为 DEAD

## 调度流程

### 完整流程图

```mermaid
sequenceDiagram
    participant Task as 运行任务
    participant Wait as os_wait
    participant TCB as TCB 更新
    participant Res as 资源队列
    participant ISR as Timer0 ISR
    participant Sched as 调度器
    participant Next as 下一个任务

    Task->>Wait: os_wait(type, obj, tick)
    Wait->>Wait: 进入临界区
    Wait->>TCB: 设置 wait_type、wait_obj、wait_tick
    Wait->>TCB: 设置状态为 WAIT
    Wait->>Res: 添加到 wait_mask（如适用）
    Wait->>TCB: 清除就绪标志
    Wait->>ISR: 设置 OS_SCHED_REASON=1、TF0=1
    Wait->>Wait: 退出临界区
    Wait-->>Task: 返回（任务阻塞）
    
    Note over Task,ISR: 发生上下文切换
    
    ISR->>ISR: Timer0 溢出
    ISR->>ISR: 检查 OS_SCHED_REASON
    ISR->>Sched: 调用调度器
    Sched->>Sched: 扫描就绪任务
    Sched->>Sched: 选择最高优先级
    Sched->>Next: 上下文切换到下一个任务
    Next->>Next: 执行任务代码
```

### 详细步骤

1. **阻塞操作**
   - 任务调用阻塞 API
   - API 调用 `os_wait()`
   - 进入临界区

2. **等待设置**
   - 设置 TCB 等待字段
   - 添加到资源等待队列
   - 设置任务状态为 WAIT
   - 清除就绪标志

3. **调度触发**
   - 设置 `OS_SCHED_REASON = 1`
   - 设置 `TF0 = 1`（触发 Timer0）
   - 退出临界区

4. **上下文切换**
   - Timer0 ISR 触发
   - 调度器扫描任务
   - 选择最高优先级的就绪任务
   - 发生上下文切换

5. **任务执行**
   - 下一个任务运行
   - 前一个任务阻塞

## 唤醒流程

### 完整流程图

```mermaid
sequenceDiagram
    participant ISR as ISR/Post
    participant Res as 资源
    participant Wake as wake_task
    participant TCB as TCB 更新
    participant Sched as 调度器
    participant Task as 阻塞任务

    ISR->>Res: Post 操作（sem_post、mutex_unlock 等）
    Res->>Res: 更新资源状态
    Res->>Wake: wake_task(tid, flag)
    Wake->>Res: 从 wait_mask 中移除
    Wake->>Res: 递减 wait_cnt
    Wake->>TCB: 设置 wait_flag
    Wake->>TCB: 清除 wait_type
    Wake->>TCB: 清除 wait_obj
    Wake->>TCB: 清除 wait_tick
    Wake->>TCB: 设置状态为 READY
    Wake->>TCB: 设置就绪标志
    Wake->>Sched: 设置 OS_SCHED_REASON=1、TF0=1
    Wake-->>Res: 返回
    
    Note over Res,Task: 发生上下文切换
    
    Sched->>Task: 任务被选中
    Task->>Task: 恢复执行
    Task->>Task: 检查 wait_flag
    Task->>Task: 从 os_wait 返回
```

### 详细步骤

1. **资源操作**
   - ISR 或任务执行 post 操作
   - 资源状态更新
   - 识别等待任务

2. **唤醒任务调用**
   - 调用 `wake_task(tid, flag)`
   - 临界区保护

3. **队列清理**
   - 从 wait_mask 中移除任务
   - 递减 wait_cnt
   - 清除资源关联

4. **TCB 更新**
   - 设置 wait_flag（TIMEOUT 或 SIGNAL）
   - 清除 wait_type
   - 清除 wait_obj
   - 清除 wait_tick
   - 设置状态为 READY
   - 在 `OS_PROCESS_OK` 中设置就绪标志

5. **调度触发**
   - 设置 `OS_SCHED_REASON = 1`
   - 设置 `TF0 = 1`
   - 任务将被调度

## 中断处理流程

### 完整流程图

```mermaid
sequenceDiagram
    participant HW as 硬件
    participant ISR as ISR 处理程序
    participant API as ISR 安全 API
    participant Res as 资源
    participant Sched as 调度器
    participant Task as 等待任务

    HW->>ISR: 发生中断
    ISR->>ISR: 保存上下文
    ISR->>API: 调用 ISR 安全 API（event_set_from_isr 等）
    API->>Res: 更新资源值
    API->>Res: 递增 pending_signal
    API->>Sched: os_schedule_request()
    Sched->>Sched: 设置 OS_SCHED_REASON=1
    Sched->>Sched: 设置 TF0=1
    API-->>ISR: 返回
    ISR->>ISR: 恢复上下文
    ISR-->>HW: 返回
    
    Note over HW,Task: Timer0 中断触发
    
    Sched->>Task: 唤醒任务
    Task->>Task: 处理事件
```

### 详细步骤

1. **中断入口**
   - 发生硬件中断
   - 硬件保存上下文
   - ISR 处理程序执行

2. **ISR 安全 API 调用**
   - 事件设置、信号量释放或消息发送
   - 资源值更新
   - 挂起信号递增

3. **调度请求**
   - 调用 `os_schedule_request()`
   - 设置调度触发
   - 触发 Timer0 中断

4. **ISR 退出**
   - 上下文恢复
   - 从中断返回

5. **延迟调度**
   - Timer0 中断触发
   - 调度器处理唤醒
   - 任务恢复

## 延时流程

### 完整流程图

```mermaid
sequenceDiagram
    participant Task as 任务
    participant Delay as os_delay
    participant Wait as os_wait
    participant TCB as TCB 更新
    participant Timer as Timer0 ISR
    participant Tick as 时钟处理程序
    participant Wake as wake_task

    Task->>Delay: os_delay(tick)
    Delay->>Wait: os_wait(WAIT_DELAY, INVALID_ID, tick)
    Wait->>Wait: 进入临界区
    Wait->>TCB: 设置 wait_type=WAIT_DELAY
    Wait->>TCB: 设置 wait_tick=tick
    Wait->>TCB: 设置状态=WAIT
    Wait->>TCB: 清除就绪标志
    Wait->>Timer: 设置 OS_SCHED_REASON=1、TF0=1
    Wait->>Wait: 退出临界区
    Wait-->>Delay: 返回
    Delay-->>Task: 返回（任务阻塞）
    
    Note over Task,Timer: 任务阻塞，定时器中断
    
    Timer->>Tick: 时钟周期发生
    Tick->>TCB: 递减 wait_tick
    Tick->>Tick: wait_tick == 0?
    Tick->>Wake: 是，调用 wake_task
    Wake->>TCB: 设置状态=READY
    Wake->>TCB: 设置就绪标志
    Wake->>Timer: 设置 OS_SCHED_REASON=1
    Wake-->>Tick: 返回
    Tick-->>Timer: 返回
    
    Note over Timer,Task: 调度器选择任务
    
    Task->>Task: 恢复执行
```

### 详细步骤

1. **延时调用**
   - 任务调用 `os_delay(tick)`
   - 调用 `os_wait(WAIT_DELAY, INVALID_ID, tick)`

2. **等待设置**
   - 设置 wait_type 为 WAIT_DELAY
   - 设置 wait_tick 为 tick 值
   - 无资源对象（INVALID_ID）
   - 设置状态为 WAIT
   - 清除就绪标志

3. **调度触发**
   - 设置调度触发
   - 任务阻塞

4. **时钟周期处理**
   - 每次 Timer0 中断
   - 递减 wait_tick
   - 检查 wait_tick == 0

5. **唤醒**
   - 当 wait_tick 达到 0
   - 使用 WAIT_TIMEOUT 调用 `wake_task()`
   - 任务返回到 READY 状态

6. **任务恢复**
   - 调度器选择任务
   - 任务恢复执行
   - 从 `os_delay()` 返回

## 互斥锁加锁流程

### 完整流程图

```mermaid
sequenceDiagram
    participant Task as 任务
    participant Lock as mutex_lock
    participant Res as 资源
    participant Prio as 优先级继承
    participant Wait as os_wait
    participant TCB as TCB 更新

    Task->>Lock: os_mutex_lock(mid)
    Lock->>Lock: 验证参数
    Lock->>Res: 检查所有者
    Res-->>Lock: 所有者状态
    Lock->>Lock: 当前任务所有者?
    Lock-->>Task: 是，返回错误（无递归）
    
    alt 互斥锁空闲
        Lock->>Res: 设置所有者 = 当前任务
        Lock->>TCB: 设置 base_prio
        Lock->>TCB: 设置 cur_prio
        Lock-->>Task: 返回成功
    else 互斥锁被持有
        Lock->>Prio: 检查优先级
        Prio->>TCB: 获取当前任务优先级
        Prio->>TCB: 获取所有者优先级
        alt 等待任务优先级更高
            Prio->>TCB: 提升所有者 cur_prio
            Prio->>TCB: 更新 OS_PROCESS_OK
        end
        Lock->>Wait: os_wait(WAIT_MUTEX, mid, 0)
        Wait->>TCB: 设置 wait_type=WAIT_MUTEX
        Wait->>Res: 添加到 wait_mask
        Wait->>TCB: 设置状态=WAIT
        Lock-->>Task: 返回（进入等待队列）
    end
```

### 详细步骤

1. **加锁请求**
   - 任务调用 `os_mutex_lock(mid)`
   - 验证互斥锁 ID

2. **递归检查**
   - 检查当前任务是否为所有者
   - 如果是，返回错误（无递归锁）

3. **空闲互斥锁**
   - 如果互斥锁空闲（所有者 = INVALID_ID）
   - 将所有者设置为当前任务
   - 设置任务优先级
   - 立即返回成功

4. **被持有的互斥锁**
   - 如果互斥锁被另一个任务持有
   - 检查优先级继承
   - 如果等待任务优先级更高：
     - 提升所有者优先级
     - 更新所有者的 `OS_PROCESS_OK`
   - 调用 `os_wait(WAIT_MUTEX, mid, 0)`
   - 添加到等待队列
   - 返回（任务阻塞）

## 信号量等待流程

### 完整流程图

```mermaid
sequenceDiagram
    participant Task as 任务
    participant Wait as sem_wait
    participant Res as 资源
    participant WaitCore as os_wait
    participant TCB as TCB 更新

    Task->>Wait: os_sem_wait(sid)
    Wait->>Res: 检查值
    Res-->>Wait: 值状态
    
    alt value > 0（快速路径）
        Wait->>Res: 递减值
        Wait-->>Task: 返回 WAIT_SIGNAL
    else value == 0（慢速路径）
        Wait->>WaitCore: os_wait(WAIT_SEM, sid, 0)
        WaitCore->>TCB: 设置 wait_type=WAIT_SEM
        WaitCore->>Res: 添加到 wait_mask
        WaitCore->>TCB: 设置状态=WAIT
        WaitCore->>TCB: 清除就绪标志
        WaitCore-->>Wait: 返回（阻塞）
        Wait-->>Task: 返回任务 ID
    end
```

### 详细步骤

1. **信号量等待**
   - 任务调用 `os_sem_wait(sid)`
   - 进入临界区

2. **快速路径检查**
   - 检查 value > 0
   - 如果是：
     - 递减值
     - 立即返回 WAIT_SIGNAL
     - 无阻塞，无上下文切换

3. **慢速路径**
   - 如果 value == 0
   - 调用 `os_wait(WAIT_SEM, sid, 0)`
   - 设置 wait_type
   - 添加到 wait_mask
   - 设置状态为 WAIT
   - 任务阻塞

4. **唤醒**
   - 当调用 `os_sem_post()` 时
   - 使用 WAIT_SIGNAL 唤醒任务
   - 从 `os_sem_wait()` 返回

## 消息队列发送流程

### 完整流程图

```mermaid
sequenceDiagram
    participant Task as 任务
    participant Send as msgq_send
    participant Queue as 队列结构
    participant Wait as os_wait
    participant Wake as wake_task

    Task->>Send: os_msgq_send(q, obj, data, nonblock)
    Send->>Queue: 检查计数与大小
    Queue-->>Send: 队列状态
    
    alt 队列未满
        Send->>Queue: 在 head 写入数据
        Send->>Queue: 递增 head
        Send->>Queue: 递增 count
        Send->>Queue: 接收者等待?
        alt 是
            Send->>Wake: wake_task(receiver_id)
            Wake->>Queue: 从 wait_mask 中移除
            Wake->>Queue: 设置状态=READY
        end
        Send-->>Task: 返回成功
    else 队列已满
        alt nonblock == 1
            Send-->>Task: 返回失败
        else nonblock == 0
            Send->>Wait: os_wait(WAIT_MSG_SEND, obj, 0)
            Wait->>Queue: 添加到 wait_mask
            Wait->>Queue: 设置状态=WAIT
            Send-->>Task: 返回（阻塞）
        end
    end
```

### 详细步骤

1. **发送请求**
   - 任务调用 `os_msgq_send()`
   - 检查队列状态

2. **队列未满**
   - 在 head 处将数据写入缓冲区
   - 递增 head（带环绕）
   - 递增 count
   - 检查接收者是否等待
   - 如果是，唤醒接收者
   - 返回成功

3. **队列已满**
   - 检查 nonblock 标志
   - 如果非阻塞：返回失败
   - 如果阻塞：
     - 调用 `os_wait(WAIT_MSG_SEND, obj, 0)`
     - 添加到发送者等待队列
     - 任务阻塞

4. **唤醒**
   - 当接收者消费消息时
   - 空间变为可用
   - 发送者被唤醒
   - 发送操作完成

## 调用流程摘要

### 常见模式

1. **阻塞操作**：所有阻塞操作通过 `os_wait()`
2. **唤醒**：所有唤醒通过 `wake_task()`
3. **调度**：通过 Timer0 中断触发调度
4. **临界区**：所有状态更改由临界区保护

### 性能考虑

- **快速路径**：许多操作具有快速路径优化
- **上下文切换**：阻塞操作导致上下文切换
- **临界区**：临界区中的时间最小
- **ISR 安全**：ISR 操作延迟到任务上下文

### 调试调用流程

- **跟踪函数调用**：在关键函数处添加断点
- **监视状态**：观察 TCB 和资源状态更改
- **检查触发**：验证调度触发
- **验证转换**：确保状态转换正确
