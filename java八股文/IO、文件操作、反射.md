## 一、IO / NIO 一页复习卡

### 1. IO 的本质

IO 是程序与外部数据源之间的数据传输。

```text
读取 Input：外部数据源 → Java 程序
写出 Output：Java 程序 → 外部数据源
```

外部数据源可以是文件、网络、设备或内存区域，今天主要讨论文件。

### 2. 字节流和字符流

| 对比 | 字节流 | 字符流 |
|---|---|---|
| 处理视角 | 原始 `byte` | 按编码解码后的字符 |
| 常见抽象 | `InputStream` / `OutputStream` | `Reader` / `Writer` |
| 适合场景 | 图片、视频、压缩包、任意二进制 | 文本读取和写入 |
| 关键风险 | 不关心字符编码 | 编码选择错误会乱码 |

面试表达：

> 字节流处理原始字节，不关心字符编码，适合二进制数据；字符流按照指定编码把字节解码成字符，适合文本。复制 JPG 应使用字节流，读取 UTF-8 文本可以使用字符流并明确 UTF-8。

注意：

- 字节流不代表每次只能读取一个字节，也可以使用 `byte[]` 批量读取。
- 文本文件在磁盘上仍然存储为字节；字符流负责“字节 ↔ 字符”的编码转换。

### 3. 缓冲解决什么问题

```text
大量小粒度 IO
→ 多次请求底层 IO
→ 固定开销被频繁支付

使用 Buffer
→ 每次读取/写入一批数据
→ 程序从内存缓冲区消费
→ 减少底层 IO 次数
```

面试表达：

> Buffered 流通过一次处理一批数据，减少频繁的小粒度底层 IO 操作，因此通常更高效。缓冲区大小有限，不等于把整个文件全部加载进内存。

常见名字只需认识：

```text
BufferedInputStream / BufferedOutputStream
BufferedReader / BufferedWriter
```

### 4. 传统 IO 与 NIO

```text
传统 IO：以 Stream 为中心

NIO：以 Channel + Buffer 为中心

文件 ↔ Channel ↔ Buffer ↔ 程序
```

| 概念 | 最小理解 |
|---|---|
| `Channel` | 数据传输通道 |
| `Buffer` | 暂时存放一批数据的内存区域 |
| `FileChannel` | 面向文件的 Channel，可配合 Buffer 分块读写 |
| `Path` | 对文件或目录路径的描述 |
| `Files` | 常用文件操作的高层便捷 API |

面试表达：

> 传统 IO 主要以 Stream 为中心；NIO 主要以 Channel 和 Buffer 为中心，数据通常通过 Channel 进入 Buffer，再由程序处理。NIO 还提供非阻塞等模型，在高并发网络场景价值较大，但不代表任何情况下都比传统 IO 快。

### 5. 选择边界

```text
二进制数据
→ 字节视角

文本内容
→ 字符/编码视角

简单文件读取、写入、复制
→ Path + Files

需要显式控制分块传输
→ FileChannel + Buffer
```

`Files` 本身就是 NIO 文件 API，不能把“Files”和“NIO”当成两个对立选项。

### 6. 常见坑

- 忘记明确文本编码，导致乱码。
- 把特别大的文件一次性收集进 `List`，造成内存压力，严重时可能 `OutOfMemoryError`。
- `Files.lines()` 返回的文件 Stream 没有关闭，造成文件句柄泄漏。
- 相对路径基于当前工作目录解析，容易触发 `NoSuchFileException`。
- 把 `flush()` 和 `close()` 混淆：`flush()` 刷新待写数据，`close()` 释放底层资源。
- 把终端乱码误认为文件一定读取错误；文件解码和终端输出编码是两层问题。

---

## 二、NIO 最小实操：统计文本行数

最终代码：[`work/day2-nio/NioLineCounter.java`](work/day2-nio/NioLineCounter.java)  
测试文件：[`work/day2-nio/sample.txt`](work/day2-nio/sample.txt)

```java
import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.stream.Stream;

public class NioLineCounter {

    public static void main(String[] args) {
        if (args.length != 1) {
            System.err.println("用法: java NioLineCounter <文件路径>");
            return;
        }

        Path path = Path.of(args[0]);

        try (Stream<String> lines =
                 Files.lines(path, StandardCharsets.UTF_8)) {
            long lineCount = lines.count();
            System.out.println("文件行数: " + lineCount);
        } catch (IOException e) {
            System.err.println("读取文件失败: " + path.toAbsolutePath());
            e.printStackTrace();
        }
    }
}
```

### 为什么这样写

| 代码 | 原因 |
|---|---|
| `args.length != 1` | 先校验输入，避免直接访问不存在的参数 |
| `Path.of(args[0])` | 将命令行路径转换为 `Path` |
| `UTF_8` | 明确文本解码方式 |
| `try-with-resources` | 离开代码块时自动调用 `lines.close()` |
| `lines.count()` | 流式统计，不构造保存全部行的 `List` |
| `catch (IOException)` | 输出失败路径和异常堆栈 |

### 可能出什么错，怎么排

```text
NoSuchFileException
→ 先检查实际传入的路径
→ 检查当前工作目录
→ 检查文件是否存在

MalformedInputException / 乱码
→ 检查文件真实编码
→ 区分文件解码问题和终端显示编码问题

内存占用过高
→ 检查是否使用 readAllLines()/toList() 保存全部内容

文件长期占用
→ 检查 Files.lines() 返回的 Stream 是否关闭
```

排查堆栈时先看：

1. 异常类型说明“发生了什么”。
2. 堆栈中第一个属于自己代码的位置说明“从哪里触发”。

### AI 代码 Review 结论

不推荐：

```java
Stream<String> lines = Files.lines(path, StandardCharsets.UTF_8);
List<String> allLines = lines.toList();
return allLines.size();
```

问题：Stream 未及时关闭，并且 `toList()` 保存所有行，大文件内存风险较高。

推荐：

```java
try (Stream<String> lines =
         Files.lines(path, StandardCharsets.UTF_8)) {
    return lines.count();
}
```

`count()` 负责消费数据并统计；`close()` 负责释放文件句柄，两者职责不同。

---

## 三、反射一页复习卡

### 1. 反射是什么

> Java 反射允许程序在运行时获取类的信息，并动态访问构造器、字段和方法，甚至创建对象和调用方法。

普通代码：

```java
User user = new User();
```

编译时已经知道类型是 `User`。

运行时才拿到类名：

```java
Class<?> clazz = Class.forName("com.example.User");
```

### 2. 四个核心概念

```text
Class       → 我是谁？类型信息的入口
Constructor → 怎样创建对象？
Method      → 有哪些方法，怎样调用？
Field       → 有哪些字段，怎样访问？
```

```java
Class<?> clazz = User.class;

clazz.getDeclaredConstructors();
clazz.getDeclaredMethods();
clazz.getDeclaredFields();

Method method = clazz.getDeclaredMethod("getName");
Object result = method.invoke(user);
```

- `getDeclaredMethods()` 返回方法信息，不会执行方法。
- `method.invoke(user)` 中的 `user` 是目标对象。
- `invoke()` 统一返回 `Object`；原方法返回 `void` 时结果为 `null`。

### 3. 框架为什么需要反射

Spring 框架发布时，不知道未来项目会定义哪些业务类。例如 Spring 发布时，项目中的 `OrderService` 还不存在。

```text
项目启动
→ Spring 扫描类和注解
→ 发现 @Service
→ 获取 Class 信息
→ 查看构造器
→ 创建 Bean
→ 注入依赖
```

通用性：同一套 Spring 代码可以管理不同项目未来定义的各种 Bean。  
动态性：具体处理哪个类，是程序运行时扫描后决定的，不是硬编码在 Spring 框架中。

其他框架场景：

```text
Jackson → JSON 与 Java 对象转换
MyBatis → 查询结果映射到 Java 对象
```

### 4. 反射的代价

- 可读性和可维护性可能下降。
- 类型安全弱于普通直接调用。
- 方法不存在等错误可能从编译期推迟到运行期。
- 存在一定运行时开销。
- 私有访问可能受到访问控制和模块系统限制。

面试表达：

> 框架编写时不知道未来项目中的具体业务类，因此需要反射提供通用性和运行时动态能力，例如扫描注解、创建 Bean 和注入依赖。普通业务代码的类型通常明确，直接调用更容易阅读、维护并获得编译期检查，因此不建议为了显得高级而滥用反射。

注意：Spring 事务主要依赖 AOP 代理。反射是框架动态能力的一部分，不能简单说“事务完全由反射实现”。