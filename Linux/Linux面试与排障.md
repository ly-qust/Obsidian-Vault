---
tags: [Linux, 故障排查, Java, 运维, 面试]
priority: P0
status: learning
last_review: 2026-08-29
---

# Linux 面试与排障

> 核心不是背命令全集，而是形成“现象 → 分层检查 → 证据 → 结论”的排障链。命令查询见 [[Linux/Linux命令速查]]。

## 1. Java 服务访问不了

```text
确认服务是否部署、服务名和访问地址
→ systemctl status 查看服务状态
→ ps 确认进程是否存在
→ ss 确认目标端口是否监听
→ curl 本机地址，区分应用与外部入口问题
→ 应用日志 / journalctl 查看启动和请求错误
→ 检查 MySQL、Redis、RabbitMQ 等依赖
→ 检查防火墙、Nginx、地址与路由
```

常用证据：

```bash
systemctl status smart-renew
ps -ef | grep java
ss -lntp | grep 8080
curl -v http://127.0.0.1:8080/health
journalctl -u smart-renew -n 100 --no-pager
```

本机 `curl` 成功只证明本机到应用这一段可用，不能证明 Nginx、防火墙、DNS 和外部网络正常。

## 2. Java 服务启动失败

按以下顺序定位：

1. `systemctl status` 或 `ps`：服务是否反复退出，是否残留旧进程。
2. 配置：配置文件路径、环境变量、Profile、数据库和中间件地址是否正确。
3. 端口：用 `ss -lntp` 检查端口冲突。
4. JDK：`java -version`，确认运行版本与构建要求一致。
5. 权限：运行用户是否能读配置、写日志并访问工作目录。
6. 依赖：MySQL、Redis、RabbitMQ 等是否可连接。
7. 日志：优先看首个根因异常，不要只看最后一行“启动失败”。

## 3. CPU 高

```text
top 观察总体负载和高占用进程
→ 定位 PID
→ ps 确认进程命令和运行用户
→ Java 场景继续结合线程、JVM 指标和同一时间段日志
```

CPU 高只是现象，可能来自业务流量、死循环、频繁 GC、线程忙等。JVM 深入排查见 [[java八股文/Java后端面试精炼笔记/03-JVM内存-GC与排查]]。

## 4. 内存高

- `free -h` 看系统整体内存与可用内存。
- `top` 看哪个进程占用较高及变化趋势。
- Linux 页面缓存会利用空闲内存，不能只因 `free` 很小就断定内存泄漏。
- Linux 进程内存不等于 Java Heap；Java 还可能涉及元空间、线程栈、直接内存等。

需要进入 JVM 层时转到 [[java八股文/Java后端面试精炼笔记/03-JVM内存-GC与排查]]。

## 5. 磁盘满

```text
df -h 查看文件系统空间
→ df -i 查看 inode 是否耗尽
→ du -xh --max-depth=1 定位大目录
→ find 查找大文件或异常日志
→ 确认后再清理、归档或调整日志策略
```

- 磁盘空间满：数据块不足。
- inode 满：即使还有容量，也可能无法创建新文件，常见于大量小文件。
- 不要未确认文件用途就直接删除日志或数据文件。

## 6. 端口不通

```text
ss 确认服务是否监听、监听在哪个地址
→ curl 访问 127.0.0.1:端口
→ curl 访问本机实际 IP:端口
→ 检查 firewalld / 安全组
→ ip route 检查路由
→ 再检查 Nginx 或上游网络
```

`0.0.0.0:8080` 通常表示监听所有 IPv4 地址；`127.0.0.1:8080` 只允许本机路径访问。完整网络分层见 [[计算机网络面试精炼笔记/07-网络故障排查]]。

## 7. 权限错误

```bash
ls -l path
chmod 640 config.yml
chown app:app path
```

先确认运行用户、所有者、用户组及目录逐级权限，再按最小权限调整。不要把 `chmod 777` 当成通用修复，它会扩大读写执行权限并掩盖真正的用户和目录配置问题。

## 8. 日志定位

| 目的 | 命令 |
|---|---|
| 实时看新增日志 | `tail -f app.log` |
| 分页检索大日志 | `less app.log`，用 `/ERROR` 搜索 |
| 按关键词和上下文定位 | `grep -n -A 3 -B 3 "ERROR" app.log` |
| 查看 systemd 服务日志 | `journalctl -u smart-renew --since today` |

先记录故障时间、请求路径、用户或 traceId，再筛选对应时间窗口；注意日志中的密钥、Token、个人信息应脱敏。

## 9. 网络不通

```text
ip addr 确认地址与网卡状态
→ ip route 确认默认路由和目标路由
→ ping 作为基础连通性证据之一
→ ss 确认监听和连接
→ curl 验证 TCP/HTTP 路径与响应
→ 检查防火墙、DNS、Nginx 和对端服务
```

`ping` 失败可能只是 ICMP 被禁，成功也不代表目标 TCP 端口、TLS 和业务接口正常。完整排查链见 [[计算机网络面试精炼笔记/07-网络故障排查]]。

## 10. 60 秒面试回答

> Linux 故障排查我会先确认现象、时间和影响范围，再按服务、进程、端口、本机请求、日志、依赖和网络入口逐层缩小范围。比如 Java 服务访问不了，会先看 systemctl 和进程，再用 ss 看监听、curl 验证本机接口，随后查 journalctl 和应用日志，并检查数据库、中间件、Nginx、防火墙与路由。CPU、内存和磁盘问题也先定位系统层证据，再进入 JVM 或应用层，不把单个命令结果直接当成根因。

## Reference

- [[Linux/命令操作]]
- [[Linux/未命名]]
- [[Linux/目录结构]]
- [[Linux/防火墙操作]]

