rman中的备份与恢复:

## 备份
备份的rman脚本
```
$ more all_backup.rcv
run{
configure retention policy to recovery window of 200 days;        #设置两百天备份过期              
configure controlfile autobackup on;
configure controlfile autobackup format for device type disk to '/orabak/rmanbak/backupfiles/all/%F.bak';
allocate channel c1 type disk;
allocate channel c2 type disk;
allocate channel c3 type disk;
allocate channel c4 type disk;
backup full tag 'dbfull' format '/orabak/rmanbak/backupfiles/all/full%u_%s_%p' database;
sql 'alter system archive log current';
backup filesperset 3 format '/orabak/rmanbak/backupfiles/all/arch%u_%s_%p'
archivelog all delete input;           #备份归档可选，可以单独定期备份
crosscheck backup;
delete noprompt expired backup;
report obsolete;
delete noprompt obsolete;
release channel c1;
release channel c2;
release channel c3;
release channel c4;
}
```

执行rman脚本
```
#!/bin/sh
export ORACLE_SID=orcl
export ORACLE_HOME=/u01/app/oracle/product/11.2.0/db_1
export PATH=$ORACLE_HOME/bin:$PATH
export TIMESTAMP=`date +'%Y_%m_%d_%H:%M'`
rman target / nocatalog cmdfile=/home/oracle/rmanbak/scripts/all_backup.rcv log=/orabak/rmanbak/logs/log_all_${TIMESTAMP}.log
```

## 恢复
