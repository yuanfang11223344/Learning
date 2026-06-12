# Verilog 中 Inout 双向总线与三态门

> 来源：Gemini 对话，2026-06-05

---

## 一、Inout 是什么

### Q: inout 双向总线是什么？

**A:** `inout` 是双向端口——同一根物理连线既可以输出数据也可以输入数据。

```verilog
module bidir_iface (
    inout  wire [7:0] data_bus,   // 双向总线
    input  wire        oe,        // 输出使能（高有效）
    input  wire [7:0]  data_out,
    output wire [7:0]  data_in
);
    assign data_bus = oe ? data_out : 8'bz;  // 输出：oe=1 时驱动；否则高阻
    assign data_in  = data_bus;              // 输入：始终可读总线状态
endmodule
```

**工作模式：**

| oe | data_bus 状态 | data_in |
|----|-------------|---------|
| 1 | 驱动 data_out 的值 | 读到 data_out |
| 0 | 高阻 Z（不驱动） | 读到外部驱动的值 |

---

## 二、三态门（TBUF）内部结构

```
         +-----------+
data_out |           |
-------->|  TBUF     |-------> data_bus (inout)
   oe    | (三态门)  |
-------->|           |
         +-----------+
              ^
              |
         data_in（始终可读总线）
```

TBUF 在标准单元库中对应 `TBUFX1`、`TBUFX2` 等。内部由传输门 + 使能控制逻辑组成。oe=1 时传输门导通，oe=0 时传输门高阻。

---

## 三、Wire / Reg / Inout 的区别

### Q: 我平时的 wire/reg 是什么类型？

**A:** 都是**单向**信号。三者对比如下：

| 类型 | 信号方向 | 典型连接 | 综合后硬件 | 可多驱 |
|------|---------|---------|-----------|--------|
| `wire` | 单向 | 组合逻辑输出、模块互联 | 物理连线 | 否（需 MUX 合并） |
| `reg` | 单向 | always 块内被赋值 | DFF 或组合逻辑 | 否 |
| `inout` | 双向 | 芯片引脚、片外总线 | 三态 IO Pad | 是（分时共享） |

---

## 四、什么时候用 Inout

| 场景 | 是否用 inout |
|------|------------|
| I²C SDA 线 | 是（开漏 + 上拉） |
| SPI MISO/MOSI | 否（单向各一根） |
| DDR DQ 数据线 | 是（读/写分时） |
| 芯片内部模块互联 | 否（用 MUX） |
| 普通 GPIO | 是（可配置输入/输出） |

> 现代 ASIC 内部尽量避免三态总线——时序不可控、功耗分析复杂。片外用 inout 是物理需求。

---

## 五、开漏（Open-Drain）与三态的区别

两者常被混淆：

| | 三态（Tri-state） | 开漏（Open-Drain） |
|------|------|------|
| 高电平 | TBUF 主动驱动 | 外部上拉电阻拉高 |
| 低电平 | TBUF 主动驱动 | NMOS 下拉 |
| 典型协议 | 系统总线 | I²C、SMBus |
| 功耗 | 低 | 上拉电阻静态功耗 |

I²C 的 SDA 是开漏 + 外部上拉，不是三态——这是面试常见考点。

---

## 六、总结与举一反三

### 核心要点

1. **Inout = TBUF 实现的双向端口**，oe=1 驱动、oe=0 高阻。
2. **Wire 和 Reg 都是单向**，Inout 是唯一的双向端口类型。
3. **ASIC 内部避免 inout**，仅在 IO Pad 层使用。

### 举一反三

- **IO Pad 库单元**：`PADIN`（纯输入）、`PADOUT`（纯输出）、`PADBID`（双向，内含 TBUF + 输入 buffer）、`PADVDD/PADVSS`（电源地）。
- **多驱线与 Z 态**：仿真中如果两个 TBUF 同时 oe=1 且驱动不同的值 → 总线冲突（X 态）。硬件上可能导致大电流烧毁。
- **Pull-Up / Pull-Down**：内部弱上拉/下拉可以防止输入悬空时的 X 态，stdcell 库中对应 `PULLUP` / `PULLDOWN` 单元。
