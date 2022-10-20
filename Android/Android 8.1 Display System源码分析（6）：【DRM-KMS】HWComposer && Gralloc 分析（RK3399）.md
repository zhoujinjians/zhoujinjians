---
title: Android 8.1 Display System源码分析（6）：【DRM/KMS】HWComposer && Gralloc 分析（RK3399）
cover: https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/personal.website/post.cover.pictures.00015.jpg
categories: 
  - Display
tags:
  - Android
  - Linux
  - Graphics
toc: true
abbrlink: 20200908
date: 2020-09-08 09:25:00
---


注：文章都是通过阅读 Android  && Linux 平台源码、各位前辈总结的资料、加上自己的思考分析总结出来的，其中难免有理不对的地方，欢迎大家批评指正。**文章为个人学习、研究、欣赏之用。图文内容、源码整理自互联网，如有侵权，请联系删除（◕‿◕）**，转载请注明出处（ ©Android @Linux @Rockchip版权所有），谢谢(๑乛◡乛๑) 、（ ͡° ͜ʖ ͡°）、（ಡωಡ）！！！。
（==**文章基于 Kernel-4.4**==）&&（==**文章基于 Android-8.1**==）
[【开发板 - Firefly-RK3399 7.85寸1024x768显示屏模组（MIPI接口）】](http://wiki.t-firefly.com/zh_CN/Firefly-RK3399/compile_android8.1_firmware.html#)

正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢！！！
[（1）【[RK3288][Android6.0] 调试笔记 --- display数据帧的dump）】](https://blog.csdn.net/kris_fei/article/details/75278854) 
[（2）【Hardware Composer 2.0】](https://blog.linuxplumbersconf.org/2016/ocw/system/presentations/4185/original/LPC%20HWC%202.0%20&%20drm_hwcomposer%20.pdf) 

#### （一）、DRM/KMS for Android介绍
##### （1）、Pre-DRM world

![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.drmkmsAndroid/DRM_KMS-android-pre-drm-world.png)
**Issues：**
![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.drmkmsAndroid/DRM_KMS-android-pre-drm-world-Issues.png)
##### （2）、DRM world

![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.drmkmsAndroid/DRM_KMS-android-drm-world.png)
**Objectives：**
![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.drmkmsAndroid/DRM_KMS-android-drm-world-Objectives.png)

##### （3）、Future
![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.drmkmsAndroid/DRM_KMS-android-drm-world-future.png)

#### （二）、Android drm_hwcomposer介绍
##### （1）、Android Hardware Composer 2.0
What is Android’s Hardware Composer?
● Determines the most efficient way to composite buffers with the available
hardware
○ Overlay planes
○ GPU composition
○ Blit/2D engine
● Often device specific and written by the display hardware OEM
###### Hardware Composer 1.x
![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.drmkmsAndroid/Android-hardware-composer1.x.png)
###### Key Differences between 1.x and 2.0
Increases API functions from 12 to 43
● Adds support for HDR, color transform matrix, dataspaces, etc.
● Renames prepare() / set() to validate() / present()
● Replaces speculative fences with non-speculative fences
###### Hardware Composer 1.x Sync Fences
![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.drmkmsAndroid/Android-hardware-composer1.x.Sync-Fences.png)
###### Hardware Composer 2.0 Sync Fences
![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.drmkmsAndroid/Android-hardware-composer2.0.Sync-Fences.png)
##### （2）、Android drm_hwcomposer
![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.drmkmsAndroid/DRM_KMS-android-drm-hwcomposer.png)
###### HWC1
![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.drmkmsAndroid/DRM_KMS-android-drm-hwc1.png)
###### HWC2
![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.drmkmsAndroid/DRM_KMS-android-drm-hwc2.png)
这里使用的就是HWC2。
#### （三）、Android drm_hwcomposer Code Overview
![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.drmkmsAndroid/DRM_KMS-android-drm-hwc-code-overview1.png)
![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.drmkmsAndroid/DRM_KMS-android-drm-hwc-code-overview2.png)
![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.drmkmsAndroid/DRM_KMS-android-drm-hwc-code-overview3.png)

分析hwcomposer之前，先来分析gralloc。
#### （四）、gralloc代码分析

![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.drmkmsAndroid/FireFly-Rk3399-DRM-KMS-drm_hwcomposer-gralloc.png)
#### （五）、hwcomposer代码分析
![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.drmkmsAndroid/FireFly-Rk3399-DRM-KMS-drm_hwcomposer-hwcomposer.png)

#### （六）、HWC数据帧的dump
##### （1）、设置property
>setprop sys.dump true

抓取的帧会按数字排列，还带分辨率参数。
![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.drmkmsAndroid/DRM_KMS-android-drm-hwcomposer-sys.dump.png)
##### （2）、adb pull /data/dump/
>adb pull /data/dump/

##### （3）、抓到的bin文件可以用软件7yuv打开查看，格式设定为RGBA8888


![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.drmkmsAndroid/DRM_KMS-android-drm-hwcomposer-launcher.png)

![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.drmkmsAndroid/DRM_KMS-android-drm-hwcomposer-navigationbar.png)

![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.drmkmsAndroid/DRM_KMS-android-drm-hwcomposer-statusbar.png)


![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.drmkmsAndroid/DRM_KMS-android-drm-hwcomposer-wallpaper.png)

最终合成效果图
![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.drmkmsAndroid/DRM_KMS-android-drm-hwcomposer-display-all.png)


#### （七）、参考资料(特别感谢)：

[（1）【[RK3288][Android6.0] 调试笔记 --- display数据帧的dump）】](https://blog.csdn.net/kris_fei/article/details/75278854) 
[（2）【Hardware Composer 2.0】](https://blog.linuxplumbersconf.org/2016/ocw/system/presentations/4185/original/LPC%20HWC%202.0%20&%20drm_hwcomposer%20.pdf) 
