---
title: GnuPG实战
date: 2016-08-18 21:49:24
categories:
- Security
tags:
- GnuPG
---

这两天有一个大新闻，和美国国家安全局（NSA）有密切关系的方程式组织（`Equation Group`）遭到`The Shadow Brokers`攻击，大量黑客工具被泄露到网上，如果你感兴趣，一定已经下载下来把玩了一番:)。

将文件解压缩后，会看到文件用了[GnuPG](https://www.gnupg.org/)工具进行加密，简单了解了下GnuPG怎么用，现在分享出来。

<!-- more -->

```
$ unzip EQGRP-Auction-Files.zip -d EQGRP-Auction-Files
$ ls EQGRP-Auction-Files
eqgrp-auction-file.tar.xz.gpg
eqgrp-auction-file.tar.xz.gpg.sig
eqgrp-free-file.tar.xz.gpg
eqgrp-free-file.tar.xz.gpg.sig
public.key.asc
sha256sum.txt
sha256sum.txt.sig
```

## 安装

CentOS安装GnuPG：

```
# yum install gnupg2
```

## 对称加密/解密

加密解密使用相同的密钥。

### 加密：

```
$ gpg -c --cipher-algo AES256 demo.txt
```
按提示输入密码，生成`demo.txt.gpg`加密文件。

### 解密：

```
$ gpg demo.txt.gpg
```

按提示输入密码，生成`demo.txt`解密文件。

### 解密Equation Group:

`The Shadow Brokers`释放了文件`eqgrp-free-file.tar.xz.gpg`的密码`theequationgroup`，可以直接解密；文件`eqgrp-auction-file.tar.xz.gpg`出价5亿美刀拍卖，只能坐等后面好戏了:)。

```
$ gpg eqgrp-free-file.tar.xz.gpg
```

按提示输入密码：

![decrypt](http://7xtc3e.com1.z0.glb.clouddn.com/gpg-introduction/decrypt.png)

得到文件`eqgrp-free-file.tar.xz`，直接解压缩：

```
$ tar -xvf eqgrp-free-file.tar.xz
```

得到Fireware文件夹，所有秘密尽收眼底：

```
$ tree Firewall/ -L 1 -F
Firewall/
├── BANANAGLEE/
├── BARGLEE/
├── BLATSTING/
├── BUZZDIRECTION/
├── EXPLOITS/
├── OPS/
├── padding
├── SCRIPTS/
├── TOOLS/
└── TURBO/
```

## 非对称加密/解密

公钥加密，私钥解密。

### 生成密钥对：

```
$ gpg --gen-key
```

列出公钥:

```
$ gpg --list-keys
```

列出私钥:

```
$ gpg --list-secret-keys
```

导出公钥：

```
gpg -a -o public-key.txt --export key-identifier
```

这个公钥文件`pulibc-key.txt`就可以拿给别人用了。


### 加密：

如果需要使用的公钥不存在，需要提前导入。

```
gpg --encrypt --recipient key-identifier demo.txt

or 

gpg -e -r key-identifier demo.txt
```

生成加密文件demo.txt.gpg。

### 解密：

```
gpg --output demo.txt --decrypt demo.txt.gpg

or 

gpg demo.txt.gpg
```

生成解密文件demo.txt。

## 签名/验签
> 非对称加密的另一个用途是用于数字签名，签署者使用他的私钥（应用一个签名算法）来签署文档。验证者使用签署者的公钥（公开的）验证文档。当一个文档被签署时，任何人都能验证它，因为任何人都能访问签署者的公钥。由于私钥的保密性，签名是无法伪造的。

### 签名：

```
$ gpg --armor -output demo.txt.sig --detach-sign demo.txt

or

$ gpg -a -o demo.txt.sig -b demo.txt
```

按提示输入私钥密码，输出`demo.txt.sig`签名文件。

### 验签：

```
$ gpg --verify demo.txt.sig demo.txt
gpg: Signature made Thu 18 Aug 2016 05:08:22 PM CST using RSA key ID 26940C51
gpg: Good signature from "xikangjie (test) <xikangjie@consen.cn>"
```

如果demo.txt文件存在，可以省略.

```
$ gpg --verify demo.txt.sig 
```

### Equation Group验签：

```
$ gpg --verify sha256sum.txt.sig 
gpg: Signature made Mon 01 Aug 2016 11:23:02 AM CST using RSA key ID CB5C0C1B
gpg: Can't check signature: No public key
```

提示没有公钥，这就是文件`public.key.asc`的作用，将其导入：

```
$ gpg --import public.key.asc 
gpg: key CB5C0C1B: public key "The Shadow Broker <theshadowbroker@mail.i2p>" imported
gpg: Total number processed: 1
gpg:               imported: 1  (RSA: 1)
```

再次验签：

```
$ gpg --verify sha256sum.txt.sig
gpg: Signature made Mon 01 Aug 2016 11:23:02 AM CST using RSA key ID CB5C0C1B
gpg: Good signature from "The Shadow Broker <theshadowbroker@mail.i2p>"
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 8369 E676 9C13 F624 3103  15E6 0412 4F2C CB5C 0C1B
```
警告公钥不受信任，将其签入设为信任：

```
$ gpg --sign-key 'The Shadow Broker'
```

再次验签：

```
$ gpg --verify sha256sum.txt.sig 
gpg: Signature made Mon 01 Aug 2016 11:23:02 AM CST using RSA key ID CB5C0C1B
gpg: Good signature from "The Shadow Broker <theshadowbroker@mail.i2p>"
```

**参考文档：**

- [GPG入门教程](http://www.ruanyifeng.com/blog/2013/07/gpg.html)
- [RSA算法原理（一）](http://www.ruanyifeng.com/blog/2013/06/rsa_algorithm_part_one.html)
- [RSA算法原理（二）](http://www.ruanyifeng.com/blog/2013/07/rsa_algorithm_part_two.html)
- [使用 GnuPG 实现文件加密和数字签名](https://archboy.org/2013/05/15/gnupg-pgp-encrypt-decrypt-file-and-digital-signing-easy-tutorial/)
- [GPG Encryption Guide](http://www.tutonics.com/2012/11/gpg-encryption-guide-part-1.html)
- [NSA's Hacking Group Hacked! Bunch of Private Hacking Tools Leaked Online](http://thehackernews.com/2016/08/nsa-hacking-tools.html)
