---
tags: [Linux, 命令, 速查, 运维]
priority: P1
status: reference
last_review: 2026-08-29
---

# Linux 命令速查

> 只保留秋招、部署和基础排障高频命令。排障场景见 [[Linux/Linux面试与排障]]。

## 文件与目录

| 命令 | 高频用法 |
|---|---|
| `pwd` | 显示当前目录 |
| `ls` | `ls -alh` 查看隐藏文件、权限和可读大小 |
| `cd` | `cd /path`；`cd ..`；`cd -` |
| `mkdir` | `mkdir -p app/logs` |
| `cp` | `cp file backup/`；目录使用 `cp -r` |
| `mv` | 移动或重命名 |
| `rm` | 删除文件；递归和强制参数使用前确认路径 |

## 查看文件与日志

| 命令 | 高频用法 |
|---|---|
| `cat` | 查看较短文本 |
| `less` | 分页查看大文件；`/关键词` 搜索 |
| `tail` | `tail -n 100 app.log`；`tail -f app.log` |

## 查找

| 命令 | 高频用法 |
|---|---|
| `find` | `find /var/log -name "*.log"`；`find / -type f -size +500M` |
| `grep` | `grep -n "ERROR" app.log`；`grep -A 3 -B 3 "ERROR" app.log` |

## 进程与服务

| 命令 | 高频用法 |
|---|---|
| `ps` | `ps -ef | grep java` |
| `top` | 查看 CPU、内存和高占用进程 |
| `kill` | `kill PID`；仅在必要时用 `kill -9 PID` |
| `systemctl` | `status/start/stop/restart service` |
| `journalctl` | `journalctl -u service -n 100 --no-pager` |

## 内存与磁盘

| 命令 | 高频用法 |
|---|---|
| `free` | `free -h` 查看系统内存 |
| `df` | `df -h` 看空间；`df -i` 看 inode |
| `du` | `du -sh *`；`du -xh --max-depth=1 /var` |

## 网络

| 命令 | 高频用法 |
|---|---|
| `ip` | `ip addr`；`ip route` |
| `ss` | `ss -lntp` 看 TCP 监听；`ss -antp` 看连接 |
| `ping` | 基础 ICMP 连通性测试，不代表端口和应用一定可用 |
| `curl` | `curl -v http://127.0.0.1:8080/health` |

## 权限

| 命令 | 高频用法 |
|---|---|
| `chmod` | 修改权限，如 `chmod 750 run.sh`；避免习惯性 777 |
| `chown` | 修改所有者和组，如 `chown -R app:app /opt/app` |

## 打包与传输

| 命令 | 高频用法 |
|---|---|
| `tar` | `tar -zcvf app.tar.gz app/`；`tar -zxvf app.tar.gz -C /opt` |
| `scp` | `scp app.jar user@host:/opt/app/` |
| `rsync` | `rsync -av src/ user@host:/backup/` |

## 一条排障记忆链

```text
systemctl → ps → ss → curl → journalctl/tail/grep → 依赖 → 防火墙/Nginx/网络
```

完整参数和低频命令回查 [[Linux/命令操作]]、[[Linux/未命名]]。

