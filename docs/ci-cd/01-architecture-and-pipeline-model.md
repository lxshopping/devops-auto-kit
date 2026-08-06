# 架构演进与 Jenkins Pipeline 执行模型

> 本文整理自已落地的 Jenkins + Argo CD 流水线实践；示例中的地址、仓库、应用和凭据标识均已脱敏。

## 1. 文档定位与阶段结论

这份文档不是项目汇报，也不是只描述“做了什么”的成果清单，而是把这一阶段真正形成的认知、设计决策、故障根因和代码实现串成一套可复用的 CI/CD 知识体系。阅读重点不是记住某个 Jenkinsfile，而是理解每层职责、每个状态边界以及为什么最终代码会演进成现在的样子。

本阶段最重要的结论可以压缩为四句话：

- GitOps 仓库中的声明式配置是部署期望状态，Jenkins 不再直接修改 Kubernetes 运行态。
- Jenkins 仍然负责“推动发布并等待结果”，但健康判断委托给 Argo CD，而不是自己用 sleep、grep Pod 或 kubectl 脚本猜测。
- Jenkinsfile 只保留项目差异和流程编排，通用能力下沉到 Shared Library；参数是项目接入面，代码是平台能力面。
- 并发问题必须按资源作用域治理：同一 Job 用 `disableConcurrentBuilds()`，跨 Job 共享 GitOps 仓库用 Jenkins Controller 级全局 `lock()`。

当前体系已经覆盖普通 Java 项目、Java 单仓多子项目和前端项目，完成了 GitLab Push 自动触发、分支环境映射、动态 Agent、构建日志归档、Kaniko 镜像、Kustomize 更新、Argo CD 健康等待、邮件/企业微信通知以及并发保护。生产分支、渐进式发布、长期指标和发布中心接入仍属于下一阶段。

### 1.1 本阶段的能力边界

| 范围 | 当前状态 | 说明 |
|---|---|---|
| 测试与预上线 | 已落地 | `master -> test`，`release -> stage` |
| 普通 Java | 已落地 | Maven、Sonar、Kaniko、GitOps、Argo CD |
| Java 多子项目 | 已落地 | Reactor 全量编译，按参数构建镜像和发布 |
| 前端项目 | 已落地 | Node/pnpm 构建、dist 制品、Nginx 镜像、日志邮件 |
| GitOps 并发保护 | 已落地 | 同 Job 禁并发 + 跨 Job 全局锁 |
| 生产发布 | 预留 | 前端保留 prod 命令，但当前分支映射尚未接入 prod |
| Argo Rollouts | 架构预留 | 尚未形成正式灰度流程 |
| 发布中心 | 接入准备 | 已有标准参数、状态、日志与通知基础 |

### 1.2 这份手册应该怎样使用

这不是一份要求从头背到尾的说明书，而是一张可检索的技术地图。后续遇到类似问题时，先按现象定位章节，再依次回答以下问题：

1. 问题发生在源码、构建、制品、镜像、GitOps 还是运行健康层？
2. 这个环节的输入和输出分别是什么，权威状态由谁给出？
3. 当前代码在哪一层实现：Jenkinsfile、`vars/` 入口还是 `src/` 实现类？
4. 失败前有没有保留日志、制品、commit、镜像和 Argo 状态？
5. 修复是在解决根因，还是只为某个旧项目增加兼容层？

文档中的代码分为三类：已经在附件源码中核对的当前实现、由当前实现抽取的推荐模板、下一阶段拟改造代码。凡是建议代码，正文会明确标为“改进方案”，避免与已上线代码混淆。

### 1.3 本阶段形成的知识主干

| 知识域 | 必须掌握的核心判断 | 对应实现 |
|---|---|---|
| 职责边界 | Jenkins 产生制品和修改期望状态，Argo CD 判断同步与健康 | Jenkinsfile、Kustomize Git、`argoDeploy` |
| 流水线分层 | 项目差异留在 Jenkinsfile，公共行为进入 Shared Library | `vars/devops.groovy`、`src/com/example/devops` |
| 触发归一化 | Webhook、Jenkins 环境变量和手动参数最终形成同一上下文 | `PipelineInit.groovy` |
| 可追溯交付 | 源码 commit、镜像 Tag、Kustomize commit、Argo revision 互相关联 | checkout、Kaniko、Kustomize、通知 |
| 构建正确性 | 退出码成功之外，还要验证 Jar/dist 的结构和语义 | `MavenBuild`、`frontendBuild`、产物校验 |
| 并发一致性 | 工作区冲突与远端 Git 事务冲突必须分别治理 | `disableConcurrentBuilds()`、`lock()` |
| 失败语义 | `Synced` 只说明期望状态已应用，`Healthy` 才说明资源健康 | Argo CD Sync/Health |
| 兼容与治理 | 旧项目兼容参数应可见、可移除，不能升级为全局默认 | ssh-rsa、`--shamefully-hoist`、legacy provider |

## 2. 从旧模式到当前架构的演进

### 2.1 旧 Jenkins CD 模式暴露的问题

最早的流水线把 Jenkins 同时当作编译机、镜像构建机、部署器和运行状态判断器。典型发布脚本包含 `kubectl apply`、`sleep 30`、`kubectl get pod | grep Running` 或简单 HTTP 探活。这个模式短期容易实现，但规模扩大后会出现以下结构性问题：

- 发布逻辑分散在 Job、本地脚本和固定 Jenkins 节点上，难以统一版本和规范。
- Jenkins 只能知道脚本是否执行完，不能持续感知资源是否漂移、是否 Degraded、是否后来又失效。
- 构建失败、GitOps 失败和应用运行失败混在一个脚本里，故障边界不清晰。
- 新集群或新 namespace 需要复制 Job、脚本和 YAML，恢复与扩容成本高。
- 基于脚本的直接发布难以自然演进到 Canary、Blue/Green 和基于指标的推进。

真正的问题不是 Shell 本身，而是“过程脚本”被当成了“系统状态”。脚本执行成功只说明一次命令返回了 0，不等于期望配置已经持久化，也不等于应用现在健康。

### 2.2 最初新版设计：先完成 CI/CD 解耦

最初的新版设计确定了三条原则：

1. Jenkins 负责编译、测试、扫描、镜像构建和更新 Kustomize Git。
2. GitOps 仓库保存期望状态，所有发布都变成可追溯的 Git commit。
3. Argo CD 监听 Git、同步 Kubernetes、检测漂移、自愈并给出健康状态。

这一阶段解决的是架构职责问题。Kustomize 使用 `base/` 表达通用资源，使用 `overlays/test`、`overlays/stage`、`overlays/prod` 表达环境差异；Argo CD Application 指向具体 overlay，并通过 automated sync、prune、selfHeal 维持状态。

### 2.3 当前实现对最初原则的修正

最初文档中有一句“Jenkins 不判断服务是否 Ready”。当前代码不是完全不等待，而是做了更准确的职责修正：

> Jenkins 不再自己实现 Ready 判断；Jenkins 更新 Git 后，调用 Argo CD refresh/wait，以 Argo CD 的 Sync 和 Health 结果作为流水线结论。

这不是重新耦合回旧 Jenkins CD。两者差异在于：旧模式由 Jenkins 解析 Pod 和脚本输出；当前模式由 Argo CD 持续观察 Kubernetes，Jenkins 只是等待权威状态。因此 Jenkins 可以在发布失败时停止后续阶段并发送完整调试信息，同时不复制一套 Kubernetes 健康判断逻辑。

### 2.4 阶段演进路线

| 阶段 | 主要目标 | 形成的关键能力 |
|---|---|---|
| 旧模式 | 完成一次发布 | Jenkins 直接执行 Shell/kubectl |
| GitOps 基础架构 | CI/CD 解耦 | Jenkins 改 Git，Argo CD 部署和自愈 |
| Shared Library 标准化 | 降低重复 | 初始化、检出、构建、镜像、通知、GitOps 公共封装 |
| 项目类型扩展 | 覆盖实际项目 | 普通 Java、多子项目、前端三套标准路径 |
| 稳定性治理 | 解决真实故障 | 日志制品、子模块鉴权、旧 Node 兼容、全局锁 |
| 下一阶段 | 平台化和度量 | 发布中心、指标、渐进式交付、契约测试 |

### 2.5 最初设计与当前落地的差异矩阵

最初设计解决了方向问题，但真正进入 Java 多模块、旧前端、动态 Agent 和多个 Job 并发后，很多“概念正确”的描述需要转化为更严格的工程约束。下面的差异是本阶段最值得沉淀的架构演进。

| 设计点 | 最初方案 | 当前落地 | 为什么调整 |
|---|---|---|---|
| Jenkins 定位 | Jenkins 只做 CI，不判断 Ready | Jenkins 不自行解析 Pod，但会等待 Argo Sync/Health | 流水线需要得到确定发布结论，同时避免复制健康判断逻辑 |
| 部署动作 | Jenkins 更新 Git，Argo 自动同步 | Kustomize commit 后 `refreshBeforeWait=true`、`triggerSync=false` | 保持 automated sync 单一部署入口，同时缩短状态发现延迟 |
| Job 模型 | 按分支/环境可能复制 Job | 一个项目一个 Job，分支映射环境 | 减少凭据、Webhook、通知和模板漂移 |
| 分支参数 | 早期以 `test/release` 表示环境 | `master/release` 表示源码分支，映射 `test/stage` | 分支与环境是不同概念，不能混用同一字段 |
| 代码检出 | Jenkins Git 插件或 HTTP 凭据 | 显式 SSH fetch/checkout，返回短/完整 commit | 统一凭据、旧 ssh-rsa、子模块与可追溯标签 |
| 构建节点 | 固定 Jenkins Node | Kubernetes Cloud 动态 Agent Pod | 隔离构建环境、按需创建、结束回收 |
| 工具依赖 | 节点本地安装 Maven/Gradle/Node | `tools`、`frontend-tools`、`kaniko` 容器固化版本 | 消除节点漂移，便于复现 |
| 镜像构建 | Docker daemon / 脚本 | Kaniko 无 daemon 构建 | 降低宿主机耦合和高权限 Socket 风险 |
| Java 多模块 | 单应用模型直接复制 | Reactor 全量编译，参数只控制镜像和部署 | 编译依赖图与发布集合不能混为一谈 |
| 前端构建 | Dockerfile 内或脚本临时构建 | frontend-tools 先产出 dist 和日志，Kaniko 再封装 | 构建失败与镜像失败可分离，制品可独立验收 |
| 配置管理 | Nginx 配置可能打进镜像 | ConfigMap/Kustomize 挂载环境配置 | 同一运行镜像模型更清晰，环境差异进入 GitOps |
| 日志 | 依赖 Jenkins Console | Maven/前端完整日志、tail、邮件和 artifact | 失败后证据仍可下载，通知可直接定位 |
| Kustomize 更新 | 每个 Job 自己 clone/push | Library 统一 fresh clone、校验、commit、push | 消除目录假设和实现漂移 |
| 并发 | `disableConcurrentBuilds()` | 同 Job禁并发 + 跨 Job Controller 全局锁 | 两种机制保护的资源作用域不同 |
| 健康判断 | sleep、grep Running | Argo CD Sync/Health + 业务冒烟 | 进程状态、资源健康、业务正确性三层分开 |
| 回滚 | 重新执行旧脚本或手工改 YAML | 优先 `git revert` GitOps commit | 期望状态和审计历史保持一致 |

### 2.6 架构改造的五条不变量

后续无论增加生产、发布中心还是 Argo Rollouts，都应守住以下不变量：

- 同一份源码在流水线中只解析一次真实 revision，后续不得由各工具重新猜版本。
- 所有可部署镜像必须使用不可变 Tag，不能以 `latest` 作为发布事实。
- Kubernetes 期望状态的变更必须进入 Git；紧急手工修改也应尽快回写或回滚。
- 对共享 Git 分支的读-改-写必须具备并发控制或乐观重试，不能依赖“碰撞概率低”。
- 流水线失败必须能定位到具体层和具体对象，不能只留下“Stage failed”。

## 3. 当前总体架构与状态边界

[[ARCH_DIAGRAM]]

### 3.1 组件职责

| 组件 | 负责什么 | 不负责什么 |
|---|---|---|
| GitLab | 业务代码、Webhook、提交身份 | 不保存集群运行状态 |
| Jenkins Pipeline | 流程编排、项目参数、阶段门禁 | 不直接 apply Kubernetes 资源 |
| Shared Library | 通用实现、校验、日志、凭据和错误处理 | 不固化具体项目服务清单 |
| 动态 Agent Pod | 提供隔离且可重复的构建容器 | 不保存 Controller 级锁状态 |
| Harbor | 保存不可变镜像制品 | 不决定部署哪一个 Tag |
| Kustomize Git | 保存部署期望状态和环境差异 | 不执行同步 |
| Argo CD | Sync、Health、漂移感知、自愈、调试 | 不编译业务代码 |
| Kubernetes | 承载最终运行态 | 不保存构建过程信息 |

### 3.2 四类状态必须分开理解

一次发布至少包含四类状态，排障时不能混在一起：

1. 源码状态：业务仓库 commit、分支、子模块 commit 是否正确。
2. 构建状态：Maven/Node 是否成功，产物内容是否语义正确，镜像是否推送。
3. 期望部署状态：Kustomize 是否写入正确镜像，Git commit 是否成功推送。
4. 运行健康状态：Argo CD 是否 Synced，资源是否 Healthy，应用是否可用。

例如“Argo CD Synced 但 Deployment Degraded”表示第三类状态已经成功，第四类状态失败。它不能被描述成“GitOps 没发布成功”，也不能仅通过重新 push 代码解决。

### 3.3 一次标准发布的数据链

- `APP_BRANCH` 决定 `DEPLOY_ENV` 和 Argo Application。
- Git checkout 返回短 commit 与完整 commit。
- commit 参与生成镜像 Tag，保证镜像可追溯。
- Kaniko 推送镜像后设置 `CURRENT_IMAGE`。
- Kustomize 使用 `images.name = fullImage` 修改 overlay。
- GitOps commit 被 Argo CD 观察并同步。
- Argo CD 的 Sync/Health 摘要进入通知，失败时补充 diff/debug。

这条链必须能够回答三个问题：发布了哪份源码、生成了哪个镜像、GitOps 最终引用了哪个镜像。

### 3.4 关键变量就是流水线的数据契约

Pipeline Stage 之间没有传统应用那样稳定的内存对象边界。当前实现主要通过 `env`、返回值和工作区元数据传递信息，因此变量命名和写入时机本身就是接口设计。

| 变量/文件 | 生产者 | 消费者 | 失败时的判断 |
|---|---|---|---|
| `APP_BRANCH` | `PipelineInit` | checkout、Sonar、测试、通知 | 为空或错误会导致整条环境链偏移 |
| `DEPLOY_ENV` | `PipelineInit` | Maven Profile、前端命令、Kustomize、Argo | 必须来自 `branchEnvMap`，禁止 Stage 自行推导 |
| `APP_GIT_COMMIT` | `GitCheckout` | 镜像 Tag、Sonar version、通知 | 必须在 checkout 后立即校验和记录 |
| `APP_GIT_COMMIT_FULL` | `GitCheckout` | 严格审计与回溯 | 短 commit 冲突时使用完整值确认 |
| `CURRENT_IMAGE` | prepare-image/Kaniko | Kustomize、通知 | 必须是包含 Registry、仓库和不可变 Tag 的完整地址 |
| `KUSTOMIZE_COMMIT` | `UpdateKustomizeRepo` | Argo 关联、通知、回滚 | 无变更时仍返回当前 HEAD，不能留空 |
| `ARGOCD_SUMMARY/DEBUG` | `DeployArgoCD` | Notification | 区分成功摘要与失败调试信息 |
| `.jenkins-artifacts/*.jar-path` | 多服务产物校验 | Kaniko Context 准备 | 把“已验证的结果”传给后续阶段，避免重复 glob |
| `logs/frontend-build.log` | `frontendBuild` | archive、邮件 HTML/附件 | 即使 Stage 失败也应尽量保留 |

建议把这些字段视为发布协议：修改名称、含义或写入时机时，要同时检查所有消费者，并通过三类 Jenkinsfile 做回归。

### 3.5 状态与责任的快速判断表

| 看到的现象 | 最先检查 | 暂时不要先做 |
|---|---|---|
| checkout 失败 | 分支、SSH 凭据、实际 URL、子模块 URL | 清 Maven/pnpm 缓存 |
| Maven/Node 非零退出 | 完整构建日志、工具版本、实际命令 | 直接进入 Argo CD |
| 命令成功但镜像异常 | Jar/dist 结构与内容语义 | 只看 Stage 绿色 |
| Kaniko 失败 | Context、Dockerfile、Registry 鉴权 | 改 Kustomize |
| Git push non-fast-forward | 全局锁范围、目标分支、并发 Job | 重试整个流水线掩盖竞争 |
| Argo OutOfSync | Git revision、Application path、auto-sync | 先查业务日志 |
| Argo Synced + Degraded | Pod 事件、镜像内容、配置、探针 | 反复提交同一 Kustomize 变更 |
| Argo Healthy 但接口异常 | Robot/API 冒烟与应用依赖 | 把 Healthy 当成业务验收 |

## 4. Jenkinsfile 与 Shared Library 的分层

当前 Jenkinsfile 通过 `@Library('devops-shared-library') _` 加载共享库。代码层主要有三种形态：

- `vars/devops.groovy`：统一入口，负责把 Jenkinsfile 调用转发到实现类。
- `vars/frontendBuild.groovy`、`vars/kustomizeUpdate.groovy`：面向 Pipeline 的全局步骤。
- `src/com/example/devops/*.groovy`：具有校验、状态和复用逻辑的实现类。

### 4.1 调用关系

| Jenkinsfile 调用 | 实现 | 主要职责 |
|---|---|---|
| `devops.initPipeline()` | `PipelineInit` | 触发来源、分支、环境、Argo App 初始化 |
| `devops.gitCheckout()` | `GitCheckout` | SSH 检出、旧 ssh-rsa、子模块重写、commit 返回 |
| `devops.mvnPackage()` | `MavenBuild` | Maven 执行、日志归档、GitLab 状态、失败传播 |
| `frontendBuild()` | `frontendBuild.groovy` | pnpm 安装、环境构建、dist 校验和打包 |
| `devops.kaniko()` | `Kaniko` | Registry 鉴权、镜像构建和推送 |
| `kustomizeUpdate()` | `UpdateKustomizeRepo` | 加锁、clone、set image、commit、push |
| `argoDeploy()` / `DeployArgoCD` | 附件未包含实现 | refresh、等待 Sync/Health、失败调试 |
| `devops.notification*()` | `Notification` + `Email/WeChat` | 成功/失败多渠道通知 |
| `devops.*JavaRelease*()` | `JavaMultiServiceRelease` | 多服务解析、校验、镜像、顺序发布 |

### 4.2 为什么 Jenkinsfile 仍显式写默认映射

`PipelineInit` 内部已有默认值：`master -> test`、`release -> stage`；具体 Jenkinsfile 又显式传入相同 `branchEnvMap`。这不是无意义重复，而是两层配置的不同职责：

- Library 默认值提供安全兜底，让普通项目少配置也能运行。
- Jenkinsfile 显式值记录项目契约，便于模板复制、代码评审和未来单项目覆盖。
- 如果统一规则变化，可升级 Library 默认；如果某个项目需要例外，可只改该 Jenkinsfile。

因此当前保留两层是合理的。知识点是“默认值属于平台，显式参数属于项目契约”，而不是机械追求零重复。

### 4.3 一个项目一个 Job，而不是一个分支一个 Job

一个 Jenkinsfile 对应一个项目，通常只需要一个 Jenkins Job。`master` 和 `release` 由初始化逻辑映射到不同环境，不需要复制两个 Job。这样可以避免凭据、通知、Webhook、缓存和流水线逻辑在多个 Job 之间漂移。

分支差异应该进入参数与映射，项目差异进入环境配置和 `SERVICE_CONFIG`，公共能力进入 Shared Library。

### 4.4 `steps` 到底是什么

`UpdateKustomizeRepo` 的构造函数接收 Jenkins Pipeline 当前脚本对象：

```groovy
class UpdateKustomizeRepo implements Serializable {
    def steps

    UpdateKustomizeRepo(steps) {
        this.steps = steps
    }
}
```

这里的 `steps` 不是“实例化 lock 类”，而是 Pipeline 的执行上下文。Jenkinsfile 中能直接写 `echo`、`sh`、`lock`、`withCredentials`、`container`、`dir`，是因为 Jenkins 把这些 Pipeline Step 注入脚本绑定；普通 `src/` 类没有这层自动绑定，所以需要通过 `steps.echo(...)`、`steps.sh(...)` 间接调用。

可以把它理解成一组由 Jenkins 运行时提供的能力接口：

| 调用 | 能力来源 | 作用 |
|---|---|---|
| `steps.echo` | Pipeline Basic Steps | 写构建日志 |
| `steps.sh` | Durable Task | 在 Agent 容器中执行 Shell，并可等待/恢复 |
| `steps.lock` | Lockable Resources Plugin | 获取 Controller 管理的资源锁 |
| `steps.withCredentials` | Credentials Binding | 在闭包作用域内注入并掩码凭据 |
| `steps.container` | Kubernetes Plugin | 切换到 Agent Pod 指定容器执行 |
| `steps.dir` | Pipeline Basic Steps | 切换相对工作目录，并在闭包结束后恢复 |

因此 `new UpdateKustomizeRepo(this)` 的含义是：创建一个普通 Groovy 对象，同时把当前 Pipeline 的执行能力交给它；真正的 `lock`、`sh` 等动作仍由 Jenkins 插件执行。

### 4.5 闭包如何表达“在某个作用域中执行”

Groovy 闭包是可传递的代码块。下面的大括号不是 `lock` 的配置对象，而是作为最后一个参数传给 `lock` 的 Closure：

```groovy
steps.lock(resource: lockResource) {
    steps.echo "acquired lock: ${lockResource}"
    this.kustomizeCommit = executeUpdate()
}
```

其语义可以写成伪代码：

```text
等待资源可用
获取资源锁
try:
    执行闭包中的 clone/edit/commit/push
finally:
    释放资源锁
```

代码里看不到 `unlock()`，是因为 `lock` Step 在闭包正常结束、抛异常、Pipeline 中止时都负责退出作用域并释放锁。`withCredentials`、`container`、`dir` 也是同一种“进入作用域 -> 执行闭包 -> 自动恢复/清理”的模式。

闭包可以读取外部变量，例如 `lockResource`，这叫捕获外部上下文；但它不等同于 Python 装饰器。Python 装饰器通常在函数定义阶段把一个函数包装成另一个函数，Groovy/Pipeline 这里是运行时把代码块交给一个 Step 控制执行范围。二者都能做横切能力，但触发时机和语义不同。

### 4.6 为什么类要 `implements Serializable`

Jenkins Pipeline 使用 CPS 转换，把流水线执行状态保存下来，以便 Controller 重启、Agent 暂时离线后继续。只要某个对象可能跨越 `sh`、`input`、`sleep` 等可暂停 Step 存活，就可能被 Jenkins 序列化。

Shared Library 实现类声明 `implements Serializable` 是为了允许保存对象状态，但这不代表任意对象都安全。以下对象尤其要谨慎：

- 非序列化第三方客户端、打开的文件句柄、网络连接。
- 复杂 Java Stream、迭代器或运行时 Closure。
- 把大量业务对象长时间保存在类字段中，并跨多个 Pipeline Step 使用。

当前 `JavaMultiServiceRelease` 的公开方法设计为无状态入口，跨 Stage 数据通过参数、`env` 和工作区文件传递，能降低 CPS 恢复风险。插入排序不用 `sort { ... }` Closure，也是为了规避曾出现的 `CpsClosure2 mismatch`。

### 4.7 `vars/` 与 `src/` 的边界

`vars/devops.groovy` 直接运行在 Pipeline 脚本上下文中，所以可以把 `this` 传给实现类：

```groovy
def initPipeline(Map config = [:]) {
    return new PipelineInit(this).init(config)
}

def mvnPackage(Map config = [:]) {
    return new MavenBuild(this).packageJava(config)
}
```

这种门面层有三个作用：

1. Jenkinsfile 只记住稳定入口，例如 `devops.mvnPackage(...)`。
2. `src/` 类可以按职责拆分、单独校验参数和编写测试。
3. 以后替换内部类或增加兼容逻辑时，不必同时修改所有项目 Jenkinsfile。

不应把所有实现都堆进 `vars/devops.groovy`。当一个步骤出现较多参数、状态、私有校验或多个公开动作时，应下沉为 `src/` 类；简单单入口 Pipeline Step，例如 `frontendBuild.groovy`，保留在 `vars/` 也可以。

### 4.8 Shared Library 版本管理与变更策略

当前 Jenkinsfile 使用 `@Library('devops-shared-library') _`，如果 Library 配置跟随默认分支，公共代码变更会同时影响多个 Job。后续应形成以下发布纪律：

- 破坏性修改先在测试 Job 或独立 Library 分支验证。
- 为稳定版本打 Tag；重大改造时 Jenkinsfile 可临时固定 `@Library('devops-shared-library@<tag>') _`。
- 公共默认参数尽量向后兼容，新增参数给出安全默认值。
- 改动 `PipelineInit`、`GitCheckout`、`UpdateKustomizeRepo` 等基础类时，至少回归普通 Java、多服务 Java、标准前端、旧前端四类样例。
- 保留旧 Jenkinsfile 和 Library Tag 作为短期回退点，但 GitOps 状态回退仍优先通过 `git revert` 完成。

### 4.9 面试中怎样解释这层设计

可以用一句话概括：Jenkinsfile 是项目声明和流程编排，Shared Library 是可测试、可复用的流水线领域层，具体构建仍由 Maven、pnpm、Kaniko、Kustomize 和 Argo CD 各自负责。

如果继续追问“为什么不把 Jenkinsfile 做到最短”，应说明：过度隐藏 Stage 会降低 Blue Ocean/Stage View 可读性，也会让项目负责人看不到流程门禁。当前保留显式 Stage、环境映射和 `SERVICE_CONFIG`，只把重复实现下沉，是可维护性和可见性的平衡。

## 5. 触发、分支与环境初始化

### 5.1 分支来源优先级

`PipelineInit` 对多种触发入口做了归一化，优先级如下：

1. GitLab Webhook：`gitlabBranch`、`gitlabSourceBranch`、`GITLAB_REFS_HEAD`、`GITLAB_REF`。
2. Jenkins 环境：`BRANCH_NAME`、`GIT_BRANCH`。
3. 手动或发布中心参数：默认参数名 `APP_BRANCH`。
4. 默认分支：当前为 `master`。

随后统一移除 `refs/heads/`、`origin/` 和 `*/` 前缀，再校验是否属于允许分支。

```groovy
String triggerBranch = firstNonEmpty([
    webhookBranch,
    jenkinsBranch,
    manualBranch,
    defaultBranch
])

if (!allowedBranches.contains(triggerBranch)) {
    script.error "Unsupported APP_BRANCH=${triggerBranch}"
}
```

### 5.2 初始化产出

初始化成功后统一设置：

- `APP_BRANCH`：规范化后的业务分支。
- `APP_GIT_REF`：形如 `*/master` 的 Git 引用。
- `NOTIFY_BRANCH`：通知使用的真实分支。
- `BUILD_TRIGGER_TYPE`：GitLab、Jenkins 环境、手动/发布中心或默认。
- `DEPLOY_ENV`：test/stage/prod。
- `ARGOCD_APP`：当前分支对应的 Application。

这些值成为后续构建、镜像、Kustomize、Argo CD 和通知的共同上下文，避免每个 Stage 自己猜分支。

### 5.3 GitLab Push 与手动构建的统一

Webhook 使用默认参数自动进入完整流程，手动构建允许覆盖服务范围和通知渠道。多服务项目中 `APP_SERVICES` 的默认值就是 Push 的默认发布集合；人工需要只发一个服务时修改文本参数，无需改代码。

关键经验是：触发方式只决定参数来源，不应产生两套流水线。

### 5.4 `PipelineInit` 的完整判断链

初始化不是简单读取 `params.APP_BRANCH`，而是先收集候选值，再按确定优先级选第一个非空值：

```groovy
String webhookBranch = firstNonEmpty([
    envValue('gitlabBranch'),
    envValue('gitlabSourceBranch'),
    envValue('GITLAB_REFS_HEAD'),
    envValue('GITLAB_REF')
])

String jenkinsBranch = firstNonEmpty([
    envValue('BRANCH_NAME'),
    envValue('GIT_BRANCH')
])

String manualBranch = normalizeBranch(paramValue(manualParamName))

String triggerBranch = firstNonEmpty([
    normalizeBranch(webhookBranch),
    normalizeBranch(jenkinsBranch),
    manualBranch,
    defaultBranch
])
```

这段代码解决了一个容易忽略的问题：Declarative 参数即使是“手动构建参数”，Webhook 触发时也可能存在默认值。如果先读 `params.APP_BRANCH`，Push 到 release 仍可能拿到默认 master。当前实现把 Webhook 放在参数之前，从来源优先级上消除了误发布。

`normalizeBranch` 只做格式归一化，不做环境映射：

```groovy
return value
    .replaceFirst(/^refs\/heads\//, '')
    .replaceFirst(/^origin\//, '')
    .replaceFirst(/^\*\//, '')
```

归一化后先检查 `allowedBranches`，再查 `branchEnvMap` 和 `argocdAppMap`。这两次 fail-fast 很重要：未知分支不能静默落到 test，缺少 Argo App 也不能等到 deploy Stage 才发现。

### 5.5 默认映射与项目显式映射为何同时存在

Library 中的默认值：

```groovy
Map branchEnvMap = (config.branchEnvMap ?: [
    master : 'test',
    release: 'stage'
]) as Map
```

Jenkinsfile 中仍显式传相同 Map：

```groovy
devops.initPipeline(
    allowedBranches: ['master', 'release'],
    branchEnvMap: [master: 'test', release: 'stage'],
    argocdAppMap: [
        master : env.ARGOCD_APP_TEST,
        release: env.ARGOCD_APP_STAGE
    ]
)
```

保留的原因不是技术上不能删除，而是为了把当前项目契约放在项目代码旁边。Library 默认是“没有配置时仍能工作的保底”，Jenkinsfile 参数是“这个项目明确采用什么规则”。未来某项目需要 `develop -> dev` 时，只修改该项目；平台统一规则变化时，仍可更新默认值。

### 5.6 初始化测试矩阵

| 触发场景 | 输入样例 | 预期 `APP_BRANCH` | 预期来源 | 预期环境 |
|---|---|---|---|---|
| GitLab Push master | `gitlabBranch=master`，参数默认 release | master | gitlab-webhook | test |
| GitLab Push release | `GITLAB_REF=refs/heads/release` | release | gitlab-webhook | stage |
| Multibranch | `BRANCH_NAME=release` | release | jenkins-env | stage |
| 手动构建 | `params.APP_BRANCH=master` | master | manual-or-release-center | test |
| 无任何输入 | 所有候选为空 | master | default | test |
| 不允许分支 | `feature/demo` | 立即失败 | gitlab-webhook | 不进入构建 |
| 缺 Argo 映射 | branch 有效、`argocdAppMap` 无对应项 | 立即失败 | 已识别 | 不进入 checkout |

验证时应在 `init(branch)` 日志中同时检查原始变量、规范化结果和最终变量，不能只看 `APP_BRANCH`。这能定位是 GitLab 插件未注入变量、变量名称变化，还是优先级实现错误。

### 5.7 新增生产分支时应改哪些位置

生产接入不能只在 `branchEnvMap` 增加一行。至少需要同步检查：

1. `parameters.APP_BRANCH` 是否允许生产分支或 Tag。
2. `allowedBranches` 与 `branchEnvMap`。
3. `argocdAppMap` 与真实生产 Application。
4. 前端 `FRONTEND_BUILD_CMD_PROD` 或 Maven Profile。
5. Kustomize `overlays/prod` 与 Git 分支策略；当前更适合 devops 到 prod 的受控 promotion，而不是 Jenkins 直接写保护分支。
6. Sonar、Robot、审批、Sync Window 和通知门禁。
7. 回滚和数据库变更策略。

因此“增加 prod”是交付策略变更，不是一个 Map 配置项变更。

## 6. Jenkins Kubernetes 动态 Agent 与构建环境

当前 Pipeline 使用 `agent { label 'jnlp-slave' }`，每次构建由 Jenkins Kubernetes Cloud 动态创建 Agent Pod，构建结束后回收。所有 Agent 仍连接同一个 Jenkins Controller，因此它们共享 Job 配置、构建历史、凭据和 Lockable Resources 状态。

### 6.1 容器职责

| 容器 | 主要工具 | 使用阶段 |
|---|---|---|
| `jnlp` | Jenkins Agent 通信 | 整个流水线 |
| `tools` | Git、SSH、Maven、Gradle、Kustomize、Argo CLI 等 | 检出、Java 构建、GitOps、Argo 等待 |
| `frontend-tools` | Node.js 22、pnpm 9 | 前端依赖安装与编译 |
| `kaniko` | `/kaniko/executor` | 无 Docker daemon 的镜像构建与推送 |
| containerd 相关容器 | 兼容特定构建需求 | Jenkins Cloud 模板扩展能力 |

tools 基础镜像已经补充 Gradle，可直接执行 `/usr/local/gradle/bin/gradle build`。构建工具固化到镜像而不是每次在线安装，提升了可重复性。

### 6.2 缓存与工作区不是同一概念

- Maven 缓存：`/home/jenkins/.m2/repository`，通过 `-Dmaven.repo.local` 显式使用。
- pnpm 缓存：`/home/jenkins/.pnpm-store`，通过 `pnpm config set store-dir` 使用。
- 工作区：保存当次源码、日志和制品，应当按 Job/构建隔离，不应被当作依赖缓存。

PVC 缓存减少依赖重复下载，但也可能放大损坏缓存和版本污染。遇到“只在 Jenkins 失败、本地正常”时，需要区分是源码、工具镜像还是共享缓存问题，不要直接清空整个工作区作为第一反应。

### 6.3 Controller 与 Agent 的锁边界

Lockable Resources 的状态由 Jenkins Controller 管理。即使构建在不同动态 Agent Pod 中，只要属于同一个 Controller，`lock(resource: 'kustomize-devops')` 就能跨 Job、跨 Pod 生效。如果未来拆成多个独立 Controller，这个锁不再是全局锁，需要外部锁或 Git 侧并发控制。

### 6.4 从固定 Node 到动态 Agent 的改进点

最初固定 Jenkins Node 同时承担源码、缓存、Docker、Maven 和脚本，节点上一次手工升级就可能影响所有 Job。动态 Agent 的价值不只是“构建结束自动删除”，还包括把运行条件声明化：

| 维度 | 固定 Node | Kubernetes 动态 Agent |
|---|---|---|
| 工具版本 | 依赖节点长期状态 | 由容器镜像版本决定 |
| 并发能力 | 受单机资源和 executor 数限制 | 可按集群资源并行扩容 |
| 隔离 | Job 共享文件和进程 | 每次构建独立 Pod/容器 |
| 清理 | 需要脚本维护 | Pod 回收清理临时层 |
| 缓存 | 本地磁盘快但不可迁移 | PVC 显式挂载，可跨 Pod 复用 |
| 故障复现 | 依赖“那台机器当时状态” | 使用同一镜像和参数更容易复现 |

但动态 Pod 不会自动解决共享 PVC、同一 Jenkins workspace 路径、远端 Git 分支等共享资源问题。隔离边界必须逐个确认，不能因为“Pod 不同”就默认没有竞争。

### 6.5 容器切换不是启动另一条流水线

```groovy
container('tools') {
    dir('app') {
        sh 'mvn clean package'
    }
}
```

`container('tools')` 只是让闭包内的命令在同一个 Agent Pod 的 tools 容器执行；工作区通常通过共享 volume 同时挂载给 tools、frontend-tools 和 kaniko。正因为文件可跨容器看到，frontend-tools 生成的 `dist.tar.gz` 才能被 kaniko 直接消费。

排障时要确认三件事：容器是否存在、工具是否在该容器 PATH 中、各容器看到的 workspace 挂载路径是否一致。`which git`、`node -v`、`pnpm -v`、`mvn -v` 等版本输出不是多余日志，而是构建环境证据。

### 6.6 Jenkins Cloud 镜像改造的经验

本阶段对 Agent 模板和工具镜像的调整包括：

- 增加 containerd 相关容器以适配特定容器操作需求。
- 在 tools 基础镜像中固化 Gradle，使流水线可直接执行 `/usr/local/gradle/bin/gradle build`。
- 前端使用独立 Node 22 + pnpm 9 镜像，避免把 Java 和前端依赖全部塞进一个大镜像。
- 镜像构建交给 kaniko 容器，不依赖宿主机 Docker Socket。

工具镜像升级应当像应用发布一样可追溯：固定镜像 Tag、记录工具版本、先跑代表性项目，再更新 Pod Template。不要使用不可追溯的 `latest` 直接替换所有 Agent。

### 6.7 缓存故障的判断与处理

共享缓存有三类典型问题：旧索引损坏、依赖版本污染、并发写入异常。排查顺序建议如下：

1. 从构建日志确认实际缓存目录，而不是根据 PVC 名猜测。
2. 在同一工具镜像中用临时空缓存复现一次；若成功，再证明缓存相关。
3. 只隔离具体 group/package 或创建新的任务级缓存目录，避免直接清空整个 PVC。
4. 修复后保留 lockfile/POM 和工具版本证据，避免把偶发下载成功误判为根因消失。

Maven 使用 `-Dmaven.repo.local=/home/jenkins/.m2/repository`，pnpm 使用 `pnpm config set store-dir /home/jenkins/.pnpm-store`。显式参数比只依赖 HOME 默认值更可靠，也便于日志审计。

## 7. Git 检出、SSH 与子模块鉴权

### 7.1 主仓库检出

`GitCheckout` 不依赖 Declarative Pipeline 默认 checkout，而是显式完成：

- 清理并创建指定工作目录。
- 绑定 Jenkins SSH 私钥凭据。
- 设置 `GIT_TERMINAL_PROMPT=0`，禁止构建卡在交互提示。
- 兼容旧 Git 服务的 `ssh-rsa`。
- `git init`、添加 remote、浅拉取目标分支并创建本地分支。
- 返回 7 位短 commit 和完整 commit。

```bash
export GIT_SSH_COMMAND="/usr/bin/ssh -i $SSH_KEY_FILE \
  -o IdentitiesOnly=yes -o BatchMode=yes \
  -o HostKeyAlgorithms=+ssh-rsa \
  -o PubkeyAcceptedKeyTypes=+ssh-rsa \
  -o StrictHostKeyChecking=no"

git fetch --depth=1 origin "${branch}"
git checkout -B "${branch}" FETCH_HEAD
```

短 commit 用于镜像标签和日常展示，完整 commit 用于严格追溯。镜像标签必须由 Pipeline 显式传给 Kaniko，Kaniko 不再自己猜分支或 Git Tag。

### 7.2 HTTP 子模块地址重写为 SSH

部分历史前端项目的 `.gitmodules` 仍保存 HTTP 地址，但 Jenkins 使用的是 SSH 私钥。直接更新子模块会因为凭据类型不匹配而失败。最终方案不是在构建时永久修改业务仓库，而是通过 Git 命令级 `insteadOf` 临时重写：

```groovy
rewriteMap["https://git.example.com/"] = "git@git.example.com:"

git -c 'url.git@git.example.com:.insteadOf=https://git.example.com/' \
    submodule update --init --recursive
```

这样主仓库和子模块共用 Jenkins SSH 凭据，同时避免一次性修改大量历史项目。长期看仍应逐步把 `.gitmodules` 迁移到统一 SSH 地址，运行时重写是兼容层，不是永久标准。

### 7.3 曾经出现的目录问题

历史实现曾在 `dir('kustomize')` 中调用 checkout，却没有保证 Git 插件真正把仓库检出到该目录，后续命令出现 `fatal: not in a git directory`。当前实现改成显式工作目录、`git init/fetch` 或 `git clone <repo> <workdir>`，并在执行前验证目录和 `kustomization.yaml`，消除了“逻辑目录”和“真实 Git 根目录”不一致的问题。

### 7.4 为什么选择显式 `git init/fetch/checkout`

`GitCheckout` 当前不依赖 Jenkins 默认 checkout，原因是需要同时控制工作目录、浅克隆、SSH 参数、子模块 URL 重写和 commit 元数据。核心过程如下：

```bash
rm -rf .git
git init
git remote remove origin >/dev/null 2>&1 || true
git remote add origin "${repoUrl}"
git fetch --depth=1 origin "${branch}"
git checkout -B "${branch}" FETCH_HEAD
git rev-parse HEAD > .git_commit_full
git rev-parse --short=7 HEAD > .git_commit_short
```

这里 `checkout -B` 会让本地分支明确指向刚 fetch 的 `FETCH_HEAD`，而不是沿用 workspace 中可能存在的旧分支。`--depth=1` 适合只需要当前 revision 的构建，但如果后续 Sonar blame、变更集或版本生成需要完整历史，应把浅克隆设计成参数，而不是在构建中临时执行不可控的 `git fetch --unshallow`。

### 7.5 SSH 凭据的正确模型

当前仓库使用 SSH 地址时，Jenkins 凭据应是 SSH Username with private key：

- Username 使用 Git 服务 SSH 用户，通常是 `git`，不是个人 GitLab 用户名、邮箱或 root。
- Private Key 必须与 GitLab 中登记的公钥配对，不能只复制公钥文本。
- `sshUserPrivateKey` 把私钥临时写入文件，并通过 `keyFileVariable` 暴露路径。
- `IdentitiesOnly=yes` 强制只使用绑定的私钥，避免 Agent 里其他默认 key 干扰。
- `BatchMode=yes` 和 `GIT_TERMINAL_PROMPT=0` 禁止后台构建等待密码交互。

HTTP/PAT 与 SSH 私钥是两套不同凭据模型。HTTP 拉取可用 Username + PAT，SSH 地址必须使用私钥；子模块地址如果仍为 HTTP，就需要 URL 重写或单独 HTTP 凭据，不能因为主仓库 SSH 成功就认为子模块也会成功。

### 7.6 `insteadOf` 为什么必须作用在实际命令上

当前实现把映射构造成 Git 临时配置参数：

```groovy
options << "-c ${shellQuote("url.${to}.insteadOf=${from}")}"
```

并把同一组选项传给：

```bash
git ${rewriteOptions} submodule sync --recursive
git ${rewriteOptions} submodule update --init --recursive
```

如果只执行一次 `git config` 但配置作用域错误，或只在 `sync` 时带 `-c`、在 `update` 时漏掉，真正 fetch 子模块时仍会使用原 HTTP URL。排障应输出三类证据：`.gitmodules` 原始 URL、有效 rewrite 配置、`git submodule status --recursive` 最终状态。

### 7.7 旧 `ssh-rsa` 参数的定位

```bash
-o HostKeyAlgorithms=+ssh-rsa
-o PubkeyAcceptedKeyTypes=+ssh-rsa
```

这两个参数用于兼容旧 Git SSH 服务，不是现代 SSH 的推荐长期配置。前者控制服务端主机密钥算法，后者允许客户端公钥签名算法。它们应只存在于访问旧仓库的兼容路径中，并配合 Git 服务升级计划逐步移除。

`StrictHostKeyChecking=no` 和 `UserKnownHostsFile=/dev/null` 也降低了主机身份校验强度。当前做法解决了动态 Agent 无 known_hosts 的可用性问题；更成熟方案是把受信任 Git 主机 key 以 ConfigMap/Secret 注入并启用严格校验。

### 7.8 检出故障排查清单

```bash
git --version
/usr/bin/ssh -V
ssh -vvv -i "$SSH_KEY_FILE" -o IdentitiesOnly=yes git@<git-host>
git config -f .gitmodules --get-regexp 'submodule\..*\.url'
git config --local --get-regexp '^url\..*\.insteadOf' || true
git submodule sync --recursive
git submodule update --init --recursive
git submodule status --recursive
git rev-parse --show-toplevel
git rev-parse HEAD
```

在 Jenkins 中不要把私钥内容或完整 Token 打到日志。调试重点是变量是否存在、文件权限、实际 URL、算法协商和 Git 根目录，不是输出秘密本身。
