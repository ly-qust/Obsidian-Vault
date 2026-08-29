---
tags: [AI应用, Context, Memory, Structured Output, 面试]
priority: P0
status: learning
last_review: 2026-08-29
---

# Context、Memory 与 Structured Output

## 一句话结论

Context 是本次模型调用携带的上下文，Memory 是跨请求保存和恢复的会话状态；Structured Output 则把模型结果变成可校验的数据契约，三者都不能替代权限校验和业务规则。

## 因果链

```text
conversationId + userId
→ 读取最近会话
→ 截断 / 摘要 / 过滤无关历史
→ 拼接当前问题、系统规则和必要检索证据
→ 调用模型
→ 按 schema / DTO 解析校验
→ 失败时重试一次或返回安全降级
→ 保存本轮结果与可追踪元数据
```

## Context 与 Memory 怎么区分

| 概念 | 面试理解 | 主要风险 |
|---|---|---|
| Context | 当前一次请求发给模型的系统消息、历史消息、用户问题、检索片段 | 过长、无关、敏感信息污染 |
| Memory | 以用户/会话为边界持久化或缓存的历史状态 | 串会话、无限增长、过期和隐私 |
| RAG 引用 | 当前问题相关的外部事实片段 | 过期、错召回、来源不可信 |
| Structured Output | 结果的机器可读格式和字段约束 | 字段缺失、类型错误、模型只“伪装”成 JSON |

## Memory 的工程边界

1. 用 `userId + conversationId` 作为隔离边界，不能只用一个全局内存列表。
2. 只携带最近或相关历史；上下文窗口有限，历史越长不代表答案越好。
3. 对历史做截断、摘要或按主题过滤，并保留当前问题的必要指代关系。
4. 记忆存储要考虑 TTL、删除、隐私和敏感字段脱敏；“记住”不是无限保存。
5. 业务权限和事实以数据库、当前请求和知识库为准，历史对话不能绕过权限。
6. 并发请求同一会话时要考虑写入顺序、重复提交和版本冲突。

## Structured Output 的正确链路

```text
定义字段和类型
→ 通过 schema / JSON mode / prompt 约束模型
→ 收到响应
→ 解析 JSON
→ 校验必填字段、类型、枚举和范围
→ 不通过则记录原因并降级，不能直接写入业务事实
```

### 从 langchain4j 截图提炼的可搜索概念

- Structured Output：把文本输出转换为 JSON、对象或对象列表。
- 三种约束强度：JSON Schema、Prompt + JSON Mode、仅靠 Prompt；约束越弱，应用校验责任越大。
- Schema 应写出字段名、字符串/整数/数字/布尔类型和 required 字段。
- “返回了 JSON 文本”不等于“得到合法业务对象”，仍要反序列化和业务校验。
- 截图示例用 `ResponseFormat`、`JsonSchema` 和 `ChatRequest` 描述输出格式；这是框架示例，不代表 SmartRenew 已使用该 SDK。
- 流式 Token 输出适合逐字展示，通常不适合作为需要完整 schema 校验的最终结构化结果。

## Java / SmartRenew 关联

- SmartRenew 请求模型本身使用 `AiChatRequest`，响应使用 `AiChatResponse` 和 `AiCitationDTO`，这是服务边界上的 JSON DTO，不应说成“模型已经完全结构化输出”。
- `AiConversationContextService` 负责读取会话轮次、构造对话状态；`AiChatServiceImpl` 只携带有限历史并处理追问解析。
- `AiChatLogRecorder` 记录会话、引用、fallback、状态、模型和 trace_id，说明“记忆/审计”和“给模型的上下文”是两个概念。
- 当前项目的结构化输出重点是 HTTP DTO、引用列表和内部 trace；模型自由文本仍由 `AiBusinessAnswerComposer` 组织和兜底。

## 容易答错

- “Memory 就是把所有聊天记录拼到 Prompt”：会导致上下文超长、串话和隐私问题。
- “有了 JSON Schema 就不会幻觉”：Schema 只约束形状，不保证事实正确。
- “Structured Output 可以直接写数据库”：仍要做业务校验、权限校验和状态机校验。
- “conversationId 足够隔离用户”：必须和认证得到的 userId 绑定，不能让客户端任意读取其他人的会话。
- “最近消息越多越好”：应保留对当前问题有用的历史，并控制 token 预算。

## 高频追问

1. 多用户聊天如何避免记忆串线？
2. 会话历史太长怎么处理？
3. JSON 解析成功但字段不合法怎么办？
4. 为什么模型结构化输出不能替代数据库约束？
5. 对话历史、RAG 引用和业务事实分别应该由谁负责？

## Reference

- [[langchain4j]]
- [[SR/SmartRenew面试精炼笔记/00-SmartRenew项目知识地图]]
- [[AI应用工程面试精炼笔记/03-RAG核心链路]]
