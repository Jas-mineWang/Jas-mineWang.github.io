---
layout: post
title: "KubeRay 实战（二）：从 Mac 安全访问 Ray Dashboard，并用 CLI 读懂集群"
date: 2026-09-01 10:00:00 +0800
categories: [ray, kuberay, kubernetes, networking, observability]
---

在 Kubernetes Master 上执行：

```bash
kubectl port-forward svc/raycluster-demo-head-svc 8265:8265
```

终端已经显示：

```text
Forwarding from 127.0.0.1:8265 -> 8265
Forwarding from [::1]:8265 -> 8265
```

但 Mac 浏览器访问 `Master-IP:8265` 仍然失败。这个现象不是 Ray Dashboard 故障，而是对回环地址、端口转发和 SSH 隧道的作用范围理解不清。

> 验证环境：KubeRay 1.5.1、Ray 2.46.0、RKE2。示例中的地址和主机名均已泛化。

> 系列导航：[（一）KubeRay 架构]({% post_url 2026-09-01-kuberay-architecture-crd-operator-labels %}) · **（二）Dashboard 与远程访问** · [（三）Job 与自动扩缩容]({% post_url 2026-09-01-ray-jobs-tasks-actors-autoscaling %}) · [（四）RayService 与 Serve]({% post_url 2026-09-01-rayservice-ray-serve-request-path %})

## Dashboard 在哪里

Ray Dashboard 默认由 Head 节点提供，端口通常是 `8265`。KubeRay 创建的 Head Service 会暴露该端口：

```bash
kubectl get svc raycluster-demo-head-svc
```

可能看到：

```text
10001/TCP,8265/TCP,6379/TCP,8080/TCP,8000/TCP
```

在 Kubernetes 集群内部，可以通过 Service DNS 访问；在集群外部，`ClusterIP` 通常不可直达，需要 Ingress、VPN、LoadBalancer 或端口转发。

## 第一段：kubectl port-forward

在能访问 Kubernetes API 的 Master 或运维机上执行：

```bash
kubectl port-forward \
  svc/raycluster-demo-head-svc \
  8265:8265
```

它建立的是：

```text
运维机 127.0.0.1:8265
        ↓ kubectl port-forward
Kubernetes Service:8265
        ↓
Ray Head Pod:8265
```

验证：

```bash
curl -i http://127.0.0.1:8265/api/version
```

正常响应类似：

```text
HTTP/1.1 200 OK
Content-Type: application/json

{"version":"4","ray_version":"2.46.0",...}
```

如果这一步失败，问题在 Service、Endpoint、Head Pod 或 `kubectl port-forward`，还没有涉及 Mac。

## 为什么直接用 Master IP 不通

查看监听地址：

```bash
ss -lntp | grep ':8265'
```

如果显示：

```text
127.0.0.1:8265
[::1]:8265
```

就表示只监听运维机自己的回环网卡：

- 运维机上的 `curl 127.0.0.1:8265` 可以访问；
- Mac 的 `127.0.0.1` 指向 Mac 自己；
- `Master-IP:8265` 也无法命中仅绑定在回环地址上的监听器。

`ss` 输出中右侧的 `0.0.0.0:*` 是对端地址，不代表本地监听在所有网卡。

可以让 `kubectl port-forward` 绑定 `0.0.0.0`，但直接暴露 Ray Dashboard 往往不是好选择。Dashboard 通常不应未经认证地暴露到不可信网络。

## 第二段：SSH 本地端口转发

在 Mac 上建立 SSH 隧道：

```bash
ssh -N \
  -L 18265:127.0.0.1:8265 \
  -p 2222 \
  admin@master.example.com
```

逐项解释：

- `ssh`：先通过 SSH 连接远端运维机；
- `-p 2222`：SSH Server 的监听端口，替代默认的 22；
- `-N`：不在远端执行 Shell 命令，只做转发；
- `-L`：建立 Local Forward，本地监听一个端口；
- `18265`：Mac 本地监听端口；
- `127.0.0.1:8265`：由 SSH 远端主机访问的目标地址。

这里最容易误解的是：

> `-L` 中的 `127.0.0.1:8265` 是从 SSH 远端主机的视角解析，不是从 Mac 的视角解析。

完整链路变成：

```text
Mac 浏览器
http://127.0.0.1:18265
        ↓
Mac 上的 SSH Client
        ↓ SSH 加密连接，目标端口 2222
远端运维机
127.0.0.1:8265
        ↓ kubectl port-forward
Head Service:8265
        ↓
Ray Dashboard
```

然后在 Mac 验证：

```bash
curl -i http://127.0.0.1:18265/api/version
```

浏览器访问：

```text
http://127.0.0.1:18265
```

这里使用 `http`，除非前面另行配置了 TLS 终止，否则不要写成 `https`。

## SSH 为什么能转发 8265

SSH Server 监听 22 或 2222，只表示建立 SSH 会话时连接这个端口。会话建立后，SSH 协议可以在一条加密连接里承载多个逻辑通道，包括：

- 交互式 Shell；
- 文件传输；
- 本地端口转发；
- 远程端口转发；
- SOCKS 代理。

因此“SSH 连接 2222”和“最终访问 8265”并不冲突。2222 是隧道入口，8265 是隧道内部访问的目标。

## Dashboard 各栏目怎么读

Dashboard 是一个聚合视图，不同栏目对应 Ray 的不同运行对象。

### Overview

快速判断集群是否健康：

- Active、Pending、Recent failures；
- CPU、GPU、memory、object store memory；
- Resource Demands；
- 最近的 Jobs 和 Serve Deployments；
- 结构化 Events。

如果页面提示缺少 Prometheus/Grafana，表示历史时序图不可用，不等于 Ray 集群不可用。

### Jobs

查看通过 Ray Jobs API 提交的应用：

- Submission ID；
- Driver 状态；
- 入口命令；
- Job 日志；
- 成功或失败原因。

命令行对应：

```bash
ray job list --address http://127.0.0.1:18265
ray job status --address http://127.0.0.1:18265 <submission-id>
ray job logs --address http://127.0.0.1:18265 <submission-id>
```

### Cluster

查看 Ray 节点和 Raylet：

- Head/Worker 节点；
- 节点 IP、状态和资源；
- 进程与系统信息；
- 每个节点的日志入口。

命令行对应：

```bash
ray list nodes --address http://127.0.0.1:18265
```

### Actors

查看长期存在的有状态 worker，例如 Serve Replica、JobSupervisor 或用户 Actor：

```bash
ray list actors --address http://127.0.0.1:18265
```

### Metrics

用于资源和性能时间序列。完整体验通常需要 Prometheus 和 Grafana。

### Logs

用于按节点、文件名和关键字检索 Ray 日志。需要注意不同 CLI 子命令接受的地址类型并不完全相同。

在 Ray 2.46 中，下面这种写法可能失败：

```bash
ray logs cluster raylet.out \
  --address http://127.0.0.1:18265
```

原因是该子命令期待 GCS/bootstrap 风格地址，而不是 Dashboard HTTP URL。最直接的 Kubernetes 排障方式是进入目标 Pod 读取日志：

```bash
kubectl exec <head-pod> -c ray-head -- \
  tail -n 200 /tmp/ray/session_latest/logs/raylet.out
```

## 8265 不是所有 Ray 协议的统一入口

不同接口使用不同端口和协议：

| 接口 | 常见端口 | 主要协议/用途 |
| --- | --- | --- |
| Dashboard、Jobs、State、Serve 管理 | `8265` | HTTP/REST，日志流可能使用 WebSocket |
| GCS | `6379` | Ray 内部 RPC/控制通信 |
| Ray Client | `10001` | Ray Client 通信 |
| Serve HTTP | `8000` | 业务 HTTP 请求 |

外部执行 `ray job submit --address http://127.0.0.1:18265` 时，只通过 8265 把入口命令提交给 Jobs Server。Job 真正在集群里启动后，Driver 会使用集群内部的 `RAY_ADDRESS=<head-pod-ip>:6379` 连接 GCS。

## 一套分层排查顺序

遇到 Dashboard 无法访问时，不要一次检查所有组件，按路径逐段验证：

```text
1. Head Pod 是否 Ready
2. Head Service 是否有 Endpoint
3. 运维机 curl 127.0.0.1:8265 是否返回 200
4. 运维机 ss 是否监听 127.0.0.1:8265
5. Mac 是否监听 127.0.0.1:18265
6. SSH 隧道是否仍在运行
7. Mac curl 127.0.0.1:18265 是否返回 200
8. 最后再检查浏览器
```

每一步只验证一段链路，定位会比反复刷新浏览器快得多。

## 总结

> `kubectl port-forward` 解决 Kubernetes Service 到远端运维机回环端口的连接，SSH `-L` 再把 Mac 回环端口连接到远端回环端口。Dashboard 走 HTTP 8265，但 Job 进入集群后会使用内部 GCS 地址工作。

下一篇将通过四个 `num_cpus=1` 的 Task，完整观察 Ray 从 2 CPU 扩到 4 CPU，再缩回最小 Worker 数量的过程。

## 参考资料

- [Ray Jobs：访问远程集群](https://docs.ray.io/en/latest/cluster/running-applications/job-submission/quickstart.html#using-a-remote-cluster)
- [Kubernetes Port Forward](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/)
- [Ray Dashboard](https://docs.ray.io/en/latest/ray-observability/getting-started.html)
- [Ray State CLI](https://docs.ray.io/en/latest/ray-observability/reference/cli.html)
