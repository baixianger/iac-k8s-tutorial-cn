---
layout: default
title: 13 — K8s ConfigMap / Secret / PVC
---

# 第 13 章：K8s — ConfigMap / Secret / PVC

> 上一章：[12 — Job / CronJob / Namespace](12-k8s-job-cronjob.html) · [章节索引](./)

容器是无状态的（理论上）。但应用需要：

- **配置**（API URL、特性开关、缓存大小）
- **密钥**（数据库密码、第三方 API key）
- **持久化数据**（数据库本体、用户上传文件）

K8s 把这三件事**显式拆成三个资源**：ConfigMap、Secret、PersistentVolumeClaim。这一章一次讲完。

## 1. ConfigMap —— 注入配置

### 1.1 一个 ConfigMap 的样子

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: my-app
data:
  API_BASE: "https://api.example.com"
  CACHE_SIZE_MB: "256"
  FEATURE_NEW_UI: "false"
  config.yaml: |
    log_level: info
    timeout: 30s
    retries: 3
```

`data` 下面就是 key-value 对。值是字符串 —— 数字也得加引号（`"256"` 不是 `256`）。

`config.yaml: |` 这种 block scalar 用法可以塞**整个文件内容**进去，挂载时变成一个文件。

### 1.2 怎么进 Pod —— 三种方式

#### 1.2.1 当环境变量

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello
spec:
  containers:
    - name: app
      image: my-app:1.0
      env:
        - name: API_BASE
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: API_BASE
```

容器里 `echo $API_BASE` → `https://api.example.com`。

如果想**一次性把整个 ConfigMap 都灌成环境变量**：

```yaml
      envFrom:
        - configMapRef:
            name: app-config
```

容器里所有 ConfigMap 的 key 都是环境变量。简单暴力。

#### 1.2.2 当文件挂载

```yaml
spec:
  containers:
    - name: app
      image: my-app:1.0
      volumeMounts:
        - name: config-vol
          mountPath: /etc/app
          readOnly: true
  volumes:
    - name: config-vol
      configMap:
        name: app-config
```

容器里：

```bash
ls /etc/app
# API_BASE  CACHE_SIZE_MB  FEATURE_NEW_UI  config.yaml
cat /etc/app/config.yaml
# log_level: info
# timeout: 30s
# ...
```

每个 ConfigMap key 变成一个文件。`config.yaml` 这种带换行的就是个完整 yaml 文件。

#### 1.2.3 命令行参数

```yaml
      args:
        - "--api-base"
        - "$(API_BASE)"        # 注意 $(VAR) 不是 ${VAR}
      env:
        - name: API_BASE
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: API_BASE
```

少用 —— 环境变量或文件更清晰。

### 1.3 ConfigMap 改了 Pod 不会自动 reload

**重要 gotcha**：你 `kubectl apply` 改了 ConfigMap，已经跑着的 Pod 不会自动看到新值。

- **环境变量**：永远不会刷新（环境变量是进程启动时注入的）
- **文件挂载**：~1 分钟后会自动同步（kubelet 周期性 reconcile）

实战做法：改 ConfigMap 后用 `kubectl rollout restart` 强制重建 Pod：

```bash
kubectl edit configmap app-config       # 改值
kubectl rollout restart deployment/api  # 强制 Deployment 重建所有 Pod
```

### 1.4 创建 ConfigMap 的快捷方式

```bash
# 从 literal
kubectl create configmap app-config \
  --from-literal=API_BASE=https://api.example.com \
  --from-literal=CACHE_SIZE_MB=256

# 从文件
kubectl create configmap app-config --from-file=config.yaml

# 从整个目录
kubectl create configmap app-config --from-file=./conf.d/

# 看一眼
kubectl get cm app-config -o yaml
```

## 2. Secret —— 注入密钥

Secret 跟 ConfigMap 几乎一样，**唯一区别**：值在 etcd 里**默认 base64 编码**（不是加密！只是不"裸"）。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-password
  namespace: my-app
type: Opaque
data:
  password: c2VjcmV0MTIz       # base64 of "secret123"
stringData:                     # 写明文，K8s apply 时自动 base64
  api_key: "sk-or-1234567890"
```

> ⚠️ **base64 不是加密**。任何能 `kubectl get secret` 的人都能 `base64 -d` 拿明文。Secret 的安全性来自 **RBAC**（谁能读这个 Secret）和 **etcd at-rest encryption**（要单独打开），不是 base64 本身。

### 2.1 怎么进 Pod —— 跟 ConfigMap 一样

环境变量：

```yaml
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-password
              key: password
```

文件挂载：

```yaml
      volumeMounts:
        - name: secret-vol
          mountPath: /etc/secrets
          readOnly: true
  volumes:
    - name: secret-vol
      secret:
        secretName: db-password
```

文件挂载在生产里**比环境变量好**：环境变量会被 `ps -e` / 子进程意外暴露；挂载文件需要主动 read 才能拿到。

### 2.2 Secret 的几种 type

| `type` | 用途 |
|---|---|
| `Opaque`（默认） | 通用 key-value 密钥 |
| `kubernetes.io/dockerconfigjson` | 拉私有镜像的 docker login 凭证 |
| `kubernetes.io/tls` | TLS 证书 + 私钥（Ingress 用） |
| `kubernetes.io/service-account-token` | K8s 自己生成的 ServiceAccount token |

实际项目 99% 用 `Opaque`。

### 2.3 创建 Secret

```bash
# 从 literal（K8s 自动 base64）
kubectl create secret generic db-password \
  --from-literal=password='secret123' \
  --from-literal=username='admin'

# 从文件
kubectl create secret generic ssh-key --from-file=id_rsa=~/.ssh/id_rsa

# Docker registry
kubectl create secret docker-registry my-registry \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=pass
```

### 2.4 永远不要把 Secret yaml 放进 git

即使 base64，**也不要 commit**。Secret yaml 应该：

- 只在本地 / CI 临时生成
- 用 [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) 等工具加密后再 commit
- 或用外部 secret manager（Vault / GCP Secret Manager / AWS Secrets Manager）

> 项目里见过：`secrets.yaml` 进了 git 仓库，几个月后才发现。这是常见漏洞。**审计你 git 历史里的 yaml 文件**。

## 3. PersistentVolumeClaim —— 持久化存储

Pod 默认是无状态的：删了 Pod，容器里写的数据全丢。**PVC** 让 Pod 拥有**重启不丢**的存储。

### 3.1 三层模型

```
PersistentVolume (PV) —— 实际存储后端（GCE PD / NFS / Ceph / hostPath）
   ↑ 被绑定
PersistentVolumeClaim (PVC) —— "我要 10Gi 的存储"，namespace 级
   ↑ 被引用
Pod —— mount 这个 PVC
```

类比：

- PV = "硬盘"（基础设施层）
- PVC = "我的硬盘需求单"（应用层）
- Pod = 用户

### 3.2 PVC 长啥样

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
  namespace: my-app
spec:
  accessModes:
    - ReadWriteOnce          # RWO：单节点读写（最常用）
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard  # 由 StorageClass 决定后端
```

### 3.3 accessModes —— 关键

| 模式 | 缩写 | 含义 |
|---|---|---|
| `ReadWriteOnce` | RWO | 一个节点（多个 Pod 在同节点 OK） |
| `ReadOnlyMany` | ROX | 多节点只读 |
| `ReadWriteMany` | RWX | 多节点读写 |
| `ReadWriteOncePod` | RWOP | 单 Pod（K8s 1.27+ 才稳定） |

**默认 RWO 是单节点**。多个 Pod 跨节点共写一份数据，必须 RWX —— 但**不是所有存储后端都支持 RWX**：

| 后端 | RWO | RWX |
|---|---|---|
| GCE Persistent Disk | ✅ | ❌（要 [GCSFuse](https://cloud.google.com/storage/docs/gcs-fuse) 或 Filestore） |
| AWS EBS | ✅ | ❌（要 EFS） |
| NFS | ✅ | ✅ |
| Ceph | ✅ | ✅ |

项目里跨 worker 共享 `/data/websites` 目录用的就是 NFS RWX。

### 3.4 StorageClass —— "动态"分配 PV

写 PVC 不写 PV 的现代做法：依赖 **StorageClass** 动态创建 PV。

```bash
kubectl get storageclass
# NAME                 PROVISIONER             AGE
# standard (default)   pd.csi.storage.gke.io   30d
# fast-ssd             pd.csi.storage.gke.io   30d
# nfs-manual           ...                     30d
```

PVC 里写 `storageClassName: standard` —— K8s 就用 GCE PD CSI driver 自动开一块磁盘绑给你。

### 3.5 Pod 挂 PVC

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: app
      image: my-app:1.0
      volumeMounts:
        - name: data
          mountPath: /var/data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: app-data         # 引用上面那个 PVC
```

容器里 `/var/data` 就是 10Gi 持久存储。Pod 重启 / Deployment 滚动升级 —— 数据还在。

### 3.6 几个生产规矩

- **删 Pod 不会删 PVC**。删 PVC 才会真销毁数据。
- **删 Namespace 会连带删 PVC**。慎用 `kubectl delete ns`。
- **PV reclaim policy** 决定 PVC 删了后 PV 怎么办：`Delete`（销毁底层存储） / `Retain`（保留）。生产推荐 `Retain`，避免误删数据。
- **PVC 一旦创建后，size 一般只能扩不能缩**（取决于 StorageClass）。

## 4. ConfigMap vs Secret vs PVC 一表对照

| 资源 | 大小限制 | 适合 | 不适合 |
|---|---|---|---|
| ConfigMap | 1 MiB | 配置文件、特性开关 | 大文件、敏感数据 |
| Secret | 1 MiB | 密码、API key、TLS 证书 | 大文件、配置 |
| PVC | 视后端 | 持久化数据 | 配置 |

### 4.1 一个真实组合例子

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: my-app
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: api
          image: my-app:1.0
          # 配置：从 ConfigMap
          envFrom:
            - configMapRef:
                name: app-config
          # 密钥：从 Secret
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-password
                  key: password
          # 持久化：从 PVC
          volumeMounts:
            - name: data
              mountPath: /var/data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: app-data
```

三件事各司其职：配置走环境变量、密钥走密码、持久化走挂载。

## 5. 学完这一章应该会什么

- ✅ 知道 ConfigMap / Secret 几乎一样，差别只是 base64 + 命名习惯
- ✅ 理解 base64 ≠ 加密，Secret 安全靠的是 RBAC + etcd encryption
- ✅ 能选环境变量 vs 文件挂载（生产推荐文件挂载）
- ✅ 知道 ConfigMap 改了要 `kubectl rollout restart` 才生效
- ✅ 理解 PV / PVC / StorageClass 三层模型，类比"硬盘 / 需求单 / 用户"
- ✅ 知道 RWO vs RWX，知道哪些后端支持 RWX
- ✅ 知道删 Namespace 会连带删 PVC（避免误删数据）

K8s 概念部分讲完。下一章开始进入工具实战 —— `kubectl` 命令。

---

> 下一章：[14 — kubectl 命令实战](14-kubectl.html) · [章节索引](./)
