---
title: "从「一句话」到「一直跑」：Agent 是怎么学会自己干活的"
date: 2026-07-03
tags: ["Agent", "LLM", "ReAct", "LangGraph", "AutoGPT", "多Agent协作", "AI开发"]
author: "Shamnick"
---

# 从「一句话」到「一直跑」：Agent 是怎么学会自己干活的

你盯着终端。Agent 已经跑了 47 轮。每一次都输出「Thought: I need to search for more information」，然后搜出来的东西跟上一次一模一样。你看了眼 token 计数器——烧了 200 万 token 了。任务是个简单的「帮我整理这周的 GitHub issues」。它好像卡住了，又好像没有。

这不是你的错。这是 Agent Loop 设计里最经典的坑。我写这篇文章就是想聊聊这件事：Agent 是怎么从「你问一句它答一句」变成「你给一个目标它自己一直跑直到跑通或者跑崩」的。以及，你现在能用的那些框架，它们的 Loop 到底长什么样。

---

## ReAct 之前：Prompt 是唯一的手段

2022 年，想让 GPT 做复杂任务只有一个办法：把提示词写得更聪明。

Chain-of-Thought（Wei et al., 2022）让人在 prompt 里加一句「Let's think step by step」，模型的推理能力就显著提升。Self-Consistency 更进一步：采样多条推理路径，多数投票——但本质还是单次调用。

Toolformer（Schick et al., 2023）往前迈了一步：让 LLM 自己学会调用搜索引擎和计算器。但调用是嵌入在单次推理里的，不存在「调完再想」这回事。

这一整年的共同瓶颈很明确：**一次 forward pass 的计算量是固定的**。你 prompt 写得再花哨，有些任务天然需要多步——先查资料、再理解、再查、再推翻之前的结论。这在单次推理里根本做不到。

---

## ReAct：一个从没被充分讨论过的转折点

2022 年 10 月，Yao et al. 发了 ReAct。在当时看起来只是另一篇 prompt engineering 论文，现在回头看，它干了件很夸张的事：**把「推理」和「行动」从两个阶段变成同一个循环**。

```python
# ReAct 的核心模式
context = [f"Question: {question}"]

while True:
    response = llm(context + ["Thought:", "Action:"])
    
    if "Finish" in response.action:
        return response.final_answer
    
    observation = execute_tool(response.action)
    context.append(f"Observation: {observation}")
```

这十几行代码捅破了一层窗户纸：Agent 不需要在「把所有事情想清楚」和「开始干活」之间二选一。它可以边想边干。每一步都带着上一步的观察作为新的推理依据。

**为什么这件事重要？** 因为 ReAct 把 LLM 从一个「被动回答器」变成了一个「主动探索器」。它不追求一次给出最优答案，而是追求每一步都让下一步更有可能找到答案。这个思维转变直接催生了后面的一切——AutoGPT、LangGraph、Claude Code。

---

## 2023 年春天：Agent 寒武纪

ReAct 播下的种子在 2023 年 3 月到 6 月集体发芽。

AutoGPT 3 月 30 号上线 GitHub，150k+ stars。它做的事说起来也不复杂：给定一个目标，自动分解、搜索、执行、迭代。但用户体验是颠覆性的：你不需要写 prompt 了，只需要说「帮我调研这个市场」。Agent 自己去想该搜什么、搜完怎么归纳、归纳完下一步做什么。

同一时间，BabyAGI 从另一个角度验证了同一件事：把任务队列、优先级排序、执行、结果评估塞进一个循环，用极简代码跑通了「目标驱动的持续执行」。斯坦福的 Generative Agents 论文（Park et al., 2023）则从学术上奠定了 Agent 记忆架构：Observation → Reflection → Planning。

**这个阶段最重要的发现是：LLM 不只是推理引擎，更是决策引擎。** Loop 成为 Agent 的骨架——`Observe → Think → Act → Observe`。而且人们发现，这个骨架可以很薄。AutoGPT 第一版的核心循环不到 100 行 Python。

---

## 框架怎么解决 Loop 的三个核心问题

热乎劲过去之后，真问题浮出水面。有状态管理、可中断恢复、多 Agent 协作——这三个问题不解决，Agent Loop 就只是 demo。

### 问题一：循环怎么停

最简单的解法是 `max_steps`。硬上限。AutoGPT 和 BabyAGI 都这么干。问题是硬上限永远不是「刚好」——要么太短任务没做完，要么太长浪费 token。

Reflexion（Shinn et al., 2023）换了个思路：加一个 Evaluator 来判断「任务做完了吗」。这个 Evaluator 可以是 LLM，也可以是规则。Hermes Agent 的 Goal Mode 把这套逻辑工程化了：每个 turn 之后，一个独立的 judge 模型检查目标完成度，未完成且预算够就继续，否则停。

**为什么重要？** 依赖 LLM 自己判断「是否完成」有个经典坑：LLM 可能永远觉得不够完美。Reflexion 式的「外部 judge」比「自判断」可靠得多。

### 问题二：上下文怎么管理

每轮调用工具，工具返回结果就要塞进下一轮的上下文。跑 20 轮之后，对话历史里 90% 是工具输出。三种常见策略：

- **压缩**：Hermes Agent 在接近 context limit 时自动摘要历史
- **滑动窗口**：AutoGPT 只保留最近 N 轮
- **选择性检索**：Generative Agents 的 Memory Stream——存进向量库，每次只检索相关的

实际落地时通常是混合的：滑动窗口保证不炸，摘要保证不丢重要信息。

### 问题三：中断了怎么办

LangGraph 的 Durable Execution 解决了这件事：每个节点执行后自动 checkpoint。Agent 跑了 3 小时崩了，从 checkpoint 恢复，不用从零开始。Hermes Kanban 的持久化是另一个思路——任务状态存在 SQLite 里，dispatcher 重启后接着扫描，哪个 worker 挂了就重新 dispatch。

---

## 五种真实世界的 Loop 实现

### 1. 手写 while 循环（AutoGPT 风格）

```python
def agent_loop(goal, max_steps=100):
    task_list = [goal]
    memory = []
    
    for _ in range(max_steps):
        task = prioritize(task_list)
        response = llm(system="You are an autonomous agent...",
                        context={"goal": goal, "memory": memory, "task": task})
        result = execute(response.action, response.params)
        memory.append({"task": task, "result": result})
        if response.done:
            return result
```

简单直接，但缺少收敛保证和错误恢复。适合快速实验，不适合生产。

### 2. ReAct 模式

```python
# Thought → Action → Observation 交替
def react_loop(question):
    context = [f"Question: {question}"]
    
    while True:
        response = llm(context + ["Thought:", "Action:"])
        if "Finish" in response.action:
            return response.final_answer
        observation = execute_tool(response.action)
        context.append(f"Observation: {observation}")
```

一步包含一重推理，用轨迹（trajectory）替代单次最佳答案。所有现代 Agent 框架的底层都是这个模式的变体。

### 3. 有状态图（LangGraph）

```python
from langgraph.graph import StateGraph

class AgentState(TypedDict):
    messages: list
    next_action: str

graph = StateGraph(AgentState)
graph.add_node("think", llm_reason)
graph.add_node("act", execute_tool)
graph.add_conditional_edges("think", 
    lambda s: "act" if s["next_action"] == "tool_call" else "finish")
graph.add_edge("act", "think")
```

循环逻辑被分离到路由/边中。LLM 只关注当前节点决策。支持 checkpoint、human-in-the-loop、嵌套子图。

### 4. 带断路器的生产级 Loop

```python
def safe_agent_loop(goal):
    last_actions = deque(maxlen=5)
    state_hash = None
    stale_steps = 0
    
    while step < max_steps:
        current_action = agent.decide(state)
        
        # 重复检测：Agent 在兜圈子
        if list(last_actions).count(current_action) >= 3:
            raise LoopDetectedError("重复动作检测")
        
        result = agent.execute(current_action)
        
        # 状态停滞检测
        new_hash = hash(str(state))
        stale_steps = 0 if new_hash != state_hash else stale_steps + 1
        if stale_steps > 10:
            raise StagnationError("Agent 状态停滞 10 轮")
```

生产环境的两道防线：重复动作检测 + 状态停滞检测。少了任何一道，Agent 都可能静默卡死。

### 5. 多 Agent 调度循环（Kanban Swarm）

```
Dispatcher Loop:
  while True:
    扫描 SQLite 看板 → 找到 ready 任务
    → atomic claim → 启动 worker profile
    → worker 内部跑 Agent Loop
    → worker complete/block
    → 依赖引擎自动晋升下游任务
    → 继续扫描
```

从「一个 Agent 的循环」跳到「一群 Agent 的循环」。dispatcher 本身是一个持续运行的循环，但循环体不是 LLM 调用——是看板状态机。

---

## 从单 Agent 到多 Agent：Loop 的跃迁

单 Agent Loop 的问题是物理上限。一个 LLM 一次只能关注一件事，context window 再大也有用完的时候。

多 Agent Loop（Swarm）把「一个大脑的内部循环」外化成「多个大脑的社会协作」。LangGraph 的 subgraph、CrewAI 的 Crew、Hermes 的 Kanban Swarm 都是这个思路的实现。

带来的好处很直接：每个 Agent 可以配专属工具和 prompt，多个 Agent 可以同时处理不同子任务。代价是协调成本——Agent 之间的通信协议、任务依赖管理、死锁检测。

MetaGPT 提供了一个有意思的视角：把软件工程的 SOP（标准作业流程）硬编码为多 Agent 流水线。Product Manager → Architect → Engineer → QA，每个角色输出标准化的结构化文档。循环发生在上游和下游之间——QA 发现问题 → Engineer 修复 → 再 QA。这不只是一个 Agent 在循环，而是整条流水线在循环。

---

## 十条实践原则

以下是从三个年份、五个框架、无数次「Agent 又卡了」的血泪里总结出来的。

1. **从 ReAct 开始。** 别上来就画复杂的状态图。多数场景一个 ReAct 循环就够了。
2. **永远有硬上限。** max_steps、timeout、token budget，三选二。Agent 可能在你睡觉的时候烧完所有余额。
3. **别让 LLM 自己判断「做完了没」。** 加一个独立 judge 或者结构化校验——LLM 永远觉得还能再改改。
4. **工具调用要幂等。** Agent 可能在 loop 里调同一个工具好几次。非幂等的工具调两次可能搞崩整个状态。
5. **日志记录每一步。** 出问题的时候你要能回放轨迹，否则 debug 靠猜。
6. **上下文管理要分层。** 工作记忆（当前窗口）≠ 长期记忆（向量库）。混一起的结果是 token 成本 O(n²)。
7. **先验证再行动。** 多 Agent 场景加一个 verifier gate——比如 Kanban Swarm 里 verifier 先审 researcher 的输出再交给 writer。
8. **失败要体面。** 工具调用失败不应该炸掉整个循环。每个工具都应该有 fallback。
9. **用对抗场景测试。** 给 Agent 矛盾指令、全挂的工具、不可能完成的目标。看它怎么反应。
10. **生产环境一定留人类入口。** 全自动 Agent 做了坏事，你要能按停止键。LangGraph 的 interrupt、Hermes 的 block/unblock 都是这个用途。

---

## 七个反模式

| 反模式 | 症状 | 解法 |
|--------|------|------|
| 循环无上限 | Agent 永远不返回 | max_steps + timeout |
| LLM 自判断收敛 | 永远不满意 | 外部 judge 或结构化检查 |
| 全量历史传递 | token 炸了 | 滑动窗口 + 摘要压缩 |
| 无状态追踪 | 中断后从零来 | checkpoint / 持久化 |
| 工具结果不截断 | 100KB 返回全塞进下一轮 | 限制工具返回长度 |
| 吞错误 | Agent 基于错误假设继续跑 | 显式错误传播 |
| 过早抽象 | 上来就搞多 Agent 架构 | 从 ReAct 开始，按需加复杂 |

---

## 终点是什么

2025 年有个越来越明显的趋势：**最好的 Loop 是你看不见的 Loop。**

AutoGPT 时代，Agent 每一步都喊出来：「THOUGHTS: I should search...」「REASONING: This result suggests...」。Claude Code 时代，你看到的是 diff 而不是思考过程。Manus 更进一步——show, don't tell。

OpenAI 的 o 系列模型在内部干了件更有意思的事：模型回答前自己跑隐式的 Chain-of-Thought。这不就是一个内置的 Loop 吗？MCP（Anthropic）和 A2A（Google）在标准化 Agent 与工具、Agent 与 Agent 的通信协议。三条线汇在一起：推理时隐式 Loop + 标准化工具协议 + 标准化 Agent 间通信 = Agent Loop 变成基础设施，不再是应用层要关心的事。

未来的 Agent 开发大概会是这样：你描述目标，框架自动选 ReAct 还是 LangGraph 还是 Swarm，Agent 在你看不到的地方想、搜、写、验证、修。你只需要看结果。Loop 变成了编译器——没人在乎编译器内部怎么做寄存器分配，但它不工作的时候所有人都急。

---

*本文基于 Hermes Agent Kanban Swarm 调研流水线合成：两名 researcher 分别完成「技术演进脉络」和「Loop 模式实现」调研，经 verifier 审核后由 writer 合成。所有引用的论文、框架和代码模式均来自调研文档中标注的原始来源。*
