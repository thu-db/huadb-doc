## 概述

本课程实验均在 [HuaDB](https://github.com/thu-db/huadb) 上进行，HuaDB 数据库系统由 C++ 语言编写，支持基础的页面存储、故障恢复、缓存管理、查询解析、查询优化、查询处理等功能。课程提供 HuadDB 的基础实验框架，每次实验中，你需要在框架中填充相应的函数实现代码，使之通过该次实验的所有测例。

## 环境配置

HuaDB 使用了 C++17 标准，开始实验前，请确保你的开发环境支持 C++17 标准。

目前 HuaDB 仅支持 macOS 和 Linux 操作系统，使用 Windows 的同学建议使用虚拟机进行实验。

HuaDB 代码下载与提交需要使用 [git](https://git-scm.com/) 工具，代码编译需要使用 [CMake](https://cmake.org/) 及 [Make](https://www.gnu.org/software/make/) 工具，且需要安装 [gcc](https://gcc.gnu.org/) 或 [clang](https://clang.llvm.org/) 编译器。此外，代码调试中可能会用到调试器 [gdb](https://www.sourceware.org/gdb/) 或 [lldb](https://lldb.llvm.org/)。开始实验前，请确保你的开发环境安装了这些工具并可以正常使用，如没有，请根据你使用的操作系统选择对应的命令进行环境配置。

### Debian/Ubuntu

```bash
sudo apt install git g++ make cmake gdb
```

### macOS

macOS 的开发者工具已包含 clang 编译器、lldb 调试器以及 git 和 Make 工具，仅需安装 CMake 即可。

首先安装 [homebrew](https://brew.sh/)，安装后运行：

```bash
brew install cmake
```

实验框架支持以下操作系统（发行版）版本：

1. Ubuntu 20.04 及以上
2. Debian 10 及以上
3. macOS Monterey 及以上

在其他的环境（如 Windows, WSL, BSD 等）也可能可以编译和运行，但不保证可以正常工作。

## 开始前的工作

### 克隆仓库

首先，你需要使用 git 将 HuaDB 代码仓库克隆到你的开发机上，对于选修《数据库专题训练》课程的同学，助教已经在 git 上为你创建好仓库，使用如下命令进行克隆 (将 202x 改为你的选修年份，20xxxxxxxx 改为你的学号)：

```bash
git clone https://git.tsinghua.edu.cn/dbtrain/202x/dbtrain-20xxxxxxxx.git dbtrain
```

HuaDB 代码随时可能会有更新，你需要将更新的部分同步到你的本地。为了新增一个远程仓库，在你的 dbtrain 目录下执行：

```bash
git remote add upstream https://git.tsinghua.edu.cn/dbtrain/dbtrain-lab-template.git
```

实验代码更新时，执行如下命令将更新的部分合并入你的代码：

```bash
git pull upstream master
```

合并过程中可能产生冲突，冲突时请手动解决所有冲突并执行 `git add` 和 `git commit` 指令完成合并。

无论你是否选修《数据库专题训练》课程，**请不要将你的代码放到任何公有仓库上**。

### 编译及测试

直接运行 `make` 即可完成编译 HuaDB，你也可以通过设置 `CMAKE_BUILD_PARALLEL_LEVEL` 环境变量来进行多核编译：

```bash
CMAKE_BUILD_PARALLEL_LEVEL=$(nproc) make
```

通过以上指令可以编译出 debug 版本的程序，为便于调试，我们不建议编译 release 版本。

编译生成的文件位于 `build/debug` 目录，实验过程中只需要关注 `build/debug/bin` 目录中的可执行程序即可，具体包括：

1. shell: 数据库程序，运行后可以与数据库进行交互。

2. sqllogictest: 测试程序，用于批量测试。

运行如下命令来验证你编译出的数据库程序和测试程序可以正常运行：

```bash
make lab0
```

如果产生如下输出：

```bash
Test: 1/3
lab0/10-basic.test PASS
lab0/20-expression.test PASS
lab0/30-set_and_show.test PASS
lab0/40-error.test PASS
Test: 2/3
lab0/10-basic.test PASS
lab0/20-expression.test PASS
lab0/30-set_and_show.test PASS
lab0/40-error.test PASS
Test: 3/3
lab0/10-basic.test PASS
lab0/20-expression.test PASS
lab0/30-set_and_show.test PASS
lab0/40-error.test PASS
```

表示程序正常运行。

如果程序报错，可以根据报错信息对照[测试说明](../overview)进行排查。

此外，你可以通过如下命令进入数据库交互界面：

```bash
make shell
```

你可以在交互界面中运行一些基础的 DDL 命令，如：

```bash
Welcome to HuaDB. Type "\?" or "\h" for help
huadb> \?

   \? or \h              show help message
   \c [database_name]    change database
   \d                    show tables
   \d [table_name]       describe table
   \l                    show databases
   \q                    quit

huadb> \l
+---------------+
| database_name |
+---------------+
| system        |
| huadb         |
+---------------+
(2 rows)

huadb> create table test(id int, info varchar(10));
 CREATE TABLE
huadb> \d
+------------+
| table_name |
+------------+
| test       |
+------------+
(1 row)

huadb> \d test
+------+---------+------+
| name | type    | size |
+------+---------+------+
| id   | int     | 4    |
| info | varchar | 10   |
+------+---------+------+
(2 rows)

huadb> drop table test;
 DROP TABLE
huadb> \d
+------------+
| table_name |
+------------+
(0 rows)

huadb> \q
```

这些命令的名称参考了 PostgreSQL 客户端 [psql](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMANDS) 命令的设计，如果你有过 psql 的使用经验，应该会熟悉这些命令。

在完成实验 1 之前，你的程序仅支持在默认数据库 huadb 中创建表、删除表和查看表的结构，还不支持数据库的创建/切换/删除，也不支持在表中插入和读取数据。

至此，环境配置与验证工作已经结束，你可以开始[实验](../lab1/1-intro)，或阅读[实验框架说明](../overview)。
