# Agent Roles Specification

> Status: Draft v0.1 – 2025-06-28  
> Author: AI Trading Architecture Assistant  
> Related: `agent-system-overview.md`

---

## Table of Roles

| Agent | Purpose | Key APIs / Tools | KPIs |
|-------|---------|------------------|------|
| ChiefAgent | Task planning, delegation, aggregation | LangChain AgentExecutor | Decision latency < 500 ms; Return on Decision (RoD) |
| DataAgent-Market | Stream market & derivatives data | ExchangeAdapter, Kafka | Data freshness < 2 s |
| DataAgent-Sentiment | Fetch social & news sentiment | Twitter API, NewsAPI | Coverage %, NLP score accuracy |
| StrategyAgent-* | Run specific strategy model | Strategy classes | Sharpe, hit-rate |
| RiskAgent | Enforce risk rules | RiskProxy | Violations blocked %, false positive <5% |
| InvestorAgent | Allocate capital, size trade | Optimizer | Portfolio Sharpe, drawdown |
| ReporterAgent | Summarise decisions to user | OpenAI GPT-4o, Markdown | Report latency <1 s, readability |
| OpsAgent | Monitor infra, heal | Prometheus API, k8s | MTTR, alert accuracy |

---

## Responsibility Matrix (RACI)

| Phase | C | R | A | I |
|-------|---|---|---|---|
| Data Refresh | Chief | DataAgents | Ops | Risk, Strategies |
| Signal Generation | Chief | StrategyAgents | Risk | Investor |
| Risk Assessment | Risk | Risk | Chief | Strategies |
| Portfolio Build | Investor | Investor | Chief | Ops |
| Reporting | Reporter | Reporter | Chief | User |
| Incident | Chief | Ops | User | All |

---

## Extensibility

* Agents register via entrypoint `agents.registry`.  
* New Agent must implement BaseAgent protocol (heartbeat, handle_message, get_state).

---

*End of roles spec.* 