# Ops Agent Specification

> Status: Draft v0.1 – 2025-06-29  

---

## 0  Objective

自动化运维监控与自愈：收集 Prometheus/Loki 指标，生成 `ops.summary`，触发风控 fail-closed 或 K8s 重启。

---

## 1  Responsibilities

1. 聚合 `/metrics` → 计算 SLO（latency, error rate）。  
2. 连续 3 次 Agent crash → 调用 K8s API `PATCH /deployments/..` 重启。  
3. CPU≥90% 超 5 分钟 → 发布 `chief.command` 调整 replicaCount。  
4. 每 10 分钟发 `ops.summary` 至 ReportingAgent。

---

## 2  Bus Topics

| Topic | Direction | Payload |
|-------|-----------|---------|
| `ops.summary` | → Reporting | `{slo_ok:bool, restarts:int, cpu:...,mem:...}` |
| `ops.alert`   | → Chief | `{type:'high_cpu', detail:{}}` |

---

## 3  External Integrations

* **Prometheus HTTP API** – `/api/v1/query` for instant metrics.  
* **K8s API** (in-cluster service account) – deployment patch.  
* **Loki** – query latest ERROR logs for alert context.

---

## 4  Prom Metrics

| Metric | Type | Description |
|--------|------|-------------|
| `ops_restarts_total` | counter | 自动重启次数 |
| `ops_alerts_total` | counter | alert 触发次数 |

---

*End of Ops Agent spec.* 