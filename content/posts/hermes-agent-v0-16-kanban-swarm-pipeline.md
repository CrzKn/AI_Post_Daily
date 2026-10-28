---
title: "Hermes Agent v0.16 的 Kanban Swarm：把多 Agent 协作变成一条可恢复的流水线"
date: 2026-10-28T18:58:00+08:00
tags: ["hermes-agent", "multi-agent", "kanban"]
author: "CrzKn"
---

## 开场：你的 agent 卡在第 7 步

你给 agent 布置了一个多步的活：抓 20 篇论文、逐篇摘要、交叉比对、写综述、配图、推仓库。前六步顺风顺水，第七步它开始「遗忘」——上下文窗口被塞满，它把第二步的结论悄悄改成了另一个意思；你想中途改一句需求，唯一的办法是从头再跑一遍。四十分钟白烧，token 照扣。

问题不在模型不够强，而在「一个 agent 串行干完」这个假设本身：任务一旦超过一个上下文窗口、或者需要中途改需求，串行 agent 就不再省钱，而是烧钱。

Hermes Agent 给出的答案是 Kanban Swarm。先说清版本事实，免得被标题带偏：Kanban 看板本身首进正式版是 v0.13.0（重落地提交 2026-04-30，[reland #17805](https://github.com/NousResearch/hermes-agent/commit/c868425467)），[`hermes kanban swarm` 拓扑助手](https://github.com/NousResearch/hermes-agent/commit/3ee7a5546d)首进 v0.15.0（提交 2026-05-18），而 [v0.16.0](https://github.com/NousResearch/hermes-agent/releases/tag/v2026.6.5)（2026-06-06，tag v2026.6.5）补上的是 `goal_mode`、任务附件、并发上限与 run 终止接口。它要解决的，正是把多 agent 协作写成一条可恢复、可审计、可中途插人的持久流水线。下面按「是什么 → 怎么跑 → 实战坑 → 什么时候别用」的顺序拆开讲。

## Swarm 是什么：把协作写成状态机

**结论先说：Kanban Swarm 的价值不在「更多 agent」，而在把一次性会话变成持久化、带状态的任务队列。** 看板重落地提交的标题就是「[durable multi-profile collaboration board](https://github.com/NousResearch/hermes-agent/commit/c868425467)」——每张卡片是一个进程，而不是一条待办。

### 三个原语：卡片、依赖边、dispatcher

- **卡片（card）**：核心是**带状态的进程**，状态全集是 `triage | todo | ready | running | blocked | review | done | archived`。卡片不是「要做的事」，它知道自己在哪一步、谁在跑、跑了多久、上次失败因为什么。
- **依赖边（parent→child link）**：父卡完成的 `summary` + `metadata` 会原样注入子卡的上下文，子卡在 `## Parent task results` 里逐字看到。所以父链是**数据契约**，不只是先后顺序。
- **dispatcher**：唯一的调度者，按 60 秒一个 tick 扫看板、派活、回收失联 claim。

这三样合起来，让「多 agent 协作」从「你在脚本里手动 fork 几个进程」变成「往一张表里插状态，让调度器自己跑」。

### 和 delegate_task、多开进程的边界

最容易混淆的是三种做法：`delegate_task`、手动多开 `hermes` 子进程、Kanban swarm。

[`delegate_task`](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation)是一次性 RPC（fork→join），子 agent 匿名、无持久状态，进程重启就没了，也不能中途插人。多开 `hermes` 子进程是独立会话，进程活着它就在，但没有统一的状态记录，失败只能手动重跑。Kanban swarm 是持久队列 + 状态机：具名 profile、独立 OS 进程、重启后可 block→unblock→重跑、任意时刻 comment 插人、`failure_limit` 自动重试。一句话：**前两个是「调用」，swarm 是「运营」。**

这个边界直接决定成本：`delegate_task` 默认并发 10，一次性调用另受 `oneshot_max_children`（默认 2）约束；swarm 的并发则由 `kanban.max_in_progress` 决定，而且能跨小时存活。

## 运行机制：一次任务在 swarm 里怎么走完

**结论：dispatcher 只做「派活」这一件事，所有状态迁移都落在 SQLite 里，谁都能中途介入。**

### 谁在派活：dispatcher 与 claim

dispatcher 跑在 gateway 内，配置键与默认值如下（出处：[docs Kanban「Dispatcher」](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban)与源码 [kanban_watchers_dispatcher.py](https://github.com/NousResearch/hermes-agent/blob/main/gateway/kanban_watchers_dispatcher.py)）：

```
kanban.dispatch_in_gateway          = true   # dispatcher 在 gateway 内跑
kanban.dispatch_interval_seconds    = 60     # 扫描间隔，下限 1 秒
kanban.max_in_progress              = 未设    # 看板级并发上限，未设=无限
kanban.max_in_progress_per_profile  = 未设    # 单 profile 并发上限
kanban.failure_limit                = 2      # 连续 spawn 失败 2 次自动 block
HERMES_KANBAN_CLAIM_TTL_SECONDS     = 900    # claim 租约 15 分钟（环境变量，非 config）
```

注意：Kanban worker 的并发**不由** `delegation.max_concurrent_children`（默认 10）决定——那个键只在 worker 内部再调 `delegate_task` 时生效。所以你要控 swarm 的并发，改的是上面两个 `max_in_progress` 键。

### worker 的一生：claim → heartbeat → complete / block

worker 被 spawn 后，先 claim 一张卡（租约 15 分钟，键 `HERMES_KANBAN_CLAIM_TTL_SECONDS`），然后靠 heartbeat 续命。心跳断了，dispatcher 会在 `stranded_threshold_seconds`（默认 1800 秒，见 [kanban_diagnostics.py](https://github.com/NousResearch/hermes-agent/blob/main/hermes_cli/kanban_diagnostics.py)）后把卡判为 stranded 回收，重新派活。正常收尾是 `kanban_complete`；做不下去就 `kanban_block`。

**所以你要注意两件事**：一是长任务必须有心跳，超过一小时不发心跳会被回收重跑；二是收尾状态只有「complete / block / request_review」三条路，没有「我猜完成了」的中间态。补充一句：进程 crash 不等于 done——崩了卡会走 reclaim 重派，但退出码 78 这类 provider 认证失败实测会停在 `gave_up` 等人工介入（见第四节）。

### 卡住了怎么办：四种 block 与 review 通道

`kanban_block` 有四种 kind，路由不同（源码 [kanban_db.py](https://github.com/NousResearch/hermes-agent/blob/main/hermes_cli/kanban_db.py) 的 `VALID_BLOCK_KINDS`）：`dependency` → 回 `todo`，父卡完成自动恢复；`needs_input` / `capability` / `transient` → 进 `blocked`，等人工。同一种 kind 反复 block→unblock 累计到 `BLOCK_RECURRENCE_LIMIT`（默认 2 次），就送进 `triage`。

关键区别：[`kanban_request_review`](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban-worker-lanes)进 `review` 列，**不计入** block 复发（原文「without touching block recurrence accounting」）；`kanban_request_changes` 同样不记。只有 `kanban_block` 累加。所以「找人审」用 review，「真的被卡」才用 block。

还有一个坑：别把支援卡 link 到它要解救的被阻塞父卡下面——那样会死锁；pre-created 的下游卡，父卡也应在完成时直接 `kanban_complete` 释放，否则 review 与依赖互相等。

## 拓扑实战：一条 5 卡片的博客流水线

**结论：卡片怎么切、父链怎么连，决定了吞吐——本拓扑实测是一条串行链，不是并行 swarm。**

### 我是怎么切的：拆解 / 调研 / 写作 / 审校 / 发布

五张卡，每张一个 profile：orchestrator 拆任务 → researcher 查事实 → writer 写正文 → reviewer 审校 → publisher 发布。因为是串行接力，每一棒只依赖上一棒，吞吐被最慢那一棒锁死。拓扑如下：

```
wechat-blog-swarm (created_by)
└── t_cac205e5 orchestrator [done]
    └── t_d38cf5c8 researcher [done]
        └── t_529ca900 writer [done]
            └── t_dc7072e3 reviewer [done]
                └── t_2f780b85 publisher [todo]
```

父链交接走的是 `kanban_complete(summary=..., metadata=...)`：结构化事实放 `metadata`，文件路径放顶层 `artifacts`，人读的说明放 `summary`。只有顶层 `artifacts` 参数会被内核在清理 scratch 前复制进持久附件存储（单文件 25 MB），写进 `metadata.artifacts` 的路径只是普通文本，内核不搬文件——这是本流水线最重要的交接约定。

### 跑起来暴露的三件事

第一，**这是链，不是树**。`task_links` 里 4 条边、每父仅 1 子，没有 fan-out/fan-in，吞吐 = 最慢那张卡。真正的 swarm 形状是根卡挂 N 张并行 worker，再 verifier→synthesizer 收口。别被「swarm」这个词骗了：卡片数多不等于并行。要真并行，得把可并行的棒挂到同一张根卡下做 fan-out。

第二，**toolset 裁剪是边界，也是交付风险**。orchestrator 卡只开了 `kanban_*` 工具，没有 `write_file`，`plan.md` 没能落盘，4733 字符的计划只能走 comment + metadata 通道。这是「防越权」的设计权衡，不是 bug；对策是建卡时在 body 里声明依赖哪类工具，要写文件的卡别用只留 `kanban` 的 profile。

第三，**失败语义和文档对不上**。researcher 卡实测撞上退出码 78（provider 认证失败），走的是 `crashed → gave_up → ready`，靠人工 unblock 才恢复（16:52:39 崩、17:06:07 恢复，空 13 分钟），与源码注释「首次即 park blocked」不符。这是实测观察，不是文档保证——别默认文档里的失败语义就是线上行为。

## 什么时候不该用 Swarm

**结论：swarm 的固定开销是「状态维护 + 派活轮询」，任务太小或交互太强时，这笔开销是纯亏。**

三条明确的别用判据：

1. **单文件小改动**：改个函数名、调个常量，一次会话就能干完，切卡片、建父链、等 tick 反而更慢。
2. **强实时交互**：需要你和 agent 你来我往的调试，Kanban 是异步队列，60 秒才扫一次，交互节奏对不上——你问一句要等下一个 tick 才有回应。
3. **无产物交接的纯问答**：没有要在卡片间传递的文件或结构化事实，父链数据契约完全用不上，`delegate_task` 或单次会话更合适。

一句话：swarm 解决的是「跨上下文窗口、要中途改需求、要可审计地并行」的问题，不是「更快地答一个问题」。

## 落地清单：第一次用 swarm 要做的 5 个决定

1. **切几刀**：按「能独立完成、有明确产物」切卡，别为切而切。
2. **谁来干**：每卡一个 profile，工具集和身份要对得上（要写文件的卡别只开 `kanban`）。
3. **并发上限**：设好 `max_in_progress` 与 `max_in_progress_per_profile`，别让 swarm 失控。
4. **交接约定**：文件走顶层 `artifacts`，事实走 `metadata`，多卡共用素材放 `dir:<绝对路径>` 而非 scratch。
5. **失败预案**：明确哪种 block 用哪种 kind，长任务保证心跳，撞热点就留 `hotspot:` 注释。
