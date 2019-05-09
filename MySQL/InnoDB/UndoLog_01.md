# MySQL · 引擎特性 · InnoDB Undo Log

[原文](http://mysql.taobao.org/monthly/2015/04/01/)

本文是对整个 Undo 生命周期过程的阐述，代码分析基于当前最新的 MySQL 5.7 版本。本文也可以作为了解整个 Undo 模块的代码导读。由于涉及到的模块众多，因此部分细节并未深入。

## 前言

`Undo Log 是 InnoDB MVCC 事务特性的重要组成部分`。当我们对记录做了变更操作时就会产生 undo 记录，undo 记录默认被记录到系统表空间（`ibdata`）中，但从 5.6 开始，也可以使用独立的 Undo 表空间。

Undo 记录中存储的是老版本数据，`当一个旧的事务需要读取数据时，为了能读取到老版本的数据，需要顺着 undo 链找到满足其可见性的记录`。当版本链很长时，通常可以认为这是个比较耗时的操作（例如：[bug#69812](https://bugs.mysql.com/bug.php?id=69812)）。

大多数对数据的变更操作包括 **INSERT/DELETE/UPDATE**，**`其中 INSERT 操作在事务提交前只对当前事务可见，因此产生的 Undo 日志可以在事务提交后直接删除`**（谁会对刚插入的数据有可见性需求呢！！），而对于 UPDATE/DELETE 则需要维护多版本信息，**`在 InnoDB 里，UPDATE 和 DELETE 操作产生的 Undo 日志会被归成一类，即 update_undo`**。

## 基本文件结构

为了保证事务并发操作时，写各自的 Undo Log 不产生冲突，**InnoDB 采用回滚段的方式来维护 Undo Log 的并发写入和持久化**。回滚段实际上是一种 Undo 文件组织方式，**每个回滚段又有多个 `Undo Log Slot`**。具体的文件组织方式如下图所示：

![](https://raw.githubusercontent.com/CHXU0088/github_libraries/master/Pic/MySQL/undo_segment_20190505.png)

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

![](https://raw.githubusercontent.com/CHXU0088/github_libraries/master/Pic/MySQL/trx_sys_rseg_undo_20150401.png)

## 回滚段的分配

`当开启一个读写事务时（或者从只读事务转换为读写事务），我们需要预先为事务分配一个回滚段`：

对于只读事务，如果产生对临时表的写入，则需要为其分配回滚段，使用**临时表回滚段（第1~32号回滚段）**，函数入口：`trx_assign_rseg --> trx_assign_rseg_low --> get_next_noredo_rseg`。

**`在 MySQL 5.7 中事务默认以只读事务开启，当随后判定为读写事务时，则转换成读写模式，并为其分配事务ID和回滚段`**，调用函数：`trx_set_rw_mode --> trx_assign_rseg_low --> get_next_redo_rseg`。

普通回滚段的分配方式如下：

1. 采用 **round-robin** 的轮询方式来赋予回滚段给事务，如果回滚段被标记为 **skip_allocation**（`这个 undo tablespace 太大了，purge 线程需要对其进行 truncate 操作`），则跳到下一个；

2. 选择一个回滚段给事务后，会将该回滚段的 `rseg->trx_ref_count` 递增，这样该回滚段所在的 undo tablespace 文件就不可以被 truncate 掉；

3. 临时表回滚段被赋予 `trx->rsegs->m_noredo`，普通读写操作的回滚段被赋予 `trx->rsegs->m_redo`；**如果事务在只读阶段使用到临时表，随后转换成读写事务，那么会为该事务分配两个回滚段**。

## 回滚段的使用

当产生数据变更时，我们需要使用 Undo Log 记录下变更前的数据以维护多版本信息。**`Insert 和 Delete/Update 分开记录 Undo，因此需要从回滚段单独分配 Undo Slot`**。

入口函数：`trx_undo_report_row_operation`

流程如下：

1. 判断当前变更的是否是临时表，如果是临时表，则采用临时表回滚段来分配，否则采用普通的回滚段；

2. 临时表操作记录 Undo 时不写 redo log；

3. 操作类型为 **`TRX_UNDO_INSERT_OP`**，且未分配 **`Insert Undo Slot`** 时，调用函数 `trx_undo_assign_undo` 进行分配；

4. 操作类型为 **`TRX_UNDO_MODIFY_OP`**，且未分配 **`Update Undo Slot`** 时，调用函数 `trx_undo_assign_undo` 进行分配。

我们来看看函数 `trx_undo_assign_undo` 的流程：

1. 首先总是从 **`cahced list`** 上分配 **trx_undo_t**（函数 `trx_undo_reuse_cached`，**当满足某些条件时，事务提交时会将其拥有的 `trx_undo_t` 放到 `cached list` 上，这样新的事务可以重用这些 undo 对象，而无需去扫描回滚段**，寻找可用的 slot，在后面的事务提交一节会介绍到）；

    * 对于 **INSERT**，从 `trx_rseg_t::insert_undo_cached` 上获取，并修改头部重用信息（`trx_undo_insert_header_reuse`）及预留XID空间（`trx_undo_header_add_space_for_xid`）

    * 对于 **DELETE/UPDATE**，从 `trx_rseg_t::update_undo_cached` 上获取，并在 `Undo Log HDR Page` 上创建新的 `Undo Log Header`（`trx_undo_header_create`），及预留XID存储空间（`trx_undo_header_add_space_for_xid`）

    * 获取到 **trx_undo_t** 对象后，会从 cached list 上移除掉。并初始化 `trx_undo_t` 相关信息（`trx_undo_mem_init_for_reuse`），将 `trx_undo_t::state` 设置为 **TRX_UNDO_ACTIVE**

2. 如果没有 cache 的 trx_undo_t，则需要从回滚段上分配一个空闲的 Undo Slot（`trx_undo_create`），并创建对应的 Undo 页，进行初始化；

    * **`一个回滚段可以支持 1024 个事务并发`**，如果不幸回滚段都用完了（通常这几乎不会发生），会返回错误 **DB_TOO_MANY_CONCURRENT_TRXS**

    * 每一个 `Undo Log Segment` 实际上对应一个独立的段，段头的起始位置在 UNDO 头 Page 的 `TRX_UNDO_SEG_HDR + TRX_UNDO_FSEG_HEADER` 偏移位置（见下图）

3. **已分配给事务的 `trx_undo_t` 会加入到链表 `trx_rseg_t::insert_undo_list` 或者 `trx_rseg_t::update_undo_list上`**；

4. 如果是数据词典操作（DDL）产生的 Undo，主要是表级别操作，例如创建或删除表，还需要记录操作的 `table id` 到 `Undo Log Header`中（`TRX_UNDO_TABLE_ID`），同时将 `TRX_UNDO_DICT_TRANS` 设置为 **TRUE**（`trx_undo_mark_as_dict_operation`）。

总的来说，**Undo Header Page** 主要包括如下信息： 

![](https://raw.githubusercontent.com/CHXU0088/github_libraries/master/Pic/MySQL/undo_log_header_page_20190401.png)

## Undo 日志的写入

入口函数：`trx_undo_report_row_operation`

当分配了一个 Undo Slot，同时初始化完可用的空闲区域后，就可以向其中写入 Undo 记录了。写入的 `page no` 取自 `undo->last_page_no`，初始情况下和 `hdr_page_no` 相同。

对于 **INSERT_UNDO**，调用函数 `trx_undo_page_report_insert` 进行插入，记录格式大致如下图所示：

![](https://raw.githubusercontent.com/CHXU0088/github_libraries/master/Pic/MySQL/insert_undo_format_20150401.png)

对于 **UPDATE_UNDO**，调用函数 `trx_undo_page_report_modify` 进行插入，记录格式大致如下图所示：

![](https://raw.githubusercontent.com/CHXU0088/github_libraries/master/Pic/MySQL/update_undo_format_20150401.png)

**在写入的过程中，可能出现单页面空间不足的情况，导致写入失败**，我们需要将刚刚写入的区域清空重置（`trx_undo_erase_page_end`），同时申请一个新的page（`trx_undo_add_page`）加入到 undo segment 上，同时将 `undo->last_page_no` 指向新分配的 page，然后重试。

完成 `Undo Log` 写入后，构建新的回滚段指针并返回（`trx_undo_build_roll_ptr`），**`回滚段指针包括 undo log 所在的回滚段id、日志所在的 page no、以及 page 内的偏移量`，需要记录到聚集索引记录中**。

## 事务 Prepare 阶段

入口函数：`trx_prepare_low`

**`当事务完成需要提交时，为了和 BINLOG 做 XA，InnoDB 的 commit 被划分成了两个阶段：prepare 阶段和 commit 阶段`**，本小节主要讨论下 prepare 阶段 undo 相关的逻辑。

为了在崩溃重启时知道事务状态，需要将事务设置为 **prepare**，MySQL 5.7 对临时表 undo 和普通表 undo 做了分别处理，前者在写 undo 日志时总是不需要记录 redo，后者则需要记录。

分别设置 `insert undo` 和 `update undo` 的状态为 **prepare**，调用函数 `trx_undo_set_state_at_prepare`，过程也比较简单，找到 **undo log slot** 对应的头页面（`trx_undo_t::hdr_page_no`)，将页面段头的 **TRX_UNDO_STATE** 设置为 `TRX_UNDO_PREPARED`，同时修改其他对应字段，如下图所示（对于外部显式XA 所产生的 XID，这里不做讨论）：

