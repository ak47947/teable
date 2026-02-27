# Teable 技术架构详述

---

## 一、整体架构图

```
┌───────────────────────────────────────────────────────┐
│                     浏览器客户端                        │
│   Next.js（Pages Router）                              │
│   ┌────────┐ ┌────────┐ ┌────────┐ ┌──────────┐       │
│   │ Space  │ │  Base  │ │ Table  │ │ View/Form│       │
│   └────────┘ └────────┘ └────────┘ └──────────┘       │
│   SDK（React hooks + Jotai stores + ShareDB client）   │
└───────────────────────┬───────────────────────────────┘
                HTTP REST API / WebSocket（OT）
┌───────────────────────▼───────────────────────────────┐
│                 NestJS Backend                         │
│  ┌──────────┐ ┌──────────┐ ┌─────────────────────┐   │
│  │ REST API │ │ ShareDB  │ │  Calculation Engine  │   │
│  │(OpenAPI) │ │(OT/WS)   │ │  (Field DAG + Outbox)│   │
│  └──────────┘ └──────────┘ └─────────────────────┘   │
│  ┌──────────┐ ┌──────────┐ ┌──────────────────────┐  │
│  │  Auth    │ │  AI Svc  │ │  Storage Adapter     │  │
│  │(Passport)│ │(AI SDK)  │ │  Local/Minio/S3/OSS  │  │
│  └──────────┘ └──────────┘ └──────────────────────┘  │
│                 IDbProvider（SQL 抽象层）               │
└──────────────┬────────────────────────────────────────┘
               │
      ┌────────▼────────┐      ┌──────────────┐
      │ PostgreSQL / SQLite │   │  Redis        │
      │  (Prisma ORM)   │      │  BullMQ队列   │
      └─────────────────┘      │  ShareDB Pub  │
                               └──────────────┘
```

---

## 二、后端架构（NestJS）

### 2.1 模块组织

后端采用 NestJS 模块体系，按 **功能域** 拆分（`src/features/`），共 47 个子模块：

```
features/
├── auth/            # 认证（Passport 策略 + 权限服务）
├── record/          # 记录 CRUD、查询构建、TypeCast
├── field/           # 字段 CRUD、字段模型实例化
├── view/            # 视图 CRUD、视图服务
├── table/           # 表元数据 CRUD
├── base/            # Base CRUD
├── space/           # Space CRUD
├── calculation/     # 计算引擎（DAG、Link、Batch）
├── aggregation/     # 聚合查询
├── share/           # 视图分享
├── collaborator/    # 协作者管理
├── invitation/      # 邀请流程
├── notification/    # 站内通知
├── access-token/    # PAT 管理
├── oauth/           # OAuth 2.0 Server
├── ai/              # AI 服务（多 Provider）
├── chat/            # AI 对话
├── attachment/      # 文件上传/下载/签名 URL
├── import/ export/  # CSV/Excel 导入导出
├── plugin/          # 插件管理 + 官方 Chart 插件
├── dashboard/       # Dashboard 管理
├── comment/         # 记录评论
├── trash/           # 回收站
├── undo-redo/       # 撤销重做
├── template/        # 模板管理
├── setting/         # 系统设置
├── integrity/       # 数据完整性检查
└── ...
```

### 2.2 请求处理链

```
HTTP Request
  → Helmet / CORS / Body Parser（中间件）
  → Auth Guard（JWT / Session / AccessToken / Anonymous）
  → Permission Guard（resource × action）
  → Controller
  → Service（业务逻辑 + Prisma + IDbProvider）
  → Response
```

### 2.3 实时协同（ShareDB）

```
WebSocket 连接
  → ShareDbWebSocketServer（认证 + 路由）
  → ShareDbAdapter（后端 DB 操作适配器）
    ├── submit()    ← 接收 op，写入 ops 表，触发计算引擎
    ├── getOps()    ← 返回历史 ops（用于客户端追赶）
    └── get()       ← 返回 doc 当前快照
  → Redis PubSub（多副本广播，可选）
```

- `ops` 表记录每个 doc（field/view/record）的所有操作历史
- 客户端基于 `sharejs/sharedb-client` 订阅 doc 变更
- `packages/core/src/op-builder/` 提供操作构建工具

### 2.4 数据库抽象层（IDbProvider）

```typescript
interface IDbProvider {
  // 查询构建
  filterQuery(queryBuilder, fieldMap, filter): Knex.QueryBuilder
  sortQuery(queryBuilder, fieldMap, sort): Knex.QueryBuilder
  groupQuery(queryBuilder, fieldMap, group): Knex.QueryBuilder
  searchQuery(queryBuilder, fieldMap, search): Knex.QueryBuilder
  aggregationQuery(queryBuilder, fieldMap, aggregation): Knex.QueryBuilder
  
  // DDL
  createColumn(tableId, field): Promise<void>
  alterColumn(tableId, field): Promise<void>
  dropColumn(tableId, fieldId): Promise<void>
  
  // 辅助
  duplicateTable(fromId, toId): Promise<void>
}
```

- `postgres.provider.ts`（~32K）实现 PostgreSQL 方言
- `sqlite.provider.ts`（~25K）实现 SQLite 方言
- 运行时通过 PRISMA_DATABASE_URL 的 scheme 注入正确实现

---

## 三、计算引擎

### 3.1 核心问题

Teable 的字段（尤其 Formula / Lookup / Rollup / Link）存在**跨表、跨记录的依赖关系**。  
当某记录的字段值变化，需要按拓扑顺序级联重算所有受影响字段。

### 3.2 依赖图

```
Record 变更
  → reference 表（字段依赖边）
  → 拓扑排序（DAG 遍历）
  → Link 字段：裂变为多个 recordId
  → 批量计算受影响记录的所有字段
```

- `reference` 表：存储 `(fromFieldId, toFieldId)` 依赖边
- `link.service.ts`（~67K）：处理关联字段的复杂裂变逻辑
- `reference.service.ts`：维护 reference 表的增删

### 3.3 异步 OutBox 模式

为避免大规模数据更新阻塞请求，采用 OutBox 队列：

```
写操作 → 记录到 computed_update_outbox
Worker   → 轮询 outbox → 执行计算 → 通过 ShareDB 广播变更
失败     → 重试（最多 8 次，指数退避）→ 超限 → dead_letter 表
```

---

## 四、前端架构（Next.js）

### 4.1 页面路由

```
pages/
├── space/[spaceId]/     # Space 首页
├── base/[baseId]/       # Base 内视图
│   ├── [tableId]/       # 表格视图（Grid/Kanban/...）
│   └── dashboard/       # 仪表盘
├── share/[shareId]/     # 公开分享视图
├── auth/                # 登录/注册/重置密码
├── setting/             # 个人设置
├── admin/               # 管理员后台
└── developer/           # 开发者 API Token 管理
```

### 4.2 状态管理

| 层 | 技术 | 说明 |
|---|---|---|
| 服务端数据 | SWR（`@tanstack/react-query` 部分） | API 数据缓存与同步 |
| 实时数据 | ShareDB Doc（`useDoc` hook） | 字段/视图/记录 OT 状态 |
| 全局 UI | Jotai atoms | 侧边栏、选中行、过滤器等 |
| 路由参数 | Next.js router | tableId、viewId、baseId |

### 4.3 视图渲染

```
<View>
  ├── <GridView>      → 虚拟滚动表格（canvas 渲染 @teable/grid）
  ├── <KanbanView>    → 按 singleSelect 分组拖拽看板
  ├── <GalleryView>   → 卡片网格
  ├── <CalendarView>  → 月/周日历
  ├── <FormView>      → 表单收集
  └── <ListView>      → 简单列表
```

Grid 视图使用 Canvas 渲染（非 DOM），支持百万行级别虚拟滚动。

### 4.4 SDK（`packages/sdk`）

前端 SDK 提供：
- **React Context**：`TableProvider`、`ViewProvider`、`FieldsProvider`、`RecordProvider`
- **Hooks**：`useTable`、`useView`、`useFields`、`useRecords`、`useAggregation` 等
- **Model 层**：`Table`、`Field`、`View`、`Record` 类（封装 ShareDB doc）
- **Plugin Bridge**：插件 iframe ↔ 主应用 PostMessage 通信

---

## 五、认证体系

```
策略（Passport.js）
├── local           # 邮箱/密码
├── session         # Cookie Session
├── jwt             # JWT Bearer Token
├── access-token    # PAT（Personal Access Token）
├── github          # OAuth GitHub（可选）
├── google          # OAuth Google（可选）
├── oidc            # OIDC 通用（可选）
└── anonymous       # 公开分享访问
```

**权限系统**：`permission.service.ts` 维护完整的 `(resourceType, action) → roles` 矩阵，  
Guard 在请求时根据当前用户在资源上的角色做拦截。

---

## 六、AI 架构

```
AiService
├── getModelInstance(modelKey)
│   ├── 解析 modelKey: type@model@name
│   ├── AI Gateway（统一代理）→ createGateway()
│   └── 标准 Provider（openai/anthropic/google/...）
├── generateStream()    → SSE 流式输出
├── generateText()      → 同步文本生成
└── getAIConfig(baseId) → Space 级覆盖 + 实例级配置
```

**配置层级**：实例级（Admin Setting） > Space 级（Integration） > 默认

---

## 七、文件存储架构

```
AttachmentsService
├── StorageAdapter（策略模式）
│   ├── LocalStorage   → 磁盘路径
│   ├── MinioStorage   → MinIO SDK
│   ├── S3Storage      → AWS S3 SDK
│   └── AliyunStorage  → 阿里云 OSS
├── 上传流程：
│   客户端 → 请求签名 URL → 直传到存储 → 回调通知后端 → 记录 attachments 表
└── 私有文件：
│   访问时生成签名 URL（HMAC，有效期 6 天）
└── 加密选项：AES-128-CBC（可配置）
```

---

## 八、可观测性

| 组件 | 功能 |
|---|---|
| **Pino Logger** | 结构化日志，支持 JSON 格式 |
| **Sentry** | 前后端错误聚合、性能采样 |
| **OpenTelemetry** | 链路追踪（`instrument.ts`）|
| **Health Check** | `/api/health`，检查 DB/Redis/存储连通性 |
| **Swagger** | 自动生成 API 文档（`/api-doc`） |
