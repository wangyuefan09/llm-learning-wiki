---
标题: RAG评估与可观测性
类型: 学习地图
主题: RAG
学习顺序: 9
状态: 已整理
创建日期: 2026-07-23
更新日期: 2026-07-24
来源:
  - 用户原始学习记录
  - 用户评估专题整理
  - https://github.com/wangyuefan09/agentic-rag
---

# RAG评估与可观测性

## 评估要解决的问题

RAG 评估不是只计算一个总分，而是回答五个问题：

1. 原始文档是否被正确解析；
2. 知识是否被组织成合理的 chunk；
3. 检索是否找到了支撑答案的证据；
4. 生成结果是否忠于证据、正确且完整；
5. Agent 的路由、改写和循环是否带来了足够收益。

完整的评估链路是：

```text
构建可复用评估集
→ 分层定义指标
→ 建立基线
→ 单变量对比
→ 按问题类型分组
→ 分析失败样本
→ 通过 trace 定位问题环节
```

## 专题学习顺序

1. [[09-1-RAG评估集与实验设计|RAG评估集与实验设计]]
2. [[09-2-RAG检索评估指标|RAG检索评估指标]]
3. [[09-3-RAG生成与引用评估|RAG生成与引用评估]]
4. [[09-4-Agentic RAG评估|Agentic RAG评估]]
5. [[09-5-RAG问题定位与评估面试问答|RAG问题定位与评估面试问答]]

## 评估分层

| 层级 | 核心问题 | 典型指标 |
| --- | --- | --- |
| 文档解析 | 正确知识是否进入系统 | 解析成功率、阅读顺序错误率、表格结构准确率 |
| Chunk | 知识边界是否合理 | 长度分布、fallback 比例、语义完整性 |
| 检索 | 是否找到并排好证据 | Hit Rate@K、Recall@K、Precision@K、MRR、nDCG@K |
| 生成 | 答案是否可信、切题和完整 | Faithfulness、Answer Relevance、Correctness、Completeness |
| 引用与拒答 | 结论是否有证据，无答案时是否正确兜底 | 引用正确率、引用覆盖率、误答率、错误拒答率 |
| Agent | 动态决策是否有净收益 | 守卫准确率、Rewrite 成功率、证据误删率、循环次数 |
| 系统 | 质量收益的代价是什么 | P50/P95 延迟、Token、模型调用次数、失败率 |

## 可观测性和评估的区别

- 可观测性回答“这次请求经过了什么步骤，哪一步耗时或失败”。
- 评估回答“返回的证据和答案质量怎么样，新版本是否优于基线”。

Langfuse 适合记录 trace、span、generation、耗时、Token 和用户反馈；Ragas 或自定义评估器适合对固定数据集做离线回放。两者可以组合，但不能互相替代。

## 项目当前实现

### Langfuse

当前代码已经实现：

- 普通 RAG 的请求、query embedding、检索、Prompt 和生成观测；
- Agentic RAG 的请求级 trace；
- guardrail、检索、评分、改写和生成等节点观测；
- `trace_id` 返回和 `/feedback` 评分接口；
- 正常零结果与检索基础设施异常的区分；
- 应用关闭时执行 Langfuse flush/shutdown。

源码位置：

- `src/services/langfuse/client.py`
- `src/services/langfuse/tracer.py`
- `src/services/agents/agentic_rag.py`
- `src/routers/agentic_ask.py`

### 评估流水线

当前项目没有 Ragas 依赖、正式风电评估集和离线评估脚本。因此，可观测能力是已实现项，检索与答案质量结论仍是待验证项。

## 问题定位原则

当答案错误时，不先调 Prompt，而是按以下顺序检查：

```text
原文中是否有答案？
→ 解析结果中是否保留了答案？
→ 正确证据是否进入合理 chunk？
→ 正确 chunk 是否进入 Top-K？
→ 是否被权限或 metadata filter 过滤？
→ 是否被文档评分节点误删？
→ LLM 是否忽略或曲解了已给证据？
```

这个顺序用于区分解析、chunk、检索、路由和生成问题，避免只看最终答案猜测根因。

## 面试回答主线

> 我会把 RAG 评估拆成文档解析、chunk、检索、生成和 Agent 工作流五层。检索层用带相关文档或 chunk 标注的问题集，通过 Hit Rate@K、Recall@K、MRR 和 nDCG@K 对比 BM25、纯向量和 RRF；生成层关注 Faithfulness、Answer Relevance、Correctness、完整性和引用正确性；Agent 还要评估守卫准确率、Query Rewrite 的检索增益、证据误删率、延迟和 Token 代价。最后我会用 Langfuse 保留链路证据，结合自动指标和人工抽样分析失败样本。

## 顺序导航

- 上一节：[[08-Agentic RAG工作流|Agentic RAG工作流]]
- 专题第一节：[[09-1-RAG评估集与实验设计|RAG评估集与实验设计]]
- 下一主节：[[10-RAG知识增强与面试问题清单|RAG知识增强与面试问题清单]]
- 返回索引：[[00-RAG学习索引|RAG学习索引]]
