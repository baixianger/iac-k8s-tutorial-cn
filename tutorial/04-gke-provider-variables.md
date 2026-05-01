---
layout: default
title: 04 — GKE Provider 与 Variables
---

# 第 04 章：GKE — Provider 与 Variables

> 上一章：[03 — Terraform 工作流](03-terraform-workflow.html) · [章节索引](./)

从这一章开始就是真实代码。我们会逐字解读一个跑在生产里的 GKE Autopilot stack，文件位置约定为 `cloud/deployments/stacks/gke/infra/`。

这一章覆盖前两个文件：

- `main.tf` — Terraform 自身配置 + provider 块
- `variables.tf` — 输入参数定义

## 1. main.tf — Provider 配置

```hcl
# main.tf — Provider 配置

terraform {
  required_version = ">= 1.6"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 6.10"
    }
    google-beta = {
      source  = "hashicorp/google-beta"
      version = "~> 6.10"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

provider "google-beta" {
  project = var.project_id
  region  = var.region
}
```

### 1.1 `terraform { }` 块逐行解

```hcl
terraform {
  required_version = ">= 1.6"
```

最低 Terraform CLI 版本 1.6。如果操作员本地是 1.5，跑 plan 会立刻报错。**为什么写**：避免不同人本地 Terraform 版本太旧，跑出怪错误。

```hcl
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 6.10"
    }
```

声明这个项目用 `hashicorp/google` 这个 provider 插件，版本约束 `~> 6.10`。

- `source = "hashicorp/google"` 等于 Terraform Registry 上的 `registry.terraform.io/hashicorp/google`，是 Google 官方维护的 provider
- `version = "~> 6.10"` 用 `~>` 是"悲观约束" —— 允许 6.10.0、6.10.1、...，但**不允许 6.11.0**
  - `>= 6.10` 允许 6.11、6.12...（更松）
  - `= 6.10.0` 严格固定（更紧）

```hcl
    google-beta = {
      source  = "hashicorp/google-beta"
      version = "~> 6.10"
    }
  }
}
```

`google-beta` 是同一家 Google 提供的 "beta API" provider。GKE 一些较新功能（如 Autopilot 的某些字段）只在 google-beta 提供，所以同时引两个。

### 1.2 `provider "google" { }` 块逐行解

```hcl
provider "google" {
  project = var.project_id
  region  = var.region
}
```

具体配置 google provider 怎么连：

- `project = var.project_id` —— 操作哪个 GCP 项目（值从 `variables.tf` 来）
- `region = var.region` —— 默认操作哪个区域（每个 resource 也可以自己 override）

**没写 credentials**：因为我们用 ADC（Application Default Credentials），由 `gcloud auth application-default login` 注入到 `~/.config/gcloud/application_default_credentials.json`。Terraform 自动找它，**不需要在代码里硬写 credentials 路径** —— 这样 `.tf` 文件可以放心 git checkin。

### 1.3 第二个 `provider "google-beta" { }`

跟上面一样配置，但是给 `google-beta` provider 用的。如果某个 resource 写了 `provider = google-beta`，它就用这块配置（你会在 cluster.tf 看到这个用法）。

### 1.4 这个文件你能改什么

| 想改的 | 改哪里 |
|---|---|
| 升级到 google provider 7.x | 改 `version = "~> 7.0"` |
| 改默认 GCP 项目 | 改 `var.project_id` 默认值（在 variables.tf） |
| 改默认区域 | 改 `var.region` 默认值（在 variables.tf） |
| 用本地 GCP credentials JSON 文件 | 加 `credentials = file("path/to/key.json")` 到 provider 块 —— 但 ADC 是更好的方案 |

## 2. variables.tf — 输入参数

```hcl
variable "project_id" {
  description = "GCP project ID hosting the cluster + AR + GCS."
  type        = string
  default     = "my-cloud-project"
}

variable "region" {
  description = "Regional GKE control plane (gives 3-zone HA). Frankfurt (europe-west3)..."
  type        = string
  default     = "europe-west3"
}

variable "cluster_name" {
  description = "Cluster name. Sandbox tests should use a different value..."
  type        = string
  default     = "my-cluster"
}

variable "admin_cidr" {
  description = "CIDR allowed to hit the public control plane endpoint."
  type        = string
  default     = "203.0.113.42/32"
}

variable "master_cidr" {
  description = "CIDR for the private control plane endpoint (must be /28)."
  type        = string
  default     = "172.16.0.0/28"
}

variable "ar_repo_id" {
  description = "Artifact Registry repository ID."
  type        = string
  default     = "my-app"
}

variable "websites_bucket_suffix" {
  description = "Suffix for the websites-data bucket; full name is `<project>-<suffix>`."
  type        = string
  default     = "websites-data"
}

variable "audit_bucket_suffix" {
  default = "audit"
  type    = string
}
```

### 2.1 一个 `variable` 块的结构

```hcl
variable "name" {       # name = 变量名，引用时用 var.name
  description = "..."   # 用途说明（plan/apply 时打印；新人看代码也靠这个）
  type        = string  # 类型（5 大基本类型之一，或 object）
  default     = "..."   # 默认值；不写 default 就强制要求操作员输入
}
```

### 2.2 8 个变量逐个看

| 变量 | 类型 | 默认 | 用来做什么 |
|---|---|---|---|
| `project_id` | string | `my-cloud-project` | 整个 stack 都用它，决定资源在哪个 GCP 项目 |
| `region` | string | `europe-west3` | Frankfurt（德国） |
| `cluster_name` | string | `my-cluster` | GKE 集群名，也是 VPC / NAT 等资源名前缀 |
| `admin_cidr` | string | `203.0.113.42/32` | 你的家庭 IP（操作 kubectl 必备） |
| `master_cidr` | string | `172.16.0.0/28` | 私有控制面 CIDR，必须 /28，跟你 VPC 不能冲突 |
| `ar_repo_id` | string | `my-app` | Artifact Registry repo 名 |
| `websites_bucket_suffix` | string | `websites-data` | bucket 名后缀，完整名 `my-cloud-project-websites-data` |
| `audit_bucket_suffix` | string | `audit` | 审计 bucket 后缀 |

### 2.3 三种覆盖默认的方式

操作员要改某个变量值，有三种方式（优先级高 → 低）：

1. **命令行 `-var`**（一次性）：

   ```bash
   terraform apply -var="project_id=my-sandbox" -var="admin_cidr=1.2.3.4/32"
   ```

2. **环境变量 `TF_VAR_xxx`**（适合 CI）：

   ```bash
   export TF_VAR_project_id=my-sandbox
   export TF_VAR_admin_cidr=1.2.3.4/32
   terraform apply
   ```

3. **`terraform.tfvars` 文件**（推荐，持久化）：

   ```hcl
   # cloud/deployments/stacks/gke/infra/terraform.tfvars
   project_id = "my-sandbox"
   admin_cidr = "1.2.3.4/32"
   ```

   `terraform.tfvars` 会被自动加载。这个文件**不进 git**（`.gitignore` 排除），存私人项目 ID / IP。

项目里通常还会有 `terraform.tfvars.example`（**进 git**）作为模板：

```hcl
# project_id   = "my-cloud-project"   # 默认就是这个
# admin_cidr   = "1.2.3.4/32"         # 务必改成你的 /32
# cluster_name = "my-cluster-test"    # sandbox 用 -test 后缀
```

### 2.4 派生变量（不直接定义，从其他变量算出）

GKE stack 里有几个"派生"名字 —— 不是变量但靠变量拼出来。比如 `storage.tf`：

```hcl
resource "google_storage_bucket" "websites" {
  name = "${var.project_id}-${var.websites_bucket_suffix}"
  # 渲染成: "my-cloud-project-websites-data"
}
```

**好处**：你只改 `project_id` 一个地方，bucket 名 + AR 完整路径全部跟着变。新建项目从零部署只需改 `project_id`，剩下名字自动连贯。

## 3. 学完这一章应该会什么

- ✅ 看到 `terraform { required_providers { ... } }` 知道在干啥、为什么 `~>` 是悲观约束
- ✅ 知道 GCP credentials 不写在 `.tf` 里，靠 ADC 自动发现
- ✅ 知道一个 variable 块的 4 个字段（name / description / type / default）
- ✅ 会在 3 种方式之间选择如何 override 变量
- ✅ 看到 `${var.project_id}-${var.suffix}` 这种插值知道在做什么

下一章看 `network.tf`，VPC + subnet + NAT。

---

> 下一章：[05 — GKE 网络](05-gke-network.html) · [章节索引](./)
