# 中文精读翻译：Automate Workflows with Hooks

原文：https://code.claude.com/docs/en/hooks-guide

说明：本文是基于官方 guide 的中文精读翻译与结构化改写，不是逐字全文搬运。

## 核心定义

Claude Code hooks 是用户定义的自动化处理器，会在 Claude Code 生命周期中的特定时刻运行。最常见的是 shell command。它们让某些动作稳定发生，而不是依赖模型自己记得去做。

官方 guide 给出的典型目标包括：

- Claude 需要输入或权限时发送通知。
- Claude 编辑文件后自动格式化。
- 在修改敏感文件前阻止操作。
- 上下文压缩后重新注入关键项目背景。
- 审计配置变化。
- 当前目录或关键文件变化时刷新环境。
- 自动批准非常具体、低风险的权限请求。

## 快速开始

添加 hook 的基本方式是在 settings 文件中加入 `hooks` 配置块。用户级配置通常在 `~/.claude/settings.json`，项目级配置通常在 `.claude/settings.json`。

示例结构：

```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "your notification command"
          }
        ]
      }
    ]
  }
}
```

配置后，在 Claude Code 中运行 `/hooks` 可以查看 hook 是否被识别。`/hooks` 是浏览和检查入口；新增、修改、删除 hook 仍然要编辑 settings JSON，或让 Claude 帮你改配置。

## 常见自动化模式

### 通知

`Notification` 事件在 Claude Code 需要用户输入、权限批准或发送通知时触发。matcher 可以为空，表示匹配所有通知；也可以只匹配 `permission_prompt` 或 `idle_prompt` 等具体通知类型。

这个模式适合长任务：你可以离开终端，等系统通知提醒你回来处理权限或下一步输入。

### 自动格式化

`PostToolUse` 配合 `Edit|Write` matcher，可以在 Claude 修改文件后立即运行格式化工具。例如前端项目可以调用 Prettier，Go 项目可以调用 gofmt，Python 项目可以调用 ruff format。

关键点：

- `PostToolUse` 发生在工具成功之后。
- 它不能撤销已经发生的修改。
- 如果格式化失败，应把错误输出给 Claude 或用户，让后续 turn 能处理。

### 阻止敏感文件编辑

`PreToolUse` 发生在工具执行之前，所以它适合做强制策略。脚本读取 hook 输入里的 `tool_input.file_path`，命中 `.env`、锁文件、`.git/` 等敏感路径时返回阻止信号。

这个用法比在提示词中写“不要修改这些文件”更可靠，因为 hook 是代码级拦截。

### 压缩后重新注入上下文

当上下文窗口变满，Claude Code 会压缩对话。压缩可能丢失一些重要细节。可以用 `SessionStart` 的 `compact` matcher，在压缩后的会话开始阶段重新输出重要提醒，例如测试命令、当前 sprint、架构约束。

如果是每次会话都要加载的稳定背景，优先考虑 `CLAUDE.md`；如果是压缩后动态补充，hook 更合适。

### 审计配置变化

`ConfigChange` 在配置文件变化时触发。它适合记录谁在会话期间改变了 settings、skills 或相关配置，也可以按策略阻止变化。

matcher 可按配置来源过滤，例如用户 settings、项目 settings、本地 settings、托管策略 settings 或 skills。

### 目录/文件变化后刷新环境

`CwdChanged` 和 `FileChanged` 适合和 direnv、devbox、nix 等环境管理工具配合。目录变化或 `.envrc`、`.env` 等文件变化后，把环境导出到 `CLAUDE_ENV_FILE`，让 Claude 的 Bash 后续命令能拿到最新环境。

### 自动批准权限请求

`PermissionRequest` 可以在权限对话框出现前自动返回批准决策。官方文档特别强调 matcher 要尽可能窄，例如只批准 `ExitPlanMode`，不要空 matcher 自动批准全部操作。

自动批准是一把很锋利的工具，适合减少重复确认，不适合绕过安全边界。

## Hook 如何工作

事件触发后，Claude Code 会运行所有匹配 hook。多个 hook 并行执行，相同命令会去重。command hook 从 stdin 读取 JSON；HTTP hook 从 POST body 读取 JSON。hook 可以通过 exit code 或结构化 JSON 返回结果。

常见事件包括：

- 会话级：`SessionStart`、`SessionEnd`。
- turn 级：`UserPromptSubmit`、`Stop`、`StopFailure`。
- 工具级：`PreToolUse`、`PermissionRequest`、`PostToolUse`、`PostToolUseFailure`、`PostToolBatch`。
- 环境和配置：`ConfigChange`、`CwdChanged`、`FileChanged`。
- 上下文：`PreCompact`、`PostCompact`。
- 并发和团队：`SubagentStart`、`SubagentStop`、`TeammateIdle`、`TaskCreated`、`TaskCompleted`。

## handler 类型

大多数 hook 用 `type: "command"`。如果需要判断而不是固定规则，可以用：

- `prompt`：把 hook 输入交给模型判断，返回 `{ "ok": true }` 或 `{ "ok": false, "reason": "..." }`。
- `agent`：实验性。启动子 agent 检查文件、搜索代码、运行工具，再返回判断。
- `http`：把事件 JSON POST 到 HTTP 服务，由外部服务返回控制结果。

## 限制与故障排查

重要限制：

- command hook 只能通过 stdout、stderr、exit code、结构化 JSON 交流。
- `PostToolUse` 已经发生在动作之后，不能用来阻止这次工具调用。
- `PermissionRequest` 不在非交互 print mode 中触发，自动化场景应考虑 `PreToolUse`。
- 多个 `PreToolUse` hook 同时改写同一工具输入时，最后完成者生效，顺序不确定。
- `PreToolUse` 的 deny 可以在 bypass permission mode 中仍然阻止工具调用；但 allow 不能突破 settings 中的 deny 规则。

排查 hook 不触发：

- 运行 `/hooks` 确认配置已加载。
- 检查 matcher 大小写。
- 确认事件类型是否选对。
- 检查 JSON 是否有效。
- Windows 下注意 PowerShell profile 输出可能污染 JSON stdout。

