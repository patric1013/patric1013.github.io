---
title: "AI Infra 第2章笔记（二）：GEMM 分块与概率论"
published: 2026-11-03T20:00:00
description: "这一篇前后两半看似无关，其实是同一个问题的两面：GEMM 分块回答“同一个公式怎样算得更快”（换计算顺序、复用数据、但浮点结果可能变），概率论回答“模型输出的数字到底是什么意思”（logits → 分布 → 似然 →…"
tags: ["AI Infra", "数学", "GEMM", "概率论"]
category: "AI Infra 学习笔记"
draft: false
---

这一篇前后两半看似无关，其实是同一个问题的两面：**GEMM 分块回答“同一个公式怎样算得更快”（换计算顺序、复用数据、但浮点结果可能变），概率论回答“模型输出的数字到底是什么意思”（logits → 分布 → 似然 → loss）。** 前半是后面 CUDA 模块的数学前提，后半是 Softmax 和交叉熵的前提。

> [!NOTE]
> 本篇对应原文 [AIInfraGuide · 第2章：数学基础](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/) 的 §3–4（作者 [caomaolufei](https://github.com/caomaolufei/AIInfraGuide)，MIT 许可）。每节开头的引用块摘自原文，文中的“原文 x.y”可以点击跳转到原文对应小节。问答是按第1章的提问思路整理的，代码输出均为实际运行结果。

## 3.1 为什么可以分块计算

> **📖 原文 [3.1 为什么可以分块计算](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#31-为什么可以分块计算)**
>
> 把矩阵沿 $M$、$N$ 和归约维 $K$ 切成块。以 $2\times2$ block 为例：
>
> $$
> \mathbf{A}=
> \begin{bmatrix}
> \mathbf{A}_{11} & \mathbf{A}_{12}\\
> \mathbf{A}_{21} & \mathbf{A}_{22}
> \end{bmatrix},\qquad
> \mathbf{B}=
> \begin{bmatrix}
> \mathbf{B}_{11} & \mathbf{B}_{12}\\
> \mathbf{B}_{21} & \mathbf{B}_{22}
> \end{bmatrix}
> $$
>
> 则：
>
> $$
> \mathbf{C}_{11}
> = \mathbf{A}_{11}\mathbf{B}_{11}
> + \mathbf{A}_{12}\mathbf{B}_{21}
> $$

### Q1. 分块计算为什么在数学上是对的？能不能用代码验证？

关键在于 $C_{ij} = \sum_k A_{ik}B_{kj}$ 这个求和可以**拆成几段分别求和再相加**。把 $K$ 维切成两段，求和就变成“前一段的和 + 后一段的和”，这就是 $\mathbf{C}_{11} = \mathbf{A}_{11}\mathbf{B}_{11} + \mathbf{A}_{12}\mathbf{B}_{21}$。$M$、$N$ 方向的切分更简单，只是把输出矩阵分成几个区域各自计算。

用一个 $4 \times 6$ 乘 $6 \times 4$ 的矩阵，按 2×2 的输出块、沿 $K$ 方向每次取 3 个来算：

```python
A, B = torch.randn(4, 6), torch.randn(6, 4)
C = torch.zeros(4, 4)
for i in (0, 2):              # 输出块的行起点（M 方向）
    for j in (0, 2):          # 输出块的列起点（N 方向）
        for k in (0, 3):      # 沿归约维 K 分段累加
            C[i:i+2, j:j+2] += A[i:i+2, k:k+3] @ B[k:k+3, j:j+2]
torch.allclose(C, A @ B, atol=1e-6)   # True
```

这三层循环就是 GPU GEMM Kernel 的骨架：外两层循环对应“每个线程块负责一个输出块”，最内层循环对应“沿 $K$ 方向一块一块地把数据搬进片上内存”。

### Q2. 原文的五步“取一小块、搬到片上、重复使用、沿 K 迭代、写回 HBM”，HBM、共享内存、寄存器分别是什么？

它们是 GPU 上从大到小、从慢到快的三层存储：

| 存储 | 容量量级（A100） | 速度 | 谁能访问 |
| --- | --- | --- | --- |
| HBM（显存） | 40～80 GB | 最慢，约 2 TB/s | 所有线程 |
| 共享内存（shared memory） | 每个 SM 约 100～200 KB | 快很多 | 同一个线程块内 |
| 寄存器 | 每个线程最多 255 个 | 最快 | 线程自己 |

类比第1章的知识：HBM 像硬盘上的大文件，共享内存像读进内存的一个 buffer，寄存器像函数里的局部变量。分块的意义就是“**一次从 HBM 搬一块进来，在片上反复用很多次，再搬下一块**”，用片上的快存储换掉大量的慢访问。具体的硬件细节在第5章 GPU 硬件和模块二的 CUDA 部分会展开。

### Q3. “它改变计算顺序和数据搬运，不改变目标公式”，那结果会不会变？

数学上不变，**浮点结果可能变**。因为分块改变了加法的顺序，而浮点加法不满足结合律。[原文 3.6](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#36-浮点分块结果为什么可能不同) 专门讲这个，见下面。

## 3.2 朴素 GEMM 的计算与访存

> **📖 原文 [3.2 朴素 GEMM 的计算与访存](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#32-朴素-gemm-的计算与访存)**
>
> 对于 $(M,K)@(K,N)$：
>
> - 输出元素数：$MN$；
> - 每个元素执行 $K$ 次乘加；
> - FLOPs 约为 $2MKN$。
>
> 朴素实现若每次乘加都从全局内存加载 $A_{ik}$ 和 $B_{kj}$，同一个元素会被大量重复读取。GEMM 的高性能来自数据复用：
>
> - $A$ 的一个元素被同一输出行的多个列复用；
> - $B$ 的一个元素被同一输出列的多个行复用；
> - 累加器在寄存器中复用 $K$ 次。

### Q4. 朴素 GEMM 中“同一个元素会被大量重复读取”，重复了多少次？

看 $A_{ik}$ 被谁用到：计算第 $i$ 行的所有输出 $C_{i1}, C_{i2}, \dots, C_{iN}$ 都要用它，一共 $N$ 次。同理 $B_{kj}$ 被第 $j$ 列的 $M$ 个输出用到。

如果每次用都从 HBM 读一遍，总读取量约为 $2MNK$ 个元素；而矩阵本身只有 $MK + KN$ 个元素。以 $M=N=K=4096$ 为例：

- 朴素读取：$2 \times 4096^3 \approx 1.4 \times 10^{11}$ 次；
- 实际需要：$2 \times 4096^2 \approx 3.4 \times 10^{7}$ 个元素。

相差 4096 倍，也就是每个元素平均被重复读了 4096 次。高性能 GEMM 的核心就是把这 4096 次中的绝大部分换成从片上读。

### Q5. “累加器在寄存器中复用 K 次”是什么意思？

计算一个 $C_{ij}$ 需要把 $K$ 个乘积加起来。朴素写法可能是每加一次就写回显存：

```python
for k in range(K):
    C[i, j] += A[i, k] * B[k, j]   # 如果 C 在显存里，每次都要读改写
```

高效写法是用一个局部变量（寄存器）累加，加完 $K$ 次再写一次：

```python
acc = 0.0                          # 寄存器里的累加器
for k in range(K):
    acc += A[i, k] * B[k, j]
C[i, j] = acc                      # 只写回一次
```

和第1章讲 Python 性能时“先在局部变量里算完再赋值”是同样的道理，只是在 GPU 上差距更大。

## 3.3 一个简化的算术强度估算

> **📖 原文 [3.3 一个简化的算术强度估算](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#33-一个简化的算术强度估算)**
>
> 算术强度定义为：
>
> $$
> \text{Arithmetic Intensity}
> = \frac{\text{FLOPs}}{\text{bytes transferred from target memory}}
> $$
>
> 理想情况下，FP16 GEMM 每个输入矩阵只从 HBM 读一次，输出写一次，粗略字节数为：
>
> $$
> 2(MK+KN+MN)
> $$

### Q6. 算术强度是什么？为什么要关心它？

算术强度 = **每从显存搬一个字节，能做多少次计算**（FLOPs / Byte）。它决定一个算子“卡在算力上还是卡在带宽上”。

拿 A100 粗略估算：FP16 Tensor Core 峰值约 312 TFLOPS，HBM 带宽约 2 TB/s，两者相除约 **156 FLOPs/Byte**。这个数叫“拐点”（ridge point）：

- 算子的算术强度 **高于** 156：数据搬得够快，瓶颈在算力，叫 compute-bound；
- 算子的算术强度 **低于** 156：算力闲着等数据，瓶颈在带宽，叫 memory-bound。

这就是 Roofline 模型的核心思想，模块二讲 CUDA 优化时会画成图。

### Q7. 代入几个具体 shape，GEMM 的算术强度差别有多大？

按原文公式 $AI \approx \frac{2MKN}{2(MK+KN+MN)}$（FP16 每个元素 2 字节）：

```python
def ai(M, K, N, b=2):
    return 2 * M * K * N / (b * (M * K + K * N + M * N))

ai(4096, 4096, 4096)     # 1365.3  方阵大 GEMM：远高于 156，compute-bound
ai(4096, 4096, 16384)    # 1820.4
ai(8, 4096, 4096)        # 8.0     batch 很小的推理：memory-bound
ai(1, 4096, 4096)        # 1.0     矩阵乘向量（GEMV）：严重 memory-bound
```

这解释了一个很重要的现象：**大模型推理解码阶段每次只生成一个 token，线性层退化成矩阵乘向量，算术强度只有 1 左右**，GPU 算力大部分闲置，速度几乎完全由“把权重从显存读一遍要多久”决定。这也是为什么推理优化里的 batching、量化（权重变小，读得更快）这么重要，模块四会详细讲。

### Q8. 原文练习 2：$M=4096, K=4096, N=16384$ 的 FP16 GEMM，FLOPs 和最小字节数是多少？

- FLOPs：$2 \times 4096 \times 4096 \times 16384 \approx 5.50 \times 10^{11}$；
- 最小字节数：$2 \times (4096 \times 4096 + 4096 \times 16384 + 4096 \times 16384) \approx 3.02 \times 10^{8}$ 字节，约 302 MB；
- 算术强度：$5.50 \times 10^{11} / 3.02 \times 10^{8} \approx 1820$ FLOPs/Byte，compute-bound。

## 3.4 tile 不是越大越好

> **📖 原文 [3.4 tile 不是越大越好](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#34-tile-不是越大越好)**
>
> 更大的 tile 提高数据复用，却会消耗更多：
>
> - 共享内存；
> - 寄存器；
> - 每个 block 的线程和同步；
> - 边界处理成本。
>
> 资源占用过高会降低一个 SM 同时驻留的 block/warp 数量，影响延迟隐藏。tile 选择是在数据复用、占用率、指令效率和形状适配之间折中。

### Q9. tile 越大复用越多，为什么还会变慢？“占用率”是什么？

GPU 靠**同时驻留很多线程**来掩盖访存延迟：一组线程在等数据时，切换到另一组线程计算。一个 SM（流式多处理器）能同时驻留的线程数受共享内存、寄存器总量限制。

tile 太大时，每个线程块占用的共享内存和寄存器太多，一个 SM 上只能放下很少的线程块。等数据时没有别的线程可切换，SM 就闲着。“占用率”（occupancy）就是实际驻留的线程数占最大可驻留数的比例。

可以类比成食堂打饭：窗口（SM）数量固定，每个人（线程块）点的菜越多（tile 越大），单人效率越高，但排在窗口前的人越少；一旦某个人在等菜，窗口就空转。所以 tile 大小是个折中，cuBLAS 会针对不同 shape 选不同 tile。

## 3.5 尾块与 padding

> **📖 原文 [3.5 尾块与 padding](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#35-尾块与-padding)**
>
> 当 $M,N,K$ 不是 tile size 的整数倍时，最后一块会越过有效边界。常见做法：
>
> - 加边界判断/掩码；
> - 把输入 padding 到对齐尺寸；
> - 为常见整齐 shape 提供快路径，其他 shape 走通用路径。
>
> padding 会增加计算和存储，分支会增加控制开销。哪种方式更快取决于 shape 和硬件。

### Q10. 尾块问题具体是什么样？为什么不直接 padding？

假设 tile 是 128，而 $M = 1000$：$1000 = 7 \times 128 + 104$，最后一块只有 104 行有效。Kernel 如果不管，按 128 行去读，就会读到矩阵之外的内存（越界，结果错或直接崩溃）。

两种处理方式的代价：

| 方式 | 做法 | 代价 |
| --- | --- | --- |
| 掩码/边界判断 | 每次读写前判断 `if row < M` | 每个线程多一些分支指令 |
| padding | 把矩阵补成 1024 行，多出来的填 0 | 多算 24 行、多占显存，还要多一次拷贝 |

这也是为什么很多模型的 hidden size、词表大小会刻意设成 64 或 128 的倍数（比如把词表从 50257 补到 50304）：shape 对齐时 Kernel 走最快的路径，速度可以快不少。

## 3.6 浮点分块结果为什么可能不同

> **📖 原文 [3.6 浮点分块结果为什么可能不同](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#36-浮点分块结果为什么可能不同)**
>
> 实数加法满足结合律，但浮点加法不严格满足：
>
> $$
> (a+b)+c \ne a+(b+c)
> $$
>
> 不同 tile、线程归约树、Tensor Core 路径会改变累加顺序，所以优化前后结果可能不逐位一致。正确性验证应使用适合 dtype 和问题规模的误差容限，并关注误差是否随归约长度系统性放大。

### Q11. 浮点加法不满足结合律，能举个实际的例子吗？

最小的例子：

```python
(0.1 + 0.2) + 0.3 == 0.1 + (0.2 + 0.3)   # False
```

在 GPU 上更常见的是**大量数求和，顺序不同结果不同**：

```python
torch.manual_seed(1)
v = torch.randn(1_000_000)
v.sum().item()                               # -984.26904296875
v[torch.randperm(v.numel())].sum().item()    # -984.269287109375  打乱顺序再求和
v.double().sum().item()                      # -984.2690572155757 float64 参考值
```

两个 float32 结果在第 7 位有效数字上就不一样了。这不是 bug，是有限精度的必然结果。

### Q12. 那我做实验时，同一份代码跑两次结果不一样，是不是就是这个原因？

很可能是原因之一，但不是全部。GPU 训练不可复现常见的来源：

1. **归约顺序不确定**：比如 `atomicAdd`（原子加），多个线程谁先加完是不确定的，`index_add_`、`scatter_add_` 以及某些卷积的反向都会用到；
2. **cuDNN 自动选算法**：`torch.backends.cudnn.benchmark = True` 会针对 shape 测试并选最快的卷积算法，不同次可能选到不同算法；
3. 随机数种子没固定、DataLoader 多进程的随机性等。

想要逐位复现，可以：

```python
torch.manual_seed(0)
torch.backends.cudnn.benchmark = False
torch.use_deterministic_algorithms(True)   # 遇到非确定性算子会直接报错
```

代价是可能变慢。日常实验只要求“误差在容限内一致”，写论文做消融时则最好固定种子并多跑几次取平均。

## 4.1 随机变量和概率分布

> **📖 原文 [4.1 随机变量和概率分布](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#41-随机变量和概率分布)**
>
> 随机变量把随机结果映射为数。离散随机变量用概率质量函数：
>
> $$
> p_X(x)=P(X=x),\qquad \sum_x p_X(x)=1
> $$
>
> 连续随机变量用概率密度函数：
>
> $$
> p_X(x)\ge0,\qquad
> \int_{-\infty}^{\infty}p_X(x)\,dx=1
> $$

### Q13. 概率质量函数和概率密度函数有什么区别？密度可以大于 1 吗？

- **概率质量函数（PMF）**：离散变量，$p(x)$ 本身就是概率，所以一定在 $[0,1]$，加起来等于 1。比如下一个 token 是 “the” 的概率是 0.03。
- **概率密度函数（PDF）**：连续变量，$p(x)$ 是“单位长度上的概率”，**可以大于 1**。只有积分（曲线下的面积）才是概率。

例子：在 $[0, 0.5]$ 上均匀分布，密度是 2（因为 $2 \times 0.5 = 1$），密度值大于 1 完全合法。这也是为什么原文说“连续变量在单点的概率为 0”：一个点没有宽度，面积为 0。

### Q14. 表格里的分布在 PyTorch 里分别怎么用？

```python
torch.bernoulli(torch.full((5,), 0.9))          # Bernoulli：Dropout 的 mask，每个位置 90% 保留
torch.multinomial(probs, num_samples=1)         # Categorical：按概率采样下一个 token
torch.rand(3, 4)                                # Uniform[0, 1)
torch.randn(3, 4)                               # 标准 Gaussian：均值 0、方差 1
nn.init.uniform_(w, -a, a); nn.init.normal_(w, 0, 0.02)   # 初始化
```

`torch.distributions` 里还有完整的分布对象（可以算 log_prob、熵、KL），在强化学习、VAE 里常用。

## 4.2 联合概率、边缘概率与条件概率

> **📖 原文 [4.2 联合概率、边缘概率与条件概率](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#42-联合概率边缘概率与条件概率)**
>
> 联合概率描述多个事件同时发生：
>
> $$
> P(X=x,Y=y)
> $$
>
> 对另一个变量求和或积分得到边缘概率：
>
> $$
> P(X=x)=\sum_y P(X=x,Y=y)
> $$

### Q15. 联合、边缘、条件概率能不能用一个表格例子说明？

假设统计了数据集里图片的“类别”和“拍摄环境”：

| | 室内 | 室外 | 边缘概率 $P(\text{类别})$ |
| --- | --- | --- | --- |
| 螺丝 | 0.30 | 0.10 | 0.40 |
| 螺母 | 0.20 | 0.40 | 0.60 |
| 边缘概率 $P(\text{环境})$ | 0.50 | 0.50 | 1 |

- **联合概率**：表格中间的每个格子，$P(\text{螺丝}, \text{室内}) = 0.30$；
- **边缘概率**：对一行或一列求和，$P(\text{螺丝}) = 0.30 + 0.10 = 0.40$。“边缘”这个名字就来自写在表格边上；
- **条件概率**：只看某一列，再重新归一化。$P(\text{螺丝} \mid \text{室内}) = 0.30 / 0.50 = 0.6$。

条件概率的直觉就是“**缩小世界**”：已知是室内拍的，就只看室内那一列，在这一列里重新算比例。

### Q16. 自回归的链式分解 $P(x_1, \dots, x_T) = \prod_t P(x_t \mid x_{<t})$ 是怎么来的？

就是条件概率定义 $P(A, B) = P(A)P(B \mid A)$ 反复使用：

$$
P(x_1, x_2, x_3) = P(x_1) \cdot P(x_2 \mid x_1) \cdot P(x_3 \mid x_1, x_2)
$$

这一步是**恒等式，没有任何近似**。语言模型做的事就是用一个网络去学每一项 $P(x_t \mid x_{<t})$，也就是“给定前文，下一个 token 的分布”。

所以原文说训练和推理是“同一概率模型的两个过程”：

- **训练**：整句话都已知，一次前向同时算出每个位置的 $P(x_t \mid x_{<t})$，加起来就是整句的对数概率（因果 mask 保证位置 $t$ 看不到后面）；
- **推理**：从第一个 token 开始，每次只算下一个位置的分布，采样出一个 token，再接到输入后面继续。

### Q17. 训练时“一次前向就算出所有位置”，推理时却要一个一个生成，为什么不能推理时也一次算完？

因为推理时**后面的 token 还不存在**。训练时第 5 个位置的输入是真实的第 1～4 个 token，这些在数据里都有；推理时第 5 个位置的输入是模型自己刚生成的第 4 个 token，必须先生成出来才能继续。

这就是自回归推理慢的根本原因，也是 KV Cache（把前面算过的 K、V 存下来不重复算）和推测解码（先用小模型猜几个 token，大模型一次验证）这些优化存在的意义。

## 4.3 独立与条件独立

> **📖 原文 [4.3 独立与条件独立](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#43-独立与条件独立)**
>
> 若：
>
> $$
> P(X,Y)=P(X)P(Y)
> $$
>
> 则 $X,Y$ 独立。独立比“不相关”更强；协方差为 0 不一定独立。

### Q18. “独立比不相关更强，协方差为 0 不一定独立”，有反例吗？

经典反例：$X$ 在 $[-1, 1]$ 上均匀分布，令 $Y = X^2$。$Y$ 完全由 $X$ 决定，显然不独立，但协方差约为 0：

```python
x = torch.rand(1_000_000) * 2 - 1
y = x ** 2
((x - x.mean()) * (y - y.mean())).mean()   # 0.0002，约等于 0
```

原因是协方差只衡量**线性**关系。$Y = X^2$ 是一个 U 形关系：$X$ 往正方向走 $Y$ 变大，往负方向走 $Y$ 也变大，两边的线性趋势正好抵消。

### Q19. i.i.d. 假设在训练中具体体现在哪？被破坏会怎样？

i.i.d. = independent and identically distributed，独立同分布。它体现在：

- **mini-batch 梯度是全量梯度的无偏估计**（[原文 7.1](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#71-全量梯度sgd-与-mini-batch)），前提是 batch 里的样本是从同一个分布独立抽出来的；
- **DataLoader 要 `shuffle=True`**：如果数据按类别排好序，一个 batch 里全是同一类，梯度就严重偏向这一类，训练会震荡甚至发散；
- **分布式训练用 `DistributedSampler`**：保证每个 rank 拿到不重叠、且同分布的数据。

对你的小样本研究也相关：few-shot 的 episode 采样（每次随机抽 N 类、每类 K 张）本身就是在构造“任务层面的 i.i.d.”。如果训练和测试的类别分布差异很大（比如合成数据训练、真实数据测试），就是在打破“同分布”假设，这正是域差（domain gap）问题。

## 4.4 Bayes 公式

> **📖 原文 [4.4 Bayes 公式](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#44-bayes-公式)**
>
> $$
> P(X\mid Y)
> = \frac{P(Y\mid X)P(X)}{P(Y)}
> $$
>
> 其中 $P(X)$ 是先验，$P(Y\mid X)$ 是似然，$P(X\mid Y)$ 是后验。虽然常规 LLM 训练不直接手算 Bayes 公式，但它是概率推断、参数估计和不确定性建模的基础。

### Q20. 先验、似然、后验能不能用一个分类的例子解释？

还是用螺丝的例子。设 $X$ 是“这张图是不是螺丝”，$Y$ 是“检测器报警了”：

- **先验** $P(X)$：没看图之前，产线上 1% 的零件是螺丝；
- **似然** $P(Y \mid X)$：如果真是螺丝，检测器 95% 会报警；如果不是，也有 5% 会误报；
- **后验** $P(X \mid Y)$：检测器报警了，这张图真是螺丝的概率是多少？

$$
P(\text{螺丝} \mid \text{报警}) = \frac{0.95 \times 0.01}{0.95 \times 0.01 + 0.05 \times 0.99} \approx 0.16
$$

只有 16%！因为螺丝本来就少（先验小），误报的绝对数量反而更多。这个例子说明在类别严重不平衡时，只看准确率和召回率会被误导，要结合先验去看精确率。

## 4.5 期望：概率加权平均

> **📖 原文 [4.5 期望：概率加权平均](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#45-期望概率加权平均)**
>
> 离散变量：
>
> $$
> \mathbb{E}[X]=\sum_x xP(X=x)
> $$
>
> 连续变量：
>
> $$
> \mathbb{E}[X]=\int x p(x)\,dx
> $$

### Q21. 期望的线性性“不要求变量独立”，这个性质有什么用？

它让很多计算变得非常简单。比如一个 batch 的平均 loss：

$$
\mathbb{E}\left[\frac{1}{B}\sum_{i=1}^{B} L_i\right] = \frac{1}{B}\sum_{i=1}^{B}\mathbb{E}[L_i]
$$

不管样本之间有没有相关性，期望都可以直接拆开。这正是“mini-batch 平均梯度的期望等于全量梯度”成立的原因。

对比：**方差**就没有这么好的性质，$\operatorname{Var}(X+Y) = \operatorname{Var}(X) + \operatorname{Var}(Y)$ 必须要求独立（或至少不相关），见 [原文 4.6](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#46-方差与标准差)。

### Q22. 训练目标 $\min_\theta \mathbb{E}_{(x,y)\sim p_{data}}[\mathcal{L}]$ 和代码里的 `loss.mean()` 是什么关系？

公式里的期望是对“真实世界所有可能的数据”求平均，这个分布我们拿不到。代码里能做的是：

1. 用训练集近似真实分布（经验风险）；
2. 再用一个 mini-batch 近似整个训练集。

所以 `loss = F.cross_entropy(logits, y)`（默认 `reduction="mean"`）就是在用 $B$ 个样本的平均去估计那个期望。两层近似，第一层带来泛化问题（训练集和真实分布的差距），第二层带来梯度噪声（[原文 7.1](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#71-全量梯度sgd-与-mini-batch)）。

## 4.6 方差与标准差

> **📖 原文 [4.6 方差与标准差](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#46-方差与标准差)**
>
> 方差衡量围绕均值的波动：
>
> $$
> \operatorname{Var}(X)
> = \mathbb{E}[(X-\mu)^2]
> = \mathbb{E}[X^2]-\mu^2
> $$
>
> 标准差与原变量单位相同：
>
> $$
> \sigma = \sqrt{\operatorname{Var}(X)}
> $$

### Q23. $\operatorname{Var}(X) = \mathbb{E}[X^2] - \mu^2$ 这两种写法哪个好？

数学上完全相等，**数值上第二种可能很差**。当数据均值很大、方差很小时，$\mathbb{E}[X^2]$ 和 $\mu^2$ 是两个非常接近的大数，相减后有效数字几乎全被抵消：

```python
x = (1e4 + torch.randn(100000)).float()   # 均值 1万，方差 1
(x * x).mean() - x.mean() ** 2            # 16.0     ← 错得离谱
((x - x.mean()) ** 2).mean()              # 1.00248  ← 先减均值再平方（two-pass）
x.double().var(unbiased=False)            # 1.00248  float64 参考值
```

第一种写法在 float32 下算出 16，真实值是 1。这就是 [原文 8.6](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#86-灾难性消减) 讲的“灾难性消减”。所以 LayerNorm 这种要算方差的算子，实现时都用 two-pass 或 Welford 算法。

### Q24. 方差和标准差有什么区别？为什么初始化用 `std=0.02` 而不说方差？

标准差是方差开根号，**单位和原变量相同**。比如权重的值大约在 ±0.02 附近波动，用标准差说更直观；方差是 0.0004，不好想象。

代码里要注意两者别混：`torch.randn(n) * 0.02` 得到的是标准差 0.02（方差 0.0004）的分布；`nn.init.normal_(w, mean=0, std=0.02)` 参数名明确写的是 `std`。

## 4.7 协方差与相关系数

> **📖 原文 [4.7 协方差与相关系数](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#47-协方差与相关系数)**
>
> 协方差衡量两个变量共同变化：
>
> $$
> \operatorname{Cov}(X,Y)
> = \mathbb{E}[(X-\mu_X)(Y-\mu_Y)]
> $$
>
> 标准化得到相关系数：
>
> $$
> \rho_{X,Y}
> = \frac{\operatorname{Cov}(X,Y)}{\sigma_X\sigma_Y}
> $$

### Q25. 协方差和相关系数有什么区别？为什么要“标准化”？

协方差的大小受变量**单位**影响：把身高从米换成厘米，协方差就放大 100 倍，但两个变量的关系并没变。相关系数除以两个标准差后，取值固定在 $[-1, 1]$：

- $+1$：完全正线性相关；
- $0$：没有线性关系（但可能有非线性关系，见 [原文 4.3](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#43-独立与条件独立) 的反例）；
- $-1$：完全负线性相关。

所以比较“关系强弱”用相关系数，做矩阵运算（比如 PCA）用协方差矩阵。

### Q26. 协方差矩阵为什么一定是“对称半正定”？PCA 怎么用它？

- **对称**：$\operatorname{Cov}(X_i, X_j) = \operatorname{Cov}(X_j, X_i)$，交换两个变量结果不变；
- **半正定**：对任意向量 $\mathbf{a}$，$\mathbf{a}^\top\mathbf{\Sigma}\mathbf{a} = \operatorname{Var}(\mathbf{a}^\top\mathbf{x}) \ge 0$，因为方差不可能是负数。

PCA 对 $\mathbf{\Sigma}$ 做特征分解（对称矩阵一定能做，而且特征向量互相正交）：特征值最大的那个特征向量，就是数据方差最大的方向。分析特征时，比如看 backbone 提出来的特征在合成数据和真实数据上是否分布一致，常先用 PCA 降到 2 维画散点图。

## 4.8 样本均值、方差与 mini-batch

> **📖 原文 [4.8 样本均值、方差与 mini-batch](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#48-样本均值方差与-mini-batch)**
>
> 给定样本 $x_1,\ldots,x_n$，样本均值：
>
> $$
> \bar{x}=\frac{1}{n}\sum_{i=1}^{n}x_i
> $$
>
> 常用无偏样本方差：
>
> $$
> s^2=\frac{1}{n-1}\sum_{i=1}^{n}(x_i-\bar{x})^2
> $$

### Q27. $n-1$ 和 $n$ 到底用哪个？PyTorch 默认是哪个？

- 除以 $n-1$：**无偏**估计，用来从样本推断总体方差（统计学的场景）；
- 除以 $n$：就是这组数自己的方差（总体方差形式）。

**PyTorch 的 `torch.var` 和 `torch.std` 默认除以 $n-1$，而 LayerNorm 内部除以 $n$**，这正是原文提醒的坑：

```python
t = torch.tensor([1., 2., 3., 4.])
t.var()                        # 1.6667  默认 correction=1，除以 n-1 = 3
t.var(unbiased=False)          # 1.25    除以 n = 4
torch.var(t, correction=0)     # 1.25    新写法，同上

F.layer_norm(t, (4,))                                   # tensor([-1.3416, -0.4472, 0.4472, 1.3416])
(t - t.mean()) / torch.sqrt(t.var(unbiased=False) + 1e-5)  # 结果相同 ← 必须用 unbiased=False
```

如果手写 LayerNorm 时用了默认的 `t.var()`，结果会有一个系统性的小误差，单元测试对不上还很难找原因。另外 NumPy 的 `np.var` 默认除以 $n$，和 PyTorch 正好相反，两个库混用时要特别小心。

### Q28. 为什么除以 $n-1$ 才是“无偏”？

直觉解释：样本方差用的是**样本均值** $\bar{x}$，而 $\bar{x}$ 本身就是从这组数里算出来的，它会天然地“贴近”这组数，导致 $\sum(x_i - \bar{x})^2$ 比用真实均值 $\mu$ 算出来的偏小。除以 $n-1$ 而不是 $n$ 正好把这个偏小补回来。

$n$ 很大时（比如 LayerNorm 的 $H = 4096$），$n$ 和 $n-1$ 几乎没区别；$n$ 很小时（比如上面 $n=4$）差别很明显。

## 4.9 最大似然与负对数似然

> **📖 原文 [4.9 最大似然与负对数似然](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#49-最大似然与负对数似然)**
>
> 给定数据 $D=\{(x_i,y_i)\}$，最大似然估计选择让观测数据最可能的参数：
>
> $$
> \theta^*
> = \arg\max_\theta
> \prod_i p_\theta(y_i\mid x_i)
> $$
>
> 乘积容易数值下溢，且不便求导。取对数把乘积变成求和：
>
> $$
> \theta^*
> = \arg\max_\theta
> \sum_i \log p_\theta(y_i\mid x_i)
> $$

### Q29. 为什么“乘积容易数值下溢”？取对数为什么能解决？

一句话的概率是每个 token 概率的乘积。假设每个 token 平均概率 0.1，一段 400 个 token 的文本：

```python
p = torch.full((400,), 0.1, dtype=torch.float64)
p.prod()          # 0.0        ← 1e-400 比 float64 能表示的最小数还小，下溢成 0
p.log().sum()     # -921.03    ← 对数和，完全没问题
torch.full((50,), 0.1).prod()   # 0.0  float32 下 50 个 token 就下溢了
```

一旦变成 0，所有句子的概率都是 0，没法比较，取 log 还会得到 $-\infty$。取对数把乘积变成求和，数值范围就从“指数级”变成了“线性级”，-921 这样的数随便存。

而且求导更方便：乘积的导数要用乘法法则展开一长串，求和的导数就是各项导数相加。

### Q30. 最大似然和交叉熵 loss 是什么关系？

**是同一个东西**。负对数似然 $-\sum_i \log p_\theta(y_i \mid x_i)$ 中，$p_\theta(y_i \mid x_i)$ 就是模型给正确类别的概率。这正是 [原文 5.5](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#55-交叉熵) 中 one-hot 标签下交叉熵的形式 $-\log p_y$。

所以 `F.cross_entropy` 本质上就是在做最大似然估计。下一篇会从另一个角度（信息论）再推一次，得到同样的式子。

## 第2章系列目录

- [导读与易错点速查](/posts/ai-infra/ch2-00-guide/)
- [（一）符号、张量与线性代数](/posts/ai-infra/ch2-01-tensor-linear-algebra/)
- **（二）GEMM 分块与概率论**（本篇）
- [（三）Softmax、交叉熵与反向传播](/posts/ai-infra/ch2-03-softmax-backprop/)
- [（四）优化、归一化、数值精度与综合算例](/posts/ai-infra/ch2-04-optimization-numerics/)

其他章节：

- [第1章 编程语言基础：导读与易错点速查](/posts/ai-infra/ch1-00-guide/)

