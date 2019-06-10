在11gR2中DBRM(database resource manager，11gR2中新的后台进程，会在Alert.log告警日志中反映OS操作系统最近5分钟是否有剧烈的swap活动了， 具体的日志如下：



```
WARNING: Heavy swapping observed on system in last 5 mins.
pct of memory swapped in [3.07%] pct of memory swapped out [4.44%].
Please make sure there is no memory pressure and the SGA and PGA
are configured correctly. Look at DBRM trace file for more details.
```

进一步诊断可以观察DBRM后台进程的trace：



```
[oracle@vrh2 trace]$ cat VPROD2_dbrm_5466.trc

Trace file /s01/orabase/diag/rdbms/vprod/VPROD2/trace/VPROD2_dbrm_5466.trc

Oracle Database 11g Enterprise Edition Release 11.2.0.3.0 - 64bit Production

With the Partitioning, Real Application Clusters, Automatic Storage Management, OLAP,

Data Mining and Real Application Testing options

ORACLE_HOME = /s01/orabase/product/11.2.0/dbhome_1

System name:    Linux

Node name:      vrh2.oracle.com

Release:        2.6.32-200.13.1.el5uek

Version:        #1 SMP Wed Jul 27 21:02:33 EDT 2011

Machine:        x86_64

Instance name: VPROD2

Redo thread mounted by this instance: 2

Oracle process number: 7

Unix process pid: 5466, image: oracle@vrh2.oracle.com (DBRM)


*** 2011-12-29 22:08:14.627

*** SESSION ID:Heavy swapping observed on system in last 5 mins-last system boot165.1) 2011-12-29 22:08:14.627

*** CLIENT ID:Heavy swapping observed on system in last 5 mins-last planner system) 2011-12-29 22:08:14.627

*** SERVICE NAME:Heavy swapping observed on system in last 5 mins-wife swapping中文) 2011-12-29 22:08:14.627

*** MODULE NAME:Heavy swapping observed on system in last 5 mins-swapping) 2011-12-29 22:08:14.627

*** ACTION NAME:Heavy swapping observed on system in last 5 mins-the day of swapping) 2011-12-29 22:08:14.627


kgsksysstop: blocking mode (2) timestamp: 1325214494612191

kgsksysstop: successful

kgsksysresume: successful


*** 2011-12-29 22:08:43.869

PQQ: Active Services changed

PQQ: Old service table

SvcIdx  SvcId Active ActDop
     5      5      1      0
     6      6      1      0

PQQ: New service table

SvcIdx  SvcId Active ActDop
     1      1      1      0
     2      2      1      0
     5      5      1      0
     6      6      1      0

2012-01-02 01:49:39.805820 : GSIPC:KSXPCB: msg 0x9bc353f0 status 34, type 12, dest 1, rcvr 0


*** 2012-01-02 01:49:54.509

PQQ: Skipping service checks

Trace file /s01/orabase/diag/rdbms/vprod/VPROD2/trace/VPROD2_dbrm_5466.trc

Oracle Database 11g Enterprise Edition Release 11.2.0.3.0 - 64bit Production

With the Partitioning, Real Application Clusters, Automatic Storage Management, OLAP,

Data Mining and Real Application Testing options

ORACLE_HOME = /s01/orabase/product/11.2.0/dbhome_1

System name:    Linux

Node name:      vrh2.oracle.com

Release:        2.6.32-200.13.1.el5uek

Version:        #1 SMP Wed Jul 27 21:02:33 EDT 2011

Machine:        x86_64

Instance name: VPROD2

Redo thread mounted by this instance: 2

Oracle process number: 7

Unix process pid: 5466, image: oracle@vrh2.oracle.com (DBRM)


*** 2012-01-03 03:05:54.518

*** SESSION ID:Heavy swapping observed on system in last 5 mins-last system boot165.1) 2012-01-03 03:05:54.518

*** CLIENT ID:Heavy swapping observed on system in last 5 mins-last planner system) 2012-01-03 03:05:54.518

*** SERVICE NAME:Heavy swapping observed on system in last 5 mins-wife swapping中文) 2012-01-03 03:05:54.518

*** MODULE NAME:Heavy swapping observed on system in last 5 mins-swapping) 2012-01-03 03:05:54.518

*** ACTION NAME:Heavy swapping observed on system in last 5 mins-the day of swapping) 2012-01-03 03:05:54.518


PQQ: Skipping service checks

kgsksysstop: blocking mode (2) timestamp: 1325577954530079
kgsksysstop: successful
kgsksysresume: successful


*** 2012-01-03 03:05:59.270

PQQ: Active Services changed

PQQ: Old service table

SvcIdx  SvcId Active ActDop
     5      5      1      0
     6      6      1      0

PQQ: New service table

SvcIdx  SvcId Active ActDop
     1      1      1      0
     2      2      1      0
     5      5      1      0
     6      6      1      0

PQQ: Checking service limits


*** 2012-01-07 02:06:51.856

PQQ: Skipping service checks

PQQ: Checking service limits


*** 2012-01-08 23:12:11.302

PQQ: Skipping service checks

Heavy swapping observed in last 5 mins:    [pct of total memory][bytes]


*** 2012-01-09 22:39:51.619

total swpin [ 3.07%][124709K], total swpout [ 4.44%][180120K]
vm stats captured every 30 secs for last 5 mins:

swpin:                 swpout:
[ 0.27%][     11096K]  [ 0.25%][     10451K]
[ 0.27%][     11240K]  [ 0.29%][     12000K]
[ 0.29%][     12001K]  [ 0.02%][       853K]
[ 0.16%][      6849K]  [ 0.02%][       966K]
[ 0.53%][     21604K]  [ 0.09%][      4031K]
[ 0.10%][      4415K]  [ 0.03%][      1414K]
[ 0.43%][     17808K]  [ 0.37%][     15016K]
[ 0.64%][     25972K]  [ 1.61%][     65515K]
[ 0.26%][     10560K]  [ 0.88%][     36051K]
[ 0.07%][      3164K]  [ 0.83%][     33823K]
```

可以看到dbrm收集到了短期内的swapin和swapout数据，这样便于我们诊断由swap造成的性能或者hang问题。




解决OS 系统严重swap的一些思路：


1.  诊断是否存在内存泄露的进程，解决内存泄露
2.  调优SGA/PGA ，减少oracle对内存的占用
3.  利用  echo 3 > /proc/sys/vm/drop_caches 命令可以暂时释放一些cache的内存
4. 调整系统VM内存管理参数， 例如Linux上sysctl.conf中的以下几个参数

```
vm.min_free_kbytes   :Raising the value in /proc/sys/vm/min_free_kbytes will cause the system to start reclaiming memory at an earlier time than it would have before.


vm.vfs_cache_pressure :   At the default value of vfs_cache_pressure = 100 the kernel will attempt to reclaim dentries and inodes at a “fair” rate with respect to pagecache and swapcache reclaim. Decreasing vfs_cache_pressure causes the kernel to prefer to retain dentry and inode caches. Increasing vfs_cache_pressure beyond 100 causes the kernel to prefer to reclaim dentries and inodes.


vm.swappiness  : default 60 ;Apparently /proc/sys/vm/swappiness on Red Hat Linux allows the admin to tune how aggressively the kernel swaps out processes’ memory. Decreasing the  swappiness setting may result in improved Directory performance as the kernel

holds more of the server process in memory longer before swapping it out.
```

设置以下值，减少out of memory的可能性:

```
# Oracle-Validated setting for vm.min_free_kbytes is 51200 to avoid OOM killer

vm.min_free_kbytes = 51200

#vm.swappiness = 40

vm.vfs_cache_pressure = 200
```
