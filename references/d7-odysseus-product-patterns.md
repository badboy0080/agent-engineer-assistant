# Domain 7: Odysseus 产品化工程模式

调研对象：`pewdiepie-archdaemon/odysseus`，`dev` 分支。  
调研时间：2026-07-03。  
定位：Odysseus 更像一个自托管、本地优先的 AI 工作台。它的价值在于展示 Agent 产品化时如何处理权限、工具策略、MCP、RAG、memory、邮件、文件、UI、服务端路由和安全边界。

---

## D7.1 产品形态

Odysseus 不是单个 Agent 脚本，而是完整应用：

- FastAPI/后端路由；
- 静态前端；
- 本地文件/文档/邮件/图库/任务/记忆；
- MCP servers；
- Codex/Claude 集成；
- RAG/search/research 服务；
- shell、文件、模型 serving 等高权限工具；
- admin/non-admin 用户模型；
- Docker、桌面启动、CI、安全扫描。

可迁移规范：
- 只要 Agent 能读写本地文件、发邮件、执行 shell、访问 MCP，就应按“管理后台”思路设计权限。
- Agent 产品要把工具层、路由层、用户层、审计层一起设计，而不是只设计模型 prompt。

## D7.2 Tool Policy

Odysseus 的 `src/tool_policy.py` 把每个 turn 的工具策略组合成一个 `ToolPolicy`：

- disabled_tools；
- hidden_tools；
- guide_only mode；
- block_all_tool_calls；
- reason_for(tool)；
- 根据用户“不要用工具”等表达检测 guide-only turn。

可迁移规范：
- 用户要求“不要执行/只分析”时，系统应真的隐藏或禁用工具。
- 工具禁用要影响 prompt 中可见工具，也要影响实际 dispatch gate。
- 每个被拦截的工具调用都应有原因，便于模型和用户理解。

## D7.3 Plan Mode 只读工具

Odysseus 的 `src/tool_security.py` 使用 allowlist 实现 Plan Mode：

- 只允许明确读工具；
- 写文件、发邮件、manage_*、模型 serving、MCP 等默认禁用；
- schema 加载失败时用静态 mutator 列表兜底；
- 对未知/模糊工具 fail closed。

可迁移规范：
- Plan Mode 应是“默认禁止写，显式允许读”。
- 新增工具时，如果没有声明 read-only，就应默认视为高风险。
- 对 MCP 工具，如果没有 `readOnlyHint`，可用名称动词做保守判断，但模糊时必须当作 mutating。

## D7.4 Admin / Non-admin 权限边界

Odysseus 明确区分 admin 与 non-admin：

- 邮件、MCP、token、webhook、server 控制等能力对 non-admin 默认禁用。
- 任意 `mcp__` 工具对 non-admin 视为 blocked。
- 即使有内部 loopback token，agent tool dispatch 仍要检查 session owner 是否 admin。

可迁移规范：
- 路由层 admin 校验不等于 Agent 工具层安全。
- 内部工具通道不能绕过 owner/session 权限。
- 所有 admin 工具要能回答：谁调用、从哪个 session、调用了什么、结果是什么。

## D7.5 Prompt Injection Hardening

Odysseus 的 `src/prompt_security.py` 把外部内容包装为不可信数据：

- web results；
- fetched pages；
- emails；
- transcripts；
- tool output；
- saved memories；
- skill text；
- notes。

核心做法：
- 用固定 guard marker 包住不可信内容。
- 在 system prompt 中声明外部内容只是数据，不是指令。
- 对 guard marker 做转义，防止外部文本提前关闭边界。
- label 也要清洗，不能让 label 自己变成注入载体。

可迁移规范：
- 不可信内容进入 LLM 前必须有边界。
- 不要把外部内容直接拼进 system role。
- 不要在最终回答里暴露 wrapper 文案，除非用户明确问 prompt 安全。

## D7.6 文件与路径安全

Odysseus 的 `src/tool_execution.py` 对文件工具做了多层路径策略：

1. 解析真实路径。
2. 检查敏感子路径 denylist。
3. 检查 allowlist 根目录。
4. 支持 active workspace containment。
5. 对大小写绕过做防护。

可迁移规范：
- 模型提供的 path 永远不能直接执行读写。
- denylist 要先于 allowlist，避免允许根目录里包含 `.ssh`、`.gnupg`、env、token 等敏感文件。
- workspace 是每个 turn 的上下文状态，不应散落在各工具自己判断。

## D7.7 MCP Manager

Odysseus 的 `src/mcp_manager.py` 支持 stdio、SSE、Streamable HTTP 等 MCP transport，并记录工具 schema。

关键工程点：
- MCP 连接错误要返回可执行安装/修复建议。
- 第三方 MCP schema 渲染到 prompt 前要做字段长度和参数数量限制。
- 优先使用 MCP annotations，例如 `readOnlyHint`、`destructiveHint`。
- 没有 hint 时用名称动词做只读判断，但默认保守。

可迁移规范：
- MCP server 是第三方扩展面，默认不可信。
- MCP 工具不只要“能连上”，还要被权限、Plan Mode、schema 预算和审计系统管住。

## D7.8 Context Budget

Odysseus 的 `src/context_budget.py` 根据模型 context window 动态计算输入 token budget：

- 用户显式设置预算时尊重用户设置。
- 自动模式按 context window 的比例扩展，但有硬上限。
- context length 未知时用保守默认值。

可迁移规范：
- 长上下文模型不应被固定小预算浪费能力。
- 但也不能因为模型有 1M context 就无限塞材料。
- 预算要区分显式配置和自动推导。

## 规划时怎么使用 D7

当用户要做“本地 AI 工作台”“私有化 Agent 产品”“能读写文件/发邮件/接 MCP 的 Agent”时，必须覆盖：
- 用户角色和权限矩阵。
- 工具策略和 Plan Mode 工具白名单。
- 文件路径 allowlist/denylist。
- 不可信输入包装。
- MCP server 管理、schema 限制和只读/写判断。
- admin 工具的 owner/session 校验。
- 日志、审计、错误回放和用户可见提示。
