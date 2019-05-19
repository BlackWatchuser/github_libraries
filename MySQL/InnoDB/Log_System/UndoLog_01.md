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

## 事务 Prepare

入口函数：`trx_prepare_low`

**`当事务完成需要提交时，为了和 BINLOG 做 XA，InnoDB 的 commit 被划分成了两个阶段：prepare 阶段和 commit 阶段`**，本小节主要讨论下 prepare 阶段 undo 相关的逻辑。

为了在崩溃重启时知道事务状态，需要将事务设置为 **prepare**，MySQL 5.7 对临时表 undo 和普通表 undo 做了分别处理，前者在写 undo 日志时总是不需要记录 redo，后者则需要记录。

分别设置 `insert undo` 和 `update undo` 的状态为 **prepare**，调用函数 `trx_undo_set_state_at_prepare`，过程也比较简单，找到 **undo log slot** 对应的头页面（`trx_undo_t::hdr_page_no`)，将页面段头的 **TRX_UNDO_STATE** 设置为 `TRX_UNDO_PREPARED`，同时修改其他对应字段，如下图所示（对于外部显式XA 所产生的 XID，这里不做讨论）：

![](https://raw.githubusercontent.com/CHXU0088/github_libraries/master/Pic/MySQL/trx_undo_hdr_page_no_20150401.png)

Tips：InnoDB 层的 XID 是如何获取的呢？**当 InnoDB 的参数 `innodb_support_xa` 打开时，在执行事务的第一条 SQL 时，就会去注册 XA，根据第一条 SQL 的 query id 拼凑 XID 数据，然后存储在事务对象中**。参考函数：`trans_register_ha`。

### 事务 Commit

**`当事务 commit 时，需要将事务状态设置为 COMMIT 状态，这里同样通过 Undo 来实现的`**。

入口函数：`trx_commit_low-->trx_write_serialisation_history`

在该函数中，**需要将该事务包含的 Undo 都设置为完成状态，先设置 insert undo，再设置 update undo（`trx_undo_set_state_at_finish`）**，完成状态包含三种：

* 如果当前的 undo log 只占一个 page，且占用的 header page 大小使用不足其 **3/4** 时（`TRX_UNDO_PAGE_REUSE_LIMIT`），则状态设置为 **`TRX_UNDO_CACHED`**，该 undo 对象随后会加入到 `undo cache list` 上；

* 如果是 insert_undo（undo 类型为 `TRX_UNDO_INSERT`），则状态设置为 **`TRX_UNDO_TO_FREE`**；

* 如果不满足 a 和 b，则表明该 Undo 可能需要 Purge 线程去执行清理操作，状态设置为 **`TRX_UNDO_TO_PURGE`**。

在确认状态信息后，写入 `undo header page` 的 **TRX_UNDO_STATE** 中。

如果当前事务包含 update undo，并且 Undo 所在回滚段不在 Purge 队列时，还需要将当前 Undo 所在的回滚段（及当前最大的事务号）加入 Purge 线程的 Purge 队列（`purge_sys->purge_queue`）中（参考函数：`trx_serialisation_number_get`）。

对于 update undo 需要调用 `trx_undo_update_cleanup` 进行清理操作，清理的过程包括：

1. 将 undo log 加入到 **`history list上`**，调用：`trx_purge_add_update_undo_to_history`：

    * 如果该 undo log 不满足 cache 的条件（状态为 `TRX_UNDO_CACHED`，如上述），则将其占用的 slot 设置为 **`FIL_NULL`**，意为 slot 空闲，同时更新回滚段头的 `TRX_RSEG_HISTORY_SIZE` 值，将当前 undo 占用的 page 数累加上去；

    * 将当前 undo 加入到回滚段的 **`TRX_RSEG_HISTORY`** 链表上，作为链表头节点，节点指针为 UNDO 头的 `TRX_UNDO_HISTORY_NODE`；

    * 更新 `trx_sys->rseg_history_len`（也就是 **show engine innodb status** 看到的 **history list**），如果只有普通的 update_undo，则加1，如果还有临时表的 update_undo，则加2，然后唤醒 purge 线程；

    * 将当前事务的 `trx_t::no` 写入 undo 头的 `TRX_UNDO_TRX_NO` 段；

    * 如果不是 **delete-mark** 操作，将 undo 头的 `TRX_UNDO_DEL_MARKS` 更新为 **false**;

    * 如果 undo 所在回滚段的 `rseg->last_page_no` 为 **FIL_NULL**，表示该回滚段的旧记录清理已经完成，进行如下赋值，记录这个回滚段上第一个需要 purge 的 undo 记录信息：

    ```
    rseg->last_page_no = undo->hdr_page_no;
    rseg->last_offset = undo->hdr_offset;
    rseg->last_trx_no = trx->no;
    rseg->last_del_marks = undo->del_marks;
    ```

2. 如果 undo 需要 cache，将 undo 对象放到回滚段的 **`update_undo_cached`** 链表上；否则释放 undo 对象（`trx_undo_mem_free`）。

注意：上面只清理了 **update_undo**，**insert_undo** 的清理直到事务释放记录锁、从读写事务链表清除、以及关闭 readview 后才进行，调用函数 `trx_undo_insert_cleanup`：

1. 如果 undo 状态为 **`TRX_UNDO_CACHED`**，则加入到回滚段的 **`insert_undo_cached`** 链表上；

2. 否则，将该 undo 所占的 segment 及其所占用的回滚段的 slot 全部释放掉（`trx_undo_seg_free`），修改当前回滚段的大小（`rseg->curr_size`），并释放 undo 对象所占的内存（`trx_undo_mem_free`），**`和 update_undo 不同，insert_undo 并未放到 History List 上`**。

事务完成提交后，需要将其使用的回滚段引用计数 `rseg->trx_ref_count` 减1；

## 事务的回滚

如果事务因为异常或被显式回滚，那么所有的数据变更都要改回去。这里就要借助 undo 日志中的数据来进行恢复了。

入口函数为：`row_undo_step --> row_undo`

操作也比较简单，析取老版本记录，做逆向操作即可：**`对于标记删除的记录清理标记删除标记；对于 in-place 更新，将数据回滚到最老版本；对于插入操作，直接删除聚集索引和二级索引记录（row_undo_ins）`**。

具体的操作中，**先回滚二级索引记录（`row_undo_mod_del_mark_sec`、`row_undo_mod_upd_exist_sec`、`row_undo_mod_upd_del_sec`），再回滚聚集索引记录（`row_undo_mod_clust`）**。这里不展开描述，可以参阅对应的函数。

## 多版本并发控制

InnoDB 的多版本使用 undo 来构建，这很好理解，undo 记录中包含了记录更改前的镜像，如果更改数据的事务未提交，对于隔离级别大于等于 **Read Committed** 的事务而言，它不应该看到已修改的数据，而是应该给它返回老版本的数据。

入口函数：`row_vers_build_for_consistent_read`

**`由于在修改聚集索引记录时，总是存储了 回滚段指针 和 事务id，可以通过该指针找到对应的 undo 记录，通过事务id来判断记录的可见性`**。当旧版本记录中的事务id对当前事务而言是不可见时，则继续向前构建，直到找到一个可见的记录或者到达版本链尾部（关于事务可见性及 read view，可以参阅我们之前的月报）。

Tips 1：**构建老版本记录（`trx_undo_prev_version_build`）需要持有 page latch，因此如果 Undo 链太长的话，其他请求该 page 的线程可能等待时间过长导致 Crash，最典型的就是备库备份场景**：

当备库使用 InnoDB 表存储复制位点信息时（`relay_log_info_repository=TABLE`），逻辑备份显式开启一个 read view 并且执行了长时间的备份时，这中间都无法对 **slave_relay_log_info** 表做 purge 操作，导致版本链极其长；当开始备份 **slave_relay_log_info** 表时，就需要去花很长的时间构建老版本；复制线程由于需要更新 slave_relay_log_info 表，因此会陷入等待 page latch 的场景，最终有可能导致信号量等待超时，实例自杀（[bug#74003](https://bugs.mysql.com/bug.php?id=74003)）。

Tips 2：在构建老版本的过程中，总是需要创建 heap 来存储旧版本记录，实际上这个 heap 是可以重用的，无需总是重复构建（[bug#69812](https://bugs.mysql.com/bug.php?id=69812)）。

Tips 3：如果回滚段类型是 INSERT，就完全没有必要去看 Undo 日志了，因为一个未提交事务的新插入记录，对其他事务而言总是不可见的。

Tips 4：对于聚集索引我们知道其记录中存有修改该记录的事务id，我们可以直接判断是否需要构建老版本（`lock_clust_rec_cons_read_sees`），但 **`对于二级索引记录，并未存储事务id，而是每次更新记录时，同时更新记录所在的 page 上的事务id（PAGE_MAX_TRX_ID），如果该事务id对当前事务是可见的，那么就无需去构建老版本了，否则就需要去回表查询对应的聚集索引记录，然后判断可见性（lock_sec_rec_cons_read_sees）`**。

## Purge 清理操作

从上面的分析我们可以知道：**`update_undo 产生的日志会放到 history list 中，当这些旧版本无人访问时，需要进行清理操作；另外页内标记删除的操作也需要从物理上清理掉`**。**后台 Purge 线程负责这些工作**。

入口函数：`srv_do_purge --> trx_purge`

1. 确认可见性

    在开始尝试 purge 前，Purge 线程会先克隆一个最老的活跃视图（`trx_sys->mvcc->clone_oldest_view`），所有在 read view 开启之前提交的事务所做的事务变更都是可以清理的。

2. 获取需要 purge 的 undo 记录（`trx_purge_attach_undo_recs`）

    从 history list 上读取多个 undo 记录，并分配到多个 Purge 线程的工作队列上（`(purge_node_t*) thr->child->undo_recs`），默认一次最多取 300 个 undo 记录，可通过参数 `innodb_purge_batch_size` 参数调整。

3. Purge 工作线程

    当完成任务的分发后，各个工作线程（包括协调线程）开始进行 purge 操作；入口函数：`row_purge_step -> row_purge -> row_purge_record_func`。

    主要包括两种：一种是记录直接被标记删除了，这时候需要物理清理所有的聚集索引和二级索引记录（`row_purge_record_func`）；另一种是 **`聚集索引 in-place 更新`** 了，但二级索引上的记录顺序可能发生变化，而**二级索引的更新总是标记删除 + 插入**，因此需要根据回滚段记录去检查二级索引记录序是否发生变化，并执行清理操作（`row_purge_upd_exist_or_extern`）。

4. 清理 history list

    从前面的分析我们知道，insert undo 在事务提交后，Undo Segment 就释放了。而 update undo 则加入了 history list，为了将这些文件空间回收重用，需要对其进行 **truncate** 操作；默认每处理 128 轮 Purge 循环后，Purge 协调线程需要执行一次 purge history list 操作。

    入口函数：`trx_purge_truncate --> trx_purge_truncate_history`

    从回滚段的 HISTORY 文件链表上开始遍历释放 Undo Log Segment，由于 history 链表是按照 trx no 有序的，因此遍历 truncate 直到完全清除，或者遇到一个还未 purge 的 undo log（trx no 比当前 purge 到的位置更大）时才停止。

关于 Purge 操作的逻辑实际上还算是比较复杂的代码模块，这里只是简单的介绍了下，以后有时间再展开描述。

## 崩溃恢复

**`当实例从崩溃中恢复时，需要将活跃的事务从 Undo 中提取出来，对于 ACTIVE 状态的事务直接回滚，对于 Prepare 状态的事务，如果该事务对应的 binlog 已经记录，则提交，否则回滚事务`**。

实现的流程也比较简单，首先先做 Redo（`recv_recovery_from_checkpoint_start`），Undo 是受 Redo 保护的，因此可以从 Redo 中恢复（临时表 Undo 除外，临时表 Undo 是不记录 Redo 的）。

在 Redo 日志应用完成后，完成初始化数据字典子系统（**dict_boot**），随后开始初始化事务子系统（`trx_sys_init_at_db_start`），Undo 段的初始化即在这一步完成。

在初始化 Undo 段时（`trx_sys_init_at_db_start -> trx_rseg_array_init -> ... -> trx_undo_lists_init`），会根据每个回滚段 page 中的 slot 是否被使用来恢复对应的 undo log，读取其状态信息和类型等信息，创建内存结构，并存放到每个回滚段的 undo list 上。

当初始化完成 Undo 内存对象后，就要据此来恢复崩溃前的事务链表了（`trx_lists_init_at_db_start`），根据每个回滚段的 **`insert_undo_list`** 来恢复插入操作的事务（`trx_resurrect_insert`），根据 **`update_undo_list`** 来恢复更新事务（`trx_resurrect_update`），如果既存在插入又存在更新，则只恢复一个事务对象。另外除了恢复事务对象外，还要恢复表锁及读写事务链表，从而恢复到崩溃之前的事务场景。

当从 Undo 恢复崩溃前活跃的事务对象后，会去开启一个后台线程来做事务回滚和清理操作（`recv_recovery_rollback_active -> trx_rollback_or_clean_all_recovered`），**`对于处于 ACTIVE 状态的事务直接回滚，对于既不 ACTIVE 也非 PREPARE 状态的事务，直接则认为其是提交的，直接释放事务对象。但完成这一步后，理论上事务链表上只存在 PREPARE 状态的事务`**。

随后很快我们进入 **XA Recover** 阶段，MySQL 使用 **`内部 XA，即通过 Binlog 和 InnoDB 做 XA 恢复。在初始化完成引擎后，Server 层会开始扫描最后一个 Binlog 文件，搜集其中记录的 XID（MYSQL_BIN_LOG::recover），然后和InnoDB 层的事务 XID 做对比。如果 XID 已经存在于 Binlog 中了，对应的事务需要提交；否则需要回滚事务`**。

Tips：为何只需要扫描最后一个 binlog 文件就可以了？因为 **`在每次 Rotate 到一个新的 binlog 文件之前，总是要保证前一个 binlog 文件中对应的事务都提交并且 sync redo 到磁盘了`**，也就是说，前一个 binlog 文件中的事务在崩溃恢复时肯定是出于提交状态的。

******