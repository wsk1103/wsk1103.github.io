---
title: "percona-tool文档说明（7）- 实用类"

tags:
  - MySQL
  - percona-tool
  
---

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


![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/11.png)


![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/13.png)



归档前的准备  
需要配置client字符集为utf-8,如果你用了utf-8的编码,防止归档数据为乱码  


### pt-find
#### pt-find [OPTIONS] [DATABASES]
查找MySQL中的表并执行操作，类似GUN的find命令。默认操作是打印数据库和表名。
```java
pt-find --ctime +0 --engine InnoDB --password=password
```
查找0天前所有用InnoDB创造的表并且打印出来

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/25.png)


```java
pt-find --engine InnoDB --exec "ALTER TABLE %D.%N ENGINE=MyISAM" –password=”” test
```
找到InnoDB格式的数据表，并将其转化为MyISAM格式

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/27.png)



```
pt-find --tablesize +1k –password=password test
```

寻找数据库test中，大于5k的表，并打印出来

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/29.png)



```
pt-find --printf "%T\t%D.%N\n" | sort -rn
```

找到所有表并打印它们的总数据和索引大小，并首先对最大的表进行排序

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/31.png)




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


### pt-align

#### pt-align [files]

将表的信息按列对齐打印输出。如果没有指定文件，则默认输出STDIN。  
没有使用命令输出结果

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/3.png)



使用命令输出结果

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/5.png)



使用例子

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/7.png)

