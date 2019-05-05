# MySQL · 引擎特性 · InnoDB Undo Log

[原文](http://mysql.taobao.org/monthly/2015/04/01/)

本文是对整个 Undo 生命周期过程的阐述，代码分析基于当前最新的 MySQL 5.7 版本。本文也可以作为了解整个 Undo 模块的代码导读。由于涉及到的模块众多，因此部分细节并未深入。

## 前言

`Undo Log 是 InnoDB MVCC 事务特性的重要组成部分`。当我们对记录做了变更操作时就会产生 undo 记录，undo 记录默认被记录到系统表空间（`ibdata`）中，但从 5.6 开始，也可以使用独立的 Undo 表空间。

Undo 记录中存储的是老版本数据，`当一个旧的事务需要读取数据时，为了能读取到老版本的数据，需要顺着 undo 链找到满足其可见性的记录`。当版本链很长时，通常可以认为这是个比较耗时的操作（例如：[bug#69812](https://bugs.mysql.com/bug.php?id=69812)）。

大多数对数据的变更操作包括 **INSERT/DELETE/UPDATE**，**`其中 INSERT 操作在事务提交前只对当前事务可见，因此产生的 Undo 日志可以在事务提交后直接删除`**（谁会对刚插入的数据有可见性需求呢！！），而对于 UPDATE/DELETE 则需要维护多版本信息，**`在 InnoDB 里，UPDATE 和 DELETE 操作产生的 Undo 日志会被归成一类，即 update_undo`**。

## 基本文件结构

为了保证事务并发操作时，写各自的 Undo Log 不产生冲突，**InnoDB 采用回滚段的方式来维护 Undo Log 的并发写入和持久化**。回滚段实际上是一种 Undo 文件组织方式，**每个回滚段又有多个 `Undo Log Slot`**。具体的文件组织方式如下图所示：

![](https://raw.githubusercontent.com/CHXU0088/github_libraries/master/Pic/undo_segment_20190505.png)

上图展示了基本的 Undo 回滚段布局结构，其中:

1. **`rseg0 预留在系统表空间 ibdata 中`**；

2. **`rseg1 ~ rseg32 这32个回滚段存放于临时表的系统表空间中`**；

3. **`rseg33 ~ 则可根据配置存放到独立 Undo 表空间中`**（如果没有打开独立 Undo 表空间，则仍存放于系统表空间 ibdata 中）；

**如果我们使用独立 Undo Tablespace，则总是从第一个 Undo Space 开始轮询分配 Undo 回滚段**。大多数情况下这是 OK 的，但假设我们将回滚段的个数从33开始依次递增配置到128，就可能导致所有的回滚段都存放在同一个 Undo Space 中。（参考函数 `trx_sys_create_rsegs` 以及 [bug#74471](https://bugs.mysql.com/bug.php?id=74471)）

**`每个回滚段维护了一个段头页，在该 Page 中又划分了 1024 个 slot（TRX_RSEG_N_SLOTS），每个 slot 又对应到一个 undo log 对象`**，因此**理论上 InnoDB 最多支持 96 * 1024 个普通事务**。

## 关键结构体

为了便于管理和使用 undo 记录，在内存中维持了如下关键结构体对象：

1. 所有回滚段都记录在 `trx_sys->rseg_array`，数组大小为**128**，分别对应不同的回滚段；

2. `rseg_array` 数组类型为 `trx_rseg_t`，用于维护回滚段相关信息；

3. 每个回滚段对象 `trx_rseg_t` 还要管理 undo log 信息，对应结构体为 `trx_undo_t`，使用多个链表来维护 `trx_undo_t` 信息；

4. 事务开启时，会专门给他指定一个回滚段，以后该事务用到的 undo log 页，就从该回滚段上分配;

5. 事务提交后，需要 purge 的回滚段会被放到 purge 队列上（`purge_sys->purge_queue`）。

各个结构体之间的联系如下：