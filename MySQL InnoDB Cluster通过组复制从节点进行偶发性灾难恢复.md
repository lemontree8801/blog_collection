- [原文链接](https://mysqlmed.wordpress.com/2020/06/11/mysql-innodb-cluster-disaster-recovery-contingency-via-a-group-replication-slave/)


# MySQL InnoDB Cluster通过组复制从节点进行偶发性灾难恢复

最近，有人让我研究一下InnoDB Cluster的灾难恢复应该如何做。

如果您正在阅读本文，那么我假设您熟悉什么是MySQL InnoDB Cluster，以及它是如何配置的，包含哪些组件等。

提示：8.0版本中的InnoDB Cluster（组复制、Shell&Router）相比5.7版本有了很大的改进。请尝试一下。

因此，既然我们要考虑如何最好地满足需求，即为我们的InnoDB Cluster创建一个灾备，让我们开始吧。

基本上，拓扑如下：
![enter image description here](https://mysqlmed.files.wordpress.com/2020/06/idc_gr_arch.png)
InnoDB Cluster主站点与一个组复制灾备恢复站点

生活已足够艰辛，所以我们要尽可能自动化，所以，InnoDB Cluster完成了其中的一些工作，但是还有部分工作需要人工处理，所以我们理想场景是：

**主站点：**
- InnoDB Cluster*3
- Router
- 访问主（动态）&灾备（静态）
- 全部自动化

**灾备站点：**
- Group Replication*2
- Router
- 访问主（动态）&灾备（静态）
- 从主节点配置的异步复制

**额外的/LBR 第三站点：**
- MySQL Router（静态）路由链接到主节点路由
- 允许主站点2个实例自动故障转移
- 一旦主站点确认中断或者使用不同的路由端口人工重新路由到灾备站点
- 建议冗余以减少SPOF。

## 让我们开始吧
首先，如果您不确定您在做什么，我建议您查看以下内容：

https://thesubtlepath.com/mysql/innodb-cluster-managing-async-integration/

https://scriptingmysql.wordpress.com/2019/03/29/replicating-data-between-two-mysql-group-replication-sets-using-regular-asynchronous-replication-with-global-transaction-identifiers-gtids

### 主节点
- 三台CentOS7服务器
- 8.0.20：Server，Shell&Router

所有服务器都检查并验证了它们的SELinux、防火墙、端口、/etc/hosts等等。出于显而易见的原因，我们希望在所有主机之间使用私有IP。

我从https://edelivery.oracle.com 下载了所有MySQL企业版rpm包并在3个节点执行以下命令（在这里分别称为："centos1"、"centos2"、"centos3"）：
```
sudo yum install -y mysql-*8.0.20*rpm
sudo systemctl start mysqld.service
sudo systemctl enable mysqld.service
sudo grep 'A temporary password is generated for root@localhost' /var/log/mysqld.log |tail -1
mysql -uroot -p
```

一旦进入，我们想要控制产生二进制日志的命令，"alter user"和"create user"会在后面引起问题，因此这里设置了"sql_log_bin=OFF"：
```
SET sql_log_bin = OFF;
alter user 'root'@'localhost' identified by 'passwd';
create user 'ic'@'%' identified by 'passwd';
grant all on . to 'ic'@'%' with grant option;
flush privileges;
SET sql_log_bin = ON;
```

这个"ic@%"用户将在整个过程中使用，因为我们不知道何时以及哪个实例最终成为主实例或克隆实例，所以通过使用此单用户设置，我可以自己更轻松地进行工作。对于您的生产设置，请更深入地研究特定的权限。
```
mysqlsh --uri root @ localhost：3306
dba.checkInstanceConfiguration（'ic @ centos01：3306'）
dba.configureInstance（'ic @ centos01：3306'）;
```
重启和改变都输入"Y"。

**在服务器中的一个节点（这将成为单主实例）**
```
\ connect ic @ centos01：3306
cluster = dba.createCluster（“ mycluster”）
cluster.status（）
cluster.addInstance（“ ic @ centos02：3306”）
cluster.addInstance（“ ic @ centos03：3306”）
cluster.status（）;
```
现在，您将看到，我们已经安装了二进制文件并启动了centos2和centos3上的实例，"addInstance"克隆了我们的主节点。这让一切变得简单很多。

提示：在克隆之前，如果您需要之前的数据，请在此之前处理，我们会删除数据。

现在我们有一个3实例组成的单主节点InnoDB Cluster。

**设置本地"InnoDB Cluster感知"Router**：
```
mkdir -p /opt/mysql/myrouter
chown -R mysql:mysql /opt/mysql/myrouter
cd /opt/mysql
mysqlrouter --bootstrap ic@centos02:3306 -d /opt/mysql/myrouter -u mysql
./myrouter/start.sh
```
现在Router已经启动并运行。

### 组复制的灾备站点
这里我做了一个错误的示范，只设置了一个两节点的组复制。如果您想知道您应该如何做，请参阅：
https://scriptingmysql.wordpress.com/2019/03/28/mysql-8-0-group-replication-three-server-installation/

请再次检查端口、防火墙、SELinux、/etc/hosts。

环境：
- 2台Oracle Linux服务器。
- 8.0.20版本：MySQL Server、Router&Shell

在2个从节点服务器上执行以下命令（olslave01&olslave02）：
```
sudo yum install -y mysql-8.0.20rpm
sudo systemctl start mysqld.service
sudo systemctl enable mysqld.service
sudo grep 'A temporary password is generated for root@localhost' /var/log/mysqld.log |tail -1
mysql -uroot -p
SET sql_log_bin = OFF;
alter user 'root'@'localhost' identified by 'passwd';
create user 'ic'@'%' identified by 'passwd';
grant all on . to 'ic'@'%' with grant option;
flush privileges;
SET sql_log_bin = ON;
```
现在我们已经创建了基础服务，我们需要从InnoDB Cluster中的主实例上克隆数据。

这意味着需要安装[mysql clone](https://mysqlserverteam.com/clone-create-mysql-instance-replica/)插件。

**在集群主节点**：
```
SET sql_log_bin = OFF;
INSTALL PLUGIN CLONE SONAME "mysql_clone.so";
GRANT BACKUP_ADMIN ON . to 'ic'@'%';
GRANT SELECT ON performance_schema.* TO 'ic'@'%';
GRANT EXECUTE ON . to 'ic'@'%';
SET sql_log_bin = ON;
```
在每个从节点：
```
INSTALL PLUGIN CLONE SONAME "mysql_clone.so";
INSTALL PLUGIN group_replication SONAME 'group_replication.so';
SET GLOBAL clone_valid_donor_list = 'centos02:3306';
GRANT CLONE_ADMIN ON . to 'ic'@'%';
# Keep this, if we ever want to clone from any of the slaves
GRANT BACKUP_ADMIN ON . to 'ic'@'%';
GRANT SELECT ON performance_schema.* TO 'ic'@'%';
GRANT EXECUTE ON . to 'ic'@'%';
```
执行克隆操作：
```
set global log_error_verbosity=3;
CLONE INSTANCE FROM 'ic'@'centos02':3306 IDENTIFIED BY 'passwd';
```
查看克隆过程的进展情况：
```
select STATE, ERROR_NO, BINLOG_FILE, BINLOG_POSITION, GTID_EXECUTED,
CAST(BEGIN_TIME AS DATETIME) as "START TIME",
CAST(END_TIME AS DATETIME) as "FINISH TIME",
sys.format_time(POWER(10,12) * (UNIX_TIMESTAMP(END_TIME) - UNIX_TIMESTAMP(BEGIN_TIME))) as DURATION
from performance_schema.clone_status \G
```
我们克隆的过程中，我们可能会遇到重复UUID的问题，因此在每个从节点，强制服务端有一个新的UUID：
```
rm /var/lib/mysql/auto.cnf
systemctl restart mysqld
```
**设置组复制配置**
节点1:
```
vi /etc/my.cnf
# GR setup
server-id =11
log-bin =mysql-bin
gtid-mode =ON
enforce-gtid-consistency =TRUE
log_slave_updates =ON
binlog_checksum =NONE
master_info_repository =TABLE
relay_log_info_repository =TABLE
transaction_write_set_extraction=XXHASH64

plugin_load_add ="group_replication.so"
group_replication = FORCE_PLUS_PERMANENT
group_replication_bootstrap_group = OFF
#group_replication_start_on_boot = ON
group_replication_group_name = 8E2F4761-C55C-422F-8684-D086F6A1DB0E
group_replication_local_address = '10.0.0.41:33061'
# Adjust the following according to IP's and numbers of hosts in group:
group_replication_group_seeds = '10.0.0.41:33061,10.0.0.42:33061'
```
在节点2：
```
server-id =22
log-bin =mysql-bin
gtid-mode =ON
enforce-gtid-consistency =TRUE
log_slave_updates =ON
binlog_checksum =NONE
master_info_repository =TABLE
relay_log_info_repository =TABLE
transaction_write_set_extraction=XXHASH64

plugin_load_add ="group_replication.so"
group_replication = FORCE_PLUS_PERMANENT
group_replication_bootstrap_group = OFF
#group_replication_start_on_boot = ON
group_replication_group_name = 8E2F4761-C55C-422F-8684-D086F6A1DB0E
group_replication_local_address = '10.0.0.42:33061'
# Adjust the following according to IP's and numbers of hosts in group:
group_replication_group_seeds = '10.0.0.41:33061,10.0.0.42:33061'
```
重启两个服务端：
```
systemctl restart mysqld
```
检查它们在一个组复制组并且插件正常：
```
mysql -uroot

SELECT * FROM performance_schema.replication_group_members;
SELECT * FROM performance_schema.replication_group_members\G

select * from information_schema.plugins where PLUGIN_NAME = 'group_replication'\G
```
现在要在所有服务器上创建恢复复制通道（尽管这是为单主节点设置的，主节点可能降级，作为一个只读从节点，所以我们需要设置这个）：
```
CHANGE MASTER TO MASTER_USER='ic', MASTER_PASSWORD='passwd' FOR CHANNEL 'group_replication_recovery';
```
在节点1:
```
SET GLOBAL group_replication_bootstrap_group=ON;
START GROUP_REPLICATION;
SET GLOBAL group_replication_bootstrap_group=OFF;
```
在节点2:
```
START GROUP_REPLICATION;
```
检查所有的服务端的`super_read_only`，并检查它们是否在一个组中：
```
select @@super_read_only;
SELECT * FROM performance_schema.replication_group_members;
```
在从节点中的一个上针对InnoDB Cluster主实例初始化Router：
```
mkdir -p /opt/mysql/myrouter
chown -R mysql:mysql /opt/mysql/myrouter
mysqlrouter --bootstrap ic@centos02:3306 -d /opt/mysql/myrouter -u mysql
cd /opt/mysql/myrouter
./start.sh
```
检测连通性：
```
mysql -uic -P6446 -h olslave01 -N -e "select @@hostname, @@port"
mysql -uic -P6446 -h centos02 -N -e "select @@hostname, @@port"
```
## 从节点的复制配置
从主节点为从服务器设置复制意味着，当主节点在组复制设置中失败时，我们必须准备以下脚本，或者保留以下命令，因为它不是自动的，所以我们需要自己控制它。

另外，尽管是一个组复制组，当主服务器在灾备节点出现故障时，我们还需要检查到组复制的灾备节点的所有连接和会话，以确保能够安全地设置该节点。

所以仅在主实例上，执行以下命令：
```
CHANGE MASTER TO
MASTER_HOST = 'olslave01',
MASTER_PORT = 6446,
MASTER_USER = 'ic',
MASTER_PASSWORD = 'passwd',
MASTER_AUTO_POSITION = 1
FOR CHANNEL 'idc_gr_replication' ;
```
如您所见，这是通过启动InnoDB Cluster的组复制灾备站点Router进行复制的。因此，当集群中主节点迁移时，Router将路由到新的主节点，而不必重新创建复制或类似的操作。
```
start slave ;
```
InnoDB Cluster主站点将数据复制到我们的灾备站点：
```
show slave status for channel 'idc_gr_replication'\G
```

## 应用连接
现在我们有了数据高可用方案，如何连接其中一个站点和另一个站点呢？

我们有了2个router，每个对应本地的存储数据站点。但是，现实情况是，我们有一个数据入口点，即centos01上的InnoDB Cluster主实例，以及4个数据副本，它们在一个本地网络（或者专线），以确保我们尽可能地安全。

您可能已经注意到，在初始化两边的Router时，它选择默认端口的6446，路由策略是`first_available`。这个端口可以根据需要进行调整。

但是，这取决于应用程序的位置，以及用户入口点，您有一个VIP、浮动IP或者类似的架构来实现负载均衡。开始问更多的问题：

**当主站点宕机会发生什么？**
如：
![enter image description here](https://mysqlmed.files.wordpress.com/2020/06/idc_gr_arch_fail.png)

好吧，有很多事情需要注意。本地的Router将负责处理故障，在这里，我有必要预先警告您，当主站点上3个实例中有2个实例丢失时会发生什么。这里，最后一个实例可以配置为`group_replication_exit_state_action='OFFLINE_MODE'`或者`ABORT`。这样，它不会通过Router作为一个有效节点使用。

现在，一旦发生了这种情况，即主站点宕机，我们需要手工干预，确保所有实例都已宕机，并启动灾备站点。现在主站点和灾备站点上引导到InnoDB Cluster的Router肯定会失败。

我们只剩下灾备站点中的组复制。

所以这里我们应该如何做？创建一个额外的Router，配置一个静态（非动态）设置，因为我们没有通过`mysql_innodb_cluster_metadata`schema中获得的IDC感知的Router的元数据，该元数据存储在`dynamic_state=/opt/mysql/myrouter/data/state.json`。

我所做的是在其他服务器上创建一个新的Router进程（instead of centos02 & olslave01, on centos01 & olslave02）。这并不适用于生产环境，因为我们希望Router更靠近应用，而不是MySQL实例。因此，实际上您可以使用一个新路径。
```
mkdir -p /opt/mysql/myrouter7001
chown -R mysql:mysql /opt/mysql/myrouter7001
cd /opt/mysql/myrouter7001
vi mysqlrouter.conf
[routing:DR_rw]
bind_address=0.0.0.0
bind_port=7001
destinations=olslave01:3306,olslave02:3306
routing_strategy=first-available
protocol=classic
```
配置完成后，启动Router
```
mysqlrouter --user=mysql --config /opt/mysql/myrouter7001/mysqlrouter.conf &
```
测试连通性：
```
mysql -uic -P7001 -h olslave02 -N -e "select @@hostname, @@port"
mysql -uic -P7001 -h centos01 -N -e "select @@hostname, @@port"
```
这个会返回olslave01，当失败时，olslave02。

当侦听6446的InnoDB Cluster引导的Router失败时，我们需要将所有请求重定向到生产IP上的端口7001（在我的例子中，是olslave02和centos01（表示这里非常奇怪，centos01不是宕机了？？））。

总结一下，我们现在有4个router在运行，2个是InnoDB Cluster感知Router，监听6446，而静态组复制灾备Router监听7001。

有很多的端口和很长的连接串。我们是否可以将其简化？

我添加了另一个Router进程，高级别的。这个Router进程，有一个硬编码的静态配置，因为我们没有两端的InnoDB Cluster感知的Router，即没有两端的`mysql_innodb_cluster_metadata`schema。

所以在我的案例中，我创建了另一个路由进程配置在centos03上（？？？centos03不是已经不可用了？？）
```
mkdir -p /opt/mysql/myrouter
chown -R mysql:mysql /opt/mysql/myrouter
cd /opt/mysql/myrouter
```
（您可能想用一些不同的方法，但是我这么做了。我可以让所有这些服务在同一个服务器上，使用不同路径。在同一台服务器上运行的Router数量没有任何限制。更多的是“端口可用性”，或者应该说是“网络安全性”问题）
```
vi mysqlrouter.conf:
[routing:Master_rw]
bind_address=0.0.0.0
bind_port=8008
destinations=centos02:6446,olslave01:6446
routing_strategy=first-available
protocol=classic

[routing:DR_rw]
bind_address=0.0.0.0
bind_port=8009
destinations=centos01:7001,olslave02:7001
routing_strategy=first-available
protocol=classic
```
以防我有点迷惑您，我在这里做的是通过**centos03:8008**为**centos02:6446**和**olslave01:6446**上InnoDB Cluster感知的Router提供一个Router接入点。然后添加或者备份访问点到**centos01:7001**和**olslave02:7001**上的静态Router。应用会有一个类似"centos03:8008,centos03:8009"的连接串。

现在，总是将主站点连接发送到centos03:8008。当我从监控中收到报错或者意味着有问题的通知时，我可以使用备用连接centos03:8009。

附注：我最初在每一侧都设置了单个Router进程，但是混合了InnoDB Cluster感知条目和额外的静态配置。这并没有真正起作用，因为从IDC启动后，动态状态条目只允许从`mysql_innodb_cluster_metadata`schema获取成员。所以它行不通。因此，需要额外的Router进程。但如果您仔细想想，让每个Router更加模块化和专用，这是一个更加原子化的解决方案。尽管要多管理一些。

然而，也许不需要高等级Router进程监听8008和8009，您想使用keepalived解决方案。看看Lefred的[MySQL Router HA with Keepalived](https://lefred.be/content/mysql-router-ha-with-keepalived/)的解决方案。
