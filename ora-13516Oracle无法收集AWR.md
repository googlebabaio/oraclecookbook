手工给一个库做快照，
```
begin
	dbms_workload_repository.create_snapshot;
end;
```
发现报错ORA-13516
```
ERROR at line 1:
ORA-13516: AWR Operation failed: only a subset of SQL can be issued
ORA-06512: at "SYS.DBMS_WORKLOAD_REPOSITORY", line 174
ORA-06512: at "SYS.DBMS_WORKLOAD_REPOSITORY", line 222
ORA-06512: at line 1
```
保证错的意思是：
```
SQL> !oerr ora 13516
13516, 00000, "AWR Operation failed: %s"
// *Cause:  The operation failed because AWR is not available. The
//          possible causes are: AWR schema not yet created; AWR
//          not enabled; AWR schema not initialized; or database
//          not open or is running in READONLY or STANDBY mode.
// *Action: check the above conditions and retry the operation.
```

经检查是因为mmon进程不存在导致，问了下运维人员，原来是他们在停数据库的时候，使用的`shutdown`命令停，然后又嫌弃停的时候等待时间过长，所以就又cancel了。然后去查alert日志，证实了这一个说法。。