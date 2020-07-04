---
layout: post
title: Shadowsocks-Rust简介
date: 2019-04-30 01:30:00
comments: true
categories: [Linux, Ubuntu, Proxy]
---

在《[Shadowsocks简介](https://www.malike.net.cn/blog/2016/11/29/shadowsocks-tutorial/)》中，我介绍了如何使用[Shadowsocks](https://shadowsocks.org/en/index.html)。但它不具备负载均衡集群能力，而结合HAProxy的集群配置复杂。

[shadowsocks-rust](https://github.com/shadowsocks/shadowsocks-rust)是Shadowsocks的Rust语言实现，它不仅具有传统Shadowsocks特性，而且还具有：

* **负载均衡** 多个Shadowsocks服务器的能力。
* 探测Shadowsocks服务器 **延迟** 的能力。

## 安装

Shadowsocks-Rust未提供DEB安装包。为了方便安装，可下载其[静态链接版本](https://github.com/shadowsocks/shadowsocks-rust/releases)

```bash
sudo tar atf shadowsocks-v1.7.0-nightly.x86_64-unknown-linux-musl.tar.xz -C /usr/local/bin
```
## 配置

以root用户创建目录/etc/shadowsocks-rust，编辑/etc/shadowsocks-rust/config.json：

```json
{
    "servers": [
        {
            "address": "127.0.0.1",
            "port": 1080,
            "password": "hello-world",
            "method": "aes-256-cfb"
            "timeout": 300
        },
        {
            "address": "127.0.0.1",
            "port": 1081,
            "password": "hello-kitty",
            "method": "aes-256-cfb"
        }
    ],
    "local_port": 8388,
    "local_address": "127.0.0.1"
}
```

* `address`为服务端地址。
* `server_port`为服务端监听端口。
* `password`为客户端和服务端预设的共享密码，它最好由安全密码生成器生成（如[LastPass](https://www.lastpass.com/)或[KeePass](http://keepass.info/)），且长度不小于6个字符。
* `timeout`为连接超时时间。
* `method`为加密算法，`aes-256-cfb`的安全性较好。
* 每个`servers`元素为一个Shadowsocks服务器配置。

因Ubuntu 14.04过保，下面仅以Ubuntu 16.04及以后版本为例。

以root用户创建/etc/systemd/system/shadowsocks-rust-local.service:

```ini
[Unit]
Description=Shadowsocks-Rust Custom Client Service.
Documentation=sslocal -h
After=network.target

[Service]
Type=simple
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE
User=nobody
Group=nogroup
ExecStart=/usr/local/bin/sslocal --log-without-time -c /etc/shadowsocks-rust/config.json

[Install]
WantedBy=multi-user.target
```

注册并启动服务：

```bash
sudo chown -R root:nogroup /etc/shadowsocks-rust
sudo chmod -R g-w,o-rwx /etc/shadowsocks-rust
sudo systemctl daemon-reload
sudo systemctl enable shadowsocks-rust-local
sudo systemctl start shadowsocks-rust-local
```

注：

* 为降低`sslocal`进程权限，以nobody用户和nogroup组运行它。
* 为防止密码泄漏，/etc/shadowsocks-rust/config.json仅root用户或nogroup组可读。
* 限制`sslocal`进程仅能监听socket.
