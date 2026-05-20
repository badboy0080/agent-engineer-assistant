# Domain 1: Agentic Architecture & Orchestration

来源：https://claudecertifications.com/claude-certified-architect/domains/agentic-architecture

---

## D1.1 Agentic Loop & Core API

### 核心机制

Agentic loop 是 Claude Agent 的基础执行模式，允许 Claude 迭代地 Plan → Act → Observe → Decide。

**stop_reason 是控制 loop 的唯一可靠信号**：
- `tool_use`：Claude 需要调用工具，loop 继续
- `end_turn`：Claude 认为任务完成，loop 终止

### 规范代码

```python
import anthropic

client = anthropic.Anthropic()
tools = [{"name": "lookup_customer", "description": "...", "input_schema": {}}]
messages = [{"role": "user", "content": "Find customer John Smith"}]

while True:
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=4096,
        tools=tools,
        messages=messages
    )
    
    # ✅ 正确：检查 stop_reason 控制 loop
    if response.stop_reason == "end_turn":
        break  # Claude 决定结束

    if response.stop_reason == "tool_use":
        tool_block = next(b for b in response.content if b.type == "tool_use")
        result = execute_tool(tool_block.name, tool_block.input)
        
        messages.append({"role": "assistant", "content": response.content})
        messages.append({
            "role": "user",
            "content": [{"type": "tool_result",
                         "tool_use_id": tool_block.id,
                         "content": result}]
        })
```

### ❌ Anti-Pattern

```python
# 错误：解析自然语言文本决定是否继续
while True:
    response = get_response()
    text = response.content[0].text
    if "task complete" in text.lower():  # ❌ 永远不要这样做
        break
    if "I'm done" in text.lower():       # ❌ 自然语言不可靠
        break
```

```python
# 错误：用固定迭代次数作为主要停止机制
for i in range(10):   # ❌ 不应该靠次数限制
    response = ...
```

---

## D1.2 Multi-Agent Orchestration

### Hub-and-Spoke 架构

中心 Coordinator 委派任务给专职 Subagent，每个 Subagent 有独立的 context。

**关键原则**：
- Coordinator 使用 `Task` tool 创建 Subagent（`allowedTools` 必须包含 `Task`）
- 单个 response 中多个 `Task` 调用 = 并行执行
- 传给 Subagent 的 context 只包含其任务相关的最小信息

### 规范代码

```python
from claude_agent import Agent, Task

coordinator = Agent(
    model="claude-sonnet-4-20250514",
    tools=[Task, summarize_results, format_report],  # 3 个工具
)

market_researcher = Agent(
    model="claude-sonnet-4-20250514",
    tools=[web_search, read_doc, extract_data, format_citation],  # 4 个工具
)

tech_analyst = Agent(
    model="claude-sonnet-4-20250514",
    tools=[read_code, grep_patterns, analyze_deps, format_report],  # 4 个工具
)

# ✅ 只传递相关 context，不传完整历史
coordinator.run("""
Research AI infrastructure market. Delegate:
1. Market research → market_researcher（Focus: market size, YoY growth, top vendors）
2. Technology analysis → tech_analyst（Focus: tech stack, dependencies, maturity）
Pass each subagent ONLY the context relevant to their task.
""")
```

### ❌ Anti-Pattern

```python
# 错误：把完整 coordinator 历史传给 subagent
Task(
    prompt="Research market size",
    context=coordinator.full_conversation_history,  # ❌ 90% 都是不相关噪音
)

# 错误：过度细分任务导致 coverage gaps
# 正确：任务拆分粒度要合理，确保没有遗漏区域
```

---

## D1.3 Hooks & Programmatic Enforcement

### Hook vs Prompt 的选择原则

| 场景 | 使用 Hook（代码强制） | 使用 Prompt（软引导） |
|------|---------------------|-------------------|
| 关键业务规则（金额限制、权限） | ✅ | ❌ |
| 合规强制执行 | ✅ | ❌ |
| 风格偏好、格式建议 | ❌ | ✅ |
| Soft preference | ❌ | ✅ |

**Hooks 类型**：
- `PostToolUse`：在工具执行后拦截/修改输出，用于数据标准化、合规检查

### 规范代码

```python
from claude_agent import Agent, Hook

def refund_limit_hook(tool_name, tool_input, tool_output):
    if tool_name == "process_refund":
        amount = tool_input.get("amount", 0)
        if amount > 500:
            return {
                "blocked": True,
                "reason": f"Refund ${amount} exceeds $500 limit",
                "action": "escalate_to_human",
                "context": {
                    "customer_id": tool_input.get("customer_id"),
                    "requested_amount": amount,
                    "agent_limit": 500,
                }
            }
    return tool_output  # 放行其他调用

agent = Agent(
    model="claude-sonnet-4-20250514",
    tools=[lookup_customer, check_order, process_refund],
    hooks={"PostToolUse": [refund_limit_hook]},
)
```

### ❌ Anti-Pattern

```python
# 错误：把关键规则放 Prompt 里（概率性，模型可能不遵守）
system_prompt = """
IMPORTANT: Never process refunds above $500.
If a refund is above $500, escalate to a human.
"""
# 这是概率性的——模型可能偶尔无视这条指令
```

### 有效的升级触发条件

✅ 有效触发：
- 用户明确要求转人工
- 情况超出 Policy 覆盖范围（Policy Gap）
- 金额/权限超出 Agent 阈值
- 重试次数耗尽

❌ 无效触发（Anti-Pattern）：
- 用户情绪为负面（sentiment != complexity）
- 模型自报置信度 < 阈值（模型自报置信度不可靠）

---

## D1.4 Session Management & Workflows

### Session 操作

```bash
# 恢复上次 session（保留完整 context）
claude --resume

# 恢复指定命名 session
claude --resume --session-name "feature-auth-redesign"

# Fork session（继承 context，分叉，不影响主 session）
claude fork_session --reason "Exploring alternative API"

# 新建命名 session
claude --session-name "sprint-47-backend"
```

| 操作 | 场景 |
|------|------|
| `--resume` | 继续之前的工作，需要完整历史 |
| `fork_session` | 探索性实验，不想污染主 context |

### 任务分解策略

| 策略 | 适用场景 |
|------|---------|
| Prompt Chaining（静态链） | 流程固定、每步输入输出可预期 |
| Dynamic Adaptive Decomposition | 任务复杂度未知、中间结果会改变后续方向 |

```python
# ✅ 动态自适应分解（优先）
agent.run("""
Analyze the codebase for issues. For each:
1. Assess severity and complexity
2. If simple: fix directly
3. If complex: create a plan first
4. After each fix: run relevant tests
Adapt your approach based on what you find.
""")
```

### Stale Context 风险

长 session 中，早期获取的数据可能已过时。缓解方法：
- 定期重新获取关键数据
- 使用 scratchpad 文件存储中间状态
