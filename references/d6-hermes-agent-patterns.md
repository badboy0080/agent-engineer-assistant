# Domain 6: Hermes Agent 工程模式

调研对象：`NousResearch/hermes-agent`，`main` 分支。  
调研时间：2026-07-03。  
定位：Hermes Agent 更像一个完整 Agent runtime，而不是单个 prompt 或简单 tool loop。它的价值在于把 loop、工具执行、预算、guardrails、memory、context engine、skill 和 MCP bridge 都做成可演进系统。

---

## D6.1 Conversation Loop

Hermes 的 `agent/conversation_loop.py` 把一次用户 turn 拆成一组稳定阶段：

1. 构建或恢复 system prompt。
2. 做 message sanitization 和 tool call argument repair。
3. 根据 context 状态做 preflight compression。
4. 调用模型，并处理 retry、failover、内容安全、输出截断。
5. 执行工具调用，把结果回填到 messages。
6. 做 memory/skill review nudges、post-turn hooks 和持久化。

可迁移规范：
- loop 不应只写成 `while True`；必须有迭代预算、失败退出原因和可观测日志。
- 模型调用前要修复或清洗消息结构，避免非法 tool call、非 ASCII/surrogate、孤儿 tool result 破坏下一轮。
- 失败恢复要区分网络、rate limit、context length、billing、content policy、provider failover，而不是统一重试。

Review 检查：
- 是否有最大迭代和预算耗尽路径。
- 是否记录 turn exit reason。
- 是否对 provider 错误分类，并给用户可执行指引。
- 是否防止相同大工具调用反复导致 stream timeout。

## D6.2 Iteration Budget 与 Tool Result Budget

Hermes 使用 `IterationBudget` 控制 agentic loop，工具执行侧还根据 context window 调整工具结果预算。

可迁移规范：
- Agent 不能无限循环，也不能只靠固定次数粗暴退出。
- 最好同时有：总迭代预算、单工具 timeout、并发工具总 deadline、工具输出截断/摘要策略。
- 对长上下文模型，预算应按模型 context window 自适应；对小模型要保守。

Anti-Pattern：
- Agent 重复调用同一工具同一参数，只是因为 prompt 要它“继续努力”。
- 工具输出无限塞回上下文，导致后续模型调用失败。

## D6.3 Tool Guardrails

Hermes 的 `agent/tool_guardrails.py` 体现了一个重要思想：工具 loop 的风险要在代码层识别。

可迁移规范：
- 记录工具名 + canonical args 的调用签名。
- 对相同工具相同参数的重复失败设置阈值。
- 对同一路径重复失败给出替代策略提示，而不是继续原地重试。
- guardrail block 要返回结构化工具结果，让模型知道为什么被阻止。

Review 检查：
- 是否识别重复失败工具调用。
- 是否把 guardrail 信息回填给模型。
- 是否区别“继续用工具诊断”和“停止同样路径重试”。

## D6.4 Tool Executor

Hermes 的 `agent/tool_executor.py` 将工具执行拆成：

- middleware；
- guardrail before_call；
- sequential / concurrent execution；
- timeout；
- cancellation；
- per-turn output budget；
- post-tool terminal event；
- memory write mirror。

可迁移规范：
- 工具执行器不应只是 `dispatch(name, args)`；它要承载权限、预算、timeout、审计和错误分类。
- 并发工具要有总 deadline，不能让某个长工具拖死整个 turn。
- 被 guardrail 阻止的工具不能被当成真实执行成功。
- 工具结果需要截断、摘要或预算 enforcement，防止污染 context。

## D6.5 Memory Provider 与 Memory Manager

Hermes 的 `agent/memory_manager.py` 把 memory 做成 provider 架构：

- provider 注册；
- tool schema 去重；
- reserved core tool name 防冲突；
- prefetch；
- sync_turn；
- on_session_switch；
- on_pre_compress；
- on_memory_write；
- on_delegation；
- bounded background executor；
- shutdown drain。

可迁移规范：
- Memory 是系统能力，不是简单追加一段“用户喜欢什么”的文本。
- 只能有明确选定的外部 memory provider，避免多个后端同时写导致冲突。
- memory provider 暴露的工具名不能覆盖核心工具。
- memory 写入要有成功门禁：只同步已经 commit 的写入，staged 或失败结果不能外传。
- memory context 要带边界包装，避免被当成用户新指令。

Review 检查：
- memory 是否有 provider 生命周期。
- session switch / fork / reset / compress 后，memory provider 是否刷新上下文。
- memory prefetch 失败是否 fail-open，并记录日志。
- memory 写入是否带 provenance。

## D6.6 Context Engine

Hermes 的 `agent/context_engine.py` 把 context 管理抽象成可插拔 engine：

- `should_compress()`
- `compress()`
- `should_compress_preflight()`
- `has_content_to_compress()`
- `reset()`
- `get_status()`
- `on_context_length_changed()`

可迁移规范：
- Context 管理应是一个明确组件，而不是散落在 prompt 里的“请总结”。
- 压缩前要允许 provider 注入不能丢的内容。
- 切换模型 context length 后，预算和阈值要重新计算。
- 手动压缩和自动压缩都要有 preflight，避免无意义压缩。

## D6.7 Skill 系统与学习图谱

Hermes 的 skill 体系包含：

- SKILL.md frontmatter；
- 模板变量替换；
- inline shell 展开但有 timeout 和输出上限；
- skill usage 统计；
- related_skills；
- learned/profile skills；
- memory 与 skill 的关联图谱。

可迁移规范：
- Skill 不只是静态说明书，应能被索引、统计、关联和迭代。
- 自动展开命令必须限制 timeout 和输出长度。
- Skill 文本也是不可信输入之一，尤其当它来自用户或外部仓库。
- Skill 之间应有 related metadata，便于组合使用。

## D6.8 Hermes Tools as MCP Bridge

Hermes 的 `agent/transports/hermes_tools_mcp_server.py` 把 Hermes 工具面通过 MCP 暴露给 Codex 子进程。

可迁移规范：
- MCP bridge 应暴露“精选工具子集”，不是把所有工具都丢给外部 runtime。
- 不适合 stateless MCP callback 的工具要排除，例如依赖 agent loop 状态的 delegation/memory/session 工具。
- MCP stdio server 的日志必须输出到 stderr，避免污染协议 stdout。
- 工具 schema 应来自权威注册表，而不是重复手写两份。

## 规划时怎么使用 D6

当用户要做“像 Hermes 一样的 Agent runtime”时，应优先问清：
- 是否需要多 provider 模型路由和 failover。
- 是否需要长期 memory 和 skill 自演进。
- 是否需要 context engine 可插拔。
- 是否需要把工具暴露给其他 runtime 或 MCP client。
- 是否需要桌面、CLI、server 多入口共享一个工具执行核心。

输出方案时要明确：哪些能力属于 v1 必须做，哪些属于后续 runtime 化演进。
