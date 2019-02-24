1.实例启动,只能一个能启动,报错:
Crash with kjzdattdlm: Can not attach to DLM

可参考:
1386843.1



2.CRS显示正在运行的dbinstance是offline状态
且一旦加入到集群就会是的速度非常慢



3.sqlplus密码用有特殊字符
使用转移来解决:
```
sqlplus 'username/"password"'@service name
```



4.解决Oracle 11g R2 RAC 无法在客户端通过scanIP连接数据库
在服务端可以正常使用,但是在客户端使用plsql连的时候就报错`ORA-12545`

MetaLink：ORA-12545 or ORA-12537 While Connecting to RAC through SCAN name
[ID 970619.1]得到解决方法，修改数据库的`local_listener`参数：

最终发现是`local_listener`中涉及到的地址用的是域名,改成ip地址即可
如:
```
改之前:
alter system set local_listener='(DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=TCP)(HOST=raconde1-vip)(PORT=1521))))' sid='devdb1';

改之后:
alter system set local_listener='(DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=TCP)(HOST=172.16.0.194)(PORT=1521))))' sid='devdb2';
```


5.修改scan_listener的名字
```
srvctl stop scan_listener
srvctl stop scan

修改dns或者hosts里面相应的scan的名字
root操作:

srvctl modify scan -n new_scan_name

srvctl start scan
srvctl start scan_listener
srvctl status scan
```


6.客户端不能正常连接oracle，监听状态为"Not All Endpoints Registered"
```
Solution

From 11.2 onwards, all listeners should be runing from GRID_HOME, listener and listener_scan<n> entry should be added automatically into listener.ora, no manual editing is required for TCP definition.

1. Stop the listener running from RDBMS ORACLE_HOME

$<RDBMS ORACLE_HOME>/bin/lsnrctl stop LISTENER

2. stop the listener from GRID_HOME

$<GRID_HOME>/bin/srvctl stop listener -n <node name>
$<GRID_HOME>/bin/srvctl stop scan_listener -i <scan#>

eg:

$<GRID_HOME>/bin/srvctl stop listener -n racnode1
$<GRID_HOME>/bin/srvctl stop scan_listener -i 1

If above command fails to stop the tnslsnr process, please use "kill -9 <pid of tnslsnr>" to stop the LISTENER and LISTENER_SCAN1 process.

3. remove any manually added LISTENER definition from listener.ora if it exists

4. restart the LISTENER and LISTENER_SCAN1  from GRID_HOME

$<GRID_HOME>/bin/srvctl start listener -n <node name>
$<GRID_HOME>/bin/srvctl start scan_listener -i <scan#>

5. check crsctl stat res -t output, they both should show ONLINE status now.



参考资料：Listener in INTERMEDIATE status with "Not All Endpoints Registered" [ID 1454439.1]
```
