backup archivelog all delete all input 说明



RMAN提供了`DELETE ALL INPUT`参数，加在BACKUP命令后，则会在完成备份后自动删除归档目录中已备份的归档日志（即会产生一个归档日志的备份文件，但是归档日志会被删除）。如果备份文件不丢失的话，则不会导致数据丢失。
