---
layout: default
title: 端到端工程教学
---

# Enetpulse 端到端工程教学

> 中文 / 零基础 / 章节式 / 循序渐进
>
> 从"我想部署一个云原生项目"开始，到"它在生产真的跑起来"为止——途中你会学完：**Terraform、Kubernetes 概念、kubectl、Python SDK、双云部署（GKE + Hetzner）**。

## 适合谁

- 看到 `.tf` 文件不知道怎么读
- 听过 K8s 但分不清 Pod / Deployment / Job 区别
- 想看懂本项目的 `cloud/deployments/stacks/gke/` 和 `cloud/deployments/stacks/hetzner/`
- 需要给客户解释技术选型时心里有底

## 章节清单

> ✅ 全部 20 章已完成 — 每章独立，可单独读

| 章 | 标题 | 主题 | 状态 |
|---|---|---|---|
| 00 | [学习路线图](00-roadmap.html) | 整体目标 + 章节地图 | ✅ |
| 01 | [IaC 与 Terraform 基础](01-iac-and-terraform.html) | 为什么不用手点 Console | ✅ |
| 02 | [HCL 语法](02-hcl-syntax.html) | 块、参数、表达式 | ✅ |
| 03 | [Terraform 工作流](03-terraform-workflow.html) | init/plan/apply/state | ✅ |
| 04 | [GKE — Provider 与 Variables](04-gke-provider-variables.html) | `main.tf`, `variables.tf` | ✅ |
| 05 | [GKE — 网络](05-gke-network.html) | `network.tf` (VPC, subnet, NAT) | ✅ |
| 06 | [GKE — Autopilot 集群](06-gke-cluster.html) | `cluster.tf` | ✅ |
| 07 | [GKE — 存储与 IAM](07-gke-storage-iam.html) | `storage.tf`, `iam.tf` | ✅ |
| 07b | [GKE — 镜像仓库（Artifact Registry）](07b-gke-artifact-registry.html) | `registry.tf` + Docker 工作流 | ✅ |
| 08 | [GKE — Outputs](08-gke-outputs.html) | `outputs.tf` | ✅ |
| 09 | [Hetzner 假设 Terraform](09-hetzner-hypothetical.html) | 同样 IaaS，换 provider | ✅ |
| 10 | [Hetzner 真实路径](10-hetzner-real-path.html) | `hetzner-k3s` + `cluster.yaml` | ✅ |
| 11 | [K8s 概念 — Pod 与 Deployment](11-k8s-pod-deployment.html) | 集群里"跑啥东西"的最小单元 | ✅ |
| 12 | [K8s 概念 — Job / CronJob / Namespace](12-k8s-job-cronjob.html) | 一次性任务、定时任务 | ✅ |
| 13 | [K8s 状态 — ConfigMap / Secret / PVC](13-k8s-configmap-secret-pvc.html) | 配置、密钥、持久化 | ✅ |
| 14 | [kubectl 命令实战](14-kubectl.html) | get / describe / apply / logs / exec | ✅ |
| 15 | [Kustomize 编排 manifest](15-kustomize.html) | 多环境复用 | ✅ |
| 16 | [Kubernetes Python SDK](16-k8s-python-sdk.html) | 程序化集群管理 | ✅ |
| 17 | [端到端走一遍](17-end-to-end.html) | 从 .tf 写下第一行 → 数据落 GCS 全过程 | ✅ |
| 18 | [FAQ](18-faq.html) | 常见疑问 + 故障排查 | ✅ |

## 学习顺序建议

```
        ┌─→ 01-08 GKE Terraform 路径    ─┐
00      │                                 ├─→ 11-13 K8s 概念  ─→ 14-16 工具实战 ─→ 17 e2e ─→ 18 FAQ
路线图  └─→ 09-10 Hetzner 路径           ─┘
```

**最快路径**（只要会改 GKE）：00 → 01 → 03 → 04 → 06 → 07 → 14 → 18（~3 小时）

**完整路径**（双云 + K8s + Python SDK 全懂）：00 → 18 全顺序（~10-12 小时）

## 怎么用这个教程

每章标题点进去就是一篇独立 markdown，跟你的浏览器一个标签页一篇就好。每章完了再进下一章——不需要一口气读完，**循序渐进**。

读到哪个概念有疑问，回到 [FAQ](18-faq.html) 看看。

## 单文件版（如果你喜欢一篇长文滑到底）

[Terraform 单文件教程](../terraform-tutorial-single-file.html) — 比章节版更老、更紧凑、纯 Terraform（没 K8s/kubectl/Python SDK 部分）。**不推荐主修**，但作为快速参考有用。

---

> 教程作者：本项目工程团队 | 反馈：在 GitHub Issue 留言
