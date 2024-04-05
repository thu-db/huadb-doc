# FAQ

Q: 运行 `lab2/20-recover.test` 测例时，在 restart 之后出现报错 "Table "recover" does not exist" 是什么原因？

A: 可能与 lab 1 的实现方式有关，如果在 lab 1 中采用反向链表连接页面（即链表指针从页面 n 指向页面 n - 1），由于实验框架的一些缺陷，目前没有对 first_page_id\_ 变量进行持久化，数据库重启后 Table 类的 first_page_id\_ 变量将被重置为 0，如采用反向链表则只能读到第 0 页的内容。如果在 lab 1 中使用了反向链表，需要将实现方式改为正向链表，从页面 n 指向页面 n + 1。
