本次配置是RAC--RAC的adg
采用的配置方式和以往不同的是,这次将采用rman+duplicate+spfile的方式进行配置
特点是:
1.采用pfile启动备库到nomount状态
2.不需要提前创建备库的standby控制文件
3.使用nohup+网络传输的方式进行数据同步
