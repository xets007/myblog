---
title: segfault段错误问题修复
date: 2017-11-28 21:36:22
categories: Linux
tags:
- segfault
- Xen
---

segfault段错误是软件开发中经常会遇到的错误，该错误是由非法内存访问造成的，如空指针引用；在只读内存区域进行写操作；访问受保护的内存区域等。在不同场景下，segfault的解决难度大不相同，比如有源代码并且segfault很容易复现，重新编译一个调试版可执行文件，用gdb调试马上就能定位问题。但是如果segfault很难复现，或者没有调式版可用呢？又如何去定位问题呢？

最近就在线上环境遇到一个qemu-ndb造成的segfault错误，影响很严重。qemu-nbd要用到nbd内核模块，segfault错误出现时，qemu-nbd相关进程卡主无响应，造成业务无法正常运行，甚至无法强制kill掉qemu-nbd进程，只能断电重启服务器使业务临时恢复正常运行。根据现象判断，qemu-nbd进程处于uninterruptible sleep状态，通过ps命令可以看出qemu-nbd进程的状态为D(uninterruptible sleep)。相较于interruptible sleep状态，uninterruptible sleep状态下的进程对信号没有响应，无法对它发送SIGKILL信号，也就无法kill掉，只能重启服务器。该状态出现一般是IO出了问题，必须从根本上解决该问题，才能避免一些不必要的麻烦。

<!-- more -->

```
$ ps aux | grep nbd
root     13647  0.0  0.0 295388  7344 ?        Dsl  Nov23   0:00 /usr/local/lib/xen/bin/qemu-nbd -c /dev/nbd10 /tmp/vm.img
```

在这种场景下，遇到了以下几个难点：

1. 难以复现，线上几十台服务器大约每隔1周有1~2台上qemu-nbd会出现segfault错误，几乎无法通过手工方式复现；
2. 有源代码，但是在本地测试调式版qemu-nbd，性能极差，无法投入线上使用，否则影响线上业务正常运行；
3. 线上运行的qemu-nbd在编译时加了`-O2 -fomit-frame-pointer`优化参数，增加了反汇编代码的分析难度；
4. 线上运行的qemu-nbd符号表信息被strip掉了，进一步增加了反汇编代码的分析难度；
5. segfault错误出现时在/var/log/message中只有一行日志信息，没有堆栈等更详细的信息。

```
kernel: qemu-nbd[13647]: segfault at 100 ip 0000561c8817d1c7 sp 00007fc8f1ce9c00 error 4 in qemu-nbd[561c88107000+155000]
```

**也就是说，只能通过这一行日志信息，加上qemu-nbd二进制文件和源代码，去定位具体哪一行代码造成了segfault。**

首先看下segfault日志的含义：

```
qemu-nbd[13647]                    可执行文件名[进程ID]
at 100                             出错时访问的内存地址
ip 0000561c8817d1c7                出错时的指令指针
sp 00007fc8f1ce9c00                出错时的栈指针
error 4                            错误码
qemu-nbd[561c88107000+155000]      进程虚拟内存区域(VMA)对应的文件名[VMA起始地址+VMA大小]   
```

错误码的定义:

``` c
/*
 * Page fault error code bits:
 *
 *   bit 0 ==    0: no page found   1: protection fault
 *   bit 1 ==    0: read access     1: write access
 *   bit 2 ==    0: kernel-mode access  1: user-mode access
 *   bit 3 ==               1: use of reserved bit detected
 *   bit 4 ==               1: fault was an instruction fetch
 *   bit 5 ==               1: protection keys block access
 */
enum x86_pf_error_code {

    PF_PROT     =       1 << 0,
    PF_WRITE    =       1 << 1,
    PF_USER     =       1 << 2,
    PF_RSVD     =       1 << 3,
    PF_INSTR    =       1 << 4,
    PF_PK       =       1 << 5,
};
```

由此可见，错误码4(100)表示从用户态读时出现no page found。

生成反汇编代码：

```
$ objdump -d -M intel qemu-nbd > qemu-nbd.asm
```

有了以上信息，即可通过ip指令指针在反汇编代码中定位出错指令。指令地址：

```
0x0000561c8817d1c7 - 0x561c88107000 = 0x761c7
```

由于线上运行的qemu-nbd的符号信息被strip掉了（为了使发布的可执行文件尽可能小，并增加逆向难度，一般会将符号信息剔除掉），所以从反汇编代码中很难确定0x761c7指令地址到底对应的是源代码的哪一行。

不过万幸的是，当初编译xen时的编译环境还在，没有strip掉符号信息的qemu-nbd版本还在，位于`tools/qemu-xen-dir/qemu-nbd`，strip版本位于`dist/install/usr/local/lib/xen/bin/qemu-nbd` (线上环境的qemu-nbd就是使用的该二进制文件)。现在对比下非strip和strip版本的反汇编代码：

```
$ objdump -d -M intel tools/qemu-xen-dir/qemu-nbd > qemu-nbd.asm
$ objdump -d -M intel dist/install/usr/local/lib/xen/bin/qemu-nbd > qemu-nbd-strip.asm
$ vim -d qemu-nbd.asm qemu-nbd-strip.asm
```

![qemu-nbd-asm-rbx](http://7xtc3e.com1.z0.glb.clouddn.com/qemu-nbd-segfault/qemu-nbd-asm-rbx.png)

有符号信息对于定位源代码帮助非常大，strip版本函数入口信息都没了，定位出错函数无异于大海捞针。

![qemu-nbd-asm](http://7xtc3e.com1.z0.glb.clouddn.com/qemu-nbd-segfault/qemu-nbd-asm.png)

由汇编代码可以看出问题出在函数`bdrv_co_flush`，执行指令`mov    rax,QWORD PTR [rdx+0x100]`时出错，此时rdx为0，所以segfault日志信息中才有`at 100`。

进一步分析汇编代码，由指令`mov    rbx,rdi`确定rbx为函数`bdrv_co_flush`的第一个参数`BlockDriverState *bs`，根据数据结构[struct BlockDriverState](http://xenbits.xen.org/gitweb/?p=qemu-xen.git;a=blob;f=include/block/block_int.h;h=1e939de4fe5a3b04b418edb4c108e82c3c93a0f8;hb=4220231eb22235e757d269722b9f6a594fbcb70f#l427)及内存对齐（此处按8字节对齐，其中CoQueue为包含两个指针的结构体，所以占16字节），确定`[rbx+0x38]`为`bs->drv`即rdx，同理根据结构体[struct BlockDriver](http://xenbits.xen.org/gitweb/?p=qemu-xen.git;a=blob;f=include/block/block_int.h;h=1e939de4fe5a3b04b418edb4c108e82c3c93a0f8;hb=4220231eb22235e757d269722b9f6a594fbcb70f#l87)可推算出`[rdx+0x100]`为`bs->drv->bdrv_co_flush`。（由于qemu-xen源码一直在更新，分析源码时只能使用当初编译xen时使用的qemu-xen版本）

![qemu-nbd-bs](http://7xtc3e.com1.z0.glb.clouddn.com/qemu-nbd-segfault/qemu-nbd-bs.png)

至此可以确定segfault是由于`bs-drv`为NULL造成的，即产生了空指针引用。函数代码位于[block/io.c](http://xenbits.xen.org/gitweb/?p=qemu-xen.git;a=blob;f=block/io.c;h=420944d80db104188445e198ce45eb890fc6edfb;hb=4220231eb22235e757d269722b9f6a594fbcb70f#l2293) 2293行。

``` c
2292     /* Write back all layers by calling one driver function */
2293     if (bs->drv->bdrv_co_flush) {
2294         ret = bs->drv->bdrv_co_flush(bs);
2295         goto out;
2296     }
```

确定了引起segfault的具体代码，问题就很好解决了。其时qemu主分支就在最近对此问题已经有一定修复，[block: Guard against NULL bs->drv](https://git.qemu.org/?p=qemu.git;a=commitdiff;h=d470ad42acfc73c45d3e8ed5311a491160b4c100;hp=93bbaf03ff7fd490e823814b8f5d6849a7b71a64)。

如果能进一步深入分析引起`bs->drv`为空指针的原因，并能手动复现，找出攻击路径，即可发起DOS(拒绝服务)攻击。
