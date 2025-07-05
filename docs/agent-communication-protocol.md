# Agent Communication Protocol

This document defines the standard topics and message payloads used for communication between agents on the Redis message bus.

## 1. General Principles

- **Topic Naming Convention**: `category.event_type` (e.g., `data.tick`, `ops.error`).
- **Payload Format**: All payloads are JSON objects. Each payload has two top-level keys: `headers` and `body`.
  - `headers`: Contains metadata about the message (added automatically by the Bus).
    - `msg_id`: Unique identifier for the message.
    - `ts`: UTC timestamp in ISO 8601 format.
    - `reply_to`: (Optional) The topic to send a reply to for request/response patterns.
    - `corr_id`: (Optional) Correlation ID to match a response with a request.
  - `body`: Contains the actual message content.

---

## 2. System-Level Topics

### `agent.heartbeat`

- **Direction**: `Agent` -> `Bus`
- **Description**: A periodic signal indicating that an agent is alive and running.
- **Body**:
  ```json
  {
    "agent": "agent_name"
  }
  ```

### `ops.error`

- **Direction**: `Agent` -> `Bus`
- **Subscriber(s)**: `ReportingAgent`, `OpsAgent` (Future)
- **Description**: Published when an agent encounters a critical, unhandled error.
- **Body**:
  ```json
  {
    "agent": "agent_name",
    "error": "description of the error"
  }
  ```

---

## 3. Data Topics

### `data.tick`

- **Direction**: `DataMarketAgent` -> `Bus`
- **Description**: Represents a single trade that occurred on the exchange.
- **Body**:
  ```json
  {
    "symbol": "BTCUSDT",
    "ts": 1672531200,
    "price": 16500.50,
    "qty": 0.05
  }
  ```

### `data.book`

- **Direction**: `DataMarketAgent` -> `Bus`
- **Description**: Represents the top-of-book snapshot (best bid and ask).
- **Body**:
  ```json
  {
    "symbol": "BTCUSDT",
    "ts": 1672531200,
    "bid": 16500.00,
    "bid_qty": 1.2,
    "ask": 16500.50,
    "ask_qty": 0.8
  }
  ```

---

## 4. Strategy & Orchestration Topics

### `strategy.analyze.request`

- **Direction**: `OrchestratorAgent` -> `StrategyAgent`
- **Description**: A request for a strategy agent to perform an analysis.
- **Body**:
  ```json
  {
    "strategy_name": "simple_ma",
    "symbol": "BTCUSDT",
    "data": {
      "ticks": [ ... ], // Can include recent data to ensure consistency
      "book": { ... }
    }
  }
  ```

### `strategy.analyze.response`

- **Direction**: `StrategyAgent` -> `OrchestratorAgent`
- **Description**: The result of a strategy analysis.
- **Body**:
  ```json
  {
    "strategy_name": "simple_ma",
    "symbol": "BTCUSDT",
    "analysis": {
      "moving_average": 16501.23,
      "signal": "NEUTRAL" // e.g., BUY, SELL, NEUTRAL
    }
  }
  ```

---

## 5. Risk and Execution Topics

### `risk.validate.request`

- **Direction**: `OrchestratorAgent` -> `RiskAgent`
- **Description**: A request to validate a potential trading signal against risk rules.
- **Body**:
  ```json
  {
    "symbol": "BTCUSDT",
    "signal": "BUY",
    "price": 16501.23,
    "qty": 0.01 // Proposed quantity
  }
  ```

### `risk.validate.response`

- **Direction**: `RiskAgent` -> `OrchestratorAgent`
- **Description**: The result of a risk validation.
- **Body**:
  ```json
  {
    "approved": true, // or false
    "reason": "Checks passed" // or "Cool-down period active"
  }
  ```

### `execution.order.request`

- **Direction**: `RiskAgent` -> `ExecutionAgent` (or a gateway)
- **Description**: A final, approved order to be sent to the exchange.
- **Body**:
  ```json
  {
    "symbol": "BTCUSDT",
    "side": "BUY", // BUY or SELL
    "price": 16501.23,
    "qty": 0.01,
    "order_type": "LIMIT"
  }
  ```

### `execution.order.fill`

- **Direction**: `ExecutionAgent` -> `Bus`
- **Subscriber(s)**: `ReportingAgent`, `PortfolioAgent`
- **Description**: Reports that a trade has been successfully executed (a fill).
- **Body**:
  ```json
  {
    "symbol": "BTCUSDT",
    "side": "BUY",
    "price": 16501.20, // Actual fill price
    "qty": 0.01,
    "order_id": "exec_123",
    "trade_id": "trade_456"
  }
  ```

---

## 6. Portfolio Topics

### `portfolio.state.request`

- **Direction**: `Any Agent` -> `PortfolioAgent`
- **Description**: A request for the current portfolio state.
- **Body**:
  ```json
  {
    "symbols": ["BTCUSDT", "ETHUSDT"] // Optional: list of symbols to query. If empty, return all.
  }
  ```

### `portfolio.state.response`

- **Direction**: `PortfolioAgent` -> `Bus` (as reply)
- **Description**: The current state of positions held in the portfolio.
- **Body**:
  ```json
  {
    "positions": {
      "BTCUSDT": {
        "qty": 0.5,
        "avg_entry_price": 16450.75,
        "unrealized_pnl": 25.12
      }
    },
    "cash": 9500.00
  }
  ```

---

## 7. System Control Topics

### `system.control.update`

- **Direction**: `ChiefAgent` -> `Bus`
- **Subscriber(s)**: `OrchestratorAgent`, `RiskAgent`
- **Description**: A broadcast message to control the operational state of the entire system.
- **Body**:
  ```json
  {
    "command": "HALT_TRADING" // or "RESUME_TRADING"
    "reason": "Maximum drawdown limit of 5% reached."
  }
  ```
