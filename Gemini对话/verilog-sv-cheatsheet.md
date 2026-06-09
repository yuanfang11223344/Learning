# SystemVerilog 常用特性速查（vs 传统 Verilog）

> 来源：Gemini 对话，2026-05-12 ~ 2026-06-05

---

## 一、全 0 / 全 1 赋值

### Q: A = '1 表示全 1，全 0 是不是一样？

**A:** 是的。`'0` 全 0，`'1` 全 1，自动适配位宽。

```systemverilog
wire [7:0] a;
assign a = '1;  // a = 8'b11111111（自动适配 8 位宽）
assign a = '0;  // a = 8'b00000000

// 对比传统 Verilog：
assign a = {8{1'b1}};  // 需要手动指定重复次数
```

### Q: 对于数组，'{default:'1} 和 '1 是不是都可以表示数组全 1？

**A:** 不完全一样。

```systemverilog
reg [3:0] arr [0:7];  // 8 个元素，每个 4-bit

// 方法一：'1 直接给整个数组赋全 1
arr = '1;                    // 每个元素都是 4'b1111

// 方法二：'{default:'1} 用 default 模式
arr = '{default: '1};        // 同样效果，更显式

// 区别：
// '1 是 "fill with all ones"（全局填充）
// '{default:'1} 是 "all unspecified elements get '1"（模式填充，可部分覆盖）
arr = '{0: 4'd3, default: '1};  // 索引 0 赋 3，其余全 1
```

---

## 二、赋值模式 `'{...}`

### Q: Unpacked 数组怎么一次性赋值？（`'{...}` vs `{...}`）

**A:** Verilog 的 `{}` 只能拼接标量或 Packed 向量。SystemVerilog 的 `'{}` 是赋值模式（Assignment Pattern），可以给 Unpacked 数组整体赋值。

```systemverilog
reg [2:0] dfe_a_array [16:0];  // Unpacked：17 个元素

// ✅ SystemVerilog：单引号赋值模式
dfe_a_array = '{3'd3, 3'd2, 3'd1, ...};   // 列表顺序：从左到右对应高索引到低索引

// ❌ Verilog：大括号拼接对 Unpacked 无效
dfe_a_array = {3'd3, 3'd2, 3'd1, ...};    // 语法错误！
```

### Q: `pm_l2_r <= '{default : $signed(1'b0)}` 符合语法么？

**A:** 完全合法。对 unpacked 数组整体赋初值，所有元素设为 signed 0。如果要设非零初值，`'{default : '0}` 或 `'{default : $signed(1'b0)}` 都可以。

---

## 三、Signed 数组与 $signed()

### Q: wire signed 数组的每个元素是什么类型？

**A:** 取决于维度声明顺序。`signed` 只作用于它紧邻的 Packed 维。

```systemverilog
// A[i] 是 signed，A[i][j] 也是 signed（signed 向下传播）
wire signed [0:8] [PM_WIDTH-1:0] A [DOP/2-1:0];

// pm_normalize[i] 是 signed（signed 修饰 packed 维）
// pm_normalize[i][j] 也是 signed
wire signed [15:0] pm_normalize [DOP/2-1:0] [0:8];

// 如果想控制更细粒度：
assign A[i][j] = $signed(B[i][j]) - $signed(B[i][4]);
// 不管原始类型，$signed() 强制转为 signed
```

### Q: 维度顺序会影响索引剥离，怎么理解？

**A:** 数组 `wire signed [W-1:0] arr [0:8] [DOP/2-1:0]`：
- `arr[i]` 剥离最**右边**的维度 `[DOP/2-1:0]`，结果是 `[W-1:0]` 的 packed 向量 × 9 个
- `arr[i][j]` 再剥离中间维度 `[0:8]`，结果是 `[W-1:0]` 的单个 packed 向量
- 维度从右到左剥离！声明时维度写在右边的先被索引剥离

```
wire signed [15:0] data [0:3] [0:8];
//            \__/       \__/   \__/
//          packed维    unpacked unpacked
//          (最后剥离)  (中间)   (最先被索引剥离)

data[i]      → 剥离 [0:8]，得到 9 个 [15:0] 向量
data[i][j]   → 再剥离 [0:3]，得到 1 个 [15:0] 向量
data[i][j][k]→ 再剥离 [15:0]，得到 1 bit
```

---

## 四、Streaming Operator（流操作符）

### Q: 怎么把多维数组和扁平总线互转？

**A:** 用 `{>>{...}}` 或 `{<<{...}}`。

```systemverilog
wire [DOP_FLOW-1:0] [2:0] flow_array [FLOW_NUM-1:0];
wire [DOP*3-1:0] flow_bus;

// >> 从左到右流式拼装
assign flow_bus = { >> {flow_array} };

// << 从右到左流式拼装
assign flow_bus_rev = { << {flow_array} };
```

- `>>` = 从数组左边元素开始依次拼接（左元素→高位）
- `<<` = 从数组右边元素开始依次拼接（右元素→高位）
- 综合为纯连线，零面积开销

---

## 五、`$countones` 与统计操作

```systemverilog
// SystemVerilog 内置统计函数
wire [7:0] data = 8'b11001010;
$display("ones = %0d", $countones(data));  // 输出：4

// 等价于手写的：
integer cnt = 0;
for (integer i = 0; i < 8; i++) cnt += data[i];
```

---

## 六、总结与举一反三

### 核心要点

1. **`'0` / `'1` 自动适配位宽**——告别 `{N{1'b0}}`。
2. **`'{...}` 是 Unpacked 数组的整体赋值方式**——注意是单引号，不是普通 `{}`。
3. **`{>>{...}}` 流操作符是多维数组↔扁平总线的最佳方案**——零开销、一行搞定。
4. **`signed` 只修饰紧跟其后的 Packed 维**——如果数组中每个元素都要 signed，把 `signed` 放在最左边的 Packed 维前面。

### 举一反三

- **`.sv` 文件 vs `.v` 文件**：很多工具的默认语言模式由后缀决定。如果用了 SV 语法但文件是 `.v`，xvlog/VCS 可能报 `not allowed in this mode of verilog`。
- **`$signed()` vs `$unsigned()`**：只做类型转换不改变 bit 值。`$signed(3'b111)` = -1（3-bit signed）。
- **多维数组的 `$dimensions()`**：SV 提供 `$dimensions(arr)` 返回数组维数，`$left(arr, dim)` 返回指定维的左边界，可用于编写更通用的参数化代码。
- **`typedef` 简化复杂数组声明**：`typedef wire [7:0] [3:0] packed_2d_t; packed_2d_t my_array [0:15];` 让代码可读性大幅提升。
