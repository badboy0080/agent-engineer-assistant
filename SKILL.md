---
name: agent-engineer-assistant
description: 这个 skill 用于辅助 Agent 工程师进行规范化的 Agent 系统构建和代码 review。当用户需要：(1) 根据需求规划 Agent 架构和构建方案，(2) review 已有的 Agent 代码是否符合生产级规范，(3) 识别 Anti-Pattern 并给出正确替代方案时，应使用此 skill。规范来源于 Claude Certified Architect 认证课程的五大 Domain。
---

# Agent 工程师辅助 & Review Skill

## 概述

本 skill 基于 Claude Certified Architect 认证课程的五大核心领域，为 Agent 工程师提供：
1. **规划辅助**：根据用户需求，输出规范的 Agent 架构设计方案
2. **代码 Review**：检查代码是否符合生产级 Agent 规范，识别 Anti-Pattern

## 五大核心 Domain

| Domain | 主题 | 参考文件 |
|--------|------|---------|
| D1 | Agentic Architecture & Orchestration | `references/d1-agentic-architecture.md` |
| D2 | Tool Design & MCP Integration | `references/d2-tool-design-mcp.md` |
| D3 | Claude Code Configuration & Workflows | `references/d3-claude-code-config.md` |
| D4 | Prompt Engineering & Structured Output | `references/d4-prompt-engineering.md` |
| D5 | Context Management & Reliability | `references/d5-context-management.md` |

---

## 工作模式

### 模式一：规划 Agent 构建方案

当用户描述需求时，按以下框架输出方案：

#### 规划输出结构

```
1. 架构选型（单 Agent / 多 Agent Hub-and-Spoke）
2. Agent 角色定义（Coordinator / Subagent 职责拆分）
3. 工具清单（每个 Agent ≤5 个工具）
4. 关键业务规则的执行机制（Hook vs Prompt）
5. Context 管理策略
6. 会话管理方案（Session / Fork / Resume）
7. 错误处理 & 上报设计
8. CLAUDE.md 配置建议
```

#### 规划时必须考虑的强制规范

- **循环控制**：必须用 `stop_reason` 控制 agentic loop，禁止解析自然语言文本判断终止
- **工具数量**：单个 Agent 工具数 ≤ 5，超过则拆分为多 Agent
- **Hook 使用**：关键业务规则（金额限制、权限校验）必须用 Hook 强制执行，不能靠 Prompt
- **Context 传递**：Subagent 只传递与其任务相关的最小 Context
- **错误区分**：access failure（查不到）和 empty result（没有数据）必须明确区分

---

### 模式二：代码 Review

对用户提交的 Agent 代码，按以下 Checklist 逐项检查，输出 Review 报告。

#### Review Checklist

**[D1] Agentic Loop & 架构**
- [ ] Loop 终止条件是否检查 `stop_reason`，而非解析文本内容
- [ ] 工具调用结果是否正确 append 到 messages
- [ ] 多 Agent 场景是否使用 Hub-and-Spoke，而非平铺结构
- [ ] Subagent 是否有 context 隔离（不共享完整 coordinator 历史）
- [ ] `fork_session` vs `--resume` 使用是否正确

**[D2] 工具设计 & MCP**
- [ ] 每个 Agent 工具数是否 ≤ 5
- [ ] 工具描述是否包含：输入格式、边界条件、示例、返回说明
- [ ] 错误响应是否包含 `isError`、`errorCategory`、`isRetryable` 字段
- [ ] access failure 和 empty result 是否明确区分（禁止把 failure 返回为 `[]`）
- [ ] MCP 配置是否用 `${ENV_VAR}` 引用密钥，禁止硬编码
- [ ] 内置工具选择是否正确（Read/Write/Edit/Bash/Grep/Glob 各归其位）

**[D3] Claude Code 配置**
- [ ] CLAUDE.md 层级是否合理（user / project / directory）
- [ ] 是否使用模块化配置（`@import`、`.claude/rules/`）而非单一超长文件
- [ ] 复杂任务是否启用 Plan Mode，简单任务是否避免不必要的 Plan
- [ ] CI/CD 是否使用 `-p` flag + 独立 session（generator 和 reviewer 隔离）
- [ ] Skill 是否设置了 `context: fork` 和 `allowed-tools` 约束

**[D4] Prompt 工程 & 结构化输出**
- [ ] Prompt 是否使用明确可量化的标准（如"超过50行"而非"较长"）
- [ ] 复杂任务是否使用 2-4 个 few-shot 示例
- [ ] 结构化输出是否通过 `tool_use` 而非解析自然语言
- [ ] Validation-retry loop 是否把具体错误信息 append 到 messages
- [ ] 是否避免同 session 自我 review（禁止 `--resume` 后立即 review）

**[D5] Context 管理 & 可靠性**
- [ ] 是否使用 Case Facts Block 保存关键信息（禁止依赖 progressive summarization）
- [ ] 是否考虑 "lost in the middle" 效应，将关键信息放在 context 首尾
- [ ] 升级触发条件是否基于客观标准（用户明确请求、Policy Gap、金额超限、重试耗尽）
- [ ] 是否避免基于情绪（sentiment）或模型自报置信度触发升级
- [ ] 多类型文档是否按类型分层跟踪准确率（禁止只看总体平均）
- [ ] 信息来源是否带 provenance（source、confidence、retrieved_at）

---

## Review 报告格式

```markdown
## Agent Code Review 报告

### 总体评分
[优秀 / 良好 / 需要改进 / 存在严重问题]

### ✅ 符合规范
- [列出做得好的地方]

### ❌ 发现的问题

#### [问题1] [所属 Domain] - [严重程度: Critical/Warning/Info]
**问题描述**：[说明是什么 Anti-Pattern]
**有问题的代码**：
```代码
[problematic code]
```
**正确做法**：
```代码
[corrected code]
```

### 🔧 优化建议
- [非必须但值得考虑的改进]
```

---

## 快速参考：核心 Anti-Pattern 列表

读取 `references/anti-patterns.md` 获取完整的反模式速查表。

---

## 使用流程

**规划请求**：当用户描述 Agent 需求时：
1. 读取对应 domain 的 references 文件以获取详细规范
2. 按规划输出结构生成方案
3. 在方案中明确标注每个设计决策的依据

**Review 请求**：当用户提交代码时：
1. 先快速过一遍全部代码，识别涉及哪些 Domain
2. 按 Review Checklist 逐项检查
3. 按报告格式输出，严重问题（Critical）必须给出正确代码示例
