# Jenkins Shared Library

Shared Library 用于把多个 Jenkinsfile 中重复的检出、构建、镜像、GitOps、部署等待和通知逻辑收敛为统一能力。

## 推荐分层

```text
shared-library/
├── vars/
│   ├── devops.groovy              # 稳定门面
│   └── kustomizeUpdate.groovy     # 可直接调用的全局 Step
└── src/com/example/devops/
    ├── PipelineInit.groovy
    ├── GitCheckout.groovy
    ├── MavenBuild.groovy
    ├── FrontendBuild.groovy
    ├── Kaniko.groovy
    ├── UpdateKustomizeRepo.groovy
    ├── JavaMultiServiceRelease.groovy
    └── Notification.groovy
```

## 关键原则

1. Jenkinsfile 保留项目差异，Library 保存跨项目稳定行为。
2. `src/` 类通过构造参数接收 Pipeline `steps`，再调用 `steps.sh`、`steps.lock`、`steps.withCredentials` 等 DSL。
3. 实现类通常需要 `implements Serializable`，避免 Pipeline CPS 持久化时保存不可序列化状态。
4. `lock {}`、`withCredentials {}`、`container {}` 和 `dir {}` 都是作用域闭包；退出闭包时由 Step 自动释放或恢复资源。
5. 先保存日志、制品与状态，再用 `error` 传播失败；不要让第一次非零退出吞掉诊断上下文。
6. 公共库应有版本纪律：开发分支验证、稳定 Tag、兼容期和回滚入口。

## 重点阅读

- [Jenkinsfile 与 Shared Library 分层、steps/闭包/CPS](../../docs/ci-cd/01-architecture-and-pipeline-model.md#4-jenkinsfile-与-shared-library-的分层)
- [Shared Library API 速查](../../docs/ci-cd/06-knowledge-roadmap-and-interview.md#附录-d-shared-library-api-速查)
- [配置骨架](../../docs/ci-cd/06-knowledge-roadmap-and-interview.md#附录-e-三类-jenkinsfile-配置骨架)
