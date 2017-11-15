---
title: SSH远程端口转发
date: 2016-11-22 19:04:45
tags: SSH
categories: Toolbox
---

自从开发完Xenforce+XenforceHub，虚拟机镜像部署变得异常简单，直接在要部署镜像的服务器上运行`xenforce pull`命令即可将需要的镜像从XenforceHub下载到本地，镜像可直接使用，无需更改配置，而且`xenforce pull`命令还会处理好镜像依赖，将依赖的镜像也下载下来。镜像部署变得完美，一切变得自动化，一切按照理想的方式工作着。

<!-- more -->

![](http://7xtc3e.com1.z0.glb.clouddn.com/ssh-remote-port-forwarding/pull.png)

但是，因业务原因，一部分服务器部署在隔离网，导致服务器无法访问XenforceHub，无法使用`xenforce pull`命令部署镜像，一夜回到解放前，即使有`scp`、`rsync`、`nc`等一些优秀工具，可以借助用来手动同步镜像，还是太麻烦，程序猿懒起来，自己都害怕。最理想的方式还是用`xenforce pull`部署镜像，即使服务器在隔离网内。程序猿的首要任务是把不可能变得可能，把复杂变得简单，把低效变得高效。

隐隐约约想起当初学SSH的时候，SSH有个很精妙的用法，比如A、B两台服务器之间要实现互通，但是现在A可以访问B，B却不能访问A，那么B怎样访问A呢？可以借助SSH的远程端口转发功能。今天仔细研究了下，果然派上了用场。

以服务器server1为例，XenforceHub服务器可以访问server1，但是server1无法访问XenforceHub，在XenforceHub服务器上运行以下命令：

```
ssh -R 9000:localhost:9000 xikangjie@server1
```

这样，在服务器server1上访问localhost:9000，就相当于访问XenforceHub:9000，又可以愉快地用命令`xenforce pull`部署镜像了。

![](http://7xtc3e.com1.z0.glb.clouddn.com/ssh-remote-port-forwarding/pull2.gif)

---

参考：

- [SSH原理与运用（二）：远程操作与端口转发](http://www.ruanyifeng.com/blog/2011/12/ssh_port_forwarding.html)
- [实战 SSH 端口转发](http://www.ibm.com/developerworks/cn/linux/l-cn-sshforward/)
- [SSH Tunnel - Local and Remote Port Forwarding Explained With Examples](http://blog.trackets.com/2014/05/17/ssh-tunnel-local-and-remote-port-forwarding-explained-with-examples.html)
