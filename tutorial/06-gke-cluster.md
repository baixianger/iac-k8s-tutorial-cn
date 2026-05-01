---
layout: default
title: 06 — GKE Autopilot 集群
---

# 第 06 章：GKE — Autopilot 集群（cluster.tf）

> 上一章：[05 — GKE 网络](05-gke-network.html) · [章节索引](./)

终于到主菜：集群本身。这一章解读 `cluster.tf`，一个 resource 块、十几个嵌套字段。

## 1. cluster.tf 全文

```hcl
resource "google_container_cluster" "autopilot" {
  provider = google-beta
  name     = var.cluster_name
  location = var.region

  enable_autopilot    = true
  deletion_protection = false

  network    = google_compute_network.gke.id
  subnetwork = google_compute_subnetwork.gke.id

  ip_allocation_policy {
    cluster_secondary_range_name  = "pods"
    services_secondary_range_name = "services"
  }

  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false
    master_ipv4_cidr_block  = var.master_cidr
    master_global_access_config {
      enabled = true
    }
  }

  master_authorized_networks_config {
    cidr_blocks {
      cidr_block   = var.admin_cidr
      display_name = "admin"
    }
  }

  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }

  release_channel {
    channel = "REGULAR"
  }

  addons_config {
    gcs_fuse_csi_driver_config {
      enabled = true
    }
  }
}
```

## 2. 顶部声明

```hcl
resource "google_container_cluster" "autopilot" {
  provider = google-beta
```

- `google_container_cluster` 是 GKE 集群的资源类型
- `provider = google-beta` 显式指定用 google-beta（Autopilot 一些字段在 google-beta 才有）

```hcl
  name     = var.cluster_name
  location = var.region
```

- `name` —— 集群名
- `location` —— **regional 集群**（3-zone HA）；如果填 zone（如 `europe-west3-a`）就是 zonal 集群（单可用区，便宜但不抗 zone 故障）

## 3. Autopilot 开关

```hcl
  enable_autopilot = true
```

**这一行决定模式**：

| 值 | 模式 | 计费 | 你管什么 |
|---|---|---|---|
| `true` | Autopilot | 按 pod 计费 / 按 ComputeClass 节点计费 | Pod、Deployment 即可 |
| `false` | Standard | 按节点计费 | 还要管 node pool、autoscaler |

**为什么选 Autopilot**：

- 工作负载是 bursty + ephemeral 的（一会儿 100 个 pod 一会儿 0 个），不需要持续运行 node pool
- Google 帮管节点 → 少操心 OS 升级、kernel 补丁、bin packing
- 成本透明 —— 知道每个 pod 多少钱

```hcl
  deletion_protection = false
```

允许 `terraform destroy`。生产环境应该 `true`，避免误删。Sandbox 给 false 方便清理。

## 4. 网络绑定

```hcl
  network    = google_compute_network.gke.id
  subnetwork = google_compute_subnetwork.gke.id
```

引用 `network.tf` 创建的 VPC + subnet。Terraform 自动建依赖：先 VPC → subnet → cluster。

```hcl
  ip_allocation_policy {
    cluster_secondary_range_name  = "pods"
    services_secondary_range_name = "services"
  }
```

告诉 GKE 用 subnet 的两个 secondary range（在 `network.tf` 定义的 `pods` 和 `services`）。**name 必须和 subnet 里的 `range_name` 完全一致**。

## 5. 私有集群配置

```hcl
  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false
    master_ipv4_cidr_block  = var.master_cidr
    master_global_access_config {
      enabled = true
    }
  }
```

- `enable_private_nodes = true` —— 节点没有 public IP（更安全），出公网走 NAT
- `enable_private_endpoint = false` —— 控制面（API server）有 public IP，但只允许下面 `master_authorized_networks_config` 里的 CIDR 访问
- `master_ipv4_cidr_block = var.master_cidr` —— 控制面私有 IP CIDR（必须 /28，且与 VPC 不冲突）
- `master_global_access_config.enabled = true` —— 全球可访问控制面（不限制只在 region 内）

> **为什么不直接 `enable_private_endpoint = true`？** 那样的话连 API server 都没有 public IP，操作员只能从 VPC 内（如 jump host）访问。开发期太麻烦。生产环境推荐打开 + 配 jump host，不过本教程用的是更简单的"public endpoint + IP 白名单"方案。

## 6. 控制面访问白名单

```hcl
  master_authorized_networks_config {
    cidr_blocks {
      cidr_block   = var.admin_cidr
      display_name = "admin"
    }
  }
```

只允许 `admin_cidr`（你的家 IP）连接控制面 API。如果操作员换了 IP，要改 `var.admin_cidr` 重 apply。

要加多个 IP（比如同事），写多个 `cidr_blocks` 块：

```hcl
  master_authorized_networks_config {
    cidr_blocks {
      cidr_block   = "203.0.113.42/32"
      display_name = "admin"
    }
    cidr_blocks {
      cidr_block   = "1.2.3.4/32"
      display_name = "colleague"
    }
  }
```

或者用 `dynamic` 块从 `list` 里展开（进阶，省略）。

## 7. Workload Identity

```hcl
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }
```

启用 **Workload Identity** —— 让 Pod 能用 GCP IAM 权限访问 GCS / AR / 其他 GCP 服务，**不需要在 Pod 里挂 service account JSON key**。

这是 GCP 推荐的最佳实践。具体绑定怎么写在第 07 章 IAM 里讲。

## 8. Release channel

```hcl
  release_channel {
    channel = "REGULAR"
  }
```

GKE 升级策略：

| Channel | 啥时候拿到新版本 | 适合 |
|---|---|---|
| `RAPID` | 最新（含 alpha 字段） | 测新功能 |
| **`REGULAR`** | 主流稳定（推荐） | 大多数生产 |
| `STABLE` | 落后几个月 | 极保守环境 |

控制面会按所选 channel 自动滚动升级（在 maintenance window 内）。

## 9. Addons：GCSFuse

```hcl
  addons_config {
    gcs_fuse_csi_driver_config {
      enabled = true
    }
  }
```

启用 GCSFuse CSI driver —— 让 Pod 能 **mount GCS bucket 当本地目录**用。

> 比传统的 `gsutil cp` / SDK 上传方便：pod 看到的就是一个普通的目录，写文件 = 写 bucket。读 / 写性能不如本地盘，但对于配置文件、批量产物这种"读一次写一次"的场景刚好。

## 10. 这个文件你能改什么

| 想改的 | 改哪里 | 风险 |
|---|---|---|
| 关闭私有节点 | `enable_private_nodes = false` | 🟡 安全降级（节点暴露 public IP） |
| 升 release channel | `channel = "RAPID"` | 🟡 拿 alpha 功能但 bug 多 |
| 加一个 admin IP | 加 `cidr_blocks { ... }` 块 | 🟢 安全 |
| 关 GCSFuse | `enabled = false` | 🟡 破坏 web data 访问 |
| 切 Standard 模式 | `enable_autopilot = false` + 加 node pool | 🔴 大改，集群重建 |
| 改 location（region → zone） | 改 `location` 值 | 🔴 forces replacement |

## 11. 学完这一章应该会什么

- ✅ 知道 `enable_autopilot` 是分水岭（Autopilot vs Standard）
- ✅ 理解 private cluster + master authorized networks 是怎么配出"安全且可访问"的控制面
- ✅ 理解 Workload Identity 启用是 cluster 级，使用是 IAM 级
- ✅ 看到 release channel 知道集群版本怎么升级
- ✅ 知道 GCSFuse addon 是项目把 GCS 当文件系统用的关键

---

> 下一章：[07 — GKE 存储与 IAM](07-gke-storage-iam.html) · [章节索引](./)
