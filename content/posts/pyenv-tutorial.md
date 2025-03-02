---
title: pyenv简介——Debian/Ubuntu中管理多版本Python
date: 2016-05-21
categories: [Python, Debian, Ubuntu]
---

[pyenv](https://github.com/yyuu/pyenv)是管理Python版本的工具，它支持在多个Python版本间切换。

## 一、安装

```bash
git clone https://github.com/pyenv/pyenv.git ~/.pyenv
cd ~/.pyenv
git checkout `git describe --abbrev=0 --tags`
```

将`PYENV_ROOT`和`pyenv init`加入bash的~/.bashrc（或zsh的~/.zshrc）

```bash
echo 'export PATH=~/.pyenv/bin:$PATH' >> ~/.bashrc
echo 'export PYENV_ROOT=~/.pyenv' >> ~/.bashrc
echo 'eval "$(pyenv init -)"' >> ~/.bashrc
echo 'export PYTHON_BUILD_MIRROR_URL="http://mirrors.sohu.com/python/"' >> ~/.bashrc
```

## 二、常用命令

### 列表可安装的Python版本

```python
pyenv install -l
```

除了Python官方版本，还支持

* anaconda
* ironpython
* jython
* miniconda
* pypy
* stackless

### 安装指定版本的Python

安装过程，实际为下载并编译指定版本的Python源码，故需系统安装：

```bash
sudo apt-get install -y build-essential zlib1g-dev libssl-dev libffi-dev
```

还可选择安装：

```bash
sudo apt-get install libsqlite3-dev libbz2-dev liblzma-dev libreadline-dev
```

然后：

```bash
pyenv install 3.6.8
pyenv rehash
```

* 源码（如Python-3.6.8.tar.gz）缓存在 `.pyenv/cache` 目录中，在安装完后可手动删除。
* Python版本安装在~/.pyenv/versions目录中。

### 卸载指定版本的Python

```bash
pyenv uninstall 3.6.8
```

### 设置shell的Python版本

```bash
pyenv shell 3.6.8
```

等同于

```bash
export PYENV_VERSION=3.6.8
```

清除`PYENV_VERSION`

```bash
pyenv shell --unset
```

## 三、安装pyenv-virtualenv

pyenv-virtual是pyenv的插件，它支持管理多个virtualenv

```bash
git clone https://github.com/yyuu/pyenv-virtualenv.git ~/.pyenv/plugins/pyenv-virtualenv
echo 'eval "$(pyenv virtualenv-init -)"' >> ~/.bashrc
```

### 创建virtualenv

```bash
pyenv virtualenv 3.6.8 aiohttp-virtual-env
```

* 创建aiohttp-virtual-env之前，须先安装Python 3.6.8（通过系统或pyenv安装）。
* aiohttp-virtual-env存储在~/.pyenv/versions/3.6.8/envs目录中，且在~/.pyenv/versions目录中建立同名符号链接。

### 删除virtualenv

```bash
pyenv uninstall aiohttp-virtual-env
```

### 列表virtualenv

```bash
pyenv virtualenvs
```

### 激活/禁用virtualenv

```bash
pyenv activate aiohttp-virtual-env
pyenv deactivate
```

### 迁移virtualenv

将指定virtualenv，迁移至另一virtualenv，须安装pyenv插件pyenv-pip-migrate：

```bash
git clone https://github.com/pyenv/pyenv-pip-migrate.git ~/.pyenv/plugins/pyenv-pip-migrate
```

然后：

```bash
pyenv migrate aiohttp-virtual-env hello-virtual-env
```

## 四、升级

```bash
cd $PYENV_ROOT
git fetch origin
git checkout `git describe --abbrev=0 --tags`
```

更简单的办法为安装pyenv插件pyenv-update：

```bash
git clone https://github.com/pyenv/pyenv-update.git ~/.pyenv/plugins/pyenv-update
```

它不仅能更新pyenv，还能更新pyenv所有已安装的插件：

```bash
pyenv update
```

## 五、配置Upstart脚本

若Python程序须通过Upstart启动，则其Upstart脚本可以类似：

```
# service name

description "service description ..."

respawn

setuid <username>
setgid <group>

env PYENV_ROOT=/home/<username>/.pyenv
env PATH=/home/<username>/.pyenv/bin:/sbin:/usr/sbin:/bin:/usr/bin
env PYENV_VERSION=<python version or virtualenv name>

chdir <app dir>

script
        eval "$(pyenv init -)"
        exec ./<app>
end script
# vim: ts=4 sw=4 sts=4 ft=upstart
```

或

```
# service name

description "service description ..."

respawn

setuid <username>
setgid <group>

env PYENV_ROOT=/home/<username>/.pyenv
env PATH=/home/<username>/.pyenv/shims:/home/<username>/.pyenv/bin:/sbin:/usr/sbin:/bin:/usr/bin
env PYENV_VERSION=<python version or virtualenv name>

chdir <app dir>

exec ./<app>
# vim: ts=4 sw=4 sts=4 ft=upstart
```

* `username`为服务运行的用户名，通常为`PYENV_ROOT`所属用户。
* `group`为服务运行的组名，通常为`PYENV_ROOT`所属组。
* `PYENV_VERSION`为Python版本号或virtualenv的名字。
* `app dir`为Python程序的目录。
* `app`为Python程序或启动脚本。

## 六、配置Systemd脚本

若Python程序须通过Systemd启动，则其Systemd脚本类似：

```ini
...

[Service]
...
User=<username>
Group=<group>
Environment="PATH=/home/<username>/.pyenv/shims:/home/<username>/.pyenv/bin:/sbin:/usr/sbin:/bin:/usr/bin"
Environment="PYENV_ROOT=/home/<username>/.pyenv"
Environment="PYENV_VERSION=<python version or virtualenv name>"
WorkingDirectory=<app dir>
ExecStart=<app dir>/<app>
...
```

* `username`为服务运行的用户名，通常为`PYENV_ROOT`所属用户。
* `group`为服务运行的组名，通常为`PYENV_ROOT`所属组。
* `PYENV_VERSION`为Python版本号或virtualenv的名字。
* `app dir`为Python程序的目录。
* `app`为Python程序或启动脚本。

## 七、制作Docker镜像

若不希望使用Python的官方Docker镜像，可利用pyenv创建镜像：

```Dockerfile
FROM ubuntu:bionic
MAINTAINER ...

ENV DEBIAN_FRONTEND noninteractive
ENV PYTHON_BUILD_MIRROR_URL http://mirrors.sohu.com/python/
ENV PYENV_ROOT /root/.pyenv
ENV PYENV_VERSION 3.6.8
ENV PATH $PYENV_ROOT/shims:$PYENV_ROOT/bin:$PATH

RUN set -eux; \
    apt-get update; \
    savedAptMark="$(apt-mark showmanual)"; \
    apt-get install -y git wget build-essential zlib1g-dev libssl-dev libffi-dev libsqlite3-dev libbz2-dev liblzma-dev libreadline-dev; \
    git clone --depth 1 https://github.com/pyenv/pyenv.git $PYENV_ROOT; \
    cd $PYENV_ROOT; \
    git checkout `git describe --abbrev=0 --tags`; \
    pyenv install 3.6.8; \
    apt-mark auto '.*' > /dev/null; \
    [ -z "$savedAptMark" ] || apt-mark manual $savedAptMark; \
    apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
    apt-get clean; \
    rm -rf /var/lib/apt/lists $PYENV_ROOT/cache/* /tmp/*
```
