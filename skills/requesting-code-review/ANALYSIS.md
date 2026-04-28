# Requesting Code Review Skill 深度分析

## 一、技能定位

**类型：** Technique（技术指导型）+ 工具集成型

**解决的核心问题：** AI 写完代码后不愿意主动审查，或者审查流于形式。需要一套标准化的审查机制，确保问题在合并前被发现。

**在整个工作流中的位置：**

```
brainstorming → writing-plans → executing-plans → requesting-code-review
                                                    (评审)
```

作为质量关，在执行过程中和执行完成后都可以触发。

---

## 二、文件结构

```
requesting-code-review/
├── SKILL.md           # 主文件：审查请求流程（~106 行）
└── code-reviewer.md   # Subagent prompt 模板（~147 行）
```

**主从分离：** SKILL.md 定义"何时请求、如何处理反馈"，code-reviewer.md 定义"审查者如何审查"。两个文件服务于不同的读者：
- SKILL.md → 主 session 的 AI（请求方）
- code-reviewer.md → 被派出的 subagent（审查方）

---

## 三、SKILL.md 结构解剖

### 3.1 核心原则

```markdown
**Core principle:** Review early, review often.
```

一句话定义整个技能的精神。后面的所有内容都是这个原则的具体化。

### 3.2 上下文隔离设计

```markdown
The reviewer gets precisely crafted context for evaluation — never your session's history.
This keeps the reviewer focused on the work product, not your thought process,
and preserves your own context for continued work.
```

**这是最重要的设计决策。** 审查者看不到你的对话历史，只能看到你精心准备的上下文。

为什么？
1. **避免偏见传染** —— 审查者不会因为你"很辛苦"而放宽标准
2. **节省上下文** —— subagent 的上下文窗口留给代码审查，不浪费在历史对话上
3. **模拟真实 Code Review** —— 真实的 PR Reviewer 也只看 diff，不看你的聊天记录

### 3.3 审查时机

```markdown
**Mandatory:**
- After each task in subagent-driven development
- After completing major feature
- Before merge to main

**Optional but valuable:**
- When stuck (fresh perspective)
- Before refactoring (baseline check)
- After fixing complex bug
```

**"Review early, review often" 的具体化：** 不是"完成后才审查"，而是在关键节点审查。特别是 subagent-driven-development 中每个 task 完成后都要审查——防止问题累积。

"When stuck" 是一个有趣的设计——当 AI 卡住时，换一个"人"（subagent）来看看，经常能发现盲点。

---

## 四、审查请求流程

### 4.1 三步流程

```
1. Get git SHAs → 2. Dispatch code-reviewer subagent → 3. Act on feedback
```

### 4.2 SHA 范围选择

```bash
BASE_SHA=$(git rev-parse HEAD~1)  # or origin/main
HEAD_SHA=$(git rev-parse HEAD)
```

用 git SHA 精确定位审查范围。不是"审查最近的所有代码"，而是"审查这两个 commit 之间的代码"。这确保审查者聚焦在正确的变更上。

### 4.3 占位符模板

```markdown
- {WHAT_WAS_IMPLEMENTED} - What you just built
- {PLAN_OR_REQUIREMENTS} - What it should do
- {BASE_SHA} - Starting commit
- {HEAD_SHA} - Ending commit
- {DESCRIPTION} - Brief summary
```

5 个占位符，每个都有明确的目的：
- **WHAT_WAS_IMPLEMENTED** → 审查者知道审查什么
- **PLAN_OR_REQUIREMENTS** → 审查者有判断标准
- **BASE_SHA / HEAD_SHA** → 审查者知道审查范围
- **DESCRIPTION** → 简要上下文

### 4.4 反馈分级处理

```markdown
- Fix Critical issues immediately
- Fix Important issues before proceeding
- Note Minor issues for later
- Push back if reviewer is wrong (with reasoning)
```

**四级处理策略：**

| 级别 | 行动 | 原因 |
|------|------|------|
| Critical | 立即修 | Bug / 安全问题 / 数据丢失风险 |
| Important | 前进前修 | 架构问题 / 功能缺失 |
| Minor | 记下来 | 代码风格 / 优化建议 |
| 审查者错了 | 要反驳 | 不是所有 feedback 都是对的 |

最后一条特别重要——**允许 push back**。这是对 AI "盲目接受反馈"倾向的对抗。

---

## 五、code-reviewer.md（审查者模板）

### 5.1 审查清单

```markdown
**Code Quality:**
- Clean separation of concerns?
- Proper error handling?
- Type safety (if applicable)?
- DRY principle followed?
- Edge cases handled?

**Architecture:**
- Sound design decisions?
- Scalability considerations?
- Performance implications?
- Security concerns?

**Testing:**
- Tests actually test logic (not mocks)?
- Edge cases covered?
- Integration tests where needed?
- All tests passing?

**Requirements:**
- All plan requirements met?
- Implementation matches spec?
- No scope creep?
- Breaking changes documented?

**Production Readiness:**
- Migration strategy (if schema changes)?
- Backward compatibility considered?
- Documentation complete?
- No obvious bugs?
```

5 个维度，每个维度 4-5 个检查项。这是一个全面的审查框架，但关键是——**审查者必须对每一项给出具体结论，不能"looks good"了事。**

### 5.2 输出格式

```markdown
### Strengths
[What's well done? Be specific.]

### Issues
#### Critical (Must Fix)
[Bugs, security issues, data loss risks, broken functionality]
#### Important (Should Fix)
[Architecture problems, missing features, poor error handling, test gaps]
#### Minor (Nice to Have)
[Code style, optimization opportunities, documentation improvements]

**For each issue:**
- File:line reference
- What's wrong
- Why it matters
- How to fix (if not obvious)

### Assessment
**Ready to merge?** [Yes/No/With fixes]
```

**关键约束：**
- 每个 issue 必须有 file:line 引用 —— 不允许模糊反馈
- 必须给出明确的 merge 判断 —— 不允许"看起来还行"
- 必须先列出 Strengths —— 避免全盘否定，保持建设性

### 5.3 Critical Rules

```markdown
**DON'T:**
- Say "looks good" without checking
- Mark nitpicks as Critical
- Give feedback on code you didn't review
- Be vague ("improve error handling")
- Avoid giving a clear verdict
```

每一条 DON'T 都对应 AI 审查者的常见问题：
- 不审查就说 "looks good" → AI 的偷懒倾向
- 把 nitpick 标为 Critical → AI 倾向于把所有问题都标为高优先级
- 审查没看的代码 → AI 可能只看了 diff 的一部分就给出全局评价
- 模糊反馈 → "improve error handling" 没有可操作性
- 不给明确判断 → AI 倾向于模棱两可

---

## 六、与工作流的集成

```markdown
**Subagent-Driven Development:** Review after EACH task
**Executing Plans:** Review after each batch (3 tasks)
**Ad-Hoc Development:** Review before merge
```

三种工作流有不同的审查频率：
- Subagent-Driven 每个 task 审查（最高频，质量最高）
- Executing Plans 每 3 个 task 审查（中等频率）
- Ad-Hoc 只在合并前审查（最低频率）

这个梯度设计体现了实用主义——不是所有场景都需要最高频的审查。

---

## 七、Example 部分

```markdown
[Just completed Task 2: Add verification function]

You: Let me request code review before proceeding.

[Dispatch superpowers:code-reviewer subagent]

[Subagent returns]:
  Strengths: Clean architecture, real tests
  Issues:
    Important: Missing progress indicators
    Minor: Magic number (100) for reporting interval
  Assessment: Ready to proceed

You: [Fix progress indicators]
[Continue to Task 3]
```

示例展示了完整的审查循环：请求 → 审查 → 修复 → 继续。特别值得注意的是：
- 审查者同时给出了 Strengths 和 Issues（不是只挑毛病）
- Important issue 被立即修复
- Minor issue 只是被 noted（没有在示例中修复）

---

## 八、Red Flags 部分

```markdown
**Never:**
- Skip review because "it's simple"
- Ignore Critical issues
- Proceed with unfixed Important issues
- Argue with valid technical feedback

**If reviewer wrong:**
- Push back with technical reasoning
- Show code/tests that prove it works
- Request clarification
```

"Skip review because it's simple" 被放在第一位——这是最常见的偷懒理由。与 brainstorming 的 "This Is Too Simple To Need A Design" 形成呼应。

"If reviewer wrong" 给出了具体的 push back 方式：用技术推理、用代码证明、请求澄清。而不是直接忽略。

---

## 九、核心套路速查表

| 套路 | 做法 | 目的 |
|------|------|------|
| 上下文隔离 | 审查者不看到对话历史 | 避免偏见传染，模拟真实 Review |
| SHA 精确定位 | 用 git diff 范围审查 | 聚焦在正确的变更上 |
| 占位符模板 | 5 个精心设计的上下文字段 | 确保审查者有足够信息 |
| 四级反馈处理 | Critical/Important/Minor/Push back | 区分轻重缓急 |
| 允许 push back | "Argue with reasoning" | 防止盲目接受反馈 |
| 结构化输出 | Strengths/Issues/Assessment | 强制具体、不允许模糊 |
| 审查频率梯度 | 按工作流类型调整频率 | 实用主义 |
| Red Flags | 明确禁止行为 | 对抗常见偷懒借口 |
| 主从分离 | SKILL.md + code-reviewer.md | 请求方和审查方各取所需 |
