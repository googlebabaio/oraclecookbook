## kkjcre1p: unable to spawn jobq slave process, error 1089

今天数据库在改ip后，shutdown的时候不能正常关闭，后台alert日志报错：
```
kkjcre1p: unable to spawn jobq slave process, error 1089
```

查了下资料，参考：Kkjcre1p: Unable To Spawn Jobq Slave Process, Error 1089 [ID 344275.1]
警告原因如下：
>If a job is about to be spawned when shutdown of database is in progress, you will see these errors in the alert log file and this is perfectly valid.

## 解决方法：
### 设置_JOB_QUEUE_INTERVAL更大值，减少出现该警告概率
One workaround that we can suggest is to set an underscore parameter `_JOB_QUEUE_INTERVAL=120` or greater value
The default value is 60 but when we change to 120 there are less chances of getting the above warnings in the alert log file.

### 由bug`23102157`引起，打补丁
>补丁链接 https://updates.oracle.com/download/23102157.html
