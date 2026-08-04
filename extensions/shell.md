# HRTOS Shell 模块使用说明

## 模块介绍

HRTOS Shell 模块为嵌入式实时操作系统提供轻量级命令行交互接口，通过 UART 串口实现人机交互。

### 为什么 RTOS 需要 Shell

在嵌入式系统开发中，Shell 提供了便捷的系统调试和监控手段：

- **系统调试**：无需重新烧录程序即可查看系统状态
- **状态查看**：实时监控任务、信号量、消息队列等内核对象
- **参数控制**：动态调整系统参数和配置
- **功能测试**：快速验证系统功能模块

### HRTOS Shell 应用场景

Shell 模块特别适用于以下场景：

- 开发阶段的系统调试
- 现场故障诊断
- 系统参数动态配置
- 用户交互界面
- 自动化测试脚本执行

---

## Shell 模块架构

Shell 模块采用分层架构设计，各层职责清晰：

```
用户输入
    ↓
UART 接口
    ↓
Shell 输入处理
    ↓
命令解析
    ↓
命令匹配
    ↓
命令执行
```

### 文件职责说明

#### 核心文件

- **shell.c**：Shell 核心逻辑，包含初始化、输入处理、命令解析
- **shell_cmd.c**：命令表定义和命令执行调度
- **shell_port.c**：硬件移植接口，实现字符输入输出

#### Shell 子目录

- **init.c**：Shell 初始化
- **input.c**：UART 中断输入处理
- **main.c**：Shell 任务主循环
- **parse_line.c**：命令行解析，生成 argc/argv 参数
- **printf.c**：格式化输出函数
- **prompt.c**：命令提示符显示
- **shell.c**：Shell 周期处理
- **shell_port.c**：移植接口实现

#### Cmd 子目录

- **execute.c**：命令执行逻辑和字符串比较
- **help.c**：help 命令实现
- **task.c**：task 命令实现（任务管理）
- **msg.c**：msg 命令实现（消息队列管理）
- **wait.c**：wait 命令实现（等待状态查看）
- **version.c**：version 命令实现（版本信息）
- **shell_cmd.c**：命令表定义

---

## HRTOS Shell 运行流程

结合 main.c，Shell 的启动和运行流程如下：

### Shell 初始化

```c
void hrtos_main(void)
{
    shell_init();           // 初始化 Shell
    bsp_uart_init();        // 初始化 UART
    os_event_init(SHELL_EVENT_RX);  // 初始化 Shell 事件
    
    // 创建 Shell 任务
    os_task_create(shell_task, 1, 1, 6);
    
    // 创建 UART 中断任务
    os_task_create(uart_isr, 0, 9, 0);
}
```

### UART 初始化

UART 配置在 `bsp_uart_init()` 中完成：

```c
void bsp_uart_init(void)
{
    SCON = 0x50;        // 8位数据，可变波特率
    TMOD &= 0x0F;       // 设置定时器模式
    TMOD |= 0x20;       // 定时器1模式2
    TL1 = 0xFD;         // 设置定时初始值
    TH1 = 0xFD;         // 设置定时初始值
    ET1 = 0;            // 禁止定时器中断
    TR1 = 1;            // 定时器1开始计时
    EA = 1;             // 开总中断
    ES = 1;             // 开串口中断
}
```

### 输入接收

UART 中断接收字符：

```c
void uart_isr(void) // interrupt 4
{
    char ch;
    
    if (RI)
    {
        ch = SBUF;
        RI = 0;
        
        // 回车处理
        if ((ch == '\r') || (ch == '\n'))
        {
            shell_line[shell_pos] = '\0';
            if (shell_pos > 0)
            {
                os_event_set_from_isr(SHELL_EVENT_RX);
            }
        }
        // 退格处理
        else if ((ch == '\b') || (ch == 0x7F))
        {
            if (shell_pos > 0)
            {
                shell_pos--;
            }
        }
        // 普通字符
        else if (shell_pos < (SHELL_MAX_LINE_LEN - 1))
        {
            shell_line[shell_pos++] = ch;
        }
    }
}
```

### 命令解析

Shell 任务周期处理：

```c
static void shell_task(void)
{
    shell_init();
    
    while(1)
    {
        shell_process();
        os_delay(10);
    }
}
```

### 命令执行

```c
void shell_process(void)
{
    os_event_wait(SHELL_EVENT_RX, 0);
    
    shell_parse_line(shell_line);
    
    shell_pos = 0;
    shell_line[0] = '\0';
    
    os_event_delete(SHELL_EVENT_RX);
    
    shell_prompt();
}
```

---

## 快速开始

### 最小使用示例

#### 初始化 Shell

```c
#include "shell.h"

void hrtos_main(void)
{
    // 初始化 Shell
    shell_init();
    
    // 初始化 UART
    bsp_uart_init();
    
    // 初始化 Shell 事件
    os_event_init(SHELL_EVENT_RX);
}
```

#### 创建 Shell 任务

```c
static void shell_task(void)
{
    shell_init();
    
    while(1)
    {
        shell_process();
        os_delay(10);
    }
}

void hrtos_main(void)
{
    shell_init();
    bsp_uart_init();
    os_event_init(SHELL_EVENT_RX);
    
    // 创建 Shell 任务
    os_task_create(
        shell_task,
        1,          // 任务 ID
        1,          // 优先级
        6           // 堆栈大小
    );
}
```

#### UART 配置

UART 配置在 BSP 层完成，根据目标 MCU 修改 `bsp_uart_init()`：

```c
void bsp_uart_init(void)
{
    SCON = 0x50;        // 8位数据，可变波特率
    TMOD &= 0x0F;
    TMOD |= 0x20;       // 定时器1模式2
    TL1 = 0xFD;         // 波特率 9600
    TH1 = 0xFD;
    ET1 = 0;
    TR1 = 1;
    EA = 1;
    ES = 1;
}
```

#### 启动运行

编译烧录后，通过串口终端（如 SecureCRT、Putty）连接：

- 波特率：9600
- 数据位：8
- 停止位：1
- 校验位：无

连接成功后，将看到提示符：

```
HRTOS> 
```

---

## Shell 配置说明

### 宏配置

| 名称 | 类型 | 说明 |
|------|------|------|
| SHELL_MAX_LINE_LEN | 宏 | 命令行最大长度，默认 32 |
| SHELL_MAX_ARGC | 宏 | 最大命令参数个数，默认 8 |
| SHELL_EVENT_RX | 宏 | Shell 事件标志，默认 1 |
| SHELL_CMD_COUNT | 宏 | 命令表命令数量，默认 5 |

### 全局变量

| 名称 | 类型 | 说明 |
|------|------|------|
| shell_line | char[] | Shell 输入缓冲区 |
| shell_pos | unsigned char | 当前输入位置 |

### 配置建议

根据实际需求调整配置：

- 增加命令行长度：修改 `SHELL_MAX_LINE_LEN`
- 增加参数个数：修改 `SHELL_MAX_ARGC`
- 修改事件标志：修改 `SHELL_EVENT_RX`

---

## Shell API 参考

### shell_init

**功能说明**：初始化 Shell 模块

**函数原型**：
```c
void shell_init(void);
```

**参数说明**：无

**返回值**：无

**使用示例**：
```c
void hrtos_main(void)
{
    shell_init();
    bsp_uart_init();
    os_event_init(SHELL_EVENT_RX);
}
```

---

### shell_process

**功能说明**：Shell 周期处理函数，在 Shell 任务中循环调用

**函数原型**：
```c
void shell_process(void);
```

**参数说明**：无

**返回值**：无

**使用示例**：
```c
static void shell_task(void)
{
    shell_init();
    
    while(1)
    {
        shell_process();
        os_delay(10);
    }
}
```

---

### shell_putc

**功能说明**：输出一个字符到串口

**函数原型**：
```c
void shell_putc(char ch);
```

**参数说明**：
- `ch`：要输出的字符

**返回值**：无

**使用示例**：
```c
shell_putc('A');
```

---

### shell_puts

**功能说明**：输出字符串到串口

**函数原型**：
```c
void shell_puts(const char *str);
```

**参数说明**：
- `str`：要输出的字符串

**返回值**：无

**使用示例**：
```c
shell_puts("Hello HRTOS\r\n");
```

---

### shell_printf

**功能说明**：格式化输出，支持 %d、%u、%x、%s、%c 格式

**函数原型**：
```c
void shell_printf(const char *fmt, ...);
```

**参数说明**：
- `fmt`：格式化字符串
- `...`：可变参数

**返回值**：无

**使用示例**：
```c
shell_printf("Task ID: %d, Priority: %d\r\n", task_id, priority);
shell_printf("Value: 0x%x\r\n", value);
```

---

### shell_prompt

**功能说明**：显示 Shell 命令提示符

**函数原型**：
```c
void shell_prompt(void);
```

**参数说明**：无

**返回值**：无

**使用示例**：
```c
shell_prompt();  // 输出 "HRTOS> "
```

---

### shell_parse_line

**功能说明**：解析命令行，将输入字符串分割为 argc/argv 格式

**函数原型**：
```c
void shell_parse_line(char *line);
```

**参数说明**：
- `line`：命令行字符串

**返回值**：无

**使用示例**：
```c
char line[] = "task list";
shell_parse_line(line);
```

---

### shell_cmd_execute

**功能说明**：执行 Shell 命令

**函数原型**：
```c
void shell_cmd_execute(int argc, char *argv[]);
```

**参数说明**：
- `argc`：参数个数
- `argv`：参数列表

**返回值**：无

**使用示例**：
```c
char *argv[] = {"task", "list"};
shell_cmd_execute(2, argv);
```

---

## 内置命令说明

### help 命令

**功能**：显示所有可用命令列表

**使用方法**：
```
help
```

**参数**：无

**示例**：
```
HRTOS> help
Available commands:
  help
  version
  task
  wait
  msg
```

---

### version 命令

**功能**：显示系统版本信息

**使用方法**：
```
version
```

**参数**：无

**示例**：
```
HRTOS> version
HRTOS Version 1.0.0
```

---

### task 命令

**功能**：任务管理，包括查看任务列表、挂起/恢复任务、修改优先级

**使用方法**：
```
task list              查看任务状态
task suspend <id>      挂起任务
task resume <id>       恢复任务
task priority <id> <pr> 修改任务优先级
```

**参数**：
- `id`：任务 ID（0-15）
- `pr`：优先级（0-8）

**示例**：
```
HRTOS> task list
ID STATE PRIO
0  1     1
1  1     2
2  1     3

HRTOS> task suspend 1
suspend ok

HRTOS> task resume 1
resume ok

HRTOS> task priority 1 5
priority ok
```

---

### wait 命令

**功能**：查看阻塞任务列表

**使用方法**：
```
wait list
```

**参数**：无

**示例**：
```
HRTOS> wait list
ID  TYPE        OBJ  TICK  FLAG
1   SEM         0    100   0
2   MSG_RECV    1    50    1
```

**输出说明**：
- `ID`：任务 ID
- `TYPE`：等待类型（DELAY、SEM、MUTEX、MSG_SEND、MSG_RECV、EVENT、MAILBOX）
- `OBJ`：等待对象 ID
- `TICK`：等待超时时间
- `FLAG`：等待标志

---

### msg 命令

**功能**：查看消息队列信息

**使用方法**：
```
msg list
```

**参数**：无

**示例**：
```
HRTOS> msg list
ID  SIZE  COUNT  HEAD  TAIL
0   32    5      0     3
1   64    10     2     8
```

**输出说明**：
- `ID`：消息队列 ID
- `SIZE`：队列大小
- `COUNT`：当前消息数量
- `HEAD`：队列头位置
- `TAIL`：队列尾位置

---

## 自定义命令扩展

### 命令注册方式

在 `Src/Cmd/shell_cmd.c` 中的命令表添加新命令：

```c
const shell_cmd_t shell_cmd_table[] =
{
    { "help",    shell_cmd_help,    "show command list"      },
    { "version", shell_cmd_version, "show system version"    },
    { "task",    shell_cmd_task,    "task management"        },
    { "wait",    shell_cmd_wait,    "show waiting tasks"     },
    { "msg",     shell_cmd_msg,     "message management"     },
    { "mycmd",   shell_cmd_mycmd,   "my custom command"      }  // 新增命令
};
```

### 命令名称定义

命令名称需满足：
- 小写字母
- 不含特殊字符
- 简洁明了

### 参数处理方式

命令函数原型：

```c
void shell_cmd_mycmd(int argc, char *argv[]);
```

参数说明：
- `argc`：参数个数（包括命令本身）
- `argv`：参数数组，argv[0] 为命令名

### 执行函数编写

完整示例：

```c
void shell_cmd_mycmd(int argc, char *argv[])
{
    // 检查参数个数
    if (argc < 2)
    {
        shell_puts("Usage:\r\n");
        shell_puts("  mycmd <param>\r\n");
        return;
    }
    
    // 处理参数
    if (shell_strcmp(argv[1], "on") == 0)
    {
        shell_puts("Turn ON\r\n");
        // 执行开启操作
    }
    else if (shell_strcmp(argv[1], "off") == 0)
    {
        shell_puts("Turn OFF\r\n");
        // 执行关闭操作
    }
    else
    {
        shell_puts("Unknown parameter\r\n");
    }
}
```

### 完整示例

添加一个 LED 控制命令：

```c
// 在 shell_cmd.c 中添加命令表项
{ "led",     shell_cmd_led,     "LED control"     }

// 在单独文件中实现命令函数
void shell_cmd_led(int argc, char *argv[])
{
    u8 led_id;
    
    if (argc < 3)
    {
        shell_puts("Usage:\r\n");
        shell_puts("  led <id> <on|off>\r\n");
        return;
    }
    
    led_id = shell_atoi(argv[1]);
    
    if (shell_strcmp(argv[2], "on") == 0)
    {
        // 点亮 LED
        shell_printf("LED %d ON\r\n", led_id);
    }
    else if (shell_strcmp(argv[2], "off") == 0)
    {
        // 熄灭 LED
        shell_printf("LED %d OFF\r\n", led_id);
    }
    else
    {
        shell_puts("Invalid command\r\n");
    }
}
```

使用示例：
```
HRTOS> led 1 on
LED 1 ON

HRTOS> led 1 off
LED 1 OFF
```

---

## Shell 移植说明

### 移植文件

移植到不同 MCU 需要修改以下文件：

- **shell_port.c**：字符输入输出接口
- **bsp_uart_init()**：UART 初始化
- **uart_isr()**：UART 中断处理

### 字符输入接口

在 UART 中断中调用 Shell 输入处理：

```c
void uart_isr(void)
{
    char ch;
    
    if (RI)
    {
        ch = SBUF;
        RI = 0;
        
        // 回车处理
        if ((ch == '\r') || (ch == '\n'))
        {
            shell_line[shell_pos] = '\0';
            if (shell_pos > 0)
            {
                os_event_set_from_isr(SHELL_EVENT_RX);
            }
        }
        // 退格处理
        else if ((ch == '\b') || (ch == 0x7F))
        {
            if (shell_pos > 0)
            {
                shell_pos--;
            }
        }
        // 普通字符
        else if (shell_pos < (SHELL_MAX_LINE_LEN - 1))
        {
            shell_line[shell_pos++] = ch;
        }
    }
}
```

### 字符输出接口

实现 `shell_putc()` 和 `shell_puts()`：

```c
void shell_putc(char ch)
{
    bsp_uart_send_byte((unsigned char)ch);
}

void shell_puts(const char *str)
{
    bsp_uart_send_str_ram(str);
}
```

### UART 适配

根据目标 MCU 修改 `bsp_uart_init()`：

```c
void bsp_uart_init(void)
{
    // 配置 UART 参数
    // 设置波特率
    // 使能中断
}
```

### 移植步骤

1. 实现底层 UART 驱动
2. 实现 `bsp_uart_init()`
3. 实现 `bsp_uart_send_byte()`
4. 实现 `bsp_uart_send_str_ram()`
5. 修改 UART 中断处理函数
6. 调整波特率和中断优先级

---

## HRTOS 调试应用示例

### 查看任务状态

使用 `task list` 命令查看所有任务：

```
HRTOS> task list
ID STATE PRIO
0  1     1
1  1     2
2  1     3
```

**应用场景**：
- 确认任务是否正确创建
- 检查任务优先级配置
- 监控任务运行状态

### 查看消息队列

使用 `msg list` 命令查看消息队列状态：

```
HRTOS> msg list
ID  SIZE  COUNT  HEAD  TAIL
0   32    5      0     3
```

**应用场景**：
- 监控消息队列使用情况
- 检测消息积压
- 调试消息传递问题

### 查看等待状态

使用 `wait list` 命令查看阻塞任务：

```
HRTOS> wait list
ID  TYPE        OBJ  TICK  FLAG
1   SEM         0    100   0
2   MSG_RECV    1    50    1
```

**应用场景**：
- 诊断死锁问题
- 查看任务等待原因
- 监控系统资源竞争

### 任务控制

动态控制任务运行：

```
HRTOS> task suspend 2
suspend ok

HRTOS> task resume 2
resume ok

HRTOS> task priority 2 5
priority ok
```

**应用场景**：
- 临时挂起调试任务
- 动态调整任务优先级
- 测试任务调度策略

### 查看版本信息

```
HRTOS> version
HRTOS Version 1.0.0
```

**应用场景**：
- 确认固件版本
- 现场版本确认
- 问题追溯

---

## Demo 说明

### 硬件环境

Demo 基于 51 单片机：
- MCU：8051 系列
- 外设：UART 串口
- 波特率：9600

### 软件结构

Demo 包含以下任务：

1. **Shell 任务**：处理命令行输入
2. **UART 中断任务**：接收串口数据
3. **Demo 任务 1**：LED 闪烁测试
4. **Demo 任务 2**：LED 闪烁测试

### 任务结构

```c
void hrtos_main(void)
{
    shell_init();
    bsp_uart_init();
    os_event_init(SHELL_EVENT_RX);
    
    // Shell 任务
    os_task_create(shell_task, 1, 1, 6);
    
    // UART 中断任务
    os_task_create(uart_isr, 0, 9, 0);
    
    // Demo 任务
    os_task_create(demo_task1, 2, 1, 2);
    os_task_create(demo_task2, 3, 2, 2);
}
```

### 运行效果

系统启动后，通过串口终端连接，显示提示符：

```
HRTOS> 
```

可以输入命令进行交互：

```
HRTOS> help
Available commands:
  help
  version
  task
  wait
  msg

HRTOS> task list
ID STATE PRIO
0  1     1
1  1     2
2  1     1
3  1     2

HRTOS> version
HRTOS Version 1.0.0
```

### 操作示例

1. 连接串口终端（9600, 8N1）
2. 看到 `HRTOS>` 提示符
3. 输入 `help` 查看命令列表
4. 输入 `task list` 查看任务状态
5. 输入 `version` 查看版本信息

---

## 常见问题

### 无法输入字符

**可能原因**：
- UART 未正确初始化
- 波特率不匹配
- 中断未使能

**解决方法**：
- 检查 `bsp_uart_init()` 配置
- 确认终端波特率设置为 9600
- 检查 EA、ES 中断使能位

### 命令无响应

**可能原因**：
- Shell 任务未创建
- 事件未初始化
- 命令解析错误

**解决方法**：
- 确认 `os_task_create()` 创建了 Shell 任务
- 检查 `os_event_init(SHELL_EVENT_RX)` 调用
- 查看命令拼写是否正确

### UART 乱码

**可能原因**：
- 波特率不匹配
- 定时器配置错误
- 晶振频率不匹配

**解决方法**：
- 确认终端和 MCU 波特率一致
- 检查定时器重装值（TL1、TH1）
- 根据实际晶振频率调整定时器值

### 命令无法识别

**可能原因**：
- 命令拼写错误
- 命令未注册
- 命令表未更新

**解决方法**：
- 使用 `help` 查看可用命令
- 检查命令表是否包含该命令
- 确认命令名称拼写正确（小写）

### 参数错误

**可能原因**：
- 参数个数不足
- 参数类型错误
- 参数超出范围

**解决方法**：
- 使用命令的 help 功能查看用法
- 检查参数个数是否正确
- 确认参数值在有效范围内

### Shell 提示符不显示

**可能原因**：
- `shell_init()` 未调用
- `shell_prompt()` 未执行
- 输出函数未实现

**解决方法**：
- 确认在 `hrtos_main()` 中调用了 `shell_init()`
- 检查 `shell_prompt()` 函数实现
- 验证 `shell_puts()` 输出功能

---

## 附录

### A. 命令速查表

| 命令 | 功能 | 示例 |
|------|------|------|
| help | 显示命令列表 | `help` |
| version | 显示版本信息 | `version` |
| task list | 查看任务状态 | `task list` |
| task suspend | 挂起任务 | `task suspend 1` |
| task resume | 恢复任务 | `task resume 1` |
| task priority | 修改优先级 | `task priority 1 5` |
| wait list | 查看阻塞任务 | `wait list` |
| msg list | 查看消息队列 | `msg list` |

### B. 配置参数速查表

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| SHELL_MAX_LINE_LEN | 32 | 命令行最大长度 |
| SHELL_MAX_ARGC | 8 | 最大参数个数 |
| SHELL_EVENT_RX | 1 | Shell 事件标志 |
| SHELL_CMD_COUNT | 5 | 命令数量 |

### C. 错误码说明

Shell 模块使用字符串返回错误信息：

- `Unknown command`：命令不存在
- `Invalid task id`：任务 ID 无效
- `Invalid priority`：优先级无效
- `suspend failed`：挂起失败
- `resume failed`：恢复失败
- `priority failed`：优先级设置失败

---

**更新日期**：2026-07-23  
**适用版本**：HRTOS 3.32+
