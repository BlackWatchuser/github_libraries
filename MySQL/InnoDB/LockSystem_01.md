# MySQL · 引擎特性 · InnoDB 事务锁系统

[原文](http://mysql.taobao.org/monthly/2016/01/01/)

## 前言

本文的目的是对 InnoDB 的事务锁模块做个简单的介绍，使读者对这块有初步的认识。本文先介绍**行级锁**和**表级锁**的相关概念，再介绍其内部的一些实现；最后以两个有趣的案例结束本文。

本文所有的代码和示例都是基于当前最新的 `MySQL 5.7.10` 版本。

## 行级锁

InnoDB 支持到行级别粒度的并发控制，本小节我们分析下几种常见的行级锁类型，以及在哪些情况下会使用到这些类型的锁。

### LOCK_REC_NOT_GAP

`锁带上这个FLAG时，表示这个锁对象只是单纯的锁在记录上，不会锁记录之前的GAP。` 在 RC 隔离级别下一般加的都是该类型的记录锁（**但唯一二级索引上的 duplicate key 检查除外，总是加 LOCK_ORDINARY 类型的锁**）。

### LOCK_GAP

`表示只锁住一段范围，不锁记录本身，通常表示两个索引记录之间，或者索引上的第一条记录之前，或者最后一条记录之后的锁。`可以理解为一种区间锁，一般在 RR 隔离级别下会使用到GAP锁。

你可以通过切换到 RC 隔离级别，或者开启选项`innodb_locks_unsafe_for_binlog`来避免GAP锁。这时候**只有在检查外键约束或者duplicate key检查时才会使用到GAP LOCK**。

### LOCK_ORDINARY(Next-Key Lock)

`也就是所谓的NEXT-KEY锁，包含记录本身及记录之前的GAP。`**当前 MySQL 默认情况下使用 RR 的隔离级别，而 NEXT-KEY LOCK 正是为了解决 RR 隔离级别下的幻读问题。**所谓幻读就是一个事务内执行相同的查询，会看到不同的行记录。在 RR 隔离级别下这是不允许的。

假设索引上有记录 1, 4, 5, 8, 12 我们执行类似语句：SELECT ... WHERE col > 10 FOR UPDATE。如果我们不在 (8, 12) 之间加上 Gap锁，另外一个 Session 就可能向其中插入一条记录，例如9，当再执行一次相同的 SELECT FOR UPDATE 时，就会看到新插入的记录。

这也是为什么**插入一条记录时，需要判断下一条记录上是否加锁**了。

### LOCK_S（共享锁）

`共享锁的作用通常用于在事务中读取一条行记录后，不希望它被别的事务锁修改，但所有的读请求产生的LOCK_S锁是不冲突的。`在InnoDB里有如下几种情况会请求S锁：

1. `普通查询在隔离级别为SERIALIZABLE会给记录加LOCK_S锁。`但这也取决于场景：非事务读（auto-commit）在 SERIALIZABLE 隔离级别下，无需加锁（不过在当前最新的 5.7.10 版本中，SHOW ENGINE INNODB STATUS 的输出中不会打印只读事务的信息，只能从 informationschema.innodb_trx 表中获取到该只读事务持有的锁个数等信息）。

2. `类似SQL SELECT ... IN SHARE MODE，会给记录加S锁，其他线程可以并发查询，但不能修改。`基于不同的隔离级别，行为有所不同:

    * RC隔离级别： LOCK_REC_NOT_GAP | LOCK_S；

    * RR隔离级别：**如果查询条件为唯一索引且是唯一等值查询时，加的是 LOCK_REC_NOT_GAP | LOCK_S**；**对于非唯一条件查询，或者查询会扫描到多条记录时，加的是 LOCK_ORDINARY | LOCK_S 锁，也就是记录本身+记录之前的 GAP**；

3. `通常INSERT操作是不加锁的，但如果在插入或更新记录时，检查到duplicate key（或者有一个被标记删除的duplicate key），对于普通的INSERT/UPDATE，会加LOCK_S锁，而对于类似REPLACE INTO或者INSERT ... ON DUPLICATE这样的SQL加的是X锁。`而针对不同的索引类型也有所不同：

    * 对于**聚集索引**（参阅函数：row_ins_duplicate_error_in_clust），隔离级别小于等于RC时，加的是LOCK_REC_NOT_GAP类似的S或者X记录锁。否则加LOCK_ORDINARY类型的记录锁（NEXT-KEY LOCK）；

    * 对于**二级唯一索引**，`若检查到重复键，当前版本总是加 LOCK_ORDINARY 类型的记录锁(函数：row_ins_scan_sec_index_for_duplicate)。`实际上按照 RC 的设计理念，不应该加 GAP锁（bug#68021），官方也事实上尝试修复过一次，即对于 RC 隔离级别加上 LOCK_REC_NOT_GAP，但却引入了另外一个问题，导致二级索引的唯一约束失效（bug#73170），感兴趣的可以参阅我写的这篇博客，由于这个严重bug，官方很快又把这个fix给revert掉了。

4. 外键检查

    `当删除一条父表上的记录时，需要去检查是否有引用约束（row_pd_check_references_constraints），这时候会扫描子表（dict_table_t::referenced_list）上对应的记录，并加上共享锁。`按照实际情况又有所不同。我们举例说明：

    使用RC隔离级别，两张测试表：

    ``` sql
    create table t1 (a int, b int, primary key(a));
    create table t2 (a int, b int, primary key (a), key(b), foreign key(b) references t1(a));
    insert into t1 values (1,2), (2,3), (3,4), (4,5), (5,6), (7,8), (10,11);
    insert into t2 values (1,2), (2,2), (4,4);
    ```
    
    | t1.a | t1.b | \| | t2.a | t2.b |
    |:-:|:-:|:-:|:-:|:-:|
    | 1 | 2 | \| | 1 | 2 |
    | 2 | 3 | \| | 2 | 2 |
    | 3 | 4 | \| | 4 | 4 |
    | 4 | 5 | \| |
    | 5 | 6 | \| |
    | 7 | 8 | \| |
    | 10 | 11 | \| |

    执行SQL：delete from t1 where a = 10;

    * 在t1表记录10上加 LOCK_REC_NOT_GAP|LOCK_X锁；
    * 在t2表的supremum记录（表示最大记录）上加 LOCK_ORDINARY|LOCK_S锁，即锁住(4, ~)区间；

    执行SQL：delete from t1 where a = 2;

    * 在t1表记录(2,3)上加 LOCK_REC_NOT_GAP|LOCK_X锁；
    * 在t2表记录(1,2)上加 LOCK_REC_NOT_GAP|LOCK_S锁，这里检查到有引用约束，因此无需继续扫描(2,2)就可以退出检查，判定报错；

    执行SQL：delete from t1 where a = 3;

    * 在t1表记录(3,4)上加 LOCK_REC_NOT_GAP|LOCK_X 锁；
    * 在t2表记录(4,4)上加 LOCK_GAP|LOCK_S锁；

    另外从代码里还可以看到，如果扫描到的记录被标记删除时，也会加LOCK_ORDINARY|LOCK_S 锁。具体参阅函数：row_ins_check_foreign_constraint；

5. `INSERT ... SELECT 插入数据时，会对SELECT的表上扫描到的数据加LOCK_S锁`。

### LOCK_X（排他锁）

`排他锁的目的主要是避免对同一条记录的并发修改。`通常对于 UPDATE 或 DELETE 操作，或者类似 SELECT ... FOR UPDATE 操作，都会对记录加排他锁。

我们以如下表为例：

``` sql
create table t1 (a int, b int, c int, primary key(a), key(b));
insert into t1 values (1,2,3), (2,3,4),(3,4,5), (4,5,6),(5,6,7);
```

执行 SQL（**通过二级索引查询**）：update t1 set c = c + 1 where b = 3;

* RC 隔离级别：1.锁住二级索引记录，为 NOT GAP X 锁；2.锁住对应的聚集索引记录，也是 NOT GAP X锁；

* RR 隔离级别：1.锁住二级索引记录，为 LOCK_ORDINARY|LOCK_X 锁；2.锁住聚集索引记录，为 NOT GAP X锁；

执行 SQL（**通过聚集索引检索，更新二级索引数据**）：update t1 set b = b + 1 where a = 2;

* 对聚集索引记录加 LOCK_REC_NOT_GAP|LOCK_X锁;

* `在标记删除二级索引时，检查二级索引记录上的锁（lock_sec_rec_modify_check_and_lock），如果存在和 LOCK_X|LOCK_REC_NOT_GAP 冲突的锁对象，则创建锁对象并返回等待错误码；否则无需创建锁对象；`

* 当到达这里时，我们已经持有了聚集索引上的排他锁，因此能保证别的线程不会来修改这条记录。（**修改记录总是先聚集索引，再二级索引的顺序**），即使不对二级索引加锁也没有关系。但如果已经有别的线程已经持有了二级索引上的记录锁，则需要等待。

* 在标记删除后，需要插入更新后的二级索引记录时，依然要`遵循插入意向锁的加锁原则`。

我们**考虑上述两种 SQL 的混合场景，一个是先锁住二级索引记录，再锁聚集索引；另一个是先锁聚集索引，再检查二级索引冲突，因此在这类并发更新场景下，可能会发生死锁**。

不同场景，不同隔离级别下的加锁行为都有所不同，例如在 RC 隔离级别下，不符合 WHERE 条件扫描到的记录，会被立刻释放掉，但 RR 级别则会持续到事务结束。可以通过 GDB，断点函数`lock_rec_lock`来查看某条 SQL 如何执行加锁操作。

### LOCK_INSERT_INTENTION(插入意向锁)

**`INSERT INTENTION 锁是GAP锁的一种`**，如果有多个 session 插入同一个 GAP 时，他们无需互相等待，例如当前索引上有记录4和8，两个并发 session 同时插入记录6, 7。他们会分别为(4,8)加上GAP锁，但相互之间并不冲突（因为插入的记录不冲突）。

**当向某个数据页中插入一条记录时，总是会调用函数`lock_rec_insert_check_and_lock`进行锁检查（构建索引时的数据插入除外），会去检查当前插入位置的下一条记录上是否存在锁对象**，这里的下一条记录不是指的物理连续，而是按照逻辑顺序的下一条记录。

如果下一条记录上不存在锁对象：若记录是二级索引上的，先更新二级索引页上的最大事务ID为当前事务的ID；直接返回成功。

如果下一条记录上存在锁对象，就需要判断该锁对象是否锁住了 GAP。如果 GAP 被锁住了，并判定和插入意向GAP锁冲突，当前操作就需要等待，加的锁类型为**LOCK_X | LOCK_GAP | LOCK_INSERT_INTENTION**，并进入等待状态。但是 **`插入意向锁之间并不互斥`** 。这意味着在同一个 GAP 里可能有多个申请插入意向锁的会话。

### 锁表更新

我们知道 `GAP锁是在一个记录上描述的，表示记录及其之前的记录之间的GAP。`但**如果记录之前发生了插入或者删除操作，之前描述的GAP就会发生变化，InnoDB需要对锁表进行更新**。

对于**数据插入**，假设我们当前在记录[3,9]之间有会话持有锁(不管是否和插入意向锁冲突)，现在插入一条新的记录5，需要调用函数`lock_update_insert`。这里会遍历所有在记录9上的记录锁，如果这些锁不是插入意向锁并且是 LOCK_GAP 或者 NEXT-KEY LOCK（没有设置LOCK_REC_NOT_GAP标记，lock_rec_inherit_to_gap_if_gap_lock），就会为这些会话的事务增加一个新的锁对象，锁的类型为 LOCK_REC | LOCK_GAP，锁住的GAP范围在本例中为(3,5)。**所有符合条件的会话都继承了这个新的GAP，避免之前的GAP锁失效**。

对于**数据删除**，调用函数`lock_update_delete`，这里会遍历在被删除记录上的记录锁，**当符合如下条件时，需要为这些锁对应的事务增加一个新的GAP锁，锁的Heap No为被删除记录的下一条记录**：

``` c
lock_rec_inherit_to_gap
        for (lock = lock_rec_get_first(lock_sys->rec_hash, block, heap_no);
             lock != NULL;
             lock = lock_rec_get_next(heap_no, lock)) {

                if (!lock_rec_get_insert_intention(lock)
                    && !((srv_locks_unsafe_for_binlog
                          || lock->trx->isolation_level
                          <= TRX_ISO_READ_COMMITTED)
                         && lock_get_mode(lock) ==
                         (lock->trx->duplicates ? LOCK_S : LOCK_X))) {
                        lock_rec_add_to_queue(
                                LOCK_REC | LOCK_GAP | lock_get_mode(lock),
                                heir_block, heir_heap_no, lock->index,
                                lock->trx, FALSE);
                }
        }
```

从上述判断可以看出，即使在 RC 隔离级别下，也有可能继承 LOCK GAP锁，这也是**当前版本InnoDB唯一的意外：判断 Duplicate Key 时目前容忍 GAP 锁。**上面这段代码实际上在最近的版本中才做过更新，更早之前的版本可能存在二级索引损坏，感兴趣的可以阅读我的这篇博客。

完成GAP锁继承后，会将所有等待该记录的锁对象全部唤醒（`lock_rec_reset_and_release_wait`）。

### LOCK_PREDICATE

从 MySQL 5.7 开始，MySQL 整合了`boost.geometry`库以更好的支持空间数据类型，并支持在 Spatial 数据类型的列上构建索引，在 InnoDB 内，这个索引和普通的索引有所不同，基于 R-TREE 的结构，目前支持对2D数据的描述，暂不支持3D.

R-TREE 和 BTREE 不同，它能够描述多维空间，而多维数据并没有明确的数据顺序，因此无法在 RR 隔离级别下构建 NEXT-KEY 锁以避免幻读，因此 InnoDB 使用称为 Predicate Lock 的锁模式来加锁，会锁住一块查询用到的被称为 MBR（minimum bounding rectangle/box）的数据区域。因此这个锁不是锁到某个具体的记录之上的，可以理解为一种 Page 级别的锁。

Predicate Lock 和普通的记录锁或者表锁（如上所述）存储在不同的 lock hash 中，其相互之间不会产生冲突。

Predicate Lock 相关代码见`lock/lock0prdt.cc`文件

关于 Predicate Lock 的设计参阅官方WL#6609。

由于这块的代码量比较庞大，目前小编对 InnoDB 的 Spatial 实现了解有限，本文暂不对此展开，将在后面单独专门介绍 Spatial Index 时，再细细阐述这块内容。

### 隐式锁

`InnoDB 通常对插入操作无需加锁，而是通过一种“隐式锁”的方式来解决冲突。`**聚集索引记录中存储了事务id，如果另外有个session查询到了这条记录，会去判断该记录对应的事务id是否属于一个活跃的事务，并协助这个事务创建一个记录锁，然后将自己置于等待队列中。**该设计的思路是基于大多数情况下新插入的记录不会立刻被别的线程并发修改，而创建锁的开销是比较昂贵的，涉及到全局资源的竞争。

关于隐式锁转换，上一期的月报《[InnoDB 事务子系统](http://mysql.taobao.org/monthly/2015/12/01/)》中我们已经介绍过了，这里不再赘述。

### 锁的冲突判定

锁模式的兼容性矩阵通过如下数组进行快速判定：

``` c
static const byte lock_compatibility_matrix[5][5] = {
/** IS IX S X AI /
/ IS / { TRUE,  TRUE,  TRUE,  FALSE, TRUE},
/ IX / { TRUE,  TRUE,  FALSE, FALSE, TRUE},
/ S /  { TRUE,  FALSE, TRUE,  FALSE, FALSE},
/ X /  { FALSE, FALSE, FALSE, FALSE, FALSE},
/ AI / { TRUE,  TRUE,  FALSE, FALSE, FALSE}
};
```

**对于记录锁而言，锁模式只有 LOCK_S 和 LOCK_X，其他的 FLAG 用于锁的描述，如前述 LOCK_GAP、LOCK_REC_NOT_GAP 以及 LOCK_ORDINARY、LOCK_INSERT_INTENTION 四种描述。** 在比较两个锁是否冲突时，即使不满足兼容性矩阵，在如下几种情况下，依然认为是相容的，无需等待（参考函数lock_rec_has_to_wait）

* 对于 GAP 类型（锁对象建立在supremum上或者申请的锁类型为LOCK_GAP）且申请的不是插入意向锁时，无需等待任何锁，这是因为不同Session对于相同GAP可能申请不同类型的锁，而GAP锁本身设计为不互相冲突；

* LOCK_ORDINARY 或者 LOCK_REC_NOT_GAP 类型的锁对象，无需等待 LOCK_GAP 类型的锁；

* LOCK_GAP 类型的锁无需等待 LOCK_REC_NOT_GAP 类型的锁对象；

* 任何锁请求都无需等待插入意向锁。

## 表级锁

`InnoDB的表级别锁包含五种锁模式：LOCK_IS、LOCK_IX、LOCK_X、LOCK_S以及LOCK_AUTO_INC锁`，锁之间的相容性遵循数组`lock_compatibility_matrix`中的定义。

`InnoDB表级锁的目的是为了防止DDL和DML的并发问题。`但从 5.5 版本开始引入 MDL 锁后，InnoDB层的表级锁的意义就没那么大了，MDL 锁本身已经覆盖了其大部分功能。以下我们介绍下几种 InnoDB 表锁类型。

### LOCK_IS / LOCK_IX

也就是所谓的意向锁，这实际上可以理解为一种“暗示”未来需要什么样行级锁，IS表示未来可能需要在这个表的某些记录上加共享锁，IX表示未来可能需要在这个表的某些记录上加排他锁。**意向锁是表级别的，IS和IX锁之间相互并不冲突，但与表级S/X锁冲突**。

在对记录加S锁或者X锁时，必须保证其在相同的表上有对应的意向锁或者锁强度更高的表级锁。

### LOCK_X

当加了 LOCK_X 表级锁时，所有其他的表级锁请求都需要等待。通常有这么几种情况需要加X锁：

* `DDL操作的最后一个阶段（ha_innobase::commit_inlace_alter_table）对表上加LOCK_X锁，以确保没有别的事务持有表级锁。`通常情况下，Server 层 MDL 锁已经能保证这一点了，在 DDL 的 commit 阶段是加了排他的 MDL 锁的。**但诸如外键检查或者崩溃恢复的事务（正在进行某些操作），这些操作都是直接 InnoDB 自治的，不走 Server 层，也就无法通过 MDL 所保护**；

* 当设置会话的`autocommit`变量为**OFF**时，执行`LOCK TABLE tbname WRITE`这样的操作会加表级的 LOCK_X 锁（ha_innobase::external_lock）；

* `对某个表空间执行discard或者import操作时，需要加LOCK_X锁`（ha_innobase::discard_or_import_tablespace）。

### LOCK_S

* `在DDL的第一个阶段，如果当前DDL不能通过ONLINE的方式执行，则对表加LOCK_S锁`（prepare_inplace_alter_table_dict）；

* 当设置会话的`autocommit`变量为**OFF**时，执行`LOCK TABLE tbname READ`这样的操作会加表级的 LOCK_S 锁（ha_innobase::external_lock）。

从上面的描述我们可以看到 LOCK_X 及 LOCK_S 锁在实际的大部分负载中都很少会遇到。主要还是互相不冲突的 LOCK_IS 及 LOCK_IX 锁。一个有趣的问题是，`每次加表锁时，总是要扫描表上所有的表级锁对象，检查是否有冲突的锁。`很显然，如果我们在同一张表上的更新并发度很高，这个链表就会非常长。

基于大多数表锁不冲突的事实，我们在 RDS MYSQL 中对各种表锁对象进行计数；在检查是否有冲突时，例如当前申请的是意向锁，如果此时 LOCK_S 和 LOCK_X 的锁计数都是0，就可以认为没有冲突，直接忽略检查。由于检查是在持有全局大锁`lock_sys->mutex`下进行的。在单表大并发下，这个优化的效果还是非常明显的，可以减少持有全局大锁的时间。

### LOCK_AUTO_INC

`AUTO_INC锁加在表级别，和AUTO_INC、表级S锁以及X锁不相容。锁的范围为SQL级别，SQL结束后即释放。`AUTO_INC 的加锁逻辑和 InnoDB 的锁模式相关，这里在简单介绍一下。

通常对于自增列，我们既可以显式指定该值，也可以直接用NULL，系统将自动递增并填充该列。我们还可以在批量插入时混合使用者两种方式。不同的分配方式，其具体行为受到参数`innodb_autoinc_lock_mode`的影响。但在基于STATEMENT模式复制时，可能会影响到复制的数据一致性，[官方文档](https://dev.mysql.com/doc/refman/5.7/en/innodb-auto-increment-handling.html)有详细描述，不再赘述，只说明下锁的影响。

自增锁模式通过参数`innodb_autoinc_lock_mode`来控制，加锁选择参阅函数`ha_innobase::innobase_lock_autoinc`。

具体的，有以下几个值：

**`AUTOINC_OLD_STYLE_LOCKING（0）`**

也就是所谓的**传统加锁模式（在5.1版本引入这个参数之前的策略）**，`在该策略下，会在分配前加上AUTO_INC锁，并在SQL结束时释放掉。该模式保证了在STATEMENT复制模式下，备库执行类似INSERT ... SELECT这样的语句时的一致性`，因为**这样的语句在执行时无法确定到底有多少条记录，只有在执行过程中不允许别的会话分配自增值，才能确保主备一致**。

很显然这种锁模式非常影响并发插入的性能，但却保证了一条 SQL 内自增值分配的连续性。

**`AUTOINC_NEW_STYLE_LOCKING（1）`**

这是 InnoDB 的默认值。在该锁模式下：

* 普通的 INSERT 或 REPLACE 操作会先加一个`dict_table_t::autoinc_mutex`，然后去判断表上是否有别的线程加了 LOCK_AUTO_INC 锁，如果有的话，释放 autoinc_mutex，并使用 `OLD STYLE` 的锁模式。否则，在预留本次插入需要的自增值之后，就快速的将 autoinc_mutex 释放掉。很显然，对于普通的并发 INSERT 操作，都是无需加 LOCK_AUTO_INC 锁的。因此大大提升了吞吐量；

* 但是**对于一些批量插入操作，例如：LOAD DATA，INSERT ... SELECT 等还是使用 OLD STYLE 的锁模式，SQL执行期间加LOCK_AUTO_INC 锁**。

`和传统模式相比，这种锁模式也能保证STATEMENT模式下的复制安全性，但却无法保证一条插入语句内的自增值的连续性`，并且在执行一条混合了显式指定自增值和使用系统分配两种方式的插入语句时，可能存在一定的自增值浪费。

例如执行SQL：

``` sql
INSERT INTO t1 (c1,c2) VALUES (1,'a'), (NULL,'b'), (5,'c'), (NULL,’d’）
```

假设当前 AUTO_INCREMENT 值为 101，在传统模式下执行完后，下一个自增值为103，而在新模式下，下一个可用的自增值为105，因为在开始执行 SQL 时，会先预取了 [101, 104] 4个自增值，这和插入行的个数相匹配，然后将 AUTO_INCREMENT 设为105，导致自增值103和104被浪费掉。

**`AUTOINC_NO_LOCKING（2）`**

`这种模式下只在分配时加个mutex即可，很快就释放，不会像NEW STYLE那样在某些场景下会退化到传统模式。` **因此设为2时不能保证批量插入的复制安全性**。

### 关于自增锁的小BUG

这是 MariaDB 的 Jira 上报的一个小bug，在 row 模式下，由于不走 Parse 的逻辑，我们并不能知道行记录是通过批量导入还是普通 INSERT 产生的，因此 COMMAND 类型为 SQLCOM_END，而在判断是否加自增锁时，是通过 COMMAND 类型是否为 SQLCOM_INSERT 或者 SQLCOM_REPLACE 来决定是否忽略加 AUTO_INC 锁。这个额外的锁开销，会导致在使用 ROW 模式时，InnoDB 总是加 AUTO_INC 锁，加 AUTO_INC 锁又涉及到全局事务资源的开销，从而导致性能下降。

修复的方式也比较简单，将 SQLCOM_END 这个 COMMAND 类型也纳入考虑。

具体参阅[Jira链接](https://mariadb.atlassian.net/browse/MDEV-7578)。

## 事务锁管理

**InnoDB 所有的事务锁对象都是挂在全局对象 `lock_sys` 上，同时每个事务对象上也维持了其拥有的事务锁**，每个表对象（`dict_table_t`）上也维持了构建在其上的表级锁对象。

如下图所示：

![](https://raw.githubusercontent.com/CHXU0088/github_libraries/master/Pic/innodb_lock_20160101.png)

### 加表级锁

* 首先从当前事务的 `trx_lock_t::table_locks` 中查找是否已经加了等同或更高级别的表锁，如果已经加锁了，则直接返回成功（`lock_table_has`）；

* 检查当前是否有和正在申请的锁模式冲突的表级锁对象（`lock_table_other_has_incompatible`）；
    * 直接遍历链表 `dict_table_t::locks` 链表

* 如果存在冲突的锁对象，则需要进入等待队列（lock_table_enqueue_waiting）
    * 创建等待锁对象 （`lock_table_create`）
    * 检查是否存在死锁（`DeadlockChecker::check_and_resolve`），当存在死锁时：如果当前会话被选作牺牲者，就移除锁请求（`lock_table_remove_low`），重置当前事务的 wait_lock 为空，并返回错误码 DB_DEADLOCK；若被选成胜利者，则锁等待解除，可以认为当前会话已经获得了锁，返回成功
    * 若没有发生死锁，设置事务对象的相关变量后，返回错误码 DB_LOCK_WAIT，随后进入锁等待状态

* 如果不存在冲突的锁，则直接创建锁对象（`lock_table_create`），加入队列。

**`lock_table_create`** : 创建锁对象

* 当前请求的是 AUTO-INC 锁时；

    * 递增 `dict_table_t::n_waiting_or_granted_auto_inc_locks`。前面我们已经提到过，当这个值非0时，对于自增列的插入操作就会退化到**OLD-STYLE**；
    * 锁对象直接引用已经预先创建好的 `dict_table_t::autoinc_lock`，并加入到 `trx_t::autoinc_locks` 集合中;

* 对于非 AUTO-INC 锁，则从一个pool中分配锁对象：

    * 在事务对象 `trx_t::lock` 中，维持了两个pool，一个是 ` trx_lock_t::rec_pool`，预分配了一组**用于行级锁分配的锁对象**，另外一个是 `trx_lock_t::table_pool`，**用于表级锁的锁对象分配**。`通过预分配内存的方式，可以避免在持有全局大锁时(lock_sys->mutex)进行昂贵的内存分配操作。`rec_pool 和 table_pool 预分配的大小都为8个锁对象（`lock_trx_alloc_locks`）；
    * 如果 table_pool 已经用满，则走内存分配，创建一个锁对象；

* 构建好的锁对象分别加入到事务的 `trx_t::lock.trx_locks` 链表上以及表对象的 `dict_table_t::locks` 链表上；

* 构建好的锁对象加入到当前事务的 `trx_t::lock.table_locks` 集合中。

**可以看到锁对象会加入到不同的集合或者链表中，通过挂载到事务对象上，可以快速检查当前事务是否已经持有表锁；通过挂到表对象的锁链表上，可以用于检查该表上的全局冲突情况。**

### 加行级锁

**行级锁加锁的入口函数为 `lock_rec_lock`，其中第一个参数 impl 如果为 TRUE，则在当前记录上已有的锁和 LOCK_X|LOCK_REC_NOT_GAP 锁不冲突时，就无需创建锁对象。**（见上文关于记录锁 LOCK_X 相关描述部分），为了描述清晰，下文的流程描述，默认 impl 为 FALSE。

**`lock_rec_lock`**：

* 首先尝试 **`fast lock`** 的方式，对于冲突少的场景，这是比较普通的加锁方式（`lock_rec_lock_fast`）, 符合如下情况时，可以走 fast lock：

    * `记录所在的page上没有任何记录锁时，直接创建锁对象，加入rec_hash，并返回成功`；
    * 记录所在的 page 上只存在一个记录锁，并且属于当前事务，且这个记录锁预分配的 bitmap 能够描述当前的 heap no（预分配的bit数为创建锁对象时的 page 上记录数 + 64，参阅函数 `RecLock::lock_size`），则直接设置对应的bit位并返回；

* 无法走 fast lock 时，再调用 **`slow lock`** 的逻辑（`lock_rec_lock_slow`）：

    * 判断当前事务是否已经持有了一个优先级更高的锁，如果是的话，直接返回成功（`lock_rec_has_expl`）；

    * 检查是否存在和当前申请锁模式冲突的锁（`lock_rec_other_has_conflicting`），如果存在的话，就创建一个锁对象（`RecLock::RecLock`），并加入到等待队列中（`RecLock::add_to_waitq`），这里会进行死锁检测；

    * 如果没有冲突的锁，则进入队列（`lock_rec_add_to_queue`）：已经有在同一个 Page 上的锁对象且没有别的会话等待相同的heap no时，可以直接设置对应的bitmap（`lock_rec_find_similar_on_page`）；否则需要创建一个新的锁对象；

* 返回错误码，对于 **DB_LOCK_WAIT, DB_DEADLOCK** 等错误码，会在上层进行处理。

### 等待及死锁判断

当发现有冲突的锁时，调用函数 `RecLock::add_to_waitq` 进行判断：

* 如果持有冲突锁的线程是内部的后台线程（例如后台**dict_state**线程），这个线程不会被一个高优先级的事务取消掉，因为`总是优先保证内部线程正常执行`；

* **比较当前会话和持有锁的会话的事务优先级**，调用函数 `trx_arbitrate` 返回被选作牺牲者的事务；

    * 当前发起请求的会话是后台线程，但持有锁的会话设置了高优先级时，选择当前线程作为牺牲者；
    * 持有锁的线程为后台线程时，在第一步已经判断了，不会选作牺牲者；
    * **如果两个会话都设置了优先级，低优先级的被选做牺牲者，优先级相同时，请求者被选做牺牲者**（`thd_tx_arbitrate`）；

    * PS: 目前最新版本的 5.7 还不支持用户端设置线程优先级（但增加一个配置 session 变量的接口非常容易)；

* 如果当前会话的优先级较低，或者另外一个持有锁的会话为后台线程，这时候若当前会话设置了优先级，直接报错，并返回错误码 

    * 默认不设置优先级时，请求锁的会话也会被选作 `victim_trx`，但只创建锁等待对象，不会直接返回错误；

* 当持有锁的会话被选作牺牲者时，说明当前会话肯定设置了高优先级，这时候会走 `RecLock::enqueue_priority` 的逻辑；

    * 如果持有锁的会话在等待另外一个不同的锁时，或者持有锁的事务不是 readonly 的，当前会话会被回滚掉；

    * 开始跳队列，直到当前会话满足加锁条件（`RecLock::jump_queue`）；

        * 请求的锁对象跳过阻塞它的锁对象，直接操作 hash 链表，将锁对象往前挪；

        * 从当前 lock，向前遍历链表，逐个判断是否有别的会话持有了相同记录上的锁（`RecLock::is_on_row`），并将这些会话标记为回滚（`mark_trx_for_rollback`）,同时将这些事务对象搜集下来，以待后续处理（但直接阻塞当前会话的事务会被立刻回滚掉）；

    * 高优先级的会话非常具有杀伤力，其他低优先级会话即使拿到了锁，也会被它所干掉。

不过实际场景中，我们并没有多少机会去设置事务的优先级，这里先抛开这个话题，只考虑默认的场景，即所有的事务优先级都未设置。

在创建了一个处于 WAIT 状态的锁对象后，我们需要进行死锁检测（`RecLock::deadlock_check`），**死锁检测采用深度优先遍历的方式，通过事务对象上的 `trx_t::lock.wait_lock` 构造事务的 wait-for graph 进行判断，当最终发现一个锁请求等待闭环时，可以判定发生了死锁。另外一种情况是，如果检测深度过长（即锁等待的会话形成的检测链路非常长），也会认为发生死锁，最大深度默认为 `LOCK_MAX_DEPTH_IN_DEADLOCK_CHECK`，值为200**。

当发生死锁时，需要选择一个牺牲者（`DeadlockChecker::select_victim()`）来解决死锁，通常事务权重低的回滚（`trx_weight_ge`）。

* 修改了非事务表的会话具有更高的权重；

* 如果两个表都修改了、或者都没有修改事务表，那么就**根据的事务的 undo 数量加上持有的事务锁个数来决定权值（`TRX_WEIGHT`）**；

* 低权重的事务被回滚，高权重的获得锁对象。

Tips：对于一个经过精心设计的应用，我们可以从业务上避免死锁，而**死锁检测本身是通过持有全局大锁来进行的，代价非常高昂**，在阿里内部的应用中，由于有专业的团队来保证业务 SQL 的质量，我们可以选择性的禁止掉死锁检测来提升性能，尤其是在热点更新场景，带来的性能提升非常明显，极端高并发下，甚至能带来数倍的提升。

当无法立刻获得锁时，会将错误码传到上层进行处理（`row_mysql_handle_errors`）：

* `DB_LOCK_WAIT`：

    * 具有高优先级的事务已经搜集了会阻塞它的事务链表，这时会统一将这些事务回滚掉（`trx_kill_blocking`）；

    * 将当前的线程挂起（`lock_wait_suspend_thread`），等待超时时间取决于 session 级别配置（`innodb_lock_wait_timeout`），默认为50秒；

    * 如果当前会话的状态设置为 running，一种是被选做死锁检测的牺牲者，需要回滚当前事务，另外一种是在进入等待前已经获得了事务锁，也无需等待；

    * 获得等待队列的一个空闲slot（`lock_wait_table_reserve_slot`）：

        * 系统启动时，已经创建好了足够用的 slot 数组，类型为 `srv_slot_t`，挂在 `lock_sys->waiting_threads` 上；

        * 分配 slot 时，从 slot 数组的第一个元素开始遍历，直到找到一个空闲的 slot。注意这里存在的一个性能问题是，如果挂起的线程非常多，每个新加入挂起等待的线程都需要遍历直到找到一个空闲的 slot。实际上如果每次遍历都从上次分配的位置往后找，到达数组末尾在循环到数组头，这样可以在高并发高锁冲突场景下获得一定的性能提升；

    * 如果会话在 InnoDB 层（通常为true），则强制从 InnoDB 层退出，确保其不占用 `innodb_thread_concurrency` 的槽位。然后进入等待状态。被唤醒后，会再次强制进入 InnoDB 层；

    * 被唤醒后，释放 slot（`lock_wait_table_release_slot`）；

    * 若被选作死锁的牺牲者了，返回上层回滚事务；若等待超时了，则**根据参数 `innodb_rollback_on_timeout` 的配置，默认为 OFF 只回滚当前 SQL，设置为 ON 表示回滚整个事务**。

* `DB_DEADLOCK`: 直接回滚当前事务。

### 释放锁及唤醒

大多数情况下事务锁都是在事务提交时释放，但有两种意外：

* **AUTO-INC 锁在 SQL 结束时直接释放（`innobase_commit --> lock_unlock_table_autoinc`）**；

* **在 RC 隔离级别下执行 DML 语句时，从引擎层返回到 Server 层的记录，如果不满足 where 条件，则需要立刻 unlock 掉（`ha_innobase::unlock_row`）**。

除这两种情况外，其他的事务锁都是在事务提交时释放的（`lock_trx_release_locks --> lock_release`）。事务持有的所有锁都维护在链表 `trx_t::lock.trx_locks` 上，依次遍历释放即可。

对于行锁，从全局 hash 中删除后，还需要判断别的正在等待的会话是否可以被唤醒（`lock_rec_dequeue_from_page`）。例如如果当前释放的是某个记录的X锁，那么所有的S锁请求的会话都可以被唤醒。

这里的移除锁和检查的逻辑开销比较大，尤其是大量线程在等待少量几个锁时。`当某个锁从 hash 链上移除时，InnoDB实际上通过遍历相同page上的所有等待的锁，并判断这些锁等待是否可以被唤醒。而判断唤醒的逻辑又一次遍历，这是因为当前的链表维护是基于<space, page no>的，并不是基于Heap no构建的。`关于这个问题的讨论，可以参阅bug#53825。官方开发Sunny也提到虽然使用 <space, page no, heap no> 来构建链表，移除 Bitmap 会浪费更多的内存，但效率更高，而且现在的内存也没有以前那么昂贵。

对于表锁，如果表级锁的类型不为 LOCK_IS，且当前事务修改了数据，就将表对象的 `dict_table_t::query_cache_inv_id` 设置为当前最大的事务id。在检查是否可以使用该表的 Query Cache 时会使用该值进行判断（`row_search_check_if_query_cache_permitted`），如果某个用户会话的事务对象的 `low_limit_id（即最大可见事务id）`比这个值还小，说明它不应该使用当前 table cache 的内容，也不应该存储到 query cache 中。

表级锁对象的释放调用函数 `lock_table_dequeue`。

注意**在释放锁时，如果该事务持有的锁对象太多，每释放1000（`LOCK_RELEASE_INTERVAL`）个锁对象，会暂时释放下 `lock_sys->mutex` 再重新持有，防止InnoDB hang住**。

## 两个有趣的案例

本小节我们来分析几个比较有趣的死锁案例。

### 普通的并发插入导致的死锁

``` sql
create table t1 (a int primary key); 
```

开启三个会话执行： 

``` sql
insert into t1(a) values (2);
```

| session 1 | session 2 | session 3 |
| :-- | :-- | :-- |
| BEGIN; INSERT ... | | |
| | INSERT（Block，为 session 1 创建 X 锁，并等待 S 锁）| |
| | | INSERT (Block，同上等待 S 锁）|
| ROLLBACK，释放锁 | | |
| | 获得 S 锁 | 获得 S 锁 |
| | 申请插入意向 X 锁，等待 session 3 | |
| | | 申请插入意向 X 锁，等待 session 2 |

上述描述了互相等待的场景，因为 **`插入意向X锁和S锁是不相容的`** 。这也是一种**典型的锁升级导致的死锁**。如果 session 1 执行 COMMIT 的话，则另外两个线程都会因为 `Duplicate Key` 失败。

这里需要解释下为何要申请插入意向锁，因为 **ROLLBACK 回滚时原记录是被标记删除的** 。而我们**尝试插入的记录和这个标记删除的记录是相邻的（键值相同），根据插入意向锁的规则，插入位置的下一条记录上如果存在与插入意向X锁冲突的锁时，则需要获取插入意向X锁**。

另外一种类似（但产生死锁的原因不同）的场景是在一张同时存在聚集索引和唯一索引的表上，通过replace into的方式插入冲突的唯一键，可能会产生死锁，在3月份的月报，我已经专门描述过这个问题，感兴趣的可以延伸阅读下。

#### 又一个并发插入的死锁现象

两个会话参与。在 RR 隔离级别下：

例表如下：

``` sql
create table t1 (a int primary key ,b int);
insert into t1 values (1,2),(2,3),(3,4),(11,22);
```

| session 1 | session 2 |
| :-- | :-- |
| begin; select * from t1 where a = 5 for update;（获取[11, 22] 上的 GAP X 锁） | |
| | begin; select * from t1 where a = 5 for update;（同上，GAP 锁之间不冲突）|
| insert into t1 values (4, 5);（等待 session 1）| |
| | insert into t1 values (4, 5);（需要等待 session 2，死锁）|

引起这个死锁的原因是`非插入意向的GAP X锁和插入意向X锁之间是冲突的`。

------