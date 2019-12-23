---
title: "理解 Keepalived"

url: "https://wsk1103.github.io/"

tags:
  - 架构
  - Keepalived
---

## 1. Keepalived 是什么
[官网链接](https://www.keepalived.org/)

Keepalived is a routing software written in C. The main goal of this project is to provide simple and robust facilities for loadbalancing and high-availability to Linux system and Linux based infrastructures. Loadbalancing framework relies on well-known and widely used Linux Virtual Server (IPVS) kernel module providing Layer4 loadbalancing. Keepalived implements a set of checkers to dynamically and adaptively maintain and manage loadbalanced server pool according their health. On the other hand high-availability is achieved by VRRP protocol. VRRP is a fundamental brick for router failover. In addition, Keepalived implements a set of hooks to the VRRP finite state machine providing low-level and high-speed protocol interactions. In order to offer fastest network failure detection, Keepalived implements BFD protocol. VRRP state transition can take into account BFD hint to drive fast state transition. Keepalived frameworks can be used independently or all together to provide resilient infrastructures.

Keepalived is free software; you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation; either version 2 of the License, or (at your option) any later version.

Keepalive 是一款可以实现高可靠的软件，通常部署在大于2台的服务器上，其中一台作为主服务器，其余的作为备用服务器。Keepalived 可以对本机上的进程进行检测，一旦 Master (主)检测出某个进程出现问题，将自己切换成 Backup (副)状态，然后通知另外一个节点切换成 Master (主)状态。

## 2. Keepalived 安装

```
[root@localhost /]# cd /usr/local/src/
[root@localhost src]# wget http://www.keepalived.org/software/keepalived-2.0.19.tar.gz
[root@localhost keepalived-2.0.19]# tar -zxvf keepalived-2.0.19.tar.gz
[root@localhost keepalived-2.0.19]# cd keepalived-2.0.19/
[root@localhost keepalived-2.0.19]# ./configure --prefix=/usr/loacl/keepalived
[root@localhost keepalived-2.0.19]# make && make install
```

安装过程可能出现的问题
- 缺少C编译器
```
configure: error: in `/usr/local/keepalived-2.0.11':
configure: error: no acceptable C compiler found in $PATH
See `config.log' for more details
```
解决方案
```
缺少C编译器 安装GCC软件套件 yum install gcc
```

- 缺少 openssl-devel 

```
configure: error: 
  !!! OpenSSL is not properly installed on your system. !!!
  !!! Can not include OpenSSL headers files.            !!!
```
解决方案
```
yum -y install openssl-devel
```

- 缺少 libnl libnl-devel

```
*** WARNING - this build will not support IPVS with IPv6. Please install libnl/libnl-3 dev libraries to support IPv6 with IPVS.
```
解决方案

```
yum -y install libnl libnl-devel
```

## 2. 配置 Keepalived 和 开机启动

```
[root@localhost keepalived]# cd /usr/loacl/keepalived/
# keepalived启动脚本变量引用文件，默认文件路径是/etc/sysconfig/
[root@localhost keepalived]# cp etc/sysconfig/keepalived  /etc/sysconfig/ 
 
# 将keepalived主程序加入到环境变量（安装目录下）
[root@localhost keepalived]# cp sbin/keepalived /usr/sbin/
 
# keepalived启动脚本（源码目录下），放到/etc/init.d/目录下就可以使用service命令便捷调用
[root@localhost keepalived]# cp /usr/local/src/keepalived-2.0.19/keepalived/etc/init.d/keepalived /etc/init.d/
 
# 将配置文件放到默认路径下
[root@localhost keepalived]# mkdir /etc/keepalived
[root@localhost keepalived]# cp /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/

# 添加为系统服务
[root@localhost keepalived]# chkconfig --add keepalived
# 设置开机启动
[root@localhost keepalived]# chkconfig keepalived on

# 启动服务
[root@localhost keepalived]# service keepalived start
Starting keepalived (via systemctl):                       [  确定  ]


```

## 3. keepalived.conf 配置文件说明

```
! Configuration File for keepalived

global_defs {
    # 邮件通知配置
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc  #发邮件人
   smtp_server 192.168.200.1  #发送邮件的服务器地址
   smtp_connect_timeout 30   #连接超时时间
   router_id LVS_DEVEL #设置keepalived的唯一ID，不能一致，一般可以把本地IP当做唯一ID 
   vrrp_skip_check_adv_addr
   vrrp_strict
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}


vrrp_instance VI_1 {
    state MASTER   #这里主服务器为Master，如果为备用服务器为 BackUp
    interface eth0    #本地网卡名称，通过 ifconfig 得知
    virtual_router_id 51  #虚拟路由的 ID 号,2个节点的设置必须一致,相同的 VRID 为一个组，他将决定多播的MAC地址
    priority 100    #节点的优先级，范围为 0-254 ，Master的优先级必须必BackUp的高。
    advert_int 1   #组播信息发送的时间间隔，默认为1s。2个节点的设置必须一致。
    
    #设置账户校验信息，2个节点必须一致。
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    
    #虚拟IP池，2个节点也必须一致。
    virtual_ipaddress {
        192.168.200.16
        192.168.200.17
        192.168.200.18
    }
}

virtual_server 192.168.200.100 443 {
    delay_loop 6   #健康检查时间间隔
    lb_algo rr   #lvs调度算法rr|wrr|lc|wlc|lblc|sh|dh
    lb_kind NAT    #负载均衡转发规则NAT|DR|RUN
    persistence_timeout 50   #会话保持时间
    protocol TCP    #使用的协议

    real_server 192.168.201.100 443 {
        weight 1   #默认为1,0为失效
        SSL_GET {  
            url {   #检查url，可以指定多个
              path /
              digest ff20ad2481f97b1754ef3e12ecd3a9cc  #检查后的摘要信息
            }
            url {
              path /mrtg/
              digest 9b3a0c85a887a256d6939da88aabd8cd
            }
            connect_timeout 3   #链接超时时长，秒
            retry 3   #重试次数
            delay_before_retry 3   #在尝试之前延迟多长时间
        }
    }
}
```








