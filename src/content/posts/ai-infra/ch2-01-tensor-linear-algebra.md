---
title: "AI Infra 第2章笔记（一）：符号、张量与线性代数"
published: 2026-11-03T21:00:00
description: "这一篇的主线：读公式先问 shape、代价、风险三件事 → 工程里的张量 = 数据 + shape + stride + dtype → 广播和矩阵乘法本质上都是“哪些维度对齐、哪些维度被归约” → 秩、SVD、Lo…"
tags: ["AI Infra", "数学", "线性代数", "PyTorch"]
category: "AI Infra 学习笔记"
draft: false
---

这一篇的主线：**读公式先问 shape、代价、风险三件事 → 工程里的张量 = 数据 + shape + stride + dtype → 广播和矩阵乘法本质上都是“哪些维度对齐、哪些维度被归约” → 秩、SVD、LoRA 讲的是同一件事：矩阵里的信息其实没那么多。** 很多问题都可以拿第1章的“名字绑定对象”“行主序 offset”对照着理解。

> [!NOTE]
> 本篇对应原文 [AIInfraGuide · 第2章：数学基础](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/) 的 §1–2（作者 [caomaolufei](https://github.com/caomaolufei/AIInfraGuide)，MIT 许可）。每节开头的引用块摘自原文，文中的“原文 x.y”可以点击跳转到原文对应小节。问答是按第1章的提问思路整理的，代码输出均为实际运行结果。

## 1.1 本章符号

> **📖 原文 [1.1 本章符号](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#11-本章符号)**
>
> | 符号 | 含义 |
> |------|------|
> | $x$ | 标量，或语境明确时的变量 |
> | $\mathbf{x}$ | 向量 |
> | $\mathbf{X}$ | 矩阵或高阶张量 |
> | $X_{ij}$ | 矩阵第 $i$ 行第 $j$ 列元素 |
> | $\mathbb{R}^{m \times n}$ | $m$ 行、$n$ 列的实矩阵集合 |
> | $\mathbf{X}^\top$ | 转置 |
> | $\mathbf{X}^{-1}$ | 逆矩阵（若存在） |
> | $\lVert \mathbf{x} \rVert_2$ | 向量的 L2 范数 |
> | $\mathbb{E}[X]$ | 期望 |
> | $\operatorname{Var}(X)$ | 方差 |
> | $\nabla_x f$ | $f$ 对 $x$ 的梯度 |
> | $\partial f / \partial x$ | 偏导数 |
> | $\odot$ | 逐元素乘法 |
>
> 深度学习代码常用 batch-first 记法：
>
> - $B$：batch size；
> - $S$：sequence length；
> - $H$：hidden size；
> - $V$：vocabulary size；
> - $N_h$：attention head 数；
> - $D_h = H / N_h$：每个 head 的维度。

### Q1. 为什么向量用粗体小写 $\mathbf{x}$、矩阵用粗体大写 $\mathbf{X}$？代码里根本没有粗细之分呀

这只是**写给人看的约定**，目的是一眼看出“这个量有几维”：

| 写法 | 含义 | PyTorch 里通常是 |
| --- | --- | --- |
| $x$ | 标量 | `float` 或 shape 为 `()` 的张量 |
| $\mathbf{x}$ | 向量 | shape `(H,)` |
| $\mathbf{X}$ | 矩阵或更高维张量 | shape `(M, N)`、`(B, S, H)` |
| $X_{ij}$ | 第 $i$ 行第 $j$ 列的**一个数**（所以不加粗） | `X[i, j]` |

代码确实不区分粗细，所以读代码时要靠**变量名 + shape 注释**来补回这个信息。原文后面所有推导都默认你会先看 shape，这也是为什么很多工程代码会在每行后面写 `# (B, S, H)` 这样的注释。

### Q2. $\mathbb{R}^{m \times n}$ 这个符号怎么读？

读作“$m$ 乘 $n$ 的实矩阵集合”。$\mathbf{W} \in \mathbb{R}^{H \times 4H}$ 就是说：$\mathbf{W}$ 是一个 $H$ 行、$4H$ 列、每个元素都是实数的矩阵。翻译成代码就是：

```python
W = torch.empty(H, 4 * H)   # W.shape == (H, 4H)
```

$\mathbb{R}^{B \times S \times H}$ 同理，是一个三维张量。数学上写“集合属于”，工程上读成“shape 是这个”就行。

### Q3. $B, S, H, V, N_h, D_h$ 分别是什么？为什么 $D_h = H / N_h$？

| 符号 | 含义 | 典型值（LLaMA-7B 量级） |
| --- | --- | --- |
| $B$ | batch size，一次喂几条序列 | 1～64 |
| $S$ | sequence length，每条序列多少个 token | 2048、4096 |
| $H$ | hidden size，每个 token 用多长的向量表示 | 4096 |
| $V$ | 词表大小 | 32000 |
| $N_h$ | 注意力头的个数 | 32 |
| $D_h$ | 每个头的维度 | 128 |

多头注意力是把长度为 $H$ 的向量**切成 $N_h$ 段**，每段交给一个头：$4096 = 32 \times 128$。所以 $D_h = H / N_h$ 是“切分”带来的约束，不是额外的超参数。这也意味着 $H$ 必须能被 $N_h$ 整除，否则 reshape 会直接报错。

如果你做的是视觉（比如 ViT），把 $S$ 换成“图片被切成的 patch 数”，其余完全一样。

## 1.2 读公式时先问三个问题

> **📖 原文 [1.2 读公式时先问三个问题](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#12-读公式时先问三个问题)**
>
> 以语言模型输出投影为例：
>
> $$
> \mathbf{Y} = \mathbf{X}\mathbf{W}, \qquad
> \mathbf{X} \in \mathbb{R}^{(BS) \times H},\quad
> \mathbf{W} \in \mathbb{R}^{H \times V}
> $$
>
> 第一问，**形状是否匹配**？内维 $H$ 相同，输出是 $(BS) \times V$。

### Q4. $2BSHV$ 里的 2 是哪来的？

每个输出元素 $Y_{ij} = \sum_{k=1}^{H} X_{ik} W_{kj}$ 要做 $H$ 次**乘法**和 $H$ 次**加法**（严格说是 $H-1$ 次加法，工程估算忽略这个 1）。原文约定“一次乘法和一次加法各算一个浮点操作”，所以每个输出元素约 $2H$ 个 FLOPs。

输出一共有 $(BS) \times V$ 个元素，于是总量是：

$$
(BS) \times V \times 2H = 2BSHV
$$

记一个万能口诀：**矩阵乘法的 FLOPs = 2 × 输出元素数 × 被归约的那一维长度**。$(M,K) @ (K,N)$ 就是 $2MNK$。

### Q5. FLOPs 和 FLOPS 是一回事吗？

不是，差一个小写 s 和一个时间单位：

- **FLOPs**（s 小写）：浮点运算的**次数**，衡量“这个算子要算多少”。
- **FLOPS**（全大写，floating-point operations per second）：每秒能做多少次，衡量“硬件有多快”。

用法是：耗时下限 ≈ FLOPs ÷ FLOPS。比如一次 $2.2 \times 10^{12}$ FLOPs 的投影，放到每秒 $3 \times 10^{14}$ 次（约 300 TFLOPS）的 GPU 上，理论最快约 7 毫秒。实际达不到理论峰值，这正是后面讲“算术强度”的原因。

### Q6. 第三问“实现风险”具体指什么？举个能落地的例子

以原文的输出投影 $(BS) \times H$ 乘 $H \times V$ 为例，令 $B=8, S=2048, H=4096, V=32000$：

- **存储风险**：输出 logits 有 $8 \times 2048 \times 32000 \approx 5.2 \times 10^8$ 个元素，FP32 下约 2 GB，仅这一个中间张量就可能挤爆显存；
- **数值风险**：logits 后面接 Softmax，直接 `exp` 大数会溢出（见 [原文 5.2](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#52-数值稳定的-softmax)）；
- **性能风险**：权重是否连续存储、是否转置，决定 GEMM 能不能走最快的路径。

所以“三问”不是走形式：第一问保证不报错，第二问估出耗时和显存，第三问提前知道哪里会出 `nan` 或 OOM。

## 2.1 标量、向量、矩阵和张量

> **📖 原文 [2.1 标量、向量、矩阵和张量](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#21-标量向量矩阵和张量)**
>
> - **标量**是一个数，如学习率 $\eta = 10^{-4}$；
> - **向量**是一维有序数组，如一个 token 的隐藏状态 $\mathbf{x} \in \mathbb{R}^{H}$；
> - **矩阵**是二维数组，如线性层权重 $\mathbf{W} \in \mathbb{R}^{H \times 4H}$；
> - **张量**是多维数组，如一批序列的隐藏状态 $\mathbf{X} \in \mathbb{R}^{B \times S \times H}$。
>
> 工程中的“张量”还带有数学 shape 之外的信息：
>
> ```text
> 数据地址 + shape + stride + dtype + device + layout
> ```
>
> 两个张量 shape 相同，不代表内存布局相同；数值相同，也不代表 dtype 和误差特性相同。

### Q7. 标量、向量、矩阵和张量，在 PyTorch 里是不是都叫 Tensor？

是的，PyTorch 里它们都是 `torch.Tensor`，区别只在 `ndim`（维度数）：

```python
torch.tensor(3.0).ndim          # 0 → 标量
torch.randn(4096).ndim          # 1 → 向量
torch.randn(4096, 16384).ndim   # 2 → 矩阵
torch.randn(8, 2048, 4096).ndim # 3 → 三阶张量
```

数学书会把“张量”专门留给三维及以上，但工程里统一叫 tensor，靠 shape 区分。

### Q8. “数据地址 + shape + stride + dtype + device + layout”，这六项分别是什么？

可以把一个张量想成**一张指向一段内存的说明书**：

| 字段 | 回答的问题 | 查看方式 |
| --- | --- | --- |
| 数据地址 | 数据从内存哪里开始 | `x.data_ptr()` |
| shape | 逻辑上有几维、每维多长 | `x.shape` |
| stride | 每一维走一步，要跳过多少个元素 | `x.stride()` |
| dtype | 每个元素怎么编码（几个字节、什么格式） | `x.dtype` |
| device | 数据在 CPU 内存还是哪块 GPU 显存上 | `x.device` |
| layout | 稠密（strided）还是稀疏等其他格式 | `x.layout` |

这和第1章讲的“Python 变量名是贴在对象上的标签”是一个思路：**张量对象本身只是元数据，真正的数字在另一块内存里**。所以多个张量可以共享同一块内存，只是“说明书”不同，下一节的转置就是例子。

### Q9. “数值相同，也不代表 dtype 和误差特性相同”怎么理解？

同样存的是 `0.1`，不同 dtype 存进去的其实是不同的近似值：

```python
torch.tensor(1e-3, dtype=torch.float16).item()   # 0.0010004043579101562
torch.tensor(1e-3, dtype=torch.bfloat16).item()  # 0.00099945068359375
```

打印出来看着都像 `0.001`，但误差不同，后续累加 1 万次以后差别会被放大。这部分在 [第四篇](/posts/ai-infra/ch2-04-optimization-numerics/) 的数值计算里会详细展开。

## 2.2 Shape、索引和 stride

> **📖 原文 [2.2 Shape、索引和 stride](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#22-shape索引和-stride)**
>
> 考虑一个 shape 为 $(2, 3)$ 的行主序矩阵：
>
> $$
> \mathbf{X} =
> \begin{bmatrix}
> 1 & 2 & 3 \\
> 4 & 5 & 6
> \end{bmatrix}
> $$
>
> 底层连续存储为 `[1, 2, 3, 4, 5, 6]`，以元素为单位的 stride 是 $(3, 1)$。元素 $X_{ij}$ 的线性偏移为：
>
> $$
> \text{offset}(i,j) = 3i + j
> $$

### Q10. stride 到底是什么？和第1章的行主序 offset 是一回事吗？

是同一件事的推广。第1章讲过行主序 `A[M][N]` 的偏移是 `row × N + col`。把系数单独拿出来写成一个元组，就是 stride：

- shape `(2, 3)` 的行主序矩阵，stride 是 `(3, 1)`：行号加 1 要跳 3 个元素，列号加 1 跳 1 个；
- 于是 $\text{offset}(i, j) = 3i + 1 \cdot j$，和 `row × N + col` 完全一样。

推广到任意维度：$\text{offset} = \sum_d \text{index}_d \times \text{stride}_d$。三维张量 shape `(B, S, H)` 连续存储时 stride 是 `(S*H, H, 1)`。

### Q11. 转置为什么“通常无需复制数据”？那转置后的数据在哪？

转置只改说明书，不动数据：

```python
x = torch.arange(6).reshape(2, 3)
t = x.T
x.stride(), t.stride()            # ((3, 1), (1, 3))
t.is_contiguous()                 # False
x.data_ptr() == t.data_ptr()      # True —— 指向同一块内存
```

`t` 只是把 shape 从 `(2,3)` 改成 `(3,2)`、stride 从 `(3,1)` 换成 `(1,3)`。读 `t[i, j]` 时按 $1 \cdot i + 3 \cdot j$ 去找，正好找到 `x[j, i]`。所以如果你改 `t[0, 1] = 100`，`x[1, 0]` 也会变成 100，这和第1章“两个名字指向同一个对象”是一回事。

### Q12. `view` 和 `reshape` 有什么区别？为什么原文说 reshape“是否复制取决于 layout”？

- `view`：**保证不复制**，只改说明书。如果新 shape 用 stride 表示不出来，就直接报错；
- `reshape`：能 view 就 view，**view 不了就偷偷复制一份**。

```python
t.view(6)
# RuntimeError: view size is not compatible with input tensor's size and stride ...
r = t.reshape(6)
r                                 # tensor([0, 3, 1, 4, 2, 5])
r.data_ptr() == x.data_ptr()      # False —— 已经复制了
```

为什么 `t` 不能 view 成一维？一维张量要求“相邻元素在内存里也相邻”（stride 为 1），但 `t` 按行读出来是 `0, 3, 1, 4, 2, 5`，在内存里是跳着的，没有一个 stride 能描述这个顺序，只能复制重排。

实际写代码的建议：**想确保共享内存就用 `view`，报错说明你对布局的理解错了；不在乎就用 `reshape`**。

### Q13. `contiguous()` 是干什么的？什么时候必须调用？

`contiguous()` 的意思是“如果不是连续布局，就复制成连续的；已经连续就原样返回”：

```python
c = t.contiguous()
c.stride()                        # (2, 1) —— 变成标准行主序
c.data_ptr() == x.data_ptr()      # False
```

常见的必须场景：

1. 对转置/切片后的张量调用 `view`，比如多头注意力里 `x.transpose(1, 2).contiguous().view(B, S, H)`；
2. 把张量传给自定义 C++/CUDA 算子。手写 Kernel 通常假设输入是连续的，用第1章的 `row × N + col` 去算地址，传进一个转置视图就会读错数据，而且**不报错**。

原文最后一句“同时思考逻辑维度和物理布局”，指的就是这个坑。

## 2.3 逐元素运算与广播

> **📖 原文 [2.3 逐元素运算与广播](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#23-逐元素运算与广播)**
>
> 逐元素加法要求对应元素能配对：
>
> $$
> \mathbf{C} = \mathbf{A} + \mathbf{B}, \qquad C_{ij} = A_{ij} + B_{ij}
> $$
>
> 广播允许某些长度为 1 或缺失的维度被逻辑扩展。例如：
>
> $$
> \mathbf{X} \in \mathbb{R}^{B \times S \times H}, \qquad
> \mathbf{b} \in \mathbb{R}^{H}
> $$

### Q14. 广播的规则“从尾部维度向前对齐”，能用一个例子走一遍吗？

以 $\mathbf{X}$ shape `(8, 2048, 4096)` 加 bias $\mathbf{b}$ shape `(4096,)` 为例，先右对齐：

```text
X:      8   2048   4096
b:                 4096
```

从最后一维往前逐对检查：

1. `4096` 对 `4096`：相等 ✓
2. `2048` 对“不存在”：规则 3，缺失的维度当作 1，可以扩展 ✓
3. `8` 对“不存在”：同上 ✓

结果 shape `(8, 2048, 4096)`。逻辑上相当于把 `b` 复制了 $8 \times 2048$ 份，但实际不会真的复制。

再看一个失败的例子：`(8, 2048, 4096)` 加 `(2048,)`。右对齐后是 `4096` 对 `2048`，不相等，也都不是 1，直接报错。

### Q15. “错误 shape 也可能恰好能广播，代码不报错却算错语义”，有真实例子吗？

最经典的是回归任务的 loss。模型输出 `pred` 常常是 `(B, 1)`，标签 `target` 是 `(B,)`：

```python
pred = torch.randn(4, 1)
target = torch.randn(4)
(pred - target).shape     # torch.Size([4, 4])  ← 不是 (4,)！
F.mse_loss(pred, target)
# UserWarning: Using a target size (torch.Size([4])) that is different to the input size (torch.Size([4, 1])) ...
```

按规则右对齐：`(4, 1)` 和 `(4,)` → `(4, 1)` 和 `(1, 4)` → 结果 `(4, 4)`。于是每个预测值和**所有**标签都做了差，loss 变成一个毫无意义的平均值，模型还能“正常”训练下去，只是效果莫名其妙地差。

防法很简单：在 loss 之前加一句 `assert pred.shape == target.shape`，或者 `pred.squeeze(-1)`。

### Q16. 广播“不会先复制出完整张量”，那它在内存里是怎么实现的？

还是靠 stride：被扩展的维度 stride 设为 0。stride 为 0 的意思是“这一维走多少步都停在原地”，所以同一个元素被反复读到。

```python
b = torch.randn(4096)
b.expand(8, 2048, 4096).stride()   # (0, 0, 1)
```

这就是为什么原文说“后续 Kernel 的访存仍会受影响”：逻辑上是 $8 \times 2048 \times 4096$ 个元素，物理上只有 4096 个，Kernel 要能正确处理 stride 为 0 的情况。

## 2.4 点积：相似度与加权求和

> **📖 原文 [2.4 点积：相似度与加权求和](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#24-点积相似度与加权求和)**
>
> 两个 $n$ 维向量的点积：
>
> $$
> \mathbf{x}^\top \mathbf{y} = \sum_{i=1}^{n}x_i y_i
> $$
>
> 它有两种常用直觉。

### Q17. 点积的“加权求和”和“方向相似性”是同一个公式，为什么有两种理解？

公式只有一个 $\mathbf{x}^\top\mathbf{y} = \sum_i x_i y_i$，两种理解来自**你怎么看待这两个向量**：

- **加权求和**：一个向量是“权重”，另一个是“值”。比如 Attention 最后一步，用注意力概率 $p_1, \dots, p_S$ 对 $S$ 个 value 加权，权重和为 1，结果就是加权平均；
- **方向相似性**：两个向量是同一种东西，比如两个 token 的表示。点积大说明方向接近。

Attention 的 $\mathbf{Q}\mathbf{K}^\top$ 用的是第二种理解（衡量 query 和 key 有多匹配），后面的 $\mathbf{P}\mathbf{V}$ 用的是第一种（按匹配程度加权平均 value）。一个公式在同一个模块里出现两次，含义不同。

### Q18. 点积和余弦相似度有什么区别？什么时候用哪个？

余弦相似度 = **把两个向量都归一化成长度 1 之后的点积**：

$$
\cos\theta = \frac{\mathbf{x}^\top\mathbf{y}}{\lVert\mathbf{x}\rVert_2\lVert\mathbf{y}\rVert_2}
$$

区别在于是否在乎**长度**：

- 点积：长度越长，点积越大。一个范数很大的向量会和谁都“很像”；
- 余弦：只看方向，取值在 $[-1, 1]$。

和你的小样本研究直接相关：原型网络（Prototypical Network）、很多 few-shot 方法都会在“欧氏距离 / 点积 / 余弦”之间做选择。用余弦时通常还要乘一个温度系数（比如 10 或可学习），否则相似度被压在 $[-1,1]$，经过 Softmax 后概率太平，训练信号太弱。这个温度和 [原文 5.4](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#54-温度参数) 的温度是同一个东西。

```python
import torch.nn.functional as F
f = F.normalize(features, dim=-1)        # (N, D) 每行长度变成 1
p = F.normalize(prototypes, dim=-1)      # (C, D)
logits = 10.0 * f @ p.T                  # (N, C)，就是余弦相似度 × 温度
```

### Q19. 为什么分母加 $\epsilon$？$\epsilon$ 取多少？

如果某个向量全是 0（比如被 padding 的位置），范数为 0，分母为 0，结果就是 `nan`，而且 `nan` 会沿着反向传播污染所有参数。

加一个很小的 $\epsilon$（常见 `1e-8` 或 `1e-12`）保证分母不为 0。PyTorch 的 `F.normalize` 默认 `eps=1e-12`，`F.cosine_similarity` 默认 `eps=1e-8`。在 FP16 下要小心：FP16 能表示的最小正规数约 $6.1 \times 10^{-5}$，`1e-12` 会直接变成 0，等于没加，所以半精度下通常把归一化放在 FP32 里做。

## 2.5 范数：长度、误差与约束

> **📖 原文 [2.5 范数：长度、误差与约束](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#25-范数长度误差与约束)**
>
> 常见向量范数：
>
> $$
> \lVert \mathbf{x} \rVert_1 = \sum_i |x_i|
> $$
>
> $$
> \lVert \mathbf{x} \rVert_2 = \sqrt{\sum_i x_i^2}
> $$
>
> $$
> \lVert \mathbf{x} \rVert_\infty = \max_i |x_i|
> $$

### Q20. L1、L2、L∞ 范数分别在什么场景用？

| 范数 | 公式直觉 | 典型用途 |
| --- | --- | --- |
| L1 | 各元素绝对值之和 | L1 正则（让参数变稀疏）、MAE loss |
| L2 | 欧氏长度 | 梯度裁剪、权重衰减、MSE、向量归一化 |
| L∞ | 绝对值最大的那个元素 | 比较两个实现的**最大误差**，找出最坏的那个点 |

做 Infra 时最常用的是 L∞：验证你写的 Kernel 和参考实现是否一致，最关心的是“最差的那个元素差了多少”，用 `(out - ref).abs().max()` 就是 L∞ 范数。

### Q21. Frobenius 范数和 L2 范数是什么关系？

Frobenius 范数就是**把矩阵拍平成一个长向量后的 L2 范数**：

```python
A = torch.randn(3, 4)
torch.linalg.norm(A)            # 默认就是 Frobenius
torch.linalg.norm(A.flatten())  # 结果相同
```

注意“矩阵的 2-范数”是另一个东西（最大奇异值），后面 [原文 2.12](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#212-奇异值分解svd) 会用到。`torch.linalg.norm(A, ord=2)` 算的是谱范数，不是 Frobenius。名字很像，含义不同。

### Q22. `atol` 和 `rtol` 是什么？`torch.allclose` 默认值是多少？

判断条件是：

$$
|\hat{x} - x| \le \text{atol} + \text{rtol} \cdot |x|
$$

`torch.allclose` 的默认值是 `rtol=1e-05, atol=1e-08`。两部分分工：

- `rtol`（相对容差）：管**大数**。$10^6$ 和 $10^6 + 5$ 的差是 5，但相对误差只有 $5 \times 10^{-6}$，应该算相等；
- `atol`（绝对容差）：管**接近 0 的数**。参考值是 0 时，相对误差的分母是 0，没法用，只能看绝对差。

```python
torch.allclose(torch.tensor(1e-9), torch.tensor(2e-9))       # True：差 1e-9 < atol
torch.allclose(torch.tensor(1e6),  torch.tensor(1e6 + 5))    # True：5 < 1e-5 * 1e6 = 10
torch.allclose(torch.tensor(1e6),  torch.tensor(1e6 + 50))   # False
```

验证 FP16/BF16 Kernel 时默认值太严，会一直报不相等，通常要放宽到 `rtol=1e-3, atol=1e-3` 左右（BF16 更宽）。`torch.testing.assert_close` 会按 dtype 自动选默认容差，比 `allclose` 更省心。

## 2.6 矩阵乘法是一组点积

> **📖 原文 [2.6 矩阵乘法是一组点积](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#26-矩阵乘法是一组点积)**
>
> 若：
>
> $$
> \mathbf{A} \in \mathbb{R}^{M \times K}, \qquad
> \mathbf{B} \in \mathbb{R}^{K \times N}
> $$
>
> 则：
>
> $$
> \mathbf{C} = \mathbf{A}\mathbf{B}
> \in \mathbb{R}^{M \times N}
> $$

### Q23. 为什么说“矩阵乘法是一组点积”？

看公式 $C_{ij} = \sum_k A_{ik}B_{kj}$：左边是 $\mathbf{A}$ 的**第 $i$ 行**，右边是 $\mathbf{B}$ 的**第 $j$ 列**，两者逐元素相乘再求和，正好是一次点积。

所以 $\mathbf{C}$ 的每个元素都是“$\mathbf{A}$ 的某一行 · $\mathbf{B}$ 的某一列”，一共 $M \times N$ 次点积，每次长度 $K$。这也解释了 shape 规则：

```text
(M, K) @ (K, N) -> (M, N)
     ↑     ↑
     这两个 K 必须相等，因为点积要求两个向量一样长
```

### Q24. $\alpha\mathbf{A}\mathbf{B} + \beta\mathbf{C}$ 这种形式有什么用？为什么不直接写 $\mathbf{A}\mathbf{B}$？

这是 BLAS（线性代数基础库）的标准接口，把几种常见操作合成一次 Kernel 调用：

- $\beta = 0$：就是普通的 $\mathbf{C} = \alpha\mathbf{A}\mathbf{B}$；
- $\beta = 1$：结果**累加**到已有的 $\mathbf{C}$ 上，比如梯度累加、残差相加；
- $\alpha$：顺带做缩放，比如 Attention 的 $1/\sqrt{D_h}$。

好处是减少一次读写：如果先算 $\mathbf{A}\mathbf{B}$ 写回显存，再读出来加到 $\mathbf{C}$ 上，就多搬了一遍数据。这是后面“算子融合”思想最早的样子。PyTorch 里对应 `torch.addmm(C, A, B, beta=β, alpha=α)`。

### Q25. “选择不同结合顺序可以显著改变 FLOPs”，差别能有多大？

非常大。设 $\mathbf{A}, \mathbf{B}$ 都是 $n \times n$，$\mathbf{c}$ 是 $n \times 1$ 的向量，$n = 4096$：

- $(\mathbf{A}\mathbf{B})\mathbf{c}$：先算矩阵乘矩阵 $2n^3$，再乘向量 $2n^2$，合计约 $1.37 \times 10^{11}$ FLOPs；
- $\mathbf{A}(\mathbf{B}\mathbf{c})$：两次矩阵乘向量，各 $2n^2$，合计约 $6.7 \times 10^{7}$ FLOPs。

相差约 **2000 倍**，数学结果却一样。LoRA 的实现就利用了这一点：算 $\mathbf{x}\mathbf{A}^\top\mathbf{B}^\top$ 时一定是先算 $\mathbf{x}\mathbf{A}^\top$（得到很窄的 $r$ 维），而不是先把 $\mathbf{A}^\top\mathbf{B}^\top$ 乘出来（见 [原文 9.5](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#95-算例五lora-的参数与计算)）。

### Q26. Python 里 `*` 和 `@` 有什么区别？

| 运算符 | 含义 | 数学符号 |
| --- | --- | --- |
| `a * b` | 逐元素乘（会广播） | $\odot$ |
| `a @ b` | 矩阵乘法 | $\mathbf{A}\mathbf{B}$ |

```python
a = torch.randn(5)
b = torch.randn(5, 1)
(a * b).shape   # (5, 5) —— 广播后的逐元素乘，不是点积
(a @ a).shape   # ()     —— 一维 @ 一维 = 点积，标量
```

很多 bug 来自把 `*` 当成 `@` 用：shape 恰好能广播时不会报错（又是 [原文 2.3](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#23-逐元素运算与广播) 的坑）。

## 2.7 Batched Matrix Multiplication

> **📖 原文 [2.7 Batched Matrix Multiplication](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#27-batched-matrix-multiplication)**
>
> 多头 Attention 中常见：
>
> $$
> \mathbf{Q} \in \mathbb{R}^{B \times N_h \times S_q \times D_h}
> $$
>
> $$
> \mathbf{K} \in \mathbb{R}^{B \times N_h \times S_k \times D_h}
> $$
>
> 转置最后两维后：

### Q27. 四维张量的矩阵乘法到底怎么算？前两维为什么叫 batch 维？

`torch.matmul` 的规则是：**只对最后两维做矩阵乘法，前面的维度当作“一摞矩阵”**。

```python
q = torch.randn(2, 8, 16, 64)   # (B, Nh, Sq, Dh)
k = torch.randn(2, 8, 32, 64)   # (B, Nh, Sk, Dh)
(q @ k.transpose(-2, -1)).shape # (2, 8, 16, 32) = (B, Nh, Sq, Sk)
```

等价于写两层循环：

```python
out = torch.empty(2, 8, 16, 32)
for b in range(2):
    for h in range(8):
        out[b, h] = q[b, h] @ k[b, h].T   # (16, 64) @ (64, 32) -> (16, 32)
```

一共 $2 \times 8 = 16$ 个互相独立的小矩阵乘法，所以前两维叫 batch 维。GPU 会把这 16 个一起算，这就是 batched GEMM。

### Q28. `k.transpose(-2, -1)` 里的 -2、-1 是什么意思？

负数索引从后往前数，和 Python 列表的 `a[-1]` 一样：-1 是最后一维，-2 是倒数第二维。`transpose(-2, -1)` 就是交换最后两维，`(B, Nh, Sk, Dh)` 变成 `(B, Nh, Dh, Sk)`。

用负数的好处是**不管前面有几个 batch 维都能用**，同一行代码对三维、四维张量都成立。另外，`.T` 在高维张量上已不推荐使用（它会反转所有维度），高维时请用 `.mT` 或 `transpose(-2, -1)`。

### Q29. `einsum` 是什么？和 `@` 比哪个好？

`einsum` 用字母给每一维命名，直接写出“哪些维度保留、哪些被求和”：

```python
torch.einsum("bhqd,bhkd->bhqk", q, k).shape   # (2, 8, 16, 32)
```

读法：输入里有 `d`、输出里没有 `d` → 沿 `d` 求和（归约维）；`b`、`h` 两边都有 → batch 维；`q`、`k` 是保留的外侧维度。

它和 `q @ k.transpose(-2, -1)` 结果相同，好处是**省去手动转置**、意图更清楚，坏处是新手不熟悉。性能上两者通常都会落到同一个 batched GEMM 上。读论文源码时 einsum 很常见，值得会读。

## 2.8 线性变换与仿射变换

> **📖 原文 [2.8 线性变换与仿射变换](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#28-线性变换与仿射变换)**
>
> 严格的线性变换满足：
>
> $$
> f(a\mathbf{x}+b\mathbf{y})
> = af(\mathbf{x})+bf(\mathbf{y})
> $$
>
> 矩阵乘法 $f(\mathbf{x})=\mathbf{W}\mathbf{x}$ 是线性变换。神经网络中所谓“Linear 层”通常还包含 bias：
>
> $$
> \mathbf{y}=\mathbf{W}\mathbf{x}+\mathbf{b}
> $$

### Q30. 线性变换和仿射变换的区别只是有没有 bias 吗？为什么要区分？

是的，区别就是 $+\mathbf{b}$。严格的线性变换必须满足 $f(\mathbf{0}) = \mathbf{0}$（把 $a=b=0$ 代入定义可得），而 $\mathbf{W}\mathbf{0} + \mathbf{b} = \mathbf{b} \ne \mathbf{0}$，所以带 bias 的不是线性变换。

工程上区分它是为了理解**哪些性质还成立**：

- 线性变换可以和别的线性变换合并、可以和缩放交换顺序；
- 仿射变换也能合并，但 bias 要跟着一起变换。

### Q31. “多个没有激活函数的线性层可以合并成一个”，能验证一下吗？

$$
\mathbf{W}_2(\mathbf{W}_1\mathbf{x} + \mathbf{b}_1) + \mathbf{b}_2 = (\mathbf{W}_2\mathbf{W}_1)\mathbf{x} + (\mathbf{W}_2\mathbf{b}_1 + \mathbf{b}_2)
$$

右边就是一个新的仿射层，权重 $\mathbf{W}_2\mathbf{W}_1$，bias $\mathbf{W}_2\mathbf{b}_1 + \mathbf{b}_2$：

```python
W1, b1 = torch.randn(5, 3), torch.randn(5)
W2, b2 = torch.randn(2, 5), torch.randn(2)
x = torch.randn(3)
torch.allclose(W2 @ (W1 @ x + b1) + b2,
               (W2 @ W1) @ x + (W2 @ b1 + b2), atol=1e-5)   # True
```

所以堆 100 层不带激活的线性层，表达能力和 1 层一样。中间插入 ReLU/GELU 这种非线性函数后，就合并不了了，网络才真正“变深”。

反过来，这个性质在推理优化里很有用：BatchNorm 在推理时是一个仿射变换，可以直接**融合进前面卷积层的权重**里，少做一次运算。

## 2.9 转置、单位矩阵与逆矩阵

> **📖 原文 [2.9 转置、单位矩阵与逆矩阵](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#29-转置单位矩阵与逆矩阵)**
>
> 转置交换行列：
>
> $$
> (\mathbf{A}^\top)_{ij} = A_{ji}
> $$
>
> 常用规则：
>
> $$
> (\mathbf{A}\mathbf{B})^\top
> = \mathbf{B}^\top\mathbf{A}^\top
> $$

### Q32. $(\mathbf{A}\mathbf{B})^\top = \mathbf{B}^\top\mathbf{A}^\top$ 为什么顺序要反过来？

从 shape 就能看出来必须反过来：$\mathbf{A}$ 是 $(M,K)$、$\mathbf{B}$ 是 $(K,N)$，$\mathbf{A}\mathbf{B}$ 是 $(M,N)$，转置后是 $(N,M)$。

- $\mathbf{B}^\top\mathbf{A}^\top$：$(N,K) @ (K,M) = (N,M)$ ✓
- $\mathbf{A}^\top\mathbf{B}^\top$：$(K,M) @ (N,K)$，内维 $M \ne N$ 根本乘不了 ✗

这条规则在 [原文 6.6](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#66-线性层的反向传播) 推线性层反向传播时会反复用到。记不住时就**拿 shape 去凑**，能乘得起来的那个顺序就是对的。

### Q33. 为什么“数值实现通常不显式求逆”？`inv(A) @ b` 和 `solve(A, b)` 差别有多大？

两个原因：**更慢**（求逆要算出整个 $n \times n$ 的逆矩阵，而解方程只需要一个向量）和**更不准**。

用一个出名的病态矩阵（10 阶 Hilbert 矩阵，条件数约 $1.6 \times 10^{13}$）做实验，真解全是 1：

```python
n = 10
A = torch.tensor([[1/(i+j+1) for j in range(n)] for i in range(n)], dtype=torch.float64)
x_true = torch.ones(n, dtype=torch.float64)
b = A @ x_true
(torch.linalg.inv(A) @ b - x_true).abs().max()    # 2.87e-03
(torch.linalg.solve(A, b) - x_true).abs().max()   # 3.55e-05
```

即使用 float64，显式求逆的误差也比直接求解大了约 80 倍。原文提到的 LU、QR、Cholesky 都是把 $\mathbf{A}$ 分解成好解的三角矩阵，再一步步代入，`torch.linalg.solve` 内部用的就是 LU 分解。

“条件数”的含义在 [原文 8.7](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#87-条件数输入误差会被放大多少) 会正式讲，这里先记住：条件数越大，矩阵越接近“不可逆”，误差越容易被放大。

## 2.10 线性相关、张成空间与秩

> **📖 原文 [2.10 线性相关、张成空间与秩](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#210-线性相关张成空间与秩)**
>
> 如果一组向量中某个向量能由其他向量线性组合得到，它们线性相关。矩阵的秩（rank）可以理解为独立方向的数量：
>
> $$
> \operatorname{rank}(\mathbf{A})
> \le \min(M,N)
> $$
>
> 秩低意味着矩阵中的信息存在冗余，可以由更小的两个矩阵近似：
>
> $$
> \mathbf{A} \approx
> \mathbf{U}\mathbf{V},\qquad
> \mathbf{U}\in\mathbb{R}^{M\times r},
> \mathbf{V}\in\mathbb{R}^{r\times N},
> \quad r \ll \min(M,N)
> $$

### Q34. 秩的直觉“独立方向的数量”怎么理解？能举个秩为 1 的例子吗？

最简单的秩 1 矩阵是一个列向量乘一个行向量（外积）：

```python
u = torch.randn(4, 1)
v = torch.randn(1, 6)
torch.linalg.matrix_rank(u @ v)            # 1
torch.linalg.matrix_rank(torch.randn(4, 6)) # 4 = min(4, 6)
```

`u @ v` 虽然有 $4 \times 6 = 24$ 个数，但每一行都是 $\mathbf{v}$ 的倍数（第 $i$ 行 $= u_i \mathbf{v}$），本质上只有“一个方向”的信息。存它其实只需要 $4 + 6 = 10$ 个数。

随机矩阵几乎总是“满秩”，秩等于 $\min(M,N)$。真实网络的权重和权重**更新量**往往接近低秩，这是下面 LoRA 能成立的前提。

### Q35. 参数量从 $MN$ 降到 $r(M+N)$，具体能省多少？

以 $M = N = 4096$ 为例，完整矩阵有 $4096^2 \approx 1678$ 万个参数：

| 秩 $r$ | $r(M+N)$ | 占原来的比例 |
| --- | --- | --- |
| 8 | 65,536 | 0.39% |
| 16 | 131,072 | 0.78% |
| 64 | 524,288 | 3.13% |

规律：比例 $= r(M+N)/(MN) = 2r/4096$，和 $r$ 成正比。$r$ 要远小于 $M$、$N$ 才划算；当 $r$ 接近 $M/2$ 时就和原矩阵一样大了。

## 2.11 特征值与特征向量

> **📖 原文 [2.11 特征值与特征向量](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#211-特征值与特征向量)**
>
> 对方阵 $\mathbf{A}$，如果存在非零向量 $\mathbf{v}$ 和标量 $\lambda$ 使：
>
> $$
> \mathbf{A}\mathbf{v}=\lambda\mathbf{v}
> $$
>
> 则 $\mathbf{v}$ 是特征向量，$\lambda$ 是对应特征值。直觉上，矩阵变换在这个方向上只缩放，不改变方向。

### Q36. 特征向量的“只缩放、不改变方向”怎么理解？

把矩阵 $\mathbf{A}$ 想成一个“变换”：输入一个向量，输出另一个向量。大部分向量变换后既变长短又变方向，但特征向量很特殊，变换后还在原来那条直线上，只是长度乘了 $\lambda$。

最直观的例子是对角矩阵：$\begin{bmatrix}1.1 & 0\\0 & 0.9\end{bmatrix}$ 的两个特征向量就是两个坐标轴方向，特征值分别是 1.1 和 0.9。

### Q37. 原文说特征值能解释“梯度经过多层 Jacobian 连乘时为什么爆炸或消失”，能演示一下吗？

用上面那个矩阵连乘：沿特征值 1.1 的方向每层放大 10%，沿 0.9 的方向每层缩小 10%：

```python
A = torch.tensor([[1.1, 0.0], [0.0, 0.9]])
v = torch.tensor([1.0, 1.0])
torch.linalg.matrix_power(A, 10) @ v   # tensor([2.5937, 0.3487])
torch.linalg.matrix_power(A, 50) @ v   # tensor([1.1739e+02, 5.1538e-03])
```

50 层以后，一个分量放大了 117 倍，另一个缩小到原来的 0.5%。特征值只比 1 大一点或小一点，连乘几十次就是指数级的差别。深层网络的反向传播就是许多 Jacobian 的连乘，这正是梯度爆炸 / 消失的数学根源，[原文 7.4](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#74-梯度消失与爆炸) 会接着讲。

## 2.12 奇异值分解（SVD）

> **📖 原文 [2.12 奇异值分解（SVD）](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#212-奇异值分解svd)**
>
> 任意实矩阵 $\mathbf{A}\in\mathbb{R}^{M\times N}$ 都可以写成：
>
> $$
> \mathbf{A} = \mathbf{U}\mathbf{\Sigma}\mathbf{V}^\top
> $$
>
> 其中：
>
> - $\mathbf{U}$ 的列是左奇异向量；
> - $\mathbf{V}$ 的列是右奇异向量；
> - $\mathbf{\Sigma}$ 对角线上的 $\sigma_1\ge\sigma_2\ge\cdots\ge0$ 是奇异值。

### Q38. SVD 和特征分解有什么区别？为什么说 SVD“任意矩阵都可以”？

| | 特征分解 | SVD |
| --- | --- | --- |
| 适用矩阵 | 方阵，而且不是所有方阵都行 | 任意 $M \times N$ 矩阵 |
| 结果 | $\mathbf{A} = \mathbf{Q}\mathbf{\Lambda}\mathbf{Q}^{-1}$ | $\mathbf{A} = \mathbf{U}\mathbf{\Sigma}\mathbf{V}^\top$ |
| 值 | 特征值可以是负数、复数 | 奇异值总是 $\ge 0$ 的实数，从大到小排好 |

神经网络的权重大多不是方阵（比如 $4096 \times 11008$），所以分析权重时用的几乎都是 SVD。可以把 SVD 理解成三步：$\mathbf{V}^\top$ 先旋转，$\mathbf{\Sigma}$ 沿各坐标轴缩放，$\mathbf{U}$ 再旋转一次。奇异值就是每个方向的缩放倍数。

### Q39. 截断 SVD 的“最优近似”有多好？误差怎么算？

Frobenius 范数下，丢掉的误差正好是**被丢掉的那些奇异值的平方和开根号**。造一个“本质秩为 2、加了一点噪声”的矩阵验证：

```python
M = torch.randn(64, 2) @ torch.randn(2, 48) + 0.01 * torch.randn(64, 48)
torch.linalg.svdvals(M)[:5]   # tensor([52.5966, 45.8707, 0.1390, 0.1349, 0.1309])

U, S, Vh = torch.linalg.svd(M, full_matrices=False)
r = 2
Mr = U[:, :r] @ torch.diag(S[:r]) @ Vh[:r]
torch.linalg.norm(M - Mr)           # 0.5370
S[r:].pow(2).sum().sqrt()           # 0.5370  ← 完全一致
torch.linalg.norm(M - Mr) / torch.linalg.norm(M)   # 0.0077，相对误差不到 1%
```

前两个奇异值是 52 和 46，第三个突然掉到 0.14，这个“断崖”就说明矩阵本质上是秩 2。看奇异值曲线有没有断崖，是判断一个权重矩阵能不能压缩的常用方法。

`full_matrices=False` 的意思是只算“瘦”版本的 $\mathbf{U}$、$\mathbf{V}$，否则 $\mathbf{U}$ 会是 $64 \times 64$，多出来的列用不上。

## 2.13 LoRA 的低秩更新

> **📖 原文 [2.13 LoRA 的低秩更新](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#213-lora-的低秩更新)**
>
> 对预训练权重 $\mathbf{W}_0\in\mathbb{R}^{d_{out}\times d_{in}}$，LoRA 冻结 $\mathbf{W}_0$，只训练低秩更新：
>
> $$
> \mathbf{W} = \mathbf{W}_0 + \Delta\mathbf{W}
> $$
>
> $$
> \Delta\mathbf{W}
> = \frac{\alpha}{r}\mathbf{B}\mathbf{A}
> $$
>
> 其中：

### Q40. $\mathbf{W}_0$ 冻结是什么意思？代码里怎么体现？

“冻结”就是不让它参与梯度更新：

```python
base = nn.Linear(4096, 4096)
base.weight.requires_grad_(False)   # W0 冻结：不算梯度、优化器不更新
```

`requires_grad=False` 后，反向传播不会为它算梯度，优化器也不会为它创建 Adam 状态。所以真正省下的是“**梯度 + 优化器状态**”，而前向时 $\mathbf{W}_0$ 照样要参与计算，这正是原文最后一段强调的。

### Q41. LoRA 里 $\mathbf{B}$ 和 $\mathbf{A}$ 是怎么初始化的？为什么 $\mathbf{B}$ 要全零？

原论文的做法是：$\mathbf{A}$ 用随机初始化，$\mathbf{B}$ 初始化为**全零**。这样一开始 $\Delta\mathbf{W} = \mathbf{B}\mathbf{A} = \mathbf{0}$，模型和预训练模型完全一样，微调从“不破坏原模型”的状态开始。

那 $\mathbf{B} = 0$ 会不会导致梯度也是 0、永远学不动？不会。$\partial L / \partial \mathbf{B}$ 依赖的是 $\mathbf{A}\mathbf{x}$（不为零），所以第一步 $\mathbf{B}$ 就有梯度；反过来 $\mathbf{A}$ 的梯度依赖 $\mathbf{B}$，第一步为零，等 $\mathbf{B}$ 动起来后才开始更新。如果两个都初始化为零，就真的谁也学不动了。

```python
class LoRALinear(nn.Module):
    def __init__(self, base: nn.Linear, r: int = 16, alpha: int = 32):
        super().__init__()
        self.base = base.requires_grad_(False)
        self.A = nn.Parameter(torch.randn(r, base.in_features) * 0.01)  # (r, din)
        self.B = nn.Parameter(torch.zeros(base.out_features, r))        # (dout, r)
        self.scale = alpha / r

    def forward(self, x):                         # x: (..., din)
        return self.base(x) + self.scale * (x @ self.A.T) @ self.B.T
```

注意 `forward` 里的括号：先 `x @ A.T` 得到 $r$ 维，再乘 `B.T`，就是 [原文 2.6](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#26-矩阵乘法是一组点积) 说的“结合顺序决定代价”。

### Q42. $\alpha / r$ 这个缩放因子有什么用？

它让你**换 $r$ 时不用重新调学习率**。$r$ 越大，$\mathbf{B}\mathbf{A}$ 是越多项的和，数值会随 $r$ 变大；除以 $r$ 把这个影响抵消掉。实践中常把 $\alpha$ 设成 $r$ 的 1～2 倍（比如 $r=16, \alpha=32$），之后改 $r$ 时保持 $\alpha/r$ 不变即可。

### Q43. “低秩只减少可训练参数，不自动消除基座权重和前向计算成本”，那 LoRA 到底省了什么？

逐项列出来就清楚了（以全参数 Adam 微调为对比）：

| 显存项 | 全参数微调 | LoRA |
| --- | --- | --- |
| 基座权重 $\mathbf{W}_0$ | 要存 | **照样要存** |
| 梯度 | 每个参数一份 | 只有 $\mathbf{A}$、$\mathbf{B}$ 有 |
| Adam 状态（两份） | 每个参数两份 | 只有 $\mathbf{A}$、$\mathbf{B}$ 有 |
| 前向激活 | 要存 | **照样要存** |
| 前向计算量 | $\mathbf{x}\mathbf{W}_0^\top$ | 多出两个小 GEMM |

所以 LoRA 省的是“梯度 + 优化器状态”这一大块（Adam 下约占全参数训练的 3/4，见 [原文 7.3](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#73-adam-的尺度自适应直觉)），基座和激活一点没省。想继续省就要叠加 QLoRA（把基座量化成 4bit）、梯度检查点（少存激活）等技术。

部署时两种选择：把 $\frac{\alpha}{r}\mathbf{B}\mathbf{A}$ 加进 $\mathbf{W}_0$ 合并成一个矩阵（推理零额外开销，但每换一个 adapter 都要重新合并），或者保持分开（多两个小 GEMM，但可以在同一个服务里给不同用户切换不同 adapter）。

## 第2章系列目录

- [导读与易错点速查](/posts/ai-infra/ch2-00-guide/)
- **（一）符号、张量与线性代数**（本篇）
- [（二）GEMM 分块与概率论](/posts/ai-infra/ch2-02-gemm-probability/)
- [（三）Softmax、交叉熵与反向传播](/posts/ai-infra/ch2-03-softmax-backprop/)
- [（四）优化、归一化、数值精度与综合算例](/posts/ai-infra/ch2-04-optimization-numerics/)

其他章节：

- [第1章 编程语言基础：导读与易错点速查](/posts/ai-infra/ch1-00-guide/)

