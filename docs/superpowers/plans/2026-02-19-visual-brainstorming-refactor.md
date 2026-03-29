# 视觉头脑风暴重构实现计划

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**目标：** 将视觉头脑风暴从阻塞式 TUI 反馈模型重构为非阻塞的"浏览器显示、终端命令"架构。

**架构：** 浏览器成为交互式显示；终端保持会话通道。服务器将用户事件写入每个屏幕的 `.events` 文件，Claude 在下一轮读取它。消除 `wait-for-feedback.sh` 和所有 `TaskOutput` 阻塞。

**技术栈：** Node.js (Express, ws, chokidar), vanilla HTML/CSS/JS

**规范：** `docs/superpowers/specs/2026-02-19-visual-brainstorming-refactor-design.md`

---

## 文件映射

| 文件 | 操作 | 职责 |
|------|------|------|
| `lib/brainstorm-server/index.js` | 修改 | 服务器：添加 `.events` 文件写入、新屏幕时清除、替换 `wrapInFrame` |
| `lib/brainstorm-server/frame-template.html` | 修改 | 模板：移除反馈页脚、添加内容占位符 + 选择指示器 |
| `lib/brainstorm-server/helper.js` | 修改 | 客户端 JS：移除 send/feedback 函数、收窄到点击捕获 + 指示器更新 |
| `lib/brainstorm-server/wait-for-feedback.sh` | 删除 | 不再需要 |
| `skills/brainstorming/visual-companion.md` | 修改 | 技能说明：重写循环为非阻塞流程 |
| `tests/brainstorm-server/server.test.js` | 修改 | 测试：为新模板结构和 helper.js API 更新测试 |

---

## 块 1：服务器、模板、客户端、测试、技能

### 任务 1：更新 `frame-template.html`

**文件：**
- 修改：`lib/brainstorm-server/frame-template.html`

- [ ] **步骤 1：移除反馈页脚 HTML**

将 feedback-footer div（第 227-233 行）替换为选择指示器栏：

```html
  <div class="indicator-bar">
    <span id="indicator-text">点击上方选项，然后返回终端</span>
  </div>
```

同时将 `#claude-content` 内的默认内容（第 220-223 行）替换为内容占位符：

```html
    <div id="claude-content">
      <!-- CONTENT -->
    </div>
```

- [ ] **步骤 2：用指示器栏 CSS 替换反馈页脚 CSS**

移除 `.feedback-footer`、`.feedback-footer label`、`.feedback-row` 以及 `.feedback-footer` 内的 textarea/button 样式（第 82-112 行）。

添加指示器栏 CSS：

```css
    .indicator-bar {
      background: var(--bg-secondary);
      border-top: 1px solid var(--border);
      padding: 0.5rem 1.5rem;
      flex-shrink: 0;
      text-align: center;
    }
    .indicator-bar span {
      font-size: 0.75rem;
      color: var(--text-secondary);
    }
    .indicator-bar .selected-text {
      color: var(--accent);
      font-weight: 500;
    }
```

- [ ] **步骤 3：验证模板渲染**

运行测试套件检查模板是否仍能加载：
```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：测试 1-5 应仍然通过。测试 6-8 可能失败（预期 —— 它们断言旧结构）。

- [ ] **步骤 4：提交**

```bash
git add lib/brainstorm-server/frame-template.html
git commit -m "Replace feedback footer with selection indicator bar in brainstorm template"
```

---

### 任务 2：更新 `index.js` — 内容注入和 `.events` 文件

**文件：**
- 修改：`lib/brainstorm-server/index.js`

- [ ] **步骤 1：编写失败的测试用于 `.events` 文件写入**

在 `tests/brainstorm-server/server.test.js` 中 Test 4 区域之后添加 —— 一个发送带 `choice` 字段的 WebSocket 事件并验证 `.events` 文件被写入的新测试：

```javascript
    // Test: Choice events written to .events file
    console.log('Test: Choice events written to .events file');
    const ws3 = new WebSocket(`ws://localhost:${TEST_PORT}`);
    await new Promise(resolve => ws3.on('open', resolve));

    ws3.send(JSON.stringify({ type: 'click', choice: 'a', text: 'Option A' }));
    await sleep(300);

    const eventsFile = path.join(TEST_DIR, '.events');
    assert(fs.existsSync(eventsFile), '.events file should exist after choice click');
    const lines = fs.readFileSync(eventsFile, 'utf-8').trim().split('\n');
    const event = JSON.parse(lines[lines.length - 1]);
    assert.strictEqual(event.choice, 'a', 'Event should contain choice');
    assert.strictEqual(event.text, 'Option A', 'Event should contain text');
    ws3.close();
    console.log('  PASS');
```

- [ ] **步骤 2：运行测试验证它失败**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：新测试失败 — `.events` 文件尚不存在。

- [ ] **步骤 3：编写失败的测试用于 `.events` 文件在新屏幕时清除**

添加另一个测试：

```javascript
    // Test: .events cleared on new screen
    console.log('Test: .events cleared on new screen');
    // .events file should still exist from previous test
    assert(fs.existsSync(path.join(TEST_DIR, '.events')), '.events should exist before new screen');
    fs.writeFileSync(path.join(TEST_DIR, 'new-screen.html'), '<h2>New screen</h2>');
    await sleep(500);
    assert(!fs.existsSync(path.join(TEST_DIR, '.events')), '.events should be cleared after new screen');
    console.log('  PASS');
```

- [ ] **步骤 4：运行测试验证它失败**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：新测试失败 — `.events` 在屏幕推送时未清除。

- [ ] **步骤 5：在 `index.js` 中实现 `.events` 文件写入**

在 WebSocket `message` 处理器（第 74-77 行）中，在 `console.log` 之后添加：

```javascript
    // Write user events to .events file for Claude to read
    if (event.choice) {
      const eventsFile = path.join(SCREEN_DIR, '.events');
      fs.appendFileSync(eventsFile, JSON.stringify(event) + '\n');
    }
```

在 chokidar `add` 处理器（第 104-111 行）中，添加 `.events` 清除：

```javascript
    if (filePath.endsWith('.html')) {
      // Clear events from previous screen
      const eventsFile = path.join(SCREEN_DIR, '.events');
      if (fs.existsSync(eventsFile)) fs.unlinkSync(eventsFile);

      console.log(JSON.stringify({ type: 'screen-added', file: filePath }));
      // ... existing reload broadcast
    }
```

- [ ] **步骤 6：用注释占位符注入替换 `wrapInFrame`**

替换 `wrapInFrame` 函数（第 27-32 行）：

```javascript
function wrapInFrame(content) {
  return frameTemplate.replace('<!-- CONTENT -->', content);
}
```

- [ ] **步骤 7：运行所有测试**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：新的 `.events` 测试通过。现有测试可能仍有来自旧断言的失败（在任务 4 中修复）。

- [ ] **步骤 8：提交**

```bash
git add lib/brainstorm-server/index.js tests/brainstorm-server/server.test.js
git commit -m "Add .events file writing and comment-based content injection to brainstorm server"
```

---

### 任务 3：简化 `helper.js`

**文件：**
- 修改：`lib/brainstorm-server/helper.js`

- [ ] **步骤 1：移除 `sendToClaude` 函数**

删除 `sendToClaude` 函数（第 92-106 行）—— 函数体和页面接管 HTML。

- [ ] **步骤 2：移除 `window.send` 函数**

删除 `window.send` 函数（第 120-129 行）—— 与已移除的 Send 按钮关联。

- [ ] **步骤 3：移除表单提交和输入更改处理器**

删除表单提交处理器（第 57-71 行）和输入更改处理器（第 73-89 行），包括 `inputTimeout` 变量。

- [ ] **步骤 4：移除 `pageshow` 事件监听器**

删除我们之前添加的 `pageshow` 监听器（不再有 textarea 可清除）。

- [ ] **步骤 5：将点击处理器收窄到只处理 `[data-choice]`**

替换点击处理器（第 36-55 行）为更窄的版本：

```javascript
  // Capture clicks on choice elements
  document.addEventListener('click', (e) => {
    const target = e.target.closest('[data-choice]');
    if (!target) return;

    sendEvent({
      type: 'click',
      text: target.textContent.trim(),
      choice: target.dataset.choice,
      id: target.id || null
    });
  });
```

- [ ] **步骤 6：在选择点击时添加指示器栏更新**

在点击处理器中的 `sendEvent` 调用之后添加：

```javascript
    // Update indicator bar
    const indicator = document.getElementById('indicator-text');
    if (indicator) {
      const label = target.querySelector('h3, .content h3, .card-body h3')?.textContent?.trim() || target.dataset.choice;
      indicator.innerHTML = '<span class="selected-text">' + label + ' selected</span> — return to terminal to continue';
    }
```

- [ ] **步骤 7：从 `window.brainstorm` API 中移除 `sendToClaude`**

更新 `window.brainstorm` 对象（第 132-136 行）以移除 `sendToClaude`：

```javascript
  window.brainstorm = {
    send: sendEvent,
    choice: (value, metadata = {}) => sendEvent({ type: 'choice', value, ...metadata })
  };
```

- [ ] **步骤 8：运行测试**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```

- [ ] **步骤 9：提交**

```bash
git add lib/brainstorm-server/helper.js
git commit -m "Simplify helper.js: remove feedback functions, narrow to choice capture + indicator"
```

---

### 任务 4：为新结构更新测试

**文件：**
- 修改：`tests/brainstorm-server/server.test.js`

**注意：** 以下行号引用来自 _原始_ 文件。任务 2 在文件中更早插入新测试，因此实际行号会偏移。通过它们的 `console.log` 标签查找测试（例如 "Test 5:", "Test 6:")。

- [ ] **步骤 1：更新 Test 5（完整文档断言）**

找到 Test 5 断言 `!fullRes.body.includes('feedback-footer')`。将其更改为：完整文档也不应该有指示器栏（它们按原样提供）：

```javascript
    assert(!fullRes.body.includes('indicator-bar') || fullDoc.includes('indicator-bar'),
      'Should not wrap full documents in frame template');
```

- [ ] **步骤 2：更新 Test 6（片段包装）**

第 125 行：将 `feedback-footer` 断言替换为指示器栏断言：

```javascript
    assert(fragRes.body.includes('indicator-bar'), 'Fragment should get indicator bar from frame');
```

同时验证内容占位符被替换（片段内容出现，占位符注释不出现）：

```javascript
    assert(!fragRes.body.includes('<!-- CONTENT -->'), 'Content placeholder should be replaced');
```

- [ ] **步骤 3：更新 Test 7（helper.js API）**

第 140-142 行：更新断言以反映新的 API 表面：

```javascript
    assert(helperContent.includes('toggleSelect'), 'helper.js should define toggleSelect');
    assert(helperContent.includes('sendEvent'), 'helper.js should define sendEvent');
    assert(helperContent.includes('selectedChoice'), 'helper.js should track selectedChoice');
    assert(helperContent.includes('brainstorm'), 'helper.js should expose brainstorm API');
    assert(!helperContent.includes('sendToClaude'), 'helper.js should not contain sendToClaude');
```

- [ ] **步骤 4：用指示器栏测试替换 Test 8（sendToClaude 主题）**

替换 Test 8（第 145-149 行）—— `sendToClaude` 不再存在。改为测试指示器栏：

```javascript
    // Test 8: Indicator bar uses CSS variables (theme support)
    console.log('Test 8: Indicator bar uses CSS variables');
    const templateContent = fs.readFileSync(
      path.join(__dirname, '../../lib/brainstorm-server/frame-template.html'), 'utf-8'
    );
    assert(templateContent.includes('indicator-bar'), 'Template should have indicator bar');
    assert(templateContent.includes('indicator-text'), 'Template should have indicator text element');
    console.log('  PASS');
```

- [ ] **步骤 5：运行完整测试套件**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：所有测试通过。

- [ ] **步骤 6：提交**

```bash
git add tests/brainstorm-server/server.test.js
git commit -m "Update brainstorm server tests for new template structure and helper.js API"
```

---

### 任务 5：删除 `wait-for-feedback.sh`

**文件：**
- 删除：`lib/brainstorm-server/wait-for-feedback.sh`

- [ ] **步骤 1：验证没有其他文件导入或引用 `wait-for-feedback.sh`**

搜索代码库：
```bash
grep -r "wait-for-feedback" /Users/drewritter/prime-rad/superpowers/ --include="*.js" --include="*.md" --include="*.sh" --include="*.json"
```

预期引用：只有 `visual-companion.md`（在任务 6 中重写）和可能的发布说明（历史记录，按原样保留）。

- [ ] **步骤 2：删除文件**

```bash
rm lib/brainstorm-server/wait-for-feedback.sh
```

- [ ] **步骤 3：运行测试确认没有破坏**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：所有测试通过（没有测试引用此文件）。

- [ ] **步骤 4：提交**

```bash
git add -u lib/brainstorm-server/wait-for-feedback.sh
git commit -m "Delete wait-for-feedback.sh: replaced by .events file"
```

---

### 任务 6：重写 `visual-companion.md`

**文件：**
- 修改：`skills/brainstorming/visual-companion.md`

- [ ] **步骤 1：更新 "How It Works" 描述（第 18 行）**

将关于接收反馈 "as JSON" 的句子替换为：

```markdown
服务器监视目录中的 HTML 文件并将最新的一个提供给浏览器。你编写 HTML 内容，用户在浏览器中看到它并可以点击选择选项。选择被记录到 `.events` 文件，你可以在下一轮读取它。
```

- [ ] **步骤 2：更新片段描述（第 20 行）**

从帧模板提供的内容描述中移除 "feedback footer"：

```markdown
**内容片段 vs 完整文档：** 如果你的 HTML 文件以 `<!DOCTYPE` 或 `<html` 开头，服务器按原样提供服务（仅注入 helper 脚本）。否则，服务器会自动将你的内容包装在帧模板中 —— 添加页眉、CSS 主题、选择指示器和所有交互式基础设施。**默认编写内容片段。** 只有当你需要完全控制页面时才编写完整文档。
```

- [ ] **步骤 3：重写 "The Loop" 部分（第 36-61 行）**

用以下内容替换整个 "The Loop" 部分：

```markdown
## The Loop

1. **编写 HTML** 到 `screen_dir` 中的新文件：
   - 使用语义文件名：`platform.html`、`visual-style.html`、`layout.html`
   - **不要重用文件名** — 每个屏幕获得一个新鲜文件
   - 使用 Write 工具 — **不要使用 cat/heredoc**（会将噪声倒入终端）
   - 服务器自动提供最新文件

2. **告诉用户期望什么并结束你的回合：**
   - 提醒他们 URL（每一步，不只是第一步）
   - 简要总结屏幕上显示的内容（例如，"显示主页的 3 个布局选项"）
   - 让他们在终端中回复："看一下然后告诉我你的想法。如果你愿意，点击选择一个选项。"

3. **在你的下一轮** — 用户在终端回复后：
   - 读取 `$SCREEN_DIR/.events`（如果存在）—— 这包含用户的浏览器交互（点击、选择）作为 JSON 行
   - 与用户的终端文本合并以获得完整画面
   - 终端消息是主要反馈；`.events` 提供结构化交互数据

4. **迭代或推进** — 如果反馈更改当前屏幕，编写一个新文件（例如 `layout-v2.html`）。只有当当前步骤被验证后才进入下一个问题。

5. 重复直到完成。
```

- [ ] **步骤 4：替换 "User Feedback Format" 部分（第 165-174 行）**

用以下内容替换：

```markdown
## 浏览器事件格式

当用户在浏览器中点击选项时，它们的交互被记录到 `$SCREEN_DIR/.events`（每行一个 JSON 对象）。当你推送新屏幕时文件自动清除。

```jsonl
{"type":"click","choice":"a","text":"Option A - Simple Layout","timestamp":1706000101}
{"type":"click","choice":"c","text":"Option C - Complex Grid","timestamp":1706000108}
{"type":"click","choice":"b","text":"Option B - Hybrid","timestamp":1706000115}
```

完整的事件流显示用户的探索路径 —— 它们可能在确定之前点击多个选项。最后的 `choice` 事件通常是最终选择，但点击模式可以揭示犹豫或值得询问的偏好。

如果 `.events` 不存在，用户没有与浏览器交互 —— 只使用他们的终端文本。
```

- [ ] **步骤 5：更新 "Writing Content Fragments" 描述（第 65 行）**

移除 "feedback footer" 引用：

```markdown
只编写页面内部的内容。服务器会自动将其包装在帧模板中（页眉、主题 CSS、选择指示器和所有交互式基础设施）。
```

- [ ] **步骤 6：更新 Reference 部分（第 200-203 行）**

移除关于 "JS API" 的 helper.js 引用描述 —— API 现在很简单。保留路径引用：

```markdown
## Reference

- Frame template (CSS reference): `${CLAUDE_PLUGIN_ROOT}/lib/brainstorm-server/frame-template.html`
- Helper script (client-side): `${CLAUDE_PLUGIN_ROOT}/lib/brainstorm-server/helper.js`
```

- [ ] **步骤 7：提交**

```bash
git add skills/brainstorming/visual-companion.md
git commit -m "Rewrite visual-companion.md for non-blocking browser-displays-terminal-commands flow"
```

---

### 任务 7：最终验证

- [ ] **步骤 1：运行完整测试套件**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：所有测试通过。

- [ ] **步骤 2：手动烟雾测试**

手动启动服务器并验证流程端到端工作：

```bash
cd /Users/drewritter/prime-rad/superpowers && lib/brainstorm-server/start-server.sh --project-dir /tmp/brainstorm-smoke-test
```

编写测试片段、在浏览器中打开、点击选项、验证 `.events` 文件被写入、验证指示器栏更新。然后停止服务器：

```bash
lib/brainstorm-server/stop-server.sh <screen_dir from start output>
```

- [ ] **步骤 3：验证没有陈旧引用保留**

```bash
grep -r "wait-for-feedback\|sendToClaude\|feedback-footer\|send-to-claude\|TaskOutput.*block.*true" /Users/drewritter/prime-rad/superpowers/ --include="*.js" --include="*.md" --include="*.sh" --include="*.html" | grep -v node_modules | grep -v RELEASE-NOTES | grep -v "\.md:.*spec\|plan"
```

预期：发布说明和 spec/plan 文档之外没有匹配（它们是历史记录）。

- [ ] **步骤 4：如需清理则最终提交**

```bash
git status
# Review untracked/modified files, stage specific files as needed, commit if clean
```