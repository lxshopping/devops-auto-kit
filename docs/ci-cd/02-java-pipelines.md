# 普通 Java 与单仓多服务流水线

> 本文整理自已落地的 Jenkins + Argo CD 流水线实践；示例中的地址、仓库、应用和凭据标识均已脱敏。

## 8. 普通 Java 项目标准流水线

普通 Java 项目 `demo-java-app` 的当前顺序是：初始化、检出、准备镜像、Maven、CI、Kaniko、GitOps/Argo、预上线集成测试、通知。

### 8.1 关键流程

1. `initPipeline` 解析分支并映射环境。
2. `gitCheckout` 检出到 `app/`，记录短/完整 commit。
3. 镜像地址设置为 `${IMAGE_REPO}:${APP_GIT_COMMIT}`。
4. `mvnPackage` 使用持久化 Maven 仓库完成 `clean package`。
5. Sonar 扫描：master 提交扫描后继续，release 等待质量门。
6. Kaniko 构建并推送镜像。
7. Kustomize 在锁内更新镜像，Argo CD 自动同步并等待健康。
8. release 分支执行 Robot 集成测试。
9. post 中无论成功或失败都发送统一通知。

### 8.2 Maven 日志为什么要在 Shared Library 采集

Maven 构建日志必须在命令执行位置采集，否则 Stage 失败后 post 只能看到 Jenkins 控制台，邮件无法获得完整上下文。`MavenBuild` 使用 `returnStatus` 保留退出码，先生成完整日志和 tail 文件、归档并更新 GitLab 状态，最后才调用 `error` 让流水线失败。

```groovy
int status = script.sh(
    script: """
        set +e
        ${command} >> '${logFile}' 2>&1
        status=\$?
        tail -n ${tailLines} '${logFile}' | tee '${tailFile}' || true
        exit \$status
    """,
    returnStatus: true
)

script.archiveArtifacts(
    artifacts: "${logFile},${tailFile}",
    allowEmptyArchive: true
)

if (status != 0) {
    script.error "mvn package failed, exitCode=${status}"
}
```

这段实现体现了重要顺序：先保留证据，再传播失败。否则失败发生得越早，可用于排查的信息反而越少。

### 8.3 Sonar 质量门策略

`waitScan=false` 不代表未扫描，而是“提交扫描任务但不等待质量门”。当前策略是：

- master/test：优先反馈速度，不阻塞后续镜像和部署。
- release/stage：等待质量门，质量不达标停止镜像构建或发布。

该策略属于环境门禁，不应在 Sonar 工具类中写死；Jenkinsfile 根据分支显式传入最清晰。

### 8.4 当前代码中的一个占位点

`CI` 并行分支中的 `Unit Test` Stage 目前只执行 `printenv`，真正的测试是否运行取决于前面的 `mvn clean package`。因此它更像流水线占位符，而不是独立测试能力。后续应选择一种明确方式：要么把真实单测命令放入该 Stage，要么删除/重命名占位 Stage，避免界面显示造成误解。

### 8.5 普通 Java Jenkinsfile 的最小项目配置

普通 Java 项目 Jenkinsfile 只需要声明项目差异：

```groovy
environment {
    IMAGE_REPO       = 'harbor.example.com/demo-project/demo-java-app'
    IMAGE_CREDENTIAL = 'registry'
    PROJECT          = 'demo-java-app'
    KUSTOMIZE_PATH   = 'apps/demo-java-app'

    APP_GIT_URL        = 'git@git.example.com:demo/business-platform/demo-service/demo-java-app.git'
    APP_GIT_CREDENTIAL = 'gitlab-ssh-key'

    KUSTOMIZE_GIT_URL        = 'git@git.example.com:demo/devops/kustomize.git'
    KUSTOMIZE_GIT_CREDENTIAL = 'gitlab-ssh-key'
    KUSTOMIZE_BRANCH         = 'devops'
}
```

接入时最容易出错的不是 Groovy 语法，而是四个名称不一致：`PROJECT`、Harbor 仓库名、Kustomize `images.name`、Argo Application 名。建议在首次构建前做静态核对，不要等 `kustomize edit set image` 后才发现替换目标错误。

### 8.6 Maven 构建封装的失败语义

`MavenBuild` 故意用 `set +e` 和 `returnStatus: true` 接管退出码：

```groovy
int status = script.sh(
    script: """
        set +e
        ${command} >> '${logFile}' 2>&1
        status=\$?
        tail -n ${tailLines} '${logFile}' | tee '${tailFile}' || true
        exit \$status
    """,
    returnStatus: true
)
```

如果直接使用默认 `sh`，Maven 非零退出会立刻抛异常，后面的读取 tail、写 `BUILD_TASKS`、归档和更新 GitLab status 都不会执行。当前顺序是：

1. 执行 Maven 并保留真实退出码。
2. 生成 tail，读取并清理过长内容。
3. 写入 `MVN_BUILD_STATUS`、`MVN_BUILD_EXIT_CODE` 和通知摘要。
4. 归档完整日志与 tail。
5. 更新 GitLab commit status。
6. 最后 `script.error(...)` 恢复失败语义。

这就是“收集证据”和“让流水线失败”之间的正确顺序。任何构建封装都可以复用同一模式。

### 8.7 Maven 命令与缓存策略

默认命令由 `mvnBin + goals + repoLocal + extraArgs` 组装，也允许调用方传完整 `command` 覆盖。实际项目中应优先用结构化参数，只有 Wrapper、特殊 Shell 或多命令链才使用 `command`。

```groovy
devops.mvnPackage(
    goals: "clean package -U -Dmaven.test.skip=true -P${env.DEPLOY_ENV}",
    repoLocal: '/home/jenkins/.m2/repository',
    tailLines: 10000,
    maxChars: 100000,
    updateGitlabStatus: true
)
```

注意 `-Dmaven.test.skip=true` 会跳过测试编译和执行；`-DskipTests` 通常只跳过执行。当前多服务流程选择前者是发布效率取舍，不应在文档或 Stage 名称中宣称已经完成独立单测。若质量门需要覆盖率，应重新设计测试和扫描顺序。

### 8.8 Sonar 与 Maven 的关系

普通 Java 与多服务项目都先完成 Maven 编译，再运行 Sonar。多服务项目的 Sonar 扫描整个 Reactor，不随 `APP_SERVICES` 缩小，因为选择参数只描述发布集合，不描述代码质量检查范围。

```groovy
boolean needWaitQG = env.APP_BRANCH == 'release'

devops.scanJava(
    projectKey     : env.SONAR_PROJECT_KEY,
    projectName    : env.SONAR_PROJECT_NAME,
    projectVersion : "${env.APP_GIT_COMMIT}-${env.BUILD_NUMBER}",
    autoDetectPaths: true,
    waitScan       : needWaitQG
).start()
```

`waitScan=false` 只是不阻塞等待 Quality Gate，不代表扫描没有发起。排查时应分别确认 scanner 退出码、Sonar 项目是否收到分析、质量门是否等待和 webhook 是否能回调 Jenkins。

### 8.9 普通 Java 下一步建议代码

当前普通 Java Jenkinsfile 没有统一使用 `disableConcurrentBuilds()`，而前端和多服务模板已经使用。若同一个 Job 的重复构建会共享 workspace、覆盖相同环境或造成旧 revision 后发布，建议补齐：

```groovy
options {
    timeout(time: 20, unit: 'MINUTES')
    gitLabConnection('gitlab')
    skipDefaultCheckout(true)
    disableConcurrentBuilds(abortPrevious: false)
    buildDiscarder(logRotator(
        numToKeepStr: '30',
        artifactNumToKeepStr: '10'
    ))
}
```

`abortPrevious: false` 表示新构建排队而不是中断旧构建。若未来希望“新 commit 自动取消同 Job 的旧 commit”，需要先评估旧构建是否已经推送镜像或修改 GitOps，不能只为节省时间直接开启中断。

## 9. Java 单仓多子项目流水线

### 9.1 核心矛盾：编译范围与发布范围不是一回事

一个仓库包含多个 Maven 模块时，最容易犯的错误是让“本次要发布哪些服务”直接决定 Maven 只编译哪些模块。实际项目中模块之间存在父 POM、公共模块和间接依赖，使用 `-pl/-am` 很容易因为 Reactor 关系、模块命名或外部 commons 版本导致编译不完整。

最终方案明确分成两层：

- Maven 从根 POM 全量编译整个 Reactor，保证依赖完整和编译结果一致。
- `APP_SERVICES` 只控制哪些服务验证 Jar、制作镜像和更新部署。

这条原则同时解决了“全量构建太浪费镜像”和“按模块构建容易缺依赖”的冲突。

### 9.2 `SERVICE_CONFIG` 是项目发布元数据

多服务的差异继续留在 Jenkinsfile，Shared Library 不知道具体有哪些业务服务。每项配置包含：

| 字段 | 含义 | 使用位置 |
|---|---|---|
| `moduleDir` | 模块目录 | 校验目录和 POM |
| `artifactGlob` | 可执行 Jar 匹配规则 | 产物唯一性校验 |
| `imageName` | Harbor 镜像名 | 镜像仓库拼接 |
| `deployName` | Kustomize `images.name` 和部署名 | `set image`、Argo App 命名 |
| `kustomizePath` | 应用根目录 | 自动追加 `overlays/<env>` |
| `deployOrder` | 部署顺序 | 串行发布排序 |

```groovy
def SERVICE_CONFIG = [
    'config-service': [
        moduleDir   : 'config-service',
        artifactGlob: 'config-service/target/config-service-*.jar',
        imageName   : 'config-service',
        deployName  : 'config-service',
        kustomizePath: 'apps/demo-microservices/config-service',
        deployOrder : 10
    ],
    'gateway-service': [deployOrder: 20, /* 其他字段省略 */],
    'auth-service'   : [deployOrder: 30, /* 其他字段省略 */]
]
```

服务清单属于项目事实，因此放在 Jenkinsfile 合理；解析、排序、安全校验和执行属于通用能力，因此放在 `JavaMultiServiceRelease`。

### 9.3 完整执行链

1. 解析 `APP_SERVICES`：支持逗号和换行，去重，拒绝未知服务。
2. 按 `deployOrder` 排序并禁止重复顺序值。
3. 先检出并 `mvn clean install` 外部 `common-lib`。
4. 再检出业务仓库，从根 POM 全量 `clean package`。
5. Sonar 对整个 Reactor 扫描，不随服务选择缩小。
6. 校验每个目标服务的目录、POM、Dockerfile 和唯一可执行 Jar。
7. 为每个服务生成只有 `Dockerfile` 和 `target/app.jar` 的独立 Kaniko Context。
8. 串行构建所选服务镜像。
9. 按部署顺序逐个更新 Kustomize，并逐个等待对应 Argo Application Healthy。

### 9.4 为什么镜像 Tag 包含两个仓库 commit

多服务项目先安装 `common-lib`，业务 Jar 的字节内容同时受 commons 和业务仓库影响。如果镜像只使用业务仓库 commit，同一业务 commit 在 commons 变化后可能覆盖旧镜像 Tag。

当前使用：

```groovy
env.IMAGE_TAG = "${env.APP_GIT_COMMIT}-${env.COMMONS_GIT_COMMIT}"
```

这个 Tag 建立了二元可追溯关系，能回答“业务代码没变，为什么镜像变了”。

### 9.5 产物唯一性校验

`artifactGlob` 不能直接拿第一个匹配文件，因为 Maven target 中可能同时存在主 Jar、sources、javadoc 和 original。Shared Library 使用 `find` 排除这些辅助文件，并强制匹配数量等于 1：

- 0 个：说明模块未编译、路径错误或 Profile 不匹配。
- 多个：说明通配规则过宽，不能确定应该发布哪个 Jar。
- 1 个：把确定路径写入 `.jenkins-artifacts/<service>.jar-path`，供后续镜像阶段复用。

这种“先校验、后消费”的元数据方式，避免镜像阶段再次用不同逻辑猜 Jar。

### 9.6 独立 Kaniko Context

每个服务的镜像上下文只包含两个文件：

```text
.docker-context/<service>/
├── Dockerfile
└── target/app.jar
```

代码还会校验文件数必须等于 2。这样可以防止把整个多模块仓库、其他服务 Jar、`.git` 或缓存意外发送给 Kaniko，减少上下文体积并提升镜像可预测性。

### 9.7 串行部署与部分失败语义

服务按 config-server、gateway、auth 顺序发布，每个服务都执行“Kustomize 更新 -> Argo CD 等待”。如果 gateway 失败，auth 不再开始，但 config-server 可能已经成功。跨多个 Argo Application 的发布不是原子事务，不能假设流水线失败会自动整体回滚。

因此多服务发布需要明确记录以下状态：selected、image-built、kustomize-updated、healthy、failed、not-started。当前代码主要通过 Stage、`CURRENT_RELEASE_SERVICE` 和日志体现，尚未形成完整聚合状态模型，这是后续通知和发布中心接入的重要改进点。

### 9.8 CPS 兼容经验

Jenkins Pipeline 会把 Groovy Closure 转换为 CPS 对象，复杂排序 Closure 曾容易触发 `CpsClosure2 mismatch`。当前代码用显式循环实现插入排序，避免在可序列化 Pipeline 状态中执行不兼容 Closure。知识点不是必须手写排序，而是 Shared Library 中涉及 Pipeline CPS 时，要优先选择行为确定、序列化边界清楚的实现。

### 9.9 需求是怎样被拆成两个集合的

原需求同时包含两句话：“Push 默认发布三个服务”和“手动可以任意选择服务”。如果直接根据选择结果执行 `mvn -pl`，就把 UI 参数变成了 Maven 依赖图控制器。最终设计把集合拆开：

| 集合 | 来源 | 范围 | 作用 |
|---|---|---|---|
| 编译集合 | 根 `pom.xml` Reactor | 全部 Maven 模块 | 生成一致、完整的模块产物 |
| 发布集合 | `APP_SERVICES` + `SERVICE_CONFIG` | 已登记且本次选择的服务 | Jar 验证、镜像构建、Kustomize、Argo |

这种拆法有明确代价：即使只发 auth，也会编译全部模块；但它换来了编译确定性、简单接入和更低的依赖误判风险。只有在全量编译耗时成为有数据支撑的瓶颈后，才值得基于 Maven Reactor 图或变更路径做增量优化。

### 9.10 `APP_SERVICES` 解析与安全校验

参数允许换行或英文逗号，解析逻辑会去空白、去重并保留首次出现顺序：

```groovy
String[] tokens = value.split('[,\\r\\n]+')
for (String token : tokens) {
    String serviceName = token.trim()
    if (serviceName && !serviceNames.contains(serviceName)) {
        serviceNames.add(serviceName)
    }
}
```

随后执行四层校验：

1. 至少选择一个服务。
2. 服务必须存在于 `SERVICE_CONFIG`，未知值直接失败并输出允许列表。
3. 服务名只允许字母、数字、点、下划线和中划线，阻断命令/路径注入。
4. 每个服务配置字段完整，且 `deployOrder` 是唯一整数。

路径还会拒绝绝对路径、反斜杠、空字符、换行和 `..` 父目录跳转。虽然 `SERVICE_CONFIG` 来自受控 Jenkinsfile，不是公网输入，这些校验仍然有价值：它把配置错误变成 init/validate 阶段的可读错误，而不是在 Shell 中产生不可预测行为。

### 9.11 为什么显式校验 `deployOrder` 唯一

排序不能只保证“能排”，还要保证业务顺序没有歧义。若 gateway 和 auth 都配置 20，Groovy 排序仍会给出一个结果，但这个结果可能依赖 Map/输入顺序。当前代码维护 `deployOrderOwners`，发现重复立即失败：

```groovy
if (deployOrderOwners.containsKey(orderKey)) {
    fail("deployOrder 重复：deployOrder=${deployOrder}, " +
         "services=${deployOrderOwners[orderKey]},${serviceName}")
}
```

这体现了配置系统的重要原则：不要为错误配置“猜一个可运行结果”，应在影响发布前拒绝歧义。

### 9.12 外部 commons 仓库为何进入镜像版本

多服务流程先检出 commons，再执行：

```groovy
devops.mvnPackage(
    goals: 'clean install -U -Dmaven.test.skip=true',
    repoLocal: '/home/jenkins/.m2/repository'
)
```

业务仓库编译时会消费刚安装到 Maven 本地仓库的 commons。此时构建输入实际是二元组：

```text
(businessCommit, commonsCommit) -> applicationJar -> image
```

因此 `IMAGE_TAG=${APP_GIT_COMMIT}-${COMMONS_GIT_COMMIT}` 比只用业务 commit 更准确。更进一步的改进是记录完整 commit、POM 版本和依赖清单，并考虑把 commons 发布到正式 Maven 私服，而不是由每次业务流水线先 install；但在当前约束下，双 commit Tag 已解决覆盖和追溯问题。

### 9.13 产物校验为何写元数据文件

产物校验阶段将唯一 Jar 路径写入：

```text
.jenkins-artifacts/auth-service.jar-path
```

镜像阶段只读取这个文件，不再执行第二次 glob。这样有三个好处：

- 校验和消费使用同一结果，避免两个 Stage 的匹配规则漂移。
- 日志能明确显示最终选择了哪个 Jar。
- 后续可以扩展写入 SHA-256、大小、Maven GAV 等制品元数据。

推荐下一步在校验后增加：

```bash
sha256sum "${JAR_FILE}" > "${ARTIFACT_METADATA_DIR}/${SERVICE_NAME}.jar.sha256"
unzip -p "${JAR_FILE}" META-INF/MANIFEST.MF | sed -n '1,80p'
```

这属于改进方案，不是当前已上线代码。它可以在镜像异常时证明“进入 Context 的 Jar 是否就是预期字节”。

### 9.14 最小 Kaniko Context 的安全价值

当前代码复制项目根 Dockerfile，并把已验证 Jar 固定命名为 `target/app.jar`：

```bash
mkdir -p "${DOCKER_CONTEXT}/target"
cp "${DOCKERFILE_PATH}" "${DOCKER_CONTEXT}/Dockerfile"
cp "${JAR_FILE}" "${DOCKER_CONTEXT}/target/app.jar"

CONTEXT_FILE_COUNT=$(find "${DOCKER_CONTEXT}" -type f | wc -l)
test "${CONTEXT_FILE_COUNT}" -eq 2
```

这等于为所有子服务定义统一 Dockerfile 契约：Dockerfile 不再知道 Maven 模块目录和版本号，只消费 `target/app.jar`。如果未来不同服务需要不同 JVM 参数，优先通过运行时环境变量、Deployment 配置或显式服务级 Dockerfile 字段表达，不要在 Shell 中按服务名写大量 if/else。

### 9.15 多服务发布不是数据库事务

当前发布顺序为：每个服务先更新自己的 Kustomize，再等待自己的 Argo Application。假设三个服务状态如下：

| 服务 | 镜像 | Kustomize | Argo | 最终状态 |
|---|---|---|---|---|
| config-server | 成功 | 成功 | Healthy | 已发布 |
| gateway | 成功 | 成功 | Degraded | 发布失败但 Git 已更新 |
| auth | 成功 | 未开始 | 未开始 | 未发布 |

流水线整体是 FAILURE，但不能简化成“三个服务都失败”。重跑前要决定：只重跑 gateway/auth、整体 revert，还是修复 gateway 后继续。当前代码通过停止后续循环避免扩大失败面，但还缺少逐服务状态表。

推荐的状态模型：

```groovy
releaseState[serviceName] = [
    selected          : true,
    imageBuilt        : false,
    kustomizeCommit   : '',
    argoSync           : '',
    argoHealth         : '',
    finalStatus        : 'PENDING',
    error              : ''
]
```

每一步完成后更新，并在 `post { always }` 中输出聚合摘要。未来发布中心也可以直接消费这组结构。

### 9.16 多服务流程验收用例

1. Push master，不改参数：三服务按 10/20/30 顺序构建和发布。
2. 手动只填 `auth-service`：Maven 全量编译，只生成 auth 镜像并只更新 auth Application。
3. 输入逗号、换行和重复服务：解析结果去重并按 deployOrder 重排。
4. 输入未知服务：init 立即失败，并输出合法服务清单。
5. 修改某服务 `artifactGlob` 为无匹配：在镜像前失败，日志显示 0 个文件。
6. 制造两个可执行 Jar：唯一性校验失败并列出全部匹配。
7. 删除 Dockerfile：validate release services 失败，不进入 Maven/镜像。
8. 同时启动两个不同项目 Job：构建阶段并行，Kustomize 临界区排队。
9. gateway Argo 失败：auth 不开始，通知能指出当前失败服务；这是当前需要增强的验收项。

### 9.17 面试表达：如何讲清这个改造

建议按“矛盾—拆分—校验—结果”回答：单仓多模块既要求保证 Maven 依赖完整，又要求人工选择少量服务发布；我把编译范围与发布范围拆开，根 POM 全量 Reactor 编译，`APP_SERVICES` 只控制产物校验、最小 Context、镜像和 GitOps；再通过服务白名单、路径安全、Jar 唯一性和发布顺序校验把配置错误前置。最终 Push 可默认发三项，手工可选任意已登记服务，并保持镜像与两个仓库 commit 的可追溯关系。
