---
layout: default
title: 18 — FAQ
---

# 第 18 章：FAQ

> 上一章：[17 — 端到端走一遍](17-end-to-end.html) · [章节索引](./)

常见疑问 + 故障排查。按主题分组，遇到问题来这里查。

## Terraform

### Q1：Terraform vs Pulumi vs CloudFormation 有什么区别？

| 工具 | 写啥 | 跨云 | 适合 |
|---|---|---|---|
| Terraform | HCL | ✅ | 主流默认选择 |
| OpenTofu | HCL（同 Terraform） | ✅ | Terraform 1.5 fork，license MPL，多数场景互换 |
| Pulumi | TypeScript / Python / Go | ✅ | 喜欢真实编程语言（带类型系统、可单元测试） |
| CloudFormation | YAML / JSON | ❌（只 AWS） | 只用 AWS 且要深度集成 |
| ARM Templates | JSON | ❌（只 Azure） | 只用 Azure |

学 Terraform 最实用 —— 你就业市场上看到 90% 的 IaC 招聘要求都是它。

### Q2：terraform.tfstate 误删了怎么办？

灾难。两条路：

1. **从备份恢复**：`terraform.tfstate.backup` 是上一次 apply 前的快照。复制回去。
2. **`terraform import`**：把每个云上资源重新"认领"进 state（耗时、易错）。

例子：

```bash
terraform import google_compute_network.gke projects/my-cloud-project/global/networks/my-cluster-vpc
terraform import google_compute_subnetwork.gke projects/my-cloud-project/regions/europe-west3/subnetworks/my-cluster-subnet
# ... 重复每一个 resource
```

**生产化第一件事**：远程 state（GCS / S3）+ state 备份。

### Q3：plan 标了 `forces replacement`，能不能不重建？

不能。`forces replacement` 表示这个字段在云上**只读** —— 改它必须删了重建。

**唯一的解法是不要改这个字段**，或者改 var 时给一个临时新名字让旧的保留（手动 rename）。

### Q4：怎么并行 apply 多个 stack（GKE + Hetzner）？

每个 stack 一个目录、各自有自己的 `tfstate`。两个 terminal 同时 `terraform apply` 互不干扰。

```
stacks/
├── gke/infra/         ← 自己的 tfstate
└── hetzner/infra/     ← 自己的 tfstate
```

如果用远程 state，两个 stack 的 `backend "gcs"` 配不同 `prefix` 即可。

### Q5：怎么调试 `Error: Error 400` 这种神秘报错？

```bash
TF_LOG=DEBUG terraform apply 2>&1 | tee tf-debug.log
```

DEBUG 输出每个 API 请求 / 响应的细节。仔细 grep `Error` 上下文。

### Q6：Terraform 升级 provider 风险大不大？

中等。看 release notes，重点看 **breaking changes**。

```bash
# 锁定的是次版本，避免大版本升级
version = "~> 6.10"     # 允许 6.10.x，6.11+ 不允许
```

升级流程：

1. 改 version：`~> 6.10` → `~> 7.0`
2. `terraform init -upgrade`
3. `terraform plan`（看没有 breaking 的 schema 变化）
4. `terraform apply`

## Kubernetes — Pod 起不来

### Q7：Pod 一直 `Pending`

```bash
kubectl describe pod my-pod | grep -A 10 Events
```

最常见原因：

| Event 关键字 | 原因 | 解 |
|---|---|---|
| `Insufficient cpu` / `memory` | requests 太大 | 降 requests，或升 ComputeClass |
| `unbound immediate PersistentVolumeClaims` | PVC 没 bind | 看 `kubectl get pvc`，StorageClass 是否存在 |
| `node(s) had taints, that the pod didn't tolerate` | 节点有 taint | 加 tolerations 或换节点池 |
| `pod has unbound NodeSelector` | nodeSelector 不匹配任何节点 | 改 nodeSelector 或加 label 给节点 |

### Q8：`ImagePullBackOff`

```bash
kubectl describe pod my-pod | grep -A 5 'Failed.*pull'
```

| 错误关键字 | 原因 |
|---|---|
| `not found` / `manifest unknown` | image tag 不存在 / 拼错 |
| `unauthorized` | 私有镜像仓库认证失败，要配 imagePullSecrets |
| `dial tcp: i/o timeout` | 节点出网不通（NAT 配错？防火墙拦了？） |

### Q9：`CrashLoopBackOff`

容器进程不停崩。

```bash
kubectl logs my-pod                # 当前
kubectl logs my-pod --previous      # 上一次崩前的
kubectl describe pod my-pod         # exit code
```

退出码：

| Exit code | 含义 |
|---|---|
| 0 | 正常退出（但被 restartPolicy=Always 重启） |
| 1 | 应用错误（看日志） |
| 137 | OOMKilled —— 内存超限 |
| 139 | SIGSEGV —— 段错误 |
| 143 | SIGTERM —— 被 K8s 主动终止 |

### Q10：Pod `OOMKilled`

容器内存超过 `limits.memory`。三个解：

1. 升 `limits.memory`（拍脑袋加 50%）
2. 优化代码（找内存泄漏）
3. **配置层**：调小 cache / batch / 并发数，让进程少占内存

## Kubernetes — 集群运维

### Q11：ConfigMap 改了，Pod 没变化

Pod 不会自动 reload。两个解：

```bash
# 文件挂载方式：约 1 分钟自动同步（kubelet reconcile）
# 环境变量方式：永远不会刷新

# 手动触发 Pod 重建
kubectl rollout restart deployment/api
```

### Q12：CronJob 没按时跑

可能原因：

1. **时区**：默认 UTC，不是本地时间
2. **`concurrencyPolicy: Forbid`** + 上一次 Job 还在跑 —— 这次跳过
3. **`suspend: true`** —— 检查 `kubectl get cronjob -o yaml | grep suspend`
4. **controller-manager 卡住**（罕见）—— 重启 K8s 控制面

### Q13：Service 访问不通

```bash
# 1. Service 选到 Pod 了吗？
kubectl get svc hello -o yaml | grep selector
kubectl get pods -l <selector>     # 应该有 pod

# 2. Pod 是 Ready 吗？
kubectl get pods                    # READY 列要 1/1

# 3. 端口对吗？
kubectl get svc hello -o yaml | grep -A 3 ports
# Service 的 targetPort 必须等于 Pod 容器的 containerPort

# 4. 集群内 DNS 通吗？
kubectl run -it --rm dnstest --image=busybox -- nslookup hello.my-app
```

### Q14：删 Namespace 卡住 `Terminating`

通常是 finalizer 没清。

```bash
kubectl get ns my-app -o yaml | grep finalizers
# 看哪个 controller 还 hold 着

# 强删（慎用）：
kubectl get ns my-app -o json | \
  jq '.spec.finalizers = []' | \
  kubectl replace --raw /api/v1/namespaces/my-app/finalize -f -
```

## GKE 特定

### Q15：Workload Identity 不生效（Pod 拿不到 GCS 权限）

按这个顺序检查：

```bash
# 1. K8s SA 注解对吗？
kubectl get sa my-sa -n my-app -o yaml | grep gcp-service-account
# 应该有 iam.gke.io/gcp-service-account: <gsa-email>

# 2. GSA 上的 IAM 绑定有吗？
gcloud iam service-accounts get-iam-policy event-job@my-cloud-project.iam.gserviceaccount.com
# 应该有 roles/iam.workloadIdentityUser 给 [my-app/my-sa]

# 3. GSA 在目标 bucket 上有权限吗？
gsutil iam get gs://my-cloud-project-audit | grep event-job
# 应该有 roles/storage.objectAdmin

# 4. Pod 用了对的 SA 吗？
kubectl get pod my-pod -o yaml | grep serviceAccount
```

少哪个补哪个。

### Q16：GKE Autopilot 调度报 `ScaleUpFailed: Quota exceeded`

GCE 配额不够。最常见的是 `SSD_TOTAL_GB`（每个 Autopilot 节点 ~100GB pd-balanced，默认 500 GB → 4 节点上限）。

#### 查当前配额(命令选错了会一直查不出来)

```bash
# ❌ 错的：什么都不输出。
gcloud compute project-info describe \
  --format='value(quotas)' | grep -i ssd
```

为什么空?`project-info describe` 列的是**项目全局**配额(`CPUS_ALL_REGIONS` / `NETWORKS` / ...),而 `SSD_TOTAL_GB` / `CPUS` / `INSTANCES` / `IN_USE_ADDRESSES` 都是**区域级**配额,挂在每个 region 资源下面,不在项目根。

```bash
# ✅ 对的:查区域配额。
gcloud compute regions describe europe-west3 --format=json \
  | jq '.quotas[] | select(.metric|test("SSD|CPUS|INSTANCES|IN_USE_ADDRESSES"))'
```

输出形如:

```json
{ "limit": 500, "metric": "SSD_TOTAL_GB",     "usage": 0 }
{ "limit": 24,  "metric": "INSTANCES",        "usage": 0 }
{ "limit": 200, "metric": "CPUS",             "usage": 0 }
{ "limit": 8,   "metric": "IN_USE_ADDRESSES", "usage": 0 }
```

如果命令报 `Required 'compute.googleapis.com' API not enabled` —— 17 章 §1.3 漏开了,补 `gcloud services enable compute.googleapis.com`。

#### 申请加 quota

去 Console → IAM & Admin → Quotas & System Limits → filter `Service: Compute Engine API` + `Region: europe-west3` → 选中 → Edit Quotas → 填新值 + 业务理由。

经验值:smoke test 把 `SSD_TOTAL_GB` 加到 700 GB 够用;生产 500 inpage pod 并发要加到 5000 GB(每节点 ~100 GB × 36 节点 + 余量)。新项目首次申请通常 24–48h,有付款历史的项目 30 分钟到几小时。

### Q17：怎么省 GKE 的钱？

- **关掉 deletion_protection 之后真去 destroy 不用的 cluster**
- **Autopilot 比 Standard 在 bursty 工作负载上更省**
- **schedule pod / cron 错峰** —— 同时段并发 pod 越多，节点越多
- **关私有 endpoint，删 NAT** —— 如果你没用 NAT 出公网（极少数）
- **stop GCS bucket 的 versioning** —— 不需要的话别开

## Hetzner 特定

### Q18：hetzner-k3s 装不上 nfs-common

项目踩过的坑。**`additional_post_k3s_commands` 必须放顶层**，不能放 pool 级（pool 级被工具默默忽略）。

```yaml
# ✅ 顶层
additional_post_k3s_commands:
  - apt-get update
  - apt-get install -y nfs-common

# ❌ pool 级（无效）
worker_node_pools:
  - name: pool-1
    additional_post_k3s_commands:    # 工具不读这个
      - ...
```

或者加一个 DaemonSet 兜底：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: install-nfs-common
spec:
  template:
    spec:
      hostPID: true
      containers:
        - name: installer
          image: ubuntu:24.04
          command: ["nsenter", "-t", "1", "-m", "-u", "-i", "-n", "-p", "--", "bash", "-c"]
          args:
            - apt-get update && apt-get install -y nfs-common
          securityContext:
            privileged: true
```

### Q19：Hetzner VM 邮件发到我了，是密码？

每个直接 `hcloud server create` **没带 `--ssh-key`** 时，Hetzner 把 root 密码邮件给项目所有者。

**永远带 `--ssh-key`**：

```bash
hcloud server create --name probe --type cx22 --image ubuntu-24.04 \
  --ssh-key <your-key-name>     # ← 必须！
```

或者用 hetzner-k3s（已经替你处理好）。

### Q20：怎么把 Hetzner k3s 集群升级 K8s 版本？

```bash
# 改 cluster.yaml: k3s_version: v1.34.6+k3s1 → v1.34.7+k3s1
hetzner-k3s upgrade --config cluster.yaml
# 自动 cordon / drain / 升级一个节点 / uncordon → 下一个
```

## 通用 / 工具

### Q21：怎么调试集群网络问题（Pod 之间不通）？

```bash
# 1. 起一个 debug pod
kubectl run -it --rm debug --image=nicolaka/netshoot -- bash

# 2. 在 debug pod 里
nslookup hello.my-app                # DNS 通吗？
curl -v hello.my-app:80              # HTTP 通吗？
ping hello.my-app                    # ICMP 通吗？（CNI 可能拦）
traceroute hello.my-app              # 路径
ip route                             # 路由表
```

[`nicolaka/netshoot`](https://github.com/nicolaka/netshoot) 这个镜像装了 100+ 网络工具，K8s 网络 debug 必备。

### Q22：在哪里看 K8s 全局事件？

```bash
kubectl get events -A --sort-by='.lastTimestamp'
```

或者只看 Warning：

```bash
kubectl get events -A --field-selector type=Warning
```

实时跟：

```bash
kubectl get events -A --watch
```

### Q23：`kubectl` 太啰嗦怎么办？

装这些：

- [`kubectx`](https://github.com/ahmetb/kubectx) —— 切 context
- [`kubens`](https://github.com/ahmetb/kubectx) —— 切 namespace
- [`stern`](https://github.com/stern/stern) —— 多 Pod 日志聚合
- [`k9s`](https://k9scli.io/) —— TUI 集群浏览器，比 kubectl 顺手 10 倍
- [`kubie`](https://github.com/sbstp/kubie) —— shell prompt 显示当前 context / ns

`alias k=kubectl` 是最低收益最高的一行。

### Q24：怎么知道集群里资源用得多吗？

```bash
# 节点级
kubectl top node

# Pod 级
kubectl top pod -A

# 按 namespace 汇总
kubectl top pod -A | awk 'NR>1 {ns[$1]+=$3} END {for (n in ns) print n, ns[n]}'
```

`kubectl top` 需要 `metrics-server`，多数云厂商默认装好。Hetzner k3s **默认没装**，要单独 apply。

## 为什么 / 设计

### Q25：为什么 K8s 这么复杂？

因为它是**分布式系统**。要解决：

- 节点故障（自愈）
- 服务发现（动态地址）
- 配置管理（多环境）
- 滚动升级（不中断服务）
- 资源调度（bin packing）
- 跨主机网络（CNI）
- 持久化（动态 PV）

**任何一个生产化分布式系统都会演化成 K8s 那个样子** —— K8s 只是把这些问题统一抽象了。

### Q26：能不能不用 K8s？

可以。看场景：

- 单机一个 docker-compose 跑得起来 → 不用 K8s
- 几台 VM systemd 管 → 不用 K8s
- 一个云厂商的 serverless 撑住 → 不用 K8s
- 跨云、多服务、动态扩缩、多团队 → K8s 当之无愧

**K8s 是工具不是宗教**。能不上就不上。

### Q27：学完这套教程下一步学什么？

按兴趣 / 工作选：

- **真实项目**：找一个开源项目用 K8s 部署，给它写 Helm chart 或 Kustomize
- **CI/CD**：ArgoCD / Flux —— GitOps 是 K8s 的标配
- **可观测性**：Prometheus + Grafana + Loki + Tempo
- **服务网格**：Istio / Linkerd —— mTLS、流量管理、observability
- **安全**：RBAC 精细化 / Pod Security Standards / OPA Gatekeeper
- **多集群**：Karmada / Cluster API
- **Operator**：写一个自定义 CRD + controller，看 K8s extension 怎么做

---

## 还有问题

教程范围内的 issue 欢迎在 [GitHub repo 提 issue](https://github.com/baixianger/iac-k8s-tutorial-cn/issues)。

教程范围外的 K8s 问题，推荐去：

- [Kubernetes 官方文档](https://kubernetes.io/docs/)
- [r/kubernetes](https://reddit.com/r/kubernetes)
- [CNCF Slack](https://slack.cncf.io/)（很活跃）

---

> [章节索引](./)
