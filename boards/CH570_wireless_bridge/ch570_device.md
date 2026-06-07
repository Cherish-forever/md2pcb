# CH570 无线串口桥 — 从机端（目标端）设计文档

> 版本：v0.2（增加远程 ISP 串口烧录章节，含电路/时序/AT指令/状态机）
> 日期：2026-06-08
> 状态：草稿，待审核完善
> 配套文档：`ch570_host.md`（主机端 / PC 端）

## 1. 角色定位

**从机端**：焊接在目标板上（或做成一个小模块插在目标板上），通过 UART 连接目标 MCU。从机端通过 2.4G 无线与主机端通信，实现两件事：

1. **无线串口透传** — PC ↔ 目标 MCU 的 printf 日志 + 命令行交互
2. **无线 ISP 烧录** — PC 端一键远程烧录目标 MCU 固件，无需碰板子

```
PC 串口终端 ──USB── CH570 主机 ──2.4G──► CH570 从机 ──UART── 目标 MCU
                                                              │
                                          ┌───────────────────┤
                                          │   BOOT0 控制      │
                                          │   RESET 控制      │
                                          └───────────────────┘
```

- **下行（PC→目标）**：2.4G 接收 → CH570 缓冲 → UART TX → 目标 MCU RX
- **上行（目标→PC）**：目标 MCU TX → UART RX → CH570 缓冲 → 2.4G 发送
- **控制通道**：主机发 ISP 指令 → 从机 GPIO 操作 BOOT0/RESET → 控制目标 MCU 启动模式

## 2. 芯片选型

| 特性 | CH570（从机端要求） |
|------|-------------------|
| UART | ✅ 必须（连接目标 MCU） |
| 2.4G 无线 | ✅ 必须 |
| USB | ❌ 不需要（从机不接 PC） |
| GPIO | 少量（状态 LED + 可选 BOOT/EN 控制） |
| 功耗 | **关键指标**——从机端由目标板供电 |
| 推荐封装 | **DFN10**（最小）或 **TSSOP16**（好焊） |

> 从机端不需要 USB，解放了引脚。DFN10 封装 3×3mm，几乎不占目标板面积。

## 3. 硬件设计

### 3.1 最小系统（DFN10 / TSSOP16）

```
从机端 CH570
├── VCC    ← 目标板 3.3V 或 5V
├── GND    ← 目标板 GND
├── UART TX → 目标 MCU RX         （ISP 串口）
├── UART RX ← 目标 MCU TX         （ISP 串口）
├── GPIO0  → 目标 MCU BOOT0       （远程 ISP 控制，高阻+外部下拉）
├── GPIO1  → 目标 MCU RESET       （远程复位/ISP，开漏输出）
├── GPIO2  → LED 状态指示          （可选）
├── RF     → 2.4G PCB 天线 / 陶瓷天线
```

**对外接口**（6 针 2.54mm 排针）：

| 针号 | 信号 | 方向 | 说明 |
|------|------|------|------|
| 1 | VCC | ← 目标板 | 3.3V 或 5V 供电 |
| 2 | GND | ↔ | 公共地 |
| 3 | TX | → 目标 MCU RX | 串口发送 |
| 4 | RX | ← 目标 MCU TX | 串口接收 |
| 5 | BOOT | → 目标 MCU BOOT0 | ISP 模式控制 |
| 6 | RST | → 目标 MCU RESET | 复位控制 |

> 最少只需要接 4 根（VCC/GND/TX/RX）就能用无线串口。BOOT 和 RST 是可选的 ISP 功能引脚。

### 3.2 无线串口（核心功能）

CH570 的 UART 直接连目标 MCU 的 UART：

```
CH570 UART_TX ──────► 目标 MCU UART_RX
CH570 UART_RX ◄────── 目标 MCU UART_TX
CH570 GND    ──────── 目标 MCU GND
```

- 电平匹配：同电压域（都 3.3V），不需电平转换
- 波特率：由 CH570 固件配置，需匹配目标 MCU 的 UART 波特率
- 如目标 MCU 是 5V，CH570 有一个 5V tolerant GPIO，可以接 RX；TX 需要分压或二极管降压

### 3.3 远程 ISP 串口烧录（核心卖点）

这是从机端相比普通无线串口模块**最大的差异优势**。通过 GPIO 控制目标 MCU 的 BOOT0 和 RESET 引脚，从机端可以远程操纵目标 MCU 进入 Bootloader，然后通过 UART 透传 ISP 协议完成固件烧录。

#### 3.3.1 电路连接

```
从机端 CH570                       目标 MCU (以 STM32F103 为例)
┌──────────────┐                ┌─────────────────────────┐
│              │                │                         │
│ UART_TX ─────┼────────────────┼──► USART1_RX (PA10)    │
│ UART_RX ─────┼────────────────┼──◄ USART1_TX (PA9)     │
│              │                │                         │
│ GPIO_BOOT0 ──┼──┬─────────────┼──► BOOT0                │
│              │  │             │      ┆                  │
│              │  └── 10kΩ GND  │   内部上拉 ~40kΩ        │
│              │     (默认下拉) │      ┆                  │
│              │                │      └── VCC (3.3V)    │
│              │                │                         │
│ GPIO_RST ────┼────────────────┼──► NRST                 │
│              │                │      ┆                  │
│              │                │   10kΩ VCC (外部上拉)    │
│              │                │      ┆                  │
│              │                │   100nF GND (去抖)      │
│              │                │                         │
│ GND ─────────┼────────────────┼──► GND                  │
└──────────────┘                └─────────────────────────┘
```

**BOOT0 电路说明**：

- CH570 GPIO 输出推挽模式
- GPIO 拉高 → BOOT0=1 → 复位后进入 Bootloader
- GPIO 拉低（或高阻 + 外部下拉）→ BOOT0=0 → 正常启动
- 默认状态：**GPIO 高阻**，靠外部 10kΩ 下拉确保 BOOT0 为低，不影响目标 MCU 正常启动
- ISP 时：GPIO 切换为推挽输出，拉高 BOOT0

> **关键安全设计**：GPIO 默认高阻态而非输出低。这样即使从机端 CH570 未上电或处于复位状态，BOOT0 也不会被意外拉低（某些 MCU 的 BOOT0 内部上拉很弱，外部强下拉可能导致无法进入 Bootloader，反过来外部强上拉又会使芯片永远进 Bootloader）。默认高阻 + 弱下拉是最安全的——"不干预"状态。

**RESET 电路说明**：

- CH570 GPIO 输出开漏模式
- GPIO 拉低 → NRST 拉低 → MCU 复位
- GPIO 释放（高阻）→ NRST 被外部 10kΩ 上拉回到高电平 → MCU 运行
- 100nF 对地电容：硬件去抖，防止干扰脉冲误复位

**注意事项**：

- BOOT0 和 RESET 的 GPIO 在 CH570 上电初始化阶段必须保持高阻态，避免目标 MCU 上电时序被干扰
- GPIO 默认状态要跟目标 MCU 的默认启动模式兼容
- 不同目标 MCU 的 BOOT 引脚命名不同，见下表

#### 3.3.2 主流 MCU 的 BOOT 引脚对照

| MCU 系列 | BOOT 引脚 | 进入 Bootloader 条件 | ISP 接口 |
|----------|-----------|---------------------|----------|
| STM32F1/F4 | BOOT0 | BOOT0=1, 复位 | USART1 (PA9/PA10) |
| STM32G0/G4 | BOOT0 + BOOT1 | 取决于 nBOOT_SEL option byte | USART1/USART2 |
| GD32 | BOOT0 | BOOT0=1, 复位 | USART0 (PA9/PA10) |
| CH32V003 | PD1 (IO_NRST 复用) | 上电时 PD1=低 | USART1 (PD5/PD6) |
| CH32V103 | BOOT0 | BOOT0=1, 复位 | USART1 |
| CH32V307 | BOOT0 | BOOT0=1, 复位 | USART1 |
| ESP32 | GPIO0 | GPIO0=0, 复位 | UART0 (TX/RX) |
| ESP32-C3 | GPIO9 | GPIO9=0, 复位 | UART0 |
| ATmega328P | RESET | 复位后 Bootloader 等待 1 秒 | UART0 |

> CH570 的两个 GPIO 可以覆盖上表中绝大部分 MCU。注意 CH32V003 比较特殊——它用 PD1 脚（也是 NRST）做 BOOT，需要特殊适配。

#### 3.3.3 ISP 烧录完整时序

```
                   ISP 烧录流程时序图

主机端 (PC)               从机端 (CH570)              目标 MCU
    │                          │                         │
    │  [CMD_ISP_START]         │                         │
    │──── 2.4G ──────────────►│                         │
    │                          │ GPIO_BOOT0 = HIGH       │
    │                          │ GPIO_RST = LOW          │
    │                          │ delay(10ms)             │
    │                          │ GPIO_RST = HIGH-Z       │── BOOT0=1
    │                          │ (释放复位)              │   NRST 上升沿
    │                          │ delay(50ms)             │── MCU 启动
    │                          │                         │   进入 Bootloader
    │                          │                         │
    │  [CMD_ISP_ACK]           │                         │
    │◄─── 2.4G ───────────────│                         │
    │                          │                         │
    │  ISP 握手 (0x7F)         │                         │
    │──── 2.4G ──────────────►│──── UART ──────────────►│── Bootloader
    │                          │                         │   回复 ACK(0x79)
    │                          │◄─── UART ───────────────│
    │◄─── 2.4G ───────────────│                         │
    │                          │                         │
    │  ISP 命令序列             │                         │
    │  (Get/Erase/Write/...)   │                         │
    │════ 2.4G 透传 ══════════│════ UART 透传 ══════════│
    │                          │                         │
    │  ISP 完成                 │                         │
    │                          │ GPIO_BOOT0 = HIGH-Z      │
    │                          │ (释放 BOOT0)            │
    │                          │ GPIO_RST = LOW          │
    │                          │ delay(5ms)              │
    │                          │ GPIO_RST = HIGH-Z       │── BOOT0=0
    │                          │ (释放复位)              │   NRST 上升沿
    │                          │                         │── 正常启动
    │                          │                         │   运行新固件
```

> **耗时估算（115200 波特率，Flash 64KB）**：
> - 进入 Bootloader：~60ms
> - 擦除全片：~3 秒
> - 写入 64KB：~15 秒
> - 校验：~5 秒
> - 复位到正常运行：~10ms
> - **总计：约 25 秒**（有线 ISP 约 20 秒，无线增加约 5 秒延迟开销）

#### 3.3.4 主机端 PC 工具适配

从 PC 端看，ISP 烧录跟有线没有本质区别——主机端 CH570 的 USB CDC 就是一个串口。现有烧录工具直接可用：

| 工具 | 目标 MCU | 命令行示例 |
|------|---------|-----------|
| STM32CubeProgrammer | STM32 | `STM32_Programmer_CLI -c port=COM3 -w firmware.hex` |
| stm32flash | STM32 | `stm32flash -w firmware.bin /dev/ttyUSB0` |
| WCHISPTool | CH32 | GUI 选择 COM 口，一键下载 |
| esptool | ESP32 | `esptool.py --port COM3 write_flash 0x0 firmware.bin` |
| avrdude | ATmega | `avrdude -c arduino -P COM3 -p m328p -U flash:w:firmware.hex` |

> PC 烧录工具完全无感知无线桥的存在——它以为在跟一个有线串口通信。

#### 3.3.5 ISP 协议帧格式

从机端固件定义了专用的 ISP 控制帧，与透传数据分开处理。这些帧通过 2.4G 在主机和从机之间交换，**不出现在目标 MCU 的 UART 上**：

| 命令帧 | 方向 | 负载 | 说明 |
|--------|------|------|------|
| `CMD_ISP_START` | 主机→从机 | 无 | 进入 ISP 模式：拉 BOOT0 + 复位目标 |
| `CMD_ISP_ACK` | 从机→主机 | 状态码 | 确认目标已进入 Bootloader（或失败） |
| `CMD_ISP_EXIT` | 主机→从机 | 无 | 退出 ISP：释放 BOOT0 + 复位目标 |
| `CMD_ISP_EXIT_ACK` | 从机→主机 | 状态码 | 确认目标已回到正常模式 |
| `CMD_ISP_ABORT` | 主机→从机 | 无 | 异常终止（如用户取消烧录） |

> 所有 ISP 控制帧使用 2.4G 协议的控制通道（或带内特殊标记），不与 UART 透传数据混合。进入 ISP 模式后，从机切换到"ISP 透传模式"——所有 2.4G 数据直接转 UART，不做 +++ 转义检测。

#### 3.3.6 错误处理

| 异常场景 | 从机端处理 |
|----------|-----------|
| Bootloader 握手超时（2 秒无 ACK） | 重试 1 次进入 ISP 流程；仍失败则回复 `CMD_ISP_ACK: ERROR_NO_BOOTLOADER` |
| ISP 过程中 2.4G 断连超时（10 秒无心跳） | 自动执行 `ISP_EXIT`，释放 BOOT0 + 复位目标，让目标回到正常运行 |
| ISP 过程中目标 MCU 意外断开 | UART RX 超时 → 通知主机端暂停；超时超过 5 秒 → 自动 ABORT |
| 烧录数据校验失败 | 主机端检测到（校验在 PC 端做），发 `CMD_ISP_ABORT`，从机端退出并复位目标 |

> **掉线保护是 ISP 无线桥的生命线**。一旦主机端断开，从机端不能把目标 MCU 永远锁在 Bootloader 里。10 秒超时自动复位是硬性要求。

### 3.4 供电

```
目标板 3.3V ──→ CH570 VCC
                   │
              内置 LDO → 内核 + 射频
```

或

```
目标板 5V ──→ CH570 VCC（芯片内置 5V→3.3V LDO）
```

> CH570 功耗低（1.2mA/MHz + 射频发射峰值约 20mA），从目标板取几十 mA 完全没压力。不会拖垮目标板的 3.3V LDO。

### 3.5 天线

从机端如果做成独立小板：

- **PCB 蛇形天线**：免费，占 PCB 面积约 15×8mm，2.4G 常用方案
- **陶瓷贴片天线**（如 AN2051）：¥0.3-0.5，体积 3.2×1.6mm，省面积

如果从机端直接焊接在目标板预留焊盘上，天线需要伸出目标板边缘，避免被大面积铜皮遮挡。

## 4. 固件架构

### 4.1 数据流

```
          ┌──────────────────────────────────┐
          │         CH570 从机固件             │
          │                                  │
2.4G RX ──►│  RX 环形缓冲 (1KB)              │
          │        ↓                        │
          │  UART TX FIFO                    │──► UART TX → 目标
          │                                  │
UART RX ──►│  RX 环形缓冲 (1KB)              │
          │        ↓                        │
          │  2.4G TX FIFO                    │──► 2.4G TX
          │                                  │
          └──────────────────────────────────┘
```

### 4.2 固件模块

| 模块 | 功能 |
|------|------|
| 2.4G 协议栈 | 无线收发、自动重传、ACK |
| UART 驱动 | 与目标 MCU 通信，波特率可配置 |
| 环形缓冲 | 双向各 1KB |
| 连接管理 | 配对、心跳、超时 |
| ISP 控制 | BOOT0/RESET GPIO 控制 |
| AT 指令 | 串口透传模式下，特定转义序列用于从机配置 |

### 4.3 波特率处理

这是从机端的关键设计决策：

```
主机端 (USB CDC) → 2.4G @ 2Mbps → 从机端 → UART @ 目标波特率
        ↑                               ↑
    "无限速"                       可能成为瓶颈
```

**情况一：目标波特率 ≤ 921600**
- 2.4G @ 2Mbps 完全够用，无线不是瓶颈
- 直接透传，无需流控

**情况二：目标波特率 > 921600（如 2Mbaud）**
- 2.4G @ 2Mbps 接近上限，需要流控
- 主机端暂停发送，等待从机端 UART TX 缓冲清空
- 通过 2.4G ACK 机制自然实现反压

> CH347 那种 9Mbaud 波特率在无线桥下不现实——无线带宽不够。

### 4.4 AT 指令设计

从机端正常上电后进入"透传模式"，所有 UART 数据直接转发。AT 指令分两类：**目标端 AT**（发给目标 MCU，经 CH570 透传）和**从机本地 AT**（发给 CH570 本身，用于配置从机参数）。

#### 4.4.1 从机本地 AT 指令（发给 CH570 本身）

从 PC 端通过无线发送，用于配置从机参数：

```
主机发 (2.4G):  +++ (停顿 1 秒不发送任何数据)
从机回复:       OK\r\n（进入 AT 模式）

主机发:         AT+BAUD=115200\r\n
从机回复:       OK\r\n（UART 波特率已改为 115200）

主机发:         AT+BOOT=1\r\n
从机回复:       OK\r\n（GPIO_BOOT0 已拉高）

主机发:         AT+RST=0\r\n
从机回复:       OK\r\n（GPIO_RST 已拉低，目标复位中）

主机发:         AT+RST=1\r\n
从机回复:       OK\r\n（GPIO_RST 已释放）

主机发:         AT+EXIT\r\n
从机回复:       OK\r\n（回到透传模式）
```

#### 4.4.2 ISP 快捷指令

为方便 ISP 流程，提供组合指令，一键操作：

| 指令 | 等价操作 | 说明 |
|------|---------|------|
| `AT+ISP_ENTER` | BOOT=1 → RST=0 → delay(10ms) → RST=1 → delay(50ms) | 一键让目标进入 Bootloader |
| `AT+ISP_EXIT` | BOOT=0 → RST=0 → delay(5ms) → RST=1 | 一键让目标回到正常模式 |

> **设计原则**：AT 指令通过 2.4G 控制通道传输，不与 UART 透传数据混合。主机端固件负责将 PC 端发来的 AT 指令识别并转发到专用控制通道。

#### 4.4.3 完整 AT 指令表

| 指令 | 参数 | 说明 |
|------|------|------|
| `AT` | 无 | 测试连接，回复 OK |
| `AT+BAUD=<rate>` | 1200~921600 | 设置 UART 波特率 |
| `AT+BOOT=<0\|1>` | 0 或 1 | 控制 BOOT0 GPIO |
| `AT+RST=<0\|1>` | 0 或 1 | 控制 RESET GPIO |
| `AT+ISP_ENTER` | 无 | 一键进入 ISP 模式 |
| `AT+ISP_EXIT` | 无 | 一键退出 ISP 模式 |
| `AT+CHANNEL=<n>` | 0~125 | 设置 2.4G 频道 |
| `AT+POWER=<dBm>` | -20~+7 | 设置发射功率 |
| `AT+INFO` | 无 | 查询固件版本/信号强度 |
| `AT+EXIT` | 无 | 退出 AT 模式，回到透传 |

### 4.6 关键固件逻辑（伪代码）

```
// ===== 全局状态 =====
typedef enum {
    STATE_TRANSPARENT,    // 透传模式（默认）
    STATE_AT_MODE,        // AT 指令模式
    STATE_ISP_MODE,       // ISP 透传模式（进入 Bootloader 后）
    STATE_ISP_ENTERING,   // 正在进入 ISP（BOOT0/RST 操作中）
} device_state_t;

device_state_t state = STATE_TRANSPARENT;

// ===== 初始化 =====
void init():
    RF_Init(channel, tx_power)
    RF_SetAddress(device_addr, host_addr)
    UART_Init(DEFAULT_BAUD)            // 默认 115200
    GPIO_Init(BOOT0_PIN, HIGH_Z)       // 默认高阻，不干预目标
    GPIO_Init(RST_PIN, OPEN_DRAIN)     // 开漏模式
    GPIO_Release(RST_PIN)              // 释放 RESET
    LED_Init()
    watchdog_enable(5s)               // 看门狗防止固件卡死

// ===== ISP 时序函数 =====
void isp_enter():
    state = STATE_ISP_ENTERING
    GPIO_BOOT0_HIGH()                  // 拉高 BOOT0
    GPIO_RST_LOW()                     // 拉低 RESET
    delay_ms(10)
    GPIO_RST_RELEASE()                 // 释放 RESET
    delay_ms(50)                       // 等待 MCU 启动 Bootloader
    state = STATE_ISP_MODE

void isp_exit():
    GPIO_BOOT0_HIGH_Z()                // 释放 BOOT0（回到默认下拉）
    GPIO_RST_LOW()
    delay_ms(5)
    GPIO_RST_RELEASE()                 // 释放 RESET → MCU 正常启动
    delay_ms(10)
    state = STATE_TRANSPARENT

// ===== 主循环 =====
void main_loop():
    watchdog_feed()

    switch state:

    case STATE_TRANSPARENT:
        // 下行：无线 → 目标 UART
        if RF_Data_Available():
            packet = RF_Read()
            if is_isp_control_frame(packet):
                handle_isp_command(packet)     // ISP 控制帧
            else:
                UART_Write(packet.data)        // 正常透传

        // 上行：目标 UART → 无线
        if UART_RX_Available():
            data = UART_Read(up_to_32_bytes)
            RF_Send(data)

        // 心跳维持连接
        if heartbeat_timeout:
            RF_Send_Beacon()

        break

    case STATE_ISP_MODE:
        // ISP 透传：不检测 +++ 转义，全速转发
        if RF_Data_Available():
            packet = RF_Read()
            if is_isp_control_frame(packet):
                handle_isp_command(packet)
            else:
                UART_Write(packet.data)

        if UART_RX_Available():
            data = UART_Read(up_to_32_bytes)
            RF_Send(data)

        // ISP 超时保护：10 秒无数据自动退出
        if isp_idle_timeout(10s):
            isp_exit()                     // 不能把 MCU 锁在 Bootloader
        break

    case STATE_AT_MODE:
        // 处理 AT 指令
        if RF_Data_Available():
            cmd = RF_Read()
            if cmd == "AT+ISP_ENTER":
                isp_enter()
                RF_Send("OK\r\n")
            elif cmd == "AT+ISP_EXIT":
                isp_exit()
                RF_Send("OK\r\n")
            elif cmd == "AT+EXIT":
                state = STATE_TRANSPARENT
                RF_Send("OK\r\n")
            elif cmd.startswith("AT+BAUD="):
                new_baud = parse_baud(cmd)
                UART_SetBaud(new_baud)
                save_config()               // 写到 InfoFlash 持久化
                RF_Send("OK\r\n")
            // ... 其他 AT 指令
        break

// ===== ISP 控制帧处理 =====
void handle_isp_command(packet):
    switch packet.cmd:
    case CMD_ISP_START:
        isp_enter()
        // 等待目标 Bootloader 握手
        if wait_for_bootloader_ack(2s):
            RF_Send(CMD_ISP_ACK, STATUS_OK)
        else:
            isp_exit()                     // 握手失败，复位目标
            RF_Send(CMD_ISP_ACK, STATUS_ERR_NO_BOOTLOADER)

    case CMD_ISP_EXIT:
        isp_exit()
        RF_Send(CMD_ISP_EXIT_ACK, STATUS_OK)

    case CMD_ISP_ABORT:
        isp_exit()                         // 立即退出 ISP
```

## 5. 物理形态选项

### 选项 A：独立小板（推荐）

```
┌─────────────────────────┐
│     陶瓷天线             │
│  ┌──────────────────┐  │
│  │     CH570         │  │
│  │     (DFN10)       │  │
│  └──────────────────┘  │
│  VCC GND TX RX BOOT RST│
│  ┌──────────────────┐  │
│  │  6P 排针 2.54mm   │  │
│  └──────────────────┘  │
└─────────────────────────┘
     ~12×18mm 小板
     直插目标板排母
```

### 选项 B：裸芯片焊盘

```
目标 PCB 上预留 CH570 焊盘 + 天线区域，
需要时手动焊接或 SMT 贴装。
```

### 选项 C：邮票孔模块

```
CH570 + 天线 + 退耦电容 做成带邮票孔的模组，
可以贴到任何板上。
```

## 6. BOM（极简版，DFN10 封装）

### 6.1 基础版（仅无线串口，不含 ISP 控制）

| 序号 | 器件 | 型号/规格 | 封装 | 数量 | 备注 |
|------|------|-----------|------|------|------|
| 1 | 主控 IC | CH570 | DFN10 (3×3mm) | 1 | 或 TSSOP16 |
| 2 | 电容 | 0.1µF | 0402 | 2 | VCC 退耦 |
| 3 | 电容 | 10µF | 0603 | 1 | VCC 大容量 |
| 4 | LED | 0402 | 0402 | 1 | 状态指示（可选） |
| 5 | 电阻 | 1kΩ | 0402 | 1 | LED 限流（可选） |
| 6 | 天线 | 陶瓷天线 AN2051 | SMD | 1 | 或 PCB 天线 |
| 7 | 排针 | 1×6P 2.54mm | — | 1 | VCC/GND/TX/RX/BOOT/RST |
| 8 | PCB | 约 12×18mm | — | 1 | 双面板 |

**基础版 8 项，造价约 ¥3-5。**

### 6.2 ISP 增强版（增加 BOOT0/RESET 外围电路）

在基础版上增加：

| 序号 | 器件 | 型号/规格 | 封装 | 数量 | 备注 |
|------|------|-----------|------|------|------|
| 9 | 电阻 | 10kΩ | 0402 | 1 | BOOT0 下拉（默认低） |
| 10 | 电阻 | 10kΩ | 0402 | 1 | RESET 上拉 |
| 11 | 电容 | 100nF | 0402 | 1 | RESET 去抖 |

**ISP 增强版 11 项，造价约 ¥4-6。**

> 电阻和电容直接集成在从机小板上，目标板上不需要额外元件。6 针排针插上即用。

## 7. 与主机端的协作

| 通信方向 | 从机端职责 | 主机端职责 |
|----------|-----------|-----------|
| 目标←PC | 2.4G 收 → UART 发 | USB CDC 收 → 2.4G 发 |
| 目标→PC | UART 收 → 2.4G 发 | 2.4G 收 → USB CDC 发 |
| 连接管理 | 响应配对，回复心跳 | 发起配对，发送心跳 |
| ISP 控制 | 操作 BOOT0/RESET GPIO | 发送 ISP 指令 |
| 配置 | AT 指令处理 | 透明转发 AT 指令 |

## 8. 进阶功能展望

| 功能 | 说明 | 可行性 |
|------|------|--------|
| 远程复位 | 主机发指令，从机拉 RESET | ✅ 简单（已实现） |
| 远程 ISP 烧录 | 控制 BOOT0 + RESET，UART 透传 ISP 协议 | ✅ 已详细设计（见 3.3 节） |
| 多从机 | 一台主机连多个从机（需要寻址） | ⚠️ 中等 |
| 固件 OTA | 通过无线升级从机端 CH570 固件 | ✅ CH570 原生支持 |
| SWD 透传 | GPIO 模拟 SWD 协议，通过 2.4G 桥接 | ❌ 远程调试不可行（延迟问题），但刷固件可行 |
| 低功耗休眠 | 从机空闲时 RF 休眠，UART RX 唤醒 | ⚠️ 需处理唤醒延迟 |

## 9. 待确认事项

- [ ] 目标 MCU 的 UART 引脚是否支持最高 921600 波特率
- [ ] 2.4G 持续传输 32 字节/包的吞吐量实测（理论上 2Mbps ≈ 250KB/s，实际约 100KB/s）
- [ ] AT 指令的 `+++` 转义序列是否会和正常数据冲突（需加超时保护）
- [ ] ISP 控制脚需要多大的灌电流（评估 CH570 GPIO 驱动能力）
- [ ] 从机端是否需要独立可调的波特率配置（还是固定 115200）
- [ ] 天线在目标板上被金属遮挡时的信号衰减
- [ ] 多套设备同时工作时的频道分配策略
