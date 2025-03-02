---
layout: post
title: 为tolua的deb包作一个补丁
comments: true
categories: [Lua, Debian, Linux]
---

近日在学习tolua时发现在Kubuntu 11.10 amd64平台下将其链接到so时报告如下错误：

>relocation R_X86_64_32 against '.rodata' can not be used when making a shared object; recompile with -fPIC

为此，我决定在其原deb基础上加一个补丁，这样生成的新包可以安装到其他开发机器上，省去了每次重编译tolua的重复劳动。


为构建和修改deb安装必要的工具（配置quilt）：

	sudo apt-get install build-essential debhelper quilt

下载libtolua-dev的源码，创建补丁add-fpic-O2-for-amd64.patch，并将config文件加入其中：

	apt-get source libtolua-dev
	cd tolua-5.1.3
	mkdir -p debian/patches
	quilt new add-fpic-O2-for-amd64.patch
	quilt add config

将config文件中，如下内容

```makefile
CFLAGS = -g $(WARN) $(INC)
CPPFLAGS = -g $(WARN) $(INC)
```

替换为

```makefile
CFLAGS = -fPIC -O2 -pipe -g $(WARN) $(INC)
CPPFLAGS = -fPIC -O2 -pipe  -g $(WARN) $(INC)
CFLAGS = -fPIC -O2 -pipe -g $(WARN) $(INC)
```

生成补丁add-fpic-O2-for-amd64.patch

	quilt refresh

为补丁增加描述信息

	quilt header -e

其具体内容如下：

	Description: Fix relocation R_X86_64_32 against '.rodata' error for shared object.
	Author: Like Ma <likemartinma@gmail.com>

	  * config: add -fPIC -O2 -pipe to CFLAGS and CPPFLAGS

通过执行

	dch -R

在debian/changelog顶部增加日志信息：

	tolua (5.1.3-2) unstable; urgency=low

	  * Fix relocation R_X86_64_32 against '.rodata' error for shared object.

	-- Like Ma <likemartinma@gmail.com> Mon, 31 Oct 2011 15:23:09 +0800

构建libtolua-dev_5.1.3-2_amd64.deb:

	dpkg-buildpackage -d -uc -us -rfakeroot
