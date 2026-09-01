---
layout: post
title: "KubeRay 实战（一）：Operator、RayCluster CRD 与 Pod Selector 到底是什么关系"
date: 2026-09-01 09:00:00 +0800
categories: [ray, kuberay, kubernetes, distributed-computing]
---

第一次在 Kubernetes 中部署 Ray 时，很容易把 KubeRay Operator、RayCluster、Head Service 和一组自动生成的 Pod 混成同一个东西。再看到下面这条命令，问题会更多：

```bash
kubectl get pods --selector=ray.io/cluster=raycluster-demo
```

`ray.io/cluster` 从哪里来？它是 KubeRay 的配置项、Kubernetes 的字段，还是我们可以随便定义的字符串？这篇文章从 Kubernetes 控制器模型出发，把这些对象的边界梳理清楚。

> 验证环境：KubeRay 1.5.1、Ray 2.46.0、RKE2。不同版本生成的字段和状态可能略有差异。

> 系列导航：**（一）KubeRay 架构** · [（二）Dashboard 与远程访问]({% post_url 2026-09-01-ray-dashboard-remote-access-observability %}) · [（三）Job 与自动扩缩容]({% post_url 2026-09-01-ray-jobs-tasks-actors-autoscaling %}) · [（四）RayService 与 Serve]({% post_url 2026-09-01-rayservice-ray-serve-request-path %})

## 先看全景：声明、控制器与实际资源

```text
RayCluster CRD
    ↓ 定义 Kubernetes 中可以出现 kind: RayCluster
RayCluster 对象 raycluster-demo
    ↓ 描述 Head、Worker、资源与扩缩容边界
KubeRay Operator
    ↓ watch + reconcile
Service、Head Pod、Worker Pod、RBAC 等 Kubernetes 资源
    ↓ Pod 内执行 ray start
真正运行的 Ray 集群
```

这条链路里，每一层的职责都不同。

### CRD：扩展 Kubernetes API 的类型定义

CRD 是 CustomResourceDefinition。安装 KubeRay Operator 时，集群中会注册 `RayCluster`、`RayJob` 和 `RayService` 等自定义资源类型。

```bash
kubectl get crd | grep ray.io
```

查看 RayCluster 的字段定义：

```bash
kubectl explain raycluster.spec
kubectl explain raycluster.spec.workerGroupSpecs
```

CRD 只定义“对象允许长什么样”，不会自己创建 Pod。

### RayCluster：一份期望状态

下面是一份精简后的 RayCluster：

```yaml
apiVersion: ray.io/v1
kind: RayCluster
metadata:
  name: raycluster-demo
spec:
  rayVersion: "2.46.0"
  headGroupSpec:
    template:
      spec:
        containers:
          - name: ray-head
            image: rayproject/ray:2.46.0
  workerGroupSpecs:
    - groupName: cpu-group
      replicas: 1
      minReplicas: 1
      maxReplicas: 3
      template:
        spec:
          containers:
            - name: ray-worker
              image: rayproject/ray:2.46.0
```

它表达的是：需要一个 Head，以及 `cpu-group` 中一个初始 Worker。它不是进程，也不是命令。

因此在 Shell 中直接输入资源名称：

```bash
raycluster-demo-head
```

得到 `command not found` 是正常的。Pod 名、Service 名和容器名都是 Kubernetes 对象标识，不会自动变成 Linux 可执行文件。

### Operator：持续执行 reconcile

KubeRay Operator watch RayCluster。当实际状态与期望状态不一致时，它会执行调谐：

- 创建 Head Service；
- 创建 Head Pod；
- 创建或删除 Worker Pod；
- 注入启动参数和环境变量；
- 在启用 Autoscaler 时创建相关 RBAC；
- 将结果写回 RayCluster 的 `status`。

Operator 不是一次性安装脚本。只要它在运行，就会持续处理对象变化。

```bash
kubectl get pods \
  -l app.kubernetes.io/name=kuberay-operator \
  -o wide
```

## Head Pod 与 Worker Pod 的分工

一个 RayCluster 通常包含一个 Head 和若干 Worker：

```text
Head Pod
├── GCS：集群元数据和控制状态
├── Dashboard / Jobs Server：默认 8265
├── Raylet：节点资源与 worker 进程管理
└── Object Store：分布式对象存储

Worker Pod
├── Raylet
├── Object Store
└── Python worker 进程：执行 Task 或承载 Actor
```

Head 也是一个 Ray 节点。如果它向 Ray 注册了逻辑 CPU，Scheduler 同样可以把用户 Task 放到 Head。生产集群常把 Head 的 Ray 逻辑 CPU 设为 0，让它专注于控制面；Kubernetes 仍应给 Head 容器配置足够的真实 CPU 和内存。

## Head Service 暴露哪些端口

KubeRay 会为 Head 创建 Service，常见端口如下：

| 端口 | 用途 |
| --- | --- |
| `8265` | Ray Dashboard、Jobs API、State API、Serve 管理 API |
| `6379` | GCS Server |
| `10001` | Ray Client |
| `8000` | Ray Serve HTTP |
| `8080` | Prometheus metrics |

查看实际端口：

```bash
kubectl get svc raycluster-demo-head-svc
```

Service 名称也不是命令。它的作用是在 Kubernetes 网络中为一组后端 Pod 提供稳定入口。

## Selector 到底是什么

Kubernetes Label 是附着在对象上的键值对：

```yaml
metadata:
  labels:
    ray.io/cluster: raycluster-demo
    ray.io/node-type: head
```

Selector 是查询条件：

```bash
kubectl get pods \
  --selector=ray.io/cluster=raycluster-demo
```

也可以简写为：

```bash
kubectl get pods -l ray.io/cluster=raycluster-demo
```

这条命令的意思不是“连接 Ray 集群”，而是：

```text
从当前 namespace 的所有 Pod 中，
筛选 label ray.io/cluster 等于 raycluster-demo 的对象。
```

继续增加条件：

```bash
kubectl get pods \
  -l 'ray.io/cluster=raycluster-demo,ray.io/node-type=head'
```

多个条件之间是 AND：既属于该 RayCluster，同时又是 Head。

### 标签是谁添加的

`ray.io/cluster`、`ray.io/node-type` 等标签通常由 KubeRay Operator 在生成 Pod 时添加。先看真实标签，比死记文档更可靠：

```bash
kubectl get pod <pod-name> --show-labels
```

或者：

```bash
kubectl get pod <pod-name> -o jsonpath='{.metadata.labels}'
```

### 可以自己定义 Label 吗

可以。Kubernetes 允许在模板中添加自定义 Label：

```yaml
template:
  metadata:
    labels:
      workload.example.com/team: ml-platform
      workload.example.com/tier: batch
```

然后查询：

```bash
kubectl get pods -l workload.example.com/team=ml-platform
```

但不要随意覆盖 `ray.io/*`、`app.kubernetes.io/*` 等由 KubeRay 或 Helm 管理的标签。这些标签可能被 Service Selector、Operator 调谐或运维命令依赖。

还要区分两个完全不同的概念：

- Kubernetes Label Selector：筛选 Kubernetes 对象，或控制 Pod 调度；
- Ray 逻辑资源/调度约束：决定 Ray Task、Actor 放到哪个 Ray 节点。

即使二者都叫 label 或 selector，也不属于同一个调度系统。

## 如何从 RayCluster 追到 Pod

一组实用的只读命令：

```bash
# 查看声明和状态
kubectl get raycluster raycluster-demo -o yaml

# 查看该集群所有 Pod
kubectl get pods -l ray.io/cluster=raycluster-demo -o wide

# 只看 Head
kubectl get pods \
  -l 'ray.io/cluster=raycluster-demo,ray.io/node-type=head'

# 只看 Worker
kubectl get pods \
  -l 'ray.io/cluster=raycluster-demo,ray.io/node-type=worker'

# 查看事件和调度原因
kubectl describe pod <pod-name>

# 查看 Operator 调谐日志
kubectl logs <kuberay-operator-pod>
```

如果修改 CR 后状态没有更新，可以比较：

```bash
kubectl get raycluster raycluster-demo \
  -o jsonpath='generation={.metadata.generation} observedGeneration={.status.observedGeneration}{"\n"}'
```

- `generation`：用户期望状态变化了多少代；
- `observedGeneration`：控制器已经处理到哪一代。

二者长期不一致时，应优先检查 Operator 日志和 CR 的 Events。

## 总结

最重要的职责边界是：

> CRD 定义类型，RayCluster 描述期望，KubeRay Operator 执行调谐，Pod 承载 Ray 进程，Label 负责标识对象，Selector 只负责筛选。

理解这条链路后，`kubectl get pods --selector=...` 就不再是一条需要死记的魔法命令，而是一次普通的 Kubernetes 标签查询。

下一篇将继续解决另一个常见困惑：为什么 Master 上已经监听 `127.0.0.1:8265`，Mac 浏览器仍然打不开 Ray Dashboard。

## 参考资料

- [Ray on Kubernetes](https://docs.ray.io/en/latest/cluster/kubernetes/index.html)
- [RayCluster Configuration](https://docs.ray.io/en/latest/cluster/kubernetes/user-guides/config.html)
- [Kubernetes Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
- [Ray Logical Resources](https://docs.ray.io/en/latest/ray-core/scheduling/resources.html)
