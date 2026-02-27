# Teable 项目总览

> 本文档梳理 Teable 开源版的整体架构、技术栈与核心模块组成，供二次开发参考。

---

## 一、项目简介

Teable 是一款基于 **PostgreSQL/SQLite** 的开源无代码/低代码数据库前端，核心能力是将关系型数据库以友好的类 Airtable 界面呈现，支持多种视图、公式、关联字段、协作、插件等。  
采用 **AGPL-3.0** 许可（商业版另有条款）。

---

## 二、仓库结构

```
teable/
├── apps/
│   ├── nestjs-backend/   # 后端（NestJS + ShareDB）
│   ├── nextjs-app/       # 前端（Next.js）
│   └── playground/       # 开发/演示沙箱
├── packages/
│   ├── core/             # 字段模型、公式引擎、操作构建器（纯 TS，前后端共用）
│   ├── sdk/              # 前端 SDK（React hooks/组件/context）
│   ├── openapi/          # OpenAPI 类型定义（zod schema + 接口类型）
│   ├── db-main-prisma/   # Prisma schema、迁移文件
│   ├── ui-lib/           # 通用 UI 组件库（基于 shadcn/ui + Radix）
│   ├── formula/          # 公式解析器（ANTLR4 文法 + 语义）
│   ├── common-i18n/      # 国际化资源
│   └── icons/            # 图标库
├── plugins/              # 官方插件（如 Chart）
├── dockers/              # Docker Compose 部署配置
└── scripts/              # 构建/运维脚本
```

---

## 三、技术栈

| 层 | 技术 |
|---|---|
| **前端框架** | Next.js 14（Pages Router） |
| **前端状态** | Jotai + Zustand（局部）|
| **前端 UI** | shadcn/ui、Radix UI、Tailwind CSS |
| **后端框架** | NestJS（Express 平台） |
| **实时协同** | ShareDB（OT 算法）+ WebSocket |
| **数据库** | PostgreSQL（主推）/ SQLite（轻量部署）|
| **ORM** | Prisma |
| **缓存** | Memory / SQLite / Redis（BullMQ 队列） |
| **文件存储** | 本地磁盘 / MinIO / AWS S3 / 阿里云 OSS |
| **AI 集成** | Vercel AI SDK，支持 OpenAI / Anthropic / Google 等多 Provider |
| **邮件** | Nodemailer（SMTP） |
| **可观测性** | Sentry（错误追踪）+ Pino（日志）+ OpenTelemetry（链路追踪）|
| **认证** | Passport.js（本地/JWT/Session/GitHub/Google/OIDC/OAuth2） |
| **包管理** | pnpm Monorepo（Turborepo） |
| **API 文档** | Swagger（NestJS Swagger 模块） |

---

## 四、核心设计原则

1. **"Database as Table"**：每个 Teable 表（Table）对应数据库中一张真实的物理表，记录直接存在该表中，不做 EAV（Entity-Attribute-Value）。
2. **OT 协同**：字段、记录、视图的所有变更通过 ShareDB 的 OT（Operational Transformation）机制实现多端实时同步。
3. **双数据库适配**：通过 `IDbProvider` 接口层，同一业务逻辑可分别生成 PostgreSQL/SQLite 方言的 SQL，降低部署门槛。
4. **插件化扩展**：前后端均提供插件接口（Plugin SDK + Plugin API），可独立开发并注册到平台。
5. **公式引擎独立**：公式解析与执行在 `packages/formula` 和 `packages/core` 中独立维护，前后端共用相同实现。

---

## 五、部署模式

| 模式 | 说明 |
|---|---|
| Standalone | 单容器，SQLite + 本地存储，适合快速试用 |
| Compose（推荐）| PostgreSQL + Redis + MinIO，生产可用 |
| Cluster | 多副本 + Redis PubSub（ShareDB 分布式） |
| Docker Swarm | 企业级水平扩展 |

---

## 六、文档索引

| 文档 | 说明 |
|---|---|
| [02-features.md](./02-features.md) | 功能清单 |
| [03-architecture.md](./03-architecture.md) | 技术架构详述 |
| [04-data-model.md](./04-data-model.md) | 数据存储模型 |
