# Domain 5: Context Management & Reliability

来源：https://claudecertifications.com/claude-certified-architect/domains/context-management

---

## D5.1 Context 优化 & 定位

### 两大核心问题

**1. Progressive Summarization 风险**

反复摘要会丢失关键细节：

```
原始：John Smith (ACC-12345) 订单 #98765，促销价 $99.99，实收 $150.00
第1次摘要：客户反映促销价格问题
第2次摘要：客户有账单问题
# 客户名、账号、订单号、具体金额、促销码——全部丢失
```

**2. "Lost in the Middle" 效应**

长 context 中，模型对中间部分的信息召回率最低，首尾最高。

### 解决方案：Case Facts Block

把关键信息放在 context 开头，以结构化表格保存，**永远不摘要这个块**：

```markdown
## CASE FACTS (Do not summarize — reference directly)

| Field | Value |
|-------|-------|
| Customer | John Smith |
| Account ID | ACC-12345 |
| Order | #98765 |
| Expected Price | $99.99 (promotion SUMMER2026) |
| Charged Price | $150.00 |
| Overcharge | $50.01 |
| Priority | High (7-year customer + overcharge) |

## RULES
- Always address customer as "Mr. Smith"
- Refund $50.01 is within $500 agent limit
```

---

## D5.2 升级 & 错误传播

### 有效升级触发条件 vs Anti-Pattern

| ✅ 有效触发 | ❌ 无效触发（Anti-Pattern） |
|------------|--------------------------|
| 用户明确要求转人工 | 用户情绪为负面（sentiment） |
| 情况超出 Policy 覆盖（Policy Gap） | 模型自报置信度 < 阈值 |
| 金额/权限超出 Agent 阈值 | 请求复杂度（主观判断） |
| 重试次数耗尽 | — |

```python
def should_escalate(context):
    # ✅ 有效触发
    if context.customer_requested_human:
        return True, "Customer explicitly requested human agent"
    if context.policy_gap_detected:
        return True, "No policy covers this situation"
    if context.amount > AGENT_REFUND_LIMIT:
        return True, f"Amount {context.amount} exceeds limit"
    if context.retry_count >= MAX_RETRIES:
        return True, "Exhausted retry attempts"
    
    # ❌ 永远不要这样
    # if context.sentiment == "negative": return True
    # if context.model_confidence < 0.7: return True
    
    return False, None
```

### 错误传播规范

Subagent 失败时，必须向 Coordinator 报告结构化错误，不能静默失败：

```python
# ❌ 错误：静默返回空结果
return {"results": []}  # Coordinator 以为查了没有，实际是查不到

# ✅ 正确：结构化错误上报
return {
    "isError": True,
    "errorCategory": "timeout",
    "isRetryable": True,
    "context": {
        "attempted": "Database lookup",
        "partial_results": [...],  # 已完成的部分
        "failed_at": "customer_db",
    }
}
```

---

## D5.3 Context 退化 & 长 Session 管理

### Context 退化症状

- 之前设定的约束开始被忽略
- 回答质量下降
- 对早期信息的引用出现错误

### 缓解策略

**1. Scratchpad 文件**

```python
agent.run("""
Before starting complex analysis:
1. Create a scratchpad file: progress.md
2. Record key findings as you discover them
3. Update progress.md after each major step
4. If context gets long, use /compact
5. After /compact, re-read progress.md to restore context
""")
```

**2. Subagent 委派冗长任务**

让 Subagent 做文件级的详细探索，只把摘要结果返回给 Coordinator，保持 Coordinator context 干净。

**3. 分层准确率追踪**

不能只看总体准确率，必须按文档类型分层追踪，否则会掩盖特定类型的失败：

```python
# ❌ 错误：只看总体平均
total_correct = 950
accuracy = 95.0%  # "看起来很好！"
# 实际：Invoices 70%（严重失败），Receipts 99.8%（总体被拉平）

# ✅ 正确：分层追踪
def track_accuracy(results):
    by_type = {}
    for r in results:
        doc_type = r["document_type"]
        by_type.setdefault(doc_type, {"correct": 0, "total": 0})
        by_type[doc_type]["total"] += 1
        if r["is_correct"]:
            by_type[doc_type]["correct"] += 1
    
    for doc_type, stats in by_type.items():
        accuracy = stats["correct"] / stats["total"] * 100
        print(f"{doc_type}: {accuracy:.1f}%")
# Invoice: 70.0% ← ALERT!
# Receipt: 97.8% ← OK
```

---

## D5.4 Human Review & 信息溯源（Provenance）

### 信息溯源结构

```python
from dataclasses import dataclass
from datetime import datetime
from typing import Literal

@dataclass
class DataWithProvenance:
    value: str | float | dict
    source: str              # "customer-db", "invoice-pdf", "web"
    confidence: Literal["verified", "extracted", "inferred", "estimated"]
    retrieved_at: datetime
    agent_id: str            # 哪个 Subagent 提供的

# 置信度排序：verified(4) > extracted(3) > inferred(2) > estimated(1)
```

### 冲突解决

当多个 Subagent 提供冲突数据时：
1. 按 confidence 选最可信来源
2. **记录冲突到 audit log**（不能静默选择）
3. 向 Coordinator 暴露冲突，而非自己决定

### Human-in-the-Loop 检查点

在以下场景必须暂停等待人工确认：
- 不可逆操作（删除、大额交易）
- 多来源数据严重冲突
- 超出 Agent 权限阈值

检查点呈现内容：当前状态、可选行动、推荐行动及依据、潜在风险。
