

## 函数参数

1.python支持**函数默认参数**，`greeting="欢迎"`：调用的时候可以不传第二个参数，自动使用默认值

```
greet("张三")           # greeting 使用默认 "欢迎" → 欢迎，张三
greet("李四", "你好")   # 传入实参，覆盖默认值 → 你好，李四
```


2.
```
def total(*nums):
    # *nums 把多个传入参数打包成元组 tuple
    return sum(nums)

print(total(1,2,3))   # 6
print(total(10,20))   #30
```
`*args`（这里变量名叫 nums，约定俗成写 args）：接收**任意数量位置参数**，内部变成元组`tuple`


3.
```
def show(**kw):
    # **kw 把传入的关键字参数打包成字典 dict
    print(kw)

show(env="dev", port=8000)
# 输出：{'env': 'dev', 'port': 8000}
```

- `**kwargs`：只接收`key=value`形式的**关键字参数**，在函数内部得到一个字典`dict`
- 只能放在参数列表的**最后面**
- 调用不能传位置参数，只能传`key=val`

## 作用域
```python
# Python 只有函数作用域，没有块作用域
x = 1
if True:
    y = 2  #此处不同于java，这里的y不是局部变量，在函数外也可以用 
print(y)  
# 全局变量修改需要 global 关键字
count = 0
def increment():
    global count
    count += 1
```



## list/dict是引用传递
```python
# list/dict 是引用传递
a = [1, 2, 3]
b = a          # b 指向同一个对象!
b.append(4)
print(a)       # [1, 2, 3, 4] ← a 也被改了!

# 正确做法: 显式复制
c = a.copy()   # 或 list(a)  
c.append(5)
print(a)       # [1, 2, 3, 4] ← a 不受影响
```


### 清洗函数清单

| 函数                         | 作用            | 对应 Java                          |
| -------------------------- | ------------- | -------------------------------- |
| `clean_policy_text(raw)`   | 去空白、压缩多余空格    | `replaceAll("\\s+", " ").trim()` |
| `normalize_question(raw)`  | 清洗 + 小写 + 去标点 | 链式调用多个方法                         |
| `truncate_text(text, max)` | 超长截断加 "..."   | `substring()` + 判断               |
```
"""  
Day 2 核心任务: 政策文本清洗函数  
对应学习计划: 写政策文本清洗、空白处理、问题规范化函数  
覆盖: 正常、空值、中文、超长文本至少 10 组测试  
"""  
  
from __future__ import annotations  
  
  
# ============================================================  
# 清洗函数 (Day 2 要写的核心代码)  
# ============================================================  
  
  
def clean_policy_text(raw: str) -> str:  
    """  
    清洗政策文本: 去除首尾空白、压缩多余空白、统一换行。  
  
    类似 Java:      public static String cleanPolicyText(String raw) {          return raw.replaceAll("\\s+", " ").trim();      }  
    Args:        raw: 原始政策文本，可能包含多余空白、换行、制表符等  
    Returns:        清洗后的文本，连续空白压缩为单个空格  
    """    # 先替换所有空白字符(空格/制表/换行)为单个空格  
    cleaned: str = " ".join(raw.split())  
    return cleaned.strip()  
  
  
def normalize_question(raw: str) -> str:  
    """  
    规范化用户问题: 去空白、转小写、去除末尾标点。  
  
    类似 Java:      public static String normalizeQuestion(String raw) {          return raw.trim().toLowerCase().replaceAll("[.!?]+$", "");      }  
    Args:        raw: 用户问题，可能包含多余空白和标点  
    Returns:        规范化后的问题，用于检索匹配  
    """    normalized: str = clean_policy_text(raw)  # 复用 clean_policy_text    normalized = normalized.lower()  # 转小写，类似 Java: raw.trim().toLowerCase()    # 去除末尾的 . ！ ？ 等标点  
    import re  
    normalized = re.sub(r"[.！？.!?]+$", "", normalized)  
    return normalized  
  
  
def truncate_text(text: str, max_length: int = 200) -> str:  
    """  
    截断文本，防止超长内容。  
  
    类似 Java:      public static String truncate(String text, int maxLength) {          if (text.length() <= maxLength) return text;          return text.substring(0, maxLength) + "...";      }  
    Args:        text: 原始文本  
        max_length: 最大长度，默认 200    Returns:        截断后的文本，超长时末尾加 "..."    """    if len(text) <= max_length:  
        return text  
    return text[:max_length] + "..."  
  
  
# ============================================================  
# 测试数据  
# ============================================================  
  
  
# 正常政策文本  
POLICY_NORMAL: str = "    国补政策申请指南    \n\n  适用于2024年度\n    以旧换新补贴    "  
# 包含多余空白的超长文本  
POLICY_LONG: str = "这是一段测试用的超长政策文本，" + "重复内容 用于测试截断功能。\n\n" * 20  
  
# 包含特殊空白的文本  
POLICY_WHITESPACE: str = " \t\n\r  国补政策申请指南  \t\n\r "  
# 包含中英文混合的文本  
POLICY_MIXED: str = "  申请国补政策要求提交身份证复印件和发票原件  ID: 110101199001011234  "  
  
# ============================================================  
# 测试函数 (10+ 组测试)  
# ============================================================  
  
  
def test_clean_policy_text_normal() -> bool:  
    """测试1: 正常文本清洗"""  
    result: str = clean_policy_text(POLICY_NORMAL)  
    # 连续空白应压缩为单个空格  
    assert "  " not in result, f"存在连续空白: {result}"  
    assert not result.startswith(" "), f"应以非空白开头: {result}"  
    assert not result.endswith(" "), f"应以非空白结尾: {result}"  
    # 内容应保留  
    assert "国补政策申请指南" in result, f"内容丢失: {result}"  
    print(f"  [测试1] 正常文本清洗: {result[:50]}...")  
    return True  
  
  
def test_clean_policy_text_empty() -> bool:  
    """测试2: 空字符串"""  
    result: str = clean_policy_text("")  
    assert result == "", f"空字符串应返回空: {result!r}"  
    print("  [测试2] 空字符串清洗: OK")  
    return True  
  
  
def test_clean_policy_text_only_whitespace() -> bool:  
    """测试3: 纯空白字符串"""  
    result: str = clean_policy_text("   \t\n   ")  
    assert result == "", f"纯空白应返回空: {result!r}"  
    print("  [测试3] 纯空白清洗: OK")  
    return True  
  
  
def test_clean_policy_text_mixed() -> bool:  
    """测试4: 中英文混合"""  
    result: str = clean_policy_text(POLICY_MIXED)  
    assert "国补政策" in result, f"中文内容应保留: {result}"  
    assert "ID:" in result, f"英文内容应保留: {result}"  
    assert "  " not in result, f"无连续空白: {result}"  
    print(f"  [测试4] 中英文混合清洗: {result[:60]}...")  
    return True  
  
  
def test_clean_policy_text_long() -> bool:  
    """测试5: 超长文本(不截断，只清洗)"""  
    result: str = clean_policy_text(POLICY_LONG)  
    assert len(result) < len(POLICY_LONG), "清洗后长度应减少"  
    assert "  " not in result, "无连续空白"  
    print(f"  [测试5] 超长文本清洗: 原长{len(POLICY_LONG)}, 清洗后{len(result)}")  
    return True  
  
  
def test_normalize_question_normal() -> bool:  
    """测试6: 正常问题规范化"""  
    result: str = normalize_question("  国补政策是什么?  ")  
    assert result == "国补政策是什么", f"标点应去除: {result!r}"  
    assert result == result.lower(), f"应转小写: {result!r}"  
    print(f"  [测试6] 正常问题规范化: '{result}'")  
    return True  
  
  
def test_normalize_question_with_punctuation() -> bool:  
    """测试7: 带多种标点的问句"""  
    result: str = normalize_question("申请国补需要什么材料...")  
    assert "..." not in result, f"标点应去除: {result!r}"  
    print(f"  [测试7] 多标点问句规范化: '{result}'")  
    return True  
  
  
def test_normalize_question_empty() -> bool:  
    """测试8: 空问题"""  
    result: str = normalize_question("")  
    assert result == "", f"空输入应返回空: {result!r}"  
    print("  [测试8] 空问题规范化: OK")  
    return True  
  
  
def test_normalize_question_already_clean() -> bool:  
    """测试9: 已清洗的问题"""  
    result: str = normalize_question("申请国补需要什么材料")  
    assert result == "申请国补需要什么材料", f"已清洗应不变: {result!r}"  
    print(f"  [测试9] 已清洗问题规范化: '{result}'")  
    return True  
  
  
def test_truncate_text_short() -> bool:  
    """测试10: 短文本不截断"""  
    result: str = truncate_text("短文本", max_length=200)  
    assert result == "短文本", f"短文本不应截断: {result!r}"  
    print("  [测试10] 短文本不截断: OK")  
    return True  
  
  
def test_truncate_text_long() -> bool:  
    """测试11: 超长文本截断"""  
    result: str = truncate_text(POLICY_LONG, max_length=50)  
    assert len(result) <= 53, f"截断后不应超过 max_length+3: {len(result)}"  # 50 + "..."  
    assert result.endswith("..."), f"截断文本应以省略号结尾: {result!r}"  
    print(f"  [测试11] 超长文本截断: {len(result)} 字符, 以省略号结尾")  
    return True  
  
  
def test_truncate_text_exact_length() -> bool:  
    """测试12: 刚好等于 max_length 不截断"""  
    result: str = truncate_text("正好二十个字以内", max_length=10)  
    assert result == "正好二十个字以内", f"刚好不应截断: {result!r}"  
    print("  [测试12] 刚好等于长度不截断: OK")  
    return True  
  
def test_clean_policy_text_null_like() -> bool:  
    """测试: 仅包含换行和制表符"""  
    result = clean_policy_text("\n\t\r\n  \t  ")  
    assert result == ""  
    print("  [测试] 换行+制表符清洗: OK")  
    return True  
  
# ============================================================  
# 测试入口  
# ============================================================  
  
  
def run_all_tests() -> tuple[int, int]:  
    """运行所有测试，返回 (通过数, 总数)"""  
    tests: list[tuple[str, callable]] = [  
        ("正常文本清洗", test_clean_policy_text_normal),  
        ("空字符串", test_clean_policy_text_empty),  
        ("纯空白", test_clean_policy_text_only_whitespace),  
        ("中英文混合", test_clean_policy_text_mixed),  
        ("超长文本清洗", test_clean_policy_text_long),  
        ("正常问题规范化", test_normalize_question_normal),  
        ("多标点问句", test_normalize_question_with_punctuation),  
        ("空问题", test_normalize_question_empty),  
        ("已清洗问题", test_normalize_question_already_clean),  
        ("短文本不截断", test_truncate_text_short),  
        ("超长文本截断", test_truncate_text_long),  
        ("刚好等于长度", test_truncate_text_exact_length),  
        ('仅包含换行和制表符', test_clean_policy_text_null_like),  
    ]  
  
    passed: int = 0  
    failed: int = 0  
  
    for name, func in tests:  
        try:  
            func()  
            passed += 1  
        except Exception as e:  
            failed += 1  
            print(f"  [FAIL] {name}: {e}")  
  
    return passed, len(tests)  
  
  
def main() -> None:  
    """Day 2 清洗函数演示入口"""  
    print("=" * 50)  
    print("  Day 2 — 政策文本清洗函数 + 12 组测试")  
    print("=" * 50)  
  
    passed, total = run_all_tests()  
  
    print(f"\n{'=' * 50}")  
    print(f"  测试结果: {passed}/{total} 通过")  
    if passed == total:  
        print("  Day 2 全部通过! [OK]")  
    else:  
        print(f"  [FAIL] {total - passed} 个失败，请检查")  
    print("=" * 50)  
  
  
if __name__ == "__main__":  
    main()
```