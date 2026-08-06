# 前端流水线与 Kaniko 镜像构建

> 本文整理自已落地的 Jenkins + Argo CD 流水线实践；示例中的地址、仓库、应用和凭据标识均已脱敏。

## 10. 前端标准流水线

前端项目的关键改造是把“安装与编译”和“镜像封装”拆开。`frontend-tools` 生成 `dist`、日志和压缩制品，Kaniko 只把已生成的制品装入 Nginx 镜像。

### 10.1 构建与镜像职责分离

| 阶段 | 输入 | 输出 | 失败含义 |
|---|---|---|---|
| frontend-build | package.json、lockfile、源码、环境变量 | `dist/`、`dist.tar.gz`、build log | 依赖或业务编译失败 |
| build-image | Dockerfile、`dist.tar.gz` | Nginx 镜像 | 镜像上下文、Registry 或 Kaniko 失败 |
| deploy | 镜像地址、Kustomize overlay | GitOps commit、Argo 状态 | 配置更新或运行健康失败 |

分层后可以单独下载 dist 和日志验证，而不需要从失败镜像中反推前端构建过程。

### 10.2 环境构建命令参数化

Jenkinsfile 根据 `DEPLOY_ENV` 选择命令：

```groovy
def frontendBuildCmdMap = [
    test : env.FRONTEND_BUILD_CMD_TEST,
    stage: env.FRONTEND_BUILD_CMD_STAGE,
    prod : env.FRONTEND_BUILD_CMD_PROD
]

env.FRONTEND_BUILD_CMD = frontendBuildCmdMap[env.DEPLOY_ENV]
```

项目差异通过以下参数表达，而不是复制流水线：

- `installExtraArgs`：兼容特殊依赖布局。
- `preBuildCmd`：构建前生成配置或修正历史项目。
- `postBuildCmd`：构建后校验或调整产物。
- `frozenLockfile`：是否严格按 lockfile 安装。
- `packageMode`：压缩目录本身还是目录内容。

### 10.3 pnpm 缓存与 lockfile 策略

通用模板默认使用持久化 store、`--prefer-offline` 和 `--frozen-lockfile`。这适合依赖定义规范的新项目：lockfile 与 package.json 不一致时立即失败，防止 Jenkins 自动改依赖树。

旧项目可能没有有效 pnpm lockfile，或者 lockfile 与历史 package.json 不一致。此时可以暂时设置 `frozenLockfile: false`，但正确方向仍是重新生成并提交可复现的 lockfile，而不是长期删除 `package.json`、忽略依赖定义或每次随机解析最新版依赖。

### 10.4 旧项目的依赖提升与命令不可见

Node.js 12/npm 6 时代的项目迁移到 Node.js 22/pnpm 9 后，常见症状包括 `spawn vite ENOENT`、某些 CLI 在脚本中不可见、Peer Dependency 无法按旧 npm 方式提升。`demo-ui` 当前使用：

```groovy
FRONTEND_INSTALL_EXTRA_ARGS = "--shamefully-hoist"
frozenLockfile = false
```

`--shamefully-hoist` 是兼容旧依赖布局的过渡方案，让依赖更接近旧 npm 的扁平 node_modules。它解决的是依赖可见性，不是业务环境变量问题，也不应该作为所有新项目默认值。

### 10.5 Node 22、OpenSSL 3 与内存参数

旧 Webpack 使用的加密算法可能与 Node 22/OpenSSL 3 不兼容。项目 Jenkinsfile 增加：

```groovy
NODE_OPTIONS = "--max-old-space-size=8192 --openssl-legacy-provider"
```

无需在 `npm run` 或 `pnpm run` 命令中再次引用 `NODE_OPTIONS`。Declarative Pipeline 的 environment 会把它注入子进程，Node 启动时自动读取。因此它既会作用于 npm/pnpm 启动的 Node 进程，也会作用于 Webpack/Vite 子进程。

两个参数解决的问题不同：

- `--openssl-legacy-provider`：旧 Webpack 与 OpenSSL 3 的兼容层。
- `--max-old-space-size=8192`：提高 V8 堆上限，降低大项目构建 OOM 风险。

这两者都是旧项目过渡措施。长期方案仍是升级 Webpack、Sass、xlsx-style 等依赖。

### 10.6 `packageMode` 必须与 Nginx root 对齐

`frontendBuild` 支持两种压缩结构：

| 模式 | 打包命令 | 解压后的结构 | 适用 Nginx root |
|---|---|---|---|
| `directory` | `tar -czf dist.tar.gz dist` | `<target>/dist/index.html` | root 指向包含 dist 的父目录 |
| `contents` | `tar -czf dist.tar.gz -C dist .` | `<target>/index.html` | root 直接指向解压目录 |

如果模式与 Dockerfile/Nginx root 不一致，镜像构建可以成功，但访问会出现 404。这类问题必须在制品结构层验证，不能只看 Kaniko 退出码。

### 10.7 镜像标签为什么包含环境和构建号

前端同一 commit 在 test 和 stage 会读取不同环境变量，产物可能不同；同一环境也可能因依赖或参数变化重复构建。当前 Tag 为：

```groovy
env.IMAGE_TAG = "${APP_GIT_COMMIT}-${DEPLOY_ENV}-${BUILD_NUMBER}"
```

这避免 test/stage 共用同一 commit Tag，也避免重复构建覆盖历史镜像。与后端“源码 commit 即制品”不同，前端构建时环境注入更强，因此标签维度也不同。

### 10.8 Nginx 配置与镜像解耦

前端镜像保留统一 Nginx 运行环境，环境差异配置由 Kustomize `configMapGenerator` 生成 ConfigMap，再在 Deployment 中挂载。Dockerfile 不再 `ADD conf.d /etc/nginx/conf.d`。

```yaml
configMapGenerator:
  - name: demo-ui-nginx-conf
    files:
      - default.conf=nginx/conf.d/default.conf
```

`.dockerignore` 应放在业务项目 Docker build context 根目录、通常与 Dockerfile 同级；它不属于 tools 镜像，也不属于 pnpm 缓存目录。

### 10.9 `demo-ui` 的 `dist/undefined` 案例

release/stage 构建曾经出现 `dist/undefined/`，文件名类似 `*.undefined.<timestamp>.js`。加入 Node 22 兼容参数后构建命令可以退出 0，但错误产物仍被打入镜像，最终 Argo CD 显示 Synced 而 Deployment 长时间 Progressing/Degraded。

最终根因是 `.env.stage` 缺少 `vue.config.js` 实际读取的变量：

```dotenv
NODE_ENV=stage
VITE_PUBLIC_PATH=/demo-ui/
VITE_PROJECT_NAME=demo-ui
VUE_APP_PROJECT_NAME=demo-ui
VUE_APP_VERSION=0.1
```

`VITE_PROJECT_NAME` 不会自动替代 `VUE_APP_PROJECT_NAME`。项目混用了 Vite 风格的 `VITE_*` 与 Vue CLI 风格的 `VUE_APP_*`，而实际构建配置仍读取后者。

这个案例形成了非常重要的经验：退出码成功只证明命令完成，不证明产物语义正确。当前 `frontendBuild` 只校验 dist 目录存在，后续应增加项目可配置的语义校验，例如禁止路径出现 `undefined`，并要求目标目录 `dist/demo-ui` 存在。

### 10.10 `frontendBuild` 的执行顺序与 Shell 语义

前端通用步骤通过 `withEnv` 把 Groovy 参数转换为 Shell 环境变量，然后在 frontend-tools 容器中一次执行：

1. 删除旧 dist 和压缩包，创建日志和 pnpm store 目录。
2. 输出项目、分支、环境、短/完整 commit、时间与工作目录。
3. 输出 Node/npm/pnpm 版本。
4. 配置并显示 pnpm store。
5. 按开关组装 install 参数并执行 `pnpm install`。
6. 可选执行 `preBuildCmd`。
7. 执行按环境解析后的 build 命令。
8. 可选执行 `postBuildCmd`。
9. 检查 dist，输出文件预览。
10. 按 `packageMode` 生成 tar，并预览 tar 内容。

```bash
set -euo pipefail

{
    pnpm install ${INSTALL_ARGS}
    bash -lc "${FRONTEND_BUILD_CMD}"
    test -d "${DIST_DIR}"
    tar -czf "${DIST_TAR}" "${DIST_DIR}"
} 2>&1 | tee "${LOG_FILE}"
```

四个 Shell 选项分别代表：`-e` 遇到未处理的失败立即退出，`-u` 使用未定义变量时报错，`pipefail` 让管道中任一命令失败都使管道失败，`tee` 同时写控制台和日志。没有 `pipefail` 时，前端命令失败但 tee 成功，Stage 可能被误判为成功。

### 10.11 构建命令为什么在 init 阶段解析

```groovy
def frontendBuildCmdMap = [
    test : env.FRONTEND_BUILD_CMD_TEST,
    stage: env.FRONTEND_BUILD_CMD_STAGE,
    prod : env.FRONTEND_BUILD_CMD_PROD
]

String cmd = frontendBuildCmdMap[env.DEPLOY_ENV]
if (!cmd?.trim()) {
    error "未配置前端构建命令：DEPLOY_ENV=${env.DEPLOY_ENV}"
}
env.FRONTEND_BUILD_CMD = cmd.trim()
```

解析放在 init，而不是 build Stage，目的是尽早验证环境契约。若 release 已映射 stage，但项目没有 stage script，应在下载依赖之前失败。环境负责选择配置，构建函数只执行传入的确定命令，这也让 `frontendBuild` 不需要理解 master/release。

### 10.12 依赖安装参数的分层

| 参数 | 默认 | 适用场景 | 风险/退出条件 |
|---|---|---|---|
| `preferOffline` | true | 优先使用 PVC store，降低网络依赖 | store 损坏时需隔离验证 |
| `frozenLockfile` | true | 标准项目，可复现安装 | package.json 与 lock 不一致会按设计失败 |
| `installExtraArgs` | 空 | 老项目临时兼容 | 必须记录原因和移除计划 |
| `--shamefully-hoist` | 非全局默认 | 旧 npm 扁平依赖项目 | 可能掩盖未声明的直接依赖 |
| `--no-frozen-lockfile` | 非全局默认 | 旧项目迁移过渡 | 每次解析结果可能漂移 |

标准项目应提交 `pnpm-lock.yaml` 并保持 frozen。旧项目临时放宽时，要在 Jenkinsfile 明确配置，而不是改 Library 默认值；否则一个项目的技术债会扩散到所有项目。

### 10.13 `NODE_OPTIONS` 如何自动生效

Declarative `environment` 中设置：

```groovy
NODE_OPTIONS = '--max-old-space-size=8192 --openssl-legacy-provider'
```

Jenkins 会把它注入 Agent 进程环境。pnpm/npm 启动 Node 脚本时，Node 运行时自动读取 `NODE_OPTIONS`，所以 package.json 不需要写成 `node $NODE_OPTIONS ...`。验证可以在构建日志输出：

```bash
node -p 'process.env.NODE_OPTIONS'
node -p 'process.versions.openssl'
```

不要把 legacy provider 误当成 `dist/undefined` 的修复。前者处理运行时加密兼容，后者是构建配置变量缺失，两类问题可能连续出现，但根因独立。

### 10.14 Dockerfile、tar 结构和 Nginx root 的契约

当前前端 Dockerfile 只负责运行时封装，例如：

```dockerfile
FROM harbor.example.com/comm-images/nginx:stable
ADD dist.tar.gz /home/www/htdocs/framwork/
```

若 `packageMode=directory`，解压后是 `/home/www/htdocs/framwork/dist/index.html`，Nginx root 应指向该 dist；若 `packageMode=contents`，解压后直接是 `/home/www/htdocs/framwork/index.html`。三者必须联动：

```text
packageMode + Dockerfile ADD 目标 + nginx root = 最终 index.html 路径
```

`.dockerignore` 放在 Kaniko context 根目录，通常与 Dockerfile 同级。它控制发送给镜像构建器的文件，不属于 tools 镜像，也不应放到 pnpm 缓存目录。当前 Context 仍为项目目录，建议至少排除 `.git`、`node_modules`、非必要源码和日志；更彻底的方案是像多服务 Java 一样生成最小 Context，只包含 Dockerfile 和 dist.tar.gz。

### 10.15 前端制品语义校验改进方案

当前代码只有 `test -d "${DIST_DIR}"`。建议为 `frontendBuild` 增加以下可选参数：

```groovy
List requiredPaths = (cfg.requiredPaths ?: []) as List
String forbiddenPattern = cfg.forbiddenPattern ?: ''
long minDistBytes = (cfg.minDistBytes ?: 1) as long
```

对应 Shell 校验可以生成并执行：

```bash
test -d "${DIST_DIR}"
test "$(du -sb "${DIST_DIR}" | awk '{print $1}')" -ge "${MIN_DIST_BYTES}"

test -f "${DIST_DIR}/demo-ui/index.html"

if find "${DIST_DIR}" -print | grep -E '(^|[./_-])undefined([./_-]|$)'; then
    echo 'ERROR: dist contains undefined path or filename'
    exit 3
fi

tar -tzf "${DIST_TAR}" | grep -q '^dist/demo-ui/index.html$'
```

这段是下一阶段建议实现。通用 Library 不应写死 `hr`，而应由项目传 `requiredPaths: ['dist/demo-ui/index.html']`；`forbiddenPattern` 可默认检查 `undefined`。校验应在打 tar 前后各做一次，分别保证源目录和最终制品契约。

### 10.16 前端 Sonar 的接入位置

前端扫描应放在 checkout 之后、镜像构建之前。若扫描需要依赖解析，可以放在 frontend-build 后；若只做源码静态分析，可独立 Stage。建议沿用 Java 的门禁规则：master/test 提交分析但不等待，release/stage 等待 Quality Gate。

```groovy
stage('sonar scan') {
    steps {
        container('tools') {
            dir('app') {
                script {
                    boolean waitQG = env.APP_BRANCH == 'release'
                    devops.scanWithParams(
                        projectKey : env.SONAR_PROJECT_KEY,
                        projectName: env.SONAR_PROJECT_NAME,
                        sources    : 'src',
                        waitScan   : waitQG
                    ).start()
                }
            }
        }
    }
}
```

具体方法名和参数必须以现有 Sonar 类源码为准；本次附件未包含该实现，所以这里是接入设计，不是已核对的当前代码。

### 10.17 前端流程的验收用例

1. master 和 release 分别解析为 test/stage 命令，并生成不同环境 Tag。
2. 标准项目 lockfile 不一致时按预期失败；修复并提交 lockfile 后成功。
3. 旧项目使用 `--shamefully-hoist` 后 CLI 可见，但日志明确显示兼容参数。
4. build 命令失败时 Stage 失败，完整日志仍被 archive。
5. dist 为空、目录错误或包含 undefined 时由语义校验阻断。
6. `tar -tzf` 中的路径与 Dockerfile/Nginx root 对齐。
7. Kaniko 失败与前端 build 失败在 Stage 和日志中可区分。
8. 同 Job 重复触发时排队，test/stage 不再清理同一工作目录。
9. Argo Synced/Healthy 后验证实际入口 URL 和静态资源路径，而不是只看 Deployment。

## 11. Kaniko 镜像构建

### 11.1 当前实现

`Kaniko` 要求 Pipeline 显式传入镜像仓库、Tag、凭据、Dockerfile 和 Context。它从镜像仓库地址提取 Registry，绑定 Jenkins username/password 凭据，生成 `/kaniko/.docker/config.json`，然后执行：

```bash
/kaniko/executor \
  --context="${context}" \
  --dockerfile="${dockerfile}" \
  --destination="${repo}:${tag}" \
  --cleanup \
  --cache=false
```

成功后设置 `CURRENT_IMAGE`，失败时更新 GitLab commit status 并让 Pipeline 失败。构建整体使用 `retry(3)`，用来吸收 Registry 短暂抖动等偶发问题。

### 11.2 为什么不用 Docker daemon

Kaniko 可以在 Kubernetes Agent Pod 内直接构建并推送镜像，不需要挂载宿主机 Docker Socket，减少节点耦合和高权限风险。普通 Java 可以直接用业务仓库作为 Context；多服务项目和前端项目则先生成受控的最小 Context。

### 11.3 当前取舍

当前 `--cache=false` 优先保证行为简单和确定，但会牺牲重复构建速度。若以后启用 Kaniko 缓存，需要同时定义缓存仓库、过期策略、Dockerfile 分层规范和故障回退，不能只改一个开关。

### 11.4 Registry 鉴权实现

Kaniko 没有复用 Docker daemon 登录态，而是在容器内生成标准 Docker config：

```groovy
script.withCredentials([script.usernamePassword(
    credentialsId: this.credentialsId,
    usernameVariable: 'USERNAME',
    passwordVariable: 'PASSWORD'
)]) {
    def auth = script.sh(
        script: 'printf "%s:%s" "$USERNAME" "$PASSWORD" | base64 | tr -d "\\n"',
        returnStdout: true
    ).trim()
}
```

```json
{
  "auths": {
    "harbor.example.com": {
      "auth": "<base64(username:password)>"
    }
  }
}
```

Registry 从 repo 的第一个 `/` 之前提取，例如 `harbor.example.com/demo-project/demo-java-app` 得到 `harbor.example.com`。若 repo 没有命名空间或格式错误，认证配置也会错误，因此 `repo` 和 `credentialsId` 在执行前必须 fail-fast。

### 11.5 为什么 Tag 必须由 Pipeline 显式传入

旧实现容易在 kaniko 容器里通过 Git 状态猜 Tag，但多容器工作区、浅克隆、多仓库输入和前端环境构建会让这种猜测失真。当前 `Kaniko.kaniko(...)` 强制 `imageTag` 非空：

```groovy
if (!this.imageTag?.trim()) {
    script.error 'image tag is empty, please pass explicit image tag'
}
this.fullAddress = "${this.repo}:${this.imageTag}"
```

镜像版本策略由上层业务上下文决定：普通 Java 用业务 commit，多服务用业务+commons commit，前端用 commit+环境+BUILD_NUMBER。Kaniko 只执行“把指定 Context 构建为指定镜像”，职责更单一。

### 11.6 `retry(3)` 应该覆盖什么

当前 build 整体放在 `retry(3)` 中，可以吸收 Registry 短暂不可达、推送连接中断等偶发失败。但重试不能替代输入校验：Dockerfile 语法错误、Jar/dist 缺失、凭据错误每次都会失败，重复三次只会浪费时间。

后续可根据错误类型缩小重试范围：Context 校验和 Dockerfile 检查不重试，仅对 push/网络类失败重试；同时在每次 retry 前重置临时认证和错误状态。没有可靠分类前，保留当前简单策略，但应在日志中能看到每次尝试。

### 11.7 镜像阶段验证清单

- `repo`、`tag`、Dockerfile、Context 都非空且日志可见。
- Context 中没有凭据、`.git`、其他服务 Jar 或无关缓存。
- Registry host 与 config.json 中 key 完全一致，包括端口。
- 推送完成后 `CURRENT_IMAGE` 是完整地址。
- Harbor 中能按 Tag 找到镜像，并可记录 digest；未来推荐把 digest 写入通知或 GitOps。
- Dockerfile 基础镜像使用明确版本，避免上游 `latest` 变化导致同一输入不可复现。
