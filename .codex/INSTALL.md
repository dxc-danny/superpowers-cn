# 为 Codex 安装超级能力

通过原生技能发现启用 Codex 中的超级能力技能。只需克隆并创建符号链接。

## 前置条件

- Git

## 安装

1. **克隆超级能力仓库：**
   ```bash
   git clone https://github.com/obra/superpowers.git ~/.codex/superpowers
   ```

2. **创建技能符号链接：**
   ```bash
   mkdir -p ~/.agents/skills
   ln -s ~/.codex/superpowers/skills ~/.agents/skills/superpowers
   ```

   **Windows (PowerShell):**
   ```powershell
   New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.agents\skills"
   cmd /c mklink /J "$env:USERPROFILE\.agents\skills\superpowers" "$env:USERPROFILE\.codex\superpowers\skills"
   ```

3. **重启 Codex**（退出并重新启动 CLI）以发现技能。

## 从旧引导迁移

如果你在原生技能发现之前安装了超级能力，你需要：

1. **更新仓库：**
   ```bash
   cd ~/.codex/superpowers && git pull
   ```

2. **创建技能符号链接**（上面的步骤2）— 这是新的发现机制。

3. **从 `~/.codex/AGENTS.md` 中删除旧的引导块** — 任何引用 `superpowers-codex bootstrap` 的块都不再需要。

4. **重启 Codex。**

## 验证

```bash
ls -la ~/.agents/skills/superpowers
```

你应该看到一个符号链接（在 Windows 上是 junction）指向你的超级能力技能目录。

## 更新

```bash
cd ~/.codex/superpowers && git pull
```

技能通过符号链接即时更新。

## 卸载

```bash
rm ~/.agents/skills/superpowers
```

也可以删除克隆的仓库：`rm -rf ~/.codex/superpowers`。
