---
title: "AI Infra 第1章笔记（六）：C++ 编译、链接、模板与 CMake"
published: 2026-11-03T06:00:00
description: "主线：C++ 一个文件一个文件地编译，所以需要头文件共享声明、链接器把调用和定义连起来 → 模板是\"类型的形参\"，在编译期为每种类型生成代码 → CMake 不编译代码，而是生成构建文件。"
tags: ["AI Infra", "C++", "CMake"]
category: "AI Infra 学习笔记"
draft: false
---

主线：**C++ 一个文件一个文件地编译，所以需要头文件共享声明、链接器把调用和定义连起来 → 模板是"类型的形参"，在编译期为每种类型生成代码 → CMake 不编译代码，而是生成构建文件。**

> [!NOTE]
> 本篇对应原文 [AIInfraGuide · 第1章：编程语言基础](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/) 的 §5（作者 [caomaolufei](https://github.com/caomaolufei/AIInfraGuide)，MIT 许可）。每节开头的引用块摘自原文，下面的问答是我学习时提出的问题和整理后的解答；文中的“原文 x.y”都可以点击跳转到原文对应小节。

## 5.1 从源代码到可执行文件

> **📖 原文 [5.1 从源代码到可执行文件](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#51-从源代码到可执行文件)**
>
> ```text
> 源文件 .cpp
>   └─ 预处理：展开 #include、宏和条件编译
>       └─ 编译：语法/类型检查并生成汇编
>           └─ 汇编：生成目标文件 .o
>               └─ 链接：解析符号并生成可执行文件或库
> ```
>
> 可以逐步观察：
>
> ```bash
> g++ -E main.cpp -o main.ii       # 只预处理
> g++ -S main.ii -o main.s         # 生成汇编
> g++ -c main.cpp -o main.o        # 生成目标文件
> g++ main.o -o app                # 链接
> ```

### Q1. 原文 `add.h` / `add.cpp`：为什么源文件又重新描述了一下声明？头文件的作用是什么？

（`add.h` 末尾的 `Copy` 是网页复制按钮的文字。）

**源文件里那行不是"又声明了一遍"，而是定义。** `int add(int lhs, int rhs);` 是声明（"有这么个函数，长这样"）；`int add(int lhs, int rhs) { return lhs + rhs; }` 是定义（"具体怎么做"）。定义必须写出完整签名再跟函数体，**一个定义本身也包含了声明**。菜单上写着"宫保鸡丁"是声明，厨房里这道菜的做法是定义，做法里当然要写菜名。

**为什么需要头文件：C++ 是一个文件一个文件编译的。** `g++ main.cpp add.cpp -o app` 时，编译 `main.cpp` 的编译器**看不到** `add.cpp`。它遇到 `add(2, 3)` 时要知道 add 存在吗、几个参数、什么类型、返回什么——靠声明。函数体在哪，编译完之后由**链接器**连起来：

```text
main.cpp ──编译──> main.o（"调用了 add，地址待定"）
add.cpp  ──编译──> add.o （add 的机器码）
main.o + add.o ──链接──> app
```

这正是[原文 5.1](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#51-从源代码到可执行文件) 的"预处理 → 编译 → 汇编 → 链接"流程，也解释了[原文 5.2](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#52-编译错误链接错误与运行时装载错误) 的 `undefined reference`（只声明未定义、漏链接库）属于**链接错误**而不是编译错误。

**`#include` 就是复制粘贴**：把 `add.h` 的内容原样复制到这一行（原文"预处理：展开 #include"，可以用 `g++ -E` 看到）。为什么不在每个 .cpp 手写声明？10 个文件就要写 10 遍，改了参数要改 10 处。头文件的作用：**集中存放"接口"，供所有需要的文件共享。**

**`add.cpp` 自己也 include `add.h`**：让编译器检查声明和定义是否一致，问题更早暴露。规范：每个 `.cpp` 先 include 自己的 `.h`。

**`#pragma once`**：一个头文件被间接 include 两次时内容会重复，它表示"同一次编译里只粘贴一次"。

**为什么不全写在头文件里**：两个 .cpp 都 include 含定义的头文件，链接时出现两份 add，报 `multiple definition`（普通函数整个程序只能定义一次；`inline` 和模板是例外，见 5.5）。分开写的好处：

- **编译更快**：只改 `add.cpp` 只需重编它
- **隐藏实现**：只看头文件就知道怎么用
- **发布库**：LibTorch、CUDA 给你一堆 `.h` 加 `.so`/`.lib`，include 头文件就能调用（[原文 5.3](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#53-静态库与动态库) 动态库）

Python 的 `import` 直接读整个模块，不需要单独声明。头文件机制是为了支持**分别编译**，从 C 继承；C++20 引入模块（`import`）想逐步取代。

## 5.5 模板：在编译期生成类型特化代码

> **📖 原文 [5.5 模板：在编译期生成类型特化代码](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#55-模板在编译期生成类型特化代码)**
>
> ```cpp
> template <typename T>
> T add(T lhs, T rhs) {
>     return lhs + rhs;
> }
>
> auto i = add(1, 2);        // 实例化 add<int>
> auto f = add(1.0f, 2.0f);  // 实例化 add<float>
> ```
>
> 模板定义通常要放在头文件中，因为编译器在使用点需要看到完整定义才能实例化。CUDA 和高性能计算代码大量使用模板来表达：
>
> - 数据类型：`float`、`half`、`bfloat16`；
> - tile 大小和线程块配置；
> - 布局与转置选项；
> - 编译期分支和特化实现。

### Q2. `std::vector<float>` 里的 `<>` 是什么含义？

`<>` 里写的是**模板参数**，意思是"这个容器（或函数）要用在什么类型上"。`std::vector` 本身不是完整类型，是"动态数组的通用设计图"，尖括号把空填上：`std::vector<float>`、`std::vector<int>`、`std::vector<Buffer>`。

和 Python 类型注解几乎一样：`values: list[float]`、`table: dict[str, int]` 对应 `std::vector<float> values;`、`std::map<std::string, int> table;`。区别：Python 的 `list[float]` 只是提示，C++ 的 `vector<float>` 是强制的，只能放 float。

为什么这样设计：没有模板就得写 `FloatVector`、`IntVector`……模板只写一份通用代码。编译器看到 `vector<float>` 时根据设计图**专门生成一个装 float 的类**，所以 `vector<float>` 和 `vector<int>` 是两个完全不同的类型（原文"在编译期生成类型特化代码"）。

见过的尖括号都是同一个东西：`std::unique_ptr<Buffer>`、`std::span<float>`、`std::make_unique<Buffer>(1024)`（函数也可以是模板）、`std::unique_ptr<std::FILE, FileCloser>`（两个参数）、`std::array<float, 4>`（参数也可以是数字，即原文的"非类型模板参数"）。别混淆：`#include <vector>` 是头文件名，`cout << x` 是输出运算符，`a < b` 是比较。

### Q3. 创建数组时不是写 `a[10]` 吗，这里却没有 `<>`？

`float a[10]` 是 C/C++ **语言内置**的数组语法（类型在前、大小在 `[]`），不是模板。`vector` 是标准库用模板写的类，按类的方式用：类型在 `<>`，大小作为构造函数参数在 `()`。

|  | 元素类型写在 | 大小写在 | 大小能否改变 |
| --- | --- | --- | --- |
| `float a[10]` | 最前面 | `[]` | 不能 |
| `std::vector<float> v(10)` | `<>` | `()` | 能 |
| `std::array<float, 10>` | `<>` | `<>` | 不能 |

`std::array` 把大小也放进 `<>`，是因为大小必须编译期确定，作为模板参数；vector 大小运行时可变，作为普通参数。访问元素三者都用 `[]`。

两个坑：

- `std::vector<float> v[10];` 能编译，但意思是**10 个空 vector 组成的原生数组**。
- 圆括号和花括号不同：`v1(10)` 是 10 个 0；`v2{10}` 是 1 个元素值为 10；`v3(10, 1.0f)` 是 10 个 1.0；`v4{10, 1.0f}` 是 2 个元素。花括号是"初始值列表"，创建指定大小用圆括号。

### Q4. 类的方法调用时有时也没有括号呀（`class File { public: private: };`）？

`public:` / `private:` 是**访问控制标签**，给下面的成员分区，后面跟冒号，不是函数。（补全：`private` 后要冒号，`}` 后要分号。）`class File { ... };` 是在**定义**类，不是调用。

| 写法 | 是什么 | 有没有括号 |
| --- | --- | --- |
| `public:` / `private:` | 访问控制标签 | 没有，用冒号 |
| `class File { ... };` | 定义类 | 没有 |
| `v.size()` | 调用方法 | 必须有（`v.size` 不是调用，同 Python） |
| `p.x` | 访问成员变量 | 没有 |
| `std::vector<float> v;` | 调用默认构造函数 | 可以不写 |
| `std::vector<float> v(10);` | 带参数调用构造函数 | 有 |

著名的坑：`std::vector<float> v();` 会被理解成**声明了一个函数** v（无参数、返回 vector），因为 `类型 名字()` 和函数声明一模一样。没参数时写 `v;` 或 `v{};`。

### Q5. 我的意思是 File 没有 `<>`，vector 为什么有？

**File 是普通类，vector 是模板类。** File 的成员都是具体类型，定义时就全确定了，直接 `File f("data.bin");`。vector 在标准库里大致是：

```cpp
template <typename T>
class vector {
    T* data_;           // T 是什么？定义时还不知道
    std::size_t size_;
};
```

**有没有 `<>`，取决于这个类定义时开头有没有 `template <...>`。** 你也能写模板类：`template <typename T> class Box { ... };` 然后 `Box<int> a(42); Box<std::string> b("hi");`。

C++17 起编译器能从初始值推断时可省：`std::vector v = {1.0f, 2.0f};` 推断为 `vector<float>`；模板函数也常能推断 `std::max(3, 5)`。但 `std::make_unique<Buffer>(1024)` 必须写——编译器只看到 1024，推断不出你要造 Buffer。

### Q6. `template <typename T>` 是什么？

告诉编译器：**下面这个类（或函数）里有一个叫 T 的类型，暂时不确定，使用时再填。** `template` 是关键字；`<...>` 列出模板参数；`typename T` 声明一个**类型参数**（T 是 type 首字母，惯例）。必须紧贴在类或函数定义前，只对紧跟的那一个生效。

**T 是"类型的形参"**：普通函数 `int square(int x)` 的 x 是值的占位符；模板 `template <typename T> T square(T x)` 的 T 是类型的占位符。`square<int>(3)` 时编译器把所有 T 替换成 int，生成真正的函数；`square(2.5f)` 自动推断 T = float。用到几种类型就生成几份，**编译时完成，没有运行时开销**——这也是模板定义通常要放在头文件里的原因（原文：编译器在使用点要看到完整定义才能实例化）。

```cpp
template <class T>                     // class 和 typename 在这里等价
template <typename K, typename V>      // 多个参数，如 std::map<K, V>
template <typename T, std::size_t N>   // 数字参数，如 std::array<T, N>
```

Python 最接近的是 `T = TypeVar("T")` + `class Box(Generic[T])`（[第三篇 Q2](/posts/ai-infra/ch1-03-decorator-context-exception/#q2-p--paramspecpr--typevarr-怎么理解)），但 Python 的 T 只给类型检查器看，C++ 的 T 会真的让编译器为每种类型生成专门代码。CUDA 和高性能代码大量用模板表达 dtype（`float`、`half`、`bfloat16`）和 tile 大小。

## 5.7 最小 CMake 项目

> **📖 原文 [5.7 最小 CMake 项目](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#57-最小-cmake-项目)**
>
> 目录：
>
> ```text
> vector_ops/
> ├── CMakeLists.txt
> ├── include/vector_ops.h
> └── src/vector_ops.cpp
> ```
>
> `CMakeLists.txt`：

### Q7. CMake 的作用是什么？

CMake 是**跨平台的构建系统生成器**。它本身不编译代码，而是根据 `CMakeLists.txt` **生成**真正负责编译的构建文件（Linux 上的 Makefile、Ninja 文件，Windows 上的 Visual Studio 工程），再交给 make、ninja、MSVC 调用编译器。

**为什么需要**：C/C++ 项目源文件多、依赖库多（OpenCV、CUDA、Eigen），头文件路径、链接选项、编译参数都要手动指定，不同系统和编译器写法还不一样。CMake 用一套统一描述写清"有哪些源文件、生成什么、依赖什么"。

```cmake
cmake_minimum_required(VERSION 3.18)
project(demo LANGUAGES CXX CUDA)
find_package(OpenCV REQUIRED)          # 自动查找依赖库
add_executable(demo main.cpp kernel.cu)
target_link_libraries(demo ${OpenCV_LIBS})
```

```bash
mkdir build && cd build
cmake ..        # 读 CMakeLists.txt，检查环境，生成 Makefile
make -j8        # 真正编译（或 cmake --build . -j8）
```

主要工作：`find_package` 查找依赖；管理 C++ 标准、优化级别、宏定义；组织子目录、静态库/动态库、target 之间的依赖（原文"现代 CMake 的核心是 target"）；外部构建，产物放 `build/` 不污染源码。原文用的 `cmake -S . -B build` + `cmake --build build --parallel` 是同一件事的现代写法。

**深度学习里常碰到的场景**：编译带 C++/CUDA 扩展的库（nvdiffrast、pytorch3d、自定义 CUDA kernel）；`pip install` 卡在 "Building wheel… cmake"（通常是 CMake 版本太低、找不到 CUDA、编译器版本不匹配）；用 LibTorch、TensorRT、ONNX Runtime 写 C++ 推理程序。排查方向：CMake 版本、`CUDA_HOME`、gcc 与 CUDA 是否兼容、依赖库有没有被找到。

### Q8. 报 `Command 'cmake' not found, but can be installed with: snap install cmake / apt install cmake`，所以安装 cmake 不需要 sudo 权限？

**不对，两条都需要 sudo**，提示里只是省略了：`sudo snap install cmake --classic` 或 `sudo apt install cmake`，因为要往系统目录（`/usr/bin`）里装。

没有 sudo（如实验室服务器）的装法：

1. **pip（最简单）**：`pip install cmake`，装进当前 Python 环境，`cmake --version` 即可用。
2. **conda**：`conda install -c conda-forge cmake`。
3. **官方预编译包**：下载 `cmake-x.x.x-linux-x86_64.tar.gz` 解压到自己的目录，`export PATH=$HOME/cmake-x.x.x-linux-x86_64/bin:$PATH` 写进 `~/.bashrc`（[第七篇 Q2](/posts/ai-infra/ch1-07-linux-env/#q2-解释-starship-安装命令每行的含义) 讲 PATH 和 `.bashrc`）。

如果是为了 `pip install` 某个要编译的包才装 cmake，前两种最省事，且和 Python 环境绑在一起。版本选择和环境组织见 [第七篇](/posts/ai-infra/ch1-07-linux-env/)。

## 系列目录

- [导读与易错点速查](/posts/ai-infra/ch1-00-guide/)
- [（一）最小工具箱与 Python 对象模型](/posts/ai-infra/ch1-01-python-objects/)
- [（二）作用域、类与生成器](/posts/ai-infra/ch1-02-scope-class-generator/)
- [（三）装饰器、上下文管理器与异常处理](/posts/ai-infra/ch1-03-decorator-context-exception/)
- [（四）Python 并发与性能分析](/posts/ai-infra/ch1-04-concurrency-profiling/)
- [（五）C/C++ 内存、生命周期与资源管理](/posts/ai-infra/ch1-05-cpp-memory/)
- **（六）C++ 编译、链接、模板与 CMake**（本篇）
- [（七）Linux 开发基本功与环境隔离](/posts/ai-infra/ch1-07-linux-env/)

