---
title: Android 10 Camera源码分析6：Camera HAL3简介 && Camera Provider分析
cover: https://raw.githubusercontent.com/zhoujinjianOS/PicGo/master/post.cover.pictures/bing-wallpaper-2018.04.36.jpg
categories: 
 - Camera
tags:
 - Camera
toc: true
abbrlink: 20220315
date: 2022-03-15 03:15:00
---

注：文章都是通过阅读各位前辈总结的资料 Android 10.0 && Linux（Kernel 4.19）Rockchip平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除（◕‿◕），转载请注明出处（©Rockchip ©Android @Linux 版权所有），谢谢。

（==**文章基于 Kernel-4.19**==）&&（==**文章基于 Android 10.0**==）

[【zhoujinjianOS.com博客原图链接】](https://github.com/zhoujinjianOS) 

[【开发板 Khadas Edge V】](https://www.khadas.cn/edge-v)

[【开发板 Khadas Edge V Android 10.0 && Linux（Kernel 4.19）源码链接】](https://github.com/khadas/linux/tree/khadas-edge-Qt)



正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢！！！

 [Android Camera HAL3简介](https://deepinout.com/android-camera/android-camera-hal3-intro.html) 

 [Android P之Camera HAL3流程分析（0）](https://blog.csdn.net/Vincentywj/article/details/86759599) 

 [[Android O] Camera 服务启动流程简析](https://blog.csdn.net/qq_16775897/article/details/81240600) 

 [Android Camera Provider启动流程](https://blog.csdn.net/weixin_41944449/article/details/99453461) 

 [【Android10.0 进程间通信系列】](https://blog.csdn.net/yiranfeng/category_9850785.html)

 [【Android P HIDL 之 CameraProvider】](https://blog.csdn.net/a185531353/article/details/93223686)


--------------------------------------------------------------------------------

==源码（部分）==：

**F:\Khadas_Edge_Android_Q\system\hwservicemanager\**

**F:\Khadas_Edge_Android_Q\hardware\interfaces\camera\**

**F:\Khadas_Edge_Android_Q\system\libhidl\**

**/Khadas_Edge_Android_Q/out/soong/.intermediates/hardware/interfaces/camera/provider/2.4/android.hardware.camera.provider@2.4_genc++/gen/android/hardware/camera/**

==源码（部分）==：

--------------------------------------------------------------------------------



始于谷歌的Treble开源项目，基于接口与实现的分离的设计原则，谷歌加入了Camera Provider这一抽象层，该层作为一个独立进程存在于整个系统中，并且通过HIDL这一自定义语言成功地将Camera Hal Module从Camera Service中解耦出来，承担起了对Camera HAL的封装工作，纵观整个Android系统，对于Camera Provider而言，对上是通过HIDL接口负责与Camera Service的跨进程通信，对下通过标准的HAL3接口下发针对Camera的实际操作，这俨然是一个中央枢纽般的调配中心的角色，而事实上正是如此，由此看来，对Camera Provider的梳理变得尤为重要，接下来就以我个人理解出发来简单介绍下Camera Provider。

Camera Provider通过提供标准的HIDL接口给Camera Service进行调用，保持与Service的正常通信，其中谷歌将HIDL接口的定义直接暴露给平台厂商进行自定义实现，其中为了极大地减轻并降低开发者的工作量和开发难度，谷歌很好地封装了其跨进程实现细节，同样地，Camera Provider通过标准的HAL3接口，向下控制着具体的Camera HAL Module，而这个接口依然交由平台厂商负责去实现，而进程内部则通过简单的函数调用，将HIDL接口与HAL3接口完美的衔接起来，由此构成了Provider整体架构。

![](https://raw.githubusercontent.com/zhoujinjianOS/PicGo/master/Android.10.Camera.06/camera_provider_arc.png)

由图中可以看出Camera Provider进程由两部分组成，一是运行在系统中的主程序通过提供了标准的HIDL接口保持了与Camera Service的跨进程通讯，二是为了进一步扩展其功能，通过dlopen方式加载了一系列So库，而其中就包括了实现了Camera HAL3接口的So库，而HAL3接口主要定义了主要用于实现图像控制的功能，其实现主要交由平台厂商或者开发者来完成，所以Camera HAL3 So库的实现各式各样，在Rockchip平台上，这里的实现就是camera.rk30board.so。

在开始梳理Rockchip Camera HAL3 So库实现之前，不防先从上到下，以接口为主线简单梳理下Camera Provider的各个部分:



## （一）、Camera HIDL 接口

首先需要明确一个概念，就是HIDL是一种自定义语言，其核心是接口的定义，而谷歌为了使开发者将注意力落在接口的定义上而不是机制的实现上，主动封装了HIDL机制的实现细节，开发者只需要通过*.hal文件定义接口，填充接口内部实际的实现即可，接下来来看下具体定义的几个主要接口：

![](https://raw.githubusercontent.com/zhoujinjianOS/PicGo/master/Android.10.Camera.06/cmaera_hidl.png)

因为HIDL机制本身是跨进程通讯的，所以Camera Service本身通过HIDL接口获取的对象都会有Bn端和Bp端，分别代表了Binder两端，接下来为了方便理解，我们都省略掉Bn/Bp说法,直接用具体接口类代表，忽略跨进程两端的区别。

#### （1）、ICameraProvider.hal

``` cpp
F:\Khadas_Edge_Android_Q\hardware\interfaces\camera\provider\2.4\ICameraProvider.hal
package android.hardware.camera.provider@2.4;

import ICameraProviderCallback;
import android.hardware.camera.common@1.0::types;
import android.hardware.camera.device@1.0::ICameraDevice;
import android.hardware.camera.device@3.2::ICameraDevice;

interface ICameraProvider {

    /**
     * setCallback:
     * Provide a callback interface to the HAL provider to inform framework of
     * asynchronous camera events. The framework must call this function once
     * during camera service startup, before any other calls to the provider
     * (note that in case the camera service restarts, this method must be
     * invoked again during its startup).
     */
    setCallback(ICameraProviderCallback callback) generates (Status status);

    /**
     * getVendorTags:
     * Retrieve all vendor tags supported by devices discoverable through this
     * provider. The tags are grouped into sections.
     */
    getVendorTags() generates (Status status, vec<VendorTagSection> sections);

    /**
     * getCameraIdList:
     * Returns the list of internal camera device interfaces known to this
     * camera provider. These devices can then be accessed via the hardware
     * service manager.
     */
    getCameraIdList()
            generates (Status status, vec<string> cameraDeviceNames);

    /**
     * isSetTorchModeSupported:
     * Returns if the camera devices known to this camera provider support
     * setTorchMode API or not. If the provider does not support setTorchMode
     * API, calling to setTorchMode will return METHOD_NOT_SUPPORTED.
     */
    isSetTorchModeSupported() generates (Status status, bool support);

    /**
     * getCameraDeviceInterface_VN_x:
     * Return a android.hardware.camera.device@N.x/ICameraDevice interface for
     * the requested device name. This does not power on the camera device, but
     * simply acquires the interface for querying the device static information,
     * or to additionally open the device for active use.
     *
     * A separate method is required for each major revision of the camera device
     * HAL interface, since they are not compatible with each other.
     *
     * Valid device names for this provider can be obtained via either
     * getCameraIdList(), or via availability callbacks from
     * ICameraProviderCallback::cameraDeviceStatusChange().
     */
    getCameraDeviceInterface_V1_x(string cameraDeviceName) generates
            (Status status,
             android.hardware.camera.device@1.0::ICameraDevice device);
    getCameraDeviceInterface_V3_x(string cameraDeviceName) generates
            (Status status,
             android.hardware.camera.device@3.2::ICameraDevice device);

};

```

该文件中定义了ICameraProvider接口类，由CameraProvider继承并实现，在Camera Provider启动的时候被实例化，主要接口如下：

- getCameraDeviceInterface_V3_x: 该方法主要用于Camera Service获取ICameraDevice，通过该对象可以控制Camera 设备的诸如配置数据流、下发request等具体行为。
- setCallback： 将Camera Service 实现的ICameraProviderCallback传入Camera Provider，一旦Provider有事件产生时便可以通过该对象通知Camera Service。

#### （2）、ICameraProviderCallback.hal

```c
F:\Khadas_Edge_Android_Q\hardware\interfaces\camera\provider\2.4\ICameraProviderCallback.hal
package android.hardware.camera.provider@2.4;

import android.hardware.camera.common@1.0::types;
interface ICameraProviderCallback {

    /**
     * cameraDeviceStatusChange:
     *
     * Callback to the camera service to indicate that the state of a specific
     * camera device has changed.
     *
     * On camera service startup, when ICameraProvider::setCallback is invoked,
     * the camera service must assume that all internal camera devices are in
     * the CAMERA_DEVICE_STATUS_PRESENT state.
     *
     * The provider must call this method to inform the camera service of any
     * initially NOT_PRESENT devices, and of any external camera devices that
     * are already present, as soon as the callbacks are available through
     * setCallback.
     */
    cameraDeviceStatusChange(string cameraDeviceName,
            CameraDeviceStatus newStatus);

    /**
     * torchModeStatusChange:
     *
     * Callback to the camera service to indicate that the state of the torch
     * mode of the flash unit associated with a specific camera device has
     * changed. At provider registration time, the camera service must assume
     * the torch modes are in the TORCH_MODE_STATUS_AVAILABLE_OFF state if
     * android.flash.info.available is reported as true via the
     * ICameraDevice::getCameraCharacteristics call.
     */
    torchModeStatusChange(string cameraDeviceName,
            TorchModeStatus newStatus);

};


```

该文件中定义了ICameraProviderCallback回调接口类，该接口由Camera Service 中的CameraProviderManager::ProviderInfo继承并实现，在Camera Service 启动的时候被实例化，通过调用ICameraProvider::setCallback接口注册到Camera Provider中，其主要接口如下：

- cameraDeviceStatusChange： 将Camera 设备状态上传至Camera Service，状态由CameraDeviceStatus定义

#### （3）、ICameraDevice.hal

```cpp
ackage android.hardware.camera.device@3.2;

import android.hardware.camera.common@1.0::types;
import ICameraDeviceSession;
import ICameraDeviceCallback;

interface ICameraDevice {
    /**
     * Get camera device resource cost information.
     */
    getResourceCost() generates (Status status, CameraResourceCost resourceCost);

    /**
     * getCameraCharacteristics:
     * Return the static camera information for this camera device. This
     * information may not change between consecutive calls.
     */
    getCameraCharacteristics() generates
            (Status status, CameraMetadata cameraCharacteristics);

    /**
     * setTorchMode:
     * Turn on or off the torch mode of the flash unit associated with this
     * camera device. If the operation is successful, HAL must notify the
     * framework torch state by invoking
     * ICameraProviderCallback::torchModeStatusChange() with the new state.
     */
    setTorchMode(TorchMode mode) generates (Status status);

    /**
     * open:
     * Power on and initialize this camera device for active use, returning a
     * session handle for active operations.
     */
    open(ICameraDeviceCallback callback) generates
            (Status status, ICameraDeviceSession session);
    dumpState(handle fd);

};

```

该文件中定义了ICameraDevice接口类，由CameraDevice::TrampolineDeviceInterface_3_2实现，其主要接口如下:

- open： 用于创建一个Camera设备，并且将Camera Service中继承ICameraDeviceCallback并实现了相应接口的Camera3Device作为参数传入Provider中，供Provider上传事件或者图像数据。

- getCameraCharacteristics：用于获取Camera设备的属性。

#### （4）、ICameraDeviceCallback.hal

```cpp
package android.hardware.camera.device@3.2;

import android.hardware.camera.common@1.0::types;

interface ICameraDeviceCallback {

    /**
     * processCaptureResult:
     *
     * Send results from one or more completed or partially completed captures
     * to the framework.
     * processCaptureResult() may be invoked multiple times by the HAL in
     * response to a single capture request. This allows, for example, the
     * metadata and low-resolution buffers to be returned in one call, and
     * post-processed JPEG buffers in a later call, once it is available. Each
     * call must include the frame number of the request it is returning
     * metadata or buffers for. Only one call to processCaptureResult
     * may be made at a time by the HAL although the calls may come from
     * different threads in the HAL.
     *
     * A component (buffer or metadata) of the complete result may only be
     * included in one process_capture_result call. A buffer for each stream,
     * and the result metadata, must be returned by the HAL for each request in
     * one of the processCaptureResult calls, even in case of errors producing
     * some of the output. A call to processCaptureResult() with neither
     * output buffers or result metadata is not allowed.
     *
     * The order of returning metadata and buffers for a single result does not
     * matter, but buffers for a given stream must be returned in FIFO order. So
     * the buffer for request 5 for stream A must always be returned before the
     * buffer for request 6 for stream A. This also applies to the result
     * metadata; the metadata for request 5 must be returned before the metadata
     * for request 6.
     */
    processCaptureResult(vec<CaptureResult> results);

    /**
     * notify:
     * Asynchronous notification callback from the HAL, fired for various
     * reasons. Only for information independent of frame capture, or that
     * require specific timing. Multiple messages may be sent in one call; a
     * message with a higher index must be considered to have occurred after a
     * message with a lower index.
     */
    notify(vec<NotifyMsg> msgs);

};

```

该文件中定义了ICameraDeviceCallback接口类，由Camera Service中的Camera3Device继承并实现，通过调用ICameraDevice::open方法注册到Provider中，其主要接口如下：

- processCaptureResult: 一旦有图像数据产生会通过调用该方法将数据以及meta data上传至Camera Service。
- notify: 通过该方法上传事件至Camera Service中，比如shutter事件等。

#### （5）、ICameraDeviceSession.hal

```c
F:\Khadas_Edge_Android_Q\hardware\interfaces\camera\device\3.3\ICameraDeviceSession.hal
    
package android.hardware.camera.device@3.3;

import android.hardware.camera.common@1.0::Status;
import android.hardware.camera.device@3.2::ICameraDeviceSession;
import android.hardware.camera.device@3.2::StreamConfiguration;

/**
 * Camera device active session interface.
 *
 * Obtained via ICameraDevice::open(), this interface contains the methods to
 * configure and request captures from an active camera device.
 *
 */
interface ICameraDeviceSession extends @3.2::ICameraDeviceSession {

    /**
     * configureStreams_3_3:
     *
     * Identical to @3.2::ICameraDeviceSession.configureStreams, except that:
     *
     * - The output HalStreamConfiguration now contains an overrideDataspace
     *   field, to be used by the HAL to select a different dataspace for some
     *   use cases when dealing with the IMPLEMENTATION_DEFINED pixel format.
     *
     * Clients may invoke either this method or
     * @3.2::ICameraDeviceSession.configureStreams() for stream configuration.
     * This method is recommended for clients to use since it provides more
     * flexibility.
     */
    configureStreams_3_3(StreamConfiguration requestedConfiguration)
            generates (Status status,
                    @3.3::HalStreamConfiguration halConfiguration);

};
F:\Khadas_Edge_Android_Q\hardware\interfaces\camera\device\3.2\ICameraDeviceSession.hal
    
package android.hardware.camera.device@3.2;

import android.hardware.camera.common@1.0::types;

/**
 * Camera device active session interface.
 * Obtained via ICameraDevice::open(), this interface contains the methods to
 * configure and request captures from an active camera device.
 *
 */
interface ICameraDeviceSession {

    /**
     * constructDefaultRequestSettings:
     * Create capture settings for standard camera use cases.
     */
    constructDefaultRequestSettings(RequestTemplate type) generates
            (Status status, CameraMetadata requestTemplate);

    /**
     * configureStreams:
     * Reset the HAL camera device processing pipeline and set up new input and
     * output streams. This call replaces any existing stream configuration with
     * the streams defined in the streamList. This method must be called at
     * least once before a request is submitted with processCaptureRequest().
     */
    configureStreams(StreamConfiguration requestedConfiguration)
            generates (Status status,
                    HalStreamConfiguration halConfiguration);

    /**
     * processCaptureRequest:
     * Send a list of capture requests to the HAL. The HAL must not return from
     * this call until it is ready to accept the next set of requests to
     * process. Only one call to processCaptureRequest() must be made at a time
     * by the framework, and the calls must all be from the same thread. The
     * next call to processCaptureRequest() must be made as soon as a new
     * request and its associated buffers are available. In a normal preview
     * scenario, this means the function is generally called again by the
     * framework almost instantly. If more than one request is provided by the
     * client, the HAL must process the requests in order of lowest index to
     * highest index.
     */
    processCaptureRequest(vec<CaptureRequest> requests,
            vec<BufferCache> cachesToRemove)
            generates (Status status, uint32_t numRequestProcessed);

    /**
     * getCaptureRequestMetadataQueue:
     * Retrieves the queue used along with processCaptureRequest. If
     * client decides to use fast message queue to pass request metadata,
     */
    getCaptureRequestMetadataQueue() generates (fmq_sync<uint8_t> queue);

    /**
     * getCaptureResultMetadataQueue:
     * Retrieves the queue used along with
     * ICameraDeviceCallback.processCaptureResult.
     */
    getCaptureResultMetadataQueue() generates (fmq_sync<uint8_t> queue);

    /**
     * flush:
     * Flush all currently in-process captures and all buffers in the pipeline
     * on the given device. Generally, this method is used to dump all state as
     * quickly as possible in order to prepare for a configure_streams() call.
     */
    flush() generates (Status status);

    /**
     * close:
     * Shut down the camera device.
     */
    close();
};

```

该文件中定义了ICameraDeviceSession接口类，由CameraDeviceSession::TrampolineSessionInterface_3_3继承并实现，其主要接口如下：

- constructDefaultRequestSettings：用于创建默认的Request配置项。
- configureStreams：用于配置数据流，其中包括了output buffer/Surface/图像格式大小等属性。
- processCaptureRequest：下发request到Provider中，一个request对应着一次图像需求。
- close: 关闭当前会话。

## （二）、Camera Provider 主程序

接下来进入到Provider内部去看看，整个进程是如何运转的，这个服务进程的启动很简单，主要动作是注册该 CameraProvider，以便 CameraServer 启动时能找到它。需要注意的是，此时 CameraProvider 还未实例化与初始化。以下图为例进行分析:

![](https://raw.githubusercontent.com/zhoujinjianOS/PicGo/master/Android.10.Camera.06/android.hardware.camera.provider@2.4-service.png)

```
F:\Khadas_Edge_Android_Q\hardware\interfaces\camera\provider\2.4\default\android.hardware.camera.provider@2.4-service_64.rc
service vendor.camera-provider-2-4 /vendor/bin/hw/android.hardware.camera.provider@2.4-service
    class hal
    user cameraserver
    group audio camera input drmrpc
    ioprio rt 4
    capabilities SYS_NICE
    writepid /dev/cpuset/camera-daemon/tasks /dev/stune/top-app/tasks
```

在android启动的过程中，init进程调用该脚本启动 camera provider 服务。根据该目录下的 Android.bp 可以知道，其实就是运行该目录下 service.cpp 编译的可执行文件，service.cpp 内容如下：

```cpp
F:\Khadas_Edge_Android_Q\hardware\interfaces\camera\provider\2.4\default\service.cpp

int main()
{
    ALOGI("CameraProvider@2.4 legacy service is starting.");
    // The camera HAL may communicate to other vendor components via
    // /dev/vndbinder
    android::ProcessState::initWithDriver("/dev/vndbinder");
    status_t status;
    if (kLazyService) {
        status = defaultLazyPassthroughServiceImplementation<ICameraProvider>("legacy/0",
                                                                              /*maxThreads*/ 6);
    } else {
        status = defaultPassthroughServiceImplementation<ICameraProvider>("legacy/0",
                                                                          /*maxThreads*/ 6);
    }
    return status;
}

```

根据以上代码可以得知：

- android::ProcessState::initWithDriver ：camera HAL 通过 /dev/vndbinder 驱动可与其他模块的HAL进行通信
- defaultPassthroughServiceImplementation ：创建默认为直通模式（passthrough）的 CameraProvider 服务；

```cpp
F:\Khadas_Edge_Android_Q\system\libhidl\transport\include\hidl\LegacySupport.h
template <class Interface, typename Func>
__attribute__((warn_unused_result)) status_t registerPassthroughServiceImplementation(
    Func registerServiceCb, const std::string& name = "default") {
    /* 获得CameraProvider实例化对象（不是Binder代理）,（此处的name为 “legacy/0”） */
    sp<Interface> service = Interface::getService(name, true /* getStub */);
    ......
   /* 将 CameraProvider 注册为一个服务，其他进程需要使用camera HAL层时，通过Binder
     * 得到 CameraProvider 代理类即可操作 camera HAL层，不需要每次都dlopen(HAL.so)
     * */
    status_t status = registerServiceCb(service, name);

    if (status == OK) {
        ALOGI("Registration complete for %s/%s.",
            Interface::descriptor, name.c_str());
    } else {
        ALOGE("Could not register service %s/%s (%d).",
            Interface::descriptor, name.c_str(), status);
    }

    return status;
}
}  // namespace details
template <class Interface>
__attribute__((warn_unused_result)) status_t defaultPassthroughServiceImplementation(
    const std::string& name, size_t maxThreads = 1) {
    configureRpcThreadpool(maxThreads, true);
    status_t result = registerPassthroughServiceImplementation<Interface>(name);
    ......
    joinRpcThreadpool();
    return UNKNOWN_ERROR;
}
template<class Interface>
__attribute__((warn_unused_result))
status_t defaultPassthroughServiceImplementation(size_t maxThreads = 1) {
    return defaultPassthroughServiceImplementation<Interface>("default", maxThreads);
}
```

#### （1）、获取 CameraProvider 实例对象

从 LegacySupport.h 可以知道，defaultPassthroughServiceImplementation为模板类函数，将会通过 ==sp<ICameraProvider> service = ICameraProvider::getService(name, true /* getStub */)== 获取 CameraProvider 实例化对象，以上操作，将会进入 CameraProviderAll.cpp。

```cpp
/Khadas_Edge_Android_Q/out/soong/.intermediates/hardware/interfaces/camera/provider/2.4/android.hardware.camera.provider@2.4_genc++/gen/android/hardware/camera/provider/2.4/CameraProviderAll.cpp
    
::android::sp<ICameraProvider> ICameraProvider::getService(const std::string &serviceName, const bool getStub) {
    return ::android::hardware::details::getServiceInternal<BpHwCameraProvider>(serviceName, true, getStub);
}

```



```cpp
F:\Khadas_Edge_Android_Q\system\libhidl\transport\include\hidl\HidlTransportSupport.h
template <typename BpType, typename IType = typename BpType::Pure,
          typename = std::enable_if_t<std::is_same<i_tag, typename IType::_hidl_tag>::value>,
          typename = std::enable_if_t<std::is_same<bphw_tag, typename BpType::_hidl_tag>::value>>
sp<IType> getServiceInternal(const std::string& instance, bool retry, bool getStub) {
    using ::android::hidl::base::V1_0::IBase;

    sp<IBase> base = getRawServiceInternal(IType::descriptor, instance, retry, getStub);
    ......
    if (base->isRemote()) {
        // getRawServiceInternal guarantees we get the proper class
        return sp<IType>(new BpType(getOrCreateCachedBinder(base.get())));
    }

    return IType::castFrom(base);
}

```

其中，参数 BpHwCameraProvider::descriptor 为android.hardware.camera.provider@2.4::ICameraProvider，instance 为 “legacy/0” ，retry 为 true，getStub 为 true。

```cpp
F:\Khadas_Edge_Android_Q\system\libhidl\transport\ServiceManagement.cpp
sp<::android::hidl::base::V1_0::IBase> getRawServiceInternal(const std::string& descriptor,
                                                             const std::string& instance,
                                                             bool retry, bool getStub) {
    ...

	/* getStub 为 true，直通模式，将返回CameraProvider实例对象 */
	if (getStub || vintfPassthru || vintfLegacy) {
        /* 获取ServiceManager代理 */
        const sp<IServiceManager> pm = getPassthroughServiceManager();
        if (pm != nullptr) {
            /* 获取CameraProvider实例对象 */
            sp<IBase> base = pm->get(descriptor, instance).withDefault(nullptr);
            if (!getStub || trebleTestingOverride) {
                base = wrapPassthrough(base);
            }
            return base;
        }
    }

    return nullptr;
}

struct PassthroughServiceManager : IServiceManager1_1 {
    static void openLibs(
        const std::string& fqName,
        const std::function<bool /* continue */ (void* /* handle */, const std::string& /* lib */,
                                                 const std::string& /* sym */)>& eachLib) {
        //fqName looks like android.hardware.foo@1.0::IFoo
        /* fqName = android.hardware.camera.provider@2.4::ICameraProvider */
        size_t idx = fqName.find("::");

        if (idx == std::string::npos ||
                idx + strlen("::") + 1 >= fqName.size()) {
            LOG(ERROR) << "Invalid interface name passthrough lookup: " << fqName;
            return;
        }

        std::string packageAndVersion = fqName.substr(0, idx);
        std::string ifaceName = fqName.substr(idx + strlen("::"));

        /* prefix = android.hardware.camera.provider@2.4-impl */
        const std::string prefix = packageAndVersion + "-impl";
        /* sym = HIDL_FETCH_ICameraProvider */
        const std::string sym = "HIDL_FETCH_" + ifaceName;

        constexpr int dlMode = RTLD_LAZY;
        void* handle = nullptr;

        dlerror(); // clear

        static std::string halLibPathVndkSp = android::base::StringPrintf(
            HAL_LIBRARY_PATH_VNDK_SP_FOR_VERSION, details::getVndkVersionStr().c_str());
        std::vector<std::string> paths = {HAL_LIBRARY_PATH_ODM, HAL_LIBRARY_PATH_VENDOR,
                                          halLibPathVndkSp, HAL_LIBRARY_PATH_SYSTEM};

        for (const std::string& path : paths) {
            std::vector<std::string> libs = search(path, prefix, ".so");

            for (const std::string &lib : libs) {
                const std::string fullPath = path + lib;

                /* 经过上面的一些添加转换，最终
                 * fullPath = /vendor/lib/hw/android.hardware.camera.provider@2.4-impl.so
                 * */
                if (path == HAL_LIBRARY_PATH_SYSTEM) {
                    handle = dlopen(fullPath.c_str(), dlMode);
                } else {
                    handle = android_load_sphal_library(fullPath.c_str(), dlMode);
                }
                ......
                /* Lambda表达式代入 */
                if (!eachLib(handle, lib, sym)) {
                    return;
                }
            }
        }
    }

    Return<sp<IBase>> get(const hidl_string& fqName,
                          const hidl_string& name) override {
        sp<IBase> ret = nullptr;

        /* [&] 此处为Lambda表达式，简单理解为函数指针即可，先执行 openLibs() */
        openLibs(fqName, [&](void* handle, const std::string &lib, const std::string &sym) {
            /* handle ：dlopen() 的返回值
             * lib ：android.hardware.camera.provider@2.4-impl.so
             * sym ：HIDL_FETCH_ICameraProvider
             */
            IBase* (*generator)(const char* name);
            /* 返回 HIDL_FETCH_ICameraProvider() 函数对应的函数地址 */
            *(void **)(&generator) = dlsym(handle, sym.c_str());
            if(!generator) {
                const char* error = dlerror();
                ......
                dlclose(handle);
                return true;
            }

            /* 执行 HIDL_FETCH_ICameraProvider() 函数，该函数返回CameraProvider实例对象保存在ret，
             * 所以get()函数将返回 ret 
             * */
            ret = (*generator)(name.c_str());

            if (ret == nullptr) {
                dlclose(handle);
                return true; // this module doesn't provide this instance name
            }

            // Actual fqname might be a subclass.
            // This assumption is tested in vts_treble_vintf_test
            using ::android::hardware::details::getDescriptor;
            std::string actualFqName = getDescriptor(ret.get());
            CHECK(actualFqName.size() > 0);
            registerReference(actualFqName, name);
            return false;
        });

        return ret;
    }
}
```

get() 函数传递进来的fpName为 android.hardware.camera.provider@2.4::ICameraProvider ，name为 legacy/0。

```cpp
F:\Khadas_Edge_Android_Q\hardware\interfaces\camera\provider\2.4\default\CameraProvider_2_4.cpp
    extern "C" ICameraProvider* HIDL_FETCH_ICameraProvider(const char* name);

template<typename IMPL>
CameraProvider<IMPL>* getProviderImpl() {
    CameraProvider<IMPL> *provider = new CameraProvider<IMPL>();
    ......
    return provider;
}

ICameraProvider* HIDL_FETCH_ICameraProvider(const char* name) {
    using namespace android::hardware::camera::provider::V2_4::implementation;
    ICameraProvider* provider = nullptr;
    /* 传递进来的 name 为 legacy/0，而 kLegacyProviderName 定义为 legacy/0 */
    if (strcmp(name, kLegacyProviderName) == 0) {
        /* 创建LegacyCameraProviderImpl_2_4对象，构造函数将会调用initialize() 函数 */
        provider = getProviderImpl<LegacyCameraProviderImpl_2_4>();
    } else if (strcmp(name, kExternalProviderName) == 0) {
        provider = getProviderImpl<ExternalCameraProviderImpl_2_4>();
    } else {
        ALOGE("%s: unknown instance name: %s", __FUNCTION__, name);
    }

    return provider;
}


 F:\Khadas_Edge_Android_Q\hardware\interfaces\camera\provider\2.4\default\LegacyCameraProviderImpl_2_4.cpp
     
     LegacyCameraProviderImpl_2_4::LegacyCameraProviderImpl_2_4() :
        camera_module_callbacks_t({sCameraDeviceStatusChange,
                                   sTorchModeStatusChange}) {
    mInitFailed = initialize();
}

bool LegacyCameraProviderImpl_2_4::initialize() {
    camera_module_t *rawModule;
    /* 在通过 hw_get_module() 加载HAL层so：其实是通过获取各种android属性
     * （在设备端可以通过 getprop 命令查看当前设备支持的属性），
     * 得到HAL so的名称（*variant_keys[]），而后探测、加载该HAL so库，并通
     * 过 dlsym() 函数返回标识符为 HAL_MODULE_INFO_SYM_AS_STR 的HMI地址
     * （由于各个HAL层代码最终会通过 HAL_MODULE_INFO_SYM 修饰，编译器识别到
     * 该符号时将会将标示地址导出为HMI符号，从而在加载HAL so时可以获取）
     */    
    int err = hw_get_module(CAMERA_HARDWARE_MODULE_ID,
            (const hw_module_t **)&rawModule);
    ......
    /* rawModule 将指向 HAL 中的 camera_module_t 类型结构体，
     * 此时，CameraProvider 与 camera HAL 绑定成功，可以通过
     * CameraProvider操作camera HAL
     */
    /* 创建 CameraModule 对象 */
    /* CameraModule.cpp：android/hardware/interfaces/camera/common/1.0/default */    
    mModule = new CameraModule(rawModule);
   /* mModule->init()主要完成以下操作：
     * 1. 当camera HAL的 module_api_version >= CAMERA_MODULE_API_VERSION_2_4，将调用HAL->init()
     * 2. 通过 HAL getNumberOfCameras() 获取设置camera数量，并将该参数设置为 mCameraInfoMap 容器的大小
     * */
    err = mModule->init();
    ......

    ALOGI("Loaded \"%s\" camera module", mModule->getModuleName());

    // Setup vendor tags here so HAL can setup vendor keys in camera characteristics
    VendorTagDescriptor::clearGlobalVendorTagDescriptor();
    if (!setUpVendorTags()) {
        ALOGE("%s: Vendor tag setup failed, will not be available.", __FUNCTION__);
    }

    // Setup callback now because we are going to try openLegacy next
    err = mModule->setCallbacks(this);
    ......
    mNumberOfLegacyCameras = mModule->getNumberOfCameras();
    /* 将获取camera信息并保存，其中将有HAL version 信息，应用
     * 层将会检查HAL层版本信息从而确认调用不同的API实现相机应用
     */    
    for (int i = 0; i < mNumberOfLegacyCameras; i++) {
        struct camera_info info;
        auto rc = mModule->getCameraInfo(i, &info);
        ......
        char cameraId[kMaxCameraIdLen];
        snprintf(cameraId, sizeof(cameraId), "%d", i);
        std::string cameraIdStr(cameraId);
        mCameraStatusMap[cameraIdStr] = CAMERA_DEVICE_STATUS_PRESENT;

        addDeviceNames(i);
    }

    return false; // mInitFailed
}
F:\Khadas_Edge_Android_Q\hardware\interfaces\camera\common\1.0\default\CameraModule.cpp
 CameraModule::CameraModule(camera_module_t *module) : mNumberOfCameras(0) {
    if (module == NULL) {
        ALOGE("%s: camera hardware module must not be null", __FUNCTION__);
        assert(0);
    }
    mModule = module;
}

int CameraModule::init() {
    ATRACE_CALL();
    int res = OK;
    if (getModuleApiVersion() >= CAMERA_MODULE_API_VERSION_2_4 &&
            mModule->init != NULL) {
        ATRACE_BEGIN("camera_module->init");
        res = mModule->init();
        ATRACE_END();
    }
    mNumberOfCameras = getNumberOfCameras();
    mCameraInfoMap.setCapacity(mNumberOfCameras);
    return res;
}

```

>  01-18 08:50:15.868   264   264 I CamPrvdr@2.4-legacy: Loaded "Rockchip Camera3HAL Module" camera module

至此，已获得CameraProvider实例对象，最终返回赋值给 registerPassthroughServiceImplementation() 函数中的 service 。

#### （2）、将 CameraProvider 注册为服务

在得到CameraProvider实例对象之后，将通过 service->registerAsService(name) 进行服务注册。

```cpp
/Khadas_Edge_Android_Q/out/soong/.intermediates/hardware/interfaces/camera/provider/2.4/android.hardware.camera.provider@2.4_genc++/gen/android/hardware/camera/provider/2.4/CameraProviderAll.cpp
    
    ::android::status_t ICameraProvider::registerAsService(const std::string &serviceName) {
    return ::android::hardware::details::registerAsServiceInternal(this, serviceName);
}
F:\Khadas_Edge_Android_Q\system\libhidl\transport\ServiceManagement.cpp
    status_t registerAsServiceInternal(const sp<IBase>& service, const std::string& name) {
    ......

    sp<IServiceManager1_2> sm = defaultServiceManager1_2();
    ......
    bool registered = false;
    Return<void> ret = service->interfaceChain([&](const auto& chain) {
        registered = sm->addWithChain(name.c_str(), service, chain).withDefault(false);
    });
    ......
    if (registered) {
        onRegistrationImpl(getDescriptor(service.get()), name);
    }

    return registered ? OK : UNKNOWN_ERROR;
}

F:\Khadas_Edge_Android_Q\system\libhidl\transport\ServiceManagement.cpp
    Return<bool> ServiceManager::addWithChain(const hidl_string& name,
                                          const sp<IBase>& service,
                                          const hidl_vec<hidl_string>& chain) {
    ......

    auto callingContext = getBinderCallingContext();

    return addImpl(name, service, chain, callingContext);
}
bool ServiceManager::addImpl(const std::string& name,
                             const sp<IBase>& service,
                             const hidl_vec<hidl_string>& interfaceChain,
                             const AccessControl::CallingContext& callingContext) {
    ......

    for(size_t i = 0; i < interfaceChain.size(); i++) {
        const std::string fqName = interfaceChain[i];

        PackageInterfaceMap &ifaceMap = mServiceMap[fqName];
        HidlService *hidlService = ifaceMap.lookup(name);

        if (hidlService == nullptr) {
            ifaceMap.insertService(
                std::make_unique<HidlService>(fqName, name, service, callingContext.pid));
        } else {
            hidlService->setService(service, callingContext.pid);
        }

        ifaceMap.sendPackageRegistrationNotification(fqName, name);
    }

    bool linkRet = service->linkToDeath(this, kServiceDiedCookie).withDefault(false);
    ......

    return true;
}
```

排除一切问题后，把需要注册的服务，注册到mServiceMap中，最后建立一个linkToDeath，当服务挂掉时会接收到通知，做一些清理工作。注册好服务之后，camerservice就可以获取到。

#### （3）、总结：

在系统初始化的时候，系统会去运行android.hardware.camera.provider@2.4-service程序启动Provider进程，并加入HW Service Manager中接受统一管理，在该过程中实例化了一个LegacyCameraProviderImpl_2_4对象，并在其构造函数中通过hw_get_module标准方法获取HAL的camera_module_t结构体,并将其存入CameraModule对象中，之后通过调用该camera_modult_t结构体的init方法初始化HAL Module，紧接着调用其get_number_of_camera方法获取当前HAL支持的Camera数量，最后通过调用其set_callbacks方法将LegcyCameraProviderImpl_2_4（LegcyCameraProviderImpl_2_4继承了camera_modult_callback_t）作为参数传入camera.rk30board.so中，接受来自camera.rk30board.so中的数据以及事件，当这一系列动作完成了之后，Camera Provider进程便一直便存在于系统中，监听着来自Camera Service的调用。



![](https://raw.githubusercontent.com/zhoujinjianOS/PicGo/master/Android.10.Camera.06/Camera_Provider_Interface.png)

接下来以上图为例简单介绍下Provider中几个重要流程：

- Camera Service通过调用ICameraProvider的getCameraDeviceInterface_v3_x接口获取ICameraDevice，在此过程中，Provider会去实例化一个CameraDevice对象，并且将之前存有camera_modult_t结构体的CameraModule对象传入CameraDevice中，这样就可以在CameraDevice内部通过CameraModule访问到camera_module_t的相关资源，然后将CameraDevice内部类TrampolineSessionInterface_3_3（该类继承并实现了ICameraDevice接口）返回给Camera Service。
- Camera Service通过之前获取的ICameraDevice，调用其open方法来打开Camera设备，接着在Provider中会去调用CameraDevice对象的open方法，在该方法内部会去调用camera_module_t结构体的open方法，从而获取到HAL部分的camera3_device_t结构体，紧接着Provider会实例化一个CameraDeviceSession对象，并且将刚才获取到的camera3_device_t结构体以参数的方式传入CameraDeviceSession中，在CameraDeviceSession的构造方法中又会调用CameraDeviceSession的initialize方法，在该方法内部又会去调用camera3_device_t结构体的ops内的initialize方法开始HAL部分的初始化工作，最后CameraDeviceSession对象被作为camera3_callback_ops的实现传入HAL，接收来自HAL的数据或者具体事件，当一切动作都完成后，Provider会将CameraDeviceSession::TrampolineDeviceInterface_3_2（该类继承并实现了ICameraDeviceSession接口）对象通过HIDL回调的方法返回给Camera Service中。

- Camera Service通过调用ICameraDevcieSession的configureStreams_3_3接口进行数据流的配置，在Provider中，最终会通过调用之前获取的camera3_device_t结构体内ops的configure_streams方法下发到HAL中进行处理。
- Camera Service通过调用ICameraDevcieSession的processCaptureRequest接口下发request请求到Provider中，在Provider中，最终依然会通过调用获取的camera3_device_t结构体内ops中的process_capture_request方法将此次请求下发到HAL中进行处理。

从整个流程不难看出，这几个接口最终对应的是HAL3的接口，并且Provider并没有经过太多复杂的额外的处理。

## （三）、Camera HAL3 接口

HAL硬件抽象层(Hardware Abstraction Layer)，是谷歌开发的用于屏蔽底层硬件抽象出来的一个软件层， 每一个平台厂商可以将不开源的代码封装在这一层，仅仅提供二进制文件。
该层定义了自己的一套通用标准接口，平台厂商务必按照以下规则定义自己的Module:

- 每一个硬件模块都通过hw_module_t来描述，具有固定的名字HMI
- 每一个硬件模块都必须实现hw_module_t里面的open方法，用于打开硬件设备，并返回对应的操作接口集合
- 硬件的操作接口集合使用hw_device_t 来描述，并可以通过自定义一个更大的包含hw_device_t的结构体来拓展硬件操作集合

其中代表硬件模块的是hw_module_t，对应的设备是通过hw_device_t来描述，这两者的定义如下：

```cpp
F:\Khadas_Edge_Android_Q\hardware\libhardware\include\hardware\hardware.h
    typedef struct hw_module_t {
    /** tag must be initialized to HARDWARE_MODULE_TAG */
    uint32_t tag;
    /**
     * The API version of the implemented module. The module owner is
     * responsible for updating the version when a module interface has
     * changed.
     */
    uint16_t module_api_version;
#define version_major module_api_version
    uint16_t hal_api_version;
#define version_minor hal_api_version

    /** Identifier of module */
    const char *id;

    /** Name of this module */
    const char *name;

    /** Author/owner/implementor of the module */
    const char *author;

    /** Modules methods */
    struct hw_module_methods_t* methods;

    /** module's dso */
    void* dso;

#ifdef __LP64__
    uint64_t reserved[32-7];
#else
    /** padding to 128 bytes, reserved for future use */
    uint32_t reserved[32-7];
#endif

} hw_module_t;

typedef struct hw_module_methods_t {
    /** Open a specific device */
    int (*open)(const struct hw_module_t* module, const char* id,
            struct hw_device_t** device);

} hw_module_methods_t;

typedef struct hw_device_t {
    uint32_t tag;
    uint32_t version;

    /** reference to the module this device belongs to */
    struct hw_module_t* module;

    /** padding reserved for future use */
#ifdef __LP64__
    uint64_t reserved[12];
#else
    uint32_t reserved[12];
#endif

    /** Close this device */
    int (*close)(struct hw_device_t* device);

} hw_device_t;

```

从上面的定义可以看出，主要是通过hw_module_t 代表了模块，通过其open方法用来打开一个设备，而该设备是用hw_device_t来表示，其中除了用来关闭设备的close方法外，并无其它方法，由此可见谷歌定义的HAL接口，并不能满足绝大部分HAL模块的需要，所以谷歌想出了一个比较好的解决方式，那便是将这两个基本结构嵌入到更大的结构体内部，同时在更大的结构内部定义了各自模块特有的方法，用于实现模块的功能，这样，一来对上保持了HAL的统一规范，二来也扩展了模块的功能。

基于上面的方式，谷歌便针对Camera 提出了HAL3接口，其中主要包括了用于代表一系列操作主体的结构体以及具体操作函数，接下来我们分别进行详细介绍：

#### （1）、camera_module

```cpp
F:\Khadas_Edge_Android_Q\hardware\libhardware\include\hardware\camera_common.h
typedef struct camera_module {
    /**
     * Common methods of the camera module.  This *must* be the first member of
     * camera_module as users of this structure will cast a hw_module_t to
     * camera_module pointer in contexts where it's known the hw_module_t
     * references a camera_module.
     */
    hw_module_t common;

    /**
     * get_number_of_cameras:
     * Returns the number of camera devices accessible through the camera
     * module.  The camera devices are numbered 0 through N-1, where N is the
     * value returned by this call. The name of the camera device for open() is
     * simply the number converted to a string. That is, "0" for camera ID 0,
     * "1" for camera ID 1.
     */
    int (*get_number_of_cameras)(void);

    /**
     * get_camera_info:
     * Return the static camera information for a given camera device. This
     * information may not change for a camera device.
     */
    int (*get_camera_info)(int camera_id, struct camera_info *info);

    /**
     * set_callbacks:
     * Provide callback function pointers to the HAL module to inform framework
     * of asynchronous camera module events. The framework will call this
     * function once after initial camera HAL module load, after the
     * get_number_of_cameras() method is called for the first time, and before
     * any other calls to the module.
     */
    int (*set_callbacks)(const camera_module_callbacks_t *callbacks);

    /**
     * get_vendor_tag_ops:
     * Get methods to query for vendor extension metadata tag information. The
     * HAL should fill in all the vendor tag operation methods, or leave ops
     * unchanged if no vendor tags are defined.
     */
    void (*get_vendor_tag_ops)(vendor_tag_ops_t* ops);

    /**
     * open_legacy:
     * Open a specific legacy camera HAL device if multiple device HAL API
     * versions are supported by this camera HAL module. For example, if the
     * camera module supports both CAMERA_DEVICE_API_VERSION_1_0 and
     * CAMERA_DEVICE_API_VERSION_3_2 device API for the same camera id,
     * framework can call this function to open the camera device as
     * CAMERA_DEVICE_API_VERSION_1_0 device.
     */
    int (*open_legacy)(const struct hw_module_t* module, const char* id,
            uint32_t halVersion, struct hw_device_t** device);

    /**
     * set_torch_mode:
     * Turn on or off the torch mode of the flash unit associated with a given
     * camera ID. If the operation is successful, HAL must notify the framework
     * torch state by invoking
     * camera_module_callbacks.torch_mode_status_change() with the new state.
     */
    int (*set_torch_mode)(const char* camera_id, bool enabled);

    /**
     * init:
     * This method is called by the camera service before any other methods
     * are invoked, right after the camera HAL library has been successfully
     * loaded. It may be left as NULL by the HAL module, if no initialization
     * in needed.
     */
    int (*init)();

    /**
     * get_physical_camera_info:
     * Return the static metadata for a physical camera as a part of a logical
     * camera device. This function is only called for those physical camera
     * ID(s) that are not exposed independently. In other words, camera_id will
     * be greater or equal to the return value of get_number_of_cameras().
     */
    int (*get_physical_camera_info)(int physical_camera_id,
            camera_metadata_t **static_metadata);

    /**
     * is_stream_combination_supported:
     * Check for device support of specific camera stream combination.
     */
    int (*is_stream_combination_supported)(int camera_id,
            const camera_stream_combination_t *streams);

    /**
     * notify_device_state_change:
     *
     * Notify the camera module that the state of the overall device has
     * changed in some way that the HAL may want to know about.
     */
    void (*notify_device_state_change)(uint64_t deviceState);

    /* reserved for future use */
    void* reserved[2];
} camera_module_t;

```

由定义不难发现，camera_module_t包含了hw_module_t，主要用于表示Camera模块，其中定义了诸如get_number_of_cameras以及set_callbacks等扩展方法，而camera3_device_t包含了hw_device_t，主要用来表示Camera设备，其中定义了camera3_device_ops操作方法集合，用来实现正常获取图像数据以及控制Camera的功能。

#### （2）、camera3_stream_configuration

```cpp
F:\Khadas_Edge_Android_Q\hardware\libhardware\include\hardware\camera3.h
    typedef struct camera3_stream_configuration {
    /**
     * The total number of streams requested by the framework.  This includes
     * both input and output streams. The number of streams will be at least 1,
     * and there will be at least one output-capable stream.
     */
    uint32_t num_streams;

    /**
     * An array of camera stream pointers, defining the input/output
     * configuration for the camera HAL device.
     *
     * At most one input-capable stream may be defined (INPUT or BIDIRECTIONAL)
     * in a single configuration.
     *
     * At least one output-capable stream must be defined (OUTPUT or
     * BIDIRECTIONAL).
     */
    camera3_stream_t **streams;

    /**
     * >= CAMERA_DEVICE_API_VERSION_3_3:
     *
     * The operation mode of streams in this configuration, one of the value
     * defined in camera3_stream_configuration_mode_t.  The HAL can use this
     * mode as an indicator to set the stream property (e.g.,
     * camera3_stream->max_buffers) appropriately. For example, if the
     * configuration is
     * CAMERA3_STREAM_CONFIGURATION_CONSTRAINED_HIGH_SPEED_MODE, the HAL may
     * want to set aside more buffers for batch mode operation (see
     * android.control.availableHighSpeedVideoConfigurations for batch mode
     * definition).
     *
     */
    uint32_t operation_mode;

    /**
     * >= CAMERA_DEVICE_API_VERSION_3_5:
     *
     * The session metadata buffer contains the initial values of
     * ANDROID_REQUEST_AVAILABLE_SESSION_KEYS. This field is optional
     * and camera clients can choose to ignore it, in which case it will
     * be set to NULL. If parameters are present, then Hal should examine
     * the parameter values and configure its internal camera pipeline
     * accordingly.
     */
    const camera_metadata_t *session_parameters;
} camera3_stream_configuration_t;

```

该结构体主要用来代表配置的数据流列表，内部装有上层需要进行配置的数据流的指针，内部的定义简单介绍下：

- num_streams: 代表了来自上层的数据流的数量，其中包括了output以及input stream。
- streams: 是streams的指针数组，包括了至少一条output stream以及至多一条input stream。
- operation_mode: 当前数据流的操作模式，该模式在camera3_stream_configuration_mode_t中被定义，HAL通过这个参数可以针对streams做不同的设置。
- session_parameters: 该参数可以作为缺省参数，直接设置为NULL即可，CAMERA_DEVICE_API_VERSION_3_5以上的版本才支持。

#### （3）、camera3_stream_t

结构体camera3_stream_t的代码定义如下：

```cpp
F:\Khadas_Edge_Android_Q\hardware\libhardware\include\hardware\camera3.h
    typedef struct camera3_stream {

    /*****
     * Set by framework before configure_streams()
     */

    /**
     * The type of the stream, one of the camera3_stream_type_t values.
     */
    int stream_type;

    /**
     * The width in pixels of the buffers in this stream
     */
    uint32_t width;

    /**
     * The height in pixels of the buffers in this stream
     */
    uint32_t height;

    /**
     * The pixel format for the buffers in this stream. Format is a value from
     * the HAL_PIXEL_FORMAT_* list in system/core/include/system/graphics.h, or
     * from device-specific headers.
     */
    int format;

    /*****
     * Set by HAL during configure_streams().
     */

    /**
     * The gralloc usage flags for this stream, as needed by the HAL. The usage
     * flags are defined in gralloc.h (GRALLOC_USAGE_*), or in device-specific
     * headers.
     */
    uint32_t usage;

    /**
     * The maximum number of buffers the HAL device may need to have dequeued at
     * the same time. The HAL device may not have more buffers in-flight from
     * this stream than this value.
     */
    uint32_t max_buffers;

    /**
     * A handle to HAL-private information for the stream. Will not be inspected
     * by the framework code.
     */
    void *priv;

    /**
     * A field that describes the contents of the buffer. The format and buffer
     * dimensions define the memory layout and structure of the stream buffers,
     * while dataSpace defines the meaning of the data within the buffer.
     */
    android_dataspace_t data_space;

    int rotation;

    /**
     * The physical camera id this stream belongs to.
     */
    const char* physical_camera_id;

    /* reserved for future use */
    void *reserved[6];

} camera3_stream_t;
```

该结构体主要用来代表具体的数据流实体，在整个的配置过程中，需要在上层进行填充，当下发到HAL中后，HAL会针对其中的各项属性进行配置，这里便简单介绍下其内部的各个元素的意义：

- stream_type: 表示数据流的类型，类型在camera3_stream_type_t中被定义。
- width： 表示当前数据流中的buffer的宽度。
- height: 表示当前数据流中buffer的高度。
- format: 表示当前数据流中buffer的格式，该格式是在system/core/include/system/graphics.h中被定义。
- usage： 表示当前数据流的gralloc用法，其用法定义在gralloc.h中。
- max_buffers： 指定了当前数据流中可能支持的最大数据buffer数量。
- data_space: 指定了当前数据流buffer中存储的图像数据的颜色空间。
- rotation：指定了当前数据流的输出buffer的旋转角度，其角度的定义在camera3_stream_rotation_t中，该参数由Camera Service进行设置，必须在HAL中进行设置，该参数对于input stream并没有效果。
- physical_camera_id： 指定了当前数据流从属的物理camera Id。



#### （4）、camera3_stream_buffer_t

```cpp
typedef struct camera3_stream_buffer {
    /**
     * The handle of the stream this buffer is associated with
     */
    camera3_stream_t *stream;

    /**
     * The native handle to the buffer
     */
    buffer_handle_t *buffer;

    /**
     * Current state of the buffer, one of the camera3_buffer_status_t
     * values. The framework will not pass buffers to the HAL that are in an
     * error state. In case a buffer could not be filled by the HAL, it must
     * have its status set to CAMERA3_BUFFER_STATUS_ERROR when returned to the
     * framework with process_capture_result().
     */
    int status;

    /**
     * The acquire sync fence for this buffer. The HAL must wait on this fence
     * fd before attempting to read from or write to this buffer.
     */
     int acquire_fence;

    /**
     * The release sync fence for this buffer. The HAL must set this fence when
     * returning buffers to the framework, or write -1 to indicate that no
     * waiting is required for this buffer.
     */
    int release_fence;

} camera3_stream_buffer_t;
```

该结构体主要用来代表具体的buffer对象，其中重要元素如下：

- stream: 代表了从属的数据流
- buffer：buffer句柄

#### （5）、核心接口函数解析

HAL3的核心接口都是在camera3_device_ops中被定义，代码定义如下：

```cpp
F:\Khadas_Edge_Android_Q\hardware\libhardware\include\hardware\camera3.h
typedef struct camera3_device_ops {

    /**
     * initialize:
     * One-time initialization to pass framework callback function pointers to
     * the HAL. Will be called once after a successful open() call, before any
     * other functions are called on the camera3_device_ops structure.
     */
    int (*initialize)(const struct camera3_device *,
            const camera3_callback_ops_t *callback_ops);

    /**********************************************************************
     * Stream management
     */

    /**
     * configure_streams:
     *
     * CAMERA_DEVICE_API_VERSION_3_0 only:
     *
     * Reset the HAL camera device processing pipeline and set up new input and
     * output streams. This call replaces any existing stream configuration with
     * the streams defined in the stream_list. This method will be called at
     * least once after initialize() before a request is submitted with
     * process_capture_request().
     */
    int (*configure_streams)(const struct camera3_device *,
            camera3_stream_configuration_t *stream_list);

    /**
     * register_stream_buffers:
     */
    int (*register_stream_buffers)(const struct camera3_device *,
            const camera3_stream_buffer_set_t *buffer_set);

    /**********************************************************************
     * Request creation and submission
     */

    /**
     * construct_default_request_settings:
     *
     * Create capture settings for standard camera use cases.
     *
     * The device must return a settings buffer that is configured to meet the
     * requested use case, which must be one of the CAMERA3_TEMPLATE_*
     * enums. All request control fields must be included.
     */
    const camera_metadata_t* (*construct_default_request_settings)(
            const struct camera3_device *,
            int type);

    /**
     * process_capture_request:
     *
     * Send a new capture request to the HAL. The HAL should not return from
     * this call until it is ready to accept the next request to process. Only
     * one call to process_capture_request() will be made at a time by the
     * framework, and the calls will all be from the same thread. The next call
     * to process_capture_request() will be made as soon as a new request and
     * its associated buffers are available. In a normal preview scenario, this
     * means the function will be called again by the framework almost
     * instantly.
     */
    int (*process_capture_request)(const struct camera3_device *,
            camera3_capture_request_t *request);

    /**********************************************************************
     * Miscellaneous methods
     */

    /**
     * get_metadata_vendor_tag_ops:
     * Get methods to query for vendor extension metadata tag information. The
     * HAL should fill in all the vendor tag operation methods, or leave ops
     * unchanged if no vendor tags are defined.
     */
    void (*get_metadata_vendor_tag_ops)(const struct camera3_device*,
            vendor_tag_query_ops_t* ops);

    void (*dump)(const struct camera3_device *, int fd);

    int (*flush)(const struct camera3_device *);

    void (*signal_stream_flush)(const struct camera3_device*,
            uint32_t num_streams,
            const camera3_stream_t* const* streams);
 
    int (*is_reconfiguration_required)(const struct camera3_device*,
            const camera_metadata_t* old_session_params,
            const camera_metadata_t* new_session_params);

    /* reserved for future use */
    void *reserved[6];
} camera3_device_ops_t;
```

从代码中可以看见，该结构体定义了一系列的函数指针，用来指向平台厂商实际的实现方法，接下来就其中几个方法简单介绍下：

**a) initialize**
该方法必须在camera_module_t中的open方法之后，其它camera3_device_ops中方法之前被调用，主要用来将上层实现的回调方法注册到HAL中，并且根据需要在该方法中加入自定义的一些初始化操作，另外，谷歌针对该方法在性能方面也有严格的限制，该方法需要在5ms内返回，最长不能超过10ms。

**b) configure_streams**
该方法在完成initialize方法之后，在调用process_capture_request方法之前被调用，主要用于重设当前正在运行的Pipeline以及设置新的输入输出流，其中它会将stream_list中的新的数据流替换之前配置的数据流。在调用该方法之前必须确保没有新的request下发并且当前request的动作已经完成，否则会引起无法预测的错误。一旦HAL调用了该方法，则必须在内部配置好满足当前数据流配置的帧率，确保这个流程的运行的顺畅性。
其中包含了两个参数，分别是camera3_device以及stream_list(camera3_stream_configuration_t ),其中第二个参数是上层传入的数据流配置列表，该列表中必须包含至少一个output stream，同时至多包含一个input stream。
另外，谷歌针对该方法有着严格的性能要求，平台厂商在实现该方法的时候，需要在500ms内返回，最长不能超过1000ms。

**c) construct_default_request_settings**
该方法主要用于构建一系列默认的Camera Usecase的capture 设置项，通过camera_metadata_t来进行描述，其中返回值是一个camera_metadata_t指针，其指向的内存地址是由HAL来进行维护的，同样地，该方法需要在1ms内返回，最长不能超过5ms。

**d) process_capture_request**
该方法用于下发单次新的capture request到HAL中， 上层必须保证该方法的调用都是在一个线程中完成，而且该方法是异步的，同时其结果并不是通过返回值给到上层，而是通过HAL调用另一个接口process_capture_result()来将结果返回给上层的，在使用的过程中，通过in-flight机制，保证短时间内下发足够多的request，从而满足帧率要求。

该方法的性能依然受到谷歌的严格要求，规定其需要在一帧图像处理完的时长内返回，最长不超过4帧图像处理完成的时长，比如当前预览帧率是30帧，则该方法的操作耗时最长不能超过120ms，否则便会引起明显的帧抖动，从而影响用户体验。

**e) dump**
该方法用于打印当前Camera设备的状态，一般是由上层通过dumpsys工具输出debug dump信息或者主动抓取bugreport的时候被调用，该方法必须是非阻塞实现，同时需要保证在1ms内返回，最长不能超过10ms。

**f) flush**
当上层需要执行新的configure_streams的时候，需要调用该方法去尽可能快地清除掉当前已经在处理中的或者即将处理的任务，为配置数据流提供一个相对稳定的环境，其具体工作如下：

- 所有的还在流转的request会尽可能快的返回
- 并未开始进行流转的request会直接返回，并携带错误信息
- 任何可以打断的硬件操作会立即被停止
- 任何无法进行打断的硬件操作会在当前状态下进行休眠

flush会在所有的buffer都得以释放，所有request都成功返回后才真正返回，该方法需要在100ms内返回，最长不能超过1000ms。

上面的一系列方法是上层直接对下控制Camera Hal，而一旦Camera Hal产生了数据或者事件的时候，可以通过camera3_callback_ops中定义的回调方法将数据或者事件返回至上层，该结构体定义如下：

```cpp
F:\Khadas_Edge_Android_Q\hardware\libhardware\include\hardware\camera3.h
typedef struct camera3_callback_ops {

    /**
     * process_capture_result:
     *
     * Send results from a completed capture to the framework.
     * process_capture_result() may be invoked multiple times by the HAL in
     * response to a single capture request. This allows, for example, the
     * metadata and low-resolution buffers to be returned in one call, and
     * post-processed JPEG buffers in a later call, once it is available. Each
     * call must include the frame number of the request it is returning
     * metadata or buffers for.
     */
    void (*process_capture_result)(const struct camera3_callback_ops *,
            const camera3_capture_result_t *result);

    /**
     * notify:
     *
     * Asynchronous notification callback from the HAL, fired for various
     * reasons. Only for information independent of frame capture, or that
     * require specific timing. The ownership of the message structure remains
     * with the HAL, and the msg only needs to be valid for the duration of this
     * call.
     */
    void (*notify)(const struct camera3_callback_ops *,
            const camera3_notify_msg_t *msg);

    camera3_buffer_request_status_t (*request_stream_buffers)(
            const struct camera3_callback_ops *,
            uint32_t num_buffer_reqs,
            const camera3_buffer_request_t *buffer_reqs,
            /*out*/uint32_t *num_returned_buf_reqs,
            /*out*/camera3_stream_buffer_ret_t *returned_buf_reqs);

    void (*return_stream_buffers)(
            const struct camera3_callback_ops *,
            uint32_t num_buffers,
            const camera3_stream_buffer_t* const* buffers);

} camera3_callback_ops_t;

```

其中常用的回调方法主要有两个：用于返回数据的process_capture_result以及用于返回事件的notify，接下来分别介绍下：

**1、process_capture_result**
该方法用于返回HAL部分产生的metadata和image buffers，它与request是多对一的关系，同一个request，可能会对应到多个result，比如可以通过调用一次该方法用于返回metadata以及低分辨率的图像数据，再调用一次该方法用于返回jpeg格式的拍照数据，而这两次调用时对应于同一个process_capture_request动作。

同一个Request的Metadata以及Image Buffers的先后顺序无关紧要，但是同一个数据流的不同Request之间的Result必须严格按照Request的下发先后顺序进行依次返回的，如若不然，会导致图像数据显示出现顺序错乱的情况。

该方法是非阻塞的，而且并且必须要在5ms内返回。

**2、notify**
该方法用于异步返回HAL事件到上层，必须非阻塞实现，而且要在5ms内返回。

谷歌为了将系统框架和平台厂商的自定义部分相分离，在Android上推出了Treble项目，该项目直接将平台厂商的实现部分放入vendor分区中进行管理，进而与system分区保持隔离，这样便可以在相互独立的空间中进行各自的迭代升级，而互不干扰，而在相机框架体系中，便将Camera HAL Module从Camera Service中解耦出来，放入独立进程Camera Provider中进行管理，而为了更好的进行跨进程访问，谷歌针对Provider提出了HIDL机制用于Camera Servic对于Camera Provier的访问，而HIDL接口的实现是在Camera Provider中实现，针对Camera HAL Module的控制又是通过谷歌制定的Camera HAL3接口来完成，所以由此看来，Provider的职责也比较简单，通过HIDL机制保持与Camera Service的通信，通过HAL3接口控制着Camera HAL Module。



## （四）、Camera Service启动流程概览

系统启动时，就会启动 CameraProvider 服务。它将 Camera HAL 从 cameraserver 进程中分离出来，作为一个独立进程 android.hardware.camera.provider@2.4-service 来控制 HAL。
这两个进程之间通过 HIDL 机制进行通信。它的主要功能（如下图所示）是将 service 与 HAL 隔离，以方便 HAL 部分进行独立升级。这其实和 APP 与 Framework 之间的 Binder 机制类似，通过引入一个进程间通信机制而针对不同层级进行解耦（从 Local call 变成了 Remote call）。

![](https://raw.githubusercontent.com/zhoujinjianOS/PicGo/master/Android.10.Camera.06/Cameraserver_treble.png)

总体逻辑顺序：

- provider 进程启动，注册；

- cameraserver 进程启动，注册，初始化；

- cameraserver 获取远端 provider（此时实例化 CameraProvider 并初始化）。
  上图中，实线箭头是调用关系。左边是 cameraserver 进程中的动作，右边则是 provider 进程中的动作，它们之间通过 ICameraProvider 联系在了一起，而这个东西与 HIDL 相关，我们可以不用关心它的实现方式。

  ![](https://raw.githubusercontent.com/zhoujinjianOS/PicGo/master/Android.10.Camera.06/cameraserver_hidl_cameraprovier.png)

由图可见：

- cameraserver 一侧，Cameraservice 类依旧是主体。它通过 CameraProviderManager 来管理对 CameraProvider 的操作。此处初始化的最终目的是连接上 CameraProvider。
- provider 一侧，最终主体是 CameraProvider。初始化最终目的是得到一个 mModule，通过它可以直接与 HAL 接口定义层进行交互。



#### （1）、CameraService 的启动与初始化

一般来说应该是 Provider 服务先启动，然后 Cameraserver 再启动，并 ”连接“ 到 Provider。前面已经分析了 Provider 的启动，现在就来看看 Cameraserver 的启动流程。

```cpp
F:\Khadas_Edge_Android_Q\frameworks\av\camera\cameraserver\cameraserver.rc
    service cameraserver /system/bin/cameraserver
    class main
    user cameraserver
    group audio camera input drmrpc readproc
    ioprio rt 4
    writepid /dev/cpuset/camera-daemon/tasks /dev/stune/top-app/tasks
    rlimit rtprio 10 10

F:\Khadas_Edge_Android_Q\frameworks\av\camera\cameraserver\main_cameraserver.cpp
    #define LOG_TAG "cameraserver"
#define LOG_NDEBUG 0

#include "CameraService.h"
#include <hidl/HidlTransportSupport.h>

using namespace android;

int main(int argc __unused, char** argv __unused)
{
    signal(SIGPIPE, SIG_IGN);

    // Set 5 threads for HIDL calls. Now cameraserver will serve HIDL calls in
    // addition to consuming them from the Camera HAL as well.
    hardware::configureRpcThreadpool(5, /*willjoin*/ false);

    sp<ProcessState> proc(ProcessState::self());
    //defaultServiceManager的实现容易混淆，系统中有两个实现，在调用时需要注意方法所在的域，
    /* 获得 ServiceManager 的代理类 BpServiceManager, 注意不是 HwServiceManager 的代理类
     * ServiceManager 这个服务管理器是用来跟上层framework交互的，原理跟 HwServiceManager一样
     */   
    sp<IServiceManager> sm = defaultServiceManager();

    ALOGI("ServiceManager: %p", sm.get());
    CameraService::instantiate();
    ALOGI("ServiceManager: %p done instantiate", sm.get());
    ProcessState::self()->startThreadPool();
    IPCThreadState::self()->joinThreadPool();
}
//01-18 08:50:15.771   326   326 I cameraserver: ServiceManager: 0xe90231a0
//01-18 08:50:15.771   326   326 I CameraService: CameraService started (pid=326)
//01-18 08:50:15.772   326   326 I CameraService: CameraService process starting
```

实例化只有简单的一行代码，但实例化的过程并不那么简单。这个 instantiate() 接口并不是定义在 CameraService 类中的，而是定义在 BinderService 类里（而 CameraService 继承了它）。在此处，它的作用是创建一个 CameraService（通过 new 的方式），并将其加入到 ServiceManager 中（注意，在这一过程中，CameraService 被强指针引用了）。

==**CameraService::instantiate()的实现，将CameraService注册到ServiceManager**==

```cpp
F:\Khadas_Edge_Android_Q\frameworks\av\services\camera\libcameraservice\CameraService.h
    //注册的SERVICE："media.camera"
    // Implementation of BinderService<T>
    static char const* getServiceName() { return "media.camera"; }

F:\Khadas_Edge_Android_Q\frameworks\native\libs\binder\include\binder\BinderService.h
    /*BinderService 是 CameraService 的父类，模板类 SERVICE = CameraService*/
    template<typename SERVICE>
class BinderService
{
public:
    
    static status_t publish(bool allowIsolated = false,
                            int dumpFlags = IServiceManager::DUMP_FLAG_PRIORITY_DEFAULT) {
        /* 得到ServiceManager的代理类BpServiceManager*/
        sp<IServiceManager> sm(defaultServiceManager());
        /* SERVICE = CameraService， 注册CameraService 实例化对象*/
        /*new CameraService() 得到实例化对象，智能指针第一次引用回调用 onFirstRef() 函数, 后面分析*/
        return sm->addService(String16(SERVICE::getServiceName()), new SERVICE(), allowIsolated,
                              dumpFlags);//注册的SERVICE："media.camera"
    }

    ......
    /* 静态类属性的， 所以可以直接调用 */
    static void instantiate() { publish(); }

    static status_t shutdown() { return NO_ERROR; }

private:
    ......
};
```



由于首次被强指针引用时，就会调用 `onFirstRef()` 函数执行初始化之类的业务逻辑，所以现在就看看 CameraService 在此处实现了什么逻辑。

```cpp
F:\Khadas_Edge_Android_Q\frameworks\av\services\camera\libcameraservice\CameraService.cpp
    void CameraService::onFirstRef()
{
    ALOGI("CameraService process starting");

    BnCameraService::onFirstRef();

    // Update battery life tracking if service is restarting
    BatteryNotifier& notifier(BatteryNotifier::getInstance());
    notifier.noteResetCamera();
    notifier.noteResetFlashlight();

    status_t res = INVALID_OPERATION;

    res = enumerateProviders();
    if (res == OK) {
        mInitialized = true;
    }

    mUidPolicy = new UidPolicy(this);
    mUidPolicy->registerSelf();
    mSensorPrivacyPolicy = new SensorPrivacyPolicy(this);
    mSensorPrivacyPolicy->registerSelf();
    sp<HidlCameraService> hcs = HidlCameraService::getInstance(this);
    if (hcs->registerAsService() != android::OK) {
        ALOGE("%s: Failed to register default android.frameworks.cameraservice.service@1.0",
              __FUNCTION__);
    }

    // This needs to be last call in this function, so that it's as close to
    // ServiceManager::addService() as possible.
    CameraService::pingCameraServiceProxy();
    ALOGI("CameraService pinged cameraservice proxy");
}

status_t CameraService::enumerateProviders() {
    status_t res;

    std::vector<std::string> deviceIds;
    {
        Mutex::Autolock l(mServiceLock);

        if (nullptr == mCameraProviderManager.get()) {
            mCameraProviderManager = new CameraProviderManager();
            res = mCameraProviderManager->initialize(this);
            if (res != OK) {
                ALOGE("%s: Unable to initialize camera provider manager: %s (%d)",
                        __FUNCTION__, strerror(-res), res);
                return res;
            }
        }


        // Setup vendor tags before we call get_camera_info the first time
        // because HAL might need to setup static vendor keys in get_camera_info
        // TODO: maybe put this into CameraProviderManager::initialize()?
        mCameraProviderManager->setUpVendorTags();

        if (nullptr == mFlashlight.get()) {
            mFlashlight = new CameraFlashlight(mCameraProviderManager, this);
        }

        res = mFlashlight->findFlashUnits();
        if (res != OK) {
            ALOGE("Failed to enumerate flash units: %s (%d)", strerror(-res), res);
        }
        //遍历mProviders
        deviceIds = mCameraProviderManager->getCameraDeviceIds();
    }


    for (auto& cameraId : deviceIds) {
        String8 id8 = String8(cameraId.c_str());
        if (getCameraState(id8) == nullptr) {
            onDeviceStatusChanged(id8, CameraDeviceStatus::PRESENT);
        }
    }

    return OK;
}
```

- 首先将 `CameraProviderManager` 实例化，然后调用 `initialize()` 接口将其初始化，传入的参数是 `this` 指针，指向当前 CameraService 实例的地址。
- mServiceProxy的赋值，上面是CameraProviderManager的初始化过程，CameraProviderManager就是管理camera Service与camera provider之间通信的工程管理类，两个参数，其中第二个参数就是远程代理类。这个参数已经是默认赋值了。

```cpp
F:\Khadas_Edge_Android_Q\frameworks\av\services\camera\libcameraservice\common\CameraProviderManager.h
    struct HardwareServiceInteractionProxy : public ServiceInteractionProxy {
        ......
        virtual sp<hardware::camera::provider::V2_4::ICameraProvider> getService(
                const std::string &serviceName) override {
            return hardware::camera::provider::V2_4::ICameraProvider::getService(serviceName);
        }

        virtual hardware::hidl_vec<hardware::hidl_string> listServices() override;
    };
    
//可以看到initialize函数proxy为默认值：sHardwareServiceInteractionProxy
status_t initialize(wp<StatusListener> listener,
        ServiceInteractionProxy *proxy = &sHardwareServiceInteractionProxy)

F:\Khadas_Edge_Android_Q\frameworks\av\services\camera\libcameraservice\common\CameraProviderManager.cpp
    status_t CameraProviderManager::initialize(wp<CameraProviderManager::StatusListener> listener,
        ServiceInteractionProxy* proxy) {
    std::lock_guard<std::mutex> lock(mInterfaceMutex);
    ......
    mListener = listener;
    mServiceProxy = proxy;
    mDeviceState = static_cast<hardware::hidl_bitfield<provider::V2_5::DeviceState>>(
        provider::V2_5::DeviceState::NORMAL);

    // Registering will trigger notifications for all already-known providers
    bool success = mServiceProxy->registerForNotifications(
        /* instance name, empty means no filter */ "",
        this);
    ......

    for (const auto& instance : mServiceProxy->listServices()) {
        this->addProviderLocked(instance);
    }

    IPCThreadState::self()->flushCommands();

    return OK;
}
status_t CameraProviderManager::addProviderLocked(const std::string& newProvider) {
    for (const auto& providerInfo : mProviders) {
        //检查已知的 Provider 中是否已有名为 `legacy/0` 的。
        if (providerInfo->mProviderName == newProvider) {
            ALOGW("%s: Camera provider HAL with name '%s' already registered", __FUNCTION__,
                    newProvider.c_str());
            return ALREADY_EXISTS;
        }
    }

    sp<provider::V2_4::ICameraProvider> interface;
    //根据 legacy/0 从服务代理处获取 CameraProvider 接口
    interface = mServiceProxy->getService(newProvider);
    ......
    //通过 ProviderInfo 来保存当前 Provider 相关信息。
    sp<ProviderInfo> providerInfo = new ProviderInfo(newProvider, this);
    status_t res = providerInfo->initialize(interface, mDeviceState);
    ......
    //记录当前 Provider。
    mProviders.push_back(providerInfo);

    return OK;
}


```



#### （2）、CameraProviderManager::ProviderInfo::initialize

```cpp
status_t CameraProviderManager::ProviderInfo::initialize(
        sp<provider::V2_4::ICameraProvider>& interface,
        hardware::hidl_bitfield<provider::V2_5::DeviceState> currentDeviceState) {
    status_t res = parseProviderName(mProviderName, &mType, &mId);
    ......
    ALOGI("Connecting to new camera provider: %s, isRemote? %d",
            mProviderName.c_str(), interface->isRemote());

    // Determine minor version
    auto castResult = provider::V2_5::ICameraProvider::castFrom(interface);
    if (castResult.isOk()) {
        mMinorVersion = 5;
    } else {
        mMinorVersion = 4;
    }

    // cameraDeviceStatusChange callbacks may be called (and causing new devices added)
    // before setCallback returns
    hardware::Return<Status> status = interface->setCallback(this);

    hardware::Return<bool> linked = interface->linkToDeath(this, /*cookie*/ mId);
......

    ALOGV("%s: Setting device state for %s: 0x%" PRIx64,
            __FUNCTION__, mProviderName.c_str(), mDeviceState);
    notifyDeviceStateChange(currentDeviceState);

    res = setUpVendorTags();
    ......
    // Get initial list of camera devices, if any
    std::vector<std::string> devices;
    hardware::Return<void> ret = interface->getCameraIdList([&status, this, &devices](
            Status idStatus,
            const hardware::hidl_vec<hardware::hidl_string>& cameraDeviceNames) {
        status = idStatus;
        if (status == Status::OK) {
            for (auto& name : cameraDeviceNames) {
                uint16_t major, minor;
                std::string type, id;
                status_t res = parseDeviceName(name, &major, &minor, &type, &id);
                if (res != OK) {
                    ALOGE("%s: Error parsing deviceName: %s: %d", __FUNCTION__, name.c_str(), res);
                    status = Status::INTERNAL_ERROR;
                } else {
                    devices.push_back(name);
                    mProviderPublicCameraIds.push_back(id);
                }
            }
        } });
    ......

    mIsRemote = interface->isRemote();

    sp<StatusListener> listener = mManager->getStatusListener();
    for (auto& device : devices) {
        std::string id;
        status_t res = addDevice(device, common::V1_0::CameraDeviceStatus::PRESENT, &id);
        ......
    }
    //Log:Camera provider legacy/0 ready with 1 camera devices
    ALOGI("Camera provider %s ready with %zu camera devices",
            mProviderName.c_str(), mDevices.size());

    mInitialized = true;
    return OK;
}
```

#### （3）、CameraProviderManager::ProviderInfo::addDevice

```cpp
status_t CameraProviderManager::ProviderInfo::addDevice(const std::string& name,
        CameraDeviceStatus initialStatus, /*out*/ std::string* parsedId) {
    //CameraProviderManager: Enumerating new camera device: device@3.3/legacy/0
    ALOGI("Enumerating new camera device: %s", name.c_str());

    uint16_t major, minor;
    std::string type, id;

    status_t res = parseDeviceName(name, &major, &minor, &type, &id);
    ......

    std::unique_ptr<DeviceInfo> deviceInfo;
    switch (major) {
        case 1:
            deviceInfo = initializeDeviceInfo<DeviceInfo1>(name, mProviderTagid,
                    id, minor);
            break;
        case 3:
            deviceInfo = initializeDeviceInfo<DeviceInfo3>(name, mProviderTagid,
                    id, minor);
            break;
        default:
            ALOGE("%s: Device %s: Unknown HIDL device HAL major version %d:", __FUNCTION__,
                    name.c_str(), major);
            return BAD_VALUE;
    }
    if (deviceInfo == nullptr) return BAD_VALUE;
    deviceInfo->mStatus = initialStatus;
    bool isAPI1Compatible = deviceInfo->isAPI1Compatible();

    mDevices.push_back(std::move(deviceInfo));

    mUniqueCameraIds.insert(id);
    ......
    if (parsedId != nullptr) {
        *parsedId = id;
    }
    return OK;
}
```

#### （4）、CameraProviderManager::ProviderInfo::initializeDeviceInfo

```cpp

F:\Khadas_Edge_Android_Q\frameworks\av\services\camera\libcameraservice\common\CameraProviderManager.cpp
template<class DeviceInfoT>
std::unique_ptr<CameraProviderManager::ProviderInfo::DeviceInfo>
    CameraProviderManager::ProviderInfo::initializeDeviceInfo(
        const std::string &name, const metadata_vendor_id_t tagId,
        const std::string &id, uint16_t minorVersion) {
    Status status;

    auto cameraInterface =
            startDeviceInterface<typename DeviceInfoT::InterfaceT>(name);
    ......

    CameraResourceCost resourceCost;
    cameraInterface->getResourceCost([&status, &resourceCost](
        Status s, CameraResourceCost cost) {
                status = s;
                resourceCost = cost;
            });
    ......
    for (auto& conflictName : resourceCost.conflictingDevices) {
        uint16_t major, minor;
        std::string type, id;
        status_t res = parseDeviceName(conflictName, &major, &minor, &type, &id);
        ......
        conflictName = id;
    }

    return std::unique_ptr<DeviceInfo>(
        new DeviceInfoT(name, tagId, id, minorVersion, resourceCost, this,
                mProviderPublicCameraIds, cameraInterface));
}

F:\Khadas_Edge_Android_Q\frameworks\av\services\camera\libcameraservice\common\CameraProviderManager.cpp
    
    template<>
sp<device::V3_2::ICameraDevice>
CameraProviderManager::ProviderInfo::startDeviceInterface
        <device::V3_2::ICameraDevice>(const std::string &name) {
    Status status;
    sp<device::V3_2::ICameraDevice> cameraInterface;
    hardware::Return<void> ret;
    const sp<provider::V2_4::ICameraProvider> interface = startProviderInterface();
    ......
    ret = interface->getCameraDeviceInterface_V3_x(name, [&status, &cameraInterface](
        Status s, sp<device::V3_2::ICameraDevice> interface) {
                status = s;
                cameraInterface = interface;
            });
    ......
    return cameraInterface;
}
//getCameraDeviceInterface_V3_x
F:\Khadas_Edge_Android_Q\hardware\interfaces\camera\provider\2.4\default\LegacyCameraProviderImpl_2_4.cpp
    Return<void> LegacyCameraProviderImpl_2_4::getCameraDeviceInterface_V3_x(
        const hidl_string& cameraDeviceName,
        ICameraProvider::getCameraDeviceInterface_V3_x_cb _hidl_cb)  {
	ALOGI("LegacyCameraProviderImpl_2_4 getCameraDeviceInterface_V3_x");

    std::string cameraId, deviceVersion;
    bool match = matchDeviceName(cameraDeviceName, &deviceVersion, &cameraId);
    ......
    std::string deviceName(cameraDeviceName.c_str());
    ssize_t index = mCameraDeviceNames.indexOf(std::make_pair(cameraId, deviceName));
    ......

    if (mCameraStatusMap.count(cameraId) == 0 ||
            mCameraStatusMap[cameraId] != CAMERA_DEVICE_STATUS_PRESENT) {
        _hidl_cb(Status::ILLEGAL_ARGUMENT, nullptr);
        return Void();
    }

    sp<android::hardware::camera::device::V3_2::implementation::CameraDevice> deviceImpl;

    ......
    // ICameraDevice 3.2 and 3.3
    // Since some Treble HAL revisions can map to the same legacy HAL version(s), we default
    // to the newest possible Treble HAL revision, but allow for override if needed via
    // system property.
    switch (mPreferredHal3MinorVersion) {
        ......
        case 3: { // Map legacy camera device v3 HAL to Treble camera device HAL v3.3
            ALOGV("Constructing v3.3 camera device");
            deviceImpl = new android::hardware::camera::device::V3_3::implementation::CameraDevice(
                    mModule, cameraId, mCameraDeviceNames);
           ..
            break;
        }
        ......
    }

    _hidl_cb (Status::OK, deviceImpl->getInterface());
    return Void();
}

//01-18 08:50:15.089   258   258 I CamPrvdr@2.4-legacy: Loaded "Rockchip Camera3HAL Module" camera module
//01-18 08:50:15.091   258   258 V CamPrvdr@2.4-legacy: Preferred HAL 3 minor version is 3
//01-18 08:50:15.113   258   435 I CamPrvdr@2.4-legacy: LegacyCameraProviderImpl_2_4 getCameraDeviceInterface_V3_x
//01-18 08:50:15.113   258   435 V CamPrvdr@2.4-legacy: Constructing v3.3 camera device


CameraProviderManager::ProviderInfo::DeviceInfo3::DeviceInfo3(const std::string& name,
        const metadata_vendor_id_t tagId, const std::string &id,
        uint16_t minorVersion,
        const CameraResourceCost& resourceCost,
        sp<ProviderInfo> parentProvider,
        const std::vector<std::string>& publicCameraIds,
        sp<InterfaceT> interface) :
        DeviceInfo(name, tagId, id, hardware::hidl_version{3, minorVersion},
                   publicCameraIds, resourceCost, parentProvider) {
    // Get camera characteristics and initialize flash unit availability
    Status status;
    hardware::Return<void> ret;
    ret = interface->getCameraCharacteristics([&status, this](Status s,
                    device::V3_2::CameraMetadata metadata) {
                status = s;
                if (s == Status::OK) {
                    camera_metadata_t *buffer =
                            reinterpret_cast<camera_metadata_t*>(metadata.data());
                    size_t expectedSize = metadata.size();
                    int res = validate_camera_metadata_structure(buffer, &expectedSize);
                    if (res == OK || res == CAMERA_METADATA_VALIDATION_SHIFTED) {
                        set_camera_metadata_vendor_id(buffer, mProviderTagid);
                        mCameraCharacteristics = buffer;
                    } else {
                        ALOGE("%s: Malformed camera metadata received from HAL", __FUNCTION__);
                        status = Status::INTERNAL_ERROR;
                    }
                }
            });
    ......
    mIsPublicallyHiddenSecureCamera = isPublicallyHiddenSecureCamera();
    status_t res = fixupMonochromeTags();
    ......
    auto stat = addDynamicDepthTags();
    ......
    res = deriveHeicTags();
    ......
    camera_metadata_entry flashAvailable =
            mCameraCharacteristics.find(ANDROID_FLASH_INFO_AVAILABLE);
    ......
    queryPhysicalCameraIds();
    ......
    if (mIsLogicalCamera) {
        for (auto& id : mPhysicalIds) {
            if (std::find(mPublicCameraIds.begin(), mPublicCameraIds.end(), id) !=
                    mPublicCameraIds.end()) {
                continue;
            }

            hardware::hidl_string hidlId(id);
            ret = interface_3_5->getPhysicalCameraCharacteristics(hidlId,
                    [&status, &id, this](Status s, device::V3_2::CameraMetadata metadata) {
                status = s;
                if (s == Status::OK) {
                    camera_metadata_t *buffer =
                            reinterpret_cast<camera_metadata_t*>(metadata.data());
                    size_t expectedSize = metadata.size();
                    int res = validate_camera_metadata_structure(buffer, &expectedSize);
                    if (res == OK || res == CAMERA_METADATA_VALIDATION_SHIFTED) {
                        set_camera_metadata_vendor_id(buffer, mProviderTagid);
                        mPhysicalCameraCharacteristics[id] = buffer;
                    } .......
                }
            });
            .......
        }
    }

    if (!kEnableLazyHal) {
        // Save HAL reference indefinitely
        mSavedInterface = interface;
    }
} 
//sp<InterfaceT> interface) :
//        DeviceInfo(name, tagId, id, hardware::hidl_version{3, minorVersion},
//                   publicCameraIds, resourceCost, parentProvider)
```

#### （5）、CameraService 与 CameraProvider衔接

继续分析CameraProviderManager::addProviderLocked中的mServiceProxy->getService(newProvider)即为HardwareServiceInteractionProxy->getService(newProvider)

```cpp

F:\Khadas_Edge_Android_Q\frameworks\av\services\camera\libcameraservice\common\CameraProviderManager.h
// Standard use case - call into the normal generated static methods which invoke
// the real hardware service manager
struct HardwareServiceInteractionProxy : public ServiceInteractionProxy {
    virtual bool registerForNotifications(
            const std::string &serviceName,
            const sp<hidl::manager::V1_0::IServiceNotification>
            &notification) override {
        return hardware::camera::provider::V2_4::ICameraProvider::registerForNotifications(
                serviceName, notification);
    }
    virtual sp<hardware::camera::provider::V2_4::ICameraProvider> getService(
            const std::string &serviceName) override {
        return hardware::camera::provider::V2_4::ICameraProvider::getService(serviceName);
    }

    virtual hardware::hidl_vec<hardware::hidl_string> listServices() override;
};
```



```cpp
/Khadas_Edge_Android_Q/out/soong/.intermediates/hardware/interfaces/camera/provider/2.4/android.hardware.camera.provider@2.4_genc++/gen/android/hardware/camera/provider/CameraProviderAll.cpp
    
::android::sp<ICameraProvider> ICameraProvider::getService(const std::string &serviceName, const bool getStub) {
    return ::android::hardware::details::getServiceInternal<BpHwCameraProvider>(serviceName, true, getStub);
}


F:\Khadas_Edge_Android_Q\system\libhidl\transport\ServiceManagement.cpp
sp<::android::hidl::base::V1_0::IBase> getRawServiceInternal(const std::string& descriptor,
                                                             const std::string& instance,
                                                             bool retry, bool getStub) {
    using Transport = ::android::hidl::manager::V1_0::IServiceManager::Transport;
    using ::android::hidl::manager::V1_0::IServiceManager;
    sp<Waiter> waiter;

    sp<IServiceManager1_1> sm;
    Transport transport = Transport::EMPTY;
    if (kIsRecovery) {
        transport = Transport::PASSTHROUGH;
    } else {
        sm = defaultServiceManager1_1();
        ......
        Return<Transport> transportRet = sm->getTransport(descriptor, instance);
        ......
        transport = transportRet;
    }

    const bool vintfHwbinder = (transport == Transport::HWBINDER);
    const bool vintfPassthru = (transport == Transport::PASSTHROUGH);
    ......

    for (int tries = 0; !getStub && (vintfHwbinder || vintfLegacy); tries++) {
        ......
        //查询ICameraProvider这个hidl服务，得到IBase对象
        Return<sp<IBase>> ret = sm->get(descriptor, instance);
       ......
        sp<IBase> base = ret;
        if (base != nullptr) {
            Return<bool> canCastRet =
                details::canCastInterface(base.get(), descriptor.c_str(), true /* emitError */);

            if (canCastRet.isOk() && canCastRet) {
                if (waiter != nullptr) {
                    waiter->done();
                }
                return base; // still needs to be wrapped by Bp class.
            }

            if (!handleCastError(canCastRet, descriptor, instance)) break;
        }

        ......
    }
......
    if (getStub || vintfPassthru || vintfLegacy) {
        const sp<IServiceManager> pm = getPassthroughServiceManager();
        if (pm != nullptr) {
            sp<IBase> base = pm->get(descriptor, instance).withDefault(nullptr);
            if (!getStub || trebleTestingOverride) {
                base = wrapPassthrough(base);
            }
            return base;
        }
    }

    return nullptr;
}

F:\Khadas_Edge_Android_Q\system\libhidl\transport\include\hidl\HidlTransportSupport.h
template <typename BpType, typename IType = typename BpType::Pure,
          typename = std::enable_if_t<std::is_same<i_tag, typename IType::_hidl_tag>::value>,
          typename = std::enable_if_t<std::is_same<bphw_tag, typename BpType::_hidl_tag>::value>>
sp<IType> getServiceInternal(const std::string& instance, bool retry, bool getStub) {
    using ::android::hidl::base::V1_0::IBase;
    //这里通过sm->get(ICameraProvider::descriptor, serviceName)查询ICameraProvider这个hidl服务，得到IBase对象后，再通过ICameraProvider::castFrom转换为ICameraProvider对象。
    sp<IBase> base = getRawServiceInternal(IType::descriptor, instance, retry, getStub);
    ......
    if (base->isRemote()) {
        // getRawServiceInternal guarantees we get the proper class
        /* 获取 CameraProvider 代理类对象(binder) */
        return sp<IType>(new BpType(getOrCreateCachedBinder(base.get())));
    }
    /* 转换为 BpHwCameraProvider 对象 */
    return IType::castFrom(base);
}

```

```cpp
01-18 08:50:15.779   326   326 I CameraProviderManager: Connecting to new camera provider: external/0, isRemote? 1
01-18 08:50:15.779   326   326 V CameraProviderManager: initialize: Setting device state for external/0: 0x0
01-18 08:50:15.780   326   326 V CameraProviderManager: Request to start camera provider: external/0
01-18 08:50:15.780   326   326 I CameraProviderManager: Camera provider external/0 ready with 0 camera devices
01-18 08:50:15.893   326   326 I CameraProviderManager: Connecting to new camera provider: legacy/0, isRemote? 1
01-18 08:50:15.894   326   326 V CameraProviderManager: initialize: Setting device state for legacy/0: 0x0
01-18 08:50:15.894   326   326 V CameraProviderManager: Request to start camera provider: legacy/0
01-18 08:50:15.896   326   326 I CameraProviderManager: Enumerating new camera device: device@3.3/legacy/0
01-18 08:50:15.896   326   326 V CameraProviderManager: Request to start camera provider: legacy/0
01-18 08:50:15.905   326   326 E CameraProviderManager: DeviceInfo3: Converted ICameraDevice instance to nullptr
01-18 08:50:15.905   326   326 I CameraProviderManager: Camera provider legacy/0 ready with 1 camera devices
01-18 08:50:15.905   326   422 W CameraProviderManager: addProviderLocked: Camera provider HAL with name 'external/0' already registered
01-18 08:50:15.906   326   326 I CameraService: onDeviceStatusChanged: Status changed for cameraId=0, newStatus=1
01-18 08:50:15.906   326   326 I CameraService: onDeviceStatusChanged: Unknown camera ID 0, a new camera is added
01-18 08:50:15.906   326   422 I CameraService: onDeviceStatusChanged: Status changed for cameraId=0, newStatus=1
01-18 08:50:15.907   326   326 V CameraService: updateStatus: Status has changed for camera ID 0 from 0 to 0x1
01-18 08:50:15.907   326   422 W CameraProviderManager: addProviderLocked: Camera provider HAL with name 'legacy/0' already registered
```



Binder 分为 Bp 以及 Bn，Bp 对应 client ， Bn 对应 server。

CameraProvider 服务，通过 ICameraProvider::getService 实例化 CameraProvider，然后通过toBinder函数转换为BnHwCameraProvider，注册到HwServiceManager中。

CameraService 服务，通过 ICameraProvider::getService 拿到 BpHwCameraProvider，然后通过binder通信调用到CameraProvider，进而调用Camera HAL。



上层 framework 通过 ServiceManager( /dev/binder ) 得到 CameraService 服务，而
CameraService 通过 HwServiceManager( /dev/hwbinder ) 得到 CameraProvider 服务，而 CameraProvider 与 Camera HAL 绑定。这样上层 framework 就能够访问 Camera HAL 层了。
