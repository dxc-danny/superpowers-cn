# 文档审查系统实现计划

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan.

**目标：** 为头脑风暴和 writing-plans 技能添加规范和计划文档审查循环。

**架构：** 在每个技能目录中创建审查者提示模板。修改技能文件以在文档创建后添加审查循环。使用 Task 工具和通用子代理进行审查者分派。

**技术栈：** Markdown 技能文件，通过 Task 工具分派子代理

**规范：** docs/superpowers/specs/2026-01-22-document-review-system-design.md

---

## 块 1：规范文档审查者

此块为头脑风暴技能添加规范文档审查者。

### 任务 1：创建规范文档审查者提示模板

**文件：**
- 创建：`skills/brainstorming/spec-document-reviewer-prompt.md`

- [ ] **步骤 1：** 创建审查者提示模板文件

```markdown
# 规范文档审查者提示模板

在分派规范文档审查者子代理时使用此模板。

**目的：** 验证规范完整、一致且准备好进行实现规划。

**在以下之后分派：** 规范文档写入 docs/superpowers/specs/ 后
```
Task tool (general-purpose):
  description: "Review spec document"
  prompt: |
    You are a spec document reviewer. Verify this spec is complete and ready for planning.

    **Spec to review:** [SPEC_FILE_PATH]

    ## What to Check

    | Category | What to Look For |
    |----------|------------------|
    | Completeness | TODOs, placeholders, "TBD", incomplete sections |
    | Coverage | Missing error handling, edge cases, integration points |
    | Consistency | Internal contradictions, conflicting requirements |
    | Clarity | Ambiguous requirements |
    | YAGNI | Unrequested features, over-engineering |

    ## CRITICAL

    Look especially hard for:
    - Any TODO markers or placeholder text
    - Sections saying "to be defined later" or "will spec when X is done"
    - Sections noticeably less detailed than others

    ## Output Format

    ## Spec Review

    **Status:** ✅ Approved | ❌ Issues Found

    **Issues (if any):**
    - [Section X]: [specific issue] - [why it matters]

    **Recommendations (advisory):**
    - [suggestions that don't block approval]
```

**审查者返回：** 状态、问题（如果有）、建议
```

- [ ] **步骤 2：** 验证文件已正确创建

运行：`cat skills/brainstorming/spec-document-reviewer-prompt.md | head -20`
预期：显示标题和目的部分

- [ ] **步骤 3：提交**

```bash
git add skills/brainstorming/spec-document-reviewer-prompt.md
git commit -m "feat: add spec document reviewer prompt template"
```

---

### 任务 2：将审查循环添加到头脑风暴技能

**文件：**
- 修改：`skills/brainstorming/SKILL.md`

- [ ] **步骤 1：** 阅读当前头脑风暴技能

运行：`cat skills/brainstorming/SKILL.md`

- [ ] **步骤 2：** 在 "After the Design" 之后添加审查循环部分

找到 "After the Design" 部分，并在文档之后、实施之前添加新的 "Spec Review Loop" 部分：

```markdown
**规范审查循环：**
编写规范文档后：
1. 分派 spec-document-reviewer 子代理（参见 spec-document-reviewer-prompt.md）
2. 如果 ❌ 发现问题：
   - 修复规范文档中的问题
   - 重新分派审查者
   - 重复直到 ✅ 批准
3. 如果 ✅ 批准：继续进行实施设置

**审查循环指导：**
- 编写规范的同一代理修复它（保留上下文）
- 如果循环超过 5 次迭代，提请人工指导
- 审查者是顾问性的——如果你认为反馈不正确，解释分歧
```

- [ ] **步骤 3：** 验证更改

运行：`grep -A 15 "Spec Review Loop" skills/brainstorming/SKILL.md`
预期：显示新的审查循环部分

- [ ] **步骤 4：提交**

```bash
git add skills/brainstorming/SKILL.md
git commit -m "feat: add spec review loop to brainstorming skill"
```

---

## 块 2：计划文档审查者

此块为 writing-plans 技能添加计划文档审查者。

### 任务 3：创建计划文档审查者提示模板

**文件：**
- 创建：`skills/writing-plans/plan-document-reviewer-prompt.md`

- [ ] **步骤 1：** 创建审查者提示模板文件

```markdown
# 计划文档审查者提示模板

在分派计划文档审查者子代理时使用此模板。

**目的：** 验证计划块完整、与规范匹配且有正确的任务分解。

**在以下之后分派：** 每个计划块编写完成后
```
Task tool (general-purpose):
  description: "Review plan chunk N"
  prompt: |
    You are a plan document reviewer. Verify this plan chunk is complete and ready for implementation.

    **Plan chunk to review:** [PLAN_FILE_PATH] - Chunk N only
    **Spec for reference:** [SPEC_FILE_PATH]

    ## What to Check

    | Category | What to Look For |
    |----------|------------------|
    | Completeness | TODOs, placeholders, incomplete tasks, missing steps |
    | Spec Alignment | Chunk covers relevant spec requirements, no scope creep |
    | Task Decomposition | Tasks atomic, clear boundaries, steps actionable |
    | Task Syntax | Checkbox syntax (`- [ ]`) on tasks and steps |
    | Chunk Size | Each chunk under 1000 lines |

    ## CRITICAL

    Look especially hard for:
    - Any TODO markers or placeholder text
    - Steps that say "similar to X" without actual content
    - Incomplete task definitions
    - Missing verification steps or expected outputs

    ## Output Format

    ## Plan Review - Chunk N

    **Status:** ✅ Approved | ❌ Issues Found

    **Issues (if any):**
    - [Task X, Step Y]: [specific issue] - [why it matters]

    **Recommendations (advisory):**
    - [suggestions that don't block approval]
```

**审查者返回：** 状态、问题（如果有）、建议
```

- [ ] **步骤 2：** 验证文件已创建

运行：`cat skills/writing-plans/plan-document-reviewer-prompt.md | head -20`
预期：显示标题和目的部分

- [ ] **步骤 3：提交**

```bash
git add skills/writing-plans/plan-document-reviewer-prompt.md
git commit -m "feat: add plan document reviewer prompt template"
```

---

### 任务 4：将审查循环添加到 Writing-Plans 技能

**文件：**
- 修改：`skills/writing-plans/SKILL.md`

- [ ] **步骤 1：** 阅读当前技能文件

运行：`cat skills/writing-plans/SKILL.md`

- [ ] **步骤 2：** 添加逐块审查部分

在 "Execution Handoff" 部分之前添加：

```markdown
## 计划审查循环

完成计划的每个块后：

1. 为当前块分派 plan-document-reviewer 子代理
   - 提供：块内容、规范文档路径
2. 如果 ❌ 发现问题：
   - 修复该块中的问题
   - 重新为该块分派审查者
   - 重复直到 ✅ 批准
3. 如果 ✅ 批准：继续到下一块（如果是最后一块则到执行交接）

**块边界：** 使用 `## Chunk N: <name>` 标题来分隔块。每个块应 ≤1000 行且逻辑上自包含。
```

- [ ] **步骤 3：** 更新任务语法示例以使用复选框

将任务结构部分更改为显示复选框语法：

```markdown
### Task N: [Component Name]

- [ ] **Step 1:** Write the failing test
  - File: `tests/path/test.py`
  ...
```

- [ ] **步骤 4：** 验证审查循环部分已添加

运行：`grep -A 15 "Plan Review Loop" skills/writing-plans/SKILL.md`
预期：显示新的审查循环部分

- [ ] **步骤 5：** 验证任务语法示例已更新

运行：`grep -A 5 "Task N:" skills/writing-plans/SKILL.md`
预期：显示复选框语法 `### Task N:`

- [ ] **步骤 6：提交**

```bash
git add skills/writing-plans/SKILL.md
git commit -m "feat: add plan review loop and checkbox syntax to writing-plans skill"
```

---

## 块 3：更新计划文档头

此块更新计划文档头模板以引用新的复选框语法要求。

### 任务 5：更新 Writing-Plans 技能中的计划头模板

**文件：**
- 修改：`skills/writing-plans/SKILL.md`

- [ ] **步骤 1：** 阅读当前计划头模板

运行：`grep -A 20 "Plan Document Header" skills/writing-plans/SKILL.md`

- [ ] **步骤 2：** 更新头模板以引用复选框语法

计划头应该注意任务和步骤使用复选框语法。更新头注释：

```markdown
> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Tasks and steps use checkbox (`- [ ]`) syntax for tracking.
```

- [ ] **步骤 3：** 验证更改

运行：`grep -A 5 "For agentic workers:" skills/writing-plans/SKILL.md`
预期：显示更新后的头，包含复选框语法提及

- [ ] **步骤 4：提交**

```bash
git add skills/writing-plans/SKILL.md
git commit -m "docs: update plan header to reference checkbox syntax"
```