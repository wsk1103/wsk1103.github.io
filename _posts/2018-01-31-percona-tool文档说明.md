---
title: "percona-tool文档说明"

tags:
  - MySQL
  - percona-tool

---


[汇总目录官方文档地址：](https://www.percona.com/doc/percona-toolkit/2.2/index.html  )

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/1.png)

基本所有涉及到数据库的操作，都需要填写相应的DNS命令，例如用户名，密码，数据库，数据库表等等。

### pt-align

#### pt-align [files]

将表的信息按列对齐打印输出。如果没有指定文件，则默认输出STDIN。  
没有使用命令输出结果

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/3.png)

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/4.png)

使用命令输出结果

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/5.png)

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/6.png)

使用例子

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/7.png)

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/8.png)

### pt-archiver
#### pt-archiver [OPTIONS] --source DSN --where WHERE
将数据库的表里的数据存储到另外一个表或者文件里。总而言之：就是用来归档数据。  
作用：  
- 清理线上过期数据；
- 导出线上数据，到线下数据作处理；
- 清理过期数据，并把数据归档到本地归档表中，或者远端归档服务器。 
 
*注意：* pt-archiver操作的表必须有主键  
具体使用，从一张表导入到另外一张表，要注意的是新表必须是已经建立好的一样的表结构，不会自动创建表，而且where条件是必须指定的，如果没有添加 –-no-delete，则默认会删除原来的表：
![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/9.png)

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/10.png)

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/11.png)

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/12.png)

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/13.png)

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/14.png)


归档前的准备  
需要配置client字符集为utf-8,如果你用了utf-8的编码,防止归档数据为乱码  

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
### pt-config-diff
#### pt-config-diff [OPTIONS] CONFIG CONFIG [CONFIG...]

比较多份配置文件的不同  
```java
pt-config-diff h=host1 h=host2
```
比较2个地址中配置文件的不同
```mysql
pt-config-diff /etc/my.cof h=host2
```
比较本地配置文件和远程配置文件的mysqld的不同
```java
pt-config-diff /etc/my.cof /etc/wsk.cof
```
比较2个文件找那个mysqld的不同  
结果输出：

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/17.png)

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/18.png)

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

### pt-diskstats
#### pt-diskstats
直接显示磁盘IO信息，与iostat类似，但是更详细。

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/19.png)

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/20.png)

实时循环显示数据结果
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

### pt-fifo-split
#### pt-fifo-split [OPTIONS] [FILE]
模拟切割文件，并通过管道传递给先入先出队列而不用真正的切割文件。

```java
pt-fifo-split --lines 1000000 hugefile.txt
``` 
使用pt-fifo-split分割一个大文件，每次读1000000行
 
pt-fifo-split 默认会在/tmp下面建立一个fifo文件，并读取大文件中的数据写入到fifo文件，每次达到指定行数就往fifo文件中打印一个EOF字符，读取完成以后，关闭掉fifo文件并移走，然后重建fifo文件，打印更多的行。这样可以保证你每次读取的时候都能读取到制定的行数直到读取完成。注意此工具只能工作在类unix操作系统。
 
常用选项：
- --fifo /tmp/pt-fifo-split，指定fifo文件的路径；
- --offset 0，如果不打算从第一行开始读，可以设置这个参数；
- --lines 1000，每次读取的行数；
- --force，如果fifo文件已经存在，就先删除它，然后重新创建一个fifo文件；
### pt-find
#### pt-find [OPTIONS] [DATABASES]
查找MySQL中的表并执行操作，类似GUN的find命令。默认操作是打印数据库和表名。
```java
pt-find --ctime +0 --engine InnoDB --password=sk.w1103
```
查找0天前所有用InnoDB创造的表并且打印出来

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/25.png)

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/26.png)

```java
pt-find --engine InnoDB --exec "ALTER TABLE %D.%N ENGINE=MyISAM" –password=”” test
```
找到InnoDB格式的数据表，并将其转化为MyISAM格式

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/27.png)

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/28.png)


```
pt-find --tablesize +1k –password=sk.w1103 test
```

寻找数据库test中，大于5k的表，并打印出来

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/29.png)

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/30.png)


```
pt-find --printf "%T\t%D.%N\n" | sort -rn
```

找到所有表并打印它们的总数据和索引大小，并首先对最大的表进行排序

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/31.png)

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/32.png)

### pt-summary
#### pt-summary
查看当前系统的信息

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/33.png)

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/34.png)

### pt-heartbeat
#### pt-heartbeat [OPTIONS] [DSN] --update|--monitor|--check|--stop
监视MySQL的延迟操作
### pt-mysql-summary
##### pt-mysql-summary [OPTIONS]
查看当前MySQL的详细信息

```
pt-mysql-summary –p=sk.w1103
```


![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/35.png)

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/36.png)


### pt-fk-error-logger
#### pt-fk-error-logger [OPTIONS] [DSN]
记录MySQL外键的错误信息
### pt-index-usage
#### pt-index-usage [OPTIONS] [FILES]
从日志中读取查询并分析它们如何使用索引
### pt-query-digest
#### pt-query-digest [OPTIONS] [FILES] [DSN]
从日志，进程列表和tcpdump分析MySQL查询
### pt-pmp
#### pt-pmp [OPTIONS] [FILES]
聚合所选程序的GDB堆栈跟踪
### pt-mext
#### pt-mext [OPTIONS] -- COMMAND
查看MySQL 的SHOW GLOBAL STATUS的许多示例并排。
### pt-kill
#### pt-kill [OPTIONS] [DSN]
杀死符合特定条件的MySQL查询

```
pt-kill --busy-time 60 --kill
```

杀死查询时间大于60s的语句。

```
pt-kill --busy-time 60 --print
```

打印但是不杀死查询时间大于60s的语句。

```
pt-kill --match-command Sleep --kill --victims all --interval 10
```

每过10s检查并杀死睡眠状态的进程。

```
pt-kill --match-state login --print --victims all
```

打印但是不杀死所有进程。  

**常用参数说明**  
- no-version-check：不最新检查版本
- host：连接数据库的地址
- port：连接数据库的端口
- user：连接数据库的用户名
- passowrd：连接数据库的密码
- charset：指定字符集
- match-command：指定杀死的查询类型
- match-user：指定杀死的用户名,即杀死该用户的查询
- busy-time：指定杀死超过多少秒的查询
- kill：执行kill命令
- victims：表示从匹配的结果中选择,类似SQL中的where部分,all是全部的查询
- interal：每隔多少秒检查一次
- print：把kill的查询打印出来


### pt-ioprofile
#### pt-ioprofile [OPTIONS] [FILE]
监视进程IO并打印文件表和I / O活动。
### pt-slave-find
#### pt-slave-find [OPTIONS] [DSN]
查找并打印MySQL从属的复制层次结构树。
### pt-show-grants
#### pt-show-grants [OPTIONS] [DSN]
规范化并打印MySQL授权，以便可以有效地复制，比较和版本控制它们。

```
pt-show-grants –p=sk.w1103
```

打印MySQL的所有用户权限

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/37.png)

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/38.png)

### pt-visual-explain
#### pt-visual-explain [OPTIONS] [FILES]
将EXPLAIN输出格式化为树。
### pt-variable-advisor
#### pt-variable-advisor [OPTIONS] [DSN]
分析MySQL变量并就可能出现的问题提出建议。
### pt-upgrade
#### pt-upgrade [OPTIONS] LOGS|RESULTS DSN [DSN]
验证查询结果在不同服务器上是否相同。

```
pt-upgrade h=host1 h=host2 slow.log
```

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
### pt-table-checksum
#### pt-table-checksum [OPTIONS] [DSN]
主要用来检查主从数据是否一致，
 
### pt-table-checksum
通过在主服务器上执行校验和查询来执行在线复制一致性检查，这会在与主服务器不一致的副本上生成不同的结果。可选DSN指定主主机。如果发现任何差异，或者发生任何警告或错误，则工具的“退出状态”不为零。以上命令将连接到localhost上的复制主机，每个表的校验和，并在每个检测到的副本上报告结果
### pt-stalk
#### pt-stalk [OPTIONS]
出现问题时收集有关MySQL的取证数据
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
### pt-sift
#### pt-sift FILE|PREFIX|DIRECTORY
浏览由pt-stalk创建的文件

### DSN
**DSN的详细参数：**
- a:查询
- A:字符集
- b：true代表禁用binlog
- D：数据库
- u：数据库链接账号
- p：数据库链接密码
- h：主机IP
- F：配置文件位置
- i：是否使用某索引
- m：插件模块
- P：端口号
- S：socket文件
- t：表
### OPTIONS
- --ask-pass  
连接数据库的时候提示密码
- --charset  
类型：string  
简写 –A  
字符类似设置  
- --config  
类型：数组  
配置文件。如果该值为必须的情况下，必须放在命令首位（相当于default-file）。  
- --database  
类型：string  
简写：-D   
连接数据库  
- --defaults-file  
简写：-F  
类型：string  
仅从给定文件中读取mysql选项。必须提供绝对路径名。  
- --help  
帮助并退出  
- --host  
类型：string  
简写：-h  
连接地址  
- --[no]ignore-case  
对比变量的时候忽略大小写。  
- --ignore-variables  
类型：数组  
忽略，并不进行比较   
- --password  
类型：string  
简写：-p  
连接密码  
- --port  
类型：int  
简写：-P  
连接端口  
- --[no]report  
将对比不同的报告写到磁盘中。  
- --socket  
类型：string  
简写：-S  
套接字连接  
- -user  
类型：string  
简写：-u  
用户名  