---
layout: post
title: Ubuntu 11.10 VirtualBox的Host-only网卡上外网和DHCP永久地址
comments: true
categories:
---

VirtualBox支持各种虚拟网络：NAT, Bridge Adapter, Internal Network和Host-only Adapter等。其中Bridged Adapter最为简单和常用，它几乎是0配置，直接桥接有线或无线物理网卡就可以与互联网通信。

然而，我工作场所内部网和家里内部网的网段不相同，DHCP存在一定租赁时间，如果使用Bridged Adapter并DHCP获取IP地址的时候，虚拟机地址经常会改变。为此，我将笔记本电脑的VirtualBox虚拟机都修改为Host-only Adapter模式。


一个问题是Host-only Adapter（网段为192.168.56.0/24）默认不能与互联网通信。google之后发现网上早有人遇到类似问题，他们给出的解决办法是在/etc/rc.local中加入：

	iptables -t nat -I POSTROUTING -s 192.168.56.0/24 -j MASQUERADE

另一个问题是VirtualBox内置DHCP的IP租赁时间设置，也无法将MAC地址与IP地址静态绑定，这造成虚拟机IP地址每隔一段时间改变一次，给使用带来诸多不方便。另一方面，我也不想静态设置IP地址，因为如果这样做，我必须每安装一次虚拟机都要重新设置IP地址。

以前就听说过dnsmasq，不仅集成DNS、DHCP和TFTP功能，而且占用资源很少，设置也相对简单。

1. 安装dnsmasq

	sudo apt-get install dnsmasq


1. 打开/etc/dnsmasq.conf，针对vboxnet0配置DHCP。

```
interface=vboxnet0

# 192.168.56.1是默认网关（host机器的vboxnet0地址）
# 208.67.222.222和208.67.220.220是DNS地址(这里使用了OpenDNS)
dhcp-option=vboxnet0,option:dns-server,192.168.56.1,208.67.222.222,208.67.220.220

# 192.168.56.2和192.168.56.254为分配地址范围
# infinite表示IP永远不过期
dhcp-range=vboxnet0,192.168.56.2,192.168.56.254,infinite
```

1. 重启动dnsmasq

	sudo service dnsmasq restart

当然，dnsmasq也支持MAC地址与IP地址静态绑定。比如，在/etc/dnsmasq.conf中针对MAC地址08:00:27:81:51:85，分配机器名vbox-xp，分配IP地址192.168.56.2

```
dhcp-host=vbox-xp,08:00:27:81:51:85,192.168.56.2
```

最后，不要忘了重启动dnsmasq。
