# Domain 4: Prompt Engineering & Structured Output

来源：https://claudecertifications.com/claude-certified-architect/domains/prompt-engineering

---

## D4.1 明确标准 vs 模糊指令

生产 Prompt 必须使用可量化的标准，不能用模糊描述。

| ❌ 模糊 | ✅ 明确 |
|--------|--------|
| "flag long functions" | "flag functions exceeding 50 lines" |
| "find security issues" | "identify hardcoded strings matching sk-, pk-, key-" |
| "make it better" | 具体列出改进标准 |

**误报（False Positive）的危害**：工具报了太多非问题，开发者开始忽略所有告警，包括真正的问题（Alert Fatigue）。

```python
# ✅ 明确的 Prompt
explicit_prompt = """
Review this code and flag ONLY the following:
1. Functions exceeding 50 lines of code
2. Async operations missing try-catch error handling
3. Hardcoded strings matching API key patterns (sk-, pk-, key-)
4. Public functions missing JSDoc documentation

For each issue:
- File path and line number
- Which rule (1-4) was violated
- Severity: critical (3) | warning (1,2) | info (4)
- One-line fix suggestion
"""
```

---

## D4.2 Few-Shot Prompting

- **最优数量**：2-4 个示例（超过 6 个边际效益递减，还浪费 token）
- **格式一致性**：所有示例必须用相同输出格式
- **边界覆盖**：至少包含一个边界/歧义示例
- **最适合**：任务边界模糊、有歧义的场景

```python
few_shot_prompt = """
Classify customer reviews. Provide sentiment and reasoning.

Example 1 (Clear positive):
Input: "Absolutely love this product!"
Output: {"sentiment": "positive", "confidence": "high", "reasoning": "Strong positive language"}

Example 2 (Ambiguous — mixed):
Input: "Great features but battery life is disappointing."
Output: {"sentiment": "mixed", "confidence": "medium", "reasoning": "Positive on features, negative on battery"}

Example 3 (Edge case — sarcasm):
Input: "Oh wonderful, another update that breaks everything."
Output: {"sentiment": "negative", "confidence": "medium", "reasoning": "Sarcastic positive masking frustration"}

Now classify:
Input: "{user_review}"
"""
```

---

## D4.3 通过 tool_use 获取结构化输出

`tool_use` 保证 JSON **结构**合规，但不保证**语义**正确。

| 保证什么 | 不保证什么 |
|---------|---------|
| 输出符合 JSON Schema | 字段值在语义上正确 |
| 字段类型正确 | vendor_name 是否准确 |
| required 字段存在 | 数值计算是否正确 |

```python
extract_tool = {
    "name": "extract_invoice",
    "description": "Extract structured data from an invoice",
    "input_schema": {
        "type": "object",
        "properties": {
            "vendor_name": {"type": "string"},
            "invoice_number": {"type": "string"},
            "date": {"type": "string", "description": "ISO 8601"},
            "total": {"type": "number"},
            "document_type": {
                "type": "string",
                "enum": ["standard_invoice", "credit_note", "proforma", "other"]
            },
            "document_type_detail": {
                "type": "string",
                "description": "Required if document_type is other"
            }
        },
        "required": ["vendor_name", "invoice_number", "date", "total", "document_type"]
    }
}

# 强制使用此工具 = 保证 Schema 合规
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    tools=[extract_tool],
    tool_choice={"type": "tool", "name": "extract_invoice"},
    messages=[{"role": "user", "content": f"Extract: {invoice}"}]
)
```

**用 tool_use 后仍需语义验证**：

```python
data = extract_via_tool_use(invoice)
errors = []
if not re.match(r"\d{4}-\d{2}-\d{2}", data["date"]):
    errors.append("Invalid date format")
if data["total"] <= 0:
    errors.append("Total must be positive")
if errors:
    retry_with_errors(invoice, errors)
```

---

## D4.4 Validation-Retry Loop & Multi-Pass Review

### Validation-Retry Loop

关键原则：**把具体错误信息 append 到 messages，不是 generic retry**

```python
def extract_with_validation(document, max_retries=3):
    messages = [{"role": "user", "content": f"Extract: {document}"}]
    
    for attempt in range(max_retries):
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            tools=[extract_tool],
            tool_choice={"type": "tool", "name": "extract_invoice"},
            messages=messages,
        )
        
        data = parse_tool_response(response)
        errors = validate(data)
        
        if not errors:
            return data  # 成功
        
        # ✅ 把具体错误 append 进对话，让模型自纠
        messages.append({"role": "assistant", "content": response.content})
        messages.append({
            "role": "user",
            "content": f"Validation failed. Fix these errors:\n"
                       + "\n".join(f"- {e}" for e in errors)
                       + "\nRe-extract with corrections."
        })
    
    raise ExtractionError(f"Failed after {max_retries} attempts")
```

### Multi-Pass Review（同 session 自我 review 是 Anti-Pattern）

生成和 review 必须在不同 session：同 session review 会因为保留了生成时的推理 context 而产生确认偏差，看不出自己的错误。

代码示例：

```python
candidate_diff = run_generation_session(task="implement feature")
review_report = run_fresh_session(
    task="review diff for Agent safety risks",
    input=candidate_diff,
)
assert review_report.session_id != candidate_diff.session_id
```

### Batch 处理策略

| 同步处理 | Batch API |
|---------|-----------|
| 需要立即结果 | 延迟容忍 |
| 阻塞型任务 | 50% 成本节省 |
| — | 24h 处理窗口 |

代码示例：

```python
batch_items = [
    {"custom_id": f"invoice-{i}", "prompt": make_extract_prompt(doc)}
    for i, doc in enumerate(documents)
]

for item in batch_items:
    assert item["custom_id"]  # 用于结果回填和错误定位
```
