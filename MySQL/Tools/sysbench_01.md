# SYSBENCH

[GitHub](https://github.com/akopytov/sysbench) | [Benchmarks](https://dev.mysql.com/downloads/benchmarks.html)

## Building and Installing From Source

``` shell
# git clone https://github.com/akopytov/sysbench

# yum -y install make automake libtool pkgconfig libaio-devel

# yum install mysql57-devel.x86_64 openssl-devel

# cd sysbench/ && ./autogen.sh

# ./configure

# make -j && make install

# sysbench --version
sysbench 1.1.0-faaff4f

# which sysbench
/usr/local/bin/sysbench
```

## Usage

``` shell

```

``` shell
# sysbench cpu help
sysbench 1.1.0-faaff4f (using bundled LuaJIT 2.1.0-beta3)

cpu options:
  --cpu-max-prime=N upper limit for primes generator [10000]

# sysbench memory help
sysbench 1.1.0-faaff4f (using bundled LuaJIT 2.1.0-beta3)

memory options:
  --memory-block-size=SIZE    size of memory block for test [1K]
  --memory-total-size=SIZE    total size of data to transfer [100G]
  --memory-scope=STRING       memory access scope {global,local} [global]
  --memory-hugetlb[=on|off]   allocate memory from HugeTLB pool [off]
  --memory-oper=STRING        type of memory operations {read, write, none} [write]
  --memory-access-mode=STRING memory access mode {seq,rnd} [seq]

# sysbench fileio help
sysbench 1.1.0-faaff4f (using bundled LuaJIT 2.1.0-beta3)

fileio options:
  --file-num=N                  number of files to create [128]
  --file-block-size=N           block size to use in all IO operations [16384]
  --file-total-size=SIZE        total size of files to create [2G]
  --file-test-mode=STRING       test mode {seqwr, seqrewr, seqrd, rndrd, rndwr, rndrw}
  --file-io-mode=STRING         file operations mode {sync,async,mmap} [sync]
  --file-async-backlog=N        number of asynchronous operatons to queue per thread [128]
  --file-extra-flags=[LIST,...] list of additional flags to use to open files {sync,dsync,direct} []
  --file-fsync-freq=N           do fsync() after this number of requests (0 - don't use fsync()) [100]
  --file-fsync-all[=on|off]     do fsync() after each write operation [off]
  --file-fsync-end[=on|off]     do fsync() at the end of test [on]
  --file-fsync-mode=STRING      which method to use for synchronization {fsync, fdatasync} [fsync]
  --file-merged-requests=N      merge at most this number of IO requests if possible (0 - don't merge) [0]
  --file-rw-ratio=N             reads/writes ratio for combined test [1.5]
```

------

# sysbench - 测试 CPU 性能

## 前言

sysbench 是一个多线程的 Linux 测试工具，可以进行 CPU 性能测试。**`对 CPU 的测试，主要是进行素数的加法运行`**。

## 常用参数

**`--cpu-max-prime`**：素数生成数量的上限

如果设置为3，则表示2、3、5（要计算 1-5 共5次）；如果设置为10，则表示2、3、5、7、11、13、17、19、23、29（要计算 1-29 共29次）；**默认值为：10000**。

**`--threads`**：线程数

如果设置为1，则 sysbench 仅启动1个线程进行素数的计算；如果设置为2，则 sysbench 会启动2个线程，同时分别进行素数的计算；**默认值为 1**，即单线程。

**`--time`**：运行时长，单位秒

如果设置为 5，则 sysbench 会在5秒内循环往复进行素数计算，从输出结果可以看到在5秒内完成了几次；

比如配合 `--cpu-max-prime=3`，则表示第一轮算得3个素数， 如果时间还有剩就再进行一轮素数计算，直到时间耗尽；默认值为10。

相同时间，比较的是谁完成的 event 多。

**`--events`**：event 上限次数

每完成一轮就叫一个 event ，设置为100，则表示当完成100次 event 后，即使时间还有剩，也停止运行；默认值为0，则表示不限 event 次数；

相同 event 次数，比较的是谁用时更少。

## 例子

``` shell
# sysbench cpu --cpu-max-prime=20000 --threads=2 --time=30 run

sysbench 1.0.15 (using system LuaJIT 2.0.5)
Running the test with following options:
Number of threads: 2 # 线程个数
Initializing random number generator from current time
Prime numbers limit: 20000 # 素数上线
Initializing worker threads...

Threads started!


CPU speed:

    events per second: 633.14 # 所有线程平均每秒完成 event 的个数
    General statistics:
    total time: 30.0024s # 总共消耗的时间
    total number of events: 18997 # 所有线程完成的 event 个数

Latency (ms):

    min: 3.11 # 完成1次 event 的最少耗时
    avg: 3.16 # 所有 event 的平均耗时
    max: 8.80 # 完成1次 event 的最多耗时

    95th percentile: 3.25 # 95% 次 event 完成的时间
    sum: 59995.90 # 所有线程的时间综合

Threads fairness:

    events (avg/stddev): 9498.5000/5.50 # 平均每个线程完成 envet 的次数，后一个值是标准差

    execution time (avg/stddev): 29.9980/0.00 # 平均每个线程的平均耗时，后一个值是标准差
```

**`stddev(标准差)：在相同时间内，多个线程分别完成的素数计算次数是否稳定，如果数值越低，则表示多个线程的结果越接近(即越稳定)。该参数对于单线程无意义`**。

## 总结

如果多台服务器进行 CPU 性能对比，**`当线程和素数个数一定时：相同时间，比较 event；相同 event，比较时间；时间和 event 都相同，比较 stddev (标准差)`**。

------

# R4.2XLARGE 与 R5.2XLARGE 性能对比

``` shell
# cat /proc/meminfo
```
| KEY | R4_VALUE | R5_VALUE |
| :- | -: | -: | 
| MemTotal | 62882948 kB | 65161240 kB |
| MemFree | 43305008 kB | 64475824 kB |
| MemAvailable | 57313640 kB | 64318476 kB |
| Buffers | 340908 kB | 90772 kB |
| Cached | 13664492 kB | 303788 kB |
| SwapCached | 0 kB | 0 kB |
| Active | 5742560 kB | 283184 kB |
| Inactive | 12933652 kB | 146708 kB |
| Active(anon) | 4670828 kB | 35340 kB |
| Inactive(anon) | 52 kB | 48 kB |
| Active(file) | 1071732 kB | 247844 kB |
| Inactive(file) | 12933600 kB | 146660 kB |
| Unevictable | 0 kB | 0 kB |
| Mlocked | 0 kB | 0 kB |
| SwapTotal | 0 kB | 0 kB |
| SwapFree | 0 kB | 0 kB |
| Dirty | 32 kB | 44 kB |
| Writeback | 0 kB | 0 kB |
| AnonPages | 4670792 kB | 35324 kB |
| Mapped | 40560 kB | 28200 kB |
| Shmem | 68 kB | 64 kB |
| Slab | 743008 kB | 91100 kB |
| SReclaimable | 711076 kB | 74268 kB |
| SUnreclaim | 31932 kB | 16832 kB |
| KernelStack | 3920 kB | 2896 kB |
| PageTables | 15300 kB | 3984 kB |
| NFS_Unstable | 0 kB | 0 kB |
| Bounce | 0 kB | 0 kB |
| WritebackTmp | 0 kB | 0 kB |
| CommitLimit | 31441472 kB | 32580620 kB |
| Committed_AS | 51995252 kB | 434976 kB |
| VmallocTotal | 34359738367 kB | 34359738367 kB |
| VmallocUsed | 0 kB | 0 kB |
| VmallocChunk | 0 kB | 0 kB |
| AnonHugePages | 0 kB | 0 kB |
| ShmemHugePages | 0 kB | 0 kB |
| ShmemPmdMapped | 0 kB | 0 kB |
| HugePages_Total | 0 | 0 |
| HugePages_Free | 0 | 0 |
| HugePages_Rsvd | 0 | 0 |
| HugePages_Surp | 0 | 0 |
| Hugepagesize | 2048 kB | 2048 kB |
| DirectMap4k | 12288 kB | 14308 kB |
| DirectMap2M | 1036288 kB | 2299904 kB |
| DirectMap1G | 65011712 kB | 65011712 kB |

``` shell
# cat /proc/cpuinfo

# R4.2XLARGE:

processor	: 7
vendor_id	: GenuineIntel
cpu family	: 6
model		: 79
model name	: Intel(R) Xeon(R) CPU E5-2686 v4 @ 2.30GHz
stepping	: 1
microcode	: 0xb00002a
cpu MHz		: 1497.723
cache size	: 46080 KB
physical id	: 0
siblings	: 8
core id		: 3
cpu cores	: 4
apicid		: 7
initial apicid	: 7
fpu		: yes
fpu_exception	: yes
cpuid level	: 13
wp		: yes
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht syscall nx pdpe1gb rdtscp lm constant_tsc rep_good nopl xtopology aperfmperf eagerfpu pni pclmulqdq ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch fsgsbase bmi1 hle avx2 smep bmi2 erms invpcid rtm rdseed adx xsaveopt
bugs		:
bogomips	: 4661.96
clflush size	: 64
cache_alignment	: 64
address sizes	: 46 bits physical, 48 bits virtual
power management:

# R5.2XLARGE:

processor	: 7
vendor_id	: GenuineIntel
cpu family	: 6
model		: 85
model name	: Intel(R) Xeon(R) Platinum 8175M CPU @ 2.50GHz
stepping	: 4
microcode	: 0x200005e
cpu MHz		: 2500.000
cache size	: 33792 KB
physical id	: 0
siblings	: 8
core id		: 3
cpu cores	: 4
apicid		: 7
initial apicid	: 7
fpu		: yes
fpu_exception	: yes
cpuid level	: 13
wp		: yes
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss ht syscall nx pdpe1gb rdtscp lm constant_tsc rep_good nopl xtopology nonstop_tsc aperfmperf eagerfpu pni pclmulqdq ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch fsgsbase tsc_adjust bmi1 hle avx2 smep bmi2 erms invpcid rtm mpx avx512f avx512dq rdseed adx smap clflushopt clwb avx512cd avx512bw avx512vl xsaveopt xsavec xgetbv1 xsaves ida arat pku ospke
bugs		:
bogomips	: 5000.00
clflush size	: 64
cache_alignment	: 64
address sizes	: 46 bits physical, 48 bits virtual
power management:
```

------

## CPU

| KEY | R4_VALUE | R5_VALUE |
| :- | -: | -: |
| events per second | 2172.20 | 2920.17 |
| Latency (95%, ms) | 3.75 | 2.81 |
| events ( avg / stddev ) | 16292.3750 / 24.74 | 21902.1250 / 18.38 |

``` shell
# sysbench cpu --cpu-max-prime=20000 --threads=8 --time=60 run

# R4.2XLARGE

sysbench 1.1.0-faaff4f (using bundled LuaJIT 2.1.0-beta3)

Running the test with following options:
Number of threads: 8
Initializing random number generator from current time


Prime numbers limit: 20000

Initializing worker threads...

Threads started!

CPU speed:
    events per second:  2172.20

Throughput:
    events/s (eps):                      2172.2014
    time elapsed:                        60.0032s
    total number of events:              130339

Latency (ms):
         min:                                    3.24
         avg:                                    3.68
         max:                                   13.02
         95th percentile:                        3.75
         sum:                               479926.71

Threads fairness:
    events (avg/stddev):           16292.3750/24.74
    execution time (avg/stddev):   59.9908/0.00

# R5.2XLARGE

sysbench 1.1.0-faaff4f (using bundled LuaJIT 2.1.0-beta3)

Running the test with following options:
Number of threads: 8
Initializing random number generator from current time


Prime numbers limit: 20000

Initializing worker threads...

Threads started!

CPU speed:
    events per second:  2920.17

Throughput:
    events/s (eps):                      2920.1721
    time elapsed:                        60.0023s
    total number of events:              175217

Latency (ms):
         min:                                    2.34
         avg:                                    2.74
         max:                                   18.78
         95th percentile:                        2.81
         sum:                               479946.04

Threads fairness:
    events (avg/stddev):           21902.1250/18.38
    execution time (avg/stddev):   59.9933/0.00
```

## Mem

| KEY | R4_VALUE | R5_VALUE |
| :- | -: | -: |
| OPS | 857068.80 per second | 1229058.21 per second |
| 带宽 | 13391.70 MiB/sec | 19204.03 MiB/sec |

``` shell
# sysbench memory --memory-block-size=16k --memory-total-size=12G run

# R4.2XLARGE

sysbench 1.1.0-faaff4f (using bundled LuaJIT 2.1.0-beta3)

Running the test with following options:
Number of threads: 1
Initializing random number generator from current time


Running memory speed test with the following options:
  block size: 16KiB
  total size: 12288MiB
  operation: write
  scope: global

Initializing worker threads...

Threads started!

Total operations: 786432 (857068.80 per second)

12288.00 MiB transferred (13391.70 MiB/sec)


Throughput:
    events/s (eps):                      857068.7964
    time elapsed:                        0.9176s
    total number of events:              786432

Latency (ms):
         min:                                    0.00
         avg:                                    0.00
         max:                                    0.06
         95th percentile:                        0.00
         sum:                                  688.94

Threads fairness:
    events (avg/stddev):           786432.0000/0.00
    execution time (avg/stddev):   0.6889/0.00

# R5.2XLARGE

sysbench 1.1.0-faaff4f (using bundled LuaJIT 2.1.0-beta3)

Running the test with following options:
Number of threads: 1
Initializing random number generator from current time


Running memory speed test with the following options:
  block size: 16KiB
  total size: 12288MiB
  operation: write
  scope: global

Initializing worker threads...

Threads started!

Total operations: 786432 (1229058.21 per second)

12288.00 MiB transferred (19204.03 MiB/sec)


Throughput:
    events/s (eps):                      1229058.2136
    time elapsed:                        0.6399s
    total number of events:              786432

Latency (ms):
         min:                                    0.00
         avg:                                    0.00
         max:                                    0.08
         95th percentile:                        0.00
         sum:                                  537.52

Threads fairness:
    events (avg/stddev):           786432.0000/0.00
    execution time (avg/stddev):   0.5375/0.00

```

## IOPS

``` shell
# sysbench fileio --file-total-size=50G --file-test-mode=rndrw --file-io-mode=async prepare / run / cleanup

# R4.2XLARGE + 500 GB gp2

Throughput:
         read:  IOPS=2400.26 37.50 MiB/s (39.33 MB/s)
         write: IOPS=1608.15 25.13 MiB/s (26.35 MB/s)
         fsync: IOPS=3540.56

Latency (ms):
         min:                                  0.00
         avg:                                  0.13
         max:                                 14.66
         95th percentile:                      0.72
         sum:                               9950.20

# R5.2XLARGE + 500 GB gp2

Throughput:
         read:  IOPS=2768.13 43.25 MiB/s (45.35 MB/s)
         write: IOPS=1846.08 28.85 MiB/s (30.25 MB/s)
         fsync: IOPS=4059.71

Latency (ms):
         min:                                  0.00
         avg:                                  0.11
         max:                                 23.48
         95th percentile:                      0.63
         sum:                               9963.82

```

------

# sysbench 的安装和做性能测试

[原文](http://imysql.cn/node/312)

sysbench 是一个模块化的、跨平台、多线程基准测试工具，主要用于评估测试各种不同系统参数下的数据库负载情况。关于这个项目的详细介绍请看：http://sysbench.sourceforge.net。
它主要包括以下几种方式的测试：

1. cpu性能
2. 磁盘io性能
3. 调度程序性能
4. 内存分配及传输速度
5. POSIX线程性能
6. 数据库性能(OLTP基准测试)

目前sysbench主要支持 MySQL, PgSQL, Oracle 这3种数据库。

## 一、安装

首先，在 http://sourceforge.net/projects/sysbench 下载源码包。
接下来，按照以下步骤安装：

``` shell
tar zxf sysbench-0.4.8.tar.gz
cd sysbench-0.4.8
./configure && make && make install
strip /usr/local/bin/sysbench
```

以上方法适用于 MySQL 安装在标准默认目录下的情况，如果 MySQL 并不是安装在标准目录下的话，那么就需要自己指定 MySQL 的路径了。比如我的 MySQL 喜欢自己安装在 /usr/local/mysql 下，则按照以下方法编译：

``` shell
./configure --with-mysql-includes=/usr/local/mysql/include --with-mysql-libs=/usr/local/mysql/lib && make && make install
```

当然了，**`用上面的参数编译的话，就要确保你的 MySQL lib 目录下有对应的 so 文件，如果没有，可以自己下载 devel 或者 share 包来安装`**。
另外，如果想要让 sysbench 支持 pgsql/oracle 的话，就需要在编译的时候加上参数 **--with-pgsql** 或者
**--with-oracle** 这 2 个参数默认是关闭的，只有 MySQL 是默认支持的。

## 二、开始测试

编译成功之后，就要开始测试各种性能了，测试的方法官网网站上也提到一些，但涉及到 OLTP 测试的部分却不够准确。在这里我大致提一下：

### 1. CPU 性能测试

``` shell
sysbench --test=cpu --cpu-max-prime=20000 run
```

CPU 测试主要是进行素数的加法运算，在上面的例子中，指定了最大的素数为 20000，自己可以根据机器 CPU 的性能来适当调整数值。

### 2. 线程测试

sysbench --test=threads --num-threads=64 --thread-yields=100 --thread-locks=2 run

### 3. 磁盘 IO 性能测试

``` shell
sysbench --test=fileio --num-threads=16 --file-total-size=3G --file-test-mode=rndrw prepare / run / cleanup
```

上述参数指定了最大创建16个线程，创建的文件总大小为 3G，文件读写模式为随机读。

### 4. 内存测试

``` shell
sysbench --test=memory --memory-block-size=8k --memory-total-size=4G run
```

上述参数指定了本次测试整个过程是在内存中传输 4G 的数据量，每个 block 大小为 8K。

### 5、OLTP 测试

``` shell
sysbench --test=oltp --mysql-table-engine=myisam --oltp-table-size=1000000 \
--mysql-socket=/tmp/mysql.sock --mysql-user=test --mysql-host=localhost \
--mysql-password=test prepare / run / cleanup
```

上述参数指定了本次测试的表存储引擎类型为 myisam，这里需要注意的是，官方网站上的参数有一处有误，即 --mysql-table-engine，官方网站上写的是 --mysql-table-type，这个应该是没有及时更新导致的。另外，指定了表最大记录数为 1000000，其他参数就很好理解了，主要是指定登录方式。测试 OLTP 时，可以自己先创建数据库 sbtest，或者自己用参数 --mysql-db 来指定其他数据库。--mysql-table-engine 还可以指定为 innodb 等 MySQL 支持的表存储引擎类型。

------