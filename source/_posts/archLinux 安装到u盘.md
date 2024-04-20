---
title: archLinux 安装到u盘
date: 2020-02-19 17:09:12
updated: 2020-02-19 17:09:12
tags:
 - arch linux
 - 教程
categories: 教程
keywords: archlinux
copyright: true
---

> Lenovo G510    ubuntu 18.04   8G u盘
>
> 启动方式：legacy bios  非 UEFI

## virtualBox

1. 下载[archlinux镜像](https://mirrors.tuna.tsinghua.edu.cn/archlinux/iso/) , virtualBox中新建archlinux虚拟机

PS：virtualBox u盘识别需要

```
usermod -a -G vboxusers $USER
# 重启系统
```

​    在Ubuntu Host中 ssh连接 archlinux虚拟机：

<!-- more -->

```
# 在archlinux中
systemctl start sshd
passwd   #设置密码

# virtualBox 网络配置中设置端口转发规则
主机   1111       客户机    22

# 在ubuntu 中
ssh -l root 127.0.0.1 -p 1111
```

## U 盘

1. **分区**      在archlinux 中

```
lsblk   
# 查看u盘盘符，假设为sdb
fdisk /dev/sdb
主要选项：o
 boot分区：n
       +128M
 根分区：n
 其余默认
 写入修改：w
 打印: p
```

2. **格式化** 

```
mkfs.fat -F32 /dev/sdb1
mkfs.ext4 /dev/sdb2
```

3.  **挂载**

```
mount /dev/sdb2 /mnt
```

## ArchLinux

1. **安装**

```
# 修改软件源为国内源
cd /etc/pacman.d
cp mirrorlist mirrorlist.bk
cat mirrorlist.bk | grep China -A 1 | grep -v '-' > mirrorlist

pacstrap /mnt base base-devel linux linux-firmware dhcpcd
genfstab -L /mnt >> /mnt/etc/fstab
```

2.  **配置**

```
#进入u盘archlinux
arch-chroot /mnt
# 密码
passwd
# 时间
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
hwclock --systohc
# 基础软件
pacman -S vim dialog wpa_supplicant ntfs-3g networkmanager
# 语言
vim /etc/locale.gen  # 取消en_US.UTF-8 zh_CN.UTF-8的注释
locale-gen
vim /etc/locale.conf  # 添加 LANG=en_US.UTF-8
# 主机名
echo 'myhostname' > /etc/hostname
# hosts
vim /etc/hosts
# 添加内容 
			127.0.0.1	localhost
			::1		localhost
			127.0.1.1	myhostname.localdomain	myhostname
```

3. **引导**

```
pacman -S intel-ucode os-prober ntfs-3g grub
grub-install --target=i386-pc /dev/sdb
grub-mkconfig -o /boot/grub/grub.cfg
```

4.  **收工**

```
exit
umount /mnt
```

## 启动

重启，Fn + F12选择启动介质  或 Fn+F2进入Bios选择leagcy First, 并调整u盘为第一个 即可进入u盘的archlinux系统，经测试 编辑的文件会保留， 重启后再进入文件仍在。

8G u盘过小，无法安装图形界面。可选用32G 等



参考：

[viseator大神的博客](https://www.viseator.com/2017/05/17/arch_install/)