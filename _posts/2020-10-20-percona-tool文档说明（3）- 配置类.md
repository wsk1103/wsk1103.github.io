---
title: "percona-tool文档说明（3）- 配置类"

tags:
  - MySQL
  - percona-tool
  
---


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



### pt-mysql-summary
##### pt-mysql-summary [OPTIONS]
查看当前MySQL的详细信息

```
pt-mysql-summary –p=password


```


![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/35.png)



### pt-variable-advisor
#### pt-variable-advisor [OPTIONS] [DSN]
分析MySQL变量并就可能出现的问题提出建议。


