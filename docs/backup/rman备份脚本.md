## rman备份脚本
### 全备
```
run{
sql 'alter system switch logfile';
crosscheck archivelog all;
backup as backupset full database;
crosscheck backup;
delete noprompt expired backup;
delete noprompt obsolete;
delete noprompt archivelog until time 'sysdate-8' all;
backup current controlfile;
sql 'alter database backup controlfile to trace';
sql 'alter system switch logfile';
}
```

### 0级增量备份
```
run{
sql 'alter system switch logfile';
crosscheck archivelog all;
backup as backupset incremental level 0 database;
crosscheck backup;
delete noprompt expired backup;
delete noprompt obsolete;
delete noprompt archivelog until time 'sysdate-8' all;
backup current controlfile;
sql 'alter database backup controlfile to trace';
sql 'alter system switch logfile';
}
```

0级增量备份与全备的区别：
二者都是全备，但是0级增量备份可以用于增量备份恢复的基础，而单独的全备不能用于增量备份的恢复基础！

### 1级增量备份
```
run{
sql 'alter system switch logfile';
crosscheck archivelog all;
backup as backupset incremental level 1 database;
crosscheck backup;
delete noprompt expired backup;
delete noprompt obsolete;
delete noprompt archivelog until time 'sysdate-8' all;
sql 'alter system switch logfile';
}
```


## crontab脚本
```
0,10,20,30,40,50 * * * * /u01/app/oracle/scripts/rmanL0Call.sh >> /u01/app/oracle/scripts/rmanL0Call.log 2>&1
```
