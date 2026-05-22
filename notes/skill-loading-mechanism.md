# Superpowers 的 Skill 加载机制：为什么不同于普通 Skills

## 核心差异

Superpowers 的 skill 加载机制和 Claude Code 标准 skill 加载机制有根本区别。这不是实现细节的差异，而是**设计哲学的差异**。

## 普通 Skill 的加载方式

标准做法是把 skill 放在 `~/.claude/skills/` 或 `.claude/skills/` 下：

```
~/.claude/skills/
  my-skill/
    SKILL.md
```

SKILL.md 的 frontmatter 描述何时该用这个 skill：

```yaml
---
description: "Run project linter"
when_to_use: "After writing JavaScript code"
---
```

Claude Code 自动扫描发现这些 skill，Agent 根据 `description` 和 `when_to_use` **自行判断**是否调用。整个流程是：

```
Agent 看到任务 → 自己判断是否需要 skill → 决定调用或跳过
```

**这是一种"建议"模式**——skill 告诉 Agent "你可以用我"，但用不用由 Agent 决定。

## Superpowers 的加载方式

Superpowers 用了一条完全不同的路径：

```
hooks.json → SessionStart hook → session-start 脚本 → 读取 using-superpowers SKILL.md → JSON 转义 → additionalContext 注入
```

在会话启动时，**强制**把 `using-superpowers` 的完整内容注入到 Agent 上下文中，不依赖 Agent 的判断。

**这是一种"命令"模式**——不是告诉 Agent "你可以用我"，而是告诉 Agent "你必须用我"。

## 为什么不能用普通方式？

### 原因一：鸡生蛋问题

`using-superpowers` 的核心规则是：

> If you think there is even a 1% chance a skill might apply to what you are doing, you ABSOLUTELY MUST invoke the skill.

这段话把 Agent 调用 skill 的决策阈值从"我觉得需要才用"改成了"1% 可能就必须用"。

如果 `using-superpowers` 自己也用普通 skill 的方式加载，就变成了循环依赖：

```
Agent 需要先决定"我应该调用 using-superpowers"
  → 但它怎么知道自己应该调用？
  → 因为它还不知道自己有 superpowers
  → 所以它不会调用 using-superpowers
  → 所以它不知道自己有 superpowers
  → ...
```

**用 hook 强制注入，就是为了打破这个循环。** 不依赖 Agent 的判断，而是在它"出生"的那一刻就告诉它"你有 superpowers"。

### 原因二：决策阈值问题

即使 Agent 能发现普通 skill，它的默认行为也是"够简单就不调用"。

| 场景 | 普通 skill（建议模式） | Superpowers（命令模式） |
|------|----------------------|----------------------|
| 用户："帮我加个登录页面" | Agent 判断"不算复杂"，跳过 brainstorming 直接写代码 | Agent 想"涉及认证、安全性，有 1% 可能该用 brainstorming"，调用 skill |
| 用户："修个 typo" | Agent 判断"太简单，不需要 skill"，直接改 | Agent 判断"确实不需要 skill"，直接改（1% 规则允许合理跳过） |
| 用户："重构整个认证模块" | Agent 判断"复杂，可能需要 skill"，调用 | Agent 判断"复杂，必须用 skill"，调用 |

区别在于：普通 skill 的调用是 Agent 的**选项**，superpowers skill 的调用是 Agent 的**义务**。选项可以跳过，义务不能。

### 原因三：一致性保障

普通 skill 的调用频率高度依赖 Agent 当次的表现——同一个任务，不同对话可能走不同的路。Superpowers 的 hook 注入确保了**每次会话的起点一致**——Agent 永远知道自己有 superpowers，不会因为"忘了"或"觉得不需要"而跳过。

## 两种机制对比

| 维度 | 普通 Skill | Superpowers Skill |
|------|-----------|-------------------|
| 加载方式 | Claude Code 自动扫描 skills 目录 | Hook 强制注入到上下文 |
| 加载时机 | Agent 决定调用时才加载完整内容 | 会话启动时就在上下文中 |
| 依赖 Agent 判断 | 是——Agent 自己决定是否调用 | 否——`using-superpowers` 无条件注入 |
| 调用意愿 | 弱——"建议使用" | 强——"1% 可能就必须" |
| Agent 可跳过 | 可以，且经常跳过 | 很难合理化跳过 |
| 一致性 | 低——不同对话可能走不同路 | 高——每次会话起点一致 |
| Token 开销 | 低——按需加载 | 有开销——每次会话都注入入口指南 |
| 适用场景 | 简单的工具性 skill | 需要改变 Agent 行为模式的 skill |

## 本质：从"被动发现"到"主动注入"

普通 skill 的哲学是**被动发现**——skill 在那里等着，Agent 需要时来找。

Superpowers 的哲学是**主动注入**——不等 Agent 来找，而是先把"你有超能力"这个认知植入到 Agent 的"潜意识"中（system prompt 级别的上下文），然后由这个认知驱动 Agent 主动调用其他 skill。

这也解释了为什么只有 `using-superpowers` 需要强制注入，而 brainstorming、tdd 等其他 13 个 skill 仍然用普通方式按需加载——因为那些 skill 改变的是"怎么做"（方法），而 `using-superpowers` 改变的是"会不会去做"（意愿）。方法可以按需学习，意愿必须提前植入。
