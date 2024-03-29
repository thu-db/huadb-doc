# 高级功能

## 任务 1：垃圾回收（1 分） { #t1 }

### 实验描述 { #t1_intro }

基础功能中删除仅采用标记删除的方法，没有添加垃圾回收的功能，高级功能要求在此基础上补全 Vacuum 机制，实现主动垃圾回收功能。可参考 PostgreSQL 文档中关于 [Vacuum](https://www.postgresql.org/docs/current/routine-vacuuming.html) 的描述。

### 实现思路 { #t1_detail }

-   步骤 1：思考 Vacuum 过程中节约空间的收益与执行 Vacuum 的 IO 成本，设计一种合理的算法流程（可以参考 PostgreSQL 的 Vacuum 执行过程）。
-   步骤 2：理解 DatabaseEngine::ExecuteSql 函数的实现，并补充 Vacuum 函数的实现代码。
-   步骤 3：在 Table 和 TablePage 类中添加支持 Vacuum 操作的函数，根据步骤 1 的策略实现 Vacuum 函数。

## 任务 2：空闲空间管理（2 分） { #t2 }

### 实验描述 { #t2_intro }

基础功能中仅要求实现最简单的逐页遍历的方式来选择记录的插入页面，本任务要求在此基础上添加空闲空间数组来加速记录插入页面的选择过程。同时设计新的页面结构，按照页面的方式管理空闲空间数组。如果想了解现有数据库的实现方式，可参考 PostgreSQL 的 [Free Space Map](https://www.postgresql.org/docs/current/storage-fsm.html)（[中文解析 1](http://mysql.taobao.org/monthly/2019/03/06/)，[中文解析 2](https://zhmin.github.io/posts/postgresql-fsm-file/)）。

实现该功能时不要求通过基础功能的 `lab1/50-buffer_pool` 测例，但需要保证其他测例可以正常通过。

### 实现思路 { #t2_detail }

-   步骤 1：设计内存中的空闲空间数组，插入记录时根据记录大小和空闲空间数组快速寻找可以插入数据记录的页面。
-   步骤 2：空闲空间数组的页面化管理，按照 TablePage 的思路设计新的页面结构，将空闲空间数组组织为页面格式进行存储。
