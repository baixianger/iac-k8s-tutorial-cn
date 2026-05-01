---
layout: default
title: 02 — HCL 语法
---

# 第 02 章：HCL 语法

> 上一章：[01 — IaC 与 Terraform 基础](01-iac-and-terraform.html) · [章节索引](./)

Terraform 文件用 **HCL**（HashiCorp Configuration Language），一种声明式 DSL。文件后缀 `.tf`。

这章读完之后，你看到任何 `.tf` 文件至少不会迷路 —— 知道哪是块、哪是参数、哪是引用。

## 1. 一个 .tf 文件的结构

```hcl
# 这是注释（# 或 // 都行）
/*
  也支持 C 风格的多行注释
*/

terraform {                    # 块（block）
  required_version = ">= 1.6"  # 参数（argument）
}

variable "project_id" {        # 块带 1 个 label
  type    = string             # 参数
  default = "my-cloud-project"
}

resource "google_compute_network" "gke" {  # 块带 2 个 label
  name = "my-vpc"
}
```

**核心结构是 `block`**：

```hcl
块类型 "label1" "label2" {
  参数1 = 值1
  参数2 = 值2
  
  嵌套块 {
    嵌套参数 = 嵌套值
  }
}
```

`label1` / `label2` 是块的"名字" —— 不同块类型的 label 数量不同：

| 块类型 | label 数量 | 例子 |
|---|---|---|
| `terraform` | 0 | `terraform { ... }` |
| `provider` | 1 | `provider "google" { ... }` |
| `variable` | 1 | `variable "project_id" { ... }` |
| `resource` | 2 | `resource "google_compute_network" "gke" { ... }` |
| `data` | 2 | `data "google_project" "current" { ... }` |
| `output` | 1 | `output "cluster_name" { ... }` |

`resource` 的两个 label：第一个是**类型**（provider 决定有哪些类型），第二个是你自己起的**本地名**（在同一个 `.tf` 项目里要唯一，但只是给其他 `.tf` 引用用，跟云上实际名字没关系）。

## 2. 七个常见块类型

| 块类型 | 干啥 | 项目里多少个 |
|---|---|---|
| `terraform` | 配 Terraform 自身（版本、provider 列表、backend） | 必有 1 个 |
| `provider` | 配某个 provider 的连接信息（如 GCP 项目、credentials） | 每个 provider 1 个 |
| `variable` | 定义输入参数 | 一个 `var.x` 一个 |
| `resource` | 创建一个云资源 | 多个，每个对应一个云对象 |
| `data` | **读取**已存在的云资源（不创建） | 较少 |
| `output` | apply 结束打印的输出 | 多个，给后续工具用 |
| `locals` | 局部变量（计算一次复用多次） | 可选 |
| `module` | 引用别人写的 .tf 模板 | 进阶 |

`resource` vs `data` 的差别要记住：

- `resource` = "请帮我创建一个" —— Terraform 会去 API 创建对应资源
- `data` = "请告诉我这个资源是啥样的" —— 只读，不创建任何东西

例子：

```hcl
# 创建一个新的 VPC
resource "google_compute_network" "gke" {
  name = "my-vpc"
}

# 读取已经存在的 GCP 项目元信息
data "google_project" "current" {}

# 用读到的属性
output "project_number" {
  value = data.google_project.current.number
}
```

## 3. 类型系统（5 大基本类型）

```hcl
variable "name"     { type = string  }   # "hello"
variable "count"    { type = number  }   # 42
variable "enabled"  { type = bool    }   # true / false
variable "list"     { type = list(string) }   # ["a", "b", "c"]
variable "map"      { type = map(string)  }   # { foo = "bar", baz = "qux" }
```

复合类型 —— `object` 嵌套 schema：

```hcl
variable "cluster_config" {
  type = object({
    name      = string
    node_count = number
    zones     = list(string)
  })

  default = {
    name       = "my-cluster"
    node_count = 3
    zones      = ["europe-west3-a", "europe-west3-b", "europe-west3-c"]
  }
}
```

实际项目里大部分变量是 `string`，少数 `bool` / `list(string)`。`object` 用得不多 —— 因为 Terraform 写起来就是堆变量，不需要太多嵌套。

## 4. 引用与表达式

引用别的资源 / 变量：

```hcl
var.cluster_name                         # 引用 variable "cluster_name"
google_compute_network.gke.id            # 引用 resource 的 .id 属性
google_compute_network.gke.self_link     # 引用 resource 的 .self_link 属性

"${var.cluster_name}-vpc"                # 字符串插值，结果如 "my-cluster-vpc"
"${var.cluster_name}-${var.region}"      # 多变量拼接
```

字符串插值 `"${...}"` 是最常用的 —— 把变量值嵌进字符串里。

引用 `module` 输出 / `data` 读取：

```hcl
module.foo.output_name                   # 引用 module 的输出
data.google_project.current.project_id   # 引用 data 块查到的属性
```

引用 `local`（局部变量）：

```hcl
locals {
  bucket_name = "${var.project_id}-${var.suffix}"
  tags        = { env = "prod", team = "data" }
}

resource "google_storage_bucket" "audit" {
  name = local.bucket_name    # 注意是 local.x，不是 locals.x
}
```

## 5. 函数（少量但常用）

HCL 内置一批函数：

```hcl
length(["a", "b", "c"])                  # → 3
upper("hello")                           # → "HELLO"
lower("HELLO")                           # → "hello"
join("-", ["foo", "bar"])                # → "foo-bar"
split("-", "foo-bar-baz")                # → ["foo", "bar", "baz"]
file("${path.module}/script.sh")         # 读文件
templatefile("conf.tpl", { ip = "1.2.3.4" })  # 渲染模板
jsonencode({ a = 1, b = "two" })         # → "{\"a\":1,\"b\":\"two\"}"
yamldecode(file("config.yaml"))          # YAML → object
```

详细列表：<https://developer.hashicorp.com/terraform/language/functions>

实际项目最常用的就是 `templatefile`（生成 cloud-init / 用户脚本）和 `file`（读 SSH key 文件）。

## 6. 条件与循环

**三元运算**：

```hcl
resource "google_storage_bucket" "audit" {
  name     = var.enable_audit ? "${var.project_id}-audit" : "disabled"
  location = var.region
}
```

**count 循环**：根据数字创建 N 份资源：

```hcl
resource "google_compute_address" "node_ip" {
  count = 3                              # 创建 3 个
  name  = "node-ip-${count.index}"       # node-ip-0, node-ip-1, node-ip-2
}
```

**for_each 循环**：根据 map / set 创建多份：

```hcl
resource "google_storage_bucket" "data" {
  for_each = {
    audit    = "audit"
    websites = "websites-data"
  }
  name = "${var.project_id}-${each.value}"
}
```

`for_each` 比 `count` 更好 —— `count` 删一个会让后面所有 index 后移（破坏性 reorder），`for_each` 用 key 索引则不会。

**列表推导**（少用，但偶尔有用）：

```hcl
locals {
  node_names = [for i in range(3) : "node-${i}"]
  # → ["node-0", "node-1", "node-2"]
}
```

## 7. 实战：改一行变量看会发生什么

假设 `variables.tf` 里：

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

如果你把 `default` 改成 `"my-test"`，下一次 `terraform plan` 会说：

```diff
~ resource "google_compute_network" "gke" {
~   name: "my-cluster-vpc" → "my-test-vpc"  # forces replacement
}
```

`forces replacement` 表示这个改动需要 **删掉重建** —— 因为 GCP 的 VPC name 不能 in-place 改。`terraform apply` 会先 destroy 旧的再创建新的，集群可能会断网。

> **教训**：改某些参数 = 销毁重建。`terraform plan` 会标 `forces replacement` 或 `~ in-place` —— 前者危险，后者安全。**永远在 apply 前看 plan**。

## 8. 学完这一章应该会什么

- ✅ 看到 `.tf` 文件能分清块、参数、嵌套块
- ✅ 看到 `var.x` / `resource_type.name.attr` / `${...}` 能立刻知道是引用还是插值
- ✅ 知道 `count` 和 `for_each` 的差别，以及为什么生产代码偏好后者
- ✅ 看到 plan 输出里 `forces replacement` 知道意味着什么

## 9. HCL 不是 Terraform 专属

HCL 也是 Packer / Vault / Consul / Nomad（HashiCorp 全家桶）的配置语言。它们语法基本一致，只是块类型和参数不同。

学好 HCL 一次，受益的不只是 Terraform。

---

> 下一章：[03 — Terraform 工作流](03-terraform-workflow.html) · [章节索引](./)
