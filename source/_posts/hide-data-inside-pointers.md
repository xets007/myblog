---
title: 在指针中隐藏数据
date: 2017-01-07 22:29:24
tags:
- pointer
- memory alignment
- bitwise operation 
categories:
- Linux
---

上周的博客文章[《Linux内核模块升级》](https://consen.github.io/2016/12/30/upgrade-linux-kernel-module/)中，讲了通过升级Linux内核驱动模块xen-evtchn，解决掉一个Xen卡死问题，原因是port_user中port重用，触发kernel BUG。evtchn中port_user的实现，对指针的应用很巧妙：

1. 将指针存储在无符长整形（unsigned long）中。
2. 将port是否enabled保存在无符长整形从低地址开始的第一个bit中。

<!-- more -->

下面是从[evtchn.c](https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/tree/drivers/xen/evtchn.c?h=v3.10.20)中抽取的关于port_user的实现：

```
struct per_user_data {
    ...
};

static unsigned long *port_user;

...

static inline struct per_user_data *get_port_user(unsigned port)
{
    return (struct per_user_data *)(port_user[port] & ~1);
}

static inline void set_port_user(unsigned port, struct per_user_data *u)
{
    port_user[port] = (unsigned long)u;
}

static inline bool get_port_enabled(unsigned port)
{
    return port_user[port] & 1;
}

static inline void set_port_enabled(unsigned port, bool enabled)
{
    if (enabled)
        port_user[port] |= 1;
    else
        port_user[port] &= ~1;
}

...

static int __init evtchn_init(void)
{

    ...

    port_user = kcalloc(NR_EVENT_CHANNELS, sizeof(*port_user), GFP_KERNEL);
    if (port_user == NULL)
        return -ENOMEM;

    ...
}
```

C语言中对指针进行位操作是非法的，x64系统中指针占用8字节，unsigned long也占用8字节，所以将指针存储在unsigned long中，对unsigned long进行位操作。但是问题来了，将port是否enabled保存在第一个bit，那就得要求第一个bit在位操作之前一定为0。指针转换为unsigned long后，从低地址开始的第一个bit真的为0吗？

这就涉及到内存对齐了。CPU从内存中读写数据不是一次1字节，而是一次访问word size字节，一般为4字节或8字节，这样做当然是为了提升性能，所以一个变量在内存中的地址是可以整除1,2,4或8的，这就是内存对齐。如在x64系统下，unsigned long占用8字节，那么一个unsigned long变量在内存中的地址一定是可以被8整除，如地址可以为0x7fff4fcce908，但绝不会为0x7fff4fcce901，0x7fff4fcce907等不能被8整除的数值。

**任何能被8整除的整数，转换为二进制后，从低地址开始的3个bit一定为0。**

这个也很容易理解，并没有复杂的数学推理，整数8的二进制为`1000`，所以任何能被8整除的整数，转换为二进制后，从低地址开始的3个bit一定为0。举一反三下，任何能被4整除的整数，转换为二进制后，从低地址开始的2个bit一定为0，任何能被2整除的整数，转换为二进制后，从低地址开始的1个bit一定为0。4的二进制为`100`，2的二进制为`10`。

了解了内存对齐机制，上面port_user的实现就很好理解了。

其实，在Linux内核中，**红黑树的实现就利用了指针隐藏数据**。红黑树节点的定义为rb_node（[include/linux/rbtree.h](https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/include/linux/rbtree.h#n36)）：

```
struct rb_node {
    unsigned long  __rb_parent_color;
    struct rb_node *rb_right;
    struct rb_node *rb_left;
} __attribute__((aligned(sizeof(long))));
```

`__rb_parent_color`既用来存储父节点指针，也用来存储节点颜色，第一个bit为0表示红1表示黑。

再看下对父节点指针的访问（[include/linux/rbtree.h](https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/include/linux/rbtree.h#n48)）：

```
#define rb_parent(r)   ((struct rb_node *)((r)->__rb_parent_color & ~3))
```

对颜色的访问（[include/linux/rbtree_augmented.h](https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/include/linux/rbtree_augmented.h#n102)）:

```
#define __rb_color(pc)     ((pc) & 1)
#define rb_color(rb)       __rb_color((rb)->__rb_parent_color)
```

---

说到内存对齐，前段时间发过一条[微博](http://weibo.com/1996731561/Emg4Ytfty)，只有一行代码：

```
127         contentsz = (length + 7) & ~7
```

这行代码摘自Xen源码[tools/python/xen/migration/libxl.py](http://xenbits.xenproject.org/gitweb/?p=xen.git;a=blob;f=tools/python/xen/migration/libxl.py;hb=HEAD#l127)，很巧妙。简单讲下这行代码的用途，假设现在以8字节为单位存储数据，不足8字节的用0补齐，那么现在一段数据data的实际长度为length，存储后的长度则为contentsz，这样就可以算出总共填充了（contentsz - length）字节的0。如length为8，contentsz则为8，length为9，contentsz则为16。

---

参考：

- [Hide data inside pointers](http://arjunsreedharan.org/post/105266490272/hide-data-inside-pointers)
- [Data Alignment](http://www.songho.ca/misc/alignment/dataalign.html)
- [Word (computer architecture)][1]

[1]: https://en.wikipedia.org/wiki/Word_(computer_architecture)
