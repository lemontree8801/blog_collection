- [原文链接](https://lefred.be/content/mysql-innodb-cluster-is-the-router-a-single-point-of-failure/)
- [原文链接](https://lefred.be/content/mysql-router-ha-with-keepalived/)


# MySQL Router高可用
## MySQL InnoDB Cluster：router是单点故障吗？
众所周知，MySQL InnoDB Cluster由三个组件组成：
- 至少3节点的组复制cluster
- 用于管理cluster的MySQL Shell
- 将流量从应用服务器路由到cluster的MySQL Router

当介绍解决方案时，一个最主要的问题是我应该把router放在哪？答案始终是相同的：最佳位置是安装router在应用服务器！

router是一个轻量级的进程，它从cluster的元数据中获取配置，不需要很多资源或维护。

所以理想安装如下：

![enter image description here](https://lefred.be/wp-content/uploads/2018/07/spof01.png)

然而由于很多原因，有时人们并不想把MySQL Router与应用服务器放在同一台服务器上，想要将Router放在它自己的服务实例上（希望它们不是在同一物理机上的VM）。当然，这种设置是可能的，如下所示，但是，在特定情况下，router可能是spof（单点故障）。这意味着如果router crash或者安装router的服务器宕机，您的应用将无法连接到数据库。

![enter image description here](https://lefred.be/wp-content/uploads/2018/07/spof02.png)

如果您真的想要这样设置，您必须为MySQL Router安装高可用。这样的高可用设置非常依赖您的基础架构，因为您可能必须使用VIP，例如，在Linux和Windows上，这是不同的管理方式。

因此，如果您想要在推荐的应用服务器之外的其他地方安装MySQL Router，您不得不自己维护Router的高可用。

当然，有很多方案可以实现MySQL Router的高可用，我将演示其中一些。下一篇文章将介绍[如何使用Pacemaker设置MySQL Router的高可用](https://lefred.be/content/mysql-router-ha-with-pacemaker/)，请不要错过。

### 原理
原理很简单，您的MySQL Router运行在多台服务器上（至少不止一台），和一个VIP，应用程序使用VIP连接router。如果VIP在的router服务器宕机，VIP将自动转移到其他可用的router服务器/节点上：

![enter image description here](https://lefred.be/wp-content/uploads/2018/07/vip01.png)

如果router宕机，VIP转移到其他节点上：

![enter image description here](https://lefred.be/wp-content/uploads/2018/07/vip02.png)

当然，如果您的应用程序可以使用轮训（大多数MySQL Connector允许），您可以有多个VIP资源，如下：

![enter image description here](https://lefred.be/wp-content/uploads/2018/07/vip03.png)

![enter image description here](https://lefred.be/wp-content/uploads/2018/07/vip04.png)

### 总结
如果您遵循建议，MySQL Router不是单点故障，但是如果您想要在自身的服务器上搭建，那么您必须为它搭建高可用，因为这非常依赖您的基础架构。

## 使用Keepalived构建MySQL Router高可用
在解释了如何为不在应用服务器安装MySQL Router实现高可用，以及如何使用Pacemaker实现高可用之后，本文解释如何使用keepalived为MySQL Router实现高可用。

Keepalived非常流行，可能是因为它很容易使用。我们可以使用2个或多个服务器。原理与前面的文章相同，如果router崩溃，用于应用服务器连接MySQL的VIP被转移到MySQL Router仍在运行的另一台服务器上。

让我们看一下配置，在这个案例中使用了2台服务器，mysql1和mysql2。

### 配置
开始配置2个router节点。配置文件为`/etc/keepalived/keepalived.conf`，连接router的VIP是192.168.87.5。

我们决定那一台是主节点那一台是备用节点：`mysqlrouter1`将是主节点，`mysqlrouter2`将是备节点。

#### mysqlrouter1
```
global_defs {
   notification_email {
     lefred @ lefred.be
   }
   notification_email_from mycluster @ lefred.be 
   smtp_server 192.168.200.1
   smtp_connect_timeout 30
   router_id LVS_DEVEL
   vrrp_skip_check_adv_addr
   vrrp_strict
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}

vrrp_script chk_mysqlrouter {
  script "/bin/killall -0 /usr/bin/mysqlrouter" # check the haproxy process
  interval 2 # every 2 seconds
  weight 2 # add 2 points if OK
  fall 2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth1
    virtual_router_id 51
    priority 102
    advert_int 1
    virtual_ipaddress {
        192.168.87.5
    }
    track_script {
        chk_mysqlrouter
    }
}
```
关键的地方在于`state`设置为master，优先级部分。

#### mysqlrouter2
```
global_defs {
   notification_email {
     lefred @ lefred.be
   }
   notification_email_from mycluster @ lefred.be 
   smtp_server 192.168.200.1
   smtp_connect_timeout 30
   router_id LVS_DEVEL
   vrrp_skip_check_adv_addr
   vrrp_strict
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}

vrrp_script chk_mysqlrouter {
  script "/bin/killall -0 /usr/bin/mysqlrouter" # check the haproxy process
  interval 2 # every 2 seconds
  weight 2 # add 2 points if OK
  fall 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth1
    virtual_router_id 51
    priority 101
    advert_int 1
    virtual_ipaddress {
        192.168.87.5
    }
    track_script {
        chk_mysqlrouter
    }
}
```
我们看到`state`和优先级是不同的。

现在我们启动`keepalived`，查看VIP已经在主节点（mysqlrouter1）被启动：
```
[root@mysqlrouter1 ~]# systemctl start keepalived
[root@mysqlrouter1 ~]# ip add sho eth1
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 08:00:27:ab:eb:b4 brd ff:ff:ff:ff:ff:ff
    inet 192.168.87.3/24 brd 192.168.87.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet 192.168.87.5/32 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:feab:ebb4/64 scope link 
       valid_lft forever preferred_lft forever
```

可以看到192.168.87.5在eth1上已经可用。

如果我们停止mysqlrouter1上的`mysqlrouter`，我们可以看到最多2秒后，此IP将转移到mysqlrouter2上：
```
[root@mysqlrouter1 ~]# systemctl stop mysqlrouter
[root@mysqlrouter1 ~]# ip add sho eth1
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 08:00:27:ab:eb:b4 brd ff:ff:ff:ff:ff:ff
    inet 192.168.87.3/24 brd 192.168.87.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:feab:ebb4/64 scope link 
       valid_lft forever preferred_lft forever


[root@mysqlrouter2 ~]# ip add sho eth1
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 08:00:27:f9:23:f1 brd ff:ff:ff:ff:ff:ff
    inet 192.168.87.4/24 brd 192.168.87.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet 192.168.87.5/32 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fef9:23f1/64 scope link 
       valid_lft forever preferred_lft forever
```

