# Codex 工具映射

技能使用 Claude Code 的工具名称。当你在技能中遇到这些时，使用你的平台等效工具：

| 技能引用 | Codex 等效工具 |
|-----------------|------------------|
| `Task` 工具（调度子代理） | `spawn_agent`（参见 [命名代理调度](#named-agent-dispatch)） |
| 多个 `Task` 调用（并行） | 多个 `spawn_agent` 调用 |
| Task 返回结果 | `wait` |
| Task 自动完成 | `close_agent` 释放槽位 |
| `TodoWrite`（任务跟踪） | `update_plan` |
| `Skill` 工具（调用技能） | 技能原生加载 —— 只需按照说明操作 |
| `Read`、`Write`、`Edit`（文件） | 使用你的原生文件工具 |
| `Bash`（运行命令） | 使用你的原生 shell 工具 |

## 子代理调度需要多代理支持

添加到你的 Codex 配置（`~/.codex/config.toml`）：

```toml
[features]
multi_agent = true
```

这启用了 `spawn_agent`、`wait` 和 `close_agent`，用于 `dispatching-parallel-agents` 和 `subagent-driven-development` 等技能。

## 命名代理调度

Claude Code 技能引用命名代理类型，如 `superpowers:code-reviewer`。
Codex 没有命名代理注册表 —— `spawn_agent` 从内置角色（`default`、`explorer`、`worker`）创建通用代理。

当技能说要调度命名代理类型时：

1. 找到代理的提示文件（例如 `agents/code-reviewer.md` 或技能的本地提示模板，如 `code-quality-reviewer-prompt.md`）
2. 读取提示内容
3. 填充任何模板占位符（`{BASE_SHA}`、`{WHAT_WAS_IMPLEMENTED}` 等）
4. 用填充内容作为 `message` 调度一个 `worker` 代理

| 技能指令 | Codex 等效工具 |
|-------------------|------------------|
| `Task tool (superpowers:code-reviewer)` | `spawn_agent(agent_type="worker", message=...)` 其中包含 `code-reviewer.md` 内容 |
| 带内联提示的 `Task tool (general-purpose)` | `spawn_agent(message=...)` 包含相同提示 |

### 消息框架

`message` 参数是用户级输入，不是系统提示。为最大程度遵循指令进行结构化：

```
Your task is to perform the following. Follow the instructions below exactly.

<agent-instructions>
[从代理的 .md 文件填充的提示内容]
</agent-instructions>

Execute this now. Output ONLY the structured response following the format
specified in the instructions above.
```

- 使用任务委托框架（"Your task is..."）而不是角色框架（"You are..."）
- 将指令包装在 XML 标签中 —— 模型将带标签的块视为权威
- 以明确的执行指令结束，以防止指令被总结

### 何时可以移除此解决方法

此方法补偿了 Codex 的插件系统尚未在 `plugin.json` 中支持 `agents` 字段。当 `RawPluginManifest` 获得 `agents` 字段时，插件可以 symlink 到 `agents/`（镜像现有的 `skills/` symlink），技能可以直接调度命名代理类型。

## 环境检测

创建工作树或完成分支的技能应在继续之前用只读 git 命令检测环境：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

- `GIT_DIR != GIT_COMMON` → 已在链接的工作树中（跳过创建）
- `BRANCH` 为空 → 分离的 HEAD（无法从沙盒分支/推送/PR）

参见 `using-git-worktrees` 步骤 0 和 `finishing-a-development-branch` 步骤 1，了解每个技能如何使用这些信号。

## Codex App 完成

当沙盒阻止分支/推送操作（外部管理工作树中的分离 HEAD）时，代理提交所有工作并告知用户使用 App 的原生控件：

- **"创建分支"** — 命名分支，然后通过 App UI 提交/推送/PR
- **"交接到本地"** — 将工作转移到用户的本地检出

代理仍可以运行测试、暂存文件，并输出建议的分支名称、提交消息和 PR 描述，供用户复制。