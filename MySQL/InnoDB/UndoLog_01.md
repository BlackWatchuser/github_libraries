# MySQL · 引擎特性 · InnoDB Undo Log

[原文](http://mysql.taobao.org/monthly/2015/04/01/)

本文是对整个 Undo 生命周期过程的阐述，代码分析基于当前最新的 MySQL 5.7 版本。本文也可以作为了解整个 Undo 模块的代码导读。由于涉及到的模块众多，因此部分细节并未深入。

## 前言

`Undo Log 是 InnoDB MVCC 事务特性的重要组成部分`。当我们对记录做了变更操作时就会产生 undo 记录，undo 记录默认被记录到系统表空间（`ibdata`）中，但从 5.6 开始，也可以使用独立的 Undo 表空间。

Undo 记录中存储的是老版本数据，`当一个旧的事务需要读取数据时，为了能读取到老版本的数据，需要顺着 undo 链找到满足其可见性的记录`。当版本链很长时，通常可以认为这是个比较耗时的操作（例如：[bug#69812](https://bugs.mysql.com/bug.php?id=69812)）。

大多数对数据的变更操作包括 **INSERT/DELETE/UPDATE**，**`其中 INSERT 操作在事务提交前只对当前事务可见，因此产生的 Undo 日志可以在事务提交后直接删除`**（谁会对刚插入的数据有可见性需求呢！！），而对于 UPDATE/DELETE 则需要维护多版本信息，**`在 InnoDB 里，UPDATE 和 DELETE 操作产生的 Undo 日志会被归成一类，即 update_undo`**。

## 基本文件结构

为了保证事务并发操作时，写各自的 Undo Log 不产生冲突，**InnoDB 采用回滚段的方式来维护 Undo Log 的并发写入和持久化**。回滚段实际上是一种 Undo 文件组织方式，**每个回滚段又有多个 `Undo Log Slot`**。具体的文件组织方式如下图所示：

