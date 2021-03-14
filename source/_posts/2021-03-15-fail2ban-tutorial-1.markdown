---
layout: post
title: Fail2Ban 简介 （一）
date: 2021-03-15 07:15:05
comments: true
categories: fail2ban
---

Fail2Ban 是入侵检测软件框架，保护计算机免受暴力破解（brute-force attack）。以 Python 语言编写，能运行于具有包（packet）控制或防火墙的 POSIX 系统，如 iptables 或 TCP Wrapper.

鉴于互联网针对 VPS 暴力破解 SSH 越来越频繁，以下以 Ubuntu 为例，介绍如何利用 Fail2Ban 有效禁止 SSH 破解。

## 一、安装

```bash
sudo apt-get install -y fail2ban
```

## 二、配置

Fail2Ban 配置文件格式 [INI](https://en.wikipedia.org/wiki/INI_file)，存于 `/etc/fail2ban` 目录：

* `fail2ban.conf` : `fail2ban` 程序运行的日志和数据库等参数。
* `jail.conf` : 禁止（ban）相关参数。
* `filter.d/*` : `jail.conf` 中涉及“节”（section，如 `[sshd]` ）的过滤器，由正则表达式构成。
* `action.d/*` : `jail.conf` 中涉及“节”（section，如 `[sshd]` ）的处理器。

它们皆为安装文件，直接修改将导致后续升级 **无法自动合并** 配置文件。Fail2Ban 提供了自定义配置文件的机制:

* `fail2ban.conf` 可依此通过 `fail2ban.d/*` 和 `fail2ban.local` 来重定义相关选项。
* `jail.conf` 可依此通过 `jail.d/*` 和 `jail.local` 来重定义相关选项。

默认安装，`/etc/fail2ban/jail.d/defaults-debian.conf` **已启用** `sshd` 的 jail

通常，除 `jail.conf` 外，不需要改变配置。以下着重介绍 `jail.conf` 中的参数，它们不仅是默认（全局）参数（隶属于 `[DEFAULT]`），而且可在具体 `jail` 中重定义（如 `[sshd]`）。

### 常用参数

* `ignoreip` : 忽略不 IP 地址（CIDR 格式）或机器名，以空格分隔。
* `bantime` : 主机被禁止时长，默认 600 秒。
* `maxretry` : 在 `findtime` 时间窗口中，允许主机认证失败次数。达到最大次数，主机将被禁止。
* `findtime` : 查找主机认证失败的时间窗口。 **不意味** 着每隔 `findtime` 时间扫描一次日志。

高版本 Fail2ban 支持 `s` （秒）, `m` （分）和 `d` （天）作为时间单位，如 `10m` 和 `1d`

### `backend`

默认 `auto` ，支持 4 种文件修改的监控后端：

* `pynotify` : [inotify](https://en.wikipedia.org/wiki/Inotify) 的 Python 绑定。在 Debian/Ubuntu 中，对应 `python3-pyinotify` 包。
* `gamin` :  KDE 和 GNOME 等桌面使用的文件监控系统，通常基于 [inotify](https://en.wikipedia.org/wiki/Inotify) 实现。 在 Debian/Ubuntu 中，对应 `gamin` 包。
* `polling` : 周期性扫描相关系统日志。相对于其它几种后端，或将占用 **更多** CPU 时间。
* `systemd` :  通过 Systemd 的 Python 绑定来访问 systemd 日志，`logpath` 将无效。 在 Debian/Ubuntu 中，对应 `python3-systemd` 包。
* `auto` : 依此尝试 `pyinotify` , `gamin` 和 `polling`

通常，在安装 `fail2ban` 安装包时，将自动安装 `iptables` , `whois` , `python3-pyinotify` 和 `python3-systemd` ，即启用 `pyinotify` 后端。除非 APT 配置 `APT::Install-Recommends "false";` （如：`/etc/apt/apt.conf`）或 `--no-install--recommends` 。

### 最佳实践

建议将 `bantime` 设为 `86400` （即 1 天）。如 `jail.local` :

```
[DEFAULT]
bantime = 86400
findtime = 600
maxretry = 3
```

或

```
[sshd]
bantime = 86400
findtime = 600
maxretry = 3
```

修改后，须重载 `fail2ban` ：

```bash
sudo service fail2ban reload
```

`/var/log/fail2ban.log` 将出现类似：

```
2021-02-20 17:04:59,580 fail2ban.observer       [2615]: INFO    Observer start...
2021-02-20 17:04:59,583 fail2ban.database       [2615]: INFO    Connected to fail2ban persistent database '/var/lib/fail2ban/fail2ban.sqlite3'
2021-02-20 17:04:59,583 fail2ban.jail           [2615]: INFO    Creating new jail 'sshd'
2021-02-20 17:04:59,593 fail2ban.jail           [2615]: INFO    Jail 'sshd' uses pyinotify {}
2021-02-20 17:04:59,596 fail2ban.jail           [2615]: INFO    Initiated 'pyinotify' backend
2021-02-20 17:04:59,597 fail2ban.filter         [2615]: INFO      maxLines: 1
2021-02-20 17:04:59,611 fail2ban.filter         [2615]: INFO      maxRetry: 3
2021-02-20 17:04:59,611 fail2ban.filter         [2615]: INFO      findtime: 600
2021-02-20 17:04:59,611 fail2ban.actions        [2615]: INFO      banTime: 86400
2021-02-20 17:04:59,611 fail2ban.filter         [2615]: INFO      encoding: UTF-8
2021-02-20 17:04:59,611 fail2ban.filter         [2615]: INFO    Added logfile: '/var/log/auth.log' (pos = 34274, hash = 5cdc6285962a0352611a54aa860667fc35ededc1)
2021-02-20 17:04:59,614 fail2ban.jail           [2615]: INFO    Jail 'sshd' started
2021-02-20 17:08:12,674 fail2ban.server         [2615]: INFO    Reload all jails
2021-02-20 17:08:12,674 fail2ban.server         [2615]: INFO    Reload jail 'sshd'
2021-02-20 17:08:12,674 fail2ban.filter         [2615]: INFO      maxLines: 1
2021-02-20 17:08:12,674 fail2ban.filter         [2615]: INFO      maxRetry: 3
2021-02-20 17:08:12,675 fail2ban.filter         [2615]: INFO      findtime: 600
2021-02-20 17:08:12,675 fail2ban.actions        [2615]: INFO      banTime: 86400
2021-02-20 17:08:12,675 fail2ban.filter         [2615]: INFO      encoding: UTF-8
2021-02-20 17:08:12,675 fail2ban.server         [2615]: INFO    Jail 'sshd' reloaded
2021-02-20 17:08:12,675 fail2ban.server         [2615]: INFO    Reload finished.
```

## 三、查看状态

```bash
sudo fail2ban-client status sshd
```

输出类似：

```
Status for the jail: sshd
|- Filter
|  |- Currently failed: 2
|  |- Total failed:     5343
|  `- File list:        /var/log/auth.log
`- Actions
   |- Currently banned: 178
   |- Total banned:     1354
   `- Banned IP list:   ...
```

列表 iptables 的规则：

```
iptables -S
```

输出类似：

```
-P INPUT ACCEPT
-P FORWARD ACCEPT
-P OUTPUT ACCEPT
-N fail2ban-ssh
-A INPUT -p tcp -m multiport --dports 22 -j fail2ban-ssh
-A fail2ban-nginx-http-auth -j RETURN
-A fail2ban-ssh -s <IP 1> -j REJECT --reject-with icmp-port-unreachable
-A fail2ban-ssh -s <IP 2> -j REJECT --reject-with icmp-port-unreachable
...
-A fail2ban-ssh -j RETURN
```
