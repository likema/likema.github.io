---
title: Rinse简介——Debian/Ubuntu中创建RPM安装环境
date: 2014-07-15 20:34:58 +0800
categories: [Linux, Debian, Ubuntu]
---

[Rinse](http://collab-maint.alioth.debian.org/rinse/) 是 Debian 创建 RPM 发行版本（如 CentOS ，Scientific Linux 和 openSUSE）的工具。它可以被用于创建各种 RPM 发行版本的 chroot 环境。

以下基于Ubuntu 12.04 amd64，主要以创建CentOS 6 x86\_64为例。

## 安装Rinse ##

```sh
sudo apt-get install -y rinse
```

## 创建CentOS 6 ##

```sh
sudo rinse --distribution centos-6 --arch amd64 --directory centos-6
```

运行该命令将创建 CentOS 6 amd64 于当前工作目录的 `centos-6` 目录中。其中，

* `--distribution` 指定发行版本，类似还可以centos-{4,5}， fedora-core-{4,5,6,7,8,9}和opensuse-{10.1,10.2,10.3,11.0,11.1,12.1}等。可以下述命令获取：

```sh
rinse --list-distributions
```

具体对应于 `/etc/rinse/*.packages` 的模板名，它们主要包含 RPM 包列表。换一句话说，你根据需要定制自己的模板。另一方面，你也可以通过 `--pkgs-dir` 指定不同于 `/etc/rinse` 的模板目录。

* `--arch` : 指定架构，amd64 表示 64 位架构， i386 表示 32 位架构。缺省为 i386 .
* `--directory` : 指定为安装目录，安装结束后便可以 chroot 该目录了。

另外，需要额外安装某些包，可以通过指定 `--add-pkg-list` 来完成。

## 配置RPM缓存 ##

Rinse 默认缓存目录为 `/var/rinse/cache` ，它极大缩短了重复运行同样命令的时间：

* `--cache` : `0` 指禁用缓存，缺省为 `1` .
* `--cache-dir` : 指定缓存目录。
* `--clean-cache` ：清除缓存。

## 定制安装后执行脚本 ##

`--after-post-install` , `--before-post-install` 和 `--post-install` 顾名思义，需要指出的是 `--post-install` 默认执行 `/usr/lib/rinse/<distribution>/post-install.sh` .

## 如何提高安装速度？ ##

通过修改 `/etc/rinse/rinse.conf` 中对应发行版的镜像地址可以加速安装，如CentOS 6 x86_64的镜像地址可以修改为

	http://centos.ustc.edu.cn/centos/6/os/x86_64/CentOS/

也可以通过 `--config` 指定配置文件。

若内网存在 HTTP cache 服务器（如 Squid )，还可以设置环境变量 `http_proxy` 来缓存 RPM 以及加速安装，如：

```sh
sudo http_proxy=http://<http proxy address> rinse --distribution centos-6 --arch amd64 --directory centos-6
```
