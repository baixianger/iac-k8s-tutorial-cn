---
layout: default
title: 17 — 端到端走一遍
---

# 第 17 章：端到端走一遍

> 上一章：[16 — Kubernetes Python SDK](16-k8s-python-sdk.html) · [章节索引](./)

把前面所有学的串起来 —— 从写第一行 `.tf` 到数据真正落进 GCS bucket。这一章是 GKE 路径的"操作手册级"演练。

> 用 Hetzner 的话，前几步换成 `hetzner-k3s create` 即可，后面 K8s 这部分完全一样。

## 1. 前置条件（Phase 0）

### 1.1 装好工具

```bash
# macOS
brew install hashicorp/tap/terraform
brew install --cask google-cloud-sdk
brew install kubernetes-cli           # kubectl
brew install kustomize                 # 单独装更新更快（kubectl 内嵌的版本经常落后）

# 验证
terraform version    # ≥ 1.6
gcloud version
kubectl version --client
kustomize version
```

### 1.2 GCP 认证

两个**完全独立**的认证流程：

```bash
# 流程 A: gcloud CLI 认证（给 gcloud 命令用）
gcloud auth login

# 流程 B: ADC 认证（给 Terraform 用）
gcloud auth application-default login
```

**两个都要做**。少一个，要么 gcloud 报错，要么 terraform 报 `oauth2: invalid_grant`。

### 1.3 创建 / 选好 GCP 项目

```bash
# 新建一个
gcloud projects create my-cloud-project   # 全球唯一
gcloud config set project my-cloud-project

# 启用必要 API
gcloud services enable container.googleapis.com    \
                       compute.googleapis.com       \
                       artifactregistry.googleapis.com \
                       storage.googleapis.com       \
                       iam.googleapis.com           \
                       cloudresourcemanager.googleapis.com

# 链接 billing（不链不能开 GKE）
# 在 Console 里给 project 关联一个 billing account
```

GCP 给你 $300 免费额度，跑这个教程绰绰有余。

## 2. Phase 1 —— Terraform apply

### 2.1 准备文件

把项目里 `cloud/deployments/stacks/gke/infra/` 下的所有 `.tf` 文件复制到本地（或者 `git clone` 项目本身）。

```bash
cd cloud/deployments/stacks/gke/infra/
ls
# main.tf  variables.tf  network.tf  cluster.tf  storage.tf  iam.tf  outputs.tf
```

### 2.2 自定义 terraform.tfvars

```bash
cp terraform.tfvars.example terraform.tfvars
vim terraform.tfvars
```

```hcl
# terraform.tfvars
project_id   = "my-cloud-project"
region       = "europe-west3"
cluster_name = "my-cluster"
admin_cidr   = "1.2.3.4/32"        # ← 改成你的 IP（curl ifconfig.me）
```

### 2.3 init + plan + apply

```bash
terraform init
# - 下载 hashicorp/google + hashicorp/google-beta provider
# - 初始化 .terraform/ 目录

terraform plan -out=tfplan
# 看仔细：
# Plan: 18 to add, 0 to change, 0 to destroy.
# - VPC + subnet + NAT
# - GKE Autopilot 集群
# - 2 个 GCS bucket
# - 1 个 Artifact Registry
# - 几个 GSA + IAM 绑定

terraform apply tfplan
# 输入 yes 确认
# 等 ~10 分钟（Autopilot 集群创建慢）
```

完成后会看到 outputs：

```
Outputs:

ar_repo_url       = "europe-west3-docker.pkg.dev/my-cloud-project/my-app"
audit_bucket      = "my-cloud-project-audit"
cluster_endpoint  = <sensitive>
cluster_name      = "my-cluster"
nat_egress_ip     = "35.198.X.Y"
region            = "europe-west3"
websites_bucket   = "my-cloud-project-websites-data"
```

### 2.4 拿 kubeconfig

```bash
gcloud container clusters get-credentials my-cluster --region europe-west3
# 自动写到 ~/.kube/config 里

kubectl get nodes
# Autopilot 节点是按需的；刚开集群可能为空
# NAME   STATUS   ROLES   AGE   VERSION
```

## 3. Phase 2 —— 镜像

### 3.1 配 Docker auth

```bash
gcloud auth configure-docker europe-west3-docker.pkg.dev
# 把 GCP 的 token 注入到 Docker 配置
```

### 3.2 build + push

```bash
# 假设你的应用 Dockerfile 在 ./app/
docker build -t europe-west3-docker.pkg.dev/my-cloud-project/my-app/event-job:v1.0.0 ./app/

# Push
docker push europe-west3-docker.pkg.dev/my-cloud-project/my-app/event-job:v1.0.0
```

## 4. Phase 3 —— K8s manifest

### 4.1 准备 manifest

假设项目布局：

```
cloud/deployments/k8s/
├── base/
│   ├── kustomization.yaml
│   ├── event-job-cronjob.yaml
│   ├── service-account.yaml
│   └── ...
└── gke/
    ├── kustomization.yaml          ← namespace, image, Workload Identity SA
    └── patches/
```

### 4.2 ServiceAccount 配 Workload Identity 注解

`base/service-account.yaml`：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: event-agent-sa
  namespace: browser-service
  annotations:
    iam.gke.io/gcp-service-account: event-job@my-cloud-project.iam.gserviceaccount.com
```

那个 GSA 已经在 Phase 1 由 `iam.tf` 创建好了，且 Workload Identity 绑定也已经存在。这里只需注解 K8s SA 让它"指向"那个 GSA。

### 4.3 CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: event-cron
  namespace: browser-service
spec:
  schedule: "5 0 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: event-agent-sa     # ← 用上面的 SA
          restartPolicy: OnFailure
          containers:
            - name: event
              image: europe-west3-docker.pkg.dev/my-cloud-project/my-app/event-job:v1.0.0
              env:
                - name: AUDIT_BUCKET
                  value: my-cloud-project-audit
                - name: WEBSITES_BUCKET
                  value: my-cloud-project-websites-data
```

### 4.4 Apply

```bash
kubectl apply -k cloud/deployments/k8s/gke/

# 验证
kubectl get cronjob -n browser-service
# NAME          SCHEDULE     SUSPEND   ACTIVE   LAST SCHEDULE   AGE
# event-cron    5 0 * * *    False     0        <none>          5s

kubectl get sa -n browser-service event-agent-sa -o yaml | grep gcp-service-account
# annotations:
#   iam.gke.io/gcp-service-account: event-job@my-cloud-project.iam.gserviceaccount.com
```

## 5. Phase 4 —— 触发 + 验证

### 5.1 立刻手动触发一次

```bash
kubectl create job -n browser-service manual-test \
  --from=cronjob/event-cron

kubectl get pods -n browser-service -l job-name=manual-test --watch
# manual-test-abc12   0/1   ContainerCreating   0   2s
# manual-test-abc12   1/1   Running             0   15s
# manual-test-abc12   0/1   Completed           0   45s    ✓
```

### 5.2 看日志

```bash
kubectl logs -n browser-service -l job-name=manual-test --tail=50
# 2026-01-01 00:00:00 INFO Starting event extraction
# 2026-01-01 00:00:05 INFO Got 770 events
# 2026-01-01 00:00:10 INFO Wrote audit to gs://my-cloud-project-audit/...
# 2026-01-01 00:00:15 INFO Done.
```

### 5.3 GCS 验证

```bash
gsutil ls gs://my-cloud-project-audit/multi-step/
# gs://my-cloud-project-audit/multi-step/{site}/...

gsutil cat gs://my-cloud-project-audit/multi-step/{site}/.../scheduled.json | jq '.events | length'
# 770
```

🎉 端到端打通：从 `terraform apply` 写下的 `.tf` 文件 → 集群 → Pod → GCS 数据。

## 6. 路上常见的 7 个错误

### 6.1 `terraform apply` 卡在 GKE 集群创建

```
google_container_cluster.autopilot: Still creating... [10m elapsed]
```

正常。Autopilot 创建确实要 8–12 分钟。10 分钟以上还卡，去 Console 看 cluster 状态。

### 6.2 `kubectl get nodes` 是空的

Autopilot 节点是 **按需创建** 的。没 Pod 调度时确实没节点。Apply 一个 Pod 后再看。

### 6.3 `ImagePullBackOff` 拉镜像失败

```
Events:
  Warning  Failed   ...   Failed to pull image "europe-west3-docker.pkg.dev/...": rpc error: code = NotFound
```

可能原因：

1. 镜像 push 失败 —— 重 `docker push`
2. 镜像 tag 拼错 —— `kubectl describe pod` 看具体 image 字符串
3. AR 权限不够 —— GKE 节点的 GSA 默认有 AR pull 权限，但如果手动改过 IAM 可能丢失
4. 跨项目 AR —— 不同 GCP 项目，要给 AR 配 cross-project IAM

### 6.4 `Forbidden: serviceaccount cannot create jobs`

RBAC 不够。看 ServiceAccount 上的 Role / RoleBinding：

```bash
kubectl describe rolebinding -n browser-service
```

补上对应的 verbs（`create`, `delete`, ...）。

### 6.5 GCS 写 403

Workload Identity 链路断了。沿着第 07 章那张图反查：

1. K8s SA 上有 `iam.gke.io/gcp-service-account` 注解？
2. GSA 上有 `roles/iam.workloadIdentityUser` 给那个 K8s SA？
3. GSA 上有 `roles/storage.objectAdmin` 在那个 bucket 上？

少哪个补哪个。

### 6.6 CronJob 触发了但 Job 没起

```bash
kubectl get cronjob event-cron -n browser-service -o yaml | grep -A 5 status
```

看 `lastScheduleTime` —— 如果 schedule 时间没到，等。如果时间过了但没 Job，看 controller manager 日志。常见是 namespace ResourceQuota / LimitRange 卡住。

### 6.7 Pod stuck Pending

```bash
kubectl describe pod my-pod | grep -A 20 Events
# Warning  FailedScheduling  ...  0/3 nodes are available: 3 Insufficient cpu
```

→ requests 太大，Autopilot 找不到合适节点。降 requests 或换更小 ComputeClass。

```
Warning  FailedScheduling  ...  pod has unbound immediate PersistentVolumeClaims
```

→ PVC 还没 bind。`kubectl get pvc` 看状态，可能 StorageClass 不对。

## 7. 一次完整 destroy

教程做完想清空 GCP 不再花钱：

```bash
# 1. 先删 K8s 工作负载（不删的话 Cluster 删不掉）
kubectl delete -k cloud/deployments/k8s/gke/

# 2. terraform destroy
cd cloud/deployments/stacks/gke/infra/
terraform destroy
# 确认 yes
# 等 5-10 分钟
```

destroy 完账单应该 → 0。`gcloud projects list` 项目还在（不收费），不想要可以 `gcloud projects delete`。

## 8. 学完这一章应该会什么

- ✅ 完整走过：本地 `terraform apply` → push image → `kubectl apply -k` → 数据落 GCS
- ✅ 触发 CronJob 用 `kubectl create job --from=cronjob/...` 测试
- ✅ Pod 出问题三板斧（describe / logs / exec）熟练
- ✅ 7 个常见错误闭眼能定位
- ✅ 知道完整 destroy 路径，避免持续烧钱

**整套教程的核心内容到此讲完**。下一章是常见问题集，遇到具体疑问回去查。

---

> 下一章：[18 — FAQ](18-faq.html) · [章节索引](./)
