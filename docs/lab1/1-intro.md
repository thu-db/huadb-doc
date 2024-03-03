# 实验 1: 页面组织与缓存管理

## 实验概述 { #intro }

页面组织与缓存管理在数据库中扮演着重要的角色。页面组织对数据库的性能和效率有直接和重要的影响，合理的页面组织可以优化数据在磁盘上的存储布局，从而提高数据的访问效率。页面组织还可以影响事务处理的性能和并发控制的效率，合适的页面组织方式可以减少事务处理中的冲突和锁竞争，从而提高系统的并发性能。

缓存管理对数据库的性能和效率同样至关重要，缓存管理模块通过将常用的数据存储在内存中，使数据库不必每次都从磁盘中读取数据，降低了对磁盘 IO 的需求，减少了访问磁盘的开销。

在本次实验中，你需要实现变长记录的页面组织，并为缓存管理模块添加 LRU 替换策略。

![](../pics/lab1-overview.svg)

## 实验目标 { #goal }

本次实验要求完成如下基础功能：

1. 变长记录的页面组织：将长度不等的变长记录按照页面形式进行数据组织，实现对于变长记录页面的增删改查操作。

2. LRU 缓存替换算法：基于页面的访问序列管理缓存池，按照 LRU 缓存替换策略进行换出缓存页面的选择。

在基础功能之上，实验框架支持完成以下高级功能：

1. 垃圾回收：实验框架中采用标记方式删除数据，删除后的数据仍然占用磁盘空间。在此基础上添加 vacuum 功能，实现主动垃圾回收机制。

2. 优化堆表的插入效率：实验框架中数据表按照堆表形式进行组织，在此基础上设计多级空闲空间数组优化数据插入效率。

## 关联知识点 { #knowledge }

本次实验关联数据库存储章节，重点涉及如下的知识点：

1. 页面组织：变长记录的页面组织方式.
2. 文件组织：堆表组织、空闲空间数组的设计.
3. 缓冲区：缓存替换算法的实现将涉及到各类缓存替换策略的原理与算法流程.

## 相关代码模块 { #code }

本次实验涉及到代码中如下的功能模块：

-   [table](https://github.com/thu-db/huadb/tree/main/src/table)：数据表相关类.

    -   [record](https://github.com/thu-db/huadb/blob/main/src/table/record.h)：记录类。
    -   [table](https://github.com/thu-db/huadb/blob/main/src/table/table.h)：数据表类，需要补充记录插入和删除函数。
    -   [table_page](https://github.com/thu-db/huadb/blob/main/src/table/table_page.h)：变长记录页面类，需要补充页面内部记录插入和删除的函数。
    -   [table_scan](https://github.com/thu-db/huadb/blob/main/src/table/table_scan.h)：用于全表扫描，需要补充获取下条记录的函数来实现数据遍历。

-   [storage](https://github.com/thu-db/huadb/tree/main/src/storage)：用于管理内存和外存的交互
    -   [buffer_strategy](https://github.com/thu-db/huadb/tree/main/src/storage/buffer_strategy.h)：缓存替换算法的抽象类。
    -   [lru_buffer_strategy](https://github.com/thu-db/huadb/tree/main/src/storage/lru_buffer_strategy.h)：LRU 缓存替换算法类，需要补全实现。

基础功能需要补充约 100 行代码，本次实验的基础功能部分是后续所有实验的基础，未完成本次实验将影响后续实验的进行。

相关功能模块的示意图如下：

![](../pics/lab1-details.svg)
