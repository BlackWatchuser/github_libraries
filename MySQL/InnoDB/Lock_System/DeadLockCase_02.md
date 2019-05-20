# 死锁案例（四）

> **`本文介绍一例三个并发 insert 导致的死锁，根本原因还是在于 insert 唯一键时申请插入意向锁这个特殊的 GAP 锁。`**

## 一、案例分析

### 1.1 环境准备 

Percona server 5.6，**RR** 模式：

``` sql
CREATE TABLE `t6` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `a` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  unique KEY `idx_a` (`a`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8mb4;

insert into t6 values (1,2),(2,8),(3,9),(4,11),(5,19);
```

| session1 | session2 | session3 |
| :- | :- | :- |
| begin; |  |  |
| insert into t6 (id,a) values (6,15); | begin; |  |
|  | insert into t6 (id,a) values (7,15); | begin; |
|  |  | insert into t6 (id,a) values (8,15); |
| rollback; |  |  ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction |

### 1.2 死锁日志

``` sql
------------------------
LATEST DETECTED DEADLOCK
------------------------

2017-09-18 10:03:50 7f78eae30700

*** (1) TRANSACTION:

TRANSACTION 462308725, ACTIVE 18 sec inserting, thread declared inside InnoDB 1
mysql tables in use 1, locked 1
LOCK WAIT 4 lock struct(s), heap size 1184, 2 row lock(s), undo log entries 1
MySQL thread id 3825465, OS thread handle 0x7f78eaef4700, query id 781148519 localhost root update

insert into t6 (id,a) values (7,15)

*** (1) WAITING FOR THIS LOCK TO BE GRANTED:

RECORD LOCKS space id 227 page no 4 n bits 80 index `idx_a` of table `test`.`t6` trx id 462308725 lock_mode X insert intention waiting

*** (2) TRANSACTION:

TRANSACTION 462308726, ACTIVE 10 sec inserting, thread declared inside InnoDB 1
mysql tables in use 1, locked 1
4 lock struct(s), heap size 1184, 2 row lock(s), undo log entries 1
MySQL thread id 3825581, OS thread handle 0x7f78eae30700, query id 781148528 localhost root update

insert into t6 (id,a) values (8,15)

*** (2) HOLDS THE LOCK(S):

RECORD LOCKS space id 227 page no 4 n bits 80 index `idx_a` of table `test`.`t6` trx id 462308726 lock mode S

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:

RECORD LOCKS space id 227 page no 4 n bits 80 index `idx_a` of table `test`.`t6` trx id 462308726 lock_mode X insert intention waiting

*** WE ROLL BACK TRANSACTION (2)
```

### 1.3 死锁分析

首先依然要再次强调 insert 插入操作的加锁逻辑：

**`第一阶段：唯一性约束检查，先申请 LOCK_S + LOCK_ORDINARY`**

**`第二阶段：获取阶段一的锁并且成功完成 insert 操作`**

**`插入的位置有 Gap 锁：LOCK_INSERT_INTENTION，为了防止其他 insert 唯一键冲突`**

**`新数据插入：LOCK_X + LOCK_REC_NOT_GAP`**

对于 insert 操作来说，若发生唯一约束冲突，则需要对冲突的唯一索引加上 **`S Next-Key Lock`**。从这里会发现，即使是 **RC** 事务隔离级别，也同样会存在 **`Next-Key Lock`** 锁，从而阻塞并发。然而，文档没有说明的是，对于检测到冲突的唯一索引，等待线程在获得 S Lock 之后，还需要对下一个记录进行加锁，在源码中由函数 `row_ins_scan_sec_index_for_duplicate` 进行判断。

其次，我们需要了解锁的兼容性矩阵：

|  | GAP | Insert Intention | Record | Next-Key |
| :- | :- | :- | :- | :- |
| GAP | 兼容 | 兼容 | 兼容 | 兼容 |
| Insert Intention | 冲突 | 兼容 | 兼容 | 冲突 |
| Record | 兼容 | 兼容 | 冲突 | 冲突 |
| Next-Key | 兼容 | 兼容 | 冲突 | 冲突 |

从兼容性矩阵我们可以得到如下结论:

INSERT操作之间不会有冲突。

GAP,Next-Key会阻止Insert。

GAP和Record,Next-Key不会冲突

Record和Record、Next-Key之间相互冲突。

已有的Insert锁不阻止任何准备加的锁。

这个案例是三个会话并发执行的，我打算一步一步来分析每个步骤执行完之后的事务日志。

#### 第一步 session1 执行插入操作：

``` sql
insert into t6 (id,a) values (6,15);

---TRANSACTION 462308737, ACTIVE 5 sec

1 lock struct(s), heap size 360, 0 row lock(s), undo log entries 1

MySQL thread id 3825779, OS thread handle 0x7f78eacd9700, query id 781149440 localhost root init

show engine innodb status

TABLE LOCK table `test`.`t6` trx id 462308737 lock mode IX
```

因为是第一个插入的语句，所以唯一性冲突检查通过，成功插入 (6,15)。此时 session1 会话持有 (6,15) 的 **`LOCK_X|LOCK_REC_NOT_GAP`** 锁。参考 `INSERT sets an exclusive lock on the inserted row. This lock is an index-record lock, not a next-key lock (that is, there is no gap lock) and does not prevent other sessions from inserting into the gap before the inserted row.`

#### 第二步 session2 执行插入操作：

``` sql
insert into t6 (id,a) values (7,15);

---TRANSACTION 462308738, ACTIVE 4 sec inserting

mysql tables in use 1, locked 1

LOCK WAIT 2 lock struct(s), heap size 360, 1 row lock(s), undo log entries 1

MySQL thread id 3825768, OS thread handle 0x7f78ea9c9700, query id 781149521 localhost root update

insert into t6 (id,a) values (7,15)

------- TRX HAS BEEN WAITING 4 SEC FOR THIS LOCK TO BE GRANTED:

RECORD LOCKS space id 227 page no 4 n bits 80 index `idx_a` of table `test`.`t6` trx id 462308738 lock mode S waiting

------------------

TABLE LOCK table `test`.`t6` trx id 462308738 lock mode IX

RECORD LOCKS space id 227 page no 4 n bits 80 index `idx_a` of table `test`.`t6` trx id 462308738 lock mode S waiting

---TRANSACTION 462308737, ACTIVE 66 sec

2 lock struct(s), heap size 360, 1 row lock(s), undo log entries 1

MySQL thread id 3825779, OS thread handle 0x7f78eacd9700, query id 781149526 localhost root init

show engine innodb status

TABLE LOCK table `test`.`t6` trx id 462308737 lock mode IX

RECORD LOCKS space id 227 page no 4 n bits 80 index `idx_a` of table `test`.`t6` trx id 462308737 lock_mode X locks rec but not gap
```

首先，session2 的 insert 申请了 IX 锁，因为 session1 会话已经插入成功并且持有唯一键 a=15 的 X 行锁 ，故而 session2 insert 进行唯一性检查，先申请 **`LOCK_S + LOCK_ORDINARY`**，事务日志列表中提示 **`lock mode S waiting`**。

#### 第三部 session3 执行插入操作：

``` sql
insert into t6(id,a) values(8,15);

---TRANSACTION 462308739, ACTIVE 3 sec inserting

mysql tables in use 1, locked 1

LOCK WAIT 2 lock struct(s), heap size 360, 1 row lock(s), undo log entries 1

MySQL thread id 3825764, OS thread handle 0x7f78ea593700, query id 781149555 localhost root update

insert into t6(id,a) values(8,15)

------- TRX HAS BEEN WAITING 3 SEC FOR THIS LOCK TO BE GRANTED:

RECORD LOCKS space id 227 page no 4 n bits 80 index `idx_a` of table `test`.`t6` trx id 462308739 lock mode S waiting

------------------

TABLE LOCK table `test`.`t6` trx id 462308739 lock mode IX

RECORD LOCKS space id 227 page no 4 n bits 80 index `idx_a` of table `test`.`t6` trx id 462308739 lock mode S waiting

---TRANSACTION 462308738, ACTIVE 35 sec inserting

mysql tables in use 1, locked 1

LOCK WAIT 2 lock struct(s), heap size 360, 1 row lock(s), undo log entries 1

MySQL thread id 3825768, OS thread handle 0x7f78ea9c9700, query id 781149521 localhost root update

insert into t6(id,a) values(7,15)

------- TRX HAS BEEN WAITING 35 SEC FOR THIS LOCK TO BE GRANTED:

RECORD LOCKS space id 227 page no 4 n bits 80 index `idx_a` of table `test`.`t6` trx id 462308738 lock mode S waiting

------------------

TABLE LOCK table `test`.`t6` trx id 462308738 lock mode IX

RECORD LOCKS space id 227 page no 4 n bits 80 index `idx_a` of table `test`.`t6` trx id 462308738 lock mode S waiting

---TRANSACTION 462308737, ACTIVE 97 sec

2 lock struct(s), heap size 360, 1 row lock(s), undo log entries 1

MySQL thread id 3825779, OS thread handle 0x7f78eacd9700, query id 781149560 localhost root init

show engine innodb status

TABLE LOCK table `test`.`t6` trx id 462308737 lock mode IX

RECORD LOCKS space id 227 page no 4 n bits 80 index `idx_a` of table `test`.`t6` trx id 462308737 lock_mode X locks rec but not gap
```

与会话 session2 的加锁申请流程一致，都在等待 session1 释放锁资源。

#### 第四步 sess1 执行回滚操作，sess2 不提交

session1 **`rollback;`**

此时，session2 插入成功，session3 出现死锁。此时，session2 insert 插入成功，还未提交，事务列表如下:

``` sql
------------
TRANSACTIONS
------------

Trx id counter 462308744

Purge done for trx s n:o < 462308744 undo n:o < 0 state: running but idle

History list length 1866

LIST OF TRANSACTIONS FOR EACH SESSION:

---TRANSACTION 462308737, not started

MySQL thread id 3825779, OS thread handle 0x7f78eacd9700, query id 781149626 localhost root init

show engine innodb status

---TRANSACTION 462308739, not started

MySQL thread id 3825764, OS thread handle 0x7f78ea593700, query id 781149555 localhost root cleaning up

---TRANSACTION 462308738, ACTIVE 75 sec

5 lock struct(s), heap size 1184, 3 row lock(s), undo log entries 1

MySQL thread id 3825768, OS thread handle 0x7f78eadce700, query id 781149608 localhost root cleaning up

TABLE LOCK table `test`.`t6` trx id 462308738 lock mode IX

RECORD LOCKS space id 227 page no 4 n bits 80 index `idx_a` of table `test`.`t6` trx id 462308738 lock mode S

RECORD LOCKS space id 227 page no 4 n bits 80 index `idx_a` of table `test`.`t6` trx id 462308738 lock mode S

RECORD LOCKS space id 227 page no 4 n bits 80 index `idx_a` of table `test`.`t6` trx id 462308738 lock_mode X insert intention

RECORD LOCKS space id 227 page no 4 n bits 80 index `idx_a` of table `test`.`t6` trx id 462308738 lock mode S locks gap before rec
```

## 二、死锁的原因

1. session1 insert 成功并针对 a=15 的唯一键加上 X 锁。

2. session2 执行 insert 插入 (6,15)，在插入之前进行唯一性检查发现和 session1 的已经插入的记录重复，需要申请 **`LOCK_S|LOCK_ORDINARY`**，但与 **session1** 的（**`LOCK_X | LOCK_REC_NOT_GAP`**）冲突，加入等待队列，等待 session1 释放锁。

3. sess3 执行insert 插入(7,15), 在插入之前进行唯一性检查发现和sess1的已经插入的记录重复键需要申请LOCK_S|LOCK_ORDINARY, 但与sess1 的(LOCK_X | LOCK_REC_NOT_GAP)冲突,加入等待队列,等待sess1 释放锁。
 sess1 执行rollback, sess1 释放索引a=15 上的排他记录锁(LOCK_X | LOCK_REC_NOT_GAP),此后 sess2和sess3 获得S锁(LOCK_S|LOCK_ORDINARY)成功,sess2和sess3都要请求索引a=15上的排他记录锁(LOCK_X | LOCK_REC_NOT_GAP),日志中提示 lock_mode X insert intention。由于X锁与S锁互斥，sess2和sess3都等待对方释放S锁，于是出现死锁，MySQL 选择回滚其中之一。

# 三、总结

死锁分析是已经很有挑战的事情，尤其对于insert 唯一键冲突,要分多个阶段去申请，也要理解锁的兼容矩阵。对于这块我还有需要在学习了解的知识点，本文算是抛砖引玉，如有分析理解不正确的地方，望大家指正。

