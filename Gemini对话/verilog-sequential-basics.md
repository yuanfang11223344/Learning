# Verilog 时序逻辑：DFF、Latch、复位与 always 块

> 来源：Gemini 对话，2026-05-22 ~ 2026-06-09

---

## 一、同步复位与异步复位

### Q: 同步复位和异步复位有什么区别？

**A:** 核心区别在于复位是否依赖时钟。

```verilog
// 异步复位：复位信号不受时钟控制，立即生效
always @(posedge clk or negedge rst_n)
    if (!rst_n)
        q <= 1'b0;
    else
        q <= d;

// 同步复位：复位只在时钟边沿生效
always @(posedge clk)
    if (!rst_n)
        q <= 1'b0;
    else
        q <= d;
```

| | 异步复位 | 同步复位 |
|------|---------|------|
| 敏感列表 | `posedge clk or negedge rst_n` | `posedge clk` |
| 复位速度 | 立即生效 | 等下一个时钟沿 |
| recovery/removal | 需要检查 | 不需要（是数据路径） |
| FPGA 资源 | 触发器自带异步复位端 | 综合为 MUX + DFF |
| DFT | 复位树需要额外处理 | 友好（复位是数据） |

### Q: 标准的 D 触发器自带异步复位引脚，没有同步复位端？

**A:** 对。D 触发器硬件上天然有的控制引脚通常是异步的（CLR/RESET）。但要实现同步复位也很简单——在 D 输入前加一个 MUX，RST 有效时 MUX 选 0 送入 D。综合器看到 `if(rst) q<=0` 会自动用 MUX 实现同步复位，不会浪费额外的触发器引脚。

---

## 二、Latch 的产生与避免

### Q: 为什么在 always@(*) 里面只写 if 不写 else 会产生 Latch？

**A:** 综合器分析组合逻辑时，必须确保**所有输入组合下输出都有确定值**。

```verilog
// ❌ 产生 Latch
always @(*)
    if (en)
        q = d;
// en=0 时 q 没有被赋值 → 综合器推断：q 需要"记住"上一个值 → Latch

// ✅ 不会产生 Latch
always @(*)
    if (en)
        q = d;
    else
        q = 1'b0;
// 所有分支都赋值了 → 纯组合逻辑
```

### Q: 虽然 always@(posedge clk) 在 en 拉低时也不变，为什么它不产生 Latch？

**A:** 因为 `posedge clk` 声明了这是**时序逻辑**。时钟边沿之间的保持是 DFF 的固有特性，不需要额外推断存储单元。

```
always @(posedge clk) → 只会推导 DFF（主从两级锁存器级联）
always @(*)          → 推导组合逻辑，不完整赋值 → Latch
```

### Q: 你的 en 拉低时输出的 Q 是不是保持 D 的最后一个值？

**A:** 对。对于 Latch，EN=0 时 Q 锁住 EN 下降沿瞬间 D 的值。对于 DFF，没有时钟边沿时 Q 保持上一个时钟边沿采样的值。两者都能"记住"，但触发机制不同——Latch 是电平锁存，DFF 是边沿采样。

### Q: 我发现 en 时序很紧时，综合工具会把时序逻辑综合成 Latch？

**A:** 正常综合器不会把 `always @(posedge clk)` 综合成 Latch。如果出现这种情况，检查：(1) 敏感列表是否误写成 `always @(*)`；(2) 时钟约束是否正确；(3) 综合日志里的推断原因。更可能是组合逻辑路径的 Latch 推断被误读。

---

## 三、always @(*) 的用法

### Q: 在 Verilog 中 always @(*) 的用法？

**A:** `always @(*)` 是**组合逻辑**的自动敏感列表，等价于列出所有输入信号。

```verilog
// 组合逻辑描述：统计 ones 个数
wire [3:0] data_in;
reg  [2:0] ones_cnt;

always @(*) begin
    ones_cnt = 3'd0;
    for (integer i = 0; i < 4; i = i + 1) begin
        if (data_in[i])
            ones_cnt = ones_cnt + 1;
    end
end
```

规则：
- 用 `=`（阻塞赋值），不用 `<=`
- 所有被赋值的变量必须在**所有分支**都有赋值
- 被读取的信号会**自动**加入敏感列表

### Q: 为什么 always@(*) 里组合逻辑变量要声明为 reg？

**A:** 这是 Verilog 语法的历史包袱。`reg` 只表示"在 procedural block 中被赋值"，**不代表**实际硬件是寄存器。`always @(*)` 里的 `reg` 综合出来是组合逻辑（线 + 门）。

```
Verilog 语法 ≠ 硬件映射
reg 在 always @(*) 中 → 综合为 wire + 门
reg 在 always @(posedge clk) 中 → 综合为 DFF
```

---

## 四、寄存器结构与 CE

### Q: 在数字电路中寄存器长啥样？

**A:** 最基本的 D 触发器只有 D、CLK、Q 三个端口。实际工程中常见的是带 CE（使能）和 RST（复位）的寄存器。

```
           +-----------+
D -------->| D      Q  |------> Q
CLK ------>| >         |          > 表示边沿触发
CE ------->| CE        |
RST ------>| R         |
           +-----------+
```

Xilinx FPGA 中这就是 **FDRE** 原语。

### Q: 你的寄存器的使能信号有么？

**A:** 最简 DFF 图没有使能，因为那是"最基础"的模型——每个 CLK↑ 都更新一次。带使能的寄存器内部是 MUX+DFF 结构：

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

EN=1 时 MUX 选 D（装载），EN=0 时 MUX 选 Q（保持）。

### Q: CP 是啥？

**A:** CP = Clock Pulse，即时钟信号，等价于 CLK。一般在教材和原理图中用 CP 表示时钟输入端。

---

## 五、总结与举一反三

### 核心要点

1. **异步复位用 `negedge rst_n` 入敏感列表，同步复位不。** 异步复位要关注 recovery/removal 时序。
2. **`always @(posedge clk)` 永远推导 DFF，不写 else 也没事。** DFF 天生会保持。
3. **`always @(*)` 所有分支必须赋值，否则得 Latch。** 这是组合逻辑的基本约束。
4. **CE（Clock Enable）本质是 MUX+DFF**，不是控制时钟通断。FPGA 用 CE，ASIC 才用 Clock Gating。

### 举一反三

- **Hold 时序修复**：有时需要故意让某条路径"多延迟一点"，可以插入 LVT→HVT 的替换，或者直接插入 DELAY cell。但 hold 修复最常见的还是插 buffer。
- **Latch 的合法用途**：ICG 内部（防毛刺）、Balloon Latch（Retention 保持）、Time Borrowing（高性能设计）。
- **从 FDRE 到标准单元库**：`DFFEX1`(SAED) = `FDRE`(Xilinx) = 同步复位 + 使能 DFF。不同厂家命名不同，但硬件结构完全一致。详见 `synopsys_stdcell_guide.md` 第 3 节。
- **跨时钟域**：DFF 同步器（2级/3级）是处理异步信号的基础，但 CDN 的约束（set_max_delay、set_false_path）同样重要。
