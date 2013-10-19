---
layout: post
title: MakeProjectCreator (MPC) 简介
comments: true
categories:
---

[Makefile, Project, and Workspace Creator (MPC)](http://www.ociweb.com/products/mpc) 原始OCI为网络通信库ACE开发的跨平台工程文件生成工具，它可以生成很多种工程文件，例如Windows平台的Borland Developer Studio, Borland C++ Build 2007, Borland Makefile，Microsoft eMbedded VC3, Microsoft Visual C++ 6/7/71/8/9/10, nmake；Linux平台的automake, make等。

下面将以Windows平台的nmake和vc9为例介绍MPC的使用方法。首先，我们需要下载

 1. [ActivePerl](http://www.activestate.com/activeperl)，双击执行安装;
 1. [MPC](http://download.ociweb.com/MPC/)，以解压到C:盘为例，设置系统环境变量MPC_ROOT为C:\MPC，以及将%MPC_ROOT%追加系统环境变量PATH。当然，你也可以通过双击执行C:\MPC\registry.pl将生成工程文件的功能注册到右键菜单中。

## 一、一个Hello World工程 ##

首先，建立目录helloworld，并且在这个目录下且创建helloworld.cpp，内容如下：

```cpp
int main ()
{
    std::cout << "Hello World!" << std::endl;
}
```

然后，创建MPC工程文件helloworld.mpc，内容如下：

```
project {
	exename = helloworld
}
```

最后，打开Visual Studio 2008 Command Prompt（也可以用VS2003或2005的），进入helloworld的目录, 然后输入

    mwc.pl -type nmake

就生成了格式为nmake的Makefile，它默认包含两个配置Debug和Release。让我们编译一下Debug版本：

    nmake CFG="Win32 Debug"

你会发现helloworld.exe就生成在当前目录。

或许你希望debug和release生成的exe分别放在debug和release目录，修改一下MPC文件

```
project {
	exename = helloworld
	specific (nmake, vc6, vc7, vc71, vc8, vc9, vc10) {
		Debug::install = debug
		Release::install = release
	}
}
```

或许你更喜欢用IDE，让我们生成VS2008的工程文件吧：

	mwc.pl -type vc9

## 二、一个简单dll工程 ##

首先，建立目录hellodll，并且在这个目录下且创建hellodll.h，内容如下：

```c
#ifndef HELLO_DLL_H
#define HELLO_DLL_H

#ifdef HELLO_BUILD_DLL
#   define HELLO_Export __declspec(dllexport)
#else
#   define HELLO_Export __declspec(dllimport)
#endif /* HELLO_BUILD_DLL */

#ifdef __cplusplus
extern "C" {
#endif /* __cplusplus */

HELLO_Export void print_hello ();

#ifdef __cplusplus
} /* extern "C" */
#endif /* __cplusplus */

#endif /* HELLO_DLL_H */
```

然后，创建文件hellodll.cpp，内容如下：

```c
#include "hellodll.h"

void print_hello ()
{
    std::cout << "Hello World!" << std::endl;
}
```

最后，创建文件hellodll.mpc，内容如下

```
project {
	sharedname = hello
	dynamicflags += HELLO_BUILD_DLL
	specific(nmake, vc6, vc7, vc71, vc8, vc9, vc10) {
		Debug::dllout = debug
		Release::dllout = release
	}
}
```

之后，执行如下命令，可以生成Debug和Release的dll。注意Debug的dll是hellod.dll。

    mwc.pl -type nmake
    nmake
    nmake CFG="Win32 Release"

接下来，我们做一个工程来驱动这个dll

## 三、hello dll的测试工程 ##

首先，在原来的目录hellodll下建立文件hellowtest.cpp，内容如下：

```c
#include "hellodll.h"

int main ()
{
    print_hello ();
    return 0;
}
```

然后，创建文件hellotest.mpc，内容如下：

```
project {
	after += hellodll
	exename = hellotest
	libs += hello

	Source_Files {
		hellotest.cpp
	}

	specific(nmake, vc6, vc7, vc71, vc8, vc9, vc10) {
		Debug::libpaths = debug
		Debug::install = debug
		Release::libpaths = Release
		Release::install = Release
	}
}
```

注意到helloworld.mpc和hellotest.mpc的差异吗？后者的after语句表示hellotest.mpc依赖于hellodll.mpc，必须在其后编译。另外，我在后者增加了Source_Files块，顾名思义，其作用是指定工程源文件(cpp和c)列表。如果没有它，MPC默认会将hellotest.mpc所在目录的所有源文件作为这个工程的源文件，这显然不是我们希望看到的；同理，需要在hellodll.mpc增加Source_Files块，修改后的内容如下：

```
project {
	sharedname = hello
	dynamicflags += HELLO_BUILD_DLL

	Source_Files {
		hellodll.cpp
	}

	specific (nmake, vc6, vc7, vc71, vc8, vc9, vc10) {
		Debug::dllout = debug
		Release::dllout = Release
	}
}
```

类似Source_Files块，还有Header_Files, Inline_Files和Resoure_Files等，具体可以查看MPC官方文档。
最后，重新执行一下

    mwc.pl -type nmake
    nmake
    nmake CFG="Win32 Release"

关于MPC更多的内容和细节可以下载[OCI的《TAO开发指南》的MPC样章](http://downloads.ociweb.com/MPC/docs/html/MakeProjectCreator.html)

{{ page.date | date_to_string }}
