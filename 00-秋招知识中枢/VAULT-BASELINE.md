---
tags: [Vault审计, Git基线, 统计]
priority: P1
status: review
last_review: 2026-08-28
---

# Vault Batch 0 基线

## 统计口径

- 统计根目录：当前 Vault 根目录。
- 排除目录：`.git`、`.obsidian`、`.claude`、`.claudian`。
- Markdown：递归统计扩展名为 `.md` 的文件。
- 行数：读取文件正文后统一按 CRLF / LF 换行拆分；空文件计 0 行。
- 图片引用：分别统计 Markdown Wiki 嵌入，并在删除前另查 `.canvas`、`.base` 等 Obsidian 资源。

## 当前基线（2026-08-28）

| 指标 | 数值 |
|---|---:|
| 内容文件总数 | 405 |
| Markdown 文件 | 169 |
| Markdown 总行数 | 30,614 |
| 图片文件 | 233 |
| 内容目录 | 33 |
| Markdown 小文件（≤150 B） | 20 |
| Wiki 语法块（含图片嵌入） | 357 |
| 笔记 Wiki 链接 | 189 |
| 明显缺失笔记链接 | 0 |
| `.canvas` 文件 | 0 |
| `.base` 文件 | 3 |

图片当前另有 16 张未被 Markdown Wiki 嵌入、4 组完全重复；这些只进入候选，不代表可以删除。

## Git 基线

- 分支：`vault-reorg-2026-08`
- Batch 0 提交：`0eea56d docs(vault): add autumn recruitment knowledge hub`
- `.obsidian/workspace.json` 保持未提交，未加入本次基线。
- 本文件及其他 Batch 0 文件内部只使用 Wiki 相对链接。
