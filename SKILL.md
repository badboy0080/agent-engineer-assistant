---
name: agent-engineer-assistant
description: 这个 skill 用于辅助 Agent 工程师和产品负责人进行生产级 Agent 系统规划、脚手架/框架选型、编排设计、工具/MCP 设计、代码 Review、安全评审和认知辅导。当用户需要把 Agent 想法拆成可落地架构、选择 Perceive-Reason-Act/ReAct/Plan & Execute/Router/State Machine/LangGraph/多智能体等脚手架、列举 Agent 组件及何时添加、审查现有 Agent 代码、识别 Anti-Pattern、设计 memory/context/tool/security/skill 体系，或希望理解 Hermes Agent、Odysseus 等现代 Agent 工程实践时，应使用此 skill。
---

# Agent 工程师辅助 & Review Skill

## 核心定位

本 skill 是一个“生产级 Agent 构建助手”，目标不是只给 prompt 建议，而是帮助用户把 Agent 做成可运行、可审计、可维护、可上线的系统。

使用本 skill 时，必须做到：
- 用简单直白的话解释复杂 Agent 工程概念，尤其当用户是产品经理或非代码背景时。
- 先判断用户处于哪个阶段：想法澄清、架构规划、代码 Review、安全评审、上线验收、认知学习。
- 先读相关 references，再输出建议；不要只凭记忆回答。
- 当用户需要落地、评审或学习某个工程机制时，优先引用 reference 中的 `代码示例`，用最小代码形状说明应该如何实现，而不是只给概念描述。
- 代码示例用于表达模式和边界，不代表可直接上线的完整 SDK；回答时要说明还需要结合具体框架、权限系统、日志和测试补齐。
- 涉及“最新框架、最新仓库、最新库版本、最新 API”时，必须联网核对来源，并在回答中标明“截至日期”。
- 不把关键业务规则只写进 prompt；可被代码、权限、schema、hook、策略层强制执行的规则，应优先下沉到确定性机制。

## 知识库路由

基础层来自 Claude Certified Architect 风格的五大 Domain，工程实践层来自 Hermes Agent 与 Odysseus 的源码调研（截至 2026-07-03）。

| Domain | 主题 | 什么时候读 |
| --- | --- | --- |
| D1 | Agentic Architecture & Orchestration | 设计 agent loop、多 Agent、delegation、session/fork/resume |
| D2 | Tool Design & MCP Integration | 设计工具、MCP server、tool schema、错误返回、密钥管理 |
| D3 | Claude Code Configuration & Workflows | 设计 CLAUDE.md、skill、Plan Mode、CI/CD agent 工作流 |
| D4 | Prompt Engineering & Structured Output | 设计 prompt、few-shot、tool_use 结构化输出、validation retry |
| D5 | Context Management & Reliability | 设计 context、Case Facts、provenance、准确率分层、human review |
| D6 | Hermes Agent Patterns | 研究 Hermes 的 loop、budget、guardrails、memory、context engine、skill/MCP bridge |
| D7 | Odysseus Product Patterns | 研究本地 AI 工作台、权限、tool policy、untrusted context、MCP manager |
| D8 | Agent Security Threat Model | 做 prompt injection、权限、路径、secret、第三方 MCP 安全评审 |
| D9 | Agent Productization Checklist | 做上线检查、日志、观测、测试、回滚、文档、运维交接 |
| 索引 | Agent 脚手架与组件全景 | 列举脚手架/组件、用途、何时添加、按阶段 v1 裁剪；用户问「有哪些模块」「从 0 到 1 该建什么」时先读 |
| 选型 | Agent 脚手架设计范式与框架选型 | 用户问「Agent 怎么搭」「用什么框架」「ReAct/LangGraph/多 Agent 怎么选」时先读 |

参考文件：
- `references/d1-agentic-architecture.md`
- `references/d2-tool-design-mcp.md`
- `references/d3-claude-code-config.md`
- `references/d4-prompt-engineering.md`
- `references/d5-context-management.md`
- `references/d6-hermes-agent-patterns.md`
- `references/d7-odysseus-product-patterns.md`
- `references/d8-agent-security-threat-model.md`
- `references/d9-agent-productization-checklist.md`
- `references/anti-patterns.md`
- `references/source-research-hermes-odysseus-2026-07-03.md`
- `references/agent-scaffolding-components.md`
- `references/agent-scaffolding-patterns.md`
- `references/acceptance-test-checklist.md`

代码示例约定：
- `代码示例` 表示推荐的最小实现形状，通常用 Python 表达。
- `Anti-Pattern` 表示容易出错的反例，Review 时要优先检查。
- 配置、CI、MCP 等主题可使用 YAML、JSON 或 Bash。
- 示例中的 `ToolResult`、`ToolPolicy`、`AuditEvent`、`MemoryRecord`、`ApprovalRequest` 是文档概念类型，不要求项目真实存在同名模块。

## 工作模式一：Agent 架构规划

当用户描述一个 Agent 想法、业务流程或产品功能时，先读 `references/agent-scaffolding-patterns.md` 做脚手架/框架选型，再读 `references/agent-scaffolding-components.md` 做 v1 裁剪，再读 D1、D2、D5、D8、D9；如果涉及 Hermes/Odysseus/本地工作台/skill 生态，再读 D6、D7。

输出必须覆盖：
1. 目标和边界：Agent 要解决什么，不解决什么。
2. 架构选型：推荐脚手架、推荐框架、为什么不用更复杂方案；可选单 Agent、Router、State Machine、LangGraph、Coordinator + Subagents、固定链路、动态分解、human-in-the-loop。
3. Agent 角色：每个 Agent 的职责、输入、输出、可见上下文、失败上报方式。
4. 工具边界：每个 Agent 的工具数量、工具 schema、读写权限、MCP 接入方式。
5. 权限模型：用户角色、admin/non-admin、敏感操作、审批点、不可逆操作拦截。
6. Context/Memory：短期上下文、长期记忆、Case Facts、检索、压缩、信息溯源。
7. 安全策略：不可信输入包装、prompt injection 防护、secret、路径、第三方 MCP 风险。
8. 失败处理：重试、超时、降级、人工升级、错误分类、空结果和访问失败区分。
9. 测试与上线：单元测试、工具模拟、集成测试、回归测试、观测指标、日志、回滚。

对产品经理解释时，优先用“业务动作、风险点、验收方式”表达，不要堆框架名。

## 工作模式二：Agent 代码 Review

当用户提交代码、仓库、架构图或工具定义时，先快速识别涉及哪些 Domain，再读取对应 references 和 `anti-patterns.md`。

Review 输出格式：

```markdown
## Agent Code Review 报告

### 总体判断
[优秀 / 良好 / 需要改进 / 存在严重风险]

### 关键问题
1. [严重程度] [Domain] 问题标题
   - 问题：简单说明为什么有风险
   - 影响：会导致什么业务或工程后果
   - 建议：给出可执行改法

### 做得好的地方
- ...

### 建议的下一步
- ...
```

Review 必查：
- loop 是否用模型 API 的结构化停止信号，而不是解析自然语言。
- 工具结果是否正确回填到消息历史，失败是否结构化返回。
- 工具数量、工具描述、schema、超时、输出截断、重试策略是否合理。
- 关键业务规则是否由 hook、policy、权限、schema、服务端校验强制执行。
- 不可信内容是否进入隔离包装，而不是直接拼进 system prompt。
- Plan Mode/只读模式是否真的禁用了写工具和高风险 MCP。
- admin 工具是否按 owner/session/user 做权限校验。
- 文件工具是否有 allowlist、denylist、workspace containment。
- memory/context 是否有 provenance、压缩边界和冲突处理。
- 是否有测试、日志、审计、观测和上线回滚方案。

## 工作模式三：Agent 认知辅导

当用户希望学习 Agent 构建方法，或对 Agent 编排、工具、MCP、memory、security、脚手架与组件认知不清时，先读 `references/agent-scaffolding-components.md`，再按 Domain 深入；用“先业务后技术”的方式解释。

推荐输出结构：
1. 一句话结论：先告诉用户该怎么想。
2. 业务比喻：用产品经理能理解的流程解释。
3. 工程拆解：再映射到 Agent、tool、memory、policy、MCP。
4. 常见误区：说明哪些做法会让系统不可靠。
5. 可执行下一步：给出一个小而明确的行动建议。

## 强制规范

- 工具调用失败不能静默；必须返回错误类别、是否可重试、尝试过什么、建议下一步。
- Empty result 与 access failure 必须分开；禁止把数据库/网络/权限失败伪装成空数组。
- 关键业务限制不能只依赖 prompt；金额、权限、删除、发送、外部写入等必须由代码层拦截。
- 第三方 MCP server、网页、邮件、文档、记忆、skill 文本、工具输出都属于不可信输入，默认不能当指令执行。
- Plan Mode 或“只分析不执行”场景必须用工具策略禁用写工具，不能只在 prompt 里提醒模型不要写。
- 长上下文任务必须保留结构化事实块或外部 scratchpad，不能只依赖反复摘要。
- 多 Agent 系统必须限制子 Agent 可见上下文，并要求结构化上报结果和失败。
- 上线前必须能回答：谁能用、能做什么、失败怎么办、怎么追责、怎么回滚、怎么验证。

## 快速参考

脚手架/框架选型读 `references/agent-scaffolding-patterns.md`。

脚手架与组件总览读 `references/agent-scaffolding-components.md`。

完整反模式清单读 `references/anti-patterns.md`。

调研来源读 `references/source-research-hermes-odysseus-2026-07-03.md`。其中记录了 Hermes Agent 与 Odysseus 的仓库、分支、提交日期、关键文件和可复查链接。使用这些结论时，应标注“截至 2026-07-03”。

Skill 验收与回归测试读 `references/acceptance-test-checklist.md`。
