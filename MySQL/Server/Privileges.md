# MySQL 权限存储与管理

[原文](http://mysql.taobao.org/monthly/2015/10/10/)

## 权限相关的表

### 系统表

MySQL用户权限信息都存储在以下系统表中，用户权限的创建、修改和回收都会同步更新到系统表中。
  
```
mysql.user            // 用户信息
mysql.db              // 库上的权限信息
mysql.tables_priv     // 表级别权限信息
mysql.columns_priv    // 列级别权限信息
mysql.procs_priv      // 存储过程和函数的权限信息
mysql.proxies_priv    // MySQL proxy 权限信息
```

> mysql.db存储是库的权限信息，不是存储实例有哪些库。`MySQL查看实例有哪些数据库是通过在数据目录下查找有哪些目录文件得到的。`

### information_schema 表

information_schema 下有以下权限相关的表可供查询：

```
USER_PRIVILEGES
SCHEMA_PRIVILEGES
TABLE_PRIVILEGES
COLUMN_PRIVILEGES
```

## 权限缓存

`用户在连接数据库的过程中，为了加快权限的验证过程，系统表中的权限会缓存到内存中。` 例如：

```
mysql.user         => 数组 acl_users
mysql.db           => 数组 acl_dbs
mysql.tables_priv  => hash表 column_priv_hash
mysql.columns_priv => hash表 column_priv_hash
mysql.procs_priv   => hash表 proc_priv_hash 和 func_priv_hash
```

另外，acl_cache 缓存 db级别的权限信息。例如：执行 use db 时，会尝试从 acl_cache 中查找并更新当前数据库权限（thd->security_ctx->db_access）。

### 权限更新过程

以 grant select on test.t1 为例:

1. 更新系统表 mysql.user，mysql.db，mysql.tables_priv；
2. 更新缓存 acl_users，acl_dbs，column_priv_hash；
3. 清空 acl_cache。

## FLUSH PRIVILEGES

`FLUSH PRIVILEGES 会重新从系统表中加载权限信息来构建缓存。`

当我们通过SQL语句直接修改权限系统表来修改权限时，权限缓存是没有更新的，这样会导致权限缓存和系统表不一致。**因此通过这种方式修改权限后，应执行FLUSH PRIVILEGES来刷新缓存，从而使更新的权限生效。**

通过 GRANT/REVOKE/CREATE USER/DROP USER 来更新权限是不需要 FLUSH PRIVILEGES 的。

> 当前连接修改了权限信息时，现存的其他客户连接是不受影响的，权限在客户端下一次请求时生效。

------

''' sql
select user,host from mysql.user;
'''
