# SYSBENCH_MYSQL

``` shell
sysbench /usr/local/share/sysbench/oltp_insert.lua --mysql-host='10.0.17.101' --mysql-port=3309 --mysql-user='dba_ops' --mysql-password='Vip18#kid1T17' --mysql-db='sysbench' --warmup-time=600 --time=3600 --events=0 --threads=16 --tables=16 --table_size=10000000 --mysql_storage_engine='innodb' --rate=0 --histogram=on --percentile=99 prepare / run / cleanup
```