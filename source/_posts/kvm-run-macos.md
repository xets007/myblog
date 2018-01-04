---
title: KVM运行macOS
date: 2017-12-23 20:08:22
categories:
- Virtualization
tags:
- KVM
- macOS
- TAP
- bridge
---

最近尝试基于Xen运行macOS操作系统，之前已经有人做过类似的研究：

- [Snow Leopard Server on Xen](http://wiki.osx86project.org/wiki/index.php/Snow_Leopard_Server_on_Xen)
- [MACOS XEN: SNOW LEOPARD AS GUEST ON A XEN DOMU](https://www.bisente.com/2011/03/15/macos-xen-snow-leopard-as-guest-on-a-xen-domu/)
- [Qubes 3 MacOSX](https://groups.google.com/forum/#!msg/qubes-users/RiVntUzgJmY/rXMtXD3WKQAJ)

虽然可正常运行，但缺少后续研究，也没有官方支持，所以目前为止Xen只能勉强运行老版本OSX操作系统。运行OSX系统，需要对Xen hypervisor进行修改，如支持MSR 0x35，OSX系统启动时需要据此确定CPU核心数和线程数，但是这个特性连Intel官方开发文档都没有明确说明，所以Xen官方拒绝将此修改合并到Xen主分支代码。

- [Xen-devel PATCH x86/hvm: add support for MSR 0x35](https://lists.xen.org/archives/html/xen-devel/2014-09/msg02524.html)
- [Xen-devel PATCH x86/traps: hypervisor leaf 0x40000010 timing info](https://lists.xen.org/archives/html/xen-devel/2014-09/msg02525.html)

自己在本地修改Xen源码，虽然可安装、启动OSX-10.10系统，但还是遇到了其他问题，如外设模拟、系统驱动等，所以系统还是没法正常使用，只好作罢，转而投向更加活跃、有社区支持的[OSX-KVM](https://github.com/kholia/OSX-KVM)项目，基于KVM运行macOS系统。

<!-- more -->

## 准备工作

运行KVM需要CPU支持虚拟化扩展，即Intel VT-x或AMD SVM，如果功能没开启，需要在BISO开启。运行OSX需要KVM和QEMU支持，对KVM和QEMU的修改都已合并到主分支代码，如果系统自带的KVM或QEMU版本太老，OSX是无法正常运行的，至少需要Linux kernel >= 4.7，QEMU >= 2.6。

在[The Linux Kernel Archives](https://www.kernel.org/)下载最新稳定版Linux内核编译安装，注意KVM相关虚拟化参数配置。

```
$ uname -r
4.14.0
```

加载KVM内核模块：

```
$ sudo modprobe kvm_intel
$ lsmod | grep kvm
kvm_intel             192512  0
kvm                   512000  1 kvm_intel
irqbypass              16384  1 kvm
```

在[Download QEMU](https://www.qemu.org/download/#source)下载最新稳定版QEMU编译安装。

```
$ /usr/local/bin/qemu-system-x86_64 --version                                          
QEMU emulator version 2.11.0                        
Copyright (c) 2003-2017 Fabrice Bellard and the QEMU Project developers
```

## 安装macOS

接着按照[OSX-KVM](https://github.com/kholia/OSX-KVM)系统安装文档，安装macOS系统。我直接使用的`qemu-system-x86_64`，没有借助`libvirt`。

```
$ cat boot-macOS.sh
qemu-system-x86_64 -enable-kvm -m 2048 -cpu Penryn,kvm=off,vendor=GenuineIntel \
      -machine pc-q35-2.4 \
      -smp 2,cores=1 \
      -usb -device usb-kbd -device usb-tablet \
      -device isa-applesmc,osk="ourhardworkbythesewordsguardedpleasedontsteal(c)AppleComputerInc" \
      -kernel ./enoch_rev2902_boot \
      -smbios type=2 \
      -device ich9-intel-hda -device hda-duplex \
      -device ide-drive,bus=ide.2,drive=MacHDD \
      -drive id=MacHDD,if=none,file=./mac_hdd.img \
      -netdev tap,id=net0,ifname=tap0,script=no,downscript=no -device e1000-82545em,netdev=net0,id=net0,mac=52:54:00:c9:18:27 \
      -monitor stdio \
      -device ide-drive,bus=ide.0,drive=MacDVD \
      -drive id=MacDVD,if=none,snapshot=on,file=./'Install_macOS_Sierra_OS_X_10.12.iso'
      -vnc 0.0.0.0:0 -k en-us
```

安装完，将MacDVD -device及ISO -drive配置注释掉，启动虚拟机。

![macos_sierra](http://7xtc3e.com1.z0.glb.clouddn.com/kvm-run-macos/macos_10.12.jpg)

## 网络配置

虚拟机可正常运行后，接下来最重要的工作是网络配置，使虚拟机可正常连接网络。

既想让虚拟机可连接外网（但不暴露给外网，外网不能直接访问虚拟机），又不影响主机的网络访问，比较简单的方式是使用TAP interfaces，TAP直接将二层以太帧发送给用户态程序，这正是虚拟网卡(vNIC)期望的工作方式。虚拟机的IP，可通过主机dnsmasq提供的DHCP服务自动获取，然后将虚拟机网络流量通过主机iptables SNAT转发到外网，实现虚拟机访问网络。虚拟机网络连接拓扑：

![network](http://7xtc3e.com1.z0.glb.clouddn.com/kvm-run-macos/macos-network.jpg)

### 网桥virbr0创建

创建持久网桥，这样主机重启后就不必重新配置：

```
$ cat /etc/sysconfig/network-scripts/ifcfg-virbr0 
DEVICE=virbr0
TYPE=Bridge
BOOTPROTO=static
IPADDR=172.16.1.1
NETMASK=255.255.255.0
ONBOOT=yes
```

### iptables SNAT

配置`/etc/sysconfig/iptables`添加NAT规则:

```
-A POSTROUTING -s 172.16.1.0/24 ! -d 172.16.1.0/24 -j MASQUERADE
```

配置`/etc/sysctl.conf`开启IPv4转发功能：

```
net.ipv4.ip_forward = 1
```

### dnsmasq

配置dnsmasq，给虚拟机提供DHCP和DNS服务。

### 虚拟机网卡配置

将之前虚拟机启动脚本中的网卡配置，改成：

```
-netdev tap,id=net100,ifname=tap100,script=/etc/qemu-ifup.sh,downscript=no -device e1000-82545em,netdev=net100,id=net100,mac=00:16:3e:eb:ca:65
```

```
-netdev tap,id=net101,ifname=tap101,script=/etc/qemu-ifup.sh,downscript=no -device e1000-82545em,netdev=net101,id=net101,mac=00:16:3e:eb:ca:66
```

注意虚拟网卡是e1000-82545em，不是e1000，否则虚拟机网卡无法正常工作。参见[QEMU 2.1 Changelog](https://wiki.qemu.org/ChangeLog/2.1)：

> More variants of the e1000 NIC are supported (e1000-82540em, e1000-82544gc, e1000-82545em). The default e1000 fails with versions of Mac OS X starting at 10.9, while the new e1000-82545em works with all versions of Mac OS X starting at 10.6.

```
$ cat /etc/qemu-ifup.sh 
#!/bin/sh

switch=virbr0

echo "$0: adding tap interface \"$1\" to bridge \"$switch\""
ifconfig $1 0.0.0.0 up
brctl addif ${switch} $1

exit 0
```

启动虚拟机后，虚拟机就可正常访问网络了。

在使用过程中，发现一个问题，TAP的MAC地址是内核随机分配的，而网桥的MAC地址又是绑定到网桥上所有设备的MAC地址的最小值，这样如果有多台虚拟机同时运行，动态创建关闭虚拟机，会造成网桥MAC地址改变，进而会造成正在运行的虚拟机产生网络丢包，影响网络性能。下面用实验进行详细说明。

* 创建虚拟机前virbr0 MAC地址：

```
$ ifconfig virbr0
virbr0    Link encap:Ethernet  HWaddr 00:00:00:00:00:00  
          inet addr:172.16.1.1  Bcast:172.16.1.255  Mask:255.255.255.0
          inet6 addr: fe80::e0a8:9cff:fe32:c4fe/64 Scope:Link
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:956628 errors:0 dropped:0 overruns:0 frame:0
          TX packets:566603 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:474140424 (452.1 MiB)  TX bytes:1787345607 (1.6 GiB)
```

* 创建第一台虚拟机，virbr0和tap101 MAC地址：

```
$ ifconfig virbr0
virbr0    Link encap:Ethernet  HWaddr 62:93:B7:8A:03:B7  
          inet addr:172.16.1.1  Bcast:172.16.1.255  Mask:255.255.255.0
          inet6 addr: fe80::e0a8:9cff:fe32:c4fe/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:956628 errors:0 dropped:0 overruns:0 frame:0
          TX packets:566605 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:474140424 (452.1 MiB)  TX bytes:1787345827 (1.6 GiB)

$ ifconfig tap101
tap101    Link encap:Ethernet  HWaddr 62:93:B7:8A:03:B7  
          inet6 addr: fe80::6093:b7ff:fe8a:3b7/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:1 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 b)  TX bytes:90 (90.0 b)
```

虚拟机网卡：

![macos101-en0](http://7xtc3e.com1.z0.glb.clouddn.com/kvm-run-macos/macos101-en0.png)

* 在第一台虚拟机中，进行ping网络测试，同时启动第二台虚拟机。如果第二台虚拟机的TAP MAC地址小于第一台虚拟机TAP MAC地址，网桥virbr0 MAC地址会发生改变，第一台虚拟机中ping网络测试会产生丢包。

```
tap101    Link encap:Ethernet  HWaddr 62:93:B7:8A:03:B7  
          inet6 addr: fe80::6093:b7ff:fe8a:3b7/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:14676 errors:0 dropped:0 overruns:0 frame:0
          TX packets:22129 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:1138569 (1.0 MiB)  TX bytes:30607004 (29.1 MiB)

tap102    Link encap:Ethernet  HWaddr 2E:FA:07:80:C6:23  
          inet6 addr: fe80::2cfa:7ff:fe80:c623/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:1 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 b)  TX bytes:90 (90.0 b)

virbr0    Link encap:Ethernet  HWaddr 2E:FA:07:80:C6:23  
          inet addr:172.16.1.1  Bcast:172.16.1.255  Mask:255.255.255.0
          inet6 addr: fe80::e0a8:9cff:fe32:c4fe/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:977654 errors:0 dropped:0 overruns:0 frame:0
          TX packets:581415 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:475446338 (453.4 MiB)  TX bytes:1830140398 (1.7 GiB)
```

![macos101-ping](http://7xtc3e.com1.z0.glb.clouddn.com/kvm-run-macos/macos101-ping.png)

虚拟机需要重新通过ARP探测网桥MAC地址，才能重新恢复正常网络访问。关闭第二台虚拟机，virbr0 MAC地址又会发生改变，再次造成第一台虚拟机网络丢包。

虚拟机内tcpdump抓包，注意看，MAC地址从`62:93:B7:8A:03:B7`变成`2E:FA:07:80:C6:23`。

![macos101-tcpdump](http://7xtc3e.com1.z0.glb.clouddn.com/kvm-run-macos/macos101-tcpdump.png)

一种解决方案是，将所有虚拟机的TAP MAC地址配置成同一个值，这样网桥MAC地址不会发生改变，也不影响虚拟机网络访问。修改/etc/qemu-ifup.sh:

```
$ cat /etc/qemu-ifup.sh 
#!/bin/sh

switch=virbr0

echo "$0: adding tap interface \"$1\" to bridge \"$switch\""
ifconfig $1 hw ether fe:ff:ff:ff:ff:ff
ifconfig $1 0.0.0.0 up
brctl addif ${switch} $1

exit 0
```

参考：

- [QEMU User Documentation](https://qemu.weilnetz.de/doc/qemu-doc.html)
- [QEMU Configuring Guest Networking](http://www.linux-kvm.org/page/Networking)
- [How does the libvirt deal with the vnet mac address](https://www.redhat.com/archives/libvirt-users/2015-April/msg00149.html)
- [Linux bridge: MAC addresses and dynamic ports](http://backreference.org/2010/07/28/linux-bridge-mac-addresses-and-dynamic-ports/)
- [Tap Interfaces and Linux Bridge](http://www.innervoice.in/blogs/2013/12/08/tap-interfaces-linux-bridge/)
- [Linux Bridge - how it works](https://goyalankit.com/blog/linux-bridge)
- [Wikipedia TUN/TAP](https://en.wikipedia.org/wiki/TUN/TAP)
- [Linux kernel TUN/TAP device driver](https://www.kernel.org/doc/Documentation/networking/tuntap.txt)
- [How to configure a Linux bridge interface](http://xmodulo.com/how-to-configure-linux-bridge-interface.html)
