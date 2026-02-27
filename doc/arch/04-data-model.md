# Teable 数据存储模型

> 基于 `packages/db-main-prisma/prisma/template.prisma` 梳理，同时说明"元数据层"与"记录数据层"的双层存储设计。

---

## 一、核心设计：双层存储

Teable 的数据存储分为两层：

| 层 | 位置 | 内容 |
|---|---|---|
| **元数据层** | Prisma 管理的固定表 | Space / Base / Table / Field / View 等结构描述 |
| **记录数据层** | 动态创建的用户表 | 每个 Table 对应一张数据库物理表，按需创建 |

这是与 Airtable 这类"一切 EAV"方案的本质区别：**用户数据直接存在原生 SQL 表中，查询性能等同于直接操作数据库**。

---

## 二、元数据表 ER 图（主干）

```
Space
  └── Base (space_id)
        └── TableMeta (base_id)           ← 表元数据
              ├── Field (table_id)         ← 字段定义
              ├── View (table_id)          ← 视图定义
              ├── PluginPanel (table_id)   ← 插件面板
              └── PluginContextMenu (table_id)

User
  └── Account (user_id)                   ← 第三方账号绑定
  └── AccessToken (user_id)               ← PAT

Collaborator ─── (resource_type + resource_id → Space|Base)
Invitation   ─── (space_id | base_id)

Plugin
  └── PluginInstall (plugin_id, base_id)

Dashboard (base_id)

ComputedUpdateOutbox ─── ComputedUpdateOutboxSeed
Reference (from_field_id → to_field_id)  ← 字段依赖图
Ops (collection + doc_id + version)      ← OT 操作日志

Trash       ← Space/Base 级回收站
TableTrash  ← 表内字段/视图快照
RecordTrash ← 记录快照

RecordHistory (table_id, record_id, field_id)
Comment (table_id, record_id)
Notification (to_user_id)

Template ── TemplateCategory
Task ── TaskRun
OAuthApp ── OAuthAppSecret ── OAuthAppToken
           └── OAuthAppAuthorized

BaseNode (base_id, parent_id, resource_type, resource_id)  ← 树形导航
BaseNodeFolder (base_id, name)

Integration (resource_id, type)      ← 外部集成配置（AI/其他）
Setting (name, content)              ← KV 全局设置
UserLastVisit                        ← 最近访问记录
Waitlist                             ← 等待名单
```

---

## 三、关键表详解

### 3.1 Space

```sql
id            CUID  PK
name          STRING
credit        INT?         -- 用量配额（云版本）
is_template   BOOL?        -- 是否为模板 Space
deleted_time  DATETIME?    -- 软删除
created_by / last_modified_by
```

### 3.2 Base

```sql
id            CUID  PK
space_id      → Space
name / icon / order
schema_pass   STRING?  -- 数据库连接密码（加密存储）
deleted_time  DATETIME?
```

### 3.3 TableMeta

```sql
id              CUID  PK（同时是动态记录表的表名前缀标识）
base_id         → Base
name / icon / description
db_table_name   STRING   -- 底层物理表名（如 tbl_xxx）
db_view_name    STRING?  -- 关联的 DB 视图名
version         INT      -- OT version
order           FLOAT
deleted_time    DATETIME?
```

> **关键**：`db_table_name` 指向实际存储记录的物理表，Teable 会在 Base 对应的 schema/DB 中动态 CREATE TABLE。

### 3.4 Field

```sql
id                    CUID  PK
table_id              → TableMeta
name / description / type
options               JSON  -- 字段选项（如单选的 choices）
cell_value_type       STRING  -- string/number/boolean/datetime/...
is_multiple_cell_value BOOL
db_field_type         STRING  -- 底层 DB 列类型（text/integer/real/...）
db_field_name         STRING  -- 底层列名（如 fld_xxx）
not_null / unique / is_primary
is_computed           BOOL  -- Formula/Rollup/Lookup 等计算字段
is_lookup             BOOL
lookup_linked_field_id -- 关联的 Link 字段 ID
lookup_options        JSON  -- Lookup 配置
ai_config             JSON  -- AI 字段配置
version / order / deleted_time
```

### 3.5 View

```sql
id              CUID  PK
table_id        → TableMeta
name / type     -- grid/kanban/gallery/calendar/form/list
sort            JSON
filter          JSON
group           JSON
options         JSON  -- 视图特定选项
column_meta     JSON  -- 列宽、隐藏、顺序
is_locked       BOOL
enable_share    BOOL
share_id        STRING UNIQUE  -- 分享 token
share_meta      JSON  -- 分享访问控制（密码/范围）
version / order / deleted_time
```

### 3.6 Ops（OT 操作日志）

```sql
id          CUID
collection  STRING  -- "field" | "view" | "record_xxx"
doc_id      STRING  -- 文档 ID（field_id / view_id / record_id）
doc_type    STRING
version     INT
operation   JSON    -- ShareDB op 内容
created_by
UNIQUE(collection, doc_id, version)
```

所有结构变更（字段创建/修改、视图配置变更等）都以 Op 形式追加写入，支持历史重放。

### 3.7 Reference（字段依赖边）

```sql
from_field_id  → Field  -- 依赖方（Formula 中引用了 toField）
to_field_id    → Field  -- 被依赖方（变化时触发 from 重算）
UNIQUE(to_field_id, from_field_id)
```

### 3.8 ComputedUpdateOutbox（异步计算队列）

```sql
id                   CUID
base_id / seed_table_id / seed_record_ids
change_type          STRING  -- update/create/delete
steps / edges        JSON   -- 拓扑排序结果
status               STRING  -- pending/running/done/failed
attempts / max_attempts(8)
next_run_at / locked_at / locked_by
plan_hash            STRING  -- 同 plan 去重
affected_table_ids / affected_field_ids  STRING[]
```

---

## 四、记录数据层（动态物理表）

每创建一个 Teable Table，后端会执行：

```sql
CREATE TABLE "tbl_{tableId}" (
  __id       TEXT PRIMARY KEY,   -- CUID，记录 ID
  __version  INTEGER DEFAULT 1,  -- OT 乐观锁版本
  __auto_number BIGINT,          -- 自动编号字段
  __row_default BOOLEAN,         -- 默认行标记
  __created_time  TIMESTAMP,
  __last_modified_time TIMESTAMP,
  __created_by    TEXT,
  __last_modified_by TEXT,

  -- 用户字段（按需 ALTER TABLE ADD COLUMN）
  fld_{fieldId1}  TEXT,
  fld_{fieldId2}  INTEGER,
  ...

  -- Link 字段外键
  __fk_{linkFieldId}  TEXT[],    -- PostgreSQL 数组
)
```

**字段命名规则**：
- 用户字段：`fld_{fieldId}`（避免 name 冲突）
- 系统字段：双下划线前缀 `__`
- Link 外键：`__fk_{linkFieldId}`（存储关联记录 ID 列表）

**Link 关联实现**：
- 多对多：两张表各存一个 Link 字段，互存对方 record ID 数组
- Lookup/Rollup：查询时 JOIN 关联表，或通过计算引擎预计算写入
- 公式计算字段：不占用 DB 列（`is_computed=true`），实时计算

---

## 五、附件存储

```
attachments 表（元数据）:
  token / hash / size / mimetype / path
  width / height（图片）
  thumbnail_path

attachments_table 表（使用关系）:
  attachment_id → attachments
  table_id / record_id / field_id
  name（用户上传时的文件名）
```

物理文件按照存储 Provider 不同，存放在：
- 本地：`.assets/uploads/{hash}`
- MinIO/S3/OSS：`public/` 或 `private/` bucket 下按 hash 分级目录

---

## 六、缓存层

| Provider | 适用场景 |
|---|---|
| **Memory** | 开发/测试，进程内 LRU |
| **SQLite** | 单机部署，文件 KV（默认） |
| **Redis** | 多副本部署，BullMQ 队列 + ShareDB PubSub |

缓存主要用于：
- `PerformanceCacheService`：权限缓存、字段 Map 缓存（减少 DB 查询）
- BullMQ：异步任务队列（计算、邮件发送等）
- ShareDB Redis PubSub：多实例 OT 操作广播

---

## 七、数据迁移

```
packages/db-main-prisma/prisma/
├── template.prisma          # 源模板（含 {{PRISMA_PROVIDER}} 占位符）
├── postgres/
│   └── schema.prisma        # 生成的 PostgreSQL schema
│   └── migrations/          # PostgreSQL 迁移文件（时间戳命名）
└── sqlite/
    └── schema.prisma        # 生成的 SQLite schema
    └── migrations/          # SQLite 迁移文件
```

构建时脚本将 `template.prisma` 中的 `{{PRISMA_PROVIDER}}` 替换为 `postgresql` 或 `sqlite`，生成对应方言的 schema。  
运行 `prisma migrate deploy` 完成初始化或升级。

---

## 八、数据一致性保障

| 机制 | 说明 |
|---|---|
| **OT 版本号** | `TableMeta.version`、`Field.version`、`View.version`，乐观并发控制 |
| **Outbox + DeadLetter** | 计算任务最终一致，失败归档不丢失 |
| **RecordHistory** | 逐字段变更快照，可审计 |
| **Trash 软删除** | 所有删除先进回收站，支持恢复 |
| **Integrity 模块** | 定期检查字段依赖一致性（`features/integrity`）|
| **数据库事务** | Prisma 关键操作封装在事务中 |
