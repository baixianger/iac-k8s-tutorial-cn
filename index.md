---
layout: default
title: 端到端云原生工程教学
---

# 端到端云原生工程教学

> 中文 / 零基础 / 章节式 / 循序渐进
>
> 从「我想部署一个云原生项目」开始，到「它在生产真的跑起来」为止——途中你会学完：**Terraform、Kubernetes 概念、kubectl、Python SDK、双云部署（GKE + Hetzner）**。

## 适合谁

- 看到 `.tf` 文件不知道怎么读
- 听过 K8s 但分不清 Pod / Deployment / Job 区别
- 想搭一套真实可用的云原生项目，而不是 `kubectl run nginx` 教程
- 需要给客户解释技术选型时心里有底

## 两个版本

| 版本 | 范围 | 适合 |
|---|---|---|
| **[章节式教程](tutorial/)** ← 推荐 | 19 章：Terraform + K8s + kubectl + Python SDK + 双云端到端 | 系统学习、循序渐进 |
| [Terraform 单文件版](terraform-tutorial-single-file.html) | 14 章：纯 Terraform（GKE + Hetzner） | 快速参考、只关心 Terraform |

## 课程范围

```
        ┌─→ Terraform — IaC 入门 + GKE/Hetzner 真实路径
入口    │
       ─┼─→ Kubernetes — Pod / Deployment / Job / CronJob / ConfigMap / Secret / PVC
        │
        └─→ 工具实战 — kubectl / Kustomize / Python SDK
                              ↓
                     端到端走一遍 + FAQ
```

**学完之后你应该能：**

1. 看懂任何 `.tf` 文件，给新项目从零写 Terraform
2. 看懂任何 K8s manifest，能 `kubectl apply` 一套自己的工作负载
3. 用 Python `kubernetes` SDK 程序化管理集群
4. 把同一套应用在 GKE 和 Hetzner 之间互相搬迁

## 怎么用这个教程

每章独立，浏览器一个标签页一篇就好。每章完了再进下一章——**不需要一口气读完**。

读到哪个概念有疑问，去 FAQ 看看，或者回章节地图找上下文。

---

> 教程基于一个真实的中规模生产项目（体育数据爬虫系统）的部署经验整理。代码片段和 Terraform 配置都来自实战，不是 toy demo。
