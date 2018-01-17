---
title: 使用QEMU和GDB调试Linux内核
date: 2018-01-17 20:02:00
categories:
- Linux
tags:
- kernel
- GDB
- Debugging
---

排查Linux内核Bug，研究内核机制，除了查看资料阅读源码，还可通过调试器，动态分析内核执行流程。

QEMU模拟器原生支持GDB调试器，这样可以很方便地使用GDB的强大功能对操作系统进行调试，如设置断点；单步执行；查看调用栈、查看寄存器、查看内存、查看变量；修改变量改变执行流程等。

<!-- more -->

## 编译调试版内核

对内核进行调试需要解析符号信息，所以得编译一个调试版内核。

```
$ cd linux-4.14
$ make menuconfig
$ make -j 20
```

这里需要开启内核参数`CONFIG_DEBUG_INFO`和`CONFIG_GDB_SCRIPTS`。GDB提供了Python接口来扩展功能，内核基于Python接口实现了一系列辅助脚本，简化内核调试，开启`CONFIG_GDB_SCRIPTS`参数就可以使用了。

```
Kernel hacking  ---> 
    [*] Kernel debugging
    Compile-time checks and compiler options  --->
        [*] Compile the kernel with debug info
        [*]   Provide GDB scripts for kernel debugging
```

## 构建initramfs根文件系统

Linux系统启动阶段，boot loader加载完内核文件vmlinuz后，内核紧接着需要挂载磁盘根文件系统，但如果此时内核没有相应驱动，无法识别磁盘，就需要先加载驱动，而驱动又位于`/lib/modules`，得挂载根文件系统才能读取，这就陷入了一个两难境地，系统无法顺利启动。于是有了initramfs根文件系统，其中包含必要的设备驱动和工具，boot loader加载initramfs到内存中，内核会将其挂载到根目录`/`,然后运行`/init`脚本，挂载真正的磁盘根文件系统。

这里借助[BusyBox](https://www.busybox.net/)构建极简initramfs，提供基本的用户态可执行程序。

编译BusyBox，配置`CONFIG_STATIC`参数，编译静态版BusyBox，编译好的可执行文件`busybox`不依赖动态链接库，可以独立运行，方便构建initramfs。

```
$ cd busybox-1.28.0
$ make menuconfig
```

```
Settings  --->
    [*] Build static binary (no shared libs)
```

```
$ make -j 20
$ make install
```

会安装在`_install`目录:

```
$ ls _install 
bin  linuxrc  sbin  usr
```

创建initramfs，其中包含BusyBox可执行程序、必要的设备文件、启动脚本`init`。这里没有内核模块，如果需要调试内核模块，可将需要的内核模块包含进来。`init`脚本只挂载了虚拟文件系统`procfs`和`sysfs`，没有挂载磁盘根文件系统，所有调试操作都在内存中进行，不会落磁盘。

```
$ mkdir initramfs
$ cd initramfs
$ cp ../_install/* -rf ./
$ mkdir dev proc sys
$ sudo cp -a /dev/{null, console, tty, tty1, tty2, tty3, tty4} dev/
$ rm linuxrc
$ vim init
$ chmod a+x init
$ ls
$ bin   dev  init  proc  sbin  sys   usr
```

init文件内容：

```
#!/bin/busybox sh         
mount -t proc none /proc  
mount -t sysfs none /sys  

exec /sbin/init
```

打包initramfs:

```
$ find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../initramfs.cpio.gz
```

## 调试

启动内核：

```
$ cd busybox-1.28.0
$ qemu-system-x86_64 -s -kernel /path/to/vmlinux -initrd initramfs.cpio.gz -nographic -append "console=ttyS0"
```

- `-s`是`-gdb tcp::1234`缩写，监听1234端口，在GDB中可以通过`target remote localhost:1234`连接；
- `-kernel`指定编译好的调试版内核；
- `-initrd`指定制作的initramfs;
- `-nographic`取消图形输出窗口，使QEMU成简单的命令行程序；
- `-append "console=ttyS0"`将输出重定向到console，将会显示在标准输出stdio。

启动后的根目录：

```
/ # ls                    
bin   dev  init  proc  root  sbin  sys   usr 
```

由于系统自带的GDB版本为7.2，内核辅助脚本无法使用，重新编译了一个新版GDB：

```
$ cd gdb-7.9.1
$ ./configure --with-python=$(which python2.7)
$ make -j 20
$ sudo make install
```

启动GDB:

```
$ cd linux-4.14
$ /usr/local/bin/gdb vmlinux
(gdb) target remote localhost:1234
```

使用内核提供的GDB辅助调试功能：

```
(gdb) apropos lx                                    
function lx_current -- Return current task          
function lx_module -- Find module by name and return the module variable                                 
function lx_per_cpu -- Return per-cpu variable      
function lx_task_by_pid -- Find Linux task by PID and return the task_struct variable                    
function lx_thread_info -- Calculate Linux thread_info from task variable                                
function lx_thread_info_by_pid -- Calculate Linux thread_info from task variable found by pid            
lx-cmdline --  Report the Linux Commandline used in the current kernel                                   
lx-cpus -- List CPU status arrays                   
lx-dmesg -- Print Linux kernel log buffer           
lx-fdtdump -- Output Flattened Device Tree header and dump FDT blob to the filename                      
lx-iomem -- Identify the IO memory resource locations defined by the kernel                              
lx-ioports -- Identify the IO port resource locations defined by the kernel                              
lx-list-check -- Verify a list consistency          
lx-lsmod -- List currently loaded modules           
lx-mounts -- Report the VFS mounts of the current process namespace                                      
lx-ps -- Dump Linux tasks                           
lx-symbols -- (Re-)load symbols of Linux kernel and currently loaded modules                             
lx-version --  Report the Linux Version of the current kernel
(gdb) lx-cmdline 
console=ttyS0
```

在函数`cmdline_proc_show`设置断点，虚拟机中运行`cat /proc/cmdline`命令即会触发。

```
(gdb) b cmdline_proc_show                           
Breakpoint 1 at 0xffffffff81298d99: file fs/proc/cmdline.c, line 9.                                      
(gdb) c                                             
Continuing.                                         

Breakpoint 1, cmdline_proc_show (m=0xffff880006695000, v=0x1 <irq_stack_union+1>) at fs/proc/cmdline.c:9
9               seq_printf(m, "%s\n", saved_command_line);                                              
(gdb) bt
#0  cmdline_proc_show (m=0xffff880006695000, v=0x1 <irq_stack_union+1>) at fs/proc/cmdline.c:9
#1  0xffffffff81247439 in seq_read (file=0xffff880006058b00, buf=<optimized out>, size=<optimized out>, ppos=<optimized out>) at fs/seq_file.c:234
#2  0xffffffff812908b3 in proc_reg_read (file=<optimized out>, buf=<optimized out>, count=<optimized out>, ppos=<optimized out>) at fs/proc/inode.c:217
#3  0xffffffff8121f174 in do_loop_readv_writev (filp=0xffff880006058b00, iter=0xffffc900001bbb38, ppos=<optimized out>, type=0, flags=<optimized out>) at fs/read_write.c:694
#4  0xffffffff8121ffed in do_iter_read (file=0xffff880006058b00, iter=0xffffc900001bbb38, pos=0xffffc900001bbd00, flags=0) at fs/read_write.c:918
#5  0xffffffff81220138 in vfs_readv (file=0xffff880006058b00, vec=<optimized out>, vlen=<optimized out>, pos=0xffffc900001bbd00, flags=0) at fs/read_write.c:980
#6  0xffffffff812547ed in kernel_readv (offset=<optimized out>, vlen=<optimized out>, vec=<optimized out>, file=<optimized out>) at fs/splice.c:361
#7  default_file_splice_read (in=0xffff880006058b00, ppos=0xffffc900001bbdd0, pipe=<optimized out>, len=<optimized out>, flags=<optimized out>) at fs/splice.c:416
#8  0xffffffff81253c7c in do_splice_to (in=0xffff880006058b00, ppos=0xffffc900001bbdd0, pipe=0xffff8800071a1f00, len=16777216, flags=<optimized out>) at fs/splice.c:880
#9  0xffffffff81253f77 in splice_direct_to_actor (in=<optimized out>, sd=0xffffc900001bbe18, actor=<optimized out>) at fs/splice.c:952
#10 0xffffffff812540e3 in do_splice_direct (in=0xffff880006058b00, ppos=0xffffc900001bbec0, out=<optimized out>, opos=<optimized out>, len=<optimized out>, flags=<optimized out>) at fs/splice.c:1061
#11 0xffffffff8122147f in do_sendfile (out_fd=<optimized out>, in_fd=<optimized out>, ppos=0x0 <irq_stack_union>, count=<optimized out>, max=<optimized out>) at fs/read_write.c:1434
#12 0xffffffff812216f5 in SYSC_sendfile64 (count=<optimized out>, offset=<optimized out>, in_fd=<optimized out>, out_fd=<optimized out>) at fs/read_write.c:1495
#13 SyS_sendfile64 (out_fd=1, in_fd=3, offset=0, count=<optimized out>) at fs/read_write.c:1481
#14 0xffffffff8175edb7 in entry_SYSCALL_64_fastpath () at arch/x86/entry/entry_64.S:203
#15 0x0000000000000000 in ?? ()
(gdb) p saved_command_line
$2 = 0xffff880007e68980 "console=ttyS0"
```

## 获取当前进程

《深入理解Linux内核》第三版第三章--进程，讲到内核采用了一种精妙的设计来获取当前进程。

![](http://7xtc3e.com1.z0.glb.clouddn.com/debug-linux-kernel-with-qemu-and-gdb/process.jpg)

Linux把跟一个进程相关的`thread_info`和内核栈stack放在了同一内存区域，内核通过esp寄存器获得当前CPU上运行的进程的`thread_info`地址，由于进程描述符task字段在`thread_info`结构体中偏移量为0，进而获得task地址。相关汇编指令如下：

```
movl $0xffffe000, %ecx      /* 内核栈大小为8K，屏蔽低13位有效位。
andl $esp, %ecx
movl (%ecx), p
```

指令运行后，p就获得当前CPU上运行进程的描述符指针。

然而在调试器中调了下，发现这种机制早已经被废弃掉了。`thread_info`结构体中只剩下一个字段flags，进程描述符字段task已经删除，无法通过`thread_info`获取进程描述符了。

而且进程的`thread_info`也不再位于进程内核栈底了，而是放在了进程描述符`task_struct`结构体中，见提交[sched/core: Allow putting thread_info into task_struct](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=15f4eae70d365bba26854c90b6002aaabb18c8aa)和[x86: Move thread_info into task_struct](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=15f4eae70d365bba26854c90b6002aaabb18c8aa)，这样也无法通过esp寄存器获取`thread_info`地址了。

```
(gdb) p $lx_current().thread_info
$5 = {flags = 2147483648}
```

这样做是从安全角度考虑的，一方面可以防止esp寄存器泄露后进而泄露进程描述符指针，二是防止内核栈溢出覆盖`thread_info`。

Linux内核从2.6引入了Per-CPU变量，获取当前指针也是通过Per-CPU变量实现的。

```
(gdb) p $lx_current().pid
$50 = 77
(gdb) p $lx_per_cpu("current_task").pid
$52 = 77
```


参考：

- [Tips for Linux Kernel Development](http://eisen.io/slides/jeyu_tips_for_kernel_dev_cmps107_2017.pdf)
- [How to Build A Custom Linux Kernel For Qemu ](http://mgalgs.github.io/2015/05/16/how-to-build-a-custom-linux-kernel-for-qemu-2015-edition.html)
- [Linux Kernel System Debugging](https://blog.0x972.info/?d=2014/11/27/18/45/48-linux-kernel-system-debugging-part-1-system-setup)
- [Debugging kernel and modules via gdb](https://www.kernel.org/doc/html/latest/dev-tools/gdb-kernel-debugging.html)
- [BusyBox simplifies embedded Linux systems](https://www.ibm.com/developerworks/library/l-busybox/index.html)
- [Custom Initramfs](https://wiki.gentoo.org/wiki/Custom_Initramfs)
- [Per-CPU variables](https://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html)
- [Linux kernel debugging with GDB: getting a task running on a CPU](http://slavaim.blogspot.com/2017/09/linux-kernel-debugging-with-gdb-getting.html)
