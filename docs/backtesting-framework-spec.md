# Backtesting Framework Specification

> Status: Draft v0.1 – 2025-06-29  
> Author: AI Trading Architecture Assistant  
> Scope: Historical & intraday event-driven backtesting engine for Biteasy strategies & agents.

---

## 0  Goals

1. Support **vectorised daily** backtests for rapid parameter search.  
2. Support **event-driven intraday** simulation (orderbook, websockets) for latency-aware logic.  
3. Re-use existing **Agent** classes by plugging Bus into simulated clock.  
4. Produce benchmark metrics (CAGR, Sharpe, Sortino, maxDD, turnover, risk stats).  
5. Portable outputs: Parquet trades, JSON metrics, HTML report.

---

## 1  Architecture

```text
       DataLoader --->  ReplayClock   --->  Bus   --->  Agents (Strategy/Risk/Investor)
                           ^                             |
MarketEvents CSV/Parquet ---+                             |
                           |                              v
                      MetricsCollector <--- TradeExecutor (sim)
```
* `ReplayClock` drives virtual time; publishes `bus.publish("clock.tick", {ts})`.
* `DataLoader` streams historical market events / funding / sentiment as bus topics (`data.*`).
* Existing RiskProxy / InvestorAgent reused with minimal patch.
* `TradeExecutor` converts TradePlan → fills; updates PnL & positions.
* `MetricsCollector` aggregates performance and writes outputs.

---

## 2  Simulation Modes

| Mode | Granularity | Use-case |
|------|-------------|----------|
| **vector** | daily OHLCV dataframe | fast grid search (1000s runs) |
| **event**  | tick or 1-s book snapshots | latency / slippage study |
| **walk-forward** | rolling re-fit InvestorAgent every N days | realistic evaluation |

---

## 3  Config Example (YAML)

```yaml
mode: event
start: "2024-01-01"
end: "2024-04-01"
symbols: [BTCUSDT]
strategies:
  - fr_arb_v2
  - tf_fut_v2
initial_equity: 1_000_000
slippage_bps: 2
commission_bps: 1
``` 

---

## 4  Metrics
* `perf.equity_curve` – equity vs time.  
* `perf.drawdown` – rolling maxDD.  
* `perf.turnover_daily` – % equity.  
* `risk.exposure` – gross/net.  
* `risk.leverage` – intraday max.

---

## 5  Implementation Milestones

| ID | Task | ETA |
|----|------|-----|
| BT-1 | ReplayClock & DataLoader skeleton | W1D2 |
| BT-2 | TradeExecutor & MetricsCollector | W1D3 |
| BT-3 | CLI `python backtest.py config.yaml` | W1D4 |
| BT-4 | HTML report template | W1D5 |

---

*End of backtesting spec.* 