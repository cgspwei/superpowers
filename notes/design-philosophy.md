# Superpowers 设计思想深度分析

## 一句话概括

Superpowers 不是工具，是**嵌入 AI 编程 Agent 大脑的方法论操作系统**。

---

## 核心问题：为什么 AI Agent 需要 Superpowers？

AI 编程 Agent 有一个天然缺陷：**它们倾向于走捷径**。面对一个编程请求，Agent 的默认行为是立刻开始写代码——跳过设计、跳过测试、跳过验证。这在简单任务上能工作，但在复杂项目中会导致大量返工。

Superpowers 的核心洞察是：**Agent 的问题不是能力不足，而是缺乏约束和流程**。它不是给 Agent 更多能力，而是给 Agent 更多纪律。

---

## 设计思想一：把流程文档当代码对待

Superpowers 最反直觉的设计决策是：**每个 skill 文件（SKILL.md）不是文档，是行为塑造代码**。

具体体现在 `writing-skills` skill 中：

> Writing skills IS Test-Driven Development applied to process documentation.

技能的创作过程完全遵循 TDD 的 RED-GREEN-REFACTOR 循环：
- **RED**：在没有 skill 的情况下让 Agent 执行任务，记录它的确切失败行为和"合理化借口"
- **GREEN**：写最小化的 skill，专门针对那些具体的违规行为
- **REFACTOR**：发现新的漏洞，填补，再测试，直到无懈可击

这意味着 skill 中的每一句话都经过了实验验证。一词之差会改变 Agent 的行为。`CLAUDE.md` 明确写道：

> Skills are behavior-shaping content, not prose — word-level changes affect agent behavior; don't rewrite without evals

---

## 设计思想二："Iron Law" 模式——用绝对禁令抵抗合理化

Superpowers 反复使用一种特定的写作模式，可以称为"**铁律模式**"：

```
NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST
NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST  
NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE
```

这不是风格选择，是**心理工程**。AI Agent 在压力下会生成听起来合理的理由来绕过规则。Superpowers 用三种机制对抗这一点：

**1. 明确封堵每个已知漏洞**

TDD skill 中，仅仅说"先写测试"是不够的，还要明确禁止所有变体：
```
Write code before test? Delete it. Start over.
No exceptions:
- Don't keep it as "reference"
- Don't "adapt" it while writing tests
- Don't look at it
- Delete means delete
```

**2. "违反字面即违反精神"原则**

每个严格 skill 都包含：
> Violating the letter of the rules is violating the spirit of the rules.

这句话专门切断了"我理解精神，所以可以灵活变通"这类整个类别的合理化。

**3. 合理化借口对照表**

每个 skill 都包含一个 "Common Rationalizations" 表格，把 Agent 在压力下会生成的具体借口一一列出，并附上反驳。这是从 baseline 测试中收集的真实失败案例，不是凭空想象的。

---

## 设计思想三：线性流程 + 硬性关卡

整个开发工作流是一个严格的线性管道：

```
brainstorming → writing-plans → subagent-driven-development
                                    ↓ (每个任务)
                              实现 → spec review → quality review
                                    ↓ (全部完成)
                              finishing-a-development-branch
```

关键设计：**每个阶段有硬性关卡（HARD-GATE）**，不允许跳过。

`brainstorming` skill 明确：
```
HARD-GATE: Do NOT invoke any implementation skill, write any code, scaffold any project, 
or take any implementation action until you have presented a design and the user has approved it.
This applies to EVERY project regardless of perceived simplicity.
```

这个设计的意图是：**把 Agent 最危险的倾向（立刻写代码）变成一个被明确禁止的行为**，而不是一个被建议避免的行为。"建议"在压力下会被忽略，"禁止"则更难绕过。

---

## 设计思想四：Subagent 架构解决上下文污染问题

`subagent-driven-development` 的核心洞察是：**长对话上下文是质量的敌人**。

随着对话进行，Agent 会：
- 忘记早期的约束
- 被前面任务的实现细节污染判断
- 失去对整体计划的关注

解决方案：**每个任务派发一个全新的 subagent，主 Agent 只负责协调**。

主 Agent 的职责：
- 从计划文件中提取任务（只读一次）
- 为每个 subagent 精确构造它需要的上下文
- 在 subagent 完成后运行两阶段审查

两阶段审查的顺序有意义：**先审查 spec 合规（是否多做/少做），再审查代码质量**。顺序颠倒会导致对一个不满足需求的高质量实现浪费审查精力。

模型选择也被明确规定：
- 机械性实现任务（1-2个文件，规格清晰）→ 便宜快速的模型
- 集成和判断任务 → 标准模型  
- 架构和审查任务 → 最强的可用模型

这是**成本意识的架构设计**，不是让 Agent 无限制地使用最强模型。

---

## 设计思想五：1% 规则——宽松触发，严格执行

`using-superpowers` skill 规定：

> If you think there is even a 1% chance a skill might apply to what you are doing, you ABSOLUTELY MUST invoke the skill.

这是一个刻意的不对称设计。触发门槛极低（1%），但触发后的执行要求极高（必须严格遵循）。

原因在于 Agent 的认知偏差：它们会系统性地低估技能适用的概率（"这只是个简单问题"），但一旦正确加载了 skill，遵循它的概率会大幅上升。

"Red Flags" 表格列出了所有常见的低估模式：
| 想法 | 现实 |
|------|------|
| "这只是个简单问题" | 问题也是任务，检查 skill |
| "我先需要更多上下文" | Skill 检查在提问之前 |
| "这个 skill 太重了" | 简单的事会变复杂，用它 |

---

## 设计思想六：零依赖作为核心约束

Superpowers 坚持零外部依赖（no npm packages, no external services），这不只是工程决策，是**哲学约束**：

- **普适性**：技能必须在所有 harness（Claude Code、Codex、Cursor、Gemini CLI 等）上工作
- **可维护性**：没有依赖意味着没有版本冲突、没有供应链风险
- **可审计性**：纯 markdown 文件，任何人都可以完整阅读和理解系统的全部内容

这个约束倒逼了一个优雅的架构：所有"功能"都通过语言表达，而不是通过代码执行。

---

## 设计思想七：CSO（Claude Search Optimization）

这是 Superpowers 中最独特的工程洞察之一。

Skill 的 `description` 字段被 Agent 用来决定"是否需要加载这个 skill"。如果描述包含了 skill 的执行流程摘要，Agent 会直接按描述行事，**跳过阅读 skill 全文**。

这在 `subagent-driven-development` 中发现了一个真实 bug：描述写了"code review between tasks"，Agent 就只做了一次 review。描述改成纯触发条件后，Agent 才正确读取 skill 内容，执行两阶段审查。

由此得出规则：
- ❌ 描述中不能包含工作流摘要
- ✅ 描述只包含触发条件（"Use when..."）
- ✅ 包含症状词汇（"race condition"、"flaky"、"blocked"）
- ✅ 使用 Agent 会搜索的关键词

这是**为 AI 阅读习惯优化文档**，类似于 SEO，但目标是让 AI 正确决策何时加载何种信息。

---

## 设计思想八："Human Partner" 不只是措辞

Superpowers 坚持使用"your human partner"而不是"the user"。这不是风格选择，而是语义工程：

- "user" 暗示服务关系——Agent 执行指令
- "human partner" 暗示协作关系——双方共同决策

这个术语影响了 Agent 的行为模式：
- 遇到阻塞时，主动向 human partner 寻求架构层面的帮助（而不是继续尝试技术 fix）
- 在关键决策点暂停，而不是假设自己能猜到正确答案
- 当发现问题时，提出"我们应该重新考虑架构"，而不是"我来修复它"

`systematic-debugging` skill 明确写道：当 3+ 次修复尝试都失败，**必须停下来和 human partner 讨论架构问题**，不允许第 4 次尝试。

---

## 设计思想总结

Superpowers 的核心哲学可以用一句话概括：

**把软件工程中最重要的人类纪律（TDD、系统调试、验证驱动）转化为 AI Agent 无法合理化绕过的强制约束。**

它的方法论层次：
1. **识别 AI Agent 的失败模式**（通过 baseline 测试收集真实案例）
2. **将正确行为形式化为带强制语气的文本**（Iron Laws, Hard Gates）
3. **用 TDD 方法测试和迭代文本本身**（skills 是被测试过的代码）
4. **通过 session hook 在每次会话开始时注入**（自动触发，无需用户干预）

最终结果：一个不依赖 Agent 记忆、不依赖用户提醒、在每次新会话中自动生效的开发方法论系统。
