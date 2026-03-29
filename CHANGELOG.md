# 更新日志

## [5.0.5] - 2026-03-17

### 修复

- **Brainstorm 服务器 ESM 修复**：将 `server.js` 重命名为 `server.cjs`，以便在 Node.js 22+ 上正确启动头脑风暴服务器，因为根目录的 `package.json` 中的 `"type": "module"` 导致 `require()` 失败。（PR #784 由 @sarbojitrana 提交，修复了 #774、#780、#783）
- **Windows 上的 Brainstorm 所有者 PID**：在 Windows/MSYS2 上跳过 `BRAINSTORM_OWNER_PID` 生命周期监控，因为 Node.js 无法看到 PID 命名空间。防止服务器在 60 秒后自终止。30 分钟的空闲超时仍然作为安全网。（#770，来自 PR #768 的文档由 @lucasyhzhu-debug 提交）
- **stop-server.sh 可靠性**：在实际报告成功之前验证服务器进程是否已死亡。最多等待 2 秒优雅关闭，升级到 `SIGKILL`，如果进程存活则报告失败。（#723）

### 更改

- **执行交接**：在计划编写后恢复用户在 subagent-driven-development 和 executing-plans 之间的选择。推荐使用 subagent-driven 但不再强制。（回退 `5e51c3e`）
