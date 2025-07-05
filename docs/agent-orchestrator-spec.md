# Agent Orchestrator Specification

> Status: Draft v0.2 – 2025-06-29  
> Author: AI Trading Architecture Assistant  
> Related: `agent-system-overview.md`, `agent-communication-spec.md`

---

## 0  Purpose

The **Orchestrator** is the runtime kernel that:
1. Bootstraps all registered Agents (Chief, Data, Strategy, etc.).
2. Provides the intra-process **Bus** defined in communication spec.
3. Handles scheduling (cron / once / delay).
4. Detects failures, restarts Agents with back-off.

---

## 1  Lifecycle

| Phase | Action |
|-------|--------|
| `init` | load config, import agents.registry, create Bus |
| `start` | instantiate Agents, call `agent.start()` (async) |
| `run` | tick loop: heartbeats, cron triggers |
| `supervise` | watch task errors; if Agent crashes >3 times/5 min → disable + notify CA |
| `shutdown` | graceful cancel tasks, flush Bus |

---

## 2  Scheduling

* Cron spec in YAML:
```yaml
schedule:
  data_market: "*/1 * * * *"   # every minute
  investor: "0 */1 * * *"      # hourly
```
* Orchestrator uses `aiocron` to schedule `Bus.publish("cron.<agent>")`.

---

## 3  Fault Domains

| Agent Group | Isolation | Restart Policy |
|-------------|-----------|----------------|
| DataAgents | per agent `asyncio.Task` | always restart |
| StrategyAgents | per agent | restart, max 5 | 
| Critical (Risk, CA) | dedicated Task | fail-stop (orchestrator shutdown) |

---

## 4  Health & Metrics

* `/healthz` – returns 200 if all critical agents alive.
* `/metrics` – standard exporter plus `agent_state{agent}` gauge (0-down/1-up).

---

## 5  Implementation Sketch

```python
bus = Bus()
registry = load_registry()  # entry points via pkg_resources
agents = [cls(bus) for cls in registry]

for ag in agents:
    bus.subscribe(f"cron.{ag.name}", ag.handle_cron)
    asyncio.create_task(supervise_agent(ag))

await bus.start()
```

---

## 6  Ops & Monitoring (NEW)

| Endpoint | Description |
|----------|-------------|
| `/healthz` (port 8000) | returns `ok` if all critical agents alive (`agent_state` gauge =1) else 503 |
| `/metrics` (port 8001) | Prometheus export incl. `agent_state{agent}` |

*Supervisor*: Orchestrator restarts agent on crash with exponential back-off (max 60 s) and sets `agent_state=0`.  Three consecutive failures escalates to ChiefAgent.

---

*End of orchestrator spec.* 