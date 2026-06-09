# ASIC 标准单元库中 ULVT / LVT / SVT / RVT / NVT / HVT / EHVT 详解

## 1. 阈值电压 Vth 是什么

MOSFET 从关断到导通存在一个临界点：当栅极电压超过某个值后，沟道反型，漏源之间开始流过电流。这个临界值就是**阈值电压**（Threshold Voltage，简称 Vth 或 Vt）：

$$
V_{GS} > V_{th} \quad \Rightarrow \quad \text{晶体管导通}
$$

在先进工艺（如 7nm、5nm、3nm）的标准单元库中，同一逻辑门会提供多个 Vt 版本：

```
INV_X1_ULVT    # 超低阈值
INV_X1_LVT     # 低阈值
INV_X1_SVT     # 标准阈值
INV_X1_HVT     # 高阈值
```

它们的布尔函数完全一致（都是 Y = ~A），但电特性——速度与漏电——差异显著。

---

## 2. Vth 如何决定速度

MOS 管饱和区电流（简化模型）：

$$
I_{on} \propto (V_{GS} - V_{th})^{\alpha}, \quad 1 < \alpha < 2
$$

数字电路中 Vgs ≈ Vdd，令过驱动电压 $V_{ov} = V_{dd} - V_{th}$：

$$
I_{on} \propto V_{ov}^{\alpha}
$$

门传播延迟：

$$
t_{pd} \approx \frac{C_L \cdot V_{dd}}{I_{on}}
$$

**结论：Vth ↓ → Vov ↑ → Ion ↑ → tpd ↓ → 速度 ↑**

所以 ULVT 最快，EHVT 最慢。

---

## 3. Vth 如何决定漏电

晶体管关断后仍有亚阈值电流：

$$
I_{sub} = I_0 \cdot e^{\frac{V_{gs} - V_{th}}{n V_T}} \left(1 - e^{-\frac{V_{ds}}{V_T}}\right)
$$

当 Vgs = 0（关断状态），简化：

$$
I_{leak} \propto e^{-\frac{V_{th}}{n V_T}}
$$

其中 n 为亚阈值摆幅因子（1.2~1.5），V_T 为热电压（室温约 26mV）。

这意味着 **Vth 每降低约 60mV（n=1 时），漏电增加约一个数量级**。

**结论：Vth ↓ → 漏电指数级 ↑**

---

## 4. 七种 Vt 一览

| 缩写 | 全称 | Vth 水平 | 速度 | 漏电 | 核心用途 |
|------|------|----------|------|------|----------|
| **ULVT** | Ultra-Low Vth | 最低 | 极快 | 极大 | 极关键路径、Fmax 瓶颈 |
| **LVT** | Low Vth | 低 | 快 | 大 | Setup 关键路径 |
| **SVT** | Standard Vth | 标准 | 中 | 中 | 默认平衡选择 |
| **RVT** | Regular Vth | 标准 | 中 | 中 | 同 SVT（命名差异） |
| **NVT** | Normal Vth | 标准 | 中 | 中 | 同 SVT（命名差异） |
| **HVT** | High Vth | 高 | 慢 | 小 | 非关键路径、降漏电 |
| **EHVT** | Extra-High Vth | 最高 | 极慢 | 极小 | Always-On、常电域 |

> RVT、NVT 与 SVT 本质相同，只是不同 Foundry/PDK 的命名习惯不同。TSMC 习惯用 SVT/HVT，某些库用 RVT，还有些用 NVT。三者可以互换理解。

---

## 5. 速度-漏电排序

```
速度（快 → 慢）:
  ULVT  >  LVT  >  SVT ≈ RVT ≈ NVT  >  HVT  >  EHVT

漏电（大 → 小）:
  ULVT  >  LVT  >  SVT ≈ RVT ≈ NVT  >  HVT  >  EHVT
```

速度和漏电排序方向一致——**没有又快又省电的单元**。这就是 Vt 选型的根本权衡。

---

## 6. 各 Vt 详细分析

### 6.1 ULVT — 极限速度，代价高昂

**适用条件严格**：仅当 LVT 仍无法收敛 setup、且该路径是 Fmax 瓶颈时才使用。

- 典型场景：CPU 核内关键路径、NPU MAC 阵列、SerDes 内部高速逻辑、高速乘法器
- 漏电代价：可比 SVT 高 5~20 倍（取决于工艺）
- 策略：单芯片内 ULVT 占比通常 < 3%

### 6.2 LVT — Setup 收敛主力

LVT 是后端 Timing ECO 中最常用的提速手段。

- 典型场景：ALU、加法树、Cache tag 比较、高速总线仲裁
- 比 SVT 快 15%~30%，漏电高 2~5 倍
- 策略：Setup 违例路径优先换 LVT，收敛后对正 slack 路径换回 HVT

### 6.3 SVT / RVT / NVT — 默认基线

综合工具的默认选择。如果不是极端性能场景也不是极端低功耗场景，用 SVT 就够了。

- 典型场景：状态机、普通控制逻辑、DMA 控制、总线桥、通用寄存器
- 策略：综合时 70%~80% 的 cell 应该是 SVT/RVT

### 6.4 HVT — 降功耗的主力

当路径有充足正 slack 时，把 SVT 换成 HVT 几乎不会影响功能，但能显著降低静态功耗。

- 典型场景：非关键控制路径、配置寄存器、低速外设（I2C、SPI、UART、GPIO）、JTAG
- 比 SVT 慢 20%~40%，漏电低 50%~80%
- 策略：Leakage recovery ECO 中 HVT 是首选

### 6.5 EHVT — Always-On 域的标配

芯片中有一部分电路永远不掉电（RTC、PMU、唤醒逻辑），对它们来说漏电是 24×7 的开销。

- 典型场景：RTC、PMU、Watchdog、唤醒检测、电源序列控制
- 速度不是问题（这些电路通常工作在 kHz 级）
- 策略：Always-On 域尽可能全用 EHVT，慎用 LVT

---

## 7. 选型决策矩阵

| 面临的情况 | 动作 | 原因 |
|-----------|------|------|
| Setup slack < 0 | SVT→LVT 或 LVT→ULVT | 提升速度，抢回负 slack |
| Setup slack >> 0（如 +300ps） | LVT→SVT 或 SVT→HVT | 用余量换功耗 |
| Leakage 超标 | 非关键路径大面积 LVT→HVT | 指数级降漏电 |
| Hold slack < 0 | 插 delay buffer；或 LVT→SVT→HVT | 增加路径延迟 |
| 常规路径，无特殊约束 | 保持 SVT | 平衡、稳妥 |
| Always-On 域 | HVT / EHVT | 待机功耗优先 |
| Fmax 差最后几 ps | 少量 ULVT | 最后的手段 |

---

## 8. Vt Swap 实操示例

### Setup 修复
```
Before:  NAND2_X1_SVT + INV_X2_SVT  →  Slack = -35ps
After:   NAND2_X1_LVT  + INV_X2_LVT   →  Slack = +8ps
代价:    Leakage 增加约 3x（仅这两个 cell）
```

### Leakage 回收
```
Before:  NAND2_X1_LVT + INV_X2_LVT  →  Slack = +600ps（过度设计）
After:   NAND2_X1_HVT + INV_X2_HVT  →  Slack = +250ps（仍满足）
收益:    Leakage 降低约 60%
```

### Hold 修复（慎用 Vt 方式）
```
方式一（换 Vt）:  LVT → HVT（增加延迟，但可能影响其他路径的 setup）
方式二（推荐）:  插入 delay cell 或 buffer（精确可控）
```

---

## 9. 多 Vt 库在 EDA 流程中的形态

DC/FC 中加载多 Vt 库：

```tcl
set target_library [list \
    sc9mc_ss0p72v125c_ulvt.db \
    sc9mc_ss0p72v125c_lvt.db  \
    sc9mc_ss0p72v125c_svt.db  \
    sc9mc_ss0p72v125c_hvt.db  \
]
```

工具自动根据约束在四个 Vt 版本间选择。也可以施加 Vt 使用比例约束：

```tcl
# 限制 ULVT 不得超过总 cell 的 5%
set_max_percentage -ulvt 5
```

或用 `set_dont_use` 禁止某些 Vt 在特定模块中出现（如 Always-On 域禁止 LVT）。

---

## 10. PVT Corner 与 Vt 选择的互动

| Corner | 条件 | 难点 | Vt 对策 |
|--------|------|------|---------|
| **SS** (Slow-Slow) | 慢工艺 + 低电压 + 高温 | Setup 最难收敛 | 关键路径多用 LVT |
| **FF** (Fast-Fast) | 快工艺 + 高电压 + 低温 | Hold 最难收敛 | 控制低 Vt 比例，防 hold 恶化 |
| **TT** (Typical) | 标称条件 | 基准参考 | 默认 SVT 为主 |

Setup 和 Hold 是矛盾的——在 SS 角修 setup 时大量用 LVT，到 FF 角可能导致 hold 违例。所以 Vt 优化需要多 corner 同时检查，不能只看一个 corner。

---

## 11. 三条核心工程纪律

**一、别把 LVT 当默认。** LVT 是"药"，不是"饭"。默认用 SVT，哪里 setup 不过才下药。全芯片 LVT 比例超过 30% 就要警惕静态功耗。

**二、别一刀切禁用 HVT。** HVT 不是"垃圾单元"，它是用正 slack 换功耗的筹码。Slack 很大的路径不用 HVT 等于白送功耗。

**三、不同域不同策略。** CPU 核内和 Always-On 域的 Vt 策略完全不同——前者可以接受较高漏电换频率，后者漏电优先于一切。

---

## 12. 一句话总结

| Vt | 一句话 |
|----|--------|
| ULVT | 速度极限，漏电代价极高，仅用于 Fmax 瓶颈 |
| LVT | Setup 修复主力，关键路径的首选 |
| SVT/RVT/NVT | 默认基线，平衡之选 |
| HVT | 降漏电主力，用 slack 余量换低功耗 |
| EHVT | 常电域的守护者，漏电最低 |
