# 基于 Hyperliquid 的订单流失衡（OFI）微观结构交易系统（完整版）

> **定位说明**：
> 本文不是“OFI 指标介绍”，而是一份**可落地的微观结构 Alpha 系统设计文档**。它在原有订单流失衡（Order Flow Imbalance, OFI）理论基础上，系统性补全了 **信号过滤、状态识别、信号组合、执行与风险控制**，并针对 **Hyperliquid 永续合约交易所**进行定制。

---

## 一、为什么必须“重写”OFI

传统 OFI 文章往往隐含一个危险假设：

> **OFI 本身就是一个可直接交易的 Alpha**

在真实市场（尤其是加密永续）中，这是不成立的。

* OFI 测量的是 **盘口买卖力量的不平衡**，而不是价格必然上涨或下跌
* OFI 是 **高噪声、强状态依赖** 的微观量
* 不经过状态过滤和组合，OFI 极易被 spoof、撤单、薄流动性放大

**结论：**

> **OFI 必须作为“信号成分（Signal Component）”存在，而不是完整策略**

---

## 二、Hyperliquid 为什么适合 OFI

Hyperliquid 具备传统 CEX 与链上透明度的结合特征：

* 单一中央限价订单簿（Central LOB）
* 极低延迟的 WebSocket L2 diff
* 流动性相对真实，spoof 成本高
* 永续合约为主，执行反馈极快

这使得：

> **OFI 在 Hyperliquid 上更接近“真实订单流压力”，而非噪声残影**

---

## 三、完整微观结构 Alpha Pipeline（Hyperliquid 版）

```
数据层 → 状态识别 → Alpha 信号层 → 决策融合 → 执行层 → 风险与反馈
```

OFI 在系统中 **贯穿三层**：

* Alpha 信号层（压力因子）
* 决策层（权重 / 条件）
* 执行层（aggressiveness 调节）

---

## 四、① 数据层（Event-driven）

### 必需数据

* L2 Order Book（增量 diff，L1–L5）
* Trades（taker buy / sell）
* Mid Price / Mark Price

**关键原则：**

* OFI 基于 **order book 更新事件**，不是基于成交
* 必须本地维护一致的订单簿状态

---

## 五、② 状态识别层（Regime Filter）

这是 **是否允许 Alpha 发声的开关层**。

### 1. 流动性状态（最重要）

```text
depth = Σ(bid_L1–L5) + Σ(ask_L1–L5)
spread = ask1 - bid1
```

规则示例：

```text
if depth < Q25 or spread > Q90:
    禁止交易
```

---

### 2. 波动率状态

```text
vol = EWMA(std(mid_price_return))
```

* vol > Q95：禁止开新仓
* vol > Q99：强制风控

---

### 3. 清算 / 极端状态（Hyperliquid 特有）

* 突发大成交 + 深度瞬间消失
* 强趋势下盘口失效

```text
if liquidation_cluster_detected:
    冻结 Alpha N 秒
```

---

## 六、③ Alpha 信号层（多因子，而非单 OFI）

### 1. Lead-Lag（方向信息）

最常见组合：

* BTC-PERP → ETH-PERP

```text
LeadLag = sign(Δ mid_price_lead over 50–200ms)
```

> Lead-Lag 决定 **“往哪边做”**

---

### 2. OFI（盘口压力因子）

#### 基础定义（Hyperliquid 稳定版）

```text
OFI = Δbid_size@best_bid - Δask_size@best_ask
```

#### 三重标准化（必须）

```text
OFI_scaled = OFI / depth_L1–L5 / σ_mid_price
```

> OFI 不再是“数量”，而是 **单位流动性 + 单位风险下的压力**

---

### 3. Aggressive Volume（真假过滤）

```text
AV = taker_buy_volume - taker_sell_volume
```

| OFI | AV | 含义           |
| --- | -- | ------------ |
| 强   | 强  | 真买卖力量        |
| 强   | 弱  | spoof / 撤单诱导 |
| 弱   | 强  | 成交驱动型波动      |

---

### 4. Micro-price（微结构领先指标）

```text
MP = (ask1*bid_size + bid1*ask_size) / (bid_size + ask_size)
```

---

## 七、④ 决策融合层（State-aware Alpha）

### 线性组合（基础）

```text
alpha_raw =
    w1 * LeadLag +
    w2 * OFI_scaled +
    w3 * AV
```

### 状态条件化

```text
if liquidity_low:
    w2 ↓
if volatility_high:
    w1 ↓
```

### 非线性压缩（必须）

```text
alpha = tanh(alpha_raw)
```

> 防极端、稳仓位、防过拟合

---

## 八、⑤ 执行层（Execution ≠ 下单）

OFI 在此 **第二次发挥作用**。

```text
if OFI_z > 3:
    IOC / Market
elif OFI_z > 1:
    Post-only limit
else:
    不交易
```

**Hyperliquid 实战原则：**

* 绝大多数时间使用 post-only
* 只有在 Lead-Lag + 极端 OFI 同向时才吃单

---

## 九、⑥ 风险与反馈层（闭环系统）

### 1. 仓位与清算控制

```text
position ≤ equity × leverage × risk_factor
liquidation_distance > buffer
```

---

### 2. 执行质量反馈（信号可信度）

```text
if OFI 强 but slippage ↑ and fill_rate ↓:
    降低 OFI 权重
```

> 系统会动态“学会不信盘口”

---

## 十、最小可运行逻辑（伪代码）

```python
if not regime_ok():
    return

leadlag = leadlag("BTC", "ETH")
ofi = ofi_scaled("ETH")
av = aggressive_volume("ETH")

alpha = tanh(0.5*leadlag + 0.3*ofi + 0.2*av)

size = base_size * abs(alpha)
side = BUY if alpha > 0 else SELL

execute(symbol="ETH-PERP", side=side, size=size, aggressiveness=ofi)
```

---

## 十一、核心认知总结

> **OFI 是市场是否“配合你交易”的探针**
> **不是预测价格的水晶球**

在 Hyperliquid 这种高透明、快反馈的永续市场：

* 单一 OFI ≠ Alpha
* OFI + Lead-Lag + 流动性状态 + 执行控制 = 可交易系统

---

## 十二、结语

本文的目标不是教你“怎么算 OFI”，而是回答一个更重要的问题：

> **OFI 在一个真实、可长期运行的交易系统里，应该放在哪？**

答案是：

> **它是贯穿信号、决策、执行三层的核心微观结构组件，但永远不是孤立策略。**

---

**完**

---

# 附录：评估意见的修正与吸收（Author Revision Note）

本系统在内部评估中收到关于数学严谨性、信号组合与反馈机制的专业审查意见。以下内容并非简单回应，而是将**合理批评正式吸收进系统设计**，作为本文档的组成部分。

## 1. OFI 标准化与量纲问题（修正说明）

原有 OFI 标准化形式：

[ OFI_{scaled} = \frac{OFI}{Depth \cdot \sigma_{mid}} ]

在表达上容易被误解为试图构造具备明确经济量纲的解释性变量。本文在此明确：

**OFI 的标准化目标并非经济解释，而是条件化与稳定化（conditioning & stabilization）。**

因此，标准化被正式拆分为两步：

1. **流动性条件化**：
   [ OFI^{liq}_t = \frac{OFI_t}{Depth_t} ]

2. **统计标准化（EWMA / Rolling Z-score）**：
   [ OFI^{z}_t = \frac{OFI^{liq}_t - \mu_t}{\sigma_t} ]

该处理的目的在于使 OFI 在不同波动率与流动性环境下保持可比较性，而非赋予其可解释的经济量纲。

---

## 2. Alpha 权重设置的修正

原文档中使用的固定权重（如 0.5 / 0.3 / 0.2）在多时间尺度微观结构信号中隐含了不合理的稳定性假设。本文档已正式修正该点：

* 不再推荐固定权重
* 权重应满足以下至少一种机制：

**（A）基于信号有效性的动态权重（IC / Hit-rate）**
**（B）相关性约束下的风险均衡权重**
**（C）基于 Regime 的条件化权重（实盘优先）**

在实际工程中，推荐方案（C），以避免在线优化带来的不稳定性。

---

## 3. Lead-Lag 信号建模选择的澄清

评估指出使用 (sign()) 函数可能丢失幅度信息。本文档在此明确补充：

* 在亚秒级微观结构中，幅度信息高度噪声化
* 使用方向信号是**主动的噪声抑制选择**，而非简化处理
* 幅度信息由 OFI 与 Aggressive Volume 等信号补充

Lead-Lag 时间窗口（如 50–200ms）并非任意设定，而应通过：

* 最大化 lead-return 或 cross-correlation
* 或作为 Regime-dependent 参数动态调整

---

## 4. 反馈与信号可信度机制的形式化

原文档中的反馈机制为启发式描述，现已明确最低数学形式要求：

[ Trust_t = (1-\alpha) Trust_{t-1} + \alpha \cdot Score_t ]

其中：

* Score_t 由成交质量、滑点与方向一致性构成
* 更新频率与裁剪范围（clip）需显式定义

该反馈机制的目标是调节信号权重与执行激进程度，而非构造完整学习模型。

---

## 5. 修正后关键问题总结

| 类别          | 修正后定义          | 状态    |
| ----------- | -------------- | ----- |
| OFI 标准化     | 条件化与统计稳定化目标已明确 | ✅ 已修正 |
| Alpha 权重    | 固定权重已弃用        | ✅ 已修正 |
| Lead-Lag 定义 | 建模动机与窗口选择已澄清   | ✅ 已修正 |
| 反馈机制        | 最低数学形式已给出      | ✅ 已修正 |

**上述修正已被视为系统设计的一部分，而非附加说明。**
