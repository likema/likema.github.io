---
layout: post
title: nvm简介——Debian/Ubuntu中管理多版本Node.js
comments: true
categories: [NodeJS, Debian, Ubuntu]
---

[nvm](https://github.com/creationix/nvm)是管理Node.js版本的工具，它支持在多个Node.js版本间切换。

## 一、安装

```bash
git clone https://github.com/creationix/nvm.git ~/.nvm
cd ~/.nvm
git checkout `git describe --abbrev=0 --tags`
```

激活nvm

```bash
. ~/.nvm/nvm.sh
```

为了每次登录后自动激活nvm，需将`NMV_DIR`、`nvm.sh`和补齐加入bash的~/.bashrc（或zsh的~/.zshrc）

```bash
export NVM_DIR=~/.nvm
[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"
[ -r $NVM_DIR/bash_completion ] && . $NVM_DIR/bash_completion
```

在国内，为了加速下述安装，可在bash的~/.bashrc（或zsh的~/.zshrc）加入：

```bash
export NVM_NODEJS_ORG_MIRROR=https://npm.taobao.org/mirrors/node
```

## 二、常用命令

### 列表可安装的Node.js版本

```bash
nvm ls-remote
```

除了Node.js官方版本，还支持io.js

### 安装指定版本的Node.js

```bash
nvm install 10.16.2
```

它会自动下载指定版本的Node.js二进制包（不需要编译源码），安装在~/.nvm/versions/node

通常，最好安装最近的长周期版本：

```bash
nvm install --lts
```

### 卸载指定版本的Node.js

```bash
nvm uninstall 10.16.2
```

### 设置shell的Node.js版本
```bash
nvm use 10.16.2
```

它将Node.js指定版本的bin路径加入PATH.

还原环境变量PATH

```bash
nvm deactivate
```

### 迁移npm至新版本的Node.js

```
nvm install node --reinstall-packages-from=node
```

或

```
nvm install v10.16.2 --reinstall-packages-from=10.16.0
```

### .nvmrc

它存储在工程根目录中，用于记录该工程依赖的Node.js版本

```bash
echo 10.16.2 > .nvmrc
```

进入工程目录（当前目录），运行

```bash
nvm use
```

将根据.nvmrc指定shell的Nodejs版本


## 三、升级nvm

```bash
cd $NVM_DIR
git fetch origin
git checkout `git describe --abbrev=0 --tags`
```

升级完成后，需要重新激活nvm

```bash
. $NVM_DIR/nvm.sh
```

## 四、制作Docker镜像

若不希望使用NodeJS的官方Docker镜像，可利用nvm创建镜像：

```Dockerfile
FROM ubuntu:bionic
MAINTAINER ...

ENV DEBIAN_FRONTEND noninteractive
ENV NVM_NODEJS_ORG_MIRROR=https://npm.taobao.org/mirrors/node
ENV NODE_VERSION 10.16.2
ENV PATH /root/.nvm/versions/node/v$NODE_VERSION/bin:$PATH

RUN set -eux; \
    apt-get update; \
    apt-get install --no-install-recommends -y wget ca-certificates; \
    wget -O- https://raw.githubusercontent.com/creationix/nvm/v0.33.11/install.sh | bash; \
    apt-get remove --purge -y wget ca-certificates; \
    apt-get autoremove --purge -y; \
    apt-get clean; \
    rm -rf /var/lib/apt/lists/* /root/.nvm/.cache
```
