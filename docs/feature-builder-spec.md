# FeatureBuilder Specification

> Status: Draft v0.1 – 2025-06-29  
> Related: `data-agents-spec.md`, `strategy-agents-spec.md`

---

## 0  Purpose
将来自多 DataAgents 的原始 payload 转换为统一、时序对齐的 FeatureDict，用于策略计算与回测。

---

## 1  Inputs & Outputs
| Source Topic | Msg | Notes |
|--------------|-----|-------|
| `data.tick` | `MarketDataTick` | 实时价格 (≤250 ms) |
| `data.funding` | `FundingRate` | 1 min funding |
| `data.sentiment` | `SentimentScore` | 5 min NLP 情绪 |
| _(extensible)_ | `data.macro`, `data.metrics` | | 

Publish topic **`data.features`**:
```jsonc
{
  "ts": 1727486400,
  "symbol": "BTCUSDT",
  "price_close": 64321.1,
  "funding_apr": 0.125,
  "sentiment_score": 0.34,
  "lag": 180          // ms max lag among inputs
}
```

---

## 2  Alignment Logic
1. Store latest payload per topic in dict `latest[topic]`.  
2. 当 `latest` 同时包含 tick+funding+sentiment 时：
   • 取各 ts 最大值 → `ts_out`；计算 `lag = ts_out - min(ts_i)`。  
3. 生成 FeatureDict，缺失字段填 `null`，若 lag > 3 s 则标记 `stale=1`。

---

## 3  Performance
* 预期发布频率 ~1 msg/sec。  
* 内存占用 < 1 MB，只保留最后一条。

---

## 4  Metrics
| Metric | Type | Description |
|--------|------|-------------|
| `feature_publish_total` | Counter | 总输出条数 |
| `feature_lag_ms` | Histogram | 输入最大滞后 |

---

## 5  Implementation Tasks
| ID | Task | ETA |
|----|------|-----|
| FB-1 | `feature_builder.py` subscribe & merge | done W2D1 |
| FB-2 | Prometheus metrics | W2D1 |
| FB-3 | Unit test lag calc | W2D2 |

---

*End of spec.* 