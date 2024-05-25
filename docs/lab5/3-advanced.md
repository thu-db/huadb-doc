# 高级功能

## 任务 1：投影算子下推（1 分） { #t1 }

### 实验描述 { #t1_intro }

基础功能中完成了选择和连接算子的下推，高级功能中要求在此基础上添加投影算子的下推功能，可通过 `set enable_projection_pushdown = true` 打开投影下推开关。

### 实现思路 { #t1_detail }

-   步骤 1：实现 PushdownProjection 函数，对形如 `select a.id, b.id from a, b` 的 SQL 进行投影下推。
-   步骤 2：增加对查询中连接谓词相关列的处理，对形如 `select a.score from a, b where a.id = b.id` 的 SQL 实现投影下推。

设计测例时需要使用 explain 语句验证投影下推的正确性。

<!-- 还需要使用不带 explain 的 select 语句验证查询结果无误。 -->

## 任务 2：基于动态规划算法的连接顺序选择（2 分） { #t2 }

### 实验描述 { #t2_intro }

基础功能中的连接顺序选择使用的是贪心算法，高级功能中要求在此基础上添加基于动态规划的连接顺序选择算法。

### 实现思路 { #t2_detail }

在 ReorderJoin 函数中根据 join_order_algorithm\_ 的值选择使用的算法类型，并补充动态规划算法相关代码。
