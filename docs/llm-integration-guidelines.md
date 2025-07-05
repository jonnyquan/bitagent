# LLM Integration Guidelines

> Status: Draft v0.1 – 2025-06-29

---

## 0  Prompt Structure

```text
<system>
 You are {role}. Follow JSON schema strictly.
</system>
<assistant>
 Acknowledged.
</assistant>
<user>
 {instruction}  
 ### Context  
 {yaml block}
</user>
```

* Always return JSON with `name`, `summary`, `actions` keys.  
* Use delimiters ```json to avoid parsing ambiguity.

---

## 1  Tool Invocation

| Tool | When to use | Required Fields |
|------|-------------|-----------------|
| code_edit | 修改/创建文件 | `target_file`, `code_edit` |
| bus.publish | 需要触发运行时 | `topic`, `payload` |

LLM MUST include only one tool call per output.

---

## 2  Temperature & Model Routing

| Task | Model | temp |
|------|-------|------|
| Long-form reasoning | gpt-4o | 0.2 |
| Quick classification | gpt-3.5-turbo | 0.1 |
| Code generation | gpt-4o | 0.15 |

ChiefAgent may override via `llm.route` broadcast.

---

## 3  Security

* Strip secrets from prompt using regex `(?i)(api[_-]?key)`.  
* Disallow tool calls to `run_terminal_cmd` unless pre-approved.

---

*End of LLM integration guidelines.* 