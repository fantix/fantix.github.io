---
layout: post
title:  "从无到有，在 ArchLinux 上编译 x32 ABI 的 GCC, Glibc 和 Python"
date:   2014-02-17 16:47:06
categories: linux
---

首先翻译[维基百科对 x32 ABI 的定义](http://en.wikipedia.org/wiki/X32_ABI)：

> x32 ABI (应用程序二进制接口) 是一个开发中的 Linux 项目，为 x32 ABI 编译的程序将能运行在 x86-64 指令集环境的 64 位模式下，但仅仅使用 32 位大小的指针和数据类型。这尽管限制了这个程序最多只能使用 4GB 的虚拟地址空间，但却减少了这个程序占用的内存，并且在某些情况下能使程序运行的更快。最好的测评结果来自 SPEC CPU2000 中的 181.mcf，结果显示 x32 ABI 的版本比 x86-64 的要快 32%。CPU2000 测评的整数计算部分，x32 比 x86-64 平均要快 5-8%，但有时也会慢很多；在浮点运算的速度上，x32 较 x86-64 并没有优势。

简单来说呢，就是为 x32 编译的程序，既能获得 64 位系统跑的快的优势，又只使用跟 32 位程序差不多少的内存。如果你的计算机只有 4GB 或更少的内存，但却拥有一颗强劲的 64 位 CPU 的话，那 x32 将能更大程度地榨取你硬件的潜力。（注，单进程不使用超过 4GB 内存的，理论上也可以受益于 x32，只是说如果你的笔记本已经在跑 32GB 内存了的话，那估计也不用省这一小半的内存了——再买条 32GB 的内存去吧。）

警告，x32 ABI 是一个开发中的 Linux 项目，所以碰到一些 bug 是很正常的。笔者假设你已经有一台运行着 x86_64 ArchLinux 的机器。

1. 内核
=====

**更新：自从 ArchLinux kernel 3.5.4-1 之后，x32 已经是默认开启的了，内核版本号高于 3.5.4-1 的 ArchLinux 用户请跳过本步骤。**

OK，可以着手开始编译了。是的，我们要从源码开始编译，白手起家。到底多白呢，从 Linux Kernel 开始。Linux Kernel 自 [3.4 开始支持 x32 ABI](http://kernelnewbies.org/Linux_3.4#head-039c9d273884c9639937c10d68b4a3214869eb4b)，只需要打开一个内核选项就可以了：

```
CONFIG_X86_X32=y
```

如果你不想按照 [ArchLinux wiki 上的每一步](https://wiki.archlinux.org/index.php/Kernels/Compilation/Arch_Build_System)来自己配置编译你的内核的话，最简单的办法就是使用[笔者提供的 AUR 包](https://aur.archlinux.org/packages.php?ID=61029)(**已删除**)——基于 ArchLinux 官方内核的 x32 版本。所以呢，如果你已经安装了 [yaourt](https://aur.archlinux.org/packages.php?ID=5863)，那么执行下面的命令就够了：

```bash
yaourt -S linux-x32
```

编译内核花 1 个小时以上很正常。安装好之后，你可以修改你的系统引导文件（`/boot/grub/menu.lst` 或者其他引导配置文件），重启进入新装的内核。

2. 种子
=======

**更新：编译好的二进制包已经上传至 [ArchLinux 中文社区软件仓库](https://github.com/archlinuxcn/repo#arch-linux-chinese-community-repository)，如果你想马上拥有支持 x32 的编译器的话，可以直接从那里安装，跳过步骤 2、3、4。**

因为 x32 的 GCC 和 Glibc 有鸡生蛋蛋生鸡的问题，所以我们在编译正式的 x32 GCC 和 Glibc 之前呢，需要先下两个种子。

第一个其实是一个缺失的头文件，最终的 x32 Glibc 将会包含这个头文件，但是在目前需要先用到它的情况下，我们可以[暂时做一个软链接](http://sourceware.org/glibc/wiki/x32)：

```bash
sudo ln -s /usr/include/gnu/stubs-64.h /usr/include/gnu/stubs-x32.h
```

同样地，笔者已经为这个软链接做好了一个 [AUR 包](https://aur.archlinux.org/packages/glibc-x32-seed/)，如果你是用的是 ArchLinux，那么你没有必要手动创建这个软链接（还得记着装完两个种子之后删除它）。这个 AUR 包会在安装正式的 x32 Glibc 的时候自动卸载，删除无用的软链接。安装命令如下：

```bash
yaourt -S glibc-x32-seed
```

第二个种子是一个临时的 GCC 编译器，其中一半是我们即将自己编译的，另一半 x32 的库我们无法自己编译（因为缺少 x32 的 Glibc），所以要先用[别人编译好的](https://sites.google.com/site/x32abi/documents)。x32 ABI 需要 GCC 的版本至少为 4.7。在配置 GCC 的时候，尤其要注意添加 x32 的支持：

```bash
cd gcc-4.7.1 ./configure --with-multilib-list=m32,m64,mx32
```

然后 `make`，数分钟之后 `make` 会失败，因为我们还没有 x32 的 Glibc。这时候，把下好的 `gcc-x32-seed.tar.bz2` 解压到 `gcc` 文件夹下面就好了。具体的步骤可以参考[这一篇文章](http://sourceware.org/glibc/wiki/x32)，比较省事儿的办法仍然是继续用笔者提供的 [AUR 包](https://aur.archlinux.org/packages/gcc-x32-seed/)，自动完成下载、编译和安装：

```bash
yaourt -S gcc-x32-seed
```

3. Glibc
========

终于要开始编译正式的 x32 Glibc 了。Glibc [自 2.16 开始支持 x32 ABI](http://sourceware.org/ml/libc-alpha/2012-06/msg00807.html)，所以如果你无法使用笔者的 AUR 包的话，你需要先去下载一份 [Glibc 的源码](http://www.gnu.org/software/libc/libc.html)，解压之后，配置需要一下几点注意：

1. 指定使用你的 GCC 种子编译器，且使用 x32 来编译，比如 `CC="/path/to/cc -mx32 -B/path/to/gcc/libs" CXX="/path/to/g++ -mx32 -B/path/to/gcc/libs"`

2. 指定正确的环境参数：`--target=x86_64-x32-linux --build=x86_64-linux --host=x86_64-x32-linux`

3. 启用多架构支持：`--enable-multi-arch`

4. 指定与 ArchLinux 习惯一直的 `lib` 文件夹位置，在 `configparms` 文件中添加：`slibdir=/usr/libx32`

然后开始正常的编译。编译好了之后，前面做的两个种子就算圆满完成任务了，删掉他们（种子 GCC 并不影响），然后安装刚编译好的 Glibc。最后为了在 ArchLinux 中正常工作，还要做额外的几步：

1. `sudo ln -s /usr/libx32/ld-linux-x32.so.2 /usr/lib/ld-linux-x32.so.2`

2. 创建 `/etc/ld.so.conf.d/libx32-glibc.conf`，添加一行 x32 库的路径信息：`/usr/libx32`

3. 软链接区域信息：`sudo ln -s /usr/lib/locale /usr/libx32/locale`

4. 最后因为 ArchLinux 官方库的一个 bug，需要额外一个软链接：`sudo ln -s /usr/lib /libx32`

所有这些 x32 Glibc 的工作都包含在了笔者制作的 [AUR 包](https://aur.archlinux.org/packages/libx32-glibc/)中：

```bash
yaourt -S libx32-glibc
```

注意，安装这个包会提示与 glibc-x32-seed 冲突，只需要同意删除 seed 包就可以了。

4. GCC
======

胜利在望，下一步编译好正式的 x32 GCC 就算有了一个最基本的 x32 编译工具集。编译 x32 的 GCC 跟在 ArchLinux 上编译普通的 gcc-multilib 没有多大区别，只要把路径 `/usr/libx32` 设置正确就可以了——对了，还有 `configure` 的时候记着加 x32 支持：

```bash
cd gcc-4.7.1 ./configure --with-multilib-list=m32,m64,mx32 ...
```

照例，笔者提供的 [AUR 包](https://aur.archlinux.org/packages/gcc-multilib-x32/)如下：

```bash
yaourt -S gcc-multilib-x32
```

装这个 AUR 包的时候，系统会提示与一大堆包冲突——没关系，同意删掉冲突的 gcc-libs gcc-libs-multilib gcc gcc-x32-seed 是安全的，因为 gcc-multilib-x32 可以支持编译 3 种架构的程序：x86、x86_64 和 x86_x32。

装好之后，是不是打算试一下 x32 的编译效果？没问题，你可以试试下面这段简单的 C 程序：

```c
#include <stdio.h>

int main() {
    int i = 0;
    printf("Size of pointer: %d\n", (int)sizeof(&i));
    return 0;
}
```

针对三种不同的架构进行编译：

```bash
gcc -m32 test.c -o test_32
gcc -m64 test.c -o test_64
gcc -mx32 test.c -o test_x32
```

然后执行试一下：

```bash
$ ./test_32
Size of pointer: 4
$ ./test_64
Size of pointer: 8
$ ./test_x32
Size of pointer: 4
```

5. Python
=========

由于 Python 依赖的库还比较多，其中一些如 OpenSSL 还需要额外的补丁才能在 x32 上编译，笔者在这里就不一一详解了，直接用 AUR 包搞定吧：

```bash
yaourt -S libx32-readline libx32-libffi libx32-ncurses libx32-expat libx32-bzip2 libx32-gdbm libx32-openssl libx32-sqlite3 libx32-zlib python2-x32
```

写个简单的 Python 程序试一下吧：

```python
import time
placeholder = [None] * 1024 * 1024 * 8
time.sleep(600)
```

然后执行试一下：

```bash
python2 test.py &
/opt/python2-x32/bin/python2.7 test.py &
```

打开系统监视器，看看这两个进程的内存是不是几乎差了一半？

版本历史：

* 2014-2-17：Markdown 版本，增加了更新注释
* 2012-7-29：发布于 <http://blog.chinaunix.net/uid/350516.html>

