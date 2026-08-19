---
layout: post
title: "PyTorch 基础讲义：从 Tensor 到完整训练循环"
date: 2026-08-19 00:00:00 +0800
categories: [PyTorch]
tags: [Python, 深度学习, Tensor, 自动求导, 训练循环]
---

# PyTorch 基础讲义：从 Tensor 到完整训练循环

> 适合对象：刚开始学习大模型或深度学习、会写基础 Python，但还没有独立跑通训练代码的学习者。
>
> 本文只聚焦六个核心组件：`Tensor`、自动求导、`Dataset`、`DataLoader`、`nn.Module` 和优化器。学完后，你应能读懂并写出一个小型监督学习训练循环，知道每一行代码在训练链路中的位置。

## 1. 先建立全局地图

PyTorch 训练不是一堆彼此无关的 API。最常见的监督学习流程可概括为：

```text
原始样本
  -> Dataset：定义“第 i 个样本是什么”
  -> DataLoader：按 batch 取样、打乱、并行加载
  -> Tensor：承载一个 batch 的数值数据
  -> nn.Module：前向计算，得到预测值
  -> 损失函数：衡量预测和答案的差距
  -> autograd：沿计算图计算梯度
  -> optimizer：按照梯度更新模型参数
  -> 下一轮 batch / epoch
```

可以把它类比为做题与订正：`Dataset` 是题库，`DataLoader` 是每次发给你的题目批次，`nn.Module` 是当前解题方法，损失函数是判卷结果，自动求导算出“每个解题步骤该往哪个方向调整”，优化器负责真正修改方法。训练的目标不是让某一个 batch 的分数最高，而是在未见过的数据上也能表现好。

### 1.1 三个必须分清的概念

- **样本（sample）**：一条数据，例如一张图片、一句文本或一个用户行为记录。
- **特征（feature / x）**：给模型的输入。例如图片像素、文本 token 的编号或表格的数值列。
- **标签（label / y）**：希望模型给出的正确答案。例如“猫/狗”、情感类别或房价。
- **batch**：一次一起送入模型的多个样本。`batch_size=32` 表示每次处理 32 条样本。
- **epoch**：训练集被完整遍历一遍。数据集有 1,000 条、batch 大小为 100 时，一个 epoch 约有 10 个 batch。

后文用二分类问题演示：输入是一组四维数字，标签是 0 或 1。真实业务中，输入可能是图像、token 或向量，但训练链路相同。

## 2. 环境与第一段检查代码

### 2.1 安装

请先准备 Python 3.10 或更新版本，并在虚拟环境中安装 PyTorch。CPU 学习版可直接执行：

```bash
pip install torch
```

若要使用 NVIDIA GPU，请以 [PyTorch 官方安装页](https://pytorch.org/get-started/locally/) 针对你的 CUDA 驱动和操作系统生成的命令为准。不要随意把网上某条 CUDA 安装命令复制到本机，因为驱动、CUDA 与 PyTorch 二进制版本需要匹配。

### 2.2 验证安装和设备

```python
import torch

print(torch.__version__)
print(torch.cuda.is_available())
print(torch.cuda.get_device_name(0) if torch.cuda.is_available() else "使用 CPU")
```

输出 `True` 只表示 PyTorch 能访问 CUDA，并不表示所有张量和模型已经在 GPU 上。设备归属需要由代码显式控制。

```python
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
print(device)
```

后文会把模型、输入和标签都移动到同一个 `device`。这是避免“Expected all tensors to be on the same device”报错的基本规则。

## 3. Tensor：深度学习中的数值容器

`torch.Tensor` 是 PyTorch 的核心数据结构。它可以理解为带有**形状、数据类型和设备信息**的多维数组，并支持 GPU 计算和自动求导。

### 3.1 创建 Tensor

```python
import torch

scalar = torch.tensor(3.14)                  # 0 维，标量
vector = torch.tensor([1.0, 2.0, 3.0])       # 1 维，向量
matrix = torch.tensor([[1, 2], [3, 4]])      # 2 维，矩阵
zeros = torch.zeros(2, 3)                    # 全 0，形状为 (2, 3)
ones = torch.ones(2, 3)                      # 全 1
random_x = torch.randn(2, 3)                 # 标准正态分布随机数
sequence = torch.arange(0, 10, 2)            # 0, 2, 4, 6, 8
```

`torch.tensor(...)` 通常用于把已有 Python 数据转换成 Tensor；`zeros`、`randn` 等工厂函数更适合按形状创建训练数据或参数。

### 3.2 最先看 `shape`、`dtype` 和 `device`

```python
x = torch.randn(4, 3, dtype=torch.float32)
print(x.shape)   # torch.Size([4, 3])
print(x.dtype)   # torch.float32
print(x.device)  # cpu
print(x.ndim)    # 2
```

- `shape`：每个维度的长度。上例 `(4, 3)` 可读成 4 行、每行 3 个数。
- `dtype`：数据类型。神经网络输入和可学习参数通常用 `float32`；类别标签通常用 `int64`，也就是 `torch.long`。
- `device`：数据所在设备，例如 `cpu` 或 `cuda:0`。

在训练报错时，请先打印这三个属性。大量问题本质上是形状、类型或设备不一致。

### 3.3 形状约定：batch 维度通常在最前面

大多数模型默认第一维是 batch：

```python
images = torch.randn(32, 3, 224, 224)  # 32 张 RGB 图像，通道、高、宽
features = torch.randn(32, 10)         # 32 条样本，每条 10 个特征
logits = torch.randn(32, 5)            # 32 条样本，对 5 个类别的未归一化得分
```

对于表格数据，常见形状是 `(batch_size, feature_count)`；对图像常见为 `(batch_size, channels, height, width)`；对语言模型的 token 编号常见为 `(batch_size, sequence_length)`。

### 3.4 索引、切片与变形

```python
x = torch.arange(12).reshape(3, 4)
print(x)
# tensor([[ 0,  1,  2,  3],
#         [ 4,  5,  6,  7],
#         [ 8,  9, 10, 11]])

first_row = x[0]         # 形状 (4,)
last_column = x[:, -1]   # 形状 (3,)
block = x[0:2, 1:3]      # 形状 (2, 2)
flat = x.reshape(-1)     # 形状 (12,)
```

`reshape` 只改变“怎么看这些元素”，元素总数必须一致。`-1` 让 PyTorch 自动推断该维长度。模型前后的维度不匹配，是初学者最常见的问题之一；每次变形前后都应该确认 `shape`。

### 3.5 常用运算与广播

```python
x = torch.tensor([[1.0, 2.0], [3.0, 4.0]])
y = torch.tensor([10.0, 20.0])

print(x + y)     # y 被广播到每一行
print(x * 2)     # 标量逐元素乘法
print(x.mean())  # 所有元素的平均值
print(x.sum(dim=0))  # 沿第 0 维求和，得到每一列的和
print(x @ x.T)   # 矩阵乘法
```

**逐元素乘法** `x * y` 与**矩阵乘法** `x @ y` 含义不同。线性层核心使用矩阵乘法；激活函数、掩码和损失的很多操作则是逐元素计算。

广播（broadcasting）会自动扩展长度为 1 的维度，写法简洁但也可能悄悄产生错误结果。例如标签形状为 `(32,)`、预测形状为 `(32, 1)` 时，不要依赖广播“凑合能算”，而要先弄清损失函数期望的形状。

### 3.6 与 NumPy、Python 标量互转

```python
import numpy as np

array = np.array([[1, 2], [3, 4]], dtype=np.float32)
x = torch.from_numpy(array)  # 通常与 NumPy 数组共享内存
array[0, 0] = 99
print(x[0, 0])               # 可能也变成 99

value = x[0, 0].item()       # 单元素 Tensor 转 Python 数值
cpu_array = x.cpu().numpy()  # GPU Tensor 先移到 CPU 再转 NumPy
```

训练循环里用于日志的 `loss.item()` 很重要：它把单元素 Tensor 变成普通 Python 浮点数，避免无意间长期保留计算图。

### 3.7 Tensor 的常见坑

- `torch.tensor([1, 2])` 默认通常是整数类型，不能直接作为大多数网络层的浮点输入。需要时写 `torch.tensor([1, 2], dtype=torch.float32)`。
- 对 GPU Tensor 调用 `.numpy()` 会报错，应先 `.detach().cpu().numpy()`。
- 不要频繁在训练循环中把 Tensor 转成 Python 列表或 NumPy；这会引入设备同步和性能损耗。
- `view` 也可变形，但要求底层内存连续；不确定时优先用更稳妥的 `reshape`。

## 4. 自动求导：梯度从哪里来

训练的本质是让损失函数变小。以一个参数 `w` 为例，如果损失对 `w` 的导数（梯度）是正数，通常应把 `w` 往小调；若梯度为负数，通常应把 `w` 往大调。这就是梯度下降。

### 4.1 最小示例

```python
import torch

x = torch.tensor(3.0)
w = torch.tensor(2.0, requires_grad=True)
b = torch.tensor(1.0, requires_grad=True)

y = w * x + b
loss = (y - 10.0) ** 2
loss.backward()

print(loss.item())  # 9.0
print(w.grad)       # d(loss) / d(w)，这里是 -18
print(b.grad)       # d(loss) / d(b)，这里是 -6
```

当 `requires_grad=True` 时，PyTorch 会记录以该 Tensor 为起点的可微分运算，建立计算图。`loss.backward()` 使用链式法则从损失反向传播，计算每个叶子参数的梯度，并写入 `.grad`。

这里：

```text
y = 2 * 3 + 1 = 7
loss = (7 - 10)^2 = 9

对 w 的梯度 = 2 * (y - 10) * x = 2 * (-3) * 3 = -18
```

负梯度意味着若希望损失下降，`w` 应略微增大。实际更新由优化器完成。

### 4.2 计算图、叶子 Tensor 与 `.grad`

```python
w = torch.tensor(2.0, requires_grad=True)  # 叶子 Tensor：用户直接创建
z = w * 3                                  # 非叶子 Tensor：运算结果
loss = z ** 2
loss.backward()

print(w.grad)  # 有梯度
print(z.grad)  # 默认通常为 None
```

默认只保留叶子 Tensor 的 `.grad`，因为训练真正需要更新的是模型参数。中间结果可通过 `z.retain_grad()` 保留梯度以便调试，但不应作为日常习惯。

### 4.3 梯度会累加：每一步都要清零

```python
w = torch.tensor(1.0, requires_grad=True)

for _ in range(2):
    loss = (w * 2 - 4) ** 2
    loss.backward()
    print(w.grad)
```

第二次输出不是“新梯度”，而是两次梯度之和。这样设计是为了支持梯度累积。标准训练顺序中必须调用 `optimizer.zero_grad()`，否则参数更新会使用历史 batch 的累积梯度。

### 4.4 不需要梯度时使用 `torch.no_grad()`

验证、推理、打印预测结果时，不需要构建反向传播图：

```python
model.eval()
with torch.no_grad():
    logits = model(features)
    predictions = logits.argmax(dim=1)
```

这会减少内存占用和计算开销。注意：`model.eval()` 与 `torch.no_grad()` 作用不同，前者切换 Dropout、BatchNorm 等层的行为，后者关闭梯度记录；验证时通常二者都要使用。

### 4.5 `detach()`：切断计算图

```python
prediction = model(features)
for_log = prediction.detach().cpu()
```

`detach()` 得到一个不再追踪当前计算图的 Tensor。常用于记录指标、转 NumPy 或构造不希望反传的目标。不要用 `.data` 绕过自动求导；它可能静默破坏计算图，应使用 `detach()` 或 `torch.no_grad()`。

## 5. Dataset：定义一条样本是什么

`torch.utils.data.Dataset` 是一个抽象接口，负责告诉 PyTorch：数据集有多少条，以及如何获取第 `index` 条样本。自定义 Dataset 通常实现两个方法：

- `__len__`：返回样本数量；
- `__getitem__(index)`：返回一个样本，通常是 `(feature, label)`。

### 5.1 最小自定义 Dataset

```python
from torch.utils.data import Dataset
import torch

class ToyClassificationDataset(Dataset):
    """保存内存中二分类特征与标签的数据集。"""

    def __init__(self, features: torch.Tensor, labels: torch.Tensor) -> None:
        self.features = features.float()
        self.labels = labels.long()

    def __len__(self) -> int:
        return len(self.features)

    def __getitem__(self, index: int) -> tuple[torch.Tensor, torch.Tensor]:
        return self.features[index], self.labels[index]
```

`__getitem__` 必须对同一个 `index` 返回对应的输入与标签。特征转为 `float()`、用于 `CrossEntropyLoss` 的类别标签转为 `long()`，是这里有意处理的类型约束。

真实项目的 `__getitem__` 常在这里完成：读取图像、读取文本、tokenize、数值归一化、数据增强或无效样本过滤。要避免在这里做很重的网络请求或重复加载整份数据，因为它会让训练数据管线成为瓶颈。

### 5.2 `TensorDataset`：已有 Tensor 时的简化写法

如果数据已经整齐地放在 Tensor 中，无须重复定义类：

```python
from torch.utils.data import TensorDataset

features = torch.randn(100, 4)
labels = torch.randint(0, 2, (100,))
dataset = TensorDataset(features, labels)
```

`TensorDataset` 是适合练习和内存数据的现成实现。自定义 Dataset 的价值在于处理文件、复杂样本结构或预处理逻辑。

### 5.3 数据集切分：训练、验证、测试

- **训练集**：用于更新参数。
- **验证集**：用于挑选超参数、观察是否过拟合、决定保存哪一个模型。
- **测试集**：只在最终评估时使用，不应参与调参。

数据泄漏常来自先对全量数据拟合标准化器、再切分，或反复根据测试集分数调参。正确做法是只在训练集拟合预处理统计量，再把同一套变换用于验证和测试集。

## 6. DataLoader：按 batch 可靠地送数据

`DataLoader` 包装 Dataset，负责迭代、分 batch、可选随机打乱和多进程加载。

```python
from torch.utils.data import DataLoader

loader = DataLoader(
    dataset,
    batch_size=32,
    shuffle=True,
    num_workers=0,
)

for batch_features, batch_labels in loader:
    print(batch_features.shape, batch_labels.shape)
    break
```

### 6.1 关键参数

- `batch_size`：每次送入模型的样本数。更大通常吞吐更高，但更占显存；不是越大越好。
- `shuffle=True`：每个 epoch 打乱训练数据，避免模型被原始排列顺序误导。训练集通常开启，验证/测试集通常关闭。
- `num_workers`：额外数据加载进程数。Windows 和初学环境建议先用 `0`，确认流程正确后再调大。若使用非 0 值，应把启动训练的代码放入 `if __name__ == "__main__":` 块中。
- `drop_last=True`：丢弃最后一个不足 `batch_size` 的 batch。只有当模型或分布式训练确实要求固定 batch 大小时才考虑；普通训练默认不丢。
- `pin_memory=True`：训练使用 CUDA 且 CPU 到 GPU 拷贝成为瓶颈时可尝试开启；在 CPU 训练时没有必要。

### 6.2 一个 batch 的形状检查

```python
for features, labels in loader:
    print(features.shape)  # 例如 torch.Size([32, 4])
    print(labels.shape)    # 例如 torch.Size([32])
    break
```

请把第一次 batch 的形状检查当作固定习惯。若模型期望 `(batch, 4)`，却收到 `(4,)`，通常是漏掉了 batch 维度；若标签多出一维，应在理解损失函数要求后再使用 `squeeze`，不要盲目压缩维度。

## 7. nn.Module：把可学习计算组织为模型

`nn.Module` 是 PyTorch 模型与层的基类。子类一般在 `__init__` 中声明层和可学习组件，在 `forward` 中定义输入如何流向输出。

### 7.1 一个两层分类器

```python
import torch
from torch import nn

class BinaryClassifier(nn.Module):
    """接收四维特征并输出两个类别 logits 的小型分类网络。"""

    def __init__(self, input_dim: int = 4) -> None:
        super().__init__()
        self.network = nn.Sequential(
            nn.Linear(input_dim, 16),
            nn.ReLU(),
            nn.Linear(16, 2),
        )

    def forward(self, features: torch.Tensor) -> torch.Tensor:
        return self.network(features)
```

`nn.Linear(4, 16)` 内部包含权重和偏置，并会自动注册为模型参数。`model.parameters()` 会递归找到这些参数，优化器据此更新模型。

```python
model = BinaryClassifier()
print(sum(parameter.numel() for parameter in model.parameters()))
print(model)
```

### 7.2 `forward` 与调用模型

推荐调用 `logits = model(features)`，而不是直接调用 `model.forward(features)`。前一种写法会经过 `nn.Module` 的钩子等机制，是框架的规范入口。

模型最后输出的是两个 **logits**，即每个类别的未归一化分数，例如 `[2.3, -0.7]`。搭配 `nn.CrossEntropyLoss` 时，模型末尾**不要**再接 `Softmax`，因为该损失函数内部已包含数值稳定的 `log_softmax` 计算。

### 7.3 训练模式与评估模式

```python
model.train()  # 训练前调用
# 执行训练 batch

model.eval()   # 验证或推理前调用
# 配合 torch.no_grad() 执行验证
```

对只有 `Linear` 和 `ReLU` 的本例，两者看起来结果相同；但加入 `Dropout`、`BatchNorm` 后，模式不同会改变行为。把这两个调用写成习惯，模型变复杂后才能避免隐蔽错误。

## 8. 损失函数与优化器：如何告诉模型“该怎么改”

### 8.1 先选对损失函数

常用组合如下：

- 单标签多分类：模型输出 `(batch, classes)` 的 logits，标签为 `(batch,)` 的 `long` 类别编号，使用 `nn.CrossEntropyLoss()`。
- 回归：模型和目标通常形状相同，使用 `nn.MSELoss()` 或 `nn.L1Loss()`。
- 二分类且模型只输出一个 logit：输出与标签通常是 `(batch,)` 或 `(batch, 1)` 的浮点数，使用 `nn.BCEWithLogitsLoss()`。

“二分类”并不只对应一种写法。本文采取两个 logits + `CrossEntropyLoss`，因此标签是 `0` 或 `1` 的整数；不要把 one-hot 标签直接传给这种基础用法。

```python
criterion = nn.CrossEntropyLoss()
logits = torch.tensor([[2.0, -1.0], [-0.5, 1.2]])
labels = torch.tensor([0, 1], dtype=torch.long)
loss = criterion(logits, labels)
```

### 8.2 优化器是什么

优化器读取每个参数的 `.grad`，并按特定规则更新参数。最常用的入门选择是 AdamW：

```python
optimizer = torch.optim.AdamW(model.parameters(), lr=1e-3)
```

- `model.parameters()`：交给优化器的可学习参数。
- `lr`（learning rate，学习率）：每一步改多大。过大可能震荡甚至 NaN，过小可能训练极慢。
- `weight_decay`：常用的正则化项，有助于抑制部分过拟合；不是所有问题都必须设置。

SGD、Adam 与 AdamW 都有适用场景。学习阶段先理解“使用梯度更新参数”的共同机制，再按任务选择算法。对 Transformer，AdamW 是很常见的起点。

### 8.3 标准训练四步

每个 batch 的关键顺序必须记住：

```python
optimizer.zero_grad()     # 1. 清除上一个 batch 累积的梯度
logits = model(features)  # 2. 前向计算预测
loss = criterion(logits, labels)
loss.backward()           # 3. 反向传播，把梯度写入参数 .grad
optimizer.step()          # 4. 使用梯度更新参数
```

如果把 `optimizer.step()` 放在 `loss.backward()` 前，参数没有新梯度可用；若漏掉 `zero_grad()`，梯度会不断累加。初学时不要把这四步压成一行，清晰的顺序比短代码更重要。

## 9. 可直接运行：从数据到验证的完整示例

以下代码不依赖下载数据集。它创建一个带清晰规律的四维二分类数据集，随后完成切分、加载、训练、验证和保存最佳 checkpoint。正常情况下，验证准确率会逐渐提高；不同电脑的具体数字可能略有差异。

```python
import torch
from torch import nn
from torch.utils.data import DataLoader, Dataset, random_split


class SyntheticBinaryDataset(Dataset):
    """生成可用于演示训练流程的四维二分类数据集。

    标签由特征的线性组合和少量噪声决定，因此小型线性网络可以学习到规律。
    """

    def __init__(self, sample_count: int = 1_000) -> None:
        generator = torch.Generator().manual_seed(42)
        self.features = torch.randn(sample_count, 4, generator=generator)
        score = (
            1.2 * self.features[:, 0]
            - 0.8 * self.features[:, 1]
            + 0.5 * self.features[:, 2]
            + 0.2 * torch.randn(sample_count, generator=generator)
        )
        self.labels = (score > 0).long()

    def __len__(self) -> int:
        return len(self.features)

    def __getitem__(self, index: int) -> tuple[torch.Tensor, torch.Tensor]:
        return self.features[index], self.labels[index]


class BinaryClassifier(nn.Module):
    """将四维特征映射为两个类别 logits 的分类器。"""

    def __init__(self, input_dim: int = 4) -> None:
        super().__init__()
        self.network = nn.Sequential(
            nn.Linear(input_dim, 16),
            nn.ReLU(),
            nn.Linear(16, 2),
        )

    def forward(self, features: torch.Tensor) -> torch.Tensor:
        return self.network(features)


def evaluate(
    model: nn.Module,
    loader: DataLoader,
    criterion: nn.Module,
    device: torch.device,
) -> tuple[float, float]:
    """在验证集上计算平均损失与准确率。"""
    model.eval()
    total_loss = 0.0
    correct_count = 0
    sample_count = 0

    with torch.no_grad():
        for features, labels in loader:
            features = features.to(device)
            labels = labels.to(device)
            logits = model(features)
            loss = criterion(logits, labels)

            total_loss += loss.item() * labels.size(0)
            predictions = logits.argmax(dim=1)
            correct_count += (predictions == labels).sum().item()
            sample_count += labels.size(0)

    return total_loss / sample_count, correct_count / sample_count


def main() -> None:
    """执行数据准备、训练、验证和最佳模型保存。"""
    torch.manual_seed(42)
    device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

    dataset = SyntheticBinaryDataset()
    train_dataset, validation_dataset = random_split(
        dataset,
        [800, 200],
        generator=torch.Generator().manual_seed(42),
    )
    train_loader = DataLoader(train_dataset, batch_size=32, shuffle=True)
    validation_loader = DataLoader(validation_dataset, batch_size=64, shuffle=False)

    model = BinaryClassifier().to(device)
    criterion = nn.CrossEntropyLoss()
    optimizer = torch.optim.AdamW(model.parameters(), lr=1e-3)

    best_validation_accuracy = 0.0
    for epoch in range(1, 31):
        model.train()
        running_loss = 0.0

        for features, labels in train_loader:
            features = features.to(device)
            labels = labels.to(device)

            optimizer.zero_grad()
            logits = model(features)
            loss = criterion(logits, labels)
            loss.backward()
            optimizer.step()

            running_loss += loss.item() * labels.size(0)

        train_loss = running_loss / len(train_dataset)
        validation_loss, validation_accuracy = evaluate(
            model,
            validation_loader,
            criterion,
            device,
        )
        print(
            f"Epoch {epoch:02d} | "
            f"train_loss={train_loss:.4f} | "
            f"val_loss={validation_loss:.4f} | "
            f"val_acc={validation_accuracy:.2%}"
        )

        if validation_accuracy > best_validation_accuracy:
            best_validation_accuracy = validation_accuracy
            torch.save(
                {
                    "model_state_dict": model.state_dict(),
                    "optimizer_state_dict": optimizer.state_dict(),
                    "epoch": epoch,
                    "validation_accuracy": validation_accuracy,
                },
                "best_binary_classifier.pt",
            )

    print(f"最佳验证准确率：{best_validation_accuracy:.2%}")


if __name__ == "__main__":
    main()
```

### 9.1 逐段理解这份代码

1. `SyntheticBinaryDataset` 是题库。`__getitem__` 每次只返回一条 `(features, label)`。
2. `random_split` 把同一数据集随机分为 800 条训练数据和 200 条验证数据。生产项目中通常应按业务时间、用户或分层规则切分，而不仅是随机切分。
3. `train_loader` 开启 `shuffle=True`；`validation_loader` 不打乱，便于稳定复现验证过程。
4. `model.to(device)` 将参数移至计算设备；每个 batch 的特征和标签也必须 `.to(device)`。
5. 训练阶段使用 `model.train()`，并执行四步更新顺序。
6. `evaluate` 使用 `model.eval()` 和 `torch.no_grad()`，保证验证不更新参数、不记录梯度。
7. 准确率统计中，`argmax(dim=1)` 从两个 logits 中取分数更高的类别；`sum().item()` 得到预测正确的样本数量。
8. checkpoint 保存的不只是模型权重，还保存优化器状态、epoch 与验证指标。恢复训练时优化器状态也很重要，尤其是 AdamW 一类维护内部动量的算法。

### 9.2 恢复模型与推理

```python
checkpoint = torch.load("best_binary_classifier.pt", map_location=device)
model = BinaryClassifier().to(device)
model.load_state_dict(checkpoint["model_state_dict"])
model.eval()

one_sample = torch.tensor([[0.1, -0.2, 0.3, 0.0]], dtype=torch.float32).to(device)
with torch.no_grad():
    logits = model(one_sample)
    predicted_class = logits.argmax(dim=1).item()

print(predicted_class)
```

加载权重前，模型结构必须与保存时一致。`map_location=device` 让你可以在没有 GPU 的机器上加载曾在 GPU 上训练的 checkpoint。

## 10. 如何验证训练真的在发生

“脚本没有报错”不等于模型在学习。至少检查以下信号：

- 训练损失整体是否下降。单个 batch 波动正常，但多轮趋势应可解释。
- 验证损失和准确率是否改善。训练集变好、验证集变差可能表示过拟合。
- 参数是否真的变化。可在一次 `optimizer.step()` 前后比较某个参数。
- 小数据能否过拟合。拿 16 或 32 个样本训练足够久，模型应能几乎记住它们；如果做不到，优先检查训练链路，而不是急着换更大模型。

```python
before = model.network[0].weight.detach().clone()
# 执行一次完整的 backward 和 optimizer.step()
after = model.network[0].weight.detach().clone()
print(torch.allclose(before, after))  # 训练正常时通常为 False
```

### 10.1 最小过拟合测试为什么重要

当真实项目不收敛时，先使用极少量样本、关闭复杂数据增强、固定随机种子。若模型连几十条数据都记不住，问题多半在数据、标签、损失函数、梯度或参数更新，而不是“模型容量不够”。这是定位训练问题的高收益手段。

## 11. 高发报错与排查顺序

### 11.1 维度不匹配

典型错误：`mat1 and mat2 shapes cannot be multiplied`。

先打印：

```python
print(features.shape)
print(model)
```

若 `nn.Linear(4, 16)` 收到的最后一维不是 4，就会失败。图像送入全连接层前经常需要展平；序列数据通常不能随意展平，应先理解模型结构预期。

### 11.2 标签类型或形状错误

`CrossEntropyLoss` 的基础要求是：

```text
logits: (batch_size, number_of_classes)，float
labels: (batch_size,)，long，值范围为 [0, number_of_classes - 1]
```

因此出现 `expected scalar type Long but found Float` 时，应检查标签是否被错误地转成了浮点数；出现 `Target 2 is out of bounds` 时，应检查类别数和标签编号是否对应。

### 11.3 CPU 与 GPU 混用

典型错误：`Expected all tensors to be on the same device`。

检查模型、输入、标签和损失函数涉及的其他 Tensor：

```python
print(next(model.parameters()).device)
print(features.device)
print(labels.device)
```

不要只把模型移到 GPU，却忘记把 batch 移过去。

### 11.4 `loss.backward()` 报“不需要梯度”

常见原因：

- 在 `torch.no_grad()` 作用域内执行了训练；
- 对模型输出过早调用了 `.detach()` 或 `.item()`；
- 使用了不可微操作并把它当成损失；
- 优化器拿到的不是模型的真实参数。

检查 `loss.requires_grad` 是否为 `True`，并确认训练阶段没有误用 `model.eval()` 加 `torch.no_grad()` 的推理模板。

### 11.5 梯度不更新或训练不收敛

按以下顺序排查：

1. 验证标签是否与输入一一对应，类别编号是否正确。
2. 打印一个 batch 的 `shape`、`dtype`、`device`、标签取值范围。
3. 确认每个 batch 都有 `zero_grad -> forward -> loss -> backward -> step`。
4. 检查 `any(parameter.grad is None for parameter in model.parameters())`。
5. 对极小数据集做过拟合测试。
6. 再调整学习率、模型容量、归一化和 batch 大小。

不要一看到不收敛就连续叠加更多层、更多 epoch 和更多优化技巧；先证明最小链路正确。

### 11.6 NaN、Inf 与 OOM

- **NaN/Inf**：先检查原始输入、标签、损失和梯度是否已经出现非有限值：`torch.isfinite(tensor).all()`。再尝试降低学习率，必要时使用梯度裁剪。
- **梯度裁剪**：对 RNN、Transformer 或训练不稳定任务，常在 `loss.backward()` 后、`optimizer.step()` 前调用 `torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)`。
- **OOM（显存不足）**：先减小 batch size 和序列长度，确认没有保存带计算图的 Tensor 到列表，再考虑混合精度、梯度累积或更小模型。不要先靠清空缓存掩盖持续增长的内存泄漏。

## 12. 进阶概念的正确学习顺序

本文的六个组件是后续能力的地基。建议按下面顺序扩展：

1. 用本讲义示例跑通 CPU 训练与验证，能解释四步更新顺序。
2. 换成公开小数据集，例如 MNIST，练习图像 Tensor 形状与卷积网络。
3. 手动实现训练/验证指标记录、checkpoint 保存和恢复训练。
4. 学习学习率调度器、梯度裁剪、早停与数据增强，并为每个技巧建立可观察的对比实验。
5. 再进入 Transformer、tokenizer、注意力、混合精度、梯度累积和分布式训练。

对于大模型开发，后续经常会看到：token ids 是整数 Tensor、Embedding 输出是浮点 Tensor、attention mask 是形状可广播的 Tensor、模型是 `nn.Module`、训练器内部仍在做同样的 `zero_grad -> backward -> step`。高层框架帮你封装了细节，但不会改变这条核心链路。

## 13. 自测清单与练习

在进入下一课前，建议能不看资料回答：

- `Tensor.shape` 的第一维在训练中通常表示什么？
- 为什么 `CrossEntropyLoss` 前通常不写 `Softmax`？
- 为什么每个训练 batch 都要执行 `optimizer.zero_grad()`？
- `model.train()`、`model.eval()` 和 `torch.no_grad()` 分别解决什么问题？
- `Dataset` 与 `DataLoader` 的职责边界是什么？
- 为什么模型、features、labels 必须在同一 device 上？
- checkpoint 为什么建议同时保存模型和优化器状态？

可以按难度完成这些练习：

1. 把完整示例的 `batch_size` 分别改为 1、32、256，记录训练速度和验证准确率变化。
2. 把 `BinaryClassifier` 改成只有一层 `nn.Linear(4, 2)`，比较效果。
3. 故意删掉 `optimizer.zero_grad()`，观察损失和梯度，再恢复它。
4. 在训练集只有 32 条样本时训练 200 个 epoch，验证模型是否能过拟合。
5. 把 checkpoint 加载逻辑补进训练脚本，实现从指定 epoch 继续训练。
6. 使用自己的一个 CSV 或文本分类小数据集，先只保证数据形状、标签类型、训练循环完全正确，再追求指标。

## 14. 本讲义的边界与下一步

本文刻意没有展开 Transformer 原理、混合精度、分布式训练、数据清洗策略和模型评测体系。这些内容都重要，但在你能独立追踪一批数据从 `Dataset` 进入模型、经反向传播更新参数之前，过早堆叠它们会增加理解负担。

下一步最值得做的不是背 API，而是亲自运行完整示例，并做一次“修改后验证”：改一个维度、改一个损失函数或改一个 batch 大小，预测会发生什么，再用输出检查自己的判断。能够用现象解释训练链路，才算真正掌握了基础。