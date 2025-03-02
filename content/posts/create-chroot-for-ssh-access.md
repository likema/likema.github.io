---
title: 创建通过SSH访问的chroot
date: 2014-08-07 18:09:48 +0800
categories: [Linux, SSH]
---

我的工作环境是Ubuntu，然而经常需要CentOS来编译或测试。一般存在3种解决办法：

* 创建VirtualBox或KVM虚拟机：
	* 优点：部署容易，且可以运行各种应用（如Oracle）。
	* 缺点：运行速度相对LXC和chroot慢，特别是I/O.
* 创建CentOS的LXC：
	* 优点：运行速度快，且具有独立IP，可以通过SSH访问。
	* 缺点: 需要修改启动脚本。
* 创建CentOS的chroot：
	* 优点：不需要修改任何配置。
	* 缺点：无法直接通过SSH访问，且需要root权限才能运行chroot。

## 创建CentOS的chroot ##

具体参看[Rinse简介](/blog/2014/07/15/rinse-tutorial/)

## 创建SSH用户（或组） ##

在当前OS环境中，创建用户centos6.

```sh
sudo useradd -m -s /bin/bash centos6
```

获取centos6的uid和gid

```sh
id -u centos6
id -g centos6
```

在chroot环境中创建同名用户，且保持uid和gid相同。

```sh
sudo chroot /var/lib/centos-6 /bin/bash
groupadd -g <gid> centos6
useradd -s /bin/bash -m -u <uid> -g centos6 centos6
```

## 配置sshd ##

将下述内容追加至/etc/ssh/sshd_config

```
Match group centos6
	ChrootDirectory /var/lib/centos-6
```

确保/var/lib/centos-6的每一级目录的属主为root，且其他用户或组没有写权限。

然后，重启ssh

```
service ssh restart
```

这样保证centos6组的用户登录ssh时，chroot至/var/lib/centos-6目录中。

## 配置dev, proc和sysfs ##

chroot环境中rpm安装和卸载的前/后置脚本依赖dev, proc和sysfs，否则可能将造成安装和卸载错误。

在/etc/fstab中，增加proc和sysfs的挂载选项

```
proc /var/lib/centos-6/proc proc  defaults      0 0
none /var/lib/centos-6/sys  sysfs defaults      0 0
/dev /var/lib/centos-6/dev  none  defaults,bind 0 0
```

## 配置ssh密钥登录 ##

chroot的ssh密钥登录与常规情况的唯一区别在于公钥应存放在当前OS环境（而非chroot环境）的~/.ssh/authorized_keys，因为sshd在执行chroot之前需要检查公钥是否正确。

因此，本例中应该存放在当前OS的/home/centos-6/.ssh/authorized_keys.
