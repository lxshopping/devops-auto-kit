# Java + Kaniko + Harbor 流水线

本目录用于沉淀 Java 项目从 Maven 制品到 Kaniko 镜像并推送 Harbor 的通用做法。

## 标准执行链

1. 归一化触发分支和目标环境。
2. 使用固定凭据检出源码与需要的依赖仓库。
3. Maven 构建并保存完整日志、tail 日志和制品元数据。
4. 运行 Sonar；测试环境可不等待质量门，预上线/生产按策略等待。
5. 校验 Jar 存在且唯一，创建只包含 Dockerfile 与目标 Jar 的最小 Context。
6. Kaniko 构建不可变 Tag 并推送 Harbor。
7. 更新 Kustomize Git，等待 Argo CD Sync/Health。
8. 发送包含 commit、镜像、GitOps 和 Argo 状态的通知。

## 关键取舍

- Kaniko 不依赖 Docker daemon，适合 Kubernetes 动态 Agent。
- Maven 命令使用 `returnStatus` 收集退出码，保证失败时仍能归档日志后再终止流水线。
- Java 单仓多服务场景中，编译集合与发布集合分离：根 POM 全量编译，参数只控制镜像和部署服务。
- Jar 通配规则必须得到唯一结果，避免把 `original`、sources 或错误版本打入镜像。
- 多服务的每个服务使用独立最小 Context，避免残留文件串包。

完整实现与排障：[普通 Java 与单仓多服务流水线](../../docs/ci-cd/02-java-pipelines.md)。
