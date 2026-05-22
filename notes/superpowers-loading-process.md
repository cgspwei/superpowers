# Superpowers 启动加载全过程

本文档详细描述安装 superpowers 后，Claude Code 每次启动会话时如何加载 superpowers skill 的完整链路。

## 全局流程概览

```
Claude Code 启动会话
  → 1. 扫描已启用的 Plugin，读取各自的 hooks.json
  → 2. 触发 SessionStart 事件（匹配 startup）
  → 3. 执行 run-hook.cmd session-start
  → 4. run-hook.cmd 调用 session-start 脚本
  → 5. session-start 读取 SKILL.md，JSON 转义，输出上下文注入
  → 6. Claude Code 将上下文注入到 Agent
  → Agent 看到："你有 superpowers，1% 可能就必须调用技能"
```

---

## 第一步：Claude Code 扫描 Plugin 目录下的 hooks.json

Claude Code 启动时，会读取用户配置（`~/.claude/settings.json` 或 `.claude/settings.json`），找到所有已启用的 Plugin。对于每个已启用的 Plugin，Claude Code 会扫描其目录下的 `hooks/hooks.json`，将其中的 hooks 声明合并到全局 hooks 配置中。

**关键设计：每个 Plugin 维护自己的 `hooks.json`，不需要修改全局配置文件。** 卸载 Plugin 时只需移除启用记录，hooks 声明自动消失。

---

## 第二步：hooks.json 的详细逻辑

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|clear|compact",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" session-start",
            "async": false
          }
        ]
      }
    ]
  }
}
```

### 逐字段解析

| 字段 | 值 | 含义 |
|------|----|------|
| `SessionStart` | — | Claude Code 定义的标准 hook 事件名，在会话启动时触发 |
| `matcher` | `"startup\|clear\|compact"` | 进一步细化触发条件，匹配三种场景 |
| `type` | `"command"` | hook 的执行方式：运行 shell 命令 |
| `command` | `"...run-hook.cmd\" session-start` | 具体执行的命令 |
| `async` | `false` | 同步执行，必须等 hook 跑完 Agent 才能开始工作 |

### matcher 的三种触发场景

- **`startup`**：Claude Code 首次启动新会话
- **`clear`**：用户执行 `/clear` 清空对话
- **`compact`**：上下文溢出时 Claude Code 自动压缩对话

这三种场景的共同点是：**Agent 的上下文被重置了**，需要重新注入 superpowers 的使用指南，否则 Agent 会"忘记"自己有 superpowers。

### 为什么 async 必须是 false

如果异步执行，Agent 可能在 superpowers 上下文注入之前就开始回复用户，导致第一条消息不知道自己有 superpowers。同步执行确保了 Agent 看到完整上下文后才开工。

### 为什么命令是 run-hook.cmd 而不是直接调用 session-start

`run-hook.cmd` 是一个跨平台 polyglot 脚本（同一个文件既是 Windows batch 又是 Unix shell script），负责找到 bash 并执行实际的 hook 脚本。直接调用 `session-start` 在 Windows 上无法运行，因为 Claude Code 的 Windows 自动检测会对含 `.sh` 后缀的命令自动加 `bash` 前缀，而 `session-start` 故意没有后缀来绕过这个问题。

---

## 第三步：session-start 脚本的详细逻辑

```bash
#!/usr/bin/env bash
set -euo pipefail
```

`set -euo pipefail` 确保脚本在任何命令失败、未定义变量或管道错误时立即退出，不会静默地继续执行。

### 3.1 确定 Plugin 根目录

```bash
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
PLUGIN_ROOT="$(cd "${SCRIPT_DIR}/.." && pwd)"
```

通过脚本自身路径向上回溯一级，找到 superpowers 的安装根目录。这样无论 Plugin 安装在哪个路径下都能正确定位文件。

### 3.2 旧版迁移警告

```bash
legacy_skills_dir="${HOME}/.config/superpowers/skills"
if [ -d "$legacy_skills_dir" ]; then
    warning_message="⚠️ WARNING: Superpowers now uses Claude Code's skills system..."
fi
```

检查 `~/.config/superpowers/skills` 是否存在（旧版自定义 skills 目录）。如果存在，生成一条警告注入到上下文中，提醒用户迁移到 `~/.claude/skills`。这是一种渐进迁移策略——不删除旧文件，但通过持续提醒推动用户主动迁移。

### 3.3 读取 using-superpowers skill 内容

```bash
using_superpowers_content=$(cat "${PLUGIN_ROOT}/skills/using-superpowers/SKILL.md" 2>&1 || echo "Error reading using-superpowers skill")
```

读取 `using-superpowers` skill 的完整内容。这个 skill 是 superpowers 的"入口指南"，告诉 Agent 如何发现和使用其他 13 个 skill。如果读取失败，输出错误信息而不是让脚本崩溃。

### 3.4 JSON 转义

```bash
escape_for_json() {
    local s="$1"
    s="${s//\\/\\\\}"      # \ → \\
    s="${s//\"/\\\"}"      # " → \"
    s="${s//$'\n'/\\n}"    # 换行 → \n
    s="${s//$'\r'/\\r}"    # 回车 → \r
    s="${s//$'\t'/\\t}"    # Tab → \t
    printf '%s' "$s"
}
```

因为最终要输出 JSON 字符串，SKILL.md 的内容必须转义。这里用 bash 参数替换（`${s//old/new}`）逐类替换，比逐字符循环快得多（注释里特别提到了性能）。

### 3.5 构建上下文字符串

```bash
session_context="<EXTREMELY_IMPORTANT>\nYou have superpowers.\n\n
**Below is the full content of your 'superpowers:using-superpowers' skill - 
your introduction to using skills. For all other skills, use the 'Skill' tool:**\n\n
${using_superpowers_escaped}\n\n${warning_escaped}\n</EXTREMELY_IMPORTANT>"
```

用 `<EXTREMELY_IMPORTANT>` 标签包裹内容——这是给 Agent 的强信号，确保它不会忽略这段上下文。核心信息是：

1. "你有 superpowers"
2. 完整的 `using-superpowers` skill 内容（包含"1% 可能就必须调用技能"的强制规则）
3. 旧版迁移警告（如果有的话）

### 3.6 多平台适配输出

```bash
if [ -n "${CURSOR_PLUGIN_ROOT:-}" ]; then
  # Cursor：additional_context（顶层，snake_case）
  printf '{\n  "additional_context": "%s"\n}\n' "$session_context"
elif [ -n "${CLAUDE_PLUGIN_ROOT:-}" ] && [ -z "${COPILOT_CLI:-}" ]; then
  # Claude Code：hookSpecificOutput.additionalContext（嵌套）
  printf '{\n  "hookSpecificOutput": {\n    "hookEventName": "SessionStart",\n    "additionalContext": "%s"\n  }\n}\n' "$session_context"
else
  # Copilot CLI 或其他：additionalContext（顶层，SDK 标准）
  printf '{\n  "additionalContext": "%s"\n}\n' "$session_context"
fi
```

三种平台使用不同的 JSON 输出格式：

| 平台 | 判断条件 | JSON 格式 |
|------|---------|-----------|
| Cursor | `CURSOR_PLUGIN_ROOT` 存在 | `additional_context`（顶层，snake_case） |
| Claude Code | `CLAUDE_PLUGIN_ROOT` 存在且非 Copilot | `hookSpecificOutput.additionalContext`（嵌套） |
| Copilot CLI / 其他 | 其他情况 | `additionalContext`（顶层，camelCase） |

**为什么 Claude Code 必须用嵌套格式？** 因为 Claude Code 会同时读取 `additional_context` 和 `hookSpecificOutput` 两个字段且不做去重，如果两个都输出会导致上下文重复注入。所以必须只输出当前平台消费的那个字段。

### 3.7 跨平台入口脚本：run-hook.cmd

`hooks.json` 中的 command 指向 `run-hook.cmd`，这是一个巧妙的双格式文件：

- **在 Windows 上**：cmd.exe 执行 `@echo off` 开头的 batch 部分，找到 Git Bash 并调用 hook 脚本
- **在 Unix 上**：bash 把 `: << 'CMDBLOCK'` 到 `CMDBLOCK` 之间的内容当作 heredoc 忽略，直接执行底部的 Unix 逻辑

hook 脚本故意不加 `.sh` 后缀（如 `session-start` 而不是 `session-start.sh`），是因为 Claude Code 的 Windows 自动检测会对含 `.sh` 的命令自动加 `bash` 前缀，可能造成双重 bash 调用。

---

## 最终效果

整个链路跑完后，Agent 的上下文中多了这段内容：

```
<EXTREMELY_IMPORTANT>
You have superpowers.

**Below is the full content of your 'superpowers:using-superpowers' skill - 
your introduction to using skills. For all other skills, use the 'Skill' tool:**

[using-superpowers SKILL.md 的完整内容]
</EXTREMELY_IMPORTANT>
```

此后 Agent 知道：
1. 自己有 superpowers，有 1% 可能就必须调用 skill
2. 使用 `Skill` 工具按需加载其他 skill（如 brainstorming、tdd 等）
3. 其他 13 个 skill 不会全部预加载，而是 Agent 根据任务需要按需调用

**这是一种"懒加载"设计：** 只在启动时注入入口指南，其他 skill 按需加载，避免一次性把 14 个 SKILL.md 全部塞进上下文窗口。
