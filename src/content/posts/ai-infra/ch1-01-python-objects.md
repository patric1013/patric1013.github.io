---
title: "AI Infra 第1章笔记（一）：最小工具箱与 Python 对象模型"
published: 2026-11-03T11:00:00
description: "这一组问题的核心只有一句：Python 变量名是贴在对象上的标签，不是装对象的盒子。 赋值、传参、默认参数、拷贝，全都可以用\"标签指向哪个对象、对象本身有没有被改\"来判断。"
tags: ["AI Infra", "Python"]
category: "AI Infra 学习笔记"
draft: false
---

这一组问题的核心只有一句：**Python 变量名是贴在对象上的标签，不是装对象的盒子。** 赋值、传参、默认参数、拷贝，全都可以用"标签指向哪个对象、对象本身有没有被改"来判断。

> [!NOTE]
> 本篇对应原文 [AIInfraGuide · 第1章：编程语言基础](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/) 的 §1.2、§2.1–2.2（作者 [caomaolufei](https://github.com/caomaolufei/AIInfraGuide)，MIT 许可）。每节开头的引用块摘自原文，下面的问答是我学习时提出的问题和整理后的解答；文中的“原文 x.y”都可以点击跳转到原文对应小节。

## 1.2 开始前的最小工具箱

> **📖 原文 [1.2 开始前的最小工具箱](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#12-开始前的最小工具箱)**
>
> 以下命令不要求现在全部掌握，但应该知道它们分别回答什么问题：
>
> ```bash
> python --version       # 正在使用哪个 Python
> which python           # 这个可执行文件来自哪里
> python -m pip --version
>
> g++ --version          # C++ 编译器版本
> cmake --version        # 构建系统版本
>
> uname -a               # 内核与系统信息
> ps aux                 # 有哪些进程
> free -h                # 主机内存使用
> df -h                   # 文件系统容量
>
> nvidia-smi             # 驱动可见的 GPU 与进程
> nvcc --version         # CUDA Toolkit 中的编译器版本
> ```
>
> > **先建立诊断习惯**：环境问题出现时，先记录命令、输出、工作目录和版本，不要立刻反复重装。可观察的信息越完整，定位越快。

[原文 1.2](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#12-开始前的最小工具箱) 列了一组命令，只要求知道"每条命令回答什么问题"。这里补充其中两条：`python -m pip --version` 和 `uname -a`。

### Q1. `python -m pip --version` 里的 `-m` 是什么意思？

`-m` 是 **module** 的缩写：把后面的名字当作 Python 模块来运行。`python -m pip --version` 的意思是：用**当前这个 `python` 解释器**找到 `pip` 模块，执行它（相当于运行 `pip/__main__.py`），并把 `--version` 作为参数传给它。

**和直接写 `pip --version` 的区别**

- `pip` 是 PATH 里的一个可执行脚本，它属于哪个 Python 不一定清楚。
- 机器上同时有系统 Python、conda、venv 时，`pip` 和 `python` 可能指向不同环境，包就装到别处了。
- `python -m pip` 保证 pip 属于你此刻调用的这个 `python`。输出里会显示路径，例如 `pip 24.2 from /home/xxx/miniconda3/envs/myenv/lib/python3.10/site-packages/pip (python 3.10)`。

这正对应[原文 8.1](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#81-venvconda-与容器解决不同层次的问题) 的建议：始终用 `python -m pip`，减少"运行的是 A 环境的 Python，却调用 B 环境的 pip"这类混淆。

**其他常见的 `-m` 用法**

| 命令 | 作用 |
| --- | --- |
| `python -m venv .venv` | 创建虚拟环境 |
| `python -m http.server 8000` | 在当前目录起一个简易 HTTP 服务器 |
| `python -m torch.distributed.run ...` | PyTorch 分布式训练启动 |
| `python -m src.train` | 以包的方式运行项目脚本，相对导入（`from .utils import ...`）才能正常工作 |
| `python -m cProfile -o profile.out train.py` | [原文 3.7](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#37-从慢定位到具体代码) 的函数级 profiling |

最后一种在研究代码里很常用：直接 `python src/train.py` 时相对导入常会报错，在项目根目录用 `python -m src.train` 就能正常运行。

### Q2. `uname -a` 的输出每一段是什么意思？

示例输出：`Linux amax 7.0.0-38-generic #38~24.04.4-Ubuntu SMP PREEMPT_DYNAMIC Mon Sep 14 16:37:11 UTC 2 x86_64 x86_64 x86_64 GNU/Linux`

| 字段 | 含义 |
| --- | --- |
| `Linux` | 内核名称 |
| `amax` | 主机名（和提示符里的 amax 是同一个；AMAX 也是一家服务器厂商） |
| `7.0.0-38-generic` | 内核版本：`7.0.0` 主版本号，`-38` 是 Ubuntu 的第 38 次修订，`generic` 表示通用版内核 |
| `#38~24.04.4-Ubuntu` | 构建编号，`~24.04.4` 表示为 Ubuntu 24.04.4 构建 |
| `SMP` | Symmetric Multi-Processing，支持多 CPU、多核 |
| `PREEMPT_DYNAMIC` | 内核抢占模式可在启动时调整，属于调度细节 |
| `Mon Sep 14 16:37:11 UTC 2…` | 内核编译时间（年份被截断） |
| `x86_64 ×3` | 依次是机器硬件架构、处理器架构、硬件平台，都是 64 位 x86 |
| `GNU/Linux` | 操作系统类型 |

**对实际工作有用的结论**

- 系统是 Ubuntu 24.04，比较新。
- 架构是 x86\_64，下载预编译程序时选 `x86_64` 版本。
- 24.04 默认是 6.x 内核，这里是 7.0，说明管理员装了更新的硬件支持内核，通常为适配新 GPU/CPU。
- 想看发行版信息更直观：`cat /etc/os-release`。

### Q3. `GNU/Linux` 里的 GNU 是什么？

GNU 是一个自由软件项目，全称 **GNU's Not Unix**（递归缩写）。1983 年由 Richard Stallman 发起，目标是做一套可替代 Unix 的自由操作系统。90 年代初工具都写好了，唯独缺内核；1991 年 Linus Torvalds 写出 Linux 内核，两者合起来才是完整系统。

- **Linux**：内核，管理 CPU、内存、硬盘、GPU 等硬件。
- **GNU**：内核之上的一整套基础软件。

| 你每天在用的 GNU 软件 | 作用 |
| --- | --- |
| bash | 你正在用的 shell（Bourne Again Shell） |
| coreutils | `ls`、`cp`、`mv`、`cat`、`chmod`、`mkdir` 等基础命令 |
| gcc | C/C++ 编译器，编译 PyTorch 扩展、CUDA 代码时会用到（[原文 §5](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#5-c-编译链接模板与构建系统)） |
| glibc | C 标准库，几乎所有程序都依赖它 |
| tar、grep、sed、make | 解压、搜索、文本替换、构建 |

**顺带一个关联点**：安装 Starship 时脚本显示 `Platform: unknown-linux-musl`。musl 是 glibc 的替代 C 标准库，用它编译的程序通常静态链接，不依赖系统 glibc 版本，所以在各种 Linux 上都能直接跑——这正是[原文 5.3](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#53-静态库与动态库)/5.4 讲的静态库 vs 动态库、ABI 兼容性问题的一个真实例子。

一句话：**Linux 是内核，GNU 是内核之上的基础工具；日常说的"Linux 系统"严格讲是 GNU/Linux。**

## 2.1 名字、对象与引用

> **📖 原文 [2.1 名字、对象与引用](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#21-名字对象与引用)**
>
> Python 变量更准确的说法是“**名字绑定到对象**”。赋值通常不会复制对象，只会建立新的引用：
>
> ```python
> a = [1, 2]
> b = a
> b.append(3)
>
> print(a)        # [1, 2, 3]
> print(a is b)   # True：指向同一个对象
> print(a == b)   # True：值相等
> ```
>
> 需要区分两个运算：

### Q4. 为什么 `rebind(data)` 之后 `data` 不变？

```python
def mutate(xs: list[int]) -> None:
    xs.append(1)      # 修改对象本身

def rebind(xs: list[int]) -> None:
    xs = [1, 2]       # 只让局部名字 xs 指向新列表

data: list[int] = []
mutate(data); print(data)   # [1]
rebind(data); print(data)   # 仍然是 [1]
```

传参传的是**同一个对象的引用**，既不是副本，也不是外部变量名本身。逐步看：

1. `data = []`：`data ──→ [ ]`（对象 A）
2. `mutate(data)`：参数 `xs` 也指向 A，`data ──→ [ ] ←── xs`。`xs.append(1)` 修改的是 **A 本身**，所以通过 `data` 也能看到 `[1]`。
3. `rebind(data)`：进入时 `xs` 同样指向 A。执行 `xs = [1, 2]` 时，Python **新建对象 B，把标签 `xs` 移到 B 上**；A 没被修改，`data` 也没被改动。
4. 函数结束，局部变量 `xs` 消失，B 无人引用被回收；`data` 一直指向 A，所以仍是 `[1]`。

用 `id()` 验证：`data` 和进入函数时 `xs` 的 id 相同，`xs = [1, 2]` 之后 `xs` 的 id 变了。

| 操作 | 本质 | 调用方能否看到变化 |
| --- | --- | --- |
| `xs.append(1)`、`xs[0] = 5`、`xs.sort()` | 修改对象本身 | 能 |
| `xs = [1, 2]` | 让局部名字指向另一个对象 | 不能 |

**如果确实想在函数里"替换"列表内容**

- 原地替换：`xs[:] = [1, 2]`（切片赋值修改的是同一个对象）
- 返回新列表，由调用方重新赋值：`data = make_new()`（更推荐，意图更清楚）

训练代码里同理：函数里对传入的 `config` 字典或 tensor 做原地修改（如 `x.add_()`），外部也会变；不想影响外部就先 `copy()` / `clone()`。

### Q5. 如果一开始 `data = []`，直接 `rebind(data)` 再 `print(data)`，输出什么？

输出 `[]`。逻辑完全一样：`xs` 只是被换了指向，`data` 指向的空列表 A 从未被修改。**不论 `data` 原来是什么，`rebind` 都改变不了它。**

## 2.2 可变对象、不可变对象与拷贝

> **📖 原文 [2.2 可变对象、不可变对象与拷贝](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#22-可变对象不可变对象与拷贝)**
>
> 常见不可变对象包括 `int`、`float`、`bool`、`str`、`bytes` 和元素均不可变时的 `tuple`；常见可变对象包括 `list`、`dict`、`set` 和大多数自定义实例。

### Q6. `collect_bad` 的意思是"只改 result 对应的列表，传入的列表不会改"吗？

```python
def collect_bad(x: int, result: list[int] = []) -> list[int]:
    result.append(x)
    return result
```

**不是。** 这个函数的坑不在"传入的列表会不会被修改"，而在**默认参数 `[]` 只在 `def` 执行时创建一次，被所有调用共用**。

**情况一：调用时传入列表 → 传入的列表会被修改**

```python
my = [10]
collect_bad(1, my)
print(my)   # [10, 1]
```

`result` 和 `my` 指向同一个对象，`append` 原地修改，所以调用方能看到。这和 `mutate` 是一回事。

**情况二：不传列表 → 用默认的 `[]`，坑在这里**

```python
print(collect_bad(1))   # [1]
print(collect_bad(2))   # [1, 2]   ← 不是 [2]
print(collect_bad(3))   # [1, 2, 3]
print(collect_bad.__defaults__)   # ([1, 2, 3],)
```

```text
定义函数时：  默认值 ──→ [ ]   (对象 D，只此一个)
第 1 次调用：result ──→ D，append(1) → D = [1]
第 2 次调用：result ──→ 同一个 D，append(2) → D = [1, 2]
```

**正确写法：用 `None` 表示"未提供"**

```python
def collect_good(x: int, result: list[int] | None = None) -> list[int]:
    if result is None:
        result = []        # 每次调用都新建
    result.append(x)
    return result
```

### Q7. 情况一是不是也有问题——第二次调用时默认的空列表被"覆盖"了？

**不是。** 情况一里默认对象 D **既没被覆盖，也没被修改，只是这次没用到它**。D 一直保存在函数身上（`collect_bad.__defaults__`）。每次调用 Python 只决定"这一次的局部变量 `result` 指向谁"：传了参数就指向你传的对象，没传就指向 D。

```python
my = [10]
collect_bad(1, my)
print(collect_bad.__defaults__)  # ([],)   ← D 还是空的
print(collect_bad(2))            # [2]     ← 这次才用到 D
print(collect_bad.__defaults__)  # ([2],)
```

情况一是正常行为，不是 bug。只有当你**不希望函数修改传入列表**时，才需要在函数内复制：`result = [] if result is None else list(result)`。

### Q8. tuple、list、set、dict 的区别（详细）

|  | tuple | list | set | dict |
| --- | --- | --- | --- | --- |
| 写法 | `(1, 2, 3)` | `[1, 2, 3]` | `{1, 2, 3}` | `{"a": 1}` |
| 可变 | 否 | 是 | 是 | 是 |
| 有序 | 是 | 是 | 否 | 保持插入顺序（3.7+） |
| 允许重复 | 是 | 是 | 否，自动去重 | 键不重复，值可重复 |
| 访问方式 | 下标 `t[0]` | 下标 `l[0]` | 无下标，只能遍历或 `in` | 按键 `d["a"]` |
| `x in ...` 查找 | O(n) | O(n) | O(1) | 按键 O(1) |
| 能否作 dict 键 / set 元素 | 能（内部元素也须不可变） | 否 | 否 | 否 |

**list：可修改的序列。** 适合数量会变的一串同类数据，如每个 epoch 的 loss、一批文件路径。`append`、`xs[0] = 10`、`sort`、`pop`、切片 `xs[1:3]` 都可用。

**tuple：不可修改的序列。** 适合固定结构，如坐标 `(x, y)`、图像尺寸 `(H, W, C)`、函数返回多个值 `return 224, 224, 3` 再 `h, w, c = get_shape()` 解包。因为不可变，可作 dict 键：`cache = {(224, 224): "small"}`。两个易错点：

- `(1)` 是整数 1，单元素 tuple 必须写 `(1,)`。
- `t = ([1, 2], 3); t[0].append(9)` **可以执行**。tuple 的"不可变"是指它保存的**引用**不能改，但被引用的 list 内部仍可改——又是"标签"模型。也因此 `([1, 2], 3)` 不能作 dict 键。

**set：无序、自动去重。** `set([0, 1, 1, 2, 0])` 得 `{0, 1, 2}`，常用来数类别数。支持 `&` 交集、`|` 并集、`-` 差集。判断"某样本 ID 是否处理过"用 set 比 list 快得多。易错点：`{}` 是空 dict，空 set 要写 `set()`。

**dict：键值映射。** 配置参数、`model.state_dict()`、按名字记录指标都用它。`cfg.get("dropout", 0.1)` 在键不存在时返回默认值而不报错；`for k, v in cfg.items()` 遍历。

**为什么 set / dict 查找快、且键必须不可变？** 底层都是**哈希表**：存入时算 `hash(x)` 直接定位位置，查找几乎一步到位。如果对象内容能变，哈希值也会变，Python 就找不到它原来的位置，所以 `hash([1, 2])` 会报 `TypeError: unhashable type: 'list'`。

| 需求 | 选择 |
| --- | --- |
| 一串数据，要增删改，顺序重要 | list |
| 固定一组值，或要作 dict 的键 | tuple |
| 去重，或频繁判断"是否存在" | set |
| 按名字或 ID 查数据 | dict |

### Q9. 说 `def f(x=())` 安全，是指不用担心 x 的值改变吗？

准确说：**默认值那个对象本身不可能被修改，所以多次调用之间不会互相影响。**

```python
def f(item, x=()):
    x = x + (item,)    # 生成新 tuple，让局部变量 x 指向它
    return x

print(f(1))            # (1,)
print(f(2))            # (2,)   ← 没有累积
print(f.__defaults__)  # ((),)
```

要区分两件事：**局部变量 `x` 可以改指向**（`x += (item,)` 生成新对象，等同 rebind，只影响本次调用）；**默认对象本身不会被修改**。

例外：tuple 里含可变对象就又不安全了——`def g(item, x=([],))` 中 `x[0].append(item)` 会跨调用累积。所以：**只有默认值整体都不可变才安全**，如 `()`、`(1, 2)`、`None`、数字、字符串。

### Q10. Python 3.15 是不是新增了 frozendict？

是的。**PEP 814** 给 Python 3.15 增加了内置类型 `frozendict`，2026 年 2 月被指导委员会接受，作者 Victor Stinner 和 Donghee Na（[PEP 814](https://peps.python.org/pep-0814/)）。它补齐了对称性：tuple 对 list、frozenset 对 set，此前 dict 没有不可变版本。

```python
cfg = frozendict(lr=1e-3, batch_size=64)   # 内置，无需 import
cfg["lr"]           # 正常读取
cfg["lr"] = 5e-4    # TypeError
```

- `|` 合并时同键取右边；`|=` 也不原地改，而是生成新对象（和 tuple 的 `x += (item,)` 一个逻辑）。
- 可作安全默认参数：`def train(cfg=frozendict(lr=1e-3))`。
- 只有所有键和值都可哈希时 frozendict 才可哈希：`frozendict(a=[1, 2])` 能创建但不能 hash（同 tuple 里不能有 list 的规则）。
- 保持插入顺序。
- 3.14 及更早可用 PyPI 的 `frozendict` 包；标准库 `types.MappingProxyType` 只给只读视图，不可哈希，原 dict 仍能被改。

### Q11. 浅拷贝和深拷贝的含义，修改 src 后 shallow 和 deep 是否变化？

- **浅拷贝**：只复制最外层容器，新容器里的元素仍指向原来那些对象。`copy.copy(src)`、`src.copy()`、`list(src)`、`src[:]` 都是浅拷贝。
- **深拷贝**：递归复制所有层级，与原对象完全独立。`copy.deepcopy(src)`。

```python
src = [1, [2, 3]]
shallow = copy.copy(src)
deep = copy.deepcopy(src)
# 外层列表有三个；内层 [2, 3] 被 src 与 shallow 共享，deep 有独立一份
```

**关键看修改的是哪一层：**

| 对 src 的操作 | 修改的是 | shallow 变吗 | deep 变吗 |
| --- | --- | --- | --- |
| `src[1].append(4)` | 内层共享对象 | 变 | 不变 |
| `src[1][0] = 99` | 内层共享对象 | 变 | 不变 |
| `src.append(5)` | 外层容器 | 不变 | 不变 |
| `src[0] = 100` | 外层某个位置 | 不变 | 不变 |
| `src[1] = [7, 8]` | 外层位置改指向新对象 | 不变 | 不变 |
| `src = [0]` | 变量名改指向 | 不变 | 不变 |

```python
src[1].append(4); src.append(5); src[0] = 100
print(src)       # [100, [2, 3, 4], 5]
print(shallow)   # [1, [2, 3, 4]]   ← 只受内层修改影响
print(deep)      # [1, [2, 3]]      ← 完全不受影响
```

容器里只有不可变元素（如 `[1, 2, 3]`）时，浅拷贝和深拷贝效果一样。

**深度学习里的三个例子**（对应原文"深拷贝并非默认答案，先明确哪些独占、哪些只读共享"）：

- 嵌套配置：`cfg2 = cfg.copy()` 后改 `cfg2["optimizer"]["lr"]`，原 `cfg` 也变；需要独立就 `copy.deepcopy(cfg)`。
- 保存最佳权重：`best = model.state_dict()` 只是引用，训练继续时会被改；应 `copy.deepcopy(model.state_dict())` 或对每个 tensor `.clone()`。
- tensor：`b = a` 不复制，`b = a[:]` 通常也是共享底层数据的视图；要独立副本用 `a.clone()`。

### Q12. 所以浅拷贝类似于 C 语言的指针？

方向对，但更准确：**Python 里所有变量本来就类似 C 指针，浅拷贝相当于"复制了一个指针数组"。** 把 `src = [1, [2, 3]]` 想成 C 里的 `Obj* src[] = {a, b};`：

| Python | 类比 C | 结果 |
| --- | --- | --- |
| `x = src` | 只复制指向数组的指针 | x 和 src 是同一个数组 |
| `copy.copy(src)` | 新分配数组，逐个复制里面的指针值 | 新数组，指针仍指向原对象 |
| `copy.deepcopy(src)` | 新分配数组，并递归为每个被指对象分配新内存 | 完全独立 |

与真正 C 指针的不同：没有 `*p` / `&x` 语法；不能做指针运算；对象无引用时自动回收，不需要 `free`、不易出现野指针；CPython 中 `id(x)` 基本就是内存地址，判断同一对象用 `a is b`。

一句话：**变量名 ≈ 指针；浅拷贝 ≈ 复制一层指针；深拷贝 ≈ 连同指向的所有对象一起复制。** 这个类比会在 [第五篇](/posts/ai-infra/ch1-05-cpp-memory/) 的 C++ 指针部分直接派上用场。

## 系列目录

- [导读与易错点速查](/posts/ai-infra/ch1-00-guide/)
- **（一）最小工具箱与 Python 对象模型**（本篇）
- [（二）作用域、类与生成器](/posts/ai-infra/ch1-02-scope-class-generator/)
- [（三）装饰器、上下文管理器与异常处理](/posts/ai-infra/ch1-03-decorator-context-exception/)
- [（四）Python 并发与性能分析](/posts/ai-infra/ch1-04-concurrency-profiling/)
- [（五）C/C++ 内存、生命周期与资源管理](/posts/ai-infra/ch1-05-cpp-memory/)
- [（六）C++ 编译、链接、模板与 CMake](/posts/ai-infra/ch1-06-cpp-build/)
- [（七）Linux 开发基本功与环境隔离](/posts/ai-infra/ch1-07-linux-env/)

