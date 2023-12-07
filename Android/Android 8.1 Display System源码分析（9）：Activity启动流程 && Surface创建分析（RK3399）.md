---
title: Android 8.1 Display System源码分析（9）：Activity启动流程 && Surface创建分析（RK3399）
cover: https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/post.cover.pictures/bing-wallpaper-2018.04.18.jpg
categories: 
  - Display
tags:
  - Android
  - Linux
  - Graphics
toc: true
abbrlink: 20201208
date: 2020-12-08 09:25:00
---


注：文章都是通过阅读 Android  && Linux 平台源码、各位前辈总结的资料、加上自己的思考分析总结出来的，其中难免有理不对的地方，欢迎大家批评指正。**文章为个人学习、研究、欣赏之用。图文内容、源码整理自互联网，如有侵权，请联系删除（◕‿◕）**，转载请注明出处（ ©Android @Linux @Rockchip版权所有），谢谢(๑乛◡乛๑) 、（ ͡° ͜ʖ ͡°）、（ಡωಡ）！！！。
（==**文章基于 Kernel-4.4**==）&&（==**文章基于 Android-8.1**==）
[【开发板 - Firefly-RK3399 7.85寸1024x768显示屏模组（MIPI接口）】](http://wiki.t-firefly.com/zh_CN/Firefly-RK3399/compile_android8.1_firmware.html#)

正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢！！！

#### （一）、参考：
由于Android Framework之前已经分析过了，请参考：

[【Android 8.1 Display System源码分析（9）：Activity启动流程 && Surface创建分析（RK3399）】](https://zhoujinjian.com/posts/20190725/)

