# Writing Plans Skill 深度分析

## 一、技能定位

**类型：** Technique（技术指导型）+ Discipline-enforcing（纪律执行型）混合

**解决的核心问题：** AI 写的计划太粗略、充满 placeholder、缺少实际代码，导致执行者（AI 或人）无法按步操作。

**在整个工作流中的位置：**

```
brainstorming → writing-plans → executing-plans → requesting-code-review
                  (计划)
```

brainstorming 的 spec 完成后进入此技能，此技能完成后进入 executing-plans 或 subagent-driven-development。

---

## 二、SKILL.md 结构解剖

### 2.1 Frontmatter

```yaml
name: writing-plans
description: Use when you have a spec or requirements for a multi-step task, before touching code
```

**设计特点：**
- description 用 "Use when..." 开头，只描述触发条件
- "before touching code" 明确时序约束——必须先写计划再写代码
- 没有总结工作流（避免 AI 跳读）

### 2.2 读者模型设计

```markdown
Write comprehensive implementation plans assuming the engineer has zero context
for our codebase and questionable taste.

Assume they are a skilled developer, but know almost nothing about our toolset
or problem domain. Assume they don't know good test design very well.
```

这是整个技能最关键的设计决策之一。定义了一个具体的"目标读者"：
- **有技能** → 不需要解释什么是函数、什么是测试
- **零上下文** → 需要精确的文件路径、完整代码
- **不懂测试设计** → 测试部分必须写得特别详细

这个读者模型实际上就是给未来的 AI 实例写的——它可能在一个全新的 session 中执行计划，没有任何历史上下文。

---

## 三、Bite-Sized Task 粒度控制

### 3.1 粒度定义

```markdown
**Each step is one action (2-5 minutes):**
- "Write the failing test" - step
- "Run it to make sure it fails" - step
- "Implement the minimal code to make the test pass" - step
- "Run the tests and make sure they pass" - step
- "Commit" - step
```

每步 2-5 分钟，粒度细化到"写测试"和"跑测试"是分开的两步。这对应 TDD 的红-绿循环。

### 3.2 为什么这么细？

因为 AI 执行长步骤时容易：
1. 遗漏中间操作（"我跑一下测试"然后忘了跑）
2. 跳过验证（"测试应该会通过"然后不验证）
3. 混淆步骤顺序（先实现再写测试）

拆成原子步骤后，每一步都有明确的预期输出，执行者可以自我验证。

---

## 四、No Placeholders 红线

### 4.1 明确禁止的模式

```markdown
These are **plan failures** — never write them:
- "TBD", "TODO", "implement later", "fill in details"
- "Add appropriate error handling" / "add validation" / "handle edge cases"
- "Write tests for the above" (without actual test code)
- "Similar to Task N" (repeat the code — the engineer may be reading tasks out of order)
- Steps that describe what to do without showing how (code blocks required for code steps)
- References to types, functions, or methods not defined in any task
```

每一条都对应 AI 写计划时的真实偷懒行为：
- "TBD" → AI 不想写细节时的第一反应
- "Add appropriate error handling" → 看起来说了什么但其实什么都没说
- "Similar to Task N" → 执行者可能先看 Task 7 再看 Task 3，引用会断裂
- 不给代码只描述意图 → 执行者需要自己猜实现

### 4.2 "Similar to Task N" 的深意

特别注明"the engineer may be reading tasks out of order"。这是因为 subagent-driven-development 中，不同的 subagent 可能并行执行不同的 task。引用其他 task 的内容会导致上下文断裂。

---

## 五、计划文档模板

### 5.1 Header 模板

```markdown
# [Feature Name] Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development
> (recommended) or superpowers:executing-plans to implement this plan task-by-task.

**Goal:** [One sentence]
**Architecture:** [2-3 sentences]
**Tech Stack:** [Key technologies]
```

Header 中嵌入了执行指引——即使是新的 AI session 拿到这份计划，也能知道该用什么技能来执行。

### 5.2 Task 模板

每个 task 包含：
- **Files:** 精确的文件路径（Create / Modify / Test）
- **Steps:** checkbox 格式，每步包含完整代码和命令
- **Expected output:** 每个命令的预期结果

```markdown
- [ ] **Step 2: Run test to verify it fails**
Run: `pytest tests/path/test.py::test_name -v`
Expected: FAIL with "function not defined"
```

---

## 六、Self-Review 三项检查

```markdown
1. Spec coverage: Skim each section/requirement in the spec. Can you point to a task that implements it?
2. Placeholder scan: Search your plan for red flags
3. Type consistency: Do the types, method signatures, and property names you used in later tasks
   match what you defined in earlier tasks?
```

### 6.1 类型一致性检查的深意

```markdown
A function called `clearLayers()` in Task 3 but `clearFullLayers()` in Task 7 is a bug.
```

这是一个非常真实的 bug 来源。AI 在写长计划时，前后命名不一致是高发问题。专门列出这一项说明作者在实践中多次遇到这个问题。

### 6.2 与 brainstorming 的自审对比

brainstorming 的自审查 spec 的完整性，writing-plans 的自审查计划的一致性。两者形成互补的质量门。

---

## 七、File Structure 决策

```markdown
Before defining tasks, map out which files will be created or modified
and what each one is responsible for.
```

**先定文件结构，再定任务。** 这是因为：

1. 文件结构是分解决策的具象化——如果说不清需要几个文件，说明还没想清楚
2. 文件结构决定了任务的依赖关系——改同一个文件的任务不能并行
3. 每个文件单一职责——"Files that change together should live together"

```markdown
You reason best about code you can hold in context at once, and your edits are more
reliable when files are focused. Prefer smaller, focused files over large ones.
```

这其实是在给 AI 自己写约束——AI 的上下文窗口有限，小文件比大文件更容易正确编辑。

---

## 八、执行交接设计

完成后给出两种选择：

```markdown
1. Subagent-Driven (recommended) - I dispatch a fresh subagent per task, review between tasks
2. Inline Execution - Execute tasks in this session using executing-plans
```

**推荐 Subagent-Driven 的原因：**
- 每个 task 用全新的 subagent，没有历史上下文污染
- task 之间有 review checkpoint，问题不会累积
- 主 session 保持干净，只做协调

---

## 九、Scope Check 机制

```markdown
If the spec covers multiple independent subsystems, it should have been broken into
sub-project specs during brainstorming. If it wasn't, suggest breaking this into
separate plans — one per subsystem.
```

这是 brainstorming 的 Scope Check 的二次保险。如果 brainstorming 阶段漏掉了范围过大的问题，writing-plans 阶段还有机会拦截。

---

## 十、核心套路速查表

| 套路 | 做法 | 目的 |
|------|------|------|
| 读者模型 | "零上下文但有技能的工程师" | 迫使计划足够详细 |
| 原子步骤 | 每步 2-5 分钟，写测试和跑测试分开 | 防止执行遗漏 |
| No Placeholders | 明确列出 6 种禁止模式 | 消除模糊指令 |
| 代码内联 | 每个代码步骤必须包含实际代码 | 执行者不需要自己猜 |
| 预期输出 | 每个命令标注 Expected 结果 | 执行者可以自我验证 |
| Header 嵌入执行指引 | 计划文档自带执行技能引用 | 新 session 也能正确执行 |
| Self-Review 三项检查 | 覆盖度 / placeholder / 类型一致性 | 防止计划内部矛盾 |
| 文件结构先行 | 先定文件再定任务 | 锁定分解决策 |
| Scope 二次保险 | writing-plans 再次检查范围 | 拦截 brainstorming 的遗漏 |
