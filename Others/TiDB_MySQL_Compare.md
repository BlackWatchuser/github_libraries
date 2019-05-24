# TiDB 与 MySQL 在功能和实现上的一些比较（一）

> TiDB 是 **`PingCAP`** 公司设计的 **`开源分布式 HTAP (Hybrid Transactional and Analytical Processing) 数据库`**，结合了传统的 RDBMS 和 NoSQL 的最佳特性。TiDB **`兼容 MySQL，支持无限的水平扩展，具备强一致性和高可用性`**。TiDB 的目标是为 `OLTP (Online Transactional Processing)` 和 `OLAP (Online Analytical Processing)` 场景提供一站式的解决方案。

## 一、TiDB 的特性

TiDB 具备如下特性：

* [**`高度兼容 MySQL`**](https://www.pingcap.com/docs-cn/sql/mysql-compatibility/)

    大多数情况下，无需修改代码即可从 MySQL 轻松迁移至 TiDB，分库分表后的 MySQL 集群亦可通过 TiDB 工具进行实时迁移（Syncer / DM）。

* **`水平弹性扩展`**

    通过简单地增加新节点即可实现 TiDB 的水平扩展，按需扩展吞吐或存储，轻松应对高并发、海量数据场景。

    **`架构：存储计算分离 - 计算层“无状态” - 分布式存储/Raft`**

* **`分布式事务`**

    TiDB 100% 支持标准的 ACID 事务（**`Percolator 事务模型 - 乐观锁事务模型 - 分布式两阶段提交[优化]`**）。

    **`数据存储层自动根据 Range 进行分片（Region），根据负载情况可以将 Region 分散到各个节点（逻辑上仍是一张表，避免了传统的分库分表和依赖中间件等问题）`**

* **`真正金融级高可用`**

    相比于传统主从 (M-S) 复制方案，基于 Raft 的多数派选举协议可以提供金融级的 100% 数据强一致性保证，且在不丢失大多数副本的前提下，可以实现故障的自动恢复 (auto-failover)，无需人工介入。

    **`强一致 - Raft Leader 提供服务 - Multi Raft Group（Region，管理一段范围[Range]数据）- 各个 Group 的 Leader 可以根据负载被调度到不同的机器上（负载均衡） - 性能 - 热Region仍存在性能瓶颈（raft store + raft apply）`**

    **`存储层：数据安全 - Raft 多副本 | 自动故障转移 - Raft 的选举机制 - 心跳机制/统计信息 - PD 组件 | 计算层：无状态（上层添加负载均衡组件）`**

    **`PD（etcd / raft） 在选举的过程中无法对外提供服务，这个时间大约是3秒钟。`**

* **`一站式 HTAP 解决方案`**

    TiDB 作为典型的 OLTP 行存数据库，同时兼具强大的 OLAP 性能，配合 TiSpark，可提供一站式 HTAP 解决方案，一份存储同时处理 OLTP & OLAP，无需传统繁琐的 ETL 过程。

* **`云原生 SQL 数据库`**

    TiDB 是为云而设计的数据库，支持公有云、私有云和混合云，使部署、配置和维护变得十分简单。

**`TiDB 的设计目标是 100% 的 OLTP 场景和 80% 的 OLAP 场景，更复杂的 OLAP 分析可以通过 TiSpark 项目来完成（相比 SQL 统计，TiSpark 可以直接访问存储层 TiKV，避免 SQL 解析等过程，更高效）`**。

目前使用体验：

1. OLTP 场景，事务的表现形式和 MySQL 仍有一定区别，具体看上方关于差异的链接；

2. OLAP 场景，因为底层是分布式的（并且数据被打散到了各个节点上，条件满足的情况下可以并行处理），所以对大数据量的提取和过滤（条件下推）性能较好。

3. 单个计算节点如果进行大数据量的计算，内存使用上可能存在问题，会触发 OOM（可以通过参数限制，到达阈值时直接返回错误，不触发OOM）。

4. 存储层对硬件要求较高（安装时有 IOPS 的检测，不满足时部署会报错）；

5. 乐观事务锁模型：业务逻辑导致冲突较多时，性能会急剧下降；

6. 因为是分布式集群（节点间交互存在网络开销），故单个请求的响应时间是不如 MySQL 这种单机数据库（本地调用）的；

## 二、TiDB 的架构

![](https://raw.githubusercontent.com/CHXU0088/github_libraries/master/Pic/TiDB/TiDBArchitecture_20190522.png)

### TiDB Server

**`TiDB Server 负责接收 SQL 请求，处理 SQL 相关的逻辑，并通过 PD 找到存储计算所需数据的 TiKV 地址，与 TiKV 交互获取数据，最终返回结果。TiDB Server 是无状态的，其本身并不存储数据，只负责计算，可以无限水平扩展`**，可以通过负载均衡组件（如LVS、HAProxy 或 F5）对外提供统一的接入地址。

### PD Server

**`Placement Driver (简称 PD) 是整个集群的管理模块，其主要工作有三个：一是存储集群的元信息（某个 Key 存储在哪个 TiKV 节点）；二是对 TiKV 集群进行调度和负载均衡（如数据的迁移、Raft group leader 的迁移等）；三是分配全局唯一且递增的事务 ID`**。

PD 通过 Raft 协议保证数据的安全性。Raft 的 leader server 负责处理所有操作，其余的 PD server 仅用于保证高可用。建议部署奇数个 PD 节点。

### TiKV Server

TiKV Server 负责存储数据，从外部看 **`TiKV 是一个分布式的提供事务的 Key-Value 存储引擎。存储数据的基本单位是 Region，每个 Region 负责存储一个 Key Range（从 StartKey 到 EndKey 的左闭右开区间）的数据，每个 TiKV 节点会负责多个 Region。TiKV 使用 Raft 协议做复制，保持数据的一致性和容灾。副本以 Region 为单位进行管理，不同节点上的多个 Region 构成一个 Raft Group，互为副本。数据在多个 TiKV 之间的负载均衡由 PD 调度，这里也是以 Region 为单位进行调度`**。

### TiSpark

TiSpark 作为 TiDB 中解决用户复杂 OLAP 需求的主要组件，将 **`Spark SQL 直接运行在 TiDB 存储层上`**，同时融合 TiKV 分布式集群的优势，并融入大数据社区生态。至此，TiDB 可以通过一套系统，同时支持 OLTP 与 OLAP，免除用户数据同步的烦恼。

## 三、存储层的区别

> **`写入效率相对会慢些，但优化了读取效率 | 对[随机]写入性能有很大优化，但读取效率会下降（其实还有写放大的问题）`**

### 3.1 MySQL - B+ Tree

![](https://raw.githubusercontent.com/CHXU0088/github_libraries/master/Pic/MySQL/B_Tree_Structure_20130110.png)

``` shell
1. ROOT、NON-LEAF、LEAF(Level 0) Pages；

2. 同层（Level）的页之间通过两个指针形成一个双向链表；

3. 中间节点保存的 KV：Key → the minimum key | Value → child_page_number；
```

![](https://raw.githubusercontent.com/CHXU0088/github_libraries/master/Pic/MySQL/INDEX_Page_Overview_20130107.png)

``` shell
1. 记录与记录之间物理上是没有绝对顺序，但通过指针形成了一个单向链表；

2. 每行记录都包含一个，Maximum Transaction ID、undo pointer 和  <= MVCC；

3. Page Directory：slot → 4 - 8 记录，指向记录的offset；
```

### 3.2 TiKV - Raft + RocksDB - LSM Tree

#### RocksDB

[\[RocksDB\]](https://rocksdb.org.cn/doc/Home.html) | [\[GitHub\]](https://github.com/facebook/rocksdb/wiki)

RocksDB 是一种 **`可以存储任意二进制 KV 数据的嵌入式存储`**。RocksDB 按顺序组织所有数据，他们的通用操作是 Get(key), Put(key), Delete(Key) 以及 NewIterator()。

**`RocksDB 有三种基本的数据结构：mentable，sstfile 以及 logfile`**。

* **`mentable`** 是一种内存数据结构 —— 所有写入请求都会进入 mentable，然后选择性进入logfile。

* **`logfile`** 是一种有序写存储结构。当 mentable 被填满的时候，其会被刷到 sstfile 文件并存储起来，然后相关的 logfile 会在之后被安全地删除。

* **`sstfile`** 内的数据都是排序好的，以便于根据 key 快速搜索。

#### LSM Tree

LSM Tree 的最早概念，诞生于 1996 年 google 的 “BigTable” 论文。后世多种数据库产品对 LSM Tree 的具体实现，都有一些小的差异。采用 LSM Tree 作为存储结构的数据库有，Google 的 LevelDB, Facebook 的 RocksDB（RocksDB 来源于 LevelDB），Cassandra 和 HBase 等。

**提高写吞吐量的思路**

既然顺序写比起随机写速度更快。那得想办法将数据顺序写。

原理：是把一颗大树拆分成N棵小树， 它首先写入到内存中（内存没有寻道速度的问题，随机写的性能得到大幅提升），在内存中构建一颗有序小树，随着小树越来越大，内存的小树会flush到磁盘上。当读时，由于不知道数据在哪棵小树上，因此必须遍历所有的小树，但在每颗小树内部数据是有序的。

![](https://raw.githubusercontent.com/CHXU0088/github_libraries/master/Pic/TiDB/LSM_TREE_TIMELINE_20190522.png)

``` shell
1. WAL（Write-Ahead Log）

2. MemTable

3. Immutable Memtable

    之所以要使用 Immutable Memtable，就是为了避免将 MemTable 中的内容序列化到磁盘中时会阻塞写操作。

4. SSTable

5. Leveled Compaction

6. Bloom Filter
```

#### TiKV

![](https://raw.githubusercontent.com/CHXU0088/github_libraries/master/Pic/TiDB/tidb_storage_01_20170515.png)

![](https://raw.githubusercontent.com/CHXU0088/github_libraries/master/Pic/TiDB/tidb_storage_02_20170515.png)

![](https://raw.githubusercontent.com/CHXU0088/github_libraries/master/Pic/TiDB/tidb_storage_03_20170515.png)

## 四、MVCC

MVCC → 多版本并发控制，其相对应的是基于锁的并发控制；

优势：“快照”读不加锁，挺高了系统整体的并发度；

RR | RC 隔离级别下表现形式不同（生成 Read View 的机制不同）；

### 4.1 MySQL - MVCC - Undo

1. 无论是聚簇索引，还是二级索引，其每条记录都包含了一个 **`DELETED BIT`** 位，用于标识该记录是否是删除记录。除此之外，聚簇索引记录还有两个系统列：**`DATA_TRX_ID`**，**`DATA_ROLL_PTR`**。DATA_TRX_ID 表示产生当前记录项的事务ID；DATA _ROLL_PTR 指向当前记录项的 Undo 信息

![](https://raw.githubusercontent.com/CHXU0088/github_libraries/master/Pic/MySQL/mvcc_pk_record_20120420.jpg)

``` c
dict_table_add_system_columns(
/*==========================*/
dict_table_t*   table,  /*!< in/out: table */
mem_heap_t* heap)   /*!< in: temporary heap */
{
ut_ad(table);
ut_ad(table->n_def == (table->n_cols - table->get_n_sys_cols()));
ut_ad(table->magic_n == DICT_TABLE_MAGIC_N);
ut_ad(!table->cached);
/* NOTE: the system columns MUST be added in the following order
(so that they can be indexed by the numerical value of DATA_ROW_ID,
etc.) and as the last columns of the table memory object.
The clustered index will not always physically contain all system
columns.
Intrinsic table don't need DB_ROLL_PTR as UNDO logging is turned off
for these tables. */
dict_mem_table_add_col(table, heap, "DB_ROW_ID", DATA_SYS,
     DATA_ROW_ID | DATA_NOT_NULL,
     DATA_ROW_ID_LEN);
#if (DATA_ITT_N_SYS_COLS != 2)
#error "DATA_ITT_N_SYS_COLS != 2"
#endif
#if DATA_ROW_ID != 0
#error "DATA_ROW_ID != 0"
#endif
dict_mem_table_add_col(table, heap, "DB_TRX_ID", DATA_SYS,
     DATA_TRX_ID | DATA_NOT_NULL,
     DATA_TRX_ID_LEN);
#if DATA_TRX_ID != 1
#error "DATA_TRX_ID != 1"
#endif
if (!table->is_intrinsic()) {
dict_mem_table_add_col(table, heap, "DB_ROLL_PTR", DATA_SYS,
     DATA_ROLL_PTR | DATA_NOT_NULL,
     DATA_ROLL_PTR_LEN);
#if DATA_ROLL_PTR != 2
#error "DATA_ROLL_PTR != 2"
#endif
/* This check reminds that if a new system column is added to
the program, it should be dealt with here */
#if DATA_N_SYS_COLS != 3
#error "DATA_N_SYS_COLS != 3"
#endif
}
}
```

2. ReadView

``` c
dulint    low_limit_id;
/* 事务号 >= low_limit_id 的记录，对于当前 Read View 都是不可见的 */

dulint    up_limit_id;
/* 事务号 < up_limit_id 的记录，对于当前 Read View 都是可见的 */

ulint    n_trx_ids;
/* Number of cells in the trx_ids array */

dulint*    trx_ids;

/* Additional trx ids which the read should not see: 
   typically, these are the active transactions 
   at the time when the read is serialized,
   except the reading transaction itself; 
   the trx ids in this array are in a descending order */

dulint    creator_trx_id;
/* trx id of creating transaction, or (0, 0) used in purge */
```

3. 判断逻辑

``` c
bool changes_visible(
trx_id_t    id,
const table_name_t& name) const
MY_ATTRIBUTE((warn_unused_result))
{
    
ut_ad(id > 0);

// 如果 TRX_ID 小于 ReadView 中最小的, 则这条记录是可以看到。说明这条记录是在 select 这个事务开始之前就结束的
if (id < m_up_limit_id || id == m_creator_trx_id) {
    return(true);
}
check_trx_id_sanity(id, name);

// 如果比 ReadView 中最大的还要大，则说明这条记录是在事务开始之后进行修改的，所以此条记录不应查看到
if (id >= m_low_limit_id) {
    return(false);
} else if (m_ids.empty()) {
    return(true);
}
const ids_t::value_type*    p = m_ids.data();
return(!std::binary_search(p, p + m_ids.size(), id)); 
// 判断是否在 ReadView 中， 如果在说明在创建 ReadView 时,此条记录还处于活跃状态则不应该查询到，否则说明创建 ReadView 是此条记录已经是不活跃状态则可以查询到
}
```

### 4.2 TiDB - MVCC - GC

TiDB 采用 MVCC 的方式来进行并发控制。当对数据进行更新或者删除时，原有的数据不会被立刻删除，而是会被保留一段时间，并且在这段时间内这些旧数据仍然可以被读取。这使得写入操作和读取操作不必互斥，并使读取历史数据成为可能。

存在超过一定时间并且不再使用的版本会被清理，否则它们将始终占用硬盘空间，并对性能产生负面影响。TiDB 使用一个垃圾回收 (GC) 机制来清理这些旧数据。

#### MVCC 在 TiKV 中的实现：

**`TiKV 的 MVCC 实现是通过在 Key 后面添加 Version 来实现`**，简单来说，没有 MVCC 之前，可以把 TiKV 看做这样的：

``` shell
Key1 -> Value
Key2 -> Value
……
KeyN -> Value
```

有了 MVCC 之后，TiKV 的 Key 排列是这样的：

``` shell
Key1-Version3 -> Value
Key1-Version2 -> Value
Key1-Version1 -> Value
……
Key2-Version4 -> Value
Key2-Version3 -> Value
Key2-Version2 -> Value
Key2-Version1 -> Value
……
KeyN-Version2 -> Value
KeyN-Version1 -> Value
……
```

注意，**`对于同一个 Key 的多个版本，我们把版本号较大的放在前面，版本号小的放在后面`**（回忆一下 Key-Value 一节我们介绍过的 Key 是有序的排列），这样 **`当用户通过一个 Key + Version 来获取 Value 的时候，可以将 Key 和 Version 构造出 MVCC 的 Key，也就是 Key-Version。然后可以直接 Seek(Key-Version)，定位到第一个大于等于这个 Key-Version 的位置`**。

* 垃圾收集器（Gabage Collector）

    通过 **`垃圾收集器（Gabage Collector）`** 来移除无效的版本，避免数据库中存有越来越多的 MVCC 版本。但是我们又不能仅仅移除某个 safe point 之前的所有版本。因为对于某个 key 来说，有可能只存在一个版本，那么这个版本就必须被保存下来。在TiKV中，如果在 safe point 前存在 Put 或者 Delete 记录，那么比这条记录更旧的写入记录都是可以被移除的，不然的话只有Delete，Rollback和Lock 会被删除。

* GC

    TiDB 会周期性地进行 GC。每个 TiDB Server 启动后都会在后台运行一个 gc_worker，每个集群中会有其中一个 gc_worker 被选为 leader，leader 负责维护 GC 的状态并向所有的 TiKV Region leader 发送 GC 命令。

    * MVCC versions：每个 key 的版本个数

    * MVCC delete versions：GC 删除掉的每个 key 的版本个数

    **`tikv_gc_life_time`** 用于配置历史版本保留时间，可以手动修改；
    
    **`tikv_gc_safe_point`** 记录了当前的 safePoint，用户可以安全地使用大于 safePoint 的时间戳创建 snapshot 读取历史版本。safePoint 在每次 GC 开始运行时自动更新。

### 五、事务模型（两阶段提交 + Group Commit | Percolator）

![](https://raw.githubusercontent.com/CHXU0088/github_libraries/master/Pic/TiDB/tidb_trx_model_20160901.jpg)

**`未完待续`**

------