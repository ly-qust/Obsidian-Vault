---
tags: [Vault审计, 删除候选]
priority: P2
status: review
last_review: 2026-08-28
---

# DELETE-CANDIDATES

本表只记录候选，未经确认不删除。入链指 Markdown 中指向该笔记的 Wiki 链接；图片型文件还必须检查对应图片在 `.canvas`、`.base` 和其他 Obsidian 资源中的引用。

## 低风险候选

| 文件 | 大小 | 入链 | 当前正文 | 替代 / 处理 | 删除风险 |
|---|---:|---:|---|---|---|
| 面向对象编程.md | 0 B | 0 | 空文件 | Java 主入口 | 低 |
| outbox模式.md | 0 B | 0 | 空文件 | [[SR/SmartRenew面试精炼笔记/03-申请提交与Outbox-Inbox]] | 低 |
| spring/spring data jpa.md | 0 B | 0 | 空文件 | 暂无 | 低 |
| 调bug能力/1.超卖问题.md | 0 B | 0 | 空文件 | [[redis/Redis面试精炼笔记/01-缓存与分布式锁]] | 低 |

## 必须先检查图片或外部资源

| 文件 | 大小 | 入链 | 当前正文 | 替代 / 处理 | 删除风险 |
|---|---:|---:|---|---|---|
| ai应用/01大模型部署.md | 36 B | 0 | 1 张图片 | AI 工程主线；先查看图片 | 高 |
| Linux/docker.md | 110 B | 0 | 3 张图片 | [[docker]]；先查看图片 | 中 |
| Linux/目录结构.md | 73 B | 0 | 2 张图片 | Linux 主线；先查看图片 | 高 |
| Linux/防火墙操作.md | 36 B | 0 | 1 张图片 | Linux 排障；先查看图片 | 高 |
| java笔记/四种访问修饰符.md | 36 B | 0 | 1 张图片 | Java Reference；先查看图片 | 高 |
| java笔记/idea快捷键.md | 36 B | 0 | 1 张图片 | Java Reference；先查看图片 | 高 |
| javaweb/初始web.md | 36 B | 0 | 1 张图片 | JavaWeb Reference；先查看图片 | 高 |
| javaweb/开发规范.md | 36 B | 0 | 1 张图片 | JavaWeb Reference；先查看图片 | 高 |
| javaweb/mybatis查询.md | 37 B | 0 | 1 张图片 | MyBatis Reference；先查看图片 | 高 |
| 监控/Prometheus+Grafana.md | 73 B | 0 | 2 张图片 | 可观测性基础；先查看图片 | 高 |
| 行测/立体图形.md | 36 B | 0 | 1 张图片 | 央国企行测材料；先查看图片 | 高 |
| 行测/数量关系.md | 36 B | 0 | 1 张图片 | 央国企行测材料；先查看图片 | 高 |
| 行测/资料分析.md | 36 B | 0 | 1 张图片 | 央国企行测材料；先查看图片 | 高 |

## 其他待确认

| 文件 | 大小 | 入链 | 判断 | 删除风险 |
|---|---:|---:|---|---|
| 创建链接.md | 0 B | 1（[[欢迎]]） | 空文件，但当前仍被欢迎页引用 | 中 |
| pyAI应用/day4.md | 99 B | 0 | 只有外部实验输出目录指针 | 中 |

## 图片清理单独规则

当前发现 233 张图片、16 张未被 Markdown Wiki 嵌入、4 组完全重复图片。未被 Markdown 引用不等于可删除；还需检查 `.canvas`、`.base`、图片同名引用和人工需要保留的截图。
