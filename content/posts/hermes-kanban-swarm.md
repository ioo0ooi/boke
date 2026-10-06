---
title: "Hermes Kanban Swarm：用一块 SQLite 看板组织多智能体工作流"
date: 2026-10-06T14:31:09+08:00
tags: ["hermes", "multi-agent", "kanban", "ai-agent"]
author: "mql"
---

如果你用过 Claude Code 或 Codex 这类 coding agent，应该对「一个父 agent 现场 delegate 一批子 agent」的模式不陌生：父进程起一个 `delegate_task`，子 agent 在同一个进程里跑完，把结果收回父上下文。这种「现场分身」方案胜在简单直接，但有一个致命短板——父进程一崩，整批子 agent 的工作就蒸发了；而且整个过程不可见、难交接、事后无法追踪。

Hermes 给出了另一个答案：把多 agent 协作从「一次函数调用」升级为「一队 agent 共享一张看板」。这就是 **Kanban Swarm** 要解决的问题。它不靠花哨的分布式框架，而是在一块 SQLite 里写了一张任务图，让多个具名 agent 像团队一样对着同一块板子干活。

## 看板是什么：SQLite 之上的工作队列

官方文档有一段很精到的话：*"Every task is a row in `~/.hermes/kanban.db`; every handoff is a row anyone can read and write; every worker is a full OS process with its own identity."*

翻译成人话：看板的核心是一个 **SQLite 数据库**。每个任务（卡片）是一行，每次交接是一行注释，每个 worker 是一个独立的操作系统进程——有自己的 profile、session、工具集和内存。数据库用 WAL 模式，通过 `BEGIN IMMEDIATE` 加 CAS 来串行化写入：一个 dispatcher 赢下 claim，其他竞争者看到零行，没有重试、没有分布式锁。

卡片有几个关键字段：`assignee`（由哪个 profile 来干）、`status`（`todo / scheduled / ready / running / review / blocked / done` 等 9 态）、`parents` 依赖边、`comments` 线程、以及贯穿一生的 `runs` 与 `events` 记录。其中 `task_comments` 特别值得一提——它就是 agent 之间的协议通道，swarm 的跨 worker 沟通全靠往这张表里写结构化注释。

谁来调度这些卡片？一个常驻的 **dispatcher（调度器）**，默认跑在 gateway 进程内，每 60 秒 tick 一次：回收过期 claim、回收崩溃进程、promote 就绪任务、原子 claim、spawn 对应 profile。worker 侧不 shell 出子进程，而是用一套 `kanban_show / kanban_complete / kanban_block / kanban_heartbeat` 工具直接读写同一份 DB。

调度器 spawn worker 的方式也很朴素：它拼出一条 `hermes -p <assignee> chat -q "work kanban task <id>"` 命令，把 `HERMES_KANBAN_TASK`、`HERMES_KANBAN_BOARD` 等环境变量注入子进程，worker 一启动就知道自己该干哪张卡。同时每个 claim 带一个 15 分钟的 TTL，活着的进程会自动续期，只有真死掉的才会被回收。这套机制看似简单，却构成了 Kanban 最核心的可靠性底座：状态永不依赖任何单一进程的存活。

## 依赖调度与流水线：从 fan-out 到 fan-in

看板的杀手锏是**依赖调度**。卡片之间可以声明 `parent → child` 关系，dispatcher 会等**所有** parents 都 `done` 之后，才把子卡从 `todo` 自动 promote 到 `ready`。于是 fan-out（一张卡拆出多张并行的子卡）和 fan-in（多张卡汇聚到一张门禁卡）天然成立。

以一篇博客的生产流水线为例，你可以用四条命令搭出这样一张 DAG：

```bash
hermes kanban create "调研 Hermes Kanban" --assignee researcher
# 拿到调研卡 id 后……
hermes kanban create "写文章" --assignee writer --parent <调研卡id>
hermes kanban create "审校" --assignee reviewer --parent <写作卡id>
hermes kanban create "发布" --assignee publisher --parent <审校卡id>
```

dispatcher 会依次派发：调研卡 done 之前，写作卡一直停在 `todo`，事件流里记着一条 `dependency_wait`。这才是真正的流水线——不是「父 agent 脑子里排个顺序」，而是「顺序本身落进了数据库，任何进程都能看得见、改得动」。

更高级的用法是 `hermes kanban swarm`，一条命令直接铺开 Swarm v1 拓扑：一个立刻完成的 root 卡（作为 blackboard 与审计锚点）→ N 个并行 worker 卡 → 一张门禁 verifier 卡 → 一张 synthesizer 卡。源码里有一句很提气的话：*"Deliberately no second scheduler — a small task graph written into the existing Kanban kernel."* swarm 没有另起炉灶，它只是把一张小任务图写进了已有的看板内核。

这套实现里有两个值得细品的细节，能把「依赖调度」从口号变成可落地的工程。其一，root 卡用了一个取巧：它以 `blocked` 状态创建，等 workers、verifier、synthesizer 全部建完之后，在**同一个事务内**做一次 `blocked → done` 的 CAS 翻牌，提交后再统一 promote workers。为什么要这样？因为如果先 `ready` 再立刻 `done`，会把还没建出来的子卡一起 promote 出去。这是典型的「先建图、后点火」——确保点火瞬间整张图已经完整。

其二，依赖调度有一个必须避开的反模式：当一个 worker 因为缺东西而阻塞时，**绝不能**把「补东西的支持卡」link 成被阻塞卡的 child。那样会让支持卡排在它本该解开的卡后面——支持卡在等被阻塞卡 done，被阻塞卡在等支持卡，两张卡永远不跑，形成死锁。正确做法是只在支持卡的正文里引用那张卡的 id，而不是建立依赖边。看板内核对此也有防御：它会把「ready 子卡被降级」这类操作记成 `dependency_wait` 事件，让死锁在看板上**可见**，而不是静默卡死。

## 与 delegate_task 的差异：为何「看板即 trace」

同样是「让多个 agent 协作」，`delegate_task` 和 Kanban 是两个物种。官方把它们的区别概括成一句话：*"`delegate_task` is a function call; Kanban is a work queue where every handoff is a row any profile (or human) can see and edit."*

落在机制上，差异是实质性的：

| | `delegate_task` | Kanban |
|---|---|---|
| 形态 | RPC（fork→join） | 持久消息队列 + 状态机 |
| 父进程 | 阻塞直到子返回 | create 后 fire-and-forget |
| 子身份 | 匿名 subagent | 具名 profile，带持久记忆 |
| 可恢复性 | 无，失败即失败 | block→unblock→重跑；崩溃→reclaim |
| 人在环 | 不支持 | 任意时刻 comment/unblock |
| 审计轨迹 | 上下文压缩即丢失 | SQLite 行永久保留 |

最关键的三个词是**耐久、隔离、可追溯**。耐久：状态落在磁盘，父进程崩溃、重启都不丢任务，dispatcher 会自动 reclaim；更细一步，连续失败有计数器（`consecutive_failures`），达到默认的 `failure_limit`（2 次）就自动转成 blocked，`gave_up` 成为终态，而不是无限重试。隔离：每个 worker 是独立 OS 进程，有自己的 profile 与内存，delegate 子上下文甚至连通过 CLI 读写看板都会被栅栏拦住——隔离是硬边界，不是约定俗成。可追溯：`task_events` 是追加式事件流，`task_runs` 每次尝试一行且永不消失，`hermes kanban tail <id>` 跟单卡事件、`hermes kanban runs <id>` 翻历史尝试——看板本身就是一张分布式 trace。

## v0.16 的补全：让无人值守成为可能

一个容易搞混的版本问题：Kanban 本身在 v0.13（2026-05-07「Tenacity」）正式 ship，Swarm 拓扑在 v0.15（2026-05-28「Velocity」）成形。到了 v0.16（2026-06-05「Surface」），Kanban 相关的更新其实只有四件事，但件件都指向同一个方向——**让它能长时间无人值守**：

- **`goal_mode`**：open-ended 卡片跑一个 `/goal` 循环，worker 在同一 session 里被一个 auxiliary judge 反复评估「够不够格交差」，不够就接着干，turn 预算耗尽则 block 而非静默退出。
- **任务文件附件**：PDF、图片、源文档直接挂到卡上，worker 拿到的不是让你粘进正文的路径，而是绝对路径；还能把正文引用的图片喂给 worker 的 vision。
- **`default_assignee` 兜底 + per-profile 并发上限**：LLM 选了不存在的 profile 时有个落点，同时给单个 profile 的并发数套上缰绳。
- **`POST /runs/{run_id}/terminate`**：从 dashboard 就能掐掉一次运行。

上手路径很顺：`hermes kanban init` 建库（幂等），`hermes gateway start` 起调度宿主，然后 `create` → `assign` → 等 dispatcher 自己跑。

## 局限与适合场景

别把 Kanban Swarm 神化成通用分布式编排器。它的默认威胁模型是**单机可信本地用户**：SQLite 是单进程文件，dispatcher 默认在同机的 gateway 内，跨主机要自己安排网络与 gateway。调度粒度是**卡级而非消息级**——worker 之间没有实时协商通道，跨 worker 沟通就是写注释，swarm 的「并行」只保证同时被派发，不保证消息级协同。而且「done」通常需要 review 流程把关，不是写个 `complete` 就闭眼完结；长任务还得主动 heartbeat，否则超时会被 reclaim 重跑。还有一层容易被忽略的脆弱点：worker 的身份只是 profile 名字符串，拼错了任务会永远停在 `ready`，所以 orchestrator 在派卡前必须先 `hermes profile list` 核实角色真实存在。

所以它的甜区很清晰：**内容流水线、批量任务、需要交接与审计的团队型工作**。凡是「一份工作要跨多个角色、要扛重启、可能中途要人拍板、事后要翻旧账」的场景，一块 SQLite 看板比一百个进程内 subagent 都靠谱。等你哪天父进程崩了、工作却毫发无损地在看板上等你 reschedule 时，就会懂这块板子的分量。