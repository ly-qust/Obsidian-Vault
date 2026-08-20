## 一、基础功能

wsl -l 查看可用版本
wsl -update 更新
wsl -d 是distribution的缩写，用来制定启动哪个linux

设置默认实例
![[Pasted image 20260715164936.png]]

## 二、安装常见的工具
python   可以用这个命令将python和python3链接起来
![[Pasted image 20260715170002.png]]


nodejs  官网选择npm下载
pi  `curl -fsSL https://pi.dev/install.sh | sh`
## 三、安装
配置pi，可以使用里面的模型，去找api来配置

![[Pasted image 20260716150607.png]]
使用ai完成的坦克大战虽然是在虚拟机里面的3000端口，但是实际上在本机访问3000端口也可以，这是因为wsl有自动转发的功能

使用git vscode等操作和windows系统几乎一模一样

explorer.exe . 表示打开当前目录，也就是windows的资源管理器

![[Pasted image 20260716152912.png]]
![[Pasted image 20260716152939.png]]
这里可以通过挂载卷来看到我windows系统上的数据


安装  hermes-agent
安装 claude code  都是一行命令就能解决的


安装docker
curl -fsSL https://get.docker.com -o install-docker.sh
sudo sh install-docker.sh
但是docker在windows发行版比较好用，只需要在windows里启用wsl功能即可
![[Pasted image 20260716191957.png]]


运行redis
sudo docker run -d -p 6379:6379 redis  启动redis数据库


## 四、wsl2
![[Pasted image 20260716195554.png]]
这张图非常清晰地展示了 **WSL2 (Windows Subsystem for Linux 2)** 的底层架构原理。

结合你刚才折腾 Docker 和 Redis 的经历，理解这张图能让你彻底明白为什么要在 Windows 上用 WSL，以及它是怎么工作的。

我们从下往上、分模块来拆解这张图：

### 1. 最底层的基础：Hypervisor 虚拟化平台
*   **位置：** 图中最下方的绿色长框。
*   **作用：** 这是微软的 Hyper-V 虚拟化技术。它直接运行在计算机的物理硬件之上，扮演着“大管家”的角色。它的任务是把 CPU、内存等硬件资源合理地分配给上面的操作系统。

### 2. 并驾齐驱的两个核心（中间层）
注意看图中的两个深蓝色框：**“Windows 内核”** 和 **“Linux 内核”**。
*   **平级关系：** 这是 WSL2 架构最核心的亮点！在这张图里，Windows 内核和 Linux 内核是**平行并列**的。它们都直接坐在 Hypervisor 之上。
*   **真正的 Linux：** 在早期的 WSL1 中，Windows 是通过“翻译”的方式假装自己是 Linux，导致很多高级软件跑不起来。而 WSL2 则是微软直接塞了一个**完整的、真正的 Linux 内核**进去。这也是为什么图底部的字幕说“这种架构提高了系统的兼容性”，因为它是原生的 Linux 环境。

### 3. 用户空间（最上层）
内核之上就是我们日常使用的软件层（用户空间）。
*   **左侧 Windows 用户空间：** 运行着你日常的 Windows 软件，比如微信、浏览器、Word 等。
*   **右侧 Linux 用户空间：** 运行着 Linux 系统的各种命令和软件（比如你刚才安装的 Redis）。
*   **🐋 小鲸鱼标志 (Docker)：** 注意看右上角的 Docker 标志放在了 Linux 用户空间里。Docker 的底层极度依赖 Linux 内核的专属特性（如 cgroups 和 namespaces）。正因为 WSL2 提供了一个真正的 Linux 内核，Docker 才能在这里极其顺畅、原生地位运行，比以前在 Windows 上硬装 Docker 体验好无数倍。

### 4. 架构中的亮点细节
*   **横向的黄色双向箭头：** 在 Windows 和 Linux 用户空间之间有一个箭头。这代表着两个系统虽然内核独立，但**高度互通**。
    *   **文件互通：** 你可以在 Linux 里直接访问 Windows 的 C 盘（就是你命令行里的 `/mnt/c/`）；在 Windows 里也能直接打开 Linux 的文件。
    *   **网络互通：** 它们共享网络（localhost）。这也是为什么你刚才在 WSL 里跑的 Redis 容器映射到 6381 端口，你在 Windows 上的 Redis Insight 软件里直接连 `127.0.0.1:6381` 就能连上的原因。
*   **右侧的显卡 (GPU) 标志：** 图中 Linux 内核旁边画了一个显卡。这代表 WSL2 支持 **GPU 硬件加速**。这意味着如果你在 Linux 里面跑人工智能、深度学习（比如用 Python 跑大模型），它是可以直接调用你电脑上那张独立显卡的算力的。

### 总结
简单来说，这个架构就是：**Windows 利用底层的虚拟化技术，在自己的系统里“养”了一个真正且完整的 Linux 系统。** 它们俩各自独立运行以保证兼容性，但又在文件和网络上无缝打通以保证开发者的使用体验。

## 五、显卡直通
安装linux系统中的一键安装uv工具
curl -LsSf https://astral.sh/uv/install.sh | sh

## 六、图形界面
