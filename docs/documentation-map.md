# Biteasy Documentation Map

> Last updated: 2025-06-29 (auto-generated consolidation) – v0.5

This file provides a **single navigation hub** for all design documents under `ProjectDocs/Designs/biteasy/`.  Each entry lists purpose and latest status.

| Group | Doc | Purpose | Status |
|-------|-----|---------|--------|
| **Overview** | trading-system-overview.md | High-level concept & domain context | Draft v0.1 |
| | trading-system-detailed-plan.md | Milestone breakdown & repo layout | Draft v0.1 |
| **Data** | data-layer-spec.md | ETL & pipelines to MySQL/Redis | Draft v0.1 |
| | macro-derivatives-data-spec.md | Macro + derivatives data sourcing | Draft v0.1 |
| **Core Framework** | strategy-framework-spec.md | Strategy base interfaces & lifecycle | Draft v0.1 |
| | execution-risk-framework-spec.md | Risk separation principles | Draft v0.1 |
| | optimizer-spec.md | Semi-variance portfolio allocator | Draft v0.1 |
| **Runtime Layer** | exchange-adapter-spec.md | Exchange connectivity abstraction | Draft v0.1 |
| | strategy-runner-spec.md | Async runner supervising strategies | Draft v0.1 |
| | risk-proxy-spec.md | Real-time risk guard & rules DSL | Draft v0.2 |
| | config-schema.md | YAML + env configuration schema | Draft v0.1 |
| **AI & Agent** | ai-decision-spec.md | Multi-model signal gating & mode switch | Draft v0.1 |
| | ai-agent-spec.md | Chat-based agent orchestrating tools | Draft v0.1 |
| | chief-agent-spec.md | Master coordinator of agents | Draft v0.1 |
| | reporting-agent-spec.md | Notification/summary agent | Draft v0.1 |
| | ops-agent-spec.md | Automated ops & self-heal agent | Draft v0.1 |
| | ai-cost-management-spec.md | LLM cost control rules | Draft v0.1 |
| | llm-integration-guidelines.md | Prompt/tool guidelines | Draft v0.1 |
| **Backtesting & Ops** | backtesting-framework-spec.md | Vector/event backtest engines | Draft v0.1 |
| | monitoring-ops-spec.md | Observability stack & SLOs | Draft v0.1 |
| | deployment-spec.md | Container & Kubernetes topologies | Draft v0.1 |
| **Strategies** | strategy-fr_arb_v2-spec.md, strategy-agent-base-spec.md | Strategy logic & base class | Draft v0.1 |
| **Roadmap** | strategy-advanced-roadmap.md | Future strategy upgrades | Draft |
| **Agent Architecture** | agent-system-overview.md, agent-roles-spec.md, agent-communication-spec.md, agent-orchestrator-spec.md, data-agents-spec.md, strategy-agents-spec.md, investor-agent-spec.md, feature-builder-spec.md, data-sentiment-agent-spec.md, data-macro-agent-spec.md, data-onchain-agent-spec.md, macapp-ui-spec.md | Agent-based orchestration | Draft v0.3 |
| **UI** | macapp-ui-spec.md | Desktop UI for interaction | Draft v0.2 |

> For doc authors: when you add a new spec, update this map to keep references consistent.

---

*End of doc map.* 