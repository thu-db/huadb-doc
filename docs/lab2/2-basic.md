# 基础功能

## 任务 1：事务回滚（5 分）

本次实验中，你首先需要实现事务回滚的功能，主要涉及 log_manager.cpp 的 Rollback 函数，insert_log.cpp, delete_log.cpp 和 new_page_log.cpp 的 Undo 函数，此外，还需要修改 lab 1 中对 table.cpp 的实现，添加日志相关部分，并在 table_page.cpp 中增加 RedoInsertRecord 和 UndoDeleteRecord 的逻辑，完整实现后可以通过 `10-rollback.test` 测例。

### 步骤 1：日志追加

实验框架通过读取日志的方式来实现事务回滚，当一个事务执行 rollback 语句时，数据库会逆序读取该事务产生的日志记录，将该事务所作的修改操作按倒序依次撤销。需要注意的是，实验框架采用了与课程讲授内容不同的方法，undo 日志和 redo 日志均使用物理日志。这一方面是为了减少本次实验的工作量，另一方面在实验框架的存储设计中，undo 日志可以采用物理日志来记录，你可以思考一下在实验框架中为何使用物理日志进行 undo 操作不会影响正确性。

要在事务执行 rollback 时完成回滚操作，我们必须保证事务从开始到回滚之间所作的页面修改操作都记录在日志中，正确记录日志是实现回滚的重要前提，因此第一步，我们需要修改 table.cpp 的 InsertRecord 和 DeleteRecord 函数实现，在 write_log 参数为 true 时，添加日志写入的代码。

!!!info "系统表不写日志"

    为简化实现，实验框架中系统表相关操作不记录日志，系统表调用 InsertRecord 和 DeleteRecord 函数时会将 write_log 参数设为 false。实验也不要求对系统表操作进行回滚和重做，测例中不会出现 DDL 语句的重做和回滚。

与页面修改操作相关的日志共三种：

| 日志名称     | 类名       | 记录的信息                                                                                                                  |
| ------------ | ---------- | --------------------------------------------------------------------------------------------------------------------------- |
| 记录插入日志 | InsertLog  | 插入记录所属表的 oid，表所属数据库的 oid，插入数据的 page id，slot id，记录在页面中的偏移量，记录长度，以及记录的序列化数据 |
| 记录删除日志 | DeleteLog  | 删除记录所属表的 oid，表所属数据库的 oid，删除数据的 page id，slot id                                                       |
| 新页日志     | NewPageLog | 新页所属表的 oid，表所属数据库的 oid，前一页的 page id，新页的 page id                                                      |

具体而言，你需要在对页面进行修改操作（记录插入、记录删除、新建页面）时，通过 LogManager 的 AppendInsertLog, AppendDeleteLog 和 AppendNewPageLog 函数进行日志追加，这些函数会返回日志的 lsn，得到 lsn 后，你还需要设置页面的 page lsn，这将用于任务 3 ARIES 算法的实现。

### 步骤 2：Undo 日志读取

执行 rollback 语句时，数据库会调用 LogManager 的 Rollback 函数，这一步骤中，我们需要实现 Rollback 函数。在步骤 1 调用 AppendLog 函数的过程中，已帮你维护了 Logmanager 中的活跃事务表 att\_ 和脏页表 dpt\_ 变量，你可以在活跃事务表中查找到事务最后一条日志记录的 LSN，每条日志记录都有一个 prev_lsn\_ 字段，表示该事务前一条日志记录的 LSN，通过活跃事务表 att\_ 和 prev_lsn\_，便可以实现对一个事务相关日志记录的倒序遍历。

!!! note "LSN 的含义"

    在课程中，LSN 使用递增的连续序号表示，而在实验框架中，LSN 使用的是日志记录在日志文件中的位置表示，这种表示方法可以方便地通过 LSN 直接获取日志，无需再单独存一份 LSN 到日志位置的映射表。

得到一条日志记录的 LSN 后，你需要判断 LSN 与 flushed_lsn\_ 的大小关系。flushed_lsn\_ 表示刷新到磁盘的最大 LSN，通过比较 LSN 与 flushed_lsn\_，可以得知日志记录位于磁盘中还是 log 缓存中，进而从对应的位置获取到完整的日志记录。获取到日志记录后，调用日志记录的 Undo 函数进行回滚。

### 步骤 3：实现日志的 Undo 操作

本步骤中，你将实现 InsertLog, DeleteLog 和 NewPageLog 的 Undo 函数，以及 TablePage 类的 UndoDeleteRecord 函数，来撤销事务对页面的修改。

对于 InsertLog，你需要将插入的记录删除；对于 DeleteLog，你需要清除记录的删除标记（调用 TablePage 类的 UndoDeleteRecord 函数）；对于 NewPageLog，你需要重新设置前一页的 next_page_id\_ 字段，正确实现这些函数后，你将可以通过测例 `10-rollback.test`。

测例 `10-rollback.test` 分别进行了插入、删除、更新和新建页面的回滚测试，建议每实现完一种日志的回滚函数，进行一次测试，观察是否通过了对应操作的回滚测试。

## 任务 2：物理日志的设计（5 分）

### 实验描述

补全 log/logs 文件夹下 insert_log.h 和 delete_log.h 来完成物理日志的设计，并修改 Table 类中对应函数。

### 实现思路

日志设计的核心在于规划日志的存储格式，以及利用存储信息实现 Redo 和 Undo 的方式。可以按照如下的顺序完成本次实验：

-   步骤 0：结合课程内容，思考为什么仅使用物理日志不会影响系统正确性

-   步骤 1：补全 log/insert_log.h 和 log/delete_log.h，实现插入和删除记录的物理日志

框架已经给出了这两类日志所需的必要信息，并实现了序列化和反序列化函数。实验中要求根据存储的信息确定在重做或撤销过程中，如何利用存储信息还原操作对于页面的影响。

-   步骤 2：修改 table/table.cpp，添加记录变更时日志记录的操作

log_manager 文件中的日志管理器 LogManager 类已经提供了追加写出日志的接口，并在内部完成相关的缓存管理功能。
因此本次实验仅需要在实验 1 的基础上，利用 LogManager 提供的追加日志接口(AppendXXXLog)在记录变更时记录对应的日志即可。

完成上面两个步骤即可补全物理日志的功能。

## 任务 3：ARIES 恢复算法（5 分）

### 实验描述

补全 log_manager.cpp 中 LogManager 类的 Analyze，Redo，Undo 函数，从而补全 ARIES 故障恢复算法。

### 实现思路

ARIES 故障恢复算法是针对于 steal+no-force 数据库系统的一种故障恢复算法，在实现前需要充分理解课程中讲解的 ARIES 算法执行流程。
可以按照算法的执行顺序依次补全 3 个函数：

-   步骤 1：补全 LogManager::Analyze，ARIES 算法的分析过程

ARIES 算法的分析过程读取检查点信息，恢复故障时的数据库状态并恢复活跃事务表和脏页表。
实验框架中已经给出了数据库状态的恢复过程，本次实验中需要基于 checkpoint_lsn 恢复活跃事务表、脏页表以及算法所需的 LSN 信息。

-   步骤 2：补全 LogManager::Redo，ARIES 算法的重做过程

ARIES 的重做过程要求按照算法流程（需要步骤 1 中恢复的 LSN 信息）遍历执行日志的重做过程来恢复系统故障时缓存内的脏页状态。
该步骤的正确性依赖于物理日志设计中 Redo 函数的正确性。

-   步骤 3：补全 LogManager::Undo，ARIES 算法的撤销过程

ARIES 的撤销过程需要根据活跃事务表撤销未提交事务产生的页面更新，该步骤的正确性依赖于前两个步骤的正确性和物理日志设计中 Undo 函数的正确性。

完成上面三个步骤即可补全 ARIES 恢复算法。

## 基础功能实现顺序及测例分析

### 实现事务回滚

insert delete

### 实现故障恢复重做

insert delete new_page

### 实现 Aries
