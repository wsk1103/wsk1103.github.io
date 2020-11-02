---
title: "percona-tool文档说明（6）- 系统类"

tags:
  - MySQL
  - percona-tool
  
---


### pt-diskstats
#### pt-diskstats
直接显示磁盘IO信息，与iostat类似，但是更详细。

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/19.png)

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/20.png)

实时循环显示数据结果


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


### pt-stalk
#### pt-stalk [OPTIONS]
出现问题时收集有关MySQL的取证数据


### pt-sift
#### pt-sift FILE|PREFIX|DIRECTORY
浏览由pt-stalk创建的文件

### pt-ioprofile
#### pt-ioprofile [OPTIONS] [FILE]
监视进程IO并打印文件表和I / O活动。