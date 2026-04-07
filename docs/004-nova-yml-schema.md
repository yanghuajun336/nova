# nova.yml 配置规范（v1alpha1）

## 一、设计目标

`nova.yml` 是 NOVA 平台与业务仓库之间的核心配置契约。

它的设计目标包括：

- 为服务接入提供统一声明格式
- 为平台生成 workflow 提供输入
- 为 WebUI / CLI / API 提供统一数据模型
- 为后续治理规则提供可校验基础
- 尽量减少开发者需要手写的字段数量

## 二、顶层结构

推荐结构如下：

```yaml
apiVersion: nova.dev/v1alpha1
kind: ServicePipeline
metadata: {}
spec: {}
```

## 三、完整示例

```yaml
apiVersion: nova.dev/v1alpha1
kind: ServicePipeline

metadata:
  name: iplocator
  project: app
  repo: app/iplocator

spec:
  owner:
    team: app
    maintainers:
      - yanghuajun336

  runtime:
    language: nodejs
    version: "20"

  build:
    type: docker
    context: .
    dockerfile: Dockerfile
    enableBuild: true
    buildCommand: npm run build

  test:
    unit:
      enabled: true
      command: npm run test:unit
    integration:
      enabled: true
      command: npm run test:integration
    e2e:
      enabled: true
      command: npm run test:e2e
    coverage:
      enabled: true
      threshold: 90

  quality:
    sonar:
      enabled: true

  deploy:
    namespace:
      project: app
      service: iplocator
    environments:
      - name: dev
        autoDeploy: true
      - name: staging
        autoDeploy: true
      - name: prod
        autoDeploy: false
        approvalRequired: true
        autoRollback: true

  resources:
    network:
      domain: iplocator.internal.example.com
      port: 3000
    storage:
      - name: uploads
        size: 10Gi
        mountPath: /app/uploads

  integrations:
    jira:
      projectKey: APP
      issueFromBranch: true
```

## 四、字段说明

### 1. 顶层字段

| 字段 | 类型 | 必填 | 说明 |
|---|---|---:|---|
| `apiVersion` | string | 是 | 配置版本，当前为 `nova.dev/v1alpha1` |
| `kind` | string | 是 | 配置类型，当前为 `ServicePipeline` |
| `metadata` | object | 是 | 服务元数据 |
| `spec` | object | 是 | 服务配置主体 |

### 2. metadata

| 字段 | 类型 | 必填 | 说明 |
|---|---|---:|---|
| `metadata.name` | string | 是 | 服务名 |
| `metadata.project` | string | 是 | 项目标识 |
| `metadata.repo` | string | 是 | Gitea 仓库名，格式为 `owner/repo` |

### 3. spec.owner

| 字段 | 类型 | 必填 | 说明 |
|---|---|---:|---|
| `spec.owner.team` | string | 否 | 团队名 |
| `spec.owner.maintainers` | array | 否 | 维护人列表 |

### 4. spec.runtime

| 字段 | 类型 | 必填 | 说明 |
|---|---|---:|---|
| `spec.runtime.language` | string | 是 | 技术栈语言，支持 `nodejs` / `python` / `go` |
| `spec.runtime.version` | string | 是 | 运行时版本 |

### 5. spec.build

| 字段 | 类型 | 必填 | 说明 |
|---|---|---:|---|
| `spec.build.type` | string | 否 | 构建类型，第一版固定为 `docker` |
| `spec.build.context` | string | 否 | Docker 构建上下文 |
| `spec.build.dockerfile` | string | 是 | Dockerfile 路径 |
| `spec.build.enableBuild` | bool | 否 | 是否执行构建命令 |
| `spec.build.buildCommand` | string | 否 | 构建命令 |

### 6. spec.test

#### 单元测试

| 字段 | 类型 | 必填 | 说明 |
|---|---|---:|---|
| `spec.test.unit.enabled` | bool | 否 | 是否启用单元测试 |
| `spec.test.unit.command` | string | 条件必填 | 单元测试命令 |

#### 接口测试

| 字段 | 类型 | 必填 | 说明 |
|---|---|---:|---|
| `spec.test.integration.enabled` | bool | 否 | 是否启用接口测试 |
| `spec.test.integration.command` | string | 条件必填 | 接口测试命令 |

#### E2E 测试

| 字段 | 类型 | 必填 | 说明 |
|---|---|---:|---|
| `spec.test.e2e.enabled` | bool | 否 | 是否启用 E2E 测试 |
| `spec.test.e2e.command` | string | 条件必填 | E2E 测试命令 |

#### 覆盖率

| 字段 | 类型 | 必填 | 说明 |
|---|---|---:|---|
| `spec.test.coverage.enabled` | bool | 否 | 是否启用覆盖率检查 |
| `spec.test.coverage.threshold` | int | 否 | 覆盖率阈值，默认 90 |

### 7. spec.quality

| 字段 | 类型 | 必填 | 说明 |
|---|---|---:|---|
| `spec.quality.sonar.enabled` | bool | 否 | 是否启用 SonarQube |

### 8. spec.deploy

#### namespace

| 字段 | 类型 | 必填 | 说明 |
|---|---|---:|---|
| `spec.deploy.namespace.project` | string | 否 | Namespace 中的项目名部分 |
| `spec.deploy.namespace.service` | string | 否 | Namespace 中的服务名部分 |

#### environments

| 字段 | 类型 | 必填 | 说明 |
|---|---|---:|---|
| `name` | string | 是 | 环境名，支持 `dev` / `staging` / `prod` |
| `autoDeploy` | bool | 否 | 是否自动部署 |
| `approvalRequired` | bool | 否 | 是否需要审批 |
| `autoRollback` | bool | 否 | 是否自动回滚 |

### 9. spec.resources

#### network

| 字段 | 类型 | 必填 | 说明 |
|---|---|---:|---|
| `spec.resources.network.domain` | string | 否 | 访问域名 |
| `spec.resources.network.port` | int | 是 | 服务端口 |

#### storage

| 字段 | 类型 | 必填 | 说明 |
|---|---|---:|---|
| `name` | string | 是 | 卷名称 |
| `size` | string | 是 | 卷容量，例如 `10Gi` |
| `mountPath` | string | 是 | 挂载路径 |

### 10. spec.integrations

#### jira

| 字段 | 类型 | 必填 | 说明 |
|---|---|---:|---|
| `spec.integrations.jira.projectKey` | string | 是 | Jira 项目 Key |
| `spec.integrations.jira.issueFromBranch` | bool | 否 | 是否从分支名解析 Issue ID |

## 五、最小示例

```yaml
apiVersion: nova.dev/v1alpha1
kind: ServicePipeline

metadata:
  name: iplocator
  project: app
  repo: app/iplocator

spec:
  runtime:
    language: nodejs
    version: "20"

  build:
    dockerfile: Dockerfile
    enableBuild: true
    buildCommand: npm run build

  test:
    unit:
      enabled: true
      command: npm run test:unit
    integration:
      enabled: true
      command: npm run test:integration
    e2e:
      enabled: true
      command: npm run test:e2e
    coverage:
      enabled: true
      threshold: 90

  deploy:
    environments:
      - name: dev
      - name: staging
      - name: prod

  resources:
    network:
      port: 3000

  integrations:
    jira:
      projectKey: APP
```

## 六、默认值策略

建议由平台补齐以下默认值：

| 字段 | 默认值 |
|---|---|
| `spec.build.type` | `docker` |
| `spec.build.context` | `.` |
| `spec.build.dockerfile` | `Dockerfile` |
| `spec.test.coverage.threshold` | `90` |
| `spec.quality.sonar.enabled` | `true` |
| `dev.autoDeploy` | `true` |
| `staging.autoDeploy` | `true` |
| `prod.approvalRequired` | `true` |
| `prod.autoRollback` | `true` |
| `spec.integrations.jira.issueFromBranch` | `true` |

## 七、平台强校验规则

建议平台至少校验以下内容：

1. `metadata.name` 仅允许小写字母、数字和中划线
2. `metadata.project` 仅允许小写字母、数字和中划线
3. `metadata.repo` 必须符合 `owner/repo` 格式
4. `runtime.language` 必须为 `nodejs`、`python`、`go` 之一
5. 若测试项启用，则对应命令必填
6. 覆盖率阈值必须在 0-100 之间
7. 服务端口必须在 1-65535 之间
8. 卷挂载路径必须以 `/` 开头
9. 若存在 `prod` 环境，则必须开启审批和自动回滚
10. Jira 项目 Key 必须合法

## 八、不建议放入仓库级配置的内容

以下内容建议由平台统一管理，而不是放入 `nova.yml`：

- Jira 状态映射
- 审批链细节
- 邮��接收人策略
- Workflow 模板引用地址
- Harbor 镜像命名规则
- SonarQube 内部项目 Key
- Terraform 细节配置
- 服务锁定状态

## 九、总结

`nova.yml` 的本质不是让项目自己维护一套复杂平台逻辑，而是：

> 用最小配置声明服务需求，用平台默认规则和治理机制完成自动补全。

它是 NOVA 平台化能力的基础输入，也是后续 API、WebUI、CLI、AI Agent 的统一契约。
