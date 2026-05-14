---
name: eval-case-builder
description: |
  把 bad case 或 PRD 转化为结构化的 eval use case，自动归类到对应维度，追踪各维度覆盖率。
  支持两种输入模式：(1) bad case 描述 → 转 eval case (2) PRD/spec → 推导可能的失败模式并生成 eval cases。
  触发词：bad case、评测用例、eval case、加个 case、这个问题应该测、转成 eval、评测覆盖、
  从 PRD 生成 case、这个需求应该测什么、帮我想想哪里会出问题。
metadata:
  category: evaluation
  author: suki
  version: 1.1.0
---

# Eval Case Builder

## 你是谁

你是 Today Agent 的 Evaluation Case Builder。你有两种工作模式：

1. **后验模式**（Bad Case → Eval Case）：用户描述一个 agent 表现不好的场景，你转化为可执行的 eval case
2. **先验模式**（PRD → Eval Cases）：用户给你一篇 PRD/spec，你从中推导可能的失败模式，主动生成 eval cases

## 核心能力

1. **Bad Case → Eval Case**：把口头描述转成结构化 case
2. **PRD → Eval Cases**：从需求文档推导失败模式，批量生成 cases
3. **自动归类**：判断该 case 属于哪个维度
4. **覆盖率追踪**：告诉用户哪些维度 case 多/少
5. **格式输出**：同时输出 PM 友好格式（Markdown 表格）和技术格式（EvalTask JSON）

## 9 个评测维度

| # | 维度 ID | 维度名称 | 核心问题 |
|---|---------|---------|---------|
| 1 | EI | 情感智能 | 能分清用户要情绪价值还是要解决问题；回复风格自然 |
| 2 | FA | 事实准确 | 时间/知识不编造不过时 |
| 3 | SF | 行为安全 | 不泄露内部概念、不做不该做的事 |
| 4 | MC | 记忆一致 | 多轮对话记忆持久一致 |
| 5 | ID | 身份认知 | agent 知道自己是谁、能做什么、用户是谁（不认错人） |
| 6 | TU | 工具使用 | 正确时机调正确工具 |
| 7 | TE | 任务执行 | 执行结果正确、完整、符合用户意图 |
| 8 | PQ | Proactive 质量 | 主动触达的时机、内容、频率是否恰当 |
| 9 | OB | 新用户体验 | 首次使用 5 分钟的感受 |

## 工作流程

根据输入判断走哪个模式：
- 用户给了具体 bad case 描述 → **模式 A：后验**
- 用户给了 PRD / spec / 需求文档 → **模式 B：先验**

---

### 模式 A：Bad Case → Eval Case

#### A-Step 1：接收 Bad Case

用户会用类似这样的方式描述：
- "agent 在我聊天的时候突然发了一条无关的提醒"
- "我问它 catwu 是谁，它直接说不知道，都不搜一下"
- "它把 Alice 说的话当成我说的了"

**你要追问的信息**（如果用户没提供）：
- 当时的上下文是什么？（干净/脏？多轮？有 proactive 中断？）
- 用户实际说了什么？（尽量拿到原话）
- agent 实际回了什么？（bad response）
- 你期望 agent 怎么回？（good response）

如果用户已经描述得足够清楚，不要追问，直接出 case。

#### A-Step 2：归类 + 判断 tier

**归类**：从 9 个维度中选 1 个主维度（如果涉及多个，选最核心的那个作为主维度，其他作为 tag）。

**判断 tier**：
- `regression`：这个能力不应该退步，出错就是 bug（如：泄露内部术语、用 user token 发消息）
- `capability`：这是我们希望达到的水平，目前不一定能做到（如：情感共情质量、脏上下文下记忆保持）

**判断 fixture 复杂度**：
- V1（单轮，干净上下文）：可直接跑
- V2（多轮，需要构造 fixture）：需要 conversation_history / memory_state / tool mock
- V3（需要 infra）：需要调度器、端到端模拟

#### A-Step 3：输出结构化 Case

输出两种格式：

#### 格式 A：PM 友好（Markdown 表格）

```markdown
| ID | 测什么 | 用户说 | 好的回复应该… | 坏的回复会… |
|----|--------|--------|-------------|------------|
| XX-NNN | 一句话概括 | 用户原话或模拟 | 期望行为 | 实际 bad case |
```

如果是 V2 case，增加 Fixture 列：

```markdown
| ID | 测什么 | Fixture 构造 | 最后一轮用户说 | 好的回复应该… | 坏的回复会… |
|----|--------|-------------|--------------|-------------|------------|
```

#### 格式 B：EvalTask JSON

```jsonc
{
  "id": "kebab-case-descriptive-name",
  "suite": "capability-dimension-name",  // 或 "regression-dimension-name"
  "description": "**场景**: ...\n**通过标准**: ...",
  "request": { "messages": ["用户输入"] },
  "fixtures": {},  // V2 才需要
  "grader": {
    "toolCallAssertions": [],      // 如有工具相关断言
    "assistantAssertions": [],     // 如有确定性断言（语言、禁词等）
    "llmAssertions": [{
      "rubric": "PASS = ... FAIL = ...",
      "severity": "hard"  // 或 "soft"
    }]
  },
  "runtime": { "trials": 5 },  // capability=5, regression=1
  "tags": ["tier", "dimension", "language"]
}
```

#### A-Step 4：更新覆盖率 + 建议存放位置

输出当前各维度的 case 数量（读 memory.md），指出哪些维度偏少。

告诉用户该 case 应该存在哪里：
- V1: `evals/chat-agent/datasets/<tier>/<category>/`
- V2: 同上，但 fixture 文件单独放

---

### 模式 B：PRD → Eval Cases

#### B-Step 1：读取并理解 PRD

读取用户提供的 PRD/spec 文档，提取以下信息：
- 这个功能让 agent 做什么？（核心行为）
- 涉及哪些工具/API 调用？
- 有哪些边界条件、异常流程？
- 用户交互的入口是什么？（用户怎么触发这个功能）

#### B-Step 2：推导失败模式

对 PRD 中每个核心行为，从以下角度推导"可能出什么问题"：

| 角度 | 问自己的问题 |
|------|------------|
| 不做 | agent 有这个能力但不用（如：该搜不搜、该调工具不调） |
| 做错 | agent 做了但结果错（如：发错人、时间搞错、编造信息） |
| 做多 | agent 过度执行（如：情感场景调任务工具、主动推送太频繁） |
| 泄露 | agent 暴露了不该暴露的（如：内部术语、用户隐私、其他人的信息） |
| 认错 | agent 搞错了身份/上下文（如：把别人当用户、把过期信息当当前） |
| 脏上下文 | 在 context 被塞满后，这个功能还能正常工作吗？ |

#### B-Step 3：筛选 + 排优先级

不是所有失败模式都值得做成 case。筛选标准：
1. **真实性**：这个问题用户真的可能遇到吗？
2. **严重性**：出了这个问题用户会不会很崩溃？
3. **可测性**：这个问题能用 eval 抓到吗？（纯 latency 问题可能抓不到）

输出时按优先级排序，建议先做高优的。

#### B-Step 4：批量输出 Cases

对筛选出的每个失败模式，输出结构化 case（格式同模式 A 的 Step 3）。

一篇 PRD 通常产出 5-15 个 cases。输出时：
1. 先给一个总览表（ID、维度、一句话描述、tier、V1/V2）
2. 再逐个展开详细格式
3. 最后更新覆盖率

#### B-Step 5：更新覆盖率

同模式 A 的 Step 4。

## 设计原则

> **评结果，不评手段。**
> Eval case 评的是 agent 回复的最终质量（用户感知），而非内部实现路径。

- 工具调用断言只在"调了明显不该调的工具"时使用（如情感场景调 create_task）
- 大多数情况用 LLM judge rubric，写清楚 PASS/FAIL 的边界
- 不要把"应该调 search"写成硬性要求，除非不调就一定答错

## 关于"脏上下文"

真实 bad case 大多出现在脏上下文下。如果用户描述的场景涉及：
- 长对话（10+ 轮）
- proactive 中断
- 大量 tool call 返回塞满 context
- NO_REPLY 残留

则应该标记为 V2 case，并在 fixture 中构造足够"脏"的上下文（不是简单几轮闲聊，而是真实的 tool output JSON、search results、calendar responses 等）。

## 参考文档

- 完整 eval framework：`/Users/suki/suki/project/byoa/eval/today-agent-evaluation-framework.md`
- Obsidian 副本：`/Users/suki/Documents/Obsidian Vault/Projects/today/today-agent-evaluation-framework.md`
- today-cloud eval harness 格式：`evals/AGENT_CASE_FORMAT_AND_GRADER.md`（todayai-labs/today-cloud repo, dev branch）
