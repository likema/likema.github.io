---
title: Python常量
date: 2013-11-03
categories: [Python]
---

与C/C++不同，Python在语法上并没有定义常量，尽管[PEP 8](/blog/2013/07/25/pep8-summary/)定义了常量的命名规范为大写字母和下划线组成。

在实际项目中，常量首次赋值后， 无法阻止其他代码对其进行修改或删除。

## 现存的办法 ##

幸运的是该问题在2001年就有人给出了解决方案[Constants in Python](http://code.activestate.com/recipes/65207-constants-in-python/)，基本内容如下：

```python
class _const:
    class ConstError(TypeError):
        pass

    def __setattr__(self, name, value):
        if self.__dict__.has_key(name):
            raise self.ConstError, "Can't rebind const instance attribute (%s)" % name

        self.__dict__[name] = value

import sys
sys.modules[__name__] = _const()
```

其大致含义为：

 * 通过\_const的\_\_setattr\_\_对象方法判断该对象是否存在属性name，若存在则抛出自定义异常ConstError，否则创建该属性。
 * 将\_const实例化的对象赋值sys.modules[\_\_name\_\_]，const模块被绑定成\_const对象。\_\_name\_\_在首次载入const过程中为'const'，而[sys.modules](http://docs.python.org/2.7/library/sys.html#sys.modules)是模块名与已加载模块的dict。

如何使用const模块呢？

```python
import const
const.magic = 23
```

若再次赋值const.magic，

```python
const.magic = 88
```

则将抛出ConstError的异常。

## 如何避免常量被删除？ ##

实际项目中，常量并不希望被其他代码删除。在\_const类中加入：

```python
    def __delattr__(self, name):
        if self.__dict__.has_key(name):
            raise self.ConstError, "Can't unbind const const instance attribute (%s)" % name

        raise AttributeError, "const instance has no attribute '%s'" % name
```

如此，删除已定义的常量（假设const.magic已经赋值）：

```python
del const.magic
```

则将抛出ConstError的异常。

## 配置文件与常量 ##

实际项目中，为了让应用程序灵活部署，一般会用配置文件存储应用程序的各种参数；而这些参数通常都以常量语义存在于应用程序中。

在上述代码基础上，根据应用程序的实际情况将常量划分为3类：

* 特殊含义的数值或字符串，如 ETC_FSTAB = "/etc/fstab" 。其作用为
    * 避免程序中到处出现类似特殊值，因为人为输入特殊值的低级错误将耗费不必要的调试/测试时间
    * 另一方面, 神秘数值（magic number），如 LUN_BLOCK_SIZE = 4096，将影响程序的可读性。
* 应用程序参数的默认值，如 URLOPEN_DEFAULT_TIMEOUT = 15 。其作用主要为配置文件参数的默认值。
* 通过配置文件载入的参数，如通过[ConfigParser.SafeConfigParser](http://docs.python.org/2/library/configparser.html#ConfigParser.SafeConfigParser)解析形如ini文件的参数。

第1和2类常量作为\_const类的类属性，第3类常量可以在\_\_init\_\_方法中初始化。如：

```python
class _const:
    ETC_FSTAB = "/etc/fstab"
    LUN_BLOCK_SIZE = 4096
    URLOPEN_DEFAULT_TIMEOUT = 15

    def __init__(self):
        conf = ConfigParser.SafeConfigParser()
        conf.read(self.CONF_PATH)

        try:
            self.URLOPEN_TIMEOUT = conf.getint("DEFAULT", "urlopen_timeout")
        except:
            self.URLOPEN_TIMEOUT = self.URLOPEN_DEFAULT_TIMEOUT

    ... ...

```

某些情况下，应用程序可能并不希望const模块在被外部调用时绑定新属性（常量），实现如下：

```python
    def __init__(self):
        # Constant definition area

        _const.__setattr__ = _const._setattr_impl

    def _setattr_impl(self, name, value):
        raise self.ConstError, "Can't bind const instance attribute (%s)" % name
```

其原理为\_const对象初始化完成后将\_\_setattr\_\_设置为禁止绑定属性的实现。若在\_const类中实现\_\_setattr\_\_为禁止绑定属性，则\_\_init\_\_也将无法初始化（绑定）对象属性。

## 存在的小问题 ##

该问题并不影响该解决方案的使用，具体请参考:

http://stackoverflow.com/questions/5365562/why-is-the-value-of-name-changing-after-assignment-to-sys-modules-name
