# DevOps Auto Kit

`devops-auto-kit` 是一个面向运维开发场景的个人实战知识库，主要用于沉淀我在监控告警、CMDB、CI/CD 发布流程、Kubernetes 排障、Ansible 自动化、脚本工具、发布系统建设等方向的实践经验。

本仓库不是一个单一项目，而是一个长期维护的 DevOps / 运维开发知识体系，用来沉淀真实工作中的模板、脚本、排障 Runbook 和系统设计思路。

## 建设目标

本仓库主要有三个目标：

1. 系统沉淀个人运维开发知识体系。
2. 形成 DevOps / Kubernetes / CMDB / 监控告警方向的面试作品集。
3. 为后续可能的资料包、工具包、咨询服务或副业产品打基础。

## 主要模块

### 1. CI/CD 发布流程

目录：`jenkins/`、`argocd/`、`docs/ci-cd/`

主要沉淀：

- Jenkins Pipeline
- Jenkins Shared Library
- Kubernetes Agent
- Kaniko 镜像构建
- Harbor 镜像仓库
- ArgoCD 发布流程
- Kustomize 多环境配置管理
- Java / Vue / Django 项目发布模板

### 2. Kubernetes 排障

目录：`kubernetes/`、`docs/troubleshooting/`

主要沉淀：

- Pod 排障
- CronJob 排障
- PVC 挂载异常排障
- Ingress 访问异常排障
- Service 访问异常排障
- Supervisor 批量处理
- 日常 Kubernetes 问题处理 Runbook

### 3. 监控告警

目录：`monitoring/`、`docs/monitoring/`

主要沉淀：

- Prometheus
- Alertmanager
- PrometheusAlert
- 企业微信告警模板
- 钉钉告警模板
- P1-P5 告警等级设计
- AI 告警自动分析设计思路

### 4. 自动化脚本

目录：`scripts/`

主要沉淀：

- Shell 运维脚本
- Python 运维脚本
- Ansible 自动化脚本
- 批量巡检
- 批量采集
- 批量启停
- 批量处理 Kubernetes 资源

### 5. CMDB 和发布系统实践

目录：`cmdb/`、`docs/cmdb/`

主要沉淀：

- 硬件资产采集
- CMDB 资产字段设计
- Ansible 批量采集方案
- CMDB 与发布系统对接思路
- Django DRF + Vue 发布系统设计
- 运维平台建设实践

## 适合人群

- 运维工程师
- 运维开发工程师
- DevOps 工程师
- Kubernetes / SRE 学习者
- 中小企业技术负责人
- 准备 DevOps / Kubernetes / SRE 面试的同学

## 当前阶段

当前仓库处于持续整理阶段，内容会优先从以下方向补充：

1. Jenkins + Kaniko + Harbor + ArgoCD 发布流程。
2. PrometheusAlert 企业微信告警模板。
3. Kubernetes 常见故障排查 Runbook。
4. Ansible / Shell / Python 运维脚本。
5. CMDB 硬件资产采集与发布系统设计。

## 内容说明

本仓库内容来自真实运维开发场景，但所有内容均已脱敏和泛化处理。

生产环境使用前，请结合自身环境进行验证，不建议直接复制到生产环境执行。

涉及删除、覆盖、重启、批量修改等高风险操作的脚本，原则上必须先备份原始文件或记录原始状态，并在脚本中明确说明操作对象、影响范围和回滚方式。

## License

Apache-2.0