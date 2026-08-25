# DevOps Auto Kit

`devops-auto-kit` 是一个面向运维开发场景的个人实战知识库，主要用于沉淀监控告警、CMDB、CI/CD 发布流程、Kubernetes 排障、Ansible 自动化、脚本工具和发布系统建设等方向的实践经验。

本仓库不是一个单一项目，而是一套长期维护的 DevOps / 运维开发知识体系，用来保存真实场景中的模板、代码片段、排障 Runbook、系统设计思路和阶段复盘。

## 建设目标

1. 系统沉淀个人运维开发知识体系。
2. 形成 DevOps / Kubernetes / CMDB / 监控告警方向的面试作品集。
3. 为后续资料包、工具包、咨询服务或其他可复用产品打基础。

## 主要模块

### 1. CI/CD 发布流程

目录：`jenkins/`、`argocd/`、`docs/ci-cd/`

- Jenkins Pipeline 与 Shared Library
- Kubernetes 动态 Agent 与持久化缓存
- Maven、pnpm、Sonar 和构建日志
- Kaniko 镜像构建与 Harbor
- Kustomize 多环境配置与 GitOps 更新事务
- Argo CD 同步、健康等待和回滚
- Java、单仓多服务和前端项目模板
- 并发控制、故障 Runbook 与面试复盘

阶段性技术手册入口：[Jenkins + Argo CD CI/CD 知识体系](docs/ci-cd/README.md)

### 2. Kubernetes 排障

目录：`kubernetes/`、`docs/troubleshooting/`

- Pod、CronJob、PVC、Ingress 和 Service 排障
- Supervisor 批量处理
- 日常 Kubernetes 问题处理 Runbook

### 3. 监控告警

目录：`monitoring/`、`docs/monitoring/`

- Prometheus、Alertmanager 和 PrometheusAlert
- 企业微信、钉钉告警模板
- P1-P5 告警等级与降噪
- AI 告警分析思路

### 4. 自动化脚本

目录：`scripts/`

- Shell、Python 和 Ansible 脚本
- 批量巡检、采集、启停和 Kubernetes 资源处理

### 5. CMDB 和发布系统实践

目录：`cmdb/`、`docs/cmdb/`

- 硬件资产采集与模型设计
- Ansible 批量采集
- CMDB 与监控、发布系统的关联
- Django DRF + Vue 运维平台实践

### 6. 统一身份认证与单点登录

目录：`docs/sso/`

- FreeIPA 用户、密码、用户组与 Kerberos
- Keycloak LDAP Federation、OIDC Client、Token 与 Session
- Grafana Generic OAuth 和基于用户组的 Viewer / Editor / Admin 映射
- 用户权限变更、同步、重新登录与故障排查闭环
- 测试环境与生产环境的架构边界和演进建议

实战手册入口：[FreeIPA + Keycloak 企业级 SSO 单点登录实战](docs/sso/freeipa-keycloak-sso.md)

## 当前阶段

- 已完成 Jenkins + Shared Library + Kaniko + Kustomize + Argo CD 阶段性知识沉淀。
- 已完成 FreeIPA + Keycloak + Grafana SSO 实验闭环及 Group-Based RBAC 权限变更验证。
- 已覆盖普通 Java、Java 单仓多服务和前端三类流水线，以及 12 类真实故障闭环。
- 后续继续补充生产 promotion、渐进式发布、流水线指标、发布中心和其他运维模块。

具体计划见 [ROADMAP.md](ROADMAP.md)。

## 适合人群

- 运维工程师、运维开发工程师和 DevOps 工程师
- Kubernetes / SRE 学习者
- 中小企业技术负责人
- 准备 DevOps / Kubernetes / SRE 面试的同学

## 内容说明

本仓库内容来自真实运维开发场景，但公开内容均应脱敏和泛化处理。

生产环境使用前必须结合自身环境验证。涉及删除、覆盖、重启、批量修改等高风险操作时，应先备份或记录原始状态，并明确操作对象、影响范围和回滚方式。

## License

Apache-2.0
