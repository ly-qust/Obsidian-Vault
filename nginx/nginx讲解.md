> [!info] 当前主入口
> 面试与排障请看：[[nginx/Nginx面试与排障]]
> 本文保留为原始 Reference，需要补细节时再查。

在你的高并发政务预约系统中，Nginx 是整个系统的**“大门”**。如果把你的 Java 后端程序比作办公窗口里的办事员，Nginx 就是政务大厅的**前台接待、保安、以及自动排号机**。

让我们结合你的项目背景，详细拆解这三大核心功能：

---

### 一、 静态资源服务器 (Web Server)
**通俗理解：宣传海报和填表说明**

在 Web 应用中，有很多文件是永远不变的，比如：
*   系统的登录页、背景图。
*   预约须知、操作流程的 PDF 或是 HTML。
*   页面的样式表 (CSS) 和逻辑脚本 (JS)。

**为什么需要 Nginx 做这个？**
Java 程序（Tomcat）处理动态业务逻辑（比如扣减名额）很厉害，但处理静态文件很费劲。
*   **做法：** 把前端打包好的文件放在 Nginx 服务器上。
*   **好处：** Nginx 处理静态文件的效率极高，请求静态文件时根本不需要惊动 Java 后端，极大地节省了后端的 CPU 和内存资源。

---

### 二、 反向代理 (Reverse Proxy)
**通俗理解：办事处的“前台接待”**

当市民访问你的政务系统时，他们访问的是 Nginx 的 IP。他们并不知道，也没必要知道真正的 Java 程序运行在哪台机器、哪个端口。

**反向代理的作用：**
1.  **保护隐私：** 隐藏真实的 Java 服务端信息。外网只能看到 Nginx，攻击者很难直接攻击你的后端数据库和逻辑。
2.  **统一入口：** 如果你有多个功能模块（比如一个模块管预约，一个模块管公告），Nginx 可以根据路径分流：
    *   访问 `/api/v1/appointment` -> 转发到 Java 预约服务。
    *   访问 `/notice` -> 转发到公告服务。
3.  **解决跨域问题：** 前端和后端如果不在同一个端口，会有跨域限制。Nginx 代理可以把它们统一在同一个端口下。

---

### 三、 负载均衡 (Load Balancing) —— 【高并发核心】
**通俗理解：排号机分流（多开几个办事窗口）**

这是你的课程设计能否被称为“高并发”的关键。
单台服务器能承受的并发量是有限的（比如只能扛 500 人同时在线）。如果突然来了 2000 人怎么办？

**负载均衡的逻辑：**
1.  **横向扩展：** 你启动 4 台相同的 Java 预约服务（分别跑在 8081, 8082, 8083, 8084 端口）。
2.  **分发策略：** Nginx 接收到 2000 个请求后，按照一定的规则（比如轮询）均匀地分发给这 4 台服务器。
3.  **效果：** 每台服务器只处理 500 个请求，压力瞬间减轻，系统依然流畅。

---

### 🛠️ Nginx 核心配置长什么样？（项目实操参考）

在你的课程设计中，`nginx.conf` 里的核心配置大概会是这样：

```nginx
# 1. 定义负载均衡池（定义4个办事窗口）
upstream java_appointment_cluster {
    server 192.168.1.10:8081 weight=1; # 窗口A
    server 192.168.1.10:8082 weight=1; # 窗口B
    server 192.168.1.10:8083 weight=1; # 窗口C
}

server {
    listen       80;
    server_name  www.yuyue.gov.cn;

    # 2. 静态资源服务器：处理前端页面
    location / {
        root   /usr/share/nginx/html;
        index  index.html;
    }

    # 3. 反向代理 + 负载均衡：处理后端API请求
    location /api/ {
        # 把以 /api 开头的请求，转发到上面的集群中
        proxy_pass http://java_appointment_cluster;
        
        # 传递真实的用户IP（防止Java后端只看到Nginx的IP）
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

---

### 💡 如何在课程设计中体现出“高并发”水平？

如果你能配置 Nginx 的**“漏桶限流” (Rate Limiting)**，分数会更高。
政务系统最怕被人用脚本恶意刷票，你可以在 Nginx 增加如下配置：

```nginx
# 限制单个IP每秒只能发起 1 次预约请求
limit_req_zone $binary_remote_addr zone=myLimit:10m rate=1r/s;

server {
    location /api/appointment/ {
        limit_req zone=myLimit burst=5 nodelay; # 突发流量允许5个，超过的直接报错
        proxy_pass http://java_appointment_cluster;
    }
}
```

### 总结
*   **Web 服务器：** 减轻 Java 压力，处理图片和网页。
*   **反向代理：** 安全第一，统一入口。
*   **负载均衡：** 解决单台服务器性能瓶颈，实现“大力出奇迹”。

**建议：** 这一部分不需要死记硬背，你可以尝试在本地安装一个 Nginx，然后启动两个不同的 Spring Boot 端口，亲手配置一下负载均衡。看到请求在两个后台来回切换时，你对它的理解会瞬间通透！
