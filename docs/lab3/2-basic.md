# 基础功能

## 任务 1：解决万圣节问题（3 分） { #t1 }

还记得在实验 1 中提到的[万圣节问题](../lab1/2-basic.md#t1_s3)吗，本次实验的第一个任务便是解决这个问题，完成后你将通过 `10-halloween.test` 测例。

### 步骤 1：添加记录头信息 { #t1_s1 }

记录头信息位于 `table/record_header.h` 文件，具体包括以下几个字段：

| 变量名    | 变量类型 | 长度 | 作用                                |
| --------- | -------- | ---- | ----------------------------------- |
| deleted\_ | bool     | 1    | 标注记录是否删除（实验 3 之后弃用） |
| xmin\_    | xid      | 4    | 插入该数据的事务 id                 |
| xmax\_    | xid      | 4    | 删除该数据的事务 id                 |
| cid\_     | cid_t    | 4    | 插入该数据的事务内部 command id     |

实验框架的 TransactionManager 类会在每个事务开启时为其分配一个事务 id，记为 xid。在每个事务内部，TransactionManager 会为每条 SQL 语句分配一个 command id，记为 cid，我们将通过 cid 来解决万圣节问题。

你需要修改实验 1 插入、删除记录和实验 2 中撤销删除记录的代码，涉及 TablePage 的 InsertRecord, DeleteRecord 和 UndoDeleteRecord 函数，在插入、删除以及撤销删除记录时正确设置记录头信息，在插入记录时设置 xmin 和 cid，删除和撤销删除记录时设置 xmax。

### 步骤 2：修改可见性判断条件 { #t1_s2 }

下一步你需要修改 `table/table_scan.cpp` 中 GetNextRecord 函数关于可见性判断的部分，为了解决万圣节问题，我们不仅需要判断记录是否被删除，还要判断读入的记录是不是由当前正在执行的 SQL 语句插入的，判断依据便是记录的 xid 和 cid，当前正在执行的 SQL 语句的 xid,和 cid。正确修改判断条件后，你将通过`10-halloween.test` 测例。同时，你也要保证代码的修改不影响前两次实验的正确性。

## 任务 2：通过多版本并发控制实现可重复读隔离（3 分） { #t2 }

本任务中，你将使用多版本并发控制，来实现可重复读隔离级别，正确实现后将通过 `20-mvcc_insert.test`, `21-mvcc_delete.test`, `22-mvcc_update.test`, `23-write_skew.test` 和 `30-repeatable_read.test` 测例。

### 步骤 1：设置活跃事务集合 { #t2_s1 }

在可重复读隔离级别中，你需要获取事务开始时的活跃事务集合，用于可见性判断。TransactionManager 已为你保存了这些信息，你只需要传入 xid 即可获取事务 xid 开始时的活跃事务集合，将该信息补充到 `executors/seqscan_executor.cpp` 的 Next 函数中。

### 步骤 2：修改可见性判断条件 { #t2_s2 }

修改 `table/table_scan.cpp` 中的 GetNextRecord 函数，根据隔离级别，活跃事务 id，以及当前事务的 xid，cid，判断记录可见性。由于可见性判断的部分较为复杂，可能涉及多个分支，你可以将此部分单独设置一个函数。正确设置后将通过`20-mvcc_insert.test`, `21-mvcc_delete.test`, `22-mvcc_update.test`, `23-write_skew.test` 和 `30-repeatable_read.test` 测例。

你可能会发现，尽管通过了实验 3 的 20-30 测例，但实验 2 的 `30-aries.test` 再次出错。这是因为实验框架的默认隔离级别为读已提交，而本任务中我们对可重复读隔离级别进行了设置。你需要在判断可见性时考虑到当前的隔离级别，根据不同级别使用不同的可见性判断条件。在本任务中，你可以先忽略这个问题，后续任务我们将指导你完成其他隔离级别的可见性判断条件。

## 任务 3：实现读已提交隔离（2 分） { #t3 }

与任务 2 类似，你需要做的也是设置活跃事务集合和修改可见性判断条件，你需要思考读已提交和可重复读的判断条件有哪些不同，设置正确后将通过 `40-read_committed.test` 测例。

## 任务 4：强两阶段锁（4 分） { #t3 }

任务 2 中，我们实现了多版本并发控制，无阻塞地处理了读写冲突，本任务中，我们将使用强两阶段锁 (SS2PL)解决写写冲突问题。

### 步骤 1：实现锁管理器 LockManager { #t4_s1 }

你需要实现 LockManager 类的 LockTable, LockRow 和 ReleaseLocks 函数，这几个函数分别实现了加表锁、加行锁和释放锁的功能，你可能需要添加适当的数据结构来保存锁信息。

此外，LockManager 类还提供了两个私有函数 Compatible 和 Upgrade 接口，Compatible 函数用于判断不同事务对同一个对象（行或表）加的锁是否兼容，Upgrade 用于同一个事务对相同对象重复加锁时实现锁的升级。

### 步骤 2：为修改操作加锁 { #t4_s2 }

完成 LockManager 后，你需要在 `executors/insert_executor.cpp`, `executors/delete_executor.cpp` 和 `executors/update_executor.cpp` 中为修改操作加锁，如果加锁失败，即其他事务已经获取相应对象的锁，且锁类型不兼容，则抛出异常。

正确实现以上步骤后，你将通过 `50-lock.test` 测例。

## 任务 5：通过多版本两阶段锁实现可串行化隔离（3 分） { #t5 }

最后一个任务，你将综合以上完成的所有任务，通过多版本两阶段锁协议，实现可串行化隔离。

### 步骤 1：实现 select for update 和 select for share 的加锁 { #t5_s1 }

你需要在 `executors/lock_rows_executor.cpp` 中为 select for update 和 select for share 添加支持，根据 SQL 语句类型加读锁或写锁。

### 步骤 2：根据 select 类型获取活跃事务集合 { #t5_s2 }

修改 `executors/seqscan_executor.cpp`，在可串行化隔离级别下，根据 SQL 语句的类型，判断应采用当前读还是快照读，进而设置正确的活跃事务集合。

### 步骤 3：设置可串行化隔离级别的可见性判断条件 { #t5_s3 }

修改 `table/table_scan.cpp`，为可串行化级别设置可见性判断条件。

正确完成以上步骤后，你将通过 `60-mv2pl.test` 和 `70-write_skew_serializable.test`，完成本次实验的所有基础功能。
