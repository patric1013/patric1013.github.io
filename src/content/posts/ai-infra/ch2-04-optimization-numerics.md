---
title: "AI Infra 第2章笔记（四）：优化、归一化、数值精度与综合算例"
published: 2026-11-03T18:00:00
description: "这一篇的主线：梯度怎么用（SGD → 动量 → Adam，以及 Adam 为什么吃显存）→ 梯度为什么会坏（Jacobian 连乘）以及怎么救（残差、初始化、归一化、裁剪）→ 浮点数为什么会骗人（精度 ≠ 范围、大数…"
tags: ["AI Infra", "数学", "混合精度", "优化器"]
category: "AI Infra 学习笔记"
draft: false
---

这一篇的主线：**梯度怎么用（SGD → 动量 → Adam，以及 Adam 为什么吃显存）→ 梯度为什么会坏（Jacobian 连乘）以及怎么救（残差、初始化、归一化、裁剪）→ 浮点数为什么会骗人（精度 ≠ 范围、大数吃小数、减法抵消）→ 混合精度怎么在速度和精度之间取舍 → 用七个算例把前面所有数学换算成 shape、FLOPs 和显存。** 第三部分的数值实验最多，结论也最反直觉。

> [!NOTE]
> 本篇对应原文 [AIInfraGuide · 第2章：数学基础](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/) 的 §7–9（作者 [caomaolufei](https://github.com/caomaolufei/AIInfraGuide)，MIT 许可）。每节开头的引用块摘自原文，文中的“原文 x.y”可以点击跳转到原文对应小节。问答是按第1章的提问思路整理的，代码输出均为实际运行结果。

## 7.1 全量梯度、SGD 与 mini-batch

> **📖 原文 [7.1 全量梯度、SGD 与 mini-batch](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#71-全量梯度sgd-与-mini-batch)**
>
> 经验风险：
>
> $$
> L(\theta)=\frac{1}{N}\sum_{i=1}^{N}L_i(\theta)
> $$
>
> 全量梯度每步使用所有样本，成本高。mini-batch 梯度：
>
> $$
> \hat{g}
> = \frac{1}{B}\sum_{i\in\mathcal{B}}
> \nabla_\theta L_i(\theta)
> $$

### Q1. mini-batch 梯度是全量梯度的“随机估计”，batch 越大越好吗？

不一定。batch 变大的好处和代价：

| | batch 大 | batch 小 |
| --- | --- | --- |
| 梯度噪声 | 小（方差约按 $1/B$ 下降） | 大 |
| 硬件利用率 | 高，GEMM 更“胖”（[原文 3.3](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#33-一个简化的算术强度估算)） | 低 |
| 激活显存 | 大 | 小 |
| 每个 epoch 的更新次数 | 少 | 多 |

噪声不全是坏事：一定的梯度噪声有助于跳出尖锐的局部极小值，有观点认为这和泛化能力有关。实践中把 batch 放大 $k$ 倍时，常按“线性缩放规则”把学习率也放大约 $k$ 倍，并配合 warmup，但这条规则在 batch 极大时会失效。

### Q2. 原文说 SGD 是 $\theta - \eta\hat{g}$，那 `torch.optim.SGD` 默认就是这个吗？

默认（`momentum=0`）就是这个最朴素的版本。实际训练很少直接用纯 SGD，通常至少会开 `momentum=0.9`（见下一节），视觉任务里 SGD + 动量 + 余弦学习率仍然很常见，语言模型基本都用 AdamW。

## 7.2 动量

> **📖 原文 [7.2 动量](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#72-动量)**
>
> 一种常见动量形式：
>
> $$
> \mathbf{v}_t
> = \beta\mathbf{v}_{t-1}+(1-\beta)\mathbf{g}_t
> $$
>
> $$
> \theta_{t+1}
> = \theta_t-\eta\mathbf{v}_t
> $$
>
> 动量对历史梯度做指数加权平均，减小短期噪声，在持续一致的方向上加速。

### Q3. 动量公式里的 $\beta$ 一般取多少？“指数加权平均”是什么意思？

常用 $\beta = 0.9$。把递推式展开：

$$
\mathbf{v}_t = (1-\beta)\left(\mathbf{g}_t + \beta\mathbf{g}_{t-1} + \beta^2\mathbf{g}_{t-2} + \cdots\right)
$$

越久远的梯度权重越小，按 $\beta^k$ 指数衰减。$\beta = 0.9$ 时，大约相当于对**最近 10 步**（$1/(1-\beta)$）的梯度求平均。

直觉：梯度在某个方向上来回震荡时，正负抵消，动量把它“平滑”掉；在某个方向上一直同号时，动量越积越大，相当于加速。像一个有惯性的小球滚下山坡。

### Q4. PyTorch 的 SGD 动量和原文公式一样吗？

不完全一样。原文写的是 $\mathbf{v}_t = \beta\mathbf{v}_{t-1} + (1-\beta)\mathbf{g}_t$，PyTorch 实现的是 $\mathbf{v}_t = \beta\mathbf{v}_{t-1} + \mathbf{g}_t$（没有 $1-\beta$）。两者只差一个常数倍 $1/(1-\beta)$，相当于 PyTorch 版本的有效学习率大了 10 倍（$\beta = 0.9$ 时）。

这就是为什么**照着论文复现时，学习率要看清楚对应的是哪种实现**。原文说“一种常见动量形式”，就是在提醒这一点。

## 7.3 Adam 的尺度自适应直觉

> **📖 原文 [7.3 Adam 的尺度自适应直觉](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#73-adam-的尺度自适应直觉)**
>
> Adam 同时维护一阶矩和二阶矩的指数移动平均：
>
> $$
> \mathbf{m}_t
> = \beta_1\mathbf{m}_{t-1}
> +(1-\beta_1)\mathbf{g}_t
> $$
>
> $$
> \mathbf{v}_t
> = \beta_2\mathbf{v}_{t-1}
> +(1-\beta_2)\mathbf{g}_t^2
> $$
>
> 偏差修正后：

### Q5. Adam 的两个矩 $\mathbf{m}$ 和 $\mathbf{v}$ 分别在干什么？

- $\mathbf{m}$（一阶矩）：梯度的指数平均，就是上一节的动量，决定**往哪走**；
- $\mathbf{v}$（二阶矩）：梯度**平方**的指数平均，衡量这个参数的梯度通常有多大；
- 更新量 $\frac{\hat{\mathbf{m}}}{\sqrt{\hat{\mathbf{v}}} + \epsilon}$：用梯度的典型大小去除梯度，于是每个参数的步长都被“标准化”了。

效果是：梯度一直很大的参数，步子不会太大；梯度一直很小的参数（比如很少出现的词的 embedding），步子也不会太小。这就是“尺度自适应”。常用 $\beta_1 = 0.9, \beta_2 = 0.999$。

### Q6. “偏差修正”是在修正什么？

$\mathbf{m}$ 和 $\mathbf{v}$ 都从 0 开始。第一步时 $\mathbf{m}_1 = (1 - 0.9)\mathbf{g}_1 = 0.1\mathbf{g}_1$，比真实梯度小了 10 倍，因为“历史”全是 0，把平均值往 0 拉了。

偏差修正就是除以 $1 - \beta^t$ 把它放大回来：$\hat{\mathbf{m}}_1 = \mathbf{m}_1 / (1 - 0.9^1) = \mathbf{g}_1$。随着 $t$ 增大，$\beta^t \to 0$，修正项趋于 1，后期几乎不起作用。它只影响训练最开始的几十到几千步。

### Q7. “Adam 为每个参数保存额外状态”，具体多占多少显存？

PyTorch 的 Adam 给每个参数存两个和参数一样大的张量：

```python
opt = torch.optim.Adam(model.parameters())
# 一次 step 后：
opt.state[model.weight].keys()   # step, exp_avg (m), exp_avg_sq (v)
```

在常见的混合精度训练中，每个参数大约需要：

| 项 | 字节/参数 |
| --- | --- |
| FP16/BF16 参数 | 2 |
| FP16/BF16 梯度 | 2 |
| FP32 master 权重（[原文 8.3](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#83-舍入与机器精度)） | 4 |
| FP32 Adam $\mathbf{m}$ | 4 |
| FP32 Adam $\mathbf{v}$ | 4 |
| **合计** | **16** |

7B 参数的模型：$7 \times 10^9 \times 16 \approx 112$ GB，还没算激活。单张 80 GB 的卡根本放不下。其中优化器相关的部分（master 权重 + m + v）占了 12 字节，是参数本身的 6 倍。这就是原文说的“催生了 ZeRO 等分片技术”：把这 12 字节切开，分散到多张卡上，模块三会专门讲。

### Q8. Adam 和 AdamW 有什么区别？

区别在**权重衰减怎么加**：

- Adam 的 `weight_decay`：把 $\lambda\theta$ 加到梯度上，然后这一项也会被 $\sqrt{\hat{\mathbf{v}}}$ 除，导致梯度大的参数衰减反而变弱；
- AdamW：把权重衰减从梯度里拿出来，直接在更新时减去 $\eta\lambda\theta$，不受自适应缩放影响。

AdamW 的效果通常更好、超参数更好调，现在训练 Transformer 基本默认用 `torch.optim.AdamW`。

## 7.4 梯度消失与爆炸

> **📖 原文 [7.4 梯度消失与爆炸](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#74-梯度消失与爆炸)**
>
> 深层复合函数的梯度包含许多 Jacobian 连乘：
>
> $$
> \frac{\partial L}{\partial \mathbf{h}_0}
> = \frac{\partial L}{\partial \mathbf{h}_L}
> \prod_{l=1}^{L}
> \frac{\partial \mathbf{h}_l}
> {\partial \mathbf{h}_{l-1}}
> $$
>
> 若这些变换在相关方向上的尺度长期小于 1，梯度指数衰减；长期大于 1，梯度指数增长。

### Q9. “若变换的尺度长期小于 1，梯度指数衰减”，能做个实验看看吗？

用 20 层没有激活函数的线性层模拟，权重的尺度分别设成 0.5、1.0、1.5 倍“刚好合适”：

```python
h = torch.randn(512, 1024)
for scale in [0.5, 1.0, 1.5]:
    t = h.clone()
    for _ in range(20):
        t = t @ (torch.randn(1024, 1024) * scale / math.sqrt(1024))
    print(scale, t.std())
# 0.5   9.48e-07   ← 每层缩小一半，20 层后几乎为 0
# 1.0   1.007      ← 保持稳定
# 1.5   3.28e+03   ← 每层放大 1.5 倍，20 层后变成几千
```

$0.5^{20} \approx 10^{-6}$，$1.5^{20} \approx 3300$，和实验完全吻合。反向传播时梯度经过同样的矩阵连乘，所以梯度也会同样地消失或爆炸。这就是 [原文 2.11](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#211-特征值与特征向量) 讲的特征值连乘在实际网络里的样子。

### Q10. 训练时怎么判断是梯度消失还是梯度爆炸？

看**各层梯度范数**，而不是只看 loss：

```python
for name, p in model.named_parameters():
    if p.grad is not None:
        print(f"{name:40s} {p.grad.norm():.3e}")
```

- 前几层（靠近输入）的梯度范数比后几层小几个数量级 → 梯度消失；
- 某一步梯度范数突然飙升几百倍，随后 loss 变成 `nan` → 梯度爆炸。

训练时把全局梯度范数记录到 TensorBoard / wandb 里，是排查训练不稳定最有效的手段之一。`clip_grad_norm_` 的返回值就是裁剪前的全局范数，可以顺手记录下来（见 [原文 7.9](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#79-梯度裁剪)）。

## 7.5 残差连接为什么帮助梯度传播

> **📖 原文 [7.5 残差连接为什么帮助梯度传播](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#75-残差连接为什么帮助梯度传播)**
>
> 残差块：
>
> $$
> \mathbf{y}=\mathbf{x}+F(\mathbf{x})
> $$
>
> 其 Jacobian：
>
> $$
> \frac{\partial \mathbf{y}}{\partial \mathbf{x}}
> = \mathbf{I}+\frac{\partial F}{\partial \mathbf{x}}
> $$

### Q11. 残差连接 $\mathbf{y} = \mathbf{x} + F(\mathbf{x})$ 里的“恒等路径”为什么能救梯度？

看 Jacobian：$\frac{\partial \mathbf{y}}{\partial \mathbf{x}} = \mathbf{I} + \frac{\partial F}{\partial \mathbf{x}}$。哪怕 $F$ 的 Jacobian 很小（接近 0），总的 Jacobian 也至少是 $\mathbf{I}$，梯度不会被乘成 0。

用上面同样的实验对比（每层 scale = 0.5，原本会消失）：

```python
t, t2 = h.clone(), h.clone()
for _ in range(20):
    W = torch.randn(1024, 1024) * 0.5 / math.sqrt(1024)
    t = t @ W                 # 普通堆叠
    t2 = t2 + t2 @ W          # 残差：x + F(x)
print(t.std(), t2.std())     # 9.55e-07   9.35
```

普通堆叠后信号消失到 $10^{-6}$，残差版本没有消失。但注意，残差版本的标准差**增长**到了 9.35：每一层都在 $\mathbf{x}$ 上加了一点东西，方差逐层累加（约 $1.25^{10} \approx 9.3$）。这正是原文说的“不保证永远不会爆炸”。

### Q12. 那残差连接导致的数值逐层变大怎么处理？

这就是 Transformer 里“残差 + 归一化”总是成对出现的原因。两种放法：

- **Post-LN**（原始 Transformer）：$\mathbf{y} = \operatorname{LN}(\mathbf{x} + F(\mathbf{x}))$，每层输出都被拉回标准尺度，但深层时训练不稳定，需要 warmup；
- **Pre-LN**（GPT-2 之后的主流）：$\mathbf{y} = \mathbf{x} + F(\operatorname{LN}(\mathbf{x}))$，归一化放在分支里，主干上的恒等路径完全不受干扰，深层网络更好训练。

此外，GPT-2 等模型还会把残差分支最后一层的初始化按 $1/\sqrt{\text{层数}}$ 缩小，控制累加的增长。第3章 Transformer 的 LayerNorm 一节会详细讲。

## 7.6 初始化与方差传播

> **📖 原文 [7.6 初始化与方差传播](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#76-初始化与方差传播)**
>
> 考虑：
>
> $$
> y=\sum_{i=1}^{n}w_i x_i
> $$
>
> 若 $w_i,x_i$ 独立、均值为 0：
>
> $$
> \operatorname{Var}(y)
> = n\operatorname{Var}(w)\operatorname{Var}(x)
> $$

### Q13. $\operatorname{Var}(y) = n\operatorname{Var}(w)\operatorname{Var}(x)$ 这个公式怎么来的？能验证吗？

$y$ 是 $n$ 项 $w_i x_i$ 之和。每项的方差是 $\operatorname{Var}(w)\operatorname{Var}(x)$（均值为 0 且独立时成立），$n$ 项独立求和方差相加（[原文 4.6](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#46-方差与标准差)），所以是 $n$ 倍：

```python
for n in [64, 1024]:
    w = torch.randn(n, 4096)
    x = torch.randn(n, 4096)
    print(n, (w * x).sum(0).var())                         # 63.9 / 1022.2 ≈ n
    w2 = torch.randn(n, 4096) / math.sqrt(n)               # Var(w) = 1/n
    print(n, (w2 * x).sum(0).var())                        # 1.03 / 1.02 ≈ 1
```

不缩放时输出方差正好约等于 $n$；把权重的标准差设成 $1/\sqrt{n}$（方差 $1/n$）后，输出方差回到 1。这就是 Xavier/Kaiming 初始化的核心思想。

### Q14. Xavier 和 Kaiming 初始化有什么区别？什么时候用哪个？

| 初始化 | 方差 | 适用 |
| --- | --- | --- |
| Xavier（Glorot） | $\frac{2}{n_{in} + n_{out}}$ | tanh、sigmoid 这类关于 0 对称的激活 |
| Kaiming（He） | $\frac{2}{n_{in}}$ | ReLU 系列 |

Kaiming 多了一个 2 倍，是因为 ReLU 把一半的输入置 0，方差减半，要补回来。PyTorch 的 `nn.Linear` 默认用的是 `kaiming_uniform_` 的一个变体。大语言模型通常直接用 `normal(0, 0.02)` 这样的固定小值，再配合 Pre-LN 和残差缩放。

## 7.7 LayerNorm

> **📖 原文 [7.7 LayerNorm](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#77-layernorm)**
>
> 对一个 token 的隐藏向量 $\mathbf{x}\in\mathbb{R}^{H}$：
>
> $$
> \mu=\frac{1}{H}\sum_{i=1}^{H}x_i
> $$
>
> $$
> \sigma^2=\frac{1}{H}
> \sum_{i=1}^{H}(x_i-\mu)^2
> $$
>
> $$
> \operatorname{LayerNorm}(x_i)
> = \gamma_i
> \frac{x_i-\mu}{\sqrt{\sigma^2+\epsilon}}
> +\beta_i
> $$

### Q15. LayerNorm 和 BatchNorm 的区别是什么？为什么 Transformer 用 LayerNorm？

区别在**沿哪个维度算均值和方差**：

| | 归一化的维度 | 依赖 batch 内其他样本 |
| --- | --- | --- |
| BatchNorm | 对每个通道，沿 batch（和空间）维 | 是 |
| LayerNorm | 对每个 token，沿隐藏维 $H$ | 否 |

Transformer 用 LayerNorm 的原因正是原文说的“不依赖 batch”：

- 序列长度不一，padding 位置会污染 batch 统计；
- 自回归推理时 batch 可能只有 1，BatchNorm 的统计没有意义；
- 训练和推理行为一致（BatchNorm 推理时要用训练期的滑动平均，两者不一致）。

CNN 里 BatchNorm 仍然常用，所以你做视觉 backbone 时两种都会碰到。

### Q16. $\gamma$ 和 $\beta$ 有什么用？既然要归一化，为什么又要缩放回去？

归一化把每个 token 强制变成均值 0、方差 1，但这不一定是网络想要的分布。$\gamma$（缩放）和 $\beta$（平移）是可学习参数，让网络自己决定最终的尺度和偏移。如果网络学到 $\gamma = \sigma$、$\beta = \mu$，就能把归一化完全“撤销”。

所以归一化的真正作用不是固定分布，而是**把尺度和方向解耦**：方向由前面的层决定，尺度由 $\gamma$ 单独控制，优化起来更容易。初始化时 $\gamma = 1, \beta = 0$。

### Q17. “朴素实现多次读写 HBM”，LayerNorm 到底要读几遍数据？

按公式朴素地写：

1. 读一遍 $\mathbf{x}$，求均值 $\mu$；
2. 再读一遍 $\mathbf{x}$，求方差 $\sigma^2$；
3. 再读一遍 $\mathbf{x}$，计算归一化并写回。

三遍读、一遍写。LayerNorm 本身计算量很小，几乎全是访存时间，是典型的 memory-bound 算子（[原文 3.3](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#33-一个简化的算术强度估算)），所以读几遍直接决定速度。

优化方法：用 Welford 算法（[原文 8.6](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#86-灾难性消减)）在一遍读取中同时算出均值和方差，再一遍读取做归一化，变成两遍读、一遍写。如果 $H$ 不太大，一行数据能整个放进寄存器或共享内存，就只需要从 HBM 读一遍。

## 7.8 RMSNorm

> **📖 原文 [7.8 RMSNorm](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#78-rmsnorm)**
>
> RMSNorm 不减均值，只按均方根缩放：
>
> $$
> \operatorname{RMS}(\mathbf{x})
> = \sqrt{\frac{1}{H}\sum_i x_i^2+\epsilon}
> $$
>
> $$
> \operatorname{RMSNorm}(x_i)
> = \gamma_i\frac{x_i}{\operatorname{RMS}(\mathbf{x})}
> $$
>
> 它减少了均值相关计算，但“公式更少”不必然按同比例提升端到端速度，实际收益取决于融合、访存和模型整体瓶颈。

### Q18. RMSNorm 比 LayerNorm 少了什么？效果会变差吗？

少了两样：**不减均值**、**没有 $\beta$**。只用均方根缩放：

```python
x = torch.randn(2, 8)
rms = x / torch.sqrt(x.pow(2).mean(-1, keepdim=True) + 1e-6)
torch.allclose(rms, nn.RMSNorm(8, eps=1e-6)(x), atol=1e-6)   # True（γ 初始化为 1）
```

实践中效果和 LayerNorm 相当，LLaMA、Qwen 等主流大模型都用 RMSNorm。它的出发点是：LayerNorm 起作用的主要是“缩放”，“减均值”的贡献不大。

### Q19. “公式更少”为什么不一定按同比例提速？

因为 RMSNorm 是 memory-bound 算子，时间主要花在读写 HBM 上，而不是计算上。少一次减均值省掉的是计算，读写次数基本没变。

而且在整个模型里，归一化层的耗时本来就只占很小一部分，GEMM 和 Attention 才是大头。把一个只占 3% 时间的算子加速 20%，端到端只快 0.6%。这也是第1章 原文 3.8 讲的“先找热点再优化”。

## 7.9 梯度裁剪

> **📖 原文 [7.9 梯度裁剪](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#79-梯度裁剪)**
>
> 全局 L2 范数裁剪：
>
> $$
> \mathbf{g}\leftarrow
> \mathbf{g}\cdot
> \min\left(1,\frac{c}{\lVert\mathbf{g}\rVert_2+\epsilon}\right)
> $$
>
> 当梯度范数超过阈值 $c$ 时，整体按比例缩小，方向不变。分布式训练中，若梯度被分片，计算全局范数需要跨 rank 汇总局部平方和：
>
> $$
> \lVert\mathbf{g}\rVert_2
> = \sqrt{\sum_r\sum_{i\in r}g_i^2}
> $$

### Q20. 梯度裁剪的公式能验证一下吗？`clip_grad_norm_` 返回的是什么？

```python
p = nn.Parameter(torch.zeros(3))
p.grad = torch.tensor([3., 4., 0.])            # 范数 = 5
total = torch.nn.utils.clip_grad_norm_([p], max_norm=1.0)
total        # tensor(5.)                        ← 返回裁剪前的范数
p.grad       # tensor([0.6000, 0.8000, 0.0000])  ← 乘了 1/5，方向不变
p.grad.norm()  # tensor(1.0000)
```

梯度从 `[3, 4, 0]` 缩成 `[0.6, 0.8, 0]`，方向不变，长度变成 1。返回值是裁剪**前**的全局范数，很适合记录到日志里监控训练。

### Q21. 为什么是“全局”范数，而不是每个参数各自裁剪？

逐参数裁剪会**改变梯度的方向**：一个参数被缩小、另一个没被缩小，合起来的方向就变了。全局裁剪把所有参数的梯度拼成一个大向量统一缩放，方向保持不变，只是限制了一步最多走多远。

这样在“大部分时候不干预，偶尔出现异常大梯度时兜底”，是 Transformer 训练的标配，通常 `max_norm=1.0`。

### Q22. 分布式训练时算全局范数为什么需要通信？

模型并行或 ZeRO 下，每个 rank 只持有一部分参数的梯度。全局范数 $\sqrt{\sum_r \sum_{i \in r} g_i^2}$ 需要所有 rank 的平方和：

1. 每个 rank 先算自己那部分的平方和（一个标量）；
2. AllReduce 求和；
3. 开根号得到全局范数，每个 rank 用同一个系数缩放自己的梯度。

注意是先汇总**平方和**再开根号，而不是各自开根号再相加，否则数学上不对。这和 [原文 9.6](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#96-算例六分布式均值为什么要加权) “分布式均值要汇总 sum 和 count”是同一个道理：**跨 rank 合并统计量时，要合并能直接相加的那种形式**。

## 8.1 浮点数是有限集合

> **📖 原文 [8.1 浮点数是有限集合](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#81-浮点数是有限集合)**
>
> 典型二进制浮点数表示为：
>
> $$
> (-1)^s \times \text{significand}
> \times 2^{\text{exponent}}
> $$
>
> 位数被分给符号、指数和尾数（有效数字）。指数位决定动态范围，尾数位决定相邻可表示数的精细程度。

### Q23. 浮点数为什么是“有限集合”？32 位能表示多少个数？

32 位最多只有 $2^{32} \approx 43$ 亿种组合，而实数有无穷多个。所以绝大多数实数都只能用“离它最近的那个可表示数”来近似。

“指数位决定范围，尾数位决定精度”可以类比科学计数法 $1.234 \times 10^5$：指数（5）决定数量级能有多大，尾数（1.234）的位数决定同一数量级内能分多细。

### Q24. 各种格式的最大值、最小值、精度具体是多少？

用 `torch.finfo` 直接查：

```python
for dt in [torch.float32, torch.float16, torch.bfloat16]:
    fi = torch.finfo(dt)
    print(dt, fi.max, fi.tiny, fi.eps)
```

| 格式 | 最大值 | 最小正规数 | eps（1 附近的间距） |
| --- | --- | --- | --- |
| FP32 | $3.40 \times 10^{38}$ | $1.18 \times 10^{-38}$ | $1.19 \times 10^{-7}$ |
| FP16 | **65504** | $6.10 \times 10^{-5}$ | $9.77 \times 10^{-4}$ |
| BF16 | $3.39 \times 10^{38}$ | $1.18 \times 10^{-38}$ | $7.81 \times 10^{-3}$ |

一眼看出：BF16 的范围和 FP32 几乎一样（指数位都是 8 位），但精度比 FP16 还差 8 倍；FP16 精度更好，但最大值只有 65504。

### Q25. TF32 是什么？我代码里没写过 TF32，会用到吗？

TF32 不是一种存储格式，而是 NVIDIA Ampere（A100）及以后的 GPU 上 Tensor Core 的一种**计算模式**：输入存储还是 FP32，但矩阵乘法时只用 10 位尾数去乘，结果用 FP32 累加。速度比真正的 FP32 矩阵乘快很多倍，精度略降。

你可能已经在不知不觉中用到了：

```python
torch.backends.cuda.matmul.allow_tf32   # 矩阵乘默认 False（PyTorch 1.12 起）
torch.backends.cudnn.allow_tf32         # 卷积默认 True
torch.set_float32_matmul_precision("high")   # 推荐写法：允许 FP32 矩阵乘用 TF32
```

所以如果你在 A100 上训练 CNN，即使全是 FP32，卷积实际也是用 TF32 算的。做数值对比实验时（比如对比 CPU 和 GPU 结果），要记得这一点。

## 8.2 精度和动态范围是两件事

> **📖 原文 [8.2 精度和动态范围是两件事](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#82-精度和动态范围是两件事)**
>
> FP16 尾数字段比 BF16 多，因此同一数量级下通常能表示更细的差别；但 FP16 指数位少，最大有限值只有 65504，非零数的范围也窄得多。

### Q26. 精度和动态范围是两件事，能用例子说明吗？

看“一个数加 1 之后还能不能区分出来”：

```python
for dt in [torch.float16, torch.bfloat16]:
    for base in [1.0, 256.0, 2048.0]:
        print(dt, base, (torch.tensor(base, dtype=dt) + 1).item())
# float16   1.0 → 2.0     256.0 → 257.0    2048.0 → 2048.0  ← 2049 表示不了
# bfloat16  1.0 → 2.0     256.0 → 256.0    2048.0 → 2048.0  ← 257 就表示不了
```

- **精度**：BF16 在 256 附近相邻两个可表示数的间距是 2，所以 256 + 1 被舍回 256；FP16 到 2048 附近才出现这个问题。FP16 精度更好。
- **范围**：但 $300^2 = 90000$ 超过了 FP16 的最大值，变成 `inf`；BF16 算出 90112（不精确，但没溢出）。BF16 范围更大。

```python
torch.tensor(300., dtype=torch.float16) ** 2    # inf
torch.tensor(300., dtype=torch.bfloat16) ** 2   # 90112.
```

训练里**溢出是致命的**（`inf` 会变成 `nan`，一步就毁掉整个模型），而精度差一点只是让结果有些噪声。所以大模型训练更偏爱 BF16。

## 8.3 舍入与机器精度

> **📖 原文 [8.3 舍入与机器精度](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#83-舍入与机器精度)**
>
> 很多十进制小数不能被二进制浮点精确表示：
>
> ```python
> >>> 0.1 + 0.2 == 0.3
> False
> ```
>
> 浮点数间距随数值绝对值增大而增大。若 $|\delta|$ 远小于当前 $x$ 附近的可表示间距：
>
> $$
> \operatorname{fl}(x+\delta)=x
> $$

### Q27. “优化器常保留 FP32 master weights”是为什么？能演示一下更新被吞掉吗？

参数更新量通常很小（学习率 $10^{-4}$ 乘以梯度），而参数本身在 1 附近。BF16 在 1 附近的间距是 0.0078：

```python
w = torch.tensor(1.0, dtype=torch.bfloat16)
(w - torch.tensor(1e-3, dtype=torch.bfloat16)).item()   # 1.0  ← 更新被完全吞掉
w16 = torch.tensor(1.0, dtype=torch.float16)
(w16 - torch.tensor(1e-4, dtype=torch.float16)).item()  # 1.0  ← FP16 也一样
```

更新量小于间距的一半，加上去后又被舍入回原值，**这一步训练等于白做**。如果每一步都这样，模型就永远不会动。

解决办法：额外保存一份 FP32 的权重（master weights），更新在 FP32 上做（FP32 在 1 附近的间距是 $1.2 \times 10^{-7}$，能记住很小的更新），然后再转成 BF16 给前向和反向用。这就是 [原文 7.3](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#73-adam-的尺度自适应直觉) 显存表里“FP32 master 权重 4 字节”的来源。

## 8.4 非结合性和归约顺序

> **📖 原文 [8.4 非结合性和归约顺序](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#84-非结合性和归约顺序)**
>
> 由于每步都舍入：
>
> $$
> \operatorname{fl}(\operatorname{fl}(a+b)+c)
> \ne
> \operatorname{fl}(a+\operatorname{fl}(b+c))
> $$
>
> 并行归约使用树形加法，不同线程数、block 划分和原子操作顺序可能产生略不同结果。非确定性不一定代表实现错误，但必须在可接受误差内，并符合任务对复现性的要求。

### Q28. 原文练习 6：对 `[1, 1e-3, 1e-3, ...]` 求和，不同精度和顺序下误差有多大？

1 后面跟 1 万个 0.001，真实和是 11。用 Python 循环逐个累加（模拟最朴素的串行求和）：

```python
for dt in [torch.float16, torch.bfloat16, torch.float32]:
    d = torch.tensor(1e-3, dtype=dt)
    acc = torch.tensor(1.0, dtype=dt)          # 大数在前
    for _ in range(10000): acc = acc + d
    small_first = torch.tensor(0.0, dtype=dt)  # 小数先加，最后加 1
    for _ in range(10000): small_first = small_first + d
    small_first = small_first + 1
    print(dt, acc.item(), small_first.item())
# float16    4.0      5.0      ← 真实值 11
# bfloat16   1.0      1.5
# float32    11.0008  11.0004
```

BF16 从 1 开始加，一次都没加上去（每次 0.001 都被吞掉，见上一节）；FP16 加到 4 左右也停住了（4 附近的间距是 0.0039，0.001 小于它的一半）。调整顺序（先把小数加在一起）有所改善，但仍然错得离谱，因为加到一定程度后，累加器本身也变成了“大数”。

### Q29. 那 PyTorch 里对 FP16 张量直接 `.sum()` 也会这么离谱吗？

不会：

```python
v = torch.cat([torch.ones(1), torch.full((10000,), 1e-3)])
v.half().sum()       # 11.0078
v.bfloat16().sum()   # 11.0
```

因为 PyTorch 的 `sum` 内部用 **FP32 累加器**，而且是分块/树形求和，最后才转回 FP16。这就是原文列的“提高归约精度的手段”中的前两条：更高精度累加、树形求和。

教训：**手写 Kernel 时累加器的类型必须显式设为 FP32**。如果偷懒用了输入的 FP16 类型做累加器，长向量的归约就会出现上面那种严重错误，而且不会报错。

## 8.5 溢出、下溢和非有限值

> **📖 原文 [8.5 溢出、下溢和非有限值](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#85-溢出下溢和非有限值)**
>
> - **overflow**：结果绝对值超过最大有限值，常变成 $\pm\infty$；
> - **underflow**：结果太接近 0，进入 subnormal 或舍入为 0；
> - `NaN`：如 $0/0$、$\infty-\infty$ 或非法运算产生；
> - `inf` 参与后续运算可能迅速传播为 `NaN`。
>
> 排查 loss 变成 `NaN` 时，应找到**第一个**非有限张量，而不是只盯最终 loss。常见源头有过大 logits、除零、负数开方、无效 mask、梯度溢出和错误的自定义 Kernel 边界。

### Q30. `NaN` 都是怎么产生的？能列一下吗？

```python
torch.tensor(float("inf")) - torch.tensor(float("inf"))   # nan
torch.tensor(0.) / torch.tensor(0.)                       # nan
torch.sqrt(torch.tensor(-1.))                             # nan
torch.softmax(torch.tensor([float("-inf")] * 3), 0)       # tensor([nan, nan, nan])
```

最后一个在 Attention 里很常见：如果某一行的 mask **全部**被屏蔽（全是 $-\infty$），Softmax 的分母是 0，整行变成 `nan`。比如一个 batch 里某条样本全是 padding。FlashAttention 等实现会专门处理这种情况。

### Q31. “应该找第一个非有限张量”，具体怎么找？

两个常用工具：

```python
# 方法一：让 autograd 在反向时报出第一个产生 nan 的算子（会变慢，只在调试时开）
torch.autograd.set_detect_anomaly(True)

# 方法二：给每个模块挂 hook，前向时检查输出
def check(name):
    def hook(module, inp, out):
        if isinstance(out, torch.Tensor) and not torch.isfinite(out).all():
            raise RuntimeError(f"non-finite output in {name}")
    return hook

for name, m in model.named_modules():
    m.register_forward_hook(check(name))
```

只盯着最终的 loss 是 `nan` 没有意义，因为 `nan` 会传播，最终所有东西都是 `nan`。要找到它第一次出现在哪一层，才能对症下药：是 logits 太大、除了零，还是数据里本来就有坏值。

## 8.6 灾难性消减

> **📖 原文 [8.6 灾难性消减](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#86-灾难性消减)**
>
> 两个非常接近的大数相减时，高位有效数字抵消，剩余结果的相对误差可能巨大。例如用：
>
> $$
> \operatorname{Var}(X)
> = \mathbb{E}[X^2]-\mathbb{E}[X]^2
> $$
>
> 计算方差时，两项可能都很大且接近，相减后精度损失严重。

### Q32. 原文练习 7：Welford 和 $\mathbb{E}[x^2] - \mathbb{E}[x]^2$ 在“大均值、小方差”数据上差多少？

数据：均值 $10^4$、方差 1，float32：

```python
x = (1e4 + torch.randn(100000)).float()
(x * x).mean() - x.mean() ** 2     # 16.0      ← 公式法：错了 16 倍
((x - x.mean()) ** 2).mean()       # 1.00248   ← two-pass
x.double().var(unbiased=False)     # 1.00248   ← float64 参考
```

为什么错这么多？$\mathbb{E}[x^2] \approx 10^8$，float32 只有约 7 位有效数字，在 $10^8$ 附近的间距约为 8。两个约 $10^8$ 的数相减，真实差值才 1，比间距还小，结果完全由舍入误差决定。

Welford 的写法（逐个更新）：

```python
def welford(xs):
    n, mu, M2 = 0, 0.0, 0.0
    for v in xs:
        n += 1
        d = v - mu            # 和“当前均值”的差，数值很小，不会抵消
        mu += d / n
        M2 += d * (v - mu)
    return M2 / n             # 总体方差
```

它从不计算 $x^2$ 这样的大数，每一步都只处理“和均值的差”，所以没有灾难性消减。我用 float32 在 2 万个样本上测得 1.0097，float64 参考值 1.0092。相比公式法的 16，已经非常接近了。

### Q33. Welford “能合并不同分块的统计量”是怎么合并的？

两块数据分别有 $(n_a, \mu_a, M_{2,a})$ 和 $(n_b, \mu_b, M_{2,b})$，合并公式是：

$$
n = n_a + n_b,\quad \delta = \mu_b - \mu_a,\quad \mu = \mu_a + \delta\frac{n_b}{n},\quad M_2 = M_{2,a} + M_{2,b} + \delta^2\frac{n_a n_b}{n}
$$

这样每个线程块各自算一段，最后两两合并，就是一棵并行归约树。LayerNorm 的 CUDA Kernel 正是这么做的。这和 [原文 9.4](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#94-算例四online-softmax-的分块合并) 的 Online Softmax 合并是同一种思路：**找一个能分块计算、再合并的“状态”**。

## 8.7 条件数：输入误差会被放大多少

> **📖 原文 [8.7 条件数：输入误差会被放大多少](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#87-条件数输入误差会被放大多少)**
>
> 问题的条件数描述输入微小扰动对输出的影响。对可逆矩阵，在某个一致范数下：
>
> $$
> \kappa(\mathbf{A})
> = \lVert\mathbf{A}\rVert
>   \lVert\mathbf{A}^{-1}\rVert
> $$
>
> 条件数大表示问题病态：即使算法实现正确、输入只含很小舍入误差，输出也可能变化很大。

### Q34. 条件数大是什么意思？“问题病态”和“算法不稳定”有什么区别？

- **病态问题**：问题本身就很敏感，输入有一点点误差，正确答案就会变很多。这和你用什么算法无关。[原文 2.9](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#29-转置单位矩阵与逆矩阵) 的 Hilbert 矩阵条件数 $1.6 \times 10^{13}$，意味着输入的相对误差可能被放大约 $10^{13}$ 倍；
- **不稳定算法**：问题本身不敏感，但算法引入了额外误差。比如用 $\mathbb{E}[x^2] - \mathbb{E}[x]^2$ 算方差，问题本身很好算，是公式选错了。

对应的解决办法也不同：不稳定算法可以**换算法**（换成 Welford）；病态问题换算法也没用，只能**改问题**（比如加正则化）或用更高精度缓解。

## 8.8 混合精度训练的基本模式

> **📖 原文 [8.8 混合精度训练的基本模式](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#88-混合精度训练的基本模式)**
>
> 混合精度不是“把所有东西都换成 FP16”。常见模式是：
>
> 1. 大型矩阵乘使用 FP16/BF16 输入，Tensor Core 加速；
> 2. 乘加在 FP32 或更高的指定累加精度中完成；
> 3. 归约、归一化、Softmax 等敏感操作保留或内部提升到 FP32；
> 4. 优化器状态和/或 master weights 保留 FP32；
> 5. 根据硬件和算子选择输出 dtype。
>
> 每个框架和硬件的具体策略不同，应查实际算子文档和 profiler，而不是从输入 dtype 猜内部计算路径。

### Q35. 混合精度在 PyTorch 里怎么写？“哪些算子用低精度”是谁决定的？

用 `torch.autocast`，它会根据**算子白名单**自动选择精度：

```python
model = model.cuda()
optimizer = torch.optim.AdamW(model.parameters())   # 参数本身保持 FP32（master weights）

for x, y in loader:
    with torch.autocast(device_type="cuda", dtype=torch.bfloat16):
        logits = model(x)                  # matmul/conv 自动用 BF16
        loss = F.cross_entropy(logits, y)  # softmax/log/sum 等自动提升到 FP32
    loss.backward()
    optimizer.step()
    optimizer.zero_grad()
```

autocast 的大致规则：矩阵乘、卷积这种能用 Tensor Core 加速、且对精度不太敏感的算子用低精度；`exp`、`log`、`softmax`、`sum`、归一化这种容易溢出或需要累加的算子用 FP32。正好对应原文列的五条模式。

这也是为什么原文说“不要从输入 dtype 猜内部计算路径”：同样是 BF16 输入，`matmul` 内部用 FP32 累加，`softmax` 被 autocast 提升到 FP32 计算。

## 8.9 Loss Scaling 为什么有用

> **📖 原文 [8.9 Loss Scaling 为什么有用](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#89-loss-scaling-为什么有用)**
>
> 反向传播中的 FP16 小梯度可能下溢为 0。令缩放因子为 $S$：
>
> $$
> L' = S L
> $$
>
> 则：
>
> $$
> \nabla L'=S\nabla L
> $$

### Q36. Loss Scaling 能演示一下吗？为什么乘了再除回来就有用？

FP16 能表示的最小正数（含次正规数）约 $5.96 \times 10^{-8}$，更小的梯度直接变成 0：

```python
g = torch.tensor(1e-8)
g.half()                               # 0.0       ← 下溢，梯度丢了
(g * 65536).half()                     # 0.000655  ← 放大 65536 倍后能表示
((g * 65536).half().float() / 65536)   # 9.997e-09 ← 在 FP32 里除回去，几乎无损
```

关键在于**乘和除发生在不同的精度里**：放大发生在反向开始时（loss × S），整个反向在 FP16 里进行，小梯度因为被放大了所以不会下溢；除回去发生在优化器更新前，在 FP32 里做，所以不会再下溢。

PyTorch 里用 `GradScaler` 自动完成动态 scaling：

```python
scaler = torch.amp.GradScaler()
with torch.autocast("cuda", dtype=torch.float16):
    loss = model(x, y)
scaler.scale(loss).backward()     # loss × S 后反向
scaler.unscale_(optimizer)        # 梯度 ÷ S（要做梯度裁剪的话，必须先 unscale）
torch.nn.utils.clip_grad_norm_(model.parameters(), 1.0)
scaler.step(optimizer)            # 若梯度有 inf/nan 就跳过这一步
scaler.update()                   # 调整 S：溢出则减小，稳定一段时间则增大
```

### Q37. 为什么 BF16 通常不需要 Loss Scaling？

BF16 的指数位和 FP32 一样，最小正规数约 $1.18 \times 10^{-38}$，$10^{-8}$ 这样的梯度完全能表示，不存在下溢问题。所以用 BF16 时一般只用 autocast，不用 GradScaler，代码更简单。

但 BF16 精度更差，所以 [原文 8.3](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#83-舍入与机器精度) 的“更新被吞掉”问题更严重，master weights 更加必要。原文最后一句也提醒：不用 Loss Scaling 不代表不会出 `nan`，模型本身的问题（比如 logits 爆炸）照样会导致不稳定。

## 8.10 稳定算法通常也更适合融合

> **📖 原文 [8.10 稳定算法通常也更适合融合](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#810-稳定算法通常也更适合融合)**
>
> 数值稳定与性能优化并不总是冲突：
>
> - Stable Softmax 先做 max 归约，再做 exp/sum 归约；Online Softmax 可合并分块状态；
> - `log_softmax + NLL` 融合减少中间概率写回；
> - Welford 同时维护统计量，适合 LayerNorm 的块级归约；
> - GEMM 用低精度输入、高精度累加兼顾吞吐与误差。
>
> 关键是选择具有可组合状态的数学形式，让 Kernel 能在有限片上存储中分块处理。

### Q38. “稳定算法通常也更适合融合”，这和直觉相反吧？

直觉上，稳定算法要多做事（减最大值、维护额外状态），应该更慢。但原文的观察是：**稳定算法往往天然具有“可分块合并的状态”**，而这正是融合 Kernel 需要的：

| 算法 | 维护的状态 | 好处 |
| --- | --- | --- |
| Online Softmax | 最大值 $m$ + 指数和 $l$ | 一遍扫描、可分块，FlashAttention 的基础 |
| Welford | $n, \mu, M_2$ | 一遍求出均值和方差，可分块合并 |
| `log_softmax + NLL` 融合 | LSE | 不写回中间概率张量 |

反过来，不稳定的 $\mathbb{E}[x^2] - \mathbb{E}[x]^2$ 虽然也能一遍算完，但结果不能用。所以 Kernel 设计的思路通常是：先找一个数值稳定、且状态能合并的数学形式，再去写代码。

## 9.1 算例一：Transformer 线性层的 shape 与代价

> **📖 原文 [9.1 算例一：Transformer 线性层的 shape 与代价](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#91-算例一transformer-线性层的-shape-与代价)**
>
> 输入：
>
> $$
> \mathbf{X}\in\mathbb{R}^{B\times S\times H}
> $$
>
> 权重：
>
> $$
> \mathbf{W}\in\mathbb{R}^{H\times 4H}
> $$

### Q39. 算例一里“把前两维展平”是怎么回事？线性层不是只支持二维输入吗？

`nn.Linear` 只作用在**最后一维**，前面有几维都可以，内部的做法就是展平成二维再做 GEMM：

```python
x = torch.randn(8, 2048, 4096)             # (B, S, H)
x2 = x.reshape(-1, 4096)                   # (B*S, H) = (16384, 4096)，连续张量的 reshape 不复制
y2 = x2 @ W                                # (16384, 4096) @ (4096, 16384)
y = y2.reshape(8, 2048, 16384)             # 还原成 (B, S, 4H)
```

所以对线性层来说，“batch”和“sequence”没有区别，都是一堆独立的 token。$8 \times 2048 = 16384$ 个 token 共享同一个权重，这让 GEMM 的 $M$ 维很大，算术强度很高（[原文 3.3](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#33-一个简化的算术强度估算)）。

### Q40. $2.2 \times 10^{12}$ FLOPs 在实际 GPU 上要多久？

$8 \times 8 \times 2048 \times 4096^2 \approx 2.20 \times 10^{12}$。在 A100（BF16 峰值约 312 TFLOPS）上：

$$
\frac{2.2 \times 10^{12}}{312 \times 10^{12}} \approx 7 \text{ ms}
$$

实际能达到峰值的 60%～80%，大约 9～12 ms。这只是一个 FFN 的升维投影，一个 Transformer 层有好几个这样的 GEMM，一个模型有几十层，再乘以反向的 2 倍……这就是大模型训练按“GPU 天”甚至“GPU 年”算的原因。

## 9.2 算例二：多头 Attention 的完整维度

> **📖 原文 [9.2 算例二：多头 Attention 的完整维度](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#92-算例二多头-attention-的完整维度)**
>
> 从：
>
> $$
> \mathbf{X}\in\mathbb{R}^{B\times S\times H}
> $$
>
> 生成 Q/K/V 后 reshape 并转置：
>
> $$
> \mathbf{Q},\mathbf{K},\mathbf{V}
> \in\mathbb{R}^{B\times N_h\times S\times D_h}
> $$

### Q41. 多头 Attention 里 $(B, S, H) \to (B, N_h, S, D_h)$ 的 reshape 和转置，代码怎么写？

```python
B, S, H, Nh = 2, 16, 512, 8
Dh = H // Nh                                # 64
x = torch.randn(B, S, H)
q = nn.Linear(H, H)(x)                      # (B, S, H)
q = q.view(B, S, Nh, Dh)                    # (B, S, Nh, Dh)：把 H 切成 Nh 段
q = q.transpose(1, 2)                       # (B, Nh, S, Dh)：把 head 维提到前面当 batch 维
# ... 同样得到 k, v，算完 attention 后：
o = torch.randn(B, Nh, S, Dh)
o = o.transpose(1, 2).contiguous().view(B, S, H)   # 合并 head：转回来，必须 contiguous 才能 view
```

最后一行就是 [原文 2.2](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#22-shape索引和-stride) 讲的情况：`transpose` 之后不连续，直接 `view` 会报错，要先 `contiguous()`（或者用 `reshape`）。

### Q42. 原文练习 9：$B=2, N_h=32, S=8192$ 时 FP16 的 score 矩阵有多大？

$$
2 \times 32 \times 8192^2 \times 2 \text{ 字节} = 8 \text{ GiB}
$$

原文的例子 $B=1, S=32768$ 是 64 GiB。规律：**序列长度翻倍，score 矩阵变成 4 倍**，这就是 Attention 的 $O(S^2)$ 显存。而且这只是一层、只是 score，还没算 Softmax 后的概率和反向需要保存的中间量。不用 FlashAttention 的话，长序列训练根本跑不起来。

## 9.3 算例三：为什么除以 $\sqrt{D_h}$

> **📖 原文 [9.3 算例三：为什么除以 $\sqrt{D_h}$](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#93-算例三为什么除以-sqrtd_h)**
>
> 假设 $q_i,k_i$ 独立、均值 0、方差 1：
>
> $$
> \mathbf{q}^\top\mathbf{k}
> =\sum_{i=1}^{D_h}q_i k_i
> $$
>
> 每项 $q_i k_i$ 期望约为 0、方差约为 1，则和的方差约为 $D_h$，标准差约为 $\sqrt{D_h}$。

### Q43. 除以 $\sqrt{D_h}$ 的效果能做实验看吗？

```python
for dh in [16, 64, 128]:
    q, k = torch.randn(100000, dh), torch.randn(100000, dh)
    s = (q * k).sum(1)
    print(dh, s.var(), (s / math.sqrt(dh)).var())
# 16    15.9    0.992
# 64    63.8    0.997
# 128  128.3    1.002
```

不缩放时方差正好约等于 $D_h$，缩放后都约等于 1，和原文推导一致。

再看对 Softmax 的影响：对 64 个 key 做 Softmax，看最大概率的平均值：

```text
D_h = 64：  不缩放 0.777，缩放后 0.118
```

不缩放时，平均最大概率接近 0.8，几乎是 one-hot，注意力只盯着一个位置。缩放后分布平缓得多。这就是原文说的“Softmax 进入极端饱和区”：分布接近 one-hot 时，Softmax 的梯度（[原文 6.7](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#67-softmax-的-jacobian)）接近 0，模型很难学。

### Q44. 这个 $\sqrt{D_h}$ 和温度是什么关系？

除以 $\sqrt{D_h}$ 就是一个**固定的温度** $T = \sqrt{D_h}$（[原文 5.4](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#54-温度参数)）。$D_h = 128$ 时 $T \approx 11.3$。它不是调出来的超参数，而是根据方差推导出来、专门抵消点积尺度随维度增长的那一部分。

## 9.4 算例四：Online Softmax 的分块合并

> **📖 原文 [9.4 算例四：Online Softmax 的分块合并](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#94-算例四online-softmax-的分块合并)**
>
> 一行 logits 被切成多个 block。已处理部分的最大值和指数和为 $(m,l)$：
>
> $$
> m=\max_i x_i,\qquad
> l=\sum_i e^{x_i-m}
> $$
>
> 新 block 的状态为 $(m_b,l_b)$。合并最大值：
>
> $$
> m_{new}=\max(m,m_b)
> $$

### Q45. Online Softmax 的合并公式能用代码验证吗？

把 10 个数按每块 3 个切开，一块一块更新 $(m, l)$：

```python
z = torch.randn(10) * 5
m, l = -float("inf"), 0.0
for blk in z.split(3):
    mb = blk.max().item()
    lb = torch.exp(blk - mb).sum().item()
    m_new = max(m, mb)
    l = math.exp(m - m_new) * l + math.exp(mb - m_new) * lb   # 把旧的和新的都换到同一个基准
    m = m_new
l, torch.exp(z - z.max()).sum().item()   # (1.41705149, 1.41705143)  一致
```

关键是 $e^{m - m_{new}}$ 这一项：之前的指数和是以旧最大值 $m$ 为基准算的，现在最大值变成了 $m_{new}$，需要乘上 $e^{m - m_{new}}$ 把它“换算”到新基准。如果最大值没变，这个系数就是 1。

### Q46. 为什么这个技巧对 FlashAttention 这么关键？

普通 Attention 要先算完整的 $S \times S$ 分数矩阵，才能对每一行做 Softmax（因为要知道整行的最大值）。Online Softmax 让 Softmax 可以**一块一块**地算：每读进一块 K，就更新一次 $(m, l)$ 和输出累加器，不需要等整行算完。

于是整个 $S \times S$ 矩阵永远不需要完整存在显存里，只在片上存一小块。这就把 [原文 9.2](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#92-算例二多头-attention-的完整维度) 那 64 GiB 的显存需求消除了。模块二的 FlashAttention 一节会逐行讲这个算法。

## 9.5 算例五：LoRA 的参数与计算

> **📖 原文 [9.5 算例五：LoRA 的参数与计算](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#95-算例五lora-的参数与计算)**
>
> 完整线性层：
>
> $$
> \mathbf{y}=\mathbf{x}\mathbf{W}_0^\top
> $$
>
> LoRA：
>
> $$
> \mathbf{y}
> =\mathbf{x}\mathbf{W}_0^\top
> +\frac{\alpha}{r}
>   (\mathbf{x}\mathbf{A}^\top)\mathbf{B}^\top
> $$

### Q47. 原文练习 8：$4096 \times 4096$ 权重、rank 8/16/64 的 LoRA，参数比例和额外 FLOPs 是多少？

每个 token 的前向 FLOPs：原始 GEMM 是 $2 \times 4096 \times 4096 \approx 3.36 \times 10^7$；LoRA 多出两个小 GEMM，$2 \times 4096 \times r + 2 \times r \times 4096 = 16384r$。

| $r$ | 新增参数 | 参数比例 | 额外 FLOPs/token | 额外计算比例 |
| --- | --- | --- | --- | --- |
| 8 | 65,536 | 0.39% | 131,072 | 0.39% |
| 16 | 131,072 | 0.78% | 262,144 | 0.78% |
| 64 | 524,288 | 3.13% | 1,048,576 | 3.13% |

额外计算比例和参数比例一样，因为每个参数每个 token 都做一次乘加。看起来很小，但小 GEMM 的算术强度低、Kernel 启动有固定开销，实测延迟增加往往比这个比例更明显。这也是推理时倾向于合并权重的原因。

## 9.6 算例六：分布式均值为什么要加权

> **📖 原文 [9.6 算例六：分布式均值为什么要加权](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#96-算例六分布式均值为什么要加权)**
>
> 两个 rank 的局部样本数分别为 $n_1,n_2$，局部均值为 $\mu_1,\mu_2$。全局均值不是简单的 $(\mu_1+\mu_2)/2$，除非 $n_1=n_2$。正确值：
>
> $$
> \mu=
> \frac{n_1\mu_1+n_2\mu_2}{n_1+n_2}
> $$
>
> 更一般地，每个 rank 先汇总局部 `sum` 和 `count`，AllReduce 后再相除。分布式 loss、指标和吞吐统计都要明确分母，尤其是最后一个不完整 batch 或序列 mask 不同的场景。

### Q48. 原文练习 10：两个 rank 分别有 7 个和 5 个有效 token，局部 loss 均值 2.0 和 3.0，全局均值是多少？

$$
\mu = \frac{7 \times 2.0 + 5 \times 3.0}{7 + 5} = \frac{29}{12} \approx 2.417
$$

简单平均得到 $(2.0 + 3.0)/2 = 2.5$，错误在于把 5 个 token 的 rank 和 7 个 token 的 rank 当成了同样重要。正确做法：每个 rank 汇报 `(sum, count)`，即 `(14.0, 7)` 和 `(15.0, 5)`，AllReduce 后得到 `(29.0, 12)`，再相除。

### Q49. 这个问题在实际训练中会在哪里出现？

最常见的是**语言模型的变长序列**：

- 每个 rank 上的 batch 里 padding 数量不同，有效 token 数不同；
- `F.cross_entropy(..., ignore_index=-100)` 默认在**本 rank 内**对有效 token 求平均，然后 DDP 再对各 rank 的梯度求平均，于是变成了“平均的平均”，和全局 token 平均不一致；
- 梯度累积时同理，每个 micro-batch 的有效 token 数不同。

严格的做法是用 `reduction="sum"` 算总 loss，再除以全局有效 token 数。很多框架后来都专门修复过这个问题。虽然差别通常不大，但在对比实验、复现论文数字时可能就是那一点差距的来源。

## 9.7 算例七：显存估算

> **📖 原文 [9.7 算例七：显存估算](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#97-算例七显存估算)**
>
> 若一个张量 shape 为 $(B,S,H)$，元素字节数为 $b$，其连续存储大小：
>
> $$
> M=B\times S\times H\times b
> $$
>
> 例如 BF16 的 $(8,4096,8192)$：
>
> $$
> 8\times4096\times8192\times2
> =536{,}870{,}912\ \text{bytes}
> =512\ \text{MiB}
> $$

### Q50. 512 MiB 是怎么算的？MiB 和 MB 有什么区别？

$8 \times 4096 \times 8192 \times 2 = 536{,}870{,}912$ 字节。

- MiB（mebibyte）= $2^{20}$ = 1,048,576 字节，所以是正好 512 MiB；
- MB（megabyte）= $10^6$ 字节，所以约 537 MB。

`nvidia-smi` 显示的 `MiB` 是二进制单位，硬盘厂商标的 GB 通常是十进制，所以“80GB”显卡在系统里显示约 81920 MiB。估算显存时用哪个都行，但比较时要统一。

### Q51. 除了参数和激活，训练显存还有哪些“看不见”的部分？

原文列的后几项在实际中经常占好几 GB：

- **CUDA context**：一个进程初始化 CUDA 后就固定占用几百 MB；
- **cuBLAS/cuDNN workspace**：矩阵乘、卷积需要的临时空间；
- **PyTorch 缓存分配器的碎片**：释放的显存不会马上还给系统，而是缓存起来复用，`nvidia-smi` 看到的占用会比实际张量大；
- **通信 buffer**：DDP 的梯度分桶、NCCL 的缓冲区。

看实际情况可以用：

```python
torch.cuda.memory_allocated() / 2**30    # 张量实际占用 (GiB)
torch.cuda.memory_reserved() / 2**30     # 缓存分配器向系统申请的总量 (GiB)
torch.cuda.max_memory_allocated() / 2**30   # 峰值
```

`reserved` 和 `allocated` 的差就是缓存和碎片。OOM 时先打印这几个数，比盲目减 batch 更有效。

## 9.8 从数学式到 Kernel 的固定检查表

> **📖 原文 [9.8 从数学式到 Kernel 的固定检查表](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#98-从数学式到-kernel-的固定检查表)**
>
> 看到一个新算子时，按以下顺序拆解：
>
> 1. 输入、输出和中间量的 shape 是什么？
> 2. 哪些维度保留，哪些维度归约？
> 3. 能否分块？分块状态如何合并？
> 4. FLOPs 和理论最小访存量是多少？
> 5. 是否存在广播、转置或非连续 stride？
> 6. 哪些操作对精度敏感，需要高精度累加？
> 7. 是否会出现 exp 溢出、除零、消减或长归约误差？
> 8. 中间张量能否融合消除？
> 9. 边界 shape 和尾块如何处理？
> 10. 用什么参考实现、dtype 容差和极端输入验证？
>
> 这份检查表把抽象数学转换成可执行的 Infra 工作流。

### Q52. 这份十条检查表怎么用？能拿 Softmax 走一遍吗？

以对 `(B, S, V)` 的 logits 做 Softmax 为例：

1. **shape**：输入输出都是 `(B, S, V)`，中间有每行的最大值和指数和 `(B, S)`；
2. **保留 / 归约**：保留 B、S，沿 V 归约（求 max、求 sum）；
3. **能否分块**：能，Online Softmax 的 $(m, l)$ 可以合并（[原文 9.4](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#94-算例四online-softmax-的分块合并)）；
4. **FLOPs / 访存**：每个元素几次运算，算术强度极低，memory-bound，优化重点是少读几遍；
5. **广播 / stride**：除以分母时 `(B, S, 1)` 广播到 `(B, S, V)`；V 维要连续，否则读取很慢；
6. **精度敏感**：exp 和 sum 要在 FP32 里做；
7. **数值风险**：exp 溢出（减最大值）、全 mask 行的 0/0（特殊处理）；
8. **融合**：如果后面接交叉熵，融合成 `log_softmax + NLL`，不写回概率张量；
9. **尾块**：V 不是块大小的整数倍时要掩码（词表 32000、50257 都可能不对齐）；
10. **验证**：和 float64 的参考实现比较；输入要包括大 logits（1000）、全相等、全 $-\infty$ 这些极端情况。

这十条正好把这一章的每一节都用上了一遍。后面读 CUDA 模块的每个算子时，都可以先按这个表过一遍。

## 第2章系列目录

- [导读与易错点速查](/posts/ai-infra/ch2-00-guide/)
- [（一）符号、张量与线性代数](/posts/ai-infra/ch2-01-tensor-linear-algebra/)
- [（二）GEMM 分块与概率论](/posts/ai-infra/ch2-02-gemm-probability/)
- [（三）Softmax、交叉熵与反向传播](/posts/ai-infra/ch2-03-softmax-backprop/)
- **（四）优化、归一化、数值精度与综合算例**（本篇）

其他章节：

- [第1章 编程语言基础：导读与易错点速查](/posts/ai-infra/ch1-00-guide/)

