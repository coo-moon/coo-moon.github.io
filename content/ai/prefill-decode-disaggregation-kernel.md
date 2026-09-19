---
title: "为什么要把 LLM 推理引擎劈成两半？Prefill/Decode 分离的内核原理"
date: 2026-09-19T08:00:00+08:00
draft: false
tags: ["大模型推理", "AI Infra", "分布式系统", "vLLM", "KV Cache"]
categories: ["AI 学习笔记"]
slug: "prefill-decode-disaggregation-kernel"
summary: "2026 年，推理取代训练成为 AI 竞赛的主战场，而 PD 分离（Prefill-Decode Disaggregation）正成为从 vLLM、SGLang 到 NVIDIA Dynamo 所有推理引擎的默认架构。本文从 Roofline 模型讲清'读题'和'吐字'为什么无法共处一块 GPU，并指出这场变革其实是数据库世界 OLTP/OLAP 分离旧智慧在 AI Infra 的一次完整重演。"
---

## 引子：训练是昨天的新闻，推理才是今天的战争

英伟达 GTC 2026 上，黄仁勋把一个词挂上了主题演讲：**"inference inflection point"（推理拐点）**。IEEE Spectrum 的概括更直白："训练已经是昨天的新闻，CIO 们只想聊推理。"

这条主线在 2026 年 9 月仍在加速：vLLM 官方博客刚发布《为什么你的单机 vLLM 也需要 Prefill-Decode 分离》，把 PD 分离从"数据中心级奢侈品"下放到了 8 卡单机；OpenAI、Anthropic 的 API 背后，Kimi、DeepSeek 的公开部署数据里，"prefill 集群 + decode 集群"的组合反复出现。

一个服务大模型的引擎，为什么要像离婚分财产一样，把计算"劈成两半"？这背后是一套非常"系统内核味"的原理——以及一段数据库行业讲了三十年的老故事。

## 一、先讲人话：一间办公室里住了两个不打架才怪的员工

把一次 LLM 请求想象成一位顾问在办公室里干活：

- **Prefill（读题）**：你把一份 100 页的方案递给他，他从头到尾精读一遍，做笔记（生成 KV Cache）。这一步**可以并行**，一目十行，是纯粹的重体力脑力活；
- **Decode（写结论）**：读完之后，他一个词一个词地往外写结论。每写一个字，都要把**全部笔记重新翻一遍**才能落笔。这一步**天然串行**，脑子大部分时间在等笔记。

问题来了：这家公司只有一间办公室（一块 GPU），而且是合用办公——

每当顾问 A 在"精读 100 页"（prefill 占满算力），前台顾问 B 正在"一个字一个字写报告"（decode），B 的节奏就被卡住：用户会看到聊天窗口的字**突然卡住半秒，又唰唰往外蹦**。反过来，为了不打扰 B，让 A 每次只读 5 页读二十次（这就是 chunked prefill），A 的"读完第一句话"的时间就被拖长了。

这就是推理服务的两个生死指标在互相拉扯：

| 指标 | 全称 | 用户感知 | 由谁决定 |
|---|---|---|---|
| **TTFT** | Time To First Token | 发出问题后多久"开始响应" | prefill |
| **TPOT** | Time Per Output Token | 回答过程中文字蹦出的流畅度 | decode |

**colocation（混部）的原罪：两个指标被同一块 GPU 上的同一批人质事件绑架，永远只能保一头。**

## 二、内核事实：两个阶段在物理上是两种工作

从算子层面看，两个阶段的"体质"完全不同。系统学界用 **Roofline 模型**来刻画：一块 GPU 有两个天花板——算力（FLOPs/s）和显存带宽（Byte/s），一个负载的**算术强度**（每读 1 字节做多少次运算，FLOPs/Byte）决定它撞哪个天花板。

```
算术强度 (FLOPs/Byte)
   ↑
   │        ▓▓▓ prefill：200~400 FLOPs/Byte
   │        ▓▓▓ → compute-bound（撞算力天花板）
─────────── 拐点 ──────────────────────────
   │ ░░░ decode：0.5~8 FLOPs/Byte
   │ ░░░ → memory-bound（撞带宽天花板）
   └──────────────────────────────→
```

- **prefill**：一次前向要处理整个 prompt 的所有 token，矩阵乘又大又规整，GPU 算力吃得满满的；
- **decode**：每个 step 只算 **1 个 token**，但必须把全部模型权重 + 这个请求的全部 KV Cache 从 HBM 里搬一遍。算 1 个乘法要读一堆数据，算力利用率常常只有个位数百分比——**decode 不是在计算，是在搬运**。

更狠的是硬件演化方向。微软 Splitwise 论文（ISCA 2024）量过一条数据：从 A100 到 H100，**算力涨了 3.43 倍，显存带宽只涨了 1.64 倍**。也就是说，"-balanced 的卡"每换代一次，对 prefill 更快、对 decode 却越来越喂不饱带宽——**同一块卡对两个阶段一个过剩、一个饥饿**。

既然体质不同、想要的机器也不同，硬关在一间办公室就是互相伤害：

| 维度 | Prefill | Decode |
|---|---|---|
| 瓶颈资源 | 算力（FLOPs） | 显存带宽 + 容量 |
| 并行性 | token 级并行，可拆 | 严格自回归，不可拆 |
| 理想硬件 | 最新旗舰卡（算力涨得快） | 老卡/带宽卡也够用（Splitwise 的异构池思路） |
| 批处理 | 小 batch 就打满算力 | 大 batch 摊薄权重搬运 |
| 优化目标 | TTFT | TPOT |

## 三、把手术室和门诊分开：PD 分离的账怎么算

DistServe（UCSD Hao AI Lab，OSDI 2024）给出了这个领域的"判决性实验"：把 prefill 和 decode 分到不同 GPU 池，各自选自己的并行策略和资源配比，结果是——**同样的 SLO 约束下多服务 7.4 倍请求，或者把 SLO 收紧 12.6 倍**，而此前所有混部系统（哪怕加了 chunked prefill）都做不到。

它同时提出了一个值得记住的指标观：**goodput**。吞吐量（throughput）里掺着一堆"算完了但没人要"的废 token——超时的请求对在线服务等于零。

> goodput = 在 P90 TTFT < 200ms 且 P90 TPOT < 50ms 这类约束内，能稳定服务的最大请求速率。
> **"Batching is free, interference is not."** 推理系统的目标函数从"每秒算多少 token"换成"每秒交付多少达标请求"，是整个 2024-2026 架构转向的思想起点。

但分离有一笔必须支付的账：**prefill 机器算出来的 KV Cache，要物理搬运给 decode 机器**。

来算一笔算术：以 Llama-3-70B（GQA，8 个 KV head）为例，每 token 的 KV 约 **320KB**，128K 长上下文的 KV 就是 **~40GB**——这相当于把一部 4K 电影从一台机器传给另一台，而且在线服务要求"传得比生成一个字还快"。

这笔账能算平，靠的是数据中心网络这 20 年的积累：

- **RDMA**（远程直接内存访问）：绕过双方内核与 TCP 协议栈，网卡对网卡零拷贝，单卡 400Gbps 起步，NVLink/InfiniBand 域内更高；
- **NIXL**：NVIDIA 为 Dynamo 开源的推理传输库，把 RDMA/GPUDirect 封装成 KV 搬运专用通道；
- **Mooncake Transfer Engine**：月之暗面开源的传输引擎，拓扑感知、多网卡聚合，已成为 vLLM/SGLang 社区事实上的搬运后端之一。

DistServe 的实测结论是：典型场景下 **KV 传输耗时小于一个 decode step**——搬运被流水线藏进了生成的缝隙里，用户无感。手术室的病历传到了门诊室，而病人（用户）甚至没注意到医生换了一个。

## 四、更深一层：从"分开算"到"分开存"——KV 成为一等公民

如果你读过我上一篇[《你的 Agent 每一步都在重复付钱》](/posts/kv-cache-prefix-caching-agent-kernel/)，那里讲的是**单实例内**的前缀缓存；PD 分离再往前推一步：既然 KV 已经要在机器之间流动，那它凭什么不是一种**可以独立成层的资产**？

Mooncake（Kimi 的 serving 平台，获 **FAST 2025 最佳论文**）的答案是 **KVCache-centric 架构**：把 GPU 集群里闲置的 CPU、DRAM、SSD、NIC 池化成一个分级 KV 存储池，全局共享、跨实例复用——论文标题就叫"**Trading More Storage for Less Computation**"（用更多存储换更少计算）。

它在生产里跑出的数字：

- 真实业务 trace 下，SLO 约束内的有效请求容量提升 **59%~498%**（长上下文模拟场景最高 525%）；
- 日常服务 Kimi，**每天处理超过 1000 亿 token**，数千节点规模；
- 2025 年 7 月，Kimi K2 在 128 × H200 上以 PD 分离 + 大规模专家并行部署，公开数据为 prefill **22.4 万 token/s**、decode **28.8 万 token/s**。

SGLang 团队则在 96 × H100 上复现了 DeepSeek-R1 的 PD 分离服务：3 节点（24 卡）prefill + 9 节点（72 卡）decode，单节点 5.23 万输入 TPS / 2.23 万输出 TPS——首次以开源栈逼近 DeepSeek 论文里的官方数字。注意这个 **3:9 的资源配比**：prefill 和 decode 的机器比例本身就成了一个被调度器动态调优的"自由度"，这正是 DistServe 说的"把 SLO 满足问题拆成两个独立的优化问题"。

而 NVIDIA **Dynamo**（GTC 2025 发布，开源）把这个模式做成了数据中心操作系统层：prefill/decode worker 成为一等公民，KV-aware 路由决定请求发给谁，NIXL 负责搬运，官方宣称在 GB200 NVL72 上服务 DeepSeek-R1 相比上代基线**每 GPU 吞吐最高提升 30 倍**（厂商数据，谨慎看待）。vLLM 的 PD 分离则通过 `KVConnector` 抽象落地——有趣的是它的最新动向恰恰是**反向的**：在单机 8 卡内用 PCIe 共享内存做"轻量 PD 分离"，说明这套抽象已经普惠到"离婚不必分居两地"的程度。

## 五、内核理论彩蛋：这就是数据库 30 年前干过的事

把时间轴拉长，PD 分离不是 AI 的发明，而是系统架构的一条老规律在新一轮硬件瓶颈下的**重演**：

| 数据库史（1990s~2010s） | LLM 推理（2024~2026） |
|---|---|
| OLAP 报表 vs OLTP 交易，混合负载互相干扰 → 分库/分流 | prefill vs decode 混部干扰 → PD 分离 |
| 吞吐优先 → **QoS/SLO 感知调度** | 吞吐优先 → **goodput 调度** |
| 过载时 kill 长事务 / 拒绝准入 | Mooncake 的 prediction-based early rejection |
| Shared-Nothing 分库 → 计算存储分离（Snowflake、湖仓） | 分离计算 → KV 存储池化（Mooncake Store、FlexKV） |
| 物化视图：预计算换查询延迟 | 前缀缓存：预存 KV 换 TTFT |

规律只有一句话：**每当硬件的瓶颈换位置（磁盘 IO → 内存带宽 → 网络），系统就会沿着新瓶颈被重新拆开、再重新组装。** 数据库当年因为磁盘太慢发明了一整个宇宙，推理引擎今天因为 HBM 带宽太贵正在发明它的镜像宇宙。

## 六、冷静一下：什么时候不该分离

反过来说，PD 分离不是银弹，DistServe 团队自己的复盘也承认它引入了真实的复杂度：

1. **低并发 / 个人部署**：一台 4090 上跑 Ollama 谈分离是行为艺术，混部的 KV 还在本地显存零拷贝，反而最快；
2. **输出极短的负载**（分类、rerank、embedding）：decode 只有几个 token，搬运和跨池调度的开销占比失控；
3. **网络不行**：没有 RDMA/高带宽内网，跨节点搬 KV 的账当场算不平——这正是 vLLM 走单机 PCIe 轻量分离路线的原因；
4. **工程复杂度**：两套异构资源池的调度、KV 泄漏与失败域管理、P/D 配比随流量漂移的弹性扩缩——运维面积指数级上涨。社区最新的 arXiv 工作（如 unifying prefill/decode 的弹性方案）正是在补"什么时候该聚、什么时候该散"这门课。

所以 2026 年的现实格局是：**chunked prefill + continuous batching 依然是 80% 场景的默认答案，PD 分离是高并发在线服务的入场券。**

## 七、全景速查表

| 系统 | 出处 | 关键主张 / 已公开数据 |
|---|---|---|
| **DistServe** | OSDI 2024 | 命名 PD 分离；goodput 视角；7.4x 请求 / 12.6x 收紧 SLO |
| **Splitwise** | ISCA 2024（微软） | 阶段异构：1.4x 吞吐、省 20% 成本；量化算力/带宽剪刀差 |
| **Mooncake** | FAST'25 最佳论文 | KVCache-centric；59%~498% 容量提升；日均千亿 token |
| **vLLM P/D** | 社区 | `KVConnector` 抽象 + xPyD；单机 PCIe 轻量分离 |
| **SGLang PD** | 社区 | 96×H100 服务 DeepSeek-R1，逼近官方吞吐 |
| **NVIDIA Dynamo** | GTC'25 开源 | prefill/decode worker + KV-aware 路由 + NIXL；GB200 上宣称最高 30x |

## 结语：把"贵"变成一个架构问题

2026 年 AI 竞争的焦点正在完成一次迁移：**从"训练一个更强的模型"变成"以更低的成本服务它"**。而 PD 分离这场架构运动的方法论价值，远大于它省下的 GPU 钱：

1. **先诊断物理瓶颈，再设计架构**——算力 bound 还是带宽 bound，决定了你该买什么卡、怎么摆集群；
2. **指标即架构**——当目标函数从 throughput 换成 goodput，旧架构里"不可能"的事情突然变得显然；
3. **老系统智慧永不过时**——面对任何 AI Infra 热词，先问一句"这是分布式系统的哪个老问题的新马甲？"OLTP/OLAP、存算分离、QoS 调度……这些 90 年代的思想正批量在 GPU 集群上转世。

下次当你看到 ChatGPT/Kimi 的回答丝滑地一个 token 一个 token 往外蹦时，可以会心一笑：这丝滑背后，是一间办公室被拆成了两间，中间修了一条 400Gbps 的传送带。

---

**延伸阅读**

- DistServe: Disaggregating Prefill and Decoding for Goodput-optimized LLM Serving (OSDI 2024, arXiv:2401.09670)
- Mooncake: Trading More Storage for Less Computation (FAST 2025 Best Paper, arXiv:2407.00079)
- Splitwise: Efficient Generative LLM Inference Using Phase Splitting (ISCA 2024)
- vLLM 官方文档《Disaggregated Prefilling》与博客《Why Your Single-Node vLLM Setup Needs Prefill-Decode Disaggregation》
- 本站前作：[KV Cache 命中率与前缀缓存的内核原理](/posts/kv-cache-prefix-caching-agent-kernel/)
