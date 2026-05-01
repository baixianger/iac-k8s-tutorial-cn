---
layout: default
title: 10 — Hetzner 真实路径（hetzner-k3s）
---

# 第 10 章：Hetzner 真实路径 —— hetzner-k3s

> 上一章：[09 — Hetzner 假设 Terraform](09-hetzner-hypothetical.html) · [章节索引](./)

上一章我们用纯 Terraform 写了 ~220 行假设的 Hetzner 路径。这一章看真实项目是怎么做的：用 `hetzner-k3s` 这个 CLI + 一份 `cluster.yaml`，一行命令拉起整个集群。

学完这章，你能回答：**"为什么不用 Terraform？哪些场景应该用 Terraform？"**

## 1. hetzner-k3s 是什么

[`hetzner-k3s`](https://github.com/vitobotta/hetzner-k3s) 是 Vito Botta 维护的开源 CLI。一句话：

> 一个工具，一份 yaml，自动开 Hetzner VM + 装 k3s + 配集群。

它内部干的事跟我们上一章 Terraform 设计的一样：建 network → 建 firewall → 建 servers → 在每台机器上装 k3s。但**整个流程封装成一个命令**。

```bash
hetzner-k3s create --config cluster.yaml
# 一气呵成：Network → Firewall → CP VM → Worker VMs → 装 k3s → 输出 kubeconfig
```

## 2. cluster.yaml 真实样子

```yaml
---
hetzner_token: <REDACTED>
cluster_name: my-app
kubeconfig_path: "~/.kube/hetzner-config"
k3s_version: v1.34.6+k3s1

networking:
  ssh:
    port: 22
    use_agent: false
    public_key_path: "~/.ssh/hetzner.pub"
    private_key_path: "~/.ssh/hetzner"
  allowed_networks:
    ssh:
      - 203.0.113.42/32
    api:
      - 203.0.113.42/32
  public_network:
    ipv4: true
    ipv6: true
  private_network:
    enabled: true
    subnet: 10.0.0.0/16
    existing_network_name: ""
  cni:
    enabled: true
    encryption: false
    mode: flannel

datastore:
  mode: etcd

schedule_workloads_on_masters: false

masters_pool:
  instance_type: cx43
  instance_count: 1
  location: fsn1

worker_node_pools:
  - name: pool-1
    instance_type: cx33
    instance_count: 3
    location: fsn1
    autoscaling:
      enabled: true
      min_instances: 1
      max_instances: 10

additional_post_k3s_commands:
  - apt-get update
  - apt-get install -y nfs-common
```

50 行不到。对比我们假设的 220 行 Terraform —— 工具帮你省了大概 4 倍代码。

## 3. 概念逐字看

### 3.1 顶部 + 网络

```yaml
hetzner_token: <REDACTED>
cluster_name: my-app
kubeconfig_path: "~/.kube/hetzner-config"
k3s_version: v1.34.6+k3s1
```

跟 Terraform 的 `var.hcloud_token` / `var.cluster_name` 是一回事。

```yaml
networking:
  ssh:
    public_key_path: "~/.ssh/hetzner.pub"
    private_key_path: "~/.ssh/hetzner"
  allowed_networks:
    ssh:  [203.0.113.42/32]
    api:  [203.0.113.42/32]
```

`public_key_path` —— hetzner-k3s 自己读这个文件、自己上传到 Hetzner 当 SSH key（替代 Terraform 里的 `hcloud_ssh_key` 资源）。

`allowed_networks` —— hetzner-k3s 自动建对应的 `hcloud_firewall`，规则跟我们 Terraform 写的一样。

### 3.2 节点池

```yaml
masters_pool:
  instance_type: cx43
  instance_count: 1
  location: fsn1

worker_node_pools:
  - name: pool-1
    instance_type: cx33
    instance_count: 3
    location: fsn1
    autoscaling:
      enabled: true
      min_instances: 1
      max_instances: 10
```

替代 Terraform 里的 `hcloud_server` × N + cloud-init 模板。

**多了一个 Terraform 没有的能力**：`autoscaling`。hetzner-k3s 内置集成 [Cluster Autoscaler](https://github.com/kubernetes/autoscaler)，根据集群里 Pending pod 的数量自动加 / 删 worker。Terraform 是声明式，**不动态扩缩** —— 要扩缩你得改 `var.worker_count` 然后 apply。这是真实生产的硬刚需。

### 3.3 装包钩子

```yaml
additional_post_k3s_commands:
  - apt-get update
  - apt-get install -y nfs-common
```

每个节点 k3s 装完后跑这两行。我们项目用 NFS 挂 PVC，需要每个 worker 装 `nfs-common`，用这个钩子最干净。

> ⚠️ 实战坑（项目踩过）：hetzner-k3s 2.4.9 版本曾经在 init-0.sh 里用 `${PIPESTATUS[0]}`，但 Ubuntu 默认 `/bin/sh` 是 dash 不是 bash，导致 `nfs-common` 装失败。修复办法是把 `additional_post_k3s_commands` 放在**顶层**（不能放在 pool 级 —— pool 级会被工具默默忽略 ⚠️），或者加一个 `install-nfs-common` DaemonSet 兜底。

### 3.4 配额隐藏点

```yaml
datastore:
  mode: etcd
```

控制面用 etcd 存集群状态（k3s 默认是 sqlite，单节点用够了；多控制面或大集群得用 etcd）。

```yaml
schedule_workloads_on_masters: false
```

不让用户 pod 调度到控制面。生产推荐设 `false`，控制面专心跑 k8s 系统组件。

## 4. 真实运维流程

### 4.1 创建集群

```bash
hetzner-k3s create --config cluster.yaml
# 输出：
# - kubeconfig: ~/.kube/hetzner-config
# - control plane public IP: <ip>
# - 4 个 hcloud_server 创建好（1 cp + 3 worker）
# - k3s 起来
# - 节点 Ready
```

### 4.2 动态扩 worker

```yaml
# 改 cluster.yaml: worker_node_pools[0].instance_count: 5
```

```bash
hetzner-k3s create --config cluster.yaml
# hetzner-k3s 是 idempotent 的：
# 已经存在的资源不动，只加 2 台新 worker
```

### 4.3 destroy

```bash
hetzner-k3s delete --config cluster.yaml
# 删所有 4 台 server + network + firewall
```

### 4.4 升级 k3s

```yaml
# 改 k3s_version: v1.34.7+k3s1
```

```bash
hetzner-k3s upgrade --config cluster.yaml
# 滚动升级，自动 cordon / drain / 升级 / uncordon
```

## 5. hetzner-k3s 帮你做了什么 vs 没做什么

| 事项 | Terraform 你要写 | hetzner-k3s 自动 |
|---|---|---|
| 建 network / subnet | `hcloud_network` + `hcloud_network_subnet` | ✅ |
| 建 firewall + 规则 | `hcloud_firewall` + 多个 rule | ✅ |
| 建 SSH key | `hcloud_ssh_key` | ✅ |
| 建 control plane | `hcloud_server` + cloud-init | ✅ |
| 建 worker × N | `hcloud_server` count + cloud-init | ✅ |
| 装 k3s | cloud-init 里 `curl get.k3s.io` | ✅ |
| etcd mode | 自己在 cloud-init 里 | ✅ |
| 整集群 token 管理 | 自己 `cluster_token` 变量 + 传给 cloud-init | ✅ |
| Cluster Autoscaler | ❌（Terraform 是静态的） | ✅ 内置集成 |
| 滚动升级 | ❌（要自己搞 cordon/drain 流程） | ✅ |
| 让你看 plan diff | ✅ `terraform plan` | ❌ |
| state 跟踪、跨工具协作 | ✅ `tfstate` | ❌（依赖 hcloud API + cluster.yaml 一致性） |
| 任意 Hetzner 资源（如 LB、Volume）声明 | ✅ | ❌（只管 K8s 集群相关） |

**结论**：hetzner-k3s 在"建 K8s 集群"这件事上更专、更省事；Terraform 在"管理任意 Hetzner 资源"上更通用。两者**可以共存**：用 hetzner-k3s 拉集群，用 Terraform 管 Volume / LB / DNS 之类的周边。

## 6. GKE vs Hetzner 一表对照（两边都看过了再总结）

| 概念 | GKE Terraform | Hetzner（hetzner-k3s） |
|---|---|---|
| 工具 | Terraform + `hashicorp/google` | `hetzner-k3s` CLI + cluster.yaml |
| 认证 | ADC（环境注入） | yaml 里 `hetzner_token` |
| 网络 | 显式 VPC + 2 个 secondary range（pods/services） | yaml `private_network.subnet` 一行 |
| 子网大小 | nodes /20, pods /14, services /20 | 一个 /16（CNI 自管 pod IP） |
| 防火墙 | 通过 cluster `master_authorized_networks_config` | yaml `allowed_networks` |
| NAT / 出口 IP | 显式 `google_compute_address` + `google_compute_router_nat` | 默认每 VM 自带 public IP |
| 节点 | Autopilot 自动管，不写代码 | yaml `masters_pool` + `worker_node_pools` |
| K8s 安装 | Autopilot 内置 | hetzner-k3s 跑 cloud-init `curl get.k3s.io` |
| 自动扩缩 | Autopilot 按 pod 自动 | Cluster Autoscaler（hetzner-k3s 内置集成） |
| 存储 | `google_storage_bucket` + GCSFuse mount | NFS Server / Hetzner Object Storage（S3-compatible） |
| Service Account | `google_service_account` + Workload Identity | 没原生 IAM —— pod 凭据自己管 secret |
| 配额 | 区域级 GCE quota（SSD_TOTAL_GB / CPUS / INSTANCES） | 项目级 server limit（默认 10 台 VM 上限，可申请加） |
| 计费 | 按 pod / 节点资源（Autopilot 模式区分） | 按 VM 类型固定月费 |
| 扩容速度 | Autopilot 自动 1-3 min | hetzner-k3s autoscaler 自动；纯 Terraform 路径要改 var + apply |
| 学习曲线 | 一开始难（Workload Identity / Autopilot 概念） | 一开始易（VM 你都懂，加个 K8s） |

## 7. Terraform 哲学保持不变

虽然 Hetzner 真实路径用了 hetzner-k3s 而不是 Terraform，**Terraform 的核心范式两边一样**：

1. **写文本文件描述期望状态**
2. **`plan` 看 diff** —— hetzner-k3s 没有 plan，是它的弱项
3. **`apply` 落地**
4. **state 跟踪当前状态**
5. **`destroy` 拆光**

学完一个 IaC 工具，第二个就很快上手 —— 只要查"这个工具有哪些资源类型 / 字段" 就行。

## 8. 引申：你能给任何云写 Terraform

Terraform 支持 100+ provider，覆盖所有主流云 + 第三方服务：

- AWS: `hashicorp/aws`
- Azure: `hashicorp/azurerm`
- GCP: `hashicorp/google` ← 我们用的
- Hetzner: `hetznercloud/hcloud` ← 假设用的
- Cloudflare（DNS / WAF）: `cloudflare/cloudflare`
- DigitalOcean: `digitalocean/digitalocean`
- GitHub（管 repo / team）: `integrations/github`
- Datadog（监控）: `DataDog/datadog`

**多云 Terraform** 是常见模式：一个 `main.tf` 同时声明 GCP + AWS + Cloudflare 资源。一个 PR 改完 → review → apply，跨云改动一气呵成。

## 9. 学完这一章应该会什么

- ✅ 理解 hetzner-k3s 是 Hetzner 上"建 K8s 集群"的专门工具
- ✅ 看 `cluster.yaml` 能对应到上一章的 Terraform 资源
- ✅ 知道为什么用 hetzner-k3s 而不是纯 Terraform（autoscaler + 滚升）
- ✅ 看清楚 GKE Autopilot vs Hetzner k3s 的工程取舍
- ✅ 理解 IaC 工具范式跨工具共通

至此 **Terraform / IaC 部分讲完**。下一章开始进入 Kubernetes 概念 —— 集群已经有了，是时候学集群里"跑啥东西"。

---

> 下一章：[11 — K8s Pod 与 Deployment](11-k8s-pod-deployment.html) · [章节索引](./)
