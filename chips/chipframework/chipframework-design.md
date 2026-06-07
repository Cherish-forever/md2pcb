# Chip Framework 设计文档

> 状态：草稿，待讨论完善
> 日期：2026-05-17
> 版本：v0.52

---

## 1. 项目概述

**愿景**：让芯片前端设计成本降到 0。任何公司都可以通过 DTS 描述硬件，框架自动生成 Verilog，让芯片设计不再是个封闭的小圈子。

**变革**：

| 现在 | 未来 |
|------|------|
| 需要 RTL 工程师团队 | 写 DTS → 自动生成 Verilog |
| 门槛高、成本高 | 小公司也能做 |
| 封闭的小圈子 | 开放生态 |

**框架 = DTS 前端 + Amaranth 底层 = 芯片设计的 Linux 时刻**

---

## 2. 核心设计哲学

### 2.0 DTS 的职责边界（必读）

> **DTS 只做一件事：实例化设备对象并传参。它不描述 RTL 行为。**

这是理解整个框架的起点。很多误读都源于对 DTS 角色的错误预期。

| 层 | 做什么 | 不做什么 |
|----|--------|---------|
| **DTS** | 实例化模块、传参、声明连接关系 | **不描述**状态机、流水线、时序逻辑、任何行为 |
| **Binding (YAML)** | 声明模块的参数 schema（名称、类型、约束、默认值） | **不描述**模块内部实现 |
| **Python/Amaranth** | 实现全部 RTL 行为（逻辑、状态机、流水线、总线仲裁……） | 不暴露给用户（用户只面对 DTS） |

DTS 是配置层，不是设计层。它告诉框架"实例化什么、配什么参数、连到哪"，所有硬件行为由 Python/Amaranth 模块承载。这类似于 Linux DTS：DTS 告诉内核"这里有个 UART，寄存器基址 0x10000000，时钟 100MHz"，而 UART 驱动的行为逻辑在内核代码里，不在 DTS 里。

框架降低的成本是"拼装已有模块"——如果用户需要的模块已经存在，就只需要写 DTS；如果用户需要全新的模块，则需要在 Python/Amaranth 层实现。后者的难度与模块复杂度成正比，框架不消除这一成本。

### 2.1 一切皆可被 DTS 描述的模块

> **一切皆可被 DTS 描述的模块。框架提供 DTS + 模块的组合机制，让用户构建芯片。**

**核心原则**：一切皆模块，模块皆 DTS，递归嵌套。

**框架的本质**：
- 框架预设了一系列模块（CPU 核、外设、总线路由等）
- 用户通过 DTS 组合这些模块，构建出自己想要的芯片
- 每个模块由一个 binding 文件描述（YAML），包含实现细节和配置参数
- CPU 核的 binding 和 UART/GPIO 的 binding 完全等价，都是 `dts/bindings/` 下的一个文件

**两种方式**：

| 方式 | 说明 |
|------|------|
| **使用预设 CPU** | 直接引用 `compatible = "mychip,rv32im-6stage"` |
| **自己实现 CPU** | 在 `dts/bindings/` 下写一个新的 binding 文件 |

**CPU 核本质上和 UART、GPIO 没有任何区别，都是可配置的模块。**

### 2.2 一切皆 Binding，模块等价

> **CPU 核是一个 binding 文件，不是一个 DTS 文件。**

`compatible` 指向的是 binding 文件（YAML），不是另一个 DTS。CPU 核（fetch/alu/lsu/regfile）是 binding 文件内部的实现细节，不暴露为独立的 DTS 节点。

**DTS 层**：所有模块（CPU、UART、GPIO）都是 DTS 中平等的节点，CPU 没有任何特殊地位。

```
顶层 SoC DTS
│
├── cpu@0 {
│       compatible = "mychip,rv32im-6stage";  ← 引用 binding 文件
│   };
│
├── uart0 { compatible = "mychip,uart"; }
│
└── gpio0 { compatible = "mychip,gpio"; }
```

**Binding 层**（用户不可见，框架内部使用）：

```
# dts/bindings/riscv/rv32im-6stage.yaml
compatible: "mychip,rv32im-6stage"

# CPU 内部由 binding 文件定义，包含：
# - fetch/alu/lsu/regfile 等子模块的实现
# - 流水线配置参数
# - 可选的扩展接口（乘法器、除法器、FPU 等）
```

**关键区分**：
- **DTS**：描述 SoC 的组成模块（哪些 CPU、哪些外设、地址映射）
- **Binding**：描述每个模块的实现细节、配置参数、接口定义
- CPU 核是一个 binding，和 UART/GPIO binding 完全等价

### 2.3 Binding 的三重职责

> **Binding 是模块对外暴露的 schema。用户无需翻阅源码，只需查看 binding 文件就知道有哪些参数、参数名叫什么、合法范围是什么。**

源码定义了参数接口（吃什么参数、什么类型、什么范围），但源码不是给用户读的。binding 把这个接口对外声明出来，形成模块作者与用户之间的契约。

**三重职责**：

| 职责 | 说明 |
|------|------|
| **接口契约** | 声明参数名、类型、默认值、约束范围。用户按 binding 写 DTS，不能乱写 |
| **命名隔离** | DTS 用 `fifo-depth`（kebab-case），Python 用 `fifo_depth`（snake_case），binding 完成映射 |
| **校验边界** | 不合法的值在 binding 层拦下，报"DTS 第 8 行：fifo-depth 应为 1-256 的 2 的幂"，而不是 Python traceback |

**流程**：

```
DTS: fifo-depth = <64>;
        │
        ▼
binding: properties.fifo-depth → type: int, constraint: 2^n, default: 16
        │
        ├── 校验：64 是合法值 ✓
        ├── 映射：fifo-depth → fifo_depth
        │
        ▼
UART16550(fifo_depth=64)    # 模块内部用 Python 命名
```

**没有 binding 时**：用户需要翻源码找参数、猜 DTS 命名规则、写错在实例化时才炸且错误信息是 Python traceback。

**有 binding 后**：binding 是模块自带了说明书，框架在解析 DTS 时就能校验，错误信息精确到 DTS 行号和具体约束。

**校验分层**：

| 层 | 校验内容 | 示例 |
|----|---------|------|
| **Binding 层** | 单参数校验：类型、必填、枚举、固定值、自定义约束 | `fifo-depth` 必须是 2 的幂，范围 1-256 |
| **Python `__init__` 层** | 跨参数依赖校验 | `fifo-depth=64` 且启用了 DMA 时，`fifo-width` 必须为 8 |

Binding 层校验在 DTS 解析阶段执行，错误信息精确到 DTS 行号。跨参数依赖由模块的 `__init__` 自行检查——因为这类依赖是模块实现逻辑的一部分，不适合放在声明式的 binding YAML 中描述。

### 2.4 模块间连接

模块之间的连接在 DTS 中通过 `<&模块 信号名>` phandle 声明，在 `connect()` 中通过 Amaranth Signature 完成实际连接。

**DTS 中声明连接关系**：

```dts
/* 串口连接 CPU0 的控制接口 */
uart0: serial@10000000 {
    compatible = "mychip,uart";
    interrupt-parent = <&cpu0>;
    ctrl = <&cpu0 ctrl>;           /* 引用 cpu0 的 ctrl 信号 */
};

/* SPI 连接 DMA 的 channel 接口 */
spi0: spi@20000000 {
    compatible = "mychip,spi";
    dma-req = <&dma0 channel>;     /* 引用 dma0 的 channel 信号 */
};
```

**connect() 中解析引用并连接**：

```
模块 connect(my_node, full_dts):
    │
    ├── my_node.get_property("ctrl") → phandle &cpu0 + 信号名 "ctrl"
    ├── full_dts 查找到 cpu0 对象
    ├── 取 cpu0 的 Amaranth Signature
    └── Amaranth.connect(my_ctrl_sig, cpu0.ctrl_sig)
```

**设计一致性**：一切皆 DTS，地址映射、时钟域、中断路由、模块间连接全部在 DTS 中通过 phandle 引用声明。

**底层机制**：DTS 中的 `<&cpu0 ctrl>` 在解析后指向目标对象的 Amaranth Signature。`Amaranth.connect()` 对两端的 Signature 做类型检查——同类型的 Signature（如两个 `UARTSignature`）可以直接连接，类型不匹配则在 `soc.connect()` 阶段报错退出。这意味着 DTS 层不需要描述信号级别的接口兼容性——那是 Amaranth Signature 类型系统在连接阶段自动校验的。

### 2.5 模块自连接机制

**职责划分**：

| 组件 | 职责 |
|------|------|
| **DTS 编译器** | DTS 节点检查、DTS 合并、模块实例化、返回 soc 对象 |
| **soc.connect()** | 遍历所有模块，调用每个模块的 `connect()` |
| **模块对象** | `__init__` 只做 HDL 初始化；`connect()` 中自己查找目标、自己连信号 |

```
compile_dts("mychip.dts")
    │
    └── 解析 DTS → 遍历每个有 compatible 的节点
             │
             ├── @ip_register 查找对应的 Python 类
             ├── 实例化：ModuleClass(dts_node)   ← 只做 HDL 初始化
             └── 存入 soc 对象
             
返回 soc
    │
    ▼
soc.connect()
    │
    └── 遍历所有已实例化的模块
          └── module.connect(my_dts_node, full_dts)
                │
                ├── 模块从 full_dts 查找引用的目标
                ├── 模块自己连接信号
                └── full_dts 就是模块发现机制
```

**模块编写**：

```python
class MyModule(ModuleBase):
    def __init__(self, dts, ...):
        # 只做 HDL 逻辑初始化，不做连接
        self.output_signal = Signal(32)

    def connect(self, my_dts_node, full_dts):
        # 从 full_dts 查找引用的目标模块
        target = full_dts.find_module("other_module")
        self.connect_signal(self.output_signal, target.input_signal)
```

**发送方知道要发给谁，接收方不需要声明输入**：

```python
class FetchModule(ModuleBase):
    def __init__(self, dts, ...):
        self.inst_out = Signal(512)     # 只声明输出

    def connect(self, my_node, full_dts):
        decode = full_dts.find_module("decode")  # full_dts 查找目标
        self.connect_signal(self.inst_out, decode.inst_in)
```

**full_dts 即模块发现机制**：不需要独立的模块扫描/注册系统。DTS 解析时所有节点和 `compatible` 已就位，模块在 `connect()` 中直接通过 full_dts 查找引用的节点。

### 2.6 模块开发基本准则

**模块可以自由设计，但接口必须兼容。**

| 可以自由设计 | 约束 |
|-------------|------|
| HDL 逻辑 | 接口类型必须兼容 |
| 内部实现 | 信号连接必须兼容 |
| 处理流程 | 不兼容则构建失败 |
| 功能特性 | |

**验证流程**：

```
soc.connect() 时：
    │
    ├── 模块通过 full_dts 查找引用的目标模块
    │
    ├── 取目标模块的 Amaranth Signature
    │
    ├── Amaranth.connect() 检查签名是否兼容
    │     ├── 兼容 → 自动连接
    │     └── 不兼容 → 报错退出，打印类型不匹配详情
```

**好处**：
- 框架提供灵活性，用户在接口契约内自由发挥
- 配置错误尽早暴露
- 模块自文档化（接口契约即文档）

---

## 3. 目录结构

```
chipframework/
│
├── scripts/                    # 辅助工具脚本
│   ├── __init__.py
│   ├── flash.py               # 烧录命令
│   └── sim.py                 # 仿真命令
│
├── arch/                       # 架构 + 总线
│   │
│   ├── common/                # 通用可复用模块
│   │   ├── exec/              # 执行单元
│   │   │   ├── alu.py         # ALU
│   │   │   └── shifter.py     # 移位器
│   │   │
│   │   └── common/             # 通用模块
│   │       ├── adder.py       # 加法器
│   │       ├── multiplier.py   # 乘法器
│   │       └── decoder.py     # 译码器基类
│   │
│   ├── bus/                    # 总线设计
│   │   ├── __init__.py
│   │   ├── base.py             # 总线基类
│   │   ├── ahb.py              # AHB 总线
│   │   ├── apb.py              # APB 总线
│   │   └── matrix.py           # 总线矩阵
│   │
│   ├── riscv/                  # RISC-V 架构
│   │   ├── core/               # RISC-V 通用设计
│   │   │   ├── regfile.py      # 寄存器组
│   │   │   ├── csr.py          # CSR 定义
│   │   │   ├── isa.py          # ISA 描述
│   │   │   └── decoder.py      # 译码器
│   │
│   ├── arm/                    # ARM 架构（后续）
│   │   └── core/
│   │
│   └── x86/                   # x86 架构（后续）
│
├── devices/                    # 外设设备库（类似 Zephyr drivers）
│   ├── uart/                   # 多种 UART 实现
│   │   ├── 16550.py            # 16550 UART
│   │   ├── liteuart.py         # 轻量级 UART
│   │   └── pl011.py            # ARM PL011
│   │
│   ├── gpio/
│   ├── timer/
│   │   └── riscv_clint.py      # RISC-V CLINT
│   ├── spi/
│   ├── i2c/
│   ├── interrupt/              # 中断控制器
│   │   ├── plic.py             # RISC-V PLIC
│   │   └── nvic.py            # ARM NVIC
│   └── memory/
│       ├── sram.py
│       └── dram.py
│
├── dts/                        # 设备树
│   ├── bindings/               # Binding 文件
│   │   ├── common/             # 通用 binding
│   │   │   ├── simple-bus.yaml
│   │   │   └── interrupt-controller.yaml
│   │   │
│   │   ├── riscv/              # RISC-V CPU binding（固化设计）
│   │   │   ├── rv32im-6stage.yaml       # 6级流水线
│   │   │   ├── rv32im-8stage.yaml       # 8级流水线
│   │   │   ├── rv32im-superscalar-8.yaml  # 超标量8发射
│   │   │   └── rv64gc-rich.yaml          # 64位完整支持
│   │   │
│   │   └── mychip/             # 厂商 binding
│   │       ├── uart.yaml
│   │       ├── gpio.yaml
│   │       ├── timer.yaml
│   │       └── spi.yaml
│   │
│   ├── riscv/                  # RISC-V SoC 的 DTS
│   │   ├── mychip-rv32.dts
│   │   └── mychip-rv64.dts
│   │
│   └── registry/               # IP 注册机制
│       ├── __init__.py
│       ├── base.py            # IP 基类
│       ├── registry.py         # 注册表
│       └── decorators.py       # @ip_register 装饰器
│
├── soc/                        # SoC 代码实现
│   └── mychip/
│
├── boards/                     # 板级支持（类似 Zephyr boards）
│   ├── __init__.py
│   ├── rv32_mychip.py          # RV32 SoC（自构建，含 DTS 引用 + Verilog 生成）
│   └── rv64_mychip.py          # RV64 SoC（后续）
│
├── tests/                      # 测试框架
│   ├── conftest.py
│   ├── framework/              # 测试工具
│   │   ├── simulator.py
│   │   └── verilator_runner.py
│   │
│   └── unit/
│       ├── test_alu.py
│       ├── test_adder.py
│       └── test_shifter.py
│
├── examples/                   # 示例项目
│   ├── basic/
│   │   ├── hello_uart/         # UART 打印示例
│   │   └── blink_led/          # LED 闪烁示例
│   └── riscv/
│       └── minimal_cpu/        # 最小 RISC-V CPU 示例
│
├── docs/                       # 文档
│   ├── api/                    # API 文档
│   ├── design/                 # 设计文档
│   └── tutorial/               # 教程
│
├── include/                    # 公共接口（供用户 IP 调用）
│   ├── __init__.py
│   ├── constants.py            # 公共常量
│   ├── enums.py                # 公共枚举
│   ├── types.py                # 公共类型定义
│   └── utils.py                # 公共工具函数
│
├── .github/                    # GitHub 配置
│   └── workflows/
│       └── ci.yml             # CI 测试
│
├── .gitignore                  # Git 忽略文件
│
├── CONTRIBUTING.md             # 贡献指南
│
├── __init__.py                 # 顶层包入口
├── pyproject.toml
├── README.md
├── LICENSE
└── VERSION                     # 版本号
```

### 与 Zephyr 目录对比

| Zephyr | 你的框架 | 说明 |
|--------|---------|------|
| `arch/riscv/` | `arch/riscv/` | CPU 架构 |
| `dts/` | `dts/` | 设备树源文件 |
| `dts/bindings/` | `dts/bindings/` | Binding 定义 |
| `dts/riscv/` | `dts/riscv/` | RISC-V SoC DTS |
| `drivers/` | `devices/` | 外设实现 |
| `soc/` | `soc/` | SoC 代码 |
| `boards/` | `boards/` | 板级支持 |
| `samples/` | `examples/` | 示例项目 |
| `doc/` | `docs/` | 文档 |
| — | `arch/common/` | 额外：通用微架构模块 |
| — | `arch/bus/` | 总线设计 |
| — | `dts/registry/` | IP 注册机制 |
| — | `boards/` | 板子脚本（自构建） |
| — | `include/` | 公共接口（供用户 IP 调用） |
| — | `.github/` | GitHub 配置 |
| — | `CONTRIBUTING.md` | 贡献指南 |
| — | `VERSION` | 版本号 |
| — | `__init__.py` | 顶层包入口 |

---

## 4. 核心设计思想

### 4.1 CPU 模块化设计

把 CPU 拆成两个层次：

**层次一：零散可复用模块**（`arch/common/`）

```
arch/common/          # 架构无关模块，可自由组合
├── exec/             # ALU、移位器、乘法器、除法器
├── fetch/            # 取指单元（可配置取指宽度：1/2/4/8/16 条）
├── mem/              # 访存单元（LSU）
└── common/           # 寄存器文件、CSR、PMP、分支预测器
```

**层次二：CPU 绑定文件**（`dts/bindings/riscv/`）

```
dts/bindings/riscv/
├── rv32im-6stage.yaml      # 6级流水线，单发射
├── rv32im-8stage.yaml      # 8级流水线，单发射
├── rv32im-superscalar-8.yaml  # 超标量，一次取指8条
├── rv32im-superscalar-16.yaml # 超标量，一次取指16条
└── rv64gc-rich.yaml        # 64位，支持浮点和压缩指令
```

**binding 文件是固化设计**，描述一个完整 CPU 的所有实现细节：取指宽度、流水线级数、发射宽度、是否支持乘法除法、是否有 FPU、缓存大小等。

**用户的使用方式**：

| 方式 | 说明 |
|------|------|
| **使用预置 binding** | 直接引用 `compatible = "mychip,rv32im-6stage"` |
| **自己组合新 binding** | 用 `arch/common/` 的零散模块组合出新的 CPU，写成 binding 文件 |

**用户自己组合 CPU 的例子**：

```
arch/common/fetch/     → 取指单元，支持配置取指宽度
arch/common/exec/      → ALU + 乘法器 + 除法器
arch/common/mem/       → LSU
arch/common/common/    → 寄存器文件 + CSR

用户新写 dts/bindings/riscv/my-custom-8stage.yaml
    - 取指宽度: 8 条指令
    - 流水线: 8 级
    - 发射宽度: 4 发射
    - 支持乘法+除法+FPU

用户的 DTS:
    cpu@0 {
        compatible = "mychip,my-custom-8stage";
    };
```

**设计原则**：
- `arch/common/` 里的模块是"原料"，可以自由组合
- `dts/bindings/` 里的 binding 是"成品"，是固化设计，不可再拆分
- 用户的 DTS 只引用 binding，不涉及模块级别的组合细节

**Boot 流程**：

CPU 上电后从哪取第一条指令，由 CPU binding 的 `reset-vector` 决定。binding 提供默认值，用户 DTS 可覆盖。

```yaml
# dts/bindings/riscv/rv32im-6stage.yaml
compatible: "mychip,rv32im-6stage"
reset-vector: 0x00000000   # 默认复位向量
```

```dts
/* 用户 DTS 覆盖 */
cpu0: cpu@0 {
    compatible = "mychip,rv32im-6stage";
    reset-vector = <0x20000000>;  /* 从 Boot ROM 启动 */
};
```

**设计原则**：

| 规则 | 说明 |
|------|------|
| CPU binding 设默认值 | 不写 `reset-vector` 就用 binding 默认 |
| 用户 DTS 可覆盖 | 和所有 property 一样，遵循 binding → DTS → overlay 的优先级链 |
| Boot ROM / SRAM 是普通外设 | 在 DTS 中声明 `reg = <0x20000000 0x10000>`，和其他外设无异 |

### 4.2 总线抽象

总线设计放在 `arch/bus/`，提供 AHB/APB/AXI 等总线实现。外设继承总线基类接口即可挂载到总线上。

**核心分工**：

| 组件 | 职责 |
|------|------|
| **Bus Binding** | 预留参数，允许用户配置哪些设备绑定到该总线（如 `devices: [uart0, gpio0, spi0]`） |
| **Bus 对象** | 解析自身 DTS 节点下的子节点，收集 `reg` 地址范围，生成地址解码器 |
| **外设** | 声明自己的 `reg` 地址范围，不关心地址解码逻辑 |

**DTS 结构**：

```dts
ahb-bus {
    compatible = "mychip,ahb";

    cpu0: cpu@0 { };

    /* 高速设备挂 AHB */
    dma0: dma@10000000 {
        reg = <0x10000000 0x10000>;
    };
};

apb-bus {
    compatible = "mychip,apb";

    /* 低速外设挂 APB */
    uart0: serial@0 {
        reg = <0x0 0x1000>;
    };

    gpio0: gpio@1000 {
        reg = <0x1000 0x1000>;
    };
};
```

**解析流程**：

```
compile_dts() 遍历 DTS 树
    │
    ├── 遇到 compatible = "mychip,ahb" → 实例化 AHB 对象
    │     └── AHB.__init__(dts_node)
    │           ├── 遍历 dts_node 的子节点
    │           ├── 收集每个子节点的 reg 地址范围
    │           ├── 生成地址解码器（根据地址范围将访问路由到对应外设）
    │           └── 检测地址重叠 → 报错
    │
    └── 遇到 compatible = "mychip,apb" → 实例化 APB 对象
          └── 同上，独立解析自己的地址空间
```

**Binding 中配置设备归属**：

```yaml
# dts/bindings/bus/ahb.yaml
compatible: "mychip,ahb"
bus-type: "AHB"
# devices 参数允许用户指定哪些设备挂到此总线
# 不指定则总线对象自动收集 DTS 子节点
```

```yaml
# dts/bindings/bus/apb.yaml
compatible: "mychip,apb"
bus-type: "APB"
```

用户可以在 DTS 中将外设节点放在对应总线节点下，总线对象自行解析。不同类型总线（AHB/APB/AXI）由各自的 `compatible` 区分，每种总线独立收集并解码自己子节点的地址范围。重叠地址在解析阶段报错。

**多主设备支持**：当总线下挂载了 CPU 或 DMA 控制器等主设备时，总线对象还需处理仲裁——多主同时访问同一外设时，由总线对象的仲裁策略（固定优先级/轮转等）决定访问顺序。仲裁策略作为总线 binding 的可配置参数暴露。

总线协议（AHB/APB/AXI）自身包含仲裁机制和命令缓冲，多主并发访问的死锁规避由总线实现层处理，框架层不需要额外介入。

**外设访问权限**：所有外设的寄存器空间对所有 CPU 可见。但总线对象对每个外设的地址空间做了访问控制——同一时刻只允许一个 CPU 写入，多 CPU 可同时读取。这意味着写操作由总线层串行化，读操作不阻塞。

**数据宽度与对齐**：CPU↔总线↔模块之间的接口全部通过 Amaranth Signature 定义。Signature 限定了每一端的位宽——CPU 端的总线 Signature、总线自身的端口 Signature、模块的总线接口 Signature 三者类型统一。宽度不匹配在 `soc.connect()` 阶段由 `Amaranth.connect()` 检测并报错，不需框架额外处理。不对齐访问（misaligned access）由各 CPU binding 根据 ISA 规范决定——RISC-V 可配置为硬件支持或触发异常。

### 4.3 模块基类 (ModuleBase)

所有模块（外设、CPU核等）都继承同一个模块基类，统一管理。

**ModuleBase 包含**：

```
┌─────────────────────────────────────────┐
│           ModuleBase (基类)              │
├─────────────────────────────────────────┤
│  总线接口    │ 挂载信息、地址、数据宽度     │
│  时钟域     │ 引用 clock-domain 节点，支持多域 │
│  公共功能    │ 初始化、复位、启用、禁用...   │
└─────────────────────────────────────────┘
                     ▲
                     │
         ┌───────────┴───────────┐
         │                       │
   ┌─────┴─────┐          ┌──────┴──────┐
   │  UART16550 │          │   SimpleGPIO │
   │  私有寄存器 │          │  私有寄存器  │
   │  私有功能   │          │  私有功能    │
   └───────────┘          └─────────────┘
```

| 内容 | 说明 |
|------|------|
| **总线接口** | 地址、数据宽度、总线类型 |
| **时钟域** | 引用 clock-domain 节点 |
| **公共功能** | enable(), disable(), reset() |

### 4.4 时钟域设计

> **clock-domain 也是 DTS 节点。一切皆模块，模块皆 DTS。**

**核心原则**：clock-domain 按**工况**定义（全速、均衡、低功耗、深度休眠），不是按模块定义。模块通过 phandle 引用自己支持的时钟域，和其他模块引用的方式完全一致。

**clock-domain DTS 节点**：

```dts
clocks {
    full-speed: clock@0 {
        compatible = "mychip,clock-domain";
        clock-source = <&pll0>;
        frequency = <400000000>;
        reset = <&rst-sys>;
        clock-gate;                /* 允许时钟暂停 */
        enable;                    /* 上电默认使能 */
    };

    balanced: clock@1 {
        compatible = "mychip,clock-domain";
        clock-source = <&pll1>;
        frequency = <100000000>;
        reset = <&rst-sys>;
        clock-gate;                /* 允许时钟暂停 */
    };

    low-power: clock@2 {
        compatible = "mychip,clock-domain";
        clock-source = <&pll2>;
        frequency = <32768>;
        reset = <&rst-sys>;
        clock-gate;                /* 允许时钟暂停 */
    };

    deep-sleep: clock@3 {
        compatible = "mychip,clock-domain";
        clock-source = <&pll3>;
        frequency = <32768>;
        reset = <&rst-sys>;
        clock-gate;                /* 允许时钟暂停 */
    };
};
```

**模块引用 clock-domain**：

```dts
/* 多工况模块：支持多个时钟域，按场景切换 */
uart0: serial@10000000 {
    compatible = "mychip,uart";
    clock-domain = <&full-speed>, <&balanced>, <&low-power>;
};

/* 固定时钟模块：不管什么工况都需要固定时钟 */
usb0: usb@20000000 {
    compatible = "mychip,usb";
    clock-domain = <&full-speed>;
};
```

**设计原则**：

| 规则 | 说明 |
|------|------|
| clock-domain 是 DTS 节点 | 和 GPIO、UART 一样，有 `compatible`，被 phandle 引用 |
| 按工况定义 | full-speed / balanced / low-power / deep-sleep，对应功耗-性能频谱 |
| 模块声明自己支持哪些 | `clock-domain` 是 phandle list，多工况模块列多个，固定模块列一个 |
| 框架/软件选择活跃域 | 根据当前功耗策略选择对应时钟源，切换时框架自动处理 |
| Binding 可指定默认值 | 模块 binding 可以声明默认的 clock-domain，用户 DTS 可覆盖 |
| `clock-gate` | boolean，该时钟域是否支持门控暂停（框架可动态开关时钟） |
| `enable` | boolean，上电后该时钟域默认使能，不写则默认禁用 |

**Binding 中声明默认**：

```yaml
# dts/bindings/mychip/uart.yaml
compatible: "mychip,uart"

clock-domain:
  default: ["balanced"]
  choices: ["full-speed", "balanced", "low-power"]
```

用户 DTS 必须从 `choices` 中选择，不写就用 `default`。

**CDC 实现**：跨时钟域信号同步使用 Amaranth 标准库 `amaranth.lib.cdc`，包括 `FFSynchronizer`（单比特同步）、`AsyncFIFO`（多比特数据跨域）等。框架在信号跨越 clock-domain 边界时自动插入对应的 CDC 组件，模块作者无需手写同步器。

```
功耗工况切换：
        │
        ▼
┌─────────────────────────────────────┐
│  全速 (400MHz)    │  均衡 (100MHz)   │
│  PLL0 + rst-sys   │  PLL1 + rst-sys  │
├───────────────────┼──────────────────┤
│  低功耗 (32KHz)   │  深度休眠 (32KHz) │
│  PLL2 + rst-sys   │  PLL3 + rst-sys  │
└─────────────────────────────────────┘
        │
        ▼
   模块选择：UART → full-speed / balanced / low-power
            USB  → full-speed（固定）
```

### 4.5 中断路由设计

每个 CPU 核有自己的中断模块，外设可以选择挂载到哪个 CPU 的中断模块。

**中断配置层级**：

| 层级 | 定义位置 | 说明 |
|------|----------|------|
| **默认值** | Binding 文件 | `interrupt-parent = cpu0` |
| **覆盖** | 用户 DTS | 可选 |
| **修改** | 用户 overlay | 可选 |

**架构**：

```
┌─────────────────────────────────────────────────────┐
│  CPU 0                                              │
│  ┌─────────────────────────────────┐               │
│  │ PLIC/NVIC                       │               │
│  │  external interrupts: 0-31       │               │
│  └─────────────────────────────────┘               │
│       ↑          ↑          ↑                     │
│       │          │          │                     │
│    UART0       GPIO0      TIMER0                    │
│                                                     │
├─────────────────────────────────────────────────────┤
│  CPU 1                                              │
│  ┌─────────────────────────────────┐               │
│  │ PLIC/NVIC                       │               │
│  │  external interrupts: 0-31       │               │
│  └─────────────────────────────────┘               │
│       ↑          ↑                                │
│       │          │                                │
│    SPI0       DMA0                                │
└─────────────────────────────────────────────────────┘
```

**Binding 定义**：

```yaml
# dts/bindings/mychip/uart.yaml
compatible: "mychip,uart"

interrupt-parent: "cpu0"   # 默认挂到 CPU0
```

**核间中断 (IPI)**：IPI 是一个独立的中断模块，有自己的 binding 和 DTS 节点，和 PLIC、GPIO 等模块完全平等。多核场景下，CPU 通过 `interrupt-parent` 引用对应的 IPI 控制器，核间中断的路由和外部中断使用同一套 phandle 机制。

```dts
/* IPI 作为独立模块 */
ipi0: ipi-controller {
    compatible = "mychip,ipi";
    interrupt-parent = <&cpu0>, <&cpu1>;  /* 连接两个 CPU */
};
```

### 4.6 IOMUX 设计

IOMUX 控制器负责引脚功能复用。每个引脚有自己的名字，值是功能选择列表。

**GPIO 模块**：

```dts
/* SoC DTS 中定义 GPIO 控制器 */
gpioa: gpio@30000000 {
    compatible = "mychip,gpio";
    reg = <0x30000000 0x1000>;
};

gpiob: gpio@30001000 {
    compatible = "mychip,gpio";
    reg = <0x30001000 0x1000>;
};
```

**引脚格式**：

```dts
iomux: iomux@20000000 {
    compatible = "mychip,iomux";
    reg = <0x20000000 0x1000>;

    /* GPIO 组 A */
    GPIOA {
        PA0  = <&gpioa 0>, <&uart4 tx>;   /* 功能 0: gpioa[0] 作为 GPIO */
        PA1  = <&gpioa 1>, <&uart4 rx>;   /* 功能 1: uart4 rx */
    };

    /* GPIO 组 B */
    GPIOB {
        PB0  = <&gpiob 0>, <&spi0 sck>;   /* 功能 0: gpiob[0] 作为 GPIO */
    };

    /* 也可以叫 GPIO0、GPIO1 */
    GPIO0 {
        P0   = <&gpioa 2>, <&spi0 mosi>;
    };
};
```

引脚嵌套在 GPIO 容器节点内，标明所属的 GPIO 组。每个引脚的属性值是功能选择列表（有序，第一个是默认功能）。

**格式解析**：

```
<&gpio组 bit索引> → GPIO bit 引用（gpioa, gpiob, gpioc...）
<&模块 信号名>   → 模块信号签名（tx, rx, scl, sda...）
```

**自动匹配流程**：

```
引脚 GPIOA.PA1 选择功能 1 = <&uart4 rx>
        │
        ├── 找到 uart4 模块
        ├── 获取 uart4.rx 信号的 signature
        └── 连接 uart4.rx → 引脚 PA1
```

**设计原则**：
- 外设只需要声明有 tx, rx 接口
- IOMUX 定义引脚到信号的映射
- 框架自动查找签名并连接
- 用户可通过 overlay 修改引脚映射

### 4.7 完整 SoC DTS 示例

以下是一个完整的 SoC DTS 示例，展示所有模块的组合：

```dts
/ {
    /* 时钟域 */
    clocks {
        full-speed: clock@0 {
            compatible = "mychip,clock-domain";
            clock-source = <&pll0>;
            frequency = <400000000>;
            reset = <&rst-sys>;
            clock-gate;
            enable;
        };

        balanced: clock@1 {
            compatible = "mychip,clock-domain";
            clock-source = <&pll1>;
            frequency = <100000000>;
            reset = <&rst-sys>;
            clock-gate;
            enable;
        };

        low-power: clock@2 {
            compatible = "mychip,clock-domain";
            clock-source = <&pll2>;
            frequency = <32768>;
            reset = <&rst-sys>;
            clock-gate;
        };

        deep-sleep: clock@3 {
            compatible = "mychip,clock-domain";
            clock-source = <&pll3>;
            frequency = <32768>;
            reset = <&rst-sys>;
            clock-gate;
        };
    };

    /* 多核 CPU 容器 */
    cpus {
        cpu0: cpu@0 {
            compatible = "mychip,rv32im";
            clock-domain = <&full-speed>, <&balanced>;
        };

        cpu1: cpu@1 {
            compatible = "mychip,rv32im";
            clock-domain = <&full-speed>, <&balanced>;
        };
    };

    /* GPIO 控制器 */
    gpioa: gpio@30000000 {
        compatible = "mychip,gpio";
        reg = <0x30000000 0x1000>;
        clock-domain = <&balanced>, <&low-power>;
    };

    gpiob: gpio@30001000 {
        compatible = "mychip,gpio";
        reg = <0x30001000 0x1000>;
        clock-domain = <&balanced>, <&low-power>;
    };

    /* 串口 */
    uart4: serial@40001000 {
        compatible = "mychip,uart";
        reg = <0x40001000 0x1000>;
        interrupt-parent = <&cpu0>;
        clock-domain = <&full-speed>, <&balanced>, <&low-power>;
    };

    /* 固定时钟模块 */
    usb0: usb@20000000 {
        compatible = "mychip,usb";
        reg = <0x20000000 0x1000>;
        clock-domain = <&full-speed>;
    };

    /* IOMUX 控制器 */
    iomux: iomux@20000000 {
        compatible = "mychip,iomux";
        reg = <0x20000000 0x1000>;
        clock-domain = <&balanced>;
    };
};
```

### 4.8 模块接口标准化

**设计原则**：接口统一，实现灵活。

不同外设实现（16550, PL011, LiteUART）：
- **接口统一**：tx, rx
- **内部实现不同**：寄存器布局不同

**好处**：
- 可替换：可以换不同实现，不改连接
- 可测试：可以用 LiteUART 替换 16550 测试
- 灵活性：内部实现自由，内部寄存器按需设计

**标准接口签名**：

```python
# 标准 UART 签名
class UARTSignature(wiring.Signature):
    def __init__(self):
        super().__init__()
        self.tx = Out(8)   # 输出
        self.rx = In(8)    # 输入

# 标准 GPIO 签名
class GPIOSignature(wiring.Signature):
    def __init__(self):
        super().__init__()
        self.out = Out(1)
        self.in_ = In(1)

# 标准 I2C 签名
class I2CSignature(wiring.Signature):
    def __init__(self):
        super().__init__()
        self.scl = Out(1)
        self.sda = In(1)
        self.sda_out = Out(1)

# 标准 SPI 签名
class SPISignature(wiring.Signature):
    def __init__(self):
        super().__init__()
        self.mosi = Out(1)
        self.miso = In(1)
        self.sck = Out(1)
        self.cs = Out(1)
```

**模块实现示例**：

```python
# 16550 UART - 实现 A
class UART16550(ModuleBase):
    def __init__(self):
        super().__init__()
        self.uart = UARTSignature()  # 统一接口

# PL011 UART - 实现 B
class PL011UART(ModuleBase):
    def __init__(self):
        super().__init__()
        self.uart = UARTSignature()  # 统一接口

# LiteUART - 实现 C
class LiteUART(ModuleBase):
    def __init__(self):
        super().__init__()
        self.uart = UARTSignature()  # 统一接口
```

所有实现都暴露相同的 `uart.tx`、`uart.rx` 接口，但内部寄存器不同。用户通过 `compatible` 选择实现。

### 4.9 IP 注册与参数传递

**注册**：`@ip_register` 装饰器把 `compatible` 字符串映射到对应的 Python 类。

```python
# devices/uart/16550.py
from chipframework.dts.registry import ip_register

@ip_register("ns16550", "mychip,uart-16550")
class UART16550(ModuleBase):
    def __init__(self, dts_node):
        super().__init__(dts_node)
        self.uart = UARTSignature()
```

**参数传递**：`__init__` 只接收 `dts_node`，模块自己从中取参数。

```python
class UART16550(ModuleBase):
    def __init__(self, dts_node):
        super().__init__(dts_node)
        # binding 已在校验阶段完成类型/范围检查，此处直接取
        self.fifo_depth = dts_node.get_property("fifo-depth", default=16)
        self.baud_rate  = dts_node.get_property("baud-rate", default=115200)
```

**校验链**：

```
DTS 节点属性
    │
    ├── binding properties schema（类型、范围、约束、枚举）
    ├── 校验不通过 → 报错，指明 DTS 行号和具体约束
    │
    ▼
UART16550(dts_node)
    └── get_property("fifo-depth", default=16)  ← 拿到的已经是校验过的值
```

**兼容属性链**：`@ip_register` 支持多个 compatible，用于 IP 演进时保持向后兼容。

```python
# v1 时代
@ip_register("mychip,uart-v1")
class UART16550(ModuleBase):
    def __init__(self, dts_node):
        # v1 功能

# 一年后发布 v2，加新功能
@ip_register("mychip,uart-v2", "mychip,uart-v1")
class UART16550V2(ModuleBase):
    def __init__(self, dts_node):
        self.fifo_depth = dts_node.get_property("fifo-depth", default=16)
        self.dma_enable = dts_node.get_property("dma", default=False)  # v2 新增
```

旧用户的 DTS 不需要改，`compatible = "mychip,uart-v1"` 仍然能匹配到新类，实例化后 self.dma_enable=False 走老路径。

**DTS 中的兼容链**（多选一，框架按顺序匹配）：

```dts
uart0: serial@10000000 {
    compatible = "mychip,uart-v2", "mychip,uart-v1";  /* 优先匹配 v2 */
};
```

**核心作用**：
- 从 DTS 的 `compatible` 属性按顺序匹配对应的 Python 类
- 绑定 binding properties 进行参数校验
- 传入 dts_node，模块自行取参数
- 一份代码兼容多版本 DTS

---

## 5. compile_dts 机制

### 5.1 两阶段：实例化 + 连接

模块实例化采用**两阶段模式**，`compile_dts()` 只负责实例化，连接由 `soc.connect()` 完成。

**阶段一 `compile_dts()`：实例化所有模块**

```
compile_dts("mychip.dts", overlay="overlay.dts")
    │
    ├── 合并 base.dts + overlay.dts
    │
    ├── 遍历每个有 compatible 的 DTS 节点
    │     ├── @ip_register 查找对应的 Python 类
    │     ├── ModuleClass(dts_node)  ← 只做 HDL 初始化
    │     └── 存入 soc 对象
    │
    └── 返回 soc 对象
```

模块 `__init__` 只初始化 HDL 逻辑和参数，不做任何连接。

**阶段二 `soc.connect()`：连接所有模块**

```
soc.connect()
    │
    └── 遍历所有已实例化的模块
          └── module.connect(my_dts_node, full_dts)
                │
                ├── 从 full_dts 查找引用的目标模块
                ├── 模块自己连接信号
                └── full_dts 即模块发现机制
```

**connect() 是可选的**：

| 模块类型 | 需要 connect() |
|----------|---------------|
| 有外部连接的模块 | ✅ 需要 |
| 独立模块（只连总线） | ❌ 不需要 |

**好处**：
- 模块可以在任何顺序下创建，`__init__` 不依赖其他模块
- 连接在实例化之后统一进行
- `full_dts` 作为发现机制，无需独立的模块注册/扫描系统

### 5.2 完整流程

```bash
cd Grain && PYTHONPATH=/home/wei/work python3 boards/rv32_mychip.py
```

```
1. 板子脚本引用 DTS 文件
   └─ soc = compile_dts_file("dts/riscv/mychip-rv32.dts")
                ↓
2. _discover_modules() 导入所有模块，触发 @ip_register
   └─ 32 个模块类注册到 IP registry
                ↓
3. 解析 DTS + 合并 overlay
   └─ 新增节点：直接在 {} 内添加
   └─ 修改属性：用 &node@addr 引用
   └─ 节点不存在 → 报错
                ↓
4. 遍历所有 compatible 节点，@ip_register 查找类，实例化
   └─ 模块 __init__ 只做 HDL 初始化
   └─ compatible 找不到对应实现 → 报错
                ↓
5. 返回 soc 对象
                ↓
6. soc.connect()
   └─ 遍历模块，调用 connect(my_node, full_dts)
   └─ 模块自己查找目标、自己连信号
                ↓
7. verilog.convert(soc, ports=[])
   └─ Amaranth 后端 → Verilog 文件
```

### 5.3 DTS Overlay 机制

类似 Zephyr，用户可以覆盖/扩展现有 DTS：

```dts
/* my-overlay.dts */

/* 修改已有节点 */
&uart0 {
    clock-frequency = <100000000>;  /* 覆盖 */
};

/* 新增节点 */
&bus {
    my_encoder: encoder@20000000 {
        compatible = "mychip,encoder";
        reg = <0x20000000 0x10000>;
    };
};
```

### 5.4 SoC 类的动态性

不同的 DTS 编译出不同的 SoC 类：

```python
MySoC1 = compile_dts("chip_a.dts")   # 有 UART + GPIO
MySoC2 = compile_dts("chip_b.dts")   # 有 SPI + I2C + DMA

soc1.uart0     # 存在
soc1.spi0      # 没有

soc2.uart0     # 没有
soc2.spi0      # 存在
soc2.dma0      # 存在
```

---

## 6. 用户 workflow

用户通过 DTS 描述硬件，板子脚本负责构建并输出 Verilog。

**板子脚本即构建入口**。每个板子是一个自包含的 Python 文件，引用 DTS 并完成编译+连接+生成：

```python
# boards/rv32_mychip.py
import sys
from pathlib import Path
sys.path.insert(0, str(Path(__file__).parent.parent.parent))

from grain import compile_dts_file
from amaranth.back import verilog

soc = compile_dts_file("dts/riscv/mychip-rv32.dts")
soc.connect()

v = verilog.convert(soc, ports=[])
Path("build/mychip-rv32.v").write_text(v)
```

**运行**：

```bash
cd Grain && PYTHONPATH=/home/wei/work python3 boards/rv32_mychip.py
```

**DTS 文件独立保存**，板子脚本只引用不内联：

```
dts/riscv/mychip-rv32.dts   ← 硬件描述（纯 DTS）
boards/rv32_mychip.py       ← 构建脚本（引用 DTS + 生成 Verilog）
```

**框架只负责前端**：DTS 解析 → 模块发现 → 实例化 → 信号连接 → 输出 Verilog。综合到比特流由下游工具（Vivado/Quartus）完成。

---

## 7. Amaranth 作为底层引擎

框架在 Amaranth 之上构建，底层能力全部来自 Amaranth：

| 能力 | 使用 |
|------|------|
| **HDL 生成** | `amaranth.back.verilog` |
| **CDC** | `amaranth.lib.cdc`（FFSynchronizer、AsyncFIFO 等） |
| **仿真** | `amaranth.sim` |
| **信号连接** | Amaranth Signature + `connect()` |

框架不重新实现这些底层能力，专注于 DTS 解析、模块组合、参数校验、连接编排。

**构建流程**：

```
DTS + Binding
      │
      ▼
compile_dts()    ← 框架：解析 DTS、校验参数、实例化模块
      │
      ▼
soc.connect()    ← 框架：遍历模块，调用连接逻辑
      │
      ▼
Amaranth 输出    ← Amaranth 后端：生成 Verilog
```

---

## 8. 三级验证体系

仿真使用 `amaranth.sim`，框架提供测试辅助（激励注入、覆盖率收集、参考模型比对）。

### 8.1 模块级验证（Unit Verification）

**验证对象**：`arch/common/` 下的每个独立模块

| 模块类型 | 验证内容 | 测试方法 |
|----------|----------|----------|
| ALU | 加减乘除、与或非移位 | 随机向量 + 参考模型比对 |
| RegFile | 读写功能、读写冲突 | 穷举所有寄存器读写 |
| LSU | 加载存储、不同数据宽度 | 边界测试 |
| BranchPredictor | 分支预测准确率 | trace-driven 仿真 |
| Cache | 命中/未命中、替换策略 | 随机访问序列 |

### 8.2 内核级验证（Core Verification）

**验证对象**：完整的 CPU 内核

| 测试类型 | 测试内容 | 方法 |
|----------|----------|------|
| ISA 合规性 | 执行 RISC-V ISA 测试套件 | riscv-tests |
| 指令覆盖 | 覆盖所有指令类型 | 随机指令生成 |
| 异常处理 | 非法指令、地址错误、ECALL 等 | 异常注入测试 |
| 中断响应 | 定时器中断、外部中断 | 中断压力测试 |
| 流水线冒险 | 数据 hazard、控制 hazard | 定向 hazard 测试 |

### 8.3 芯片级验证（SoC Verification）

**验证对象**：完整 SoC（CPU + 总线 + 外设）

| 测试类型 | 测试内容 | 测试方法 |
|----------|----------|----------|
| 外设功能 | UART/GPIO/SPI/I2C 等 | 外设驱动 + 环路测试 |
| 总线互联 | 地址路由、读写时序 | 总线 monitor 覆盖率 |
| 中断路由 | 外设→CPU 中断路径 | 中断触发验证 |
| 内存映射 | 验证各外设地址映射正确 | 地址扫描测试 |

---

## 9. 软件栈

- 基于 Zephyr 开发
- 由用户自行集成，不在框架范围内

---

## 10. Git 扩展机制

```
chipframework (主仓库)
    │
    ├── scripts/           # 构建脚本
    ├── arch/             # 架构（common/bus/riscv/arm/x86）
    ├── devices/          # 设备库
    ├── dts/              # DTS + bindings + registry
    ├── include/          # 公共接口
    ├── soc/              # 官方 SoC 支持
    ├── boards/           # 官方板级支持
    ├── tests/            # 测试
    ├── examples/         # 示例
    └── docs/              # 文档
    │
    └── (扩展)             # 通过 git submodule 或类似机制
          │
          ├── vendor/chip-a    # 厂商 A 的 IP (downstream)
          ├── vendor/chip-b    # 厂商 B 的 IP
          └── community/xxx    # 社区扩展
```

---

## 11. 里程碑规划

### M1: 框架搭建 + 通用模块

**目标**：
- 目录结构确定
- 基础通用模块实现（带测试）
- 测试框架搭建完成
- 可验证：能跑通 "模块 → Verilog 生成 → 仿真测试" 流程

**交付物**：
```
chipframework/
├── boards/                      # 板子脚本（自构建入口）
│   └── rv32_mychip.py
├── arch/
│   ├── common/
│   │   ├── exec/
│   │   │   ├── alu.py          # P0
│   │   │   └── shifter.py      # P0
│   │   └── common/
│   │       ├── adder.py        # P0
│   │       └── multiplier.py   # P1
│   │
│   └── bus/                    # 总线基类 (P0)
│       ├── base.py
│       ├── ahb.py
│       ├── apb.py
│       └── matrix.py
│
├── dts/
│   └── registry/               # IP 注册机制 (P0)
│       ├── base.py
│       ├── registry.py
│       └── decorators.py
│
├── tests/unit/                 # 单元测试
├── pyproject.toml
├── README.md
├── LICENSE
└── VERSION
```

**验收标准**：
- ✅ 目录结构存在且符合规划
- ✅ adder.py 可生成 Verilog
- ✅ shifter.py 可生成 Verilog
- ✅ alu.py 可生成 Verilog（集成 adder + shifter）
- ✅ 所有模块单元测试通过 (pytest)
- ✅ 生成的 Verilog 可被仿真器综合

---

## 12. 待讨论问题

### 已确认

| 主题 | 决定 |
|------|------|
| 许可证 | Apache 2.0 |
| 软件栈 | 基于 Zephyr，用户自行集成 |
| 注册机制 | 装饰器 `@ip.register` |
| 模块接口 | Sink/Source + 专用接口 |
| 目录结构 | 与 Zephyr 对齐 |
| 总线位置 | `arch/bus/` |
| Compatible | 一对多：同一 IP 可注册多个 compatible 字符串，支持版本演进 |
| Git 管理 | 单仓库 + 扩展机制 |
| DTS 语法 | 标准 Linux DTS，`<&module signal>` phandle |
| Binding 格式 | YAML，参考 Zephyr binding 格式，扩展 constraint/signals/clock-domain |

### 待确认

| 主题 | 说明 |
|------|------|
| 模块间通信协议 | 具体定义？ |
| 多核/异构 | 设计预留还是初期实现？ |
| 外设优先级 | 除了 UART/GPIO/Timer，还需什么？ |

---

## 附录 A: 待完善的架构设计点

以下是需要进一步分析和设计的维度，按优先级排列：

| 优先级 | 维度 | 说明 |
|--------|------|------|
| 中 | **跨模块信号** | 除了点对点连接，有没有广播信号？ |
| 中 | **异常/中断处理** | CPU 内部异常怎么路由？陷阱向量怎么描述？ |
| 中 | **安全特性** | PMP、RISC-V 特权模式（m-mode/u-mode）怎么描述？ |
| 中 | **缓存一致性** | 多核场景下缓存一致性怎么管理？ |
| 中 | **调试接口** | JTAG/SWD、断点、观察点怎么描述？ |
| 低 | **内存属性** | Cacheable、bufferable 等属性怎么描述？ |
| 低 | **DMA 通道申请** | 软件怎么申请 DMA 通道？ |
| 低 | **中断优先级** | PLIC 优先级怎么配置？ |
| 低 | **复位层级** | 冷复位、热复位、局部复位怎么组织？ |
| 低 | **仿真支持** | 怎么和 Verilator/仿真器对接？ |
| 低 | **形式验证** | 怎么给模块附加 formal properties？ |

**目标**：让框架足够灵活、可扩展、更抽象。

---

## 变更记录

| 日期 | 版本 | 变更内容 |
|------|------|----------|
| 2026-04-18 | v0.1 | 初始版本，基于讨论 |
| 2026-04-18 | v0.2 | 新增 `examples/`、`docs/`、`boards/`、`VERSION`；IP 注册机制移至 `dts/registry/` |
| 2026-04-18 | v0.3 | 新增用户 workflow（Section 3.5）、`include/` 目录、`__init__.py` 顶层入口 |
| 2026-04-18 | v0.4 | 新增 `.github/` (CI配置)、`.gitignore`、`CONTRIBUTING.md` (贡献指南) |
| 2026-04-18 | v0.5 | 新增核心设计哲学：一切皆可被 DTS 描述的模块 |
| 2026-04-18 | v0.6 | 更新用户 workflow：命令行驱动 `chipframework build -b board -s script.py` |
| 2026-04-18 | v0.7 | 补充核心设计哲学：框架提供 DTS + 模块组合机制，用户可使用预设 CPU 或自己组合 |
| 2026-04-18 | v0.8 | 新增 compile_dts 内部机制（解析外设地址、挂载到总线） |
| 2026-04-18 | v0.9 | 简化总线设计：总线只关心地址，DTS 树形结构描述总线与外设关系 |
| 2026-04-18 | v0.10 | 完善 compile_dts 流程：合并 DTS → 生成 build_space/soc.py → import 返回类 |
| 2026-04-18 | v0.11 | 新增时钟域设计（clock-domain 分层、PLL 后端时钟分配模块） |
| 2026-04-18 | v0.12 | 新增外设公共寄存器抽象（时钟选择、电源开关、分频系数等） |
| 2026-04-18 | v0.13 | 重构为模块基类 (ModuleBase)，包含总线接口、时钟-复位域、公共寄存器、公共功能 |
| 2026-04-18 | v0.14 | 新增中断路由设计（每个 CPU 有独立中断模块，外设默认挂到 CPU0） |
| 2026-04-18 | v0.15 | 新增愿景声明：让芯片前端设计成本降到 0 |
| 2026-04-18 | v0.16 | 新增核心原则：DTS 嵌套结构，一切皆模块、模块皆 DTS、递归嵌套 |
| 2026-04-18 | v0.17 | 新增模块互联设计：模块连接在 DTS 中描述，Amaranth 自动处理互联 |
| 2026-04-18 | v0.18 | 新增 IOMUX 设计（引脚功能复用、外设自动匹配） |
| 2026-04-18 | v0.19 | 新增待完善的架构设计点清单（15个维度） |
| 2026-04-18 | v0.20 | 重构 compile_dts 机制：模块自连接机制（模块自己处理信号连接） |
| 2026-04-18 | v0.21 | 新增模块开发基本准则（可以自由设计，但接口必须兼容） |
| 2026-04-18 | v0.22 | 新增两阶段实例化 + 延迟连接设计（connect_all 可选） |
| 2026-04-18 | v0.23 | 更新 IOMUX 设计：<&module signal> 格式，每个 <> 是单个功能，指向模块信号签名 |
| 2026-04-18 | v0.24 | 更新 IOMUX 设计：引脚改为数字编号，更符合 FPGA 实际情况 |
| 2026-04-18 | v0.25 | ~~新增 IOMUX 分离设计：IOMUX 控制器在 devices/，引脚映射在 boards/pinmap.dts~~ (v0.41 移除 pinmap) |
| 2026-04-18 | v0.26 | 新增 GPIO 模块说明：gpioa, gpiob, gpioc 在 SoC DTS 中定义 |
| 2026-04-18 | v0.27 | 新增完整 SoC DTS 示例，展示 CPU 容器、GPIO、UART、IOMUX 组合 |
| 2026-04-18 | v0.28 | 新增模块接口标准化设计：Signature 类定义统一接口，不同实现可替换 |
| 2026-04-18 | v0.29 | 新增 IP 注册机制说明：@ip_register 装饰器把 compatible 映射到 Python 类 |
| 2026-04-18 | v0.30 | 修正 CPU 设计：CPU 是 binding 文件（固化设计），不是 DTS 嵌套；arch/common/ 是零散模块，dts/bindings/riscv/ 是预置 CPU binding |
| 2026-05-16 | v0.31 | 新增 2.3 Binding 的三重职责：接口契约、命名隔离、校验边界。后续章节顺延 |
| 2026-05-16 | v0.32 | 重写 4.4 时钟域：clock-domain 改 DTS 节点，按工况定义（全速/均衡/低功耗/深度休眠），模块通过 phandle list 引用 |
| 2026-05-16 | v0.33 | 重写 2.5、5.1、5.2：两阶段明确为 compile_dts 实例化 + soc.connect 连接；full_dts 即模块发现机制；附录移除"模块发现机制" |
| 2026-05-16 | v0.34 | 重写 4.9：参数传递（dts_node 唯一入参）、校验链、兼容属性链（IP 演进向后兼容）；DTS 语法和 Binding 格式确认 |
| 2026-05-16 | v0.35 | 重写第 6、7、8 节：新增 Amaranth 作为底层引擎；框架只负责前端（DTS→Verilog）；仿真使用 amaranth.sim |
| 2026-05-16 | v0.36 | 修正 2.4：移除 CPU 内部连线例子，改为模块间 phandle 引用（<&模块 信号>）；2.6 验证流程从实例化阶段移到 connect() 阶段 |
| 2026-05-16 | v0.37 | 修正 4.6 IOMUX：引脚命名（PA0/PA1）、嵌套 GPIO 容器节点 |
| 2026-05-16 | v0.38 | 移除 ModuleBase 公共寄存器（CLOCK_SEL/CLOCK_DIV/POWER/RESET），时钟切换由 clock-domain 选域完成 |
| 2026-05-16 | v0.39 | 4.4 clock-domain 增加 `clock-gate`（门控暂停）和 `enable`（默认使能）属性；4.7 完整 SoC 示例加入 clock-domain 引用 |
| 2026-05-16 | v0.40 | 更新 12 节：Compatible 改为"一对多"；DTS 语法和 Binding 格式从"待确认"移至"已确认" |
| 2026-05-16 | v0.41 | 修正目录树：`dts/` 从 `devices/` 下移到顶层；`registry/` 缩进修正；移除 pinmap（引脚映射直接写在 IOMUX DTS 节点中） |
| 2026-05-16 | v0.42 | 附录 A 清理；4.5 compatible 写法统一为字符串 |
| 2026-05-16 | v0.43 | 4.1 新增 Boot 流程：`reset-vector` 在 CPU binding 设默认值，用户 DTS 可覆盖 |
| 2026-05-17 | v0.44 | 新增 2.0 DTS 的职责边界：明确 DTS 只负责实例化对象和传参，不描述 RTL 行为 |
| 2026-05-17 | v0.45 | 4.4 补充 CDC 实现说明（amaranth.lib.cdc）；2.4 补充 connect() 底层机制（Amaranth Signature 类型检查） |
| 2026-05-17 | v0.46 | 重写 4.2 总线抽象：总线对象自行解析 DTS 子节点并生成地址解码器；补充仲裁和多主说明 |
| 2026-05-17 | v0.47 | 4.2 补充数据宽度与对齐：CPU↔总线↔模块接口由 Amaranth Signature 统一约束，宽度不匹配在 connect 阶段报错 |
| 2026-05-17 | v0.48 | 4.2 补充：多主死锁规避由总线协议自身处理，框架层不介入 |
| 2026-05-17 | v0.49 | 4.5 补充核间中断 (IPI)：IPI 是独立模块，有自己的 DTS 节点，使用 phandle 引用 CPU |
| 2026-05-17 | v0.50 | 4.2 补充外设访问权限：所有 CPU 可见，总线层写互斥读共享 |
| 2026-05-17 | v0.51 | 2.3 新增校验分层：Binding 层做单参数校验，跨参数依赖由 Python __init__ 处理 |
| 2026-05-17 | v0.52 | 更新目录结构、用户 workflow：板子脚本即构建入口，DTS 单独存文件，scripts/build.py 移除 |
