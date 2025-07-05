# Strategy Spec – Funding Rate Arbitrage v2 (fr_arb_v2)

> Status: **Implemented v1.1** – 2025-07-01  
> Related: `strategy-agents-spec.md`, `risk-proxy-spec.md`

---

## 0  Idea
Utilize the spread between *perpetual contract Funding* and *spot* markets. If the annualized funding APR exceeds the trading cost + target profit threshold, build a **buy spot / sell perpetual** position; otherwise, short spot + buy perpetual.

---

## 1  Inputs
| Feature | Source | Note |
|---------|--------|------|
| `funding_apr` | FeatureBuilder | 3×8h rate ×365 |
| `price_close` | FeatureBuilder | spot price |
| `sentiment_score` | FeatureBuilder | used to filter bearish sentiment |
| Depth 10bps | DataDepthAgent (future) | determines maximum notional |

---

## 2  Signal Rules
1. **Entry Long Basis (buy spot / sell perp)**
   • `funding_apr` > 15 %  &  `sentiment_score` ≥ -0.2.
2. **Entry Short Basis**
   • `funding_apr` < -10 % &  `sentiment_score` ≤ 0.2.
3. **Exit** when `funding_apr` returns to ±5 % band or after 3 days.

---

## 3  TradePlan Output
```jsonc
{
  "strategy":"fr_arb_v2",
  "side":"buy_spot_sell_perp",
  "size": 50000,
  "confidence": 0.75,
  "edge": 0.003,
  "ts": 1727486400
}
```
`edge = funding_apr - 5% buffer`.

---

## 4  Parameters
```yaml
min_apr_long: 0.15
min_apr_short: -0.10
cooldown_min: 45
max_notional_usd: 150000
ai_gate:
  enabled: true
  min_conf: 0.65
```

---

## 5  Metrics
`frarb_active_positions`, `frarb_edge` Gauge; `frarb_signal_total` Counter.

---

## 6  Tasks
| ID | Task | ETA |
|----|------|-----|
| ST-FR-1 | Implement StrategyAgent subclass | W2D2 |
| ST-FR-2 | Unit tests rule triggers | W2D2 |
| ST-FR-3 | Integration with RiskProxy | W2D3 |

---

*End of spec.*