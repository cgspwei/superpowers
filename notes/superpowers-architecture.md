# Superpowers 架构解析：构成与 Hook 机制

## 一、Superpowers 是什么

Superpowers 是一个零依赖的多平台插件，通过在 AI 编码 agent 会话启动时注入行为规范（skills），让 agent 遵循一套软件开发方法论执行任务。

支持的平台（harness）：Claude Code、Codex CLI、Codex App、Cursor、Factory Droid、Gemini CLI、OpenCode、GitHub Copilot CLI。

---

## 二、Superpowers 的构成

Superpowers 不只是"一堆 skills"，而是由四个关键部分组成的系统：

| 组成部分 | 作用 | 仅有 skills 能实现吗？ |
|---|---|---|
| **14 个 Skills** | 定义 AI agent 的行为规范（TDD、调试、计划等） | 能定义，但没人读 |
| **SessionStart Hook** | 在会话启动时**强制注入** `using-superpowers` | 不能 — 没有注入路径 |
| **平台检测逻辑** | 同一份脚本适配 Claude Code / Cursor / Copilot CLI / Gemini CLI | 不能 — 各平台 hook JSON 格式不同 |
| **`using-superpowers` 元技能** | 强制 agent 在做事前先检查和调用技能 | 不能 — 没有执行纪律 |

### 打个比方

> Skills 是**法律条文**，Hook 是**执法机关**，`using-superpowers` 是**执法意识**。

- 没有法律条文 → 没有规则可循
- 没有执法机关 → 法律形同虚设（AI agent 不会主动去读 skills 目录）
- 没有执法意识 → 执法机关睁一只眼闭一只眼（agent 觉得"这个太简单不需要技能"就跳过了）

### 最小闭环

```
SessionStart Hook
  → 注入 using-superpowers（元技能）
    → 强制 agent 检查是否有适用技能
      → 调用具体 skill（TDD / debugging / planning...）
        → agent 按技能规范执行任务
```

四步缺一不可。如果只是"一堆 markdown 文件放在 skills 目录里"，AI agent 根本不知道它们存在，也不会主动去读。

---

## 三、14 个 Skills 一览

| Skill | 用途 |
|---|---|
| `brainstorming` | 编码前的苏格拉底式设计推敲 |
| `writing-plans` | 把工作拆成 2-5 分钟的任务，含精确文件路径和验证步骤 |
| `executing-plans` | 批量执行，在关键节点暂停让人确认 |
| `subagent-driven-development` | 每个任务两阶段审查（规格合规 → 代码质量） |
| `test-driven-development` | 强制 RED-GREEN-REFACTOR 循环 |
| `systematic-debugging` | 4 阶段根因排查流程 |
| `verification-before-completion` | 确保修复真的生效了才能宣告完成 |
| `requesting-code-review` | 提交审查前的检查清单 |
| `receiving-code-review` | 响应审查反馈 |
| `using-git-worktrees` | 在新分支上创建隔离工作区 |
| `finishing-a-development-branch` | 合并/PR 决策流程 |
| `dispatching-parallel-agents` | 并发 subagent 工作流 |
| `writing-skills` | 创建和测试新技能的元技能（TDD 应用于流程文档） |
| `using-superpowers` | 引导/启动技能，在会话开始时自动注入 |

---

## 四、Hook 机制详解

### 4.1 声明式注册（不是安装脚本）

Superpowers 通过 `.claude-plugin/plugin.json` + `hooks/hooks.json` **声明式地**告诉 Claude Code 有哪些 hooks，不需要手动运行安装脚本。

`hooks/hooks.json` 内容：

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

这里声明了一个 **SessionStart** 类型的 hook，匹配 `startup|clear|compact` 事件，触发时执行 `run-hook.cmd session-start`。

### 4.2 执行链路

```
Claude Code 启动会话
  → 触发 SessionStart hook
    → 执行 run-hook.cmd session-start
      → 调用 bash 执行 hooks/session-start 脚本
        → 读取 skills/using-superpowers/SKILL.md
        → JSON 转义内容
        → 输出 JSON 给 Claude Code 注入为上下文
```

### 4.3 平台检测与 JSON 格式适配

`session-start` 脚本通过环境变量自动检测当前运行的 harness，输出不同格式的 JSON：

```bash
if [ -n "${CURSOR_PLUGIN_ROOT:-}" ]; then
  # Cursor → additional_context (snake_case)
  printf '{\n  "additional_context": "%s"\n}\n' "$session_context"
elif [ -n "${CLAUDE_PLUGIN_ROOT:-}" ] && [ -z "${COPILOT_CLI:-}" ]; then
  # Claude Code → hookSpecificOutput.additionalContext (nested)
  printf '{\n  "hookSpecificOutput": {\n    "hookEventName": "SessionStart",\n    "additionalContext": "%s"\n  }\n}\n' "$session_context"
else
  # Copilot CLI 等 → additionalContext (top-level)
  printf '{\n  "additionalContext": "%s"\n}\n' "$session_context"
fi
```

| 平台 | 环境变量 | JSON 格式 |
|---|---|---|
| Cursor | `CURSOR_PLUGIN_ROOT` | `{ "additional_context": "..." }` |
| Claude Code | `CLAUDE_PLUGIN_ROOT`（且无 `COPILOT_CLI`） | `{ "hookSpecificOutput": { "additionalContext": "..." } }` |
| Copilot CLI | 其他 | `{ "additionalContext": "..." }` |

### 4.4 跨平台工具映射

不同平台调用技能的工具名不同：

| 平台 | 调用技能的工具 |
|---|---|
| Claude Code | `Skill` tool |
| Copilot CLI | `skill` tool |
| Gemini CLI | `activate_skill` tool |

---

## 五、`using-superpowers` 元技能：让技能系统真正生效

### 5.1 强制技能检查

```
哪怕只有 1% 的可能某个技能适用，你也必须调用它。
如果技能适用，你没有选择，必须使用它。
这是不可协商的，不可选的，你不能自我合理化地绕过它。
```

这防止了 AI agent 跳过技能直接干活。

### 5.2 指令优先级

1. **用户明确指令**（CLAUDE.md、GEMINI.md、AGENTS.md、直接请求）— 最高优先级
2. **Superpowers 技能** — 与系统默认行为冲突时覆盖默认
3. **系统默认 prompt** — 最低优先级

用户说"不用 TDD"就真的不用。

### 5.3 反自我合理化清单

| Agent 的想法 | 现实 |
|---|---|
| "这只是个简单问题" | 问题也是任务，检查技能 |
| "我需要先了解更多上下文" | 技能检查在澄清问题之前 |
| "这不需要正式技能" | 如果有技能存在，就用它 |
| "技能杀鸡用牛刀了" | 简单的事情会变复杂，用它 |

### 5.4 决策流程

收到用户消息后：

1. 要进入计划模式？→ 先检查是否已做过头脑风暴
2. 是否有任何技能可能适用？→ 有 1% 可能就调用
3. 有检查清单？→ 为每一项创建 todo
4. 严格遵循技能执行

---

## 六、总结

**Superpowers 是一个通过 Hook 自动注入、强制执行的技能驱动系统。Skills 是它的内容核心，但 Hook + 元技能才是让它真正生效的关键。**

- 没有 Hook → skills 不会被加载
- 没有元技能 → agent 会找借口跳过技能
- 没有 skills → 没有行为规范可执行

三者缺一不可，构成了一个自举的闭环系统。

---

## 七、常见疑问：为什么不能只靠 skill 的 meta 自动触发？

### 疑问

每个 skill 的 frontmatter 都有 `name` 和 `description`，比如：

```yaml
# skills/brainstorming/SKILL.md
---
name: brainstorming
description: "You MUST use this before any creative work..."
---
```

```yaml
# skills/systematic-debugging/SKILL.md
---
name: systematic-debugging
description: Use when encountering any bug, test failure, or unexpected behavior
---
```

既然 description 已经写了触发条件，为什么不能让平台自动匹配 meta 来调用 skill，而需要 `using-superpowers` + Hook 强制注入？

### 回答

**Meta 解决的是"被发现"，而不是"被调用"和"被执行"。** 光靠 meta 自动触发有三个关键缺陷：

#### 1. Agent 不会主动去查 Skill 工具

AI agent 有 `Skill` 工具可用，但**默认不会主动调用它**——就像手机装了 App 但没人想起来打开。`using-superpowers` 通过 Hook 强制注入，解决的是"知晓 + 习惯"问题：

- **"你有 superpowers"** — 告知技能系统存在
- **"1% 可能就必须调用"** — 执行纪律
- **"用 Skill 工具访问"** — 访问方式

#### 2. Agent 会自我合理化地跳过

单个 skill 的 meta 只管自己的触发条件，但**没有哪个 skill 管"agent 整体上是否在逃避使用技能系统"**。`using-superpowers` 里的反自我合理化清单：

| Agent 的想法 | 现实 |
|---|---|
| "这不需要正式技能" | 如果有技能存在，就用它 |
| "技能杀鸡用牛刀了" | 简单的事情会变复杂，用它 |
| "我先做这一件小事" | 做任何事之前先检查 |

这是**元层面的纪律**，不是某个 skill 的 description 能覆盖的。

#### 3. Meta 不解决技能间优先级和流程编排

用户说"做一个功能"，可能同时触发 `brainstorming`、`writing-plans`、`test-driven-development`。哪个先？`using-superpowers` 定义了明确优先级：先 process skills，再 implementation skills。流程纪律需要**元技能**来编排。

### 对比总结

| 维度 | 只靠 skill meta | `using-superpowers` + Hook |
|---|---|---|
| agent 知道技能存在吗？ | 不知道，除非主动查 Skill 工具 | Hook 强制注入，第一轮就知道 |
| agent 会偷懒跳过吗？ | 很容易自我合理化 | "1% 可能就必须调用" + 反合理化清单 |
| 技能间优先级谁管？ | 没人管，各自为政 | 元技能定义 process > implementation |
| 跨平台一致性能保证吗？ | 各平台匹配算法不同，行为不可控 | 统一注入同一份规范 |

---

## 八、关于"为什么不全加载所有 skill"的纠正

### 错误论证（已纠正）

> ~~14 个 skill 的完整内容加起来超过 100KB，全加载太重了~~

这个论证是**错误的**。如果只加载 meta（name + description），14 个 skill 的 frontmatter 加起来仅 1-2KB，根本不是负担。实际上 Gemini CLI 就是在会话启动时自动加载所有 skill 的 meta：

> **In Gemini CLI:** Skills activate via the `activate_skill` tool. **Gemini loads skill metadata at session start** and activates the full content on demand.

### 正确区分：Meta vs 完整内容

| 层面 | 体积 | 加载时机 | 作用 |
|---|---|---|---|
| **Meta**（name + description） | 极轻，1-2KB | 平台注册表自动维护，Gemini 启动时就加载 | 让 agent 发现和选择 skill |
| **完整内容**（SKILL.md 全文） | 每个 2-8KB | agent 选定 skill 后按需加载 | 指导 agent 具体行为 |

### Hook 为什么只注入 `using-superpowers` 而不把 14 个 meta 也注入？

**因为不需要**——平台的 `Skill` 工具已经能展示所有 skill 的 meta，agent 调用一下就能看到列表。Hook 注入 `using-superpowers` 要解决的不是"agent 看不到 skill 列表"的问题，而是**agent 压根不会想到要去调用 `Skill` 工具**。

### 完整调用链路

```
会话启动
  → Hook 强制注入 using-superpowers（唯一预加载完整内容的 skill）
    → agent 知道："我有 superpowers，要用 Skill 工具访问其他技能"

用户发消息："修个 bug"
  → agent 想到："1% 可能适用，必须检查"（using-superpowers 的纪律）
    → agent 调用 Skill 工具，列出可用技能
      → 平台返回所有 skill 的 name + description（meta 在这里起作用！）
        → agent 看到 systematic-debugging: "Use when encountering any bug"
          → agent 选择调用 systematic-debugging
            → 平台加载该 skill 的完整 SKILL.md 内容
              → agent 按规范执行
```

### 修正后的类比

> - 各 skill 的 meta = App Store 里的应用列表（名称 + 简介，一直都在，很轻量）
> - `using-superpowers` = 一个弹窗通知："嘿，你有这些 App，遇到合适场景必须打开"
> - skill 的完整内容 = App 本体（选中了才下载安装）
> - 没有弹窗通知，App 就静静地躺在商店里，没人想起来去搜
