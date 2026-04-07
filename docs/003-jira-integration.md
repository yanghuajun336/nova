# NOVA 与 Jira 联动设计

## 一、设计目标

Jira 与 NOVA 的联动目标不是简单地“能跳转”，而是建立一条贯穿研发交付全过程的统一追踪链路。

核心设计原则是：

> 以 Jira Issue ID 作为代码、流水线、部署、回滚、审批、通知的统一追踪键。

这样可以实现：

- 需求与代码分支关联
- 代码与流水线关联
- 流水线与部署关联
- 部署与回滚关联
- 回滚与 Bug 创建关联

## 二、总体联动方向

Jira 与 NOVA 的联动包括两个方向：

### 1. NOVA → Jira

由平台或 workflow 调用 Jira API，完成：

- 状态回写
- 评论回写
- 部署结果通知
- 自动创建 Bug Issue

### 2. Jira → NOVA

后续可通过 Jira Webhook 实现：

- 任务状态变化后触发平台动作
- 自动生成分支
- 自动同步某些交付动作

一期建议优先完成 **NOVA → Jira**。

## 三、Issue ID 作为链路主键

建议统一使用分支命名规则，将 Jira Issue ID 直接放入分支名中：

```text
feature/APP-123-add-iplocator
hotfix/APP-456-fix-timeout
```

NOVA 可以通过正则解析：

```text
[A-Z]+-\d+
```

提取 Issue ID，例如：

- `APP-123`
- `PAY-456`

该 Issue ID 将作为以下动作的关联键：

- 流水线运行记录
- 部署记录
- 审批记录
- 回滚记录
- 邮件通知
- Jira 评论
- Jira Bug 自动创建

## 四、Jira 状态映射策略

已知前提：

- Jira 各项目状态统一
- 当前状态包括：
  - 待办
  - 进行中
  - 已完成
  - 部分项目存在“暂停状态”

因此建议将状态映射设计为 **平台全局配置**，而不是每个服务单独配置。

### 建议平台内部状态映射

| 平台事件 | Jira 状态 |
|---|---|
| 开发开始 / 首次 Push | 进行中 |
| CI / 构建中 | 进行中 |
| dev/staging 部署成功 | 进行中 |
| 生产部署成功 | 已完成 |
| 发布失败 / 回滚 | 进行中 |
| 人工暂停处理 | 暂停状态（如项目支持） |

说明：

- 由于 Jira 状态较少，很多流水线内部状态不一定直接映射到 Jira 工作流状态
- 更细粒度的信息建议通过 **评论** 来体现，而不是状态本身

## 五、评论回写设计

建议在以下关键节点向 Jira 自动写入评论：

### 1. 流水线开始

示例：

```text
NOVA 自动通知：

已检测到与该 Issue 关联的代码提交，流水线已开始执行。

服务：iplocator
仓库：app/iplocator
分支：feature/APP-123-add-iplocator
触发人：yanghuajun336
```

### 2. CI 失败

示例：

```text
NOVA 自动通知：

流水线执行失败。

服务：iplocator
阶段：单元测试
结果：失败
建议：请查看 Gitea 流水线日志定位问题
```

### 3. Staging 部署成功

示例：

```text
NOVA 自动通知：

服务已成功部署至 Staging 环境。

服务：iplocator
环境：staging
镜像：harbor.nova.internal/app/iplocator:1.0.0-a1b2c3d-20260407
```

### 4. 生产审批中

示例：

```text
NOVA 自动通知：

该服务已进入生产发布审批阶段，审批通过后将执行生产部署。

服务：iplocator
环境：prod
审批状态：待审批
```

### 5. 生产部署成功

示例：

```text
NOVA 自动通知：

服务已成功部署至生产环境。

服务：iplocator
环境：prod
结果：成功
镜像：harbor.nova.internal/app/iplocator:1.0.0-a1b2c3d-20260407
```

### 6. 自动回滚触发

示例：

```text
NOVA 自动通知：

生产部署失败，已自动回滚至上一版本。

服务：iplocator
环境：prod
失败原因：健康检查未通过
回滚版本：harbor.nova.internal/app/iplocator:0.9.9-f9e8d7c-20260401
```

## 六、生产发布成功后的状态回写

建议生产部署成功后，将关联 Jira Issue 状态更新为：

```text
已完成
```

这样 Jira 能体现出任务已经完成交付。

## 七、自动回滚时的 Jira 联动

自动回滚场景是 Jira 联动设计中的重点。

### 流程

1. 生产部署失败
2. NOVA 自动回滚至上一版本
3. 在原 Issue 下写入失败与回滚评论
4. 自动创建一个新的 Jira Bug Issue
5. 将服务在 NOVA 中标记为锁定状态
6. 运维或负责人处理完 Bug 后，手动解除锁定

### 自动创建 Bug Issue 的建议字段

- Issue 类型：Bug
- 标题：`[自动回滚] iplocator 生产环境部署失败`
- 描述包含：
  - 原关联 Issue ID
  - 服务名
  - 失败时间
  - 部署镜像版本
  - 回滚目标版本
  - 失败原因
  - 日志入口
  - 流水线入口

### 与原 Issue 的关系

建议在 Bug 描述中记录来源 Issue，例如：

```text
来源交付 Issue：APP-123
```

后续如有需要，也可以进一步利用 Jira 的 issue link 机制建立“关联缺陷”。

## 八、Jira 与分支策略结合

推荐强制约束以下分支格式：

```text
feature/{JIRA-ISSUE}-{description}
hotfix/{JIRA-ISSUE}-{description}
```

例如：

```text
feature/APP-123-add-iplocator
hotfix/APP-456-fix-timeout
```

如果分支不符合规范，NOVA 可以：

- 拒绝进入完整流水线
- 或仅执行基础校验，不允许进入部署阶段
- 在日志中明确提示规范要求

## 九、Jira API 集成建议

由于使用的是 Jira Cloud，建议采用官方 REST API。

### 认证方式

- 使用 Jira API Token
- 由 NOVA 平台统一保管
- 不直接暴露给项目仓库或业务 workflow 配置

### 一期建议能力

- 更新 Issue 状态
- 添加评论
- 创建 Bug Issue
- 查询项目 Key 是否存在

## 十、一期实现边界

一期建议实现以下能力：

- 从分支名解析 Jira Issue ID
- 流水线关键节点评论回写
- 生产部署成功后状态回写为“已完成”
- 自动回滚时创建 Bug
- 平台统一维护 Jira 状态映射

一期暂不强制实现：

- Jira Webhook 反向触发 NOVA
- 自动创建开发分支
- 自动关闭所有子任务
- 复杂工作流状态流转引擎

## 十一、统一配置建议

由于 Jira 状态在各项目中统一，建议在 NOVA 平台内部配置一份全局映射，而不是放入每个仓库的 `nova.yml` 中。

业务仓库中只保留：

```yaml
integrations:
  jira:
    projectKey: APP
    issueFromBranch: true
```

平台内部再维护：

- 状态映射
- 评论模板
- Bug 创建模板
- Jira API 访问配置

## 十二、总结

Jira 与 NOVA 的联动，不应停留在“流水线结束后发个链接”，而应成为整个平台交付治理的一部分。

通过 Jira Issue ID 贯穿全链路，NOVA 可以把需求、代码、构建、部署、回滚和故障处理连接成一个完整闭环。
