# MySQL · 引擎特性 · InnoDB Redo Log

## 前言

InnoDB 有两块非常重要的日志，一个是 **Undo Log**，另外一个是 **Redo Log**，`前者用来保证事务的原子性以及 InnoDB 的 MVCC，后者用来保证事务的持久性`。

和大多数关系型数据库一样，InnoDB 记录了对数据文件的物理更改，并保证总是日志先行，也就是所谓的 `WAL`，`即在持久化数据文件前，保证之前的 redo 日志已经写到磁盘`。

`LSN（log sequence number）` 用于记录日志序号，它是一个不断递增的 `unsigned long long` 类型整数。在 InnoDB 的日志系统中，LSN 无处不在，它既用于表示修改脏页时的日志序号，也用于记录 `checkpoint`，通过 LSN，可以具体的定位到其在 redo log 文件中的位置。

**为了管理脏页，在 Buffer Pool 的每个 instance上 都维持了一个 `flush list`，flush list 上的 page 按照修改这些 page 的LSN号进行排序。** `因此定期做 redo checkpoint 时，选择的 LSN 总是所有 bp instance 的 flush list 上最老的那个page（拥有最小的LSN）。`由于采用 WAL 的策略，每次事务提交时需要持久化 redo log 才能保证事务不丢。而延迟刷脏页则起到了合并多次修改的效果，避免频繁写数据文件造成的性能问题。

由于 InnoDB 日志组的特性已经被废弃（redo 日志写多份），归档日志（InnoDB Archive Log）特性也在 5.7 被彻底移除，本文在描述相关逻辑时会忽略这些逻辑。另外限于篇幅，InnoDB 崩溃恢复的逻辑将在下期讲述，本文重点阐述 redo log 产生的生命周期以及 MySQL 5.7 的一些改进点。

本文的分析基于最新的 MySQL 5.7.7-RC 版本。

## InnoDB 日志文件

InnoDB 的 redo log 可以通过参数 `innodb_log_files_in_group` 配置成多个文件，另外一个参数 `innodb_log_file_size` 表示每个文件的大小。因此总的 redo log 大小为 **innodb_log_files_in_group * innodb_log_file_size**。

Redo Log 文件以 ib_logfile[number] 命名，日志目录可以通过参数 `innodb_log_group_home_dir` 控制。Redo log 以顺序的方式写入文件，写满时则回溯到第一个文件，进行覆盖写。（但`在做 redo checkpoint 时，也会更新第一个日志文件的头部 checkpoint 标记，所以严格来讲也不算顺序写`）。

```
   ----------------<<<---------------
   ⬇                               ⬆  
ib_logfile0 => ib_logfile1 => ib_logfile2
```

在InnoDB内部，逻辑上ib_logfile被当成了一个文件，对应同一个space id。由于是使用512字节block对齐写入文件，可以很方便的根据全局维护的LSN号计算出要写入到哪一个文件以及对应的偏移量。

Redo log文件是循环写入的，在覆盖写之前，总是要保证对应的脏页已经刷到了磁盘。在非常大的负载下，Redo log可能产生的速度非常快，导致频繁的刷脏操作，进而导致性能下降，通常在未做checkpoint的日志超过文件总大小的76%之后，InnoDB 认为这可能是个不安全的点，会强制的preflush脏页，导致大量用户线程stall住。如果可预期会有这样的场景，我们建议调大redo log文件的大小。可以做一次干净的shutdown，然后修改Redo log配置，重启实例。

除了redo log文件外，InnoDB还有其他的日志文件，例如为了保证truncate操作而产生的中间日志文件，包括 truncate innodb 表以及truncate undo log tablespace，都会产生一个中间文件，来标识这些操作是成功还是失败，如果truncate没有完成，则需要在 crash recovery 时进行重做。有意思的是，根据官方worklog的描述，最初实现truncate操作的原子化时是通过增加新的redo log类型来实现的，但后来不知道为什么又改成了采用日志文件的方式，也许是考虑到低版本兼容的问题吧。

关键结构体
log_sys对象
log_sys是InnoDB日志系统的中枢及核心对象，控制着日志的拷贝、写入、checkpoint等核心功能。它同时也是大写入负载场景下的热点模块，是连接InnoDB日志文件及log buffer的枢纽，对应结构体为log_t。

其中与 redo log 文件相关的成员变量包括：

