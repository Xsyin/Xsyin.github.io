---
title: Linux Kernel 运行调试
date: 2020-02-19 17:09:12
updated: 2020-02-19 17:09:12
tags:
  linux
categories: linux
copyright: true
---

## 前置环境

```
sudo apt-get install qemu libssl-dev libncurses5-dev gcc-5-aarch64-linux-gnu build-essential gdb-multiarch
sudo update-alternatives --install /usr/bin/aarch64-linux-gnu-gcc aarch64-linux-gnu-gcc /usr/bin/aarch64-linux-gnu-gcc-5 5
```

<!-- more -->

## BusyBox

```
cd busybox-1_31_0/
export ARCH=arm
export CROSS_COMPILE=arm-linux-gnueabi-
make menuconfig
#build optios --->  static binary 
make -j4 && make install
```



## Linux Kernel(https://github.com/figozhang/runninglinuxkernel_4.0)

```
cd linux-4.14.143
#拷贝_install_arm64, 并拷贝etc
cd _install_arm64
mkdir dev mnt
cd ..
export ARCH=arm
export CROSS_COMPILE=arm-linux-gnueabi-
make vexpress_defconfig
make bzImage -j4 ARCH=arm CROSS_COMPILE=arm-linux-gnueabi-
make dtbs
```

## qemu

```
qemu-system-arm -M vexpress-a9 -smp 4 -m 200M -kernel arch/arm/boot/zImage -append "rdinit=/linuxrc console=ttyAMA0 loglevel=8" -dtb arch/arm/boot/dts/vexpress-v2p-ca9.dtb -nographic

qemu-system-arm -M vexpress-a9 -smp 4 -m 100M -kernel arch/arm/boot/zImage -dtb arch/arm/boot/dts/vexpress-v2p-ca9.dtb -nographic -append "rdinit=/linuxrc console=ttyAMA0 loglevel=8 slub_debug kmemleak=on" --fsdev local,id=kmod_dev,path=$PWD/kmodules,security_model=none -device virtio-9p-device,fsdev=kmod_dev,mount_tag=kmod_mount -s -S
```

## Vim

```
git clone https://github.com/VundleVim/Vundle.vim.git ~/.vim/bundle/Vundle.vim
#.vimrc
------------------------------------------------
" Vundle manage
set nocompatible           " be improved, required
filetype off

" set the runtime path to include Vundle and initialize
set rtp+=~/.vim/bundle/Vundle.vim
call vundle#begin()

" let Vundle manage Vundle, required
Plugin 'VundleVim/Vundle.vim'

" All of your Plugins must be added before the following line
call vundle#end()             " required
filetype plugin indent on      " required
------------------------------------------------------
sudo apt-get install ctags
$ ctags -R      "在项目目录下
$ :set tags=tags  " vim中
sudo apt install cscope
$ cscope -Rbq    "在项目目录下
$ cscope add        cscope find
```

