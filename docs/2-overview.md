## 实验要求 { #requirement }

每次实验聚焦于实验框架的 1-2 个模块，我们已为你提供了相应的类以及函数声明，你需要完成其中部分函数的实现。每次实验分为基础功能和高级功能两个部分，在基础功能中，所有需要你来完成的代码部分均标注了`// LAB n BEGIN`，n 表示第 n 次实验，你可以通过编辑器的全局搜索功能快速定位每次实验需要补充的代码的位置。高级功能需要修改的部分没有明确的标注，部分高级功能相关代码有`// LAB n ADVANCED BEGIN`标注，但并非所有高级功能都有标注，你需要根据自己的实现思路对实验框架进行修改。

在实现基础功能的过程中，你可以添加新的函数和变量，但不要修改已有函数的声明。

实现高级功能时，你可以对实验框架任意修改，建议单独开一个分支完成实验，避免影响后续实验。

## 运行 { #run }

实验代码根目录的 Makefile 文件提供了多种命令的支持，具体包括：

-   `make lab1-debug`：编译数据库（仅用于实验 1）。
-   `make` 或 `make debug`：编译数据库（debug 版本）。
-   `make release`：编译数据库（release 版本）。
-   `make shell`：进入数据库交互界面。
-   `make labn`：运行 lab 0 至 lab n 的所有测例（0 <= n <= 5）。
-   `make labn-only`：仅运行 lab n 的测例（0 <= n <= 5）。
-   `make labn/xx`：运行所有符合通配符 test/labn/xx\* 匹配的测例。
-   `make clean`：删除编译生成的文件（数据库程序相关文件）。
-   `make destroy`：删除数据库文件（数据库存储的数据文件）。

我们采用系统表管理数据库的元信息，包括如下三个表：

| 表名            | 作用                                                                                                        |
| --------------- | ----------------------------------------------------------------------------------------------------------- |
| huadb_table     | 存储所有表的 oid (object identifier)，所属数据库的 oid，表名，表的结构，以及表的行数（用于实验 5 基数估计） |
| huadb_database  | 存储数据库的 oid，以及数据库名                                                                              |
| huadb_statistic | 存储列的统计信息，包括直方图和 distinct value (不同值的个数)                                                |

在完成[实验 1 的任务 1](lab1/2-basic.md#t1) 之前，由于数据库还不支持记录的增删改查功能，系统表无法正常工作，你需要使用 `make lab1-debug` 进行编译，该编译指令通过定义 SIMPLE_CATALOG 宏，实现用 catalog 文件夹下的 simple_catalog 来替代 system_catalog，此时你的数据库将不使用系统表，使用文件来代替系统表的功能。在完成实验 1 的任务 1 之后，使用 `make` 或 `make debug` 进行编译，来使数据库具备正常的系统表功能。

**完成实验 1 的任务 1 后**，你可以进入 system 数据库中查看系统表的结构：

```
huadb=> \c system
 Change to database system
system=> \d
+-----------------+
| table_name      |
+-----------------+
| huadb_table     |
| huadb_database  |
| huadb_statistic |
+-----------------+
(3 rows)

system=> \d huadb_table
+-------------+---------+------+
| name        | type    | size |
+-------------+---------+------+
| table_oid   | uint    | 4    |
| db_oid      | uint    | 4    |
| table_name  | varchar | 32   |
| schema      | varchar | 1024 |
| cardinality | int     | 4    |
+-------------+---------+------+
(5 rows)

system=> \d huadb_database
+---------+---------+------+
| name    | type    | size |
+---------+---------+------+
| db_oid  | uint    | 4    |
| db_name | varchar | 32   |
+---------+---------+------+
(2 rows)

system=> \d huadb_statistic
+-------------+---------+------+
| name        | type    | size |
+-------------+---------+------+
| table_name  | varchar | 32   |
| column_name | varchar | 32   |
| histogram   | varchar | 1024 |
| n_distinct  | int     | 4    |
+-------------+---------+------+
(4 rows)
```

## 数据库文件结构 { #file_structure }

当通过 `make shell` 进入交互界面访问数据库时，数据库生成的文件位于 `huadb_data` 文件夹。通过 `make labn` 进行批量测试时，数据库文件位于 `huadb_test/huadb_data` 文件夹。

数据库相关文件有如下几类：

-   `init`: 空文件，存在时表示数据库已初始化。
-   `control`: 数据库控制文件，存储数据库下一个可用的 xid（事务 id）、lsn、oid，以及数据库是否正常关闭等信息。
-   `log`: 数据库日志文件。
-   `next_lsn`: 下一条日志的 lsn。
-   `master_record`: 最新一次检查点的位置。

后三个文件为实验 2 相关文件，你在实验 1 中无需考虑。

此外，还会存在若干数据文件，具体数量取决于数据库中表的数量。每个表对应一个文件，文件路径为 `db_oid/table_oid`。

其中，系统数据库和系统表的 oid 为固定值，如系统数据库的 oid 为 1，实验 1 中所用的 tmp 数据库的 oid 为 2，其余系统表的 oid 可以在源码的 `common/constants` 中查看。普通数据库和表（即用户自己定义的数据库和表）的 oid 从 10000 开始递增分配。

## 测试 { #test }

每次实验均为端到端 SQL 测试，不设单元测试，测试框架基于 [sqllogictest](https://www.sqlite.org/sqllogictest/doc/trunk/about.wiki) 。

### 测试文件格式 { #test_format }

每个 sqllogictest 测试文件由一系列测试记录 (record) 组成，每条记录包含测试 SQL 语句以及语句的期望输出。测试记录分为 statement 和 query 两类。

对于 statement 记录，我们不指定期望输出结果，只判断语句是否成功执行，对应的 SQL 通常由两部分组成：

```
statement ok/error <label>
SQL
```

第一部分为 statement ok 或 statement error，表示这是一条 statement 记录，期望执行成功 (对应 ok) 或执行失败 (对应 error)；第二行为对应的 SQL。例如：

```sql
statement ok
create table test(id int, info varchar(10));

statement error
drop table not_exist;
```

在如上例子中，我们期望第一条 create table 语句成功执行，第二条 drop table 语句（试图删除一个不存在的表）执行失败，我们只关注语句是否成功执行，不关注语句的输出结果。

其中 label 是一个可选项，表示这条语句对应的客户端。在实验 2 和实验 3 中，我们会涉及到多个事务交替运行的场景，需要模拟多个客户端并发执行查询，此时需要在 label 字段指定每个 SQL 对应的客户端，例如：

```sql
statement ok C1
begin;

statement ok C2
begin;

statement ok C1
rollback;

statement ok C2
commit;
```

以上例子模拟了 C1 和 C2 两个客户端并发执行查询的场景，首先客户端 C1 开启事务，随后客户端 C2 开启事务，之后客户端 C1 将事务回滚，C2 将事务提交。

对于 query 记录，我们不仅期望查询语句成功执行，还会对执行结果进行比对，判断执行结果是否正确，每条 query 记录由以下四个部分组成：

```
query <sort-mode> <label>
SQL
----
result
```

第一部分包含 query, sort-mode 和 label。

query 表示这是一条 query 记录。

sort-mode 是一个可选项，表示比较 SQL 输出结果前是否进行排序，默认为 nosort，即在比较结果之前不对结果排序，这种模式适用于要求查询结果有序的情况，如包含 order by 的 SQL 语句。此外还可以指定为 rowsort，表示比较结果之前按行进行排序，将排序后的结果进行比较，这种情况适用于对于对查询结果顺序没有要求的语句。

label 的含义与 statement 语句的 label 含义相同。

第二部分为查询对应的 SQL 语句。

第三部分为`----`分隔符，此行之后是查询结果部分。

第四部分为查询结果，不同字段之间用空格分隔，暂不支持字段内包含空格的情况，测试时请确保数据内不包含空格，否则可能引发测试程序的误报。

例如：

```sql
statement ok
create table test(id int, info varchar(10));

query
insert into test values(1, 'aaa'), (2, 'bbb');
----
2

query rowsort
select * from test;
----
1 aaa
2 bbb

query
update test set id = 3 where id = 1;
----
1

query rowsort
select * from test;
----
2 bbb
3 aaa

query
delete from test;
----
2

statement ok
drop table test;
```

### 测试程序输出 { #test_output }

测试成功

```
xxx.test PASS
```

测试失败，分为以下几种情况：

-   期望执行成功，实际执行失败

```
xxx.test ERROR
xxx.test:<行号>
Unexpected error: <错误信息>
```

-   期望执行失败，实际执行成功

```
xxx.test ERROR
xxx.test:<行号>
Unexpected success
```

-   查询输出结果与期望结果不一致

```
xxx.test ERROR
xxx.test:<行号>
Unexpected error: Wrong Result
Your Result:
<你的查询结果>

Expected Result:
<期望查询结果>
```

此外，测试程序还会在测试文件夹`huadb_test`下生成两个文件：`yours.log`和`expected.log`，分别对应你的查询结果和期望查询结果。对于一些结果行数较多的查询，直接观察终端输出可能难以发现哪些行与期望输出不一致，此时可以使用`diff`工具对比两个文件的差异。

-   段错误

```
xxx.test Segmentation fault (core dumped)
```

```
xxx.test Bus error: 10
```

```
xxx.test Trace/BPT trap: 5
```

这些错误表示你的代码中存在一些非法操作，如空指针解引用、数组越界访问、栈溢出等。

-   死循环

```
xxx.test
```

如果你的测试程序长期保持在这个界面，没有 PASS 或 ERROR 的输出，那么你的程序很可能进入了死循环。
