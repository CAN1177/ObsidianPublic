# LLM Memory 管理

## 核心问题

大模型是**无状态**的，每次调用不记得之前聊过什么。需要我们自己管理 Messages（对话历史），才能让模型延续话题。

## ChatMessageHistory

LangChain 封装了 `ChatMessageHistory` API，用于存储 messages，支持多种后端：

- 内存
- Redis
- 数据库
- ……

> ⚠️ LangChain 旧的 Memory API 已废弃，因为完全可以自己实现。

## 三种 Memory 管理策略

| 策略 | 做法 | 说明 |
|------|------|------|
| **截断 (Truncation)** | 超出条数/Token 上限时，直接丢弃最早的 message | 简单粗暴 |
| **总结 (Summarization)** | 调用大模型生成对话摘要，用摘要替代原始 messages | Cursor 的做法：超 Token 触发总结 |
| **检索 (RAG)** | 对话存入向量数据库，通过语义检索召回相关历史 | 精准定位相关上下文 |

## 进阶玩法：总结 + 检索

在 Milvus 中存储对话摘要，结合语义检索，既能压缩历史又能精准召回，是做 AI Agent 的标配。

## 一句话

> 管好 Memory = 随时接上之前的话题，这是 AI Agent 开发的基础能力。
