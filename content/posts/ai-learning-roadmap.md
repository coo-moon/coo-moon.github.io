---
title: "AI 学习路线：从零到 Agent 开发"
date: 2026-08-03
draft: false
tags: ["AI", "学习路线", "大模型", "Agent"]
categories: ["AI 学习笔记"]
summary: "一份面向工程师的 AI 系统学习路线图，从基础概念到 Agent 开发实战，60 天可执行计划。"
weight: 1
---

## 为什么要系统学习 AI？

大模型时代，AI 正在重塑软件开发的方式。作为工程师/技术爱好者，理解 AI 的工作原理、学会使用大模型、甚至开发 AI Agent，已经不再是「可选项」。

这篇文章是我为自己规划的 **60 天 AI 系统学习路线**，也分享给同样想入门的朋友。

## 路线总览

```
基础概念 → Prompt 工程 → API 开发 → RAG 应用 → Agent 开发
  (Day 1-10)   (Day 11-20)  (Day 21-35) (Day 36-50) (Day 51-60)
```

## 第一阶段：基础概念（Day 1-10）

### 目标
理解大语言模型（LLM）的基本原理和工作方式。

### 核心内容

| 主题 | 说明 | 推荐资源 |
|------|------|----------|
| 什么是 LLM | Transformer 架构、Token、上下文窗口 | [3Blue1Brown 神经网络系列](https://www.youtube.com/playlist?list=PLZHQObOWTQDNU6R1_670wDrpZTy3wrIa0) |
| 模型训练流程 | 预训练 → 指令微调 → RLHF | [OpenAI 文档](https://platform.openai.com/docs) |
| 模型能力边界 | 幻觉、上下文长度、知识截止日期 | 实际使用中积累 |
| 主流模型对比 | GPT-4, Claude, Llama, GLM | [LMSYS Leaderboard](https://lmssy.org) |

### 实践
- [ ] 注册并使用至少 2 个主流大模型 API
- [ ] 用同一个问题测试不同模型，感受差异
- [ ] 理解 Token 计费方式

## 第二阶段：Prompt 工程（Day 11-20）

### 目标
掌握与 AI 高效沟通的技巧。

### 核心技巧

```text
1. 角色设定    → "你是一位资深 Python 工程师"
2. 上下文提供  → 给足够的背景信息
3. 格式约束    → "用 JSON 格式输出"
4. 示例引导    → Few-shot learning
5. 思维链      → "请一步步分析"
6. 自我检查    → "请检查上面的回答是否有误"
```

### 实践
- [ ] 用 Prompt 完成一个实际任务（写代码/分析数据/翻译润色）
- [ ] 尝试 Few-shot 和 Chain-of-Thought 技巧
- [ ] 对比不同 Prompt 的输出质量

## 第三阶段：API 开发（Day 21-35）

### 目标
用代码调用大模型 API，构建简单应用。

### 示例：第一个 AI 应用

```python
import openai

client = openai.OpenAI(api_key="your-api-key")

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "system", "content": "你是一个友善的助手。"},
        {"role": "user", "content": "解释什么是向量数据库"}
    ],
    temperature=0.7
)

print(response.choices[0].message.content)
```

### 实践
- [ ] 搭建一个命令行 AI 对话工具
- [ ] 实现 Stream/流式输出
- [ ] 添加上下文记忆（多轮对话）
- [ ] 了解 Function Calling / Tool Use

## 第四阶段：RAG 检索增强生成（Day 36-50）

### 目标
让 AI 基于你的私有数据回答问题。

### 技术栈

```
文档 → 分块(Chunking) → 向量化(Embedding) → 向量数据库 → 检索 → LLM 生成
```

| 组件 | 推荐方案 |
|------|----------|
| Embedding 模型 | OpenAI text-embedding, BGE |
| 向量数据库 | ChromaDB, FAISS, Qdrant |
| 框架 | LangChain, LlamaIndex |

### 实践
- [ ] 用 LangChain 搭建文档问答系统
- [ ] 用自己的笔记/文档做知识库
- [ ] 测试不同 Chunk 大小对效果的影响

## 第五阶段：Agent 开发（Day 51-60）

### 目标
开发能自主使用工具、完成复杂任务的 AI Agent。

### 核心概念

- **ReAct 模式**：Reasoning（推理）+ Acting（行动）循环
- **Tool Use**：让 AI 调用搜索、代码执行、API 等外部工具
- **Multi-Agent**：多个 Agent 协作完成复杂任务

```python
# Agent 的核心循环
def agent_loop(task):
    while not task.done():
        thought = llm.think(task.state)     # 思考下一步
        action = llm.decide(thought)        # 选择行动
        result = execute_tool(action)        # 执行工具
        task.update(result)                  # 更新状态
    return task.final_answer
```

### 实践
- [ ] 实现一个能搜索网页回答问题的 Agent
- [ ] 搭建一个能读写文件的 Agent
- [ ] 尝试多 Agent 协作（如 CrewAI / Hermes Agent）

## 学习资源汇总

### 入门
- 📖 [OpenAI Cookbook](https://cookbook.openai.com/) — 实战示例库
- 📖 [LangChain 文档](https://python.langchain.com/) — LLM 应用框架
- 🎥 [Andrej Karpathy YouTube](https://www.youtube.com/@AndrejKarpathy) — 深入理解 LLM

### 进阶
- 📖 [Prompt Engineering Guide](https://www.promptingguide.ai/)
- 📄 [Attention Is All You Need](https://arxiv.org/abs/1706.03762) — Transformer 原论文
- 🎥 [Karpathy LLM 101](https://www.youtube.com/watch?v=zjkBMFhNj_g)

### 中文资源
- 📖 [LangChain 中文文档](https://python.langchain.com.cn/)
- 📖 [WaytoAGI](https://waytoagi.com/) — 中文 AI 知识库

## 学习心得

> **动手 > 看教程**。每个阶段都要有实际产出，哪怕是很小的项目。

> **不要追求「完全理解」再开始**。AI 领域发展太快，边学边做是最好的策略。

> **记录学习过程**。写笔记、发博客、分享代码，这些都会加深你的理解。

---

这篇路线图会随着我的学习进度持续更新。如果你也在学习 AI，欢迎在评论区交流！
