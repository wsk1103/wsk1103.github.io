---
title: "理解 zookeeper"

url: "https://wsk1103.github.io/"

tags:
  - zookeeper
  - 架构
---


## 官网
- [http://zookeeper.apache.org/](http://zookeeper.apache.org/)
- ZooKeeper Wiki：[https://cwiki.apache.org/confluence/display/ZOOKEEPER/Index](https://cwiki.apache.org/confluence/display/ZOOKEEPER/Index)
-


## zookeeper 是什么
ZooKeeper is a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services. All of these kinds of services are used in some form or another by distributed applications. Each time they are implemented there is a lot of work that goes into fixing the bugs and race conditions that are inevitable. Because of the difficulty of implementing these kinds of services, applications initially usually skimp on them, which make them brittle in the presence of change and difficult to manage. Even when done correctly, different implementations of these services lead to management complexity when the applications are deployed.
ZooKeeper是一种集中式服务，用于维护配置信息，命名，提供分布式同步和提供组服务。所有这些类型的服务都以分布式应用程序的某种形式使用。每次实施分布式应用都需要做很多工作来修复不可避免的错误和竞争条件。zookeeper就是为了解决这些问题。
- zookeeper 可以被用作注册中心
- 构建集群的时候，最好是奇数台服务器。

zookeeper 是一个典型的分布式数据一致性的解决方案，分布式的应用可以使用 zookeeper 来实现发布/订阅，服务命名，负载均衡，分布式协调/通知，Master选举，集群管理，分布式锁和分布式队列等。

最常见的 zookeeper  就是用来当做分布式注册中心。例如 Dubbo 就是使用 zookeeper 作为分布式注册中心，spring cloud 1 也是使用 zookeeper ，不过在 spring cloud 2 后，就建议使用 euraka 了。

## 概念
#### Session 会话
session 指的是 zookeeper 服务器与客户端之间的通讯。在 zookeeper 中，一个客户端连接指客户端与服务器之间的一个 TCP 长连接。客户端启动时，必须先与服务器建立一个 TCP 连接，从第一次建立的时候，客户端会话的生命周期也就是开始了。通过这个长连接，客户端能够通过心跳反应检测和服务器保持这个会话，也能够向 zookeeper 服务器发送请求并接受响应，同时还能通过该连接来接收服务器的 watch 事件通知。Session 的sessionTimeout 表示一个客户端会话的超时时间。当服务器压力过大、网络波动或客户端断开连接等原因时，只要在 sessionTimeout 规定的时间内能够重新连接，那么就表示该会话依然有效。

在为客户端创建会话之前，服务器会为每个客户端分配一个 sessionID 。该 sessionID 必须保证全局唯一，因为该 sessionID 是zookeeper 会话的一个重要标识，许多与会话相关的功能都需要这个 sessionID。

#### Znode 数据节点
zookeeper 将所有的数据存储在内存中，数据模型是一棵树(Znode Tree),由 / 进行分隔的路径，就是一个 Znode。

在 zookeeper 中，node可以分为持久节点和临时节点。
持久节点：一旦这个Znode 被创建时，除非 zookeeper 主动删除，否则这个 Znode 会一直保存在 zookeeper 上。
临时节点：会话周期和session绑定，当会话失效，这个临时节点就会被删除。

#### STAT
ZooKeeper命名空间中的每个znode都有一个与之关联的stat结构，类似于Unix/Linux文件系统中文件的stat结构。 znode的stat结构中的字段显示如下，各自的含义如下：

- cZxid：这是导致创建znode更改的事务ID。
- mZxid：这是最后修改znode更改的事务ID。
- pZxid：这是用于添加或删除子节点的znode更改的事务ID。
- ctime：表示从1970-01-01T00:00:00Z开始以毫秒为单位的znode创建时间。
- mtime：表示从1970-01-01T00:00:00Z开始以毫秒为单位的znode最近修改时间。
- dataVersion：表示对该znode的数据所做的更改次数。
- cversion：这表示对此znode的子节点进行的更改次数。
- aclVersion：表示对此znode的ACL进行更改的次数。
- ephemeralOwner：如果znode是ephemeral类型节点，则这是znode所有者的 session ID。 如果znode不是ephemeral节点，则该字段设置为零。
- dataLength：这是znode数据字段的长度。
- numChildren：这表示znode的子节点的数量。


#### watcher 事件监听
zookeeper 允许用户在指定节点上注册一些 watcher ，并且在一些特定的事件触发时，zookeeper 会将该事件通知到 对应的客户端上。这个机制是 zookeeper 实现分布式协调服务的重要特性。

#### ACL
zookeeper 采用 ACL（AccessControlLists）策略来进行权限控制，类型UNIX 文件系统的权限控制。 zookeeper 定义了5种权限。
- CRETE:创建子节点的权限
- READ:获取节点数据和子节点列表的权限
- WRITE:更新节点数据的权限
- DELETE:删除子节点的权限
- ADMIN:设置节点的ACL的权限
注意：CREATE和DELETE都是针对子节点的。

## 特点
- 顺序一致性：从同一客户端发起的事物请求，最终将会严格的安装顺序被应用到 zookeeper 中。
- 原子性：所有的事物请求的处理结果在集群的所有服务器上的情况都是一样的。
- 单一系统映像：无论客户端连接到哪个zookeeper ，看到的数据模型都是一致的。
- 可靠性：一旦一次更改请求被应用，更改的结果就会被持久化，直到下一次被更改覆盖。

## ZAB 协议和 Paxos 算法
#### ZAB 协议
zookeeper atomic broadcast 原子广播

分布式协调服务 zookeeper 专门设计的一种支持崩溃恢复的原子广播协议。在zookeeper 中，主要依赖ZAB 协议来实现分布式数据一致性，基于该协议， zookeeper 实现了一种主备模式的系统架构保持集群中各个副本之间数据的一致性。

#### ZAB 协议的2种基本模式：崩溃恢复 和 消息广播




