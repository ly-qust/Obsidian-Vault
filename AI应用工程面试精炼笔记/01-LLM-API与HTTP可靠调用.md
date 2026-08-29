---
tags: [AI应用, LLM, HTTP, 可靠性, 面试]
priority: P0
status: learning
last_review: 2026-08-29
---

# LLM API 与 HTTP 可靠调用

## 一句话结论

调用大模型不是“调用一个本地函数”，而是一次可能超时、返回错误、格式不稳定且需要保护密钥的外部 HTTP 请求；可靠性必须由应用层补齐。

## 因果链

```text
用户请求
→ 鉴权 / 参数校验
→ 读取受控配置
→ 设置连接与读取超时
→ 发起 HTTP 请求
→ 检查状态码
→ 解析并校验响应
→ 按错误类型重试或降级
→ 记录 trace_id、耗时、状态和模型信息
```

## 面试回答

我会把模型当作外部依赖处理：API Key 放在环境变量或密钥管理中，不写入代码和日志；连接超时、读取超时都要设置；收到响应先检查 2xx，再解析 JSON；网络抖动、限流和服务暂时不可用可以有限重试，鉴权失败、参数错误等不可重试错误直接失败。对话类接口还要准备知识库或固定模板兜底，并通过 trace_id、请求耗时和模型状态定位问题。

## 高价值要点

1. **超时分层**：连接超时解决“连不上”，读取超时解决“连上但迟迟没有响应”；只设置一个默认超时往往不够清晰。
2. **状态码先于 JSON**：401/403、429、5xx 和业务错误的处理策略不同，不能直接 `response.json()` 假定请求成功。
3. **重试要分类**：超时、短暂网络错误、部分 5xx 可以有限重试；无效 Key、参数错误、余额或权限问题重试无意义。
4. **退避与上限**：重试要有次数、退避和总耗时上限，避免多个请求同时重试造成雪崩；写操作还要先确认幂等性。
5. **密钥不进日志**：日志可记录 provider、model、request_id、状态和耗时，但不要打印完整 Authorization、Prompt 中的敏感信息或响应中的隐私数据。
6. **异常要保留根因**：把底层异常转换为业务异常时保留 cause；不要用宽泛 `except Exception` 把编程错误吞掉。
7. **同步与异步匹配**：异步 Web 路由要使用异步 HTTP 客户端并 `await`；同步客户端放进事件循环会阻塞其他请求。
8. **可观测性**：trace_id 要贯穿入口、模型请求、检索和降级结果；否则只能看到“AI 失败”，无法还原链路。

## 从 langchain4j 截图提炼的可搜索概念

- `ChatModel`：负责与模型交互的基础抽象。
- `AI Service`：以接口方法描述对话能力，框架代理负责组装调用；截图中使用 `AiServices.create` / `AiServices.builder`。
- `@SystemMessage`：声明模型的系统角色、职责和边界，不应把它当作权限系统本身。
- 模型配置包含 provider、model name、API Key 和 endpoint；配置注入与密钥保护仍属于应用工程问题。
- 流式响应可以用 TokenStream 或响应流逐段传给前端，但流式输出与结构化输出的校验策略不同。

## Java / Python / SmartRenew 关联

- Java：SmartRenew 的 `AiConfiguration` 使用 Spring `RestClient`，通过 request factory 配置 connect timeout 和 read timeout；`DeepSeekAiModelClient` 检查启用开关与 API Key，再把模型异常交给上层。
- SmartRenew 配置：模型地址、模型名、Key 和超时通过 `smartrenew.ai.deepseek` 配置，生产配置使用环境变量占位符。
- SmartRenew 处理：模型不可用时，服务根据检索结果走知识库兜底或明确不可用，而不是把异常直接暴露给用户。
- Python：[[pyAI应用/day6HTTP 方法、状态码、header、JSON、连接读取超时httpx同步与异步]] 已整理 `httpx` 超时、`raise_for_status()`、AsyncClient、JSON 降级和 trace_id。

## 容易答错

- “设置 timeout 就一定不会阻塞”：还要考虑连接池、线程/事件循环、重试总预算和上游排队。
- “HTTP 200 就一定成功”：业务响应仍可能包含错误状态或不可解析内容。
- “任何失败都重试”：鉴权、参数和配额错误重试只会放大问题。
- “模型 API Key 放在 application.yml 方便管理”：配置文件可能被提交、备份或打印，敏感值应外置并脱敏。
- “流式输出更可靠”：流式只是交互方式，仍要处理断连、半包、结束标记和前端重连。

## 高频追问

1. 读取超时与连接超时有什么区别？
2. 429 和 500 是否都应该重试？重试上限如何定？
3. 如何避免模型请求失败时用户看到 500？
4. 如何在日志里定位某次模型调用但不泄露 Prompt 和 Key？
5. 为什么异步接口不能直接调用同步 HTTP 客户端？

## Reference

- [[langchain4j]]
- [[pyAI应用/day6HTTP 方法、状态码、header、JSON、连接读取超时httpx同步与异步]]
- [[AI应用工程面试精炼笔记/00-AI应用工程知识地图]]
