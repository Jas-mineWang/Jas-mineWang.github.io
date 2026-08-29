---
layout: post
title: "PyTorch DDP 训练的数据到底怎么走：DataLoader、DistributedSampler 与 DDP 的职责边界"
date: 2026-08-29 14:00:00 +0800
categories: [pytorch, distributed-training, deep-learning]
---

在 PyTorch 分布式训练中，`DataLoader`、`DistributedSampler` 和 `DistributedDataParallel`（DDP）经常同时出现。它们像是一套完整的数据流水线，但职责并不相同。

先记住最重要的结论：

> `DistributedSampler` 负责“每个进程读哪些数据”，`DataLoader` 负责“怎么把数据读成 batch”，DDP 负责“怎么同步各进程的模型梯度”。**DDP 不会自动把 CPU 上的 batch 搬到 GPU。**

## 三个组件分别做什么

| 组件 | 比喻 | 核心职责 | 输入 → 输出 |
| --- | --- | --- | --- |
| `DataLoader` | 生产线上的传送带 | 读取样本、调用预处理、组装 batch，并可以用多个 worker 并行加载 | 样本索引 → CPU Tensor Batch |
| `DistributedSampler` | 传送带上的分拣员 | 按 `rank` 为每个训练进程分配当前 epoch 需要读取的样本索引 | 完整数据集索引 → 当前 rank 的索引子集 |
| `DDP` | 多条生产线之间的协调器 | 包装各进程中的模型副本，并在反向传播时通过 All-Reduce 同步梯度 | 各 GPU 的局部梯度 → 平均后的一致梯度 |

一个典型的单机多卡 DDP 任务会启动多个进程，通常是“一个进程对应一块 GPU”。每个进程都拥有自己的 sampler、DataLoader、模型副本和优化器。

## 真正的数据流

训练数据大致按以下顺序流动：

```text
硬盘中的数据
    ↓
DistributedSampler 产生当前 rank 的样本索引
    ↓
DataLoader 读取、预处理并组装 CPU batch
    ↓
训练代码显式调用 batch.to(device)
    ↓
当前 GPU 上的 DDP 模型执行前向传播
    ↓
loss.backward()
    ↓
DDP 在反向传播过程中同步各 GPU 的梯度
    ↓
各进程的 optimizer.step() 独立执行，模型参数保持一致
```

这里最容易被误解的是 CPU 到 GPU 这一步。DDP 包装的是模型，它不会接管 DataLoader 返回的数据，所以必须在训练循环中显式调用：

```python
inputs = inputs.to(device, non_blocking=True)
targets = targets.to(device, non_blocking=True)
```

## 一份职责清晰的 DDP 训练代码

```python
import os

import torch
import torch.distributed as dist
from torch.nn.parallel import DistributedDataParallel as DDP
from torch.utils.data import DataLoader, DistributedSampler


def train():
    # torchrun 会为每个进程注入 LOCAL_RANK、RANK 和 WORLD_SIZE
    local_rank = int(os.environ["LOCAL_RANK"])
    torch.cuda.set_device(local_rank)
    dist.init_process_group(backend="nccl")

    device = torch.device("cuda", local_rank)

    dataset = MyDataset()

    # 当前进程只读自己的那份样本索引
    sampler = DistributedSampler(
        dataset,
        num_replicas=dist.get_world_size(),
        rank=dist.get_rank(),
        shuffle=True,
        drop_last=False,
    )

    dataloader = DataLoader(
        dataset,
        batch_size=32,
        sampler=sampler,
        shuffle=False,  # 顺序由 sampler 管理，这里不能再开 shuffle
        num_workers=4,
        pin_memory=True,
    )

    # 先把模型放到当前 GPU，再用 DDP 包装
    model = MyModel().to(device)
    model = DDP(
        model,
        device_ids=[local_rank],
        output_device=local_rank,
    )

    optimizer = torch.optim.AdamW(model.parameters(), lr=1e-4)
    criterion = torch.nn.CrossEntropyLoss()

    try:
        for epoch in range(100):
            # 让各 rank 在新 epoch 使用一致但不同于上一轮的乱序结果
            sampler.set_epoch(epoch)

            for inputs, targets in dataloader:
                # DataLoader 返回的 batch 仍在 CPU，需要显式搬到 GPU
                inputs = inputs.to(device, non_blocking=True)
                targets = targets.to(device, non_blocking=True)

                optimizer.zero_grad(set_to_none=True)
                outputs = model(inputs)
                loss = criterion(outputs, targets)
                loss.backward()  # DDP 在这个过程中同步梯度
                optimizer.step()
    finally:
        dist.destroy_process_group()


if __name__ == "__main__":
    train()
```

使用 4 块 GPU 启动：

```bash
torchrun --standalone --nproc_per_node=4 train.py
```

## 为什么每个 epoch 都要调用 `set_epoch`

`DistributedSampler` 需要让所有 rank 使用相同的随机种子生成同一份全局乱序列表，然后再切分给不同 rank。

如果没有调用 `sampler.set_epoch(epoch)`，每个 epoch 的样本顺序都可能完全相同。这不会让训练立即报错，却会削弱 shuffle 的意义。

## 样本一定不重复、不遗漏吗

不一定。当数据集大小无法被 `world_size` 整除时：

- `drop_last=False` 时，sampler 会补齐一些索引，因此少量样本可能重复。
- `drop_last=True` 时，sampler 会丢掉不足以平均分配的尾部样本。

这是为了保证每个 rank 执行相同数量的训练 step，避免某些进程提前结束后，其他进程还在等待梯度通信。

## Batch Size 到底是多少

DataLoader 中的 `batch_size=32` 是**单个进程、单块 GPU 的局部 batch size**。

如果有 4 块 GPU，且没有使用梯度累积，则：

```text
全局 batch size = 32 × 4 = 128
```

增加 GPU 数量会改变全局 batch size，可能需要同时调整学习率和学习率预热策略。

## 常见误区

### 1. 认为 DDP 会自动搬运 batch

DDP 只管理模型和梯度通信。模型和 batch 都必须由代码放到正确的 GPU。

### 2. 同时设置 `sampler` 和 `shuffle=True`

DataLoader 中 `sampler` 与 `shuffle=True` 不能同时使用。分布式乱序应交给 `DistributedSampler` 完成。

### 3. 把 `num_workers` 理解为 GPU 数量

`num_workers` 是**每个训练进程**用来加载数据的 CPU worker 数量。例如 4 个 DDP 进程且 `num_workers=4`，理论上会创建 16 个 DataLoader worker，需要结合 CPU 核数和内存调整。

### 4. 认为 DDP 在 `optimizer.step()` 时同步参数

DDP 的核心同步发生在 `loss.backward()` 期间。各进程得到一致的梯度后，再各自执行相同的 `optimizer.step()`，因此参数仍保持一致。

## 总结

用一句话概括这三者的分工：

> Sampler 决定“读哪些”，DataLoader 决定“怎么读”，训练代码负责“搬到哪块 GPU”，DDP 负责“怎么让各 GPU 的梯度保持一致”。

把这条边界理清后，排查“数据是否重复”、“batch 为什么还在 CPU”、“为什么多卡训练卡住”等问题会容易很多。

## 参考资料

- [PyTorch `DistributedDataParallel` 文档](https://docs.pytorch.org/docs/stable/generated/torch.nn.parallel.DistributedDataParallel.html)
- [PyTorch `DistributedSampler` 与 DataLoader 文档](https://docs.pytorch.org/docs/stable/data.html#torch.utils.data.distributed.DistributedSampler)
- [PyTorch `pin_memory` 与 `non_blocking` 教程](https://docs.pytorch.org/tutorials/intermediate/pinmem_nonblock.html)
