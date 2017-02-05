---
layout: post
title: nvm简介——Debian/Ubuntu中管理多版本Node.js
comments: true
categories: [nodejs, Debian, Ubuntu]
---

[nvm](https://github.com/creationix/nvm)是管理Node.js版本的工具，它支持在多个Node.js版本间切换。

## 一、安装nvm

```bash
git clone https://github.com/creationix/nvm.git ~/.nvm
cd ~/.nvm
git checkout `git describe --abbrev=0 --tags`
```

激活nvm

```bash
. $NVM_DIR/nvm.sh
```

为了每次登录后自动激活nvm，需要将`NMV_DIR`、`nvm.sh`和补齐加入bash的~/.bashrc（或zsh的~/.zshrc）

```bash
export NVM_DIR=~/.nvm
[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"
[ -r $NVM_DIR/bash_completion ] && . $NVM_DIR/bash_completion
```

## 二、nvm常用命令

### 列表可安装的Node.js版本

```bash
nvm ls-remote
```

除了Node.js官方版本，还支持io.js

### 安装指定版本的Node.js

```bash
nvm install 6.2.1
```

它会自动下载指定版本的Node.js二进制包（不需要编译源码），安装在~/.nvm/versions/node

### 卸载指定版本的Node.js

```bash
nvm uninstall 6.2.1
```

### 设置shell的Node.js版本
```bash
nvm use 6.2.1
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
nvm install v6.2.1 --reinstall-packages-from=5.0
```

### .nvmrc

它存储在工程根目录中，用于记录该工程依赖的Node.js版本

```bash
echo 6.2.1 > .nvmrc
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
