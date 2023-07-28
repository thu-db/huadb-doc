# LAB 4 执行器实验文档

## 实验概述

本次实验主要关注于 SQL 执行流程，重点为 join 算子的实现。

## 实验任务

1. 完成 AndConditionNode 和 OrConditionNode 的构建过程。
2. 实现 JoinNode 的构建和执行过程。

## 相关模块

1. optim 模块：负责新建执行算子节点。
2. join_node 模块：定义了 join 算子的执行过程。

## 基础功能实现顺序

1. optim/optim.cpp: 实现 AndConditionNode 和 OrConditionNode 的 visit 函数。
2. optim/optim.cpp: 实现 JoinConditionNode 的 visit 函数。
3. optim/optim.cpp: 在 Select 的 visit 函数中添加连接算子的部分。
4. oper/join_node.cpp: 实现 Next 函数，进行连接运算。

## 可选高级功能

1. 实现多种 join 算法(1分)：至少实现三种不同的连接算法，如 Nested Loop Join, Sort Merge Join, Hash Join，设计测例比较不同算法的效率（仅要求2表Join的比较，要求涉及测例中join双方涉及条目数量有数量级差异的多组测试）。
2. 基于外存的 join 算子(1分)：至少实现一种外存算法，如外存 Hash、外存 Sort Merge 等。注意外存算法同样需要进行页式管理，设计的测例要求中间结果超过一个页面存储容量，同时要在报告中写明中间结果占用页面的回收机制。
3. 实现聚合算子(1分)：实现 SUM, AVG, MIN, MAX, COUNT 算子以及配套的GROUP BY算子，并设计测例验证正确性。其中测例中GROUP BY算子要求至少达到3个分组，其中至少有1个分组涉及的数据条目超过2条。

高级功能满分 3 分。

## 实现要点

1. 对于多个 AND 或 OR 连接的条件，解析器会将条件解析为左深树，即所有 AND 结点的右孩子均为比较条件或连接条件，不会是其他 AND OR 条件。

2. 所有 visit 函数必须具有返回值，即必须以 return 语句结尾。如不需要返回值可返回 nullptr，没有返回值可能出现 Segmentation fault。

3. 实现 visit 函数时可参考 optim 中其他节点 visit 函数的代码。

4. 使用并查集维护表的连接信息，并查集实现代码位于 utils/uf_set.h 文件。

## 截止时间

2023 年 5 月 7 日（第十一周周日）晚 23:59 分。
