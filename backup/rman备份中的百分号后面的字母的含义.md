backup incremental level 0 database format='LEV0_%d_%t_%U_%s＿%p'
format=string 文件路径和名称的格式串，其中可包含宏变量：

|  符号 |  含义 |
|---|---|
|%c| copy ID|
|%p| backup piece ID|
|%s| backup set ID|
|%e| log sequence|
|%h| log thread ID|
|%d| database name|
|%n| database name(x填充到8个字符)|
|%I| DBID|
|%f| file ID |
|%F| DBID, day, month, year, and sequencer的复合 |
|%N| tablespace name|
|%t| timestamp|
|%M| mh mm格式|
|%Y| year yyyy格式|
|%u| backup set+time((x填充到8个字符)|
|%U| %u_%p_%c|
|%%| %|

The format specifier %U is replaced with unique filenames for the files when you take backups.
the %F element of the format string combines the DBID, day, month, year, and sequence number to generate a unique filename. %F must be included in any control file autobackup format.
