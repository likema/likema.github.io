---
layout: post
title: git-buildpackage示例（二）
comments: true
categories:
---

在[《git-buildpackage示例（一）》](/blog/2012/02/19/git-buildpackage-1/)中，我介绍了如何利用git-buildpackage为Ubuntu已有包做一个补丁包的办法。

当时，我的补丁是基于tolua 5.1.3版本。一段时间后，tolua的作者释放了5.1.4版本。问题出现了，如何将我的补丁合并到5.1.4版本中呢？

下面我将继续使用git-buildpackage来解决合并上游新版本的问题：

1. 下载tolua 5.1.4源码包（假设放在git工作目录上层）：

```
wget http://www.tecgraf.puc-rio.br/~celes/tolua/tolua-5.1.4.tar.gz
cd tolua
git-import-orig -u 5.1.4 ../tolua-5.1.4.tar.gz
```

2. 手动解决遇到的冲突（如src/bin/Makefile）并提交更新：

```
git commit
```

3. 这时，运行

```
git log --format=%d:%s
```

输出：

```
 (HEAD, master):Merge commit 'upstream/5.1.4'
 (upstream/5.1.4, upstream):Imported Upstream version 5.1.4
 (debian/5.1.3-2):Fix relocation R_X86_64_32 against '.rodata' can not be used when making a shared object
 (debian/5.1.3-1):Imported Debian patch 5.1.3-1
 (upstream/5.1.3):Imported Upstream version 5.1.3
```

upstream/5.1.4分支被创建，且将其合并到master分支中。如此，master分支合并完毕，接下来将合并debian的patches。

4. 重整patch-queue：

```
gbp-pq rebase
```

手动解决遇到的冲突：

```
git rm -f src/bin/toluabind.c
git rebase --continue
```

5. 导出patch-queue（至master分支）

```
git clean -df
gbp-pq export
```

6. 指定版本号5.1.4-1自动生成snapshot的debian/changelog：

```
git-dch -a -S -N 5.1.4-1
git add debian/changelog
git add debian/patches/0001-mkdir-for-tolua-lib-archive-and-remove-temp-files.patch
git commit -m "Update patches from debian/5.1.3-1"
```

测试构建新的deb包。为了避免污染当前环境，这里指定git首先导出源码至../tolua-build目录：

```
git-buildpackage --git-export-dir=../tolua-build --git-ignore-new
```

7. 生成release的版本信息，并构建release的deb包：

```
git-dch -a -R
git ci --amend
git tag debian/5.1.4-1
git-buildpackage --git-export-dir=../tolua-build --git-ignore-new
```

到此为止，我已经演示了git-buildpackage合并上游版本的过程。不难发现，git-buildpackage充分利用了git的特点，在很大程度上简化了补丁开发和维护的过程。
