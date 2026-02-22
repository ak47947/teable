# Teable 自动化功能 API 接口设计

> **文档版本**: v1.0  
> **创建日期**: 2026-02-05  
> **基础路径**: `/api/base/{baseId}/automation`

---

## 📋 目录

1. [工作流管理 (Workflows)](#1-工作流管理)
2. [执行历史 (Executions)](#2-执行历史)
3. [元数据查询 (Metadata)](#3-元数据查询)
4. [错误代码定义](#4-错误代码定义)

---

## 1. 工作流管理 (Workflows)

### 1.1 创建工作流
- **Endpoint**: `POST /workflow`
- **Payload**:
```json
{
  "name": "高优先级任务通知",
  "description": "当任务优先级设为高时发送通知",
  "tableId": "tbl_xxx",
  "trigger": {
    "type": "field_changed",
    "config": {
      "fieldId": "fld_priority"
    }
  },
  "conditions": {
    "operator": "and",
    "conditions": [
      {
        "fieldId": "fld_priority",
        "operator": "equals",
        "value": "High"
      }
    ]
  },
  "actions": [
    {
      "type": "send_notification",
      "config": {
        "userIds": ["{{record.assignee}}"],
        "message": "新任务: {{record.title}}"
      },
      "order": 1
    }
  ]
}
```

### 1.2 获取工作流列表
- **Endpoint**: `GET /workflow`
- **Query**: `tableId=xxx&enabled=true`
- **Response**: `AutomationWorkflow[]`

### 1.3 更新工作流
- **Endpoint**: `PATCH /workflow/{id}`
- **Payload**: `Partial<AutomationWorkflow>`

---

## 2. 执行历史 (Executions)

### 2.1 获取列表
- **Endpoint**: `GET /workflow/{id}/execution`
- **Query**: `status=FAILED&skip=0&take=20`
- **Response**:
```json
{
  "data": [
    {
       "id": "exe_xxx",
       "status": "SUCCESS",
       "startTime": "2026-02-05T10:00:00Z",
       "duration": 120
    }
  ],
  "total": 1200
}
```

### 2.2 获取日志详情
- **Endpoint**: `GET /execution/{id}/log`

---

## 3. 元数据查询 (Metadata)

用于前端动态渲染配置表单。

### 3.1 获取所有可用的触发器
- **Endpoint**: `GET /metadata/triggers`
- **Response**:
```json
[
  {
    "type": "record_created",
    "name": "记录创建时",
    "configSchema": { ... }
  }
]
```

### 3.2 获取所有可用的动作
- **Endpoint**: `GET /metadata/actions`

---

## 4. 错误代码定义

| 错误代码 | 说明 |
|---------|------|
| `INVALID_WORKFLOW_CONFIG` | 工作流配置校验失败(例如 Trigger 缺失) |
| `CIRCULAR_DEPENDENCY` | 检测到潜在的循环触发风险 |
| `EXECUTION_QUOTA_EXCEEDED` | 达到每小时执行限额 |
| `CONDITION_EVAL_ERROR` | 条件表达式解析错误 |

---

**文档结束**
