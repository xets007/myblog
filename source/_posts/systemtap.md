---
title: SystemTap
date: 2018-01-04 21:15:49
categories:
- Linux
tags:
- SystemTap
---


[SystemTap](https://sourceware.org/systemtap/wiki)是一款Linux跟踪探测（trace/probe）工具，可收集Linux运行信息，用于性能诊断和问题排查等。SystemTap不需要每次都对内核修改、编译、安装、重启等繁杂冗余的操作，通过其自定义的脚本语言，即可编写功能丰富强大的探测模块，动态载入内核，对系统进行监控，获取指定信息。

SystemTap可以监控系统调用、内核函数及其他发生在内核的事件（event），当事件触发时，运行指定处理例程（handler）。

<!-- more -->

## 原理

SystemTap基于[Kprobes](https://www.kernel.org/doc/Documentation/kprobes.txt)实现，Kprobes允许在内核代码任意地址设置断点，当断点触发时，运行指定的处理例程。Kprobes有两种探测类型：kprobes和kretprobes，kprobes可以探测内核任意指令，kretprobes探测函数返回。当注册一个kprobe时，会拷贝一份探测指令，然后将指令的第一个字节替换成断点指令（如int3），断点触发时运行指定处理函数，然后运行原始指令。当注册一个kretprobe，会在探测函数入口点注册一个常规的kprobe，当函数调用时，会获取返回地址，然后将返回地址替换成一个跳转地址（trampoline address），当函数返回时，运行跳转地址，此时处理函数执行，然后再运行原始返回地址。

SystemTap的运行分以下5个步骤：

1. parse，解析脚本，转换成解析树（parse tree），这个阶段还会进行预处理，语法语义检查。
2. elaboration，解析脚本中符号和引用，依赖tapsets（预编写的脚本库）和调试信息。
3. translate, 将第2步输出转换成C代码。
4. build，将C代码编译成内核模块。
5. run，加载内核模块并运行。

![systemtap-arch](http://7xtc3e.com1.z0.glb.clouddn.com/systemtap/systemtap-arch.png)

stap命令完成前4步，第5步由staprun和stapio完成。

![systemtap-flow](http://7xtc3e.com1.z0.glb.clouddn.com/systemtap/systemtap-flow.png)

## 安装

各Linux发行版如Ubuntu、CentOS等，都提供了SystemTap安装包，直接通过包管理命令安装即可，这里采用[源码编译](https://sourceware.org/git/?p=systemtap.git;a=blob_plain;f=README;hb=HEAD)方式安装。

从[SystemTap Releases](https://sourceware.org/systemtap/wiki/SystemTapReleases)下载最新版源码，编译安装，依赖的第三方工具及开发包直接通过包管理命令安装即可。

```
$ cd systemtap-3.2
$ ./configure
$ make -j 20
$ sudo make install
```

SystemTap借助调试信息来确定内核探测点地址，需要调试版内核，这里重新编译Linux-4.14内核。

```
$ cd linux-4.14-debuginfo
$ vim Makefile
```

可以修改`Makefile`中的`EXTRAVERSION`为`EXTRAVERSION = -debuginfo`，这样安装后内核版本为`4.14.0-debuginfo`，以便和非调试版内核做区分。

```
$ cp ../linux-4.14/.config ./
$ make menuconfig
```

配置内核参数：

```
General setup  --->
    -*- Kernel->user space relay support (formerly relayfs)
    [*] Kprobes
Kernel hacking  ---> 
    [*] Kernel debugging
    Compile-time checks and compiler options  --->
        [*] Compile the kernel with debug info
        -*- Debug Filesystem
```

配置完后，检查`.config`配置文件中以下参数都已配置：

```
CONFIG_DEBUG_INFO
CONFIG_KPROBES
CONFIG_RELAY
CONFIG_DEBUG_FS
```

编译安装内核：

```
$ make -j 20
$ sudo make modules_install
$ sudo make install
```

重启系统，检查SystemTap是否安装成功：

```
$ uname -r
4.14.0-debuginfo

$ sudo stap -ve 'probe begin { log("hello world") exit() }'
Pass 1: parsed user script and 469 library scripts using 182836virt/49676res/5624shr/44624data kb, in 330usr/30sys/350real ms.
Pass 2: analyzed script: 1 probe, 2 functions, 0 embeds, 0 globals using 183628virt/50828res/6016shr/45416data kb, in 10usr/0sys/8real ms.
Pass 3: translated to C into "/tmp/stapplkAR0/stap_b9273d87672a231f62a6c5ceae813d9a_1099_src.c" using 183760virt/50828res/6016shr/45548data kb, in 0usr/0sys/0real ms.
Pass 4: compiled C into "stap_b9273d87672a231f62a6c5ceae813d9a_1099.ko" in 12300usr/2260sys/14357real ms.
Pass 5: starting run.
hello world
Pass 5: run completed in 0usr/20sys/356real ms.
```

## 使用

SystemTap安装好后，自带的示例脚本位于`/usr/local/share/systemtap/examples/`，编写脚本可以作为参考。tapsets位于`/usr/local/share/systemtap/tapset/`。

SystemTap脚本语言借鉴了dtrace和awk，SystemTap脚本以`.stp`作为文件扩展名，包含的探测点采用如下格式编写：

```
probe event {statement}
```

这里以探测sshd进程的系统调用统计为例，了解SystemTap脚本的使用。

```
global syscalllist

probe begin {
  printf("SSHD Monitoring Started (10 seconds)...\n")
}

probe syscall.*
{
  if (execname() == "sshd") {
    syscalllist[name]++
  }
}

probe timer.ms(10000) {
  foreach ( name in syscalllist ) {
    printf("%s = %d\n", name, syscalllist[name] )
  }
  exit()
}

probe end {
  printf("SSHD Monitoring Ended\n")
}
```

这里脚本运行10s，输出结果：

```
$ sudo stap sshd_profile.stp
Syslog Monitoring Started (10 seconds)...           
rt_sigprocmask = 144                                
read = 36                                           
select = 72                                         
write = 36                                          
ioctl = 12
SSHD Monitoring Ended
```

调用exit()退出，也可通过`Ctrl-c`退出。

SystemTap会缓存脚本转换后的C语言和内核模块，如果脚本没有更改，再次运行时不必重新构建，直接使用之前的。

也可显示保存构建的内核模块，供后续直接使用，`-m`指定内核模块名，`-p4`表示处理到第4阶段停止：

```
$ sudo stap -v -m sshd_profile.ko -p4 sshd_profile.stp
```

`staprun`运行编译好的模块：

```
$ sudo staprun sshd_profile.ko
```

Cuckoo沙箱Linux检测引擎就使用了SystemTap，可以参见下一篇博客[Cuckoo沙箱Linux检测引擎](https://consen.github.io/2018/01/05/cuckoo-sandbox-linux-analyzer/)。

参考：

* [Systemtap tutorial](https://sourceware.org/systemtap/tutorial.pdf)
* [SystemTap Beginners Guide](https://sourceware.org/systemtap/SystemTap_Beginners_Guide.pdf)
* [Architecture of systemtap: a Linux trace/probe tool](https://sourceware.org/systemtap/archpaper.pdf)
* [SystemTap Language Reference](https://sourceware.org/systemtap/langref.pdf)
* [SystemTap: Instrumenting the Linux Kernel for Analyzing Performance and Functional Problems](http://www.redbooks.ibm.com/redpapers/pdfs/redp4469.pdf)
* [Linux introspection and SystemTap](https://www.ibm.com/developerworks/library/l-systemtap/index.html)
* [Kernel debugging with Kprobes](https://www.ibm.com/developerworks/library/l-kprobes/index.html)
