# Teable 自动化功能技术架构设计

> **文档版本**: v1.0  
> **创建日期**: 2026-02-05  
> **状态**: 详细设计完成

---

## 📋 目录

1. [系统总体架构](#1-系统总体架构)
2. [数据库设计 (DB Schema)](#2-数据库设计)
3. [后端模块设计](#3-后端模块设计)
4. [触发器系统 (Triggers)](#4-触发器系统)
5. [条件判断系统 (Conditions)](#5-条件判断系统)
6. [动作执行系统 (Actions)](#6-动作执行系统)
7. [执行引擎核心 (Engine)](#7-执行引擎核心)
8. [内部事件流](#8-内部事件流)

---

## 1. 系统总体架构

Teable 自动化系统采用**事件驱动**与**插件化**架构。

- **Event-Driven**: 通过监听 Teable 内部的记录操作事件(创建/更新/删除)来驱动工作流。
- **Pluggable**: 触发器、条件和动作均作为插件注册到系统,便于横向扩展。
- **Async Execution**: 基于消息队列 (BullMQ) 异步执行,确保不阻塞核心业务流程。

---

## 2. 数据库设计

基于 Prisma 的模型设计。

### 2.1 Workflow 模型
存储自动化的逻辑配置。

```prisma
model AutomationWorkflow {
  id          String    @id @default(uuid())
  baseId      String
  tableId     String?
  name        String
  description String?
  enabled     Boolean   @default(true)
  
  // 核心逻辑存储为 Json
  trigger     Json      // { type: string, config: any }
  conditions  Json?     // { operator: 'and'|'or', conditions: any[] }
  actions     Json      // Array<{ type: string, config: any, order: number }>
  
  // 执行配置
  config      Json?     // { maxRetries: number, concurrency: number, etc. }
  
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
  deletedAt   DateTime?
}
```

### 2.2 Execution 模型
记录每一次工作流的运行情况。

```prisma
model AutomationExecution {
  id              String           @id @default(uuid())
  workflowId      String
  status          ExecutionStatus  @default(PENDING)
  triggerData     Json             // 触发时的原始数据
  startTime       DateTime         @default(now())
  endTime         DateTime?
  duration        Int?             // 毫秒
  errorMessage    String?          
  errorStack      String?
  
  logs            AutomationLog[]
}

enum ExecutionStatus {
  PENDING
  RUNNING
  SUCCESS
  FAILED
  SKIPPED
}
```

---

## 3. 后端模块设计

模块划分 (基于 NestJS):

- **AutomationModule**: 根模块,协调各组件。
- **WorkflowModule**: 负责工作流的管理 (CRUD)。
- **TriggerModule**: 触发器注册中心及事件分发。
- **ConditionModule**: 条件评估器。
- **ActionModule**: 动作执行中心及各个动作插件的实现。
- **ExecutionModule**: 并行执行控制、日志记录、重试逻辑。

---

## 4. 触发器系统 (Triggers)

### 4.1 触发器接口
所有触发器必须实现 `ITrigger` 接口。

```typescript
interface ITrigger {
  type: string;
  name: string;
  onEvent(event: BaseEvent): Promise<TriggerResult | null>;
  validateConfig(config: any): boolean;
  getSchema(): any; // 返回配置的 JSON Schema 用于前端渲染
}
```

### 4.2 常用触发器实现
- **RecordCreatedTrigger**: 过滤 `record.created` 事件。
- **FieldChangedTrigger**: 过滤 `record.updated` 事件,对比指定字段。
- **ScheduleTrigger**: 利用 `node-cron` 基于配置生成定时任务。

---

## 5. 条件判断系统 (Conditions)

### 5.1 条件评估引擎
`ConditionEvaluator` 类负责解析嵌套的逻辑判断。

主要逻辑:
1. 获取当前触发记录的数据快照。
2. 解析配置中的字段 ID,从快照中提取值。
3. 应用操作符 (Operator):
   - 字符串: 精确匹配、包含、前缀、为空、非空。
   - 数值: 等于、大于、小于、范围。
   - 选项: 包含、不包含。
4. 递归处理 `AND` / `OR` 嵌套组。

---

## 6. 动作执行系统 (Actions)

### 6.1 动作接口
实现 `IAction` 接口。

```typescript
interface IAction {
  type: string;
  execute(config: any, context: ExecutionContext): Promise<ActionResult>;
}
```

### 6.2 上下文变量解析
在执行动作前,系统会扫描配置中的变量占位符 `{{...}}`,并从上下文 (`ExecutionContext`) 中注入真实值。

支持的变量路径:
- `{{trigger.recordId}}`
- `{{record.fields.fldXXXXXXXX}}`
- `{{action1.result.id}}` (流水线模式下引用前序动作的结果)

---

## 7. 执行引擎核心 (Engine)

执行逻辑伪代码:

```typescript
async run(workflowId, triggerData) {
  const workflow = await getWorkflow(workflowId);
  const execution = await createExecution(workflowId);
  
  const context = {
    trigger: triggerData,
    record: await fetchCurrentRecord(triggerData),
    outputs: {}
  };

  try {
    // 1. 判断条件
    if (workflow.conditions && !evaluator.eval(workflow.conditions, context)) {
       return skipExecution(execution);
    }
    
    // 2. 顺序执行动作
    for (const action of workflow.actions) {
       const resolvedConfig = variableResolver.resolve(action.config, context);
       const result = await actionRegistry.get(action.type).execute(resolvedConfig);
       context.outputs[action.order] = result;
    }
    
    await finishSuccess(execution);
  } catch (e) {
    await failExecution(execution, e);
  }
}
```

---

## 8. 内部事件流

1. **Service Layer**: 用户修改记录,调用 `RecordService.update`。
2. **EventEmitter**: 更新成功后,`RecordService` 异步发射 `RECORD_UPDATED` 事件。
3. **Trigger Listener**: `AutomationTriggerListener` 捕获该事件。
4. **Registry Lookup**: 查找所有监听此表格且匹配配置的 `Workflow`。
5. **Queue Push**: 将待执行任务推送到 `BullMQ`。
6. **Worker Processing**: 背景 Worker 领取任务,调用 `ExecutionEngine.run`。

---

**文档结束**

> 如有疑问或建议,请联系项目负责人
