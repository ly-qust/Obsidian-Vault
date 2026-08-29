> [!info] 当前主入口
> 面试与排障请看：[[nginx/Nginx面试与排障]]
> 本文保留为原始 Reference，需要补细节时再查。

# Nginx 知识总结（运维入门实战版）  
  
## 一、Nginx 是什么  
  
Nginx 是一个高性能的：  
  
- Web 服务器  
- 反向代理服务器  
- 负载均衡器  
- 静态资源服务器  
  
在运维场景中，Nginx 最常见的作用有：  
  
1. 提供网页访问服务  
2. 部署静态页面  
3. 把前端请求转发给后端服务  
4. 统一管理 80/443 端口入口  
5. 记录访问日志和错误日志  
  
---  
  
## 二、学习 Nginx 的核心目标  
  
学 Nginx 不能只停留在“会安装”，而要做到：  
  
- 会安装  
- 会启动  
- 会访问  
- 会修改配置  
- 会看日志  
- 会反向代理  
- 会排查常见故障  
  
---  
  
## 三、Nginx 常见目录  
  
### 1. 主配置文件  
  
常见位置： 
  
``` bash  
/etc/nginx/nginx.conf
```

这是 Nginx 的主配置文件。

---

### 2. 站点配置目录

不同系统常见位置不同：

#### Ubuntu / Debian

/etc/nginx/sites-available/  
/etc/nginx/sites-enabled/

#### CentOS / Rocky / AlmaLinux

/etc/nginx/conf.d/

---

### 3. 默认网页目录

#### Ubuntu 常见

/var/www/html

#### CentOS 常见

/usr/share/nginx/html

---

### 4. 日志目录

/var/log/nginx/

常见日志文件：

- `access.log`：访问日志
- `error.log`：错误日志

---

## 四、Nginx 的安装与启动

### 1. 安装

#### Ubuntu / Debian

sudo apt update  
sudo apt install nginx -y

#### CentOS / Rocky / AlmaLinux

sudo yum install nginx -y

或者：

sudo dnf install nginx -y

---

### 2. 启动服务

sudo systemctl start nginx

---

### 3. 设置开机自启

sudo systemctl enable nginx

---

### 4. 查看状态

systemctl status nginx

重点看：

Active: active (running)

表示 Nginx 正常运行。

---

## 五、Nginx 基础验证方法

### 1. 查看 80 端口是否监听

ss -lntp | grep :80

---

### 2. 本机访问测试

curl http://127.0.0.1

如果返回 HTML 内容，说明服务基本正常。

---

### 3. 浏览器访问测试

http://服务器IP

查看本机 IP：

ip a

---

## 六、Nginx 最常用的运维命令

### 1. 检查配置文件语法

sudo nginx -t

这是最重要的命令之一。

如果输出：

syntax is ok  
test is successful

说明配置没有语法错误。

---

### 2. 重载配置

sudo systemctl reload nginx

适合小改配置后生效，不中断服务。

---

### 3. 重启服务

sudo systemctl restart nginx

适合服务异常或配置修改较大时。

---

### 4. 查看运行状态

systemctl status nginx

---

### 5. 查看访问日志

tail -f /var/log/nginx/access.log

---

### 6. 查看错误日志

tail -f /var/log/nginx/error.log

---

### 7. 查看 systemd 管理日志

journalctl -u nginx -n 50

---

## 七、修改默认网页内容

### 1. 先找到默认网页文件

可能在：

/var/www/html/index.html

或者：

/usr/share/nginx/html/index.html

---

### 2. 备份原文件

sudo cp /usr/share/nginx/html/index.html /usr/share/nginx/html/index.html.bak

---

### 3. 写入自己的页面

echo '<h1>Hello Nginx</h1><p>这是我部署的第一个页面</p>' | sudo tee /usr/share/nginx/html/index.html

如果你的目录是 `/var/www/html`，就改成对应路径。

---

### 4. 重新访问验证

curl http://127.0.0.1

如果看到自己的内容，说明静态页面部署成功。

---

## 八、Nginx 最重要的功能：反向代理

### 1. 什么是反向代理

反向代理就是：

- 用户访问 Nginx 的 80 端口
- Nginx 再把请求转发给后端服务
- 后端返回结果后，Nginx 再返回给客户端

用户只需要访问 Nginx，不需要直接访问后端端口。

---

### 2. 最简单的反向代理示例

假设后端服务跑在：

127.0.0.1:9000

Nginx 配置示例：

server {  
    listen 80;  
    server_name _;  
  
    location /app/ {  
        proxy_pass http://127.0.0.1:9000/;  
    }  
}

这样访问：

http://服务器IP/app/

就会被转发到后端 `9000` 端口。

---

## 九、Nginx 反向代理实战思路

### 1. 先启动一个简单后端

mkdir -p /tmp/nginx-demo  
cd /tmp/nginx-demo  
echo '这是后端服务返回的内容' > index.html  
python3 -m http.server 9000

---

### 2. 测试后端是否正常

curl http://127.0.0.1:9000

---

### 3. 配置 Nginx 代理

server {  
    listen 80;  
    server_name _;  
  
    location /app/ {  
        proxy_pass http://127.0.0.1:9000/;  
    }  
}

---

### 4. 检查配置并重载

sudo nginx -t  
sudo systemctl reload nginx

---

### 5. 访问验证

curl http://127.0.0.1/app/

如果能返回后端内容，说明反向代理成功。

---

## 十、Nginx 常见故障与排查方法

---

### 故障 1：Nginx 启动失败

#### 常见原因

- 配置文件语法错误
- 端口被占用
- 配置路径写错
- 权限问题

#### 排查顺序

systemctl status nginx  
sudo nginx -t  
journalctl -u nginx -n 50  
tail -f /var/log/nginx/error.log

---

### 故障 2：80 端口被占用

#### 现象

Nginx 启动时报错，无法绑定 80 端口。

#### 排查命令

ss -lntp | grep :80

#### 结论

如果发现别的程序占用了 80 端口，Nginx 就无法正常启动。

---

### 故障 3：访问页面返回 502 Bad Gateway

#### 常见原因

- 后端服务没启动
- 后端端口不通
- `proxy_pass` 地址写错

#### 排查命令

tail -f /var/log/nginx/error.log  
ss -lntp | grep 9000  
curl http://127.0.0.1:9000

#### 核心理解

`502` 往往不是 Nginx 本身坏了，而是它代理的后端服务有问题。

---

### 故障 4：访问页面返回 403 Forbidden

#### 常见原因

- 目录权限不够
- Nginx 无法读取网页目录
- 资源路径配置错误

#### 排查命令

tail -f /var/log/nginx/error.log  
ls -ld /usr/share/nginx/html  
ls -ld /var/www/html

#### 结论

403 常常与权限有关。

---

### 故障 5：本机能访问，其他机器访问不了

#### 常见原因

- 防火墙未放行 80 端口
- 云服务器安全组未开放
- 网络策略限制

#### 排查命令

curl http://127.0.0.1  
curl http://服务器IP  
ss -lntp | grep :80

#### 查看防火墙

##### CentOS / Rocky

sudo firewall-cmd --list-all  
sudo firewall-cmd --permanent --add-service=http  
sudo firewall-cmd --reload

##### Ubuntu

sudo ufw status  
sudo ufw allow 80/tcp

---

## 十一、Nginx 排障核心顺序

以后 Nginx 出问题，按这个顺序查：

### 1. 服务是否运行

systemctl status nginx

### 2. 配置是否正确

sudo nginx -t

### 3. 端口是否监听

ss -lntp | grep :80  
ss -lntp | grep :443

### 4. 本机是否可以访问

curl http://127.0.0.1

### 5. 日志写了什么

tail -f /var/log/nginx/error.log  
tail -f /var/log/nginx/access.log  
journalctl -u nginx -n 50

### 6. 如果是代理问题，后端是否正常

ss -lntp | grep 9000  
curl http://127.0.0.1:9000

---

## 十二、Nginx 学习后的核心理解

Linux 运维里，学 Nginx 最关键的不是背配置，而是形成以下思维：

### 1. 改配置前先备份

sudo cp 配置文件 配置文件.bak

---

### 2. 改完配置先检查语法

sudo nginx -t

---

### 3. 配置生效常用 reload

sudo systemctl reload nginx

---

### 4. 服务起不来先看状态和日志

systemctl status nginx  
journalctl -u nginx -n 50

---

### 5. 页面访问失败要区分问题类型

- 服务没起来
- 端口没监听
- 配置写错
- 后端挂了
- 权限不对
- 防火墙拦截

---

## 十三、Nginx 实战中必须会的命令汇总

sudo apt install nginx -y  
sudo yum install nginx -y  
sudo systemctl start nginx  
sudo systemctl enable nginx  
systemctl status nginx  
sudo nginx -t  
sudo systemctl reload nginx  
sudo systemctl restart nginx  
ss -lntp | grep :80  
curl http://127.0.0.1  
ip a  
tail -f /var/log/nginx/access.log  
tail -f /var/log/nginx/error.log  
journalctl -u nginx -n 50
