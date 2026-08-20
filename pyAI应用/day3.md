

## 1.类型标注
```
# Java: String name = "政策助手";
name: str = "政策助手"

# Java: int count = 42;
count: int = 42

# Java: List<String> tags = Arrays.asList("ai", "rag");
tags: list[str] = ["ai", "rag"]

# Java: Map<String, Integer> scores = Map.of("AI", 95);
scores: dict[str, int] = {"AI": 95}

# Java: Optional<String> name = Optional.empty();
optional_value: str | None = None
```

## 2.模拟包模块（类比Java来看）
![[Pasted image 20260815141637.png]]


## 3.![[Pasted image 20260815142231.png]]

1. Python `__init__`是构造函数，`self`等价 Java 的`this`；
2. class 内部直接写的变量是**类变量，等价 Java static**；`self.xxx`是实例变量；
3. 实例方法第一个参数必须是 self；
4. Python 没有 Java 的 public/private 访问权限；类型注解只是提示，运行不强制。


## 4. dataclass
好处有三点：
- 自动创建 `__init__`，不用手写构造函数
- 自动创建 `__repr__`，打印可读对象（否则打印的是内存地址）
- 自动创建 `__eq__`，比较相等（否则比较的是内存地址，数据相同但内存地址不相同）

###  5.能解释 default_factory

为什么用 `field(default_factory=list)` 而不是 `tags=[]`
- 默认参数只创建一次，所有实例共享，这样会导致后续创建的实例共享同一份数据，无法做到更新等操作
- `default_factory` 每次调用创建新对象，每个实例独立