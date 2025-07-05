# Config Schema – Biteasy Agent Platform

> Status: **Active v0.2** – 2025-07-01
> Author: AI Trading Architecture Assistant
> Scope: Define the canonical **YAML configuration** format, now with Pydantic-based validation.

---

## 0  Goals

1. Single declarative source of truth for which Agents load at runtime.
2. Support **cron**, **once**, and **interval** triggers per agent.
3. Parameterize each agent with strongly-typed values via **Pydantic models**.
4. Hot-reload friendly: Orchestrator can watch the file and apply diffs (add/remove/param change).
5. Minimal coupling – config file uses logical names, not hard-coded Python import paths.

---

## 0.1 Implementation Status

| ID | Task | Status | Notes |
|----|------|--------|-------|
| CS-1 | YAML Parser & Validation | **In Progress** | Migrating from basic `dataclass` to `Pydantic` for robust validation. |
| CS-2 | Orchestrator Integration | Done | Orchestrator loads agents from this config. |
| CS-3 | Hot-Reload Watcher | *Pending* | To be implemented using `watchfiles`. |
| CS-4 | Unit Tests | *Pending* | To be added for the new Pydantic models. |

---

## 1  Top-Level Keys

| Key | Type | Description |
|-----|------|-------------|
| `agents` | mapping | Map of `<agent_name>` to **AgentConfig**. Required. |
| `cron_timezone` | string | IANA TZ for cron parsing (e.g., "UTC", "Asia/Shanghai"). Defaults to "UTC". |
| `logging` | mapping | Optional log level overrides for different modules. |

### 1.1  AgentConfig

The configuration for a single agent.

```yaml
<agent_name>:
  enabled: true         # bool (default true). If false, the agent is ignored.
  entry_point: src.agents.data_market.DataMarketAgent  # Dotted path to the agent class.
  schedule: "*/1 * * * *"  # See section 2 for schedule format.
  params:               # Arbitrary mapping passed to Agent.__init__(**params).
    key: value
```

### 1.2 Logging Config

The `logging` block allows setting specific log levels for different parts of the application.

```yaml
logging:
  default: INFO  # The default log level for all loggers.
  src.strategies: DEBUG # Sets the log level for all modules under src.strategies to DEBUG.
  src.gateway: WARNING # Sets a specific logger to a higher level.
```

---

## 2  Schedule Format

The `schedule` key supports three formats:
*   **`@once`**: The agent is started once at application boot and runs indefinitely. Ideal for persistent connections (like WebSocket clients) or services.
*   **`interval: <duration>`**: The agent is run at a fixed interval. The duration format is `\d+[smh]` (e.g., `10s`, `5m`, `1h`).
*   **`cron_string`**: A standard 5-part cron string (e.g., `0 */4 * * *` for every 4 hours).

---

## 3  Validation Schema (Pydantic Model)

We use Pydantic for validation, providing clear error messages and type safety. The models below represent the expected structure.

```python
# (Illustrative Python code for src/config_loader.py)
from pydantic import BaseModel, Field
from typing import Dict, Any, Literal

class AgentConfig(BaseModel):
    enabled: bool = True
    entry_point: str
    schedule: str = "@once"
    params: Dict[str, Any] = Field(default_factory=dict)

class LoggingConfig(BaseModel):
    default: str = "INFO"
    # Allows arbitrary keys for logger names
    class Config:
        extra = "allow"

class RootConfig(BaseModel):
    cron_timezone: str = "UTC"
    logging: LoggingConfig = Field(default_factory=LoggingConfig)
    agents: Dict[str, AgentConfig]
```

---

## 4  Full Example

```yaml
cron_timezone: "Asia/Shanghai"

logging:
  default: INFO
  src.strategies.fr_arb_v2: DEBUG

agents:
  live_market_agent_btc:
    enabled: true
    entry_point: src.agents.data_live.LiveMarketAgent
    schedule: "@once"
    params:
      symbol: "BTCUSDT"
      skip_ssl_verify: true

  fr_arb_strategy_v2:
    enabled: true
    entry_point: src.strategies.fr_arb_v2.StrategyFRArbV2
    schedule: "interval:30s"
    params:
      symbol: "BTCUSDT"
      trade_qty: 0.01
      entry_threshold_bps: 5
      exit_threshold_bps: 1

  reporting_agent:
    enabled: true
    entry_point: src.agents.reporting.ReportingAgent
    schedule: "0 8 * * *" # Run daily at 8 AM Shanghai time
    params: {}
```

---

*End of config-schema spec.*