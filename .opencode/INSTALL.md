# 为 OpenCode 安装超级能力

## 前置条件

- 已安装 [OpenCode.ai](https://opencode.ai)

## 安装

将超级能力添加到你的 `opencode.json`（全局或项目级）的 `plugin` 数组中：

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git"]
}
```

重启 OpenCode。就是这样 — 插件会自动安装并注册所有技能。

通过询问来验证："Tell me about your superpowers"

## 从旧的基于符号链接的安装迁移

如果你之前使用 `git clone` 和符号链接安装了超级能力，请删除旧设置：

```bash
# 删除旧符号链接
rm -f ~/.config/opencode/plugins/superpowers.js
rm -rf ~/.config/opencode/skills/superpowers

# 也可以删除克隆的仓库
rm -rf ~/.config/opencode/superpowers

# 如果你为超级能力添加了 skills.paths，也要从 opencode.json 中删除
```

然后按照上面的安装步骤操作。

## 使用

使用 OpenCode 原生的 `skill` 工具：

```
use skill tool to list skills
use skill tool to load superpowers/brainstorming
```

## 更新

当你重启 OpenCode 时，超级能力会自动更新。

要固定特定版本：

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git#v5.0.3"]
}
```

## 故障排除

### 插件未加载

1. 检查日志：`opencode run --print-logs "hello" 2>&1 | grep -i superpowers`
2. 验证你的 `opencode.json` 中的插件行
3. 确保你运行的是最新版本的 OpenCode

### 技能未找到

1. 使用 `skill` 工具列出发现的内容
2. 检查插件是否正在加载（见上文）

### 工具映射

当技能引用 Claude Code 工具时：
- `TodoWrite` → `todowrite`
- 带子代理的 `Task` → `@mention` 语法
- `Skill` 工具 → OpenCode 原生的 `skill` 工具
- 文件操作 → 你的原生工具

## 获取帮助

- 报告问题：https://github.com/obra/superpowers/issues
- 完整文档：https://github.com/obra/superpowers/blob/main/docs/README.opencode.md
