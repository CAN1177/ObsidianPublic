
> 原文：https://my.feishu.cn/wiki/UjrXw5FaMi0L7nktVg7c9o9cnzh

---

## 一、核心问题：大模型的幻觉

大模型的知识取决于训练数据集，它不知道：
- **最近发生的事情**（训练截止后的新信息）
- **企业内部私有文档**

但它不会说"不知道"，而是**胡乱编造**，这就是「幻觉」。

---

## 二、RAG 是什么

**RAG = Retrieval 检索 + Augmented 增强 + Generation 生成**

三步走：
1. **检索**：从知识库中查到与用户问题相关的文档片段
2. **增强**：把文档片段作为背景知识放入 Prompt
3. **生成**：大模型基于增强后的 Prompt 生成回答

> 一句话：在 Prompt 给到大模型之前，先查知识库，把相关文档塞进去。

---

## 三、为什么需要向量

关键词搜索**无法**实现语义搜索。例如：

- 用户搜「水果」→ 需要匹配「苹果」「香蕉」「草莓」

### 向量化的直觉理解

假设用两个维度：**食用性** 和 **硬度**（0~1）：

| 概念 | 食用性 | 硬度 | 向量 |
|------|--------|------|------|
| 水果 | 0.9 | 0.3 | [0.9, 0.3] |
| 苹果 | 0.9 | 0.5 | [0.9, 0.5] |
| 香蕉 | 0.9 | 0.1 | [0.9, 0.1] |
| 石头 | 0.1 | 0.9 | [0.1, 0.9] |

→ 水果、苹果、香蕉向量相近；水果和石头差距大。

### 余弦相似度

通过两个向量**夹角的余弦值**判断相似度——夹角越小，相似度越高。实际应用中是几百维的向量，原理相同。

**通过向量计算实现语义检索！**

---

## 四、嵌入模型

- **Embedding Model**：专门把知识（文本/图片/语音）转成向量
- 和大语言模型（LLM）是不同的模型，价格便宜很多
- 向量化时会在元信息中记录来源文档

---

## 五、RAG 完整流程

```
用户 Prompt → 嵌入模型向量化 → 向量数据库中做相似度检索
→ 找到语义最相近的文档块 → 加入 Prompt 作为背景知识 → 大模型生成回答
```

---

## 六、代码实践（LangChain）

```js
// 核心依赖
@langchain/core
@langchain/openai
@langchain/classic

// 三个关键对象
const model = new ChatOpenAI({...});       // 大语言模型
const embeddings = new OpenAIEmbeddings({...}); // 嵌入模型

// 核心 API 调用链
const vectorStore = await MemoryVectorStore.fromDocuments(documents, embeddings);
const retriever = vectorStore.asRetriever({ k: 3 });       // 取相似度最高的3个
const retrievedDocs = await retriever.invoke(question);     // 语义检索
const scoredResults = await vectorStore.similaritySearchWithScore(question, 3); // 含评分

// 检索结果注入 Prompt，再给大模型回答
```

### 关键 API 总结

| API | 作用 |
|-----|------|
| `fromDocuments(docs, embeddings)` | 向量化文档并存入数据库 |
| `asRetriever({ k })` | 指定查询相似度最大的 k 个文档 |
| `similaritySearchWithScore(query, k)` | 带相似度评分的检索 |
| `retriever.invoke(query)` | 执行语义检索 |

---

## 七、总结

1. **幻觉问题**的解法：先检索知识库，将相关文档注入 Prompt
2. **关键词做不到语义搜索**，必须用向量 + 余弦相似度
3. **嵌入模型**专门负责向量化，与 LLM 是两个模型
4. **LangChain 的 RAG 流程**：存入文档 → 向量化 → 检索 → 增强 Prompt → 生成回答
5. 企业**内部文档智能助手**，本质上就是一个 RAG 应用
