# 内核模块

## 模块简介

内核模块是 HRTOS 的核心组件，提供系统初始化、任务创建、调度控制、内存管理等基础功能。该模块负责系统的启动、中断处理、任务调度等关键操作。

### 设计目的

- 提供系统初始化和启动功能
- 实现任务注册和创建机制
- 控制调度模式切换
- 提供系统级钩子函数
- 管理内存操作

### 使用场景

- 系统启动时初始化
- 创建用户任务
- 切换调度模式
- 注册系统钩子函数
- 系统级异常处理

## 数据类型

本模块无特殊数据类型定义。

## 宏定义

本模块无公开宏定义。

## API 函数列表

| 函数名称 | 功能说明 |
|---|---|
| os_dispatch_select | 调度器选择中断号 |
| os_task_create | 任务创建 |
| os_interrupt_init | 中断初始化 |
| os_interrupt_exit | 中断退出 |
| os_memset | 内存填充 |
| os_data_to_xdata | 数据区到扩展数据区拷贝 |
| os_xdata_to_data | 扩展数据区到数据区拷贝 |
| os_scheduler_mode_switch | 调度模式切换 |
| os_set_scheduler | 调度器选择 |
| hrtos_main | HRTOS 默认用户主函数 |
| os_slab_overflow | 系统溢出处理 |
| os_slab_overflow_register | 注册溢出钩子函数 |
| os_task_cleanup | 系统任务清理 |
| os_idle_task | 系统 idle 任务 |
| os_idle_hook_register | 注册 idle 钩子函数 |

## API 详细说明

### os_dispatch_select

函数原型：

```c
char os_dispatch_select(unsigned char interrupt_id);
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| interrupt_id | unsigned char | 中断编号 |

返回值：

- 返回调度器选择的中断号

功能说明：

调度器选择中断号，用于系统中断分发机制。该函数根据传入的中断 ID 选择相应的处理路径。

调用示例：

```c
char id = os_dispatch_select(2);
```

注意事项：

- 需要在中断上下文中调用
- 中断编号必须在合法范围内

### os_task_create

函数原型：

```c
char os_task_create(unsigned int addr, unsigned char id, unsigned char pr, unsigned char sd);
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| addr | unsigned int | 任务函数地址 |
| id | unsigned char | 任务编号 |
| pr | unsigned char | 任务优先级 |
| sd | unsigned char | 任务堆栈层数 |

返回值：

- 1：创建成功
- -1：创建失败（参数错误、资源不足、重复注册）

功能说明：

创建并注册一个新任务。函数会：

1. 参数合法性检查（优先级、任务编号、地址）
2. 堆栈空间分配（使用位图管理）
3. 任务控制块初始化
4. 根据任务类型（标准任务/高速任务/中断处理）进行不同处理
5. 设置任务初始状态为就绪态

标准任务优先级范围：0-7
高速任务优先级：8
中断处理优先级：9-10

调用示例：

```c
// 创建标准任务
char result = os_task_create((unsigned int)task_func, 0, 3, 5);
if (result == 1) {
    // 任务创建成功
}
```

注意事项：

- 任务编号不能超过 OS_PROCESS_MAX+1
- 优先级不能超过 OS_SCHED_LEVEL_MAX
- 堆栈层数有限制（标准任务≤6，高速任务≤25）
- 不能重复注册同一任务编号
- 需要在系统初始化阶段调用

### os_interrupt_init

函数原型：

```c
void os_interrupt_init();
```

参数说明：

无

返回值：

无

功能说明：

MyOS 的初始化及中断支持。该函数负责：

1. 保存当前堆栈指针
2. 初始化系统堆栈
3. 保护现场寄存器
4. 跳转到中断服务程序

调用示例：

```c
os_interrupt_init();
```

注意事项：

- 仅在 MyOS 模式下使用
- 需要在系统启动时调用

### os_interrupt_exit

函数原型：

```c
void os_interrupt_exit();
```

参数说明：

无

返回值：

无

功能说明：

中断退出处理。根据当前中断状态选择正确的返回路径：

- 普通中断返回
- 中断嵌套返回
- MyOS 模式返回

调用示例：

```c
os_interrupt_exit();
```

注意事项：

- 必须在中断服务程序结束时调用
- 会根据 OS_ISR_FLAG 判断当前是否在中断中

### os_memset

函数原型：

```c
void os_memset(void *buffer, unsigned char c, unsigned int count);
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| buffer | void* | 目标缓冲区地址 |
| c | unsigned char | 填充值 |
| count | unsigned int | 填充长度 |

返回值：

无

功能说明：

将指定缓冲区填充为指定值。类似标准库的 memset 函数。

调用示例：

```c
u8 buffer[32];
os_memset(buffer, 0, 32);
```

注意事项：

- 需要确保缓冲区有足够空间
- count 参数以字节为单位

### os_data_to_xdata

函数原型：

```c
void os_data_to_xdata(u8 addr, u16 addr2);
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| addr | u8 | 源地址（DATA 区） |
| addr2 | u16 | 目标地址（XDATA 区） |

返回值：

无

功能说明：

将数据从 DATA 区拷贝到 XDATA 区。

调用示例：

```c
os_data_to_xdata(src_addr, dst_addr);
```

注意事项：

- 地址必须在合法范围内
- 用于 51 单片机特殊存储区操作

### os_xdata_to_data

函数原型：

```c
void os_xdata_to_data(u16 addr2, u8 addr);
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| addr2 | u16 | 源地址（XDATA 区） |
| addr | u8 | 目标地址（DATA 区） |

返回值：

无

功能说明：

将数据从 XDATA 区拷贝到 DATA 区。

调用示例：

```c
os_xdata_to_data(src_addr, dst_addr);
```

注意事项：

- 地址必须在合法范围内
- 用于 51 单片机特殊存储区操作

### os_scheduler_mode_switch

函数原型：

```c
char os_scheduler_mode_switch(unsigned char id);
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| id | unsigned char | 调度模式标识 |

返回值：

- 调度模式切换结果

功能说明：

切换调度器模式：

- id=1：实时模式（基于优先级）
- id=0：通用模式（基于时间片）

调用示例：

```c
os_scheduler_mode_switch(1);  // 切换到实时模式
```

注意事项：

- 模式切换会影响调度行为
- 建议在系统初始化时确定调度模式

### os_set_scheduler

函数原型：

```c
char os_set_scheduler(unsigned char id);
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| id | unsigned char | 调度器标识 |

返回值：

- 调度器设置结果

功能说明：

选择调度器类型：

- id=1：HRTOS 调度器
- id=0：MYOS 调度器

调用示例：

```c
os_set_scheduler(1);  // 使用 HRTOS 调度器
```

注意事项：

- 需要在系统启动时调用
- 一旦设置不建议频繁切换

### hrtos_main

函数原型：

```c
void hrtos_main();
```

参数说明：

无

返回值：

无

功能说明：

HRTOS 默认的用户主函数。系统启动后会跳转到该函数执行用户代码。

调用示例：

```c
// 用户实现该函数
void hrtos_main() {
    // 用户初始化代码
    // 创建任务
    // 启动调度
}
```

注意事项：

- 用户必须实现该函数
- 该函数是系统启动后的入口点

### os_slab_overflow

函数原型：

```c
void os_slab_overflow();
```

参数说明：

无

返回值：

无

功能说明：

系统发生堆栈溢出时调用的处理函数。默认实现可能导致系统复位。

调用示例：

```c
// 通常由系统自动调用
```

注意事项：

- 可以通过 os_slab_overflow_register 注册自定义处理函数
- 发生溢出表示严重错误

### os_slab_overflow_register

函数原型：

```c
void os_slab_overflow_register(void (*hook)(void));
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| hook | void (*)(void) | 溢出处理钩子函数指针 |

返回值：

无

功能说明：

注册堆栈溢出时的用户处理函数。当系统检测到堆栈溢出时，会调用注册的钩子函数。

调用示例：

```c
void my_overflow_handler() {
    // 用户自定义溢出处理
}

os_slab_overflow_register(my_overflow_handler);
```

注意事项：

- 钩子函数应尽可能简单
- 避免在钩子函数中调用复杂操作

### os_task_cleanup

函数原型：

```c
char os_task_cleanup(unsigned char id);
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| id | unsigned char | 任务编号 |

返回值：

- 清理结果

功能说明：

系统任务清理，释放任务占用的资源并清理相关数据结构。

调用示例：

```c
os_task_cleanup(0);
```

注意事项：

- 通常由系统内部调用
- 用户一般不需要直接调用

### os_idle_task

函数原型：

```c
void os_idle_task();
```

参数说明：

无

返回值：

无

功能说明：

系统 idle 任务，当没有其他就绪任务时运行。可以执行低优先级后台任务或进入低功耗模式。

调用示例：

```c
// 系统自动调用
```

注意事项：

- 可以通过 os_idle_hook_register 注册自定义 idle 钩子
- idle 任务优先级最低

### os_idle_hook_register

函数原型：

```c
void os_idle_hook_register(void (*hook)(void));
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| hook | void (*)(void) | idle 钩子函数指针 |

返回值：

无

功能说明：

注册 idle 任务钩子函数。当系统进入 idle 任务时，会调用注册的钩子函数。

调用示例：

```c
void my_idle_hook() {
    // 自定义 idle 处理
    // 可以进入低功耗模式
}

os_idle_hook_register(my_idle_hook);
```

注意事项：

- 钩子函数会被频繁调用
- 避免在钩子函数中执行耗时操作
- 钩子函数中不能调用阻塞 API
