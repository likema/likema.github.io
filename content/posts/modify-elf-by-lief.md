---
title: 通过 LIEF 修改 ELF 解决 glibc 兼容性问题
date: 2021-08-15 21:25:15
categories: [Linux, ELF, LIEF, Glibc, Python, C, C++]
---

数月前，我读了 [Linux 修改 ELF 解决 glibc 兼容性问题](https://mp.weixin.qq.com/s/2NNqYGYcCo2TEX4yfqG0qQ) ，颇受启发，但暂无用武之地。

最近，我遇到了几乎一样的问题，即 `clock_gettime` 兼容性。

## 最小化问题

为了将问题最小化，这里我以 `hello.c` 模拟该问题：


```c
#include <stdio.h>
#include <time.h>

int main() {
    struct timespec ts;
    clock_gettime(CLOCK_REALTIME, &ts);
    printf("%lu, %ld\n", ts.tv_sec, ts.tv_nsec);
    return 0;
}
```

在 Ubuntu 20.04 中，用系统内置 GCC 9.3.0 构建它：

```bash
gcc -fno-stack-protector -o hello hello.c -s
```

* `-fno-stack-protector` : 禁用 stack 保护特性。目的为最小化未定义符号，最小化问题场景。
* `-s` : 除去调试相关符号，目的同上。


通过 `readelf -sV hello` 展示其符号：

```
Symbol table '.dynsym' contains 8 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     .....
     2: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND clock_gettime@GLIBC_2.17 (2)
     3: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND printf@GLIBC_2.2.5 (3)
     ......

Version symbols section '.gnu.version' contains 8 entries:
 Addr: 0x0000000000000526  Offset: 0x000526  Link: 6 (.dynsym)
  000:   0 (*local*)       0 (*local*)       2 (GLIBC_2.17)    3 (GLIBC_2.2.5)
  ......

Version needs section '.gnu.version_r' contains 1 entry:
 Addr: 0x0000000000000538  Offset: 0x000538  Link: 7 (.dynstr)
  000000: Version: 1  File: libc.so.6  Cnt: 2
  0x0010:   Name: GLIBC_2.2.5  Flags: none  Version: 3
  0x0020:   Name: GLIBC_2.17  Flags: none  Version: 2
```

问题在于修改 `.gnu.version_r` 段偏移 `0x0020` 的相关值。回顾 `Elfxx_Vernaux` 结构：

```c
typedef struct {
    Elfxx_Word    vna_hash;  // 库名称 (如 GLIBC_2.17) 的 hash 值
    Elfxx_Half    vna_flags;
    Elfxx_Half    vna_other; // .gnu.version 段中符号的版本值
    Elfxx_Word    vna_name;  // 库名称 (如 GLIBC_2.17) 为相对 .dynstr 段偏移
    Elfxx_Word    vna_next;  // 下一条记录相对本记录首地址的偏移
} Elfxx_Vernaux;
```

实际仅须修改 `vna_hash` 和 `vna_name` ，前者实为后者的校验和。

以下提供 2 种方法修改 `hello` 二进制文件，使其无须重编译就能运行于 RHEL/CentOS 6 中。

## 通过 dd 修改

```bash
# 复制 hello 偏移 0x538 + 0x10 = 0x548 的 4 byte 至文件 glibc225_vna_hash ；
dd if=hello of=glibc225_vna_hash skip=$((16#548)) bs=1 count=4

# 复制 hello 偏移 0x538 + 0x10 + 0x08 = 0x550 的 4 byte 至文件 glibc225_vna_hash
dd if=hello of=glibc225_vna_name skip=$((16#550)) bs=1 count=4
```

* `0x538` 为 `.gnu.version_r` 段首地址。
* `0x10` 为 `GLIBC_2.2.5` 记录的段偏移，即 `Elfxx_Vernaux::vna_hash` 偏移。
* `0x08` 为 `GLIBC_2.2.5` 记录内 `Elfxx_Vernaux::vna_name` 偏移。

为保持 `hello` 不改变， 修改 `hello` 至 `hello-dd`

```bash
cp hello hello-dd

# 复制 glibc225_vna_hash 至 hello-dd 偏移 0x530 + 0x20 = 0x558
dd if=glibc225_vna_hash of=hello-dd seek=$((16#558)) bs=1 conv=notrunc

# 复制 glibc225_vna_name 至 hello-dd 偏移 0x530 + 0x20 + 0x08 = 0x560
dd if=glibc225_vna_name of=hello-dd seek=$((16#560)) bs=1 conv=notrunc
```

* `0x20` 为 `GLIBC_2.1.7` 记录的段偏移，即 `Elfxx_Vernaux::vna_hash` 偏移。
* `0x08` 为 `GLIBC_2.1.7` 记录内 `Elfxx_Vernaux::vna_name` 偏移。

通过 `readelf -sV hello` 观察修改是否生效：

```
Symbol table '.dynsym' contains 8 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     ......
     2: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND clock_gettime@GLIBC_2.2.5 (2)
     3: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND printf@GLIBC_2.2.5 (3)
     ......

Version symbols section '.gnu.version' contains 8 entries:
 Addr: 0x0000000000000526  Offset: 0x000526  Link: 4 (.dynsym)
  000:   0 (*local*)       0 (*local*)       2 (GLIBC_2.2.5)   3 (GLIBC_2.2.5)
  ......

Version needs section '.gnu.version_r' contains 1 entry:
 Addr: 0x0000000000000538  Offset: 0x000538  Link: 26 (.dynstr)
  000000: Version: 1  File: libc.so.6  Cnt: 2
  0x0010:   Name: GLIBC_2.2.5  Flags: none  Version: 3
  0x0020:   Name: GLIBC_2.2.5  Flags: none  Version: 2
```

最后，将 `librt.so.1` 添加至 `hello-dd` 的库依赖列表中。

```bash
patchelf --add-needed librt.so.1 hello-dd
```

## 通过 LIEF

[LIEF](https://lief.quarkslab.com/doc/latest/intro.html) 是一个能分析和修改 ELF , PE , MachO 和 Android 格式的库，它提供了 C/C++ 和 Python 的 API.

以下通过 LIEF 的 Python API 达到上述同样的效果。

### 安装

```bash
pip3 install lief
```

### 写脚本

创建 `lief-hello.py` :

```python
import lief


binary = lief.parse('hello')
glibc225_aux = clock_gettime_aux = None

for i in binary.imported_symbols:
    # 查找 GLIBC_2.2.5 的 symbol_version_auxiliary 记录，
    if i.symbol_version.symbol_version_auxiliary.name == 'GLIBC_2.2.5':
        glibc225_aux = i.symbol_version.symbol_version_auxiliary
    # 查找 clock_gettime 的 symbol_version_auxiliary 记录，
    elif i.name == 'clock_gettime':
        clock_gettime_aux = i.symbol_version.symbol_version_auxiliary

# 复制 GLIBC_2.2.5 记录 name 和 hash 至 clock_gettime 记录对应字段中。
if glibc225_aux and clock_gettime_aux:
    clock_gettime_aux.name = glibc225_aux.name
    clock_gettime_aux.hash = glibc225_aux.hash
    binary.add_library('librt.so.1')  # 相当于 patchelf --add-needed librt.so.1
    binary.write('hello-lief')
```

以上 `symbol_version_auxiliary` 相当于前述 `Elfxx_Vernaux`

### 执行脚本

执行 `python3 lief-hello.py` 将生成 `hello-lief` 。

通过 `readelf -sV hello-lief` 观察修改是否生效：

```
Symbol table '.dynsym' contains 8 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     ......
     2: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND clock_gettime@GLIBC_2.2.5 (2)
     3: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND printf@GLIBC_2.2.5 (3)
     ......

Version symbols section '.gnu.version' contains 8 entries:
 Addr: 0x0000000000002526  Offset: 0x002526  Link: 6 (.dynsym)
  000:   0 (*local*)       0 (*local*)       2 (GLIBC_2.2.5)   3 (GLIBC_2.2.5)
  ......

Version needs section '.gnu.version_r' contains 1 entry:
 Addr: 0x0000000000002538  Offset: 0x002538  Link: 7 (.dynstr)
  000000: Version: 1  File: libc.so.6  Cnt: 2
  0x0010:   Name: GLIBC_2.2.5  Flags: none  Version: 3
  0x0020:   Name: GLIBC_2.2.5  Flags: none  Version: 2
```
