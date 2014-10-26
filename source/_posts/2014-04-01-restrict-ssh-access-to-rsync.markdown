---
layout: post
title: 利用SSH限制rsync的访问目录
comments: true
categories: [Linux, SSH]
---

我经常使用VPS来共享数据给朋友同事。由于VPS通常内存较小，不太适合安装ftp服务器；而http服务器常常只有下载的功能；更重要是某些数据较大，每天可能都在改变，需要增量同步上传或下载。

rsync是这类场景的良好选择。一方面，通过SSH服务器来工作，它不常驻内存，节约了VPS内存使用；另一方面，它优秀的二进制增量同步功能，不仅减少了同步时间，也节约了VPS有限的带宽。

在实际使用过程中，我遇到了下述问题：

* 针对某些目录，某些用户仅能读（下载）;
* 针对某些目录，某些用户仅能写（上传）;
* 具有读或写权限的用户不能登录SSH，也不能执行任何程序;
* 具有读或写权限的用户不能访问其他目录。

诚然，这些问题可以通过操作系统用户相对复杂的权限控制（包括目录基本权限和ACL以及SELinux等）。然而，通过搜索和学习，我发现强大的SSH早已具备这样的接口：

* 通过密钥对，简化用户使用，让用户彻底摆脱密码的记忆（当然用户加密私钥，还是要自己记密码的）；
* 通过~/.ssh/authorized\_keys的command选项（通过[man authorized_keys](http://man.he.net/man5/authorized_keys)阅读详细内容），设置脚本过滤掉业务上无效的指令；
* 还是通过command选项，限制不同密钥有不同权限，如密钥A只能读，密钥B只能写。

下面，我将通过具体示例来解释command选项:

## 限制用户仅能读 ##

```sh
#!/bin/sh

DATA_DIR=/home/share/data

case "$SSH_ORIGINAL_COMMAND" in
rsync\ --server*)
        TARGET=`echo "$SSH_ORIGINAL_COMMAND" | awk '{ print $NF }'`
        if echo "$SSH_ORIGINAL_COMMAND" | grep -q " --sender " && [ "$TARGET" = $DATA_DIR ]; then
                $SSH_ORIGINAL_COMMAND
        else
                echo "Rejected"
        fi
        ;;
*)
        echo "Rejected"
        ;;
esac
```

该脚本（存储在文件/usr/local/bin/validate_pull_share）限制用户share仅能下载$DATA_DIR的文件, 它需要跟公钥配置在服务器的/home/share/.ssh/authorized_keys，如：

```
command="/usr/local/bin/validate_pull_share" ssh-rsa <用户公钥A>
```

* command选项用于指定脚本，该脚本可以过滤一些远程命令（禁止执行），它相当于远程命令的前置脚本。
* $SSH_ORIGINAL_COMMAND是sshd传递给command脚本环境变量，表示ssh客户端需要远程执行的命令。
* 第6行限制仅能执行rsync服务端指令。
* 第7行获取$SSH_ORIGINAL_COMMAND的最后一个参数（这里没有考虑有空格的目录），这个参数在这里是rsync需要访问的目录$TARGET.
* 第8行为命令合法性判断, 由两个条件组成：
	* 判断--sender参数是否存在于$SSH_ORIGINAL_COMMAND中，--sender表示该命令为下载命令；
	* 判断下载目录$TARGET是否与规定的目录一致。


## 限制用户仅能写 ##

```sh
#!/bin/sh

DATA_DIR=/home/share/data

case "$SSH_ORIGINAL_COMMAND" in
rsync\ --server*)
        TARGET=`echo "$SSH_ORIGINAL_COMMAND" | awk '{ print $NF }'`
        if ! echo "$SSH_ORIGINAL_COMMAND" | grep -q " --sender " && [ "$TARGET" = $DATA_DIR ]; then
                $SSH_ORIGINAL_COMMAND
        else
                echo "Rejected"
        fi
        ;;
*)
        echo "Rejected"
        ;;
esac
```

实际仅与读限制脚本相差一行（第8行），仅允许没有--sender参数的命令（即上传命令）。


该脚本（存储在文件/usr/local/bin/validate_push_share）同样需要跟另一个公钥配置在服务器的/home/share/.ssh/authorized_keys，如：

```
command="/usr/local/bin/validate_pull_share" ssh-rsa <用户公钥B>
```

## 总结 ##

通过上述脚本和配置实现了不同密钥访问同一目录的不同权限控制。还可以在这些脚本基础上扩展到多个目录或文件的访问控制。以及访问时间的控制，如某时段只能读，另外时段只能写；避免正在写数据的时候，有用户读数据造成数据不一致的问题。
