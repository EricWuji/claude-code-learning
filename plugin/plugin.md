# Claude Code Plugin：作用、解决的问题与构建方式

资料来源：

- Claude Code Create plugins: https://code.claude.com/docs/en/plugins
- Claude Code Plugin marketplaces: https://code.claude.com/docs/en/plugin-marketplaces

## 1. Plugin 是什么

Claude Code 的 plugin 是一种可分发的扩展包。它可以把 Claude Code 的多种扩展能力打包到一个目录中，包括：

- skills：可被 Claude 自动调用或用户通过命令触发的能力说明。
- agents：自定义 subagent 或主线程 agent。
- hooks：在 Claude Code 生命周期节点自动执行的规则和脚本。
- MCP servers：给 Claude Code 接入外部工具、数据源、API。
- LSP servers：给 Claude Code 提供语言级代码智能。
- background monitors：后台监听日志、文件或外部状态，并把事件通知给 Claude。
- default settings：插件启用后应用的默认设置。
- bin executables：插件启用时加入 Bash 工具 `PATH` 的可执行文件。

一句话：plugin 是把一组 Claude Code 工作流能力“产品化、命名空间化、可安装、可升级、可团队共享”的包装形式。

## 2. Plugin 解决什么问题

如果只在某个项目里写 `.claude/commands`、`.claude/agents`、`.claude/settings.json`，这些配置可以工作，但有几个问题：

- 只能在当前项目使用，复制到别的项目很麻烦。
- 团队成员之间难以统一版本。
- 配置更新后没有清晰的分发和升级机制。
- skill/command 名字容易冲突，例如多个项目都有 `/review`。
- 很难把 skills、agents、hooks、MCP、LSP 作为一个完整能力包交付。

plugin 解决这些问题：

- 共享：一个插件可以给团队或社区安装。
- 复用：同一套能力可跨项目使用。
- 版本：通过 `plugin.json` 的 `version` 或 git commit 管理更新。
- 命名空间：插件 skill 使用 `/plugin-name:skill-name`，避免冲突。
- 分发：通过 marketplace 发布、安装、更新。
- 组合：把 skill、agent、hook、MCP、LSP、monitor 组合成一个完整工具包。

## 3. Plugin 和 standalone 配置怎么选

| 方式 | 目录/形态 | 适合场景 |
| --- | --- | --- |
| Standalone configuration | 单项目 `.claude/` 目录 | 个人工作流、项目专用配置、快速试验 |
| Plugin | 带 `.claude-plugin/plugin.json` 的插件目录 | 团队共享、跨项目复用、版本发布、marketplace 分发 |

建议路径：

1. 先在 `.claude/` 里快速试验 skill、agent、hook。
2. 稳定后再迁移到 plugin。
3. 如果要给团队使用，再创建 marketplace。

## 4. 最小 plugin 结构

一个最小插件长这样：

```text
my-first-plugin/
├── .claude-plugin/
│   └── plugin.json
└── skills/
    └── hello/
        └── SKILL.md
```

注意：`.claude-plugin/` 目录里只放 `plugin.json`。不要把 `skills/`、`agents/`、`hooks/` 放进 `.claude-plugin/` 里面，它们应该在插件根目录。

## 5. `plugin.json` 示例

`my-first-plugin/.claude-plugin/plugin.json`：

```json
{
  "name": "my-first-plugin",
  "description": "A greeting plugin to learn the basics",
  "version": "1.0.0",
  "author": {
    "name": "Your Name"
  },
  "repository": "https://github.com/your-org/my-first-plugin",
  "license": "MIT"
}
```

字段含义：

- `name`：插件唯一标识，也是 skill 命名空间。比如插件名是 `my-first-plugin`，skill `hello` 会变成 `/my-first-plugin:hello`。
- `description`：插件管理器中展示的描述。
- `version`：版本号。显式设置后，只有 bump 版本用户才会收到更新；不设置时，git 分发通常用 commit SHA 作为版本。
- `author`：作者信息。
- `repository` / `license`：发布和归属信息。

## 6. 添加一个 skill

`my-first-plugin/skills/hello/SKILL.md`：

```markdown
---
description: Greet the user with a personalized message
---

# Hello Skill

Greet the user named "$ARGUMENTS" warmly and ask how you can help them today.
```

启动 Claude Code 测试：

```bash
claude --plugin-dir ./my-first-plugin
```

在 Claude Code 中调用：

```text
/my-first-plugin:hello Alex
```

这里的 `$ARGUMENTS` 会捕获用户在 skill 名后面输入的参数。

## 7. Plugin 可以包含哪些目录

| 路径 | 作用 |
| --- | --- |
| `.claude-plugin/plugin.json` | 插件 manifest，描述名称、版本、作者等元数据 |
| `skills/<name>/SKILL.md` | 新式 skill 目录，推荐用于新插件 |
| `commands/*.md` | 旧式/扁平命令文件；新插件优先用 `skills/` |
| `agents/*.md` | 自定义 agents |
| `hooks/hooks.json` | 插件提供的 hooks |
| `.mcp.json` | 插件提供的 MCP server 配置 |
| `.lsp.json` | 插件提供的 LSP server 配置 |
| `monitors/monitors.json` | 后台 monitor 配置 |
| `bin/` | 插件启用时加入 Bash 工具 `PATH` 的可执行文件 |
| `settings.json` | 插件启用时应用的默认设置 |

## 8. 加入 hooks

插件中的 hook 放在 `hooks/hooks.json`：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path' | xargs npm run lint:fix"
          }
        ]
      }
    ]
  }
}
```

这和 `.claude/settings.json` 中的 hooks 格式一致，但插件化后可以随插件安装和更新。

## 9. 加入 MCP server

插件根目录可以放 `.mcp.json`：

```json
{
  "mcpServers": {
    "docs-search": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp"]
    }
  }
}
```

插件启用后，Claude Code 会把该 MCP server 纳入可用 MCP 配置。工具名仍遵循 `mcp__<server>__<tool>`。

## 10. 加入 LSP server

`my-plugin/.lsp.json`：

```json
{
  "go": {
    "command": "gopls",
    "args": ["serve"],
    "extensionToLanguage": {
      ".go": "go"
    }
  }
}
```

LSP 插件让 Claude Code 获得代码智能，例如跳转、诊断、符号信息。注意：用户机器上仍需要安装对应 language server binary，例如 `gopls`。

## 11. 加入 background monitor

`my-plugin/monitors/monitors.json`：

```json
[
  {
    "name": "error-log",
    "command": "tail -F ./logs/error.log",
    "description": "Application error log"
  }
]
```

插件启用时，Claude Code 会自动启动 monitor。命令 stdout 的每一行会作为通知进入 Claude Code session。适合日志、测试状态、外部系统状态监控。

## 12. 加入默认 settings

`my-plugin/settings.json`：

```json
{
  "agent": "security-reviewer"
}
```

这个示例表示插件启用后，把插件中的 `security-reviewer` agent 设为主线程 agent。官方文档说明当前默认 settings 支持范围有限，例如 `agent` 和 `subagentStatusLine`。

## 13. 本地测试 plugin

开发时不需要先发布 marketplace，可以直接加载本地插件：

```bash
claude --plugin-dir ./my-plugin
```

也可以加载 zip：

```bash
claude --plugin-dir ./my-plugin.zip
```

同时加载多个插件：

```bash
claude --plugin-dir ./plugin-one --plugin-dir ./plugin-two
```

改动插件后，在 Claude Code 中运行：

```text
/reload-plugins
```

它会重新加载 plugins、skills、agents、hooks、plugin MCP servers 和 plugin LSP servers。

验证点：

- skill 是否能通过 `/plugin-name:skill-name` 调用。
- agent 是否出现在 `/agents`。
- hook 是否按事件触发。
- MCP server 是否可连接。
- LSP 是否提供预期代码智能。

## 14. 从 `.claude/` 迁移到 plugin

如果你已有 standalone 配置，可以这样迁移：

```bash
mkdir -p my-plugin/.claude-plugin
```

创建 manifest：

```json
{
  "name": "my-plugin",
  "description": "Migrated from standalone configuration",
  "version": "1.0.0"
}
```

复制已有配置：

```bash
cp -r .claude/commands my-plugin/
cp -r .claude/agents my-plugin/
cp -r .claude/skills my-plugin/
```

hooks 不能直接复制 settings 文件，应把 `hooks` 对象放到：

```text
my-plugin/hooks/hooks.json
```

测试：

```bash
claude --plugin-dir ./my-plugin
```

迁移后，如果同名能力同时存在于 standalone 和 plugin，可能会出现重复。稳定后可以移除原来的 `.claude/` 配置，避免混淆。

## 15. Marketplace 是什么

plugin marketplace 是插件目录。它不是插件本身，而是一个 catalog，列出有哪些插件、每个插件在哪里、版本如何解析、如何更新。

Marketplace 解决的问题：

- 集中发现：用户从一个目录看到团队/社区插件。
- 集中分发：不用手工发 zip 或复制目录。
- 自动更新：用户可以刷新 marketplace，安装新版本。
- 多来源支持：插件可以来自 GitHub、git URL、本地路径、npm 包、子目录等。
- 团队治理：组织可以要求团队使用指定 marketplace。

## 16. 最小 marketplace 结构

```text
my-marketplace/
├── .claude-plugin/
│   └── marketplace.json
└── plugins/
    └── quality-review-plugin/
        ├── .claude-plugin/
        │   └── plugin.json
        └── skills/
            └── quality-review/
                └── SKILL.md
```

`my-marketplace/.claude-plugin/marketplace.json`：

```json
{
  "name": "team-tools",
  "description": "Internal Claude Code plugins for the engineering team",
  "owner": {
    "name": "Platform Team",
    "email": "platform@example.com"
  },
  "plugins": [
    {
      "name": "quality-review",
      "source": {
        "source": "path",
        "path": "./plugins/quality-review-plugin"
      }
    }
  ]
}
```

## 17. Marketplace plugin source 示例

### 17.1 相对路径

```json
{
  "name": "quality-review",
  "source": {
    "source": "path",
    "path": "./plugins/quality-review-plugin"
  }
}
```

适合 marketplace 仓库中包含插件目录。

### 17.2 GitHub repo

```json
{
  "name": "code-formatter",
  "source": {
    "source": "github",
    "repo": "acme-corp/code-formatter",
    "ref": "v1.0.0"
  }
}
```

### 17.3 Git URL

```json
{
  "name": "internal-tools",
  "source": {
    "source": "git",
    "url": "https://gitlab.example.com/team/internal-tools.git",
    "ref": "main"
  }
}
```

### 17.4 Git subdirectory

```json
{
  "name": "review-tools",
  "source": {
    "source": "github",
    "repo": "acme-corp/claude-plugins",
    "path": "plugins/review-tools",
    "ref": "main"
  }
}
```

### 17.5 npm package

```json
{
  "name": "npm-plugin",
  "source": {
    "source": "npm",
    "package": "@acme/claude-plugin",
    "version": "^1.0.0"
  }
}
```

## 18. 安装和管理 marketplace

在 Claude Code 里添加 marketplace：

```text
/plugin marketplace add ./my-marketplace
/plugin marketplace add acme-corp/claude-plugins
/plugin marketplace add https://example.com/marketplace.json
```

CLI 方式：

```bash
claude plugin marketplace add acme-corp/claude-plugins
claude plugin marketplace add acme-corp/claude-plugins@v2.0
claude plugin marketplace add https://gitlab.example.com/team/plugins.git
claude plugin marketplace add https://example.com/marketplace.json
claude plugin marketplace add ./my-marketplace
```

项目级共享 marketplace：

```bash
claude plugin marketplace add acme-corp/claude-plugins --scope project
```

安装插件：

```text
/plugin install quality-review@team-tools
```

更新 marketplace：

```text
/plugin marketplace update
```

## 19. 测试和校验 marketplace

校验 JSON：

```bash
claude plugin validate .
```

或在 Claude Code 中：

```text
/plugin validate .
```

本地测试：

```text
/plugin marketplace add ./path/to/marketplace
/plugin install test-plugin@marketplace-name
```

## 20. 版本与发布通道

Marketplace 可以通过不同 ref 或 SHA 提供不同发布通道，例如 `stable` 和 `latest`：

```json
{
  "name": "stable-tools",
  "plugins": [
    {
      "name": "code-formatter",
      "source": {
        "source": "github",
        "repo": "acme-corp/code-formatter",
        "ref": "stable"
      }
    }
  ]
}
```

另一个 marketplace 可以指向 `latest`。团队管理员可以通过 managed settings 把不同用户组绑定到不同 marketplace。

注意：两个通道必须解析到不同版本。如果 `plugin.json` 有显式 `version`，不同 ref 上的版本号也要不同；否则 Claude Code 会认为它们是同一个版本。

## 21. 构建 plugin 的推荐流程

一个稳妥流程：

1. 在 `.claude/` 中快速试验能力。
2. 稳定后创建插件目录和 `.claude-plugin/plugin.json`。
3. 把 skill、agent、hook、MCP、LSP、monitor 移入插件根目录。
4. 用 `claude --plugin-dir ./my-plugin` 本地测试。
5. 用 `/reload-plugins` 快速验证修改。
6. 补 README、版本号、license、repository。
7. 用 `claude plugin validate .` 校验。
8. 创建 marketplace，把插件登记进去。
9. 本地添加 marketplace 并安装插件测试。
10. 发布到 GitHub/git host 或内部私有仓库。

## 22. 安全和治理建议

- 只安装可信来源的 plugin。
- plugin 可能包含 hooks、MCP servers、bin executables，这些都可能执行本地命令或连接外部系统。
- 团队 marketplace 应该经过 review。
- 私有插件用私有仓库或组织 marketplace 管理。
- 对 hooks 和 MCP server 特别谨慎，因为它们可能在后台执行或访问真实外部系统。
- 使用项目级或 managed marketplace 控制团队默认可用的插件集合。

