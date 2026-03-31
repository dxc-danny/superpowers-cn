---
name: requesting-code-review
description: Use when completing tasks, implementing major features, or before merging to verify work meets requirements
---

# 请求代码审查

调度 superpowers:code-reviewer 子代理，在问题扩散之前捕获问题。审查者获得精确定制的评估上下文 —— 不继承你的会话历史。这让审查者专注于工作成果，而非你的思维过程，并保留你自己的上下文用于持续工作。

**核心原则：** 早审查，常审查。

## 何时请求审查

**必须：**
- 在 subagent-driven development 的每个任务完成后
- 完成主要功能后
- 合并到 main 之前

**可选但有价值：**
- 卡住时（新的视角）
- 重构前（基线检查）
- 修复复杂 bug 后

## 如何请求

**1. 获取 git SHA：**
```bash
BASE_SHA=$(git rev-parse HEAD~1)  # 或 origin/main
HEAD_SHA=$(git rev-parse HEAD)
```

**2. 调度 code-reviewer 子代理：**

使用 Task 工具，类型为 superpowers:code-reviewer，填写 `code-reviewer.md` 中的模板。

**占位符：**
- `{WHAT_WAS_IMPLEMENTED}` - 你刚构建的内容
- `{PLAN_OR_REQUIREMENTS}` - 它应该做什么
- `{BASE_SHA}` - 起始提交
- `{HEAD_SHA}` - 结束提交
- `{DESCRIPTION}` - 简要概述

**3. 处理反馈：**
- 立即修复关键问题
- 在继续之前修复重要问题
- 记录次要问题供后续处理
- 如果审查者错了，据理反驳

## 示例

```
[刚完成任务 2：添加验证函数]

你：让我在继续之前请求代码审查。

BASE_SHA=$(git log --oneline | grep "Task 1" | head -1 | awk '{print $1}')
HEAD_SHA=$(git rev-parse HEAD)

[调度 superpowers:code-reviewer 子代理]
  WHAT_WAS_IMPLEMENTED: 会话索引的验证和修复函数
  PLAN_OR_REQUIREMENTS: docs/superpowers/plans/deployment-plan.md 中的任务 2
  BASE_SHA: a7981ec
  HEAD_SHA: 3df7661
  DESCRIPTION: 添加了 verifyIndex() 和 repairIndex()，包含 4 种问题类型

[子代理返回]：
  优点：架构清晰，有真实测试
  问题：
    重要：缺少进度指示器
    次要：报告间隔使用魔数 (100)
  评估：可以继续

你：[修复进度指示器]
[继续任务 3]
```

## 与工作流集成

**Subagent-Driven Development：**
- 每个任务后审查
- 在问题扩散之前捕获
- 继续下一个任务前修复

**Executing Plans：**
- 每批（3 个任务）后审查
- 获取反馈，应用，继续

**临时开发：**
- 合并前审查
- 卡住时审查

## 红旗

**绝不：**
- 因为"很简单"跳过审查
- 忽略关键问题
- 在重要问题未修复的情况下继续
- 对有效的技术反馈进行争论

**如果审查者错了：**
- 用技术论据反驳
- 展示证明其有效的代码/测试
- 请求澄清

参见模板：requesting-code-review/code-reviewer.md
