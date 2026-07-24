---
标题: 向量数据库与ANN索引
类型: 学习笔记
主题: RAG
学习顺序: 6
状态: 已整理
创建日期: 2026-07-23
更新日期: 2026-07-24
来源:
  - 用户原始学习记录
  - https://github.com/wangyuefan09/agentic-rag
---

# 向量数据库与ANN索引

## 向量数据库解决什么问题

传统数据库擅长精确条件、范围、关联和事务查询；向量检索系统擅长按距离寻找语义上相近的对象。很多产品同时支持结构化字段、全文检索和向量检索，因此二者不是严格对立关系。

常见方案：

- 专用向量数据库：Milvus、Qdrant、Weaviate 等；
- 传统系统扩展：PostgreSQL + pgvector、OpenSearch、Elasticsearch、MongoDB Atlas Vector Search；
- 云服务：各云厂商托管的向量检索或搜索产品。

## OpenSearch和pgvector的取舍

### OpenSearch

适合同时需要 BM25、向量检索、字段过滤、聚合和混合排序的搜索场景。工程上可以在同一个查询系统中完成关键词与语义召回。

### pgvector

适合结构化业务数据和向量数据规模可控、希望减少基础设施种类、并依赖 PostgreSQL 事务与 SQL 过滤的场景。

不能简单使用“百万以下一定选 pgvector，百万以上一定选 Milvus”这类阈值。硬件、索引参数、查询过滤、写入方式和延迟目标都会影响结果。

## 项目当前选型

`agentic-rag` 使用：

- PostgreSQL 保存文档元数据、状态、处理租约、版本信息和表格原始结构；
- OpenSearch 保存正文/表格 chunk、业务过滤字段、权限字段和向量；
- OpenSearch 使用 BM25、HNSW 和 RRF 完成混合检索。

主要索引字段：

### 标识与版本

```text
chunk_id、doc_id、kb_id、chunk_index、version、status
```

### 正文与表格

```text
chunk_text、title、section_title
chunk_type、table_id、row_start、row_end
```

### 风电业务字段

```text
document_type、wind_farm_id、turbine_model、component、fault_code
```

### 权限与溯源

```text
permission_ids、source_filename、content_hash、embedding_model、created_at、updated_at
```

Mapping 使用 `dynamic=strict`，并在 `_meta.schema_version` 中保存版本。应用启动时校验 schema version、表格字段、向量字段类型和维度；不兼容时停止启动，不自动删除索引。

## 向量维度的存储估算

一个 float32 占 4 字节。以 2560 维为概念示例：

```text
2560 × 4 = 10240 字节 ≈ 10 KiB
```

一百万条原始向量约 10.24 GB（十进制），但实际磁盘和内存还包括 HNSW 图、文本字段、倒排索引、副本和系统开销。

当前项目示例配置是 1024 维占位值，不绑定 2560 维模型，也没有一百万 chunk 的实际统计。维度估算属于通用容量分析，不能写成当前项目生产数据。

## 精确搜索与近似搜索

### 精确KNN

对全部候选向量计算距离，再选择最近的 K 个。结果精确，但计算量随数据规模增加。

### ANN

通过图、聚类或量化减少需要比较的候选数量，以少量召回损失换取查询速度。生产向量检索通常使用 ANN。

## HNSW

HNSW 使用多层近邻图。查询从稀疏高层快速定位区域，再进入更密的底层寻找近邻。

优势：

- 查询延迟通常较低；
- 召回率通常较高；
- 支持增量插入。

代价：

- 图结构占用额外内存；
- 构建成本较高；
- 参数会影响召回、延迟和空间。

项目 Mapping 当前配置：

- engine：`nmslib`；
- `m=16`；
- `ef_construction=512`；
- distance：`cosinesimil`。

`ef_search` 没有在当前 Mapping 中显式配置，不能把某个值描述成项目实际调优参数。

## IVF-Flat

IVF 先把向量聚类到多个倒排单元，查询时只搜索最接近的若干簇，再在簇内做距离计算。

优势是索引结构相对直观、内存压力通常低于 HNSW；不足是效果对聚类和探测簇数量敏感，数据分布变化后可能需要重建或重新训练索引。

## 核心调优参数

### HNSW

- `m`：每个节点的连接规模，增大通常提高召回并增加内存；
- `ef_construction`：建图搜索范围，增大通常提高索引质量并减慢构建；
- `ef_search`：查询搜索范围，增大通常提高召回并增加延迟。

### IVF

- 聚类簇数量；
- 查询时探测的簇数量；
- 训练样本与实际数据分布的一致性。

## 项目当前文档更新流程

### 新增

1. 对源文件计算 SHA-256；
2. 使用 `(kb_id, content_hash)` 唯一约束去重；
3. 解析、切分、向量化；
4. 先写 staging 切片；
5. 校验后切换为 ready；
6. 更新 PostgreSQL 文档状态。

### 修改或重新索引

1. 通过数据库 token 和租约取得 reindex 权限；
2. 生成 `version + 1` 的 staging 切片和表格；
3. 校验 Embedding 数量、批量写入和版本状态；
4. 切换活动版本；
5. 更新 PostgreSQL；
6. 清理旧版本；
7. 失败时执行补偿回滚。

### 删除

1. CAS 抢占 `deleting` 和删除租约；
2. 下线切片并确认 ready 数量为零；
3. 数据库标记 `deleted`；
4. 物理删除切片并确认无残留；
5. 删除本地源文件；
6. 中断后由新 token 接管过期状态继续清理。

这是跨 PostgreSQL 和 OpenSearch 的可验证切换、租约和补偿方案，不是严格的分布式事务。

## 权限过滤与ANN

项目在构造 BM25 和向量查询时都加入相同的业务字段、状态和权限 filter，避免先召回无权限内容再在应用层过滤。

当前公开入口只允许检索公开文档；可信权限上下文需要未来认证层在服务端创建。已有过滤能力不等于完整认证系统。

## 常见瓶颈

- chunk 过多导致向量和图索引膨胀；
- 高频过滤字段 Mapping 不合理；
- 高选择性 filter 影响 ANN 可用候选；
- 候选集过大导致融合或评分变慢；
- 真正延迟可能来自 Embedding、LLM 文档评分或生成，而不是 OpenSearch。

定位时应对各阶段分别计时，避免把端到端慢查询直接归因于向量库。

## 关联笔记

- [[05-Embedding与语义空间|Embedding与语义空间]]
- [[07-混合检索与结果融合|混合检索与结果融合]]
- [[09-RAG评估与可观测性|RAG评估与可观测性]]

## 顺序导航

- 上一节：[[05-Embedding与语义空间|Embedding与语义空间]]
- 下一节：[[07-混合检索与结果融合|混合检索与结果融合]]
