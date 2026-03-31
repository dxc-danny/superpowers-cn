---
name: executing-plans
description: Use when you have a written implementation plan to execute in a separate session with review checkpoints
---

# 执行计划

## 概述

加载计划，严格审查，执行所有任务，完成后报告。

**开始时宣布：** "我正在使用 executing-plans 技能来实现这个计划。"

**注意：** 告诉你的伙伴，Superpowers 在有 subagent 支持的情况下效果更好。如果运行在支持 subagent 的平台上（如 Claude Code 或 Codex），工作质量会显著提高。如果有 subagent 可用，请使用 superpowers:subagent-driven-development 而不是本技能。

## 流程

### 步骤 1：加载和审查计划
1. 阅读计划文件
2. 严格审查 —— 找出关于计划的任何问题或疑虑
3. 如果有疑虑：在开始前向你的伙伴提出
4. 如果没有疑虑：创建 TodoWrite 并继续

### 步骤 2：执行任务

对于每个任务：
1. 标记为进行中
2. 严格遵循每个步骤（计划有细粒度的步骤）
3. 按规定运行验证
4. 标记为已完成

### 步骤 3：完成开发

所有任务完成并验证后：
- 宣布："我正在使用 finishing-a-development-branch 技能来完成这项工作。"
- **必需的子技能：** 使用 superpowers:finishing-a-development-branch
- 遵循该技能验证测试、呈现选项、执行选择

## 何时停止并寻求帮助

**立即停止执行当：**
- 遇到阻塞（缺少依赖、测试失败、指令不清晰）
- 计划有阻碍开始的严重缺陷
- 不理解某条指令
- 验证反复失败

**寻求澄清而非猜测。**

## 何时回到前面的步骤

**回到审查（步骤 1）当：**
- 伙伴根据你的反馈更新了计划
- 基本方法需要重新思考

**不要强行突破阻塞** —— 停下来并询问。

## 记住
- 先严格审查计划
- 严格遵循计划步骤
- 不要跳过验证
- 当计划说要引用技能时就引用
- 遇到阻塞时停止，不要猜测
- 未经你的伙伴明确同意，绝不在 main/master 分支上开始实施

## 集成

**必需的工作流技能：**
- **superpowers:using-git-worktrees** —— 必需：在开始前设置隔离的工作空间
- **superpowers:writing-plans** —— 创建本技能执行的计划
- **superpowers:finishing-a-development-branch** —— 所有任务完成后完成开发
