---> sql有问题
---> 频繁的登录都会执行这个sql
---> 造成大量硬解析
---> 造成等待事件library cache lock metux X
---> 造成session没有释放
---> 造成process满了
---> 系统连不上了

```
Question: What does the Library cache: mutex x concurrency wait signify?  Can you explain how to reduce Library cache mutex x waits?  My AWR report shows high counts for "library cache mutex x" events.

Answer:  Mutexes are objects that exist within the operating system to provide access to shared memory structures. They are similar to latches, which will be covered in following chapters, as they are serialized mechanisms used to control access to shared data structures within the Oracle SGA.  See my notes here on mutexes.

High waits for library cache mutex x can be due to:

Not pinning hot PL/SQL with dbms_shared_pool.markhot.
A too-small value for shared_pool_size (memory_target).
Setting cursor_sharing=similar (which should be changed to exact or force).

Settings for session_cached_cursors.
Here we run a ASH query to see the library cache mutex x events:

select *
from
   (select
      event,
      count(1)
   from
      v$active_session_history
   where
      sample_time > (sysdate - 20/1440)
   group by
      event
   order by 2 desc)
where rownum < 10;

EVENT                                    COUNT(1)
---------------------------------------- ----------
library cache: mutex X                        6543

We can also run this query to get the P1 (references the object) and counts:

select
   event,
   p1,
   count(1)
from
   v$active_session_history
where
   sample_time > (sysdate - 20/1440)
and
   event = 'library cache: mutex X'
group by
   event, p1
order by 3;

EVENT                                    P1         COUNT(1)
---------------------------------------- ---------- ----------
library cache: mutex X                    421399181          1
library cache: mutex X                   3842104349          1
library cache: mutex X                   1412465886        297
library cache: mutex X                   2417922189      50615

In plain English, the Library cache: mutex x event simply means that Oracle is performing a library cache operation.  The PL/SQL (procedures, packages) that are the target for the "library cache mutex x" can be pinned into the library cache using the dbms_shared_pool.markhot procedure, thus reducing the mutex events for the object.

exec dbms_shared_pool.markhot(schema=>'SYS',objname=>'DBMS_RANDOM',NAMESPACE=>1);
exec dbms_shared_pool.markhot(schema=>'SYS',objname=>'DBMS_RANDOM',NAMESPACE=>2);
exec dbms_shared_pool.markhot(schema=>'SYS',objname=>'DBMS_OUTPUT',NAMESPACE=>1);
exec dbms_shared_pool.markhot(schema=>'SYS',objname=>'DBMS_OUTPUT',NAMESPACE=>2);

Above, we have marked the dbms_random package in the library cache.

When you see high "library cache mutex x" events, you will want to see which library cache objects are the target of the operation.  This query will expose the mutex target:

select *
from
   (select
      case when (kglhdadr = kglhdpar)
      then 'Parent'
      else 'Child '||kglobt09 end cursor,
      kglhdadr ADDRESS,
      substr(kglnaobj,1,20) NAME,
      kglnahsh HASH_VALUE,
      kglobtyd TYPE,
      kglobt23 LOCKED_TOTAL,
      kglobt24 PINNED_TOTAL,
      kglhdexc EXECUTIONS,
      kglhdnsp NAMESPACE
   from
      x$kglob
     -- where kglobtyd != 'CURSOR'
   order by
      kglobt24 desc)
where
   rownum <= 10;

References:

- MOSC:  "Wait event: library cache: mutex X" (MOSC Document ID 727400.1)

- MOSC Bug 20879889 - Fixed in 11.2.0.4

- MOSC Patch 20879889: INSERT INTO MV LOG LEAVING TOO MANY OPEN CURSORS AFTER UPGR TO 11.2.0.4
```

参考:
http://blog.itpub.net/18474/viewspace-1060805/
https://blog.csdn.net/mchdba/article/details/51299062
https://blog.csdn.net/u010719917/article/details/52947839
http://www.oracleplus.net/arch/1124.html










# 其他一些疑点
RFS network connection lost at host 'iscyzdg' error 3135
Warning: VKTM detected a time drift.



# dia0进程
参考:
http://bbs.landingbj.com/t-0-173361-1.html
dia0 High Memory Usage (文档 ID 1376981.1)


diag进程：它为后台诊断进程，用于获得实例中有关进程失败等的诊断信息(用于执行oradebug命令)。


DIA0：另一个数据库诊断进程，负责检测Oracle数据库中的挂起(hang)和死锁的处理。


mman（memory manager）进程：
自动内存管理进程。作用是每分钟都检查AWR性能信息，并根据这些信息决定SGA组件最佳分布。
通过STATISTICS_LEVEL设置统计级别；SGA_TARGET设置SGA总大小


CJQ0（job queue coordinator）进程：
数据库定时任务
Jnnn进程：
最多可以有1 000个作业队列进程:J000，J001，…，J999
Jnnn进程处理完一个作业后再处理下一个作业.每个作业队列进程一次只运行一个作业,直至完成.如果需同时运行多个作业,就需多个进程.
这里不存在多线程或作业的抢占,一旦运行一个作业，就会一直运行到完成(或失败),Jnnn进程退出.
ORACLE在开始时只会启动一个进程，即作业队列协调器(CJQ0)，它在作业队列表中看到需要运行的作业时，才会启动Jnnn进程。
如果Jnnn进程完成其工作，并发现没有要处理的新作业，此时Jnnn进程就会退出.
JOB_QUEUE_PROCESSES参数的设置是用户可调的


RVWR（recover writer）进程：
为flashback database提供日志记录，把数据块的前镜像写入日志。


DBRM：数据库资源管理进程, (The database resource manager process)，负责设置资源计划和其他的资源管理的工作。


VKTM：virtual keeper of time，用于提供wall-clock time，（每秒钟更新一次）。提供每二十毫秒更新一次的reference-time counter，看起来有点类似计时器的功能（适用于RAC）。


CTWR（change tracking writer）进程：
跟踪数据块的变化，把数据块地址记录到change_tracking file文件中
RMAN做增量恢复时通过这个文件来确定哪些数据库发生了变化，并进行备份。
https://blog.csdn.net/easonulove/article/details/41348723
