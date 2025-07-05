# Agent Communication Specification – Biteasy

> Status: **v1.2 – Risk Integration (2025-06-30)**  
> Author: AI Trading Architecture Assistant  
> Related: `agent-system-overview.md`, `agent-roles-spec.md`, `bus.py`, `schemas/models.py`

---

## 0. Goals

1.  **Decoupling**: Agents must be fully decoupled, depending only on message schemas, not on direct references to other agents.
2.  **Clarity & Safety**: Message structures must be explicit, strongly-typed, and validated to prevent runtime data errors.
3.  **Performance & Scalability**: The system must support high-speed, in-process communication for core logic, while allowing for durable, cross-process communication for scalability and external integration.
4.  **Testability**: The communication layer must be easily mockable or replaceable for unit and integration testing.

---

## 1. Architecture: Dual-Bus System

We will employ a dual-bus architecture that leverages the implementations in `codes/src/bus.py`:

1.  **Real-Time Bus (`InMemoryBus`)**:
    *   **Transport**: An in-process, `asyncio`-based bus.
    *   **Purpose**: Used for all high-frequency, real-time communication between agents within the main application process. This includes market data flow, signal generation, and order requests.
    *   **Characteristics**: Extremely low latency, high throughput, but not persistent.

2.  **Event-Sourcing Bus (`RedisBus`)**:
    *   **Transport**: A Redis Pub/Sub-based bus.
    *   **Purpose**: Acts as a durable event log. Every critical message sent on the `InMemoryBus` can also be mirrored to the `RedisBus`.
    *   **Characteristics**:
        *   **Durability**: Provides a persistent record of all system events.
        *   **Replayability**: Allows for replaying event streams for backtesting, debugging, or system recovery.
        *   **External Integration**: Enables external monitoring, analytics, or logging systems to consume the event stream without impacting the core application's performance.

---

## 2. Interaction Patterns

Agents primarily interact through two patterns supported by the bus:

1.  **Publish/Subscribe (Pub/Sub)**: This is the default one-to-many communication method. An agent publishes an event to a topic, and all subscribed agents receive it. This is ideal for broadcasting information like market data or system-wide alerts.
2.  **Request/Reply**: This is a one-to-one pattern used when an agent needs a direct response from another. The requesting agent sends a message on a specific topic and waits for a reply on a temporary, private topic. This is crucial for imperative actions and queries.

### 2.1. Chief Agent Interaction Patterns

The `ChiefAgent`, as the central decision-maker, heavily utilizes the **Request/Reply** pattern to orchestrate the team. For example:
*   **Risk Validation**: Before issuing a final trade order, the `ChiefAgent` **must** send a `TradeValidationRequest` to the `RiskAgent` and wait for an `APPROVED` response.
*   **Information Gathering**: It can directly query the `PortfolioAgent` for the current state (`PortfolioStateRequest`) before allocating capital.

---

## 3. Message Structure

All messages exchanged on the bus are encapsulated in a generic `Event` object, defined in `codes/src/schemas/models.py`. It consists of a `header` and a `body`.

-   **`header` (`EventHeader`)**: Contains metadata automatically attached by the bus.
    -   `msg_id` (uuid.UUID): A unique UUID for the message.
    -   `topic` (str): The topic the message was published on.
    -   `ts` (datetime): UTC timestamp in ISO 8601 format.
    -   `corr_id` (Optional[uuid.UUID]): Correlation ID for matching responses in request/reply patterns.
    -   `reply_to` (Optional[str]): The topic to send a reply to, used by the bus for the request/reply pattern.
-   **`body` (Pydantic `BaseModel`)**: The actual message content, defined by a specific Pydantic model.

---

## 4. Topic Naming Convention

Topics are strings that follow a consistent `domain.event_name` pattern to maintain clarity and order.

-   **`data.*`**: Raw or derived data events (e.g., `data.market_tick.btc_usd`).
-   **`strategy.*`**: Events related to strategy logic (e.g., `strategy.signal`).
-   **`chief.*`**: Events originating from or directed to the Chief Agent (e.g., `chief.decision`).
-   **`risk.validation.*`**: Risk validation events (e.g., `risk.validation.request`, `risk.validation.response`).
-   **`execution.*`**: Order execution events (e.g., `execution.order_request`, `execution.order_fill`).
-   **`portfolio.*`**: Portfolio state and management events (e.g., `portfolio.state`, `portfolio.state_request`).
-   **`system.*`**: System-wide control and status events (e.g., `system.heartbeat`).
-   **`ops.*`**: Operational events (e.g., `ops.error`).

---

## 5. Core Message Schemas (Pydantic Models)

Below are the key Pydantic models for system events, defined in `codes/src/schemas/models.py`. This is not an exhaustive list.

```python
# Note: This is a documentation snippet.
# The source of truth is codes/src/schemas/models.py

# --- Base & Common Models ---
class EventHeader(BaseModel): ...
class Event(BaseModel, Generic[T]): ...

# --- Data Events ---
class MarketDataTick(BaseModel): ...

# --- Strategy Events ---
class StrategySignal(BaseModel): ...

# --- Chief Agent Events ---
class ChiefDecision(BaseModel): ...

# --- Risk & Execution Events ---
class TradeValidationRequest(BaseModel):
    """A request sent to the RiskAgent to validate a potential trade."""
    order_request: TradeOrderRequest

class TradeValidationResponse(BaseModel):
    """A response from the RiskAgent after validating a trade."""
    decision: Literal["APPROVED", "REJECTED"]
    reason: Optional[str] = None
    order_request: TradeOrderRequest

class TradeOrderRequest(BaseModel): ...
class TradeOrderFill(BaseModel): ...

# --- Portfolio Events ---
class PortfolioState(BaseModel): ...
class Position(BaseModel): ...

# --- System Control & Ops Events ---
class SystemControlCommand(BaseModel): ...
class AgentHealthStatus(BaseModel): ...
class OpsError(BaseModel): ...
```

---

## 6. Serialization

-   **In-Process**: Pydantic model objects are passed directly on the `InMemoryBus` for maximum performance. No serialization is needed.
-   **Cross-Process/Durable**: For the `RedisBus`, Pydantic models are serialized to JSON using `model_dump_json()`.

---

## 7. Error Handling

-   Handler exceptions within agents are caught by the bus's `_safe_call` wrapper.
-   An `ops.error` event is published containing the details of the failure.
-   For `request/reply` patterns, a failed handler will result in the `asyncio.Future` being resolved with an exception, which the requester must handle.

---
*End of communication spec v1.2.*
