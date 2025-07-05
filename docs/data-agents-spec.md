# Data Agents Specification – Biteasy

> Status: **Implemented v1.1** – 2025-07-01
> Author: AI Trading Architecture Assistant  
> Related: `agent-system-overview.md`, old `data-layer-spec.md`

---

## 0  Overview

DataAgents are specialised collectors providing real-time & batch data to the Agent bus.  Each agent focuses on one domain and publishes standardised payloads.

| Agent | Domain | Source | Refresh | Topic |
|-------|--------|--------|---------|-------|
| `data_market` | Trades / Orderbook | Binance WS `aggTrade`, `bookTicker` | ≤250 ms | `data.tick`, `data.book` |
| `data_funding` | Funding / Basis | Binance REST + Calc | 1 min | `data.funding` |
| `data_sentiment` | Twitter & News | Twitter v2, NewsAPI | 5 min | `data.sentiment` |
| `data_macro` | Rates, VIX, CPI | FRED API | daily | `data.macro` |
| `data_onchain` | BTC Netflow | Glassnode | 4 h | `data.onchain` |

---

## 1  Message Schemas

### 1.1 `MarketDataTick`
```jsonc
{ "symbol":"BTCUSDT", "ts":1727486400, "price":64321.1, "qty":0.03 }
```

### 1.2 `FundingRate`
```jsonc
{ "symbol":"BTCUSDT", "ts":1727486400, "rate":0.00018 }
```

Full schemas defined in `src/agents/schemas.py` (Pydantic).

---

## 2  Feature Mapping

| Feature | From Agent | Description |
|---------|------------|-------------|
| `price_close` | data_market | last trade price at bar close |
| `funding_apr` | data_funding | rate * 3 × 365 |
| `sentiment_score` | data_sentiment | range −1…1 (FinBERT) |
| `macro_regime` | data_macro | bullish / neutral / risk-off |
| `netflow_std` | data_onchain | z-score of BTC exchange netflow |

FeatureBuilder in `core/feature_builder.py` joins latest payloads into single dict.

---

## 3  Rate Limits & Cost

| API | Free Quota | Cost Plan | Notes |
|-----|-----------|-----------|-------|
| Twitter v2 | 500 req/24h | $100/mo | tweets search |
| NewsAPI | 100 req/day | $449/mo enterprise | fetch headlines |
| Glassnode | 50 endpoints | $29/mo | on-chain metrics |

Budget ≤ $600 / month.

---

## 4  Implementation Milestones

| ID | Task | ETA |
|----|------|-----|
| DA-1 | Pydantic schemas | W1D1 |
| DA-2 | data_market WS consumer | W1D1 |
| DA-3 | funding poller | W1D1 |
| DA-4 | sentiment collector + FinBERT | W1D2 |
| DA-5 | macro & on-chain cron | W1D3 |

---

*End of DataAgents spec.* 