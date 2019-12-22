---
title: "Keepalived + Nginx 实现高可用"

url: "https://wsk1103.github.io/"

tags:
  - 架构
  - Nginx
  - Keepalived
---

PS： 理解**Keepalived** 

**Keepalived** + **Nginx** 实现高可用的思路：
1. 请求不是直接打到 Nginx 上，而是先通过 Keepalived （虚拟IP，VIP）
2. Keepalived 应该能监控 Nginx 的生命状态

![流程图](https://raw.githubusercontent.com/wsk1103/images/master/Keepalived/1.2.png)

#### 实现：

1. 准备至少2台服务器

IP地址
- 192.168.134.128（设置为Master）
- 192.168.134.129（设置为BackUp）


1. 安装 和启动 keepalived 


2. 修改 keepalived 配置文件

```
[root@wsk1103 ~]# vi /etc/keepalived/keepalived.conf
```

- 修改Master的配置文件 192.168.134.128

修改之前可以先复制一份备份。
```
! Configuration File for keepalived

global_defs {
   router_id keepalived_128 #设置keepalived的唯一ID，不能一致，一般可以把本地IP当做唯一ID

}

vrrp_script nginx_check {
    script "/etc/keepalived/nginx_check.sh" #定时检测nginx状态的脚步
    interval 2 #每2秒检测一次
    weight -20 #权重
}

vrrp_instance VI_1 {
    state MASTER #这里设置为Master
    interface ens32 #本地网卡名称，通过 ifconfig 得知
    virtual_router_id 200 #虚拟路由的 ID 号,2个节点的设置必须一致,相同的 VRID 为一个组，他将决定多播的MAC地址
    priority 50 #节点的优先级，范围为 0-254 ，Master的优先级必须必BackUp的高。
    advert_int 1 #组播信息发送的时间间隔，默认为1s。2个节点的设置必须一致。
    
    #设置账户校验信息，2个节点必须一致。
    authentication { 
        auth_type PASS
        auth_pass 1111
    }
    
    #虚拟IP池，2个节点也必须一致。
    virtual_ipaddress {
        192.168.134.200 #虚拟IP，可以设置多个。
        #192.168.134.201
    }
    
    # 脚本配置,不能写在 vrrp_script 后面，否则会导致脚本不生效。
    track_script {
        nginx_check
    }
}
```
- 修改 BackUp 的配置文件 192.168.134.129

```
! Configuration File for keepalived

global_defs {
   router_id keepalived_129 #设置keepalived的唯一ID，不能一致，一般可以把本地IP当做唯一ID

}

vrrp_script nginx_check {
    script "/etc/keepalived/nginx_check.sh" #定时检测nginx状态的脚步
    interval 2 #每2秒检测一次
    weight -20 #权重
}

vrrp_instance VI_1 {
    state BACKUP #这里设置为 BackUp
    interface ens33 #本地网卡名称，通过 ifconfig 得知
    virtual_router_id 200 #虚拟路由的 ID 号,2个节点的设置必须一致,相同的 VRID 为一个组，他将决定多播的MAC地址
    priority 40 #节点的优先级，范围为 0-254 ，Master的优先级必须必BackUp的高。
    advert_int 1 #组播信息发送的时间间隔，默认为1s。2个节点的设置必须一致。
    
    #设置账户校验信息，2个节点必须一致。
    authentication { 
        auth_type PASS
        auth_pass 1111
    }
    
    #虚拟IP池，2个节点也必须一致。
    virtual_ipaddress {
        192.168.134.200 #虚拟IP，可以设置多个。
        #192.168.134.201
    }
    
    # 脚本配置,不能写在 vrrp_script 后面，否则会导致脚本不生效。
    track_script {
        nginx_check
    }
}

```

3. 创建check_nginx 脚本

```
[root@localhost /]# cd /etc/keepalived/
[root@wsk1103 keepalived]# touch nginx_check.sh
[root@wsk1103 keepalived]# chmod 755 nginx_check.sh 
[root@wsk1103 keepalived]# sh nginx_check.sh
```

```
#!/bin/bash
counter=$(ps -C nginx --no-heading|wc -l)
if [ "${counter}" = "0" ]; then
    /usr/sbin/nginx
    sleep 2
    counter=$(ps -C nginx --no-heading|wc -l)
    if [ "${counter}" = "0" ]; then
        /etc/init.d/keepalived stop
    fi
fi
```
> 脚本含义  
> 判断 Nginx 是否已经启动（进程 Nginx 的个数 = 0,没有启动）。  
> 如果没有启动，则启动 Nginx ，并且延迟2秒后启动，防止抢占资源。  
> 再次判断 Nginx 有没有成功启动，如果没有启动成功，则 kill 了 keepalived 。

4. 重启 keepalived 和查看 IP 绑定情况

```
[root@wsk1103 keepalived]# service keepalived restart
Redirecting to /bin/systemctl restart keepalived.service
[root@wsk1103 keepalived]# ip addr
2: ens32: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:00:4d:76 brd ff:ff:ff:ff:ff:ff
    inet 192.168.134.128/24 brd 192.168.134.255 scope global noprefixroute dynamic ens32
       valid_lft 1363sec preferred_lft 1363sec
    inet 192.168.134.200/32 scope global ens32    #这里可以看到VIP已经绑定到对应的网卡。
       valid_lft forever preferred_lft forever
    inet6 fe80::b8c2:db98:5162:2e02/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

5. 修改 Nginx 的初始页面 index.htm

```
[root@localhost keepalived]# cd /usr/share/nginx/html
[root@localhost html]# vi index.html
```
在 **128** 的服务器上，加入 *128* 的字样。在 **129** 的服务器上，加入 *129* 的字样

6. 检验

6.1 页面访问 192.168.134.200  
![流程图](https://raw.githubusercontent.com/wsk1103/images/master/Keepalived/1.3.png)

6.2 停止 128 的 keepalived 服务

```
[root@wsk1103 keepalived]# service keepalived stop
Redirecting to /bin/systemctl stop keepalived.service
```
可以看到访问已经转移到 **129** 的服务器上了。
![流程图](https://raw.githubusercontent.com/wsk1103/images/master/Keepalived/1.4.png)

6.3 重启 128 的keepalived 服务

```
[root@wsk1103 keepalived]# service keepalived start
Redirecting to /bin/systemctl start keepalived.service
```
重启后，**128** 的 **keepalived** 重新接管，继续做为 **Master**
![流程图](https://raw.githubusercontent.com/wsk1103/images/master/Keepalived/1.5.png)

6.4 停止 128 的 nginx，观察 nginx 是否会重新启动

```
[root@wsk1103 keepalived]# service nginx stop
Redirecting to /bin/systemctl stop nginx.service
[root@wsk1103 keepalived]# ps -ef |grep nginx
root      21962      1  0 13:27 ?        00:00:00 nginx: master process /usr/sbin/nginx
root      21963  21962  0 13:27 ?        00:00:00 nginx: worker process
root      21965  21962  0 13:27 ?        00:00:00 nginx: worker process
root      21966  21962  0 13:27 ?        00:00:00 nginx: worker process
root      21967  21962  0 13:27 ?        00:00:00 nginx: worker process
root      21974  21321  0 13:27 pts/0    00:00:00 grep --color=auto nginx
```





