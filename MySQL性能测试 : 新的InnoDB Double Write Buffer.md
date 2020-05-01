- [原文链接](http://dimitrik.free.fr/blog/posts/mysql-80-perf-new-dblwr.html)


# MySQL性能测试 : 新的InnoDB Double Write Buffer
新的MySQL8.0.20版本重新设计了InnoDB Double Write（DBLWR），确实是一个大的历史烦人的事情。为什么在过去这么痛苦，让我们付出了这么多精力，我无法更好地解释，因为从2018年开始，我已经在下面一篇关于MySQL基于IO负载的文章中说过了。这个故事并不完整，因为它缺少2019年的那一篇（稍后再讲），但是如果你（重新）读过上面的这篇文章提到的内容，您会更好理解接下来的内容。

但至少现在这篇文章是关于好消息的——新的DBLWR以及它如何帮助解决历史上MySQL性能问题！由于一张图片胜过百万字，我将尽量节省三百万字（因为本文中有三张图片）

我会跳过所有的新的设计细节（我认为Sunny会更好的第一手解释所有的）——我只会提到以下内容：
- DBLWR不再是“系统表空间”的一部分，可以被放置在任何地方（如果您有可能使用不同的存储存放DBLWR文件，您完全可以摆脱DBLWR对您主存储的影响），但默认情况下，DBLWR和您的数据存储在相同的目录下。
- 您可以配置想要使用多少个DBLWR文件
- 以及每个DBLWR文件有多少页（这也与最终的DBLWR文件大小直接相关）

>更多配置选项的详细内容，您可以参阅[ MySQL 8.0 doc about DBLWR ](https://dev.mysql.com/doc/refman/8.0/en/innodb-doublewrite-buffer.html)，这里做了详细的解释。

废话不多说，让我们看以下测试结果

## 测试负载场景
- 这里使用简单的Sysbench OLTP_RW测试用例
- 8张表，每张表五千万行数据（大约100G数据）
- InnoDB Buffer Pool大小从
	- 128G（纯内存负载）
	- 到64G （部分IO限制）
	- 到32G （重IO限制）
- 并发从1，2，4....1024
- 在MySQL8.0.19和8.0.20做相同测试

## 配置信息
- 服务器：48cores-HT Intel(R) Xeon(R) Platinum 8268 CPU @ 2.90GHz, 192GB RAM
- 存储： Intel Optane NVMe, XFS 文件系统
- OS：OL7.6, kernel-4.14.35-1902.301.1.el7uek.x86_64
- MySQL配置文件：完整的`my.cnf`在文章末尾
- 注意：DBLWR文件是默认配置，与DATA和REDO存储在同一个目录中（也是默认配置）

>下面几句话是关于下列图表的。——它们并不是真正的“用户友好”（因为营销人员会喜欢），它们只是反应了在不同增长的负载下获得的实际TPS水平。随着负载的增长，TPS也在增长（从左到右依次是1个并发、2个并发、4个并发，依次类推）——如果您有不清楚的地方，请随时联系我。

## 128G 缓冲池
![enter image description here](http://dimitrik.free.fr/blog/perf/8020_dblwr/img1.png)
- 如预期，只要MySQL实例没有二次写页的延时，DBLWR就不会影响内存负载。
- （这可以通过使用更快的存储或者更大的redo等来实现）。
- 刷页是在后台进行的，只要它的速度快到足以跟上您Redo写的速度，您就是安全的。

## 64G 缓冲池
![enter image description here](http://dimitrik.free.fr/blog/perf/8020_dblwr/img2.png)
- 然而，一旦您的负载受到IO限制，结果就变了...
- 任何页写入将延时您的IO读取！
- 旧的DBLWR设计不是为高并发而设计的。
- 但是，如您所见，在并发64之前，TPS仍然正常。

## 32G缓冲池
![enter image description here](http://dimitrik.free.fr/blog/perf/8020_dblwr/img3.png)
- 32G缓冲池大小仅可以缓存1/3的数据
- 有“统一”的访问模式，我们这里几乎不受IO限制
- 因此，对TPS总体影响很大
- 但是DBLWR的影响与之前的测试结果非常相似——并发能达到64，然后性能下降严重。

## 总结
- 到目前为止，一个历史上大的令人烦人的事情没有了。--归功于Sunny!!
- 但是注意，MySQL8.0是“连续发布”模式
- 这意味着更多有趣的东西将会到来
- 所以，请继续关注

感谢您使用MySQL!

## 附录

### my.cnf
```
[mysqld]

# general
 max_connections=4000
 back_log=4000
 ssl=0
 table_open_cache=8000
 table_open_cache_instances=16
 default_authentication_plugin=mysql_native_password
 default_password_lifetime=0
 max_prepared_stmt_count=512000
 skip_log_bin=1
 character_set_server=latin1
 collation_server=latin1_swedish_ci
 skip-character-set-client-handshake
 transaction_isolation=REPEATABLE-READ

# files
 innodb_file_per_table
 innodb_log_file_size=1024M
 innodb_log_files_in_group=16
 innodb_open_files=4000

# buffers
 innodb_buffer_pool_size=128000M / 64000M / 32000M
 innodb_buffer_pool_instances=24
 innodb_log_buffer_size=64M

# tune
 innodb_doublewrite=1
 innodb_thread_concurrency=0
 innodb_flush_log_at_trx_commit=1
 innodb_max_dirty_pages_pct=90
 innodb_max_dirty_pages_pct_lwm=10

 join_buffer_size=32K
 sort_buffer_size=32K
 innodb_use_native_aio=1
 innodb_stats_persistent=1
 innodb_spin_wait_delay=6

 innodb_max_purge_lag_delay=300000
 innodb_max_purge_lag=0
 innodb_flush_method=O_DIRECT
 innodb_checksum_algorithm=crc32
 innodb_io_capacity=20000
 innodb_io_capacity_max=40000
 innodb_lru_scan_depth=1000
 innodb_change_buffering=none
 innodb_read_only=0
 innodb_page_cleaners=24
 innodb_undo_log_truncate=off

# perf special
 innodb_adaptive_flushing=1
 innodb_flush_neighbors=0
 innodb_read_io_threads=16
 innodb_write_io_threads=4
 innodb_purge_threads=4
 innodb_adaptive_hash_index=0

# monitoring
 innodb_monitor_enable='%'
 performance_schema=ON

# etc.
 loose_log_error_verbosity=3
 secure_file_priv=
 core_file
 innodb_buffer_pool_in_core_file=off
```

### 15.6.4 Doublewrite Buffer
 Doublewrite buffer是一个存储，InnoDB将页写入InnoDB数据文件适当位置之前，会将缓冲池中页刷新到该存储中。如果操作系统，存储子系统，或者`mysqld`进程在页写入中途崩溃，InnoDB可以在崩溃恢复中从doublewrite buffer中找到一份好的备份。

虽然数据写了2次，doublewrite buffer不会需要2倍的IO负载和2倍的IO操作。数据将以一个大的连续块写入到doublewrite buffer中，操作系统单次调用`fsync()`（除非`innodb_flush_method`被设置为`O_DIRECT_NO_FSYNC`）。

MySQL8.0.20之前，doublewrite buffer存储在InnoDB 系统表空间中。从MySQL8.0.20开始，doublewrite buffer存储在双写文件中。


doublewrite buffer配置提供以下参数：

- `innodb_doublewrite`
 `innodb_doublewrite`参数控制是否启用doublewrite buffer。多数场景下默认是启用的。为了禁用doublewrite buffer，设置`innodb_doublewrite=0`或者启动MySQL服务时加`--skip-innodb-doublewrite`选项。例如在性能压测的场景下，如果您更关注性能而不是数据可靠性，您可以禁用doublewrite buffer。

	如果doublewrite buffer位于支持原子写的Fusion-io设备上，则自动禁用doublewrite buffer，并使用Fusion-io原子写来执行数据文件写。但是，要注意`innodb_doublewrite`设置是全局的。当doublewrite buffer被禁用时，它对不在Fusion-io硬件上的数据文件也禁用。该功能仅在Fusion-io硬件上支持，在Linux中仅支持Fusion-io NVMFS。为了充分利用这个功能，建议将`innodb_flush_method=O_DIRECT`

- `innodb_doublewrite_dir`
	`innodb_doublewrite_dir`（8.0.20引入）定义了InnoDB创建双写文件的目录。如果目录没有指定，双写文件创建在` innodb_data_home_dir`目录下，没有指定默认在数据目录下。
	
	哈希符'#'会自动创建在指定目录名前缀，避免与shema名冲突。然而，如果使用了'.', '#'. 或者'/'指定了目录前缀，则不在目录名前缀没有哈希符'#'。

	理想情况下，双写目录应该放在最快的存储上。

- `innodb_doublewrite_files`
	`innodb_doublewrite_files`参数定义了双写文件的数量。默认情况下，每个缓冲池实例都会创建2个双写文件：一个刷新列表双写文件和一个LRU列表双写文件。
	
	刷新列表双写文件用于从缓冲池刷新列表中刷新页。刷新列表双写文件默认大小是` InnoDB page size * doublewrite page bytes`.
	
	LRU列表双写文件是用于刷新从缓冲池LRU列表的页。它也包括单个页刷新的槽。LRU列表双写文件默认大小为`InnoDB page size * (doublewrite pages + (512 / the number of buffer pool instances))`，512是为单个页刷新保留的槽的总数。
	
	至少有2个双写文件。双写文件的最大数量是缓冲池实例的两倍。（缓冲池实例的数量由参数`innodb_buffer_pool_instances`控制）

	双写文件有以下格式：`#ib_page_size_file_number.dblwr`。例如，下面的双写文件是在一个InnoDB页大小为16KB，单个缓冲池的MySQL实例上创建：
	```
	#ib_16384_0.dblwr
	#ib_16384_1.dblwr
	```
	
	`innodb_doublewrite_files`参数用于高级性能调优。默认设定已经适用于大多数用户。

- `innodb_doublewrite_pages`
	`innodb_doublewrite_pages`参数（MySQL8.0.20引入）控制每个线程双写页的最大数量。如果这个值没有指定，`innodb_doublewrite_pages`设置为`innodb_write_io_threads`值。这个参数用于高级性能调优。默认值已经适用于大多数用户。

- `innodb_doublewrite_batch_size`
	`innodb_doublewrite_batch_size`参数（MySQL8.0.20引入）控制一批写入双写页的数量。这个参数用于高级性能调优。默认值已经适用于大多数用户。
