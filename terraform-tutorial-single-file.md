---
layout: default
title: Terraform 完整教学（GKE + Hetzner）
---

# Terraform 完整教学（GKE + Hetzner）

> 中文 / 零基础 / 逐字逐句 / 用本项目的真实代码教学
>
> 读完之后你应该能：(1) 看懂任何 `.tf` 文件 (2) 修改本项目的 GKE/Hetzner 部署 (3) 给新项目从零写 Terraform。

## 章节地图

| 章 | 主题 | 时长 |
|---|---|---|
| [第 0 章：为什么要学 Terraform](#第-0-章：为什么要学-terraform) | 概念 + 心智模型 | 5 分钟 |
| [第 1 章：核心概念 4 个动词 + state](#第-1-章：核心概念) | 工作流 + 状态机 | 5 分钟 |
| [第 2 章：HCL 语法](#第-2-章：hcl-语法) | 块、参数、表达式 | 10 分钟 |
| [第 3 章：GKE — main.tf](#第-3-章：gke--maintf--provider-配置) | Provider 配置 | 10 分钟 |
| [第 4 章：GKE — variables.tf](#第-4-章：gke--variablestf--输入参数) | 输入参数 | 10 分钟 |
| [第 5 章：GKE — network.tf](#第-5-章：gke--networktf--vpc-subnet-nat) | VPC/subnet/NAT | 15 分钟 |
| [第 6 章：GKE — cluster.tf](#第-6-章：gke--clustertf--autopilot-集群) | Autopilot 集群 | 15 分钟 |
| [第 7 章：GKE — storage.tf](#第-7-章：gke--storagetf--gcs-bucket) | GCS bucket | 10 分钟 |
| [第 8 章：GKE — iam.tf](#第-8-章：gke--iamtf--service-account--权限) | GSA + 权限绑定 | 15 分钟 |
| [第 9 章：GKE — outputs.tf](#第-9-章：gke--outputstf--输出) | 输出值 | 5 分钟 |
| [第 10 章：Hetzner — 假设用 Terraform 怎么写](#第-10-章：hetzner--假设用-terraform-怎么写) | 完整 Hetzner 设计 | 30 分钟 |
| [第 11 章：GKE vs Hetzner 对照](#第-11-章：gke-vs-hetzner-对照) | 一表对比 | 5 分钟 |
| [第 12 章：实战练习](#第-12-章：实战练习) | 自己改改看 | 自由 |
| [第 13 章：FAQ](#第-13-章：faq) | 常见疑问 | 自由 |

总时长：~2.5 小时认真读 + 练习。

---

## 第 0 章：为什么要学 Terraform

### 0.1 你之前怎么开云资源

没用 Terraform 之前，你大概率是这样开云资源：

1. 浏览器打开 GCP Console / Hetzner 控制台
2. 找到 "Create Cluster" / "Add Server" 按钮
3. 点点点填表
4. 等几分钟创建完成
5. 得到一个集群 / 一台 VM

**问题**：

- 没记录。哪个按钮你点了？哪个字段填了什么？三个月后回头看不知道
- 不可重复。同事想搞一个一样的，要全程跟着你点
- 容易出错。手工填的 IP / 配额 / 标签很容易写错
- 没法 review。改了什么没人能 PR review

### 0.2 IaC（Infrastructure as Code）的概念

把"基础设施"变成"代码"。你不再点按钮——你**写一个文本文件**描述："我想要一个集群、3 个节点、1 个网络、2 个 bucket"。然后跑一条命令，让某个工具把它创建出来。

这个文本文件可以：

- 放进 git，所有改动有 commit history
- PR review，team member 可以看你要改什么
- 直接复制给同事，他在他自己账号跑出来一样的环境
- 升级 / 回滚跟代码一样：`git revert` 然后重 apply

### 0.3 Terraform 在 IaC 里是什么角色

Terraform 是 IaC 工具的一种。它的工作方式：

```
你写 .tf 文件 (描述你想要什么)
        ↓
terraform plan (Terraform 跟云服务商对话，对比当前实际 vs 你想要的，列出 diff)
        ↓
terraform apply (Terraform 执行 diff：创建缺的、修改不一致的、删除多余的)
        ↓
云上资源 = 你 .tf 文件描述的样子
```

**最关键的一点**：Terraform 是 **声明式** 的，不是 **命令式** 的。

- 命令式：你写"开一台 VM、装 nginx、配防火墙、改 DNS"——一步步告诉它干什么
- **声明式**：你写"我想要一个开着 nginx 的 VM、防火墙开 80 端口"——告诉它**最终状态**长什么样，怎么达到由它决定

这就是为什么你可以**反复 apply**——Terraform 每次都对比"实际状态 vs 你描述的状态"，只做必要的改动。

### 0.4 我们项目的两条部署路径

| 集群目标 | 实际用什么工具 | 本教学假设 |
|---|---|---|
| **GKE Autopilot** (Google Cloud) | Terraform（已有） | 真实情况 |
| **Hetzner k3s** (Hetzner Cloud) | hetzner-k3s + cluster.yaml（**不是** Terraform） | **教学假设也用 Terraform** |

为什么真实 Hetzner 不用 Terraform？因为 `hetzner-k3s` 这个工具一站式做了"建 VM + 安装 k3s + 配集群"三件事，比 Terraform-only 路径省事。但**为了教学**，我们假设 Hetzner 也用 Terraform，这样可以横向对比 google 和 hcloud 两个 provider，把 Terraform 学透。

---

## 第 1 章：核心概念

### 1.1 四个核心动词

```
terraform init     # 第一次跑：下载 provider 插件
terraform plan     # 看一眼：现在 vs 想要的差异
terraform apply    # 执行：把差异落地
terraform destroy  # 拆除：把所有资源删了
```

| 动词 | 干啥 | 何时跑 | 危险度 |
|---|---|---|---|
| `init` | 下载 provider 二进制（如 hashicorp/google）到 `.terraform/` 目录；初始化 backend | 第一次 / 添加新 provider / 升级 provider | 🟢 安全 |
| `plan` | 对比 state 跟 .tf 文件，**只读**模式列出会做什么 | 改完文件想验证 | 🟢 安全 |
| `apply` | 真做 plan 列出的事 — **创建/修改/删除真实资源** | review 完 plan 满意 | 🔴 不可逆（部分资源） |
| `destroy` | 把当前 state 里所有资源删了 | sandbox 环境清理 | 🔴 危险 |

### 1.2 State —— Terraform 怎么知道"现在云上有啥"

Terraform 在本地维护一个文件叫 `terraform.tfstate`（JSON 格式），里面记录："上次 apply 之后，云上长这个样子"。

```
        你的 .tf 文件                 terraform.tfstate           真实云上
   "我想要 VPC name=foo"      ↔    "VPC name=foo, id=net-123"    ↔   id=net-123
   "我想要 cluster name=bar"  ↔    "cluster bar, id=clu-456"     ↔   id=clu-456
       (期望状态)                    (Terraform 已知状态)              (实际状态)
```

`terraform plan` 做的事：
1. 读 `terraform.tfstate`（已知状态）
2. 跟 .tf 文件对比（期望状态）
3. 跑 API 询问云：你那边对吗？（实际状态）
4. 三方一致 → 啥都不做
5. 不一致 → 列出 plan：哪些要改

**几个 state 的常识**：

- state 文件**包含敏感信息**（API key 名、IP 等），不要 git commit
- 多人协作要用**远程 state**（如 GCS bucket、S3、Terraform Cloud），不能各自一份本地 state
- 一旦云上资源被你**手工删除**（绕过 Terraform），state 就跟实际不一致——下次 plan 会很奇怪

### 1.3 .tfstate 在我们项目里在哪

```
cloud/deployments/stacks/gke/infra/
├── terraform.tfstate          ← state（gitignored）
├── terraform.tfstate.backup   ← 上次 apply 前的快照
├── *.tf                        ← 我们写的代码（git checked in）
└── terraform.tfvars            ← 操作员自定义变量（gitignored）
```

`terraform.tfstate` **不会进 git**——它跟着你本机走。我们的项目目前没有用远程 state，是因为只有少数操作员在跑 apply。**生产化前应该改成远程 state**（GCS bucket）。

---

## 第 2 章：HCL 语法

Terraform 文件用 **HCL**（HashiCorp Configuration Language），一种声明式 DSL。文件后缀 `.tf`。

### 2.1 一个 .tf 文件的结构

```hcl
# 这是注释（# 或 // 都行）

terraform {                    # 块（block）
  required_version = ">= 1.6"  # 参数（argument）
}

variable "project_id" {        # 块带 label
  type    = string             # 参数
  default = "my-cloud-project"
}

resource "google_compute_network" "gke" {  # 块带 type + name 两个 label
  name = "my-vpc"
}
```

**核心结构 `block`**：

```hcl
块类型 "label1" "label2" {
  参数1 = 值1
  参数2 = 值2
  
  嵌套块 {
    嵌套参数 = 嵌套值
  }
}
```

### 2.2 常见块类型

| 块类型 | 干啥 | 例子 |
|---|---|---|
| `terraform` | 配 Terraform 自身（版本、provider 列表、backend） | 必有 1 个 |
| `provider` | 配某个 provider 的连接信息（如 GCP 项目、credentials） | 每个 provider 1 个 |
| `variable` | 定义输入参数 | 一个 `var.x` 一个 |
| `resource` | 创建一个云资源 | 多个，每个对应一个云对象 |
| `data` | **读取**已存在的云资源（不创建） | 较少 |
| `output` | apply 结束打印的输出 | 多个，给后续工具用 |
| `locals` | 局部变量（计算一次复用多次） | 可选 |
| `module` | 引用别人写的 .tf 模板 | 进阶 |

### 2.3 类型系统（5 大基本类型）

```hcl
variable "name"     { type = string  }   # "hello"
variable "count"    { type = number  }   # 42
variable "enabled"  { type = bool    }   # true / false
variable "list"     { type = list(string) }   # ["a", "b", "c"]
variable "map"      { type = map(string)  }   # { foo = "bar", baz = "qux" }

# object — 嵌套 schema
variable "config" {
  type = object({
    host = string
    port = number
  })
}
```

我们项目变量大多是 `string`，少数 `bool` / `list(string)`。

### 2.4 引用与表达式

引用别的资源 / 变量：

```hcl
var.cluster_name                         # 引用 variable "cluster_name"
google_compute_network.gke.id            # 引用 resource "google_compute_network" 名 "gke" 的 .id 属性
google_compute_network.gke.self_link     # 同上 .self_link 属性

"${var.cluster_name}-vpc"                # 字符串插值，结果如 "my-cluster-vpc"
"${var.cluster_name}-${var.region}"      # 多变量拼接
```

字符串插值 `"${...}"` 是最常用的——把变量值嵌进字符串里。

引用 module 输出 / data 读取：

```hcl
module.foo.output_name                   # 引用 module 的输出
data.google_project.current.project_id   # 引用 data 块查到的属性
```

### 2.5 实战：改一行变量看会发生什么

假设我们项目 `variables.tf` 里：

```hcl
variable "cluster_name" {
  default = "my-cluster"
}
```

而 `network.tf` 里：

```hcl
resource "google_compute_network" "gke" {
  name = "${var.cluster_name}-vpc"  # 实际渲染成 "my-cluster-vpc"
}
```

如果你改 `default = "my-test"`，下一次 `terraform plan` 会说：

```
~ resource "google_compute_network" "gke" {
    name: "my-cluster-vpc" → "my-test-vpc"  (forces replacement)
}
```

`forces replacement` 表示这个改动需要**删掉重建**——因为 GCP 的 VPC name 不能 in-place 改。`terraform apply` 会先 destroy 旧的再创建新的，集群可能会断网。

> **教训**：改某些参数 = 销毁重建。`terraform plan` 会标 `forces replacement` 或 `~ in-place`——前者危险，后者安全。

---

## 第 3 章：GKE — main.tf — Provider 配置

**项目位置**：`cloud/deployments/stacks/gke/infra/main.tf`

```hcl
# 文件 1/9: main.tf — Provider 配置

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

### 3.1 `terraform { }` 块逐行解

```hcl
terraform {
```

声明这是 Terraform 自身的配置块。每个项目有 1 个。

```hcl
  required_version = ">= 1.6"
```

最低 Terraform CLI 版本 1.6。如果操作员本地 Terraform 是 1.5，跑 plan 会立刻报错。**为什么要写**：避免不同人本地 Terraform 版本太旧，跑出怪错误。

```hcl
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 6.10"
    }
```

声明这个项目用 `hashicorp/google` 这个 provider 插件，版本约束 `~> 6.10`（即 6.10.x，不允许 6.11+）。

- `source = "hashicorp/google"` 等于 Terraform Registry 上的 `registry.terraform.io/hashicorp/google`，是 Google 官方维护的 provider
- `version = "~> 6.10"` 用 `~>` 是"悲观约束"——允许 6.10.0, 6.10.1, ..., 但不允许 6.11.0
  - `>= 6.10` 允许 6.11, 6.12...（更松）
  - `= 6.10.0` 严格固定（更紧）

```hcl
    google-beta = {
      source  = "hashicorp/google-beta"
      version = "~> 6.10"
    }
  }
}
```

`google-beta` 是同一家 Google 提供的"beta API" provider。GKE 一些较新功能（如 Autopilot 的某些字段）只在 google-beta 提供，所以我们同时引两个。

### 3.2 `provider "google" { }` 块逐行解

```hcl
provider "google" {
  project = var.project_id
  region  = var.region
}
```

具体配置 google provider 怎么连：

- `project = var.project_id` — 操作哪个 GCP 项目（值从 `variables.tf` 来）
- `region = var.region` — 默认操作哪个区域（每个 resource 也可以自己 override）

**没写 credentials**：因为我们用 ADC（Application Default Credentials），由 `gcloud auth application-default login` 注入到 `~/.config/gcloud/application_default_credentials.json`。Terraform 自动找它，不需要在代码里硬写 credentials 路径——这样 .tf 文件可以放心 git checkin。

### 3.3 第二个 `provider "google-beta" { }`

跟上面一样配置，但是给 `google-beta` provider 用的。如果 main.tf 里某个 resource 写了 `provider = google-beta`，它就用这块配置。

### 3.4 这个文件你能改什么

| 想改的 | 改哪里 |
|---|---|
| 升级到 google provider 7.x | 改 `version = "~> 7.0"` |
| 改默认 GCP 项目 | 改 `var.project_id` 默认值（在 variables.tf） |
| 改默认区域 | 改 `var.region` 默认值（在 variables.tf） |

---

## 第 4 章：GKE — variables.tf — 输入参数

**项目位置**：`cloud/deployments/stacks/gke/infra/variables.tf`

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
  default     = "188.180.104.0/32"
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

### 4.1 一个 `variable` 块的结构

```hcl
variable "name" {       # name = 变量名，引用时用 var.name
  description = "..."   # 用途说明（plan/apply 时打印；新人看代码也靠这个）
  type        = string  # 类型（5 大基本类型之一，或 object）
  default     = "..."   # 默认值；不写 default 就**强制要求**操作员输入
}
```

### 4.2 8 个变量逐个看

| 变量 | 类型 | 默认 | 用来做什么 |
|---|---|---|---|
| `project_id` | string | `my-cloud-project` | 整个 stack 都用它，决定资源在哪个 GCP 项目 |
| `region` | string | `europe-west3` | Frankfurt（德国），跟 Hetzner 物理就近 |
| `cluster_name` | string | `my-cluster` | GKE 集群名，也是 VPC / NAT 等资源名前缀 |
| `admin_cidr` | string | `188.180.104.0/32` | 你的家庭 IP（操作 kubectl 必备） |
| `master_cidr` | string | `172.16.0.0/28` | 私有控制面 CIDR，必须 /28，跟你 VPC 不能冲突 |
| `ar_repo_id` | string | `my-app` | Artifact Registry repo 名 |
| `websites_bucket_suffix` | string | `websites-data` | bucket 名后缀，完整名 `my-cloud-project-websites-data` |
| `audit_bucket_suffix` | string | `audit` | 审计 bucket 后缀 |

### 4.3 三种覆盖默认的方式

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

   `terraform.tfvars` 会被自动加载。这个文件**不进 git**（`.gitignore` 排除），存你的私人项目 ID / IP。

我们项目还有 `terraform.tfvars.example`（**进 git**）作为模板：

```hcl
# project_id = "my-cloud-project"      # 默认就是这个
# admin_cidr = "1.2.3.4/32"           # 务必改成你的 /32
# cluster_name = "my-cluster-test" # sandbox 用 -test 后缀
```

### 4.4 派生变量（不直接定义，从其他变量算出）

我们 GKE 有几个"派生"名字，不是变量但靠变量拼出来。比如 storage.tf：

```hcl
resource "google_storage_bucket" "websites" {
  name = "${var.project_id}-${var.websites_bucket_suffix}"
  # 渲染成: "my-cloud-project-websites-data"
}
```

**好处**：你只改 `project_id` 一个地方，bucket 名 + AR 完整路径全部跟着变。新建项目从零部署只需改 `project_id`，剩下名字自动连贯。

---

## 第 5 章：GKE — network.tf — VPC, subnet, NAT

**项目位置**：`cloud/deployments/stacks/gke/infra/network.tf`

```hcl
resource "google_compute_network" "gke" {
  name                    = "${var.cluster_name}-vpc"
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "gke" {
  name          = "${var.cluster_name}-subnet"
  ip_cidr_range = "10.10.0.0/20"
  region        = var.region
  network       = google_compute_network.gke.id

  secondary_ip_range {
    range_name    = "pods"
    ip_cidr_range = "10.20.0.0/14"
  }
  secondary_ip_range {
    range_name    = "services"
    ip_cidr_range = "10.24.0.0/20"
  }

  private_ip_google_access = true
}

resource "google_compute_address" "nat" {
  name   = "${var.cluster_name}-nat"
  region = var.region
}

resource "google_compute_router" "nat" {
  name    = "${var.cluster_name}-nat-router"
  network = google_compute_network.gke.id
  region  = var.region
}

resource "google_compute_router_nat" "nat" {
  name                               = "${var.cluster_name}-nat"
  router                             = google_compute_router.nat.name
  region                             = var.region
  nat_ip_allocate_option             = "MANUAL_ONLY"
  nat_ips                            = [google_compute_address.nat.self_link]
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"

  log_config {
    enable = true
    filter = "ERRORS_ONLY"
  }
}
```

### 5.1 这文件管什么

VPC（虚拟私有网络）+ subnet（子网）+ NAT（对外出口）。**集群没有 VPC 跑不起来**——所有 Autopilot node 必须在某个 VPC 内。

### 5.2 第一个 resource：VPC

```hcl
resource "google_compute_network" "gke" {
  name                    = "${var.cluster_name}-vpc"
  auto_create_subnetworks = false
}
```

- `resource` 块声明"我要创建一个 VPC"
- 第一个 label `google_compute_network` = 资源**类型**（由 google provider 提供）
- 第二个 label `gke` = 资源**本地名字**（你自己起的，仅 Terraform 内部引用用）
- `name = "${var.cluster_name}-vpc"` = VPC 在 GCP 上**实际显示**的名字（如 "my-cluster-vpc"）
- `auto_create_subnetworks = false` = 关闭"自动建子网"——我们手工指定子网

> **本地名字 vs 实际名字**：`gke` 是你写代码引用时用的（如 `google_compute_network.gke.id`）；`my-cluster-vpc` 是 GCP 真给的名字。两者不需要一样。

### 5.3 第二个 resource：subnet

```hcl
resource "google_compute_subnetwork" "gke" {
  name          = "${var.cluster_name}-subnet"
  ip_cidr_range = "10.10.0.0/20"
  region        = var.region
  network       = google_compute_network.gke.id
```

- `name` — 实际名字
- `ip_cidr_range = "10.10.0.0/20"` — node 用的 CIDR，4096 个 IP
- `region = var.region` — 哪个区域
- `network = google_compute_network.gke.id` — 引用上面那个 VPC，让 Terraform 知道这个 subnet 属于哪个 VPC（同时建立**依赖关系**：先建 VPC 再建 subnet）

```hcl
  secondary_ip_range {
    range_name    = "pods"
    ip_cidr_range = "10.20.0.0/14"
  }
  secondary_ip_range {
    range_name    = "services"
    ip_cidr_range = "10.24.0.0/20"
  }
```

GKE 需要两个**次要 IP 段**——pod 网络（每 pod 一个 IP）和 service 网络。这是 GKE 特殊要求，普通 GCE VM 用不到。

- `pods` 给 `/14`（262,144 IP）— 一个 pod 一个 IP，500 pod 是九牛一毛
- `services` 给 `/20`（4,096 IP）— 一个 service 一个 IP

```hcl
  private_ip_google_access = true
}
```

让私有节点能访问 Google API（如 GCS、AR）即使没有 public IP。**Autopilot 私有节点的必需配置**——否则 pod 拉镜像、写 GCS 都会失败。

### 5.4 第三/四/五个 resource：NAT 三件套

```hcl
resource "google_compute_address" "nat" {
  name   = "${var.cluster_name}-nat"
  region = var.region
}
```

预留一个**静态外部 IP**——给 NAT 网关用。这就是我们项目"NAT egress IP"——**所有 pod 出公网都从这一个 IP 出**。DataImpulse 等代理服务可以把这个 IP 加白名单。

```hcl
resource "google_compute_router" "nat" {
  name    = "${var.cluster_name}-nat-router"
  network = google_compute_network.gke.id
  region  = var.region
}
```

NAT router——GCP 上 NAT 必须挂在一个 Router 上。

```hcl
resource "google_compute_router_nat" "nat" {
  router                             = google_compute_router.nat.name
  nat_ip_allocate_option             = "MANUAL_ONLY"
  nat_ips                            = [google_compute_address.nat.self_link]
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"
```

- `router` — 挂在上面那个 router 上
- `nat_ip_allocate_option = "MANUAL_ONLY"` — 不要 GCP 自动分配，用我们指定的 IP（即 `google_compute_address.nat`）
- `nat_ips = [...]` — 只用我们这一个静态 IP
- `source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"` — 整个 VPC 都通过这个 NAT 出公网

```hcl
  log_config {
    enable = true
    filter = "ERRORS_ONLY"
  }
}
```

打开 NAT 错误日志，方便日后调试连接问题。

### 5.5 资源依赖关系（Terraform 自动推断）

Terraform 自动从代码里**推断依赖顺序**：

```
google_compute_network.gke   (VPC)
    ↓ 被引用 ↓
google_compute_subnetwork.gke   (subnet 依赖 VPC)
    ↓ 被引用 ↓
google_compute_router.nat   (router 依赖 VPC)
    ↓ 被引用 ↓
google_compute_router_nat.nat  (NAT 依赖 router + IP)

google_compute_address.nat (独立)
    ↓ 被引用 ↓
google_compute_router_nat.nat   (NAT 依赖 IP)
```

`terraform apply` 会按拓扑顺序建：先 VPC + IP，再 subnet + router，最后 NAT。**你不需要手工写顺序**——它从 `.id`、`.name` 这种引用自动推。

---

## 第 6 章：GKE — cluster.tf — Autopilot 集群

**项目位置**：`cloud/deployments/stacks/gke/infra/cluster.tf`

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

### 6.1 顶部声明

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

- `name` — 集群名
- `location` — 区域级集群（3-zone HA）；如果填 zone（如 `europe-west3-a`）就是 zonal 集群（单可用区，便宜但不抗 zone 故障）

### 6.2 Autopilot 开关

```hcl
  enable_autopilot    = true
```

**这一行决定模式**：

- `true` → Autopilot（Google 管节点，按 pod 计费 / 按 ComputeClass 节点计费）
- `false` → Standard（你自己管 node pool，按节点计费）

我们用 Autopilot 是因为 inpage 工作负载是 bursty + ephemeral，不需要持续运行 node pool。

```hcl
  deletion_protection = false
```

允许 `terraform destroy`。生产环境应该 `true`，避免误删。

### 6.3 网络绑定

```hcl
  network    = google_compute_network.gke.id
  subnetwork = google_compute_subnetwork.gke.id
```

引用 network.tf 创建的 VPC + subnet。Terraform 自动建依赖：先 VPC → subnet → cluster。

```hcl
  ip_allocation_policy {
    cluster_secondary_range_name  = "pods"
    services_secondary_range_name = "services"
  }
```

告诉 GKE 用 subnet 的两个 secondary range（在 network.tf 定义的 `pods` 和 `services`）。

### 6.4 私有集群配置

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

- `enable_private_nodes = true` — 节点没有 public IP（更安全），出公网走 NAT
- `enable_private_endpoint = false` — 控制面（API server）有 public IP，但只允许下面 `master_authorized_networks_config` 里的 CIDR 访问
- `master_ipv4_cidr_block = var.master_cidr` — 控制面私有 IP CIDR（必须 /28，且与 VPC 不冲突）
- `master_global_access_config.enabled = true` — 全球可访问控制面（不限制只在 region 内）

### 6.5 控制面访问白名单

```hcl
  master_authorized_networks_config {
    cidr_blocks {
      cidr_block   = var.admin_cidr
      display_name = "admin"
    }
  }
```

只允许 `admin_cidr`（你的家 IP）连接控制面 API。如果操作员换了 IP，要改 `var.admin_cidr` 重 apply。

### 6.6 Workload Identity

```hcl
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }
```

启用 Workload Identity——让 Pod 能用 GCP IAM 权限访问 GCS / AR / 其他 GCP 服务，**不需要在 Pod 里挂 service account JSON key**。这是 GCP 推荐的最佳实践。

### 6.7 Release channel

```hcl
  release_channel {
    channel = "REGULAR"
  }
```

GKE 升级策略：

- `RAPID` — 最新版本，bug 多
- `REGULAR` — 主流稳定（我们用这个）
- `STABLE` — 最稳定，落后几个月

### 6.8 Addons：GCSFuse

```hcl
  addons_config {
    gcs_fuse_csi_driver_config {
      enabled = true
    }
  }
```

启用 GCSFuse CSI driver——让 Pod 能 mount GCS bucket 当本地目录用。**这就是我们 `data/websites/` 配置怎么进 pod 的**。

### 6.9 这文件你能改什么

| 想改的 | 改哪里 |
|---|---|
| 关闭私有节点 | `enable_private_nodes = false`（不推荐，安全降级）|
| 升 release channel | `channel = "RAPID"` |
| 加一个 admin IP | 加多个 `cidr_blocks { ... }` 块 |
| 关 GCSFuse | `enabled = false`（会破坏 web data 访问）|
| 切 Standard 模式 | `enable_autopilot = false` + 加 node pool 块（大改）|

---

## 第 7 章：GKE — storage.tf — GCS bucket

**项目位置**：`cloud/deployments/stacks/gke/infra/storage.tf`

```hcl
resource "google_storage_bucket" "websites" {
  name                        = "${var.project_id}-${var.websites_bucket_suffix}"
  location                    = var.region
  uniform_bucket_level_access = true
  force_destroy               = true

  versioning {
    enabled = true
  }

  lifecycle_rule {
    action { type = "Delete" }
    condition {
      num_newer_versions = 5
      with_state         = "ARCHIVED"
    }
  }
}

resource "google_storage_bucket" "audit" {
  name                        = "${var.project_id}-${var.audit_bucket_suffix}"
  location                    = var.region
  uniform_bucket_level_access = true
  force_destroy               = true

  lifecycle_rule {
    action { type = "Delete" }
    condition {
      age = 90
    }
  }
}
```

### 7.1 第一个 bucket：websites

```hcl
resource "google_storage_bucket" "websites" {
  name = "${var.project_id}-${var.websites_bucket_suffix}"
  # 渲染成: "my-cloud-project-websites-data"
```

bucket name **全球唯一**——所以拼了 project_id 前缀避免撞名。

```hcl
  location = var.region
```

bucket 在哪个区域。注意 GCS 有多种 location 类型：region（如 `europe-west3`）、multi-region（如 `EU`）、dual-region。我们用单区域。

```hcl
  uniform_bucket_level_access = true
```

启用统一 IAM 模型（推荐）。关掉的话同时支持 ACL（旧式权限），管理混乱。

```hcl
  force_destroy = true
```

`terraform destroy` 时允许销毁带数据的 bucket（默认 false，会保护防误删）。我们 sandbox 用 true，**生产应改 false**。

```hcl
  versioning {
    enabled = true
  }
```

打开对象版本控制——同名文件覆写时保留旧版本（有点像 git）。给"误删保护"用。

```hcl
  lifecycle_rule {
    action { type = "Delete" }
    condition {
      num_newer_versions = 5
      with_state         = "ARCHIVED"
    }
  }
}
```

生命周期规则：**已归档的旧版本如果同名新版本超过 5 个就删**。避免无限堆积。

### 7.2 第二个 bucket：audit

```hcl
resource "google_storage_bucket" "audit" {
  ...
  lifecycle_rule {
    action { type = "Delete" }
    condition {
      age = 90
    }
  }
}
```

跟 websites 类似，但生命周期规则是**90 天后自动删**。审计数据保留 3 个月，符合大多数合规要求。

### 7.3 总结

两个 bucket，都 `pd-balanced` 默认（GCS 自己的存储，跟 GCE SSD 配额无关）。命名 derived from `project_id`，自动适应不同部署。

---

## 第 8 章：GKE — iam.tf — Service Account + 权限

**项目位置**：`cloud/deployments/stacks/gke/infra/iam.tf`（略选关键部分）

```hcl
# 创建 Service Account
resource "google_service_account" "event_job" {
  account_id   = "event-job"
  display_name = "GSA: Event Job (single-step scraping)"
}

# 绑定 Workload Identity（让 K8s ServiceAccount 能"扮演"这个 GSA）
resource "google_service_account_iam_member" "event_job_wi" {
  service_account_id = google_service_account.event_job.id
  role               = "roles/iam.workloadIdentityUser"
  member             = "serviceAccount:${var.project_id}.svc.id.goog[browser-service/event-agent-sa]"
}

# 给 GSA 在 audit bucket 上 objectAdmin 权限（写 audit 数据）
resource "google_storage_bucket_iam_member" "audit_writer_event" {
  bucket = google_storage_bucket.audit.name
  role   = "roles/storage.objectAdmin"
  member = "serviceAccount:${google_service_account.event_job.email}"
}
```

### 8.1 Service Account（GSA）= GCP 这边的"权限主体"

```hcl
resource "google_service_account" "event_job" {
  account_id   = "event-job"
  display_name = "GSA: Event Job (single-step scraping)"
}
```

GSA = Google Service Account。可以理解为"**一个能访问 GCP 资源的虚拟机器人账号**"。

- `account_id = "event-job"` 决定了它的邮箱：`event-job@{project}.iam.gserviceaccount.com`

### 8.2 Workload Identity 绑定 = 给 K8s pod 配 GCP 权限

```hcl
resource "google_service_account_iam_member" "event_job_wi" {
  service_account_id = google_service_account.event_job.id
  role               = "roles/iam.workloadIdentityUser"
  member             = "serviceAccount:${var.project_id}.svc.id.goog[browser-service/event-agent-sa]"
}
```

读法："让 K8s ServiceAccount `browser-service/event-agent-sa` 能扮演（impersonate）GSA `event-job`"。

- `member` 的 syntax `serviceAccount:{project}.svc.id.goog[{namespace}/{ksa-name}]` 是 Workload Identity 专用格式
- `roles/iam.workloadIdentityUser` 是允许"扮演"的权限

**结果**：你的 pod 用 `event-agent-sa` 这个 K8s ServiceAccount 启动后，自动获得 `event-job` GSA 在 GCP 上的所有权限——但**整个过程不需要 JSON key 文件**，纯 Workload Identity 短期 token。

### 8.3 GSA 拿到的实际权限

```hcl
resource "google_storage_bucket_iam_member" "audit_writer_event" {
  bucket = google_storage_bucket.audit.name
  role   = "roles/storage.objectAdmin"
  member = "serviceAccount:${google_service_account.event_job.email}"
}
```

这才是给 GSA "实际能干啥"的权限：

- 在 `my-cloud-project-audit` bucket 上有 `storage.objectAdmin` 权限
- `objectAdmin` 包含 read / write / delete 对象——比 `objectViewer` 强，比 `admin` 弱（不能改 bucket 配置）

### 8.4 串起来：pod 写 audit 的链路

```
Pod (browser-service/event-agent-sa K8s ServiceAccount)
    ↓ Workload Identity binding
GSA: event-job@{project}.iam.gserviceaccount.com
    ↓ storage.objectAdmin role
GCS bucket: my-cloud-project-audit
    ↓ 写
gs://my-cloud-project-audit/multi-step/{site}/{sport}-{key8}/{date}/scheduled.json
```

每个箭头都是一个 IAM 检查点。**少一个**绑定，整条链路就断。我们之前踩的坑（`storage.objectAdmin missing`）就是在第三个箭头那里——只给了 `objectViewer` 没给 `objectAdmin`。

### 8.5 这文件你能改什么

| 想改的 | 改哪里 |
|---|---|
| 加一个新的 GSA（如新工作负载） | 复制一个 `google_service_account` 块 |
| 给现有 GSA 加新 bucket 权限 | 加一个 `google_storage_bucket_iam_member` 块 |
| 收紧权限（如 viewer 而非 admin） | 改 `role = "roles/storage.objectViewer"` |
| 改 K8s ServiceAccount 名字 | 改 `member` 里 `[browser-service/event-agent-sa]` 的 SA 名 |

---

## 第 9 章：GKE — outputs.tf — 输出

**项目位置**：`cloud/deployments/stacks/gke/infra/outputs.tf`（关键部分）

```hcl
output "cluster_name" {
  value = google_container_cluster.autopilot.name
}

output "region" {
  value = var.region
}

output "ar_repo_url" {
  value = "${var.region}-docker.pkg.dev/${var.project_id}/${var.ar_repo_id}"
}

output "websites_bucket" {
  value = google_storage_bucket.websites.name
}

output "audit_bucket" {
  value = google_storage_bucket.audit.name
}

output "k8s_manager_gsa_email" {
  value = google_service_account.k8s_manager.email
}

output "nat_egress_ip" {
  value = google_compute_address.nat.address
}

output "cluster_endpoint" {
  value     = google_container_cluster.autopilot.endpoint
  sensitive = true
}
```

### 9.1 output 是干嘛的

`output` 块声明 Terraform apply 完之后**对外暴露什么值**。三个用途：

1. **打印给操作员看**——apply 成功后终端打印
2. **其他工具读**——如 `use-gke.sh` 脚本调 `terraform output -raw cluster_name` 拿值
3. **跨 module 引用**——如果将来用 module 化，外部 module 通过 output 读子 module 的值

### 9.2 `value = ...` 的写法

```hcl
output "cluster_name" {
  value = google_container_cluster.autopilot.name
}
```

可以引用任何 resource 的属性。`google_container_cluster.autopilot.name` 就是从 cluster.tf 那个 cluster 资源拿 `name` 属性。

```hcl
output "ar_repo_url" {
  value = "${var.region}-docker.pkg.dev/${var.project_id}/${var.ar_repo_id}"
  # 渲染成: europe-west3-docker.pkg.dev/my-cloud-project/my-app
}
```

可以拼字符串——AR repo 完整 URL 不在任何资源属性里直接给（GCP API 不返回这个 URL），但用 region + project + repo 能拼出来。

### 9.3 `sensitive = true`

```hcl
output "cluster_endpoint" {
  value     = google_container_cluster.autopilot.endpoint
  sensitive = true
}
```

`cluster_endpoint` 是 API server 地址，敏感。Terraform 会在 plan/apply 输出里把这个值打码成 `<sensitive>`，避免漏到日志。

读它得用 `terraform output -raw cluster_endpoint`。

### 9.4 我们项目怎么用 outputs

```bash
terraform output -raw cluster_name           # → "my-cluster"
terraform output -raw ar_repo_url            # → "europe-west3-docker.pkg.dev/my-cloud-project/my-app"
terraform output -json                       # → 所有 outputs 的 JSON
```

`use-gke.sh` 脚本就是调 `terraform output -json` 把所有输出读进 shell 变量，再 envsubst 渲染 .env 文件。

---

## 第 10 章：Hetzner — 假设用 Terraform 怎么写

> ⚠️ **现实情况**：本项目 Hetzner 路径用的是 `hetzner-k3s` + cluster.yaml，**不是** Terraform。本章是**教学假设**——假设我们用 Terraform 重写一遍，会长什么样。
>
> 这种对比对学习有价值，因为同一套 Terraform 工具，换了 provider 就能管完全不同的云。

### 10.1 Hetzner Cloud Terraform Provider

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

### 10.2 假设的项目结构

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

### 10.3 main.tf — 类比 GKE

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

### 10.4 variables.tf

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
  default     = "188.180.104.0/32"
  type        = string
}
```

**注意点**：

- `hcloud_token` 标 `sensitive = true` ——log/output 不会泄露
- `control_plane_type = "cx43"` 是 Hetzner 的实例规格代号（8 vCPU 32 GB），**类比 GKE 我们的 `n2-standard-16`** 但 Hetzner 是固定规格，不像 GCP 那么多种类
- `location = "fsn1"` Falkenstein（德国 Hetzner 主机房之一）

### 10.5 network.tf — 类比 GKE 但是简化

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

**对比 GKE network.tf**：

| 概念 | GKE | Hetzner |
|---|---|---|
| 网络 | `google_compute_network` | `hcloud_network` |
| 子网 | `google_compute_subnetwork` | `hcloud_network_subnet` |
| 防火墙 | （没单独建——靠 master_authorized_networks_config） | `hcloud_firewall` |
| 访问控制 | 通过 cluster 配置 | 通过独立 firewall 资源 + 绑到 server |

### 10.6 ssh.tf — Hetzner 特有

```hcl
resource "hcloud_ssh_key" "admin" {
  name       = "${var.cluster_name}-admin"
  public_key = var.admin_ssh_public_key
}
```

GKE 用 IAM 配 kubectl 访问，**不需要**给节点 SSH——节点是 managed 的。Hetzner 不一样，节点是你的 VM，要 SSH 上去做事。

### 10.7 servers.tf — 这是 Hetzner Terraform 最大的部分

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

**新概念**：

- `count = var.worker_count`：让一个 resource 复制 N 份。N=3 就建 3 个 worker。引用第 i 个用 `hcloud_server.worker[i]`
- `count.index`：循环里的"第几个"，从 0 开始。`count.index + 1` 就是 1, 2, 3...用来生成 `worker-w1, worker-w2, worker-w3` 这样的名字
- `templatefile()`：读 cloud-init 模板，把变量代入。这就是 Hetzner 怎么"在新建 VM 上自动装 k3s"——通过 cloud-init 脚本

**`user_data` 是关键**：

GKE 不需要这个，因为 Autopilot 内置 K8s 自动起。Hetzner 得自己装 k3s。Cloud-init 模板大概长这样：

```yaml
# cloud-init/control-plane.yaml
#cloud-config
runcmd:
  - curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.34.6+k3s1 K3S_TOKEN=${cluster_token} sh -
  - cp /etc/rancher/k3s/k3s.yaml /root/kubeconfig.yaml
```

控制面节点装完 k3s，worker 节点用类似脚本但传 `K3S_URL=https://${control_plane_ip}:6443` join 进集群。

### 10.8 outputs.tf

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

`for w in ...` 是 Terraform 的 for expression——遍历数组生成新数组。Python list comprehension 类似。

### 10.9 这套 Hetzner Terraform 对比真实项目

真实项目（hetzner-k3s + cluster.yaml）：

```yaml
# cluster.yaml (摘自我们项目)
cluster_name: my-app
kubeconfig_path: "~/.kube/hetzner-config"
k3s_version: v1.34.6+k3s1

masters_pool:
  instance_type: cx43
  instance_count: 1
  location: fsn1

worker_node_pools:
  - name: pool-1
    instance_type: cx33
    instance_count: 3
    location: fsn1
```

**hetzner-k3s 一站式做的事 = 我们 Terraform 设计的 servers.tf + cloud-init**。优势：少写代码。劣势：少了 Terraform 的状态管理 / plan 预览 / 跨资源依赖追踪。

---

## 第 11 章：GKE vs Hetzner 对照

### 11.1 一表看完两边的核心差异

| 概念 | GKE Terraform | Hetzner Terraform |
|---|---|---|
| Provider | `hashicorp/google` | `hetznercloud/hcloud` |
| 认证 | ADC（环境注入） | `token = var.hcloud_token` 显式 |
| 网络 | `google_compute_network` + 2 个 secondary range（pods/services） | `hcloud_network` + `hcloud_network_subnet`（无 K8s-specific 段）|
| 子网大小 | nodes /20, pods /14, services /20 | 一个 /24（K8s 内部 CNI 自管 pod IP） |
| 防火墙 | 通过 cluster `master_authorized_networks_config` | 独立 `hcloud_firewall` resource，绑到 server |
| NAT / 出口 IP | `google_compute_address` + `google_compute_router_nat` | 默认每 VM 自带 public IP，或用 `hcloud_floating_ip` 浮动 IP |
| 节点 | Autopilot 自动管，不写代码 | 显式 `hcloud_server` 资源，加 `user_data` cloud-init |
| K8s 安装 | Autopilot 内置 | cloud-init 跑 `curl get.k3s.io` |
| 存储 | `google_storage_bucket` + GCSFuse mount | （没原生对象存储 provider；Hetzner Object Storage 是 S3-compatible，得 boto3 用） |
| Service Account | `google_service_account` + Workload Identity | （Hetzner 没内置 IAM，pod 凭据要自己管 secret） |
| 配额 | 区域级 GCE quota（SSD_TOTAL_GB, CPUS, INSTANCES）卡住扩容 | 项目级 server limit（默认每项目 10 台 VM 上限），但很容易申请加 |
| 计费 | 按 pod / 节点资源（Autopilot 模式区分） | 按 VM 类型固定月费（cx33 ~€4/月） |
| 扩容速度 | Autopilot 自动 1-3 min | hetzner-k3s 自动扩，但要自己 hetzner-k3s 命令；纯 Terraform 要改 var.worker_count + apply |

### 11.2 Terraform 哲学保持不变

虽然 provider 不同，但 Terraform 的**核心范式**两边一样：

1. **写 .tf 文件描述期望状态**
2. **`terraform plan` 看 diff**
3. **`terraform apply` 落地**
4. **`terraform.tfstate` 跟踪当前状态**
5. **`terraform destroy` 拆光**

**学完一个 provider，第二个就很快上手**——只要查"这个 provider 有哪些 resource type" 就行。

### 11.3 引申：你能给任何云写 Terraform

Terraform 支持 100+ provider，覆盖所有主流云 + 第三方服务：

- AWS: `hashicorp/aws`
- Azure: `hashicorp/azurerm`
- GCP: `hashicorp/google` ← 我们用的
- Hetzner: `hetznercloud/hcloud` ← 假设用的
- Cloudflare（DNS / WAF）: `cloudflare/cloudflare`
- DigitalOcean: `digitalocean/digitalocean`
- GitHub（管 repo / team）: `integrations/github`
- Datadog（监控）: `DataDog/datadog`

**多云 Terraform** 是常见模式：一个 `main.tf` 同时声明 GCP + AWS + Cloudflare 资源。你写一个 PR 改完 → review → apply，跨云改动一气呵成。

---

## 第 12 章：实战练习

### 练习 1：读懂下面这段代码做什么

```hcl
resource "google_compute_address" "extra_egress" {
  name         = "${var.cluster_name}-extra-egress"
  address_type = "EXTERNAL"
  region       = var.region
}
```

**思考**：这个 resource type 是什么？属于哪个文件？建出来是什么？

**答案**：
- `google_compute_address` = 静态外部 IP
- 应该放在 `network.tf`（属于网络层）
- 建出来一个全球唯一的静态 public IPv4 地址，名字 `my-cluster-extra-egress`

### 练习 2：改 admin_cidr 让另一个人也能 kubectl

你的同事在 `1.2.3.4`，你想让他也能 kubectl GKE 集群。怎么改？

**答案**：

cluster.tf 里 `master_authorized_networks_config` 现在只有一个 cidr_blocks 块。可以加一个：

```hcl
master_authorized_networks_config {
  cidr_blocks {
    cidr_block   = var.admin_cidr
    display_name = "admin"
  }
  cidr_blocks {
    cidr_block   = "1.2.3.4/32"
    display_name = "colleague"
  }
}
```

或者更优雅——把 admin_cidr 改成 list：

```hcl
# variables.tf
variable "admin_cidrs" {
  type    = list(object({ cidr = string, name = string }))
  default = [
    { cidr = "188.180.104.0/32", name = "admin" },
    { cidr = "1.2.3.4/32",       name = "colleague" },
  ]
}

# cluster.tf
master_authorized_networks_config {
  dynamic "cidr_blocks" {
    for_each = var.admin_cidrs
    content {
      cidr_block   = cidr_blocks.value.cidr
      display_name = cidr_blocks.value.name
    }
  }
}
```

`dynamic` 块是 Terraform 高级特性——根据列表动态生成多个嵌套块。

### 练习 3：加一个新 GCS bucket

需求：加一个名为 `{project}-backups` 的 bucket，180 天后自动删。

**答案**（加到 storage.tf）：

```hcl
resource "google_storage_bucket" "backups" {
  name                        = "${var.project_id}-backups"
  location                    = var.region
  uniform_bucket_level_access = true
  force_destroy               = false   # backup 谨慎一些

  lifecycle_rule {
    action { type = "Delete" }
    condition {
      age = 180
    }
  }
}
```

跑 `terraform plan` 会看到 `+ google_storage_bucket.backups`，apply 后 GCP 上就有这个 bucket 了。

### 练习 4：把当前 Hetzner cluster.yaml 翻成 Terraform

我们项目 `cloud/deployments/stacks/hetzner/infra/cluster.yaml` 节选：

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
```

**翻译成 Terraform**（基于第 10 章的设计）：

```hcl
# variables.tf
variable "control_plane_type" { default = "cx43" }
variable "worker_type"        { default = "cx33" }
variable "worker_count"       { default = 3 }
variable "location"           { default = "fsn1" }

# servers.tf
resource "hcloud_server" "control_plane" {
  server_type = var.control_plane_type
  location    = var.location
  # ...
}

resource "hcloud_server" "worker" {
  count       = var.worker_count
  server_type = var.worker_type
  location    = var.location
  # ...
}
```

但是！**hetzner-k3s 自带的 cluster autoscaler 集成**（动态扩 worker）就没了——纯 Terraform 改 worker_count 是手动 plan + apply。这是为什么生产环境我们用 hetzner-k3s 而不是纯 Terraform。

---

## 第 13 章：FAQ

### Q1: Terraform 跟 K8s 是什么关系？

**Terraform 管 IaaS（基础设施）**：VPC、VM、bucket、IAM。在 K8s 集群**之外**。

**K8s 管 PaaS（平台）**：pod、service、deployment、secret。在 K8s 集群**之内**。

Terraform 把"集群"这个对象建出来，但**集群里的 deployment / pod 不在 Terraform 管辖**。我们项目里 K8s 资源用 `kubectl apply -k` + kustomize 管。

理论上 Terraform 也能管 K8s 资源（用 `kubernetes` provider），但工作流不顺手——K8s manifests 改动很频繁，`terraform apply` 比 `kubectl apply` 慢。所以业界惯例：**Terraform 管底，kubectl/kustomize/helm 管上**。

### Q2: 改了 .tf 文件不 apply 会怎样？

**啥都不会发生**。Terraform 只在你跑 `apply` 时改实际资源。改文件只是改"期望状态"。

但下次别人 / 你 跑 plan 会看到 diff。

### Q3: 别人 apply 过我没 apply 怎么办？

如果你们**共享 state**（远程 state 在 GCS bucket），你 plan 之前会自动拉最新 state，就跟别人对齐了。

如果你们**各自本地 state**——这是 bug，各自 state 会发散。**生产应该用远程 state**：

```hcl
# main.tf
terraform {
  backend "gcs" {
    bucket = "my-app-terraform-state"
    prefix = "gke-infra/"
  }
}
```

### Q4: terraform.tfstate 误删了怎么办？

灾难。要么从备份恢复（`terraform.tfstate.backup` 是上一次 apply 前的）。要么 `terraform import` 把每个云上资源重新"认领"进 state（耗时、易错）。

**生产要做**：远程 state + state 备份策略。

### Q5: Terraform vs Pulumi vs CloudFormation 有什么区别？

| 工具 | 谁的 | 哪种语言 | 优劣 |
|---|---|---|---|
| **Terraform** | HashiCorp（被 IBM 收购） | HCL（自家 DSL） | 开源、跨云、社区最大 |
| **Pulumi** | 创业公司 | TypeScript / Python / Go（你熟的语言）| 学习曲线低，但社区小 |
| **CloudFormation** | AWS 官方 | YAML / JSON | 只支持 AWS，但 AWS 用得很顺 |
| **CDK** | AWS / Cloud Development Kit | TS / Python 生成 CFN | "TS 写云"，AWS 主流推 |

我们用 Terraform 的原因：跨云（GCP + Hetzner）、HCL 写 IaC 比通用语言更可读、社区文档丰富。

### Q6: 我改了 quota（如 SSD_TOTAL_GB），要更新 Terraform 吗？

**不用**。配额是 GCP 项目级账号属性，不在任何 Terraform resource 里——你在 Console 申请，Google 批了之后立刻生效。Terraform 看不到也不需要看。

### Q7: Terraform 能滚动升级 K8s 集群版本吗？

可以，改 cluster.tf 里加 `min_master_version = "1.36"`，apply 触发升级。但 GKE 升级是分 zone 进行，可能造成短暂中断。**生产建议手工 + 灰度**，不建议 Terraform 自动化。

---

## 学完之后

如果你认真读完上面 13 章，你应该能：

✅ 看懂任何 `.tf` 文件
✅ 修改我们项目的 GKE 部署（变量 / 资源 / 输出 / 权限）
✅ 给一个新项目从零写 Terraform
✅ 在 GCP / Hetzner / 任何云之间迁移知识

**下一步学习**：
- Terraform module（把 .tf 文件打包成可复用的"组件"）
- 远程 state（生产必备）
- Workspace（同一份代码部署到 dev/staging/prod 多环境）
- 与 GitHub Actions 集成（PR 自动 plan、merge 自动 apply）

每一项都是新的 1-2 小时学习量。读完本教程已经覆盖了**生产 80%** 的常用功能。

---

> 教程作者：本项目工程团队 | 生成时间：2026-05-01 | 反馈：在 GitHub Issue 留言
