
impdp命令特殊用途，可以将数据库的一个用户迁移到另一台机器上的数据库的用户中。如果目标用户不存在，还可以对应的创建该用户。
 快速的把A库上的用户迁移到B库上。
下面就来看一下命令格式：
B库下执行命令：
```
impdp username/passwd@dbsname schema=userA remap_schema=userA:userB remap_tablespace=tbsA:tbsB network_link=dblink_to_userA_on_userB
```
说明： Userid: Username/passwd@dbsname。用户建议为system。
Remap_schema: userA:userB。数据库用户映射。 同用户的话，此参数省略
Remap_tablespace: tbsA:tbsB。默认表空间映射。
Schemas: userA。必须是dblink中指定用户。建议不指定。
Directory: 该种模式下，此参数指定的是日志文件的路径。如果不指定，则路径默认为data_pump_dir。
Network_link: 在B库上创建的连接到A库的dblink。


不过有几个前提：
1、username：这个操作的数据库用户建议是system，如果是其他用户的话就需要有dba权限的用户才能执行；
2、dblink：必须能够连接到对应库上的数据库用户下。


优点：只是不再将数据导出后导入，而是直接将数据从源库导入到目的库。

Impdp  system/password@目标库  directory=DMPDIR  schemas=TESTI  network_link=目标库上建 的dblink remap_schema=TESTI:TESTA

说明：directory定义的路径是在目标库上定义的路径。network_link同样是在目标库上定义的tnsnames.ora中的信息。

上面语句的操作是将源库的TESTI用户的数据，导入到目标库的TESTA用户下。

如果从原库导出schema A，且db_link建立在schema A上，则原库的该schema A用户需具有exp_full_database权限否则会报错：

With the Partitioning, OLAP and Data Mining options ORA-31631: privileges are required ORA-39149: cannot link privileged user to non-privileged user

1、这个操作是局域网内迁移数据最方便的工具，不过也可能是速度最慢的工具。

2、同时还可用此方法导表空间，单独的表等等.....tablespaces=xxx_tbs即可。...

3、在目标库上建立到源端的db_link的时候，可以针对system用户建立，这样就可以导出导入全库数据或者表空间数据。

4、当针对某个用户A创建db_link时，需要给该用户Aexp_full_database的权限才可以导出该schema得数据。

5、在导入的过程中注意目标数据库存在表数据的情况，可采用table_exists_action来处理。
