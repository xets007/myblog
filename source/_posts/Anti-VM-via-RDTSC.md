---
title: Anti-VM之RDTSC指令
date: 2016-09-12 12:08:17
categories: Security
tags:
- Anti-VM
- RDTSC
---

上篇[《Anti-VM之CPUID指令》](https://consen.github.io/2016/09/11/Anti-VM-via-CPUID/)，通过CPUID指令获取CPU信息，来判断环境是否是虚拟的。这篇从时间角度判断环境是否是虚拟的。

Intel和AMD为处理器增加了虚拟化扩展（Intel **VT-x**, AMD **AMD-V**），使得操作系统不做修改就可虚拟化。以Intel VT-x为例，VMM（Virtual Machine Monitor）运行在root模式，虚拟机运行在非root模式，在非root模式下，一些指令如RDMSR，WRMSR，CPUID需要在VMM才能执行，即会触发 **VM exit**, 从而带来时间上的开销，通过统计时间，就可以判断当前环境是否是虚拟的。

<!-- more -->

![blackhole](http://7xtc3e.com1.z0.glb.clouddn.com/blackhole.png)

> The Time Stamp Counter (TSC) is a 64-bit register present on all x86 processors since the Pentium. It counts the number of cycles since reset. The instruction RDTSC returns the TSC in EDX:EAX. In x86-64 mode, RDTSC also clears the higher 32 bits of RAX and RDX. Its opcode is 0F 31.

TSC是一个64位寄存器，统计CPU时钟周期，可以用来高精度统计时间。RDTSC指令读取TSC值。

RDTSC和CPUID都是非特权（unprivileged）指令，可以在用户态调用。

检测代码 [rdtsc.c](https://github.com/consen/demo/blob/master/c/syntax/asm/rdtsc.c) ：

```
#include <stdio.h>
#include <unistd.h>

#define TRUE 1
#define FALSE 0

static inline unsigned long long rdtsc_diff() {
    unsigned long long ret, ret2;
    unsigned eax, edx;
    __asm__ volatile("rdtsc" : "=a" (eax), "=d" (edx));
    ret  = ((unsigned long long)eax) | (((unsigned long long)edx) << 32);
    __asm__ volatile("rdtsc" : "=a" (eax), "=d" (edx));
    ret2  = ((unsigned long long)eax) | (((unsigned long long)edx) << 32);
    printf("(%llu - %llu) ", ret, ret2);
    return ret2 - ret;
}

static inline unsigned long long rdtsc_diff_vmexit() {
    unsigned long long ret, ret2;
    unsigned eax, edx;
    __asm__ volatile("rdtsc" : "=a" (eax), "=d" (edx));
    ret  = ((unsigned long long)eax) | (((unsigned long long)edx) << 32);
    /* vm exit forced here. it uses: eax = 0; cpuid; */
    __asm__ volatile("cpuid" : /* no output */ : "a"(0x00));
    /**/
    __asm__ volatile("rdtsc" : "=a" (eax), "=d" (edx));
    ret2  = ((unsigned long long)eax) | (((unsigned long long)edx) << 32);
    printf("(%llu - %llu) ", ret, ret2);
    return ret2 - ret;
}

int cpu_rdtsc() {
    int i;
    unsigned long long avg = 0, sub;
    for (i = 0; i < 10; i++) {
        sub = rdtsc_diff();
        avg = avg + sub;
        printf("rdtsc difference: %d\n", sub);
        usleep(500);
    }
    avg = avg / 10;
    printf("difference average is: %d\n", avg);
    return (avg < 1000 && avg > 0) ? FALSE : TRUE;
}

int cpu_rdtsc_force_vmexit() {
    int i;
    unsigned long long avg = 0, sub;
    for (i = 0; i < 10; i++) {
        sub = rdtsc_diff_vmexit();
        avg = avg + sub;
        printf("rdtsc_force_vmexit difference: %d\n", sub);
        usleep(500);
    }   
    avg = avg / 10; 
    printf("difference average is: %d\n", avg);
    return (avg < 1000 && avg > 0) ? FALSE : TRUE;
}

int main()
{
    printf("cpu_rdtsc: %d\n", cpu_rdtsc());
    printf("cpu_rdtsc_force_vmexit: %d\n", cpu_rdtsc_force_vmexit());

    return 0;
}
```

在物理机上运行结果：

```
(2724639288020470 - 2724639288020494) rdtsc difference: 24
(2724639289813128 - 2724639289813156) rdtsc difference: 28
(2724639291344684 - 2724639291344708) rdtsc difference: 24
(2724639292869608 - 2724639292869668) rdtsc difference: 60
(2724639294412600 - 2724639294412628) rdtsc difference: 28
(2724639295888716 - 2724639295888744) rdtsc difference: 28
(2724639297385732 - 2724639297385760) rdtsc difference: 28
(2724639298878740 - 2724639298878768) rdtsc difference: 28
(2724639300371292 - 2724639300371320) rdtsc difference: 28
(2724639301864924 - 2724639301864952) rdtsc difference: 28
difference average is: 30
cpu_rdtsc: 0
(2724639303466468 - 2724639303466608) rdtsc_force_vmexit difference: 140
(2724639304950316 - 2724639304950440) rdtsc_force_vmexit difference: 124
(2724639306452456 - 2724639306452576) rdtsc_force_vmexit difference: 120
(2724639307949500 - 2724639307949624) rdtsc_force_vmexit difference: 124
(2724639309446084 - 2724639309446208) rdtsc_force_vmexit difference: 124
(2724639310963224 - 2724639310963344) rdtsc_force_vmexit difference: 120
(2724639312455892 - 2724639312456012) rdtsc_force_vmexit difference: 120
(2724639313952608 - 2724639313952728) rdtsc_force_vmexit difference: 120
(2724639315444756 - 2724639315444880) rdtsc_force_vmexit difference: 124
(2724639316941332 - 2724639316941452) rdtsc_force_vmexit difference: 120
difference average is: 123
cpu_rdtsc_force_vmexit: 0

```


在VMware虚拟机上运行结果：

```
(2201234633835839 - 2201234633835857) rdtsc difference: 18
(2201234638344307 - 2201234638344325) rdtsc difference: 18
(2201234641440343 - 2201234641440361) rdtsc difference: 18
(2201234645264790 - 2201234645264814) rdtsc difference: 24
(2201234648970696 - 2201234648970714) rdtsc difference: 18
(2201234652584847 - 2201234652584865) rdtsc difference: 18
(2201234655954692 - 2201234655954710) rdtsc difference: 18
(2201234660290766 - 2201234660290790) rdtsc difference: 24
(2201234662928102 - 2201234662928126) rdtsc difference: 24
(2201234666890620 - 2201234666890638) rdtsc difference: 18
difference average is: 19
cpu_rdtsc: 0
(2201234670145762 - 2201234670148696) rdtsc_force_vmexit difference: 2934
(2201234674344607 - 2201234674347196) rdtsc_force_vmexit difference: 2589
(2201234677321168 - 2201234677322647) rdtsc_force_vmexit difference: 1479
(2201234681439262 - 2201234681441659) rdtsc_force_vmexit difference: 2397
(2201234684594947 - 2201234684596750) rdtsc_force_vmexit difference: 1803
(2201234689163149 - 2201234689165630) rdtsc_force_vmexit difference: 2481
(2201234693566946 - 2201234693568884) rdtsc_force_vmexit difference: 1938
(2201234699323186 - 2201234699325493) rdtsc_force_vmexit difference: 2307
(2201234702590461 - 2201234702592354) rdtsc_force_vmexit difference: 1893
(2201234706444177 - 2201234706446514) rdtsc_force_vmexit difference: 2337
difference average is: 2215
cpu_rdtsc_force_vmexit: 1

```

参考：

- [Wikipedia Time Stamp Counter](https://en.wikipedia.org/wiki/Time_Stamp_Counter)
- [rdtsc x86 instruction to detect virtual machines](http://blog.badtrace.com/tag/antivm/)
- [Virtualization Detection: New Strategies and Their Effectiveness](https://people.eecs.berkeley.edu/~cthompson/papers/virt-detect.pdf)
- [Xen TSC Mode HOWTO](https://xenbits.xen.org/docs/unstable/misc/tscmode.txt)
- [Xen Feature Levelling](http://xenbits.xen.org/docs/unstable/features/feature-levelling.html)
- [GCC-Inline-Assembly-HOWTO](http://www.ibiblio.org/gferg/ldp/GCC-Inline-Assembly-HOWTO.html)
- [Pafish](https://github.com/a0rtega/pafish)
