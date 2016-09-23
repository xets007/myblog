---
title: Anti-VM之CPUID指令
date: 2016-09-11 21:10:03
categories: Security
tags:
- Anti-VM
- CPUID
---

相对于传统的静态分析检测，**动态分析**技术可以获得恶意样本的详细行为信息。恶意样本作者为了规避动态检测，采取了各种手段，如Anti-VM, Anti-Sandbox, Anti-Debugging。

**Anti-VM**是为了检测当前运行环境是否是虚拟环境，如果是，恶意样本会停止恶意行为，让安全研究员误认为此样本是良性的。比如恶意样本如果在物理环境下运行，会去连接C&C服务器，当检测到是在虚拟环境下运行时，则去连接正常的域名。

<!-- more -->

![redpillbluepill](http://7xtc3e.com1.z0.glb.clouddn.com/MatrixBluePillRedPill2.jpg)

动态检测不管借助哪种虚拟化工具，如VMware、VirtualBox、Xen、KVM等，都会留下蛛丝马迹，表征当前环境是虚拟的。

接下来就借助`CPUID`指令，检测当前环境是否是虚拟的。

`CPUID`指令用来获取CPU信息，将功能号作为参数传入`EAX`寄存器，CPUID将信息返回到`EAX、EBX、ECX、EDX`寄存器。下面是两个检测虚拟环境的功能号：

- EAX = 1，获取CPU功能信息，如果ECX第31位为1， 则表明有hypervisor存在，即环境是虚拟的。
- EAX = 0x40000000, 获取hypervisor信息，返回12字节长度字符串：

```
"KVMKVMKVM\0\0\0"    KVM
"Microsoft Hv"       Microsoft Hyper-V or Windows Virtual PC
"VMwareVMware"       VMware
"XenVMMXenVMM"       Xen
"prl hyperv  "       Parallels
"VBoxVBoxVBox"       VirtualBox
```

检测代码 [cpuid.c](https://github.com/consen/demo/blob/master/c/syntax/asm/cpuid2.c) ：

```c
#include <stdio.h>
#include <stdint.h>
#include <string.h>

#define TRUE  1
#define FALSE 0

typedef char* string;

static inline int cpuid_hv_bit() {
    int ecx;
    __asm__ volatile("cpuid" \
            : "=c"(ecx) \
            : "a"(0x01));
    return (ecx >> 31) & 0x1;
}

static inline void cpuid_hv_vendor_00(char * vendor) {
    int ebx = 0, ecx = 0, edx = 0;

    __asm__ volatile("cpuid" \
            : "=b"(ebx), \
            "=c"(ecx), \
            "=d"(edx) \
            : "a"(0x40000000));
    sprintf(vendor  , "%c%c%c%c", ebx, (ebx >> 8), (ebx >> 16), (ebx >> 24));
    sprintf(vendor+4, "%c%c%c%c", ecx, (ecx >> 8), (ecx >> 16), (ecx >> 24));
    sprintf(vendor+8, "%c%c%c%c", edx, (edx >> 8), (edx >> 16), (edx >> 24));
    vendor[12] = 0x00;
}

int cpu_hv() {
    return cpuid_hv_bit() ? TRUE : FALSE;
}

void cpu_write_hv_vendor(char * vendor) {
    cpuid_hv_vendor_00(vendor);
}

int cpu_known_vm_vendors() {
    const int count = 6;
    int i;
    char cpu_hv_vendor[13];
    string strs[count];
    strs[0] = "KVMKVMKVM\0\0\0"; /* KVM */
    strs[1] = "Microsoft Hv"; /* Microsoft Hyper-V or Windows Virtual PC */
    strs[2] = "VMwareVMware"; /* VMware */
    strs[3] = "XenVMMXenVMM"; /* Xen */
    strs[4] = "prl hyperv  "; /* Parallels */
    strs[5] = "VBoxVBoxVBox"; /* VirtualBox */
    cpu_write_hv_vendor(cpu_hv_vendor);
    for (i = 0; i < count; i++) {
        if (!memcmp(cpu_hv_vendor, strs[i], 12)) return TRUE;
    }
    return FALSE;
}

int main()
{
    char cpu_hv_vendor[13];
    cpu_write_hv_vendor(cpu_hv_vendor);

    printf("CPU Hypervisor Bit: %d\n", cpu_hv());
    printf("Hypervisor: %s, is known vm vendor: %s\n", cpu_hv_vendor, cpu_known_vm_vendors() ? "true" : "false");

    return 0;
}
```

在物理机下运行结果：

```
CPU Hypervisor Bit: 0
Hypervisor: , is known vm vendor: false
```

在VMware虚拟机下运行结果：

```
CPU Hypervisor Bit: 1
Hypervisor: VMwareVMware, is known vm vendor: true
```

参考：

- [Wikipedia CPUID](https://en.wikipedia.org/wiki/CPUID)
- [x86 architecture CPUID](http://www.sandpile.org/x86/cpuid.htm)
- [Xen Feature Levelling](http://xenbits.xen.org/docs/unstable/features/feature-levelling.html)
- [GCC-Inline-Assembly-HOWTO](http://www.ibiblio.org/gferg/ldp/GCC-Inline-Assembly-HOWTO.html)
- [Pafish](https://github.com/a0rtega/pafish)
