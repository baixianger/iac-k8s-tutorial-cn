---
layout: default
title: 16 — Kubernetes Python SDK
---

# 第 16 章：Kubernetes Python SDK

> 上一章：[15 — Kustomize 编排 manifest](15-kustomize.html) · [章节索引](./)

`kubectl` 是给人用的，**Python SDK 是给程序用的**。如果你要写：

- 一个 web 服务，根据 API 请求动态创建 Job
- 一个 controller / operator，watch 集群事件做自动化
- 一个 CI 脚本，部署前后做检查

—— 你需要的是 [`kubernetes`](https://github.com/kubernetes-client/python) 这个 Python 包。

## 1. 安装 + 加载

```bash
pip install kubernetes
```

```python
from kubernetes import client, config

# 本地开发：从 ~/.kube/config 加载
config.load_kube_config()

# 集群内（Pod 里跑）：从 ServiceAccount 自动加载
# config.load_incluster_config()
```

K8s 客户端两种加载方式：

| 函数 | 何时用 |
|---|---|
| `load_kube_config()` | 本地（kubectl 同机器） |
| `load_incluster_config()` | Pod 里跑（自动读 `/var/run/secrets/kubernetes.io/serviceaccount/`） |

实战写法：

```python
from kubernetes import client, config

def load_k8s():
    """Try in-cluster first, fall back to local kubeconfig."""
    try:
        config.load_incluster_config()
    except config.ConfigException:
        config.load_kube_config()

load_k8s()
```

这样同一份代码本地跑、集群里跑都行。

## 2. API 客户端的层次

K8s API 按 resource 类型分组：

| Python 类 | 管什么 |
|---|---|
| `CoreV1Api` | Pod, Service, ConfigMap, Secret, PVC, Namespace, Node, Event |
| `AppsV1Api` | Deployment, ReplicaSet, StatefulSet, DaemonSet |
| `BatchV1Api` | Job, CronJob |
| `NetworkingV1Api` | Ingress, NetworkPolicy |
| `RbacAuthorizationV1Api` | Role, RoleBinding, ClusterRole, ClusterRoleBinding |
| `StorageV1Api` | StorageClass |

每个类初始化一次：

```python
v1 = client.CoreV1Api()
apps = client.AppsV1Api()
batch = client.BatchV1Api()
```

## 3. 基本 CRUD

### 3.1 List（"get pods"）

```python
# 列所有 namespace 的 Pod
pods = v1.list_pod_for_all_namespaces(watch=False)
for p in pods.items:
    print(f"{p.metadata.namespace}/{p.metadata.name}: {p.status.phase}")

# 列某个 namespace 的 Pod
pods = v1.list_namespaced_pod(namespace="my-app")

# 按 label 过滤
pods = v1.list_namespaced_pod(
    namespace="my-app",
    label_selector="app=hello,env=prod",
)
```

### 3.2 Get（"describe pod my-pod"）

```python
pod = v1.read_namespaced_pod(name="my-pod", namespace="my-app")
print(pod.spec.containers[0].image)
print(pod.status.container_statuses[0].ready)
```

### 3.3 Create（"apply"）

写一个 Pod object 然后调 `create_*`：

```python
from kubernetes import client

pod = client.V1Pod(
    metadata=client.V1ObjectMeta(
        name="hello",
        namespace="my-app",
        labels={"app": "hello"},
    ),
    spec=client.V1PodSpec(
        containers=[
            client.V1Container(
                name="hello",
                image="nginx:1.27",
                ports=[client.V1ContainerPort(container_port=80)],
            )
        ],
        restart_policy="Always",
    ),
)

v1.create_namespaced_pod(namespace="my-app", body=pod)
```

每个 K8s 字段都有对应的 Python 类（`V1Pod` / `V1PodSpec` / `V1Container` / ...）。**字段名是 snake_case**（`container_port`），yaml 里的 camelCase（`containerPort`）由 SDK 自动转。

### 3.4 用 yaml 文件创建（更接近 kubectl apply）

```python
import yaml
from kubernetes import client, utils

with open("deployment.yaml") as f:
    docs = list(yaml.safe_load_all(f))

api_client = client.ApiClient()
for doc in docs:
    utils.create_from_dict(api_client, doc, namespace="my-app")
```

`create_from_dict` 会自动选对应的 `*_Api` 类。**写代码省 80% 的事**。

### 3.5 Update / Patch

```python
# Replace（覆盖式）—— 谨慎
pod.spec.containers[0].image = "nginx:1.28"
v1.replace_namespaced_pod(name="my-pod", namespace="my-app", body=pod)

# Patch（部分更新）—— 推荐
patch = {"spec": {"containers": [{"name": "hello", "image": "nginx:1.28"}]}}
v1.patch_namespaced_pod(name="my-pod", namespace="my-app", body=patch)

# 改 Deployment 里的 image —— 触发滚动升级
patch = {"spec": {"template": {"spec": {"containers": [{"name": "api", "image": "my-app:v2"}]}}}}
apps.patch_namespaced_deployment(name="api", namespace="my-app", body=patch)
```

### 3.6 Delete

```python
v1.delete_namespaced_pod(name="my-pod", namespace="my-app")
v1.delete_namespaced_pod(
    name="my-pod",
    namespace="my-app",
    body=client.V1DeleteOptions(grace_period_seconds=0),  # 强删
)
```

## 4. Watch —— 监听变化

```python
from kubernetes import client, config, watch

config.load_kube_config()
v1 = client.CoreV1Api()

w = watch.Watch()
for event in w.stream(v1.list_namespaced_pod, namespace="my-app", timeout_seconds=60):
    pod = event["object"]
    event_type = event["type"]   # ADDED / MODIFIED / DELETED
    print(f"{event_type} {pod.metadata.name}: {pod.status.phase}")
```

适合写 controller / operator —— "监听 Pod 变化、做相应反应"。

## 5. 一个真实例子：动态创建 Job

模仿真实项目里 K8s Manager 的 "创建 inpage Job" 模式：

```python
from kubernetes import client, config
from datetime import datetime

config.load_incluster_config()
batch = client.BatchV1Api()

def create_inpage_job(*, name: str, image: str, env: dict, namespace="my-app"):
    job = client.V1Job(
        api_version="batch/v1",
        kind="Job",
        metadata=client.V1ObjectMeta(
            name=name,
            namespace=namespace,
            labels={"app": "inpage", "created-at": datetime.utcnow().isoformat()},
        ),
        spec=client.V1JobSpec(
            backoff_limit=2,
            active_deadline_seconds=1800,
            ttl_seconds_after_finished=3600,
            template=client.V1PodTemplateSpec(
                metadata=client.V1ObjectMeta(labels={"app": "inpage"}),
                spec=client.V1PodSpec(
                    restart_policy="OnFailure",
                    containers=[
                        client.V1Container(
                            name="inpage",
                            image=image,
                            env=[
                                client.V1EnvVar(name=k, value=v)
                                for k, v in env.items()
                            ],
                            resources=client.V1ResourceRequirements(
                                requests={"cpu": "500m", "memory": "1Gi"},
                                limits={"cpu": "2", "memory": "4Gi"},
                            ),
                        )
                    ],
                ),
            ),
        ),
    )

    batch.create_namespaced_job(namespace=namespace, body=job)
    return name


# 用法
create_inpage_job(
    name="inpage-flashscore-1234",
    image="my-cloud-project/inpage:v1.0.0",
    env={"WEBSITE_ID": "flashscore", "EVENT_URL": "https://..."},
)
```

实际生产 wrapper 还会加：

- 重试（API 偶发 5xx）
- 等 Job 起来（`watch` 直到 status.startTime）
- 抓日志（`v1.read_namespaced_pod_log`）
- 失败时清理（`delete_namespaced_job`）

## 6. 错误处理

K8s API 抛 `kubernetes.client.exceptions.ApiException`：

```python
from kubernetes.client.exceptions import ApiException

try:
    v1.read_namespaced_pod(name="nonexistent", namespace="my-app")
except ApiException as e:
    if e.status == 404:
        print("Not found")
    elif e.status == 403:
        print("Forbidden — check RBAC")
    else:
        raise
```

`e.status` 是 HTTP 状态码，`e.body` 是 K8s 返回的 JSON 错误体。

## 7. Pod 里跑 Python SDK 需要的 RBAC

Pod 里要调 K8s API，得给 ServiceAccount 配 RBAC。比如要管 Job：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-job-creator
  namespace: my-app
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: job-manager
  namespace: my-app
rules:
  - apiGroups: ["batch"]
    resources: ["jobs"]
    verbs: ["get", "list", "create", "delete", "patch"]
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: job-manager-binding
  namespace: my-app
subjects:
  - kind: ServiceAccount
    name: my-job-creator
    namespace: my-app
roleRef:
  kind: Role
  name: job-manager
  apiGroup: rbac.authorization.k8s.io
```

然后让 Pod 用这个 SA：

```yaml
spec:
  serviceAccountName: my-job-creator
```

`load_incluster_config()` 时 SDK 自动用这个 SA 的 token，受 RBAC 限制。

## 8. 学完这一章应该会什么

- ✅ 知道 in-cluster vs local kubeconfig 两种加载方式
- ✅ 会列 / 读 / 创建 / 更新 / 删除 Pod / Deployment / Job
- ✅ 写 Python 模型时知道 snake_case ↔ camelCase 自动转
- ✅ 用 `watch.Watch().stream(...)` 监听集群事件
- ✅ 程序化创建 Job 时记得配 backoff / activeDeadline / ttl
- ✅ 给 Pod 配最小够用的 RBAC，不滥用 cluster-admin

工具篇至此讲完。下一章：把前面所有学的串起来，端到端走一遍。

---

> 下一章：[17 — 端到端走一遍](17-end-to-end.html) · [章节索引](./)
