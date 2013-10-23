---
layout: post
title: git-buildpackage示例（一）
comments: true
categories: [Git, Debian]
---

在《为tolua的deb包作一个补丁》中，我介绍了如何利用quilt为Ubuntu已有包做一个补丁包的办法。可以看出quilt具有一定的版本管理能力，然而与流行版本管理系统相比，功能较弱也不灵活。

从[Debian New Maintainers' Guide](http://www.debian.org/doc/manuals/maint-guide/)中，了解到deb包的制作和维护管理已经与现有流行版本管理系统结合在了一块，其中一款工具为git-buildpackage，它将包制作和维护，特别是第三方补丁包维护，与git紧密的结合了起来。

下面仍然以tolua的补丁制作为例，一步一步展示git-buildpackage的基本操作。

* 安装必要的工具：

```sh
sudo apt-get install git-buildpackage build-essential debhelper quilt
```

* 下载libtolua-dev的源码（建立upstream目录单独存放Ubuntu的deb源码包是为了保证清洁和正确）：

```sh
mkdir upstream
apt-get source libtolua-dev
```

* 导入upsteam的dsc文件（将生成与目录upstream同级的目录tolua）：

```sh
cd ..
git-import-dsc upstream/tolua_5.1.3-1.dsc
cd tolua
```

这时，运行

```sh
git log --format=%d:%s
```

输出：

```plain
 (HEAD, debian/5.1.3-1, master):Imported Debian patch 5.1.3-1
 (upstream/5.1.3, upstream):Imported Upstream version 5.1.3
```

从下至上，首条提交导入了tolua 5.1.3的源码，次条提交导入了deb包维护者的deb包文件(debian/*)；并且建立了upstream和master两个分支，标签upstream/5.1.3位于upstream分支上，标签debian/5.1.3-1位于master分支头部。

此外，upstream分支用于维护源码作者的发布版本更新情况，master分支用于维护deb包描述文件及其补丁文件。git-buildpackage工具集的正确运行将依赖于标签upstream/5.1.3和debian/5.1.3-1，不能随意删改。

* 导入quilt patches到patch queue中——创建patch-queue/master分支，并将debian/patches/*逐一变成该分支的提交，并自动切换到该分支上：

```sh
gbp-pq import
```

* 执行make后发现构建目标libtolua.a的生成目录lib不存在，这是git只针对文件做版本，所以upstream导入git时，该目录被忽略了。为此，我将src/lib/Makefile中

```makefile
$T: $(OBJS)
	$(AR) $@ $(OBJS)
	$(RANLIB) $@
```

修改为：

```makefile
$T: $(OBJS)
	mkdir -p $(@D)
	$(AR) $@ $(OBJS)
	$(RANLIB) $@
```

这样，它将在每次构建该目标时，创建该目标所在目录。更进一步不难发现，src/bin/tolua_lua.o和src/bin/toluabind.c为受版本控制的中间文件，将影响构建的正确运行。为此，删除这两个文件并提交日志。

```sh
git rm -f src/bin/tolua_lua.o src/bin/toluabind.c
git commit -a -m "mkdir for tolua lib archive and remove temp files"
```

此时，可以正确make该工程了。

* 修复x86_64链接问题，将config文件中，如下内容

```makefile
CFLAGS= -g $(WARN) $(INC)
CPPFLAGS= -g $(WARN) $(INC)
```

替换为

```makefile
CFLAGS= -fPIC -O2 -pipe -g $(WARN) $(INC)
CPPFLAGS= -fPIC -O2 -pipe  -g $(WARN) $(INC)
```

并提交日志：

```sh
git commit -a -m "Fix relocation R_X86_64_32 against '.rodata' erro for shared object."
```

* 导出patch-queue——将其分支提交逐一转化为debian/patches目录下的补丁文件（为保证正确运行，清理掉中间文件）：

```sh
git clean -df
gbp-pq export
```

* 指定版本号5.1.3-2自动生成snapshot的debian/changelog：

```sh
git-dch -S -a -N 5.1.3-2
```

debian/changelog的新增内容如下：

```plain
tolua (5.1.3-2~1.gbp896bed) UNRELEASED; urgency=low

  ** SNAPSHOT build @896bede5a4eb6f3967cdfe94ea2ef419235e7183 **

  * UNRELEASED

 -- Like Ma   Sun, 19 Feb 2012 01:07:06 +0800
```

提交相关修改：

```sh
git add debian/changelog \
		debian/patches/series \
		debian/patches/0001-mkdir-for-tolua-lib-archive-and-remove-temp-files.patch \
		debian/patches/0002-Fix-relocation-R_X86_64_32-against-.rodata-can-not-b.patch
git commit -m "Fix relocation R_X86_64_32 against '.rodata' error for shared object"
```

测试构建新的deb包。为了避免污染当前环境，这里指定git首先导出源码至../tolua-build目录：

```sh
git-buildpackage --git-export-dir=../tolua-build --git-ignore-new
```

可以看出，上述debian/changelog新增信息，除版本号5.1.3-2外（新包的版本信息），并无实在意义，仅用于测试deb包的构建。

* 生成release的版本信息，并构建release的deb包：

```sh
git checkout src/bin/tolua_lua.o src/bin/toluabind.c
git-dch -R -a
git commit -a --amend
git-buildpackage --git-export-dir=../tolua-build --git-ignore-new
git tag debian/5.1.3-2
```

注意，前面仅仅在patch-queue分支上删除的两个中间文件，并未在master分支上删除它们，所以重新checkout它们以保证后续构建的正确运行。

这里的关键命令git-dch -R -a自动生成了release的版本信息，当然我们也可以根据需要再修改它们。最后一条命令，给master分支的HEAD加上标签debian/5.1.3-2，git-buildpackage将依赖于它才能继续正确工作。


到此为止，我已经演示了git-buildpackage的补丁制作过程，可以观察../tolua-build目录中生成的文件，它们就是我们可以用于发布的deb包源码文件，形式上与ubuntu/debian中apt-get source得到的一样。
