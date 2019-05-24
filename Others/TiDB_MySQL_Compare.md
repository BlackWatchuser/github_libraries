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


三、LSM Tree的历史由来

LSM Tree的最早概念，诞生于1996年google的“BigTable”论文。后世多种数据库产品对LSM Tree的具体实现，都有一些小的差异。采用LSM Tree作为存储结构的数据库有，Google的LevelDB, Facebook的RockDB(RockDB来源于LevelDB), Cassandra,HBase等。

四、提高写吞吐量的思路
既然顺序写比起随机写速度更快。那得想办法将数据顺序写。

4.1 一种方式是数据来后，直接顺序落盘
这拥有很高的写速度。但是当我们想要查寻一个数据的时候，由于存储下的数据本身是无序的（写的值本身无法控制顺序），无法使用任何算法进行优化，只能挨个查询，读取速度是很慢的。

4.2 另一种方式，是保证落盘的数据是顺序写入的同时，还保证这些数据是有序的
而请求写入的数据本身是无序且不可预测的，如何保证落盘的数据是有序的呢？这就需要利用内存访问速度比硬盘快的原理。将写入的请求，先在内存中缓存起来，按一定的有序结构组织，达到一定量后，再写入硬盘，从而使得硬盘顺序写入了有序的数据。提高数据的写入速度同时，方便了后续基于有序数据的查找(有序的数据结构，可以通过二分查找等算法进行进行快速查询，具体查找算法，得看是哪种有序结构)

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

2.


### 4.2 TiDB - MVCC - 

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



**`未完待续`**

------