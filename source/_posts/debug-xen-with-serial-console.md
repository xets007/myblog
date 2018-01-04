---
title: 使用串口调试Xen
date: 2017-07-26 19:56:45
categories:
- Virtualization
tags:
- Xen
- Serial Console 
---

在使用Xen的过程中，会遇到各种问题，掌握一些常见的调试技巧，对于解决问题能起到事半功倍的效果。网上一些Xen调试相关的资源：

1. [Debugging Xen](https://wiki.xenproject.org/wiki/Debugging_Xen)
2. [Xen Serial Console](https://wiki.xenproject.org/wiki/Xen_Serial_Console)
3. [Xen Debugging](http://www-archive.xenproject.org/files/xensummit_intel09/xen-debugging.pdf)

Xen hypervisor的运行信息写到了内存中，可以通过`xl dmesg`命令显示，并没有写入到日志文件，所以不太方便查看。如果遇到更严重的情况，Xen在启动阶段就出错了，则无法进入Dom0运行`xl dmesg`命令查看启动信息，只能通过串口查看。

使用串口调试Xen，需要两台物理机，通过串口线连接，一台运行Xen，另一台通过串口查看输出信息。

<!-- more -->

## 串口线选择

串口线注意选择交叉线而不是直连线，一端用于发送，一端用于接收。

![crossover](http://7xtc3e.com1.z0.glb.clouddn.com/debug-xen-with-serial-console/crossover.png)

接口也要选对，看是公对公，母对母，还是公对母。

![serial_cable](http://7xtc3e.com1.z0.glb.clouddn.com/debug-xen-with-serial-console/seriale_cable.jpg)

## 串口连通性测试

在对Xen进行串口配置之前，可以先测试下串口连通性。

查看串口信息，这里有两个串口（COM1和COM2），Linux下叫做ttyS0和ttyS1。

```
$ dmesg  | grep tty
console [tty0] enabled
00:03: ttyS1 at I/O 0x2f8 (irq = 3, base_baud = 115200) is a 16550A
serial8250: ttyS0 at I/O 0x3f8 (irq = 4, base_baud = 115200) is a 16550A
```

在两台设备上同时运行`minicom -s`，配置串口。

![minicom](http://7xtc3e.com1.z0.glb.clouddn.com/debug-xen-with-serial-console/minicom.png)

配好后，输入任意字符，如果在另一台设备的`minicom`中能看到对应的输出，说明串口连接正常。

## Xen串口配置

在运行Xen的设备上，配置Grub开机启动项，Xen参数添加`loglvl=all guest_loglvl=all console=com2 com2=115200,8n1`，Dom0内核参数添加`console=hvc0`。

如果不想在串口中错过每一条信息，可在Xen参数添加`sync_console`，强制同步串口输出，当然这只适用于调试，因为需要将所有信息从串口输出，严重影响hypervisor性能，虚拟机会特别卡。

## 串口调试Xen

Xen设备配置好后，在另一台设备上运行`minicom -s -C xen.log`，（`-C xen.log`将串口输出同时保存在xen.log文件，方便后续查看）,重启Xen设备。

Xen设备启动成功后，在串口上看到是Dom0，按6次`Ctrl-a`进入Xen。就可以看到虚拟机运行过程中，hypervisor的输出信息，可以更改Xen源码，添加更多printk()日志输出。也可以通过Xen debug keys获取相应信息，输入`h`，输出可用的kyes，输入key，输出对应信息。

```
(XEN) 'h' pressed -> showing installed handlers
(XEN)  key '%' (ascii '25') => trap to xendbg
(XEN)  key '*' (ascii '2a') => print all diagnostics
(XEN)  key '0' (ascii '30') => dump Dom0 registers
(XEN)  key 'A' (ascii '41') => toggle alternative key handling
(XEN)  key 'D' (ascii '44') => dump VT-x EPT tables
(XEN)  key 'H' (ascii '48') => dump heap info
(XEN)  key 'I' (ascii '49') => dump HVM irq info
(XEN)  key 'M' (ascii '4d') => dump MSI state
(XEN)  key 'N' (ascii '4e') => trigger an NMI
(XEN)  key 'Q' (ascii '51') => dump PCI devices
(XEN)  key 'R' (ascii '52') => reboot machine
(XEN)  key 'V' (ascii '56') => dump iommu info
(XEN)  key 'a' (ascii '61') => dump timer queues
(XEN)  key 'c' (ascii '63') => dump ACPI Cx structures
(XEN)  key 'd' (ascii '64') => dump registers
(XEN)  key 'e' (ascii '65') => dump evtchn info
(XEN)  key 'g' (ascii '67') => print grant table usage
(XEN)  key 'h' (ascii '68') => show this message
(XEN)  key 'i' (ascii '69') => dump interrupt bindings
(XEN)  key 'm' (ascii '6d') => memory info
(XEN)  key 'n' (ascii '6e') => NMI statistics
(XEN)  key 'o' (ascii '6f') => dump iommu p2m table
(XEN)  key 'q' (ascii '71') => dump domain (and guest debug) info
(XEN)  key 'r' (ascii '72') => dump run queues
(XEN)  key 's' (ascii '73') => dump softtsc stats
(XEN)  key 't' (ascii '74') => display multi-cpu clock info
(XEN)  key 'u' (ascii '75') => dump NUMA info
(XEN)  key 'v' (ascii '76') => dump VT-x VMCSs
(XEN)  key 'w' (ascii '77') => synchronously dump console ring buffer (dmesg)
(XEN)  key 'z' (ascii '7a') => dump IOAPIC info
```

参考：

* [How to Capture Xen Hypervisor and Kernel Messages using a Serial Cable](https://en.opensuse.org/How_to_Capture_Xen_Hypervisor_and_Kernel_Messages_using_a_Serial_Cable)
* [Xen Hypervisor Command Line Options](https://xenbits.xen.org/docs/unstable/misc/xen-command-line.html)
* [Crossover or "Null Modem" vs. Straight Through Serial Cable](https://www.decisivetactics.com/support/view?article=crossover-or-null-modem-vs-straight-through-serial-cable)
* [8-N-1](https://en.wikipedia.org/wiki/8-N-1)
