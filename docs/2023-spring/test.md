# 测试说明

## 环境配置

本实验依赖 cmake, make, g++ 构建工具，其中 g++ 需要支持 C++17 标准，编译解析器需要 flex 和 bison 工具，需先在本地环境安装这些依赖。

由于实验框架底层文件操作调用了 Unix 相关接口，故不支持 Windows 环境，Windows 用户建议使用 WSL，或使用 Virtualbox 或 VMWare 开一个 Ubuntu 虚拟机，然后参考下面的 Ubuntu 环境配置。

### Ubuntu 环境配置

建议使用 Ubuntu 22.04 及更高版本，使用 apt 安装依赖：

```bash
sudo apt install cmake make flex bison g++
```

### macOS 环境配置

使用 homebrew 安装依赖：

```bash
brew install cmake flex bison
```

### 测试是否配置好了环境

```bash
g++ -v
flex --version
bison --version
cmake --version
```

## 运行实验框架

配好环境后，按照以下步骤进行操作：

### 拉取实验仓库并重命名

使用 git clone 和 mv 命令（将 20xxxxxxxx 替换为你的学号）：

```bash
git clone git@git.tsinghua.edu.cn:dbtrain/2023/dbtrain-20xxxxxxx.git
mv dbtrain-20xxxxxxx dbtrain-lab
```

仓库重命名操作是必须的，否则测试脚本无法正常工作。

### 拉取测试仓库

在**同一文件夹**下，使用 git clone 命令拉取测试仓库。

```bash
git clone git@git.tsinghua.edu.cn:dbtrain/dbtrain-lab-test.git
```

测试仓库应与实验仓库在同一父文件夹下，目录结构应为：

```
<diretory>
├── dbtrain-lab
│   ├── src
│   ├── CMakeLists.txt
│   ├── .gitlab-ci.yml
│   ├── ...
├── dbtrain-lab-test
│   ├── lab1
│   ├── check.py
│   ├── ...
└── ...
```

新一次实验发布时，测试仓库可能会更新，开始新实验前请先通过`git pull`命令更新测试仓库。

### 编译实验框架

进入 `dbtrain-lab` 目录，运行如下命令进行编译：

```bash
cd dbtrain-lab
mkdir build && cd build
cmake ..
make -j4
```

或使用写好的编译脚本文件：

```bash
./build.sh
```

由于实验框架中部分函数需要同学们自己实现，所以编译过程中出现 `non-void function does not return a value [-Wreturn-type]` 或 `no return statement in function returning non-void [-Wreturn-type]` 的 warning 是正常的。

如果环境配置没有问题，应成功编译出可执行程序 `cli` ：

```
...
[ 94%] Linking CXX static library ../lib/libthdb.a
[ 94%] Built target thdb
Scanning dependencies of target cli
[ 97%] Building CXX object src/CMakeFiles/cli.dir/cli.cpp.o
[100%] Linking CXX executable ../bin/cli
[100%] Built target cli
```

运行 `./bin/cli` 即可进入数据库交互界面：

```
./bin/cli
dbtrain> show databases;
+----------+
| Database |
+----------+

dbtrain> create database test;
+---------+
| SUCCESS |
+---------+

dbtrain> show databases;
+----------+
| Database |
+----------+
| test     |
+----------+
(1 rows)

dbtrain>
```

通过 ctrl+c 或 ctrl+d 即可退出数据库交互界面。

数据库交互程序支持 `-s` 参数，该参数用于控制结果打印格式，加上该参数后将只打印必要字段，不打印表格框：

```
./bin/cli -s
dbtrain> use test;
SUCCESS

dbtrain> create table t(id int, name char(4), score float);
SUCCESS

dbtrain> desc t;
Name | Type | Length
id | INT | 4
name | STRING | 4
score | FLOAT | 8
```

该参数主要用于测试脚本进行结果比对，你在本地与数据库交互时无需使用该参数。

## 测试脚本使用方法

测试仓库目录结构如下：

```
dbtrain-lab-test
├── README.md
├── check.py
├── lab1
│   ├── result
│   │   ├── 00_setup.result
│   │   ├── 10_basic.result
│   │   ├── 20_error.result
│   │   ├── 30_long_text.result
│   │   └── 40_many_rows.result
│   └── test
│       ├── 00_setup.sql
│       ├── 10_basic.sql
│       ├── 20_error.sql
│       ├── 30_long_text.sql
│       └── 40_many_rows.sql
└── test.sh
```

`test.sh`是为 CI 准备的脚本，本地测试不会用到它。

`check.py` 为测试脚本，在第 n 次实验中，该脚本会通过 `../dbtrain-lab/build/bin/cli -s` 命令运行数据库，枚举 lab{1..n}/test 目录下的所有文件，将这些文件按照序号从小到大的顺序依次输入数据库。同时脚本会在 labx 文件夹下建立 tmp 文件夹，将数据库的标准输出重定向到 tmp 文件夹下的文件中，最后将 tmp 文件夹的文件与 result 文件夹的文件内容进行对比，文件内容一致即通过测试。

由于标准错误 stderr 不会被重定向到文件中，因此你可以在实验代码中使用 cerr 输出调试信息，cerr 输出的信息不会影响测试结果。

你可以通过 `-l` 或 `--lab` 参数控制脚本进行前几次 lab 的测试，如 `python3 check.py -l 3` 则会运行 lab1, lab2 和 lab3 的测试。

脚本默认会运行 test 目录下相应 lab 的所有文件，你可以通过 `-u` 或 `--until` 参数控制最后一次 lab 中脚本运行的文件，如 `python3 check.py -l 2 -u 10` 将只会运行 lab 2 的 00 和 10 两个测试文件（以及 lab 1 的所有测试文件）。

### result 文件格式说明

result 文件的每个结果对应 sql 文件中一条 SQL 的期望输出，两个结果之间用一个空行隔开，一个典型的结果格式如下：

```
-- 10.show tables;
Tables
t_basic
t_basic_2
```

第一行表示当前运行的是第几条 SQL 及对应的 SQL 语句，仅用于增加文件可读性，测试脚本比对结果时会去掉该行。

接下来的几行表示期望输出结果，SQL 对输出结果顺序没有要求，脚本会先将结果排序后再进行比对，因此如果你的数据库输出如下：

```
Tables
t_basic_2
t_basic
```

也可以通过该条测试。

### 测试脚本输出

测试通过时，脚本会输出：

```
Test 00_setup PASSED
Test 10_basic PASSED
Test 20_error PASSED
Test 30_long_text PASSED
Test 40_many_rows PASSED
5 / 5 cases PASSED
```

测试失败分为如下几种情况：

1.结果数目错误：

```
Incorrect number of results
Expected:
26
Got:
23
Test 00_setup FAILED
```

测试 00_setup 中共 25 条 SQL，期望输出 26 个结果（包括退出数据库时输出的 Bye），但实际只输出了 23 个结果，可查看是否有某些 SQL 运行失败导致没有输出结果。

2.结果行数错误：

```
SQL 20
Incorrect length
Expected:
3
Got:
1
Test 30_long_text FAILED
```

测试 30_long_text 的第 20 条 SQL 期望输出 3 行，实际输出 1 行。

3.结果错误：

```
SQL 14
Incorrect result
Expected:
age | INT | 4
Got:
age | INT | 12884901892
Test 00_setup FAILED
```

测试 00_setup 的第 14 条 SQL 输出结果与期望输出不一致，可在 result 文件中查看第 14 条 SQL 及对应输出。

4.异常退出：

```
Traceback (most recent call last):
  File "check.py", line 105, in main
    if test_case.check():
  File "check.py", line 46, in check
    tmp_results = tmp_result_file.read().splitlines()
  File "/usr/lib/python3.8/codecs.py", line 322, in decode
    (result, consumed) = self._buffer_decode(data, self.errors, final)
UnicodeDecodeError: 'utf-8' codec can't decode byte 0x96 in position 347: invalid start byte
Test 40_many_rows FAILED
```

脚本运行错误，具体原因需根据报错信息推测，本示例中是由于输出文本中包含异常字符导致运行错误。
