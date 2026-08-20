
这份审查清单（Checklist）非常具有实战价值，几乎涵盖了后端开发（尤其是对接外部 API 或大模型时）最容易踩坑的点。

为了让你更直观地理解，我针对清单里的每一项，为你准备了**“❌ 错误示范（踩坑版）”**和**“✅ 正确示范（标准版）”**的代码对比。

---

### 🔴 P0 级别（致命错误：会导致服务挂掉、安全事故）

#### 1. 超时必须设置
**场景**：大模型服务器卡住了，如果不设置超时，你的程序就会一直傻等，直到你的服务器连接数耗尽，彻底死机。
*   **❌ 错误示范**：
    ```python
    # 危险：如果网络不通或对面不返回，这行代码会永远卡住（阻塞）
    response = httpx.get("https://api.openai.com/v1/models") 
    ```
*   **✅ 正确示范**：
    ```python
    # 安全：最多等 10 秒，等不到就抛出 TimeoutException，程序可以继续往下走或报错
    response = httpx.get("https://api.openai.com/v1/models", timeout=10.0)
    ```

#### 2. `raise_for_status()` 必须调用
**场景**：API 报错了（比如你没钱了返回 HTTP 402），但程序以为成功了，继续解析 JSON，导致崩溃。
*   **❌ 错误示范**：
    ```python
    response = client.post("/chat", json=data)
    # 如果此时 response.status_code 是 401 (未授权)
    # 下面这行依然会执行，但很可能会报错 JSONDecodeError，因为错误页面可能不是 JSON
    return response.json() 
    ```
*   **✅ 正确示范**：
    ```python
    response = client.post("/chat", json=data)
    # 如果不是 2xx 成功状态码，这里会立刻抛出 HTTPStatusError，及时掐断错误流程
    response.raise_for_status() 
    return response.json()
    ```

#### 3. 密钥不进入日志
**场景**：API Key 被打印到日志系统（如 ELK、阿里云 SLS）中，运维人员或黑客看到了日志，直接盗刷你的钱。
*   **❌ 错误示范**：
    ```python
    api_key = "sk-xxxxxxxxxxxxxx"
    # 绝对禁止！日志系统里直接明文可见你的钱袋子
    logger.info(f"请求大模型，使用的密钥是: {api_key}") 
    ```
*   **✅ 正确示范**：
    ```python
    # 只把密钥放进 headers 里，日志只打印脱敏信息或业务参数
    headers = {"Authorization": f"Bearer {api_key}"}
    logger.info(f"请求大模型，当前用户 ID: {user_id}") 
    ```

#### 4. 不吞异常（`except Exception`）
**场景**：你写错了一个变量名，引发了 `NameError`，结果被宽泛的 `except` 吃掉了，查 Bug 查到怀疑人生。
*   **❌ 错误示范**：
    ```python
    try:
        data = resqonse.json() # 注意这里拼写错了，应该是 response
    except Exception:
        # 糟糕：拼写错误引发的 NameError 被吞了，直接返回空，你根本不知道发生了什么
        return None 
    ```
*   **✅ 正确示范**：
    ```python
    try:
        data = response.json()
    except json.JSONDecodeError as e: # 只捕获你预期的特定异常
        logger.error(f"解析大模型返回失败: {e}")
        return None
    ```

#### 5. async 路由用 AsyncClient
**场景**：在 FastAPI 等异步框架中，错误使用了同步的 `httpx.Client`，导致一个请求卡住了整个服务器的所有请求。
*   **❌ 错误示范**：
    ```python
    @app.get("/chat")
    async def chat_api():
        # 灾难：同步请求会阻塞整个事件循环，其他用户的请求进都进不来！
        response = httpx.get("https://api...") 
        return response.text
    ```
*   **✅ 正确示范**：
    ```python
    @app.get("/chat")
    async def chat_api():
        # 正确：使用异步客户端，遇到网络等待时主动让出 CPU 给其他请求
        async with httpx.AsyncClient() as client:
            response = await client.get("https://api...")
        return response.text
    ```

---

### 🟠 P1 级别（高风险：容易导致线上大面积故障）

#### 1. 异常链保留 (`raise ... from e`)
**场景**：你想封装一下底层的错误，但不小心把原始的报错位置弄丢了，导致报错栈只有最后一行，无法定位根因。
*   **❌ 错误示范**：
    ```python
    try:
        return float("abc")
    except ValueError as e:
        # 丢失线索：Traceback 里只会显示这一行报错，不知道底层的 "abc" 是从哪来的
        raise BusinessError("数据转换失败") 
    ```
*   **✅ 正确示范**：
    ```python
    try:
        return float("abc")
    except ValueError as e:
        # 完美：Traceback 会完整打印两段报错，告诉你是因为 ValueError 导致了 BusinessError
        raise BusinessError("数据转换失败") from e 
    ```

#### 2. 错误码区分可重试/不可重试
**场景**：网络抖动可以重试，但如果提示你“余额不足”，你疯狂重试 100 次也没用，只会白白消耗服务器资源。
*   **✅ 正确示范（伪代码）**：
    ```python
    if error.type == "timeout":
        # 网络超时，可以重试 3 次
        retry(request)
    elif error.type == "invalid_api_key":
        # 密钥错了，不可重试，直接抛出异常中断
        raise SystemExit("密钥错误，请检查配置！")
    ```

#### 3. `trace_id` 贯穿全程
**场景**：每天几百万条日志，某个用户的请求报错了，你怎么把他的几十条相关日志串起来？
*   **✅ 正确示范**：
    ```python
    # 生成全局唯一的追踪 ID (比如 UUID)
    trace_id = "tr-123456"
    logger.info(f"[{trace_id}] 开始处理用户请求")
    try:
        do_something()
    except Exception as e:
        # 报错时也要带上 trace_id，方便顺藤摸瓜找问题
        logger.error(f"[{trace_id}] 处理失败: {e}") 
    ```

#### 4. JSON 解析失败有降级
**场景**：你要求大模型返回 JSON `{"name": "Tom"}`，但大模型偶尔发癫，返回了 `这里是结果：{"name": "Tom"}`，导致 `response.json()` 崩溃。
*   **❌ 错误示范**：
    ```python
    # 大模型一旦不听话返回非格式化文本，程序直接 500 崩溃
    return response.json() 
    ```
*   **✅ 正确示范**：
    ```python
    try:
        return response.json()
    except ValueError:
        # 降级处理：不崩溃，返回一个明确的默认结构，或者通过正则手动提取
        logger.warning("模型返回了非标准 JSON，原样保留")
        return {"error": "parse_failed", "raw_content": response.text}
    ```

---

### 🟡 P2 级别（规范问题：关乎代码好不好维护）

#### 1. 注释解释“为什么”
*   **❌ 错误示范（废话注释）**：
    ```python
    # 调用 chat 方法发请求
    response = self.chat(data) 
    ```
*   **✅ 正确示范（提供上下文）**：
    ```python
    # 大模型 Qwen 处理长文本需要较长耗时，此处单独将此请求的 timeout 放宽至 60 秒
    response = self.chat(data, timeout=60.0)
    ```

#### 2. 函数单一职责
**场景**：不要把“洗菜、切菜、炒菜”全写在一个名叫 `eat()` 的函数里。
*   **❌ 错误示范**：
    ```python
    def chat_with_model(user_id):
        # 1. 查数据库找用户历史记录 (不属于 client 的职责)
        history = db.query(user_id) 
        # 2. 拼接 Prompt
        prompt = f"{history}...请回答"
        # 3. 发网络请求
        res = client.post(prompt)
        # 4. 存数据库 (不属于 client 的职责)
        db.save(res)
    ```
*   **✅ 正确示范**：
    ```python
    # Client 里面的 chat 方法应该非常纯粹，只负责“网络发包”这一件事
    def chat(self, messages: list) -> str:
        res = self._client.post("/chat", json={"messages": messages})
        return res.json()["choices"][0]["message"]["content"]
    ```

总结来说，这套清单是一个成熟的高级工程师踩过无数坑之后总结出来的“护身符”。严格按照这套规范写代码，你的程序线上出 Bug 的概率至少降低 80%。