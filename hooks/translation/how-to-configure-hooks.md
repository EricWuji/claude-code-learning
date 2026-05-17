# 中文精读翻译：Claude Code Power User Customization - How to Configure Hooks

原文：https://claude.com/blog/how-to-configure-hooks

说明：本文是 Anthropic blog 的中文精读翻译与结构化改写，不是逐字全文搬运。该博文发布日期为 2025-12-11，文中提到“8 种 hook types”的说法反映当时写作背景；截至本次整理读取的官方 reference，事件种类已经更多，应以 reference 为准。

## 文章主旨

即使 Claude Code 工作流已经很顺滑，日常使用中也会积累摩擦：

- Claude 每次写文件后，你还要手动跑格式化。
- Claude 每次运行常用命令，都弹同样的权限确认。
- 每个会话开始，你都要重复粘贴项目背景。

hooks 的作用就是消除这些重复摩擦。它们像触发器一样，在 Claude Code 的关键操作前后执行你配置的逻辑、脚本或命令。

## hook 的定位

hook 是你定义的自动命令，会在 Claude Code 会话中特定事件发生时运行。它可以用于：

- 在操作执行前拦截。
- 注入 agent 需要的上下文。
- 自动化重复批准。
- 在危险操作发生前阻止。
- 把团队规则变成程序化治理。

## 为什么 hook 重要

文章强调的核心价值是“确定性”。自然语言指令容易被模型遗忘、误解或在长上下文中被稀释；hook 是运行时机制，只要事件发生、matcher 命中，它就会执行。

这使 hook 特别适合以下规则：

- 必须执行：如格式化、记录日志。
- 必须禁止：如不能修改生产配置或密钥。
- 必须检查：如停止前必须测试通过。
- 必须提醒：如等待用户输入时通知。

## 配置思路

配置 hook 时先回答三个问题：

1. 要拦截哪个生命周期事件？
2. 只针对哪些工具、参数或通知类型？
3. handler 要做什么，以及失败时如何反馈？

对应到配置字段：

- event：`hooks` 对象下的 key，例如 `PreToolUse`。
- matcher：缩小触发范围，例如 `Bash`、`Edit|Write`、`permission_prompt`。
- handler：真正执行的 `command`、`prompt`、`agent`、`http` 等。

## 典型场景

### 自动化格式化

文件写入后运行格式化工具，是最常见的 hook。它适合放在 `PostToolUse`，因为格式化发生在 Claude 修改文件之后。

价值：

- 保持仓库风格一致。
- 减少人工提醒。
- 让 Claude 后续看到格式化后的真实文件。

### 权限与安全策略

危险文件、危险命令、生产路径，适合放在 `PreToolUse`。在工具调用前阻止，比事后审计更可靠。

示例策略：

- 禁止修改 `.env`。
- 禁止运行部署命令。
- 禁止直接执行数据库迁移。
- 要求测试通过统一脚本运行。

### 动态上下文注入

有些上下文不适合静态写进 `CLAUDE.md`：

- 最近提交。
- 当前分支。
- 当前 sprint。
- 外部系统状态。
- 目录变化后的环境变量。

这类信息可以通过 hook 动态输出，作为补充上下文提供给 Claude。

### 通知与协作

长时间任务中，Claude 常会等待用户授权或下一步输入。`Notification` hook 可以把这些等待状态接入桌面通知、Slack、Teams 或其他系统。

## 调试建议

文章的调试思路可以概括为：

- 用 `/hooks` 确认配置是否加载。
- 先让命令在终端独立运行成功。
- 检查 matcher 是否命中。
- 检查 JSON 是否有效。
- 对返回 JSON 的 hook，保证 stdout 干净。
- 出问题时打开 debug/verbose 输出。

## 与官方 reference 的关系

这篇 blog 更像“为什么用、怎么思考”的入门文章；reference 是精确语法和事件字段的准绳。实际落地时建议：

1. 从 blog 理解 hooks 可以解决哪些摩擦。
2. 从 guide 复制最接近的配置模式。
3. 从 reference 查事件输入输出、exit code 和安全边界。

## 读后实践建议

优先从低风险、高收益的 hook 开始：

- `Notification`：只通知，不改状态。
- `PostToolUse` 格式化：动作简单、可回滚。
- `PreToolUse` 阻止敏感文件：规则明确、价值高。

谨慎使用：

- 宽 matcher 的自动批准。
- 会修改大量文件的后台脚本。
- 未审查的远程 HTTP hook。
- 需要访问密钥或私有数据的 hook。

