---
title: "AI Infra 第2章笔记（三）：Softmax、交叉熵与反向传播"
published: 2026-11-03T19:00:00
description: "这一篇的主线：logits 经过 Softmax 变成分布 → 减最大值保证不溢出，LogSumExp 保证取 log 不出事 → 交叉熵 = 负对数似然 = 熵 + KL → 链式法则沿计算图把梯度从 loss 传…"
tags: ["AI Infra", "数学", "Softmax", "反向传播"]
category: "AI Infra 学习笔记"
draft: false
---

这一篇的主线：**logits 经过 Softmax 变成分布 → 减最大值保证不溢出，LogSumExp 保证取 log 不出事 → 交叉熵 = 负对数似然 = 熵 + KL → 链式法则沿计算图把梯度从 loss 传回每个参数 → 线性层的反向是两次 GEMM，Softmax + 交叉熵的梯度简化成 `p - y`。** 这一篇几乎每个结论都能用 PyTorch 的 autograd 直接验证，下面的代码都跑过。

> [!NOTE]
> 本篇对应原文 [AIInfraGuide · 第2章：数学基础](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/) 的 §5–6（作者 [caomaolufei](https://github.com/caomaolufei/AIInfraGuide)，MIT 许可）。每节开头的引用块摘自原文，文中的“原文 x.y”可以点击跳转到原文对应小节。问答是按第1章的提问思路整理的，代码输出均为实际运行结果。

## 5.1 Logits 不是概率

> **📖 原文 [5.1 Logits 不是概率](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#51-logits-不是概率)**
>
> 模型最后一层输出 $V$ 个任意实数：
>
> $$
> \mathbf{z}=(z_1,z_2,\ldots,z_V)
> $$
>
> 这些值叫 logits。它们可以为负，也不要求和为 1。Softmax 把它们转换为分类分布：
>
> $$
> p_i = \operatorname{softmax}(\mathbf{z})_i
> = \frac{e^{z_i}}{\sum_{j=1}^{V}e^{z_j}}
> $$

### Q1. logits 这个名字是什么意思？为什么不直接让模型输出概率？

名字来自 logistic 回归里的 log-odds（对数几率），现在泛指“Softmax 之前的那组任意实数”。

不让模型直接输出概率，是因为概率有两个约束：每个值在 $[0,1]$ 之间、加起来等于 1。网络最后一层是线性层，输出可以是任何实数，很难直接满足这两个约束。做法是让网络自由输出 logits，再用 Softmax 这个固定的函数“翻译”成概率：

- $e^{z_i}$ 保证每一项都大于 0；
- 除以总和保证加起来是 1。

### Q2. “Softmax 对统一平移不敏感”是什么意思？为什么这很重要？

所有 logits 同时加上同一个数 $c$，Softmax 结果不变：

```python
z = torch.tensor([1000., 1001., 1002.])
torch.softmax(z, 0)       # tensor([0.0900, 0.2447, 0.6652])
torch.softmax(z - 5, 0)   # tensor([0.0900, 0.2447, 0.6652])  完全一样
```

原因：$\frac{e^{z_i + c}}{\sum_j e^{z_j + c}} = \frac{e^c \cdot e^{z_i}}{e^c \cdot \sum_j e^{z_j}}$，分子分母的 $e^c$ 约掉了。

这说明 **Softmax 只关心 logits 之间的差，不关心绝对大小**。这个性质有两个用处：一是下一节的“减最大值”稳定技巧；二是解释为什么 logits 整体漂移不影响预测，模型只需要学“正确类比错误类高多少”。

## 5.2 数值稳定的 Softmax

> **📖 原文 [5.2 数值稳定的 Softmax](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#52-数值稳定的-softmax)**
>
> 若直接计算 $e^{1000}$，有限精度浮点数会溢出。令：
>
> $$
> m=\max_j z_j
> $$
>
> 稳定形式为：
>
> $$
> p_i=
> \frac{e^{z_i-m}}
> {\sum_j e^{z_j-m}}
> $$

### Q3. 直接计算 $e^{1000}$ 会发生什么？

float32 能表示的最大数约 $3.4 \times 10^{38}$，而 $e^{89} \approx 4.5 \times 10^{38}$ 就已经超了：

```python
z = torch.tensor([1000., 1001., 1002.])
torch.exp(z)                       # tensor([inf, inf, inf])
torch.exp(z) / torch.exp(z).sum()  # tensor([nan, nan, nan])   ← inf / inf = nan
torch.softmax(z, 0)                # tensor([0.0900, 0.2447, 0.6652])  PyTorch 内部减了最大值
```

FP16 更严重，最大值只有 65504，$e^{11.1}$ 就溢出了。所以任何手写的 Softmax（包括后面 CUDA 模块里写的 Kernel）**第一步都必须减最大值**。

### Q4. 减最大值能防溢出，那很小的数下溢成 0 怎么办？

原文的意思是：减完最大值后，最大项变成 $e^0 = 1$，其他项都 $\le 1$。如果某一项小到下溢成 0，说明它比最大项小了几十个数量级，**在概率上本来就可以忽略**，变成 0 没关系。

真正的问题出现在**后面还要取 log** 的时候：

```python
zz = torch.tensor([0., -200.])
torch.log(torch.softmax(zz, 0))   # tensor([0., -inf])   ← 先变成 0，log(0) = -inf
torch.log_softmax(zz, 0)          # tensor([0., -200.])  ← 正确值是 -200
```

真实的对数概率是 -200，是个完全正常的数，但“先 softmax 再 log”会得到 $-\infty$，再参与 loss 计算就变成 `nan`。所以原文说要“直接用稳定的 `log_softmax`”，下一节的 LogSumExp 就是它的实现原理。

### Q5. `stable_softmax` 伪代码里为什么要遍历三次？能不能少遍历几次？

原文的写法需要三次遍历：

1. 找最大值 `max(x)`；
2. 算每个 `exp(value - maximum)` 并求和；
3. 每个值除以分母。

每次遍历都要从显存读一遍数据。对于很长的向量（比如词表 V = 128000），这三次读取就是 Softmax 的主要开销。

能少吗？可以。第 2 步和第 3 步可以合并到写回时做，而第 1、2 步可以用 **Online Softmax** 合并成一次遍历：边扫描边更新“当前最大值”和“当前指数和”，最大值变了就把之前的和重新缩放一下。这在 [原文 9.4](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#94-算例四online-softmax-的分块合并) 有推导，模块二的 Online Softmax 一节会写成 CUDA Kernel，FlashAttention 的核心也是这个技巧。

## 5.3 LogSumExp

> **📖 原文 [5.3 LogSumExp](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#53-logsumexp)**
>
> 定义：
>
> $$
> \operatorname{LSE}(\mathbf{z})
> = \log\sum_j e^{z_j}
> $$
>
> 稳定形式：
>
> $$
> \operatorname{LSE}(\mathbf{z})
> = m + \log\sum_j e^{z_j-m}
> $$

### Q6. LogSumExp 是什么？为什么叫“平滑的最大值”？

$\operatorname{LSE}(\mathbf{z}) = \log\sum_j e^{z_j}$。它的值总是比最大值大一点点，但不超过最大值加 $\log V$：

$$
\max_j z_j \le \operatorname{LSE}(\mathbf{z}) \le \max_j z_j + \log V
$$

如果最大值比其他值大很多，LSE 几乎就等于最大值；如果所有值差不多，LSE 比最大值大 $\log V$。所以它像一个“软化版”的 max，而且处处可导（真正的 max 在两个值相等时不可导）。

稳定形式 $m + \log\sum_j e^{z_j - m}$ 就是把最大值提出来，括号里的部分最大项是 1，不会溢出，最小也是 $\log 1 = 0$，不会出现 $\log 0$。

### Q7. `F.cross_entropy` 要求传入 logits 还是概率？传错会怎样？

**传 logits**。`F.cross_entropy` 内部已经做了 `log_softmax + nll_loss`，如果你先手动 softmax 再传进去，就做了两次 Softmax：

```python
logits = torch.tensor([[2.0, 1.0, 0.1]])
y = torch.tensor([0])
F.cross_entropy(logits, y)                     # 0.4170  正确
-torch.log_softmax(logits, 1)[0, 0]            # 0.4170  手算验证
F.cross_entropy(torch.softmax(logits, 1), y)   # 0.8021  ← 错：做了两次 softmax
F.nll_loss(torch.softmax(logits, 1), y)        # -0.6590 ← 错：nll_loss 要的是 log 概率
```

最危险的是第三种：**代码能跑，loss 也在下降，只是一直下降得很慢、效果不好**。两次 Softmax 会把分布压得很平，梯度变小。如果你看到某个开源代码在模型最后加了 `nn.Softmax`，又用 `CrossEntropyLoss`，基本就是这个 bug。

记法：`cross_entropy` 吃 logits，`nll_loss` 吃 `log_softmax` 的输出，两者都不吃概率。

## 5.4 温度参数

> **📖 原文 [5.4 温度参数](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#54-温度参数)**
>
> 带温度 $T>0$ 的 Softmax：
>
> $$
> p_i(T)=
> \frac{e^{z_i/T}}{\sum_j e^{z_j/T}}
> $$
>
> - $T<1$：放大 logit 差异，分布更尖锐；
> - $T>1$：缩小差异，分布更平坦；
> - $T\to0^+$：逐渐接近只选择最大 logit；
> - $T\to\infty$：逐渐接近均匀分布。
>
> 温度改变的是采样分布，不等于修改模型权重。工程实现应先缩放 logits，再应用稳定 Softmax，并谨慎处理非常小的温度。

### Q8. 温度 $T$ 具体对分布有什么影响？能看一下数字吗？

同一组 logits `[2.0, 1.0, 0.1]`，不同温度下：

```python
z = torch.tensor([2., 1., 0.1])
torch.softmax(z / 0.5, 0)   # tensor([0.8638, 0.1169, 0.0193])  T<1 更尖
torch.softmax(z / 1,   0)   # tensor([0.6590, 0.2424, 0.0986])  原始分布
torch.softmax(z / 2,   0)   # tensor([0.5017, 0.3043, 0.1940])  T>1 更平
torch.softmax(z / 100, 0)   # tensor([0.3366, 0.3332, 0.3302])  接近均匀
```

直觉：除以 $T$ 等于把 logits 之间的**差距**缩放了 $1/T$ 倍。Softmax 只关心差距（[原文 5.1](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#51-logits-不是概率)），差距被放大分布就尖，差距被缩小分布就平。

### Q9. 温度在哪些地方会用到？

| 场景 | 温度的作用 |
| --- | --- |
| LLM 生成 | $T<1$ 输出更确定、更保守；$T>1$ 更有创造性但更容易胡说 |
| 知识蒸馏 | 老师模型用 $T>1$ 软化分布，让学生学到“错误类别之间的相对关系” |
| 对比学习 / 小样本 | 余弦相似度只在 $[-1,1]$，除以 0.07 或乘以 10 之类的温度把差距拉开 |

“谨慎处理非常小的温度”指的是：$T \to 0$ 时 $z/T$ 会变得极大，直接 `exp` 必然溢出。所以要先除以 $T$ 再走稳定 Softmax；$T = 0$ 时通常直接改成 greedy（取 argmax），不去真的除以 0。

## 5.5 交叉熵

> **📖 原文 [5.5 交叉熵](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#55-交叉熵)**
>
> 真实分布 $\mathbf{q}$ 与模型分布 $\mathbf{p}$ 的交叉熵：
>
> $$
> H(\mathbf{q},\mathbf{p})
> = -\sum_i q_i\log p_i
> $$
>
> 若真实标签是 one-hot，正确类别为 $y$：
>
> $$
> q_y=1,\quad q_{i\ne y}=0
> $$

### Q10. 交叉熵公式里 $\mathbf{q}$ 和 $\mathbf{p}$ 哪个是标签、哪个是模型输出？顺序能换吗？

$H(\mathbf{q}, \mathbf{p}) = -\sum_i q_i \log p_i$，**$\mathbf{q}$ 是真实分布（标签），$\mathbf{p}$ 是模型预测**。log 只作用在模型预测上。

顺序不能换。如果写成 $-\sum_i p_i \log q_i$，而 $\mathbf{q}$ 是 one-hot（除了正确类，其他都是 0），就会出现 $\log 0 = -\infty$。

### Q11. one-hot 标签时交叉熵为什么只剩 $-\log p_y$？如果标签不是 one-hot 呢？

one-hot 只有正确类 $q_y = 1$，其他 $q_i = 0$，求和里只剩一项：$-1 \cdot \log p_y$。

标签不是 one-hot 的情况也很常见：

- **Label smoothing**：把 one-hot 的 1 改成 0.9，剩下 0.1 平均分给其他类，防止模型过度自信。PyTorch 里是 `F.cross_entropy(logits, y, label_smoothing=0.1)`；
- **知识蒸馏**：用老师模型的输出分布作为 $\mathbf{q}$；
- **Mixup / CutMix**：两张图混合，标签也按比例混合，比如 `0.7 × 猫 + 0.3 × 狗`。

这些情况下就要用完整的 $-\sum_i q_i \log p_i$，`F.cross_entropy` 也支持直接传入概率形式的 target。

### Q12. 交叉熵 loss 的数值有什么参考意义？刚开始训练时应该是多少？

一个很实用的检查：**模型刚初始化时，输出接近均匀分布**，每个类概率约 $1/V$，所以初始 loss 应该约等于 $\log V$：

- 10 分类：$\log 10 \approx 2.30$；
- 1000 分类（ImageNet）：$\log 1000 \approx 6.91$；
- LLaMA 词表 32000：$\log 32000 \approx 10.37$。

如果你的初始 loss 明显偏离这个值（比如 1000 分类一开始就是 50），说明初始化或数据处理有问题，比如标签错位、logits 尺度过大。这是调试训练代码时最先该看的数字之一。

## 5.6 信息熵

> **📖 原文 [5.6 信息熵](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#56-信息熵)**
>
> 分布自身的熵：
>
> $$
> H(\mathbf{p})
> = -\sum_i p_i\log p_i
> $$
>
> 熵衡量不确定性：
>
> - 全部概率集中在一个类别时，熵最小；
> - $V$ 个类别均匀分布时，熵最大，为 $\log V$。

### Q13. 熵的“均匀分布时最大，为 $\log V$”能验证一下吗？

```python
V = 8
p = torch.full((V,), 1 / V)
-(p * p.log()).sum()    # 2.0794
math.log(V)             # 2.0794
```

直觉：熵衡量“平均要问几个是/否问题才能猜中答案”。8 个等可能的结果，用二分法要问 3 次，$\log_2 8 = 3$ bit；用自然对数就是 $\ln 8 \approx 2.08$ nat。分布越集中越好猜，熵越小；全部集中在一个类别时熵为 0，不用猜。

### Q14. nat 和 bit 有什么区别？用哪个都行吗？

只是对数的底不同，相差一个常数倍：$1 \text{ nat} = 1/\ln 2 \approx 1.443 \text{ bit}$。`torch.log` 是自然对数，所以 PyTorch 里的 loss 默认单位都是 nat。

比较不同论文的 loss 或 bits-per-byte 时要注意单位：有些论文报的是 bits，需要乘或除以 $\ln 2$ 换算。

## 5.7 KL 散度

> **📖 原文 [5.7 KL 散度](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#57-kl-散度)**
>
> 从分布 $\mathbf{q}$ 到 $\mathbf{p}$ 的 KL 散度：
>
> $$
> D_{KL}(\mathbf{q}\lVert\mathbf{p})
> = \sum_i q_i\log\frac{q_i}{p_i}
> $$
>
> 它满足：
>
> $$
> D_{KL}(\mathbf{q}\lVert\mathbf{p})\ge0
> $$

### Q15. KL 散度为什么不对称？能用数字感受一下吗？

```python
q = torch.tensor([0.7, 0.2, 0.1])
p = torch.tensor([0.4, 0.4, 0.2])
(q * (q / p).log()).sum()   # KL(q||p) = 0.1838
(p * (p / q).log()).sum()   # KL(p||q) = 0.1920
```

两个方向的值不同。直觉上，$D_{KL}(\mathbf{q} \| \mathbf{p})$ 是“**按 $\mathbf{q}$ 的视角**”去衡量 $\mathbf{p}$ 有多不像：$\mathbf{q}$ 认为重要的地方（$q_i$ 大），如果 $\mathbf{p}$ 给的概率很小，惩罚就很大；$\mathbf{q}$ 认为不重要的地方，$\mathbf{p}$ 怎么样都无所谓。换个视角，权重就不同了。

两个方向在实践中有不同的倾向：

- 最小化 $D_{KL}(\mathbf{q} \| \mathbf{p})$（前向 KL，交叉熵训练就是这个）：$\mathbf{p}$ 会尽量覆盖 $\mathbf{q}$ 的所有可能，宁可分散；
- 最小化 $D_{KL}(\mathbf{p} \| \mathbf{q})$（反向 KL）：$\mathbf{p}$ 倾向于只抓住 $\mathbf{q}$ 的一个主要峰值。

### Q16. “交叉熵 = 熵 + KL”能验证一下吗？为什么说最小化交叉熵等价于最小化 KL？

```python
H_q  = -(q * q.log()).sum()   # 真实分布自己的熵
H_qp = -(q * p.log()).sum()   # 交叉熵：0.9856
H_q + (q * (q / p).log()).sum()   # 熵 + KL：0.9856  完全一致
```

训练时 $\mathbf{q}$ 是数据标签，和模型参数无关，所以 $H(\mathbf{q})$ 是常数。对参数求梯度时常数消失，最小化交叉熵和最小化 KL 的梯度完全相同。

推论：**交叉熵 loss 的下限不是 0，而是 $H(\mathbf{q})$**。用 label smoothing 时 $\mathbf{q}$ 不是 one-hot，$H(\mathbf{q}) > 0$，所以 loss 永远降不到 0，这很正常。

### Q17. PyTorch 的 `F.kl_div` 参数顺序很奇怪，怎么用才对？

这是一个出名的坑。`F.kl_div(input, target)` 计算的是 $D_{KL}(\text{target} \| \text{input})$，而且**`input` 要传 log 概率，`target` 默认传概率**：

```python
F.kl_div(p.log(), q, reduction="sum")   # 0.1838 = KL(q||p)
```

三个注意点：

1. 第一个参数是**模型输出的 log 概率**（`log_softmax` 的结果），不是 logits，也不是概率；
2. 计算的方向是 KL(第二个参数 ‖ 第一个参数)，和参数顺序是反的；
3. 默认 `reduction="mean"` 会除以**元素总数**而不是 batch 大小，数学上不对。官方建议用 `reduction="batchmean"`。

知识蒸馏的标准写法：

```python
loss_kd = F.kl_div(
    F.log_softmax(student_logits / T, dim=-1),
    F.softmax(teacher_logits / T, dim=-1),
    reduction="batchmean",
) * T * T   # 乘 T² 是为了让梯度尺度不随温度变化
```

## 5.8 Perplexity

> **📖 原文 [5.8 Perplexity](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#58-perplexity)**
>
> 若平均 token 负对数似然为 $\bar{L}$，困惑度：
>
> $$
> \operatorname{PPL}=e^{\bar{L}}
> $$
>
> 它可直觉理解为模型在每一步面对的“有效候选数”。但不同 tokenizer、数据预处理、上下文长度和是否忽略特殊 token 都会影响 PPL，不能脱离评测设置直接横比。

### Q18. 困惑度（PPL）怎么理解成“有效候选数”？

如果模型每一步都在 $k$ 个候选里均匀地猜，每个 token 的 loss 就是 $\log k$，PPL 就是 $e^{\log k} = k$。所以 PPL = 10 可以理解为“模型平均在 10 个词之间犹豫”。

几个参考值：

- 平均 loss 为 2.0：PPL $= e^2 \approx 7.39$；
- 完全随机猜 GPT-2 的 50257 个 token：PPL = 50257；
- 训练好的大模型在常见文本上：个位数到十几。

### Q19. 为什么不同模型的 PPL 不能直接比？

因为 PPL 是“**每个 token**”的平均。tokenizer 不同，同一句话被切成的 token 数就不同：词表大的 tokenizer 切得粗、token 少，每个 token 包含的信息多，PPL 天然偏高。所以跨 tokenizer 比较时，更公平的指标是 bits-per-byte 或 bits-per-character（按原始文本长度归一化）。

## 5.9 Top-k 与 Top-p 采样

> **📖 原文 [5.9 Top-k 与 Top-p 采样](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#59-top-k-与-top-p-采样)**
>
> 模型输出分布后，解码策略决定如何选 token：
>
> - Greedy：选择最大概率 token；
> - Top-k：只在概率最高的 $k$ 个 token 中采样；
> - Top-p（nucleus）：选择累计概率至少达到 $p$ 的最小候选集合，再归一化采样。
>
> Top-p 集合大小会随分布尖锐程度动态变化。实现时通常先过滤 logits，把被排除项设为负无穷，再执行稳定 Softmax 和采样。

### Q20. Top-p 采样的“累计概率至少达到 $p$ 的最小集合”，能用代码走一遍吗？

设排好序的概率是 `[0.5, 0.2, 0.15, 0.1, 0.05]`，$p = 0.8$：

```python
probs = torch.tensor([0.5, 0.2, 0.15, 0.1, 0.05])
sorted_p, idx = probs.sort(descending=True)
cum = sorted_p.cumsum(0)          # tensor([0.50, 0.70, 0.85, 0.95, 1.00])
keep = (cum - sorted_p) < 0.8     # tensor([True, True, True, False, False])
idx[keep]                         # tensor([0, 1, 2])
```

`cum - sorted_p` 是“加入当前 token **之前**的累计概率”。只要加入之前还没到 0.8，就把当前 token 留下。所以保留了前 3 个（0.5 + 0.2 + 0.15 = 0.85 ≥ 0.8）。之后把保留的三个概率重新归一化（除以 0.85），再从中采样。

如果直接写 `cum < 0.8`，只会留下前 2 个（0.70），累计概率不够 0.8，这是一个常见的差一错误。

### Q21. Top-k 和 Top-p 哪个更好？为什么原文说 Top-p“动态变化”？

Top-k 的问题是 $k$ 固定：

- 模型很确定时（比如“中华人民共和”下一个字），真正合理的只有 1 个，但 $k=50$ 仍会把 49 个不合理的选项放进来；
- 模型很不确定时（比如故事开头），合理的选项可能有几百个，$k=50$ 又太少。

Top-p 按累计概率截断，分布尖的时候自动只留几个，分布平的时候自动多留。实践中常两者一起用，比如 `top_k=50, top_p=0.9`，再配合温度。

“先把被排除项设为负无穷”是因为 $e^{-\infty} = 0$，Softmax 后这些位置的概率正好是 0，其他位置自动重新归一化，一步完成过滤和归一化。

## 6.1 导数：局部变化率

> **📖 原文 [6.1 导数：局部变化率](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#61-导数局部变化率)**
>
> 单变量函数的导数：
>
> $$
> f'(x)=
> \lim_{h\to0}\frac{f(x+h)-f(x)}{h}
> $$
>
> 导数表示在 $x$ 附近，输入微小变化如何影响输出：
>
> $$
> f(x+\Delta x)
> \approx f(x)+f'(x)\Delta x
> $$

### Q22. 导数的近似式 $f(x + \Delta x) \approx f(x) + f'(x)\Delta x$ 有什么用？

它是整个优化的基础：**知道导数，就知道参数往哪边改、改一点点 loss 会变多少**。

例子：$f(x) = x^2$ 在 $x = 3$ 处，$f'(3) = 6$。如果 $x$ 增加 0.01，loss 大约增加 $6 \times 0.01 = 0.06$（真实值 $3.01^2 - 9 = 0.0601$）。反过来，想让 loss 变小，就让 $x$ 朝导数的反方向走，这就是梯度下降。

注意“约等于”只在 $\Delta x$ 很小时成立。学习率太大，一步走太远，这个线性近似就不准了，loss 可能不降反升，这是学习率不能太大的数学原因。

## 6.2 偏导数与梯度

> **📖 原文 [6.2 偏导数与梯度](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#62-偏导数与梯度)**
>
> 多变量标量函数 $f(x_1,\ldots,x_n)$ 对单个变量的变化率是偏导。把所有偏导排成向量得到梯度：
>
> $$
> \nabla_{\mathbf{x}}f=
> \begin{bmatrix}
> \frac{\partial f}{\partial x_1}\\
> \vdots\\
> \frac{\partial f}{\partial x_n}
> \end{bmatrix}
> $$
>
> 梯度指向函数上升最快的方向，因此梯度下降沿负梯度更新：
>
> $$
> \mathbf{x}_{t+1}
> = \mathbf{x}_t-\eta\nabla f(\mathbf{x}_t)
> $$

### Q23. 梯度和导数有什么区别？为什么梯度“指向上升最快的方向”？

- **导数**：单变量函数的变化率，是一个数；
- **梯度**：多变量函数对每个变量的偏导数排成的向量，是一个方向。

“上升最快”可以这样理解：沿方向 $\mathbf{d}$（长度为 1）走一小步，函数变化约为 $\nabla f \cdot \mathbf{d}$，这是一个点积。由 [原文 2.4](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#24-点积相似度与加权求和) 的 $\mathbf{x}^\top\mathbf{y} = \lVert\mathbf{x}\rVert\lVert\mathbf{y}\rVert\cos\theta$，当 $\mathbf{d}$ 和梯度同方向（$\cos\theta = 1$）时变化最大。所以负梯度方向就是下降最快的方向。

### Q24. 代码里 `param.grad` 的 shape 为什么和 `param` 一样？

因为 loss 是一个**标量**，对每个参数元素都有一个偏导数，所以梯度和参数一一对应、shape 相同。权重是 `(4096, 4096)`，梯度也是 `(4096, 4096)`。

这也意味着**梯度占的显存和参数一样大**，这是训练比推理多占显存的第一个原因（第二个原因是优化器状态，见 [原文 7.3](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#73-adam-的尺度自适应直觉)）。

## 6.3 Jacobian 与 Hessian

> **📖 原文 [6.3 Jacobian 与 Hessian](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#63-jacobian-与-hessian)**
>
> 向量函数 $\mathbf{y}=f(\mathbf{x})$ 的 Jacobian：
>
> $$
> \mathbf{J}_{ij}
> = \frac{\partial y_i}{\partial x_j}
> $$
>
> 它描述每个输入分量对每个输出分量的局部影响。

### Q25. Jacobian 和梯度是什么关系？

梯度是 Jacobian 的特例：

- 函数输出是**标量**（比如 loss）：Jacobian 只有一行，就是梯度（转置一下）；
- 函数输出是**向量**（比如一个隐藏层）：Jacobian 是一个矩阵，第 $i$ 行第 $j$ 列表示第 $j$ 个输入对第 $i$ 个输出的影响。

一个 $(4096 \to 4096)$ 的层，Jacobian 是 $4096 \times 4096$ 的矩阵，每个样本一个。如果对整个 batch、整个网络都显式存 Jacobian，显存完全不够，所以 autograd 从来不存完整的 Jacobian（见下一节）。

## 6.4 链式法则

> **📖 原文 [6.4 链式法则](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#64-链式法则)**
>
> 若：
>
> $$
> y=f(u),\qquad u=g(x)
> $$
>
> 则：
>
> $$
> \frac{dy}{dx}
> = \frac{dy}{du}\frac{du}{dx}
> $$

### Q26. “自动微分不会显式创建巨大 Jacobian，而是计算 vector-Jacobian product”，这是什么意思？

反向传播的公式看上去是 Jacobian 连乘：$\frac{\partial L}{\partial \mathbf{x}} = \frac{\partial L}{\partial \mathbf{h}_2}\frac{\partial \mathbf{h}_2}{\partial \mathbf{h}_1}\frac{\partial \mathbf{h}_1}{\partial \mathbf{x}}$。

关键在于**从左往右算**：最左边的 $\frac{\partial L}{\partial \mathbf{h}_2}$ 是一个向量（因为 $L$ 是标量），向量乘矩阵还是向量。所以每一步都是“向量 × Jacobian = 向量”（vector-Jacobian product，VJP），永远不需要存两个 Jacobian 相乘的结果。

这和 [原文 2.6](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#26-矩阵乘法是一组点积) 讲的“结合顺序决定代价”是同一个道理：$(\mathbf{v}^\top\mathbf{J}_2)\mathbf{J}_1$ 比 $\mathbf{v}^\top(\mathbf{J}_2\mathbf{J}_1)$ 便宜得多。而且每个算子的 VJP 都可以不构造 Jacobian 直接算，比如线性层的 VJP 就是一次矩阵乘法（[原文 6.6](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#66-线性层的反向传播)）。

PyTorch 里 `loss.backward()` 就是在做这件事。如果对非标量调用 `y.backward(v)`，传入的 `v` 就是那个“vector”。

## 6.5 计算图与反向模式自动微分

> **📖 原文 [6.5 计算图与反向模式自动微分](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#65-计算图与反向模式自动微分)**
>
> 设：
>
> $$
> a=xy,\qquad b=a+x,\qquad L=b^2
> $$
>
> 前向阶段保存必要中间量。反向从：
>
> $$
> \frac{\partial L}{\partial L}=1
> $$

### Q27. 原文 $a = xy, b = a + x, L = b^2$ 的例子，能用 PyTorch 验证吗？

取 $x = 2, y = 3$：$a = 6$，$b = 8$，$L = 64$。按原文公式 $\frac{\partial L}{\partial x} = 2b + 2by = 16 + 48 = 64$，$\frac{\partial L}{\partial y} = 2bx = 32$：

```python
x = torch.tensor(2.0, requires_grad=True)
y = torch.tensor(3.0, requires_grad=True)
a = x * y
b = a + x
L = b ** 2
L.backward()
x.grad, y.grad      # (tensor(64.), tensor(32.))
```

和手推完全一致。注意 $x$ 有两条路径影响 $L$：一条是 $x \to a \to b$，一条是 $x \to b$ 直接相加。两条路径的梯度要**相加**，原文的 $2b + 2by$ 就是两项之和。

### Q28. “一个张量被多处使用时要累积梯度”，和 `optimizer.zero_grad()` 有什么关系？

PyTorch 实现“累积”的方式是：**每次 backward 都把梯度加到 `.grad` 上，而不是覆盖**。这样一个张量被多处使用时，每条路径的梯度自然就加在了一起。

副作用是：如果你不清零，**上一轮的梯度也会被加进来**：

```python
w = torch.tensor(1.0, requires_grad=True)
for _ in range(3):
    (w * 2).backward()
w.grad   # tensor(6.)  ← 三次的梯度 2 被加在了一起
```

所以训练循环里每一步都要 `optimizer.zero_grad()`。这个“累加”行为也有正面用途，就是**梯度累积**：显存只够 batch=8，但想要 batch=32 的效果，可以连续做 4 次 forward + backward 再 `step()` 一次：

```python
for i, (x, y) in enumerate(loader):
    loss = model(x, y) / 4          # 除以累积步数，保持梯度尺度
    loss.backward()                 # 梯度在 .grad 上累加
    if (i + 1) % 4 == 0:
        optimizer.step()
        optimizer.zero_grad()
```

### Q29. 前向阶段“保存必要中间量”是保存什么？为什么训练显存比推理大这么多？

反向时要用到前向的中间结果。比如 $L = b^2$ 的导数是 $2b$，反向时就需要知道 $b = 8$；$a = xy$ 对 $x$ 的导数是 $y$，就需要保存 $y$。

整个网络每一层都要存这些“激活值”，直到反向用完才释放。层数越多、batch 越大、序列越长，激活就越大。推理时不需要反向，用 `torch.no_grad()` 包起来就不保存这些中间量，所以显存小很多。

```python
with torch.no_grad():        # 推理时：不建计算图、不存中间量
    out = model(x)
```

“梯度检查点”（gradient checkpointing）就是用时间换显存：前向时只存少数几层的激活，反向需要时再重新算一遍前向。

## 6.6 线性层的反向传播

> **📖 原文 [6.6 线性层的反向传播](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#66-线性层的反向传播)**
>
> 采用 batch 行向量约定：
>
> $$
> \mathbf{Y}=\mathbf{X}\mathbf{W}+\mathbf{b}
> $$
>
> 其中：
>
> $$
> \mathbf{X}\in\mathbb{R}^{B\times d_{in}},\quad
> \mathbf{W}\in\mathbb{R}^{d_{in}\times d_{out}},\quad
> \mathbf{Y}\in\mathbb{R}^{B\times d_{out}}
> $$

### Q30. 线性层反向的三个公式，能用 PyTorch 验证吗？`nn.Linear` 的权重为什么是转置的？

原文用的是行向量约定 $\mathbf{Y} = \mathbf{X}\mathbf{W} + \mathbf{b}$，$\mathbf{W}$ 是 $(d_{in}, d_{out})$。但 PyTorch 的 `nn.Linear` 存的权重是 **$(d_{out}, d_{in})$**，计算时用的是 $\mathbf{Y} = \mathbf{X}\mathbf{W}^\top + \mathbf{b}$。把原文公式里的 $\mathbf{W}$ 换成 `weight.T` 就对上了：

```python
B, din, dout = 4, 3, 5
X = torch.randn(B, din, requires_grad=True)
lin = nn.Linear(din, dout)          # lin.weight.shape == (5, 3) = (dout, din)
Y = lin(X)
G = torch.randn(B, dout)            # 假装这是上游传来的 dL/dY
Y.backward(G)

torch.allclose(X.grad, G @ lin.weight)            # True  dX = G W^T（W^T 就是 lin.weight）
torch.allclose(lin.weight.grad, G.T @ X)          # True  dW^T = G^T X
torch.allclose(lin.bias.grad, G.sum(0))           # True  db = 对 batch 维求和
```

PyTorch 为什么要存成 $(d_{out}, d_{in})$？一个原因是每一行对应一个输出神经元，按行存储时“一个输出神经元的所有权重”在内存里是连续的，和传统的 $\mathbf{y} = \mathbf{W}\mathbf{x}$ 列向量写法一致。看别人的公式和代码时，先确认是哪种约定，再用 shape 去对。

### Q31. 这三个公式怎么记？每次都要推一遍吗？

不用推，**用 shape 凑**就够了，这正是原文给出 shape 检查的用意：

- $d\mathbf{X}$ 必须是 $(B, d_{in})$，手上有 $\mathbf{G}$ $(B, d_{out})$ 和 $\mathbf{W}$ $(d_{in}, d_{out})$，唯一能凑出来的是 $\mathbf{G}\mathbf{W}^\top$；
- $d\mathbf{W}$ 必须是 $(d_{in}, d_{out})$，唯一能凑出来的是 $\mathbf{X}^\top\mathbf{G}$；
- $d\mathbf{b}$ 是 $(d_{out},)$，bias 在每个样本上都加了一次（广播），所以梯度要把 batch 维加起来。

最后一条有个通用规律：**前向时被广播的维度，反向时就要求和**。前向“一份复制成很多份用”，反向“很多份的梯度汇总回一份”。

### Q32. “一个线性层的反向包含两次 GEMM”，训练比推理的计算量大多少？

前向是 1 次 GEMM（$\mathbf{X}\mathbf{W}$），反向是 2 次（$d\mathbf{X}$ 和 $d\mathbf{W}$），规模相同。所以一个训练步的计算量约是前向的 **3 倍**。

这就是估算训练算力时常用的经验公式的来源：训练 $N$ 个参数的模型、$D$ 个 token，总计算量约 $6ND$ FLOPs：前向 $2ND$（每个参数每个 token 一次乘加），反向 $4ND$。

## 6.7 Softmax 的 Jacobian

> **📖 原文 [6.7 Softmax 的 Jacobian](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#67-softmax-的-jacobian)**
>
> Softmax：
>
> $$
> p_i=\frac{e^{z_i}}{\sum_k e^{z_k}}
> $$
>
> 其偏导：
>
> $$
> \frac{\partial p_i}{\partial z_j}
> = p_i(\delta_{ij}-p_j)
> $$

### Q33. 公式里的 $\delta_{ij}$ 是什么？

克罗内克 delta：$i = j$ 时等于 1，否则等于 0。它就是单位矩阵的元素写法，$\delta_{ij} = I_{ij}$。

所以 $\frac{\partial p_i}{\partial z_j} = p_i(\delta_{ij} - p_j)$ 分两种情况：

- $i = j$（对角线）：$p_i(1 - p_i)$，自己的 logit 变大，自己的概率变大；
- $i \ne j$（非对角线）：$-p_i p_j$，别人的 logit 变大，自己的概率变小（因为总和固定为 1）。

### Q34. $\operatorname{diag}(\mathbf{p}) - \mathbf{p}\mathbf{p}^\top$ 这个矩阵公式能验证吗？

```python
z = torch.randn(4)
p = torch.softmax(z, 0)
J = torch.autograd.functional.jacobian(lambda t: torch.softmax(t, 0), z)
torch.allclose(J, torch.diag(p) - torch.outer(p, p), atol=1e-6)   # True
```

`torch.outer(p, p)` 就是 $\mathbf{p}\mathbf{p}^\top$（外积）。

### Q35. “反向仍可在 $O(V)$ 时间内计算，无需物化 $V \times V$ 矩阵”是怎么做到的？

反向需要的是 VJP：$\mathbf{g}^\top\mathbf{J}$，其中 $\mathbf{g}$ 是上游梯度。把公式代进去：

$$
\mathbf{g}^\top(\operatorname{diag}(\mathbf{p}) - \mathbf{p}\mathbf{p}^\top) = \mathbf{g} \odot \mathbf{p} - (\mathbf{g}^\top\mathbf{p})\,\mathbf{p}
$$

$\mathbf{g}^\top\mathbf{p}$ 是一个标量（一次点积），剩下都是逐元素运算，总共 $O(V)$。如果真的构造 $V \times V$ 的 Jacobian，词表 128000 时一个样本就要 $128000^2 \times 4$ 字节 ≈ 65 GB。

```python
dz = p * g - (g @ p) * p     # Softmax 反向的实际实现方式
```

## 6.8 Softmax + 交叉熵的简洁梯度

> **📖 原文 [6.8 Softmax + 交叉熵的简洁梯度](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#68-softmax--交叉熵的简洁梯度)**
>
> one-hot 标签 $\mathbf{y}$，损失：
>
> $$
> L=-\sum_i y_i\log p_i
> $$
>
> 与 Softmax 联合求导后得到：
>
> $$
> \frac{\partial L}{\partial z_i}=p_i-y_i
> $$

### Q36. $p_i - y_i$ 这个梯度能验证一下吗？

```python
z = torch.randn(5, requires_grad=True)
y = 2
F.cross_entropy(z[None], torch.tensor([y])).backward()
z.grad
# tensor([ 0.0308,  0.0700, -0.8895,  0.0734,  0.7154])
torch.softmax(z.detach(), 0) - F.one_hot(torch.tensor(y), 5).float()
# tensor([ 0.0308,  0.0700, -0.8895,  0.0734,  0.7154])  完全一致
```

推导思路（原文练习 4）：$L = -\log p_y$，所以 $\frac{\partial L}{\partial p_y} = -\frac{1}{p_y}$，其他 $\frac{\partial L}{\partial p_i} = 0$。再乘上 Softmax 的 Jacobian 第 $y$ 行：

$$
\frac{\partial L}{\partial z_j} = -\frac{1}{p_y} \cdot p_y(\delta_{yj} - p_j) = p_j - \delta_{yj} = p_j - y_j
$$

$p_y$ 正好约掉了，这就是“简洁”的来源。

### Q37. 这个梯度为什么说“预测越错，修正力度越大”？它有上限吗？

看正确类的梯度 $p_y - 1$：

- 模型很有把握且答对（$p_y = 0.99$）：梯度 $-0.01$，几乎不用改；
- 模型很有把握但答错（$p_y = 0.01$）：梯度 $-0.99$，大力修正。

而且梯度的绝对值**永远不超过 1**（因为 $p_i \in [0, 1]$），不会因为预测极端错误而爆炸。这是 Softmax + 交叉熵在分类任务里训练稳定的重要原因。

对比：如果分别算 Softmax 和 log，中间的 $-1/p_y$ 在 $p_y \to 0$ 时会趋于无穷大，数值上很危险。这就是框架要把两者**融合**成一个算子的原因（[原文 5.3](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#53-logsumexp)）。

## 6.9 梯度检查

> **📖 原文 [6.9 梯度检查](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC2%E7%AB%A0-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80/#69-梯度检查)**
>
> 自动微分实现可用有限差分做小规模检查：
>
> $$
> \frac{\partial f}{\partial x_i}
> \approx
> \frac{f(\mathbf{x}+\epsilon\mathbf{e}_i)
> -f(\mathbf{x}-\epsilon\mathbf{e}_i)}{2\epsilon}
> $$
>
> 中心差分通常比单边差分准确，但 $\epsilon$ 不能太大或太小：太大有截断误差，太小会被浮点舍入误差淹没。梯度检查适合小尺寸、双精度和确定性函数，不适合直接检查带随机性或巨大张量的完整训练任务。

### Q38. 有限差分的 $\epsilon$ 取多少合适？为什么“太小反而不准”？

用 $f(x) = \sin x$ 在 $x = 1$ 处测试，真实导数是 $\cos 1$：

```python
f = lambda t: torch.sin(t).sum()
x0 = torch.tensor([1.0], dtype=torch.float64)
for eps in [1e-1, 1e-4, 1e-8, 1e-12]:
    g = (f(x0 + eps) - f(x0 - eps)) / (2 * eps)
    print(eps, abs(g.item() - math.cos(1.0)))
# 0.1     9.0e-04   ← eps 太大：截断误差（曲线不是直线）
# 1e-4    9.0e-10   ← 最好
# 1e-8    2.6e-09
# 1e-12   1.2e-05   ← eps 太小：舍入误差（两个几乎相等的数相减）
```

误差先降后升，呈 U 形：

- $\epsilon$ 大：用割线近似切线，曲线弯曲带来的误差（截断误差）大；
- $\epsilon$ 小：$f(x+\epsilon)$ 和 $f(x-\epsilon)$ 几乎相等，相减时有效数字被抵消（又是灾难性消减），再除以很小的 $2\epsilon$ 把误差放大。

### Q39. 为什么原文说梯度检查“适合双精度”？float32 不行吗？

float32 的有效数字只有 7 位左右，U 形曲线的最低点高得多：

```python
x32 = torch.tensor([1.0])                  # float32
# eps=1e-1: 9.0e-04   eps=1e-3: 1.4e-05   eps=1e-6: 3.9e-03
```

最好也只有 $10^{-5}$ 量级，根本分不清是“梯度写错了”还是“数值误差”。所以 `torch.autograd.gradcheck` 要求输入必须是 float64：

```python
x = torch.randn(3, dtype=torch.float64, requires_grad=True)
torch.autograd.gradcheck(lambda t: (t ** 3).sum(), (x,))   # True
```

以后写自定义 `autograd.Function` 或 C++ 扩展的反向时，用 `gradcheck` 在 float64 的小输入上测一遍，是最省事的正确性检查。

## 第2章系列目录

- [导读与易错点速查](/posts/ai-infra/ch2-00-guide/)
- [（一）符号、张量与线性代数](/posts/ai-infra/ch2-01-tensor-linear-algebra/)
- [（二）GEMM 分块与概率论](/posts/ai-infra/ch2-02-gemm-probability/)
- **（三）Softmax、交叉熵与反向传播**（本篇）
- [（四）优化、归一化、数值精度与综合算例](/posts/ai-infra/ch2-04-optimization-numerics/)

其他章节：

- [第1章 编程语言基础：导读与易错点速查](/posts/ai-infra/ch1-00-guide/)

