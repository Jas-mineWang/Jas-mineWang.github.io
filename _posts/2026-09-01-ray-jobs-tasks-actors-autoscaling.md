---
layout: post
title: "KubeRay 实战（三）：从 Ray Job、Task、Actor 到 Worker Pod 自动扩缩容"
date: 2026-09-01 11:00:00 +0800
categories: [ray, kuberay, kubernetes, autoscaling, distributed-computing]
---

Ray 的自动扩缩容不是读取宿主机 CPU 使用率，然后发现“CPU 很忙”就增加 Pod。它关注的是 Task、Actor 和 Placement Group 声明的**逻辑资源需求**。

本文用一个可重复的实验还原完整链路：初始集群只有 2 个 Ray 逻辑 CPU，提交 4 个各需 1 CPU 的 Task，观察 Worker 从 1 个扩到 3 个，再在任务结束后缩回 1 个。

> 验证环境：KubeRay 1.5.1、Ray 2.46.0、RKE2。文中的名称、IP 和 Job ID 均已泛化。

> 系列导航：[（一）KubeRay 架构]({% post_url 2026-09-01-kuberay-architecture-crd-operator-labels %}) · [（二）Dashboard 与远程访问]({% post_url 2026-09-01-ray-dashboard-remote-access-observability %}) · **（三）Job 与自动扩缩容** · [（四）RayService 与 Serve]({% post_url 2026-09-01-rayservice-ray-serve-request-path %})

## 先分清六个概念

```text
Ray Job
└── Driver
    ├── 普通 Task → Python worker 进程
    └── Actor → 专用 Python worker 进程

Ray Cluster
├── Head Pod / Ray Node
│   └── Raylet 管理 worker 进程
└── Worker Pod / Ray Node
    └── Raylet 管理 worker 进程
```

### Job

一次完整的 Ray 应用运行。`ray job submit` 向 Dashboard Jobs API 提交入口命令和 Runtime Environment，Job 最终会变成 `SUCCEEDED`、`FAILED` 或 `STOPPED`。

### Driver

执行用户入口代码的 Python 进程。Driver 调用 `ray.init()` 连接集群，提交 Task、创建 Actor，并通过 `ray.get()` 等待结果。

### Task

Remote Function 的一次异步调用：

```python
@ray.remote(num_cpus=1)
def work(i):
    return i

ref = work.remote(1)
```

`@ray.remote` 只是定义 Remote Function，`work.remote()` 才创建 Task，并立即返回 `ObjectRef`。

### Actor

用 `@ray.remote` 修饰类后创建的有状态远程对象：

```python
@ray.remote(num_cpus=1)
class Counter:
    def __init__(self):
        self.value = 0

    def add(self, number):
        self.value += number
        return self.value

counter = Counter.remote()
ref = counter.add.remote(1)
```

普通 Task 可以在可用 worker 进程间灵活调度；Actor 会占用一个专用 worker 进程，其方法共享对象状态。

### Worker Pod / Ray Node

KubeRay 创建的 Kubernetes Pod，提供一组 Ray 逻辑资源。一个 Worker Pod 可以有多个 CPU，也可以运行多个 Python worker 进程。

### Python worker 进程

真正执行 Task 函数体或承载 Actor 的进程。日志中的 `pid=...` 指的是这种进程，而不是 Kubernetes Pod。

因此“Worker”必须结合上下文判断：它可能指 KubeRay Worker Pod，也可能指 Ray 的 Python worker 进程。

## 开启 KubeRay Autoscaler

RayCluster 中启用：

```yaml
spec:
  enableInTreeAutoscaling: true
  workerGroupSpecs:
    - groupName: cpu-group
      replicas: 1
      minReplicas: 1
      maxReplicas: 3
```

字段大小写必须准确：

```text
enableInTreeAutoscaling
```

可以先让 CRD 自己回答：

```bash
kubectl explain raycluster.spec.enableInTreeAutoscaling
```

启用后，Head Pod 应包含两个容器：

```text
ray-head
autoscaler
```

检查：

```bash
kubectl get pods \
  -l 'ray.io/cluster=raycluster-demo,ray.io/node-type=head' \
  -o jsonpath='{range .items[*]}{.metadata.name}{" containers="}{.spec.containers[*].name}{" ready="}{.status.containerStatuses[*].ready}{"\n"}{end}'
```

如果修改前已经存在 Head Pod，Operator 不一定会为了加入 sidecar 主动删除一个正在运行的 Head。测试环境可以在确认可接受 Ray 会话中断后删除旧 Head，让 Operator 按最新模板重建；生产环境必须先设计升级和状态恢复方案。

新 Head 就绪后，`ray status` 才会出现 Autoscaler 生成的状态摘要：

```bash
kubectl exec <head-pod> -c ray-head -- ray status
```

没有 Autoscaler sidecar 时，固定规模 Ray 集群仍然可以运行，但 `ray status` 可能显示 `No cluster status`。这不等于 GCS、Task 或 Raylet 已经失效；可以改用 Dashboard、State API、`ray list nodes` 和 `ray.cluster_resources()` 观察固定集群。

## 扩容前的基线

初始状态：

```text
Active:
 1 headgroup
 1 workergroup

Total Usage:
 0.0/2.0 CPU

Total Demands:
 (no resource demands)
```

这表示 Head 和 Worker 各向 Ray 注册 1 个逻辑 CPU，总计 2 CPU。

Ray CPU 是调度用的逻辑令牌，不是 CPU 核绑定或 cgroup 隔离。`num_cpus=1` 会限制调度并发，但 Ray 不会阻止函数内部再启动多个线程。Kubernetes CPU request/limit 与 Ray 逻辑 CPU 最好保持合理对应。

## 提交四个占用 CPU 配额的 Task

在能访问 Dashboard 8265 且安装了 Ray CLI 的客户端执行：

```bash
export RAY_DASHBOARD=http://127.0.0.1:18265
export DEMO_JOB="autoscale-demo-$(date +%s)"

ray job submit \
  --address "$RAY_DASHBOARD" \
  --submission-id "$DEMO_JOB" \
  --no-wait \
  -- python -c '
import ray
import socket
import time

ray.init()

@ray.remote(num_cpus=1)
def hold_cpu(index):
    hostname = socket.gethostname()
    print(f"task {index} started on {hostname}", flush=True)
    time.sleep(300)
    return hostname

refs = [hold_cpu.remote(i) for i in range(4)]
print("Submitted 4 tasks:", refs, flush=True)
print("Results:", ray.get(refs), flush=True)
'
```

这段代码没有 Actor。它创建的是：

```text
1 个 Ray Job
1 个 Driver
4 个普通 Task
4 个 ObjectRef
0 个 Actor
```

每个 Task 在 300 秒内持续占用 1 个 Ray 逻辑 CPU，即使 `time.sleep()` 并不持续消耗物理 CPU。

## `.remote()` 如何一路触发 Pod 扩容

```text
Driver 调用 4 次 hold_cpu.remote()
        ↓
Ray Scheduler 尝试为 4 个 Task 分配 4 CPU
        ↓
初始只有 2 CPU，剩余 Task 等待
        ↓
GCS 中出现 pending resource demand
        ↓
Autoscaler sidecar 读取需求并计算目标 Worker 数
        ↓
Autoscaler 修改 RayCluster WorkerGroup replicas
        ↓
KubeRay Operator 观察到 replicas 增加
        ↓
Kubernetes Scheduler 放置新 Worker Pod
        ↓
镜像拉取、容器初始化、Raylet 启动
        ↓
新 Raylet 向 GCS 注册资源
        ↓
等待中的 Task 获得 CPU 并开始运行
```

`.remote()` 不会直接调用 Kubernetes API。Ray Scheduler 只负责在 Ray 资源中调度；Autoscaler 负责计算规模；KubeRay Operator 才负责创建 Pod。

## 捕捉到的扩容中间态

实验中某一时刻，`ray status` 显示：

```text
Active:
 1 headgroup
 2 workergroup
Pending:
 IP not yet assigned: workergroup, waiting

Total Usage:
 3.0/3.0 CPU

Total Demands:
 {'CPU': 1.0}: 1+ pending tasks/actors
```

逐项解释：

- Head 和两个 Worker 已注册，共 3 CPU；
- 3 CPU 已被三个 Task 全部占用；
- 第四个 Task 仍需要 1 CPU；
- 第三个 Worker 正在创建，还没有完成 IP 分配或 Ray 注册；
- 没有 Recent failures，扩容决策本身正常。

Kubernetes 同期状态可以呈现为：

```text
head                    2/2   Running
worker-initial          1/1   Running
worker-scale-up-1       1/1   Running
worker-scale-up-2       0/1   Init:0/1
```

`kubectl describe pod worker-scale-up-2` 的事件如果只是：

```text
Successfully assigned ...
Pulling image "rayproject/ray:2.46.0"
```

说明它正在经历新节点冷启动，并非调度失败。只有出现 `FailedScheduling`、`ErrImagePull`、`ImagePullBackOff` 或 CNI 错误时，才进入相应故障分支。

## 一个 Pod 能否同时运行四个 Task

可以，前提是该 Pod 所属 Ray 节点至少注册了 4 个可用逻辑 CPU，而且没有其他 GPU、内存或自定义资源约束：

```text
一个 Worker Pod：4 Ray CPU
└── Raylet
    ├── Python worker → task 0（1 CPU）
    ├── Python worker → task 1（1 CPU）
    ├── Python worker → task 2（1 CPU）
    └── Python worker → task 3（1 CPU）
```

本次实验之所以扩成四个 Ray 节点，是因为每个节点只注册了约 1 CPU，而每个 Task 请求 1 CPU。

如果 Kubernetes 只限制 1 个物理 CPU，却让 Ray 注册 4 个逻辑 CPU，Ray 仍可能并发启动四个 Task，但 CPU 密集型进程会争抢同一个物理 CPU。反过来，即使容器有 4 CPU，如果 Ray 只注册 1 CPU，Scheduler 仍只会同时放置一个 `num_cpus=1` Task。

## 从 Job 日志验证调度结果

```bash
ray job logs \
  --address "$RAY_DASHBOARD" \
  "$DEMO_JOB"
```

最终结果经过泛化后类似：

```text
Submitted 4 tasks: [ObjectRef(...), ObjectRef(...), ...]
task 0 started on head
task 1 started on worker-initial
task 2 started on worker-scale-up-1
task 3 started on worker-scale-up-2
Results: ['head', 'worker-initial', 'worker-scale-up-1', 'worker-scale-up-2']
```

`ObjectRef` 是未来结果的引用。`ray.get(refs)` 会等待全部 Task，并按 `refs` 的输入顺序返回结果。

Ray 默认会折叠来自集群不同进程但形式相似的日志，可能显示：

```text
[repeated 2x across cluster]
```

这不表示 Task 少执行了，应结合最终 Results、Task State 和 Replica/Actor 信息判断。

## Task 完成后为什么又缩回去了

300 秒后，四个 Task 返回，Driver 的 `ray.get()` 获得全部结果，入口进程以退出码 0 结束：

```bash
ray job status \
  --address "$RAY_DASHBOARD" \
  "$DEMO_JOB"
```

最终状态：

```text
Job succeeded
```

随后：

```text
Task 结束
  ↓
Resource Demand 消失
  ↓
新增 Worker 进入空闲
  ↓
超过 idle timeout
  ↓
Autoscaler 把 replicas 从 3 降回 minReplicas=1
  ↓
KubeRay Operator 删除两个临时 Worker Pod
```

最终又回到：

```text
1 Head + 1 Worker = 2 Ray CPU
```

## Actor 也能触发扩容吗

可以。Autoscaler 关心 Task、Actor 和 Placement Group 的逻辑资源需求。

如果创建四个明确请求 1 CPU 的 Actor：

```python
@ray.remote(num_cpus=1)
class Counter:
    def __init__(self):
        self.value = 0

actors = [Counter.remote() for _ in range(4)]
```

就会产生 4 CPU 的 Actor 资源需求。Actor 生命周期通常长于普通 Task，因此只要这些 Actor 存活并保留资源，相应 Worker 就不会因为空闲而缩容。

但一个 Actor 上连续调用四次方法：

```python
counter.add.remote(1)
counter.add.remote(2)
counter.add.remote(3)
counter.add.remote(4)
```

仍然只是一个 Actor。普通同步 Actor 的方法默认在同一专用 worker 进程中顺序执行，不等于四个可并行的普通 Task。

## Ray Autoscaler 不扩 Kubernetes 物理节点

本次实验扩的是 Ray Worker Pod：

```text
Ray Task/Actor Demand
    ↓
Ray Autoscaler 增减 Worker Pod
```

如果 Kubernetes 节点没有足够 CPU 或内存，新 Worker Pod 会停在 `Pending`：

```text
Worker Pod Pending
    ↓
Kubernetes Cluster Autoscaler
    ↓
增加 Kubernetes Node
```

第二层需要另行配置 Kubernetes Cluster Autoscaler。Ray Autoscaler 不会直接创建 RKE2、云主机或物理服务器。

## 总结

这次实验最重要的不是“成功扩出了两个 Pod”，而是建立了完整证据链：

> Job 启动 Driver，Driver 用 `.remote()` 提交 Task，Task 的逻辑资源声明形成 Demand，Ray Autoscaler 修改 Worker 副本数，KubeRay Operator 创建 Pod，Raylet 注册后 Scheduler 放置 Task，需求消失后再反向缩容。

下一篇将从批处理转向在线服务：RayService 如何把底层 RayCluster 与长期运行的 Ray Serve Application、Deployment 和 Replica 组合在一起。

## 参考资料

- [Ray Tasks](https://docs.ray.io/en/latest/ray-core/tasks.html)
- [Ray Actors](https://docs.ray.io/en/latest/ray-core/actors.html)
- [Ray Logical Resources](https://docs.ray.io/en/latest/ray-core/scheduling/resources.html)
- [KubeRay Autoscaling](https://docs.ray.io/en/latest/cluster/kubernetes/user-guides/configuring-autoscaling.html)
- [Ray Jobs CLI](https://docs.ray.io/en/latest/cluster/running-applications/job-submission/cli.html)
