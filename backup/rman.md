
<!-- toc -->

# rman中的备份与恢复

## 创建目录存放相应的备份
```
mkdir -p /orabak/logs
mkdir -p /orabak/rmanbak/{datafile,archivelog,controlfile,spfile}
```

## rman全备
备份的rman脚本
```
$ more all_backup.rcv
run{            
configure controlfile autobackup on;
configure controlfile autobackup format for device type disk to '/orabak/rmanbak/controlfile/controlfile_%F.bak';
allocate channel c1 type disk;
allocate channel c2 type disk;
allocate channel c3 type disk;
allocate channel c4 type disk;
backup full tag 'dbfull' format '/orabak/rmanbak/datafile/full_%u_%s_%p' database;
sql 'alter system archive log current';
backup archivelog all format '/orabak/rmanbak/archivelog/arch_%u_%s_%p';
backup spfile format '/orabak/rmanbak/spfile/spfile_%d_%U';
delete noprompt expired backup;
delete noprompt obsolete;
release channel c1;
release channel c2;
release channel c3;
release channel c4;
}
```

> 如果设置了`configure controlfile autobackup on;` 那么controlfile和spfile都会自动被备份的,当然为了更好的找到它们,我们还是可以指定路径


执行rman脚本
```
#!/bin/sh
export ORACLE_SID=orcl
export ORACLE_HOME=/u01/app/oracle/product/11.2.0/db_1
export PATH=$ORACLE_HOME/bin:$PATH
export TIMESTAMP=`date +'%Y_%m_%d_%H:%M'`
rman target / nocatalog cmdfile=/home/oracle/rmanbak/scripts/all_backup.rcv log=/orabak/logs/rman_bk_all_${TIMESTAMP}.log
```

## 恢复

### 只恢复某个数据文件
保证数据库能启动到`mount`状态
```
rman > restore datafile 1;
rman > recover datafile 1;
rman > alter database open;
```


### 恢复spfile
```
# 就算spfile丢失了,可以鼓捣启动到nomount状态,再restore spfile
RMAN> startup nomount;
RMAN> restore spfile from '/orabak/rmanbak/spfile/spfile_xxxx.bak';
RMAN> startup nomount force;
```

### 恢复全库
建设全库都丢了,那么恢复的步骤就是
1. 恢复spfile
2. 恢复controlfile
3. restore数据库
4. 添加archive的目录进行恢复
5. 打开数据库

```
step1: 恢复spfile
RMAN> startup nomount;
RMAN> restore spfile from '/orabak/rmanbak/spfile/spfile_xxxx.bak';
RMAN> startup nomount force;

step2: 恢复控制文件
RMAN>  restore controlfile from '/orabak/rmanbak/controlfile/controlfile_xxx'; //还原控制文件

RMAN> alter database mount; //启动数据库到mount状态

step3: restore数据库
RMAN> list backup of database; //查看备份集
RMAN> restore database;

step4: 添加archive的目录进行恢复
RMAN> catalog start with '/orabak/rmanbak/archivelog/';  //将归档的日志也导入进来
RMAN> restore database; //还原数据文件

step5: 打开数据库
RMAN> alter database open;
```

# 参考
https://www.cnblogs.com/storymedia/p/4536553.html
https://blog.csdn.net/cuiyan1982/article/details/78327506
