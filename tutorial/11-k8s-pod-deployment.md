---
layout: default
title: 11 — K8s Pod 与 Deployment
---

# 第 11 章：K8s — Pod 与 Deployment

> 上一章：[10 — Hetzner 真实路径](10-hetzner-real-path.html) · [章节索引](./)

集群已经建好（GKE 或 Hetzner k3s 任选）。现在学集群里"跑啥东西"的最小单元。

这一章建立 K8s 的核心心智模型 —— **Pod 不是孤立运行的，永远被某种控制器管着**。

## 1. Pod —— K8s 的最小调度单元

### 1.1 什么是 Pod

**Pod = 1 个或多个容器的封装**，共享：

- 网络（同一个 IP，互相 `localhost:port` 直连）
- 存储卷（mount 同一份 volume）
- 生命周期（一起启动、一起终止）

**最常见的 Pod 是单容器**：一个 Pod = 一个 nginx 容器。多容器场景叫 "sidecar pattern"（一个主容器 + 一个辅助容器，比如 envoy 代理、日志收集器）。

### 1.2 一个最简单的 Pod manifest

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello
spec:
  containers:
    - name: hello
      image: nginx:1.27
      ports:
        - containerPort: 80
```

四段式（**所有 K8s 资源**都是这四段）：

| 段 | 含义 | 例子 |
|---|---|---|
| `apiVersion` | API 版本 | `v1` / `apps/v1` / `batch/v1` |
| `kind` | 资源类型 | `Pod` / `Deployment` / `Service` |
| `metadata` | 元信息 | `name`, `namespace`, `labels` |
| `spec` | 期望状态 | 你要什么 |

> **声明式回归**：你写 spec，K8s 自己想办法让实际状态跟 spec 一致。同一个范式，跟 Terraform 完全一致。

### 1.3 Apply 一下试试

```bash
kubectl apply -f pod.yaml         # 创建（或更新）
kubectl get pods                  # 看一眼
# NAME    READY   STATUS    RESTARTS   AGE
# hello   1/1     Running   0          5s

kubectl describe pod hello        # 看详细
kubectl logs hello                # 看日志
kubectl exec -it hello -- bash    # 进容器
kubectl delete pod hello          # 删了
```

## 2. 为什么生产不直接用 Pod

**Pod 是脆弱的**。它会因为以下任一原因消失：

- 节点宕机
- 资源不足被驱逐
- 你 `kubectl delete pod` 了
- 容器进程 crash 而 restartPolicy=Never

**Pod 没了就没了** —— K8s 不会自动帮你建一个新的。它只会在原 Pod 里 restart 容器（按 `restartPolicy`），但如果整个 Pod 被删，就没人管了。

生产环境希望"我想要 3 个 nginx 一直跑着，宕一个自动起一个"。这时就需要 **控制器**。

## 3. ReplicaSet —— 维持 N 份 Pod

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: hello-rs
spec:
  replicas: 3                # 我要 3 份
  selector:
    matchLabels:
      app: hello
  template:                  # Pod 模板
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - name: hello
          image: nginx:1.27
```

- `replicas: 3` —— 期望状态 = 3 个 Pod
- `selector.matchLabels` —— 用 label 找"哪些 Pod 归我管"
- `template` —— Pod 的模板（少了 `apiVersion` 和 `kind`，因为是嵌套的）

**ReplicaSet controller 永远在做**：

```
loop forever:
    实际数量 = count(pods with label app=hello)
    if 实际数量 < 3: 创建一个 Pod
    if 实际数量 > 3: 删一个 Pod
    sleep 几秒
```

这就是 K8s 自愈的本质 —— controller 不停对比期望 vs 实际，做差异调和（reconcile loop）。

### 3.1 试试看

```bash
kubectl apply -f rs.yaml
kubectl get pods
# hello-rs-abc12   1/1   Running   0   5s
# hello-rs-def34   1/1   Running   0   5s
# hello-rs-gh56i   1/1   Running   0   5s

# 手动删一个，观察 controller 自动补
kubectl delete pod hello-rs-abc12
kubectl get pods
# hello-rs-def34   1/1   Running   0   30s
# hello-rs-gh56i   1/1   Running   0   30s
# hello-rs-jkl78   1/1   Running   0   2s   ← 新建的！
```

## 4. Deployment —— 比 ReplicaSet 多一层管理

ReplicaSet 解决了"维持 N 份"的问题，但**改镜像版本时它不优雅** —— 直接把所有 Pod 重启，瞬间断流。

**Deployment** 是 ReplicaSet 之上的一层抽象，给两个主要能力：

1. **滚动升级**：升级镜像时一次只换一个 Pod，保证服务不中断
2. **回滚**：升级出问题，一键回到前一个版本

### 4.1 一个真实的 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
  namespace: default
  labels:
    app: hello
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # 升级时最多多 1 个
      maxUnavailable: 0     # 升级时最少不少
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - name: hello
          image: nginx:1.27
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 10
```

### 4.2 关键字段逐个看

```yaml
spec:
  replicas: 3
```

期望 3 份。

```yaml
  selector:
    matchLabels:
      app: hello
```

Deployment 用 `app: hello` label 找它管的 Pod。**这个 selector 必须跟 template.metadata.labels 一致** —— 否则 Deployment 找不到自己的 Pod。

```yaml
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

升级策略：

- `maxSurge: 1` —— 升级中可以多出 1 个 Pod（变成 4 个），不会更多
- `maxUnavailable: 0` —— 升级中**不允许**少于 3 个可用，避免服务降级

| `maxSurge` × `maxUnavailable` | 效果 |
|---|---|
| 1 × 0 | 先加 1 个新版 → 等就绪 → 删 1 个旧版（最稳，慢） |
| 0 × 1 | 先删 1 个旧版 → 加 1 个新版（快，但短暂少 1 个） |
| 25% × 25%（默认） | 平衡 |

```yaml
      containers:
        - resources:
            requests:
              cpu: "100m"           # 0.1 CPU 核
              memory: "128Mi"
            limits:
              cpu: "500m"           # 0.5 CPU 核
              memory: "512Mi"
```

资源管理：

- `requests` —— Pod 启动时需要保证拿到的最小资源（K8s 调度时按这个挑节点）
- `limits` —— Pod 能使用的最大资源，超过 CPU 限制会被 throttle，超过 memory 会被 OOM kill

**生产建议**：requests 一定要写（不写就会被任意调度）。limits 看场景 —— 内存务必写（否则一个 Pod 能把节点撑爆），CPU 可以宽松。

```yaml
          readinessProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 5
```

**Readiness probe** = "你准备好接流量了吗？" 没就绪的 Pod 不会被加入 Service 后端。常用 HTTP / TCP / exec 三种检测。

```yaml
          livenessProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 10
```

**Liveness probe** = "你还活着吗？" 失败次数超阈值 → K8s 重启容器。

> **教训**：滥用 livenessProbe 会让脆弱的 app 永远 restart loop。如果不知道选什么，**只配 readiness、不配 liveness** 是更安全的默认。

### 4.3 看一下背后的层次

`Deployment` apply 之后，K8s 内部其实是这样的：

```
Deployment "hello"
    ↓ 管
ReplicaSet "hello-7d8c4f5b6"   (一个版本)
    ↓ 管
Pod "hello-7d8c4f5b6-abc12"
Pod "hello-7d8c4f5b6-def34"
Pod "hello-7d8c4f5b6-gh56i"
```

升级镜像时：

```
Deployment "hello" (镜像换成 nginx:1.28)
    ↓ 管
ReplicaSet "hello-7d8c4f5b6" (旧版, replicas: 0)   ← 旧 RS 缩到 0 但保留
ReplicaSet "hello-9f8e2d1c4" (新版, replicas: 3)   ← 新 RS 起到 3
    ↓
Pod "hello-9f8e2d1c4-..." × 3
```

旧 ReplicaSet 不删 —— 留着是为了**回滚**（`kubectl rollout undo`）。

## 5. kubectl 实战

```bash
# 创建 / 更新
kubectl apply -f deployment.yaml

# 看
kubectl get deployments
kubectl get rs                       # ReplicaSets
kubectl get pods -l app=hello       # 按 label 过滤

# 升级镜像
kubectl set image deployment/hello hello=nginx:1.28
# 等于改 yaml 里的 image: nginx:1.28 然后 apply

# 看升级进度
kubectl rollout status deployment/hello

# 历史
kubectl rollout history deployment/hello

# 回滚
kubectl rollout undo deployment/hello
kubectl rollout undo deployment/hello --to-revision=2

# 缩放
kubectl scale deployment/hello --replicas=5

# 重启（强制重建所有 Pod，常用于改了 ConfigMap）
kubectl rollout restart deployment/hello
```

## 6. Service —— 给 Deployment 配个稳定地址

Pod 的 IP 是临时的。Deployment 滚动升级时旧 Pod 会消失、新 Pod 起来，IP 都变。客户端怎么找？

**Service** 给一组 Pod 一个稳定的虚拟 IP + DNS 名。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello
spec:
  selector:
    app: hello                # 匹配 Deployment 里的 label
  ports:
    - port: 80                # Service 暴露的端口
      targetPort: 80          # 转发到 Pod 的哪个端口
  type: ClusterIP             # 默认，集群内访问
```

类型：

| `type` | 谁能访问 | 用途 |
|---|---|---|
| `ClusterIP`（默认） | 集群内 | 内部服务 |
| `NodePort` | 集群外通过任一 node IP | 简单暴露 |
| `LoadBalancer` | 公网 | 真实生产对外 |

集群内访问：`http://hello.default.svc.cluster.local` 或简写 `http://hello`。

> Service 内容比较多，单独可以再写一章。这里只讲它跟 Deployment 的关系：**Deployment 跑 Pod，Service 帮其他人找到这些 Pod**。

## 7. 学完这一章应该会什么

- ✅ 理解 Pod 是最小调度单元，但 Pod 自己很脆弱
- ✅ 理解 ReplicaSet 维持 N 份，Deployment 在 ReplicaSet 上加滚动升级 + 回滚
- ✅ 看 Deployment manifest 能识别 selector / template / strategy / resources / probes 五大要素
- ✅ 知道 livenessProbe 配错的代价，默认 readiness only 更安全
- ✅ 用 `kubectl apply / get / rollout` 操作 Deployment 不慌
- ✅ 知道 Service 是给 Pod 配稳定地址的

下一章看 K8s 怎么跑"一次性"和"定时"任务。

---

> 下一章：[12 — K8s Job / CronJob / Namespace](12-k8s-job-cronjob.html) · [章节索引](./)
