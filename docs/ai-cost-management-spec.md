# AI Cost Management Specification

> Status: Draft v0.1 – 2025-06-29

---

## 0  Goal

控制 OpenAI / LLM 费用，同时保证决策质量。

---

## 1  Techniques

1. **Embedding Cache** – pgvector similarity search，命中率 80 %。  
2. **Prompt 模板重用** – 系统 prompt 缓存 ID。  
3. **动态模型路由** – trivial 问题走 `gpt-3.5-turbo`; high-stakes 走 `gpt-4o`。  
4. **批量调用** – consolidate multi-strategy questions into single prompt when possible。  
5. **Budget Guard** – ChiefAgent 维护 `budget_spent_usd` gauge；当日>limit→降级模型或阻止调用。

---

## 2  Bus Messages

| Topic | Payload |
|-------|---------|
| `llm.call` | `{model,prompt,tokens_est}` – emitted before调用 |
| `llm.result` | `{model,tokens,usd}` – after调用 |
| `llm.rate_limit` | `{until_ts}` – ChiefAgent broadcast 限流 |

---

## 3  Metric

| Metric | Type | Desc |
|--------|------|------|
| `llm_tokens_daily` | counter | 当日累计 tokens |
| `llm_cost_usd_daily` | counter | 当日费用 |

---

*End of cost spec.* 