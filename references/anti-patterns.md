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

### AP-21 不可信输入直接进入 system prompt [D8]
网页、邮件、RAG 文档、工具输出、长期记忆、skill 文本都可能包含 prompt injection。必须用明确边界包装为“数据”，不能拼进 system prompt 当作指令。

### AP-22 Plan Mode 只靠 prompt 禁止写操作 [D7/D8]
Plan Mode 或“只分析不执行”时，必须在工具策略层禁用写文件、发邮件、执行 shell、调用 mutating MCP 等工具。只在 prompt 里写“不要执行”不可靠。

### AP-23 MCP schema 未做截断和清洗 [D7/D8]
第三方 MCP server 的工具名、参数名、描述都属于外部输入。渲染到 prompt 前应限制参数数量、字段长度和总长度，并清洗异常 token，避免 prompt 污染和上下文膨胀。

### AP-24 工具失败后静默结束 [D6/D9]
工具 timeout、权限失败、not found、schema 错误后，Agent 不能直接沉默或假装完成。必须说明失败类型，并选择修正参数、重试、降级或请求人工处理。

### AP-25 把 shell 当万能工具 [D7/D8]
能用专用 read/search/edit/list 工具时，不应用 shell、Python、curl、requests 绕过工具边界。shell 通常权限更大、审计更弱、路径风险更高。

### AP-26 文件工具缺少 allowlist / denylist / workspace containment [D7/D8]
文件读写工具必须先解析真实路径，再检查敏感路径 denylist 和允许根目录。禁止让模型控制的路径直接读写 `.ssh`、`.gnupg`、env、token、shell rc 等敏感文件。

### AP-27 admin 工具未按 owner/session 校验 [D7/D8]
即使后端有 admin 路由，Agent 工具分发层也必须校验当前 session owner 是否有权限。不能因为是“内部 loopback”就让非 admin 用户间接调用 admin 工具。

### AP-28 memory 写入没有成功门禁和来源记录 [D6/D8]
长期记忆写入必须确认真实提交成功，不能把 staged、失败或不可解析结果同步给外部 provider。写入应带 session、tool、agent、source 等 provenance。

### AP-29 context 压缩没有保护关键事实 [D5/D6]
压缩前必须保护 Case Facts、任务目标、权限边界、未完成事项和失败记录。只做自然语言摘要容易丢失订单号、金额、用户权限等关键事实。

### AP-30 上线前没有产品化验收清单 [D9]
没有测试矩阵、权限矩阵、日志审计、错误处理、回滚方案和人工升级流程的 Agent，只能算 demo，不能算生产系统。
