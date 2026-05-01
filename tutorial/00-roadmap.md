---
layout: default
title: 00 — 学习路线图
---

# 第 00 章：学习路线图

> 在写第一行代码之前，先看清楚你要学到哪里、为什么这么排序、每一段的产出是什么。

## 你最终要做出来什么

读完整套教程，你应该能从零搭出一套这样的系统：

```
你的笔记本
    │
    │  terraform apply
    ▼
GCP / Hetzner 上自动开出来：
    ├── 虚拟私有网络 (VPC / 私有子网)
    ├── Kubernetes 集群 (GKE Autopilot 或 k3s)
    ├── 对象存储 (GCS / Hetzner Object Storage)
    ├── 容器镜像仓库 (Artifact Registry)
    └── 服务账号 + 权限 (IAM / Workload Identity)
                    │
                    │  kubectl apply -k
                    ▼
        集群里跑起来：
            ├── Deployment    (常驻服务)
            ├── CronJob       (定时任务)
            ├── ConfigMap     (配置)
            ├── Secret        (密钥)
            └── PersistentVolumeClaim (持久化数据)
                    │
                    │  Python SDK
                    ▼
        程序化地：
            ├── 创建/删除工作负载
            ├── 读 Pod 日志
            └── 监控集群状态
```

不是 toy demo。**生产环境的每一层都覆盖**：从 IaC 到容器编排到程序化管理。

## 为什么这样安排章节

```
00 路线图（你在这里）
    │
    ├──→ Terraform 路径 (01-10)
    │        ├── 01-03: 概念基础（IaC / HCL / 工作流）
    │        ├── 04-08: GKE 真实代码逐字解读
    │        └── 09-10: Hetzner 对照（同一思路，换一家云）
    │
    ├──→ Kubernetes 路径 (11-13)
    │        ├── 11: Pod / Deployment（"跑啥"的最小单元）
    │        ├── 12: Job / CronJob / Namespace（一次性 + 定时 + 隔离）
    │        └── 13: ConfigMap / Secret / PVC（配置 / 密钥 / 持久化）
    │
    ├──→ 工具实战路径 (14-16)
    │        ├── 14: kubectl（90% 的日常操作）
    │        ├── 15: Kustomize（多环境复用 manifest）
    │        └── 16: Python SDK（程序化集群管理）
    │
    └──→ 收官 (17-18)
             ├── 17: 端到端 — 从 .tf 写下第一行到数据落 GCS
             └── 18: FAQ — 常见疑问 + 故障排查
```

**核心思路：先 IaC 后 K8s**。原因：

- 没有集群，K8s 概念没有载体。先把基础设施 `terraform apply` 出来，再学集群里的东西更有手感。
- IaC 是云原生的"地基"思维 —— 一切都用代码描述，可重现、可 review、可回滚。这种思维一旦建立，看任何后端工程都不会再迷失。
- Terraform 的概念面比 K8s 小（4 个动词 + state 文件 + provider，就完了）。先学小的再学大的。

## 每段学完应该会什么

### 01-10 Terraform 路径学完

- ✅ 看到任何 `.tf` 文件能逐行读懂
- ✅ 给新项目从零写 `main.tf` / `variables.tf` / `network.tf`
- ✅ 理解 `state` 是 Terraform 的核心，不是 `.tf` 文件本身
- ✅ 会用 `terraform plan` 预读 diff，敢按 `apply`
- ✅ 能在 GKE 和 Hetzner 之间互相对照（"在 GCP 这是 `google_compute_network`，在 Hetzner 是 `hcloud_network`，同一个抽象"）
- ✅ 知道 `enetpulse-k3s`（Hetzner 实际用的工具）和真正的 Terraform 路径的差别在哪

### 11-13 Kubernetes 概念学完

- ✅ 能讲清楚 Pod / Deployment / ReplicaSet 三者的关系
- ✅ 知道什么时候用 Job（一次性）vs CronJob（定时）vs Deployment（常驻）
- ✅ 会读任何一份 K8s YAML manifest（apiVersion / kind / metadata / spec 四段式）
- ✅ 理解为什么 K8s 把"配置"和"密钥"分开（ConfigMap vs Secret）
- ✅ 知道 PVC 是什么、为什么 Pod 重启数据不丢

### 14-16 工具实战学完

- ✅ kubectl 的 5 个高频命令（`get` / `describe` / `apply` / `logs` / `exec`）肌肉记忆
- ✅ 用 Kustomize 维护 dev / staging / prod 三套环境，避免复制粘贴 manifest
- ✅ 能用 Python `kubernetes` 客户端创建 Job、读 Pod 状态、删除资源
- ✅ 看本项目的 `k8s_manager` 时不再心慌

### 17-18 收官

- ✅ 走完一条完整路径：本地写代码 → terraform apply → 镜像推到 AR → kubectl apply → Pod 起来 → 数据落 bucket
- ✅ 知道每一步出错时该看哪里、问什么问题

## 学习节奏建议

每章独立，不需要一次读完。建议节奏：

| 节奏 | 一周读多少 | 适合 |
|---|---|---|
| 慢热型 | 1-2 章 | 完全没碰过云 / 想配合实操 |
| 标准型 | 3-4 章 | 有一点 Linux 基础，想系统学 |
| 冲刺型 | 全部一次扫完 | 已经在做云原生，查漏补缺 |

**实操强烈建议**：每读完一章，去自己的 GCP / Hetzner 账号试一遍。光读不动手，三天就忘。Terraform 部分尤其如此 —— `terraform plan` / `terraform apply` 的反馈回路是教程教不出来的。

## 三条最快路径（如果你不想全读）

### 只关心 Terraform

00 → 01 → 02 → 03 → 04 → 06 → 07 → 18

略过：HCL 细节（02 的部分）、Hetzner（09-10）、K8s（11-16）、e2e（17）

### 只关心 K8s

00 → 11 → 12 → 13 → 14 → 18

前提：你已经有一个集群（来自 GKE 控制台 / minikube / kind / OrbStack），可以跳过 Terraform 部分

### 想看一个真实生产系统怎么部署

00 → 17 → 回看相关章节

直接读端到端走一遍，遇到不懂的概念再倒回去查对应章节

## 几个会反复出现的"心智模型"

整套教程里有几个核心思维方式，提前打个预防针：

### 1. 声明式 ≠ 命令式

你不再写"先做 A，再做 B，最后做 C"。你写"我想要的最终状态是这样"，让工具自己去算怎么变到那个状态。

Terraform 是声明式的，Kubernetes 也是声明式的。这是云原生的根基思维。

### 2. 一切都是资源（Resource / Object）

无论是 GCP 上的 VPC、Hetzner 上的 VM、K8s 里的 Pod，都被抽象成"资源"。每个资源有：

- 类型（type）—— 它是什么
- 名字（name）—— 怎么唯一标识它
- 配置（spec）—— 它该长什么样
- 状态（status）—— 它现在实际是什么样

这个 4 元组在 Terraform 和 Kubernetes 里**完全同构**。学完一个，另一个就只是换了语法。

### 3. State 是真相，不是 .tf 或 .yaml 文件

`.tf` 文件描述"我想要的"。云上实际资源是"真实存在的"。state 文件是 Terraform 用来追踪两者对应关系的桥梁。

K8s 也一样：你 `apply` 的 YAML 是"想要的"，etcd 里存的是"实际状态"。controller 永远在比对、调和。

理解了 state 是真相，你就理解了为什么不能手动改云上资源、为什么不能直接编辑 etcd。

### 4. 凡是手动操作，都是漏洞

只要你"在 Console 里点了一下"，这个改动就出了 IaC 的视野。下次有人 `terraform apply`，可能就会被覆盖；或者有人在新环境里复现，永远复现不出来。

学云原生的本质是：**让自己不要再手动点按钮**。每一次手动操作都是欠下的债，迟早要还。

---

## 准备好了吗

下一步：[第 01 章 — IaC 与 Terraform 基础](01-iac-and-terraform.html)。

如果你还没有 GCP / Hetzner 账号，可以先去注册：

- **GCP**：注册送 $300 + 一年免费 tier，够把这套教程跑完。
- **Hetzner Cloud**：最便宜的 VM €4/月（CX22），跑教程一周大概 €1。

注册不强制，你也可以纯读到底再决定动不动手。
