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




## 

``` shell
# /opt/sysbench/src/sysbench --help
Usage:
  sysbench [options]... [testname] [command]

Commands implemented by most tests: prepare run cleanup help

General options:
  --threads=N                     number of threads to use [1]
  --events=N                      limit for total number of events [0]
  --time=N                        limit for total execution time in seconds [10]
  --warmup-time=N                 execute events for this many seconds with statistics disabled before the actual benchmark run with statistics enabled [0]
  --forced-shutdown=STRING        number of seconds to wait after the --time limit before forcing shutdown, or 'off' to disable [off]
  --thread-stack-size=SIZE        size of stack per thread [64K]
  --thread-init-timeout=N         wait time in seconds for worker threads to initialize [30]
  --rate=N                        average transactions rate. 0 for unlimited rate [0]
  --report-interval=N             periodically report intermediate statistics with a specified interval in seconds. 0 disables intermediate reports [0]
  --report-checkpoints=[LIST,...] dump full statistics and reset all counters at specified points in time. The argument is a list of comma-separated values representing the amount of time in seconds elapsed from start of test when report checkpoint(s) must be performed. Report checkpoints are off by default. []
  --debug[=on|off]                print more debugging info [off]
  --validate[=on|off]             perform validation checks where possible [off]
  --help[=on|off]                 print help and exit [off]
  --version[=on|off]              print version and exit [off]
  --config-file=FILENAME          File containing command line options
  --luajit-cmd=STRING             perform LuaJIT control command. This option is equivalent to 'luajit -j'. See LuaJIT documentation for more information

Pseudo-Random Numbers Generator options:
  --rand-type=STRING   random numbers distribution {uniform, gaussian, special, pareto, zipfian} to use by default [special]
  --rand-seed=N        seed for random number generator. When 0, the current time is used as an RNG seed. [0]
  --rand-spec-iter=N   number of iterations for the special distribution [12]
  --rand-spec-pct=N    percentage of the entire range where 'special' values will fall in the special distribution [1]
  --rand-spec-res=N    percentage of 'special' values to use for the special distribution [75]
  --rand-pareto-h=N    shape parameter for the Pareto distribution [0.2]
  --rand-zipfian-exp=N shape parameter (exponent, theta) for the Zipfian distribution [0.8]

Log options:
  --verbosity=N verbosity level {5 - debug, 0 - only critical messages} [3]

  --percentile=N       percentile to calculate in latency statistics (1-100). Use the special value of 0 to disable percentile calculations [95]
  --histogram[=on|off] print latency histogram in report [off]

General database options:

  --db-driver=STRING  specifies database driver to use ('help' to get list of available drivers) [mysql]
  --db-ps-mode=STRING prepared statements usage mode {auto, disable} [auto]
  --db-debug[=on|off] print database-specific debug information [off]


Compiled-in database drivers:
  mysql - MySQL driver

mysql options:
  --mysql-host=[LIST,...]          MySQL server host [localhost]
  --mysql-port=[LIST,...]          MySQL server port [3306]
  --mysql-socket=[LIST,...]        MySQL socket
  --mysql-user=STRING              MySQL user [sbtest]
  --mysql-password=STRING          MySQL password []
  --mysql-db=STRING                MySQL database name [sbtest]
  --mysql-ssl=STRING               SSL mode. This accepts the same values as the --ssl-mode option in the MySQL client utilities. Disabled by default [disabled]
  --mysql-ssl-key=STRING           path name of the client private key file
  --mysql-ssl-ca=STRING            path name of the CA file
  --mysql-ssl-cert=STRING          path name of the client public key certificate file
  --mysql-ssl-cipher=STRING        use specific cipher for SSL connections []
  --mysql-compression[=on|off]     use compression, if available in the client library [off]
  --mysql-debug[=on|off]           trace all client library calls [off]
  --mysql-ignore-errors=[LIST,...] list of errors to ignore, or "all" [1213,1020,1205]
  --mysql-dry-run[=on|off]         Dry run, pretend that all MySQL client API calls are successful without executing them [off]

Compiled-in tests:
  fileio - File I/O test
  cpu - CPU performance test
  memory - Memory functions speed test
  threads - Threads subsystem performance test
  mutex - Mutex performance test

See 'sysbench <testname> help' for a list of options for each test.
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

如果多台服务器进行 CPU 性能对比，当线程和素数个数一定时：相同时间，比较 event；相同 event，比较时间；时间和 event 都相同，比较 stddev (标准差)。

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

