---
tags: [AI应用, RAG, 检索增强生成, 面试]
priority: P0
status: learning
last_review: 2026-08-29
---

# RAG 核心链路

## 一句话结论

RAG 的关键不是“接一个向量库”，而是把可信文档加工成可检索片段，在查询时召回、过滤和排序相关证据，再让模型基于证据回答并保留引用。

## 两条链

### 离线/启动索引链

```text
原始文档
→ 清洗与按标题切 section
→ 按合理粒度切 chunk
→ 生成 embedding
→ 保存向量、文本和元数据
→ 服务启动时加载或构建索引
```

### 在线查询链

```text
用户问题
→ 结合历史改写为独立问题
→ 生成 query embedding
→ 按意图 / 领域做 metadata filter
→ 相似度召回
→ 词法、主题、优先级和来源重排
→ 去重并取 Top-K
→ 置信度判断
→ 把片段和引用交给模型
→ 无可靠证据时澄清或安全兜底
```

## 从 langchain4j 截图提炼的可搜索概念

- RAG（Retrieval-Augmented Generation）：检索增强生成，用外部知识补足模型的时效性和幻觉风险。
- 建立索引通常包含文档收集、预处理、文档切片、Embedding、向量表示和向量存储。
- 切片可按固定大小、语义边界或递归分割；切太大检索不准，切太小又缺上下文。
- 查询阶段先把问题转成向量，在向量库做相似度搜索，再对相关片段排序。
- Top-K 片段会和用户问题合并成增强 Prompt，模型负责组织最终回答。
- 标准化 RAG 可以替换文档加载器、Embedding 模型和内容检索器，灵活度更高但维护成本也更高。
- `DocumentLoader` / `DocumentSplitter` / `EmbeddingModel` / `EmbeddingStore` / `ContentRetriever` 是可搜索的组件关键词；它们是学习资料中的框架抽象。
- 引用来源和检索分数可以返回给前端，帮助用户判断回答依据；引用不是事实正确的绝对证明。

## SmartRenew 实际实现映射

以下结论来自项目源码与 `backend/src/main/resources/ai/knowledge` 资源，状态只描述当前能核对到的代码：

- **[已实现] 文档加载**：`KnowledgeBaseLoader` 从 `ai/knowledge/*.md` 加载知识，并按标题 section 和 token 粒度切块。
- **[已实现] 索引**：`KnowledgeEmbeddingIndex` 维护内存索引；`LocalHashingEmbeddingModel` 提供本地 Hashing Embedding。
- **[已实现] 检索**：`KnowledgeRetriever` 支持查询改写、Embedding 搜索、严格/宽松/仅范围过滤、去重和重排。
- **[已实现] 证据返回**：检索结果转成 `AiCitationDTO`，回答中可以携带来源和摘要。
- **[已实现] 置信度控制**：`AiRagConfidenceGuard` 根据无结果、最高分和来源混杂等情况判断是否应该澄清或兜底。
- **[已实现] 失败降级**：`AiChatServiceImpl` 在模型关闭、Key 缺失或模型失败时，可基于知识片段返回兜底，或明确提示不可用。
- **[边界] 当前证据没有证明使用外部向量数据库、复杂 reranker 或生产级分布式索引**；面试不要擅自扩大为这些技术。

## 检索质量怎么排查

1. **没有召回**：看文档是否加载、切片是否为空、Embedding 是否异常、过滤条件是否过严。
2. **召回不相关**：看 query rewrite、切片边界、关键词、向量模型和重排权重。
3. **召回了错误口径**：给国家政策、平台规则、状态说明加 scope/source metadata，并按意图过滤。
4. **答案有证据却仍然编造**：检查 Prompt 是否明确“只能基于片段”、上下文是否混入无关内容，以及是否把低分结果交给了模型。
5. **资料过期**：记录文档版本、更新时间和来源；必要时重新建索引，而不是只调 Top-K。
6. **引用不可信**：引用应来自实际召回 chunk，不能由模型自行编造 URL、标题或政策数字。

## 容易答错

- “RAG 等于向量数据库”：向量库只是存储/检索组件，文档治理、切片、过滤、重排、Prompt、引用和降级同样重要。
- “Top-K 越大越好”：会增加噪声和 token 成本，应该结合问题和上下文预算。
- “命中片段就一定正确”：检索相似不等于事实可靠，还要看来源、版本和业务口径。
- “有 RAG 就不需要模型”：RAG 负责提供证据，模型负责语言组织；也可以在模型不可用时走模板/片段兜底。
- “Embedding 分数是概率”：相似度分数只是排序信号，阈值要结合实现和测试校准。

## Java / SmartRenew 关联

- SmartRenew 的 AI 业务问题先经过意图路由，再按政策、平台规则、材料、状态等范围检索。
- RAG 只负责政策与流程解释，不参与审核结论、额度扣减、发券和核销决策。
- AI 知识资源与 SmartRenew 业务主线可结合 [[SR/SmartRenew面试精炼笔记/00-SmartRenew项目知识地图]] 复习，但项目事实仍以实际资源和代码为准。

## 高频追问

1. 为什么要做 query rewrite？
2. 如何避免国家政策和平台演示规则混答？
3. 切片粒度如何影响召回和答案质量？
4. 没有召回、低置信度和模型失败分别怎么降级？
5. 为什么不能把 RAG 结果直接当作业务最终事实？

## Reference

- [[langchain4j]]
- [[SR/SmartRenew面试精炼笔记/00-SmartRenew项目知识地图]]
- [[AI应用工程面试精炼笔记/02-Context-Memory与Structured-Output]]
