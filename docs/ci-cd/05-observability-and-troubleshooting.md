# 通知、可追溯性与故障排查 Runbook

> 本文整理自已落地的 Jenkins + Argo CD 流水线实践；示例中的地址、仓库、应用和凭据标识均已脱敏。

## 15. 通知、日志与制品追溯

### 15.1 通知链路

`Notification` 支持 email、wechat、email+wechat 和 none。成功与失败共用基础信息、部署信息、Argo 摘要、Build Tasks 和快捷链接。

- 企业微信只发摘要，避免消息过长。
- 前端邮件把完整构建日志转换为 HTML，并附加日志文件。
- Maven 把构建 tail 追加到 `BUILD_TASKS`，完整日志和 tail 同时归档。
- Argo 失败时优先展示 debug/diff，成功时展示 summary。
- 通知异常被捕获并记录，不反向把已经成功的发布判为失败。

### 15.2 前端失败时为什么仍有日志

`frontend-build` Stage 使用 `post { always { archiveArtifacts ... } }`。即使 pnpm install 或 build 失败，只要日志已经生成，就会被归档；`allowEmptyArchive` 避免在极早失败时归档动作覆盖原始错误。

构建脚本使用 `set -euo pipefail`，输出通过 `tee` 写入日志。`pipefail` 确保前面的 pnpm/build 失败不会被 `tee` 的成功退出码掩盖。

### 15.3 当前通知实现的边界

- `Email.normalizeRecipients` 支持中英文逗号和分号，但最终把 to/cc 合并到同一收件人列表，当前并没有真正独立抄送语义。
- 多服务 Jenkinsfile 设置了 `NOTIFY_IMAGE_TAG` 和 `NOTIFY_RELEASE_SERVICES`，但当前附件中的 `Notification` 没有读取这两个字段；最终通知可能只展示最后一个 `CURRENT_IMAGE`。应补充多服务发布计划和逐服务状态摘要。
- `argoDeploy`、`notifyUtils` 的具体实现未包含在本次附件中，当前文档只能从调用契约确认其行为，后续知识库应把它们纳入同一代码清单。

### 15.4 通知渠道为什么由参数控制

Jenkinsfile 提供：

```groovy
choice(
    name: 'NOTIFY_CHANNELS',
    choices: ['email', 'email,wechat', 'wechat', 'none']
)
```

`Notification.normalizeChannels` 同时兼容中英文逗号和分号，并拒绝不支持的值。通知渠道属于发布请求差异，适合参数化；发送格式、异常处理和公共字段属于平台能力，留在 Library。

前端没有直接把完整日志发到企业微信，而是把同一通知拆成两次渠道调用：邮件带 HTML 正文和附件，企业微信只带摘要。这避免消息长度限制和 Markdown 展示问题，也使失败排查无需先打开 Jenkins Console。

### 15.5 通知失败为何不反向改变发布结果

```groovy
try {
    new Email().send(...)
} catch (Exception e) {
    echo "[Notification] email notify failed: ${e.getMessage()}"
}
```

发布成功后 SMTP 短暂失败，不应把已经 Healthy 的应用标记为发布失败；因此通知异常被捕获并记录。相反，如果业务要求“未通知就不算完成”，应把通知建模成单独状态或补偿任务，而不是让 post 里的邮件异常覆盖真实部署状态。

### 15.6 Email 实现的收件人语义

`normalizeRecipients` 支持逗号、分号和中文标点，但 `send/sendHtml` 最终执行：

```groovy
String allRecipients = [emailTo, emailCc]
    .findAll { it?.trim() }
    .join(',')

emailext(to: allRecipients, ...)
```

也就是说当前 `cc` 只是合并到 `to`，不是 Email Extension 的真实 `cc` 字段。Jenkinsfile 注释已经把原抄送人员直接写进 `NOTIFY_EMAIL_TO`，这是与实现一致的用法。若以后需要真实抄送，应修改 `mailArgs` 明确传 `cc: emailCc`，并回归空 to/只有 cc/多个分隔符等场景。

### 15.7 日志长度与敏感信息控制

Maven tail 允许 10000 行但通过 `maxChars` 再截断；Argo 摘要限制约 1800 字符；前端完整日志作为邮件 HTML 和附件。不同渠道限制不同，不能用同一份无限正文发送。

日志进入邮件前应继续确保：

- HTML 转义 `& < >`，避免构建输出破坏邮件结构。
- 不输出 Token、私钥、Registry 密码、完整 Credentials 文件。
- Jenkins Credentials masking 只是最后防线，Shell 不应 `set -x` 输出包含秘密的命令展开。
- 对超长日志保留“完整 artifact + 邮件关键片段 + Jenkins URL”三级入口。

### 15.8 多服务通知改进方案

当前 `Notification` 只读取 `CURRENT_IMAGE`，循环发布后往往只剩最后一个服务镜像；虽然 Jenkinsfile 设置 `NOTIFY_RELEASE_SERVICES` 和 `NOTIFY_IMAGE_TAG`，Library 未消费。

建议通知正文新增表格或等宽摘要：

```text
service             image-built  gitops     argo-health  status
config-service   yes          a1b2c3d    Healthy      SUCCESS
gateway-service         yes          d4e5f6a    Degraded     FAILURE
auth-service            yes          -          -            NOT_STARTED
```

数据应来自逐服务状态模型，而不是在 post 中根据 `CURRENT_RELEASE_SERVICE` 猜测。邮件可展示完整表，企业微信只展示失败项和链接。

## 16. 实际问题闭环案例库

### 16.1 Webhook、手动参数与环境不一致

**需求与现象。** 希望 GitLab Push 无需人工填写参数即可发布默认环境，同时保留手动构建入口。实际曾出现 Push release 时仍读取参数默认 master，或者不同 Jenkinsfile 使用不同 GitLab 变量名，最终环境选择不一致。

**根因。** Webhook、Multibranch、普通 Job 参数和发布中心会从不同变量提供分支；Declarative 参数在自动触发时也可能始终存在默认值。如果每个 Stage 自己用 `params.APP_BRANCH ?: env.gitlabBranch`，优先级很容易写反。

**最终方案。** 所有来源只在 `PipelineInit` 解析一次，优先级固定为 GitLab Webhook -> Jenkins 分支环境 -> 手动参数 -> 默认分支；随后规范化前缀、检查白名单、映射环境和 Argo Application。

```groovy
String triggerBranch = firstNonEmpty([
    webhookBranch,
    jenkinsBranch,
    manualBranch,
    defaultBranch
])

script.env.APP_BRANCH = triggerBranch
script.env.DEPLOY_ENV = valueFromMap(branchEnvMap, triggerBranch)
script.env.ARGOCD_APP = valueFromMap(argocdAppMap, triggerBranch)
```

**验证。** 分别 Push master/release、手动选择 master/release，并在 init 日志核对原始变量、`BUILD_TRIGGER_TYPE`、`APP_BRANCH`、`DEPLOY_ENV`、`ARGOCD_APP`。使用一个 Job 完成全部验证，不新建分支专用 Job。

**预防复发。** 新触发来源只允许扩展 `PipelineInit`，业务 Stage 不再读取原始 GitLab 变量。触发入口可以不同，内部上下文必须统一。

### 16.2 Git 子模块在机器上可拉取、Jenkins 中失败

**现象。** 在宿主机手工 `ssh -T` 或 clone 成功，但 Jenkins Agent 更新子模块时仍访问 `https://git.example.com/...`，报用户名密码或仓库权限错误。

**证据。** 主仓库 URL 已是 `git@git.example.com:...`，`.gitmodules` 仍是历史 HTTP URL；Jenkins 绑定的是 SSH 私钥而不是 HTTP PAT。说明“SSH 可用”只证明主仓库链路，不代表子模块实际走 SSH。

**根因。** 子模块使用自己的 URL；早期 `insteadOf` 配置没有作用到真正执行 fetch 的 `submodule update`，或配置方向写反。

**最终方案。** Jenkinsfile 在启用 submodule 时建立 `from HTTP -> to SSH` Map，`GitCheckout` 把它转换为临时 `-c url.<ssh>.insteadOf=<http>`，并在同一个凭据作用域中执行 sync/update。

```groovy
rewriteMap[env.GIT_SUBMODULE_REWRITE_FROM] =
    env.GIT_SUBMODULE_REWRITE_TO

git ${rewriteOptions} submodule sync --recursive
git ${rewriteOptions} submodule update --init --recursive
```

**验证。** 输出 `.gitmodules` 原 URL、有效 rewrite 配置和 `git submodule status --recursive`；必要时开启 Git trace 观察实际请求 URL，但不得打印凭据。

**长期治理。** 逐步把业务仓库 `.gitmodules` 迁移为统一 SSH 地址，运行时 rewrite 只保留为兼容层。

### 16.3 pnpm frozen lockfile 失败

**现象。** 修改 `package.json` 后本地可以安装，Jenkins `pnpm install --frozen-lockfile` 失败，提示 lockfile 与 package manifest 不一致。

**根因。** frozen 模式的职责就是拒绝在 CI 中改写依赖解析结果。这不是 pnpm 故障，而是仓库提交不完整；本地可能自动更新了 `pnpm-lock.yaml`，但没有一起提交。

**标准解决。** 使用项目约定 pnpm 版本重新执行 install，审查 lockfile 变化并与 package.json 一起提交。Jenkins 保持 `frozenLockfile: true`。

**旧项目过渡。** 对没有有效 lockfile 的历史项目，可在该项目 Jenkinsfile 显式：

```groovy
frontendBuild(
    frozenLockfile: false,
    preferOffline : true
)
```

Library 会改用 `--no-frozen-lockfile`。该配置不能升级为通用默认，并应登记恢复 frozen 的技术债。

**验证。** 构建日志必须显示最终 install 命令和是否发现 `pnpm-lock.yaml`；连续两次在空 node_modules 下安装应得到相同 lockfile/依赖版本。

**错误做法。** 删除 package.json 项、长期忽略 lockfile 或每次在 Jenkins 动态修改依赖，只会让构建失去可复现性。

### 16.4 `spawn vite ENOENT`

**现象。** `pnpm install` 完成，但执行旧项目脚本时报 `spawn vite ENOENT`，看起来像 vite 没安装或 PATH 丢失。

**排查证据。** 检查 package.json 中 vite 是否为直接 devDependency、`node_modules/.bin/vite` 是否存在、pnpm 版本和 node_modules linker/hoist 行为。若旧 npm 项目依赖上层间接依赖暴露，pnpm 严格布局会使脚本不可见。

**过渡解决。** 项目级增加：

```groovy
FRONTEND_INSTALL_EXTRA_ARGS = '--shamefully-hoist'
```

该参数让布局更接近旧 npm 扁平 node_modules，使历史隐式依赖暂时可见。

**根治方向。** 把 vite 和项目实际使用的 CLI 明确写入直接依赖，升级锁文件，逐步移除 `--shamefully-hoist`。如果 CLI 已存在仍报 ENOENT，还需检查 package script、工作目录、可执行权限和 Shell，不要机械追加 hoist。

**验证。** `pnpm exec vite --version` 或 `test -x node_modules/.bin/vite` 成功，并且流水线进入真正业务编译阶段。兼容参数解决依赖布局，不能替代依赖声明治理。

### 16.5 Node 22/OpenSSL 构建失败

**现象。** 项目从旧 Node 环境迁移到 Node 22 后，Webpack/crypto 报 OpenSSL 不支持的算法，或大型构建出现 JavaScript heap out of memory。

**根因。** Node 22 使用 OpenSSL 3，旧 Webpack 哈希算法与新安全默认不兼容；内存错误则是 V8 堆上限不足。两个症状可能同时出现，但解决的是两个独立限制。

**当前兼容方案。** 只在受影响项目 Jenkinsfile 增加：

```groovy
environment {
    NODE_OPTIONS = '--max-old-space-size=8192 --openssl-legacy-provider'
}
```

Node 子进程自动读取，无需在 package.json 重新拼接。若日志仍未生效，输出 `process.env.NODE_OPTIONS` 验证注入链，而不是把变量直接写入源码。

**长期方案。** 升级 Webpack、Sass、xlsx-style 等旧依赖，验证后移除 legacy provider；根据实际峰值调整内存，而不是所有项目统一 8 GiB。

**验证与边界。** 构建能够进入并完成编译，只能证明运行时兼容问题解决；若产物路径仍出现 undefined，应继续检查业务环境变量，不能把两个问题合并归因。

### 16.6 构建成功但产物为 `undefined`

**完整现象链。** release/stage 最初报 `spawn vite ENOENT`；加入依赖布局和 Node 兼容后命令退出 0，但产物生成到 `dist/undefined/`，文件名为 `*.undefined.<timestamp>.js`。错误制品被打进镜像后，Argo CD 显示 Synced，Deployment 长时间 Progressing 并最终 Degraded。

**关键判断。** Node/OpenSSL 修复只解释“编译进程为何能运行”，不能解释业务变量为什么是 undefined。Argo Synced 说明 GitOps 已应用指定镜像，也不能证明镜像内容正确。

**根因。** `vue.config.js` 实际读取 Vue CLI 变量 `VUE_APP_PROJECT_NAME` 和 `VUE_APP_VERSION`，`.env.stage` 只有 Vite 风格 `VITE_PROJECT_NAME`。`VITE_*` 不会自动替代 `VUE_APP_*`。

**最终配置。**

```dotenv
NODE_ENV=stage
VITE_PUBLIC_PATH=/demo-ui/
VITE_PROJECT_NAME=demo-ui
VUE_APP_PROJECT_NAME=demo-ui
VUE_APP_VERSION=0.1
VITE_DROP_CONSOLE=false
```

**验证。** 重新构建后必须同时确认：命令退出 0、存在 `dist/demo-ui/`、文件名包含 `0.1`、没有任何 undefined 路径、tar 结构正确、镜像部署后 Pod Healthy 和页面可访问。

**预防复发。** 在 `frontendBuild` 增加 required path 和 forbidden pattern 语义校验；中长期统一 Vite/Vue CLI 变量体系。成功退出是过程成功，制品验收才是交付成功。

### 16.7 Java 多模块选择性构建失败

**需求。** Push 默认发布 config-server、gateway、auth，手动可以只选一个或多个服务，服务清单未来还会变化。

**失败方案。** 直接把 `APP_SERVICES` 转成 Maven `-pl/-am`，希望“选择谁就只编译谁”。实际出现 commons/父 POM/间接依赖缺失、模块坐标与目录名不一致、Profile 下 Reactor 关系变化等问题。

**根因。** 发布选择表达的是交付意图，Maven Reactor 表达的是构建依赖图；两者不是同一个集合。让文本参数控制 Reactor，会把复杂依赖判断推给 Jenkinsfile。

**最终方案。**

```groovy
devops.mvnPackage(
    goals: "clean package -U -Dmaven.test.skip=true -P${env.DEPLOY_ENV}"
)

devops.buildJavaReleaseImages(
    serviceNames: env.RELEASE_SERVICES,
    serviceConfig: SERVICE_CONFIG,
    ...
)
```

Maven 总是从根 POM 全量编译；服务参数只进入产物验证、镜像和部署阶段。

**验证。** 手动只选择 auth 时，日志仍显示所有 Reactor 模块编译，但 Harbor 只新增 auth 镜像，Kustomize/Argo 只更新 auth。编译图由构建系统决定，发布集合由交付系统决定。

### 16.8 Jar 匹配错误与镜像内容不确定

**现象。** 模块 target 可能同时包含主 Jar、`.original.jar`、sources 和 javadoc；旧脚本使用通配符取第一个文件，存在把错误 Jar 放进镜像的风险。多服务共用仓库 Context 时，也可能带入其他服务产物。

**最终方案第一层：唯一 Jar。** `find` 使用 artifactGlob，并排除三类辅助文件；匹配数量必须严格为 1。0 个和多个都失败，并把全部匹配写入日志。

```bash
find "${ARTIFACT_DIR}" -maxdepth 1 -type f \
  -name "${ARTIFACT_NAME}" \
  ! -name '*-sources.jar' \
  ! -name '*-javadoc.jar' \
  ! -name '*.original.jar' \
  | sort -u > "${MATCH_FILE}"

test "$(wc -l < "${MATCH_FILE}")" -eq 1
```

**最终方案第二层：最小 Context。** 把选中的 Jar 固定复制为 `target/app.jar`，Context 只允许 Dockerfile 和 app.jar 两个文件。

**验证。** 日志记录匹配列表、最终 `.jar-path`、Jar 大小和 Context 文件清单；Kaniko 使用服务独立目录。可进一步增加 SHA-256 作为改进。

**可复用原则。** 不让下游“猜一个产物”，而是上游验证后把确定结果作为元数据传递。

### 16.9 Kustomize push non-fast-forward

**现象。** 多个前端/Java Job 同时发布到测试或预上线，都会更新同一 Kustomize 仓库 devops 分支。两个构建各自 clone 后修改不同文件，其中一个 push 成功，另一个报 non-fast-forward。

**为什么修改不同文件也冲突。** Git push 比较的是分支 HEAD，不是文件是否相同。两个 Job 都基于 commit A，分别生成 B 和 C；远端先变成 B 后，C 不再是 B 的快进提交。

**错误认识。** `disableConcurrentBuilds()` 只限制同一 Job；只在 `git push` 外加锁也不够，因为 clone/edit 已经基于过期 HEAD 完成。

**最终方案。** 安装 Lockable Resources Plugin，由 `UpdateKustomizeRepo` 默认获取 `kustomize-devops`，锁覆盖 fresh clone、overlay 校验、edit、commit、push。

```groovy
steps.lock(resource: lockResource) {
    this.kustomizeCommit = executeUpdate()
}
```

push 结束闭包退出即自动释放，Argo wait 在锁外。

**验证。** 使用两个不同 Job 同时触发，第二个日志显示 waiting，第一 push 结束后第二 acquired，并从最新远端 HEAD clone 后成功提交。锁粒度必须覆盖完整读-改-写竞争窗口。

### 16.10 前端测试与预上线共享目录冲突

**现象。** 同一前端 Job 的 master/test 与 release/stage 构建重叠，checkout 和 frontend-build 都操作 `app/`、`dist/`、`dist.tar.gz` 和 logs，出现目录非空、文件被清理或制品串用。

**根因。** Job 使用相同 workspace 路径；动态 Agent 是否不同取决于 Jenkins workspace/volume 配置，不能假设所有文件天然隔离。即使使用不同 Pod，共享 PVC 上固定目录也会竞争。

**最终方案。** 前端 Jenkinsfile 增加：

```groovy
options {
    disableConcurrentBuilds()
    buildDiscarder(logRotator(
        numToKeepStr: '30',
        artifactNumToKeepStr: '10'
    ))
}
```

同时保留按构建隔离工作目录的设计方向，例如 `${JOB_NAME}/${BUILD_NUMBER}` 或 Jenkins 原生独立 workspace；依赖缓存继续使用独立 PVC 目录，不与 workspace 混用。

**验证。** 快速连续触发同一 Job，第二个在 Job 级排队；每次归档的日志和 tar 对应自己的 BUILD_NUMBER。再用两个不同 Job 验证 Kustomize 全局锁，因为 Job 级禁并发不能覆盖跨 Job。

**可复用原则。** 本地文件冲突和远端 Git 事务冲突是两类资源竞争，必须分别治理。

### 16.11 Argo CD Synced 但 Degraded

**现象。** Kustomize commit 已推送，Application 显示 Synced，Deployment 先 Progressing，超时后 Degraded。流水线因此失败。

**状态解释。** Synced 证明 Git 中期望状态已经应用到集群；Degraded 证明至少一个受管资源健康检查失败。两者同时出现完全合理，不能称为“Argo 没同步”。

**排查顺序。**

1. 确认 Deployment 当前 image 等于本次 `CURRENT_IMAGE`。
2. 看 ReplicaSet/Pod events：ImagePull、调度、Mount、探针。
3. 看 current/previous logs：应用启动、配置、端口。
4. 复核镜像内容和构建制品；前端检查 index/static path，Java 检查 Jar。
5. 最后做业务入口验证。

```bash
kubectl -n <ns> get deploy <name> -o jsonpath='{..image}'
kubectl -n <ns> describe deploy <name>
kubectl -n <ns> get events --sort-by=.lastTimestamp | tail -n 80
kubectl -n <ns> logs <pod> --all-containers --previous --tail=300
```

**本次实际根因。** `demo-ui` 镜像内是 `dist/undefined` 错误制品，源于 `.env.stage` 缺少 `VUE_APP_PROJECT_NAME/VUE_APP_VERSION`。GitOps 和 Argo 正确部署了一个错误镜像。

**处理。** 修复源码环境配置、重新构建不可变新 Tag、更新 GitOps，让 Argo 恢复 Healthy。不要在旧镜像上手工修容器，也不要用同一可变 Tag 覆盖后期待节点一定拉到新内容。

**可复用原则。** 先判断失败属于源码、制品、期望状态还是运行健康，再进入对应层；状态名本身就是排障路由。

### 16.12 案例记录模板

后续每次重要问题都按以下模板补充，避免只留下最终命令：

| 字段 | 记录内容 |
|---|---|
| 背景/需求 | 为什么要改，原流程的限制是什么 |
| 现象 | 用户可见结果、Stage、错误文本、时间线 |
| 证据 | 日志、commit、镜像、配置、状态组合 |
| 根因 | 技术机制与触发条件，不写成“配置问题”泛称 |
| 排除项 | 哪些相似问题已证明不是根因 |
| 最终方案 | 代码、配置、命令和设计取舍 |
| 验证 | 正常、失败、并发、回滚场景怎样证明 |
| 预防 | fail-fast、语义校验、监控、测试或升级计划 |
| 可复用原则 | 可以迁移到其他项目的一句话判断 |

## 17. 当前代码审视与待改进点

### 17.1 已经合理的设计

- Shared Library 无状态方法通过参数、环境变量和工作区元数据跨 Stage 传递信息。
- 多服务显式安全校验路径、服务名、顺序和 Jar 唯一性，失败尽量前置。
- GitOps 锁放在 Library 内，所有调用者自动获得保护，避免靠每个 Jenkinsfile 自觉实现。
- 前端日志归档在 Stage post 中，失败证据不会因 Pipeline 中断丢失。
- Argo CD 等待不占用 GitOps 锁，兼顾一致性与并发效率。

### 17.2 需要进入下一轮的代码项

| 优先级 | 问题 | 当前影响 | 建议方向 |
|---|---|---|---|
| P0 | 前端缺少制品语义校验 | `dist/undefined` 仍可能退出 0 | 增加可配置 requiredPaths/forbiddenPattern |
| P0 | 多服务缺少聚合状态模型 | 失败通知难区分已完成和未开始服务 | 记录 selected/built/updated/healthy/failed |
| P1 | 前端 integration test 条件为 prod | 当前只映射 test/stage，Stage 实际不可达 | 明确是 stage 冒烟还是未来 prod 占位 |
| P1 | 普通 Java Unit Test Stage 是占位 | UI 容易误认为已独立执行单测 | 放入真实命令或移除占位 |
| P1 | 多服务通知字段未被消费 | 邮件可能只显示最后一个镜像 | Notification 增加服务清单和逐项结果 |
| P1 | 单 Java 未统一 `disableConcurrentBuilds()` | 与前端/多服务模板不一致 | 按工作区和发布语义决定是否补齐 |
| P2 | Kustomize 备份目录可能累积 | 持久工作区可能占用磁盘 | 按保留数/时间清理备份 |
| P2 | Kaniko 缓存关闭 | 重复镜像构建较慢 | 完成缓存仓库和失效策略后再启用 |
| P2 | 老前端兼容参数较多 | 技术债持续积累 | 升级 Webpack/Sass/xlsx-style 和 lockfile |

### 17.3 未附源码的关键能力

本次附件没有包含 `DeployArgoCD/argoDeploy`、`Sonar`、`notifyUtils`、`Robot` 的实现。当前可以确认调用参数和外部行为，但完整知识库还需要补充：

- Argo 等待的 revision 判定、Sync/Health 条件、超时和 debug 收集规则。
- Sonar 自动路径探测、质量门等待和 webhook 回调机制。
- 前端日志转 HTML 的截断、转义和附件策略。
- Robot 测试集与项目/服务映射。

知识文档应该明确“代码已核对”与“根据调用契约推断”的边界，避免把推断写成实现事实。

### 17.4 建议的 Shared Library 测试分层

| 层级 | 目标 | 适合覆盖 |
|---|---|---|
| 纯 Groovy 单元测试 | 不启动 Jenkins，验证纯函数 | 分支规范化、服务解析、路径安全、顺序、布尔解析 |
| JenkinsPipelineUnit | 模拟 Pipeline steps | `error`、`withCredentials`、`lock` 调用顺序、env 写入 |
| 测试 Jenkins Job | 验证插件和容器真实行为 | Kubernetes Agent、GitLab status、Lockable Resources、archive |
| 集成环境端到端 | 验证交付链 | Harbor、Kustomize Git、Argo Sync/Health、通知 |

基础逻辑越早在纯测试中覆盖，越不需要通过真实发布验证 Groovy 错误。尤其应覆盖：

- Webhook 分支优先于手动默认参数。
- `refs/heads/release` 正确归一化。
- 未知服务、重复 deployOrder、`../` 路径被拒绝。
- `lockEnabled='false'` 真正解析为 false。
- 无 Kustomize 变化时仍返回非空 commit。

### 17.5 配置契约应从注释升级为校验

目前许多项目契约写在注释中，例如 Nginx root 与 packageMode 对齐、Application 必须 automated sync、`deployName` 等于 Kustomize images.name。下一步应尽量转化为机器校验：

- init 校验环境构建命令存在。
- frontend-build 校验 required paths 和 forbidden patterns。
- Kustomize commit 前执行 `kustomize build` 并确认目标镜像只出现预期次数。
- Argo wait 校验目标 revision/image，而不仅是最终 Healthy。
- 新项目接入脚本检查路径、Application 和 Registry 仓库命名关系。

原则是：能在 1 分钟内静态发现的问题，不应等到 10 分钟后的 Deployment Degraded 才发现。

### 17.6 环境与凭据硬编码治理

当前 Jenkinsfile 中仍包含 Registry、GitLab、Argo CD、Sonar 等地址，这是现阶段清晰可用的项目配置，但扩展到多集群后会重复。建议分三层管理：

1. 全局基础设施地址和默认凭据 ID：Jenkins Folder/Global 环境或 Library 配置。
2. 环境映射和 Argo Application：项目 Jenkinsfile，便于评审。
3. Secret 值：只保存在 Jenkins Credentials，不进入 Git 或日志。

不要为了“去硬编码”把所有值藏进不可见全局配置。项目所有者仍应在 Jenkinsfile 看到影响发布目标的关键映射。

### 17.7 生产就绪前的必补项

- GitOps devops -> prod 的 promotion/审批模型，保护 prod 分支。
- 生产 Argo RBAC、Sync Window、回滚权限和审计。
- 数据库变更的前向兼容与回滚策略。
- 镜像 digest/SBOM/漏洞扫描与签名策略。
- 多服务部分成功的明确处理和发布单状态。
- 配置/Secret 变更与镜像变更的关联审计。
- 发布 SLO、错误预算和回滚演练。

## 18. 排障方法：按交付链逐层收敛

遇到发布失败时，建议固定按以下顺序定位，不要一开始就进入 Pod：

1. 初始化：检查 `BUILD_TRIGGER_TYPE`、`APP_BRANCH`、`DEPLOY_ENV`、`ARGOCD_APP`。
2. 源码：检查短/完整 commit、分支、子模块状态、commons commit。
3. 构建：查看 Maven/前端完整日志与退出码，确认工具版本和实际命令。
4. 制品：检查 Jar 唯一性、dist 目录、文件名、tar 内容和大小。
5. 镜像：确认完整 Tag、Harbor 是否存在、Kaniko Context 是否正确。
6. GitOps：确认 overlay、`images.name`、Kustomize commit 和远端分支。
7. Argo CD：先看 OutOfSync/Synced，再看 Progressing/Degraded/Healthy。
8. Kubernetes：最后检查事件、镜像拉取、探针、日志和业务接口。

### 18.1 新项目接入验收清单

- 项目只创建一个 Job，Webhook 和手动构建都能触发。
- master/test、release/stage 映射与实际环境一致。
- Git SSH 凭据、旧 ssh-rsa 和子模块策略验证通过。
- Maven/Node 工具版本和缓存目录明确。
- 构建日志在失败场景仍能归档。
- 镜像 Tag 能唯一追溯到源码和环境。
- Kustomize `images.name` 与 Pipeline `appName/deployName` 一致。
- overlay 路径和 Argo Application 指向同一环境。
- Application 已开启与 `triggerSync=false` 相匹配的 automated sync。
- 并发测试确认同 Job 排队、跨 Job GitOps 锁排队。
- Argo Synced/Healthy 后执行至少一个业务冒烟验证。

### 18.2 普通 Java 新项目接入步骤

1. 从标准普通 Java Jenkinsfile 复制一份，只改项目声明，不复制新 Job 到每个分支。
2. 配置 `PROJECT`、`IMAGE_REPO`、`KUSTOMIZE_PATH`、Git URL、Sonar key 和两套 Argo App。
3. 确认仓库根目录 Dockerfile 能消费 Maven 产物，Maven 命令与 Profile 正确。
4. 在 Kustomize overlay 中确认 `images.name == PROJECT`。
5. 创建一个 Jenkins Job，设置 GitLab webhook，手动先跑 master/test。
6. 验证 commit -> image -> Kustomize commit -> Argo revision 的追溯链。
7. 再跑 release/stage，确认 Sonar 等待和 Robot 条件。
8. 制造一次 Maven 失败，确认完整日志/tail/邮件仍可用。
9. 与另一 Job 并发更新 GitOps，确认全局锁有效。

### 18.3 Java 多服务新仓库接入步骤

1. 确认根 POM 能全量编译，公共依赖来源明确。
2. 为每个可部署服务填写 `moduleDir/artifactGlob/imageName/deployName/kustomizePath/deployOrder`。
3. 默认 `APP_SERVICES` 只放常规 Push 需要发布的服务，不等于仓库全部模块。
4. 先调用 resolve/validate，不执行镜像，验证未知服务和顺序失败语义。
5. 全量 Maven 后运行 Jar 唯一性校验，检查 metadata 文件。
6. 构建一个服务，检查 Context 只有两个文件和 Harbor Tag 正确。
7. 再验证默认多服务顺序和部分失败行为。
8. 明确 commons/业务 commit Tag 规则，避免可变依赖覆盖镜像。
9. 为通知和发布中心预留逐服务状态输出。

### 18.4 前端新项目接入步骤

1. 确认使用 Node/pnpm 版本、package scripts、lockfile 和 dist 目录。
2. 配置 test/stage/prod build command Map，但只开放当前允许环境。
3. 标准项目保持 frozen；只有验证过的旧项目才加 hoist/legacy 参数。
4. 确认 submodule 是否存在，若 URL 为 HTTP 则配置临时 rewrite。
5. 对齐 `packageMode`、tar 内容、Dockerfile ADD 目标和 Nginx root。
6. 配置 required paths、最小大小和 undefined 禁止模式；在该能力落地前用 `postBuildCmd` 做项目级校验。
7. master/test 首次构建时下载并检查日志、dist.tar.gz 和镜像内容。
8. release/stage 验证环境变量，不允许只看 build exit code。
9. 访问入口页面和静态资源，确认 base/public path。
10. 验证失败邮件附件、同 Job 排队和跨 Job GitOps 锁。

### 18.5 发布失败后的 15 分钟处置流程

**第 0-3 分钟：定层。** 从 Stage 和 Argo 状态判断失败层；记录 Job、BUILD_NUMBER、业务 commit、CURRENT_IMAGE、Kustomize commit、Argo App。

**第 3-8 分钟：收证据。** 下载 Maven/前端日志和制品；检查 Harbor Tag、Kustomize diff、Argo summary/debug；若 Degraded 再抓 events/logs。

**第 8-12 分钟：判断影响。** 单服务还是多服务，是否已有服务 Healthy，是否有旧构建仍运行，是否涉及数据库/配置协议。

**第 12-15 分钟：选择动作。** 输入错误/代码错误则修复新构建；运行环境临时故障可安全重试；错误 GitOps commit 使用 revert；多服务部分成功由负责人确认继续或整体回退。

整个过程不要修改同一镜像 Tag、force push GitOps 分支或在没有记录的情况下直接 kubectl 改线上状态。

### 18.6 常用验证命令速查

```bash
# Git 与本次源码
git rev-parse --short=7 HEAD
git rev-parse HEAD
git submodule status --recursive

# Java 制品
find . -path '*/target/*.jar' -type f | sort
sha256sum <app.jar>

# 前端制品
find dist -maxdepth 3 -type f | head -n 100
find dist -print | grep -E 'undefined' && exit 1 || true
tar -tzf dist.tar.gz | head -n 100

# Kustomize
kustomize build --load-restrictor LoadRestrictionsNone <overlay>
git diff --cached
git log -n 5 --oneline

# Argo CD
argocd app get <app> --refresh
argocd app wait <app> --sync --health --timeout 900

# Kubernetes
kubectl -n <ns> get deploy,rs,pod -o wide
kubectl -n <ns> get events --sort-by=.lastTimestamp | tail -n 80
kubectl -n <ns> logs <pod> --all-containers --tail=300
```

### 18.7 回滚决策表

| 失败类型 | 首选动作 | 不建议动作 |
|---|---|---|
| 构建未产生镜像 | 修复代码/依赖后新构建 | 改 Kustomize |
| 镜像已推送但未改 Git | 修复并推新 Tag，或停止 | 删除历史镜像掩盖问题 |
| GitOps 已更新、Argo 未同步 | 修正 Application/权限，或 revert | 直接 kubectl apply 另一份 YAML |
| Synced + Degraded，镜像错误 | 修复新 Tag并更新 Git，必要时先 revert | 覆盖同一 Tag |
| 集群临时资源/网络故障 | 恢复环境后重新等待/安全重试 | 无证据重复完整发布 |
| 多服务部分成功 | 按兼容性决定继续或逐 commit revert | 假设流水线失败已自动回滚 |
| 数据库不可逆变更 | 走预先定义的数据恢复/前向修复 | 只回滚镜像 |
