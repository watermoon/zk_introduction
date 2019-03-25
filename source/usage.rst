
ZooKeeper 使用
################
ZooKeeper 一个中心化的服务, 用于维护配置信息, 命名服务(naming), 提供分布式同步和集群服务(group services)。
这份说明主要是为了快速开始使用 ZooKeeper, 主要面向想快速尝试一下的开发人员。主要包含简单的单 ZK 服务器的安装
指令, 几个验证 ZK 正在工作的命令, 和一个简单的编程例子。如果想进行一个商业的部署, 请参考 ZK 管理员指引 [1]_ 。

依赖项
======

下载
======
可以到 http://zookeeper.apache.org/releases.html 下载最新的稳定发行版本

单机部署
========
配置
-----
下载后解压 ZK 服务器(一个单一的 jar 文件)。ZK 服务器运行需要一个配置文件, 在 conf/zoo.cfg 中, 内容如下

source code below ::

    tickTime=2000               # 最小时间单位, 最小的会话超时为两倍 tickTime
    dataDir=/var/lib/zookeeper  # 数据目录, 需要存在且开始的时候为空
    clientPort=2181             # 客户端连接的监听端口


启动命令
> bin/zkServer.sh start
注: 
windows 启动不要加 start, 直接 bin\zkServer.cmd 即可

管理 ZooKeeper 的存储
=====================

连接到 ZooKeeper
> bin/zkCli.sh -server 127.0.0.1:2181

连上后会看到类似这样的输出(我实际看到的别这多得多)

source code below::

    Connecting to localhost:2181
    log4j:WARN No appenders could be found for logger (org.apache.zookeeper.ZooKeeper).
    log4j:WARN Please initialize the log4j system properly.
    Welcome to ZooKeeper!
    JLine support is enabled
    [zkshell: 0]

    > help
    > ls /
    > create /zk_test mydata
    > ls /
    > get /zk_test
    > set /zk_test junk
    > get /zk_test
    > delete /zk_test
    > ls

编程连接到 ZooKeeper
====================
ZK 提供 Java 和 C 的绑定, C 的版本有两个变种: 单线程和多线程, 区别只在于消息循环。更多内容请看 [2]_


运行一个复制集的 ZooKeeper
==========================
运行在同一个应用中的复制集中的服务器(replicated group of servers)称为一个 quonum(o(╯□╰)o不会翻译)。
在复制集模式(replicated mode)下, 同一 quonum 下的所有服务器拥有相同配置文件的拷贝。
注: 在复制集下模式下, 需要至少 3 台服务器; 并且强烈建议拥有奇数个服务器。如果只有两台服务器, 不足以形成
一个 quonum 的多数(form a majority quorum)。此时, 会比单台服务器更不稳定, 因为有两个单点。
复制集模式的配置和单机模式差不多, 只有一点的小区别。下面是一个配置例子

source code below ::

    tickTime=2000
    dataDir=/var/lib/zookeeper
    clientPort=2181
    initLimit=5                 # 用于限制一个 quonum 中的 ZK 服务器连接到 leader 服务器的时长
    syncLimit=2                 # 限制 quonum 中服务器和 leader 的时间差
    server.1=zoo1:2888:3888
    server.2=zoo2:2888:3888
    server.3=zoo3:2888:3888


ZK 服务器起来后, 通过在数据目录的 myid 文件(文件包含服务器 ID)可以知道自己是谁。
端口
* 2888 - 用于 ZK 服务器之间的通信, 和 follower 连到接 leader
* 3888 - 用于 leader 选举

其它优化
========
还有一些其它的配置参数可以极大提升性能的, 例如为了降低延迟可以指定专门的事务日志目录。某人情况下, 事务日志放在
和数据目录、myid 文件同个目录。参数 dataLogDir 参数指明事务日志目录


引用资料
=========
.. [1] 官方文档 http://zookeeper.apache.org/doc/current/zookeeperAdmin.html
.. [2] 官方编程指南 http://zookeeper.apache.org/doc/current/zookeeperProgrammers.html