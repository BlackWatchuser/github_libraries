# MySQL · 引擎特性 · InnoDB Redo Log

[原文](http://mysql.taobao.org/monthly/2015/05/01/)

## 前言

InnoDB 有两块非常重要的日志，一个是 **Undo Log**，另外一个是 **Redo Log**，`前者用来保证事务的原子性以及 InnoDB 的 MVCC，后者用来保证事务的持久性`。

和大多数关系型数据库一样，InnoDB 记录了对数据文件的物理更改，并保证总是日志先行，也就是所谓的 `WAL`，`即在持久化数据文件前，保证之前的 redo 日志已经写到磁盘`。

`LSN（log sequence number）` 用于记录日志序号，它是一个不断递增的 `unsigned long long` 类型整数。在 InnoDB 的日志系统中，LSN 无处不在，它既用于表示修改脏页时的日志序号，也用于记录 `checkpoint`，通过 LSN，可以具体的定位到其在 Redo Log 文件中的位置。

**为了管理脏页，在 Buffer Pool 的每个 instance上 都维持了一个 `flush list`，flush list 上的 page 按照修改这些 page 的LSN号进行排序。** `因此定期做 redo checkpoint 时，选择的 LSN 总是所有 bp instance 的 flush list 上最老的那个page（拥有最小的LSN）。`由于采用 WAL 的策略，每次事务提交时需要持久化 redo log 才能保证事务不丢。而延迟刷脏页则起到了合并多次修改的效果，避免频繁写数据文件造成的性能问题。

由于 InnoDB 日志组的特性已经被废弃（Redo 日志写多份），归档日志（InnoDB Archive Log）特性也在 5.7 被彻底移除，本文在描述相关逻辑时会忽略这些逻辑。另外限于篇幅，InnoDB 崩溃恢复的逻辑将在下期讲述，本文重点阐述 Redo Log 产生的生命周期以及 MySQL 5.7 的一些改进点。

本文的分析基于最新的 MySQL 5.7.7-RC 版本。

## InnoDB 日志文件

InnoDB 的 redo log 可以通过参数 `innodb_log_files_in_group` 配置成多个文件，另外一个参数 `innodb_log_file_size` 表示每个文件的大小。因此总的 redo log 大小为 **innodb_log_files_in_group * innodb_log_file_size**。

Redo Log 文件以 ib_logfile[number] 命名，日志目录可以通过参数 `innodb_log_group_home_dir` 控制。Redo log 以顺序的方式写入文件，写满时则回溯到第一个文件，进行覆盖写。（但`在做 redo checkpoint 时，也会更新第一个日志文件的头部 checkpoint 标记，所以严格来讲也不算顺序写`）。

```
   ----------------<<<---------------
   ⬇                               ⬆  
ib_logfile0 => ib_logfile1 => ib_logfile2
```

在 InnoDB 内部，逻辑上 ib_logfile[*] 被当成了一个文件，对应同一个 space_id。由于是**使用 512 字节 block 对齐写入文件**，可以很方便的根据全局维护的 LSN 号计算出要写入到哪一个文件以及对应的偏移量。

`Redo Log 文件是循环写入的，在覆盖写之前，总是要保证对应的脏页已经刷到了磁盘`。在非常大的负载下，Redo Log 可能产生的速度非常快，导致频繁的刷脏操作，进而导致性能下降，**通常在未做 checkpoint 的日志超过文件总大小的 `76%` 之后，InnoDB 认为这可能是个不安全的点，会强制的 preflush 脏页，导致大量用户线程 stall 住**。如果预期可能会有这样的场景，我们建议调大 Redo Log 文件的大小。可以做一次干净的 shutdown，然后修改 Redo Log 配置，重启实例。

除了 Redo Log 文件外，InnoDB 还有其他的日志文件，例如`为了保证 truncate 操作而产生的中间日志文件，包括 truncate innodb 表以及 truncate undo log tablespace，都会产生一个中间文件，来标识这些操作是成功还是失败`，**如果 truncate 没有完成，则需要在 crash recovery 时进行重做**。有意思的是，根据官方worklog 的描述，最初实现 truncate 操作的原子化时是通过增加新的 Redo Log 类型来实现的，但后来不知道为什么又改成了采用日志文件的方式，也许是考虑到低版本兼容的问题吧。

## 关键结构体

### log_sys 对象

**log_sys 是 InnoDB 日志系统的中枢及核心对象，控制着日志的拷贝、写入、checkpoint 等核心功能**。它同时也是大写入负载场景下的热点模块，是连接 InnoDB 日志文件及 log buffer 的枢纽，对应结构体为 `log_t`。

其中与 Redo Log 文件相关的成员变量包括：

| 变量名 | 描述 |
| :- | :- |
| log_groups | 日志组，当前版本仅支持一组日志，对应类型为 `log_group_t`，**包含了当前日志组的文件个数、每个文件的大小、space id 等信息** |
| lsn_t log_group_capacity | 表示**当前日志文件的总容量**，值为：(Redo Log 文件总大小 - Redo 文件个数 * LOG_FILE_HDR_SIZE) * 0.9，`LOG_FILE_HDR_SIZE` 为 4*512 字节 |
| lsn_t max_modified_age_async | **异步** `Preflush Dirty Page` 点 |
| lsn_t max_modified_age_sync | **同步** `Preflush Dirty Page` 点 |
| lsn_t max_checkpoint_age_async | **异步** `checkpoint` 点 |
| lsn_t max_checkpoint_age | **同步** `checkpoint` 点 |

上述几个 **sync / async** 点的计算方式可以参阅函数 `log_calc_max_ages`，以如下实例配置为例：

```
innodb_log_files_in_group = 4

innodb_log_file_size = 4G

总文件大小: 17179869184
```

各个成员变量值及占总文件大小的比例：

```
log_sys->log_group_capacity = 15461874893 (90%)

log_sys->max_modified_age_async = 12175607164 (71%)

log_sys->max_modified_age_sync = 13045293390 (76%)

log_sys->max_checkpoint_age_async = 13480136503 (78%)

log_sys->max_checkpoint_age = 13914979615 (81%)
```

通常的：

在当前未刷脏的最老lsn和当前lsn的距离超过 `max_modified_age_async`（**71%**）时，且开启了选项 `innodb_adaptive_flushing` 时，`page cleaner` 线程会去尝试做更多的 **Dirty Page Flush** 工作，避免脏页堆积。在当前未刷脏的最老lsn和当前lsn的距离超过 `max_modified_age_sync`（**76%**） 时，**`用户线程需要去做同步刷脏，这是一个性能下降的临界点，会极大的影响整体吞吐量和响应时间。`** 当上次 checkpoint 的lsn和当前lsn超过 `max_checkpoint_age`（**81%**），**`用户线程需要同步地做一次 checkpoint，需要等待 checkpoint 写入完成。`** 当上次 checkpoint 的lsn和当前lsn的距离超过 `max_checkpoint_age_async`（**78%**）但小于 `max_checkpoint_age`（**81%**）时，用户线程做一次异步 checkpoint（后台异步线程执行 `CHECKPOINT` 信息写入操作），无需等待 checkpoint 完成。

`log_group_t` 结构体主要成员如下表所示：

| 变量名 | 描述 |
| :- | :- |
| ulint n_files | ib_logfile 的文件个数 |
| lsn_t file_size| ib_logfile 的文件大小 |
| ulint space_id | Redo Log 的 space id，固定大小，值为 `SRV_LOG_SPACE_FIRST_ID` |
| ulint state | `LOG_GROUP_OK` 或者 `LOG_GROUP_CORRUPTED` |
| lsn_t lsn | group 内写到的 lsn | 
| lsn_t lsn_offset | 上述 lsn 对应的文件偏移量 |
| byte** file_header_bufs | buffer 区域，用于设定日志文件头信息，并写入 ib_logfile。当切换到新的 ib_logfile 时，更新该文件的起始 lsn，写入头部。头部信息还包含：`LOG_GROUP_ID`，`LOG_FILE_START_LSN`（当前文件起始 lsn），`LOG_FILE_WAS_CREATED_BY_HOT_BACKUP`（函数`log_group_file_header_flush`）|
| lsn_t scanned_lsn | 用于辅助记录崩溃恢复时扫描到的 lsn 号 |
| byte* checkpoint_buf | checkpoint 缓冲区，用于向日志文件写入 checkpoint 信息（下文详细描述）|

与 Redo Log 内存缓冲区相关的成员变量包括：

| 变量名 | 描述 |
| :- | :- |
| ulint buf_free | Log Buffer 中当前空闲可写的位置 |
| byte* buf | Log Buffer 起始位置指针 |
| ulint buf_size | Log Buffer 的大小，受参数 `innodb_log_buffer_size` 控制，但可能会自动 extend |
| ulint max_buf_free | 值为：`log_sys->buf_size / LOG_BUF_FLUSH_RATIO - LOG_BUF_FLUSH_MARGIN`，其中：`LOG_BUF_FLUSH_RATIO = 2`，`LOG_BUF_FLUSH_MARGIN = (4 * 512 + 4* page_size)`，`page_size` 默认为 16k，**当 buf_free 超过该值时，可能触发用户线程去写 redo**；在事务拷贝 redo 到 buffer 后，也会判断该值，如果超过 buf_free，设置 `log_sys->check_flush_or_checkpoint` 为 **true** |
| ulint buf_next_to_write | Log Buffer 偏移量，下次写入 redo 文件的起始位置，即本次写入的结束位置 |
| volatile bool is_extending | Log Buffer 是否正在进行扩展（防止过大的 `redo log entry` 无法写入 Buffer）, 实际上，**当写入的 redo log 长度超过 buf_size/2 时，就会去调用函数 `log_buffer_extend`**，一旦扩展 Buffer，就不会在缩减回去了！|
| ulint write_end_offset | 本次写入的结束位置偏移量（从逻辑来看有点多余，直接用log_sys->buf_free 就行了）|

和 `checkpoint` 检查点相关的成员变量：

| 变量名 | 描述 |
| :- | :- |
| ib_uint64_t next_checkpoint_no | 每完成一次 checkpoint 递增该值 |
| lsn_t last_checkpoint_lsn | 最近一次 checkpoint 时的 lsn，每完成一次 checkpoint，将 `next_checkpoint_lsn` 的值赋给 `last_checkpoint_lsn` | 
| lsn_t next_checkpoint_lsn | 下次 checkpoint 的 lsn（本次发起的 checkpoint 的 lsn ） |
| mtr_buf_t* append_on_checkpoint | 5.7 新增，在做 DDL 时（例如增删列），会先将包含 `MLOG_FILE_RENAME2` 日志记录的 buf 挂到这个变量上。在 DDL 完成后，再清理掉。`log_append_on_checkpoint` 主要是防止 DDL 期间 crash 产生的数据字典不一致。 该变量在如下 commit 加上： a5ecc38f44abb66aa2024c70e37d1f4aa4c8ace9 |
| ulint n_pending_checkpoint_writes | 大于 0 时，表示有一个 checkpoint 写入操作正在进行。用户发起 checkpoint 时，递增该值。后台线程完成 checkpoint 写入后，递减该值（`log_io_complete`）|
| rw_lock_t checkpoint_lock | **checkpoint 锁，每次写 checkpoint 信息时需要加 X 锁**。由异步 io 线程释放该 X 锁 |
| byte* checkpoint_buf | checkpoint 信息缓冲区，每次 checkpoint 前，先写该 buf，再将 buf 刷到磁盘 |

其他状态变量

| 变量名 | 描述 |
| :- | :- |
| bool check_flush_or_checkpoint | 当该变量被设置时，用户线程可能需要去做一些检查释放（要刷 log buffer、或是做 preflush、checkpoint 等）的工作，以防止 Redo 空间不足 |
| lsn_t write_lsn | 最近一次完成写入到文件的 LSN |
| lsn_t current_flush_lsn | 当前正在 fsync 的 LSN |
| lsn_t flushed_to_disk_lsn | 最近一次完成fsync到文件的 LSN |
| ulint n_pending_flushes | 表示 pending 的 redo fsync，这个值最大为1 |
| os_event_t flush_event | 若当前有正在进行的 fsync，并且本次请求也是 fsync 操作，则需要等待上次 fsync 操作完成 |

`log_sys` 与日志文件及日志缓冲区的关系可用下图来表示：


