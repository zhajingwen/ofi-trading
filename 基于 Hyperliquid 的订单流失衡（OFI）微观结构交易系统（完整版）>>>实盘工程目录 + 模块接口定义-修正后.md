# 基于 Hyperliquid 的订单流失衡（OFI）微观结构交易系统（完整版）>>>实盘工程目录 + 模块接口定义（修正后）

好，这一步我们**直接进入"可实盘工程"的层级**。
下面给你的是一套 **真实可维护、可扩展、符合 Hyperliquid 特性的实盘工程目录 + 模块接口定义**。
不是 demo，而是**量化团队常见的最小可长期演化架构（Minimum Viable Trading System）**。

**版本说明**：本文档已根据专业评估意见进行修正，所有修正内容已整合进接口定义。

---

# 一、总体设计原则（先立规矩）

在 Hyperliquid 上做微观结构，你的工程必须满足：

1. **事件驱动（Event-driven）**
   ❌ bar / sleep loop
   ✅ WS → callback → signal

2. **解耦（Loose Coupling）**

   * 数据 ≠ 信号 ≠ 执行 ≠ 风控

3. **可降级（Fail-safe）**

   * 任何一层异常 → 系统自动降频或停手

4. **可观测（Observable）**

   * 每一个 OFI 决策都能回溯

---

# 二、工程目录结构（实盘级）

```text
hyperliquid_ofi_trader/
│
├── config/
│   ├── symbols.yaml
│   ├── thresholds.yaml
│   └── risk.yaml
│
├── core/
│   ├── engine.py              # 主事件引擎
│   ├── clock.py               # 本地时间 & 延迟监控
│   └── state.py               # 全局交易状态
│
├── data/
│   ├── ws_client.py            # Hyperliquid WebSocket
│   ├── orderbook.py            # 本地 L2 Order Book
│   ├── trades.py               # 成交流处理
│   └── market_state.py         # mid / mark / vol
│
├── regime/
│   ├── liquidity.py            # 深度 / spread 状态
│   ├── volatility.py           # 波动率 Regime
│   ├── liquidation.py          # 极端 / 清算检测
│   └── regime_detector.py     # 市场状态检测器（新增）
│
├── signals/
│   ├── ofi.py                  # OFI 计算
│   ├── leadlag.py              # Lead-Lag
│   ├── aggressive_volume.py    # Taker imbalance
│   ├── microprice.py           # Micro-price
│   └── signal_state.py         # 信号缓存
│
├── alpha/
│   ├── combiner.py             # Alpha 融合
│   ├── conditioning.py         # 状态条件化
│   └── compression.py          # tanh / clip
│
├── execution/
│   ├── router.py               # 下单路由
│   ├── order_manager.py        # 订单生命周期
│   └── tactics.py              # post-only / IOC 逻辑
│
├── risk/
│   ├── position.py             # 仓位管理
│   ├── exposure.py             # 杠杆 / 风险敞口
│   └── kill_switch.py          # 紧急停止
│
├── feedback/
│   ├── slippage.py             # 滑点分析
│   ├── fill_rate.py            # 成交质量
│   └── trust_score.py          # 信号可信度
│
├── utils/
│   ├── ewma.py
│   ├── stats.py
│   └── logger.py
│
├── main.py                     # 启动入口
└── requirements.txt
```

---

# 三、核心模块接口定义（重点）

下面我**只给"关键模块"的接口定义**，这些决定了你系统的上限。

---

## 1️⃣ data/orderbook.py（OFI 的地基）

```python
class OrderBook:
    def __init__(self, depth=5):
        self.bids = {}
        self.asks = {}

    def apply_diff(self, side, price, size):
        """
        side: 'bid' or 'ask'
        size=0 → remove
        """

    def best_bid(self) -> tuple[price, size]
    def best_ask(self) -> tuple[price, size]

    def depth(self, levels=5) -> float
```

📌 **要求**：

* apply_diff 必须 O(1)
* 不允许重建 snapshot

---

## 2️⃣ signals/ofi.py（OFI 核心）

```python
class OFICalculator:
    def __init__(self, orderbook):
        self.prev_bid_size = None
        self.prev_ask_size = None

    def update(self) -> float:
        """
        返回原始 OFI
        """
```

```python
class OFINormalizer:
    def scale_by_liquidity(self, ofi, depth) -> float:
        """
        流动性条件化
        
        OFI^{liq} = OFI / Depth
        """
    
    def scale_by_zscore(self, ofi_liq, ofi_history, window=1000) -> float:
        """
        统计标准化（EWMA / Rolling Z-score）
        
        OFI^{z} = (OFI^{liq} - μ) / σ
        """
    
    def scale(self, ofi, depth, volatility) -> float:
        """
        完整标准化流程（兼容旧接口）
        
        1. 流动性条件化
        2. 统计标准化
        """
```

📌 **注意**：

* OFI 只在 L1 变化时更新
* 不和 trade 混用

📌 **重要修正**：

* **原设计问题**：标准化量纲不明确
* **修正方案**：明确标准化目标为条件化与稳定化
  - 第一步：流动性条件化（消除规模影响）
  - 第二步：统计标准化（EWMA Z-score，消除波动率影响）

---

## 3️⃣ regime/liquidity.py（生死开关）

```python
class LiquidityRegime:
    def update(self, orderbook):
        ...

    def tradable(self) -> bool
```

规则完全放这里，不散落在策略中。

---

## 3️⃣+1️⃣ regime/regime_detector.py（新增：市场状态检测器）

```python
class RegimeDetector:
    def __init__(self, liquidity_window=1000, volatility_window=1000,
                 liquidity_q25=None, volatility_q90=None):
        """
        市场状态检测器
        
        将连续的市场指标转换为离散的状态标签，供权重调整使用
        """
    
    def update_liquidity(self, depth: float):
        """更新流动性历史"""
    
    def update_volatility(self, volatility: float):
        """更新波动率历史"""
    
    def detect_liquidity_regime(self) -> str:
        """
        检测流动性状态
        
        Returns:
            'low' | 'normal' | 'high'
        """
    
    def detect_volatility_regime(self) -> str:
        """
        检测波动率状态
        
        Returns:
            'low' | 'normal' | 'high'
        """
    
    def detect_extreme_state(self) -> bool:
        """
        检测极端状态（低流动性 + 高波动率）
        """
    
    def get_regime(self) -> dict:
        """
        获取完整市场状态
        
        Returns:
            {
                'liquidity': 'low' | 'normal' | 'high',
                'volatility': 'low' | 'normal' | 'high',
                'extreme': bool
            }
        """
```

📌 **用途**：为 AlphaCombiner 提供状态信息，用于动态权重调整

---

## 4️⃣ signals/leadlag.py（方向来源）

```python
class LeadLagSignal:
    def __init__(self, lead_symbol, lag_symbol, time_window_ms=100, 
                 use_sign=True, use_weighted=False):
        """
        Args:
            lead_symbol: Lead 资产符号（如 "BTC-PERP"）
            lag_symbol: Lag 资产符号（如 "ETH-PERP"）
            time_window_ms: 时间窗口（毫秒）
            use_sign: 是否使用 sign()（默认 True，推荐）
            use_weighted: 是否使用加权版本（可选增强）
        """
    
    def update(self, lead_mid, lag_mid):
        """更新价格"""
    
    def compute(self) -> int:
        """
        计算 Lead-Lag 信号
        
        返回 -1 / 0 / +1（标准版本）
        或 float（加权版本）
        """
    
    def get_signal_info(self) -> dict:
        """
        获取信号信息（用于监控和调试）
        
        Returns:
            {
                'signal': int/float,
                'delta': float,
                'method': 'sign' | 'weighted' | 'raw',
                'window_ms': int
            }
        """
```

📌 **强制要求**：

* Lead 和 Lag 必须不同 symbol
* 时间窗口 < 300ms

📌 **设计选择说明**：

* **为什么使用 sign()？**
  - 在高频尺度（50-200ms），方向信号比幅度信号更稳定
  - 幅度信息已在 OFI 和 AV 中体现
  - 信号职责分离：Lead-Lag 提供方向，OFI 提供强度

* **可选增强版本**：
  - 使用 `use_weighted=True` 启用加权版本
  - 公式：`sign(Δp) * min(1.0, |Δp| / σ)`
  - 保留幅度信息但标准化

---

## 5️⃣ alpha/combiner.py（灵魂模块）

```python
class AlphaCombiner:
    def __init__(self, base_weights: dict = None):
        """
        Args:
            base_weights: 基础权重配置
                {
                    'leadlag': 0.5,
                    'ofi': 0.3,
                    'av': 0.2
                }
        """
    
    def compute_weights(self, regime: dict) -> dict:
        """
        基于市场状态计算动态权重
        
        Args:
            regime: 市场状态字典
                {
                    'liquidity': 'low' | 'normal' | 'high',
                    'volatility': 'low' | 'normal' | 'high',
                    'extreme': bool
                }
        
        Returns:
            调整后的权重字典
        """
    
    def compute(self, signals: dict, regime: dict) -> float:
        """
        计算 Alpha 值
        
        Args:
            signals: 信号字典
                {
                    'leadlag': float,  # -1, 0, 1
                    'ofi': float,      # 标准化后的 OFI
                    'av': float        # 标准化后的 Aggressive Volume
                }
            regime: 市场状态字典
        
        Returns:
            Alpha 值，范围 [-1, 1]
        """
    
    def get_current_weights(self, regime: dict) -> dict:
        """
        获取当前状态下的权重（用于监控和调试）
        """
```

📌 **重要修正**：

* **原设计问题**：固定权重违反时间尺度一致性
* **修正方案**：基于市场状态的动态权重调整
  - 高波动率 → 降低 Lead-Lag 权重（慢信号）
  - 低流动性 → 降低 OFI 权重（易受 spoof）
  - 极端状态 → 降低所有信号权重

📌 这里 **不允许下单**
这里只输出 alpha（-1 ~ +1）

---

## 6️⃣ execution/router.py（执行与 OFI 的交汇点）

```python
class ExecutionRouter:
    def route(self, alpha, ofi, position_state):
        """
        决定：
        - 下不下
        - 用什么订单
        - 下多大
        """
```

典型策略：

```python
if ofi > HIGH:
    IOC
elif ofi > MID:
    post_only
else:
    skip
```

---

## 7️⃣ risk/kill_switch.py（必须有）

```python
class KillSwitch:
    def check(self, state) -> bool:
        """
        True → kill trading
        """
```

触发条件：

* 延迟异常
* 深度消失
* 连续滑点

---

## 8️⃣ feedback/trust_score.py（高级但关键）

```python
class SignalTrust:
    def __init__(self, alpha=0.95, min_trust=0.1, max_trust=1.0,
                 expected_slippage=0.001, min_samples=10):
        """
        Args:
            alpha: EMA 衰减因子（越大，历史权重越高）
            min_trust: 最小信任分数（防止权重降为 0）
            max_trust: 最大信任分数
            expected_slippage: 预期滑点（用于计算质量分数）
            min_samples: 最小样本数（低于此值不更新）
        """
    
    def compute_quality(self, slippage: float, fill_rate: float,
                       expected_slippage: float = None) -> float:
        """
        计算信号质量分数
        
        公式：
        quality = fill_rate - 0.5 * max(0, (slippage - expected) / expected)
        
        Returns:
            质量分数，范围 [0, 1]
        """
    
    def update(self, signal_name: str, slippage: float, fill_rate: float,
              expected_slippage: float = None) -> float:
        """
        更新信号信任分数
        
        数学形式：
        trust_t = α * trust_{t-1} + (1-α) * quality_t
        
        Args:
            signal_name: 信号名称（'ofi', 'leadlag', 'av'）
            slippage: 实际滑点
            fill_rate: 成交率
            expected_slippage: 预期滑点
        
        Returns:
            更新后的信任分数
        """
    
    def get_weight_adjustment(self, signal_name: str) -> float:
        """
        获取权重调整因子（用于 AlphaCombiner）
        
        Returns:
            权重调整因子，范围 [min_trust, max_trust]
        """
    
    def get_trust_report(self) -> dict:
        """
        获取信任分数报告（用于监控和调试）
        """
```

📌 **重要修正**：

* **原设计问题**：反馈机制未形式化，更新规则不明确
* **修正方案**：使用 EMA 更新的数学形式化
  - 数学公式：`trust_t = α * trust_{t-1} + (1-α) * quality_t`
  - 质量分数：`quality = fill_rate - slippage_penalty`
  - 噪声处理：EMA 平滑 + 最小样本数约束

📌 **这是系统"自我学习"的入口**

---

# 四、主引擎流程（main loop）

```python
on_ws_event(event):
    update_orderbook(event)
    update_trades(event)

    # 更新状态检测器
    depth = orderbook.depth(levels=5)
    volatility = market_state.get_volatility()
    regime_detector.update_liquidity(depth)
    regime_detector.update_volatility(volatility)
    
    # 获取市场状态
    regime = regime_detector.get_regime()
    
    # 检查是否可交易
    if not regimes.tradable():
        return

    # 计算信号
    signals = {
        'leadlag': leadlag_signal.compute(),
        'ofi': ofi_calculator.get_scaled(),
        'av': aggressive_volume.compute()
    }
    
    # 使用动态权重计算 Alpha
    alpha = combiner.compute(signals, regime)
    
    # 应用信任分数调整（如果启用）
    if trust_manager:
        weights = combiner.compute_weights(regime)
        for signal_name in weights:
            trust_factor = trust_manager.get_weight_adjustment(signal_name)
            weights[signal_name] *= trust_factor
        # 重新计算 alpha
        alpha_raw = sum(weights[k] * signals[k] for k in signals)
        alpha = tanh(alpha_raw)

    if abs(alpha) < alpha_min:
        return

    execution.route(alpha, signals['ofi'], position)
    
# 订单成交回调：更新信任分数
on_order_filled(order_id, fill_info):
    signal_name = fill_info['signal_name']
    slippage = fill_info['slippage']
    fill_rate = fill_info['fill_rate']
    expected_slippage = fill_info.get('expected_slippage', 0.001)
    
    trust_manager.update(signal_name, slippage, fill_rate, expected_slippage)
```

📌 **注意**：

* 没有 while True
* 没有 sleep
* 全是事件触发

---

# 五、这套架构的"隐含优势"

✅ OFI 不会污染策略逻辑
✅ Lead-Lag 可随时替换
✅ Execution 可单独升级
✅ 风控能"一键拉闸"
✅ 可逐步加 ML / Online learning

---

# 六、现实建议（非常重要）

如果你现在：

* 👤 **一个人** → 先实现 `data + ofi + execution`
* 👥 **小团队** → 严格按模块拆
* 🚀 **准备实盘** → 先只做 BTC→ETH

---

**文档版本**：v2.0（修正版）
**最后更新**：2024年
**状态**：✅ 所有评估意见已吸收并整合进接口定义
