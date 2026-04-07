# NOVA Overview

NOVA is an enterprise-grade DevOps platform design focused on platformized CI/CD for small teams managing multiple microservices.

## Background

The platform is intended for a team of 5-10 people, with 20-50 microservice-oriented projects, mainly using Node.js, Python, and Go. The existing infrastructure includes Gitea 1.25.5, Jira Cloud, a Kubernetes cluster with multiple namespaces, and Tencent Cloud CVM resources.

## Core goals

- Provide a platformized CI/CD experience
- Reduce developer cognitive load through templates and defaults
- Standardize build, test, quality, deployment, approval, rollback, and reporting
- Integrate Jira across delivery lifecycle
- Support future AI-agent driven operations through CLI

## Key principles

- Governance in platform, execution in workflow
- Developers provide minimal configuration
- Production deployment must require approval
- Production deployment must support automatic rollback
- Jira issue IDs should be the trace key across the lifecycle

## Initial scope

- Multi-language workflow templates for Node.js, Python, and Go
- Jira integration with status sync and issue creation on rollback
- Harbor image build and push
- Kubernetes deployment to dev, staging, and prod
- Email notifications and weekly reports
- WebUI plus CLI oriented experience

## Out of scope for initial phase

- Full multi-cluster management
- Advanced deployment strategies such as blue-green and canary
- Database migration workflow standardization
- Deep compliance and security policy framework
