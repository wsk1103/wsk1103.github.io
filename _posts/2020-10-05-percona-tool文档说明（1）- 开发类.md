---
title: "percona-tool文档说明（1）- 开发类"

tags:
  - MySQL
  - percona-tool
  
---



### pt-duplicate-key-checker
#### pt-duplicate-key-checker [OPTIONS] [DNS]
查找数据库中重复的索引和外键。  
根据结果，我们可以看出重复的索引信息，包括索引定义，列的数据类型，以及修复建议。  
索引没有什么问题，如果有问题则会显示有问题的索引，并提供删除的sql语句  
没有问题：

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/21.png)

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/22.png)

存在重复索引

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/22.png)

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/24.png)


### pt-online-schema-change
#### pt-online-schema-change [OPTIONS] DNS
ALTER操作但是表没有锁定它们

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/15.png)

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/16.png)

```
pt-online-schema-change --alter “add column col int(11) Default null” h=host,P=3306,D=db,t=table,u=username,p=password --execute  
```

**参数说明**：  
其中   --alter “add column col int(11) Default null”    为OPTIONS里面的内容，接下来是DNS的内容。  
- D：数据库名
- t：数据库表名
- u：数据库登录用户名（可有可无，默认root）
- p：数据库密码（可有可无，默认没有密码）
- h：数据库地址（当使用远程连接的时候，本地的数据库IP必须被远程的数据库所允许请求连接）（可有可无，默认localhost）
- P：端口（可有可无，默认3306）
- -execute 表示执行该语句。


**PS:**  
当参数有特殊符号的时候，使用’’(单引号)括起来。  
OPTIONS中的””的命令内容无法使用多命令进行批操作。


### pt-show-grants
#### pt-show-grants [OPTIONS] [DSN]
规范化并打印MySQL授权，以便可以有效地复制，比较和版本控制它们。

```
pt-show-grants –p=password
```

打印MySQL的所有用户权限

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/37.png)

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/38.png)


### pt-upgrade
#### pt-upgrade [OPTIONS] LOGS|RESULTS DSN [DSN]
验证查询结果在不同服务器上是否相同。

```
pt-upgrade h=host1 h=host2 slow.log
```
