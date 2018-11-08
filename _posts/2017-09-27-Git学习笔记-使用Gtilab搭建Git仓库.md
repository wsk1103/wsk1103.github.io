---
title: "Git学习笔记-使用GtiLab搭建Git仓库"

tags:
  - Git
  - 学习笔记
---

#### 官方文档：https://about.gitlab.com/installation/#centos-6

**本机系统：**
```
[root@localhost wsk]# cat /etc/issue
CentOS release 6.6 (Final)
Kernel \r on an \m
```

**git版本：**
```
[root@localhost build]# git --version
git version 1.7.1
```
**gitlab版本：**
```
[root@localhost build]# cat /opt/gitlab/embedded/service/gitlab-rails/VERSION
11.3.4-ee
```
注意：gitlab需要glibc的版本至少2.17。centos自带的版本为2.12，会安装不了。
查看glibc版本

```
[root@localhost wsk]# ldd --version
ldd (GNU libc) 2.12
Copyright (C) 2012 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
Written by Roland McGrath and Ulrich Drepper.
```
#### 升级GLIBC
**1. 下载安装GLIBC tar包（切换用户到root）**

```
wget http://ftp.gnu.org/gnu/glibc/glibc-2.17.tar.gz
tar –zxvf glibc-2.17.tar.gz
cd glibc-2.17
mkdir build
cd build
../configure --prefix=/usr --disable-profile --enable-add-ons --with-headers=/usr/include --with-binutils=/usr/bin
make –j4
make install
```

**2. 更新后查看版本**

```
[root@localhost wsk]# strings /lib64/libc.so.6 | grep GLIBC
GLIBC_2.2.5
GLIBC_2.2.6
GLIBC_2.3
GLIBC_2.3.2
GLIBC_2.3.3
GLIBC_2.3.4
GLIBC_2.4
GLIBC_2.5
GLIBC_2.6
GLIBC_2.7
GLIBC_2.8
GLIBC_2.9
GLIBC_2.10
GLIBC_2.11
GLIBC_2.12
GLIBC_2.13
GLIBC_2.14
GLIBC_2.15
GLIBC_2.16
GLIBC_2.17
GLIBC_PRIVATE
```

#### 接下来可以安装gitlab了

**1. 安装并配置必要的依赖项**

在CentOS 6（和RedHat / Oracle / Scientific Linux 6）上，以下命令还将在系统防火墙中打开HTTP和SSH访问。

```
sudo yum install -y curl policycoreutils-python openssh-server cronie

sudo lokkit -s http -s ssh
```
接下来，安装Postfix以发送通知电子邮件。如果要使用其他解决方案发送电子邮件，请跳过此步骤并在安装GitLab后配置外部SMTP服务器。
```
sudo yum install postfix
sudo service postfix start
sudo chkconfig postfix on
```
在Postfix安装期间，可能会出现配置屏幕。选择“Internet Site”并按Enter键。使用服务器的外部DNS作为“邮件名称”，然后按Enter键。如果出现其他屏幕，请继续按Enter键接受默认值。


**2. 添加GitLab软件包存储库并安装软件包（大小456M，安装的过程可能会经常断，需要重新执行命令）**

添加GitLab包存储库。

```
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.rpm.sh | sudo bash
```
接下来，安装GitLab包。
将“http://gitlab.example.com（本机的话，可以改为http://localhost）”更改为您要访问GitLab实例的URL。
安装将自动配置并启动该URL的GitLab。
HTTPS在安装后需要其他配置

```
sudo EXTERNAL_URL="http://gitlab.example.com" yum -y install gitlab-ee
```
**3. 在GitLab中启用相对URL**

 （可选）如果缺少资源，可以使用以下命令关闭Unicorn和Sidekiq，暂时释放一些内存：

```
sudo gitlab-ctl stop unicorn
sudo gitlab-ctl stop sidekiq
```
在/etc/gitlab/gitlab.rb中设置external_url


```
vi /etc/gitlab/gitlab.rb
```


寻找external_url
修改为

```
external_url "http://localhost"
```
重新配置GitLab以使更改生效：

```
sudo gitlab-ctl reconfigure
```
重新启动服务，以便Unicorn和Sidekiq获取更改

```
sudo gitlab-ctl restart
```

**4. 登录url进行操作**。

打开浏览器，访问 http://localhost (如果和apache等发生了冲突了，可以使用nginx反向代理或者修改端口。)

当第一次打开页面的时候，需要修改密码，修改密码后既可以登录系统。
管理员角色账号为root
管理员操作界面，至此，安装完成。
![image](http://106.12.105.253/images/note/20181012143134.png)

**5. 创建新的项目**

和在github上穿甲的类似。

**6. 客户端GIT拉取gitlab上的项目**

配置本地的账号密码。

```
$ git config --global user.name "wsk1103"
$ git config --global user.email "wsk1261709167@gmail.com"
```
生成SSH密钥

```
$ ssh-keygen -t rsa -C "wsk1261709167@gmail.com"
```

打开自己的GitLab，在setting中，点击SSH侧边栏，填入id_rsa.pub里面的东西。

clone项目到本地

```
$ git clone git@192.168.254.136:root/myspring.git
```

完成。