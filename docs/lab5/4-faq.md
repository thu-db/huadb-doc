# FAQ

Q: 在做投影下推功能时，explain 输出的查询计划正确，但是执行查询时报错 `Column index out of range` 是什么原因？

A: 投影节点的 exprs\_ 通常为 ColumnValue 类型的变量，ColumnValue 包含一个 col_idx\_ 变量，表示该列在子节点中 OutputColumns() 的下标。通常在投影下推前，投影节点的子节点为连接节点，连接节点的 OutputColumns() 为其两个子节点的 OutputColumns() 的并集。而下推后的投影节点的子节点一般为 SeqScan 节点，其 OutputColumns() 的内容与顺序和连接节点不同，从而导致 col_idx\_ 也需要改变。要解决这个问题，就需要在下推投影节点时重新维护 ColumnValue 的 col_idx\_ 信息，同时还可能需要维护最上层投影节点以及连接节点中涉及到的列的 col_idx\_ 信息。完整维护 col_idx\_ 信息较为繁琐，本次实验中不要求投影下推可以正确运行查询，只需要在 explain 输出的查询计划中正确展示投影下推结果即可。
