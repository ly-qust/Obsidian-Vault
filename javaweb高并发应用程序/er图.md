可以，下面我直接根据你这套表结构，给你整理一个 **ER 图（Mermaid 版）**，你可以直接复制到支持 Mermaid 的编辑器里查看，比如：

- Markdown 编辑器
    
- Typora
    
- Obsidian
    
- Mermaid Live Editor
    
- 部分 IDEA 插件
    

---

## ER 图（Mermaid）

```mermaid
erDiagram

    SYS_USER {
        BIGINT id PK
        VARCHAR username UK
        VARCHAR password_hash
        VARCHAR nickname
        VARCHAR mobile UK
        VARCHAR email
        VARCHAR status
        DATETIME create_time
        DATETIME update_time
    }

    SYS_ROLE {
        BIGINT id PK
        VARCHAR role_code UK
        VARCHAR role_name
        VARCHAR status
        DATETIME create_time
        DATETIME update_time
    }

    SYS_USER_ROLE {
        BIGINT id PK
        BIGINT user_id FK
        BIGINT role_id FK
        DATETIME create_time
        DATETIME update_time
    }

    CAMPAIGN {
        BIGINT id PK
        VARCHAR campaign_code UK
        VARCHAR campaign_name
        VARCHAR category
        VARCHAR status
        DATETIME start_time
        DATETIME end_time
        DECIMAL total_budget
        VARCHAR description
        DATETIME create_time
        DATETIME update_time
    }

    RULE_VERSION {
        BIGINT id PK
        BIGINT campaign_id FK
        INT version_no
        VARCHAR category
        VARCHAR status
        JSON rule_content
        DATETIME effective_time
        DATETIME expire_time
        DATETIME create_time
        DATETIME update_time
    }

    QUOTA_POOL {
        BIGINT id PK
        BIGINT campaign_id FK
        VARCHAR category
        DECIMAL total_amount
        DECIMAL available_amount
        DECIMAL frozen_amount
        DECIMAL used_amount
        VARCHAR status
        BIGINT version
        DATETIME create_time
        DATETIME update_time
    }

    QUOTA_LOG {
        BIGINT id PK
        BIGINT quota_pool_id FK
        BIGINT campaign_id FK
        BIGINT application_id FK
        VARCHAR biz_no
        VARCHAR operation_type
        DECIMAL change_amount
        DECIMAL before_amount
        DECIMAL after_amount
        BIGINT operator_id
        VARCHAR remark
        DATETIME create_time
        DATETIME update_time
    }

    APPLICATION {
        BIGINT id PK
        VARCHAR application_no UK
        BIGINT campaign_id FK
        BIGINT user_id FK
        VARCHAR category
        VARCHAR status
        BIGINT current_review_task_id
        BIGINT rule_version_id FK
        DECIMAL requested_amount
        DECIMAL approved_amount
        DATETIME submit_time
        DATETIME complete_time
        JSON ext_info
        DATETIME create_time
        DATETIME update_time
    }

    APPLICATION_MATERIAL {
        BIGINT id PK
        BIGINT application_id FK
        VARCHAR material_type
        VARCHAR file_url
        VARCHAR file_name
        VARCHAR file_hash
        VARCHAR status
        DATETIME create_time
        DATETIME update_time
    }

    REVIEW_TASK {
        BIGINT id PK
        VARCHAR task_no UK
        BIGINT application_id FK
        BIGINT campaign_id FK
        BIGINT user_id FK
        VARCHAR status
        BIGINT current_assignee_id FK
        VARCHAR review_result
        INT priority
        DATETIME lock_time
        DATETIME finish_time
        BIGINT version
        DATETIME create_time
        DATETIME update_time
    }

    REVIEW_LOG {
        BIGINT id PK
        BIGINT review_task_id FK
        BIGINT application_id FK
        BIGINT operator_id FK
        VARCHAR action
        VARCHAR before_status
        VARCHAR after_status
        VARCHAR remark
        DATETIME create_time
        DATETIME update_time
    }

    VOUCHER {
        BIGINT id PK
        VARCHAR voucher_code UK
        BIGINT campaign_id FK
        BIGINT application_id FK
        BIGINT user_id FK
        BIGINT quota_pool_id FK
        VARCHAR category
        VARCHAR status
        DECIMAL amount
        DATETIME issue_time
        DATETIME expire_time
        DATETIME write_off_time
        BIGINT version
        DATETIME create_time
        DATETIME update_time
    }

    MERCHANT {
        BIGINT id PK
        VARCHAR merchant_code UK
        VARCHAR merchant_name
        VARCHAR category
        VARCHAR contact_name
        VARCHAR contact_phone
        VARCHAR status
        DATETIME create_time
        DATETIME update_time
    }

    WRITE_OFF {
        BIGINT id PK
        VARCHAR write_off_no UK
        BIGINT voucher_id FK
        BIGINT merchant_id FK
        BIGINT campaign_id FK
        BIGINT user_id FK
        VARCHAR status
        DECIMAL write_off_amount
        DATETIME write_off_time
        DATETIME create_time
        DATETIME update_time
    }

    MQ_INBOX {
        BIGINT id PK
        VARCHAR message_key UK
        VARCHAR topic
        VARCHAR consumer_group
        VARCHAR biz_type
        VARCHAR biz_key
        VARCHAR status
        DATETIME consumed_time
        DATETIME create_time
        DATETIME update_time
    }

    AI_CHAT_LOG {
        BIGINT id PK
        VARCHAR biz_type
        BIGINT biz_id
        BIGINT user_id FK
        TEXT prompt_text
        TEXT response_text
        VARCHAR model_name
        VARCHAR status
        VARCHAR trace_id
        DATETIME create_time
        DATETIME update_time
    }

    OPERATION_LOG {
        BIGINT id PK
        VARCHAR biz_type
        BIGINT biz_id
        BIGINT operator_id FK
        VARCHAR operator_name
        VARCHAR action
        VARCHAR category
        VARCHAR status
        VARCHAR trace_id
        JSON detail_json
        DATETIME create_time
        DATETIME update_time
    }

    SYS_USER ||--o{ SYS_USER_ROLE : has
    SYS_ROLE ||--o{ SYS_USER_ROLE : grants

    CAMPAIGN ||--o{ RULE_VERSION : owns
    CAMPAIGN ||--o{ QUOTA_POOL : owns
    CAMPAIGN ||--o{ APPLICATION : receives
    CAMPAIGN ||--o{ REVIEW_TASK : generates
    CAMPAIGN ||--o{ VOUCHER : issues
    CAMPAIGN ||--o{ WRITE_OFF : belongs
    CAMPAIGN ||--o{ QUOTA_LOG : records

    RULE_VERSION ||--o{ APPLICATION : matched_by

    SYS_USER ||--o{ APPLICATION : submits
    APPLICATION ||--o{ APPLICATION_MATERIAL : contains
    APPLICATION ||--|| REVIEW_TASK : creates
    APPLICATION ||--o{ REVIEW_LOG : has
    APPLICATION ||--o| VOUCHER : produces
    APPLICATION ||--o{ QUOTA_LOG : relates_to

    SYS_USER ||--o{ REVIEW_TASK : applicant
    SYS_USER ||--o{ REVIEW_TASK : assignee
    REVIEW_TASK ||--o{ REVIEW_LOG : records

    QUOTA_POOL ||--o{ QUOTA_LOG : changes
    QUOTA_POOL ||--o{ VOUCHER : supports

    SYS_USER ||--o{ VOUCHER : owns
    VOUCHER ||--o| WRITE_OFF : used_in

    MERCHANT ||--o{ WRITE_OFF : handles
    SYS_USER ||--o{ WRITE_OFF : performs

    SYS_USER ||--o{ AI_CHAT_LOG : triggers
    SYS_USER ||--o{ OPERATION_LOG : operates
```

---

## 这个 ER 图怎么理解

你这个库其实可以分成 **6 个模块**：

### 1. 权限模块

- `sys_user`
    
- `sys_role`
    
- `sys_user_role`
    

这是标准 RBAC。

---

### 2. 活动与规则模块

- `campaign`
    
- `rule_version`
    

一个活动可以有多个规则版本。

关系：

- `campaign 1 —— n rule_version`
    

---

### 3. 额度模块

- `quota_pool`
    
- `quota_log`
    

一个活动下可以有多个额度池；  
一个额度池对应很多额度变更日志。

关系：

- `campaign 1 —— n quota_pool`
    
- `quota_pool 1 —— n quota_log`
    

---

### 4. 申请审核模块

- `application`
    
- `application_material`
    
- `review_task`
    
- `review_log`
    

关系主线：

- 用户提交申请
    
- 申请上传材料
    
- 系统生成审核任务
    
- 审核过程产生审核日志
    

---

### 5. 券与核销模块

- `voucher`
    
- `merchant`
    
- `write_off`
    

关系主线：

- 申请通过后生成券
    
- 券在商户处核销
    
- 一张券最多对应一次核销
    

---

### 6. 日志与支撑模块

- `mq_inbox`
    
- `ai_chat_log`
    
- `operation_log`
    

这些是偏系统支撑的表。

---

## 你这套业务主链路

从业务角度，核心流程可以概括成：

```text
用户
 → 申请 application
 → 上传材料 application_material
 → 命中规则 rule_version
 → 进入审核任务 review_task
 → 产生审核日志 review_log
 → 审核通过后生成 voucher
 → 占用/变更 quota_pool 与 quota_log
 → 在 merchant 处 write_off 核销
```

---

## 我再给你一个“更适合答辩/PPT”的简化版 ER 主图

如果你觉得上面那个太复杂，答辩时可以用这个简化版：

```mermaid
erDiagram
    SYS_USER ||--o{ APPLICATION : submits
    CAMPAIGN ||--o{ APPLICATION : belongs
    CAMPAIGN ||--o{ RULE_VERSION : has
    CAMPAIGN ||--o{ QUOTA_POOL : has
    APPLICATION ||--o{ APPLICATION_MATERIAL : has
    APPLICATION ||--|| REVIEW_TASK : creates
    REVIEW_TASK ||--o{ REVIEW_LOG : records
    APPLICATION ||--o| VOUCHER : generates
    QUOTA_POOL ||--o{ QUOTA_LOG : records
    QUOTA_POOL ||--o{ VOUCHER : supports
    VOUCHER ||--o| WRITE_OFF : writeoff
    MERCHANT ||--o{ WRITE_OFF : handles
```

这个版本更适合：

- 课程设计答辩
    
- 项目汇报 PPT
    
- 系统设计说明书
    

---

## 建议你下一步这样做

你这套表已经比较完整了，但我建议你再补 3 个东西：

### 1. 给状态字段做枚举说明

比如：

- `application.status`
    
- `review_task.status`
    
- `voucher.status`
    
- `write_off.status`
    

否则后面写代码容易混乱。

---

### 2. 补充逻辑外键说明

例如：

- `application.current_review_task_id` 现在没有外键约束
    
- 这属于“逻辑关联”，最好在设计文档里说明
    

---

### 3. 再出一张“业务流程图”

ER 图说明的是表关系，老师/面试官通常还会看：

- 用户申请流程图
    
- 审核流转图
    
- 发券核销流程图
    

这样你的“智焕项目”就更完整了。

---

如果你要，我下一条可以直接继续给你输出这 3 个中的任意一个：

1. **PlantUML 版 ER 图**
    
2. **适合论文/课程设计文档的数据库设计说明**
    
3. **申请→审核→发券→核销 的业务流程图**