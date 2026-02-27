# Teable 功能清单

> 基于源码（`apps/nestjs-backend/src/features/`、`apps/nextjs-app/src/features/`）梳理，覆盖后端模块与前端页面。

---

## 一、空间与组织管理

| 功能 | 说明 |
|---|---|
| **Space（空间）** | 顶层资源隔离单元，可包含多个 Base |
| **Base（数据库）** | 类似 Airtable 的 Base，包含多个 Table；支持图标、密码保护 |
| **Organization（组织）** | 多租户组织层，可管理多个 Space |
| **Pin（置顶）** | 用户可将常用 Space/Base 置顶，按 order 排序 |
| **BaseNode / 文件夹** | Base 内资源（表、视图等）可按树形结构组织，支持文件夹嵌套 |
| **Template（模板）** | 系统/用户模板，支持分类、发布，可一键克隆为新 Base |
| **Trash（回收站）** | Space/Base/Table 级软删除，支持恢复和永久删除 |

---

## 二、数据表（Table）

| 功能 | 说明 |
|---|---|
| **Table CRUD** | 创建、重命名、删除、排序、复制 |
| **Table Trash** | 表内字段/记录删除历史快照，支持还原 |
| **数据库连接视图** | `db-connection` 功能，可直接查询底层 SQL |
| **ERD 视图** | 展示表间关联关系（Entity Relationship Diagram） |

---

## 三、字段系统（Field）

Teable 支持 **20+ 种字段类型**：

| 类别 | 字段类型 |
|---|---|
| **基础** | 单行文本、长文本、数字、复选框、单选、多选 |
| **日期时间** | 日期（含时区格式化）、创建时间、最后修改时间 |
| **用户** | 创建人、最后修改人、用户（多选） |
| **媒体** | 附件（图片/文件，支持缩略图） |
| **关联** | Link（跨表关联，支持一对多/多对多）、Lookup（引用关联值） |
| **计算** | 公式（Formula）、汇总（Rollup）、条件汇总（ConditionalRollup）、自动编号 |
| **交互** | 按钮（Button，可触发 AI/Automation）、评分（Rating） |

**字段能力**：
- 每个字段有 `options`（字段配置）、`cellValueType`（单值/多值）、`dbFieldType`（底层 DB 类型）
- 支持字段级 `notNull`、`unique` 约束
- 字段变更通过 OT 操作传播，自动触发计算引擎

---

## 四、记录（Record）

| 功能 | 说明 |
|---|---|
| **CRUD** | 增删改查，支持批量操作 |
| **TypeCast** | 粘贴/导入时自动类型转换（字符串 → 日期/数字等） |
| **搜索** | 全文/字段搜索，跨视图 |
| **记录历史** | 字段级变更历史（before/after 快照） |
| **记录注释（Comment）** | 富文本评论、引用回复、表情回应、@订阅通知 |
| **Undo/Redo** | 多步操作撤销重做（基于 OT ops 重放） |
| **Selection** | 批量选中、复制、粘贴、清空 |

---

## 五、视图（View）

支持 **6 种视图类型**：

| 视图 | 说明 |
|---|---|
| **Grid（表格）** | 默认电子表格视图，虚拟滚动，支持行高、列宽 |
| **Gallery（图册）** | 以卡片形式展示，适合图片/媒体类数据 |
| **Kanban（看板）** | 按单选字段分组，支持拖拽 |
| **Calendar（日历）** | 按日期字段展示，周/月视图 |
| **Form（表单）** | 对外收集，支持字段隐藏/必填/说明 |
| **List（列表）** | 简化行列表，适合轻量查阅 |

**视图通用能力**：
- 排序（Sort）、过滤（Filter）、分组（Group）
- 字段显示/隐藏（columnMeta）
- 共享（Share，可公开访问，带访问控制）
- 视图锁定（isLocked）

---

## 六、协作与权限

| 功能 | 说明 |
|---|---|
| **Collaborator** | Space/Base 级别协作者，角色：Owner / Editor / Commenter / Viewer |
| **邀请** | 邀请链接 + 邮件邀请，有过期时间 |
| **通知** | 站内通知（关注评论、@提及、协作变更） |
| **Organization** | 组织级成员管理 |
| **Authority Matrix** | 细粒度权限矩阵（resource × action） |

---

## 七、认证与安全

| 功能 | 说明 |
|---|---|
| **本地账密** | 邮箱 + 密码（可禁用） |
| **社交登录** | GitHub、Google、OIDC（扩展点） |
| **Session / JWT** | 双 Token 体系，支持长期会话 |
| **AccessToken（PAT）** | 个人访问令牌，支持 scope 和资源范围限制 |
| **OAuth 2.0** | 可注册第三方 OAuth 应用，标准授权码流程 |
| **Turnstile** | Cloudflare 人机验证（可选） |

---

## 八、数据导入 / 导出

| 功能 | 说明 |
|---|---|
| **导入** | 支持 CSV、Excel（.xlsx/.xls） |
| **导出** | 按视图过滤/排序导出 CSV/Excel |
| **附件** | 上传到本地/MinIO/S3/OSS，支持私有 URL 签名 |

---

## 九、Dashboard（仪表盘）

- 每个 Base 可创建多个 Dashboard
- Dashboard 通过插件（Chart Plugin 等）渲染可视化组件
- 布局自由拖拽

---

## 十、插件系统（Plugin）

| 组件 | 说明 |
|---|---|
| **Plugin** | 插件元数据注册（名称/图标/URL/位置/状态） |
| **PluginInstall** | 插件在 Base 中的实例（含 storage 配置存储） |
| **PluginPanel** | Table 级插件面板（表格右侧抽屉） |
| **PluginContextMenu** | 记录行右键菜单插件 |
| **官方 Chart 插件** | 内置图表插件（位于 `plugins/chart`），支持柱状图/折线图/饼图等 |

插件通过 PostMessage Bridge 与主应用通信（`packages/sdk/src/plugin-bridge`）。

---

## 十一、AI 功能

| 功能 | 说明 |
|---|---|
| **AI 配置** | 实例级 + Space 级 AI 配置，支持多 Provider |
| **LLM Provider** | OpenAI / Anthropic / Google / Mistral / AI Gateway（统一代理） |
| **AI 字段（ai-config）** | 字段级 AI 配置，如自动填充、按钮触发生成 |
| **Chat（AI 对话）** | 基于表格上下文的 AI 问答 |
| **流式生成** | Server-Sent Events 流式输出 |
| **多模态** | 支持 Vision（图像理解）、PDF 解析、图像生成 |
| **AI Gateway** | 统一 API 代理（`ai-gateway.vercel.sh`） |

---

## 十二、自动化（Automation）

- 前端已有 `automation` 页面入口（`apps/nextjs-app/src/features/app/automation`）
- 目前为预留模块，后端尚在开发中

---

## 十三、计算引擎（Calculation Engine）

| 功能 | 说明 |
|---|---|
| **字段依赖图** | 分析字段间依赖关系，构建有向无环图（DAG） |
| **拓扑排序计算** | 记录变更→触发依赖字段按拓扑顺序重算 |
| **Link 裂变** | 跨表关联字段变更时，正确传播到所有关联记录 |
| **OutBox 模式** | 异步计算任务队列（`ComputedUpdateOutbox`），保证最终一致 |
| **DeadLetter** | 失败任务归档，最多重试 8 次 |

---

## 十四、系统管理（Admin）

| 功能 | 说明 |
|---|---|
| **用户管理** | 列表、停用、永久删除 |
| **全局设置** | SMTP / 存储 / AI / 安全等配置 |
| **模板管理** | 创建、发布、分类管理 |
| **Waitlist** | 早期访问白名单 |
| **健康检查** | `/health` 端点，监控 DB / 存储状态 |
| **可观测性** | Sentry 错误追踪 + OpenTelemetry 链路追踪 |
