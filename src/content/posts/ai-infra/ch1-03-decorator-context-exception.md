---
title: "AI Infra 第1章笔记（三）：装饰器、上下文管理器与异常处理"
published: 2026-11-03T09:00:00
description: "这四小节共享一个主题：在\"调用边界\"或\"代码块边界\"附加行为。装饰器包住整个函数，with 包住一段代码，try/finally 保证收尾一定发生，raise ... from 让错误在边界上带着根因往外传。"
tags: ["AI Infra", "Python"]
category: "AI Infra 学习笔记"
draft: false
---

这四小节共享一个主题：**在"调用边界"或"代码块边界"附加行为**。装饰器包住整个函数，`with` 包住一段代码，`try/finally` 保证收尾一定发生，`raise ... from` 让错误在边界上带着根因往外传。

> [!NOTE]
> 本篇对应原文 [AIInfraGuide · 第1章：编程语言基础](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/) 的 §2.6–2.9（作者 [caomaolufei](https://github.com/caomaolufei/AIInfraGuide)，MIT 许可）。每节开头的引用块摘自原文，下面的问答是我学习时提出的问题和整理后的解答；文中的“原文 x.y”都可以点击跳转到原文对应小节。

## 2.6 装饰器：在调用边界附加行为

> **📖 原文 [2.6 装饰器：在调用边界附加行为](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#26-装饰器在调用边界附加行为)**
>
> 装饰器接收一个可调用对象，并返回另一个可调用对象。下面的计时器保留了原函数的元信息：
>
> ```python
> from collections.abc import Callable
> from functools import wraps
> from time import perf_counter
> from typing import ParamSpec, TypeVar
>
> P = ParamSpec("P")
> R = TypeVar("R")
>
> def timed(fn: Callable[P, R]) -> Callable[P, R]:
>     @wraps(fn)
>     def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
>         start = perf_counter()
>         try:
>             return fn(*args, **kwargs)
>         finally:
>             elapsed_ms = (perf_counter() - start) * 1_000
>             print(f"{fn.__name__}: {elapsed_ms:.3f} ms")
>     return wrapper
>
> @timed
> def preprocess(values: list[float]) -> list[float]:
>     return [value * 2 for value in values]
> ```
>
> `@timed` 等价于 `preprocess = timed(preprocess)`。`functools.wraps` 会保留函数名、文档字符串等属性，这对日志、测试和框架反射很重要。

### Q1. 原文 `timed` 装饰器是什么意思？（逐行，先去掉类型注解）

**前提：函数也是一个"东西"。** `f = hello`（无括号）是把函数本身交给 f，`f()` 才是调用。函数既能当参数传入，也能被返回——装饰器就靠这点。

去掉注解后的等价代码：

```python
from functools import wraps
from time import perf_counter

def timed(fn):                     # fn 是一个函数，如 preprocess
    @wraps(fn)                     # 把 fn 的名字、文档复制给 wrapper
    def wrapper(*args, **kwargs):  # 只是定义，不运行
        start = perf_counter()     # 调用前：记开始时间
        try:
            return fn(*args, **kwargs)   # 调用原函数，原样返回
        finally:
            elapsed_ms = (perf_counter() - start) * 1_000
            print(f"{fn.__name__}: {elapsed_ms:.3f} ms")
    return wrapper                 # 返回函数本身，不调用

@timed
def preprocess(values):
    return [value * 2 for value in values]
```

（原文末尾的 `Copy` 是网页的复制按钮，不是代码。）

`@timed` 等价于 `preprocess = timed(preprocess)`：**从此 `preprocess` 这个名字指向 `wrapper`**，原函数保存在 `wrapper` 内部的 `fn` 里（闭包，见 [第二篇](/posts/ai-infra/ch1-02-scope-class-generator/)）。

调用 `preprocess([1.0, 2.0])` 的完整过程：

1. 实际执行 `wrapper([1.0, 2.0])`，此时 `args = ([1.0, 2.0],)`。
2. 记录 `start`。
3. 调用 `fn([1.0, 2.0])` 得 `[2.0, 4.0]`，准备返回。
4. 返回前先执行 `finally`，打印 `preprocess: 0.002 ms`。
5. `[2.0, 4.0]` 交给调用者。

这就是标题"在调用边界附加行为"：结果不变，只在调用前后多做了事。`1_000` 就是 1000，`:.3f` 保留三位小数。

### Q2. `P = ParamSpec("P")`、`R = TypeVar("R")` 怎么理解？

**`P`、`R` 是占位符**，像数学里的未知数。`timed` 要装饰各种函数，无法写死参数类型，但想表达"装饰后参数和返回值跟原来一样"——"原来的那个"需要一个名字来指代。

- 装饰 `preprocess` 时：`P = (values: list[float])`，`R = list[float]`。
- 装饰 `add(a: int, b: int) -> int` 时：`P = (a: int, b: int)`，`R = int`。
- `R` 是 `TypeVar`，代表**一个类型**；`P` 是 `ParamSpec`，代表**整个参数列表**（几个参数、名字、类型、默认值打包成一体），`TypeVar` 表达不了它。
- `*args: P.args, **kwargs: P.kwargs` 表示"wrapper 接收的参数正好是 P 那一套"。

不用它们写成 `Callable[..., Any]`，运行完全一样，但 IDE 不再知道 `preprocess` 接收什么、返回什么，`preprocess("hello")` 不会有提示。**只影响类型检查，运行时不起作用。**

**这两行语法本身**：是普通赋值，创建两个对象。左边变量名和括号里字符串**必须相同**（旧语法的历史包袱，对象创建时不知道自己会赋给哪个变量）；`R = TypeVar("T")` 会被类型检查器报错。名字惯例：`T` 泛型、`R` 返回值、`P` 参数、`K`/`V` 键值。Python 3.12+ 可写成 `def timed[**P, R](fn: Callable[P, R]) -> Callable[P, R]:`，`**P` 与 `**kwargs` 呼应，且只在函数内有效。

### Q3. `*args, **kwargs` 的作用；`*` 和 `**` 的区别

**一句话：`*` 处理按位置传的参数（元组），`**` 处理按名字传的参数（字典）。**

```python
def f(*args, **kwargs):
    print(args, kwargs)
f(1, 2, x=3, y=4)   # (1, 2) {'x': 3, 'y': 4}

def intro(name, age): ...
intro(*("小明", 20))                  # 拆成 intro("小明", 20)
intro(**{"name": "小明", "age": 20})  # 拆成 intro(name="小明", age=20)
intro(*{"name": ..., "age": ...})     # 只拆出键：intro("name", "age")
```

|  | `*` | `**` |
| --- | --- | --- |
| 处理什么参数 | 按位置传 `f(1, 2)` | 按名字传 `f(x=1)` |
| 定义函数时 | 打包成元组 | 打包成字典 |
| 调用函数时 | 把列表/元组拆开 | 把字典拆开 |

- `args`、`kwargs` 只是惯例名（kwargs = keyword arguments），起作用的是星号。
- 顺序必须 `*` 在前 `**` 在后，`def f(**kwargs, *args)` 是语法错误。
- 装饰器里：`wrapper(*args, **kwargs)` 先全收下，`fn(*args, **kwargs)` 再原样拆开，像一个**中转站**。例：`add(1, 2, c=3)` → `args=(1, 2)`、`kwargs={'c': 3}` → `fn(1, 2, c=3)`。

### Q4. 所以函数后加括号，其实可以理解为元组？

**不对。** 函数后的括号是"调用"语法，不是元组。只有定义里写了 `*args`，Python 才把多出来的位置参数**收集**成元组。证据：

- `f(1, 2, x=3)` 合法，`(1, 2, x=3)` 是语法错误。
- 元组靠的是逗号：`(1)` 是整数，`(1,)` 才是元组；而 `f(1)` 不需要写成 `f(1,)`。
- `def g(a, b)` 调用 `g(1, 2)`，函数内根本没有元组；`def h(*args)` 调用 `h(1, 2)` 才得 `(1, 2)`。
- `h((1, 2))` 得 `((1, 2),)`，那是传了一个本身是元组的参数。

直觉对的一点：位置参数是"按顺序排列的一串值"，和元组很像，所以 Python 选用元组装它们——但那是 `*args` 的设计。

### Q5. 为什么要 `@wraps(fn)`？"wrapper 冒充 preprocess"这句不理解

每个函数都有 `__name__`（名字）和 `__doc__`（三引号里的说明）。装饰后 `preprocess` 这个变量里装的其实是 `wrapper`，不加 `wraps`：

```python
print(preprocess.__name__)   # wrapper   ← 名字变了
print(preprocess.__doc__)    # None      ← 说明丢了
```

**外套比喻**：给人套了件外套，问"你叫什么"，回答的是外套标签。若给 `load_data`、`preprocess`、`train` 都加 `@timed`，它们的 `__name__` 全是 `'wrapper'`，日志只会显示"wrapper 出错了"。测试框架、Web 框架（pytest、FastAPI、Click）也靠函数名识别函数。`wraps` 把原函数的标签复制到 wrapper 上，对应原文"对日志、测试和框架反射很重要"。

（`timed` 内部打印用的是 `fn.__name__`，`fn` 是原函数，不受影响；`wraps` 解决的是**外部代码**看到的名字。）

### Q6. 如果 `preprocess` return 的是另一个函数，`preprocess.__name__` 是什么？

还是 `'preprocess'`（加了 `wraps` 的前提下）。`__name__` 是函数本身的属性，与返回什么无关：

| 写法 | 加了 `@wraps` | 没加 `@wraps` |
| --- | --- | --- |
| `preprocess.__name__` | `'preprocess'` | `'wrapper'` |
| `preprocess().__name__` | `'make_inner'` | `'make_inner'` |

`preprocess.__name__` 问"这台榨汁机叫什么牌子"，`preprocess().__name__` 问"它榨出来的是什么果汁"。

### Q7. `try` 和 `finally` 分别什么时候执行？

不是"触发"：**`try` 里的代码总是先执行；`finally` 总是在 `try` 结束后执行，不管正常结束还是出错。**

- 正常结束：输出 A 然后 B。
- `try` 里出错：剩余代码不执行，`finally` 仍执行，**执行完再把错误抛出去**。
- `try` 里有 `return`（本例）：函数不会立刻离开，先执行 `finally`，再把返回值交出去。

### Q8. 为什么非要 `try/finally`？先写 finally 的内容再写 try 的内容不行吗？

| 写法 | 正常运行 | 原函数出错 |
| --- | --- | --- |
| 先计时再运行 | 耗时永远约 0，错误 | 同左 |
| 先运行、存结果、再计时 | 正确 | **不打印耗时** |
| try/finally | 正确 | **照样打印耗时**，再抛出错误 |

- 先计时后运行：两次 `perf_counter()` 之间什么都没做。
- `return` 后面的 `print` 永远不执行，所以要么存 `result` 再打印，要么用 `finally`。
- 出错时也打印耗时很有用：网络请求等 30 秒才超时，知道这 30 秒才能定位是"卡住了"而不是"一开始就失败"。

### Q9. `perf_counter() - start` 两边不是同一个东西吗？

不是。`start` 存的是第一次调用的**结果（一个数字）**，`perf_counter()` 每次调用都"看一眼现在的时间"返回**新的数字**。开始做题看一眼钟写在纸上，做完再看一眼钟，相减即用时。如果写成 `start = perf_counter`（无括号），那才是"同一个东西"——存的是函数，没看时间。

### Q10. `@timed` 可以直接用吗，还是要先 `def timed`？

必须先定义（或导入），否则 `NameError: name 'timed' is not defined`，且定义要写在使用之前。实际项目常把装饰器放在 `utils.py`，用 `from utils import timed` 导入。

- 内置、无需导入：`@property`、`@staticmethod`、`@classmethod`
- 需要导入：`@functools.lru_cache`、`@dataclasses.dataclass`、`@contextlib.contextmanager`

### Q11. 常见的 `@property` 是什么作用？

**让方法像属性一样访问，不加括号。**

```python
class Circle:
    def __init__(self, radius):
        self.radius = radius
    @property
    def area(self):
        return 3.14159 * self.radius ** 2

c = Circle(2)
c.area          # 12.566，每次访问都在运行 area 方法
```

三个用途：

1. **值是算出来的，始终正确**：改 `c.radius = 3` 后 `c.area` 自动变 28.274；若在 `__init__` 写死 `self.area` 就不会更新。
2. **只读属性**：`c.area = 100` 报 `AttributeError: ... has no setter`。
3. **修改时做检查**（配合 `.setter`）：

```python
    @property
    def radius(self):
        return self._radius
    @radius.setter
    def radius(self, value):
        if value <= 0:
            raise ValueError("半径必须大于 0")
        self._radius = value
```

PyTorch 张量里 `x.shape`、`x.device`、`x.T` 不加括号，就是类似机制；`x.size()`、`x.sum()` 是普通方法。`@property` 也是装饰器，等价于 `area = property(area)`。**适合**访问很快、像"读取一个特征"的东西；**不适合**耗时或有副作用的操作（读文件、训练），那些应写成 `obj.train()` 让调用者知道"在执行操作"。

### Q12. `self._radius` 的下划线不加也可以吧？

分两层：

- **下划线本身不是必须的**，只是惯例（"内部用，别直接碰"）。Python 没有真正的私有，`c._radius = -1` 照样能改，还绕过了 setter 检查。写成 `self.r` 也能工作。
- **但存储名不能和 property 同名**。若写 `self.radius`，getter 里 `return self.radius` 会再次调用 getter，setter 里 `self.radius = value` 会再次调用 setter，报 `RecursionError: maximum recursion depth exceeded`——定义 property 后，任何对 `self.radius` 的读写都会转去调用你写的方法。

大家用 `_radius` 是因为一眼能看出它和 `radius` 是一对，并提醒别人通过 `radius` 来改。双下划线 `self.__radius` 会触发自动改名，一般单下划线就够。

## 2.7 上下文管理器：让资源必定释放

> **📖 原文 [2.7 上下文管理器：让资源必定释放](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#27-上下文管理器让资源必定释放)**
>
> `with` 把资源的获取和释放放在同一个词法范围内，即使发生异常，也会执行清理逻辑：
>
> ```python
> from contextlib import contextmanager
> from collections.abc import Iterator
> from time import perf_counter
>
> @contextmanager
> def timer(name: str) -> Iterator[None]:
>     start = perf_counter()
>     try:
>         yield
>     finally:
>         elapsed_ms = (perf_counter() - start) * 1_000
>         print(f"{name}: {elapsed_ms:.3f} ms")
>
> with timer("load data"):
>     data = [value for value in range(100_000)]
> ```
>
> 文件、锁、数据库连接、临时目录、自动混合精度上下文都适合这种模式。它和 C++ 的 RAII 解决的是同一个问题：**资源生命周期必须有清晰边界**。

### Q13. `@timed` 和 `with timer()` 的使用区别

```python
@contextmanager
def timer(name="block"):
    start = perf_counter()
    try:
        yield                    # with 里的代码在这里运行
    finally:
        print(f"{name}: {(perf_counter() - start) * 1_000:.3f} ms")
```

结构和 `timed` 几乎一样，区别只在中间运行什么：`timed` 运行 `fn(...)`，`timer` 运行 `yield` 处即 `with` 块里的代码。

|  | `@timed` | `with timer()` |
| --- | --- | --- |
| 计时对象 | 整个函数 | 任意一段代码 |
| 写在哪里 | 函数定义上方 | 要计时的代码周围 |
| 生效范围 | 该函数的每一次调用 | 只有这一处 |
| 名字 | 自动用 `fn.__name__` | 自己起，可写"第 3 个 epoch 的数据加载" |
| 适合场景 | 很多函数长期计时 | 临时测某几行，或测别人库里的函数 |

一个函数里分段测只能用 `with`：

```python
def train_step(batch):
    with timer("前向"):
        out = model(batch)
    with timer("反向"):
        loss = loss_fn(out)
        loss.backward()
```

（注意：若在 GPU 上这样计时，还要考虑[原文 3.6](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#36-正确做基准测试) 说的 Kernel 异步执行，CPU 计时器只量到"提交任务"的时间，需在边界同步。）

### Q14. `def timer(name: str) -> Iterator[None]:` 怎么理解？

- `name: str`：参数是字符串。
- `-> Iterator[None]`：函数里**有 `yield`，它就是生成器函数**，调用时不直接运行代码，而是返回一个生成器（属于迭代器），所以返回类型是 `Iterator`。方括号里是**每次 `yield` 交出的值的类型**：光秃秃的 `yield` 等于 `yield None`，所以是 `[None]`；`yield 1` 就是 `Iterator[int]`。
- 若 `yield` 交出了值，就能用 `as` 接住：`with opened("a.txt") as msg:`。
- 另一种写法 `Generator[None, None, None]`，三个位置分别是 yield 类型、send 类型、return 类型，效果一样。

### Q15. `with timer("load data"): data = [... range(100_000)]` 会打印很多次吗？

只打印**一次**。列表推导式内部循环 10 万次，但对 `with` 来说只是一条语句，打印的是总耗时。**`with` 这一行被执行几次，就打印几次**：`with` 放在 `for` 里每轮打印；`for` 放在 `with` 里只打印一次合计。

`100_000` 就是 `100000`（Python 3.6+），下划线只为好读；不能放开头结尾、不能连写两个，小数也可用 `3.141_592`。

### Q16. `timer` 里的 `yield` 怎么理解，为什么不直接 `return`？

**`return` 是"结束"，`yield` 是"暂停，等会儿还要回来"**（书签比喻）。计时要做三件事：① 记开始时间（timer 的代码）② 运行要计时的代码（你写在 with 里的）③ 打印耗时（timer 的代码）。timer 必须在 ① 和 ③ 之间**把控制权让出去**，这正是 `yield` 的能力。换成 `return` 函数在 ① 之后就彻底结束，且 `@contextmanager` 要求必须有 `yield`。

|  | 要计时的东西 | 怎么在中间运行它 |
| --- | --- | --- |
| `timed` | 一个函数 `fn` | 直接调用 `fn(...)` |
| `timer` | `with` 里的一段代码 | `yield` 暂停，把控制权让出去 |

### Q17. 可以这样理解 `with` 吗：先运行 with 后的函数，再运行 with 下的代码，结束后再返回开头？

大方向对，两处更准确：第一步是**运行 timer 到 `yield` 为止**；最后不是"返回开头"，而是**回到 `yield` 暂停处运行后半部分**，开头不会再执行。

```text
timer 前半 → with 里的代码 → timer 后半 → 后面的代码
   ①              ②              ③           ④
```

**`yield` 把 `timer` 切成"进入时做的事"和"离开时做的事"两半，`with` 里的代码夹在中间。** 即使 `with` 里出错，③ 也会执行（因为 `try/finally`）。

### Q18. 进入/离开不是因为 `yield` 导致的吗？`with` 是不是二次执行？`@contextmanager` 的作用？

**方向反了**：是 `with` 先规定"进入调用 `__enter__`、离开调用 `__exit__`"（[第二篇](/posts/ai-infra/ch1-02-scope-class-generator/) 的协议），`with` 根本不知道 `yield` 的存在。用类写 Timer 完全不需要 `yield`：

```python
class Timer:
    def __init__(self, name):
        self.name = name
    def __enter__(self):
        self.start = perf_counter()
    def __exit__(self, exc_type, exc, tb):
        print(f"{self.name}: {(perf_counter() - self.start) * 1_000:.3f} ms")
```

`with` 做的事相当于：`t.__enter__()` → `try: 块内代码` → `finally: t.__exit__(...)`。可以说调用了两次方法，但**不是把同一个函数运行两遍**：`@contextmanager` 版本的 timer 从头到尾只运行一遍，`start = perf_counter()` 不会运行第二次。

**`@contextmanager` 的作用**：把带 `yield` 的函数转换成拥有 `__enter__`/`__exit__` 的对象。不加它，`with timer(...)` 报 `TypeError: 'generator' object does not support the context manager protocol`。它大致自动生成：

```python
class 自动生成的类:
    def __init__(self, gen):
        self.gen = gen
    def __enter__(self):
        return next(self.gen)        # 运行到 yield；yield 的值给 as
    def __exit__(self, exc_type, exc, tb):
        try:
            next(self.gen)           # 从 yield 继续，运行后半部分
        except StopIteration:
            pass                     # 生成器正常结束
```

（真实实现还会把 with 里的错误"扔回" yield 处，所以 timer 的 `finally` 能捕捉到。）

|  | 用类 | 用 `@contextmanager` |
| --- | --- | --- |
| 要写的东西 | `__init__`、`__enter__`、`__exit__` | 一个函数 + 一个 `yield` |
| 数据传递 | 存在 `self.start` | 普通局部变量 `start` |
| 可读性 | 前后逻辑分在两个方法 | 从上往下读 |

`open(...)`、`torch.no_grad()` 是用类实现的，执行顺序完全相同。原文说它"和 C++ 的 RAII 解决的是同一个问题：资源生命周期必须有清晰边界"（见 [第五篇](/posts/ai-infra/ch1-05-cpp-memory/)）。

## 2.8 异常处理：只捕获能够处理的错误

> **📖 原文 [2.8 异常处理：只捕获能够处理的错误](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#28-异常处理只捕获能够处理的错误)**
>
> ```python
> def parse_world_size(raw: str) -> int:
>     try:
>         world_size = int(raw)
>     except ValueError as exc:
>         raise ValueError(f"WORLD_SIZE 必须是整数，实际为 {raw!r}") from exc
>
>     if world_size <= 0:
>         raise ValueError("WORLD_SIZE 必须大于 0")
>     return world_size
> ```
>
> 建议遵循三条原则：
>
> 1. 捕获具体异常，不要用空的 `except:` 吞掉 `KeyboardInterrupt` 等信号；
> 2. 只有在能恢复、补充上下文或转换抽象层时才捕获；
> 3. 用 `raise ... from exc` 保留根因链。

### Q19. `try`、`except`、`pass` 的语法

**`try/except`**：尝试运行，出了**类型匹配**的错误就执行 `except`，程序不崩溃。没出错时 `except` 完全跳过；出错后 `try` 里剩下的代码不执行。类型不匹配（如 `int("abc")` 抛 ValueError 但只写了 `except ZeroDivisionError`）不会被捕获。可以写多个 `except` 分别处理，`except ValueError as e` 拿到错误对象查看信息。

|  | 什么时候执行 |
| --- | --- |
| `except` | 出错且类型匹配时 |
| `finally` | 不管出没出错，一定执行 |

**`pass`**："这里什么都不做"。Python 规定冒号后必须有缩进代码块，空着会 `IndentationError: expected an indented block`（Python 靠缩进判断代码块）。`pass` 就像必填栏里写的"无"。场景：先占位 `def train_model(): pass`；空类 `class MyError(Exception): pass`；捕获后故意不处理 `except StopIteration: pass`。

**为什么 `__exit__` 里要捕获 `StopIteration`？** timer 只有一个 `yield`：第一次 `next` 运行到 yield；第二次 `next` 运行完后半部分，后面再无 yield，函数结束，`next` 抛 `StopIteration`（Python 用抛错的方式通知"没有下一段了"）。这正是期望结果，所以接住并忽略。

**提醒**：不要写空的 `except: pass`，它会吞掉所有错误，bug 无从发现。

### Q20. 解释 `parse_world_size` 和三条原则

**背景**：`WORLD_SIZE` 是 PyTorch 分布式训练的环境变量，表示参与训练的总进程数（2 台机器 × 4 卡 = 8）。环境变量读出来**一定是字符串**。

```python
def parse_world_size(raw: str) -> int:
    try:
        world_size = int(raw)                 # "abc"、"" 会抛 ValueError
    except ValueError as exc:                 # 只捕获 ValueError，存到 exc
        raise ValueError(f"WORLD_SIZE 必须是整数，实际为 {raw!r}") from exc
    if world_size <= 0:                       # "0"、"-3" 能转成功，但不合理
        raise ValueError("WORLD_SIZE 必须大于 0")
    return world_size
```

- `{raw!r}` 用 `repr()` 格式显示，带引号：空字符串显示为 `''`，`"8 "` 显示为 `'8 '`，一眼看出多了空格。
- 为什么捕获再抛？对比报错：直接报 `invalid literal for int() with base 10: 'abc'` 不知道是哪里；处理后直接告诉你"是 WORLD\_SIZE 配错了"。
- 第 7–8 行用 `if` 不用 `try`：这里没有"可能失败的操作"，只是检查值。

**三条原则：**

1. **捕获具体异常**：空 `except:` 会捕获 `KeyboardInterrupt`（Ctrl+C）和 `SystemExit`（`sys.exit()`），按 Ctrl+C 程序都停不下来；变量名拼错的 `NameError` 也会被吞掉。
2. **只在有意义时捕获**——能恢复（配置文件不存在就用默认配置）、补充上下文（本例）、转换抽象层（把底层 `sqlite3.OperationalError`、`ConnectionError` 统一成你的 `UserDataError`）。都不符合就别捕获，否则程序带着错误状态继续运行，问题会在更远处以更奇怪的方式暴露。
3. **`raise ... from exc` 保留根因链**：报错时显示两个错误，中间写 `The above exception was the direct cause of the following exception:`，既有清楚说明又有原始位置。

**最后一句"不要把异常用于高频控制流"**：抛出和捕获比普通 `if` 慢得多，且读者会误以为"这里可能出意外"。

```python
# 不推荐：每个新词触发一次 KeyError
try:
    counts[word] += 1
except KeyError:
    counts[word] = 1
# 推荐
counts[word] = counts.get(word, 0) + 1
```

判断：正常运行中经常出现的情况用 `if` / `.get()`；不应发生、偶尔出现的问题才用异常。

### Q21. `raise` 的语法是什么？

**主动抛出错误**：当前执行立刻停止，错误沿调用链一层层往上传，直到被某个 `except` 接住；到最外层都没人接，程序报错退出并打印 traceback。

| 形式 | 用途 |
| --- | --- |
| `raise ValueError("信息")` | 抛出新错误（最常用） |
| `raise`（单独，只能在 except 里） | 原样再抛，如打完日志仍让上层知道 |
| `raise 新错误 from exc` | 抛新错误并保留原始错误为根因 |
| `raise 新错误 from None` | 抛新错误并隐藏原始错误（原始错误只会干扰时） |

| 常用错误类型 | 什么时候用 |
| --- | --- |
| `ValueError` | 类型对值不对，如学习率为负 |
| `TypeError` | 类型不对 |
| `KeyError` | 字典找不到键 |
| `FileNotFoundError` | 文件不存在 |
| `RuntimeError` | 其他运行时问题，不好归类时 |
| `NotImplementedError` | 功能未实现，常用于"子类必须重写" |

自定义：`class ConfigError(Exception): pass`，调用者可专门 `except ConfigError:`。与 `return` 对比：都让函数立刻结束；`return` 是正常完成交出结果，`raise` 是无法完成交出错误。

### Q22. 为什么这里要 `from exc`，而 `if` 后面的 `raise` 没有？

`from exc` 的作用是**把新错误和一个已经发生的旧错误关联起来**，只有在 `except` 里确实有旧错误时才需要。`if world_size <= 0` 后的 `raise` 是第一个错误，没有 `exc` 这个变量，无从 `from`。（`print` 只是显示文字，不抛错，更不会有 from。）

在 `except` 里不写 `from`，Python 也会自动关联，但显示 `During handling of the above exception, another exception occurred:`——听起来像你的错误处理代码自己出了 bug。写了 `from exc` 才准确表达"故意把旧错误转换成更清楚的新错误"。

| `raise` 的位置 | 有没有旧错误 | 要不要 `from` |
| --- | --- | --- |
| 普通代码里，如 `if` 判断后 | 没有 | 不需要也没法写 |
| `except` 里转换旧错误 | 有 | 建议写 `from exc` |

## 2.9 类型标注与数据类

> **📖 原文 [2.9 类型标注与数据类](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#29-类型标注与数据类)**
>
> 类型标注主要服务于静态检查、IDE 和读者，不会自动在运行时验证类型：
>
> ```python
> from dataclasses import dataclass
> from collections.abc import Sequence
>
> @dataclass(frozen=True)
> class ShardSpec:
>     rank: int
>     world_size: int
>
> def shard(values: Sequence[int], spec: ShardSpec) -> Sequence[int]:
>     return values[spec.rank::spec.world_size]
> ```
>
> 常用选择：

### Q23. `@dataclass(frozen=True)` 怎么理解？和 staticmethod、classmethod 有什么区别？

`@dataclass` 让你只写字段名和类型（`lr: float`，**类型注解是必须的**，dataclass 靠它识别字段），自动生成 `__init__`、`__repr__`、`__eq__`，字段可以有默认值 `epochs: int = 10`。适合配置、一条样本、一次实验结果这类"主要装数据"的类。

`frozen=True`：对象创建后所有字段**只读**，`c.lr = 0.1` 报 `FrozenInstanceError`。好处：

- 防止配置在程序各处传递时被某个函数偷偷改掉。
- 可作 dict 的键、放进 set：`results = {Config(0.01, 32): 0.85}`（呼应 [第一篇](/posts/ai-infra/ch1-01-python-objects/) 的可哈希规则）。
- 想要新参数就 `from dataclasses import replace; c2 = replace(c, lr=0.1)`，原对象不变。

`@dataclass(frozen=True)` 带括号：`dataclass(frozen=True)` 先返回一个"配置好的装饰器"，再装饰类，和 `@wraps(fn)` 同一种写法。

**区分**：`@dataclass` 装饰**整个类**，`@property`/`@staticmethod`/`@classmethod` 装饰**类里的方法**。

| 装饰器 | 写在哪里 | 作用 | 需要导入吗 |
| --- | --- | --- | --- |
| `@dataclass` | `class` 上面 | 自动生成 `__init__`、打印、比较 | 需要 |
| `@dataclass(frozen=True)` | `class` 上面 | 同上，且创建后不可修改 | 需要 |
| `@property` | 方法上面 | 方法像属性一样访问 | 不用 |
| `@staticmethod` | 方法上面 | 不接收 `self`/`cls` 的普通函数 | 不用 |
| `@classmethod` | 方法上面 | 第一个参数是类，常用作备选构造器（如 `from_diameter`、`from_pretrained`） | 不用 |

## 系列目录

- [导读与易错点速查](/posts/ai-infra/ch1-00-guide/)
- [（一）最小工具箱与 Python 对象模型](/posts/ai-infra/ch1-01-python-objects/)
- [（二）作用域、类与生成器](/posts/ai-infra/ch1-02-scope-class-generator/)
- **（三）装饰器、上下文管理器与异常处理**（本篇）
- [（四）Python 并发与性能分析](/posts/ai-infra/ch1-04-concurrency-profiling/)
- [（五）C/C++ 内存、生命周期与资源管理](/posts/ai-infra/ch1-05-cpp-memory/)
- [（六）C++ 编译、链接、模板与 CMake](/posts/ai-infra/ch1-06-cpp-build/)
- [（七）Linux 开发基本功与环境隔离](/posts/ai-infra/ch1-07-linux-env/)

