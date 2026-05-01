---
layout: default
title: 01 — IaC 与 Terraform 基础
---

# 第 01 章：IaC 与 Terraform 基础

> 上一章：[00 — 学习路线图](00-roadmap.html) · [章节索引](./)

为什么不能继续点 Console。这章只讲概念，不写代码 —— 但讲完之后，后面 1700 行的 `.tf` 你就有"为什么这么写"的回答。

## 1. 你之前怎么开云资源

没用 IaC 之前，你大概率是这样开云资源的：

1. 浏览器打开 GCP Console / Hetzner 控制台
2. 找到 "Create Cluster" / "Add Server" 按钮
3. 点点点填表
4. 等几分钟创建完成
5. 得到一个集群 / 一台 VM

**问题**：

- **没记录**。哪个按钮你点了？哪个字段填了什么？三个月后回头看不知道
- **不可重复**。同事想搞一个一样的，要全程跟着你点
- **容易出错**。手工填的 IP / 配额 / 标签很容易写错
- **没法 review**。改了什么没人能 PR review
- **没法回滚**。点错了想撤销，要再点一遍反操作 —— 但你不一定记得"反操作"该怎么点

每一个手动操作都是欠下的债，迟早要还。

## 2. IaC（Infrastructure as Code）的概念

**把"基础设施"变成"代码"**。你不再点按钮 —— 你写一个文本文件描述：

> "我想要一个集群、3 个节点、1 个网络、2 个 bucket"

然后跑一条命令，让某个工具把它创建出来。

这个文本文件可以：

- **放进 git** — 所有改动有 commit history，谁什么时候改了什么一目了然
- **PR review** — team member 在合并前可以看你要改什么
- **直接复制给同事** — 他在他自己账号跑出来的环境跟你的一样
- **升级 / 回滚跟代码一样** — `git revert` 然后重 apply

这就把"运维"从"我点了某些按钮但记不清"变成"我有版本控制的源代码"。

## 3. Terraform 在 IaC 里是什么角色

Terraform 是众多 IaC 工具的一种。它的工作方式：

```
你写 .tf 文件 (描述你想要什么)
        ↓
terraform plan
   (Terraform 跟云服务商对话，对比当前实际 vs 你想要的，列出 diff)
        ↓
terraform apply
   (Terraform 执行 diff：创建缺的、修改不一致的、删除多余的)
        ↓
云上资源 = 你 .tf 文件描述的样子
```

**最关键的一点**：Terraform 是 **声明式** 的，不是 **命令式** 的。

| 风格 | 你写什么 | 工具做什么 |
|---|---|---|
| 命令式 | "开一台 VM、装 nginx、配防火墙、改 DNS" | 一步步执行你写的命令 |
| **声明式** | "我想要一个开着 nginx 的 VM、防火墙开 80 端口" | 自己算出"为了达到这个状态，需要做哪些操作" |

声明式的好处是 **幂等**：你可以反复 apply 同一份 `.tf`，每次 Terraform 都对比"实际状态 vs 你描述的状态"，只做必要的改动。已经一致的就跳过。

> 这是云原生第一个核心思维。后面你会看到 **Kubernetes 也是声明式的** —— 你写 `Deployment` 描述"我想要 3 个 nginx 副本"，K8s 自己保证集群里始终有 3 个。同样的思维，换了一层抽象。

## 4. IaC 工具不止 Terraform 一家

| 工具 | 谁的 | 哪种语言 | 一句话 |
|---|---|---|---|
| Terraform | HashiCorp | HCL（自创 DSL） | 跨云、最普及，本教程主修 |
| OpenTofu | Linux Foundation | HCL（同 Terraform） | Terraform 1.5 fork，2023 起独立维护，license 改 MPL，**大部分场景跟 Terraform 完全互换** |
| Pulumi | Pulumi 公司 | TypeScript / Python / Go / C# | 用真实编程语言写 IaC，喜欢类型系统的可以试 |
| AWS CloudFormation | AWS | YAML / JSON | 只能给 AWS 用，跨云不行 |
| Google Cloud Deployment Manager | Google | YAML | 只能给 GCP 用，已经被 Google 自己冷处理 |
| Ansible | Red Hat | YAML | 偏向"配置管理"（在已有机器上装东西），不是建机器；可以跟 Terraform 配合 |

**为什么本教程选 Terraform**：

1. 跨云通用 —— GKE 用它，Hetzner 也用它（虽然 Hetzner 真实路径用了 hetzner-k3s，但概念同构）
2. 生态最大 —— 几乎每个云 / SaaS 都有 provider
3. 写法简单 —— HCL 比 YAML 好读，比 TypeScript 学习曲线短

学完 Terraform 你也基本看得懂 OpenTofu / Pulumi —— 思维一致，只是语法换了。

## 5. 两条部署路径

本教程会贯穿两个云的对比 —— 同一套思维方式，落到不同的 provider 上：

| 集群目标 | 实际用什么工具 | 本教程怎么对待 |
|---|---|---|
| **GKE Autopilot**（Google Cloud） | Terraform（已有完整 stack） | 真实情况，逐字解读 |
| **Hetzner k3s**（Hetzner Cloud） | hetzner-k3s + cluster.yaml | 第 09 章假设也用 Terraform 怎么写；第 10 章再讲真实路径 |

**为什么真实 Hetzner 不用 Terraform**：因为 `hetzner-k3s` 这个工具一站式做了"建 VM + 安装 k3s + 配集群"三件事，比 Terraform-only 路径省事。

**但为了教学**，第 09 章假设 Hetzner 也用 Terraform —— 这样可以横向对比 `google` 和 `hcloud` 两个 provider，把 Terraform 学透。然后第 10 章告诉你真实路径长啥样。

## 6. 学完这一章应该会什么

- ✅ 能给非技术人员解释清楚"什么是 IaC、为什么不点 Console"
- ✅ 知道声明式 vs 命令式的差别
- ✅ 知道 Terraform 不是唯一选择，但是当前最普及的选择
- ✅ 准备好下一章看真实代码

## 7. 准备工作

下一章开始就要看真实 `.tf` 文件了。如果你想边读边动手：

```bash
# 1. 装 Terraform CLI（macOS）
brew install hashicorp/tap/terraform
terraform version    # 应该 ≥ 1.6

# 2. 装 gcloud CLI（macOS）
brew install --cask google-cloud-sdk
gcloud auth login
gcloud auth application-default login    # ⚠️ 这一步是 Terraform 用的

# 3. 准备一个 GCP 项目（注册送 $300 + 永久免费 tier）
gcloud projects create my-cloud-project    # 改成你想要的项目 ID
gcloud config set project my-cloud-project
```

不强制 —— 你也可以先把后面几章读完再回来动手。

---

> 下一章：[02 — HCL 语法](02-hcl-syntax.html) · [章节索引](./)
