# 数据库课程实验

## 概述

本课程实验均在 [thdb](https://github.com/thu-db/thdb) 上进行，thdb 数据库系统由 C++ 语言编写，支持基础的页面存储、故障恢复、缓存管理、查询解析、查询优化、查询处理等功能。课程提供 thdb 的基础实验框架，每次实验中，你需要在框架中填充相应的函数实现代码，使之通过该次实验的所有测例。

## 环境配置

thdb 使用了 C++17 标准，开始实验前，请确保你的开发环境支持 C++17 标准。

目前 thdb 仅支持 macOS 和 Linux 操作系统，使用 Windows 的同学建议使用 WSL 虚拟机进行实验。

thdb 代码下载与提交需要使用 [git](https://git-scm.com/) 工具，代码编译需要使用 [CMake](https://cmake.org/) 及 [Make](https://www.gnu.org/software/make/) 工具，且需要安装 [gcc](https://gcc.gnu.org/) 或 [clang](https://clang.llvm.org/) 编译器。此外，代码调试中可能会用到调试器 [gdb](https://www.sourceware.org/gdb/) 或 [lldb](https://lldb.llvm.org/)。开始实验前，请确保你的开发环境安装了这些工具并可以正常使用，如没有，请根据你使用的操作系统与发行版选择对应的命令进行环境配置：Debian/Ubuntu，

### Debian/Ubuntu

```bash
sudo apt install git g++ make cmake gdb
```

### Fedora

```bash
sudo dnf install git g++ make cmake gdb
```

### macOS

macOS 的开发者工具已包含 clang 编译器、lldb 调试器以及 git 和 Make 工具，仅需安装 CMake 即可。

首先安装 [homebrew](https://brew.sh/)，安装后运行：

```bash
brew install cmake
```
