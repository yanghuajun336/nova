# NOVA 中央 Workflow 模板体系设计

## 一、设计目标

NOVA 的中央 Workflow 模板体系是平台执行层的核心。

它的设计目标包括：

- 支持 Node.js、Python、Go 三种主要语言
- 实现模板集中治理
- 让业务仓库不再手写复杂 CI/CD
- 将语言差异封装在模板内部
- 通过平台自动生成业务仓库入口 workflow
- 为审批、回滚、Jira 联动、邮件通知提供统一执行逻辑

## 二、总体设计思路

建议采用：

> 中央模板仓库 + 业务仓库轻量入口文件

的模式。

### 业务仓库只保留

- 源代码
- Dockerfile
- `nova.yml`
- 平台自动生成的 `.gitea/workflows/ci.yml`

### 中央模板仓库负责

- 多语言 workflow 模板
- 通用脚本
- 部署脚本
- Jira 脚本
- 邮件脚本
- 回滚脚本
- 资源渲染模板

## 三、推荐仓库结构

建议中央模板仓库名为：

```text
devops/nova-pipeline-templates
```

推荐目录结构如下：

```text
devops/nova-pipeline-templates
├── .gitea/
│   └── workflows/
│       ├── nodejs.yml
│       ├── python.yml
│       ├── go.yml
│       ├── deploy-common.yml
│       ├── approval-common.yml
│       └── rollback-common.yml
│
├── scripts/
│   ├── common/
│   ├── nodejs/
│   ├── python/
│   ├── go/
│   ├── sonar/
│   ├── harbor/
│   ├── deploy/
│   ├── jira/
│   └── mail/
│
├── templates/
│   ├── workflow/
│   ├── k8s/
│   └── mail/
│
├── policies/
│   ├── branch_policy.yaml
│   ├── deploy_policy.yaml
│   └── jira_status_mapping.yaml
│
└── docs/
```

## 四、平台与模板的职责边界

### 平台负责

- 解析 `nova.yml`
- 应用默认值
- 校验配置
- 选择语言模板
- 生成业务仓库入口 workflow
- 管理模板版本
- 管理审批与服务锁定
- 记录运行数据和报表

### 模板负责

- checkout 源码
- 执行构建和测试
- 执行 SonarQube 扫描
- 构建和推送镜像
- 执行部署
- 健康检查
- 自动回滚
- 调用 Jira / 邮件 / 平台回调接口

原则：

> 平台做治理与编排，模板做执行与回调。

## 五、业务仓库入口文件设计

业务仓库中的入口 workflow 不由开发者手写，而由 NOVA 平台自动生成。

### 示例

```yaml
name: NOVA CI/CD

on:
  push:
    branches:
      - develop
      - main
      - release/**
      - feature/**
      - hotfix/**
  pull_request:
    branches:
      - main
      - develop

jobs:
  pipeline:
    uses: devops/nova-pipeline-templates/.gitea/workflows/nodejs.yml@main
    with:
      service_name: "iplocator"
      project_name: "app"
      repo_name: "app/iplocator"
      runtime_version: "20"
      dockerfile: "Dockerfile"
      build_context: "."
      enable_build: true
      build_command: "npm run build"
      unit_test_enabled: true
      unit_test_command: "npm run test:unit"
      integration_test_enabled: true
      integration_test_command: "npm run test:integration"
      e2e_test_enabled: true
      e2e_test_command: "npm run test:e2e"
      coverage_enabled: true
      coverage_threshold: 90
      sonar_enabled: true
      jira_project_key: "APP"
      issue_from_branch: true
      service_port: 3000
      service_domain: "iplocator.internal.example.com"
    secrets: inherit
```

## 六、为什么不在模板运行时直接读取 `nova.yml`

理论上模板运行时也可以读取仓库中的 `nova.yml`，但不建议这样做。

建议由平台先完成：

- 配置解析
- 默认值补全
- 合法性校验
- 参数展开

然后生成明确的 `with` 参数传给模板。

这样做的好处：

1. 模板逻辑更简单
2. 错误更早暴露
3. 不依赖 runner 上的额外 YAML 解析工具
4. 参数更直观，方便排查
5. 适合未来 WebUI、CLI、API 统一使用

## 七、模板输入参数设计

建议三种语言模板统一参数名，减少平台复杂度。

### 通用参数

- `service_name`
- `project_name`
- `repo_name`
- `runtime_version`
- `dockerfile`
- `build_context`
- `enable_build`
- `build_command`
- `unit_test_enabled`
- `unit_test_command`
- `integration_test_enabled`
- `integration_test_command`
- `e2e_test_enabled`
- `e2e_test_command`
- `coverage_enabled`
- `coverage_threshold`
- `sonar_enabled`
- `jira_project_key`
- `issue_from_branch`
- `service_port`
- `service_domain`

## 八、主模板内部阶段设计

以语言主模板为例，建议包含以下阶段：

### Stage 1：准备阶段

- checkout 代码
- 分支命名校验
- 解析 Jira Issue ID
- 识别部署环境
- 生成镜像 Tag

### Stage 2：语言环境准备

根据语言安装运行时，安装依赖并做缓存。

### Stage 3：构建与测试

- 条件执行构建
- 条件执行单元测试
- 条件执行接口测试
- 条件执行 E2E 测试
- 校验覆盖率

### Stage 4：质量扫描

- 执行 SonarQube 扫描
- 等待 quality gate
- 不通过则失败

### Stage 5：镜像构建与推送

- 登录 Harbor
- 构建 Docker 镜像
- 推送镜像
- 输出镜像地址

### Stage 6：部署决策

根据触发分支决定：

- Pull Request / feature / hotfix：只跑 CI
- develop：部署到 dev
- release/*：部署到 staging
- main：走生产审批后部署

### Stage 7：部署执行

- 渲染 Deployment / Service / HTTPRoute / PVC
- 应用到 Kubernetes
- 等待 rollout 完成
- 执行健康检查

### Stage 8：回滚处理

仅针对生产环境：

- 部署失败时自动回滚
- 发邮件
- 创建 Jira Bug
- 回调平台锁定服务

### Stage 9：后置动作

- 更新 Jira 状态
- 写 Jira 评论
- 回调 NOVA 平台记录流水线结果
- 发送结果通知

## 九、语言模板差异设计

### Node.js 模板

负责：

- Node 版本设置
- `npm ci` / `pnpm install`
- Node.js 项目构建和测试

### Python 模板

负责：

- Python 版本设置
- `pip install` 或 Poetry 安装
- `pytest` 系列测试命令

### Go 模板

负责：

- Go 版本设置
- `go mod download`
- `go build`
- `go test`

### 统一原则

除语言准备和默认命令建议不同外，其余逻辑尽量保持一致：

- Jira 联动
- SonarQube
- Harbor
- 部署
- 回滚
- 邮件通知
- 平台回调

## 十、分支与环境映射

建议固化为平台规则，不开放业务项目自定义：

| 分支 / 事件 | 环境 | 行为 |
|---|---|---|
| Pull Request | 无 | 仅 CI |
| `feature/*` | 无 | 仅 CI |
| `hotfix/*` | 无 | 仅 CI |
| `develop` | `dev` | 自动部署 |
| `release/*` | `staging` | 自动部署 |
| `main` | `prod` | 审批后部署 |

## 十一、模板版本治理

模板需要有统一的版本治理策略。

### 第一阶段建议

统一使用稳定版本标签，例如：

```yaml
uses: devops/nova-pipeline-templates/.gitea/workflows/nodejs.yml@v1
```

### 后续建议

平台支持：

- 查看每个服务当前使用的模板版本
- 平台批量升级模板版本
- 某些服务按需锁定在旧版本

## 十二、模板参数控制原则

需要特别注意：

> 不要把模板做成一个超级大而全的参数怪物。

第一版建议仅暴露必要参数：

- 服务标识
- 构建命令
- 测试命令
- 覆盖率阈值
- Jira 项目
- 端口 / 域名

复杂策略尽量放在平台内部治理，不要让每个服务自己定制太多。

## 十三、资源生成策略

开发者只声明：

- 端口
- 域名
- 卷信息

平台或模板负责自动生成：

- Deployment
- Service
- HTTPRoute
- PVC

开发者不直接提交复杂 K8s YAML，以降低接入门槛和治理成本。

## 十四、总结

NOVA 中央 Workflow 模板体系的核心价值在于：

- 模板集中治理
- 项目最小接入成本
- 多语言统一抽象
- 平台规则强约束
- 易于后续扩展

它是 NOVA 从“设计理念”走向“可执行平台能力”的关键桥梁。
