# Testing Strategy – Biteasy

> Status: Draft v0.1 – 2025-06-29  

---

## 0  Pyramid

| Layer | Tool | Scope |
|-------|------|-------|
| Unit  | pytest + pytest-asyncio | bus functions, agents logic, utils |
| Integration | pytest (docker-compose) | orchestrator + agents in-process |
| E2E Smoke | GitHub Actions workflow | K8s staging deployment healthz |

---

## 1  Unit Tests

* **bus_loop**: publish/request, delayed schedule.  
* **RiskProxy**: depth, cooldown, leverage, pos_limit branches.  
* **InvestorAgent**: optimisation result shape, turnover calc.
* **ChiefAgent**: decision triggers, budget guard.
* **ReportingAgent**: formatting & channel dispatch.

Guidelines:
* Use fixtures for Bus / fake DB.  
* Keep tests <500 ms; mock external I/O.

---

## 2  Integration Tests

Scenario | Steps
---------|------
**Hot Reload** | Start Orchestrator → modify agents.yaml → expect new agent state=1 within 5s.
**Plan Flow** | Emit funding & book msgs → strategy.plan → RiskProxy approve → Investor alloc.
**Ops Auto-failover** | Simulate agent crash 3× → OpsAgent emits restart request → mock K8s API called.

---

## 3  Coverage & CI

* Target **85 %** line coverage.  
* `pytest --cov=tempcode -q`.  
* Coverage badge pushed to README via `actions/upload-artifact`.

---

## 4  Nose-Js & Tastains (JS frontend)

*Future note*: if web-dashboard added, unit/integration tests will follow Nose-Js & Tastains guidelines.

---

*End of testing strategy.* 