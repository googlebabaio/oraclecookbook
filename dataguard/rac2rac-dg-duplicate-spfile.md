<!-- toc -->
# 一、概述
## 1.概述
本次配置是RAC to RAC的adg
采用的配置方式和以往不同的是，这次没有使用broker来辅助完成，而是采用rman+duplicate+spfile的方式进行配置。
特点是:
- 1.采用临时的pfile启动备库到nomount状态，真正的spfile内容写到rman脚本中
- 2.不需要提前创建备库的standby控制文件
- 3.使用nohup+网络传输的方式进行数据同步,不需要做备份
- 4.适合任何的通用环境

## 2.环境介绍
```
############# standby ##############
#Public
10.220.220.16 racdg1
10.220.220.17 racdg2

#Private
192.168.220.16 racdg1-priv
192.168.220.17 racdg2-priv

#Virtual
10.220.220.18 racdg1-vip
10.220.220.19 racdg2-vip

#SCAN
10.220.220.20 racdg-scan

############# primary ##############
#Public
10.220.220.11 racnode1
10.220.220.12 racnode2

#Private
192.168.220.11 racnode1-priv
192.168.220.12 racnode2-priv

#Virtual
10.220.220.13 racnode1-vip
10.220.220.14 racnode2-vip

#SCAN
10.220.220.15 rac-scan
```

# 二、配置过程：

## 1.配置监听与tnsnames
### 1.1 监听
备库因为在安装cluster的过程中已经配置好监听local和scan的监听了，所以只需要配置一个临时的监听，用作主库连接备库用，即可。

临时监听：
```grid
su - grid

LISTENER_TMP =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = racdg1-vip)(PORT = 1521))
    )
  )


SID_LIST_LISTENER_TMP =
(SID_LIST =
    (SID_DESC =
     (GLOBAL_DBNAME = racdg)
     (ORACLE_HOME = /u01/app/oracle/product/11.2.0/db_1)
     (SID_NAME = racdg1)
    )
)
```
注意使用的时候，要把grid的local listener都关掉，只对其中一个节点使用临时监听
### 1.2 配置tnsnames
配置tnsname，需要用tnsping检查即可。
注意：
需要修改 2套环境的tnsnames，共计4个地方的tnsnames.ora


参考其中一个tnsnames.ora
```
ORCL =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = rac-scan)(PORT = 11521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = orcl)
    )
  )

RACDG =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = racdg-scan)(PORT = 11521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = racdg)
    )
  )

RACDG1 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = racdg1-vip)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = racdg)
      (INSTANCE_NAME = racdg1)
    )
  )
```



## 2.主库配置第一次配置：
### 2.1 归档模式
### 2.2 主库本地的归档路径
```
alter system set log_archive_dest_1='location=+ARCH' scope=both sid='*';
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

### 2.4 检查 remote_login_passwordfile
remote_login_passwordfile 必须设置为EXCLUSIVE
```
show parameter remote_login_passwordfile
```

### 2.5 修改主库参数
```
alter system set LOG_ARCHIVE_CONFIG='DG_CONFIG=(orcl,racdg)';
alter system set LOG_ARCHIVE_DEST_1='LOCATION=+ARCH VALID_FOR=(ALL_LOGFILES,ALL_ROLES) DB_UNIQUE_NAME=orcl' sid='*';
alter system set LOG_ARCHIVE_DEST_2='SERVICE=racdg LGWR ASYNC VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE) DB_UNIQUE_NAME=racdg compression=enable MAX_CONNECTIONS=4 reopen=60 ' sid='*';
alter system set FAL_SERVER='racdg';
alter system set STANDBY_FILE_MANAGEMENT=AUTO;
alter system set LOG_ARCHIVE_DEST_STATE_2=defer sid='*';
```

### 2.6 创建standby redo log
```
ALTER DATABASE ADD STANDBY LOGFILE thread 1 group 9  '+ORADATA' SIZE 100M;
ALTER DATABASE ADD STANDBY LOGFILE thread 1 group 10  '+ORADATA' SIZE 100M;
ALTER DATABASE ADD STANDBY LOGFILE thread 1 group 11  '+ORADATA' SIZE 100M;
ALTER DATABASE ADD STANDBY LOGFILE thread 1 group 12  '+ORADATA' SIZE 100M;
ALTER DATABASE ADD STANDBY LOGFILE thread 2 group 13  '+ORADATA' SIZE 100M;
ALTER DATABASE ADD STANDBY LOGFILE thread 2 group 14  '+ORADATA' SIZE 100M;
ALTER DATABASE ADD STANDBY LOGFILE thread 2 group 15  '+ORADATA' SIZE 100M;
ALTER DATABASE ADD STANDBY LOGFILE thread 2 group 16  '+ORADATA' SIZE 100M;
ALTER DATABASE ADD STANDBY LOGFILE thread 1 group 17  '+ORADATA' SIZE 100M;
ALTER DATABASE ADD STANDBY LOGFILE thread 2 group 18  '+ORADATA' SIZE 100M;
```

### 2.7 创建pfile传到备库
```
create pfile='/home/oracle/pfile.ora' from spfile;
```
这个仅用作参考

## 3.备库配置
### 3.1 两个节点都要创建adump的目录
```
mkdir -p /u01/app/oracle/admin/racdg/adump
```

### 3.2 从主库copy密码


### 3.3 使用一个临时的pfile文件将数据库启动到nomount状态即可

```
DB_NAME='orcl'
DB_UNIQUE_NAME='racdg'
DB_BLOCK_SIZE=8192
SGA_TARGET=8589934592
db_create_file_dest=''
control_files='+ORADATA/racdg/controlfile/control01.ctl','+ORADATA/racdg/controlfile/control02.ctl'
```

### 3.4 用nomount+pfile的方式启动备库实例1，并验证
经过验证这个地方，其实是需要先建立一个临时监听的，不然连不上备库的实例1。
```
sqlplus sys/Oracle123@orcl as sysdba
sqlplus sys/Oracle123@racdg1 as sysdba
```

### 3.5 在备库执行duplicate的脚本
vi create_sb.sh
```
rman target sys/Oracle123@orcl auxiliary sys/Oracle123@racdg1  <<EOF
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
set db_file_name_convert='+DATA/orcl','+ORADATA/racdg'
set log_file_name_convert='+ARCH/orcl','+ORADATA/racdg','+DATA/orcl','+ORADATA/racdg'
set db_unique_name='racdg'
set control_files='+ORADATA/racdg/controlfile/control01.ctl','+ORADATA/racdg/controlfile/control02.ctl'
set audit_file_dest='/u01/app/oracle/admin/racdg/adump'
set db_create_file_dest='+ORADATA'
set remote_listener='racdg-scan:11521'
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
alter system set LOG_ARCHIVE_CONFIG='DG_CONFIG=(orcl,racdg)';
alter system set LOG_ARCHIVE_DEST_1='LOCATION=+ORADATA VALID_FOR=(ALL_LOGFILES,ALL_ROLES) DB_UNIQUE_NAME=racdg';
alter system set LOG_ARCHIVE_DEST_2='SERVICE=orcl LGWR ASYNC VALID_FOR=(ONLINE_LOGFILE,PRIMARY_ROLE) DB_UNIQUE_NAME=orcl compression=enable MAX_CONNECTIONS=4 reopen=60 ';
alter system set STANDBY_FILE_MANAGEMENT=AUTO;
alter system set FAL_SERVER='orcl';
alter system set instance_number = 1 sid='racdg1' scope=spfile;
alter system set instance_number = 2 sid='racdg2' scope=spfile;
alter system set undo_tablespace='UNDOTBS1' sid='racdg1'  scope=spfile;
alter system set undo_tablespace='UNDOTBS2' sid='racdg2'  scope=spfile;
alter system set thread = 1 sid='racdg1'  scope=spfile;
alter system set thread = 2 sid='racdg2'  scope=spfile;
alter system set "_trace_files_public"=true  scope=spfile;
alter system set "_optimizer_use_feedback"=false  scope=spfile;
alter system set LOG_ARCHIVE_MAX_PROCESSES=4;
```

### 5.2 创建备库的spfile，并放入到asm磁盘组中
```
create pfile='/tmp/init.ora' from spfile;
shutdown immediate;
startup pfile='/tmp/init.ora' nomount;
create spfile='+ORADATA/racdg/spfile.ora' from pfile='/tmp/init.ora';
shutdown immediate;
```

```
cd $ORACLE_HOME/dbs

vi initracdg1.ora
spfile='+ORADATA/RACDG/spfile.ora'
```

在另外一个节点也要做：
```
vi initracdg2.ora
```

### 5.3 备库开始追日志
```
$ sqlplus / as sysdba
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
```

### 5.5 备库启动到active dataguard的模式
在追了一段时间archivelog后，切换到实时应用redo
```
SQL> alter database recover managed standby database cancel;
SQL> alter database open;
SQL> ALTER DATABASE RECOVER MANAGED STANDBY DATABASE USING CURRENT LOGFILE DISCONNECT FROM SESSION;
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
$ srvctl add database -d racdg -o $ORACLE_HOME -p +ORADATA/racdg/spfile.ora -r physical_standby
$ srvctl add instance -d racdg -n racdg1 -i racdg1
$ srvctl add instance -d racdg -n racdg2 -i racdg2
```

注意：将备节点的数据库加入到CRS中进行管理，但CRS在启动后并不会自动对其追加归档，在CRS重启后需要手动执行追加日志的操作，命令如下：
```
SQL> ALTER DATABASE RECOVER MANAGED STANDBY DATABASE USING CURRENT LOGFILE DISCONNECT FROM SESSION;
```

至此，搭建RAC到RAC的adg完成。
