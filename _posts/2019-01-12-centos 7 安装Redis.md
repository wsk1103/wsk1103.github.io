---
title: "Centos 7 安装Redis"

url: "https://wsk1103.github.io/"

tags:
  - 架构
  - Redis
---



### 环境
Centos 7  
redis-5.0.4

## 开始安装

安装的过程中可以直接使用 **yum** ，但是这样安装的Redis版本比较低

```
[root@izwz9ga6l7ls6ozy9ylwbdz wsk]# yum install redis
.....
Installed:
  redis.x86_64 0:3.2.12-2.el7                                                                                   
Complete!
```
当我们需要高版本的时候，可以使用 **wget** 命令

#### 1. 下载redis安装包

```
[root@izwz9ga6l7ls6ozy9ylwbdz redis]# wget http://download.redis.io/releases/redis-5.0.4.tar.gz
```
#### 2. 解压压缩包

```
[root@izwz9ga6l7ls6ozy9ylwbdz redis]# tar zxvf redis-5.0.4.tar.gz
```

#### 3. 编译安装

先编译库
```
[root@izwz9ga6l7ls6ozy9ylwbdz redis-5.0.4]# make MALLOC=libc
```

再编译资源

```
[root@izwz9ga6l7ls6ozy9ylwbdz redis-5.0.4]# cd src && make install
    CC Makefile.dep

Hint: It's a good idea to run 'make test' ;)

    INSTALL install
    INSTALL install
    INSTALL install
    INSTALL install
    INSTALL install
```
至此，就算安装完成了。

## 启动Redis
有多种方法可以启动
#### 1. ./redis-server
在目录 src 下执行

```
[root@izwz9ga6l7ls6ozy9ylwbdz src]# ./redis-server
```
但是这样执行的 Redis 是前台任务，会随着终端的退出而关闭。  
如果想要后台一直运行，可以这样

```
[root@izwz9ga6l7ls6ozy9ylwbdz src]# nohup ./redis-server &
[1] 20876

```

#### redis-server
直接启动 Redis 服务
```
[root@izwz9ga6l7ls6ozy9ylwbdz src]# redis-server
```
同理，但是这样执行的 Redis 是前台任务，会随着终端的退出而关闭。  
如果想要后台一直运行，可以这样

```
[root@izwz9ga6l7ls6ozy9ylwbdz src]# nohup redis-server &
[1] 20948
```

#### 查看Redis服务
先输入 **redis-cli** ，接下来 可以 **ping** 一下
```
[root@izwz9ga6l7ls6ozy9ylwbdz redis]# redis-cli
127.0.0.1:6379> ping
PONG
```

设置一个 **string** 看看效果。

```
127.0.0.1:6379> set wsk 1103
OK
127.0.0.1:6379> get wsk
"1103"

```
#### 退出redis-cli

```
127.0.0.1:6379> quit
```
#### 设置开机自动启动
一般作为一个Redis服务器，当开机的时候，需要自动启动Redis。将Redis服务作为守护线程(daemon)。

修改 redis.conf 配置文件，修改为守护线程模式。

在 redis-5.0.4 目录下，编辑redis.conf
```
[root@izwz9ga6l7ls6ozy9ylwbdz redis-5.0.4]# vi redis.conf
```
搜索 **daemonize**   
默认情况下为 **daemonize no** ，修改为 **daemonize yes**

复制一份 **redis.conf** 到 **/etc/redis/** 下，并重命名为 **6379.conf**

```
[root@izwz9ga6l7ls6ozy9ylwbdz redis-5.0.4]# mkdir /etc/redis
[root@izwz9ga6l7ls6ozy9ylwbdz redis-5.0.4]# cp redis.conf /etc/redis/6379.conf
```

复制一份 **utils/redis_init_script** 到 **/etc/init.d/**

```
[root@izwz9ga6l7ls6ozy9ylwbdz utils]# cp /home/wsk/redis/redis-5.0.4/utils/redis_init_script /etc/init.d/
```

先测试启动一下：

```
[root@izwz9ga6l7ls6ozy9ylwbdz utils]# cd /etc/init.d/
[root@izwz9ga6l7ls6ozy9ylwbdz init.d]# ./redis_init_script start
Starting Redis server...
2317:C 29 Mar 2019 13:37:58.739 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
2317:C 29 Mar 2019 13:37:58.739 # Redis version=5.0.4, bits=64, commit=00000000, modified=0, pid=2317, just started
2317:C 29 Mar 2019 13:37:58.739 # Configuration loaded
```

设置开启启动

```
[root@izwz9ga6l7ls6ozy9ylwbdz init.d]# chkconfig redis_init_script on
```
当要取消的时候，把 on 修改为 off

重启服务器

```
[root@izwz9ga6l7ls6ozy9ylwbdz init.d]# reboot
```

重启后测试

```
[root@izwz9ga6l7ls6ozy9ylwbdz ~]# redis-cli
```

## 卸载Redis
#### 先关闭进程
查询进程，根据 **PID** 关闭

```
[root@izwz9ga6l7ls6ozy9ylwbdz ~]# ps -ef | grep redis
root       821     1  0 13:39 ?        00:00:00 /usr/local/bin/redis-server 127.0.0.1:6379
root      2360  2273  0 13:51 pts/0    00:00:00 grep --color=auto redis
[root@izwz9ga6l7ls6ozy9ylwbdz ~]# kill -9 821
```

#### 查找 Redis 相关文件

```
[root@izwz9ga6l7ls6ozy9ylwbdz ~]# find / -name redis*
/home/wsk/redis
/home/wsk/redis/redis-5.0.4
/home/wsk/redis/redis-5.0.4/utils/redis_init_script.tpl
/home/wsk/redis/redis-5.0.4/utils/redis-copy.rb
/home/wsk/redis/redis-5.0.4/utils/redis_init_script
/home/wsk/redis/redis-5.0.4/utils/redis-sha1.rb
/home/wsk/redis/redis-5.0.4/tests/integration/redis-cli.tcl
/home/wsk/redis/redis-5.0.4/tests/support/redis.tcl
/home/wsk/redis/redis-5.0.4/redis.conf
/home/wsk/redis/redis-5.0.4/src/redis-check-rdb.o
/home/wsk/redis/redis-5.0.4/src/redis-cli.c
/home/wsk/redis/redis-5.0.4/src/redis-benchmark.o
/home/wsk/redis/redis-5.0.4/src/redis-cli
/home/wsk/redis/redis-5.0.4/src/redis-server
/home/wsk/redis/redis-5.0.4/src/redis-check-aof.c
/home/wsk/redis/redis-5.0.4/src/redis-check-rdb
/home/wsk/redis/redis-5.0.4/src/redisassert.h
/home/wsk/redis/redis-5.0.4/src/redis-check-aof.o
/home/wsk/redis/redis-5.0.4/src/redis-check-aof
/home/wsk/redis/redis-5.0.4/src/redis-trib.rb
/home/wsk/redis/redis-5.0.4/src/redis-benchmark.c
/home/wsk/redis/redis-5.0.4/src/redis-benchmark
/home/wsk/redis/redis-5.0.4/src/redis-check-rdb.c
/home/wsk/redis/redis-5.0.4/src/redis-sentinel
/home/wsk/redis/redis-5.0.4/src/redismodule.h
/home/wsk/redis/redis-5.0.4/src/redis-cli.o
/home/wsk/redis/redis-5.0.4.tar.gz
/run/redis_6379.pid
/run/systemd/generator.late/redis_init_script.service
/run/systemd/generator.late/runlevel5.target.wants/redis_init_script.service
/run/systemd/generator.late/runlevel4.target.wants/redis_init_script.service
/run/systemd/generator.late/runlevel3.target.wants/redis_init_script.service
/run/systemd/generator.late/runlevel2.target.wants/redis_init_script.service
/run/systemd/generator.late/redis_6379.service
/usr/lib/python2.7/site-packages/pip/_vendor/cachecontrol/caches/redis_cache.pyc
/usr/lib/python2.7/site-packages/pip/_vendor/cachecontrol/caches/redis_cache.py
/usr/lib/systemd/system/redis-sentinel.service
/usr/lib/systemd/system/redis.service
/usr/share/licenses/redis-3.2.12
/usr/share/doc/redis-3.2.12
/usr/share/man/man1/redis-server.1.gz
/usr/share/man/man1/redis-cli.1.gz
/usr/share/man/man1/redis-sentinel.1.gz
/usr/share/man/man1/redis-check-rdb.1.gz
/usr/share/man/man1/redis-benchmark.1.gz
/usr/share/man/man1/redis-check-aof.1.gz
/usr/share/man/man5/redis.conf.5.gz
/usr/share/man/man5/redis-sentinel.conf.5.gz
/usr/bin/redis-cli
/usr/bin/redis-server
/usr/bin/redis-check-rdb
/usr/bin/redis-check-aof
/usr/bin/redis-benchmark
/usr/bin/redis-sentinel
/usr/local/bin/redis-cli
/usr/local/bin/redis-server
/usr/local/bin/redis-check-rdb
/usr/local/bin/redis-check-aof
/usr/local/bin/redis-benchmark
/usr/libexec/redis-shutdown
/etc/redis
/etc/redis-sentinel.conf
/etc/rc.d/init.d/redis_init_script
/etc/rc.d/init.d/redis.sh
/etc/systemd/system/redis-sentinel.service.d
```

#### 删除文件
可以根据目录进行删除。

```
[root@izwz9ga6l7ls6ozy9ylwbdz ~]# rm -rf /home/wsk/redis
```

