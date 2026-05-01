---
layout: default
title: 12 — K8s Job / CronJob / Namespace
---

# 第 12 章：K8s — Job / CronJob / Namespace

> 上一章：[11 — Pod 与 Deployment](11-k8s-pod-deployment.html) · [章节索引](./)

Deployment 解决的是"常驻服务"。但很多任务**不是常驻**：

- 一次性数据迁移
- 每晚 0 点跑一次报表
- 每 5 分钟抓一次外部 API

K8s 给这些场景两个专门资源：**Job** 和 **CronJob**。最后顺带学 **Namespace**，因为生产集群一定会用到。

## 1. Job —— 跑到完成就停

### 1.1 Pod 的限制

普通 Pod 默认 `restartPolicy: Always` —— 容器退出（无论成功失败）就 restart。这不适合"跑完就结束"的任务。

Job 用 `restartPolicy: OnFailure` 或 `Never`，让任务**真正能终止**：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: import-data
spec:
  template:
    spec:
      containers:
        - name: importer
          image: my-app:1.0
          command: ["python", "import.py"]
      restartPolicy: OnFailure       # 失败重试，成功就停
  backoffLimit: 3                     # 最多重试 3 次（4 次尝试）
  activeDeadlineSeconds: 3600         # 1 小时不结束就杀掉
```

### 1.2 关键字段

```yaml
spec:
  backoffLimit: 3
```

失败重试次数。重试间隔指数退避（10s, 20s, 40s, ...）。

```yaml
  activeDeadlineSeconds: 3600
```

**整个 Job 的硬上限** —— 1 小时不管成功失败一律杀掉。**生产建议必须配** —— 否则 Pod 卡死会无限挂着烧资源。

```yaml
  ttlSecondsAfterFinished: 3600
```

完成 1 小时后自动清理这个 Job 资源（不然 `kubectl get jobs` 会越来越多）。

### 1.3 并行 Job

```yaml
spec:
  parallelism: 5            # 同时跑 5 个 Pod
  completions: 20           # 总共要 20 次成功
```

跑 20 次任务，每次 5 个并行 —— 总共起 20 个 Pod，4 批跑完。**适合数据切片处理**：每个 Pod 处理一部分输入。

### 1.4 操作

```bash
kubectl apply -f job.yaml
kubectl get jobs
# NAME           COMPLETIONS   DURATION   AGE
# import-data    1/1           45s        2m

kubectl get pods -l job-name=import-data
# import-data-abc12   0/1   Completed   0   2m

kubectl logs -l job-name=import-data
kubectl delete job import-data       # 删 Job 会连带删 Pod
```

## 2. CronJob —— 定时跑 Job

CronJob 是 Job 的"定时调度器"。每到时间就生成一个新 Job，让那个 Job 跑起来。

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-report
spec:
  schedule: "5 0 * * *"               # 每天 0:05（UTC）
  timeZone: "Etc/UTC"                 # 不写就是控制面所在时区，最好显式
  concurrencyPolicy: Forbid           # 上一个还没跑完不开新的
  successfulJobsHistoryLimit: 3       # 留 3 个成功记录
  failedJobsHistoryLimit: 3           # 留 3 个失败记录
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: reporter
              image: my-app:1.0
              command: ["python", "report.py"]
          restartPolicy: OnFailure
      backoffLimit: 2
      activeDeadlineSeconds: 1800
```

### 2.1 schedule —— cron 语法

```
"5 0 * * *"
 │ │ │ │ └──── 周几（0-6, 0=周日）
 │ │ │ └────── 月份（1-12）
 │ │ └──────── 日（1-31）
 │ └────────── 小时（0-23）
 └──────────── 分钟（0-59）
```

常见模式：

| schedule | 意思 |
|---|---|
| `*/5 * * * *` | 每 5 分钟 |
| `0 * * * *` | 每小时整点 |
| `0 0 * * *` | 每天 0:00 |
| `0 0 * * 0` | 每周日 0:00 |
| `0 0 1 * *` | 每月 1 号 0:00 |
| `5 0 * * *` | 每天 0:05 |

> ⚠️ **时区坑**：默认 schedule 走控制面的 UTC（K8s 1.25+ 用 `timeZone` 字段才能显式声明）。如果你写 "8:00 跑"想要的是北京时间，UTC 是 0:00 —— 但服务器的 0 点不一定就是北京 8 点（夏令时偏移），**强烈建议把所有 schedule 跟代码逻辑都用 UTC**。

### 2.2 concurrencyPolicy

| 值 | 行为 |
|---|---|
| `Allow`（默认） | 即使上一个 Job 还没结束，也开新 Job |
| `Forbid` | 上一个没结束就跳过这次（**最常用**） |
| `Replace` | 杀掉上一个，开新的 |

如果你的任务可能跑超过 schedule 间隔（例如每 5 分钟跑一次但单次需要 6 分钟），用 `Forbid` 避免雪崩。

### 2.3 操作

```bash
kubectl apply -f cronjob.yaml

kubectl get cronjobs
# NAME              SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
# nightly-report    5 0 * * *     False     0        12h             1d

# 看每次触发产生的 Job
kubectl get jobs -l cronjob-name=nightly-report
# nightly-report-28893900   1/1   45s   12h
# nightly-report-28893960   1/1   42s   11h

# 暂停（不删，停止生成新 Job）
kubectl patch cronjob nightly-report -p '{"spec":{"suspend":true}}'

# 立刻手动触发一次（测试用）
kubectl create job manual-test --from=cronjob/nightly-report

# 恢复
kubectl patch cronjob nightly-report -p '{"spec":{"suspend":false}}'
```

`suspend: true` 是运维"按暂停键"的常用手法 —— 比如发现 CronJob 在偷流量，先 patch suspend，再调查。

## 3. Namespace —— 集群里的"虚拟分区"

集群里东西多了，会乱。Namespace 是 K8s 的逻辑分组：

```bash
kubectl get ns
# NAME                STATUS   AGE
# default             Active   30d        ← 不指定 namespace 都进这里
# kube-system         Active   30d        ← K8s 系统组件（CoreDNS、kube-proxy）
# kube-public         Active   30d
# kube-node-lease     Active   30d
```

### 3.1 创建一个 Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
  labels:
    team: data
    env: prod
```

或者命令行：

```bash
kubectl create namespace my-app
```

### 3.2 把资源放进 Namespace

在 metadata 里写 `namespace`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
  namespace: my-app    # ← 这里
spec:
  ...
```

或者命令行临时覆盖：

```bash
kubectl apply -f deployment.yaml -n my-app
```

### 3.3 切换默认 Namespace

```bash
kubectl config set-context --current --namespace=my-app
# 之后 kubectl get pods 就只看 my-app 下的
```

或者 `kubens`（Krew 插件）让切换更顺手：

```bash
kubens my-app
```

### 3.4 跨 Namespace 访问

Service 的全名是 `<service>.<namespace>.svc.cluster.local`。同 namespace 可省略：

```
http://hello                                       # 同 namespace
http://hello.my-app                                # 跨 namespace 缩写
http://hello.my-app.svc.cluster.local              # 完整 FQDN
```

### 3.5 哪些资源是 namespaced，哪些是集群级

```bash
# Namespaced（属于某个 ns）
kubectl api-resources --namespaced=true | head
# NAME            SHORTNAMES   APIVERSION   NAMESPACED   KIND
# pods            po           v1           true         Pod
# deployments     deploy       apps/v1      true         Deployment
# services        svc          v1           true         Service
# configmaps      cm           v1           true         ConfigMap
# secrets                      v1           true         Secret
# jobs                         batch/v1     true         Job

# Cluster-scoped（不属于 ns）
kubectl api-resources --namespaced=false | head
# nodes           no           v1           false        Node
# namespaces      ns           v1           false        Namespace
# clusterroles                 rbac.authorization.k8s.io/v1   false   ClusterRole
# persistentvolumes  pv        v1           false        PersistentVolume
# storageclasses     sc        storage.k8s.io/v1   false   StorageClass
```

记忆：**Pod / Service / ConfigMap / Secret / Deployment / Job 都是 namespaced**；**Node / PV / StorageClass / ClusterRole 是集群级**。

### 3.6 Namespace 能做什么不能做什么

**能**：

- 命名空间隔离 —— 不同 namespace 同名资源不冲突
- ResourceQuota —— 限制 namespace 总 CPU/内存/Pod 数
- NetworkPolicy —— 控制 namespace 之间网络流量
- RBAC —— 给某 namespace 单独授权

**不能**：

- **不是安全边界** —— 默认 namespace 之间网络是通的，要靠 NetworkPolicy 隔离
- **不能存数据** —— 删 namespace 会连带删里面所有资源（包括 PVC）

## 4. 三个新概念怎么搭配

实际生产 layout：

```
Namespace: my-app
    ├── Deployment: api-server     (常驻 HTTP API)
    ├── Deployment: worker         (常驻 background worker)
    ├── CronJob:    nightly-batch  (每天跑批)
    ├── Service:    api-server     (内部访问 api)
    ├── ConfigMap:  app-config     (下章讲)
    └── Secret:     api-keys       (下章讲)

Namespace: my-app-staging
    ├── Deployment: api-server  (staging 副本)
    ├── ...

Namespace: monitoring
    ├── Deployment: prometheus
    ├── Deployment: grafana
```

每个 namespace 独立的"应用 + 配套" —— 同一个集群跑多个环境 / 多个团队。

## 5. 学完这一章应该会什么

- ✅ 知道 Job 跟 Deployment 的差别（"跑完就停" vs "永远跑"）
- ✅ 配 Job 时记得加 `activeDeadlineSeconds` + `ttlSecondsAfterFinished`
- ✅ 看 cron 表达式不头大，并知道时区坑（默认 UTC）
- ✅ 用 `concurrencyPolicy: Forbid` 防止 CronJob 雪崩
- ✅ 理解 Namespace 是逻辑隔离，不是安全隔离
- ✅ 切 namespace 用 `kubectl config set-context --current --namespace=...`
- ✅ 看到 `<svc>.<ns>.svc.cluster.local` 知道是 Service 的 DNS 全名

下一章看怎么把"配置"和"密钥"塞进容器，以及怎么让 Pod 数据**重启不丢**。

---

> 下一章：[13 — ConfigMap / Secret / PVC](13-k8s-configmap-secret-pvc.html) · [章节索引](./)
