- [原文链接](https://scalegrid.io/blog/exploring-mysql-binlog-server-ripple/)


# MySQL Binlog Server——Ripple
MySQL没有限制在复制拓扑中连接主节点的从节点的数量。然而随着从节点的数量增加，它们会对主节点资源产生负面影响，因为二进制日志需要以不同的速度发送给不同的从节点。如果主节点数据修改很多，那么仅二进制日志就可能使主节点的网络接口达到饱和。

这个问题的经典解决方案是部署一个`binlog server`——位于主节点和它的从节点之间的中间代理服务器。`Binlog server`被设置为主节点的一个从节点，反过来，它又充当原始从节点的主节点。它从主节点接收二进制日志事件，但并不应用，而是将它们提供给所有其他从节点。通过这种方式，主节点上的负载大大减少，同时，`binlog server`可以更有效地向从节点提供二进制日志，因为它并不需要应用二进制日志。
![enter image description here](https://scalegrid.io/blog/wp-content/uploads/2020/05/MySQL-Binglog-Server-Deployment-Diagram.jpg)

[`Ripple`](https://github.com/google/mysql-ripple)是由Pavel Ivanov开发的一个开源的`binlog server`。Percona之前的一篇名为[ MySQL Ripple: The First Impression of a MySQL Binlog Server](https://www.percona.com/blog/2019/03/15/mysql-ripple-first-impression-of-mysql-binlog-server/)的博客很好地介绍了如何部署和使用`Ripple`。我更详细的研究了`Ripple`,并通过本文分享我的看法。

## 用法

### 1.支持基于GTID的复制
`Ripple`仅支持GTID模式，不支持文件和基于位点的复制。如果主节点以非GTID模式运行， 您会得到以下错误输出：
```
Failed to read packet: Got error reading packet from server: The replication sender thread cannot start in AUTO_POSITION mode: this server has GTID_MODE = OFF instead of ON.
```

您可以使用命令行选项`-ripple_server_id`和`-ripple_server_uuid`指明`ripple`节点的`Server_id`和`UUID`。

这两个参数都是可选参数，如果没有指定，`Ripple`使用默认的112211的`server_id`，uuid将自动生成。

### 2.使用复制用户和密码连接主节点
连接主节点时，您可以使用命令行选项`-ripple_master_user` 和 `-ripple_master_password`指定复制用户和密码。

### 3.连接`Ripple`节点
您可以使用命令行选项`-ripple_server_ports`和`-ripple_server_address`指定`Ripple`服务器的连接端点。确保使用`-ripple_server_address`指明`Ripple`服务的可访问的主机名或IP地址。否则，默认情况下，`Ripple`将绑定到本地主机，因此您无法远程连接。

### 4.从节点连接`Ripple`节点设置
您可以使用`CHANGE MASTER TO`命令连接`Ripple`节点与从节点。

为了确保`Ripple`能够验证用于连接它的密码，需要通过指定选项`-ripple_server_password_hash`启动`Ripple`。

例如，您可以使用以下命令启动`Ripple`节点：
```
rippled -ripple_datadir=./binlog_server -ripple_master_address= <master ip>  -ripple_master_port=3306 -ripple_master_user=repl -ripple_master_password='password' -ripple_server_ports=15000 -ripple_server_address='172.31.23.201' -ripple_server_password_hash='EF8C75CB6E99A0732D2DE207DAEF65D555BDFB8E'
```

您可以在从节点使用以下`CHANGE MASTER TO`命令连接：
```
CHANGE MASTER TO master_host='172.31.23.201', master_port=15000, master_password=’XpKWeZRNH5#satCI’, master_user=’rep’
```

请注意，为`Ripple`节点指定的密码哈希对应与`CHANGE MASTER TO`命令中使用的文本密码。目前，`Ripple`不基于用户名进行身份验证，只要密码匹配，就接收任何非空用户名。

### 5.`Ripple`节点管理
可以从任意标准MySQL客户端使用MySQL协议监控和管理`Ripple`节点。在[mysql-ripple](https://github.com/google/mysql-ripple/blob/master/mysql_slave_session.cc)GitHub页面的源代码中，您可以直接看到所支持的一组有限的命令。

以下是一些有用的命令：
```
SELECT @@global.gtid_executed; – 根据下载的二进制日志查看Ripple节点的GTID集
STOP SLAVE; – 断开Ripple节点与主节点的连接
START SLAVE; – Ripple节点与主节点连接
```

## 已知问题&改进建议
### 1.没有看到有选项可以启用`Ripple`节点到主节点的SSL复制
因此，`Ripple`节点无法连接到要求加密连接的主节点。试图连接将导致错误：
```
0322 09:01:36.555124 14942 mysql_master_session.cc:164] Failed to connected to host: <Hosname>, port: 3306, err: Failed to connect: Connections using insecure transport are prohibited while --require_secure_transport=ON.
```

### 2.无法启用`Ripple`节点的半同步选项
我使用选项`-ripple_semi_sync_slave_enabled=true`启动`Ripple`节点。

连接它时，主节点能够将`Ripple`节点检测为启用半同步的从节点。
```
mysql> show status like 'rpl%';
------------------------------------------------------
| Variable_name                              | Value |
------------------------------------------------------
| Rpl_semi_sync_master_clients               | 1     |
| Rpl_semi_sync_master_status                | ON    |
| Rpl_semi_sync_slave_status                 | OFF   |
------------------------------------------------------
```
然而，尝试在半同步模式下执行事务需要等待`rpl_semi_sync_master_timeout`，该值为180000。
```
mysql> create database d12;
Query OK, 1 row affected (3 min 0.01 sec)
```
我可以看到半同步在主节点被关闭：
```
mysql> show status like 'rpl%';
+--------------------------------------------+-------+
| Variable_name                              | Value |
+--------------------------------------------+-------+
| Rpl_semi_sync_master_clients               | 1     |
| Rpl_semi_sync_master_status                | OFF   |
| Rpl_semi_sync_slave_status                 | OFF   |
+--------------------------------------------+-------+
```
对应的MySQL错误日志片段：
```
2020-03-21T10:05:56.000612Z 52 [Note] Start binlog_dump to master_thread_id(52) slave_server(112211), pos(, 4)
2020-03-21T10:05:56.000627Z 52 [Note] Start semi-sync binlog_dump to slave (server_id: 112211), pos(, 4)
20020-03-21T10:08:55.873990Z 2 [Warning] Timeout waiting for reply of binlog (file: mysql-bin.000010, pos: 350), semi-sync up to file , position 4.
2020-03-21T10:08:55.874044Z 2 [Note] Semi-sync replication switched OFF.
```
在MySQL Ripple的GitHub页面中也有类似的问题。

### 3.`Ripple`节点的从节点使用并行复制遇到的问题
我经常遇到从节点的SQL线程因以下报错而中断：
```
Last_SQL_Error: Cannot execute the current event group in the parallel mode. Encountered event Gtid, relay-log name /mysql_data/relaylogs/relay-log.000005, position 27023962 which prevents execution of this event group in parallel mode. Reason: The master event is logically timestamped incorrectly.
```
通过对中继日志和上面的位置进行分析，可以发现此时的“序列号”被重置为1。我找到了原主节点发生二进制日志轮换的原因。通常，对直连从节点，会有一个rotate事件，由于这个事件，中继日志也会基于主节点二进制日志的rotation而rotate。我的猜测是，可以检测到这种情况，并且可以通过并行线程来处理序列号重置。但是，当序列号发生变化而中继日志没有rotate时，我们会看到并行线程失败。

这个issue描述了这个问题：[slave parallel thread failure while syncing from binlog server #26](https://github.com/google/mysql-ripple/issues/26)

### 4.mysqlbinlog程序不能用于`Ripple`节点生成的二进制日志
试图用mysqlbinlog解析二进制日志会抛出错误：
```
ERROR: Error in Log_event::read_log_event(): 'Found invalid event in binary log', data_len: 43, event_type: -106
```
这个问题在这里也被提及：[Not able to open the binary log files using mysqlbinlog utility. #25](https://github.com/google/mysql-ripple/issues/25)。

作者承认这是一个众所周知的问题。我觉得支持mysqlbinlog用于调试是很有用的。

## 总结
以上就是我的简单测试报告。在我更深入使用`Ripple`后，我会更新这篇博文。总的来说，我发现它使用起来简单明了，并且有潜力成为MySQL环境中binlog server的标准。
