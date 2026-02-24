# 实验 4:查询处理

## 实验概述 { #intro }

查询处理是指数据库针对一条查询，将磁盘存储的页面数据转换为符合查询所需结果的处理过程。火山模型是一种经典的拉取式执行器模型，通过抽象算子的执行逻辑，上层算子逐层从下层算子拉取数据，算子内部完成处理后响应上层的拉取请求。

算子的实现是本次实验的重点。算子是查询计划树的节点，数据库系统逐层调用算子的拉取函数完成查询的实际执行。部分算子在内部处理过程中需要存储中间结果来加速算子执行，例如连接算子以及聚合算子等，这种情况一般会使用内存作为中间结果的缓存。但是，部分算子在执行过程中将产生巨大存储容量的中间结果，不能够简单地存储到内存中（例如大表间的连接运算）。这类算子需要引入磁盘作为外部存储，一般称为外存算子。外存算子的执行效率对于磁盘 IO 的开销敏感，在实现上相较于内存算子更为复杂。本次实验要求补全几个常见的算子：连接、排序、Limit、聚合。考虑到不同算子本身实现难度的差异，基础功能中要求完成 Limit 算子、排序算子以及内存中的 NestedLoopJoin、MergeJoin 两类连接算子，高级功能在此基础上要求设计外存排序算子，同时也可以选择实现难度较高的哈希连接算子以及聚合算子。

![](../pics/lab4-overview.svg)

## 实验目标 { #goal }

本次实验要求完成如下基础功能：

1. 实现 Limit 算子。

2. 实现内存排序算子。

3. 实现内存连接算子：实现嵌套循环连接和归并连接两类连接算子。

在基础功能之上，实验框架支持完成以下高级功能：

1. 外存排序算子：在内存排序算子的基础上添加内存限制，使用磁盘存储中间结果，并设计内外存数据交互机制加速外存算法处理。

2. 内存哈希连接算子和聚合算子：实现内存存储的哈希表，并基于哈希表实现内存哈希连接算子以及聚合算子，中间结果利用哈希表保存。

## 关联知识点 { #knowledge }

本次实验关联查询处理与执行器章节，重点涉及以下知识点：

1. 查询算子：理解查询算子的概念。
2. 连接算法：包括嵌套循环连接算法和排序归并。
3. （可选）火山模型：执行器采用火山模型，理解火山模型将有助于理解查询的执行流程。

## 相关代码模块 { #code }

本次实验涉及到代码中以下功能模块：

-   [excutors](https://github.com/thu-db/huadb/blob/main/src/executors)：执行器相关类
    -   [executor](https://github.com/thu-db/huadb/blob/main/src/executors/executor.h)：执行器算子的抽象类。
    -   [limit_executor](https://github.com/thu-db/huadb/blob/main/src/executors/limit_executor.h)：Limit 算子，用于限制输出数量。
    -   [orderby_executor](https://github.com/thu-db/huadb/blob/main/src/executors/orderby_executor)：排序算子。
    -   [nested_loop_join_executor](https://github.com/thu-db/huadb/blob/main/src/executors/nested_loop_join_executor.h)：嵌套循环连接算子。
    -   [merge_join_executor](https://github.com/thu-db/huadb/blob/main/src/executors/merge_join_executor.h)：归并连接算子。
    -   [hash_join_executor](https://github.com/thu-db/huadb/blob/main/src/executors/hash_join_executor.h)：(高级功能) 哈希连接算子。
    -   [aggregate_executor](https://github.com/thu-db/huadb/blob/main/src/executors/aggregate_executor.h)：(高级功能) 聚合算子。

本次实验依赖于实验 1，请确保完成实验 1 再开始本次实验。

相关功能模块的抽象示意图如下：

![](../pics/lab4-details.svg)
