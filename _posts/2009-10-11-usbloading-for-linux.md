---
layout: post
title:  "Linux USB引导盘制作"
date:   2009-10-11 +0800
categories: Linux
tags: Linux
author: dbtan
---

* content
{:toc}

### 一、系统启动过程：

```
开机→初始化BIOS→→启动引导器bootloader→→装载内核kernel→→启动init
                    （GRUB）
```





#### · 第一步，初始化BIOS：设置启动顺序等基本输入输出系统。

MBR：Master Boot Recorder主引导分区 （512字节，0柱面、0磁头、第1个扇区）
446字节MBC：主引导代码（找可引导分区）
64字节DPT：4个16字节的主分区信息
2字节：55AA（十六进制数）表示结束。

#### · 第二步，启动引导器bootloader：

##### ⑴ grub：/boot/grub/grub.conf配置文件

```
# grub.conf generated by anaconda
#
# Note that you do not have to rerun grub after making changes to this file
# NOTICE: You do not have a /boot partition. This means that
# all kernel and initrd paths are relative to /, eg.
# root (hd0,0)
# kernel /boot/vmlinuz-version ro root=/dev/sda1
# initrd /boot/initrd-version.img
#boot=/dev/sda
default=0
timeout=5
splashimage=(hd0,0)/boot/grub/splash.xpm.gz
hiddenmenu
title Red Hat Enterprise Linux Server (2.6.18-8.el5)
root (hd0,0)
```

**注**：hd0：第一个硬盘 ， 0：第一个分区
`hd0`是由`/boot/grub/device.map`硬盘映射的。
`kernel /boot/vmlinuz-2.6.18-8.el5 ro root=LABEL=/ rhgb quiet` 安静：不显示selinux提示的错误信息等其它信息。

**注**：根分区为卷标为／的。
`initrd /boot/initrd-2.6.18-8.el5.img`

**注**：

```
[root@vm5: /boot]#ls
config-2.6.18-8.el5 initrd-2.6.18-8.el5.img System.map-2.6.18-8.el5
grub symvers-2.6.18-8.el5.gz vmlinuz-2.6.18-8.el5
[root@vm5: /boot]#mkdir initrd
[root@vm5: /boot]#ls
config-2.6.18-8.el5 initrd symvers-2.6.18-8.el5.gz vmlinuz-2.6.18-8.el5
grub initrd-2.6.18-8.el5.img System.map-2.6.18-8.el5
[root@vm5: /boot]#gunzip < initrd-2.6.18-8.el5.img > initrd/initrd.img
[root@vm5: /boot]#cd initrd
[root@vm5: /boot/initrd]#ls
initrd.img
[root@vm5: /boot/initrd]#file initrd.img
initrd.img: ASCII cpio archive (SVR4 with no CRC)
[root@vm5: /boot/initrd]#cpio -iv < initrd.img
sys
lib
lib/mptspi.ko
lib/ext3.ko 载入：ext3驱动模块
lib/mptbase.ko
lib/sd_mod.ko 载入：sd驱动模块
lib/ohci-hcd.ko 载入：other芯片驱动模块
lib/jbd.ko
lib/mptscsih.ko
lib/ehci-hcd.ko 载入：USB2.0驱动模块
lib/uhci-hcd.ko 载入：USB1.1驱动模块
lib/scsi_mod.ko 载入：SCSI驱动模块
lib/scsi_transport_spi.ko
etc
proc
init
bin
bin/nash
bin/modprobe
bin/insmod
sysroot
dev
dev/mapper
dev/tty7
dev/tty3
dev/tty2
dev/tty0
dev/ttyS0
dev/tty6
dev/ram
dev/ram1
dev/rtc
dev/zero
dev/tty11
dev/ptmx
dev/tty
dev/tty5
dev/console
dev/tty10
dev/ttyS1
dev/tty1
dev/tty8
dev/tty4
dev/null
dev/tty9
dev/ram0
dev/tty12
dev/ttyS2
dev/systty
dev/ttyS3
sbin
6761 blocks
[root@vm5: /boot/initrd]#ls
bin dev etc init initrd.img lib proc sbin sys sysroot
[root@vm5: /boot/initrd]#vim init
#!/bin/nash
mount -t proc /proc /proc
setquiet
echo Mounting proc filesystem
echo Mounting sysfs filesystem
mount -t sysfs /sys /sys
echo Creating /dev
mount -o mode=0755 -t tmpfs /dev /dev
mkdir /dev/pts
mount -t devpts -o gid=5,mode=620 /dev/pts /dev/pts
mkdir /dev/shm
mkdir /dev/mapper
echo Creating initial device nodes
mknod /dev/null c 1 3
mknod /dev/zero c 1 5
mknod /dev/systty c 4 0
mknod /dev/tty c 5 0
mknod /dev/console c 5 1
mknod /dev/ptmx c 5 2
mknod /dev/rtc c 10 135
mknod /dev/tty0 c 4 0
mknod /dev/tty1 c 4 1
mknod /dev/tty2 c 4 2
mknod /dev/tty3 c 4 3
mknod /dev/tty4 c 4 4
mknod /dev/tty5 c 4 5
mknod /dev/tty6 c 4 6
mknod /dev/tty7 c 4 7
mknod /dev/tty8 c 4 8
mknod /dev/tty9 c 4 9
mknod /dev/tty10 c 4 10
mknod /dev/tty11 c 4 11
mknod /dev/tty12 c 4 12
mknod /dev/ttyS0 c 4 64
```

##### ⑵ initrd：解决驱动问题（rd：run disk）

`initrd /boot/initrd-2.6.18-8.el5.img`

#### 第三步，装载内核kernel：

`kernel /boot/vmlinuz-2.6.18-8.el5 ro root=LABEL=/ rhgb quiet`

#### 第四步，初始化init：

`/etc/inittab配置文件 →→初始化 /etc/rc.d/`

```
#
# inittab This file describes how the INIT process should set up
# the system in a certain run-level.
#
# Author: Miquel van Smoorenburg, miquels@drinkel.nl.mugnet.org
# Modified for RHS Linux by Marc Ewing and Donnie Barnes
#
# Default runlevel. The runlevels used by RHS are:
# 0 - halt (Do NOT set initdefault to this)
# 1 - Single user mode
# 2 - Multiuser, without NFS (The same as 3, if you do not have networking)
# 3 - Full multiuser mode
# 4 - unused
# 5 - X11
# 6 - reboot (Do NOT set initdefault to this)
#
id:3:initdefault: 设置默认的运行级别为：3
# System initialization.
si::sysinit:/etc/rc.d/rc.sysinit sysinit:一定要运行完后面的脚本，再继续运行后面，有错也不停（继续运行后面程序）
l0:0:wait:/etc/rc.d/rc 0 wait:等运行完后面脚本，再继续运行会面，有错就停。
l1:1:wait:/etc/rc.d/rc 1
l2:2:wait:/etc/rc.d/rc 2
l3:3:wait:/etc/rc.d/rc 3
l4:4:wait:/etc/rc.d/rc 4
l5:5:wait:/etc/rc.d/rc 5
l6:6:wait:/etc/rc.d/rc 6
# Trap CTRL-ALT-DELETE
ca::ctrlaltdel:/sbin/shutdown -t3 -r now
# When our UPS tells us power has failed, assume we have a few minutes
# of power left. Schedule a shutdown for 2 minutes from now.
# This does, of course, assume you have powerd installed and your
# UPS connected and working correctly.
pf::powerfail:/sbin/shutdown -f -h +2 "Power Failure; System Shutting Down"
# If power was restored before the shutdown kicked in, cancel it.
pr:12345:powerokwait:/sbin/shutdown -c "Power Restored; Shutdown Cancelled"
# Run gettys in standard runlevels
1:2345:respawn:/sbin/mingetty tty1 respawn:可重生
2:2345:respawn:/sbin/mingetty tty2
3:2345:respawn:/sbin/mingetty tty3
4:2345:respawn:/sbin/mingetty tty4
5:2345:respawn:/sbin/mingetty tty5
6:2345:respawn:/sbin/mingetty tty6
# Run xdm in runlevel 5
x:5:respawn:/etc/X11/prefdm –nodaemon
/etc/inittab配置文件 →→初始化： /etc/rc.d/rc.sysinit 键盘、鼠标驱动
```

#### 第五步 `/etc/rc.d/rc` 运行级别

`/etc/rc.d/rc.local`

系统环境配置

> **说明**：
>
> ⑴ initrd先于kernel，如果kernel中有硬盘等驱动就不用initrd了。RHEL5的kernel中没有硬盘驱动，所以得先initrd再装载kernel。
>
> ⑵ 当前模块配置文件在：/boot/config-2.6.18-8.el5中，`/usr/local/src/`uname -r`/.config`文件
>
> ⑶ /boot/grub中，menu.lst -> ./grub.conf
>
> ⑷ 在第四步中，
> /etc/inittab配置文件 →→初始化 /etc/rc.d/rc.sysinit 键盘、鼠标驱动
> /etc/rc.d/rc 运行级别
> /etc/rc.d/rc.local 系统环境配置

### 二、U盘引导制作方法、步骤：

#### ⑴ 对U盘分区、格式化为EXT3文件系统并且加可引导。

```
[root@vm5: ~]#fdisk /dev/sdb
按a，加可引导。
[root@vm5: ~]#partprobe
[root@vm5: ~]#mke2fs -j /dev/sdb1
```

#### ⑵ 安装目录树。

```
[root@vm5: /mnt/Server]#rpm -ivh --nodeps --force --root=/mnt/ filesystem-2.4.0-1.i386.rpm
```

#### ⑶ 安装grub。

```
[root@vm5: /mnt/Server]#rpm -ivh --nodeps --force --root=/mnt/ grub-0.97-13.i386.rpm
```

#### ⑶ 拷贝应用程序到U盘。注：不要覆盖刚刚生成的文件或目录。

```
[root@vm5: ~]#cp -rf /bin/* /mnt/bin/
[root@vm5: ~]#cp -rf /sbin/* /mnt/sbin/
[root@vm5: ~]#cp -rf /usr/bin/* /mnt/usr/bin/
[root@vm5: ~]#cp -rf /usr/sbin/* /mnt/usr/sbin/
```

#### ⑷ 拷贝库文件到U盘。注：不要覆盖刚刚生成的文件或目录。

```
[root@vm5: ~]#cp -rf /lib/* /mnt/lib/
[root@vm5: ~]#cp -rf /usr/lib/* /mnt/usr/lib/ ←如果文件太大，可不拷
```

#### ⑸ 拷贝`/boot/*`到U盘之后，在U盘中修改`/mnt/boot/grub/grub.conf`配置文件、修改`/mnt/boot/grub/device.map`硬盘映射文件等。

```
[root@vm5: ~]#cp -rf /boot/* /mnt/boot/
[root@vm5: /mnt]#vim boot/grub/grub.conf
改：default=0
timeout=5
hiddenmenu
title Red Hat Enterprise Linux Server USB LINUX (2.6.18-8.el5)
root (hd0,0)
kernel /boot/vmlinuz-2.6.18-8.el5 ro root=/dev/sdb1 init=/bin/bash
注：不能有多于的“空格”，否则，无法引导成功！
initrd /boot/initrd_usb.img
[root@vm5: /mnt]#vim boot/grub/device.map
写：(hd0) /dev/sdb
```

#### ⑹ 创建设备文件。 块 主 从

```
[root@vm5: ~]#mknod /mnt/dev/sdb b 8 16
[root@vm5: ~]#mknod /mnt/dev/sdb1 b 8 17
说明：主设备号：用同一个驱动。
从设备号：记录分区号。
[root@vm5: /mnt/Server]#rpm -ivh kernel-doc-2.6.18-8.el5.noarch.rpm安装内核文档，查看/usr/share/doc/kernel-doc-2.6.18/Documentation/devices.txt文档来学习主设备号、从设备号等说明。
```

#### ⑺ 加载驱动到initrd。 注：加载顺序不能错！

```
[root@vm5: ~]#mkinitrd --with=sd_mod --with=scsi_mod --with=uhci-mod --with=ehci-hcd --with=usb-storage /mnt/boot/ initrd_usb.img `uname -r`
说明：initrd_usb.img为U盘中/mnt/boot/grub/grub.conf配置文件中initrd /boot/initrd_usb.img 。
```

#### ⑻ 拷贝/etc/fstab和/etc/mtab，并加以修改。

```
[root@vm5: ~]#cp /etc/fstab /mnt/etc/fstab
只写：/dev/sdb1 / ext3 defaults 1 1
[root@vm5: ~]#cp /etc/mtab /mnt/etc/mtab
只写：/dev/sdb1 / ext3 rw 0 0
```

#### ⑼ 切换U盘/mnt/为／根分区。
```
[root@vm5: ~]#chroot /mnt
```

#### ⑽ 重装sdb，修复MBR。
```
sh-3.1#grub-install /dev/sdb
sh-3.1#grub-install --recheck /dev/sdb
注：检测（也可不用）
至此，U盘引导盘制作完毕！用sync命令同步一下磁盘，重启系统。
改BIOS为USB-HDD为第一启动，即可用U盘引导Linux系统了！！！
```

-- The End --