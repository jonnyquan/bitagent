# DataSentimentAgent Specification

> Status: Draft v0.1 – 2025-06-29  
> Related: `data-agents-spec.md`, `agent-system-overview.md`

---

## 0  Purpose
收集 Twitter & News 关键信息，使用 NLP 模型计算情绪分数并通过 Bus 发布 `data.sentiment` 消息，供 FeatureBuilder 和策略使用。

---

## 1  External APIs
| Source | Endpoint | Throttle | Cost |
|--------|----------|----------|------|
| Twitter v2 | `/tweets/search/recent` | 450 req/15 min | Free dev / $100 pro |
| NewsAPI | `/v2/top-headlines` | 100 req/day | $449/mo |

---

## 2  Processing Pipeline
1. **Collector** – Poll Twitter/News every 5 min (cron) for keywords `btc,bitcoin,crypto,binance fed`。  
2. **Cleaner** – strip urls/mentions, dedupe headlines。  
3. **SentimentModel** – FinBERT or HuggingFace `cardiffnlp/twitter-roberta-base-sentiment-latest`。  
4. **Aggregator** – 10 min rolling average → score ∈ [-1,1]。  
5. **Publisher** – Bus topic `data.sentiment` payload:
```jsonc
{ "ts": 1727486400, "score": 0.34, "n": 125 }
```

---

## 3  Cron & Caching
* Schedule: `*/5 * * * *` via Orchestrator cron。  
* Cache last 24 h tweets in SQLite to avoid duplicates。

---

## 4  Implementation Tasks
| ID | Task | ETA |
|----|------|-----|
| DS-1 | `data_sentiment.py` agent skeleton | W2D1 |
| DS-2 | HuggingFace pipeline + async batch scoring | W2D1 |
| DS-3 | Dedupe + SQLite cache | W2D1 |
| DS-4 | Prom metrics `sentiment_score` | W2D1 |

---

*End of spec.* 