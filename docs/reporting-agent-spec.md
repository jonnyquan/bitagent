# Reporting Agent Specification

> Status: **Partially Implemented v1.1** – 2025-07-01
> **Implementation Note:** The current implementation focuses on the `OpsAgent` aspect of reporting, providing a system-wide health check (heartbeat) mechanism. Daily PnL reports and external channel integration are planned for future versions.

---

## 0  Purpose

实时/定期向主人及运营团队推送重要事件与摘要。

---

## 1  Channels

| Channel | Transport | Notes |
|---------|-----------|-------|
| Slack   | Webhook   | `SLACK_WEBHOOK_URL` env |
| Webhook | HTTP POST | Custom endpoints |
| MacApp  | WebSocket | relay via Orchestrator gateway |

---

## 2  Report Types

| Type | Trigger | Template |
|------|---------|----------|
| 日报 | cron `0 18 * * *` | PnL, KPI 表格,风险曲线 |
| 风险告警 | `risk.violations_total` > threshold | `:warning: 风险 {rule} count={cnt}` |
| 盈亏里程碑 | equity crosses +5% or -5% | celebratory / caution message |

---

## 3  Bus Interaction

* 订阅 `report.request` → 立即生成摘要并回复。  
* 自动监听 `risk.violation`, `perf.equity_milestone` 等事件生成告警。

---

## 4  Metrics

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `reports_sent_total` | counter | channel,type | 报告发送次数 |

---

## 5  Config

```yaml
agents:
  reporting:
    entry_point: agents.reporting.ReportingAgent
    schedule: "0 18 * * *"
    params:
      slack_webhook: "https://hooks.slack.com/..."
```

---

*End of Reporting Agent spec.* 