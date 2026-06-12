# SerDes均衡与时钟恢复总览

> 整理日期：2026-06-12  
> 来源：本周 ChatGPT 对话导出  
> 范围：FFE、CTLE、DFE、CDR

## 总览

这组笔记围绕高速 SerDes 接收链路的关键问题展开：信道低通导致高频衰减、码间串扰和眼图闭合；发送端和接收端需要通过不同类型的均衡和时钟恢复机制，让采样点重新落在足够可靠的位置。

## 主题拆分

- [[SerDes发送端FFE前馈均衡]]：发送端预失真，使用码元间隔的FIR抽头提前补偿信道ISI。
- [[SerDes接收端CTLE连续时间线性均衡]]：接收端模拟连续时间高频提升，改善低通信道造成的边沿变慢和眼图闭合。
- [[SerDes接收端DFE判决反馈均衡]]：接收端基于历史判决反馈抵消post-cursor ISI，重点改善采样时刻。
- [[SerDes接收端CDR时钟数据恢复]]：从数据跳变中恢复采样相位，使采样点动态跟随眼图中心。

## 链路关系

```text
TX data
  -> TX FFE
  -> Channel / Package / Connector
  -> RX CTLE
  -> Sampler / Slicer
  -> DFE feedback
  -> CDR phase tracking
  -> Deserializer / PCS
```

## 关键区别

| 模块 | 位置 | 域 | 主要处理对象 | 优点 | 主要限制 |
| --- | --- | --- | --- | --- | --- |
| FFE | 发送端 | 离散码元/数字或混合信号 | pre/post cursor ISI预补偿 | 可提前整形，减轻接收端压力 | 发射幅度受限，可能降低主光标能量 |
| CTLE | 接收端前端 | 模拟连续时间 | 低通衰减、高频不足 | 不依赖判决，先打开眼图 | 可能放大高频噪声 |
| DFE | 接收端判决后反馈 | 判决反馈/非线性 | post-cursor ISI | 不直接放大输入噪声，采样点处效果强 | 不能直接处理pre-cursor，可能错误传播 |
| CDR | 接收端时钟恢复 | 闭环相位控制 | 采样相位误差 | 动态跟随眼图中心 | 依赖数据跳变和环路设计 |

## 后续补充方向

- SerDes均衡自适应算法：LMS、sign-sign LMS、眼图监测与BER优化。
- CDR相位检测器：Alexander bang-bang PD、linear PD、early-late结构。
- 抖动指标：jitter transfer、jitter tolerance、jitter generation。
- 训练流程：协议初始化、preset、coefficient update、link training。
