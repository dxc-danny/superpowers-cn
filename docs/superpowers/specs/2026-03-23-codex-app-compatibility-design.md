# Codex App 兼容性：Worktree 和 Finishing 技能适配

使超能力技能在 Codex App 的沙盒化 worktree 环境中工作，而不破坏现有的 Claude Code 或 Codex CLI 行为。

**工单：** PRI-823

## 动机

Codex App 在其管理的 git worktrees 内运行代理——detached HEAD，位于 `$CODEX_HOME/worktrees/` 下，带有阻止 `git checkout -b`、`git push` 和网络访问的 Seatbelt 沙盒。三个超能力技能假设拥有不受限制的 git 访问权限：`using-git-worktrees` 创建手动 worktrees 并使用命名分支，`finishing-a-development-branch` 按分支名称合并/推送/PR，以及 `subagent-driven-development` 需要两者都支持。

Codex CLI（开源终端工具）**没有**这个冲突——它没有内置的 worktree 管理。我们的手动 worktree 方法填补了那里的隔离空白。问题专门针对 Codex App。

## 实证发现

在 2026-03-23 的 Codex App 中测试：

| 操作 | workspace-write 沙盒 | Full access 沙盒 |
|---|---|---|
| `git add` | 可用 | 可用 |
| `git commit` | 可用 | 可用 |
| `git checkout -b` | **被阻止**（无法写入 `.git/refs/heads/`） | 可用 |
| `git push` | **被阻止**（网络 + `.git/refs/remotes/`） | 可用 |
| `gh pr create` | **被阻止**（网络） | 可用 |
| `git status/diff/log` | 可用 | 可用 |

额外发现：
- `spawn_agent` 子代理**共享**父线程的文件系统（通过标记文件测试确认）
- "Create branch" 按钮出现在 App 标题中，无论 worktree 从哪个分支启动
- App 的原生完成流程：Create branch → Commit modal → Commit and push / Commit and create PR
- `network_access = true` 配置在 macOS 上静默损坏（issue #10390）

## 设计：只读环境检测

三个只读 git 命令在不产生副作用的情况下检测环境：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

两个信号：

- **IN_LINKED_WORKTREE：** `GIT_DIR != GIT_COMMON` —— 代理在由其他方创建的 worktree 中（Codex App、Claude Code Agent 工具、之前的技能运行或用户创建）
- **ON_DETACHED_HEAD：** `BRANCH` 为空 —— 不存在命名分支

为什么用 `git-dir != git-common-dir` 而不是检查 `show-toplevel`：
- 在正常仓库中，两者都解析到同一个 `.git` 目录
- 在链接的 worktree 中，`git-dir` 是 `.git/worktrees/<name>` 而 `git-common-dir` 是 `.git`
- 在子模块中，两者相等——避免 `show-toplevel` 会产生的误报
- 通过 `cd && pwd -P` 解析处理相对路径问题（`git-common-dir` 在正常仓库返回相对路径 `.git` 但在 worktree 中返回绝对路径）和符号链接（macOS `/tmp` → `/private/tmp`）

### 决策矩阵

| 链接的 Worktree？ | Detached HEAD？ | 环境 | 动作 |
|---|---|---|---|
| 否 | 否 | Claude Code / Codex CLI / 正常 git | 完整技能行为（不变） |
| 是 | 是 | Codex App worktree（workspace-write） | 跳过 worktree 创建；在完成时提供 handoff payload |
| 是 | 否 | Codex App（Full access）或手动 worktree | 跳过 worktree 创建；完整完成流程 |
| 否 | 是 | 不寻常（手动 detached HEAD） | 正常创建 worktree；在完成时警告 |

## 变更

### 1. `using-git-worktrees/SKILL.md` — 添加步骤 0（约 12 行）

在"概述"和"目录选择过程"之间的新部分：

**步骤 0：检查是否已在隔离工作区中**

运行检测命令。如果 `GIT_DIR != GIT_COMMON`，完全跳过 worktree 创建。改为：
1. 跳到"运行项目设置"子部分（在创建步骤下的"运行项目设置"中）——`npm install` 等是幂等的，为安全起见值得运行
2. 然后"验证干净的基线"——运行测试
3. 用分支状态报告：
   - 在分支上："已在分支 `<name>` 的隔离工作区 `<path>` 中。测试通过。准备好实现。"
   - Detached HEAD："已在隔离工作区 `<path>` 中（detached HEAD，外部管理）。测试通过。注：完成时需要创建分支。准备好实现。"

如果 `GIT_DIR == GIT_COMMON`，继续下面的完整 worktree 创建流程（不变）。

安全验证（.gitignore 检查）在步骤 0 触发时跳过——对外部创建的 worktree 无关。

更新 Integration 部分的"被以下调用"条目。将每个的描述从上下文特定的文本改为："确保隔离工作区（创建或验证已存在）"。例如，`subagent-driven-development` 条目从"REQUIRED: Set up isolated workspace before starting"改为"REQUIRED: Ensures isolated workspace (creates one or verifies existing)"。

**沙盒回退：** 如果 `GIT_DIR == GIT_COMMON` 且技能继续到创建步骤，但 `git worktree add -b` 因权限错误而失败（例如 Seatbelt 沙盒拒绝），将此视为延迟检测到的受限环境。回退到步骤 0"已在工作区"行为——在当前目录运行设置和基线测试，相应报告。

在步骤 0 报告后，停止。不要继续目录选择或创建步骤。

**其他全部不变：** 目录选择、安全验证、创建步骤、项目设置、基线测试、快速参考、常见错误、红旗。

### 2. `finishing-a-development-branch/SKILL.md` — 添加步骤 1.5 + 清理保护（约 20 行）

**步骤 1.5：检测环境**（在步骤 1"验证测试"之后，步骤 2"确定基础分支"之前）

运行检测命令。三条路径：

- **路径 A** 完全跳过步骤 2 和 3（不需要基础分支或选项）。
- **路径 B 和 C** 正常通过步骤 2（确定基础分支）和步骤 3（呈现选项）。

**路径 A — 外部管理的 worktree + detached HEAD**（`GIT_DIR != GIT_COMMON` 且 `BRANCH` 为空）：

首先，确保所有工作已暂存并提交（`git add` + `git commit`）。Codex App 的完成控件对已提交的工作进行操作。

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

分支名称派生：如果有 ticket ID 则使用（例如 `pri-823/codex-compat`），否则将计划标题的前 5 个单词转为 slug，否则省略。避免在分支名称中包含敏感内容（漏洞描述、客户名称）。

跳到步骤 5（清理对于外部管理的 worktree 是 no-op）。

**路径 B — 外部管理的 worktree + 命名分支**（`GIT_DIR != GIT_COMMON` 且 `BRANCH` 存在）：

正常呈现 4 选项菜单。（步骤 5 的清理保护将独立重新检测外部管理状态。）

**路径 C — 正常环境**（`GIT_DIR == GIT_COMMON`）：

正常呈现 4 选项菜单（不变）。

**步骤 5 清理保护：**

在清理时重新运行 `GIT_DIR` vs `GIT_COMMON` 检测（不要依赖早期技能输出——完成技能可能在不同会话中运行）。如果 `GIT_DIR != GIT_COMMON`，跳过 `git worktree remove` —— 主机环境拥有此工作区。

否则，像今天一样检查并移除。注意：现有步骤 5 文本说"For Options 1, 2, 4"但快速参考表和常见错误部分说"Options 1 & 4 only"。新保护在现有逻辑之前添加，不改变触发清理的选项。

**其他全部不变：** 选项 1-4 逻辑、快速参考、常见错误、红旗。

### 3. `subagent-driven-development/SKILL.md` 和 `executing-plans/SKILL.md` — 每处 1 行编辑

两个技能都有相同的 Integration 部分行。从：
```
- superpowers:using-git-worktrees - REQUIRED: Set up isolated workspace before starting
```
改为：
```
- superpowers:using-git-worktrees - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
```

**其他全部不变：** 调度/审查循环、提示模板、模型选择、状态处理、红旗。

### 4. `codex-tools.md` — 添加环境检测文档（约 15 行）

末尾两个新部分：

**环境检测：**

```markdown
## 环境检测

创建 worktree 或完成分支的技能应在继续之前用只读 git 命令检测其环境：

\```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
\```

- `GIT_DIR != GIT_COMMON` → 已在链接的 worktree 中（跳过创建）
- `BRANCH` 为空 → detached HEAD（无法从沙盒分支/推送/PR）

参见 `using-git-worktrees` 步骤 0 和 `finishing-a-development-branch`
步骤 1.5 了解每个技能如何使用这些信号。
```

**Codex App 完成：**

```markdown
## Codex App 完成

当沙盒阻止分支/推送操作（外部管理的 worktree 中的 detached HEAD）时，
代理提交所有工作并告知用户使用 App 的原生控件：

- **"Create branch"** — 命名分支，然后通过 App UI 提交/推送/PR
- **"Hand off to local"** — 将工作转移到用户的本地检出

代理仍可运行测试、暂存文件，并为用户输出建议的分支
名称、提交消息和 PR 描述以供复制。
```

## 不变更的内容

- `implementer-prompt.md`、`spec-reviewer-prompt.md`、`code-quality-reviewer-prompt.md` — 子代理提示未触及
- `executing-plans/SKILL.md` — 只有 1 行 Integration 描述变更（与 `subagent-driven-development` 相同）；所有运行时行为不变
- `dispatching-parallel-agents/SKILL.md` — 无 worktree 或完成操作
- `.codex/INSTALL.md` — 安装过程不变
- 4 选项完成菜单——为 Claude Code 和 Codex CLI 精确保留
- 完整 worktree 创建流程——为非 worktree 环境精确保留
- 子代理调度/审查/迭代循环——不变（文件系统共享已确认）

## 范围摘要

| 文件 | 变更 |
|---|---|
| `skills/using-git-worktrees/SKILL.md` | +12 行（步骤 0） |
| `skills/finishing-a-development-branch/SKILL.md` | +20 行（步骤 1.5 + 清理保护） |
| `skills/subagent-driven-development/SKILL.md` | 1 行编辑 |
| `skills/executing-plans/SKILL.md` | 1 行编辑 |
| `skills/using-superpowers/references/codex-tools.md` | +15 行 |

约 50 行添加/变更，跨 5 个文件。零新文件。零破坏性变更。

## 未来考虑

如果第三个技能需要相同的检测模式，将其提取到共享的 `references/environment-detection.md` 文件（方法 B）。现在不需要——只有 2 个技能使用它。

## 测试计划

### 自动化（在实现后的 Claude Code 中运行）

1. 正常仓库检测 —— 断言 IN_LINKED_WORKTREE=false
2. 链接 worktree 检测 —— `git worktree add` 测试 worktree，断言 IN_LINKED_WORKTREE=true
3. Detached HEAD 检测 —— `git checkout --detach`，断言 ON_DETACHED_HEAD=true
4. 完成技能 handoff 输出 —— 在受限环境中验证 handoff 消息（非 4 选项菜单）
5. **步骤 5 清理保护** —— 创建链接 worktree（`git worktree add /tmp/test-cleanup -b test-cleanup`），`cd` 进入其中，运行步骤 5 清理检测（`GIT_DIR` vs `GIT_COMMON`），断言它**不会**调用 `git worktree remove`。然后 `cd` 回主仓库，运行相同的检测，断言它**会**调用 `git worktree remove`。之后清理测试 worktree。

### 手动 Codex App 测试（5 个测试）

1. Worktree 线程中的检测（workspace-write）—— 验证 GIT_DIR != GIT_COMMON，空分支
2. Worktree 线程中的检测（Full access）—— 相同检测，不同沙盒行为
3. 完成技能 handoff 格式 —— 验证代理发出 handoff payload，非 4 选项菜单
4. 完整生命周期 —— 检测 → 提交 → 完成检测 → 正确行为 → 清理
5. **Local 线程中的沙盒回退** —— 启动 Codex App **Local 线程**（workspace-write 沙盒）。提示："Use the superpowers skill `using-git-worktrees` to set up an isolated workspace for implementing a small change." 预检查：`git checkout -b test-sandbox-check` 应因 `Operation not permitted` 失败。预期：技能检测 `GIT_DIR == GIT_COMMON`（正常仓库），尝试 `git worktree add -b`，遇到 Seatbelt 拒绝，优雅回退到步骤 0"已在工作区"行为——在当前目录运行设置、基线测试、报告就绪。成功：代理优雅恢复，无神秘错误消息。失败：代理打印原始 Seatbelt 错误、重试或以混乱输出放弃。

### 回归

- 现有 Claude Code 技能触发测试仍然通过
- 现有 subagent-driven-development 集成测试仍然通过
- 正常 Claude Code 会话：完整 worktree 创建 + 4 选项完成仍然工作