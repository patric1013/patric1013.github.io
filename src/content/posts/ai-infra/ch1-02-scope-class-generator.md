---
title: "AI Infra 第1章笔记（二）：作用域、类与生成器"
published: 2026-11-03T10:00:00
description: "三小节串起来是一条线：名字怎么查找（LEGB）→ 函数能带走外层名字（闭包）→ 对象把数据存在自己身上（self）→ 语法背后调用约定好的方法（协议）→ __iter__/yield 让对象按需产出数据（生成器）。"
tags: ["AI Infra", "Python"]
category: "AI Infra 学习笔记"
draft: false
---

三小节串起来是一条线：**名字怎么查找（LEGB）→ 函数能带走外层名字（闭包）→ 对象把数据存在自己身上（self）→ 语法背后调用约定好的方法（协议）→ `__iter__`/`yield` 让对象按需产出数据（生成器）。**

> [!NOTE]
> 本篇对应原文 [AIInfraGuide · 第1章：编程语言基础](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/) 的 §2.3–2.5（作者 [caomaolufei](https://github.com/caomaolufei/AIInfraGuide)，MIT 许可）。每节开头的引用块摘自原文，下面的问答是我学习时提出的问题和整理后的解答；文中的“原文 x.y”都可以点击跳转到原文对应小节。

## 2.3 作用域、闭包与 LEGB

> **📖 原文 [2.3 作用域、闭包与 LEGB](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#23-作用域闭包与-legb)**
>
> Python 按 **LEGB** 顺序查找名字：Local（局部）→ Enclosing（外层函数）→ Global（模块）→ Builtins（内置）。

### Q1. 怎么理解 LEGB、闭包和延迟绑定？

**LEGB** 是 Python 查找名字的顺序，找到就停：

1. **L**ocal：当前函数内部
2. **E**nclosing：包住它的外层函数
3. **G**lobal：模块顶层
4. **B**uiltins：内置名字，如 `print`、`len`

**闭包**：`make_multiplier` 里的 `multiply` 自己没定义 `scale`，按 LEGB 在外层（E）找到。`make_multiplier(2.0)` 返回之后，`double` 仍然记得 `scale=2.0`——"函数 + 它引用的外层变量"打包在一起就叫闭包。可以理解成**函数工厂**：传 2.0 得到"乘 2 函数"，传 3.0 得到"乘 3 函数"。

**延迟绑定**：闭包记住的是**变量**，不是当时的值，调用时才去读。

一句话：**闭包保存的是对外层变量的引用，值要等调用时才读取。**

### Q2. `make_multiplier` 返回的是函数吗？`x` 没有定义呀？

对，返回的是 `multiply` 本身（注意没加括号调用）。`x` 是 `multiply` 的**参数**，等调用返回的函数时才传入。两个值分两次给：

```python
double = make_multiplier(2.0)  # 只确定 scale=2.0，x 还不存在
double(3.0)                     # 这一步才把 3.0 传给 x → 3.0 * 2.0 = 6.0
```

可以把 `double` 理解为 `def double(x): return x * 2.0`，只不过是工厂"生产"出来的。

### Q3. `Callable` 是什么？不写会报错吗？

`Callable` 是类型注解，意思是"能在后面加括号执行的东西"。`Callable[[float], float]` 拆开读：`[float]` 是参数列表，后面的 `float` 是返回值。所以 `def make_multiplier(scale: float) -> Callable[[float], float]` 表示"接收 float，返回一个 float→float 的函数"。

- 函数、lambda、类（调用类即创建实例）、定义了 `__call__` 的对象都算 Callable，可用内置 `callable(obj)` 判断。
- **不写完全不会报错**：注解是可选的，运行时基本被忽略（对应[原文 2.9](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#29-类型标注与数据类)"类型标注不会自动在运行时验证类型"）。
- 反而要注意：**写了 `Callable` 却没 `from collections.abc import Callable`**，多数版本定义函数时会 `NameError`，因为注解在定义时求值。Python 3.14 起注解改为延迟求值，不会立刻报错。

### Q4. `nonlocal` / `global` 这段是什么意思？

只读外层变量没问题，**一赋值就出错**：

```python
def counter():
    count = 0
    def add():
        count += 1   # UnboundLocalError
        return count
    return add
```

只要函数里给某个名字赋值，Python 就把它当作**局部变量**；`count += 1` 要先读局部 `count`，可它还没赋值。解决：在 `add` 里加 `nonlocal count`。

- `nonlocal`：修改**外层函数**的变量（E 层）
- `global`：修改**模块顶层**的变量（G 层）

原文最后一句的意思：尽量用参数传入、`return` 传出，函数行为只取决于输入，更容易看懂和测试；`global` 尤其少用，任何地方都可能悄悄改掉它。

### Q5. 逐行解释 `bad` / `good` 两行代码

`bad = [lambda: i for i in range(3)]` 等价于：

```python
bad = []
for i in range(3):
    bad.append(lambda: i)   # 存的是函数，没有调用
```

`good = [lambda i=i: i for i in range(3)]` 的关键是 `i=i`：左边的 `i` 是 lambda 自己的参数名，右边的 `i` 是循环当前值。默认参数在**创建函数时**求值，所以相当于依次创建了 `lambda i=0: i`、`lambda i=1: i`、`lambda i=2: i`。两个 `i` 同名容易晕，换个名字就清楚：`lambda v=i: v`。

### Q6. `bad.append(lambda: i)` 这一行不理解；lambda 和 int 有什么区别？`fn` 是什么？

`lambda: i` 冒号左边是空的，表示无参数，等价于 `def f(): return i`。`bad.append(...)` 放进去的是**函数本身**，不是执行结果：

```python
bad.append(i)           # 放进去的是数字，比如 0
bad.append(lambda: i)   # 放进去的是函数，调用时才返回 i
```

**纸条比喻**：数字是"答案本身"；函数是一张写着"去看看 i 现在是几"的纸条，要照着做（调用）才得到结果。

```python
print(a[0])      # 0
print(bad[0])    # <function <lambda> at 0x...>
print(bad[0]())  # 2，加括号调用后才得到数字
```

`fn` 只是普通的循环变量名，`[fn() for fn in good]` 等价于 `for fn in good: result.append(fn())`——依次取出每个函数并调用。

### Q7. 为什么 `good` 返回的不是 2？

- bad 的纸条写的是"去看看 **i** 现在是几"，去看时 `i` 已是 2。
- good 的纸条在写的那一刻就**把当时的数字抄下来了**（0、1、2）。

用 LEGB 解释更准确：bad 的函数里没有自己的 `i`，只能等调用时去外层找，找到循环结束后的 2；good 的函数自带参数 `i`，按 LEGB **先在 Local 找到它**，根本不会去读外面的 `i`。

### Q8. LEGB 的规则会在找到之后"消失"吗？

不会。LEGB 是**每次使用名字都重新执行**的查找规则：

```python
x = 1
def f(): return x
print(f())   # 1
x = 100
print(f())   # 100，重新查找
```

- 找到就停：Local 找到 `x` 后，全局的 `x` 不被读取也不被修改，只是被"挡住"。
- 变量的生命周期是另一回事：函数结束 Local 变量通常消失，但若被闭包引用就会被保留（`make_multiplier` 能记住 `scale` 的原因）。

### Q9. 什么是作用域？

**作用域（scope）是一个名字能被访问到的范围。** 比喻：部门办公室里的东西（局部变量）只有本部门能用；大厅里的东西（全局变量）所有办公室都看得到；下班关门（函数结束）办公室就被清理。

```python
x = 10
def f():
    y = 20
    print(x, y)  # 10 20
f()
print(y)         # NameError
```

**易踩的坑**：Python 里只有函数（以及类和模块）创建新作用域，`if`、`for`、`while` 不会：

```python
for i in range(3):
    z = i * 2
print(i, z)   # 2 4，循环结束后仍能访问
```

这和 C++/Java 不同，也正是 `bad` 例子里循环结束后 `i` 仍停在 2 的原因。

### Q10. 如果给 `print` 赋值，内置的 `print` 会失效？

会，但只是被"挡住"：`print = 5` 后，LEGB 在 Global 层就找到 5 并停止，`print("hi")` 报 `TypeError: 'int' object is not callable`。找回：`del print`，或 `import builtins; builtins.print("hi")`。遮挡只影响定义它的那个作用域。

实际最常见的坑是把内置名用作变量名：`list`、`sum`、`max`、`id`、`input`（深度学习代码里尤其常见）。改成 `total`、`max_val`、`inputs` 等。

## 2.4 类、数据模型与协议

> **📖 原文 [2.4 类、数据模型与协议](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#24-类数据模型与协议)**
>
> Python 的特殊方法让对象接入语言协议。例如，实现 `__len__` 后可调用 `len(obj)`，实现 `__iter__` 后可被 `for` 遍历。

### Q11. `@dataclass`、`@classmethod`、`@staticmethod` 分别有什么作用？

三者都是装饰器（[第三篇](/posts/ai-infra/ch1-03-decorator-context-exception/) 详述装饰器本身）。

- `@dataclass`：只声明字段，自动生成 `__init__`、`__repr__`、`__eq__`。
- `@classmethod`：第一个参数是 `cls`（类本身），最常用于**备选构造器**。
- `@staticmethod`：不接收 `self` 也不接收 `cls`，就是放在类里的普通函数。

| 方法里需要用到什么 | 用哪种 |
| --- | --- |
| 实例的数据（`self.x`） | 普通方法 |
| 类本身，比如要创建对象 | `@classmethod` |
| 两者都不需要 | `@staticmethod` |

### Q12. dataclass 的 `Point` 例子没看懂

对比不用 dataclass 的普通类：

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

p = Point(1, 2)
print(p)                 # <__main__.Point object at 0x10a3f...>
print(p == Point(1, 2))  # False
```

两个问题：打印出来是内存地址；x、y 完全一样 `==` 却是 False。用了 `@dataclass` 后，`x: float`、`y: float` 声明了两个字段，自动生成：

1. `__init__`：能直接 `Point(1, 2)`
2. `__repr__`：`print(p)` 显示 `Point(x=1, y=2)`
3. `__eq__`：`==` 逐字段比较值

### Q13. "普通类默认比较的是是不是同一个对象"，这个不理解

**两个一模一样的杯子**：问"长得一样吗"——一样；问"是同一个杯子吗"——不是。普通类的 `==` 默认问的是第二个问题。

```python
p = Point(1, 2)   # 造第一个杯子，贴标签 p
q = Point(1, 2)   # 又造第二个，贴标签 q
p == q            # False

r = p             # 没造新杯子，只多贴一个标签
p == r            # True
print(id(p), id(q), id(r))   # p 与 r 相同，q 不同
```

`@dataclass` 把 `==` 改成问"长得一样吗"。想问"是不是同一个"用 `is`。这就是[原文 2.1](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#21-名字对象与引用) 的 `is`（身份）vs `==`（值）。

### Q14. 如果 p、q 都赋值为 `1`，或都赋值为 `[1]`，`p == q` 吗？

两种情况 `p == q` 都是 **True**：int、list、str 等内置类型本来就定义了按内容比较的 `==`。只有自己写、又没定义 `__eq__` 的类才退回"是不是同一个对象"。

换成 `is`：`[1] is [1]` 为 False（写两次就造了两个列表）；`1 is 1` 为 True，因为 CPython 让小整数（-5 到 256）共用同一对象，这是实现细节。规则：**比较值永远用 `==`，`is` 只用于 `x is None` 这类情况。**

### Q15. "特殊方法让对象接入语言协议"怎么理解？

**Python 的内置语法，背后都在调用对象的特殊方法。** `len(obj)` 实际执行 `obj.__len__()`，`for x in obj` 调用 `obj.__iter__()`。"协议"就是约定：实现了约定名字的方法，就能用对应语法。删掉 `__len__`，`len(p)` 就报 `TypeError: object of type 'Playlist' has no len()`。

| 你写的语法 | Python 调用的方法 |
| --- | --- |
| `len(obj)` | `__len__` |
| `for x in obj` | `__iter__` |
| `obj[i]` | `__getitem__` |
| `a == b` | `__eq__` |
| `print(obj)` | `__repr__` / `__str__` |
| `a + b` | `__add__` |
| `obj()` | `__call__` |

PyTorch `Dataset` 要求实现 `__len__` 和 `__getitem__`，就是在用这套协议：DataLoader 通过 `len(dataset)` 拿样本数，再用 `dataset[i]` 取第 i 个样本。

### Q16. 声明 dataclass 就能直接用这些特殊方法吗？`__iter__` 怎么用？

**不能。** `@dataclass` 默认只生成 `__init__`、`__repr__`、`__eq__`（`order=True` 再加 `<`、`>` 等）。`__len__`、`__iter__` 要自己写，因为"长度""遍历"指什么只有你知道。

`for` 循环在背后做的事：

```python
it = iter(["a", "b"])   # 1. 先拿到迭代器
while True:
    try:
        s = next(it)    # 2. 反复要下一个
        print(s)
    except StopIteration:
        break           # 3. 没有了就停
```

**发牌员比喻**：迭代器每次 `next()` 发一张牌，发完报告 StopIteration。`__iter__` 的任务就是**返回一个发牌员**；`return iter(self.songs)` 等于"把内部列表的发牌员交出去"。想自定义发牌方式就用 `yield`，它会自动造出一个发牌员：`return` 返回后函数结束，`yield` 返回一个值后只是**暂停**。

### Q17. 表格里其他几个特殊方法（`__call__`、`__enter__/__exit__`、`__next__`、`__getitem__`）

- **`__call__`**：让对象像函数一样调用，`double(3)` 实际执行 `double.__call__(3)`，效果与闭包 `make_multiplier` 一样。PyTorch 的 `model(x)` 能运行，是因为 `nn.Module` 实现了 `__call__`，内部调用 `forward` 并处理 hook——所以推荐 `model(x)` 而不是 `model.forward(x)`。
- **`__enter__` / `__exit__`**：配合 `with`，进入时调用 `__enter__`（返回值赋给 `as` 后的变量），离开时调用 `__exit__` 做收尾，即使中途报错也执行。`torch.no_grad()` 就是这样：进入时关梯度，离开时恢复（[第三篇](/posts/ai-infra/ch1-03-decorator-context-exception/) 详述）。
- **`__next__`**：发牌员的"发下一张"，发完 `raise StopIteration`。`Countdown` 例子中 `__iter__` 返回 `self`，自己就是发牌员。实际开发中用 `yield` 写 `__iter__` 更简单，很少手写 `__next__`。
- **`__getitem__`**：支持方括号取值。`key` 不一定是整数：`ds[1:3]` 收到 slice 对象，`obj["name"]` 收到字符串，如何处理由你决定。

补充：`def __call__(self, x): return x * self.scale` 没写返回类型只是为了简短；完整写法是 `def __call__(self, x: float) -> float`，`__init__` 标 `-> None`。

### Q18. `__getitem__` 的用途是什么？

让自己的对象也能像列表一样用 `[]` 取东西，**而且取值时可以执行任意操作，不只是查找**。

```python
class MyDataset:
    def __init__(self, paths):
        self.paths = paths              # 只保存路径，几乎不占内存

    def __getitem__(self, idx):
        img = load_image(self.paths[idx])  # 用到第几张才读第几张
        img = augment(img)                  # 顺便做增强
        return img
```

100 万张图片若在 `__init__` 一次读完，内存就爆了。自监督学习每次取样本随机生成两个增强视图，正是写在 `__getitem__` 里。为什么不写个 `get(idx)`？因为 DataLoader 内部写的是 `dataset[i]`，只认这套约定——遵守它，DataLoader 才能帮你打乱、组批、多进程加载。

### Q19. `img = load_image(...)`，img 绑定的是图片读进内存的地址吗？

差不多，更准确地说：**img 是一个标签，贴在内存里的一个图片对象上**（PIL Image、numpy 数组或 tensor）。它不"是"图片本身，也不是文件路径，而是指向内存对象的引用；CPython 中 `id(img)` 基本就是该对象的内存地址。

```python
img = load_image("a.jpg")   # 创建对象 A，贴标签 img
img = augment(img)          # augment 生成新对象 B，标签移到 B
```

若没有其他标签贴在 A 上，A 会被自动回收——这就是按需读取不会越积越多的原因。

### Q20. augment 之后内存里的张量变了，但图片本身没变？

对，**硬盘上的文件不会变**，除非显式 `save`。所以每个 epoch 都能对同一张图做不同的随机增强。内存里的对象怎么变分两种：

- **生成新对象**：原对象 A 不变，另造 B。torchvision 大多数变换（裁剪、翻转）是这样。
- **原地修改**：直接改 A。PyTorch 中以下划线结尾的方法，如 `x.add_(1)`、`x.clamp_(0, 1)`。

自监督从同一张图生成两个视图时，若 augment 原地修改 img，生成 v2 时用的就是被改过的图了，应先 `img.clone()`。

### Q21. 如果想像 list 一样直接读取？

若指"先全部读进内存"：把读取移到 `__init__`（`self.images = [load_image(p) for p in paths]`），`__getitem__` 只 `return self.images[idx]`，负索引和切片也自动支持。

|  | 按需读取 | 预先全部读入 |
| --- | --- | --- |
| 内存占用 | 很小，只存路径 | 等于整个数据集 |
| 取值速度 | 每次读硬盘 | 直接从内存取 |
| 适合场景 | 大数据集，如 ImageNet | 小数据集，如 CIFAR、MNIST |

注意：增强仍应放在 `__getitem__` 里，否则每个 epoch 都是同一个增强结果；预先读入时列表里是唯一一份原图，原地增强会把它永久改掉。

若只是想让 `obj[0]` 能用：

```python
class MyList:
    def __init__(self, data):
        self.data = data
    def __getitem__(self, idx):
        return self.data[idx]

class Squares:                 # 不存数据，现场计算
    def __getitem__(self, idx):
        return idx * idx
```

**`obj[x]` 只是 `obj.__getitem__(x)` 的简便写法**，方括号里写什么就传什么，返回什么由你决定。

### Q22. 为什么返回 `self.data[idx]` 而不是 `data[idx]`？`self.data` 会在函数外消失吗？为什么它存在对象里？

- `data` 是 `__init__` 的参数，是**局部变量**，`__init__` 结束就消失。在 `__getitem__` 里写 `data[idx]` 会 `NameError`。
- `self.data` 存在**对象**身上，对象存在多久它就存在多久，与哪个函数结束无关。对象没有任何标签引用时（如 `m = None`）才被回收，`self.data` 随之释放。

为什么在对象里？因为 **`self` 就是那个对象本身**。`m = MyList([10, 20, 30])` 背后：先建空对象 → 调用 `MyList.__init__(新对象, [10, 20, 30])`，新对象作为 `self` 传入 → 结束后贴上标签 `m`。所以 `self` 和 `m` 是同一个对象。每个对象有个属性字典：

```python
print(m.__dict__)   # {'data': [10, 20, 30]}
m.name = "我的列表"   # 在类外也能这样存
```

`self.xxx = ...` 和 `m.xxx = ...` 没有本质区别。**储物柜比喻**：`__init__` 往柜子里放东西，其他方法通过 `self` 打开同一个柜子取出来。

### Q23. 再介绍一下 classmethod 和 staticmethod（Dog 例子）

区别在于 **Python 自动给第一个参数传什么**：

```python
class Dog:
    species = "犬科"              # 类属性，所有狗共享
    def __init__(self, name):
        self.name = name          # 实例属性
    def bark(self): ...           # d.bark() → 自动传 d 给 self
    @classmethod
    def info(cls): ...            # Dog.info() → 自动传 Dog 给 cls
    @staticmethod
    def add(a, b): ...            # Dog.add(1, 2) → 什么都不传
```

- 普通方法能访问 `self.name` 这种每个对象各自的数据。
- 类方法只能访问类层面的东西（`cls.species`），拿不到某只狗的名字。
- 静态方法就是放在类里方便归类的普通函数，如 `Dog.human_age(3)` 返回 21。

### Q24. `from_str` 这段代码看不懂（逐步执行）

```python
@classmethod
def from_str(cls, s):
    name, age = s.split(",")
    return cls(name, int(age))
```

调用 `Dog.from_str("旺财,3")`：

1. 因 `@classmethod`，`cls = Dog`，`s = "旺财,3"`。
2. `"旺财,3".split(",")` 得 `["旺财", "3"]`，解包为 `name="旺财"`、`age="3"`（仍是字符串）。
3. `int(age)` 转成 3；`cls(name, 3)` 就是 `Dog("旺财", 3)`，调用 `__init__` 并返回。

先不看 classmethod，它和普通函数 `def make_dog(s): ... return Dog(name, int(age))` **做的事完全相同**，只是被放进了类里，`Dog(...)` 换成 `cls(...)`。用 `cls` 的好处体现在继承：`Puppy.from_str("小白,1")` 时 `cls` 是 `Puppy`，创建出的是 Puppy；写死 `Dog(...)` 就永远只能造 Dog。

### Q25. `Dog("旺财", 3)` 和 `Dog.from_str("旺财,3")` 的区别？

**结果没有区别**，区别是你手里数据的形式：已分好两个参数就直接用 `Dog(...)`；拿到的是一整行字符串（如逐行读 `dogs.txt`）就用 `from_str`，它帮你拆分和转换。`from_str` 不是新功能，只是**便捷入口**，最终还是调用 `Dog(...)`。

### Q26. 这个例子里 `@classmethod` 起什么作用？

**让你不用先有一只狗，就能通过类 `Dog` 直接调用 `from_str`。** 去掉它，`from_str` 成为普通方法，`Dog.from_str("旺财,3")` 会把字符串当成 `self`，报缺少参数 s；你只能先造一只没用的狗 `d = Dog("随便", 0)` 再 `d.from_str(...)`——为了"创建"狗却得先有狗，显然不合理。

你的总结"把类的调用更加灵活了"是对的，更准确地说是让**创建对象的方式**更灵活：`__init__` 只能有一个，类方法可以有很多个，各对应一种数据来源——`Dog.from_str`、`Dog.from_dict`、`Dog.from_file`。看到 `.from_xxx(...)` 基本就是"用其他格式的数据创建对象"，例如 Hugging Face 的 `AutoModel.from_pretrained("bert-base")`。

### Q27. 那 staticmethod 呢？

作用与 classmethod 相反：**不用来创建对象，而是与类相关的普通工具函数**，Python 什么都不自动传。功能上和写在类外完全一样，放进类里只是归类（工具放进对应工具箱）。实际中 `@staticmethod` 用得最少。

|  | 自动传入 | 典型用途 |
| --- | --- | --- |
| 普通方法 | 对象 `self` | 操作对象自己的数据 |
| `@classmethod` | 类 `cls` | 用其他格式的数据创建对象 |
| `@staticmethod` | 什么都不传 | 与类相关的工具函数 |

### Q28. `def from_str(cls, s)` 有两个参数，调用时只传了一个？

第一个参数 `cls` 由 Python **自动填入**：`Dog.from_str("旺财,3")` 实际执行 `from_str(Dog, "旺财,3")`。和 `self` 一样：`d.bark()` 实际是 `bark(d)`。

规律：**点号前面的东西会被自动传给第一个参数。**

- `d.bark()` → 对象 `d` 传给 `self`
- `Dog.from_str(...)` → 类 `Dog` 传给 `cls`（因为 `@classmethod`）
- `Dog.human_age(3)` → 什么都不传（因为 `@staticmethod`），定义里也就没有 `self`/`cls`

### Q29. 原文 `def parse(cls, value: str) -> "Device"`，返回类型为什么加引号？

返回值类型就是 `Device` 类的对象。加引号是**前向引用**：执行到 `parse` 这一行时 `class Device` 还没定义完，`Device` 这个名字还不存在，而注解在定义时求值，会 `NameError`。写成字符串，Python 只当文字存起来，IDE 和 mypy 能理解它。

其他写法：

- 文件开头 `from __future__ import annotations`，所有注解延迟求值，可直接写 `-> Device`。
- Python 3.11+ 用 `from typing import Self`，`-> Self` 表示"当前这个类"，对子类也成立（同 `cls(...)` 的道理）。
- Python 3.14 起注解默认延迟求值，直接写也不报错。

`Device.parse("cuda:0")` 得 `kind="cuda"`、`index="0"`，返回 `Device(kind="cuda", index=0)`。`cls(kind=kind, index=...)` 用关键字参数，比按位置传更清晰。

## 2.5 迭代器与生成器：按需生产数据

> **📖 原文 [2.5 迭代器与生成器：按需生产数据](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#25-迭代器与生成器按需生产数据)**
>
> **可迭代对象**能返回迭代器；**迭代器**保存遍历状态，并通过 `__next__` 逐个返回元素。生成器函数是编写迭代器的简洁方式：
>
> ```python
> from collections.abc import Iterator
>
> def read_batches(path: str, batch_size: int) -> Iterator[list[str]]:
>     batch: list[str] = []
>     with open(path, encoding="utf-8") as file:
>         for line in file:
>             batch.append(line.rstrip("\n"))
>             if len(batch) == batch_size:
>                 yield batch
>                 batch = []
>     if batch:
>         yield batch
> ```
>
> `yield` 返回一个值后暂停函数，下一次迭代从暂停位置继续。与一次性构造完整列表相比，生成器的峰值内存通常只与一个批次相关，适合流式数据管线。

### Q30. 迭代器与生成器的英文

| 中文 | 英文 | 说明 |
| --- | --- | --- |
| 可迭代对象 | iterable | 能被 `for` 遍历，实现了 `__iter__`，如 list |
| 迭代器 | iterator | 保存遍历状态，实现 `__next__` |
| 生成器 | generator | 生成器函数被调用后返回的对象 |
| 生成器函数 | generator function | 函数体中含 `yield` 的函数 |
| 生成器表达式 | generator expression | 如 `(x*x for x in range(3))` |

易混：list 是 iterable 但不是 iterator，`iter(lst)` 才得到 iterator。

### Q31. 原文 `read_batches` 这里也没有生成器呀？

有。**判断标准：函数体里只要出现 `yield`，它就是生成器函数。** `read_batches` 有两处 `yield`。调用时函数体**不会立即执行**，而是返回生成器对象：

```python
g = read_batches("data.txt", 2)
print(g)   # <generator object read_batches at 0x...>
```

假设文件 5 行 a–e，`batch_size=2`，逐步执行：

1. 第一次取值：读入 a、b，满 2 条，`yield` 交出 `["a", "b"]`，函数**暂停**。
2. 第二次取值：从暂停处继续，清空 batch，读入 c、d，交出 `["c", "d"]`。
3. 第三次取值：读入 e，文件读完，for 结束；batch 剩 `["e"]`，由最后的 `if batch: yield batch` 交出。
4. 第四次取值：函数执行完毕，自动抛出 StopIteration，外层 for 结束。

**省内存**：同一时间最多只保存一个 batch，即使文件 100GB 也只占一个批次的内存。

原文三条注意事项：

1. **只能消费一次**：发牌员发完就空了，要再遍历就重新调用 `read_batches(...)`。
2. **异常推迟**：见下一问。
3. **转成 list 失去优势**：`list(read_batches(...))` 一次性取出所有批次，等于直接读整个文件。

### Q32. "异常推迟"不理解

对比路径写错时：

```python
def read_all(path):
    with open(path) as f:
        return f.readlines()
data = read_all("不存在.txt")   # ← 立刻报 FileNotFoundError

def read_lines(path):
    with open(path) as f:
        for line in f:
            yield line
g = read_lines("不存在.txt")    # ← 不报错！
for line in g:                  # ← 到这里才报错
    ...
```

原因：调用生成器函数时**函数体一行都不执行**，`open` 还没跑；`for` 第一次要元素时才执行到它。实际项目中创建和使用可能相隔很远（中间加载模型花 5 分钟），报错位置指向循环，根源却在创建处。

解决：把需要提前检查的放到生成器外面——

```python
def read_lines(path):
    if not os.path.exists(path):
        raise FileNotFoundError(path)
    return _gen(path)       # read_lines 无 yield，是普通函数，立即检查

def _gen(path):
    with open(path) as f:
        for line in f:
            yield line
```

## 系列目录

- [导读与易错点速查](/posts/ai-infra/ch1-00-guide/)
- [（一）最小工具箱与 Python 对象模型](/posts/ai-infra/ch1-01-python-objects/)
- **（二）作用域、类与生成器**（本篇）
- [（三）装饰器、上下文管理器与异常处理](/posts/ai-infra/ch1-03-decorator-context-exception/)
- [（四）Python 并发与性能分析](/posts/ai-infra/ch1-04-concurrency-profiling/)
- [（五）C/C++ 内存、生命周期与资源管理](/posts/ai-infra/ch1-05-cpp-memory/)
- [（六）C++ 编译、链接、模板与 CMake](/posts/ai-infra/ch1-06-cpp-build/)
- [（七）Linux 开发基本功与环境隔离](/posts/ai-infra/ch1-07-linux-env/)

