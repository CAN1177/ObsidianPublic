# Milvus 向量数据库 — 复习笔记

---

## 一、为什么需要向量数据库？

| 场景 | MySQL | Milvus |
|------|-------|--------|
| 精确查询（id=123） | ✅ 擅长 | ❌ 不适合 |
| 关键词模糊搜索（LIKE） | ✅ 勉强可用 | ❌ 不适合 |
| 多表关联查询（JOIN） | ✅ 核心能力 | ❌ 不支持 |
| 语义检索（"今天开心的那件事"） | ❌ 做不到 | ✅ 核心能力 |
| 相似度检索（相似图片/文本推荐） | ❌ 做不到 | ✅ 核心能力 |
| RAG 知识库 / 长期记忆 | ❌ 不适合 | ✅ 最佳实践 |

**一句话总结**：MySQL 做**结构化精确查询**，Milvus 做**非结构化语义检索**，各司其职。

---

## 二、Milvus 核心概念

### 层级结构

```
┌─────────────────────────────────┐
│          Milvus 集群            │
│  ┌───────────────────────────┐  │
│  │       Database            │  │  ← 逻辑隔离，类似 MySQL 的 database
│  │  ┌─────────────────────┐  │  │
│  │  │     Collection       │  │  │  ← 类似 MySQL 的 table
│  │  │  ┌───────────────┐  │  │  │
│  │  │  │    Entity      │  │  │  │  ← 类似 MySQL 的 row
│  │  │  │  (数据行)       │  │  │  │
│  │  │  └───────────────┘  │  │  │
│  │  └─────────────────────┘  │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

### 关键概念对照

| Milvus | MySQL | 说明 |
|--------|-------|------|
| Database | Database | 逻辑数据库 |
| Collection | Table | 存储数据的集合，需预先定义 Schema |
| Entity | Row | 单条数据记录 |
| Field | Column | 字段，标量字段或向量字段 |
| Vector Field | — | Milvus 特有，存储 embedding 向量 |
| Index | Index | 向量索引（IVF_FLAT、HNSW 等） |
| Partition | Partition | 物理分区，加速查询 |

### Schema 设计要点

- **主键字段**（`DataType.VARCHAR` 或 `DataType.INT64`，需开启 `autoID` 或手动赋值）
- **标量字段**（存 id、标题、时间戳等元数据）
- **向量字段**（存 embedding，需指定维度 `dim`，如 OpenAI 的 1536 维）
- Collection 创建后不可修改 Schema，需提前规划好

### 索引类型

| 索引 | 原理 | 适用场景 |
|------|------|----------|
| **FLAT** | 暴力搜索，100% 召回 | 小数据集，追求精度 |
| **IVF_FLAT** | 聚类 + 暴力搜索 | 中等规模，平衡精度和速度 |
| **IVF_PQ** | 聚类 + 乘积量化压缩 | 大规模，内存敏感 |
| **HNSW** | 分层可导航小世界图 | 高精度 + 高速，内存占用大 |
| **SCANN** | 量化 + 图索引 | 超大规模，Google ScaNN 算法 |

---

## 三、CRUD 操作概览（Node.js SDK）

### 连接 Milvus

```typescript
import { MilvusClient } from '@zilliz/milvus2-sdk-node';

const client = new MilvusClient({
  address: 'localhost:19530',
  username: 'root',
  password: 'Milvus',
});
```

### 创建 Database

```typescript
await client.createDatabase({ db_name: 'my_ai_app' });
await client.use({ db_name: 'my_ai_app' });
```

### 创建 Collection + Schema

```typescript
await client.createCollection({
  collection_name: 'diary_entries',
  fields: [
    { name: 'id', data_type: 'VarChar', is_primary_key: true, max_length: 36 },
    { name: 'title', data_type: 'VarChar', max_length: 512 },
    { name: 'content', data_type: 'VarChar', max_length: 65535 },
    { name: 'timestamp', data_type: 'Int64' },
    { name: 'embedding', data_type: 'FloatVector', dim: 1536 },
  ],
});
```

### 创建索引

```typescript
await client.createIndex({
  collection_name: 'diary_entries',
  field_name: 'embedding',
  index_name: 'embedding_idx',
  index_type: 'IVF_FLAT',
  metric_type: 'COSINE',
  params: { nlist: 128 },
});
```

### 插入数据（Insert）

```typescript
await client.insert({
  collection_name: 'diary_entries',
  data: [{
    id: uuidv4(),
    title: '今天天气真好',
    content: '阳光明媚...',
    timestamp: Date.now(),
    embedding: await getEmbedding('今天天气真好：阳光明媚...'),
  }],
});
```

### 语义检索（Search）

```typescript
const results = await client.search({
  collection_name: 'diary_entries',
  vector: await getEmbedding('上次去公园开心的一天'),
  limit: 5,
  output_fields: ['title', 'content', 'timestamp'],
});
```

### 删除

```typescript
await client.delete({
  collection_name: 'diary_entries',
  filter: 'id == "xxx"',
});
```

---

## 四、RAG 流程 + Milvus（AI 日记本实战）

```
                        ┌──────────────────┐
                        │    用户提问       │
                        │ "开心的一天"       │
                        └────────┬─────────┘
                                 │
                                 ▼
                        ┌──────────────────┐
                        │  Embedding 模型   │
                        │  文本 → 向量      │
                        └────────┬─────────┘
                                 │
                                 ▼
                        ┌──────────────────┐
                        │   Milvus 检索     │
                        │  Top-K 最相关日记  │
                        └────────┬─────────┘
                                 │
                                 ▼
                        ┌──────────────────┐
                        │   组装 Prompt     │
                        │  检索结果 + 问题   │
                        └────────┬─────────┘
                                 │
                                 ▼
                        ┌──────────────────┐
                        │     LLM 回答      │
                        │  "你在3月15日记录  │
                        │   的那次公园散步..." │
                        └──────────────────┘
```

### 核心代码模式

1. **写日记** → 生成 embedding → 存入 Milvus（同时存 MySQL 做持久化备份）
2. **查日记** → 用户自然语言 → 生成 embedding → Milvus 语义检索 Top-K → 返回结果
3. **RAG 对话** → 检索结果注入 LLM context → LLM 基于日记内容回答问题

---

## 五、双写模式

生产实践中，MySQL 和 Milvus 通常**同时写入**：

```
写请求
  │
  ├─→ MySQL  存储完整数据（标题、内容、时间戳、用户ID...）
  │     作用：精确查询、多表关联、数据备份
  │
  └─→ Milvus  存储向量数据（id + embedding + 摘要）
        作用：语义检索、相似推荐
```

### 双写注意事项

- 用**消息队列**或**事务发件箱模式**保证最终一致性
- Milvus insert 后需调用 `flush` 或等待数据落盘才能被检索到
- MySQL 保留全量数据，Milvus 故障时可从 MySQL 重建向量索引

---

## 六、PostgreSQL（pgvector）对比

PostgreSQL 通过 **pgvector** 扩展也能做向量检索，是 Milvus 的重要替代方案。

| 维度 | Milvus | PostgreSQL + pgvector |
|------|--------|----------------------|
| **定位** | 专业向量数据库 | 关系型数据库 + 向量扩展 |
| **向量检索算法** | FLAT / IVF_FLAT / IVF_PQ / HNSW / SCANN / GPU 加速 | IVFFlat / HNSW |
| **索引类型丰富度** | ⭐⭐⭐⭐⭐ 很多种 | ⭐⭐⭐ 基础够用 |
| **向量维度上限** | 32,768 | 16,000（可调） |
| **精确查询 + JOIN** | ❌ 弱（新版本支持标量过滤，但无 JOIN） | ✅ 原生 SQL，多表关联强 |
| **语义检索（亿级）** | ⭐⭐⭐⭐⭐ 分布式架构，高并发 | ⭐⭐⭐ 受限于单机 PG，需分库 |
| **部署复杂度** | 需要独立服务（etcd + MinIO + Milvus） | 只需装一个 PG 扩展 |
| **数据一致性** | 双写可能不一致 | ✅ 事务内同时写向量和标量 |
| **内存占用** | 较高（独立进程） | 和 PG 共享内存，相对可控 |
| **运维成本** | 多组件，需 docker compose | 单一 PG 实例 |
| **生态成熟度** | 向量检索领域标杆，社区活跃 | PG 生态成熟，扩展逐渐完善 |
| **GPU 加速** | ✅ 支持 | ❌ 不支持 |

### 选择建议

| 场景 | 推荐 |
|------|------|
| 数据量百万级以下，已有 PG，不想加新组件 | **pgvector** |
| 需要强事务一致性（向量+业务数据原子写入） | **pgvector** |
| 亿级以上向量数据，高并发检索 | **Milvus** |
| 需要多种索引策略调优 | **Milvus** |
| AI Agent 项目，MVP 阶段 | **pgvector 快速起步** → 规模大了迁移 Milvus |
| 生产级 AI 知识库 / 记忆系统 | **Milvus**（专业、稳定、性能好） |

### pgvector 示例速览

```sql
-- 安装扩展
CREATE EXTENSION vector;

-- 建表
CREATE TABLE diary (
  id UUID PRIMARY KEY,
  title TEXT,
  content TEXT,
  embedding vector(1536)
);

-- 创建索引
CREATE INDEX ON diary USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);

-- 语义检索
SELECT title, content, 1 - (embedding <=> '[0.1, 0.2, ..., 0.8]') AS similarity
FROM diary
ORDER BY embedding <=> '[0.1, 0.2, ..., 0.8]'
LIMIT 5;
```

> `<=>` 是余弦距离操作符，`<#>` 是负内积，`<->` 是 L2 距离。

---

## 七、AI Agent 项目中的应用

Milvus / 向量数据库在 AI Agent 中承担的关键能力：

### 1. 长期记忆（Long-term Memory）
- 存储用户历史对话摘要 + embedding
- 新对话时检索相关历史，注入 context
- 让 Agent "记住"用户的偏好、背景、历史决策

### 2. 知识库（Knowledge Base / RAG）
- 产品文档、企业 wiki、FAQ → chunk → embedding → Milvus
- 用户提问时检索相关知识片段
- 实现"基于私域知识回答问题"

### 3. 工具/函数检索（Tool Retrieval）
- Agent 有上百个 tool 时，根据用户意图检索最相关的 tool
- 比全量塞 prompt 更省 token

### 4. 多模态检索
- 图片搜索（CLIP embedding）
- 视频片段检索
- 跨模态问答

### 5. 推荐系统
- 基于用户行为 embedding 做相似推荐
- 内容推荐、用户匹配等

---

## 八、面试/简历关键词

写简历和面试时可以围绕这些点展开：

- **"基于 Milvus 构建了 Agent 长期记忆系统"** → 讲 schema 设计（用户 id + 会话摘要 + embedding + 时间戳），语义检索历史对话
- **"使用 Milvus + RAG 实现企业知识库问答"** → 讲文档切分策略、embedding 选型、检索 Top-K 调优
- **"MySQL + Milvus 双写架构"** → 讲一致性保障、故障恢复、从 MySQL 重建向量索引
- **"针对不同场景选型：pgvector vs Milvus"** → 体现技术视野和架构决策能力
- **"Milvus 索引选型和性能调优"** → IVF_FLAT vs HNSW 的取舍、nlist/M 参数调优

---

## 九、常用命令速查

```bash
# Docker Compose 启动 Milvus
docker compose up -d

# 检查服务状态
docker compose ps

# Attu GUI 默认地址
open http://localhost:3000

# 停止
docker compose down
```

---

## 十、关键总结

1. **MySQL 精确查，Milvus 语义查** —— 分工明确，互相补充
2. **Schema 先行** —— Collection 创建后不能改结构，要提前设计好
3. **索引影响检索性能** —— HNSW 最快但占内存，IVF_FLAT 平衡，FLAT 太慢
4. **双写是生产标配** —— MySQL 做数据底座，Milvus 做检索加速
5. **pgvector 是轻量替代** —— 数据量不大时，用 PG 一键搞定，免维护成本
6. **面试必备** —— 向量数据库是 AI Agent 的标配基础设施，每个 Agent 项目都能讲出故事
