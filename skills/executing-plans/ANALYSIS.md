# Executing Plans Skill 深度分析

## 一、技能定位

**类型：** Guidance（指导型）技能

**解决的核心问题：** AI 执行计划时要么死板执行不思考，要么自作主张偏离计划。需要在"严格遵循"和"灵活判断"之间找到平衡。

**在整个工作流中的位置：**

```
brainstorming → writing-plans → executing-plans → requesting-code-review
                                  (执行)
```

作为 writing-plans 的下游，是 Inline Execution 模式的执行技能（与 subagent-driven-development 是替代方案）。

---

## 二、极简设计哲学

executing-plans 是四个技能中最短的一个（~71 行），只有三个步骤：

```
Step 1: Load and Review Plan
Step 2: Execute Tasks
Step 3: Complete Development
```

**为什么这么短？**

因为这个技能的复杂性已经被上游的两个技能吸收了：
- **brainstorming** 已经完成了设计决策
- **writing-plans** 已经把每一步的具体代码写好了
- executing-plans 只需要"照着做"

这是一个重要的设计原则：**每个技能只解决一个问题，复杂性由流水线分摊。**

---

## 三、三步流程详解

### 3.1 Step 1: Load and Review Plan

```markdown
1. Read plan file
2. Review critically - identify any questions or concerns about the plan
3. If concerns: Raise them with your human partner before starting
4. If no concerns: Create TodoWrite and proceed
```

**关键设计：先审查再执行。** 不是拿到计划就埋头干，而是先评估计划是否有问题。这是一个 meta 层面的检查——执行者可能会发现计划作者没注意到的问题。

**"If concerns: Raise them"** —— 明确允许停止。这是对 AI "不敢质疑"倾向的对抗。

### 3.2 Step 2: Execute Tasks

```markdown
For each task:
1. Mark as in_progress
2. Follow each step exactly (plan has bite-sized steps)
3. Run verifications as specified
4. Mark as completed
```

"Follow each step exactly" —— 这是和 Step 1 的审查配合的。审查时不执行，执行时不审查。一旦决定执行，就严格遵循。

### 3.3 Step 3: Complete Development

```markdown
- Announce: "I'm using the finishing-a-development-branch skill to complete this work."
- **REQUIRED SUB-SKILL:** Use superpowers:finishing-a-development-branch
```

完成后必须调用 `finishing-a-development-branch`，不能自己决定"做完了"。这确保了收尾工作（测试、提交等）也遵循标准化流程。

---

## 四、停止条件设计

### 4.1 When to Stop and Ask for Help

```markdown
**STOP executing immediately when:**
- Hit a blocker (missing dependency, test fails, instruction unclear)
- Plan has critical gaps preventing starting
- You don't understand an instruction
- Verification fails repeatedly
```

这四条停止条件对应四种真实的执行障碍：
1. **外部依赖缺失** —— 不是执行者能解决的
2. **计划有缺陷** —— 需要回到 writing-plans 修复
3. **指令不清** —— 需要人类澄清
4. **验证反复失败** —— 说明实现可能方向错了

### 4.2 "Don't force through blockers"

```markdown
**Ask for clarification rather than guessing.**
```

这是对 AI "我自己想办法解决"倾向的对抗。AI 遇到问题时倾向于自己编造解决方案而不是求助，这经常导致更大的问题。

---

## 五、与 Subagent-Driven Development 的关系

```markdown
**Note:** Tell your human partner that Superpowers works much better with access to
subagents. The quality of its work will be significantly higher if run on a platform
with subagent support (such as Claude Code or Codex). If subagents are available,
use superpowers:subagent-driven-development instead of this skill.
```

**主动推荐更好的方案。** executing-plans 是"够用"的方案，subagent-driven-development 是"更好"的方案。技能本身会告诉你"如果有更好的工具，别用我"。

这体现了对用户负责的态度——不是所有环境都支持 subagent，所以需要这个 fallback 方案，但不应该隐瞒有更好的选择。

---

## 六、与其他技能的集成

```markdown
**Required workflow skills:**
- superpowers:using-git-worktrees - REQUIRED: Set up isolated workspace before starting
- superpowers:writing-plans - Creates the plan this skill executes
- superpowers:finishing-a-development-branch - Complete development after all tasks
```

三个关联技能分别对应：
- **工作前：** 用 worktree 隔离工作区
- **工作时：** 执行 writing-plans 产出的计划
- **工作后：** 用标准流程收尾

这形成了一个完整的生命周期管理。

---

## 七、与 brainstorming 的设计对比

| 维度 | brainstorming | executing-plans |
|------|---------------|-----------------|
| 长度 | ~165 行 | ~71 行 |
| 复杂性来源 | 需要引导开放式对话 | 计划已经写好了 |
| 决策点 | 多（4 个菱形 + 2 个回环） | 少（主要是"停不停"的决策） |
| 人机交互 | 频繁（一次一问） | 稀少（只在遇到问题时） |
| 说服策略 | Authority + Commitment + Social Proof | 轻量 Authority |

brainstorming 需要大量说服策略因为 AI 有强烈的"跳过设计"冲动。executing-plans 只需要简单的"停下来问"就够了，因为 AI 的执行冲动反而是被鼓励的。

---

## 八、核心套路速查表

| 套路 | 做法 | 目的 |
|------|------|------|
| 极简设计 | 只有 3 步 | 复杂性已由上游技能吸收 |
| 先审后做 | Step 1 审查计划再执行 | 防止执行有缺陷的计划 |
| 明确停止条件 | 4 种必须停止的情况 | 防止 AI 硬闯阻碍 |
| 禁止猜测 | "Ask for clarification rather than guessing" | 防止 AI 遇到问题自己编造方案 |
| 强制收尾技能 | 完成后必须调用 finishing-a-development-branch | 确保标准化收尾 |
| 主动降级 | 推荐使用更好的 subagent 方案 | 对用户负责 |
| TodoWrite 跟踪 | 每个任务标记 in_progress/completed | 进度可视化 |
| Worktree 隔离 | REQUIRED 使用 git worktree | 保护主分支 |
