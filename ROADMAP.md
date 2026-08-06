# Roadmap

## 已完成

- Jenkins + Shared Library + Kubernetes Agent 流水线分层。
- 普通 Java、Java 单仓多服务和前端三类标准交付链。
- Maven/pnpm 构建日志、制品归档和成功/失败通知。
- Kaniko 无 Docker daemon 镜像构建与 Registry 鉴权。
- Kustomize GitOps 更新、Argo CD Sync/Health 等待和回滚思路。
- GitLab Webhook 与手动参数的分支/环境统一初始化。
- 同 Job 禁并发与跨 Job GitOps 全局锁。
- 真实故障案例、项目接入 Runbook、API 速查和面试复盘。

## 进行中

- 前端制品语义校验：阻断 `undefined`、缺少入口文件和异常小制品。
- Java 多服务逐服务状态聚合与通知摘要。
- Argo CD observed revision 与本次 Kustomize commit 的精确关联。
- Shared Library 单元测试、测试 Job 和端到端验收分层。
- 流水线成功率、Stage 耗时、锁等待和 Argo 等待时间指标。
- 旧前端依赖升级，逐步移除 hoist 与 OpenSSL legacy 兼容。

## 后续计划

- 生产 promotion、审批、Sync Window 和回滚演练。
- Argo Rollouts Canary / Blue-Green 与 Prometheus 指标分析。
- 发布中心触发 Jenkins，并回写逐服务、镜像、GitOps 和 Argo 状态。
- Kubernetes 常见故障排查 Runbook。
- Prometheus/Grafana 告警规则与企业微信通知示例。
- CMDB、自动化巡检和发布系统前后端实践。
