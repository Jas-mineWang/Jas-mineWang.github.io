---
layout: post
title: "KubeRay 实战（四）：RayService、Ray Serve 与一次 HTTP 请求的完整链路"
date: 2026-09-01 12:00:00 +0800
categories: [ray, kuberay, kubernetes, ray-serve, microservices]
---

普通 Ray Job 适合一次性批处理、训练或数据计算；Ray Serve 面向长期运行的在线服务。到了 Kubernetes 环境，KubeRay 又提供 RayService CR，把底层 RayCluster、Serve 配置、健康检查和稳定访问入口组合成一个声明式对象。

本文以官方水果商店示例为线索，回答四个问题：

1. RayService 和 RayCluster 是什么关系；
2. Application、Deployment、Replica 分别是什么；
3. Dashboard 8265 与 Serve 8000 有什么区别；
4. `POST /fruit/` 到底触发了什么。

> 验证环境：KubeRay 1.5.1、Ray 2.46.0、RKE2。示例资源名已泛化。

> 系列导航：[（一）KubeRay 架构]({% post_url 2026-09-01-kuberay-architecture-crd-operator-labels %}) · [（二）Dashboard 与远程访问]({% post_url 2026-09-01-ray-dashboard-remote-access-observability %}) · [（三）Job 与自动扩缩容]({% post_url 2026-09-01-ray-jobs-tasks-actors-autoscaling %}) · **（四）RayService 与 Serve**

## RayService 是更高一层的声明

```text
RayService CR：rayservice-demo
├── spec.rayClusterConfig
│   └── 底层 RayCluster：Head + Worker
├── spec.serveConfigV2
│   ├── fruit_app
│   └── math_app
├── rayservice-demo-head-svc
│   └── Dashboard / 管理入口 8265
└── rayservice-demo-serve-svc
    └── Serve 业务入口 8000
```

创建 RayService 后，KubeRay Controller 会：

1. 根据 `rayClusterConfig` 创建底层 RayCluster；
2. 等待 Ray 集群就绪；
3. 根据 `serveConfigV2` 部署 Serve Application；
4. 创建稳定的 Head Service 和 Serve Service；
5. 持续检查 Application 和 Deployment 健康；
6. 更新配置时协调集群升级与流量切换。

查看三个层次：

```bash
kubectl get rayservice
kubectl get raycluster
kubectl get pods -l ray.io/is-ray-node=yes
```

RayService 不是 RayCluster 的替代品。它内部仍然需要一个 RayCluster 提供分布式计算资源，只是在上面增加了 Serve 应用生命周期管理。

## Application、Deployment、Replica 的层级

```text
Application
└── Deployment
    └── Replica：真正运行的 Ray Actor
```

### Application：应用和 HTTP 路由边界

一个 Application 定义一套可以独立访问的 Serve 应用：

```yaml
applications:
  - name: fruit_app
    import_path: fruit.deployment_graph
    route_prefix: /fruit
```

它主要包含：

- 应用名；
- 外部路由前缀；
- Python 部署图入口；
- Runtime Environment；
- 一组 Deployment 配置。

Application 自己不是运行进程，因此 Dashboard 中 Application 的 Replica 数通常显示 `-`。

### Deployment：应用内部的功能组件

Deployment 通常对应一个 `@serve.deployment` 修饰的 Python 类：

```python
@serve.deployment
class MangoStand:
    def check_price(self, amount):
        return self.price * amount
```

它定义：

- 使用哪个 Python 类；
- 创建多少 Replica；
- 每个 Replica 的 CPU、GPU；
- 用户配置；
- 扩缩容和健康检查参数。

注意：Ray Serve Deployment 不是 Kubernetes `apps/v1 Deployment`。Kubernetes 不会为每个 Serve Deployment 创建一个独立 Pod。

### Replica：真正工作的 Actor

每个 Replica 本质上是一个常驻 Ray Actor：

```text
MangoStand Deployment
├── Replica 1 → Ray Actor → 某个 Ray Pod 中的专用 worker 进程
└── Replica 2 → Ray Actor → 另一个 Ray Pod 中的专用 worker 进程
```

一个 Ray Worker Pod 可以承载多个 Deployment 的多个 Replica Actor。请求到达某个 Deployment 后，Serve 会选择健康 Replica 处理。

## 示例中的两个 Application

```text
fruit_app                         route_prefix: /fruit
├── FruitMarket                  1 Replica，HTTP 入口
├── MangoStand                   2 Replicas
├── OrangeStand                  1 Replica
└── PearStand                    1 Replica

math_app                          route_prefix: /calc
├── Router                       1 Replica，HTTP 入口
├── Adder                        1 Replica
└── Multiplier                   1 Replica
```

Dashboard 的 Serve 页面中：

- `Application status RUNNING × 2`：两个应用已经运行；
- Application 的 Route prefix：外部 HTTP 入口；
- Deployment `HEALTHY`：所需 Replica 已健康；
- `MangoStand Replicas=2`：有两个长期运行的 Replica Actor；
- 点击 Replica 数量可以追到 Actor ID、PID 和 Ray 节点；
- Deployment Logs 可以查看业务代码日志。

顶部的 Controller 与 Proxy 也是 Serve 系统组件：

- Serve Controller：维护 Application、Deployment 和 Replica 的目标状态；
- HTTP Proxy：接收外部请求并按路由转发到入口 Deployment；
- Proxy 数量取决于部署配置和 Ray 节点布局。

## Dashboard 8265 与 Serve 8000

RayService 通常创建两个稳定 Service。

### Head Service：管理面

```text
rayservice-demo-head-svc:8265
```

用于：

- Ray Dashboard；
- Jobs API；
- State API；
- Serve 管理 API；
- 运维和排障。

### Serve Service：业务面

```text
rayservice-demo-serve-svc:8000
```

用于：

- REST API；
- 模型推理请求；
- 在线业务流量；
- 其他 Kubernetes 应用调用。

最简记忆：

```text
8265：管理员看和管
8000：业务客户端调用
```

RayService 升级底层 RayCluster 时，稳定 Service 名可以继续作为客户端入口，KubeRay 在新集群健康后调整后端指向。

## 一次水果请求到底做了什么

从同 namespace 的 Kubernetes Pod 中发送：

```bash
curl -X POST \
  -H 'Content-Type: application/json' \
  http://rayservice-demo-serve-svc:8000/fruit/ \
  -d '["MANGO", 2]'
```

逐项解释：

- `POST`：HTTP 请求方法；
- `Content-Type: application/json`：请求体是 JSON；
- `rayservice-demo-serve-svc`：Kubernetes Service DNS；
- `8000`：Serve HTTP Proxy；
- `/fruit/`：匹配 `fruit_app` 的 route prefix；
- `["MANGO", 2]`：水果类型为 MANGO，数量为 2。

短 Service 名通常只在同 namespace 内解析。跨 namespace 可以使用：

```text
rayservice-demo-serve-svc.<namespace>.svc.cluster.local
```

完整请求链路：

```text
curl Pod
  ↓ HTTP POST
Kubernetes Service:8000
  ↓
Ray Serve HTTP Proxy
  ↓ 根据 /fruit/ 选择 fruit_app
FruitMarket Replica Actor
  ↓ 解析 ["MANGO", 2]
MangoStand Replica Actor
  ↓ check_price(2)
单价 3 × 数量 2
  ↓
HTTP 响应 6
```

示例中的入口代码大致是：

```python
async def __call__(self, request):
    fruit, amount = await request.json()
    return await self.check_price(fruit, amount)
```

`FruitMarket` 根据 `MANGO` 找到 MangoStand 的 DeploymentHandle，然后调用：

```python
await mango_stand.check_price.remote(2)
```

MangoStand 在 Serve 配置中的 `user_config.price` 是 3，因此预期响应：

```text
6
```

## HTTP 请求不是 Ray Job

这条 `curl` 没有创建新的 Ray Job。

```text
Ray Job
├── 通过 8265 Jobs API 提交入口程序
├── 启动 Driver
├── 适合批处理、训练、一次性计算
└── 程序完成后退出

Ray Serve Request
├── 通过 8000 访问已运行的 Application
├── 由常驻 Proxy 和 Replica Actor 处理
├── 适合 API、推理、在线服务
└── 单次请求结束后 Application 继续运行
```

Ray Job 可以调用 Serve，也可以用代码启动 Serve；但使用 RayService 时，通常应让 `serveConfigV2` 成为 Serve 部署的声明式事实来源，避免 Job 和 KubeRay Controller 同时修改同一套应用状态。

Dashboard 的 Recent Jobs 显示 `No jobs yet` 也不矛盾：KubeRay 是根据 RayService 配置通过 Serve 管理 API部署应用，不是通过 Jobs API 创建一个普通 Ray Job。

## 为什么没有请求，CPU 仍然被占用

Serve Replica 是常驻 Actor。即使当前没有 HTTP 流量，只要 Deployment 配置给 Replica 声明了 CPU：

```yaml
ray_actor_options:
  num_cpus: 0.1
```

Ray Scheduler 就会在 Replica 生命周期内为它保留相应逻辑资源。因此 Overview 中可能长期看到例如：

```text
0.8/3.0 CPU
```

这表示约 0.8 个 Ray 逻辑 CPU 已被 Actor 保留，不等于操作系统 CPU 正在持续进行 0.8 核的计算。

同理，Resource Status 中 `0B memory` 只表示没有声明 Ray 的逻辑 memory 调度资源，不表示 Python 进程没有实际内存占用。

## 如何读 MangoStand 的调度事件

启动阶段可能出现：

```text
resource request cannot be scheduled right now:
{
  'node:__internal_implicit_resource_fruit_app:MangoStand': 1.0,
  'CPU': 0.1
}
```

示例配置中 MangoStand 类似：

```yaml
name: MangoStand
num_replicas: 2
max_replicas_per_node: 1
ray_actor_options:
  num_cpus: 0.1
```

含义是：

- 需要两个 MangoStand Replica；
- 每个 Replica 请求 0.1 CPU；
- 同一个 Ray 节点最多放一个 MangoStand Replica；
- Serve 使用内部隐式资源实现这个每节点副本约束。

`node:__internal_implicit_resource_...` 不是 Kubernetes Label，也不需要用户手动定义。如果事件只出现在启动期，而当前所有 Deployment 都是 `HEALTHY`、Total Demands 为空，说明它是已经恢复的历史等待状态。

如果警告持续出现且 Application 无法 Running，应检查：

```bash
kubectl describe rayservice rayservice-demo
kubectl get raycluster
kubectl get pods -l ray.io/is-ray-node=yes -o wide
```

然后从 Serve 页面进入对应 Deployment/Replica 日志，并检查 Autoscaler 与 KubeRay Operator 日志。

## 从集群外访问 Serve

`rayservice-demo-serve-svc` 默认是 ClusterIP，只能在 Kubernetes 网络内访问。测试时可以转发：

```bash
kubectl port-forward \
  svc/rayservice-demo-serve-svc \
  8000:8000
```

再从同一台机器访问：

```bash
curl -X POST \
  -H 'Content-Type: application/json' \
  http://127.0.0.1:8000/fruit/ \
  -d '["MANGO", 2]'
```

如果浏览器或客户端位于另一台 Mac，还需要像上一篇访问 Dashboard 那样增加 SSH 隧道。生产环境应使用带认证、TLS、超时和流量治理能力的 Ingress/Gateway，而不是长期依赖 `kubectl port-forward`。

## 总结

> RayService 把 RayCluster 与 Serve 配置封装成一个 Kubernetes 声明；Application 定义应用和外部路由，Deployment 定义内部组件，Replica 是真正运行的 Actor。Dashboard 8265 属于管理面，Serve 8000 属于业务面；一次 HTTP 请求只调用已经存在的 Replica，不会创建新的 Ray Job。

至此，整个系列从 KubeRay 对象模型、Dashboard 网络路径、Ray Job 自动扩缩容，一直走到了长期在线的 Ray Serve 请求链路。

## 参考资料

- [RayService Quickstart](https://docs.ray.io/en/latest/cluster/kubernetes/getting-started/rayservice-quick-start.html)
- [Deploy Ray Serve on Kubernetes](https://docs.ray.io/en/latest/serve/production-guide/kubernetes.html)
- [Ray Serve Architecture](https://docs.ray.io/en/latest/serve/architecture.html)
- [KubeRay 1.5.1 RayService Sample](https://github.com/ray-project/kuberay/blob/v1.5.1/ray-operator/config/samples/ray-service.sample.yaml)
- [Fruit Application Source](https://github.com/ray-project/test_dag/blob/78b4a5da38796123d9f9ffff59bab2792a043e95/fruit.py)
