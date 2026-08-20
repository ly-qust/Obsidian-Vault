

### 1）**WebSocket 相关问题**

- **疑问**：什么是 WebSocket？它和 HTTP 请求有什么区别？为什么要使用 WebSocket？
    
- **解释**：WebSocket 是一种**双向通信协议**，相比于 HTTP，它允许服务器主动推送数据给客户端，而无需客户端不断发起请求（如轮询）。常用于 **实时通信** 的场景，比如在线聊天、实时数据展示等。
    

---

### 2）**为什么没有 Service 层，而是直接在 Controller 中操作数据库？**

- **疑问**：为什么项目中直接在 `UserController` 中操作 `UserMapper` 进行数据库操作，没有使用 Service 层？这是不是不规范？
    
- **解释**：这种设计并不违反 Spring 的规范，实际上它是一种**简化开发的方式**。通过继承 `HttpController`，我们可以通过泛型直接获取通用的 CRUD 方法，减少了 Service 层的编写。但这也使得每个 Controller 的职责过于庞大，实际上这种做法在复杂项目中可能不太推荐，通常会把数据库操作封装到 Service 层，再由 Controller 调用。
    

---

### 3）**`@CrossOrigin` 的作用**

- **疑问**：`@CrossOrigin` 是做什么的？什么时候需要加上？
    
- **解释**：`@CrossOrigin` 是用来解决跨域请求的问题。它允许其他域（例如前端和后端不同域名/端口）访问你的后端接口。通常在 **前后端分离** 的项目中，如果前端和后端不在同一个域名/端口下，需要使用 `@CrossOrigin` 解决浏览器的跨域问题。
    

---

### 4）**`HttpController` 中的 `mapper` 是怎么知道具体 `Mapper`（如 `UserMapper`）的？**

- **疑问**：`HttpController` 中的 `mapper` 怎么知道我具体 `UserMapper` 中的方法？为什么 `mapper.selectCount` 在 `HttpController` 里可以调用，而不需要在 `UserMapper` 中再写？
    
- **解释**：`mapper` 是通过 Spring 的依赖注入和泛型类型的替换来知道具体使用哪个 Mapper（例如 `UserMapper`）。在 `HttpController` 里，我们定义了 `M extends BaseMapper<B>`，这就意味着**任何继承了 `BaseMapper` 的具体 Mapper 都可以被注入并使用**，而 `BaseMapper` 已经为我们提供了 `selectCount`、`selectList` 等通用方法。只要继承了 `BaseMapper`，就可以直接调用。
    

---

### 5）**Mapper 中是否需要定义 `selectCount` 等方法？**

- **疑问**：`selectCount` 这种方法是 `BaseMapper` 提供的，具体 `Mapper` 需要额外定义 `selectCount` 吗？如果其他具体 `Mapper` 的方法不同，是不是就不能用 `HttpController`？
    
- **解释**：不需要在每个 `Mapper` 中重新定义 `selectCount` 这样的通用方法。`BaseMapper` 提供了这些基础方法，任何继承了 `BaseMapper` 的 `Mapper` 都能直接使用这些方法。如果需要自定义查询（例如 `getUser()`），可以在 `Mapper` 中定义并在 `Controller` 中使用。**`HttpController` 的通用 CRUD 方法是基于 `BaseMapper` 的，**如果 `Mapper` 不继承 `BaseMapper`，就无法直接使用 `HttpController` 中的这些通用方法。
    

---

### 6）**`HttpController` 中的 Mapping 和路径匹配问题**

- **疑问**：在 `HttpController` 中使用 `@GetMapping("{id}")` 这样的路径占位符时，为什么它能同时处理分页（例如 `/user/page1`）和删除（例如 `/user/12`）请求？
    
- **解释**：`@GetMapping("{id}")` 使用了路径变量 `id`，它可以匹配任何形如 `/user/{id}` 的请求。通过判断 `id` 的值（是否包含 `page`），代码决定是执行分页还是删除操作。这种设计虽然能工作，但在 RESTful API 中不太规范，通常分页和删除会分开处理，建议将分页和删除的操作路径设计成不同的 URL，避免混淆。
    

---

### 7）**`NotNull` 注解和 `NotNullUtil` 工具类的作用**

- **疑问**：`@NotNull` 注解是怎么工作的？它是如何与 `NotNullUtil` 配合使用，来进行字段的非空验证的？
    
- **解释**：`@NotNull` 注解用于标记字段，表示该字段不能为 `null` 或空字符串。`NotNullUtil` 工具类通过反射扫描对象的字段，检查标记了 `@NotNull` 的字段是否为空。如果为空，它会返回相应的错误提示信息。这个过程完全自动化，不需要手动检查每个字段，可以提高代码的简洁性和可维护性。
    

---

### 8）**`@TableField(exist = false)` 的作用**

- **疑问**：在 `Device` 类中，为什么 `category`、`pin` 等字段使用了 `@TableField(exist = false)` 注解？
    
- **解释**：`@TableField(exist = false)` 注解的作用是告诉 MyBatis-Plus，这些字段**不需要映射到数据库表**。这些字段通常用于**存储从其他表中查询到的数据**，比如联表查询时填充的值。它们不会被持久化到数据库，而只是作为临时字段存在于 Java 对象中。
    

---

### 总结：

- **WebSocket**：用于实现前后端的实时通信，常用于实时更新场景。
    
- **`@CrossOrigin`**：解决跨域问题，允许其他域访问后端接口。
    
- **`HttpController`**：作为公共控制器，提供通用的增删改查方法，子类 Controller 继承它并使用具体的 `Mapper` 进行数据操作。
    
- **`selectCount` 等通用方法**：由 `BaseMapper` 提供，不需要每个 `Mapper` 都写。
    
- **`@NotNull` 注解和 `NotNullUtil`**：用于校验字段不能为空，自动化处理字段校验逻辑。
    
- **`@TableField(exist = false)`**：用于标记类中的字段不与数据库表映射，通常用于存放从数据库外部或其他查询结果中获取的数据。
    
