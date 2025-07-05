# StrategyAgent Base Specification

> Status: Draft v0.1 – 2025-06-29  
> Related: `strategy-agents-spec.md`

---

## 0  Responsibilities
1. 订阅 `data.features`，维护指标窗口。  
2. 根据特定规则生成 `TradePlan` 并发布 `strategy.plan`。  
3. 支持 **warm_up / cooldown / active** 状态机。  
4. 内置 AI Gate 前置过滤（可选）。

---

## 1  Abstract Interface
```python
class StrategyAgent(BaseAgent):
    name: str
    warmup_bars: int = 60
    cooldown_sec: int = 60

    async def on_bar(self, feat: FeatureDict): ...  # per incoming bar
    def generate_plan(self) -> Optional[TradePlan]: ...
```

Framework 调用顺序：
1. 收到 features → push to deque → if len>=warmup → `on_bar()`。  
2. 若 `generate_plan()` 返回值 → 发布 Bus。

---

## 2  State Machine
| State | Entry | Exit |
|-------|-------|------|
| warm_up | bars < warmup_bars | → active |
| active | default | on veto → cooldown |
| cooldown | `cooldown_sec` | timer结束 → active |

---

## 3  Prometheus Metrics
`strategy_signal_total{strategy}`, `strategy_cooldown_sec`, `strategy_state` Gauge。

---

## 4  Tasks
| ID | Task | ETA |
|----|------|-----|
| SB-1 | Implement base class in `strategies/base.py` | W2D2 |
| SB-2 | Metrics hooks & tests | W2D2 |

---

*End of spec.*