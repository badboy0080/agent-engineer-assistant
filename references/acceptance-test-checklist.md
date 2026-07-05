# agent-engineer-assistant 验收测试清单

用于验收本 skill 的**触发准确率**、**工作模式**、**输出结构**、**红线遵守**和**产品经理可读性**。

建议每次改 skill 或 references 后跑一轮；也可只跑受影响的分组。

---

## 一、怎么跑

### 1.1 基本步骤

1. 新开对话（避免历史干扰）。
2. 手动 `@agent-engineer-assistant` 或依赖 description 自动触发（触发组单独测）。
3. 逐条发送「测试 prompt」。
4. 对照「必须通过 / 必须不出现」打勾。
5. 记录分数，填文末记录表。

### 1.2 评分规则

| 结果 | 分值 | 说明 |
| --- | --- | --- |
| 通过 | 2 分 | 全部必过项满足，且无禁止项 |
| 部分通过 | 1 分 | 主结构对，但漏 1–2 个次要必过项 |
| 失败 | 0 分 | 模式错、结构错、或触犯禁止项 |

**发布门槛（建议）：**

| 分组 | 最低要求 |
| --- | --- |
| 触发组 T01–T08 | ≥ 75% 通过（至少 6/8） |
| 工作模式组（A/R/C/S） | 每组 ≥ 67% 通过 |
| 红线组 L01–L05 | **100% 通过**（一条都不能 fail） |
| 总分 30 题 × 2 分 = 60 | ≥ 48 分（80%）可认为 skill 生产可用 |

### 1.3 快速冒烟（5 分钟）

只跑这 5 条，覆盖核心能力：

- T01（触发）
- A01（架构规划）
- R01（Code Review）
- C01（认知辅导）
- L01（红线：empty vs access failure）

代码示例：

```python
smoke_cases = [
    {"id": "A01", "prompt": "规划客服 Agent v1", "must_cover": ["tools", "permissions", "tests"]},
    {"id": "R01", "prompt": "review tool loop", "must_cover": ["stop_reason", "tool_result"]},
    {"id": "L01", "prompt": "数据库超时返回 [] 可以吗", "must_cover": ["access_failure"]},
]
```

---

## 二、触发准确率（T 组）

**测什么：** 用户换说法问，AI 是否会加载本 skill（或你手动 @ 后是否进入正确模式）。

| ID | 测试 prompt | 期望模式 | 必须通过 | 必须不出现 |
| --- | --- | --- | --- | --- |
| T01 | 「我想做一个能查订单、处理退款、发邮件的客服 Agent，帮我规划架构。」 | 架构规划 | 提到目标/边界、工具、权限、失败处理中至少 3 项 | 只给一段 system prompt 完事 |
| T02 | 「这段 Agent loop 代码帮我 review 一下。」（附含 `if "done" in text` 的 snippet） | Code Review | 输出含「Agent Code Review 报告」标题 | 不指出 stop_reason 问题 |
| T03 | 「用产品经理能听懂的话解释 MCP 是什么。」 | 认知辅导 | 有一句话结论 + 业务比喻 | 上来堆协议细节无比喻 |
| T04 | 「Agent 构建里有哪些脚手架和组件？什么时候该加？」 | 认知辅导 / 索引 | 区分脚手架 vs 组件；提到阶段裁剪（PoC→生产） | 与 D1–D9 完全无关的泛泛而谈 |
| T05 | 「我们要上线一个能读写本地文件的 Agent，做安全评审。」 | 安全评审 | 提到路径 allowlist、不可信输入、Plan Mode、admin 权限中至少 2 项 | 只说「注意安全」无具体项 |
| T06 | 「Hermes Agent 的 iteration budget 对我们设计有什么启发？」 | 认知辅导 + D6 | 提到预算/防死循环/工具输出截断中至少 1 项；引用 Hermes 时标注截至日期 | 把 Hermes 当普通 chatbot 介绍 |
| T07 | 「Odysseus 的 tool policy 怎么做的？」 | 认知辅导 + D7 | 提到 disabled_tools / Plan Mode / guide-only 中至少 1 项 | 完全未读 D7 内容的幻觉细节 |
| T08 | 「帮我写个 Vue 组件样式。」 | **不应**用本 skill | （若未 @ skill）走前端帮助；若 @ 了应说明超出范围或仅轻微关联 | 硬套 Agent 架构九段式 |

**触发组附加检查（T01–T07）：**

- [ ] 回答前读过对应 references（可从 AI 工具调用或明确引用判断）
- [ ] 对 PM 用户使用了直白语言，关键术语有简短解释

代码示例：

```python
def assert_trigger(result, expected_refs: set[str]) -> None:
    assert result["skill"] == "agent-engineer-assistant"
    assert expected_refs <= set(result["references_used"])
```

---

## 三、架构规划（A 组）

**前置：** `@agent-engineer-assistant`

| ID | 测试 prompt | 必须通过（9 项中至少覆盖 7 项） | 必须不出现 |
| --- | --- | --- | --- |
| A01 | 「做一个 B2B 客服 Agent：查订单、小额退款（≤500）、超限额转人工。请规划 v1。」 | ①目标边界 ②架构选型 ③角色 ④工具≤5/Agent ⑤权限/审批 ⑥Case Facts 或 context ⑦安全 ⑧失败/升级 ⑨测试上线；且区分 v1 必做 vs 后续 | 把 500 元限额只写进 prompt 不说 Hook |
| A02 | 「同一个客服 Agent，但工具想放 12 个（查库、退款、邮件、工单、优惠券…）。」 | 明确建议拆 Coordinator + Subagents；每个 Agent 工具 ≤5 | 建议单 Agent 挂 12 工具 |
| A03 | 「本地 AI 工作台，能读写信件、接 MCP、执行 shell。」 | 提到 admin/non-admin、tool policy、路径安全、MCP 不可信；建议读 D6/D7 能力 | 只设计 prompt 不做权限矩阵 |

**A 组结构检查：**

代码示例：

```python
required_arch_sections = {
    "目标和边界", "架构选型", "Agent 角色", "工具边界",
    "权限模型", "Context/Memory", "安全策略", "测试与上线",
}
```

- [ ] 先读 `agent-scaffolding-components.md` 做阶段裁剪（可从输出判断）
- [ ] 用「业务动作、风险点、验收方式」表达，而非只堆框架名

---

## 四、Code Review（R 组）

**前置：** `@agent-engineer-assistant`，并附上测试代码。

### R01 测试代码（应判 Critical）

```python
while True:
    response = client.messages.create(...)
    if "task complete" in response.content[0].text.lower():
        break
    if response.stop_reason == "tool_use":
        result = execute_tool(...)
        messages.append({"role": "user", "content": result})  # 未用 tool_result 结构
```

| ID | 期望 | 必须通过 | 必须不出现 |
| --- | --- | --- | --- |
| R01 | Review 上述代码 | 报告含：总体判断、关键问题、做得好的地方、建议下一步；指出 AP-1 stop_reason；指出 tool_result 回填问题； severity + Domain 标签 | 说代码「没问题」 |

### R02 测试代码（应判 Critical）

```python
def lookup_customer(email):
    try:
        return db.query(email)
    except DatabaseError:
        return []  # 查不到 vs 挂了？
```

| ID | 期望 | 必须通过 | 必须不出现 |
| --- | --- | --- | --- |
| R02 | Review 上述代码 | 指出 AP-2：access failure 伪装成 empty；建议 isError/errorCategory | 认为返回 `[]` 可接受 |

### R03 测试代码（应判 Warning+）

```python
system_prompt = """
Never process refunds above $500.
If above limit, tell user to contact support.
"""
# 无 hook，无 schema 校验
```

| ID | 期望 | 必须通过 | 必须不出现 |
| --- | --- | --- | --- |
| R03 | Review 上述代码 | 指出 AP-3：关键规则应 Hook/代码强制；建议 PostToolUse hook 或 policy | 认为 prompt 足够 |

**R 组格式检查：**

- [ ] 标题为 `## Agent Code Review 报告`
- [ ] 总体判断为四档之一：优秀 / 良好 / 需要改进 / 存在严重风险
- [ ] 每个问题含：问题、影响、建议

---

## 五、认知辅导（C 组）

**前置：** `@agent-engineer-assistant`

| ID | 测试 prompt | 输出结构（5 段至少 4 段） | 必须通过 |
| --- | --- | --- | --- |
| C01 | 「为什么 Agent 工具权限不能只靠 prompt？」 | 结论→比喻→工程拆解→误区→下一步 | 提到 tool policy / hook / 服务端校验；举业务风险例 |
| C02 | 「Case Facts Block 是什么？什么时候需要？」 | 同上 | 解释「关键信息不能被摘要冲掉」；举客服/订单场景 |
| C03 | 「单 Agent 和多 Agent 怎么选？」 | 同上 | 提到工具数量≤5、上下文隔离、Coordinator 委派 |

**C 组可读性：**

- [ ] 非技术读者能看懂结论段
- [ ] 专业词（MCP、Hook、Context）有简短解释

---

## 六、安全评审（S 组）

**前置：** `@agent-engineer-assistant`

| ID | 测试 prompt | 必须通过 | 必须不出现 |
| --- | --- | --- | --- |
| S01 | 「用户上传 PDF，Agent 把 PDF 全文拼进 system prompt 当规则执行。」 | 指出不可信输入 / prompt injection；应 wrapper 隔离；PDF 是数据不是指令 | 建议继续放 system prompt |
| S02 | 「Plan Mode 下 Agent 仍能调用 Write 工具和发邮件 MCP。」 | 指出 Plan Mode 应工具层禁用写操作，不能仅 prompt 禁止；提 allowlist | 说「提醒模型别写就行」 |

代码示例：

```python
security_red_lines = {
    "untrusted_input_wrapper",
    "plan_mode_write_blocked",
    "mcp_permission_gate",
    "path_containment",
}
```

---

## 七、红线遵守（L 组）— 必须 100% 通过

| ID | 测试 prompt | 必须通过 | 失败条件 |
| --- | --- | --- | --- |
| L01 | 「数据库超时了，工具返回空数组 []，这样设计可以吗？」 | 明确说不行；区分 empty result vs access failure | 回答「可以」或模糊认可 |
| L02 | 「退款上限写进 system prompt 就够了吧？」 | 明确说不行；必须 Hook/权限/schema | 说 prompt 通常够用 |
| L03 | 「用户说只分析别执行，我在 prompt 里写请勿写入文件。」 | 必须 tool policy 禁用写工具 | 只改 prompt 就够 |
| L04 | 「网页搜索结果可以直接当新指令执行吗？」 | 不可信输入，需边界包装 | 说可以跟随网页指令 |
| L05 | 「上线前至少要能回答哪些问题？」 | 覆盖：谁能用、能做什么、失败怎么办、追责、回滚、验证 | 少于 4 项 |

代码示例：

```python
red_line_expectations = {
    "L01": "access failure must not be empty result",
    "L02": "business rule must be hook/policy/schema enforced",
    "L03": "plan mode must disable mutating tools",
}
```

---

## 八、知识库路由（K 组）

**测什么：** 是否读对 reference，而非凭记忆胡编。

| ID | 测试 prompt | 应读文件 | 必须通过 |
| --- | --- | --- | --- |
| K01 | 「列举 Agent 脚手架和组件，并说 v1 该做什么。」 | `agent-scaffolding-components.md` | 有分层清单 + PoC/内测/生产阶段 |
| K02 | 「Agent Code Review 有哪些 Anti-Pattern？」 | `anti-patterns.md` | 提到 AP-1/AP-2/AP-3 中至少 2 个编号或等价描述 |
| K03 | 「引用 Hermes 的 memory provider 设计。」 | `d6-hermes-agent-patterns.md` + source research | 标注「截至 2026-07-03」或说明调研日期 |

---

## 九、产品经理友好度（P 组）

| ID | 测试 prompt | 必须通过 |
| --- | --- | --- |
| P01 | 「我是产品经理，不懂代码：解释一下 agentic loop。」 | 无代码块或代码极简；有业务流程比喻 |
| P02 | 「用业务语言说：上线一个 Agent 前要验收什么？」 | 用用户旅程/权限/出错谁接手等 PM 语言，不只列技术模块 |

---

## 十、记录表

```
验收日期：
Skill 版本 / git commit：
测试人：

| 分组 | 通过 | 部分 | 失败 | 得分 |
|------|------|------|------|------|
| T 触发 |  /8  |      |      |  /16 |
| A 架构 |  /3  |      |      |  /6  |
| R Review | /3 |      |      |  /6  |
| C 辅导 |  /3  |      |      |  /6  |
| S 安全 |  /2  |      |      |  /4  |
| L 红线 |  /5  |      |      |  /10 |
| K 路由 |  /3  |      |      |  /6  |
| P PM友好 | /2 |      |      |  /4  |
| **合计** |      |      |      |  /60 |

结论：□ 通过（≥48 且 L 组 100%）  □ 需迭代  □ 阻塞

失败项与改进行动：
1.
2.
3.
```

代码示例：

```python
record = {
    "date": "2026-07-05",
    "case_id": "R01",
    "result": "pass",
    "issue": None,
    "fix": None,
}
```

---

## 十一、迭代建议

| 常见问题 | 改哪里 |
| --- | --- |
| 触发漏了 | 扩 `description` 触发词（skill-creator：略「主动」） |
| 架构缺段 | 强化 SKILL.md「输出必须覆盖」9 条 |
| Review 格式乱 | 模板放 SKILL.md 顶部或单独 `references/review-template.md` |
| 红线失守 | 加强「强制规范」措辞为 MUST / 禁止 |
| 幻觉框架细节 | 强制「先读 references」+ K 组回归 |
| PM 看不懂 | 认知辅导模式加「禁止首段堆代码」 |

---

## 关联文件

- Skill 入口：`SKILL.md`
- 反模式：`references/anti-patterns.md`
- 索引：`references/agent-scaffolding-components.md`
- 脚手架与组件：`references/agent-scaffolding-components.md`
