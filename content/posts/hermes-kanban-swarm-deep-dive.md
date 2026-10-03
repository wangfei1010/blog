---
title: "Hermes Agent Kanban Swarm 深度解析：面向 AI 开发者的多智能体协作编排"
date: 2026-10-03
tags: ["AI", "Agent", "多智能体", "工程实践"]
author: "wangfei"
---

## 一个 Agent 撑不住的长任务

你把一份 8000 字的调研笔记丢给一个 Agent，让它顺手写完文章、校对、发 PR。四十分钟后你回来，它还在第七轮工具调用里打转，上下文里塞满了前面每一次网页抓取的原始快照——而你写在第一句里的验收标准，早被挤出了它的注意力范围。

单个 Agent 跑长任务，会稳定地在三处崩掉。第一处是上下文膨胀：所有中间产物都留在同一个 context window 里，越到后面越贵，也越容易忘掉早期约定。第二处是单点失败：一次限流、一次网络抖动，整条流水线从零重跑，成本直接翻倍。第三处是不可观测：你只看得到最终输出，看不到它此刻卡在哪一步、失败过几次、还在不在跑。

Hermes Agent 的 Kanban Swarm 用三件机制分别对冲：跨卡只传 summary 与 metadata，原始材料落盘成文件；每张卡有独立的 run 记录，连续失败 2 次就自动停下；`runs`、`log`、`context` 三条命令把中间状态摊在命令行里。理解这三条，比记住命令更重要。

## Kanban Swarm 的八个核心概念

| 概念 | 解决的问题 |
|---|---|
| board | 看板即硬隔离边界，每个项目一个 kanban.db |
| task | 一张卡，状态机共 9 态：triage、todo、scheduled、ready、running、blocked、review、done、archived |
| assignee | 承担这张卡的 profile，也就是“角色” |
| 父子依赖 | 决定这张卡何时 ready，同时是上下文交接通道 |
| dispatcher | 跑在 gateway 进程内的调度器，默认每 60 秒一跳 |
| claim | 原子认领，默认 TTL 15 分钟 |
| workspace | 每卡独立工作目录，scratch / dir / worktree 三选一 |
| run 日志 | 每次尝试一行 task_runs，失败可追溯 |

这八个概念里只有 assignee 是“人”的抽象，其余七个都在做同一件事：把“一个 Agent 的一次会话”拆成数据库里可调度、可回收、可审计的行。想通这一点，后面的命令都只是对这张表的增删改查。

还有一个容易忽略的边界：worker 拿到的是 14 个 `kanban_*` 工具，比如 `kanban_show`、`kanban_complete`、`kanban_block`、`kanban_heartbeat`、`kanban_comment`、`kanban_create`、`kanban_link`、`kanban_request_review`，它不会 shell 出去调 `hermes kanban`——终端后端可能是 Docker、Modal 或 SSH，那里未必有可执行文件。

## 命令拓扑：从建卡到并发派发

`hermes kanban swarm` 是一等公民命令，自 v0.15.0（2026-05-28）引入，语义是“一条命令生成 Kanban Swarm v1 图”：root 黑板卡 + N 个并行 worker + 门控 verifier + 门控 synthesizer。

```bash
hermes kanban swarm "Design a multi-region failover plan" \
  --worker researcher:"Research existing failover patterns" \
  --worker architect:"Draft the target topology" \
  --worker sre:"Cost + operational risk review" \
  --verifier reviewer --synthesizer writer
```

这里有个实测出来的坑：官方文档给的示例是 `--workers researcher,architect,sre`，但 argparse 里根本没有这个参数，照抄会直接报 unrecognized arguments。真实签名是可重复的 `--worker PROFILE:TITLE[:SKILL,SKILL]`，加上必填的 `--verifier` 与 `--synthesizer`。

不想用一条命令，手工串图也一样直接（`init` 是每个 board 的一次性动作，之后所有卡都活在同一张 kanban.db 里）：

```bash
hermes kanban init
hermes kanban create "Research: swarm internals" --assignee researcher
# 记下打印出来的卡 id，例如 t_4de006aa
hermes kanban create "Draft the deep dive" --assignee writer --parent t_4de006aa
hermes kanban list --status ready
hermes kanban show t_4de006aa
hermes kanban tail t_4de006aa
```

swarm 建出来的图有个容易被忽略的细节：root 卡先以阻塞态建卡，随即被标记完成，好让 N 个 worker 立刻起跑，而它自己继续当共享黑板与审计锚点。整张图是原子提交的，dispatcher 和看板要么看不到新 swarm，要么看到完整拓扑，不会出现半连接的图；带 `--idempotency-key` 重复调用时，命令会从黑板里复原已有节点，而不是重复建图。

黑板本身也不是新造的东西：它是挂在 root 卡上的一条结构化 JSON 注释，前缀为 `[swarm:blackboard] `，各节点的 id 与父子边全部落在既有的 task_comments 与 task_links 行里。没有第二个调度器，所以运维时你面对的仍然是同一套 `list`、`show`、`tail`。

## 依赖推进：父卡 done，子卡自动 ready

建卡时就把边串好。`create_task` 会按父卡状态决定子卡的初始态：父卡全部 done，子卡直接建为 ready；父卡还没完成，子卡落在 todo，等最后一个父卡完成的瞬间由 `recompute_ready` 提升。

为什么重要：这意味着你不需要写 cron 去轮询，也不需要人工点“下一阶段”。dispatcher 每 60 秒一跳，认领 ready 卡、派生 worker、等它 complete，再顺手解锁下游。verifier 就是这条链上的一道门：它的卡被强制加载 `requesting-code-review` skill，只有以 `metadata={"gate": "pass"}` 完成，synthesizer 才会从 todo 变 ready。

卡住时还有一层保护。worker 主动 block 时要选 kind：`dependency` 表示在等上游，卡回 todo 自动恢复，不需要人；`needs_input`、`capability`、`transient` 落在 blocked，浮到人面前。同一个原因反复 block 又 unblock 累计到 2 次，会发 `block_loop_detected` 并把卡送进 triage——这是 DB 层的确定性守卫，不是模型判断。run 记录同样可追溯：一次尝试一行 task_runs，被尝试 3 次就有 3 行；从未被认领就直接完成的卡会合成一条零时长 run，避免交接链路断开。

## 工程要点：粒度、交接、落盘、重试

最容易被低估的一条是：summary 和 metadata 才是跨卡上下文，文件只是接力棒。父卡完成时，它的 summary 与 metadata 会原样进入子卡上下文的 `## Parent task results` 区块。所以 brief 要写清验收标准，产物要落盘，交接只传“做完了什么、证据在哪”。

| workspace 类型 | 完成时的行为 |
|---|---|
| scratch（默认） | 工作目录被删除；`kanban_complete(artifacts=[...])` 声明的文件先复制进附件存储 |
| dir:<绝对路径> | 保留。共享目录用它；相对路径会在派发时被拒绝 |
| worktree | 保留，git worktree 加每卡独立分支 |

交接还有一条硬约束：completed 事件的 summary 首行上限 400 字符，完整内容留在 run 行上，所以摘要必须写最要紧的那句。粒度也要克制：一张卡最好对应一次能在一轮会话里做完、且有明确产物的事；切得太碎，交接摘要占比过高，切得太粗，又退回单 Agent 长任务。

三个值得记住的数字：`failure_limit` 默认 2，连续派生失败就自动 block；`dispatch_stale_timeout_seconds` 默认 14400，即 4 小时，运行超过 4 小时且最近 1 小时没有 heartbeat 才回收，而且不计失败；附件上限 25 MB。成本控制上，`max_in_progress` 未设置时按内存推导，约为 MemTotal 除以 512 MiB，夹在 2 到 8 之间，避免一次派生十几个 worker 把机器打爆。

## 真实案例：五角色博客流水线

本文的生产过程本身就是一条 Kanban Swarm 流水线，五个角色各自交棒。

| 角色 | 做什么 | 交接什么 |
|---|---|---|
| orchestrator | 读选题，写出验收标准 | brief 路径加验收条款 |
| researcher | 核实命令、配置默认值、版本时间线 | 调研文档加未确认清单 |
| writer | 按 brief 写初稿 | 初稿加字数与分部大纲 |
| reviewer | 只查基础语法与事实一致性 | 终稿加校对清单 |
| publisher | 发 PR 到博客仓库 | PR 链接 |

每张卡在独立 workspace 里跑，产物落盘给下一张卡当接力棒，父子边就是交接通道。reviewer 这一环只做错别字、标点、术语大小写与事实一致性检查；一旦发现结构崩坏或数字与调研冲突，它用 `kanban_request_changes` 退回 writer，而不是自己动手重写。整条链跑完只向用户发一次通知：PR 建好之后。

真实卡号是 t_8b186c6f → t_4de006aa → t_9c958132 → t_0e3b92cf，任何一环挂掉，`hermes kanban runs <id>` 都能看到它是第几次尝试、怎么死的。

## 踩坑、排查与方案取舍

三个最常踩的坑。第一，brief 里不写验收标准，下游只能猜，最终产物一定跑偏。第二，把文件路径当上下文传，父卡不留 summary，子卡打开目录什么都看不懂。第三，以为卡住是模型不行——其实典型签名只有三种：`spawn_failed`，profile 名不存在或 workspace 挂不上，累计到 `failure_limit` 后自动 block；`stale`，4 小时没有 heartbeat，被重置回 ready 且不计失败；`respawn_guarded`，上一轮是限流错误或刚成功过，dispatcher 本轮故意不派生。排查入口是 `hermes kanban runs <id>`、`hermes kanban log <id>`、`hermes kanban context <id>`，最后一条会打印 worker 实际看到的完整上下文，专治“我以为它看到了”。

什么时候不该用它？如果任务是一次性的、几轮就完，直接 `delegate_task` 更省事——subagent 默认并发上限 10，开销比建卡小得多，代价是审计轨迹会随上下文压缩一起消失。如果需要多人实时讨论、反复改口径，群聊式协作更自然。Kanban Swarm 的甜蜜点是长任务、多角色、需要留痕：你会反复回头看“这张卡当初是怎么变成现在这样的”。
