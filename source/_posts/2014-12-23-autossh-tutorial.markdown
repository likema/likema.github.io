---
layout: post
title: AutoSSH简介
comments: true
categories: [Linux, Ubuntu, SSH, Proxy]
---

[autossh](http://www.harding.motd.ca/autossh/) (Automatically restart SSH sessions and tunnels)，它在运行的时候启动一个SSH进程，并监控该进程的健康状况；当SSH进程崩溃或停止通信时，它将重启动SSH进程。

## 命令选项 ##

```bash
autossh [-V] [-M port[:echo_port]] [-f] [SSH_OPTIONS]
```

* **-M port[:echo\_port]** 指定监控端口（和echo端口，默认为前者加1）。
    * 若希望使远程标准inetd的echo服务（默认端口为7），则指定echo\_port，仅需服务监听地址为localhost。
    * 若port设置为0，则将禁用监控功能。仅在ssh退出后重启它。

* **-f** 使autossh在后台运行。

另外，autossh还提供了一组环境变量来控制其行为, 这里仅介绍几个有代表性的，其可以man autossh

* **AUTOSSH_FIRST_POLL** 指定首次论询测试时间。
* **AUTOSSH_POLL** 指定连接论询时间，默认600。若该值小于两次网络超时（默认15秒），则网络超将被调整为该值的1/2
* **AUTOSSH_GATETIME** 指定等待SSH连接成功建立的时间，默认30秒，超时表示首次运行失败，将退出autossh。若设为0，则禁用该功能，通常用于启动时运行autossh。
* **AUTOSSH_MAXLIFETIME** 指autossh最长运行时间，达到该时间，autossh将退出，并杀死SSH进程。
* **AUTOSSH_MAXSTART** 指定SSH最大启动次数。默认-1，表示无限制。

## Ubuntu配置方法 ##

基于Ubuntu 12.04或14.04，以SSH动态代理（即SSH翻墙）为例。init script和Upstart都可以将autossh变成服务，然Upstart的*respawn*容错能力更强，它能在服务进程掉线，重新启动该服务。

```
# autossh

description "autossh daemon"

start on (net-device-up IFACE=eth0 or net-device-up IFACE=wlan0)
stop on (net-device-down IFACE=eth0 and net-device-down IFACE=wlan0)

respawn

setuid like
setgid like

exec /usr/bin/autossh -M64000 -q -N -D localhost:12348 sshproxy
```

* *setuid*和*setgid*为了让autossh运行在指定的用户和用户组上。
* *start on*表示当eth0或wlan0激活时，启动autossh，*stop on*反之。其目的为避免系统启动或网络掉线时，频繁尝试启动autossh。

## 更好的办法 ##

最近OpenSSH都支持选项**ServerAliveInterval**和**ServerAliveCountMax**，实际为建立在SSH协议上的心跳测试。当测试失败后，SSH客户端进程将退出。通过Upstart的respawn功能重启SSH客户端进程，也能达到autossh目的。

仍以SSH动态代理为例：

```
# sshproxy

description "ssh proxy"

start on (net-device-up IFACE=eth0 or net-device-up IFACE=wlan0)
stop on (net-device-down IFACE=eth0 and net-device-down IFACE=wlan0)

respawn

setuid like
setgid like

exec /usr/bin/ssh \
    -oServerAliveInterval=300 \
    -oServerAliveCountMax=2 \
    -q -N -D localhost:12348 sshproxy
```
