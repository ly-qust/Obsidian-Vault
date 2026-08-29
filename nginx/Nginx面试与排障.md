---
tags: [Nginx, 反向代理, 负载均衡, WebSocket, 故障排查, 面试]
priority: P0
status: learning
last_review: 2026-08-29
---

# Nginx 面试与排障

## 一句话结论

Nginx 是 Java 应用前的统一入口，核心价值是静态资源、反向代理、TLS 终止、基础负载均衡和入口排障；遇到错误要区分入口配置、网络连接与上游处理问题。

## 1. Nginx 在 Java 应用前做什么

- **静态资源服务器**：直接提供 HTML、CSS、JavaScript、图片等资源。
- **统一入口**：对外暴露 80/443，把 `/api/` 等请求转发给 Java 服务。
- **反向代理**：隐藏内部服务地址，并集中处理路由、Header、日志、TLS 等入口能力。
- **基础负载均衡**：将请求分发到多个上游实例。
- **入口保护**：可做基础连接、请求速率限制；复杂业务限流仍需要应用和整体架构配合。

Nginx 不是 Java 服务的替代品，也不能替代应用内部的认证、业务校验和数据一致性控制。

## 2. 正向代理与反向代理

| 类型 | 代理谁 | 客户端主要感知 |
|---|---|---|
| 正向代理 | 代理客户端访问外部目标 | 客户端知道并连接代理，目标服务看到的是代理 |
| 反向代理 | 代理服务端接收客户端请求 | 客户端访问统一入口，不必知道真实上游地址 |

## 3. `upstream`、负载均衡与 `proxy_pass`

```nginx
upstream backend_pool {
    server backend1:8080;
    server backend2:8080;
}

server {
    listen 80;

    location /api/ {
        proxy_pass http://backend_pool/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

- `upstream` 定义一组上游服务；默认可按轮询思想分发，请求分发不代表业务天然无状态或数据天然一致。
- `proxy_pass` 指定请求转发目标。其 URI 是否带尾部 `/` 会影响路径拼接，修改后要用实际请求验证。
- `Host` 保留客户端访问的主机信息，便于虚拟主机、业务路由和日志分析。
- `X-Real-IP` 通常传递直接客户端地址；`X-Forwarded-For` 记录经过的代理链。应用只能在可信代理边界内使用这些 Header，不能无条件相信客户端自行伪造的值。

## 4. HTTPS 与 TLS 终止

Nginx 可以在入口监听 443、配置证书并完成 TLS 握手，再把解密后的 HTTP 请求转发给内部上游，这称为在 Nginx 终止 TLS。内部链路是否继续使用 TLS取决于安全边界和部署要求。协议细节见 [[计算机网络面试精炼笔记/04-HTTP-HTTPS与TLS]]。

## 5. WebSocket 代理

WebSocket 从 HTTP 握手升级为长连接，代理时需要正确传递升级相关 Header：

```nginx
location /ws/ {
    proxy_pass http://backend_pool;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
}
```

还要结合长连接超时、上游服务状态和前端重连策略排查，不能只看到 101 就认为后续连接永远稳定。

## 6. 限流

Nginx 可在入口基于共享区域和请求速率做基础限流，保护上游免受突发流量影响。限流阈值需要基于容量和业务测试设置；它不能替代用户级、接口级业务规则，也不能单独保证高并发数据一致性。

## 7. 502 Bad Gateway

502 的核心含义是：Nginx 作为网关/代理时，没有从上游获得可作为正常代理响应处理的有效结果。常见于上游未启动、端口/地址错误、连接被拒绝、协议不匹配、上游提前断开等，但不能绝对写成“后端一定挂了”。

```text
确认 upstream 配置
→ systemctl / docker ps 看上游服务
→ ss 看上游监听地址和端口
→ 从 Nginx 所在环境 curl 上游
→ 看 Nginx error.log 和上游日志
→ 核对 proxy_pass、协议、路径和网络
```

## 8. 504 Gateway Timeout

504 通常表示 Nginx 等待上游连接或响应超过了代理超时。排查上游慢请求、下游依赖、连接池/线程池、数据库、网络以及超时配置。先找为何变慢，再决定是否调整超时；单纯调大超时可能只是延迟暴露问题。

## 9. 通用排障链

```text
systemctl status nginx
→ nginx -t
→ ss 确认 80/443 监听
→ curl 本机入口
→ 查看 access.log / error.log / journalctl
→ curl 上游服务
→ 检查 proxy_pass、Header、路径、超时和防火墙
```

系统与网络层排查见 [[Linux/Linux面试与排障]]、[[计算机网络面试精炼笔记/07-网络故障排查]]。

## 10. 60 秒面试回答

> Java 应用前常放 Nginx，用于静态资源、统一入口、反向代理、TLS 终止、基础负载均衡和入口限流。upstream 定义上游池，proxy_pass 负责转发，并通过 Host、X-Real-IP、X-Forwarded-For 传递必要请求信息。遇到 502，我会检查上游进程、监听端口、本机 curl、Nginx error log 和代理配置，因为 502 只是未获得有效上游响应，不等于后端一定宕机；504 则重点排查等待上游超时及上游处理变慢。

## Reference

- [[nginx/nginx(linux)]]
- [[nginx/nginx讲解]]
