---
layout: post
title: pyenv简介——Debian/Ubuntu中管理多版本Python
comments: true
categories: [Python, Debian, Ubuntu]
---

[pyenv](https://github.com/yyuu/pyenv)是管理Python版本的工具，它支持在多个Python版本间切换。

## 一、安装pyenv

```bash
git clone https://github.com/yyuu/pyenv.git ~/.pyenv
```

将`PYENV_ROOT`和`pyenv init`加入bash的~/.bashrc（或zsh的~/.zshrc）

```bash
echo 'export PATH=~/.pyenv/bin:$PATH' >> ~/.bashrc
echo 'export PYENV_ROOT=~/.pyenv' >> ~/.bashrc
echo 'eval "$(pyenv init -)"' >> ~/.bashrc
```

## 二、pyenv常用命令
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

```bash
pyenv install 3.5.1
pyenv rehash
```


它会自动下载并编译指定版本的Python源码，这需要系统安装：

```bash
sudo apt-get install -y build-essential zlib1g-dev libssl-dev
```

还可选择安装：

```bash
sudo apt-get install libsqlite3-dev libbz2-dev  libreadline-dev
```

安装完成后：

* 源码（如~/Python-3.5.1.tar.gz）缓存在.pyenv/cache目录中，在安装完后可手动删除。
* Python版本安装在~/.pyenv/versions目录中。

### 卸载指定版本的Python

```bash
pyenv unstall 3.5.1
```

### 设置shell的Python版本

```bash
pyenv shell 3.5.1
```

等同于

```bash
export PYENV_VERSION=3.5.1
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
pyenv virtualenv 3.5.1 aiohttp-virtual-env
```

* 创建aiohttp-virtual-env之前，须先安装Python 3.5.1（通过系统或pyenv安装）。
* aiohttp-virtual-env存储在~/.pyenv/versions/3.5.1/envs目录中，且在~/.pyenv/versions目录中建立同名符号链接。

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

## 四、配置Upstart脚本

若python程序须要通过Upstart启动，则其Upstart脚本可以类似：

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



