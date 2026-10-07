---
title: "当 subagent 不可靠：Kanban Swarm 的持久化编排拆解"
date: 2026-10-07T16:29:23+08:00
tags: ["hermes", "kanban", "multi-agent", "swarm", "orchestration"]
author: "swinstall"
---

## 一、先问一个让你肉疼的问题

你有一个「并行研究 → 汇总成稿」的任务。用 `delegate_task` 起三个 subagent 干活，跑到一半，父进程 OOM 或者你手滑 `Ctrl-C` 了——三个子任务的上下文、产出、中间结论，全部没了。没有重跑入口，没有审计，你只能重来一遍。

这不是配置问题，是模型选错了。官方文档把这两件事的差别讲得很直白（kanban.md:112）：`delegate_task` 是一次函数调用（fork → join，父任务阻塞等返回，子任务是匿名 subagent，失败即失败）；Kanban 是一条工作队列，每次交接都是一行 SQLite 记录，任何 profile（或人）都能读、能写、能续跑。

三个有据可查的痛点，对应三条机制：
- 进程内 subagent 脆弱（父进程一死全丢）→ Kanban 用外部进程 + SQLite 行兜底；
- 「需要人介入」的任务无处停放（delegate_task 不支持人在环）→ Kanban 有 `blocked` 列 + comment 作为跨 agent 协议；
- 事后复盘难 → `task_runs` 表每次 claim 记一行，尝试历史永久留痕。

## 二、持久化模型：为什么 claim 必须是原子的

每个任务是一条 `tasks` 表的行。真正的调度核心不在「状态字段」上，而在一条 compare-and-swap 上。dispatcher 认领任务不是「先读 status 再改成 running」，而是：

```sql
UPDATE tasks SET status='running', claim_lock=?, claim_expires=?
WHERE id=? AND status='ready' AND claim_lock IS NULL
```

这条 UPDATE 跑在 `BEGIN IMMEDIATE` 事务里（kanban_db_connect.py:1191）。SQLite 单写者语义保证「至多一个赢家」——`rowcount != 1` 就返回 None，代表丢掉了竞争。为什么重要：多个 dispatcher（比如 gateway 内置的 + 你误启的 `hermes kanban daemon`）同时抢同一张卡时，不可能双认领；claim 的原子性正是「任务不丢失、不重复执行」的根。

第二层关键是依赖：`task_links` 表一行就是一条 parent→child 依赖边（对外表现为任务的 `parents`），它同时干两件事：一是调度门——子任务只在所有父任务 `done` 之后才从 `todo` 提升为 `ready`（父依赖未满足时 claim 被拒，写 `claim_rejected` 事件并降回 `todo`）；二是上下文交接通道——子任务 worker 的上下文里含一段 `## Parent task results`，逐字带入每个父任务的 completion `summary` 和 `metadata`。依赖既是闸门，也是数据管道。

再往下是 dispatcher 的调度循环。一次 tick 的顺序固定：先 reclaim 掉 stale/crashed 的 running 任务，再把 `todo` 提升为 `ready`，最后原子地 claim 每个可 spawn 的 ready/review 行并调用 spawn。spawn 不是库内函数，而是起一个完整的 OS 进程——`hermes -p <profile> --cli ... chat -q "work kanban task <task.id>"`——并注入 `HERMES_KANBAN_TASK`/`HERMES_KANBAN_WORKSPACE` 等环境变量。源码里有一行值得记住：spawn 时 `env.pop(DELEGATED_CHILD_ENV_MARKER)`，注释写着「This is the grant boundary: the dispatcher assigned this new worker's task」。这是 board worker 与 `delegate_task` 匿名子任务之间的权限分界。

## 三、Swarm v1：一个写进 SQLite 的静态 DAG

`hermes kanban swarm` 做的事情比它听起来朴素得多。源码 docstring 说得很诚实：刻意没有第二个调度器——它只是把一张小任务图写进已有的 Kanban kernel。

```text
root (建图即 done)
 ├─ worker A (ready)  ┐
 ├─ worker B (ready)  ┘→ verifier (todo) → synthesizer (todo)
```

四层的职责：
- root：创建为 `blocked`，随后在同一事务内被一条内联 CAS 翻成 `done`。它立即完成是为了放行并行 worker，同时继续充当共享黑板与审计锚点。
- 并行 worker：每个 `--worker` 一张卡，`parents=[root]`，初始 `ready`。
- verifier：`parents=[所有 worker]`，skills 硬编码为 `requesting-code-review`，body 要求「只有证据充分时才以 `metadata {"gate":"pass"}` 完成，否则 block 并写清缺失项」。
- synthesizer：`parents=[verifier]`，skills 硬编码 `humanizer`，body 要求「在 verifier 通过门禁前不要开始」。

共享黑板不是新服务，而是根卡上的一串结构化 JSON 注释，前缀 `[swarm:blackboard] `。建图后立即写一条 `key="topology"`，值为 `{root_id, worker_ids, verifier_id, synthesizer_id, goal}`；worker 往根卡追加 `{"key","value"}` 注释，同 key 后者覆盖前者，作者被记进 `_authors` 以便追责。好处是：dashboard、notifier、slash 命令、dispatcher 全部照常工作，因为状态就是 `task_comments`/`task_events` 里已有的行。

最需要注意的一点：`kanban_swarm.py` 全长 300 行，只做建图、黑板读写和参数解析，没有任何 swarm 层的失败/重试/回滚逻辑。所以 Swarm 的失败语义 = 底层 Kanban 的通用语义：某个 worker block 了，verifier 的 `parents` 里还有未 done 的卡，整条链就停住，等通用 dispatcher 的 `failure_limit`（默认 2 次）熔断或人介入。Swarm 是一张一次性写死的静态 DAG，编排智能发生在建图之前（人写 `--worker` 列表），或之上（另起一张 orchestrator 卡去修图），而不是运行中自适应重规划。

## 四、CLI 实战：命令必须逐字可复制

先讲一个反直觉的事实：官方文档里给的 swarm 命令跑不通。文档写 `--workers researcher,architect,sre`，实测报 `unrecognized arguments: --workers a,b`。真实 flag 是可重复的 `--worker PROFILE:TITLE[:SKILL,SKILL]`。所以下面每条命令都以本机 `v0.21.5` 的 `--help` 逐字为准。

在给 swarm 命令前，先看它到底省了什么：swarm 只是 `create` + 父子依赖 + 一条黑板约定的语法糖。你可以手搓同一张图：

```bash
hermes kanban create "Root" --json                        # 实测 status=ready
hermes kanban complete <root_id> --result "plan ready"    # 关键：把 root 翻成 done，否则下面两张 worker 卡会永远停在 todo
hermes kanban create "Alpha worker" --assignee alpha --parent <root_id> --json
hermes kanban create "Beta worker" --assignee beta --parent <root_id> --json
hermes kanban create "Verify" --assignee verifier --parent <alpha_id> --parent <beta_id> --json
```

为什么重要：`--parent` 可重复传入，构成多入度依赖，产生的依赖边与 swarm 子命令一致；但 swarm 额外在你看不见的一步里，用同一事务 CAS 把 root 直接建成 `done`（手动版靠上面第二行 `complete` 补齐）。看懂这五行，你就明白 Swarm 没有魔法，只是把「建图 + 写黑板 + 原子激活」封装进一个事务。

swarm 子命令是它的快捷方式：

```bash
hermes kanban boards create probe-swarm --name "Swarm Probe"
hermes kanban --board probe-swarm swarm "Verify swarm topology end to end" \
  --worker alpha:"Alpha worker" \
  --worker beta:"Beta worker":humanizer \
  --verifier verifier-profile \
  --synthesizer synth-profile \
  --json
```

返回 JSON 带 `root_id / worker_ids / verifier_id / synthesizer_id`。为什么重要：建图是原子提交的，dispatcher 和 dashboard 读者要么看不到新 swarm，要么看到完整拓扑，绝不会看到半连接的图。

看拓扑与状态：

```bash
hermes kanban --board probe-swarm list
hermes kanban --board probe-swarm show <root_id> --json   # 读黑板：解析 [swarm:blackboard] 前缀注释
```

`list` 里你会看到 root `done`、两个 worker `ready`、verifier 和 synthesizer 都是 `todo`——这就是父依赖门禁在起作用：verifier 要等两个 worker 都 done，synthesizer 要等 verifier。

手动推一轮 dispatcher（不依赖 gateway 的 60 秒 tick）：

```bash
hermes kanban --board probe-swarm dispatch --dry-run --json
```

为什么重要：dispatcher 默认跑在 gateway 进程内（`kanban.dispatch_in_gateway: true`），单次 `dispatch` 让你不启动 gateway 就能观察「reclaim stale → promote ready → claim spawn」的完整一 tick，且 `--dry-run` 不产生副作用。

清理：

```bash
hermes kanban boards rm probe-swarm --delete
```

为什么重要：dispatcher 每 tick 会扫描**所有** board，探针板用完即删可避免残留的空板被反复扫入调度范围。

## 五、什么时候别用 Swarm

- 父 agent 只要一个短推理答案、无人参与、结果要回灌自己上下文 → 用 `delegate_task`，别上队列。
- 工作单元 < ~10 且是串行链 → 直接做，fan-out 是过度设计。
- 需要运行中自适应改图、动态重规划 → Swarm v1 做不到，它是静态 DAG；你得自建 orchestrator 卡。
- 需要跨主机共享同一块板 → 不支持，Kanban 是 single-host 设计。
- 想要「集群级 OS 隔离」→ 不是，文档自陈它只是 single-user lifecycle guard，不防对 DB 的直接写。

## 六、小结

Kanban 的 claim 原子性 + `parents` 门禁，把「任务不丢、不重复、依赖不裸奔」这些调度里最难的部分交给了 SQLite 的单写者语义，而不是某个调度器的内存状态。Swarm v1 则把这种能力收敛成一张静态 DAG，用一个共享黑板替代第二个调度器。它的价值不在「智能重规划」，而在「把并行 worker、守门人、成稿人的交接全部落到可审计的持久行上」。真要用它做重规划，别指望 `swarm` 子命令——自己写一张 orchestrator 卡，在图上做 `link`/`unblock`，那才是这层抽象的边界。
