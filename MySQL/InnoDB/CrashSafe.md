# MySQL · 引擎特性 · InnoDB 崩溃恢复过程

[原文](http://mysql.taobao.org/monthly/2015/06/01/)

在前面两期月报中，我们详细介绍了 InnoDB [Redo Log](http://mysql.taobao.org/monthly/2015/05/01/) 和 [Undo Log](http://mysql.taobao.org/monthly/2015/04/01/) 的相关知识，本文将介绍 InnoDB 在崩溃恢复时的主要流程。

本文代码分析基于 `MySQL 5.7.7-RC` 版本，函数入口为：`innobase_start_or_create_for_mysql`，这是一个非常冗长的函数，本文只涉及和崩溃恢复相关的代码。

## 初始化崩溃恢复

首先初始化崩溃恢复所需要的内存对象：

``` c
recv_sys_create();
recv_sys_init(buf_pool_get_curr_size());
```

**`当 InnoDB 正常 Shutdown，在 Flush Redo Log 和脏页后，会做一次完全同步的 checkpoint，并将 checkpoint 的 LSN 写到 ibdata 的第一个 page 中（fil_write_flushed_lsn）`**。

在重启实例时，会打开系统表空间 ibdata，并读取存储在其中的 LSN：

``` c
err = srv_sys_space.open_or_create(false, &sum_of_new_sizes, &flushed_lsn);
```

**`上述调用将从 ibdata 中读取的 LSN 存储到变量 flushed_lsn 中，表示上次 shutdown 时的 checkpoint 点，在后面做崩溃恢复时会用到。另外这里也会将 double write buffer 内存储的 page 载入到内存中（buf_dblwr_init_or_load_pages），如果 ibdata 的第一个 page 损坏了，就从 dblwr 中恢复出来`**。

Tips：注意在 MySQL 5.6.16 之前的版本中，如果 InnoDB 表空间的第一个 page 损坏了，就认为无法确定这个表空间的 space id，也就无法决定使用 dblwr 中的哪个 page 来进行恢复，InnoDB 将崩溃恢复失败（bug#70087）， 由于**每个数据页上都存储着表空间id**，因此后面将这里的逻辑修改成往后多读几个page，并尝试不同的 page size，直到找到一个完好的数据页，（参考函数 `Datafile::find_space_id()`）。**因此为了能安全的使用 `double write buffer` 保护数据，建议使用 5.6.16 及之后的 MySQL 版本**。


## 恢复 truncate 操作

**`为了保证对 undo log 独立表空间和用户独立表空间进行 truncate 操作的原子性，InnoDB 采用文件日志的方式为每个 truncate 操作创建一个独特的文件，如果在重启时这个文件存在，说明上次 truncate 操作还没完成实例就崩溃了，在重启时，我们需要继续完成 truncate 操作`**。

这一块的崩溃恢复是独立于 `redo log` 系统之外的。

对于 `undo log` 表空间恢复，在初始化 undo 子系统时完成：

``` c
err = srv_undo_tablespaces_init(
        create_new_db,
        srv_undo_tablespaces,
        &srv_undo_tablespaces_open);
```

对于用户表空间，扫描数据目录，找到 truncate 日志文件：`如果文件中没有任何数据，表示 truncate 还没开始；如果文件中已经写了一个 MAGIC NUM，表示 truncate 操作已经完成了；这两种情况都不需要处理`。

``` c
err = TruncateLogParser::scan_and_parse(srv_log_group_home_dir);
```

**`但对用户表空间 truncate 操作的恢复是 redo log apply 完成后才进行的，这主要是因为恢复 truncate 可能涉及到系统表的更新操作（例如重建索引），需要在 redo apply 完成后才能进行`**。

## 进入 Redo 崩溃恢复逻辑

入口函数：

```
err = recv_recovery_from_checkpoint_start(flushed_lsn);
```

传递的参数 `flushed_lsn` 即为从 ibdata 第一个 page 读取的 LSN，主要包含以下几步：

**Step 1**：为每个 `buffer pool instance` 创建一棵红黑树，指向 `buffer_pool_t::flush_rbt`，**主要用于加速插入 `flush list`（`buf_flush_init_flush_rbt`）**；

**Step 2**：**读取存储在第一个 `Redo Log` 文件头的 `CHECKPOINT LSN`，并根据该 LSN 定位到 redo 日志文件中对应的位置，从该 checkpoint 点开始扫描**。

在这里会调用三次 `recv_group_scan_log_recs` 扫描 redo log 文件：

1. 第一次的目的是找到 **`MLOG_CHECKPOINT`** 日志

    MLOG_CHECKPOINT 日志中记录了 CHECKPOINT LSN，**当该日志中记录的 LSN 和日志头中记录的 CHECKPOINT LSN 相同时，表示找到了符合的MLOG_CHECKPOINT LSN，将扫描到的LSN号记录到 `recv_sys->mlog_checkpoint_lsn` 中**（在 5.6 版本里没有这一次扫描）。

    **`MLOG_CHECKPOINT`** 在 [WL#7142](https://dev.mysql.com/worklog/task/?id=7142) 中被引入，其目的是为了简化 InnoDB 崩溃恢复的逻辑，根据 WL#7142 的描述，包含几点改进：

    （1）避免崩溃恢复时读取每个 ibd 的第一个 page 来确认其 space id;

    （2）无需检查 `$datadir/*.isl`，新的日志类型记录了文件全路径，并消除了 isl 文件和实际 ibd 目录的不一致可能带来的问题；

    （3）自动忽略那些还没有导入到 InnoDB 的 ibd 文件（例如在执行 `IMPORT TABLESPACE` 时 Crash）;

    （4）引入了新的日志类型 MLOG_FILE_DELETE 来跟踪 ibd 文件的删除操作。

    这里可能会产生的问题是，如果 MLOG_CHECKPOINT 日志和文件头记录的 CHECKPOINT LSN 差距太远的话，在第一次扫描时可能花费大量的时间做无谓的解析，感觉这里还有优化的空间。

    在我的测试案例中，由于崩溃时施加的负载比较大，MLOG_CHECKPOINT 和 CHECKPOINT 点的 LSN 相差约 1G 的 Redo Log。

2. 第二次扫描，再次从 checkpoint 点开始重复扫描，存储日志对象

    日志解析后的对象类型为 **`recv_t`**，包含：日志类型、长度、数据、开始和结束 LSN。**日志对象的存储使用 hash 结构，根据 space id 和 page no 计算 hash 值，相同页上的变更作为链表节点链在一起**，大概结构可以表示为：

    ![](https://raw.githubusercontent.com/CHXU0088/github_libraries/master/Pic/MySQL/recv_hash.png)

    扫描的过程中，会基于 MLOG_FILE_NAME 和 MLOG_FILE_DELETE 这样的 Redo 日志记录来构建 `recv_spaces`，存储 space id 到文件信息的映射（`fil_name_parse –> fil_name_process`），这些文件可能需要进行崩溃恢复（实际上第一次扫描时，也会向 `recv_spaces` 中插入数据，但只到 MLOG_CHECKPOINT 日志记录为止）。

    Tips：**`在一次 checkpoint 后第一次修改某个表的数据时，总是先写一条 MLOG_FILE_NAME 日志记录；通过该类型的日志可以跟踪一次 CHECKPOINT 后修改过的表空间，避免打开全部表。`** 在第二次扫描时，总会判断将要修改的表空间是否在 recv_spaces 中，如果不存在，则认为产生了严重的错误，拒绝启动（`recv_parse_or_apply_log_rec_body`）。

    默认情况下，Redo Log 以一批 64KB（RECV_SCAN_SIZE）为单位读入到 `log_sys->buf` 中，然后调用函数 `recv_scan_log_recs` 处理日志块。这里会判断到日志块的有效性：是否是完整写入的、日志块 checksum 是否正确，另外也会根据一些标记位来做判断：

    * 在每次写入 Redo Log 时，总会将写入的起始 block 头的 `flush bit` 设置为 true，表示一次写入的起始位置，因此在重启扫描日志时，也会根据 flush bit 来推进扫描的 LSN 点；

    * 每次写 Redo 时，还会在每个 block 上记录下一个 `checkpoint no`（每次做 checkpoint 都会递增），由于日志文件是循环使用的，因此需要根据 `checkpoint no` 判断是否读到了老旧的 Redo 日志。

    对于合法的日志，会拷贝到缓冲区 `recv_sys->buf` 中，调用函数 `recv_parse_log_recs` 解析日志记录。这里会根据不同的日志类型分别进行处理，并尝试进行 apply，堆栈为：

    ``` c
    recv_parse_log_recs
        --> recv_parse_log_rec
            --> recv_parse_or_apply_log_rec_body
    ```

    如果想理解 InnoDB 如何基于不同的日志类型进行崩溃恢复的，非常有必要细读函数 `recv_parse_or_apply_log_rec_body`，这里是 Redo 日志 apply 的入口。

    例如：如果解析到的日志类型为 `MLOG_UNDO_HDR_CREATE`，就会从日志中解析出事务ID，为其重建 Undo Log 头（`trx_undo_parse_page_header`）；如果是一条插入操作标识（`MLOG_REC_INSERT` 或者 `MLOG_COMP_REC_INSERT`），就需要从中解析出索引信息（`mlog_parse_index`）和记录信息（`page_cur_parse_insert_rec`）；或者解析一条 `IN-PLACE UPDATE`（`MLOG_REC_UPDATE_IN_PLACE`）日志，则调用函数 `btr_cur_parse_update_in_place`。

    **`第二次扫描只会应用 MLOG_FILE_* 类型的日志，记录到 recv_spaces 中，对于其他类型的日志在解析后存储到哈希对象里`**。然后调用函数 `recv_init_crash_recovery_spaces` 对涉及的表空间进行初始化处理：

    * 首先会打印两条我们非常熟悉的日志信息：

    ```
    [Note] InnoDB: Database was not shutdown normally!
    [Note] InnoDB: Starting crash recovery.
    ```

    * 如果 `recv_spaces` 中的表空间未被删除，且 ibd 文件存在时，则表明这是个普通的文件操作，将该 table space 加入到 `fil_system->named_spaces` 链表上（`fil_names_dirty`），后续可能会对这些表做 redo apply 操作；

    * 对于已经被删除的表空间，我们可以忽略日志apply，将对应表的space id在recv_sys->addr_hash上的记录项设置为RECV_DISCARDED;

    * 调用函数buf_dblwr_process()，该函数会检查所有记录在double write buffer中的page，其对应的数据文件页是否完好，如果损坏了，则直接从dblwr中恢复;

    * 最后创建一个临时的后台线程，线程函数为recv_writer_thread，这个线程和page cleaner线程配合使用，它会去通知page cleaner线程去flush崩溃恢复产生的脏页，直到recv_sys中存储的redo记录都被应用完成并彻底释放掉(recv_sys->heap == NULL)

3. 如果第二次扫描 hash 表空间不足，无法全部存储到 hash 表中，则发起第三次扫描，清空 hash，重新从 checkpoint 点开始扫描

    hash 对象的空间最大一般为 `buffer pool size - 512` 个 page 大小。

    **第三次扫描不会尝试一起全部存储到 hash 里，而是一旦发现 hash 不够了，就立刻 apply redo 日志**。但是......如果总的日志需要存储的 hash 空间略大于可用的最大空间，那么一次额外的扫描开销还是非常明显的。

**`简而言之，第一次扫描找到正确的 MLOG_CHECKPOINT 位置；第二次扫描解析 Redo 日志并存储到 hash 中；如果 hash 空间不够用，则再来一轮重新开始，解析一批，应用一批`**。

三次扫描后，hash 中通常还有 Redo 日志没有被应用掉。这个留在后面来做，随后将 `recv_sys->apply_log_recs` 设置为 **true**，并从函数 `recv_recovery_from_checkpoint_start` 返回。

**`对于正常 shutdown 的场景，一次 checkpoint 完成后是不记录 MLOG_CHECKPOINT 日志的，如果扫描过程中没有找到对应的日志，那就认为上次是正常 shutdown 的`**，不用考虑崩溃恢复了。

Tips：偶尔我们会看到日志中报类似这样的信息：**"The log sequence number xxx in the system tablespace does not match the log sequence number xxxx in the ib_logfiles!"** **`从内部逻辑来看是因为 ibdata 中记录的 LSN 和 ib_logfile 中记录的 checkpoint lsn 不一致，但系统又判定无需崩溃恢复时会报这样的错。单纯从 InnoDB 实例来看是可能的，因为做 checkpint 和更新 ibdata 不是原子的操作`**，这样的日志信息一般我们也是可以忽略的。

## 初始化事务子系统（`trx_sys_init_at_db_start`）

这里会涉及到读入 Undo 相关的系统页数据，在崩溃恢复状态下，所有的 page 都要先进行日志 apply 后，才能被调用者使用，例如如下堆栈：

``` c
trx_sys_init_at_db_start
    --> trx_sysf_get -->
        ....->buf_page_io_complete --> recv_recover_page
```

因此 **`在初始化回滚段的时候，我们通过读入回滚段页并进行 redo log apply，就可以将回滚段信息恢复到一致的状态，从而能够“复活”在系统崩溃时活跃的事务，并维护到读写事务链表中`**。对于处于 `prepare` 状态的事务，我们后续需要做额外处理。

关于事务如何从崩溃恢复中复活，参阅4月份的月报《MySQL · 引擎特性 · InnoDB Undo Log 漫游》最后一节。

## 应用 Redo 日志（`recv_apply_hashed_log_recs`）

根据之前搜集到的 `recv_sys->addr_hash` 中的日志记录，依次将 page 读入内存，并对每个 page 进行崩溃恢复操作（`recv_recover_page_func`）：

* 已经被删除的表空间，直接跳过其对应的日志记录；

* 在读入需要恢复的文件页时，会主动尝试采用预读的方式多读点 page（`recv_read_in_area`），搜集最多连续 32 个（`RECV_READ_AHEAD_AREA`）需要做恢复的 page no，然后发送异步读请求。page 读入 buffer pool 时，会主动做崩溃恢复逻辑；

* **`只有 LSN 大于数据页上 LSN 的日志才会被 Apply`**; 忽略被 truncate 的表的 Redo 日志；

* **在恢复数据页的过程中不产生新的 Redo 日志**；

* 在完成修复 page 后，需要将脏页加入到 `buffer pool` 的 `flush list`上；由于 InnoDB 需要保证 `flush list` 的有序性，而崩溃恢复过程中修改 page 的 LSN 是基于 Redo 的 LSN 而不是全局的 LSN，无法保证有序性；**InnoDB 另外维护了一颗红黑树来维持有序性，每次插入到 flush list 前，查找红黑树找到合适的插入位置，然后加入到 flush list 上（`buf_flush_recv_note_modification`）**。

## 完成崩溃恢复（`recv_recovery_from_checkpoint_finish`）

在完成所有 Redo 日志 Apply 后，基本的崩溃恢复也完成了，此时可以释放资源，等待 `recv writer` 线程退出（崩溃恢复产生的脏页已经被清理掉），释放红黑树，回滚所有数据词典操作产生的非 prepare 状态的事务（`trx_rollback_or_clean_recovered`）。

### 无效数据的清理及事务回滚：

调用函数 `recv_recovery_rollback_active` 完成下述工作：

* 删除临时创建的索引，例如在 DDL 创建索引时 Crash 残留的临时索引（`row_merge_drop_temp_indexes()`）；

* 清理 InnoDB 临时表（`row_mysql_drop_temp_tables`）；

* 清理全文索引的无效的辅助表（`fts_drop_orphaned_tables()`）；

* 创建后台线程，线程函数为 `trx_rollback_or_clean_all_recovered`，和在 `recv_recovery_from_checkpoint_finish` 中的调用不同，该后台线程会回滚所有不处于 prepare 状态的事务。

至此 InnoDB 层的崩溃恢复算是告一段落，只剩下处于 prepare 状态的事务还有待处理，而这一部分需要和 Server 层的 Binlog 联合来进行崩溃恢复。

## Binlog/InnoDB XA Recover

回到 Server 层，在初始化完了各个存储引擎后，如果 Binlog 打开了，我们就可以通过 Binlog 来进行 XA恢复:

* 首先扫描最后一个 Binlog 文件，找到其中所有的 XID 事件，并将其中的 XID 记录到一个 hash 结构中（**`MYSQL_BIN_LOG::recover`**）；

* 然后**对每个引擎调用接口函数 `xarecover_handlerton`，拿到每个事务引擎中处于 prepare 状态的事务 xid，如果这个 xid 存在于 binlog 中，则提交；否则回滚事务**。

很显然，如果我们弱化配置的持久性（`innodb_flush_log_at_trx_commit != 1` 或者 `sync_binlog != 1`）， 宕机可能导致两种丢数据的场景：

1. 引擎层提交了，但 binlog 没写入，备库丢事务；

2. 引擎层没有 prepare，但 binlog 写入了，主库丢事务；

即使我们将参数设置成 `innodb_flush_log_at_trx_commit =1` 和 `sync_binlog = 1`，也还会面临这样一种情况：**`主库 crash 时还有 binlog 没传递到备库，如果我们直接提升备库为主库，同样会导致主备不一致，老主库必须根据新主库重做，才能恢复到一致的状态。针对这种场景，我们可以通过开启 semisync 的方式来解决`**，一种可行的方案描述如下：

1. 设置双1强持久化配置;

2. 将 semisync 的超时时间设到极大值，同时使用 `semisync AFTER_SYNC` 模式，**即用户线程在写入 Binlog 后，引擎层提交前等待备库 ACK**；

3. 基于步骤1的配置，我们可以保证在主库 crash 时，所有老主库比备库多出来的事务都处于 prepare 状态；

4. 备库完全 Apply 日志后，记下其执行到的 Relay Log 对应的位点，然后将备库提升为新主库；

5. 将老主库的最后一个 Binlog 进行截断，截断的位点即为步骤4记录的位点;

6. 启动老主库，那些已经传递到备库的事务都会提交掉，未传递到备库的 Binlog 都会回滚掉。

------