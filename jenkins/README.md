# Jenkins CI/CD 实战

本模块沉淀 Jenkins Pipeline、Shared Library、Kubernetes Agent、Kaniko、Harbor 和 GitOps 发布相关的模板与代码说明。

## 设计边界

- Jenkinsfile 是项目声明与流程编排层，保留 Stage、仓库、镜像、分支环境映射和项目特有配置。
- `vars/` 是对 Jenkinsfile 暴露的稳定门面，`src/` 保存可测试、可复用的实现类。
- 构建日志、凭据作用域、输入校验、失败语义和通知行为由 Shared Library 统一。
- Jenkins 产生制品和修改 GitOps 期望状态；Argo CD 判断同步与 Kubernetes 资源健康。

## 当前目录

```text
jenkins/
├── java-kaniko-harbor/      # Java 构建、Kaniko 和 Harbor 说明
├── vue-kaniko-harbor/       # 前端构建与镜像模板（待补）
├── django-kaniko-harbor/    # Django 镜像模板（待补）
└── shared-library/          # Shared Library 分层与入口说明
```

## 技术手册

- [CI/CD 总览](../docs/ci-cd/README.md)
- [架构演进与 Pipeline 执行模型](../docs/ci-cd/01-architecture-and-pipeline-model.md)
- [普通 Java 与单仓多服务流水线](../docs/ci-cd/02-java-pipelines.md)
- [前端流水线与 Kaniko](../docs/ci-cd/03-frontend-and-kaniko.md)
- [Kustomize、Argo CD 与并发控制](../docs/ci-cd/04-gitops-kustomize-argocd-locking.md)
