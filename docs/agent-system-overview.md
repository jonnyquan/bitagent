# Agent-Based Trading Platform – System Overview

> Status: **v1.1 – Core Trading Workflow Implemented (2025-07-01)**  
> Author: AI Trading Architecture Assistant  
> This document introduces the **Agent Mode**: an AI-orchestrated trading team emulating a real hedge-fund desk.

---

## 0.  Concept

Transform Biteasy from a classic signal-pipeline into an **agentic trading organisation**. A `ChiefAgent` (future) coordinates a fleet of specialised AI Agents—data scouts, strategists, risk managers, and executors—to deliver adaptive and explainable trading decisions.

**Current Status (v1.1):** We have implemented a robust, end-to-end trading workflow. The system can ingest both live and historical data, generate strategy signals, perform pre-trade risk checks, execute orders on a live (testnet) exchange, and monitor its own health.

---

## 1. Agent Roster

| Component | Status | Responsibility (v1.1) | Future Enhancements |
|---|---|---|---|
| **`Data Agents`** | Implemented | `LiveMarketAgent` connects to live WebSocket feeds. `BacktestDataPlayer` replays historical data chronologically. | Add more data sources (on-chain, news, etc.). |
| **`StrategyAgent`** | Implemented | `StrategyFRArbV2` implements a funding rate arbitrage strategy. | Add more complex strategies and a library of reusable signal components. |
| **`RiskProxyAgent`** | Implemented | Intercepts signals and performs pre-trade checks (cooldown, max notional). | Add more sophisticated checks (position limits, drawdown controls). |
| **`ExecutionAgent`** | Implemented | `LiveExecutionAgent` uses an `IExchangeAdapter` to place orders on the exchange (live or testnet). | Implement a `SimulatedExchange` for accurate backtest PnL calculation. |
| **`ReportingAgent`** | Implemented | Performs periodic health checks on all agents and reports their status. | Generate daily PnL reports, send alerts to Slack/email. |
| **`SignalConverter`**| Implemented | A utility agent that translates multi-leg `ArbitrageSignal`s into individual `TradeOrderRequest`s. | |
| **`ChiefAgent`** | Planned | Orchestrate complex multi-signal decisions. | Add sophisticated decision logic using LLMs. |
| **`InvestorAgent`** | Planned | Perform portfolio optimization and capital allocation. | Dynamic capital management based on strategy performance. |

---

## 2. Core Execution Cycle (v1.1)

The current system operates on a linear, event-driven flow with a critical risk-checking step:

1.  **Data Ingestion**: `LiveMarketAgent` (live) or `BacktestDataPlayer` (backtest) publishes `MarketDataTick` and `FundingRate` events.
2.  **Signal Generation**: `StrategyFRArbV2` listens for data, and if its conditions are met, publishes an `ArbitrageSignal` event.
3.  **Risk Assessment**: `RiskProxyAgent` intercepts the `ArbitrageSignal`, validates it against configured rules (e.g., cooldown, size), and if it passes, publishes it to an "approved" topic.
4.  **Signal Conversion**: `SignalConverterAgent` listens for approved signals and breaks them down into individual `TradeOrderRequest` events.
5.  **Execution**: `LiveExecutionAgent` listens for order requests, uses its `CCXTAdapter` to place the trades on the exchange, and publishes a `TradeOrderFill` event upon success.
6.  **Monitoring**: `ReportingAgent` periodically sends out a `HealthCheckRequest` and reports on the status of all agents.

### 2.1. Sequence Diagram (v1.1)

```mermaid
sequenceDiagram
    participant Data as Data Agents
    participant Strat as StrategyFRArbV2
    participant Risk as RiskProxyAgent
    participant Conv as SignalConverter
    participant Exec as LiveExecutionAgent
    participant Report as ReportingAgent

    loop Data Flow
        Data->>+Bus: MarketDataTick / FundingRate
    end

    Strat->>+Bus: Listens for data
    Note over Strat: Arbitrage opportunity detected
    Strat->>+Bus: ArbitrageSignal

    Risk->>+Bus: Intercepts ArbitrageSignal
    Note over Risk: Applies cooldown & notional checks
    Risk->>+Bus: Publishes approved ArbitrageSignal

    Conv->>+Bus: Listens for approved signals
    Note over Conv: Breaks signal into legs
    Conv->>+Bus: TradeOrderRequest (Leg 1)
    Conv->>+Bus: TradeOrderRequest (Leg 2)

    Exec->>+Bus: Listens for order requests
    Note over Exec: Places orders via CCXTAdapter
    Exec->>+Bus: TradeOrderFill

    loop Health Check (Periodic)
        Report->>+Bus: HealthCheckRequest
        Strat-->>-Report: HealthCheckResponse
        Risk-->>-Report: HealthCheckResponse
        Exec-->>-Report: HealthCheckResponse
    end
```

---

## 3. Next Steps & Roadmap

With the core workflow in place, the next phases will focus on:
1.  **Backtest PnL Calculation**: Implementing the `SimulatedExchange` to enable accurate performance analysis of strategies.
2.  **Enhancing Intelligence**: Making the `StrategyAgent` and future `ChiefAgent` smarter with more complex rules and risk awareness.
3.  **Expanding Strategies**: Adding more diverse strategies to the system.
4.  **UI & Observability**: Developing a user interface for interaction and connecting to a full monitoring stack (Prometheus, Grafana).

*End of overview.* 