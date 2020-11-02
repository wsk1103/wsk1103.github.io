---
title: "percona-tool文档说明（4）- 监控类"

tags:
  - MySQL
  - percona-tool
  
---

### pt-deadlock-logger
记录MySQL死锁的原因日志  
#### pt-deadlock-logger [OPTIONS] DSN
```java
pt-deadlock-logger h=host1
```
在host1上打印死锁的日志

```java
pt-deadlock-logger h=host1 --iterations 1
```
在host1上打印死锁日志并退出

```java
pt-deadlock-logger h=host1 --dest h=host2,D=percona_schema,t=deadlocks
```
将host1上的死锁信息打印到host2对应的数据库表中


### pt-fk-error-logger
#### pt-fk-error-logger [OPTIONS] [DSN]
记录MySQL外键的错误信息

### pt-mext
#### pt-mext [OPTIONS] -- COMMAND
查看MySQL 的SHOW GLOBAL STATUS的许多示例并排。


### pt-query-digest
#### pt-query-digest [OPTIONS] [FILES] [DSN]
从日志，进程列表和tcpdump分析MySQL查询


