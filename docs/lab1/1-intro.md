# 实验 1: 缓存管理与页面组织

## 实验概述

本次实验为缓存管理与页面组织的实验，意图通过实现记录的页面组织和经典缓存替换算法来更好地理解数据库系统页面管理模块的相关功能。

数据库页面管理承担了数据库系统中内存与磁盘的交互枢纽，一方面通过页面缓存的管理实现高效的磁盘页面访问，同时需要具备将不同类型数据页面解析成特定格式化数据的功能，以便于应对上层不同功能模块的各类 API 接口调用。

页面替换算法和高效率的页面组织是页面管理的难点。高效的磁盘访问依赖于缓存的高命中率，但不规则的页面访问模式增大了缓存替换算法的设计难度；而页面组织需要充分考虑不同类型的页面（记录、索引等）的同时需要结合数据压缩、数据加密等技术保证效率和安全性。全部实现上述内容对于本次实验来讲过于宏大，因此在本次实验的基础功能中，仅要求完成常规的变长形式记录数据的页面组织和经典的 LRU 缓存替换算法。

![](../pics/lab1-overview.svg)

## 实验目标

本次实验要求完成如下基础功能：

1. 变长记录页面组织：将长度不等的变长记录按照页面形式进行数据组织，实现对于变长记录页面的增删改查操作。

2. LRU 缓存替换算法：基于页面的访问序列管理缓存池，按照 LRU 缓存替换策略进行换出缓存页面的选择。

在基础功能之上，实验框架支持完成以下高级功能：

1. 垃圾回收：实验框架中采用标记方式删除数据，删除后的数据仍然占用磁盘空间。在此基础上添加 vacuum 功能，实现主动垃圾回收机制。

2. 优化堆表的插入效率：实验框架中数据表按照堆表形式进行组织，在此基础上设计多级空闲空间数组优化数据插入效率。

## 关联知识点

本次实验关联数据库存储章节，重点涉及如下的知识点：

1. 页面组织：变长记录的页面组织任务将涉及到变长记录的组织方式
2. 文件组织：堆表的插入效率优化任务将涉及到堆表组织、空闲空间数组的设计
3. 缓冲区：缓存替换算法的实现将涉及到各类缓存替换策略的原理与算法流程

## 相关代码模块

本次实验涉及到代码中如下的功能模块：

-   [table](https://github.com/thu-db/huadb/tree/main/src/table)：数据表相关类

    -   [record](https://github.com/thu-db/huadb/blob/main/src/table/record.h)：记录类，已经完成。
    -   [table](https://github.com/thu-db/huadb/blob/main/src/table/table.h)：数据表类，需要补充记录插入和删除函数。
    -   [table_page](https://github.com/thu-db/huadb/blob/main/src/table/table_page.h)：变长记录页面类，需要补充页面内部记录插入和删除的函数。
    -   [table_scan](https://github.com/thu-db/huadb/blob/main/src/table/table_scan.h)：用于全表扫描，需要补充获取下条记录的函数来实现数据遍历。

-   [storage](https://github.com/thu-db/huadb/tree/main/src/storage)：用于管理内存和外存的交互
    -   [buffer_strategy](https://github.com/thu-db/huadb/tree/main/src/storage/buffer_strategy.h)：缓存替换算法的抽象类。
    -   [lru_buffer_strategy](https://github.com/thu-db/huadb/tree/main/src/storage/lru_buffer_strategy.h)：LRU 缓存替换算法类，需要补全实现。

相关功能模块的抽象示意图如下：

![](../pics/lab1-details.svg)

<!--TODO:添加部分教材中的示意图-->
