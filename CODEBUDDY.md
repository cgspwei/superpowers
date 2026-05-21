# CODEBUDDY.md

This file provides guidance to CodeBuddy Code when working with code in this repository.

## What This Project Is

Superpowers is a zero-dependency multi-harness plugin that injects a software development methodology into AI coding agents at session start. It consists of composable markdown **skills** (not compiled code) that shape agent behavior. Skills auto-trigger via a SessionStart hook that reads `skills/using-superpowers/SKILL.md` and injects it as `EXTREMELY_IMPORTANT` context.

Supported harnesses: Claude Code, Codex CLI, Codex App, Cursor, Factory Droid, Gemini CLI, OpenCode, GitHub Copilot CLI.

## Repository Structure

```
skills/              # 14 composable skills (each is a directory with SKILL.md)
hooks/session-start  # Bash script that bootstraps the plugin at session start
scripts/             # bump-version.sh, sync-to-codex-plugin.sh
tests/               # Harness-specific integration test suites (bash)
docs/                # Harness-specific documentation
.claude-plugin/      # Claude Code plugin config (plugin.json)
.codex-plugin/       # Codex plugin config
.cursor-plugin/      # Cursor plugin config
.opencode/           # OpenCode plugin config + JS entry point
gemini-extension.json
package.json         # Minimal; main field points to .opencode/plugins/superpowers.js
```

### Skills

| Skill | Purpose |
|---|---|
| `brainstorming` | Socratic design refinement before coding |
| `writing-plans` | Break work into 2-5 minute tasks with exact file paths and verification steps |
| `executing-plans` | Batch execution with human checkpoints |
| `subagent-driven-development` | Two-stage review per task (spec compliance, then code quality) |
| `test-driven-development` | RED-GREEN-REFACTOR cycle enforcement |
| `systematic-debugging` | 4-phase root cause process |
| `verification-before-completion` | Ensure fixes actually work before declaring done |
| `requesting-code-review` | Pre-review checklist |
| `receiving-code-review` | Responding to feedback |
| `using-git-worktrees` | Isolated workspaces on new branches |
| `finishing-a-development-branch` | Merge/PR decision workflow |
| `dispatching-parallel-agents` | Concurrent subagent workflows |
| `writing-skills` | Meta-skill for creating and testing new skills (TDD applied to process docs) |
| `using-superpowers` | Introduction/bootstrap skill injected at session start |

## Commands

There is no build step — skills are markdown files deployed directly.

```bash
# Version management
./scripts/bump-version.sh

# Sync to Codex plugin
./scripts/sync-to-codex-plugin.sh
```

Tests are harness-specific integration suites under `tests/`. Each subdirectory contains bash scripts for a specific harness (e.g., `tests/claude-code/`, `tests/codex-plugin-sync/`).

## How the Hook Works

`hooks/session-start` is a bash script that:
1. Reads `skills/using-superpowers/SKILL.md`
2. JSON-escapes the content
3. Emits a different JSON format based on which harness is running (detected via env vars: `CURSOR_PLUGIN_ROOT`, `CLAUDE_PLUGIN_ROOT`, `COPILOT_CLI`)

## Contributing and PRs

This repo has a **94% PR rejection rate**. Read `CLAUDE.md` in full before any PR work.

**Will not be accepted:**
- Third-party dependencies (Superpowers is zero-dependency by design)
- Domain-specific or project-specific skills (publish as a separate plugin)
- Skill "compliance" rewrites without eval evidence showing improvement
- Bulk/batch PRs or speculative fixes without a real experienced problem
- PRs without human review of the complete diff

**Required before any PR:**
1. Fill in every section of `.github/PULL_REQUEST_TEMPLATE.md` — no placeholders
2. Search open AND closed PRs for duplicates; reference what you found
3. Verify a real problem exists (not theoretical)
4. Confirm the change belongs in core (general-purpose, not project-specific)
5. Show the human partner the complete diff and get explicit approval

**Skill changes** require: using `superpowers:writing-skills` for development, adversarial pressure testing across multiple sessions, before/after eval results in the PR. Skills are behavior-shaping code, not prose.

**New harness support** requires a session transcript proving `brainstorming` auto-triggers when the user sends "Let's make a react todo list" in a clean session.

**Contribution branch:** work off the `dev` branch.

## Design Invariants

- **Zero dependencies** — no npm packages, no external services required
- **Skills are behavior-shaping content, not prose** — word-level changes affect agent behavior; don't rewrite without evals
- **"Your human partner"** is deliberate terminology, not interchangeable with "the user"
- **Auto-triggering via bootstrap** — skills are useless without the session hook loading `using-superpowers` at start
- Skills must work across all supported harnesses
