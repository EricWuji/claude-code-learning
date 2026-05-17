# Claude Code 中的 MCP 服务：作用、配置与使用方式

资料来源：

- Claude Code 官方文档: https://code.claude.com/docs/en/mcp
- Claude Code Best Practice: https://github.com/shanraisshan/claude-code-best-practice/blob/main/best-practice/claude-mcp.md
- FastMCP Claude Code integration: https://fastmcp.mintlify.app/integrations/claude-code
- FastMCP install docs: https://gofastmcp.com/cli/install-mcp
- FastMCP running servers docs: https://gofastmcp.com/cli/running

## 1. MCP 是什么

MCP，全称 Model Context Protocol，是一种给 AI agent 接入外部工具、数据源和 API 的开放协议。放到 Claude Code 里理解，MCP server 就是“给 Claude Code 增加工具能力的外部服务”。

没有 MCP 时，Claude 通常只能依赖你粘贴进聊天窗口的上下文。接入 MCP 后，Claude Code 可以通过 MCP server 直接访问外部系统，例如：

- issue tracker：读取 Jira/GitHub issue，然后实现需求。
- GitHub：查看 PR、issue、仓库信息，辅助 code review。
- Sentry/监控平台：查看错误日志和线上指标。
- PostgreSQL/数据库：查询业务数据。
- Figma/Slack/Gmail：读取设计、消息或生成草稿。
- Playwright/Chrome：打开浏览器、截图、测试 UI。
- Context7/DeepWiki：把最新库文档或 GitHub 仓库结构拉进上下文。

一句话：MCP 把 Claude Code 从“只会读当前代码和你粘贴的内容”扩展成“可以操作一组外部工具的开发 agent”。

## 2. MCP 服务在 Claude Code 中怎么工作

Claude Code 和 MCP server 的关系可以拆成 5 步：

1. 你在配置中声明 MCP server。
2. Claude Code 启动或需要时连接这些 server。
3. server 向 Claude Code 暴露 tools、resources、prompts 等能力。
4. Claude 在任务中需要外部能力时，选择相应 MCP tool。
5. Claude Code 根据权限规则执行工具调用，把结果返回给 Claude。

典型调用链：

```text
用户需求
  -> Claude 规划任务
  -> Claude 发现需要外部系统
  -> 选择 mcp__server__tool
  -> Claude Code 调用 MCP server
  -> MCP server 访问外部 API/本地进程/浏览器/数据库
  -> 返回结果
  -> Claude 基于结果继续编码、分析或回复
```

MCP server 本质上不一定是远端服务。它可以是：

- 本地进程：Claude Code 通过 stdio 启动，比如 `npx -y @playwright/mcp`。
- 远端 HTTP 服务：Claude Code 通过 HTTP endpoint 连接。
- 旧式 SSE 服务：官方文档标注 SSE 已不推荐，能用 HTTP 时优先用 HTTP。
- plugin 提供的 MCP server：插件启用后带来 MCP 配置。
- Claude Code 自己作为 MCP server：供其他客户端连接 Claude Code 的能力。

## 3. MCP server 的类型

| 类型 | 配置方式 | 适合场景 | 说明 |
| --- | --- | --- | --- |
| `stdio` | `command` + `args` | 本地脚本、本地浏览器、数据库代理、Node/Python 包 | Claude Code 启动一个本地进程，通过 stdin/stdout 通信 |
| `http` | `type: "http"` + `url` | 云服务、团队共享工具、官方 connector | 官方推荐的远程传输方式 |
| `sse` | `type: "sse"` 或 CLI `--transport sse` | 老 connector | 已不推荐；有 HTTP 时优先 HTTP |

官方文档还提到，JSON 配置里 `type` 可以使用 `streamable-http` 作为 `http` 的别名，因为 MCP specification 中使用这个传输名称。

## 4. MCP config 放在哪里

Claude Code 的 MCP 配置有多层 scope。

| Scope | 位置 | 用途 |
| --- | --- | --- |
| Local | 通过 `claude mcp add` 默认写入本地项目配置 | 只给当前本地 checkout 使用，适合个人实验 |
| Project | 仓库根目录 `.mcp.json` | 团队共享，适合提交到 git |
| User | `~/.claude.json` 的 `mcpServers` | 用户个人所有项目通用 |
| Subagent | agent frontmatter 的 `mcpServers` | 只给某个 subagent 使用 |
| Managed | 管理员策略，如 `managed-mcp.json` 或 policy | 组织统一控制 |

实践建议：

- 团队都要用的 server 放 `.mcp.json`。
- 个人 token、个人工具、试验 server 放用户级或 local scope。
- 只给某个 agent 用的工具放 subagent scope。
- 组织安全要求用 managed config 或 allowlist/denylist。

## 5. `.mcp.json` 示例

下面是一个项目级 `.mcp.json` 示例，适合放在仓库根目录：

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
    "company-api": {
      "type": "http",
      "url": "https://mcp.example.com/mcp"
    }
  }
}
```

含义：

- `context7`、`playwright`、`deepwiki` 是本地 stdio server。Claude Code 会启动这些命令。
- `company-api` 是远端 HTTP MCP server。Claude Code 直接连 URL。
- server 名会影响工具名。比如 `playwright` server 暴露的 `browser_snapshot` 工具，在权限规则里通常叫 `mcp__playwright__browser_snapshot`。

## 6. 带密钥的配置示例

不要把 API key 明文提交进 `.mcp.json`。用环境变量：

```json
{
  "mcpServers": {
    "company-api": {
      "type": "http",
      "url": "https://mcp.example.com/mcp?token=${MCP_API_TOKEN}"
    },
    "airtable": {
      "command": "npx",
      "args": ["-y", "airtable-mcp-server"],
      "env": {
        "AIRTABLE_API_KEY": "${AIRTABLE_API_KEY}"
      }
    }
  }
}
```

本地启动 Claude Code 前设置环境变量：

```bash
export MCP_API_TOKEN="your-token"
export AIRTABLE_API_KEY="your-airtable-key"
claude
```

如果是 stdio server，Claude Code 会在它启动的 server 进程环境里设置 `CLAUDE_PROJECT_DIR`，值是项目根目录。Node server 可读 `process.env.CLAUDE_PROJECT_DIR`，Python server 可读 `os.environ["CLAUDE_PROJECT_DIR"]`。

## 7. 用 CLI 添加 MCP server

### 7.1 添加远端 HTTP server

```bash
claude mcp add --transport http notion https://mcp.notion.com/mcp
```

带认证 header：

```bash
claude mcp add --transport http secure-api https://api.example.com/mcp \
  --header "Authorization: Bearer $MCP_API_TOKEN"
```

### 7.2 添加本地 stdio server

```bash
claude mcp add --transport stdio playwright -- npx -y @playwright/mcp
```

带环境变量：

```bash
claude mcp add --transport stdio --env AIRTABLE_API_KEY=$AIRTABLE_API_KEY airtable \
  -- npx -y airtable-mcp-server
```

注意：`--transport`、`--env`、`--scope`、`--header` 这些选项要放在 server name 前面；`--` 后面才是 MCP server 的实际启动命令和参数。

## 8. MCP server 审批与权限

项目级 `.mcp.json` 里的 server 通常需要用户批准。可以在 `.claude/settings.json` 中控制项目 server 是否自动批准。

```json
{
  "enableAllProjectMcpServers": false,
  "enabledMcpjsonServers": ["context7", "playwright"],
  "disabledMcpjsonServers": ["dangerous-server"]
}
```

含义：

- `enableAllProjectMcpServers: true`：自动批准 `.mcp.json` 里的所有 server。
- `enabledMcpjsonServers`：只自动批准列出的 server。
- `disabledMcpjsonServers`：拒绝列出的 server。

MCP tools 的权限规则使用 `mcp__<server>__<tool>` 命名：

```json
{
  "permissions": {
    "allow": [
      "mcp__context7__*",
      "mcp__playwright__browser_snapshot",
      "mcp__playwright__browser_navigate"
    ],
    "deny": [
      "mcp__dangerous-server__*"
    ]
  }
}
```

常见策略：

- 文档查询类工具可以宽一点，如 `mcp__context7__*`。
- 浏览器自动化工具建议按工具名开放，如只允许 snapshot/navigate，谨慎开放表单提交。
- 能写外部系统、发消息、改数据库的工具要保守。

## 9. Claude Code 如何使用 MCP 服务

连接 MCP server 后，Claude Code 会把 MCP server 暴露的能力纳入工具系统。使用方式主要有 4 类。

### 9.1 作为 tools 使用

这是最常见的方式。Claude 看到任务需要外部能力时，会调用 MCP tool。

示例 prompt：

```text
Use Context7 to check the latest Next.js routing docs, then update our app router code.
```

Claude 可能会调用：

```text
mcp__context7__resolve-library-id
mcp__context7__get-library-docs
```

再根据返回文档修改代码。

### 9.2 作为 resources 使用

MCP server 可以暴露 resources。你可以在 Claude Code 中引用资源，让 Claude 把它当成上下文读取。

适合：

- 数据库 schema。
- API 文档。
- 监控 dashboard snapshot。
- repo wiki。

### 9.3 作为 prompts/commands 使用

一些 MCP server 会暴露 prompts。Claude Code 可以把 MCP prompt 当成命令执行，类似“由 MCP server 提供的可复用工作流”。

适合：

- 固定 code review 流程。
- 固定 release checklist。
- 团队内部分析模板。

### 9.4 作为 channel 接收外部事件

官方文档提到，MCP server 也可以作为 channel，把 Telegram、Discord、webhook 等外部事件推入 Claude Code session，让 Claude 在你离开时响应外部消息。

## 10. Tool Search：默认延迟加载 MCP tools 是什么意思

Claude Code 官方文档里说的“默认把 MCP tools 延迟加载”，指的是：Claude Code 不一定在会话一开始就把所有 MCP tool 的完整描述都塞进模型上下文，而是先把大量 MCP tools 暂时放在一个可搜索的工具目录里。Claude 需要某类能力时，先通过 `ToolSearch` 找到相关 MCP tools，再把命中的工具描述加载进当前上下文，然后才能调用具体工具。

这解决的是上下文窗口和工具数量的矛盾。每个 tool 都有名称、描述、参数 schema、返回说明。如果你接了 8 个 MCP server，每个 server 暴露 20 个工具，会话一开始就可能有 160 个工具描述。它们会占用 token，也会增加模型选择工具时的噪音。延迟加载的目标是：先不把低概率用到的工具全部展示给 Claude，等任务真的需要时再加载。

可以把它想成 IDE 的 command palette：

```text
不延迟加载：
  会话开始
    -> 直接把所有 MCP tools 的完整说明放进上下文
    -> Claude 每次推理都看见全部工具
    -> 简单直接，但工具多时浪费上下文

延迟加载：
  会话开始
    -> Claude 先知道有一个 ToolSearch 能查工具
    -> MCP tools 的完整说明先不全部进入上下文
    -> 任务需要时，Claude 调用 ToolSearch 搜索相关工具
    -> Claude Code 加载匹配的 MCP tool 描述
    -> Claude 再调用 mcp__server__tool
```

举例：你配置了 `context7`、`playwright`、`github`、`sentry`、`postgres`、`slack` 六个 MCP server。用户只问“查一下 Next.js 最新 app router 文档并修改代码”。在延迟加载模式下，Claude 不需要一开始就看到 Sentry、Slack、Postgres 的全部工具；它可以先搜索“library docs / context7 / Next.js”，加载 Context7 相关工具，再调用 `mcp__context7__...`。

### 10.1 延迟加载不等于 server 没连接

这里容易误解。延迟加载的是“tool 描述进入模型上下文”的时机，不是说 `.mcp.json` 没生效，也不是说 server 永远没启动。

更准确地说：

- MCP server 配置仍然会被 Claude Code 读取。
- server 是否启动/连接取决于 transport、scope、审批状态和 Claude Code 的运行策略。
- tool 的完整说明可能不会在会话开始时全部给模型。
- Claude 可以通过 `ToolSearch` 按需发现和加载具体 tool。
- 一旦 tool 被加载，后续就能按普通 MCP tool 调用，名称仍是 `mcp__<server>__<tool>`。

### 10.2 为什么默认要这么做

默认延迟加载有三个好处：

- 节省上下文：工具描述少占 token，留更多空间给代码、错误日志、设计说明和对话历史。
- 降低干扰：Claude 不必在大量无关工具中选择，减少误用工具的概率。
- 支持大规模 MCP：团队可以配置更多 server，而不是因为工具列表太大导致会话一开始就臃肿。

代价是：Claude 第一次使用某类 MCP tool 前，可能多一步 `ToolSearch`。这一步是为了找到正确工具，通常比把所有工具一直放进上下文更划算。

### 10.3 如何控制 Tool Search

Claude Code 用 `ENABLE_TOOL_SEARCH` 控制行为：

```bash
# 默认行为：能延迟就延迟；部分平台或代理不支持时会回退
claude

# 强制启用延迟加载
ENABLE_TOOL_SEARCH=true claude

# 阈值模式：如果工具描述超过上下文窗口 5%，就延迟加载
ENABLE_TOOL_SEARCH=auto:5 claude

# 禁用延迟加载：所有 MCP tools 在会话开始时 upfront 加载
ENABLE_TOOL_SEARCH=false claude
```

几种模式的含义：

| 值 | 行为 |
| --- | --- |
| 未设置 | 默认延迟加载；在不支持的环境中回退为 upfront 加载 |
| `true` | 强制所有 MCP tools deferred，通过 Tool Search 按需发现 |
| `auto` | 如果工具描述占上下文比例不高，就 upfront；太多时 deferred |
| `auto:N` | 自定义阈值，`N` 是 0-100 的百分比 |
| `false` | 禁用 Tool Search，所有 MCP tools upfront 加载 |

如果你想在配置里关闭 `ToolSearch` 这个工具，也可以用权限规则：

```json
{
  "permissions": {
    "deny": ["ToolSearch"]
  }
}
```

### 10.4 什么时候使用 `alwaysLoad`

如果某个 MCP server 的工具很少，而且你希望 Claude 每个 turn 都能直接看到它，不想先经过搜索，可以给这个 server 加 `alwaysLoad: true`：

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

使用场景：

- server 只暴露 1-3 个很核心的工具。
- 几乎每个任务都会用到它。
- 你希望 Claude 不需要搜索就能直接调用。

不要给很多 server 都加 `alwaysLoad`。这会把延迟加载的收益抵消掉，让上下文又被大量工具描述占满。

## 11. 用 FastMCP 自建 MCP server 后，如何让 Claude Code 发现

FastMCP 是 Python 生态里搭建 MCP server 的常用方式。它让你用 Python 函数、装饰器和类型标注定义 MCP tools、resources、prompts。Claude Code 发现 FastMCP server 的关键不在“Claude 自动扫描你的 Python 文件”，而在你必须把这个 server 注册到 Claude Code 的 MCP 配置里。

完整链路是：

```text
你写 server.py
  -> 用 FastMCP 声明 tools/resources/prompts
  -> 用 fastmcp install 或 claude mcp add 注册 server
  -> Claude Code 读取 MCP 配置
  -> Claude Code 启动 stdio server 或连接 HTTP server
  -> server 向 Claude Code 报告可用 tools/resources/prompts
  -> Claude Code 把这些能力暴露给 Claude
  -> Claude 在任务中通过 ToolSearch 或直接工具列表发现它们
```

也就是说，“发现”分两层：

- 客户端发现 server：Claude Code 从 `.mcp.json`、`~/.claude.json`、local scope 或 `claude mcp add` 的配置里知道要连接哪个 server。
- 模型发现 tool：Claude Code 连接 server 后，server 通过 MCP 协议列出 tools/resources/prompts；Claude 再通过 upfront tool list 或 `ToolSearch` 找到具体工具。

### 11.1 写一个最小 FastMCP server

`server.py`：

```python
from fastmcp import FastMCP

mcp = FastMCP(name="Dice Roller")


@mcp.tool
def roll_dice(n_dice: int) -> list[int]:
    """Roll n six-sided dice and return the results."""
    import random

    return [random.randint(1, 6) for _ in range(n_dice)]


@mcp.resource("dice://rules")
def dice_rules() -> str:
    """Basic dice rules."""
    return "Each die has six sides. roll_dice(n_dice) returns n random values from 1 to 6."


if __name__ == "__main__":
    mcp.run()
```

这段代码定义了：

- 一个 tool：`roll_dice`，Claude 可以调用它。
- 一个 resource：`dice://rules`，Claude 可以引用它作为上下文。
- 默认 transport：stdio。FastMCP 的 `mcp.run()` 默认使用 stdio，适合 Claude Code 本地启动。

### 11.2 推荐方式：用 FastMCP 自动注册到 Claude Code

FastMCP 2.10.3 之后提供了 `fastmcp install claude-code`。它会调用 Claude Code 内置的 MCP 管理系统，把你的本地 server 注册进去。

```bash
fastmcp install claude-code server.py
```

如果你的 server 对象不叫 `mcp`，可以显式指定：

```bash
fastmcp install claude-code server.py:my_custom_server
```

指定 server 名：

```bash
fastmcp install claude-code server.py \
  --server-name "dice-roller"
```

有依赖时要显式声明，因为 MCP 客户端通常在隔离环境中运行 server，不能假设它能访问你当前 shell 里的所有包：

```bash
fastmcp install claude-code server.py \
  --server-name "data-tools" \
  --with pandas \
  --with requests
```

使用 requirements 文件：

```bash
fastmcp install claude-code server.py \
  --with-requirements requirements.txt
```

传环境变量：

```bash
fastmcp install claude-code server.py \
  --server-name "weather-server" \
  --env API_KEY=your-api-key \
  --env DEBUG=true
```

或从 `.env` 文件读取：

```bash
fastmcp install claude-code server.py \
  --server-name "weather-server" \
  --env-file .env
```

安装完成后，重新打开或刷新 Claude Code 会话。然后你可以直接问：

```text
Roll three dice for me.
```

Claude Code 连接到这个 MCP server 后，Claude 就能看到或搜索到 `roll_dice` 这个工具，并调用它。

### 11.3 手工方式：用 `claude mcp add` 注册 stdio server

如果你想完全控制 Claude Code 怎么启动 FastMCP server，可以手工注册：

```bash
claude mcp add dice-roller -- uv run --with fastmcp fastmcp run /absolute/path/to/server.py
```

这个命令的意思是：

- `dice-roller`：Claude Code 里这个 MCP server 的名字。
- `--`：前面是 Claude Code 的配置参数，后面是实际启动 server 的命令。
- `uv run --with fastmcp ...`：用 `uv` 创建/使用隔离环境，并确保安装 `fastmcp`。
- `fastmcp run /absolute/path/to/server.py`：启动你的 FastMCP server，默认 stdio transport。

带环境变量：

```bash
claude mcp add weather-server \
  --env API_KEY=secret \
  --env DEBUG=true \
  -- uv run --with fastmcp fastmcp run /absolute/path/to/server.py
```

指定 scope：

```bash
# 只给当前项目本地使用
claude mcp add dice-roller --scope local -- uv run --with fastmcp fastmcp run /absolute/path/to/server.py

# 所有项目通用
claude mcp add dice-roller --scope user -- uv run --with fastmcp fastmcp run /absolute/path/to/server.py

# 写入项目共享配置
claude mcp add dice-roller --scope project -- uv run --with fastmcp fastmcp run /absolute/path/to/server.py
```

### 11.4 手写 `.mcp.json` 让 Claude Code 发现

如果你希望团队共享这个 server，可以把配置写进仓库根目录的 `.mcp.json`：

```json
{
  "mcpServers": {
    "dice-roller": {
      "command": "uv",
      "args": [
        "run",
        "--with",
        "fastmcp",
        "fastmcp",
        "run",
        "${CLAUDE_PROJECT_DIR:-.}/tools/mcp/dice_server.py"
      ]
    }
  }
}
```

这里的重点：

- `.mcp.json` 只负责告诉 Claude Code 如何启动 server。
- `command` 是要运行的程序。
- `args` 是传给程序的参数。
- `dice-roller` 是 server 名，会进入工具名，例如 `mcp__dice-roller__roll_dice`。
- `${CLAUDE_PROJECT_DIR:-.}` 让配置能从项目根目录定位文件；给默认值是为了避免某些配置展开时变量不存在。

配合 `.claude/settings.json` 自动批准这个项目 server：

```json
{
  "enabledMcpjsonServers": ["dice-roller"],
  "permissions": {
    "allow": [
      "mcp__dice-roller__roll_dice"
    ]
  }
}
```

### 11.5 HTTP 方式：FastMCP server 作为远端服务

如果你希望多人共享一个 FastMCP server，或者 server 部署在远端机器上，可以用 HTTP transport。

本地启动 HTTP server：

```bash
fastmcp run server.py --transport http --host 127.0.0.1 --port 8000
```

FastMCP 默认 MCP endpoint 通常是：

```text
http://127.0.0.1:8000/mcp/
```

然后注册到 Claude Code：

```bash
claude mcp add --transport http dice-roller http://127.0.0.1:8000/mcp/
```

或写入 `.mcp.json`：

```json
{
  "mcpServers": {
    "dice-roller": {
      "type": "http",
      "url": "http://127.0.0.1:8000/mcp/"
    }
  }
}
```

stdio 和 HTTP 的选择：

| 方式 | Claude Code 如何连接 | 适合场景 |
| --- | --- | --- |
| stdio | Claude Code 启动本地进程 | 本地开发、个人工具、单用户 server |
| HTTP | 你先启动/部署 server，Claude Code 连 URL | 多客户端共享、远端服务、团队平台 |

### 11.6 如何确认 Claude Code 已经发现

安装或配置后，可以按这个顺序排查：

1. 确认 server 能独立运行：

```bash
fastmcp run server.py
```

2. 如果用了 `claude mcp add`，查看 Claude Code MCP 配置：

```bash
claude mcp list
```

3. 重启 Claude Code 会话，或按当前环境支持的方式刷新 MCP 配置。

4. 在 Claude Code 中明确要求使用这个 server：

```text
Use the dice-roller MCP server to roll three dice.
```

5. 如果工具很多，Claude 可能先用 `ToolSearch` 找到 `roll_dice`，再调用 `mcp__dice-roller__roll_dice`。

6. 如果看不到工具，检查：

- server 名是否和权限规则一致。
- `.mcp.json` 是否在项目根目录。
- 项目 MCP server 是否被批准。
- `uv`、`fastmcp`、Python 版本和依赖是否可用。
- stdio server 是否把日志写到了 stdout。MCP stdio 通信要求 stdout 保持协议消息，普通日志应写 stderr。
- HTTP server URL 是否包含正确路径，例如 `/mcp/`。

### 11.7 什么时候用 FastMCP

FastMCP 适合：

- 把你已有 Python 函数包装成 Claude Code tools。
- 连接内部 API、数据库、文件处理脚本。
- 快速做团队内部 MCP 原型。
- 同时暴露 tools、resources、prompts。

如果你的 server 只是给 Claude Code 本地用，优先 stdio。如果要被多人或多个客户端共用，优先 HTTP。

## 12. 经典 MCP 组合

来自 best-practice 文档的建议可以总结为：

| 阶段 | 推荐 MCP | 用途 |
| --- | --- | --- |
| Research | Context7 / DeepWiki | 查最新库文档、仓库架构、API 关系 |
| Debug | Playwright / Claude in Chrome | 浏览器自动化、截图、DOM/console/network 调试 |
| Document | Excalidraw | 生成架构图、流程图、系统设计草图 |

经验重点不是“装越多越好”，而是装少数高频、可信、权限清楚的 server。太多 MCP server 会增加上下文噪音、权限复杂度和 prompt injection 风险。

## 13. 安全注意事项

MCP server 会把 Claude Code 连接到真实外部系统，所以风险比普通上下文更高。

建议：

- 只连接你信任的 server。
- 谨慎连接会读取网页、issue、邮件、Slack 的 server，因为外部内容可能包含 prompt injection。
- 不把 API key 提交进 `.mcp.json`。
- 对写操作工具设置更严格权限。
- 团队共享 `.mcp.json` 前做 review。
- 对远程 server 使用 HTTPS 和明确域名。
- 组织环境使用 managed MCP config、allowlist、denylist。

## 14. 最小可用实践

如果刚开始给 Claude Code 配 MCP，可以从这个组合开始：

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
    }
  }
}
```

再配合权限：

```json
{
  "permissions": {
    "allow": [
      "mcp__context7__*",
      "mcp__playwright__browser_snapshot",
      "mcp__playwright__browser_navigate"
    ]
  },
  "enabledMcpjsonServers": ["context7", "playwright"]
}
```

这个起点覆盖两类最常用能力：查最新文档、验证浏览器界面。等确实需要 GitHub、Sentry、数据库、Slack 等系统时，再逐个增加 server。
