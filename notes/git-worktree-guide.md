# Git Worktree 详解

## 它解决什么问题？

传统 Git 工作流有一个痛点：**同一时间只能看到一个分支的代码**。

假设你正在 `feature-A` 上开发，写到一半，突然需要切到 `main` 分支修一个紧急 bug：

```bash
# 传统方式
git stash          # 先暂存 feature-A 的半成品
git checkout main  # 切换分支
# ... 修 bug、提交、推送 ...
git checkout feature-A
git stash pop      # 恢复半成品
```

**worktree 的思路：** 不用切换，直接开一个"平行目录"，两个分支同时存在，互不干扰。

---

## 核心概念

Git 仓库实际上由两部分组成：

```
你的项目/
├── 工作区 (working tree)   ← 你平时看到的代码文件
└── .git/                   ← 仓库数据（所有分支、提交历史）
```

`git worktree` 的作用是：**让同一个 `.git/` 仓库挂载多个工作区**。

```
.git/  (共享的仓库数据)
  ├── 工作区1: ~/project/          → 在 main 分支
  ├── 工作区2: ~/project/.worktrees/feature-A/  → 在 feature-A 分支
  └── 工作区3: ~/project/.worktrees/hotfix/     → 在 hotfix 分支
```

三个目录指向同一个 `.git`，但各自在不同的分支上，可以同时打开、同时修改。

---

## 常用命令

### 1. 创建 worktree

```bash
# 创建新分支并挂载 worktree
git worktree add ../my-feature -b my-feature

# 基于指定分支创建（默认基于 HEAD）
git worktree add ../my-feature -b my-feature origin/main

# 使用已有分支（分支已存在）
git worktree add ../my-feature my-feature
```

### 2. 查看所有 worktree

```bash
git worktree list
```

输出类似：

```
/Users/simonwang/project          abc123 [main]
/Users/simonwang/project/.worktrees/feature-A   def456 [feature-A]
/Users/simonwang/project/.worktrees/hotfix      ghi789 [hotfix]
```

### 3. 删除 worktree

```bash
# 删除 worktree 目录（分支保留）
git worktree remove ../my-feature

# 删除 worktree 同时删除分支
git worktree remove ../my-feature --delete-branch
```

### 4. 清理已删除目录的残留引用

如果手动删了目录（没用 `git worktree remove`），Git 会报 "worktree is stale"：

```bash
git worktree prune   # 清理残留引用
```

---

## 典型使用场景

### 场景 1：紧急修 bug，不想 stash

```
当前在 feature 分支开发到一半
→ git worktree add ../hotfix -b hotfix
→ cd ../hotfix，修 bug，提交，推送
→ 回到原目录，继续 feature 开发
```

### 场景 2：同时对比两个分支的代码

两个目录同时打开在编辑器里，可以直观对比，不需要来回 checkout。

### 场景 3：在不同分支上运行不同的构建/测试

比如 main 分支跑完整测试，feature 分支只跑单元测试，两个可以并行。

---

## 和类似方案的对比

| | Git Worktree | `git clone` 第二个副本 | 传统 `checkout` 切换 |
|---|---|---|---|
| 磁盘占用 | 共享 `.git`，节省空间 | 完整复制 `.git`，占用大 | 最小 |
| 切换成本 | 零（并行存在） | 零（并行存在） | 需要 stash/commit |
| 共享配置 | 共享（同一 `.git`） | 不共享 | 共享 |
| 适合场景 | 短期并行任务 | 长期并行开发 | 单任务顺序开发 |

---

## 注意事项

1. **不能在同一分支上创建两个 worktree** — 每个分支同一时间只能挂载一个工作区

2. **worktree 目录要加入 `.gitignore`** — 否则 `git status` 会显示一堆未跟踪文件（Superpowers 的技能里也专门强调了这点，见 `skills/using-git-worktrees/SKILL.md` 第 87-95 行）

3. **删除 worktree 要用命令，不要手动删目录** — 手动删会留下残留引用，需要用 `git worktree prune` 清理

4. **IDE 支持程度不一** — VSCode 对 worktree 支持较好，可以在不同窗口打开不同 worktree；部分 IDE 可能识别不准

---

## 总结

**Git worktree 就是"同一仓库、多目录、多分支并行"的工具**，最适合"正在开发，突然要切到别的分支做点事"的场景。Superpowers 的 `using-git-worktrees` 技能在每次执行实施计划前自动设置隔离工作空间，确保主分支不会被半成品代码污染。
