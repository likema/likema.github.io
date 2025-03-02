---
title: SSH翻墙集群
date: 2015-03-15 21:30:43 +0800
categories: [Linux, Ubuntu, SSH, Proxy, Nginx, HAProxy]
---

SSH动态代理是国内较为常见的翻墙方法。正如[SSH隧道简介](/blog/2014/10/27/ssh-tunnel-tutorial/)所说，它存在不少有优点。

然而在实际使用中，它存在如下缺点：

* 与PPTP等VPN协议相比，它的连接不稳定。前者应该具备协议级断线重传机制。
* 基于廉价VPS，导致它的连接不稳定。而且廉价VPS容易掉线，有时需要用户自己找在线客户修复，进一步延长了掉线时间。
* 由于上述缺点，不适合小型公司多人使用。

在大概2年前，我摸索出SSH动态代理集群的办法。并将之部署于我所服务的公司，成功负载了20-30人日常翻墙学习与工作的需求。

## 原理 ##

SSH动态代理，即为SOCK5代理，所以我们需要的是SOCK5集群。

若搜索[socks 5 load balance](https://www.google.com/search?hl=en&q=socks+5+load+balance)不难发现一些有用信息：

[What is the best way to load balance multiple sock5 proxys on seperate VM's in the same datacenter?](http://serverfault.com/questions/517971/what-is-the-best-way-to-load-balance-multiple-sock5-proxys-on-seperate-vms-in-t)

我将分别介绍3种方法搭建SOCK5集群：

 1. 利用第三方模块[nginx\_tcp\_proxy\_module](https://github.com/yaoweibin/nginx_tcp_proxy_module)。
 2. Nginx 1.9开始支持[TCP Load Balancing](http://nginx.com/resources/admin-guide/tcp-load-balancing/)。
 3. [HAProxy](http://www.haproxy.org/)

关于SSH动态代理的配置方法，请参看[AutoSSH简介](/blog/2014/12/23/autossh-tutorial/)

## nginx\_tcp\_proxy\_module的配置方法 ##

Ubuntu的Nginx并没有将nginx\_tcp\_proxy\_module编译进去。为了简化安装，我基于Ubuntu的Nginx包，做了Nginx的[PPA](https://launchpad.net/~likemartinma/+archive/ubuntu/net):

* 升级Nginx版本
* 加入nginx\_tcp\_proxy\_module

添加我的PPA

```bash
sudo add-apt-repository ppa:likemartinma/net
sudo apt-get -y update
```

若未安装nginx，则

```bash
sudo apt-get install -y nginx
```

若已安装nginx，则

```bash
sudo apt-get -y upgrade
```

在/etc/nginx/nginx.conf中，增加如下内容：

```nginx
tcp {
    access_log /var/log/nginx/tcp_access.log;

    upstream ssh_cluster {
        # simple round-robin
        server 127.0.0.1:12345;
        server 127.0.0.1:12346;
        server 127.0.0.1:12347;

        check interval=3000 rise=2 fall=5 timeout=1000;
    }

    server {
        listen 9999;
        proxy_pass ssh_cluster;
    }
}
```

为了查看集群的状态，在/etc/nginx/sites-enabled/default的中，增加如下内容：

```nginx
server {
    ...

    location /status {
        tcp_check_status;
    }
}
```

重启Nginx:

```
service nginx restart
```

如此，访问http\://&lt;cluster IP&gt;/status将能查看集群的详细状态。

## Nginx 1.9的配置方法 ##

Ubuntu 15.10之前的官方Nginx版本都小于1.9，须通过ppa:nginx/development升级nginx。

添加ppa:nginx/development

```bash
sudo add-apt-repository ppa:nginx/development
sudo apt-get -y update
```

若未安装nginx，则

```bash
sudo apt-get install -y nginx
```

若已安装nginx，则

```bash
sudo apt-get -y upgrade
```

在/etc/nginx/nginx.conf中，增加如下内容：

```nginx
stream {
    upstream ssh_cluster {
        least_conn;
        server 127.0.0.1:12345;
        server 127.0.0.1:12346;
        server 127.0.0.1:12347;
    }

    server {
        listen 9999;
        proxy_pass ssh_cluster;
    }
}
```

重启Nginx:

```
service nginx restart
```

## HAProxy的配置方法 ##

安装haproxy

```
sudo apt-get install -y haproxy
```

在/etc/haproxy/haproxy.cfg中，增加如下内容：

```
frontend socks5
    mode tcp
    bind *:9999
    default_backend ssh_cluster

backend ssh_cluster
    mode tcp
    balance roundrobin
    server vps1 127.0.0.1:12345 weight 1 check inter 30000
    server vps2 127.0.0.1:12346 weight 1 check inter 30000
    server vps3 127.0.0.1:12347 weight 1 check inter 30000
```

为了查看集群的状态，在/etc/haproxy/haproxy.cfg中，增加如下内容：

```
listen stats :9090
    balance
    mode http
    stats enable
    stats auth admin:admin
```

默认安装，haproxy处于不活动状态，须要激活它。

在/etc/default/haproxy中，修改如下行：

```
ENABLED=1
```

最后，启动haproxy:

```
service haproxy start
```

如此，访问http\://&lt;cluster IP&gt;:9090/haproxy?stats将能查看集群的详细状态。

## 总结 ##

 * nginx\_tcp\_proxy\_module有简单的集群状态页面。
 * nginx 1.9没有集群状态查页面，仅能通过错误日志/var/log/ngnix/error.log来查看掉线的集群节点。
 * haproxy不仅有完善的集群状态页面，而且不需要任何PPA，应该是最佳选择。
 * 上述3种方法都缺乏认证机制，只能部署于家庭或企业内网。当然也可以部署于个人电脑，事实上，我就是这样使用的。

