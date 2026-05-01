---
layout: default
title: 08 — GKE Outputs
---

# 第 08 章：GKE — Outputs（outputs.tf）

> 上一章：[07 — 存储与 IAM](07-gke-storage-iam.html) · [章节索引](./)

GKE 的最后一个文件。`outputs.tf` 比前面任何一个都简单 —— 它只声明 "apply 完之后要把哪些值暴露出去"。

## 1. outputs.tf

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

## 2. output 是干嘛的

`output` 块声明 Terraform apply 完之后**对外暴露什么值**。三个用途：

1. **打印给操作员看** —— apply 成功后终端打印
2. **其他工具读** —— 如 `use-gke.sh` 脚本调 `terraform output -raw cluster_name` 拿值
3. **跨 module 引用** —— 如果将来用 module 化，外部 module 通过 output 读子 module 的值

## 3. `value = ...` 的写法

```hcl
output "cluster_name" {
  value = google_container_cluster.autopilot.name
}
```

可以引用任何 resource 的属性。`google_container_cluster.autopilot.name` 就是从 `cluster.tf` 那个 cluster 资源拿 `name` 属性。

```hcl
output "ar_repo_url" {
  value = "${var.region}-docker.pkg.dev/${var.project_id}/${var.ar_repo_id}"
  # 渲染成: europe-west3-docker.pkg.dev/my-cloud-project/my-app
}
```

可以拼字符串 —— AR repo 完整 URL 不在任何资源属性里直接给（GCP API 不返回这个 URL），但用 region + project + repo 能拼出来。

## 4. `sensitive = true`

```hcl
output "cluster_endpoint" {
  value     = google_container_cluster.autopilot.endpoint
  sensitive = true
}
```

`cluster_endpoint` 是 API server 地址，敏感。Terraform 会在 plan/apply 输出里把这个值打码成 `<sensitive>`，避免漏到日志：

```
$ terraform apply
...
Outputs:

cluster_endpoint = <sensitive>
cluster_name = "my-cluster"
```

要拿真值得用 `terraform output -raw cluster_endpoint`：

```bash
$ terraform output -raw cluster_endpoint
35.198.X.Y
```

`-raw` 表示"不加引号、不打码、原值输出"。适合管道给其他命令用。

## 5. 项目里怎么用 outputs

读单个值：

```bash
terraform output -raw cluster_name           # → "my-cluster"
terraform output -raw ar_repo_url            # → "europe-west3-docker.pkg.dev/my-cloud-project/my-app"
terraform output -raw nat_egress_ip          # → "35.198.X.Y"
```

读全部值（JSON）：

```bash
terraform output -json
```

输出像这样：

```json
{
  "ar_repo_url": {
    "sensitive": false,
    "type": "string",
    "value": "europe-west3-docker.pkg.dev/my-cloud-project/my-app"
  },
  "cluster_endpoint": {
    "sensitive": true,
    "type": "string",
    "value": "35.198.X.Y"
  },
  "cluster_name": {
    "sensitive": false,
    "type": "string",
    "value": "my-cluster"
  },
  ...
}
```

可以用 `jq` 解：

```bash
terraform output -json | jq -r '.cluster_name.value'
# → "my-cluster"
```

### 5.1 用 envsubst 自动生成 `.env` 文件

实际项目里常见的模式：

```bash
# 模板文件 .env.template
GKE_CLUSTER=${cluster_name}
GKE_REGION=${region}
AR_REPO=${ar_repo_url}
NAT_IP=${nat_egress_ip}

# 渲染脚本 use-gke.sh
export $(terraform -chdir=cloud/deployments/stacks/gke/infra output -json | \
         jq -r 'to_entries[] | "\(.key)=\(.value.value)"')
envsubst < .env.template > .env
```

`source use-gke.sh` 后，`.env` 就有所有 Terraform output。Python / Node 应用直接读环境变量。

## 6. 几个注意点

### 6.1 output 改了不会触发资源变更

`output` 块只是输出，不是资源。改了 output 的 `value` 表达式重 apply 不会创建/修改任何云资源 —— 只是新一次 apply 后输出值变了。

### 6.2 output 也支持 `description`

虽然项目省略了，但加 `description` 是好习惯：

```hcl
output "cluster_name" {
  description = "GKE cluster name. Used by use-gke.sh to gen kubeconfig."
  value       = google_container_cluster.autopilot.name
}
```

`terraform output` 不会打印 description，但读 `.tf` 的人会看到。

### 6.3 output 不能引用 `var.x` 之外的 input

只能引用 `var.x`、`resource.x.y`、`module.x.y`、`data.x.y`、`local.x` 等 —— 不能引用环境变量或外部命令输出。如果要的话，先用 `data "external"` 或 `locals` 块导入。

## 7. 这个文件你能改什么

| 想改的 | 改哪里 |
|---|---|
| 加一个新 output | 加一个 `output` 块 |
| 把某个 output 标敏感 | 加 `sensitive = true` |
| 拼一个新的 derived URL | 用字符串插值组合 var.x + resource.x.y |
| 删一个 output | 直接删块。下次 apply 该 output 不再显示，**不影响资源** |

## 8. 学完这一章应该会什么

- ✅ 知道 output 的三个用途（打印 / 其他工具读 / module 引用）
- ✅ 看 `terraform output -raw <name>` 知道在干啥
- ✅ 看 `sensitive = true` 知道为什么默认输出会打码
- ✅ 看到 `terraform output -json | jq` 这种 pipe 不再陌生

GKE Terraform stack 至此读完。下一章看 Hetzner —— **同样思维，换一家云的 provider**，能让你看清"Terraform 的抽象"和"具体云的差别"分别在哪。

---

> 下一章：[09 — Hetzner 假设 Terraform](09-hetzner-hypothetical.html) · [章节索引](./)
