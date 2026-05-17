# 中文精读翻译：Hooks Reference

原文：https://code.claude.com/docs/en/hooks

说明：本文是官方 hooks reference 的中文结构化翻译与学习版摘要，不是逐字全文搬运。

## 文档定位

Hooks reference 是 Claude Code hooks 的技术参考，覆盖：

- 生命周期事件。
- 配置 schema。
- JSON 输入与输出格式。
- exit code 语义。
- 异步 hook。
- HTTP hook。
- prompt hook。
- MCP tool hook。
- 安全建议和调试方法。

如果第一次使用 hook，先读 guide；需要查事件字段、决策格式、边界条件时，再查 reference。

## 生命周期

hook 在 Claude Code 会话中的特定点触发。事件触发且 matcher 命中后，Claude Code 会把事件上下文传给 hook handler。

三类高频节奏：

- 每个会话一次：`SessionStart`、`SessionEnd`。
- 每个 turn 一次：`UserPromptSubmit`、`Stop`、`StopFailure`。
- agentic loop 中每次工具调用：`PreToolUse`、`PermissionRequest`、`PostToolUse`、`PostToolUseFailure`。

还有一些独立/异步事件，例如 `Notification`、`ConfigChange`、`CwdChanged`、`FileChanged`、`WorktreeCreate`、`WorktreeRemove`。

## 配置结构

典型配置形态：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "path/to/script"
          }
        ]
      }
    ]
  }
}
```

含义：

- 第一层 key 是事件名。
- 每个事件下是 hook group 数组。
- `matcher` 用于筛选该事件下的具体情况。
- 内层 `hooks` 是真正执行的 handler 列表。
- handler 可以是 `command`、`http`、`prompt`、`agent` 等类型。

## 输入格式

所有 hook 输入通常包含公共字段：

- `session_id`：当前会话 ID。
- `transcript_path`：会话 transcript 文件路径。
- `cwd`：当前工作目录。
- `hook_event_name`：事件名。

不同事件会追加自己的字段。例如：

- `PreToolUse` / `PostToolUse`：包含工具名、工具输入、工具响应或错误。
- `Notification`：包含通知类型和消息。
- `ConfigChange`：包含配置来源和文件路径。
- `CwdChanged`：包含旧目录和新目录。
- `PreCompact`：包含压缩触发方式和自定义压缩指令。
- `PostCompact`：包含压缩 summary。
- `SessionEnd`：包含退出原因。

## 输出格式

Claude Code 会读取 stdout 中的 JSON 来控制行为，但只在 hook 以 `exit 0` 结束时处理 JSON。不要让 shell profile、调试日志或额外文本污染 stdout。

通用字段：

- `continue`：为 `false` 时停止 Claude 后续处理。
- `stopReason`：停止原因，展示给用户。
- `suppressOutput`：隐藏 stdout debug 输出。
- `systemMessage`：展示给用户的警告信息。

事件相关字段：

- `decision` / `reason`：用于阻止、允许、反馈。
- `hookSpecificOutput`：用于更复杂的事件输出，必须包含 `hookEventName`。

输出注入上下文时有长度限制；过长内容会写入会话文件并用预览替代。

## exit code

常用约定：

- `0`：hook 成功。Claude Code 可读取 stdout 中的 JSON。
- `2`：对支持决策控制的事件，通常表示阻止或反馈。
- 其他非零：hook 失败，通常不会被当作正常策略结果。

使用结构化 JSON 时，保持 stdout 只有 JSON 对象。

## 决策控制

### `PreToolUse`

适合在工具执行前做安全检查。可以阻止工具调用，也可以给 Claude 返回原因，让它调整计划。

典型场景：

- 禁止 `rm -rf`。
- 禁止编辑 `.env`。
- 禁止修改生成文件。
- 把不合规命令引导到项目脚本，例如要求使用 `make test`。

### `PermissionRequest`

在权限对话框显示前触发。可以返回批准、拒绝或修改当前 session 权限模式。应该只匹配非常窄的工具或场景。

### `PostToolUse`

工具成功后触发，适合格式化、审计、运行非阻塞检查、追加上下文。它不能阻止已经完成的工具动作。

### `Stop`

Claude 完成响应时触发。可以用于检查任务是否完整；如果 hook 判断未完成，可以阻止停止并把 reason 反馈给 Claude，让它继续。

注意：`Stop` 每次响应结束都可能触发，不等于“整个用户任务已经完成”。

### `PreCompact` / `PostCompact`

`PreCompact` 可在压缩前阻止压缩，`PostCompact` 可在压缩后读取 summary 并执行后续动作。压缩 hook 适合保护关键上下文。

## prompt-based hook

当判断不是简单规则，而是需要语义判断时，可以使用 `type: "prompt"`。Claude Code 会把 hook 输入和你的 prompt 发给模型，模型返回：

```json
{ "ok": true }
```

或：

```json
{ "ok": false, "reason": "未满足的条件" }
```

适合场景：

- 判断最终回答是否覆盖用户要求。
- 判断提交信息是否符合规范。
- 判断工具调用是否看起来符合某条语义规则。

不适合高安全强制策略。高安全策略应该尽量用确定性 command hook。

## agent-based hook

`type: "agent"` hook 会启动子 agent。它可以读文件、搜索代码、运行工具，然后返回 `{ "ok": true/false }`。

适合需要检查真实代码库状态的场景：

- 停止前确认测试是否通过。
- 检查生成代码是否触及特定架构边界。
- 搜索仓库确认某个约束是否被破坏。

官方文档标注 agent hook 为实验性，生产流程优先使用 command hook。

## HTTP hook

HTTP hook 把事件 JSON 发送给 HTTP endpoint。endpoint 返回 JSON body，语义与 command hook stdout 类似。

适合：

- 团队共享审计服务。
- 统一策略中心。
- 把 hook 接入内部平台或云函数。

注意：HTTP status code 本身不能表达阻止语义；要通过 response body 中的 hook 输出字段控制。

## async hook

command hook 可配置 `async: true`。异步 hook 启动后，Claude Code 不等待它完成，会继续执行。

适合：

- 长测试。
- 构建。
- 外部 API 调用。
- 后台日志上传。

限制：

- 不能阻止工具调用。
- 不能返回会影响当前动作的决策。
- 输出通常在后续 turn 作为上下文返回。
- prompt hook 不支持 async。

## Windows PowerShell

Windows 可在 command hook 中设置：

```json
{
  "type": "command",
  "shell": "powershell",
  "command": "Write-Host 'File written'"
}
```

Claude Code 会直接启动 PowerShell，通常会自动检测 PowerShell 7 的 `pwsh.exe`，否则回退到 Windows PowerShell 5.1 的 `powershell.exe`。

注意：如果 PowerShell profile 启动时输出文本，可能污染 JSON stdout，导致 JSON validation failed。需要保证返回 JSON 的 hook stdout 只有 JSON。

## 安全建议

command hook 拥有当前系统用户权限。它可以访问、修改、删除当前用户可访问的文件，所以应当像对待本地自动化脚本一样审查。

建议：

- 验证并清洗输入。
- shell 变量加引号。
- 防止路径穿越。
- 使用绝对路径。
- 跳过 `.env`、密钥、`.git/` 等敏感文件。
- 不要自动批准过宽权限。
- 对团队共享 hook 做代码审查。

