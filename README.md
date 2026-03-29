# 超级能力

超级能力（Superpowers）是一套完整的软件开发生工作流，专为你的编码代理设计。它基于一组可组合的"技能"和一些初始指令，确保你的代理能够正确使用它们。

## 工作原理

从你启动编码代理的那一刻起，它就开始工作了。当你发现自己在构建某个东西时，它**不会**直接跳进去写代码。相反，它会退后一步，问你真正想要做什么。

一旦从对话中提炼出规范，它会将内容分成足够小且易于阅读和理解的块展示给你。

在你批准设计后，你的代理会整理出一个足够清晰的实现计划，即使是给一个缺乏品味、没有判断力、没有项目上下文、厌恶测试的热情初级工程师也能follow。它强调真正的红/绿 TDD、YAGNI（你不需要它）和 DRY。

接下来，当你 say "go" 时，它会启动一个**子代理驱动开发**流程，让代理完成每个工程任务，检查和审查他们的工作，并持续推进。通常情况下，Claude 能够自主工作几个小时而不偏离你制定的计划。

还有很多内容，但这是系统的核心。由于技能是自动触发的，你不需要做任何特殊的事情。你的编码代理本身就拥有超级能力。

## 赞助

如果超级能力帮助你赚到了钱，我非常感激你考虑[赞助我的开源工作](https://github.com/sponsors/obra)。

谢谢！

- Jesse

## 安装

**注意：** 安装方式因平台而异。Claude Code 或 Cursor 有内置的插件市场。Codex 和 OpenCode 需要手动设置。

### Claude Code 官方市场

超级能力可以通过[官方 Claude 插件市场](https://claude.com/plugins/superpowers)获取。

从 Claude 市场安装插件：

```bash
/plugin install superpowers@claude-plugins-official
```

### Claude Code（通过插件市场）

在 Claude Code 中，首先注册市场：

```bash
/plugin marketplace add obra/superpowers-marketplace
```

然后从这个市场安装插件：

```bash
/plugin install superpowers@superpowers-marketplace
```

### Cursor（通过插件市场）

在 Cursor Agent 聊天中，从市场安装：

```text
/add-plugin superpowers
```

或在插件市场中搜索"superpowers"。

### Codex

告诉 Codex：

```
Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.codex/INSTALL.md
```

**详细文档：** [docs/README.codex.md](docs/README.codex.md)

### OpenCode

告诉 OpenCode：

```
Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.opencode/INSTALL.md
```

**详细文档：** [docs/README.opencode.md](docs/README.opencode.md)

### Gemini CLI

```bash
gemini extensions install https://github.com/obra/superpowers
```

更新：

```bash
gemini extensions update superpowers
```

### 验证安装

在你选择的平台开始一个新会话，并询问一些应该触发技能的事情（例如"帮我计划这个功能"或"让我们调试这个问题"）。代理应该自动调用相关的超级能力技能。

## 基本工作流

1. **brainstorming** - 在写代码之前激活。通过问题细化粗糙的想法，探索替代方案，分节展示设计以供验证。保存设计文档。

2. **using-git-worktrees** - 设计批准后激活。在新分支上创建隔离的工作空间，运行项目设置，验证干净的测试基线。

3. **writing-plans** - 设计批准后激活。将工作分解为大小适中的任务（每个2-5分钟）。每个任务都有精确的文件路径、完整的代码和验证步骤。

4. **subagent-driven-development** 或 **executing-plans** - 计划激活后激活。为每个任务分配新的子代理进行两阶段审查（规范合规性，然后是代码质量），或者批量执行并在人工检查点停止。

5. **test-driven-development** - 在实现期间激活。强制执行 RED-GREEN-REFACTOR：写失败的测试，看它失败，写最少的代码，看它通过，提交。删除在测试之前写的代码。

6. **requesting-code-review** - 在任务之间激活。根据计划审查，按严重程度报告问题。关键问题阻止进度。

7. **finishing-a-development-branch** - 任务完成后激活。验证测试，提供选项（合并/PR/保留/丢弃），清理工作树。

**代理在任何任务之前都会检查相关技能。** 这是强制性的工作流，不是建议。

## 内容概览

### 技能库

**测试**
- **test-driven-development** - RED-GREEN-REFACTOR 周期（包括测试反模式参考）

**调试**
- **systematic-debugging** - 四阶段根因流程（包括根因追踪、深度防御、基于条件的等待技术）
- **verification-before-completion** - 确保它真的被修复了

**协作**
- **brainstorming** - 苏格拉底式设计细化
- **writing-plans** - 详细的实现计划
- **executing-plans** - 带检查点的批量执行
- **dispatching-parallel-agents** - 并发子代理工作流
- **requesting-code-review** - 预审清单
- **receiving-code-review** - 响应反馈
- **using-git-worktrees** - 并行开发分支
- **finishing-a-development-branch** - 合并/PR 决策工作流
- **subagent-driven-development** - 快速迭代与两阶段审查（规范合规性，然后是代码质量）

**元技能**
- **writing-skills** - 按照最佳实践创建新技能（包括测试方法论）
- **using-superpowers** - 技能系统介绍

## 理念

- **测试驱动开发** - 始终先写测试
- **系统化而非临时** - 流程优先于猜测
- **复杂性简化** - 简单性作为主要目标
- **证据胜于声明** - 在宣布成功之前验证

阅读更多：[Superpowers for Claude Code](https://blog.fsck.com/2025/10/09/superpowers/)

## 贡献

技能直接存放在这个仓库中。贡献方式：

1. Fork 仓库
2. 为你的技能创建一个分支
3. 按照 `writing-skills` 技能创建和测试新技能
4. 提交 PR

参见 `skills/writing-skills/SKILL.md` 获取完整指南。

## 更新

当你更新插件时，技能会自动更新：

```bash
/plugin update superpowers
```

## 许可证

MIT 许可证 - 参见 LICENSE 文件了解详情

## 社区

超级能力由 [Jesse Vincent](https://blog.fsck.com) 和 [Prime Radiant](https://primeradiant.com) 的其他人构建。

如需社区支持、问题解答和分享你用超级能力构建的内容，请加入 [Discord](https://discord.gg/Jd8Vphy9jq)。

## 支持

- **Discord**: [加入我们的 Discord](https://discord.gg/Jd8Vphy9jq)
- **问题**: https://github.com/obra/superpowers/issues
- **市场**: https://github.com/obra/superpowers-marketplace
