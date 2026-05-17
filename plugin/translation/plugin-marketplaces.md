# 中文精读翻译：Create and distribute a plugin marketplace

原文：https://code.claude.com/docs/en/plugin-marketplaces

说明：本文是 Claude Code 官方 marketplace 文档的中文精读翻译与结构化改写，不是逐字全文搬运。

## 文档定位

Plugin marketplace 是一个插件目录，用于把 Claude Code 插件分发给团队或社区。它提供集中发现、版本跟踪、自动更新，并支持多种插件来源，例如 git 仓库、本地路径、npm 包等。

如果只是想安装现成插件，应读“Discover and install plugins”。如果要创建插件本体，应读“Create plugins”。

## 创建 marketplace 的整体流程

分发 marketplace 通常包含四步：

1. 创建一个或多个 plugin，plugin 可包含 skills、agents、hooks、MCP servers、LSP servers。
2. 创建 `marketplace.json`，列出插件以及它们的来源。
3. 托管 marketplace，例如 GitHub、GitLab 或其他 git host。
4. 让用户通过 `/plugin marketplace add` 添加 marketplace，并安装其中插件。

Marketplace 上线后，维护者通过 push 更新仓库，用户通过 `/plugin marketplace update` 刷新本地副本。

## 本地 marketplace walkthrough

官方示例创建一个包含 `quality-review` skill 的 marketplace。

目录：

```bash
mkdir -p my-marketplace/.claude-plugin
mkdir -p my-marketplace/plugins/quality-review-plugin/.claude-plugin
mkdir -p my-marketplace/plugins/quality-review-plugin/skills/quality-review
```

plugin manifest：

```json
{
  "name": "quality-review",
  "description": "Code quality review plugin",
  "version": "1.0.0"
}
```

skill：

```markdown
---
description: Review code for quality, maintainability, and potential issues
---

Review the provided code for quality, maintainability, and potential issues.
```

marketplace file：

```json
{
  "name": "my-marketplace",
  "description": "My local plugin marketplace",
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

添加并测试：

```text
/plugin marketplace add ./my-marketplace
/plugin install quality-review@my-marketplace
/quality-review:quality-review
```

## Marketplace schema

`marketplace.json` 描述 marketplace 的名字、所有者、插件列表和可选元数据。

常见结构：

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

必需字段：

- `name`：marketplace 名称。
- `plugins`：插件条目数组。

owner 字段可用于展示负责人或组织信息。可选字段可以提供描述、主页、仓库、license 等。

## Plugin entries

每个 plugin entry 至少包含：

- `name`：插件名。
- `source`：插件来源。

可选字段可覆盖展示信息或提供额外 metadata。

## Plugin sources

Marketplace 支持多种插件来源。

### 相对路径

```json
{
  "name": "quality-review",
  "source": {
    "source": "path",
    "path": "./plugins/quality-review-plugin"
  }
}
```

适合 marketplace 仓库中直接包含插件目录。

### GitHub repository

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

`ref` 可以是 branch、tag 或 commit SHA。

### Git repository

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

适合非 GitHub git host。

### Git subdirectory

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

适合 monorepo。

### npm package

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

适合通过 npm 分发插件包。

## Advanced plugin entries

高级条目可包含更多 metadata、依赖、版本约束或严格模式。严格模式可限制 marketplace 中不符合要求的字段或结构，适合团队治理。

## 托管和分发 marketplace

官方推荐用 GitHub 托管，因为用户可以用 `owner/repo` 简写添加：

```text
/plugin marketplace add acme-corp/claude-plugins
```

也可以使用其他 git 服务：

```text
/plugin marketplace add https://gitlab.example.com/team/plugins.git
```

或者直接提供远程 `marketplace.json` URL：

```text
/plugin marketplace add https://example.com/marketplace.json
```

私有仓库也可使用，但用户需要具备对应认证权限。

## 本地测试

分享前应先校验：

```bash
claude plugin validate .
```

或在 Claude Code 内：

```text
/plugin validate .
```

添加本地 marketplace：

```text
/plugin marketplace add ./path/to/marketplace
```

安装测试插件：

```text
/plugin install test-plugin@marketplace-name
```

## 要求团队使用 marketplace

团队可以通过项目级或 managed settings 预配置 marketplace。这样成员进入项目时能看到统一的插件目录。

常见用途：

- 给所有开发者分发内部 review skill。
- 给特定团队分发项目专用 MCP/LSP 插件。
- 管理组织允许安装的插件来源。

## 容器预装插件

开发容器或 CI 环境可以预先添加 marketplace 和安装插件，让 Claude Code 在容器中开箱即用。适合标准化团队环境。

## Managed marketplace restrictions

组织管理员可以配置 marketplace 限制。常见策略：

- allowlist：只允许指定 marketplace。
- denylist：阻止指定 marketplace。
- extraKnownMarketplaces：预置额外 marketplace。
- 按用户组分配不同 marketplace。

这适合安全要求高的组织，避免开发者安装未知来源插件。

## 版本解析和发布通道

可以用不同 marketplace 或同一 marketplace 中不同 ref 形成发布通道，例如：

- stable：指向稳定 tag/branch。
- latest：指向最新 branch。

示例：

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

另一个 marketplace 可指向：

```json
{
  "name": "latest-tools",
  "plugins": [
    {
      "name": "code-formatter",
      "source": {
        "source": "github",
        "repo": "acme-corp/code-formatter",
        "ref": "latest"
      }
    }
  ]
}
```

如果插件显式声明 `version`，不同发布通道上的 `version` 必须不同；否则 Claude Code 会认为它们是同一版本并跳过更新。

## 依赖版本固定

插件可以限制依赖版本范围，防止依赖插件升级破坏当前插件。官方文档提到 `{plugin-name}--v{version}` git tag 约定、semver 范围和多个约束的合并规则。

## CLI 管理 marketplace

Claude Code 提供非交互 CLI 子命令，等价于交互 session 中的 `/plugin marketplace`。

添加 marketplace：

```bash
claude plugin marketplace add <source> [options]
```

`<source>` 可以是：

- GitHub `owner/repo` 简写。
- git URL。
- 远程 `marketplace.json` URL。
- 本地目录。

示例：

```bash
claude plugin marketplace add acme-corp/claude-plugins
claude plugin marketplace add acme-corp/claude-plugins@v2.0
claude plugin marketplace add https://gitlab.example.com/team/plugins.git
claude plugin marketplace add https://example.com/marketplace.json
claude plugin marketplace add ./my-marketplace
```

常用选项：

- `--scope user|project|local`：声明 marketplace 的作用域。
- `--sparse <paths...>`：只 checkout 指定路径，适合 monorepo。

还可以列出、移除、更新 marketplace。具体命令可用 `claude plugin marketplace --help` 查看。

## 常见故障

文档列出的排查方向包括：

- Marketplace 未加载：检查路径、URL、权限和 JSON 位置。
- JSON validation errors：用 validate 命令校验 schema。
- 插件安装失败：检查 source 是否可访问、插件目录是否包含 manifest。
- 私有仓库认证失败：检查 git 凭据和访问权限。
- 离线环境更新失败：确认网络和缓存策略。
- git 操作超时：检查仓库大小、网络和 sparse checkout。
- URL-based marketplace 中相对路径失效：相对路径更适合 git-hosted marketplace。
- 安装后文件找不到：检查 source path、subdirectory 和打包结构。

## 总结

Marketplace 是 plugin 的分发层。plugin 负责定义能力，marketplace 负责让别人发现、安装、更新和治理这些能力。团队内部推广 Claude Code 扩展时，推荐把成熟插件放进私有 marketplace，并通过项目或 managed settings 控制可见范围。

