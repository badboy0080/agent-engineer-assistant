# Agent 工程师辅助 & Review Skill

基于 **Claude Certified Architect** 认证课程五大核心领域，为 Agent 工程师提供规范化的架构规划和代码 Review 能力。

## 这个 Skill 能做什么

| 模式            | 做什么                       | 适用场景               |
| ------------- | ------------------------- | ------------------ |
| **规划模式**      | 根据需求输出 Agent 架构设计方案       | 搭建新 Agent 时不知道从哪下手 |
| **Review 模式** | 按 30+ 条 Checklist 检查代码规范性 | 代码写完了，想查漏补缺        |

## 五大核心领域

| 域名 | 主题             | 学什么                                              |
| -- | -------------- | ------------------------------------------------ |
| D1 | Agent 架构 & 编排  | 单/多 Agent 选择、agentic loop、Hook 执行、会话管理           |
| D2 | 工具设计 & MCP     | 工具描述规范、错误响应、工具数量控制（≤5）、密钥安全                      |
| D3 | Claude Code 配置 | CLAUDE.md 层级、Plan Mode、CI/CD Session 隔离、Skill 约束 |
| D4 | Prompt 工程      | 量化标准、Few-Shot 示例、结构化输出、自我 Review 禁忌              |
| D5 | Context 管理     | Case Facts Block、升级触发条件、信息溯源、准确率分层追踪             |

## 使用方法

### 模式一：让 Skill 帮你做规划

直接在对话中描述你的需求，比如：

> "我想做一个能处理客户退款、查订单、发邮件的客服 Agent，帮我规划一下架构"

Skill 会按以下结构输出方案：

1. 架构选型（单 Agent / 多 Agent Hub-and-Spoke）
2. Agent 角色定义（Coordinator / Subagent 职责拆分）
3. 工具清单（每个 Agent ≤ 5 个工具）
4. 关键业务规则的执行机制（Hook vs Prompt）
5. Context 管理策略
6. 会话管理方案（Session / Fork / Resume）
7. 错误处理 & 上报设计
8. CLAUDE.md 配置建议

### 模式二：让 Skill 帮你做 Review

把代码贴给对话，说：

> "帮我 review 一下这段 Agent 代码，看看有没有 Anti-Pattern"

Skill 会输出结构化的 Review 报告：

- **总体评分**：优秀 / 良好 / 需要改进 / 存在严重问题
- **符合规范**：列出做得好的地方
- **发现的问题**：按所属 Domain + 严重程度（Critical/Warning/Info）标注，给出修改前后代码对比
- **优化建议**：非必须但值得考虑的改进

## 项目结构

```
agent-engineer-assistant/
├── SKILL.md                          # Skill 入口，定义工作模式和 Checklist
├── README.md                         # 本文件
└── references/                       # 五大领域详细规范文档
    ├── anti-patterns.md              # 20 条常见反模式速查表
    ├── d1-agentic-architecture.md    # Agent 架构 & 编排规范
    ├── d2-tool-design-mcp.md         # 工具设计 & MCP 集成规范
    ├── d3-claude-code-config.md      # Claude Code 配置规范
    ├── d4-prompt-engineering.md      # Prompt 工程规范
    └── d5-context-management.md      # Context 管理 & 可靠性规范
```

## 核心设计原则

这个 Skill 强行约束了以下几个"红线"，也是 Review 时最常发现的问题：

- **循环控制**：必须用 `stop_reason` 控制 agentic loop，禁止解析自然语言文本判断终止
- **工具数量**：单个 Agent 工具数 ≤ 5，超过则拆分为多 Agent
- **Hook 使用**：关键业务规则（金额限制、权限校验）必须用 Hook 强制执行，不能靠 Prompt
- **错误区分**：access failure（查不到）和 empty result（没有数据）必须明确区分
- **密钥安全**：MCP 配置必须用环境变量引用密钥，禁止硬编码

## 适用人群

- 正在用 Claude Code / SOLO 搭建 Agent 应用的开发者
- 需要规范化团队 Agent 代码质量的架构师
- 准备 Claude Certified Architect 认证考试的学员

