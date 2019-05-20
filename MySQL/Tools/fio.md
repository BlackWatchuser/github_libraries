AWS:

``` shell
# /opt/fio-2.1.10/fio -direct=1 -iodepth=8 -rw=randwrite -ioengine=libaio -bs=16k -size=10G -lockmem=8G -numjobs=1 -runtime=300 -group_reporting -time_based -filename=iotest -name=Rand_Write_Testing

Rand_Write_Testing: (g=0): rw=randwrite, bs=16K-16K/16K-16K/16K-16K, ioengine=libaio, iodepth=8
fio-2.1.10
Starting 1 process
Rand_Write_Testing: Laying out IO file(s) (1 file(s) / 10240MB)
Jobs: 1 (f=1): [w] [100.0% done] [0KB/26720KB/0KB /s] [0/1670/0 iops] [eta 00m:00s]
Rand_Write_Testing: (groupid=0, jobs=1): err= 0: pid=9682: Fri Jan 11 11:19:00 2019
  write: io=14339MB, bw=48944KB/s, iops=3058, runt=300003msec
    slat (usec): min=2, max=19133, avg=14.67, stdev=46.47
    clat (usec): min=470, max=72782, avg=2598.68, stdev=674.59
     lat (usec): min=479, max=72792, avg=2613.62, stdev=678.86
    clat percentiles (usec):
     |  1.00th=[ 1576],  5.00th=[ 2448], 10.00th=[ 2480], 20.00th=[ 2512],
     | 30.00th=[ 2544], 40.00th=[ 2576], 50.00th=[ 2608], 60.00th=[ 2608],
     | 70.00th=[ 2640], 80.00th=[ 2672], 90.00th=[ 2704], 95.00th=[ 2768],
     | 99.00th=[ 2864], 99.50th=[ 2992], 99.90th=[ 7840], 99.95th=[11200],
     | 99.99th=[44288]
    bw (KB  /s): min=20064, max=146656, per=100.00%, avg=48963.03, stdev=4265.00
    lat (usec) : 500=0.01%, 750=0.67%, 1000=0.18%
    lat (msec) : 2=0.31%, 4=98.58%, 10=0.19%, 20=0.05%, 50=0.02%
    lat (msec) : 100=0.01%
  cpu          : usr=0.54%, sys=8.88%, ctx=912453, majf=0, minf=9
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=100.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.1%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=0/w=917707/d=0, short=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=8

Run status group 0 (all jobs):
  WRITE: io=14339MB, aggrb=48943KB/s, minb=48943KB/s, maxb=48943KB/s, mint=300003msec, maxt=300003msec

Disk stats (read/write):
  xvdb: ios=19/922010, merge=0/57318, ticks=28/2469604, in_queue=2469620, util=98.95%
```

ALI:

``` shell
# /opt/fio-2.1.10/fio -direct=1 -iodepth=8 -rw=randwrite -ioengine=libaio -bs=16k -size=10G -lockmem=8G -numjobs=1 -runtime=300 -group_reporting -time_based -filename=iotest -name=Rand_Write_Testing

Rand_Write_Testing: (g=0): rw=randwrite, bs=16K-16K/16K-16K/16K-16K, ioengine=libaio, iodepth=8
fio-2.1.10
Starting 1 process
Rand_Write_Testing: Laying out IO file(s) (1 file(s) / 10240MB)
Jobs: 1 (f=1): [w] [100.0% done] [0KB/29136KB/0KB /s] [0/1821/0 iops] [eta 00m:00s]
Rand_Write_Testing: (groupid=0, jobs=1): err= 0: pid=2561: Fri Jan 11 11:20:42 2019
  write: io=18317MB, bw=62517KB/s, iops=3907, runt=300016msec
    slat (usec): min=4, max=112532, avg=13.59, stdev=217.85
    clat (usec): min=242, max=693845, avg=2032.14, stdev=7801.32
     lat (usec): min=472, max=693857, avg=2045.98, stdev=7804.26
    clat percentiles (usec):
     |  1.00th=[  564],  5.00th=[  596], 10.00th=[  620], 20.00th=[  644],
     | 30.00th=[  668], 40.00th=[  692], 50.00th=[  716], 60.00th=[  740],
     | 70.00th=[  780], 80.00th=[  852], 90.00th=[ 1608], 95.00th=[ 7328],
     | 99.00th=[30592], 99.50th=[41728], 99.90th=[84480], 99.95th=[113152],
     | 99.99th=[288768]
    bw (KB  /s): min= 7776, max=102848, per=100.00%, avg=62591.80, stdev=15194.50
    lat (usec) : 250=0.01%, 500=0.01%, 750=63.03%, 1000=22.26%
    lat (msec) : 2=5.89%, 4=2.26%, 10=2.39%, 20=2.09%, 50=1.73%
    lat (msec) : 100=0.28%, 250=0.05%, 500=0.01%, 750=0.01%
  cpu          : usr=1.19%, sys=7.65%, ctx=1008191, majf=0, minf=28
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=100.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.1%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=0/w=1172262/d=0, short=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=8

Run status group 0 (all jobs):
  WRITE: io=18317MB, aggrb=62517KB/s, minb=62517KB/s, maxb=62517KB/s, mint=300016msec, maxt=300016msec

Disk stats (read/write):
  vdb: ios=32/1173841, merge=16/44205, ticks=37/2485449, in_queue=2485283, util=82.77%
```

------

AWS:

``` shell
Device:         rrqm/s   wrqm/s     r/s     w/s    rMB/s    wMB/s avgrq-sz avgqu-sz   await  svctm  %util
xvdb              0.00     1.00    0.00 3061.00     0.00    47.81    31.99    31.94   10.44   0.33 100.00

# /opt/fio-2.1.10/fio -direct=1 -iodepth=32 -rw=randwrite -ioengine=libaio -bs=16k -size=10G -lockmem=8G -numjobs=1 -runtime=300 -group_reporting -time_based -filename=iotest -name=Rand_Write_Testing

Rand_Write_Testing: (g=0): rw=randwrite, bs=16K-16K/16K-16K/16K-16K, ioengine=libaio, iodepth=32
fio-2.1.10
Starting 1 process
Jobs: 1 (f=1): [w] [100.0% done] [0KB/34176KB/0KB /s] [0/2136/0 iops] [eta 00m:00s]
Rand_Write_Testing: (groupid=0, jobs=1): err= 0: pid=11723: Fri Jan 11 11:27:40 2019
  write: io=14390MB, bw=49115KB/s, iops=3069, runt=300011msec
    slat (usec): min=2, max=7033, avg=10.27, stdev=53.03
    clat (usec): min=506, max=57443, avg=10412.28, stdev=699.62
     lat (usec): min=515, max=57453, avg=10422.84, stdev=697.95
    clat percentiles (usec):
     |  1.00th=[10048],  5.00th=[10304], 10.00th=[10304], 20.00th=[10432],
     | 30.00th=[10432], 40.00th=[10432], 50.00th=[10432], 60.00th=[10432],
     | 70.00th=[10432], 80.00th=[10560], 90.00th=[10560], 95.00th=[10560],
     | 99.00th=[10688], 99.50th=[10816], 99.90th=[14784], 99.95th=[16192],
     | 99.99th=[19584]
    bw (KB  /s): min=48411, max=146848, per=100.00%, avg=49135.11, stdev=3999.35
    lat (usec) : 750=0.07%, 1000=0.11%
    lat (msec) : 2=0.19%, 4=0.02%, 10=0.54%, 20=99.06%, 50=0.01%
    lat (msec) : 100=0.01%
  cpu          : usr=0.84%, sys=5.35%, ctx=915945, majf=0, minf=9
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=100.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.1%, 64=0.0%, >=64=0.0%
     issued    : total=r=0/w=920942/d=0, short=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=32

Run status group 0 (all jobs):
  WRITE: io=14390MB, aggrb=49115KB/s, minb=49115KB/s, maxb=49115KB/s, mint=300011msec, maxt=300011msec

Disk stats (read/write):
  xvdb: ios=0/921092, merge=0/92, ticks=0/9587640, in_queue=9587604, util=98.93%
```

ALI:

``` shell
Device:         rrqm/s   wrqm/s     r/s     w/s    rMB/s    wMB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
vdb               0.00     0.00    0.00 9390.00     0.00   146.72    32.00    31.86    3.38    0.00    3.38   0.11 100.00

# /opt/fio-2.1.10/fio -direct=1 -iodepth=32 -rw=randwrite -ioengine=libaio -bs=16k -size=10G -lockmem=8G -numjobs=1 -runtime=300 -group_reporting -time_based -filename=iotest -name=Rand_Write_Testing

Rand_Write_Testing: (g=0): rw=randwrite, bs=16K-16K/16K-16K/16K-16K, ioengine=libaio, iodepth=32
fio-2.1.10
Starting 1 process
Jobs: 1 (f=1): [w] [100.0% done] [0KB/115.3MB/0KB /s] [0/7374/0 iops] [eta 00m:00s]
Rand_Write_Testing: (groupid=0, jobs=1): err= 0: pid=4312: Fri Jan 11 11:27:54 2019
  write: io=54537MB, bw=186132KB/s, iops=11633, runt=300035msec
    slat (usec): min=3, max=49425, avg= 9.31, stdev=49.66
    clat (usec): min=482, max=937325, avg=2739.94, stdev=8342.56
     lat (usec): min=492, max=937334, avg=2749.50, stdev=8342.93
    clat percentiles (usec):
     |  1.00th=[  620],  5.00th=[  684], 10.00th=[  716], 20.00th=[  780],
     | 30.00th=[  836], 40.00th=[  892], 50.00th=[  964], 60.00th=[ 1064],
     | 70.00th=[ 1224], 80.00th=[ 1528], 90.00th=[ 3568], 95.00th=[11840],
     | 99.00th=[36096], 99.50th=[48896], 99.90th=[93696], 99.95th=[118272],
     | 99.99th=[205824]
    bw (KB  /s): min=30336, max=249408, per=100.00%, avg=186391.19, stdev=30645.22
    lat (usec) : 500=0.01%, 750=15.00%, 1000=38.79%
    lat (msec) : 2=31.98%, 4=4.72%, 10=3.92%, 20=2.72%, 50=2.40%
    lat (msec) : 100=0.39%, 250=0.07%, 500=0.01%, 750=0.01%, 1000=0.01%
  cpu          : usr=3.21%, sys=13.45%, ctx=2311830, majf=0, minf=29
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=100.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.1%, 64=0.0%, >=64=0.0%
     issued    : total=r=0/w=3490374/d=0, short=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=32

Run status group 0 (all jobs):
  WRITE: io=54537MB, aggrb=186131KB/s, minb=186131KB/s, maxb=186131KB/s, mint=300035msec, maxt=300035msec

Disk stats (read/write):
  vdb: ios=1520/3490604, merge=0/61, ticks=1374/9519654, in_queue=9520345, util=98.37%
```

#################################################################

AWS:

``` shell
Device:         rrqm/s   wrqm/s     r/s     w/s    rMB/s    wMB/s avgrq-sz avgqu-sz   await  svctm  %util
xvdb              0.00     2.00    0.00 3062.00     0.00    47.88    32.02   150.66   49.18   0.33 100.00

# /opt/fio-2.1.10/fio -direct=1 -iodepth=64 -rw=randwrite -ioengine=libaio -bs=16k -size=10G -lockmem=8G -numjobs=8 -runtime=300 -group_reporting -time_based -filename=iotest -name=Rand_Write_Testing

Rand_Write_Testing: (g=0): rw=randwrite, bs=16K-16K/16K-16K/16K-16K, ioengine=libaio, iodepth=64
...
fio-2.1.10
Starting 8 processes
Rand_Write_Testing: Laying out IO file(s) (1 file(s) / 10240MB)
fio: pid=15107, got signal=9.0% done] [0KB/0KB/0KB /s] [0/0/0 iops] [eta 1158050367d:12h:43m:30s]
fio: pid=15110, got signal=9.0% done] [0KB/997KB/0KB /s] [0/62/0 iops] [eta 22h:45m:19s]
Jobs: 6 (f=6): [wwKwwKww] [23.6% done] [0KB/0KB/0KB /s] [0/0/0 iops] [eta 16m:57s]
Rand_Write_Testing: (groupid=0, jobs=8): err= 0: pid=15105: Fri Jan 11 11:41:06 2019
  write: io=14320MB, bw=48869KB/s, iops=3054, runt=300061msec
    slat (usec): min=2, max=244618, avg=1949.18, stdev=6972.78
    clat (usec): min=612, max=540159, avg=123754.48, stdev=47598.55
     lat (usec): min=684, max=540195, avg=125703.96, stdev=47979.19
    clat percentiles (msec):
     |  1.00th=[   46],  5.00th=[   58], 10.00th=[   66], 20.00th=[   83],
     | 30.00th=[   94], 40.00th=[  108], 50.00th=[  119], 60.00th=[  131],
     | 70.00th=[  145], 80.00th=[  163], 90.00th=[  188], 95.00th=[  208],
     | 99.00th=[  258], 99.50th=[  277], 99.90th=[  326], 99.95th=[  347],
     | 99.99th=[  412]
    bw (KB  /s): min= 3288, max=30412, per=16.68%, avg=8153.44, stdev=1657.77
    lat (usec) : 750=0.01%, 1000=0.01%
    lat (msec) : 2=0.01%, 4=0.01%, 10=0.06%, 20=0.13%, 50=1.58%
    lat (msec) : 100=33.18%, 250=63.80%, 500=1.24%, 750=0.01%
  cpu          : usr=0.09%, sys=1.30%, ctx=640449, majf=0, minf=56
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.1%, >=64=100.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.1%, >=64=0.0%
     issued    : total=r=0/w=916485/d=0, short=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=64

Run status group 0 (all jobs):
  WRITE: io=14320MB, aggrb=48869KB/s, minb=48869KB/s, maxb=48869KB/s, mint=300061msec, maxt=300061msec

Disk stats (read/write):
  xvdb: ios=55/923271, merge=27/90781, ticks=1188/44470760, in_queue=44471788, util=95.79%
```

ALI:

``` shell
Device:         rrqm/s   wrqm/s     r/s     w/s    rMB/s    wMB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
vdb               0.00     0.00    0.00 2723.00     0.00    42.55    32.00   149.03   54.61    0.00   54.61   0.37 100.00

# /opt/fio-2.1.10/fio -direct=1 -iodepth=64 -rw=randwrite -ioengine=libaio -bs=16k -size=10G -lockmem=8G -numjobs=8 -runtime=300 -group_reporting -time_based -filename=iotest -name=Rand_Write_Testing

Rand_Write_Testing: (g=0): rw=randwrite, bs=16K-16K/16K-16K/16K-16K, ioengine=libaio, iodepth=64
...
fio-2.1.10
Starting 8 processes
Rand_Write_Testing: Laying out IO file(s) (1 file(s) / 10240MB)
fio: pid=7312, got signal=90.0% done] [0KB/0KB/0KB /s] [0/0/0 iops] [eta 1158050195d:10h:32m:47s]
fio: pid=7306, got signal=91.3% done] [0KB/0KB/0KB /s] [0/0/0 iops] [eta 04m:59s]
fio: pid=7311, got signal=90.0% done] [0KB/0KB/0KB /s] [0/0/0 iops] [eta 245d:20h:32m:27s]
fio: pid=7305, got signal=90.0% done] [0KB/0KB/0KB /s] [0/0/0 iops] [eta 491d:16h:59m:58s]
fio: pid=7310, got signal=90.0% done] [0KB/878KB/0KB /s] [0/54/0 iops] [eta 737d:13h:27m:27s]
Jobs: 3 (f=3): [KKwwwKKK] [30.9% done] [0KB/34272KB/0KB /s] [0/2142/0 iops] [eta 11m:28s]
Rand_Write_Testing: (groupid=0, jobs=8): err= 0: pid=7305: Fri Jan 11 11:41:03 2019
  write: io=9385.2MB, bw=32029KB/s, iops=2003, runt=300052msec
    slat (usec): min=4, max=410035, avg=1010.88, stdev=6560.74
    clat (usec): min=569, max=734698, avg=94846.23, stdev=63351.44
     lat (usec): min=618, max=734722, avg=95857.54, stdev=63699.65
    clat percentiles (msec):
     |  1.00th=[    6],  5.00th=[   15], 10.00th=[   26], 20.00th=[   42],
     | 30.00th=[   56], 40.00th=[   71], 50.00th=[   84], 60.00th=[   99],
     | 70.00th=[  117], 80.00th=[  141], 90.00th=[  178], 95.00th=[  212],
     | 99.00th=[  302], 99.50th=[  338], 99.90th=[  433], 99.95th=[  457],
     | 99.99th=[  545]
    bw (KB  /s): min= 2434, max=23552, per=33.48%, avg=10721.95, stdev=3129.70
    lat (usec) : 750=0.01%, 1000=0.02%
    lat (msec) : 2=0.16%, 4=0.52%, 10=2.62%, 20=3.96%, 50=18.47%
    lat (msec) : 100=34.99%, 250=36.80%, 500=2.42%, 750=0.02%
  cpu          : usr=0.27%, sys=1.22%, ctx=179221, majf=0, minf=89
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.1%, >=64=100.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.1%, >=64=0.0%
     issued    : total=r=0/w=601111/d=0, short=r=0/w=0/d=0

Run status group 0 (all jobs):
  WRITE: io=9385.2MB, aggrb=32029KB/s, minb=32029KB/s, maxb=32029KB/s, mint=300052msec, maxt=300052msec

Disk stats (read/write):
  vdb: ios=256/603216, merge=3/80651, ticks=17579/44203397, in_queue=44220570, util=97.61%
fio: file hash not empty on exit
```

------

AWS:

``` shell
Device:         rrqm/s   wrqm/s     r/s     w/s    rMB/s    wMB/s avgrq-sz avgqu-sz   await  svctm  %util
xvdb              0.00     0.00    0.00 3064.00     0.00    47.88    32.00   154.37   50.46   0.33 100.00

# /opt/fio-2.1.10/fio -direct=1 -iodepth=64 -rw=randwrite -ioengine=libaio -bs=16k -size=10G -lockmem=8G -numjobs=4 -runtime=300 -group_reporting -time_based -filename=iotest -name=Rand_Write_Testing

Rand_Write_Testing: (g=0): rw=randwrite, bs=16K-16K/16K-16K/16K-16K, ioengine=libaio, iodepth=64
...
fio-2.1.10
Starting 4 processes
Jobs: 4 (f=4): [wwww] [34.9% done] [0KB/0KB/0KB /s] [0/0/0 iops] [eta 09m:29s]
Rand_Write_Testing: (groupid=0, jobs=4): err= 0: pid=19268: Fri Jan 11 11:58:51 2019
  write: io=14269MB, bw=48693KB/s, iops=3043, runt=300063msec
    slat (usec): min=2, max=151511, avg=1305.92, stdev=5270.73
    clat (usec): min=732, max=329906, avg=82807.26, stdev=22223.25
     lat (usec): min=775, max=329933, avg=84113.46, stdev=22363.24
    clat percentiles (msec):
     |  1.00th=[   45],  5.00th=[   51], 10.00th=[   56], 20.00th=[   69],
     | 30.00th=[   75], 40.00th=[   77], 50.00th=[   79], 60.00th=[   83],
     | 70.00th=[   93], 80.00th=[  100], 90.00th=[  108], 95.00th=[  121],
     | 99.00th=[  147], 99.50th=[  165], 99.90th=[  229], 99.95th=[  245],
     | 99.99th=[  277]
    bw (KB  /s): min= 6286, max=40701, per=25.02%, avg=12184.06, stdev=1537.02
    lat (usec) : 750=0.01%, 1000=0.01%
    lat (msec) : 2=0.02%, 4=0.02%, 10=0.30%, 20=0.11%, 50=3.33%
    lat (msec) : 100=76.08%, 250=20.10%, 500=0.04%
  cpu          : usr=0.12%, sys=1.32%, ctx=352660, majf=0, minf=32
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.1%, >=64=100.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.1%, >=64=0.0%
     issued    : total=r=0/w=913193/d=0, short=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=64

Run status group 0 (all jobs):
  WRITE: io=14269MB, aggrb=48693KB/s, minb=48693KB/s, maxb=48693KB/s, mint=300063msec, maxt=300063msec

Disk stats (read/write):
  xvdb: ios=0/923185, merge=0/57537, ticks=0/46289644, in_queue=46289508, util=98.30%
```

ALI:

``` shell
Device:         rrqm/s   wrqm/s     r/s     w/s    rMB/s    wMB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
vdb               0.00     0.00    0.00 1914.00     0.00    29.91    32.00   147.23   79.83    0.00   79.83   0.52 100.00

# /opt/fio-2.1.10/fio -direct=1 -iodepth=64 -rw=randwrite -ioengine=libaio -bs=16k -size=10G -lockmem=8G -numjobs=4 -runtime=300 -group_reporting -time_based -filename=iotest -name=Rand_Write_Testing

Rand_Write_Testing: (g=0): rw=randwrite, bs=16K-16K/16K-16K/16K-16K, ioengine=libaio, iodepth=64
...
fio-2.1.10
Starting 4 processes
Rand_Write_Testing: Laying out IO file(s) (1 file(s) / 10240MB)
fio: pid=11161, got signal=9done] [0KB/0KB/0KB /s] [0/0/0 iops] [eta 1158050195d:10h:14m:54s]
Jobs: 3 (f=3): [wKww] [24.9% done] [0KB/31856KB/0KB /s] [0/1991/0 iops] [eta 15m:20s]
Rand_Write_Testing: (groupid=0, jobs=4): err= 0: pid=11160: Fri Jan 11 11:58:51 2019
  write: io=7598.5MB, bw=25917KB/s, iops=1619, runt=300218msec
    slat (usec): min=4, max=2761.1K, avg=1258.62, stdev=11851.61
    clat (usec): min=639, max=2953.2K, avg=117259.12, stdev=132841.25
     lat (usec): min=695, max=2968.8K, avg=118518.07, stdev=133513.79
    clat percentiles (msec):
     |  1.00th=[    5],  5.00th=[   15], 10.00th=[   27], 20.00th=[   44],
     | 30.00th=[   60], 40.00th=[   75], 50.00th=[   89], 60.00th=[  105],
     | 70.00th=[  126], 80.00th=[  155], 90.00th=[  215], 95.00th=[  302],
     | 99.00th=[  660], 99.50th=[  906], 99.90th=[ 1287], 99.95th=[ 1434],
     | 99.99th=[ 2900]
    bw (KB  /s): min=   25, max=22176, per=34.73%, avg=9002.18, stdev=3812.94
    lat (usec) : 750=0.01%, 1000=0.02%
    lat (msec) : 2=0.15%, 4=0.63%, 10=2.71%, 20=3.70%, 50=16.96%
    lat (msec) : 100=32.58%, 250=35.96%, 500=5.40%, 750=1.18%, 1000=0.33%
    lat (msec) : 2000=0.35%, >=2000=0.04%
  cpu          : usr=0.22%, sys=1.06%, ctx=158783, majf=0, minf=87
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.1%, >=64=100.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.1%, >=64=0.0%
     issued    : total=r=0/w=486303/d=0, short=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=64

Run status group 0 (all jobs):
  WRITE: io=7598.5MB, aggrb=25917KB/s, minb=25917KB/s, maxb=25917KB/s, mint=300218msec, maxt=300218msec

Disk stats (read/write):
  vdb: ios=13/489286, merge=2/74729, ticks=448/44246916, in_queue=44247025, util=98.67%
```