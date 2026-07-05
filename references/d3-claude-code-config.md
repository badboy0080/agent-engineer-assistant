# Domain 3: Claude Code Configuration & Workflows

来源：https://claudecertifications.com/claude-certified-architect/domains/claude-code-config

---

## D3.1 CLAUDE.md 层级

| 层级 | 路径 | 作用域 | git提交 |
|------|------|--------|---------|
| 用户级 | `~/.claude/CLAUDE.md` | 所有项目 | ❌ |
| 项目级 | `.claude/CLAUDE.md` | 当前项目 | ✅ |
| 目录级 | `src/api/CLAUDE.md` | 该目录及子目录 | ✅ |

**优先级**：目录级 > 项目级 > 用户级

模块化结构：`.claude/rules/` 目录中的 `.md` 文件自动加载；用 `@import ./rules/xxx.md` 引用其他文件。

**Anti-Pattern**：把所有规则写进一个 800 行的 CLAUDE.md，应拆分为模块。

代码示例：

```text
.claude/
├── CLAUDE.md
└── rules/
    ├── agent-safety.md
    ├── tool-policy.md
    └── testing.md
src/payments/
└── CLAUDE.md
```

```markdown
# .claude/CLAUDE.md
@import ./rules/agent-safety.md
@import ./rules/tool-policy.md

所有支付相关代码还必须遵守 src/payments/CLAUDE.md。
```

---

## D3.2 Commands vs Skills

| 特性 | Commands | Skills |
|------|----------|--------|
| context | 主 context | `context: fork`（隔离） |
| 工具限制 | 无 | `allowed-tools` |
| 适用 | 简单快捷操作 | 复杂探索任务 |

Skill frontmatter：

```yaml
---
context: fork
allowed-tools:
  - Read
  - Edit
  - Grep
argument-hint: "file or directory to refactor"
---
```

代码示例：

```yaml
# .claude/commands/smoke.md
---
argument-hint: "test target"
---
Run the smallest smoke check for $ARGUMENTS and summarize failures.
```

```yaml
# skills/agent-review/SKILL.md
---
name: agent-review
description: Review Agent code for loop, tool, MCP, memory, and safety risks.
---

Read the relevant reference first, then report findings by severity.
```

---

## D3.3 Plan Mode

- **用 Plan Mode**：多步骤、影响广、架构性任务
- **直接执行**：简单、定义清晰的任务

TDD 迭代模式：先写测试 → 运行（失败）→ 实现 → 运行（通过）→ 精化

代码示例：

```python
READ_ONLY_TOOLS = {"Read", "Grep", "Glob", "List"}

def allowed_tools_for_mode(mode: str, all_tools: set[str]) -> set[str]:
    if mode == "plan":
        return all_tools & READ_ONLY_TOOLS
    return all_tools

assert "Edit" not in allowed_tools_for_mode("plan", {"Read", "Edit"})
```

---

## D3.4 CI/CD 集成

关键 flag：`-p`（非交互）、`--output-format json`、`--json-schema`

**Session 隔离原则**：生成 session 和 review session 必须分离，禁止 `--resume` 后立即 review。

```bash
# ✅ 正确
claude -p "Write new auth module"    # Session A
claude -p "Review this diff: ..."    # Session B（新 session）

# ❌ 错误
claude -p "Write new auth module"
claude --resume -p "Review your code"  # 同 session，存在确认偏差
```

Batch API：50% 成本节省，适合延迟容忍型批量任务（24h窗口），用 `custom_id` 追踪。

代码示例：

```bash
claude -p "Implement feature X" --output-format json > build-session.json
git diff > candidate.diff
claude -p "Review candidate.diff for production Agent risks" \
  --output-format json \
  < candidate.diff > review-session.json
```

```json
{"custom_id": "review-agent-diff-001", "task": "agent_code_review"}
```
