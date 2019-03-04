# 调整顺序
这个地方特别需要强调一下操作的步骤，先做需要重启的操作：

1. 调整数据库实例1的参数
2. 关闭数据库实例1，再调整asm的参数
3. 重启节点1的asm，重启节点1的实例
4. 在没有问题的情况下，再在第2个节点重复上面的操作
5. 以上均没有问题后，再修改操作系统的huge page的参数，方法也是一个节点修改完启动好之后再修改第二个节点
6. 主要注意amm与huge page的问题

## 一、操作系统
### 1.设置huge_page
```
修改/etc/sysctl.conf 中vm.nr_hugepages为（64*1024）/2=32768
修改完成后重启数据库实例，执行 grep Huge /proc/meminfo 查看hugepage是否生效
```

### 2.节点1启动ntp服务

配置同节点2的/etc/ntp.conf文件，配置完成后启动ntpd服务`service ntpd start`，并加入重启自动启动`check ntpd on`


### 3.配置历史命令保存与记录时间
```
编辑/etc/profile 文件，加入如下三行：
export HISTFILESIZE=1000
export HISTSIZE=1000
export HISTTIMEFORMAT=%Y-%m-%d:%H-%M-%S

保存后,再 source /etc/profile
```

## 二、asm实例
### 1.增大asm实例的memory_target
```
# su - grid
$ sqlplus / as sysasm
SQL> create pfile='/tmp/lmpfile.ora' from spfile;
SQL> alter system set memory_max_target=2G scope=spfile sid='+ASM1';
SQL> alter system set memory_target=2G scope=spfile sid='+ASM1';
重启实例

再第二个节点执行同样操作
SQL> alter system set memory_max_target=2G scope=spfile sid='+ASM2';
SQL> alter system set memory_target=2G scope=spfile sid='+ASM2';
```

## 三、数据库方面
### 0.每做一次需要修改系统参数的操作时都先备份spfile
```
create pfile='/home/oracle/pfile.bak' from spfile;
```

### 1.关闭DRM
注意，最好是一个节点一个节点进行操作
```
# su - oracle
SQL> alter system set “_gc_undo_affinity”=FALSE sid='xxx1' scope=spfile;
SQL> alter system set “_gc_policy_time”=0 sid='xxx1' scope=spfile;
```

注意，如果两个节点的参数不一致的话，有可能在启动到mount状态的时候，就会报错：
`ORA-01105，ORA-01606`
解决办法是查看初始化参数和隐藏参数在两个节点的值,看哪些不一致
```
参考隐藏参数
set linesize 333
col name for a35
col description for a66
col value for a30
SELECT   i.ksppinm name,
   i.ksppdesc description,
   CV.ksppstvl VALUE
FROM   sys.x$ksppi i, sys.x$ksppcv CV
   WHERE   i.inst_id = USERENV ('Instance')
   AND CV.inst_id = USERENV ('Instance')
   AND i.indx = CV.indx
   AND i.ksppinm LIKE '/_gc%' ESCAPE '/'
ORDER BY   REPLACE (i.ksppinm, '_', '');


查看不同的参数
col name format a25
col value format a55
set lines 210
select * from (select name,value,inst_id from gv$parameter where inst_id=1) a full join (select name,value,inst_id from gv$parameter where inst_id=2) b
on a.name = b.name where a.value <> b.value;

```


### 2.调整redo大小
```
查询redo的相关信息
select * from gv$log ;
select * from gv$logfile ;
alter system switch logfile;

如果有adg,还需要注意standby redo log是否需要调整.

添加 redo log:
alter database add logfile thread 1 group xxxxx('+磁盘组的名字','+磁盘组的名字') size 500M;
...
alter database add logfile thread 1 group xxxxx('+磁盘组的名字','+磁盘组的名字') size 500M;

alter database add logfile thread 2 group xxxxx('+磁盘组的名字','+磁盘组的名字') size 500M;
...
alter database add logfile thread 2 group xxxxx('+磁盘组的名字','+磁盘组的名字') size 500M;

切换
alter system switch logfile; 或者 alter system archive log current ;
alter system checkpoint

删除以前的redo log
alter database drop logfile group XXX;
````

###3.高资源消耗表记录有55个
需要与业务结合起来看，根据列出的资源对象，如果是表，则查看其对应的SQL的执行计划是否合理。


### 4.表并行度
_**以下几个需要注意，最好是停业务，或者业务不繁忙的时候操作，因为涉及到表和索引的碎片整理等**_

```
查询
select owner,table_name,degree from dba_tables where degree>1;

需要与业务人员沟通,看这几个表是做什么，再来判断是否需要关闭并行
```

### 5.索引并行度与高度分析
```
查询高度>3的索引
select owner,index_name,table_owner,table_name,BLEVEL from dba_indexes where blevel > 3;

对这些索引进行重建（rebuild）
alter index index_name rebuild; ## 能停业务,可以用它
alter index index_name rebuild online; ##支持dml,但是会有更多的锁
```

### 6.表碎片率大于50%的数量有2个记录
```
通过shrink space来调整以上表的碎片。
alter table table_name enable row movement; ##shrink space前,先开启行迁移
alter table table_name shrink space;
再根据实际情况关闭行迁移
alter table table_name disable row movement;
```

### 7.碎片率大于50%的数量记录
```
col owner format a15;
col table_name format a30;
col index_name format a30;

select
  idx.owner owner,
  idx.table_name tablename,
  idx.index_name index_name,
  idx.blocks idx_blocks,
  tbl.blocks tbl_blocks,
  trunc(idx.blocks/tbl.blocks*100)/100 pct
from
  (select i.owner owner ,i.index_name index_name,SUM(S1.blocks) blocks,i.table_owner table_owner,
	 i.table_name table_name
	 from dba_segments s1,dba_indexes i
	 where
	  s1.owner=i.owner and s1.segment_name=i.index_name and
	  i.owner not in ('SYS','OUTLN','SYSTEM','MGMT_VIEW','SYSMAN','DBSNMP','WMSYS','XDB',
	  'DIP','GOLDENGATE','CTXSYS' )
	 GROUP BY i.owner ,i.index_name ,i.table_owner , i.table_name  ) idx,
  (select	t.owner owner ,t.table_name table_name,SUM(s2.blocks) blocks
	 from dba_segments s2,dba_tables t
	 where
	  s2.owner=t.owner and s2.segment_name=t.table_name and
	  t.owner not in ('SYS','OUTLN','SYSTEM','MGMT_VIEW','SYSMAN','DBSNMP','WMSYS','XDB',
	  'DIP','GOLDENGATE','CTXSYS' )
	  GROUP BY  T.OWNER,T.TABLE_NAME
	  ) tbl
where
	idx.table_owner=tbl.owner and
	idx.table_name=tbl.table_name and
	(idx.blocks/tbl.blocks)>0.5 and
	idx.blocks>200
	order by 4;

对这些索引进行重建（rebuild）
alter index index_name rebuild; ## 能停业务,可以用它
alter index index_name rebuild online; ##支持dml,但是会有更多的锁
```
