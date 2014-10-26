---
layout: post
title: Rinse简介——Debian/Ubuntu中创建RPM安装环境
date: 2014-07-15 20:34:58 +0800
comments: true
categories: [Linux, Debian, Ubuntu]
---

[Rinse](http://collab-maint.alioth.debian.org/rinse/)是一个Debian环境中创建RPM发行版本（如CentOS，Scientific Linux和openSUSE）的工具。你可以利用它轻松创建各种RPM发行版本的chroot环境。

以下基于Ubuntu 12.04 amd64，主要以创建CentOS 6 x86\_64为例。

## 安装Rinse ##

```sh
sudo apt-get install -y rinse
```

## 创建CentOS 6 ##

```sh
sudo rinse --distribution centos-6 --arch amd64 --directory centos-6
```

运行该命令将创建CentOS 6 amd64于当前工作目录的centos-6目录中。其中，

* --distribution指定发行版本，类似还可以centos-{4,5}， fedora-core-{4,5,6,7,8,9}和opensuse-{10.1,10.2,10.3,11.0,11.1,12.1}等。可以下述命令获取：

```sh
rinse --list-distributions
```

具体对应于/etc/rinse/*.packages的模板名，它们主要包含RPM包列表。换一句话说，你根据需要定制自己的模板。另一方面，你也可以通过--pkgs-dir指定不同于/etc/rinse的模板目录。

* --arch指定架构，amd64表示64位架构，i386表示32位架构。缺省为i386.
* --directory指定为安装目录，安装结束后便可以chroot该目录了。

另外，需要额外安装某些包，可以通过指定--add-pkg-list来完成。

## 配置RPM缓存 ##

rinse默认使用/var/rinse/cache作为缓存目录，它大大缩短了重复运行同样命令的时间。具体通过：

* --cache 0指禁用缓存，缺省为1
* --cache-dir指定不同于/var/rinse/cache作为缓存目录。
* --clean-cache指清楚缓存

## 定制安装后执行脚本 ##

--after-post-install, --before-post-install和--post-install顾名思义，需要指出的是--post-install默认执行/usr/lib/rinse/<distribution>/post-install.sh.

## 如何提高安装速度？ ##

通过修改/etc/rinse/rinse.conf中对应发行版的镜像地址可以加速安装，如CentOS 6 x86_64的镜像地址可以修改为

	http://centos.ustc.edu.cn/centos/6/os/x86_64/CentOS/

也可以通过--config指定不同于/etc/rinse/rinse.conf的配置文件。

若内网存在HTTP cache服务器（如Squid)，还可以设置环境变量http_proxy来缓存rpm以及加速安装，如：

```sh
sudo http_proxy=http://<http proxy address> rinse --distribution centos-6 --arch amd64 --directory centos-6
```
