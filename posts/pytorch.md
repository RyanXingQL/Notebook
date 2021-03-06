# PYTORCH

- [PYTORCH](#pytorch)
  - [基础](#基础)
  - [多卡](#多卡)

## 基础

<details>
<summary><b>安装</b></summary>
<p>

根据 CUDA 教程，安装好系统推荐的 NVIDIA 驱动时，CUDA 就自动安装好了。注意，`nvidia-smi` 不准确，`nvcc -V` 才是准确的 CUDA 版本。

确定所需 PT 版本。在官网查看兼容的 CUDA 版本。若不满足，可重装 CUDA 及对应的最高版本 NVIDIA 驱动。

按照[官网](https://pytorch.org/get-started/locally/)提供的完整指令，用 PIP 安装。例：
  
```bash
pip install torch==1.6.0+cu101 torchvision==0.7.0+cu101 -f https://download.pytorch.org/whl/torch_stable.html
```

可以指定 CUDA 版本，推荐。

或用 CONDA 安装（不推荐）：

```bash
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/pytorch/
condo create -n pt python=3.7
conda activate pt
```

</p>
</details>

<details>
<summary><b>VISDOM</b></summary>
<p>

更推荐在高版本 PT 中使用 TENSORBOARD。

- 安装：`python -m pip install visdom`。
- 基础指令：
  
  ```python
  from visdom import Visdom
  
  viz = Visdom()
  viz.line([x], [y], win='loss', opts=dict(title='loss vs. iter, legend=['loss']), update='append')
  viz.image(img, win='a image')
  ```

- 开启服务：`python -m visdom.server`。
- 查看远程服务器的 VISDOM，需要先转接。
  
  ```bash
  ssh 18097:127.0.0.1:8097 x@xxx.xx.xx.xx
  ```
  
  其中 8097 是服务器端口，18097 是本机端口。
  
  然后查看 `http://localhost:18097` 即可。

</p>
</details>

<details>
<summary><b><code>torch.mean</code></b></summary>
<p>

```python
torch.mean(input, dim, keepdim=False)
```

```python
>>> a = torch.randn(4, 4)
>>> a
tensor([[-0.3841,  0.6320,  0.4254, -0.7384],
        [-0.9644,  1.0131, -0.6549, -1.4279],
        [-0.2951, -1.3350, -0.7694,  0.5600],
        [ 1.0842, -0.9580,  0.3623,  0.2343]])
>>> torch.mean(a, 1)
tensor([-0.0163, -0.5085, -0.4599,  0.1807])
>>> torch.mean(a, 1, True)
tensor([[-0.0163],
        [-0.5085],
        [-0.4599],
        [ 0.1807]])
```

重点说明 `dim`。

例子为 h=4，w=4 的矩阵。当 `dim=0` 时，意思是将 h=4 缩减为 1，因此是将对应 w（相同 h）的值求平均（纵向求平均）。当 `dim=1` 时，意思是将 w=4 缩减为 1，因此是将对应 h（相同 w）的值求平均（横向求平均）。

</p>
</details>

## 多卡

- **DDP 例程，各种后端对比等**：参见[PT 官方文档](https://pytorch.org/docs/stable/distributed.html)。

<details>
<summary><b>NCCL 后端 + launch 启动 + DistributedSampler</b></summary>
<p>

1. NCCL：由 NVIDIA 提供，不支持 CPU，GPU 支持极佳。推荐。
2. launch：使用 `torch.distributed.launch` 启动 DDP 模式，启动命令略复杂。
3. `DistributedSampler`：数据需要自动分配到各进程。

```python
"""
test.py
$ CUDA_VISIBLE_DEVICES=0,1 \
python -m torch.distributed.launch --nproc_per_node=2 \
test.py
"""
import os
import torch
import torch.nn as nn
from torch.utils.data import Dataset, DataLoader
from torch.utils.data.distributed import DistributedSampler
from torch.nn.parallel import DistributedDataParallel as DDP


torch.set_num_threads(6)
print("the number of cpu threads: {}".format(torch.get_num_threads()))

input_size = 5
output_size = 2
batch_size = 30
data_size = 90


class RandomDataset(Dataset):
    def __init__(self, size, length):
        self.len = length
        self.data = torch.randn(length, size)

    def __getitem__(self, index):
        return self.data[index]

    def __len__(self):
        return self.len


class Model(nn.Module):
    def __init__(self, input_size, output_size):
        super(Model, self).__init__()
        self.fc = nn.Linear(input_size, output_size)

    def forward(self, input):
        output = self.fc(input)
        return output

# 1) 初始化
torch.distributed.init_process_group(backend="nccl")

# 2） 配置每个进程的GPU
# 自动获取local_rank
local_rank = torch.distributed.get_rank()
torch.cuda.set_device(local_rank)
device = torch.device("cuda", local_rank)

# 创建数据集
dataset = RandomDataset(input_size, data_size)
# 3）使用DistributedSampler
rand_loader = DataLoader(dataset=dataset,
                         batch_size=batch_size,
                         sampler=DistributedSampler(dataset))

# 创建模型，转移到对应设备
model = Model(input_size, output_size)
model.to(device)

if torch.cuda.device_count() > 1:
    if local_rank == 0:
        print("Let's use", torch.cuda.device_count(), "GPUs!")
    # 4) 用DDP封装模型
    model = DDP(model, device_ids=[local_rank], output_device=local_rank)

for data in rand_loader:
    if torch.cuda.is_available():
        input_var = data.to(device)
    else:
        input_var = data

    output = model(input_var)
    print(f"Model {local_rank}, input size: {input_var.size()}, "+\
        f"output size: {output.size()}")

```

输出：

```bash
*****************************************
Setting OMP_NUM_THREADS environment variable for each process to be 1 in default, to avoid your system being overloaded, please further tune the variable for optimal performance in your application as needed.
*****************************************
the number of cpu threads: 6
the number of cpu threads: 6
Let's use 2 GPUs!
Model 0, input size: torch.Size([30, 5]), output size: torch.Size([30, 2])
Model 0, input size: torch.Size([15, 5]), output size: torch.Size([15, 2])
Model 1, input size: torch.Size([30, 5]), output size: torch.Size([30, 2])
Model 1, input size: torch.Size([15, 5]), output size: torch.Size([15, 2])
```

可以看到，原本 90 容量的数据集，被分为了 45 和 45。由于 BS 为 30，因此 sample 了两次，第一次 30 个，第二次 15 个。

PT 自动将线程数设为了 1。而我通过 `nproc` 或 `htop` 发现我有 12 个核心，除以进程数 2，得到 6 线程/进程。因此我手动设 `num_threads` 为 6。

</p>
</details>

<details>
<summary><b>GLOO 后端 + MP 启动 + 模型读写方式</b></summary>
<p>

1. Gloo：PT 原生，CPU 支持完美，GPU 支持一般。
2. MP：自动分配进程，执行命令更简单。适合给其他人分享代码。

```python
"""
test.py
$ python test.py
"""
import os
import torch
import torch.distributed as dist
import torch.nn as nn
import torch.optim as optim
import torch.multiprocessing as mp
from torch.nn.parallel import DistributedDataParallel as DDP


def setup(rank, world_size):
    # 若非本机，要在init中指定init_method='tcp://10.1.1.20:23456'
    # 应该是随便设的通信端口
    os.environ['MASTER_ADDR'] = 'localhost'
    os.environ['MASTER_PORT'] = '12355'

    # initialize the process group
    # 这里用的是gloo
    dist.init_process_group("gloo", rank=rank, world_size=world_size)


class ToyModel(nn.Module):
    """任意模型"""
    def __init__(self):
        super(ToyModel, self).__init__()
        self.net1 = nn.Linear(10, 10)
        self.relu = nn.ReLU()
        self.net2 = nn.Linear(10, 5)

    def forward(self, x):
        return self.net2(self.relu(self.net1(x)))


def cleanup():
    dist.destroy_process_group()


def demo_basic(rank, world_size):
    """不涉及模型保存和加载，只展示基本DDP操作"""
    # 显示顺序无所谓，是同时跑的
    print(f"Running basic DDP example on rank {rank}.")

    # 每个进程使用DDP前必须的操作
    setup(rank, world_size)

    # 将模型转移到对应设备上（注意rank），然后用DDP wrap
    model = ToyModel().to(rank)
    ddp_model = DDP(model, device_ids=[rank])

    # 注意以下引用的是DDP model
    loss_fn = nn.MSELoss()
    optimizer = optim.SGD(ddp_model.parameters(), lr=0.001)

    optimizer.zero_grad()
    outputs = ddp_model(torch.randn(20, 10))
    labels = torch.randn(20, 5).to(rank)
    # 当backward返回时，梯度已经自动同步了
    loss_fn(outputs, labels).backward()
    optimizer.step()

    # 最后结束掉
    cleanup()


def demo_checkpoint(rank, world_size):
    """展示DDP模型的保存和加载，以及基本DDP操作"""
    # 显示顺序无所谓，是同时跑的
    print(f"Running DDP checkpoint example on rank {rank}.")

    # 每个进程使用DDP前必须的操作
    setup(rank, world_size)

    # 将模型转移到对应设备上（注意rank），然后用DDP wrap
    model = ToyModel().to(rank)
    ddp_model = DDP(model, device_ids=[rank])

    # 注意以下引用的是DDP model
    loss_fn = nn.MSELoss()
    optimizer = optim.SGD(ddp_model.parameters(), lr=0.001)

    # 只需要在一个进程中保存模型！这里选择的是rank=0进程
    CHECKPOINT_PATH = "model.checkpoint"
    if rank == 0:
        torch.save(ddp_model.state_dict(), CHECKPOINT_PATH)

    # 相当于一个同步检查点。只有当所有进程都到达此处时，才会继续
    # 这里为什么要用呢？因为要保证process 0保存模型完毕，process 1之后才能加载！
    dist.barrier()

    # 注意map的设置
    # 由于是在rank=0进程保存的模型，因此是从rank=0映射到rank=0,1,...
    # 不map的后果：https://www.zhihu.com/question/67209417/answer/866488638
    map_location = {'cuda:%d' % 0: 'cuda:%d' % rank}
    ddp_model.load_state_dict(
        torch.load(CHECKPOINT_PATH, map_location=map_location))

    optimizer.zero_grad()
    outputs = ddp_model(torch.randn(20, 10))
    labels = torch.randn(20, 5).to(rank)
    loss_fn = nn.MSELoss()
    # 当backward返回时，梯度已经自动同步了
    # 因此，不再需要dist.barrier()来确保同步
    loss_fn(outputs, labels).backward()
    optimizer.step()

    if rank == 0:
        os.remove(CHECKPOINT_PATH)  # 个人感觉可做可不做

    # 最后结束掉
    cleanup()


def run_demo(demo_fn, world_size):
    # 通过spawn创建多个子进程，调用join使其一起工作
    mp.spawn(demo_fn,
             args=(world_size,),
             nprocs=world_size,
             join=True)


if __name__ == "__main__":
    n_gpus = torch.cuda.device_count()
    print(f"got {n_gpus} gpus.")
    run_demo(demo_basic, 2)
    run_demo(demo_checkpoint, 2)
```

输出：

```bash
got 2 gpus.
Running basic DDP example on rank 0.
Running basic DDP example on rank 1.
Running DDP checkpoint example on rank 1.
Running DDP checkpoint example on rank 0.
```

</p>
</details>

<details>
<summary><b>原理</b></summary>
<p>

有时，单卡显存不足，我们需要多卡才能跑得动；有时，虽然单卡显存足够，但单卡利用率饱和，多卡可以提高运算速度。

有以下几种方式：

**（1）DataParallel**

- 适用于单机多卡。
- 每次 forward 都要复制模型。
- 单进程，受限于 GIL 竞争。
- 代码改动最少，但效率低。

**（2）DistributedDataParallel**

强烈推荐。

- 适用于单机多卡和多机多卡。
- 额外需要 `init_process_group` 操作。
- 多进程并行，不受 GIL 影响。
- 在 DDP 建立时单次广播模型，无须每次 forward 广播。
- 可以和 model parallel 组合使用，即每个 process 单独执行 model parallel。参见官方教程最后。

其他例如 model parallel，参见[教程](https://pytorch.org/tutorials/intermediate/model_parallel_tutorial.html)。

`torch.distributed` 主要有3个组件，参见[文档](https://pytorch.org/docs/master/notes/ddp.html)。我们主要用 `Distributed Data-Parallel Training (DDP)`。

DDP 原理：模型在 DDP 建立之初分发到各进程。每个进程输入各自的数据进行 forward。backward 后，计算梯度，分发至各进程。各进程分别进行相同的参数更新，从而保证各模型一致性。

注意：

1. 不同进程之间不可共享 GPU。即一块卡只能用于一个进程。
2. 要合理分配各进程的负荷，让它们的完成时间接近。否则，要指定 `init_process_group`中的 `timeout`，避免超时。
3. 只需要在一个进程中保存模型，但加载时要分发到所有进程。方法：指定好 `map_location` 参数。若未指定，模型会先导入到 CPU，然后被分发到所有进程。此时，所有进程将共享同样的设备。

主要参考：

- [官方教程](https://pytorch.org/tutorials/intermediate/ddp_tutorial.html)
- [知乎](https://zhuanlan.zhihu.com/p/178402798)
- [知乎](https://zhuanlan.zhihu.com/p/76638962)

</p>
</details>
