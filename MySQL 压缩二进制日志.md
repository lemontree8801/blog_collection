- [原文链接](https://mysql.wisborg.dk/2020/05/07/mysql-compressed-binary-logs/)

[TOC]
# MySQL 压缩二进制日志
在繁忙的服务器上，二进制日志最终可能成为磁盘空间使用量最大的来源之一。这意味着更高的I/O，更大的备份（您备份了您的二进制日志，是吗？），从节点获取日志时可能会有更多的网络流量，等等。通常，二进制日志压缩效果很好，所以人们一直希望有一个功能可以在MySQL使用二进制日志时对其进行压缩。从MySQL8.0.20开始，现在可以了。我将在这篇博文中看看这个新功能。

## 配置
二进制日志压缩功能由2个参数控制，一个用于启用压缩，一个用于指定压缩级别。总结如下表所示。
|                  参数名                   | 默认值 | 允许设置范围 |
| :---------------------------------------: | :----: | :----------: |
|      binlog_transaction_compression       |  OFF   |    OFF/ON    |
| binlog_transaction_compression_level_zstd |   3    |     1-22     |

这两个参数名称很容易解释。`binlog_transaction_compression`指定是否启用压缩，`binlog_transaction_compression_level_zstd`指定压缩级别。在较高的级别上，以增加CPU使用的代价实现更好的压缩（原则上来说——参见稍后博文中的测试）。

这两个参数都可以在全局和会话的范围内动态设置。但是，不允许在事务执行过程中更改会话值。如果您这样做，您会得到以下报错：
```
mysql> SET SESSION binlog_transaction_compression = ON;
ERROR: 1766 (HY000): The system variable binlog_transaction_compression cannot be set when there is an ongoing transaction.
```
尽管配置二进制日志压缩很容易，但是您应该知道有一些限制。

## 限制
限制简单归结为只压缩事务行事件。不压缩基于语句的事件、GTID信息、循环事件、非事务表的基于行事件等。另外，如果您有一个事务中包含非事务性更改，那么这两个更改都不会被压缩。

如果您使用二进制日志的所有默认值，并使用InnoDB存储引擎（默认），压缩将起作用。有关限制的完整列表，请参阅手册中的[二进制日志事务压缩](https://dev.mysql.com/doc/refman/8.0/en/binary-log-transaction-compression.html)。
>这也意味着每个事务都是独立压缩的。请参阅博客后面的示例理解其中含义。

正如我经常说的，监控是了解系统的关键。怎样从监控中查看二进制日志压缩功能？

## 监控
有二个方法可以监控二进制日志压缩功能的性能。一个Performance Schema中的压缩统计表和新的阶段事件。

Performance Schema中的`binary_log_transaction_compression_stats`表包含了从上次MySQL重启（或者上次表被截断）的压缩的统计信息。对二进制日志，这张表有两行，一行记录压缩事件，一行记录未压缩事件。从节点对中继日志也类似地记录两行数据。记录二进制日志的示例如下：
```
mysql> SELECT * FROM binary_log_transaction_compression_stats\G
*************************** 1. row ***************************
                            LOG_TYPE: BINARY
                    COMPRESSION_TYPE: ZSTD
                 TRANSACTION_COUNTER: 15321
            COMPRESSED_BYTES_COUNTER: 102796461
          UNCOMPRESSED_BYTES_COUNTER: 252705572
              COMPRESSION_PERCENTAGE: 59
                FIRST_TRANSACTION_ID: 74470a0c-8ea4-11ea-966e-080027effed8:30730
  FIRST_TRANSACTION_COMPRESSED_BYTES: 313
FIRST_TRANSACTION_UNCOMPRESSED_BYTES: 363
         FIRST_TRANSACTION_TIMESTAMP: 2020-05-07 19:26:37.744437
                 LAST_TRANSACTION_ID: 74470a0c-8ea4-11ea-966e-080027effed8:46058
   LAST_TRANSACTION_COMPRESSED_BYTES: 712
 LAST_TRANSACTION_UNCOMPRESSED_BYTES: 1627
          LAST_TRANSACTION_TIMESTAMP: 2020-05-07 19:38:14.149782
*************************** 2. row ***************************
                            LOG_TYPE: BINARY
                    COMPRESSION_TYPE: NONE
                 TRANSACTION_COUNTER: 20
            COMPRESSED_BYTES_COUNTER: 5351
          UNCOMPRESSED_BYTES_COUNTER: 5351
              COMPRESSION_PERCENTAGE: 0
                FIRST_TRANSACTION_ID: 74470a0c-8ea4-11ea-966e-080027effed8:30718
  FIRST_TRANSACTION_COMPRESSED_BYTES: 116
FIRST_TRANSACTION_UNCOMPRESSED_BYTES: 116
         FIRST_TRANSACTION_TIMESTAMP: 2020-05-07 19:26:37.508155
                 LAST_TRANSACTION_ID: 74470a0c-8ea4-11ea-966e-080027effed8:31058
   LAST_TRANSACTION_COMPRESSED_BYTES: 116
 LAST_TRANSACTION_UNCOMPRESSED_BYTES: 116
          LAST_TRANSACTION_TIMESTAMP: 2020-05-07 19:30:30.840767
2 rows in set (0.0026 sec)
 
mysql> SHOW BINARY LOGS;
+---------------+-----------+-----------+
| Log_name      | File_size | Encrypted |
+---------------+-----------+-----------+
| binlog.000142 |       240 | No        |
| binlog.000143 |      4933 | No        |
| binlog.000144 |  28238118 | No        |
| binlog.000145 |  24667167 | No        |
| binlog.000146 |  39221771 | No        |
| binlog.000147 |  11944631 | No        |
| binlog.000148 |       196 | No        |
+---------------+-----------+-----------+
7 rows in set (0.0005 sec)
```
该示例启用了GTID，除了`binlog.000142`所有二进制日志均是在重启之后创建。

这表明有15341个事务（74470a0c-8ea4-11ea-966e-080027effed8:30718-46058），其中15321个已被压缩，平均压缩比为59%（将252,705,572字节压缩至102,796,461字节）。还有第一个和最近一次被压缩的事务的统计信息。类似的，有20个“事务”不能被压缩。

在知道压缩率的同时，您还需要知道执行压缩和解压缩的开销。可以使用Performance Schema中的两个阶段事件进行监控：
- stage/sql/Compressing transaction changes.
- stage/sql/Decompressing transaction changes.
（句号是名字的一部分。）

默认情况下，这两个事件的instrument均未启用，要收集信息，您还需要启用`events_stages_current`的consumer。例如，在配置文件中启用：
```
[mysqld]
performance-schema-instrument = "stage/sql/%Compressing transaction changes.=ON"
performance-schema-consumer-events-stages-current = ON
```
>启用阶段检测会产生一些开销。如果您考虑在生产系统上启用这些功能，您需要先测试影响。

您可以将其与`wait/io/file/sql/binlog`事件（默认启用）进行比较，后者是执行I/O所花费时间。例如：
```
mysql> SELECT EVENT_NAME, COUNT_STAR,
              FORMAT_PICO_TIME(SUM_TIMER_WAIT) AS total_latency,
              FORMAT_PICO_TIME(MIN_TIMER_WAIT) AS min_latency,
              FORMAT_PICO_TIME(AVG_TIMER_WAIT) AS avg_latency,
              FORMAT_PICO_TIME(MAX_TIMER_WAIT) AS max_latency
         FROM performance_schema.events_stages_summary_global_by_event_name
        WHERE EVENT_NAME LIKE 'stage/sql/%transaction changes.'\G
*************************** 1. row ***************************
   EVENT_NAME: stage/sql/Compressing transaction changes.
   COUNT_STAR: 15321
total_latency: 6.10 s
  min_latency: 22.00 ns
  avg_latency: 397.96 us
  max_latency: 982.12 ms
*************************** 2. row ***************************
   EVENT_NAME: stage/sql/Decompressing transaction changes.
   COUNT_STAR: 0
total_latency:   0 ps
  min_latency:   0 ps
  avg_latency:   0 ps
  max_latency:   0 ps
2 rows in set (0.0008 sec)
 
mysql> SELECT *
         FROM sys.io_global_by_wait_by_latency
        WHERE event_name = 'sql/binlog'\G
*************************** 1. row ***************************
   event_name: sql/binlog
        total: 27537
total_latency: 4.83 min
  avg_latency: 10.51 ms
  max_latency: 723.92 ms
 read_latency: 138.25 us
write_latency: 290.69 ms
 misc_latency: 4.82 min
   count_read: 37
   total_read: 1002 bytes
     avg_read:   27 bytes
  count_write: 16489
total_written: 99.26 MiB
  avg_written: 6.16 KiB
1 row in set (0.0015 sec)
```
这个例子中，我使用`sys.io_global_by_wait_by_latency`视图，因为它自动让延时变得易读。对压缩/解压缩阶段的延迟，`FORMAT_PICO_TIME()`函数对其进行转换。

这个例子中，MySQL花费了6.21秒来压缩二进制日志，每个事务平均不到400微秒。相比，二进制日志文件执行I/O花费了4.8分钟。

在启用压缩前，应检查写入和读取二进制日志文件花费的时间，以便确定性能变化。您还应该检查CPU使用变化。

上述输出，它显示压缩比为59%，但是对不同类型的负载呢？

## 不同负载示例
为确定压缩的效果，我执行了一系列任务，并比较了压缩和不压缩时二进制日志的大小。为了进行比较，我还手工压缩系列测试的中未压缩的二进制日志，以查看最佳压缩率（与MySQL使用的每次事务压缩不同）。除了给定测试所需的设置外，测试都是使用默认配置执行的。

已测试一下负载：
- 批量插入：加载` employees`示例数据库。
- 批量更新：更新了`employees.salaries`表中所有行的`salary`列：` UPDATE employees.salaries SET salary = salary + 1`。
- 批量加载：用sysbench为`oltp_read_write`基准测试做准备，每个表加载100000行数据。
- OLTP负载：用sysbench执行`oltp_read_write`基准测试，`--events=15000`，使用先前测试的四张表。
- 单行删除：删除sysbench测试的一张表的100000行数据。这些行被逐个删除，这代表压缩最坏情况，因为事务非常小，每个删除的行的二进制日志中只有前一个映像。

单行删除可以使用MySQL Shell轻松执行，例如使用Python：
```
from datetime import datetime
 
for i in range(100000):
    if i % 5000 == 0:
        print('{0}: i = {1}'.format(datetime.now(), i))
    session.run_sql('DELETE FROM sbtest.sbtest1 WHERE id = {0}'.format(i+1))
print('{0}: i = 1000000'.format(datetime.now()))
```
每个任务生成25M-82M的未压缩二进制日志。

测试采用以下设定：
- 不压缩
- 启用压缩
- 加密但是不压缩
- 加密并启用压缩
- MySQL中不压缩+用`zstd`压缩

由于MySQL使用的是 [Zstandard 压缩算法](https://en.wikipedia.org/wiki/Zstandard)，所以选择`zstd`进行压缩。如果您想自己尝试，您可以从Facebook's Github仓库下载Zstandard源码，其中包括编译说明。

下表列出了每种组合的二进制日志的字节大小。
|     测试     |   正常   |   压缩   |   加密   | 加密+压缩 |   zstd   |
| :----------: | :------: | :------: | :------: | :-------: | :------: |
|  加载雇员库  | 66378639 | 28237933 | 66379151 | 28238634  | 26892387 |
|   批量更新   | 85698320 | 24667051 | 85698832 | 24667677  | 24779953 |
| sysbench准备 | 76367740 | 39221052 | 76368252 | 39221806  | 39775067 |
| sysbench执行 | 26190200 | 11933713 | 26190712 | 11936561  | 8468625  |
|   单行删除   | 47300156 | 39492400 | 47300668 | 39637684  | 16525219 |

加密的二进制日志压缩和未加密的二进制日志表明压缩是在加密之前完成的。（加密的数据不能很好地压缩。）因为是否启用加密没有区别，所以只会进一步讨论正常的（未压缩的）、压缩的和`zstd`结果。二进制日志的大小可以在图中看到：

![enter image description here](https://i2.wp.com/mysql.wisborg.dk/wp-content/uploads/2020/05/binlog_compress.png?w=636&ssl=1)

不同压缩方案的二进制日志大小。

毫不意外地，大批量加载和更新是压缩二进制日志最佳效果，分别是未压缩大小的29%和51%。Sysbench的OLTP负载也可以压缩到正常的46%。同样不奇怪的是，压缩的二进制日志大小是未压缩二进制日志的83%，所以单行删除的压缩效果几乎没有那么好。

当比较MySQL压缩的二进制日志和使用`zstd`手工压缩的二进制日志时，批量负载的文件大小大致相同，这也反映出对于大事务，按每个事务进行压缩等同于压缩整个文件。当事务大小变小时，每个事务压缩的相对效率会降低，这对单行删除尤其明显。

另一个要考虑的因素是压缩级别。

## 举例-压缩级别
在压缩级别上有一些奇怪的地方，简单来说就是不需要更改设置。

第一个奇怪的地方是，允许的值是1-22，但是`zstd`只支持1-19。MySQL文档中没有解释两者区别。

第二个奇怪的地方是，通过改变`binlog_transaction_compression_level_zstd`的值，压缩的二进制日志的大小实际上没有变化，从表中可以看出：
| Level/Test |  MySQL   |    -     |   zstd   |    -    |
| :--------: | :------: | :------: | :------: | :-----: |
|     -      |   加载   |   OLTP   |   加载   |  OLTP   |
|     1      | 28238142 | 11935450 | 34483207 | 8531545 |
|     3      | 28237933 | 11933713 | 26892387 | 8468625 |
|     11     | 28238128 | 11902669 | 24737194 | 6639524 |
|     19     | 28238246 | 11937664 | 18867187 | 5724300 |
|     22     | 28238125 | 11910022 |          |         |

分别使用压缩级别1、3、11、19和包括使用`zstd`手工压缩二进制日志。“加载”用于加载雇员数据库，OLTP是sysbench做`oltp_read_write`基准压测。数据也可以在图中看到：

![enter image description here](https://i2.wp.com/mysql.wisborg.dk/wp-content/uploads/2020/05/binlog_compress_levels.png?w=636&ssl=1)

二进制日志大小与压缩级别的关系

可以看出，无论MySQL中使用的压缩级别如何，文件大小基本上没有差异，而对于`zstd`，随着压缩级别的增加，文件大小如预期一样减小。对于级别1的加载测试，MySQL压缩效果甚至比`zstd`压缩效果好。就像从未在MySQL中设置压缩级别。
>一种可能的解释是，Zstandard支持针对给定数据类型（创建字典）训练算法。这特别有助于改进小数据的压缩。我不知道MySQL是否使用字典，如果使用字典，是否所有的压缩级别都大致相同。

## 总结
新的二进制日志事务压缩非常有效，可以很好减少I/O，磁盘使用量和网络使用量。您应该启用它吗？

除非CPU非常紧张，否则应该启用二进制日志压缩。从这些测试中可以看出，二进制日志占用磁盘空间可以大致减少一半，但与往常一样，它依赖于负载，您应该针对负载进行测试。
>您可以在协议级别启用压缩传送二进制日志。没有理由同时启用二进制日志事务压缩和协议压缩。

另一方面，目前没有理由改变压缩等级。
