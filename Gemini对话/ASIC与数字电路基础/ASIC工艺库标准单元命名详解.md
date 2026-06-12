# ASIC 工艺库标准单元命名详解

> 来源：Gemini 对话，2026-06-09

---

## 一、完整命名分段解析

### Q: CMPE42D1COTBWP228H8P57CPDLVT 代表什么意思？

**A:** 这是 SMIC/TSMC 等 Foundry 的 stdcell 完整命名。分段：

| 段位 | 示例 | 含义 |
|------|------|------|
| 功能 | `CMPE42` | 4-2 Compressor（4输入2输出的压缩器，常用于乘法器） |
| 驱动变体 | `D1` | Drive 1（驱动强度级别，D1/D2/D4 递增） |
| 版图/工艺 | `COTBWP228H8P57CPD` | 具体工艺节点、PDK 版本、晶体管类型标识 |
| 阈值类型 | `LVT` | Low Vt |

### 常见功能前缀速查

| 前缀 | 全称 | 功能 |
|------|------|------|
| `CMPE` | Compressor | 乘法器压缩树（Wallace Tree 节点） |
| `AOI` | AND-OR-Invert | 与或非复合门 |
| `OAI` | OR-AND-Invert | 或与非复合门 |
| `FA` | Full Adder | 全加器 |
| `HA` | Half Adder | 半加器 |
| `DFF` | D Flip-Flop | D 触发器 |
| `SDFF` | Scan DFF | 扫描触发器 |
| `INV` | Inverter | 反相器 |
| `BUF` | Buffer | 缓冲器 |
| `MUX` | Multiplexer | 多路选择器 |
| `CLK` | Clock | 时钟相关单元 |
| `TBUF` | Tri-state Buffer | 三态缓冲器 |
| `DEL` | Delay | 延迟单元 |

### 常见 Vt 后缀速查

| 后缀 | 速度 | 漏电 | 用途 |
|------|------|------|------|
| `ULVT` | 极快 | 极大 | Fmax 瓶颈 |
| `LVT` | 快 | 大 | Setup 关键路径 |
| `SVT/RVT/NVT` | 中等 | 中等 | 默认逻辑 |
| `HVT` | 慢 | 小 | 非关键路径 |
| `EHVT` | 极慢 | 极小 | Always-On 域 |

---

## 二、从 Timing Report 一眼判断路径属性

### Q: 看 Timing Report 时怎么快速判断路径的硬件速度属性？

**A:** 直接扫一眼路径上 cell 名称的 Vt 后缀：

```
U1: CMPE42D1...LVT  → 低阈值 → 速度快，漏电大 → 这条路径已经用了快速单元
U2: NAND2X1...HVT   → 高阈值 → 速度慢，漏电小 → 如果 setup 不过，可换 LVT
U3: INVX8...SVT     → 标准阈值 → 平衡 → 默认状态
```

**实战判断逻辑：**
- Setup 违例 + 路径上全是 HVT → 换 LVT 立竿见影
- Slack 很大 + 路径上全是 LVT → 换 HVT 降漏电而不影响 timing
- 路径混合 LVT/HVT/SVT → 是经过 EDA 工具优化的合理状态

---

## 三、不同 Foundry 命名风格对比

| Foundry | 命名风格 | 示例 |
|---------|---------|------|
| TSMC | 较短，功能 + 驱动 + Vt | `NAND2D1BWP7T` |
| SMIC | 较长，含工艺标识 | `NAND2X1COTBWP228H8P57CPDSVT` |
| SAED（教育库） | 最简洁 | `NAND2X1`, `DFFRX1` |
| Xilinx（FPGA 原语） | 功能导向 | `FDRE`, `LUT6`, `CARRY8` |

---

## 四、总结与举一反三

### 核心要点

1. **命名 = 功能 + 驱动 + Vt + 工艺**——看后缀秒判速度和漏电。
2. **Vt 后缀是 Timing Report 的最高频信息**——一眼看出路径是否还有优化空间。
3. **不同 Foundry 命名不同，但结构规律一致**——功能前缀 + 参数 + Vt 后缀。

### 举一反三

- **命名和 Liberty 文件的对应**：`.lib` 中 `cell(NAND2X1)` 的 `cell_leakage_power` 字段直接反映 Vt 后缀——ULVT 漏电是 HVT 的 10~50 倍。
- **和标准单元库的关系**：相同功能前缀 + 不同 Vt 后缀 = 同一逻辑门的不同阈值版本，它们在版图上完全相同（仅注入不同）。详见 `Synopsys常见ASIC标准单元库详解.md` 第 0 节。
- **Timing ECO 中 Vt Swap 的依据**：看 Timing Report → 锁定 HVT 单元 → 换 LVT → 重跑 STA 验证。这是最常用的 ECO 流程之一。
