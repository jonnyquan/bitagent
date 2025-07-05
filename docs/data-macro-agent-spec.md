# DataMacroAgent Specification

> Status: Draft v0.1 – 2025-06-29  
> Related: `data-agents-spec.md`, `feature-builder-spec.md`

---

## 0  Purpose
抓取宏观经济指标（US10Y, DXY, VIX, CPI YoY 等）并生成宏观风险状态，发布 `data.macro` 与 `data.macro_regime` 两类消息。

---

## 1  Data Sources
| Ticker | Source | Endpoint | Freq | Note |
|--------|--------|----------|------|------|
| US10Y Yield | FRED | `DGS10` | daily | 16:00 ET 发布 |
| DXY | Stooq API | `dxy` | 1 h | | 
| VIX | AlphaVantage | `VIX` | 5 min | free 25 req/day |
| CPI YoY | FRED | `CPIAUCSL` | monthly | | 

---

## 2  Macro Regime Logic
* **Risk-On**：US10Y < 3.5% AND VIX < 18 AND DXY 下跌趋势。  
* **Neutral**：介于两者之间。  
* **Risk-Off**：US10Y > 4.2% OR VIX > 25。

发布：
```jsonc
{ "ts": 1727486400, "regime": "off", "teny": 4.35, "vix": 26.7, "dxy": 106.2 }
```
Topic: `data.macro_regime`

---

## 3  Scheduling & Caching
* FRED 指标：每日 00:30 UTC 拉取昨日值。  
* VIX/DXY：每 30 min。  
* SQLite 缓存上一条值，避免重复发布。

---

## 4  Metrics
`macro_regime` Gauge 0,1,2 (on/neutral/off) ；`fred_request_fail_total` Counter。

---

## 5  Tasks
| ID | Task | ETA |
|----|------|-----|
| DM-1 | Agent skeleton + regime calc | W2D1 |
| DM-2 | Prom metrics | W2D1 |
| DM-3 | Unit tests thresholds | W2D2 |

---

*End of spec.* 