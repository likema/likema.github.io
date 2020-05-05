---
layout: post
title: LXD简介（一）
date: 2020-05-05 17:19:21
comments: true
categories: [Ubuntu,Linux,Snap,LXD]
---

[LXD](https://linuxcontainers.org/lxd/introduction/) 是基于LXC容器的管理程序（hypervisor），它由开发 Ubuntu 的公司 Canonical 创建和维护。

它由3个组建构成：

* `lxd` ：系统守护进程，它导出能被本地和网络访问的 RESTful API
* `lxc` ：客户端命令行，它能跨网络管理多个容器主机。
* `nova-compute-lxd` ： OpenStack Nova 插件，它使 OpenStack 如虚拟机一般，管理容器。

## 一、安装

因 Ubuntu 16.04 LXD 2.x 和 Ubuntu 18.04 LXD 3.x 版本都较旧，且 Ubuntu 20.04 放弃 `apt` 安装 LXD 。

请据 [Snap简介](/blog/2020/04/30/snap-tutorial/) 安装 LXD

为避免每次 `sudo lxc` ，可将 `lxd` 组加入当前（非root）用户的附加组：

```bash
sudo usermod -aG lxd $USER
```

## 二、初始化

```bash
sudo lxd init
```

输出：

```bash
Would you like to use LXD clustering? (yes/no) [default=no]:

# 配置存储池
Do you want to configure a new storage pool? (yes/no) [default=yes]:
Name of the new storage pool [default=default]:
Name of the storage backend to use (btrfs, dir, lvm, zfs, ceph) [default=zfs]: dir

Would you like to connect to a MAAS server? (yes/no) [default=no]:

# 创建虚拟网络
Would you like to create a new local network bridge? (yes/no) [default=yes]:
What should the new bridge be called? [default=lxdbr0]:
What IPv4 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]:
What IPv6 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]:

# LXD 服务的网络配置
Would you like LXD to be available over the network? (yes/no) [default=no]: yes
Address to bind LXD to (not including port) [default=all]:
Port to bind LXD to [default=8443]:
Trust password for new clients:
```

参看： [How to install LXD on Ubuntu](https://snapcraft.io/install/lxd/ubuntu)

### 修改密码

```bash
lxc config set core.trust_password '<password>'
```

## 三、镜像

LXD 默认提供3个远程镜像：

* `ubuntu`: Ubuntu 稳定版镜像
* `ubuntu-daily`: Ubuntu 每日构建的镜像
* `images`: [其它发行版本的镜像](https://images.linuxcontainers.org/)，主要包括:
  * Alpine
  * Archlinux
  * Centos/Oracle
  * Debian
  * Fedora
  * Gentoo
  * OpenSUSE
  * OpenWRT
  * Ubuntu

### 列表本地镜像

```bash
lxc image list
```

### 列表Ubuntu镜像

```bash
lxc image list ubuntu:
```

### 列表其他镜像

```bash
lxc image list images:
```

### 注册镜像

因上述镜像都在国外，复制镜像速度非常慢。可注册清华大学的镜像来加速使用：

```bash
lxc remote add tuna-images https://mirrors.tuna.tsinghua.edu.cn/lxc-images/ --protocol=simplestreams --public
```

以下主要以`tuna-images`为例。

### 复制镜像

```bash
lxc image cp tuna-images:ubuntu/20.04 local: --copy-aliases --auto-update --public
```

* `--copy-aliases` : 复制所有镜像别名。 每个镜像可能有多个别名，如 Ubuntu 20.04 的镜像的别名为 `20.04` 和 `focal` 等。
* `--auto-uppdate` : 自动更新镜像
* `--public` : 公开镜像，让其它机器可以复制它。后面将进一步介绍如何公开 LXD 端口。


## 四、容器

### 创建容器，但不启动

默认为 64-bit 容器：

```bash
lxc init tuna-images:ubuntu/20.04 focal
```

若须 32-bit 容器，则

```bash
lxc init tuna-images:ubuntu/20.04/i386 focal
```

### 创建容器，并启动容器

```bash
lxc launch tuna-images:ubuntu/20.04 focal
```

相当于

```bash
lxc init tuna-images:ubuntu/20.04 focal
lxc start focal
```

直接从远程镜像初始化或启动，存在如下缺点：

* 每次初始化或启动，可能会从远程下载镜像（若存在更新），造成初始化或启动速度缓慢。
* 未复制任何别名至本地，不利于复用。如 `lxc image ls` ：

```
+-----------------------+--------------+--------+--------------------------------------+--------------+-----------+---------+-----------------------------+
|         ALIAS         | FINGERPRINT  | PUBLIC |             DESCRIPTION              | ARCHITECTURE |   TYPE    |  SIZE   |         UPLOAD DATE         |
+-----------------------+--------------+--------+--------------------------------------+--------------+-----------+---------+-----------------------------+
|                       | 36e7b3c6bdea | yes    | Ubuntu focal amd64 (20200504_07:42)  | x86_64       | CONTAINER | 97.40MB | May 5, 2020 at 8:07am (UTC) |
+-----------------------+--------------+--------+--------------------------------------+--------------+-----------+---------+-----------------------------+
```

故最佳实践为首先复制镜像，然后初始化或启动：

```bash
lxc image cp tuna-images:ubuntu/20.04 local: --copy-aliases --auto-update --public
lxc launch ubuntu/20.04 focal
```

### 启动/停止容器

```bash
lxc start focal
lxc stop focal
```

### 列表容器

```bash
lxc list
```

或 快速列表

```bash
lxc list --fast
```

### 查看容器详细信息

```bash
lxc info focal
```

### 运行命令

```bash
sudo lxc exec focal -- /bin/bash
```

### 上传/下载文件

```bash
lxc file pull focal/etc/hosts .
lxc file push /etc/hosts focal/tmp/tmp
```

## 五、虚拟机

LXD 3.19 开始支持创建虚拟机:

简单的说，所有容器相关的命令加`--vm`。

### 复制镜像

若类似复制容器命令：

```bash
lxc image copy tuna-images:ubuntu:20.04 local: --copy-aliases --auto-update --public --vm
```

则可能 **覆盖** 本地容器 ubuntu 20.04 的别名，造成后者 **无别名**：

```
+-----------------------+--------------+--------+-------------------------------------+--------------+-----------------+----------+-----------------------------+
|         ALIAS         | FINGERPRINT  | PUBLIC |             DESCRIPTION             | ARCHITECTURE |      TYPE       |   SIZE   |         UPLOAD DATE         |
+-----------------------+--------------+--------+-------------------------------------+--------------+-----------------+----------+-----------------------------+
| ubuntu/focal (7 more) | 1bb3c2f730c5 | yes    | Ubuntu focal amd64 (20200504_07:42) | x86_64       | VIRTUAL-MACHINE | 231.06MB | May 5, 2020 at 8:08am (UTC) |
+-----------------------+--------------+--------+-------------------------------------+--------------+-----------------+----------+-----------------------------+
|                       | 36e7b3c6bdea | yes    | Ubuntu focal amd64 (20200504_07:42) | x86_64       | CONTAINER       | 97.40MB  | May 5, 2020 at 8:07am (UTC) |
+-----------------------+--------------+--------+-------------------------------------+--------------+-----------------+----------+-----------------------------+
```

为了避免上述问题，自定义镜像别名，如以 `vm/` 作为镜像别名前缀：

```bash
lxc cpi tuna-images:ubuntu/20.04 local: --auto-update --public --vm --alias vm/ubuntu/focal --alias vm/ubuntu/20.04
```

### 创建虚拟机

```bash
lxc launch tuna-images:ubuntu/20.04 focal-vm --vm --profile default --profile vm
```

Ubuntu 16.04 默认内核 4.4 ，将遇到

```
Creating focal-vm
Starting focal-vm
Error: Failed to run: modprobe vhost_vsock: modprobe: FATAL: Module vhost_vsock not found in directory /lib/modules/4.4.0-176-generic
Try `lxc info --show-log local:focal-vm` for more info
```

须安装 4.15 (HWE) 内核，并重启：

```bash
sudo apt-get install --install-recommends linux-generic-hwe-16.04
```

## 六、快照

### 创建只读快照

```bash
lxc snapshot focal focal-s0
```

注：无列表快照的直接操作，只能通过获取容器的详细信息 `lxc info` 来获取的快照名字。

### 还原快照

```bash
lxc restore focal focal-s0
```

### 删除快照

```bash
lxc delete focal/focal-s0
```

## 七、别名

### 创建别名

模仿 Docker 删除镜像的 `docker rmi` ， 创建 `lxc rmi` ：

```bash
lxc alias add rmi 'image rm'
```

类似：

```bash
lxc alias add lsi 'image ls'
lxc alias add cpi 'image cp'
lxc alias add infoi 'image info'
```

### 列表别名

```bash
lxc alias ls
```
### 删除别名

```bash
lxc alias rm infoi
```

## 八、剖析

lxd将数据存放于 `/var/snap/lxd/common/lxd` ：

 * `images`: 存放镜像文件
 * `lxc`: 存放容器
 * `lxd.db`：lxd元数据数据库，基于sqlite3
 * `server.crt`：服务器证书
 * `server.key`：服务器密钥
 * `unix.socket`：lxd监听的本地套接口


## 参考

* [LXD Getting started - command line](https://linuxcontainers.org/lxd/getting-started-cli/)
* [LXD in 4 Easy Steps](https://ubuntu.com/blog/lxd-in-4-easy-steps)
* [How to use virtual machines in LXD](https://blog.simos.info/how-to-use-virtual-machines-in-lxd/)
* [Trying LXD virtual machines](https://discuss.linuxcontainers.org/t/trying-lxd-virtual-machines/6182)
* [Using command aliases in LXD to exec a shell](https://blog.simos.info/using-command-aliases-in-lxd-to-exec-a-shell/)
