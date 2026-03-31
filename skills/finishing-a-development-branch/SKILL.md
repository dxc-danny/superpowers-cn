---
name: finishing-a-development-branch
description: Use when implementation is complete, all tests pass, and you need to decide how to integrate the work - guides completion of development work by presenting structured options for merge, PR, or cleanup
---

# 完成开发分支

## 概述

通过呈现清晰的选项并处理所选工作流来引导开发工作的完成。

**核心原则：** 验证测试 → 呈现选项 → 执行选择 → 清理。

**开始时宣布：** "我正在使用 finishing-a-development-branch 技能来完成这项工作。"

## 流程

### 步骤 1：验证测试

**在呈现选项之前，验证测试通过：**

```bash
# 运行项目的测试套件
npm test / cargo test / pytest / go test ./...
```

**如果测试失败：**
```
测试失败（<N> 个失败）。必须在完成前修复：

[显示失败]

在测试通过之前不能进行 merge/PR。
```

停止。不要继续到步骤 2。

**如果测试通过：** 继续到步骤 2。

### 步骤 2：确定基础分支

```bash
# 尝试常见基础分支
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

或询问："此分支从 main 分叉 —— 这样正确吗？"

### 步骤 3：呈现选项

准确呈现这 4 个选项：

```
实施完成。你想怎么做？

1. 在本地 merge 回 <base-branch>
2. 推送并创建 Pull Request
3. 保持现状（我稍后处理）
4. 丢弃此工作

选择哪个？
```

**不要添加解释** —— 保持选项简洁。

### 步骤 4：执行选择

#### 选项 1：本地 Merge

```bash
# 切换到基础分支
git checkout <base-branch>

# 拉取最新
git pull

# Merge 功能分支
git merge <feature-branch>

# 在合并结果上验证测试
<test command>

# 如果测试通过
git branch -d <feature-branch>
```

然后：清理 worktree（步骤 5）

#### 选项 2：推送并创建 PR

```bash
# 推送分支
git push -u origin <feature-branch>

# 创建 PR
gh pr create --title "<title>" --body "$(cat <<'EOF'
## 摘要
<2-3 条变更内容>

## 测试计划
- [ ] <验证步骤>
EOF
)"
```

然后：清理 worktree（步骤 5）

#### 选项 3：保持现状

报告："保持分支 <name>。Worktree 保留在 <path>。"

**不要清理 worktree。**

#### 选项 4：丢弃

**先确认：**
```
这将永久删除：
- 分支 <name>
- 所有提交：<commit-list>
- <path> 处的 worktree

输入 'discard' 确认。
```

等待精确确认。

如果确认：
```bash
git checkout <base-branch>
git branch -D <feature-branch>
```

然后：清理 worktree（步骤 5）

### 步骤 5：清理 Worktree

**对于选项 1、2、4：**

检查是否在 worktree 中：
```bash
git worktree list | grep $(git branch --show-current)
```

如果是：
```bash
git worktree remove <worktree-path>
```

**对于选项 3：** 保留 worktree。

## 快速参考

| 选项 | Merge | Push | 保留 Worktree | 清理分支 |
|--------|-------|------|---------------|----------------|
| 1. 本地 Merge | ✓ | - | - | ✓ |
| 2. 创建 PR | - | ✓ | ✓ | - |
| 3. 保持现状 | - | - | ✓ | - |
| 4. 丢弃 | - | - | - | ✓ (强制) |

## 常见错误

**跳过测试验证**
- **问题：** 合并损坏的代码，创建失败的 PR
- **修复：** 在提供选项前始终验证测试

**开放式问题**
- **问题：** "接下来我该做什么？" → 模糊
- **修复：** 准确呈现 4 个结构化选项

**自动清理 worktree**
- **问题：** 在可能需要时删除 worktree（选项 2、3）
- **修复：** 仅对选项 1 和 4 清理

**丢弃前不确认**
- **问题：** 意外删除工作
- **修复：** 要求输入 "discard" 确认

## 红旗

**绝不：**
- 在测试失败的情况下继续
- 在结果上不验证测试就合并
- 不确认就删除工作
- 未经明确请求就 force-push

**始终：**
- 在提供选项前验证测试
- 准确呈现 4 个选项
- 选项 4 要获得输入确认
- 仅对选项 1 和 4 清理 worktree

## 集成

**被调用：**
- **subagent-driven-development**（步骤 7）—— 所有任务完成后
- **executing-plans**（步骤 5）—— 所有批次完成后

**配合：**
- **using-git-worktrees** —— 清理该技能创建的 worktree
