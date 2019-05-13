# MySQL · 引擎特性 · InnoDB 崩溃恢复过程

[原文](http://mysql.taobao.org/monthly/2015/06/01/)

在前面两期月报中，我们详细介绍了 InnoDB [Redo Log](http://mysql.taobao.org/monthly/2015/05/01/) 和 [Undo Log](http://mysql.taobao.org/monthly/2015/04/01/) 的相关知识，本文将介绍 InnoDB 在崩溃恢复时的主要流程。

本文代码分析基于 `MySQL 5.7.7-RC` 版本，函数入口为：`innobase_start_or_create_for_mysql`，这是一个非常冗长的函数，本文只涉及和崩溃恢复相关的代码。

## 一、初始化崩溃恢复

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

上述调用将**从 ibdata 中读取的 LSN 存储到变量 `flushed_lsn` 中，表示上次 shutdown 时的 checkpoint 点，在后面做崩溃恢复时会用到。另外这里也会将 `double write buffer` 内存储的 page 载入到内存中（`buf_dblwr_init_or_load_pages`），如果 ibdata 的第一个 page 损坏了，就从 `dblwr` 中恢复出来**。

Tips：注意在 MySQL 5.6.16 之前的版本中，如果 InnoDB 表空间的第一个 page 损坏了，就认为无法确定这个表空间的 space id，也就无法决定使用 dblwr 中的哪个 page 来进行恢复，InnoDB 将崩溃恢复失败（bug#70087）， 由于**每个数据页上都存储着表空间id**，因此后面将这里的逻辑修改成往后多读几个page，并尝试不同的 page size，直到找到一个完好的数据页，（参考函数 `Datafile::find_space_id()`）。**因此为了能安全的使用 `double write buffer` 保护数据，建议使用 5.6.16 及之后的 MySQL 版本**。


## 二、 恢复 truncate 操作

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

## 三、进入 Redo 崩溃恢复逻辑

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

![]()