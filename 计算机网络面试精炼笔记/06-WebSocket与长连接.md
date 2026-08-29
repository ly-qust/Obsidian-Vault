---
tags: [计算机网络, WebSocket, 长连接, HTTP, Nginx, 面试]
priority: P0
status: learning
last_review: 2026-08-28
---

# WebSocket 与长连接

> 当前面试主入口：[[计算机网络面试精炼笔记/00-计算机网络知识地图]]

## 一句话结论

WebSocket 通常先借助 HTTP Upgrade 建立连接，再在同一 TCP 连接上提供客户端与服务端双向消息通信；长连接本身只描述连接生命周期，不等于 WebSocket。

## 1. 建立与通信因果链

```text
客户端发起 HTTP Upgrade 请求
→ 服务端同意并返回 101 Switching Protocols
→ 连接升级为 WebSocket
→ 双方在同一 TCP 连接上收发帧
→ 通过 close 帧或连接异常结束
```

WebSocket 解决的是服务端主动推送和双向实时通信，不需要客户端持续轮询。连接建立后不再按普通 HTTP 请求/响应逐条理解业务消息，而是处理 WebSocket 帧和应用消息边界。

## 2. 长连接的工程边界

- **HTTP Keep-Alive**：尽量复用 TCP 连接，减少重复握手；仍是多个 HTTP 请求/响应。
- **WebSocket**：升级后保持双向消息通道，服务端可主动推送。
- **心跳**：检测连接是否仍可用，避免假连接；心跳周期要考虑代理、负载和移动网络。
- **重连**：需要退避、最大重试、连接状态和消息幂等；不能无限快速重连形成风暴。
- **背压**：发送速度超过消费速度时，要限制缓存、丢弃低价值消息或断开慢客户端。

## 3. 高频面试回答

> WebSocket 通常通过 HTTP Upgrade 完成握手，成功后复用同一 TCP 连接进行全双工通信。它适合聊天、实时状态和通知推送；Keep-Alive 只是 HTTP 连接复用。生产上还要处理心跳、超时、重连、鉴权、消息幂等、顺序和慢客户端背压。

## 4. Java / Linux / SmartRenew 关联

- Java WebSocket 框架负责连接和帧处理，应用仍要设计消息协议、鉴权、心跳与重连；已有 WebSocket 概念来源见 [[javaweb/实训java心得]]。
- Nginx 作为反向代理时要检查 Upgrade/Connection 头、超时和代理日志；Nginx 的反向代理与故障顺序见 [[nginx/nginx(linux)]]、[[nginx/nginx讲解]]。
- Linux 用 `ss -ant` 观察连接数量和状态，配合 `curl`、Nginx access/error log、`journalctl` 判断入口和后端两段链路。
- SmartRenew 如果需要预约状态、审核结果或通知推送，可评估 WebSocket；核心交易仍要靠 HTTP/MQ、持久化、幂等和补偿保证，项目入口见 [[SR/SmartRenew面试精炼笔记/00-SmartRenew项目知识地图]]。

## 容易答错

- WebSocket 不是“HTTP 长轮询”，Upgrade 成功后通信语义已经改变。
- TCP Keep-Alive、HTTP Keep-Alive 和 WebSocket 心跳是不同层次的机制。
- 长连接不代表永不超时；Nginx、网关、防火墙、客户端和服务端都可能主动断开。
- WebSocket 只提供传输通道，不自动保证消息持久化、顺序、幂等和离线补发。

## 高频追问

1. WebSocket 为什么先走 HTTP？——利用现有端口、代理和鉴权入口完成协议升级，之后使用 WebSocket 帧通信。
2. Nginx 代理 WebSocket 为什么连不上？——检查 Upgrade/Connection 转发、代理超时、端口监听、后端状态和错误日志。
3. WebSocket 断线如何设计？——心跳探活、指数退避重连、连接状态机、消息序号/幂等和必要的离线补偿。

## 提炼来源

- [[javaweb/实训java心得]]
- [[nginx/nginx(linux)]]、[[nginx/nginx讲解]]
- [[java笔记/17网络编程]]
- [[SR/SmartRenew面试精炼笔记/00-SmartRenew项目知识地图]]
