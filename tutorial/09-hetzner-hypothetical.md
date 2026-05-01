---
layout: default
title: 09 — Hetzner 假设 Terraform
---

# 第 09 章：Hetzner —— 假设用 Terraform 怎么写

> 上一章：[08 — GKE Outputs](08-gke-outputs.html) · [章节索引](./)

> ⚠️ **现实情况**：本项目 Hetzner 路径用的是 `hetzner-k3s` + `cluster.yaml`，**不是** Terraform。本章是**教学假设** —— 假设用 Terraform 重写一遍会长什么样。
>
> 这种对比对学习有价值：**同一套 Terraform 工具，换了 provider 就能管完全不同的云**。下一章再讲真实路径。

## 1. Hetzner Cloud Terraform Provider

Hetzner Cloud 官方提供 `hetznercloud/hcloud` Terraform provider。它能管理 Hetzner 上几乎所有资源：

| Hetzner 资源 | Terraform resource type |
|---|---|
| 虚拟机 | `hcloud_server` |
| 私有网络 | `hcloud_network` |
| 子网 | `hcloud_network_subnet` |
| 防火墙 | `hcloud_firewall` |
| SSH key | `hcloud_ssh_key` |
| 浮动 IP | `hcloud_floating_ip` |
| Block storage | `hcloud_volume` |
| Load balancer | `hcloud_load_balancer` |
| 放置组（反亲和） | `hcloud_placement_group` |

## 2. 假设的项目结构

```
cloud/deployments/stacks/hetzner/infra-tf-hypothetical/
├── main.tf              # provider 配置
├── variables.tf         # 输入参数
├── network.tf           # 网络 + 子网 + 防火墙
├── ssh.tf               # SSH key
├── servers.tf           # 控制面 + worker（含 cloud-init 装 k3s）
├── outputs.tf           # 输出 IP 等
└── cloud-init/          # 节点初始化脚本
    ├── control-plane.yaml
    └── worker.yaml
```

跟 GKE 比，多了 `ssh.tf`、`servers.tf`、`cloud-init/` 三样 —— Hetzner 节点是裸 VM，需要 SSH key + 显式建机器 + 自己装 k3s。

## 3. main.tf —— 类比 GKE

```hcl
terraform {
  required_version = ">= 1.6"
  required_providers {
    hcloud = {
      source  = "hetznercloud/hcloud"
      version = "~> 1.45"
    }
  }
}

provider "hcloud" {
  token = var.hcloud_token
}
```

跟 GKE 几乎一模一样，区别：

- `source = "hetznercloud/hcloud"`（而不是 google）
- `provider "hcloud"` 块用 `token = var.hcloud_token` 显式传 API token

> Hetzner 的 token 是密钥（不是 OAuth），所以**必须**显式传给 provider。GCP 用 ADC（已登录的浏览器 token），所以不用写 credentials。

## 4. variables.tf

```hcl
variable "hcloud_token" {
  description = "Hetzner Cloud API token (rw scope)."
  type        = string
  sensitive   = true   # 不打印到日志
}

variable "cluster_name" {
  default = "my-app"
  type    = string
}

variable "location" {
  default = "fsn1"      # Falkenstein, Germany
  type    = string
}

variable "control_plane_type" {
  default = "cx43"      # 8 vCPU, 32 GB RAM
  type    = string
}

variable "worker_type" {
  default = "cx33"      # 4 vCPU, 16 GB RAM
  type    = string
}

variable "worker_count" {
  default = 3
  type    = number
}

variable "admin_ssh_public_key" {
  description = "SSH public key for root@control-plane"
  type        = string
}

variable "admin_cidr" {
  description = "CIDR allowed to SSH and reach k3s API"
  default     = "203.0.113.42/32"
  type        = string
}

variable "cluster_token" {
  description = "Shared k3s cluster token (used by control plane + workers to trust each other)"
  type        = string
  sensitive   = true
}
```

注意点：

- `hcloud_token` 标 `sensitive = true` —— log/output 不会打印明文
- `control_plane_type = "cx43"` 是 Hetzner 的实例规格代号（8 vCPU / 32 GB），**类比 GKE 的 `n2-standard-16`**，但 Hetzner 是固定规格，不像 GCP 那么多种类
- `location = "fsn1"` Falkenstein（德国 Hetzner 主机房之一）；其他常见 `nbg1`（Nuremberg）、`hel1`（Helsinki）

### 4.1 Hetzner 实例规格速查

| 代号 | vCPU | RAM | 存储 | 月费（约） | 适合 |
|---|---|---|---|---|---|
| `cx22` | 2 | 4 GB | 40 GB | €4 | dev / 单 worker |
| `cx32` | 4 | 8 GB | 80 GB | €7 | 中 worker |
| `cx33` | 4 | 16 GB | 160 GB | €11 | 标准 worker |
| `cx42` | 8 | 16 GB | 160 GB | €15 | 大 worker |
| `cx43` | 8 | 32 GB | 320 GB | €27 | 控制面 |
| `cx52` | 16 | 32 GB | 320 GB | €30 | 重 worker |

完整列表：<https://www.hetzner.com/cloud>

## 5. network.tf —— 类比 GKE 但简化

```hcl
# 私有网络（节点间通讯）
resource "hcloud_network" "private" {
  name     = "${var.cluster_name}-net"
  ip_range = "10.0.0.0/16"
}

# 子网
resource "hcloud_network_subnet" "private" {
  network_id   = hcloud_network.private.id
  type         = "cloud"
  network_zone = "eu-central"
  ip_range     = "10.0.1.0/24"
}

# 防火墙规则（开 SSH + k3s API + HTTP/HTTPS）
resource "hcloud_firewall" "k8s_admin" {
  name = "${var.cluster_name}-admin"

  # 允许从 admin_cidr 访问 SSH + k3s API
  rule {
    direction  = "in"
    protocol   = "tcp"
    port       = "22"
    source_ips = [var.admin_cidr]
  }
  rule {
    direction  = "in"
    protocol   = "tcp"
    port       = "6443"   # k3s API
    source_ips = [var.admin_cidr]
  }

  # 允许从全网访问 80/443（如果 Ingress 在用）
  rule {
    direction  = "in"
    protocol   = "tcp"
    port       = "80"
    source_ips = ["0.0.0.0/0", "::/0"]
  }
  rule {
    direction  = "in"
    protocol   = "tcp"
    port       = "443"
    source_ips = ["0.0.0.0/0", "::/0"]
  }
}
```

对比 GKE network.tf：

| 概念 | GKE | Hetzner |
|---|---|---|
| 网络 | `google_compute_network` | `hcloud_network` |
| 子网 | `google_compute_subnetwork` | `hcloud_network_subnet` |
| 防火墙 | （没单独建 —— 靠 master_authorized_networks_config） | `hcloud_firewall` |
| 访问控制 | 通过 cluster 配置 | 通过独立 firewall 资源 + 绑到 server |

## 6. ssh.tf —— Hetzner 特有

```hcl
resource "hcloud_ssh_key" "admin" {
  name       = "${var.cluster_name}-admin"
  public_key = var.admin_ssh_public_key
}
```

GKE 用 IAM 配 kubectl 访问，**不需要**给节点 SSH —— 节点是 managed 的。Hetzner 不一样，节点是你的 VM，要 SSH 上去做事（debug 节点、看 systemd 日志、检查 docker 等）。

> ⚠️ 实操坑（来自项目教训）：每个直接 `hcloud server create`（含 capacity probe / 一次性 VM）都得带 `--ssh-key <你的-key-name>`，否则 Hetzner 会**邮件**把生成的 root 密码发给项目所有人。

## 7. servers.tf —— Hetzner Terraform 最大的部分

```hcl
# 控制面节点（master）
resource "hcloud_server" "control_plane" {
  name        = "${var.cluster_name}-cp"
  image       = "ubuntu-24.04"
  server_type = var.control_plane_type
  location    = var.location

  ssh_keys     = [hcloud_ssh_key.admin.id]
  firewall_ids = [hcloud_firewall.k8s_admin.id]

  network {
    network_id = hcloud_network.private.id
    ip         = "10.0.1.2"   # 静态私有 IP
  }

  user_data = templatefile("${path.module}/cloud-init/control-plane.yaml", {
    cluster_token = var.cluster_token
  })

  depends_on = [hcloud_network_subnet.private]
}

# Worker 节点（多个）
resource "hcloud_server" "worker" {
  count = var.worker_count   # 数组：建 N 个

  name        = "${var.cluster_name}-w${count.index + 1}"
  image       = "ubuntu-24.04"
  server_type = var.worker_type
  location    = var.location

  ssh_keys     = [hcloud_ssh_key.admin.id]
  firewall_ids = [hcloud_firewall.k8s_admin.id]

  network {
    network_id = hcloud_network.private.id
    ip         = "10.0.1.${count.index + 10}"   # 10.0.1.10, 10.0.1.11, ...
  }

  user_data = templatefile("${path.module}/cloud-init/worker.yaml", {
    control_plane_ip = hcloud_server.control_plane.network[0].ip
    cluster_token    = var.cluster_token
  })

  depends_on = [hcloud_server.control_plane]
}
```

### 7.1 新概念

- `count = var.worker_count`：让一个 resource 复制 N 份。N=3 就建 3 个 worker。引用第 i 个用 `hcloud_server.worker[i]`
- `count.index`：循环里的"第几个"，从 0 开始。`count.index + 1` 就是 1, 2, 3... 用来生成 `worker-w1`, `worker-w2`, `worker-w3` 这样的名字
- `templatefile()`：读 cloud-init 模板，把变量代入。这就是 Hetzner 怎么"在新建 VM 上自动装 k3s"
- `depends_on`：显式声明依赖。这里用是因为 worker 启动时要连控制面，控制面必须先就绪

### 7.2 user_data 是关键

GKE 不需要这个，因为 Autopilot 内置 K8s 自动起。Hetzner 得自己装 k3s。Cloud-init 模板大概长这样：

```yaml
# cloud-init/control-plane.yaml
#cloud-config
runcmd:
  - curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.34.6+k3s1 K3S_TOKEN=${cluster_token} sh -
  - cp /etc/rancher/k3s/k3s.yaml /root/kubeconfig.yaml
```

```yaml
# cloud-init/worker.yaml
#cloud-config
runcmd:
  - curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.34.6+k3s1 K3S_URL=https://${control_plane_ip}:6443 K3S_TOKEN=${cluster_token} sh -
```

控制面节点装完 k3s，worker 节点用类似脚本但传 `K3S_URL=...` join 进集群。

## 8. outputs.tf

```hcl
output "control_plane_ipv4" {
  value = hcloud_server.control_plane.ipv4_address
}

output "control_plane_private_ip" {
  value = hcloud_server.control_plane.network[0].ip
}

output "worker_ipv4s" {
  value = [for w in hcloud_server.worker : w.ipv4_address]
  # 用 for 表达式遍历 worker 数组，提取每个的 ipv4_address
}

output "kubeconfig_hint" {
  value = "scp root@${hcloud_server.control_plane.ipv4_address}:/root/kubeconfig.yaml ~/.kube/hetzner-config"
}
```

`for w in ...` 是 Terraform 的 for expression —— 遍历数组生成新数组。Python list comprehension 类似。

## 9. apply 完之后干啥

```bash
# 1. 拉 kubeconfig 到本地
scp root@$(terraform output -raw control_plane_ipv4):/root/kubeconfig.yaml ~/.kube/hetzner-config

# 2. 改 kubeconfig 里的 server 地址（默认是 127.0.0.1，得改成 control plane public IP）
sed -i '' "s/127.0.0.1/$(terraform output -raw control_plane_ipv4)/" ~/.kube/hetzner-config

# 3. kubectl 测连通
KUBECONFIG=~/.kube/hetzner-config kubectl get nodes
# 应该看到 1 个 control-plane + N 个 worker，全部 Ready
```

## 10. 这套假设 Terraform 写的工作量

我估算一下：

| 文件 | 行数 |
|---|---|
| main.tf | ~15 |
| variables.tf | ~50 |
| network.tf | ~50 |
| ssh.tf | ~5 |
| servers.tf | ~50 |
| outputs.tf | ~20 |
| cloud-init/*.yaml | ~30 |
| **合计** | **~220 行** |

不少。下一章我们看真实的项目用的是什么 —— 一份 ~50 行的 `cluster.yaml`，工具一站式做完。

---

> 下一章：[10 — Hetzner 真实路径（hetzner-k3s）](10-hetzner-real-path.html) · [章节索引](./)
