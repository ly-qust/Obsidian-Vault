
```
Remove-Item -Recurse -Force .venv
```

我执行完删除venv环境命令后，执行第二条命令会报错，但是uv会自动重建环境

![[Pasted image 20260815100421.png|700]]





### ✅ 能说出 Python 与 Java 的 5 个差异

1. **缩进语法** — Java 用 `{}`, Python 用空格缩进
2. **类型声明** — Java 必须声明类型 `int x = 1;`，Python 可选 `x = 1` 或 `x: int = 1`
3. **字符串插值** — Java `String.format()`，Python `f"值={x}"`
4. **包结构** — Java `package com.example;`，Python 用目录 + `__init__.py`
5. **环境管理** — Java Maven 统一管理，Python 用 `uv`/pip + venv 单独管理

### 关闭终端后能按 README 重建环境

- 执行 `uv venv` 创建环境
- 执行 `uv run python -m app.main` 运行成功