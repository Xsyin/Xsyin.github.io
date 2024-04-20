---
title: JUNO 移植
date: 2020-02-19 17:19:12
updated: 2020-02-19 17:19:12
tags:
 - JUNO 
 - 教程
categories: 教程
keywords: JUNO
copyright: true
---

## 1. 前置工具安装

```shell
sudo apt install acpica-tools autoconf bc bison bridge-utils build-essential curl device-tree-compiler expect flex g++-multilib gcc-multilib genext2fs gperf libc6:i386 libssl-dev libstdc++6:i386 libncurses5:i386 libxml2 libxml2-dev libxml2-utils libxml-libxml-perl make openssh-server openssh-client python python-pip uuid-dev automake android-tools-adb android-tools-fastboot cython kernelshark libfreetype6-dev libpng-dev libtool net-tools nmap openjdk-8-jdk pkg-config python-dev python-mako python-matplotlib python-nose python-numpy python-wand python-wrapt screen sshpass trace-cmd tree
```

<!-- more -->

## 2. 源码下载

```
$ git clone https://git.linaro.org/landing-teams/working/arm/arm-reference-platforms.git
$ cd arm-reference-platforms/
#修改 sync_workspace.py 中215行 'libpng-dev' 为 'libpng12-dev'
$ python3 sync_workspace.py
#之后按照提示选择
```

## 3. 编译使用

### 3.1  使用已编译的镜像

​     `python3 sync_workspace.py` 中选择 `use prebuilt configuration`

使用空SD卡，执行`sudo dd if=juno.img of=/dev/sdb bs=64M` 

启动开发板，cmd时

```
Cmd> flash
Flash> eraseall
Flash> quit
Cmd> usb_on
```

启动后，flash以u盘的形式可见，复制项目中`juno-ack-android-uboot` 文件夹下内容至u盘根目录下。

将SD卡插入开发板usb口

重启开发板，完成

### 3.2 使用源码

- 下载 uboot, uefi, linux kernel, scp, optee源码

  `python3 sync_workspace.py` 中选择 `Build from source` 

执行过程半小时或一小时左右，且最后checkout时出现 dirs变量未定义使用，由于 build-scripts文件夹下没有platform文件夹导致，但后续步骤未出现问题

build-scripts文件夹下执行 `./build-all.sh -p juno -f android all`  

拷贝recovery目录下文件至开发板flash中，并拷贝output/juno/juno-android下内容替换flash中SOFTWARE目录下同名文件。

- 下载android 源码

从http://releases.linaro.org/members/arm/android/juno/19.01/ 下载 linaro_android_build_cmds.sh； pinned-manifest.xml

执行 `./linaro_android_build_cmds.sh -m pinned-manifest.xml` 下载同时会进行编译

只编译：

```
source build/envsetup.sh
lunch juno-userdebug
(make -j4 droidcore boottarball showcommands) 2>&1 |tee build-juno.log
```

打包成juno.img：

```
$ cd android/out/target/product/juno/
$ wget https://android-git.linaro.org/android/device/linaro/juno.git/plain/pack_juno_img.sh?h=linaro-oreo -O pack_juno_img.sh
$ chmod a+x pack_juno_img.sh
$ ./pack_juno_img.sh
$ sudo dd if=juno.img of=/dev/sdb bs=64M
```

不打包：

linaro-android-media-create --mmc /dev/sdX --image_size 2000M --dev vexpress --systemimage system.img --userdataimage userdata.img --boot boot.tar.bz2



### 4. 调试

 adb 连接: 网络调试

在android shell 中： setprop service.adb.tcp.port 5555

  stop adbd 

  start adbd



