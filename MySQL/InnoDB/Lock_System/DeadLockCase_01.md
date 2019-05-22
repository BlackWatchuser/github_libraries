# 死锁案例（一）

> **`DELETE 申请了 GAP 锁与 INSERT 的 GAP 锁冲突导致了死锁`**

## 一、案例分析

### 1.1 环境说明

MySQL 5.6 事务隔离级别为 **`RR`**：

``` sql
CREATE TABLE `ty` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
   `a` int(11) DEFAULT NULL,
   `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idxa` (`a`)
) ENGINE=InnoDB AUTO_INCREMENT=8 DEFAULT CHARSET=utf8mb4

insert into ty (a,b) values (2,3),(5,4),(6,7);
```

### 1.2 测试用例

| T2 | T1 |
| :- | :- |
| begin; |  |
| delete from  ty where  a=5; | begin; |
|  | delete from  ty where  a=5; |
| insert into ty(a,b) values(2,10); |  |
|  | delete from  ty where  a=5; <br>ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction |
                                                 
### 1.3 死锁日志

``` sql
------------------------
LATEST DETECTED DEADLOCK
------------------------

2017-09-09 22:34:13 7f78eab82700

*** (1) TRANSACTION:

TRANSACTION 462308399, ACTIVE 33 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 360, 1 row lock(s)
MySQL thread id 3525577, OS thread handle 0x7f896cc4b700, query id 780039657 localhost root updating

delete from ty where a=5

*** (1) WAITING FOR THIS LOCK TO BE GRANTED:

RECORD LOCKS space id 219 page no 4 n bits 72 index `idxa` of table `test`.`ty` trx id 462308399 lock_mode X waiting

*** (2) TRANSACTION:

TRANSACTION 462308398, ACTIVE 61 sec inserting, thread declared inside InnoDB 5000
mysql tables in use 1, locked 1
5 lock struct(s), heap size 1184, 4 row lock(s), undo log entries 2
MySQL thread id 3525490, OS thread handle 0x7f78eab82700, query id 780039714 localhost root update

insert into ty(a,b) values(2,10)

*** (2) HOLDS THE LOCK(S):

RECORD LOCKS space id 219 page no 4 n bits 72 index `idxa` of table `test`.`ty` trx id 462308398 lock_mode X

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:

RECORD LOCKS space id 219 page no 4 n bits 72 index `idxa` of table `test`.`ty` trx id 462308398 lock_mode X locks gap before rec insert intention waiting

*** WE ROLL BACK TRANSACTION (1)
```

### 1.4 分析死锁日志

首先要理解的是，**`对同一个字段申请加锁是需要排队`**。 

其次表 ty 中索引 idxa 为**非唯一普通索引**，我们根据事务执行的时间顺序来解释，这样比较好理解。

| id | a | b |
| :-: | :-: | :-: |
| 1 | 2 | 3 |
| 2 | 5 | 4 |
| 3 | 6 | 7 |

1. 根据死锁日志显示 事务2 也即 **session1** 执行的事务，根据 `HOLDS THE LOCK(S)` 显示 **session1** 先执行 `delete from ty where a=5`，该事务持有索引 a=5 的行锁 `lock_mode X`，**因为是 RR 隔离级别，所以 **session1** 还持有两个 gap 锁 [1,2]-[2,5] 和 [2,5]-[3,6]**。

2. 事务1 的日志 也即 **session2** 执行的事务，申请对 a=5 加锁，一个 **`rec lock`** 和两个 gap 锁，因为 **session1** 中 delete 还没释放，故 **session2** 的事务1等待 **session1** 的事务2释放 a=5 的锁资源。

3. 然后根据 **`WAITING FOR THIS LOCK TO BE GRANTED`**，提示事务2 **insert** 语句正在等待 **`lock_mode X locks gap before rec insert intention waiting`**，因为 **insert** 语句 [4,2] 介于 gap 锁 [1,2]-[2,5] 之间，所以有了提示 "**`lock_mode X locks gap`**"，**insert** 语句必须等待前面 **session2** 中 delete 获取锁并且释放锁。

于是，**session2** (**delete**) 等待 **session1** (**delete**)，**session1** (**insert**) 等待 **session2** (**delete**)，循环等待，造成死锁。

问题：如果 **session1** 执行 `insert into ty(a,b) values(5,10);` **session2** 会遇到死锁吗？

## 二、案例（二）

### 2.1 索引为唯一键

MySQL 5.6 事务隔离级别为 **`RR`**：

``` sql
CREATE TABLE `t2` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
   `a` int(11) DEFAULT NULL,
   `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  unique KEY `idxa` (`a`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8mb4;

insert into t2 (a,b) values (2,3),(5,4),(6,7)
```

### 2.2 测试用例

| T2 | T1 |
| :- | :- |
| begin; |  |
| delete from ty where a=5; | begin; |
|  | delete from ty where a=5; |
| insert into ty(a,b) values(2,10); |  |
|  | delete from  ty where  a=5; <br>ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction |

### 2.3 死锁日志

``` sql
------------------------
LATEST DETECTED DEADLOCK
------------------------

2017-09-10 00:03:31 7f78ea936700

*** (1) TRANSACTION:

TRANSACTION 462308445, ACTIVE 9 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 360, 1 row lock(s)
MySQL thread id 3526009, OS thread handle 0x7f896cc4b700, query id 780047877 localhost root updating

delete from t2 where a=5

*** (1) WAITING FOR THIS LOCK TO BE GRANTED:

RECORD LOCKS space id 221 page no 4 n bits 72 index `idxa` of table `test`.`t2` trx id 462308445 lock_mode X waiting

*** (2) TRANSACTION:

TRANSACTION 462308444, ACTIVE 17 sec inserting, thread declared inside InnoDB 5000
mysql tables in use 1, locked 1
4 lock struct(s), heap size 1184, 3 row lock(s), undo log entries 2
MySQL thread id 3526051, OS thread handle 0x7f78ea936700, query id 780047890 localhost root update

insert t2(a,b) values(5,10)

*** (2) HOLDS THE LOCK(S):

RECORD LOCKS space id 221 page no 4 n bits 72 index `idxa` of table `test`.`t2` trx id 462308444 lock_mode X locks rec but not gap

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:

RECORD LOCKS space id 221 page no 4 n bits 72 index `idxa` of table `test`.`t2` trx id 462308444 lock mode S waiting

*** WE ROLL BACK TRANSACTION (1)
```

### 2.4 分析死锁日志

首先我们要特别说明 delete 的加锁逻辑：

1. **`找到满足条件的记录，并且记录有效，则对记录加 X 锁，No Gap 锁（lock_mode X locks rec but not gap）`**;

2. **`找到满足条件的记录，但是记录无效（标识为删除的记录），则对记录加 next key 锁（同时锁住记录本身，以及记录之前的 Gap:lock_mode X）`**；

3. **`未找到满足条件的记录，则对第一个不满足条件的记录加 Gap 锁，保证没有满足条件的记录插入（locks gap before rec）`**；

其次需要大家注意的是对比两个死锁案例会发现，**session1** 事务持有的锁类型发生了变化：delete 持有的锁变为 **`lock_mode X locks rec but not gap`**。 insert 语句持有的锁变为 **`lock mode S waiting`**。原因是因为测试表结构发生了变化字段 a 由普通索引变为唯一键，**RR** 模式下对唯一键操作是没有 GAP 锁的，而且 insert 写入含有唯一键的数据是会有申请 GAP 锁的特殊情况的 **`Insert Intention Lock`**。

本例我们依然根据事务执行的时间顺序来解释，这样比较好理解。

1. 根据死锁日志显示 事务2 也即 **session1** 执行的事务，根据 **HOLDS THE LOCK(S)** 显示 **session1** 先执行 `delete from ty where a=5`，该事务持有索引 a=5 的行锁 **`lock_mode X locks rec but not gap`**。因为本例中 a 是唯一键，故没有 gap 锁。

2. 事务1 的日志也即 **session2** 执行的事务，申请对 a=5 加锁（**`X Next-key Lock`**），一个 `rec lock` 但是因为 **session1** 中 delete 已经执行完成，记录无效没有被删除，锁还没释放，故 **session2** 的 事务1 等待 **session1** 的 事务2 释放 a=5 的锁资源，日志中提示 **`lock_mode X waiting`**。 

3. 然后根据 **`WAITING FOR THIS LOCK TO BE GRANTED`**，提示 事务2 insert 语句正在等待 **`lock mode S waiting`**，为什么这次是 S 锁呢？**`因为 a 字段是一个唯一索引，所以 insert 语句会在插入前进行一次 duplicate key 的检查，需要申请 S 锁防止其他事务对 a 字段进行重复插入`**。而插入意向锁与 insert 语句必须等待前面 **session2** 中 delete 获取 a=5 的行锁并且释放锁。

于是，**session2**（**delete**）等待 **session1**（**delete**），**session1**（**insert**）等待 **session2**（**delete**），循环等待，造成死锁。

## 三、小结

本文研究了 **`RR`** 事务隔离级别下，普通索引与唯一键两种情况的死锁场景。如何避免解决此类死锁？推荐使用 **`RC 隔离级别 + ROW BASE BINLOG`**。但是 **`对于 RC/RR 模式下 ，insert 遇到唯一键冲突时的死锁不可避免`**。需要开发在设计表结构的时候减少 Unique 索引设计。

------

# 死锁案例（二）

> **`并发 DELETE 不存在的记录申请 GAP 锁导致的死锁`**

> **`RR 模式下两个事务中的 SQL 可以获取同一个 GAP 锁，导致双方事务的 INSERT 相互等待`**

## 一、案例分析

### 1.1 测试环境准备

Percona server 5.6.24 事务隔离级别为 **RR**：

``` sql
CREATE TABLE `t4` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `kdt_id` int(11) unsigned NOT NULL,
  `admin_id` int(11) unsigned NOT NULL,
  `biz` varchar(20) NOT NULL DEFAULT '1',
  `role_id` int(11) unsigned NOT NULL,
  `shop_id` int(11) unsigned NOT NULL DEFAULT '0',
  `operator` varchar(20) NOT NULL DEFAULT '0',
  `operator_id` int(11) NOT NULL DEFAULT '0',
  `create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '更新时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uniq_kid_aid_biz_rid` (`kdt_id`,`admin_id`,`role_id`,`biz`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;

INSERT INTO `t4` (`id`, `kdt_id`, `admin_id`, `biz`, `role_id`, `shop_id`, `operator`, `operator_id`, `create_time`, `update_time`)
VALUES
(1,10,1,'retail',1,0,'0',0,'2017-05-09 15:55:26','2017-05-09 15:55:26'),
(2,20,1,'retail',1,0,'0',0,'2017-05-09 15:55:40','2017-05-09 15:55:40'),
(3,30,1,'retail',1,0,'0',0,'2017-05-09 15:55:55','2017-05-09 15:55:55'),
(4,40,1,'retail',1,0,'0',0,'2017-05-09 15:56:06','2017-05-09 15:56:06'),
(5,50,1,'retail',1,0,'0',0,'2017-05-09 15:56:16','2017-05-09 15:56:16');
```

### 1.2 本测试案例场景是两个事务都删除不存的行，然后再 INSERT 记录：

| T2 | T1 |
| :- | :- |
| test [RW] 02:50:27 > begin; <br>Query OK, 0 rows affected (0.00 sec) | test [RW] 02:50:27 > begin; <br>Query OK, 0 rows affected (0.00 sec) |
| test [RW] 02:50:34 > delete from t4 where kdt_id = 15 and admin_id = 1 and biz = 'retail' and role_id = '1'; |  |
|  | test [RW] 02:50:41 > delete from t4 where kdt_id = 18 and admin_id = 2 and biz = 'retail' and role_id = '1'; |
|  | test [RW] 02:50:43 > INSERT INTO t4(`kdt_id`, `admin_id`, `biz`, `role_id`, `shop_id`, `operator`, `operator_id`, `create_time`, `update_time`) VALUES ('18', '2', 'retail', '2', '0', '0', '0', CURRENT_TIMESTAMP,CURRENT_TIMESTAMP); |
| test [RW] 02:51:02 > INSERT INTO t4(`kdt_id`, `admin_id`, `biz`, `role_id`, `shop_id`, `operator`, `operator_id`, `create_time`, `update_time`) VALUES ('15', '1', 'retail', '2', '0', '0', '0', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP); <br>ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction |  |

### 1.3 死锁日志

``` sql
------------------------
LATEST DETECTED DEADLOCK
------------------------

2017-09-11 14:51:03 7f78eaf25700

*** (1) TRANSACTION:

TRANSACTION 462308535, ACTIVE 20 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 360, 2 row lock(s), undo log entries 1
MySQL thread id 3584515, OS thread handle 0x7f78ea5f5700, query id 780258123 localhost root update

insert into t4(`kdt_id`, `admin_id`, `biz`, `role_id`, `shop_id`, `operator`, `operator_id`, `create_time`, `update_time`) VALUES ('18', '2', 'retail', '2', '0', '0', '0', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP)

*** (1) WAITING FOR THIS LOCK TO BE GRANTED:

RECORD LOCKS space id 225 page no 4 n bits 72 index `uniq_kid_aid_biz_rid` of table `test`.`t4` trx id 462308535 lock_mode X locks gap before rec insert intention waiting

*** (2) TRANSACTION:

TRANSACTION 462308534, ACTIVE 29 sec inserting, thread declared inside InnoDB 5000
mysql tables in use 1, locked 1
3 lock struct(s), heap size 360, 2 row lock(s), undo log entries 1
MySQL thread id 3584572, OS thread handle 0x7f78eaf25700, query id 780258153 localhost root update

INSERT INTO t4(`kdt_id`, `admin_id`, `biz`, `role_id`, `shop_id`, `operator`, `operator_id`, `create_time`, `update_time`) VALUES ('15', '1', 'retail', '2', '0', '0', '0', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP)

*** (2) HOLDS THE LOCK(S):

RECORD LOCKS space id 225 page no 4 n bits 72 index `uniq_kid_aid_biz_rid` of table `test`.`t4` trx id 462308534 lock_mode X locks gap before rec

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:

RECORD LOCKS space id 225 page no 4 n bits 72 index `uniq_kid_aid_biz_rid` of table `test`.`t4` trx id 462308534 lock_mode X locks gap before rec insert intention waiting

*** WE ROLL BACK TRANSACTION (2)
```

### 1.4 死锁日志分析

首先根据《死锁案例一》和《一个最不可思议的 MySQL 死锁分析》中强调 DELETE 不存在的记录是要加上 GAP 锁，事务日志中显示 **`Lock_mode X wait`**。

1. T2：`delete from t4 where kdt_id = 15 and admin_id = 1 and biz = 'retail' and role_id = '1';` 符合条件的记录不存在，导致 T2 先持有了（**`lock_mode X locks gap before rec`**）锁住 **`[(1,10,1,'retail',1,0) - (2,20,1,'retail',1,0)]`** 的区间，防止符合条件的记录插入。

2. T1 的 delete 与 T2 的 delete 一样，同样申请了（**`lock_mode X locks gap before rec`**）锁住 **`[(1,10,1,'retail',1,0) - (2,20,1,'retail',1,0)]`** 的区间。

> It is also worth noting here that **`conflicting locks can be held on a gap by different transactions`**. For example, transaction A can hold a shared gap lock (gap S-lock) on a gap while transaction B holds an exclusive gap lock (gap X-lock) on the same gap. The reason conflicting gap locks are allowed is that if a record is purged from an index, the gap locks held on the record by different transactions must be merged.

3. T1 的 insert 语句申请插入意向锁，但是插入意向锁和 T2 持有的 X GAP（**`lock_mode X locks gap before rec`**）冲突，故等待 T2 中的 GAP 锁释放。

> Gap locks in InnoDB are "purely inhibitive", **`which means they only stop other transactions from inserting to the gap. They do not prevent different transactions from taking gap locks on the same gap`**. Thus, a gap X-lock has the same effect as a gap S-lock.

4. T2 的 insert 语句申请插入意向锁，但是插入意向锁和 T1 持有的 X GAP（**`lock_mode X locks gap before rec`**）冲突，故等待 T1 中的 GAP 锁释放。

T1(INSERT ）等待T2(DELETE),T2(INSERT)等待T1(DELETE) 故而循环等待，出现死锁。
有兴趣的读者朋友可以测试一下 delete 存在记录的场景。

### 1.5 如何解决呢？

1. 先 **SELECT** 检查一下看看是否存在，然后再删除。这里也存在两个或者多个会话并发执行同一个 SELECT WHERE 条件的，这里需要开发同学做处理。

2. 使用 **`insert into on deuplicate key`** 语法不存在则插入，而不是先删除再插入。

## 二、小结

**RR** 事务隔离级别和 GAP 锁是导致死锁的常见原因，但是业务逻辑设计不合理也会触发死锁，本文的案例可以通过修改业务逻辑最终将死锁解决。

------

# 死锁案例（三）

## 一、背景知识

### 1.1 insert 锁机制

在分析死锁案例之前，我们先学习一下背景知识 insert 语句的加锁策略。我们先来看看官方定义:

> **`An insert intention lock is a type of gap lock set by INSERT operations prior to row insertion. This lock signals the intent to insert in such a way that multiple transactions inserting into the same index gap need not wait for each other if they are not inserting at the same position within the gap. Suppose that there are index records with values of 4 and 7. Separate transactions that attempt to insert values of 5 and 6, respectively, each lock the gap between 4 and 7 with insert intention locks prior to obtaining the exclusive lock on the inserted row, but do not block each other because the rows are nonconflicting.`**

相信大部分的 DBA 同行都知道在事务执行 insert 的时候会申请一把插入意向锁（**`Insert Intention Lock`**)。在多事务并发写入不同数据记录至同一索引间隙的时候，并不需要等待其他事务完成，不会发生锁等待。

假设有一个索引记录包含键值 4 和 7，不同的事务分别插入 5 和 6，每个事务都会产生一个加在 4 - 7 之间的插入意向锁，获取在插入行上的排它锁，但是不会被互相锁住，因为数据行并不冲突。

但是如果遇到唯一键呢？ 

> **`If a duplicate-key error occurs, a shared lock on the duplicate index record is set.`**

**`对于 INSERT 操作来说，若发生唯一约束冲突，则需要对冲突的唯一索引加上 S Next-key Lock`**。从这里会发现，即使是 **RC** 事务隔离级别，也同样会存在 **Next-Key Lock** 锁，从而阻塞并发。然而，文档没有说明的是，**`对于检测到冲突的唯一索引，等待线程在获得 S Lock 之后，还需要对下一个记录进行加锁`**，在源码中由函数 `row_ins_scan_sec_index_for_duplicate` 进行判断。via（MySQL REPLACE 死锁问题深入剖析）。我们可以通过如下例子进行验证：

### 1.2 验证

准备环境，默认事务隔离级别为 **RC** 模式：

``` sql
CREATE TABLE t8 (
  a int AUTO_INCREMENT PRIMARY KEY,
  b int,
  c int,
unique key ub(b)
) engine=InnoDB;

insert into t8 values (NULL,1,2)
```

| session1 | session2 |
| :- | :- |
| begin; |  |
| delete from t8 where b = 1; | begin; |
|  | insert into t8 values (NULL,1,2); |
| commit; |  |
|  | update t8 set c=13 where b=1; |

### 1.3 过程分析

在每次执行一条语句之后都执行 **`show innodb engine status`** 查看事务的状态，
**执行完 delete 语句后**，事务相关日志显示如下:

``` sql
---TRANSACTION 462308671, ACTIVE 6 sec

3 lock struct(s), heap size 360, 2 row lock(s), undo log entries 1

MySQL thread id 3796960, OS thread handle 0x7f78eaabe700, query id 781051370 localhost root init

show engine innodb status

TABLE LOCK table `test`.`t8` trx id 462308671 lock mode IX

RECORD LOCKS space id 232 page no 4 n bits 72 index `ub` of table `test`.`t8` trx id 462308671 lock_mode X locks rec but not gap

RECORD LOCKS space id 232 page no 3 n bits 72 index `PRIMARY` of table `test`.`t8` trx id 462308671 lock_mode X locks rec but not gap
```

从日志中我们可以看到 delete 语句获取了唯一索引 ub 和主键两个行级锁（**`lock_mode X locks rec but not gap`**）。

**执行完 insert 之后**，再查看 `innodb engine status`，事务相关日志显示如下:

``` sql
LIST OF TRANSACTIONS FOR EACH SESSION:

---TRANSACTION 462308676, ACTIVE 4 sec inserting

mysql tables in use 1, locked 1

LOCK WAIT 2 lock struct(s), heap size 360, 1 row lock(s), undo log entries 1

MySQL thread id 3796966, OS thread handle 0x7f78ea5c4700, query id 781051460 localhost root update

insert into t8 values (NULL,1,2)

------- TRX HAS BEEN WAITING 4 SEC FOR THIS LOCK TO BE GRANTED:

RECORD LOCKS space id 232 page no 4 n bits 72 index `ub` of table `test`.`t8` trx id 462308676 lock mode S waiting

------------------

TABLE LOCK table `test`.`t8` trx id 462308676 lock mode IX

RECORD LOCKS space id 232 page no 4 n bits 72 index `ub` of table `test`.`t8` trx id 462308676 lock mode S waiting

---TRANSACTION 462308671, ACTIVE 70 sec

3 lock struct(s), heap size 360, 2 row lock(s), undo log entries 1

MySQL thread id 3796960, OS thread handle 0x7f78eaabe700, query id 781051465 localhost root init

show engine innodb status

TABLE LOCK table `test`.`t8` trx id 462308671 lock mode IX

RECORD LOCKS space id 232 page no 4 n bits 72 index `ub` of table `test`.`t8` trx id 462308671 lock_mode X locks rec but not gap

RECORD LOCKS space id 232 page no 3 n bits 72 index `PRIMARY` of table `test`.`t8` trx id 462308671 lock_mode X locks rec but not gap
```

根据官方的介绍，并结合日志，我们可以看到 `insert into t8 values (NULL,1,2);` 在申请一把 **`S Next-key-Lock`** 锁，显示 **`lock mode S waiting`**。这里想给大家说明的是在 InnoDB 日志中如果提示 **`lock mode S / lock mode X`**，其实都是 Gap 锁，如果是行记录锁会提示 **but not gap**，请读者朋友们在自己分析死锁日志的时候注意。

session1 delete 语句提交之后，session2 的 insert 不要提交，不要提交，不要提交。再次查看 `innodb engine status`，事务相关日志显示如下：

``` sql
------------
TRANSACTIONS
------------

Trx id counter 462308678

Purge done for trxs n:o < 462308678 undo n:o < 0 state: running but idle

History list length 1845

LIST OF TRANSACTIONS FOR EACH SESSION:

---TRANSACTION 462308671, not started

MySQL thread id 3796960, OS thread handle 0x7f78eaabe700, query id 781051526 localhost root init

show engine innodb status

---TRANSACTION 462308676, ACTIVE 41 sec

3 lock struct(s), heap size 360, 2 row lock(s), undo log entries 1

MySQL thread id 3796966, OS thread handle 0x7f78ea5c4700, query id 781051460 localhost root cleaning up

TABLE LOCK table `test`.`t8` trx id 462308676 lock mode IX

RECORD LOCKS space id 232 page no 4 n bits 72 index `ub` of table `test`.`t8` trx id 462308676 lock mode S

RECORD LOCKS space id 232 page no 4 n bits 72 index `ub` of table `test`.`t8` trx id 462308676 lock mode S locks gap before rec
```

session1 中的事务因为提交已经结束。InnoDB 中的事务列表中只剩下 session2 中的 insert 的事务了。从获取锁的状态上看 insert 获取了一把 **`S Next-key Lock`** 锁和插入行之前的 **`S GAP`** 锁。看到这里大家是否有疑惑，官方文档说:

> INSERT sets an exclusive lock on the inserted row. This lock is an index-record lock, not a next-key lock (that is, there is no gap lock) and does not prevent other sessions from inserting into the gap before the inserted row.

insert 成功的记录上会加一把 X 行锁，为什么看不见呢？我们再在 session1 中执行 `update t8 set c=13 where b=1;` 并查看事务日志：

``` sql
------------
TRANSACTIONS
------------

Trx id counter 462308679

Purge done for trxs n:o < 462308678 undo n:o < 0 state: running but idle

History list length 1845

LIST OF TRANSACTIONS FOR EACH SESSION:

---TRANSACTION 462308678, ACTIVE 12 sec starting index read

mysql tables in use 1, locked 1

LOCK WAIT 2 lock struct(s), heap size 360, 1 row lock(s)

MySQL thread id 3796960, OS thread handle 0x7f78eaabe700, query id 781059217 localhost root updating

update c set c=13 where b=1

------- TRX HAS BEEN WAITING 12 SEC FOR THIS LOCK TO BE GRANTED:

RECORD LOCKS space id 232 page no 4 n bits 72 index `ub` of table `test`.`t8` trx id 462308678 lock_mode X locks rec but not gap waiting

------------------

TABLE LOCK table `test`.`t8` trx id 462308678 lock mode IX

RECORD LOCKS space id 232 page no 4 n bits 72 index `ub` of table `test`.`t8` trx id 462308678 lock_mode X locks rec but not gap waiting

---TRANSACTION 462308676, ACTIVE 5113 sec

4 lock struct(s), heap size 1184, 3 row lock(s), undo log entries 1

MySQL thread id 3796966, OS thread handle 0x7f78ea5c4700, query id 781059230 localhost root init

show engine innodb status

TABLE LOCK table `test`.`t8` trx id 462308676 lock mode IX

RECORD LOCKS space id 232 page no 4 n bits 72 index `ub` of table `test`.`t8` trx id 462308676 lock mode S

RECORD LOCKS space id 232 page no 4 n bits 72 index `ub` of table `test`.`t8` trx id 462308676 lock mode S locks gap before rec

RECORD LOCKS space id 232 page no 4 n bits 72 index `ub` of table `test`.`t8` trx id 462308676 lock_mode X locks rec but not gap
```

从日志中可以看到 **session2** 的事务持有的锁多了一把 **`lock_mode X locks rec but not gap`**，也即是 session2 对 insert 成功的记录加上的 X 行锁。 

分析至此，对于并发 insert 造成唯一键冲突的时候 insert 的加锁策略是:

**`第一阶段 为唯一性约束检查，先申请 LOCK_S + LOCK_ORDINARY`**

**`第二阶段 获取阶段一的锁并且成功执行 insert 语句`**

**`插入的位置有 Gap 锁：LOCK_INSERT_INTENTION`**

**`新数据插入：LOCK_X + LOCK_REC_NOT_GAP`**

## 二、案例分析

> **`本案例是两个事务并发 insert 唯一键冲突和 Gap 锁一起导致的死锁案例。`**

### 2.1 环境 

``` sql
create table t7(
  id int not null primary key auto_increment,
  a int not null ,
  unique key ua(a)
) engine=innodb;

insert into t7 (id,a) values (1,1),(5,4),(20,20),(25,12);
```

### 2.2 测试用例

| T1 | T2 |
| :- | :- |
| begin; | begin; |
|  | insert into t7 (id,a) values (26,10); |
| insert into t7 (id,a) values (30,10); |  |
|  | insert into t7 (id,a) values (40,9); |

### 2.3 死锁日志

``` sql
------------------------
LATEST DETECTED DEADLOCK
------------------------

2017-09-17 15:15:03 7f78eac15700

*** (1) TRANSACTION:

TRANSACTION 462308661, ACTIVE 6 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 360, 1 row lock(s), undo log entries 1
MySQL thread id 3796966, OS thread handle 0x7f78ead9d700, query id 781045166 localhost root update

insert into t7 (id,a) values (30,10)

*** (1) WAITING FOR THIS LOCK TO BE GRANTED:

RECORD LOCKS space id 231 page no 4 n bits 72 index `ua` of table `test`.`t7` trx id 462308661 lock mode S waiting

*** (2) TRANSACTION:

TRANSACTION 462308660, ACTIVE 43 sec inserting, thread declared inside InnoDB 5000
mysql tables in use 1, locked 1
4 lock struct(s), heap size 1184, 3 row lock(s), undo log entries 2
MySQL thread id 3796960, OS thread handle 0x7f78eac15700, query id 781045192 localhost root update

insert into t7 (id,a) values (40,9)

*** (2) HOLDS THE LOCK(S):

RECORD LOCKS space id 231 page no 4 n bits 72 index `ua` of table `test`.`t7` trx id 462308660 lock_mode X locks rec but not gap

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:

RECORD LOCKS space id 231 page no 4 n bits 72 index `ua` of table `test`.`t7` trx id 462308660 lock_mode X locks gap before rec insert intention waiting

*** WE ROLL BACK TRANSACTION (1)
```

### 2.4 日志分析

我们从时间线维度分析：

```
+----+----+
| id | a  |
+----+----+
|  1 |  1 |
|  5 |  4 |
| 25 | 12 |
| 20 | 20 |
+----+----+
```

**事务T2** `insert into t7 (id,a) values (26,10)` 语句 insert 成功，持有 a=10 的 X 行锁（**`X locks rec but not gap`**）；

**事务T1** `insert into t7 (id,a) values (30,10)`，因为 T2 的第一条 insert 已经插入 a=10 的记录，事务 T1 的 insert a=10 则发生唯一约束冲突，需要申请对冲突的唯一索引加上 **`S Next-key Lock（也即是 lock mode S waiting）`**；这是一个间隙锁会申请锁住 **`[4,10]`** 之间的 Gap 区域。从这里会发现，即使是 **RC** 事务隔离级别，也同样会存在 **Next-Key Lock** 锁，从而阻塞并发。

**事务T2** `insert into t7 (id,a) values (40,9)`，该语句插入的 a=9 在 事务T1 申请的 Gap 锁 **`[4,10]`** 之间，故 事务T2 的 insert 语句需要等待 事务T1 的 **S-Next-key Lock** 锁释放，在日志中显示 **`lock_mode X locks gap before rec insert intention waiting`**。

## 三、总结 

本文案例和知识点一方面从官方文档获取，另一方面是根据何登成和姜承尧两位 MySQL 技术大牛的技术分享整理，算是站在巨人的肩膀上的学习总结。在研究分析死锁案例的过程中，insert 的意向锁 和 Gap 锁这种类型的锁是比较难分析的，相信通过上面的分析总结大家能够学习到 insert 的锁机制，如何加锁和如何进行 insert 方面的死锁分析。

------