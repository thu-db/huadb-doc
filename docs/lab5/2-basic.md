# 基础功能

{%
    include-markdown "common/update.md"
%}

与上次实验类似，本次实验同样具有较为明确的目标，所有修改均在 `optimizer/normalize_predicates.cpp`，`optimizer/push_down_predicates.cpp` 和 `optimizer/join_reorder.cpp` 中完成，你需要在优化器中修改查询计划树节点，得到更优的查询计划。

完成本实验时，你需要修改节点在查询计划树中的位置。熟悉 Operator 的成员变量及构造方式将会对完成本次实验有较大的帮助，在实验过程中可能需要阅读 operators 文件夹下的代码，了解不同 Operator 的构造方式。

由于谓词的形式多种多样，为简化代码工作量，你只需对本次实验测例中出现过的谓词形式进行处理即可，具体而言：

- 由 `AND`, `OR`, `NOT` 连接的原子谓词只会是以下形式之一：`col op const`, `const op col`, `col op col`，其中 `col` 是列名，`const` 是常数，`op` 是比较操作符（=, <, >, <=, >=, !=），不会出现其他形式的表达式（如 a.col1 + 1 = 3 \* a.col2 这样的复杂形式）。
- 不会出现 `GROUP BY`。
- 不会出现聚合函数。
- 只会存在整数列和字符串列两种类型。

本次实验的测试通过三种方式：
- 针对简单查询，使用 explain 命令，该命令会输出查询计划树，测试程序会比对你优化后的查询计划树与期望结果是否完全相同。
- 针对存在多种合法查询计划的复杂查询，通过 keyword 匹配判断 explain 命令输出的查询计划中期望关键词（SeqScan，Filter，NestedLoopJoin，and，or，not，原子谓词）出现次数是否符合要求（是否出现，出现多少次）。
- 通过比对查询结果来判断优化后的查询计划是否正确。


## 任务 1：谓词规范化（6分，2个公开测试点，8个隐藏测试点，每个测试点0.6分）

SQL 解析器产出的 Filter 节点中，谓词形态多样，如果直接交给后续 pass 处理，针对相同语义的谓词，每个 pass 都得单独应付多种变体。本任务在 `NormalizePredicates` 中把 Filter 谓词收敛到统一形态。

### 步骤 1：将 `const op col` 形式改写为 `col op const`

解析器对 `5 > a.id` 与 `a.id < 5` 都会产生合法的 `Comparison` 节点，但两者的左右孩子顺序不同。后续下推与索引选择对"列在左、常量在右"的形态做模式匹配最自然，否则每处都要写两套对称分支。

本步骤你需要递归遍历谓词表达式树，对每个 `Comparison` 节点做以下处理：当其 `children_[0]` 的表达式类型是 `OperatorExpressionType::CONST`、`children_[1]` 的类型是 `OperatorExpressionType::COLUMN_VALUE` 时，交换两个孩子，并把比较符按"换边"规则调整：`<` → `>`，`<=` → `>=`，`>` → `<`，`>=` → `<=`，`=` 与 `!=` 不变。

### 步骤 2：消除 `NOT`

本步骤你需要递归遍历谓词表达式树，把所有 `Logic` 类型为 `LogicType::NOT` 的节点消除，同时保持谓词整体语义不变。可以使用以下两个等价变换：

- 当 `NOT` 直接作用于 `AND`/`OR` 时按德摩根律下沉：`NOT(p AND q)` ≡ `NOT(p) OR NOT(q)`；`NOT(p OR q)` ≡ `NOT(p) AND NOT(q)`。
- 当 `NOT` 直接作用于 `Comparison` 时，去掉外层 `NOT`，把比较符替换为其取反结果：`=` → `!=`，`<` → `>=`，`<=` → `>`，`>` → `<=`，`>=` → `<`。

### 步骤 3：拆分顶层 `AND` 为多个 `Filter`

`FilterOperator` 可能形如 `p1 AND (p2 OR (p3 AND p4))`，任务 2 无法把这种谓词向下分发到不同子树（例如一部分到 Join 左侧、一部分到右侧）。因此，本步骤你需要把顶层 `AND` 拆开（仅拆顶层；`OR` 内部的 `AND` 保留，因为整个 `OR` 是一个不可拆的逻辑单位），为每个拆分后的子表达式单独构造一个 `FilterOperator`，自下而上叠在原 Filter 的 `children_[0]` 之上，最终返回最顶层的 `FilterOperator`。在上述例子中，拆分后形成两个 `FilterOperator`：`p1` 和 `p2 OR (p3 AND p4)`。

完成以上三步后，`NormalizePredicates` 的输出满足如下不变量：每个 Filter 节点的 `predicate_` 要么是单个 `Comparison`，要么是以 `OR` 为顶层节点的逻辑表达式；表达式内部不含 `NOT`；任何 `Comparison` 的 `CONST` 操作数都在右侧。

## 任务 2：谓词下推（6分，2个公开测试点，8个隐藏测试点，每个测试点0.6分）

谓词下推的目标是把 Filter 中的谓词尽可能"贴近数据源"地放置：能在扫描算子上方过滤的就贴在扫描上方、能成为连接条件的就并入连接节点。这样能减少中间结果的数据量，并在后续的索引选择和连接代价估计中提供更多机会。

本任务要求你在 `optimizer/push_down_predicates.cpp` 的 `PushDownPredicates` 中实现一次自顶向下递归。递归过程中维护一组"待下推谓词"，建议作为递归函数的参数沿调用链传递。算法整体流程如下：经过 `FilterOperator` 节点时把它的 `predicate_` 收入待下推集合并把该 `FilterOperator` 从树中删除；经过非 `FilterOperator` 和 `SeqScan` 的单子树算子时将该集合原封不动地传递给下层算子；到达 `SeqScan` 时把所有积累到此处的待下推谓词重建为该 scan 节点上方的 `FilterOperator`；到达 `NestedLoopJoin` 时按谓词引用的表所在子树将待下推集合三分，再递归子树并处理留在本节点的谓词。完成后将通过 `10-filter-pushdown.test` 与 `20-join-pushdown.test` 两个测例。

下面分别说明递归过程中遇到不同算子时的处理方式。

### `OperatorType::FILTER` 节点

将 `filter->predicate_` 加入待下推集合，对 `filter->children_[0]` 递归，并把递归结果作为本节点的替代返回（即把当前 Filter 从树中"删除"）。

### 单子树算子（`OperatorType::PROJECTION` 等）

本任务无需在这类算子处安放谓词，直接把待下推集合透传给其唯一子树继续递归即可。

### `OperatorType::SEQSCAN` 节点

所有积累到此处的待下推谓词都要在该 scan 节点之上重建为 `FilterOperator`（每个谓词一个 Filter，逐层叠加）。注意维护 `FilterOperator` 和 scan 节点输出的列映射关系。

### `OperatorType::NESTEDLOOP` 节点

到达 `NestedLoopJoinOperator` 时，你需要先确定它左右两棵子树各自覆盖了哪些表（即出现在子树下方所有 `SeqScanOperator` 的 `GetTableNameOrAlias()`），接着对待下推集合中的每一个原子谓词，收集该谓词内部所有 `ColumnValue` 引用的表名集合，并按以下规则三分：

- 全部表名都在左子树覆盖范围内 → 派发给左子树继续递归；
- 全部表名都在右子树覆盖范围内 → 派发给右子树继续递归；
- 跨左右两侧 → 留在本连表节点处理。

把派发好的两组分别作为新的"待下推谓词"，递归处理左、右子树并将返回结果赋回 `join->children_[0]` 与 `join->children_[1]`。需要先完成子树递归再处理留在本节点的谓词，因为留在本节点的列对列谓词在重写列下标时依赖子树最终的输出列。

随后将留在本节点的谓词写入 join condition。

## 任务 3：基于贪心算法的连接顺序选择（3分，1个公开测试点，无隐藏测试点）

在本任务中，你将使用贪心算法进行多表连接的连接顺序选择，你需要补充 `optimizer/join_reorder.cpp` 中的 ReorderJoin 函数，完成后将通过最后一个测例 `30-join-order.test`。

贪心算法的实现流程在课程中已有详细的描述，文档中不再赘述，你可以参考课程 PPT 来熟悉连接顺序选择问题的贪心算法。

本任务的测例也与课程 PPT 中的例子相同，唯一的区别是在表 r3 中多插入了一条数据，这是为了避免贪心算法中将 r3 选为第一个表，从而产生不同的运行结果。你的算法运行流程应与课程 PPT 中的步骤相同，你可以对照课程 PPT 进行调试。

> 任务 1 和任务 2 的正确性将直接关系到 lab 6 的部分测试用例的正确性，任务 3 的正确性对 lab 6 的影响较小。

## 报告要求 { #report }

{%
    include-markdown "common/report.md"
%}

本次实验中，请在实验报告中描述你在每个函数中补充的代码所做的工作。
