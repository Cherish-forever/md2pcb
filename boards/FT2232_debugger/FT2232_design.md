# FT2232H 多功能调试器 — 设计文档

> 版本：v0.1（初稿，基于 CH347 设计文档模板，结合 FT2232H 数据手册）
> 日期：2026-06-08
> 状态：草稿，待审核完善

## 1. 项目概述

基于 FTDI FT2232H（LQFP64/QFN64）设计一块 USB 2.0 High-Speed 多功能调试器，目标用途：

- USB Type-C 接口连接电脑（USB 2.0 HS, 480Mbps）
- **双 MPSSE 引擎**：通道 A 用于 JTAG/SWD + SPI，通道 B 用于 UART 串口
- 两侧 2.54mm 排针引出全部可用资源
- STDC14 JTAG/SWD 调试座子（与 STLINK-V3MINI 接口兼容）
- 扩展板支持 SPI Flash / I2C EEPROM 烧录

> **与 CH347F 定位差异**：FT2232H 是 FTDI 第五代芯片，USB 2.0 High Speed（480Mbps），双通道均可独立运行 MPSSE，业界标杆级稳定性。CH347F 更紧凑（QFN28 vs LQFP64），但 FT2232H 的双独立 MPSSE + 更高 USB 带宽使其在**同时 JTAG + SPI + 高速串口**的场景下有绝对优势。

## 2. 芯片选型

### 2.1 FT2232H 家族

| 特性 | FT2232HL (LQFP64) | FT2232HQ (QFN64) | FT2232H-56Q (VQFN56) |
|------|-------------------|-------------------|------------------------|
| 封装 | LQFP64, 10×10mm | QFN64, 9×9mm | VQFN56, 7×7mm |
| 引脚数 | 64 | 64 | 56 |
| 焊接难度 | 较低 | 中等 | 较高 |
| 可用 IO 组 | ADBUS/ACBUS/BDBUS/BCBUS 全量 | 同 LQFP | 部分精简 |
| 供货 | 广泛 | 广泛 | 较少 |

**结论：选择 FT2232HL（LQFP64）。** 手工焊接友好，引脚全引出，供货充足。如果最终需要缩小体积，可降级为 FT2232HQ（QFN64），引脚兼容但需热风枪。

### 2.2 FT2232H vs CH347F 对比

| 特性 | FT2232H (LQFP64) | CH347F (QFN28) |
|------|-------------------|---------------|
| USB 速度 | **USB 2.0 High Speed (480Mbps)** | USB 2.0 Full Speed (12Mbps) |
| 通道数 | 2 个独立通道（均可 MPSSE） | 单芯片，复用模式 |
| JTAG + SPI + UART 同时 | ✅ 通道 A (MPSSE) + 通道 B (UART) | ⚠️ 分时复用 |
| 双 MPSSE | ✅ 可同时挂两个 JTAG 目标 | ❌ 只有一个 JTAG/SWD |
| VIO 独立供电 | ✅ VCCIOA / VCCIOB 独立 (1.8~3.3V) | ✅ VIO 独立 (1.2~3.3V) |
| I/O 耐压 | **5V tolerant** | 3.3V max |
| 晶振 | **12MHz 外部** | 8MHz 外部 |
| EEPROM | **必需 93C46**（存储 VID/PID/配置） | 可选（芯片内置） |
| 封装 | LQFP64 (10×10mm) | QFN28 (4×4mm) |
| 价格 | ~¥30-40 | ~¥8-15 |
| 驱动 | FTDI D2XX / VCP | WCH 专用驱动 |
| OpenOCD 支持 | ✅ 原生支持（ftdi driver） | ✅ 通过 libusb |

**选 FT2232H 的场景**：需要同时高速 JTAG + 高速 UART、需要双 JTAG 链路、需要 5V 耐压 IO 对接老设备、需要业界最稳定的驱动生态。

## 3. 板级布局

```
              ┌──────────────────────────────┐
              │       USB Type-C (上边)        │
              │    (Sink, CC 5.1kΩ下拉)       │
              ├──────────────────────────────┤
              │   93C46 EEPROM              │
              ├──────────────────────────────┤
              │                              │
  左侧排针    │   ┌──────────────────────┐   │   右侧排针
  J1 (10P)    │   │   FT2232HL LQFP64    │   │   J2 (10P)
  通道 A      │   │   (中心位置)         │   │   通道 B
  JTAG+SPI    │   │                      │   │   UART 完整
  + 电源      │   │   [LDO 3.3V + 1.2V]  │   │   + 电源
              │   │   12MHz XTAL         │   │
              │   │   退耦电容 ×10+      │   │
              │   └──────────────────────┘   │
              │                              │
              ├──────────────────────────────┤
              │     STDC14 座 (下边)         │
              │     通道 A: JTAG/SWD        │
              │     通道 B: VCP 串口        │
              └──────────────────────────────┘
```

- **上方**：USB Type-C 母座，VBUS 经 LDO 为板子供电
- **上方右侧**：93C46 EEPROM（配置存储器，必需）
- **下方**：STDC14 1.27mm 2×7 座子，JTAG/SWD（通道 A）+ 虚拟串口（通道 B）
- **左侧**：2.54mm 排针 J1（10针），通道 A 的 JTAG/SPI + 电源
- **右侧**：2.54mm 排针 J2（10针），通道 B 的 UART 完整串口 + 电源
- 预估 PCB 尺寸：30×45mm（LQFP64 较大），双面板
- **中心**：FT2232HL（LQFP64），周围布置 12MHz 晶振、多组退耦电容
- 两侧排针朝上焊接，可直接对插扩展板

## 4. 引脚定义

### 4.1 系统引脚（固定功能）

| 引脚号* | 名称 | 类型 | 功能 | 连接要求 |
|---------|------|------|------|---------|
| 7 | USBDM | I/O | USB D- | 直连 Type-C D-，串 27Ω（可选） |
| 8 | USBDP | I/O | USB D+ | 直连 Type-C D+，串 27Ω（可选） |
| 3 | VPHY | PWR | USB PHY 供电 | 3.3V |
| 4 | VPLL | PWR | PLL 供电 | 3.3V，经磁珠 + 退耦 |
| 1 | VCC | PWR | 核心供电 | 3.3V |
| 37, 64 | VCORE | PWR | 1.2V 核电压 | 外部 LDO 或芯片内置 LDO 输出 |
| 9 | OSCI | I | 晶振输入 | 12MHz 晶振 |
| 10 | OSCO | O | 晶振输出 | 12MHz 晶振 |
| 14 | RESET# | I | 复位（低有效） | 10kΩ 上拉到 VCCIO |
| 15 | TEST | I | 测试引脚 | **12kΩ (±1%) 下拉到 GND**（正常模式） |
| 62 | REF | I | 参考电阻 | **12kΩ (±1%) 到 GND**（内部偏置校准） |
| 35 | EECS | O | EEPROM 片选 | 接 93C46 CS |
| 36 | EECLK | O | EEPROM 时钟 | 接 93C46 SK |
| 38 | EEDATA | I/O | EEPROM 数据 | 接 93C46 DI/DO（需上拉） |
| 11 | VBUS_SENSE | I | USB VBUS 检测 | 分压接到 VBUS（或直接 5V tolerant） |

> \*引脚号基于 LQFP64 封装，**待数据手册最终确认**。

### 4.2 通道 A 引脚（ADBUS / ACBUS）

通道 A 配置为 **MPSSE 模式**，提供 JTAG + SPI 功能。

| 引脚号* | 名称 | MPSSE 功能 | JTAG 功能 | SPI 功能 | GPIO |
|---------|------|-----------|-----------|---------|------|
| 16 | ADBUS0 | TCK/SK | TCK | SCK | — |
| 17 | ADBUS1 | TDI/DO | TDI | MOSI | — |
| 18 | ADBUS2 | TDO/DI | TDO | MISO | — |
| 19 | ADBUS3 | TMS/CS | TMS | CS | — |
| 21 | ADBUS4 | GPIOL0 | — | — | GPIOA0 |
| 22 | ADBUS5 | GPIOL1 | — | — | GPIOA1 |
| 23 | ADBUS6 | GPIOL2 | — | — | GPIOA2 |
| 24 | ADBUS7 | GPIOL3 | — | CS2 (扩展) | GPIOA3 |
| 26 | ACBUS0 | GPIOH0 | SRST* | — | GPIOA4 |
| 27 | ACBUS1 | GPIOH1 | TRST* | — | GPIOA5 |
| 28 | ACBUS2 | GPIOH2 | — | — | GPIOA6 |
| 29 | ACBUS3 | GPIOH3 | — | — | GPIOA7 |
| 30 | ACBUS4 | GPIOH4 | — | — | GPIOA8 |
| 32 | ACBUS5 | GPIOH5 | — | — | GPIOA9 |
| 33 | ACBUS6 | GPIOH6 | LED | — | GPIOA10 |
| 34 | ACBUS7 | GPIOH7 | LED | — | GPIOA11 |

> \*SRST 和 TRST 通过 GPIOH0/GPIOH1 由软件控制，非 MPSSE 硬件自动生成。需要 OpenOCD 配置 `ftdi_layout_signal` 映射。

### 4.3 通道 B 引脚（BDBUS / BCBUS）

通道 B 配置为 **UART 模式**，提供完整的异步串口（含 MODEM 信号）。

| 引脚号* | 名称 | UART 功能 | 方向 | 备注 |
|---------|------|-----------|------|------|
| 39 | BDBUS0 | TXD | O | 串口发送 |
| 40 | BDBUS1 | RXD | I | 串口接收 |
| 41 | BDBUS2 | RTS# | O | 请求发送 |
| 42 | BDBUS3 | CTS# | I | 允许发送 |
| 43 | BDBUS4 | DTR# | O | 数据终端就绪 |
| 44 | BDBUS5 | DSR# | I | 数据设备就绪 |
| 45 | BDBUS6 | DCD# | I | 载波检测 |
| 46 | BDBUS7 | RI# | I | 振铃指示 |
| 48 | BCBUS0 | TXDEN | O | RS485 方向控制 |
| 52 | BCBUS1 | — | — | 预留（可作 GPIO） |
| 53 | BCBUS2 | RXLED# | O | 接收指示 LED |
| 54 | BCBUS3 | TXLED# | O | 发送指示 LED |
| 55 | BCBUS4 | — | — | 预留 |
| 57 | BCBUS5 | — | — | 预留 |
| 58 | BCBUS6 | — | — | 预留 |
| 59 | BCBUS7 | PWRSAV# | O | 休眠指示 |

> \*引脚号基于 LQFP64 封装，**待数据手册最终确认**。通道 B 的 BCBUS 引脚功能较通道 A 少，部分可用于 GPIO 扩展。

### 4.4 关键对比：FT2232H vs CH347F 引脚复用

| 项目 | FT2232H | CH347F |
|------|---------|--------|
| 复用方式 | **无复用**：通道 A/B 独立引脚 | **高复用**：同一引脚多功功能 |
| JTAG + UART 同时 | ✅ 各自独立引脚，无冲突 | ⚠️ 分时切换 |
| SPI + I2C + UART 同时 | ⚠️ 通道 A 做 SPI 则无 I2C 硬件支持 | ⚠️ 分时切换 |
| 冲突管理 | 几乎无冲突（双通道天然隔离） | 需拨码开关管理复用冲突 |

> **FT2232H 的最大优势**：双通道物理隔离，通道 A 全时运行 MPSSE（JTAG/SPI），通道 B 全时运行 UART，互不干扰。不像 CH347F 需要软件切换引脚功能。

## 5. 两侧排针定义

### 5.1 左侧排针：通道 A — JTAG/SPI + 电源（J1，10针）

| 针号 | 信号名 | FT2232H 引脚 | 类型 | 备注 |
|------|--------|-------------|------|------|
| 1 | 5V | — | PWR | USB VBUS 直出 |
| 2 | 3V3 | — | PWR | 板载 LDO 输出 |
| 3 | GND | — | PWR | |
| 4 | TCK / SCK | ADBUS0 (16) | O | JTAG 时钟 / SPI 时钟 |
| 5 | TDI / MOSI | ADBUS1 (17) | O | JTAG TDI / SPI MOSI |
| 6 | TDO / MISO | ADBUS2 (18) | I (FT) | JTAG TDO / SPI MISO |
| 7 | TMS / CS | ADBUS3 (19) | O | JTAG TMS / SPI 片选 |
| 8 | SRST | ACBUS0 (26) | O | 目标系统复位（GPIO 驱动） |
| 9 | TRST | ACBUS1 (27) | O | JTAG TAP 复位（GPIO 驱动） |
| 10 | GND | — | PWR | |

> **说明**：SRST 和 TRST 通过 ACBUS GPIOH 由软件驱动，非 MPSSE 硬件信号。OpenOCD 需配置 `ftdi_layout_signal nSRST -data 0x0100` 等。相比 CH347F 的硬件 SRST/TRST 专用引脚，FT2232H 需要软件配合，但在 JTAG 场景下完全够用。

### 5.2 右侧排针：通道 B — UART 完整串口 + 电源（J2，10针）

| 针号 | 信号名 | FT2232H 引脚 | 类型 | 备注 |
|------|--------|-------------|------|------|
| 1 | 5V | — | PWR | USB VBUS 直出 |
| 2 | 3V3 | — | PWR | 板载 LDO 输出 |
| 3 | GND | — | PWR | |
| 4 | UART_TX | BDBUS0 (39) | O | 串口发送 |
| 5 | UART_RX | BDBUS1 (40) | I | 串口接收 |
| 6 | UART_RTS | BDBUS2 (41) | O | 请求发送 |
| 7 | UART_CTS | BDBUS3 (42) | I | 允许发送 |
| 8 | UART_DTR | BDBUS4 (43) | O | 数据终端就绪 |
| 9 | RS485_DIR | BCBUS0 (48) | O | TXDEN（RS485 自动方向） |
| 10 | GND | — | PWR | |

> **说明**：
> - 通道 B 的 UART 与通道 A 的 JTAG/SPI **完全独立，无任何复用冲突**
> - 支持高达 12 Mbaud 波特率
> - TXDEN 信号（BCBUS0）可配合 RS485 收发器实现自动方向控制
> - 通道 B 的 TXLED#/RXLED#（BCBUS3/BCBUS2）可直接驱动指示灯

## 6. STDC14 座子（J3）引脚定义

与 STLINK-V3MINI (CN5) 完全兼容。**核心变化**：JTAG/SWD 走通道 A MPSSE，VCP 串口走通道 B UART。

| Pin | 信号 | 方向 | FT2232H 引脚 | 说明 |
|-----|------|------|-------------|------|
| 1 | T_VCP_RX | 调试器→目标 | BDBUS0 (39) TXD | 目标板 UART 接收 |
| 2 | T_VCP_TX | 目标→调试器 | BDBUS1 (40) RXD | 目标板 UART 发送 |
| 3 | VCC / VTref | 目标→调试器 | VCCIOA (6) | 目标参考电压 → 通道 A I/O 电平 |
| 4 | JTMS / SWDIO | 调试器→目标 | ADBUS3 (19) TMS | JTAG TMS / SWD 数据 |
| 5 | GND | — | GND | 地 |
| 6 | JTCK / SWCLK | 调试器→目标 | ADBUS0 (16) TCK | JTAG 时钟 / SWD 时钟 |
| 7 | GND | — | GND | 地 |
| 8 | JTDO / SWO | 目标→调试器 | ADBUS2 (18) TDO | JTAG TDO / SWO |
| 9 | NC | — | — | ARM10 KEY，悬空 |
| 10 | JTDI | 调试器→目标 | ADBUS1 (17) TDI | JTAG TDI |
| 11 | GNDDetect | 目标→调试器 | — | 目标端接地以检测连接 |
| 12 | NRST | 调试器→目标 | ACBUS0 (26) | 目标系统复位 |
| 13 | TRST / T_PWR | 调试器→目标 | ACBUS1 (27) | JTAG TAP 复位 / 目标供电 |
| 14 | NC | — | — | 预留 |

> **关键改进**：FT2232H 的双通道使得 VCP 串口（Pin 1/2）走通道 B、JTAG/SWD 走通道 A，两者完全独立。不像 CH347F 需要把 UART0 的 TXD0/RXD0 分给 STDC14。

### 6.1 STDC14 串口指示灯

FT2232H 有专用的 TXLED# / RXLED# 信号（BCBUS3 / BCBUS2），可直接驱动 LED，无需像 CH347F 那样反向驱动。

**TX 指示灯（调试器→目标）**：

```
3V3 ── LED_TX ── R_TX (1kΩ) ── BCBUS3 (TXLED#)
```

- TXLED# 在发送数据时自动拉低，LED 亮起
- 空闲时 TXLED# 为高（或被内部上拉），LED 灭
- 这是 FTDI 硬件自动行为，无需软件干预

**RX 指示灯（目标→调试器）**：

```
3V3 ── LED_RX ── R_RX (1kΩ) ── BCBUS2 (RXLED#)
```

- RXLED# 在接收数据时自动拉低，LED 亮起
- 同样为硬件自动行为

> **布局建议**：TX 绿色、RX 黄色、电源红色。FT2232H 的 LED 驱动是硬件级的，比 CH347F 的"反向驱动 TXD 脚"方案更可靠、更直观。

### 6.2 信号保护

STDC14 的 JTAG/SWD 信号线（Pin 4/6/8/10/12）建议串联 22Ω～100Ω 小电阻：
- 抑制振铃和反射
- 限制短路电流
- ESD 保护配合

所有 STDC14 数字 I/O 线建议对地加 TVS 管（如 SRV05-4 阵列）。FT2232H 的 I/O 自带 5V 耐压，抗静电能力优于 CH347F。

### 6.3 VCCIO 供电策略

FT2232H 有两个独立的 VCCIO 域，分别对应通道 A 和通道 B：

| VCCIO 域 | 供电对象 | 电压范围 | 来源 |
|---------|---------|---------|------|
| VCCIOA | ADBUS[7:0]、ACBUS[7:0]（通道 A JTAG/SPI） | 1.8~3.3V | STDC14 Pin 3（目标板 VTref） |
| VCCIOB | BDBUS[7:0]、BCBUS[7:0]（通道 B UART） | 1.8~3.3V | 板载 3.3V（或跳线选外部） |

```
VCCIOA (Pin 6) ── 跳线选择:
                   ├── 板载 3.3V ── BAT54C (防反灌)
                   │
                   └── STDC14 Pin 3 (VTref) ── 目标板参考电压

VCCIOB (Pin 10) ── 跳线选择:
                    ├── 板载 3.3V（默认）
                    │
                    └── 外部参考（如 1.8V 目标）
```

> **优势**：FT2232H 的双 VCCIO 域允许通道 A 跟随目标板电平（1.8V/2.5V/3.3V），而通道 B 始终 3.3V。这意味着即使 JTAG 目标工作在 1.8V，串口终端仍可输出 3.3V 给 PC 端软件。CH347F 只有一个 VIO，电平统一。

## 7. 电路原理

### 7.1 USB Type-C 接口

```
USB Type-C 母座
├── VBUS ──── PTC (500mA) ──┬── LDO_3.3V 输入端
│                           └── LDO_1.2V 输入端
├── D+  ──────── FT2232H Pin 8 (USBDP)
│                └── 可选串 27Ω 匹配
├── D-  ──────── FT2232H Pin 7 (USBDM)
│                └── 可选串 27Ω 匹配
├── CC1 ──── 5.1kΩ ──── GND
├── CC2 ──── 5.1kΩ ──── GND
├── GND ──────── 公共地
└── SHIELD ───── 外壳接地（可选经 1MΩ + 4.7nF 并联）
```

> **FT2232H 注意事项**：
> - USB D+/D- 可串 27Ω 匹配电阻（FTDI 推荐），但非强制
> - USB 2.0 HS 对差分阻抗要求更高（90Ω ±10%），PCB 布线需注意差分走线
> - VBUS_SENSE (Pin 11) 需要连接到 VBUS（经分压或直连，5V tolerant）
> - PTC 自恢复保险：500mA（MF-NSMF050, SMD 1206）

### 7.2 电源

FT2232H 的电源树比 CH347F 复杂得多：

```
VBUS (5V) ──┬── LDO_3.3V (500mA) ──┬── VCC (Pin 1)
            │                      ├── VPHY (Pin 3)
            │                      ├── VPLL (Pin 4, 经磁珠隔离)
            │                      ├── VCCIOA (Pin 6, 经跳线/防反灌)
            │                      ├── VCCIOB (Pin 10)
            │                      ├── EEPROM VCC
            │                      └── 各 0.1µF + 10µF 退耦
            │
            └── LDO_1.2V (200mA) ──── VCORE1 (Pin 37), VCORE2 (Pin 64)
                                      └── 各 0.1µF + 10µF 退耦
```

**电源去耦要求**（FT2232H 手册通常建议）：

| 供电域 | 电压 | 退耦电容 | 备注 |
|--------|------|---------|------|
| VCC | 3.3V | 0.1µF + 4.7µF | 核心数字供电 |
| VPHY | 3.3V | 0.1µF + 4.7µF | USB 物理层 |
| VPLL | 3.3V | 0.1µF + 4.7µF，经磁珠 | PLL 供电，对噪声敏感 |
| VCORE (×2) | 1.2V | 0.1µF + 4.7µF | 内核电压 |
| VCCIOA | 1.8~3.3V | 0.1µF + 4.7µF | 通道 A I/O |
| VCCIOB | 1.8~3.3V | 0.1µF + 4.7µF | 通道 B I/O |

> **电源方案选择**：
> - **方案一（推荐）**：1.2V 核电压使用独立的 LDO（如 ME6211C12M5, SOT-23-5），3.3V 使用另一个 LDO。两个 LDO 级联或并联均可。
> - **方案二**：如 FT2232HL 内置 1.2V LDO（需确认具体型号），VCORE 引脚只需外接退耦电容，可省去外部 1.2V LDO。

推荐 LDO：
- 3.3V: ME6211C33M5G（SOT-23-5, 500mA, 压差 100mV）
- 1.2V: ME6211C12M5（SOT-23-5, 200mA, 压差 100mV）

### 7.3 时钟

```
FT2232H Pin 9 (OSCI) ──── 12MHz 晶振 ──── FT2232H Pin 10 (OSCO)
                          │           │
                        27pF        27pF
                          │           │
                         GND         GND
```

- 晶振频率：**12MHz**（FT2232H 内部 PLL 倍频到 480MHz）
- CL = 18~20pF（典型值 18pF）
- 负载电容 27pF（含走线杂散约 5pF）
- 可选并联 1MΩ 帮助起振

> 与 CH347F 的 8MHz 晶振不同，FT2232H 必须用 12MHz。这是 FTDI 芯片的关键设计参数，不能替换。

### 7.4 EEPROM 配置

FT2232H **必须**外接 EEPROM（93C46 / 93C56 / 93C66）来存储 USB 描述符和通道配置。

```
93C46 (SOP8)
         ┌───○───┐
    CS ──┤1     8├── VCC (3.3V)
    SK ──┤2     7├── NC
    DI ──┤3     6├── NC
    DO ──┤4     5├── GND
         └──────┘

FT2232H:
  Pin 35 (EECS) ──── CS (Pin 1)
  Pin 36 (EECLK) ─── SK (Pin 2)
  Pin 38 (EEDATA) ── DI (Pin 3) + DO (Pin 4) ── 10kΩ 上拉到 3.3V
```

> **注意**：
> - EECS/EECLK/EEDATA 引脚上可能需要 10kΩ 上拉电阻
> - EEPROM 使用 FT_PROG 工具（Windows GUI）或 ftdi_eeprom（Linux CLI）编程
> - 首次使用前需烧录默认配置（VID: 0x0403, PID: 0x6010）
> - 可自定义 VID/PID/序列号/产品描述/通道模式

### 7.5 复位与 TEST

```
FT2232H Pin 14 (RESET#) ──── 10kΩ 上拉到 VCCIOA
                         └──── 可选按键到 GND

FT2232H Pin 15 (TEST) ──── 12kΩ (±1%) 下拉到 GND

FT2232H Pin 62 (REF) ──── 12kΩ (±1%) 到 GND
```

> **关键**：
> - TEST 引脚必须通过**精确 12kΩ ±1% 电阻**接地，否则芯片无法正常工作
> - REF 引脚的 12kΩ ±1% 电阻用于内部偏置电流校准，精度直接影响 USB 信号质量
> - 这两个 12kΩ 电阻是 FT2232H 设计中最容易被忽略但最关键的元件

### 7.6 退耦电容汇总

FT2232H 引脚多、电源域多，退耦电容数量远超 CH347F：

| 位置 | 数量 | 电容值 | 封装 |
|------|------|--------|------|
| VCC | 1×0.1µF + 1×4.7µF | 0402 + 0603 | 核心供电 |
| VPHY | 1×0.1µF + 1×4.7µF | 0402 + 0603 | USB PHY |
| VPLL | 1×0.1µF + 1×4.7µF | 0402 + 0603 | PLL（经磁珠） |
| VCORE1 | 1×0.1µF + 1×4.7µF | 0402 + 0603 | 内核 1 |
| VCORE2 | 1×0.1µF + 1×4.7µF | 0402 + 0603 | 内核 2 |
| VCCIOA | 1×0.1µF + 1×4.7µF | 0402 + 0603 | 通道 A I/O |
| VCCIOB | 1×0.1µF + 1×4.7µF | 0402 + 0603 | 通道 B I/O |
| LDO_3.3V | 1×0.1µF + 1×10µF | 0402 + 0805 | 3.3V LDO 输入/输出 |
| LDO_1.2V | 1×0.1µF + 1×10µF | 0402 + 0805 | 1.2V LDO 输入/输出 |

**合计：约 14×0.1µF + 10×4.7µF + 4×10µF**

## 8. BOM 清单（初稿）

| 序号 | 器件 | 型号/规格 | 封装 | 数量 | 备注 |
|------|------|-----------|------|------|------|
| 1 | 主控 IC | FT2232HL | LQFP64 (10×10mm) | 1 | 或 FT2232HQ (QFN64) |
| 2 | EEPROM | 93C46B-I/SN | SOP8 | 1 | 必需，存储 USB 配置 |
| 3 | USB 座 | Type-C 母座 16P | SMD 16P | 1 | |
| 4 | LDO 3.3V | ME6211C33M5G | SOT-23-5 | 1 | 3.3V, 500mA |
| 5 | LDO 1.2V | ME6211C12M5 | SOT-23-5 | 1 | 1.2V, 200mA（如芯片无内置） |
| 6 | 晶振 | 12MHz | 5032 或 3225 | 1 | CL≈18pF |
| 7 | 电容 | 27pF | 0402 | 2 | 晶振负载 |
| 8 | 电容 | 0.1µF | 0402 | 14 | 各电源域退耦 |
| 9 | 电容 | 4.7µF | 0603 | 10 | 各电源域退耦 |
| 10 | 电容 | 10µF | 0805 | 4 | LDO 输入/输出 |
| 11 | 电阻 | 5.1kΩ | 0402 | 2 | CC1/CC2 下拉 |
| 12 | 电阻 | 10kΩ | 0402 | 2 | RESET# 上拉 + EEDATA 上拉 |
| 13 | 电阻 | **12kΩ ±1%** | 0402 | 2 | TEST 下拉 + REF 偏置（⚠️ 精度关键） |
| 14 | 电阻 | 27Ω | 0402 | 2 | USB D+/D- 匹配（可选） |
| 15 | 排针 | 1×10P 2.54mm 直插 | — | 2 | J1, J2 |
| 16 | 排针座 | 2×7P 1.27mm 直插 | — | 1 | STDC14 (J3) |
| 17 | ESD 保护 | USBLC6-2 | SOT-23-6 | 1 | USB D+/D- 保护 |
| 18 | PTC 保险 | MF-NSMF050 (500mA) | SMD 1206 | 1 | USB VBUS 自恢复 |
| 19 | 磁珠 | 600Ω@100MHz | 0805 | 1 | VPLL 隔离 |
| 20 | 肖特基二极管 | BAT54C | SOT-23 | 2 | VCCIOA/VCCIOB 防反灌 |
| 21 | 轻触开关 | 6×6mm SMD | — | 1 | 复位按键（可选） |
| 22 | LED | 0402 红色 | 0402 | 1 | 电源指示 |
| 23 | LED | 0402 绿色 | 0402 | 1 | STDC14 TX 指示 |
| 24 | LED | 0402 黄色 | 0402 | 1 | STDC14 RX 指示 |
| 25 | 电阻 | 1kΩ | 0402 | 3 | LED 限流 |
| 26 | 电阻 | 1MΩ | 0402 | 1 | 晶振并联（可选） |
| 27 | 电阻 | 22Ω～100Ω | 0402 | 5 | JTAG/SWD 串联阻尼 |
| 28 | TVS 阵列 | SRV05-4 | SOT-23-6 | 2 | STDC14 + USB ESD 保护 |
| 29 | PCB | 约 30×45mm 1.6mm | — | 1 | 双面板（建议 4 层） |

> **BOM 对比 CH347F**：FT2232H 方案 BOM 项数增加约 30%，主要因为多电源域（LDO×2、退耦电容数量翻倍）+ EEPROM + 磁珠 + 更高的退耦要求。

## 9. 扩展板设计思路

由于 FT2232H 的双通道物理隔离，扩展板设计比 CH347F 更灵活：

### 9.1 SPI Flash 烧录扩展板

```
J1 通道 A 排母 ──┬── 3V3     → SOP8 Pin 8 (VCC)
                 ├── GND      → SOP8 Pin 4 (GND)
                 ├── SCK      → SOP8 Pin 6 (CLK)
                 ├── MOSI     → SOP8 Pin 5 (DI)
                 ├── MISO     → SOP8 Pin 2 (DO)
                 └── CS       → SOP8 Pin 1 (CS#)
```

通道 A 在 MPSSE 模式下可同时提供 JTAG 和 SPI，两不误。烧录 SPI Flash 时 JTAG 仍可用于调试。

### 9.2 双 JTAG 扩展板（特色）

FT2232H 独有的能力——两个通道都可以运行 MPSSE：

```
通道 A MPSSE ── JTAG 链路 1 ── 目标芯片 A（如 MCU）
通道 B MPSSE ── JTAG 链路 2 ── 目标芯片 B（如 FPGA）

同时调试两颗芯片，OpenOCD 支持多 FTDI 通道。
```

此场景下通道 B 的 UART 功能需要让位，但通道 B 的 MPSSE 仍可通过 bit-bang 模拟低速 UART。

### 9.3 电平转换扩展板

FT2232H 的 I/O 是 5V tolerant 的，但输出电平由 VCCIO 决定。对于需要 5V 信号的目标：

```
VCCIO → 1.8V / 2.5V / 3.3V（来自目标或板载）
信号 → TXS0108E 电平转换器 → 5V 目标
```

> 相比 CH347F（VIO max 3.3V），FT2232H 的 5V tolerant 输入意味着至少在接收方向不需要电平转换。

## 10. OpenOCD 配置参考

FT2232H 在 OpenOCD 中使用 `ftdi` 驱动，以下是典型配置：

```
# 通道 A: MPSSE JTAG
interface ftdi
ftdi_vid_pid 0x0403 0x6010
ftdi_channel 0
ftdi_layout_init 0x0038 0x007b
ftdi_layout_signal nSRST -data 0x0100 -noe 0x0100
ftdi_layout_signal nTRST -data 0x0200 -noe 0x0200

# 通道 B: 作为独立 VCP 串口，不需要 OpenOCD 配置
# 系统自动识别为 /dev/ttyUSB1
```

> **与 CH347F 的重大区别**：CH347F 使用 `ch347` 驱动，FT2232H 使用 `ftdi` 驱动。两者在 OpenOCD 中不通用，但 FTDI 驱动生态更成熟（支持更多的目标设备和协议）。

## 11. 待确认事项

- [ ] **引脚号准确性**：所有 FT2232H 引脚号基于 LQFP64 封装，需核对官方数据手册最终确认
- [ ] **1.2V 核电压**：FT2232HL 是否内置 1.2V LDO？如内置则省去外部 LDO
- [ ] **板子最终尺寸**：当前暂定 30×45mm，LQFP64 较大，需实际布局确认
- [ ] **4 层板 vs 2 层板**：FT2232H 电源域多、USB HS 对信号完整性要求高，建议 4 层板
- [ ] **EEPROM 预烧录**：是否需要出厂预烧录默认配置？还是用户自行用 FT_PROG 配置？
- [ ] **VCCIOA 供电策略**：需要独立跳线还是默认跟随 STDC14 Pin 3？
- [ ] **通道 B 预留**：是否需要引出通道 B 的 BCBUS 到排针用于 GPIO 扩展？
- [ ] **双 MPSSE 模式**：是否支持同时打开两个 MPSSE？（OpenOCD 支持但需验证）
- [ ] **REF/TEST 电阻精度**：12kΩ ±1% 至关重要，需在 BOM 中特别标注

## 12. 参考资料

- FT2232H 数据手册 (DS_FT2232H) — FTDI 官网 https://ftdichip.com
- FTDI MPSSE 应用笔记 (AN_129, AN_135 等)
- FTDI FT_PROG EEPROM 编程工具
- OpenOCD FTDI 驱动文档 — https://openocd.org
- STLINK-V3MINI 用户手册 (UM2502) — ST 官网
- CH347 数据手册 (CH347DS1.PDF) — WCH 官网
- Dangerous Prototypes FT2232 Breakout Board — http://dangerousprototypes.com
