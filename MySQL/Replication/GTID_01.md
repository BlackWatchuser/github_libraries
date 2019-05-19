



mysql> set global gtid_mode='OFF_PERMISSIVE';
Query OK, 0 rows affected (0.01 sec)
mysql> set global gtid_mode='ON_PERMISSIVE';
Query OK, 0 rows affected (0.01 sec)
mysql> set global enforce_gtid_consistency=ON;
Query OK, 0 rows affected (0.00 sec)
mysql> set global gtid_mode='ON';
Query OK, 0 rows affected (0.00 sec)

mysql> stop slave;-----从库执行
Query OK, 0 rows affected (0.00 sec)
 
mysql> set global gtid_mode='ON_PERMISSIVE';
Query OK, 0 rows affected (0.00 sec)
 
mysql> set global gtid_mode='OFF_PERMISSIVE';
Query OK, 0 rows affected (0.01 sec)
 
mysql> CHANGE MASTER TO MASTER_AUTO_POSITION=0;
Query OK, 0 rows affected (0.05 sec)
 
mysql> set global gtid_mode='OFF';
Query OK, 0 rows affected (0.00 sec)
 
mysql> start slave;
Query OK, 0 rows affected (0.00 sec)