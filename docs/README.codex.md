# Codex 超级能力

通过原生技能发现功能，将超级能力与 OpenAI Codex 结合使用的指南。

## 快速安装

告诉 Codex：

```
Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.codex/INSTALL.md
```

## 手动安装

### 前置条件

- OpenAI Codex CLI
- Git

### 步骤

1. 克隆仓库：
   ```bash
   git clone https://github.com/obra/superpowers.git ~/.codex/superpowers
   ```

2. 创建技能符号链接：
   ```bash
   mkdir -p ~/.agents/skills
   ln -s ~/.codex/superpowers/skills ~/.agents/skills/superpowers
   ```

3. 重启 Codex。

4. **对于子代理技能**（可选）：像 `dispatching-parallel-agents` 和 `subagent-driven-development` 这样的技能需要 Codex 的多代理功能。在 Codex 配置中添加：
   ```toml
   [features]
   multi_agent = true
   ```

### Windows

使用目录联接（junction）代替符号链接（无需开发者模式）：

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.agents\skills"
cmd /c mklink /J "$env:USERPROFILE\.agents\skills\superpowers" "$env:USERPROFILE\.codex\superpowers\skills"
```

## 工作原理

Codex 具有原生技能发现功能 — 它在启动时扫描 `~/.agents/skills/`，解析 SKILL.md 头信息，并根据需要加载技能。超级能力技能通过单一符号链接变得可见：

```
~/.agents/skills/superpowers/ → ~/.codex/superpowers/skills/
```

`using-superpowers` 技能会被自动发现，并强制执行技能使用规范 — 无需额外配置。

## 使用方法

技能会被自动发现。Codex 在以下情况下激活它们：
- 你提及某个技能名称（例如，"使用头脑风暴"）
- 任务与某个技能的描述匹配
- `using-superpowers` 技能指示 Codex 使用某个技能

### 个人技能

在 `~/.agents/skills/` 中创建你自己的技能：

```bash
mkdir -p ~/.agents/skills/my-skill
```

创建 `~/.agents/skills/my-skill/SKILL.md`：

```markdown
---
name: my-skill
description: Use when [condition] - [what it does]
---

# 我的技能

[你的技能内容]
```

`description` 字段是 Codex 决定何时自动激活技能的方式 — 请将其写为清晰的触发条件。

## 更新

```bash
cd ~/.codex/superpowers && git pull
```

技能会通过符号链接即时更新。

## 卸载

```bash
rm ~/.agents/skills/superpowers
```

**Windows (PowerShell):**
```powershell
Remove-Item "$env:USERPROFILE\.agents\skills\superpowers"
```

（可选）删除克隆的仓库：`rm -rf ~/.codex/superpowers`（Windows：`Remove-Item -Recurse -Force "$env:USERPROFILE\.codex\superpowers"`）。

## 故障排除

### 技能没有显示

1. 验证符号链接：`ls -la ~/.agents/skills/superpowers`
2. 检查技能是否存在：`ls ~/.codex/superpowers/skills`
3. 重启 Codex — 技能在启动时被发现

### Windows 目录联接问题

目录联接通常无需特殊权限即可工作。如果创建失败，请尝试以管理员身份运行 PowerShell。

## 获取帮助

- 报告问题：https://github.com/obra/superpowers/issues
- 主文档：https://github.com/obra/superpowers
