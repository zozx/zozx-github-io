---
layout: post
title: "开启Secure Boot的情况下使用Linux的经历"
date:   2022-1-24
tags: ["SecureBoot","EFI","Linux"]
categories: ["EFI"]
---

### 环境
首先，说一下我这边的环境是可以正常使用的，但是并不确定是否会有特殊的机型不支持。

| 环境       | 名称             |
| :--------- | :--------------- |
| 系统       | Gentoo           |
| 电脑       | Alienware M15 R1 |
| Bootloader | systemd-boot     |

### 使用PreLoader (Microsoft签名)

这个方法适合懒得折腾的朋友。首先，下载PreLoader的方案有两个，一是手动去网站上下载，网站是`https://blog.hansenpartnership.com/linux-foundation-secure-boot-system-released`，二是我在我的overlay里打了个preloader-signed的ebuild，大家可以直接拿来用，overlay的地址是`https://github.com/zozx/zozx-overlay`，可以通过如下方式开启 (想直接从repo里拿走也随意)
```bash
eselect repository enable zozx-overlay
# 或者
layman -a zozx-overlay

# 接着同步并emerge一下
emerge --sync && emerge -v preloader-signed
```

接着就是配置PreLoader了

```bash
cp /usr/share/preloader-signed/{PreLoader,HashTool}.efi path-to-bootloader # path-to-bootloader请以自己机子的bootloader位置为主
efibootmgr -v -c -L "PreLoader" -l path-to-PreLoader.efi # 没有efibootmgr的自己装一下，path-to-Preloader.efi请以自己机子复制到esp后的位置为主
reboot # 请进入BIOS并自己开启Secure Boot
```

重启之后PreLoader会报Hash有问题，并且进入HashTool，添加一下Bootloader的efi文件和vmlinuz/BzImage.efi即可

### 尾声

这种方式相对比较简单，而且不用修改BIOS里边的PK,KEK,db，比较通用，我目前使用的是自己的签名，至于方式，可以参考`https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot#Using_your_own_keys`，注意要开启Setup Mode，还有就是sbkeysync报错的话可以使用BIOS里自带的功能来更换 (部分BIOS不支持)
