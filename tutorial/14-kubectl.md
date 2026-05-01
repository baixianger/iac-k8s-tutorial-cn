---
layout: default
title: 14 — kubectl 命令实战
---

# 第 14 章：kubectl 命令实战

> 上一章：[13 — ConfigMap / Secret / PVC](13-k8s-configmap-secret-pvc.html) · [章节索引](./)

K8s 的官方 CLI。**90% 的日常运维**靠它。这一章讲 5 个高频命令 + 几个能省一半时间的小技巧。

## 1. kubeconfig —— 告诉 kubectl 连哪个集群

`kubectl` 默认读 `~/.kube/config`（一个 yaml 文件）。里面有：

```yaml
apiVersion: v1
kind: Config

clusters:                       # 集群列表（连到哪）
  - name: gke-my-cluster
    cluster:
      server: https://35.198.X.Y
      certificate-authority-data: <base64 CA>
  - name: hetzner-prod
    cluster:
      server: https://hetzner-cp:6443
      certificate-authority-data: <base64 CA>

users:                          # 用户列表（你是谁）
  - name: gke-user
    user:
      auth-provider: { ... }
  - name: hetzner-admin
    user:
      client-certificate-data: <base64 cert>
      client-key-data: <base64 key>

contexts:                       # context = (cluster, user, namespace) 三元组
  - name: gke
    context:
      cluster: gke-my-cluster
      user: gke-user
      namespace: my-app
  - name: hetzner
    context:
      cluster: hetzner-prod
      user: hetzner-admin
      namespace: my-app

current-context: gke            # 当前用哪个
```

### 1.1 切 context

```bash
kubectl config get-contexts          # 看所有 context
kubectl config use-context hetzner   # 切到 hetzner
kubectl config current-context        # 看现在在哪
```

或者用 [kubectx](https://github.com/ahmetb/kubectx)（推荐）：

```bash
kubectx                              # 列表
kubectx hetzner                      # 切
kubectx -                            # 切回上一个
```

### 1.2 多 kubeconfig 文件

如果不想合并文件，可以用 `KUBECONFIG` 环境变量指多份：

```bash
export KUBECONFIG=~/.kube/gke-config:~/.kube/hetzner-config
kubectl config get-contexts          # 两份的 context 合在一起列
```

或者临时只用某一份：

```bash
KUBECONFIG=~/.kube/hetzner-config kubectl get pods
```

### 1.3 切 namespace

```bash
kubectl config set-context --current --namespace=my-app
```

或者 [kubens](https://github.com/ahmetb/kubectx)：

```bash
kubens my-app
```

## 2. 五个高频命令

### 2.1 `kubectl get` —— 列资源

```bash
kubectl get pods                              # 列 Pod
kubectl get pods -A                           # 所有 namespace（--all-namespaces）
kubectl get pods -n kube-system               # 指定 namespace
kubectl get pods -o wide                      # 多看一些列（含 node、IP）
kubectl get pods --watch                      # 持续监听变化（ctrl+c 停）

kubectl get pod my-pod                        # 单个
kubectl get pod my-pod -o yaml                # 完整 yaml
kubectl get pod my-pod -o json | jq '.spec.containers[0].image'    # JSON + jq

kubectl get all                               # 当前 ns 所有"标准资源"（pod/svc/deploy/rs/ds/job）

# 按 label 过滤
kubectl get pods -l app=hello
kubectl get pods -l 'env in (prod, staging)'
kubectl get pods -l 'app=hello,version!=v1'

# 自定义列
kubectl get pods -o custom-columns=NAME:.metadata.name,IP:.status.podIP,NODE:.spec.nodeName
```

最常用的 alias：`alias k=kubectl`、`alias kg='kubectl get'`、`alias kgp='kubectl get pods'`。

### 2.2 `kubectl describe` —— 看资源详情

```bash
kubectl describe pod my-pod
```

输出包含：

- 元信息（labels、annotations、owner references）
- 容器规格（image、ports、env、resources）
- 当前状态
- **Events** —— 这是黄金，K8s 的诊断都在这里

Pod 起不来？`describe` 看 Events：

```
Events:
  Type     Reason       Age   From      Message
  ----     ------       ----  ----      -------
  Warning  FailedScheduling  2m   default-scheduler  0/3 nodes are available: 3 Insufficient cpu.
```

—— 节点 CPU 不够，调度失败。

### 2.3 `kubectl apply` —— 创建 / 更新

```bash
kubectl apply -f deployment.yaml             # 单文件
kubectl apply -f ./manifests/                # 整个目录
kubectl apply -f https://example.com/m.yaml  # 直接拉 URL

kubectl apply -k ./overlays/prod             # Kustomize（下章讲）
kubectl apply --dry-run=server -f f.yaml     # 不真改，让 server 验证一下
```

`apply` 是**幂等**的：

- 资源不存在 → 创建
- 资源存在 → diff 然后 patch
- yaml 跟实际一致 → no-op

它的对手是 `kubectl create`（只创建，不更新）和 `kubectl replace`（覆盖式更新，会丢字段）—— **生产用 apply**。

### 2.4 `kubectl logs` —— 看容器日志

```bash
kubectl logs my-pod                       # 当前容器日志
kubectl logs my-pod -c my-container       # 多容器 Pod 指定容器
kubectl logs my-pod --previous            # 上一次 crash 的日志（debug crashloop）

kubectl logs -f my-pod                    # follow（跟 tail -f 一样）
kubectl logs --tail=100 my-pod            # 最后 100 行
kubectl logs --since=10m my-pod           # 最近 10 分钟

# 按 label 看一组 Pod 的日志
kubectl logs -l app=hello --tail=20

# 多容器：把所有容器的日志混在一起
kubectl logs my-pod --all-containers=true
```

实战工具推荐 [`stern`](https://github.com/stern/stern)（多 Pod 日志聚合）：

```bash
stern hello                          # 跟所有名字含 hello 的 Pod
stern -l app=hello                   # label 过滤
```

### 2.5 `kubectl exec` —— 进容器

```bash
kubectl exec my-pod -- ls /etc           # 跑一条命令
kubectl exec -it my-pod -- bash          # 交互式 shell
kubectl exec -it my-pod -c my-container -- bash    # 多容器指定容器

kubectl exec my-pod -- env                # 看进程环境变量
kubectl exec my-pod -- cat /etc/resolv.conf
```

`-it` = interactive + tty。**省略 `--` 也行**但写上更明确（区分 kubectl 自己的 flag vs 容器命令的 flag）。

## 3. 排错十二条军规

### 3.1 Pod 状态读法

```
NAME            READY   STATUS              RESTARTS   AGE
my-pod          1/1     Running             0          5m   ✓ 正常
my-pod          0/1     Pending             0          5m   ⚠️ 没调度上（CPU/内存不够 / nodeSelector 不满足）
my-pod          0/1     ContainerCreating   0          1m   ⚠️ 拉镜像 / 挂 volume 中
my-pod          0/1     ImagePullBackOff    0          2m   ❌ 镜像拉不到（密码错 / image tag 不存在）
my-pod          0/1     CrashLoopBackOff    5          3m   ❌ 进程一直崩
my-pod          0/1     OOMKilled           0          30s  ❌ 内存超限被杀
my-pod          1/1     Terminating         0          1m   🟡 在删
```

### 3.2 排错三板斧

```bash
# 板斧 1: describe 看 events
kubectl describe pod my-pod | grep -A 20 Events

# 板斧 2: logs（current + previous）
kubectl logs my-pod
kubectl logs my-pod --previous

# 板斧 3: exec 进去看
kubectl exec -it my-pod -- bash
```

90% 的 Pod 问题靠这三步定位。

### 3.3 节点级问题

```bash
kubectl get nodes
kubectl describe node node-1            # CPU/内存使用、condition、events
kubectl top node                         # 资源用量（要装 metrics-server）
kubectl top pod                          # Pod 资源用量
```

## 4. 几个能省一半时间的小招

### 4.1 短名称

```bash
kubectl get po                  # pods
kubectl get svc                 # services
kubectl get deploy              # deployments
kubectl get ds                  # daemonsets
kubectl get rs                  # replicasets
kubectl get cm                  # configmaps
kubectl get pv / pvc            # persistentvolumes / pvc
kubectl get ns                  # namespaces
kubectl get sa                  # serviceaccounts
kubectl get cj                  # cronjobs
```

### 4.2 输出加 jsonpath

```bash
# 拿某个 Service 的 ClusterIP
kubectl get svc hello -o jsonpath='{.spec.clusterIP}'

# 列出所有 Pod 的 image
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'

# 拿某个 ConfigMap 的某个 key 的值
kubectl get cm app-config -o jsonpath='{.data.config\.yaml}'
```

### 4.3 patch（改一个字段不重写整个 yaml）

```bash
# 加 label
kubectl label pod my-pod tier=backend

# 注解
kubectl annotate pod my-pod owner=team-data

# 改副本数
kubectl scale deployment/hello --replicas=5

# 暂停 CronJob
kubectl patch cronjob nightly-report -p '{"spec":{"suspend":true}}'

# 改 Deployment 里的镜像
kubectl set image deployment/hello hello=nginx:1.28
```

### 4.4 `--watch` 实时观察

```bash
kubectl get pods --watch                # 看 Pod 状态变化
kubectl get events --watch              # 看集群 events
kubectl rollout status deployment/hello # 跟踪滚动升级
```

### 4.5 explain —— 文档查询

```bash
kubectl explain pod                              # 看 Pod 的 schema
kubectl explain pod.spec.containers              # 钻进去
kubectl explain pod.spec.containers.resources    # 更深
kubectl explain pod --recursive | head -50       # 整个递归
```

不用查官方文档，本地直接看。**忘记字段名时极有用**。

### 4.6 port-forward —— 本地访问集群内部 Service

```bash
kubectl port-forward svc/hello 8080:80
# 浏览器开 http://localhost:8080 → 转发到集群里 hello service:80
```

或者直接转发到 Pod：

```bash
kubectl port-forward pod/my-pod 9090:9090
```

## 5. dry-run + diff —— 改前预演

```bash
# 客户端 dry-run（不连 API server）
kubectl apply -f f.yaml --dry-run=client -o yaml

# 服务端 dry-run（连 API server 但不真创建）
kubectl apply -f f.yaml --dry-run=server

# diff —— 看 apply 后会变什么
kubectl diff -f f.yaml
```

`kubectl diff` 是**改 production 前的最低安全门槛**。永远先 diff 再 apply。

## 6. 学完这一章应该会什么

- ✅ kubeconfig 三件套（cluster / user / context）能切自如
- ✅ get / describe / apply / logs / exec 这五招肌肉记忆
- ✅ Pod 起不来时知道 `describe ... | grep Events` + `logs --previous` 的诊断流程
- ✅ 知道 `set image` / `scale` / `patch` 这种轻量改动方式
- ✅ 知道 `--dry-run=server` + `kubectl diff` 是改 prod 前的安全网
- ✅ 装 [`kubectx` + `kubens`](https://github.com/ahmetb/kubectx) 立刻让 kubectl 体验上一个台阶

下一章看怎么把 manifest 在多环境（dev / staging / prod）里复用。

---

> 下一章：[15 — Kustomize 编排 manifest](15-kustomize.html) · [章节索引](./)
