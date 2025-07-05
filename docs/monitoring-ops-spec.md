# Monitoring & Ops Specification

> Status: Draft v0.1 – 2025-06-29  
> Author: SRE Lead  

---

## 0  Monitoring Stack

| Component | Purpose |
|-----------|---------|
| Prometheus | scrape /metrics; Alertmanager integration |
| Grafana    | dashboards (trading perf, agent health) |
| Loki       | centralised logs via promtail sidecar |
| Alertmanager | route alerts → PagerDuty / Slack |
| Sentry     | Python exception tracking |

---

## 1  Key Metrics

Category | Metric | Labels | Description
---------|--------|--------|------------
Agent | `agent_state` (gauge) | agent | 0/1 alive state
Risk | `risk_violations_total` (counter) | strategy,rule | rejected plans count
Investor | `investor_leverage` (gauge) |  | current leverage
Bus | `bus_messages_total` (counter) | topic | throughput (future work)
Infra | CPU, Mem, FD | pod | K8s node exporter

---

## 2  Alert Rules (Prometheus)

* **High Risk Violations** – `rate(risk_violations_total[5m]) > 5` for ANY rule.  
* **Investor Leverage Too High** – `investor_leverage > 2.0` (warning) / `> 3.0` (critical).  
* **Agent Down** – `agent_state == 0` for 30s.

---

## 3  Dashboard Panels

1. *Overview*: Equity curve, PnL, leverage, violations bar.  
2. *Agent Health*: agent_state heatmap, heartbeat latency.  
3. *Risk*: Depth ratio, leverage ratio, violation trends.  
4. *Infra*: CPU / Mem / Pods restarts.

---

## 4  Ops Runbook (Excerpt)

| Alert | Immediate Action | Follow-up |
|-------|------------------|-----------|
| Agent Down | `kubectl logs` & restart pod | RCA within 24h |
| Leverage >3 | Halt new plans via RiskProxy `fail-closed` flag | Adjust Investor params |
| Depth Ratio >2 | Check liquidity; reduce size limits | Review markets |

---

*End of monitoring spec.* 