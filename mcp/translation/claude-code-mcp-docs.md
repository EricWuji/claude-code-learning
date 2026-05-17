# 中文精读翻译：Connect Claude Code to tools via MCP

原文：https://code.claude.com/docs/en/mcp

说明：本文是 Claude Code 官方 MCP 文档的中文精读翻译与结构化改写，不是逐字全文搬运。

## 文档定位

这篇官方文档说明如何通过 Model Context Protocol 让 Claude Code 连接外部工具。MCP 是 AI 工具集成的开放标准。Claude Code 通过 MCP server 获取外部 tools、resources、prompts，以及某些情况下的外部事件 channel。

文档的核心建议是：当你反复把某个外部系统的数据复制进聊天窗口时，就应该考虑把那个系统接成 MCP server，让 Claude Code 直接读取和操作。

## MCP 能让 Claude Code 做什么

接入 MCP server 后，可以让 Claude Code：

- 从 issue tracker 读取需求并实现功能。
- 查看 Sentry、Statsig 等监控数据。
- 查询 PostgreSQL 等数据库。
- 根据 Figma 或 Slack 中的设计资料更新代码。
- 生成 Gmail 草稿或自动化工作流。
- 通过 channel 响应 Telegram、Discord、webhook 等外部事件。

这些能力的共同点是：Claude 不再只依赖用户粘贴的上下文，而是能通过协议访问真实外部系统。

## 查找和构建 MCP server

官方建议从 Anthropic Directory 中查找经过 review 的 connector。Directory connector 和 Claude Code 使用同一套 MCP 基础设施，所以可以用 `claude mcp add` 添加。

连接 server 前要确认你信任它。能读取外部网页、issue、邮件、聊天内容的 server 会带来 prompt injection 风险。

如果要构建自己的 server：

- 看 MCP server guide 理解协议基础。
- 看 Claude connector building docs 理解认证、测试和 Directory 提交流程。
- 也可以安装官方 `mcp-server-dev` plugin，让 Claude 帮你 scaffold MCP server。

官方插件流程：

```bash
/plugin install mcp-server-dev@claude-plugins-official
/reload-plugins
/mcp-server-dev:build-mcp-server
```

之后 Claude 会询问 use case，并生成远端 HTTP 或本地 stdio server。

## 安装 MCP server

Claude Code 支持几种安装方式。

### 方式一：远端 HTTP server

HTTP 是连接远端 MCP server 的推荐方式，尤其适合云服务。

```bash
claude mcp add --transport http notion https://mcp.notion.com/mcp
```

带 bearer token：

```bash
claude mcp add --transport http secure-api https://api.example.com/mcp \
  --header "Authorization: Bearer your-token"
```

JSON 配置中，`type: "streamable-http"` 可以作为 `type: "http"` 的别名。

### 方式二：远端 SSE server

SSE 是 Server-Sent Events。官方文档标注 SSE transport 已 deprecated。能使用 HTTP server 时，应优先使用 HTTP。

```bash
claude mcp add --transport sse asana https://mcp.asana.com/sse
```

### 方式三：本地 stdio server

stdio server 是本地进程，适合需要直接访问本机或自定义脚本的工具。

```bash
claude mcp add --transport stdio playwright -- npx -y @playwright/mcp
```

带环境变量：

```bash
claude mcp add --transport stdio --env AIRTABLE_API_KEY=YOUR_KEY airtable \
  -- npx -y airtable-mcp-server
```

选项顺序很重要：`--transport`、`--env`、`--scope`、`--header` 必须在 server name 前面；`--` 后面是传给 MCP server 的实际命令和参数。

stdio server 启动时，Claude Code 会给 server 进程设置 `CLAUDE_PROJECT_DIR`，表示项目根目录。server 内部可读取这个环境变量来解析项目相对路径。

## 管理 server

官方文档包含一组 `claude mcp` 管理命令。常见操作包括：

- 列出已配置 server。
- 查看 server 详情。
- 删除 server。
- 添加 JSON 配置。
- 从 Claude Desktop 导入 MCP server。

实际使用时，推荐用 CLI 添加个人或本地 server，用 `.mcp.json` 管理团队共享 server。

## 动态工具更新和自动重连

MCP server 可以在运行中更新自己暴露的 tools。Claude Code 会识别这些变化，使工具列表能随 server 状态更新。

如果连接断开，Claude Code 会尝试自动重连。对远端 server 或长时间会话，这能减少手工重启成本。

## Push messages with channels

MCP server 可以作为 channel，主动把外部事件推给 Claude Code session。这样 Claude 可以在你离开时响应消息或 webhook。

适用场景：

- Telegram/Discord 消息。
- CI webhook。
- 监控告警。
- 外部审批事件。

## Plugin-provided MCP servers

插件也可以提供 MCP server 配置。安装插件后，相关 server 可以随插件启用。适合团队把一组工具、skills、hooks、MCP server 打包成一个可安装能力包。

## MCP 安装 scope

官方文档把 scope 分得更细：

- Local：当前本地项目，适合个人实验。
- Project：项目 `.mcp.json`，适合团队共享。
- User：用户全局配置。
- Managed：组织管理配置。
- Subagent：特定 subagent 的 MCP server。

越具体的配置通常越接近实际任务；组织级 managed 配置用于限制或统一 MCP 入口。

## 环境变量展开

`.mcp.json` 可以使用环境变量展开，例如：

```json
{
  "mcpServers": {
    "remote-api": {
      "type": "http",
      "url": "https://mcp.example.com/mcp?token=${MCP_API_TOKEN}"
    }
  }
}
```

这避免把 secret 写进仓库。对 `CLAUDE_PROJECT_DIR` 要注意：它是在 server 进程环境中设置的，项目/用户 `.mcp.json` 的 `command` 或 `args` 里如果要引用它，建议提供默认值形式，例如 `${CLAUDE_PROJECT_DIR:-.}`。

## 实用示例

官方文档给出若干典型场景：

- Sentry：让 Claude 检查线上错误并关联代码。
- GitHub：让 Claude 参与 code review、issue、PR 工作流。
- PostgreSQL：让 Claude 查询数据库。

这些例子的共同结构是：配置 server，完成认证，然后在 prompt 中明确要求 Claude 使用对应系统。

## 远程 MCP server 认证

远程 server 可使用 OAuth 或自定义 header。官方文档覆盖：

- 固定 OAuth callback port。
- 预配置 OAuth credentials。
- 覆盖 OAuth metadata discovery。
- 限制 OAuth scopes。
- 动态 headers。

安全原则是：只授予 MCP server 真实需要的 scope，不给宽权限 token。

## JSON 配置与 Claude Desktop 导入

Claude Code 可以从 JSON 添加 MCP server，也可以从 Claude Desktop 导入已有 MCP server。这样可以复用已有生态配置。

## Claude Code 作为 MCP server

官方文档还支持把 Claude Code 暴露成 MCP server，供其他客户端使用。这适合把 Claude Code 的能力接入更大的 agent/workflow 系统。

## 输出限制

MCP tool 的输出有大小限制。输出太大时会被截断或产生 warning。某些 tool 可以提高单独限制，但不建议把 MCP 当成无限数据通道。更好的做法是让 server 返回摘要、分页或可查询资源。

## MCP resources

MCP server 可以暴露 resources。Claude Code 可以引用这些资源，把它们作为上下文读取。

适合资源：

- API 文档。
- 数据库 schema。
- 仓库索引。
- 监控数据快照。

## MCP prompts as commands

MCP server 可以暴露 prompts，Claude Code 可以把它们当作命令执行。这让团队能把固定工作流封装在 server 里，例如 code review 模板、发布检查、事故复盘模板。

## MCP Tool Search

如果 MCP server 暴露大量 tools，全部预先加载会占用上下文。Claude Code 支持 tool search：默认把 MCP tools 延迟加载，Claude 需要时再查找。

可用环境变量控制：

```bash
ENABLE_TOOL_SEARCH=auto:5 claude
ENABLE_TOOL_SEARCH=false claude
```

含义：

- 默认：按需 deferred，必要时加载。
- `true`：强制 deferred。
- `auto` 或 `auto:N`：按上下文占比阈值决定。
- `false`：禁用 tool search，全部 upfront 加载。

可以给某个 server 配 `alwaysLoad: true`，让它始终预加载：

```json
{
  "mcpServers": {
    "core-tools": {
      "type": "http",
      "url": "https://mcp.example.com/mcp",
      "alwaysLoad": true
    }
  }
}
```

## Managed MCP configuration

组织可以用 managed config 控制 MCP server。官方文档给了两种思路：

1. `managed-mcp.json` 独占控制：管理员集中声明允许的 MCP server。
2. policy allowlist/denylist：通过名称、命令、URL 模式限制用户可配置的 server。

远端 server 可按 URL 匹配，例如：

```json
{
  "allowedMcpServers": [
    { "serverUrl": "https://mcp.company.com/*" },
    { "serverUrl": "https://*.internal.corp/*" }
  ]
}
```

本地 stdio server 可按命令匹配：

```json
{
  "allowedMcpServers": [
    { "serverCommand": ["npx", "-y", "approved-package"] }
  ]
}
```

allowlist 为空数组表示完全锁定，不允许用户配置任何 MCP server。denylist 为空数组表示没有额外阻止项。

## 读后总结

Claude Code MCP 的落地方式可以记成：

1. 选可信 server。
2. 放到正确 scope。
3. 用 HTTP 接远端，用 stdio 跑本地。
4. 用环境变量处理 secret。
5. 用权限规则限制 MCP tools。
6. 让 Claude 在 prompt 中明确使用对应 server。
7. server 很多时依赖 Tool Search 或只 always-load 核心 server。

