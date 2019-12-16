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

#### 实现：

1. 安装 和启动 keepalived 

```
[root@wsk1103 html]# yum install keepalived
[root@wsk1103 html]# service keepalived start
```

2. 修改 keepalived 配置文件

```

```
