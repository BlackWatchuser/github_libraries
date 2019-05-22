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

### 3.1 MySQL - B+ Tree



### 3.2 TiKV - LSM Tree

## 四、MVCC



### 4.1 MySQL - MVCC - Undo

### 4.2 TiDB - MVCC - 

### 五、事务模型


