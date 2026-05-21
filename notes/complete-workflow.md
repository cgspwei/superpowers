# Superpowers 完整开发工作流

## 概览

Superpowers 是一个线性的、关卡式（gated）的开发流程管道。每个阶段都有硬性门禁（HARD-GATE），不允许跳过。流程的启动是自动的——通过 SessionStart hook 注入 `using-superpowers` 技能。

```
会话启动 → [using-superpowers 自动注入]
     ↓
提出需求 → [brainstorming] → 设计文档 + 用户批准
     ↓
[writing-plans] → 细粒度实施计划
     ↓
[using-git-worktrees] → 隔离工作空间
     ↓
执行计划 ─┬─ [subagent-driven-development]（推荐）
          └─ [executing-plans]
     ↓
每步遵循 [test-driven-development] RED → GREEN → REFACTOR
     ↓
遇 bug → [systematic-debugging] 四阶段根因分析
     ↓
完成前 → [verification-before-completion] 运行验证
     ↓
[requesting-code-review] + [receiving-code-review]
     ↓
[finishing-a-development-branch] → 合并/PR/保留/放弃
```

---

## 阶段一：会话启动（`using-superpowers`）

**触发方式：** 自动（SessionStart hook）

每次 AI 会话开始时，`using-superpowers/SKILL.md` 的内容被作为 `EXTREMELY_IMPORTANT` 上下文注入。它建立的核心规则：

- **1% 规则：** 如果有哪怕 1% 的可能某个 skill 适用，就必须调用它
- **技能优先级：** 流程类 skill（brainstorming、debugging）优先于实现类 skill
- **用户指令最高：** CLAUDE.md / GEMINI.md / AGENTS.md 中的显式指令优先于 skill 规则

Agent 收到任何用户消息后的决策流程：

```
收到消息 → 即将进入计划模式？ → 已 brainstorm 过？ 
                                        ├─ 否 → 调用 brainstorming
                                        └─ 是 → 任何 skill 可能适用？
                                                    ├─ 是 → 调用 Skill 工具
                                                    └─ 否 → 回复
```

---

## 阶段二：头脑风暴（`brainstorming`）

**触发条件：** 任何创造性工作——创建功能、构建组件、添加功能、修改行为

**硬性门禁：**

> Do NOT invoke any implementation skill, write any code, scaffold any project, or take any implementation action until you have presented a design and the user has approved it. This applies to EVERY project regardless of perceived simplicity.

### Checklist（必须逐项完成）

1. **探索项目上下文** — 检查文件、文档、最近提交
2. **提议 Visual Companion**（如果涉及视觉问题）— 必须是独立消息，不与提问混在一起
3. **逐一追问** — 每条消息只问一个问题，多选优先，理清目的/约束/成功标准
   - 如果项目太大（多个独立子系统），先分解为子项目，每个子项目独立走完整流程
4. **提出 2-3 个方案** — 含权衡和推荐，推荐方案优先展示
5. **分段呈现设计** — 按复杂度缩放（简单几句话，复杂 200-300 字/节），逐节确认
   - 覆盖：架构、组件、数据流、错误处理、测试
   - 设计原则：隔离、清晰接口、可独立理解和测试的单元
6. **写设计文档** — 保存到 `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` 并 git commit
7. **Spec 自审** — 扫描四类问题：
   - 占位符（TBD、TODO）
   - 内部矛盾
   - 范围过大需分解
   - 歧义（可被两种理解的需求）
8. **用户审阅 Spec** — 等待确认后才可进入下一阶段
9. **调用 `writing-plans`** — 唯一允许的下一步

**关键原则：**
- 一次只问一个问题
- YAGNI（你不会需要它）—— 从所有设计中移除不必要的功能
- 递增验证—— 分段确认，不要一次性抛出整个设计

---

## 阶段三：编写实施计划（`writing-plans`）

**触发条件：** 已有 spec 或需求文档，准备写代码之前

### 计划文档结构

```markdown
# [功能名] Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) 
> or superpowers:executing-plans to implement this plan task-by-task.

**Goal:** [一句话描述]
**Architecture:** [2-3 句方案]
**Tech Stack:** [关键技术/库]
```

### 核心要求

1. **范围检查** — 如 spec 涵盖多个独立子系统，拆分为多个计划
2. **文件结构映射** — 在定义任务前，先列出每个文件的职责
3. **细粒度任务** — 每步 2-5 分钟，严格遵循 TDD 步骤：
   - 写失败测试 → 确认失败 → 写最小实现 → 确认通过 → 提交
4. **禁止占位符** — 以下均为计划失败：
   - "TBD"、"TODO"、"implement later"
   - "Add appropriate error handling"（没有实际代码）
   - "Write tests for the above"（没有实际测试代码）
   - "Similar to Task N"（必须重复代码，执行者可能乱序阅读）
5. **每步包含：** 精确文件路径 + 完整代码 + 精确命令 + 预期输出

### 任务模板

````markdown
### Task N: [组件名]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

- [ ] **Step 1: Write the failing test**
```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

- [ ] **Step 2: Run test to verify it fails**
Run: `pytest tests/path/test.py::test_name -v`
Expected: FAIL with "function not defined"

- [ ] **Step 3: Write minimal implementation**
[完整代码]

- [ ] **Step 4: Run test to verify it passes**
Run: `pytest tests/path/test.py::test_name -v`
Expected: PASS

- [ ] **Step 5: Commit**
```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
````

### 自审

1. **Spec 覆盖** — 每个 spec 需求都有对应任务？
2. **占位符扫描** — 搜索所有禁止模式
3. **类型一致性** — 后续任务中的函数名/签名与前面定义的一致？

### 执行交接

保存计划后提供选择：
- **Subagent 驱动（推荐）** — 每个 task 派发独立 subagent，任务间双阶段审查
- **内联执行** — 当前会话批量执行，检查点审查

---

## 阶段四：环境隔离（`using-git-worktrees`）

**触发条件：** 开始功能开发，需要与当前工作空间隔离

### 步骤

```
Step 0: 检测是否已在隔离环境
  ├─ GIT_DIR != GIT_COMMON（且非 submodule）→ 已在 worktree，跳到 Step 3
  └─ GIT_DIR == GIT_COMMON → 继续 Step 1

Step 1: 创建隔离工作空间
  ├─ Step 1a: 优先使用平台原生工具（EnterWorktree / WorktreeCreate 等）
  └─ Step 1b: 无原生工具时用 git worktree add（需用户同意）

Step 3: 项目设置 — 自动检测并安装依赖（npm/cargo/pip/go）

Step 4: 验证干净基线 — 运行测试确保全绿
```

**目录选择优先级：**
1. 用户声明的偏好
2. 已有的 `.worktrees/` 或 `worktrees/`
3. 已有的 `~/.config/superpowers/worktrees/<project>/`
4. 默认 `.worktrees/`（项目根目录）

---

## 阶段五：执行实施

### 选项 A：Subagent 驱动开发（`subagent-driven-development`，推荐）

**适用条件：** 有实施计划 + 任务基本独立 + 在当前会话中执行

**每个任务的循环：**

```
1. 派发实现 subagent（使用 implementer-prompt.md 模板）
   ├─ 实现者提问？→ 回答，重新派发
   └─ 实现者完成 → 继续

2. 派发 Spec 合规审查 subagent（spec-reviewer-prompt.md）
   ├─ 不合规 → 实现者修复 → 重新审查
   └─ 合规 → 继续

3. 派发代码质量审查 subagent（code-quality-reviewer-prompt.md）
   ├─ 不通过 → 实现者修复 → 重新审查
   └─ 通过 → 标记任务完成

4. 下一个任务（连续执行，不在任务间暂停请示用户）
```

**处理实现者状态：**

| 状态 | 处理 |
|------|------|
| DONE | 进入 Spec 审查 |
| DONE_WITH_CONCERNS | 读完顾虑后决定是否处理，然后进入审查 |
| NEEDS_CONTEXT | 补充信息后重新派发 |
| BLOCKED | 评估：补上下文 / 升级模型 / 拆小任务 / 上报人类 |

**模型选择策略：**

| 任务类型 | 模型选择 |
|----------|----------|
| 机械实现（1-2 文件，规格清晰） | 最便宜最快的模型 |
| 集成/判断（多文件协调，模式匹配） | 标准模型 |
| 架构/设计/审查 | 最强可用模型 |

**全部任务完成后：** 派发最终代码审查 subagent，然后调用 `finishing-a-development-branch`

**禁止事项：**
- 在 main/master 上开始实现
- 跳过任何审查
- 在 Spec 审查通过前开始代码质量审查
- 并行派发多个实现 subagent（会冲突）
- 让实现者自审替代正式审查

---

### 选项 B：内联执行（`executing-plans`）

**适用条件：** 无 subagent 支持的平台，或需要在并行会话中执行

**流程：**

```
Step 1: 加载并审查计划
  ├─ 有顾虑 → 提出后再开始
  └─ 无顾虑 → 创建 TodoWrite，开始执行

Step 2: 逐个执行任务
  - 按步骤严格执行
  - 运行指定的验证命令
  - 遇到阻塞立即停下问人

Step 3: 完成后调用 finishing-a-development-branch
```

**停下请示的时机：**
- 遇到阻塞（缺失依赖、测试失败、指令不清）
- 计划有严重缺口
- 不理解指令
- 验证反复失败

---

## 每步实现遵循 TDD（`test-driven-development`）

**铁律：**

> NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST

**RED-GREEN-REFACTOR 循环：**

```
RED:    写一个最小测试展示期望行为
  ↓
验证 RED: 运行测试，确认因"功能缺失"而失败（非拼写错误）
  ↓
GREEN:  写最简单代码使测试通过
  ↓
验证 GREEN: 运行测试，确认通过 + 其他测试仍绿 + 输出无错误/警告
  ↓
REFACTOR: 清理（去重、改命名、提取辅助函数），不添加行为
  ↓
验证仍绿 → 下一个 RED
```

**例外（需用户同意）：** 抛弃式原型、生成代码、配置文件

---

## 遇到 Bug：系统调试（`systematic-debugging`）

**铁律：**

> NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST

**四阶段流程：**

### Phase 1: 根因调查
1. 仔细读错误信息（包括完整堆栈）
2. 稳定复现
3. 检查最近变更（git diff、最近提交）
4. 多组件系统：在每个组件边界加诊断日志，定位断在哪层
5. 追踪数据流：从错误值反向追踪到源头

### Phase 2: 模式分析
1. 找同类工作代码
2. 完整阅读参考实现（不要略读）
3. 列出每一个差异
4. 理解依赖

### Phase 3: 假设与验证
1. 形成单一假设："我认为 X 是根因，因为 Y"
2. 做最小变更测试假设
3. 验证：成功 → Phase 4；失败 → 新假设

### Phase 4: 实现
1. 写失败测试（用 TDD skill）
2. 实现单点修复（一个改动，不含"顺手"改进）
3. 验证修复
4. **3+ 次修复失败 → 质疑架构，与 human partner 讨论**

**红旗：** "快速修一下"、"试试改 X 看看"、"跳过测试手动验证"——这些意味着立刻回到 Phase 1

---

## 完成前验证（`verification-before-completion`）

**铁律：**

> NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE

**门禁函数：**

```
声称任何状态前：
1. IDENTIFY: 什么命令能证明这个声明？
2. RUN: 执行完整命令（全新运行）
3. READ: 读取完整输出，检查退出码，统计失败数
4. VERIFY: 输出是否确认声明？
   - 否 → 陈述实际状态和证据
   - 是 → 用证据陈述声明
5. 只在此时：做出声明
```

**各类声明的验证要求：**

| 声明 | 必须有 | 不够的 |
|------|--------|--------|
| 测试通过 | 测试命令输出 0 failures | 之前的运行、"应该通过" |
| Linter 干净 | Linter 输出 0 errors | 部分检查、推断 |
| 构建成功 | 构建命令 exit 0 | Linter 通过 |
| Bug 已修复 | 运行原始症状验证 | 代码改了、假设修复了 |

---

## 代码审查

### 请求审查（`requesting-code-review`）

**触发时机：** subagent 驱动中每个任务后 / 重大功能完成后 / 合并前

**步骤：**
1. 获取 git SHA（`BASE_SHA` 和 `HEAD_SHA`）
2. 派发代码审查 subagent
3. 处理反馈：Critical 立即修 → Important 在继续前修 → Minor 记录稍后处理

### 接收审查反馈（`receiving-code-review`）

**核心原则：** 验证后再实现，提问而非假设，技术正确优先于社交舒适

**响应模式：**
```
1. READ: 完整阅读，不反应
2. UNDERSTAND: 用自己的话重述（或提问）
3. VERIFY: 对照代码库验证
4. EVALUATE: 对本代码库技术可行？
5. RESPOND: 技术性确认 或 有理有据的回推
6. IMPLEMENT: 逐项实现，逐项测试
```

**禁止：**
- 表演性赞同（"你说得对！"、"好建议！"）
- 未验证就实现
- 批量实现不逐项测试

**回推的情况：**
- 建议破坏现有功能
- 审查者缺少完整上下文
- 违反 YAGNI（未使用的功能）
- 与 human partner 的架构决策冲突

---

## 并行 Agent（`dispatching-parallel-agents`）

**适用条件：** 2+ 个独立问题域（不同子系统、不同 bug、无共享状态）

**流程：**
1. 按问题域分组
2. 每组创建专注的 agent 任务（特定范围 + 清晰目标 + 约束 + 预期输出）
3. 并行派发
4. 整合结果 → 检查冲突 → 运行全量测试

**不适用：** 问题相关 / 需要全局上下文 / 探索性调试 / 共享状态

---

## 完成开发分支（`finishing-a-development-branch`）

**步骤：**

```
Step 1: 验证测试全绿（失败则停止）
Step 2: 检测环境状态（普通 repo / worktree / detached HEAD）
Step 3: 确定 base 分支
Step 4: 呈现选项

普通 repo / 命名分支 worktree → 4 个选项：
  1. 本地合并回 <base-branch>
  2. 推送并创建 PR
  3. 保持现状稍后处理
  4. 放弃所有工作（需输入 "discard" 确认）

detached HEAD → 3 个选项：
  1. 推送为新分支并创建 PR
  2. 保持现状
  3. 放弃
```

**Worktree 清理规则：**
- 选项 1（合并）和 4（放弃）→ 清理 worktree + 删除分支
- 选项 2（PR）和 3（保留）→ 保持 worktree 存活
- 只清理 Superpowers 创建的 worktree（`.worktrees/`、`worktrees/`、`~/.config/superpowers/worktrees/`）

---

## 产出文件约定

| 产出 | 路径 |
|------|------|
| 设计文档 | `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` |
| 实施计划 | `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md` |
| Worktree 目录 | `.worktrees/<branch-name>/` |

（用户偏好可覆盖默认路径）
