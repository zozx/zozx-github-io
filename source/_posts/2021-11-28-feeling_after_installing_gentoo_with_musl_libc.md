---
layout: post
title: "尝试以musl作为libc安装Gentoo之后的想法和建议"
date:   2021-11-28
tags: ["musl","Gentoo","libc","建议"]
categories: [Gentoo]
---

### 前言
昨天，我在虚拟机里装了一遍musl下的Gentoo，虽然总体上没什么大问题，但是还是有一点小坑的，我在这里就简单的说一下。

### stage3
解决完分区和网络后，就要开始挑选合适的stage3了，记得要挑选带有musl标志的stage3，例如这个`stage3-amd64-musl-20211121T170545Z.tar.xz`，不过记得一点，musl只能和OpenRC配合。(至少目前在Gentoo里是这样的)

### Overlays
- 使用musl，**最基本**的一点就是要添加一个叫做`musl`的overlay，可以手动添加或者使用如下命令:
```bash
eselect repository enable musl #适用于使用eselect-repository的用户
# 或者
layman -a musl #适用于使用layman的用户
```
- 然后我建议添加一下Gentoo-zh社区里一位菊苣的overlay，这里面解决了`musl`这个overlay以及main repo里都有问题的包，并且对全局clang用户也很友好，下边是添加的方式(因为我不是layman用户，不太清楚，就不讲如何通过layman来添加了):
```bash
eselect repository add 12101111-overlay git https://github.com/12101111/overlay.git
```

### 软件包的问题
部分在main repo里的包对musl没有支持，导致会无法成功编译，那么下面就说一下这个问题：
1. 首先我建议直接mask掉main repo和`musl`里的`dev-lang/rust`这个包，使用`12101111-overlay`里的，这样会减少问题，并且不建议使用`dev-lang/rust-bin`，会出现其他包的问题。
2. 然后第二点就是main repo里的`sys-kernel/linux-headers`这个包也是需要mask掉的，否则部分包将会出现问题，例如`sys-fs/btrfs-progs`

### 尾声
最后，我还是建议刚入门的朋友先练习一下再上musl，这个东西需要有耐心，看到报错的包要试试其他overlay里的有没有问题，因为main repo对musl的支持还不是很完善。
