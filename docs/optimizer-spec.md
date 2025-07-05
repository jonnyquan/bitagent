# Optimizer Specification – Portfolio Risk Parity & Semi-Variance Min

> Status: Draft v0.1 – 2025-06-28  
> Author: AI Trading Architecture Assistant  
> Related Docs: `strategy-advanced-roadmap.md`, `macro-derivatives-data-spec.md`, `trading-system-detailed-plan.md`

---

## 0. Purpose

为多策略组合 (FR-ARB, TF-FUT, VOL-BO, BASIS-ROT) 计算 **动态权重 & 杠杆上限**，目标：
1. 达到或超过 **20 % 年化目标收益**。  
2. 在相同收益水平下最小化 **下行半方差 (Semi-Variance, SVaR)**。  
3. 考虑宏观 Regime 与杠杆限制，输出可直接用于 RiskManager/Alert-Only 调仓提醒。

---

## 1. Input Data

| Source | Table / Key | Frequency | Fields |
|--------|-------------|-----------|--------|
| Strategy PnL | `pnl_daily` | daily EoD | `ts,strategy,ret` (% of equity) |
| Macro Regime | Redis `macro.regime` | realtime | `on / neutral / off` |
| Leverage Cap | Redis `macro.leverage_cap` | realtime | float |
| Risk Metrics | Prom `strategy_var95` | daily | 1-day VaR |

*Rolling window*: last **60** trading days (≈ three months)。  
*Return series* 用 **权益百分比 (∆Equity / Equity_prev)** 归一化。

---

## 2. Optimisation Model

### 2.1 Variables
`w_i` – weight of strategy *i*, `i ∈ {FR,TF,VOL,BASIS}`  (sum ≤ 1.5).  
`L` – global leverage multiplier, `0 < L ≤ leverage_cap` (from macro)。

### 2.2 Objective
Minimise **Semi-Variance** of portfolio returns subject to expected return ≥ target.

Minimise  \( SV = \frac{1}{N} \sum_{t=1}^N \max(0,   \mu_p - r_{p,t})^2 \)  
subject to
* \( \mu_p =  \sum_i w_i \bar{r_i}  \geq  0.00055 \)  (≈20 %/yr ≃ 0.055 %/day)  
* \( 0.05 ≤ w_i ≤ 0.60 \)  
* \( w_{FR} + w_{BASIS} ≥ 0.30 \)  
* \( \sum_i w_i  =  L \leq leverage_cap \)

### 2.3 Solver
* cvxpy (`ECOS` primary, fallback `SCS`).  
* If solver fails or returns infeasible, fallback to **equal-risk-contribution** heuristic。

### 2.4 Risk-OFF Adjustments
* If `macro.regime == 'off'` → set `target_return = 0.0004` (≈ 14 % p.a.) & `leverage_cap ×= 0.8`.  
* If 7-day rolling MaxDD > 3 % → shrink `w_TF` & `w_VOL` each −10 % absolute, redistribute proportionally。

---

## 3. Rebalance Logic

| Trigger | Action |
|---------|--------|
| **Weekly** – Monday 08:00 UTC | Run optimiser, write `alloc.next` Redis hash |
| `macro.regime` changes | Immediate run; output flagged `urgent=1` |
| `portfolio_var95` > 25 % equity | Emergency reduce: scale all `w_i` by 0.8, skip optimiser |

调仓输出：
```json
{
  "timestamp": "2025-06-30T08:00:01Z",
  "weights": {"FR":0.45,"TF":0.20,"VOL":0.10,"BASIS":0.25},
  "leverage": 1.35,
  "macro_regime": "neutral",
  "urgent": 0
}
```
存储 MySQL `alloc_history` & Redis `alloc.current`；推送 `allocation_alert`。

---

## 4. Prometheus Metrics
| Metric | Type | Description |
|--------|------|-------------|
| `optimizer_run_latency_ms` | Histogram | solve time |
| `portfolio_semivar` | Gauge | current semi-variance |
| `portfolio_target_return` | Gauge | target return p.d. |
| `portfolio_leverage` | Gauge | current leverage output |

Alerts: `optimizer_failure_total` Counter >0 within 12 h → PagerDuty。

---

## 5. Implementation Tasks (OP-series)
| ID | Task | Depends | Deadline |
|----|------|---------|----------|
| OP-1 | `risk_parity_optimizer.py` – cvxpy model & CLI | data-layer | W3D1 |
| OP-2 | `optimizer_runner.py` – cron wrapper, Redis & Prom push | OP-1 | W3D2 |
| OP-3 | Unit tests: objective value, constraints, infeasible fallback | OP-1 | W3D2 |
| OP-4 | Grafana dashboard & alert rules | OP-2 | W3D3 |
| OP-5 | Integrate with AlertDispatcher – push `allocation_alert` | OP-2 | W3D3 |

---

## 6. Acceptance Criteria
1. Solver返回时间 p95 < 500 ms；若故障回退到 heuristic。  
2. 连续 4 周回测：Sharpe ≥ 1.85；MaxDD ≤ baseline-1 pp。  
3. Prometheus 指标 & Alert 呈现完整；Telegram 通知包含重平衡摘要。

---

## 7. Implementation Blueprint (Developers Reference)

### 7.1 Module Breakdown
| Module | Path | Responsibility |
|--------|------|----------------|
| DataLoader | `src/optimizer/data_loader.py` | Pull `pnl_daily`, redis keys, Prom metrics; preprocess returns DF |
| OptimizerCore | `src/optimizer/semivar_optimizer.py` | cvxpy model & heuristic fallback |
| Runner | `src/optimizer/optimizer_runner.py` | Schedule execution, persistence, alert dispatch |
| Metrics | `src/optimizer/metrics.py` | Prometheus push helpers |

#### 7.1.1 `SemiVarOptimizer` Skeleton
```python
class SemiVarOptimizer:
    def __init__(self, returns_df: pd.DataFrame, target_mu: float, leverage_cap: float,
                 lower: float = 0.05, upper: float = 0.60):
        self.r = returns_df              # shape (N, k)
        self.N, self.k = self.r.shape
        self.target = target_mu
        self.cap = leverage_cap
        self.lower, self.upper = lower, upper

    def _solve_cvxpy(self):
        import cvxpy as cp
        w = cp.Variable(self.k)
        mu_p = cp.sum(cp.matmul(self.r.mean(axis=0), w))
        u   = cp.pos(mu_p - self.r @ w)   # semi-variance auxiliaries
        obj = cp.Minimize(cp.sum_squares(u) / self.N)
        constraints = [w >= self.lower, w <= self.upper,
                       cp.sum(w) <= self.cap,
                       mu_p >= self.target,
                       w[0] + w[3] >= 0.30]   # FR + BASIS index positions
        prob = cp.Problem(obj, constraints)
        prob.solve(solver=cp.ECOS, warm_start=True)
        if prob.status not in cp.settings.SOLUTION_PRESENT:
            raise ValueError("Infeasible")
        return dict(zip(self.r.columns, w.value))

    def _heuristic(self):
        vol = self.r.std()
        inv = 1/vol
        w = inv / inv.sum()
        # clip & rescale to leverage cap
        w = np.clip(w, self.lower, self.upper)
        factor = min(self.cap / w.sum(), 1.0)
        return dict(zip(self.r.columns, (w*factor).tolist()))

    def solve(self):
        try:
            return self._solve_cvxpy()
        except Exception as e:
            logger.warning("optimizer fallback %s", e)
            return self._heuristic()
```

### 7.2 Runner Flow
```
load returns -> get regime/leverage -> choose target_mu
opt = SemiVarOptimizer(...)
weights = opt.solve()
alloc = {timestamp, weights, leverage=sum(weights.values()), macro_regime, urgent}
redis.hmset('alloc.current', alloc)
mysql.insert('alloc_history', alloc)
alert_dispatcher.push('allocation_alert', alloc)
metrics.push(alloc, latency)
```

*Cron*: K8s CronJob `0 8 * * 1` and Redis PubSub `macro.regime_changed`.

### 7.3 Unit & Integration Tests
| Test file | Focus |
|-----------|-------|
| `tests/optimizer/test_semivar_objective.py` | Check objective value vs. numpy calc |
| `tests/optimizer/test_infeasible_fallback.py` | Force infeasibility, expect heuristic |
| `tests/optimizer/test_runner_end_to_end.py` | Mock Redis/MySQL, run runner, validate writes |

### 7.4 Monitoring Dashboard
* Panels: optimizer latency, portfolio semivar, leverage, weight stack.  
* Alerts: `optimizer_failure_total > 0`, `latency_p95 > 1s`.

---

*End of Optimizer spec.* 