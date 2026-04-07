# NOVA 数据库表结构设计

## 一、设计目标

NOVA 数据库负责存储平台治理与交付过程中的核心元数据，主要包括：

- 服务基础信息
- 配置快照
- 流水线记录
- 部署记录
- 审批记录
- 回滚记录
- 服务锁定状态
- Jira 关联信息
- 报表聚合结果
- 审计日志

数据库的设计目标是：

- 支撑平台的资源管理与查询
- 支撑服务配置版本化
- 支撑部署、审批、回滚的完整追踪
- 支撑周报和趋势统计
- 为后续 CLI / AI Agent 提供统一数据基础

建议使用 PostgreSQL。

## 二、设计原则

1. **核心实体单独建表**
2. **配置快照与运行记录分离**
3. **治理状态可审计**
4. **避免把所有信息塞进单表**
5. **保留 JSON 字段承载部分可扩展数据**
6. **关键对象保留创建人、更新时间、审计字段**

## 三、核心表清单

建议至少包含以下表：

1. `services`
2. `service_configs`
3. `pipeline_runs`
4. `pipeline_run_stages`
5. `deployments`
6. `approvals`
7. `rollbacks`
8. `service_locks`
9. `jira_links`
10. `notifications`
11. `report_snapshots`
12. `audit_logs`

## 四、服务表：services

用于记录服务主实体。

### 字段建议

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | bigint | 主键 |
| `service_key` | varchar | 平台内部唯一标识 |
| `name` | varchar | 服务名 |
| `project` | varchar | 项目标识 |
| `repo` | varchar | 仓库全名 |
| `language` | varchar | 语言 |
| `runtime_version` | varchar | 运行时版本 |
| `owner_team` | varchar | 所属团队 |
| `maintainers` | jsonb | 维护人列表 |
| `status` | varchar | 服务状态，如 active / disabled |
| `created_by` | varchar | 创建人 |
| `created_at` | timestamp | 创建时间 |
| `updated_at` | timestamp | 更新时间 |

### 约束建议

- `service_key` 唯一
- `(project, name)` 唯一
- `repo` 唯一

## 五、服务配置快照表：service_configs

用于记录配置版本，支持变更追踪。

### 字段建议

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | bigint | 主键 |
| `service_id` | bigint | 关联服务 |
| `version` | int | 配置版本号 |
| `source_type` | varchar | 来源，如 webui / cli / api |
| `config_json` | jsonb | 标准化配置对象 |
| `config_yaml` | text | 原始 YAML 文本 |
| `is_active` | bool | 当前生效版本 |
| `created_by` | varchar | 创建人 |
| `created_at` | timestamp | 创建时间 |

### 设计说明

- 每次配置变更生成一条新快照
- `services` 表保留基础字段，复杂配置放 `service_configs`

## 六、流水线运行表：pipeline_runs

记录每次 workflow 的运行。

### 字段建议

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | bigint | 主键 |
| `service_id` | bigint | 关联服务 |
| `run_key` | varchar | 平台内部流水线唯一标识 |
| `external_run_id` | varchar | Gitea 外部运行 ID |
| `trigger_type` | varchar | push / pr / manual |
| `branch` | varchar | 分支名 |
| `commit_sha` | varchar | 提交 SHA |
| `triggered_by` | varchar | 触发人 |
| `jira_issue_key` | varchar | 关联 Jira Issue |
| `env` | varchar | 目标环境 |
| `status` | varchar | running / success / failed / canceled |
| `started_at` | timestamp | 开始时间 |
| `finished_at` | timestamp | 结束时间 |
| `duration_seconds` | int | 耗时 |
| `created_at` | timestamp | 创建时间 |

### 索引建议

- `(service_id, created_at desc)`
- `(status, created_at desc)`
- `(jira_issue_key)`

## 七、流水线阶段表：pipeline_run_stages

记录一次流水线中各阶段执行结果。

### 字段建议

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | bigint | 主键 |
| `pipeline_run_id` | bigint | 关联流水线运行 |
| `stage_name` | varchar | 阶段名 |
| `status` | varchar | pending / running / success / failed / skipped |
| `started_at` | timestamp | 开始时间 |
| `finished_at` | timestamp | 结束时间 |
| `duration_seconds` | int | 耗时 |
| `message` | text | 简要说明 |
| `extra_data` | jsonb | 扩展信息 |

### 阶段示例

- prepare
- install
- build
- unit_test
- integration_test
- e2e_test
- sonar
- image_build
- image_push
- deploy
- health_check
- rollback

## 八、部署记录表：deployments

记录每次部署结果。

### 字段建议

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | bigint | 主键 |
| `service_id` | bigint | 关联服务 |
| `pipeline_run_id` | bigint | 关联流水线 |
| `env` | varchar | 环境 |
| `namespace` | varchar | 目标命名空间 |
| `image` | varchar | 镜像地址 |
| `version_tag` | varchar | 镜像标签 |
| `status` | varchar | pending / success / failed / rollbacked |
| `approval_id` | bigint | 关联审批 |
| `deployed_by` | varchar | 操作触发人 |
| `deployed_at` | timestamp | 部署时间 |
| `health_check_status` | varchar | 健康检查结果 |
| `health_check_detail` | text | 健康检查详情 |
| `created_at` | timestamp | 创建时间 |

## 九、审批表：approvals

记录生产审批。

### 字段建议

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | bigint | 主键 |
| `service_id` | bigint | 关联服务 |
| `deployment_id` | bigint | 关联部署 |
| `env` | varchar | 环境 |
| `status` | varchar | pending / approved / rejected / expired |
| `approver` | varchar | 审批人 |
| `approval_token` | varchar | 邮件审批 token |
| `comment` | text | 审批意见 |
| `requested_at` | timestamp | 发起时间 |
| `approved_at` | timestamp | 审批时间 |
| `expired_at` | timestamp | 过期时间 |
| `created_at` | timestamp | 创建时间 |

## 十、回滚记录表：rollbacks

记录自动或手动回滚。

### 字段建议

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | bigint | 主键 |
| `service_id` | bigint | 关联服务 |
| `deployment_id` | bigint | 关联失败部署 |
| `env` | varchar | 环境 |
| `trigger_type` | varchar | auto / manual |
| `reason` | text | 回滚原因 |
| `from_image` | varchar | 回滚前镜像 |
| `to_image` | varchar | 回滚后镜像 |
| `jira_bug_key` | varchar | 自动创建的 Bug Issue |
| `status` | varchar | success / failed |
| `created_at` | timestamp | 创建时间 |

## 十一、服务锁定表：service_locks

用于记录服务当前是否被锁定。

### 字段建议

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | bigint | 主键 |
| `service_id` | bigint | 关联服务 |
| `is_locked` | bool | 是否锁定 |
| `reason` | text | 锁定原因 |
| `source_type` | varchar | auto / manual |
| `jira_issue_key` | varchar | 关联 Jira Issue |
| `locked_by` | varchar | 锁定人或系统 |
| `locked_at` | timestamp | 锁定时间 |
| `unlocked_by` | varchar | 解锁人 |
| `unlocked_at` | timestamp | 解锁时间 |
| `created_at` | timestamp | 创建时间 |
| `updated_at` | timestamp | 更新时间 |

### 设计建议

- 也可以拆为“当前状态表 + 历史表”
- 一期可先用单表保存最新状态和最近变更信息

## 十二、Jira 关联表：jira_links

记录服务、流水线、部署与 Jira 的关联关系。

### 字段建议

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | bigint | 主键 |
| `service_id` | bigint | 关联服务 |
| `pipeline_run_id` | bigint | 可空 |
| `deployment_id` | bigint | 可空 |
| `issue_key` | varchar | Jira Issue Key |
| `issue_type` | varchar | task / bug / story 等 |
| `relation_type` | varchar | source / rollback_bug / linked |
| `created_at` | timestamp | 创建时间 |

## 十三、通知记录表：notifications

记录邮件或其他通知的发送情况。

### 字段建议

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | bigint | 主键 |
| `service_id` | bigint | 关联服务 |
| `event_type` | varchar | approval / deploy_success / deploy_failed / rollback / weekly_report |
| `channel` | varchar | email |
| `recipient` | varchar | 接收者 |
| `subject` | varchar | 主题 |
| `status` | varchar | pending / success / failed |
| `error_message` | text | 失败信息 |
| `sent_at` | timestamp | 发送时间 |
| `created_at` | timestamp | 创建时间 |

## 十四、报表快照表：report_snapshots

用于按时间周期存储统计结果，避免每次实时全量计算。

### 字段建议

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | bigint | 主键 |
| `report_type` | varchar | weekly / monthly / summary |
| `report_key` | varchar | 如 `2026-W15` |
| `data_json` | jsonb | 报表内容 |
| `generated_at` | timestamp | 生成时间 |
| `created_at` | timestamp | 创建时间 |

## 十五、审计日志表：audit_logs

记录关键操作审计。

### 字段建议

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | bigint | 主键 |
| `actor` | varchar | 操作人 |
| `actor_type` | varchar | user / system / workflow |
| `action` | varchar | create_service / update_config / approve / reject / lock / unlock |
| `resource_type` | varchar | service / approval / deployment / workflow |
| `resource_id` | varchar | 资源 ID |
| `detail_json` | jsonb | 操作详情 |
| `created_at` | timestamp | 创建时间 |

## 十六、表关系建议

### 主要关系

- `services` 1:N `service_configs`
- `services` 1:N `pipeline_runs`
- `pipeline_runs` 1:N `pipeline_run_stages`
- `pipeline_runs` 1:N `deployments`
- `deployments` 1:1 或 1:N `approvals`
- `deployments` 1:N `rollbacks`
- `services` 1:1 `service_locks`
- `services` 1:N `jira_links`
- `services` 1:N `notifications`

## 十七、建议的枚举值

### 服务状态

- `active`
- `disabled`

### 流水线状态

- `pending`
- `running`
- `success`
- `failed`
- `canceled`

### 部署状态

- `pending`
- `success`
- `failed`
- `rollbacked`

### 审批状态

- `pending`
- `approved`
- `rejected`
- `expired`

### 锁定来源

- `auto`
- `manual`

## 十八、一期实现建议

一期不需要一次性把所有表做满。

建议优先实现：

1. `services`
2. `service_configs`
3. `pipeline_runs`
4. `deployments`
5. `approvals`
6. `rollbacks`
7. `service_locks`
8. `audit_logs`

等统计和通知完善后，再补充：

- `notifications`
- `report_snapshots`
- 更细粒度的 `jira_links`

## 十九、总结

NOVA 的数据库不是单纯“存配置”，而是承载整个平台治理状态与交付历史的核心基座。

设计时必须优先考虑：

- 配置可追踪
- 执行可追踪
- 审批可追踪
- 回滚可追踪
- 锁定可追踪
- 报表可聚合
