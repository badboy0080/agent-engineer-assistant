# Hermes Agent / Odysseus 调研来源记录

调研日期：2026-07-03  
用途：为 `agent-engineer-assistant` skill 升级提供最新工程实践参考。  
注意：以下结论只代表截至调研日期的仓库状态。后续使用“最新”判断时，必须重新联网核对。

---

## 1. Hermes Agent

- 仓库：[NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)
- 分支：`main`
- 调研时最新提交：`89acc196067c`
- 提交时间：2026-07-03T00:52:18Z
- 提交信息：`fix(dump): flag API keys visible only to the shell, not the managed backend`

重点文件：
- `agent/conversation_loop.py`：conversation loop、retry、failover、compression、memory/skill hooks。
- `agent/tool_executor.py`：顺序/并发工具执行、middleware、guardrail、timeout、tool result budget。
- `agent/tool_guardrails.py`：重复工具调用、失败路径识别、guardrail synthetic result。
- `agent/memory_manager.py`：memory provider 注册、prefetch、sync、session switch、write mirror。
- `agent/context_engine.py`：可插拔 context engine 抽象。
- `agent/skill_preprocessing.py`：SKILL.md 模板变量、inline shell 展开、timeout 和输出限制。
- `agent/learning_graph.py`：skill usage、related skills、memory 与 skill 的学习图谱。
- `agent/transports/hermes_tools_mcp_server.py`：将 Hermes 工具精选暴露为 MCP server。

提炼出的工程模式：
- Agent loop 需要预算、失败分类、message sanitization、context compression 和 post-turn hooks。
- 工具执行器应统一处理权限、guardrail、timeout、输出预算、并发和审计。
- Memory 应做成 provider 生命周期，而不是单一文本追加。
- Context 管理应可插拔，并能随模型 context length 调整。
- Skill 系统应具备 metadata、预处理、使用统计和关联能力。
- MCP bridge 应暴露精选能力，避免把有状态 loop 工具直接给 stateless MCP callback。

## 2. Odysseus

- 仓库：[pewdiepie-archdaemon/odysseus](https://github.com/pewdiepie-archdaemon/odysseus)
- 分支：`dev`
- 调研时最新提交：`8c943226f815`
- 提交时间：2026-07-02T15:28:23Z
- 提交信息：`fix(mobile): stack the model-comparison grid into one column on phones (#4979)`

重点文件：
- `THREAT_MODEL.md`：信任边界、admin/non-admin、内部 loopback、prompt injection hardening、已知安全 gap。
- `src/agent_loop.py`：工具调用 prompt、工具执行轮次、untrusted context、tool policy 接入。
- `src/tool_policy.py`：guide-only / no-tools 检测、per-turn 工具策略。
- `src/tool_security.py`：non-admin blocked tools、Plan Mode 只读 allowlist、MCP/email 工具别名策略。
- `src/tool_execution.py`：工具分发、文件路径 allowlist/denylist、workspace containment、admin tool gate。
- `src/prompt_security.py`：不可信内容包装、guard marker 转义。
- `src/context_budget.py`：按模型 context window 自适应输入预算。
- `src/mcp_manager.py`：MCP transport、schema 清洗、read-only 判断、连接错误提示。
- `src/tool_schemas.py`：OpenAI-compatible function tool schemas 与工具描述。
- `services/memory/service.py`：长期记忆服务、向量检索、remember/recall/delete。

提炼出的工程模式：
- 自托管 Agent 产品应当按“管理后台”思路设计权限，而不是只按聊天机器人设计。
- Plan Mode 应用只读工具 allowlist，而不是 prompt 软约束。
- 第三方 MCP schema 属于不可信输入，渲染到 prompt 前要限制长度和参数数量。
- 文件路径工具要先 deny 敏感路径，再检查 allowlist/workspace。
- admin 工具必须在 agent tool dispatch 层按 owner/session 校验。
- 外部内容、邮件、记忆、skill 文本和工具输出都要包装成不可信数据。

## 3. 本次未做的事

- 未复制两个仓库的大段源码。
- 未判断两个项目的商业、许可证或安全成熟度。
- 未将两个项目视为唯一正确实践，只把其中高价值工程模式沉淀为检查清单。

## 4. 后续更新建议

当用户再次要求“最新 Agent 构建规范”时，应重新核对：
- 两个仓库默认分支和最新提交。
- OpenAI Agents SDK、LangGraph、MCP 规范、Claude Code/Codex 工具策略的最新官方文档。
- 当前项目或团队实际使用的 runtime、模型、MCP server 和权限边界。
