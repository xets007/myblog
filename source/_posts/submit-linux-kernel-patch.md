---
title: 提交Linux内核Patch
date: 2018-01-19 21:35:41
categories:
- Linux
tags:
- kernel
- patch 
---

上一篇博客[使用QEMU和GDB调试Linux内核](https://consen.github.io/2018/01/17/debug-linux-kernel-with-qemu-and-gdb/)，最后使用内核提供的GDB扩展功能获取当前运行进程，发现内核已经不再使用`thread_info`获取当前进程，而是使用Per-CPU变量。而且从内核4.9版本开始，`thread_info`也不再位于内核栈底部，然而内核提供的辅助调试函数`lx_thread_info()`仍然通过内核栈底地址获取`thread_info`，很明显这是个Bug，于是决定将其修复并提交一个内核Patch，提交后很快就得到内核维护人员的回应，将Patch提交到了内核主分支。

Linux内核Patch提交还是采用邮件列表方式，不过提供了自动化工具。

<!-- more -->

## Bug修复

Bug的原因已经很明确了，先看下问题代码`scripts/gdb/linux/tasks.py`：

``` python
def get_thread_info(task):
    thread_info_ptr_type = thread_info_type.get_type().pointer()
    if utils.is_target_arch("ia64"):
        ...
    else:
        thread_info = task['stack'].cast(thread_info_ptr_type)
    return thread_info.dereference()
```

还是使用的老的流程，通过栈底地址获取`thread_info`。

从内核4.9版本开始，已将`thread_info`移到了`task_struct`(`include\linux\sched.h`)，而且一定是第一个字段：

``` c
struct task_struct {
#ifdef CONFIG_THREAD_INFO_IN_TASK
    /*
     * For reasons of header soup (see current_thread_info()), this
     * must be the first element of task_struct.
     */
    struct thread_info      thread_info;
#endif
    ...
}
```

所以修复很简单，只需要判断task的第一个字段是否为`thread_info`，如果是，则直接将其返回；如果不是，还是走原先的流程：

``` c
$ git diff ./
diff --git a/scripts/gdb/linux/tasks.py b/scripts/gdb/linux/tasks.py
index 1bf949c43b76..f6ab3ccf698f 100644
--- a/scripts/gdb/linux/tasks.py
+++ b/scripts/gdb/linux/tasks.py
@@ -96,6 +96,8 @@ def get_thread_info(task):
         thread_info_addr = task.address + ia64_task_size
         thread_info = thread_info_addr.cast(thread_info_ptr_type)
     else:
+        if task.type.fields()[0].type == thread_info_type.get_type():
+            return task['thread_info']
         thread_info = task['stack'].cast(thread_info_ptr_type)
     return thread_info.dereference()

```

## Git配置

添加用户和Email配置，用于`git send-email`发送Patch。

这里使用Gmail邮箱服务，在Linux项目`.git/config`配置中添加如下内容：

```
[user]
    name = Xi Kangjie
    email = imxikangjie@gmail.com
[sendemail]
    smtpEncryption = tls
    smtpServer = smtp.gmail.com
    smtpUser = imxikangjie@gmail.com
    smtpServerPort = 587
```

注意在Google账户配置中[允许不够安全的应用登陆](https://myaccount.google.com/lesssecureapps)，否则后面发送Patch会收到如下警告：

![](http://7xtc3e.com1.z0.glb.clouddn.com/submit-linux-kernel-patch/gmail-deny.png)

## Patch生成

Bug修复后，先检查下代码是否符合规范：

```
$ ./scripts/checkpatch.pl --file scripts/gdb/linux/tasks.py 
total: 0 errors, 0 warnings, 137 lines checked

scripts/gdb/linux/tasks.py has no obvious style problems and is ready for submission.
```

没问题就可以写提交日志了：

```
$ git add scripts/gdb/linux/tasks.py
$ git commit -s
```

![](http://7xtc3e.com1.z0.glb.clouddn.com/submit-linux-kernel-patch/commit.png)

`-s`自动添加签发人`Signed-off-by: Xi Kangjie <imxikangjie@gmail.com>`，表示该Patch是你创建的，你会对该Patch负责。日志的第一行为简短描述，会成为邮件标题（Subject），之后空一行，添加详细描述，会成为邮件内容，再空一行，添加签发人。

将最近一次提交生成Patch：

```
$ git format-patch HEAD~                           
0001-scripts-gdb-fix-get_thread_info.patch
```

再次检查Patch是否符合规范：

```
$ ./scripts/checkpatch.pl 0001-scripts-gdb-fix-get_thread_info.patch
ERROR: Please use git commit description style 'commit <12+ chars of sha1> ("<title line>")' - ie: 'commit c65eacbe290b ("sched/core: Allow putting thread_info into task_struct")'
#10:
- c65eacbe290b (sched/core: Allow putting thread_info into task_struct)

ERROR: Please use git commit description style 'commit <12+ chars of sha1> ("<title line>")' - ie: 'commit 15f4eae70d36 ("x86: Move thread_info into task_struct")'
#11:
- 15f4eae70d36 (x86: Move thread_info into task_struct)

total: 2 errors, 0 warnings, 8 lines checked

NOTE: For some of the reported defects, checkpatch may be able to
      mechanically convert to the typical style using --fix or --fix-inplace.

0001-scripts-gdb-fix-get_thread_info.patch has style problems, please review.

NOTE: If any of the errors are false positives, please report
      them to the maintainer, see CHECKPATCH in MAINTAINERS.
```

看来格式有错误，引用的提交描述不符合规范，直接修改Patch文件，再次检查：

```
$ ./scripts/checkpatch.pl 0001-scripts-gdb-fix-get_thread_info.patch
total: 0 errors, 0 warnings, 8 lines checked

0001-scripts-gdb-fix-get_thread_info.patch has no obvious style problems and is ready for submission.
```

## Patch提交

获取Patch相关维护人员：

```
$ ./scripts/get_maintainer.pl 0001-scripts-gdb-fix-get_thread_info.patch 
Jan Kiszka <jan.kiszka@siemens.com> (supporter:GDB KERNEL DEBUGGING HELPER SCRIPTS)
Kieran Bingham <kieran@bingham.xyz> (supporter:GDB KERNEL DEBUGGING HELPER SCRIPTS)
Xi Kangjie <imxikangjie@gmail.com> (commit_signer:1/1=100%,authored:1/1=100%,added_lines:2/2=100%)
linux-kernel@vger.kernel.org (open list)
```

发送Patch:

```
$ git send-email --to jan.kiszka@siemens.com --to kieran@bingham.xyz --cc linux-kernel@vger.kernel.org 0001-scripts-gdb-fix-get_thread_info.patch
0001-scripts-gdb-fix-get_thread_info.patch
(mbox) Adding cc: Xi Kangjie <imxikangjie@gmail.com> from line 'From: Xi Kangjie <imxikangjie@gmail.com>'
(body) Adding cc: Xi Kangjie <imxikangjie@gmail.com> from line 'Signed-off-by: Xi Kangjie <imxikangjie@gmail.com>'

From: Xi Kangjie <imxikangjie@gmail.com>
To: jan.kiszka@siemens.com,
        kieran@bingham.xyz
Cc: linux-kernel@vger.kernel.org,
        Xi Kangjie <imxikangjie@gmail.com>
Subject: [PATCH] scripts/gdb: fix get_thread_info
Date: Thu, 18 Jan 2018 21:01:59 +0000
Message-Id: <20180118210159.17223-1-imxikangjie@gmail.com>
X-Mailer: git-send-email 2.13.2

    The Cc list above has been expanded by additional
    addresses found in the patch commit message. By default
    send-email prompts before sending whenever this occurs.
    This behavior is controlled by the sendemail.confirm
    configuration setting.

    For additional information, run 'git send-email --help'.
    To retain the current behavior, but squelch this message,
    run 'git config --global sendemail.confirm auto'.

Send this email? ([y]es|[n]o|[q]uit|[a]ll): y
Password for 'smtp://imxikangjie@gmail.com@smtp.gmail.com:587':
OK. Log says:
Server: smtp.gmail.com
MAIL FROM:<imxikangjie@gmail.com>
RCPT TO:<jan.kiszka@siemens.com>
RCPT TO:<kieran@bingham.xyz>
RCPT TO:<linux-kernel@vger.kernel.org>
RCPT TO:<imxikangjie@gmail.com>
From: Xi Kangjie <imxikangjie@gmail.com>
To: jan.kiszka@siemens.com,
        kieran@bingham.xyz
Cc: linux-kernel@vger.kernel.org,
        Xi Kangjie <imxikangjie@gmail.com>
Subject: [PATCH] scripts/gdb: fix get_thread_info
Date: Thu, 18 Jan 2018 21:01:59 +0000
Message-Id: <20180118210159.17223-1-imxikangjie@gmail.com>
X-Mailer: git-send-email 2.13.2

Result: 250 2.0.0 OK 1516281059 v9sm14814354pfj.88 - gsmtp
```

提交成功后，就能在内核邮件列表中看到自己的邮件[[PATCH] scripts/gdb: fix get_thread_info](https://lkml.org/lkml/2018/1/18/291)，以及维护人员的回复[Re: [PATCH] scripts/gdb: fix get_thread_info](https://lkml.org/lkml/2018/1/18/516)。

维护人员确认无误后，会将Patch提交到mm-tree，[scripts-gdb-fix-get_thread_info.patch added to -mm tree](https://www.spinics.net/lists/stable/msg210845.html)，再提交到stable分支，[[patch 4/6] scripts/gdb/linux/tasks.py: fix get_thread_info](https://www.spinics.net/lists/stable/msg210851.html)。

**最终会被Linus Torvalds合并到主分支，[scripts/gdb/linux/tasks.py: fix get_thread_info](https://github.com/torvalds/linux/commit/883d50f56d263f70fd73c0d96b09eb36c34e9305)。**

看到自己的代码在世界的某个角落运转，推动世界向前发展，才是真正的享受。

参考：

- [Submitting patches: the essential guide to getting your code into the kernel](https://www.kernel.org/doc/html/latest/process/submitting-patches.html)
- [The perfect patch](https://www.ozlabs.org/~akpm/stuff/tpp.txt)
- [Linux kernel patch format](http://linux.yyz.us/patch-format.html)
- [git config](https://git-scm.com/docs/git-config), [git format-patch](https://git-scm.com/docs/git-format-patch) and [git send-email](https://git-scm.com/docs/git-send-email)
- GDB Python API [Values From Inferior](https://sourceware.org/gdb/onlinedocs/gdb/Values-From-Inferior.html) and [Types In Python](https://sourceware.org/gdb/onlinedocs/gdb/Types-In-Python.html)
- [mm tree](https://en.wikipedia.org/wiki/Mm_tree)
