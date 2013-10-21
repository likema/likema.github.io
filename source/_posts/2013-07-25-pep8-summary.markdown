---
layout: post
title: PEP 8总结
comments: true
categories: Python
---

下述内容主要源于[PEP 8 -- Style Guide for Python Code](http://www.python.org/dev/peps/pep-0008/)。

## 最大的行长度 ##

* 所有行不超过79个字符。
* docstring或comment应不超过72字符

## 补齐 ##

* 每个补齐级别为4个空格。
* 当一行操作最大行长度时，应尽可能按照各种括号作为纵向对齐的参照物（可以适当增加括号），如：

```python
from bottle import (get, post, delete, error, run, default_app, HTTPError,
                    request, response, static_file)

# Aligned with opening delimiter
foo = long_function_name(var_one, var_two,
                         var_three, var_four)

# More indentation included to distinguish this from the rest.
def long_function_name(
        var_one, var_two, var_three,
        var_four):
    print(var_one)
```

* 多行情况下，关闭括号可以出现在一行开始，如：

```python
my_dict = {
    'hello': 'foo',
    'world': 'bar',
}

my_list = [
    1, 2, 3,
    4, 5, 6,
]
result = some_function_that_takes_arguments(
    'a', 'b', 'c',
    'd', 'e', 'f',
)
```

实际工作中，不同开发语言存在不同的补齐风格要求。强烈建议不要将补齐设置在.vimrc中，而将

```python
# vim: ts=4 sw=4 sts=4 et:
```

追加在每个python源文件的最后一行，从而保证vim打开该文件时满足4个空格补齐的要求。

## 空行 ##

顶层函数或类定义的间隔为2行。
类的方法定义的间隔为1行。

## 源文件编码 ##

Python核心代码应为UTF-8（或ASCII，在Python 2中）。
源文件若在Python 2中用ASCII或在Python 3中用UTF-8，则不应出现编码声明。

```python
# -*- coding: utf-8 -*-
```

## 导入 ##

* 每行仅导入一个模块，但每行可以导入一个模块的多个函数:

```python
import os
import sys
from subprocess import Popen, PIPE
```

* 保证所有import在文件的头部，仅在模块注释后面，并先于模块的全局变量和常量。
* import应按下列顺序分组：
** 标准库的导入
** 相关第3方库的导入
** 本地应用程序/库的导入
每个分组间空1行
将类似__all__定义放在所有import后面
* 绝对导入优于相对导入
* 避免导入模块所有的内容（通配方式， from <module> import *)，在__init__.py中导出内部API除外。

## 表达式和语句中的空格 ##

* 括号前后不允许有空格
* 操作符号（如=，>和+=等）前后各1个空格
* :和,之前不允许有空格，之后仅1个空格
* 函数默认参数的等号（=）前后不允许空格

## 注释 ##

* 为所有公开模块、函数、类和方法写docstring。
* 非公开方法的comment应出现在def行之后
* [PEP-257](http://www.python.org/dev/peps/pep-0257/)描述良好的docstring惯例。多行docstring的第1行后应跟着1个空白行。
* 单行docstring可保持关闭的"""在同一行。

## 命名规范 ##

尽管Python库代码的命名存在一些混乱，新模块和包（包括第3方框架）应满足下列规范。但已有库若风格不同，应保持原来的内部一致性。

* 单下划线开头（如：_single_leading_underscore）弱内部使用，类似from M import *不导入类似符号。
* 单下划线结尾（如：single_trailing_underscore_）用于与Python关键字冲突的情况下，如classs_。
* 双下划线开头（如：__double_leading_underscore）用于类属性。
* 双下划线开头和结构用于特殊对象或属性，如__init__, __import__或__file__。多为语言定义，避免发明类似名字。
* 避免使用小写L、大写O和大写I作为单字符变量名。
* 模块名应简短、全小写，可包含下划线；包名类似，但不鼓励包含下划线。
* C/C++实现的扩展模块应伴随着提供如面向对象等高级接口的Python模块存在，且其名字以下划线开头（如_socket）
* 类名应为驼峰词（CapWords），内部使用的类以下划线开头。
* 异常名与类名一样，且应以Error结尾（若该异常确为一个错误）。
* 函数名应为小写加下划线。
* 总是使用self作为实例方法的第1个参数，总是使用cls作为类方法的第1个参数
* 常量名应为大写加下划线（如MAX_OVERFLOW）

