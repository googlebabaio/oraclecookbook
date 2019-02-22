<!-- toc -->
# 一、概述
## 1.概述
本次配置是RAC to RAC的adg
特点是:
- 1.采用临时的pfile启动备库到nomount状态，真正的spfile内容写到rman脚本中
- 2.不需要提前创建备库的standby控制文件
- 3.使用nohup+网络传输的方式进行数据同步,不需要做备份
- 4.适合任何的通用环境

## 2.环境介绍
```
############# standby ##############
#Public
10.215.212.130 ntyqxdb1
10.215.212.129 ntyqxdb2

#Private
192.168.220.16 ntyqxdb1-priv
192.168.220.17 ntyqxdb2-priv

#Virtual
10.215.212.132 ntyqxdb1-vip
10.215.212.131 ntyqxdb2-vip

#SCAN
10.215.212.133 tyqxdg-scan

############# primary ##############
#Public
10.220.220.11 qhtyqxdb01
10.220.220.12 qhtyqxdb02

#Private
192.168.220.11 qhtyqxdb01-priv
192.168.220.12 qhtyqxdb02-priv

#Virtual
10.220.220.13 qhtyqxdb01-vip
10.220.220.14 qhtyqxdb02-vip

#SCAN
10.220.220.15 rac-scan
```

# 二、配置过程：

## 1.配置监听与tnsnames
### 1.1 监听
备库因为在安装cluster的过程中已经配置好监听local和scan的监听了，所以只需要配置一个临时的监听，用作主库连接备库用，即可。

临时监听：
```
su - grid

LISTENER_TMP =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = ntyqxdb1-vip)(PORT = 11521))
    )
  )


SID_LIST_LISTENER_TMP =
(SID_LIST =
    (SID_DESC =
     (GLOBAL_DBNAME = tyqxdg)
     (ORACLE_HOME = /app/oracle/product/11.2.0)
     (SID_NAME = tyqxdg1)
    )
)
```
注意使用的时候，要把2个节点的local listener都关掉，只对其中一个节点使用临时监听

### 1.2 配置tnsnames
配置tnsname，需要用tnsping检查即可。
注意：
需要修改 2套环境的tnsnames，共计4个地方的tnsnames.ora


参考其中一个tnsnames.ora
```
qhtyqx =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = rac-scan)(PORT = 11521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = qhtyqx)
    )
  )

tyqxdg =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = tyqxdg-scan)(PORT = 11521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = tyqxdg)
    )
  )

tyqxdg1 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = ntyqxdb1-vip)(PORT = 11521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = tyqxdg)
      (INSTANCE_NAME = tyqxdg1)
    )
  )
```



## 2.主库配置第一次配置：
### 2.1 归档模式
### 2.2 主库本地的归档路径
```
alter system set log_archive_dest_1='location=+FRA' scope=both sid='*';
```

### 2.3 force模式
```
$ sqlplus / as sysdba
SQL>  select FORCE_LOGGING from v$database;
FOR
---
NO

如果返回值为：NO，则需要执行以下操作；如果返回值为YES不需要执行以下操作
SQL>  alter database force logging;
Database altered.
SQL>  select FORCE_LOGGING from v$database;
FOR
---
YES
```
### 2.4 打开补充日志
```
alter database add supplemental log data ;
select supplemental_log_data_min min from v$database ;
```

### 2.5 检查 remote_login_passwordfile
remote_login_passwordfile 必须设置为EXCLUSIVE
```
show parameter remote_login_passwordfile
```

### 2.6 修改主库参数
```
alter system set LOG_ARCHIVE_CONFIG='DG_CONFIG=(qhtyqx,tyqxdg)';
alter system set LOG_ARCHIVE_DEST_1='LOCATION=+FRA VALID_FOR=(ALL_LOGFILES,ALL_ROLES) DB_UNIQUE_NAME=qhtyqx' sid='*';
alter system set LOG_ARCHIVE_DEST_2='SERVICE=tyqxdg LGWR ASYNC VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE) DB_UNIQUE_NAME=tyqxdg compression=enable MAX_CONNECTIONS=4 reopen=60 ' sid='*';
alter system set FAL_SERVER='tyqxdg' sid='*';
alter system set STANDBY_FILE_MANAGEMENT=AUTO sid='*';
alter system set LOG_ARCHIVE_DEST_STATE_2=defer sid='*';
```

### 2.7 创建standby redo log
group号要根据实际情况来建
```
ALTER DATABASE ADD STANDBY LOGFILE thread 1 group 5  ('+DATA','+FRA') SIZE 500M;
ALTER DATABASE ADD STANDBY LOGFILE thread 1 group 6  ('+DATA','+FRA') SIZE 500M;
ALTER DATABASE ADD STANDBY LOGFILE thread 1 group 7  ('+DATA','+FRA') SIZE 500M;
ALTER DATABASE ADD STANDBY LOGFILE thread 1 group 8  ('+DATA','+FRA') SIZE 500M;
ALTER DATABASE ADD STANDBY LOGFILE thread 2 group 9  ('+DATA','+FRA') SIZE 500M;
ALTER DATABASE ADD STANDBY LOGFILE thread 2 group 10  ('+DATA','+FRA') SIZE 500M;
ALTER DATABASE ADD STANDBY LOGFILE thread 2 group 11  ('+DATA','+FRA') SIZE 500M;
ALTER DATABASE ADD STANDBY LOGFILE thread 2 group 12  ('+DATA','+FRA') SIZE 500M;
ALTER DATABASE ADD STANDBY LOGFILE thread 1 group 13  ('+DATA','+FRA') SIZE 500M;
ALTER DATABASE ADD STANDBY LOGFILE thread 2 group 14  ('+DATA','+FRA') SIZE 500M;
```

### 2.8 创建pfile传到备库
```
create pfile='/home/oracle/pfile.ora' from spfile;
```
这个仅用作参考

## 3.备库配置
### 3.1 两个节点都要创建adump的目录
```
mkdir -p /u01/app/oracle/admin/tyqxdg/adump
```

### 3.2 从主库copy密码


### 3.3 使用一个临时的pfile文件将数据库启动到nomount状态即可
init.ora
```
DB_NAME='qhtyqx'
DB_UNIQUE_NAME='tyqxdg'
DB_BLOCK_SIZE=8192
SGA_TARGET=81067507712
db_create_file_dest=''
control_files='+DATA/tyqxdg/controlfile/control01.ctl','+FRA/tyqxdg/controlfile/control02.ctl'
```

### 3.4 用nomount+pfile的方式启动备库实例1，并验证
经过验证这个地方，其实是需要使用临时监听的，不然只用scanip是连不上备库的实例1。
```
sqlplus sys/Oracle123@qhtyqx as sysdba
sqlplus sys/Oracle123@tyqxdg as sysdba
```

### 3.5 在备库执行duplicate的脚本
vi create_sb.sh
```
rman target sys/Oracle123@qhtyqx auxiliary sys/Oracle123@tyqxdg  <<EOF
run
{
allocate channel ch001 type disk;
allocate channel ch002 type disk;
allocate channel ch003 type disk;
allocate channel ch004 type disk;
allocate auxiliary channel ch005 type disk;
allocate auxiliary channel ch006 type disk;
allocate auxiliary channel ch007 type disk;
allocate auxiliary channel ch008 type disk;
duplicate target database for standby nofilenamecheck from active database
spfile
set db_file_name_convert='+DATA/qhtyqx','+DATA/tyqxdg'
set log_file_name_convert='+DATA/qhtyqx','+DATA/tyqxdg','+FRA/qhtyqx','+FRA/tyqxdg'
set db_unique_name='tyqxdg'
set control_files='+DATA/tyqxdg/controlfile/control01.ctl','+FRA/tyqxdg/controlfile/control02.ctl'
set audit_file_dest='/u01/app/oracle/admin/tyqxdg/adump'
set db_create_file_dest='+DATA'
set remote_listener='tyqxdg-scan:11521'
set thread='1'
set instance_number='1';
release channel ch001;
release channel ch002;
release channel ch003;
release channel ch004;
release channel ch005;
release channel ch006;
release channel ch007;
release channel ch008;
}
exit
EOF
```

在后台执行，避免操作终端中断带来的影响
```
chmod +x  create_sb.sh
nohup sh create_sb.sh > c1.log &
```

执行完成后，顺便检查备库的standby redolog是否有了。


## 4.duplicate完成后主库的操作

修改主库参数，打开归档的开关
```
alter system set LOG_ARCHIVE_DEST_STATE_2=enable sid='*';
```

## 5.duplicate完成后备库的操作
### 5.1 修改参数
```
alter system set LOG_ARCHIVE_CONFIG='DG_CONFIG=(qhtyqx,tyqxdg)';
alter system set LOG_ARCHIVE_DEST_1='LOCATION=+FRA VALID_FOR=(ALL_LOGFILES,ALL_ROLES) DB_UNIQUE_NAME=tyqxdg';
alter system set LOG_ARCHIVE_DEST_2='SERVICE=qhtyqx LGWR ASYNC VALID_FOR=(ONLINE_LOGFILE,PRIMARY_ROLE) DB_UNIQUE_NAME=qhtyqx compression=enable MAX_CONNECTIONS=4 reopen=60 ';
alter system set STANDBY_FILE_MANAGEMENT=AUTO;
alter system set FAL_SERVER='qhtyqx';
alter system set instance_number = 1 sid='tyqxdg1' scope=spfile;
alter system set instance_number = 2 sid='tyqxdg2' scope=spfile;
alter system set undo_tablespace='UNDOTBS1' sid='tyqxdg1'  scope=spfile;
alter system set undo_tablespace='UNDOTBS2' sid='tyqxdg2'  scope=spfile;
alter system set thread = 1 sid='tyqxdg1'  scope=spfile;
alter system set thread = 2 sid='tyqxdg2'  scope=spfile;
alter system set "_trace_files_public"=true  scope=spfile;
alter system set "_optimizer_use_feedback"=false  scope=spfile;
alter system set LOG_ARCHIVE_MAX_PROCESSES=4;
```

### 5.2 创建备库的spfile，并放入到asm磁盘组中
```
create pfile='/tmp/init.ora' from spfile;
shutdown immediate;
startup pfile='/tmp/init.ora' nomount;
create spfile='+DATA/tyqxdg/spfiletyqxdg.ora' from pfile='/tmp/init.ora';
shutdown immediate;
```

```
cd $ORACLE_HOME/dbs

vi initntyqxdg1.ora
spfile='+DATA/tyqxdg/spfiletyqxdg.ora'
```

在另外一个节点也要做：
```
vi initntyqxdg2.ora
```

### 5.3 备库开始追日志
```
$ sqlplus / as sysdba
SQL> startup mount
SQL>  ALTER DATABASE RECOVER MANAGED STANDBY DATABASE DISCONNECT FROM SESSION;
```

主库执行切换日志
```
$ sqlplus / as sysdba
SQL>  ALTER SYSTEM ARCHIVE LOG CURRENT;
SQL>  ALTER SYSTEM ARCHIVE LOG CURRENT;
SQL>  ALTER SYSTEM ARCHIVE LOG CURRENT;
SQL>  ALTER SYSTEM ARCHIVE LOG CURRENT;
SQL>  ALTER SYSTEM ARCHIVE LOG CURRENT;
```

### 5.4 查看主备库的alert日志以及相应视图
查看相应的视图
```
SQL> select * from v$archive_gap;
SQL> select error from v$archive_dest;
```

这个地方出问题遇到的最多的就是当使用了非默认的1521端口后,local_listener没有配

### 5.5 备库启动到active dataguard的模式
在追了一段时间archivelog后，切换到实时应用redo
```
SQL> alter database recover managed standby database cancel;
SQL> alter database open;
SQL> ALTER DATABASE RECOVER MANAGED STANDBY DATABASE USING CURRENT LOGFILE DISCONNECT FROM SESSION;
```

注意！如果打开报错如下:
```
SQL> alter database recover managed standby database cancel;

Database altered.

SQL>  alter database open;

 alter database open
*
ERROR at line 1:
ORA-10458: standby database requires recovery
ORA-01152: file 1 was not restored from a sufficiently old backup
ORA-01110: data file 1: '+DATA/tyqxdg/datafile/system.260.1000300965'
```
那么就很明显需要看下是不是主库的日志没有传过来了，去看这个视图：
```
select error from v$archive_dest;
```

### 5.6 验证
```
SQL> SELECT thread#,max(sequence#) from v$archived_log where applied='YES' GROUP BY THREAD#;
SQL> select * from v$archive_gap;
```

### 5.7 启动备库的另外一个节点
```
$ sqlplus / as sysdba
SQL>  startup;
```

### 5.8 将实例信息加入到grid中
```
$ srvctl add database -d tyqxdg -o $ORACLE_HOME -p +DATA/tyqxdg/spfiletyqxdg.ora -r physical_standby
$ srvctl add instance -d tyqxdg -n ntyqxdb1 -i tyqxdg1
$ srvctl add instance -d tyqxdg -n ntyqxdb2 -i tyqxdg2
```

注意：将备节点的数据库加入到CRS中进行管理，但CRS在启动后并不会自动对其追加归档，在CRS重启后需要手动执行追加日志的操作，命令如下：
```
SQL> ALTER DATABASE RECOVER MANAGED STANDBY DATABASE USING CURRENT LOGFILE DISCONNECT FROM SESSION;
```

至此，搭建RAC到RAC的adg完成。
