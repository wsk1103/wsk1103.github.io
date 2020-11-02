---
title: "percona-tool文档说明（5）- 复制类"

tags:
  - MySQL
  - percona-tool
  
---

### pt-heartbeat
#### pt-heartbeat [OPTIONS] [DSN] --update|--monitor|--check|--stop
监视MySQL的延迟操作


### pt-slave-find
#### pt-slave-find [OPTIONS] [DSN]
查找并打印MySQL从属的复制层次结构树。


### pt-slave-restart
#### pt-slave-restart [OPTIONS] [DSN]
在发生错误后，重启MySQL。


### pt-slave-delay
#### pt-slave-delay [OPTIONS] SLAVE_DSN [MASTER_DSN]
使从库的数据比主库的数据落后。

```
pt-slave-delay --delay 1m --interval 15s --run-time 10m slavehost
```

根据需要启动和停止从属服务器，使其落后于主服务器



### pt-table-checksum
#### pt-table-checksum [OPTIONS] [DSN]
主要用来检查主从数据是否一致。通过在主服务器上执行校验和查询来执行在线复制一致性检查，这会在与主服务器不一致的副本上生成不同的结果。可选DSN指定主主机。如果发现任何差异，或者发生任何警告或错误，则工具的“退出状态”不为零。以上命令将连接到localhost上的复制主机，每个表的校验和，并在每个检测到的副本上报告结果

### pt-table-sync
#### pt-table-sync [OPTIONS] DSN [DSN]
有效地同步MySQL表数据。使用对两个库不一致的数据进行同步，他能够自动发现两个实例间不一致的数据，然后进行sync操作，pt-table-sync无法同步表结构，和索引等对象，只能同步数据。

```
pt-table-sync --execute h=host1,D=db,t=tbl h=host2
```

同步数据库中表1到另外数据库的表1数据

```
pt-table-sync --execute host1 host2 host3
```

将host1上的所有表同步到host2和host3：

```
pt-table-sync --execute --sync-to-master slave1
```

使slave1具有与其复制主机相同的数据

```
pt-table-sync --execute --replicate test.checksum master1
```

解决test.checksum在master1的所有从站上发现的差异

```
pt-table-sync --execute --replicate test.checksum --sync-to-master slave1
```

与上面相同，但仅解决slave1上的差异

```
pt-table-sync --execute --sync-to-master h=master2,D=db,t=tbl
```

在master-master复制配置中同步master2

```
pt-table-sync --execute h=master1,D=db,t=tbl master2
```

同步所有库和表

```
pt-table-sync --charset=utf8 --ignore-databases=mysql,sys,percona u=root,p=root,h=host1,P=3306 u=root,p=root,h=host2,P=3306 --execute --print
```

忽略库  
--ignore-databases=指定要忽略的库  