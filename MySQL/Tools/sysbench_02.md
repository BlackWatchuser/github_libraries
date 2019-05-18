# SYSBENCH 在美团点评中的应用

[原文](https://tech.meituan.com/2017/07/14/sysbench-meituan.html)

如何快速入门数据库？以我个人经验来看，数据库功能和性能测试是一条不错的捷径。当然从公司层面，数据库测试还有更多实用的功能。这方面，美团点评使用的是知名工具 [sysbench](https://github.com/akopytov/sysbench)，主要是用来解决以下几个问题：

* 统一测试方法，以便测试结果的可重复和可对比。

* 结合美团点评的业务特点和硬件特性，得到最优的参数配置。

* 扩展 sysbench 的测试能力，比如增加对 JSON 测试的支持。

数据库测试虽然入门简单，但是却能在测试中获得对数据库、操作系统等的感性认识，为日后深入的研究数据库和性能调优打下很好的基础。如果你不满足于仅仅使用测试工具，还想开发自己的测试工具，那么在本文的最后，还会从源码层面解读 sysbench 的高性能秘密。

## 测试可重复性

如果只是把测试工具运行起来，获得一个输出结果，那么测试就变成一个没有任何技术含量，也没有实际意义的事情。一个经得起推敲的测试，首先 **`要保证测试结果的可重复性。测试结果的可重复性可能与系统的硬件、操作系统的版本、I/O调度算法、CPU调度算法、数据库版本、数据库的配置、测试工具、测试时间等息息相关`**。美团点评集团 DBA 有两位数以上，如果没有一个统一的测试平台，那么每个 DBA 的测试结果将难以比较，整个团队也无法有效协作。为了解决这个问题，我们做了几点的强制统一：测试工具及其参数的统一、MySQL 配置的统一。

### 为何选用 sysbench

当前可采用的测试工具有 sysbench、TPCC-MySQL 以及公司或者个人开发的压测工具。从功能上来讲，无论采用哪种方式都可以满足要求。美团点评采用 sysbench 有如下几点考虑：

1. sysbench 作为业界流行的测试工具，绝大多数 DBA 都熟悉它。

2. MySQL 几大主流厂商 Oracle、Percona 等在发布性能数据时，都采用 sysbench 作为测试工具。使用相同的测试工具，更便于复现测试结果。

3. sysbench 目前已支持 MySQL 8.0 的测试，因此从长远来看，该工具将会持续活跃。美团点评可以充分利用开源社区资源，降低测试工具的维护成本。

4. sysbench 内嵌了 Lua 脚本。在不需要修改核心的 C 语言代码的情况下，通过增添或者修改 Lua 脚本，即可扩展新的测试场景，大大提高了 DBA 人员对 sysbench 的掌控能力。

### 测试参数统一

sysbench 提供了丰富的测试选项，包括测试表数量、单表数据量、测试预热时间等。我们根据美团点评的业务特征和使用 sysbench 的经验，为了避免新同学走不必要的弯路和降低测试的时间，将部分测试参数统一。另外在测试中，对 MySQL 核心参数做了强制统一。比如 **`sync_binlog=1，innodb_flush_log_at_trx_commit=2，innodb_io_capacity=2000`** 等。这里不一一赘述。

这里简要介绍 sysbench 测试过程中涉及到的几个重要参数：

| 参数名 | 参数值 | 解释 |
| :- | :- | :- |
| tables | 16 | 测试中使用的表分别为 sbtest1 … sbtest16 |
| table_size | 25,000,000 | 每个表的数据量 |
| threads | 16 | 测试线程数 | 
| time | 3600 | 测试时间，单位为秒 |
| warmup_time | 600 | 预热时间，预防冷数据对测试结果影响 |
| rate | 0 | 如果该值不为 0，则整个测试变成生产者和消费者模式。如果该值较大，超出 MySQL 的处理能力，则会造成请求积压，响应时间延长，无法反应真实的 OLTP 业务特性 |
| histogram | on | 输出测试过程中系统响应时间的分布 |
| percentile | 99 | 输出 99 线的响应时间 |

### sysbench 助力参数优化

相对于业务更新速度，数据库的变化较为缓慢，然而影响 MySQL 数据库性能的因素却不断呈现出来。硬件方面，比如 SATA、SSD、PCIe 的出现，让数据库的 IO 能力相对于传统的机械硬盘有几百甚至上千倍的提升；CPU 多核技术的发展，让单台服务器拥有上百个核；单 GB 内存的价格越来越低，服务器配置的内存也越来越大。软件方面，MySQL 5.7 的出现，增强了多核处理能力，提高了从库的复制速度等。

如何将新硬件和新版本的性能发挥到极致，是每个 DBA 都会遇到的问题。美团点评 DBA 团队在前期理论调研后，会设计符合公司业务特征的场景进行严格的性能测试，确定最终的参数配置方案。下面以确定 MySQL 5.7 中多线程复制的 **`slave_parallel_workers`** 参数为例，来了解如何使用 sysbench 来优化参数配置。

为了测试从库的复制速度，我们使用 sysbench 的 **`oltp_write_only.lua`**（包括增、删、改）在主库制造负载（**TPS:33,336**），观察从库的 TPS，如下图所示。

![](https://raw.githubusercontent.com/CHXU0088/github_libraries/master/Pic/MySQL/slave_parallel_workers_20170714.png "slave_parallel_workers")

从上图看出，工作线程在 8 时，从库的 TPS 达到最大。到这里为止，对于数据库新人来说，我们可以很自信的宣称自己学会了通过测试进行数据库调优。但是我们不能只满足于此，应该做更深入的探究。比如，多线程复制的原理是怎样的？如何进一步提升从库的 TPS ？建议有兴趣的读者可以继续调研 **`binlog_group_commit_sync_delay`** 和 **`binlog_group_commit_sync_no_delay_count`** 参数。

## sysbench 可扩展性

### 测试场景可扩展

sysbench 不仅具有丰富的功能，还具有优良的设计与实现。为了把测试场景完全交给客户定制，所有的测试用例，均使用 Lua 编写；如果需要支持新的数据库，只要实现 sysbench 提供的 10 来个接口即可；而其他通用功能，均由 sysbench 提供。架构图如下：

![](https://raw.githubusercontent.com/CHXU0088/github_libraries/master/Pic/MySQL/sysbench_arch_20170714.png "架构图")

使用过 sysbench 或者其他同类型测试工具的都知道，**`数据库测试分为三个阶段，包括prepare 阶段、warmup 阶段、运行阶段`**。这三个过程的实现完全使用 Lua 来控制，因此很容易定制。**`sysbench 提供的默认测试用例有只读测试、只写测试、读写混合等`**。这些测试用例也是用 Lua 实现的，通过修改这些测试用例，测试人员可以很快的掌握编写自己测试用例的技巧。

比如，日前在 **`评估 MySQL 5.7 JSON 替代 MongoDB 的可行性`**。与业务人员交流过程中发现，业务中并没有使用 MongoDB 的一些复杂特性，比如内嵌 JS 代码、map/reduce 等特性，但是其 TPS 较高，较为关注 MySQL 5.7 + JSON 与 MongoDB 的性能比较。因此需要一款可以测试 MySQL JSON 性能的测试工具。在前一节的分析中，我们只需更改 sysbench 中几个 Lua 文件即可拥有这样的测试工具。

![](https://raw.githubusercontent.com/CHXU0088/github_libraries/master/Pic/MySQL/sysbench_pwr_20170714.png "测试过程")

### 性能可伸缩

**`sysbench 的高性能有两个方面，一方面是其采用多线程结构，同时模拟多个客户端去并发操作，这方面无需赘言；另一方面是其高效的性能收集`**，比如多个线程同时执行多个任务，那么性能信息的更新可能会存在热点等状况，本节来解密其高性能的数据收集技术。

## 性能数据收集

数据库的性能往往不能用简单的 TPS 或者 QPS 来反映，还需要知道压测过程中系统的运行是否平稳（响应时间和 QPS 等）。

![](https://raw.githubusercontent.com/CHXU0088/github_libraries/master/Pic/MySQL/tps_histogram_20170714.png "直方图")

如果仅给出系统的最大 TPS，比如 10000 左右，可能掩盖了系统中的重要信息。比如上图中，系统的 TPS 随着时间，周期性的严重抖动，值得数据库和开发人员关注。通过打开 sysbench 的周期性报表，即可获得这样的统计信息。

### 引入热点的性能信息收集

在上图中，可以看到 TPS 和 QPS 的数值。**`在多线程编程环境中，想要获得一段时间内执行的事务数量或者 SQL 数量，可以通过引入一个原子变量，当执行完一个事务和 SQL 后，就自增一次`**。

如此一来，全局的事务计数器与 SQL 计数器就会成为多个线程竞争的热点，影响 sysbench 的扩展性甚至严重干扰测试结果，尤其是在目前的多核处理架构，如下图所示：

![](https://raw.githubusercontent.com/CHXU0088/github_libraries/master/Pic/MySQL/multi_thread_hot_point_20170714.png)

据某著名数据库专家的话，**`凡是有热点的地方，解决之道只需一个字：拆。比如大家耳熟能详的分库分表，将大表拆成小表，大库拆成小库`**；像在大内存的系统中，MySQL 会自动创建多个 buffer pool instance，就是为了避免多个线程同时去竞争一个互斥量。sysbench 在解决这个问题时，也不能例外。它 **`为每个工作线程都分配一个局部的计数器，增加计数时，只需更新线程内部的计数器；当需要获得全局计数时，把局部计数器的值汇总即可`**。这种办法获得的计数值精确度比上一种办法要低，但是其可以线性扩展，而且在性能数据收集这个角度其精确度已经足够了。具体代码在 `sb_counter.c` 中，有兴趣的可以下载代码阅读。

![](https://raw.githubusercontent.com/CHXU0088/github_libraries/master/Pic/MySQL/sysbench_counter_shard_20170714.png)

### SQL 响应时间分布

在评估数据库的响应时间时，我们经常会提到 **`90线、95线和99线（分别代表90%，95%和99%的响应时间在某个值之下）`**。因为 **`最大值和最小值往往受偶然因素的影响很大，而平均值往往会淹没更多的细节`**。一个 SQL 响应时间的分布可以让 DBA 更好的了解数据库的性能，因此优秀的测试工具必须支持这个功能。

在实际的编程中，我们往往会遇到一个矛盾的问题。数据库的响应时间往往差距很大，比如快的可能在 0.01 ms 以下，而遇到数据库抖动或者复杂查询时，可能到秒级别，甚至几十秒都有可能。如果使用 **`算术刻度`**，比如单位为 0.01 ms，那么就需要长度为千万级别的整型数组去表示，耗费大量内存。而且在响应时间为秒级别时，如此精确的计数也没有必要。我们需要的是随着响应时间越小，精度越高，响应时间越长，精度可以适当放低，而 **`“对数刻度”`** 正好具有这种特性。

### 对数刻度

sysbench 正是使用该方法做时间统计。当 sysbench 得到一个响应时间时，通过 **`k=floor((log(response_time) +6.908) * 55.35 + 0.5)`** ，获得**刻度值k**。**`当响应时间为 response_time=0.001 时，k 为 0；response_time=0.01 时，k = 128；response_time=10 时，k = 509；response_time=100 时，k = 1023。随着响应时间的增加，k 的变化越缓慢。其中横轴为刻度 k，纵轴为响应时间，单位为 ms`**。

**`当测试完成时，需要将 k 转化为响应时间。算法为 respone_time = exp((k/55.535)-6.908)。这样就可以使用较少的空间，完成较大时间跨度的记录，而且精度是动态变化的，响应时间越小，精度越高`**。

![](https://raw.githubusercontent.com/CHXU0088/github_libraries/master/Pic/MySQL/sysbench_k_20170714.png "精确值与误差")

### 响应时间收集之热点

在官方给出的 MySQL 性能测试数据库中，我们可以看到在高端机型上 QPS 已经达到百万，即使在一般的企业级服务器，也能达到几十万的级别。在前面的介绍中知道，响应时间是记录在一个数组上的，如果响应时间比较稳定，假设有 50% 的响应时间是落在一个刻度上，那么该刻度对应的变量就会被每秒更新几十万次，形成一个更新热点。参考下图：

![](https://raw.githubusercontent.com/CHXU0088/github_libraries/master/Pic/MySQL/latency_hot_point_20170714.png)

在前面性能信息收集上也遇到类似的热点问题，当然我们也可以给每个线程各配备一个 response[1024] 的数组来避免热点。sysbench 采用了类似的方法，但是做了些改变。它也是采用多个 response[1024] 的数组，但是其数量被固定为 128 个。

### 响应时间收集之避免热点

![](https://raw.githubusercontent.com/CHXU0088/github_libraries/master/Pic/MySQL/sysbench_reponse_array_20170714.png)

## 结论

美团点评运用 sysbench 进行性能测试以调整 MySQL 配置参数，也扩展了 sysbench 的功能来做 JSON 测试。通过对其源码研究，我们了解了其良好的功能扩展性以及其性能扩展性。未来美团点评会在 sysbench 上做进一步的定制，比如将测试做成服务化，让开发和运维人员能够方便使用 sysbench 做数据库的容量测试，也可以让数据库爱好者更快上手数据库测试。

## 作者简介

广友，美团点评到店综合事业群资深 MySQL DBA，2012 年毕业于中国科学技术大学，2017 年加入美团点评，长期致力于 MySQL 及周边工具的研究。

金龙，2014 年加入新美大，主要从事相关的数据库运维、高可用和相关的运维平台建设。对运维高可用与架构相关感兴趣的同学可以关注个人微信公众号“自己的设计师”，定期推送运维相关原创内容。

> 【思考题】  
> 文中从最简单的性能测试开始，然后进一步探讨性能优化，到从脚本层面进行功能扩展，最后去分析其设计和实现的优秀之处。在整个过程中，我们需要广泛的掌握 Linux 下性能监测与调优工具，也需要深入分析一种数据库或者数据库测试工具的源码。大家在专精一门技术的同时，往往还有多种辅助技术。聊一聊你在工作过程中最得力的辅助技术，以及如何用它来解决技术问题。

------