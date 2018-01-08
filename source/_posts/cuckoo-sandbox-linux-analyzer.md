---
title: Cuckoo沙箱Linux检测引擎
date: 2018-01-05 16:43:38
categories:
- Security
tags:
- Cuckoo
- Sandbox
- SystemTap
---

[Cuckoo](https://cuckoosandbox.org/)沙箱是一款开源自动化恶意软件分析系统，将可疑文件或URL在隔离环境中运行，生成详细的行为报告。

Cuckoo支持在Windows、macOS、Linux、Android四种操作系统平台下分析恶意文件（包括可执行文件、Office文档、PDF文档、邮件等多种类型）和恶意网站。相较于静态分析，沙箱动态分析可获得可疑文件的详细行为信息，如可疑文件创建的进程、调用的函数、创建删除或下载的文件、请求的网络流量等，进而对可疑文件进行深度判定。

<!-- more -->

Cuckoo的整体运行流程为：提交样本->启动虚拟机->往虚拟机投递样本和检测引擎并运行->回传检测结果->Processing模块处理结果->Signatures模块过规则->Reporting模块生成报告。

沙箱核心组件之一是动态检测引擎，Cuckoo沙箱Linux检测引擎基于SystemTap实现，关于SystemTap可以参见上一篇博客[SystemTap](https://consen.github.io/2018/01/04/systemtap/)，SystemTap在Linux Guest虚拟机中的安装参见[Installing the Linux guest](https://cuckoo.sh/docs/installation/guest/linux.html)。

首先打补丁[escape_delimiters](https://github.com/cuckoosandbox/cuckoo/blob/master/stuff/systemtap/escape_delimiters.patch)，将字符串中的特殊字符转换成十六进制表示，方便后续日志分析:

```
$ sudo patch /usr/share/systemtap/tapset/uconversions.stp < escape_delimiters.patch
```

然后将监控脚本[strace.stp](https://github.com/cuckoosandbox/cuckoo/blob/master/stuff/systemtap/strace.stp)转换成内核模块：

```
$ sudo stap -p4 -r $(uname -r) strace.stp -m stap_ -v
```

检测引擎运行时，[stap.py](https://github.com/cuckoosandbox/cuckoo/blob/master/cuckoo/data/analyzer/linux/modules/auxiliary/stap.py)直接运行生成的内核模块，监控样本进程：

```
    self.proc = subprocess.Popen([
        "staprun", "-vv",
        "-x", str(os.getpid()),
        "-o", "stap.log",
        path,
    ], stderr=subprocess.PIPE)
```

这里`path`即`stap_.ko`内核模块;`-x`指定target进程ID，可在SystemTap脚本通过target()获取。

生成的日志`stap.log`会回传到主机，由[processing](https://github.com/cuckoosandbox/cuckoo/blob/master/cuckoo/processing/platform/linux.py)模块分析，最终由reporting模块生成报告。

由监控脚本[strace.stp](https://github.com/cuckoosandbox/cuckoo/blob/master/stuff/systemtap/strace.stp)可见（引用的tapsets函数可参见[SystemTap Tapset Reference Manual](https://sourceware.org/systemtap/tapsets/)），引擎会监控样本进程及其子进程的所有系统调用并生成日志：

```
probe nd_syscall.* {
    if (pid() == target()) next         # skip our own helper process
    if (!target_set_pid(pid())) next    # skip unrelated processes
    ...
}

function report(syscall_name, syscall_argstr, syscall_retstr) {
    ...
    if (syscall_retstr == "")
        printf("%s%s(%s)\n", prefix, syscall_name, syscall_argstr)
    else
        printf("%s%s(%s) = %s\n", prefix, syscall_name, syscall_argstr, syscall_retstr)
    ...
}
```

生成的日志可参见测试样例[log.stap](https://github.com/cuckoosandbox/cuckoo/blob/master/tests/files/log.stap)：

```
Mon Jun 19 16:58:31 2017.445170 python@b774dcf9[680] execve("/usr/bin/sh", ["sh", "-c", "/tmp/helloworld.sh"], ["LANGUAGE=en_US:en", "HOME=/root", "LOGNAME=root", "PATH=/usr/bin:/bin", "LANG=en_US.UTF-8", "SHELL=/bin/sh", "PWD=/root"]) = -2 (ENOENT)
Mon Jun 19 16:58:31 2017.517266 sh@b77825f7[680] brk(0x0) = -2118402048
Mon Jun 19 16:58:31 2017.521264 sh@b77838c1[680] access("/etc/ld.so.nohwcap", F_OK) = -2 (ENOENT)
```

对日志的解析参见测试脚本[test_stap.py](https://github.com/cuckoosandbox/cuckoo/blob/master/tests/test_stap.py)。

投递样本helloworld.sh进行分析：

```
!/bin/bash               
echo "hello world!"       
cat /etc/passwd           
ping -c 4 www.360.com
```

生成的行为分析进程树：

![](http://7xtc3e.com1.z0.glb.clouddn.com/cuckoo-sandbox-linux-analyzer/linux-analyzer1.png)

部分系统调用信息：

![](http://7xtc3e.com1.z0.glb.clouddn.com/cuckoo-sandbox-linux-analyzer/linux-analyzer2.png)
