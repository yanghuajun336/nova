# NOVA

## 项目简介

NOVA 是一个面向企业级场景的 DevOps 平台设计项目，目标是围绕 Gitea、Jira、Harbor、Kubernetes、SonarQube 等工具，建设一套适用于中小团队、多语言微服务架构的平台化 CI/CD 体系。

当前仓库主要用于沉淀 NOVA 的产品设计、架构设计、配置规范、工作流模板设计以及后续实现路线。

## 当前阶段

当前处于 **架构设计阶段**，重点工作包括：

- 明确平台定位与边界
- 设计 `nova.yml` 配置规范
- 设计 WebUI 与 CLI 交互模型
- 设计中央 Workflow 模板体系
- 规划 Jira、Gitea、Kubernetes 等外部系统集成方式
- 输出后续 API、数据库、审批流等设计文档

## 平台目标

- 面向多语言微服务的标准化 CI/CD 平台
- 降低开发者接入和维护流水线的成本
- 通过模板和默认规则实现平台治理
- 打通 Jira 与代码、流水线、部署、回滚的交付全链路
- 支持生产审批、自动回滚、服务锁定、邮件通知与周报
- 为后续 AI Agent 驱动的 CLI / Skill 能力预留接口

## 核心原则

- **治理在平台，执行在工作流**
- **开发者最小配置，平台自动补全**
- **生产环境必须审批**
- **生产环境必须自动回滚**
- **Jira Issue ID 作为交付全链路追踪键**
- **多语言统一抽象，差异由模板封装**

## 文档索引

- [总体概览](./docs/000-overview.md)
- [产品定位](./docs/001-product-positioning.md)
- [总体架构设计](./docs/002-architecture-overview.md)
- [Jira 联动设计](./docs/003-jira-integration.md)
- [nova.yml 配置规范](./docs/004-nova-yml-schema.md)
- [WebUI 表单设计](./docs/005-webui-form-design.md)
- [中央 Workflow 模板体系设计](./docs/006-workflow-template-design.md)
- [实施路线图](./docs/roadmap.md)

## 建议目录结构

```text
docs/
├── 000-overview.md
├── 001-product-positioning.md
|── 002-architecture-overview.md
├── 003-jira-integration.md
├── 004-nova-yml-schema.md
├── 005-webui-form-design.md
├── 006-workflow-template-design.md
└── roadmap.md
```

## 下一步规划

下一阶段建议继续输出以下设计文档：

- NOVA 后端 API 设计
- NOVA 数据库表结构设计
- 审批流设计
- 资源编排设计
- CLI / AI Agent Skill 设计
- 平台权限模型设计

## 项目名称

- 平台英文名：**NOVA**
- 平台中文名：**星核**

寓意：以统一、稳定、可演进的平台能力，驱动整个研发交付体系持续运转。
