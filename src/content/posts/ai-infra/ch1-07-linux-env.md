---
title: "AI Infra 第1章笔记（七）：Linux 开发基本功与环境隔离"
published: 2026-11-03T05:00:00
description: "这一篇分两部分：前半是在实验室服务器（Ubuntu 24.04、无 root、base 是管理员装的 conda）上配置终端的实践，串起原文 7.1 文本流与管道、7.3 权限、7.5 资源查看、7.6 环境变量与 …"
tags: ["AI Infra", "Linux", "conda"]
category: "AI Infra 学习笔记"
draft: false
---

这一篇分两部分：前半是在实验室服务器（Ubuntu 24.04、无 root、base 是管理员装的 conda）上配置终端的实践，串起[原文 7.1](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#71-文件目录与文本流) 文本流与管道、7.3 权限、7.5 资源查看、7.6 环境变量与 PATH；后半是[原文 8.1](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#81-venvconda-与容器解决不同层次的问题) 的环境隔离在没有 sudo 的服务器上的具体做法。

> [!NOTE]
> 本篇对应原文 [AIInfraGuide · 第1章：编程语言基础](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/) 的 §7–8（作者 [caomaolufei](https://github.com/caomaolufei/AIInfraGuide)，MIT 许可）。每节开头的引用块摘自原文，下面的问答是我学习时提出的问题和整理后的解答；文中的“原文 x.y”都可以点击跳转到原文对应小节。

## 延伸：在没有 root 的服务器上配置终端

原文没有专门讲终端配置，这一组是装 Starship、ble.sh 时遇到的问题，主要用到[原文 7.1](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#71-文件目录与文本流) 的管道和 7.6 的 PATH。

### Q1. 在实验室服务器（base 环境）能装类似 oh-my-zsh 这种有颜色提示的东西吗？

可以，**不需要 root**，都装在自己的 home 目录，不影响别人。两种思路：

**方案一：留在 bash，只换提示符（推荐）**。用 **Starship**：单文件程序，自带颜色、git 分支、conda 环境名、命令耗时，bash/zsh 都能用。

```bash
mkdir -p ~/.local/bin
curl -sS https://starship.rs/install.sh | sh -s -- -b ~/.local/bin
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(starship init bash)"' >> ~/.bashrc
source ~/.bashrc
```

**方案二：真正用上 zsh + oh-my-zsh**。`which zsh` 检查；没有就 `conda install -c conda-forge zsh`（base 没写权限就单独建环境）；国内用 gitee 镜像安装 oh-my-zsh；`conda init zsh`；没有 root 一般不能 `chsh`，在 `~/.bashrc` 末尾加：

```bash
if [[ $- == *i* ]] && command -v zsh >/dev/null; then exec zsh; fi
```

**这一行必须放在 `[[ $- == *i* ]]` 判断里**（只在交互式终端切换），否则 VS Code Remote、scp 会连接失败。推荐插件 zsh-syntax-highlighting、zsh-autosuggestions。动 `.bashrc` 前先 `cp ~/.bashrc ~/.bashrc.bak`。只想要颜色和 git/conda 信息，方案一最不容易出问题。

### Q2. 解释 Starship 安装命令每行的含义

- `mkdir -p ~/.local/bin`：创建个人程序目录；`-p` 上级目录不存在一并创建、已存在不报错（[原文 7.1](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#71-文件目录与文本流) 的 `mkdir -p runs/exp01/logs`）。
- `curl -sS https://starship.rs/install.sh | sh -s -- -b ~/.local/bin`：用**管道** `|` 连接两部分（[原文 7.1](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#71-文件目录与文本流)"管道把左侧标准输出连接到右侧标准输入"）。`curl -sS` 下载安装脚本，`-s` 静默无进度条、`-S` 出错仍显示错误；`sh -s` 让 sh 从管道读脚本；`--` 表示后面是传给脚本的参数而不是给 sh 的；`-b ~/.local/bin` 指定装到自己目录（默认 `/usr/local/bin` 要 sudo）。
- `echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc`：`>>` 是**追加**，写成 `>` 会**覆盖整个文件**，千万别写错。配置作用是把 `~/.local/bin` 加到 PATH 最前面，输入 `starship` 时系统能找到。`~/.bashrc` 是每次打开 bash 终端都自动执行的配置文件。注意这里用**单引号**，`$HOME`、`$PATH` 不会在写入时展开，而是在每次 `.bashrc` 执行时才展开（[原文 7.2](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#72-引号变量与退出码)"单引号不展开变量"）。
- `echo 'eval "$(starship init bash)"' >> ~/.bashrc`：`starship init bash` 输出一段 bash 代码，`eval` 执行它，替换提示符样式。
- `source ~/.bashrc`：让当前终端立即重新加载配置。[原文 7.6](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#76-环境变量与动态库路径) 的提醒同理：改 `.bashrc` 不会改变已经运行的终端或 IDE，要重新加载或重启。

`echo ... >>` 两行只需执行一次，重复执行会出现重复行，用 `tail ~/.bashrc` 确认。

### Q3. 卡在 `> Installing Starship, please wait…`，不知道进度

基本是**从 GitHub 下载太慢或连不上**，脚本用静默模式所以看不到进度。先 `Ctrl + C` 中断，任选：

1. **conda 装**：`conda install -c conda-forge starship`（conda-forge 有国内镜像），装好后只需写 `eval` 那一行。
2. **手动下载看进度**：`cd ~/.local/bin && curl -L -o starship.tar.gz <Tarball URL>`，去掉 `-s` 显示进度条，`-L` 跟随 GitHub 跳转，再 `tar -xzf starship.tar.gz && rm starship.tar.gz`。
3. **本地下载再传上去**：本地浏览器下载后 `scp starship-*.tar.gz cyx@服务器地址:~/.local/bin/`，再到服务器解压（[原文 7.7](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#77-ssh-与文件传输) 文件传输）。

装完 `starship --version` 检查。

### Q4. `conda install -c conda-forge starship` 这个会安装到 root 里面？

**不一定，取决于 conda 装在哪里。** `conda install` 装进**当前环境的目录**，不是 root 用户的目录。先 `conda info --base`：

- 输出 `/home/cyx/miniconda3` 这类：conda 是你自己装的，装到你的 home 下，放心。
- 输出 `/opt/anaconda3`、`/usr/local/miniconda3` 这类系统路径：base 是管理员装的、全实验室共用。要么没写权限报 `Permission denied`，要么有写权限但会影响其他人，不建议。替代：建自己的环境 `conda create -n tools -c conda-forge starship`（放在 `~/.conda/envs/`），在 `.bashrc` 里用完整路径 `eval "$(~/.conda/envs/tools/bin/starship init bash)"`，不需激活；或用 Q3 方法二、三手动下载。

### Q5. 装好了 Starship，想让写命令行时有提示和补全，怎么做？

bash 里最接近 oh-my-zsh"灰色历史提示 + 语法高亮"的是 **ble.sh**，也装在自己目录，和 Starship 兼容。

**方式 A：打包版（推荐，不需编译）**：下载 `ble-nightly.tar.xz` → `tar xJf` → `bash ble-nightly/ble.sh --install ~/.local/share` → 清理。GitHub 慢同样可以本地下载再 scp。**方式 B：git 编译**（需要 git、make、gawk）：`git clone --recursive ... && make -C ble.sh install PREFIX=~/.local`。

修改 `~/.bashrc`（先备份），**位置很重要**：

```bash
[[ $- == *i* ]] && source -- ~/.local/share/blesh/ble.sh --attach=none   ← 文件最开头
...原有内容...
# >>> conda initialize >>> ... <<< conda initialize <<<
export PATH="$HOME/.local/bin:$PATH"
eval "$(starship init bash)"
[[ ! ${BLE_VERSION-} ]] || ble-attach                                    ← 文件最末尾
```

效果：灰色历史提示（`→` 或 `End` 接受整条、`Alt + F` 接受下一个单词）；命令存在正常颜色、拼错红色；`Tab` 弹出候选菜单；`Ctrl + R` 搜索历史。`git che<Tab>` 这类参数补全靠 bash-completion，`ls /usr/share/bash-completion/bash_completion` 存在即可。出问题 `cp ~/.bashrc.bak ~/.bashrc` 恢复。

### Q6. 我已经安装好了 Starship（为什么还要装别的）？

两个工具负责不同的事：

| 工具 | 负责什么 |
| --- | --- |
| Starship | 只负责**提示符的样式**：`cyx in 🌐 amax in ~/.local/bin ❯` 这一行的颜色、目录、git 分支、conda 环境 |
| ble.sh | 负责**输入命令时的体验**：灰色历史提示、语法高亮、Tab 补全菜单 |

Starship 本身不提供命令提示和补全，两者同时用互不冲突。

### Q7. bash 和 zsh 的区别

两者都是 **shell**（终端里负责解释和执行命令的程序），基础语法大部分相同，`ls`、`cd`、`conda activate`、`python train.py` 用法完全一样。

| 方面 | bash | zsh |
| --- | --- | --- |
| 普及程度 | 绝大多数 Linux 服务器的默认 shell | macOS 默认；Linux 上通常需另装 |
| 原生交互体验 | 朴素，Tab 补全较基础 | 更强，菜单选择、大小写模糊匹配 |
| 插件生态 | 较少（ble.sh、oh-my-bash） | 非常丰富（oh-my-zsh） |
| 写脚本 | 事实标准 | 能跑大部分 bash 脚本，细节不同 |
| 配置文件 | `~/.bashrc` | `~/.zshrc` |

几个语法差异：数组下标 bash 从 0、zsh 默认从 1；通配符匹配不到时 bash 原样传下去、zsh 报 `no matches found`（所以 zsh 里 `pip install "pkg[extra]"` 要加引号）；zsh 里 `$var` 默认不按空格拆分，bash 会（[原文 7.2](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#72-引号变量与退出码)"未加引号的变量可能发生单词分割"，bash 脚本里几乎总应写 `"$variable"`）。

建议：服务器上 **bash + Starship + ble.sh** 已接近 zsh + oh-my-zsh 体验，不用处理 chsh、conda 初始化、VS Code Remote 兼容；Mac 上直接 oh-my-zsh；写脚本开头 `#!/bin/bash`、用 `bash xxx.sh` 运行。

### Q8. 在 Linux 上 bash 更兼容是吗？

是的：几乎所有发行版预装 bash 并作为默认 shell；网上教程和脚本（安装脚本、Slurm 作业脚本、项目 `run.sh`）基本按 bash 写；`conda init`、VS Code Remote 等工具默认适配 bash；换台服务器配置几乎不用调。

两点说明："兼容"主要指写脚本和运行环境，zsh 作为交互 shell 在 Linux 上一样稳定；**`/bin/sh` 不一定是 bash**——Ubuntu 上指向 **dash**（更精简），`#!/bin/sh` 或 `sh xxx.sh` 运行时 bash 特有语法（`[[ ]]`、数组）可能报错，应改用 `#!/bin/bash` 或 `bash xxx.sh`，可用 `ls -l /bin/sh` 确认。（[原文 7.2](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#72-引号变量与退出码) 推荐的 `#!/usr/bin/env bash` + `set -euo pipefail` 也是 bash 写法。）

## 7.3 权限模型

> **📖 原文 [7.3 权限模型](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#73-权限模型)**
>
> ```bash
> ls -l script.sh
> # -rwxr-x---  owner group ... script.sh
>
> chmod u+x script.sh          # 给所有者增加执行权限
> chmod 640 config.yaml        # owner: rw, group: r, other: none
> ```
>
> `r/w/x` 对文件分别表示读、写、执行；对目录则表示列出名称、修改目录项、穿越目录。权限问题先用 `ls -l`、`id` 和父目录权限定位，不要习惯性 `chmod 777`。

### Q9. `chmod +x` 的含义

给文件加上"可执行"权限，让它能作为程序直接运行。`chmod` = change mode；`+` 添加（`-` 去掉）；`x` = execute（另外两种 `r` 读、`w` 写）。没有执行权限运行会报 `Permission denied`。

`ls -l ~/.local/bin/starship` 输出 `-rwxr-xr-x 1 cyx cyx 9.8M ... starship`，那串分三组：

| 位置 | 对象 | 示例 | 含义 |
| --- | --- | --- | --- |
| 第 1 组 | 文件所有者 | `rwx` | 可读、可写、可执行 |
| 第 2 组 | 同组用户 | `r-x` | 可读、可执行 |
| 第 3 组 | 其他人 | `r-x` | 可读、可执行 |

`-` 表示没有该权限。`chmod +x` 默认给所有人加，只给自己加写 `chmod u+x 文件名`（u = user，[原文 7.3](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#73-权限模型) 的 `chmod u+x script.sh`）。`tar` 解压出来的 starship 一般已带执行权限，`ls -l` 看到有 `x` 就不用再加。[原文 7.3](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#73-权限模型) 还提醒：权限问题先用 `ls -l`、`id` 和父目录权限定位，不要习惯性 `chmod 777`。

## 7.5 查看 CPU、内存、磁盘和网络

> **📖 原文 [7.5 查看 CPU、内存、磁盘和网络](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#75-查看-cpu内存磁盘和网络)**
>
> | 问题 | 常用命令 |
> |------|----------|
> | CPU/负载 | `top`、`uptime`、`mpstat` |
> | 进程内存 | `ps`、`top`、`pmap -x PID` |
> | 系统内存 | `free -h`、`vmstat` |
> | 磁盘容量 | `df -h` |
> | 目录占用 | `du -sh PATH`、`du -h --max-depth=1` |
> | 文件 I/O | `iostat`、`iotop` |
> | 监听端口 | `ss -lntp` |
> | 连接测试 | `curl -v URL`、`nc -vz HOST PORT` |
>
> “内存不够”至少可能指主机 RAM、GPU 显存、共享内存 `/dev/shm`、进程地址空间限制或磁盘满。先确认资源种类再行动。

### Q10. `df -h` 里的 tmpfs 是什么？

**tmpfs 是存放在内存里的文件系统**：看起来和普通目录一样能读写，但数据在 RAM 里。速度很快；**重启后清空**，只放临时文件；**显示的 Size 是上限，不是预先占用**（`/run` 显示 26G 实际只用了 5.9M，写入才占内存）。

| 挂载点 | 作用 |
| --- | --- |
| `/run` | 系统运行时临时文件（PID、socket） |
| `/dev/shm` | **共享内存**，供进程间交换数据，上限 126G |
| `/run/lock` | 锁文件，防止程序重复启动 |
| `/run/user/1031` 等 | 每个已登录用户一个，数字是 UID（`id -u` 查看自己的） |

从上限推测这台机器约 **252G 内存**：Ubuntu 默认 `/dev/shm` 设为内存一半、`/run` 设为十分之一，可用 `free -h` 确认（[原文 7.5](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#75-查看-cpu内存磁盘和网络) 资源命令表）。

**和实验相关**：PyTorch DataLoader 在 `num_workers > 0` 时就是通过 `/dev/shm` 在进程间传数据（[第四篇 Q11](/posts/ai-infra/ch1-04-concurrency-profiling/#q11-count_primes--processpoolexecutor-解释一下如何体现数据搬运有成本) 的"数据搬运"在系统层面的落点）。当时已用 53G，可能有人在跑多进程训练，或有进程异常退出留下残留。遇到 `RuntimeError: DataLoader worker ... Bus error / unable to write to file </torch_xxx>` 一般就是 `/dev/shm` 不够，降低 `num_workers` 或找管理员。这也是[原文 7.5](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#75-查看-cpu内存磁盘和网络) 说的"内存不够至少可能指主机 RAM、GPU 显存、共享内存 `/dev/shm`……先确认资源种类再行动"。

（顺带：当时在编译 gawk 是为 ble.sh 方式 B，用方式 A 打包版就不需要 gawk 和 make。）

## 8.1 `venv`、conda 与容器解决不同层次的问题

> **📖 原文 [8.1 `venv`、conda 与容器解决不同层次的问题](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#81-venvconda-与容器解决不同层次的问题)**
>
> | 工具 | 隔离范围 | 适合场景 |
> |------|----------|----------|
> | `venv` | Python 包与解释器环境 | 纯 Python 项目、依赖简单 |
> | conda | Python + 一部分原生库与工具链 | 科学计算、复杂二进制依赖 |
> | Docker/容器 | 用户态文件系统、依赖与进程环境 | 部署、CI、跨机器复现 |
>
> `venv` 示例：
>
> ```bash
> python -m venv .venv
> source .venv/bin/activate
> python -m pip install --upgrade pip
> python -m pip install -r requirements.txt
> ```

[原文 8.1](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#81-venvconda-与容器解决不同层次的问题) 说 `venv`、conda、容器解决不同层次的问题；下面两个问题是"在没有 sudo、base 由管理员安装的服务器上，怎么搭一个以后能扩展的环境"的具体答案。

### Q11. 当前 Python 在 `/opt/miniconda3/bin/python`，没有 sudo 也可以装 cmake 吗？为了以后的扩展性你建议怎么做？

可以，但要注意：`/opt/miniconda3` 通常是管理员装的公共 conda，**一般没有写权限**。直接在 base 里 `pip install cmake` 或 `conda install cmake` 很可能 `Permission denied`；pip 还可能自动退回 `--user` 装进 `~/.local`，以后容易乱。先确认：

```bash
ls -ld /opt/miniconda3
touch /opt/miniconda3/test_write && echo 可写 || echo 不可写
```

**建议：不动 base，建自己的 conda 环境。** 即使 `/opt/miniconda3` 不可写，也能用它创建环境，conda 会自动放到 `~/.conda/envs/` 下：

```bash
conda create -n research python=3.10 -y
conda activate research
conda install -c conda-forge cmake ninja -y
cmake --version
```

找不到 `conda activate` 就先 `conda init bash` 再重开终端。为什么扩展性最好：

- **不依赖管理员**：以后装 PyTorch、CUDA toolkit（`conda install -c nvidia cuda-toolkit`）、gcc（`conda install -c conda-forge gxx`）都不需要 sudo——这对应[原文 8.1](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#81-venvconda-与容器解决不同层次的问题) 说 conda 能隔离"Python + 一部分原生库与工具链"，而 venv 只隔离 Python 包。
- **项目隔离**：不同论文各建一个环境，依赖版本冲突时互不影响。
- **可复现**：`conda env export > environment.yml` 记录环境（原文推荐 `--from-history` 只导出直接依赖），换服务器一键重建。但原文也提醒：完整环境还包括操作系统、驱动、编译器、硬件架构，一个依赖文件不是全部环境描述。
- **cmake 和 ninja 一起装**：编译 CUDA 扩展时很多库优先用 ninja，快很多。

两个小建议：在 `~/.condarc` 里写 `envs_dirs: [~/.conda/envs]` 和 `pkgs_dirs: [~/.conda/pkgs]`，让以后的环境默认装在自己目录；家目录配额小（几十 GB）就改到数据盘，如 `/data/你的用户名/conda/envs`，避免几个 PyTorch 环境撑满配额（服务器上有 `/home/data1`–`/home/data4` 几块数据盘，见 [Q10](#q10-df--h-里的-tmpfs-是什么) 的 `df -h` 输出）。

### Q12. 建议用什么版本的 Python 和 CMake？

**推荐组合：Python 3.11 + CMake 3.31（3.x 最新）。**

```bash
conda create -n research python=3.11 -y
conda activate research
conda install -c conda-forge "cmake>=3.27,<4" ninja -y
```

**为什么 Python 3.11**：

- 深度学习生态里兼容性最稳：PyTorch、timm、transformers、open\_clip、各种 3D/渲染库基本都有 3.11 预编译包，很少需要自己编译。
- 比 3.8/3.9 快不少，也不会像 3.8 那样已停止维护。
- 3.12、3.13 也能用，但一些老研究代码和带 CUDA 扩展的库可能还没有对应 wheel，装时容易卡在编译。
- 要复现的仓库明确写了版本就照它来，可单独建环境。

**为什么 CMake 选 3.x 而不是 4.x**：CMake 4.0 起删除了对很老写法的兼容，很多研究仓库还写着 `cmake_minimum_required(VERSION 2.8)` 或 `3.1`，用 4.x 直接报 `Compatibility with CMake < 3.5 has been removed from CMake.`。限定在 3.x 最省心；必须用 4.x 时临时 `export CMAKE_POLICY_VERSION_MINIMUM=3.5`。

**装完检查**（对应[原文 8.4](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#84-诊断-gpu-环境的固定顺序)"诊断 GPU 环境的固定顺序"前两步：可执行文件来自哪里）：

```bash
python --version
cmake --version
which python cmake   # 确认都指向 ~/.conda/envs/research/ 下
```

之后装 PyTorch 时按 `nvidia-smi` 显示的驱动版本选安装命令。[原文 8.3](https://caomaolufei.github.io/AIInfraGuide/guides/%E6%A8%A1%E5%9D%97%E4%B8%80-%E5%89%8D%E7%BD%AE%E7%9F%A5%E8%AF%86/%E7%AC%AC1%E7%AB%A0-%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80/#83-gpu-软件栈四个容易混淆的部分) 的补充：`nvidia-smi` 右上角的 "CUDA Version" 是驱动**可支持的最高** CUDA 兼容级别，不证明装了同版本 `nvcc`；框架实际用的运行时要看 `torch.version.cuda`。

## 系列目录

- [导读与易错点速查](/posts/ai-infra/ch1-00-guide/)
- [（一）最小工具箱与 Python 对象模型](/posts/ai-infra/ch1-01-python-objects/)
- [（二）作用域、类与生成器](/posts/ai-infra/ch1-02-scope-class-generator/)
- [（三）装饰器、上下文管理器与异常处理](/posts/ai-infra/ch1-03-decorator-context-exception/)
- [（四）Python 并发与性能分析](/posts/ai-infra/ch1-04-concurrency-profiling/)
- [（五）C/C++ 内存、生命周期与资源管理](/posts/ai-infra/ch1-05-cpp-memory/)
- [（六）C++ 编译、链接、模板与 CMake](/posts/ai-infra/ch1-06-cpp-build/)
- **（七）Linux 开发基本功与环境隔离**（本篇）

