# SYSBENCH_MYSQL

``` shell
sysbench /usr/local/share/sysbench/oltp_insert.lua --mysql-host='10.21.154.187' --mysql-port=3306 --mysql-user='sysbench' --mysql-password='Pyep_0630' --mysql-db='sysbench' --warmup-time=600 --time=3600 --events=0 --tables=16 --threads=11 --percentile=99   --mysql_storage_engine='innodb' --table_size=500000 prepare / run / cleanup
```