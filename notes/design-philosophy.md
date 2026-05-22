# Superpowers 设计思想深度分析

## Superpowers 是什么

Superpowers（[github.com/obra/superpowers](https://github.com/obra/superpowers)）是 Jesse Vincent 创建的开源项目，目前 GitHub 202k+ star，是 AI 编程 Agent 领域影响力最大的方法论项目之一。它是一个跨平台的方法论 Plugin，支持 Claude Code、Codex CLI、GitHub Copilot CLI、Gemini CLI、Cursor、OpenCode、Factory Droid 等多个平台。安装后为 Agent 注入一套系统化的工作方法论。它包含 14 个 skill（技能），覆盖从需求设计到代码实现的完整开发流程：

Superpowers 的本质不是给 Agent 更多能力，而是给 Agent 更多纪律。它强制约束 Agent 的开发流程——必须先设计再写代码、必须先写测试再实现、必须验证通过才能声称完成，不允许在任何关键节点偷工减料。

用一句话概括：**Superpowers 不是工具，是嵌入 AI 编程 Agent 大脑的方法论操作系统。**

---

## 为什么 AI 编程 Agent 需要 Superpowers？

AI 编程 Agent 的行为模式经历了一个清晰的演进：

**2023-2024 年：裸奔时代。** 早期 Agent（GPT-4 + 简单 prompt、早期 Cursor 等）确实会一上来就写代码。面对一个编程请求，默认行为是立刻产出实现——跳过设计、跳过测试、跳过验证。这在简单任务上能工作，但在复杂项目中导致大量返工。不是 Agent 不愿意想，而是当时的 system prompt 工程还不成熟，没有引导 Agent "先理解再动手"。

**2025-2026 年：软引导时代。** 主流 Agent（Claude Code、Cursor、Copilot Workspace、Windsurf 等）已经在 system prompt 层面做了"先理解再动手"的引导，不会再裸奔式地直接写代码。各家的 harness 提供了**软引导**——"先理解需求""遇到不确定的地方问用户""分步实施"。简单问题上各家 Agent 表现都不错。

**但软引导的问题是：设计质量没有下限保障。** 复杂项目中，Agent 可能想了两步就觉得"够了，开始写吧"——没有机制确保它覆盖了所有维度、考虑了多个方案、检查了内部一致性。软引导像安全驾驶提示（"请注意路况"），提醒有效但无法强制执行。

**Superpowers 代表的是：硬约束时代。** 它的核心洞察是：**Agent 的问题不是不会想，而是想的下限没有保底**。Superpowers 像闯红灯摄像头（闯了就拦住你），把"建议你先设计"变成"你必须先设计，否则不准动代码"。它不是给 Agent 更多能力，而是给 Agent 更多纪律——在各家 harness 软引导的基础上，补上下限保障。

三个时代的演进本质：**从"不引导"到"软引导"到"硬约束"**，每一步都在提高 Agent 行为质量的下限。

---

## Superpowers 里有什么：14 个 Skill 的三层架构

14 个 skill 不是平铺的列表，而是按角色分为三层：

### 入口层（1 个 skill）

| Skill | 作用 |
|-------|------|
| **using-superpowers** | 会话启动时通过 hook 强制注入，建立"1% 可能就必须调用 skill"的认知 |

这个 skill 是整个系统的心脏。它不在线性工作流中占据某一步，而是作为一个**元认知植入**——让 Agent 从第一条消息就知道自己有 superpowers，并且在任何决策点都优先考虑调用 skill。它必须用 hook 强制注入，而不是像其他 skill 一样放在 skills 目录等 Agent 自己发现，因为如果 Agent 不知道自己有 superpowers，它就不会主动调用任何 skill（鸡生蛋问题）。

### 工作流层（8 个 skill）

这些 skill 构成一条严格的线性管道，每一步都有 HARD-GATE，不允许跳过：

| Skill | 作用 |
|-------|------|
| **brainstorming** | 需求澄清与方案设计，强制提出 2-3 个方案对比，HARD-GATE：禁止在用户批准设计前写任何代码 |
| **using-git-worktrees** | 创建隔离的工作空间，确保新功能开发不影响主分支 |
| **writing-plans** | 将设计转化为可执行的任务计划，每个任务 2-5 分钟粒度 |
| **subagent-driven-development** | 按计划分派子 Agent 执行（支持 subagent 的平台），主 Agent 负责协调和两阶段审查 |
| **executing-plans** | 按计划顺序执行（不支持 subagent 的平台），在关键节点设置 review checkpoint |
| **test-driven-development** | 测试驱动开发，红-绿-重构循环，Iron Law：没有失败测试就不能写生产代码 |
| **requesting-code-review** | 发起 code review，派发审查 subagent |
| **receiving-code-review** | 接收并处理 review 反馈，要求技术验证而非盲目接受 |
| **finishing-a-development-branch** | 分支收尾，验证测试通过，决定 merge/PR/cleanup |

### 保障层（5 个 skill）

这些 skill 不在线性流程中固定出现，但在特定场景下必须调用：

| Skill | 触发条件 | 作用 |
|-------|---------|------|
| **systematic-debugging** | 遇到 bug、测试失败、异常行为 | 系统化调试，Iron Law：没有根因分析就不能修 bug |
| **verification-before-completion** | 即将声称工作完成 | 强制运行验证命令，Iron Law：没有验证证据就不能声称完成 |
| **dispatching-parallel-agents** | 面对多个独立问题 | 并行分派 Agent，每个 Agent 处理一个独立问题域 |
| **writing-skills** | 需要创建或修改 skill | 用 TDD 方法写 skill：先测基线行为，再写 skill，再验证合规 |
| **using-superpowers** | (同入口层) | 也作为保障层的一部分，持续提醒 Agent 调用 skill |

---

## 7 步工作流

三层架构中的工作流层 skill 串成了一条 7 步开发流水线：

```
Step 1        Step 2           Step 3          Step 4              Step 5      Step 6              Step 7
┌──────────┐ ┌──────────────┐ ┌────────────┐ ┌────────────────┐ ┌─────────┐ ┌──────────────────┐ ┌─────────────────────┐
│Brainstorm│→│ Git Worktrees │→│Write Plans │→│ Subagent/Exec  │→│   TDD   │→│ Code Review +    │→│ Finish Branch       │
│          │ │              │ │            │ │                │ │         │ │ Verification     │ │                     │
│需求澄清   │ │隔离工作空间    │ │任务计划     │ │按计划执行       │ │测试驱动  │ │审查 + 验证        │ │分支收尾              │
│方案设计   │ │              │ │            │ │                │ │         │ │                  │ │                     │
└──────────┘ └──────────────┘ └────────────┘ └────────────────┘ └─────────┘ └──────────────────┘ └─────────────────────┘
```

### Step 1：Brainstorming — 需求澄清与方案设计

Agent 不能直接开始写代码。必须先通过协作对话理解需求，提出 2-3 个方案并对比 trade-off，最终呈现设计并获得用户批准。

**HARD-GATE：** 禁止在用户批准设计前执行任何实现动作——包括写代码、搭脚手架、调用实现 skill。不管项目看起来多简单。

### Step 2：Git Worktrees — 创建隔离工作空间

为每个功能分支创建独立的 git worktree，确保新功能开发不会影响主工作区。优先使用平台原生工具（Claude Code 的 worktree 命令、Codex 的 isolate 等），没有原生工具时回退到 git worktree。

### Step 3：Writing Plans — 编写任务计划

将设计文档转化为可执行的任务计划。每个任务粒度为 2-5 分钟，包含：要写什么测试、要跑什么验证、要改哪些文件。计划保存到 `docs/superpowers/plans/` 目录。

### Step 4：Subagent-Driven / Executing Plans — 按计划执行

根据平台能力选择执行方式：
- **支持 subagent 的平台**（Claude Code、Codex CLI）→ `subagent-driven-development`：每个任务派发一个全新的 subagent，主 Agent 只负责协调和审查
- **不支持 subagent 的平台** → `executing-plans`：在当前会话中按计划顺序执行，在关键节点设置 review checkpoint

### Step 5：TDD — 测试驱动开发

每个任务的实现必须遵循红-绿-重构循环：先写失败测试 → 写最小代码通过 → 重构。已经写了代码但没写测试？删掉重来。"这只是个简单改动"不是跳过 TDD 的理由。

### Step 6：Code Review + Verification — 审查与验证

这一步实际包含三个 skill 的协同：
- **requesting-code-review**：发起代码审查，派发审查 subagent
- **receiving-code-review**：接收审查反馈，要求技术验证而非盲目接受（禁止"你说得对！"式敷衍）
- **verification-before-completion**：完成前强制运行验证命令，没有验证证据就不能声称完成

### Step 7：Finish Branch — 分支收尾

验证所有测试通过，根据环境呈现选项（merge / PR / cleanup），执行选择并清理工作空间。

---

### 工作流之外：保障层 skill 的触发时机

7 步工作流是"正常路径"，但开发中总会遇到意外。保障层 skill 不在工作流的固定位置，但在特定触发条件下必须调用：

| 触发条件 | 必须调用的 skill | Iron Law |
|---------|-----------------|----------|
| 遇到 bug、测试失败、异常行为 | `systematic-debugging` | 没有根因分析就不能修 bug |
| 即将声称工作完成 | `verification-before-completion` | 没有验证证据就不能声称完成 |
| 面对多个独立问题 | `dispatching-parallel-agents` | 一个 Agent 一个问题域，并行处理 |
| 需要创建或修改 skill | `writing-skills` | 没有测过基线行为就不知道 skill 教了什么 |

保障层 skill 的设计原则和入口层一样：**不依赖 Agent 的判断，而是用硬性规则强制触发**。"只是个简单 bug，我直接修"——不行，必须先做根因分析。"我确定代码能跑"——不行，必须跑验证命令。

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

## 设计思想三：HARD-GATE — 把"建议"变成"禁止跳过"

7 步工作流中，每个阶段的衔接处都有硬性关卡（HARD-GATE）。

`brainstorming` skill 明确：
```
HARD-GATE: Do NOT invoke any implementation skill, write any code, scaffold any project, 
or take any implementation action until you have presented a design and the user has approved it.
This applies to EVERY project regardless of perceived simplicity.
```

这个设计的意图是：**把 Agent 的软引导（"建议先设计"）升级为硬约束（"必须先设计"）**。各家 harness 的软引导在简单问题上够用，但在复杂项目中，Agent 会在压力下缩减设计深度。HARD-GATE 把"建议"变成了"禁止跳过"，确保设计质量不会因为压力而缩水。

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

**把软件工程中最重要的人类纪律（TDD、系统调试、验证驱动）转化为 AI Agent 无法合理化绕过的强制约束——在各家 harness 软引导的基础上，补上下限保障。**

它的方法论层次：
1. **识别 AI Agent 的失败模式**（通过 baseline 测试收集真实案例）
2. **将正确行为形式化为带强制语气的文本**（Iron Laws, Hard Gates）
3. **用 TDD 方法测试和迭代文本本身**（skills 是被测试过的代码）
4. **通过 session hook 在每次会话开始时注入**（自动触发，无需用户干预）

最终结果：一个不依赖 Agent 记忆、不依赖用户提醒、在每次新会话中自动生效的开发方法论系统。
