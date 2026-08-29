---
tags: [Vault审计, 删除候选]
priority: P2
status: review
last_review: 2026-08-29
---

# DELETE-CANDIDATES

本表记录 Batch 8 的安全清理结果。删除只采用可验证的窄条件：Markdown 必须为 0 字节、无 Wiki 入链、无图片或其他资源引用且已有明确替代；图片只删除完全相同 SHA 的未引用副本，并保留一个已核验版本。

## 已实际删除的 Markdown

| 文件 | 删除依据 | 替代内容 | 风险检查结果 |
|---|---|---|---|
| 面向对象编程.md | 0 B、无入链、无资源 | Java 主入口及 Java 面向对象基础 Reference | 全库 Wiki 入链为 0 |
| outbox模式.md | 0 B、无入链、无资源 | [[SR/SmartRenew面试精炼笔记/03-申请提交与Outbox-Inbox]] | Outbox/Inbox 主线已存在 |
| 调bug能力/1.超卖问题.md | 0 B、无入链、无资源 | [[redis/Redis面试精炼笔记/01-缓存与分布式锁]] | Redis 并发与超卖主线已存在 |

## 已实际删除的图片副本

以下文件均为完全相同 SHA 的未引用副本；对应重复组保留了至少一个版本，且已检查 Markdown Wiki Embed、标准 Markdown 图片、`.base`、`.canvas` 和其他文本资源引用。

| 删除文件 | 保留的同 SHA 版本 |
|---|---|
| 图片/Pasted image 20260121101130.png | 图片/Pasted image 20260121101134.png |
| 图片/Pasted image 20260810090405.png | 图片/Pasted image 20260810090410.png |
| 图片/Pasted image 20251127220933.png | 图片/Pasted image 20251127220438.png |
| 图片/Pasted image 20260810092024.png | 图片/Pasted image 20260810092055.png |
| 图片/Pasted image 20260810092027.png | 图片/Pasted image 20260810092055.png |
| 图片/Pasted image 20260810092033.png | 图片/Pasted image 20260810092055.png |
| 图片/Pasted image 20260810092043.png | 图片/Pasted image 20260810092055.png |
| 图片/Pasted image 20260810092052.png | 图片/Pasted image 20260810092055.png |

## 最终保留的候选

| 文件 | 大小 | 入链 | 当前正文 | 替代 / 处理 | 删除风险 |
|---|---:|---:|---|---|---|
| spring/spring data jpa.md | 0 B | 0 | 空文件 | 暂无 | 低 |
| 创建链接.md | 0 B | 1（[[欢迎]]） | 空文件，但当前仍被欢迎页引用 | 保留 | 中 |
| pyAI应用/day4.md | 99 B | 0 | 只有外部实验输出目录指针 | 保留，非空且暂无替代 | 中 |

## 图片型小笔记（已检查，全部保留）

| 文件 | 大小 | 入链 | 当前正文 | 替代 / 处理 | 删除风险 |
|---|---:|---:|---|---|---|
| ai应用/01大模型部署.md | 227 B | 0 | 1 张图片 | 本地部署、开放 API、云服务平台对比图；保留 | 高 |
| Linux/docker.md | 254 B | 0 | 3 张图片 | 镜像/容器、Volume、Compose 结构图；保留 | 中 |
| Linux/目录结构.md | 73 B | 0 | 2 张图片 | Linux 目录职责与路径对比图；保留 | 高 |
| Linux/防火墙操作.md | 36 B | 0 | 1 张图片 | firewalld 状态、端口和 reload 操作图；保留 | 高 |
| java笔记/四种访问修饰符.md | 36 B | 0 | 1 张图片 | 四种访问修饰符可见性矩阵图；保留 | 高 |
| java笔记/idea快捷键.md | 36 B | 0 | 1 张图片 | IDEA 高频快捷键截图；保留 | 高 |
| javaweb/初始web.md | 36 B | 0 | 1 张图片 | B/S、C/S、静态和动态资源关系图；保留 | 高 |
| javaweb/开发规范.md | 36 B | 0 | 1 张图片 | REST URL 与 HTTP 方法对照图；保留 | 高 |
| javaweb/mybatis查询.md | 37 B | 0 | 1 张图片 | `@Param` 使用场景图；保留 | 高 |
| 监控/Prometheus+Grafana.md | 203 B | 0 | 2 张图片 | Prometheus/Grafana 监控设计与 AI 指标维度图；保留 | 高 |
| 行测/立体图形.md | 36 B | 0 | 1 张图片 | 行测立体图形解题材料图；保留 | 高 |
| 行测/数量关系.md | 36 B | 0 | 1 张图片 | 行测数量关系解题材料图；保留 | 高 |
| 行测/资料分析.md | 36 B | 0 | 1 张图片 | 行测资料分析公式材料图；保留 | 高 |

## 图片清理单独规则

Batch 8 前发现 233 张图片、16 张未被 Markdown Wiki 嵌入、4 组完全重复图片。删除 8 个重复且无引用副本后当前为 225 张；普通“未被 Markdown 引用”的图片不等于可删除，其他图片型候选均因包含独有学习材料而保留。`.base` 文件 3 个、`.canvas` 文件 0 个，删除目标未被这些资源或其他文本文件引用。
