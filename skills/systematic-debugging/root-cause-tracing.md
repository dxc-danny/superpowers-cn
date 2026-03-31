# 根因追踪

## 概述

Bug 经常在调用栈深处显现（git init 在错误的目录、文件创建在错误的位置、数据库用错误的路径打开）。你的本能是在错误出现的地方修复，但那只是治疗症状。

**核心原则：** 沿着调用链向后追踪，直到找到原始触发点，然后在源头修复。

## 何时使用

```dot
digraph when_to_use {
    "Bug 在调用栈深处出现？" [shape=diamond];
    "能向后追踪？" [shape=diamond];
    "在症状点修复" [shape=box];
    "追踪到原始触发点" [shape=box];
    "更好：还要添加纵深防御" [shape=box];

    "Bug 在调用栈深处出现？" -> "能向后追踪？" [label="yes"];
    "能向后追踪？" -> "追踪到原始触发点" [label="yes"];
    "能向后追踪？" -> "在症状点修复" [label="no - dead end"];
    "追踪到原始触发点" -> "更好：还要添加纵深防御";
}
```

**使用场景：**
- 错误发生在执行深处（不在入口点）
- 堆栈跟踪显示很长的调用链
- 不清楚无效数据来自哪里
- 需要找到哪个测试/代码触发了问题

## 追踪过程

### 1. 观察症状
```
Error: git init failed in /Users/jesse/project/packages/core
```

### 2. 找到直接原因
**什么代码直接导致这个？**
```typescript
await execFileAsync('git', ['init'], { cwd: projectDir });
```

### 3. 问：谁调用了这个？
```typescript
WorktreeManager.createSessionWorktree(projectDir, sessionId)
  → 由 Session.initializeWorkspace() 调用
  → 由 Session.create() 调用
  → 由测试中的 Project.create() 调用
```

### 4. 继续向上追踪
**传递了什么值？**
- `projectDir = ''` (空字符串！)
- 空字符串作为 `cwd` 解析为 `process.cwd()`
- 那就是源代码目录！

### 5. 找到原始触发点
**空字符串从哪里来？**
```typescript
const context = setupCoreTest(); // 返回 { tempDir: '' }
Project.create('name', context.tempDir); // 在 beforeEach 之前访问！
```

## 添加堆栈跟踪

当你无法手动追踪时，添加 instrumentation：

```typescript
// 在有问题的操作之前
async function gitInit(directory: string) {
  const stack = new Error().stack;
  console.error('DEBUG git init:', {
    directory,
    cwd: process.cwd(),
    nodeEnv: process.env.NODE_ENV,
    stack,
  });

  await execFileAsync('git', ['init'], { cwd: directory });
}
```

**关键：** 在测试中使用 `console.error()`（不是 logger - 可能不显示）

**运行并捕获：**
```bash
npm test 2>&1 | grep 'DEBUG git init'
```

**分析堆栈跟踪：**
- 查找测试文件名
- 找到触发调用的行号
- 识别模式（同一个测试？同一个参数？）

## 找到哪个测试导致污染

如果某些东西在测试中出现但不知道是哪个测试：

使用本目录中的二分脚本 `find-polluter.sh`：

```bash
./find-polluter.sh '.git' 'src/**/*.test.ts'
```

逐个运行测试，在第一个污染者处停止。使用方法见脚本。

## 真实示例：空的 projectDir

**症状：** `.git` 创建在 `packages/core/`（源代码）中

**追踪链：**
1. `git init` 在 `process.cwd()` 运行 ← 空 cwd 参数
2. WorktreeManager 用空的 projectDir 调用
3. Session.create() 传递了空字符串
4. 测试在 beforeEach 之前访问了 `context.tempDir`
5. setupCoreTest() 最初返回 `{ tempDir: '' }`

**根本原因：** 顶层变量初始化访问了空值

**修复：** 将 tempDir 改为 getter，在 beforeEach 之前访问时抛出异常

**还添加了纵深防御：**
- 第一层：Project.create() 验证目录
- 第二层：WorkspaceManager 验证非空
- 第三层：NODE_ENV 防护拒绝在 tmpdir 之外进行 git init
- 第四层：git init 前的堆栈跟踪日志

## 关键原则

```dot
digraph principle {
    "找到直接原因" [shape=ellipse];
    "能向上一级追踪？" [shape=diamond];
    "向后追踪" [shape=box];
    "这是源头？" [shape=diamond];
    "在源头修复" [shape=box];
    "在每层添加验证" [shape=box];
    "Bug 不可能发生" [shape=doublecircle];
    "绝不只是修复症状" [shape=octagon, style=filled, fillcolor=red, fontcolor=white];

    "找到直接原因" -> "能向上一级追踪？";
    "能向上一级追踪？" -> "向后追踪" [label="yes"];
    "能向上一级追踪？" -> "绝不只是修复症状" [label="no"];
    "向后追踪" -> "这是源头？";
    "这是源头？" -> "向后追踪" [label="no - keeps going"];
    "这是源头？" -> "在源头修复" [label="yes"];
    "在源头修复" -> "在每层添加验证";
    "在每层添加验证" -> "Bug 不可能发生";
}
```

**永远不要只修复错误出现的地方。** 追踪回去找到原始触发点。

## 堆栈跟踪技巧

**在测试中：** 使用 `console.error()` 而不是 logger - logger 可能被抑制
**在操作之前：** 在危险操作之前记录，而不是之后失败时
**包含上下文：** 目录、cwd、环境变量、时间戳
**捕获堆栈：** `new Error().stack` 显示完整调用链

## 实际影响

来自调试会话（2025-10-03）：
- 通过 5 层追踪找到根本原因
- 在源头修复（getter 验证）
- 添加了 4 层防御
- 1847 个测试通过，零污染