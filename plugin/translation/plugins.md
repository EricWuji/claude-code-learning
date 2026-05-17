# 中文精读翻译：Create plugins

原文：https://code.claude.com/docs/en/plugins

说明：本文是 Claude Code 官方 plugin 创建文档的中文精读翻译与结构化改写，不是逐字全文搬运。

## 文档定位

这篇文档讲如何创建 Claude Code plugin。plugin 可以把 skills、agents、hooks、MCP servers、LSP servers 和后台 monitors 打包成一个可共享、可版本化、可分发的扩展。

如果只是想安装现成插件，应读“Discover and install plugins”。如果要完整技术规格，应查 plugins reference。

## 什么时候用 plugin，什么时候用 standalone 配置

Claude Code 支持两种扩展方式：

- standalone 配置：项目里的 `.claude/` 目录。
- plugin：带 `.claude-plugin/plugin.json` 的插件目录。

standalone 适合：

- 单项目定制。
- 个人工作流。
- 快速试验 skills 或 hooks。
- 想使用短命令名，例如 `/hello`。

plugin 适合：

- 分享给团队或社区。
- 跨多个项目复用。
- 需要版本控制和更新机制。
- 通过 marketplace 分发。
- 接受命名空间 skill，例如 `/my-plugin:hello`。

官方建议：先用 `.claude/` 快速迭代，稳定后再转成 plugin。

## 快速开始

创建一个插件的最小步骤：

1. 创建插件目录。
2. 创建 `.claude-plugin/plugin.json` manifest。
3. 添加 skill。
4. 用 `--plugin-dir` 本地测试。
5. 给 skill 增加参数。

### 创建目录

```bash
mkdir my-first-plugin
mkdir my-first-plugin/.claude-plugin
```

### 创建 manifest

`my-first-plugin/.claude-plugin/plugin.json`：

```json
{
  "name": "my-first-plugin",
  "description": "A greeting plugin to learn the basics",
  "version": "1.0.0",
  "author": {
    "name": "Your Name"
  }
}
```

字段含义：

- `name`：唯一标识，也是 skill 命名空间。
- `description`：在插件管理器中展示。
- `version`：可选。设置后，只有版本号变化用户才收到更新；不设置时，git 分发通常使用 commit SHA。
- `author`：可选，用于归属。

### 添加 skill

Skills 位于插件根目录的 `skills/` 下。每个 skill 是一个包含 `SKILL.md` 的目录：

```bash
mkdir -p my-first-plugin/skills/hello
```

`my-first-plugin/skills/hello/SKILL.md`：

```markdown
---
description: Greet the user with a friendly message
disable-model-invocation: true
---

Greet the user warmly and ask how you can help them today.
```

插件名是 `my-first-plugin`，skill 目录名是 `hello`，所以命令是：

```text
/my-first-plugin:hello
```

命名空间的原因是避免多个插件拥有同名 skill 时冲突。

### 本地测试

```bash
claude --plugin-dir ./my-first-plugin
```

启动后运行：

```text
/my-first-plugin:hello
```

### 增加 skill 参数

`$ARGUMENTS` 会捕获用户在 skill 名后输入的文本：

```markdown
---
description: Greet the user with a personalized message
---

# Hello Skill

Greet the user named "$ARGUMENTS" warmly and ask how you can help them today.
```

重新加载插件：

```text
/reload-plugins
```

然后调用：

```text
/my-first-plugin:hello Alex
```

## Plugin 目录结构

插件不仅能包含 skill，还能包含 agents、hooks、MCP servers、LSP servers 和 monitors。

重要规则：不要把 `commands/`、`agents/`、`skills/`、`hooks/` 放在 `.claude-plugin/` 目录里。`.claude-plugin/` 只放 `plugin.json`。

常见路径：

- `.claude-plugin/plugin.json`：manifest。
- `skills/`：推荐的新式 skill 目录。
- `commands/`：扁平 markdown 命令；新插件优先使用 `skills/`。
- `agents/`：自定义 agent。
- `hooks/hooks.json`：事件 hook。
- `.mcp.json`：MCP server 配置。
- `.lsp.json`：LSP server 配置。
- `monitors/monitors.json`：后台 monitor。
- `bin/`：插件启用时加入 Bash `PATH` 的可执行文件。
- `settings.json`：插件默认 settings。

## 开发更复杂的 plugin

### 添加 skills

Plugin skill 的 `SKILL.md` 应包含 frontmatter，特别是 `description`。Claude 通过 description 判断什么时候使用该 skill。

示例：

```markdown
---
description: Reviews code for best practices and potential issues. Use when reviewing code, checking PRs, or analyzing code quality.
---

When reviewing code, check for:
1. Code organization and structure
2. Error handling
3. Security concerns
4. Test coverage
```

安装或修改后，运行 `/reload-plugins`。

### 添加 LSP servers

LSP plugin 给 Claude Code 提供语言级代码智能。常见语言优先使用官方 marketplace 中的预构建 LSP 插件。只有需要支持未覆盖语言时才自建。

`.lsp.json` 示例：

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

用户安装插件后，本机仍需要安装对应 language server binary。

### 添加 background monitors

monitor 可以在后台监听日志、文件或外部状态。插件启用时，Claude Code 会自动启动 monitor。

`monitors/monitors.json`：

```json
[
  {
    "name": "error-log",
    "command": "tail -F ./logs/error.log",
    "description": "Application error log"
  }
]
```

命令 stdout 每一行都会作为通知发送给 Claude。

### 默认 settings

插件根目录可以提供 `settings.json`。当前主要支持 `agent` 和 `subagentStatusLine`。

```json
{
  "agent": "security-reviewer"
}
```

这会在插件启用后激活插件内的 `security-reviewer` agent 作为主线程 agent。`settings.json` 的优先级高于 `plugin.json` 中声明的 settings；未知字段会被忽略。

## 组织复杂 plugin

复杂插件应按功能组织目录。完整目录布局和组织模式应参考 plugin directory structure 和 plugins reference。

## 本地测试

开发时用：

```bash
claude --plugin-dir ./my-plugin
```

也可以加载 zip：

```bash
claude --plugin-dir ./my-plugin.zip
```

如果本地插件和已安装 marketplace 插件同名，本地 `--plugin-dir` 在当前 session 中优先。被 managed settings 强制启用/禁用的插件例外。

修改后运行：

```text
/reload-plugins
```

这个命令会重新加载 plugins、skills、agents、hooks、plugin MCP servers 和 plugin LSP servers。

测试点：

- skill 能否通过 `/plugin-name:skill-name` 调用。
- agent 是否出现在 `/agents`。
- hooks 是否按预期触发。

可以同时加载多个插件：

```bash
claude --plugin-dir ./plugin-one --plugin-dir ./plugin-two
```

也可以用 `--plugin-url` 从 zip URL 加载测试，但只应使用你信任的 URL。

## 调试 plugin

如果插件不工作：

1. 检查目录结构，确保组件目录在插件根目录，不在 `.claude-plugin/` 内。
2. 单独测试 skill、agent、hook 等组件。
3. 使用官方 validation 和 debugging 工具。

## 分享 plugin

准备分享时：

1. 添加 README，写清安装和使用方法。
2. 选择版本策略：显式 `version` 或 git commit SHA。
3. 创建或使用 marketplace 分发。
4. 让团队成员先测试。

如果要提交到 Anthropic 官方 marketplace，可使用 Claude.ai 或 Console 中的提交表单。

## 从 standalone 配置迁移到 plugin

如果已有 `.claude/` 配置，可以迁移成 plugin。

创建结构：

```bash
mkdir -p my-plugin/.claude-plugin
```

manifest：

```json
{
  "name": "my-plugin",
  "description": "Migrated from standalone configuration",
  "version": "1.0.0"
}
```

复制配置：

```bash
cp -r .claude/commands my-plugin/
cp -r .claude/agents my-plugin/
cp -r .claude/skills my-plugin/
```

hooks 需要迁移到 `my-plugin/hooks/hooks.json`，格式复制原 settings 中的 `hooks` 对象。

本地测试：

```bash
claude --plugin-dir ./my-plugin
```

迁移后变化：

- standalone 只在一个项目可用，plugin 可通过 marketplace 分享。
- `.claude/commands/` 变为插件目录下的 `commands/`。
- settings 中的 hooks 变为插件的 `hooks/hooks.json`。
- 安装方式从手工复制变成 `/plugin install`。

## 下一步

用户可继续学习如何发现和安装插件、如何配置团队 marketplace。开发者可继续阅读 marketplace 分发、plugins reference、skills、subagents、hooks 和 MCP 文档。

