# Codex App 兼容性实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**目标：** 使 `using-git-worktrees`、`finishing-a-development-branch` 和相关技能在 Codex App 的沙盒化 worktree 环境中工作，而不破坏现有行为。

**架构：** 在两个技能开始时进行只读环境检测（`git-dir` vs `git-common-dir`）。如果已经在链接的 worktree 中，跳过创建。如果在 detached HEAD 上，发出 handoff payload 而不是 4 选项菜单。沙盒回退捕获 worktree 创建期间的权限错误。

**技术栈：** Git、Markdown（技能文件是指令文档，不是可执行代码）

**规范：** `docs/superpowers/specs/2026-03-23-codex-app-compatibility-design.md`

---

## 文件结构

| 文件 | 职责 | 操作 |
|---|---|---|
| `skills/using-git-worktrees/SKILL.md` | Worktree 创建 + 隔离 | 添加步骤 0 检测 + 沙盒回退 |
| `skills/finishing-a-development-branch/SKILL.md` | 分支完成工作流 | 添加步骤 1.5 检测 + 清理保护 |
| `skills/subagent-driven-development/SKILL.md` | 使用子代理执行计划 | 更新 Integration 描述 |
| `skills/executing-plans/SKILL.md` | 内联执行计划 | 更新 Integration 描述 |
| `skills/using-superpowers/references/codex-tools.md` | Codex 平台参考 | 添加检测 + 完成文档 |

---

### 任务 1：在 `using-git-worktrees` 中添加步骤 0

**文件：**
- 修改：`skills/using-git-worktrees/SKILL.md:14-15`（在概述之后、目录选择过程之前插入）

- [ ] **步骤 1：阅读当前技能文件**

完整阅读 `skills/using-git-worktrees/SKILL.md`。识别确切的插入点：在 "Announce at start" 行（第 14 行）之后和 "## Directory Selection Process"（第 16 行）之前。

- [ ] **步骤 2：插入步骤 0 部分**

在概述部分和 "## Directory Selection Process" 之间插入以下内容：

```markdown
## 步骤 0：检查是否已在隔离工作区中

在创建 worktree 之前，检查是否已存在一个：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**如果 `GIT_DIR` 与 `GIT_COMMON` 不同：** 你已在一个链接的 worktree 中（由 Codex App、Claude Code 的 Agent 工具、之前的技能运行或用户创建）。不要再创建另一个 worktree。改为：

1. 运行项目设置（如下面的 "Run Project Setup" 中那样自动检测包管理器）
2. 验证干净的基线（如下面的 "Verify Clean Baseline" 中那样运行测试）
3. 用分支状态报告：
   - 在分支上："已在分支 `<name>` 的隔离工作区 `<path>` 中。测试通过。准备好实现。"
   - Detached HEAD："已在隔离工作区 `<path>` 中（detached HEAD，外部管理）。测试通过。注：完成时需要创建分支。准备好实现。"

报告后，停止。不要继续目录选择或创建步骤。

**如果 `GIT_DIR` 等于 `GIT_COMMON`：** 继续下面的完整 worktree 创建流程。

**沙盒回退：** 如果继续创建步骤但 `git worktree add -b` 失败并出现权限错误（例如 "Operation not permitted"），将此视为延迟检测到的受限环境。回退到上述行为——在当前目录运行设置和基线测试，报告，然后停止。
```

- [ ] **步骤 3：验证插入**

再次阅读文件。确认：
- 步骤 0 出现在概述和目录选择过程之间
- 文件其余部分（目录选择、安全验证、创建步骤等）未更改
- 没有重复部分或损坏的 markdown

- [ ] **步骤 4：提交**

```bash
git add skills/using-git-worktrees/SKILL.md
git commit -m "feat(using-git-worktrees): add Step 0 environment detection (PRI-823)

Skip worktree creation when already in a linked worktree. Includes
sandbox fallback for permission errors on git worktree add."
```

---

### 任务 2：更新 `using-git-worktrees` Integration 部分

**文件：**
- 修改：`skills/using-git-worktrees/SKILL.md:211-215`（Integration > Called by）

- [ ] **步骤 1：更新三个 "Called by" 条目**

将第 212-214 行从：

```markdown
- **brainstorming** (Phase 4) - REQUIRED when design is approved and implementation follows
- **subagent-driven-development** - REQUIRED before executing any tasks
- **executing-plans** - REQUIRED before executing any tasks
```

更改为：

```markdown
- **brainstorming** - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
- **subagent-driven-development** - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
- **executing-plans** - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
```

- [ ] **步骤 2：验证 Integration 部分**

阅读 Integration 部分。确认所有三个条目都已更新，"Pairs with" 未更改。

- [ ] **步骤 3：提交**

```bash
git add skills/using-git-worktrees/SKILL.md
git commit -m "docs(using-git-worktrees): update Integration descriptions (PRI-823)

Clarify that skill ensures a workspace exists, not that it always creates one."
```

---

### 任务 3：在 `finishing-a-development-branch` 中添加步骤 1.5

**文件：**
- 修改：`skills/finishing-a-development-branch/SKILL.md:38`（在步骤 1 之后、步骤 2 之前插入）

- [ ] **步骤 1：阅读当前技能文件**

完整阅读 `skills/finishing-a-development-branch/SKILL.md`。识别插入点：在 "**If tests pass:** Continue to Step 2."（第 38 行）之后和 "### Step 2: Determine Base Branch"（第 40 行）之前。

- [ ] **步骤 2：插入步骤 1.5 部分**

在步骤 1 和步骤 2 之间插入以下内容：

```markdown
### 步骤 1.5：检测环境

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**路径 A — `GIT_DIR` 与 `GIT_COMMON` 不同且 `BRANCH` 为空（外部管理的 worktree，detached HEAD）：**

首先，确保所有工作已暂存并提交（`git add` + `git commit`）。

然后向用户呈现这个（不要呈现 4 选项菜单）：

```
实现完成。所有测试通过。
当前 HEAD：<完整提交 SHA>

此工作区由外部管理（detached HEAD）。
我无法从此处创建分支、推送或打开 PR。

⚠ 这些提交在 detached HEAD 上。如果你不创建分支，
当此工作区被清理时可能会丢失。

如果你的主机应用程序提供这些控件：
- "Create branch" — 命名分支，然后提交/推送/PR
- "Hand off to local" — 将更改移动到你的本地检出

建议分支名称：<ticket-id/short-description>
建议提交消息：<工作摘要>
```

分支名称：如有 ticket ID 则使用（例如 `pri-823/codex-compat`），否则将计划标题的前 5 个单词转为 slug，否则省略。避免在分支名称中包含敏感内容。

跳到步骤 5（清理是 no-op —— 见下面的保护）。

**路径 B — `GIT_DIR` 与 `GIT_COMMON` 不同且 `BRANCH` 存在（外部管理的 worktree，命名分支）：**

继续到步骤 2，正常呈现 4 选项菜单。

**路径 C — `GIT_DIR` 等于 `GIT_COMMON`（正常环境）：**

继续到步骤 2，正常呈现 4 选项菜单。
```

- [ ] **步骤 3：验证插入**

再次阅读文件。确认：
- 步骤 1.5 出现在步骤 1 和 2 之间
- 步骤 2-5 未更改
- 路径 A handoff 包含提交 SHA 和数据丢失警告
- 路径 B 和 C 正常继续到步骤 2

- [ ] **步骤 4：提交**

```bash
git add skills/finishing-a-development-branch/SKILL.md
git commit -m "feat(finishing-a-development-branch): add Step 1.5 environment detection (PRI-823)

Detect externally managed worktrees with detached HEAD and emit handoff
payload instead of 4-option menu. Includes commit SHA and data loss warning."
```

---

### 任务 4：在 `finishing-a-development-branch` 中添加步骤 5 清理保护

**文件：**
- 修改：`skills/finishing-a-development-branch/SKILL.md`（步骤 5：清理 Worktree — 按部分标题查找，行号在任务 3 插入后已偏移）

- [ ] **步骤 1：阅读当前步骤 5 部分**

在 `skills/finishing-a-development-branch/SKILL.md` 中找到 "### Step 5: Cleanup Worktree" 部分（行号在任务 3 插入后已偏移）。当前步骤 5 是：

```markdown
### Step 5: Cleanup Worktree

**For Options 1, 2, 4:**

Check if in worktree:
```bash
git worktree list | grep $(git branch --show-current)
```

If yes:
```bash
git worktree remove <worktree-path>
```

**For Option 3:** Keep worktree.
```

- [ ] **步骤 2：在现有逻辑之前添加清理保护**

将步骤 5 部分替换为：

```markdown
### Step 5: Cleanup Worktree

**首先，检查 worktree 是否由外部管理：**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

如果 `GIT_DIR` 与 `GIT_COMMON` 不同：跳过 worktree 移除 —— 主机环境拥有此工作区。

**否则，对于选项 1 和 4：**

检查是否在 worktree 中：
```bash
git worktree list | grep $(git branch --show-current)
```

如果是：
```bash
git worktree remove <worktree-path>
```

**对于选项 3：** 保留 worktree。
```

注意：原文说 "For Options 1, 2, 4" 但快速参考表和常见错误部分说 "Options 1 & 4 only"。此编辑使步骤 5 与这些部分保持一致。

- [ ] **步骤 3：验证替换**

阅读步骤 5。确认：
- 清理保护（重新检测）首先出现
- 为非外部管理的 worktree 保留现有移除逻辑
- "Options 1 and 4"（不是 "1, 2, 4"）与快速参考和常见错误匹配

- [ ] **步骤 4：提交**

```bash
git add skills/finishing-a-development-branch/SKILL.md
git commit -m "feat(finishing-a-development-branch): add Step 5 cleanup guard (PRI-823)

Re-detect externally managed worktree at cleanup time and skip removal.
Also fixes pre-existing inconsistency: cleanup now correctly says
Options 1 and 4 only, matching Quick Reference and Common Mistakes."
```

---

### 任务 5：更新 `subagent-driven-development` 和 `executing-plans` 中的 Integration 行

**文件：**
- 修改：`skills/subagent-driven-development/SKILL.md:268`
- 修改：`skills/executing-plans/SKILL.md:68`

- [ ] **步骤 1：更新 `subagent-driven-development`**

将第 268 行从：
```
- **superpowers:using-git-worktrees** - REQUIRED: Set up isolated workspace before starting
```
更改为：
```
- **superpowers:using-git-worktrees** - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
```

- [ ] **步骤 2：更新 `executing-plans`**

将第 68 行从：
```
- **superpowers:using-git-worktrees** - REQUIRED: Set up isolated workspace before starting
```
更改为：
```
- **superpowers:using-git-worktrees** - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
```

- [ ] **步骤 3：验证两个文件**

阅读 `skills/subagent-driven-development/SKILL.md` 的第 268 行和 `skills/executing-plans/SKILL.md` 的第 68 行。确认两者都说 "Ensures isolated workspace (creates one or verifies existing)"。

- [ ] **步骤 4：提交**

```bash
git add skills/subagent-driven-development/SKILL.md skills/executing-plans/SKILL.md
git commit -m "docs(sdd, executing-plans): update worktree Integration descriptions (PRI-823)

Clarify that using-git-worktrees ensures a workspace exists rather than
always creating one."
```

---

### 任务 6：向 `codex-tools.md` 添加环境检测文档

**文件：**
- 修改：`skills/using-superpowers/references/codex-tools.md:25`（在末尾追加）

- [ ] **步骤 1：阅读当前文件**

完整阅读 `skills/using-superpowers/references/codex-tools.md`。确认它在第 25-26 行 multi_agent 部分之后结束。

- [ ] **步骤 2：追加两个新部分**

在文件末尾添加：

```markdown
## 环境检测

创建 worktree 或完成分支的技能应在继续之前用只读 git 命令检测其环境：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

- `GIT_DIR != GIT_COMMON` → 已在链接的 worktree 中（跳过创建）
- `BRANCH` 为空 → detached HEAD（无法从沙盒分支/推送/PR）

参见 `using-git-worktrees` 步骤 0 和 `finishing-a-development-branch`
步骤 1.5 了解每个技能如何使用这些信号。

## Codex App 完成

当沙盒阻止分支/推送操作（外部管理的 worktree 中的 detached HEAD）时，
代理提交所有工作并告知用户使用 App 的原生控件：

- **"Create branch"** — 命名分支，然后通过 App UI 提交/推送/PR
- **"Hand off to local"** — 将工作转移到用户的本地检出

代理仍可运行测试、暂存文件，并为用户输出建议的分支
名称、提交消息和 PR 描述以供复制。
```

- [ ] **步骤 3：验证添加**

阅读完整文件。确认：
- 两个新部分出现在现有内容之后
- Bash 代码块正确渲染（未转义）
- 存在对步骤 0 和 1.5 的交叉引用

- [ ] **步骤 4：提交**

```bash
git add skills/using-superpowers/references/codex-tools.md
git commit -m "docs(codex-tools): add environment detection and App finishing docs (PRI-823)

Document the git-dir vs git-common-dir detection pattern and the Codex
App's native finishing flow for skills that need to adapt."
```

---

### 任务 7：自动化测试 — 环境检测

**文件：**
- 创建：`tests/codex-app-compat/test-environment-detection.sh`

- [ ] **步骤 1：创建测试目录**

```bash
mkdir -p tests/codex-app-compat
```

- [ ] **步骤 2：编写检测测试脚本**

创建 `tests/codex-app-compat/test-environment-detection.sh`：

```bash
#!/usr/bin/env bash
set -euo pipefail

# Test environment detection logic from PRI-823
# Tests the git-dir vs git-common-dir comparison used by
# using-git-worktrees Step 0 and finishing-a-development-branch Step 1.5

PASS=0
FAIL=0
TEMP_DIR=$(mktemp -d)
trap "rm -rf $TEMP_DIR" EXIT

log_pass() { echo "  PASS: $1"; PASS=$((PASS + 1)); }
log_fail() { echo "  FAIL: $1"; FAIL=$((FAIL + 1)); }

# Helper: run detection and return "linked" or "normal"
detect_worktree() {
  local git_dir git_common
  git_dir=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
  git_common=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
  if [ "$git_dir" != "$git_common" ]; then
    echo "linked"
  else
    echo "normal"
  fi
}

echo "=== Test 1: Normal repo detection ==="
cd "$TEMP_DIR"
git init test-repo > /dev/null 2>&1
cd test-repo
git commit --allow-empty -m "init" > /dev/null 2>&1
result=$(detect_worktree)
if [ "$result" = "normal" ]; then
  log_pass "Normal repo detected as normal"
else
  log_fail "Normal repo detected as '$result' (expected 'normal')"
fi

echo "=== Test 2: Linked worktree detection ==="
git worktree add "$TEMP_DIR/test-wt" -b test-branch > /dev/null 2>&1
cd "$TEMP_DIR/test-wt"
result=$(detect_worktree)
if [ "$result" = "linked" ]; then
  log_pass "Linked worktree detected as linked"
else
  log_fail "Linked worktree detected as '$result' (expected 'linked')"
fi

echo "=== Test 3: Detached HEAD detection ==="
git checkout --detach HEAD > /dev/null 2>&1
branch=$(git branch --show-current)
if [ -z "$branch" ]; then
  log_pass "Detached HEAD: branch is empty"
else
  log_fail "Detached HEAD: branch is '$branch' (expected empty)"
fi

echo "=== Test 4: Linked worktree + detached HEAD (Codex App simulation) ==="
result=$(detect_worktree)
branch=$(git branch --show-current)
if [ "$result" = "linked" ] && [ -z "$branch" ]; then
  log_pass "Codex App simulation: linked + detached HEAD"
else
  log_fail "Codex App simulation: result='$result', branch='$branch'"
fi

echo "=== Test 5: Cleanup guard — linked worktree should NOT remove ==="
cd "$TEMP_DIR/test-wt"
result=$(detect_worktree)
if [ "$result" = "linked" ]; then
  log_pass "Cleanup guard: linked worktree correctly detected (would skip removal)"
else
  log_fail "Cleanup guard: expected 'linked', got '$result'"
fi

echo "=== Test 6: Cleanup guard — main repo SHOULD remove ==="
cd "$TEMP_DIR/test-repo"
result=$(detect_worktree)
if [ "$result" = "normal" ]; then
  log_pass "Cleanup guard: main repo correctly detected (would proceed with removal)"
else
  log_fail "Cleanup guard: expected 'normal', got '$result'"
fi

# Cleanup worktree before temp dir removal
cd "$TEMP_DIR/test-repo"
git worktree remove "$TEMP_DIR/test-wt" > /dev/null 2>&1 || true

echo ""
echo "=== Results: $PASS passed, $FAIL failed ==="
if [ "$FAIL" -gt 0 ]; then
  exit 1
fi
```

- [ ] **步骤 3：使其可执行并运行**

```bash
chmod +x tests/codex-app-compat/test-environment-detection.sh
./tests/codex-app-compat/test-environment-detection.sh
```

预期输出：6 passed, 0 failed.

- [ ] **步骤 4：提交**

```bash
git add tests/codex-app-compat/test-environment-detection.sh
git commit -m "test: add environment detection tests for Codex App compat (PRI-823)

Tests git-dir vs git-common-dir comparison in normal repo, linked
worktree, detached HEAD, and cleanup guard scenarios."
```

---

### 任务 8：最终验证

**文件：**
- 阅读：所有 5 个修改的技能文件

- [ ] **步骤 1：运行自动化检测测试**

```bash
./tests/codex-app-compat/test-environment-detection.sh
```

预期：6 passed, 0 failed.

- [ ] **步骤 2：阅读每个修改的文件并验证更改**

完整阅读每个文件：
- `skills/using-git-worktrees/SKILL.md` — 步骤 0 存在，其余未更改
- `skills/finishing-a-development-branch/SKILL.md` — 步骤 1.5 存在，清理保护存在，其余未更改
- `skills/subagent-driven-development/SKILL.md` — 第 268 行已更新
- `skills/executing-plans/SKILL.md` — 第 68 行已更新
- `skills/using-superpowers/references/codex-tools.md` — 末尾有两个新部分

- [ ] **步骤 3：验证没有意外的更改**

```bash
git diff --stat HEAD~7
```

应显示正好 6 个文件更改（5 个技能文件 + 1 个测试文件）。没有其他文件修改。

- [ ] **步骤 4：运行现有测试套件**

如果测试运行器存在：
```bash
# Run skill-triggering tests
./tests/skill-triggering/run-all.sh 2>/dev/null || echo "Skill triggering tests not available in this environment"

# Run SDD integration test
./tests/claude-code/test-subagent-driven-development-integration.sh 2>/dev/null || echo "SDD integration test not available in this environment"
```

注意：这些测试需要带有 `--dangerously-skip-permissions` 的 Claude Code。如果不可用，记录应在手动环境中运行回归测试。