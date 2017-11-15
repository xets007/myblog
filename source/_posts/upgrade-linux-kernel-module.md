---
title: Linux内核模块升级
date: 2016-12-30 22:14:54
tags:
- Xen
- kernel
categories: Linux
---

目前沙箱虚拟化是基于Xen实现的，沙箱集群大概运行一两个月，会有一台服务器出现Xen卡死的情况，xl命令无法使用，虚拟机无法创建，不得不重启服务器。在项目开发初期，通过重启来解决这样的问题是可以接受的，毕竟出错的概率不是很高，但如果不从根本上解决此问题，心里就像压了一块石头。这样的产品也不能卖给客户，如果你对客户说：“服务器卡死了没关系，重启下就好”，后果会怎样？

现在就彻底解决下这个问题。

<!-- more -->

Xen卡死后，`/var/log/messages`记录的日志中有个kernel BUG:

```
Dec 23 00:51:38 w-xen09 kernel: ------------[ cut here ]------------
Dec 23 00:51:38 w-xen09 kernel: kernel BUG at drivers/xen/evtchn.c:264!
Dec 23 00:51:38 w-xen09 kernel: invalid opcode: 0000 [#3] SMP
Dec 23 00:51:38 w-xen09 kernel: Modules linked in: xt_physdev tun blktap xen_pciback xen_netback xen_blkback xen_gntalloc xen_gntdev xen_evtchn xenfs xen_privcmd bridge stp llc ipv6 ipt_MASQUERADE iptable_nat nf_conntrack_ipv4 nf_defrag_ipv4 nf_nat_ipv4 nf_nat nf_conntrack iptable_filter ip_tables sg serio_raw gpio_ich iTCO_wdt iTCO_vendor_support hpilo hpwdt coretemp freq_table mperf crc32_pclmul crc32c_intel ghash_clmulni_intel microcode pcspkr igb ptp pps_core lpc_ich ioatdma dca acpi_power_meter hwmon shpchp ext4 jbd2 mbcache sd_mod crc_t10dif aesni_intel ablk_helper cryptd lrw gf128mul glue_helper aes_x86_64 pata_acpi ata_generic ata_piix hpsa mgag200 ttm drm_kms_helper dm_mirror dm_region_hash dm_log dm_mod
Dec 23 00:51:38 w-xen09 kernel: CPU: 11 PID: 7959 Comm: xenstored Tainted: G      D      3.10.20-11.el6.centos.alt.x86_64 #1
Dec 23 00:51:38 w-xen09 kernel: Hardware name: HP ProLiant DL360p Gen8 HY, BIOS P71 08/02/2014
Dec 23 00:51:38 w-xen09 kernel: task: ffff881003548040 ti: ffff880ff03fa000 task.ti: ffff880ff03fa000
Dec 23 00:51:38 w-xen09 kernel: RIP: e030:[<ffffffffa03276f8>]  [<ffffffffa03276f8>] evtchn_bind_to_user+0xb8/0xc0 [xen_evtchn]
Dec 23 00:51:38 w-xen09 kernel: RSP: e02b:ffff880ff03fbe48  EFLAGS: 00010282
Dec 23 00:51:38 w-xen09 kernel: RAX: ffff880ff1490668 RBX: 00000000000000cd RCX: 0000000000000000
Dec 23 00:51:38 w-xen09 kernel: RDX: 0000000000000008 RSI: 00000000000000cd RDI: ffff881001655840
Dec 23 00:51:38 w-xen09 kernel: RBP: ffff880ff03fbe78 R08: 0000000000001782 R09: ffff880ff03fbe88
Dec 23 00:51:38 w-xen09 kernel: R10: 0000000000000000 R11: 0000000000000206 R12: 0000000000000000
Dec 23 00:51:38 w-xen09 kernel: R13: 00000000000000cd R14: 0000000000000000 R15: 00007fffc9028ec0
Dec 23 00:51:38 w-xen09 kernel: FS:  00007f8bf955f740(0000) GS:ffff8810a8d60000(0000) knlGS:ffff8810a8d60000
Dec 23 00:51:38 w-xen09 kernel: CS:  e033 DS: 0000 ES: 0000 CR0: 0000000080050033
Dec 23 00:51:38 w-xen09 kernel: CR2: 00007ff95089a260 CR3: 0000001002e02000 CR4: 0000000000042660
Dec 23 00:51:38 w-xen09 kernel: DR0: 0000000000000000 DR1: 0000000000000000 DR2: 0000000000000000
Dec 23 00:51:38 w-xen09 kernel: DR3: 0000000000000000 DR6: 00000000ffff0ff0 DR7: 0000000000000400
Dec 23 00:51:38 w-xen09 kernel: Stack:
Dec 23 00:51:38 w-xen09 kernel: 00000000000000fb 0000000000000001 ffff880ff03fbe78 ffff881001655840
Dec 23 00:51:38 w-xen09 kernel: 0000000000000000 0000000000084501 ffff880ff03fbec8 ffffffffa032797e
Dec 23 00:51:38 w-xen09 kernel: 00000001f03f1782 ffffffff000000cd 0000000100001782 ffff880ff03fbf40
Dec 23 00:51:38 w-xen09 kernel: Call Trace:
Dec 23 00:51:38 w-xen09 kernel: [<ffffffffa032797e>] evtchn_ioctl+0x13e/0x304 [xen_evtchn]
Dec 23 00:51:38 w-xen09 kernel: [<ffffffff811aa349>] do_vfs_ioctl+0x89/0x350
Dec 23 00:51:38 w-xen09 kernel: [<ffffffff811aa6b1>] SyS_ioctl+0xa1/0xb0
Dec 23 00:51:38 w-xen09 kernel: [<ffffffff810e00d6>] ? __audit_syscall_exit+0x246/0x2f0
Dec 23 00:51:38 w-xen09 kernel: [<ffffffff815fe719>] system_call_fastpath+0x16/0x1b
Dec 23 00:51:38 w-xen09 kernel: Code: 74 19 85 c0 74 04 0f 0b eb fe 48 8b 05 b2 1b 00 00 4a c7 04 e8 00 00 00 00 eb c3 48 8d 75 d0 bf 03 00 00 00 e8 da 6f 02 e1 eb d7 <0f> 0b eb fe 0f 1f 40 00 55 48 89 e5 48 83 ec 40 48 89 5d d8 4c
Dec 23 00:51:38 w-xen09 kernel: RIP  [<ffffffffa03276f8>] evtchn_bind_to_user+0xb8/0xc0 [xen_evtchn]
Dec 23 00:51:38 w-xen09 kernel: RSP <ffff880ff03fbe48>
Dec 23 00:51:38 w-xen09 kernel: ---[ end trace b4e323e25997f660 ]---
```

如果你了解Linux内核开发（不了解也没关系，可以参考下[The Linux Kernel Module Programming Guide](http://www.tldp.org/LDP/lkmpg/2.6/html/index.html)，这篇教程基于kernel-2.6编写的，如果你使用的内核版本比较新，如kernel-3.10，教程中一些代码可能会编译出错，因为新内核将某些模块重构了或者基于安全考虑将老版内核的一些功能屏蔽掉了），一定能马上看出这是Linux内核驱动模块xen_evtchn出了问题。

kernel BUG发生在`drivers/xen/evtchn.c:264`，目前使用的内核版本是kernel-3.10.20，所以得查看kernel-3.10.20中的[evtchn.c](https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/tree/drivers/xen/evtchn.c?h=v3.10.20)代码：

``` c
256     /*
257      * Ports are never reused, so every caller should pass in a
258      * unique port.
259      *
260      * (Locking not necessary because we haven't registered the
261      * interrupt handler yet, and our caller has already
262      * serialized bind operations.)
263      */
264     BUG_ON(get_port_user(port) != NULL);
```

当`get_port_user(port) != NULL`时，BUG_ON()就打印出了上面的出错信息，包括寄存器和堆栈等信息。再看下`get_port_user()`:

``` c
72 /*
73  * Who's bound to each port?  This is logically an array of struct
74  * per_user_data *, but we encode the current enabled-state in bit 0.
75  */
76 static unsigned long *port_user;
77 static DEFINE_SPINLOCK(port_user_lock); /* protects port_user[] and ring_prod */
78
79 static inline struct per_user_data *get_port_user(unsigned port)
80 {
81     return (struct per_user_data *)(port_user[port] & ~1);
82 }
```

代码比较简单，可以看出是port重用了。而最新的内核中，此问题已经被修复掉了，见提交记录[xen/evtchn: improve scalability by using per-user locks](https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/drivers/xen/evtchn.c?id=73cc4bb0c79eebe1f0e92b700d9fe8d1c9b061bb)。所以解决此问题可以升级内核，也可以只将内核模块xen_evtchn升级下，我们采用改动比较小的后者。

Linux内核模块是不能跨内核使用的，也就是说给A内核编译的内核模块无法在B内核上使用，除非采取一些tricks。比如我现在运行的内核版本是kernel-3.10.20，那么给kernel-2.6.32编译的内核模块就无法在kernel-3.10.20上使用，写个简单的demo说明下。

```
$ uname -r
3.10.20-11.el6.centos.alt.x86_64
```

最简单的内核模块，[hello world](https://github.com/consen/demo/blob/master/linux/kernel-module/hello-2.6/hello.c):

``` c
#include <linux/module.h>   /* Needed by all modules */
#include <linux/kernel.h>   /* Needed for KERN_INFO */
#include <linux/init.h>     /* Needed for the macros */

static int __init hello_init(void)
{
    printk(KERN_INFO "Hello, world\n");
    return 0;
}

static void __exit hello_exit(void)
{
    printk(KERN_INFO "Goodbye, world\n");
}

module_init(hello_init);
module_exit(hello_exit);
```

[Makefile](https://github.com/consen/demo/blob/master/linux/kernel-module/hello-2.6/Makefile)文件，注意指定的版本号是kernel-2.6.32:

```
obj-m += hello.o

all:
        make -C /lib/modules/2.6.32-220.el6.x86_64/build M=$(PWD) modules

clean:
        make -C /lib/modules/2.6.32-220.el6.x86_64/build M=$(PWD) clean
```

编译，生成hello.ko文件：

```
$ make
make -C /lib/modules/2.6.32-220.el6.x86_64/build M=/data1/home/xikangjie/github/demo/linux/kernel-module/hello-2.6 modules
make[1]: Entering directory `/usr/src/kernels/2.6.32-220.el6.x86_64'
  CC [M]  /data1/home/xikangjie/github/demo/linux/kernel-module/hello-2.6/hello.o
  Building modules, stage 2.
  MODPOST 1 modules
  CC      /data1/home/xikangjie/github/demo/linux/kernel-module/hello-2.6/hello.mod.o
  LD [M]  /data1/home/xikangjie/github/demo/linux/kernel-module/hello-2.6/hello.ko.unsigned
  NO SIGN [M] /data1/home/xikangjie/github/demo/linux/kernel-module/hello-2.6/hello.ko
make[1]: Leaving directory `/usr/src/kernels/2.6.32-220.el6.x86_64'
$ ls
hello.c  hello.ko  hello.ko.unsigned  hello.mod.c  hello.mod.o  hello.o  Makefile  modules.order  Module.symvers
```

插入内核模块，可以发现会出错，因为与当前内核版本不匹配，这个内核模块只能在kernel-2.6.32-220.el6.x86_64上正常运行:

```
$ sudo insmod ./hello.ko
insmod: error inserting './hello.ko': -1 Invalid module format
```

同时，`/var/log/messages`中对应的日志信息：

```
kernel: hello: disagrees about version of symbol module_layout
```

将Makefile中的内核版本改为当前内核版本，`/lib/modules/$(shell uname -r)/build`，重新编译下，hello.ko就可正常加载了，同时`/var/log/messages`也输出了`Hello, world`。

使用的是kernel-3.10.20，最好就在kernel-3.10.20的开发环境下将新版xen_evtchn重新编译下。需要注意的是，如果新版xen_evtchn使用了新内核提供的一些功能，是无法在kernel-3.10.20上编译的，只能使用新版内核开发环境。所以只需要拿到修复了Xen卡死问题的evtchn.c代码即可，不需要是最新，我们选用提交[xen: remove deprecated IRQF_DISABLED](https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/drivers/xen/evtchn.c?id=af09d1a73aed4e83ee095f2dabdc09386e31f2ea)中的[evtchn.c](https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/drivers/xen/evtchn.c?id=af09d1a73aed4e83ee095f2dabdc09386e31f2ea)代码，既将Xen卡死问题解决了，又没有使用新版内核功能，可以在kernel-3.10.20上编译。

然后安装当前内核kernel-3.10.20的开发环境，即一堆头文件，在CentOS下是kernel-devel-3.10.20-11.el6.centos.alt.x86_64.rpm：

```
$ yum info kernel-devel-3.10.20-11.el6.centos.alt.x86_64
Installed Packages
Name        : kernel-devel
Arch        : x86_64
Version     : 3.10.20
Release     : 11.el6.centos.alt
Size        : 32 M
Repo        : installed
Summary     : Development package for building kernel modules to match the kernel.
URL         : http://www.kernel.org/
License     : GPLv2
Description : This package provides the kernel header files and makefiles
            : sufficient to build modules against the kernel package.
```

安装好后会有目录`/usr/src/kernels/3.10.20-11.el6.centos.alt.x86_64/`，而且可以发现当前内核版本中/lib/modules/3.10.20-11.el6.centos.alt.x86_64/build链接也正常显示了：

```
build -> ../../../usr/src/kernels/3.10.20-11.el6.centos.alt.x86_64
```

接着编写Makefile：

```
obj-m += xen-evtchn.o
xen-evtchn-objs := evtchn.o

all:
        make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
        make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

运行`make`命令编译，生成内核模块`xen-evtchn.ko`，拷贝到/lib/modules/3.10.20-11.el6.centos.alt.x86_64/kernel/drivers/xen/目录下，将老版`xen-evtchn.ko`覆盖（以防万一，可以将老版`xen-evtchn.ko`先备份下），`reboot`重启系统。


---

参考：

- [The Linux Kernel Module Programming Guide](http://www.tldp.org/LDP/lkmpg/2.6/html/index.html)
- [Linux Kernel Makefiles](https://www.kernel.org/doc/Documentation/kbuild/makefiles.txt)
