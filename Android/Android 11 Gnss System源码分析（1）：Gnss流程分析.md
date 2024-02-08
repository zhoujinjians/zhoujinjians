---
title: Android 11 Gnss System源码分析（1）：Gnss流程分析
cover: https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/post.cover.pictures/bing-wallpaper-2018.04.51.jpg
categories: 
 - Display
tags:
 - Display
toc: true
abbrlink: 20230516
date: 2023-05-16 16:16:16
---
注：文章都是通过阅读各位前辈总结的资料、Android 11 Rockchip平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除（◕‿◕），转载请注明出处（©Rockchip ©Android @Linux 版权所有），谢谢。

（==**文章基于 Android 11.0**==）

[【zhoujinjian.com博客原图链接】](https://github.com/zjjzhoujinjian) 

[【开发板】](https://wiki.radxa.com/Rockpi4)

[【开发板 源码链接】](https://wiki.radxa.com/Rockpi4/rockpi-android11)



正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢，由于不喜欢图片水印，去除了水印，敬请谅解！！！

[zhoujinjian.com](http://zhoujinjian.com) 



--------------------------------------------------------------------------------

==源码（部分）==：

**xxx**

==源码（部分）==：

--------------------------------------------------------------------------------

## （一）、Gnss  Hal  Service初始化

### （1）、service.rc启动脚本

```c++
Z:\newcc\lagvm\lagvm\LINUX\android\vendor\unicore_gnss\service2_0\android.hardware.gnss@2.0-service-unicore.rc
service gnss_service  /vendor/bin/hw/android.hardware.gnss@2.0-service-unicore
    class hal
    user root
    group gps system root
    seclabel u:r:hal_gnss_unicore:s0


```

### （2）、new Gnss()

> Z:\newcc\lagvm\lagvm\LINUX\android\vendor\unicore_gnss\service2_0\service.cpp

![image-20221102171931273](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102171931273.png)

![image-20221102172644988](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102172644988.png)

## （二）、LocationManagerService初始化（system_server）

### （1）、initializeGnss

![image-20221102173007024](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102173007024.png)

![image-20221101151325709](./Android 11 Sensor System源码分析（1）：Sensor流程分析.assets/image-20221101151325709.png)

### （2）、new GnssManagerService()

![image-20221102173108154](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102173108154.png)

### （3）、GnssManagerService.isGnssSupported()

![image-20221102173411950](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102173411950.png)

![image-20221102173452169](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102173452169.png)

### （4）、android_location_GnssLocationProvider_set_gps_service_handle()

![image-20221102173620183](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102173620183.png)

### （5）、GnssLocationProvider()初始化

![image-20221102180733802](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102180733802.png)

![image-20221102180845843](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102180845843.png)

![image-20221102180907027](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102180907027.png)

### （6）、android_location_GnssLocationProvider_init()初始化

![image-20221102181204334](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102181204334.png)

```c++
01-01 08:00:09.607   452   452 D Gnss    :   #00 pc 0000000000014ca0  /vendor/bin/hw/android.hardware.gnss@2.0-service-unicore (android::hardware::gnss::V2_0::implementation::Gnss::setCallback_2_0(android::sp<android::hardware::gnss::V2_0::IGnssCallback> const&)+120)
01-01 08:00:09.607   452   452 D Gnss    :   #01 pc 000000000005f508  /apex/com.android.vndk.v30/lib64/android.hardware.gnss@2.0.so (android::hardware::gnss::V2_0::BnHwGnss::_hidl_setCallback_2_0(android::hidl::base::V1_0::BnHwBase*, android::hardware::Parcel const&, android::hardware::Parcel*, std::__1::function<void (android::hardware::Parcel&)>)+316)
01-01 08:00:09.607   452   452 D Gnss    :   #02 pc 00000000000623ac  /apex/com.android.vndk.v30/lib64/android.hardware.gnss@2.0.so (android::hardware::gnss::V2_0::BnHwGnss::onTransact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+3260)
01-01 08:00:09.607   452   452 D Gnss    :   #03 pc 0000000000093fac  /apex/com.android.vndk.v30/lib64/libhidlbase.so (android::hardware::BHwBinder::transact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+68)
01-01 08:00:09.607   452   452 D Gnss    :   #04 pc 0000000000097f44  /apex/com.android.vndk.v30/lib64/libhidlbase.so (android::hardware::IPCThreadState::getAndExecuteCommand()+1076)
01-01 08:00:09.607   452   452 D Gnss    :   #05 pc 000000000009918c  /apex/com.android.vndk.v30/lib64/libhidlbase.so (android::hardware::IPCThreadState::joinThreadPool(bool)+96)
01-01 08:00:09.608   452   452 D Gnss    :   #06 pc 000000000000f88c  /vendor/bin/hw/android.hardware.gnss@2.0-service-unicore (main+184)
01-01 08:00:09.608   452   452 D Gnss    :   #07 pc 00000000000499fc  /apex/com.android.runtime/lib64/bionic/libc.so (__libc_init+108)
    
    
Stack Trace:
  RELADDR           FUNCTION                                                                                                                                                                                                                       FILE:LINE
  0000000000014ca0  android::hardware::gnss::V2_0::implementation::Gnss::setCallback_2_0(android::sp<android::hardware::gnss::V2_0::IGnssCallback> const&)+120                                                                                     vendor/unicore_gnss/service2_0/Gnss.cpp:739
  000000000005f508  android::hardware::gnss::V2_0::BnHwGnss::_hidl_setCallback_2_0(android::hidl::base::V1_0::BnHwBase*, android::hardware::Parcel const&, android::hardware::Parcel*, std::__1::function<void (android::hardware::Parcel&)>)+316  out/soong/.intermediates/hardware/interfaces/gnss/2.0/android.hardware.gnss@2.0_genc++/gen/android/hardware/gnss/2.0/GnssAll.cpp:1169
  00000000000623ac  android::hardware::gnss::V2_0::BnHwGnss::onTransact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+3260                      out/soong/.intermediates/hardware/interfaces/gnss/2.0/android.hardware.gnss@2.0_genc++/gen/android/hardware/gnss/2.0/GnssAll.cpp:1898
  0000000000093fac  android::hardware::BHwBinder::transact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+68                                     system/libhwbinder/Binder.cpp:116
  v-------------->  android::hardware::IPCThreadState::executeCommand(int)                                                                                                                                                                         system/libhwbinder/IPCThreadState.cpp:1206
  0000000000097f44  android::hardware::IPCThreadState::getAndExecuteCommand()+1076                                                                                                                                                                 system/libhwbinder/IPCThreadState.cpp:459
  000000000009918c  android::hardware::IPCThreadState::joinThreadPool(bool)+96                                                                                                                                                                     system/libhwbinder/IPCThreadState.cpp:559
  000000000000f88c  main+184                                                                                                                                                                                                                       vendor/unicore_gnss/service2_0/service.cpp:37
  00000000000499fc  __libc_init+108                                                                                                                                                                                                                bionic/libc/bionic/libc_init_dynamic.cpp:151

```

### （7）、BnHwGnss::_hidl_setCallback_2_0

> Z:\newcc\lagvm\lagvm\LINUX\android\out\soong\.intermediates\hardware\interfaces\gnss\2.0\android.hardware.gnss@2.0_genc++\gen\android\hardware\gnss\2.0\GnssAll.cpp

![image-20221102181850722](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102181850722.png)

### （8）、Gnss::setCallback_2_0

> Z:\newcc\lagvm\lagvm\LINUX\android\vendor\unicore_gnss\service2_0\Gnss.cpp

![image-20221102181946310](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102181946310.png)

### （9）、unicore_gps_init()

> Z:\newcc\lagvm\lagvm\LINUX\android\vendor\unicore_gnss\libgps\unicore_gps.c

![image-20221102182254102](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102182254102.png)

> Z:\newcc\lagvm\lagvm\LINUX\android\vendor\unicore_gnss\libgps\unicore_gps.c

初始化NMEA读取线程gps_nmea_thread，location && NMEA上报线程gps_timer_thread。

此时Gps还未Start，空转睡眠。

![image-20221102190702854](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102190702854.png)

## （三）、App（Location Client）注册监听

### （1）、LocationManager.requestLocationUpdates()

![image-20221102174739107](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102174739107.png)

![image-20221102175145022](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102175145022.png)

![image-20221102175251005](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102175251005.png)

### （2）、new Receiver(后面分发给App需要)

![image-20221102195241707](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102195241707.png)

![image-20221102195309422](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102195309422.png)

### （3）、LocationManagerService.requestLocationUpdates()

![image-20221102175419557](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102175419557.png)


### （4）、LocationManagerService.requestLocationUpdatesLocked()

![image-20221102175513754](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102175513754.png)

### （5）、LocationManagerService.applyRequirementsLocked()

![image-20221102175700414](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102175700414.png)

### （6）、GnssLocationProvider.updateRequirements()

![image-20221102175750233](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102175750233.png)

### （7）、GnssLocationProvider.startNavigating()

![image-20221102175856874](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102175856874.png)

### （8）、GnssLocationProvider.native_start()

![image-20221102175938886](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102175938886.png)

### （9）、android_location_GnssLocationProvider_start()

![image-20221102180133378](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102180133378.png)

```c++
11-02 17:10:33.464   452   452 D unicore_gps::unicore_gps_start: #00 pc 0000000000047054  /vendor/lib64/libgps.so (dumping_callstack+64)
11-02 17:10:33.464   452   452 D unicore_gps::unicore_gps_start: #01 pc 000000000004e87c  /vendor/lib64/libgps.so (unicore_gps_start+72)
11-02 17:10:33.464   452   452 D unicore_gps::unicore_gps_start: #02 pc 00000000000139dc  /vendor/bin/hw/android.hardware.gnss@2.0-service-unicore (android::hardware::gnss::V2_0::implementation::Gnss::start()+152)
11-02 17:10:33.464   452   452 D unicore_gps::unicore_gps_start: #03 pc 000000000007fa10  /apex/com.android.vndk.v30/lib64/android.hardware.gnss@1.0.so (android::hardware::gnss::V1_0::BnHwGnss::_hidl_start(android::hidl::base::V1_0::BnHwBase*, android::hardware::Parcel const&, android::hardware::Parcel*, std::__1::function<void (android::hardware::Parcel&)>)+148)
11-02 17:10:33.464   452   452 D unicore_gps::unicore_gps_start: #04 pc 0000000000061e6c  /apex/com.android.vndk.v30/lib64/android.hardware.gnss@2.0.so (android::hardware::gnss::V2_0::BnHwGnss::onTransact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+1916)
11-02 17:10:33.464   452   452 D unicore_gps::unicore_gps_start: #05 pc 0000000000093fac  /apex/com.android.vndk.v30/lib64/libhidlbase.so (android::hardware::BHwBinder::transact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+68)
11-02 17:10:33.464   452   452 D unicore_gps::unicore_gps_start: #06 pc 0000000000097f44  /apex/com.android.vndk.v30/lib64/libhidlbase.so (android::hardware::IPCThreadState::getAndExecuteCommand()+1076)
11-02 17:10:33.464   452   452 D unicore_gps::unicore_gps_start: #07 pc 000000000009918c  /apex/com.android.vndk.v30/lib64/libhidlbase.so (android::hardware::IPCThreadState::joinThreadPool(bool)+96)
11-02 17:10:33.464   452   452 D unicore_gps::unicore_gps_start: #08 pc 000000000000f88c  /vendor/bin/hw/android.hardware.gnss@2.0-service-unicore (main+184)
11-02 17:10:33.464   452   452 D unicore_gps::unicore_gps_start: #09 pc 00000000000499fc  /apex/com.android.runtime/lib64/bionic/libc.so (__libc_init+108)

    Stack Trace:
  RELADDR           FUNCTION                                                                                                                                                                                                             FILE:LINE
  0000000000047054  dumping_callstack+64                                                                                                                                                                                                 vendor/unicore_gnss/libgps/callstack.cpp:6
  000000000004e87c  unicore_gps_start+72                                                                                                                                                                                                 vendor/unicore_gnss/libgps/unicore_gps.c:2666
  00000000000139dc  android::hardware::gnss::V2_0::implementation::Gnss::start()+152                                                                                                                                                     vendor/unicore_gnss/service2_0/Gnss.cpp:449
  000000000007fa10  android::hardware::gnss::V1_0::BnHwGnss::_hidl_start(android::hidl::base::V1_0::BnHwBase*, android::hardware::Parcel const&, android::hardware::Parcel*, std::__1::function<void (android::hardware::Parcel&)>)+148  out/soong/.intermediates/hardware/interfaces/gnss/1.0/android.hardware.gnss@1.0_genc++/gen/android/hardware/gnss/1.0/GnssAll.cpp:1610
  0000000000061e6c  android::hardware::gnss::V2_0::BnHwGnss::onTransact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+1916            out/soong/.intermediates/hardware/interfaces/gnss/2.0/android.hardware.gnss@2.0_genc++/gen/android/hardware/gnss/2.0/GnssAll.cpp:1766
  0000000000093fac  android::hardware::BHwBinder::transact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+68                           system/libhwbinder/Binder.cpp:116
  v-------------->  android::hardware::IPCThreadState::executeCommand(int)                                                                                                                                                               system/libhwbinder/IPCThreadState.cpp:1206
  0000000000097f44  android::hardware::IPCThreadState::getAndExecuteCommand()+1076                                                                                                                                                       system/libhwbinder/IPCThreadState.cpp:459
  000000000009918c  android::hardware::IPCThreadState::joinThreadPool(bool)+96                                                                                                                                                           system/libhwbinder/IPCThreadState.cpp:559
  000000000000f88c  main+184                                                                                                                                                                                                             vendor/unicore_gnss/service2_0/service.cpp:37
  00000000000499fc  __libc_init+108                                                                                                                                                                                                      bionic/libc/bionic/libc_init_dynamic.cpp:151


```

### （10）、_hidl_start(V2_0)

> Z:\newcc\lagvm\lagvm\LINUX\android\out\soong\.intermediates\hardware\interfaces\gnss\2.0\android.hardware.gnss@2.0_genc++\gen\android\hardware\gnss\2.0\GnssAll.cpp

![image-20221102182758178](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102182758178.png)

### （11）、_hidl_start(V1_0)

> Z:\newcc\lagvm\lagvm\LINUX\android\out\soong\.intermediates\hardware\interfaces\gnss\1.0\android.hardware.gnss@1.0_genc++\gen\android\hardware\gnss\1.0\GnssAll.cpp

![image-20221102183003446](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102183003446.png)

### （12）、unicore_gps_start

> Z:\newcc\lagvm\lagvm\LINUX\android\vendor\unicore_gnss\libgps\unicore_gps.c

![image-20221102183109723](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102183109723.png)

## （四）、、Gnss  Hal  Service运转（读取NMEA，解析并上报Location）

### （1）、gps_state_start

> Z:\newcc\lagvm\lagvm\LINUX\android\vendor\unicore_gnss\libgps\unicore_gps.c

![image-20221102190432773](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102190432773.png)

### （2）、gps_wakeup()

> Z:\newcc\lagvm\lagvm\LINUX\android\vendor\unicore_gnss\libgps\unicore_gps.c

![image-20221102190957507](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102190957507.png)

### （3）、gps_opentty()

> Z:\newcc\lagvm\lagvm\LINUX\android\vendor\unicore_gnss\libgps\unicore_gps.c

![image-20221102191132758](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102191132758.png)

### （4）、gps_nmea_thread()读取NMEA并且解析

![image-20221102191306070](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102191306070.png)

### （5）、nmea_reader_parse()

![image-20221102191456206](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102191456206.png)

会解析读取到的NMEA，由于在室内，定位不成功，我手动设置了一组NMEA。

![image-20221102195515769](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102195515769.png)

```c++
定位：江西南昌学院（南昌校区）附件
$GPGSA,A,2,01,02,03,06,09,14,17,19,22,28,30,,1.0,0.7,0.7*3F\n\
$GPGSV,4,1,16,01,09,049,30,02,15,253,36,03,35,052,43,06,45,268,35,1*6F\n\
$GPGSV,4,2,16,09,05,136,24,14,64,172,34,17,57,012,47,22,11,040,31,1*6D\n\
$GPGSV,4,3,16,28,79,234,40,04,04,104,,19,45,330,,24,02,299,,1*6B\n\
$GPGSV,4,4,16,30,06,194,,39,,,34,41,,,39,50,,,38,1*56\n\
$GPGGA,055957.00,2846.901314,N,11551.149287,E,1,11,0.7,55.4,M,-5.0,M,,*4A\n\
$GPVTG,76.0,T,79.6,M,24.9,N,46.1,K,A*26\n\
$GPRMC,055957.00,A,2846.901314,N,11551.149287,E,24.9,76.0,020621,3.5,W,A*25\n\
$BDGSA,A,2,01,02,03,07,08,10,13,14,,,,,1.0,0.7,0.7,4*32\n\
$BDGSV,3,1,11,01,43,130,38,02,40,230,37,03,55,189,48,04,31,115,36,0,4*6B\n\
$BDGSV,3,2,11,05,18,251,42,07,67,154,44,08293,39,10,77,281,39,0,4*6D";
```

![image-20221102191858722](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102191858722.png)

### （6）、gps_timer_thread()上报解析好了的Location和卫星数据

![image-20221102192405548](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102192405548.png)

```c++
11-02 17:10:33.851   452  2275 D Gnss    :   #00 pc 0000000000012908  /vendor/bin/hw/android.hardware.gnss@2.0-service-unicore (android::hardware::gnss::V2_0::implementation::Gnss::locationCb(GpsLocation*)+116)
11-02 17:10:33.851   452  2275 D Gnss    :   #01 pc 000000000004aba8  /vendor/lib64/libgps.so (gps_timer_thread+652)
11-02 17:10:33.852   452  2275 D Gnss    :   #02 pc 0000000000011890  /vendor/bin/hw/android.hardware.gnss@2.0-service-unicore (threadFunc(void*)+12)
11-02 17:10:33.852   452  2275 D Gnss    :   #03 pc 00000000000afecc  /apex/com.android.runtime/lib64/bionic/libc.so (__pthread_start(void*)+64)
11-02 17:10:33.852   452  2275 D Gnss    :   #04 pc 0000000000050408  /apex/com.android.runtime/lib64/bionic/libc.so (__start_thread+64)
    
    Stack Trace:
  RELADDR           FUNCTION                                                                           FILE:LINE
  0000000000012908  android::hardware::gnss::V2_0::implementation::Gnss::locationCb(GpsLocation*)+116  vendor/unicore_gnss/service2_0/Gnss.cpp:133
  000000000004aba8  gps_timer_thread+652                                                               vendor/unicore_gnss/libgps/unicore_gps.c:2349
  0000000000011890  threadFunc(void*)+12                                                               vendor/unicore_gnss/service2_0/ThreadCreationWrapper.cpp:21
  00000000000afecc  __pthread_start(void*)+64                                                          bionic/libc/bionic/pthread_create.cpp:347
  0000000000050408  __start_thread+64                                                                  bionic/libc/bionic/clone.cpp:53

```

![image-20221102193325531](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102193325531.png)

## （五）、System_Server（Location）接收 Gnss Hal Service数据并分发

### （1）、GnssCallback::gnssLocationCb_2_0

> Z:\newcc\lagvm\lagvm\LINUX\android\frameworks\base\services\core\jni\com_android_server_location_GnssLocationProvider.cpp

![image-20221102194155774](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102194155774.png)

![image-20221102194237721](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102194237721.png)

### （2）、GnssLocationProvider.reportLocation()

![image-20221102194405639](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102194405639.png)

![image-20221102194448093](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102194448093.png)

### （3）、LocationManagerService.handleLocationChangedLocked()

### ![image-20221102194648736](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102194648736.png)

![image-20221102194725717](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102194725717.png)

### （4）、Receiver.callLocationChangedLocked(location)

![image-20221102195623692](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102195623692.png)

### （5）、ILocationListener.onLocationChanged(new Location(location))

![image-20221102195711445](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102195711445.png)

### （6）、截图撒花

![image-20221102200143027](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Gnss_System/Android11_Gnss01/image-20221102200143027.png)

## （六）、参考资料(特别感谢)：

### [（1）xxx](https://zhoujinjian.com/)

```c++
diff --git a/libgps/Android.mk b/libgps/Android.mk
index 0a96c4b..56bc3d6 100755
--- a/libgps/Android.mk
+++ b/libgps/Android.mk
@@ -8,9 +8,9 @@ LOCAL_C_INCLUDES := $(LOCAL_PATH)/halheader \
 #LOCAL_MODULE_RELATIVE_PATH := hw
 LOCAL_VENDOR_MODULE:= true
 
-LOCAL_SHARED_LIBRARIES := liblog libcutils libhardware libc libutils
+LOCAL_SHARED_LIBRARIES := liblog libcutils libhardware libc libutils libutilscallstack 
 
-LOCAL_SRC_FILES := unicore_gps.c unicore_geoid.c unicore_agnss.c cJSON.c  unicore_fw_update.c  unicore_fw.c
+LOCAL_SRC_FILES := unicore_gps.c unicore_geoid.c unicore_agnss.c cJSON.c  unicore_fw_update.c  unicore_fw.c  callstack.cpp
 LOCAL_C_FLAGS += -Wno-unused-parameter -D__NO_XPRINTF__
 
 
diff --git a/libgps/unicore_config.c b/libgps/unicore_config.c
index af919f6..fd5bbcc 100755
--- a/libgps/unicore_config.c
+++ b/libgps/unicore_config.c
@@ -23,11 +23,8 @@
 
 #define  DFR(...)   ALOGD(__VA_ARGS__)
 
-#ifdef GPS_DEBUG
-#  define  D(...)   ALOGD(__VA_ARGS__)
-#else
-#  define  D(...)   ((void)0)
-#endif
+#define  D(...)   ALOGD(__VA_ARGS__)
+
 
 #define NR_LOGS	10
 #define LOG_FILESIZE	(100*1024*1024)
diff --git a/libgps/unicore_config.h b/libgps/unicore_config.h
index bd1425d..e0344a2 100755
--- a/libgps/unicore_config.h
+++ b/libgps/unicore_config.h
@@ -35,9 +35,9 @@
 #define GPS_DEVICE	"/dev/ttyHS2"
 #endif
 // desired working baud
-static int current_bps = 115200;
+static int current_bps = 460800;
 // factory default baud
-static int default_bps = 9600;
+static int default_bps = 460800;
 
 #define UDP_PORT	5744
 #define TCP_PORT	5744
diff --git a/libgps/unicore_gps.c b/libgps/unicore_gps.c
index d0b6cb8..0165ec6 100755
--- a/libgps/unicore_gps.c
+++ b/libgps/unicore_gps.c
@@ -51,9 +51,32 @@
 #include <hardware/hardware.h>
 #include "unicore_agnss.h"
 #include "unicore_config.h"
-
-
-
+#include "callstack.h"
+
+//#include <utils/CallStack.h>
+
+//c not work
+//#define dumping_callstack(...)                           \
+//    do {                                                 \
+//        ALOGD(__VA_ARGS__);                              \
+//        android::CallStack callstack;                    \
+//        callstack.update();                              \
+//        callstack.log(LOG_TAG, ANDROID_LOG_DEBUG, "  "); \
+//    }while (false)
+
+
+//zhoujinjian add for test
+static char* testNmeaError = "$GPGSA,A,2,01,02,03,06,09,14,17,19,22,28,30,,1.0,0.7,0.7*3F\n\
+$GPGSV,4,1,16,01,09,049,30,02,15,253,36,03,35,052,43,06,45,268,35,1*6F\n\
+$GPGSV,4,2,16,09,05,136,24,14,64,172,34,17,57,012,47,22,11,040,31,1*6D\n\
+$GPGSV,4,3,16,28,79,234,40,04,04,104,,19,45,330,,24,02,299,,1*6B\n\
+$GPGSV,4,4,16,30,06,194,,39,,,34,41,,,39,50,,,38,1*56\n\
+$GPGGA,055957.00,2846.901314,N,11551.149287,E,1,11,0.7,55.4,M,-5.0,M,,*4A\n\
+$GPVTG,76.0,T,79.6,M,24.9,N,46.1,K,A*26\n\
+$GPRMC,055957.00,A,2846.901314,N,11551.149287,E,24.9,76.0,020621,3.5,W,A*25\n\
+$BDGSA,A,2,01,02,03,07,08,10,13,14,,,,,1.0,0.7,0.7,4*32\n\
+$BDGSV,3,1,11,01,43,130,38,02,40,230,37,03,55,189,48,04,31,115,36,0,4*6B\n\
+$BDGSV,3,2,11,05,18,251,42,07,67,154,44,08293,39,10,77,281,39,0,4*6D";
 
 static struct serial_icounter_struct icount, icount0;
 time_t icount_seconds;
@@ -213,7 +236,7 @@ static int bGetFormalNMEA = 1;
 static int started    = 0;
 static int continue_thread = 1;
 static int bOrionShutdown = 0;
-static int agnss_thread = 0;
+static int agnss_thread = 1;
 
 static char isNmeaThreadExited = 0;
 static char isTimerThreadExited = 0;
@@ -2101,8 +2124,13 @@ static void gps_nmea_thread( void*  arg )
 #endif
 					gps_log_write(buf, ret);
 					tcp_write(tcp_fd, buf, ret);
-					for (nn = 0; nn < ret; nn++)
-						nmea_reader_addc(reader, buf[nn]);
+					//for (nn = 0; nn < ret; nn++)
+					//	nmea_reader_addc(reader, buf[nn]);
+                    //zhoujinjian add for debug;
+                    int realsize = strlen(testNmeaError);
+                    for (nn = 0; nn < realsize; nn++) {
+					    nmea_reader_addc(reader, testNmeaError[nn]);
+					}
 				} else {
 					DFR("Error on NMEA read :%s",strerror(errno));
 					if(errno != EAGAIN && errno != EINTR) {
@@ -2420,15 +2448,14 @@ int gps_opentty(GpsState *state)
 
 	if(strlen(prop) <= 0)
 	{
-		state->fd  = -1;
-		return state->fd;
+		//state->fd  = -1;
+		//return state->fd;
 	}
-
 	if(state->fd != -1)
 		gps_closetty(state);
 
 	do {
-		state->fd = open( prop, O_RDWR | O_NOCTTY | O_NONBLOCK);
+		state->fd = open( GPS_DEVICE, O_RDWR | O_NOCTTY | O_NONBLOCK); /*/dev/ttyHS2*/
 	} while (state->fd < 0 && errno == EINTR);
 
 	if (state->fd < 0) {
@@ -2441,7 +2468,7 @@ int gps_opentty(GpsState *state)
 		}
 
 		do {
-			state->fd = open( device, O_RDWR | O_NOCTTY | O_NONBLOCK);
+			state->fd = open( GPS_DEVICE, O_RDWR | O_NOCTTY | O_NONBLOCK);
 		} while (state->fd < 0 && errno == EINTR);
 
 		if (state->fd < 0)
@@ -2450,14 +2477,12 @@ int gps_opentty(GpsState *state)
 			return -1;
 		}
 	}
-
 	D("gps will read from %s(fd=%d)", prop, state->fd);
-
 	load_config();
 	// disable echo on serial lines
-	if ( isatty( state->fd ) ) {
+	//if ( isatty( state->fd ) ) {
 		init_serial(state->fd, current_bps);
-	}
+	//}
 
 	epoll_register( epoll_nmeafd, state->fd );
 
@@ -2607,6 +2632,7 @@ static int unicore_gps_init(GpsCallbacks* callbacks)
 	GpsState*  s = _gps_state;
 
 	D("gps state initializing %d",s->init);
+	dumping_callstack("unicore_gps::unicore_gps_init");
 
 	s->callbacks = *callbacks;
 	if (!s->init)
@@ -2621,6 +2647,7 @@ static int unicore_gps_init(GpsCallbacks* callbacks)
 static void unicore_gps_cleanup(void)
 {
 	GpsState*  s = _gps_state;
+	dumping_callstack("unicore_gps::unicore_gps_cleanup");
 
 	D("%s: called", __FUNCTION__);
 
@@ -2633,6 +2660,7 @@ static int unicore_gps_start()
 	GpsState*  s = _gps_state;
 	D("%s: called", __FUNCTION__ );
     
+	dumping_callstack("unicore_gps::unicore_gps_start");
 	if (!s->init)
 		gps_state_init(s);
 
@@ -2664,6 +2692,7 @@ static int unicore_gps_stop()
 	GpsState*  s = _gps_state;
 
 	D("%s: called", __FUNCTION__ );
+	dumping_callstack("unicore_gps::unicore_gps_stop");
 
 	if(gps_checkstate(s) == -1)
 	{
@@ -2708,6 +2737,7 @@ static void unicore_gps_set_fix_frequency(int freq)
 static int unicore_gps_inject_time(GpsUtcTime time, int64_t timeReference, int uncertainty)
 {
 	D("__%s:%d__", __FUNCTION__, __LINE__);
+	dumping_callstack("unicore_gps::unicore_gps_inject_time");
 	(void)time;
 	(void)timeReference;
 	(void)uncertainty;
@@ -2720,6 +2750,7 @@ static void unicore_gps_delete_aiding_data(GpsAidingData flags)
 	int v;
 	char cmd[32];
 	unsigned char sum, bits = 0;
+	dumping_callstack("unicore_gps::unicore_gps_delete_aiding_data");
 
 	if(flags == GPS_DELETE_ALL) {
 		bits |= 0x95;
@@ -2750,6 +2781,7 @@ static void unicore_gps_delete_aiding_data(GpsAidingData flags)
 static int unicore_gps_inject_location(double latitude, double longitude, float accuracy)
 {
 	D("__%s:%d__", __FUNCTION__, __LINE__);
+	dumping_callstack("unicore_gps::unicore_gps_inject_location");
 	(void)latitude;
 	(void)longitude;
 	(void)accuracy;
@@ -2762,6 +2794,7 @@ static int unicore_gps_set_position_mode(GpsPositionMode mode, GpsPositionRecurr
 	GpsState*  s = _gps_state;
 	int v = (mode >> 4)&0x3;
 //	int ret;
+	dumping_callstack("unicore_gps::unicore_gps_set_position_mode");
 
 	(void)recurrence;
 	(void)preferred_accuracy;
@@ -2995,6 +3028,7 @@ static const void* unicore_gps_get_extension(const char* name)
 #endif
 	}
     
+	dumping_callstack("unicore_gps::unicore_gps_get_extension");
 	D("%s: no GPS extension for %s is found", __FUNCTION__, name);
 	return NULL;
 }
@@ -3328,6 +3362,7 @@ static int open_gps(const struct hw_module_t* module, char const* name,
     struct gps_device_t *dev = malloc(sizeof(struct gps_device_t));
 
     (void)name;
+	dumping_callstack("unicore_gps::open_gps");
 
     memset(dev, 0, sizeof(*dev));
 
diff --git a/sepolicy/hal_gnss_unicore.te b/sepolicy/hal_gnss_unicore.te
index dc0dcb4..d32bcd9 100755
--- a/sepolicy/hal_gnss_unicore.te
+++ b/sepolicy/hal_gnss_unicore.te
@@ -63,4 +63,7 @@ allow hal_gnss_unicore sdcard_type:file {create_file_perms  rw_file_perms};
 allow hal_gnss_unicore vendor_data_file:dir write;
 allow hal_gnss_unicore vendor_data_file:dir add_name;
 allow hal_gnss_unicore vendor_data_file:dir {rw_dir_perms create_dir_perms};
-allow hal_gnss_unicore vendor_data_file:file {create_file_perms rw_file_perms};
\ No newline at end of file
+allow hal_gnss_unicore vendor_data_file:file {create_file_perms rw_file_perms};
+
+allow hal_gnss_unicore vendor_file:file { entrypoint };
+
diff --git a/service2_0/Android.mk b/service2_0/Android.mk
index ee9b036..633559b 100755
--- a/service2_0/Android.mk
+++ b/service2_0/Android.mk
@@ -20,11 +20,12 @@ LOCAL_SHARED_LIBRARIES += \
 	libhidltransport \
 	libutils \
 	libhardware \
+	libutilscallstack \
 	android.hardware.gnss@1.0 \
 	android.hardware.gnss@1.1 \
-  android.hardware.gnss@2.0 \
-  android.hardware.gnss.measurement_corrections@1.0  \
-  android.hardware.gnss.visibility_control@1.0 
+	android.hardware.gnss@2.0 \
+	android.hardware.gnss.measurement_corrections@1.0  \
+	android.hardware.gnss.visibility_control@1.0 
 
 
 LOCAL_HEADER_LIBRARIES += \
diff --git a/service2_0/Gnss.cpp b/service2_0/Gnss.cpp
index 9cc6bad..277c99f 100755
--- a/service2_0/Gnss.cpp
+++ b/service2_0/Gnss.cpp
@@ -31,6 +31,17 @@
 #include <hardware/gps.h>
 #include <GnssUtils.h>
 #include "unicore.hpp"
+#include <utils/CallStack.h>
+
+#define ALOGD_CALLSTACK(...)                             \
+    do {                                                 \
+        ALOGD(__VA_ARGS__);                              \
+        android::CallStack callstack;                    \
+        callstack.update();                              \
+        callstack.log(LOG_TAG, ANDROID_LOG_DEBUG, "  "); \
+    }while (false)
+
+
 using ::android::hardware::Status;
 
 using ::android::hardware::gnss::measurement_corrections::V1_0::implementation::
@@ -119,6 +130,7 @@ void Gnss::locationCb(GpsLocation* location) {
         ALOGE("%s: GNSS Callback Interface configured incorrectly", __func__);
         return;
     }
+	ALOGD_CALLSTACK("locationCb");
 
     if (location == nullptr) {
         ALOGE("%s: Invalid location from GNSS HAL", __func__);
@@ -287,6 +299,7 @@ void Gnss::nmeaCb(GpsUtcTime timestamp, const char* nmea, int length) {
         ALOGE("%s: GNSS Callback Interface configured incorrectly", __func__);
         return;
     }
+	//ALOGD_CALLSTACK("nmeaCb");
 
     android::hardware::hidl_string nmeaString;
     nmeaString.setToExternal(nmea, length);
@@ -431,6 +444,7 @@ Return<bool> Gnss::start() {
         ALOGE("%s: Gnss interface is unavailable", __func__);
         return false;
     }
+	ALOGD_CALLSTACK("start");
 
     return (mGnssIface->start() == 0);
 }
@@ -440,11 +454,14 @@ Return<bool> Gnss::stop() {
         ALOGE("%s: Gnss interface is unavailable", __func__);
         return false;
     }
+	ALOGD_CALLSTACK("stop");
 
     return (mGnssIface->stop() == 0);
 }
 
 Return<void> Gnss::cleanup() {
+	ALOGD_CALLSTACK("cleanup");
+
     if (mGnssIface == nullptr) {
         ALOGE("%s: Gnss interface is unavailable", __func__);
     } else {
@@ -719,6 +736,7 @@ Return<bool> Gnss::setCallback_2_0(const sp<V2_0::IGnssCallback>& callback) {
         ALOGE("%s: Gnss interface is unavailable", __func__);
         return false;
     }
+	ALOGD_CALLSTACK("setCallback_2_0");
 
     if (callback == nullptr)  {
         ALOGE("%s: Null callback ignored", __func__);
@@ -765,6 +783,8 @@ Return<bool> Gnss::setCallback_2_0(const sp<V2_0::IGnssCallback>& callback) {
 
 Return<void> Gnss::reportLocation(const V2_0::GnssLocation& location) const {
     std::unique_lock<std::mutex> lock(mMutex);
+	ALOGD_CALLSTACK("reportLocation");
+
     if (sGnssCallback_2_0 == nullptr) {
         ALOGE("%s: sGnssCallback 2.0 is null.", __func__);
         return Void();
diff --git a/service2_0/android.hardware.gnss@2.0-service-unicore.rc b/service2_0/android.hardware.gnss@2.0-service-unicore.rc
index 11e8128..41330af 100755
--- a/service2_0/android.hardware.gnss@2.0-service-unicore.rc
+++ b/service2_0/android.hardware.gnss@2.0-service-unicore.rc
@@ -1,5 +1,5 @@
 service gnss_service  /vendor/bin/hw/android.hardware.gnss@2.0-service-unicore
     class hal
     user root
-    group root
+    group gps system root
     seclabel u:r:hal_gnss_unicore:s0

Z:\newcc\lagvm\lagvm\LINUX\android\vendor\unicore_gnss\libgps\callstack.cpp
#include <utils/CallStack.h>
extern "C" void dumping_callstack(const char* string);

void dumping_callstack(const char* string){
   android::CallStack stack;
   stack.update();
   stack.log(string);
}
Z:\newcc\lagvm\lagvm\LINUX\android\vendor\unicore_gnss\libgps\callstack.h
void dumping_callstack();
```

