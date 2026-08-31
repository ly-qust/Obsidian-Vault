---
tags: [Java, IO, NIO, 网络, 序列化]
priority: P0
status: learning
---

# IO、NIO、网络与序列化

## 一句话结论

Java IO 的核心是“数据从哪里来、以字节还是字符解释、调用线程怎样等待、资源怎样关闭”；网络编程只是把另一端换成 socket。

> [!note] 主解释位置
> Java API → system call → FD/Socket、Blocking/Non-blocking、IO Multiplexing、select/poll/epoll 与 Sync/Async 的跨层链路统一见 [[操作系统面试精炼笔记/03-用户态内核态系统调用与FD]]、[[操作系统面试精炼笔记/06-IO多路复用与epoll]]。本文只保留 Java IO API、协议和工程选型。

## 一、字节流与字符流

- `InputStream/OutputStream` 处理原始字节，适合图片、压缩包、协议和任意二进制。
- `Reader/Writer` 处理字符，必须明确字符集。
- `InputStreamReader/OutputStreamWriter` 是字节与字符之间的桥梁。
- `Buffered*` 通过缓冲减少系统调用和小块传输。

文本没有“天然编码”。服务端统一显式使用 UTF-8，避免依赖机器默认字符集。

```java
try (BufferedReader reader = Files.newBufferedReader(path, StandardCharsets.UTF_8)) {
    return reader.lines().toList();
}
```

## 二、Files 与 Path

现代文件操作优先 `Path/Files`：路径拼接、存在性、复制移动、目录遍历、权限属性更清晰。

工程边界：

- 用户输入路径必须防止路径穿越。
- 大文件不要无条件 `readAllBytes`。
- 临时文件应限制目录、权限和生命周期。
- 写关键文件可先写临时文件，再在同文件系统内原子移动（是否支持需验证）。

## 三、缓冲、零拷贝与内存映射

- 缓冲减少用户态与内核态交互次数，但缓冲过大也增加内存和延迟。
- `FileChannel.transferTo/transferFrom` 可利用底层能力减少复制路径，实际效果依赖 OS/JDK。
- `MappedByteBuffer` 把文件区域映射到进程地址空间，适合随机访问大文件；需考虑页错误、文件生命周期和主动释放边界。
- “零拷贝”通常表示减少 CPU 参与的数据复制，不代表物理上完全没有任何数据移动。

OS 关联：[[操作系统面试精炼笔记/03-用户态内核态系统调用与FD]]。

## 四、BIO、NIO 与 AIO

跨层定义和 epoll 因果链见 [[操作系统面试精炼笔记/06-IO多路复用与epoll]]。

Java 侧只记 API 抽象：

| 模型 | Java 重点 | 工程边界 |
|---|---|---|
| BIO | Stream/Socket 阻塞式直线代码 | 大量平台线程等待成本高；虚拟线程可改变承载方式 |
| NIO | Channel、Buffer、Selector | Selector 只给就绪通知，EventLoop 不能跑慢业务 |
| AIO | CompletionHandler/Future 等完成通知 | 底层路径依 JDK/OS，不等于固定内核实现 |

Selector 是跨平台 Java 抽象，在 Linux 上实现可能利用 epoll，但不能把两个名字直接画等号。

## 五、Buffer 高频概念

- capacity：容量。
- position：下一次读/写位置。
- limit：当前可操作上界。
- `flip()`：从写模式切到读模式。
- `clear()`：准备重新写，不会把底层字节全部清零。
- `compact()`：保留未读数据并压到前部，继续写入。

## 六、TCP 是字节流

TCP 提供可靠、有序、面向连接的字节流，但没有消息边界。一次 send 不保证对应一次 read。

应用协议需要定义：

- 固定长度。
- 分隔符。
- 长度字段 + 消息体。
- 自描述协议。

必须处理半包、粘包、超时、连接关闭和重复请求。完整网络主线见 [[计算机网络面试精炼笔记/00-计算机网络知识地图]]。

## 七、Socket 到系统调用

完整主链见 [[操作系统面试精炼笔记/03-用户态内核态系统调用与FD]]。Java 侧记住：Socket/InputStream 是对象/API，FD 是进程句柄，内核 Socket 是网络对象；三者不能直接当同一个概念。

read 阻塞可能是等待数据、连接未建立、对端未响应或缓冲条件未满足；不能看到“read timeout”就断言网络丢包。监听 Socket 与 accept 后的 connected Socket 也要分开。

## 八、HTTP 客户端可靠调用

必须分别考虑：

- DNS 时间。
- connect timeout。
- TLS 握手。
- write timeout。
- read timeout。
- 连接池等待。
- 整体调用 deadline。

重试只适合满足幂等或具备幂等键的操作，并使用有限次数、退避和抖动。不要对所有 4xx/5xx 或超时无脑重试。

## 九、连接池

连接池复用昂贵连接并限制并发，但会引入新的排队点。

观察指标：活动连接、空闲连接、等待线程、获取连接耗时、超时、连接寿命和失败率。

线程池 200、数据库连接池 20 时，大量任务只会等待连接。容量设计必须把线程池、HTTP 连接池、数据库连接池和下游 QPS 一起看。

## 十、序列化

Java 原生序列化耦合类结构，存在兼容、安全和性能问题，不适合作为开放网络协议默认方案。

常见选择：

- JSON：可读、生态广，但体积和解析成本较高，类型约束需校验。
- Protobuf：契约明确、体积小、兼容演进规则清晰，需维护 schema。
- Java 内部对象：不要直接把实体对象结构当长期消息契约。

反序列化不可信输入前必须限制类型、大小、深度和资源消耗，避免任意类型实例化和反序列化漏洞。

## 十一、资源泄漏与排障

- try-with-resources 关闭流、连接、Statement、ResultSet。
- FD 持续升高：检查文件、socket、进程管道未关闭。
- 大量 CLOSE_WAIT：本端收到对方关闭但应用未正确 close。
- 大量 TIME_WAIT：通常是主动关闭连接的一端，结合连接复用和流量判断。
- 上传下载 OOM：检查整文件读入内存、缓冲、并发数和临时文件。

## 十二、高频追问

1. 字节流和字符流如何选择？
2. `flip` 和 `clear` 分别做什么？
3. NIO 一定比 BIO 快吗？
4. TCP 为什么会出现半包和粘包？
5. 虚拟线程是否让 NIO 失去价值？
6. 为什么连接池既提高性能又可能形成瓶颈？

## Reference

- [[java笔记/15IO流]]、[[java笔记/java中Stream，file和IO的核心知识]]
- [[java笔记/17网络编程]]
- [[计算机网络面试精炼笔记/04-HTTP-HTTPS与TLS]]
- [[计算机网络面试精炼笔记/07-网络故障排查]]
