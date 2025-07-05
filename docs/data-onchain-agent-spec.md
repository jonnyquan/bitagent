# DataOnchainAgent Specification

> Status: Draft v0.1 – 2025-06-29  
> Related: `data-agents-spec.md`, `feature-builder-spec.md`

---

## 0  Purpose
提供 BTC Exchange Netflow、Dormancy、MVRV 等链上指标，支撑中期策略与宏观判断。

---

## 1  Data Sources (Glassnode)
| Metric | API Parameter | Refresh | Note |
|--------|---------------|---------|------|
| Net Transfer Volume from/to Exchanges (BTC) | `net_transfers_volume_(btc)` | 4 h | sign of accumulation |
| Dormancy | `entity-adjusted_dormancy` | 1 d | holder activity |
| MVRV Z-Score | `mvrv_z_score` | 1 d | valuation | 

---

## 2  Message Schema
Topic `data.onchain`
```jsonc
{
  "ts": 1727486400,
  "netflow_std": -1.2,
  "dormancy": 0.45,
  "mvrv": 1.8
}
```
*`netflow_std`* 为 365 d z-score。

---

## 3  Scheduling
* 4 h Cron `0 */4 * * *`。

---

## 4  Metrics
`onchain_api_fail_total`, `netflow_std` Gauge。

---

## 5  Tasks
| ID | Task | ETA |
|----|------|-----|
| OC-1 | Agent skeleton + Glassnode client | W2D1 |
| OC-2 | Z-score calculation util | W2D1 |
| OC-3 | Prom metrics | W2D1 |

---

*End of spec.* 