# Java IO / Stream / File — 要点速览

下面把你给的文档内容浓缩、结构化并补充关键示例。目标是：要点突出、重点掌握、随手可用的参考页。每一节都有短示例，方便复制到工程里测试。

---

# 概览（一句话）

Java 的 IO 由 `java.io` 提供两套并行体系：**字节流**（处理二进制）和**字符流**（处理文本），配合 `File`、缓冲流、转换流与序列化类完成文件、目录、控制台等输入输出。实际开发中按数据类型（文本/二进制）选流、注意编码一致和资源关闭。

---

# 核心概念（必读）

- **流（Stream）**：有序的数据序列，分方向（输入 / 输出），分类型（字节 / 字符）。
    
- **抽象层次**：
    
    - 字节流顶层：`InputStream`, `OutputStream`
        
    - 字符流顶层：`Reader`, `Writer`
        
- **设计思想**：抽象类 + 子类实现 + 可叠加包装（Decorator），例如把 `FileInputStream` 包装到 `BufferedInputStream`。
    
- **选择原则**：
    
    - 二进制（图片、音频、压缩包） → 字节流
        
    - 文本（.txt、.csv、配置文件） → 字符流（按编码处理）
        
- **性能**：对大文件优先使用 `Buffered` 系列；对需要随机读写使用 `RandomAccessFile`。
    
- **资源管理**：必须关闭流（推荐 try-with-resources）。
    

---

# 控制台 IO

## 输入

- `Scanner`：API 简洁，适合快捷解析（数字/字符串/分隔符）。
    
    ```java
    Scanner sc = new Scanner(System.in);
    int n = sc.nextInt();
    String s = sc.next();
    sc.close();
    ```
    
- `BufferedReader` + `InputStreamReader`：适合逐行读取、需处理异常。
    
    ```java
    BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
    String line = br.readLine();
    ```
    

## 输出

- `System.out.print/println`（由 `PrintStream` 提供）是最常用。
    
- `PrintWriter` 可用于格式化输出并支持自动刷新：
    
    ```java
    PrintWriter pw = new PrintWriter(new OutputStreamWriter(System.out), true);
    pw.printf("Hello %s%n", "world");
    ```
    

---

# 字节流（处理二进制）

## 重要类（常用）

- `InputStream` / `OutputStream`（抽象）
    
- `FileInputStream` / `FileOutputStream`
    
- `BufferedInputStream` / `BufferedOutputStream`
    
- `DataInputStream` / `DataOutputStream`（Java 原生类型）
    
- `ObjectInputStream` / `ObjectOutputStream`（对象序列化）
    

## 核心方法

- `int read()`、`int read(byte[] b)`、`void write(int b)`、`void write(byte[] b)`、`flush()`、`close()`、`available()`
    

## 示例：拷贝二进制文件（带缓冲）

```java
try (InputStream in = new BufferedInputStream(new FileInputStream("src.png"));
     OutputStream out = new BufferedOutputStream(new FileOutputStream("dest.png"))) {
    byte[] buf = new byte[8192];
    int len;
    while ((len = in.read(buf)) != -1) {
        out.write(buf, 0, len);
    }
    out.flush();
} catch (IOException e) {
    e.printStackTrace();
}
```

---

# 字符流（处理文本）

## 重要类（常用）

- `Reader` / `Writer`（抽象）
    
- `FileReader` / `FileWriter`（系统默认编码）
    
- `BufferedReader`（`readLine()`） / `BufferedWriter`
    
- `InputStreamReader` / `OutputStreamWriter`（字节⇄字符，指定编码）
    
- `PrintWriter`（格式化、自动刷新）
    

## 为什么用转换流（`InputStreamReader`/`OutputStreamWriter`）

避免乱码：字符流必须明确编码（推荐 UTF-8）。`FileReader`/`FileWriter` 使用平台默认编码，容易出错。

## 示例：按 UTF-8 读写文本（带缓冲）

```java
try (BufferedReader br = new BufferedReader(new InputStreamReader(new FileInputStream("a.txt"), "UTF-8"));
     BufferedWriter bw = new BufferedWriter(new OutputStreamWriter(new FileOutputStream("b.txt"), "UTF-8"))) {

    String line;
    while ((line = br.readLine()) != null) {
        bw.write(line);
        bw.newLine();
    }
    bw.flush();
} catch (IOException e) {
    e.printStackTrace();
}
```

---

# File 与目录操作（常见任务）

## `File` 核心用法

- 表示路径（文件或目录），不等于文件内容
    
- 常用方法：`exists()`, `isFile()`, `isDirectory()`, `mkdir()`, `mkdirs()`, `list()`, `listFiles()`, `delete()`, `renameTo()`, `length()`, `getAbsolutePath()`
    

## 创建目录

```java
File dir = new File("path/to/dir");
if (!dir.exists()) {
    dir.mkdirs(); // 多级创建
}
```

## 列出目录

```java
File dir = new File("path");
File[] files = dir.listFiles();
for (File f : files) {
    System.out.println(f.getName() + (f.isDirectory() ? " [DIR]" : " [FILE]"));
}
```

## 递归删除目录（示例）

```java
public static boolean deleteRecursively(File file) {
    if (file.isDirectory()) {
        File[] children = file.listFiles();
        if (children != null) {
            for (File c : children) {
                if (!deleteRecursively(c)) return false;
            }
        }
    }
    return file.delete();
}
```

---

# 随机访问（`RandomAccessFile`）

- 支持任意位置读/写，常用于数据库引擎、文件索引等。
    

```java
try (RandomAccessFile raf = new RandomAccessFile("data.bin", "rw")) {
    raf.seek(100); // 跳到 offset=100
    int x = raf.readInt();
    raf.seek(200);
    raf.writeUTF("hello");
}
```

---

# 对象序列化

- `ObjectOutputStream` / `ObjectInputStream`，对象必须 `implements Serializable`。
    
- 注意：序列化与类的 `serialVersionUID`、字段兼容性、敏感数据（不要直接序列化密码）有关。
    

示例：

```java
// 序列化
try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("obj.dat"))) {
    oos.writeObject(myObject);
}
// 反序列化
try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("obj.dat"))) {
    MyClass obj = (MyClass) ois.readObject();
}
```

---

# 常见错误与最佳实践（必须记住）

1. **忘记关闭流 → 资源泄露**：使用 try-with-resources 自动关闭。
    
2. **编码不一致 → 乱码**：读写文本时始终指定相同编码（推荐 UTF-8）。
    
3. **读取单字节/字符时性能差**：对大文件使用缓冲流（`Buffered*`）。
    
4. **不要用 `FileReader/FileWriter` 处理非平台编码的文本**（它们使用平台默认编码）。
    
5. **`File.delete()` 只能删除空目录**：删除目录前先删除其内容（递归）。
    
6. **并发读写时要考虑同步或使用 NIO / Files API**（或文件锁）。
    
7. **处理大量小文件时性能问题**：考虑合并/打包或使用数据库/对象存储。
    
8. **异常处理**：IO 操作常抛 `IOException`，要妥善记录与处理资源回滚。
    

---

# 调试 IO 问题的检查清单（quick checklist）

- 路径是否正确（相对 vs 绝对）？
    
- 文件是否存在？权限是否允许读/写？
    
- 是否使用了正确的编码？（尤其跨平台）
    
- 是否已关闭流？是否使用 try-with-resources？
    
- 是否使用缓冲流来提高性能？
    
- 是否捕获并打印异常堆栈（便于定位）？
    

---

# 常用代码片段汇总（便于复制）

## 1) Try-with-resources 模板

```java
try (InputStream in = new FileInputStream("in");
     OutputStream out = new FileOutputStream("out")) {
    // do IO
} catch (IOException e) {
    e.printStackTrace();
}
```

## 2) 读取全部文本到 `String`

```java
StringBuilder sb = new StringBuilder();
try (BufferedReader br = new BufferedReader(new InputStreamReader(new FileInputStream("t.txt"), "UTF-8"))) {
    String line;
    while ((line = br.readLine()) != null) {
        sb.append(line).append(System.lineSeparator());
    }
}
String all = sb.toString();
```

## 3) 写字符串到文件（覆盖）

```java
try (BufferedWriter bw = new BufferedWriter(new OutputStreamWriter(new FileOutputStream("out.txt"), "UTF-8"))) {
    bw.write("Hello");
    bw.newLine();
    bw.flush();
}
```

---

# 小结（快速记忆卡）

- 二进制 → 字节流（`InputStream`/`OutputStream`）
    
- 文本 → 字符流（`Reader`/`Writer`），指定编码用 `InputStreamReader` / `OutputStreamWriter`
    
- 大文件 → 用 `Buffered*`
    
- 随机访问 → `RandomAccessFile`
    
- 对象持久化 → `Object*Stream`（注意兼容性与安全）
    
- 一律用 try-with-resources 管理资源，编码要一致
    

## 我的总结

-Properties类可以读配置文件![[Pasted image 20251111200409.png]]

---

如果你愿意，我可以 **生成一个“IO 核心类对照表（Markdown 或 CSV）“**，包含：类名、用途、典型构造/方法、示例代码片段和适用场景，便于直接插入 README 或开发手册。要 Markdown 表格还是 CSV？或者我直接把一页可打印的速查卡做成 PDF 给你？