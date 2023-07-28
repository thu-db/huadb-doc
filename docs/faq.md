# 常见问题

**Q: 测试脚本运行 insert 正常，但在交互式命令行中复制 insert 语句运行时会卡出。**

A: 部分 insert 语句较长，在交互式命令行无法工作，可以使用输入重定向方式运行，如：

```bash
./bin/cli < /path/to/00_setup.sql
```

**Q: 测试脚本运行异常且没有输出，交互式命令行中复制 SQL 运行正常。**

A: 同上，尝试使用输入重定向方式运行 SQL。通常此类问题是由指针异常导致的，可以使用 gdb 等调试器进行调试。

**Q: parser 解析器无法正常工作，如输入正确的 SQL 语句却显示 Syntax Error，删除 build 文件夹重新编译依然无效。**

A: 可能是由于 flex 和 bison 生成的 cpp 文件出现异常，如被编辑器格式化等。可删除 parser 文件夹下的 lex.yy.cpp, sql.tab.cpp 和 sql.tab.h 文件，然后重新编译。

**Q: 编译报错: ISO C++17 does not allow 'register' storage class specifier [-Wregister]**

A: 通常是由于使用了旧版 flex 和 bison，导致生成的代码含有 clang 编译器不支持的 register 关键词，可以通过升级 flex 和 bison 版本解决。注意：mac 系统默认会使用系统自带的 flex 和 bison，通常版本较老，可以使用 homebrew 安装，并按照安装过程中的输出信息设置相应的环境变量。

**Q: 我的本地测试正常，但是 CI 无法通过测试**

A: 通常是本地编译器（如clang）与评测机编译器（g++）不同导致的，可参考文档中的评测机环境，在相同的环境下编译运行代码。
