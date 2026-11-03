---
title: "AI Infra 第1章笔记：导读与易错点速查"
published: 2026-11-03T12:00:00
description: "学习 AIInfraGuide 第1章时提出的 159 个问题的系列笔记导读，附易错点速查表和自测清单。"
tags: ["AI Infra", "Python", "C++", "Linux"]
category: "AI Infra 学习笔记"
draft: false
---

这个系列是我学习 [AIInfraGuide 第1章：编程语言基础](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/) 时的提问笔记：5 次对话里一共问了 159 个问题，按原文小节重新归位，分成 7 篇。

> [!NOTE]
> 原文：[AIInfraGuide · 第1章：编程语言基础](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/)，作者 [caomaolufei](https://github.com/caomaolufei/AIInfraGuide)，MIT 许可。每篇里每一节开头的引用块摘自原文，并附有跳转到原文对应小节的链接；问答部分是我自己的提问和整理后的解答。

## 怎么读

- **问题标题**尽量保留当时的提问原话，方便回忆“我当时卡在哪”。
- **“原文 x.y”** 指向原教程的小节，点击即可跳到原文对应位置。
- **“第N篇 Qm”** 指向本系列其他文章里的问题。
- 原文 [§6 Python 与 C++ 互操作](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#6-python-与-c-互操作)、[§9 Git](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#9-git-协作与问题定位方法)、[§10 综合实战](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#10-综合实战给-python-添加一个-c-算子) 暂时还没有提问，所以没有对应文章。

## 文章列表

| 文章 | 对应原文小节 | 问题数 |
| --- | --- | --- |
| [（一）最小工具箱与 Python 对象模型](/posts/ai-infra/ch1-01-python-objects/) | [1.2](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#12-开始前的最小工具箱)、[2.1](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#21-名字对象与引用)、[2.2](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#22-可变对象不可变对象与拷贝) | 12 |
| [（二）作用域、类与生成器](/posts/ai-infra/ch1-02-scope-class-generator/) | [2.3](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#23-作用域闭包与-legb)、[2.4](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#24-类数据模型与协议)、[2.5](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#25-迭代器与生成器按需生产数据) | 32 |
| [（三）装饰器、上下文管理器与异常处理](/posts/ai-infra/ch1-03-decorator-context-exception/) | [2.6](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#26-装饰器在调用边界附加行为)、[2.7](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#27-上下文管理器让资源必定释放)、[2.8](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#28-异常处理只捕获能够处理的错误)、[2.9](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#29-类型标注与数据类) | 23 |
| [（四）Python 并发与性能分析](/posts/ai-infra/ch1-04-concurrency-profiling/) | [3.1](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#31-先分清并发与并行)、[3.2](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#32-gil-到底限制了什么)、[3.3](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#33-线程共享内存也共享风险)、[3.4](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#34-多进程绕开-gil但数据搬运有成本)、[3.5](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#35-asyncio协作式-io-并发)、[3.6](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#36-正确做基准测试)、[3.7](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#37-从慢定位到具体代码)、[3.8](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#38-python-性能优化的优先级) | 26 |
| [（五）C/C++ 内存、生命周期与资源管理](/posts/ai-infra/ch1-05-cpp-memory/) | [4.1](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#41-值地址指针与引用)、[4.2](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#42-数组指针与连续内存)、[4.3](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#43-对象存储期与生命周期)、[4.4](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#44-raii把资源绑定到对象生命周期)、[4.5](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#45-智能指针与所有权)、[4.6](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#46-拷贝移动与-rule-of-zero)、[4.7](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#47-const把不可修改写进接口) | 46 |
| [（六）C++ 编译、链接、模板与 CMake](/posts/ai-infra/ch1-06-cpp-build/) | [5.1](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#51-从源代码到可执行文件)、[5.5](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#55-模板在编译期生成类型特化代码)、[5.7](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#57-最小-cmake-项目) | 8 |
| [（七）Linux 开发基本功与环境隔离](/posts/ai-infra/ch1-07-linux-env/) | 延伸（终端配置）、[7.3](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#73-权限模型)、[7.5](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#75-查看-cpu内存磁盘和网络)、[8.1](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#81-venvconda-与容器解决不同层次的问题) | 12 |

## 易错点速查

下表收集的是提问过程中**自己提出、后来被纠正的理解**，复习时最值得回看。

| 我当时的理解 | 更准确的说法 | 出处 |
| --- | --- | --- |
| `apt/snap install cmake` 提示里没写 sudo，所以不需要 sudo | 两条都要 sudo；没有 sudo 用 pip、conda 或预编译包 | [第六篇 Q8](/posts/ai-infra/ch1-06-cpp-build/#q8-报-command-cmake-not-found-but-can-be-installed-with-snap-install-cmake--apt-install-cmake所以安装-cmake-不需要-sudo-权限) |
| `conda install` 会装到 root 里 | 装进当前环境目录；base 是管理员装的才会没权限 | [第七篇 Q4](/posts/ai-infra/ch1-07-linux-env/#q4-conda-install--c-conda-forge-starship-这个会安装到-root-里面) |
| `collect_bad` 只改默认列表，传入的列表不会改 | 传入列表会被原地修改；坑在默认 `[]` 只创建一次、跨调用共享 | [第一篇 Q6](/posts/ai-infra/ch1-01-python-objects/#q6-collect_bad-的意思是只改-result-对应的列表传入的列表不会改吗) |
| 情况一里默认空列表被"覆盖"了 | 默认对象 D 没被碰，只是这次 `result` 指向了传入的列表 | [第一篇 Q7](/posts/ai-infra/ch1-01-python-objects/#q7-情况一是不是也有问题第二次调用时默认的空列表被覆盖了) |
| 浅拷贝类似 C 指针 | Python 变量本来就像指针；浅拷贝是"复制一个指针数组" | [第一篇 Q12](/posts/ai-infra/ch1-01-python-objects/#q12-所以浅拷贝类似于-c-语言的指针) |
| C 的 `b = a` 类似 Python 深拷贝 | C 是值拷贝，Python 是不拷贝只绑定；C 结构体默认拷贝其实是浅拷贝 | [第五篇 Q6](/posts/ai-infra/ch1-05-cpp-memory/#q6-所以-c-的-b--a-就类似于-python-的深拷贝) |
| Python 浅拷贝让变量指向同一个对象 | 那是赋值；浅拷贝会新建外层对象，只共享内层 | [第五篇 Q7](/posts/ai-infra/ch1-05-cpp-memory/#q7-python-的浅拷贝会让变量指向同一个对象不就类似-c-的指针吗) |
| `int& ref` 是 ref 的地址 | 声明里的 `&` 是引用类型，表达式里的 `&` 才是取地址 | [第五篇 Q2](/posts/ai-infra/ch1-05-cpp-memory/#q2-int-ref-这个不是-ref-的地址吗) |
| 函数后的括号可以理解为元组 | 括号是调用语法；只有 `*args` 才把多余位置参数收集成元组 | [第三篇 Q4](/posts/ai-infra/ch1-03-decorator-context-exception/#q4-所以函数后加括号其实可以理解为元组) |
| `with` = 先运行函数、再运行块、再返回开头 | 先运行到 `yield`，块结束后回到 `yield` 处继续，开头不会再执行 | [第三篇 Q17](/posts/ai-infra/ch1-03-decorator-context-exception/#q17-可以这样理解-with-吗先运行-with-后的函数再运行-with-下的代码结束后再返回开头) |
| 进入/离开是 `yield` 导致的，`with` 是二次执行 | 进入/离开是 `with` 的协议（`__enter__`/`__exit__`）；函数只运行一遍，被 `yield` 切成两段 | [第三篇 Q18](/posts/ai-infra/ch1-03-decorator-context-exception/#q18-进入离开不是因为-yield-导致的吗with-是不是二次执行contextmanager-的作用) |
| `perf_counter() - start` 两边是同一个东西 | `start` 存的是第一次调用的结果，`perf_counter()` 每次返回新时间 | [第三篇 Q9](/posts/ai-infra/ch1-03-decorator-context-exception/#q9-perf_counter---start-两边不是同一个东西吗) |
| 先写 finally 的内容再写 try 的内容效果一样 | 计时会变成 0；且只有 `finally` 能在原函数出错时仍打印耗时 | [第三篇 Q8](/posts/ai-infra/ch1-03-decorator-context-exception/#q8-为什么非要-tryfinally先写-finally-的内容再写-try-的内容不行吗) |
| await 结束后 a 好了也要等 b 做完 | 谁先准备好就先回到谁；只有 main 在 `gather` 处等全部完成 | [第四篇 Q19](/posts/ai-infra/ch1-04-concurrency-profiling/#q19-await-结束后-a-好了是会等-b-做完才回到-a-吗) |
| 直接调用 worker a 和 b 就能并发 | 不加 await 根本不执行；依次 await 是串行；要先提交给事件循环 | [第四篇 Q20](/posts/ai-infra/ch1-04-concurrency-profiling/#q20-为什么要用-gather直接调用-worker-a-和-b-不行吗) |
| 线程里可以用 `a`，进程不行 | 两者都能用；区别是线程共享引用、进程拿到序列化后的副本 | [第四篇 Q13](/posts/ai-infra/ch1-04-concurrency-profiling/#q13-如果用线程创建-a--50_000--4-后-map-里就能用-a-吗) |
| Linux 子进程不会重新导入文件 | 只在 `fork` 时不会；Python 3.14 起 Linux 默认 `forkserver` 也会 | [第四篇 Q12](/posts/ai-infra/ch1-04-concurrency-profiling/#q12-windows-和-macos-子进程会重新导入文件linux-不会吗) |
| `size_t i` 的 i 最大就是 4 | `size_t` 是类型，4 是调用者通过 `size` 参数传进来的 | [第五篇 Q18](/posts/ai-infra/ch1-05-cpp-memory/#q18-指针不知道它指向的数组有多长不理解size_t-i-的-i-最大不就是-4-吗) |
| File 和 Buffer 分别靠禁用复制和 make\_unique 保证唯一 | 两者是同一机制：唯一性来自 `unique_ptr` 类型里的 `= delete`，不是 `make_unique` 函数 | [第五篇 Q34](/posts/ai-infra/ch1-05-cpp-memory/#q34-file-靠类里禁用复制保证唯一性buffer-是靠-make_unique-和-make_shared-吗) |
| `operator=` 出现等号就是调用 | 等号左边是新声明的对象就调构造函数，已存在的对象才调 `operator=` | [第五篇 Q30](/posts/ai-infra/ch1-05-cpp-memory/#q30-operator-什么时候才算调用出现等号就是调用吗) |
| `= delete` 是返回错误 | 是编译期禁止，代码根本编译不过，和运行时 `throw` 不同 | [第五篇 Q32](/posts/ai-infra/ch1-05-cpp-memory/#q32--delete-是返回错误吗) |
| Buffer 可共享，所以不该交给 unique\_ptr | 唯一所有权只限制谁负责释放，不限制谁能借用；需要时可 move 成 shared\_ptr | [第五篇 Q38](/posts/ai-infra/ch1-05-cpp-memory/#q38-buffer-应该是可以共享的吧你传给了-unique_ptr但-unique_ptr-不可以共享呀) |
| `add.cpp` 又重新声明了一遍 | 那是定义，定义本身包含声明；头文件为分别编译共享接口 | [第六篇 Q1](/posts/ai-infra/ch1-06-cpp-build/#q1-原文-addh--addcpp为什么源文件又重新描述了一下声明头文件的作用是什么) |

## 自测清单

- [ ] 能画出 `mutate` 和 `rebind` 的标签图，说明为什么一个改了调用方而另一个没改
- [ ] 能解释默认可变参数为什么跨调用共享，并用 `__defaults__` 验证
- [ ] 能说出 `src[1].append(4)` 和 `src[1] = [7, 8]` 对浅拷贝的不同影响
- [ ] 能用 LEGB 解释 `bad` 返回 `[2, 2, 2]`、`good` 返回 `[0, 1, 2]`
- [ ] 能说出 `Dog.from_str("旺财,3")` 里 `cls` 是怎么被自动传入的
- [ ] 能不看资料写出带 `@wraps` 的计时装饰器和 `@contextmanager` 计时器
- [ ] 能说出 `raise ... from exc` 与不写 `from` 时报错提示的区别
- [ ] 能根据 I/O 密集、CPU 密集选择线程、进程或 asyncio，并说明 GIL 的作用范围
- [ ] 能说出 `count_primes` 例子为什么搬运成本低，以及怎么改写会变慢
- [ ] 能解释为什么父进程初始化 CUDA 后 `fork` 会报错，以及 `spawn`、`torchrun` 怎么避免
- [ ] 能说明 cProfile 中 `tottime` 和 `cumtime` 的区别，以及 GPU 异步对它的影响
- [ ] 能区分 `int*`、`int&`、`const int&`、`const int*`、`int* const`
- [ ] 能写出 `A[row][col]` 的行主序偏移，并说明 PyTorch 转置后为什么不连续
- [ ] 能解释 File 类为什么要 `= delete` 拷贝，以及改用 `unique_ptr` 成员后为什么都不用写
- [ ] 能说出 `unique_ptr`、`shared_ptr`、`weak_ptr` 各自何时释放对象
- [ ] 能说明为什么 `return values;` 比 `return std::move(values);` 更好
- [ ] 能解释头文件、`#include`、链接器在分别编译中的分工
- [ ] 能在无 sudo 的服务器上建一个独立 conda 环境，并用 `which` 确认工具来源

## 系列目录

- **导读与易错点速查**（本篇）
- [（一）最小工具箱与 Python 对象模型](/posts/ai-infra/ch1-01-python-objects/)
- [（二）作用域、类与生成器](/posts/ai-infra/ch1-02-scope-class-generator/)
- [（三）装饰器、上下文管理器与异常处理](/posts/ai-infra/ch1-03-decorator-context-exception/)
- [（四）Python 并发与性能分析](/posts/ai-infra/ch1-04-concurrency-profiling/)
- [（五）C/C++ 内存、生命周期与资源管理](/posts/ai-infra/ch1-05-cpp-memory/)
- [（六）C++ 编译、链接、模板与 CMake](/posts/ai-infra/ch1-06-cpp-build/)
- [（七）Linux 开发基本功与环境隔离](/posts/ai-infra/ch1-07-linux-env/)
