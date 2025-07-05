# Investor Agent Specification

> Status: Draft v0.2 – 2025-06-29  
> Related: optimizer-spec.md, risk-proxy-spec.md

---

## 0  Objective

InvestorAgent consumes approved `TradePlan`s and decides capital allocation & leverage, producing **PortfolioAllocation** message.

---

## 1  Inputs

* TradePlans list (post RiskAgent)  
* Current portfolio positions (DataAgent-Portfolio)  
* Leverage cap from RiskAgent  
* Macro regime

---

## 2  Algorithm Stack

1. **Semi-Variance Optimizer** (existing) – base weights.  
2. **RL Overlay** – PPO agent adjusts weights using historical simulation reward.  
3. Fallback: Inverse-vol heuristic.

---

## 3  PortfolioAllocation Schema

```jsonc
{
  "ts": 1727486400,
  "weights": {"fr_arb_v2":0.4,"tf_fut_v2":0.3,...},
  "leverage": 1.2,
  "notes": "sv_opt + rl adj"
}
```

---

## 4  Constraints

* `sum w ≤ leverage_cap`  
* each `w_i` between strategy min/max bounds.  
* daily turnover ≤ 200 % of equity.

---

## 5  Implementation Milestones

| ID | Task | ETA |
|----|------|-----|
| IA-1 | Wrap `semivar_optimizer` into InvestorAgent | ✔︎ Done (tempcode) |
| IA-2 | RL overlay integration (stable-baselines3 PPO) | W1D3 |
| IA-3 | Metrics export & documentation | W1D3 |
| IA-4 | Unit tests & backtest harness | W1D4 |

---

## 6  Prometheus Metrics (NEW)

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `investor_turnover` | gauge | None | Daily turnover ratio vs equity |
| `investor_leverage` | gauge | None | Portfolio leverage decided |
| `investor_rl_reward` | gauge | None | Latest RL overlay reward |

---

## 7  YAML Sample Rules (NEW)

```yaml
agents:
  investor:
    schedule: "0 */1 * * *"   # hourly cron
    params:
      min_weight: 0.01
      max_weight: 0.5
      default_leverage: 1.0
```

---

*End of Investor spec.* 