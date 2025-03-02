---
title: AutoSSH简介
date: 2014-12-23
categories: [Linux, Ubuntu, SSH, Proxy]
---

[autossh](http://www.harding.motd.ca/autossh/) (Automatically restart SSH sessions and tunnels) 在运行的时候启动一个 ssh 进程，并监控该进程的健康状况；当 ssh 进程崩溃或停止通信时，它将重启动 ssh 进程。

## 命令选项 ##

```bash
autossh [-V] [-M port[:echo_port]] [-f] [SSH_OPTIONS]
```

* `-M port[:echo_port]` 指定监控端口（和echo端口，默认为前者加1）。
    * 若希望使远程标准 inetd 的 echo 服务（默认端口为7），则指定 `echo_port` ，仅需服务监听地址为 `localhost`。
    * 若 `port` 设置为 `0` ，则将禁用监控功能。仅在 ssh 退出后重启它。

* `-f` 使 autossh 在后台运行。

另外，autossh 还提供了一组环境变量来控制其行为, 这里仅介绍几个有代表性的，其可以 `man autossh`

* `AUTOSSH_FIRST_POLL` : 首次论询测试时间。
* `AUTOSSH_POLL` : 连接论询时间，默认 600 。若该值小于两次网络超时（默认 15 秒），则网络超将被调整为该值的 1/2
* `AUTOSSH_GATETIME` : 等待 ssh 连接成功建立的时间，默认 30 秒，超时表示首次运行失败，将退出 autossh 。若设为 0 ，则禁用该功能，通常用于启动时运行 autossh 。
* `AUTOSSH_MAXLIFETIME` : autossh 最长运行时间，达到该时间，autossh 将退出，并杀死 ssh 进程。
* `AUTOSSH_MAXSTART` : ssh 最大启动次数。默认-1，表示无限制。

## Ubuntu 配置方法 ##

以 SSH 动态代理（即 SSH 翻墙）为例:

### Ubuntu 12.04/14.04

init script 和 Upstart 都可以将 autossh 变成服务，然 Upstart 的 `respawn` 容错能力更强，它能在服务进程掉线，重新启动该服务。

创建 `/etc/init/autossh.conf` :

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

* `setuid` 和 `setgid` 为了让 autossh 运行在指定的用户和用户组上。
* `start on` 表示当 `eth0` 或 `wlan0` 激活时，启动 autossh ， `stop on` 反之。其目的为避免系统启动或网络掉线时，频繁尝试启动 autossh 。
* `sshproxy` 为 ssh 别名，须在 `setuid` 和 `setgid` 指定用户的 `~/.ssh/config` 中配置。

开始服务：

```bash
sudo start autossh
```

### Ubuntu 16.04/18.04/20.04

systemd 也能自动重启掉线的服务进程。

创建 `/etc/systemd/system/autossh.service` :

```
[Unit]
Description=SSH Proxy
After=network.target

[Service]
Type=simple
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE
User=like
Group=like
ExecStart=/usr/bin/autossh -M64000 -q -N -D localhost:12348 sshproxy

[Install]
WantedBy=multi-user.target
```

开启服务：

```bash
sudo systemctl enable autossh
sudo systemctl start autossh
```

## 更好的办法 ##

最近 OpenSSH 都支持选项 `ServerAliveInterval` 和 `ServerAliveCountMax` ，实际为建立在 SSH 协议上的心跳测试。当测试失败后， SSH 客户端进程将退出。通过 Upstart 的 respawn 功能重启 SSH 客户端进程，也能达到 autossh 目的。

仍以 SSH 动态代理为例：

### Ubuntu 12.04/14.04

创建 `/etc/init/sshproxy.conf` :

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

开始服务：

```bash
sudo start sshproxy
```

### Ubuntu 16.04/18.04/20.04

创建 `/etc/systemd/system/sshproxy.service` :

```
[Unit]
Description=SSH Proxy
After=network.target

[Service]
Type=simple
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE
User=like
Group=like
ExecStart=/usr/bin/ssh -oServerAliveInterval=300 -oServerAliveCountMax=2 -q -N -D localhost:12348 sshproxy
Restart=always
RestartSec=3s
StartLimitInterval=0

[Install]
WantedBy=multi-user.target
```

`Restart` , `RestartSec` 和 `RestartSec` 为当对等端 `sshd` 停止或网络异常时，服务 sshproxy 将退出并每 3 秒重启服务，直至连接上对等端 `sshd` . 参见 [Systemd service that is always restarted](https://selivan.github.io/2017/12/30/systemd-serice-always-restart.html)

开启服务：

```bash
sudo systemctl enable sshproxy
sudo systemctl start sshproxy
```

若存在多个 SSH 动态代理，则可模板化服务。

创建 `/etc/systemd/system/sshproxy@.service` :

```
[Unit]
Description=SSH Proxy
After=network.target

[Service]
Type=simple
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE
User=like
Group=like
ExecStart=/usr/bin/ssh -qN sshproxy-%i
Restart=always
RestartSec=3s
StartLimitInterval=0

[Install]
WantedBy=multi-user.target
```

为每个动态代理在 `~/.ssh/config` 中创建别名：

```
host sshproxy-hello
    HostName <hello ip>
    Port <port>
    User sshproxy
    ServerAliveInterval 300
    ServerAliveCountMax 2
    DynamicForward localhost:<socks port>
    IdentityFile ~/.ssh/sshproxy-hello/id_rsa
```

开启服务：

```bash
sudo systemctl enable sshproxy@hello
sudo systemctl start sshproxy@hello
```
