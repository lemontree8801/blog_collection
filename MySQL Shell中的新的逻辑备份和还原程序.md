- [原文链接](https://elephantdolphin.blogspot.com/2020/07/new-logical-backup-and-restore.html)

# MySQL Shell中的新的逻辑备份和还原程序
8.0.21版的MySQL Shell或者`mysqlsh`带来三个新的程序用于执行逻辑备份和还原。这个设计用于更容易迁移您的5.7或者8.0中的数据到新的MySQL Data Service，但是它们本身也可以很好地工作。它们具有压缩功能，可以显示活动进度，并且可以将任务分散到多个线程上。

`util.dumpInstance()`用于备份整个MySQL实例，`util.dumpSchemas()`让您决定备份哪个schema，`util.loadDump()`是还原工具。

## 备份实例
```
util.dumpInstance("/tmp/instance",{ "showProgress" : "true" })
```
<为简洁输出，忽略一部分内容>
```
>1 thds dumping - 100% (52.82K rows / ~52.81K rows), 0.00 rows/s, 0.00 B/s uncompressed, 0.00 B/s compressed         
Duration: 00:00:00s                                                                                        
Schemas dumped: 4                                                                                          
Tables dumped: 26                                                                                          
Uncompressed data size: 3.36 MB                                                                            
Compressed data size: 601.45 KB                                                                            
Compression ratio: 5.6                                                                                     
Rows written: 52819                                                                                        
Bytes written: 601.45 KB                                                                                   
Average uncompressed throughput: 3.36 MB/s                                                                 
Average compressed throughput: 601.45 KB/s   
```
这是在一个在SAS盘，有限内存，最新版的Fedora系统的笔记本上执行的结果。我用这些程序在个更大的实例上执行，发现其性能出色。

这个应用程序和这篇博文中其他功能有日志选项，我建议您设置`pager`通过`\h util.dumpInstance`阅读在线帮助。

## Schema 备份
我创建了一个名为`demo`的快速测试数据库，表名为`x`，4个整型字段，1个整型主键，大约一百万记录。以下输出是在一个十年前的笔记本上的输出结果。

```
JS > util.dumpSchemas(['demo'],"/tmp/demo")
Acquiring global read lock
All transactions have been started
Locking instance for backup
Global read lock has been released
Writing global DDL files
Preparing data dump for table `demo`.`x`
Writing DDL for schema `demo`
Writing DDL for table `demo`.`x`
Data dump for table `demo`.`x` will be chunked using column `id`
Running data dump using 4 threads.
NOTE: Progress information uses estimated values and may not be accurate.
Data dump for table `demo`.`x` will be written to 1 file
1 thds dumping - 100% (1000.00K rows / ~997.97K rows), 577.69K rows/s, 19.89 MB/s uncompressed, 8.58 MB/s compressed
Duration: 00:00:01s                                                                                                 
Schemas dumped: 1                                                                                                   
Tables dumped: 1                                                                                                    
Uncompressed data size: 34.44 MB                                                                                    
Compressed data size: 14.85 MB                                                                                      
Compression ratio: 2.3                                                                                              
Rows written: 999999                                                                                                
Bytes written: 14.85 MB                                                                                             
Average uncompressed throughput: 20.11 MB/s                                                                         
Average compressed throughput: 8.67 MB/s    
```

性能非常出色。您可以将schema名放入JSON数组中作为第一个参数，一次性备份多个schema。

## 还原
除非可以从中还原，否则世界上最好的备份是没有用的。我将表`x`重命名为y，然后恢复数据。

在执行之前请确认`local_infile`设置为`ON`。

```
JS > util.loadDump("/tmp/demo")
Loading DDL and Data from '/tmp/demo' using 4 threads.
Target is MySQL 8.0.21. Dump was produced from MySQL 8.0.21
Checking for pre-existing objects...
Executing common preamble SQL
Executing DDL script for schema `demo`
Executing DDL script for `demo`.`x`
[Worker003] demo@x@@0.tsv.zst: Records: 999999  Deleted: 0  Skipped: 0  Warnings: 0
Executing common postamble SQL                        
                                     
1 chunks (1000.00K rows, 34.44 MB) for 1 tables in 1 schemas were loaded in 20 sec (avg throughput 1.72 MB/s)
0 warnings were reported during the load.
```
## 总结
这三个应用程序是保护您的数据安全的非常快速和强大的工具。也许今天还可以看到`mysqldump`。这三个应用程序和克隆插件证明，您可以比以往更快地保存、复制和恢复数据。
