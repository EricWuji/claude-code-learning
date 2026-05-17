# Claude Code Hooks：作用、种类与使用示例

资料来源：

- Claude Code 官方 guide: https://code.claude.com/docs/en/hooks-guide
- Claude Code 官方 reference: https://code.claude.com/docs/en/hooks
- Anthropic blog: https://claude.com/blog/how-to-configure-hooks
- Zhihu: https://zhuanlan.zhihu.com/p/1986905872081917295 （本次抓取时页面返回错误，未能读取正文）

## 1. Hook 是什么

Claude Code 的 hook 是在 Claude Code 生命周期特定节点自动触发的处理器。最常见的形式是 shell command，也可以是 HTTP endpoint、prompt-based hook、agent-based hook 或 MCP tool hook。事件触发后，Claude Code 会把事件上下文以 JSON 传给 hook；hook 可以读取输入、执行脚本、返回结构化 JSON，甚至阻止工具调用或给 Claude 注入额外上下文。

它的核心价值是把“希望 Claude 遵守的规则”变成确定性执行的机制：

- 自动化重复动作：编辑后格式化、测试、记录日志。
- 强制项目规则：阻止修改 `.env`、`.git/`、锁文件等敏感文件。
- 提升交互效率：Claude 需要用户确认时弹系统通知。
- 注入动态上下文：会话开始、目录变化、压缩后重新补充项目状态。
- 组织治理与审计：记录配置变更、工具调用、权限请求。

一句话：`CLAUDE.md` 是“告诉模型应该怎么做”，hook 是“在运行时一定会执行的代码”。

## 2. Hook 的运行模型

基本流程：

1. Claude Code 到达某个生命周期事件，例如准备调用工具、工具调用完成、用户提交 prompt、会话结束。
2. Claude Code 根据 event 名和 matcher 判断哪些 hook 命中。
3. 命中的 hook 并行执行；相同命令会去重。
4. Claude Code 把事件 JSON 传给 hook。command hook 通过 stdin 接收；HTTP hook 通过 POST body 接收。
5. hook 通过 exit code、stdout/stderr 或结构化 JSON 控制后续行为。

常见配置位置：

- `~/.claude/settings.json`：用户级，对所有项目生效。
- `.claude/settings.json`：项目级，可提交到仓库。
- `.claude/settings.local.json`：项目级本地配置，通常不提交。
- 管理策略 settings：组织级。
- plugin 的 `hooks/hooks.json`：随插件启用。
- skill 或 agent frontmatter：在对应组件活跃时生效。

调试入口：

- 在 Claude Code 中运行 `/hooks` 查看已加载 hook。
- 使用 `--debug` 或 verbose 模式查看执行细节。
- matcher 区分大小写，事件类型要匹配实际触发点。

## 3. Hook 事件种类

官方 reference 中的 hook 事件较多，可以按“生命周期位置”理解。

| 事件 | 触发时机 | 典型用途 |
| --- | --- | --- |
| `Setup` | `--init-only`，或 print mode 下的 `--init` / `--maintenance` | CI/脚本里的显式初始化、一次性准备 |
| `SessionStart` | 会话开始或恢复 | 注入项目状态、加载环境变量 |
| `UserPromptSubmit` | 用户提交 prompt 后，Claude 处理前 | 给用户 prompt 附加上下文、审计请求 |
| `UserPromptExpansion` | 用户输入的命令扩展成 prompt 前 | 阻止或审计 slash command 扩展 |
| `PreToolUse` | 工具调用执行前 | 阻止危险命令、改写工具输入、强制策略 |
| `PermissionRequest` | 即将显示权限确认对话框时 | 自动批准非常窄范围的安全操作 |
| `PermissionDenied` | auto mode 分类器拒绝工具调用时 | 提示模型可换一种安全方式重试 |
| `PostToolUse` | 工具调用成功后 | 格式化、测试、日志、追加上下文 |
| `PostToolUseFailure` | 工具调用失败后 | 收集错误、给 Claude 反馈诊断信息 |
| `PostToolBatch` | 一批并行工具调用结束后 | 批量审计、汇总工具结果 |
| `Notification` | Claude Code 发送通知时 | 桌面通知、聊天通知 |
| `SubagentStart` | 创建 subagent 时 | 记录并发任务、限制派生行为 |
| `SubagentStop` | subagent 完成时 | 验证子任务结果、汇总输出 |
| `TaskCreated` | 通过 `TaskCreate` 创建任务时 | 审计任务创建、拦截不合规任务 |
| `TaskCompleted` | 任务标记完成时 | 检查完成条件、通知外部系统 |
| `Stop` | Claude 完成一次响应时 | 检查是否真的完成任务、要求继续修复 |
| `StopFailure` | turn 因 API 错误结束时 | 记录错误；输出和 exit code 不影响流程 |
| `TeammateIdle` | agent team 成员即将空闲 | 分发新任务、维持团队工作流 |
| `InstructionsLoaded` | 加载 `CLAUDE.md` 或 `.claude/rules/*.md` 时 | 审计规则加载、提醒关键约束 |
| `ConfigChange` | 会话期间配置文件变化 | 审计或阻止配置变化 |
| `CwdChanged` | 工作目录变化 | 重新加载 direnv/devbox/nix 环境 |
| `FileChanged` | 被 watch 的文件变化 | 配置文件变化后刷新环境或状态 |
| `WorktreeCreate` | 创建 worktree 时 | 自定义隔离目录创建逻辑 |
| `WorktreeRemove` | 删除 worktree 时 | 清理临时状态、归档信息 |
| `PreCompact` | 上下文压缩前 | 阻止压缩或记录压缩前状态 |
| `PostCompact` | 上下文压缩后 | 重新注入关键上下文、记录 summary |
| `Elicitation` | MCP server 请求用户输入时 | 程序化响应 MCP 表单 |
| `ElicitationResult` | MCP elicitation 结果返回 server 前 | 审计或改写用户输入结果 |
| `SessionEnd` | 会话结束时 | 清理、日志、保存统计信息 |

## 4. Hook 处理器种类

| 类型 | 配置字段 | 适用场景 | 注意事项 |
| --- | --- | --- | --- |
| `command` | `type: "command"` + `command` | 最常用。运行本地脚本、格式化、校验、审计 | 以当前系统用户权限运行，必须谨慎 |
| `prompt` | `type: "prompt"` + `prompt` | 需要模型判断，但只依赖 hook 输入 JSON | 默认轻量模型；返回 `{ "ok": true/false }` |
| `agent` | `type: "agent"` + `prompt` | 判断需要读文件、搜索代码、运行工具 | 实验性；生产建议优先用 command |
| `http` | `type: "http"` + `url` | 把事件交给本地/远端服务处理 | 通过 response body 返回 JSON 控制结果 |
| `mcp_tool` | MCP tool hook | 复用 MCP 工具生态 | 适合团队已有 MCP 工具链 |

`async: true` 只适用于 command hook。异步 hook 不阻塞 Claude，也不能阻止工具调用；它的结果会在后续 turn 作为上下文返回。

## 5. Hook 如何控制 Claude Code

### 5.1 exit code

常见 command hook 信号：

- `exit 0`：成功；如 stdout 是结构化 JSON，Claude Code 会读取控制字段。
- `exit 2`：阻止或反馈。对 `PreToolUse` 等事件，stderr 通常会返回给 Claude 作为错误/反馈，让它调整做法。
- 其他非 0：通常表示 hook 自身失败，进入 debug/错误输出路径。

不要混用“exit 2 + JSON”。官方 reference 强调：结构化 JSON 只在 exit 0 时处理。

### 5.2 结构化 JSON

常见控制字段：

- `continue: false`：停止 Claude 后续处理。
- `stopReason`：展示给用户的停止原因。
- `systemMessage`：展示给用户的警告。
- `decision` / `reason`：部分事件支持的阻止/允许语义。
- `hookSpecificOutput`：事件专属输出，例如 `PermissionRequest` 的自动批准、`PostToolUse` 的 `additionalContext`。

### 5.3 matcher 与 if

- `matcher` 用于匹配事件细分条件。例如 `PostToolUse` 中用 `Edit|Write` 匹配编辑类工具。
- `if` 可按工具名和参数进一步过滤，类似权限规则，如 `Bash(git *)`、`Edit(*.ts)`。
- `if` 只适用于工具相关事件：`PreToolUse`、`PostToolUse`、`PostToolUseFailure`、`PermissionRequest`、`PermissionDenied`。

## 6. 经典例子：阻止修改敏感文件

场景：希望 Claude 永远不能改 `.env`、`.git/`、`package-lock.json` 等文件。仅写在 `CLAUDE.md` 中属于软约束，模型可能遗忘；用 `PreToolUse` hook 可以在工具执行前强制拦截。

### 6.1 创建 Bash 脚本

`.claude/hooks/protect-files.sh`：

```bash
#!/usr/bin/env bash
set -euo pipefail

# Claude Code 会把本次工具调用的信息以 JSON 形式传到 stdin。
# 这里用 jq 取出 Claude 准备编辑或写入的文件路径。
input="$(cat)"
file_path="$(echo "$input" | jq -r '.tool_input.file_path // empty')"

protected_patterns=(".env" "package-lock.json" ".git/")

for pattern in "${protected_patterns[@]}"; do
  if [[ "$file_path" == *"$pattern"* ]]; then
    echo "Blocked: $file_path matches protected pattern '$pattern'" >&2
    exit 2
  fi
done

exit 0
```

给脚本加执行权限：

```bash
chmod +x .claude/hooks/protect-files.sh
```

这段脚本的关键点：

- `cat` 读取 Claude Code 传入的 JSON。
- `jq -r '.tool_input.file_path // empty'` 从 JSON 里取出目标文件路径。
- `protected_patterns` 是不允许修改的文件/目录模式。
- 命中敏感路径时，向 stderr 写原因，并 `exit 2`。
- `exit 2` 会让 Claude Code 阻止本次工具调用，并把原因反馈给 Claude。

如果你更熟悉 Python，也可以写成下面这样：

`.claude/hooks/protect-files.py`：

```python
#!/usr/bin/env python3
import json
import sys

data = json.load(sys.stdin)
file_path = data.get("tool_input", {}).get("file_path", "")

protected_patterns = [".env", "package-lock.json", ".git/"]

for pattern in protected_patterns:
    if pattern in file_path:
        print(
            f"Blocked: {file_path} matches protected pattern '{pattern}'",
            file=sys.stderr,
        )
        sys.exit(2)

sys.exit(0)
```

### 6.2 注册 hook

`.claude/settings.json`：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PROJECT_DIR}/.claude/hooks/protect-files.sh\""
          }
        ]
      }
    ]
 
```

如果使用的是 python code

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "python3 /path/to/hooks.py" // 如果这里很多，可以用bash脚本
          }
        ]
      }
    ]
  }
}
```

### 6.3 运行效果

当 Claude 准备调用 `Edit` 或 `Write` 修改敏感文件时：

1. `PreToolUse` 事件触发。
2. Claude Code 把工具输入 JSON 传给 Bash 脚本。
3. 脚本检查 `tool_input.file_path`。
4. 命中敏感路径时输出错误并 `exit 2`。
5. Claude Code 阻止工具调用，并把错误反馈给 Claude。
6. Claude 会尝试换一种不修改敏感文件的方案。

这个例子经典，因为它体现了 hook 最重要的价值：把“不要改敏感文件”从自然语言建议升级为运行时强制策略。

## 7. 其他常见用法

### 7.1 编辑后自动格式化

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write"
          }
        ]
      }
    ]
  }
}
```

适合 JavaScript/TypeScript 项目。这个例子假设 Linux/macOS 环境中已经安装了 `jq` 和 `npx`。

### 7.2 Claude 需要输入时通知

Linux 桌面环境可以用 `notify-send`：

```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "permission_prompt",
        "hooks": [
          {
            "type": "command",
            "command": "notify-send 'Claude Code' 'Claude Code needs your attention'"
          }
        ]
      }
    ]
  }
}
```

### 7.3 压缩后重新注入上下文

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "compact",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Reminder: use the repo test command before reporting completion.'"
          }
        ]
      }
    ]
  }
}
```

### 7.4 自动批准非常窄的权限请求

```json
{
  "hooks": {
    "PermissionRequest": [
      {
        "matcher": "ExitPlanMode",
        "hooks": [
          {
            "type": "command",
            "command": "echo '{\"hookSpecificOutput\":{\"hookEventName\":\"PermissionRequest\",\"decision\":{\"behavior\":\"allow\"}}}'"
          }
        ]
      }
    ]
  }
}
```

注意：不要把 matcher 设成空字符串或 `.*` 来自动批准所有权限请求。官方文档明确建议保持 matcher 尽可能窄。

## 8. 安全边界

hook 不是沙箱。command hook 以当前系统用户权限运行，可以读取、修改、删除当前用户能访问的文件。应遵守：

- hook 脚本必须经过代码审查。
- 输入 JSON 不可信，要验证路径、命令和参数。
- shell 变量要加引号。
- 使用绝对路径或 `${CLAUDE_PROJECT_DIR}`。
- 避免处理 `.env`、私钥、token、`.git/` 等敏感内容。
- 不要写过宽的自动批准规则。
- 异步 hook 不能用于强制策略，因为它不会阻塞触发动作。

## 9. 和 CLAUDE.md、skills、subagents 的区别

- `CLAUDE.md`：长期项目说明，影响模型行为，但不是强制执行。
- skills：给 Claude 增加专门能力、说明和可执行资源。
- subagents：隔离上下文执行专门任务。
- hooks：在生命周期节点自动执行，适合确定性规则、自动化和治理。

实践上可以组合使用：`CLAUDE.md` 写原则，skill 提供工作流，subagent 负责复杂子任务，hook 负责不可妥协的自动检查和拦截。
