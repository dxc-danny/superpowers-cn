---
name: writing-plans
description: Use when you have a spec or requirements for a multi-step task, before touching code
---

# 编写计划

## 概述

编写全面的实施计划，假设工程师对我们的代码库一无所知且品味可疑。记录他们需要知道的一切：每个任务要修改哪些文件、代码、可能需要检查的文档、如何测试。将整个计划以细粒度任务的形式给出。DRY。YAGNI。TDD。频繁提交。

假设他们是熟练的开发者，但对我们的工具集或问题领域几乎一无所知。假设他们不太了解好的测试设计。

**开始时宣布：** "我正在使用 writing-plans 技能来创建实施计划。"

**上下文：** 这应该在专用 worktree 中运行（由 brainstorming 技能创建）。

**保存计划到：** `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`
-（用户对计划位置的偏好覆盖此默认值）

## 范围检查

如果规格涵盖多个独立子系统，它应该在 brainstorming 期间被分解为子项目规格。如果没有，建议将其分解为单独的计划 —— 每个子系统一个。每个计划应该自行产生可工作、可测试的软件。

## 文件结构

在定义任务之前，先规划哪些文件将被创建或修改，以及每个文件的职责。这是分解决策被锁定的地方。

- 设计具有清晰边界和良好定义接口的单元。每个文件应该有一个清晰的职责。
- 你对能同时掌握上下文的代码推理得最好，当文件专注时你的编辑更可靠。偏爱更小、更专注的文件，而非做太多事的大文件。
- 一起更改的文件应该放在一起。按职责拆分，而非按技术层。
- 在现有代码库中，遵循既定模式。如果代码库使用大文件，不要单方面重构 —— 但如果你正在修改的文件变得笨重，在计划中包含拆分是合理的。

这个结构为任务分解提供依据。每个任务应该产生独立有意义的自包含更改。

## 细粒度任务粒度

**每个步骤是一个动作（2-5 分钟）：**
- "写失败的测试" —— 步骤
- "运行它确保它失败" —— 步骤
- "写最少的代码让测试通过" —— 步骤
- "运行测试确保它们通过" —— 步骤
- "提交" —— 步骤

## 计划文档头部

**每个计划必须以此头部开始：**

```markdown
# [功能名称] 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**目标：** [一句话描述构建内容]

**架构：** [2-3 句关于方法的描述]

**技术栈：** [关键技术/库]

---
```

## 任务结构

````markdown
### 任务 N：[组件名称]

**文件：**
- 创建：`exact/path/to/file.py`
- 修改：`exact/path/to/existing.py:123-145`
- 测试：`tests/exact/path/to/test.py`

- [ ] **步骤 1：编写失败的测试**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

- [ ] **步骤 2：运行测试验证它失败**

运行：`pytest tests/path/test.py::test_name -v`
预期：FAIL with "function not defined"

- [ ] **步骤 3：写最少的实现**

```python
def function(input):
    return expected
```

- [ ] **步骤 4：运行测试验证它通过**

运行：`pytest tests/path/test.py::test_name -v`
预期：PASS

- [ ] **步骤 5：提交**

```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
````

## 无占位符

每个步骤必须包含工程师需要的实际内容。以下是**计划失败** —— 永远不要写：
- "TBD"、"TODO"、"稍后实现"、"填写细节"
- "添加适当的错误处理" / "添加验证" / "处理边缘情况"
- "为上述写测试"（没有实际测试代码）
- "类似于任务 N"（重复代码 —— 工程师可能乱序阅读任务）
- 描述要做什么但不展示如何做的步骤（代码步骤需要代码块）
- 引用在任何任务中未定义的类型、函数或方法

## 记住
- 始终使用精确的文件路径
- 每个步骤都有完整代码 —— 如果步骤更改了代码，展示代码
- 带预期输出的精确命令
- DRY、YAGNI、TDD、频繁提交

## 自我审查

写完完整计划后，用新的眼光看规格并对照计划检查。这是你自己运行的检查清单 —— 不是子代理调度。

**1. 规格覆盖：** 浏览规格的每个部分/需求。你能指出实现它的任务吗？列出任何差距。

**2. 占位符扫描：** 在你的计划中搜索红旗 —— 上述"无占位符"部分的任何模式。修复它们。

**3. 类型一致性：** 你在后面的任务中使用的类型、方法签名和属性名与前面定义的一致吗？任务 3 中的 `clearLayers()` 但任务 7 中的 `clearFullLayers()` 是一个 bug。

如果发现问题，inline 修复。不需要重新审查 —— 直接修复然后继续。如果发现规格需求没有对应任务，添加任务。

## 执行交接

保存计划后，提供执行选择：

**"计划完成并保存到 `docs/superpowers/plans/<filename>.md`。两种执行选项：**

**1. Subagent-Driven（推荐）** —— 我为每个任务调度一个新子代理，任务间审查，快速迭代

**2. 内联执行** —— 使用 executing-plans 在此会话中执行任务，批次执行带检查点

**选择哪个？"**

**如果选择了 Subagent-Driven：**
- **必需的子技能：** 使用 superpowers:subagent-driven-development
- 每个任务新子代理 + 两阶段审查

**如果选择了内联执行：**
- **必需的子技能：** 使用 superpowers:executing-plans
- 批次执行带审查检查点
