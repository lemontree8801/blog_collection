- [原文链接](https://mysqlserverteam.com/mysql-shell-adminapi-whats-new-in-8-0-23/)

MySQL开发团队很高兴地宣布新的MySQL Shell AdminAPI8.0.23版已经发布！

除了一些bug修复和小的更改之外，还包括一些关于**监控/故障排除**和**性能**的重要增强。

## MySQL Shell AdminAPI

### 集群诊断

检查集群的运行情况，在集群不是100%健康时进行故障定位及排查是DBA的主要工作之一。AdminAPI通过整合监控信息使得这个操作变得简单：

- <Cluster>.status([options])

这个版本中，我们扩展了`status()`命令，以提供更多诊断错误的相关信息。

通过几个例子来说明。"一图胜过千言万语！"

**集群成员从集群中被驱逐**

8.0.23之前的版本，当集群成员从集群中被驱逐，它仅简单显示为(`MISSING`)状态。然而，有许多原因导致成员变为(`MISSING`)状态，例如组复制停止、成员崩溃或者某些复制错误导致。

使用Cluster.status()的扩展选项是有价值的，因为它至少可以提供组复制所报告的实例成员角色。然而，它没有提供任何导致问题原因的额外信息。

出于这些原因，我们在`Cluster.status()`的默认输出中包含了这些内容：

1. 当对应的实例状态不为`ONLINE`时的成员状态。这个信息直接来自`performance_schema.replication_group_members`。
2. 每个实例的新增`instanceErrors` 字段，显示可以检测到的非`ONLINE` 实例的诊断信息。

组复制中手工停止实例，输出如下：

![https://mysqlserverteam.com/wp-content/uploads/2021/01/instance_error-2.png](https://mysqlserverteam.com/wp-content/uploads/2021/01/instance_error-2.png)
> 注意：此信息取决于实例是否可访问。

**脑裂**

脑裂是指某个实例已不属于主集群组，但它仍然上报自己处于`ONLINE`状态，并且可访问：

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/17cf01c7-23be-40a9-b55e-932e71f7e3c8/Untitled.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/17cf01c7-23be-40a9-b55e-932e71f7e3c8/Untitled.png)

**应用错误**

如果发生复制错误，该成员会长时间处于`RECOVERING`状态，直到最终失败，状态变为`MISSING` 。检查的方法是查看错误日志。

显然这对用户不友好，所以我们还列出成员如果进入`ERROR`状态，它的原因是什么：

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/20e79267-1735-4762-ac41-65a10fe514d8/Untitled.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/20e79267-1735-4762-ac41-65a10fe514d8/Untitled.png)

**其他诊断**

某些特定场景，例如从备份恢复集群成员，即使使用相同的主机和端口，也可能对`server_uuid`进行更改，这样它可以自动重新加入集群。但是，由于server_uuid是被用作实例的唯一标识，AdminAPI不会知道该实例已经重新连接，只会将其标记为`MISSING`。

类似地，一个在组复制中的实例，但是其元数据未更新，现在会在`Cluster.status()`中标记并上报。

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7e1e298b-f587-45f8-a7e7-94e40ea0858c/Untitled.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7e1e298b-f587-45f8-a7e7-94e40ea0858c/Untitled.png)

**总结**

在新的字段`instanceErrors`中检测并识别了以下问题：

- 未启用`super_read_only`的`SECONDARY`成员
- 恢复通道报错
- 应用通道报错
- 是组复制成员，但元数据未更新
- `OFFLINE`成员(GR插件暂停)
- 脑裂
- 成员的server_uuid与元数据中记录的不匹配。

### 复制相关信息

类似的，在`ReplicaSet.status()` 中，我们在新的recovery字段中记录了当成员执行增量恢复时复制恢复通道的信息。

> 注：只有当扩展列有值>0时，该信息才可用。

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/4296df4b-fa96-4cb8-ae47-85574635e011/Untitled.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/4296df4b-fa96-4cb8-ae47-85574635e011/Untitled.png)

### 多线程复制应用

MySQL InnoDB Cluster和InnoDB ReplicaSet使用不同的复制机制，分别是组复制和异步复制。然而，尽管这两种复制协议在数据传播方面是不同的，但都依赖于异步机制来处理和应用binlog的更改。主节点上的事务提交到从节点上的事务提交的时间间隔通常称为复制延迟。

也就是说，两种技术都有复制滞后的问题；这是MySQL DBA在生产环境中必须面对的最令人担忧的问题之一。

幸运的是，自从MySQL5.7以来，这个方面已经有了很大改善。例如，在MySQL8中，基于每个事务的`WRITESET` ，引入了一种新的跟踪独立事务的机制。通过评估哪些事务没有相互依赖进而可以并行应用binlog，这样大大提高了应用日志的吞吐量。

尽管这些为了改善复制延迟问题的改进已经上线一段时间，您需要调整设置，并且在此版本之前，复制应用默认使用的是单线程。考虑到许多常见的工作负载都有大量并发小事务，而且现在大多数服务器都有很高的处理能力，并行复制会有很大的不同！

基于以上原因，InnoDB Cluster和InnoDB ReplicaSet现在默认支持和启用并行复制。

**先决条件**

InnoDB Cluster和ReplicaSet需要MySQL服务端进行适当的设置，以便运行。为了在默认情况下启用并行复制，我们需要调整以下参数：

- `transaction_write_set_extraction=XXHASH64` (已经是InnoDB Cluster的先决条件而不是InnoDB ReplicaSet的)
- `slave_parallel_type=LOGICAL_CLOCK`
- `binlog_transaction_dependency_tracking=WRITESET`
- `slave_preserve_commit_order=ON`

关于如何检查和配置您的实例以准备使用InnoDB Cluster/ReplicaSet的方式没有任何变化，这些命令只是简单的增加了检查和自动启用上面列出的设置：

1. 检查先决条件：`dba.checkInstanceConfiguration()`
2. 使用以下步骤配置实例：`dba.configureInstance()`/ `dba.configureReplicaSetInstance()`

以下是使用`dba.configureInstance()` 为InnoDB Cluster配置实例的示例：

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/18116e9c-3de1-4b35-b2db-871eddb7ac19/Untitled.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/18116e9c-3de1-4b35-b2db-871eddb7ac19/Untitled.png)

**回放线程**

多线程复制依赖于多个执行任务的线程。线程的数量可以根据用户的情况进行配置和调整。我们认为4是一个适合典型部署和业务负载的合理数字，因此我们将其设置为默认值。

当您在为InnoDB Cluster/ReplicaSet配置实例时，可以更改默认值。`dba.configureInstance()` / `dba.configureReplicaSetInstance()` 增加了新的名为`applierWorkerThreads`的新选项。

例如：

```python
mysql-js> dba.configureInstance("clusteradmin@t480:3330", {applierWorkerThreads: 2, restart: true})
```

而且，与其他设置一样，您可以使用.options()命令查看Cluster/ReplicaSet的设置。

**调整回放线程的数量**

如果您需要调整正在运行的Cluster/ReplicaSet的线程数，您可以使用`dba.configureInstance()` / `dba.configureReplicaSetInstance()`命令。与配置实例时类似，您可以更改当前的线程数:

例如：

```bash
mysql-js> dba.configureInstance("clusteradmin@t480:3320", {applierWorkerThreads: 16})
```

> 注意：请注意，即使您可以更改在线成员的设置，它也不会立即生效，它需要实例重新加入(停止并启动GR插件)。

**Cluster/ReplicaSet升级会受到影响吗？**

当升级8.0.23之前的MySQL服务端组成的Cluster或ReplicaSet和MySQL Shell时，多线程复制可能因为这些设置不是必须的而并未启用。

MySQL Shell在运行`.status()` 命令时会检测到错误，并指导您进行相应更改，以启用多线程复制。

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c6837766-1328-45d8-a454-f575dad9c5df/Untitled.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c6837766-1328-45d8-a454-f575dad9c5df/Untitled.png)

## BUG修复

### BUG＃26649039 –Shell无法识别新UUID的成员重新加入

如果将集群成员从集群中删除，然后使用诸如MEB的备份中恢复，那么此实例无论是自动或者通过`Cluster.rejoinInstance()` 重新加入集群，它都会被标记为`(MISSING)` 。

这是因为AdminAPI使用server_uuid作为实例的唯一标识，从备份恢复后`server_uuid`可能发生变化，AdminAPI不认为这是同一个实例。

这与上述的新`Cluster.status()` 诊断那样被修复，即在实例重新加入时添加了新的检查，当通过UUID没有在元数据中找到时，会使用它的主机和端口进行搜索，如果找到，元数据将根据用于重新加入操作的选项进行更新。

### BUG＃31757737 – InnoDB Cluster操作应自动连接到主节点

如果shell的活跃会话连接到的是集群的从节点上，InnoDB Cluster的操作，例如Cluster.addInstance()或者Cluster.rejoinInstance()会失败。然而，考虑到Shell能够知道哪个成员是主节点，并且所有集群成员具有相同的集群管理凭据，命令应该不会失败，并且应自动连接主节点。

这就是解决这个bug的方法。现在，无论您在哪个成员建立连接来操作集群对象，操作都将相应地在正确的成员上执行。

### BUG#27882663 – `Cluster.Status()`没有显示不在元数据中的活跃的组复制成员

`Cluster.status()` 没有显示不属于元数据中的集群成员信息。这部分信息只有通过`Cluster.rescan()` 看到。然而，如果不显示组复制中的所有成员，即使元数据中没有出现，也会隐藏集群中意外/不在预期中的实例(未由InnoDB Cluster管理)。

这在Cluster.status()的改进中得到了修复，方法是列出组成集群的成员(即使没有在元数据中注册)，并指导用户将这些成员注册到元数据中。

### BUG#32112864 – `REBOOTCLUSTERFROMCOMPLETEOUTAGE()` 未排除在`"REMOVEINSTANCES"` 列表中的实例

`dba.rebootClusterFromCompleteOutage()`需要MySQL Shell在活跃会话中做：

- 获取在元数据中注册的集群成员。
- 确定哪个集群成员具有GTID的超集。
- 如果活跃会话不是GTID的超集成员，命令中止，并提示用户哪一个实例具有GTID超集。

但是，GTID超集的检查是在可以被Shell访问到的所有实例中完成的(在集群的元数据中有注册)。

如果实例具有不同的GTID集合，用户想要将其从集群中删除，操作将被阻塞，因为Shell不能确定哪个实例具有GTID超集。根据这个观点，不同的实例都可以被认为是最新的。

此外，用户应该可以选择具体的实例重建集群，即使它不是最新的，比如只要通过命令的选项或者提示说明其他的实例不打算重新加入到此集群中。

如果用户显式地设置`removeInstances` 参数或者在实例重新加入的提示中一直回答"否"，这样这些实例不参与GTID超集验证。

### BUG#31428813 – `DBA.UPGRADEMETADATA()` 失败并报错: UNKNOWN COLUMN ‘MYSQL.ROUTER’ IN ‘FIELD LIST’

如果sql_mode中使用`ANSI_QUOTES`，通过`dba.upgradeMetadata()` 更新元数据时候失败。

这是由特定查询导致的，该查询将数据插入`metadata`数据库中的`routers`表中，使用双引号将字符串引起来。当sql_mode中使用`ANSI_QUOTES` 时，MySQL将双引号视为标识符引号，而不是字符串引号，从而在运行该查询时导致错误。

该补丁通过在升级元数据命令时，在AdminAPI准备使用的会话中，包括了完整性检查，确保该会话使用的sql_mode使用默认值，以避免用户设置不兼容。

### BUG#32152133 – 替换 `CHANGE MASTER/START SLAVE`术语为`CHANGE SOURCE/START REPLICA`

同MySQL服务端一样，复制相关功能中过时的术语得到了更新，同时在必要时保持向后的兼容。

专门处理了以下MySQL复制命令：

- CHANGE MASTER TO
- START SLAVE UNTIL MASTER_LOG_POS, MASTER_LOG_FILE

替换为

- CHANGE REPLICATION SOURCE TO
- START REPLICA UNTIL SOURCE_LOG_POS, SOURCE_LOG_FILE

相关参数：

- MASTER_HOST
- MASTER_PORT
- MASTER_*

替换为

- SOURCE_HOST
- SOURCE_PORT
- SOURCE_*

您还可以在MySQL术语更新的博客中获得更详细的信息。
