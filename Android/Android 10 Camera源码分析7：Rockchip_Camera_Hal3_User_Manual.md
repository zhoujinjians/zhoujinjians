---
title: Android 10 Camera源码分析7：Rockchip_Camera_Hal3_User_Manual
cover: https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/post.cover.pictures/bing-wallpaper-2018.04.37.jpg
categories: 
 - Camera
tags:
 - Camera
toc: true
abbrlink: 20220401
date: 2022-04-01 04:01:00
---

注：文章都是通过阅读各位前辈总结的资料 Android 10.0 && Linux（Kernel 4.19）Rockchip平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除（◕‿◕），转载请注明出处（©Rockchip ©Android @Linux 版权所有），谢谢。

（==**文章基于 Kernel-4.19**==）&&（==**文章基于 Android 10.0**==）

[【zhoujinjian.com博客原图链接】](https://github.com/zhoujinjiana) 

[【开发板 Khadas Edge V】](https://www.khadas.cn/edge-v)

[【开发板 Khadas Edge V Android 10.0 && Linux（Kernel 4.19）源码链接】](https://github.com/khadas/linux/tree/khadas-edge-Qt)



正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢！！！

 [Rockchip_Camera_Hal3_User_Manual](https://usermanual.wiki/Document/camerahal3usermanual20.954814665/view) 


--------------------------------------------------------------------------------

==源码（部分）==：

**F:\Khadas_Edge_Android_Q\hardware\rockchip\camera\**

==源码（部分）==：

--------------------------------------------------------------------------------


## （一）、Camera Hal3框架

#### （1）、Camera Hal3 基本框架

![](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/Android.10.Camera.07/rk_camera_hal3_base_arc.png)

Camera hal3 在android框架中所处的位置如上图，对上，主要实现Framework 一整套API接口，响应其控制命令，返回数据与控制参数结果。对下，主要是通V4l2框架实现与kernel的交互。3a控制则是通control loop 接口与camera_engine_isp交互。另外，其中一些组件或功能的实现也会调用到其他一些第三方库，如cameraBuffer 相关，会调用到Galloc相关库，jpeg编码则会调用到Hwjpeg相关库。

#### （2）、代码目录简要说明

![](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/Android.10.Camera.07/rk_camera_hal3_dir.png)

#### （3）、Camera Hal3 基本组件

![](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/Android.10.Camera.07/rk_camera_hal3_base_comp.png)

Camera hal3中的模块主要包括AAL与PSL。

- AAL：主要负责与framework交互，camera_module与API接口实例camera3_device_ops在此模块定义。该模块对此API加以封装，并将请求发往PSL，并等待接收PSL返回相应数据流与控制参数。

- PSL：则是物理层的具体实现，基中gcss、GraphConifg、MediaController主要负责配置文件xml的解析，底层pipeline的配置，ControlUnit 主要负责与camera_engine_isp的交互，以实现3a的控制，并中转一些请求以及Metadata的处理收集上报。ImgUnit、OutputFrameWork、postProcessPipeline则主要负责获取数据帧并做相应处理以及上报。V4l2device、V4l2Subdevice则是负责与v4l2驱动交互，实现具体的io操作。

#### （4）、Camera hal3 与Frame work 交互时序

关于framework 与hal交互的API详细说明文档可以参考：<android_root>/hardware/libhardware/include/hardware/camera3.h

![](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/Android.10.Camera.07/rk_camera_hal3_call_flow.png)

Stack Trace:

```cpp
//Stack Trace: initCameraHAL()
//  RELADDR   FUNCTION                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         FILE:LINE
  000d7477  initCameraHAL()+18                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               hardware/rockchip/camera/Camera3HALModule.cpp:317
  v------>  call_function(char const*, void (*)(int, char**, char**), char const*)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           bionic/linker/linker_soinfo.cpp:353
  0002fa63  __dl__ZL10call_arrayIPFviPPcS1_EEvPKcPT_jbS5_+162                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                bionic/linker/linker_soinfo.cpp:387
  0002fc65  __dl__ZN6soinfo17call_constructorsEv+372                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         bionic/linker/linker_soinfo.cpp:431
  0002071b  __dl_$t.75+622                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   bionic/linker/linker.cpp:2291
  v------>  dlopen_ext(char const*, int, android_dlextinfo const*, void const*)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              bionic/linker/dlfcn.cpp:138
  0001d083  __dl___loader_android_dlopen_ext+38                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              bionic/linker/dlfcn.cpp:150
  00001053  android_dlopen_ext+4                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             bionic/libdl/libdl.cpp:133
  000010c9  android_load_sphal_library+108                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   system/core/libvndksupport/linker.c:74
  v------>  load                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             hardware/libhardware/hardware.c:104
  000011f7  hw_get_module_by_class+482                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       hardware/libhardware/hardware.c:248
  00001401  hw_get_module+40                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 hardware/libhardware/hardware.c:265
  000132b5  android::hardware::camera::provider::V2_4::implementation::LegacyCameraProviderImpl_2_4::initialize()+28                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         hardware/interfaces/camera/provider/2.4/default/LegacyCameraProviderImpl_2_4.cpp:270
  0001327d  android::hardware::camera::provider::V2_4::implementation::LegacyCameraProviderImpl_2_4::LegacyCameraProviderImpl_2_4()+124                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      hardware/interfaces/camera/provider/2.4/default/LegacyCameraProviderImpl_2_4.cpp:263
  v------>  CameraProvider                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   hardware/interfaces/camera/provider/2.4/default/CameraProvider_2_4.h:40
  v------>  android::hardware::camera::provider::V2_4::implementation::CameraProvider<android::hardware::camera::provider::V2_4::implementation::LegacyCameraProviderImpl_2_4>* android::hardware::camera::provider::V2_4::implementation::getProviderImpl<android::hardware::camera::provider::V2_4::implementation::LegacyCameraProviderImpl_2_4>()                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        hardware/interfaces/camera/provider/2.4/default/CameraProvider_2_4.cpp:37
  00002073  HIDL_FETCH_ICameraProvider+94                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    hardware/interfaces/camera/provider/2.4/default/CameraProvider_2_4.cpp:54
  00041e0b  android::hardware::PassthroughServiceManager::get(android::hardware::hidl_string const&, android::hardware::hidl_string const&)::{lambda(void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)#1}::operator()(void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&) const+62                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           system/libhidl/transport/ServiceManagement.cpp:433
  v------>  std::__1::__function::__value_func<bool (void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)>::operator()(void*&&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&) const                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              external/libcxx/include/functional:1799
  v------>  std::__1::function<bool (void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)>::operator()(void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&) const                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                external/libcxx/include/functional:2347
  0003ef39  android::hardware::PassthroughServiceManager::openLibs(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::function<bool (void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)> const&)+1032                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   system/libhidl/transport/ServiceManagement.cpp:410
  00040c83  android::hardware::PassthroughServiceManager::get(android::hardware::hidl_string const&, android::hardware::hidl_string const&)+54                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               system/libhidl/transport/ServiceManagement.cpp:421
  0003fbab  android::hardware::details::getRawServiceInternal(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, bool, bool)+1090                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   system/libhidl/transport/ServiceManagement.cpp:818
  000106bb  android::sp<android::hardware::camera::provider::V2_4::ICameraProvider> android::hardware::details::getServiceInternal<android::hardware::camera::provider::V2_4::BpHwCameraProvider, android::hardware::camera::provider::V2_4::ICameraProvider, void, void>(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, bool, bool)+134                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      system/libhidl/transport/include/hidl/HidlTransportSupport.h:152
  000107d3  android::hardware::camera::provider::V2_4::ICameraProvider::getService(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, bool)+6                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     out/soong/.intermediates/hardware/interfaces/camera/provider/2.4/android.hardware.camera.provider@2.4_genc++/gen/android/hardware/camera/provider/2.4/CameraProviderAll.cpp:1363
  00001107  int android::hardware::details::registerPassthroughServiceImplementation<android::hardware::camera::provider::V2_4::ICameraProvider, int android::hardware::registerPassthroughServiceImplementation<android::hardware::camera::provider::V2_4::ICameraProvider>(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)::{lambda(android::sp<android::hardware::camera::provider::V2_4::ICameraProvider> const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)#1}>(int android::hardware::registerPassthroughServiceImplementation<android::hardware::camera::provider::V2_4::ICameraProvider>(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)::{lambda(android::sp<android::hardware::camera::provider::V2_4::ICameraProvider> const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)#1}, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)+22  system/libhidl/transport/include/hidl/LegacySupport.h:32
  v------>  int android::hardware::registerPassthroughServiceImplementation<android::hardware::camera::provider::V2_4::ICameraProvider>(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        system/libhidl/transport/include/hidl/LegacySupport.h:63
  v------>  int android::hardware::defaultPassthroughServiceImplementation<android::hardware::camera::provider::V2_4::ICameraProvider>(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, unsigned int)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           system/libhidl/transport/include/hidl/LegacySupport.h:79
  000010b1  main+108                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         hardware/interfaces/camera/provider/2.4/default/service.cpp:49
  00059127  __libc_init+66                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   bionic/libc/bionic/libc_init_dynamic.cpp:136
  0000102f  _start_main+38                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   bionic/libc/arch-common/bionic/crtbegin.c:45
  00004456  <unknown>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        <anonymous:e81ea000>


Stack Trace: openCameraHardware()
  RELADDR   FUNCTION                                                                                                                                                                                                                                                                                                                                                          FILE:LINE
  000d6eb7  openCameraHardware(int, hw_module_t const*, hw_device_t**)+182                                                                                                                                                                                                                                                                                                    hardware/rockchip/camera/Camera3HALModule.cpp:98
  000d759f  hal_dev_open(hw_module_t const*, char const*, hw_device_t**)+234                                                                                                                                                                                                                                                                                                  hardware/rockchip/camera/Camera3HALModule.cpp:191
  0001cbb9  android::hardware::camera::common::V1_0::helper::CameraModule::open(char const*, hw_device_t**)+48                                                                                                                                                                                                                                                                hardware/interfaces/camera/common/1.0/default/CameraModule.cpp:397
  00015ba1  android::hardware::camera::device::V3_2::implementation::CameraDevice::open(android::sp<android::hardware::camera::device::V3_2::ICameraDeviceCallback> const&, std::__1::function<void (android::hardware::camera::common::V1_0::Status, android::sp<android::hardware::camera::device::V3_2::ICameraDeviceSession> const&)>)+388                                hardware/interfaces/camera/device/3.2/default/CameraDevice.cpp:212
  00016203  android::hardware::camera::device::V3_2::implementation::CameraDevice::TrampolineDeviceInterface_3_2::open(android::sp<android::hardware::camera::device::V3_2::ICameraDeviceCallback> const&, std::__1::function<void (android::hardware::camera::common::V1_0::Status, android::sp<android::hardware::camera::device::V3_2::ICameraDeviceSession> const&)>)+66  hardware/interfaces/camera/device/3.2/default/CameraDevice_3_2.h:138
  000133c1  android::hardware::camera::device::V3_2::BnHwCameraDevice::_hidl_open(android::hidl::base::V1_0::BnHwBase*, android::hardware::Parcel const&, android::hardware::Parcel*, std::__1::function<void (android::hardware::Parcel&)>)+288                                                                                                                              out/soong/.intermediates/hardware/interfaces/camera/device/3.2/android.hardware.camera.device@3.2_genc++/gen/android/hardware/camera/device/3.2/CameraDeviceAll.cpp:852
  00013a5f  android::hardware::camera::device::V3_2::BnHwCameraDevice::onTransact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+390                                                                                                                                        out/soong/.intermediates/hardware/interfaces/camera/device/3.2/android.hardware.camera.device@3.2_genc++/gen/android/hardware/camera/device/3.2/CameraDeviceAll.cpp:1025
  000689a1  android::hardware::BHwBinder::transact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+52                                                                                                                                                                        system/libhwbinder/Binder.cpp:116
  v------>  android::hardware::IPCThreadState::executeCommand(int)                                                                                                                                                                                                                                                                                                            system/libhwbinder/IPCThreadState.cpp:1204
  0006af8d  android::hardware::IPCThreadState::getAndExecuteCommand()+924                                                                                                                                                                                                                                                                                                     system/libhwbinder/IPCThreadState.cpp:461
  0006be61  android::hardware::IPCThreadState::joinThreadPool(bool)+56                                                                                                                                                                                                                                                                                                        system/libhwbinder/IPCThreadState.cpp:561
  00076f0d  android::hardware::PoolThread::threadLoop()+12                                                                                                                                                                                                                                                                                                                    system/libhwbinder/ProcessState.cpp:61
  0000d8bf  android::Thread::_threadLoop(void*)+186                                                                                                                                                                                                                                                                                                                           system/core/libutils/Threads.cpp:746
  000a6867  __pthread_start(void*)+20                                                                                                                                                                                                                                                                                                                                         bionic/libc/bionic/pthread_create.cpp:338
  00060087  __start_thread+30                                                                                                                                                                                                                                                                                                                                                 bionic/libc/bionic/clone.cpp:53


//Stack Trace: construct_default_request_settings()
//  RELADDR   FUNCTION                                                                                                                                                                                                                                                                                                                                   FILE:LINE
  0006d665  android::camera2::Camera3HAL::construct_default_request_settings(int)+92                                                                                                                                                                                                                                                                   hardware/rockchip/camera/AAL/Camera3HAL.cpp:295
  0006dc15  android::camera2::hal_dev_construct_default_request_settings(camera3_device const*, int)+92                                                                                                                                                                                                                                                hardware/rockchip/camera/AAL/Camera3HAL.cpp:76
  00019c27  android::hardware::camera::device::V3_2::implementation::CameraDeviceSession::constructDefaultRequestSettingsRaw(int, android::hardware::hidl_vec<unsigned char>*)+110                                                                                                                                                                     hardware/interfaces/camera/device/3.2/default/CameraDeviceSession.cpp:847
  00019b67  android::hardware::camera::device::V3_2::implementation::CameraDeviceSession::constructDefaultRequestSettings(android::hardware::camera::device::V3_2::RequestTemplate, std::__1::function<void (android::hardware::camera::common::V1_0::Status, android::hardware::hidl_vec<unsigned char> const&)>)+50                                  hardware/interfaces/camera/device/3.2/default/CameraDeviceSession.cpp:837
  000049bf  android::hardware::camera::device::V3_2::implementation::CameraDeviceSession::TrampolineSessionInterface_3_2::constructDefaultRequestSettings(android::hardware::camera::device::V3_2::RequestTemplate, std::__1::function<void (android::hardware::camera::common::V1_0::Status, android::hardware::hidl_vec<unsigned char> const&)>)+66  hardware/interfaces/camera/device/3.3/default/CameraDeviceSession.h:88
  0001f431  android::hardware::camera::device::V3_2::BnHwCameraDeviceSession::_hidl_constructDefaultRequestSettings(android::hidl::base::V1_0::BnHwBase*, android::hardware::Parcel const&, android::hardware::Parcel*, std::__1::function<void (android::hardware::Parcel&)>)+200                                                                     out/soong/.intermediates/hardware/interfaces/camera/device/3.2/android.hardware.camera.device@3.2_genc++/gen/android/hardware/camera/device/3.2/CameraDeviceSessionAll.cpp:839
  0000b655  android::hardware::camera::device::V3_3::BnHwCameraDeviceSession::onTransact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+396                                                                                                          out/soong/.intermediates/hardware/interfaces/camera/device/3.3/android.hardware.camera.device@3.3_genc++/gen/android/hardware/camera/device/3.3/CameraDeviceSessionAll.cpp:501
  000689a1  android::hardware::BHwBinder::transact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+52                                                                                                                                                 system/libhwbinder/Binder.cpp:116
  v------>  android::hardware::IPCThreadState::executeCommand(int)                                                                                                                                                                                                                                                                                     system/libhwbinder/IPCThreadState.cpp:1204
  0006af8d  android::hardware::IPCThreadState::getAndExecuteCommand()+924                                                                                                                                                                                                                                                                              system/libhwbinder/IPCThreadState.cpp:461
  0006be61  android::hardware::IPCThreadState::joinThreadPool(bool)+56                                                                                                                                                                                                                                                                                 system/libhwbinder/IPCThreadState.cpp:561
  00076f0d  android::hardware::PoolThread::threadLoop()+12                                                                                                                                                                                                                                                                                             system/libhwbinder/ProcessState.cpp:61
  0000d8bf  android::Thread::_threadLoop(void*)+186                                                                                                                                                                                                                                                                                                    system/core/libutils/Threads.cpp:746
  000a6867  __pthread_start(void*)+20                                                                                                                                                                                                                                                                                                                  bionic/libc/bionic/pthread_create.cpp:338
  00060087  __start_thread+30                                                                                                                                                                                                                                                                                                                          bionic/libc/bionic/clone.cpp:53


//Stack Trace: configure_streams()
//  RELADDR   FUNCTION                                                                                                                                                                                                                                                                                                                                                        FILE:LINE
  0006d35b  android::camera2::Camera3HAL::configure_streams(camera3_stream_configuration*)+162                                                                                                                                                                                                                                                                              hardware/rockchip/camera/AAL/Camera3HAL.cpp:250
  0006daa9  android::camera2::hal_dev_configure_streams(camera3_device const*, camera3_stream_configuration*)+92                                                                                                                                                                                                                                                            hardware/rockchip/camera/AAL/Camera3HAL.cpp:60
  00004711  android::hardware::camera::device::V3_3::implementation::CameraDeviceSession::configureStreams_3_3(android::hardware::camera::device::V3_2::StreamConfiguration const&, std::__1::function<void (android::hardware::camera::common::V1_0::Status, android::hardware::camera::device::V3_3::HalStreamConfiguration const&)>)+264                                 hardware/interfaces/camera/device/3.3/default/CameraDeviceSession.cpp:88
  00004d03  android::hardware::camera::device::V3_3::implementation::CameraDeviceSession::TrampolineSessionInterface_3_3::configureStreams_3_3(android::hardware::camera::device::V3_2::StreamConfiguration const&, std::__1::function<void (android::hardware::camera::common::V1_0::Status, android::hardware::camera::device::V3_3::HalStreamConfiguration const&)>)+66  hardware/interfaces/camera/device/3.3/default/CameraDeviceSession.h:123
  0000b313  android::hardware::camera::device::V3_3::BnHwCameraDeviceSession::_hidl_configureStreams_3_3(android::hidl::base::V1_0::BnHwBase*, android::hardware::Parcel const&, android::hardware::Parcel*, std::__1::function<void (android::hardware::Parcel&)>)+222                                                                                                     out/soong/.intermediates/hardware/interfaces/camera/device/3.3/android.hardware.camera.device@3.3_genc++/gen/android/hardware/camera/device/3.3/CameraDeviceSessionAll.cpp:414
  0000b749  android::hardware::camera::device::V3_3::BnHwCameraDeviceSession::onTransact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+640                                                                                                                               out/soong/.intermediates/hardware/interfaces/camera/device/3.3/android.hardware.camera.device@3.3_genc++/gen/android/hardware/camera/device/3.3/CameraDeviceSessionAll.cpp:578
  000689a1  android::hardware::BHwBinder::transact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+52                                                                                                                                                                      system/libhwbinder/Binder.cpp:116
  v------>  android::hardware::IPCThreadState::executeCommand(int)                                                                                                                                                                                                                                                                                                          system/libhwbinder/IPCThreadState.cpp:1204
  0006af8d  android::hardware::IPCThreadState::getAndExecuteCommand()+924                                                                                                                                                                                                                                                                                                   system/libhwbinder/IPCThreadState.cpp:461
  0006be61  android::hardware::IPCThreadState::joinThreadPool(bool)+56                                                                                                                                                                                                                                                                                                      system/libhwbinder/IPCThreadState.cpp:561
  00076f0d  android::hardware::PoolThread::threadLoop()+12                                                                                                                                                                                                                                                                                                                  system/libhwbinder/ProcessState.cpp:61
  0000d8bf  android::Thread::_threadLoop(void*)+186                                                                                                                                                                                                                                                                                                                         system/core/libutils/Threads.cpp:746
  000a6867  __pthread_start(void*)+20                                                                                                                                                                                                                                                                                                                                       bionic/libc/bionic/pthread_create.cpp:338
  00060087  __start_thread+30                                                                                                                                                                                                                                                                                                                                               bionic/libc/bionic/clone.cpp:53


//Stack Trace: process_capture_request()
//  RELADDR   FUNCTION                                                                                                                                                                                                                                                                                                                                                                                                                 FILE:LINE
  0006d6f3  android::camera2::Camera3HAL::process_capture_request(camera3_capture_request*)+86                                                                                                                                                                                                                                                                                                                                       hardware/rockchip/camera/AAL/Camera3HAL.cpp:323
  0006dd7d  android::camera2::hal_dev_process_capture_request(camera3_device const*, camera3_capture_request*)+92                                                                                                                                                                                                                                                                                                                    hardware/rockchip/camera/AAL/Camera3HAL.cpp:93
  0001b4bb  android::hardware::camera::device::V3_2::implementation::CameraDeviceSession::processOneCaptureRequest(android::hardware::camera::device::V3_2::CaptureRequest const&)+1286                                                                                                                                                                                                                                              hardware/interfaces/camera/device/3.2/default/CameraDeviceSession.cpp:1258
  0001af49  android::hardware::camera::device::V3_2::implementation::CameraDeviceSession::processCaptureRequest(android::hardware::hidl_vec<android::hardware::camera::device::V3_2::CaptureRequest> const&, android::hardware::hidl_vec<android::hardware::camera::device::V3_2::BufferCache> const&, std::__1::function<void (android::hardware::camera::common::V1_0::Status, unsigned int)>)+52                                  hardware/interfaces/camera/device/3.2/default/CameraDeviceSession.cpp:1146
  00004aa5  android::hardware::camera::device::V3_2::implementation::CameraDeviceSession::TrampolineSessionInterface_3_2::processCaptureRequest(android::hardware::hidl_vec<android::hardware::camera::device::V3_2::CaptureRequest> const&, android::hardware::hidl_vec<android::hardware::camera::device::V3_2::BufferCache> const&, std::__1::function<void (android::hardware::camera::common::V1_0::Status, unsigned int)>)+72  hardware/interfaces/camera/device/3.3/default/CameraDeviceSession.h:100
  0001f8b5  android::hardware::camera::device::V3_2::BnHwCameraDeviceSession::_hidl_processCaptureRequest(android::hidl::base::V1_0::BnHwBase*, android::hardware::Parcel const&, android::hardware::Parcel*, std::__1::function<void (android::hardware::Parcel&)>)+348                                                                                                                                                             out/soong/.intermediates/hardware/interfaces/camera/device/3.2/android.hardware.camera.device@3.2_genc++/gen/android/hardware/camera/device/3.2/CameraDeviceSessionAll.cpp:1055
  0000b699  android::hardware::camera::device::V3_3::BnHwCameraDeviceSession::onTransact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+464                                                                                                                                                                                        out/soong/.intermediates/hardware/interfaces/camera/device/3.3/android.hardware.camera.device@3.3_genc++/gen/android/hardware/camera/device/3.3/CameraDeviceSessionAll.cpp:523
  000689a1  android::hardware::BHwBinder::transact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+52                                                                                                                                                                                                                               system/libhwbinder/Binder.cpp:116
  v------>  android::hardware::IPCThreadState::executeCommand(int)                                                                                                                                                                                                                                                                                                                                                                   system/libhwbinder/IPCThreadState.cpp:1204
  0006af8d  android::hardware::IPCThreadState::getAndExecuteCommand()+924                                                                                                                                                                                                                                                                                                                                                            system/libhwbinder/IPCThreadState.cpp:461
  0006be61  android::hardware::IPCThreadState::joinThreadPool(bool)+56                                                                                                                                                                                                                                                                                                                                                               system/libhwbinder/IPCThreadState.cpp:561
  00076f0d  android::hardware::PoolThread::threadLoop()+12                                                                                                                                                                                                                                                                                                                                                                           system/libhwbinder/ProcessState.cpp:61
  0000d8bf  android::Thread::_threadLoop(void*)+186                                                                                                                                                                                                                                                                                                                                                                                  system/core/libutils/Threads.cpp:746
  000a6867  __pthread_start(void*)+20                                                                                                                                                                                                                                                                                                                                                                                                bionic/libc/bionic/pthread_create.cpp:338
  00060087  __start_thread+30                                                                                                                                                                                                                                                                                                                                                                                                        bionic/libc/bionic/clone.cpp:53


//Stack Trace: hal_dev_close()
//  RELADDR   FUNCTION                                                                                                                                                                                                                                      FILE:LINE
  000d6fdd  hal_dev_close(hw_device_t*)+180                                                                                                                                                                                                               hardware/rockchip/camera/Camera3HALModule.cpp:221
  00016ef5  android::hardware::camera::device::V3_2::implementation::CameraDeviceSession::close()+148                                                                                                                                                     hardware/interfaces/camera/device/3.2/default/CameraDeviceSession.cpp:1321
  00004bb3  android::hardware::camera::device::V3_2::implementation::CameraDeviceSession::TrampolineSessionInterface_3_2::close()+4                                                                                                                       hardware/interfaces/camera/device/3.3/default/CameraDeviceSession.h:118
  0002007f  android::hardware::camera::device::V3_2::BnHwCameraDeviceSession::_hidl_close(android::hidl::base::V1_0::BnHwBase*, android::hardware::Parcel const&, android::hardware::Parcel*, std::__1::function<void (android::hardware::Parcel&)>)+118  out/soong/.intermediates/hardware/interfaces/camera/device/3.2/android.hardware.camera.device@3.2_genc++/gen/android/hardware/camera/device/3.2/CameraDeviceSessionAll.cpp:1308
  0000b721  android::hardware::camera::device::V3_3::BnHwCameraDeviceSession::onTransact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+600             out/soong/.intermediates/hardware/interfaces/camera/device/3.3/android.hardware.camera.device@3.3_genc++/gen/android/hardware/camera/device/3.3/CameraDeviceSessionAll.cpp:567
  000689a1  android::hardware::BHwBinder::transact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+52                                                    system/libhwbinder/Binder.cpp:116
  v------>  android::hardware::IPCThreadState::executeCommand(int)                                                                                                                                                                                        system/libhwbinder/IPCThreadState.cpp:1204
  0006af8d  android::hardware::IPCThreadState::getAndExecuteCommand()+924                                                                                                                                                                                 system/libhwbinder/IPCThreadState.cpp:461
  0006be61  android::hardware::IPCThreadState::joinThreadPool(bool)+56                                                                                                                                                                                    system/libhwbinder/IPCThreadState.cpp:561
  v------>  int android::hardware::defaultPassthroughServiceImplementation<android::hardware::camera::provider::V2_4::ICameraProvider>(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, unsigned int)        system/libhidl/transport/include/hidl/LegacySupport.h:85
  000010b9  main+116                                                                                                                                                                                                                                      hardware/interfaces/camera/provider/2.4/default/service.cpp:49
  00059127  __libc_init+66                                                                                                                                                                                                                                bionic/libc/bionic/libc_init_dynamic.cpp:136
  0000102f  _start_main+38                                                                                                                                                                                                                                bionic/libc/arch-common/bionic/crtbegin.c:45
  00004456  <unknown>                                                                                                                                                                                                                                     <anonymous:e81ea000>

```



#### （5）、Camera Hal3 实现详细时序



![](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/Android.10.Camera.07/rk_camera_hal3_call_flow_detailed.png)

此图主要描绘了configure_streams流程，process_capture_request流程在Hal3中的具体实现逻辑，时序图以该两个API接口为起始点，直到hal3下发v4l2相关ioctl并返回相关数据结果为止。该流程图基本涵盖了Hal3中的主要模块上图中符号代表循环执行，也即表示此处有线程正在等待事件的到来。Hal层的运行也正是由这些事件驱动（即上面的红色键头

#### （6）、数据扭转核心

``` cpp
F:\Khadas_Edge_Android_Q\hardware\rockchip\camera\psl\rkisp1\workers\PostProcessPipeline.cpp
status_t
PostProcessUnitDigitalZoom::processFrame(const std::shared_ptr<PostProcBuffer>& in,
                              const std::shared_ptr<PostProcBuffer>& out,
                              const std::shared_ptr<ProcUnitSettings>& settings) {
    PERFORMANCE_ATRACE_CALL();
    CameraWindow& crop = settings->cropRegion;
    int jpegBufCount = settings->request->getBufferCountOfFormat(HAL_PIXEL_FORMAT_BLOB);

    bool mirror_handing = false;
#ifdef MIRROR_HANDLING_FOR_FRONT_CAMERA
    //for front camera mirror handling
    mirror_handing = PlatformData::facing(mPipeline->getCameraId()) == CAMERA_FACING_FRONT;
#endif
    LOGD("@%s : mirror handleing %d", __FUNCTION__, mirror_handing);

    // check if zoom is required
    if (mBufType != kPostProcBufTypeExt &&
        crop.width() ==  mApa.width() && crop.height() == mApa.height()) {
        // HwJpeg encode require buffer width and height align to 16 or large enough.
        // digital zoom out buffer is internal gralloc buffer with size 2xWxH, so it
        // can always meet the Hwjpeg input condition. we use it as a workaround
        // for capture case
        if(jpegBufCount != 0) {
            LOGD("@%s : Use digital zoom out gralloc buffer as hwjpeg input buffer", __FUNCTION__);
        } else if(mirror_handing) {
            LOGD("@%s : use digitalZoom do mirror for front camera", __FUNCTION__);
        } else {
            return STATUS_FORWRAD_TO_NEXT_UNIT;
        }
    }

    if (!checkFmt(in->cambuf.get(), out->cambuf.get())) {
        LOGE("%s: unsupported format, only support NV12 or NV21 now !", __FUNCTION__);
        return UNKNOWN_ERROR;
    }
    // map crop window to in-buffer crop window
    int mapleft, maptop, mapwidth, mapheight;
    float wratio = (float)crop.width() / mApa.width();
    float hratio = (float)crop.height() / mApa.height();
    float hoffratio = (float)crop.left() / mApa.width();
    float voffratio = (float)crop.top() / mApa.height();

    mapleft = in->cambuf->width() * hoffratio;
    maptop = in->cambuf->height() * voffratio;
    mapwidth = in->cambuf->width() * wratio;
    mapheight = in->cambuf->height() * hratio;
    // should align to 2
    mapleft &= ~0x1;
    maptop &= ~0x1;
    mapwidth &= ~0x3;
    mapheight &= ~0x3;
    // do digital zoom
    LOGD("%s: crop region(%d,%d,%d,%d) from (%d,%d), infmt %d,%d, outfmt %d,%d",
         __FUNCTION__, mapleft, maptop, mapwidth, mapheight,
         in->cambuf->width(), in->cambuf->height(),
         in->cambuf->format(),
         in->cambuf->v4l2Fmt(),
         out->cambuf->format(),
         out->cambuf->v4l2Fmt());
    // try RGA firstly
    RgaCropScale::Params rgain, rgaout;

    rgain.fd = in->cambuf->dmaBufFd();
    if (in->cambuf->format() == HAL_PIXEL_FORMAT_YCrCb_NV12 ||
        in->cambuf->format() == HAL_PIXEL_FORMAT_YCbCr_420_888 ||
        in->cambuf->format() == HAL_PIXEL_FORMAT_IMPLEMENTATION_DEFINED ||
        in->cambuf->v4l2Fmt() == V4L2_PIX_FMT_NV12)
        rgain.fmt = HAL_PIXEL_FORMAT_YCrCb_NV12;
    else
        rgain.fmt = HAL_PIXEL_FORMAT_YCrCb_420_SP;
    rgain.vir_addr = (char*)in->cambuf->data();
    rgain.mirror = mirror_handing;
    rgain.width = mapwidth;
    rgain.height = mapheight;
    rgain.offset_x = mapleft;
    rgain.offset_y = maptop;
    rgain.width_stride = in->cambuf->width();
    rgain.height_stride = in->cambuf->height();

    rgaout.fd = out->cambuf->dmaBufFd();
    if (out->cambuf->format() == HAL_PIXEL_FORMAT_YCrCb_NV12 ||
        out->cambuf->format() == HAL_PIXEL_FORMAT_YCbCr_420_888 ||
        out->cambuf->format() == HAL_PIXEL_FORMAT_IMPLEMENTATION_DEFINED ||
        out->cambuf->v4l2Fmt() == V4L2_PIX_FMT_NV12)
        rgaout.fmt = HAL_PIXEL_FORMAT_YCrCb_NV12;
    else
        rgaout.fmt = HAL_PIXEL_FORMAT_YCrCb_420_SP;
    rgaout.vir_addr = (char*)out->cambuf->data();
    rgaout.mirror = false;
    rgaout.width = out->cambuf->width();
    rgaout.height = out->cambuf->height();
    rgaout.offset_x = 0;
    rgaout.offset_y = 0;
    rgaout.width_stride = out->cambuf->width();
    rgaout.height_stride = out->cambuf->height();

	LOGI("@%s, in, mHandle:%p", __FUNCTION__, *( in->cambuf->getBufferHandle()));
	LOGI("@%s, out, mHandle:%p", __FUNCTION__, *( out->cambuf->getBufferHandle()));

    /*if (RgaCropScale::CropScaleNV12Or21(&rgain, &rgaout)) {
        LOGW("%s: digital zoom by RGA failed, use arm instead...", __FUNCTION__);
        PERFORMANCE_ATRACE_NAME("SWCropScale");
        ImageScalerCore::cropComposeUpscaleNV12_bl(
                         in->cambuf->data(), in->cambuf->height(), in->cambuf->width(),
                         mapleft, maptop, mapwidth, mapheight,
                         out->cambuf->data(), out->cambuf->height(), out->cambuf->width(),
                         0, 0, out->cambuf->width(), out->cambuf->height());
    }*/

    return OK;
}
```
将如下代码注释，就不会显示Camera图像了，绿屏.

```cpp
/*if (RgaCropScale::CropScaleNV12Or21(&rgain, &rgaout)) {
    LOGW("%s: digital zoom by RGA failed, use arm instead...", __FUNCTION__);
    PERFORMANCE_ATRACE_NAME("SWCropScale");
    ImageScalerCore::cropComposeUpscaleNV12_bl(
                     in->cambuf->data(), in->cambuf->height(), in->cambuf->width(),
                     mapleft, maptop, mapwidth, mapheight,
                     out->cambuf->data(), out->cambuf->height(), out->cambuf->width(),
                     0, 0, out->cambuf->width(), out->cambuf->height());
}*/
```

![](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/Android.10.Camera.07/IMG_20210701_121858.jpg)

反之得到正常图像.

![](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/Android.10.Camera.07/IMG_20210701_121747.jpg)



## （二）、Dump说明

#### （1）、属性说明

persist.vendor.camera.dump：表示相关数据的dump 开关, 属性值对应不同数据流 
例：

```bash
adb shell setprop persist.vendor.camera.dump 1  #dump preview
adb shell setprop persist.vendor.camera.dump 2  #dump video
adb shell setprop persist.vendor.camera.dump 4  #dump zsl
adb shell setprop persist.vendor.camera.dump 8  #dump jpeg
adb shell setprop persist.vendor.camera.dump 16 #dump raw
```


// 上述raw 并非从sensor直接出来的数据，而是从isp拿到的数据，还未在Hal层处理过

```bash
adb shell setprop persist.vendor.camera.dump 11 #dump preview + video + jpeg
```

- persist.vendor.camera.dump.skip  属性表示跳过前面n帧 

```bash
例：$adb shell setprop persist.vendor.camera.dump.skip 10
```

表示前面10帧不dump

- persist.vendor.camera.dump.interval 属性是表示dump 帧的间隔 

```bash
例：$adb shell setprop persist.vendor.camera.dump.interval 0
```

表示隔10帧dump 一帧

- persist.vendor.camera.dump.count 属性表示dump 帧的总帧数 
```bash
例：$adb shell setprop persist.vendor.camera.dump.count 100
```

表示总共只dump 100帧

- persist.vendor.camera.dump.path  属性表示dump 帧的路径 

```bash
例：$adb shell setprop persist.vendor.camera.dump.path /data/dump/
```

表示dump的路径为/data/dump/   (最后的”/” 不能省)

以下是一个完整例子，表示dump 预览帧，前面10帧不dump, 每隔10帧dump 一次，总共dump 100帧，路径为/data/dump/

```bash
adb shell setprop persist.vendor.camera.dump 1
adb shell setprop persist.vendor.camera.dump.skip 10
adb shell setprop persist.vendor.camera.dump.interval 10
adb shell setprop persist.vendor.camera.dump.count 100
adb shellsetprop persist.vendor.camera.dump.path /data/
```



#### （2）、未生成dump文件问题

```bash
我设置的路径：rk3399_Android10:/data/misc/cameraserver #
```

![](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/Android.10.Camera.07/rk_cam_dump_path.png)



## （三）、版本说明

1. 通过读取属性值获取

``` cpp
$adb shell 
$getprop | grep cam.hal3.ver
```

2. 也可以通过查看logcat 获取

``` cpp
$adb shell
$logcat | grep "Hal3 Release version"
```

#### （1）、Hal3版本获取

``` cpp
F:\>adb shell
rk3399_Android10:/ #  getprop | grep cam.hal3.ver
[vendor.cam.hal3.ver]: [v2.1.0]
```

