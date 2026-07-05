# Domain 2: Tool Design & MCP Integration

来源：https://claudecertifications.com/claude-certified-architect/domains/tool-design-mcp

---

## D2.1 工具描述最佳实践

工具描述是模型决定何时、如何使用工具的唯一依据。描述越详细越好。

### 好的工具描述包含

1. **搜索/操作的目标**：搜索什么？操作什么？
2. **输入格式规范**：类型、格式要求、示例值
3. **边界条件**：哪些值合法，哪些不合法
4. **返回说明**：返回什么，空结果是否为错误
5. **互斥参数**：如只能传 email 或 phone 之一

### ✅ 规范示例

```json
{
  "name": "lookup_customer",
  "description": "Search for a customer by email, phone number, or account ID. Returns customer profile including name, account status, and order history summary. Input: exactly ONE of email, phone, or account_id. Email must contain @. Phone must be E.164 format (e.g., +15551234567). Account ID must start with ACC-. Returns: customer object or empty array if not found. Note: empty result means customer not found, this is NOT an error.",
  "input_schema": {
    "type": "object",
    "properties": {
      "email": {
        "type": "string",
        "description": "Customer email address (must contain @)"
      },
      "phone": {
        "type": "string",
        "description": "Phone in E.164 format, e.g., +15551234567"
      },
      "account_id": {
        "type": "string",
        "description": "Account ID starting with ACC-, e.g., ACC-12345"
      }
    }
  }
}
```

### ❌ Anti-Pattern

```json
{
  "name": "search",
  "description": "Searches for stuff",  // ❌ 完全不够
  "input_schema": {
    "type": "object",
    "properties": {
      "query": {"type": "string"}  // ❌ 没有格式说明
    }
  }
}
```

---

## D2.2 结构化错误响应

### 错误响应必须包含的字段

| 字段 | 说明 |
|------|------|
| `isError` | `true` 表示工具调用失败 |
| `errorCategory` | 错误分类：`validation` / `auth` / `not_found` / `rate_limit` / `timeout` |
| `isRetryable` | 告知 agent 重试是否有意义 |
| `context.attempted` | 描述尝试了什么操作 |
| `context.suggestion` | 建议下一步操作 |

### ⚠️ 关键区分：Access Failure vs Empty Result

这是生产系统中最危险的错误之一：

| 情况 | 含义 | 返回方式 |
|------|------|---------|
| 数据库返回空 | 确实没有匹配数据 | `{"isError": false, "customers": [], "metadata": {...}}` |
| 数据库连接失败 | 根本没有查到 | `{"isError": true, "errorCategory": "timeout", ...}` |

**绝对不能把 access failure 返回为 `[]`**——Agent 会误认为"查了没有结果"。

### ✅ 规范代码

```json
{
  "access_failure_example": {
    "isError": true,
    "errorCategory": "timeout",
    "isRetryable": true,
    "context": {
      "attempted": "Customer lookup by email: user@example.com",
      "service": "customer-database",
      "timeout_ms": 5000,
      "suggestion": "Retry after 2 seconds or try account ID lookup"
    }
  },
  "empty_result_example": {
    "isError": false,
    "customers": [],
    "metadata": {
      "searched_by": "email",
      "query": "user@example.com",
      "results_count": 0
    }
  }
}
```

### ❌ Anti-Pattern

```python
# 数据库挂了，但返回空数组
return {"customers": []}
# Agent 理解为：没有这个客户
# 实际是：根本没查到
```

---

## D2.3 工具分配 & 数量控制

### 核心原则：每个 Agent ≤ 5 个工具

研究表明 4-5 个工具是单 Agent 的最优工具数量。工具太多导致选择质量下降。

### 拆分策略

```python
# ❌ 错误：单 Agent 18 个工具
overloaded_agent = Agent(
    tools=[
        lookup_customer, update_account, verify_identity,
        find_order, process_refund, update_shipping,
        track_package, send_email, send_sms,
        create_ticket, escalate, search_kb,
        check_inventory, apply_coupon, schedule_callback,
        log_interaction, generate_report, update_preferences,
    ]  # 18 个工具 —— 选择质量严重退化
)

# ✅ 正确：Coordinator + 专职 Subagent
coordinator = Agent(tools=[Task, summarize_results, format_response])  # 3 个
customer_agent = Agent(tools=[lookup_customer, update_account, verify_identity, check_status])  # 4 个
order_agent = Agent(tools=[find_order, process_refund, update_shipping, track_package])  # 4 个
comms_agent = Agent(tools=[send_email, send_sms, create_ticket, escalate_human])  # 4 个
```

### tool_choice 参数

```python
# auto: 模型自己决定是否使用工具（默认）
tool_choice={"type": "auto"}

# any: 必须使用某个工具
tool_choice={"type": "any"}

# 强制使用特定工具（保证结构化输出）
tool_choice={"type": "tool", "name": "extract_invoice"}
```

---

## D2.4 MCP Server 配置

### 配置层级

| 文件 | 作用域 | 用途 |
|------|--------|------|
| `.mcp.json` | 项目级，可 git 提交，团队共享 | 项目工具配置 |
| `~/.claude.json` | 用户级，个人配置 | 个人工具偏好 |

### 密钥安全：必须用环境变量

```json
// ✅ 正确：引用环境变量
{
  "mcpServers": {
    "jira": {
      "command": "npx",
      "args": ["@company/jira-mcp-server"],
      "env": {
        "JIRA_URL": "${JIRA_URL}",
        "JIRA_TOKEN": "${JIRA_TOKEN}"
      }
    }
  }
}

// ❌ 错误：硬编码密钥（会被 git commit，等于泄露）
{
  "mcpServers": {
    "jira": {
      "env": {
        "JIRA_TOKEN": "PASTE_REAL_TOKEN_HERE"
      }
    }
  }
}
```

---

## D2.5 内置工具选择规范

Claude Code 6 个内置工具的正确使用场景：

| 工具 | 正确使用 | 不要用 Bash 替代 |
|------|---------|----------------|
| `Read` | 读取文件内容理解代码 | ✅ 不用 `Bash("cat file")` |
| `Write` | 从头创建新文件 | ✅ 不用 `Bash("echo ... > file")` |
| `Edit` | 修改已有文件的特定部分 | ✅ 不用 Write 覆盖整个文件 |
| `Grep` | 在代码库搜索模式 | ✅ 不用 `Bash("grep -r ...")` |
| `Glob` | 按 pattern 查找文件 | ✅ 不用 `Bash("find . -name ...")` |
| `Bash` | 执行命令（测试、构建、无内置替代的操作） | — |

```python
# ✅ 正确示例
Read("config.json")          # 读取文件
Write("tests/new-test.ts", content)  # 新建文件
Edit("server.ts", old_text, new_text)  # 修改特定行
Grep("getUserById", "src/")  # 搜索引用
Glob("**/*.test.ts")         # 查找测试文件
Bash("npm test")             # 运行测试（无内置替代）

# ❌ 错误示例
Bash("cat config.json")      # 应用 Read
Bash("echo '...' > new.ts")  # 应用 Write
Bash("grep -r 'func' src/")  # 应用 Grep
```
