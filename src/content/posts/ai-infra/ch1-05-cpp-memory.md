---
title: "AI Infra 第1章笔记（五）：C/C++ 内存、生命周期与资源管理"
published: 2026-11-03T07:00:00
description: "这一章的主线：值、指针、引用是三种不同的\"访问对象\"方式 → 指针不带长度，所以要\"指针 + 长度\"或 span → 资源绑定到对象生命周期（RAII），用 = delete 禁止复制 → 智能指针把这套机制通用化 …"
tags: ["AI Infra", "C++"]
category: "AI Infra 学习笔记"
draft: false
---

这一章的主线：**值、指针、引用是三种不同的"访问对象"方式 → 指针不带长度，所以要"指针 + 长度"或 span → 资源绑定到对象生命周期（RAII），用 `= delete` 禁止复制 → 智能指针把这套机制通用化 → 移动转交所有权，Rule of Zero 让你什么都不用写 → `const` 把只读写进接口。** 很多问题都和 [第一篇](/posts/ai-infra/ch1-01-python-objects/) 的 Python 引用模型对照着理解。

> [!NOTE]
> 本篇对应原文 [AIInfraGuide · 第1章：编程语言基础](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/) 的 §4（作者 [caomaolufei](https://github.com/caomaolufei/AIInfraGuide)，MIT 许可）。每节开头的引用块摘自原文，下面的问答是我学习时提出的问题和整理后的解答；文中的“原文 x.y”都可以点击跳转到原文对应小节。

## 4.1 值、地址、指针与引用

> **📖 原文 [4.1 值、地址、指针与引用](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#41-值地址指针与引用)**
>
> ```cpp
> #include <iostream>
>
> int main() {
>     int value = 42;
>     int* ptr = &value;  // ptr 保存 value 的地址
>     int& ref = value;   // ref 是 value 的别名
>
>     *ptr = 7;           // 解引用指针并写入
>     ref += 1;
>
>     std::cout << value << '\n';  // 8
> }
> ```
>
> 可以先用这张表建立直觉：
>
> | 概念 | 能否为空 | 能否改指向 | 常见用途 |
> |------|----------|------------|----------|
> | 值 `T` | 否 | 不适用 | 拥有一个独立对象 |
> | 指针 `T*` | 可以是 `nullptr` | 可以 | 可选对象、数组、底层接口、非拥有观察 |
> | 引用 `T&` | 语言语义上必须绑定对象 | 不可以重新绑定 | 必须存在的别名、函数参数 |
> | 常量引用 `const T&` | 必须绑定对象 | 不可以 | 避免复制且不允许修改 |

### Q1. 解释 `int* ptr = &value; int& ref = value;` 这段代码

- `int value = 42;`：栈上的 int 变量。
- `int* ptr = &value;`：`&value` 取地址，存进指针变量 `ptr`。`ptr` 本身是一个独立变量，装的是地址。
- `int& ref = value;`：`ref` 是 `value` 的**别名**，不是新变量，对 `ref` 的操作就是对 `value` 的操作。
- `*ptr = 7;`：解引用，顺着地址找到 `value`，改成 7。
- `ref += 1;`：等价于 `value += 1`，变成 8。

|  | 指针 `int*` | 引用 `int&` |
| --- | --- | --- |
| 本质 | 存地址的独立变量 | 已有对象的别名 |
| 访问目标 | 写 `*ptr` | 直接写 `ref` |
| 能否为空 | 可以是 `nullptr` | 必须绑定有效对象 |
| 能否改指向 | 可以 `ptr = &other;` | 不能，初始化后终身绑定 |
| 必须初始化 | 不必（但不初始化很危险） | 必须 |

一般优先用引用（如 `void f(const std::string& s)` 避免拷贝）；需要"可能为空"、中途换指向或对接 C 接口时用指针；涉及所有权时用智能指针（见 4.5）。

### Q2. `int& ref` 这个不是 ref 的地址吗？

**不是。** 同一个符号在不同位置意思不同：

| 写法 | 位置 | 含义 |
| --- | --- | --- |
| `int& ref` | 声明（类型里） | 引用类型 |
| `&value` | 表达式里 | 取地址 |
| `int* ptr` | 声明（类型里） | 指针类型 |
| `*ptr` | 表达式里 | 解引用 |

`ref` 的值就是 `value` 的值；想要 ref 的地址写 `&ref`，会发现 `&ref == &value`，因为本来就是同一个对象。`int& ref` 和 `int &ref` 等价，写成前者是为了强调 `&` 属于类型。

### Q3. 什么是解引用？

**顺着指针里存的地址，找到那个地址上的对象本身。** 指针是门牌号，解引用是按门牌号走进房间读或改东西。`*ptr = 7` 改的是 `ptr` 指向的 `value`；写成 `ptr = 7` 是想把指针本身改成地址 7，编译报错。

结构体指针访问成员用 `->`：`pp->x = 5` 等价于 `(*pp).x = 5`。

坑：只有指针确实指向有效对象时才能解引用。解引用 `nullptr` 或未初始化的野指针 `int* p2; *p2 = 1;` 都是**未定义行为**（[原文 4.8](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#48-未定义行为为什么危险)）。引用必须绑定已有对象，没有"空引用"问题，这是它更安全的原因。

### Q4. `int* a = b` 呢？

取决于 `b` 的类型：

- `b` 是 `int`：**编译报错**，不能把 int 转成 int\*，你想要的是 `int* a = &b;`。
- `b` 是 `int*`：可以，把 b 里的地址复制给 a，两个独立的指针指向同一处；`*a = 7` 后 `*b` 也是 7，但 `a = nullptr` 不影响 b。
- `b` 是 `int` 数组：可以，数组名自动转换成首元素地址（等价 `&b[0]`），叫**数组退化（decay）**。

区分：`int* a = b;` 里的 `*` 属于类型，初始化 a 本身；`*a = b;` 里的 `*` 是解引用，把 b 的值写进 a 指向的对象。

### Q5. `int a = 8, b = a`，b 是指针还是 int 8？

`b` 是普通 int，值为 8。`b = a` 是**拷贝**，之后 a 和 b 各占一块内存，`b = 100` 不影响 a。只有写了 `*` 才是指针，写了 `&` 才是引用。

坑：一行声明多个变量时 `*` 和 `&` 只修饰紧跟的名字——`int* x, y;` 里 x 是 `int*`、**y 是 int**；两个都是指针要写 `int *x, *y;`。很多规范建议一行只声明一个变量。

### Q6. 所以 C 的 `b = a` 就类似于 Python 的深拷贝？

方向对但说法不准：**C/C++ 的 `b = a` 是值拷贝；Python 的 `b = a` 根本不拷贝，只是让 b 也指向同一对象**（[第一篇 Q4](/posts/ai-infra/ch1-01-python-objects/#q4-为什么-rebinddata-之后-data-不变)）。

Python 的 `b = a` 更像 C++ 的**指针赋值**而不是引用：Python 之后写 `b = [9]` 只是让 b 改指向新对象，a 不受影响，这和 C++ 指针改指向一样；C++ 引用一旦绑定就不能改绑。

int 的例子看起来一样，是因为 `b = 100` 让 b 改指向另一个对象 100，加上 Python 的 int 不可变，根本没法原地改 8。区别只在**可变对象**上暴露。

不能叫"深拷贝"：深浅拷贝是针对"对象内部还有指向别处的东西"时说的。C++ 默认的逐成员拷贝碰到指针成员只复制地址，这是**浅拷贝**：

```cpp
struct Buf { int* data; };
Buf a{new int[3]{1, 2, 3}};
Buf b = a;        // 只复制 data 这个地址
b.data[0] = 99;   // a.data[0] 也变成 99
```

`std::vector`、`std::string` 的拷贝会复制所有元素，效果接近深拷贝，但那是它们自己的拷贝构造函数实现的。

|  | C/C++ `b = a` | Python `b = a` |
| --- | --- | --- |
| 是否创建新对象 | 是（值拷贝） | 否（共享） |
| 更像什么 | 复制一份数据 | C++ 的指针赋值 |
| 碰到内部指针 | 默认只拷地址（浅） | 不涉及，根本没拷 |

PyTorch 里：`b = a` 同一个 tensor；`b = a.view(...)` 新 tensor 对象但共享底层内存；`b = a.clone()` 才真正复制数据。

### Q7. Python 的浅拷贝会让变量指向同一个对象，不就类似 C 的指针吗？

混淆了**赋值**和**浅拷贝**：

```python
a = [[1, 2], [3, 4]]
b = a                 # 赋值
c = copy.copy(a)      # 浅拷贝
id(a) == id(b)        # True，同一个列表
id(a) == id(c)        # False，c 是新列表
id(a[0]) == id(c[0])  # True，内层子列表共享
```

浅拷贝是"外层新，内层共享"：`c.append(...)` 不影响 a，`c[0].append(99)` 会影响 a。

| Python | 对应 C/C++ |
| --- | --- |
| `b = a` | 指针赋值 `int* b = a;` |
| `copy.copy(a)` | 结构体默认拷贝 `Buf b = a;`，外层新，内部指针成员仍指向同一块内存 |
| `copy.deepcopy(a)` | 自己写拷贝构造，把指针指向的数据也 `new` 一份 |

Python 的 list 本质上就是"装着指针的数组"，浅拷贝复制的是这个指针数组——和 [第一篇 Q12](/posts/ai-infra/ch1-01-python-objects/#q12-所以浅拷贝类似于-c-语言的指针) 的结论一致。

### Q8. `copy.copy` 和 `a.copy()` 的区别？`a.deepcopy()` 等于 `copy.deepcopy` 吗？

对 list：`a.copy()`、`copy.copy(a)`、`a[:]`、`list(a)` 四种**效果完全一样**，都是浅拷贝。区别在适用范围：`.copy()` 是 list、dict、set、NumPy 数组等少数类型**自己定义的方法**，普通自定义类没有，调用会 `AttributeError`；`copy.copy(x)` 是标准库**通用函数**，会找对象的 `__copy__`，没有就按通用规则复制属性。

| 写法 | 效果 |
| --- | --- |
| `list.copy()` / `dict.copy()` | 浅拷贝 |
| `numpy_array.copy()` | 复制底层数据缓冲区，不共享内存 |
| PyTorch tensor | 没有 `.copy()`；`.clone()` 复制数据；`.copy_(src)` 是把 src 原地写进当前 tensor，方向相反，易混 |

**list 没有 `.deepcopy()` 方法**，会 `AttributeError`，深拷贝必须用 `copy.deepcopy`（要递归复制所有嵌套对象并处理循环引用，统一交给标准库）。自定义类想控制行为就实现 `__deepcopy__(self, memo)`。pandas 的 `df.copy(deep=True)` 是库自己的约定。

### Q9. `std::cout << value << '\n';` 这行 C++ 语法不理解

相当于 Python 的 `print(value)`：

- `std::`：标准库命名空间，`::` 是作用域运算符（类比 `sys.stdout` 的 `sys.`）。
- `cout`：character output，标准输出对象，要 `#include <iostream>`。
- `<<`：输出运算符，"把右边送进左边的流"。`cout << value` 返回 `cout` 本身，所以能连着写 `(std::cout << value) << '\n'`。它原本是左移位（`1 << 3 == 8`），`cout` 通过运算符重载重新定义了它。
- `'\n'`：换行符。**单引号是单个字符 `char`，双引号是字符串**，C++ 区分而 Python 不区分。`cout` 不自动换行；`std::endl` 是换行并立刻刷新缓冲区，平时用 `'\n'` 就够。
- `;`：语句结束符；`// 8`：注释，相当于 `#`。

`using namespace std;` 可省掉 `std::`，练习时方便，大项目不推荐（容易名字冲突）。

### Q10. 引用为什么不能改指向，不可以重新赋值吗？

**可以赋值，但赋值改的是它绑定的那个对象的值，不是让它换绑：**

```cpp
int a = 1, b = 2;
int& ref = a;
ref = b;          // 不是改绑！而是 a = b，a 变成 2
b = 100;
std::cout << ref; // 2，ref 还是 a
```

引用初始化后，代码里每一个 `ref` 都等同于 `a`，C++ 没有任何语法能"指着引用本身"操作它（写 `&ref` 拿到的也是 a 的地址）。指针不同，`ptr` 是指针自己、`*ptr` 是目标，两者能分别修改。这是有意设计：引用一定绑定有效对象、不会中途变成指向别人，函数参数 `void f(int& x)` 可以确定 x 始终是调用者传入的变量。需要可改绑的引用用 `std::reference_wrapper`（常用于放进容器）。

### Q11. 那 `a = 10`，`a = 9` 可以吗？

可以，普通变量随时可重新赋值。两种情况不行：**重复声明** `int a = 10; int a = 9;`（带类型名就是声明新变量，同一作用域名字不能重复，Python 没有声明这一步）；**加了 const** `const int a = 10; a = 9;`。`const int& ref = a;` 同理不能通过 ref 改。

### Q12. 原文表格"常量引用 `const T&`：必须绑定对象、不可以、避免复制且不允许修改"如何理解？

`T` 是任意类型的占位符。

- **必须绑定对象**：`const int& r2;` 报错。
- **不可以（修改）**：不能**通过 r** 修改，只能读。但 a 本身没 const，`a = 20` 后 r 能看到 20——"只读窗口"。
- **避免复制且不允许修改**：主要用途是函数传参。

| 写法 | 会不会复制 | 函数能不能改调用者的数据 |
| --- | --- | --- |
| `f1(T v)` | 会（一百万个 float 复制一份） | 改的是副本 |
| `f2(T& v)` | 不会 | 能改 |
| `f3(const T& v)` | 不会 | 不能改，编译器强制 |

习惯：**只读的大对象参数用 `const T&`**；`int`、`double` 直接传值（复制一个 int 比引用还便宜，对应原文"小型标量按值传递通常最清楚"）。额外特性：`const T&` 能绑定临时值，`int& r1 = 5;` 报错而 `const int& r2 = 5;` 可以，`print("hello")` 也能直接传字面量。

类比：Python 传参不复制，但函数可以随意改传进来的 list，没有保护；`const T&` 在"不复制"上加了编译器检查的只读约束。

### Q13. 原文四种接口（`consume`/`update`/`inspect`/`optional`）类型写在前面，Python 是 `a: int` 写在后面？

|  | C++ | Python |
| --- | --- | --- |
| 参数 | `类型 名字`，如 `int a` | `名字: 类型`，如 `a: int` |
| 返回值 | 函数名前 `int f()` | 后面 `def f() -> int` |
| 无返回值 | `void` | `-> None` |

"类型在前"继承自 C；Python 3.5 起的注解采用"名字在前"，与 TypeScript、Rust、Go、Kotlin 相同。C++11 也支持尾置返回 `auto add(int a, int b) -> int`。本质区别：Python 注解只是提示（`f("hello")` 照样运行），C++ 类型必须写且编译器强制检查，而且类型里还包含"怎么传"的信息，这是 Python 表达不了的（Python 统一传对象引用）。

原文注释里的"借用"是 Rust 术语：

| C++ | Rust | Python 大致对应 |
| --- | --- | --- |
| `void consume(T value)` | `fn consume(value: T)` | 无直接对应，要自己 copy |
| `void update(T& value)` | `fn update(value: &mut T)` | 默认就能改可变对象 |
| `void inspect(const T& value)` | `fn inspect(value: &T)` | 只能靠约定不改 |
| `void optional(T* value)` | `fn optional(value: Option<&T>)` | `value: T \| None` |

### Q14. `value` 一定要写吗，不能写 `consume(int T)`？

`T` 在类型位置是占位符，`value` 是参数名。换成 int 应写 `void consume(int value);`。写 `consume(int T)` 语法能过，但 T 从类型占位符变成了变量名，容易误解；`void` 不能省。

参数名能否省略：**声明时可以**（`void consume(int);`，但写上更清楚，如 `resize(int width, int height)`）；**定义时也可以但函数体里用不了它**，只在"必须接收但用不到"时出现（重写接口、消除未使用参数警告）。Python 不行，`def f(int):` 里的 int 会被当成参数名。

## 4.2 数组、指针与连续内存

> **📖 原文 [4.2 数组、指针与连续内存](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#42-数组指针与连续内存)**
>
> 原生数组在大多数表达式中会退化为首元素指针，因此单独的 `T*` 不携带长度：
>
> ```cpp
> void scale(float* data, std::size_t size, float factor) {
>     for (std::size_t i = 0; i < size; ++i) {
>         data[i] *= factor;
>     }
> }
> ```
>
> 现代 C++ 中优先用能表达长度的类型：

### Q15. 如何用指针表示二维数组？

先记住：**`int**` 和二维数组 `int[3][4]` 不是一回事**，这是最常见的误区。

1. **静态数组 + 数组指针**：`int (*p)[4] = a;`，p 指向"含 4 个 int 的一行"。括号不能省，`int* p[4]` 是"4 个 int 指针组成的数组"。`p[1][2]` 等价于 `*(*(p + 1) + 2)`：`p + 1` 跳过一整行 → `*` 得到第 1 行并退化成 `int*` → `+ 2` 行内后移 → 外层 `*` 取值。
2. **一维指针 + 下标计算**：`int* q = &a[0][0]; q[i * cols + j]`。图像、矩阵、张量底层基本都这样存。
3. **动态 `int**`**：先 `new int*[rows]` 再每行 `new int[cols]`，释放时先逐行 `delete[]` 再 `delete[] m`。**行与行之间不连续**，所以 `int** bad = a;` 编译错误。
4. **动态但保持连续**：`int* m = new int[rows * cols]; m[i * cols + j]`；列数是编译期常量时也可 `int (*m)[4] = new int[rows][4];`。
5. **现代 C++**：`std::vector<std::vector<int>>` 方便但每行不连续；追求性能用一维 `std::vector<int>(rows * cols)` 加 `i * cols + j`。

| 写法 | 类型 | 内存连续 | 适用场景 |
| --- | --- | --- | --- |
| `int a[3][4]` + `int (*p)[4]` | 数组指针 | 是 | 大小固定 |
| `int*` + `i*cols+j` | 普通指针 | 是 | 最通用，性能好 |
| `int**` 逐行 `new` | 指针的指针 | 否 | 各行长度可不同 |
| `vector<vector<int>>` | 容器 | 否 | 写起来方便 |

第 2 种最值得吃透：PyTorch tensor 底层就是"一块连续内存 + 形状 + 步长（stride）"。

### Q16. `void scale(float* data, std::size_t size, float factor)` 里的 `std::size_t` 不理解

注意是 `std::size_t`，`_t` 不能少。它是**专门表示"大小"和"下标"的整数类型**：无符号；足够大（64 位系统通常 64 位）；`_t` 表示"这是一个类型"。`sizeof(int)`、`v.size()` 返回的都是它。为什么不用 int：int 通常 32 位，最大约 21 亿；大小本来不会为负。

三个参数：`data` 指向数组开头，`size` 元素个数，`factor` 倍数。**必须单独传 size，因为指针本身不知道数组多长**（数组传进函数退化成指针，长度丢了），所以 C 风格接口几乎都是"指针 + 长度"。

无符号的坑：`std::size_t n = 0; n - 1;` 得 18446744073709551615；倒序 `for (std::size_t i = size - 1; i >= 0; --i)` 会死循环（`i >= 0` 永远成立），可写成 `for (std::size_t i = size; i-- > 0; )`。

另：C++17 的 `std::size(arr)`（无 `_t`）获取数组或容器长度，只对真正的数组有效，对 `float*` 用不了。

### Q17. 循环里为什么是 `++i`？那 C 呢？

在 for 循环里 `++i` 和 `i++` **效果完全一样**。区别只在表达式的值：`int a = ++i;` 先加后取（6），`int b = j++;` 先取后加（5）。for 第三部分的结果没人用，所以等价。

C++ 习惯 `++i`：`i++` 要"先记下旧值再返回"，对迭代器等类类型意味着多一次拷贝；对 int、size\_t 编译器会优化成一样。于是形成"不需要旧值时统一写 `++i`"的习惯。

C 语言含义完全相同，但 C 程序员更常写 `i++`：C 没有运算符重载和迭代器类，不存在拷贝差异，沿用了 K&R 教材写法。C 版本区别：`size_t` 前没有 `std::`（来自 `<stddef.h>`）；for 内声明变量是 C99 起才支持；C 没有引用，只读参数写成 `const float* data`。

Python 没有 `++`，写 `++i` 不报错但被解析成 `+(+i)`，i 不会加 1。

### Q18. "指针不知道它指向的数组有多长"不理解，`size_t i` 的 i 最大不就是 4 吗？

混了两件事：**`size_t` 是类型，`size` 是参数**。i 最大到 3（循环到 4 停下），这个 4 不是 size\_t 规定的，而是**调用者通过 `size` 参数告诉函数的**——`scale(arr, 4, 2.0f)`。

进到函数后，`data` 里只存了首元素地址：

```text
地址:   0x1000  0x1004  0x1008  0x100C  0x1010  0x1014 ...
内容:    1.0     2.0     3.0     4.0     ???     ???
         ↑ data
```

内存上没有任何"数组到此为止"的标记（只知道一条街的起始门牌号，不知道有几户）。所以同一个 `scale` 可以处理 4 个或 100 个元素的数组。传错长度 `scale(a, 10, 2.0f)` 时函数会老实循环 10 次，改坏后面 6 个位置的内存——**越界访问**，未定义行为，不会像 Python 那样报 `IndexError`。

验证：`main` 里 `sizeof(arr)` 是 16（4 × 4 字节），传进 `f(float* data)` 后 `sizeof(data)` 是 8，只剩一个地址。

### Q19. 那 i 定义为 int 也没问题呀？

大多数情况没问题。用 size\_t 是为了避免两个边缘问题：

1. **有符号/无符号比较警告**：`int i` 与 `size_t size` 比较时 i 被转成无符号；若 `i = -1`，`(i < n)` 结果为 false（-1 变成巨大正数）。
2. **超过 int 范围**：int 最大约 21 亿（2³¹−1），21 亿个 float 是 8GB，大模型参数、大张量单个缓冲区超过 2³¹ 个元素是可能的，PyTorch 底层索引就用 64 位。

社区有争议：一派主张和标准库一致用 size\_t；另一派（如 Google 风格指南）主张下标用有符号整数，避免无符号减法绕回，需要大范围就用 `int64_t` 或 `std::ptrdiff_t`。结论：练习代码用 int 没毛病，正式代码跟项目约定走。

### Q20. 原文"行主序 `A[M][N]` 的 offset = row × N + col"如何理解？

二维矩阵在内存里是一排连续格子，**行主序就是一行接一行地排**。以 `A[3][4]` 为例：

```text
offset:  0  1  2  3 | 4  5  6  7 | 8  9  10 11
内容:    a  b  c  d | e  f  g  h | i  j  k  l
         ---row0--- | ---row1--- | ---row2---
```

找 `A[1][2]`（g）：先跳过 1 整行（1 × 4 = 4 个），再行内往右 2 个，offset = 6。公式只用列数 N，因为"跳过一行走多远"只取决于每行多长。实际地址 = 起始地址 + offset × sizeof(元素)。编译器处理 `a[1][2]` 时就是在内部算 `1 * 4 + 2`。

**列主序**：一列接一列，`offset = col × M + row`。C/C++、NumPy、PyTorch 默认行主序；Fortran、MATLAB、Eigen（默认）、cuBLAS 是列主序，调用 cuBLAS 时不注意会把矩阵当成转置。

推广为 **stride**：`offset = row × stride[0] + col × stride[1]`，行主序 (M, N) 的 stride 是 (N, 1)。`torch.zeros(3, 4).stride()` 是 `(4, 1)`；`.t()` 转置后 stride 变 `(1, 4)`、`is_contiguous()` 为 False——转置不移动数据，只换 stride，所以 `.view()` 可能报错，要先 `.contiguous()`。原文"访问顺序是否连续直接影响 CPU cache 和 GPU 合并访存"也源于此。

### Q21. `std::span` 版本的 `scale` 解释一下

`std::span<float>` 把"指针 + 长度"打包成一个对象（C++20，需 `-std=c++20`）：

- **不拥有数据**，只是视图，不复制也不释放（和 PyTorch 的 view 类似）。
- **按值传递很便宜**（一个指针 + 一个整数），参数直接写 `std::span<float>`。
- `for (float& value : values)` 是**范围 for**，相当于 Python 的 `for value in values`，不用管下标和长度，不会越界。关键在 `float&`：value 绑定真实元素，`value *= factor` 修改原数组；去掉 `&` 每次复制一份，改完就丢，函数等于什么都没做。

有趣的对照：Python 的 `for value in values: value *= 2` 行为和**没有 `&`** 的版本一样，原列表不变（只是让 value 指向新对象），要原地改得写 `values[i] *= 2`。

调用：C 数组 `scale(arr, 2.0f)`、vector `scale(v, 2.0f)` 都能自动构造；只有指针时 `scale(std::span<float>(p, 100), 2.0f)`。只读版本写 `std::span<const float>`，作用类似 `const T&`。注意 span 不拥有数据，原数组被释放后再用就访问无效内存，适合作函数参数"临时借用"，不适合长期保存（原文"调用期间底层内存必须仍然有效"）。

### Q22. `vector` 和 `float arr[N]` 的区别

|  | `float arr[N]` | `std::vector<float>` |
| --- | --- | --- |
| 大小 | 编译期固定（`float arr[n]` 用运行时 n 在标准 C++ 不允许） | 运行时可变（`push_back`、`resize`） |
| 数据存放 | 栈（局部变量时，栈通常只有几 MB） | 堆（可以几 GB），对象本身只有几个指针 |
| 内存管理 | 随作用域自动 | 析构时自动释放，不用手动 `new/delete` |
| 知道长度 | 传进函数后丢失 | 始终 `.size()` |
| 赋值 / 复制 | `b = a;` 编译错误 | 可以，复制全部元素 |
| 越界检查 | 无 | `[]` 无，`.at()` 有 |
| 内存连续 | 是 | 是 |

大小固定又想要"知道长度、能复制"就用 `std::array<float, 4>`，同样在栈上、零开销。习惯：大小不固定用 vector，固定且小用 std::array，原生数组主要在对接 C 接口时用。

### Q23. buffer 是什么？

核心都是"一块存数据的内存"，角色因场景而异：

| 场景 | buffer 指的是 |
| --- | --- |
| C/C++ 一般用法 | 一块连续内存，装一批数据（`new float[1024]`、`vector<float>(1024)`） |
| I/O（cout、文件、网络） | 暂存数据的中转区，攒够再统一处理 |
| NumPy / tensor 底层 | 真正存数字的那块连续内存（PyTorch 叫 storage） |
| PyTorch `register_buffer` | 不训练、但属于模型状态的 tensor |

"缓冲"原意是两边速度不一致时中间暂存：`cout << "hello"` 先进输出缓冲区，满了或刷新时才写屏幕——这就是 `'\n'` 与 `std::endl`（强制刷新，频繁用反而慢）的区别。`register_buffer` 的 tensor 不参与梯度更新、跟着 `.to(device)`、存进 `state_dict`，如 BatchNorm 的 `running_mean`/`running_var`，以及 MoCo 的负样本队列、EMA 统计量。

### Q24. vector 创建后怎么取第几个值？`.at()` 的"检查越界"是什么意思？

下标从 0 开始：`v[2]`、`v.at(2)`、`v.front()`、`v.back()`，都可读可写。

- `v[10]`：不检查，越界是未定义行为，可能读到看似正常的垃圾值（最难查）或崩溃。
- `v.at(10)`：取值前先判断 `if (i >= size_) throw std::out_of_range(...)`，行为最接近 Python 的 `IndexError`，可以 `catch (const std::out_of_range& e)`。

`[]` 快，`.at()` 安全，调试阶段或下标不确定时用 `.at()`。

**没有负数下标**：`v[-1]` 是越界（下标是 size\_t，-1 变成巨大的数），取最后一个用 `v.back()` 或 `v[v.size() - 1]`。空 vector 上 `front()`、`back()`、`[0]` 都是未定义行为，先 `if (!v.empty())`。传给 C 风格函数用 `v.data()` 取首元素指针：`scale(v.data(), v.size(), 2.0f)`。

## 4.3–4.4 对象生命周期与 RAII

> **📖 原文 [4.3 对象存储期与生命周期](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#43-对象存储期与生命周期)**
>
> 常见存储位置的说法是“栈”和“堆”，但更准确地应关注**对象的存储期和所有权**：
>
> ```cpp
> void example() {
>     int local = 1;                  // 自动存储期，离开作用域销毁
>     auto buffer = new float[1024];  // 动态分配，必须显式 delete[]
>     delete[] buffer;
> }
> ```
>
> 手工 `new`/`delete` 容易产生：

> **📖 原文 [4.4 RAII：把资源绑定到对象生命周期](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#44-raii把资源绑定到对象生命周期)**
>
> RAII（Resource Acquisition Is Initialization）在构造时获取资源，在析构时释放资源：
>
> ```cpp
> #include <cstdio>
> #include <stdexcept>
>
> class File {
> public:
>     explicit File(const char* path)
>         : handle_(std::fopen(path, "rb")) {
>         if (handle_ == nullptr) {
>             throw std::runtime_error("failed to open file");
>         }
>     }
>
>     ~File() {
>         std::fclose(handle_);
>     }
>
>     File(const File&) = delete;
>     File& operator=(const File&) = delete;
>
> private:
>     std::FILE* handle_;
> };
> ```
>
> 无论函数正常返回还是抛出异常，局部对象的析构函数都会执行。文件、锁、主机内存、GPU 内存、CUDA Stream 都可以用同一思想管理。

### Q25. 原文 `class File` 的 RAII 代码，C++ 语法不了解，逐段解释

**RAII**：用类把"打开文件"和"关闭文件"绑定在对象的创建和销毁上，效果类似 Python 的 `with open(path, "rb") as f:`（[第三篇](/posts/ai-infra/ch1-03-decorator-context-exception/)）。

- `#include <cstdio>`（`fopen`、`fclose`、`FILE`）、`#include <stdexcept>`（`std::runtime_error`），相当于 import。
- `class File { public: ... private: ... };`：`public:` 下的成员外面能访问，`private:` 下的只有类内部能访问，**编译器强制**（Python 只靠 `_` 约定）。最后的 `};` 分号不能少。
- `std::FILE* handle_;`：唯一的数据成员，指向"已打开文件"的指针。`std::FILE` 是 C 标准库的结构体，存文件描述符、读写位置、缓冲区等，内部字段不需要也不应该访问。\*\*handle（句柄）\*\*是"操作某个资源的凭证"。生命周期：`fopen` 拿到句柄 → `fread(buffer, 1, 100, handle_)` 读 → `fclose(handle_)` 关。C 没有方法，只能把句柄作为参数传给函数；Python 写成 `f.read()`。末尾 `_` 是成员变量命名习惯。
- **构造函数** `explicit File(const char* path)`：创建对象时自动调用，相当于 `__init__`；名字和类名相同、**没有返回类型**。`const char* path` 是 C 风格字符串。
- `: handle_(std::fopen(path, "rb"))`：**成员初始化列表**，在函数体 `{ }` 执行之前就用 fopen 的返回值初始化 `handle_`。`"rb"` 同 Python 的读 + 二进制模式。
- `if (handle_ == nullptr) throw std::runtime_error(...)`：fopen 失败返回空指针，抛异常相当于 `raise RuntimeError(...)`，构造函数抛异常意味着对象创建失败，调用者用 `try/catch` 捕获。
- **析构函数** `~File() { std::fclose(handle_); }`：对象销毁时自动调用，名字是 `~` 加类名。局部对象离开 `{ }` 时**一定**调用——正常结束、提前 `return`、中途抛异常都会。RAII 的核心：**资源的生命周期 = 对象的生命周期**（原文"让资源由对象管理，让对象由作用域管理"）。Python 的 `__del__` 时机取决于垃圾回收、不确定，所以要靠 `with`；C++ 析构时机确定，析构函数本身就起了 `with` 的作用。
- `File(const File&) = delete; File& operator=(const File&) = delete;`：禁止复制，见 Q28–Q32。

| C++ 语法 | 作用 | Python 大致对应 |
| --- | --- | --- |
| `class File { ... };` | 定义类 | `class File:` |
| `public:` / `private:` | 访问控制 | 无强制，靠 `_` |
| `File(...)` | 构造函数 | `__init__` |
| `: handle_(...)` | 成员初始化列表 | `self.handle_ = ...` |
| `explicit` | 禁止隐式转换 | 无 |
| `throw` | 抛异常 | `raise` |
| `~File()` | 析构函数，确定时机调用 | `with` 的退出部分 |
| `= delete` | 禁止某个函数 | 无 |

### Q26. `explicit`：防止隐式转换，不理解

**隐式转换**：`double d = 3;` 编译器悄悄把 int 3 转成 3.0。C++ 规定**只有一个参数的构造函数会被当成"可以从这个参数类型自动转换过来"的规则**：

```cpp
class Vector { public: Vector(int size); };
void print(const Vector& v);
print(10);   // 能编译！编译器悄悄执行 print(Vector(10))
```

写 `print(10)` 的人可能想打印数字 10，结果打印的是长度为 10 的向量，不报错却行为错误。加 `explicit` 后 `print(10)` 编译错误，必须写 `print(Vector(10))`——这个构造函数**只能被明确写出来调用**。

回到 File：没有 explicit 时 `process("data.bin")` 会悄悄 `process(File("data.bin"))`，读代码的人想不到这里打开了文件、还可能抛异常。`File f = "data.bin";`（拷贝初始化）形式上也是隐式转换，加了 explicit 后只能写 `File f("data.bin");`（直接初始化）。加了 explicit 的行为和 Python 一样：想创建对象必须明确写出来。规律：单参数构造函数习惯上都加 explicit；`std::string s = "hello";` 是少数有意允许自动转换的例子。

### Q27. `File f("data.bin")` 是在哪里定义的？

它不是定义，是**使用**这个类的代码，结构和 `int a = 10;` 相同：`类型 变量名 (传给构造函数的参数)`，相当于 Python 的 `f = File("data.bin")`。`"data.bin"` 类型是 `const char*`，匹配构造函数参数 `path` → fopen → 指针存进 `f.handle_` → 失败就抛异常。`main` 结束时 f 销毁，自动 `~File()` 关闭文件。类定义描述"长什么样、创建和销毁时做什么"，`File f(...)` 是按描述造出一个对象。

### Q28. `= delete` 那两行还是不能理解（原理）

复制有两种写法：`File b = a;`（用已有的 a 创建新对象）和 `b = a;`（b 已存在，再把 a 赋给它）。**你一个字都不写，C++ 也会自动生成两个函数**处理它们：拷贝构造函数、拷贝赋值运算符，做法是**逐成员原样复制**——File 只有 `handle_`，就是复制这个指针。

```text
a.handle_ ──┐
            ├──> 打开的 data.bin
b.handle_ ──┘
```

离开作用域时 b 析构 fclose 一次，a 析构又 fclose 一次，**同一个文件关两次**，未定义行为（"钥匙主人离开时拆房子"，钥匙复制一份就拆两次）。这和 Q6 的 `struct Buf` 浅拷贝问题完全一样。这两行就是"那两个自动生成的复制函数我不要了"，之后谁复制 File 编译器直接报错，问题在编译阶段被拦住。

为什么 Python 没这问题：Python 的 `b = a` 只是让 b 也指向同一个对象，**对象始终只有一个**；C++ 的 `File b = a;` 是真的再造一个新对象，每个对象都会在销毁时关一次文件。

### Q29. 原理理解了，但语法不理解

从熟悉的语法一步步拼：

1. 普通函数：`int add(int x, int y) { return x + y; }`
2. 只写声明：`int add(int x, int y);`
3. 声明里参数名可省：`int add(int, int);`
4. 构造函数名字和类名一样、无返回类型。"参数是另一个 File"的构造函数就是 `File(const File& other);`，省略参数名得 `File(const File&);`——这就是**拷贝构造函数**，`File b = a;` 时调用。
5. **`b = a;` 会被编译器翻译成函数调用 `b.operator=(a);`**，`operator=` 就是一个长得怪的函数名。按 `返回类型 函数名 (参数)` 写：`File& operator=(const File& other)`，返回 `File&`（返回自己，这样 `a = b = c` 能连写，固定写法）。
6. 如果允许复制，完整实现大概是 `File(const File& other) : handle_(other.handle_) {}` 和 `File& operator=(const File& other) { handle_ = other.handle_; return *this; }`（`*this` 是"当前这个对象"）——编译器自动生成的就是这两段，也正是会导致关两次的那两段。
7. 把 `{ 函数体 }` 换成 `= delete`，再省略参数名，就得到原文那两行。

| 你写的代码 | 编译器去找哪个函数 | 这个函数的声明 |
| --- | --- | --- |
| `File f("data.bin");` | 参数是字符串的构造函数 | `File(const char*)` |
| `File b = a;` | 参数是 File 的构造函数 | `File(const File&)` |
| `b = a;` | 名叫 `operator=` 的函数 | `File& operator=(const File&)` |

### Q30. `operator=` 什么时候才算调用，出现等号就是调用吗？

**不是，看等号左边的对象是不是刚刚才被创建：**

- 左边是**正在声明的新对象**（前面带类型名）→ **初始化** → 调用**构造函数**：`File b = a;` 调拷贝构造
- 左边是**已经存在的对象** → **赋值** → 调 **`operator=`**：`c = a;`

判断技巧：看这一行有没有在声明变量。其他带等号的也都不是 `operator=`：`= delete` / `= default` 是语法标记；`void f(int x = 0)` 是默认参数；`==` 对应 `operator==`；`+=` 对应 `operator+=`。`int x = 5; x = 10;` 内置类型赋值就是一条机器指令；只有类对象的 `b = a` 才被翻译成 `b.operator=(a)`。Python 的 `b = a` 永远不调用函数，也不允许重定义 `=`。

### Q31. `File& operator=(const File&) = delete;` 不会误伤所有的赋值吗？

**不会**，两层限制：

1. **写在 `class File` 里，只管 File 对象**：`int x = 2;`、`v1 = v2;`（vector 有自己的 operator=）都正常；类内部给成员赋值 `handle_ = nullptr;`（`FILE*` 是内置指针类型）也不受影响。
2. **参数类型决定"右边是什么"才匹配**：可以重载多个 `operator=`，如再加 `File& operator=(const char* path)` 用于重新打开别的文件，`f = "b.bin";` 合法，`f = other;` 被禁止。

精确地只禁止 `File 对象 = 另一个 File 对象`。（补充：C++11 还有移动赋值 `operator=(File&&)`；删掉拷贝操作后编译器也不会自动生成移动操作，所以这个 File 既不能复制也不能移动，要转交所有权得自己写。）

### Q32. `= delete` 是返回错误吗？

**不是。** 它让调用它的代码**根本编译不过**，程序都生成不出来。

|  | `= delete` | `throw` |
| --- | --- | --- |
| 什么时候发生 | 编译时 | 运行时 |
| 结果 | 编译失败，没有可执行文件 | 运行到那一行时抛异常 |
| 能不能 catch | 不能也不需要 | 可以 `try/catch` |

打开文件成不成功只有运行时才知道，所以用 throw；"复制 File"写代码时就能确定是错的，所以用 `= delete` 在编译期拦住，报 `error: use of deleted function 'File::File(const File&)'`。`= delete` 的函数**没有函数体**，不是"返回错误的函数"，而是"被标记为禁止使用的函数"。Python 只能写 `def __copy__(self): raise TypeError(...)`，要运行到 `copy.copy(f)` 才报错——这是静态类型语言的优势（[原文 5.2](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#52-编译错误链接错误与运行时装载错误) 区分编译、链接、运行时错误）。

## 4.5 智能指针与所有权

> **📖 原文 [4.5 智能指针与所有权](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#45-智能指针与所有权)**
>
> ```cpp
> #include <memory>
>
> auto unique = std::make_unique<Buffer>(1024);  // 唯一所有权
> auto shared = std::make_shared<Buffer>(1024);  // 共享所有权
> ```
>
> | 类型 | 语义 | 成本与注意点 |
> |------|------|--------------|
> | `std::unique_ptr<T>` | 唯一拥有，可移动不可复制 | 成本接近裸指针，默认首选 |
> | `std::shared_ptr<T>` | 引用计数共享拥有 | 有计数与控制块成本，循环引用需 `weak_ptr` |
> | `std::weak_ptr<T>` | 不增加引用计数的观察者 | 使用前要 `lock()` 检查对象是否仍存在 |
> | `T*` / `T&` | 通常表示非拥有访问 | 被指向对象必须活得足够久 |
>
> 如果所有权没有共享需求，不要使用 `shared_ptr`。“避免思考所有权”不是共享所有权的合理理由。

### Q33. `std::make_unique<Buffer>(1024)` 和 `std::make_shared<Buffer>(1024)` 解释一下

`Buffer` 是作者自定义的类（构造函数接收大小）。语法：`#include <memory>`；`std::make_unique<Buffer>` 在堆上 new 一个 Buffer；`(1024)` 传给 Buffer 构造函数；返回 `std::unique_ptr<Buffer>`；`auto` 让编译器推断类型。

**为什么需要**：手动 `new` 必须手动 `delete`，忘了写、中途 return 或抛异常都会泄漏。智能指针把普通指针包进对象，**在析构函数里自动 delete**——和 File 的 RAII 思路完全一样。用法：`unique->some_method()`、`(*unique).some_method()`。

**`unique_ptr`：唯一所有权**。同一时间只有一个拥有对象，销毁时 delete。**不能复制**（内部就是用 `= delete` 禁的），`auto b = a;` 编译错误；`auto c = std::move(a);` 把所有权转交给 c，a 变空。

**`shared_ptr`：共享所有权**。可以复制，内部维护**引用计数**：`auto s2 = s1;` 计数变 2，s2 销毁后变 1，最后一个销毁计数归 0 才 delete。`s1.use_count()` 查看计数。

**Python 的所有对象本质上都像 shared\_ptr**：CPython 靠引用计数管理内存（[第四篇 Q4](/posts/ai-infra/ch1-04-concurrency-profiling/#q4-介绍一下-gil)），`b = a` 不复制只共享（`sys.getrefcount(a)` 会比预期多 1，传参本身也算一次）。C++ 让你**选择**：默认值拷贝、想共享用 shared\_ptr、想独占用 unique\_ptr。

|  | `unique_ptr` | `shared_ptr` |
| --- | --- | --- |
| 所有者数量 | 只有一个 | 可以多个 |
| 能否复制 | 不能，只能 `std::move` 转交 | 可以，计数 +1 |
| 何时释放 | 这个指针销毁时 | 计数归 0 时 |
| 开销 | 和普通指针一样 | 要维护计数和控制块 |
| 类比 | 前面的 File 类 | Python 的所有对象 |

习惯：**默认用 unique\_ptr**（原文"避免思考所有权不是共享所有权的合理理由"），尽量用 make\_unique/make\_shared。坑：两个对象用 shared\_ptr 互相指着，计数永远不归 0——**循环引用**，用 weak\_ptr 解决（Q39）；Python 也有这问题，所以 CPython 另有垃圾回收器。

### Q34. File 靠类里禁用复制保证唯一性，Buffer 是靠 make\_unique 和 make\_shared 吗？

更准确：**保证唯一性的是 `unique_ptr` 类型本身，不是 `make_unique` 函数**；shared\_ptr 根本不保证唯一。`make_unique` 只是"new 一个对象并装进 unique\_ptr"的创建工具。unique\_ptr 内部大致就是 `~unique_ptr() { delete ptr_; }` + 两行 `= delete`，**和 File 是同一个机制**。

区别在保护写在"里面"还是"外面"：File 生来就不能复制；`unique_ptr<Buffer>` 禁止的是复制**指针**，如果 Buffer 自己允许复制，`Buffer copy = *p;` 照样能复制出一个独立的新 Buffer。unique\_ptr 保证的是"**这一个**堆上的 Buffer 只有一个主人"。好处是通用：任何类型都能套一层获得唯一所有权。

|  | 怎么保证只释放一次 | 保护写在哪 |
| --- | --- | --- |
| `File` | 禁止复制，只有一个主人 | 类内部 |
| `unique_ptr<T>` | 禁止复制，只有一个主人 | 外层包装，对任意 T 通用 |
| `shared_ptr<T>` | 允许复制，计数归 0 才释放 | 外层包装，对任意 T 通用 |

### Q35. 我没看到这是一个指针；File 能用这个方式实现禁止复制吗？

**看不到指针**是因为 `auto` 藏了类型，写全是 `std::unique_ptr<Buffer> unique = ...`。它**不是原生指针而是类的对象**，内部存着真正的 `Buffer*`，通过重载 `operator->`、`operator*` 用起来像指针；`if (unique)` 判空；`unique.get()` 取出原生指针。

**File 可以，而且更推荐**——关键是把 unique\_ptr 作为 File 的**成员**（外面包一层时 `File copy = *p;` 仍能复制，又会关两次）：

```cpp
struct FileCloser {
    void operator()(std::FILE* f) const { std::fclose(f); }
};

class File {
public:
    explicit File(const char* path) : handle_(std::fopen(path, "rb")) {
        if (!handle_) throw std::runtime_error("failed to open file");
    }
private:
    std::unique_ptr<std::FILE, FileCloser> handle_;
};
```

- **析构函数没了**：handle\_ 销毁时自动调用 FileCloser 去 fclose。
- **两行 `= delete` 也没了**：某个成员不能复制，编译器就不给这个类生成拷贝函数。
- **自动获得移动能力**：`File c = std::move(a);` 可以，能从函数返回、放进 vector。
- 第二个模板参数是**删除器**，默认是 delete，fopen 打开的必须 fclose。`void operator()(...)` 重载函数调用运算符，类比 Python 的 `__call__`。

这就是 **Rule of Zero**（[原文 4.6](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#46-拷贝移动与-rule-of-zero)，见 Q42）。

### Q36. 什么时候可以用 `auto` 来构造变量？

`auto` = "让编译器根据等号右边推断类型"，**必须有初始值**（`auto x;` 报错）。它**不是动态类型**：`auto a = 10; a = "hello";` 报错，a 永远是 int。

| 情况 | 用不用 `auto` |
| --- | --- |
| 右边已写明类型（`make_unique<T>`、`std::vector<T>(...)`） | 用 |
| 类型名很长（如 map 的 iterator） | 用 |
| lambda（如 `auto square = [](int x) { return x * x; };`） | 必须用 |
| 范围 for（`auto&` 修改、`const auto&` 只读） | 常用 |
| 从右边看不出类型（`auto result = compute();`） | 最好写出来 |
| 需要特定数值类型（`auto n = 0;` 是 int 不是 size\_t） | 写出来 |
| 没有初始值 | 不能用 |

坑：**auto 会丢掉引用和 const**。`int& ref = value; auto a = ref;` 的 a 是 int（复制一份）；`auto& b = ref;` 才是引用。经验：auto 是为了让代码更清楚，不是为了少打字。

### Q37. 为什么不写成 `std::unique_ptr<Buffer> unique(1024)`？

因为 **unique\_ptr 的构造函数接收的是一个指针，不是 Buffer 的构造参数**（`explicit unique_ptr(T* p)`）。它是管理者，不负责创建对象。写 `unique(1024)` 编译报错，1024 根本到不了 Buffer。不用 make\_unique 的正确写法：`std::unique_ptr<Buffer> unique(new Buffer(1024));`。`make_unique` 内部基本就是 `return std::unique_ptr<T>(new T(args...));`，把参数转交给 T 的构造函数。

**盒子比喻**：unique\_ptr 是盒子，Buffer 是里面的东西。`unique_ptr<Buffer> p(1024)` 相当于对盒子说"你的大小是 1024"，盒子只问"要我装哪个东西"。（对比 `std::vector<float> v(1024);` 可以，因为 vector 构造函数就是接收大小。）

推荐 make\_unique：不出现裸 new/delete；类型不用写两遍；异常安全。`make_shared` 还会把对象和引用计数**一次分配在一起**，少一次内存分配。

### Q38. Buffer 应该是可以共享的吧？你传给了 unique\_ptr，但 unique\_ptr 不可以共享呀

要区分**Buffer 这个类**和**某一个具体的 Buffer 对象**。类本身没规定共享或唯一，决定一个对象怎么被管理的是你把它交给谁（同一种书，有人自己收着，有的放图书馆）。

**"唯一所有权"不等于"别人不能用"**：unique\_ptr 限制的是**谁负责释放**，不是**谁能访问**。其他代码可以借用：

```cpp
void process(Buffer& buf);         // 借用
void inspect(const Buffer& buf);   // 只读借用
void maybe(Buffer* buf);           // 借用，可能为空
auto p = std::make_unique<Buffer>(1024);
process(*p); inspect(*p); maybe(p.get());
```

这正是原文那四种参数写法的用途。真正需要 shared\_ptr 的是**多个地方都要拥有它、且不确定谁最后用完**（多线程、多个对象持有同一份数据）。先 unique 后来要共享：`std::shared_ptr<Buffer> s = std::move(u);` 可以升级，反过来不行。所以**默认 unique\_ptr**。

### Q39. `std::weak_ptr<T>`："不增加引用计数的观察者，使用前要 lock() 检查"的作用？

**能"看着"一个由 shared\_ptr 管理的对象，但不拥有它，不会让它多活一秒。** shared\_ptr 会让计数 +1；原生指针在对象释放后**不知道**它没了（悬空指针）。weak\_ptr 在两者之间。

```cpp
std::weak_ptr<Buffer> observer;
{
    auto owner = std::make_shared<Buffer>(1024);   // 计数 1
    observer = owner;                               // 计数还是 1
    if (auto p = observer.lock()) { p->resize(2048); }
}   // 计数归 0，释放
if (auto p = observer.lock()) { /* 不会进来 */ } else { /* 对象已经没了 */ }
```

weak\_ptr **不能直接用**，必须 `lock()`：活着返回 shared\_ptr，已释放返回空。先换成 shared\_ptr 是因为使用期间别的线程可能销毁最后一个 shared\_ptr，`lock()` 临时计数 +1 **保证你用的时候对象不会被释放**。

**用途 1：打破循环引用**。Parent 和 Child 用 shared\_ptr 互指，两边计数降不到 0；把 Child 里改成 `std::weak_ptr<Parent> parent;`（父亲拥有孩子，孩子只是记得父亲），树、图、双向链表常这样设计。**用途 2：缓存、观察者**：`cache.lock()` 还在就复用，没了就重建，缓存不会一直占内存。Python 对应 `weakref.ref(obj)`，`r()` 相当于 `lock()`，释放后返回 None。

|  | 拥有对象 | 影响释放时机 | 能否知道已释放 |
| --- | --- | --- | --- |
| `shared_ptr` | 是 | 是，计数 +1 | 只要它在对象就在 |
| `weak_ptr` | 否 | 否 | 能，`lock()` 或 `expired()` |
| 原生指针 `T*` | 否 | 否 | 不能，会变成悬空指针 |

weak\_ptr 只能配合 shared\_ptr（要靠那块引用计数信息）。

### Q40. 由 `auto` 引出的数值类型：double 和 float、`3.14` 和 `3.14f`、double 和 long、有没有 long float

|  | `float` | `double` |
| --- | --- | --- |
| 大小 | 4 字节 | 8 字节 |
| 有效数字 | 约 7 位十进制 | 约 15–16 位 |
| 表示范围 | 约 ±3.4×10³⁸ | 约 ±1.8×10³⁰⁸ |

`float f = 3.14159265358979;` 打印出 3.14159274101257，第 8 位开始不对；`16777216.0f + 1.0f` 仍是 16777216（2²⁴ 处精度不够）。消费级 GPU 的 double 算力常只有 float 的 1/32 甚至 1/64，深度学习用 float32 乃至 float16/bfloat16。**默认精度不一致的坑**：Python float 实际是 C double；`np.array([1.0]).dtype` 是 float64；`torch.tensor([1.0]).dtype` 是 float32；`torch.from_numpy(...)` 得 float64，会报 `expected scalar type Float but found Double`，要 `.float()` 或 `.astype(np.float32)`。

**加 f 就是 float**：C++ **字面量本身也有类型**，带小数点默认 double，整数默认 int；后缀：`3.14f` float、`3.14L` long double、`10u` unsigned、`10L` long、`10LL` long long。默认 double 是从 C 继承的。易错：`3f` 错误（要 `3.f` 或 `3.0f`）；`10 / 3` 是 3（整数除法）。写 float 代码不加 f，`x * 0.5` 会悄悄提升成 double，CUDA kernel 里忘加 f 是常见性能问题。C++23 有 `std::float16_t`、`std::bfloat16_t`（后缀 `f16`、`bf16`）。

**double vs long**：double 是浮点、long 是整数（`long c = 3.7;` 得 3）。同样 8 字节：long 能精确表示 ±2⁶³ 内每一个整数；double 是 1 位符号 + 11 位指数 + 52 位尾数（科学计数法），用精度换范围，超过 2⁵³ 不能精确表示每个整数。计数、下标、ID、金额用整数；测量值、概率、权重用浮点。long 在 Windows 只有 4 字节，确定宽度用 `std::int32_t`、`std::int64_t`；`torch.long` 就是 int64。`long double` 是更高精度的浮点，和 long 无关。

**long float**：标准 C/C++ **没有**。K&R C 里它是 double 的别名，1989 年标准化时删掉，痕迹是 `scanf("%lf", &d)` 读 double 必须用 `%lf`，`printf` 时 float 自动提升成 double，都用 `%f`。浮点类型只有 float、double、long double。

## 4.6 拷贝、移动与 Rule of Zero

> **📖 原文 [4.6 拷贝、移动与 Rule of Zero](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#46-拷贝移动与-rule-of-zero)**
>
> `std::vector` 等容器拥有资源，复制会复制内容，移动则转移内部资源：
>
> ```cpp
> #include <utility>
> #include <vector>
>
> std::vector<float> make_buffer() {
>     std::vector<float> values(1'000'000, 0.0f);
>     return values;  // 编译器通常消除复制，必要时可移动
> }
>
> auto a = make_buffer();
> auto b = std::move(a);  // a 仍有效，但内容处于未指定状态
> ```
>
> 如果类只由标准库 RAII 成员组成，通常无需手写析构、复制和移动操作，这就是 **Rule of Zero**。只有直接管理特殊资源时才需要自定义这些操作，并清晰定义所有权。

### Q41. `make_buffer()` 的目的是什么？每次创建一个 1'000'000 长度的向量吗？`0.0f` 是什么？为什么叫 buffer？

这是**演示用例子**，说明两件事：函数返回很大的 vector 不会很慢；`std::move` 怎么工作。每调用一次就新建一个 100 万个 float 的 vector，`auto a = make_buffer(); auto c = make_buffer();` 互不相干。选 100 万是为了强调"对象很大"——真逐个复制开销可观，而注释说编译器通常直接把 values 构造在调用者的 a 那里，一次都不复制，做不到也会移动。实际类似函数如 `load_image(path)`、`allocate_workspace(n)`。

- `0.0f`：值为 0 的 float 字面量。`std::vector<float> values(1'000'000, 0.0f);` 是"100 万个元素、每个初始为 0"，相当于 `np.zeros(1_000_000, dtype=np.float32)`。不写也会初始化为 0，写出来更明确。
- `1'000'000`：C++14 数字分隔符，等于 `1000000`，相当于 Python 的 `1_000_000`（[第三篇 Q15](/posts/ai-infra/ch1-03-decorator-context-exception/#q15-with-timerload-data-data---range100_000-会打印很多次吗)）。
- **为什么叫 buffer**：`vector` 是**类型**，`buffer` 描述**用途**（Q23），就像 `int count` 类型是 int、用途是计数。
- `auto b = std::move(a);`：把 a 的内容**转交**给 b，不复制 100 万个元素。vector 内部有个指向堆数据的指针，b 直接接过指针、a 的被清掉，只交换几个指针和数字。"a 仍有效但未指定状态"：a 还是合法对象，可以重新赋值或正常销毁，但标准不保证里面还剩什么（通常是空），移动后别再读 a 的内容。

### Q42. 原文"Rule of Zero"和"不要随手对返回值写 `std::move`"不理解

**1. Rule of Zero**：类有析构、拷贝构造、拷贝赋值、移动构造、移动赋值这几个特殊操作，不写时编译器生成的版本都是**对每个成员分别执行对应操作**。

- **成员都是标准库 RAII 类型**（`std::vector<float> data_; std::string name_; std::unique_ptr<Buffer> cache_;`）：销毁时各自释放，复制时 vector/string 正确复制内容、unique\_ptr 不能复制于是类自动不可复制，移动时各自正确移动。每个成员都"知道自己怎么处理"，你**一个都不用写**。
- **直接管理特殊资源**（最早版本的 `std::FILE* handle_`）：原生指针不会 fclose、复制只复制地址，生成的版本是错的，**必须自己写**并"清晰定义所有权"——资源归谁？能否复制？能否转交？谁负责释放？
- **把后者变成前者**：就是 Q35 的改写。原则：尽量让每个特殊资源被一个专门的小类（或 unique\_ptr）包起来，其他类只用这些包装好的成员。

**2. 不要对返回值随手写 `std::move`**：`return values;` 时编译器做**返回值优化**（RVO），一开始就把 values 建在调用者 a 的位置，两者是**同一块内存**，零开销。写成 `return std::move(values);` 后 return 后面变成函数调用表达式，RVO 失效，只能"构造 values → 移动到 a → 销毁空 values"，本来零开销你主动加了一步；对移动很贵或不可移动的类型差别更明显。而且就算做不了 RVO，`return values;` 也会**自动**尝试移动。规则：**返回局部变量直接 `return 变量名;`**。

**3. 移动后不要假设源对象还保留原内容**：移动后对 a 安全的操作只有重新赋值、`clear()`、或什么都不做让它销毁；`a.size()` 别依赖，`a[0]` 很可能越界（交出钥匙后手里只剩空钥匙扣）。

| 原则 | 意思 |
| --- | --- |
| Rule of Zero | 成员都是会自管理的类型时，不用手写析构、复制、移动 |
| 需要自定义时 | 直接持有原生指针、文件句柄等资源时必须自己写，并想清楚所有权 |
| 返回局部变量 | 直接 `return values;`，不要加 `std::move` |
| 移动之后 | 源对象只能重新赋值、清空或销毁，不要读内容 |

## 4.7 `const`：把不可修改写进接口

> **📖 原文 [4.7 `const`：把不可修改写进接口](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#47-const把不可修改写进接口)**
>
> ```cpp
> const float* p1;        // 指向 const float：不能通过 p1 改值
> float* const p2 = ptr;  // const 指针：不能让 p2 改指向
> const float* const p3 = ptr;
> ```
>
> 成员函数末尾的 `const` 表示不修改对象的可观察状态：
>
> ```cpp
> class TensorView {
> public:
>     std::size_t size() const { return size_; }
> private:
>     std::size_t size_{};
> };
> ```

### Q43. `const float* p1`、`float* const p2`、`const float* const p3` 的区别

指针有两个东西可以改：`*p = 5.0f` 改指向的值，`p = &b` 改指针本身。**看 `const` 在 `*` 的左边还是右边**：左边锁值，右边锁指针。

| 声明 | 能改指向的值 `*p = ...` | 能改指向 `p = ...` | 必须初始化 |
| --- | --- | --- | --- |
| `float* p` | 能 | 能 | 否 |
| `const float* p1` | 不能 | 能 | 否 |
| `float* const p2` | 能 | 不能 | 是 |
| `const float* const p3` | 不能 | 不能 | 是 |

**从右往左读**（`*` 读作"指针，指向"）：`const float* p1` = p1 是指针，指向 float，这个 float 是 const；`float* const p2` = p2 是 const 的指针，指向 float。`const float*` 与 `float const*` 等价。

- `const float*` 最常用，就是 C 版本的只读参数 `float sum(const float* data, std::size_t size);`，作用和 `const T&`、`std::span<const float>` 一样。
- `float* const` **很像引用**：能改值、不能改绑，编译器很多时候就这样实现引用（只是引用自动解引用）；`const float&` 对应 `const float* const`。
- 和 `const T&` 一样，`const float* p1` 只限制"不能**通过 p1** 改"，`a = 9.0f` 后 `*p1` 是 9。

### Q44. 原文"成员函数末尾的 `const` 表示不修改对象的可观察状态"，以及 `size_{}`

`std::size_t size() const { return size_; }` 末尾的 const **承诺这个函数不会修改成员变量**，编译器强制检查（`void reset() const { size_ = 0; }` 编译错误）。原理：每个成员函数有隐藏参数 `this`；普通成员函数里是 `TensorView*`，加末尾 const 后变成 `const TensorView*`——即 Q43 的"指向 const 的指针"。

**最重要的作用：const 对象只能调 const 函数。** `void inspect(const TensorView& t) { t.size(); }` 只有 `size()` 标了 const 才能调用，否则编译器无法保证通过 t 不改对象。写类时忘了给 `size()` 加 const，别人用 `const TensorView&` 接收你的对象时连大小都查不了。规律：**只读的成员函数都应加 const**（vector 的 `size()`、`empty()` 都是）。

"可观察状态"不是"一个比特都不能改"：缓存、锁这类外部看不到的内部数据可标 `mutable`，在 const 函数里也能改（如 `norm()` 第一次计算后缓存结果），了解即可。Python 没有对应机制，方法里随时能改 `self.xxx`。

`std::size_t size_{};`：声明成员的同时给**默认初始值**，空花括号叫值初始化，数字变 0、指针变 nullptr、bool 变 false。为什么要写：C++ 内置类型成员不初始化值是**随机的**（内存残留）。`TensorView` 叫"视图"，和 span 一样不拥有数据，完整类还会有 `float* data_`，同 PyTorch 的 `.view()`。

### Q45. `void inspect(const TensorView& t)` 为什么要加 `&`、为什么要加 `const`？（含语法层面）

- **加 `&` 避免复制**：不加就按值传，复制整个对象；有些类型根本不能复制（如 File），按值传直接编译不过。
- **加 `const` 保证只读**：对调用者是承诺（看签名就知道不会被改坏）；对作者是保护（误改会编译报错）；调用更灵活（`const T&` 能接收 const 对象和临时对象，`T&` 不行）。

**语法层面**：参数声明和变量声明是同一种语法，`const TensorView& t` = `类型 名字`；`&` 在声明里、类型后面，是类型的一部分（空格位置不影响）；**传参就是"用实参初始化形参"**，调用 `inspect(x)` 等价于在函数开头执行一句 `const TensorView& t = x;`——引用绑定，和 `int& ref = value;` 一模一样。

| 函数参数写法 | 调用 `inspect(x)` 时等价于 | 结果 |
| --- | --- | --- |
| `TensorView t` | `TensorView t = x;` | 新对象，调用拷贝构造 |
| `TensorView& t` | `TensorView& t = x;` | t 是 x 的别名 |
| `const TensorView& t` | `const TensorView& t = x;` | t 是 x 的只读别名 |
| `TensorView* t` | `TensorView* t = &x;`（调用写 `inspect(&x)`） | t 存 x 的地址 |

引用参数在调用处看不出来，写法和按值传一样，区别只在声明。补充：TensorView、span 这种视图类型只有一个指针和一个大小，直接按值传也很常见。经验：**大对象、拥有数据的对象**（vector、string、File）只读时用 `const T&`；**小对象、视图、内置类型**（int、float、span、TensorView）直接按值传。

### Q46. `const TensorView& t` 和 `const TensorView* t`，const 的作用区别？

两者的 const **作用完全一样**：都修饰 TensorView，不能通过 t 修改对象，都只能调 const 成员函数。区别在引用和指针本身：

|  | `const TensorView& t` | `const TensorView* t` |
| --- | --- | --- |
| 能否通过 t 修改对象 | 不能（const 的作用） | 不能（const 的作用） |
| t 能否改指向 / 改绑 | 不能（引用特性） | 能（除非再加 `* const`） |
| 能否为空 | 不能 | 能，用前要检查 nullptr |
| 访问成员 | `t.size()` | `t->size()` |
| 调用写法 | `f1(x)` | `f2(&x)` |

可以把 `const TensorView&` 理解成 `const TensorView* const` 加上"保证非空"和"自动解引用"。经验：**参数一定有值就用 `const T&`，可能没有值才用 `const T*`**——正是[原文 4.1](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#41-值地址指针与引用) 那张表的接口意图。

## 系列目录

- [导读与易错点速查](/posts/ai-infra/ch1-00-guide/)
- [（一）最小工具箱与 Python 对象模型](/posts/ai-infra/ch1-01-python-objects/)
- [（二）作用域、类与生成器](/posts/ai-infra/ch1-02-scope-class-generator/)
- [（三）装饰器、上下文管理器与异常处理](/posts/ai-infra/ch1-03-decorator-context-exception/)
- [（四）Python 并发与性能分析](/posts/ai-infra/ch1-04-concurrency-profiling/)
- **（五）C/C++ 内存、生命周期与资源管理**（本篇）
- [（六）C++ 编译、链接、模板与 CMake](/posts/ai-infra/ch1-06-cpp-build/)
- [（七）Linux 开发基本功与环境隔离](/posts/ai-infra/ch1-07-linux-env/)

