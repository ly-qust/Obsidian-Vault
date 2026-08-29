---
tags: [SmartRenew, AI应用, RAG, 项目面试]
priority: P0
status: learning
last_review: 2026-08-29
---

# SmartRenew AI 项目映射与高频问答

## 一句话结论

SmartRenew 的 AI 是“受控的只读政策与流程问答”：请求经过鉴权、上下文、意图路由、RAG、可选模型调用、引用和降级，不参与审核、额度、发券、核销等写业务。

## 真实请求链

```text
POST /api/v1/ai/chat
→ Security 鉴权并取得当前 userId
→ AiChatController 接收 AiChatRequest
→ PromptSafetyGuard 校验输入
→ conversationId + userId 读取有限历史
→ FollowUpQuestionResolver 处理上下文追问
→ AiQuestionRouter 判断日期 / 元对话 / 业务问答 / 模型问答
→ 业务问题进入 KnowledgeRetriever
→ DeepSeekAiModelClient 可选调用模型
→ AiBusinessAnswerComposer 组织答案
→ 返回 AiChatResponse、引用、route、fallback 状态
→ AiChatLogRecorder 记录链路与结果
```

## 证据分级

### [已实现]

- `/api/v1/ai/chat` 受认证保护，Controller 从当前认证用户取得 userId。
- `AiChatServiceImpl` 实现输入校验、会话上下文、追问解析、意图路由和不同处理分支。
- `KnowledgeBaseLoader` 加载 `ai/knowledge/*.md`；`KnowledgeRetriever` 做改写、过滤、检索、去重和重排。
- `AiRagConfidenceGuard` 对无命中、低分和混合来源进行置信度判断。
- `DeepSeekAiModelClient` 以可选配置调用外部模型，配置中有连接/读取超时、模型名和环境变量 Key。
- 模型失败或不可用时，服务可基于知识片段兜底，或返回明确的不可用状态；回答可携带 `AiCitationDTO`。
- `AiConversationContextService`、`AiChatLogRecorder` 和 `AiChatExecutionTrace` 支持会话上下文、引用、fallback、耗时、模型和 trace_id 记录。
- `08-AI助手可回答问题清单.md`、`10-AI助手能力边界与安全策略.md` 明确只读问答、口径区分、引用和降级规则。

### [设计过]

- AI 与业务模块采用边界隔离，回答政策和流程，不把模型结果直接写入审核、额度、发券或核销状态。
- 国家政策口径与平台当前演示活动口径分开检索和回答；不确定时澄清或提示依据不足。
- 模型作为可替换的外部依赖，失败时回退到可靠知识片段或固定安全模板。
- 通过返回 route、answer mode、引用和 fallback 状态，支持前端解释“这次回答是怎么来的”。
- 未来可以补充流式输出、评测集、异步任务或更强的模型供应商适配，但当前资料不能把这些说成已完成。

### [学习过]

- `langchain4j.md` 中的 `ChatModel`、`AI Service`、`ChatMemory`、Structured Output、RAG、Tool Calling、MCP、Guardrail 和 SSE 示例。
- [[AI应用工程面试精炼笔记/01-LLM-API与HTTP可靠调用]] 中的外部 API 可靠性。
- [[AI应用工程面试精炼笔记/02-Context-Memory与Structured-Output]] 中的会话隔离和数据契约。
- [[AI应用工程面试精炼笔记/03-RAG核心链路]] 中的索引、检索、重排、引用和降级。

## 项目为什么选择只读 AI

```text
模型可能产生幻觉或误解用户意图
→ 资金、资格、审核和核销属于高风险事实
→ 不能让自由文本直接触发写操作
→ AI 只做解释和导航
→ 最终结果由规则、事务、数据库约束和人工审核裁决
```

这与 [[SR/SmartRenew面试精炼笔记/00-SmartRenew项目知识地图]] 的业务主链一致：AI 是辅助入口，不是业务事实来源。

## 高频面试问答

### 1. 为什么 SmartRenew 要用 RAG？

政策、活动规则、材料要求和状态解释属于项目知识。RAG 让回答先参考版本化知识片段，降低只靠模型记忆造成的幻觉，并返回引用；但最终政策和业务结果仍以官方口径、活动配置和系统事实为准。

### 2. 为什么不让大模型直接审核或扣额度？

模型输出不是可直接信任的业务事实，且审核、额度、发券、核销需要权限、事务、状态机、唯一约束和并发控制。AI 只读解释能把不确定性隔离在展示层。

### 3. 模型服务不可用时怎么办？

先区分是未启用、Key 缺失、超时、HTTP 错误还是解析失败；可重试的短暂错误有限重试，其他错误不重试。若已有高置信度知识片段，返回带兜底标记的简短解释；没有可靠证据则明确不可用并引导用户补充。

### 4. 如何避免国家政策和平台活动规则混答？

识别用户意图和口径，给知识片段增加 scope/source metadata，检索时优先过滤对应范围；回答中明确“国家政策口径”或“平台演示口径”，不确定时先澄清。

### 5. 会话追问如何实现？

用 userId 与 conversationId 读取有限历史，先把“那上限呢”改写为独立问题，再进入同一意图的检索和回答。历史不能无限拼接，也不能覆盖当前权限和业务事实。

### 6. 如何知道回答是否可信？

看检索是否命中、最高分和来源是否一致、引用是否来自真实 chunk，以及最终是否触发低置信度保护。模型回答本身不能作为唯一可信度指标。

### 7. SmartRenew 是否使用 LangChain4j 的 AI Service、Tool Calling 或 MCP？

现有项目源码可核对到自有 `AiModelClient`、RAG、路由和降级链路；`langchain4j.md` 是学习资料，不能据此宣称项目使用了 LangChain4j AI Service、Tool Calling 或 MCP。它们目前属于学习过或后续入口。

### 8. 如何保护 AI 接口？

入口需要认证和资源边界；Key、JWT、数据库密码不进入日志；限制输入长度和频率；不把用户输入当作系统指令；输出不泄露内部路径、SQL、敏感字段；异常对外给安全信息，详细根因留在受控日志。

## 一分钟项目回答

> SmartRenew 的 AI 模块定位是只读的政策和流程问答。请求先经过认证和输入校验，再按用户与会话读取有限上下文，处理追问并做意图路由；涉及平台规则、材料和状态的问题进入本地知识库检索，模型服务可配置启用，模型不可用时用可靠知识片段兜底。返回结果会带引用、route 和 fallback 状态，日志记录 trace_id、耗时和模型结果。我们明确不让 AI 参与审核、额度扣减、发券或核销，最终业务事实由规则、事务和数据库约束保证。

## Reference

- [[SR/SmartRenew面试精炼笔记/00-SmartRenew项目知识地图]]
- [[AI应用工程面试精炼笔记/01-LLM-API与HTTP可靠调用]]
- [[AI应用工程面试精炼笔记/02-Context-Memory与Structured-Output]]
- [[AI应用工程面试精炼笔记/03-RAG核心链路]]
