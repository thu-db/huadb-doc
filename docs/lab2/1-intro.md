# 实验 2: 事务处理与故障恢复

## 实验概述 { #intro }

事务处理和故障恢复是数据库系统的核心功能之一，它确保了数据库操作的原子性、一致性、隔离性和持久性，是数据库系统稳定高效运行的关键机制。

数据库系统必须保证事务处理过程中意外的系统宕机不会影响事务的 ACID 特性，但出于性能因素考量的 steal+no-force 的脏页刷盘机制让故障情况下磁盘页面的数据违背原子性和持久性。此时，事务处理过程中的日志机制和故障后的恢复算法就起到了至关重要的作用。

日志格式的设计和故障恢复算法实现是本次实验的重点。日志格式设计时需要保存必要的数据，支持重做（Redo）和撤销（Undo）操作。故障恢复算法则需要在数据库发生故障时，根据日志信息对数据进行分析、重做和撤销操作，同时也要尽可能降低故障恢复时间。本次实验要求完成物理日志的设计和 ARIES 故障恢复算法。

![](../pics/lab2-overview.svg)

## 实验目标 { #goal }

本次实验要求完成如下基础功能：

1. 事务的回滚：实现事务的 rollback 操作。
2. 物理日志的设计：设计物理日志的存储格式，并实现单个日志的重做和撤销函数。
3. ARIES 故障恢复算法：补全分析、重做、撤销三个阶段的函数实现，实现 ARIES 故障恢复算法。

在基础功能之上，实验框架支持完成以下高级功能：

1. Undo 过程中出现故障的恢复：基础功能中未考虑 Undo 过程中（包括事务 rollback 和故障恢复中的 Undo）出现故障的处理，在基础功能的基础上添加 Undo 过程中出现故障的故障恢复，核心为补偿日志记录的实现。

2. 非阻塞检查点机制：实验框架中的 Checkpoint 操作阻塞的方式来记录日志，在此基础上实现非阻塞的 Checkpoint 机制，核心为修改 Checkpoint 函数的实现。

## 关联知识点 { #knowledge }

本次实验关联事务管理以及故障恢复章节，重点涉及以下知识点：

1. 事务的概念：完成本次实验需要理解事务的原子性和持久性的概念。
2. 日志实现方式：物理日志的设计将涉及物理日志的存储格式和物理日志在故障恢复中的性质与作用。
3. ARIES 恢复算法：ARIES 算法的实现将涉及 ARIES 的应用场景和算法流程。

## 相关代码模块 { #code }

本次实验涉及到代码中以下功能模块：

-   [log](https://github.com/thu-db/huadb/tree/main/src/log)：日志相关类

    -   [log_record](https://github.com/thu-db/huadb/tree/main/src/log/log_record.h)：各类日志的抽象类，用于衍生出各种具体日志，已经完成。
    -   [log_manager](https://github.com/thu-db/huadb/tree/main/src/log/log_manager.h)：日志管理器，负责日志的记录以及故障恢复的具体执行过程，需要补充恢复算法。
    -   [log_records](https://github.com/thu-db/huadb/tree/main/src/log/log_records)：各类具体的物理日志类，需要补充其中记录增删改日志的重做和撤销。

-   [table](https://github.com/thu-db/huadb/tree/main/src/table)：数据表相关类
    -   [table](https://github.com/thu-db/huadb/tree/main/src/table/table.h)：在实验 1 的基础上添加记录变更时的日志记录功能。

根据往年经验，本次实验难度较大，请尽早开始实验。

本次实验依赖于实验 1，请确保完成实验 1 再开始本次实验。

相关功能模块的示意图如下：

![](../pics/lab2-details.svg)
