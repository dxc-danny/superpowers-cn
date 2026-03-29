# 零依赖头脑风暴服务器

用单一的零依赖 `server.js` 替换头脑风暴伴侣服务器的 vendored node_modules（express、ws、chokidar——714 个追踪文件），仅使用 Node.js 内置模块。

## 动机

将 node_modules vendoring 到 git 仓库会产生供应链风险：冻结的依赖不会获得安全补丁，714 个第三方代码文件未经审计就提交，vendor 代码的修改看起来像普通提交。虽然实际风险很低（仅限本地主机的开发服务器），但消除它很简单。

## 架构

一个单一的 `server.js` 文件（约 250-300 行），使用 `http`、`crypto`、`fs` 和 `path`。该文件承担两个角色：

- **直接运行时**（`node server.js`）：启动 HTTP/WebSocket 服务器
- **被 require 时**（`require('./server.js')`）：导出 WebSocket 协议函数用于单元测试

### WebSocket 协议

仅实现文本帧的 RFC 6455：

**握手：** 使用 SHA-1 + RFC 6455 魔数 GUID 从客户端的 `Sec-WebSocket-Key` 计算 `Sec-WebSocket-Accept`。返回 101 Switching Protocols。

**帧解码（客户端到服务器）：** 处理三种带掩码的长度编码：
- 小型：payload < 126 字节
- 中型：126-65535 字节（16 位扩展）
- 大型：> 65535 字节（64 位扩展）

使用 4 字节掩码密钥对 payload 进行 XOR 解掩码。返回 `{ opcode, payload, bytesConsumed }` 或在缓冲区不完整时返回 `null`。拒绝未掩码的帧。

**帧编码（服务器到客户端）：** 未掩码帧，相同的三种长度编码。

**处理的 Opcodes：** TEXT (0x01)、CLOSE (0x08)、PING (0x09)、PONG (0x0A)。无法识别的 opcodes 返回状态码 1003（Unsupported Data）的关闭帧。

**有意跳过：** 二进制帧、分片消息、扩展（permessage-deflate）、子协议。这些对于本地主机客户端之间的小型 JSON 文本消息是不必要的。扩展和子协议在握手中协商——通过不声明它们，它们永远不会激活。

**缓冲区累积：** 每个连接维护一个缓冲区。在 `data` 时追加并循环 `decodeFrame`，直到返回 null 或缓冲区为空。

### HTTP 服务器

三个路由：

1. **`GET /`** — 按 mtime 从屏幕目录提供最新的 `.html`。检测完整文档 vs 片段，将片段包装在框架模板中，注入 helper.js。返回 `text/html`。当没有 `.html` 文件存在时，提供带有注入 helper.js 的硬编码等待页面（"Waiting for Claude to push a screen..."）。
2. **`GET /files/*`** — 从屏幕目录提供静态文件，MIME 类型从硬编码的扩展名映射查找（html、css、js、png、jpg、gif、svg、json）。找不到时返回 404。
3. **其他所有请求** — 404。

WebSocket 升级通过 HTTP 服务器上的 `'upgrade'` 事件处理，与请求处理程序分开。

### 配置

环境变量（全部可选）：

- `BRAINSTORM_PORT` — 绑定端口（默认：随机高端口 49152-65535）
- `BRAINSTORM_HOST` — 绑定接口（默认：`127.0.0.1`）
- `BRAINSTORM_URL_HOST` — 启动 JSON 中 URL 的主机名（默认：当主机是 `127.0.0.1` 时为 `localhost`，否则与主机相同）
- `BRAINSTORM_DIR` — 屏幕目录路径（默认：`/tmp/brainstorm`）

### 启动顺序

1. 如果不存在则创建 `SCREEN_DIR`（递归 `mkdirSync`）
2. 从 `__dirname` 加载框架模板和 helper.js
3. 在配置的 host/port 上启动 HTTP 服务器
4. 启动 `fs.watch` 监听 `SCREEN_DIR`
5. 成功监听后，将 `server-started` JSON 记录到 stdout：`{ type, port, host, url_host, url, screen_dir }`
6. 将相同的 JSON 写入 `SCREEN_DIR/.server-info`，以便代理在 stdout 被隐藏时（后台执行）找到连接详情

### 应用级 WebSocket 消息

当从客户端收到 TEXT 帧时：

1. 解析为 JSON。如果解析失败，记录到 stderr 并继续。
2. 记录到 stdout 为 `{ source: 'user-event', ...event }`。
3. 如果事件包含 `choice` 属性，将 JSON 追加到 `SCREEN_DIR/.events`（每事件一行）。

### 文件监听

`fs.watch(SCREEN_DIR)` 取代 chokidar。在 HTML 文件事件上：

- 在新文件时（对存在的文件触发 `rename` 事件）：如果存在则删除 `.events` 文件（`unlinkSync`），将 `screen-added` 作为 JSON 记录到 stdout
- 在文件更改时（`change` 事件）：将 `screen-updated` 作为 JSON 记录到 stdout（**不要**清除 `.events`）
- 两个事件：向所有连接的 WebSocket 客户端发送 `{ type: 'reload' }`

按文件名进行防抖，超时约 100ms，以防止重复事件（在 macOS 和 Linux 上常见）。

### 错误处理

- 来自 WebSocket 客户端的格式错误的 JSON：记录到 stderr，继续
- 无法处理的 opcodes：关闭并返回状态 1003
- 客户端断开：从广播集中移除
- `fs.watch` 错误：记录到 stderr，继续
- 没有优雅关闭逻辑——shell 脚本通过 SIGTERM 处理进程生命周期

## 变更内容

| 之前 | 之后 |
|---|---|
| `index.js` + `package.json` + `package-lock.json` + 714 个 `node_modules` 文件 | `server.js`（单一文件） |
| express、ws、chokidar 依赖 | 无依赖 |
| 无静态文件服务 | `/files/*` 从屏幕目录服务 |

## 保持不变的内容

- `helper.js` — 无更改
- `frame-template.html` — 无更改
- `start-server.sh` — 单行更新：`index.js` → `server.js`
- `stop-server.sh` — 无更改
- `visual-companion.md` — 无更改
- 所有现有服务器行为和外部契约

## 平台兼容性

- `server.js` 仅使用跨平台 Node.js 内置模块
- `fs.watch` 在 macOS、Linux 和 Windows 上对单个扁平目录可靠
- Shell 脚本需要 bash（Windows 上的 Git Bash，这是 Claude Code 所需的）

## 测试

**单元测试**（`ws-protocol.test.js`）：通过 require `server.js` 导出直接测试 WebSocket 帧编码/解码、握手计算和协议边界情况。

**集成测试**（`server.test.js`）：测试完整服务器行为——HTTP 服务、WebSocket 通信、文件监听、头脑风暴工作流。使用 `ws` npm 包作为仅测试的客户端依赖（不向最终用户发货）。