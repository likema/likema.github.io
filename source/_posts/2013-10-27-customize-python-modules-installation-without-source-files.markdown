---
layout: post
title: 定制无源码安装Python模块
comments: true
categories: [Python]
---

Python的[distutils](http://docs.python.org/2/library/distutils.html)和[setuptools](https://pypi.python.org/pypi/setuptools)都是为开源项目设计的，Python模块分发和安装都包含该模块的源代码。实际公司工作多为闭源项目，Python模块的安装是不能包核心源代码的。

过去对distutils和setuptools的一知半解，为了达到闭源的目的，我通过书写Makefile来编译Python源码为.pyc或.pyo，完全绕开distutils和setuptools的限制。权宜之计虽然解决了一时之急，然总是让我追求标准和完美的心感到不安。为此，最近我花了一些时间来阅读distutils文档和部分源代码，终于找到了相对地道的解决办法。

根据[Extending Distutils](http://docs.python.org/2/distutils/extending.html)的描述，继承distutils.cmd.Command的子类，如distutils.command.build\_py.build\_py，并重载已有的方法来达到扩展的目的。

根据[Creating a new Distutils command](http://docs.python.org/2/distutils/apiref.html#creating-a-new-distutils-command)描述子类必须定义如下方法：

* Command.initialize\_options()
* Command.finalize\_options()
* Command.run()
* Command.sub\_commands()

并且命令install由install\_lib和install\_headers等子命令构成。

我的目的不是扩展Distutils的install命令，而是改变其行为，避免其安装源码。其实，只需要改变install\_lib的行为就足够了。

类install\_lib存在于/usr/lib/python2.7/distutils/command/install\_lib.py文件中，它的方法run源码如下：
```python
    def run(self):
        # Make sure we have built everything we need first
        self.build()

        # Install everything: simply dump the entire contents of the build
        # directory to the installation directory (that's the beauty of
        # having a build directory!)
        outfiles = self.install()

        # (Optionally) compile .py to .pyc
        if outfiles is not None and self.distribution.has_pure_modules():
            self.byte_compile(outfiles)
```

与先编译再安装的直觉相反，编译生成pyc并不发生在build方法中，而是install方法执行后。所以，若重载build方法（实际调用build_py命令），则install和byte_compile都需要修改，工作量较大且复杂度较高。

直接能想到的办法是重载install方法，使其直接编译源码，并返回None，从而使byte_compile不会被执行。

```python
import os
from distutils.core import setup
from distutils.command.install_lib import install_lib
from distutils import log
from distutils.dep_util import newer
from py_compile import compile


class InstallLib(install_lib):
    def install(self):
        for root, dirs, files in os.walk(self.build_dir):
            current = root.replace(self.build_dir, self.install_dir)
            for i in dirs:
                self.mkpath(os.path.join(current, i))

            for i in files:
                file = os.path.join(root, i)
                cfile = os.path.join(current, i) + "c"
                cfile_base = os.path.basename(cfile)
                if self.force or newer(file, cfile):
                    log.info("byte-compiling %s to %s", file, cfile_base)
                    compile(file, cfile)
                else:
                    log.debug("skipping byte-compilation of %s", file)


setup(cmdclass={"install_lib": InstallLib}, name="HelloWorld", version="1.0")
```

虽然这样做达到了目的，然而仔细思考一下，更简单的办法是等安装完成后，删除目标目录的源码文件（这里仅给出InstallLib的实现，其余部分同上）：

```python
class InstallLib(install_lib):
    def run(self):
        self.build()
        outfiles = self.install()
        if outfiles is not None and self.distribution.has_pure_modules():
            self.byte_compile(outfiles)
            for i in outfiles:
                os.unlink(i)
```

如此非常简洁，实际只增加了2行代码，其余皆copy-paste。
