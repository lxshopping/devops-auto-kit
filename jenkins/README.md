# Jenkins CI/CD 实战

本模块主要沉淀 Jenkins 在运维发布场景中的实践，包括 Jenkins Pipeline、Shared Library、Kubernetes Agent、Kaniko 构建镜像、Harbor 推送、ArgoCD 发布等内容。

## 建设目标

通过本模块沉淀一套适合中小企业内部项目使用的 CI/CD 发布模板，重点解决以下问题：

- Jenkins 如何在 Kubernetes Agent 中运行。
- Kubernetes 环境没有 docker.sock 时如何使用 Kaniko 构建镜像。
- 如何将镜像推送到 Harbor。
- 如何封装 Jenkins Shared Library，减少重复 Jenkinsfile。
- 如何结合 ArgoCD / Kustomize 实现多环境发布。
- 如何将 Java、Vue、Django 等不同类型项目统一纳入发布流程。

## 当前目录

```text
jenkins/
├── java-kaniko-harbor/      # Java 项目 Kaniko 构建与 Harbor 推送模板
├── vue-kaniko-harbor/       # Vue 项目 Kaniko 构建与 Harbor 推送模板
├── django-kaniko-harbor/    # Django 项目 Kaniko 构建与 Harbor 推送模板
└── shared-library/          # Jenkins Shared Library 封装实践