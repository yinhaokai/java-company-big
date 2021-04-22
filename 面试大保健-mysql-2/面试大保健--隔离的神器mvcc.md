什么是mvcc？
mvcc （muti-version-currency-control）多版本并发控制，是在mysql数据中用于提交读和可重复读用于控制版本的，主要是指回滚。

如何实现mvcc？
在介绍如何实现mvcc之前，首先我们得学习几个概念，首先是一致性视图和undolog和 redolog。

mvcc并发版本控制是在写时去读才使用的，且数据库的隔离级别必须为rr或者rc。

mvcc的原理
多了两列
事务id 和指向之前事务的指针

undolog
用于还原行数据
readview
视图
可重复读是事务一开始的时候就创建了一个视图
提交读是每个语句创建一个视图





