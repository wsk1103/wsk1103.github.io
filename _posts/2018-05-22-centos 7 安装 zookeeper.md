---
title: "Centos 7 安装 zookeeper"

url: "https://wsk1103.github.io/"

tags:
  - zookeeper
  - 架构
---


## 环境
- Centos 7  
- zookeeper ：3.4.13  
- 下载地址：http://apache.fayea.com/zookeeper/zookeeper-3.4.13/zookeeper-3.4.13.tar.gz

## 下载

#### 使用 wget 命令下载
先下载到本地中
```
[root@izwz9ga6l7ls6ozy9ylwbdz down]# wget http://apache.fayea.com/zookeeper/zookeeper-3.4.13/zookeeper-3.4.13.tar.gz
```
#### 解压
解压后得到一个目录 **zookeeper-3.4.13**
```
[root@izwz9ga6l7ls6ozy9ylwbdz down]# tar -zxvf zookeeper-3.4.13.tar.gz
[root@izwz9ga6l7ls6ozy9ylwbdz down]# ls
zookeeper-3.4.13  zookeeper-3.4.13.tar.gz
```
#### 复制zoo_sample.cfg
进入 **conf** 目录，然后复制 **zoo_sample.cfg** ，并重命名为 **zoo.cfg**

```
[root@izwz9ga6l7ls6ozy9ylwbdz zookeeper-3.4.13]# cd conf/
[root@izwz9ga6l7ls6ozy9ylwbdz conf]# ls
configuration.xsl  log4j.properties  zoo_sample.cfg
[root@izwz9ga6l7ls6ozy9ylwbdz conf]# cp zoo_sample.cfg zoo.cfg
[root@izwz9ga6l7ls6ozy9ylwbdz conf]# ls
configuration.xsl  log4j.properties  zoo.cfg  zoo_sample.cfg
```
#### zoo.cfg 配置文件说明

```
# The number of milliseconds of each tick
tickTime=2000
ZK中的一个时间单元。ZK中所有时间都是以这个时间单元为基础，进行整数倍配置的。
# The number of ticks that the initial
# synchronization phase can take
initLimit=10
Follower在启动过程中，会从Leader同步所有最新数据，然后确定自己能够对外服务的起始状态。
Leader允许F在 initLimit 时间内完成这个工作。通常情况下，我们不用太在意这个参数的设置。
如果ZK集群的数据量确实很大了，F在启动的时候，从Leader上同步数据的时间也会相应变长，
因此在这种情况下，有必要适当调大这个参数了。
# The number of ticks that can pass between
# sending a request and getting an acknowledgement
syncLimit=5
在运行过程中，Leader负责与ZK集群中所有机器进行通信，例如通过一些心跳检测机制，
来检测机器的存活状态。如果L发出心跳包在syncLimit之后，
还没有从F那里收到响应，那么就认为这个F已经不在线了。
注意：不要把这个参数设置得过大，否则可能会掩盖一些问题。
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just
# example sakes.
dataDir=/tmp/zookeeper
存储快照文件snapshot的目录。默认情况下，事务日志也会存储在这里。
建议同时配置参数dataLogDir, 事务日志的写性能直接影响zk性能。
# the port at which the clients will connect
clientPort=2181
默认的端口号，一般不需要更改。
# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60
单个客户端与单台服务器之间的连接数的限制，是ip级别的，默认是60，如果设置为0，那么表明不作任何限制。
请注意这个限制的使用范围，仅仅是单台客户端机器与单台ZK服务器之间的连接数限制，
不是针对指定客户端IP，也不是ZK集群的连接数限制，也不是单台ZK对所有客户端的连接数限制。
#
# Be sure to read the maintenance section of the
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
这个参数和下面的参数autopurge.purgeInterval搭配使用，这个参数指定了需要保留的文件数目。
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1
ZK提供了自动清理事务日志和快照文件的功能，这个参数指定了清理频率，单位是小时，需要配置一个1或更大的整数。
```
#### 为 zoo.cfg 配置文件增加 事务日志 属性。

放在哪了都行。
```
dataLogDir=/tmp/zookeeperLogDir
```

## 启动

进入 **bin** 目录，执行 **./zkServer.sh start**

```
[root@izwz9ga6l7ls6ozy9ylwbdz conf]# cd ../bin/
[root@izwz9ga6l7ls6ozy9ylwbdz bin]# ls
README.txt    zkCli.cmd  zkEnv.cmd  zkServer.cmd  zkTxnLogToolkit.cmd
zkCleanup.sh  zkCli.sh   zkEnv.sh   zkServer.sh   zkTxnLogToolkit.sh
[root@izwz9ga6l7ls6ozy9ylwbdz bin]# ./zkServer.sh start
ZooKeeper JMX enabled by default
Using config: /home/wsk/down/zookeeper-3.4.13/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED
```


## 停止
执行 **./zkServer.sh stop**
```
[root@izwz9ga6l7ls6ozy9ylwbdz bin]# ./zkServer.sh stop
ZooKeeper JMX enabled by default
Using config: /home/wsk/down/zookeeper-3.4.13/bin/../conf/zoo.cfg
Stopping zookeeper ... STOPPED

```
## 重启
执行 **./zkServer.sh restart**
```
[root@izwz9ga6l7ls6ozy9ylwbdz bin]# ./zkServer.sh restart
ZooKeeper JMX enabled by default
Using config: /home/wsk/down/zookeeper-3.4.13/bin/../conf/zoo.cfg
ZooKeeper JMX enabled by default
Using config: /home/wsk/down/zookeeper-3.4.13/bin/../conf/zoo.cfg
Stopping zookeeper ... STOPPED
ZooKeeper JMX enabled by default
Using config: /home/wsk/down/zookeeper-3.4.13/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED

```

## 卸载
先停止服务，执行 **./zkServer.sh stop**  
再删除 zookeeper-3.4.13 目录，执行 **rm -rf zookeeper-3.4.13**  
最后删除 快照文件 目录，执行 **rm -rf /tmp/zookeeper**


