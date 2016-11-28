---
layout: post
title: Shadowsocks简介
date: 2016-11-29 01:39:55
comments: true
categories: [Linux, Ubuntu, Proxy]
---

[Shadowsocks](https://shadowsocks.org/en/index.html)（中文名: 影梭） 是一款开源的安全SOCKS 5代理，它主要用于在大陆翻墙。

## 原理

与SSH动态代理相似，客户端呈现为SOCKS 5代理服务，客户端与服务器之间采用加密通信。服务器部署于GFW之外，从而实现代理翻墙服务。

## 特点

* 使用自行设计的协议加密通信，支持多种加密算法：AES、Blowfish、IDEA、RC4等。除创建TCP连接外无需握手，每次请求只转发1个连接，因此使用起来网速较快，在移动设备上较省电。
* 通过异步I/O和事件驱动实现，响应速度快。
* 客户端支持主流操作系统平台：Windows、Linux、OS X、Android、iOS和OpenWrt

## shadowsocks-libev简介

网络中普遍采用Python版本的[shadowsocks](https://pypi.python.org/pypi/shadowsocks)，该版本看似安装简单，却存在如下缺点：

* 没有Linux操作系统原生安装包（如：RPM和DEB）。
* 没有操作系统的服务脚本（如init.d和upstart）。
* Python程序占用内存较多，运行效率不佳。

而[shadowsocks-libev](https://github.com/shadowsocks/shadowsocks-libev)是Shadowsocks在嵌入式和低端设备的轻量级实现：

* 纯C实现，不仅占用内存极小，且运行效率更快。
* 在几乎所有Linux平台都存在原生安装包，且具有对应平台的服务启动脚本。

下面基于Ubuntu 14.04/16.04介绍它的安装和配置。

## 安装shadowsocks-libev

虽然shadowsocks-libev可以编译出DEB，但是默认没有提供Ubuntu的包仓库。为了方便安装，我将它构建于我的PPA中。

客户端和服务端的安装方法相同：

```bash
sudo add-apt-repository ppa:likemartinma/net
sudo apt-get update
sudo apt-get install shadowsocks-libev
```

## 配置shadowsocks-libev服务端

```json
{
    "server_port": 8388,
    "password": "<共享密码>",
    "timeout": 60,
    "method": "aes-256-cfb"
}
```

* `server_port`为服务端监听端口
* `password`为客户端和服务端预设的共享密码，它最好由安全密码生成器生成（如[LastPass](https://www.lastpass.com/)或[KeePass](http://keepass.info/)），且长度不小于6个字符。
* `timeout`为连接超时时间。
* `method`为加密算法，`aes-256-cfb`的安全性较好。
 
配置完成后，需重启：

```bash
sudo service shadowsocks-libev restart
```

## 配置shadowsocks-libev客户端

创建/etc/shadowsocks-libev/client.json （文件名可修改）：

```json
{
    "server": "<服务端地址>",
    "server_port": "<服务端端口>",
    "local_port": "22357",
    "password": "<共享密码>",
    "method": "aes-256-cfb"
}
```

### Ubuntu 14.04

安装包没有提供系统服务脚本，故须自己创建Upstart脚本/etc/init/ss-local.conf （文件名可修改）：

```
# ss-local

description "shadowsocks client"

start on (net-device-up IFACE=eth0 or net-device-up IFACE=wlan0)
stop on (net-device-down IFACE=eth0 and net-device-down IFACE=wlan0)

respawn

setuid nobody
setgid nogroup

exec ss-local -c /etc/shadowsocks-libev/client.json
```

为了client.json的安全：

```bash
sudo chmod og-rwx /etc/shadowsocks-libev/client.json
sudo chown root:nogroup /etc/shadowsocks-libev/client.json
```

启动客户端

```bash
sudo start ss-local
```

最后，通过修改/etc/default/shadowsocks-libev的`START=no`禁止在客户机启动服务端程序——它在客户机没有作用。

```bash
sudo service shadowsocks-libev stop
```

### Ubuntu 16.04

安装包提供了systemd的服务模板/lib/systemd/system/shadowsocks-libev-local@.service:

```bash
sudo systemctl enable shadowsocks-libev-local@client
sudo systemctl start shadowsocks-libev-local@client
```

注意，@client与client.json的基本名必须一致。

最后，禁用并停止服务端程序：

```bash
sudo systemctl disable shadowsocks-libev
sudo systemctl stop shadowsocks-libev
```

## 集群化

类似SSH集群，多个shadowsocks也可以构建SOCKS 5集群，具体请参考《[SSH翻墙集群](http://www.malike.net.cn/blog/2015/03/15/ssh-proxy-cluster/)》的“HAProxy的配置方法”。






