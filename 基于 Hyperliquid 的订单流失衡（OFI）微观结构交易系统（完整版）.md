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

### 权重选择策略（重要修正）

**原设计问题：**
原文档使用固定权重（0.5, 0.3, 0.2），违反了时间尺度一致性原则：
- OFI：快信号，半衰期 ~10-50ms
- Lead-Lag：慢信号，半衰期 ~100-500ms
- Aggressive Volume：中速信号，半衰期 ~50-200ms

**修正方案：Regime-Dependent Weight（推荐）**

基于市场状态动态调整权重：

```text
# 基础权重
base_weights = {
    'leadlag': 0.5,
    'ofi': 0.3,
    'av': 0.2
}

# 高波动率调整：降低慢信号（Lead-Lag）权重
if volatility == 'high':
    w_leadlag *= 0.5  # 减半
    w_ofi *= 1.2      # 快信号更可靠
    w_av *= 1.1

# 低流动性调整：降低 OFI 权重（易受 spoof 影响）
if liquidity == 'low':
    w_ofi *= 0.3      # 大幅降低
    w_av *= 1.3      # 成交数据更可靠

# 极端状态调整：降低所有信号权重
if extreme:
    for w in weights:
        w *= 0.5  # 保守策略
```

**优点：**
- 实时响应市场状态变化
- 不需要历史数据
- 计算简单，延迟低

### 线性组合（更新）

```text
# 使用动态权重
weights = compute_weights(regime)

alpha_raw =
    weights['leadlag'] * LeadLag +
    weights['ofi'] * OFI_scaled +
    weights['av'] * AV
```

### 状态条件化（已集成到权重计算中）

状态条件化逻辑已集成到 `compute_weights()` 方法中，无需单独处理。

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

# 计算信号
leadlag = leadlag("BTC", "ETH")
ofi = ofi_scaled("ETH")
av = aggressive_volume("ETH")

signals = {
    'leadlag': leadlag,
    'ofi': ofi,
    'av': av
}

# 获取市场状态
regime = regime_detector.get_regime()

# 使用动态权重计算 Alpha
alpha_combiner = AlphaCombiner()
alpha = alpha_combiner.compute(signals, regime)

# 应用信任分数调整（如果启用）
if trust_manager:
    weights = alpha_combiner.compute_weights(regime)
    for signal_name in weights:
        trust_factor = trust_manager.get_weight_adjustment(signal_name)
        weights[signal_name] *= trust_factor
    # 重新计算 alpha
    alpha_raw = sum(weights[k] * signals[k] for k in signals)
    alpha = tanh(alpha_raw)

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
