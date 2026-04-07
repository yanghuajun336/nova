# NOVA CLI 与 AI Agent Skill 设计

## 一、设计背景

根据当前规划，NOVA 除了 WebUI 之外，还希望提供 CLI 入口，并且这个 CLI 后续会作为 AI Agent 的 Skill 使用。

这意味着 CLI 不应只是“给人用的命令行工具”，还应具备以下特点：

- 命令结构稳定
- 输出格式可机器解析
- 参数设计与 API 统一
- 能与平台配置契约保持一致
- 适合作为 Agent 的调用接口

因此，CLI 需要在一期就从“人机双用”的角度来设计。

## 二、设计目标

NOVA CLI 的目标包括：

- 支持服务接入与配置校验
- 支持生成与查看 `nova.yml`
- 支持查询服务、部署、审批、锁定状态
- 支持部分运维操作
- 为 AI Agent 提供稳定指令层
- 保持与平台后端 API 一致

## 三、CLI 的定位

### 面向开发者

开发者可以通过 CLI：

- 初始化服务配置
- 校验本地配置
- 查看平台服务状态
- 查询最近流水线结果

### 面向运维

运维可以通过 CLI：

- 查询审批
- 查看锁定服务
- 执行解锁
- 查看资源状态
- 触发某些平台动作

### 面向 AI Agent

AI Agent 可以通过 CLI：

- 生成标准服���接入配置
- 查询某个服务的当前治理状态
- 执行可审计的查询类操作
- 在用户确认后发起受控变更

## 四、设计原则

### 1. 命令要稳定

AI Agent 依赖命令接口时，频繁变更命令结构会带来维护成本，因此命令层应尽量稳定。

### 2. 输出要结构化

所有查询型命令建议支持：

- 表格输出
- JSON 输出
- YAML 输出

其中 `--output json` 应优先服务 AI Agent。

### 3. 参数语义与 API 保持一致

CLI 不应发明另一套资源命名方式，应与 API 和 WebUI 保持统一。

### 4. 写操作必须可审计

例如：

- 锁定服务
- 解锁服务
- 生成 workflow
- 更新配置

都应通过后端 API 执行并落审计。

## 五、命令分组建议

建议按资源划分命令组。

### 1. 服务命令

```text
nova service ...
```

### 2. 配置命令

```text
nova config ...
```

### 3. Workflow 命令

```text
nova workflow ...
```

### 4. 部署命令

```text
nova deploy ...
```

### 5. 审批命令

```text
nova approval ...
```

### 6. 锁定命令

```text
nova lock ...
```

### 7. 报表命令

```text
nova report ...
```

## 六、典型命令设计

### 1. 初始化服务配置

```text
nova service init
```

交互式收集：

- 服务名
- 项目
- 仓库
- 语言
- 版本
- 构建命令
- 测试命令
- 端口
- 域名
- Jira 项目 Key

输出：

- 本地 `nova.yml`
- 或直接调用平台创建服务

### 2. 校验配置

```text
nova config validate -f nova.yml
```

功能：

- 校验 YAML 合法性
- 输出错误和警告
- 展示平台补齐后的默认值

### 3. 渲染配置

```text
nova config render -f nova.yml --output yaml
```

输出补全默认值后的完整配置。

### 4. 创建服务

```text
nova service create -f nova.yml
```

调用平台 API 创建服务，并可选：

- 自动生成 workflow
- 自动提交到仓库（后续）

### 5. 查看服务列表

```text
nova service list --project app --output table
```

### 6. 查看服务详情

```text
nova service get iplocator --output yaml
```

### 7. 生成 workflow

```text
nova workflow generate --service iplocator
```

### 8. 查看审批列表

```text
nova approval list --status pending
```

### 9. 审批通过

```text
nova approval approve --id 123
```

### 10. 审批拒绝

```text
nova approval reject --id 123 --comment "请补充验证结果"
```

### 11. 查看锁定服务

```text
nova lock list --locked true
```

### 12. 解锁服务

```text
nova lock unlock --service iplocator --jira APP-999
```

### 13. 查看部署记录

```text
nova deploy list --service iplocator --env prod
```

### 14. 查看周报

```text
nova report weekly --week 2026-W15
```

## 七、AI Agent Skill 适配设计

如果 CLI 要给 AI Agent 使用，建议满足以下能力：

### 1. 所有查询命令支持 JSON 输出

例如：

```text
nova service get iplocator --output json
```

### 2. 所有错误输出标准化

例如：

```json
{
  "code": "NOVA_VALIDATION_ERROR",
  "message": "配置校验失败",
  "errors": [
    {
      "field": "spec.runtime.language",
      "message": "仅支持 nodejs、python、go"
    }
  ]
}
```

### 3. 写操作支持 `--yes` 或确认参数

例如：

```text
nova service create -f nova.yml --yes
```

但对高风险操作仍建议要求平台侧确认或权限校验。

### 4. 结果输出应可供 Agent 继续决策

例如：

- 服务是否锁定
- 当前审批是否存在
- workflow 生成是否成功
- 配置有��些默认值被补齐

## 八、CLI 与 API 的关系

CLI 不应直接实现平台逻辑，而应主要作为 API 的一层人机友好封装。

### CLI 负责

- 参数收集
- 交互体验
- 本地文件读写
- 格式转换
- 结果呈现

### API 负责

- 真正的业务逻辑
- 配置校验
- 状态管理
- 审批治理
- 审计记录

这样可以保证：

- WebUI、CLI、AI Agent 使用同一套平台能力
- 逻辑不分叉
- 平台更易维护

## 九、认证建议

CLI 建议支持以下认证方式：

### 1. Token 登录

```text
nova auth login --token xxx
```

### 2. 环境变量

```text
NOVA_API_TOKEN=xxx
NOVA_API_BASE_URL=https://nova.internal
```

### 3. 配置文件

本地写入：

```text
~/.nova/config.yaml
```

保存：

- API 地址
- 访问 Token
- 默认输出格式
- 默认项目

## 十、技术实现建议

如果平台后端使用 Python，CLI 也可优先考虑 Python 实现，例如：

- `Typer`
- `Click`

优点：

- 与后端共享部分数据模型
- 便于快速实现
- 适合交互式 CLI

## 十一、一期建议范围

一期建议优先实现以下命令：

- `nova config validate`
- `nova config render`
- `nova service create`
- `nova service list`
- `nova service get`
- `nova workflow generate`
- `nova approval list`
- `nova lock list`
- `nova deploy list`

一期暂不建议过早实现：

- 大量写操作
- 复杂批量修改
- 本地与远程状态双向同步
- 复杂脚手架生成器

## ��二、总结

NOVA CLI 的核心价值不是替代 WebUI，而是：

> 为自动化、脚本化、AI Agent 化提供一层稳定、清晰、可审计的命令接口。

这将使 NOVA 从一开始就具备更强的可扩展性和自动化潜力。
