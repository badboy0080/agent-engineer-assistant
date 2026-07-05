# Agent 工程师辅助 & Review Skill

面向产品经理、Agent 工程师和架构负责人，提供从 Agent 想法、架构规划、工具/MCP 设计、代码 Review、安全评审到上线验收的一套生产级知识库。

本 skill 保留 Claude Certified Architect 风格的五大基础 Domain，并补充了对 Hermes Agent 与 Odysseus 的源码调研结论。使用这些源码调研结论时，应标注“截至 2026-07-03”。

## 这个 Skill 能做什么

| 模式 | 做什么 | 适用场景 |
|------|--------|---------|
| **架构规划** | 把 Agent 想法拆成角色、工具、权限、context、memory、安全和测试方案 | 从 0 到 1 设计 Agent 产品 |
| **代码 Review** | 按生产级 Agent Checklist 检查 loop、工具、MCP、权限、安全和可靠性 | 已有 Agent 代码需要查漏补缺 |
| **认知辅导** | 用产品经理能理解的话解释 Agent 编排、MCP、memory、tool policy 等概念 | 团队需要统一 Agent 构建认知 |
| **安全评审** | 检查 prompt injection、不可信输入、secret、路径、admin 工具和第三方 MCP 风险 | 准备上线或接入真实业务权限 |

## 九大核心领域

| 域名 | 主题 | 学什么 |
|------|------|--------|
| D1 | Agent 架构 & 编排 | 单/多 Agent 选择、agentic loop、Hook 执行、会话管理 |
| D2 | 工具设计 & MCP | 工具描述规范、错误响应、工具数量控制（≤5）、密钥安全 |
| D3 | Claude Code 配置 | CLAUDE.md 层级、Plan Mode、CI/CD Session 隔离、Skill 约束 |
| D4 | Prompt 工程 | 量化标准、Few-Shot 示例、结构化输出、自我 Review 禁忌 |
| D5 | Context 管理 | Case Facts Block、升级触发条件、信息溯源、准确率分层追踪 |
| D6 | Hermes Agent 工程模式 | conversation loop、iteration budget、tool guardrails、memory provider、context engine、skill/MCP bridge |
| D7 | Odysseus 产品化模式 | 本地 AI 工作台、tool policy、admin/non-admin 权限、不可信输入包装、MCP manager |
| D8 | Agent 安全威胁模型 | prompt injection、文件路径、secret、权限边界、第三方 MCP server 风险 |
| D9 | Agent 产品化清单 | 配置、日志、观测、测试、回滚、文档、上线验收 |

## 使用方法

### 模式一：让 Skill 帮你做架构规划

直接在对话中描述你的需求，比如：

> "我想做一个能处理客户退款、查订单、发邮件的客服 Agent，帮我规划一下架构，并说明哪些地方必须有权限控制和人工确认。"

Skill 会按以下结构输出方案：

1. 架构选型（单 Agent / 多 Agent Hub-and-Spoke）
2. Agent 角色定义（Coordinator / Subagent 职责拆分）
3. 工具清单（每个 Agent ≤ 5 个工具）
4. 关键业务规则的执行机制（Hook vs Prompt）
5. Context 管理策略
6. 会话管理方案（Session / Fork / Resume）
7. 错误处理 & 上报设计
8. 安全风险、测试方案和上线验收标准

### 模式二：让 Skill 帮你做 Review

把代码贴给对话，说：

> "帮我 review 一下这段 Agent 代码，看看有没有 Anti-Pattern"

Skill 会输出结构化的 Review 报告：

- **总体评分**：优秀 / 良好 / 需要改进 / 存在严重问题
- **符合规范**：列出做得好的地方
- **发现的问题**：按所属 Domain + 严重程度（Critical/Warning/Info）标注，给出修改建议
- **优化建议**：非必须但值得考虑的改进

### 模式三：让 Skill 辅助认知升级

适合产品经理或团队负责人：

> "请用产品经理能听懂的话解释，为什么 Agent 工具权限不能只靠 prompt 约束？"

Skill 会先讲业务风险，再映射到工程机制，例如 tool policy、server-side permission、hook、schema、审计日志。

### 模式四：让 Skill 做 Agent 安全评审

适合准备上线或接入真实数据前：

> "请按生产级 Agent 安全标准，评审我的 MCP 工具设计、文件权限和 memory 方案。"

Skill 会重点检查不可信输入、prompt injection、secret、文件路径、admin 工具、Plan Mode 写权限、第三方 MCP server 等问题。

## 项目结构

```
agent-engineer-assistant/
├── SKILL.md                          # Skill 入口，定义工作模式和 Checklist
├── README.md                         # 本文件
└── references/                       # 五大领域详细规范文档
    ├── anti-patterns.md              # 常见反模式速查表
    ├── d1-agentic-architecture.md    # Agent 架构 & 编排规范
    ├── d2-tool-design-mcp.md         # 工具设计 & MCP 集成规范
    ├── d3-claude-code-config.md      # Claude Code 配置规范
    ├── d4-prompt-engineering.md      # Prompt 工程规范
    ├── d5-context-management.md      # Context 管理 & 可靠性规范
    ├── d6-hermes-agent-patterns.md   # Hermes Agent 源码工程模式
    ├── d7-odysseus-product-patterns.md # Odysseus 产品化工程模式
    ├── d8-agent-security-threat-model.md # Agent 安全威胁模型
    ├── d9-agent-productization-checklist.md # 产品化上线清单
    └── source-research-hermes-odysseus-2026-07-03.md # 调研来源记录
```

## 核心设计原则

这个 Skill 强行约束了以下几个"红线"，也是 Review 时最常发现的问题：

- **循环控制**：必须用 `stop_reason` 控制 agentic loop，禁止解析自然语言文本判断终止
- **工具数量**：单个 Agent 工具数 ≤ 5，超过则拆分为多 Agent
- **Hook 使用**：关键业务规则（金额限制、权限校验）必须用 Hook 强制执行，不能靠 Prompt
- **错误区分**：access failure（查不到）和 empty result（没有数据）必须明确区分
- **密钥安全**：MCP 配置必须用环境变量引用密钥，禁止硬编码
- **不可信输入隔离**：网页、邮件、文档、记忆、skill 文本、工具输出不能直接作为 system prompt
- **权限强制执行**：Plan Mode、admin 工具、文件写入、MCP 调用必须有服务端策略约束
- **上线可验证**：必须有测试、日志、审计、观测和回滚方案

## 适用人群

- 正在用 Claude Code、Codex、Hermes、OpenAI Agents SDK、LangGraph、MCP 等搭建 Agent 应用的开发者
- 需要规范化团队 Agent 代码质量的架构师
- 需要把 Agent 产品讲清楚、管住风险、推动上线的产品经理
- 需要评审工具权限、MCP 接入、memory/context 方案的技术负责人

## 调研来源

- Hermes Agent：`NousResearch/hermes-agent`，`main` 分支，调研时最新提交 `89acc196067c`，提交时间 2026-07-03。
- Odysseus：`pewdiepie-archdaemon/odysseus`，`dev` 分支，调研时最新提交 `8c943226f815`，提交时间 2026-07-02。
- 详细来源记录见 `references/source-research-hermes-odysseus-2026-07-03.md`。
