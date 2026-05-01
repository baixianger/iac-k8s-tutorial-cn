---
layout: default
title: 15 — Kustomize 编排 manifest
---

# 第 15 章：Kustomize —— 多环境复用 manifest

> 上一章：[14 — kubectl 命令实战](14-kubectl.html) · [章节索引](./)

dev / staging / prod 三套环境，95% 的 manifest 是一样的，只有少数字段不同（image tag、replicas、namespace、env vars）。怎么办？

最差做法：**复制粘贴三份**。改了忘改一份就出事。

更好做法：**Kustomize**。一份 base + 三份 overlay。

> Kustomize 是 K8s 内置的（`kubectl apply -k`），不需要额外安装。Helm 是另一个流行选择，但学习曲线更陡 —— 推荐先掌握 Kustomize。

## 1. Kustomize 的核心思想

```
base/                          ← 公共部分
├── kustomization.yaml
├── deployment.yaml
└── service.yaml

overlays/
├── dev/
│   ├── kustomization.yaml     ← 引用 base，加 dev 特定的 patch
│   └── patches/
├── staging/
│   ├── kustomization.yaml
│   └── patches/
└── prod/
    ├── kustomization.yaml
    └── patches/
```

每个 overlay 的 `kustomization.yaml` 说"我基于 base，再加这些改动"。`kubectl apply -k overlays/prod` 会现场拼出最终 yaml 然后 apply。

## 2. base —— 公共部分

`base/deployment.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: my-app:latest
          ports:
            - containerPort: 80
          env:
            - name: ENV
              value: "default"
```

`base/service.yaml`：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 80
```

`base/kustomization.yaml`：

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

试试：

```bash
kubectl kustomize base/                  # 渲染（不 apply）
kubectl apply -k base/                   # 渲染 + apply
```

## 3. overlay —— 环境特定改动

`overlays/prod/kustomization.yaml`：

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# 引用 base
resources:
  - ../../base

# 给 base 里所有资源加 namespace
namespace: my-app-prod

# 给 base 里所有资源加 label
commonLabels:
  env: prod

# 给所有名字加前缀
namePrefix: prod-

# 修改 image（不需要写完整 patch）
images:
  - name: my-app
    newTag: v1.4.2

# 改 replicas
replicas:
  - name: api
    count: 5

# 任意 patch（最灵活）
patches:
  - target:
      kind: Deployment
      name: api
    patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/env/0/value
        value: "production"
      - op: add
        path: /spec/template/spec/containers/0/resources
        value:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "2000m"
            memory: "2Gi"
```

```bash
kubectl kustomize overlays/prod/         # 看渲染结果
kubectl apply -k overlays/prod/          # apply
```

最终 apply 出去的 Deployment 长这样（base + 所有 overlay 改动合并后）：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prod-api                         # ← namePrefix
  namespace: my-app-prod                  # ← namespace
  labels:
    env: prod                             # ← commonLabels
spec:
  replicas: 5                             # ← replicas patch
  selector:
    matchLabels:
      app: api
      env: prod                            # ← commonLabels 也加到 selector
  template:
    metadata:
      labels:
        app: api
        env: prod
    spec:
      containers:
        - name: api
          image: my-app:v1.4.2             # ← images patch
          env:
            - name: ENV
              value: "production"          # ← strategic patch
          resources:                        # ← strategic patch
            requests: { cpu: "500m", memory: "512Mi" }
            limits:   { cpu: "2000m", memory: "2Gi" }
          ports:
            - containerPort: 80
```

## 4. Kustomize 能做的所有"改动"

### 4.1 顶层字段 —— 最常用

```yaml
namespace: my-app-prod         # 所有资源加 namespace
namePrefix: prod-              # 所有资源名字加前缀
nameSuffix: -v2                # 后缀
commonLabels:
  env: prod
  team: data
commonAnnotations:
  owner: "data-team@company.com"
```

### 4.2 image —— 改镜像（最高频）

```yaml
images:
  - name: my-app                  # 匹配 base 里的 image: my-app:*
    newName: registry.example.com/my-app   # 换成新的 image 名
    newTag: v1.4.2

  # 或者只换 tag
  - name: nginx
    newTag: 1.28
```

每次发布只要改这里一个 `newTag`，整套 overlay 都跟着升级。

### 4.3 replicas

```yaml
replicas:
  - name: api
    count: 5
  - name: worker
    count: 3
```

### 4.4 patches —— 最强但啰嗦

两种写法：

**Strategic merge**（推荐，跟 K8s 自身的 patch 语义一致）：

```yaml
patches:
  - target:
      kind: Deployment
      name: api
    patch: |-
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: api
      spec:
        template:
          spec:
            containers:
              - name: api
                resources:
                  requests:
                    memory: "1Gi"
```

**JSON patch**（更精确）：

```yaml
patches:
  - target:
      kind: Deployment
      name: api
    patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/env/0/value
        value: "production"
      - op: add
        path: /spec/template/spec/containers/0/env/-
        value:
          name: NEW_FLAG
          value: "true"
```

`op` 支持 `replace` / `add` / `remove` / `move` / `copy`。

### 4.5 configMapGenerator / secretGenerator

直接从文件 / literal 生成 ConfigMap、Secret：

```yaml
configMapGenerator:
  - name: app-config
    literals:
      - LOG_LEVEL=info
      - CACHE_SIZE=256
    files:
      - config.yaml=./conf/app.yaml
```

生成的 ConfigMap 名字会自动加 hash 后缀（如 `app-config-7c5f9d2`），变化会生成新 hash —— 自动触发 Pod 滚动重建。**比手写 ConfigMap 后还要 `kubectl rollout restart` 优雅**。

如果不想自动 hash 后缀（比如你想用固定名字让其他工具引用）：

```yaml
configMapGenerator:
  - name: app-config
    literals:
      - LOG_LEVEL=info
    options:
      disableNameSuffixHash: true
```

## 5. 复杂一点：嵌套 overlay

实际项目可能这样：

```
base/                               ← 完全通用
overlays/
├── env-base/                       ← 所有 env 的公共改动
│   ├── kustomization.yaml          ← resources: [../../base]
│   └── patches/
├── dev/                            ← 基于 env-base
│   ├── kustomization.yaml          ← resources: [../env-base]
│   └── patches/
├── staging/
│   ├── kustomization.yaml          ← resources: [../env-base]
│   └── patches/
└── prod/
    ├── kustomization.yaml          ← resources: [../env-base]
    └── patches/
```

`env-base` 抽出"所有非 dev 共有"的部分（比如 resources 限制、监控注解），dev / staging / prod 再各自微调。

## 6. 项目里 Kustomize 的真实用法

```
cloud/deployments/k8s/
├── base/
│   ├── kustomization.yaml
│   ├── job-creator-deployment.yaml
│   ├── k8s-manager-deployment.yaml
│   ├── postgres-statefulset.yaml
│   ├── registry-deployment.yaml
│   └── ...
├── hetzner/
│   ├── kustomization.yaml         ← resources: [../base], 加 NodePort + NetworkPolicy
│   └── patches/
└── gke/
    ├── kustomization.yaml         ← resources: [../base], 加 GCSFuse mount + Workload Identity
    └── patches/
```

部署 Hetzner：

```bash
kubectl apply -k cloud/deployments/k8s/hetzner/
```

部署 GKE：

```bash
kubectl apply -k cloud/deployments/k8s/gke/
```

**同一份 base，两个云的 overlay 各自适配**。改 image tag 只在 overlay 里改一行。

## 7. 编辑 + 验证 + apply

```bash
# 编辑 image tag（不打开 yaml 直接命令行）
cd overlays/prod
kustomize edit set image my-app=my-app:v1.4.3

# 看渲染（不 apply）
kubectl kustomize .

# diff
kubectl diff -k .

# apply
kubectl apply -k .
```

`kustomize edit` 可以批量改：

```bash
kustomize edit set image my-app=*:v1.4.3
kustomize edit set replicas api=10
kustomize edit set namespace my-app-staging
```

## 8. Kustomize 不擅长的事

- **复杂条件逻辑**（"如果 env=prod 加 X，否则 Y"）—— Kustomize 是纯结构化合并，不支持条件。这种场景用 Helm。
- **跨集群组合**（多个云一起部署）—— 用 ArgoCD ApplicationSet 或 Flux。
- **加密 Secret** —— 用 Sealed Secrets 或外部 secret manager。

**简单场景 Kustomize 是甜点，复杂场景另选工具**。

## 9. 学完这一章应该会什么

- ✅ 理解 base + overlay 的两层模型
- ✅ 会用 `namespace` / `namePrefix` / `commonLabels` / `images` / `replicas` 这 5 个高频字段
- ✅ 看 `patches` 的 strategic merge 和 JSON patch 两种写法
- ✅ 知道 `configMapGenerator` 自动 hash 后缀的好处
- ✅ 用 `kubectl apply -k` / `kubectl kustomize` / `kubectl diff -k` 操作 Kustomize
- ✅ 知道 Kustomize vs Helm 各自适合的场景

下一章看怎么用 Python 程序化操作集群（不用 kubectl）。

---

> 下一章：[16 — Kubernetes Python SDK](16-k8s-python-sdk.html) · [章节索引](./)
