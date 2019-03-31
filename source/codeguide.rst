ZooKeeper 秩序员指南
####################
利用 ZooKeeper 开发分布式应用 [1]_

介绍
=====
这是一份给希望利用 ZooKeeper 的协调服务来开发分布式应用的开发者提供的指引。它包含一些概念性和实践性
的信息。

指引的前四节展示了一些 ZK 概念的上层讨论。这对于理解 ZK 是如何工作的以及如何用它来提供服务是很有必要
的。这部分不包含代码, 不过要求对分布式计算有一定的了解。这些章节包括:

* ZooKeeper 数据模型
* ZooKeeper 会话
* ZooKeeper 监视器(watches)
* 一致性保证 (Consistency Guarantees)

接下来的的四个章节提供实践性的变成信息, 
* 构建块: ZooKeeper 操作指引 (Building Blocks: A Guide to ZooKeeper Operations)
* 绑定
* 程序结构和简单例子
* Gotchas: 常见问题和诊断 (Troubleshooting)

本书以附录结束, 包含了其它一些 ZooKeeper 相关信息的链接。

文档中大部分信息都写成可以单独引用的材料。然而，在开始你的第一个 ZooKeeper 应用程序之前, 你很应该首先
阅读以下 ZooKeeper 数据模型和基本操作章节。同时简单编程例子对于理解 ZooKeeper 客户端应用程序非常有帮助。

ZooKeeper 数据模型
===================
ZK 有一个结构化的命名空间, 很像一个分布式的文件系统。唯一区别是命名空间中的每一个节点都有关联的数据。就
像一个文件系统允许作为一个文件又作为一个目录。路径采用规范化的(canonical), 绝对的(absoluate), 斜杠分隔
(slash-seperated)的路径, 没有相对引用。除了下面的限制外, 任何 unicode 字符都可以作为一个路径主体:
* 空字符(\u00000)不能作为路径的一部分(C 绑定会有问题)
* \u0001 - \u0019 和 \u007F - \u009F 不能使用, 因为它们不能很好的显示
* 不允许使用字符 \ud800 - \uF8FF, \uFFF0 - \uFFFF
* 符号 "." 可以使用作为其他名字的一部分, 但是 "." 和 ".." 不能单独使用
* "zookeeper" 作为保留字不能使用

节点(ZNodes)
------------
ZooKeeper 树上的每一个节点都称为一个 znode。Znodes 维护了一个包含数据变化的版本号 和 acl 变化的状态结构体。
结构体还包含时间戳。版本号和时间戳一起使得 ZK 可以验证缓存的有效性和协调更新。每次 znode 的数据变化, 版本号
都会增加。
例如, 当客户端获取数据时, 同时会得到数据的版本。当客户端执行更新或者删除时, 必须提供数据的版本。如果提供的版本
和数据的实际版本不一致, 那么操作会失败。(这一行为可以被重写)

znode 是程序员需要关注主要抽象(main abstraction), 它有以下几个特点值得拿出来说一下:

* Watches
客户端可以对 znode 设置 watches。对节点的修改会触发 watch, 并清除 watch。一个 watch 触发后 ZK 会发送
通知给客户端。更多信息查看 [2]_

* Data Access
每个节点 znode 中的数据读取、写入都是原子的。每个节点都有一个访问控制列表(Access Control List, ACL)。
ZK 并非设计成一个通用的数据库或者大对象存储, 而是用于协调数据。其数据通过以 kb 计算, 一般小于 1MB

* Ephemeral Nodes
ZK 也有临时节点的概念, 其存在的周期是会话存活期间。且不允许有子节点。

* Sequence Nodes -- Unique Naming
创建节点的时候可以让 ZK 加一个递增的计数在路径的后面, 这个计数在父节点下是唯一的, 内部表示为 4 个字节的无符号
正数, 所以如果增加值超过 2147483647 则会溢出

ZooKeeper 中的时间
-------------------
ZooKeeper 用多种方式来跟踪时间

* Zxid
ZK 状态的每一次修改都会携带一个 zxid(ZooKeeper Transaction Id), 严格递增且唯一, 可以用于判断事务发生的先后

* Version numbers
每一次修改会导致版本号加一。ZK 种有三个版本号:
    * version - znode 的数据修改
    * cversion - znode 的孩子节点的修改
    * aversion - znode 的 ACL 修改

* Ticks
ZK 用 tick 来定义事件的时间。例如状态码上传(status uplaod), 会话超时(session timeout), 节点间连接超时等

* Real time
ZK 几乎不适用真实时间, 或时钟时间, 除了在创建和修改 znode 时在 stat 结构体记录时间戳

ZooKeeper 的 Stat 结构体
-------------------------
stat 结构体的成员如下
* czxid - znode 创建的 zxid 
* mzxid - znode 最后修改的 zxid
* pzxid - znode 的子节点的最后修改 zxid
* ctime - znode 创建时, 距离当前周期(epoch)的时间, 单位毫秒
* mtime - znode 最后修改, 距离当前周期(epoch)的时间, 单位毫秒
* version - znode 修改的次数
* cversion - znode 的子节点修改的次数
* aversion - znode 的 ACL 修改次数
* ephemeralOwner - 如果 znode 是临时节点, 这里记录着会话 id。如果不是临时节点, 此值为 0
* numChildren - znode 的子节点个数

ZooKeeper 会话
==============
状态变迁图:
.. image:: _static/state_dia.jpg

连接字符串: ip1:port1,ip2:port2,...
客户端会随机选一个 ip:port 来连接, 如果失败则自动尝试下一个

v3.2.0 新增的功能, 允许在连接字符串添加路径。如 "127.0.0.1:4545/app/a" 或者
"127.0.0.1:3000,127.0.0.1:3001/app/a", 这样客户端的根路径会是 "/app/a", 后续所有的路径
也会是相对于这个根路径

客户端获得一个连接到 ZK 服务器的句柄后, ZK 对应创建了一个会话, 用一个 64 位的数表示。为了
安全, 服务器还会创建为会话 ID 创建一个密码, 客户端连接到新的服务器重建会话的时候, 把会话 ID
和密码一起发到服务器, 服务器会对此进行验证。

客户端和服务器的连接断开, ZK 客户端会自动进行重连, 不建议重新创建新的会话。

超时通知会在 session 重新建立连接后收到

引用资料
=========
.. [1] 官方文档 http://zookeeper.apache.org/doc/current/zookeeperProgrammers.html
.. [2] ZooKeeper Watches http://zookeeper.apache.org/doc/current/zookeeperProgrammers.html#ch_zkWatches