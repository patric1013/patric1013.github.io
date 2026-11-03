---
title: "AI Infra 第1章笔记（四）：Python 并发与性能分析"
published: 2026-11-03T08:00:00
description: "这一章的主线：先分清任务是 I/O 密集还是 CPU 密集 → 理解 GIL 为什么让线程不能并行算 Python → 线程共享内存但要加锁，进程绕开 GIL 但要搬数据 → asyncio 用一个线程靠 await…"
tags: ["AI Infra", "Python", "并发"]
category: "AI Infra 学习笔记"
draft: false
---

这一章的主线：**先分清任务是 I/O 密集还是 CPU 密集 → 理解 GIL 为什么让线程不能并行算 Python → 线程共享内存但要加锁，进程绕开 GIL 但要搬数据 → asyncio 用一个线程靠 `await` 切换 → 测量要预热、重复、端到端 → profiling 找到热点后才写底层扩展。**

> [!NOTE]
> 本篇对应原文 [AIInfraGuide · 第1章：编程语言基础](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/) 的 §3（作者 [caomaolufei](https://github.com/caomaolufei/AIInfraGuide)，MIT 许可）。每节开头的引用块摘自原文，下面的问答是我学习时提出的问题和整理后的解答；文中的“原文 x.y”都可以点击跳转到原文对应小节。

## 3.1 先分清并发与并行

> **📖 原文 [3.1 先分清并发与并行](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#31-先分清并发与并行)**
>
> - **并发（concurrency）**：多个任务在同一时间段内推进；
> - **并行（parallelism）**：多个任务在同一时刻真正执行。
>
> 选择并发模型前，先判断瓶颈：
>
> | 任务类型 | 典型例子 | 常见选择 |
> |----------|----------|----------|
> | I/O 密集 | 请求服务、读取大量小文件 | 线程或 `asyncio` |
> | Python CPU 密集 | 纯 Python 解析、复杂循环 | 多进程或原生扩展 |
> | 原生算子密集 | NumPy/PyTorch/CUDA 运算 | 由底层线程池/GPU 并行，Python 负责调度 |
> | 大数据跨进程 | 数据加载、共享缓存 | 多进程 + 共享内存，谨慎序列化 |

### Q1. 线程与核心的关系

- **核心是硬件**：CPU 里真正执行计算的单元，数量固定。
- **线程是软件**：程序里的一条执行路线，由操作系统创建和调度，想开多少开多少。
- **线程是活，核心是工人**，操作系统把线程分配到核心上。一个核心同一瞬间只能执行一个线程。

|  | 含义 | 需要什么 |
| --- | --- | --- |
| 并行（parallelism） | 真正同时执行 | 多个核心 |
| 并发（concurrency） | 轮流执行，看起来同时 | 一个核心也可以 |

这就是[原文 3.1](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#31-先分清并发与并行) 的定义。"8 核 16 线程"里的"线程"是**逻辑核心**（Intel 超线程 / AMD SMT），一个物理核模拟成两个，共用计算单元，性能不是翻倍。`os.cpu_count()` 返回逻辑核心数。

|  | 能否利用多核并行执行 Python 代码 |
| --- | --- |
| Python 多线程（默认构建） | 不能，受 GIL 限制 |
| Python 多进程 | 能 |
| Python 多线程（free-threaded） | 能 |

实际建议：`num_workers` 设在核心数附近或略小，过大反而抢核心、占内存；`torch.get_num_threads()` 默认通常等于物理核心数，这些 C++ 线程不受 GIL 限制。坑：很多 DataLoader 进程 × 每个进程很多计算线程，总线程数远超核心数，时间浪费在切换上，多进程场景下常要调小每进程线程数。

### Q2. 如何理解进程和线程？

**写字楼比喻**：进程是一家公司，有独立办公室和文件柜（内存），公司之间隔开；线程是公司员工，共用一间办公室和一个文件柜。每家公司至少有一个员工（主线程）。

更准确：**进程是操作系统分配资源的单位，线程是调度执行的单位**——进程负责"拥有"资源，线程负责"执行"代码。

```python
# 线程共享内存
data = []
t = threading.Thread(target=lambda: data.append(1)); t.start(); t.join()
print(data)   # [1]

# 进程不共享内存
if __name__ == "__main__":
    p = Process(target=add); p.start(); p.join()
    print(data)   # []，子进程改的是自己的副本
```

|  | 进程 | 线程 |
| --- | --- | --- |
| 是什么 | 运行中的程序，资源分配单位 | 进程里的执行路线，调度执行单位 |
| 内存 | 各自独立 | 同一进程内共享 |
| 创建开销 | 大（租办公室） | 小（给个座位） |
| 通信方式 | Queue、Pipe、共享内存 | 直接读写同一变量 |
| 崩溃影响 | 一般只影响自己 | 可能拖垮整个进程 |
| 数据竞争 | 基本没有 | 需要加锁 |
| Python 中能否真正并行 | 能，各有 GIL | 默认构建下不能 |

两个实际例子：`DataLoader(num_workers=4)` 启动 4 个子进程，每个都有 `dataset` 的副本，`dataset` 存了大量数据就会成倍占内存；`torchrun --nproc_per_node=4 train.py` 创建 4 个进程各管一张 GPU，`WORLD_SIZE` 就是所有机器上的进程总数。

## 3.2 GIL 到底限制了什么

> **📖 原文 [3.2 GIL 到底限制了什么](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#32-gil-到底限制了什么)**
>
> 常见 CPython 构建使用全局解释器锁（GIL）保护解释器内部状态。同一进程中，通常只有一个线程能同时执行 Python 字节码。因此，两个执行纯 Python CPU 循环的线程一般不能获得两倍速度。

### Q3. 什么是 CPython 构建？

拆成两部分：

- **CPython**：用 C 写的官方 Python 解释器。"Python 语言"是语法规则，"解释器"是真正运行代码的程序；从 python.org、conda 装的几乎都是 CPython（`platform.python_implementation()` 可验证）。
- **构建（build）**：同一份 C 源码打开或关闭不同编译选项，得到行为不同的解释器（同一份菜谱做成微辣版和不辣版）。

| 构建 | 特点 |
| --- | --- |
| 默认构建 | 有 GIL，同一时刻只有一个线程执行 Python 代码 |
| 自由线程构建（free-threaded） | 关闭 GIL，线程可真正并行；3.13 实验性提供，3.14 起官方支持但非默认；可执行文件如 `python3.13t`、`python3.14t` |
| 调试构建（debug） | 额外检查和调试信息，给开发 CPython 或写 C 扩展的人用 |

影响：第三方库要按构建分别编译，wheel 名里 `cp313-cp313` 对应默认构建、`cp313-cp313t` 对应 free-threaded（`cp` = CPython）。查看：`sysconfig.get_config_var("Py_GIL_DISABLED")` 为 1 表示 free-threaded；3.13+ 可用 `sys._is_gil_enabled()` 看运行时 GIL 是否关着。这正是[原文 3.2](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#32-gil-到底限制了什么) 说"GIL 是 CPython 实现细节，不应泛化成所有构建的永恒规则"的原因。

### Q4. 介绍一下 GIL

**GIL（Global Interpreter Lock）：同一时刻，只有拿到这把锁的那一个线程能执行 Python 代码。** 4 个线程、8 个核心，任意瞬间也只有一个线程在跑 Python，其余排队，交替运行（一间办公室只有一支笔）。

**为什么要有它**：CPython 用**引用计数**管理内存（`b = a` 计数 +1，`del` 计数 -1，到 0 回收）。两个线程同时改同一对象的计数可能算错，导致内存泄漏或提前回收崩溃。要么每个对象单独加锁（复杂且慢），要么加一把全局大锁——CPython 选了后者。好处：实现简单、单线程快、写 C 扩展容易。

| 任务类型 | 例子 | 多线程能否提速 |
| --- | --- | --- |
| CPU 密集型 | 纯 Python 计算、循环 | 基本不能（双线程甚至更慢，多了切换开销） |
| I/O 密集型 | 下载、读写文件、网络请求 | 能（等待 I/O 时主动释放 GIL） |

**为什么 NumPy、PyTorch 不太受影响**：核心计算在 C/C++/CUDA 里执行大块运算（如 `y = x @ w`）时主动释放 GIL。真正受限的是纯 Python 部分：自己写的循环、Python 层的数据增强、拼接样本。

**绕开办法**：多进程（每个进程有自己的 GIL，代价是不共享内存、启动慢）——这正是 DataLoader 用 `num_workers` 开**进程**的原因；向量化交给 C 扩展；free-threaded 构建（单线程略慢，依赖库都要支持）。

**常见误解：有 GIL 不代表线程安全。** `count += 1` 分"读取、加 1、写回"三步，GIL 可能在中间切换给别的线程，两个线程读到同一旧值各自写回，只加了一次。GIL 保护解释器内部，不保护你的代码逻辑，共享数据仍要加锁（见 Q7）。

## 3.3 线程：共享内存，也共享风险

> **📖 原文 [3.3 线程：共享内存，也共享风险](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#33-线程共享内存也共享风险)**
>
> ```python
> from concurrent.futures import ThreadPoolExecutor
> from urllib.request import urlopen
>
> def fetch_size(url: str) -> int:
>     with urlopen(url, timeout=10) as response:
>         return len(response.read())
>
> urls = ["https://example.com"] * 4
> with ThreadPoolExecutor(max_workers=4) as pool:
>     sizes = list(pool.map(fetch_size, urls))
> ```
>
> 线程共享进程内存，传数据便宜，但对同一可变对象的“读取—修改—写回”可能产生竞态。需要锁时让临界区尽可能小：
>
> ```python
> from threading import Lock
>
> lock = Lock()
> counter = 0
>
> def increment() -> None:
>     global counter
>     with lock:
>         counter += 1
> ```

### Q5. `ThreadPoolExecutor` + `fetch_size` 这段怎么理解？

**用 4 个线程同时下载 4 个网页，得到每个网页的字节数**，是"I/O 密集适合多线程"的典型例子。

- `urlopen(url, timeout=10)`：最多等 10 秒，防止网站不响应卡死程序。
- `with ... as response`：离开时自动关闭网络连接。`return` 写在 `with` 里没问题——return 前先完成"离开时的事"，和 `try/finally` 一样。
- `response.read()` 得到字节串（bytes），`len(...)` 即字节数。
- `["https://example.com"] * 4`：列表重复 4 次。
- **线程池**：提前准备一组线程，你把任务丢进去它自动分配（外包团队比喻）。`max_workers=4` 最多 4 个线程；离开 `with` 时**等所有任务完成再关闭**。
- `pool.map(fetch_size, urls)`：和内置 `map` 用法一样，只是分给多个线程同时执行；结果**和输入顺序一致**；返回迭代器，`list(...)` 一次取出；某个任务出错时，错误在**取这个结果时**（`list(...)` 这一步）抛出。

```text
不用线程：请求1 ████ → 请求2 ████ → 请求3 ████ → 请求4 ████   约 4 秒
4 个线程：请求1~4 同时 ████                                     约 1 秒
```

若 `fetch_size` 里是大量纯 Python 计算，就换 `ProcessPoolExecutor`，用法完全一样。

### Q6. `with ... as` 的语法

```python
with 表达式 as 变量名:
    代码块
# 大致等价于
对象 = 表达式
变量名 = 对象.__enter__()
try:
    代码块
finally:
    对象.__exit__(...)
```

**`as` 接收的是 `__enter__` 的返回值**：多数时候是对象自己（`open` 的文件对象、`urlopen` 的响应、线程池本身）；用 `@contextmanager` 时是 `yield` 交出的值。

- 用不到时省略 `as`：`with timer(...)`、`with torch.no_grad()`、`with lock`。
- 多个用逗号：`with open("in.txt") as fin, open("out.txt", "w") as fout:`，离开时**按相反顺序**关闭。
- 易误解：`with` 结束后变量**仍存在**，但资源已关闭，`f.read()` 报 `ValueError: I/O operation on closed file.`。

### Q7. `Lock` + `global counter` + `with lock:` 怎么理解？

**让多个线程安全地给同一计数器加 1**，是 GIL 那节 `count += 1` 问题的解决办法。

- `lock = Lock()`：只有"锁着/没锁"两种状态（单人卫生间一把钥匙）。**必须创建在函数外面**，所有线程用同一把；在函数里每次 `Lock()` 等于每人一把钥匙。
- `global counter`：函数里给变量赋值，Python 默认当作局部变量，`counter += 1` 会 `UnboundLocalError`；写了 `global` 才改的是外面那个（只读不赋值不需要，见 [第二篇 Q4](/posts/ai-infra/ch1-02-scope-class-generator/#q4-nonlocal--global-这段是什么意思)）。
- `with lock:`：进入时拿锁，被别人拿着就**停下等待**；离开时自动释放，相当于 `acquire()` + `try/finally: release()`。手动 `acquire` 后出错忘了 `release`，其他线程会永远等下去。

```text
不加锁：A 读 5 → B 读 5 → A 写 6 → B 写 6   ← 少加一次
加锁：  A 拿锁→读 5→写 6→放锁；B（等待…）拿锁→读 6→写 7→放锁
```

8 线程各加 10 万次：不加锁可能小于 800000（默认构建下出错概率较低但仍可能；free-threaded 下几乎每次都错），加锁恰好 800000。

注意：所有修改共享数据的地方用**同一把锁**；**锁住的代码越少越好**——网络请求等耗时操作别放锁里，只锁 `results.append(data)` 那一步（原文"让临界区尽可能小"）。

## 3.4 多进程：绕开 GIL，但数据搬运有成本

> **📖 原文 [3.4 多进程：绕开 GIL，但数据搬运有成本](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#34-多进程绕开-gil但数据搬运有成本)**
>
> ```python
> from concurrent.futures import ProcessPoolExecutor
>
> def count_primes(limit: int) -> int:
>     def is_prime(value: int) -> bool:
>         if value < 2:
>             return False
>         return all(value % divisor for divisor in range(2, int(value**0.5) + 1))
>     return sum(is_prime(value) for value in range(limit))
>
> if __name__ == "__main__":
>     with ProcessPoolExecutor() as pool:
>         results = list(pool.map(count_primes, [50_000] * 4))
> ```
>
> `if __name__ == "__main__":` 对使用 `spawn` 启动子进程的平台尤其重要，可避免子进程重新导入模块时递归创建新进程。

### Q8. `range(2, 3)` 怎么理解？

**包含开头，不包含结尾**，只含一个数 2。`range(2, 2)`、`range(5, 2)` 都是空。

- `range(5)` = `range(0, 5)` → 0–4；`range(100_000)` 正好 10 万个数。
- 第三个参数是步长：`range(0, 10, 2)` → 0, 2, 4, 6, 8；`range(10, 0, -3)` → 10, 7, 4, 1。
- 不含结尾的好处：`range(a, b)` 正好 `b - a` 个数；和下标配合 `range(len(names))`；切片 `names[0:2]` 同规则。
- `range` 不是列表，按需算出，`range(100_000_000)` 几乎不占内存（与生成器思路相同）。

### Q9. `all()` 的语法

判断**一组值是不是全部为真**，有一个为假就 `False`。常配合生成器表达式：`all(n % 2 == 0 for n in nums)`。

| 被当作假的值 | 其他都当作真 |
| --- | --- |
| `False`、`None`、`0`、`0.0`、`""`、`[]`、`{}`、`()` | `1`、`-3`、`"abc"`、`[0]` 等 |

- **空时返回 `True`**（"找不到任何一个为假的"）。
- **短路**：遇到第一个假立刻停止，后面不再检查。

|  | 意思 | 空的时候 | 何时停止 |
| --- | --- | --- | --- |
| `all()` | 全部为真吗 | `True` | 遇到第一个假 |
| `any()` | 至少一个为真吗 | `False` | 遇到第一个真 |

### Q10. `all(value % divisor for divisor in range(2, int(value**0.5) + 1))` 怎么理解？

**判断 value 是不是质数**，从里往外拆：

1. `value**0.5`：开平方，`10 ** 0.5` = 3.162…
2. `int(...) + 1`：向下取整再加 1，因为 range 不含结尾，加 1 才检查到平方根本身。
3. `range(2, ...)`：要尝试的除数，如 29 → `range(2, 6)` → 2, 3, 4, 5。
4. `value % divisor`：余数**不为 0** → 真；**为 0**（能整除）→ 假。
5. `all(...)`：**所有除数都除不尽吗？**

| value | 除数 | 余数 | 结果 |
| --- | --- | --- | --- |
| 29 | 2, 3, 4, 5 | 1, 2, 1, 4 | True，质数 |
| 9 | 2, 3 | 1, 0 | False（遇到 0 立刻停） |
| 7 | 2（`range(2, 3)`） | 1 | True |
| 2 或 3 | 空（`range(2, 2)`） | — | `all([])` = True，正确 |

只查到平方根：若 `36 = 4 × 9`，两个因数里至少一个不超过平方根。判断 100 万只需试到 1000。

注意：单独使用时 0 和 1 也返回 True，所以要先 `if value < 2: return False`。写成 `value % divisor != 0` 更直白。

### Q11. `count_primes` + `ProcessPoolExecutor` 解释一下，如何体现"数据搬运有成本"？

**逐段**：`count_primes(limit)` 统计 0 到 limit-1 有多少个质数，内层 `is_prime` 见上。`sum(is_prime(v) for v in range(limit))` 利用 **`True` 当 1、`False` 当 0**，对布尔值求和就是数 True 的个数。`ProcessPoolExecutor()` 不写 `max_workers` 默认是 `os.cpu_count()`。`[50_000] * 4` 分 4 个任务，`results` 为 `[5133, 5133, 5133, 5133]`。每个进程有自己的 GIL，4 个任务在 4 核上真正并行。

**为什么搬运有成本**：进程内存隔离，`pool.map` 必须真的把数据送过去：

```text
主进程                                     子进程
函数 + 参数 ──① 序列化（pickle）──② 管道传输──③ 反序列化 ──→ 计算
结果        ←─③ 反序列化 ←── ② 传输 ←── ① 序列化 ←────── 返回值
```

还要加上启动进程本身（新解释器、导入模块，几十到几百毫秒）。线程共享内存，没有这个问题。

**这段代码怎么"体现"的**：传进去只有一个整数 `50000`，`range(limit)` 在子进程里自己生成，传回来也只是 `5133`——**计算多、搬运少**，多进程收益才明显。原则：**尽量让子进程拿到"如何生成数据"的信息，而不是数据本身。**

对比高成本写法：主进程准备 `data = list(range(50_000))` 再 `pool.map(count_primes_in, [data] * 4)`，每个任务都要序列化 5 万个整数；几百 MB 的数组搬运时间可能超过计算本身。极端情况 `pool.map(square, range(100_000))`，`x * x` 几乎不耗时，多进程通常**慢几十到上百倍**；`chunksize=10_000` 可缓解，但最好直接单进程算。

| 情况 | 适合多进程吗 |
| --- | --- |
| 计算量大、传入传出数据小（本例） | 适合 |
| 计算量大但要传很大的数据 | 不一定，看搬运和计算谁大 |
| 每个任务计算量很小 | 不适合，搬运和调度成本占主导 |

**内层函数为什么没问题**：进程池传函数实际传的是"函数名 + 所在模块"，子进程按名字导入。`count_primes` 在模块顶层能找到；`is_prime` 在子进程运行时才在内部创建，不需传输。但直接把 lambda 或内层函数交给 `pool.map` 会报 `Can't pickle <function <lambda> ...>`，所以交给进程池的函数要定义在模块顶层。

**回到 PyTorch**：DataLoader 每个 worker 自己读取预处理，只把 batch 传回主进程；`Dataset` 只存路径、在 `__getitem__` 里读文件，和"只传 50000"同一思路（呼应 [第二篇 Q18](/posts/ai-infra/ch1-02-scope-class-generator/#q18-__getitem__-的用途是什么)）。

### Q12. "Windows 和 macOS 子进程会重新导入文件"，Linux 不会吗？

**取决于启动方式**（start method）：

| 方式 | 做法 | 会重新导入主文件吗 |
| --- | --- | --- |
| `fork` | 操作系统整个复制当前进程（复印写到一半的文件）；启动快，但会复制"不该复制的状态"；Windows 不支持 | 不会 |
| `spawn` | 启动全新解释器再导入主文件（白纸重抄）；干净但启动慢 | 会 |
| `forkserver` | 先起一个干净的服务进程，之后从它 fork；兼顾速度和干净 | 会 |

| 系统 | 默认方式 |
| --- | --- |
| Windows | `spawn`（只支持这一种） |
| macOS | `spawn`（Python 3.8 起） |
| Linux，Python 3.13 及以前 | `fork` |
| Linux，Python 3.14 起 | `forkserver` |

所以"Linux 不写也能跑"只对 3.13 及以前成立，最稳妥：**多进程代码一律写 `if __name__ == "__main__":`。**

**fork 的问题**：只复制调用它的那一个线程，却复制整个内存。若复制瞬间别的线程正拿着锁，子进程里这把锁永远是锁着的，一用就卡死。NumPy、PyTorch 后台开很多线程，用过它们再 fork 就有风险。最常见的：`RuntimeError: Cannot re-initialize CUDA in forked subprocess.`

查看与指定：`mp.get_start_method()`；`mp.set_start_method("spawn")`（创建任何进程前调用、只能一次）；局部 `ProcessPoolExecutor(mp_context=mp.get_context("spawn"))`；`DataLoader(..., multiprocessing_context="spawn")`。

### Q13. 如果用线程，创建 `a = [50_000] * 4` 后 map 里就能用 `a` 吗？

可以；但要区分**能不能用**和**有没有搬运成本**。进程池里 `pool.map(count_primes, a)` 也能运行，只是每个元素被序列化、子进程拿到的是**副本**；线程池直接交**引用**，不复制。线程甚至可以不通过参数直接读外部变量 `data[start:end]`，4 个线程读同一份。修改也互相可见：线程里 `results.append(x * 2)` 主线程能看到；进程池里主进程的 `results` 仍是空。（`list.append` 单次调用在 CPython 中安全，`counter += 1` 这种多步操作需要锁。）`pool.map` 可接收多个列表按位置配对。

**但这个例子换线程不会变快**：count\_primes 是纯 Python CPU 计算，GIL 下 4 个线程等于单线程。

|  | 线程池 | 进程池 |
| --- | --- | --- |
| 能否用 `a` 或外部变量 | 能，直接共享 | 能，但拿到副本 |
| 搬运成本 | 几乎没有 | 有（序列化 + 传输） |
| 修改是否互相可见 | 是 | 否 |
| 纯 Python 计算能否并行 | 默认构建下不能 | 能 |

选择依据是任务类型，不是"能不能用变量"；free-threaded 构建下线程既共享内存又能并行，这正是它受关注的原因。

### Q14. 原文"GPU 关联"那段怎么理解？

**CUDA 上下文**：进程第一次用 GPU（`x.cuda()`、`model.to("cuda")`）时，CUDA 为该进程在这张卡上建立的运行环境，含驱动连接、加载的 kernel、显存管理信息。特点：**每个进程每张 GPU 各一个、不能跨进程共享**；建立时**固定占几百 MB 显存**，与模型无关；懒加载。`nvidia-smi` 里每个用了 GPU 的进程都单独一行。

**问题一：初始化错误**。父进程已初始化 CUDA，再 `fork`，子进程拿到的是"看起来存在、实际不可用"的残缺副本（真实状态在 GPU 驱动里，没法简单复制），一用就报 `Cannot re-initialize CUDA in forked subprocess`。改 `spawn` 子进程从零建立自己的上下文。

**问题二：额外显存占用**。不该碰 GPU 的进程意外碰了：

- DataLoader worker 里 `torch.tensor(img).cuda()`：8 个 worker 各建上下文多占好几 GB。应在 worker 只做 CPU 工作，主进程里 `batch.to("cuda", non_blocking=True)`，配合 `pin_memory=True` 加速拷贝。
- 分布式训练所有进程挤到 0 号卡：`model.cuda()` 没指定卡、`torch.load` 把权重加载回原设备 cuda:0。应先 `local_rank = int(os.environ["LOCAL_RANK"]); torch.cuda.set_device(local_rank)`，并 `torch.load("ckpt.pt", map_location="cpu")`。

**"遵循框架推荐的启动器"**：

| 需求 | 推荐方式 |
| --- | --- |
| 单机/多机多卡训练 | `torchrun --nproc_per_node=4 train.py`（自动设 `WORLD_SIZE`、`LOCAL_RANK`） |
| 代码里自己启动多个 GPU 进程 | `torch.multiprocessing.spawn(...)` |
| 数据加载 | `DataLoader` + `num_workers`，worker 只做 CPU 工作 |

### Q15. 什么是"随意 fork"？

**没考虑当前进程是否已用过 GPU，就用 `fork` 创建还要用 GPU 的子进程。** fork 本身不是错的：

- 父进程没碰 GPU → fork → 子进程自己用 GPU：没问题
- 父进程用过 GPU → fork → 子进程只在 CPU 上干活：没问题（Linux 上 DataLoader worker 常是 fork 出来的）
- 父进程用过 GPU → fork → 子进程也要用 GPU：**报错**

容易"不小心"的原因：没指定启动方式，Linux + 3.13 及以前的 `Pool`、`Process`、`ProcessPoolExecutor` 默认都是 fork；不知道父进程已初始化 CUDA（某个库内部偷偷用了 GPU，可用 `torch.cuda.is_initialized()` 检查）；直接 `os.fork()`。避免：子进程需要 GPU 时显式 `mp.spawn(worker, nprocs=4)` 或 `set_start_method("spawn")`；训练用 `torchrun`；worker 只做 CPU。

### Q16. 为什么线程不需要 `if __name__ == "__main__"`，而进程需要？

**创建线程不会启动新的 Python，创建进程（spawn）会。**

- 线程：在同一解释器里多开执行路线，`work` 已在内存里，直接调用；`main.py` 只执行一遍。
- spawn 进程：全新空白解释器，不知道 `work` 是什么；主进程只告诉它"去 main.py 找叫 work 的函数"，于是子进程**导入 main.py**，而导入会从上到下执行顶层代码，又创建进程池……无限套娃（Python 会检测到并报 `RuntimeError`）。

```text
主进程执行 main.py
├── 定义 work
└── __name__ == "__main__" 成立 → 创建进程池 → 启动子进程
    └── 子进程导入 main.py
        ├── 定义 work                 ← 拿到需要的函数
        └── __name__ 不是 "__main__" ← 跳过，不再创建进程池
```

这行代码把文件分成\*\*"定义"部分\*\*（谁导入都执行）和\*\*"启动"部分\*\*（只有直接运行才执行）。

|  | 线程 | 进程（spawn / forkserver） | 进程（fork） |
| --- | --- | --- | --- |
| 是否启动新 Python | 否 | 是 | 否，复制当前进程 |
| 怎么拿到函数 | 内存里本来就有 | 重新导入主文件 | 复制来的内存里有 |
| 主文件是否再执行一次 | 否 | 是 | 否 |
| 是否必须写 | 否 | 是 | 否，但推荐写 |

线程写上也是好习惯：别人 `import` 你的文件只想用某个函数时，不会顺带把整个程序跑一遍。

### Q17. 子进程导入时 `__name__` 不是 `"__main__"`，那子进程叫什么？

要分清两个名字：

| `__name__`（模块名） | 值 |
| --- | --- |
| 直接运行 `python main.py` | `"__main__"` |
| 被 `import main` | `"main"` |
| 被 spawn 子进程导入 | `"__mp_main__"` |

在 `if` 外写一行 `print("当前 __name__ =", __name__)`，spawn 时会依次打印 `__main__`、`__mp_main__`、`__mp_main__`——再次证明子进程会执行顶层代码；fork 方式不会多打印。

**进程名**用 `mp.current_process().name`：主进程 `MainProcess`；子进程 `SpawnProcess-1`、`ForkProcess-1`、`ForkServerProcess-1`……操作系统层面真正区分进程的是 PID（`os.getpid()`）。

## 3.5 `asyncio`：协作式 I/O 并发

> **📖 原文 [3.5 `asyncio`：协作式 I/O 并发](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#35-asyncio协作式-io-并发)**
>
> `asyncio` 用事件循环调度协程。协程只会在 `await` 处主动让出控制权：
>
> ```python
> import asyncio
>
> async def worker(name: str, delay: float) -> str:
>     await asyncio.sleep(delay)  # 模拟非阻塞 I/O
>     return name
>
> async def main() -> None:
>     results = await asyncio.gather(
>         worker("a", 0.2),
>         worker("b", 0.1),
>     )
>     print(results)
>
> asyncio.run(main())
> ```
>
> 在事件循环中直接执行阻塞 I/O 或长时间 CPU 计算会卡住所有协程。此时应改用异步库，或把阻塞工作放到线程/进程执行器。

### Q18. 通过 `worker("a", 0.2)`、`worker("b", 0.1)` 的例子介绍 async

**只用一个线程，任务在等待时主动让出，让其他任务先运行。** 厨师比喻：烧上水不站着等，先去切菜。这个"厨师"就是**事件循环**（event loop）。

和线程的区别：线程由操作系统决定何时切换，可能在任意位置被打断；**async 只在写了 `await` 的地方切换，由任务自己决定何时让出**（原文"协作式"）。

- `async def`：定义**协程函数**，调用时不执行，返回协程对象 `<coroutine object worker at 0x...>`——和生成器很像，Python 协程最早就是在生成器基础上发展出来的。
- `await asyncio.sleep(delay)`：在这里暂停，把控制权交还事件循环（又像 `yield`）。`asyncio.sleep` 是**非阻塞**的（"0.2 秒后叫醒我"）；`time.sleep` 是**阻塞**的，整个线程停住，事件循环也卡住。这里用 sleep 模拟等网络、等数据库。
- `asyncio.gather(...)`：把多个协程交给事件循环**同时运行**，`await` 等全部完成，结果按**传入顺序**排列（`['a', 'b']`，即使 b 先完成）。
- `asyncio.run(main())`：创建事件循环 → 运行 main 到结束 → 关闭；整个程序通常只在最外层调用一次。

```text
0.00  运行 main，gather 启动 a 和 b
0.00  a 遇到 await sleep(0.2) → 让出
0.00  b 遇到 await sleep(0.1) → 让出
0.10  b 完成
0.20  a 完成 → gather 收集 ['a', 'b'] → print
```

总耗时约 0.2 秒（最长任务的时间）。**`async def` 本身不会让代码变快，要用 `gather` 这类工具让多个任务同时运行。**

**最常见的坑**：异步函数里调用 `time.sleep`、普通 `requests.get()`、普通文件读写或大量纯 Python 计算，会阻塞事件循环（原文"会卡住所有协程"）。要配合异步库，如 `aiohttp`、`httpx` 异步接口。

|  | 多进程 | 多线程 | async |
| --- | --- | --- | --- |
| 用几个线程 | 每进程至少一个 | 多个 | 一个 |
| 谁决定切换 | 操作系统 | 操作系统，任意位置 | 任务自己，只在 `await` 处 |
| 适合任务 | CPU 密集 | I/O 密集 | I/O 密集，尤其大量并发 |
| 资源开销 | 大 | 中 | 很小，一个任务只是一个对象 |
| 数据竞争 | 基本没有 | 需要加锁 | 较少 |
| 对 CPU 计算有帮助吗 | 有 | 默认构建下没有 | 没有 |

1 万个网络请求开 1 万个线程开销很大，1 万个协程只是 1 万个小对象。AI Infra 里**模型推理服务**大量使用 async（FastAPI、vLLM 服务端），因为大部分时间在等网络或等 GPU。

### Q19. `await` 结束后 a 好了，是会等 b 做完才回到 a 吗？

**不会。哪个任务等待结束，事件循环就尽快回到哪个任务。** `a` 0.1 秒、`b` 0.3 秒时，输出依次是：a 开始 → b 开始 → a 等待结束继续执行 → b 等待结束。

唯一限制：事件循环**不能打断正在运行的任务**。若 a 准备好那一刻 b 正在执行不含 `await` 的 `time.sleep(0.5)`，a 本该 0.1 秒回来，会被拖到 0.55 秒。

要区分：**`a` 自己**等待一结束就继续，不等 b；**`main`** 在 `gather` 处要等 a 和 b **都**完成才继续。

### Q20. 为什么要用 `gather`，直接调用 worker a 和 b 不行吗？

| 写法 | 结果 |
| --- | --- |
| `worker("a", 0.2)` 不加 `await` | 根本不执行，警告 `RuntimeWarning: coroutine 'worker' was never awaited` |
| 依次 `await worker(...)` | 能执行，但一个接一个，约 0.3 秒 |
| `asyncio.gather(...)` | 先全部提交，再一起等，约 0.2 秒 |
| `create_task` / `TaskGroup` | 与 gather 效果相同，写法更灵活 |

依次 `await` 时，main 卡在第一个 `await`，b 还没被创建交给事件循环，事件循环只知道 a 一个任务，只能空等。关键：**任务要先被"提交"给事件循环，才可能同时运行。**

```python
task_a = asyncio.create_task(worker("a", 0.2))   # 立刻提交，开始运行
task_b = asyncio.create_task(worker("b", 0.1))
r1 = await task_a; r2 = await task_b             # 约 0.2 秒

async with asyncio.TaskGroup() as tg:            # 3.11+，某任务出错会自动取消其他任务
    task_a = tg.create_task(worker("a", 0.2))
    task_b = tg.create_task(worker("b", 0.1))
```

（`async with` 是 `with` 的异步版本，离开时等组内所有任务完成。）

### Q21. 为什么 `asyncio.gather` 前面需要 `await`？

`gather(...)` 返回的不是结果，而是 **Future**（未完成的结果）。去掉 `await`，`print(results)` 输出 `<_GatheringFuture pending>`，且 main 打印完就结束，`asyncio.run` 关闭事件循环，a、b 被直接取消。**取餐小票比喻**：点完餐拿到的是小票，`await` 是等叫号并取餐。

更关键：`await` 是 **main 让出控制权**的地方。事件循环只有一个线程，main 运行时占着它；gather 虽登记了 a、b，要等 main 让出才轮得到它们。

| 写法 | `await` 在等什么 |
| --- | --- |
| `await asyncio.sleep(0.2)` | 等 0.2 秒过去 |
| `await worker("a", 0.2)` | 等协程结束拿返回值 |
| `await asyncio.gather(...)` | 等所有任务结束拿结果列表 |

## 3.6 正确做基准测试

> **📖 原文 [3.6 正确做基准测试](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#36-正确做基准测试)**
>
> 性能测试最常见的错误是只测一次、把初始化算进去，或者忘记 GPU 异步执行。

### Q22. `for _ in range(warmup):` 这个下划线是什么？

普通变量名，惯例表示\*\*"这个变量我用不到"\*\*。语法上 for 必须有变量，这里只为重复 5 次；写 `i` 也能运行，但读者会以为后面要用，代码检查工具也会提示未使用变量。其他用法：拆包忽略 `name, _, score = (...)`、`for _, label in dataset`；交互环境里表示上一次结果。

| 写法 | 含义 |
| --- | --- |
| `for _ in ...` | 用不到的变量 |
| `self._radius` | 内部使用，别直接碰 |
| `100_000` | 数字里只为方便阅读 |
| `__name__`、`__init__` | 前后双下划线：Python 内置特殊名字 |

这段代码在做**预热**：前几次运行往往较慢（加载缓存、分配内存；GPU 上还可能初始化 CUDA、编译 kernel），先空跑几次排除一次性开销，再重复 `repeat` 次正式计时（原文"把初始化算进去"是常见错误）。

## 3.7 从“慢”定位到具体代码

> **📖 原文 [3.7 从“慢”定位到具体代码](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#37-从慢定位到具体代码)**
>
> 建议按由粗到细的顺序：
>
> 1. **端到端计时**：确认回归真实存在；
> 2. **阶段计时**：区分数据、计算、通信和保存；
> 3. **函数级 profiling**：用 `cProfile` 找累计耗时高的函数；
> 4. **行级/采样 profiling**：进一步定位热点行或原生调用栈；
> 5. **内存 profiling**：用 `tracemalloc` 等工具找 Python 分配增长；
> 6. **GPU profiling**：进入 PyTorch Profiler、Nsight Systems/Compute 等专用工具。
>
> ```bash
> python -m cProfile -o profile.out train.py
> python -m pstats profile.out
> ```

### Q23. "端到端计时：确认回归真实存在"怎么理解？

- **回归**：性能回归（performance regression），代码改了之后**变慢了**，如合并提交后 epoch 从 10 分钟变 12 分钟。不是统计学的线性回归。
- **端到端**：从开始到结束完整一段流程（整个训练 step、一个推理请求从收到到返回），测的是用户真正感受到的耗时；相对的是局部计时（`model.forward`、tokenizer）。

**为什么第一步做这个**：感觉到的"变慢"可能不是代码问题——别的程序占了 CPU/GPU、缓存差异、只测一次碰巧波动、GPU 降频或没预热。不先确认就逐个函数排查，可能白忙。

**怎么确认**：相同条件只改一个变量（[原文 9.4](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#94-一个可复用的定位框架)"单变量实验"）；预热；重复多次看中位数；差距要明显大于波动（本身 ±5% 波动，变慢 3% 可能是噪声，20% 才说明有问题）。GPU 上要在计时前后 `torch.cuda.synchronize()`，否则只量到"发出指令"的时间。

排查顺序：**端到端计时 → profiling 找时间花在哪 → 局部计时/微基准针对可疑代码 → 再端到端确认整体变快**。像看病先量体温。

### Q24. `python -m cProfile -o profile.out train.py` 和 `python -m pstats profile.out`

第一条**记录**：`-m` 以模块方式运行（[第一篇 Q1](/posts/ai-infra/ch1-01-python-objects/#q1-python--m-pip---version-里的--m-是什么意思)），`cProfile` 是自带的分析器，记录每个函数被调用多少次、花多少时间；`-o profile.out` 保存到文件（不写就打印到屏幕，很长）。被分析时程序会变慢，可先只跑几十个 step。

第二条**查看**：进入 `%` 提示符的交互界面。

```text
% sort cumulative     按累计耗时排序
% stats 20            显示前 20 行
% sort tottime        按自身耗时排序
% stats dataset       只显示名字含 dataset 的函数
% callers __getitem__ 谁调用了它
% callees train_step  它调用了谁
% quit
```

| 列 | 含义 |
| --- | --- |
| `ncalls` | 调用次数 |
| `tottime` | 函数**自身**时间，不含它调用的函数 |
| `percall`（第一个） | `tottime / ncalls` |
| `cumtime` | **累计**时间，含它调用的所有函数 |
| `percall`（第二个） | `cumtime / ncalls` |

`train_step` 调用 `load_batch`（3 秒）和 `run_model`（2 秒）时，它的 tottime 接近 0，cumtime 是 5 秒。**按 cumtime 找"哪一大块"慢，按 tottime 找"真正在干活"的那个函数。** 例：`__getitem__` 的 tottime 30 秒占一半多 → 数据加载是瓶颈，先加 `num_workers`、减少预处理。

也可在代码里 `pstats.Stats("profile.out").sort_stats("cumulative").print_stats(20)`，或 `pip install snakeviz; snakeviz profile.out` 在浏览器看色块图。

**深度学习注意**：cProfile 只看得到 Python 层。GPU 异步，`model(x)` 发出指令立刻返回，`forward` 显示很短；`loss.item()`、`.cpu()`、`synchronize()` 要等 GPU，会显得很长——时间被记在"等待的地方"而不是"计算的地方"。CPU 瓶颈用 cProfile，GPU 瓶颈用 `torch.profiler`（原文第 6 步）。

### Q25. pstats 的目的是什么？

**读取并分析 cProfile 的记录，整理成人能看懂、能筛选的表格。** `profile.out` 是二进制（`cat` 出来是乱码），且记录了几千上万条函数调用。分成两步的好处：程序只跑一次、结果可换角度反复分析；方便改代码前后对比（`profile_old.out` vs `profile_new.out`）定位回归，也能发给同事。**行车记录仪（cProfile）+ 回放剪辑软件（pstats）。**

## 3.8 Python 性能优化的优先级

> **📖 原文 [3.8 Python 性能优化的优先级](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#38-python-性能优化的优先级)**
>
> 推荐顺序如下：
>
> 1. 先改算法复杂度和数据结构；
> 2. 减少不必要的 I/O、复制和序列化；
> 3. 把细粒度 Python 循环改成批处理/向量化原生算子；
> 4. 减少 Python 与 C++/GPU 边界的频繁往返；
> 5. 确认热点后，再写 C++、CUDA 或 Triton 扩展。
>
> 例如，一百万次小算子调用即使每次都很快，也会累积大量调度开销。把数据组成批次后调用一次大算子，通常比微调 Python 语法有效得多。

### Q26. "确认热点后，再写 C++、CUDA 或 Triton 扩展"怎么理解？

**热点（hotspot）**：少数几个函数占了大部分运行时间，要靠 profiling 拿数据确认，不能靠猜。

| 扩展 | 用在哪里 | 解决什么问题 |
| --- | --- | --- |
| C++ | CPU | 纯 Python 循环太慢；编译后快得多且不受 GIL 限制 |
| CUDA | GPU | 自己写 kernel，实现 PyTorch 没有的算子或融合多个算子减少显存读写（如 FlashAttention） |
| Triton | GPU | 语法接近 Python 的 kernel 语言，比 CUDA 易上手；`torch.compile` 内部自动生成 Triton 代码 |

**为什么一定要先确认**：

1. **优化非热点几乎没效果（阿姆达尔定律）**：100 秒里 `log_metrics` 只占 1 秒，加速 100 倍后整体 99.01 秒，只快约 1%；`__getitem__` 占 50 秒，只加速一倍就到 75 秒，快 25%。整体能快多少取决于被优化部分原本的占比。
2. **写扩展代价高**：难写（内存、线程、线程块、显存布局）、难调试（直接崩溃）、难维护（编译、适配 CUDA 版本/GPU 型号/系统）、易出隐蔽错（算错不报错）。
3. **直觉经常错**：以为模型计算最慢，实际是数据加载让 GPU 空等、循环里频繁 `loss.item()` 同步、CPU 和 GPU 反复拷贝——改几行 Python 就能解决。

**实际顺序**（对应[原文 3.8](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#38-python-性能优化的优先级) 五条）：调整用法（`num_workers`、`pin_memory`、减少 `.item()`）→ 用现成高效实现（向量化、`scaled_dot_product_attention`）→ 编译工具（`torch.compile`）→ 最后才针对确认过的热点写 Triton/CUDA/C++。**测量 → 找热点 → 先简单方法 → 不够再写扩展 → 再测量。**

## 系列目录

- [导读与易错点速查](/posts/ai-infra/ch1-00-guide/)
- [（一）最小工具箱与 Python 对象模型](/posts/ai-infra/ch1-01-python-objects/)
- [（二）作用域、类与生成器](/posts/ai-infra/ch1-02-scope-class-generator/)
- [（三）装饰器、上下文管理器与异常处理](/posts/ai-infra/ch1-03-decorator-context-exception/)
- **（四）Python 并发与性能分析**（本篇）
- [（五）C/C++ 内存、生命周期与资源管理](/posts/ai-infra/ch1-05-cpp-memory/)
- [（六）C++ 编译、链接、模板与 CMake](/posts/ai-infra/ch1-06-cpp-build/)
- [（七）Linux 开发基本功与环境隔离](/posts/ai-infra/ch1-07-linux-env/)

