# Synopsys 常见 ASIC 标准单元库详解（完整版）

> **重要前提**：严格来说，Synopsys 本身并不制造或提供物理标准单元库（Standard Cell Library）。标准单元库由 Foundry（TSMC、SMIC、UMC 等）或第三方 IP 厂商提供。Synopsys 主要提供 EDA 工具（Design Compiler、PrimeTime、ICC2、Formality）。
>
> 教学和实验环境中常用的 Synopsys 相关库：
> - **SAED 90nm / 32nm / 14nm**（Synopsys Armenia Educational Design Kit）
> - **GTECH Library**（Generic Technology，DC 综合前的中间映射）
> - **DesignWare Foundation**（`dw_foundation.sldb`，高层可综合 IP，非物理 stdcell）

以下以 **SAED32nm** 为主，结合工业标准单元库惯例，完整介绍 ASIC 标准单元分类。

---

## 0. 命名规则详解

### 工业标准单元命名格式

以 Synopsys SAED 库为例，一般格式为：

```
<FUNCTION><PARAMETERS><DRIVE>

例：NAND3X4  = 3输入NAND + 驱动强度4
    DFFRX1   = DFF + 异步复位 + 驱动强度1
    SDFFRX2  = Scan DFF + 异步复位 + 驱动强度2
```

### 常见后缀含义

| 后缀 | 含义 |
|------|------|
| `X1, X2, X4, X8, X16, X32` | 驱动强度（相对倍数），数字越大驱动能力越强 |
| `R` | 异步复位（Reset），低有效 |
| `S` | 异步置位（Set），低有效 |
| `SR` | 同时具有 Set 和 Reset |
| `E` | 带 Enable 端口 |
| `N` | 反相输出（Negated）或不带复位 |
| `T` | 带 Test（Scan）端口 |
| `Q` | 带 Q 反相输出 |
| `D` | 双输出 |

### 驱动强度 X 值的含义

`X1` 表示 1 倍驱动，`X2` 表示 2 倍驱动，以此类推。驱动强度决定了：
- **输出电阻**：X 越大，输出电阻越低，能驱动的负载越大
- **输入电容**：X 越大，输入晶体管尺寸越大，前级需要更强的驱动
- **传播延迟**：在相同负载下，X 大的单元更快
- **面积**：X 越大，单元面积越大
- **漏电**：X 越大，晶体管面积越大，漏电越高

### 驱动强度趋势一览

| 项目 | X1 → X32 趋势 |
|------|--------------|
| 驱动能力 | ↑↑ |
| 传播延迟 | ↓ |
| 面积 | ↑ |
| 动态功耗 | ↑ |
| 漏电功耗 | ↑ |
| 输入电容 | ↑ |

### 选型原则

- **轻负载**：用 X1~X2
- **中等扇出**：用 X4~X8
- **高扇出 / 长线**：用 X16~X32
- **时钟树**：优先用 CLKBUF，通常 X8~X32

### 不同 Foundry 的命名差异

同一类单元，不同厂商命名习惯不同：

| 功能 | SAED / 部分库 | 其他常见命名 |
|------|-------------|------------|
| 反相器 | INVX1 | INVD1, INV_X1, CKINV |
| 缓冲器 | BUFX1 | BUFFD1, BUF_X1 |
| 2选1 MUX | MUX21X1 | MUX2X1, MX2X1, MUX2 |
| 4选1 MUX | MUX41X1 | MUX4X1, MX4X1, MUX4 |
| 时钟门控 | CLKGATETSTX1 | ICG, CG, CKGATE, CGC |
| 锁存器 | LATCHX1 | LAT, LATN, LHQ |
| 电平转换 | LSUPX1/LSDNX1 | LS_LH, LS_HL, LVL_SHIFTER |
| 保持寄存器 | RETFFX1 | RET_DFF, RDFF |
| 低漏电单元 | (XL 后缀) | (LP 后缀), (LL 后缀) |

> 关键是理解功能而非死记命名。看 `.lib` 里的 `function` 字段才是准确的。

---

## 1. Combinational Cells（组合逻辑单元）

### 1.1 Inverter（反相器）

```
INVX1, INVX2, INVX4, INVX8, INVX16, INVX32
INVXL      (超低功耗版，驱动极小)
```

| A | Y |
|---|----|
| 0 | 1 |
| 1 | 0 |

RTL 映射：
```verilog
assign y = ~a;  // → INVX1
```

内部结构：最基本的一个 PMOS + 一个 NMOS。

工程用途：
- 逻辑求反
- 驱动反相（替代 BUF，面积更小）
- 奇数级反相器链构成延迟线
- 时钟树反转
- 振荡器环

---

### 1.2 Buffer（缓冲器）

```
BUFX1, BUFX2, BUFX4, BUFX8, BUFX16, BUFX32
BUFXL      (超低功耗版)
```

| A | Y |
|---|----|
| 0 | 0 |
| 1 | 1 |

内部结构：两级 INV 串联（INV + INV），因此面积约是 INV 的两倍。

工程用途：
- 提高扇出驱动能力
- 修复 setup/hold timing（增加延迟或增强驱动）
- 修复 max_transition / max_capacitance 违例
- CTS 中做时钟树平衡
- 隔离高扇出网络

> **工程经验**：如果能用 INV 就不要用 BUF。INV 面积更小、延迟更小。只有在必须保持逻辑极性时才用 BUF。

---

### 1.3 AND / OR

```
# AND
AND2X1, AND2X2, AND3X1, AND4X1

# OR
OR2X1, OR2X2, OR3X1, OR4X1
```

| A | B | AND | OR |
|---|---|-----|----|
| 0 | 0 | 0 | 0 |
| 0 | 1 | 0 | 1 |
| 1 | 0 | 0 | 1 |
| 1 | 1 | 1 | 1 |

内部结构：
- AND = NAND + INV
- OR = NOR + INV

> 因此 AND/OR 的面积和延迟都比对应的 NAND/NOR 大。在 CMOS 电路中，**NAND 和 NOR 是原生门**，AND/OR 是派生的。DC 综合时会尽量用 NAND/NOR 代替 AND/OR，除非逻辑必须。

---

### 1.4 NAND（与非门）

```
NAND2X1  NAND2X2  NAND2X4  NAND2X8
NAND3X1  NAND3X2  NAND3X4
NAND4X1  NAND4X2
NAND8X1  (8输入，少见)
```

| A | B | NAND2 |
|---|---|-------|
| 0 | 0 | 1 |
| 0 | 1 | 1 |
| 1 | 0 | 1 |
| 1 | 1 | 0 |

内部结构：2 个 PMOS 并联 + 2 个 NMOS 串联。

工程特点：
- **NAND 是最常用的组合逻辑门**（PMOS 并联电阻低，NMOS 串联效果可接受）
- NMOS 串联会降低下拉能力，输入端越多，延迟越大
- 一般 NAND4 是上限，超过 4 输入用两级实现更优

```verilog
assign y = ~(a & b & c);  // → NAND3X1
```

---

### 1.5 NOR（或非门）

```
NOR2X1  NOR2X2  NOR2X4
NOR3X1  NOR3X2
NOR4X1
```

| A | B | NOR2 |
|---|---|------|
| 0 | 0 | 1 |
| 0 | 1 | 0 |
| 1 | 0 | 0 |
| 1 | 1 | 0 |

内部结构：2 个 NMOS 并联 + 2 个 PMOS 串联。

工程特点：
- PMOS 串联导致上拉弱，同尺寸下 NOR 比 NAND 慢
- **业界倾向：优先用 NAND，尽量少用宽输入的 NOR**
- NOR3 以上就是"贵"门

```verilog
assign y = ~(a | b);  // → NOR2X1
```

---

### 1.6 XOR / XNOR（异或/同或）

```
XOR2X1  XOR2X2  XOR2X4
XNOR2X1  XNOR2X2  XNOR2X4
XOR3X1          (3输入异或，少见)
```

| A | B | XOR | XNOR |
|---|---|-----|------|
| 0 | 0 | 0 | 1 |
| 0 | 1 | 1 | 0 |
| 1 | 0 | 1 | 0 |
| 1 | 1 | 0 | 1 |

内部结构：较复杂，通常由传输门 + 或非/与非门组合实现，或直接用专用 XOR 版图。

工程用途：
- 加法器（半加器 = XOR + AND）
- CRC 校验
- ECC 编码/解码
- 数据加扰/解扰
- 奇偶校验
- 比较器（相等检测）

> XOR 是组合逻辑中最"贵"的基本门之一（面积大、延迟大）。设计时应尽量减少 XOR 级数。

---

### 1.7 AOI（AND-OR-INVERT，与或非）

这是 ASIC 中最常见的复合门，极其重要。

**命名规则：AOI + 各AND的输入个数 + 驱动**

| 型号 | AND 输入分解 | 示例端口 |
|------|-------------|---------|
| AOI21 | 1个2输入AND + 1个1输入AND | A, B, C |
| AOI22 | 2个2输入AND | A, B, C, D |
| AOI221 | 2输入AND + 2输入AND + 1输入AND | A, B, C, D, E |
| AOI222 | 3个2输入AND | A, B, C, D, E, F |
| AOI31 | 1个3输入AND + 1个1输入AND | A, B, C, D |
| AOI32 | 1个3输入AND + 1个2输入AND | A, B, C, D, E |
| AOI33 | 2个3输入AND | A, B, C, D, E, F |

> 所有 AOI 的信号路径：AND → OR → INV（最后一级总是一个 NOR）。

**各型号逻辑表达式（符号约定：`~`=取反，`&`=AND，`|`=OR）：**

| 型号 | 逻辑表达式 |
|------|-----------|
| AOI21 | Y = ~( (A&B) \| C ) |
| AOI22 | Y = ~( (A&B) \| (C&D) ) |
| AOI221 | Y = ~( (A&B) \| (C&D) \| E ) |
| AOI222 | Y = ~( (A&B) \| (C&D) \| (E&F) ) |
| AOI31 | Y = ~( (A&B&C) \| D ) |
| AOI32 | Y = ~( (A&B&C) \| (D&E) ) |

**AOI21 真值表：Y = ~((A&B) | C)**

| A | B | C | A&B | (A&B)\|C | Y |
|---|---|---|------|----------|---|
| 0 | 0 | 0 | 0   | 0        | 1 |
| 0 | 0 | 1 | 0   | 1        | 0 |
| 0 | 1 | 0 | 0   | 0        | 1 |
| 0 | 1 | 1 | 0   | 1        | 0 |
| 1 | 0 | 0 | 0   | 0        | 1 |
| 1 | 0 | 1 | 0   | 1        | 0 |
| 1 | 1 | 0 | 1   | 1        | 0 |
| 1 | 1 | 1 | 1   | 1        | 0 |

**AOI22 真值表：Y = ~((A&B) | (C&D))**

| A | B | C | D | A&B | C&D | (A&B)\|(C&D) | Y |
|---|---|---|---|------|------|-------------|----|
| 0 | 0 | 0 | 0 | 0   | 0   | 0           | 1 |
| 1 | 1 | 0 | 0 | 1   | 0   | 1           | 0 |
| 0 | 0 | 1 | 1 | 0   | 1   | 1           | 0 |
| 1 | 1 | 1 | 1 | 1   | 1   | 1           | 0 |

**内部结构（以 AOI21 为例）：**
- 第一步：A 和 B 经过 NMOS 串联（下拉），实现 A&B
- 第二步：AB 结果与 C 的 NMOS 并联，实现 (A&B)|C
- 第三步：PMOS 上拉网络做对偶结构
- 最终输出取反

**工程优势：**
- 一个 AOI 门等效于 AND + OR + INV 三级逻辑，但只有**一级门延迟**
- 面积比分离实现小很多（省掉中间级的驱动 buffer）
- DC 综合时会自动将合适的逻辑映射为 AOI

---

### 1.8 OAI（OR-AND-INVERT，或与非）

**命名规则：OAI + 各OR的输入个数 + 驱动**

| 型号 | OR 输入分解 | 示例端口 |
|------|-------------|---------|
| OAI21 | 1个2输入OR + 1个1输入OR | A, B, C |
| OAI22 | 2个2输入OR | A, B, C, D |
| OAI221 | 2输入OR + 2输入OR + 1输入OR | A, B, C, D, E |
| OAI222 | 3个2输入OR | A, B, C, D, E, F |
| OAI31 | 1个3输入OR + 1个1输入OR | A, B, C, D |
| OAI32 | 1个3输入OR + 1个2输入OR | A, B, C, D, E |
| OAI33 | 2个3输入OR | A, B, C, D, E, F |

> 所有 OAI 的信号路径：OR → AND → INV（最后一级总是一个 NAND）。

**各型号逻辑表达式（符号约定：`~`=取反，`&`=AND，`|`=OR）：**

| 型号 | 逻辑表达式 |
|------|-----------|
| OAI21 | Y = ~( (A\|B) & C ) |
| OAI22 | Y = ~( (A\|B) & (C\|D) ) |
| OAI221 | Y = ~( (A\|B) & (C\|D) & E ) |
| OAI31 | Y = ~( (A\|B\|C) & D ) |

**OAI21 真值表：Y = ~((A|B) & C)**

| A | B | C | A\|B | (A\|B)&C | Y |
|---|---|---|------|----------|---|
| 0 | 0 | 0 | 0    | 0        | 1 |
| 0 | 0 | 1 | 0    | 0        | 1 |
| 0 | 1 | 0 | 1    | 0        | 1 |
| 0 | 1 | 1 | 1    | 1        | 0 |
| 1 | 0 | 0 | 1    | 0        | 1 |
| 1 | 0 | 1 | 1    | 1        | 0 |
| 1 | 1 | 0 | 1    | 0        | 1 |
| 1 | 1 | 1 | 1    | 1        | 0 |

**内部结构：** 与 AOI 对称——先做 OR（NMOS 并联），再做 AND（NMOS 串联），最后 INV。

> AOI 和 OAI 共同覆盖了绝大多数复杂组合逻辑，DC 综合后通常 AOI/OAI 会占组合逻辑的 10%~20%。

---

### 1.9 Majority Gate（多数决门，部分库提供）

```
MAJ3X1   (3输入多数决：至少2个1则输出1)
```

| A | B | C | Y |
|---|---|---|---|
| 0 | 0 | 0 | 0 |
| 0 | 0 | 1 | 0 |
| 0 | 1 | 0 | 0 |
| 0 | 1 | 1 | 1 |
| 1 | 0 | 0 | 0 |
| 1 | 0 | 1 | 1 |
| 1 | 1 | 0 | 1 |
| 1 | 1 | 1 | 1 |

用途：ECC 纠错、容错电路。

---

### 1.10 组合逻辑门的输入数和驱动综合表

| 功能 | 输入数 | 常见驱动 |
|------|--------|----------|
| INV | 1 | X1~X32 |
| BUF | 1 | X1~X32 |
| NAND | 2,3,4 | X1~X8 |
| NOR | 2,3,4 | X1~X4 |
| AND | 2,3,4 | X1~X4 |
| OR | 2,3,4 | X1~X4 |
| XOR/XNOR | 2 | X1~X4 |
| AOI | 多种 | X1~X4 |
| OAI | 多种 | X1~X4 |

---

## 2. Multiplexer Cells（多路选择器）

### 2.1 基础 MUX

```
MUX21X1  MUX21X2  MUX21X4
MUX41X1  MUX41X2
MUX81X1           (8选1)
```

命名说明：`MUX21` = 2-1 MUX（2 个数据输入，1 个输出），`MUX41` = 4-1 MUX。

### 2.2 MUX 变体

```
MUX21NX1   (输出反相)
MUXI21X1   (Inverting MUX)
```

### 2.3 内部实现

MUX 通常用**传输门**实现，而不是直接用逻辑门——传输门 MUX 延迟更小、面积更省。

```verilog
assign y = sel ? a : b;         // → MUX21X1
assign y = sel[1] ? (sel[0] ? d : c) : (sel[0] ? b : a);  // → MUX41X1
```

### 2.4 工程注意事项

- MUX 输入到输出的延迟与输入位置有关（靠近输出的输入更快）
- 宽 MUX（如 MUX81）通常不如用多级 MUX21 级联效率高
- MUX 的 `sel` 信号有扇出要求，大 MUX 需要 buffered select

---

## 3. Sequential Cells（时序单元）

这是 ASIC 中**数量最多、种类最复杂**的一类。

### 3.1 基础 DFF（D Flip-Flop）

```
DFFX1  DFFX2  DFFX4
```

端口：`D, CLK, Q`

| CLK | D | Q(next) |
|-----|---|--------|
| ↑ | 0 | 0 |
| ↑ | 1 | 1 |
| 0/1 | X | Q (hold) |

```verilog
always @(posedge clk)
    q <= d;           // → DFFX1
```

内部结构：典型的主从（Master-Slave）结构，两个锁存器级联。

---

### 3.2 带异步复位的 DFF

```
DFFRX1  DFFRX2  DFFRX4
DFFRSX1          (带同步复位，部分库用不同命名)
```

端口：`D, CLK, RSTN, Q`

| RSTN | CLK | D | Q(next) |
|------|-----|---|--------|
| 0 | X | X | 0 |
| 1 | ↑ | 0 | 0 |
| 1 | ↑ | 1 | 1 |

```verilog
always @(posedge clk or negedge rst_n)
    if (!rst_n)
        q <= 1'b0;
    else
        q <= d;       // → DFFRX1
```

**异步复位 VS 同步复位：**

| | 异步复位 | 同步复位 |
|------|---------|---------|
| 复位速度 | 立即生效，不依赖时钟 | 需要等下一个时钟沿 |
| 复位释放 | 可能与时钟沿竞争（recovery/removal） | 无竞争 |
| DFT 友好性 | 需要额外处理复位树 | 好（复位是数据路径） |
| 面积 | 略大 | 略小（不需要复位 pin 的特殊 buffer tree） |
| 时序约束 | 需要 `set_recovery` / `set_removal` | 作为正常 setup/hold 检查 |

---

### 3.3 带异步置位的 DFF

```
DFFSX1  DFFSX2
```

端口：`D, CLK, SETN, Q`

| SETN | CLK | D | Q(next) |
|------|-----|---|--------|
| 0 | X | X | 1 |
| 1 | ↑ | 0 | 0 |
| 1 | ↑ | 1 | 1 |

---

### 3.4 带 Set + Reset 的 DFF

```
DFFSRX1  DFFSRX2
```

端口：`D, CLK, SETN, RSTN, Q`

两个异步控制，**通常不允许同时有效**（同时 SETN=0, RSTN=0 时 Q 行为不确定，取决于具体实现）。

---

### 3.5 带 Enable 的 DFF

```
DFFEX1   DFFEX2           (使能 + 无复位)
DFFERX1  DFFERX2          (使能 + 异步复位)
DFFESRX1                  (使能 + Set + Reset)
```

端口：`D, CLK, E, [RSTN/SETN], Q`

| E | CLK | D | Q(next) |
|---|-----|---|--------|
| 0 | ↑ | X | Q (hold) |
| 1 | ↑ | 0 | 0 |
| 1 | ↑ | 1 | 1 |

```verilog
always @(posedge clk or negedge rst_n)
    if (!rst_n)
        q <= 1'b0;
    else if (en)
        q <= d;                // → DFFERX1
```

内部实现：通常用 MUX 在 D 和 Q 之间选择，使能无效时 DFF 保持原值。

---

### 3.6 双沿触发 DFF（Dual-Edge DFF，部分库提供）

```
DEDFFX1
```

上升沿和下降沿都捕获数据。等效于两个 DFF 分别在正负沿采样后 MUX，但面积更小。

用途：
- 半速率时钟设计
- DDR 接口
- 降低时钟频率同时保持吞吐量

---

### 3.7 带 QN（反相输出）的 DFF

```
DFFQX1    DFF  + 同时输出 Q 和 QN
DFFRQX1   DFFR + Q + QN
```

用途：减少额外的 INV，节省一路反相器延迟。

---

### 3.8 Multi-Bit DFF（多位 DFF，先进工艺提供）

```
DFF2X1    (2-bit DFF, 共享时钟)
DFF4X1    (4-bit DFF)
```

将多个 DFF 封装在一起，共享时钟、复位 buffer。省面积、降功耗（共始终钟网络）。

---

## 4. Scan Cells（可测试性单元 - DFT）

### 4.1 为什么需要 Scan

DFT 插入后，芯片中 **90%+ 的寄存器会被替换为 Scan DFF**，因此 Scan DFF 是后端网表中数量最大的单类时序单元。

### 4.2 基本 SDFF

```
SDFFX1   SDFFX2   SDFFX4
```

端口：`D, SI, SE, CLK, Q`

| SE | 功能 |
|----|------|
| 0 | 正常模式：Q = D（功能路径） |
| 1 | Scan 模式：Q = SI（测试移位路径） |

内部结构：在 DFF 前端加一个 MUX（D vs SI），由 SE 选择。

---

### 4.3 SDFF 变体矩阵

```
# Scan + 异步复位
SDFFRX1   SDFFRX2   SDFFRX4

# Scan + 异步复位 + 使能
SDFFREX1

# Scan + Set + Reset
SDFFSRX1  SDFFSRX2

# Scan + 异步置位
SDFFSX1

# Scan + Enable（无复位）
SDFFEX1
```

### 4.4 Scan 单元选择经验

综合阶段用普通 DFF，DFT 插入阶段（通常用 Synopsys DFT Compiler 或 Tessent）会自动将 DFF 替换为对应的 SDFF。

DFF → SDFF 映射关系：

| 综合后 | DFT 插入后 |
|--------|-----------|
| DFFX1 | SDFFX1 |
| DFFRX1 | SDFFRX1 |
| DFFSX1 | SDFFSX1 |
| DFFERX1 | SDFFREX1 |

---

## 5. Latch Cells（锁存器 - 电平敏感）

### 5.1 基础 Latch

```
LATCHX1  LATCHX2  LATCHX4      (高电平透明，正使能)
LATCHNX1  LATCHNX2              (低电平透明，负使能)
```

> 不同 Foundry 的命名：部分库用 `LAT`（简化）, `LATN`（负使能）, `LHQ`（带 QN 输出），功能等价。

| EN | D | Q |
|----|---|----|
| 1 | 0 | 0 |
| 1 | 1 | 1 |
| 0 | X | Q (hold) |

```verilog
always @(*)
    if (en)
        q = d;          // → LATCHX1 (高有效使能)
```

内部结构：比 DFF 简单（只有一级，不需要主从结构）。

### 5.2 带复位的 Latch

```
LATCHRX1     (带复位)
LATCHSRX1    (带 Set + Reset)
```

### 5.3 Latch 的工程角色

| 用途 | 说明 |
|------|------|
| **Time Borrowing** | 在 pulsed-latch 设计中，latch 可以在半个周期内"借用"时间 |
| **Clock Gating** | 时钟门控内部的使能 latch（防止 glitch） |
| **Retention 寄存器** | 休眠时用 latch 保存状态 |
| **总线保持** | 当总线驱动器三态时保持最后值 |
| **电平同步** | 跨时钟域的低速控制信号 |

> **工程警告**：锁存器在标准 ASIC 流程中通常应**避免出现**（除非特意设计）。DC 综合时如果 RTL 写得不完整（if 没有 else，case 没有 default），会意外推断出 latch，STA 也难以分析。用 `compile_ultra -no_latch` 可以禁止。

---

## 6. Arithmetic Cells（算术运算单元）

### 6.1 Half Adder（半加器）

```
HAX1  HAX2
```

| A | B | SUM | COUT |
|---|---|-----|------|
| 0 | 0 | 0 | 0 |
| 0 | 1 | 1 | 0 |
| 1 | 0 | 1 | 0 |
| 1 | 1 | 0 | 1 |

等价于：SUM = A XOR B, COUT = A AND B

---

### 6.2 Full Adder（全加器）

```
FAX1  FAX2  FAX4
```

| A | B | CIN | SUM | COUT |
|---|---|-----|-----|------|
| 0 | 0 | 0 | 0 | 0 |
| 0 | 0 | 1 | 1 | 0 |
| 0 | 1 | 0 | 1 | 0 |
| 0 | 1 | 1 | 0 | 1 |
| 1 | 0 | 0 | 1 | 0 |
| 1 | 0 | 1 | 0 | 1 |
| 1 | 1 | 0 | 0 | 1 |
| 1 | 1 | 1 | 1 | 1 |

等价于：SUM = A XOR B XOR CIN, COUT = (A&B) | (B&CIN) | (CIN&A)

工程用途：
- 加法器、ALU
- 乘法器的部分积累加
- 计数器
- DSP 数据通路

> 直接实例化 FAX1 往往比让 DC 从 NAND+NOR 综合出加法器更优——面积更小，延迟更可控。

---

### 6.3 Half Subtractor / Full Subtractor（部分库提供）

```
HSX1       (Half Subtractor)
FSX1       (Full Subtractor)
```

### 6.4 Incrementer / Decrementer（部分库提供）

```
INC4X1     (4-bit 递增器)
DEC4X1     (4-bit 递减器)
```

---

## 7. Clock Cells（时钟树单元）

CTS（Clock Tree Synthesis）阶段大量使用。时钟单元的关键特性是**上升沿和下降沿延迟尽量对称**（balanced rise/fall delay），以减少时钟 duty cycle 失真。

### 7.1 Clock Buffer

```
CLKBUF1  CLKBUF2  CLKBUF4
CLKBUF8  CLKBUF16  CLKBUF32
```

与普通 BUF 的区别：
- Rise/Fall delay 更平衡（skew 更小）
- 对 PVT 变化不敏感
- 通常驱动能力更大（X8~X32 为主）
- 版图上电源/地更宽以应对大电流

---

### 7.2 Clock Inverter

```
CLKINVX1  CLKINVX2  CLKINVX4
CLKINVX8  CLKINVX16
```

作用：
- 产生互补时钟
- 在时钟树中作为反相节点（两个 CLKINV = 一个 CLKBUF，但面积更小）
- 调整时钟极性

---

### 7.3 Clock Gate（Integrated Clock Gating - ICG）

不同厂商命名各异，功能相同：

| SAED 风格 | 其他常见命名 | 等价含义 |
|-----------|------------|----------|
| `CLKGATETSTX1` | `ICG_X1`, `CG_X1`, `CKGATE_X1` | 带 TE 的集成时钟门控 |
| `CLKGATEENX1` | `ICGEN_X1` | 仅使能，无 TE 端口 |

```
CLKGATETSTX1  CLKGATETSTX2  CLKGATETSTX4
CLKGATEENX1                (仅使能型)
```

端口：`CLK, E, TE, GCLK`

| E | TE | GCLK |
|---|-----|------|
| 0 | 0 | 0 (gated, 关断) |
| 1 | 0 | CLK (through) |
| X | 1 | CLK (test mode, 强制透传) |

内部结构：一个负电平 latch + AND 门。

```verilog
// 综合时自动推断
always @(posedge clk)
    if (en)
        q <= d;
// compile_ultra -gate_clock → CLKGATETSTX1
```

时钟门控是**降低动态功耗最重要的手段**——将不需要翻转的寄存器时钟关断，节省时钟树和寄存器的动态功耗。

**TE 端口的作用**：DFT 模式下，TE=1 强制时钟透传，保证扫描链正常工作。

### 7.4 Clock MUX（有时钟专用）

```
CLKMUX21X1   (glitch-free 时钟切换)
```

与普通 MUX 不同，glitch-free 时钟 MUX 在两个时钟源之间切换时不会产生毛刺。

---

## 8. Low Power Cells（低功耗单元）

UPF（Unified Power Format）设计必备。

### 8.1 Isolation Cell（隔离单元）

```
ISO0X1   ISO0X2    (输出钳位到 0)
ISO1X1   ISO1X2    (输出钳位到 1)
ISOLX1             (钳位到上次值)
```

端口：`A, ISO_EN, Y`

| ISO_EN | Y |
|--------|----|
| 0 | A (正常) |
| 1 | 0 或 1 (隔离) |

作用：当某个 power domain 断电时，其输出可能浮空（floating），导致下游常电域接收到不确定值。Isolation cell 将不确定输出钳位到确定值。

实现：
- ISO0: Y = A AND NOT(ISO_EN)，用 AND 门实现
- ISO1: Y = A OR ISO_EN，用 OR 门实现

> 在 SAED 库中，ISO0 等效于 `AND2X1` + 一路使能反相（但版图通常有隔离结构）。

---

### 8.2 Level Shifter（电平转换单元）

```
LSUPX1             (Low → High:  0.8V → 1.2V)
LSDNX1             (High → Low:  1.2V → 0.8V)
LSBIDX1            (Bidirectional)

# 其他常见命名
LS_LH_X1           (Low → High, 等价于 LSUPX1)
LS_HL_X1           (High → Low, 等价于 LSDNX1)
LVL_SHIFTER_X1     (通用电平转换，部分库命名)
```

端口：`A, VDD1, VDD2, Y`

跨越不同电压域时，信号高电平不一致，需要电平转换。

内部结构：特殊设计的交叉耦合结构，确保信号正确翻转——否则可能出现"信号过去了但收不到"的情况。

---

### 8.3 Power Switch（电源开关单元）

```
HEADERX1     (Header: PMOS，断开 VDD)
FOOTERX1     (Footer: NMOS，断开 VSS)
```

这**不是标准单元**（属于 power switch cell），但在低功耗设计中必不可少。

---

### 8.4 Retention Flip-Flop（状态保持寄存器）

```
RETFFX1   RETFFX2
RETFFRX1               (带复位)
RETFFSX1               (带置位)

# 其他常见命名
RET_DFF_X1             (部分 Foundry 用此命名)
RDFFX1                 (简写版)
```

端口：`D, CLK, RSTN, SAVE, RESTORE, RET, Q`

工作模式：

| 模式 | 说明 |
|------|------|
| **正常** | 普通 DFF 行为 |
| **Save** | 关电前，主 latch 值存到 retention latch（balloon latch）中 |
| **Sleep** | 核心断电，retention latch 由常电域供电保持值 |
| **Restore** | 上电后，retention latch 的值恢复到主 latch |

Balloon Latch（气球锁存器）使用高 Vt / 厚氧化层器件，漏电极低，可以在常电域中低功耗保持状态。

---

### 8.5 Always-On Buffer / Inverter

```
AOBUFX1          (Always-On Buffer)
AOINVX1          (Always-On Inverter)
```

这些单元的供电来自常电域（VDD_Always_On），即使所在模块断电也能正常工作。用于电源控制逻辑、隔离使能信号等。

---

### 8.6 Enable Level Shifter（使能型电平转换）

```
LSENX1              (带使能控制)
```

在不需要时完全关闭，进一步降低漏电。

---

## 9. Physical Only Cells（物理专用单元）

这些单元没有逻辑功能，布局布线阶段由 P&R 工具自动插入。DC 综合阶段看不到它们。

### 9.1 Filler Cell（填充单元）

```
FILL1  FILL2  FILL4  FILL8  FILL16  FILL32  FILL64
FILL1CAP          (带 decap 的 filler)
```

作用：
- 填充标准单元行（row）中的空闲位置
- 保证 N-Well 和 P-Well 连续性
- 满足 DRC 密度要求
- 部分 filler 还内含去耦电容

数字表示 filler 宽度（以 site 或 grid 为单位）。

---

### 9.2 Welltap Cell（阱接触单元）

```
TAPCELL
TAPCELLB          (带偏置)
```

作用：
- 将 N-Well 连接到 VDD
- 将 P-Substrate 连接到 VSS
- 防止 latch-up（闩锁效应）
- 满足 DRC 最大阱接触间距要求

> 在先进工艺中，TAPCELL 的间距有严格 DRC 要求（如每隔 30μm 必须有一个）。

---

### 9.3 Endcap Cell（行末单元）

```
ENDCAP
ENDCAPL           (左侧)
ENDCAPR           (右侧)
ENDCAPLE          (左 + 外部)
ENDCAPRE          (右 + 外部)
```

作用：
- 放置在标准单元行的两端
- 封闭 N-Well / P-Well 边界
- 保证行末 DRC 规则
- 防止边缘效应（etching, lithography）

---

### 9.4 Decap Cell（去耦电容单元）

```
DCAPX1  DCAPX2  DCAP4  DCAPX8  DCAPX16  DCAPX32
```

内部：一个用 MOS 管做的电容（通常 NMOS gate 到 substrate）。

作用：
- 提供本地电荷存储，平滑电源波动
- 降低动态 IR Drop
- 抑制同时开关噪声（SSN）
- 提高电源完整性

> Decap 不是越多越好：过多的 decap 会增加漏电（栅氧化层漏电），且占用面积。

---

### 9.5 Antenna Diode（天线效应二极管）

```
ANTENNAX1
ANTENNAX2
```

作用：防止**天线效应**——在金属刻蚀过程中，长金属线作为"天线"收集电荷，可能击穿薄栅氧化层。Antenna diode 提供电荷泄放路径。

通常在 routing 后由工具自动插入。

---

### 9.6 Corner Cell（角落单元）

```
CORNER
CORNERLL   (左下)
CORNERLR   (右下)
CORNERUL   (左上)
CORNERUR   (右上)
```

围绕芯片四角放置，保证环形结构（如 guard ring）的连续性。

---

### 9.7 Boundary Cell（边界单元）

```
BOUNDARY
```

放置在 core 区域与 I/O 区域之间，形成完整的保护环。

---

### 9.8 ESD Clamp Cell（静电保护单元，部分库包含）

```
ESDCLAMP
```

提供从电源到地的 ESD 泄放路径。

---

## 10. Utility Cells（辅助逻辑单元）

### 10.1 Tie Cell（常量单元）

```
TIEHI   TIEHI_X1           (输出 VDD / 1'b1)
TIELO   TIELO_X1           (输出 VSS / 1'b0)
```

**为什么需要 Tie Cell？**

如果直接将 gate 接到 VDD/VSS：
- ESD 风险（栅极直接暴露到电源网络）
- 驱动问题（电源网络阻抗极低，如果发生意外短路可能烧毁）

Tie Cell 内部用一个小电阻或二极管隔离，保护栅极。

```verilog
assign tied_high = 1'b1;     // DC 综合 → TIEHI
assign tied_low  = 1'b0;     // DC 综合 → TIELO
```

---

### 10.2 Tri-State Buffer（三态缓冲器）

```
TBUFX1  TBUFX2  TBUFX4  TBUFX8
TBUFX16  TBUFX32
```

| EN | A | Y |
|----|---|----|
| 0 | X | Z (高阻) |
| 1 | 0 | 0 |
| 1 | 1 | 1 |

```verilog
assign y = en ? a : 1'bz;    // → TBUFX1
```

用途：
- 共享总线（多个驱动器通过三态共享一根线）
- 双向 I/O
- 存储器 bitline 驱动器

> 现代 ASIC 内部尽量避免三态总线（时序/功耗不好），多用 MUX 代替。三态主要用于 I/O pad 和存储器接口。

---

### 10.3 Pull-Up / Pull-Down

```
PULLUP           (弱上拉)
PULLDOWN         (弱下拉)
```

提供一个弱电阻连接到 VDD 或 VSS，防止输入悬空。

---

### 10.4 Delay Cell

```
DELAYX1  DELAYX2  DELAYX4
```

内部通常为多级反相器链或 RC 延迟链。

用途：
- Hold timing 修复（增加最小路径延迟）
- 脉冲展宽
- 死区时间生成

---

## 11. ECO Cells（工程变更单元）

### 11.1 Spare Cell（备用单元）

```
SPARE_AND2X1   SPARE_NAND2X1
SPARE_NOR2X1   SPARE_INVX1
SPARE_DFFRX1
SPARE_AOI21X1
```

这些单元**在初始布局时散布在芯片各处，但不连接到任何逻辑**。当发现 bug 需要 ECO 时，通过**仅修改金属层**把它们接进来，避免重新做全套 mask（成本极高）。

### 11.2 Metal-Only ECO Cell

```
ECO_AND2   ECO_NAND2
ECO_INV    ECO_BUF
ECO_DFF
```

这些单元底层（FEOL）预置了各种可能的晶体管组合，通过改金属层（BEOL）可以把它们配置成不同的逻辑功能。一个 ECO 单元可能配置为 NAND、NOR 或 INV，极大提高 ECO 灵活性。

---

## 12. DFT 辅助单元（可测试性设计）

### 12.1 Test Compression（部分库提供）

```
COMPRESSORX1       (XOR 压缩树单元)
MISRX1             (MISR 单元)
```

### 12.2 JTAG / Boundary Scan（边界扫描）

```
BSCANX1            (Boundary Scan Cell)
```

JTAG 链中每根 I/O pad 后都有一个 BSCAN 单元。

### 12.3 LBIST（逻辑内建自测试）

```
LFSRX1             (线性反馈移位寄存器单元)
PRPGX1             (伪随机模式发生器)
```

---

## 13. 特殊功能单元

### 13.1 Synchronizer Cell（同步器）

```
SYNC2X1            (2级同步器)
SYNC3X1            (3级同步器)
```

内部是两个 DFF 串联，并且版图上做了特殊处理（靠近摆放、金属屏蔽），降低亚稳态传播概率。

---

### 13.2 Pulse Latch（脉冲锁存器）

```
PLATCHX1
```

用短脉冲时钟代替电平使能，兼具 latch 的小面积和 flip-flop 的边沿特性，常用于高性能设计（如 Intel 的 pulsed-latch 风格）。

---

### 13.3 Time-Borrowing Latch

```
TBLATCHX1
```

用于 time-borrowing 设计中的特殊 latch。

---

### 13.4 Glitch-Free Clock MUX

```
GFCLKMUX21X1
```

内部带握手逻辑，保证时钟切换不产生毛刺。

---

## 14. 标准单元在 Liberty (.lib) 中的表现形式

每个标准单元在 Liberty 文件中都有一段完整的描述，理解这些字段是后端工程师的基本功。

```liberty
cell(INVX1) {
    area : 1.2;                          # 面积
    cell_leakage_power : 0.5;            # 漏电功耗 (nW)

    pin(A) {                             # 输入 pin
        direction : input;
        capacitance : 0.002;             # 输入电容 (pF)
        max_transition : 0.5;            # 允许的最大输入 transition
    }

    pin(Y) {                             # 输出 pin
        direction : output;
        function : "(!A)";               # 逻辑功能
        max_capacitance : 0.1;           # 最大负载电容 (pF)

        timing() {                       # 时序弧
            related_pin : "A";
            timing_sense : negative_unate;
            cell_rise(delay_template_7x7) {
                index_1("0.01, 0.05, 0.1, 0.3, 0.7, 1.2, 2.0");
                index_2("0.001, 0.005, 0.01, 0.03, 0.05, 0.1, 0.2");
                values("0.020, 0.025, ...");
            }
            cell_fall(delay_template_7x7) { ... }
            rise_transition(delay_template_7x7) { ... }
            fall_transition(delay_template_7x7) { ... }
        }
    }
}
```

关键字段：

| 字段 | 含义 |
|------|------|
| `area` | 单元面积（标准单元 site 面积单位） |
| `cell_leakage_power` | 静态漏电功耗 |
| `capacitance` | 输入 pin 电容，决定前级负载 |
| `max_capacitance` | 输出 pin 能驱动的最大电容 |
| `max_transition` | 输入 pin 允许的最大信号跳变时间 |
| `timing_sense` | `positive_unate` / `negative_unate` / `non_unate` |
| `function` | 逻辑函数表达式 |
| NLDM / CCS | 时序模型类型 |

---

## 15. 单元高度与 Track

标准单元库按 track 高度分类：

| Track | 典型用途 |
|-------|---------|
| 7-track | 高密度（HD），面积最小但速度最慢 |
| 9-track | 通用（GP），面积和速度平衡 |
| 12-track | 高速（HS），面积大但速度快 |

单元高度 = Track 数 × 金属间距。

---

## 16. 综合后的典型网表结构

一个 SAED32nm 综合后的 netlist 片段：

```verilog
module top (...);
    // 组合逻辑
    INVX1  u1 (.A(n1), .Y(n2));
    NAND2X1 u2 (.A(n2), .B(n3), .Y(n4));
    AOI21X1 u3 (.A(n4), .B(n5), .C(n6), .Y(n7));
    MUX21X1 u4 (.A(n8), .B(n9), .S(sel), .Y(n10));

    // 时序逻辑
    DFFRX1 u_reg_0 (.D(n10), .CLK(clk), .RSTN(rst_n), .Q(q[0]));
    SDFFRX1 u_reg_1 (.D(q[0]), .SI(scan_in), .SE(se), .CLK(clk), .RSTN(rst_n), .Q(scan_out));

    // 时钟
    CLKGATETSTX1 u_clk_gate (.CLK(clk), .E(en), .TE(te), .GCLK(gclk));
    CLKBUF8 u_clkbuf (.A(gclk), .Y(clk_buf));

    // 常量
    TIEHI u_tie_hi (.Y(VDD_tie));
    TIELO u_tie_lo (.Y(VSS_tie));
endmodule
```

---

## 17. 数量占比总结

### 综合后（pre-P&R）

| 类型 | 占比 |
|------|------|
| INV / BUF | 35% ~ 45% |
| NAND / NOR | 20% ~ 30% |
| AOI / OAI | 10% ~ 20% |
| DFF / SDFF | 10% ~ 20% |
| MUX / XOR / FA | 5% ~ 10% |
| Clock Cell | 3% ~ 10% |
| Tie / Other | 1% ~ 3% |

### P&R 后（含物理单元）

| 类型 | 占比 |
|------|------|
| 逻辑单元（以上全部） | 60% ~ 75% |
| FILL | 15% ~ 30% |
| TAPCELL / ENDCAP | 3% ~ 5% |
| DECAP | 5% ~ 10% |

---

## 18. 一个典型 STDCELL 库的完整目录树

```
Standard Cell Library
│
├── Logic Cells（组合逻辑）
│   ├── INV       反相器 (X1~X32, XL)
│   ├── BUF       缓冲器 (X1~X32, XL)
│   ├── NAND      与非门 (2/3/4/8 输入, X1~X8)
│   ├── NOR       或非门 (2/3/4 输入, X1~X4)
│   ├── AND       与门 (2/3/4 输入)
│   ├── OR        或门 (2/3/4 输入)
│   ├── XOR       异或门 (2/3 输入)
│   ├── XNOR      同或门 (2 输入)
│   ├── AOI       与或非 (AOI21/AOI22/AOI221/AOI222/AOI31/AOI32)
│   ├── OAI       或与非 (OAI21/OAI22/OAI221/OAI222/OAI31/OAI32)
│   ├── MAJ       多数决门 (MAJ3)
│   └── MUX       多路选择器 (MUX21/MUX41/MUX81)
│
├── Sequential Cells（时序逻辑）
│   ├── DFF       基础 D 触发器
│   ├── DFFR      带异步复位
│   ├── DFFS      带异步置位
│   ├── DFFSR     带 Set + Reset
│   ├── DFFE      带使能
│   ├── DFFER     带使能 + 异步复位
│   ├── DFFQX     带 QN 反相输出
│   ├── DEDFF     双沿触发（部分库）
│   ├── MBFF      多位触发器 (DFF2/DFF4)
│   ├── SDFF      扫描触发器（DFT）
│   ├── SDFFR     扫描 + 复位
│   ├── LATCH     电平锁存器
│   └── SYNC      同步器 (SYNC2/SYNC3)
│
├── Clock Cells（时钟树）
│   ├── CLKBUF    时钟缓冲器
│   ├── CLKINV    时钟反相器
│   ├── CLKGATE   集成时钟门控 (ICG)
│   └── CLKMUX    无毛刺时钟切换
│
├── Arithmetic Cells（算术运算）
│   ├── HA        半加器
│   ├── FA        全加器
│   ├── HS/FS     半减器/全减器（部分库）
│   └── INC/DEC   递增/递减器（部分库）
│
├── Low Power Cells（低功耗）
│   ├── ISO       隔离单元 (ISO0/ISO1/ISOL)
│   ├── LS        电平转换 (LS_LH/LS_HL/LS_BID)
│   ├── RETFF     保持寄存器
│   ├── AOBUF     常电缓冲器
│   ├── AOINV     常电反相器
│   ├── HEADER    电源开关 - Header
│   └── FOOTER    电源开关 - Footer
│
├── Physical Only Cells（物理专用）
│   ├── FILL      填充单元 (FILL1~FILL64)
│   ├── TAPCELL   阱接触
│   ├── ENDCAP    行末单元
│   ├── DECAP     去耦电容 (DCAP4~DCAP32)
│   ├── ANTENNA   天线二极管
│   ├── CORNER    四角单元
│   └── BOUNDARY  边界单元
│
├── Utility Cells（辅助逻辑）
│   ├── TIEHI     常量 1
│   ├── TIELO     常量 0
│   ├── TBUF      三态缓冲器
│   ├── PULLUP    弱上拉
│   ├── PULLDOWN  弱下拉
│   └── DELAY     延迟单元
│
├── ECO Cells（工程变更备用）
│   ├── SPARE_*   备用逻辑门
│   └── ECO_*     Metal-only ECO 单元
│
└── DFT Cells（测试辅助）
    ├── BSCAN      边界扫描单元
    ├── COMPRESSOR XOR 压缩树
    ├── MISR       MISR 单元
    └── LFSR       LFSR 单元
```

> 在实际综合后的网表中，出现频率最高的是 INV, BUF, NAND2, NOR2, AOI21, AOI22, OAI21, MUX21, DFF, CLKBUF——这几类往往占整个数字芯片标准单元数量的 **80% 以上**。

---

## 19. 核心总结：ASIC 工程师必知的 20 类单元

| # | 类型 | 代表单元 | 必须知道的要点 |
|---|------|---------|--------------|
| 1 | INV | `INVX1` | 面积最小，能用 INV 不用 BUF |
| 2 | BUF | `BUFX4` | 提高驱动、修时序 |
| 3 | NAND | `NAND2X1` | CMOS 中最优原语 |
| 4 | NOR | `NOR2X1` | PMOS 串联，多输入 NOR 慢 |
| 5 | AND/OR | `AND2X1` | NAND+INV 的派生门 |
| 6 | XOR/XNOR | `XOR2X1` | "最贵"基本门，尽量少用 |
| 7 | AOI | `AOI21X1` | 一级实现 AND-OR-INV，极高性价比 |
| 8 | OAI | `OAI21X1` | 一级实现 OR-AND-INV |
| 9 | MUX | `MUX21X1` | 传输门实现，Timing 与输入位置有关 |
| 10 | DFF | `DFFX1` | 最基础的寄存器 |
| 11 | DFFR | `DFFRX1` | 异步复位 DFF，注意 recovery/removal |
| 12 | SDFF | `SDFFX1` | DFT 后最常见单元 |
| 13 | DFFER | `DFFERX1` | 带 Enable + 复位的寄存器 |
| 14 | CLKBUF | `CLKBUF8` | 时钟树 backbone，平衡 rise/fall |
| 15 | CLKGATE | `CLKGATETSTX1` | 动态功耗优化核心（latch+AND） |
| 16 | LATCH | `LATCHX1` | 特殊用途，别让 DC 意外推断出来 |
| 17 | TIEHI/TIELO | `TIEHI` | 保护栅极，别直接接 VDD/VSS |
| 18 | FILL | `FILL4` | P&R 行填充，保 DRC |
| 19 | TAPCELL | `TAPCELL` | 防 latch-up |
| 20 | DECAP | `DCAP4` | 降 IR Drop |

---

> **最后一句**：Synopsys 的工具链（DC → ICC2 → PrimeTime → Formality）会在每个阶段与这些标准单元打交道，但你打开的 `.lib` / `.db` 文件来自 **Foundry 的 PDK**，不是 Synopsys。SAED 库只用于学习。
