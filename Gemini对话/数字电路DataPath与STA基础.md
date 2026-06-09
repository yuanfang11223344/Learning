# 数字电路 Data Path 与 STA 基础

> 来源：Gemini 对话，2026-06-04 ~ 2026-06-09

---

## 一、什么是 Data Path

### Q: 在数字电路中什么是数据路径（data_path）？

**A:** Data Path 是数据从**源寄存器（Launch FF）**到**目的寄存器（Capture FF）**之间经过的全部组合逻辑链。

```
        Launch FF       组合逻辑链        Capture FF
       +--------+   (AOI→NAND→XOR→MUX)   +--------+
D_in ->|  DFF   |=======================>|  DFF   |--> D_out
CLK -->|        |                        |        |
       +--------+   ←── Data Path ──→   +--------+
```

在 STA 中，Setup 时序检查的公式：

```
Data Path Delay = T_clk2q(Launch) + Σ(T_cell + T_net) + T_setup(Capture)
```

必须满足：

```
Data Path Delay < Clock Period - Clock Skew
```

---

## 二、STA 核心概念

| 概念 | 含义 |
|------|------|
| **Launch Edge** | 发送时钟沿，数据由此开始传播 |
| **Capture Edge** | 捕获时钟沿，数据必须在此之前稳定 |
| **Launch Path** | Clock → Launch FF 的 clk→q + 所有组合逻辑延迟 |
| **Capture Path** | Capture FF 的 setup time |
| **Setup Check** | Data Path 总延迟 < 时钟周期 − Setup |
| **Hold Check** | Data Path 最小延迟 > Capture FF 的 Hold |
| **Slack** | 时序余量：正 = 满足，负 = 违例 |
| **Critical Path** | Slack 最小（最接近违例）的路径 |

---

## 三、逻辑级数与取负操作

### Q: A 取负变成 -A，有逻辑级数么？

**A:** 有延迟。

```
-A = ~A + 1
```

先按位取反（~A），再加 1。这需要：
- 一级 INV（取反）
- N 位加法器（+1 进位链）

N 位进位链的逻辑级数约为 log₂(N) 到 N，取决于综合器选择的加法器架构（行波进位 vs 超前进位）。**取负不是零延迟操作。**

---

## 四、嵌套 `?:` 与 MUX 的延时对比

### Q: `assign y = (c1) ? v1 : (c2) ? v2 : v3` 是两个选择器嵌套？

**A:** 是的。嵌套 `?:` 综合为**级联 MUX**（两级串联）：

```
         +--------+
cond1 -->| SEL    |
val1  -->|  0     |     +--------+
         |  MUX1  |---->| SEL    |
cond2 -->|  1     |     |  0     |
val2  -->|        |     |  MUX2  |---> Y
         +--------+     |  1     |
val3 ------------------>|        |
                        +--------+
```

### Q: 它的延时和我直接写一个 4 选 1 MUX 的延时相比？

**A:** 级联 MUX 是**两级门延迟**。而 `case` 写的 4 选 1 综合器可能直接映射到标准单元库的 `MUX41X1`——传输门实现，**一级门延迟**。

> 宽选择器用 `case` 而非嵌套 `?:`，给综合器更多优化空间。

### Q: `assign selected_path = (A && B) ? (C ? 2'd2 : 2'd1) : (D ? 2'd0 : 2'd3)` 是几级逻辑？

**A:** 至少三级：
1. `A && B`：一级 AND
2. 内层 `C ? 2'd2 : 2'd1`：一级 MUX
3. 外层 MUX：再一级

如果 `A && B` 的结果同时驱动两个 MUX 的 select，还有扇出导致的 buffer 插入。

---

## 五、总结与举一反三

### 核心要点

1. **Data Path = FF → 组合逻辑 → FF**——STA 分析的基本单元。
2. **Setup：路径慢了不行；Hold：路径快了也不行**——两者矛盾，多 Corner 联合检查。
3. **`-A = ~A + 1` 有加法器延迟**——不要误以为是零开销。
4. **嵌套 `?:` = 级联 MUX**——宽选择用 `case`，窄选择用 `?:`。

### 举一反三

- **Clock Skew**：Launch FF 和 Capture FF 的时钟到达时间差。正 skew 帮助 setup（Capture 时钟晚到），负 skew 帮助 hold（Capture 时钟早到）。但不可控的 skew 是时序收敛的噩梦。
- **OCV（On-Chip Variation）**：同一芯片上不同位置的 cell 延迟存在差异（工艺梯度、温度梯度、电压降）。STA 通过 derating 因子（如 ±10%）建模这种不确定性。
- **CRPR（Clock Reconvergence Pessimism Removal）**：当 Launch 和 Capture 共享同一段时钟路径时，OCV 分析会对同一段路径同时施加正负 derating——这过于悲观。CRPR 修正消除这种"双重惩罚"。
- **从 Data Path 到 PPA 优化**：缩短 Data Path 的手段包括换 LVT（减 cell delay）、插 buffer 分段（减 net delay）、逻辑重构（减级数）。
