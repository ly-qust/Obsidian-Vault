## 问题一：
为什么 `ApplicationController` 不应该自己 `new ApplicationServiceImpl`？Spring 容器到底替它做了什么？

如果 Controller 自己 `new ServiceImpl`，就会直接依赖具体实现，耦合高、难以替换和测试，还可能绕过 Spring 提供的事务代理等能力。Spring 容器负责扫描组件、创建 Bean、管理依赖关系，并将 Service 注入 Controller。Controller 只负责使用，不负责创建。


## REST模块
1. 解决什么：用统一的 HTTP 规则描述资源和业务操作。
2. 没有它：URL、请求方式、参数和响应格式混乱，前后端难协作。
3. Spring Boot：通过 `@RestController`、`@RequestMapping`、`@GetMapping`、`@PostMapping`、`@RequestBody` 等接收请求并返回 JSON。
4. SmartRenew：
    - `POST /api/v1/applications`：请求体创建申请
    - `POST /api/v1/applications/{id}/submit`：路径参数提交审核
    - 返回 `Result<T>` 统一响应

面试表达：
> REST 接口通过 HTTP 方法、URL、参数、请求体和状态码明确表达一次操作；Controller 负责把 HTTP 请求转换为业务调用，再把结果转换为统一响应。

## 接口
### 接口三件套记忆法：路径、请求、响应

```
路径参数
→ 放在 URL 里
→ @PathVariable

请求体
→ 客户端发给后端
→ @RequestBody

响应体
→ 后端返回给客户端
→ Result<T>
```

例如：

```
POST /api/v1/applications/123/submit
```

```
123
→ 路径参数 id

请求体
→ 无

响应
→ Result<ApplicationCreateResponse>
```

`Result<T>` 记忆：

```
Result<T>
├── code：结果码
├── message：提示信息
└── data：业务数据 T
```

对应 JSON：

```
{
  "code": 200,
  "message": "提交成功",
  "data": {
    "id": 123,
    "status": "SUBMITTED"
  }
}
```

一句口诀：

> **参数在路径，数据在请求；结果套 Result，业务内容放 data。**


## 第一阶段速记卡：Spring Web 主链

### 1. DI：谁创建对象？

```
Spring 扫描组件
→ 创建 Bean
→ 管理依赖
→ 构造器注入
→ Controller → Service → Mapper
```

问：为什么 Controller 不自己 `new Service`？

> 自己 `new` 会依赖具体实现、耦合高、难测试，还可能绕过 `@Transactional` 等 Spring 代理能力。

关键词：

```
Spring 容器 / Bean / 构造器注入 / 解耦 / 方便测试
```

注意：

- 不叫“对象池”，叫 Spring 容器。
- 单构造器通常不用写 `@Autowired`。
- Mapper 通常是 MyBatis 创建的代理对象。

---

### 2. REST：一次请求由什么组成？

```
HTTP 方法 + URL + 参数/请求体 + 状态码 + 响应 JSON
```

SmartRenew 示例：

```
POST /api/v1/applications
请求体：ApplicationCreateRequest
响应：Result<ApplicationCreateResponse>
```

```
POST /api/v1/applications/{id}/submit
路径参数：id
请求体：无
响应：Result<ApplicationCreateResponse>
```

易错点：

> 创建申请有 DTO 请求体；提交审核只有路径参数，没有请求体。

---

### 3. 三层职责

|层|一句话职责|不应该做什么|
|---|---|---|
|Controller|管 HTTP 协议|不写核心业务|
|Service|管业务规则和事务|不拼 HTTP 响应|
|Mapper|管数据库访问|不判断业务状态|
|MySQL|保存事实、约束数据|不代替完整业务流程|

记忆：

```
Controller 管协议
Service 管业务
Mapper 管访问
MySQL 管事实与约束
```

---

### 4. 真实接口链

```
POST /api/v1/applications/{id}/submit
→ ApplicationController
→ authService.currentUser() 取得 userId
→ applicationService.submitForReview(userId, applicationId)
→ ApplicationMapper
→ MySQL
```

Service 核心流程：

```
锁定申请
→ 检查存在
→ 检查归属
→ 检查状态
→ 校验材料
→ 更新为 SUBMITTED
→ 写 Outbox
→ 构造响应
```

易错点：

> 根据 `applicationId` 查询的是申请，不是活动。

---

### 5. Validation：参数怎么被拦截？

```
JSON
→ DTO
→ @Valid
→ 字段约束
→ 校验失败异常
→ GlobalExceptionHandler
→ HTTP 400
```

真实 DTO：

```
campaignId  → @NotNull
regionCode  → @NotBlank
itemPrice   → @NotNull + 必须大于0
```

问：`itemPrice = 0` 时 Service 执行吗？

> 不执行。Spring MVC 在调用 Controller 方法前完成参数绑定和校验。

---

### 6. 校验应该放哪层？

```
非空、格式、长度、数值范围
→ DTO Validation

资格、库存、状态、归属
→ Service

唯一性、外键、最终数据正确性
→ MySQL 约束
```

例子：

|规则|层|
|---|---|
|地区编码不能为空|DTO|
|商品价格必须大于 0|DTO|
|用户是否符合活动资格|Service|
|只有草稿才能提交|Service|
|同一用户不能重复申请|Service 提示 + 数据库唯一约束兜底|

注意：

> 最终防重复靠数据库唯一约束，不是 Mapper。

---

### 7. 全局异常处理

```
Service
→ throw BizException
→ GlobalExceptionHandler
→ 映射 HTTP 状态
→ Result.fail
→ ResponseEntity
→ 调用方
```

例如：

```
APPLICATION_409
→ HTTP 409
→ 统一错误 JSON
```

问：为什么 Controller 不都写 `try/catch`？

> 重复代码多、响应不统一、职责混乱、容易漏记日志。

---

### 8. 常见状态码：排查入口

```
400 → 参数或请求格式
401 → 未认证、Token 无效
403 → 已认证但无权限
409 → 业务状态或数据冲突
500 → 服务内部异常
```

只是排查入口，不是绝对结论。

---

### 9. 日志定位

```
HTTP 响应 traceId
        =
后端日志 traceId
```

今天真实验证：

```
itemPrice = 0
→ HTTP 400
→ 参数校验失败
→ Controller 方法体未执行
→ 日志出现 Validation failed
```

---

### 10. 今天真实定位的启动 Bug

```
ClassNotFoundException
→ 先看最底层 Caused by
→ 检查 pom.xml
→ commons-lang3 被设为 test 作用域
→ 正式运行没有该依赖
```

修复判断：

```
删除 <scope>test</scope>
→ 依赖进入运行时 classpath
→ 重新构建
→ 启动成功
→ 8080 监听
→ HTTP 请求验证
```

面试总结句：

> 我定位问题时会先判断失败发生在 Security、参数绑定、Controller、Service 还是数据库层，再结合 HTTP 状态码、统一错误结构和 traceId 查日志，最后通过真实请求验证修改是否有效。