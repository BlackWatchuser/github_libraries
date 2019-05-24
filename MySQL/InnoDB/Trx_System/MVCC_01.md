# InnoDB 多版本（MVCC）实现简要分析

[原文](http://hedengcheng.com/?p=148)

## 基本知识

假设对于多版本（MVCC）的基础知识，有所了解。InnoDB 为了实现多版本的一致读，采用的是基于回滚段的协议。

### 行结构

InnoDB 表数据的组织方式为主键聚簇索引。由于采用索引组织表结构，记录的 ROWID 是可变的（索引页分裂的时候，`Structure Modification Operation，SMO`），因此二级索引中采用的是（索引键值, 主键键值）的组合来唯一确定一条记录。

**`无论是聚簇索引，还是二级索引，其每条记录都包含了一个 DELETED BIT 位，用于标识该记录是否是删除记录。除此之外，聚簇索引记录还有两个系统列：DATA_TRX_ID，DATA_ROLL_PTR。DATA_TRX_ID 表示产生当前记录项的事务ID；DATA _ROLL_PTR 指向当前记录项的 Undo 信息`**。

聚簇索引行结构（与多版本一致读有关的部分，DELETED BIT 省略）：

![]()

二级索引行结构：

![]()

从聚簇索引行结构，与二级索引行结构可以看出，**`聚簇索引中包含版本信息（事务号 + 回滚指针），二级索引不包含版本信息`**，二级索引项的可见性如何判断？下面将会给出。

### Read View

InnoDB 默认的隔离级别为 Repeatable Read（RR），可重复读。InnoDB 在开始一个 RR 读之前，会创建一个 Read View。**`Read View 用于判断一条记录的可见性`**。Read View 定义在 `read0read.h` 文件中，其中最主要的与可见性相关的属性如下：

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

简单来说，Read View 记录读开始时，所有的活动事务，这些事务所做的修改对于 Read View 是不可见的。除此之外，所有其他的小于创建 Read View 的事务号的所有记录均可见。可见包括两层含义：

* 记录可见，且 Deleted bit = 0；当前记录是可见的有效记录。

* 记录可见，且 Deleted bit = 1；当前记录是可见的删除记录。此记录在本事务开始之前，已经删除。

## 测试方法
–create table and index

create table test (id int primary key, comment char(50)) engine=InnoDB;

create index test_idx on test(comment);

–Insert

insert into test values(1, ‘aaa’);

insert into test values(2, ‘bbb’);

 

–update primary key

update test set id = 9 where id = 1;

 

–update non-primary key with different value

update test set comment = ‘ccc’ where id = 9;

 

–update non-primary key with same value

update test set comment = ‘bbb’ where id = 2 and comment = ‘bbb’;

 

–read隔离级别

repeatable read（RR）

测试结果
update primary key
代码调用流程：

ha_innobase::update_row -> row_update_for_mysql -> row_upd_step -> row_upd -> row_upd_clust_step -> row_upd_clust_rec_by_insert -> btr_cur_del_mark_set_clust_rec -> row_ins_index_entry

简单来说，就是将cluster index的旧记录标记位删除；插入一条新纪录。该语句执行完之后，数据结构如下：