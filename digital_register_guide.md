# 数字电路寄存器结构全解

## 1. 寄存器是什么

**寄存器（Register）= N 个 D 触发器（DFF）+ 共享时钟**

一个 DFF 存 1 bit，N 个 DFF 并排就能存 N bit。所以：

```
1 位寄存器 = 1 个 DFF
8 位寄存器 = 8 个 DFF
32 位寄存器 = 32 个 DFF
```

在 RTL 里写一行：

```verilog
reg [7:0] data;
```

综合后就是 8 个 DFF 排成一排，共享同一根 CLK。

---

## 2. 最简 DFF：D、CLK、Q

最基本的 D 触发器只有三个端口：

```
       +-------+
D ---->|  DFF  |----> Q
CLK -->|       |
       +-------+
```

行为表：

| CLK | D | Q(next) |
|-----|---|--------|
| ↑ | 0 | 0 |
| ↑ | 1 | 1 |
| 0/1 | X | Q 保持 |

对应 RTL：

```verilog
always @(posedge clk)
    q <= d;
```

关键特征：只在**时钟边沿**采样 D，其他时间 Q 保持不变。这称为**边沿触发（Edge Triggered）**。

---

## 3. 带使能的 DFF：加入 CE

最简单的 DFF 每个时钟沿都更新。实际电路经常需要**有条件地更新**——这就是 CE（Clock Enable）。

```verilog
always @(posedge clk)
    if (en)
        q <= d;
```

对应电路：

```
          +------+
D ------->| 1    |
Q ------->| 0    |--> DFF --> Q
          | MUX  |
          +------+
             ^
             |
            EN
```

| EN | CLK↑ | Q |
|----|------|----|
| 0 | 来了 | 保持原值 |
| 1 | 来了 | 装载 D |

内部原理就是 **MUX + DFF**：
- EN=1 → MUX 选 D，时钟沿采样新数据
- EN=0 → MUX 选 Q，时钟沿采样自己的旧值（保持）

在标准单元库中这就是 `DFFEX1`（SAED 命名）或 `FDRE` 的 CE 端口（Xilinx 原语）。

---

## 4. CE 的准确含义

CE 经常被称为"时钟使能"，但这个叫法容易误解。

**CE 不是控制时钟是否通过的。** 时钟 CLK 永远在跑，CE 只决定在该时钟边沿**是否装载新数据**：

```
时钟来了
     ↓
  检查 CE
     ↓
CE=1 → 装载 D
CE=0 → 保持 Q
```

所以更准确的叫法是 **寄存器使能（Register Enable）**。

---

## 5. CE 与 Clock Gating 的区别

| | CE（Clock Enable） | Clock Gating |
|------|------|------|
| 时钟是否进入触发器 | 是 | 否（被 AND 门阻断） |
| 实现方式 | MUX + DFF | Latch + AND 门 |
| 功耗 | 时钟树仍在翻转 | 时钟树部分停止，更省电 |
| FPGA 推荐 | 是 | 否（时钟树不可控、易产生毛刺） |
| ASIC 推荐 | 可用 | 常用（通过 ICG 单元安全实现） |
| 标准单元 | `DFFEX1` | `CLKGATETSTX1` (ICG) |

**FPGA 为什么不用 Clock Gating？**

```verilog
// 不推荐
wire gclk = clk & en;
always @(posedge gclk)   // 综合报警：Gated Clock
```

因为 `clk & en` 这个组合逻辑会产生毛刺（glitch），EN 的跳变如果正好在 CLK 高电平时，gclk 上会出现一个极窄的脉冲，导致错误触发。

FPGA 的解决方案就是用 DFF 自带的 CE 引脚，综合器直接映射到硬件上的 CE 端口，零额外逻辑。

---

## 6. 带复位 + 使能的完整寄存器

加上复位（RST），就是实际工程中最常见的寄存器形态：

```verilog
always @(posedge clk)
    if (rst)
        q <= 1'b0;
    else if (en)
        q <= d;
```

对应内部结构（两级 MUX）：

```
                +------+
Q ------------->| 0    |
D ------------->| 1    |  MUX1 (CE选择)
                +------+
                    |
                    v
                +------+
MUX1 ---------->| 0    |
0 ------------->| 1    |  MUX2 (RST选择)
                +------+
                    |
                    v
                +------+
CLK ---------->| DFF  |----> Q
                +------+
```

优先级：

```
RST > CE > D

RST=1        → Q = 0
RST=0, CE=1  → Q = D
RST=0, CE=0  → Q 保持
```

在标准单元库中，这对应 `DFFERX1`（SAED）。在 Xilinx FPGA 中就是 **FDRE** 原语。

---

## 7. Xilinx FDRE 原语

Xilinx FPGA 中一个真实的触发器资源：

```
            FDRE
    +------------------+
D ->| D            Q   |-->
CLK>| C                |
CE >| CE               |
RST>| R                |
    +------------------+
```

端口：

| 端口 | 含义 |
|------|------|
| D | 数据输入 |
| C | 时钟（上升沿触发，> 符号表示边沿触发） |
| CE | Clock Enable（寄存器使能） |
| R | 同步复位 |
| Q | 数据输出 |

综合器看到 `always @(posedge clk)` + `if(rst)` + `else if(en)` 就能直接映射成一个 FDRE，**零额外 MUX**——因为 FDRE 硬件上已经内置了 CE 和 RST 的路径。

---

## 8. DFF 选型速查

根据你的 RTL 写法，综合器自动选择对应的标准单元：

| RTL 特征 | SAED 标准单元 | Xilinx | 说明 |
|---------|-------------|--------|------|
| `@(posedge clk) q <= d` | `DFFX1` | `FD` | 最简 DFF |
| `@(posedge clk or negedge rst_n)` | `DFFRX1` | `FDR` | 异步复位 |
| `@(posedge clk) if(en) q <= d` | `DFFEX1` | `FDE` | 带使能 |
| `@(posedge clk) if(rst) ... else if(en)` | `DFFERX1` | `FDRE` | 同步复位 + 使能 |
| `@(posedge clk or negedge rst_n) if(!rst_n) ... else if(en)` | `DFFREX1` | `FDRSE` | 异步复位 + 使能 |

---

## 9. 为什么不写 else 也不会产生 Latch

这是一个经典误解。

```verilog
always @(posedge clk)
    if (rst)
        q <= 1'b0;
    else if (ce)
        q <= d;
// 没有 else！
```

**不会产生 Latch。** 原因：

- `always @(posedge clk)` → 这是**时序逻辑**，综合器只会推导 DFF
- 当 `rst=0, ce=0` 时，没有赋值语句被执行 → 综合器理解为 `q <= q`（保持）
- DFF 天然具备"保持上一个值"的能力，不需要额外逻辑

**什么时候会产生 Latch？** 只有组合逻辑：

```verilog
always @(*)       // 组合逻辑
    if (en)
        q = d;    // en=0 时 q 没有被赋值 → 综合器推导 Latch
```

口诀：

```
posedge/negedge → 永远推导 DFF
always @(*)     → 缺失赋值路径 → Latch
```

---

## 10. Latch 详解

### 10.1 Latch 是什么

**Latch（锁存器）= 电平敏感的存储器件。**

与 DFF 的关键区别：

```
DFF：  边沿触发（只在 CLK↑ 瞬间采样）
Latch：电平触发（EN=1 期间一直透明）
```

### 10.2 D Latch 的行为

```
       +---------+
D ---->| D Latch |----> Q
EN --->|         |
       +---------+
```

| EN | Q |
|----|-----|
| 1 | Q = D（透明，Q 实时跟随 D 变化） |
| 0 | Q 保持最后值 |

### 10.3 波形示意

```
EN : ____████████________
D  : 0__1__0__1__________
Q  : 0__1__0__1__________  ← EN=1 期间 Q 跟随 D 变化

EN : ____████████________
D  : 0__1__0__1__________
      ↑
    此时 EN↓，Q 锁住 0
```

### 10.4 形象比喻

|| DFF | Latch |
|------|------|------|
| 比喻 | 照相机（咔嚓一下） | 开着门的仓库 |
| 更新时机 | CLK↑ 瞬间 | EN=1 整个期间 |
| 防抖能力 | 强（只在边沿看一次） | 弱（EN=1 期间 D 毛刺全透传） |
| 时序分析 | 简单（边沿到边沿） | 复杂（电平窗口内组合环） |

### 10.5 为什么 FPGA 工程师看到 Latch 就紧张

1. **毛刺透传** — EN=1 时，D 上任何毛刺直接传递到 Q
2. **时序分析难** — Latch 的透明窗口可以形成组合逻辑环
3. **FPGA 硬件不天然支持** — FPGA 基本单元是 LUT+DFF，不是 Latch
4. **STA 工具处理复杂** — 需要额外约束

综合报告里出现 `WARNING: Latch inferred` 基本等于代码有 bug。

### 10.6 Latch 的正当用途

Latch 并非"坏器件"。在以下场景它很合理：

| 场景 | 说明 |
|------|------|
| **ICG（集成时钟门控）** | 负电平 Latch + AND，防止毛刺 |
| **Retention Register 的 Balloon Latch** | 保存断电前的状态 |
| **Time Borrowing** | 高性能设计中利用 Latch 透明期"借时间" |
| **总线保持** | 三态总线驱动器后的最后值保持 |

但在通用 FPGA/ASIC 的普通 RTL 逻辑中，**不要故意写 Latch**。

---

## 11. Latch 与 DFF 的完整对比

| 维度 | DFF | Latch |
|------|------|------|
| 触发方式 | 边沿（CLK↑ 或 CLK↓） | 电平（EN=1 或 EN=0） |
| RTL 敏感列表 | `@(posedge clk)` | `@(*)` 或 `@(d or en)` |
| 综合后器件 | 主从两级锁存器级联 | 单级锁存器 |
| 面积 | 大（约 2× Latch） | 小 |
| 输入毛刺免疫 | 强（只在边沿采样） | 弱（透明期间全透传） |
| STA 分析 | 标准 setup/hold | 需要额外时间借用约束 |
| FPGA 友好度 | 原生支持 | 不友好 |
| ASIC 低功耗设计 | 常用 | 仅特定场景（ICG/Balloon） |

---

## 12. FPGA 中的一个 Slice 长什么样

以 Xilinx 为例，一个 Slice 包含：

```
    +--------------------------------+
    |          SLICE                 |
    |                                |
    |  LUT4  -->   CARRY  -->  DFF   |
    |  (4输入)     (进位链)   (FDRE) |
    |                                |
    |  或配置为：                     |
    |  LUT5  -->   DFF               |
    |  LUT6  -->   DFF               |
    |  LUT + 分布式 RAM               |
    +--------------------------------+
```

所以 FPGA 里写 `reg [31:0] cnt`，综合器就在 32 个 Slice 中各用一个 DFF 来实现。

---

## 13. 关键工程经验

1. **RTL 写 `posedge clk`，综合就是 DFF，不是 Latch。** 这是最基础的认知，但很多新手会混淆。
2. **时序逻辑不需要 else。** `if(rst) ... else if(en)` 漏掉最后的 else 完全没问题。
3. **组合逻辑必须覆盖所有路径。** 反之，`always @(*)` 里漏掉赋值分支 → Latch。
4. **FPGA 不用 Clock Gating，用 CE。** 时钟门控交给 ICG 单元（ASIC），FPGA 用触发器自带的 CE 引脚。
5. **FDRE 是工程中最常见的寄存器形态。** DFF + CE + RST 三合一，占 FPGA 寄存器 90% 以上。
