<!-- toc -->
# undo的作用
## 1.回退事务
当执行DML操作修改数据时,UNDO数据被存放到UNDO段,而新数据则被存放到数据段中,如果事务操作存在问题,就需要回退事务,以取消事务变化.假定用户A执行了语句UPDATE emp SET sal=1000 WHERE empno=7788后发现,应该修改雇员7963的工资,而不是雇员7788的工资,那么通过执行ROLLBACK语句可以取消事务变化.当执行ROLLBACK命令时,oracle会将UNDO段的UNDO数据800（工资）写回到数据段中.

## 2.读一致性

用户检索数据库数据时,oracle 总是使用用户只能看到被提交过的数据(读取提交)或特定时间点的数据(SELECT语句时间点).这样可以确保数据的一致性.例如,当用户A执行语句 UPDATE emp SET sal=1000 WHERE empno=7788时,UNDO记录会被存放到回滚段中,而新数据则会存放到EMP段中;假定此时该数据尚未提交,并且用户B执行SELECT sal FROM emp WHERE empno=7788,此时用户B将取得UNDO数据 800,而该数据正是在UNDO记录中取得的.

## 3.事务恢复

事务恢复是例程恢复的一部分,它是由oracle server自动完成的.如果在数据库运行过程中出现例程失败(如断电,内存故障,后台进程故障等),那么当重启oracle server时,后台进程SMON会自动执行例程恢复,执行例程恢复时,oracle会重新做所有未应用的记录.回退未提交事务.

## 4.倒叙查询(FlashBack Query)

倒叙查询用于取得特定时间点的数据库数据, 它是9i新增加的特性,假定当前时间为上午11:00,某用户在上午10:00执行UPDATE emp SET sal= 3500 WHERE empno=7788语句,修改并提交了事务(雇员原工资为3000),为了取得10:00之前的雇员工资,用户可以使用倒叙查询特征.

# undo有2个著名的报错
- ora-01555
- ora-30036


undo表空间是循环使用的,所以使用率是100%也是经常可以看到的。当undo_retetion时间到期了,就把那一部分到期的undo信息清理掉。
当出现`ora-01555`，我们会结合`undo_retention`参数来判断undo的大小到底够不够，那个时候才会去扩展undo的大小。如果没有出现`ora-01555`，那就不用管undo的大小。另外，undo表空间始终会使用率达到100%的，所以不用过多的看使用率100%。

## ora-01555
读数据的时候从buffer读,buffer的数据块头有指针,如果指针有指向undo段的,那么就会去读undo的快照,如果这个查询很长,花费的时间超过了`undo_retention`里面的快照数据所保存的时间,这个时候快照就被undo移除,所以这个时候就会发生`ora-01555`

## ora-30036
`ORA-30036: unable to extend segment by 8 in undo tablespace 'UNDOTBS1'` 这个错,就是说明undo表空间已经满了,根据undo_retention设置的时间，里面没有expried或者active的数据又不能释放，并且不能扩展了，所以后面的dml操作都没有做上.，那当然就登录不起撒。

## 另外,undo表空间满了对dml有影响
undo表空间满了,如果事务都在active，这个时候就不能释放undo空间,而这个时候undo又满了，那么这个时候其他新的事务就只能等待,这个在alert日志就不能直接反应出来。
只能通过查询出问题时间段的awr或者ash，来判断是否有DML操作的等待。

# UNDO参数

## 1.UNDO_MANAGEMENT
该初始化参数用于指定UNDO 数据的管理方式.如果要使用自动管理模式,必须设置该参数为AUTO,如果使用手工管理模式,必须设置该参数为MANUAL,使用自动管理模式时, oracle会使用undo表空间管理undo管理,使用手工管理模式时,oracle会使用回滚段管理undo数据,

需要注意,使用自动管理模式时,如果没有配置初始化参数UNDO_TABLESPACE,oracle会自动选择第一个可用的UNDO表空间存放UNDO数据,如果没有可用的UNDO表空间,oracle会使用SYSTEM回滚段存放UNDO记录,并在ALTER文件中记载警告.

## 2.UNDO_TABLESPACE

该初始化参数用于指定例程所要使用的UNDO表空间,使用自动UNDO管理模式时,通过配置该参数可以指定例程所要使用的UNDO表空间.

在RAC(Real Application Cluster)结构中,因为一个UNDO表空间不能由多个例程同时使用,所有必须为每个例程配置一个独立的UNDO表空间.

## 3.UNDO_RETENTION

该初始化参数用于控制UNDO数据的最大保留时间,其默认值为900秒,从9i开始,通过配置该初始化参数,可以指定undo数据的保留时间,从而确定倒叙查询特征(Flashback Query)可以查看到的最早时间点.

# undo相关视图
## 1.显示当前实例正在使用的UNDO表空间
```
show parameter undo_tablespace
```

## 2.显示数据库的所有UNDO表空间
```
SELECT tablespace_name FROMdba_tablespaces WHERE contents=’UNDO’;
```

## 3.显示UNDO表空间统计信息

使用自动UNDO管理模式时,需要合理地设置UNDO表空间的尺寸,为例合理规划UNDO表空间尺寸,应在数据库运行的高峰阶段搜集UNDO表空间的统计信息.最终根据该统计信息确定UNDO表空间的尺寸.通过查询动态性能视图`V%UNDOSTAT`,可以搜集UNDO统计信息.
```
SELECT TO_CHAR(BEGIN_TIME,’HH24:MI:SS’) BEGIN_TIME,
TO_CHAR(END_TIME,’HH24:MI:SS’) END_TIME,UNDOBLKS
FROM V$UNDOSTAT;
```

- `BEGIN_TIME`用于标识起始统计时间
- `END_TIME`用于标识结束统计时间
- `UNDOBLKS`用于标识UNDO数据所占用的数据块个数


>另外，oracle每隔10分钟生成一行统计信息

## 4.显示UNDO段统计信息

使用自动UNDO 管理模式时,oracle会在UNDO表空间上自动建立`10个UNDO`段,通过查询动态信息视图`V$ROLLNAME`,可以显示所有联机UNDO段的名称,通过查询动态性能视图`V$ROLLLISTAT`,可以显示UNDO段的统计信息.通过在V$ROLLNAME和V$ROLLLISTAT之间执行连接查询,可以监视特定UNDO段的特定信息.
```
SELECT a.name, b.xacts, b.writes, b.extents
FROM v$rollname a, v$rollstat b
WHERE a.usn=b.usn;
```
其中：
- `Name`用于标识UNDO段的名称
- `xacts`用于标识UNDO段所包含的活动事务个数
- `Writes`用于标识在undo段上所写入的字节数
- `extents`用于标识UNDO段的区个数

## 5.显示活动事务信息

当执行DML操作时,oracle会将这些操作的旧数据放到UNDO段中
- 动态性能视图`v$session`用于显示会话的详细信息
- 动态性能视图`v$transaction`用于显示事务的详细信息
- 动态性能视图`v$rollname`用于显示联机UNDO段的名称

通过在这3个动态性能视图之间执行连接查询,可以确定正在执行事务操作的会话,事务所使用的UNDO段,以及事务所占用的UNDO块个数.
```
col username format a10
col name format a10

SELECT a.username, b.name, c.used_ublk
FROM v$session a, v$rollname b, v$transaction c
WHERE a.saddr=c.ses_addr AND b.usn=c.xidusn
AND a.username=’SCOTT’;
```

## 6.显示UNDO区信息

数据字典视图`dba_undo_extents`用于显示UNDO表空间所有区的详细信息.包括UNDO区尺寸和状态等信息.

```
SELECT extend_id, bytes, status FROM dba_undo_extents
WHERE segment_name’_SYSSMU5$’;
```
其中：
- `extent_id`用于标识区编号
- `bytes`用于标识区尺寸
- `status`用于标识区状态(ACTIVE:表示该区处于活动状态,EXPIRED:标识该区未用)
