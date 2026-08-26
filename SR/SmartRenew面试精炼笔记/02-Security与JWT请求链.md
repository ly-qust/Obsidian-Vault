---
tags: [SmartRenew, Spring-Security, JWT, 权限, 面试]
---

# Security 与 JWT 请求链

[[01-项目定位与业务主链|上一篇：项目主链]] · [[00-SmartRenew项目知识地图|知识地图]] · [[03-申请提交与Outbox-Inbox|下一篇：申请与消息]]

## 1. 必须分开的两条链

### 登录并签发 Token

```text
POST /api/auth/login
→ Controller校验请求参数
→ 查询用户
→ PasswordEncoder校验密码哈希
→ 查询角色
→ JWT写入userId、username、roles并签名
→ 返回Token
```

### 携带 Token 访问受保护接口

```text
请求进入SecurityFilterChain
→ JwtFilter提取Bearer Token
→ 校验签名、签发方、过期时间
→ 从Claims恢复用户和角色
→ 创建Authentication
→ 写入SecurityContext
→ URL角色授权
→ Controller
→ Service做资源归属校验
```

> [!warning] 易错纠正
> 过滤器在应用启动时加入过滤链，不是“提取 Token 后再注册”。`SecurityContext` 保存的是 `Authentication`；权限位于 `Authentication.authorities`。JWT 认证通过也不代表自动拥有任意业务数据的访问权。

## 2. 项目代码证据

```text
security/SecurityConfig.java
  - STATELESS
  - /api/auth/login、register放行
  - /api/v1/admin/**要求ADMIN
  - JwtFilter放在UsernamePasswordAuthenticationFilter之前

security/JwtFilter.java
  - 提取Authorization: Bearer ...
  - JwtTokenProvider.parseClaims
  - 创建AuthenticatedUser与Authentication
  - 写入SecurityContextHolder

auth/service/impl/AuthServiceImpl.java
  - login校验密码并签发JWT
  - currentUser从SecurityContext取得当前用户
```

## 3. 认证、接口授权、数据授权

| 层次 | 回答的问题 | 项目实现 |
|---|---|---|
| 认证 | 你是谁 | JWT → Authentication |
| 接口授权 | 你能否进入这类接口 | SecurityConfig 的角色规则 |
| 数据授权 | 这条数据是不是你的 | Service 按 userId/merchantId 查询或校验 |

例如普通用户即使已登录，也不能读取用户 2 的申请；商户也只能查看与当前商户绑定的核销记录。

## 4. 401 与 403

| 状态 | 含义 | 例子 |
|---|---|---|
| 401 | 未建立有效身份 | 无 Token、Token 非法或过期 |
| 403 | 已认证但权限不足 | 普通用户访问管理员接口 |

认证或授权失败时通常终止请求并返回错误，不会继续进入业务 Controller。

## 5. 四类接口测试

```text
1. 未登录访问受保护接口 → 401
2. USER访问ADMIN接口 → 403
3. 必填参数为空/格式错误 → 400类业务响应
4. 用户1访问用户2资源 → 被Service资源归属校验拒绝
```

当前学习状态：原理和代码链已梳理；四类真实请求尚未亲自运行，不能在面试中表述成“已经完成集成测试”。

## 6. 60 秒回答

> 项目把认证和授权分成两段。登录接口校验用户名与密码哈希，查询角色后签发 JWT。后续请求先经过 SecurityFilterChain，JwtFilter 校验 Token 并把包含用户与 authorities 的 Authentication 写入 SecurityContext。Security 配置负责 URL 级角色授权，进入 Controller 后，Service 还要用当前 userId 或 merchantId 做资源归属校验。没有有效身份返回 401，已有身份但角色不足返回 403。

