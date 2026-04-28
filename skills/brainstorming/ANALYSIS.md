# Brainstorming Skill 深度分析

## 一、技能定位

**类型：** Discipline-enforcing（纪律执行型）技能

**解决的核心问题：** AI 会跳过设计直接写代码，即使是最简单的任务也会跳过。整篇 SKILL.md 的每一句话都在对抗这个行为偏差。

**在整个工作流中的位置：**

```
brainstorming → writing-plans → executing-plans → requesting-code-review
  (设计)          (计划)          (执行)            (评审)
```

brainstorming 是流水线的第一个环节，它的唯一出口是 `writing-plans`。

---

## 二、文件结构

```
brainstorming/
├── SKILL.md                        # 主文件：行为指令（~165 行）
├── visual-companion.md             # 可视化伴侣详细指南（~280 行，按需加载）
├── spec-document-reviewer-prompt.md # spec 审查 subagent 模板（~50 行）
└── scripts/
    ├── start-server.sh             # 启动本地 HTTP 服务器（~149 行）
    ├── stop-server.sh              # 关闭服务器（~57 行）
    ├── server.cjs                  # 零依赖 HTTP+WebSocket 服务器（~355 行）
    ├── helper.js                   # 浏览器端交互脚本（~89 行）
    └── frame-template.html         # 浏览器页面框架模板（~215 行）
```

**设计原则：** 主文件保持简洁，重量级内容抽离到独立文件。visual-companion 有 280+ 行，但不会默认加载到上下文——只有用户同意使用可视化伴侣时才会读取。

---

## 三、SKILL.md 结构解剖

### 3.1 Frontmatter

```yaml
name: brainstorming
description: "You MUST use this before any creative work - creating features, building components, adding functionality, or modifying behavior. Explores user intent, requirements and design before implementation."
```

**关键设计：**
- 用 "You MUST" 开头强制触发
- description 只描述触发条件，不总结工作流（避免 AI 只看 description 就自以为懂了）
- 列出具体的触发场景：creating features, building components, adding functionality, modifying behavior

### 3.2 硬门控（HARD-GATE）

```markdown
<HARD-GATE>
Do NOT invoke any implementation skill, write any code, scaffold any project,
or take any implementation action until you have presented a design and the user has approved it.
This applies to EVERY project regardless of perceived simplicity.
</HARD-GATE>
```

`<HARD-GATE>` 是 Superpowers 的特殊机制，相当于"不可协商的规则"。

### 3.3 反模式防御

```markdown
## Anti-Pattern: "This Is Too Simple To Need A Design"

Every project goes through this process. A todo list, a single-function utility,
a config change — all of them. "Simple" projects are where unexamined assumptions
cause the most wasted work.
```

提前封堵 AI 最常用的借口——"这个太简单了不需要设计"。

### 3.4 9 步 Checklist

```markdown
You MUST create a task for each of these items and complete them in order:
1. Explore project context
2. Offer visual companion
3. Ask clarifying questions
4. Propose 2-3 approaches
5. Present design
6. Write design doc
7. Spec self-review
8. User reviews written spec
9. Transition to implementation
```

配合 TodoWrite 逐项追踪，防止跳步。

---

## 四、说服策略分析

brainstorming 使用了以下说服原则（参见 `writing-skills/persuasion-principles.md`）：

### 4.1 Authority（权威）—— 最高密度使用

- `<HARD-GATE>` 标签、MUST / NEVER / REQUIRED 等绝对化语言
- 消除 AI 的决策犹豫："You MUST" 不留商量余地

### 4.2 Commitment（承诺）—— Checklist 机制

- 9 步 checklist + TodoWrite 逐项完成
- 公开承诺 + 逐项追踪，让 AI 很难跳步

### 4.3 Social Proof（社会认同）—— 反模式叙事

- "Every project goes through this process"、"all of them"
- 建立社会规范：所有人都这么做，你也不例外

### 4.4 Scarcity（稀缺性）—— 唯一出口

```markdown
The ONLY skill you invoke after brainstorming is writing-plans.
```

不给你第二条路，防止 AI 擅自跳到实现。

---

## 五、流程设计的精妙之处

### 5.1 一次一问

```markdown
- Only one question per message
- Prefer multiple choice questions when possible
```

AI 倾向一次问 5 个问题来"高效"推进，但用户面对问题列表会敷衍回答。一次一问保证每个回答的质量。

### 5.2 Visual Companion 的延迟加载

**同意请求必须单独发一条消息：**

```markdown
**This offer MUST be its own message.** Do not combine it with clarifying questions,
context summaries, or any other content.
```

原因：把同意请求夹在其他内容里，用户会忽略或条件反射式同意。单独一条消息确保有意识的选择。

**每个问题独立决定用浏览器还是终端：**

```markdown
A question about a UI topic is not automatically a visual question.
"What does personality mean in this context?" is a conceptual question — use the terminal.
"Which wizard layout works better?" is a visual question — use the browser.
```

### 5.3 Spec Self-Review（自我审查）

在交给用户审查之前，先让 AI 自己审查一遍：

```markdown
1. Placeholder scan: Any "TBD", "TODO", incomplete sections? Fix them.
2. Internal consistency: Do any sections contradict each other?
3. Scope check: Is this focused enough for a single implementation plan?
4. Ambiguity check: Could any requirement be interpreted two different ways?
```

大大减少用户需要提出的修改次数。

### 5.4 双重用户审批门

```
写完 spec → AI 自审 → 用户审查 spec → 用户批准 → 才能进入 writing-plans
```

不是一次审批，是两次：设计口头确认 + spec 文件书面确认。防止"用户口头说好但没认真看"。

### 5.5 Graphviz 流程图

20 节点的 dot 流程图，包含 4 个决策菱形和 2 个回环。这不是装饰——这些正是 AI 容易出错的地方。流程图把"不确定时该怎么做"编码成了图，比纯文字更不容易被忽略。

---

## 六、Scripts 目录：零依赖可视化基础设施

### 6.1 工程决策

server.cjs 用纯 Node.js 标准库实现 HTTP + WebSocket 服务器（~350 行），没有用任何第三方包。这与 Superpowers "零依赖插件" 的项目哲学一致。

### 6.2 关键实现

- **手写 WebSocket RFC 6455 协议**（`encodeFrame` / `decodeFrame`），没有用 `ws` 库
- **文件监听自动刷新**：AI 写 HTML → `fs.watch` 检测 → WebSocket 广播 `reload` → 浏览器自动刷新
- **事件回传**：用户在浏览器点击 → helper.js 通过 WebSocket 发送事件 → 写入 `state_dir/events` → AI 下一轮读取
- **生命周期管理**：30 分钟无操作自动关闭，检测 owner 进程是否存活
- **跨平台适配**：自动检测 Codex / Windows / Gemini 环境并调整后台模式

### 6.3 frame-template.html

提供完整的 CSS 框架：
- 系统深色/浅色模式自适应
- 预制组件：options（A/B 选择）、cards（设计卡片）、mockup（线框图）、split（对比视图）、pros-cons（优缺点）
- AI 只需写内容片段，不用写 `<html>` 或 CSS

---

## 七、核心套路速查表

| 套路 | 做法 | 目的 |
|------|------|------|
| 硬门控 | `<HARD-GATE>` 标签包裹禁止行为 | 防止 AI 跳过关键步骤 |
| 反模式章 | "This Is Too Simple To Need A Design" | 提前封堵 AI 最常用的借口 |
| 强制出口 | "The ONLY skill you invoke is writing-plans" | 防止越级跳到实现 |
| 延迟加载 | visual companion 按需读取 | 节省上下文 token |
| 自审 + 用户审 | 两道审查门 | 减少返工 |
| Checklist + TodoWrite | 9 步必须逐项完成 | 防止跳步 |
| 一次一问 | "Only one question per message" | 保证交互质量 |
| 单独消息请求同意 | Visual companion offer 独立发送 | 确保有意识选择 |
| 零依赖工具链 | 纯 Node.js 实现 WebSocket | 符合项目无依赖原则 |
| 流程可视化 | Graphviz dot 流程图 | 消除决策歧义 |
