---
layout: default
title: 07b — GKE 镜像仓库（Artifact Registry）
---

# 第 07b 章：GKE — 镜像仓库（registry.tf + Docker 工作流）

> 上一章：[07 — 存储与 IAM](07-gke-storage-iam.html) · [章节索引](./)

到这里你已经有：

- 网络（05）
- 集群（06）
- bucket + GSA + Workload Identity（07）

但 K8s 跑应用得要**容器镜像**。镜像放哪？—— **Artifact Registry**（GCP 的私有 Docker 仓库，简称 AR）。

这章看一个文件 + 一段命令工作流：

- `registry.tf` —— 用 Terraform 把 AR 仓库声明出来（连同它的拉取权限）
- 然后命令行 `docker build → docker push → kubectl apply`，让镜像跑起来

如果 17 章的 `docker push` 报 `repository not found`，**99% 是 `registry.tf` 没 apply**。这章解释为什么。

## 1. registry.tf 全文

```hcl
# Artifact Registry for container images.
#
# One Docker repo per project — `enetpulse` houses event-job /
# inpage-job / job-creator / k8s-manager images. Workloads pull via
# Workload Identity (no JSON key on the host).

resource "google_artifact_registry_repository" "enetpulse" {
  location      = var.region
  repository_id = var.ar_repo_id
  description   = "Container images for the project"
  format        = "DOCKER"

  # cleanup_policies 故意没开 —— 加 "delete untagged > 7d" 会
  # 把历史镜像（往往 >30 GB）一并清掉。等你 audit 完老镜像、
  # 心里有底了再开。
}

# Compute Engine default SA（GKE Autopilot pull 镜像时用的身份）
# 需要 reader 权限。否则 Pod 起来会报 ImagePullBackOff。
resource "google_project_iam_member" "ar_reader_compute_default" {
  project = var.project_id
  role    = "roles/artifactregistry.reader"
  member  = "serviceAccount:${data.google_compute_default_service_account.default.email}"
}

data "google_compute_default_service_account" "default" {
  project = var.project_id
}
```

3 个块。一个 resource、一个 IAM grant、一个 data source。下面逐个看。

## 2. 第一个块：repo 本身

```hcl
resource "google_artifact_registry_repository" "enetpulse" {
  location      = var.region
  repository_id = var.ar_repo_id
```

`location` 跟 `var.region` 共用 —— 让 AR 跟集群同一个 region，**Pod pull 走 Google 内网**，不出 region。跨 region pull 不光慢，还要钱。

`repository_id` 跟 04 章里 `var.ar_repo_id`（默认 `my-app`）对上。

```hcl
  format = "DOCKER"
```

AR 是多格式的 —— 还能装 `MAVEN` / `NPM` / `PYTHON` / `APT` / `YUM` / `KFP`（Kubeflow 流水线）。我们只用 Docker。

```hcl
  description = "Container images for the project"
```

UI 里显示的人类可读说明。不影响行为。

### 2.1 cleanup_policies 为什么没开

AR 支持自动清理策略：

```hcl
# 这种东西故意没写
cleanup_policies {
  id     = "delete-untagged-after-7d"
  action = "DELETE"
  condition {
    tag_state  = "UNTAGGED"
    older_than = "604800s"
  }
}
```

听起来是好事 —— **但你第一次 apply 这个策略时，它会立刻去扫 repo，把所有符合条件的旧镜像直接删掉**。如果你的 repo 已经攒了几个月的历史镜像（很正常，几十 GB），那个 `terraform apply` 会变成一次大清理。

**正确做法**：先手动审一遍 repo 里有什么：

```bash
gcloud artifacts docker images list \
    europe-west3-docker.pkg.dev/${PROJECT_ID}/${AR_REPO_ID} \
    --include-tags
```

确认哪些可以删，再开 cleanup policy，或者用 `gcloud artifacts docker images delete` 手动清。

## 3. 第二个块：data source 查 default SA

```hcl
data "google_compute_default_service_account" "default" {
  project = var.project_id
}
```

`data` 块不创建任何东西 —— 它**查询**已经存在的 GCP 资源。这里查的是 "Compute Engine default service account"，每个 GCP project 都有一个，长这样：

```
123456789-compute@developer.gserviceaccount.com
```

这玩意是 GKE Autopilot 节点（VM）的运行身份。**节点用它去拉镜像**。

不能 hardcode 写死它的邮箱地址：每个 project 的项目编号不同，邮箱不一样。data source 替你查。

### 3.1 data 与 resource 的区别

| | `resource` | `data` |
|---|---|---|
| 行为 | 创建/更新/删除 | 只读查询 |
| state 变化 | apply 后写入 state | apply 后也写 state，但只是缓存 |
| 网络调用时机 | apply 阶段 | refresh 阶段（plan 之前） |
| 删除资源时 | destroy | 不会动外部资源 |

记忆口诀：**`resource` 管生死，`data` 管查询**。

## 4. 第三个块：IAM grant —— Pod 拉镜像的关键

```hcl
resource "google_project_iam_member" "ar_reader_compute_default" {
  project = var.project_id
  role    = "roles/artifactregistry.reader"
  member  = "serviceAccount:${data.google_compute_default_service_account.default.email}"
}
```

把上面查出来的 default SA 加进 `roles/artifactregistry.reader` 这个 role。

**没这一行的后果**：`terraform apply` 完，`docker push` 也成功了，但 `kubectl apply -f deployment.yaml` 之后 pod 永远是 `ImagePullBackOff`。`kubectl describe pod` 会看到：

```
Failed to pull image "europe-west3-docker.pkg.dev/.../my-app/event-job:v1.0.0":
  rpc error: code = Unknown desc = Error response from daemon:
  Head "https://europe-west3-docker.pkg.dev/v2/.../manifests/v1.0.0":
  denied: Permission "artifactregistry.repositories.downloadArtifacts" denied
```

排查这种问题的人会先怀疑镜像名拼错、tag 写错、registry 没建 —— 但本质往往是这一行 IAM grant 漏了。

### 4.1 为什么是 Compute SA，不是 07 章那个 GSA？

07 章你给应用建了 `event-agent-sa@...iam.gserviceaccount.com`，绑了 Workload Identity。**那个 SA 是给"Pod 跑业务"用的**，比如读 GCS bucket、调 BigQuery。

**镜像 pull 不走 Pod 的身份** —— 它发生在 Pod 启动之前，由**节点**（VM）发起。节点用的是 Compute Engine default SA。这两条权限路径完全独立。

```
┌─────────────────────────────────────────────────────────┐
│  GKE Autopilot 节点（GCE VM）                            │
│  身份: 123456-compute@developer.gserviceaccount.com     │   ← 这个 SA
│  ┌─────────────────────────────────────────────────┐   │   要 reader
│  │  containerd (节点的容器运行时)                  │   │   权限
│  │   │ docker pull europe-west3-docker.pkg.dev/... │───┼───→ AR
│  │   ▼                                             │   │
│  │  Pod (event-job)                                │   │
│  │   身份: KSA event-agent-sa → GSA event-agent... │───┼───→ GCS bucket
│  └─────────────────────────────────────────────────┘   │   (07 章那条路)
└─────────────────────────────────────────────────────────┘
```

### 4.2 改成 repo-scoped 而不是 project-scoped

上面用的是 `google_project_iam_member` —— **整个 project 范围**给 reader。如果你这个 project 里只有一个 AR repo，无所谓；如果有多个 repo，想精确到只某个 repo：

```hcl
resource "google_artifact_registry_repository_iam_member" "compute_reader" {
  project    = var.project_id
  location   = var.region
  repository = google_artifact_registry_repository.enetpulse.name
  role       = "roles/artifactregistry.reader"
  member     = "serviceAccount:${data.google_compute_default_service_account.default.email}"
}
```

Resource 类型从 `google_project_iam_member` 换成 `google_artifact_registry_repository_iam_member`，多了 `location` + `repository` 两个字段。**生产环境推荐这个**。Sandbox 用 project-scoped 简单一点。

## 5. 命令行工作流：build → push → pull

`terraform apply` 完，AR repo 是空的。要让 K8s 真能跑应用，还得三步：

```
本机                       AR                       GKE 节点
  │                         │                          │
  │ 1. docker build         │                          │
  │ 2. docker push ─────────→                          │
  │                         │                          │
  │                         │ 3. containerd pull ←─────│
  │                         │                          │
  │                         │                       Pod 启动
```

### 5.1 第一步：build 镜像

```bash
docker build \
    -t europe-west3-docker.pkg.dev/${PROJECT_ID}/${AR_REPO_ID}/event-job:v1.0.0 \
    ./app/
```

`-t` 后面就是镜像的"完整地址"。AR 镜像名遵循固定格式：

```
{location}-docker.pkg.dev/{project_id}/{repo_id}/{image_name}:{tag}
└──────────────┬──────────┘└─────┬────┘└───┬───┘└───┬────┘└─┬─┘
   AR 域名后缀（按 region）  你的项目  你的 repo  你的镜像  你的版本
```

**注意是 `pkg.dev` 不是 `gcr.io`** —— `gcr.io` 是上一代的 Container Registry，2024 后被 AR 取代。看到老教程写 `gcr.io/...` 的话那就是过时代码。

如果你 08 章定义了 output `ar_repo_url`，可以省心：

```bash
IMAGE_PREFIX=$(terraform output -raw ar_repo_url)
docker build -t ${IMAGE_PREFIX}/event-job:v1.0.0 ./app/
```

### 5.2 第二步：配 Docker auth

第一次 push 之前要让本地 docker 知道怎么向 AR 认证：

```bash
gcloud auth configure-docker europe-west3-docker.pkg.dev
```

这条命令实际上只改了一个文件：`~/.docker/config.json`。改之前：

```json
{
  "auths": {}
}
```

改之后：

```json
{
  "credHelpers": {
    "europe-west3-docker.pkg.dev": "gcloud"
  }
}
```

意思是：当 Docker 要 push 到 `europe-west3-docker.pkg.dev` 时，调一个外部程序叫 `docker-credential-gcloud` 拿临时 token，而不是用 username/password。这个 helper 程序在 `gcloud` 安装时一起装上的。

效果：你不需要在本地存任何长期 secret，token 每小时刷新一次，**push 出错时第一件事去查 `gcloud auth list` 看登的是不是预期账号**。

### 5.3 第三步：push

```bash
docker push europe-west3-docker.pkg.dev/${PROJECT_ID}/${AR_REPO_ID}/event-job:v1.0.0
```

push 成功后镜像就在 AR 里。可以用：

```bash
gcloud artifacts docker images list \
    europe-west3-docker.pkg.dev/${PROJECT_ID}/${AR_REPO_ID}
```

确认。

### 5.4 第四步：在 K8s 里引用

K8s manifest 里的 `image:` 字段写**完整 AR 地址**：

```yaml
spec:
  containers:
    - name: event-job
      image: europe-west3-docker.pkg.dev/my-cloud-project/my-app/event-job:v1.0.0
```

**不需要 `imagePullSecrets`**。这是新手常踩的坑：从 GHCR / Docker Hub / 其他注册表 pull 时往往要给 Pod 配 `imagePullSecrets`，拿一个 docker-registry 类型的 K8s Secret。从 AR pull 不需要，因为权限是节点级别的（4.1 节那条 IAM 链路）。

## 6. tag 命名约定 —— 永远不要用 `:latest`

```yaml
image: europe-west3-docker.pkg.dev/.../event-job:latest      # ❌ 不要
image: europe-west3-docker.pkg.dev/.../event-job:v1.0.0      # ✅ semver
image: europe-west3-docker.pkg.dev/.../event-job:abc1234      # ✅ git commit short SHA
image: europe-west3-docker.pkg.dev/.../event-job:2026-05-05  # ✅ 日期
```

**为什么 `:latest` 是坑**：K8s 默认 `imagePullPolicy: IfNotPresent` —— 节点上已经有这个 tag 就不重 pull。你 push 了新版本 `:latest`，但旧 pod 节点缓存里还是老的 `:latest`，**rollout 不会触发**。新 pod 调度到那个节点会拿到老镜像。

要么用 immutable tag（每次 build 给一个新版本号），要么强制 `imagePullPolicy: Always`（每次都从 AR 拉，慢且费流量）。生产里**统一用 immutable tag** 是更专业的做法。

## 7. 整个权限链路图（带本章新增的部分）

```
┌─────────────────────────── GCP project ──────────────────────────┐
│                                                                  │
│  ┌─ Artifact Registry ──┐         ┌─ GCS bucket ─┐              │
│  │  enetpulse repo      │         │  websites    │              │
│  └─────────┬────────────┘         └──────┬───────┘              │
│            │ reader                      │ objectViewer          │
│            ▼                             ▼                       │
│  ┌─ Compute default SA ─┐    ┌─ event-agent-sa GSA ──┐          │
│  │ ...-compute@...      │    │ event-agent@...       │          │
│  └─────────┬────────────┘    └────────┬───────────────┘          │
│            │ 节点身份                  │ Workload Identity       │
└────────────┼──────────────────────────┼──────────────────────────┘
             │                          │
             ▼                          ▼
        ┌─ GKE 节点 (GCE VM) ───────────────────┐
        │   containerd: pull 镜像              │
        │   ┌───────────────────────────────┐  │
        │   │ Pod                           │  │
        │   │   KSA event-agent-sa          │  │
        │   │   ┌───────────────────────┐   │  │
        │   │   │ container event-job   │   │  │
        │   │   │ (image 来自 AR)       │   │  │
        │   │   │ 业务调 GCS（Workload │   │  │
        │   │   │  Identity）          │   │  │
        │   │   └───────────────────────┘   │  │
        │   └───────────────────────────────┘  │
        └────────────────────────────────────────┘
```

**两条权限路径**：
- **横向蓝线 (本章)**：节点 SA → AR reader，让节点能拉镜像
- **纵向红线 (07 章)**：Pod KSA → GSA → bucket，让应用能读数据

少一条都跑不起来。`terraform apply` 时这两条都建好了；本章只解释横向那条。

## 8. 学完这一章应该会什么

- ✅ 看到 `google_artifact_registry_repository` 块知道在干啥（建容器镜像 repo）
- ✅ 知道 `data "google_compute_default_service_account"` 是查不是建
- ✅ 能 traceback 出 "Pod ImagePullBackOff" 是哪条 IAM 链路断了
- ✅ 知道节点 SA 拉镜像 ≠ Pod GSA 跑业务 —— 两条独立权限路径
- ✅ `gcloud auth configure-docker` 实际就改了 `~/.docker/config.json` 一个 credHelper
- ✅ 永远不在 K8s manifest 里写 `:latest`
- ✅ AR 镜像名格式：`{region}-docker.pkg.dev/{project}/{repo}/{image}:{tag}`
- ✅ 从 AR pull 不需要 `imagePullSecrets` —— 节点身份就够了

---

> 下一章：[08 — GKE Outputs](08-gke-outputs.html) · [章节索引](./)
