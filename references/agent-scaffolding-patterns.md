# Agent 脚手架设计范式与框架选型

本文件用于回答“我要做 Agent，应该用什么脚手架和框架？”的问题。先根据任务复杂度选脚手架，再叠加生产级能力：权限、工具策略、结构化错误、审计、观测、测试和回滚。

框架参考来源：
- OpenAI Agents SDK：适合让 runtime 管理 turns、tools、guardrails、handoffs、sessions；如果要完全自管 loop，可直接用 Responses API。
- LangGraph：适合状态图、条件跳转、循环、human-in-the-loop 和稳定业务流程。
- AutoGen：适合可组合、多 Agent、事件驱动或分布式 Agent 系统。
- CrewAI：适合 crews、tasks、flows 形式的角色协作和流程化多 Agent 应用。

---

## 快速选型矩阵

| 需求复杂度 | 推荐脚手架 | 推荐框架 | 不建议 |
| --- | --- | --- | --- |
| 简单问答、少量工具 | Perceive-Reason-Act / 最小 ReAct | Responses API / OpenAI Agents SDK | 一上来用多 Agent |
| 需要搜索、工具调用、分步观察 | ReAct | OpenAI Agents SDK / LangGraph 简化图 | 静态 prompt chain |
| 长任务、项目级目标 | Plan & Execute | OpenAI Agents SDK sessions / LangGraph | 只用一次性 prompt |
| 固定流程、审批、失败回退 | State Machine / LangGraph | LangGraph | 无状态 ReAct |
| 多能力客服或中台入口 | Router Agent | OpenAI Agents SDK handoffs / LangGraph | 所有工具挂给一个 Agent |
| 多角色协作 | Orchestrator-Workers / Role-Based | OpenAI Agents SDK handoffs / AutoGen / CrewAI | 子 Agent 共享完整上下文 |
| 方案评审、复杂决策 | Multi-Agent Debate | AutoGen / 自定义编排 | 高频生产主链路 |
| 前沿自治研究 | Swarm Agent | AutoGen Core / 自研 runtime | 商用 v1 |
| 商用生产 | 任一脚手架 + 工程化能力 | 按场景选择 | 没有权限、审计、测试 |

---

## 1. Perceive-Reason-Act 感知-思考-行动

适用场景：聊天机器人、简单问答、少量工具、单轮或短对话任务。  
不适用场景：长任务、复杂审批、多角色协作、需要稳定回退的业务流程。  
复杂度等级：低。  
推荐框架：Responses/Chat API 自管 loop；需要 managed tools 时用 OpenAI Agents SDK。

最小代码示例：

```python
def run_pra_agent(user_input: str, tools: dict) -> dict:
    perception = {"user_input": user_input, "tool_state": get_tool_state()}
    decision = llm_reason(
        system="Decide whether to answer directly or call one safe tool.",
        context=perception,
    )
    if decision["action"] == "answer":
        return {"ok": True, "answer": decision["answer"]}
    result = tools[decision["tool"]](**decision["args"])
    return {"ok": True, "answer": llm_final_answer(user_input, result)}
```

升级路径：PRA → ReAct → Plan & Execute。  
Review 检查点：是否把感知、推理、行动边界分开；工具失败是否结构化返回；是否避免让模型直接执行高风险动作。

## 2. ReAct 思考-行动-观察循环

适用场景：联网搜索、工具调用、分步解题、需要根据工具结果继续决策。  
不适用场景：严格审批流、固定状态流、需要强 SLA 的不可逆操作。  
复杂度等级：低到中。  
推荐框架：OpenAI Agents SDK；复杂状态可升级 LangGraph。

最小代码示例：

```python
def run_react(messages: list[dict], model, executor, max_steps: int = 6):
    for _ in range(max_steps):
        response = model.complete(messages, tools=executor.schemas())
        if response.stop_reason == "end_turn":
            return {"ok": True, "answer": response.text}
        if response.stop_reason != "tool_use":
            return {"ok": False, "error_category": "unexpected_stop"}
        for call in response.tool_calls:
            observation = executor.call(call.name, call.args)
            messages.append({"role": "tool", "tool_call_id": call.id, "content": observation})
    return {"ok": False, "error_category": "iteration_budget_exhausted"}
```

升级路径：ReAct → Tool Guardrails → Plan & Execute / LangGraph。  
Review 检查点：必须用结构化 stop signal；必须有 iteration budget；工具结果要正确回填。

## 3. Plan & Execute 计划-执行

适用场景：长任务、项目级 Agent、需要先拆解任务再逐步完成。  
不适用场景：简单问答、小工具调用、实时低延迟聊天。  
复杂度等级：中。  
推荐框架：OpenAI Agents SDK sessions/handoffs；复杂分支用 LangGraph。

最小代码示例：

```python
def plan_and_execute(goal: str, executor) -> dict:
    plan = llm_reason("Break the goal into ordered, testable steps.", {"goal": goal})
    results = []
    for step in plan["steps"]:
        result = executor.run(step)
        results.append({"step": step, "result": result})
        if not result["ok"] and not result.get("retryable"):
            return {"ok": False, "failed_step": step, "results": results}
    return {"ok": True, "summary": llm_summarize(results), "results": results}
```

升级路径：Plan & Execute → State Machine / LangGraph。  
Review 检查点：计划是否可验证；执行失败是否能停止或重规划；是否记录每步结果。

## 4. 三层记忆脚手架

适用场景：需要保留当前会话、近期任务状态、长期用户偏好或历史任务。  
不适用场景：一次性任务、无需个性化、敏感数据不可持久化的场景。  
复杂度等级：中。  
推荐框架：自研 memory provider；CrewAI knowledge/memory；OpenAI Agents SDK sessions 负责会话层。

最小代码示例：

```python
class MemoryStack:
    def __init__(self, long_term_store):
        self.instant: list[dict] = []
        self.short_term: dict = {}
        self.long_term_store = long_term_store

    def build_context(self, user_id: str) -> dict:
        return {
            "instant": self.instant[-12:],
            "short_term": self.short_term,
            "long_term": self.long_term_store.recall(user_id),
        }
```

升级路径：三层记忆 → 记忆检索 + 召回 → Memory Manager 生命周期。  
Review 检查点：长期记忆是否有来源；敏感信息是否禁止写入；召回内容是否包装为不可信数据。

## 5. 记忆检索 + 召回

适用场景：RAG+Agent、用户画像、历史任务复用、知识库问答。  
不适用场景：实时强一致决策、没有来源和更新时间的长期记忆。  
复杂度等级：中。  
推荐框架：向量库 + 自研 recall；CrewAI knowledge；LangGraph 节点化检索。

最小代码示例：

```python
def remember(store, user_id: str, text: str, source: dict):
    if contains_secret(text):
        return {"ok": False, "error_category": "sensitive_memory_denied"}
    vector = embed(text)
    store.upsert({"user_id": user_id, "text": text, "vector": vector, "source": source})
    return {"ok": True}

def recall(store, user_id: str, query: str) -> str:
    hits = store.search(user_id=user_id, vector=embed(query), limit=5)
    return wrap_untrusted("memory_recall", hits)
```

升级路径：检索召回 → provenance → conflict resolution → human review。  
Review 检查点：是否区分事实和推断；是否记录 retrieved_at；召回是否防 prompt injection。

## 6. Router Agent 路由分发

适用场景：客服 Agent、多能力聚合中台、入口统一但能力模块不同。  
不适用场景：只有一个能力、低流量简单 bot。  
复杂度等级：中。  
推荐框架：OpenAI Agents SDK handoffs；LangGraph 条件边；自研分类器。

最小代码示例：

```python
ROUTES = {
    "order": order_agent,
    "refund": refund_agent,
    "email": email_agent,
    "fallback": human_handoff_agent,
}

def route_request(user_input: str) -> dict:
    route = classify_intent(user_input, allowed=list(ROUTES))
    if route not in ROUTES:
        route = "fallback"
    return ROUTES[route].run({"user_input": user_input})
```

升级路径：规则路由 → 分类器路由 → LLM 语义路由 → handoff 编排。  
Review 检查点：路由置信不足是否转人工；是否避免把所有工具暴露给入口 Agent。

## 7. State Machine Agent 状态机

适用场景：审批、引导式对话、退款流程、KYC、业务流程 Agent。  
不适用场景：开放式探索、任务路径高度不确定的研究任务。  
复杂度等级：中到高。  
推荐框架：LangGraph；自研状态机。

最小代码示例：

```python
TRANSITIONS = {
    "collect_order": {"ok": "check_policy", "missing": "ask_user"},
    "check_policy": {"eligible": "approval", "ineligible": "final_answer"},
    "approval": {"approved": "execute_refund", "rejected": "final_answer"},
}

def next_state(state: str, event: str) -> str:
    return TRANSITIONS[state].get(event, "human_review")
```

升级路径：状态机 → LangGraph 条件边 → human-in-the-loop。  
Review 检查点：是否定义失败回退；未知状态是否 fail closed；高风险状态是否人工确认。

## 8. LangGraph 图编排

适用场景：复杂 Agent、循环、条件跳转、并行、稳定业务流程。  
不适用场景：简单单轮问答或 PoC。  
复杂度等级：高。  
推荐框架：LangGraph。

最小代码示例：

```python
def plan_node(state): return {**state, "plan": make_plan(state["goal"])}
def tool_node(state): return {**state, "tool_result": run_tool(state["plan"][0])}
def review_node(state): return {**state, "approved": needs_no_human_review(state)}

graph = StateGraph()
graph.add_node("plan", plan_node)
graph.add_node("tool", tool_node)
graph.add_node("review", review_node)
graph.add_edge("plan", "tool")
graph.add_conditional_edges("review", lambda s: "end" if s["approved"] else "human")
```

升级路径：State Machine → LangGraph → durable execution / checkpoint。  
Review 检查点：图节点是否职责单一；边界是否可测试；循环是否有预算。

## 9. Orchestrator-Workers 主从分层

适用场景：任务可拆分、子任务专业性强、需要汇总和纠错。  
不适用场景：简单任务、强顺序业务流程。  
复杂度等级：中到高。  
推荐框架：OpenAI Agents SDK handoffs；AutoGen；CrewAI。

最小代码示例：

```python
WORKERS = {"research": research_agent, "code": coder_agent, "review": reviewer_agent}

def orchestrate(goal: str) -> dict:
    assignments = planner.assign(goal, workers=list(WORKERS))
    reports = []
    for item in assignments:
        worker = WORKERS[item["worker"]]
        reports.append(worker.run({"task": item["task"], "context": item["minimal_context"]}))
    return synthesize_reports(reports)
```

升级路径：Router → Orchestrator-Workers → Role-Based / Debate。  
Review 检查点：子 Agent 是否只拿最小上下文；失败是否结构化上报；汇总是否标注来源。

## 10. Role-Based Multi-Agent 角色分工

适用场景：研究、编码、评审、写作、报告等角色明确的工作流。  
不适用场景：角色边界模糊、需要单一责任人的高风险动作。  
复杂度等级：高。  
推荐框架：AutoGen AgentChat；CrewAI crews/tasks；OpenAI Agents SDK handoffs。

最小代码示例：

```python
agents = {
    "planner": Agent(role="Planner", tools=[make_plan]),
    "researcher": Agent(role="Researcher", tools=[web_search, cite]),
    "reviewer": Agent(role="Reviewer", tools=[check_risks]),
    "reporter": Agent(role="Reporter", tools=[write_report]),
}

def role_pipeline(topic: str):
    plan = agents["planner"].run(topic)
    evidence = agents["researcher"].run(plan)
    review = agents["reviewer"].run(evidence)
    return agents["reporter"].run({"evidence": evidence, "review": review})
```

升级路径：Orchestrator-Workers → Role-Based → Debate for review only。  
Review 检查点：角色职责是否互斥；是否避免同 session 自我 review；是否有最终 owner。

## 11. Multi-Agent Debate 对话协作

适用场景：复杂决策、方案评审、论文润色、风险识别、架构权衡。  
不适用场景：高频生产主链路、低延迟任务、需要确定性流程的审批。  
复杂度等级：高。  
推荐框架：AutoGen teams；自研多评审 Agent。

最小代码示例：

```python
def debate(question: str, agents: list, judge) -> dict:
    arguments = [agent.run({"question": question}) for agent in agents]
    critiques = [agent.run({"question": question, "others": arguments}) for agent in agents]
    decision = judge.run({"question": question, "arguments": arguments, "critiques": critiques})
    return {"arguments": arguments, "critiques": critiques, "decision": decision}
```

升级路径：Role-Based → Debate → human decision。  
Review 检查点：是否限制轮次；是否有独立 judge；是否把 debate 用在非关键路径。

## 12. Swarm Agent 群体自治

适用场景：前沿研究、开放式探索、大规模任务认领实验。  
不适用场景：商用 v1、合规流程、强审计和强权限场景。  
复杂度等级：实验级。  
推荐框架：AutoGen Core / 自研 runtime；默认不推荐生产。

最小代码示例：

```python
class Blackboard:
    tasks: list[dict] = []
    results: list[dict] = []

def swarm_step(agent, board: Blackboard):
    task = agent.claim(board.tasks)
    if not task:
        return None
    result = agent.run(task)
    board.results.append({"agent": agent.name, "task": task, "result": result})
```

升级路径：先用 Orchestrator-Workers，只有研究需求再试 Swarm。  
Review 检查点：是否有全局预算；是否有冲突解决；是否有权限隔离和审计。

## 13. 工具插件化

适用场景：工具会持续增加、需要插件扩展、需要工具 schema 和权限统一管理。  
不适用场景：只有 1-2 个固定工具的小脚本。  
复杂度等级：中。  
推荐框架：OpenAI Agents SDK function tools；MCP；自研 registry。

最小代码示例：

```python
class ToolRegistry:
    def __init__(self):
        self.tools = {}

    def register(self, name: str, schema: dict, handler, risk: str):
        self.tools[name] = {"schema": schema, "handler": handler, "risk": risk}

    def call(self, user, session, name: str, args: dict):
        spec = self.tools[name]
        if not policy_allows(user, session, name, spec["risk"]):
            return {"ok": False, "error_category": "permission_denied"}
        return spec["handler"](**args)
```

升级路径：本地 registry → MCP manager → tool marketplace。  
Review 检查点：工具是否有 schema、risk、timeout、output limit、audit 字段。

## 14. Self-Reflection 反思自省

适用场景：草稿质量检查、代码/文案自检、结果修正。  
不适用场景：安全审批、权限判断、金融动作等必须确定性拦截的场景。  
复杂度等级：中。  
推荐框架：任意框架；生产中应作为辅助 pass，不替代测试和人工确认。

最小代码示例：

```python
def reflect_and_fix(task: str, draft: str) -> dict:
    critique = llm_reason("Find concrete issues against the rubric.", {"task": task, "draft": draft})
    if not critique["issues"]:
        return {"ok": True, "output": draft}
    revised = llm_reason("Revise only the listed issues.", {"draft": draft, "issues": critique["issues"]})
    return {"ok": True, "output": revised, "critique": critique}
```

升级路径：Self-Reflection → validation-retry → independent review session。  
Review 检查点：是否有明确 rubric；是否限制修正范围；是否不把自省当安全边界。

## 15. Observability 观测与可观测

适用场景：企业级 Agent、接真实用户、接外部工具、需要追责和排障。  
不适用场景：仅本地一次性实验可以简化，但也建议保留最小日志。  
复杂度等级：中到高。  
推荐框架：框架无关；OpenTelemetry / 自研 audit log；LangGraph/AutoGen/CrewAI 事件可接入日志。

最小代码示例：

```python
def audit_event(event: str, **fields) -> dict:
    return {
        "event": event,
        "session_id": fields["session_id"],
        "agent_id": fields.get("agent_id"),
        "tool_name": fields.get("tool_name"),
        "status": fields.get("status"),
        "error_category": fields.get("error_category"),
        "latency_ms": fields.get("latency_ms"),
        "provenance": fields.get("provenance", []),
    }
```

升级路径：print log → structured audit → trace/span → dashboard/alerts。  
Review 检查点：是否记录 session/user/tool/status/error/latency；是否脱敏；是否能复盘失败。

---

## 输出架构方案时的推荐格式

```markdown
### 推荐脚手架
[脚手架名称]，因为 [用户需求信号]。

### 推荐框架
[框架名称]，因为 [运行时/编排/多 Agent/状态流需求]。

### 为什么不选更复杂方案
[说明暂不使用 LangGraph/多 Agent/Swarm 等的理由]。

### v1 最小组件
[loop/tool/memory/policy/tests/logs]
```

