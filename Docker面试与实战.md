---
tags: [Docker, Dockerfile, Compose, SmartRenew, 面试]
priority: P0
status: learning
last_review: 2026-08-29
---

# Docker 面试与实战

## 一句话结论

Docker 用镜像交付一致的运行环境，用容器隔离进程与文件系统；面试排障要沿着进程、配置、端口、挂载、网络和依赖逐层定位。

## 1. Image 与 Container

| 概念 | 核心理解 |
|---|---|
| Image（镜像） | 只读模板，包含应用运行需要的文件、依赖和默认配置，可用于创建多个容器 |
| Container（容器） | 镜像的一次运行实例，具有隔离的进程和文件系统视图，并增加可写层 |

镜像不是“正在运行的服务”；容器也不是完整虚拟机。容器共享宿主机内核，但对进程、网络、文件系统等资源做隔离。

## 2. Dockerfile 五个高频指令

```dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY app.jar app.jar
RUN addgroup --system app && adduser --system --ingroup app app
CMD ["java", "-jar", "app.jar"]
```

| 指令 | 作用 |
|---|---|
| `FROM` | 指定基础镜像 |
| `WORKDIR` | 设置后续指令和容器启动后的默认工作目录 |
| `COPY` | 将构建上下文中的文件复制到镜像 |
| `RUN` | **构建镜像时**执行，结果进入镜像层，如安装依赖、创建用户 |
| `CMD` | **容器启动时**的默认命令，可被运行参数覆盖 |

面试重点是分清 `RUN` 与 `CMD` 的执行阶段，而不是背全部指令。

## 3. Port、Volume、Network、Environment Variable

- **Port**：`8080:8080` 表示宿主机端口映射到容器端口。容器内服务仍要监听正确地址和端口。
- **Volume**：把持久数据或配置放到容器可写层之外，避免重建容器后数据随容器消失；挂载错误也可能覆盖镜像内原有目录。
- **Network**：同一 Compose 网络中的服务通常可用服务名互相访问，如后端连接 `mysql:3306`，不能把容器内的 `localhost` 误当成其他容器。
- **Environment Variable**：把环境相关配置注入容器。密钥不能硬编码到镜像、Compose 文件、Git 或日志中，生产配置应使用受控环境变量或密钥管理方案。

## 4. Docker Compose

Compose 用 YAML 描述一组相关服务、网络、卷、依赖和健康检查，适合让多容器环境可重复启动。常用操作：

```bash
docker compose up -d
docker compose ps
docker compose logs -f backend
docker compose down
```

`depends_on` 可以表达启动依赖；若结合 `condition: service_healthy`，依赖服务需要先通过健康检查。但健康检查只证明检查命令定义的条件成立，不等于全部业务功能正常。

## 5. 四个排障命令

| 命令 | 用途 |
|---|---|
| `docker ps -a` | 看全部容器、退出状态和重启情况 |
| `docker logs <container>` | 看容器标准输出和错误日志 |
| `docker exec -it <container> sh` | 进入运行中容器检查文件、进程和网络；镜像不一定带 bash |
| `docker inspect <container>` | 看启动配置、网络、挂载、环境变量和健康状态；分享输出前先脱敏 |

## 6. Healthcheck

Healthcheck 用命令周期性判断容器是否达到预期状态，使 Compose 或编排工具能区分“进程存在”和“服务可用”。检查应轻量、稳定，并设置合理的间隔、超时、重试和启动等待期。它用于发现状态，不自动修复业务根因。

## 7. 容器启动失败排查

```text
docker ps -a 看退出码和状态
→ docker logs 找第一个根因
→ 检查环境变量和密钥是否齐全
→ 检查宿主机端口冲突与容器内监听
→ 检查 Volume 路径、权限和是否覆盖文件
→ 检查 Network、服务名和 DNS
→ 检查 MySQL、Redis、RabbitMQ 等依赖健康状态
→ 必要时 inspect 核对最终配置
```

## 8. SmartRenew 项目映射（证据边界）

当前 SmartRenew 项目仓库存在可核对的 `docker-compose.yml`，但它是本地开发依赖编排，不能扩大为完整生产部署或 Kubernetes 经验。

- **[学习过]** Docker 的镜像、容器、Compose、健康检查和启动排障方法。
- **[已实现]** `docker-compose.yml` 可核对到 MySQL、Redis、RabbitMQ 三个本地开发依赖，其中 MySQL 使用 Volume 保存数据并配置了健康检查。
- **[已实现]** 后端的生产配置支持通过环境变量注入数据库、Redis、RabbitMQ、JWT 和上传目录等配置；这说明配置边界已预留，但不等于已经完成生产容器化交付。
- **[设计过/待核验]** 如果继续容器化后端和前端，可按依赖组织服务，用服务名通信、用 Volume 保存持久数据，并补充后端健康检查与启动依赖。

面试时可以准确表达 Docker/Compose 的学习和设计能力，不把未找到证据的生产编排扩大为已实现经验，也不扩大为 Kubernetes 或复杂微服务经验。

## 9. 60 秒面试回答

> 镜像是只读构建产物，容器是镜像的运行实例。Dockerfile 中 RUN 在构建期执行，CMD 是容器启动默认命令。应用容器化时我会处理端口、Volume、网络和环境变量；多服务用 Compose 编排，并用健康检查区分进程启动和服务可用。SmartRenew 中已经有本地开发依赖的 Compose，能证明 MySQL、Redis、RabbitMQ 的依赖编排和 MySQL 数据卷；完整后端、前端生产部署仍不应在没有对应文件和运行记录时过度宣称。启动失败时按 ps -a、logs、环境变量、端口、挂载、网络和依赖服务逐层检查。

## Reference

- [[docker]]
- [[Linux/docker]]
- [[Linux/Linux面试与排障]]
