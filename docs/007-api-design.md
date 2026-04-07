# NOVA 后端 API 设计

## 一、设计目标

NOVA 后端 API 是整个平台的治理与编排骨架，主要负责：

- 服务接入与配置管理
- `nova.yml` 校验与默认值补全
- 工作流入口文件生成
- Jira 联动
- 审批管理
- 部署记录与查询
- 回滚与服务锁定
- 报表与统计数据输出

API 设计应满足以下原则：

- 面向资源建模
- 兼顾平台治理和执行回调
- 支持 WebUI、CLI、AI Agent 共用
- 保持字段与 `nova.yml` 统一
- 便于审计和扩展

## 二、API 分层

建议按功能拆分为以下几类 API：

1. 服务管理 API
2. 配置校验 API
3. Workflow 生成 API
4. Jira 集成 API
5. 审批 API
6. 部署与流水线 API
7. 服务锁定 API
8. 报表 API
9. 平台内部回调 API

## 三、统一约定

### 1. 路径前缀

建议统一使用：

```text
/api/v1
```

### 2. 统一响应结构

成功响应建议：

```json
{
  "code": "OK",
  "message": "success",
  "data": {}
}
```

失败响应建议：

```json
{
  "code": "NOVA_VALIDATION_ERROR",
  "message": "nova.yml 校验失败",
  "errors": [
    {
      "field": "spec.runtime.language",
      "message": "仅支持 nodejs、python、go"
    }
  ]
}
```

### 3. 认证建议

一期建议至少支持：

- WebUI 登录态调用
- CLI Token 调用
- Workflow 回调使用平台签名 Token

## 四、服务管理 API

### 1. 创建服务

```text
POST /api/v1/services
```

#### 请求体示例

```json
{
  "metadata": {
    "name": "iplocator",
    "project": "app",
    "repo": "app/iplocator"
  },
  "spec": {
    "runtime": {
      "language": "nodejs",
      "version": "20"
    },
    "build": {
      "dockerfile": "Dockerfile",
      "enableBuild": true,
      "buildCommand": "npm run build"
    },
    "test": {
      "unit": {
        "enabled": true,
        "command": "npm run test:unit"
      },
      "integration": {
        "enabled": true,
        "command": "npm run test:integration"
      },
      "e2e": {
        "enabled": true,
        "command": "npm run test:e2e"
      },
      "coverage": {
        "enabled": true,
        "threshold": 90
      }
    },
    "resources": {
      "network": {
        "port": 3000,
        "domain": "iplocator.internal.example.com"
      }
    },
    "integrations": {
      "jira": {
        "projectKey": "APP"
      }
    }
  }
}
```

#### 返回内容

- 服务 ID
- 当前配置快照
- 是否已生成 workflow
- 是否存在待审批资源变更

### 2. 查询服务列表

```text
GET /api/v1/services
```

支持过滤条件：

- `project`
- `language`
- `repo`
- `locked`
- `env`
- `keyword`

### 3. 查询单个服务详情

```text
GET /api/v1/services/{serviceId}
```

返回：

- 服务基础信息
- 当前配置
- 当前锁定状态
- 最近部署记录
- 最近审批记录
- 最近关联 Jira 记录

### 4. 更新服务配置

```text
PUT /api/v1/services/{serviceId}
```

功能：

- 更新服务元信息
- 更新 `nova.yml` 对应配置
- 触发重新校验
- 按需重新生成 workflow

### 5. 删除服务（可选，谨慎）

```text
DELETE /api/v1/services/{serviceId}
```

一期建议先不真正删除，改为：

- 标记停用
- 不再参与自动部署

## 五、配置校验 API

### 1. 校验配置

```text
POST /api/v1/config/validate
```

用于：

- WebUI 表单预校验
- CLI 校验本地配置
- AI Agent 提前检查

#### 请求体

- 原始 `nova.yml` 对应 JSON
- 或 YAML 文本

#### 返回

- 校验结果
- 自动补全后的配置
- 错误列表
- 警告列表

### 2. 渲染最终配置

```text
POST /api/v1/config/render
```

用于：

- 将简化输入渲染成完整配置
- 返回补默认值后的标准对象
- 用于 WebUI 右侧 YAML 预览

## 六、Workflow 生成 API

### 1. 生成入口 workflow

```text
POST /api/v1/services/{serviceId}/workflow/generate
```

功能：

- 根据服务配置选择语言模板
- 生成 `.gitea/workflows/ci.yml`
- 写入目标仓库
- 返回生成结果与模板版本

### 2. 查看当前 workflow 状态

```text
GET /api/v1/services/{serviceId}/workflow
```

返回：

- 当前模板类型
- 模板版本
- 上次生成时间
- 最近一次写入仓库结果

### 3. 手动重新生成 workflow

```text
POST /api/v1/services/{serviceId}/workflow/regenerate
```

用于：

- 配置变更后重新生成
- 平台模板升级后重新下发

## 七、Jira 集成 API

### 1. 查询 Jira 绑定信息

```text
GET /api/v1/services/{serviceId}/jira
```

返回：

- Jira 项目 Key
- 是否启用分支解析
- 最近关联 Issue
- 最近状态回写记录

### 2. 测试 Jira 连通性

```text
POST /api/v1/integrations/jira/test
```

用于：

- 检查 Jira Token 是否有效
- 检查项目 Key 是否存在

### 3. 手动同步 Issue 状态

```text
POST /api/v1/jira/issues/{issueKey}/sync
```

用于人工触发同步或重试。

## 八、审批 API

### 1. 查询审批列表

```text
GET /api/v1/approvals
```

支持过滤：

- `status`
- `serviceId`
- `env`
- `approver`
- `createdAtFrom`
- `createdAtTo`

### 2. 查询审批详情

```text
GET /api/v1/approvals/{approvalId}
```

返回：

- 服务信息
- 版本信息
- 镜像 Tag
- 环境
- 审批状态
- 审批人
- 审批意见
- 超时时间

### 3. 批准审批

```text
POST /api/v1/approvals/{approvalId}/approve
```

### 4. 拒绝审批

```text
POST /api/v1/approvals/{approvalId}/reject
```

#### 请求体示例

```json
{
  "comment": "本次变更风险较高，请补充验证后重新提交"
}
```

### 5. 邮件审批回调接口

```text
GET /api/v1/approval-actions/{token}
```

说明：

- 邮件中的“批准/拒绝”链接使用带签名 token 的 URL
- 打开链接后执行审批动作或引导到确认页

## 九、部署与流水线 API

### 1. 查询流水线记录

```text
GET /api/v1/pipeline-runs
```

支持过滤：

- `serviceId`
- `status`
- `branch`
- `env`
- `triggeredBy`
- `startedAtFrom`
- `startedAtTo`

### 2. 查询单次流水线详情

```text
GET /api/v1/pipeline-runs/{runId}
```

返回：

- 基础信息
- 关联仓库
- 关联分支
- 关联 Jira Issue
- 各阶段结果
- 镜像信息
- 部署结果

### 3. 查询部署记录

```text
GET /api/v1/deployments
```

支持过滤：

- `serviceId`
- `env`
- `status`
- `version`
- `rollbacked`

### 4. 查询单次部署详情

```text
GET /api/v1/deployments/{deploymentId}
```

返回：

- 服务信息
- 环境
- 镜像版本
- 触发流水线
- 审批信息
- 健康检查结果
- 是否发生回滚

## 十、服务锁定 API

### 1. 查询锁定状态

```text
GET /api/v1/services/{serviceId}/lock
```

### 2. 手动锁定服务

```text
POST /api/v1/services/{serviceId}/lock
```

#### 请求体示例

```json
{
  "reason": "生产问题待排查",
  "jiraIssueKey": "APP-999"
}
```

### 3. 解锁服务

```text
POST /api/v1/services/{serviceId}/unlock
```

#### 请求体示例

```json
{
  "jiraIssueKey": "APP-999",
  "comment": "问题已修复，允许继续部署"
}
```

建议规则：

- 若解锁操作关联 Jira Issue，则��台可校验其状态是否已完成或已关闭

## 十一、报表 API

### 1. 查询周报摘要

```text
GET /api/v1/reports/weekly
```

### 2. 查询服务维度统计

```text
GET /api/v1/reports/services/{serviceId}
```

### 3. 查询平台总体指标

```text
GET /api/v1/reports/summary
```

建议输出指标包括：

- 构建次数
- 成功率
- 平均耗时
- 部署次数
- 回滚次数
- 被锁定服务数
- 覆盖率趋势
- Sonar 问题趋势

## 十二、平台内部回调 API

这类 API 主要供 workflow 调用。

### 1. 流水线开始回调

```text
POST /api/v1/internal/pipeline-runs/start
```

### 2. 流水线阶段更新回调

```text
POST /api/v1/internal/pipeline-runs/{runId}/stages
```

### 3. 流水线结束回调

```text
POST /api/v1/internal/pipeline-runs/{runId}/complete
```

### 4. 部署结果回调

```text
POST /api/v1/internal/deployments/{deploymentId}/complete
```

### 5. 回滚事件回调

```text
POST /api/v1/internal/rollbacks
```

### 6. Jira 同步事件回调（可选）

```text
POST /api/v1/internal/jira/events
```

## 十三、Webhook API

### 1. Gitea Webhook（后续可选）

```text
POST /api/v1/webhooks/gitea
```

### 2. Jira Webhook（后续扩展）

```text
POST /api/v1/webhooks/jira
```

一期可以先不重点依赖 webhook，而以 workflow 主动回调平台为主。

## 十四、权限建议

### 开发者

- 可查看服务
- 可创建服务
- 可编辑自己负责的服务配置
- 可查看部署和流水线记录
- 不可解锁服务
- 不可修改平台治理项

### 运维

- 可查看全部服务
- 可审批生产发布
- 可锁定/解锁服务
- 可处理资源变更
- 可查看全局报表

### 平台管理员

- 可管理模板版本
- 可管理 Jira 全局配置
- 可管理默认值与治理策略
- 可查看和操作全部资源

## 十五、总结

NOVA API 不只是一个“配置 CRUD 接口集合”，而是整个平台治理能力的正式边界。

设计时应优先保证：

- 资源模型稳定
- 与 `nova.yml` 一致
- 便于 workflow 回调
- 便于 WebUI、CLI、AI Agent 共用
- 能够支撑后续平台演进
