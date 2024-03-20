# 基础功能

## 任务 1：事务回滚（6 分） { #t1 }

本次实验中，你首先需要实现事务回滚的功能，主要涉及 `log/log_manager.cpp` 的 Rollback 函数，`log/log_records/insert_log.cpp`, `log/log_records/delete_log.cpp` 和 `log/log_records/new_page_log.cpp` 的 Undo 函数，此外，还需要修改 lab 1 中对 Table 类相关函数的实现，添加日志相关部分，并在 `table/table_page.cpp` 中增加 RedoInsertRecord 和 UndoDeleteRecord 的逻辑，完整实现后可以通过 `10-rollback.test` 测例。

### 步骤 1：日志追加 { #t1s1 }

实验框架通过读取日志的方式来实现事务回滚，当一个事务执行 rollback 语句时，数据库会逆序读取该事务产生的日志记录，将该事务所作的修改操作按倒序依次撤销。需要注意的是，实验框架采用了与课程讲授内容不同的方法，undo 日志和 redo 日志均使用物理日志。这一方面是为了减少本次实验的工作量，另一方面在实验框架的设计中，删除操作采用标记删除，且目前没有索引功能，不存在页面分裂导致数据项发生移动的情况，因此使用物理日志回滚不会影响数据记录的正确性。

要在事务执行 rollback 时完成回滚操作，我们必须保证事务从开始到回滚之间所作的页面修改操作都记录在日志中，正确记录日志是实现回滚的重要前提。因此第一步，我们需要修改 `table/table.cpp` 的 InsertRecord 和 DeleteRecord 函数实现，在 write_log 参数为 true 时，添加日志写入的代码。

!!! info "系统表无需写日志"

    为简化实现，实验框架中系统表相关操作不记录日志，系统表调用 InsertRecord 和 DeleteRecord 函数时会将 write_log 参数设为 false。实验也不要求对系统表操作进行回滚和重做，测例中不会出现 DDL 语句的重做和回滚。

与页面修改操作相关的日志共三种：

| 日志名称     | 类名       | 记录的信息                                                                                                                  |
| ------------ | ---------- | --------------------------------------------------------------------------------------------------------------------------- |
| 记录插入日志 | InsertLog  | 插入记录所属表的 oid，表所属数据库的 oid，插入数据的 page id，slot id，记录在页面中的偏移量，记录长度，以及记录的序列化数据 |
| 记录删除日志 | DeleteLog  | 删除记录所属表的 oid，表所属数据库的 oid，删除数据的 page id，slot id                                                       |
| 新建页面日志 | NewPageLog | 新页所属表的 oid，表所属数据库的 oid，前一页的 page id，新页的 page id                                                      |

具体而言，你需要在 `table/table.cpp` 中对页面进行修改操作（记录插入、记录删除、新建页面）时，通过 LogManager 的 AppendInsertLog, AppendDeleteLog 和 AppendNewPageLog 函数进行日志追加，这些函数会返回日志的 lsn，得到 lsn 后，你还需要设置页面的 page lsn，这将用于任务 3 ARIES 算法的实现。

你可以使用 huadb_parser 的日志解析功能来打印日志内容，用法为：

```
./build/debug/bin/huadb-parser -l <数据库路径>
```

数据库路径的默认值为 huadb_data。注意此处的数据库路径为整个数据库所在文件夹的路径，而非日志文件的路径。

### 步骤 2：Undo 日志读取 { #t1s2 }

执行 rollback 语句时，数据库会调用 LogManager 的 Rollback 函数，这一步骤中，我们需要实现 Rollback 函数。在步骤 1 调用 AppendLog 函数的过程中，已帮你维护了 LogManager 中的活跃事务表 att\_ 和脏页表 dpt\_ 变量，你可以在活跃事务表中查找事务最后一条日志记录的 LSN，每条日志记录都有一个 prev_lsn\_ 字段，表示该事务前一条日志记录的 LSN，通过活跃事务表 att\_ 和 prev_lsn\_，便可以实现对一个事务相关日志记录的倒序遍历。

!!! note "LSN 的表示方法"

    在课程中，LSN 使用递增的连续序号表示，而在实验框架中，LSN 使用日志记录在日志文件中的位置来表示，这种表示方法可以方便地通过 LSN 直接从日志文件中获取日志，无需再单独存一份 LSN 到日志位置的映射表。

得到一条日志记录的 LSN 后，你需要判断 LSN 与 flushed_lsn\_ 的大小关系。flushed_lsn\_ 表示刷新到磁盘的最大 LSN，通过比较 LSN 与 flushed_lsn\_，可以得知日志记录位于磁盘中还是 log 缓存中，进而从对应的位置获取到完整的日志记录。获取到日志记录后，调用日志记录的 Undo 函数进行回滚。

### 步骤 3：实现日志的 Undo 操作 { #t1s3 }

本步骤中，你需要实现 InsertLog, DeleteLog 和 NewPageLog 的 Undo 函数，以及 TablePage 类的 UndoDeleteRecord 函数，来撤销事务对页面的修改。

测例 `10-rollback.test` 分别进行了插入、删除、更新和新建页面的回滚测试，建议每实现完一种日志的回滚函数，进行一次测试，观察是否通过了对应操作的回滚测试。

## 任务 2：Redo（4 分） { #t2 }

本任务中，你将实现日志的 redo 函数，通过 `20-recover.test` 测例。

### 步骤 1：Redo 日志读取 { #t2s1 }

你需要补充 LogManager 的 Redo 函数，从头至尾顺序读取日志，直至 next_lsn\_，调用每个日志记录的 Redo 函数。

### 步骤 2：实现日志的 Redo 操作 { #t2s2 }

与任务 1 的步骤 3 类似，你需要实现 InsertLog, DeleteLog 和 NewPageLog 的 Redo 函数，以及 TablePage 类的 RedoInsertRecord 函数，来重做事务对页面的修改，正确实现这些函数后，你将通过 `20-recover.test` 测例。

与测例 `10-rollback.test` 类似，测例 `20-recover.test` 分别对插入、删除、更新和新建页面操作进行了重做测试，每实现完一种重做日志便可以进行测试。

## 任务 3：ARIES 恢复算法（5 分） { #t3 }

本任务中，你将实现完整的 ARIES 恢复算法，实现后将通过测例 `30-aries.test`，该测例与课程 PPT 案例说明中的例子是一致的。

### 步骤 1：实现 Analyze 函数 { #t3s1 }

首先，你需要实现 Analyze 函数，对应 ARIES 算法的分析阶段。我们已为你提供了读取 master record 部分的代码，你需要从 checkpoint_lsn 开始，正序读取日志，读取过程中根据日志类型，来对活跃事务表 att\_ 和脏页表中的数据 dpt\_ 进行维护。

### 步骤 2：修改 Redo 函数 { #t3s2 }

在任务 2 中，我们每次 Redo 均将日志文件从头到尾读取并重做，其中许多 Redo 操作是没有必要的。实现了 Analyze 函数后，我们可以根据脏页表的内容确定从从哪条日志开始重做，无需从头开始重做。

### 步骤 3：实现 Undo 函数 { #t3s3 }

经过 Analyze 阶段，你将得到数据库崩溃时的活跃事务表和脏页表，接下来，你将在 Undo 函数中调用任务 1 实现的 Rollback 函数将活跃事务回滚。

完成以上步骤后，在 `30-aries.test` 中，数据库恢复后将读取到正确的数据，从而完成本次实验。

<!-- ### 步骤 3：优化 Redo 次数 { #t3s3 }

你需要根据分析阶段维护的脏页表，在 redo 时根据脏页表中的 rec_lsn\_，页面的 page_lsn\_，以及日志的 lsn 的大小关系，判断 redo 操作是否有必要进行。每进行一次 redo 操作，你需要调用 LogManager 的 IncrementRedoCount 函数，统计 redo 操作次数。

正确实现本步骤后，你将通过测例 `30-aries.test`，从而完成本次实验。 -->

## 报告要求 { #report }

{%
    include-markdown "common/report.md"
%}

本次实验中，请在实验报告中描述 ARIES 算法的实现过程。
