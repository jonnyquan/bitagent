# Chief Agent Specification

> Status: Draft v0.1 – 2025-06-29  
> Role: "AI Trading Manager" – 策略委员会主席 + 业务负责人

---

## 0  Purpose

* 统一 **指挥调度** 整个 Agent 团队。  
* 依据 KPI、预算、风险阈值 **启停/调整** 子 Agent。  
* 面向主人(用户)提供 **决策摘要** 与 **商业洞察**。  
* 处理异常 (Agent 连续崩溃、风险过高) 并执行 **应急方案**。

---

## 1  Responsibilities

| Area | Action |
|------|--------|
| KPI  | 订阅 `perf.metrics`, `risk.violations`; 计算 CAGR、Sharpe、drawdown |
| Budget | 记录 `budget_spent_usd`; 若当日 > limit → 限制 LLM 请求频率 |
| Scheduler | 通过 Bus `agent.control` 消息启动/暂停/参数热调 |
| Decision  | 生成 `chief.decision` 消息：approve_new_strategy, increase_risk, etc. |
| Reporting | 协同 ReportingAgent 推送周报/风险警告 |

---

## 2  Bus Topics

| Topic | Direction | Payload |
|-------|-----------|---------|
| `chief.command` | → 子Agent | `{cmd:'start|stop|update', target:'agent', params:{}}` |
| `chief.decision` | ← 全局 | `{type:'rebalance', detail:{…}}` |
| `agent.state` | ← 子Agent | `{agent, state}` |
| `ops.error` | ← Agent | `{agent,error}` |

---

## 3  Prometheus Metrics

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `chief_decisions_total` | counter | type | 每类决策次数 |
| `budget_spent_usd` | gauge |  | 当日累计 LLM/云成本 |
| `chief_active_agents` | gauge |  | 当前运行 agent 数量 |

---

## 4  Config Snippet

```yaml
agents:
  chief:
    entry_point: agents.chief.ChiefAgent
    schedule: "@once"
    params:
      daily_budget_usd: 50
```

---

## 5  Milestones

| ID | Task | ETA |
|----|------|-----|
| CA-1 | Basic heartbeat & metrics | W1D1 |
| CA-2 | KPI collection & decision heuristics | W1D2 |
| CA-3 | Budget enforcement & rate-limit broadcast | W1D3 |
| CA-4 | Integration with ReportingAgent | W1D4 |

---

*End of Chief Agent spec.* 