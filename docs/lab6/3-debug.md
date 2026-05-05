# 调试说明

## B+树可视化

我们在 `index/btree_index.cpp` 中写好了 `BTreeIndex::Draw()`，同学们可以利用它输出 `.dot` 文件到指定路径，并将输出的内容复制到 GraphViz 插件或网页版中观察 B+ 树的拓扑图。

## verify btree 测试指令

针对任务 1（B+ 树页面结构）和任务 2（CREATE INDEX）的测试不走"用 SELECT 把树扫一遍看结果"的路线——这样会和任务 4（索引扫描）的实现耦合：scan 没写完的同学连 build 题都没法验。本实验改用**形态判定**：对 build 完成的 B+ 树直接做结构层面的不变式检查。

### 用法

`.test` 文件中的单行指令：

```
verify btree <索引名> height=<整数>
```

举例：

```
statement ok
create table t(a int);

statement ok
insert into t values (1), (2), (3);

statement ok
create index idx on t(a);

verify btree idx height=1
```

通过 → 该测试用例 PASS；任何一条不变式失败 → 输出 `FAIL: invariant <编号>: <详细诊断>` 并报错。

### 执行逻辑流程

1. sqllogictest 读到 `verify btree idx height=N` 这一行，向引擎发 SQL `SHOW idx;`。
2. `index/btree_print.cpp` 中的 `BTreeIndex::Print()` 把整棵树（meta + 全部 leaf/internal 页 + 堆表所有行的 indexed-cols 投影）以行式字符串导出，每行一个节点，类型标签开头。
3. 测试驱动把字符串喂给独立验证器（位于 `test/src/btree/`）。
4. 验证器解析字符串到内存模型，逐条跑 6 条不变式；第一条失败处停下并报告。

> 注意：为防止同学们误改 `BTreeIndex::Print()` 进而影响评测，同学们提交测试后我们会先替换同学们提交代码中的 `index/btree_print.cpp` 为标准文件然后执行测试。

### 6 条不变式

| 编号 | 不变式 | 如果不满足可能说明哪些 bug |
|------|--------|--------------|
| 1 | **拓扑结构是平衡树**：root 必须可达全部非 meta 页；每个非 root 页恰被 1 个 internal slot 引用；无环；internal slot 的 `ptr_.slot_id_` 恒为 0；同一 internal page 的所有子页类型一致；每个叶子到根的路径长度相等 | `BuildAdd` ptr 指错；`FinishBuild` 漏登记导致孤儿页；`SetRootPage` 未调用；`SetRootPage` 没有将 meta page 设置为脏页 |
| 2 | **叶子双向链表**：从 root 下探到最左叶；沿 `next_page_id_` 走完整条链 == 全部 leaf 集合；每页的 `prev_page_id_` 与前驱一致；首叶 prev=NULL，末叶 next=NULL | `BuildAdd` 中 leaf 链的 prev/next 维护错 |
| 3 | **层内严格升序（扩展键）**：同页 `slot[i].extended_key < slot[i+1].extended_key`；叶子层跨页（按 next 链）相邻 leaf 的 `last_slot < first_slot`。比较用扩展键 `(values_, heap_rid_)`，不是 user key | `AddRecord` slot 顺序错；`Build` 排序比较器漏 `heap_rid_` 作 tiebreaker |
| 4 | **父 slot key = 子 slot[0] key**：每个 internal slot 的 `(values_, heap_rid_)` 必须严格等于其 ptr 指向页面 slot[0] 的 `(values_, heap_rid_)`（key=min 不变式） | `BuildAdd` 提升父 slot 时构造错；扩展键 tiebreaker 在 internal 层失效 |
| 5 | **树高 = 期望值**：从 root 沿 slot[0] 一路下探到 leaf，计数层数。单 leaf 即 root 高度 = 1（空树也是单 leaf）；多页树每多一层 internal 高度加一。期望值由 `verify btree` 指令的 `height=N` 给出 | `BuildAdd` 父层创建错；`FinishBuild` 自底向上 walk 漏层 |
| 6 | **叶子层 ↔ 堆表元组一一对应**：所有 leaf slot 的 `(values, heap_rid)` 构成的集合 == 堆表所有行 `(投影到索引列, rid)` 构成的集合；且叶子节点的 `slot.ptr == slot.heap_rid` | `Init` 没初始化页面导致值是垃圾；`Build` 漏行；`FinishBuild` 漏登记导致部分叶子从 root 不可达 |
