---
title: "我对rEFInd这个BootLoader的感受"
#date:   2022-1-3
date:	3-1-2022
tags: ["Bootloader","rEFInd","EFI"]
categories: ["EFI"]
---

### 前言
我经常在我的主力机上更换BootLoader，我使用过Systemd-Boot,EFI STUB(通过efibootmgr添加或在EFI固件中手动添加),GRUB2和SYSLINUX。这次我换到了rEFInd。

### 简单的介绍
rEFInd是一个仅支持EFI下使用的BootLoader，他的特色是有图形化的界面，每一个图标可以对应一个引导项，可以对应发行版选择图标，有着简单易懂的配置文件，定制性高。

### 优点
网上有不少好看的主题，动手能力强的人也可以自己定制主题。可以为启动项添加多个默认启动参数，可自动检测可执行的efi文件，同时也一样可以手动添加启动项，而且默认的配置文件中有范例

### 使用中发现的一些问题
虽然他有对多个发行版设置图标，但检测发行版的能力欠佳，在不设置的情况下，容易将内核认作未知发行版，而且它默认的主题说实话难以恭维。

### 尾声
总的来说，rEFInd算是一个很不错的BootLoader，EFI用户可以尝试一下。
