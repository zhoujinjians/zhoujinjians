---
title: Android 11 Sensor System源码分析（1）：Sensor流程分析
cover: https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/post.cover.pictures/bing-wallpaper-2018.04.50.jpg
categories: 
 - Display
tags:
 - Display
toc: true
abbrlink: 20230416
date: 2023-04-16 16:16:16
---
注：文章都是通过阅读各位前辈总结的资料、Android 11 Rockchip平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除（◕‿◕），转载请注明出处（©Rockchip ©Android @Linux 版权所有），谢谢。

（==**文章基于 Android 11.0**==）

[【iizhoujinjian.com博客原图链接】](https://github.com/iizhoujinjian) 

[【开发板】](https://wiki.radxa.com/Rockpi4)

[【开发板 源码链接】](https://wiki.radxa.com/Rockpi4/rockpi-android11)



正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢，由于不喜欢图片水印，去除了水印，敬请谅解！！！

[iizhoujinjian.com](http://iizhoujinjian.com) 



--------------------------------------------------------------------------------

==源码（部分）==：

**xxx**

==源码（部分）==：

--------------------------------------------------------------------------------

## （一）、Sensor  Hal  Service初始化

### （1）、rc启动脚本

```c++
Z:\newcc\lagvm\lagvm\LINUX\android\hardware\interfaces\sensors\1.0\default\android.hardware.sensors@1.0-service.rc
service vendor.sensors-hal-1-0 /vendor/bin/hw/android.hardware.sensors@1.0-service
    interface android.hardware.sensors@1.0::ISensors default
    class hal
    user system
    group input system wakelock uhid context_hub
    capabilities BLOCK_SUSPEND
    rlimit rtprio 10 10

on post-fs-data
    chown system system /dev/ttyHS11
    chmod 0777 /dev/ttyHS11

```

### （2）、defaultPassthroughServiceImplementation<ISensors>(2)

> Z:\newcc\lagvm\lagvm\LINUX\android\hardware\interfaces\sensors\1.0\default\service.cpp

![image-20221101134136382](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101134136382.png)

首先看看Trace：

```
01-01 08:00:04.137   459   459 D android.hardware.sensors@1.0-service: HIDL_FETCH_ISensors
01-01 08:00:04.144   459   459 D android.hardware.sensors@1.0-service:   #00 pc 00000000000098b4  /vendor/lib64/hw/android.hardware.sensors@1.0-impl.so (HIDL_FETCH_ISensors+80)
01-01 08:00:04.144   459   459 D android.hardware.sensors@1.0-service:   #01 pc 00000000000604dc  /apex/com.android.vndk.v30/lib64/libhidlbase.so (android::hardware::PassthroughServiceManager::get(android::hardware::hidl_string const&, android::hardware::hidl_string const&)::'lambda'(void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)::operator()(void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&) const+96)
01-01 08:00:04.144   459   459 D android.hardware.sensors@1.0-service:   #02 pc 000000000005b904  /apex/com.android.vndk.v30/lib64/libhidlbase.so (android::hardware::PassthroughServiceManager::openLibs(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::function<bool (void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)> const&)+948)
01-01 08:00:04.144   459   459 D android.hardware.sensors@1.0-service:   #03 pc 000000000005e9e8  /apex/com.android.vndk.v30/lib64/libhidlbase.so (android::hardware::PassthroughServiceManager::get(android::hardware::hidl_string const&, android::hardware::hidl_string const&)+92)
01-01 08:00:04.144   459   459 D android.hardware.sensors@1.0-service:   #04 pc 000000000005ca20  /apex/com.android.vndk.v30/lib64/libhidlbase.so (android::hardware::details::getRawServiceInternal(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, bool, bool)+1600)
01-01 08:00:04.144   459   459 D android.hardware.sensors@1.0-service:   #05 pc 000000000005a560  /apex/com.android.vndk.v30/lib64/libhidlbase.so (android::hardware::details::registerPassthroughServiceImplementation(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::function<int (android::sp<android::hidl::base::V1_0::IBase> const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)>, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)+80)
01-01 08:00:04.144   459   459 D android.hardware.sensors@1.0-service:   #06 pc 000000000005a8e0  /apex/com.android.vndk.v30/lib64/libhidlbase.so (android::hardware::registerPassthroughServiceImplementation(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)+56)
01-01 08:00:04.144   459   459 D android.hardware.sensors@1.0-service:   #07 pc 00000000000011dc  /vendor/bin/hw/android.hardware.sensors@1.0-service
01-01 08:00:04.144   459   459 D android.hardware.sensors@1.0-service:   #08 pc 000000000000109c  /vendor/bin/hw/android.hardware.sensors@1.0-service (main+84)
01-01 08:00:04.144   459   459 D android.hardware.sensors@1.0-service:   #09 pc 00000000000499fc  /apex/com.android.runtime/lib64/bionic/libc.so (__libc_init+108)




Stack Trace:
  RELADDR           FUNCTION                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                FILE:LINE
  00000000000098b4  HIDL_FETCH_ISensors+80                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  hardware/interfaces/sensors/1.0/default/Sensors.cpp:357
  00000000000604dc  android::hardware::PassthroughServiceManager::get(android::hardware::hidl_string const&, android::hardware::hidl_string const&)::{lambda(void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)#1}::operator()(void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&) const+96  system/libhidl/transport/ServiceManagement.cpp:401
  v-------------->  std::__1::__function::__value_func<bool (void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)>::operator()(void*&&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&) const                                                                                                     external/libcxx/include/functional:1799
  v-------------->  std::__1::function<bool (void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)>::operator()(void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&) const                                                                                                                       external/libcxx/include/functional:2347
  000000000005b904  android::hardware::PassthroughServiceManager::openLibs(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::function<bool (void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)> const&)+948                                                                                                                                                                           system/libhidl/transport/ServiceManagement.cpp:378
  000000000005e9e8  android::hardware::PassthroughServiceManager::get(android::hardware::hidl_string const&, android::hardware::hidl_string const&)+92                                                                                                                                                                                                                                                                                                                                                                                                                                      system/libhidl/transport/ServiceManagement.cpp:389
  000000000005ca20  android::hardware::details::getRawServiceInternal(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, bool, bool)+1600                                                                                                                                                                                                                                                                                                          system/libhidl/transport/ServiceManagement.cpp:815
  000000000005a560  android::hardware::details::registerPassthroughServiceImplementation(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::function<int (android::sp<android::hidl::base::V1_0::IBase> const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)>, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)+80                         system/libhidl/transport/LegacySupport.cpp:35
  000000000005a8e0  android::hardware::registerPassthroughServiceImplementation(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)+56                                                                                                                                                                                                                system/libhidl/transport/LegacySupport.cpp:77
  00000000000011dc  int android::hardware::registerPassthroughServiceImplementation<android::hardware::sensors::V1_0::ISensors, android::hardware::sensors::V1_0::ISensors>(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)+80                                                                                                                                                                                                                                                                                                                system/libhidl/transport/include/hidl/LegacySupport.h:56
  v-------------->  int android::hardware::defaultPassthroughServiceImplementation<android::hardware::sensors::V1_0::ISensors, android::hardware::sensors::V1_0::ISensors>(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, unsigned long)                                                                                                                                                                                                                                                                                                     system/libhidl/transport/include/hidl/LegacySupport.h:69
  v-------------->  int android::hardware::defaultPassthroughServiceImplementation<android::hardware::sensors::V1_0::ISensors, android::hardware::sensors::V1_0::ISensors>(unsigned long)                                                                                                                                                                                                                                                                                                                                                                                                   system/libhidl/transport/include/hidl/LegacySupport.h:81
  000000000000109c  main+84                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 hardware/interfaces/sensors/1.0/default/service.cpp:30
  00000000000499fc  __libc_init+108                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         bionic/libc/bionic/libc_init_dynamic.cpp:151

```

### （3）、defaultPassthroughServiceImplementation

> Z:\newcc\lagvm\lagvm\LINUX\android\system\libhidl\transport\include\hidl\LegacySupport.h

![image-20221101142841902](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101142841902.png)

### （4）、LegacySupport::registerPassthroughServiceImplementation

> Z:\newcc\lagvm\lagvm\LINUX\android\system\libhidl\transport\LegacySupport.cpp

![image-20221101142959869](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101142959869.png)

### （5）、ServiceManagement::getRawServiceInternal()

> Z:\newcc\lagvm\lagvm\LINUX\android\system\libhidl\transport\ServiceManagement.cpp

![image-20221101143428089](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101143428089.png)

### （6）、openLibs()

> Z:\newcc\lagvm\lagvm\LINUX\android\system\libhidl\transport\ServiceManagement.cpp

![image-20221101143657746](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101143657746.png)

### （7）、HIDL_FETCH_ISensors

> Z:\newcc\lagvm\lagvm\LINUX\android\system\libhidl\transport\ServiceManagement.cpp

![image-20221101143822142](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101143822142.png)

> Z:\newcc\lagvm\lagvm\LINUX\android\hardware\interfaces\sensors\1.0\default\Sensors.cpp

![image-20221101143907080](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101143907080.png)

### （8）、nwe Sensors()

> Z:\newcc\lagvm\lagvm\LINUX\android\hardware\interfaces\sensors\1.0\default\Sensors.cpp

![image-20221101145124072](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101145124072.png)

```
01-01 08:00:04.163   459   459 D SensorsHal:   #00 pc 00000000000064d4  /vendor/lib64/hw/sensors.msmnile.so (init_nusensors+128)
01-01 08:00:04.163   459   459 D SensorsHal:   #01 pc 000000000000c768  /vendor/lib64/hw/android.hardware.sensors@1.0-impl.so (open_sensors(hw_module_t const*, char const*, hw_device_t**)+400)
01-01 08:00:04.163   459   459 D SensorsHal:   #02 pc 000000000000865c  /vendor/lib64/hw/android.hardware.sensors@1.0-impl.so (android::hardware::sensors::V1_0::implementation::Sensors::Sensors()+388)
01-01 08:00:04.163   459   459 D SensorsHal:   #03 pc 00000000000098e4  /vendor/lib64/hw/android.hardware.sensors@1.0-impl.so (HIDL_FETCH_ISensors+128)
01-01 08:00:04.163   459   459 D SensorsHal:   #04 pc 00000000000604dc  /apex/com.android.vndk.v30/lib64/libhidlbase.so (android::hardware::PassthroughServiceManager::get(android::hardware::hidl_string const&, android::hardware::hidl_string const&)::'lambda'(void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)::operator()(void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&) const+96)
01-01 08:00:04.163   459   459 D SensorsHal:   #05 pc 000000000005b904  /apex/com.android.vndk.v30/lib64/libhidlbase.so (android::hardware::PassthroughServiceManager::openLibs(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::function<bool (void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)> const&)+948)
01-01 08:00:04.163   459   459 D SensorsHal:   #06 pc 000000000005e9e8  /apex/com.android.vndk.v30/lib64/libhidlbase.so (android::hardware::PassthroughServiceManager::get(android::hardware::hidl_string const&, android::hardware::hidl_string const&)+92)
01-01 08:00:04.163   459   459 D SensorsHal:   #07 pc 000000000005ca20  /apex/com.android.vndk.v30/lib64/libhidlbase.so (android::hardware::details::getRawServiceInternal(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, bool, bool)+1600)
01-01 08:00:04.163   459   459 D SensorsHal:   #08 pc 000000000005a560  /apex/com.android.vndk.v30/lib64/libhidlbase.so (android::hardware::details::registerPassthroughServiceImplementation(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::function<int (android::sp<android::hidl::base::V1_0::IBase> const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)>, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)+80)
01-01 08:00:04.163   459   459 D SensorsHal:   #09 pc 000000000005a8e0  /apex/com.android.vndk.v30/lib64/libhidlbase.so (android::hardware::registerPassthroughServiceImplementation(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)+56)
01-01 08:00:04.163   459   459 D SensorsHal:   #10 pc 00000000000011dc  /vendor/bin/hw/android.hardware.sensors@1.0-service
01-01 08:00:04.163   459   459 D SensorsHal:   #11 pc 000000000000109c  /vendor/bin/hw/android.hardware.sensors@1.0-service (main+84)
01-01 08:00:04.163   459   459 D SensorsHal:   #12 pc 00000000000499fc  /apex/com.android.runtime/lib64/bionic/libc.so (__libc_init+108)

Stack Trace:
  RELADDR           FUNCTION                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                FILE:LINE
  00000000000064d4  init_nusensors+128                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      hardware/omosoft/sensor/sensors_omo/nusensors.cpp:333
  000000000000c768  open_sensors(hw_module_t const*, char const*, hw_device_t**)+400                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        hardware/libhardware/modules/sensors/multihal.cpp:823
  v-------------->  sensors_open_1(hw_module_t const*, sensors_poll_device_1**)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             hardware/libhardware/include/hardware/sensors.h:727
  000000000000865c  android::hardware::sensors::V1_0::implementation::Sensors::Sensors()+388                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                hardware/interfaces/sensors/1.0/default/Sensors.cpp:95
  00000000000098e4  HIDL_FETCH_ISensors+128                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 hardware/interfaces/sensors/1.0/default/Sensors.cpp:359
  00000000000604dc  android::hardware::PassthroughServiceManager::get(android::hardware::hidl_string const&, android::hardware::hidl_string const&)::{lambda(void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)#1}::operator()(void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&) const+96  system/libhidl/transport/ServiceManagement.cpp:401
  v-------------->  std::__1::__function::__value_func<bool (void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)>::operator()(void*&&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&) const                                                                                                     external/libcxx/include/functional:1799
  v-------------->  std::__1::function<bool (void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)>::operator()(void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&) const                                                                                                                       external/libcxx/include/functional:2347
  000000000005b904  android::hardware::PassthroughServiceManager::openLibs(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::function<bool (void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)> const&)+948                                                                                                                                                                           system/libhidl/transport/ServiceManagement.cpp:378
  000000000005e9e8  android::hardware::PassthroughServiceManager::get(android::hardware::hidl_string const&, android::hardware::hidl_string const&)+92                                                                                                                                                                                                                                                                                                                                                                                                                                      system/libhidl/transport/ServiceManagement.cpp:389
  000000000005ca20  android::hardware::details::getRawServiceInternal(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, bool, bool)+1600                                                                                                                                                                                                                                                                                                          system/libhidl/transport/ServiceManagement.cpp:815
  000000000005a560  android::hardware::details::registerPassthroughServiceImplementation(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::function<int (android::sp<android::hidl::base::V1_0::IBase> const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)>, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)+80                         system/libhidl/transport/LegacySupport.cpp:35
  000000000005a8e0  android::hardware::registerPassthroughServiceImplementation(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)+56                                                                                                                                                                                                                system/libhidl/transport/LegacySupport.cpp:77
  00000000000011dc  int android::hardware::registerPassthroughServiceImplementation<android::hardware::sensors::V1_0::ISensors, android::hardware::sensors::V1_0::ISensors>(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)+80                                                                                                                                                                                                                                                                                                                system/libhidl/transport/include/hidl/LegacySupport.h:56
  v-------------->  int android::hardware::defaultPassthroughServiceImplementation<android::hardware::sensors::V1_0::ISensors, android::hardware::sensors::V1_0::ISensors>(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, unsigned long)                                                                                                                                                                                                                                                                                                     system/libhidl/transport/include/hidl/LegacySupport.h:69
  v-------------->  int android::hardware::defaultPassthroughServiceImplementation<android::hardware::sensors::V1_0::ISensors, android::hardware::sensors::V1_0::ISensors>(unsigned long)                                                                                                                                                                                                                                                                                                                                                                                                   system/libhidl/transport/include/hidl/LegacySupport.h:81
  000000000000109c  main+84                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 hardware/interfaces/sensors/1.0/default/service.cpp:30
  00000000000499fc  __libc_init+108                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         bionic/libc/bionic/libc_init_dynamic.cpp:151

```



### （9）、sensors_open_1(&mSensorModule->common, &mSensorDevice)

> Z:\newcc\lagvm\lagvm\LINUX\android\hardware\libhardware\include\hardware\sensors.h

![image-20221101145336289](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101145336289.png)

### （11）、sensors_module->common.methods->open(*it, name, &sub_hw_device)

> Z:\newcc\lagvm\lagvm\LINUX\android\hardware\libhardware\modules\sensors\multihal.cpp

![image-20221101145417368](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101145417368.png)

### （12）、open_sensors()

> Z:\newcc\lagvm\lagvm\LINUX\android\hardware\omosoft\sensor\sensors_omo\sensors.c

![image-20221101145734324](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101145734324.png)

### （13）、init_nusensors

> Z:\newcc\lagvm\lagvm\LINUX\android\hardware\xxxsoft\sensor\sensors_xxx\nusensor.cpp

![image-20221101144312295](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101144312295.png)

### （14）、new GyroSensor

> Z:\newcc\lagvm\lagvm\LINUX\android\hardware\xxxsoft\sensor\sensors_xxx\nusensor.cpp

![image-20221101144434913](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101144434913.png)

### （15）、Open内核节点初始化

> Z:\newcc\lagvm\lagvm\LINUX\android\hardware\xxxsoft\sensor\sensors_xxx\GyroSensor.cpp

![image-20221101144511006](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101144511006.png)

### （16）、sensors_poll_context_t::addSubHwDevice()

![image-20221101150607030](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101150607030.png)



![image-20221101150640127](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101150640127.png)

![image-20221101144718272](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101144718272.png)

### （18）、开始poll

```c++

Stack Trace:
  RELADDR           FUNCTION                                                       FILE:LINE
  0000000000008c70  GyroSensor::readEvents(sensors_event_t*, int)+132              hardware/omosoft/sensor/sensors_omo/GyroSensor.cpp:531
  0000000000006304  sensors_poll_context_t::pollEvents(sensors_event_t*, int)+64   hardware/omosoft/sensor/sensors_omo/nusensors.cpp:226
  000000000000691c  poll__poll(sensors_poll_device_t*, sensors_event_t*, int)+156  hardware/omosoft/sensor/sensors_omo/nusensors.cpp:305
  000000000000a7f8  writerTask(void*)+236                                          hardware/libhardware/modules/sensors/multihal.cpp:156
  00000000000afecc  __pthread_start(void*)+64                                      bionic/libc/bionic/pthread_create.cpp:347
  0000000000050408  __start_thread+64                                              bionic/libc/bionic/clone.cpp:53

```

![image-20221101152757180](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101152757180.png)

![image-20221101152850325](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101152850325.png)

![image-20221101152919616](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101152919616.png)

### （18）、registerService (ISensors/default) （Hal Sensor Service注册完成）

![image-20221101150451638](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101150451638.png)

## （二）、SensorService（system_server）

### （1）、startSensorService

![image-20221101151144815](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101151144815.png)

![image-20221101151325709](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101151325709.png)

### （2）、SensorService.publish()

SensorService继承 frameworks/native/services/sensorservice/SensorService.h

继承BinderService、BnSensorServer、Thread

```c++
class SensorService :
        public BinderService<SensorService>,
        public BnSensorServer,
        protected Thread
```


1、BinderService：frameworks/native/libs/binder/include/binder/BinderService.h
BinderService 是 Android Service 框架的主要类，是个模板类，它提供了 Service 的生命周期管理、进程间通信、请求响应处理等功能。Android 中的绝大部分 Service 都会继承此类。
new SERVICE()：new一个SensorService，然后 以 “sensorservice”为key，把sensorservice实例加入到ServiceManaer
2、BnSensorServer：frameworks/native/libs/sensor/include/sensor/ISensorServer.h
和 BinderService 主要实现IPC跨进程通信，实际继承BnInterface<ISensorServer>: frameworks/native/libs/binder/include/binder/IInterface.h
3、Thread：继承 Thread 启动 threadLoop

> SensorService 创建 onFirstRef
> 当 SensorService 第一个实例创建时，其 onFirstRef 接口将会被调用

① 在 SensorDevice 的构造函数中，会调用 hw_get_module 接口加载 Sensor HAL 的动态库
② 通过 SensorDevice，调用 Sensor HAL 提供的 get_sensors_list 接口，获取所支持的 Sensor 信息获，调用registerSensor函数把Sensor保存起来
③ SensorFusion功能，传感融合。它的主要作用就是，按照一定的算法计算系统的多个传感器对某一个值的上报的数据，得到更准确的值。
④ registerVirtualSensor注册虚拟传感器：这些虚拟的传感器步会产生真的数据，而是通过SensorFusion功能计算得到的值，作为虚拟传感的数据。分发过程中会有分析到。
⑤ 初始化一些Buffer，用他们保存sensor硬件上报的数据
⑥ 创建一个 Looper 和 SensorEventAckReceiver。其中 Looper 用于 enable sensor 后，进行数据的接收；而 SensorEventAckReceiver 则用于在 dispatch wake up sensor event 给上层后，接收上层返回的确认 ACK。
⑦ SensorService 不仅是一个服务，而且他还是一个线程，初始化工作的最后就是启动该线程执行threadLoop函数。threadLoop函数主要的工作就是，循环读取sensor硬件上传上来的数据，然后分发给应用。

```c++
void SensorService::onFirstRef() {
// ① 在 SensorDevice 的构造函数中，会调用 hw_get_module 接口加载 Sensor HAL 的动态库
    SensorDevice& dev(SensorDevice::getInstance());
    //.......
    if (dev.initCheck() == NO_ERROR) {
// ② 通过 SensorDevice，调用 Sensor HAL 提供的 get_sensors_list 接口，获取所支持的 Sensor 信息获，调用registerSensor函数把Sensor保存起来
        sensor_t const* list;
        ssize_t count = dev.getSensorList(&list);
        //.......
            for (ssize_t i=0 ; i<count ; i++) {
            //.......
                if (useThisSensor) {
                    registerSensor( new HardwareSensor(list[i]) );
                }
            }
// ③ SensorFusion功能，传感融合。它的主要作用就是，按照一定的算法计算系统的多个传感器对某一个值的上报的数据，得到更准确的值。
            // it's safe to instantiate the SensorFusion object here
            // (it wants to be instantiated after h/w sensors have been
            // registered)
            SensorFusion::getInstance();
// ④ 注册虚拟传感器：这些虚拟的传感器步会产生真的数据，而是通过SensorFusion功能计算得到的值，作为虚拟传感的数据。
           //.......
                registerSensor(new RotationVectorSensor(), !needRotationVector, true);
                registerSensor(new OrientationSensor(), !needRotationVector, true);
                registerSensor(new LinearAccelerationSensor(list, count),
                               !needLinearAcceleration, true);
                registerSensor( new CorrectedGyroSensor(list, count), true, true);
                registerSensor( new GyroDriftSensor(), true, true);
           //.......
                registerSensor(new GravitySensor(list, count), !needGravitySensor, true);
           //.......
                registerSensor(new GameRotationVectorSensor(), !needGameRotationVector, true);
           //.......
                registerSensor(new GeoMagRotationVectorSensor(), !needGeoMagRotationVector, true);
            //.......
// ⑤ 初始化一些Buffer，用他们保存sensor硬件上报的数据
            mLooper = new Looper(false);
            const size_t minBufferSize = SensorEventQueue::MAX_RECEIVE_BUFFER_EVENT_COUNT;
            mSensorEventBuffer = new sensors_event_t[minBufferSize];
            mSensorEventScratch = new sensors_event_t[minBufferSize];
            mMapFlushEventsToConnections = new wp<const SensorEventConnection> [minBufferSize];
            mCurrentOperatingMode = NORMAL;
            //.......
// ⑥ 创建一个 Looper 和 SensorEventAckReceiver。其中 Looper 用于 enable sensor 后，进行数据的接收；而 SensorEventAckReceiver 则用于在 dispatch wake up sensor event 给上层后，接收上层返回的确认 ACK。
            mAckReceiver = new SensorEventAckReceiver(this);
            mAckReceiver->run("SensorEventAckReceiver", PRIORITY_URGENT_DISPLAY);
// ⑦ SensorService 不仅是一个服务，而且他还是一个线程，初始化工作的最后就是启动该线程执行threadLoop函数。threadLoop函数主要的工作就是，循环读取sensor硬件上传上来的数据，然后分发给应用。
            run("SensorService", PRIORITY_URGENT_DISPLAY); 
            //.......
}


```

SensorService::threadLoop()
1、通过poll往hal层取sensor数据， 若没有数据的时候就一直阻塞（该阻塞功能由HAL层实现），当有数据时该函数就会返回
2、virtual sensors 相关数据计算后上报
3、通过SensorEventConnection中 sendEvents 将数据给到每个应用，每个应用都有自己的SensorEventConnection

```c++
bool SensorService::threadLoop() {
    ALOGD("nuSensorService thread starting...");
    //.......
    SensorDevice& device(SensorDevice::getInstance());
const int halVersion = device.getHalDeviceVersion();
do {
    // ① 通过poll往hal层取sensor数据， 若没有数据的时候就一直阻塞（该阻塞功能由HAL层实现），当有数据时该函数就会返回
    ssize_t count = device.poll(mSensorEventBuffer, numEventMax);//①
    if (count < 0) {
        if(count == DEAD_OBJECT && device.isReconnecting()) {
            device.reconnect();
            continue;
        } else {
            ALOGE("sensor poll failed (%s)", strerror(-count));
            break;
        }
    }
    //.......
    // handle virtual sensors
    if (count && vcount) {
        sensors_event_t const * const event = mSensorEventBuffer;
        if (!mActiveVirtualSensors.empty()) {
            size_t k = 0;
            SensorFusion& fusion(SensorFusion::getInstance());
            if (fusion.isEnabled()) {
                for (size_t i=0 ; i<size_t(count) ; i++) {
                    fusion.process(event[i]);
                }
            }
            for (size_t i=0 ; i<size_t(count) && k<minBufferSize ; i++) {
                for (int handle : mActiveVirtualSensors) {
                    if (count + k >= minBufferSize) {
                        ALOGE("buffer too small to hold all events: "
                                "count=%zd, k=%zu, size=%zu",
                                count, k, minBufferSize);
                        break;
                    }
                    sensors_event_t out;
                    sp<SensorInterface> si = mSensors.getInterface(handle);
                    if (si == nullptr) {
                        ALOGE("handle %d is not an valid virtual sensor", handle);
                        continue;
                    }
                    // ② virtual sensors 相关数据计算后上报
                    if (si->process(&out, event[i])) {
                        mSensorEventBuffer[count + k] = out;
                        k++;
                    }
                }
            }
            if (k) {
                // record the last synthesized values
                recordLastValueLocked(&mSensorEventBuffer[count], k);
                count += k;
                // sort the buffer by time-stamps
                sortEventBuffer(mSensorEventBuffer, count);
            }
        }
    }
    //.......
    // Send our events to clients. Check the state of wake lock for each client and release the
    // lock if none of the clients need it.
    bool needsWakeLock = false;
    for (const sp<SensorEventConnection>& connection : activeConnections) {
        // ② 通过SensorEventConnection 将数据给到每个应用，每个应用都有自己的SensorEventConnection
        connection->sendEvents(mSensorEventBuffer, count, mSensorEventScratch,
                mMapFlushEventsToConnections); //②
        needsWakeLock |= connection->needsWakeLock();
        // If the connection has one-shot sensors, it may be cleaned up after first trigger.
        // Early check for one-shot sensors.
        if (connection->hasOneShotSensors()) {
            cleanupAutoDisabledSensorLocked(connection, mSensorEventBuffer, count);
        }
    }

    if (mWakeLockAcquired && !needsWakeLock) {
        setWakeLockAcquiredLocked(false);
    }
} while (!Thread::exitPending());

ALOGW("Exiting SensorService::threadLoop => aborting...");
abort();
return false;
}
```
![SensorService android12-security-release](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/34a0f9a017644fb28f2e27139e4b2369.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210501012937764.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIzNDUyMzg1,size_16,color_FFFFFF,t_70)

## （三）、App（Sensor Client）注册监听

![image-20221101153232441](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101153232441.png)

![image-20221101153126901](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101153126901.png)

### （1）、SystemSensorManager.registerListenerImpl()

![image-20221101153449227](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101153449227.png)

![image-20221101153529455](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101153529455.png)

### （2）、SensorEventQueue(BaseEventQueue)初始化

![image-20221101154138885](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101154138885.png)

![image-20221101154116775](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101154116775.png)


### （3）、SensorManager::createEventQueue()

![image-20221101154230766](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101154230766.png)

### （4）、SensorService::createSensorEventConnection()

![image-20221101154347418](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101154347418.png)

### （4）、new BitTube()

![image-20221101154542443](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101154542443.png)

![image-20221101155018854](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101155018854.png)

管道封装了一层。

### （5）、new Receiver()初始化

![image-20221101155518585](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101155518585.png)

![image-20221101155808397](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101155808397.png)

### （6）、Receiver::onFirstRef()

![image-20221101194529213](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101194529213.png)

![image-20221101194915179](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101194915179.png)

![image-20221101194948003](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101194948003.png)

加入监听，这样管道另外一遍写入，就立即能收到回调。

![image-20221101195239559](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101195239559.png)

## （四）、Sensor 数据 Server（system_server）分发

### （1）、Sensor  Hal  Service获取到内核数据

> Z:\newcc\lagvm\lagvm\LINUX\android\hardware\libhardware\modules\sensors\multihal.cpp

![image-20221101190828179](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101190828179.png)

![image-20221101190926898](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101190926898.png)

![image-20221101191016932](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101191016932.png)

### （2）、SensorService（framework）poll返回

> Z:\newcc\lagvm\lagvm\LINUX\android\frameworks\native\services\sensorservice\SensorService.cpp

![image-20221101191112432](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101191112432.png)

![image-20221101191251916](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101191251916.png)

![image-20221101191205815](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101191205815.png)

### （3）、SensorEventConnection::sendEvents()

> Z:\newcc\lagvm\lagvm\LINUX\android\frameworks\native\services\sensorservice\SensorEventConnection.cpp

![image-20221101191340477](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101191340477.png)

![image-20221101191432385](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101191432385.png)

### （4）、SensorEventQueue::write()

> Z:\newcc\lagvm\lagvm\LINUX\android\frameworks\native\libs\sensor\SensorEventQueue.cpp

![image-20221101191606713](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101191606713.png)

### （5）、BitTube::sendObjects()

> Z:\newcc\lagvm\lagvm\LINUX\android\frameworks\native\libs\sensor\BitTube.cpp

![image-20221101192011504](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101192011504.png)

![image-20221101193707558](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101193707558.png)

## （五）、Sensor 数据 Client（App）接收

### （1）、BitTube::read()

> Z:\newcc\lagvm\lagvm\LINUX\android\frameworks\base\core\java\android\hardware\SystemSensorManager.java

![image-20221101193853364](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101193853364.png)

```c++
Stack Trace:
  RELADDR           FUNCTION                                                                                                           FILE:LINE
  v-------------->  android::sp<android::BitTube>::operator->() const                                                                  system/core/libutils/include/utils/StrongPointer.h:64
  000000000000c458  android::BitTube::sendObjects(android::sp<android::BitTube> const&, void const*, unsigned long, unsigned long)+24  frameworks/native/libs/sensor/BitTube.cpp:160
  v-------------->  ~BpInterface                                                                                                       frameworks/native/libs/binder/include/binder/IInterface.h:86
  000000000000c5d0  android::BpSensorEventConnection::~BpSensorEventConnection()+28                                                    frameworks/native/libs/sensor/ISensorEventConnection.cpp:115
  0000000000010678  android::SensorEventQueue::getLooper() const+12                                                                    frameworks/native/libs/sensor/SensorEventQueue.cpp:99
  00000000001576e4  (anonymous namespace)::Receiver::handleEvent(int, int, void*)+148                                                  frameworks/base/core/jni/android_hardware_SensorManager.cpp:341
  0000000000019dac  android::Looper::pollInner(int)+916                                                                                system/core/libutils/Looper.cpp:355
  00000000000199b0  android::Looper::pollOnce(int, int*, int*, void**)+112                                                             system/core/libutils/Looper.cpp:207
  v-------------->  android::Looper::pollOnce(int)                                                                                     system/core/libutils/include/utils/Looper.h:267
  v-------------->  android::NativeMessageQueue::pollOnce(_JNIEnv*, _jobject*, int)                                                    frameworks/base/core/jni/android_os_MessageQueue.cpp:110
  0000000000110f80  android::android_os_MessageQueue_nativePollOnce(_JNIEnv*, _jobject*, long, int)+44                                 frameworks/base/core/jni/android_os_MessageQueue.cpp:191

```

![image-20221101194031817](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101194031817.png)

![image-20221101194054521](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101194054521.png)

### （2）、dispatchSensorEvent()

> Z:\newcc\lagvm\lagvm\LINUX\android\frameworks\base\core\java\android\hardware\SystemSensorManager.java

![image-20221101194147721](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101194147721.png)

### （3）、SensorEventListener.onSensorChanged()

### ![image-20221101194651375](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/image-20221101194651375.png)

### （4）、截图撒花

![img](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/Android_Sensor_System/Android11_Sensor01/pvyeo-liusa.gif)

## （六）、参考资料(特别感谢)：

###  [（1）SensorService启动](https://blog.csdn.net/qq_23452385/article/details/107308202) 

### （2）Sensor分析Patch.diff

```c++
zhoujinjian@localhost:~/newcc/lagvm/lagvm/LINUX/android/frameworks/base$ git diff core/
diff --git a/core/java/android/hardware/SystemSensorManager.java b/core/java/android/hardware/SystemSensorManager.java
old mode 100644
new mode 100755
index 974913b290b..37f515b8112
--- a/core/java/android/hardware/SystemSensorManager.java
+++ b/core/java/android/hardware/SystemSensorManager.java
@@ -144,6 +144,10 @@ public class SystemSensorManager extends SensorManager {
     @Override
     protected boolean registerListenerImpl(SensorEventListener listener, Sensor sensor,
             int delayUs, Handler handler, int maxBatchReportLatencyUs, int reservedFlags) {
+
+        Log.e(TAG, "registerListenerImpl listener start : delayUs " + delayUs);
+        Log.i(TAG, "registerListenerImpl listener start", new RuntimeException("here").fillInStackTrace());
+
         if (listener == null || sensor == null) {
             Log.e(TAG, "sensor or listener is null");
             return false;
@@ -175,6 +179,7 @@ public class SystemSensorManager extends SensorManager {
                         listener.getClass().getEnclosingClass() != null
                             ? listener.getClass().getEnclosingClass().getName()
                             : listener.getClass().getName();
+                Log.e(TAG, "registerListenerImpl listener: " + fullClassName);
                 queue = new SensorEventQueue(listener, looper, this, fullClassName);
                 if (!queue.addSensor(sensor, delayUs, maxBatchReportLatencyUs)) {
                     queue.dispose();
@@ -195,6 +200,7 @@ public class SystemSensorManager extends SensorManager {
         if (sensor != null && sensor.getReportingMode() == Sensor.REPORTING_MODE_ONE_SHOT) {
             return;
         }
+        Log.i(TAG, "unregisterListenerImpl removed event listener" + sensor);

         synchronized (mSensorListeners) {
             SensorEventQueue queue = mSensorListeners.get(listener);
@@ -807,6 +813,7 @@ public class SystemSensorManager extends SensorManager {
         protected void dispatchSensorEvent(int handle, float[] values, int inAccuracy,
                 long timestamp) {
             final Sensor sensor = mManager.mHandleToSensor.get(handle);
+            Log.e(TAG, "dispatchSensorEvent start handle:" + handle);
             if (sensor == null) {
                 // sensor disconnected
                 return;
@@ -834,6 +841,7 @@ public class SystemSensorManager extends SensorManager {
                 mSensorAccuracies.put(handle, t.accuracy);
                 mListener.onAccuracyChanged(t.sensor, t.accuracy);
             }
+            Log.e(TAG, "dispatchSensorEvent onSensorChanged.");
             mListener.onSensorChanged(t);
         }

diff --git a/core/jni/android_hardware_SensorManager.cpp b/core/jni/android_hardware_SensorManager.cpp
index 777471202fd..263155b367f 100755
--- a/core/jni/android_hardware_SensorManager.cpp
+++ b/core/jni/android_hardware_SensorManager.cpp
@@ -319,12 +319,14 @@ public:
     }

     void destroy() {
+               ALOGD("debug4sensor_jni Receiver destroy");
         mMessageQueue->getLooper()->removeFd( mSensorQueue->getFd() );
     }

 private:
     virtual void onFirstRef() {
         LooperCallback::onFirstRef();
+               ALOGD("debug4sensor_jni Receiver onFirstRef fd:%d", mSensorQueue->getFd());
         mMessageQueue->getLooper()->addFd(mSensorQueue->getFd(), 0,
                 ALOOPER_EVENT_INPUT, this, mSensorQueue.get());
     }
@@ -339,10 +341,10 @@ private:
         while ((n = q->read(buffer, 16)) > 0) {
             for (int i=0 ; i<n ; i++) {
                 //zhoujinjian add for CHANGAN_75A121_SENSOR
-                //ALOGD("debug4sensor_jni handleEvent GyroSensor: %.6f gyroY: %.6f gyroZ: %.6f accX: %.6f accY: %.6f accZ: %.6f temperature: %.6f ",
-                //        buffer[i].data[0], buffer[i].data[1], buffer[i].data[2],
-                //        buffer[i].data[3], buffer[i].data[4], buffer[i].data[5],
-                //        buffer[i].data[6]);
+                ALOGD("debug4sensor_jni handleEvent GyroSensor: %.6f gyroY: %.6f gyroZ: %.6f accX: %.6f accY: %.6f accZ: %.6f temperature: %.6f ",
+                        buffer[i].data[0], buffer[i].data[1], buffer[i].data[2],
+                        buffer[i].data[3], buffer[i].data[4], buffer[i].data[5],
+                        buffer[i].data[6]);
                 if (buffer[i].type == SENSOR_TYPE_STEP_COUNTER) {
                     // step-counter returns a uint64, but the java API only deals with floats
                     float value = float(buffer[i].u64.step_counter);
@@ -403,7 +405,7 @@ private:
                     }
                     if (receiverObj.get()) {
                         //zhoujinjian add for CHANGAN_75A121_SENSOR
-                        //ALOGE("debug4sensor_jni handleEvent GyroSensor dispatchSensorEvent.");
+                        ALOGE("debug4sensor_jni handleEvent GyroSensor dispatchSensorEvent.");
                         env->CallVoidMethod(receiverObj.get(),
                                             gBaseEventQueueClassInfo.dispatchSensorEvent,
                                             buffer[i].sensor,
@@ -440,7 +442,7 @@ static jlong nativeInitSensorEventQueue(JNIEnv *env, jclass clazz, jlong sensorM
     }

     //zhoujinjian add for CHANGAN_75A121_SENSOR
-    //ALOGE("debug4sensor_jni nativeInitSensorEventQueue mode: %d" , mode);
+    ALOGE("debug4sensor_jni nativeInitSensorEventQueue mode: %d" , mode);

     sp<MessageQueue> messageQueue = android_os_MessageQueue_getMessageQueue(env, msgQ);
     if (messageQueue == NULL) {
@@ -448,6 +450,8 @@ static jlong nativeInitSensorEventQueue(JNIEnv *env, jclass clazz, jlong sensorM
         return 0;
     }

+    ALOGE("debug4sensor_jni nativeInitSensorEventQueue mode: %d" , mode);
+
     sp<Receiver> receiver = new Receiver(queue, messageQueue, eventQWeak);
     receiver->incStrong((void*)nativeInitSensorEventQueue);
     return jlong(receiver.get());




zhoujinjian@localhost:~/newcc/lagvm/lagvm/LINUX/android$ cd frameworks/native/services/
zhoujinjian@localhost:~/newcc/lagvm/lagvm/LINUX/android/frameworks/native/services$ git diff .
diff --git a/services/sensorservice/SensorDevice.cpp b/services/sensorservice/SensorDevice.cpp
index 682e3b07d..1f6a14b4e 100755
--- a/services/sensorservice/SensorDevice.cpp
+++ b/services/sensorservice/SensorDevice.cpp
@@ -535,10 +535,10 @@ ssize_t SensorDevice::pollHal(sensors_event_t* buffer, size_t count) {
                     const auto &events,
                     const auto &dynamicSensorsAdded) {
                     if (result == Result::OK) {
-                        //ALOGE("debug4sensor_dev pollHal gyroX: %.6f gyroY: %.6f gyroZ: %.6f accX: %.6f accY: %.6f accZ: %.6f temperature: %.6f ",
-                        //    buffer[0].data[0], buffer[0].data[1], buffer[0].data[2],
-                        //    buffer[0].data[3], buffer[0].data[4], buffer[0].data[5],
-                        //    buffer[0].data[6]);
+                        ALOGE("debug4sensor_dev pollHal gyroX: %.6f gyroY: %.6f gyroZ: %.6f accX: %.6f accY: %.6f accZ: %.6f temperature: %.6f ",
+                            buffer[0].data[0], buffer[0].data[1], buffer[0].data[2],
+                            buffer[0].data[3], buffer[0].data[4], buffer[0].data[5],
+                            buffer[0].data[6]);

                         convertToSensorEventsAndQuantize(convertToNewEvents(events),
                                 convertToNewSensorInfos(dynamicSensorsAdded), buffer);
@@ -1188,7 +1188,7 @@ void SensorDevice::convertToSensorEventsAndQuantize(
         android::SensorDeviceUtils::quantizeSensorEventValues(&dst[i],
                 getResolutionForSensor(dst[i].sensor));
     }
-    //ALOGD("debug4sensor_dev convertToSensorEventsAndQuantize... end");
+    ALOGD("debug4sensor_dev convertToSensorEventsAndQuantize... end");
 }

 float SensorDevice::getResolutionForSensor(int sensorHandle) {
diff --git a/services/sensorservice/SensorEventConnection.cpp b/services/sensorservice/SensorEventConnection.cpp
index 244b4bd90..9c7169ae1 100755
--- a/services/sensorservice/SensorEventConnection.cpp
+++ b/services/sensorservice/SensorEventConnection.cpp
@@ -298,10 +298,10 @@ status_t SensorService::SensorEventConnection::sendEvents(
     if (scratch) {
         size_t i=0;
         while (i<numEvents) {
-            //ALOGE("debug4sensor_svc SensorEventConnection::sendEvents gyroX: %.6f gyroY: %.6f gyroZ: %.6f accX: %.6f accY: %.6f accZ: %.6f temperature: %.6f ",
-            //    buffer[i].data[0], buffer[i].data[1], buffer[i].data[2],
-            //    buffer[i].data[3], buffer[i].data[4], buffer[i].data[5],
-            //    buffer[i].data[6]);
+            ALOGE("debug4sensor_svc SensorEventConnection::sendEvents gyroX: %.6f gyroY: %.6f gyroZ: %.6f accX: %.6f accY: %.6f accZ: %.6f temperature: %.6f ",
+                buffer[i].data[0], buffer[i].data[1], buffer[i].data[2],
+                buffer[i].data[3], buffer[i].data[4], buffer[i].data[5],
+                buffer[i].data[6]);
             int32_t sensor_handle = buffer[i].sensor;
             if (buffer[i].type == SENSOR_TYPE_META_DATA) {
                 ALOGD_IF(DEBUG_CONNECTIONS, "flush complete event sensor==%d ",
@@ -327,6 +327,8 @@ status_t SensorService::SensorEventConnection::sendEvents(
                 ALOGD_IF(DEBUG_CONNECTIONS, "First flush event for sensor==%d ",
                         buffer[i].meta_data.sensor);
                 ++i;
+
+                               ALOGE("debug4sensor_svc SensorEventConnection:: 11111 count == 0 %d", count);
                 continue;
             }

@@ -334,6 +336,8 @@ status_t SensorService::SensorEventConnection::sendEvents(
             // ignore the event and proceed to the next.
             if (flushInfo.mFirstFlushPending) {
                 ++i;
+
+                               ALOGE("debug4sensor_svc SensorEventConnection:: 22222 count == 0 %d", count);
                 continue;
             }

@@ -353,6 +357,7 @@ status_t SensorService::SensorEventConnection::sendEvents(
                         scratch[count++] = buffer[i];
                     }
                 }
+                               ALOGE("debug4sensor_svc SensorEventConnection:: 33333 do-while count == 0 %d", count);
                 i++;
             } while ((i<numEvents) && ((buffer[i].sensor == sensor_handle &&
                                         buffer[i].type != SENSOR_TYPE_META_DATA) ||
@@ -377,6 +382,8 @@ status_t SensorService::SensorEventConnection::sendEvents(
     sendPendingFlushEventsLocked();
     // Early return if there are no events for this connection.
     if (count == 0) {
+
+               ALOGE("debug4sensor_svc SensorEventConnection::count == 0 %d", count);
         return status_t(NO_ERROR);
     }

@@ -386,6 +393,7 @@ status_t SensorService::SensorEventConnection::sendEvents(
     if (mCacheSize != 0) {
         // There are some events in the cache which need to be sent first. Copy this buffer to
         // the end of cache.
+               ALOGE("debug4sensor_svc SensorEventConnection::mCacheSize != 0 %d", mCacheSize);
         appendEventsToCacheLocked(scratch, count);
         return status_t(NO_ERROR);
     }
@@ -403,6 +411,7 @@ status_t SensorService::SensorEventConnection::sendEvents(
     }

     // NOTE: ASensorEvent and sensors_event_t are the same type.
+       ALOGE("debug4sensor_svc SensorEventConnection::write %d", count);
     ssize_t size = SensorEventQueue::write(mChannel,
                                     reinterpret_cast<ASensorEvent const*>(scratch), count);
     if (size < 0) {
@@ -441,8 +450,9 @@ status_t SensorService::SensorEventConnection::sendEvents(
 }

 bool SensorService::SensorEventConnection::hasSensorAccess() {
-    return mService->isUidActive(mUid)
-        && !mService->mSensorPrivacyPolicy->isSensorPrivacyEnabled();
+    return true;
+    //return mService->isUidActive(mUid)
+    //    && !mService->mSensorPrivacyPolicy->isSensorPrivacyEnabled();
 }

 bool SensorService::SensorEventConnection::noteOpIfRequired(const sensors_event_t& event) {
@@ -756,10 +766,10 @@ int SensorService::SensorEventConnection::handleEvent(int fd, int events, void*
                 sensors_event_t sensor_event;
                 memcpy(&sensor_event, buf, sizeof(sensors_event_t));

-                //ALOGE("debug4sensor_svc SensorEventConnection::handleEvent gyroX: %.6f gyroY: %.6f gyroZ: %.6f accX: %.6f accY: %.6f accZ: %.6f temperature: %.6f ",
-                //    sensor_event.data[0], sensor_event.data[1], sensor_event.data[2],
-                //    sensor_event.data[3], sensor_event.data[4], sensor_event.data[5],
-                //    sensor_event.data[6]);
+                ALOGE("debug4sensor_svc SensorEventConnection::handleEvent gyroX: %.6f gyroY: %.6f gyroZ: %.6f accX: %.6f accY: %.6f accZ: %.6f temperature: %.6f ",
+                    sensor_event.data[0], sensor_event.data[1], sensor_event.data[2],
+                    sensor_event.data[3], sensor_event.data[4], sensor_event.data[5],
+                    sensor_event.data[6]);

                 sp<SensorInterface> si =
                         mService->getSensorInterfaceFromHandle(sensor_event.sensor);
diff --git a/services/sensorservice/SensorService.cpp b/services/sensorservice/SensorService.cpp
index 2ae6ac6f5..29e65388c 100755
--- a/services/sensorservice/SensorService.cpp
+++ b/services/sensorservice/SensorService.cpp
@@ -865,7 +865,7 @@ bool SensorService::threadLoop() {
                 break;
             }
         }
-        //ALOGD("debug4sensor_svc nuSensorService threadLoop count... %zd", count);
+        ALOGD("debug4sensor_svc nuSensorService threadLoop count... %zd", count);

         // Reset sensors_event_t.flags to zero for all events in the buffer.
         for (int i = 0; i < count; i++) {
@@ -1013,7 +1013,7 @@ bool SensorService::threadLoop() {
         bool needsWakeLock = false;

         for (const sp<SensorEventConnection>& connection : activeConnections) {
-            //ALOGD("debug4sensor_svc nuSensorService sendEvents...");
+            ALOGD("debug4sensor_svc nuSensorService sendEvents...");
             connection->sendEvents(mSensorEventBuffer, count, mSensorEventScratch,
                     mMapFlushEventsToConnections);
             needsWakeLock |= connection->needsWakeLock();
@@ -1265,12 +1265,14 @@ sp<ISensorEventConnection> SensorService::createSensorEventConnection(const Stri
     sp<SensorEventConnection> result(new SensorEventConnection(this, uid, connPackageName,
             requestedMode == DATA_INJECTION, connOpPackageName));
     if (requestedMode == DATA_INJECTION) {
+               ALOGD("debug4sensor_svc nuSensorService addEventConnectionIfNotPresent...");
+
         mConnectionHolder.addEventConnectionIfNotPresent(result);
         // Add the associated file descriptor to the Looper for polling whenever there is data to
         // be injected.
         result->updateLooperRegistration(mLooper);
     }
-    //ALOGD("debug4sensor_svc nuSensorService createSensorEventConnection...");
+    ALOGD("debug4sensor_svc nuSensorService createSensorEventConnection...requestedMode:%d", requestedMode);

     return result;
 }
@@ -1662,6 +1664,8 @@ status_t SensorService::enable(const sp<SensorEventConnection>& connection,
         } else {
             connection->setFirstFlushPending(handle, false);
         }
+               //zhoujinjian add for
+        connection->setFirstFlushPending(handle, false);
     }

     if (err == NO_ERROR) {
diff --git a/services/sensorservice/SensorService.h b/services/sensorservice/SensorService.h
old mode 100644
new mode 100755
index 052cbfe29..10226ea79
--- a/services/sensorservice/SensorService.h
+++ b/services/sensorservice/SensorService.h
@@ -53,7 +53,7 @@

 // ---------------------------------------------------------------------------
 #define IGNORE_HARDWARE_FUSION  false
-#define DEBUG_CONNECTIONS   false
+#define DEBUG_CONNECTIONS   true
 // Max size is 100 KB which is enough to accept a batch of about 1000 events.
 #define MAX_SOCKET_BUFFER_SIZE_BATCHED (100 * 1024)
 // For older HALs which don't support batching, use a smaller socket buffer size.




zhoujinjian@localhost:~/newcc/lagvm/lagvm/LINUX/android/frameworks/native/libs/sensor$ git diff .
diff --git a/libs/sensor/Android.bp b/libs/sensor/Android.bp
old mode 100644
new mode 100755
index e8154a693..cf46444c2
--- a/libs/sensor/Android.bp
+++ b/libs/sensor/Android.bp
@@ -38,6 +38,7 @@ cc_library_shared {
         "libcutils",
         "libutils",
         "liblog",
+               "libutilscallstack",
         "libhardware",
     ],

diff --git a/libs/sensor/BitTube.cpp b/libs/sensor/BitTube.cpp
old mode 100644
new mode 100755
index 93555c8a3..988d656ed
--- a/libs/sensor/BitTube.cpp
+++ b/libs/sensor/BitTube.cpp
@@ -24,6 +24,15 @@
 #include <unistd.h>

 #include <binder/Parcel.h>
+#include <utils/CallStack.h>
+
+#define ALOGD_CALLSTACK(...)                             \
+    do {                                                 \
+        ALOGD(__VA_ARGS__);                              \
+        android::CallStack callstack;                    \
+        callstack.update();                              \
+        callstack.log(LOG_TAG, ANDROID_LOG_DEBUG, "  "); \
+    }while (false)

 namespace android {
 // ----------------------------------------------------------------------------
@@ -105,6 +114,8 @@ int BitTube::getSendFd() const
 ssize_t BitTube::write(void const* vaddr, size_t size)
 {
     ssize_t err, len;
+       //ALOGD_CALLSTACK("BitTube::write");
+
     do {
         len = ::send(mSendFd, vaddr, size, MSG_DONTWAIT | MSG_NOSIGNAL);
         // cannot return less than size, since we're using SOCK_SEQPACKET
@@ -115,6 +126,8 @@ ssize_t BitTube::write(void const* vaddr, size_t size)

 ssize_t BitTube::read(void* vaddr, size_t size)
 {
+       //ALOGD_CALLSTACK("BitTube::read");
+
     ssize_t err, len;
     do {
         len = ::recv(mReceiveFd, vaddr, size, MSG_DONTWAIT);
diff --git a/libs/sensor/SensorEventQueue.cpp b/libs/sensor/SensorEventQueue.cpp
index 7c587389f..34a61baad 100755
--- a/libs/sensor/SensorEventQueue.cpp
+++ b/libs/sensor/SensorEventQueue.cpp
@@ -49,11 +49,13 @@ SensorEventQueue::~SensorEventQueue() {

 void SensorEventQueue::onFirstRef()
 {
+
     mSensorChannel = mSensorEventConnection->getSensorChannel();
 }

 int SensorEventQueue::getFd() const
 {
+       ALOGE("debug4sensor_lib SensorEventQueue::getFd %d", mSensorChannel->getFd());
     return mSensorChannel->getFd();
 }

@@ -61,20 +63,20 @@ int SensorEventQueue::getFd() const
 ssize_t SensorEventQueue::write(const sp<BitTube>& tube,
         ASensorEvent const* events, size_t numEvents) {
        //zhoujinjian add for CHANGAN_75A121_SENSOR
-       //ALOGE("debug4sensor_lib SensorEventQueue::write gyroX: %.6f gyroY: %.6f gyroZ: %.6f accX: %.6f accY: %.6f accZ: %.6f temperature: %.6f ",
-       //      events[0].data[0], events[0].data[1], events[0].data[2],
-       //      events[0].data[3], events[0].data[4], events[0].data[5],
-       //      events[0].data[6]);
+       ALOGE("debug4sensor_lib SensorEventQueue::write gyroX: %.6f gyroY: %.6f gyroZ: %.6f accX: %.6f accY: %.6f accZ: %.6f temperature: %.6f ",
+               events[0].data[0], events[0].data[1], events[0].data[2],
+               events[0].data[3], events[0].data[4], events[0].data[5],
+               events[0].data[6]);

     return BitTube::sendObjects(tube, events, numEvents);
 }

 ssize_t SensorEventQueue::read(ASensorEvent* events, size_t numEvents) {
        //zhoujinjian add for CHANGAN_75A121_SENSOR
-       //ALOGE("debug4sensor_lib SensorEventQueue::read gyroX: %.6f gyroY: %.6f gyroZ: %.6f accX: %.6f accY: %.6f accZ: %.6f temperature: %.6f ",
-       //      events[0].data[0], events[0].data[1], events[0].data[2],
-       //      events[0].data[3], events[0].data[4], events[0].data[5],
-       //      events[0].data[6]);
+       ALOGE("debug4sensor_lib SensorEventQueue::read gyroX: %.6f gyroY: %.6f gyroZ: %.6f accX: %.6f accY: %.6f accZ: %.6f temperature: %.6f ",
+               events[0].data[0], events[0].data[1], events[0].data[2],
+               events[0].data[3], events[0].data[4], events[0].data[5],
+               events[0].data[6]);

     if (mAvailable == 0) {
         ssize_t err = BitTube::recvObjects(mSensorChannel,
diff --git a/libs/sensor/SensorManager.cpp b/libs/sensor/SensorManager.cpp
old mode 100644
new mode 100755
index a4a5d135c..1886c6775
--- a/libs/sensor/SensorManager.cpp
+++ b/libs/sensor/SensorManager.cpp
@@ -227,6 +227,7 @@ Sensor const* SensorManager::getDefaultSensor(int type)

 sp<SensorEventQueue> SensorManager::createEventQueue(String8 packageName, int mode) {
     sp<SensorEventQueue> queue;
+       ALOGE("createEventQueue: connection is NULL. mode:%d", mode);

     Mutex::Autolock _l(mLock);
     while (assertStateLocked() == NO_ERROR) {

[16]+  Stopped                 git diff .


zhoujinjian@localhost:~/newcc/lagvm/lagvm/LINUX/android/hardware/libhardware$ git diff .
diff --git a/modules/sensors/multihal.cpp b/modules/sensors/multihal.cpp
index df219ff7..4a26add7 100755
--- a/modules/sensors/multihal.cpp
+++ b/modules/sensors/multihal.cpp
@@ -17,9 +17,9 @@
 #include "SensorEventQueue.h"
 #include "multihal.h"

-#define LOG_NDEBUG 1
+//#define LOG_NDEBUG 1
 //debug4sensor_hal change to 0
-//#define LOG_NDEBUG 0
+#define LOG_NDEBUG 0
 #include <log/log.h>
 #include <cutils/atomic.h>
 #include <hardware/sensors.h>
@@ -155,10 +155,10 @@ void *writerTask(void* ptr) {
         ALOGV("writerTask before poll() - bufferSize = %d", bufferSize);
         eventsPolled = device->poll(device, buffer, bufferSize);
         ALOGV("writerTask poll() got %d events.", eventsPolled);
-        //ALOGE("debug4sensor_hal writerTask poll() got gyroX: %.6f gyroY: %.6f  gyroZ: %.6f    accX: %.6f    accY: %.6f    accZ: %.6f    temperature: %.6f ",
-        //    buffer[0].data[0], buffer[0].data[1], buffer[0].data[2],
-        //    buffer[0].data[3], buffer[0].data[4], buffer[0].data[5],
-        //    buffer[0].data[6]);
+        ALOGE("debug4sensor_hal writerTask poll() got gyroX: %.6f gyroY: %.6f  gyroZ: %.6f    accX: %.6f    accY: %.6f    accZ: %.6f    temperature: %.6f ",
+            buffer[0].data[0], buffer[0].data[1], buffer[0].data[2],
+            buffer[0].data[3], buffer[0].data[4], buffer[0].data[5],
+            buffer[0].data[6]);

         if (eventsPolled <= 0) {
             if (eventsPolled < 0) {
zhoujinjian@localhost:~/newcc/lagvm/lagvm/LINUX/android/hardware/omosoft/sensor$ git diff .
diff --git a/sensors_omo/Android.mk b/sensors_omo/Android.mk
index 876638f..fc1b618 100755
--- a/sensors_omo/Android.mk
+++ b/sensors_omo/Android.mk
@@ -39,6 +39,7 @@ LOCAL_SRC_FILES := \
 LOCAL_SHARED_LIBRARIES := \
        liblog \
        libcutils \
+       libutilscallstack \
        libutils

 include $(BUILD_SHARED_LIBRARY)
diff --git a/sensors_omo/GyroSensor.cpp b/sensors_omo/GyroSensor.cpp
index b69f80f..abe7518 100755
--- a/sensors_omo/GyroSensor.cpp
+++ b/sensors_omo/GyroSensor.cpp
@@ -44,6 +44,16 @@
 #include <cutils/properties.h>

 #include "GyroSensor.h"
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

 /*****************************************************************************/
 static int         epoll_nmeafd;
@@ -518,6 +528,7 @@ int GyroSensor::readEvents(sensors_event_t* data, int count)
     if(DEBUG_GYRO_SENSOR) {
         D("GyroSensor readEvents need count ==> %d \r\n", count);
     }
+       ALOGD_CALLSTACK("GyroSensor::readEvents");

     int  nn;
     /* select && read nmea data*/
diff --git a/sensors_omo/nusensors.cpp b/sensors_omo/nusensors.cpp
index b62eef5..40060b8 100755
--- a/sensors_omo/nusensors.cpp
+++ b/sensors_omo/nusensors.cpp
@@ -32,6 +32,17 @@
 #include "nusensors.h"
 #include "LightSensor.h"
 #include "GyroSensor.h"
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

 /*****************************************************************************/

@@ -261,6 +272,7 @@ static int poll__close(struct hw_device_t *dev)
 static int poll__activate(struct sensors_poll_device_t *dev,
         int handle, int enabled) {
     sensors_poll_context_t *ctx = (sensors_poll_context_t *)dev;
+       ALOGD_CALLSTACK("poll__activate");

     LOGI("set active: handle = %d, enable = %d\n", handle, enabled);

@@ -278,6 +290,7 @@ static int poll__activate(struct sensors_poll_device_t *dev,
 static int poll__setDelay(struct sensors_poll_device_t *dev,
         int handle, int64_t ns) {
     sensors_poll_context_t *ctx = (sensors_poll_context_t *)dev;
+       ALOGD_CALLSTACK("poll__setDelay");

     LOGI("set delay: handle = %d, delay = %dns\n", handle, (int)ns);

@@ -286,6 +299,8 @@ static int poll__setDelay(struct sensors_poll_device_t *dev,

 static int poll__poll(struct sensors_poll_device_t *dev,
         sensors_event_t* data, int count) {
+       ALOGD_CALLSTACK("poll__poll");
+
     sensors_poll_context_t *ctx = (sensors_poll_context_t *)dev;
     return ctx->pollEvents(data, count);
 }
@@ -294,6 +309,7 @@ static int poll__batch(struct sensors_poll_device_1 *dev,
                       int handle, int flags, int64_t period_ns, int64_t timeout)
 {
     LOGI("set batch: handle = %d, period_ns = %dns, timeout = %dns\n", handle, (int)period_ns, (int)timeout);
+       ALOGD_CALLSTACK("poll__batch");

     sensors_poll_context_t *ctx = (sensors_poll_context_t *)dev;
     return ctx->setDelay(handle, period_ns);
@@ -303,6 +319,7 @@ static int poll__flush(struct sensors_poll_device_1 *dev,
                       int handle)
 {
     LOGI("set flush: handle = %d\n", handle);
+       ALOGD_CALLSTACK("poll__flush");
     sensors_poll_context_t *ctx = (sensors_poll_context_t *)dev;
     return ctx->flush(handle);
 }
@@ -313,6 +330,7 @@ int init_nusensors(hw_module_t const* module, hw_device_t** device)
 {
     LOGD("%s\n",SENSOR_VERSION_AND_TIME);
     int status = -EINVAL;
+       ALOGD_CALLSTACK("init_nusensors");

     sensors_poll_context_t *dev = new sensors_poll_context_t();
     if (!dev->getInitialized()) {

[17]+  Stopped                 git diff .

```

