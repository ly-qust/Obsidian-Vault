编译：
javac -d classes src\module.java src\com\...\Main.java等等
每个都写全，从根目录开始写，所有.class文件输出到classes目录下
运行：
java --module-path classes --module 模块名/主类名
`--module-path classes` 告诉 JVM “到 `classes` 目录下找模块”
--module告诉jvm运行模块名的某个主类
