---
title: "Hermes Agent v0.21 Kanban Swarm 深度解析"
date: 2026-09-15
tags: [hermes, kanban, swarm, multi-agent, ai-agent]
author: k870611
---

## 一个 agent 不够用的时候

你让一个 agent 写博客：它调研完素材，接着动笔，写完自己审一遍，再发布到 GitHub。全程一条线、一个会话。听起来很顺，但真跑起来你会发现几个绕不过去的问题。

它把调研的几千行资料塞进了同一个上下文，真正动笔写文章时，早把前面查到的关键事实稀释掉了；它审稿用的是"同一颗脑子"，写稿时留下的盲区在自查时依然看不见；更致命的是生命周期——跑到发布那一步，进程被回收、会话被压缩，前面三小时沉淀下来的中间产物没有可靠的落盘点，崩了就全丢。

根因可以概括成三句话：一个 agent 的**上下文窗口**装不下多阶段协作的全部信息；**单一模型视角**很难复核自己的产出；**进程级的生命周期**撑不起"跑几天、随时能介入"的产线。v0.21 的 Kanban Swarm 就是针对这三点做的。

## Kanban 的核心概念：把协作落到一张持久队列上

Hermes 的 Kanban 是一张 SQLite 后端的持久任务板，设计上有三条主干。

第一，**任务即状态机**。每张卡片在 9 个状态之间流转：`triage | todo | scheduled | ready | running | blocked | review | done | archived`（官方文档只列了 8 态，漏了 `scheduled`——它是"等时间而非等人"的停放态，dispatcher 不会派发它）。dispatcher 只认"处于 `ready` 且父卡片已 `done`"的任务：它原子地抢占（claim）、拉起对应 profile 的一个**独立 Hermes 进程**来执行。父卡一旦 `done`，`recompute_ready` 会把它的子卡重算提升为 `ready`——这就是"扇出 / 扇入"（fan-out / fan-in）的底层机制，整张依赖图不需要额外运行时，只是被拓扑串起来的普通卡片。

第二，**board 是硬边界，tenant 是软命名空间**。一个 board 拥有独立的 DB、workspaces 目录和 dispatcher 循环，任务绝不会跨板碰撞；tenant 在同一板内部再隔离工作区路径与记忆键。博客一条产线、另一个代码库另一条产线，物理上就分开了。

第三，**worker 拿到的是收敛过的工具集**。dispatcher 拉起 worker 时注入一整套 `HERMES_KANBAN_*` 环境变量（`TASK` / `DB` / `BOARD` / `WORKSPACES_ROOT` / `WORKSPACE` / `RUN_ID` / `CLAIM_LOCK` 等），外加 `HERMES_PROFILE`、`HERMES_TENANT`、`TERMINAL_CWD`；worker 只看到 `kanban_show / complete / block / heartbeat / comment / create / link` 这一组被 gate 住的工具，普通会话则完全看不到 `kanban_*` schema 的脚印——默认零污染。

为什么重要：把"多 agent 协作"从内存里的 RPC 调用，变成了磁盘上的持久队列。任何一步崩了都能从 `SQLite` 里恢复重跑，这是它和 `delegate_task` 最本质的分界线。

## Swarm 拓扑与 CLI：一行命令造出一张协作图

`hermes kanban swarm` 把"并行 worker → verifier → synthesizer"这张经典拓扑，一次性原子建好。在 v0.21.1 上实测，CLI 签名是这样的：

```bash
hermes kanban swarm "把博客主题整理成可发布的深度文" \
  --worker researcher:"调研 v0.21 新特性与源码" \
  --worker writer:"写 2500-3500 字中文深度文，frontmatter 含 author:k870611" \
  --verifier reviewer \
  --synthesizer publisher \
  --tenant blog \
  --created-by swarm-orchestrator \
  --idempotency-key "hermes-swarm-blog-2026-09"
```

真正值得读的是它幕后建的图（来自 `hermes_cli/kanban_swarm.py` 的 `_create_swarm_uncommitted`）：

- 先建一张 **root / blackboard 卡片**，初始状态是 `blocked`。建图完成后它被**内联激活完成**，`metadata` 打 `kind: "kanban_swarm_v1"`，之后一直作为"共享黑板 + 审计锚点"留在板上；整张图的拓扑被写进这块黑板的 `topology` 键。
- 每个 `--worker` 建一张卡片，`parents=[root]`。所以 root 完成的那一刻，所有 worker 一起被提升为 `ready`，**并行**开跑。
- 建一张 **verifier**，`parents=worker_ids`（等所有 worker 全 done 才 ready），并自动挂上 `requesting-code-review` skill。它的正文写得很硬：*"complete only with metadata `{"gate": "pass"}` when evidence is sufficient; otherwise block with exact missing work"*——校验官不是橡皮图章，证据不足就 block 并把"还缺什么"讲清楚。
- 建一张 **synthesizer**，`parents=[verifier]`，自动挂 `humanizer` skill，正文要求 *"Do not start until the verifier has passed the gate"*。

整个过程包在**一个写事务**里：要么全建成功、然后 `recompute_ready`，要么全回滚。`--idempotency-key` 命中已存在的 root 时，会直接从黑板 `topology` 恢复整张图，而不是再建一遍。

为什么重要：你不用手动 `create` 六张卡再手动 `link` 一长串边。一条命令把扇出、扇入、校验门禁、黑板锚点全部建好，且原子、幂等——不会出现"worker 建好了 verifier 却没了"的半吊子状态。

## worker 生命周期与 lane 契约

dispatcher 真正拉起一个 worker 时，构造的是一条这样的命令（见 `kanban_db_dispatch.py` 的 `_worker_argv`）：

```bash
hermes -p <profile> --cli --accept-hooks \
  [--skills X] [-m model] [--provider p] [--reasoning effort] \
  chat -q "work kanban task <task_id>"
```

两个细节决定成败：`--cli` 是**强制**的——worker 绝不能进交互式 TUI，否则它会因为没有 TTY 而静默 `exit 0`，dispatcher 会把这种"空跑"判定为"协议违规"；`chat -q "work kanban task <id>"` 是 worker 的唯一入口，它从这张卡开始工作、也靠卡片 id 定位自己的上下文。

worker 的三种合法终止方式，就是它的 lane 契约：

- `kanban_complete`：做完，把产物与 `summary / metadata` 交给下游，子卡被自动提升；
- `kanban_request_review`：做完但要先过人工或 reviewer 一关，进 `review` 列（注意它**不是** block，不会计入 unblock-loop 检测）；
- `kanban_block`：缺外部输入或遇到真正的外部障碍，挂起；用 `dependency` 类型还能"等某张父卡完成后自动回 `ready`"，不占人工。

跑长任务要周期性打 `kanban_heartbeat`。心跳超过 `dispatch_stale_timeout_seconds`（默认 4 小时）没到，dispatcher 会回收这条 claim 把任务重新置为 `ready`——重新入队不记失败，但你会丢掉当前这次 run 的进度。

## v0.21 新特性：从"能协作"到"可验收、可回收"

相比早先版本，v0.21 加了"验收与资源回收"这两块硬骨头。

**1. PR completion contract（`--completion-contract`）。** 建卡时可选声明三种之一：`local-only`（默认，纯本地完成即可）、`OWNER/REPO`、或一个精确到 head 的 GitHub PR URL。任务只有当它声明的那个 PR 的 CI 全绿、且 head 没有漂移时，dispatcher 才允许 `complete`。`kanban_pr_acceptance.py` 会用 `gh api` 走 GitHub GraphQL 拿 `headRefOid` 和分支保护里的 `requiredStatusChecks`，把"验收凭据"逐条写进任务事件日志；证据不够就退回 block，并留下 recovery 指引。为什么重要：它把"我声称做完了"升级成"CI 亲自签了字"，直接堵死"自报 success 其实没合入 / 没跑绿"的经典骗局。

**2. 迭代/预算检查点（`agent.budget_warning_ratio`）。** 在 `config.yaml` 的 `agent` 段里，把 `budget_warning_ratio` 设成一个 0~1 之间的比例（kanban worker 未显式设置时默认 0.9）：一旦本次 run 消耗达到该阈值，就给模型一个**一次性**的可见 SYSTEM NOTICE，让它知道预算快见底、该收尾或交棒，而不是被硬截断；对 kanban 任务还附带一句"diff/commit 不算完成证据"。它和 `run_budget_seconds`（墙钟预算）配合，专门治"agent 在一个任务里空转烧完预算"。为什么重要：给模型一个"软刹车"，把"预算打满才断"变成"临到边界先收口"。

**3. boards 多项目隔离。** `hermes kanban boards` 让每个项目 / 工作流各有一块独立 board（独立 DB、workspace、dispatcher 循环）：`switch/use` 切当前板，`export/import` 迁移，`set-default-workdir` 定默认工作目录。为什么重要：多项目共用一个 Hermes 实例时，任务和记忆彻底互不串线，而不再是"一块大板子里靠 tenant 硬挤"。

## 与 delegate_task 到底差在哪

最容易混的就是 `delegate_task` 和 Kanban Swarm——它们都能"拉子 agent 干活"，但定位完全不同：

| 维度 | `delegate_task` | Kanban Swarm |
|---|---|---|
| 隔离 | 独立上下文，仍共享父进程 | 完全独立的 Hermes 进程 |
| 生命周期 | 分钟级，随父进程消亡 | 小时 / 天级，持久在 SQLite |
| 状态 | 内存里的句柄，结果回填父会话 | 磁盘上的卡片状态机 + 事件日志 |
| 恢复 | 父进程崩了子任务全丢 | 崩了从 DB 重算 `ready` 重新拉起 |
| 验收 | 子 agent 自报结果 | PR contract + verifier 门禁复核 |
| 适用 | 会话内、短平快的并行子任务 | 长链路、需审计、要人工把关的产线 |

`delegate_task` 是"我顺手把几个子活儿分出去，几分钟收回来"；Kanban Swarm 是"我搭一条能跑几天、随时能介入、每步都留痕、还能让 CI 签字验收的产线"。选哪个，取决于你的活儿要不要**活得比当前会话久**。

## 实战：把这条博客产线真正跑起来

本机就有一条真实在跑的 blog board，它的产线是**串行链**（上游 handoff 喂下游），而不是一整张 swarm 并行图：

```bash
# 逐张建卡 + 父卡链接，串行链：researcher → writer → reviewer → publisher
hermes kanban create "调研 Hermes Kanban Swarm v0.21" \
  --assignee researcher --board blog
# 记下返回的卡片 id（--json 可机读），逐张挂 --parent 串起来：
hermes kanban create "撰写 2500-3500 字深度文（frontmatter author:k870611）" \
  --assignee writer --board blog --parent <researcher 卡 id>
# ...依次链 reviewer、publisher；researcher 卡 done 后，writer 自动进 ready
```

而 swarm 命令适合**并行扇出**的场景（比如"同时调研 N 个来源"）：

```bash
hermes kanban swarm "产出并发布 Hermes Kanban Swarm 深度文" \
  --worker researcher:"调研 v0.21 新特性与源码" \
  --worker writer:"写 2500-3500 字中文深度文，frontmatter 含 author:k870611" \
  --verifier reviewer \
  --synthesizer publisher \
  --board blog
```

这条链路的每一环都有明确的契约：`researcher` 把发现写进父卡 handoff（`summary / metadata`），`writer` 读的是**落盘在父卡 handoff 里**的调研结论，而不是指望它"还记得"；`reviewer` 是独立的一双眼睛，只做基础语法 / Markdown / frontmatter 体检，通过就 `complete`、否则 `request_changes` 指明位置；`publisher` 把 `content/posts/*.md` 推到 `k870611/boke`（Hugo + Pages），走 PR 合入 `main` 触发自动部署——若这张卡声明了 PR contract，则只有 PR CI 全绿才算真正发布成功。

整条链路的价值在于"可恢复 + 可验收"：任何一环崩掉，dispatcher 都会把它重新置为 `ready` 再拉起，前几环的产物都躺在 SQLite 里没丢；而门禁和 contract 保证"声称完成"和"确实完成"是同一件事。

## 常见坑与对策

把这张图真正跑顺，有几个反复踩到的坑，值得记下来。

**worker 静默空跑。** 最常见的"协议违规"是 worker 在交互式界面下没有 TTY，静默 `exit 0` 却没做任何事。对策就是上面强调的：dispatcher 拉起的一律带 `--cli`，worker 永远以 `chat -q "work kanban task <id>"` 开场，从卡片 id 定位自己的上下文，而不是裸开一个交互式会话。

**长任务被回收，进度丢失。** 心跳不是装饰。跑超过默认的 stale 超时（约 4 小时）而没打 `kanban_heartbeat`，dispatcher 会回收 claim、把任务重新置为 `ready` 再拉起——不记失败，但当前 run 的中间状态没了。对策：长抓取、长构建每几分钟打一次心跳，把关键中间结果落盘到 workspace，让重跑能从断点续。

**自报完成不等于真的完成。** 子 agent 说"上传成功 / 文件已写"只是自报。对任何有外部副作用的动作（发布、上传、开 PR），要求一个可核验的凭据（URL、PR 号、绝对路径）并自己复核；声明了 PR completion contract 的任务，则由 CI 亲自签字，自报的 success 不算数。

**半吊子图。** 手动 `create` 卡片再手动 `link`，容易建出"worker 有了、verifier 漏了"的残缺拓扑。用 `swarm` 一条命令原子建图，配 `--idempotency-key` 保证重跑幂等，就不会有这种中间态。

**block 用错类型。** 等别的任务完成、自己可以自动续跑——用 `dependency` 型 block，父卡 `done` 时自动回 `ready`，不占人工；需要人来拍板、给凭据、回答问题的才用 `needs_input`。把"等依赖"写成"等人"，会白白把任务留在人面前堵着。

**静默崩溃与协议违规分不清。** 进程直接退出且**一次终止调用都没有**，判为 crashed；正常退出却忘了调 `kanban_complete / request_review / block` 里任何一个，判为 `protocol_violation`——系统最多给 2 次合成 nudge 补提醒，连续 3 次违规直接自动 block。长任务要周期性打 `kanban_heartbeat`，否则超过 4 小时 stale 被回收时重跑丢进度。

## 何时用、何时别用

**用 Swarm / Kanban**：任务是**多阶段、需要不同视角、需要留痕与人工把关、且可能跨越较长时间**的——本例的博客产线、CI 门禁式发布、跨库协作，都是典型场景。

**别用 Kanban 去替代 `delegate_task`**：`delegate_task` 适合**一次会话内、分钟级、不需要持久状态与审计**的快速并行子任务（比如同时读三个文件、并发跑两路搜索）。反过来也别为了"显得高级"，给一个三行的小活儿套一整张 Swarm 图——建图、门禁、黑板锚点都有开销，得不偿失。

一句话记住：**内存里的并行用 `delegate_task`，磁盘上的产线用 Kanban Swarm。** 判断标准始终只有一个——你的活儿，需不需要活得比当前会话更久。
