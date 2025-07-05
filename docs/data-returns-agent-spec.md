# Data Returns Agent Specification

> Status: Draft v0.1 – 2025-06-29  
> Related: investor-agent-spec.md, risk-proxy-spec.md

---

## 0  Purpose

*Supply historical daily returns and real-time position snapshots to InvestorAgent & RiskProxy.*

---

## 1  Responsibilities

1. Query MySQL `pnl_daily` table every hour (or cron) → pivot to DataFrame → publish `data.returns`.  
2. Query latest open positions per strategy (REST / ExchangeAdapter) → publish `data.position`.  
3. Maintain in-memory cache for fast request/response (bus.request("data.returns", …)).

---

## 2  Bus Topics

| Topic | Payload | Frequency |
|-------|---------|-----------|
| `data.returns` | `{df:[{ts,strategy,ret},...]}` | hourly cron or on demand |
| `data.position` | `{strategy,pos_usd}` | on update (≤30s) |

Request/Response: `bus.request("data.returns", {})` → returns same payload.

---

## 3  Config Options

```yaml
agents:
  data_returns:
    entry_point: agents.data_returns.DataReturnsAgent
    schedule: "0 */1 * * *"  # hourly
    params:
      mysql_dsn: "mysql+aiomysql://user:pass@db/pnl"
      redis_url: "redis://localhost:6379/0"
      window_days: 90
```

---

## 4  Implementation Notes

* Use SQLAlchemy async & pandas for pivoting.  
* Cache latest returns DataFrame in self._cache; serve request quickly.  
* Positions fetched via ExchangeAdapter or DB view.  
* Include Prometheus counter `data_returns_queries_total`.

---

*End of spec.* 