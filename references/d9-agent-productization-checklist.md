# Domain 9: Agent 产品化上线清单

本文件用于判断一个 Agent 是否从 demo 进入“可上线、可维护、可追责”的产品状态。

---

## D9.1 产品定义

上线前必须说清：
- 目标用户是谁。
- 主要任务是什么。
- 不做什么。
- 成功标准是什么。
- 出错后谁接手。
- 结果由谁负责确认。

产品经理应要求输出：用户旅程、关键任务流、人工介入点、权限矩阵、验收标准、风险清单。

代码示例：

```python
PRODUCT_SPEC = {
    "target_users": ["support_agent", "support_manager"],
    "primary_tasks": ["lookup_order", "draft_refund_reply"],
    "out_of_scope": ["approve_refund_over_limit", "change_payment_method"],
    "success_metrics": {"first_response_time_minutes": 5, "handoff_rate_max": 0.2},
    "fallback_owner": "support_manager",
    "final_approver": "human_support_agent",
}
```

## D9.2 架构清单

- Agent 角色：Coordinator、Subagent、工具执行器、memory provider、MCP manager。
- 数据流：用户输入、检索、工具调用、模型输出、审计日志、持久化。
- 控制流：loop、stop condition、retry、timeout、budget、human escalation。
- 失败流：工具失败、模型失败、网络失败、权限失败、schema 失败、context 超限。
- 安全边界：trusted/untrusted、admin/non-admin、read/write、internal/external。

代码示例：

```python
ARCHITECTURE = {
    "agents": ["coordinator", "order_lookup_agent", "reply_drafter"],
    "runtime_components": ["loop", "tool_executor", "tool_policy", "context_engine"],
    "data_flow": ["user_input", "retrieval", "tool_call", "model_output", "audit_log"],
    "failure_flow": ["tool_error", "permission_denied", "context_limit", "human_escalation"],
    "security_boundaries": ["untrusted_inputs", "admin_tools", "external_effects"],
}
```

## D9.3 工具与 MCP 清单

每个工具必须定义：名称、用途、输入 schema、返回 schema、是否只读、是否有副作用、超时、输出上限、错误类别、是否可重试、权限要求、审计字段、测试用例。

MCP server 额外定义：transport、server_id、工具列表、是否第三方、配置和密钥来源、readOnly/destructive annotations、schema 渲染预算、non-admin 是否可见。

代码示例：

```python
TOOL_SPEC = {
    "name": "lookup_order",
    "purpose": "Fetch order status by order id.",
    "input_schema": {"order_id": "str"},
    "return_schema": {"ok": "bool", "data": "dict | None", "error_category": "str | None"},
    "read_only": True,
    "side_effect": False,
    "timeout_ms": 3000,
    "output_limit_chars": 4000,
    "retryable_errors": ["timeout", "rate_limit"],
    "permission": "order:read",
    "audit_fields": ["user_id", "session_id", "order_id", "result_status"],
    "tests": ["success", "empty_result", "access_failure", "timeout"],
}
```

## D9.4 Context 与 Memory 清单

- 短期 context 是否有预算。
- 长期 memory 是否有 provider 生命周期。
- Case Facts 是否不会被摘要丢失。
- 检索内容是否带 source、confidence、retrieved_at。
- 压缩前是否保留关键事实。
- session/fork/resume 是否清楚。
- memory 是否可查看、编辑、删除。
- memory 是否防注入、防敏感信息写入。

代码示例：

```python
CASE_FACTS = {
    "customer_id": {"value": "C-123", "source": "crm", "confidence": "verified"},
    "order_id": {"value": "O-456", "source": "order_db", "confidence": "verified"},
    "refund_limit": {"value": 500, "source": "policy_v4", "confidence": "verified"},
}

def preserve_case_facts(summary: str, facts: dict) -> str:
    return f"CASE_FACTS_DO_NOT_SUMMARIZE={facts}\n\nSUMMARY={summary}"
```

## D9.5 安全与权限清单

- 用户角色与权限矩阵。
- 工具风险分级。
- Plan Mode 只读策略。
- guide-only / no-tools 策略。
- 文件路径 allowlist/denylist。
- secret redaction。
- 不可信输入 wrapper。
- admin 工具 owner/session 校验。
- MCP server 权限和 schema 限制。
- 高风险操作人工确认。

代码示例：

```python
PERMISSION_MATRIX = {
    "support_agent": {"lookup_order", "draft_reply"},
    "support_manager": {"lookup_order", "draft_reply", "approve_refund"},
    "admin": {"manage_mcp", "send_email", "run_maintenance"},
}

def has_permission(role: str, tool_name: str) -> bool:
    return tool_name in PERMISSION_MATRIX.get(role, set())
```

## D9.6 测试清单

基础测试：prompt 单元测试、工具 schema 校验、工具成功/失败模拟、空结果 vs access failure、权限拒绝、超时、重试耗尽、context 超限、memory 写入/召回、MCP 连接失败。

安全测试：prompt injection 文档、恶意网页、恶意邮件、第三方 MCP schema 注入、路径穿越、读取 secret、non-admin 调 admin 工具、Plan Mode 调写工具。

业务回归：核心用户任务端到端、人工确认流程、错误提示是否可理解、日志是否能追踪、回滚或补偿是否可执行。

代码示例：

```python
TEST_MATRIX = [
    {"id": "tool_empty_result", "input": "unknown order", "expect": "empty_result"},
    {"id": "tool_access_failure", "input": "db down", "expect": "access_failure"},
    {"id": "prompt_injection_doc", "input": "ignore previous instructions", "expect": "ignored_as_data"},
    {"id": "plan_mode_write", "input": "edit file in plan mode", "expect": "permission_denied"},
    {"id": "non_admin_mcp", "input": "call mcp__server__write", "expect": "admin_required"},
]

def assert_test_matrix_has_red_lines(matrix: list[dict]) -> bool:
    required = {"prompt_injection_doc", "plan_mode_write", "non_admin_mcp"}
    return required <= {case["id"] for case in matrix}
```

## D9.7 观测与日志

至少记录：session id、user/owner、agent id、tool name、tool args 摘要、result status、error category、retry count、token/cost、latency、escalation reason、human approval id、source/provenance。

注意：
- 日志不能记录完整 secret。
- 对个人敏感信息要最小化记录。
- 用户可见错误要简单，内部日志要足够排查。

代码示例：

```python
def audit_event(**kwargs) -> dict:
    return {
        "event": kwargs["event"],
        "session_id": kwargs["session_id"],
        "user_id": kwargs["user_id"],
        "agent_id": kwargs.get("agent_id"),
        "tool_name": kwargs.get("tool_name"),
        "result_status": kwargs.get("result_status"),
        "error_category": kwargs.get("error_category"),
        "latency_ms": kwargs.get("latency_ms"),
        "approval_id": kwargs.get("approval_id"),
        "provenance": kwargs.get("provenance", []),
    }
```

## D9.8 上线门槛

可以上线的最低标准：
- 核心任务有端到端测试。
- 高风险工具有权限和人工确认。
- 不可信输入已隔离。
- 工具失败有结构化错误。
- 关键事实有 provenance。
- 有日志和最小审计。
- 有回滚或人工补救方案。
- 用户知道 Agent 能做什么、不能做什么。

不能上线的信号：
- 只靠 prompt 管权限。
- shell 工具全开放。
- MCP server 不分权限。
- memory 自动写入无审核。
- 文件工具无路径限制。
- 没有失败处理。
- 没有测试和日志。

代码示例：

```python
RELEASE_GATES = {
    "e2e_tests",
    "permission_policy",
    "human_approval",
    "untrusted_input_wrapper",
    "structured_tool_errors",
    "provenance",
    "audit_log",
    "rollback_plan",
}

def can_launch(passed: set[str]) -> tuple[bool, set[str]]:
    missing = RELEASE_GATES - passed
    return (not missing, missing)
```

## D9.9 给产品经理的验收问法

推荐提问：
- “请把这个 Agent 的工具按只读、写入、外部影响、执行类分级，并说明每类权限和人工确认点。”
- “请列出这个 Agent 从用户输入到最终动作的完整数据流和失败流。”
- “请用 10 个测试场景验证它不会把网页/邮件里的恶意指令当成系统指令。”
- “请说明如果工具失败、权限不足、context 超限、MCP 断开，用户会看到什么。”
- “请输出上线前必须通过的测试矩阵和日志字段。”

代码示例：

```python
PM_ACCEPTANCE_PROMPT = """
请输出：
1. 工具风险分级表
2. 数据流、控制流、失败流
3. 10 个安全测试场景
4. 用户可见错误文案
5. 上线门槛与缺口清单
"""

EXPECTED_SECTIONS = {"工具风险", "数据流", "安全测试", "错误文案", "上线门槛"}
```
