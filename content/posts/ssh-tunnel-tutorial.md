---
title: SSH隧道简介
date: 2014-10-27
categories: [Linux, SSH, Proxy]
---

SSH的功能非常强大，日常除了用于命令行远程登录服务器。它还具有神奇的隧道（tunnel）功能（也被称为SSH代理），可用于加密访问本地或远程主机的服务。

通常，SSH代理具有3种方式：

## SSH（正向）代理 ##

通过参数-L [bind\_address:]port:remote\_host:remote\_port，将指定本地（客户端）端口转发至远程端口上。

![SSH Proxy](/images/ssh-tunnel-proxy.png)

如上图， hosta无法直接访问hostb，但它能直接SSH登录gateway；如此通过gateway，将hosta的端口X转发至hostb的端口Y上。相当于端口X和端口Y之间建立了加密隧道。

一般来说，端口Y为hostb上某服务的监听端口。当建立隧道后，hosta将监听端口X。应用程序访问hosta的端口X，等同于访问hostb的端口Y。对于应用程序，hostb端口Y对应的服务就如同运行在hosta上。

日常工作中，客户的网络常由于信息安全而被网关（或防火墙）隔离。当我们的软件在客户网络中某服务器发生问题时，我们常需奔赴客户现场进行调试。若客户存在某机器安装了SSH服务器，且能被外部访问。就可以利用SSH正向代理的方法，快速简便的登录被隔离的服务器并进行应用调试。


## SSH反向代理 ##

通过参数-R [bind\_address:]port:remote\_host:remote\_port，将指定远程端口转发至本地（客户端）端口上。

![SSH Reverse Proxy](/images/ssh-tunnel-reverse-proxy.png)

如上图，hosta在防火墙内，无法被hostb直接访问。但它能直接SSH登录hostb；如此通过hostb，将hostb的端口X转发至hosta的端口Y上。该方法与SSH正向代理类似，所不同的是该隧道的访问方向是从服务端（hostb）至客户端(hosta），故被称为反向代理。

其应用场景也与SSH正向代理类似，所不同的是若客户不存在可供外部访问的SSH服务器时，我们可以在外网建设一个SSH服务器给客户的被隔离服务器来建立隧道。如此，我们可以访问自己的SSH服务器对应端口来调试客户服务器的应用。

更进一步，客户内网甚至不能访问外网，此时可利用客户内网一台笔记本（或台式机，它可以访问目标服务器）USB接上3G/4G手机来达到访问外部SSH服务器的目的。

![SSH Reverse Proxy Mobile](/images/ssh-tunnel-reverse-proxy-mobile.png)

## SSH动态代理 ##

通过参数-D [bind\_address:]port，利用远程服务器为访问出口，在本地建立SOCKS 4/5代理服务器。

可以形象描绘为将本地应用的端口（SOCKS客户端端口），动态转发至远程。

![SSH Dynamic Proxy](/images/ssh-tunnel-dynamic-proxy.png)

该功能广为人知的应用场景为翻墙。如上图，在国外租用VPS（hostb），客户端（hosta）通过SSH动态代理端口X（SOCKS 4/5的端口）便可以访问被GFW封锁的网络。

这种翻墙最大的优势在于

* 低成本：国外廉价低配置VPS基本满足个人翻墙需求。
* 服务端0配置：服务端只需要安装SSH服务端。
* 客户端配置简单：客户端需要安装SSH客户端，以及一条命令。
* 加密隧道：保证网络访问的数据安全。

