# MySQL 8.0 新功能列表

There are over 250 new features in MySQL 8.0. The MySQL Manual is very good, but verbose. This is a list of new features in short bullet form. We have tried very hard to make sure each feature is only mentioned once. Note the similar list for MySQL 5.7.

> **`MySQL 8.0 相对于 MySQL 5.7 新增了 250 多个新功能`**

Please download MySQL 8.0 from dev.mysql.com or from the MySQL Yum, APT, or SUSE repositories.

## SQL DML

```
1. Non-recursive CTEs [1]
2. Recursive CTEs [1]
3. Window functions [1]
4. ORDER BY and DISTINCT with ROLLUP [1]
5. LATERAL derived tables [1]
6. Outer table references in derived tables [1]
```

## SQL DDL

```
1. Instant ADD COLUMN [1]
2. Instant RENAME COLUMN [1]
3. Instant RENAME TABLESPACE [1]
4. RESTART statement [1]
5. SET PERSIST statement [1]
6. RENAME TABLES under LOCK TABLES [1]
7. Option to disallow tables without primary keys [1]
8. Character set conversion as an inplace operation [1 2]
9. CREATE TABLESPACE without DATAFILE clause [1]
10. CREATE RESOURCE GROUP [1 2]
11. ALTER RESOURCE GROUP [1 2]
12. DROP RESOURCE GROUP [1 2]
13. Expressions as DEFAULT values [1]
```

## Indexes

```
1. Invisible indexes [1]
2. Descending indexes [1]
3. Functional indexes [1]
4. Index skip scan [1]
```

## Functions

```
1. New function REGEXP_INSTR [1]
2. New function REGEXP_LIKE [1]
3. New function REGEXP_REPLACE [1]
4. New function REGEXP_SUBSTR [1]
5. New function UUID_TO_BIN [1]
6. New function BIN_TO_UUID [1]
7. New function IS_UUID [1]
8. New function GROUPING [1 2]
9. New function STATEMENT_DIGEST [1]
10. New function STATEMENT_DIGEST_TEXT [1]
11. Bit operations allowed on BINARY, VARBINARY, BLOB, TINYBLOB, MEDIUMBLOB and LONGBLOB [1]
```

## JSON

```
1. New function JSON_PRETTY [1]
2. New function JSON_STORAGE_SIZE [1]
3. New function JSON_STORAGE_FREE [1]
4. New function JSON_MERGE_PATCH [1]
5. New aggregation and window function JSON_ARRAYAGG [1]
6. New aggregation and window function JSON_OBJECTAGG [1]
7. New table function JSON_TABLE [1]
8. Faster sorting of JSON values [1]
9. Ranges in JSON patch expressions [1 2]
10. In-place updates of JSON values [1 2]
```

## GIS

```
1. Spatial reference systems (SRSs) [1 2 3 4]
2. CREATE SPATIAL REFERENCE SYSTEM statement [1 2]
3. DROP SPATIAL REFERENCE SYSTEM statement [1]
4. SRID type modifier [1 2]
5. Geographic R-trees [1]
6. New setter function ST_SRID(geometry, new_srid) [1]
7. New setter function ST_X(geometry, new_x) [1]
8. New setter function ST_Y(geometry, new_y) [1]
9. New function ST_SwapXY [1]
10. New function ST_Latitude [1]
11. New function ST_Longitude [1]
12. New function ST_Transform [1]
13. Geography support in ST_Distance, ST_Contains, ST_Crosses, ST_Disjoint, ST_Equals, ST_Intersects, ST_Overlaps, ST_Touches, ST_Within, MBRContains, MBRCoveredBy, MBRCovers, MBRDisjoint, MBREquals, MBRIntersects, MBROverlaps, MBRTouches, MBRWithin, ST_IsSimple, ST_IsValid, ST_Length, ST_Validate, ST_Area [1 2 3 4 5 6 7 8]
14. ST_Distance_Sphere for geographic geometries [1]
15. GEOMCOLLECTION as synonym to GEOMETRYCOLLECTION [1 2 3]
16. Optional SPATIAL keyword in R-tree index clauses [1]
17. Ability to specify length unit in ST_Distance [1]
```

## Character sets and collations

```
1. UTF-8 (utf8mb4) as default character set [1 2 3 4 5 6 7 8 9 10 11]
2. General Unicode 9.0 collations covering German (dictionary order), Austrian German (dictionary order), English, French (including accent insensitive Canadian French), Irish, Indonesian, Italian, Luxembourgian, Malay, Dutch (including Flemish), Portuguese (including Brazilian Portuguese), Swahili, and Zulu [1 2]
3. Language specific Unicode 9.0 collations for Czech, Danish (also valid for Norwegian), German (phonebook order), Esperanto, Spanish, Spanish (traditional), Estonian, Croatian (also valid for Serbian with latin characters, and Bosnian), Hungarian, Icelandic, Lithuanian, Latvian, Polish, Romanian, Slovak, Slovenian, Swedish, Turkish, Vietnamese, Japanese (including kana sensitive collation), and Russian (also valid for Bulgarian)
4. Unicode support in RLIKE and REGEXP [1]
```

## Information Schema

```
1. Information Schema implemented as views over data dictionary tables [1]
2. New Information Schema view VIEW_TABLE_USAGE [1]
3. New Information Schema view VIEW_ROUTINE_USAGE [1]
4. New Information Schema view KEYWORDS [1]
5. New Information Schema view COLUMN_STATISTICS [1]
6. New Information Schema view ST_GEOMETRY_COLUMNS [1]
7. New Information Schema view ST_SPATIAL_REFERENCE_SYSTEMS [1]
8. New Information Schema view ST_UNITS_OF_MEASURE [1]
```

## Performance Schema

```
1. Performance Schema Indexes [1]
2. Instrument server errors  [1]
3. Statements latency histograms [1]
4. Instrument data locks [1]
5. Pluggable performance schema tables [1]
6. Added QUERY_SAMPLE_TEXT  [1]
7. Added Thread Pool Tables  [1] (Enterprise)
```

## SHOW

```
1. SHOW now lists hidden columns [1]
2. SHOW now lists index information [1]
```

## Optimizer

```
1. Histograms [1 2 3]
2. Adaptive scan buffer size [1]
3. IO costs separation between memory and disk [1]
4. Default values in cost tables [1]
5. Sampling interface in storage engine API [1 2 3 4]
6. NOWAIT and SKIP LOCKED [1 2]
7. Avoid unnecesary index dives with FORCE INDEX [1 2 3]
8. Optimizer switch to use invisible indexes [1 2 3]
9. Increased default optimizer trace buffer size [1]
10. New hint MERGE [1]
11. New hint INDEX_MERGE [1]
12. New hint NO_INDEX_MERGE [1]
13. New hint JOIN_FIXED_ORDER [1]
14. New hint JOIN_ORDER [1]
15. New hint JOIN_PREFIX [1]
16. New hint JOIN_SUFFIX [1]
17. New hint SET_VAR [1]
18. Consider covering prefix indexes for LIKE [1]
19. Transformed statement in EXPLAIN of INSERT/UPDATE/REPLACE/DELETE [1]
```

## InnoDB

```
1. Highly scalable latch free redo log implementation [1 2].
2. Redesign of LOB infrastructure for better performance [1 2 3]
3. State of the art lock scheduler using Contention Aware Transaction Scheduling (CATS) (Contribution from University of Michigan) [1]
4. Infrastructure to do non locking parallel reads (currently used by CHECK TABLE) [1]
5. Instant add column and virtual column [1]
6. Report pages cached in the buffer pool by indexes via the information schema [1]
7. Persistent auto increment [1]
8. Manage UNDO tablespaces using SQL syntax [1]
9. New in-memory temptable storage engine for use by optimiser [1]
10. Support for BLOBs in new temptable engine [1 2]
11. Redo log encryption [1]
12. Undo log encryption [1]
13. General tablespace encryption support [1]
14. Dedicated server mode, automatically configures the buffer pool and redo log size [1 2]
15. Tablespace version support for better upgrade/downgrade experience [1]
16. Self describing tablespaces with Serialized Dictionary Information (SDI) [1]
17. Tools to manage the SDI [ 1]
18. Atomic DDL [1]
19. Remove the buffer pool mutex (Percona contribution) [1]
20. Improved purge [1]
21. Dynamically enable/disable the deadlock detector [1]
22. Redesigned IO layer is now more scalable and efficient also removed .isl files [1]
23. Extended locking semantics to skip locked rows or ignore waits on locked row [1 2]
24. Use the new error logging infra structure [1]
25. System data dictionary is now stored in InnoDB [1]
26. New configuration to generate smaller core files [1]
27. Deprecate Shared tablespaces in partitioned table [1]
28. Reclaim temporary tablespace disk space online [1]
```

## Data Dictionary

```
1. Transactional Data Dictionary [1]
2. Store all meta data in InnoDB, no FRMs, TRG etc [1,2]
3. Store redundant copy of meta data in SDI [1]
4. Atomic and crash safe DDL [1]
5. Automatic upgrade of dictionary tables, and enhanced checks [1]
```

## Network

```
1. Support multiple addresses for the –bind-address command option [1]
2. Add Admin Port [1 2 3 4 5]
3. Remove the major mutex bottlenecks for connect/disconnect performance [1]
```

## Error logging

```
1. Improved error logging in 8.0 [1]
2. Defaults change: log_error_verbosity=2 [1]
3. Added severity, error code, subsystem to error messages [1]
4. Filtering the error log [1]
5. Error logging in JSON format [1]
6. Force-print specific non-error messages to error log [1]
7. Suppress error logs of type warning or note [1]
8. Specify syslog/eventlog related system variables to component variables [1]
9. Added –log-slow-extra, for richer slow logging [1]
```

## Replication

```
1. Multi-source Replication Per Channel Filters [1]
2. Atomic DDL Recovery With The Binary Log [1]
3. Write-set Based Transaction Dependency Tracking [1 2 ]
4. Reduced Contention Between Receiver and Applier Threads [1]
5. Binary Log Encryption at Rest [1]
6. GTID Support for Temporary Tables Inside Transactions [1]
7. Partial JSON Update Replication [1]
8. Extended table metadata in the binary log [1 2]
9. RESET MASTER TO ‘x’ [1]
10. Settable GTID_PURGED When GTID_EXECUTED is Not Empty [1]
11. Sub-second Binary Logs Expiration Settings [1]
12. Non-Blocking Replication Monitoring even when Disk is Full [1]
13. Transaction Byte Length Metadata in Binary log [1]
14. Server Versions for each Transaction in the Binary Log  [1, 2, 3]
15. Support for START SLAVE UNTIL for Multi-Threaded Applier [1]
16. Delayed Replication in Microseconds [1]
17. binlog-row-event-max-size system variable [1]
18. PFS: Applier Lag and Queues Monitoring [1 2 3]
19. PFS: Read Consistent Log Positions for Backups [1]
20. PFS: Row-based Replication Applier Thread Progress [1]
21. PFS: Counters for Replication Applier Retries [1]
```

## Group Replication

```
1. Transaction Savepoint Support [1]
2. Disallow Writes to Isolated Members in Group Replication [1 2 3]
3. Group-wide Certification and Applier Stats Monitoring [1 2]
4. Options to Fine-tune the Flow-Control [1 2 3 4 5 6 7 8]
5. Support for Hostnames in the Whitelist [1]
6. Shutdown Server When Server Drops Out of the Group [1]
7. Online and User-Triggered Primary Switchover/Election [1 2]
8. Online and User-Triggered Single-to-Multi Primary Switchover [1 2]
9. Configurable Messaging Pipelining [1 2]
10. Relaxed Member Eviction [1]
11. Consistent Reads [1]
12. Consistent Reads on Primary Fail-over [1]
13. IPv6 Support [1 2]
14. Tracing for Message Passing [1]
15. Configure primary failover candidates priority [1 2]
16. PFS: Instrumented Threads [1 2]
17. PFS: Instrumented Mutexes and Condvars [1 2]
18. PFS: Instrumented Memory Used for the Message Cache [1 2]
```

## Security

```
1. Make ACL statements atomic  [1]
2. Introduce delays in authentication based on failed logins  (also in 5.7.17) [1]
3. SQL Roles [1]
4. Break the SUPER privilege into dynamic privileges [1]
5. Automatic assignment and granting of default roles when new users are created [1 23]
6. Password rotation policy enforcement [1]
7. Caching sha2 authentication plugin: a SHA256 based plugin fast enough to replace mysql_native  [1]
8. Additional safety to –skip-grant-tables (enables –skip-networking too)   [1]
9. Audit log: abort queries on rule based conditions   [1]
10. Server as a keyring backend migration tool  (also in 5.7.21)  [1]
11. JSON format, compression and encryption for audit log    (5.7.21) [1]
12. Migrate away from yaSSL  [1]
13. Support for FIPs enabled OpenSSL library  [1]
14. Old password required for SET PASSWORD for some users [1]
15. Data masking functions (also in 5.7.24) [1] (Enterprise)
16. SASL authentication for LDAP on windows   (also in 5.7.24) [1]  (Enterprise)
17. Support 2 active passwords per user account  [1]
18. A SQL function to inject data into the Audit log  [1]
19. Secure session variable setting (MYSQL_SESSION_ADMIN privilege) [1]
20. Extra authentication to allow SET PERSIST for security sensitive variables  [1]
21. Add support for users with multiple LDAP groups (also in 5.7.25) [1]
22. Checking authorization for rolling back XA-transactions [1]
23. Ensure foreign key error does not reveal information about parent table [1]
```

## Router

```
1. Persist last known metadata-server addresses [1]
2. Reset max_connect_errors on successful connections [1]
3. Build Router as part of the MySQL Server source-tree [1]
4. Added mysqlrouter_plugin_info tool [1]
5. Reduced metadata-cache TTL from 300s to 500ms [1]
6. Added routing strategies [1]
7. Added bootstrap option for –report-host [1]
8. Added bootstrap option for –account-host [1]
9. Disconnect clients to server-nodes that changed from PRIMARY to SECONDARY [1]
```

## Shell

```
1. MySQL 8.0 support for InnoDB clusters [1]
2. Remote MySQL server configuration and re-configuration for InnoDB clusters [1]
3. Extended cluster status display, including replication lag times [1]
4. Manual primary switch-over and topology re-configuration in InnoDB clusters [1]
5. Advanced cluster customizations for more use-cases and environments
6. MySQL server upgrade checker [1, 2]
7. Import JSON and JSON serialized BSON data  [1, 2]
8. Updated X DevAPI support
9. Secure password management [1, 2]
10. Display column metadata for query results  [1]
11. Direct command line execution of shell APIs [1]
12. Improved built-in help [1]
13. Screen paging [1]
14. Auto-completion [1]
15. Persisted command history [1]
16. Customizable prompts [1]
```

## Misc

```
1. Added mysqld_safe-functionality to server [1 2 3]
2. Defaults change: explicit_defaults_for_timestamp= ON [1]
3. Defaults change: max_error_count=1024 [1]
4. Renamed tx_read_only variable to transaction_read_only [1]
5. Renamed tx_isolation variable to transaction_isolation [1]
6. Defaults change: max_allowed_packet=67108864 [1]
7. Defaults change: event_scheduler=ON [1]
8. Defaults change: back_log=-1 (auto-sized) [1]
9. Defaults change: table_open_cache=4000 [1 2]
10. New Backup Lock [1]
11. Server version stored in InnoDB tablespaces [1]
12. Enable MDL Locking for Recovered and Detached Prepared XA Transactions [1]
13. Support meta data locking for Foreign Keys [1]
14. The –ssl-mode client side option to streamline SSL checking [1]
15. The service registry and the component infrastructure [1]
16. CLI interface to read the replication stream  [1]
17. UDF registration service to allow components to auto-register UDFs  [1]
18. MySQL server strings component service  [1]
19. Keyring plugin for AWS KMS  5.7.19 [1]
20. LDAP authentication plugin (client and server)  5.7.19 [1]
21. Make result set metadata transfer optional  [1]
22. Status variables service for components [1]
23. performance schema instrumentation via a component service [1]
24. System variables service for components  [1]
25. Password validation plugin implemented as a component  [1]
26. Component service to deliver signals to the host application
27. Allow plugins to use prepared statements [1]
28. INSERT/UPDATE/DELETE in query rewrite plugin [1]
29. Dynamic allocation of sort buffer [1]
30. Variable length sort keys for NO PAD collations [1]
31. Faster SELECT COUNT (*) without grouping [1]
32. Source code improvements [1]
```

Thank you for using MySQL !

转自：[MySQL 官方博客](https://mysqlserverteam.com/the-complete-list-of-new-features-in-mysql-8-0/)

------

# MySQL 5.7 新功能列表

There are over 150 new features in MySQL 5.7.

The MySQL manual is very good, but verbose. This is a list of new features in short bullet form. I have tried very hard to make sure each feature is only mentioned once. So InnoDB native partitioning could be mentioned under either InnoDB or partitioning.

> **`MySQL5.7 相比 MySQL5.6 更新 150 多个新功能`**

## Replication

```
1. Multi source replication [1]
2. Online GTID migration path [1 2 3]
3. Improved semi-sync performance [1 2]
4. Loss-less semi-sync replication [1 2]
5. Semi-sync can now wait for a configurable number of slaves [1]
6. Intra-schema parallel replication [1]
7. Ability to tune group commit via binlog_group_commit_sync_delay and binlog_group_commit_sync_no_delay_countoptions. [1 2]
8. Non-blocking SHOW SLAVE STATUS [1 2]
9. Online CHANGE REPLICATION FILTER [1]
10. Online CHANGE MASTER TO without stopping SQL thread [1]
11. Multi-threaded slave ordered commits (Sequential Consistency) [1]
12. Support SLAVE_TRANSACTION_RETRIES in multi-threaded slave mode [1]
13. A WAIT_FOR_EXECUTED_GTID_SET function has been introduced [1 2]
14. Optimize GTIDs for Passive Slaves [1 2]
15. GTID Replication no longer requires log-slave-updates be enabled
16. XA Support when the binary log is turned on [1]
17. GTIDs in the OK packet [1]
18. Better synchronization between dump and user threads when racing for the binlog [1]
19. Improved memory management of Binlog_sender [1]
20. Option to suppress "unsafe for binlog" messages in error log [1]
21. Defaults change: binlog_format=ROW
22. Defaults change: sync_binlog=1
23. Defaults change: binlog_gtid_simple_recovery=1
24. Defaults change: binlog_error_action=ABORT_SERVER
25. Defaults change: slave_net_timeout=60
```

## InnoDB

```
1. Online buffer pool resize [1]
2. Improved crash recovery performance [1]
3. Improved read-only transaction scalability [1 23 4]
4. Improved read-write transaction scalability [1 23 4]
5. Several optimizations for high performance temporary tables [1 2 3 4 5]
6. ALTER TABLE RENAME INDEX only requires meta-data change [1]
7. Increasing VARCHAR size only requires meta-data change [1]
8. ALTER TABLE performance improved [1 2]
9. Multiple page_cleaner threads [1]
10. Optimized buffer pool flushing [1]
11. New innodb_log_checksum_algorithm option [1]
12. Improved NUMA support [1]
13. General Tablespace support [1]
14. Transparent Page Compression [1]
15. innodb_log_write_ahead_size introduced to address potential 'read-on-write' with redo logs [1]
16. Fulltext indexes now support pluggable parsers [1]
17. Support for ngram and MeCab full-text parser plugins [1 2]
18. Fulltext search optimizations [1]
19. Buffer pool dump now supports innodb_buffer_pool_dump_pct [1]
20. The doublewrite buffer is now disabled on filesystems that supports atomic writes (aka Fusion-io support) [1]
21. Page fill factor is now configurable [1]
22. Support for 32K and 64K pages [1]
23. Online undo log truncation [1]
24. Update_time meta data is now updated [1]
25. TRUNCATE TABLE is now atomic [1]
26. Memcached API performance improvements [1]
27. Adaptive hash scalability improvements [1]
28. InnoDB now implements information_schema.files [1]
29. Legacy InnoDB monitor tables have been removed or replaced by global configuration settings
30. InnoDB default row format now configurable [1]
31. InnoDB now drops tables in a background thread [1]
32. InnoDB tmpdir is now configurable [1]
33. InnoDB MERGE_THRESHOLD is now configurable [1]
34. InnoDB page_cleaner threads get priority using setpriority()[1]
35. Defaults change: innodb_file_format=Barracuda
36. Defaults change: innodb_large_prefix=1
37. Defaults change: innodb_page_cleaners=4
38. Defaults change: innodb_purge_threads=4
39. Defaults change: innodb_buffer_pool_dump_at_shutdown=1
40. Defaults change: innodb_buffer_pool_load_at_startup=1
41. Defaults change: innodb_buffer_pool_dump_pct=25
42. Defaults change: innodb_strict_mode=1
43. Defaults change: innodb_checksum_algorithm=crc32
44. Defaults change: innodb_default_row_format=DYNAMIC
```

## Optimizer

```
1. Improved optimizer cost model, leading to more consistently better query plans [1 2 3 4]
2. Optimizer cost constants are now configurable on a global or per engine basis [1 2]
3. Query parser has been refactored and improved [1]
4. EXPLAIN FOR CONNECTION [1]
5. UNION ALL does not use a temporary table [1 23 4]
6. Filesort is now optimized to pack values [1]
7. Subqueries in FROM clause can now be handled same as a view (derived_merge) [1]
8. Queries using row value constructors are now optimized [1 2]
9. Optimizer now supports a new condition filtering optimization [1 2]
10. EXPLAIN FORMAT=JSON now shows cost information [1]
11. Support for STORED and VIRTUAL generated columns (aka functional indexes) [1]
12. Prepared statements refactored internally and performance improved [1 2]
13. New query hints using comment-like /*+ */syntax [1]
14. Server-side query rewrite framework [1]
15. ONLY_FULL_GROUP_BY now more standards compliant [1]
16. Support for gb18030 character set [1]
17. Improvements to Dynamic Range access [1]
18. Memory used by the range optimizer is now configurable [1]
19. Defaults change: internal_tmp_disk_storage_engine=INNODB[1]
20. Defaults change: eq_range_index_dive_limit=200
21. Defaults change: sql_mode=ONLY_FULL_GROUP_BY, STRICT_TRANS_TABLES, NO_ZERO_IN_DATE, NO_ZERO_DATE, ERROR_FOR_DIVISION_BY_ZERO, NO_AUTO_CREATE_USER, NO_ENGINE_SUBSTITUTION
22. Defaults change: optimizer_switch=condition_fanout_filter=on, derived_merge=on
23. Defaults change: EXTENDED and PARTITIONSkeywords for EXPLAIN enabled by default
```

## Security

```
1. Username size increased to 32 characters [1]
2. Support for IF [NOT] EXISTS clause in CREATE/DROP USER [1]
3. Server option to require secure transport [1]
4. Support for multiple AES Encryption modes [12]
5. Support for TLSv1.2 (with OpenSSL) and TLSv1.1 (with YaSSL) [1 2]
6. Support to LOCK/UNLOCK user accounts [1 2]
7. Support for password expiration policy [1 2]
8. Password strength enforcement
9. test database no longer created on installation
10. Anonymous users no longer created on installation
11. Random password generated by default on installation
12. New ALTER USER command
13. SET password='' now accepts a password instead of hash
14. Server now generates SSL keys by default
15. Insecure old_password hash removed [1]
16. Ability to create utility users for stored programs that can not login [1]
17. mysql.user.password field renamed as authentication_string to better describe its current usage.
18. Support for tablespace encryption [1]
```

## Performance Schema

```
1. Scalable memory allocation [1]
2. Overhead has been reduced in client connect/disconnect phases
3. Memory footprint has been reduced
4. pfs_lock implementation has been improved
5. Table IO statistics are now batched for improved performance
6. Memory usage instrumentation
7. Stored programs instrumentation
8. Replication slave instrumentation
9. Metadata Locking (MDL) Instrumentation
10. Transaction instrumentation
11. Prepared Statement instrumentation
12. Stage Progress instrumentation
13. SX-lock and rw_lock instrumentation
14. Thread status and variables
15. Defaults change: performance-schema-consumer-events_statements_history=ON
```

## GIS

```
1. InnoDB supports indexing of spatial datatypes [1]
2. Consistent naming scheme for GIS functions [1]
3. GIS has been refactored internally and is now based on Boost Geometry [1]
4. Geohash functions [1 2]
5. GeoJSON functions [1 2]
6. Functions: ST_Distance_Sphere, ST_MakeEnvelope, ST_IsValid, ST_Validate, ST_Simplify, ST_Buffer and ST_IsSimple [1 2]
```

## Triggers

```
1. Multiple triggers per event per table [1]
2. BEFORE Triggers are not processed for NOT NULL columns [1]
```

## Partitioning

```
1. Index condition pushdown optimization now supported
2. HANDLER command is now supported
3. WITHOUT VALIDATION option now supported for ALTER TABLE ... EXCHANGE PARTITION
4. Support for Transportable Tablespaces
5. Partitioning is now storage-engine native for InnoDB
```

## SYS (new)

```
1. SYS schema bundled by default [1 2]
2. 100 new views, 21 new stored functions and 26 new stored procedures to help understand and interact with Performance Schema and Information Schema [1]
```

## JSON (new)

```
1. Native JSON Data Type [1]
2. JSON Comparator
3. Short-hand JSON_EXTRACT operator (field->"json_path") [1]
4. New Document Store (5.7.12)
5. Functions: JSON_ARRAY, JSON_MERGE, JSON_OBJECT for creating JSON values [1]
6. Functions: JSON_CONTAINS, JSON_CONTAINS_PATH, JSON_EXTRACT, JSON_KEYS, JSON_SEARCH for searching JSON values [1]
7. Functions: JSON_ARRAY_APPEND, JSON_ARRAY_INSERT, JSON_INSERT, JSON_QUOTE, JSON_REMOVE, JSON_REPLACE, JSON_UNSET, JSON_UNQUOTE to modify JSON values [1]
8. Functions: JSON_DEPTH, JSON_LENGTH, JSON_TYPE, JSON_VALID to return JSON value attributes [1]
```

## Client Programs

```
1. New mysqlpump utility [1]
2. The mysql client now supports Ctrl+C to clear statement buffer
3. rewrite-db option added to mysqlbinlog [1 2]
4. New mysql_ssl_rsa_setup utility to help set up SSL [1]
5. SSL support added to mysqlbinlog
6. Idempotent mode added to mysqlbinlog
7. Client --ssl changed to now force SSL
8. Enhancements to the innochecksum utility
9. Removal of several outdated/unsafe command line utilities [1]
10. Many Perl command line clients converted to C++
11. Client Side Protocol Tracing
12. Client API method to reset connection
13. New MySQL Shell (mysqlsh) (separate download)
```

## libmysqlclient

```
1. Restricted export functions to documented MySQL C API
2. Added pkg-config support
3. Removed _r symlinks
4. Changed so version to 20 (from 18)
```

## Building

```
1. Compiler switched to GCC on Solaris
2. MySQL now compiles with Bison 3 (* change also backported)
3. CMake now used to compile on all platforms
4. Unneeded CMake checks have been removed (and unused macros removed from source files)
5. Build support for gcc, clang and MS Studio
```

## Misc

```
1. Server new connection throughput improved considerably [1]
2. mysql_install_db replaced by mysqld --initialize [1]
3. Native support for syslog [1 2]
4. Native support for systemd
5. disabled_storage_engines option allows a block list of engines
6. SET GLOBAL offline_mode=1 [1 2]
7. super_read_only option [1]
8. Detect transaction boundaries
9. Server version token and check [1
10. SELECT GET_LOCK() can now acquire multiple locks [1 2]
11. Configurable maximum statement execution time on a global and per query basis [1 2]
12. Better handling of connection id rollover
13. DTrace support [1]
14. More consistent IGNORE clause and STRICTmode
15. A number of tables in the mysql schema have moved from MyISAM to InnoDB
16. Server error log format improved to be consistent
17. Extract query digest moved from performance_schema into the server directly
18. Improved scalability of meta data locking
19. Increased control over error log verbosity
20. Stacked Diagnostic Areas
21. The server now supports a "SHUTDOWN" command
22. Removed support for custom atomics implementation
23. Removed "unique option prefix support" from server and utilities, which allowed options to be configured using multiple names.
24. Removed unsafe ALTER IGNORE TABLE functionality. Syntax remains for compatibility
25. Removed unsafe INSERT DELAYED functionality. Syntax remains for compatibility
26. Removed of outdated sql-bench scripts in distributions
27. Removal of ambiguous YEAR(2) datatype
28. Defaults change: log_warnings=2
29. Defaults change: table_open_cache_instanc
```

转自：[博客](http://www.thecompletelistoffeatures.com/)

------