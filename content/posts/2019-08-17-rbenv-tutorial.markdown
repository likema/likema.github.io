---
layout: post
title: rbenv简介——Debian/Ubuntu中管理多版本Ruby
date: 2019-08-17 16:40:54
comments: true
categories: [Ruby, Debian, Ubuntu]
---

## 一、安装

```bash
git clone git://github.com/rbenv/rbenv.git ~/.rbenv
git clone git://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
```

在国内，为了加速下述安装：

```bash
git clone https://github.com/andorchen/rbenv-taobao-mirror.git ~/.rbenv/plugins/rbenv-taobao-mirror
```

激活rbenv

```bash
eval "$(~/.rbenv/bin/rbenv init)"
```

为了每次登录后自动激活rbenv，需将`NMV_DIR`、`nvm.sh`和补齐加入bash的~/.bashrc（或zsh的~/.zshrc）

```bash
export PATH=~/.rbenv/shims:~/.rbenv/bin:$PATH
eval "$(rbenv init -)"
```

验证是否安装正确：

```bash
curl -fsSL https://github.com/rbenv/rbenv-installer/raw/master/bin/rbenv-doctor | bash
```

或

```bash
wget -qO- https://github.com/rbenv/rbenv-installer/raw/master/bin/rbenv-doctor | bash
```

输出类似：

```
Checking for `rbenv' in PATH: /root/.rbenv/bin/rbenv
Checking for rbenv shims in PATH: OK
Checking `rbenv install' support: /root/.rbenv/plugins/ruby-build/bin/rbenv-install (ruby-build 20190615-9-gf3f4193)
Counting installed Ruby versions: none
  There aren't any Ruby versions installed under `/root/.rbenv/versions'.
  You can install Ruby versions like so: rbenv install 2.2.4
Checking RubyGems settings: OK
Auditing installed plugins: OK
```

## 二、常用命令

### 列表可安装的Ruby版本

```bash
rbenv install -l
```

除了Ruby官方版本，还支持RBX和JRuby等。

### 安装指定版本的Ruby

安装过程，实际为下载并编译指定版本的Ruby源码，故需系统安装：

```
sudo apt-get install -y make gcc libssl-dev libreadline-dev zlib1g-dev
```

然后：

```bash
rbenv install 2.6.3
```

Ruby版本安装在 `~/.rbenv/versions` 目录中。

### 卸载指定版本的Ruby

```bash
rbenv uninstall 2.6.3
```

### 设置shell的Ruby版本

```bash
rbenv shell 2.6.3
```

等同于

```bash
export RBENV_VERSION=2.6.3
```

清除`RBENV_VERSION`

```bash
rbenv shell --unset
```

## 三、升级

```bash
cd ~/.rbenv
git pull
cd ~/.rbenv/plugins/ruby-build
git pull
cd ~/.rbenv/plugins/rbenv-taobao-mirror
git pull
```

## 四、配置Systemd脚本

若Ruby程序须通过Systemd启动，则其Systemd脚本类似：

```ini
...

[Service]
...
User=<username>
Group=<group>
Environment="PATH=/home/<username>/.rbenv/shims:/home/<username>/.rbenv/bin:/sbin:/usr/sbin:/bin:/usr/bin"
Environment="RBENV_ROOT=/home/<username>/.rbenv"
Environment="RBENV_VERSION=<ruby version>"
WorkingDirectory=<app dir>
ExecStart=<app dir>/<app>
...
```

* `username` 为服务运行的用户名，通常为 `RBENV_ROOT` 所属用户。
* `group` 为服务运行的组名，通常为 `RBENV_ROOT` 所属组。
* `RBENV_VERSION` 为Ruby版本号。
* `app dir` 为Ruby程序的目录。
* `app` 为Ruby程序或启动脚本。

## 五、制作Docker镜像

若不希望使用Ruby的官方Docker镜像，可利用rbenv创建镜像：

```Dockerfile
FROM ubuntu:bionic
MAINTAINER ...

ENV DEBIAN_FRONTEND noninteractive
ENV RBENV_ROOT /root/.rbenv
ENV RBENV_VERSION 2.6.3
ENV PATH $RBENV_ROOT/shims:$RBENV_ROOT/bin:$PATH
ENV RUBY_CFLAGS -O3

RUN set -eux; \
    apt-get update; \
    savedAptMark="$(apt-mark showmanual)"; \
    apt-get install -y git wget make gcc libssl-dev libreadline-dev zlib1g-dev; \
    echo "gem: --no-rdoc --no-ri" > ~/.gemrc; \
    git clone --depth 1 https://github.com/rbenv/rbenv.git $RBENV_ROOT; \
    git clone --depth 1 https://github.com/rbenv/ruby-build.git $RBENV_ROOT/plugins/ruby-build; \
    git clone --depth 1 https://github.com/andorchen/rbenv-taobao-mirror.git $RBENV_ROOT/plugins/rbenv-taobao-mirror; \
    CONFIGURE_OPTS="--disable-install-doc" MAKE_OPTS="-j`grep -c '^processor' /proc/cpuinfo`" rbenv install $RBENV_VERSION; \
    gem sources --add https://gems.ruby-china.com/ --remove https://rubygems.com/; \
    gem install bundler; \
    bundle config mirror.https://rubygems.com https://gems.ruby-china.com; \
    apt-mark auto '.*' > /dev/null; \
    [ -z "$savedAptMark" ] || apt-mark manual $savedAptMark; \
    apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
    apt-get clean; \
    rm -rf /var/lib/apt/lists /var/tmp/* /tmp/* $RBENV_ROOT/versions/*/lib/ruby/gems/*/cache/*.gem
```

* 在国内，为了加速安装gem，上述gem和bundle设置国内镜像
* `RUBY_CFLAGS` 为 `-O3` 可编译优化的Ruby程序。在某些情况，可提高应用20-30%的运行效率。
* 可针对指定C扩展（如ffi），设置编译优化标志：

```
bundle config build.ffi --with-cflags="$RUBY_CFLAGS"
```
