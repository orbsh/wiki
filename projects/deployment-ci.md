# Deployment 与 CI-Data 项目

**Deployment：** `~/world/deployment`  
**CI-Data：** `~/world/ci-data`

全公司部署配置（基于 Helm）和 CI 系统数据配置。

## Deployment 结构

| 组件 | 路径 | 用途 |
|------|------|------|
| **helm-app** | `helm-app/` | 内部基于 Helm 的应用管理工具。应用值的真实来源。|
| **values/** | `values/<cluster>/<app>.yaml` | 源文件：手动编写，helm-app 的输入配置。多个部署共享一个文件。|
| `values/*.out.yaml` | `values/<cluster>/<name>.out.yaml` | 渲染输出归档。由 helm-app 自动生成，用于与之前版本进行 diff 比较。是有意的快照，非一次性临时文件。|
| **kustomize** | 各处 | 遗留配置，正在迁移到 Helm。|
| **ansible** | 各处 | K8s 集群配置。不用于应用部署。|
| **nomad** | 各处 | 实验性。非生产关键。|
| **argo CI** | `devops/ci/argo/` | 基于 Argo Workflows 的 CI 系统。|
| **wireguard** | `devops/wireguard.*.yaml` | WireGuard 隧道配置。|

## 源文件约定

**多个部署共享一个源文件。** 它们共同放置以共享基础设施配置（SSH 密钥、S3 凭证、集群设置等）。

典型结构（`values/<cluster>/<name>.yaml`）：
```yaml
namespace: <namespace>
apiserver_host: <ip>
ingress:
  provider: istio
  level: 3            # 0: 无 1: 路由 2: +Gateway 3: +Certificate
  issuerRef: letsencrypt
stage:
  cluster: xmh-2502
  pull_registry: 172.17.138.169:5000
  push_registry: wg.tunnel:5005
ssh:                  # 此文件中所有组件共享
  hostkey:
    ed25519: ...
  keys:
    warpgate:
      root: ...
components:           # 每个 key = 一个部署
  openai-be:
    stage: { git: 1, repo: xmh/api }
    image: { repo: ..., tag: '...' }
    ssh: [{ group: warpgate }]
    service:
      - name: api
        port: 8000
        containerPort: 8000
    replica: 1
```

## 文件命名约定

| 后缀 | 角色 | 示例 |
|------|------|------|
| `.yaml`（无后缀）| **源配置** — 手动编写 | `values/xmh/ai.yaml` |
| `.out.yaml` | **渲染输出** — 由 helm-app 生成 | `values/xmh/ai.out.yaml` |

## 域名推断规则

| 域名模式 | 集群 | 阶段 |
|---------|------|------|
| `*.xinminghui.com` | `xmh` | `prod` |
| `*.pre.xinminghui.com` 或 `*.pre.*` | `xmh` | `pre` |
| `*.s`（如 `registry.s`）| — | 内部/测试 |

## Gitea URL 模式

`http://gitea.s/<org>/<repo>` → org = `<org>`，app = `<repo>`

## 新应用入职 — Deployment

1. **从 Gitea URL 推断**：`http://gitea.s/<org>/<repo>` → org、app 名称
2. **从域名派生环境**：
   - `api-communicate.xinminghui.com` → 集群 `xmh`，阶段 `prod`
   - 无 `pre.` 前缀 → `prod`，有 `pre.` → `pre`，`.s` → 内部/测试
3. **查找现有源文件**：在 `values/<cluster>/` 中查找已包含相关部署的文件。例如，`ai/skills` 和 `ai/communicate-skills` 都属于 `ai` 组 → 同一个 `values/xmh/ai.yaml`。**追加，不要新建。**
4. **确认容器端口**：**必须询问用户**容器内部端口。**不要猜测。**
5. **在 `components:` 下添加组件条目**，遵循 `stage`/`image`/`ssh`/`service`/`env`/`replica` 模式。
6. **默认**：单次部署，除非另有指定。
7. 使用占位镜像——CI 系统稍后会推送正确的标签。

## 新应用入职 — CI/CD

1. 创建 `ci-data/<cluster>/<app>.yml`
2. 引用部署：
```yaml
deployments:
- git_id: 1
  repo: <org>/<repo>
  branch: release    # prod = release, pre = dev/master
  kind: kubernetes
  op: update
  notify: true
  payload:
    registry: wg.tunnel:5002
    tg:
    - cluster: xmh-2502
      namespace: app-xmh
      deployment: <deployment-name>
      container: app
      pull_registry: 172.17.138.169:5000
      remark: '[<service-name>](<domain>)'
```

## ci-data 文件组织

**通用规则**：每个项目一个文件 → `ci-data/<cluster>/<project>.yml`

**例外 — AI 组**：AI 相关项目分组在 `ci-data/xmh/ai.yml`（敏捷工作流、滚动发布、共享依赖）。

## CI 字段约定

- **`deployment`（在 `tg` 中）**：必须匹配 helm-app 源文件中的组件名称。
- **`dockerfile`**：默认 = `Dockerfile`。扩展名 `nu` 暗示 Nushell `bx` 框架（使用 `~/data/docker.io/xy` 中的 `xy` 构建器）。
- **`project` 字段（自动模式）**：如果存在，系统推断结构并自动生成 Dockerfile。不要写 `dockerfile` 字段。

## CI 系统架构

**引擎**：Argo Workflows。  
**逻辑**：基于 Python 的控制器（`devops/ci/argo/`）。稳定，前 Nushell 时代实现。

### 设计哲学：集中配置

CI 系统旨在**消除每个仓库的配置文件**（如 `.github/workflows`）。

- **控制平面**：所有逻辑和管道在 Argo 控制器中集中定义。
- **数据平面**：配置由两个 Git 仓库驱动：
  1. `ci-data`：定义每个项目的触发器、分支和仓库设置。
  2. `deployment`：定义应用结构和部署目标。

**优势**：
- **干净的仓库**：开发者仓库只包含源代码，无 CI 噪音。
- **一致的策略**：CI 逻辑更新全局应用，无需触碰单个仓库。
- **数据驱动**：CI 行为是外部配置数据的函数，而非硬编码在仓库文件中。

## 重要约定

1. **绝不在 deployment 或 ci-data 中自动提交**。用户手动处理所有提交。
2. ci-data 文件变更 = 数据库记录，不仅仅是代码变更。
3. Helm values 遵循 helm-app 约定，而非原生 Helm。
4. 文档和注释只用英文。

## 陷阱

- **不要猜测容器端口**：必须明确询问用户。
- **不要为新应用创建单独的源文件** — 追加到现有文件。
- **不要直接写 `.out.yaml`** — 这些是 helm-app 的渲染输出。源文件无 `.out` 后缀。手动写 `.out.yaml` 会产生陈旧、不正确的输出，与实际渲染状态偏离。
- **不要克隆仓库推断部署信息** — Gitea URL 直接给出 org/app。
- **不要混淆 ci-data 与 devops/ci/argo** — 不同的 CI 系统。
- **追加时不要覆盖现有组件** — 验证补丁目标唯一（使用最后一个组件的完整块作为 `old_string`）。
- kustomize = 遗留，正在迁移。nomad = 实验性。
