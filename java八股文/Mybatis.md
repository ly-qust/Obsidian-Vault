
|对比项|MyBatis|MyBatis-Plus|
|---|---|---|
|定位|SQL 映射框架|MyBatis 增强工具|
|底层关系|基础框架|依赖并增强 MyBatis|
|基础 CRUD|通常自己写|`BaseMapper` 已提供|
|简单动态条件|XML `if/where`|Wrapper 条件构造器|
|复杂 SQL|非常灵活|通常仍回到自定义 XML|
|SQL 可见性|非常直接|Wrapper 生成，需要查看日志|
|结果映射|`resultType/resultMap`|仍可使用 MyBatis 映射|
|分页|自己写或使用插件|内置分页插件体系|
|逻辑删除|自己设计和写 SQL|`@TableLogic` 支持|
|乐观锁|自己写版本条件|乐观锁插件|
|自动填充|自己统一实现|提供字段填充机制|
|代码生成|可结合 Generator|提供代码生成器|
|学习重点|SQL、映射、动态 SQL|Wrapper、通用接口、插件配置|
|开发效率|复杂 SQL 清晰，CRUD 重复较多|标准 CRUD 开发速度快|
|控制能力|开发者直接控制 SQL|简单场景方便，复杂场景仍需 SQL|
|数据库约束|开发者设计|同样必须由开发者设计|
