---
layout: post
title: Flake8简介
comments: true
categories: [Python]
---

[Flake8](http://flake8.readthedocs.org/)包装了下列工具：

* [PyFlakes](https://launchpad.net/pyflakes)：静态检查Python代码逻辑错误的工具。
* [pep8](http://pep8.readthedocs.org/en/latest/)： 静态检查[PEP 8](/blog/2013/07/25/pep8-summary/)编码风格的工具。
* [Ned Batchelder’s McCabe script](http://nedbatchelder.com/blog/200803/python_code_complexity_microtool.html)：静态分析Python代码复杂度的工具。

它综合上述三者的功能，在简化操作的同时，还提供了扩展开发接口。

## 安装 ##

这里仅介绍Ubuntu的安装方法，其他安装方法见Flake8官网。

* 添加ppa:cjohnston/flake8。Ubuntu 12.04官方源仅提供pep8的包，而该PPA不仅提供了最新的python-flake8包，还提供最新的pep8包。Ubuntu 13.10和14.04默认已经提供最新的pep8和python-flake8，所以可以跳过这一步。

```
sudo apt-add-repository ppa:cjohnston/flake8
sudo apt-get update
sudo apt-get -y --force-yes dist-upgrade
```

* 安装python-flake8

```sh
sudo apt-get install python-flake8
```

## 使用 ##

* 递归检查当前目录的所有Python文件：

```sh
flake8 .
```

* 检查指定文件

```sh
flake8 foo.py bar.py
```

* 通过setup.py检查工程的所有Python文件：

```sh
python setup.py flake8
```

为了保证其在其他环境中正确运行，需要将flake8增加到setup_requires中，例如：

```python
setup(
    name="project",
    packages=["project"],

    setup_requires=[
        "flake8"
    ]
)
```

* 由于默认禁用代码条件复杂度检查，需要通过--max-complexity激活该功能：

```sh
flake8 --max-complexity 12 .
```

该功能对于发现代码过度复杂非常有用，根据Thomas J. McCabe, Sr（[Cyclomatic complexity](https://en.wikipedia.org/wiki/Cyclomatic_complexity)的创造者）研究，代码复杂度不宜超过10，而Flake8官网建议值为12。

## 配置 ##

* 用户相关的配置存在~/.config/flake8中，如：

```ini
[flake8]
max-complexity=12
```

个人感觉除了代码复杂度因子（max-complexity）外，其他参数的默认值已经很好，基本不需要再作配置。

* 工程相关的设置，可以存放在工程顶级目录的tox.ini或setup.cfg，格式与用户相关的配置一致。

## 与Git整合 ##

在.git/hooks目录中，创建Git的pre-commit钩子脚本，Flake8可以对每次提交的代码进行检查。该脚本如下：

```python
#!/usr/bin/python

import sys
from flake8.run import git_hook

COMPLEXITY = 12
STRICT = True

if __name__ == '__main__':
    sys.exit(git_hook(complexity=COMPLEXITY, strict=STRICT))
```

若strict为True，任何warning都将阻挡提交。否则（或缺省），warning仅会被打印到标准输出。

## 与vim整合 ##

这里仅介绍vim插件vim-flake8的安装和配置

* 安装vim插件pathogen：

```sh
mkdir -p ~/.vim/autoload ~/.vim/bundle
curl -Sso ~/.vim/autoload/pathogen.vim \
    https://raw.github.com/tpope/vim-pathogen/master/autoload/pathogen.vim
```

* 添加下列配置至~/.vimrc中：

```vim
execute pathogen#infect()
syntax on
filetype plugin indent on
```

* 安装vim-flake8：

```sh
cd ~/.vim/bundle
git clone git://github.com/nvie/vim-flake8.git
```

至此，当vim打开Python源码后，按F7就会执行Flake8对当前文件进行检查。

## 插件 ##

Flake8相比pep8的优势在于其良好的扩展性，pep8 1.4.6尚未支持命名规范的检查，却已有人开发Flake8的插件[pep8-naming](https://github.com/flintwork/pep8-naming)来弥补这个缺陷。

pep8-naming处于早期开发阶段，尚无人为其制作deb包。我花时间做了deb包，并上传到我的ppa:likemartinma/python上。通过下述步骤可以轻松安装它：

```sh
sudo add-apt-repository ppa:likemartinma/python
sudo apt-get update
sudo apt-get -y --force-yes dist-upgrade
sudo apt-get install pep8-naming
```

由于Python部分核心库的函数命令存在“历史遗留”问题，与PEP 8并不保持完全一致，如[xml.parsers.expat.xmlparser.StartElementHandler](http://docs.python.org/2/library/pyexpat.html#xml.parsers.expat.xmlparser.StartElementHandler)，这给pep8-naming带来一定的误报困扰。

解决的办法是这样的代码行追加 # noqa 的注释，从避免flake8发出类似warning。

## 存在的问题 ##

由于pep8尚未支持docstring规范的检查，也没有相关Flake8的插件。目前仅能用[pep257](https://github.com/GreenSteam/pep257)来完成docstring规范的检查。期待pep257早日衍生成Flake8的插件。
