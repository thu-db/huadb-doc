# 实验框架简介

## 测试

每次实验均为端到端 SQL 测试，不设单元测试，测试框架基于 [sqllogictest](https://www.sqlite.org/sqllogictest/doc/trunk/about.wiki) ，并在其基础上对测试格式略加修改。

每个 sqllogictest 测试文件由一系列测试记录 (record) 组成，每条记录包含测试 SQL 语句以及语句的期望输出。测试记录分为 statement 和 query 两类。

对于 statement 记录，我们不指定期望输出结果，只判断语句是否成功执行，对应的 SQL 通常由两部分组成：

```sql
statement ok/error <label>
SQL
```

第一部分为 statement ok 或 statemenr error，表示这是一条 statement 记录，期望执行成功 (对应 ok) 或执行失败 (对应 error)；第二行为对应的 SQL。例如：

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
insert into test (1, 'aaa'), (2, 'bbb');
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
