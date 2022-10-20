---
title: Android 10 Camera源码分析9：Camera startPreview流程
cover: https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/post.cover.pictures/bing-wallpaper-2018.04.39.jpg
categories: 
 - Camera
tags:
 - Camera
toc: true
abbrlink: 20220501
date: 2022-05-01 05:01:00
---
注：文章都是通过阅读各位前辈总结的资料 Android 10.0 && Linux（Kernel 4.19）Rockchip平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除（◕‿◕），转载请注明出处（©Rockchip ©Android @Linux 版权所有），谢谢。

（==**文章基于 Kernel-4.19**==）&&（==**文章基于 Android 10.0**==）

[【blog.zhoujinjian.cn博客原图链接】](https://github.com/zhoujinjianx) 

[【开发板 Khadas Edge V】](https://www.khadas.cn/edge-v)

[【开发板 Khadas Edge V Android 10.0 && Linux（Kernel 4.19）源码链接】](https://github.com/khadas/linux/tree/khadas-edge-Qt)



正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢！！！

 [Android camera预览流程](https://blog.csdn.net/weixin_41944449/article/details/102609776) 


--------------------------------------------------------------------------------

==源码（部分）==：

**F:\Khadas_Edge_Android_Q\frameworks\av\services\camera\libcameraservice\**

*F:\Khadas_Edge_Android_Q\frameworks\base\core\java\android\hardware\camera2\*

==源码（部分）==：

--------------------------------------------------------------------------------



## （一）、configure_streams()

#### （1）、Stack::createCaptureSessionInternal()  

```java
07-07 11:03:48.619  1727  1771 I CameraDevice-JV-0: configureStreamsChecked
07-07 11:03:48.619  1727  1771 I CameraDevice-JV-0: java.lang.RuntimeException: here
07-07 11:03:48.619  1727  1771 I CameraDevice-JV-0: 	at android.hardware.camera2.impl.CameraDeviceImpl.createCaptureSessionInternal(CameraDeviceImpl.java:667)
07-07 11:03:48.619  1727  1771 I CameraDevice-JV-0: 	at android.hardware.camera2.impl.CameraDeviceImpl.createCaptureSession(CameraDeviceImpl.java:514)
07-07 11:03:48.619  1727  1771 I CameraDevice-JV-0: 	at com.android.camera.one.v2.camera2proxy.AndroidCameraDeviceProxy.createCaptureSession(AndroidCameraDeviceProxy.java:84)
07-07 11:03:48.619  1727  1771 I CameraDevice-JV-0: 	at com.android.camera.one.v2.initialization.CaptureSessionCreator.createCaptureSession(CaptureSessionCreator.java:58)
07-07 11:03:48.619  1727  1771 I CameraDevice-JV-0: 	at com.android.camera.one.v2.initialization.PreviewStarter.startPreview(PreviewStarter.java:83)
07-07 11:03:48.619  1727  1771 I CameraDevice-JV-0: 	at com.android.camera.one.v2.initialization.GenericOneCameraImpl.startPreview(GenericOneCameraImpl.java:151)
07-07 11:03:48.619  1727  1771 I CameraDevice-JV-0: 	at com.android.camera.CaptureModule$19.onCameraOpened(CaptureModule.java:1564)
07-07 11:03:48.619  1727  1771 I CameraDevice-JV-0: 	at com.android.camera.one.v2.Camera2OneCameraOpenerImpl$1.onOpened(Camera2OneCameraOpenerImpl.java:180)
07-07 11:03:48.619  1727  1771 I CameraDevice-JV-0: 	at android.hardware.camera2.impl.CameraDeviceImpl$1.run(CameraDeviceImpl.java:146)
07-07 11:03:48.619  1727  1771 I CameraDevice-JV-0: 	at android.os.Handler.handleCallback(Handler.java:883)
07-07 11:03:48.619  1727  1771 I CameraDevice-JV-0: 	at android.os.Handler.dispatchMessage(Handler.java:100)
07-07 11:03:48.619  1727  1771 I CameraDevice-JV-0: 	at android.os.Looper.loop(Looper.java:214)
07-07 11:03:48.619  1727  1771 I CameraDevice-JV-0: 	at android.os.HandlerThread.run(HandlerThread.java:67)
```



#### （2）、CameraDeviceImpl.createCaptureSessionInternal() 

```java
F:\Khadas_Edge_Android_Q\frameworks\base\core\java\android\hardware\camera2\impl\CameraDeviceImpl.java
configureStreamsChecked(InputConfiguration inputConfig,
            List<OutputConfiguration> outputs, int operatingMode, CaptureRequest sessionParams)
                    throws CameraAccessException {
        ......
        checkInputConfiguration(inputConfig);

        boolean success = false;

        synchronized(mInterfaceLock) {
            checkIfCameraClosedOrInError();
            // Streams to create
            HashSet<OutputConfiguration> addSet = new HashSet<OutputConfiguration>(outputs);
            ......
            try {
                waitUntilIdle();

                mRemoteDevice.beginConfigure();
                ......

                // Add all new streams
                Slog.i(TAG, "createStream", new RuntimeException("here").fillInStackTrace());                
                for (OutputConfiguration outConfig : outputs) {
                    if (addSet.contains(outConfig)) {
                        /* 将会在这里创建并配置输出流，该流输出显示，
                         * 注意 outConfig 中有显示 surface
                         */
                        int streamId = mRemoteDevice.createStream(outConfig);
                        mConfiguredOutputs.put(streamId, outConfig);
                    }
                }
                /* 注意传递进来的方法参数，inputConfig为null，sessionParams为null */
                if (sessionParams != null) {
                    mRemoteDevice.endConfigure(operatingMode, sessionParams.getNativeCopy());
                } else {
                    /* 上述只是保存配置，这里才会真正的设置到 HAL */
                    mRemoteDevice.endConfigure(operatingMode, null);
                }

                success = true;
            } ......
        }

        return success;
    }

```

>  07-07 11:03:48.621   334   544 V Camera3-Device: Camera 0: Creating new stream 0: 4096 x 3120, format 33, dataspace 146931712 rotation 0 consumer usage 0, isShared 0, physicalCameraId 

我们主要分析 mRemoteDevice.createStream(outConfig) 的操作，在这里，将会通过Binder将相应的配置信息传输到 CameraService 端的CameraDeviceClient。

#### （3）、CameraDeviceClient::createStream()

```java
F:\Khadas_Edge_Android_Q\frameworks\av\services\camera\libcameraservice\api2\CameraDeviceClient.cpp
        public void openCamera(@NonNull String cameraId,
            @NonNull final CameraDevice.StateCallback callback, @Nullable Handler handler)
            throws CameraAccessException {

        openCameraForUid(cameraId, callback, CameraDeviceImpl.checkAndWrapHandler(handler),
                USE_CALLING_UID);
    }
openCameraForUid->openCameraDeviceUserAsync
    
 binder::Status CameraDeviceClient::createStream(
        const hardware::camera2::params::OutputConfiguration &outputConfiguration,
        /*out*/
        int32_t* newStreamId) {
    ATRACE_CALL();

    binder::Status res;
    Mutex::Autolock icl(mBinderSerializationLock);

    const std::vector<sp<IGraphicBufferProducer>>& bufferProducers =
            outputConfiguration.getGraphicBufferProducers();
    size_t numBufferProducers = bufferProducers.size();
    bool deferredConsumer = outputConfiguration.isDeferred();
    bool isShared = outputConfiguration.isShared();
    String8 physicalCameraId = String8(outputConfiguration.getPhysicalCameraId());
    bool deferredConsumerOnly = deferredConsumer && numBufferProducers == 0;

    res = checkSurfaceTypeLocked(numBufferProducers, deferredConsumer,
            outputConfiguration.getSurfaceType());

    res = checkPhysicalCameraIdLocked(physicalCameraId);

    std::vector<sp<Surface>> surfaces;
    std::vector<sp<IBinder>> binders;
    status_t err;

    // Create stream for deferred surface case.
    if (deferredConsumerOnly) {
        return createDeferredSurfaceStreamLocked(outputConfiguration, isShared, newStreamId);
    }

    OutputStreamInfo streamInfo;
    bool isStreamInfoValid = false;
    for (auto& bufferProducer : bufferProducers) {
        // Don't create multiple streams for the same target surface
        sp<IBinder> binder = IInterface::asBinder(bufferProducer);
        ssize_t index = mStreamMap.indexOfKey(binder);
        if (index != NAME_NOT_FOUND) {
            String8 msg = String8::format("Camera %s: Surface already has a stream created for it "
                    "(ID %zd)", mCameraIdStr.string(), index);
            ALOGW("%s: %s", __FUNCTION__, msg.string());
            return STATUS_ERROR(CameraService::ERROR_ALREADY_EXISTS, msg.string());
        }

        sp<Surface> surface;
        res = createSurfaceFromGbp(streamInfo, isStreamInfoValid, surface, bufferProducer,
                physicalCameraId);

        if (!res.isOk())
            return res;

        if (!isStreamInfoValid) {
            isStreamInfoValid = true;
        }

        binders.push_back(IInterface::asBinder(bufferProducer));
        surfaces.push_back(surface);
    }

    int streamId = camera3::CAMERA3_STREAM_ID_INVALID;
    std::vector<int> surfaceIds;
    bool isDepthCompositeStream = camera3::DepthCompositeStream::isDepthCompositeStream(surfaces[0]);
    bool isHeicCompisiteStream = camera3::HeicCompositeStream::isHeicCompositeStream(surfaces[0]);
    if (isDepthCompositeStream || isHeicCompisiteStream) {
        ......
    } else {
        err = mDevice->createStream(surfaces, deferredConsumer, streamInfo.width,
                streamInfo.height, streamInfo.format, streamInfo.dataSpace,
                static_cast<camera3_stream_rotation_t>(outputConfiguration.getRotation()),
                &streamId, physicalCameraId, &surfaceIds, outputConfiguration.getSurfaceSetID(),
                isShared);
    }

    if (err != OK) {
       ......
    } else {
        int i = 0;
        for (auto& binder : binders) {
            ALOGV("%s: mStreamMap add binder %p streamId %d, surfaceId %d",
                    __FUNCTION__, binder.get(), streamId, i);
            mStreamMap.add(binder, StreamSurfaceId(streamId, surfaceIds[i]));
            i++;
        }

        mConfiguredOutputs.add(streamId, outputConfiguration);
        mStreamInfoMap[streamId] = streamInfo;

        ALOGV("%s: Camera %s: Successfully created a new stream ID %d for output surface"
                    " (%d x %d) with format 0x%x.",
                  __FUNCTION__, mCameraIdStr.string(), streamId, streamInfo.width,
                  streamInfo.height, streamInfo.format);

        // Set transform flags to ensure preview to be rotated correctly.
        res = setStreamTransformLocked(streamId);

        *newStreamId = streamId;
    }

    return res;
}
Log:
07-07 11:03:48.666   334   544 V Camera3-Device: Camera 0: Created new stream
07-07 11:03:48.666   334   544 V CameraDeviceClient: createStream: mStreamMap add binder 0xe90162c0 streamId 0, surfaceId 0
07-07 11:03:48.666   334   544 V CameraDeviceClient: createStream: Camera 0: Successfully created a new stream ID 0 for output surface (4096 x 3120) with format 0x21.
    
Stack Trace:
  RELADDR   FUNCTION                                                                                                                                                                                                                                                                                                                                                      FILE:LINE
  001034a5  android::Camera3Device::createStream(std::__1::vector<android::sp<android::Surface>, std::__1::allocator<android::sp<android::Surface> > > const&, bool, unsigned int, unsigned int, int, android_dataspace_t, camera3_stream_rotation, int*, android::String8 const&, std::__1::vector<int, std::__1::allocator<int> >*, int, bool, unsigned long long)+172  frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp:1758
  000dcbd5  android::CameraDeviceClient::createStream(android::hardware::camera2::params::OutputConfiguration const&, int*)+1300                                                                                                                                                                                                                                          frameworks/av/services/camera/libcameraservice/api2/CameraDeviceClient.cpp:977
  000dd4c9  non-virtual thunk to android::CameraDeviceClient::createStream(android::hardware::camera2::params::OutputConfiguration const&, int*)+4                                                                                                                                                                                                                        aeabi_div0.c:?
  00028509  android::hardware::camera2::BnCameraDeviceUser::onTransact(unsigned int, android::Parcel const&, android::Parcel*, unsigned int)+720                                                                                                                                                                                                                          out/soong/.intermediates/frameworks/av/camera/libcamera_client/android_arm_armv7-a-neon_cortex-a15_core_shared/gen/aidl/frameworks/av/camera/aidl/android/hardware/camera2/ICameraDeviceUser.cpp:1025
  00032ed1  android::BBinder::transact(unsigned int, android::Parcel const&, android::Parcel*, unsigned int)+72                                                                                                                                                                                                                                                           frameworks/native/libs/binder/Binder.cpp:134
  0003b117  android::IPCThreadState::executeCommand(int)+770                                                                                                                                                                                                                                                                                                              frameworks/native/libs/binder/IPCThreadState.cpp:1213
  0003ad53  android::IPCThreadState::getAndExecuteCommand()+98                                                                                                                                                                                                                                                                                                            frameworks/native/libs/binder/IPCThreadState.cpp:514
  0003b2ef  android::IPCThreadState::joinThreadPool(bool)+38                                                                                                                                                                                                                                                                                                              frameworks/native/libs/binder/IPCThreadState.cpp:594
  00054665  android::PoolThread::threadLoop()+12                                                                                                                                                                                                                                                                                                                          frameworks/native/libs/binder/ProcessState.cpp:67
  0000d8bf  android::Thread::_threadLoop(void*)+186                                                                                                                                                                                                                                                                                                                       system/core/libutils/Threads.cpp:746
  000a6867  __pthread_start(void*)+20                                                                                                                                                                                                                                                                                                                                     bionic/libc/bionic/pthread_create.cpp:338
```

上面的 mDevice->createStream() 将会调用到哪里呢？

CameraDeviceClient 类是继承模板类 Camera2ClientBase，看 mDevice 的定义，我们可以知道是 sp<CameraDeviceBase> mDevice，mDevice 的赋值是在模板类Camera2ClientBase 的构造函数中，mDevice = new Camera3Device(cameraId)，通过其构造函数可以知道 mDevice 指向 Camera3Device 实例对象。所以，mDevice->createStream() 将调用到 Camera3Device::createStream() 函数。

#### （4）、Camera3Device::createStream()



```cpp
F:\Khadas_Edge_Android_Q\frameworks\av\services\camera\libcameraservice\device3\Camera3Device.cpp                                                                                                        status_t Camera3Device::createStream(const std::vector<sp<Surface>>& consumers,
        bool hasDeferredConsumer, uint32_t width, uint32_t height, int format,
        android_dataspace dataSpace, camera3_stream_rotation_t rotation, int *id,
        const String8& physicalCameraId,
        std::vector<int> *surfaceIds, int streamSetId, bool isShared, uint64_t consumerUsage) {
	...
  
    ALOGV("Camera %s: Creating new stream %d: %d x %d, format %d, dataspace %d rotation %d"
            " consumer usage %" PRIu64 ", isShared %d, physicalCameraId %s", mId.string(),
            mNextStreamId, width, height, format, dataSpace, rotation, consumerUsage, isShared,
            physicalCameraId.string());
    } else {
    	/* 创建 Camera3OutputStream 实例对象，注意第二个参数，它是 Surface，
    	 * 将保存在 Camera3OutputStream 的 mConsumer 成员 */
        newStream = new Camera3OutputStream(mNextStreamId, consumers[0],
                width, height, format, dataSpace, rotation,
                mTimestampOffset, physicalCameraId, streamSetId);
    }

	...

	/* newStream 实例对象保存到 mOutputStreams */
    res = mOutputStreams.add(mNextStreamId, newStream);

	mNeedConfig = true;

	...      
Log：
```

我们回头看看 CameraDeviceImpl::configureStreamsChecked() 方法，在该方法中，通过 mRemoteDevice 操作影响到 CameraService，在 createStream() 之后，还将会调用 endConfigure() 操作

#### （5）、CameraDeviceClient::endConfigure()

```cpp
F:\Khadas_Edge_Android_Q\frameworks\av\services\camera\libcameraservice\api2\CameraDeviceClient.cpp
binder::Status CameraDeviceClient::endConfigure(int operatingMode,
        const hardware::camera2::impl::CameraMetadataNative& sessionParams) {
    ...
	/* 和上述一样，将会调用到 Camera3Device::configureStreams() */
    status_t err = mDevice->configureStreams(sessionParams, operatingMode);
    ...
}
```

#### （6）、Camera3Device::HalInterface::configureStreams->Stack Trace

``` cpp
Stack Trace:
  RELADDR   FUNCTION                                                                                                                                                                                       FILE:LINE
  001079bf  android::Camera3Device::HalInterface::configureStreams(camera_metadata const*, camera3_stream_configuration*, std::__1::vector<unsigned int, std::__1::allocator<unsigned int> > const&)+1622  frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp:4341
  00102bf9  android::Camera3Device::configureStreamsLocked(int, android::CameraMetadata const&, bool)+836                                                                                                  frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp:2901
  00101c01  android::Camera3Device::filterParamsAndConfigureLocked(android::CameraMetadata const&, int)+136                                                                                                frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp:2054
  00104359  android::Camera3Device::configureStreams(android::CameraMetadata const&, int)+124                                                                                                              aeabi_div0.c:?
  000da883  android::CameraDeviceClient::endConfigure(int, android::CameraMetadata const&)+310                                                                                                             frameworks/av/services/camera/libcameraservice/api2/CameraDeviceClient.cpp:491
  000dac21  non-virtual thunk to android::CameraDeviceClient::endConfigure(int, android::CameraMetadata const&)+4                                                                                          aeabi_div0.c:?
  0002842f  android::hardware::camera2::BnCameraDeviceUser::onTransact(unsigned int, android::Parcel const&, android::Parcel*, unsigned int)+502                                                           out/soong/.intermediates/frameworks/av/camera/libcamera_client/android_arm_armv7-a-neon_cortex-a15_core_shared/gen/aidl/frameworks/av/camera/aidl/android/hardware/camera2/ICameraDeviceUser.cpp:956
  00032ed1  android::BBinder::transact(unsigned int, android::Parcel const&, android::Parcel*, unsigned int)+72                                                                                            frameworks/native/libs/binder/Binder.cpp:134
  0003b117  android::IPCThreadState::executeCommand(int)+770                                                                                                                                               frameworks/native/libs/binder/IPCThreadState.cpp:1213
  0003ad53  android::IPCThreadState::getAndExecuteCommand()+98                                                                                                                                             frameworks/native/libs/binder/IPCThreadState.cpp:514
  0003b2ef  android::IPCThreadState::joinThreadPool(bool)+38                                                                                                                                               frameworks/native/libs/binder/IPCThreadState.cpp:594
  00054665  android::PoolThread::threadLoop()+12                                                                                                                                                           frameworks/native/libs/binder/ProcessState.cpp:67
  0000d8bf  android::Thread::_threadLoop(void*)+186                                                                                                                                                        system/core/libutils/Threads.cpp:746
  000a6867  __pthread_start(void*)+20                                                                                                                                                                      bionic/libc/bionic/pthread_create.cpp:338
  00060087  __start_thread+30                                                                                                                                                                              bionic/libc/bionic/clone.cpp:53

```
而在 Camera3Device::configureStreams() 中，最终又将是调用到 Camera3Device::configureStreamsLocked() 函数。而在 Camera3Device::configureStreamsLocked() 函数主要进行了以下操作：将 Camera3Device::createStream() 保存到 - - mOutputStreams 中的 stream 添加到 config.streams；
通过 mInterface->configureStreams(sessionBuffer, &config, bufferSizes) 操作到HAL配置流；
通知上层，配置完成；

#### （7）、Camera3Device::HalInterface::configureStreams()

```cpp
F:\Khadas_Edge_Android_Q\frameworks\av\services\camera\libcameraservice\device3\Camera3Device.cpp
    
status_t Camera3Device::HalInterface::configureStreams(const camera_metadata_t *sessionParams,
        camera3_stream_configuration *config, const std::vector<uint32_t>& bufferSizes) {
    ATRACE_NAME("CameraHal::configureStreams");
    ......
    } else if (mHidlSession_3_3 != nullptr) {
        // We do; use v3.3 for the call
		
		CallStack stack;
		stack.update();
		stack.log("Camera3Device::configureStreams_3_3");

        ALOGV("%s: v3.3 device found", __FUNCTION__);
        auto err = mHidlSession_3_3->configureStreams_3_3(requestedConfiguration3_2,
            [&status, &finalConfiguration]
            (common::V1_0::Status s, const device::V3_3::HalStreamConfiguration& halConfiguration) {
                finalConfiguration = halConfiguration;
                status = s;
            });
        if (!err.isOk()) {
            ALOGE("%s: Transaction error: %s", __FUNCTION__, err.description().c_str());
            return DEAD_OBJECT;
        }
    } else {
        ......
    }

    ......

    return res;
}

```

上述的 mInterface 变量是在 Camera3Device::initialize() 时，保存 HalInterface 实例对象的，所以，mInterface->configureStreams() 将调用到 Camera3Device::HalInterface::configureStreams()。

Camera3Device::HalInterface::configureStreams() 将stream config转换为 HIDL 接口使用的之后，将按照 HAL 版本调用不同的接口函数，下面我们以 HAL 3.3 为例进行跟踪，将调用 mHidlSession_3_3->configureStreams_3_3()。注：mHidlSession_3_3 是在 HalInterface 构造函数中赋值的，而 mHidlSession_3_3 是 ICameraDeviceSession转换得来的，ICameraDeviceSession 则是Camera3Device::initialize() 中通过 manager->openSession() 获取的。

在上述的 mHidlSession_3_3->configureStreams_3_3() 调用，将会调用到 BpHwCameraDeviceSession::configureStreams_3_3() [android/out/soong/.intermediates/hardware/interfaces/camera/device/3.3/android.hardware.camera.device@3.3_genc++/gen/android/hardware/camera/device/3.3/CameraDeviceSessionAll.cpp]，最终调用到 CameraProvider 中的 CameraDeviceSession::configureStreams_3_3()函数。通过HIDL配置流操作进入 CameraProvider。

#### （8）、Camera3HAL::configure_streams()->Stack Trace

```cpp
Stack Trace: configure_streams()
  RELADDR   FUNCTION                                                                                                                                                                                                                                                                                                                                                        FILE:LINE
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

```



## （二）、construct_default_request_settings()



#### （1）、Stack::createCaptureRequest()

```cpp
07-07 11:03:48.923  1727  1727 V CAM_CameraAppUI: onPreviewStarted

07-07 11:03:48.924  1727  1826 I CameraDevice-JV-0: createDefaultRequest
07-07 11:03:48.924  1727  1826 I CameraDevice-JV-0: java.lang.RuntimeException: here
07-07 11:03:48.924  1727  1826 I CameraDevice-JV-0: 	at android.hardware.camera2.impl.CameraDeviceImpl.createCaptureRequest(CameraDeviceImpl.java:781)
07-07 11:03:48.924  1727  1826 I CameraDevice-JV-0: 	at com.android.camera.one.v2.camera2proxy.AndroidCameraDeviceProxy.createCaptureRequest(AndroidCameraDeviceProxy.java:90)
07-07 11:03:48.924  1727  1826 I CameraDevice-JV-0: 	at com.android.camera.one.v2.camera2proxy.CameraDeviceRequestBuilderFactory.create(CameraDeviceRequestBuilderFactory.java:38)
07-07 11:03:48.924  1727  1826 I CameraDevice-JV-0: 	at com.android.camera.one.v2.core.RequestTemplate.create(RequestTemplate.java:104)
07-07 11:03:48.924  1727  1826 I CameraDevice-JV-0: 	at com.android.camera.one.v2.core.RequestTemplate.create(RequestTemplate.java:104)
07-07 11:03:48.924  1727  1826 I CameraDevice-JV-0: 	at com.android.camera.one.v2.commands.PreviewCommand.run(PreviewCommand.java:56)
07-07 11:03:48.924  1727  1826 I CameraDevice-JV-0: 	at com.android.camera.one.v2.commands.CameraCommandExecutor$CommandRunnable.run(CameraCommandExecutor.java:52)
07-07 11:03:48.924  1727  1826 I CameraDevice-JV-0: 	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:462)
07-07 11:03:48.924  1727  1826 I CameraDevice-JV-0: 	at java.util.concurrent.FutureTask.run(FutureTask.java:266)
07-07 11:03:48.924  1727  1826 I CameraDevice-JV-0: 	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
07-07 11:03:48.924  1727  1826 I CameraDevice-JV-0: 	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
07-07 11:03:48.924  1727  1826 I CameraDevice-JV-0: 	at java.lang.Thread.run(Thread.java:919)
```


#### （2）、CameraDeviceImpl.createCaptureRequest()

```cpp
F:\Khadas_Edge_Android_Q\frameworks\base\core\java\android\hardware\camera2\impl\CameraDeviceImpl.java
   @Override
    public CaptureRequest.Builder createCaptureRequest(int templateType)
            throws CameraAccessException {
        synchronized(mInterfaceLock) {
            checkIfCameraClosedOrInError();
    
            CameraMetadataNative templatedRequest = null;
    
            /* 一样的，也是通过 mRemoteDevice 代理操作 CameraDeviceClient，
             * 将调用到 CameraDeviceClient::createDefaultRequest() */
            templatedRequest = mRemoteDevice.createDefaultRequest(templateType);
    
            ...
    
            return builder;
        }
    } 
```



#### （3）、CameraDeviceClient::createDefaultRequest()

``` cpp
F:\Khadas_Edge_Android_Q\frameworks\av\services\camera\libcameraservice\api2\CameraDeviceClient.cpp
binder::Status CameraDeviceClient::createDefaultRequest(int templateId,
        /*out*/
        hardware::camera2::impl::CameraMetadataNative* request)
{
    ...
 
    CameraMetadata metadata;
    status_t err; 
    /* 与 createStream() 相同，将会调用到 Camera3Device::createDefaultRequest() */
    if ( (err = mDevice->createDefaultRequest(templateId, &metadata) ) == OK &&                                                                                                                              
        request != NULL) {
 
        request->swap(metadata);
    }
    ... 
}
```

#### （4）、Camera3Device::createDefaultRequest()

```cpp
F:\Khadas_Edge_Android_Q\frameworks\av\services\camera\libcameraservice\common\CameraProviderManager.cpp
status_t Camera3Device::createDefaultRequest(int templateId,
        CameraMetadata *request) {
	...
    camera_metadata_t *rawRequest;
    /* 又将通过 mInterface 调用到 CameraProvider*/
    status_t res = mInterface->constructDefaultRequestSettings(
            (camera3_request_template_t) templateId, &rawRequest);
    
	{
		...
		/* 保存这个类型请求的信息 */
        set_camera_metadata_vendor_id(rawRequest, mVendorTagId);
        mRequestTemplateCache[templateId].acquire(rawRequest);
        ...
    }
}
```

#### （5）、Camera3Device::createDefaultRequest()->Stack Trace

```cpp
Stack Trace:
  RELADDR   FUNCTION                                                                                                                               FILE:LINE
  00104575  android::Camera3Device::createDefaultRequest(int, android::CameraMetadata*)+208                                                        frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp:2112
  000de733  android::CameraDeviceClient::createDefaultRequest(int, android::CameraMetadata*)+206                                                   frameworks/av/services/camera/libcameraservice/api2/CameraDeviceClient.cpp:1535
  000de8a1  non-virtual thunk to android::CameraDeviceClient::createDefaultRequest(int, android::CameraMetadata*)+4                                aeabi_div0.c:?
  0002863f  android::hardware::camera2::BnCameraDeviceUser::onTransact(unsigned int, android::Parcel const&, android::Parcel*, unsigned int)+1030  out/soong/.intermediates/frameworks/av/camera/libcamera_client/android_arm_armv7-a-neon_cortex-a15_core_shared/gen/aidl/frameworks/av/camera/aidl/android/hardware/camera2/ICameraDeviceUser.cpp:1108
  00032ed1  android::BBinder::transact(unsigned int, android::Parcel const&, android::Parcel*, unsigned int)+72                                    frameworks/native/libs/binder/Binder.cpp:134
  0003b117  android::IPCThreadState::executeCommand(int)+770                                                                                       frameworks/native/libs/binder/IPCThreadState.cpp:1213
  0003ad53  android::IPCThreadState::getAndExecuteCommand()+98                                                                                     frameworks/native/libs/binder/IPCThreadState.cpp:514
  0003b2ef  android::IPCThreadState::joinThreadPool(bool)+38                                                                                       frameworks/native/libs/binder/IPCThreadState.cpp:594
  00054665  android::PoolThread::threadLoop()+12                                                                                                   frameworks/native/libs/binder/ProcessState.cpp:67
  0000d8bf  android::Thread::_threadLoop(void*)+186                                                                                                system/core/libutils/Threads.cpp:746
  000a6867  __pthread_start(void*)+20                                                                                                              bionic/libc/bionic/pthread_create.cpp:338
  00060087  __start_thread+30                                                                                                                      bionic/libc/bionic/clone.cpp:53
```

#### （6）、Camera3HAL::construct_default_request_settings()->Stack Trace


```cpp
Stack Trace:
  RELADDR   FUNCTION                                                                                                                                                                                                                                                                                                                                   FILE:LINE
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
  v------>  int android::hardware::defaultPassthroughServiceImplementation<android::hardware::camera::provider::V2_4::ICameraProvider>(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, unsigned int)                                                                                                     system/libhidl/transport/include/hidl/LegacySupport.h:85
  000010b9  main+116                                                                                                                                                                                                                                                                                                                                   hardware/interfaces/camera/provider/2.4/default/service.cpp:49
  00059127  __libc_init+66                                                                                                                                                                                                                                                                                                                             bionic/libc/bionic/libc_init_dynamic.cpp:136
  0000102f  _start_main+38                                                                                                                                                                                                                                                                                                                             bionic/libc/arch-common/bionic/crtbegin.c:45
  00004456  <unknown>                                                                                                                                                                                                                                                                                                                                  <anonymous:ec7ce000>


```



## （三）、process_capture_request()

#### (1)、ICameraDeviceUserWrapper.submitRequestList()

```java
07-07 11:03:49.003  1727  1826 I ICameraDeviceUserWrapper: submitRequestList
07-07 11:03:49.003  1727  1826 I ICameraDeviceUserWrapper: java.lang.RuntimeException: here
07-07 11:03:49.003  1727  1826 I ICameraDeviceUserWrapper: 	at android.hardware.camera2.impl.ICameraDeviceUserWrapper.submitRequestList(ICameraDeviceUserWrapper.java:88)
07-07 11:03:49.003  1727  1826 I ICameraDeviceUserWrapper: 	at android.hardware.camera2.impl.CameraDeviceImpl.submitCaptureRequest(CameraDeviceImpl.java:1074)
07-07 11:03:49.003  1727  1826 I ICameraDeviceUserWrapper: 	at android.hardware.camera2.impl.CameraDeviceImpl.setRepeatingBurst(CameraDeviceImpl.java:1127)
07-07 11:03:49.003  1727  1826 I ICameraDeviceUserWrapper: 	at android.hardware.camera2.impl.CameraCaptureSessionImpl.setRepeatingBurst(CameraCaptureSessionImpl.java:352)
07-07 11:03:49.003  1727  1826 I ICameraDeviceUserWrapper: 	at com.android.camera.one.v2.camera2proxy.AndroidCameraCaptureSessionProxy.setRepeatingBurst(AndroidCameraCaptureSessionProxy.java:129)
07-07 11:03:49.003  1727  1826 I ICameraDeviceUserWrapper: 	at com.android.camera.one.v2.core.TagDispatchCaptureSession.submitRequest(TagDispatchCaptureSession.java:154)
07-07 11:03:49.003  1727  1826 I ICameraDeviceUserWrapper: 	at com.android.camera.one.v2.core.FrameServerImpl$Session.submitRequest(FrameServerImpl.java:58)
07-07 11:03:49.003  1727  1826 I ICameraDeviceUserWrapper: 	at com.android.camera.one.v2.commands.PreviewCommand.run(PreviewCommand.java:57)
07-07 11:03:49.003  1727  1826 I ICameraDeviceUserWrapper: 	at com.android.camera.one.v2.commands.CameraCommandExecutor$CommandRunnable.run(CameraCommandExecutor.java:52)
07-07 11:03:49.003  1727  1826 I ICameraDeviceUserWrapper: 	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:462)
07-07 11:03:49.003  1727  1826 I ICameraDeviceUserWrapper: 	at java.util.concurrent.FutureTask.run(FutureTask.java:266)
07-07 11:03:49.003  1727  1826 I ICameraDeviceUserWrapper: 	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
07-07 11:03:49.003  1727  1826 I ICameraDeviceUserWrapper: 	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
07-07 11:03:49.003  1727  1826 I ICameraDeviceUserWrapper: 	at java.lang.Thread.run(Thread.java:919)
```

#### (2)、ICameraDeviceUserWrapper.submitRequestList()

```java
F:\Khadas_Edge_Android_Q\frameworks\base\core\java\android\hardware\camera2\impl\ICameraDeviceUserWrapper.java
public class ICameraDeviceUserWrapper {
    ...
	public SubmitInfo submitRequest(CaptureRequest request, boolean streaming)
            throws CameraAccessException  {
        try {
            /* 调用到 CameraService 的 CameraDeviceClient */
            return mRemoteDevice.submitRequest(request, streaming);
        } catch (Throwable t) {
            CameraManager.throwAsPublicException(t);
            throw new UnsupportedOperationException("Unexpected exception", t);                                                                                                                              
        }   
    } 
}
```

#### (3)、Camera3Device::RequestThread::unpauseForNewRequests() ->Stack Trace

```cpp
Stack Trace:
  RELADDR   FUNCTION                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 FILE:LINE
  0010c575  android::Camera3Device::RequestThread::unpauseForNewRequests()+64                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp:6218
  000febc3  android::Camera3Device::RequestThread::queueRequestList(android::List<android::sp<android::Camera3Device::CaptureRequest> >&, long long*)+198                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp:4992
  000fe8bd  android::Camera3Device::submitRequestsHelper(android::List<android::List<android::CameraDeviceBase::PhysicalCameraSettings> const> const&, std::__1::list<std::__1::unordered_map<int, std::__1::vector<unsigned int, std::__1::allocator<unsigned int> >, std::__1::hash<int>, std::__1::equal_to<int>, std::__1::allocator<std::__1::pair<int const, std::__1::vector<unsigned int, std::__1::allocator<unsigned int> > > > > const, std::__1::allocator<std::__1::unordered_map<int, std::__1::vector<unsigned int, std::__1::allocator<unsigned int> >, std::__1::hash<int>, std::__1::equal_to<int>, std::__1::allocator<std::__1::pair<int const, std::__1::vector<unsigned int, std::__1::allocator<unsigned int> > > > > const> > const&, bool, long long*)+248  frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp:967
  001019d5  android::Camera3Device::captureList(android::List<android::List<android::CameraDeviceBase::PhysicalCameraSettings> const> const&, std::__1::list<std::__1::unordered_map<int, std::__1::vector<unsigned int, std::__1::allocator<unsigned int> >, std::__1::hash<int>, std::__1::equal_to<int>, std::__1::allocator<std::__1::pair<int const, std::__1::vector<unsigned int, std::__1::allocator<unsigned int> > > > > const, std::__1::allocator<std::__1::unordered_map<int, std::__1::vector<unsigned int, std::__1::allocator<unsigned int> >, std::__1::hash<int>, std::__1::equal_to<int>, std::__1::allocator<std::__1::pair<int const, std::__1::vector<unsigned int, std::__1::allocator<unsigned int> > > > > const> > const&, long long*)+52                  frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp:1523
  000d9e69  android::CameraDeviceClient::submitRequestList(std::__1::vector<android::hardware::camera2::CaptureRequest, std::__1::allocator<android::hardware::camera2::CaptureRequest> > const&, bool, android::hardware::camera2::utils::SubmitInfo*)+3432                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         frameworks/av/services/camera/libcameraservice/api2/CameraDeviceClient.cpp:405
  000da42b  non-virtual thunk to android::CameraDeviceClient::submitRequestList(std::__1::vector<android::hardware::camera2::CaptureRequest, std::__1::allocator<android::hardware::camera2::CaptureRequest> > const&, bool, android::hardware::camera2::utils::SubmitInfo*)+14                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      aeabi_div0.c:?
  00028385  android::hardware::camera2::BnCameraDeviceUser::onTransact(unsigned int, android::Parcel const&, android::Parcel*, unsigned int)+332                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     out/soong/.intermediates/frameworks/av/camera/libcamera_client/android_arm_armv7-a-neon_cortex-a15_core_shared/gen/aidl/frameworks/av/camera/aidl/android/hardware/camera2/ICameraDeviceUser.cpp:884
  00032ed1  android::BBinder::transact(unsigned int, android::Parcel const&, android::Parcel*, unsigned int)+72                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      frameworks/native/libs/binder/Binder.cpp:134
  0003b117  android::IPCThreadState::executeCommand(int)+770                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         frameworks/native/libs/binder/IPCThreadState.cpp:1213
  0003ad53  android::IPCThreadState::getAndExecuteCommand()+98                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       frameworks/native/libs/binder/IPCThreadState.cpp:514
  0003b2ef  android::IPCThreadState::joinThreadPool(bool)+38                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         frameworks/native/libs/binder/IPCThreadState.cpp:594
  00054665  android::PoolThread::threadLoop()+12                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     frameworks/native/libs/binder/ProcessState.cpp:67
  0000d8bf  android::Thread::_threadLoop(void*)+186                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  system/core/libutils/Threads.cpp:746
  000a6867  __pthread_start(void*)+20                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                bionic/libc/bionic/pthread_create.cpp:338
  00060087  __start_thread+30                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        bionic/libc/bionic/clone.cpp:53

```

 

Camera3Device::RequestThread::threadLoop()中waitForNextRequestBatch();在等待请求，mRequestSignal.signal唤醒线程后继续运行。

#### (4)、Camera3Device::HalInterface::processBatchCaptureRequest->Stack Trace

```cpp
Stack Trace:
  RELADDR   FUNCTION                                                                                                                                                                            FILE:LINE
  0010b155  android::Camera3Device::HalInterface::processBatchCaptureRequests(std::__1::vector<camera3_capture_request*, std::__1::allocator<camera3_capture_request*> >&, unsigned int*)+1296  frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp:4685
  0010cac9  android::Camera3Device::RequestThread::sendRequestsBatch()+264                                                                                                                      frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp:5232
  0010db51  android::Camera3Device::RequestThread::threadLoop()+664                                                                                                                             frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp:5526
  0000d945  android::Thread::_threadLoop(void*)+320                                                                                                                                             system/core/libutils/Threads.cpp:749
  000a6867  __pthread_start(void*)+20                                                                                                                                                           bionic/libc/bionic/pthread_create.cpp:338
  00060087  __start_thread+30                                                                                                                                                                   bionic/libc/bionic/clone.cpp:53

```

#### (5)、Camera3HAL::process_capture_request()->Stack Trace

```cpp
Stack Trace:
  RELADDR   FUNCTION                                                                                                                                                                                                                                                                                                                                                                                                                 FILE:LINE
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
  v------>  int android::hardware::defaultPassthroughServiceImplementation<android::hardware::camera::provider::V2_4::ICameraProvider>(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, unsigned int)                                                                                                                                                                                   system/libhidl/transport/include/hidl/LegacySupport.h:85
  000010b9  main+116                                                                                                                                                                                                                                                                                                                                                                                                                 hardware/interfaces/camera/provider/2.4/default/service.cpp:49
  00059127  __libc_init+66                                                                                                                                                                                                                                                                                                                                                                                                           bionic/libc/bionic/libc_init_dynamic.cpp:136
  0000102f  _start_main+38                                                                                                                                                                                                                                                                                                                                                                                                           bionic/libc/arch-common/bionic/crtbegin.c:45
  00004456  <unknown>                                                                                                                                                                                                                                                                                                                                                                                                                <anonymous:ec7ce000>
```

## (四)、processCaptureResult()

#### (1)、CameraDeviceSession::ResultBatcher::invokeProcessCaptureResultCallback()->Stack Trace

```cpp
Stack Trace:
  RELADDR   FUNCTION                                                                                                                                                                                                                         FILE:LINE
  00018b5d  android::hardware::camera::device::V3_2::implementation::CameraDeviceSession::ResultBatcher::invokeProcessCaptureResultCallback(android::hardware::hidl_vec<android::hardware::camera::device::V3_2::CaptureResult>&, bool)+316  hardware/interfaces/camera/device/3.2/default/CameraDeviceSession.cpp:748
  000195a5  android::hardware::camera::device::V3_2::implementation::CameraDeviceSession::ResultBatcher::processOneCaptureResult(android::hardware::camera::device::V3_2::CaptureResult&)+64                                                 hardware/interfaces/camera/device/3.2/default/CameraDeviceSession.cpp:763
  00019735  android::hardware::camera::device::V3_2::implementation::CameraDeviceSession::ResultBatcher::processCaptureResult(android::hardware::camera::device::V3_2::CaptureResult&)+100                                                   hardware/interfaces/camera/device/3.2/default/CameraDeviceSession.cpp:772
  00016641  android::hardware::camera::device::V3_2::implementation::CameraDeviceSession::sProcessCaptureResult(camera3_callback_ops const*, camera3_capture_result const*)+160                                                              hardware/interfaces/camera/device/3.2/default/CameraDeviceSession.cpp:1584
  000741e9  android::camera2::ResultProcessor::returnPendingBuffers(android::camera2::ResultProcessor::RequestState_t*)+484                                                                                                                  hardware/rockchip/camera/AAL/ResultProcessor.cpp:477
  00073d1f  android::camera2::ResultProcessor::handleBufferDone(android::camera2::ResultProcessor::Message&)+326                                                                                                                             hardware/rockchip/camera/AAL/ResultProcessor.cpp:420
  0007357b  android::camera2::ResultProcessor::messageThreadLoop()+198                                                                                                                                                                       hardware/rockchip/camera/AAL/ResultProcessor.cpp:159
  0008f2e1  android::camera2::MessageThread::threadLoop(void*)+8                                                                                                                                                                             hardware/rockchip/camera/common/MessageThread.h:66
  000a6867  __pthread_start(void*)+20                                                                                                                                                                                                        bionic/libc/bionic/pthread_create.cpp:338
  00060087  __start_thread+30                                                                                                                                                                                                                bionic/libc/bionic/clone.cpp:53

```



#### (2)、Camera3Device::processCaptureResult()->Stack Trace

```cpp
Stack Trace:
  RELADDR   FUNCTION                                                                                                                                                                                                                                                      FILE:LINE
  00100f15  android::Camera3Device::processCaptureResult(camera3_capture_result const*)+84                                                                                                                                                                                frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp:3625
  001006f5  android::Camera3Device::processOneCaptureResultLocked(android::hardware::camera::device::V3_2::CaptureResult const&, android::hardware::hidl_vec<android::hardware::camera::device::V3_4::PhysicalCameraMetadata>)+1064                                       frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp:1454
  001009dd  android::Camera3Device::processCaptureResult(android::hardware::hidl_vec<android::hardware::camera::device::V3_2::CaptureResult> const&)+224                                                                                                                  frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp:1278
  00100af3  virtual thunk to android::Camera3Device::processCaptureResult(android::hardware::hidl_vec<android::hardware::camera::device::V3_2::CaptureResult> const&)+10                                                                                                  aeabi_div0.c:?
  00019d15  android::hardware::camera::device::V3_2::BnHwCameraDeviceCallback::_hidl_processCaptureResult(android::hidl::base::V1_0::BnHwBase*, android::hardware::Parcel const&, android::hardware::Parcel*, std::__1::function<void (android::hardware::Parcel&)>)+212  out/soong/.intermediates/hardware/interfaces/camera/device/3.2/android.hardware.camera.device@3.2_genc++/gen/android/hardware/camera/device/3.2/CameraDeviceCallbackAll.cpp:422
  0001ef41  android::hardware::camera::device::V3_5::BnHwCameraDeviceCallback::onTransact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+288                            out/soong/.intermediates/hardware/interfaces/camera/device/3.5/android.hardware.camera.device@3.5_genc++/gen/android/hardware/camera/device/3.5/CameraDeviceCallbackAll.cpp:679
  000689a1  android::hardware::BHwBinder::transact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+52                                                                    system/libhwbinder/Binder.cpp:116
  v------>  android::hardware::IPCThreadState::executeCommand(int)                                                                                                                                                                                                        system/libhwbinder/IPCThreadState.cpp:1204
  0006af8d  android::hardware::IPCThreadState::getAndExecuteCommand()+924                                                                                                                                                                                                 system/libhwbinder/IPCThreadState.cpp:461
  0006be61  android::hardware::IPCThreadState::joinThreadPool(bool)+56                                                                                                                                                                                                    system/libhwbinder/IPCThreadState.cpp:561
  00076f0d  android::hardware::PoolThread::threadLoop()+12                                                                                                                                                                                                                system/libhwbinder/ProcessState.cpp:61
  0000d8bf  android::Thread::_threadLoop(void*)+186                                                                                                                                                                                                                       system/core/libutils/Threads.cpp:746
  000a6867  __pthread_start(void*)+20                                                                                                                                                                                                                                     bionic/libc/bionic/pthread_create.cpp:338
  00060087  __start_thread+30                                                                                                                                                                                                                                             bionic/libc/bionic/clone.cpp:53

```



#### (3)、Camera3OutputStream::returnBufferCheckedLocked()->Stack Trace

```cpp
Stack Trace:
  RELADDR   FUNCTION                                                                                                                                                                                                                                                                                                                                                                                                                                                  FILE:LINE
  0011ac87  android::camera3::Camera3OutputStream::returnBufferCheckedLocked(camera3_stream_buffer const&, long long, bool, std::__1::vector<unsigned int, std::__1::allocator<unsigned int> > const&, android::sp<android::Fence>*)+50                                                                                                                                                                                                                               frameworks/av/services/camera/libcameraservice/device3/Camera3OutputStream.cpp:219
  001184cd  android::camera3::Camera3IOStreamBase::returnAnyBufferLocked(camera3_stream_buffer const&, long long, bool, std::__1::vector<unsigned int, std::__1::allocator<unsigned int> > const&)+124                                                                                                                                                                                                                                                                frameworks/av/services/camera/libcameraservice/device3/Camera3IOStreamBase.cpp:242
  0011abfd  android::camera3::Camera3OutputStream::returnBufferLocked(camera3_stream_buffer const&, long long, std::__1::vector<unsigned int, std::__1::allocator<unsigned int> > const&)+60                                                                                                                                                                                                                                                                          frameworks/av/services/camera/libcameraservice/device3/Camera3OutputStream.cpp:196
  00117089  android::camera3::Camera3Stream::returnBuffer(camera3_stream_buffer const&, long long, bool, std::__1::vector<unsigned int, std::__1::allocator<unsigned int> > const&, unsigned long long)+240                                                                                                                                                                                                                                                           frameworks/av/services/camera/libcameraservice/device3/Camera3Stream.cpp:708
  000ffcc7  android::Camera3Device::returnOutputBuffers(camera3_stream_buffer const*, unsigned int, long long, bool, std::__1::unordered_map<int, std::__1::vector<unsigned int, std::__1::allocator<unsigned int> >, std::__1::hash<int>, std::__1::equal_to<int>, std::__1::allocator<std::__1::pair<int const, std::__1::vector<unsigned int, std::__1::allocator<unsigned int> > > > > const&, android::hardware::camera2::impl::CaptureResultExtras const&)+334  frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp:3185
  001013df  android::Camera3Device::processCaptureResult(camera3_capture_result const*)+1310                                                                                                                                                                                                                                                                                                                                                                          frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp:3772
  001006f5  android::Camera3Device::processOneCaptureResultLocked(android::hardware::camera::device::V3_2::CaptureResult const&, android::hardware::hidl_vec<android::hardware::camera::device::V3_4::PhysicalCameraMetadata>)+1064                                                                                                                                                                                                                                   frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp:1454
  001009dd  android::Camera3Device::processCaptureResult(android::hardware::hidl_vec<android::hardware::camera::device::V3_2::CaptureResult> const&)+224                                                                                                                                                                                                                                                                                                              frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp:1278
  00100af3  virtual thunk to android::Camera3Device::processCaptureResult(android::hardware::hidl_vec<android::hardware::camera::device::V3_2::CaptureResult> const&)+10                                                                                                                                                                                                                                                                                              aeabi_div0.c:?
  00019d15  android::hardware::camera::device::V3_2::BnHwCameraDeviceCallback::_hidl_processCaptureResult(android::hidl::base::V1_0::BnHwBase*, android::hardware::Parcel const&, android::hardware::Parcel*, std::__1::function<void (android::hardware::Parcel&)>)+212                                                                                                                                                                                              out/soong/.intermediates/hardware/interfaces/camera/device/3.2/android.hardware.camera.device@3.2_genc++/gen/android/hardware/camera/device/3.2/CameraDeviceCallbackAll.cpp:422
  0001ef41  android::hardware::camera::device::V3_5::BnHwCameraDeviceCallback::onTransact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+288                                                                                                                                                                                                                        out/soong/.intermediates/hardware/interfaces/camera/device/3.5/android.hardware.camera.device@3.5_genc++/gen/android/hardware/camera/device/3.5/CameraDeviceCallbackAll.cpp:679
  000689a1  android::hardware::BHwBinder::transact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+52                                                                                                                                                                                                                                                                system/libhwbinder/Binder.cpp:116
  v------>  android::hardware::IPCThreadState::executeCommand(int)                                                                                                                                                                                                                                                                                                                                                                                                    system/libhwbinder/IPCThreadState.cpp:1204
  0006af8d  android::hardware::IPCThreadState::getAndExecuteCommand()+924                                                                                                                                                                                                                                                                                                                                                                                             system/libhwbinder/IPCThreadState.cpp:461
  0006be61  android::hardware::IPCThreadState::joinThreadPool(bool)+56                                                                                                                                                                                                                                                                                                                                                                                                system/libhwbinder/IPCThreadState.cpp:561
  00076f0d  android::hardware::PoolThread::threadLoop()+12                                                                                                                                                                                                                                                                                                                                                                                                            system/libhwbinder/ProcessState.cpp:61
  0000d8bf  android::Thread::_threadLoop(void*)+186                                                                                                                                                                                                                                                                                                                                                                                                                   system/core/libutils/Threads.cpp:746
  000a6867  __pthread_start(void*)+20                                                                                                                                                                                                                                                                                                                                                                                                                                 bionic/libc/bionic/pthread_create.cpp:338
  00060087  __start_thread+30                                                                                                                                                                                                                                                                                                                                                                                                                                         bionic/libc/bionic/clone.cpp:53

```

#### (4)、queueBufferToConsumer()

```cpp
F:\Khadas_Edge_Android_Q\frameworks\av\services\camera\libcameraservice\device3\Camera3OutputStream.cpp
status_t Camera3OutputStream::queueBufferToConsumer(sp<ANativeWindow>& consumer,
            ANativeWindowBuffer* buffer, int anwReleaseFence) {
    /* 直接调用 ANativeWindow 对象的 queueBuffer 方法将 buffer 归还回去，
     * ANativeWindow 是 OpenGL 定义的图形接口，在 Android上 的实现就是
     * Surface 和 SurfaceFlinger，一个用于生产 buffer ，一个用于消费 buffer
     */
    return consumer->queueBuffer(consumer.get(), buffer, anwReleaseFence);
}
```

#### (5)、Camera3Device::insertResultLocked()->Stack Trace

```cpp
Stack Trace:
  RELADDR   FUNCTION                                                                                                                                                                                                                                                                                                                                                       FILE:LINE
  00108d31  android::Camera3Device::insertResultLocked(android::CaptureResult*, unsigned int)+344                                                                                                                                                                                                                                                                          frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp:3466
  00109699  android::Camera3Device::sendCaptureResult(android::CameraMetadata&, android::hardware::camera2::impl::CaptureResultExtras&, android::CameraMetadata&, unsigned int, bool, bool, std::__1::vector<android::hardware::camera2::impl::PhysicalCaptureResultInfo, std::__1::allocator<android::hardware::camera2::impl::PhysicalCaptureResultInfo> > const&)+1480  frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp:3614
  00101573  android::Camera3Device::processCaptureResult(camera3_capture_result const*)+1714                                                                                                                                                                                                                                                                               frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp:3790
  001006f5  android::Camera3Device::processOneCaptureResultLocked(android::hardware::camera::device::V3_2::CaptureResult const&, android::hardware::hidl_vec<android::hardware::camera::device::V3_4::PhysicalCameraMetadata>)+1064                                                                                                                                        frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp:1454
  001009dd  android::Camera3Device::processCaptureResult(android::hardware::hidl_vec<android::hardware::camera::device::V3_2::CaptureResult> const&)+224                                                                                                                                                                                                                   frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp:1278
  00100af3  virtual thunk to android::Camera3Device::processCaptureResult(android::hardware::hidl_vec<android::hardware::camera::device::V3_2::CaptureResult> const&)+10                                                                                                                                                                                                   aeabi_div0.c:?
  00019d15  android::hardware::camera::device::V3_2::BnHwCameraDeviceCallback::_hidl_processCaptureResult(android::hidl::base::V1_0::BnHwBase*, android::hardware::Parcel const&, android::hardware::Parcel*, std::__1::function<void (android::hardware::Parcel&)>)+212                                                                                                   out/soong/.intermediates/hardware/interfaces/camera/device/3.2/android.hardware.camera.device@3.2_genc++/gen/android/hardware/camera/device/3.2/CameraDeviceCallbackAll.cpp:422
  0001ef41  android::hardware::camera::device::V3_5::BnHwCameraDeviceCallback::onTransact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+288                                                                                                                             out/soong/.intermediates/hardware/interfaces/camera/device/3.5/android.hardware.camera.device@3.5_genc++/gen/android/hardware/camera/device/3.5/CameraDeviceCallbackAll.cpp:679
  000689a1  android::hardware::BHwBinder::transact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+52                                                                                                                                                                     system/libhwbinder/Binder.cpp:116
  v------>  android::hardware::IPCThreadState::executeCommand(int)                                                                                                                                                                                                                                                                                                         system/libhwbinder/IPCThreadState.cpp:1204
  0006af8d  android::hardware::IPCThreadState::getAndExecuteCommand()+924                                                                                                                                                                                                                                                                                                  system/libhwbinder/IPCThreadState.cpp:461
  0006be61  android::hardware::IPCThreadState::joinThreadPool(bool)+56                                                                                                                                                                                                                                                                                                     system/libhwbinder/IPCThreadState.cpp:561
  00076f0d  android::hardware::PoolThread::threadLoop()+12                                                                                                                                                                                                                                                                                                                 system/libhwbinder/ProcessState.cpp:61
  0000d8bf  android::Thread::_threadLoop(void*)+186                                                                                                                                                                                                                                                                                                                        system/core/libutils/Threads.cpp:746
  000a6867  __pthread_start(void*)+20                                                                                                                                                                                                                                                                                                                                      bionic/libc/bionic/pthread_create.cpp:338
  00060087  __start_thread+30                                                                                                                                                                                                                                                                                                                                              bionic/libc/bionic/clone.cpp:53
```

在 Camera3Device::insertResultLocked() 中的 mResultSignal 信号到底是发给谁呢？

通过搜索代码可以知道，在 FrameProcessorBase 帧处理线程中将会接收该信号。而帧处理线程是在 CameraDeviceClient 对象的 CameraDeviceClient::initializeImpl() 中启动的（open camera阶段调用到该函数）。

#### (6)、CameraDeviceClient::onResultAvailable()->Stack Trace

```cpp
Stack Trace:
  RELADDR   FUNCTION                                                                                                                                  FILE:LINE
  000e0559  android::CameraDeviceClient::onResultAvailable(android::CaptureResult const&)+92                                                          frameworks/av/services/camera/libcameraservice/api2/CameraDeviceClient.cpp:2026
  000bbc39  android::camera2::FrameProcessorBase::processListeners(android::CaptureResult const&, android::sp<android::CameraDeviceBase> const&)+464  frameworks/av/services/camera/libcameraservice/common/FrameProcessorBase.cpp:229
  000bba4f  android::camera2::FrameProcessorBase::processSingleFrame(android::CaptureResult&, android::sp<android::CameraDeviceBase> const&)+54       frameworks/av/services/camera/libcameraservice/common/FrameProcessorBase.cpp:180
  000bb893  android::camera2::FrameProcessorBase::processNewFrames(android::sp<android::CameraDeviceBase> const&)+222                                 frameworks/av/services/camera/libcameraservice/common/FrameProcessorBase.cpp:156
  000bb75f  android::camera2::FrameProcessorBase::threadLoop()+106                                                                                    frameworks/av/services/camera/libcameraservice/common/FrameProcessorBase.cpp:126
  0000d945  android::Thread::_threadLoop(void*)+320                                                                                                   system/core/libutils/Threads.cpp:749
  000a6867  __pthread_start(void*)+20                                                                                                                 bionic/libc/bionic/pthread_create.cpp:338
  00060087  __start_thread+30                                                                                                                         bionic/libc/bionic/clone.cpp:53

      
      F:\Khadas_Edge_Android_Q\frameworks\av\services\camera\libcameraservice\api2\CameraDeviceClient.cpp
          /** Device-related methods */
void CameraDeviceClient::onResultAvailable(const CaptureResult& result) {
    ATRACE_CALL();
    ALOGV("%s", __FUNCTION__);

	CallStack stack;
	stack.update();
	stack.log("CameraDeviceClient::onResultAvailable");
    /* 这里的 mRemoteCallback 就是在CameraService 创建 CameraDeviceClient
     * 实例时传递进来的 remoteCb，自然就是 Camera APP当中的成员了，它是
     * CameraDeviceImpl 类的内部类CameraDeviceCallbacks对象 */
    // Thread-safe. No lock necessary.
    sp<hardware::camera2::ICameraDeviceCallbacks> remoteCb = mRemoteCallback;
    if (remoteCb != NULL) {
        remoteCb->onResultReceived(result.mMetadata, result.mResultExtras,
                result.mPhysicalMetadatas);
    }

    for (size_t i = 0; i < mCompositeStreamMap.size(); i++) {
        mCompositeStreamMap.valueAt(i)->onResultAvailable(result);
    }
}

```

#### (7)、Stack::CameraDeviceCallbacks.onResultReceived()

```java
07-08 07:21:08.192  4654  4671 I CameraDevice-JV-0: onResultReceived
07-08 07:21:08.192  4654  4671 I CameraDevice-JV-0: java.lang.RuntimeException: here
07-08 07:21:08.192  4654  4671 I CameraDevice-JV-0: 	at android.hardware.camera2.impl.CameraDeviceImpl$CameraDeviceCallbacks.onResultReceived(CameraDeviceImpl.java:2135)
07-08 07:21:08.192  4654  4671 I CameraDevice-JV-0: 	at android.hardware.camera2.ICameraDeviceCallbacks$Stub.onTransact(ICameraDeviceCallbacks.java:180)
07-08 07:21:08.192  4654  4671 I CameraDevice-JV-0: 	at android.os.Binder.execTransactInternal(Binder.java:1021)
07-08 07:21:08.192  4654  4671 I CameraDevice-JV-0: 	at android.os.Binder.execTransact(Binder.java:994)
```

