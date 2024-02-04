---
title: Ubuntu 16.04 搭建 Khadas-Edge-V-Android10编译环境
cover: https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/post.cover.pictures/bing-wallpaper-2018.04.21.jpg
categories: 
  - Display
tags:
  - Android
  - Linux
  - Graphics
toc: true
abbrlink: 20210308
date: 2021-03-08 09:25:00
---


## 1, 搭建用于编译 Android 的环境
搭建用于编译 Android 的环境，建议使用 64 位的 Ubuntu 16.04，需要安装如下软件包：

``` bash
$ sudo apt-get install bison g++-multilib git gperf libxml2-utils make python-networkx zip
$ sudo apt-get install flex curl libncurses5-dev libssl-dev zlib1g-dev gawk minicom
$ sudo apt-get install exfat-fuse exfat-utils device-tree-compiler liblz4-tool
$ sudo apt-get install openjdk-8-jdk
```

## 2, 下载安卓源码

我们的 Khadas Edge 的 Android 源代码托管在 Github 上。有许多不同的存储库。

按如下步骤下载源代码。

步骤
#### 1）创建一个空目录来保存您的工作文件：

```
$mkdir -p WORKING_DIRECTORY
$cd WORKING_DIRECTORY
```

#### 2）首先运行 repo init 下载清单存储库：

```
android 10.0:

$repo init -u https://github.com/khadas/android_manifest.git -b khadas-edge-Qt
```

#### 3）运行 repo-sync 下拉 Android 源代码：

```
$repo sync -j4
```

初始同步操作可能需要很长时间才能完成。
提示：如果命令中途失败，您可能需要重复运行上面的命令。或者您可以尝试使用此脚本：

``` bash
#!/bin/bash
repo sync -j4
while [ $? = 1 ]; do
    echo "Sync failed, repeat again:"
    repo sync -j4
done
```

如果需要，请按 ctrl-\ 退出。

#### 4）建立开发分支：

```
$ repo start <BRANCH_NAME> --all
```

## 3, 编译安卓
**android 10.0:**
#### **编译 U-boot：**

```
$ cd PATH_YOUR_PROJECT
$ cd u-boot
$ make mrproper
$ ./make.sh kedge
```

#### 编译 kernel：

```
$ cd PATH_YOUR_PROJECT
$ cd kernel
$ make ARCH=arm64 kedge_defconfig android-10.config rk3399.config
$ make ARCH=arm64 rk3399-khadas-edge-android.img -jN
```

#### 编译 android：

```
$ cd PATH_YOUR_PROJECT
$ source build/envsetup.sh
$ lunch rk3399_Android10-userdebug
$ make installclean
$ make -jN
$ ./mkimage.sh
```

注意：替换 N 为你自己电脑实际的线程数。

执行./mkimage.sh 后，在 rockdev/Image-xxx / 目录生成完整的固件包 (xxx 是具体 lunch 的产品名)

rockdev/Image-xxx/
├── MiniLoaderAll.bin
├── boot.img
├── dtbo.img
├── kernel.img
├── misc.img
├── odm.img
├── parameter.txt
├── pcba_small_misc.img
├── pcba_whole_misc.img
├── recovery.img
├── resource.img
├── system.img
├── trust.img
├── uboot.img
├── update.img
├── vbmeta.img
├── super.img
└── vendor.img

####  **打包 update.img：**

```
$ cd PATH_YOUR_PROJECT
$ source build/envsetup.sh
$ lunch rk3399_Android10-userdebug
$ ./pack_image.sh
```

You can create the following file in the root directory.
**vim ./pack_image.sh**

```
#!/bin/bash
PROJECT_PATH=`pwd`
cd RKTools/linux/Linux_Pack_Firmware/rockdev/
./mkupdate_rk3399.sh
cd $PROJECT_PATH
mv RKTools/linux/Linux_Pack_Firmware/rockdev/update.img . 
```

[编译报错解决办法：](https://www.cnblogs.com/wang-yaz/p/9395005.html)

OpenJDK 64-Bit Server VM warning: INFO: os::commit_memory(0x0000000083e80000, 3221225472， 0) failed; error='Cannot allocate memory' (errno=12)

现执行命令 free -m 查看内存是不是还有 最主要的是 看有没有交换空间 swap （这很重要）如果没有交换空间 或者交换空间比较小  要先安装交换空间 或者增大空间 

#### （1）、创建 swapfile：

root 权限下，创建 swapfile  # dd  if=/dev/zero  of=swapfile  bs=1024  count=500000  （有时会遇到 dd 命令不识别 可能是你安装过一次了 没事 先把 swapfile 删除就 ok 了）

#### （2）、将 swapfile 设置为 swap 空间

 **mkswap swapfile**

#### （3）、启用交换空间，这个操作有点类似于 mount 操作（个人理解）：

**swapon  swapfile （删除交换空间 swapoff swapfile）**

至此增加交换空间的操作结束了，可以使用 free 命令查看 swap 空间大小是否发生变化；

但是有可能你运行 sudo dd if=/dev/zero of=/swapfile bs=1M count=1024 这条命令的时候它也会给你报个错

比如：

dd: failed to open '/swapfile': 

这个时候你只需要运行 sudo swapoff -a 即可解决。然后继续创建交换分区即可。

## 4, 使能串口 log
How to enable UART port in Android 10 source

波特率：1500000

串口连接方法

![](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/Android10.Display.0/How2Connect2SerialForDebug.jpg)



