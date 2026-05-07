# Kubernetes 排障与实战示例

本模块主要沉淀 Kubernetes 日常运维、部署、排障和脚本处理经验，包括 CronJob、PVC、Ingress、Service、Supervisor、Pod 等常见问题。

## 建设目标

通过本模块整理 Kubernetes 生产环境常见问题的处理方法，形成可复用的排障 Runbook 和 YAML 示例。

重点解决以下问题：

- Pod 启动失败如何排查。
- CronJob 到点不执行如何排查。
- PVC 挂载失败如何排查。
- Ingress 无法访问如何排查。
- Service 访问不通如何排查。
- Pod 内 Supervisor 服务如何批量重启。
- 多容器 Pod 日志如何查看。
- Kubernetes 资源修改后如何验证和回滚。

## 当前目录

```text
kubernetes/
├── cronjob/       # CronJob 示例和排障
├── examples/      # Kubernetes 通用示例
├── ingress/       # Ingress 示例和排障
├── pvc/           # PVC 示例和排障
└── supervisor/    # Pod 内 Supervisor 批量处理