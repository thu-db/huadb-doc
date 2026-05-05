# 实验 6: B+ 树索引

## B+树概述

B+ 树本质是一种面向磁盘存储而设计的多叉平衡查找树。在使用顺序扫描（SeqScan）时，必须将所有数据页从磁盘读入缓冲池，I/O 代价随数据量线性增长；若改用二叉查找树，虽能将查找复杂度降至对数级，但其结构定义只允许每个节点拥有两个孩子，因而扇出度极低，树高由记录总数的以2为底的对数决定。考虑到磁盘以页为单位进行读取，从根到叶的每一层在最坏情况下都意味着一次跨页的指针跳转，因此查找一条记录所需的磁盘 I/O 次数依然偏高。

B+ 树的核心思想是把"一页可容纳数百个键与指针"这一容量优势直接转化为节点的扇出度：让每个树节点恰好对应一个数据页，节点内部以有序的键值数组支持页内查找，一个节点有多少个键，就可以拥有多少个孩子。扇出度越高，相同记录数下树就越矮——存储数十亿条记录时高度仍仅有三到四层，从而每次查找只需读取极少的页面便能定位到目标记录所在的叶子页。此外，B+树仅在叶子节点中存放真实数据，内部节点只保存用于导航的键与子节点指针，这使得内部节点更紧凑，单页内可容纳的键数更多，扇出度因此进一步提高；同时所有叶子节点之间通过指针串联成一个有序链表，因此在执行范围查询 (e.g., `WHERE a > 10 AND a < 20`) 时可以沿链表顺序读取相邻叶子页，而无需通过上层节点读取相邻叶子页。综上，B+ 树结构是数据库系统在"磁盘 I/O 昂贵、且以页为单位进行读取"这一硬约束下，通过最大化节点扇出度，对单点查找与范围扫描两类操作进行的索引优化。

## 实验目标

本实验涉及B+树的创建和查找两类操作，需要实现的具体功能如下

1. 完善 B+ 树页面结构
- 补全 index/btree_index.cpp 中的 SetRootPageId 和 GetRootPageId 函数
- 补全 index/btree_page.cpp 的 Init 和 AddRecord 函数

2. 实现 B+ 树创建功能 (CREATE INDEX ...)
- 补全 index/btree_index.cpp 中的 Build，BuildAdd，FinishBuild 函数

3. 实现删除表 (DROP TABLE ...) 时级联删除 B+ 树索引
- 补全 catalog/system_catalog.cpp 中的 DropTable 函数

4. 实现 B+ 树索引扫描功能
- SeqScan 转 BTreeIndexScan：补全 optimizer/optimizer.cpp 中的 Optimize 和 Seq2Index 函数
- 实现 RidUnion 算子，用于拼接并去重 SeqScan 和 BTreeIndexScan 的结果：补全 executors/rid_union_executor.cpp 的 Init 和 Next 函数
- 实现 BTreeIndexScan 算子：补全 executors/btree_scan_executor.cpp 中的 Init 和 Next 函数，补全 index/btree_index.cpp 中的 Search 函数

## 关联知识点

本次实验关联数据库索引章节，重点涉及以下知识点：

1. B+树索引结构
2. B+树索引查找
