# Data Layer Specification – Binance Quant Trading System

> Status: Draft v0.1 – 2025-06-27  
> Author: AI Trading Architecture Assistant  
> Related Docs: `trading-system-overview.md`, `trading-system-detailed-plan.md`

---

## 0. Purpose & Scope

本文件描述**数据层**所有组件的功能需求、技术实现方案与验收标准，确保后续策略与执行模块拥有**可靠、低延迟且完整**的数据支撑。

---

## 1. 功能规格 (Functional Spec)

| # | 需求描述 | KPI / SLA |
|---|-----------|-----------|
| F-1 | 拉取 Binance Spot & USD-M Futures OHLCV (1m,5m,1h,1d) | < 1 min 延迟；缺口 ≤ 0.05 %/月 |
| F-2 | 拉取 Funding Rate (8h) & Open Interest (5m) | Funding 0 s 延迟 +0 s/–5 s 容忍 |
| F-3 | 实时订阅 L2 Depth@100ms & aggTrades | 99.9 % 包到达；平均延迟 < 200 ms |
| F-4 | 持久化至 **MySQL** + 归档 S3 | 查询 QPS ≥ 200（单表） |
| F-5 | 支持增量补档与 **数据一致性校验** | 每日差异报告；自动回填重试 3 次 |
| F-6 | 提供统一 Python API (`DataGateway`) 供策略调用 | 单查询 < 50 ms（内存缓存命中） |

---

## 2. 技术方案 (Technical Design)

### 2.1 Overall Pipeline

```text
┌──────────┐   REST       ┌────────────┐
│Binance   │────────────▶│  ETL Cron  │──┐  (batch)
│ REST/WS  │             └────────────┘  │
└─────┬────┘                              ▼
      │   WebSocket                ┌──────────────┐
      └───────────────────────────▶│  Streamer    │ (asyncio)
                                   └──────────────┘
                                           │ Redis pub/sub
                                           ▼
                                 ┌──────────────────┐
                                 │  MySQL           │
                                 └──────────────────┘
                                           ▼ logical_replication
                                   ┌──────────────────┐
                                   │   S3 archival    │
                                   └──────────────────┘
```

* **ETL Cron** (`src/data/etl/etl_ohlcv.py`, `etl_macro.py`, `etl_funding.py`)
  * `etl_ohlcv.py` 拉取 Spot & Perp OHLCV，5 min 间隔。
  * `etl_macro.py` 调用 FRED / Glassnode 等宏观 API（见 `macro-derivatives-data-spec.md`）。
  * `etl_funding.py` REST 补拉 8 h funding 缺口。
  * 统一通过 Airflow DAG `ohlcv_etl`, `macro_daily` 调度。

* **Streamer** (`src/data/streamer.py`)
  * 负责 **价格与深度** WebSocket：`!bookTicker`, `@aggTrade`, `@depth@100ms`。

* **Derivatives Streamer** (`src/data/derivatives_streamer.py`)
  * 负责 **资金费率 / Open Interest / 标记价格** WebSocket：`@fundingRate`, `@markPrice`, `@openInterest`.

* **Data Gateway** (`src/data/loader.py`)
  * Synchronous & async wrappers over **MySQL** / Redis cache.
  * Caches latest N candles & order-book snapshots in-memory (`lru_cache`).

### 2.2 MySQL Schema (updated)

```sql
-- 1. OHLCV
CREATE TABLE ohlcv (
    ts         DATETIME NOT NULL,
    symbol     VARCHAR(20) NOT NULL,
    interval   ENUM('1m','5m','1h','1d') NOT NULL,
    open       DECIMAL(18,8),
    high       DECIMAL(18,8),
    low        DECIMAL(18,8),
    close      DECIMAL(18,8),
    volume     DECIMAL(28,8),
    PRIMARY KEY (ts, symbol, interval)
) ENGINE=InnoDB
PARTITION BY RANGE COLUMNS(ts) (
  PARTITION p202501 VALUES LESS THAN ('2025-02-01'),
  PARTITION p202502 VALUES LESS THAN ('2025-03-01'),
  PARTITION pmax VALUES LESS THAN (MAXVALUE)
);

-- 2. funding_rate
CREATE TABLE funding_rate (
    ts      DATETIME PRIMARY KEY,
    symbol  VARCHAR(20),
    rate    DECIMAL(10,8)
) ENGINE=InnoDB
PARTITION BY RANGE COLUMNS(ts) (
  PARTITION p202501 VALUES LESS THAN ('2025-02-01'),
  PARTITION pmax VALUES LESS THAN (MAXVALUE)
);

-- 3. executions
CREATE TABLE executions (
    trade_id     BIGINT PRIMARY KEY,
    ts           DATETIME,
    symbol       VARCHAR(20),
    side         ENUM('buy','sell'),
    qty          DECIMAL(28,8),
    price        DECIMAL(18,8),
    strategy     VARCHAR(30),
    order_id     VARCHAR(40),
    KEY idx_ts_symbol (ts, symbol)
) ENGINE=InnoDB
PARTITION BY RANGE (UNIX_TIMESTAMP(ts)) (
  PARTITION p202501 VALUES LESS THAN (UNIX_TIMESTAMP('2025-02-01')),
  PARTITION pmax VALUES LESS THAN MAXVALUE
);
```

**Indexes**: cover `(symbol, ts)` for time-range queries; use `BTREE`.

### 2.3 Fault-Tolerance & Backfill

1. 每日 00:10 UTC 比较本地与 Binance `klines` 行数差异 → 生成缺口列表。
2. 缺口 ≤ 20 行：即时重拉；>20 行：分批重拉避免限速。
3. 三次失败后写入 `data_gap_log` 并触发 PagerDuty。

### 2.4 Monitoring Metrics (Prometheus)

| Metric | Type | Description |
|--------|------|-------------|
| `ingest_lag_seconds{source="ohlcv"}` | Gauge | 最新 candle ts 与系统时钟差 |
| `missing_bars_total{interval="1m"}` | Counter | 自动校验发现的缺口数量 |
| `ws_message_latency_ms` | Histogram | WebSocket 消息延迟 |
| `db_upsert_duration_ms` | Histogram | 单批次 UPSERT 耗时 |

Alert thresholds见 `docker/grafana/alerts/data_rules.yaml`.

### 2.5 Retention & Archival Policy (revised)

* Hot data stored in latest 3 monthly partitions.  
* Monthly script `ALTER TABLE … DROP PARTITION` exports partition to Parquet via `SELECT … INTO OUTFILE S3` before drop.

### 2.6 Data Quality Validator (NEW)

* Micro-service `validator.py` subscribes to Redis Stream `ts.results` (order executions) & `ts.signals` to correlate expected vs. actual candles.
* Rules:
  1. **Range Check**: OHLCV columns must be positive, `low ≤ high`, `open/close` within `[low,high]`.
  2. **Monotonic Timestamp**: `ts` strictly increasing per `symbol,interval`.
  3. **Missing Rate Alert**: If `missing_bars_total{interval="1m"} > 3` within 10 min window, send alert.
* Violations sent to `ts.alerts` with subtype `data_quality`.

### 2.7 Scheduling & Orchestration (NEW)

* **Airflow DAG** (`docker/airflow/dags/ohlcv_etl.py`)
  * `extract_raw` → `transform_normalise` → `load_mysql` → `archive_s3` tasks.
  * Run hourly; backfill DAG for historic periods.
* **K8s CronJob** as fallback when Airflow unavailable.
* All batch tasks emit Prometheus `job_success_total` / `job_failure_total`.

---

## 3. 验收标准 (Acceptance Criteria)

1. **数据完整性**：随机抽样 1,000 根 1 m K 线，与 Binance 原始数据对比误差 0。  
2. **延迟**：`ingest_lag_seconds{interval="1m"}` 99th percentile < 60 s 连续 24 h。  
3. **性能**：单表 `SELECT * FROM ohlcv WHERE symbol='BTC/USDT' AND ts>now()-interval '1 day'` 耗时 < 120 ms。  
4. **容错**：模拟网络断连 5 min，自动重连后补齐缺口。  
5. **监控**：Grafana 仪表板展示 ingest lag、WS latency、missing bars。报警邮件/Telegram 到达率 100 %。

---

## 4. TODO & 分工

| Task ID | 内容 | Owner(role) | 预计完成 |
|---------|------|-------------|----------|
| DL-1 | 在 `docker/mysql/` 目录编写 `Dockerfile` + `init.sql` （包含上述表） | Dev-Infra | D+2 |
| DL-2 | `etl_job.py`：REST 拉取历史 K 线，批量 `INSERT ... ON DUPLICATE KEY UPDATE` | Dev-Data | D+3 |
| DL-3 | `streamer.py`：WebSocket 接收 depth/trade → Redis Stream | Dev-Data | D+4 |
| DL-4 | `data_gateway.py`：统一查询接口 + LRU 本地缓存 | Backend-Eng | D+5 |
| DL-5 | Prometheus Exporter `metrics_data.py` 汇报 ingest lag、upsert 时延 | DevOps | D+5 |
| DL-6 | `validator.py` + 差异回填脚本 `backfill.py` | QA-Analyst | D+6 |

> *D = 执行计划起始日*

---

*End of spec.* 