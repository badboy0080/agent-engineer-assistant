# Anti-Pattern 速查表

完整的生产级 Agent Anti-Pattern 汇总，按严重程度排序。

---

## 🔴 Critical（必须修复）

### AP-1 用自然语言文本控制 Agentic Loop 终止 [D1]
```python
# ❌ 错误
if "task complete" in response.content[0].text:
    break
# ✅ 正确
if response.stop_reason == "end_turn":
    break
```

### AP-2 把 Access Failure 返回为空结果 [D2]
```python
# ❌ 错误：数据库挂了返回 []
return {"customers": []}
# ✅ 正确
return {"isError": True, "errorCategory": "timeout", "isRetryable": True, ...}
```

### AP-3 关键业务规则依赖 Prompt 而非 Hook [D1]
```python
# ❌ 错误：$500 限额放 Prompt（概率性）
system_prompt = "Never process refunds above $500"
# ✅ 正确：Hook 强制执行（确定性）
def refund_limit_hook(...): if amount > 500: return {"blocked": True}
```

### AP-4 MCP 配置硬编码密钥 [D2]
```json
// ❌ 错误
{"JIRA_TOKEN": "sk-abc123-real-key"}
// ✅ 正确
{"JIRA_TOKEN": "${JIRA_TOKEN}"}
```

### AP-5 同 Session 自我 Review [D3/D4]
```bash
# ❌ 错误
claude -p "Write auth module"
claude --resume -p "Review your code"  # 确认偏差
# ✅ 正确：新 session review
claude -p "Review this diff: ..."
```

---

## 🟡 Warning（应该修复）

### AP-6 单 Agent 工具数超过 5 个 [D2]
超过 5 个工具导致工具选择质量下降。应拆分为多个专职 Subagent，每个 ≤5 个工具。

### AP-7 向 Subagent 传递完整 Coordinator 历史 [D1]
```python
# ❌ 错误
Task(context=coordinator.full_conversation_history)
# ✅ 正确：只传相关 context
Task(context="Focus: market size in USD, top 3 vendors")
```

### AP-8 依赖 Progressive Summarization 保存关键信息 [D5]
重要信息（客户ID、金额、订单号）必须放在 Case Facts Block，不能靠摘要链传递。

### AP-9 使用情绪或模型置信度触发升级 [D1/D5]
```python
# ❌ 错误
if sentiment == "angry": escalate()
if model_confidence < 0.7: escalate()
# ✅ 正确：基于客观标准
if customer_requested_human: escalate()
if amount > AGENT_LIMIT: escalate()
```

### AP-10 Prompt 中使用模糊标准 [D4]
```
# ❌ 模糊
"flag long functions"
"find security issues"
# ✅ 明确
"flag functions exceeding 50 lines"
"flag strings matching patterns: sk-, pk-, key-"
```

### AP-11 所有配置放单一 CLAUDE.md [D3]
应用 `@import` 和 `.claude/rules/` 目录模块化管理配置。

### AP-12 只看总体准确率 [D5]
必须按文档类型分层追踪，总体平均会掩盖特定类别的严重失败。

---

## 🔵 Info（建议改进）

### AP-13 用 Write 覆盖文件而非 Edit [D2]
修改已有文件应用 `Edit(file, old, new)` 精确替换，不要用 `Write` 重写整个文件。

### AP-14 用 Bash 做内置工具能完成的事 [D2]
```python
Bash("cat file.txt")        # → Read("file.txt")
Bash("grep -r func src/")   # → Grep("func", "src/")
Bash("find . -name *.ts")   # → Glob("**/*.ts")
```

### AP-15 简单任务滥用 Plan Mode [D3]
Plan Mode 有额外开销，简单、定义清晰的任务应直接执行。

### AP-16 Few-Shot 示例数量过多或格式不一致 [D4]
最佳 2-4 个，超过 6 个边际效益递减。所有示例格式必须一致。

### AP-17 tool_use 后不做语义验证 [D4]
`tool_use` 只保证 JSON 结构合规，仍需验证字段值的语义正确性。

### AP-18 静态 Prompt Chain 用于动态任务 [D1]
动态复杂度的任务应使用 Dynamic Adaptive Decomposition，而非固定步骤链。

### AP-19 缺少信息溯源（Provenance）[D5]
多 Subagent 系统中，每条数据应附带 source、confidence、retrieved_at、agent_id。

### AP-20 fork_session 和 --resume 混淆使用 [D1]
- `--resume`：继续工作，需要完整历史
- `fork_session`：探索实验，不影响主 session
