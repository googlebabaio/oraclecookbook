可以在expdp的时候加入2个参数进行trace

Majority of issues use following two data pump parameters:
- METRICS=Y -- this will provide timing data on number of objects
- TRACE=480300 -- this will trace dm and dw processes


两者都是未公开的参数expdp -help看不到

```
expdp system/xxx directory=datapumpdir dumpfile=isc_exp_date.dmp logfile=isc_exp_date.log schemas=isc CONTENT=METADATA_ONLY INCLUDE=TABLE:\"IN\(\'xxx\'\)\" METRICS=Y TRACE=480300
```
开启了METRICS=Y的话，导出日志记录了每个对象的导出时间
