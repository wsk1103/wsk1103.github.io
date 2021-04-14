---
title: "percona-tool文档说明（2）- 性能类"

tags:
  - MySQL
  - percona-tool
  
---


### pt-index-usage
#### pt-index-usage [OPTIONS] [FILES]
从日志中读取查询并分析它们如何使用索引


### pt-pmp
#### pt-pmp [OPTIONS] [FILES]
聚合所选程序的GDB堆栈跟踪



### pt-visual-explain
#### pt-visual-explain [OPTIONS] [FILES]
将EXPLAIN输出格式化为树。



### pt-table-usage
#### pt-table-usage [OPTIONS] [FILES]
分析查询如何使用表。
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

