# Risk Proxy Specification – Biteasy Trading System

> Status: **Partially Implemented v1.1** – 2025-07-01  
> **Implementation Note:** The current implementation uses an event-driven `RiskProxyAgent` that subscribes to signals, which differs from the in-process `check()` call described below but fits the overall agent architecture. The core logic of rule validation remains the same.  
> Author: AI Trading Architecture Assistant  
> Scope: Real-time runtime risk guard for Alert-Only mode (future auto-trade compatible)  
> Related Docs: `execution-risk-framework-spec.md`, `strategy-runner-spec.md`, `optimizer-spec.md`

---

## 0  Purpose & Goals

The *Risk Proxy* acts as the **first line of defence** between Strategy logic and downstream order / alert generation.  It enforces **per-strategy & global risk budgets** and performs **contextual checks** (market depth, leverage caps, cooldowns) before a TradeAlert is emitted.

Goals:

1. Unified **async interface** exposed to strategies (`depth_ok`, `cooldown_ok`, `pos_limit_ok`).
2. Centralised **real-time metrics cache** (Redis) updated by background workers.
3. Pluggable **risk rules DSL** to evolve without code deploys.
4. Low-latency (<1 ms) evaluation path to avoid blocking strategy loop.

---

## 1  High-Level Architecture

```text
┌────────────┐  check()  ┌─────────────┐   Redis Pub/Sub   ┌───────────────┐
│ Strategy    │─────────▶│  RiskProxy  │◀──────────────────│  Risk Updater │
└────────────┘           └─────┬───────┘                   └───────────────┘
                               │ pass/fail
                               ▼
                          TradeAlert flow
```

* `RiskProxy` runs **in-process** with Strategy Runner; it holds an in-memory snapshot of risk parameters refreshed via Redis pub/sub (`risk.update.*`).
* `Risk Updater` is a separate async task (or microservice) pulling:
  – Orderbook snapshots → depth stats (`depth_usd@10bps`).  
  – Position data from ExchangeAdapter.  
  – Macro regime data, leverage caps.

---

## 2  Risk Dimensions & Rules

| ID | Dimension | Rule Example | Param Source |
|----|-----------|--------------|--------------|
| R-1 | Market Depth | `notional <= depth_usd@10bps * 0.1` | WS orderbook snapshot |
| R-2 | Leverage Cap | `abs(gross_exposure) ≤ account_equity * leverage_cap` | Macro regime Redis key |
| R-3 | Position Limit | `strategy_pos_usd ≤ pos_limit_usd` | Redis `pos.limit.{strategy}` |
| R-4 | Cooldown | `last_alert_ts + 30s < now` | Redis stream metadata |
| R-5 | Daily Loss | `realised_pnl_today ≥ -max_daily_loss` | MySQL PnL table |

Risk rules are defined in **YAML DSL**:

```yaml
rules:
  fr_arb_v2:
    depth_pct: 0.10   # R-1
    cooldown_sec: 45  # R-4
    pos_usd_limit: 400000 # R-3
  default:
    leverage_cap: "macro.leverage_cap" # R-2
```

---

## 3  Public Interface (Async)

```python
class RiskProxy(Protocol):
    async def depth_ok(self, symbol: str, notional: float) -> bool: ...
    async def cooldown_ok(self, strategy: str) -> bool: ...
    async def pos_limit_ok(self, strategy: str, delta_usd: float) -> bool: ...
    async def all_ok(self, *, symbol: str, strategy: str, notional: float) -> bool: ...
```

Return `True` if **all** active rules pass.  Strategies may bypass certain checks (e.g. `depth_ok=False` for TWAP child orders).

---

## 4  Data Refresh & Caching

* Redis hash `risk.snapshot` keeps latest values; Lua script ensures atomic update.
* Pub/Sub channel `risk.updated` notifies in-process RiskProxy to refresh local cache.
* Full refresh interval 1 s; critical keys (depth, leverage) 200 ms.

---

## 5  Prometheus Metrics

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `risk_checks_total` | counter | strategy,rule,result | Total checks |
| `risk_depth_ratio` | gauge | symbol | notional / available_depth |
| `risk_cooldown_sec` | gauge | strategy | seconds since last alert |
| `risk_violations_total` | counter | strategy,rule | Rejected alerts |

---

## 6  Error Handling & Fail-safe

* If **depth snapshot stale** (>2 s) → reject with `STALE_DEPTH`.
* If Redis unavailable → RiskProxy switches to *fail-closed* mode (reject all) after 3 s gap.
* All rejections raise `RiskViolation` with `.reason`;
  Strategy Runner logs & increments metrics.

---

## 7  Implementation Milestones

| ID | Task | Depends | ETA |
|----|------|---------|-----|
| RP-1 | `risk_proxy.py` in-proc impl | SR-1 | W1D2 |
| RP-2 | `risk_updater.py` depth & pos fetcher | EX-2 | W1D3 |
| RP-3 | YAML rules loader + hot reload | RP-1 | W1D3 |
| RP-4 | Prometheus metrics export | RP-1 | W1D3 |
| RP-5 | Unit tests | RP-1..3 | W1D4 |

---

*End of risk-proxy spec.* 