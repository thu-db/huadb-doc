# 基础功能

## 任务 1：变长记录的页面组织（12 分）

本任务中，你需要补全 `table.cpp`, `table_page.cpp` 以及 `table_scan.cpp`，来实现记录的增删改查功能。

实验框架的文件组织方式为堆表，页面之间采用链表连接，页面组织支持变长记录，页面大小为 256B。页面头由以下几个字段组成：

| 变量名         | 变量类型  | 长度 | 作用                   |
| -------------- | --------- | ---- | ---------------------- |
| page_lsn\_     | lsn_t     | 8    | PageLSN，实验 2 中使用 |
| next_page_id\_ | pageid_t  | 4    | 记录下一个页面的页面号 |
| lower\_        | db_size_t | 2    | 页面空闲空间起始位置   |
| upper\_        | db_size_t | 2    | 页面空闲空间终止位置   |

每条记录都由记录头、空值向量和数据组成，记录头包含如下几个字段：

| 变量名    | 变量类型 | 长度 | 作用                                      |
| --------- | -------- | ---- | ----------------------------------------- |
| deleted\_ | bool     | 1    | 标注记录是否删除                          |
| xmin\_    | xid      | 4    | （实验 3）插入该数据的事务 id             |
| xmax\_    | xid      | 4    | （实验 3）删除该数据的事务 id             |
| cid\_     | cid_t    | 4    | （实验 3）插入该数据的事务内部 command id |

在实验 1 中，你只需要关注 deleted\_ 字段即可。

变长记录中的变长字段采用长度+数据的方式记录在原地，注意与教材介绍的方法不同，为简化设计，实验框架中没有将所有变长字段放于记录最后。

本任务中，你需要补全记录插入、读取和删除的代码，正确实现后可以通过测例`10-insert.test`, `20-delete.test` 和 `30-update.test`。

### 步骤 1：记录插入

我们首先来看记录插入的部分，记录插入的上层调用位于`insert_executor.cpp`，调用了 Table 类的 InsertRecord 函数，该函数返回插入记录的 rid\_（rid\_表示一条记录的位置，由页面 id 和页面中的槽 id 组成）。

Table 类的 InsertRecord 函数是你需要实现的部分。在这个函数中，你需要找到一个页面，调用页面 TablePage 的 InsertRecord 函数插入记录。

TablePage 的 InsertRecord 函数也是需要实现的部分，你需要维护页面的 lower 和 upper 指针，以及页面的记录槽信息（记录位置与记录长度），并将记录写入页面，同时不要忘了将页面标记为脏页，只有脏页才会在缓存替换时写回磁盘。

完成这两个函数后，记录插入的部分就全部完成了，此时你可以尝试测试`10-insert.test`：

```bash
make lab1/10
```

通常情况下，你将会得到如下输出：

```console
Test: 1/3
lab1/10-insert.test ERROR
lab1/10-insert.test:9
Unexpected error: Wrong Result
Your Result:

Expected Result:
1 1.1 a
2 2.2 bb
3 3.3 ccc
```

看上去好像记录插入并没有成功？其实不是，这是因为我们还没有补全记录读取的代码，不能通过 select 语句正确读取记录。但是，你可以通过 xxd 或 hexdump 等工具查看数据页面，判断记录插入是否成功。

```
xxd huadb_test/huadb_data/2/10000
```

或

```
hexdump -C huadb_test/huadb_data/2/10000
```

若你在程序右侧输出的 ascii 码中观察到插入数据的 a, bb, ccc 等字样，则说明数据已经成功写入到磁盘。

### 步骤 2：记录读取

下面我们来补全记录读取的代码。记录读取的上层调用位于 `seqscan_executor.cpp`，调用 TableScan 的 GetNextRecord 函数。

我们首先需要补全 GetNextRecord 函数，这个函数有若干参数，这些参数用于实验 3，在本次实验你无需使用这些参数。在这个函数中，你需要维护 rid\_ 成员变量，表示目前读取到的记录的 rid，这是因为上层会多次调用 GetNextRecord 函数，每次返回一条记录，我们需要用 rid\_ 成员变量记录目前读取到了哪一条记录，下次读取时返回下一条记录。维护 rid\_变量之后，调用 TablePage 的 GetRecord 函数获取记录。当表中所有数据读取结束时，返回空指针，表示结束读取。

之后你需要补充 TablePage 的 GetRecord 函数，根据 slot_id 和表的结构 column_list，从页面中反序列化得到记录。

如果你正确实现了以上函数，此时应该可以通过测例 `10-insert.test`。如果没有通过，可以根据测试输出，利用调试器对程序进行调试。

### 步骤 3：记录删除

进行这一步骤之前，请确保你已通过测例 `10-insert.test`。

记录删除是任务 1 的最后一个步骤，你需要补充 table.cpp 和 table_page.cpp 的 DeleteRecord 函数。

table.cpp 的 DeleteRecord 函数的实现思路与记录插入类似，我们只需通过 buffer_pool\_ 获取需要删除的记录对应的页面，调用页面 TablePage 的 DeleteRecord 函数即可。

table_page.cpp 的 DeleteRecord 函数实现了页面中记录的删除，在实验的基础功能中，我们仅要求实现标记删除即可。你只需设置记录头部的 deleted\* 变量即可。

完成上述步骤后，我们还需要修改 table_scan.cpp 的 GetNextRecord 函数，读取记录时判断记录是否已经被标记为删除，不再返回已经删除的数据。

至此，若你的实现正确，可以通过 `20-delete.test` 和 `30-update.test`。你可能会好奇，为何我们的框架没有 update 函数，却可以通过 update 测例。这是因为实验框架没有实现原地更新函数，所有更新操作均由删除+插入的方式实现，你可以查看 update_executor.cpp 来了解框架的实现方式。这种实现方式会导致删除数据的空间不能被回收，造成了空间的浪费，带来的好处是在实验 2 和实验 3 中更简洁的实现。如果你对删除数据的空间回收感兴趣，可以在高级功能中实现[垃圾回收](../3-advanced)。

## 任务 2：LRU 缓存替换算法（3 分）

### 实验描述

补全 storage 文件夹下的 LRU 缓存替换策略类，实现页面缓存池的 LRU 缓存替换策略。

### 实现思路

缓存替换的核心是基于输入的页面访问序列，选择按照算法标准设计的换出页面编号。可以按照如下的顺序完成本次实验：

-   步骤 1：补全 lru_buffer_strategy.h，设计 LRUBufferStrategy 类

BufferStrategy 采用了经典的策略模式。默认的函数模板提供了缓存池调用的接口，用具体的替换策略补全纯虚的抽象接口。
首先需要再 LRUBufferStrategy 类中添加记录信息所需的成员变量。

-   步骤 2：补全 lru_buffer_strategy.cpp，补全空缺的 Access 和 Evict 策略接口。

Access 函数：表示发生了页面访问，根据访问页面更新步骤 1 中添加的成员变量。

Evict 函数：按照 LRU 算法的规则，利用步骤 1 中添加的成员变量确定最久未使用的页面作为换出页面即可。

完成上面两个步骤即可补全 LRU 缓存替换算法。

## 基础功能实现顺序及测例分析

### table_page 的插入和删除

### table 的插入和删除

### Halloween Problem

### 缓存管理
