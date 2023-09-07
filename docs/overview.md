# 实验框架简介

## 实验要求

每次实验聚焦于实验框架的一个模块，我们已为你提供了相应的类以及函数声明，你需要完成其中部分函数的实现，所有需要你来完成的代码部分均标注了`// LAB n BEGIN`，n 表示第 n 次实验，你可以通过 IDE 的全局搜索功能快速定位每次实验需要补充代码的位置。

在实现基础功能的过程中，你可以添加新的函数和变量，但不要修改已有函数的声明。

实现高级功能时，你可以对实验框架任意修改，建议单独开一个分支完成实验，避免影响后续实验。

## 运行

实验代码根目录的 Makefile 文件提供了多种命令的支持，具体包括：

-   `make` 或 `make lab1-debug`：实验 1 数据库构建。
-   `make debug`：实验 1 之后的数据库构建（debug 版本）。
-   `make release`：实验 1 之后的数据库构建（release 版本）。
-   `make shell`：进入数据库交互界面。
-   `make labn`：运行 lab 0 至 lab n 的所有测例（0 <= n <= 5）。
-   `make labn-only`：仅运行 lab n 的测例（0 <= n <= 5）。
-   `make labn/xx`：运行所有符合通配符 test/labn/x\* 匹配的测例。

我们采用系统表管理数据库的元信息，包括如下三个表：

| 表名            | 作用                                                                                    |
| --------------- | --------------------------------------------------------------------------------------- |
| huadb_table     | 存储所有表的 oid，所属数据库的 oid，表名，表的结构，以及表的行数（用于实验 5 基数估计） |
| huadb_database  | 存储数据库的 oid，以及数据库名                                                          |
| huadb_statistic | 存储列的统计信息，包括直方图和不同值的个数                                              |

完成实验 1 后，你可以进入 system 数据库中查看系统表的结构：

```console
huadb> \c system
 Change to database system
system> \d
+-----------------+
| table_name      |
+-----------------+
| huadb_table     |
| huadb_database  |
| huadb_statistic |
+-----------------+
(3 rows)

system> \d huadb_table
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

system> \d huadb_database
+---------+---------+------+
| name    | type    | size |
+---------+---------+------+
| db_oid  | uint    | 4    |
| db_name | varchar | 32   |
+---------+---------+------+
(2 rows)

system> \d huadb_statistic
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

在进行实验 1 之前，由于数据库还不支持记录的增删改查功能，系统表无法正常工作，你需要使用 catalog 文件夹下的 simple_catalog 来替代 system_catalog，因此在实验 1 中请使用 `make` 或 `make lab1-debug` 进行编译，完成实验 1 之后使用 `make debug` 进行编译。

## 测试

每次实验均为端到端 SQL 测试，不设单元测试，测试框架基于 [sqllogictest](https://www.sqlite.org/sqllogictest/doc/trunk/about.wiki) 。

### 测试文件格式

每个 sqllogictest 测试文件由一系列测试记录 (record) 组成，每条记录包含测试 SQL 语句以及语句的期望输出。测试记录分为 statement 和 query 两类。

对于 statement 记录，我们不指定期望输出结果，只判断语句是否成功执行，对应的 SQL 通常由两部分组成：

```sql
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

在如上例子中，我们期望第一条 create table 语句成功执行，第二条 drop table 语句执行失败（试图删除一个不存在的表），我们只关注语句是否成功执行，不关注语句的输出结果。

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

```sql
query <sort-mode> <label>
SQL
----
result
```

第一部分包含 query, sort-mode 和 label。

query 表示这是一条 query 记录。

sort-mode 是一个可选项，表示比较 SQL 输出结果前是否进行排序，默认为 nosort，即不对结果排序，此外还可以指定为 rowsort，表示比较前按行进行排序，将排序后的结果进行比较。对于不要求查询结果有序的语句，可以指定为 rowsort。

label 的含义与 statement 语句的 label 含义相同。

第二部分为查询对应的 SQL 语句。

第三部分为`----`分隔符，此行之后是查询结果部分。

第四部分为查询结果，不同字段之间用空格分隔，暂不支持字段内包含空格的情况，测试时请确保数据内不包含空格，否则可能引发测试程序的误报。

举例：

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

### 测试程序输出

测试成功

```console
xxx.test PASS
```

测试失败，分为以下几种情况：

-   期望执行成功，实际执行失败

```console
xxx.test ERROR
xxx.test:<行号>
Unexpected error: <错误信息>
```

-   期望执行失败，实际执行成功

```console
xxx.test ERROR
xxx.test:<行号>
Unexpected success
```

-   查询输出结果与期望结果不一致

```console
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

```console
xxx.test Segmentation fault (core dumped)
```

```console
xxx.test Trace/BPT trap: 5
```

这些错误表示你的代码中存在一些非法操作，如空指针解引用、数组越界访问、栈溢出等。

-   死循环

```console
xxx.test
```

如果你的测试程序长期保持在这个界面，没有 PASS 或 ERROR 的输出，那么你的程序很可能进入了死循环。
