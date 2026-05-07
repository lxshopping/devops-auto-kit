# ArgoCD 与 Kustomize 发布实践

本模块主要沉淀 ArgoCD、Kustomize 在多环境发布流程中的实践，包括 app-of-apps、镜像替换、多环境配置管理、同步等待、发布回滚等内容。

## 建设目标

通过本模块整理一套 GitOps 发布流程，重点解决以下问题：

- 如何使用 ArgoCD 管理 Kubernetes 应用。
- 如何使用 Kustomize 管理不同环境配置。
- 如何通过 Jenkins 更新镜像版本。
- 如何通过 ArgoCD 触发同步和等待发布结果。
- 如何组织 app-of-apps 模式。
- 如何排查 ArgoCD 发布失败问题。

## 当前目录

```text
argocd/
├── app-of-apps/     # ArgoCD app-of-apps 模式
├── examples/        # ArgoCD 示例配置
└── kustomize/       # Kustomize 多环境配置管理