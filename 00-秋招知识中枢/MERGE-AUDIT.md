---
tags: [Vault审计, 合并候选, Reference]
priority: P1
status: review
last_review: 2026-08-28
---

# MERGE / REFERENCE 审计

本表用于后续 Batch 逐领域处理。当前只建立关系，不移动旧文件。

## 已有成熟体系：只补遗漏

- Java：[[java八股文/Java后端面试精炼笔记/00-Java后端面试知识地图]]
- MySQL：[[MySQL八股文/MySQL面试精炼笔记/00-MySQL面试知识地图]]
- Redis：[[redis/Redis面试精炼笔记/00-Redis面试知识地图]]
- RabbitMQ：[[rabbitmq/RabbitMQ面试精炼笔记/00-RabbitMQ面试知识地图]]
- SmartRenew：[[SR/SmartRenew面试精炼笔记/00-SmartRenew项目知识地图]]

## 合并候选

| 来源 | 唯一主入口 / 目标 | 处理原则 |
|---|---|---|
| Linux/命令操作.md + Linux/未命名.md | Linux 面试与排障 + Linux 命令速查 | 拆分场景排障、命令表和独有内容；旧文暂不移动 |
| docker.md + Linux/docker.md | Docker 面试与实战 | 根目录 docker.md 为主要来源；Linux/docker.md 先审图片 |
| nginx/nginx(linux).md + nginx/nginx讲解.md | Nginx 面试与排障 | 保留运维排障和项目配置的独有内容 |
| java知识/线程池.md + java八股文/_整理前归档/2026-08-26/线程池.md | [[java八股文/Java后端面试精炼笔记/02-并发工具选型]] | 只补独有七参数、任务流、拒绝策略和下游容量边界 |
| java笔记/11异常处理.md + java笔记/java异常处理.md | Java 基础扩展 | 新增一篇高频问答，不复制课程全文 |
| java笔记/14泛型与枚举深化.md + java笔记/java泛型（Generics）.md | Java 基础扩展 | 只保留类型擦除、通配符和高频场景 |
| java笔记/15IO流.md + java笔记/java中Stream，file和IO的核心知识.md | Java 基础扩展 | 只保留 IO 选型、Stream 边界和 try-with-resources |
| java笔记/18反射.md + java笔记/java反射.md + java笔记/19注解.md + java笔记/java注解.md | Java 基础扩展 | 提炼反射、注解与 Spring/MyBatis 的联系 |
| MySQL 长专题 | [[MySQL八股文/MySQL面试精炼笔记/00-MySQL面试知识地图]] | 只补精炼版遗漏，原文保留 Reference |
| Redis 旧专题 | [[redis/Redis面试精炼笔记/00-Redis面试知识地图]] | 以主地图已有原始索引为准，先验证吸收情况 |
| langchain4j.md + pyAI应用 HTTP 长笔记 | AI 应用工程面试精炼笔记 | 第一轮只做 5 个入口，截图先文字化 |
| 监控/可观测性.md + 监控/ARMS.md + 监控/Prometheus+Grafana.md | 可观测性面试基础 | 只保留 Metrics、Logs、Trace 和基本排障 |

## 后续统一原则

1. 原始资料保留，主入口只保留标准答案、因果链、场景和边界。
2. Reference 笔记顶部可增加轻量提示，正文主体不重写。
3. 每个 Batch 完成后检查 Wiki 链接、图片引用和 `git diff`，再独立提交。
