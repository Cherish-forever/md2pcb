# md2pcb

> 用 Markdown 描述硬件设计。等 AI 成熟的那一天，直接把 MD 喂给它——出原理图、画 PCB。
>
> *Describe hardware in Markdown. When AI is ready, feed it the specs — and get schematics and PCB layouts back.*

---

## 核心理念

**不是等 AI 会画板子之后才开始，而是先把需求工程做扎实。**

结构化 MD 是未来 AI 可直接执行的 spec。文档越严谨，AI 生成的硬件越靠谱。

现在做的事情：
- 用 MD 把硬件设计规格写清楚
- 持续跟 AI 迭代完善这些文档
- 建立可复用的硬件设计模板

未来的事情：
- AI 从 MD 直接生成原理图
- AI 自动布局布线出 PCB
- "写文档即画板子"

---

## 仓库结构

```
md2pcb/
├── boards/              ← PCB 板级设计规格书
│   └── CH347_debug_board/
│       └── CH347_design.md       CH347F 多功能调试小板
│
├── chips/               ← 芯片/框架级设计文档
│   └── chipframework/
│       └── chipframework-design.md   DTS → Amaranth → Verilog 芯片前端框架
│
└── templates/           ← 硬件设计 MD 模板（规划中）
```

---

## 当前项目

| 项目 | 类型 | 说明 |
|------|------|------|
| [CH347F 调试小板](boards/CH347_debug_board/CH347_design.md) | PCB 设计 | USB 高速转接调试板，JTAG/SWD + SPI + I2C + 双串口 |
| [Chip Framework](chips/chipframework/chipframework-design.md) | 芯片框架 | DTS 描述硬件 → 自动生成 Verilog，让芯片前端设计成本降到 0 |

---

## 设计文档规范（WIP）

一份好的硬件设计 MD 应该包含：

- [x] 项目概述与目标用途
- [x] 芯片选型理由
- [x] 引脚定义与复用关系
- [x] 电路原理说明（可附带 ASCII 框图）
- [x] BOM 清单
- [x] 板级布局规划
- [x] 待确认事项清单
- [x] 参考资料链接

---

## License

Apache 2.0
