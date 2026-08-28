---
tags: [计算机网络, HTTP, HTTPS, TLS, 面试]
priority: P0
status: learning
last_review: 2026-08-28
---

# HTTP、HTTPS 与 TLS

> 当前面试主入口：[[计算机网络面试精炼笔记/00-计算机网络知识地图]]

## 一句话结论

HTTP 定义请求/响应语义，HTTPS 是 HTTP 运行在 TLS 保护之上；TLS 先认证和协商密钥，再用对称加密保护后续数据。

## 1. HTTP 面试主干

- 请求通常包含方法、目标、版本、Header 和可选 Body；响应包含状态码、Header 和可选 Body。
- 常见方法：GET 获取、POST 提交、PUT 整体更新、PATCH 部分更新、DELETE 删除；是否安全、幂等要结合语义回答。
- 状态码先按类别记：2xx 成功、3xx 重定向、4xx 客户端请求问题、5xx 服务端或网关处理问题。
- Header 承载认证、内容类型、缓存、压缩、追踪和连接协商等元数据；不要把 Header 当作业务 Body。
- HTTP/1.1 常见连接复用与 Host；HTTP/2 在同一连接上复用多路流并进行头部压缩，但不等于所有业务延迟自动降低。

## 2. HTTPS/TLS 因果链

```text
客户端发起 TLS ClientHello
→ 双方协商版本、密码套件等参数
→ 服务端发送证书和密钥交换材料
→ 客户端校验证书链、域名和有效期
→ 双方通过握手建立会话密钥
→ 后续 HTTP 数据使用对称加密并校验完整性
```

TLS 的核心价值是机密性、完整性和服务端身份认证。非对称密码适合握手/密钥协商，对称密码适合大量业务数据；不要说 HTTPS 只靠“加密”而没有证书认证。

## 3. 高频面试回答

> HTTP 负责资源和请求响应语义，HTTPS 则在 HTTP 与 TCP 之间加入 TLS。TLS 握手阶段协商参数、验证服务端证书并建立会话密钥，业务数据阶段使用对称加密保护机密性和完整性。排查 HTTPS 要把 DNS、TCP 建连、TLS 握手、HTTP 状态码和后端业务分开看。

## 4. Java / Linux / SmartRenew 关联

- Java 后端常通过 HTTP Client、框架客户端或网关调用服务，重点是超时、状态码、连接复用、认证 Header 和响应体边界，不在本篇复制 API 代码。
- AI 应用对外部模型 API 调用尤其要设置连接/读取/总超时，调用 `raise_for_status`、保留异常链并记录 trace_id，来源见 [[pyAI应用/day6HTTP 方法、状态码、header、JSON、连接读取超时httpx同步与异步]]。
- Linux 用 `curl -v`、`curl -I` 分辨 HTTP 头、重定向和 TLS 握手，Nginx 用 `nginx -t`、访问日志、错误日志和 `journalctl` 排查，见 [[Linux/命令操作]]、[[nginx/nginx(linux)]]。
- SmartRenew 的 JWT/认证、申请接口和回调链要把 HTTP 状态、业务错误码、幂等和 trace_id 分开记录，项目入口见 [[SR/SmartRenew面试精炼笔记/02-Security与JWT请求链]]。

## 容易答错

- HTTPS 不是 UDP/TCP 的替代品，而是 HTTP + TLS 的组合。
- 证书主要解决身份认证和公钥信任，不是把所有数据都用非对称加密传输。
- HTTP 200 只代表 HTTP 层成功，不代表业务结果一定成功。
- Keep-Alive 是连接复用/保持，WebSocket 才是升级后的双向消息通道。

## 高频追问

1. TLS 为什么还要对称加密？——握手后大量数据使用对称加密更高效，非对称密码主要承担认证和密钥协商。
2. 访问 HTTPS 失败如何定位？——按 DNS → TCP → TLS 证书/握手 → HTTP 状态 → 网关/后端日志顺序检查。
3. 4xx 和 5xx 怎么区分？——前者先查请求、认证和权限，后者先查服务、网关、依赖和服务端日志，具体状态码仍要结合接口语义。

## 提炼来源

- [[java笔记/17网络编程]]
- [[pyAI应用/day6HTTP 方法、状态码、header、JSON、连接读取超时httpx同步与异步]]
- [[Linux/命令操作]]、[[Linux/未命名]]
- [[nginx/nginx(linux)]]、[[nginx/nginx讲解]]
- [[SR/SmartRenew面试精炼笔记/02-Security与JWT请求链]]
