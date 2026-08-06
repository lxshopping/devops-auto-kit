# Kustomize、Argo CD 与并发控制

> 本文整理自已落地的 Jenkins + Argo CD 流水线实践；示例中的地址、仓库、应用和凭据标识均已脱敏。

## 12. Kustomize GitOps 更新事务

### 12.1 路径解析

`UpdateKustomizeRepo` 支持三种输入优先级：

1. 显式 `overlayPath`。
2. `appPath + overlays + overlayVariant + deployEnv`，用于 B/C 等变体。
3. `appPath + overlays + deployEnv`，兼容普通目录。

例如：

```text
apps/demo-java-app/overlays/test
demo/frontend/.../overlays/b/stage
```

调用方只需传项目根路径、环境和可选变体，Library 统一拼接并验证目标目录与 `kustomization.yaml`。

### 12.2 更新事务的完整关键区

一次 GitOps 写入不是单个 `git push`，而是一个读-改-写事务：

1. 获取全局锁。
2. 准备干净目录并 clone 指定分支。
3. 验证 overlay。
4. 执行 `kustomize edit set image`。
5. `git add`、检测是否有变化。
6. 有变化则 commit 和 push；无变化直接返回当前 commit。
7. 释放锁。

```groovy
steps.lock(resource: lockResource) {
    this.kustomizeCommit = executeUpdate()
}
```

锁必须覆盖 clone 到 push 的整个事务。如果只锁 push，两个构建仍可能基于同一旧 HEAD 分别修改，后提交者产生 non-fast-forward。

### 12.3 镜像名必须精确匹配

`appName` 是 Kustomize `images.name` 的匹配键，`image` 是完整新镜像。命令为：

```bash
kustomize edit set image ${appName}=${image}
```

如果 `appName` 与 kustomization 中的 name 不一致，可能新增错误 image 记录或没有按预期替换。接入新项目时要把项目名、镜像名、deployName 和 Kustomize images.name 的关系写入验收清单。

### 12.4 为什么每次 fresh clone

当前代码在锁内重新 clone，避免前一次构建留下脏工作区、错误分支或未提交文件。旧目录会移动到 `_backup_kustomize` 而不是直接删除，提高故障时的可恢复性。

如果 Agent 工作区长期持久化，需要为备份目录增加清理策略；如果 Agent Pod 和 workspace 构建后即回收，则累积风险较低。这是“安全备份”和“磁盘治理”之间的权衡。

### 12.5 `UpdateKustomizeRepo` 的参数解析

实现优先从调用参数读取，缺省时再从 `env` 读取：

```groovy
this.repoUrl   = args.repoUrl   ?: env.KUSTOMIZE_GIT_URL
this.credId    = args.credId    ?: env.KUSTOMIZE_GIT_CREDENTIAL
this.appName   = args.appName   ?: env.PROJECT
this.appPath   = args.appPath   ?: env.KUSTOMIZE_PATH
this.deployEnv = args.deployEnv ?: env.DEPLOY_ENV
this.image     = args.image     ?: env.CURRENT_IMAGE
```

这种设计让普通项目调用简洁，也允许多服务循环为每个服务覆盖 `appName/appPath/image`。但公共环境变量只能作为默认值，调用多服务时不能依赖前一个服务遗留的 `CURRENT_IMAGE`，所以代码显式构造并传入 image 是正确做法。

`lockEnabled` 使用严格布尔解析，只接受 Boolean 或字符串 true/false。不要使用 Groovy 的一般 truthy 规则，因为字符串 `'false'` 在普通条件判断中仍可能被视为非空真值。

### 12.6 锁资源名如何决定并发边界

```groovy
return argsResource ?:
       env.KUSTOMIZE_LOCK_RESOURCE ?:
       "kustomize-${branchName}"
```

当前所有项目写 `devops` 分支，因此默认锁为 `kustomize-devops`。这是一把保守但正确的分支级锁。可选粒度：

| 锁名策略 | 并发度 | 安全前提 |
|---|---|---|
| `kustomize-devops` | 最低 | 同一 Git 分支所有写入串行，当前最稳妥 |
| `kustomize-devops-test/stage` | 中等 | 不同环境目录完全独立，且不会同时改公共 base |
| `kustomize-<app>-<env>` | 最高 | Git push 需要可靠 rebase/retry，否则不同目录仍会争同一分支 HEAD |

仅按目录分锁并不能自动解决 Git 分支 non-fast-forward，因为不同目录最终仍向同一分支 push。要提高并发度，需要把锁优化与 Git 乐观重试或分支拆分一起设计。

### 12.7 为什么锁必须包住 clone 到 push

错误的锁法：

```groovy
cloneAndEdit()
lock(resource: 'kustomize-devops') {
    gitPush()
}
```

两个 Job 仍可能同时从 commit A clone，各自产生 A+1；先 push 的成功，后 push 的基线已经过期。正确的临界区是：

```groovy
lock(resource: 'kustomize-devops') {
    freshClone()
    validateOverlay()
    editImage()
    commitIfChanged()
    push()
}
```

锁释放发生在闭包退出时，无论 `executeUpdate()` 返回还是抛异常。Argo wait 在闭包外，因此不会把一次 10 分钟 rollout 变成其他项目 10 分钟的 Git 写锁等待。

### 12.8 无变化提交的语义

```bash
git add -A
if git diff --cached --quiet; then
    echo 'no changes to commit'
else
    git commit -m "ci: ... image ..."
    git push origin "${branch}"
fi
git rev-parse --short HEAD
```

当目标 overlay 已引用同一镜像时，不制造空 commit，但仍返回当前 HEAD。调用方应把它理解为“GitOps 期望已经是该版本”，而不是“发布失败”。不过如果当前 HEAD 是其他项目刚提交的 commit，它并不等于本应用最近一次变更 commit；后续若需要精确 Argo revision 关联，应记录目标文件内容或提交前后 HEAD，并让 Argo 等待具体 revision/镜像，而不能仅靠返回的分支 HEAD。

### 12.9 更新前后的验证命令

```bash
cd "${overlayPath}"
sed -n '1,200p' kustomization.yaml
kustomize edit set image "${appName}=${image}"
kustomize build --load-restrictor LoadRestrictionsNone . > /tmp/rendered.yaml
rg -n "image:" /tmp/rendered.yaml
git diff -- kustomization.yaml
git status --short
```

当前代码在编辑前后打印 `kustomization.yaml`，但未在附件中看到 `kustomize build` 语法渲染校验。建议在 commit 前补上该验证；这样路径、资源引用或 Patch 错误会在写入 Git 前失败。临时文件路径应使用任务专用目录，不能共用固定文件导致并发覆盖。

### 12.10 GitOps 回滚标准动作

发布回滚优先恢复 Git 期望状态：

```bash
git clone -b devops <kustomize-repo> kustomize-rollback
cd kustomize-rollback
git show <bad-commit> --
git revert <bad-commit>
git push origin devops
```

然后观察 Argo CD 将旧镜像重新同步并恢复 Healthy。不要用 `git reset --hard` + force push 改写共享分支历史，也不要只在集群执行 `kubectl set image`，否则 Argo 自愈可能把手工回滚再次覆盖。

多服务部分成功时，先用 `git show --name-only` 确认每个服务的独立 Kustomize commit，再决定 revert 哪些 commit。数据库、配置协议不兼容或不可逆迁移不能仅靠镜像回滚解决，应在发布设计中单独约束。

## 13. Argo CD 同步、健康判断与故障语义

### 13.1 当前调用契约

普通 Java 和前端的调用参数一致：

```groovy
argoDeploy(
    server               : env.ARGOCD_SERVER,
    appName              : env.ARGOCD_APP,
    tokenCredentialId    : 'argocd-token',
    timeoutSeconds       : 900,
    refreshBeforeWait    : true,
    triggerSync          : false,
    collectDebugOnFailure: true,
    includeDiffOnFailure : true,
    diffContext          : 3
).start()
```

`triggerSync=false` 表示依赖 Argo CD Application 的 automated sync。Jenkins 更新 Git 后刷新并等待，而不是主动执行第二套部署命令。前提是对应 Application 已开启自动同步；否则流水线可能一直等待目标 revision。

### 13.2 Sync 与 Health 是两个维度

| 状态组合 | 含义 | 排查方向 |
|---|---|---|
| OutOfSync | Git 期望尚未应用 | Argo refresh/sync、Repo、权限、路径 |
| Synced + Progressing | 已应用，资源仍在推进 | rollout、镜像拉取、探针、调度 |
| Synced + Degraded | GitOps 成功，应用健康失败 | Pod 日志、事件、配置、制品语义 |
| Synced + Healthy | 期望状态已应用且资源健康 | 再做业务级验证 |

Argo CD Healthy 仍不等于所有业务用例通过，所以 release/stage 或 prod 后还需要 Robot/API 冒烟测试。健康检查解决的是 Kubernetes 资源状态，业务测试解决的是功能正确性。

### 13.3 自动回滚不能被默认假设

GitOps 的回滚方式是恢复 Git 中的期望版本。当前 Pipeline 在某服务失败时会停止后续服务，但不会自动把已经成功的服务全部恢复。是否自动回滚需要产品和运维共同定义：回滚哪个 commit、是否允许数据库/配置回退、跨服务如何处理。没有这套定义前，最安全的是保留清晰状态和人工决策入口。

### 13.4 `triggerSync=false` 的前置条件

当前调用依赖 Argo CD Application 的 automated sync：Jenkins push Git 后执行 refresh/wait，不主动 `argocd app sync`。这个选择避免“Application 自动同步”和“Jenkins 主动同步”两套触发者并存。

接入新项目时必须核对 Application：

```yaml
spec:
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

如果未开启 automated sync，则 `triggerSync=false` 只会刷新并看到 OutOfSync，最终等待超时。生产若选择人工同步，则应把流水线契约改成“提交 Git 后等待审批/显式 sync”，不能沿用测试环境参数却期待相同行为。

### 13.5 Argo CLI 排障命令

下面命令用于人工复核调用契约，实际 server、token 和 `--grpc-web/--insecure` 参数按环境配置：

```bash
argocd app get "${ARGOCD_APP}" --refresh -o json
argocd app wait "${ARGOCD_APP}" \
  --sync --health --timeout 900
argocd app history "${ARGOCD_APP}"
argocd app diff "${ARGOCD_APP}" --local <rendered-manifest-dir>
argocd app resources "${ARGOCD_APP}"
```

失败时至少记录：Application source repo/path/revision、sync status、health status、operation message、resource health、目标镜像和 Kustomize commit。只有 Pod 日志而没有 Git revision，无法证明 Argo 正在部署本次流水线的版本。

### 13.6 Revision 关联是下一步重点

`refresh + wait Healthy` 可能遇到竞态：本次 Jenkins push commit A 后，另一个 Job 很快 push commit B；Argo 最终 Healthy 的可能是 B。流水线若只等待“应用 Healthy”，就可能把 B 的结果归给 A。

更严格的完成条件应同时满足：

```text
Argo observed revision >= 本次 Kustomize commit
目标资源 image == 本次 CURRENT_IMAGE
Sync == Synced
Health == Healthy
```

具体能否直接等待 commit 取决于 Application 跟踪分支、Argo API 返回字段和当前 `DeployArgoCD` 实现。本次附件未包含该源码，因此把 revision 判定列为待核对项，不能假设已实现。

### 13.7 Argo 权限模型

Jenkins 使用专用 Argo 账号/Token，不应长期依赖 admin 密码。最小权限通常包括读取 Application、刷新/同步指定 Project/Application 以及查看资源；生产权限还应受 RBAC 和 Sync Window 限制。

Token 只保存在 Jenkins Credentials，通过 `withCredentials` 注入；日志中输出 server、app、status 可以，不能输出 Token。若账号无法生成 Token，需要在 `argocd-cm` 为账号启用 `apiKey`，并在 `argocd-rbac-cm` 配置最小策略。

### 13.8 从 Argo 状态下钻到 Kubernetes

当状态为 Synced + Degraded，使用以下顺序：

```bash
kubectl -n <ns> get deploy,rs,pod -o wide
kubectl -n <ns> describe deploy <name>
kubectl -n <ns> get events --sort-by=.lastTimestamp | tail -n 80
kubectl -n <ns> describe pod <pod>
kubectl -n <ns> logs <pod> --all-containers --tail=300
kubectl -n <ns> logs <pod> --all-containers --previous --tail=300
kubectl -n <ns> get deploy <name> -o jsonpath='{..image}'
```

先验证运行中的 image 是否等于 `CURRENT_IMAGE`，再检查 ImagePull、调度、ConfigMap/Secret、探针和应用启动。否则可能花时间排查旧 ReplicaSet 或错误 Pod。

## 14. 并发控制：两种锁解决两类问题

### 14.1 作用域对比

| 机制 | 保护范围 | 能解决 | 不能解决 |
|---|---|---|---|
| `disableConcurrentBuilds()` | 同一个 Job | 同 Job 重复触发、共享工作目录互相清理 | 不同 Job 同时 push GitOps |
| `lock(resource: ...)` | 同一 Controller 下所有 Job/Agent | 跨 Job 共享资源临界区 | 多 Controller；业务层“旧发布覆盖新发布” |
| 独立工作目录 | 单次构建 | 文件覆盖、目录非空、制品串用 | Git 远端分支写冲突 |

前端和多服务 Jenkinsfile 已使用：

```groovy
options {
    disableConcurrentBuilds()
    buildDiscarder(logRotator(
        numToKeepStr: '30',
        artifactNumToKeepStr: '10'
    ))
}
```

`UpdateKustomizeRepo` 默认启用全局锁，资源名按优先级解析：显式参数、`KUSTOMIZE_LOCK_RESOURCE`、`kustomize-${branch}`。当前所有项目更新 `devops` 分支，因此默认使用 `kustomize-devops`，跨前端和 Java Job 排队。

### 14.2 为什么不能锁整个流水线

只有 GitOps 读-改-写是共享关键资源。Maven、前端编译、Kaniko 和 Argo 等待可以并行。如果把全局锁从构建开始持有到应用 Healthy，所有项目发布都会完全串行，吞吐量显著下降。

最小关键区原则是：锁住 clone/edit/commit/push，push 成功即释放；Argo CD 等待在锁外进行。这样既保证 Git 分支一致性，又允许不同应用同时滚动升级。

### 14.3 插件与验证结果

当前 Jenkins 2.462.1 使用兼容版本：

- `lockable-resources:1327.ved786b_a_197e0`
- `data-tables-api:2.1.6-1`

插件已验证 active/enabled，且没有插件要求高于 Jenkins 2.462.1。并发测试中，一个构建占用资源后，另一个构建进入队列，前者释放后后者成功 acquire，证明 Controller 级跨构建锁链路有效。

### 14.4 锁仍然不能解决什么

全局锁保证 Git 写入不冲突，但不判断哪个发布“业务上更新”。如果同一应用先后触发旧 commit 和新 commit，排队顺序仍可能导致后执行的旧构建覆盖新镜像。要解决这种陈旧发布，需要增加同应用 revision 检查、构建取消策略或发布中心状态机，不能把它归给 Git 锁。

### 14.5 `disableConcurrentBuilds()` 与 `lock()` 的执行位置

```groovy
pipeline {
    options {
        disableConcurrentBuilds()
    }
}
```

Declarative option 在 Job 级生效，从构建开始限制同一 Job。它适合保护该 Job 的工作区和同一应用发布顺序。

```groovy
steps.lock(resource: lockResource) {
    executeUpdate()
}
```

`lock` 是 Stage/步骤级临界区，资源由 Jenkins Controller 统一管理，可以跨不同 Job 和不同 Agent Pod。两者不是替代关系：一个保护 Job 生命周期，一个保护共享资源事务。

### 14.6 Lockable Resources 插件安装经验

当前 Jenkins LTS 为 2.462.1，最终采用兼容组合：

```text
lockable-resources:1327.ved786b_a_197e0
data-tables-api:2.1.6-1
```

手动安装前先备份 `$JENKINS_HOME/plugins`，并在隔离目录验证依赖和 Jenkins core 要求。插件 CLI 示例：

```bash
jenkins-plugin-cli \
  --jenkins-version 2.462.1 \
  --plugins lockable-resources:1327.ved786b_a_197e0 \
  --plugin-download-directory /tmp/lockable-bundle \
  --latest false \
  --verbose
```

`--plugin-download-directory` 是下载缓存目录概念，不应仅根据该目录是否为空判断插件最终安装位置；还要检查实际 plugin dir 中的 `.jpi/.hpi`、依赖版本、owner/permission 和 Jenkins 启动日志。重启前的硬性条件是“no plugin requires Jenkins newer than 2.462.1”。

插件恢复和安装属于 Controller 变更，应保留 plugins 备份与回退步骤；不要在依赖解析失败后只拷贝一个主插件就直接重启，因为缺依赖或 core 版本过高可能导致 Jenkins 无法正常加载插件。

### 14.7 锁功能验证流水线

最小测试应让两个构建竞争同一资源，并明确记录等待、获得和释放时间：

```groovy
pipeline {
    agent any
    stages {
        stage('lock test') {
            steps {
                lock(resource: 'kustomize-devops') {
                    echo "acquired by ${env.JOB_NAME} #${env.BUILD_NUMBER}"
                    sleep time: 30, unit: 'SECONDS'
                }
                echo 'released'
            }
        }
    }
}
```

验证不能只在同一 Job 启两个构建，因为 `disableConcurrentBuilds()` 可能已经让第二个构建排队。应使用两个不同 Job 或暂时不加 Job 级禁并发，确认第二个构建显示 waiting for lock，前一个闭包结束后再 acquired。

### 14.8 锁等待的可观测性

当前代码已经输出：

```text
waiting for lock: kustomize-devops
acquired lock: kustomize-devops
```

下一步建议记录等待开始、获得时间和持锁耗时，形成指标：

```text
lock_wait_seconds
lock_hold_seconds
kustomize_update_seconds
```

如果 wait 很高而 hold 很短，说明发布量增加，需要评估分支/仓库结构；如果 hold 很高，重点优化 clone、Git 网络和 Kustomize 校验，不要先拆细锁名。

### 14.9 陈旧构建覆盖的改进方案

全局锁只保证顺序执行，不能保证顺序符合源码新旧。可逐步增加：

1. 同一 Job 新 Push 到来时标记旧排队构建过期，但不盲目中断已经进入 deploy 的构建。
2. 在获得锁后重新查询目标分支/应用最近已发布源码 revision；若当前 commit 不是允许的最新 revision，则跳过部署。
3. 发布中心以订单状态控制同应用同环境只有一个 active release。
4. Kustomize commit message 和状态记录业务 commit，便于机器判断版本顺序。

这属于发布顺序控制，与 Git 写冲突控制是两个问题，设计时必须分开命名和度量。
