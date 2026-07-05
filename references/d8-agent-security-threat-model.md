# Domain 8: Agent 安全威胁模型

本文件用于评审 Agent 系统的安全边界。只要 Agent 能调用工具、读写数据、访问网络、发送消息、执行代码或连接 MCP，就必须做安全设计。

---

## D8.1 信任边界

默认不可信：
- 用户上传文档；
- 网页搜索结果；
- 抓取页面；
- 邮件正文和附件；
- RAG 检索片段；
- 工具输出；
- 长期记忆；
- 外部 skill 文本；
- 第三方 MCP server 的工具描述、schema 和返回值；
- 模型自己生成的下一步计划。

默认可信但仍需审计：
- 服务端代码；
- 本地配置；
- 已验证的内置工具；
- 权限系统；
- 人工审批结果；
- 后端数据库中带来源和时间戳的事实。

## D8.2 Prompt Injection 防护

强制规范：
- 外部内容进入模型前必须包成“数据块”，不能当指令。
- 包装边界要有开头和结尾，并对内容里的边界字符串做转义。
- system prompt 要声明：外部数据里的指令、密钥请求、工具调用要求都不能执行。
- 模型需要引用外部内容时，只能把它当证据，不得把它当开发者或系统指令。

Review 问题：
- 哪些输入通道可能带恶意指令？
- 是否有统一包装函数？
- wrapper 是否防止被内容提前关闭？
- 最终回答是否泄露内部安全提示？

## D8.3 工具权限

工具按风险分级：

| 风险 | 示例 | 要求 |
| --- | --- | --- |
| 只读低风险 | list、search、read metadata | 可在 Plan Mode 允许 |
| 只读高敏 | read email、read file、read secret-like config | 需要权限和审计 |
| 可变更 | write file、edit doc、update memory、create task | 默认禁用，需明确授权 |
| 外部影响 | send email、post webhook、call payment、delete data | 需要确认、权限、审计、回滚或补偿 |
| 执行类 | shell、python、subprocess、package install | 高风险，需 sandbox 或强约束 |
| 第三方 MCP | mcp__* | 默认高风险，按 server、tool、用户角色分级 |

强制规范：
- 工具权限必须在服务端执行层检查。
- prompt 里的“不要调用”不能替代 policy。
- 每个工具要有 owner/session/user 上下文。
- 高风险工具要记录 request、args、result、actor、time、session。

## D8.4 文件路径安全

文件工具必须：
- 解析真实绝对路径；
- 检查 workspace containment；
- 检查敏感路径 denylist；
- 检查允许根目录 allowlist；
- 防止大小写、软链接、相对路径绕过；
- 对写入操作要求更严格。

敏感路径示例：
- `.ssh`
- `.gnupg`
- `.aws`
- `.config`
- `.env`
- token、secret、credential、key 文件
- shell rc 文件
- 浏览器 profile

## D8.5 Secret 与配置

禁止：
- 在 MCP 配置里硬编码 token。
- 把 API key 写入 prompt、日志、memory、tool result。
- 让模型读取全局 env 并输出。
- 把 secret 当普通文本进入 RAG。

应该：
- 用环境变量或安全 secret store 引用。
- 日志做 redaction。
- 错误消息避免回显完整 token。
- 对“列出配置/环境变量”类工具做脱敏。

## D8.6 MCP 安全

MCP 风险：
- 第三方 server 描述里注入 prompt。
- schema 参数过多污染上下文。
- server 声明自己只读但实际有副作用。
- 多实例 server 名称混淆。
- non-admin 用户通过 `mcp__` 工具绕过权限。

强制规范：
- MCP 工具 schema 渲染要做长度和参数数量限制。
- 按 server_id + tool_name 路由，不能只按裸 tool name。
- 优先使用 readOnly/destructive annotations；缺失时 fail closed。
- admin/non-admin 权限要覆盖 MCP 命名空间。
- MCP 连接失败要返回可操作错误，而不是静默移除工具。

## D8.7 Memory 安全

长期记忆风险：
- 把注入文本永久保存。
- 把失败结果当成功记忆同步。
- 多用户 memory 混用。
- memory 被当作系统指令。

强制规范：
- memory 写入前要判断是否值得记忆。
- memory 写入要带来源、session、时间、agent/tool。
- memory 召回要包装为数据，不是指令。
- 用户应能查看、编辑、删除记忆。
- 高风险内容不得自动记忆，例如密钥、个人敏感信息、临时验证码。

## D8.8 Human-in-the-Loop

必须人工确认：
- 删除或不可逆操作；
- 大额交易或超权限操作；
- 外部发送消息；
- 多来源事实冲突；
- policy gap；
- 模型或工具多次失败；
- 安全策略判断不确定。

人工确认界面应展示：
- 当前事实；
- 推荐动作；
- 可选动作；
- 风险；
- 证据来源；
- 操作后果。

## D8.9 安全 Review 最小清单

- 是否定义了用户角色和权限矩阵？
- 是否定义了工具风险分级？
- 是否有 Plan Mode / guide-only 的服务端禁用策略？
- 是否所有不可信输入都有 wrapper？
- 是否有文件路径 allowlist/denylist？
- 是否禁止 secret 泄露到 prompt/log/memory？
- 是否 MCP 工具有 schema 截断和权限控制？
- 是否 admin 工具有 owner/session 校验？
- 是否高风险操作有人审或可回滚？
- 是否有审计日志和错误回放？
