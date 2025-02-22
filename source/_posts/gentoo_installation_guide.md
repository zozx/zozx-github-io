---
title: "Gentoo 安装教程"
#date: 2021-11-15
date:	15-11-2021
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

接着修改一下`make.conf`，这里贴出我自己的(经过提醒，增加了些许注释)，不建议直接抄.
```bash
# These settings were set by the catalyst build script that automatically
# built this stage.
# Please consult /usr/share/portage/config/make.conf.example for a more
# detailed example.
NTHREADS=12 # 线程数

COMMON_FLAGS="-march=skylake -O3 -pipe -fgraphite-identity -floop-nest-optimize -fno-stack-protector -fno-align-functions -fno-align-jumps -fno-align-loops -fno-align-labels"
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"
FFLAGS="${COMMON_FLAGS}"
FCFLAGS="${COMMON_FLAGS}"
LDFLAGS="-Wl,-O3 -Wl,--as-needed -Wl,--hash-style=gnu -Wl,--sort-common -Wl,--strip-all" # 该LDFLAGS会导致networkmanager装不了
RUSTFLAGS="-C opt-level=3 -C target-cpu=skylake"

# NOTE: This stage was built with the bindist Use flag enabled
PORTDIR="/var/db/repos/gentoo"
DISTDIR="/var/cache/distfiles"
PKGDIR="/var/cache/binpkgs"
PORTAGE_TMPDIR="/tmp"

# This sets the language of build output to English.
# Please keep this setting intact when reporting bugs.
LC_MESSAGES=C

MAKEOPTS="-j${NTHREADS} -l${NTHREADS}"
PORTAGE_NICENESS=15
PORTAGE_IONICE_COMMAND="ionice -c 3 -p \${PID}"
GENTOO_MIRRORS="https://mirrors.tuna.tsinghua.edu.cn/gentoo"
FETCHCOMMAND="/usr/bin/aria2c -d \${DISTDIR} -o \${FILE} --allow-overwrite=true --max-tries=8 --max-file-not-found=2 --max-concurrent-downloads=128 --connect-timeout=15 --timeout=15 --split=128 --min-split-size=2M --lowest-speed-limit=20K --max-connection-per-server=16 --uri-selector=feedback \${URI}" # 此处是用aria2代替wget
RESUMECOMMAND="${FETCHCOMMAND}"
USE="lto pgo graphite jemalloc ccache clang staging zsh-completion bluetooth pulseaudio pipewire screencast ffmpeg openssl network wifi iptables zstd lz4 7zip rar btrfs tpm gnome-keyring qemu wayland gles2 vdpau vaapi vulkan vkd3d d3d9 nvidia nvenc steamfonts trayicon systray -joystick -games -education -xinerama -firewall -networkmanager -ppp -kaccounts -webengine -kwallet -bittorrent -phonon -vlc -gtk2 -gnome -gnome-shell -gnome-online-accounts -bindist -ssp -doc -gtk-doc -handbook -spell -grub -oss -gpm"
ACCEPT_KEYWORDS="~amd64" # ~表示unstable，amd64表示架构是x86_64
ACCEPT_LICENSE="*" # 接受所有协议
EMERGE_DEFAULT_OPTS="--keep-going --with-bdeps=y --jobs=${NTHREADS} --load-average=${NTHREADS}"
L10N="en-US zh-CN en zh"
LINGUAS="en_US zh_CN en zh"
VIDEO_CARDS="nvidia intel"
ALSA_CARDS="hda-intel"
LLVM_TARGETS="X86 NVPTX"
PYTHON_TARGETS="python3_10"
PYTHON_SINGLE_TARGET="python3_10"
RUBY_TARGETS="ruby30 ruby31"
ABI_X86="64 32"
FEATURES="ccache"
CCACHE_DIR="/var/cache/ccache"
CPU_FLAGS_X86="aes avx avx2 f16c fma3 mmx mmxext pclmul popcnt rdrand sse sse2 sse3 sse4_1 sse4_2 ssse3" # 本行由app-portage/cpuid2cpuflags生成
CONFIG_PROTECT="/usr/share/sddm/scripts/Xsetup"
UNINSTALL_IGNORE="/bin /lib /lib64 /sbin /usr/sbin"
```

然后再通过如下命令设置main repo的repos.conf

```bash
mkdir -p /mnt/gentoo/etc/portage/repos.conf
cp /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
```

然后把`sync-uri`处修改一下，建议rsync的话使用广度优先搜索大学的，如下

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

之后是通过命令`emerge-webrsync`同步快照，同步完之后让我们再修改一些package的use，命令如下

```bash
echo 'app-text/ghostscript-gpl -l10n_zh-CN' > /etc/portage/package.use/ghostscript-gpl #为了不安装宋体，去掉该包的zh-CN支持
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
emerge -av networkmanager dev-vcs/git btrfs-progs neovim eselect-repository systemd-cron doas mlocate intel-microcode grub:2 # 按需安装
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
#如果去掉了passwdqc的use，不用如下配置也可以使用简单密码
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

这里的话我推荐`sys-kernel/liquorix-sources`，具体步骤如下:

```bash
eselect repository enable gentoo-zh
mkdir -p /etc/portage/package.accept_keywords; echo 'sys-kernel/liquorix-sources ~amd64' >> /etc/portage/package.accept_keywords/liquorix-sources #如果make.conf中ACCEPT_KEYWORDS="~amd64"，就不需要该步骤
emerge -av sys-kernel/liquorix-sources
eselect kernel set 1
cd /usr/src/linux
make mrproper
cp /var/db/repos/gentoo-zh/sys-kernel/liquorix-sources/config/default-config /usr/src/linux/.config #复制一份默认的配置
make modules_prepare
make menuconfig
make -jX -lX #此处X为线程数
make modules_install
make install
```

具体配置的话我就不细讲了，如果想节约时间，建议装`sys-kernel/gentoo-kernel-bin`

接着装一下GRUB

```bash
# 若使用EFI，请用如下命令
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB # --efi-directory后面接的是esp分区
# 若为BIOS，则用如下命令
grub-install --target=i386-pc /dev/sdX # /dev/sdX指的是系统盘(不是分区)

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

##### 希望大家重启之后都能如愿进入系统
