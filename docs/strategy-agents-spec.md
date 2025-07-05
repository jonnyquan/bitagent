# Strategy Agents Specification

> Status: Draft v0.1 – 2025-06-28  
> Related: old strategy-* specs, agent-system-overview.md

---

## 0  Purpose

StrategyAgents encapsulate specific trading logics (FR-ARB, TF-FUT, etc.) and output **TradePlan** messages for further processing.

---

## 1  TradePlan Schema

```jsonc
{
  "id": "uuid4",
  "strategy": "fr_arb_v2",
  "side": "long|short|buy_spot_sell_perp",
  "size": 12000,              // USD notional
  "mode": "intra|swing",
  "confidence": 0.78,
  "edge": 0.0021,             // expected return per notional
  "features": {"fr_apr":0.24,"vol_1h":0.018},
  "ts": 1727486400
}
```

---

## 2  Agent Lifecycle

1. **warm_up** – wait until all prerequisite DataAgents delivered last N bars.  
2. **active** – subscribe to `data.*` topics, maintain internal indicator state.  
3. **cooldown** – after N consecutive vetoes, pause self for T minutes.

---

## 3  Conflict Resolution

ChiefAgent aggregates multiple TradePlans:
* If two plans have same symbol opposite sides:
  • keep higher `edge × confidence`, drop the other.
* Size normalised by InvestorAgent later.

---

## 4  Parameters & AI Gate

Parameters loaded via YAML under `configs/strategy/`.  AI Gate can be enabled per strategy via
```yaml
ai_gate:
  enabled: true
  min_conf: 0.65
```

---

## 5  Implementation Milestones

| ID | Task | ETA |
|----|------|-----|
| SA-1 | Wrap fr_arb_v2 into Agent subclass | W1D1 |
| SA-2 | Wrap tf_fut_v2 | W1D1 |
| SA-3 | basis_rot & vol_bo | W1D2 |
| SA-4 | Conflict resolution helper | W1D2 |

---

*End of StrategyAgents spec.* 