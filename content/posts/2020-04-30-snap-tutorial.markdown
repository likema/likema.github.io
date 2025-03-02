---
layout: post
title: Snap简介
date: 2020-04-30 19:40:24
comments: true
categories: [Ubuntu,Linux,Snap]
---

[Snap](https://snapcraft.io/) 是 Canonical 开发的包管理系统，默认安装于 Ubuntu 16.04 及其后的发行版本中。

优势：

* 自包含：不受限于发行版的系统库版本，且每个包之间不存在运行库依赖。
* 只读挂载：应用程序不能修改或删除，且不会污染系统应用程序或库。
* 回退：内置回退旧版本。
* 快照：内置备份和恢复应用数据。
* 版本新：相比发行版更新缓慢，其应用程序版本都比较新。

劣势主要为安装包占用较多存储空间。

以下主要以 LXD 的 snap 包为例。

## 一、安装 snapd

```bash
sudo apt install -y snapd
```

查看版本

```bash
snap version
```

## 二、安装 snap 包

为避免系统可能存在旧版本 LXD 与 snap 安装的最新版 LXD 冲突：

```bash
sudo apt remove --purge lxd lxd-client
```

然后：

```bash
sudo snap install lxd
```

默认 `stable` 频道，也可以指定 `edge` 频道：

```bash
sudo snap install --channel=edge lxd
```

安装后，可切换频道。

```bash
sudo snap switch --channel=stable lxd
```

相比 RPM 和 Debian 包需手动更新，snap 包将在后台自动更新。若需手动更新，则

```bash
sudo snap refresh lxd
```

snap 还能切换频道并更新

```bash
sudo snap refresh --channel=beta lxd
```

snap 应用程序位于 `/snap/bin` ，如： `/snap/bin/lxd`

为便于使用，可将该路径追加于 `~/.bashrc` 或 `~/.zshrc` 环境变量 `PATH` ，如：

```bash
export PATH=$PATH:/snap/bin
```

### 下载包

```bash
snap download lxd
```

### 离线安装

```bash
sudo snap ack lxd_17320.assert
sudo snap install lxd_17320.snap
```

### 设置 HTTP/S 代理

当 snap 所在机器无法直接访问网络时，需设置 HTTP 和 HTTPS 代理：

```bash
sudo snap set system proxy.http="http://<proxy_addr>:<proxy_port>"
sudo snap set system proxy.https="http://<proxy_addr>:<proxy_port>"
```

参考： [How to install snap packages behind web proxy on Ubuntu 16.04](https://askubuntu.com/questions/764610/how-to-install-snap-packages-behind-web-proxy-on-ubuntu-16-04)

### 移动镜像目录

若 `/var` 不在独立分区，随着镜像下载，根分区空间或将耗尽。

假设 HOME 独立于根分区，可将镜像目录移动至 `$HOME/.lxd/images`，并 `bind` 回来：

```bash
sudo snap stop lxd
sudo mkdir -m700 $HOME/.lxd
sudo mv /var/snap/lxd/common/lxd/images $HOME/.lxd
sudo mkdir -m700 /var/snap/lxd/common/lxd/images
echo "$HOME/.lxd/images /var/snap/lxd/common/lxd/images auto bind 0 2" | sudo tee -a /etc/fstab
sudo mount -a
sudo snap start lxd
```

## 三、搜索包

```bash
snap search <snapname>
```

也可通过浏览器在应用市场 [Snapcraft](https://snapcraft.io/) 上搜索需要的包（应用）。

## 四、列表已安装的包

### 列表所有包

```bash
snap list
```

输出：

```
Name      Version    Rev    Tracking       Publisher   Notes
core      16-2.44.3  9066   latest/stable  canonical✓  core
core18    20200311   1705   latest/stable  canonical✓  base
lxd       4.0.1      14804  latest/stable  canonical✓  -
```

若 `--all` ， 则列表包的所有版本 (revision)

### 列表指定包

```bash
snap list lxd
```

输出：

```
Name  Version  Rev    Tracking       Publisher   Notes
lxd   4.0.1    14804  latest/stable  canonical✓  -
```

## 五、回退版本

```bash
sudo snap revert lxd
```

若遇到当前版本bug，则可考虑回退程序。当前跟踪的 channel 不会因上一版本源于不同 channel 而改变。

* `snap refesh` 不会更新已回退的包，除非指定包名，如：`snap refresh lxd`
*  新版本发布，将继续自动更新已回退的包。

## 六、卸载 snap 包

```bash
sudo snap remove lxd
```

卸载旧版本（释放空间）。

```bash
sudo snap remove --revision=14709 lxd
```

## 七、启用/禁用 snap 包

为避免卸载和重装而禁用：

```bash
sudo snap disable lxd
```

反之：

```bash
sudo snap enable lxd
```


## 八、服务

### 列表

```bash
sudo snap services lxd
```

输出：

```
Service       Startup  Current   Notes
lxd.activate  enabled  inactive  -
lxd.daemon    enabled  active    socket-activated
```

### 启动、停止和重启

```bash
sudo snap stop lxd.daemon
sudo snap start lxd.daemon
sudo snap restart lxd.daemon
```

停止服务，并禁用自动启动：

```bash
sudo snap stop --disable lxd.daemon
```

开始服务，并启用自动启动：

```bash
sudo snap start --disable lxd.daemon
```

### 查看日志

```bash
sudo snap logs lxd
sudo snap logs lxd.daemon
sudo snap logs lxd -f # 类似tail -f
```

## 九、快照

### 创建

```bash
sudo snap save
```

输出：

```
Set  Snap      Age    Version    Rev    Size    Notes
1    core      42.2s  16-2.44.3  9066     124B  -
1    core18    42.2s  20200311   1705     123B  -
1    lxd       42.2s  4.0.1      14890   2187B  -
```

或指定包

```bash
sudo snap save lxd
```

若 `--no-wait`， 则后台运行。

### 列表

```bash
snap saved
```

或指定Set ID

```bash
snap saved --id=1
```

### 校验

```bash
snap check-snapshot 1
```

### 还原

```bash
snap restore 1
```

### 删除

```bash
snap forget 1
```

## 十、剖析

### 包的安装

相比 RPM 和 Debian 等传统安装包，通过解开来安装。

存于 `/var/lib/snapd` ，格式为 [squashfs](https://en.wikipedia.org/wiki/SquashFS) 的 snap 包，不直接解开，而是（只读）挂载至 `/snap/<snapname>/<revision>` 目录。如：

`lxd` 的 revision 14890 的包存储于 `/var/lib/snapd/snaps/lxd_14890.snap` :

```bash
mount | grep 14890
```

发现：

```
/var/lib/snapd/snaps/lxd_14890.snap on /snap/lxd/14890 type squashfs (ro,nodev,relatime)
```

另外， `/snap/<snapname>/current` 为当前版本挂载点，它为指向 `/snap/<snapname>/<revision>` 的符号链接。

而且，每个 snap 包含了不依赖系统库的完整的运行时库。

### 包的缓存

snap 为了加速二次安装，首次安装会将 snap 包缓存至 `/var/lib/snapd/cache` 。

目前为止， snap 未提供命令清楚缓存。若需 **释放空间** ，须手动删除该目录中的文件。

### 包的运行数据

`/var/snap` 存储每个包的运行数据（或元数据）。如：`/var/snap/lxd` 主要为 lxd 的元数据。

该目录或将消耗大量的存储空间，因受制于 AppArmor ，不能通过移动目录（至另外分区）和符号链接来释放空间，须 `mount --bind` 移动的目录。

### 包的应用程序

`/snap/bin` 存储指向包的应用程序（符号链接），如：

```bash
ls -l /snap/bin/lxd
```

看到 `lxd` 仅为 `snap` 的符号链接

```
lrwxrwxrwx 13 root 29 4月  12:10 /snap/bin/lxd -> /usr/bin/snap
```

### 快照

每个包快照用独立的 zip 文件存储，包含：

* `meta.json` : 描述快照内容、配置和校验码。
* `archive.tgz` : 包含系统数据。
* `user/<username>.tgz` : 包含每个系统的用户数据。

快照存储于 `/var/lib/snapd/snapshots`

## 参考

* [Snap Getting Started](https://snapcraft.io/docs/getting-started)
* [Disable snap core service on Ubuntu 18.04](http://landofnightandday.blogspot.com/2018/06/disable-snap-core-service-on-ubuntu-1804.html)
