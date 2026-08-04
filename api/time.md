# 时间管理模块

## 模块简介

时间管理模块提供系统时间管理功能，包括系统 tick 计数、任务延时、时间转换等。该模块基于 Timer0 实现，提供精确的时间管理能力。

### 设计目的

- 提供系统 tick 计数
- 实现任务延时功能
- 支持时间单位转换
- 提供系统运行时间查询
- 配置系统定时器参数

### 使用场景

- 任务延时
- 定时操作
- 时间测量
- 系统运行时间统计
- 硬件时序等待

## 数据类型

本模块无特殊数据类型定义。

## 宏定义

本模块使用 config.h 中定义的时间配置宏：

| 宏定义 | 默认值 | 功能说明 |
|---|---|---|
| OS_TIME_ONCE_DEFAULT | 5 | 时间片数量 |
| OS_TIME_T0 | 10000 | 单个时间片长度 |

## API 函数列表

| 函数名称 | 功能说明 |
|---|---|
| os_delay_ms | 忙等待延时（毫秒） |
| os_ms_to_tick | 毫秒转 tick |
| os_sec_to_tick | 秒转 tick |
| os_tick_config | 系统定时器配置 |
| os_tick_get | 获取系统 tick 计数 |
| os_uptime_ms | 获取系统运行时间（毫秒） |
| os_delay | 任务延时（tick） |

## API 详细说明

### os_delay_ms

函数原型：

```c
void os_delay_ms(unsigned char ms);
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| ms | unsigned char | 延时毫秒数 |

返回值：

无

功能说明：

忙等待延时函数，占用 CPU，不释放处理器。仅用于短时间硬件时序等待，不适合任务延时。

调用示例：

```c
os_delay_ms(10);  // 忙等待 10 毫秒
```

注意事项：

- 占用 CPU，不释放处理器
- 仅用于短时间硬件时序等待
- 不适合任务延时
- 任务延时应使用 os_delay()

### os_ms_to_tick

函数原型：

```c
u16 os_ms_to_tick(u16 ms);
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| ms | u16 | 毫秒数 |

返回值：

- 对应的 tick 数

功能说明：

将毫秒数转换为系统 tick 数。根据当前定时器配置进行换算。

调用示例：

```c
u16 ticks = os_ms_to_tick(100);  // 将 100 毫秒转换为 tick
```

注意事项：

- 转换结果依赖于当前定时器配置
- 用于时间参数换算

### os_sec_to_tick

函数原型：

```c
u16 os_sec_to_tick(u8 sec);
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| sec | u8 | 秒数 |

返回值：

- 对应的 tick 数

功能说明：

将秒数转换为系统 tick 数。根据当前定时器配置进行换算。

调用示例：

```c
u16 ticks = os_sec_to_tick(5);  // 将 5 秒转换为 tick
```

注意事项：

- 转换结果依赖于当前定时器配置
- 用于时间参数换算

### os_tick_config

函数原型：

```c
char os_tick_config(u8 slice_count, u16 tick_value);
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| slice_count | u8 | 时间片数量 |
| tick_value | u16 | 单个时间片的大小 |

返回值：

- 0：配置成功
- -1：参数错误

功能说明：

系统定时器时间设置。配置 Timer0 的重装值和时间片数量：

1. 清零系统时钟计数（高16位和低16位）
2. 设置时间片数量（OS_TIME_ONCE = slice_count）
3. 计算 Timer0 重装值（65535 - tick_value + 1）
4. 检查重装值是否合法（必须 > 1000）
5. 设置 Timer0 高字节和低字节（OS_T0_TH0、OS_T0_TL0）

调用示例：

```c
char result = os_tick_config(5, 10000);
if (result == 0) {
    // 配置成功
}
```

注意事项：

- tick_value 不能为 0
- 计算后的重装值必须 > 1000
- 应在系统启动时调用
- 影响所有时间相关函数

### os_tick_get

函数原型：

```c
void os_tick_get(u16 *high, u16 *low);
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| high | u16* | 返回 tick 高16位 |
| low | u16* | 返回 tick 低16位 |

返回值：

无

功能说明：

获取系统 tick 计数。系统 tick 由高16位和低16位组成32位计数器。

调用示例：

```c
u16 high, low;
os_tick_get(&high, &low);
u32 total_ticks = ((u32)high << 16) | low;
```

注意事项：

- 32位计数器，低16位溢出时高16位加1
- 用于精确时间测量
- 读取时建议关中断保证原子性

### os_uptime_ms

函数原型：

```c
u32 os_uptime_ms();
```

参数说明：

无

返回值：

- 系统累计运行时间（毫秒）

功能说明：

获取系统累计运行时间（毫秒）。函数会：

1. 读取32位tick计数（高16位 + 低16位）
2. 获取单tick对应的时间（微秒）
3. 转换为毫秒
4. 返回总运行时间

计算方法：
- tick_us = 65536 - tick_value（Timer0 重装值）
- tick_ms = tick_us / 1000
- total_ms = total_tick * tick_ms

调用示例：

```c
u32 uptime = os_uptime_ms();
```

注意事项：

- 基于 Timer0 配置计算
- 使用浮点运算减少误差
- 返回值为32位毫秒数
- 读取时关中断保证原子性

### os_delay

函数原型：

```c
void os_delay(u16 tick);
```

参数说明：

| 参数 | 类型 | 说明 |
|---|---|---|
| tick | u16 | 延时的 tick 数 |

返回值：

无

功能说明：

任务延时函数，基于系统 tick 计时。当前任务进入 WAIT_DELAY 状态，到期后自动唤醒。

1. tick = 0 时直接返回
2. 检查当前任务合法性
3. 调用 os_wait(WAIT_DELAY, OS_INVALID_ID, tick) 进入等待
4. 到期后被内核唤醒

调用示例：

```c
os_delay(100);  // 延时 100 个 tick
```

注意事项：

- 会阻塞当前任务
- 释放 CPU，其他任务可运行
- 基于系统 tick 计时
- tick = 0 立即返回
- 高速任务和标准任务处理方式不同
