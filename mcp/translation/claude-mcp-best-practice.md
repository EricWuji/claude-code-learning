# 中文精读翻译：MCP Servers Best Practice

原文：https://github.com/shanraisshan/claude-code-best-practice/blob/main/best-practice/claude-mcp.md

说明：本文是原文的中文精读翻译与结构化改写，不是逐字全文搬运。

## 文档定位

这篇 best-practice 文档把 MCP server 定义为 Claude Code 连接外部工具、数据库和 API 的扩展方式。它的重点不是讲 MCP 协议细节，而是讲日常开发中哪些 server 值得装、如何配置、如何控制权限。

核心观点：MCP server 不要贪多。很多人一开始会装十几个 server，但真正每天使用的通常只有少数几个。更好的策略是围绕实际工作流选择高频 server。

## 日常推荐 MCP server

### Context7

Context7 的用途是把最新库文档拉进 Claude Code 上下文。它解决的问题是：模型训练数据可能过时，容易编出不存在或旧版本 API。需要查 Next.js、React、Tailwind、数据库 SDK 等最新文档时，Context7 很适合。

### Playwright

Playwright MCP 用于浏览器自动化。Claude 可以打开网页、导航、截图、测试表单、验证 UI 行为。对前端开发尤其有用，因为它让 Claude 不只“写代码”，还能实际查看页面是否正常。

### Claude in Chrome

Claude in Chrome 连接真实 Chrome 浏览器，可以检查 console、network、DOM 等信息。它更偏真实用户环境调试，适合排查“代码看起来没问题，但浏览器里坏了”的问题。

### DeepWiki

DeepWiki 用于获取 GitHub 仓库的结构化 wiki 式文档，包括架构、API 面、模块关系等。它适合快速理解陌生 repo。

### Excalidraw

Excalidraw MCP 用于生成架构图、流程图和系统设计草图。它适合把 Claude 的分析输出变成可视化材料。

## 推荐工作流

原文把这些 server 组织成一个三段工作流：

```text
Research: Context7 / DeepWiki
  -> Debug: Playwright / Chrome
  -> Document: Excalidraw
```

也就是：

1. 先用 Context7/DeepWiki 查文档和理解系统。
2. 再用 Playwright/Chrome 调试和验证真实运行效果。
3. 最后用 Excalidraw 生成图示和文档资产。

## 配置位置

MCP server 可以配置在：

- 项目根目录 `.mcp.json`：项目级配置，适合团队共享并提交到 git。
- 用户配置 `~/.claude.json`：用户级配置，适合个人所有项目通用。

## server 类型

原文列出两类常见 server：

| 类型 | 传输方式 | 示例 |
| --- | --- | --- |
| `stdio` | 启动本地进程 | `npx`、`python`、本地 binary |
| `http` | 连接远端 URL | HTTP/SSE endpoint |

stdio server 适合本地工具。HTTP server 适合远端 API 或团队共享服务。

## `.mcp.json` 示例

原文给出的配置形态如下：

```json
{
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp"]
    },
    "playwright": {
      "command": "npx",
      "args": ["-y", "@playwright/mcp"]
    },
    "deepwiki": {
      "command": "npx",
      "args": ["-y", "deepwiki-mcp"]
    },
    "remote-api": {
      "type": "http",
      "url": "https://mcp.example.com/mcp"
    }
  }
}
```

这个例子混合了本地 stdio server 和远端 HTTP server。

## 密钥处理

原文强调：不要把 API key 写死在 `.mcp.json` 中提交。应使用环境变量展开：

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

这样 `.mcp.json` 可以进 git，而真实 token 留在开发者本机或 CI secret 中。

## MCP server 审批设置

`.claude/settings.json` 中有几个字段可以控制 `.mcp.json` server 是否自动批准：

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `enableAllProjectMcpServers` | boolean | 自动批准项目 `.mcp.json` 里的所有 server |
| `enabledMcpjsonServers` | array | 自动批准指定 server |
| `disabledMcpjsonServers` | array | 拒绝指定 server |

推荐做法是只批准明确需要的 server，而不是无脑开启全部。

## MCP tool 权限规则

MCP tools 在权限规则里使用：

```text
mcp__<server>__<tool>
```

例如：

```json
{
  "permissions": {
    "allow": [
      "mcp__context7__*",
      "mcp__playwright__browser_snapshot"
    ],
    "deny": [
      "mcp__dangerous-server__*"
    ]
  }
}
```

含义：

- 允许 Context7 server 下的所有工具。
- 只允许 Playwright 的截图/快照工具。
- 拒绝 dangerous-server 下所有 MCP tools。

## MCP scopes

原文把 MCP server 分为三层：

| Scope | 位置 | 用途 |
| --- | --- | --- |
| Project | `.mcp.json` | 团队共享 server，提交到 git |
| User | `~/.claude.json` | 个人所有项目通用 server |
| Subagent | agent frontmatter 的 `mcpServers` | 只给特定 subagent 使用 |

优先级：

```text
Subagent > Project > User
```

也就是越具体的 scope 优先级越高。

## 读后总结

这篇 best-practice 的核心不是 MCP 越多越强，而是：

- 研究阶段用文档/仓库理解类 MCP。
- 调试阶段用浏览器/真实环境类 MCP。
- 文档阶段用图示类 MCP。
- 配置放在合适 scope。
- 密钥用环境变量。
- 权限规则要按 server 和 tool 收窄。

