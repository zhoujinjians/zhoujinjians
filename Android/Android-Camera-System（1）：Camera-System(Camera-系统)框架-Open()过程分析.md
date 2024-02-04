---
title: Android Camera System（1）：Camera 系统 框架、Open()过程分析
cover: https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/hexo.themes/bing-wallpaper-2018.04.18.jpg
categories:
  - Camera
tags:
  - Android
  - Linux
  - Camera
toc: true
abbrlink: 20190101
date: 2019-01-01 09:25:00
---

--------------------------------------------------------------------------------

注：文章都是通过阅读各位前辈总结的资料、Android 7.1.2 && Linux（kernel 3.18）Qualcomm平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除，禁止转载（©Qualcomm Technologies, Inc. 版权所有），谢谢。

[【特别感谢 - Android Camera fw学习-Armwind】](https://blog.csdn.net/armwind/article/category/6282972)
[【特别感谢 - Android Camera API2分析-Gzzaigcnforever】](https://blog.csdn.net/gzzaigcnforever/article/category/3066721)
[【特别感谢 - Android Camera 流程学习记录 Android 7.12-QQ_16775897】](https://blog.csdn.net/qq_16775897/article/category/7112759)
[【特别感谢 - 专栏：古冥的android6.0下的Camera API2.0的源码分析之旅】](https://blog.csdn.net/column/details/guming-camera.html)
Google Pixel、Pixel XL 内核代码（文章基于 Kernel-3.18）：
 [Kernel source for Pixel and Pixel XL - GitHub](https://github.com/matthewdalex/marlin)

AOSP 源码（文章基于 Android 7.1.2）：
 [ Android 系统全套源代码分享 (更新到 8.1.0_r1)](https://testerhome.com/topics/2229)

 🌀🌀：专注于Linux && Android Multimedia（Camera、Video、Audio、Display）系统分析与研究

--------------------------------------------------------------------------------
☯ Application：
☯ /packages/apps/Camera2/src/com/android/camera/

☯ Framework： 
☯ /frameworks/base/core/java/android/hardware/Camera.java

☯ JNI:
☯ /frameworks/base/core/jni/android_hardware_Camera.cpp

☯ Native:
☯ Client： 
frameworks/av/camera/CameraBase.cpp
frameworks/av/camera/Camera.cpp
frameworks/av/camera/ICamera.cpp
frameworks/av/camera/aidl/android/hardware/ICamera.aidl
frameworks/av/camera/aidl/android/hardware/ICameraClient.aidl
☯ Server： 
frameworks/av/camera/cameraserver/main_cameraserver.cpp
frameworks/av/services/camera/libcameraservice/CameraService.cpp
frameworks/av/services/camera/libcameraservice/api1/CameraClient.cpp
frameworks/av/camera/aidl/android/hardware/ICameraService.aidl

☯ HAL： 
☯ /frameworks/av/services/camera/libcameraservice/device3/
☯ /hardware/qcom/camera/QCamera2(高通HAL)
☯ /vendor/qcom/proprietary/mm-camera(高通mm-camera)
☯ /vendor/qcom/proprietary/mm-still(高通JPEG)

☯ Kernel： 
☯ /kernel/drivers/media/platform/msm/camera_v2(高通V4L2)
☯ /kernel/arch/arm/boot/dts/(高通dts)


--------------------------------------------------------------------------------
#### （一）、Android Camera System Architecture（Camera系统框架）
##### 1.1、Android Camera System总体框架（Qualcomm平台）
##### 1.1.1、首先看看Android 官方Camera总体架构：

![Alt text](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/camera.system/01-01-android_ape_fwk_camera.png)

**☯应用框架**
应用代码位于应用框架级别，它利用 android.hardware.Camera API 来与相机硬件进行互动。在内部，此代码会调用相应的 JNI 粘合类，以访问与该相机互动的原生代码。
**☯JNI**
与 android.hardware.Camera 关联的 JNI 代码位于 frameworks/base/core/jni/android_hardware_Camera.cpp 中。此代码会调用较低级别的原生代码以获取对物理相机的访问权限，并返回用于在框架级别创建 android.hardware.Camera 对象的数据。
**☯原生框架**
在 frameworks/av/camera/Camera.cpp 中定义的原生框架可提供相当于 android.hardware.Camera 类的原生类。此类会调用 IPC binder 代理，以获取对相机服务的访问权限。
**☯Binder IPC 代理**
IPC binder 代理用于促进跨越进程边界的通信。调用相机服务的 frameworks/av/camera 目录中有 3 个相机 binder 类。ICameraService 是相机服务的接口，ICamera 是已打开的特定相机设备的接口，ICameraClient 是返回应用框架的设备接口。
**☯相机服务**
位于 frameworks/av/services/camera/libcameraservice/CameraService.cpp 下的相机服务是与 HAL 进行互动的实际代码。
**☯HAL**
硬件抽象层定义了由相机服务调用且您必须实现以确保相机硬件正常运行的标准接口。
**☯内核驱动程序**
相机的驱动程序可与实际相机硬件以及您的 HAL 实现进行互动。相机和驱动程序必须支持 YV12 和 NV21 图片格式，以便在显示和视频录制时支持预览相机图片。
##### 1.1.2、Qualcomm平台Camera 架构
Qualcomm平台Camera 架构主要区别在于HAL层和Kernel层的变化，总体架构图如下：
##### 1.1.2.1、Qualcomm平台Camera总体架构

![Alt text](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/camera.system/01-02-Android-Camera-Software-Architecture.png)

##### 1.1.2.2、Qualcomm平台Camera的HAL、mm-camera

![Alt text](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/camera.system/01-03-HAL-and-mm-camera-interface.png)

##### 1.1.2.3、Qualcomm平台Camera的Kernel

![Alt text](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/camera.system/01-04-Camera-Kernel-Architecture.png)

##### 1.2、Android Camera API 2.0 全新的HAL 子系统
Android 7.1.2现在使用的是Camera API 2.0 和 Camera Device 3以及 HAL3。
##### 1.2.1、请求
应用框架针对捕获的结果向相机子系统发出请求。一个请求对应一组结果。请求包含有关捕获和处理这些结果的所有配置信息。其中包括分辨率和像素格式；手动传感器、镜头和闪光灯控件；3A 操作模式；RAW 到 YUV 处理控件；以及统计信息的生成。这样一来，便可更好地控制结果的输出和处理。一次可发起多个请求，而且提交的请求不会出现阻塞的情况。请求始终按照接收的顺序进行处理。

![Alt text](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/camera.system/01-05-camera2api.png.png)

##### 1.2.2、HAL 和相机子系统
相机子系统包括相机管道中组件的实现，例如 3A 算法和处理控件。相机 HAL 为您提供了实现您版本的这些组件所需的接口。为了保持多个设备制造商和图像信号处理器（ISP，也称为相机传感器）供应商之间的跨平台兼容性，相机管道模型是虚拟的，且不直接对应任何真正的 ISP。不过，它与真正的处理管道足够相似，因此您可以有效地将其映射到硬件。此外，它足够抽象，可支持多种不同的算法和操作顺序，而不会影响质量、效率或跨设备兼容性。
相机管道还支持应用框架开启自动对焦等功能的触发器。它还会将通知发送回应用框架，以通知应用自动对焦锁定或错误等事件。

![Alt text](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/camera.system/01-06-camera_hal_request_control.png.png)


> RAW Bayer 输出在 ISP 内部不经过任何处理。
统计信息根据原始传感器数据生成。
将原始传感器数据转换为 YUV 的各种处理块按任意顺序排列。
当显示多个刻度和剪裁单元时，所有的缩放器单元共享输出区域控件（数字缩放）。不过，每个单元都可能具有不同的输出分辨率和像素格式。

##### 1.2.3、HAL 操作摘要

☯ 捕获的异步请求来自于框架。
☯ HAL 设备必须按顺序处理请求。对于每个请求，均产生输出结果元数据以及一个或多个输出图片缓冲区。
☯ 请求和结果以及后续请求引用的流遵守先进先出规则。
☯ 指定请求的所有输出的时间戳必须完全相同，以便框架可以根据需要将它们匹配在一起。
☯ 所有捕获配置和状态（不包括 3A 例程）都包含在请求和结果中。

![Alt text](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/camera.system/01-07-camera-hal-overview-oo.png.png)

##### 1.2.4、启动和预期操作顺序
1、框架调用 camera_module_t->common.open()，而这会返回一个 hardware_device_t 结构。
2、框架检查 hardware_device_t->version 字段，并为该版本的相机硬件设备实例化相应的处理程序。如果版本是 CAMERA_DEVICE_API_VERSION_3_0，则该设备会转型为 camera3_device_t。
3、框架调用 camera3_device_t->ops->initialize() 并显示框架回调函数指针。在调用 ops 结构中的任何其他函数之前，这只会在 open() 之后调用一次。
4、框架调用 camera3_device_t->ops->configure_streams() 并显示到 HAL 设备的输入/输出流列表。
5、框架为 configure_streams 中列出的至少一个输出流分配 gralloc 缓冲区并调用 camera3_device_t->ops->register_stream_buffers()。相同的流仅注册一次。
6、框架通过调用 camera3_device_t->ops->construct_default_request_settings() 来为某些使用情形请求默认设置。这可能会在第 3 步之后的任何时间发生。
7、框架通过基于其中一组默认设置的设置以及至少一个框架之前注册的输出流来构建第一个捕获请求并将其发送到 HAL。它通过 camera3_device_t->ops->process_capture_request() 发送到 HAL。HAL 必须阻止此调用返回，直到准备好发送下一个请求。
8、框架继续提交请求，并且可能会为尚未注册的流调用 register_stream_buffers()，并调用 construct_default_request_settings 来为其他使用情形获取默认设置缓冲区。
9、当请求捕获开始（传感器开始曝光以进行捕获）时，HAL 会调用 camera3_callback_ops_t->notify() 并显示 SHUTTER 事件，包括帧号和开始曝光的时间戳。此通知调用必须在第一次调用该帧号的 process_capture_result() 之前进行。
10、在某个管道延迟后，HAL 开始使用 camera3_callback_ops_t->process_capture_result() 将完成的捕获返回到框架。这些捕获按照与提交请求相同的顺序返回。一次可发起多个请求，具体取决于相机 HAL 设备的管道深度。
11、一段时间后，框架可能会停止提交新的请求、等待现有捕获完成（所有缓冲区都已填充，所有结果都已返回），然后再次调用 configure_streams()。这会重置相机硬件和管道，以获得一组新的输入/输出流。可重复使用先前配置中的部分流；如果这些流的缓冲区已经过 HAL 注册，则不会再次注册。如果至少还有一个已注册的输出流，则框架从第 7 步继续（否则，需要先完成第 5 步）。
12、或者，框架可能会调用 camera3_device_t->common->close() 以结束相机会话。当框架中没有其他处于活动状态的调用时，它可能随时会被调用；尽管在所有发起的捕获完成（所有结果都已返回，所有缓冲区都已填充）之前，调用可能会阻塞。在 close 调用返回后，不允许再从 HAL 对 camera3_callback_ops_t 函数进行更多调用。一旦进行 close() 调用，该框架可能不会调用任何其他 HAL 设备函数。
13、在发生错误或其他异步事件时，HAL 必须调用 camera3_callback_ops_t->notify() 并返回相应的错误/事件消息。从严重的设备范围错误通知返回后，HAL 应表现为在其上调用了 close()。但是，HAL 必须在调用 notify() 之前取消或完成所有待处理的捕获，以便在调用 notify() 并返回严重错误时，框架不会收到来自设备的更多回调。在严重的错误消息返回 notify() 方法后，close() 之外的方法应该返回 -ENODEV 或 NULL。
##### 1.3、Android Graphics 学习－生产者、消费者、BufferQueue介绍

![Alt text](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/camera.system/01-08-Android-graphics-SurfaceFlinger-BufferQueue.jpg.png)

Graphics 系统详细分析请参考：【Android 7.1.2 (Android N) Android Graphics 系统分析】()

##### 1.4、Camera类之间的关系和作用
##### 1.4.1、Camera类关系总体概览

![Alt text](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/camera.system/01-09-Android-Camera-class.png)

☯ 1、ICameraClient: 这主要是一些消息发送的接口，包括帧可用通知，回调一些信息给client等消息。不过这里要注意的是，BnCameraClient对象其实是在client这端，不在CameraService端。
☯ 2、ICamera:camera的一些标准操作接口，比如startpreview，takepicuture,autofocus,所有的操作动作都是用的这一套接口。
☯ 3、ICameraService: 链接Camera服务，Camera device,获取Camera数量，Camera硬件信息，视厂角，镜头等信息。

##### 1.4.2、ICameraClient

![Alt text](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/camera.system/01-10-ICameraClient.png)

##### 1.4.3、ICamera

![Alt text](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/camera.system/01-11-ICamera.png)

##### 1.4.4、ICameraService

![Alt text](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/camera.system/01-12-ICameraService.png)

#### （二）、Android CameraService开机初始化分析
首先看下总体时序图：

![Alt text](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/camera.system/01-13-CameraService_onFirstRef.png)

##### 2.1、CameraService 初始化过程
Android启动的时候会收集系统的.rc文件，启动对应的Native Service：

``` cpp
[->\frameworks\av\camera\cameraserver\cameraserver.rc]
service cameraserver /system/bin/cameraserver
    class main
    user cameraserver
    group audio camera input drmrpc
    ioprio rt 4
    writepid /dev/cpuset/camera-daemon/tasks /dev/stune/top-app/tasks

```

``` cpp
[->\frameworks\av\camera\cameraserver\main_cameraserver.cpp]
int main(int argc __unused, char** argv __unused)
{
    signal(SIGPIPE, SIG_IGN);
    sp<ProcessState> proc(ProcessState::self());
    sp<IServiceManager> sm = defaultServiceManager();
    ALOGI("ServiceManager: %p", sm.get());
    CameraService::instantiate();
    ProcessState::self()->startThreadPool();
    IPCThreadState::self()->joinThreadPool();
}
```
CameraService继承自BinderService，instantiate也是在BinderService中定义的，此方法就是调用publish方法，所以来看publish方法：

``` cpp
[->\frameworks\native\include\binder\BinderService.h]
static status_t publish(bool allowIsolated = false) {
    sp<IServiceManager> sm(defaultServiceManager());
    //将服务添加到ServiceManager
    return sm->addService(String16(SERVICE::getServiceName()),new SERVICE(), allowIsolated);
}
```

这里，将会把CameraService服务加入到ServiceManager进行管理。 CameraService的构造时，会调用CameraService的onFirstRef方法：
##### 2.1.1、CameraService::onFirstRef()
``` cpp
void CameraService::onFirstRef()
{
    ALOGI("CameraService process starting");

    BnCameraService::onFirstRef();

    // Update battery life tracking if service is restarting
    BatteryNotifier& notifier(BatteryNotifier::getInstance());
    notifier.noteResetCamera();
    notifier.noteResetFlashlight();

    camera_module_t *rawModule;
    int err = hw_get_module(CAMERA_HARDWARE_MODULE_ID,
            (const hw_module_t **)&rawModule);
    ......

    mModule = new CameraModule(rawModule);
    err = mModule->init();
    ......
    mFlashlight = new CameraFlashlight(*mModule, *this);
    status_t res = mFlashlight->findFlashUnits();
    ......
    if (mModule->getModuleApiVersion() >= CAMERA_MODULE_API_VERSION_2_1) {
        mModule->setCallbacks(this);
    }

    CameraService::pingCameraServiceProxy();
}
```
首先会通过HAL框架的hw_get_module来创建CameraModule对象，然后会对其进行相应的初始化，并会进行一些参数的设置，如camera的数量，闪光灯的初始化，以及回调函数的设置等，到这里，Camera2 HAL的模块就初始化结束了。
##### 2.1.2、Camera 动态库加载过程
在源码中不知大家有没有注意到第二个参数是hw_module_t **module,这里是指针的指针，而我们刚才传的是camera_module_t**指针。大家可以看到camera_module_t 结构第一个域就是hw_module_t 所以这里就不难理解了。

``` cpp
源码路径：hardware/libhardware/hardware.c

/** Base path of the hal modules */
#define HAL_LIBRARY_PATH1 "/system/lib/hw"  
#define HAL_LIBRARY_PATH2 "/vendor/lib/hw"

int hw_get_module(const char *id, const struct hw_module_t **module)
{
    return hw_get_module_by_class(id, NULL, module); 
    //这里的id就是camera模块的id，每一个hal module都有对应的id，
    //区分他们就通过这个id来区分了。
 }
int hw_get_module_by_class(const char *class_id, const char *inst,
                           const struct hw_module_t **module)
{
    int status;
    int i;
    const struct hw_module_t *hmi = NULL;
    char prop[PATH_MAX];
    char path[PATH_MAX];
    char name[PATH_MAX];

    ......
    /* Loop through the configuration variants looking for a module */
    for (i=0 ; i<HAL_VARIANT_KEYS_COUNT+1 ; i++) {
        if (i < HAL_VARIANT_KEYS_COUNT) {
            if (property_get(variant_keys[i], prop, NULL) == 0) { //关键字数组，上面有宏代码。
                continue;
            }
            snprintf(path, sizeof(path), "%s/%s.%s.so",
                     HAL_LIBRARY_PATH2, name, prop);
            if (access(path, R_OK) == 0) break;

            snprintf(path, sizeof(path), "%s/%s.%s.so", //拼接完整的camera库。
                     HAL_LIBRARY_PATH1, name, prop);
            if (access(path, R_OK) == 0) break;
        } else {
            snprintf(path, sizeof(path), "%s/%s.default.so",
                     HAL_LIBRARY_PATH2, name);
            if (access(path, R_OK) == 0) break;

            snprintf(path, sizeof(path), "%s/%s.default.so",
                     HAL_LIBRARY_PATH1, name);
            if (access(path, R_OK) == 0) break;
        }
    }

    status = -ENOENT;
    if (i < HAL_VARIANT_KEYS_COUNT+1) {
        status = load(class_id, path, module); //如果上面都进行完毕，走到这里，说明已经找到库了，这里就去加载。
    }

    return status;
}
//根据id来加载hal的module
static int load(const char *id,
        const char *path,
        const struct hw_module_t **pHmi)
{
    int status;
    void *handle;
    struct hw_module_t *hmi;
    ......
    handle = dlopen(path, RTLD_NOW); //动态加载内存的api，这里的path=/system/lib/hw/camera.msm8996.so
    ......
    /* Get the address of the struct hal_module_info. */
    const char *sym = HAL_MODULE_INFO_SYM_AS_STR;   //别的地方定义#define HAL_MODULE_INFO_SYM_AS_STR  "HMI"
    hmi = (struct hw_module_t *)dlsym(handle, sym); //我们动态链接的是"HMI"这个符号。
    ......
    *pHmi = hmi; //最后将这个指针，赋给我们之前定义的 struct camera_module变量。这里模块就加载进来了。
    return status;
}

//hal代码
[->\hardware\qcom\camera\QCamera2\QCamera2Hal.cpp]
static hw_module_t camera_common = {
    .tag                    = HARDWARE_MODULE_TAG,
    .module_api_version     = CAMERA_MODULE_API_VERSION_2_4,
    .hal_api_version        = HARDWARE_HAL_API_VERSION,
    .id                     = CAMERA_HARDWARE_MODULE_ID,
    .name                   = "QCamera Module",
    .author                 = "Qualcomm Innovation Center Inc",
    .methods                = &qcamera::QCamera2Factory::mModuleMethods, //它的方法数组里绑定了open接口
    .dso                    = NULL,
    .reserved               = {0}
};

camera_module_t HAL_MODULE_INFO_SYM = {
    .common                 = camera_common,
    .get_number_of_cameras  = qcamera::QCamera2Factory::get_number_of_cameras,
    .get_camera_info        = qcamera::QCamera2Factory::get_camera_info,
    .set_callbacks          = qcamera::QCamera2Factory::set_callbacks,
    .get_vendor_tag_ops     = qcamera::QCamera3VendorTags::get_vendor_tag_ops,
    .open_legacy            = qcamera::QCamera2Factory::open_legacy,
    .set_torch_mode         = qcamera::QCamera2Factory::set_torch_mode,
    .init                   = NULL,
    .reserved               = {0}
};
struct hw_module_methods_t QCamera2Factory::mModuleMethods = {
    //open方法的绑定
    open: QCamera2Factory::camera_device_open,
};

```
Camera HAL层的open入口其实就是camera_device_open方法：
##### 2.1.3、图解camera_module和camera_device_t关系
camer module在系统中转指camera模块，camera_device_t 转指某一个camera 设备。在流程上，native framwork 先加载在hal层定义的camer_module对象，然后通过camera_module的methods open方法填充camera_device_t 结构体，并最终获取到camera ops这一整个camera最重要的操作集合。下图中我们可以看到struct hw_module_t在camera_module最上面 而camera_device_t最开始保存的是struct hw_device_t. 由此我们平时在看代码时，要注意一些指针转换。

![Alt text](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/camera.system/01-14-camera_module camera_device_t.png)

#### （三）、Android Camera Open过程
##### 3.1、Camera2 HAL层Open()过程分析

高通的Camera，它在后台会有一个守护进程daemon，daemon是介于应用和驱动之间翻译ioctl的中间层(委托处理)。本节将以Camera中的open流程为例，来分析Camera HAL的工作过程，在应用对硬件发出open请求后，会通过Camera HAL来发起open请求，而Camera HAL的open入口在QCamera2Hal.cpp进行了定义，即前面分析的Camera HAL层的open入口其实就是camera_device_open方法：

``` cpp
[->\hardware\qcom\camera\QCamera2\QCamera2Factory.cpp]
int QCamera2Factory::camera_device_open(const struct hw_module_t *module, const char *id,
        struct hw_device_t **hw_device){
    ...
    return gQCamera2Factory->cameraDeviceOpen(atoi(id), hw_device);
}
```
它调用了cameraDeviceOpen方法，而其中的hw_device就是最后要返回给应用层的CameraDeviceImpl在Camera HAL层的对象，继续分析cameraDeviceOpen方法：

``` cpp
[->\hardware\qcom\camera\QCamera2\QCamera2Factory.cpp]
int QCamera2Factory::cameraDeviceOpen(int camera_id, struct hw_device_t **hw_device){
    ...
    //Camera2采用的Camera HAL版本为HAL3.0
    if ( mHalDescriptors[camera_id].device_version == CAMERA_DEVICE_API_VERSION_3_0 ) {
        //初始化QCamera3HardwareInterface对象，这里构造函数里将会进行configure_streams以及
        //process_capture_result等的绑定
        QCamera3HardwareInterface *hw = new QCamera3HardwareInterface(
            mHalDescriptors[camera_id].cameraId, mCallbacks);
        //通过QCamera3HardwareInterface来打开Camera
        rc = hw->openCamera(hw_device);
        ...
    } else if (mHalDescriptors[camera_id].device_version == CAMERA_DEVICE_API_VERSION_1_0) {
        //HAL API为2.0
        QCamera2HardwareInterface *hw = new QCamera2HardwareInterface((uint32_t)camera_id);
        rc = hw->openCamera(hw_device);
        ...
    } else {
        ...
    }
    return rc;
}
```

此方法有两个关键点：一个是QCamera3HardwareInterface对象的创建，它是用户空间与内核空间进行交互的接口；另一个是调用它的openCamera方法来打开Camera，下面将分别进行分析。
##### 3.1.1、QCamera3HardwareInterface构造函数分析
在它的构造函数里面有一个关键的初始化，即mCameraDevice.ops = &mCameraOps，它会定义Device操作的接口：

``` cpp
[->\frameworks\base\core\java\android\hardware\camera2\impl\CameraDeviceImpl.java]
camera3_device_ops_t QCamera3HardwareInterface::mCameraOps = {
    initialize:                         QCamera3HardwareInterface::initialize,
    //配置流数据的相关处理
    configure_streams:                  QCamera3HardwareInterface::configure_streams,
    register_stream_buffers:            NULL,
    construct_default_request_settings: 
        QCamera3HardwareInterface::construct_default_request_settings,
    //处理结果的接口
    process_capture_request:            
        QCamera3HardwareInterface::process_capture_request,
    get_metadata_vendor_tag_ops:        NULL,
    dump:                               QCamera3HardwareInterface::dump,
    flush:                              QCamera3HardwareInterface::flush,
    reserved:                           {0},
};  
```
其中，会在configure_streams中配置好流的处理handle：

``` cpp
[->\frameworks\base\core\java\android\hardware\camera2\impl\CameraDeviceImpl.java]
int QCamera3HardwareInterface::configure_streams(const struct camera3_device *device,
        camera3_stream_configuration_t *stream_list){
    //获得QCamera3HardwareInterface对象
    QCamera3HardwareInterface *hw =reinterpret_cast<QCamera3HardwareInterface *>(device->priv);
    ...
    //调用它的configureStreams进行配置
    int rc = hw->configureStreams(stream_list);
    ..
    return rc;
}
```
继续追踪configureStream方法：


``` cpp
[->\frameworks\base\core\java\android\hardware\camera2\impl\CameraDeviceImpl.java]
int QCamera3HardwareInterface::configureStreams(camera3_stream_configuration_t *streamList){
    ...
    //初始化Camera版本
    al_version = CAM_HAL_V3;
    ...
    //开始配置stream
    ...
    //初始化相关Channel为NULL
    if (mMetadataChannel) {
        delete mMetadataChannel;
        mMetadataChannel = NULL;
    }
    if (mSupportChannel) {
        delete mSupportChannel;
        mSupportChannel = NULL;
    }

    if (mAnalysisChannel) {
        delete mAnalysisChannel;
        mAnalysisChannel = NULL;
    }

    //创建Metadata Channel，并对其进行初始化
    mMetadataChannel = new QCamera3MetadataChannel(mCameraHandle->camera_handle,
        mCameraHandle->ops, captureResultCb,&gCamCapability[mCameraId]->padding_info, 
        CAM_QCOM_FEATURE_NONE, this);
    ...
    //初始化
    rc = mMetadataChannel->initialize(IS_TYPE_NONE);
    ...
    //如果h/w support可用，则创建分析stream的Channel
    if (gCamCapability[mCameraId]->hw_analysis_supported) {
        mAnalysisChannel = new QCamera3SupportChannel(mCameraHandle->camera_handle,
                mCameraHandle->ops,&gCamCapability[mCameraId]->padding_info,
                CAM_QCOM_FEATURE_PP_SUPERSET_HAL3,CAM_STREAM_TYPE_ANALYSIS,
                &gCamCapability[mCameraId]->analysis_recommended_res,this);
        ...
    }

    bool isRawStreamRequested = false;
    //清空stream配置信息
    memset(&mStreamConfigInfo, 0, sizeof(cam_stream_size_info_t));
    //为requested stream分配相关的channel对象
    for (size_t i = 0; i < streamList->num_streams; i++) {
        camera3_stream_t *newStream = streamList->streams[i];
        uint32_t stream_usage = newStream->usage;
        mStreamConfigInfo.stream_sizes[mStreamConfigInfo.num_streams].width = (int32_t)newStream-
                >width;
        mStreamConfigInfo.stream_sizes[mStreamConfigInfo.num_streams].height = (int32_t)newStream-
                >height;
        if ((newStream->stream_type == CAMERA3_STREAM_BIDIRECTIONAL||newStream->usage & 
                GRALLOC_USAGE_HW_CAMERA_ZSL) &&newStream->format == 
                HAL_PIXEL_FORMAT_IMPLEMENTATION_DEFINED && jpegStream){
            mStreamConfigInfo.type[mStreamConfigInfo.num_streams] = CAM_STREAM_TYPE_SNAPSHOT;
            mStreamConfigInfo.postprocess_mask[mStreamConfigInfo.num_streams] = 
                CAM_QCOM_FEATURE_NONE;
        } else if(newStream->stream_type == CAMERA3_STREAM_INPUT) {
        } else {
            switch (newStream->format) {
                //为非zsl streams查找他们的format
                ...
            }
        }
        if (newStream->priv == NULL) {
            //为新的stream构造Channel
            switch (newStream->stream_type) {//分类型构造
            case CAMERA3_STREAM_INPUT:
                newStream->usage |= GRALLOC_USAGE_HW_CAMERA_READ;
                newStream->usage |= GRALLOC_USAGE_HW_CAMERA_WRITE;//WR for inplace algo's
                break;
            case CAMERA3_STREAM_BIDIRECTIONAL:
                ...
                break;
            case CAMERA3_STREAM_OUTPUT:
                ...
                break;
            default:
                break;
            }
            //根据前面的得到的stream的参数类型以及format分别对各类型的channel进行构造
            if (newStream->stream_type == CAMERA3_STREAM_OUTPUT ||
                    newStream->stream_type == CAMERA3_STREAM_BIDIRECTIONAL) {
                QCamera3Channel *channel = NULL;
                switch (newStream->format) {
                case HAL_PIXEL_FORMAT_IMPLEMENTATION_DEFINED:
                    /* use higher number of buffers for HFR mode */
                    ...
                    //创建Regular Channel
                    channel = new QCamera3RegularChannel(mCameraHandle->camera_handle,
                        mCameraHandle->ops, captureResultCb,&gCamCapability[mCameraId]-
                        >padding_info,this,newStream,(cam_stream_type_t)mStreamConfigInfo.type[
                        mStreamConfigInfo.num_streams],mStreamConfigInfo.postprocess_mask[
                        mStreamConfigInfo.num_streams],mMetadataChannel,numBuffers);
                    ...
                    newStream->max_buffers = channel->getNumBuffers();
                    newStream->priv = channel;
                    break;
                case HAL_PIXEL_FORMAT_YCbCr_420_888:
                    //创建YWV Channel
                    ...
                    break;
                case HAL_PIXEL_FORMAT_RAW_OPAQUE:
                case HAL_PIXEL_FORMAT_RAW16:
                case HAL_PIXEL_FORMAT_RAW10:
                    //创建Raw Channel
                    ...
                    break;
                case HAL_PIXEL_FORMAT_BLOB:
                    //创建QCamera3PicChannel
                    ...
                    break;
                default:
                    break;
                }
            } else if (newStream->stream_type == CAMERA3_STREAM_INPUT) {
                newStream->max_buffers = MAX_INFLIGHT_REPROCESS_REQUESTS;
            } else {
            }
            for (List<stream_info_t*>::iterator it=mStreamInfo.begin();it != mStreamInfo.end(); 
                    it++) {
                if ((*it)->stream == newStream) {
                    (*it)->channel = (QCamera3Channel*) newStream->priv;
                    break;
                }
            }
        } else {
        }
        if (newStream->stream_type != CAMERA3_STREAM_INPUT)
            mStreamConfigInfo.num_streams++;
        }
    }
    if (isZsl) {
        if (mPictureChannel) {
           mPictureChannel->overrideYuvSize(zslStream->width, zslStream->height);
        }
    } else if (mPictureChannel && m_bIs4KVideo) {
        mPictureChannel->overrideYuvSize(videoWidth, videoHeight);
    }

    //RAW DUMP channel
    if (mEnableRawDump && isRawStreamRequested == false){
        cam_dimension_t rawDumpSize;
        rawDumpSize = getMaxRawSize(mCameraId);
        mRawDumpChannel = new QCamera3RawDumpChannel(mCameraHandle->camera_handle,
            mCameraHandle->ops,rawDumpSize,&gCamCapability[mCameraId]->padding_info,
            this, CAM_QCOM_FEATURE_NONE);
        ...
    }
    //进行相关Channel的配置
    ...
    /* Initialize mPendingRequestInfo and mPendnigBuffersMap */
    for (List<PendingRequestInfo>::iterator i = mPendingRequestsList.begin();
                i != mPendingRequestsList.end(); i++) {
        clearInputBuffer(i->input_buffer);
        i = mPendingRequestsList.erase(i);
    }
    mPendingFrameDropList.clear();
    // Initialize/Reset the pending buffers list
    mPendingBuffersMap.num_buffers = 0;
    mPendingBuffersMap.mPendingBufferList.clear();
    mPendingReprocessResultList.clear();

    return rc;
}
```

此方法内容比较多，只抽取其中核心的代码进行说明，它首先会根据HAL的版本来对stream进行相应的配置初始化，然后再根据stream类型对stream_list的stream创建相应的Channel，主要有QCamera3MetadataChannel，QCamera3SupportChannel等，然后再进行相应的配置，其中QCamera3MetadataChannel在后面的处理capture request的时候会用到，这里就不做分析，而Camerametadata则是Java层和CameraService之间传递的元数据，见android6.0源码分析之Camera API2.0简介中的Camera2架构图，至此，QCamera3HardwareInterface构造结束，与本文相关的就是配置了mCameraDevice.ops。

##### 3.1.2、openCamera()分析
本节主要分析Module是如何打开Camera的，openCamera的代码如下：

``` cpp
[->\hardware\qcom\camera\QCamera2\HAL3\QCamera3HWI.cpp]
int QCamera3HardwareInterface::openCamera(struct hw_device_t **hw_device){
    int rc = 0;
    if (mCameraOpened) {//如果Camera已经被打开，则此次打开的设备为NULL，并且打开结果为PERMISSION_DENIED
        *hw_device = NULL;
        return PERMISSION_DENIED;
    }
    //调用openCamera方法来打开
    rc = openCamera();
    //打开结果处理
    if (rc == 0) {
        //获取打开成功的hw_device_t对象
        *hw_device = &mCameraDevice.common;
    } else
        *hw_device = NULL;
    }
    return rc;
}
```
它调用了openCamera()方法来打开Camera:

``` cpp
[->\hardware\qcom\camera\QCamera2\HAL3\QCamera3HWI.cpp]
int QCamera3HardwareInterface::openCamera()
{
    ...
    //打开camera，获取mCameraHandle
    mCameraHandle = camera_open((uint8_t)mCameraId);
    ...
    mCameraOpened = true;
    //注册mm-camera-interface里的事件处理,其中camEctHandle为事件处理Handle
    rc = mCameraHandle->ops->register_event_notify(mCameraHandle->camera_handle,camEvtHandle
            ,(void *)this);
    return NO_ERROR;
}
```
它调用camera_open方法来打开Camera，并且向CameraHandle注册了Camera 时间处理的Handle–camEvtHandle，首先分析camera_open方法，这里就将进入高通的Camera的实现了，而Mm_camera_interface.c是高通提供的相关操作的接口，接下来分析高通Camera的camera_open方法：

``` cpp
[->\vendor\qcom\proprietary\mm-camera\apps\appslib\mm_camera_interface.c]
mm_camera_vtbl_t * camera_open(uint8_t camera_idx)
{
    int32_t rc = 0;
    mm_camera_obj_t* cam_obj = NULL;
    /* opened already 如果已经打开*/
    if(NULL != g_cam_ctrl.cam_obj[camera_idx]) {
        /* Add reference */
        g_cam_ctrl.cam_obj[camera_idx]->ref_count++;
        pthread_mutex_unlock(&g_intf_lock);
        return &g_cam_ctrl.cam_obj[camera_idx]->vtbl;
    }

    cam_obj = (mm_camera_obj_t *)malloc(sizeof(mm_camera_obj_t));
    ...
    /* initialize camera obj */
    memset(cam_obj, 0, sizeof(mm_camera_obj_t));
    cam_obj->ctrl_fd = -1;
    cam_obj->ds_fd = -1;
    cam_obj->ref_count++;
    cam_obj->my_hdl = mm_camera_util_generate_handler(camera_idx);
    cam_obj->vtbl.camera_handle = cam_obj->my_hdl; /* set handler */
    //mm_camera_ops里绑定了相关的操作接口
    cam_obj->vtbl.ops = &mm_camera_ops;
    pthread_mutex_init(&cam_obj->cam_lock, NULL);
    pthread_mutex_lock(&cam_obj->cam_lock);
    pthread_mutex_unlock(&g_intf_lock);
    //调用mm_camera_open方法来打开camera
    rc = mm_camera_open(cam_obj);

    pthread_mutex_lock(&g_intf_lock);
    ...
    //结果处理，并返回
    ...
}
```
由代码可知，这里将会初始化一个mm_camera_obj_t对象，其中，ds_fd为socket fd，而mm_camera_ops则绑定了相关的接口，最后调用mm_camera_open来打开Camera，首先来看看mm_camera_ops绑定了哪些方法：


``` cpp
[->\vendor\qcom\proprietary\mm-camera\apps\appslib\mm_camera_interface.c]
static mm_camera_ops_t mm_camera_ops = {
    .query_capability = mm_camera_intf_query_capability,
    //注册事件通知的方法
    .register_event_notify = mm_camera_intf_register_event_notify,
    .close_camera = mm_camera_intf_close,
    .set_parms = mm_camera_intf_set_parms,
    .get_parms = mm_camera_intf_get_parms,
    .do_auto_focus = mm_camera_intf_do_auto_focus,
    .cancel_auto_focus = mm_camera_intf_cancel_auto_focus,
    .prepare_snapshot = mm_camera_intf_prepare_snapshot,
    .start_zsl_snapshot = mm_camera_intf_start_zsl_snapshot,
    .stop_zsl_snapshot = mm_camera_intf_stop_zsl_snapshot,
    .map_buf = mm_camera_intf_map_buf,
    .unmap_buf = mm_camera_intf_unmap_buf,
    .add_channel = mm_camera_intf_add_channel,
    .delete_channel = mm_camera_intf_del_channel,
    .get_bundle_info = mm_camera_intf_get_bundle_info,
    .add_stream = mm_camera_intf_add_stream,
    .link_stream = mm_camera_intf_link_stream,
    .delete_stream = mm_camera_intf_del_stream,
    //配置stream的方法
    .config_stream = mm_camera_intf_config_stream,
    .qbuf = mm_camera_intf_qbuf,
    .get_queued_buf_count = mm_camera_intf_get_queued_buf_count,
    .map_stream_buf = mm_camera_intf_map_stream_buf,
    .unmap_stream_buf = mm_camera_intf_unmap_stream_buf,
    .set_stream_parms = mm_camera_intf_set_stream_parms,
    .get_stream_parms = mm_camera_intf_get_stream_parms,
    .start_channel = mm_camera_intf_start_channel,
    .stop_channel = mm_camera_intf_stop_channel,
    .request_super_buf = mm_camera_intf_request_super_buf,
    .cancel_super_buf_request = mm_camera_intf_cancel_super_buf_request,
    .flush_super_buf_queue = mm_camera_intf_flush_super_buf_queue,
    .configure_notify_mode = mm_camera_intf_configure_notify_mode,
    //处理capture的方法
    .process_advanced_capture = mm_camera_intf_process_advanced_capture
};
```
接着分析mm_camera_open方法：

``` cpp
[->/hardware/qcom/camera/QCamera2/stack/mm-camera-interface/src/mm_camera.c]
int32_t mm_camera_open(mm_camera_obj_t *my_obj){
    ...
    do{
        n_try--;
        //根据设备名字，打开相应的设备驱动fd
        my_obj->ctrl_fd = open(dev_name, O_RDWR | O_NONBLOCK);
        if((my_obj->ctrl_fd >= 0) || (errno != EIO) || (n_try <= 0 )) {
            break;
        }
        usleep(sleep_msec * 1000U);
    }while (n_try > 0);
    ...
    //打开domain socket
    n_try = MM_CAMERA_DEV_OPEN_TRIES;
    do {
        n_try--;
        my_obj->ds_fd = mm_camera_socket_create(cam_idx, MM_CAMERA_SOCK_TYPE_UDP);
        usleep(sleep_msec * 1000U);
    } while (n_try > 0);
    ...
    //初始化锁
    pthread_mutex_init(&my_obj->msg_lock, NULL);
    pthread_mutex_init(&my_obj->cb_lock, NULL);
    pthread_mutex_init(&my_obj->evt_lock, NULL);
    pthread_cond_init(&my_obj->evt_cond, NULL);

    //开启线程，它的线程体在mm_camera_dispatch_app_event方法中
    mm_camera_cmd_thread_launch(&my_obj->evt_thread,
                                mm_camera_dispatch_app_event,
                                (void *)my_obj);
    mm_camera_poll_thread_launch(&my_obj->evt_poll_thread,
                                 MM_CAMERA_POLL_TYPE_EVT);
    mm_camera_evt_sub(my_obj, TRUE);
    return rc;
    ...
}
```
由代码可知，它会打开Camera的设备文件，然后开启dispatch_app_event线程，线程方法体mm_camera_dispatch_app_event方法代码如下：

``` cpp
[->/hardware/qcom/camera/QCamera2/stack/mm-camera-interface/src/mm_camera.c]
static void mm_camera_dispatch_app_event(mm_camera_cmdcb_t *cmd_cb,void* user_data){
    mm_camera_cmd_thread_name("mm_cam_event");
    int i;
    mm_camera_event_t *event = &cmd_cb->u.evt;
    mm_camera_obj_t * my_obj = (mm_camera_obj_t *)user_data;
    if (NULL != my_obj) {
        pthread_mutex_lock(&my_obj->cb_lock);
        for(i = 0; i < MM_CAMERA_EVT_ENTRY_MAX; i++) {
            if(my_obj->evt.evt[i].evt_cb) {
                //调用camEvtHandle方法
                my_obj->evt.evt[i].evt_cb(
                    my_obj->my_hdl,
                    event,
                    my_obj->evt.evt[i].user_data);
            }
        }
        pthread_mutex_unlock(&my_obj->cb_lock);
    }
}
```
最后会调用mm-camera-interface中注册好的事件处理evt_cb，它就是在前面注册好的camEvtHandle：

``` cpp
[->\hardware\qcom\camera\QCamera2\HAL3\QCamera3HWI.cpp]
void QCamera3HardwareInterface::camEvtHandle(uint32_t /*camera_handle*/,mm_camera_event_t *evt,
        void *user_data){
    //获取QCamera3HardwareInterface接口指针
    QCamera3HardwareInterface *obj = (QCamera3HardwareInterface *)user_data;
    if (obj && evt) {
        switch(evt->server_event_type) {
            case CAM_EVENT_TYPE_DAEMON_DIED:
                camera3_notify_msg_t notify_msg;
                memset(&notify_msg, 0, sizeof(camera3_notify_msg_t));
                notify_msg.type = CAMERA3_MSG_ERROR;
                notify_msg.message.error.error_code = CAMERA3_MSG_ERROR_DEVICE;
                notify_msg.message.error.error_stream = NULL;
                notify_msg.message.error.frame_number = 0;
                obj->mCallbackOps->notify(obj->mCallbackOps, &notify_msg);
                break;

            case CAM_EVENT_TYPE_DAEMON_PULL_REQ:
                pthread_mutex_lock(&obj->mMutex);
                obj->mWokenUpByDaemon = true;
                //开启process_capture_request
                obj->unblockRequestIfNecessary();
                pthread_mutex_unlock(&obj->mMutex);
                break;  

            default:
                break;
        }
    } else {
    }
}
```
由代码可知，它会调用QCamera3HardwareInterface的unblockRequestIfNecessary来发起结果处理请求：

``` cpp
[->\hardware\qcom\camera\QCamera2\HAL3\QCamera3HWI.cpp]
void QCamera3HardwareInterface::unblockRequestIfNecessary()
{
   // Unblock process_capture_request
   //开启process_capture_request
   pthread_cond_signal(&mRequestCond);
}
```
在初始化QCamera3HardwareInterface对象的时候，就绑定了处理Metadata的回调captureResultCb方法：它主要是对数据源进行相应的处理，而具体的capture请求的结果处理还是由process_capture_request来进行处理的，而这里会调用方法unblockRequestIfNecessary来触发process_capture_request方法执行，而在Camera框架中，发起请求时会启动一个RequestThread线程，在它的threadLoop方法中，会不停的调用process_capture_request方法来进行请求的处理，而它最后会回调Camera3Device中的processCaptureResult方法来进行结果处理：

``` cpp
[->/frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp]
void Camera3Device::processCaptureResult(const camera3_capture_result *result) {
    ...
    {
        ...
        if (mUsePartialResult && result->result != NULL) {
            if (mDeviceVersion >= CAMERA_DEVICE_API_VERSION_3_2) {
                ...
                if (isPartialResult) {
                    request.partialResult.collectedResult.append(result->result);
                }
            } else {
                camera_metadata_ro_entry_t partialResultEntry;
                res = find_camera_metadata_ro_entry(result->result,
                        ANDROID_QUIRKS_PARTIAL_RESULT, &partialResultEntry);
                if (res != NAME_NOT_FOUND &&partialResultEntry.count > 0 &&
                        partialResultEntry.data.u8[0] ==ANDROID_QUIRKS_PARTIAL_RESULT_PARTIAL) {
                    isPartialResult = true;
                    request.partialResult.collectedResult.append(
                        result->result);
                    request.partialResult.collectedResult.erase(
                        ANDROID_QUIRKS_PARTIAL_RESULT);
                }
            }

            if (isPartialResult) {
                // Fire off a 3A-only result if possible
                if (!request.partialResult.haveSent3A) {
                    //处理3A结果
                    request.partialResult.haveSent3A =processPartial3AResult(frameNumber,
                        request.partialResult.collectedResult,request.resultExtras);
                }
            }
        }
        ...
        //查找camera元数据入口
        camera_metadata_ro_entry_t entry;
        res = find_camera_metadata_ro_entry(result->result,
                ANDROID_SENSOR_TIMESTAMP, &entry);

        if (shutterTimestamp == 0) {
            request.pendingOutputBuffers.appendArray(result->output_buffers,
                result->num_output_buffers);
        } else {
            重要的分析//返回处理的outputbuffer
            returnOutputBuffers(result->output_buffers,
                result->num_output_buffers, shutterTimestamp);
        }

        if (result->result != NULL && !isPartialResult) {
            if (shutterTimestamp == 0) {
                request.pendingMetadata = result->result;
                request.partialResult.collectedResult = collectedPartialResult;
            } else {
                CameraMetadata metadata;
                metadata = result->result;
                //发送Capture结构，即调用通知回调
                sendCaptureResult(metadata, request.resultExtras,
                    collectedPartialResult, frameNumber, hasInputBufferInRequest,
                    request.aeTriggerCancelOverride);
            }
        }

        removeInFlightRequestIfReadyLocked(idx);
    } // scope for mInFlightLock

    if (result->input_buffer != NULL) {
        if (hasInputBufferInRequest) {
            Camera3Stream *stream =
                Camera3Stream::cast(result->input_buffer->stream);
            重要的分析//返回处理的inputbuffer
            res = stream->returnInputBuffer(*(result->input_buffer));
        } else {}
    }
}
```
分析returnOutputBuffers方法，inputbuffer的runturnInputBuffer方法流程类似：

``` cpp
[->/frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp]
void Camera3Device::returnOutputBuffers(const camera3_stream_buffer_t *outputBuffers, size_t 
        numBuffers, nsecs_t timestamp) {
    for (size_t i = 0; i < numBuffers; i++)
    {
        Camera3Stream *stream = Camera3Stream::cast(outputBuffers[i].stream);
        status_t res = stream->returnBuffer(outputBuffers[i], timestamp);
        ...
    }
}
```
方法里调用了returnBuffer方法：

``` cpp
[->/frameworks/av/services/camera/libcameraservice/device3/Camera3Stream.cpp]
status_t Camera3Stream::returnBuffer(const camera3_stream_buffer &buffer,nsecs_t timestamp) {
    //返回buffer
    status_t res = returnBufferLocked(buffer, timestamp);
    if (res == OK) {
        fireBufferListenersLocked(buffer, /*acquired*/false, /*output*/true);
        mOutputBufferReturnedSignal.signal();
    }
    return res;
}
```
再继续看returnBufferLocked,它调用了returnAnyBufferLocked方法，而returnAnyBufferLocked方法又调用了returnBufferCheckedLocked方法，现在分析returnBufferCheckedLocked：

``` cpp
[->/frameworks/av/services/camera/libcameraservice/device3/Camera3OutputStream.cpp]
status_t Camera3OutputStream::returnBufferCheckedLocked(const camera3_stream_buffer &buffer,
            nsecs_t timestamp,bool output,/*out*/sp<Fence> *releaseFenceOut) {
    ...
    // Fence management - always honor release fence from HAL
    sp<Fence> releaseFence = new Fence(buffer.release_fence);
    int anwReleaseFence = releaseFence->dup();


    if (buffer.status == CAMERA3_BUFFER_STATUS_ERROR) {
        // Cancel buffer
        res = currentConsumer->cancelBuffer(currentConsumer.get(),
                container_of(buffer.buffer, ANativeWindowBuffer, handle),
                anwReleaseFence);
        ...
    } else {
        ...
        res = currentConsumer->queueBuffer(currentConsumer.get(),
                container_of(buffer.buffer, ANativeWindowBuffer, handle),
                anwReleaseFence);
        ...
    }
    ...
    return res;
}
```

由代码可知，如果Buffer没有出现状态错误，它会调用currentConsumer的queueBuffer方法，而具体的Consumer则是在应用层初始化Camera时进行绑定的，典型的Consumer有SurfaceTexture，ImageReader等，而在Native层中，它会调用BufferQueueProducer的queueBuffer方法：

``` cpp
[->\frameworks\native\libs\gui\BufferQueueProducer.cpp]
status_t BufferQueueProducer::queueBuffer(int slot,
        const QueueBufferInput &input, QueueBufferOutput *output) {
    ...
    //初始化Frame可用的监听器
    sp<IConsumerListener> frameAvailableListener;
    sp<IConsumerListener> frameReplacedListener;
    int callbackTicket = 0;
    BufferItem item;
    { // Autolock scope
        ...
        const sp<GraphicBuffer>& graphicBuffer(mSlots[slot].mGraphicBuffer);
        Rect bufferRect(graphicBuffer->getWidth(), graphicBuffer->getHeight());
        Rect croppedRect;
        crop.intersect(bufferRect, &croppedRect);
        ...
        //如果队列为空
        if (mCore->mQueue.empty()) {
            mCore->mQueue.push_back(item);
            frameAvailableListener = mCore->mConsumerListener;
        } else {
            //否则，不为空，对Buffer进行处理，并获取FrameAvailableListener监听
            BufferQueueCore::Fifo::iterator front(mCore->mQueue.begin());
            if (front->mIsDroppable) {
                if (mCore->stillTracking(front)) {
                    mSlots[front->mSlot].mBufferState = BufferSlot::FREE;
                    mCore->mFreeBuffers.push_front(front->mSlot);
                }
                *front = item;
                frameReplacedListener = mCore->mConsumerListener;
            } else {
                mCore->mQueue.push_back(item);
                frameAvailableListener = mCore->mConsumerListener;
            }
        }

        mCore->mBufferHasBeenQueued = true;
        mCore->mDequeueCondition.broadcast();

        output->inflate(mCore->mDefaultWidth, mCore->mDefaultHeight,mCore->mTransformHint,
                static_cast<uint32_t>(mCore->mQueue.size()));

        // Take a ticket for the callback functions
        callbackTicket = mNextCallbackTicket++;

        mCore->validateConsistencyLocked();
    } // Autolock scope
    ...
    {
        ...
        if (frameAvailableListener != NULL) {
            //回调SurfaceTexture中定义好的监听IConsumerListener的onFrameAvailable方法来对数据进行处理
            frameAvailableListener->onFrameAvailable(item);
        } else if (frameReplacedListener != NULL) {
            frameReplacedListener->onFrameReplaced(item);
        }

        ++mCurrentCallbackTicket;
        mCallbackCondition.broadcast();
    }

    return NO_ERROR;
}
```
由代码可知，它最后会调用Consumer的回调FrameAvailableListener的onFrameAvailable方法，到这里，就比较清晰为什么我们在写Camera应用，为其初始化Surface时，我们需要重写FrameAvailableListener了，因为在此方法里面，会进行结果的处理，至此，Camera HAL的Open流程就分析结束了。下面给出流程的时序图： 

![Alt text](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/camera.system/01-15-open_camera.png)


#### （四）、Camera API2.0 初始化流程分析
##### 4.1、Camera2 应用层（Java层）Open()过程分析
Camera2的初始化流程与Camera1.0有所区别，本文将就Camera2的内置应用来分析Camera2.0的初始化过程。Camera2.0首先启动的是CameraActivity，而它继承自QuickActivity，在代码中你会发现没有重写OnCreate等生命周期方法，因为此处采用的是模板方法的设计模式，在QuickActivity中的onCreate方法调用的是onCreateTasks等方法，所以要看onCreate方法就只须看onCreateTasks方法即可：

``` java
[->/packages/apps/Camera2/src/com/android/camera/CameraActivity.java]
Override
public void onCreateTasks(Bundle state) {
    Profile profile = mProfiler.create("CameraActivity.onCreateTasks")
                            .start();
    ...
    mOnCreateTime = System.currentTimeMillis();
    mAppContext = getApplicationContext();
    mMainHandler = new MainHandler(this, getMainLooper());
    …
    try {
        //初始化OneCameraOpener对象
        ①mOneCameraOpener = OneCameraModule.provideOneCameraOpener(
                mFeatureConfig, mAppContext,mActiveCameraDeviceTracker,
                ResolutionUtil.getDisplayMetrics(this));
        mOneCameraManager = OneCameraModule.provideOneCameraManager();
    } catch (OneCameraException e) {...}
    …
    //建立模块信息
    ②ModulesInfo.setupModules(mAppContext, mModuleManager, mFeatureConfig);
    …
    //进行初始化
    ③mCurrentModule.init(this, isSecureCamera(), isCaptureIntent());
    …
}
```
如代码所示，重要的有以上三点，先看第一点：

``` java
[->/packages/apps/Camera2/src/com/android/camera/one/OneCameraModule.java]
public static OneCameraOpener provideOneCameraOpener(OneCameraFeatureConfig     
            featureConfig, Context context, ActiveCameraDeviceTracker 
            activeCameraDeviceTracker,DisplayMetrics displayMetrics) 
            throws OneCameraException {
    //创建OneCameraOpener对象
    Optional<OneCameraOpener> manager = Camera2OneCameraOpenerImpl.create(
              featureConfig, context, activeCameraDeviceTracker, displayMetrics);
    if (!manager.isPresent()) {
        manager = LegacyOneCameraOpenerImpl.create();
    }
    ...
    return manager.get();
}
```
它调用Camera2OneCameraOpenerImpl的create方法来获得一个OneCameraOpener对象，以供CameraActivity之后的操作使用，继续看create方法：

``` java
[->/packages/apps/Camera2/src/com/android/camera/one/OneCameraModule.java]
public static Optional<OneCameraOpener> create(OneCameraFeatureConfig 
            featureConfig, Context context, ActiveCameraDeviceTracker 
            activeCameraDeviceTracker, DisplayMetrics displayMetrics) {
    ...
    CameraManager cameraManager;
    try {
        cameraManager = AndroidServices.instance().provideCameraManager();
    } catch (IllegalStateException ex) {...}
    //新建一个Camera2OneCameraOpenerImpl对象
    OneCameraOpener oneCameraOpener = new Camera2OneCameraOpenerImpl(
                featureConfig, context, cameraManager,
                activeCameraDeviceTracker, displayMetrics);
    return Optional.of(oneCameraOpener);
}
```
很明显，它首先获取一个cameraManger对象，然后根据这个cameraManager对象来新创建了一个Camera2OneCameraOpenerImpl对象，所以第一步主要是为了获取一个OneCameraOpener对象，它的实现为Camera2OneCameraOpenerImpl类。 
继续看第二步，ModulesInfo.setupModules:

``` cpp
[->/packages/apps/Camera2/src/com/android/camera/module/ModulesInfo.java]
public static void setupModules(Context context, ModuleManager moduleManager,
            OneCameraFeatureConfig config) {
        Resources res = context.getResources();
        int photoModuleId = context.getResources().getInteger(
            R.integer.camera_mode_photo);
        //注册Photo模块
        registerPhotoModule(moduleManager, photoModuleId, 
            SettingsScopeNamespaces.PHOTO,config.isUsingCaptureModule());
        //计算你还Photo模块设置为默认的模块
        moduleManager.setDefaultModuleIndex(photoModuleId);
        //注册Videa模块
        registerVideoModule(moduleManager, res.getInteger(
            R.integer.camera_mode_video),SettingsScopeNamespaces.VIDEO);
        if (PhotoSphereHelper.hasLightCycleCapture(context)) {//开启闪光
            //注册广角镜头
            registerWideAngleModule(moduleManager, res.getInteger(
                R.integer.camera_mode_panorama),SettingsScopeNamespaces
                .PANORAMA);
            //注册光球模块
            registerPhotoSphereModule(moduleManager,res.getInteger(
                R.integer.camera_mode_photosphere),
                SettingsScopeNamespaces.PANORAMA);
        }
        //若需重新聚焦
        if (RefocusHelper.hasRefocusCapture(context)) {
            //注册重聚焦模块
            registerRefocusModule(moduleManager, res.getInteger(
                R.integer.camera_mode_refocus),
                SettingsScopeNamespaces.REFOCUS);
        }
        //如果有色分离模块
        if (GcamHelper.hasGcamAsSeparateModule(config)) {
            //注册色分离模块
            registerGcamModule(moduleManager, res.getInteger(
                R.integer.camera_mode_gcam),SettingsScopeNamespaces.PHOTO,
                config.getHdrPlusSupportLevel(OneCamera.Facing.BACK));
        }
        int imageCaptureIntentModuleId = res.getInteger(
            R.integer.camera_mode_capture_intent);
        registerCaptureIntentModule(moduleManager, 
            imageCaptureIntentModuleId,SettingsScopeNamespaces.PHOTO,
            config.isUsingCaptureModule());
}
```
代码根据配置信息，进行一系列模块的注册，其中PhotoModule和VideoModule被注册，而其他的module则是根据配置来进行的，因为打开Camera应用，既可以拍照片也可以拍视频，此处，只分析PhoneModule的注册：

``` cpp
[->/packages/apps/Camera2/src/com/android/camera/module/ModulesInfo.java]
private static void registerPhotoModule(ModuleManager moduleManager, final 
            int moduleId, final String namespace, 
            final boolean enableCaptureModule) {
        //向ModuleManager注册PhotoModule模块
        moduleManager.registerModule(new ModuleManager.ModuleAgent() {

            @Override
            public int getModuleId() {
                return moduleId;
            }

            @Override
            public boolean requestAppForCamera() {
                return !enableCaptureModule;
            }

            @Override
            public String getScopeNamespace() {
                return namespace;
            }

            @Override
            public ModuleController createModule(AppController app, Intent 
                    intent) {
                Log.v(TAG, "EnableCaptureModule = " + enableCaptureModule);
                //创建ModuleController
                return enableCaptureModule ? new CaptureModule(app) 
                        : new PhotoModule(app);
            }
        });
}
```

由代码可知，它最终是由ModuleManager来新建一个CaptureModule实例，而CaptureModule其实实现了ModuleController ，即创建了一个CaptureModule模式下的ModuleController对象，而真正的CaptureModule的具体实现为ModuleManagerImpl。 
至此，前两步已经获得了OneCameraOpener以及新建了ModuleController，并进行了注册，接下来分析第三步，mCurrentModule.init(this, isSecureCamera(), isCaptureIntent()):

``` java
[->/packages/apps/Camera2/src/com/android/camera/CaptureModule.java]
public void init(CameraActivity activity, boolean isSecureCamera, boolean 
            isCaptureIntent) {
        ...
        HandlerThread thread = new HandlerThread("CaptureModule.mCameraHandler");
        thread.start();
        mCameraHandler = new Handler(thread.getLooper());
        //获取第一步中创建的OneCameraOpener对象
        mOneCameraOpener = mAppController.getCameraOpener();

        try {
            //获取前面创建的OneCameraManager对象
            mOneCameraManager = OneCameraModule.provideOneCameraManager();
        } catch (OneCameraException e) {
            Log.e(TAG, "Unable to provide a OneCameraManager. ", e);
        }
       `...
        //新建CaptureModule的UI
        mUI = new CaptureModuleUI(activity, mAppController.
                getModuleLayoutRoot(), mUIListener);
        //设置预览状态的监听
        mAppController.setPreviewStatusListener(mPreviewStatusListener);
        synchronized (mSurfaceTextureLock) {
            //获取SurfaceTexture
            mPreviewSurfaceTexture = mAppController.getCameraAppUI()
                .getSurfaceTexture();
        }  
}
```
首先获取前面创建的OneCameraOpener对象以及OneCameraManager对象，然后再设置预览状态监听，这里主要分析预览状态的监听：
``` java
[->/packages/apps/Camera2/src/com/android/camera/CaptureModule.java]
private final PreviewStatusListener mPreviewStatusListener = new 
    PreviewStatusListener() {
        ...

        @Override
        public void onSurfaceTextureAvailable(SurfaceTexture surface, 
            int width, int height) {
            updatePreviewTransform(width, height, true);
            synchronized (mSurfaceTextureLock) {
                mPreviewSurfaceTexture = surface;
            }
            //打开Camera
            reopenCamera();
        }

        @Override
        public boolean onSurfaceTextureDestroyed(SurfaceTexture surface) {
            Log.d(TAG, "onSurfaceTextureDestroyed");
            synchronized (mSurfaceTextureLock) {
                mPreviewSurfaceTexture = null;
            }
            //关闭Camera
            closeCamera();
            return true;
        }

        @Override
        public void onSurfaceTextureSizeChanged(SurfaceTexture surface, 
            int width, int height) {
            //更新预览尺寸
            updatePreviewBufferSize();
        }
        ...
    };
```
由代码可知，当SurfaceTexture的状态变成可用的时候，会调用reopenCamera()方法来打开Camera，所以继续分析reopenCamera()方法：
``` java
[->/packages/apps/Camera2/src/com/android/camera/CaptureModule.java]
private void reopenCamera() {
    if (mPaused) {
        return;
    }
    AsyncTask.THREAD_POOL_EXECUTOR.execute(new Runnable() {
        @Override
        public void run() {
            closeCamera();
            if(!mAppController.isPaused()) {
                //开启Camera并开始预览
                openCameraAndStartPreview();
            }
        }
    });
}
```
它采用异步任务的方法，开启一个异步线程来进行启动操作，首先关闭打开的Camera，然后如果AppController不处于暂停状态，则打开Camera并启动Preview操作，所以继续分析openCameraAndStartPreview方法：
``` java
[->/packages/apps/Camera2/src/com/android/camera/CaptureModule.java]
private void openCameraAndStartPreview() {
    ...
    if (mOneCameraOpener == null) {
        Log.e(TAG, "no available OneCameraManager, showing error dialog");
        //释放CameraOpenCloseLock锁
        mCameraOpenCloseLock.release();
        mAppController.getFatalErrorHandler().onGenericCameraAccessFailure();
        guard.stop("No OneCameraManager");
        return;
    }
    // Derive objects necessary for camera creation.
    MainThread mainThread = MainThread.create();

    //查找需要打开的CameraId
    CameraId cameraId = mOneCameraManager.findFirstCameraFacing(
        mCameraFacing);
    ...
    //打开Camera
    mOneCameraOpener.open(cameraId, captureSetting, mCameraHandler,
        mainThread, imageRotationCalculator, mBurstController, 
        mSoundPlayer,new OpenCallback() {

            @Override
            public void onFailure() {
                //进行失败的处理
                ...
            }

            @Override
            public void onCameraClosed() {
                ...
            }

            @Override
            public void onCameraOpened(@Nonnull final OneCamera camera) {
                Log.d(TAG, "onCameraOpened: " + camera);
                mCamera = camera;

                if (mAppController.isPaused()) {
                    onFailure();
                    return;
                }
                ...
                mMainThread.execute(new Runnable() {
                    @Override
                    public void run() {
                        //通知UI，Camera状态变化
                        mAppController.getCameraAppUI().onChangeCamera();
                        //使能拍照按钮
                        mAppController.getButtonManager().enableCameraButton();
                    }
                });
                //至此，Camera打开成功，开始预览                    
                camera.startPreview(new Surface(getPreviewSurfaceTexture()), 
                    new CaptureReadyCallback() {
                        @Override
                        public void onSetupFailed() {
                            ...
                        }

                        @Override
                        public void onReadyForCapture() {
                            //释放锁
                            mCameraOpenCloseLock.release();
                            mMainThread.execute(new Runnable() {
                                @Override
                                public void run() {
                                    ...
                                    onPreviewStarted();
                                    ...
                                    onReadyStateChanged(true);
                                    //设置CaptureModule为Capture准备的状态监听
                                    mCamera.setReadyStateChangedListener(
                                        CaptureModule.this);
                                        mUI.initializeZoom(mCamera.getMaxZoom());                                 
                                        mCamera.setFocusStateListener(
                                            CaptureModule.this);
                                    }
                                });
                            }
                        });
                    } 
              }, mAppController.getFatalErrorHandler());
        guard.stop("mOneCameraOpener.open()");
    }
}
```

首先，它主要会调用Camera2OneCameraOpenerImpl的open方法来打开Camera，并定义了开启的回调函数，对开启结束后的结果进行处理，如失败则释放mCameraOpenCloseLock，并暂停mAppController，如果打开成功，通知UI成功，并开启Camera的Preview，并且定义了Preview的各种回调操作，这里主要分析Open过程，所以继续分析：

``` java
[->/packages/apps/Camera2/src/com/android/camera/one/v2/Camera2OneCameraOpenerImpl.java]
Override
public void open(
    ...
    mActiveCameraDeviceTracker.onCameraOpening(cameraKey);
    //打开Camera，此处调用框架层的CameraManager类的openCamera，进入frameworks层
    mCameraManager.openCamera(cameraKey.getValue(), 
        new CameraDevice.StateCallback() {
        private boolean isFirstCallback = true;
        @Override
        ...

        @Override
        public void onOpened(CameraDevice device) {
            //第一次调用此回调
            if (isFirstCallback) {
                isFirstCallback = false;
                try {
                    CameraCharacteristics characteristics = mCameraManager
                        .getCameraCharacteristics(device.getId());
                    ...
                    //创建OneCamera对象
                    OneCamera oneCamera = OneCameraCreator.create(device,
                        characteristics, mFeatureConfig, captureSetting,
                        mDisplayMetrics, mContext, mainThread,
                        imageRotationCalculator, burstController, soundPlayer,
                        fatalErrorHandler);

                    if (oneCamera != null) {
                        //如果oneCamera不为空，则回调onCameraOpened，后面将做分析
                        openCallback.onCameraOpened(oneCamera);
                    } else {
                        ...
                        openCallback.onFailure();
                    }
                } catch (CameraAccessException e) {
                    openCallback.onFailure();
                } catch (OneCameraAccessException e) {
                    Log.d(TAG, "Could not create OneCamera", e);
                    openCallback.onFailure();
                }
            }
        }
    }, handler);
    ...
}
```
至此，Camera的初始化流程中应用层的分析就差不多了，下一步将会调用CameraManager的openCamera方法来进入框架层，并进行Camera的初始化，下面将应用层的初始化时序图： 

![Alt text](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/camera.system/01-16-camera_open_java.png)

##### 4.2、Camera2 框架层（JNI & Native）Open()过程分析
由上面的分析可知，将由应用层进入到框架层处理，将会调用CameraManager的openCamera方法，并且定义了CameraDevice的状态回调函数，具体的回调操作此处不做分析，继续跟踪openCamera()方法：

``` cpp
//CameraManager.java(frameworks/base/core/java/android/hardware/camera2)
@RequiresPermission(android.Manifest.permission.CAMERA)
public void openCamera(@NonNull String cameraId,@NonNull final 
        CameraDevice.StateCallback callback, @Nullable Handler handler)
        throws CameraAccessException {
    ...
    openCameraDeviceUserAsync(cameraId, callback, handler);
}
```
由代码可知，此处与Camera1.0有明显不同，Camera1.0是通过一个异步的线程以及JNI来调用android_hardware_camera.java里面的native_setup方法来连接Camera，其使用的是C++的Binder来与CameraService进行通信的，而此处则不一样，它直接使用的是Java层的Binder来进行通信，先看openCameraDeviceUserAsync代码:

``` java
//CameraManager.java(frameworks/base/core/java/android/hardware/camera2)
private CameraDevice openCameraDeviceUserAsync(String cameraId,
        CameraDevice.StateCallback callback, Handler handler)
        throws CameraAccessException {
    CameraCharacteristics characteristics = getCameraCharacteristics(
        cameraId);
    CameraDevice device = null;
    try {
        synchronized (mLock) {
            ICameraDeviceUser cameraUser = null;
            //初始化一个CameraDevice对象
            android.hardware.camera2.impl.CameraDeviceImpl deviceImpl =
                new android.hardware.camera2.impl.CameraDeviceImpl(cameraId,
                callback, handler, characteristics);
            BinderHolder holder = new BinderHolder();
            //获取回调
            ICameraDeviceCallbacks callbacks = deviceImpl.getCallbacks();
            int id = Integer.parseInt(cameraId);
            try {
                if (supportsCamera2ApiLocked(cameraId)) {
                    //通过Java层的Binder获取CameraService                        
                    ICameraService cameraService = CameraManagerGlobal.get()
                        .getCameraService();
                    ...
                    //通过CameraService连接Camera设备
                    cameraService.connectDevice(callbacks, id, mContext
                        .getOpPackageName(), USE_CALLING_UID, holder);
                    //获取连接成功的CameraUser对象，它用来与CameraService通信
                    cameraUser = ICameraDeviceUser.Stub.asInterface(
                        holder.getBinder());
                } else {
                    //使用遗留的API
                    cameraUser = CameraDeviceUserShim.connectBinderShim(
                        callbacks, id);
                }
            } catch (CameraRuntimeException e) {
                ...
        } catch (RemoteException e) {
            ...
            //将其包装成DeviceImpl对象，供应用层使用
            deviceImpl.setRemoteDevice(cameraUser);
            device = deviceImpl;
        }
    } catch (NumberFormatException e) {
        ...
    } catch (CameraRuntimeException e) {
        throw e.asChecked();
    }
    return device;
}
```
此方法的目的是通过CameraService来连接并获取CameraDevice对象，该对象用来与Camera进行通信操作。代码首先通过Java层的Binder机制获取CameraService，然后调用其connectDevice方法来连接CaneraDevice，最后Camera返回的是CameraDeviceUser对象，而接着将其封装成Jav层CameraDevice对象，而之后所有与Camera的通信都通过CameraDevice的接口来进行。接下来分析一下Native层下的CameraDevice的初始化过程：

``` cpp
[->\frameworks\av\services\camera\libcameraservice\CameraService.cpp]
//CameraService.cpp，其中device为输出对象
status_t CameraService::connectDevice(const sp<ICameraDeviceCallbacks>& cameraCb,int cameraId,
        const String16& clientPackageName,int clientUid,/*out*/sp<ICameraDeviceUser>& device) {

    status_t ret = NO_ERROR;
    String8 id = String8::format("%d", cameraId);
    sp<CameraDeviceClient> client = nullptr;
    ret = connectHelper<ICameraDeviceCallbacks,CameraDeviceClient>(cameraCb, id,
            CAMERA_HAL_API_VERSION_UNSPECIFIED, clientPackageName, clientUid, API_2, false, false,
            /*out*/client);//client为输出对象
    ...
    device = client;
    return NO_ERROR;
}
```
Native层的connectDevice方法就是调用了connectHelper方法，所以继续分析connectHelper：

``` cpp
[->\frameworks\av\services\camera\libcameraservice\CameraService.h]
//CameraService.h
template<class CALLBACK, class CLIENT>
status_t CameraService::connectHelper(const sp<CALLBACK>& cameraCb, const String8& cameraId,
        int halVersion, const String16& clientPackageName, int clientUid,
        apiLevel effectiveApiLevel, bool legacyMode, bool shimUpdateOnly,
        /*out*/sp<CLIENT>& device) {
    status_t ret = NO_ERROR;
    String8 clientName8(clientPackageName);
    int clientPid = getCallingPid();
    ...
    sp<CLIENT> client = nullptr;
    {
        ...

        //如果有必要，给FlashLight关闭设备的机会
        mFlashlight->prepareDeviceOpen(cameraId);
        //获取CameraId
        int id = cameraIdToInt(cameraId);
        ...
        //获取Device的版本，此处为Device3
        int deviceVersion = getDeviceVersion(id, /*out*/&facing);
        sp<BasicClient> tmp = nullptr;
        //获取client对象
        if((ret = makeClient(this, cameraCb, clientPackageName, cameraId, facing, clientPid,
                clientUid, getpid(), legacyMode, halVersion, deviceVersion, effectiveApiLevel,
                /*out*/&tmp)) != NO_ERROR) {
            return ret;
        }
        client = static_cast<CLIENT*>(tmp.get());
        //调用client的初始化函数来初始化模块
        if ((ret = client->initialize(mModule)) != OK) {
            ALOGE("%s: Could not initialize client from HAL module.", __FUNCTION__);
            return ret;
        }

        sp<IBinder> remoteCallback = client->getRemote();
        if (remoteCallback != nullptr) {
            remoteCallback->linkToDeath(this);
        }
    } // lock is destroyed, allow further connect calls
    //将client赋值给输出Device
    device = client;
    return NO_ERROR;
}
```
CameraService根据Camera的相关参数来获取一个client，如makeClient方法，然后再调用client的initialize来进行初始化，首先看makeClient：
``` cpp
[->\frameworks\av\services\camera\libcameraservice\CameraService.cpp]
//CameraService.cpp，其中device为输出对象
status_t CameraService::makeClient(const sp<CameraService>& cameraService,
        const sp<IInterface>& cameraCb, const String16& packageName, const String8& cameraId,
        int facing, int clientPid, uid_t clientUid, int servicePid, bool legacyMode,
        int halVersion, int deviceVersion, apiLevel effectiveApiLevel,
        /*out*/sp<BasicClient>* client) {

    //将字符串的CameraId转换成整形
    int id = cameraIdToInt(cameraId);
    ...
    if (halVersion < 0 || halVersion == deviceVersion) {//判断Camera HAL版本是否和Device的版本相同
        switch(deviceVersion) {
          case CAMERA_DEVICE_API_VERSION_1_0:
            if (effectiveApiLevel == API_1) {  // Camera1 API route
                sp<ICameraClient> tmp = static_cast<ICameraClient*>(cameraCb.get());
                *client = new CameraClient(cameraService, tmp, packageName, id, facing,
                        clientPid, clientUid, getpid(), legacyMode);
            } else { // Camera2 API route
                ALOGW("Camera using old HAL version: %d", deviceVersion);
                return -EOPNOTSUPP;
            }
            break;
          case CAMERA_DEVICE_API_VERSION_2_0:
          case CAMERA_DEVICE_API_VERSION_2_1:
          case CAMERA_DEVICE_API_VERSION_3_0:
          case CAMERA_DEVICE_API_VERSION_3_1:
          case CAMERA_DEVICE_API_VERSION_3_2:
          case CAMERA_DEVICE_API_VERSION_3_3:
            if (effectiveApiLevel == API_1) { // Camera1 API route
                sp<ICameraClient> tmp = static_cast<ICameraClient*>(cameraCb.get());
                *client = new Camera2Client(cameraService, tmp, packageName, id, facing,
                        clientPid, clientUid, servicePid, legacyMode);
            } else { // Camera2 API route
                sp<ICameraDeviceCallbacks> tmp =
                        static_cast<ICameraDeviceCallbacks*>(cameraCb.get());
                *client = new CameraDeviceClient(cameraService, tmp, packageName, id,
                        facing, clientPid, clientUid, servicePid);
            }
            break;
          default:
            // Should not be reachable
            ALOGE("Unknown camera device HAL version: %d", deviceVersion);
            return INVALID_OPERATION;
        }
    } else {
        // A particular HAL version is requested by caller. Create CameraClient
        // based on the requested HAL version.
        if (deviceVersion > CAMERA_DEVICE_API_VERSION_1_0 &&
            halVersion == CAMERA_DEVICE_API_VERSION_1_0) {
            // Only support higher HAL version device opened as HAL1.0 device.
            sp<ICameraClient> tmp = static_cast<ICameraClient*>(cameraCb.get());
            *client = new CameraClient(cameraService, tmp, packageName, id, facing,
                    clientPid, clientUid, servicePid, legacyMode);
        } else {
            // Other combinations (e.g. HAL3.x open as HAL2.x) are not supported yet.
            ALOGE("Invalid camera HAL version %x: HAL %x device can only be"
                    " opened as HAL %x device", halVersion, deviceVersion,
                    CAMERA_DEVICE_API_VERSION_1_0);
            return INVALID_OPERATION;
        }
    }
    return NO_ERROR;
}
```
其中就是创建一个Client对象，由于此处分析的是Camera API2.0，其HAL的版本是3.0+，而Device的版本则其Device的版本即为3.0+，所以会创建一个CameraDeviceClient对象，至此，makeClient已经创建了client对象，并返回了，接着看它的初始化：

``` cpp
[->\frameworks\av\services\camera\libcameraservice\api2\CameraDeviceClient.cpp]
status_t CameraDeviceClient::initialize(CameraModule *module)
{
    ATRACE_CALL();
    status_t res;
    //调用Camera2ClientBase的初始化函数来初始化CameraModule模块
    res = Camera2ClientBase::initialize(module);
    if (res != OK) {
        return res;
    }

    String8 threadName;
    //初始化FrameProcessor
    mFrameProcessor = new FrameProcessorBase(mDevice);
    threadName = String8::format("CDU-%d-FrameProc", mCameraId);
    mFrameProcessor->run(threadName.string());
    //并注册监听，监听的实现就在CameraDeviceClient类中
    mFrameProcessor->registerListener(FRAME_PROCESSOR_LISTENER_MIN_ID,
            FRAME_PROCESSOR_LISTENER_MAX_ID, /*listener*/this,/*sendPartials*/true);
    return OK;
}
```
它会调用Camera2ClientBase的initialize方法来初始化，并且会初始化一个FrameProcessor来进行帧处理，主要是回调每一帧的ExtraResult到应用中，也就是3A相关的数据信息。而Camera1.0中各种Processor模块，即将数据打包处理后再返回到应用的模块都已经不存在，而Camera2.0中将由MediaRecorder、SurfaceView、ImageReader等来直接处理，总体来说效率更好。继续看initialize：

``` cpp
[->\frameworks\av\services\camera\libcameraservice\common\Camera2ClientBase.cpp]
template <typename TClientBase>
status_t Camera2ClientBase<TClientBase>::initialize(CameraModule *module) {
    ...
    //调用Device的initialie方法
    res = mDevice->initialize(module);
    ...
    res = mDevice->setNotifyCallback(this);
    return OK;
}
```
代码就是调用了Device的initialize方法，此处的Device是在Camera2ClientBase的构造函数中创建的：

``` cpp
[->\frameworks\av\services\camera\libcameraservice\common\Camera2ClientBase.cpp]
template <typename TClientBase>
Camera2ClientBase<TClientBase>::Camera2ClientBase(
        const sp<CameraService>& cameraService,
        const sp<TCamCallbacks>& remoteCallback,
        const String16& clientPackageName,
        int cameraId,
        int cameraFacing,
        int clientPid,
        uid_t clientUid,
        int servicePid):
        TClientBase(cameraService, remoteCallback, clientPackageName,
                cameraId, cameraFacing, clientPid, clientUid, servicePid),
        mSharedCameraCallbacks(remoteCallback),
        mDeviceVersion(cameraService->getDeviceVersion(cameraId)),
        mDeviceActive(false)
{
    ALOGI("Camera %d: Opened. Client: %s (PID %d, UID %d)", cameraId,
            String8(clientPackageName).string(), clientPid, clientUid);

    mInitialClientPid = clientPid;
    mDevice = new Camera3Device(cameraId);
    LOG_ALWAYS_FATAL_IF(mDevice == 0, "Device should never be NULL here.");
}
```
目前Camera API是2.0，而Device的API已经是3.0+了，继续看Camera3Device的构造方法。

``` cpp
[->\frameworks\av\services\camera\libcameraservice\device3\Camera3Device.cpp]
Camera3Device::Camera3Device(int id):
        mId(id),
        mIsConstrainedHighSpeedConfiguration(false),
        mHal3Device(NULL),
        mStatus(STATUS_UNINITIALIZED),
        mStatusWaiters(0),
        mUsePartialResult(false),
        mNumPartialResults(1),
        mTimestampOffset(0),
        mNextResultFrameNumber(0),
        mNextReprocessResultFrameNumber(0),
        mNextShutterFrameNumber(0),
        mNextReprocessShutterFrameNumber(0),
        mListener(NULL)
{
    ATRACE_CALL();
    camera3_callback_ops::notify = &sNotify;
    camera3_callback_ops::process_capture_result = &sProcessCaptureResult;
    ALOGV("%s: Created device for camera %d", __FUNCTION__, id);
}
```
很显然，它将会创建一个Camera3Device对象，所以，Device的initialize就是调用了Camera3Device的initialize方法：

``` cpp
[->\frameworks\av\services\camera\libcameraservice\device3\Camera3Device.cpp]
status_t Camera3Device::initialize(CameraModule *module)
{
    ...
    camera3_device_t *device;
    //打开Camera HAL层的Deivce
    res = module->open(deviceName.string(),
            reinterpret_cast<hw_device_t**>(&device));
    ...

    //交叉检查Device的版本
    if (device->common.version < CAMERA_DEVICE_API_VERSION_3_0) {
        SET_ERR_L("Could not open camera: "
                "Camera device should be at least %x, reports %x instead",
                CAMERA_DEVICE_API_VERSION_3_0,
                device->common.version);
        device->common.close(&device->common);
        return BAD_VALUE;
    }
    ...

    //调用回调函数来进行初始化，即调用打开Device的initialize方法来进行初始化
    res = device->ops->initialize(device, this);
    ...

    //启动请求队列线程
    mRequestThread = new RequestThread(this, mStatusTracker, device, aeLockAvailable);
    res = mRequestThread->run(String8::format("C3Dev-%d-ReqQueue", mId).string());
    if (res != OK) {
        SET_ERR_L("Unable to start request queue thread: %s (%d)",
                strerror(-res), res);
        device->common.close(&device->common);
        mRequestThread.clear();
        return res;
    }
    ...
    //返回初始成功
    return OK;
}
```
首先，会依赖HAL框架打开并获得相应的Device对象，具体的流程请参考android6.0源码分析之Camera2 HAL分析，然后再回调此对象的initialize方法进行初始化，最后再启动RequestThread等线程，并返回initialize成功。至此Camera API2.0下的初始化过程就分析结束了。框架层的初始化时序图如下： 

![Alt text](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/camera.system/01-17-camera_open_native.png)

##### 4.2、总结
Open()过程道路艰辛，主要为了后续Camera正常工作，添砖加瓦，铺路。下面我们列举一下，主要都准备了什么。
1、Camera应用将一些Callback函数，注册到Camera.java中，以使在线程处理函数中可以调用到相应的回调函数。
2、camera connect成功后，创建了BpCamera代理对象和BnCameraClient本地对象。
3、在JNICameraContext实现CameraListener接口，并将接口注册到客户端camera本地对象中，并在BnCameraClient本地对象中回调这些接口。
4、CameraService connect过程中，根据hal硬件版本，创建对应的CameraClient对象。在后续的初始化过程中，创建6大线程。 
最后以一个简单的工作流程图来结束博文 

![Alt text](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/camera.system/01-18-Camera_open_overview.png)


#### （五）、参考资料(特别感谢各位前辈的分析和图示)：
[Android Camera官方文档](https://source.android.com/devices/camera/)
[Android 5.0 Camera系统源码分析-CSDN博客](https://blog.csdn.net/eternity9255)
[Android Camera 流程学习记录 - StoneDemo - 博客园](http://www.cnblogs.com/stonedemo/category/1080451.html)
[Android Camera 系统架构源码分析 - CSDN博客](https://blog.csdn.net/shell812/article/category/5905525)
[Camera安卓源码-高通mm_camera架构剖析 - CSDN博客](https://blog.csdn.net/hbw1992322/article/details/75259311)
[5.2 应用程序和驱动程序中buffer的传输流程 - CSDN博客](https://blog.csdn.net/yanbixing123/article/details/52294305/)
[Camera2 数据流从framework到Hal源码分析 - 简书](https://www.jianshu.com/p/ecb1be82e6a8)
[mm-camera层frame数据流源码分析 - 简书](https://www.jianshu.com/p/1baad2a5281d)
[v4l2_capture.c分析---probe函数分析 - CSDN博客](https://blog.csdn.net/yanbixing123/article/month/2016/08/4?)
[@@Android Camera fw学习 - CSDN博客](https://blog.csdn.net/armwind/article/category/6282972)
[@@Android Camera API2分析 - CSDN博客](https://blog.csdn.net/gzzaigcnforever/article/category/3066721)
[@@Android Camera 流程学习记录 7.12- CSDN博客](https://blog.csdn.net/qq_16775897/article/category/7112759)
[@@专栏：古冥的android6.0下的Camera API2.0的源码分析之旅 - CSDN博客](https://blog.csdn.net/yangzhihuiguming/article/details/51382267)
[linux3.3 v4l2视频采集驱动框架(vfe, camera i2c driver，v4l2_subdev等之间的联系) - CSDN博客](https://blog.csdn.net/gzzaigcnforever/article/details/17751109)
[Android Camera从Camera HAL1到Camera HAL3的过渡（已更新到Android6.0 HAL3.3） - CSDN博客](https://blog.csdn.net/gzzaigcnforever/article/details/48974523)
[我心依旧之Android Camera模块FW/HAL3探学序 - CSDN博客](https://blog.csdn.net/gzzaigcnforever/article/details/48972477)
[Android Camera fw学习(四)-recording流程分析 - CSDN博客](https://blog.csdn.net/armwind/article/details/73709808)
[android camera动态库加载过程 - CSDN博客](https://blog.csdn.net/armwind/article/details/52076879)
[Android Camera API2.0下全新的Camera FW/HAL架构简述 - CSDN博客](https://blog.csdn.net/gzzaigcnforever/article/details/49468475)
