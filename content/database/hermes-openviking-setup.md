---
title: "Hermes Agent 接入 OpenViking：给 AI 助手装上一个「上下文数据库」"
date: 2026-08-25
draft: false
tags: ["AI Agent", "OpenViking", "Hermes", "记忆系统", "火山引擎"]
categories: ["数据库"]
slug: "hermes-openviking-setup"
summary: "手把手把开源的 OpenViking 上下文数据库接入 Hermes Agent：云服务开通、API 配置、资源挂载、语义检索实测，以及踩过的坑。"
---

## 为什么给 Agent 换记忆？

我一直在用 [Hermes Agent](https://github.com/NousResearch/hermes-agent) 作为日常助手（接飞书、跑 cron 定时任务、盯 A 股）。它自带一套基于 SQLite + Markdown 的内置记忆，够用但有两个瓶颈：

1. **检索是关键词匹配**——搜"之前配邮件"命中不了"Himalaya QQ邮箱"这种语义相关的内容
2. **知识散落在文件里**——持仓数据、学习笔记、项目仓库各自为政，Agent 没法"翻阅"

[OpenViking](https://github.com/volcengine/OpenViking)（火山引擎 2026 年 1 月开源，33k+ stars）解决的就是这个问题。它是一个 **Context Database**：把记忆、资源、技能统一挂成 `viking://` 虚拟文件系统，内容自动处理成 L0 摘要 / L1 概览 / L2 全文三层，Agent 用 `ls`/`find` 确定性地浏览自己的上下文。

官方 benchmark 里有个数字很说明问题：LoCoMo 长对话记忆测试，Hermes 原生记忆 33.4%，接入 OpenViking 后 **82.9%**，同时输入 token 还降了一半以上。

而且 Hermes 对 OpenViking 是**一等集成**——不用装插件，配个环境变量就完事。下面是完整过程。

## 架构一句话

```
Hermes (任何入口：飞书/CLI/cron)
    │  HTTP
    ▼
OpenViking 云服务 (api.vikingdb.cn-beijing.volces.com/openviking)
    │
    ├── viking://user/memories/     ← 会话自动沉淀的长期记忆
    └── viking://resources/         ← 手动挂载的知识库（文档/代码/网页）
```

关键点：**内置记忆不会被替换**。Hermes 的 MEMORY.md/USER.md 照常工作，OpenViking 是叠加层——新记忆双写，检索时两边都查。存量数据零迁移成本。

## 第一步：开通 OpenViking Service

两条路：本地 Docker 自部署，或直接用火山引擎托管的 SaaS。我不想再维护一个容器，选了云服务。

在火山引擎控制台搜 "OpenViking"，开通后拿到 API Key（格式是三段式：`xxx.xxx.xxx`）。

> ⚠️ 坑 1：控制台里有两种 key，要拿的是 **User Key**（普通数据访问），Root Key 是管理用的，配到 Hermes 里调数据接口会报权限错误。
>
> ⚠️ 坑 2：第一个 key 我复制过来一直 401，重新从控制台复制一遍就好了——飞书传输长字符串可能吞字符，或者 key 本身没生效。遇到 401 先怀疑 key 完整性。

## 第二步：配置 Hermes

编辑 profile 的 `.env`（我的路径是 `~/.hermes/profiles/assistant2/.env`）：

```bash
OPENVIKING_ENDPOINT=https://api.vikingdb.cn-beijing.volces.com/openviking
OPENVIKING_API_KEY=你的key
# 可选：多 agent 隔离用
# OPENVIKING_ACCOUNT=default
# OPENVIKING_USER=default
```

然后切换 provider 并验证：

```bash
hermes config set memory.provider openviking
hermes memory status
```

看到这个就成功了：

```
Provider:  openviking
Plugin:    installed ✓
Status:    available ✓
```

最后重启网关让所有入口（飞书对话、CLI、cron）加载新配置：

```bash
kill $(cat ~/.hermes/profiles/<profile>/gateway.pid)   # s6 会自动拉起新进程
```

## 第三步：往里面灌知识

这是最有意思的部分。OpenViking 的 REST API 很直白，两步挂载本地文件：

```python
# 1. multipart 上传文件 → 拿 temp_file_id
POST /api/v1/resources/temp_upload   (file=@自选股票列表.json)

# 2. 用 temp_file_id 挂载到目标 URI
POST /api/v1/resources
{
  "source_name": "自选股票列表.json",
  "temp_file_id": "upload_xxx.json",
  "to": "viking://resources/stock/",
  "reason": "A股持仓与观察列表"
}
```

我把三样东西挂了进去：

| URI | 内容 |
|---|---|
| `viking://resources/stock/` | 我的 A 股持仓+观察列表 JSON |
| `viking://resources/learning/python7days/` | Python 速成课 7 篇讲义 |
| `viking://resources/learning/cpp14days/` | C++ 速成课 14 篇讲义 |

远程仓库更简单，直接给 URL 让服务端自己拉：

```json
POST /api/v1/resources
{ "path": "https://github.com/用户名/仓库", "to": "viking://resources/blog/" }
```

> ⚠️ 坑 3：连续快速挂载会撞路径锁（HTTP 409 CONFLICT, `path_busy`），等几秒重试即可，响应里也标了 `"retryable": true`。
>
> ⚠️ 坑 4：搜索接口 `/api/v1/search/find` 不接受 `top_k` 参数（会报 `Extra inputs are not permitted`），只传 `query` 就行，返回按 `memories / resources / skills` 三组分类。

## 第四步：验收效果

挂载完成后等一两分钟让服务端完成语义处理（生成 L0/L1 摘要），然后测试：

```json
POST /api/v1/search/find
{ "query": "我目前持有哪些股票" }
```

返回结果让我有点惊喜：

```json
{
  "uri": "viking://resources/stock/.overview.md",
  "level": 1,
  "score": 0.47,
  "abstract": "本目录为面向个人A股投资者的证券持仓管理专属台账目录，
                核心收录2026年8月24日更新的个人交易跟踪文档..."
}
{
  "uri": "viking://resources/stock/自选股票列表.md",
  "level": 2,
  "abstract": "...当前账户总资产14.29万元，仓位36.3%..."
}
```

注意两点：

1. **中文语义检索准确**——文件名是 JSON，问句是自然语言，照样命中
2. **分层加载在工作**——L1 给 ~100 token 的目录摘要判断相关性，L2 才展开细节。这就是官方说的省 token：不相关的目录看一眼摘要就跳过

之后我在飞书里问"Python 列表推导式在哪篇讲的"，它直接定位到对应讲义；盯盘 cron 分析时也能引用最新持仓而不用每次读文件。

## 接入后的日常变化

- **会话结束自动沉淀**：每段对话结束后，Hermes 异步提取偏好和经验写入 `viking://user/memories/`，分 6 类（偏好/实体/事件/案例/模式）
- **每轮预取**：回复前后台检索相关记忆注入上下文，非阻塞
- **双写不丢**：内置 MEMORY.md 继续维护，两边互为备份

## 总结

整个过程不到一小时，其中大半时间在踩 key 的坑。核心就三步：

1. 开通云服务拿 User Key（或本地 Docker 起 `ghcr.io/volcengine/openviking`）
2. `.env` 写两个变量 + `hermes config set memory.provider openviking`
3. 把常用知识挂进 `viking://resources/`

如果你的 Agent 也面临"记不住、找不到、每次都要重新喂上下文"的问题，这套组合值得试。相比自己折腾 RAG 管道，它把分层索引、检索轨迹这些脏活都做掉了，而且 AGPLv3 完全开源无阉割。

---

*参考：[OpenViking GitHub](https://github.com/volcengine/OpenViking) · [Hermes 记忆提供方文档](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers)*
