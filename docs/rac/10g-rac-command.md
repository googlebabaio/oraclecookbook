<!-- toc -->

# 10gRAC 常用命令
## Oracle Clusterware的命令集可以分为4种
- 节点层：osnodes
- 网络层：oifcfg
- 集群层：crsctl, ocrcheck,ocrdump,ocrconfig
- 应用层：srvctl,onsctl,crs_stat

下面分别来介绍这些命令。

## 节点层：
只有一个命令：　osnodes，这个命令用来显示集群点列表
```
./olsnodes -n -p -i
```

## 网络层
只有一个命令：oifcfg
oifcfg 命令用来定义和修改Oracle集群需要的网卡属性，这些属性包括网卡的网段地址，子网掩码，接口类型等。要想正确的使用这个命令，必须先知道Oracle 是如何定义网络接口的，Oracle的每个网络接口包括名称，网段地址，接口类型3个属性。
oifcfg 命令的格式如下：`interface_name/subnet:interface_type`
这些属性中没有IP地址，但接口类型有两种，public和private，前者说明接口用于外部通信，用于Oracle Net和VIP 地址，而后者说明接口用于Interconnect。
接口的配置方式分为两类：global 和node-specific。 前者说明集群所有节点的配置信息相同，也就是说所有节点的配置是对称的，而后者意味着这个节点的配置和其他节点配置不同，是非对称的。

- Iflist：显示网口列表
- Getif: 获得单个网口信息
- Setif：配置单个网口
- Delif：删除网口


## 集群层
集群层是指由Clusterware组成的核心集群，这一层负责维护集群内的共享设备，并为应用集群提供完整的集群状态视图，应用集群依据这个视图进行调整。
这一层共有4个命令：crsctl, ocrcheck,ocrdump,ocrconfig. 后三个是针对OCR 磁盘的。

### 配置CRS 栈是否自启动
```
crsctl disable crs
crsctl enable crs
```

### 启动CRS
CRS没有启动在使用crs_stat命令查看状态的时候

会收到如下的报错信息
```
CRS-0184:Cannot communicate with the CRSdaemon
```
启动命令为:
```
crsctl start crs
```
停止CRS命令为:
```
crsctl stop crs
```

### 查看CRS资源状态
```
10g:
crs_stat –t

11g:
crsctl res status -t
```


### 关闭特定集群资源
查找资源名字`crs_stat | grep -i name= | cut -d '=' -f2`

找到资源名字后关闭
```
crs_stop ora.RACDB.RACDB1.inst
```

确定资源开启
```
crs_start ora.RACDB.RACDB1.inst
```

### 所有集群资源
```
关闭所有集群资源
crs_stop -all

启动所有集群资源
crs_start -all
```

### 当前节点所有CRS资源
```
crsctl start resources
```

### srvctl停止和启动节点应用
```
关闭节点应用
srvctl stop nodeapps -n rac1
srvctl start nodeapps -n rac1
```

### 对CRS后台进程做健康检查
```
crsctl check crs
```

### 检查OCR
```
ocrcheck
```

### 检查表决磁盘
```
crsctl query css votedisk
```

## 应用层
应用层就是指RAC数据库了，这一层有若干资源组成，每个资源都是一个进程或者一组进程组成的完整服务，这一层的管理和维护都是围绕这些资源进行的。有: `srvctl,onsctl, crs_stat` 三个命令。

- crs_stat 这个命令用于查看CRS维护的所有资源的运行状态
- onsctl命令用于管理配置ONS(Oracle Notification Service). ONS 是Oracle Clusterware 实现FAN EventPush模型的基础。
- srvctl是RAC维护中最常用，也是最复杂的。可以操作下面的几种资源：Database，Instance，ASM，Service，Listener 和Node Application，其中Node application又包括GSD，ONS，VIP。资源除了使用srvctl工具统一管理外，某些资源还有自己独立的管理工具，比如ONS可以使用onsctl命令进行管理；Listener 可以通过lsnrctl 管理。

### 配置数据库随CRS的启动而自动启动
```
srvctl enable database -d raw
srvctl disable database -d raw
```

### 关闭某个实例的自动启动
```
srvctl disable instance -d raw -i raw1
```

### 启动，停止对象与查看对象
在RAC 环境下启动，关闭数据库虽然仍然可以使用SQL/PLUS方法，但是更推荐使用srvctl命令来做这些工作，这可以保证即使更新CRS中运行信息，可以使用start/stop 命令启动，停止对象，然后使用status 命令查看对象状态。

#### 指定启动状态
```
srvctl start database -d raw -i raw1 -o mount
```
#### 关闭对象，并指定关闭方式
```
srvctl stop instance -d raw -i raw1 -o immediate
```
#### 停止 Oracle RAC 10g 环境
```
$ srvctl stop instance -d orcl -i orcl1
$ srvctl stop asm -n rac1
$ srvctl stop nodeapps –n rac1
```
#### 启动 Oracle RAC 10g 环境
```
$ srvctl start nodeapps -n rac1
$ srvctl start asm -n rac1
$ srvctl start instance -d orcl -i orcl1
```

#### 启动/停止所有实例及其启用的服务。
```
$ srvctl start database -d orcl
$ srvctl stop database -d orcl
```
