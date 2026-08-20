
## 1.配置优先级
- application.properties
    

```Properties
server.port=8081
```

- application.yml
    

```YAML
server:
   port: 8082
```

- application.yaml
    

```YAML
server:
   port: 8082
```

SpringBoot为了增强程序的扩展性，除了支持配置文件的配置方式以外，还支持另外两种常见的配置方式：

1. Java系统属性配置 （格式： -Dkey=value）
    

```YAML
-Dserver.port=9000
```

  

2. 命令行参数 （格式：--key=value）
    

```YAML
--server.port=10010
```

**五种配置方式的优先级：** 命令行参数 > 系统属性参数 > properties参数 > yml参数 > yaml参数

## 2.Bean
![[Pasted image 20251129140208.png]]
![[Pasted image 20251129142355.png]]
![[Pasted image 20251129143337.png]]


3.自定义starter
![[Pasted image 20251202155621.png]]