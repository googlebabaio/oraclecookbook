<!-- toc -->

# ADG的手工切换

## 一、switchover
是用户有计划的进行停机切换，能够保证不丢失数据

切换步骤:

### 1.主库操作
#### 1.0 关闭其他实例
如果RAC环境,先关闭其他实例,只留一个实例1进行操作

#### 1.1 查看主库的状态
```
SQL> select open_mode,database_role,protection_mode,protection_level,switchover_status from v$database;
```
只有当`switchover_status`这一列状态为 `TO STANDBY` 或 `SESSIONS ACTIVE`才可以进行切换

#### 1.2 主库状态切换
```
alter system switch logfile;

alter system archive log current;

alter database commit to switchover to physical standby with session shutdown;
```
如果`swtichover_status`状态为`session active`，就需要在命令中加入`with session shutdown`子句。
执行后，会发现主库其实已经关闭。
加入`with session shutdown`相当于杀掉连接进程了，此时库已经关闭了，保险起见，再执行下
```
shutdown abort
```
#### 1.3 重启主库到mount阶段
重启主库到mount阶段，确保主库不会在有连接导致数据变动
```
startup mount
```
#### 1.4 查看主库切换后的状态
```
select switchover_status from v$database;
SWITCHOVER_STATUS
--------------------
TO PRIMARY
```

另外，在实际生产环境，遇到过启动到mount状态后，SWITCHOVER_STATUS的值有可能为`recovery needed`或`switchover laten`t或`not allowed`
这个时候主库就需要apply完所有归档日志才能切换，命令如下：
```
alter database recover managed standby database disconnect from session;
```



这一步过后,也可以执行`alter database open;`
此时主库虽然这样打开，但是还是read only的状态。因为上面执行了`alter database commit to switchover to physical standby with session shutdown;`这条语句。


### 2.备库操作
#### 2.0 关闭其他实例
如果是RAC集群环境,先关闭其他实例,保证只在节点1进行操作

#### 2.1 查看备库状态
```
select open_mode,database_role,protection_mode,protection_level,switchover_status from v$database;
OPEN_MODE            DATABASE_ROLE    PROTECTION_MODE      PROTECTION_LEVEL
-------------------- ---------------- -------------------- --------------------
SWITCHOVER_STATUS
--------------------
READ ONLY            PHYSICAL STANDBY MAXIMUM PERFORMANCE  MAXIMUM PERFORMANCE
NOT ALLOWED
```

注意备库的`SWITCHOVER_STATUS`此时是`not allowed`状态，说明备库日志并未应用完。
此时如果直接执行切换主库的操作，会报错
```
SQL> alter database commit to switchover to primary with session shutdown ;
alter database commit to switchover to primary with session shutdown
*
ERROR at line 1:
ORA-16139: media recovery required
```
要等这个switchover_status状态变成to primary以后才能切

状态如果为`session active`也可以直接做下面的切换操作
#### 2.2 执行切换操作
```
SQL> alter database commit to switchover to primary with session shutdown ;
```

#### 2.3 打开新的 主库
上一步执行完成后,这个时候新的主库的状态是`mount`的,需要`open`
```
SQL> alter database open;
```

### 3.在新的备库上（原主库）执行启用日志实时应用模式
#### 3.1 查询状态
```
SQL> select open_mode from v$database;

OPEN_MODE
--------------------
READ ONLY
```

#### 3.2 执行启动日志实时应用
```
SQL> alter database recover managed standby database using current logfile disconnect from session;
```

如果之前没有打开数据库,这个时候直接执行上面的命令会报错,需要先关闭恢复,再打开数据库,最后再开实时应用
```
SQL> alter database recover managed standby database cancel;
SQL> ALTER DATABASE OPEN;
SQL> alter database recover managed standby database using current logfile disconnect from session;
```

#### 3.3 查看状态及gap
```
SQL> select open_mode from v$database;

OPEN_MODE
--------------------
READ ONLY

SQL> select status ,gap_status from v$archive_dest_status where dest_id in (1,2);

STATUS    GAP_STATUS
--------- ------------------------
VALID
VALID

SQL> select switchover_status from v$database;

SWITCHOVER_STATUS
--------------------
NOT ALLOWED

SQL> select open_mode,database_role,protection_mode,protection_level,switchover_status from v$database;

OPEN_MODE            DATABASE_ROLE    PROTECTION_MODE      PROTECTION_LEVEL     SWITCHOVER_STATUS
-------------------- ---------------- -------------------- -------------------- --------------------
READ ONLY WITH APPLY PHYSICAL STANDBY MAXIMUM PERFORMANCE  MAXIMUM PERFORMANCE  NOT ALLOWED
```

### 4.主库（新的主库，原备库）查看状态
```
SQL> select switchover_status from v$database;

SWITCHOVER_STATUS
--------------------
TO STANDBY

SQL> select status ,gap_status from v$archive_dest_status where dest_id in (1,2);

STATUS    GAP_STATUS
--------- ------------------------
VALID
VALID     NO GAP

SQL> select open_mode,database_role,protection_mode,protection_level,switchover_status from v$database;

OPEN_MODE            DATABASE_ROLE    PROTECTION_MODE      PROTECTION_LEVEL     SWITCHOVER_STATUS
-------------------- ---------------- -------------------- -------------------- --------------------
READ WRITE           PRIMARY          MAXIMUM PERFORMANCE  MAXIMUM PERFORMANCE  TO STANDBY
```


## 二、FAILOVER
- Switchover动作是不会引起数据丢失的，Standby可以保证接受并且应用所有的Redo Log数据。
- 而Failover则不好说，根据不同的保护模式（Protection Mode），一个事务在主库上面是否被commit，是取决于standby上是否接受和应用上日志数据。所以，在进行Failover的时候，是可能会丢数据的。

在进行Failover之后，Primary库实际上是退出了Oracle HA架构体系，成为游离对象。Standby在切换之后就成为新的Primary。退出了adg架构体系的老的主库，需要重新搭建后，才能再被使用。


### 1.备库操作
由于主库已经不可访问，我们所有的操作都在备库完成：
检查日志gap的问题，可以查看视图v$archive_gap。
```
SQL> select thread#, low_sequence#, high_sequence# from v$archive_gap;
```
如果没有发现明显的gap现象，说明此次的failover不会有数据损失情况。在standby端，要进行关闭apply和结束应用动作。



```
SQL> alter database recover managed standby database cancel;

SQL> alter database recover managed standby database finish force;

SQL> select database_role from v$database;

SQL> alter database commit to switchover toprimary;

SQL> alter database open; 或者 shutdown immediate + startup
```


#### 2.注意事项
这里有个坑,如果是一拖N的情况,做failover的时候，不管`log_archive_dest_state_N`是否设为了`defer`，默认在启动到primary角色的时候，会将这个参数设置为`enable`，这意味着会将变为primary角色后产生的redo和归档往其他备库上发送；解决方法是将`log_archive_dest_n`设为空
