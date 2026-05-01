# 端到端云原生工程教学

中文 / 零基础 / 章节式的云原生工程教学。涵盖：

- **Terraform** — IaC 概念、HCL 语法、GKE 和 Hetzner 的真实部署路径
- **Kubernetes** — Pod / Deployment / Job / CronJob / ConfigMap / Secret / PVC
- **工具实战** — kubectl、Kustomize、Kubernetes Python SDK
- **端到端** — 从写第一行 `.tf` 到生产数据落地

教程基于一个中规模生产项目（体育数据爬虫系统，部署在 GKE Autopilot + Hetzner 双云）的真实运维经验整理。

## 在线阅读

GitHub Pages 站点：<https://OWNER.github.io/REPO/>（部署后填）

## 本地预览

```bash
bundle install
bundle exec jekyll serve
# 浏览器打开 http://localhost:4000
```

如果没装 Jekyll：直接读 `.md` 文件即可，GitHub 上也能渲染。

## 文件结构

```
.
├── index.md                              # 站点首页
├── tutorial/
│   ├── index.md                          # 章节地图
│   ├── 00-roadmap.md                     # 学习路线图
│   ├── 01-iac-and-terraform.md           # 待写
│   ├── ...                               # 18 章共
│   └── 18-faq.md                         # FAQ
├── terraform-tutorial-single-file.md     # 纯 Terraform 单文件版（老）
├── _config.yml                           # Jekyll 配置
└── README.md                             # 本文件
```

## 反馈

问题 / 建议：在本仓库提 Issue。
