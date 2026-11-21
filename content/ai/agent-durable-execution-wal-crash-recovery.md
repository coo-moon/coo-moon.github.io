---
title: "Agent 一断电就白干：WAL、事件溯源与「可恢复执行」的内核"
date: 2026-11-21T08:00:00+08:00
draft: false
tags: ["AI Agent", "Durable Execution", "WAL", "事件溯源", "崩溃恢复"]
categories: ["AI 学习笔记"]
slug: "agent-durable-execution-wal-crash-recovery"
summary: "2026 年 11 月，Anthropic 与 DeepSeek 先后发布常驻 Agent 架构，两份文档不约而同地把状态从进程里挪了出去。这其实是数据库在 1970 年代就解决的问题：WAL、检查点、重放、幂等。本文用数据库内核的视角，拆解「可恢复执行」到底在恢复什么。"
---

# Agent 一断电就白干：WAL、事件溯源与「可恢复执行」的内核

> 2026 年 11 月，Agent 架构领域出了两份分量很重的文档。11 月 13 日，Anthropic 应用 AI 团队的 William Hu 发布《Building always-on agents》；10 月 31 日，DeepSeek Harness 团队公布 Super-agent 架构设计。
>
> 两份文档没有互相引用，却收敛到同一句话：**别把状态留在进程里**。而这正是数据库在五十年前就开始解决的问题。

## 一、先讲人话：停电后的收银台

想象一家小店，收银员记性好得惊人：谁买了什么、收了多少钱、哪笔退款批过，全记在脑子里。生意好的时候这没问题，直到某天跳闸。

灯再亮时，收银员一脸茫然。更糟的是，有些事**已经发生了**——货交出去了，钱收进来了——但"记录"没了。

老会计不会这么干。他的桌上永远摊着一本流水账：每笔交易先按时间顺序写一行，写完才去动总账。跳闸了？翻开流水账，从最后一条已确认的记录往后重放，账就回来了。

数据库把这两样东西叫 **WAL（Write-Ahead Log，预写日志）** 和 **恢复（recovery）**。而今天大多数 Agent，就是那个靠脑子记账的收银员。

## 二、现场：两份架构文档，同一句潜台词

Anthropic 那篇的判断很反直觉：**"always-on 是一种用户体验，不是让进程一直跑。"** Agent 绝大部分时间在睡觉，不烧 token；事件来了才醒，跑一个 run，把学到的东西写回去，继续睡。

它给出五个部件：inbox（收件箱）、timer（定时器）、memory（记忆）、permissions（权限）、outbox（发件箱）。其中几条纪律读起来完全是数据库运维手册：

- **inbox 是事件的唯一可信来源**：webhook、用户消息、定时器一律先落库再唤醒 worker；worker 醒来第一件事是从 inbox 拉最新事件，这样它半路死掉，新 worker 能接着处理同一条。
- **一个 inbox 同一时间只允许一个活跃 worker**，否则两个 run 会互相覆盖。
- **timer 不自己执行**：到期只是往 inbox 写一个事件，走同一条执行链。如果 timer 表和 inbox 表在同一个数据库，可以用一个事务保证"标记已触发"和"写入事件"同时生效——这就是本地事务的原子性。

DeepSeek 的 Super-agent 走得更远：Router/Worker 双层、事件驱动、计算/状态/执行环境三者分离、外挂 Watchdog。文章自己承认，这是把操作系统里的调度、缓存、存算分离、看门狗搬进 Agent。四项特征里有一项叫"持久"（Persistent）：始终在线，异常后能自己恢复。

把两边的词汇翻译成数据库术语，对应关系整齐得有点好笑：

| Agent 概念 | 数据库概念 |
|---|---|
| 一次 run | 一个事务 |
| session | 表 / 聚合根 |
| inbox | WAL / commit log |
| memory 文件 | 物化视图 / 快照 |
| timer 表 | 延迟任务表 |
| worker 租约 | 行锁 / lease |
| tool call | 外部事务（不可回滚） |
| compaction | vacuum / merge |

## 三、WAL：为什么"先写日志"能救命

数据库写数据页是**随机写**，写日志是**顺序写**，后者快一个数量级。更关键的是，崩溃时你只需要相信一件事：日志尾部之前的记录一定在磁盘上。

ARIES（Mohan 等人，ACM TODS 1992）把规则总结成三条，至今仍是 DB2、SQL Server、PostgreSQL 这一系的骨架：

1. **Write-Ahead**：任何修改先写日志；日志 fsync 落盘之后，数据页才允许落盘。
2. **Repeating history**：重启后先"重演历史"，把系统恢复到崩溃前那一刻，包括那些后来要回滚的事务。
3. **Undo**：再回滚崩溃时仍未提交的事务。

有意思的是它纵容了两种"坏习惯"：steal（未提交事务的脏页可以提前写盘）和 no-force（提交时不必把所有脏页刷盘）。这两个放纵换来了高吞吐，代价是恢复时必须能 redo 又能 undo。**性能与恢复复杂度，永远是同一枚硬币的两面。**

Agent 的 run 完全可以照抄：

```
seq  event                          磁盘上的状态
001  run.start(task=invoice#42)     running
002  tool.request(send_email,...)   pending     ← 意图先落盘
003  tool.result(ok, msg_id=...)    pending
004  run.checkpoint(progress)       running
005  run.finish                     done
```

真正的难点在 002 和 003 之间：日志说"发邮件的请求已经发出"，但没记录结果。这就是**不确定窗口**——外部世界可能已经执行了。数据库靠 redo/undo 的幂等性抹平它；Agent 没有 undo，只能"先查再补"。

## 四、三种恢复流派，三种账单

### 流派 A：快照式（checkpoint the state）

LangGraph 的 checkpointer 是典型：每个图节点执行完，把整个 state 序列化写进数据库。它提供三档 durability——`sync`（进下一节点前写）、`async`（边跑边写）、`exit`（只在结束时写），本质是在"安全"和"延迟"之间选一个点。

- 优点：恢复就是读最后一张快照，不需要重放逻辑，也不要求代码确定性。
- 代价：状态越大写得越贵；同一个 session 的记录被反复覆盖，写放大明显。上下文动辄几十万 token 的 Agent，这一条很致命。

### 流派 B：日志重放式（event-sourced replay）

Temporal（从 Uber Cadence 分叉而来）是代表。它不存状态快照，只存**事件历史**：workflow 收到的每个信号、activity 的每次返回，都按顺序记下来。worker 崩了，换一台机器从头把 workflow 代码再跑一遍，跑到需要外部结果的地方，直接从 history 里取上次的返回值——代码于是"自动走到"崩溃前的位置。

这套玩法有一条铁律：**workflow 代码必须确定性**。不能直接调 `Date.now()`、`random()`、`uuid()`，因为重放时它们会给出不同的值，代码就拐进另一条分支。Temporal 的解法是把不确定性包成 API（`Workflow.now()`、side effect），把真正有副作用、会失败的动作全部赶进 activity——activity 可以重试，结果进 history。

- 优点：日志只记结果不记状态，体积小；恢复精度到最后一个 activity。
- 代价：重放要吃 CPU；确定性约束会污染业务代码，团队必须理解"什么能放进 workflow"。

### 流派 C：把日志塞进你自己的数据库（DBOS）

DBOS 干脆不要编排服务：它是一个库，直接嵌在应用进程里，把 workflow 的 journal 写进你已经在跑的 Postgres。写入模式非常克制——workflow 开始时写一次输入，每个 step 完成后写一次返回值，结束时写一次终态。恢复时扫出未完成的 workflow，用原始输入重新调用，已完成的 step 直接跳过。

- 优点：零新增基础设施；日志和业务数据同库，可以同事务提交。
- 代价：写放大固定（step 越多写越多）；数据库成了单点，库挂了全挂。

除此之外还有 Inngest（重放 step 结果）、Restate（journal）、AWS Lambda 的 durable execution（执行窗口可长达一年）。名字不同，骨架一致：**append-only 记录 + 确定性重放 + 幂等副作用**。

选型直觉大概是这样：

| 你的处境 | 更合适的方案 |
|---|---|
| 已在 Postgres 上、团队小 | DBOS / LangGraph checkpointer |
| 长流程、跨天、要可视化 | Temporal |
| 事件驱动、已有队列 | Inngest |
| 单机小脚本 | 一张 inbox 表 + cron 足够 |

## 五、exactly-once 是谎言，幂等才是答案

分布式系统里有一条冷冰冰的结论：在不可靠网络下，你只能在 at-most-once（可能丢）和 at-least-once（可能重）之间选，**不存在 exactly-once 的消息投递**。所谓"exactly-once 语义"，全是"至少一次投递 + 接收端幂等"拼出来的幻觉。

Agent 比普通消息系统更难，因为它的副作用发生在外部世界：发了一封邮件、提了一个 PR、转了一笔钱。这些动作没有 rollback。

Anthropic 那篇专门点了一句：**事件去重只能保证本地事件不被重复处理，不能保证外部操作只发生一次。** 真正的可靠性需要三样东西一起工作：

1. **幂等键**：由 `run_id + step_id` 派生，随请求传给外部 API。Stripe 的 `Idempotency-Key` 是这个模式最著名的实现。
2. **回执查询**：动手前先问外部系统"这件事你做过吗"。
3. **可补偿动作**：真做重了，用 Saga 的反向操作抵消（退款、关 PR、发更正）。

还有一条更隐蔽的坑：**僵尸 worker**。worker 因为 GC 停顿或网络分区被误判死亡，租约被别人接管；等它"复活"，还在按旧认知写状态。数据库的解法是 fencing token——每次授予租约带一个单调递增的 epoch，所有写入都带上 epoch，存储端拒绝比当前 epoch 小的写。Agent 的 lease 也该这样：

```sql
CREATE TABLE inbox (
  id           TEXT PRIMARY KEY,          -- 上游事件 ID，天然幂等键
  session_id   TEXT NOT NULL,
  payload      JSONB NOT NULL,
  status       TEXT NOT NULL DEFAULT 'pending', -- pending|running|done|failed
  lease_epoch  BIGINT NOT NULL DEFAULT 0, -- 每次认领 +1，用于 fencing
  lease_until  TIMESTAMPTZ,
  available_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

恢复循环只有三步——认领、执行、确认：

```python
while True:
    ev = claim_next_event(worker="w1", lease=60)   # UPDATE ... WHERE status='pending' RETURNING *
    if ev is None:
        sleep(1); continue
    try:
        run_agent(ev, idem_key=f"{ev.id}:{ev.lease_epoch}")
        mark_done(ev.id, epoch=ev.lease_epoch)     # 带 epoch 校验的写
    except Crash:
        pass          # 不手动释放，等租约自然过期，让新 worker 接手
```

这段代码没有任何"智能"，但它是所有花哨 Agent 框架真正的地基。

## 六、别忽略：上下文也是状态

前面讲的是任务状态，Agent 还有另一层状态：**它当时知道什么**。

session 是表，对话历史是行，token 预算是页缓存，compaction 是 vacuum。恢复时你不仅要恢复"做到哪一步"，还要恢复"依据什么做的决定"。

Anthropic 给了一条很实用的记忆纪律：**只存以后查不回来的东西**。issue tracker 里有的，存链接就行，别拷贝。按用途分文件：行为指令、用户画像、已告知用户的内容、任务进度。

还有一个物理约束：KV Cache 在 GPU 显存里，进程一死就没了。恢复后第一件事是把系统提示、工具定义、记忆文件重新 prefill 一遍——这笔钱躲不掉。所以"状态外置"省的是正确性，不是成本；成本要靠前缀缓存和上下文精简去省。

## 七、一张工程清单

1. 先有 inbox，再有 Agent。
2. 每个 tool call 带幂等键，关键动作必须能回执查询。
3. 一个 inbox 一个活跃 worker，租约带 epoch。
4. checkpoint 只存查不回来的东西。
5. 划清确定性边界：可重放的逻辑与有副作用的动作分开。
6. 监控"有没有进展"，而不只是"进程活没活"——DeepSeek 的 Watchdog 盯的就是模型在原地打转，因为很多故障不是崩溃，是"活着但没用"。

## 八、写在最后：五十年一轮回

1970 年代，System R 用 shadow paging 保证原子性，很快发现代价太高；1980 年代 WAL 成为主流；1992 年 ARIES 把 steal/no-force + repeating history 写成教科书；2000 年代事件溯源从 DDD 社区长出来；2010 年代 Kafka 把"日志即真相"推到架构中心；2020 年代 durable execution 把它产品化；2026 年，Agent 撞上了同一堵墙。

这不是巧合。崩溃、并发、副作用不可逆，是任何"要对外部世界产生持久影响"的系统都逃不掉的三个约束。约束相同，解空间就相同。

所以那句话可以改一改：模型是 CPU，Harness 是操作系统，那么 inbox、journal、checkpoint、lease 这一层，就是 Agent 的文件系统与事务管理器。写 Agent 的人，迟早要补上数据库这门课。

留一个问题给自己：不要问"我的 Agent 会不会崩"——它一定会。要问的是：**崩了之后，它知不知道自己在哪，以及下一步该做什么。**
