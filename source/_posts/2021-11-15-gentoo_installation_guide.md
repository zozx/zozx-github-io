---
layout: post
title: "Gentoo 安装教程"
date: 2021-11-15
tags: ["Gentoo","Linux"]
categories: ["Installation Guide"]
---

### 前言
相信看到这篇文章的各位都已经对Gentoo这个Linux发行版有一定了解了，那么我们就直接开门见山，开始这篇教程，先说一下，本教程偏新手向，且比较简单，不能满足读者需求的话，可以看看这几篇：
#### 1. [Gentoo Linux 安装及使用指南](https://bitbili.net/gentoo-linux-installation-and-usage-tutorial.html)
#### 2. [在Mac上安装Gentoo Linux](https://www.yafa.moe/post/install-gentoo-on-mac/)
#### 3. [Gentoo 安装](https://litterhougelangley.club/blog/2021/05/21/gentoo/)
#### 4. [The Unorthodox Gentoo Handbook I](https://blog.bugsur.xyz/gentoo-handbook-installation/)
#### 5. [Gentoo AMD64 Handbook](https://wiki.gentoo.org/wiki/Handbook:AMD64)

### 打算

下边是我这次安装时的情况，本文以该情况为主，其他情况有问题的朋友可以在下面评论去直接问，看到的我都会尽我所能地回答。

| /        | 选择             |
| :---------- | :--------------- |
| libc        | glibc            |
| init script | systemd          |
| bootloader  | GRUB             |
| kernel      | liquorix-sources |

###  刻录USB

首先在下载一个livecd, 我推荐manjaro的或者fedora（电报里的朋友推荐ubuntu也不错，我都用过，喜欢fedora的，但是arch系有genfstab这个方便的fstab生成器，所以还是要看个人)，具体可以去`https://mirrors6.tuna.tsinghua.edu.cn`或者自己喜欢的镜像站下载
下载完之后如果你是linux用户，可以通过以下步骤刻录

```bash
dd if=<iso文件> of=/dev/<device> bs=1M status=progress
```

如果你是windows用户，则可以下载`rufus`来刻录，因为`rufus`有图形界面，就不细讲了

### 配置网络环境

进入livecd后，首先当然是要联网。如果你是有线网，则无需配置，一般已经通过`NetworkManager`或者`dhcpcd`连接

而如果你是使用无线网，可以使用`NetworkManager`(gui/tui/cli工具)或者`wpa_supplicant/iwd`(cli工具)来连接，下边使用`wpa_supplicant`的方式来连接:

```bash
wpa_passphrase <ESSID> <Password> > /etc/wpa_supplicant.conf
wpa_supplicant -iwlan0 -c/etc/wpa_supplicant.conf #wlan0根据情况更换，可能会叫做wlp112s0或其他名字
```

### 配置分区

首先，我们查看一下存储设备

```bash
fdisk -l
# 或者
lsblk
```

然后我推荐通过cfdisk进行图形化的分区

```bash
cfdisk -z /dev/nvme0n1 #nvme0n1根据情况更换
```

![我的分区](/images/2021-11-17_19-00.png)

分区结束后，轮到了格式化环节，这里，我的方案是:

| 名称           | 用途/大小                                            | 挂载点 | 文件系统 |
| -------------- | ---------------------------------------------------- | ------ | -------- |
| /dev/nvme0n1p1 | esp分区/100M(精简内核空间占用小，其实这里算是给多了) | /boot  | Fat32    |
| /dev/nvme0n1p2 | swap分区/16G(一般来说swap分区设置为内存的1-2倍即可)  | *none* | swap     |
| /dev/nvme0n1p3 | 根分区/余下全部(这里我没有细分/home等分区)           | /      | Btrfs    |

下面是具体操作

```bash
mkfs.fat -F 32 /dev/nvme0n1p1 #创建fat32的esp分区
mkswap /dev/nvme0n1p2 #创建swap分区
mkfs.btrfs /dev/nvme0n1p3 #创建根分区，使用btrfs
```

因为我使用Btrfs子分区作为根分区，所以下边还有一些小操作，不感兴趣的大家可以忽略

```bash
mount -t btrfs /dev/nvme0n1p3 /mnt
cd /mnt
btrfs subvolume create vol_root
cd /
umount -lR /mnt
```

下边挂载并进入分区

```bash
mkdir -p /mnt/gentoo
mount -t btrfs -o relatime,rw,compress=zstd:10,space_cache=v2,subvol=vol_root /dev/nvme0n1p3 /mnt/gentoo #使用其他文件系统的请看下一行
mount /dev/nvme0n1p3 /mnt/gentoo #跟着我使用btrfs的朋友请看上一行
mkdir -p /mnt/gentoo/boot
mount /dev/nvme0n1p1 /mnt/gentoo/boot
swapon /dev/nvme0n1p2
cd /mnt/gentoo
```

### 配置stage3

从任意一个镜像源下载stage3，如果在tty下，可以选择使用lynx，命令如下

```bash
lynx https://mirrors6.tuna.tsinghua.edu.cn/gentoo/releases/amd64/autobuilds
```

具体的stage3包要自己选择，可以选择默认的init script，libc以及profile

我这里就以systemd为例

下载完毕后通过以下命令解压

```bash
tar xpvf stage3-amd64-*.tar.xz --xattrs-include='*.*' --numeric-owner
```

解压完配置一下fstab，如果是livecd是arch系的，可以使用如下命令直接进行生成

```bash
genfstab -U /mnt/gentoo > /mnt/gentoo/etc/fstab
```

接着修改一下`make.conf`，我在这里就简单的展示一下我的(经过提醒，增加了些许注释)，具体可以自己微调，不建议直接抄功课，例如这里的`skylake`就是针对我的Coffee Lake架构的CPU写的，不同架构不一样

```bash
# These settings were set by the catalyst build script that automatically
# built this stage.
# Please consult /usr/share/portage/config/make.conf.example for a more
# detailed example.
NTHREADS=12 #这里是线程数

COMMON_FLAGS="-march=skylake -O3 -pipe -fgraphite-identity -floop-nest-optimize -fno-stack-protector -fno-align-functions -fno-align-jumps -fno-align-loops -fno-align-labels" #这个COMMON_FLAGS需要给gcc添加graphite的use才可以用，不知道CPU架构的可以上wiki看看或者直接使用-march=native
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"
FFLAGS="${COMMON_FLAGS}"
FCFLAGS="${COMMON_FLAGS}"
LDFLAGS="-Wl,-O3 -Wl,--as-needed -Wl,--hash-style=gnu -Wl,--sort-common -Wl,--strip-all" #这个LDFLAGS会使networkmanager这个包过不了，需要手动设置env，不建议新手使用
RUSTFLAGS="-C opt-level=3 -C target-cpu=skylake" #不知道CPU架构的可以上wiki看看或者直接使用-march=native

# NOTE: This stage was built with the bindist Use flag enabled
PORTDIR="/var/db/repos/gentoo"
DISTDIR="/var/cache/distfiles"
PKGDIR="/var/cache/binpkgs"
PORTAGE_TMPDIR="/tmp" #在systemd下，/tmp目录默认为tmpfs，即内存，不建议内存小的朋友使用

# This sets the language of build output to English.
# Please keep this setting intact when reporting bugs.
LC_MESSAGES=C

MAKEOPTS="-j${NTHREADS} -l${NTHREADS}"
PORTAGE_NICENESS=15
GENTOO_MIRRORS="https://mirrors6.tuna.edu.cn/gentoo" #请自行选择一个比较快的站点
USE="systemd lto pgo graphite ccache staging bluetooth alsa pulseaudio ffmpeg openssl network wifi networkmanager connection-sharing iptables zstd lz4 7zip rar btrfs policykit dbus qemu vdpau vaapi vulkan vkd3d d3d9 nvidia nvenc steamfonts trayicon systray -pipewire -joystick -games -education -wayland -xinerama -firewall -ppp -iwd -dhclient -elogind -kaccounts -webengine -kwallet -bittorrent -phonon -vlc -gnome -gnome-keyring -gnome-shell -gnome-online-accounts -passwdqc -bindist -clang -ssp -dhcpcd -netifrc -consolekit -doc -gtk-doc -handbook -spell -grub -oss -gpm" #USE这个东西因个人需求与电脑配置而异，不建议直接复制
ACCEPT_KEYWORDS="~amd64" #不建议新手直接全局开启带有~的keywords，随时可能出现未知问题
ACCEPT_LICENSE="*" #建议不计较license的朋友直接开*，这样可以省不少事
GRUB_PLATFORMS="efi-64" #使用GRUB+EFI的话请添加此行
EMERGE_DEFAULT_OPTS="--keep-going --with-bdeps=y --jobs=${NTHREADS} --load-average=${NTHREADS}"
L10N="en-US zh-CN en zh"
LINGUAS="en_US zh_CN en zh"
VIDEO_CARDS="nvidia intel i965 iris" #我这里是Intel核显加上NVIDIA的独显
ALSA_CARDS="hda-intel"
INPUT_DEVICES="libinput" #这个基本上可以算是承包大部分输入设备了
LLVM_TARGETS="X86 NVPTX" #一般只开X86即可，N卡可以开多个NVPTX，A卡可以开多个AMDGPU
RUBY_TARGETS="ruby30"
ABI_X86="64 32" #这个不建议开，我是因为要用wine-staging,lutris等包才贪方便装的
MICROCODE_SIGNATURES="-S" #如果要将intel的microcode编入内核就请留下这行
FEATURES="ccache" #安装ccache前不要打开
CCACHE_DIR="/var/cache/ccache" #安装ccache前不要打开
CPU_FLAGS_X86="aes avx avx2 f16c fma3 mmx mmxext pclmul popcnt rdrand sse sse2 sse3 sse4_1 sse4_2 ssse3" #这个因CPU而异，可以通过cpuid2cpuflags查看
CONFIG_PROTECT="/usr/share/sddm/scripts/Xsetup"
UNINSTALL_IGNORE="/bin /lib /lib64 /sbin /usr/sbin" #不进行usr-merge的朋友请忽略这一行
```

然后再通过如下命令设置main repo的repos.conf

```bash
mkdir -p /mnt/gentoo/etc/portage/repos.conf
cp /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
```

然后把`sync-uri`处修改一下，建议rsync的话使用北外的，如下

```bash
sync-uri = rsync://mirrors.bfsu.edu.cn/gentoo-portage
```

接着就是chroot进系统了，命令如下

```bash
cp --dereference /etc/resolv.conf /mnt/gentoo/etc/resolv.conf
mount -t proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
chroot /mnt/gentoo /bin/bash
env-update && . /etc/profile
```

之后是通过命令`emerge-webrsync`同步repo，同步完之后让我们再修改一些package的use，命令如下

```bash
echo 'app-text/ghostscript-gpl -l10n_zh-CN' > /etc/portage/package.use/ghostscript-gpl #为了避免安装宋体，去掉该包的zh-CN支持
```

如果需要更强的性能，并且时间充裕，可以开启`pgo`以及`lto`还有`graphite`优化，操作如下

```bash
echo 'sys-devel/gcc pgo lto graphite' > /etc/portage/package.use/gcc
emerge -v1 gcc
eselect gcc list
eselect gcc set X #X为更新版的gcc
env-update && . /etc/profile
emerge --depclean
```

接着读一下新闻: `eselect news read`，下一步就是更新`@world`，使用以下命令:

```bash
USE=-bluetooth emerge -1 python #此处是为了脱离循环依赖
emerge -avuDN @world
emerge --depclean
```

结束之后通过`etc-update`解决/etc里新增的文件

然后就是配置时间和locale以及hostname

```bash
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/timezone
nano -w /etc/locale.gen
------------------------------------------------
        C.UTF8 UTF-8
        en_US.UTF-8 UTF-8
        zh_CN.UTF-8 UTF-8
------------------------------------------------
locale-gen
eselect locale list #这里看看就好，暂时不用改，改了中文会乱码
eselect locale set n #n是对应的数字
echo 'ALIENWARE' > /etc/hostname #设置主机名，如果不设置，则可能为(none)或localhost
```

安装一些小工具

```bash
emerge -av networkmanager dev-vcs/git btrfs-progs neovim eselect-repository systemd-cron doas mlocate intel-microcode grub:2 #针对个人更改,例如大家如果更喜欢sudo,就可以把doas换成sudo(也可以都不用就是了)
```

然后可以针对个人进行一些配置：

```bash
systemctl enable NetworkManager
systemctl enable cron.target
#如果使用sudo,则使用visudo修改文件，设置权限
nvim /etc/doas.conf:
------------------------------------------------
        permit keepenv :wheel
        permit nopass keepenv root
------------------------------------------------
#如果像我一样去掉了passwdqc的use的话，不用如下配置也可以使用一般的密码
nvim /etc/security/passwdqc:
------------------------------------------------
        min=3,3,3,3,3
        max=8
        passphrase=0
        match=4
        similar=permit
        random=47
        enforce=none
        retry=3
------------------------------------------------
passwd
ln -sf /proc/self/mounts /etc/mtab
systemd-machine-id-setup
```

### 配置内核与bootloader

这里的话我推荐使用Houge Langley维护的`xanmod-hybrid`或者`liquorix-sources`，具体步骤如下

```bash
eselect repository enable gentoo-zh
mkdir -p /etc/portage/package.accept_keywords; echo 'sys-kernel/liquorix-sources ~amd64' >> /etc/portage/package.accept_keywords/liquorix-sources #这里说明一下，如果make.conf中ACCEPT_KEYWORDS="~amd64"，就不需要该步骤
emerge -av liquorix-sources
eselect kernel set 1
cd /usr/src/linux
make mrproper
cp /var/db/repos/gentoo-zh/sys-kernel/liquorix-sources/config/default-config /usr/src/linux/.config #复制一份默认的配置
make modules_prepare
make menuconfig
make -jX -lX #此处X为线程数，可通过lscpu查看
make modules_install
make install
```

具体配置的话我就不细讲了，然后如果想节约时间的话，可以使用`gentoo-kernel-bin`，具体步骤很简单，就是`emerge -av gentoo-kernel-bin`

接着配置一下GRUB

```bash
# 使用EFI的话，请用如下命令
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB # --efi-directory后面接的是esp分区
# BIOS的话，则使用如下命令
grub-install --target=i386-pc /dev/sdX # /dev/sdX指的是系统所在硬盘，根据情况自己查看。

#配置grub.cfg
grub-mkconfig -o /boot/grub/grub.cfg
```

然后我们简单的创建一个个人使用的用户

```bash
#下边的zozx是用户名，按需更改
useradd -m -G wheel,video,kvm,usb,users,portage,plugdev zozx
passwd zozx
```

### 尾声

退出chroot环境，取消挂载，然后重启

```bash
exit
umount -lR /mnt/gentoo
reboot
```
