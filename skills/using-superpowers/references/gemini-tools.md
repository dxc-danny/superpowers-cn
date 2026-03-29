# Gemini CLI 工具映射

技能使用 Claude Code 的工具名称。当你在技能中遇到这些时，使用你的平台等效工具：

| 技能引用 | Gemini CLI 等效工具 |
|-----------------|----------------------|
| `Read`（文件读取） | `read_file` |
| `Write`（文件创建） | `write_file` |
| `Edit`（文件编辑） | `replace` |
| `Bash`（运行命令） | `run_shell_command` |
| `Grep`（搜索文件内容） | `grep_search` |
| `Glob`（按名称搜索文件） | `glob` |
| `TodoWrite`（任务跟踪） | `write_todos` |
| `Skill` 工具（调用技能） | `activate_skill` |
| `WebSearch` | `google_web_search` |
| `WebFetch` | `web_fetch` |
| `Task` 工具（分派子代理） | 无等效 —— Gemini CLI 不支持子代理 |

## 无子代理支持

Gemini CLI 没有 Claude Code 的 `Task` 工具等效。依赖子代理分派的技能（`subagent-driven-development`、`dispatching-parallel-agents`）将通过 `executing-plans` 回退到单会话执行。

## 额外的 Gemini CLI 工具

这些工具在 Gemini CLI 中可用，但没有 Claude Code 等效：

| 工具 | 用途 |
|------|---------|
| `list_directory` | 列出文件和子目录 |
| `save_memory` | 跨会话持久化事实到 GEMINI.md |
| `ask_user` | 请求用户结构化输入 |
| `tracker_create_task` | 丰富的任务管理（创建、更新、列表、可视化） |
| `enter_plan_mode` / `exit_plan_mode` | 在进行更改之前切换到只读研究模式 |