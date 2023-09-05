# 数据库课程实验（开发中）

本文档尚处于开发状态，将于近期正式发布。

本课程实验框架进行了模块划分，每次实验关注 1-2 个功能模块，整体层次划分如下，左侧为功能示意图，右侧为层次架构图：

![](./pics/architecture.svg)

## 代码结构

实验代码主要包含以下几个模块：

-   binder: 语义解析
-   catalog: 系统表
-   common: 工具模块，包含字符串处理函数、异常相关类等
-   database: 数据库引擎
-   executors: 查询执行模块
-   log: 日志模块
-   operators: 查询计划树节点
-   optimizer: 优化器
-   planner: 查询计划生成模块
-   stoarge: 存储模块
-   table: 表相关类及函数
-   transaction: 事务模块
