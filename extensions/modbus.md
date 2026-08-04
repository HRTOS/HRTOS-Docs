# HRTOS Modbus RTU 模块使用说明

## 模块简介

### Modbus RTU 简介

Modbus RTU 是一种广泛应用于工业自动化领域的串行通信协议，采用主从模式通信，通过 CRC16 校验确保数据传输可靠性。该协议简单高效，适用于资源受限的嵌入式系统。

### HRTOS Modbus 模块作用

HRTOS Modbus RTU 模块为 HRTOS 实时操作系统提供完整的 Modbus 从站功能实现，基于任务、事件和中断机制设计，充分利用 HRTOS 的多任务调度能力，实现高效的串行通信处理。

### 支持的应用场景

- 工业传感器数据采集
- PLC 设备通信
- 智能仪表接口
- 嵌入式设备远程控制
- 51 MCU 应用开发

### 模块特点

- 基于 HRTOS 任务调度和事件同步机制
- 支持中断驱动接收，降低 CPU 占用
- 完整的帧超时检测机制
- 支持多种 Modbus 功能码
- 资源占用小，适合 51 MCU 等资源受限平台
- 清晰的分层架构，易于移植和扩展

---

## 模块架构

### 整体结构

```
应用层
 |
Modbus 任务 (modbus_task)
 |
Modbus 协议处理 (modbus_slave_process)
 |
帧管理 (modbus_frame)
 |
寄存器管理 (modbus_register)
 |
CRC 校验 (modbus_crc)
 |
UART 接口 (modbus_port)
 |
BSP 硬件抽象层 (bsp_uart)
```

### 文件职责说明

#### modbus.c
- Modbus 核心模块
- 提供 `modbus_init()` 初始化接口
- 实现 `modbus_process()` 周期处理函数
- 实现 `modbus_frame_parse()` 帧解析功能
- 负责地址、CRC、数据长度校验

#### modbus_slave.c
- Modbus 从站协议处理
- 实现功能码分发和响应生成
- 支持功能码：0x01（读线圈）、0x03（读保持寄存器）、0x05（写单个线圈）、0x06（写单个寄存器）、0x10（写多个寄存器）
- 处理设备地址匹配
- 自动添加 CRC 并发送响应

#### modbus_frame.c
- 帧接收和管理模块
- 实现 `modbus_frame_receive()` 逐字节接收
- 实现 `modbus_frame_timeout()` 帧超时检测（3.5 字符时间）
- 管理 `modbus_rx_buf` 接收缓冲区
- 通过事件机制通知处理任务

#### modbus_register.c
- 寄存器和线圈管理模块
- 实现 `modbus_reg_read()` / `modbus_reg_write()` 保持寄存器读写
- 实现 `modbus_coil_read()` / `modbus_coil_write()` 线圈读写
- 使用静态数组存储寄存器和线圈数据
- 提供地址范围保护

#### modbus_crc.c
- CRC16 校验模块
- 实现 `modbus_crc16()` 函数
- 采用 Modbus 标准 CRC16 算法（多项式 0xA001）
- 用于帧校验和响应生成

#### modbus_port.c
- 硬件端口抽象层
- 实现 `modbus_port_init()` 端口初始化
- 实现 `modbus_port_send()` 数据发送
- 调用 BSP 层 UART 接口

---

## HRTOS 集成方式

### 任务结构

#### modbus_task()
**作用**：Modbus 主处理任务

- 初始化 Modbus 模块
- 等待接收事件（`MODBUS_EVENT_RX`）
- 调用 `modbus_process()` 处理协议帧
- 删除事件标志，循环等待

#### modbus_timer_task()
**作用**：定时超时检测任务

- 周期执行（`os_delay_ms(1)`）
- 调用 `modbus_frame_timeout()` 检测帧接收超时
- 实现 3.5 字符时间超时判断

#### uart_isr()
**作用**：UART 中断服务程序

- 响应 UART 接收中断
- 读取接收到的字节
- 调用 `modbus_frame_receive()` 存入接收缓冲区
- 处理发送完成中断

### 完整数据流程

```
UART 接收中断
    ↓
uart_isr() 读取字节
    ↓
modbus_frame_receive(ch) 存入缓冲区
    ↓
modbus_timer_task() 周期检测超时
    ↓
超时后触发 MODBUS_EVENT_RX 事件
    ↓
modbus_task() 被唤醒
    ↓
modbus_process() 处理
    ↓
modbus_frame_parse() 解析帧（地址、CRC、长度）
    ↓
modbus_slave_process() 根据功能码处理
    ↓
modbus_reg_read/write() 或 modbus_coil_read/write()
    ↓
modbus_slave_send() 生成响应并添加 CRC
    ↓
modbus_port_send() 通过 UART 发送响应
```

---

## 快速开始

### 最小使用流程

根据 `main.c` 中的官方 Demo，完整的使用流程如下：

#### 步骤 1：UART 初始化

```c
bsp_uart_init();
```

#### 步骤 2：事件初始化

```c
os_event_init(MODBUS_EVENT_RX);
```

#### 步骤 3：创建 Modbus 任务

```c
// 创建定时器任务（用于超时检测）
os_task_create(
    modbus_timer_task,
    2,              // 任务 ID
    2,              // 优先级
    6               // 堆栈大小
);

// 创建 Modbus 主处理任务
os_task_create(
    modbus_task,
    1,              // 任务 ID
    3,              // 优先级
    6               // 堆栈大小
);

// 创建 UART 中断任务
os_task_create(
    uart_isr,
    0,              // 任务 ID
    9,              // 优先级（最高）
    0               // 堆栈大小（中断任务）
);
```

#### 步骤 4：启动系统

```c
void hrtos_main(void)
{
    os_slab_overflow_register(modbus_overflow);
    bsp_uart_init();
    os_event_init(MODBUS_EVENT_RX);
    
    os_task_create(modbus_timer_task, 2, 2, 6);
    os_task_create(modbus_task, 1, 3, 6);
    os_task_create(uart_isr, 0, 9, 0);
}
```

### 完整示例代码

以下代码来自当前工程 `Src/main.c`：

```c
void modbus_timer_task(void)
{
    while(1)
    {
        os_delay_ms(1);
        modbus_frame_timeout();
    }
}

void modbus_task(void)
{
    modbus_init();
    
    while(1)
    {
        os_event_wait(MODBUS_EVENT_RX, 0);
        modbus_process();
        os_event_delete(MODBUS_EVENT_RX);
    }
}

void uart_isr(void)
{
    unsigned char ch;
    char id;
    
    id = os_interrupt_id();
    if(id != 4)
    {
        os_interrupt_exit();
    }
    
    if(RI)
    {
        RI = 0;
        ch = SBUF;
        modbus_frame_receive(ch);
    }
    
    if(TI)
    {
        TI = 0;
    }
    
    os_interrupt_exit();
}

void hrtos_main(void)
{
    os_slab_overflow_register(modbus_overflow);
    bsp_uart_init();
    os_event_init(MODBUS_EVENT_RX);
    
    os_task_create(modbus_timer_task, 2, 2, 6);
    os_task_create(modbus_task, 1, 3, 6);
    os_task_create(uart_isr, 0, 9, 0);
}
```

---

## 配置说明

### 配置文件

所有配置项定义在 `Inc/modbus_config.h` 中。

### 配置项列表

| 配置宏 | 默认值 | 说明 |
|--------|--------|------|
| **MODBUS_SLAVE_ADDR** | 1 | Modbus 从机地址 |
| **MODBUS_RX_BUF_SIZE** | 68 | 接收缓冲区大小（字节） |
| **MODBUS_TX_BUF_SIZE** | 32 | 发送缓冲区大小（字节） |
| **MODBUS_REG_MAX** | 32 | 保持寄存器最大数量 |
| **MODBUS_FUNC_READ_HOLDING_REG** | 1 | 开启功能码 0x03（读保持寄存器） |
| **MODBUS_FUNC_WRITE_SINGLE_REG** | 1 | 开启功能码 0x06（写单个寄存器） |
| **MODBUS_FUNC_WRITE_MULTI_REG** | 1 | 开启功能码 0x10（写多个寄存器） |
| **MODBUS_COIL_MAX** | 8 | 线圈最大数量 |

### 配置说明

#### 从机地址配置
```c
#define MODBUS_SLAVE_ADDR 1
```
设置 Modbus 从站地址，支持广播地址 0。

#### 缓冲区配置
```c
#define MODBUS_RX_BUF_SIZE 68
#define MODBUS_TX_BUF_SIZE 32
```
根据实际通信数据量和系统内存调整。

#### 寄存器配置
```c
#define MODBUS_REG_MAX 32
```
定义保持寄存器数量，地址范围 0 ~ MODBUS_REG_MAX-1。

#### 功能码配置
```c
#define MODBUS_FUNC_READ_HOLDING_REG 1
#define MODBUS_FUNC_WRITE_SINGLE_REG 1
#define MODBUS_FUNC_WRITE_MULTI_REG 1
```
设置为 1 启用对应功能码，设置为 0 禁用。

#### 线圈配置
```c
#define MODBUS_COIL_MAX 8
```
定义线圈数量，1 bit 表示一个线圈。

---

## API 参考

### modbus.h

#### modbus_init()
**功能**：初始化 Modbus 协议模块

**函数原型**：
```c
void modbus_init(void);
```

**参数**：无

**返回值**：无

**使用示例**：
```c
void modbus_task(void)
{
    modbus_init();
    while(1)
    {
        os_event_wait(MODBUS_EVENT_RX, 0);
        modbus_process();
        os_event_delete(MODBUS_EVENT_RX);
    }
}
```

#### modbus_process()
**功能**：Modbus 周期处理，由任务周期调用

**函数原型**：
```c
void modbus_process(void);
```

**参数**：无

**返回值**：无

**使用示例**：
```c
os_event_wait(MODBUS_EVENT_RX, 0);
modbus_process();
```

#### modbus_coil_read()
**功能**：读取线圈状态

**函数原型**：
```c
unsigned char modbus_coil_read(unsigned int addr);
```

**参数**：
- `addr`：线圈地址

**返回值**：
- 0：线圈 OFF
- 1：线圈 ON

**使用示例**：
```c
unsigned char coil_status = modbus_coil_read(0);
```

#### modbus_coil_write()
**功能**：写入线圈状态

**函数原型**：
```c
void modbus_coil_write(unsigned int addr, unsigned char value);
```

**参数**：
- `addr`：线圈地址
- `value`：线圈值（0=OFF，1=ON）

**返回值**：无

**使用示例**：
```c
modbus_coil_write(0, 1);  // 设置线圈 0 为 ON
```

#### modbus_frame_timeout()
**功能**：帧超时检测，在定时任务中周期调用

**函数原型**：
```c
void modbus_frame_timeout(void);
```

**参数**：无

**返回值**：无

**使用示例**：
```c
void modbus_timer_task(void)
{
    while(1)
    {
        os_delay_ms(1);
        modbus_frame_timeout();
    }
}
```

---

### modbus_frame.h

#### modbus_frame_init()
**功能**：初始化帧处理模块

**函数原型**：
```c
void modbus_frame_init(void);
```

**参数**：无

**返回值**：无

**使用示例**：
```c
modbus_frame_init();
```

#### modbus_frame_receive()
**功能**：接收一个字节数据

**函数原型**：
```c
unsigned char modbus_frame_receive(unsigned char ch);
```

**参数**：
- `ch`：接收到的字节

**返回值**：
- 0：未完成
- 1：收到完整帧

**使用示例**：
```c
void uart_isr(void)
{
    if(RI)
    {
        RI = 0;
        ch = SBUF;
        modbus_frame_receive(ch);
    }
}
```

#### modbus_frame_parse()
**功能**：解析接收到的数据帧

**函数原型**：
```c
unsigned char modbus_frame_parse(void);
```

**参数**：无

**返回值**：
- 1：解析成功
- 0：解析失败（CRC 错误、长度错误等）

**使用示例**：
```c
if(modbus_frame_parse())
{
    modbus_slave_process(modbus_frame_get());
}
```

#### modbus_frame_get()
**功能**：获取当前数据帧

**函数原型**：
```c
modbus_frame_t *modbus_frame_get(void);
```

**参数**：无

**返回值**：指向当前帧结构的指针

**使用示例**：
```c
modbus_frame_t *frame = modbus_frame_get();
```

---

### modbus_port.h

#### modbus_port_init()
**功能**：Modbus 端口初始化

**函数原型**：
```c
void modbus_port_init(void);
```

**参数**：无

**返回值**：无

**使用示例**：
```c
modbus_port_init();
```

#### modbus_port_send()
**功能**：发送 Modbus 数据

**函数原型**：
```c
void modbus_port_send(unsigned char *buf, unsigned int len);
```

**参数**：
- `buf`：数据缓冲区指针
- `len`：数据长度

**返回值**：无

**使用示例**：
```c
unsigned char data[] = {0x01, 0x03, 0x00, 0x00};
modbus_port_send(data, 4);
```

#### modbus_port_recv()
**功能**：接收一个字节

**函数原型**：
```c
unsigned char modbus_port_recv(unsigned char *ch);
```

**参数**：
- `ch`：接收字节指针

**返回值**：接收状态

**使用示例**：
```c
unsigned char ch;
if(modbus_port_recv(&ch))
{
    // 处理接收到的字节
}
```

#### modbus_port_rx_byte()
**功能**：处理接收字节

**函数原型**：
```c
unsigned char modbus_port_rx_byte(unsigned char ch);
```

**参数**：
- `ch`：接收到的字节

**返回值**：处理状态

**使用示例**：
```c
modbus_port_rx_byte(ch);
```

---

### modbus_register.h

#### modbus_reg_read()
**功能**：读取保持寄存器

**函数原型**：
```c
unsigned int modbus_reg_read(unsigned char addr);
```

**参数**：
- `addr`：寄存器地址

**返回值**：寄存器值

**使用示例**：
```c
unsigned int value = modbus_reg_read(0);
```

#### modbus_reg_write()
**功能**：写入保持寄存器

**函数原型**：
```c
void modbus_reg_write(unsigned char addr, unsigned int value);
```

**参数**：
- `addr`：寄存器地址
- `value`：写入值

**返回值**：无

**使用示例**：
```c
modbus_reg_write(0, 0x1234);
```

---

### modbus_slave.h

#### modbus_slave_process()
**功能**：Modbus 从站请求处理

**函数原型**：
```c
void modbus_slave_process(modbus_frame_t *frame);
```

**参数**：
- `frame`：指向 Modbus 帧结构的指针

**返回值**：无

**使用示例**：
```c
if(modbus_frame_parse())
{
    modbus_slave_process(modbus_frame_get());
}
```

---

### modbus_crc.h

#### modbus_crc16()
**功能**：计算 Modbus CRC16 校验值

**函数原型**：
```c
unsigned int modbus_crc16(unsigned char *data1, unsigned int len);
```

**参数**：
- `data1`：数据缓冲区指针
- `len`：数据长度

**返回值**：CRC16 校验值

**使用示例**：
```c
unsigned char data[] = {0x01, 0x03, 0x00, 0x00};
unsigned int crc = modbus_crc16(data, 4);
```

---

## Modbus 寄存器管理

### 寄存器定义

Modbus 模块使用静态数组存储保持寄存器：

```c
static unsigned int holding_reg[MODBUS_REG_MAX];
```

寄存器地址范围：0 ~ MODBUS_REG_MAX-1（默认 0 ~ 31）

### 添加方式

寄存器在编译时通过配置宏确定数量，无需动态添加。用户可通过 `modbus_reg_write()` 预设寄存器初始值。

### 数据读取

使用 `modbus_reg_read()` 读取寄存器值：

```c
unsigned int value = modbus_reg_read(addr);
```

- 地址超出范围返回 0
- 返回值为 16 位无符号整数

### 数据写入

使用 `modbus_reg_write()` 写入寄存器值：

```c
modbus_reg_write(addr, value);
```

- 地址超出范围不执行写入
- 写入值为 16 位无符号整数

### 用户扩展方法

如需扩展寄存器功能（如关联实际硬件），可修改 `modbus_register.c` 中的读写函数：

```c
unsigned int modbus_reg_read(unsigned char addr)
{
    if(addr >= MODBUS_REG_MAX)
    {
        return 0;
    }
    
    // 用户自定义：读取实际硬件值
    // holding_reg[addr] = read_hardware(addr);
    
    return holding_reg[addr];
}

void modbus_reg_write(unsigned char addr, unsigned int value)
{
    if(addr < MODBUS_REG_MAX)
    {
        holding_reg[addr] = value;
        
        // 用户自定义：写入实际硬件
        // write_hardware(addr, value);
    }
}
```

---

## Modbus Slave 功能

### 从机初始化

从机初始化通过 `modbus_init()` 完成，包括：
- 端口初始化（`modbus_port_init()`）
- 帧处理初始化（`modbus_frame_init()`）
- 设置初始化标志

### 请求处理

从机请求处理流程：
1. UART 中断接收数据
2. 帧超时检测触发事件
3. `modbus_process()` 被调用
4. `modbus_frame_parse()` 解析帧（地址、CRC 校验）
5. `modbus_slave_process()` 根据功能码分发处理

### 数据响应流程

1. 根据功能码调用对应处理函数
2. 读取或写入寄存器/线圈
3. 生成响应帧（地址 + 功能码 + 数据）
4. 计算 CRC16 并附加到帧尾
5. 通过 `modbus_port_send()` 发送响应

### 支持功能码

| 功能码 | 名称 | 说明 | 支持状态 |
|--------|------|------|----------|
| 0x01 | Read Coils | 读取线圈 | 支持 |
| 0x03 | Read Holding Registers | 读取保持寄存器 | 支持 |
| 0x05 | Write Single Coil | 写单个线圈 | 支持 |
| 0x06 | Write Single Register | 写单个寄存器 | 支持 |
| 0x10 | Write Multiple Registers | 写多个寄存器 | 支持 |

---

## 移植说明

### 需要修改的文件

移植到不同 MCU 平台主要修改以下文件：

#### modbus_port.c
修改 UART 硬件接口调用：

```c
void modbus_port_init(void)
{
    // 替换为您的 UART 初始化代码
    // bsp_uart_init();
}

void modbus_port_send(unsigned char *buf, unsigned int len)
{
    unsigned int i;
    for (i = 0; i < len; i++)
    {
        // 替换为您的 UART 发送函数
        // bsp_uart_send_byte(buf[i]);
    }
}
```

#### BSP/bsp.c
实现 UART 硬件抽象层：

```c
void bsp_uart_init(void)
{
    // 配置 UART 参数（波特率、数据位、停止位等）
    // 使能 UART 接收中断
}

void bsp_uart_send_byte(unsigned char dat)
{
    // 发送单字节数据
    // 等待发送完成
}
```

### UART 发送

根据目标 MCU 的 UART 寄存器实现发送函数：

```c
void bsp_uart_send_byte(unsigned char dat)
{
    // 写入 UART 数据寄存器
    UART_DR = dat;
    
    // 等待发送完成
    while(!(UART_SR & UART_SR_TXE));
}
```

### UART 接收

在中断服务程序中读取接收到的字节：

```c
void uart_isr(void)
{
    unsigned char ch;
    
    // 检查接收中断标志
    if(UART_SR & UART_SR_RXNE)
    {
        ch = UART_DR;
        modbus_frame_receive(ch);
    }
    
    // 退出中断
    os_interrupt_exit();
}
```

### 中断处理

确保 UART 接收中断已使能，并正确配置中断优先级：

```c
void bsp_uart_init(void)
{
    // UART 配置
    // ...
    
    // 使能接收中断
    UART_CR |= UART_CR_RXNEIE;
    
    // 配置中断优先级
    NVIC_SetPriority(UART_IRQn, priority);
    
    // 使能中断
    NVIC_EnableIRQ(UART_IRQn);
}
```

### 定时检测

Modbus RTU 要求 3.5 字符时间的帧间隔检测。根据波特率计算定时周期：

```c
void modbus_timer_task(void)
{
    while(1)
    {
        // 根据波特率调整延时
        // 例如 9600bps，1 字符时间约 1ms
        // 3.5 字符时间约 3.5ms
        os_delay_ms(1);  // 调整为合适的值
        modbus_frame_timeout();
    }
}
```

---

## 完整 Demo 说明

### Demo 功能

官方 Demo 实现了完整的 Modbus RTU 从站功能，包括：
- UART 串口通信（9600bps）
- Modbus 帧接收和解析
- 支持功能码 0x01、0x03、0x05、0x06、0x10
- 寄存器和线圈读写
- CRC16 校验
- HRTOS 任务调度集成

### 硬件环境

- MCU：51 系列（如 STC89C52）
- UART：串口 1
- 波特率：9600bps（定时器 1 模式 2，重装值 0xFD）
- 中断：UART 接收中断

### 软件结构

```
hrtos_main()
    ├── bsp_uart_init()           // UART 初始化
    ├── os_event_init()           // 事件初始化
    ├── os_task_create()          // 创建定时器任务
    ├── os_task_create()          // 创建 Modbus 任务
    └── os_task_create()          // 创建 UART 中断任务
```

### 任务列表

| 任务名称 | 优先级 | 堆栈大小 | 功能 |
|----------|--------|----------|------|
| uart_isr | 9 | 0 | UART 中断服务 |
| modbus_timer_task | 2 | 6 | 帧超时检测 |
| modbus_task | 3 | 6 | Modbus 协议处理 |

### 运行流程

1. 系统启动，调用 `hrtos_main()`
2. 初始化 UART 和事件
3. 创建三个任务
4. UART 中断任务接收数据
5. 定时器任务检测超时
6. Modbus 任务处理协议帧
7. 响应数据通过 UART 发送

### 关键代码

#### UART 配置（BSP/bsp.c）
```c
void bsp_uart_init(void)
{
    SCON = 0x50;        // 8位数据,可变波特率
    TMOD &= 0x0F;
    TMOD |= 0x20;       // 定时器1模式2
    TL1 = 0xFD;         // 9600bps
    TH1 = 0xFD;
    ET1 = 0;            // 禁止定时器中断
    TR1 = 1;            // 定时器1开始计时
    EA = 1;
    ES = 1;             // 使能串口中断
}
```

#### 中断接收（Src/main.c）
```c
void uart_isr(void)
{
    unsigned char ch;
    char id;
    
    id = os_interrupt_id();
    if(id != 4)
    {
        os_interrupt_exit();
    }
    
    if(RI)
    {
        RI = 0;
        ch = SBUF;
        modbus_frame_receive(ch);
    }
    
    if(TI)
    {
        TI = 0;
    }
    
    os_interrupt_exit();
}
```

#### 任务创建（Src/main.c）
```c
void hrtos_main(void)
{
    os_slab_overflow_register(modbus_overflow);
    bsp_uart_init();
    os_event_init(MODBUS_EVENT_RX);
    
    os_task_create(modbus_timer_task, 2, 2, 6);
    os_task_create(modbus_task, 1, 3, 6);
    os_task_create(uart_isr, 0, 9, 0);
}
```

---

## 常见问题

### 无法通信

**可能原因**：
- 波特率不匹配
- 从机地址配置错误
- UART 硬件连接问题
- 中断未正确使能

**解决方法**：
- 检查主从机波特率是否一致
- 确认 `MODBUS_SLAVE_ADDR` 配置正确
- 检查 TX/RX 引脚连接
- 确认 UART 中断已使能（`ES=1`）

### CRC 错误

**可能原因**：
- 数据传输错误
- 噪声干扰
- 波特率偏差

**解决方法**：
- 检查通信线路质量
- 降低波特率
- 增加硬件滤波
- 检查 CRC 计算实现

### 地址错误

**可能原因**：
- 寄存器地址超出范围
- 线圈地址超出范围
- 从机地址不匹配

**解决方法**：
- 确认访问地址在 `MODBUS_REG_MAX` 范围内
- 确认线圈地址在 `MODBUS_COIL_MAX` 范围内
- 检查主站发送的从机地址

### UART 配置错误

**可能原因**：
- 定时器配置错误
- 波特率计算错误
- 中断优先级冲突

**解决方法**：
- 检查定时器模式和重装值
- 根据晶振频率计算正确的波特率
- 调整中断优先级

### 超时问题

**可能原因**：
- 定时任务周期不合适
- 帧超时阈值配置错误
- 任务调度延迟

**解决方法**：
- 调整 `modbus_timer_task()` 的延时周期
- 修改 `MODBUS_FRAME_TIMEOUT` 配置
- 检查 HRTOS 任务调度配置

### 数据格式错误

**可能原因**：
- 字节序错误
- 数据长度不匹配
- 功能码不支持

**解决方法**：
- Modbus RTU 使用大端序（高字节在前）
- 检查请求帧的数据长度字段
- 确认主站使用的功能码已启用

---

## 附录

### A. 数据帧结构

#### 请求帧格式
```
| 从机地址 | 功能码 | 数据 | CRC_L | CRC_H |
```

#### 响应帧格式
```
| 从机地址 | 功能码 | 数据 | CRC_L | CRC_H |
```

### B. 功能码示例

#### 0x03 读保持寄存器
**请求**：`01 03 00 00 00 02 CRC`
- 地址：01
- 功能码：03
- 起始地址：0000
- 寄存器数量：0002

**响应**：`01 03 04 12 34 56 78 CRC`
- 地址：01
- 功能码：03
- 字节数：04
- 数据：1234 5678

#### 0x06 写单个寄存器
**请求**：`01 06 00 10 00 64 CRC`
- 地址：01
- 功能码：06
- 寄存器地址：0010
- 写入值：0064

**响应**：`01 06 00 10 00 64 CRC`
- 原样返回请求内容

### C. 资源占用

| 资源类型 | 占用 |
|----------|------|
| RAM | 约 150 字节（缓冲区 + 寄存器） |
| ROM | 约 3 KB（代码） |
| 任务数量 | 3 个 |
| 事件数量 | 1 个 |

---
  
**更新日期**：2026-07-23  
**适用版本**：HRTOS Modbus RTU 模块
