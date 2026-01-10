好，这一步我们**直接进入“可实盘工程”的层级**。
下面给你的是一套 **真实可维护、可扩展、符合 Hyperliquid 特性的实盘工程目录 + 模块接口定义**。
不是 demo，而是**量化团队常见的最小可长期演化架构（Minimum Viable Trading System）**。

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
│   └── liquidation.py          # 极端 / 清算检测
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

下面我**只给“关键模块”的接口定义**，这些决定了你系统的上限。

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
    def scale(self, ofi, depth, volatility) -> float
```

📌 **注意**：

* OFI 只在 L1 变化时更新
* 不和 trade 混用

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

## 4️⃣ signals/leadlag.py（方向来源）

```python
class LeadLagSignal:
    def update(self, lead_mid, lag_mid) -> int:
        """
        返回 -1 / 0 / +1
        """
```

📌 **强制要求**：

* Lead 和 Lag 必须不同 symbol
* 时间窗口 < 300ms

---

## 5️⃣ alpha/combiner.py（灵魂模块）

```python
class AlphaCombiner:
    def compute(self, signals: dict, regime: dict) -> float:
        """
        signals = {
          'leadlag': float,
          'ofi': float,
          'av': float
        }
        """
```

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
    def update(self, ofi, fill, slippage):
        ...

    def weight(self) -> float
```

📌 **这是系统“自我学习”的入口**

---

# 四、主引擎流程（main loop）

```python
on_ws_event(event):
    update_orderbook(event)
    update_trades(event)

    update_regimes()
    if not regimes.tradable():
        return

    signals = compute_signals()
    alpha = combiner.compute(signals, regimes)

    if abs(alpha) < alpha_min:
        return

    execution.route(alpha, signals['ofi'], position)
```

📌 **注意**：

* 没有 while True
* 没有 sleep
* 全是事件触发

---

# 五、这套架构的“隐含优势”

✅ OFI 不会污染策略逻辑
✅ Lead-Lag 可随时替换
✅ Execution 可单独升级
✅ 风控能“一键拉闸”
✅ 可逐步加 ML / Online learning

---

# 六、现实建议（非常重要）

如果你现在：

* 👤 **一个人** → 先实现 `data + ofi + execution`
* 👥 **小团队** → 严格按模块拆
* 🚀 **准备实盘** → 先只做 BTC→ETH


