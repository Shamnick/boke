---
title: "别再写胶水代码了：Hermes Agent Kanban Swarm 如何用一张看板编排多 Agent 协奏曲"
date: 2026-07-02
tags: ["Hermes Agent", "Kanban", "多Agent协作", "AI开发", "工作流"]
author: "Shamnick"
---

> 面向 AI 开发者 · 深度技术解析 · 2026-07-02

---

## 先问你一个问题

今天上午你要做三件事：

1. 调研五个开源 RAG 方案的性能对比
2. 写一篇技术博客总结调研结果
3. 把博客发布到 GitHub Pages

你打算怎么安排？手动一条条发指令给 AI？写个 Python 脚本调度子进程？还是干脆自己干？

我猜你试过 delegate_task（Hermes Agent 的内置并行工具）。它好用，但不持久：进程一退出，所有状态烟消云散。你不能离开电脑等它跑完，更不能让另一个人接手看到你做到哪了。

这就是 **Kanban Swarm** 要解决的问题。它只是在 Hermes Agent 现有的 Kanban 看板内核上铺了一层 **薄拓扑辅助层**。你敲一行命令，它自动创建一棵依赖树（根卡片、工作卡片、验证卡片、合成卡片），然后利用看板自带的依赖晋升引擎，一步步推动流程直到交付物生成。所有状态持久化在 SQLite 里，随时可查，随时可恢复。

---

## 一张图看懂 Swarm 拓扑

Kanban Swarm 的拓扑就长这样：

```
Swarm Root / Blackboard (done)
    ├─ Worker 1 (ready) — 调研 RAG 方案
    ├─ Worker 2 (ready) — 写博客初稿
    └─ Worker N (ready) — 更多并行任务
         └─ Verifier (todo, parents=[worker_1..N])
              └─ Synthesizer (todo, parent=verifier)
```

关键的数据流：

```
Worker A → kanban_complete(summary, metadata)
Worker B → kanban_complete(summary, metadata)
               ↓（所有 worker done → verifier 自动晋升到 ready）
Verifier  → kanban_complete(metadata={"gate": "pass"})
               ↓（verifier passed → synthesizer 自动晋升到 ready）
Synthesizer → kanban_complete(summary="最终交付物")
```

### 没有第二调度器

这张图的核心看点是：**没有第二调度器**。它复用了 Kanban 内核的「依赖晋升引擎」（也就是 `kb.recompute_ready()` 那个轻量级调用）。当一个 worker 完成时，引擎自动检查所有父依赖是否已清除，然后推进下游任务。一套代码既服务了日常的 TODO 看板，也支撑了多 Agent 协作流水线，没有任何重复投资。

---

## 一行命令启动一个 Swarm

先装好 Hermes Agent v0.17+，然后：

```bash
hermes kanban swarm "调研五个 RAG 方案的性能并写成博客" \
  --worker researcher:"调研 RAG 方案":web,arxiv \
  --worker writer:"撰写博客初稿":humanizer \
  --verifier reviewer \
  --synthesizer writer
```

输出四个 ID：

```
✅ Swarm created
  Root:         t_948b3802  (done)
  Workers:      t_d4ccef12  (researcher, ready)
                t_e8a0cb95  (writer, ready)
  Verifier:     t_425d49ec  (todo)
  Synthesizer:  t_627feefa  (todo)
```

`--worker` 参数格式：`profile:任务标题:skill1,skill2,...`

敲完这行命令你就可以关掉终端去喝咖啡了。dispatcher 会自动 pick up 每个 ready 卡片，spawn 对应的 profile worker，完成后自动推进下一步。你回来时，verifier 已经审完，synthesizer 正在合。

### 一行 vs 一堆脚本

对比一下传统的做法：你要写 Python 脚本、用 `subprocess` 启动子进程、手动管理进程间通信、处理崩溃重试……Swarm 把整个流程压缩成一行 CLI 命令。这行命令背后是一次 SQLite 事务创建 4+N 张卡片，每张卡片自带 body 说明、依赖关系和 Swarm Protocol 上下文注入。下游 worker 通过 `kanban_show()` 就能看到上游的 handoff 数据，**零配置**。

---

## 四大核心场景

### 场景 1：Pipeline 流水线——一人串行，各司其职

经典的前后端依赖开发：设计 Schema → 实现 API → 编写测试。

```bash
SCHEMA=$(hermes kanban create "Design auth schema" \
    --assignee backend-dev --tenant auth-project \
    --body "Design the user/session/token schema." --json | jq -r .id)

API=$(hermes kanban create "Implement auth API endpoints" \
    --assignee backend-dev --tenant auth-project \
    --parent $SCHEMA \
    --body "POST /register, POST /login, POST /refresh, POST /logout." --json | jq -r .id)

hermes kanban create "Write auth integration tests" \
    --assignee qa-dev --tenant auth-project \
    --parent $API \
    --body "Cover happy path, wrong password, expired token, concurrent refresh."
```

只有 `SCHEMA` 进入 `ready`，`API` 和 `tests` 老老实实等在上游完成。子 worker 通过 `kanban_show()` 自动获取父任务最近一次完成的 summary + metadata。

### 场景 2：Fleet Farming——并行执行的真正价值

当你有大量独立任务需要同类型 worker 处理时，Kanban 对比 `delegate_task` 的优势就出来了。delegate_task 需要你在父会话中 fork 子 agent，父进程一退就没了。Kanban 的 worker 是持久化的，dispatcher 逐个推进，中途机器重启都不怕。

```bash
# 三个翻译任务并行
for lang in Spanish French German; do
    hermes kanban create "Translate homepage to $lang" \
        --assignee translator --tenant content-ops
done
```

Dispatcher 在 gateway 中持续运行，一个 worker 完成后自动 pick up 下一个。仪表盘的 "Lanes by profile" 视图按 assignee 分组展示 In Progress 列。

### 场景 3：带重试的角色流水线

这是 Kanban 相比于扁平 TODO 列表的核心价值。PM 写规格 → 工程师实现 → 审查者拒绝 → 工程师重试 → 审查者批准。每一步都有自己的 run 历史：

```
Spec: password reset flow (DONE, pm)
  └─ Implement password reset flow (DONE after retry, backend-dev)
      └─ Review password reset PR (READY, reviewer)
```

实现任务的 drawer 显示两次尝试：
- **Run 1**: blocked — "password strength check missing, reset link isn't single-use"
- **Run 2**: completed — "added zxcvbn check, reset tokens are now single-use"

worker 用 `kanban_block(reason="review-required: ...")` 替代 `kanban_complete` 等待审查。审查者 unblock 后，dispatcher 重新 spawn worker，新的 run 自动读取 previous run 的上下文。

### 场景 4：熔断器与崩溃恢复

现实中的 worker 会挂：缺少凭据、OOM 崩溃、瞬态网络错误。Dispatcher 有两道防线：

**熔断器**：`--max-retries 3`，三次失败后标记为 `gave_up`，gateway 通知触发。

**崩溃恢复**：迁移任务扫描 240 万行 → OOM 在 ~230 万行杀死进程 → dispatcher 检测到死 PID 释放 claim → 任务回到 ready → worker 重试时看到 crash 记录，采用 chunked 策略成功。

### 场景覆盖度

这四个场景覆盖了 **80% 以上的多 Agent 协作需求**。从最简单的串行 pipeline 到需要人工介入的 review 流程，再到大规模并行农场和崩溃自愈。Kanban 用一种统一的数据模型（卡片 + 依赖 + run 历史）解决了所有问题。相比之下，delegate_task 只能覆盖场景 2 的一部分，而且没有持久化。

---

## 九大协作模式速览

官方文档定义了 9 种协作模式，全部通过现有的 parent/child 依赖原语实现，不需要新增任何 API：

| 模式 | 形状 | 典型场景 |
|------|------|---------|
| **P1 Fan-out** | N siblings 同角色 | 并行调研 5 个方向 |
| **P2 Pipeline** | 角色链 | 每日简报 scout→editor→writer |
| **P3 Voting/quorum** | N siblings + 1 aggregator | 3 研究员 → 1 审查者择优 |
| **P4 Long-running journal** | 同 profile + 共享目录 + cron | Obsidian 知识库自动维护 |
| **P5 Human-in-the-loop** | worker 阻塞 → 用户评论 → unblock | 需要人工决策的模糊场景 |
| **P6 @mention** | 从文本中内联路由 | @reviewer 看看这个 |
| **P7 Thread-scoped workspace** | 线程中的 /kanban here | 按项目隔离 gateway 线程 |
| **P8 Fleet farming** | 一个 profile，N 个主题 | 50 个社交账号内容生成 |
| **P9 Triage specifier** | 粗略想法 → triage → specify | 把一行描述变完整规格 |

### 模式组合的威力

这 9 种模式意味着 **你不用再发明协作范式了**。遇到任何多 Agent 场景，直接对号入座，用现成的原语组合出来就行。而且因为它们都建立在同一个 Kanban 内核上，模式之间可以自由组合。比如 P2 Pipeline 里嵌套 P1 Fan-out，或者 P5 Human-in-the-loop 与 P3 Voting 混用。

---

## delegate_task vs Kanban：选对工具

这是 AI 开发者最容易混淆的一点。两者都能并行执行任务，但哲学完全不同：

| 维度 | delegate_task | Kanban Swarm |
|------|--------------|--------------|
| **形态** | RPC 调用（fork → join） | 持久化消息队列 + 状态机 |
| **持久性** | ❌ 进程退出后丢失 | ✅ SQLite，跨进程存活 |
| **可观察性** | ❌ 仅在父会话上下文 | ✅ Dashboard/CLI/notifier |
| **跨 profile** | ❌ 同一进程内子代理 | ✅ 不同 profile 可协作 |
| **依赖门控** | ❌ 无自动依赖管理 | ✅ 父-子依赖晋升 |
| **审核** | ❌ 无审核门控 | ✅ Verifier 角色 |
| **延迟** | 毫秒级（进程内） | 秒级（dispatcher tick） |
| **适用场景** | 快速推理子任务 | 多轮多人协作工作流 |

**一句话概括**：delegate_task 是函数调用，Kanban 是工作队列。

### 那到底该怎么选？

- **需要即时结果、不需要持久化** → `delegate_task`。比如同时搜索三个 API 文档、并行做几个数学计算。
- **需要持久化、可观察、人工介入** → Kanban。比如多步骤开发流水线、需要审阅的内容生产、跨 session 的长期项目。
- **两者可以混用**：你在 Kanban worker 内部可以调用 `delegate_task` 来加速子任务。它们各自解决不同的问题，但可以很好地协同工作。

---

## 最佳实践清单（直接能用）

### 1. 结构化 Handoff

每个 worker 的 `kanban_complete(metadata=...)` 应该包含：

```json
{
  "changed_files": ["path/to/file.py"],
  "verification": ["pytest tests/ -q"],
  "dependencies": ["parent task id"],
  "blocked_reason": null,
  "residual_risk": ["未测试的内容或仍需人工审查的部分"]
}
```

问自己四个问题：改了什么？怎么验证？如果失败怎么解阻塞？哪些风险是故意的？

### 2. Worker 生命周期契约

```python
def worker_lifecycle():
    # 1. 启动时读取上下文
    kanban_show()
    # 2. 在 workspace 目录工作
    cd $HERMES_KANBAN_WORKSPACE
    # 3. 长任务发心跳（>1小时任务必须每小时至少一次）
    kanban_heartbeat(note="schema drafted, writing migrations now")
    # 4. 终止
    kanban_complete(summary="...", metadata={...})
    # 或
    kanban_block(reason="review-required: ...")
```

### 3. 心跳策略

默认 stale timeout 是 4 小时。最近 1 小时无心跳 → 回收（不增加失败计数器）。>1 小时的任务必须每小时至少一次 heartbeat。

### 4. 工作区类型选择

| 类型 | 生命周期 | 适用场景 |
|------|---------|---------|
| scratch（默认） | 任务完成时删除 | 一次性任务、调研、分析 |
| dir:/abs/path | 保留 | 共享目录、Obsidian 知识库 |
| worktree:repo | Git worktree | 代码实现任务 |

---

## 写在最后

Kanban Swarm 是我在 Hermes Agent v0.17 里最喜欢的特性。整个 `kanban_swarm.py` 才 278 行，但它把「多 Agent 协作」变成了一个依赖图问题。

没有新调度器，也没重写状态机。就是在现有的 Kanban 内核上加了一层创建函数。依赖晋升引擎是现成的，卡片持久化是现成的，仪表盘和 notifier 都是现成的。Swarm 只是把碎片拼成了蓝图。

这听起来可能不那么性感，但正好是我喜欢的设计：用最少的代码，解决最多的问题，而不是反过来。

如果你正在纠结「怎么让我的 AI agent 们好好地合作干活」，不妨从 `hermes kanban swarm` 开始试起。一行命令，一张看板，一个自动推进的工作流。剩下的事情，让看板帮你盯着就好。

---

> **参考资源**
> - Kanban 官方文档：https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban
> - Kanban 教程（4 个用户故事）：https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban-tutorial
> - Kanban Swarm 源码：GitHub `hermes_cli/kanban_swarm.py`
> - 262 个社区用例：https://hermes-agent.nousresearch.com/docs/user-stories
