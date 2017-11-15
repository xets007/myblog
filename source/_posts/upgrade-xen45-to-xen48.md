---
title: 升级Xen 4.5到Xen 4.8
date: 2016-12-16 21:13:09
tags: Xen
categories: Virtualization
---

Xen官方上周发布了Xen 4.8, [What’s New with Xen Project Hypervisor 4.8?](https://blog.xenproject.org/2016/12/07/whats-new-with-xen-project-hypervisor-4-8/)，添加了很多新功能，如[Credit2](https://wiki.xenproject.org/wiki/Credit2_Scheduler_Development)调度器正式发布。Xenforce目前使用的是Xen 4.5，从性能、稳定性、功能等几方面考虑，决定升级到Xen 4.8。

<!-- more -->

虽然Xenforce在设计实现之初就考虑到向前兼容Xen，以方便升级Xen，但实际升级到Xen 4.8时还是遇到了一些问题，不过都很好地解决了。升级完后，简单测试了下，设备CPU24核、内存64G、机械硬盘、快照1G，通过快照恢复一台虚拟机只需要3秒左右，而在Xen 4.5上需要6秒左右，Xenforce性能大幅提升，沙箱整体性能也得到提升。如果继续使用Xen 4.5，要提升性能就不得不升级硬件，如CPU 32核或快照存放在固态硬盘。升级到Xen 4.8，解决了快照加载的性能瓶颈，省去了硬件升级开销。

升级流程：

- 下载源码包 [Xen Project 4.8.0](https://xenproject.org/downloads/xen-archives/xen-project-48-series/xen-project-480.html)。
- 按照README中的说明，安装依赖包，编译出错基本都是因为依赖包安装有问题。
- 配置，编译。

```
./configure
make
```

编译成功会生成dist文件夹，此文件夹可以直接拷贝到其他设备上，进行安装。

```
$ ls dist 
COPYING  install  install.sh  README
```

- 安装，运行dist/install.sh脚本，如果安装在`/usr/local`目录下，运行`ldconfig /usr/local/lib`命令加载动态链接库。
- 配置开机启动项，因为是从Xen 4.5升级过来的，直接沿用之前的`grub`配置即可，当然，如果想使用`Credit2`调度器，得在grub配置中添加`sched=credit2`，详见[Xen Hypervisor Command Line Options](http://xenbits.xen.org/docs/unstable/misc/xen-command-line.html)
- 重启设备。重启完成后，运行`xl info`命令，查看Xen版本。

![xen48](http://7xtc3e.com1.z0.glb.clouddn.com/upgrade-xen45-to-xen48/xen4.8.png)

因为之前Xenforce是在Xen 4.5上制作的虚拟机快照，升级到Xen 4.8后，快照恢复出错。Xen从4.6版本开始，快照采用了新的格式[Migration V2](https://xenbits.xen.org/docs/unstable/features/migration.html)，当然，官方也提供了工具`convert-legacy-stream`将老版快照升级到新版。

先安装依赖，进入Xen 4.8源码的`tools/python`目录，运行命令`python setup.py install`。可运行`convert-legacy-stream -h`查看使用说明。

```
convert-legacy-stream -i vm.checkpoint.45 -o vm.checkpoint -w 64 -g hvm -f libxl -x -v
```

遗憾的是转换后快照依然恢复出错，问题出在显卡上。不得不仔细研究了下Migration V2格式，参见[LibXenCtrl Domain Image Format](https://xenbits.xen.org/docs/unstable/specs/libxc-migration-stream.html)和[LibXenLight Domain Image Format](https://xenbits.xen.org/docs/unstable/specs/libxl-migration-stream.html)，最终用了一个小trick将问题解决，避免了对Xenforce做大幅改动。想起了传说中当年TK教主将一个损坏的压缩包，一个字节一个字节修复。当然，要从根本上解决此问题，还需把Xen对qemu的使用，以及快照中qemu状态保存的内容及格式，做更深入探究。

自此，Xen从4.5升级到4.8，Xenforce也无缝升级，平滑过渡。

---

参考：

- [What’s New with Xen Project Hypervisor 4.8?](https://blog.xenproject.org/2016/12/07/whats-new-with-xen-project-hypervisor-4-8/)
- [LibXenCtrl Domain Image Format](https://xenbits.xen.org/docs/unstable/specs/libxc-migration-stream.html)
- [LibXenLight Domain Image Format](https://xenbits.xen.org/docs/unstable/specs/libxl-migration-stream.html)
