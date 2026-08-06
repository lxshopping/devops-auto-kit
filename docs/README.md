# 文档与 Runbook

本模块用于沉淀 DevOps、Kubernetes、监控告警、CMDB 和发布系统方向的技术文档、排障 Runbook 与设计说明。

## 建设目标

- 将零散排障经验整理成可执行、可验证、可回滚的标准 Runbook。
- 将 Jenkins、Argo CD、Kustomize 等发布流程沉淀为技术手册。
- 将监控告警、CMDB、发布系统和运维平台建设整理为可复用方案。
- 为后续面试复盘、文章整理和项目接入提供统一资料源。

## 当前目录

```text
docs/
├── ci-cd/              # Jenkins、Kaniko、Kustomize、Argo CD 技术手册
├── cmdb/               # CMDB 和运维平台文档
├── monitoring/         # 监控告警文档
└── troubleshooting/    # Kubernetes 和运维排障 Runbook
```

## 已整理专题

### CI/CD

- [CI/CD 知识体系总览](ci-cd/README.md)
- [架构演进与 Pipeline 执行模型](ci-cd/01-architecture-and-pipeline-model.md)
- [普通 Java 与单仓多服务流水线](ci-cd/02-java-pipelines.md)
- [前端流水线与 Kaniko 镜像构建](ci-cd/03-frontend-and-kaniko.md)
- [Kustomize、Argo CD 与并发控制](ci-cd/04-gitops-kustomize-argocd-locking.md)
- [通知、可追溯性与故障排查 Runbook](ci-cd/05-observability-and-troubleshooting.md)
- [知识模型、改进路线与面试复盘](ci-cd/06-knowledge-roadmap-and-interview.md)
