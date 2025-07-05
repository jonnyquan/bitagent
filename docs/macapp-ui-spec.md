# MacApp UI Specification

> Status: Draft v0.1 – 2025-06-28  
> Status: Draft v0.2 – 2025-06-29  
> Author: AI Trading Architecture Assistant  
> Related: `agent-system-overview.md`, `agent-communication-spec.md`

---

## 0  Tech Stack

* **SwiftUI + Combine** for reactive UI.  
* **WebSocket** connection to Orchestrator (ws://localhost:8080/chat).  
* **CoreData** local cache for chat history & alerts.  
* Sandbox Apple notarised app.

---

## 1  Screens

1. **Chat** – scrollable dialogue with ChiefAgent; markdown rendering, code highlighting.  
2. **Dashboard** – equity curve, current positions, AI metrics gauges.  
3. **Alerts** – list of `TradeAlertProposal`; tap → detail → "Send" button.  
4. **Settings** – API endpoint, theme, notifications.
5. **Agent Panel** – 列表展示所有 Agent 状态 (heartbeat, cpu, mem)。

---

## 2  Component Architecture

| Component | Responsibility |
|-----------|----------------|
| `WebSocketManager` | manage WS, auto-reconnect, JSON decoding |
| `ChatViewModel` | publish messages list, send text |
| `AlertStore` | store/observe pending alerts |
| `DashboardVM` | ingest metrics via `/metrics` SSE |

State lifted by **ObservableObject**, views subscribe via `@StateObject`.

---

## 3  Message Format

```jsonc
{
  "type": "chat",
  "role": "assistant|user|system",
  "content": "markdown text",
  "ts": 1727486400
}
```

Alerts arrive as:
```jsonc
{ "type":"alert_proposal", "payload": {…TradeAlertProposal…} }
```

Gateway control:
```jsonc
{ "type":"agent_control", "agent":"risk_proxy", "cmd":"pause" }
```

---

## 4  UX Flow

1. User opens Chat, types request → `WebSocketManager.send(chat msg)`  
2. ChiefAgent responds streaming chunks → Combine publisher updates ChatView.  
3. If response contains alert proposal, `AlertStore` adds item → badge on Alerts tab.  
4. On send, MacApp POSTs `/dispatch` HTTP endpoint.

## 5  WebSocket Gateway (NEW)

Orchestrator exposes `ws://host:8080/ws` which bridges Bus topics:
* Outbound → client: `chat`, `alert_proposal`, `metrics_update` (JSON).  
* Inbound ← client: `chat`, `agent_control`.

Security: JWT signed by ChiefAgent, expiry 2h.

---

## 5  Implementation Milestones

| ID | Task | ETA |
|----|------|-----|
| UI-1 | Swift Package skeleton | W1D1 |
| UI-2 | WebSocketManager + Chat UI | UI-1 | W1D3 |
| UI-3 | Dashboard graphs (Charts) | UI-2 | W1D4 |
| UI-4 | Alerts list & action | UI-2 | W1D4 |
| UI-5 | Preferences window | UI-2 | W1D4 |

---

*End of MacApp spec.* 