

## 不重启清空共享池--***慎用！！！***
```
alter system flush shared_pool;
```

## 重建AWR脚本
```
@?\rdbms\admin\catawrtb.sql
@?\rdbms\admin\utlrp.sql

ORACLE 11g需要需要运行如下脚本：
@?\rdbms\admin\execsvrm.sql
```

> 注意在RAC环境下的话，需要取消集群参数后，待执行完成后再次修改过来：
  `alter system set cluster_database = false scope = spfile;`

## 手动创建快照
```
BEGIN
  DBMS_WORKLOAD_REPOSITORY.CREATE_SNAPSHOT();
END;
```

## 删除快照
删除99-100的快照
```
BEGIN
  DBMS_WORKLOAD_REPOSITORY.DROP_SNAPSHOT_RANGE (low_snap_id => 99,
                           high_snap_id => 100, dbid => xxxx);
END;
```

## 清除AWR数据
```
@?\rdbms\admin\catnoawr.sql
```


## 查看当前的快照保留策略
```
select * from dba_hist_wr_control;
```
## 修改快照的保留间隔
保留10天，30分钟采集一次 ，topnsql为50
```
BEGIN
  DBMS_WORKLOAD_REPOSITORY.MODIFY_SNAPSHOT_SETTINGS( retention => 10*24*60,
                 interval => 30, topnsql => 50, dbid => xxxxx);
END;
```

## 创建基线
```
基于快照号创建基线
BEGIN
    DBMS_WORKLOAD_REPOSITORY.CREATE_BASELINE (start_snap_id => 200,
                   end_snap_id => 201, baseline_name => 'my_baseline',
                   dbid => xxxx, expiration => 10);
END;

基于指定时间创建基线
BEGIN
   DBMS_WORKLOAD_REPOSITORY.CREATE_BASELINE (
      start_time      => TO_DATE ('2017-04-14 6:00:00', 'yyyy-mm-dd hh24:mi:ss'),
      end_time        => TO_DATE ('2017-04-14 8:00:00', 'yyyy-mm-dd hh24:mi:ss'),
      baseline_name   => 'peak_baseline2',
      expiration      => 10);
END;
```
> 注意：
> expiration标识保留时间为10天，如果没有设置则则该基线以及相应的快照被永久保留
> 通过dba_hist_baseline可以查询到相关信息

## 删除基线
```
BEGIN
  DBMS_WORKLOAD_REPOSITORY.DROP_BASELINE (baseline_name => 'my_baseline',
                  cascade => FALSE, dbid => xxxx);
END;
```

## 基线重命名
```
BEGIN
    DBMS_WORKLOAD_REPOSITORY.RENAME_BASELINE (
                   old_baseline_name => 'my_baseline',
                   new_baseline_name => 'your_baseline',
                   dbid => xxxx);
END;
```

## 修改缺省移动窗口基线保留值
--查看缺省的window_size
```
SELECT baseline_name, baseline_type, moving_window_size
FROM   dba_hist_baseline
WHERE  baseline_name = 'SYSTEM_MOVING_WINDOW';

BASELINE_NAME            BASELINE_TYPE MOVING_WINDOW_SIZE
------------------------ ------------- ------------------
SYSTEM_MOVING_WINDOW     MOVING_WINDOW                  8

BEGIN
    DBMS_WORKLOAD_REPOSITORY.MODIFY_BASELINE_WINDOW_SIZE (
                   window_size => 7,
                   dbid => xxxx);
END;
/

--window_size为天，只能够小于等于当前快照保留时间，否则报错，如下：

ERROR at line 1:
ORA-13541: system moving window baseline size (864000)
greater than retention (691200)
ORA-06512: at "SYS.DBMS_WORKLOAD_REPOSITORY", line 686
ORA-06512: at line 2
```

## 管理基线样本
### 创建单个基线模板
```
BEGIN
   DBMS_WORKLOAD_REPOSITORY.CREATE_BASELINE_TEMPLATE (
      start_time      => TO_DATE ('2018-08-14 09:00:00', 'yyyy-mm-dd hh24:mi:ss'),
      end_time        => TO_DATE ('2018-08-14 11:00:00', 'yyyy-mm-dd hh24:mi:ss'),
      baseline_name   => 'baseline_single',
      template_name   => 'template_single'',
      expiration      => 10,
      dbid            => xxxx);
END;
```
在上面的示例中，创建了一个单一的基线样本，并且指定了相应的时间范围，基线的名称及保留期限等。那么在这个时间范围内的相应的快照会被保留，同时这个基线可以用于后续在发现性能问题的时候进行比对。

### 创建重复基线样本
重复的基线样本指的是在将来某个特定的时间范围内，Oracle会参照这个设定的样本自动创建基线。比如，可以创建一个重复的基线样本，使得在2018年每周一9:00-10:00自动生成基线。
```
SQL> alter session set nls_date_format='yyyy-mm-dd hh24:mi:ss';

BEGIN
   DBMS_WORKLOAD_REPOSITORY.CREATE_BASELINE_TEMPLATE (
      day_of_week            => 'monday',
      hour_in_day            => 9,
      duration               => 2,
      expiration             => 30,
      start_time             => '2018-01-01 09:00:00',
      end_time               => '2018-12-31 23:00:00',
      baseline_name_prefix   => 'baseline_2018_mondays',
      template_name          => 'template_2018_mondays',
      dbid                   => xxxx);
END;
/

--查看已经创建的基线样本
SQL> select t.template_name,
  2       t.template_type,
  3       t.start_time,
  4       t.end_time,
  5       t.day_of_week,
  6       t.hour_in_day,
  7       t.duration
  8  from dba_hist_baseline_template t;

TEMPLATE_NAME         TEMPLATE_ START_TIME          END_TIME            DAY_OF_WE HOUR_IN_DAY DURATION
--------------------- --------- ------------------- ------------------- --------- ----------- --------
template_single       SINGLE    2018-08-14 09:00:00 2018-08-14 11:00:00
template_2018_mondays REPEATING 2018-01-01 09:00:00 2018-12-31 10:00:00 MONDAY             17        3
```
在上面的示例中创建了一个重复从2018年1月1日起的每周一(day_of_week)会自动生成一个基线，其开始时间为9点(hour_in_day)，其持续时间为2小时(duration)，有效期为30天(expiration)，整个基线的起止时间范围为：2018-01-01 09:00:00至2018-12-31 23:00:00，同时也指定了基线样本的名称以及基线前缀名称。


### 基线样本的删除
```
BEGIN
  DBMS_WORKLOAD_REPOSITORY.DROP_BASELINE_TEMPLATE (
                   template_name => 'template_single',
                   dbid => xxxx);
END;
```

## 查看各个对象占用SYSAUX的详细信息
@?/rdbms/admin/awrinfo.sql

## 生成AWR报告
```
--单实例下生成AWR报告
SQL> @?/rdbms/admin/awrrpt.sql

--RAC环境下生成AWR报告
SQL> @$ORACLE_HOME/rdbms/admin/awrgrpt.sql

--指定数据库实例生成AWR报告
SQL> @$ORACLE_HOME/rdbms/admin/awrrpti.sql

--生成SQL语句AWR报告
SQL> @$ORACLE_HOME/rdbms/admin/awrsqrpt.sql

--指定实例生成SQL语句AWR报告
SQL> @$ORACLE_HOME/rdbms/admin/awrsqrpi.sql

--生成比较的AWR报告
SQL> @$ORACLE_HOME/rdbms/admin/awrddrpt.sql

--RAC环境下生成比较的AWR报告
@$ORACLE_HOME/rdbms/admin/awrgdrpt.sql
```
## AWR仓库数据的导出与导入
导出：
```
SQL> @?/rdbms/admin/awrextr.sql
~~~~~~~~~~~~~
AWR EXTRACT
~~~~~~~~~~~~~
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
~  This script will extract the AWR data for a range of snapshots  ~
~  into a dump file.  The script will prompt users for the         ~
~  following information:                                          ~
~     (1) database id                                              ~
~     (2) snapshot range to extract                                ~
~     (3) name of directory object                                 ~
~     (4) name of dump file                                        ~
~~~~~~~~~~~~~~~~~~~~~

...

```

导入:
导入过程也类似
```
SQL> @?/rdbms/admin/awrload
```

## AWR相关的重要视图和数据字典
```
v$active_session_history : 显示活跃的数据库会话的活动，每秒采样一次

v$metric和v$metric_history：提供度量数据来跟踪系统性能。视图被组织成好几个组，这些组定义在v$metricgroup视图中

DBA_HIST_ACTIVE_SESS_HISTORY:内存中活动会话历史信息

DBA_HIST_BASELINE : 捕获的基线的信息

DBA_HIST_BASELINE_DETAILS: 特定基线的明细信息

DBA_HIST_BASELINE_TEMPLATE：基线模板相关信息

DBA_HIST_DATABASE_INSTANCE：数据库环境

DBA_HIST_DB_CACHE_ADVICE：根据历史数据预测在不同的cache size下的物理读

DBA_HIST_DISPATCHER：每个snapshot下调度进程的信息

DBA_HIST_DYN_REMASTER_STATS：动态remastering进程的统计信息

DBA_HIST_IOSTAT_DETAIL：按未见类型和功能来统计的历史I/O信息

DBA_HIST_SHARED_SERVER_SUMMARY：共享服务器的统计信息

DBA_HIST_SNAPSHOT：快照信息

DBA_HIST_SQL_PLAN：执行计划

DBA_HIST_WR_CONTROL：AWR控制信息
```

