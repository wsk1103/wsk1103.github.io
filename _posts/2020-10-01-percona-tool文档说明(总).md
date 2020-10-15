---
title: "percona-tool文档说明(总)"

tags:
  - MySQL
 - percona-tool
---

基本所有涉及到数据库的操作，都需要填写相应的DNS命令，例如用户名，密码，数据库，数据库表等等。

[汇总目录官方文档地址：](https://www.percona.com/doc/percona-toolkit/2.2/index.html  )

![image](https://raw.githubusercontent.com/wsk1103/images/master/percona-tool/1.png)

## 参数说明
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