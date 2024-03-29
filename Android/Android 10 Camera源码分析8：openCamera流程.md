---
title: Android 10 Camera源码分析8：openCamera流程
cover: https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/post.cover.pictures/bing-wallpaper-2018.04.38.jpg
categories: 
 - Camera
tags:
 - Camera
toc: true
abbrlink: 20220415
date: 2022-04-15 04:15:00
---
注：文章都是通过阅读各位前辈总结的资料 Android 10.0 && Linux（Kernel 4.19）Rockchip平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除（◕‿◕），转载请注明出处（©Rockchip ©Android @Linux 版权所有），谢谢。

（==**文章基于 Kernel-4.19**==）&&（==**文章基于 Android 10.0**==）

[【zhoujinjian.com博客原图链接】](https://github.com/zhoujinjianzz) 

[【开发板 Khadas Edge V】](https://www.khadas.cn/edge-v)

[【开发板 Khadas Edge V Android 10.0 && Linux（Kernel 4.19）源码链接】](https://github.com/khadas/linux/tree/khadas-edge-Qt)



正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢！！！

 [Android openCamera流程](https://blog.csdn.net/weixin_41944449/article/details/99684653) 


--------------------------------------------------------------------------------

==源码（部分）==：

**F:\Khadas_Edge_Android_Q\frameworks\av\services\camera\libcameraservice\**

==源码（部分）==：

--------------------------------------------------------------------------------


## （一）、从APP到CameraService

#### （1）、总体概览图

![](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android.10.Camera.08/openCamera.png)

```java
07-07 10:04:17.368  1450  1540 I CameraManager: openCameraDeviceUserAsync
07-07 10:04:17.368  1450  1540 I CameraManager: java.lang.RuntimeException: here
07-07 10:04:17.368  1450  1540 I CameraManager: 	at android.hardware.camera2.CameraManager.openCameraDeviceUserAsync(CameraManager.java:366)
07-07 10:04:17.368  1450  1540 I CameraManager: 	at android.hardware.camera2.CameraManager.openCameraForUid(CameraManager.java:590)
07-07 10:04:17.368  1450  1540 I CameraManager: 	at android.hardware.camera2.CameraManager.openCamera(CameraManager.java:518)
07-07 10:04:17.368  1450  1540 I CameraManager: 	at com.android.camera.one.v2.Camera2OneCameraOpenerImpl.open(Camera2OneCameraOpenerImpl.java:118)
07-07 10:04:17.368  1450  1540 I CameraManager: 	at com.android.camera.CaptureModule.openCameraAndStartPreview(CaptureModule.java:1492)
07-07 10:04:17.368  1450  1540 I CameraManager: 	at com.android.camera.CaptureModule.access$2100(CaptureModule.java:114)
07-07 10:04:17.368  1450  1540 I CameraManager: 	at com.android.camera.CaptureModule$10.run(CaptureModule.java:680)
07-07 10:04:17.368  1450  1540 I CameraManager: 	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
07-07 10:04:17.368  1450  1540 I CameraManager: 	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
07-07 10:04:17.368  1450  1540 I CameraManager: 	at java.lang.Thread.run(Thread.java:919)
```



#### （2）、App->Camera2OneCameraOpenerImpl.open()

```java
F:\Khadas_Edge_Android_Q\packages\apps\Camera2\src\com\android\camera\one\v2\Camera2OneCameraOpenerImpl.java
    
    @Override
    public void open(......) {
        try {
            Log.i(TAG, "Opening Camera: " + cameraKey);
            mCameraManager.openCamera(cameraKey.getValue(), new CameraDevice.StateCallback() {
                ......
                @Override
                public void onOpened(CameraDevice device) {
                    if (isFirstCallback) {
                        ......
                        try {
                            ......
                            OneCamera oneCamera = OneCameraCreator.create()
                                    ......
                            if (oneCamera != null) {
                                openCallback.onCameraOpened(oneCamera);
                            } else {
                                Log.d(TAG, "Could not construct a OneCamera object!");
                                openCallback.onFailure();
                            }
                        } ......
                    }
                }
            }, handler);
        } ......
    }
}

```



#### （3）、Framework->CameraManager.openCamera()

```java
F:\Khadas_Edge_Android_Q\frameworks\base\core\java\android\hardware\camera2\CameraManager.java
        public void openCamera(@NonNull String cameraId,
            @NonNull final CameraDevice.StateCallback callback, @Nullable Handler handler)
            throws CameraAccessException {

        openCameraForUid(cameraId, callback, CameraDeviceImpl.checkAndWrapHandler(handler),
                USE_CALLING_UID);
    }
openCameraForUid->openCameraDeviceUserAsync
    
        private CameraDevice openCameraDeviceUserAsync(String cameraId,
            CameraDevice.StateCallback callback, Executor executor, final int uid)
            throws CameraAccessException {
        CameraCharacteristics characteristics = getCameraCharacteristics(cameraId);
        CameraDevice device = null;

        synchronized (mLock) {

            ICameraDeviceUser cameraUser = null;

            android.hardware.camera2.impl.CameraDeviceImpl deviceImpl =
                    new android.hardware.camera2.impl.CameraDeviceImpl(
                        cameraId,
                        callback,
                        executor,
                        characteristics,
                        mContext.getApplicationInfo().targetSdkVersion);

            ICameraDeviceCallbacks callbacks = deviceImpl.getCallbacks();

            try {
                if (supportsCamera2ApiLocked(cameraId)) {
                    // Use cameraservice's cameradeviceclient implementation for HAL3.2+ devices
                    //获得"media.camera" 的 CameraService Binder代理，然后转变为ICameraService
                    ICameraService cameraService = CameraManagerGlobal.get().getCameraService();
                    ......
                    cameraUser = cameraService.connectDevice(callbacks, cameraId,
                            mContext.getOpPackageName(), uid);
                } else {
                    ......
                }
            } ......
            deviceImpl.setRemoteDevice(cameraUser);
            device = deviceImpl;
        }

        return device;
    }

```



#### （4）、openSession()->StackTrace

首先看看CameraProviderManager::openSession()

```cpp
Stack Trace:
  RELADDR   FUNCTION                                                                                                                                                                                                                                                                                                                                                                               FILE:LINE
  000afd4b  android::CameraProviderManager::openSession(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, android::sp<android::hardware::camera::device::V3_2::ICameraDeviceCallback> const&, android::sp<android::hardware::camera::device::V3_2::ICameraDeviceSession>*)+178                                                                         frameworks/av/services/camera/libcameraservice/common/CameraProviderManager.cpp:349
  000fb0e7  android::Camera3Device::initialize(android::sp<android::CameraProviderManager>, android::String8 const&)+354                                                                                                                                                                                                                                                                           frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp:127
  000ad62f  int android::Camera2ClientBase<android::CameraDeviceClientBase>::initializeImpl<android::sp<android::CameraProviderManager> >(android::sp<android::CameraProviderManager>, android::String8 const&)+170                                                                                                                                                                                frameworks/av/services/camera/libcameraservice/common/Camera2ClientBase.cpp:106
  000ad557  android::Camera2ClientBase<android::CameraDeviceClientBase>::initialize(android::sp<android::CameraProviderManager>, android::String8 const&)+46                                                                                                                                                                                                                                       frameworks/av/services/camera/libcameraservice/common/Camera2ClientBase.cpp:82
  000d8847  int android::CameraDeviceClient::initializeImpl<android::sp<android::CameraProviderManager> >(android::sp<android::CameraProviderManager>, android::String8 const&)+90                                                                                                                                                                                                                 frameworks/av/services/camera/libcameraservice/api2/CameraDeviceClient.cpp:107
  000d87bf  android::CameraDeviceClient::initialize(android::sp<android::CameraProviderManager>, android::String8 const&)+46                                                                                                                                                                                                                                                                       frameworks/av/services/camera/libcameraservice/api2/CameraDeviceClient.cpp:99
  0009f4d3  android::binder::Status android::CameraService::connectHelper<android::hardware::camera2::ICameraDeviceCallbacks, android::CameraDeviceClient>(android::sp<android::hardware::camera2::ICameraDeviceCallbacks> const&, android::String8 const&, int, int, android::String16 const&, int, int, android::CameraService::apiLevel, bool, android::sp<android::CameraDeviceClient>&)+1762  frameworks/av/services/camera/libcameraservice/CameraService.cpp:1504
  0009ecc9  android::CameraService::connectDevice(android::sp<android::hardware::camera2::ICameraDeviceCallbacks> const&, android::String16 const&, android::String16 const&, int, android::sp<android::hardware::camera2::ICameraDeviceUser>*)+208                                                                                                                                                frameworks/av/services/camera/libcameraservice/CameraService.cpp:1382
  0009f88f  virtual thunk to android::CameraService::connectDevice(android::sp<android::hardware::camera2::ICameraDeviceCallbacks> const&, android::String16 const&, android::String16 const&, int, android::sp<android::hardware::camera2::ICameraDeviceUser>*)+30                                                                                                                                aeabi_div0.c:?
  00023819  android::hardware::BnCameraService::onTransact(unsigned int, android::Parcel const&, android::Parcel*, unsigned int)+556                                                                                                                                                                                                                                                               out/soong/.intermediates/frameworks/av/camera/libcamera_client/android_arm_armv7-a-neon_cortex-a15_core_shared/gen/aidl/frameworks/av/camera/aidl/android/hardware/ICameraService.cpp:829
  00032ed1  android::BBinder::transact(unsigned int, android::Parcel const&, android::Parcel*, unsigned int)+72                                                                                                                                                                                                                                                                                    frameworks/native/libs/binder/Binder.cpp:134
  0003b117  android::IPCThreadState::executeCommand(int)+770                                                                                                                                                                                                                                                                                                                                       frameworks/native/libs/binder/IPCThreadState.cpp:1213
  0003ad53  android::IPCThreadState::getAndExecuteCommand()+98                                                                                                                                                                                                                                                                                                                                     frameworks/native/libs/binder/IPCThreadState.cpp:514
  0003b2ef  android::IPCThreadState::joinThreadPool(bool)+38                                                                                                                                                                                                                                                                                                                                       frameworks/native/libs/binder/IPCThreadState.cpp:594
  00054665  android::PoolThread::threadLoop()+12                                                                                                                                                                                                                                                                                                                                                   frameworks/native/libs/binder/ProcessState.cpp:67
  0000d8bf  android::Thread::_threadLoop(void*)+186                                                                                                                                                                                                                                                                                                                                                system/core/libutils/Threads.cpp:746
  000a6867  __pthread_start(void*)+20                                                                                                                                                                                                                                                                                                                                                              bionic/libc/bionic/pthread_create.cpp:338
  00060087  __start_thread+30                                                                                                               
```



#### （5）、ICameraService::connectDevice()

BpCameraService -> BnCameraService，进入到CameraService进程中。

```cpp
/home/zhoujinjian/Khadas_Edge_Android_Q/out/soong/.intermediates/frameworks/av/camera/libcamera_client/android_arm_armv7-a-neon_cortex-a15_core_shared/gen/aidl/frameworks/av/camera/aidl/android/hardware/ICameraService.cpp
::android::binder::Status BpCameraService::connectDevice(const ::android::sp<::android::hardware::camera2::ICameraDeviceCallbacks>& callbacks, const ::android::String16& cameraId, const ::android::String16& opPackageName, int32_t clientUid, ::android::sp<::android::hardware::camera2::ICameraDeviceUser>* _aidl_return) {
  ......
  _aidl_ret_status = remote()->transact(::android::IBinder::FIRST_CALL_TRANSACTION + 3 /* connectDevice */, _aidl_data, &_aidl_reply);
  if (UNLIKELY(_aidl_ret_status == ::android::UNKNOWN_TRANSACTION && ICameraService::getDefaultImpl())) {
     return ICameraService::getDefaultImpl()->connectDevice(callbacks, cameraId, opPackageName, clientUid, _aidl_return);
  }
  ......
  return _aidl_status;
}

    ::android::status_t BnCameraService::onTransact(uint32_t _aidl_code, const ::android::Parcel& _aidl_data, ::android::Parcel* _aidl_reply, uint32_t _aidl_flags) {
  ::android::status_t _aidl_ret_status = ::android::OK;
  switch (_aidl_code) {
  ......
  case ::android::IBinder::FIRST_CALL_TRANSACTION + 3 /* connectDevice */:
  {
    ::android::sp<::android::hardware::camera2::ICameraDeviceCallbacks> in_callbacks;
    ::android::String16 in_cameraId;
    ::android::String16 in_opPackageName;
    int32_t in_clientUid;
    ::android::sp<::android::hardware::camera2::ICameraDeviceUser> _aidl_return;
    _aidl_ret_status = _aidl_data.readStrongBinder(&in_callbacks);
    _aidl_ret_status = _aidl_data.readString16(&in_cameraId);
    _aidl_ret_status = _aidl_data.readString16(&in_opPackageName);
    _aidl_ret_status = _aidl_data.readInt32(&in_clientUid);
    ::android::binder::Status _aidl_status(connectDevice(in_callbacks, in_cameraId, in_opPackageName, in_clientUid, &_aidl_return));
    _aidl_ret_status = _aidl_status.writeToParcel(_aidl_reply);
    ......
    _aidl_ret_status = _aidl_reply->writeStrongBinder(::android::hardware::camera2::ICameraDeviceUser::asBinder(_aidl_return));
    ......
  }
  break;
  ......
}

```



#### （6）、CameraService::connectDevice()

``` cpp
F:\Khadas_Edge_Android_Q\frameworks\av\services\camera\libcameraservice\CameraService.cpp
    
    Status CameraService::connectDevice(
        const sp<hardware::camera2::ICameraDeviceCallbacks>& cameraCb,
        const String16& cameraId,
        const String16& clientPackageName,
        int clientUid,
        /*out*/
        sp<hardware::camera2::ICameraDeviceUser>* device) {

    ATRACE_CALL();
    Status ret = Status::ok();
    String8 id = String8(cameraId);
    sp<CameraDeviceClient> client = nullptr;
    String16 clientPackageNameAdj = clientPackageName;
    ......
    ret = connectHelper<hardware::camera2::ICameraDeviceCallbacks,CameraDeviceClient>(cameraCb, id,
            /*api1CameraId*/-1,
            CAMERA_HAL_API_VERSION_UNSPECIFIED, clientPackageNameAdj,
            clientUid, USE_CALLING_PID, API_2, /*shimUpdateOnly*/ false, /*out*/client);
    ......
    *device = client;
    return ret;
}


template<class CALLBACK, class CLIENT>
Status CameraService::connectHelper(const sp<CALLBACK>& cameraCb, const String8& cameraId,
        int api1CameraId, int halVersion, const String16& clientPackageName, int clientUid,
        int clientPid, apiLevel effectiveApiLevel, bool shimUpdateOnly,
        /*out*/sp<CLIENT>& device) {
    binder::Status ret = binder::Status::ok();

    String8 clientName8(clientPackageName);
    property_set("vendor.sys.camera.callprocess", clientName8.string());

    int originalClientPid = 0;

    ALOGI("CameraService::connect call (PID %d \"%s\", camera ID %s) for HAL version %s and "
            "Camera API version %d", clientPid, clientName8.string(), cameraId.string(),
            (halVersion == -1) ? "default" : std::to_string(halVersion).c_str(),
            static_cast<int>(effectiveApiLevel));

    if (shouldRejectHiddenCameraConnection(cameraId)) {
        ALOGW("Attempting to connect to system-only camera id %s, connection rejected",
              cameraId.c_str());
        return STATUS_ERROR_FMT(ERROR_DISCONNECTED,
                                "No camera device with ID \"%s\" currently available",
                                cameraId.string());

    }
    sp<CLIENT> client = nullptr;
    {
        ......
        // give flashlight a chance to close devices if necessary.
        mFlashlight->prepareDeviceOpen(cameraId);

        int facing = -1;
        int deviceVersion = getDeviceVersion(cameraId, /*out*/&facing);
        ......
        sp<BasicClient> tmp = nullptr;
         /* makeClient() 将会根据API版本以及HAL版本选择生成具体的 Client 实例 */
        if(!(ret = makeClient(this, cameraCb, clientPackageName,
                cameraId, api1CameraId, facing,
                clientPid, clientUid, getpid(),
                halVersion, deviceVersion, effectiveApiLevel,
                /*out*/&tmp)).isOk()) {
            return ret;
        }
        client = static_cast<CLIENT*>(tmp.get());
        ......
         /* client 进行初始化工作,mCameraProviderManager 是 CameraService 运行起来时，
         * 第一次强指针引用从而调用 onFirstRef() 函数创建的 CameraProviderManager
         * 实例对象，它保存了 CameraProvider 信息.
         */
        err = client->initialize(mCameraProviderManager, mMonitorTags);
        .......
    } // lock is destroyed, allow further connect calls

    // Important: release the mutex here so the client can call back into the service from its
    // destructor (can be at the end of the call)
    device = client;
    return ret;
}

```
#### （7）、CameraService::makeClient()

```cpp
F:\Khadas_Edge_Android_Q\frameworks\av\services\camera\libcameraservice\CameraService.cpp

Status CameraService::makeClient(const sp<CameraService>& cameraService,
        const sp<IInterface>& cameraCb, const String16& packageName, const String8& cameraId,
        int api1CameraId, int facing, int clientPid, uid_t clientUid, int servicePid,
        int halVersion, int deviceVersion, apiLevel effectiveApiLevel,
        /*out*/sp<BasicClient>* client) {
    if (halVersion < 0 || halVersion == deviceVersion) {
        switch(deviceVersion) {
          ......
          case CAMERA_DEVICE_API_VERSION_3_0:
          case CAMERA_DEVICE_API_VERSION_3_1:
          case CAMERA_DEVICE_API_VERSION_3_2:
          case CAMERA_DEVICE_API_VERSION_3_3:
          case CAMERA_DEVICE_API_VERSION_3_4:
          case CAMERA_DEVICE_API_VERSION_3_5:
            if (effectiveApiLevel == API_1) { // Camera1 API route
                ......
            } else { // Camera2 API route
                /* API2 HAL3 将会通过 CameraDeviceClient() 创建 client，
                 * 这一 client 最终返回到 CameraDeviceImpl 实例，被保存在 mRemoteDevice
                 */
                sp<hardware::camera2::ICameraDeviceCallbacks> tmp =
                        static_cast<hardware::camera2::ICameraDeviceCallbacks*>(cameraCb.get());
                *client = new CameraDeviceClient(cameraService, tmp, packageName, cameraId,
                        facing, clientPid, clientUid, servicePid);
            }
            break;
          ......
        }
    } else {
        ......
    }
    return Status::ok();
}
```

![](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android.10.Camera.08/app_camera_service.png)



## （二）、从CameraService到CameraProvider

由于 Android O 中加入了 Treble 机制，它带来的一个巨大变化就是将原本的 CameraServer 进程分隔成 CameraServer 与 Provider service 两个进程，它们之间通过 HIDL（一个类似 Binder 的机制）进行通信。
在这种情况下，CameraServer 一端主体为 CameraService，它将会寻找现存的 Provider service，将其加入到内部的 CameraProviderManager 中进行管理，相关操作都是通过远端调用进行的。
而 Provider service 一端的主体为 CameraProvider，它在初始化时就已经连接到 libhardware 的 Camera HAL 实现层，并以 CameraModule 来进行管理。这两个进程的启动以及初始化在系统启动时就已经进行，所以当相机应用运行时，它们的初始化工作已经完成。

#### （1）、CameraDeviceClient::CameraDeviceClient()

```cpp
F:\Khadas_Edge_Android_Q\frameworks\av\services\camera\libcameraservice\api2\CameraDeviceClient.cpp
CameraDeviceClient::CameraDeviceClient(const sp<CameraService>& cameraService,
        const sp<hardware::camera2::ICameraDeviceCallbacks>& remoteCallback,
        const String16& clientPackageName,
        const String8& cameraId,
        int cameraFacing,
        int clientPid,
        uid_t clientUid,
        int servicePid) :
    Camera2ClientBase(cameraService, remoteCallback, clientPackageName,
                cameraId, /*API1 camera ID*/ -1,
                cameraFacing, clientPid, clientUid, servicePid),
    mInputStream(),
    mStreamingRequestId(REQUEST_ID_NONE),
    mRequestIdCounter(0) {

    ATRACE_CALL();
    ALOGI("CameraDeviceClient %s: Opened", cameraId.string());
}

F:\Khadas_Edge_Android_Q\frameworks\av\services\camera\libcameraservice\common\Camera2ClientBase.cpp
template <typename TClientBase>
Camera2ClientBase<TClientBase>::Camera2ClientBase(
        const sp<CameraService>& cameraService,
        const sp<TCamCallbacks>& remoteCallback,
        const String16& clientPackageName,
        const String8& cameraId,
        int api1CameraId,
        int cameraFacing,
        int clientPid,
        uid_t clientUid,
        int servicePid):
        TClientBase(cameraService, remoteCallback, clientPackageName,
                cameraId, api1CameraId, cameraFacing, clientPid, clientUid, servicePid),
        mSharedCameraCallbacks(remoteCallback),
        mDeviceVersion(cameraService->getDeviceVersion(TClientBase::mCameraIdStr)),
        mDevice(new Camera3Device(cameraId)),
        mDeviceActive(false), mApi1CameraId(api1CameraId)
{
    ALOGI("Camera %s: Opened. Client: %s (PID %d, UID %d)", cameraId.string(),
            String8(clientPackageName).string(), clientPid, clientUid);

    mInitialClientPid = clientPid;
    LOG_ALWAYS_FATAL_IF(mDevice == 0, "Device should never be NULL here.");
}
```


#### （2）、继续看CameraService::connectHelper函数中client->initialize()

```cpp
F:\Khadas_Edge_Android_Q\frameworks\av\services\camera\libcameraservice\api2\CameraDeviceClient.cpp
    status_t CameraDeviceClient::initialize(sp<CameraProviderManager> manager,
        const String8& monitorTags) {
    return initializeImpl(manager, monitorTags);
}

template<typename TProviderPtr>
status_t CameraDeviceClient::initializeImpl(TProviderPtr providerPtr, const String8& monitorTags) {
    res = Camera2ClientBase::initialize(providerPtr, monitorTags);
    
    String8 threadName;
    mFrameProcessor = new FrameProcessorBase(mDevice);
    threadName = String8::format("CDU-%s-FrameProc", mCameraIdStr.string());
    mFrameProcessor->run(threadName.string());

    mFrameProcessor->registerListener(FRAME_PROCESSOR_LISTENER_MIN_ID,
                                      FRAME_PROCESSOR_LISTENER_MAX_ID,
                                      /*listener*/this,
                                      /*sendPartials*/true);

    auto deviceInfo = mDevice->info();
    camera_metadata_entry_t physicalKeysEntry = deviceInfo.find(
            ANDROID_REQUEST_AVAILABLE_PHYSICAL_CAMERA_REQUEST_KEYS);
   ......

    mProviderManager = providerPtr;
    return OK;
}

F:\Khadas_Edge_Android_Q\frameworks\av\services\camera\libcameraservice\common\Camera2ClientBase.cpp
    template <typename TClientBase>
status_t Camera2ClientBase<TClientBase>::initialize(sp<CameraProviderManager> manager,
        const String8& monitorTags) {
    return initializeImpl(manager, monitorTags);
}

template <typename TClientBase>
template <typename TProviderPtr>
status_t Camera2ClientBase<TClientBase>::initializeImpl(TProviderPtr providerPtr,
        const String8& monitorTags) {
    ATRACE_CALL();
    ALOGV("%s: Initializing client for camera %s", __FUNCTION__,
          TClientBase::mCameraIdStr.string());
    res = TClientBase::startCameraOps();
    ......
    res = mDevice->initialize(providerPtr, monitorTags);
    ......
    wp<CameraDeviceBase::NotificationListener> weakThis(this);
    res = mDevice->setNotifyCallback(weakThis);

    return OK;
}
```



#### （3）、Camera3Device::initialize

``` cpp
status_t Camera3Device::initialize(sp<CameraProviderManager> manager, const String8& monitorTags) {
    ATRACE_CALL();
    Mutex::Autolock il(mInterfaceLock);
    Mutex::Autolock l(mLock);

    ALOGV("%s: Initializing HIDL device for camera %s", __FUNCTION__, mId.string());
    ......
    sp<ICameraDeviceSession> session;
    ......
    status_t res = manager->openSession(mId.string(), this,
            /*out*/ &session);
    ......
    mInterface = new HalInterface(session, queue, mUseHalBufManager);
    

    ......

    return initializeCommonLocked();
}
```

#### （4）、CameraProviderManager::openSession()

```cpp
F:\Khadas_Edge_Android_Q\frameworks\av\services\camera\libcameraservice\common\CameraProviderManager.cpp
status_t CameraProviderManager::openSession(const std::string &id,
        const sp<device::V3_2::ICameraDeviceCallback>& callback,
        /*out*/
        sp<device::V3_2::ICameraDeviceSession> *session) {
	ALOGI("V3_2 openSession");
    std::lock_guard<std::mutex> lock(mInterfaceMutex);

    auto deviceInfo = findDeviceInfoLocked(id,
            /*minVersion*/ {3,0}, /*maxVersion*/ {4,0});
    if (deviceInfo == nullptr) return NAME_NOT_FOUND;

    auto *deviceInfo3 = static_cast<ProviderInfo::DeviceInfo3*>(deviceInfo);
    const sp<provider::V2_4::ICameraProvider> provider =
            deviceInfo->mParentProvider->startProviderInterface();
    if (provider == nullptr) {
        return DEAD_OBJECT;
    }
    saveRef(DeviceMode::CAMERA, id, provider);

    Status status;
    hardware::Return<void> ret;
    auto interface = deviceInfo3->startDeviceInterface<
            CameraProviderManager::ProviderInfo::DeviceInfo3::InterfaceT>();
    if (interface == nullptr) {
        return DEAD_OBJECT;
    }

	CallStack stack;
	stack.update();
	stack.log("CameraProviderManager::openSession");

    ret = interface->open(callback, [&status, &session]
            (Status s, const sp<device::V3_2::ICameraDeviceSession>& cameraSession) {
                status = s;
                if (status == Status::OK) {
                    *session = cameraSession;
                }
            });
    if (!ret.isOk()) {
        removeRef(DeviceMode::CAMERA, id);
        ALOGE("%s: Transaction error opening a session for camera device %s: %s",
                __FUNCTION__, id.c_str(), ret.description().c_str());
        return DEAD_OBJECT;
    }

    return mapToStatusT(status);
}

```

至此，Android open camera的流程操作到相应SOC厂商（Rockchip）的camera HAL层

#### （5）、CameraDevice::open()

```cpp
F:\Khadas_Edge_Android_Q\hardware\interfaces\camera\device\3.2\default\CameraDevice.cpp
Return<void> CameraDevice::open(const sp<ICameraDeviceCallback>& callback,
        ICameraDevice::open_cb _hidl_cb)  {
    Status status = initStatus();
    sp<CameraDeviceSession> session = nullptr;
    ......
    if (status != Status::OK) {
       .......
    } else {
        mLock.lock();

        ALOGV("%s: Initializing device for camera %d", __FUNCTION__, mCameraIdInt);
        session = mSession.promote();
        ......
        /** Open HAL device */
        status_t res;
        camera3_device_t *device;

        ATRACE_BEGIN("camera3->open");
        res = mModule->open(mCameraId.c_str(),
                reinterpret_cast<hw_device_t**>(&device));
        ATRACE_END();
        ......
        struct camera_info info;
        res = mModule->getCameraInfo(mCameraIdInt, &info);
        ......
        session = createSession(
                device, info.static_camera_characteristics, callback);
        ......
        mSession = session;

        IF_ALOGV() {
            session->getInterface()->interfaceChain([](
                ::android::hardware::hidl_vec<::android::hardware::hidl_string> interfaceChain) {
                    ALOGV("Session interface chain:");
                    for (const auto& iface : interfaceChain) {
                        ALOGV("  %s", iface.c_str());
                    }
                });
        }
        mLock.unlock();
    }
    _hidl_cb(status, session->getInterface());
    return Void();
}

```

#### （6）、Camera3HALModule::openCameraHardware()


```cpp
F:\Khadas_Edge_Android_Q\hardware\rockchip\camera\Camera3HALModule.cpp
int openCameraHardware(int id, const hw_module_t* module, hw_device_t** device)
{
    HAL_TRACE_CALL(CAM_GLBL_DBG_HIGH);

    if (sInstances[id])
        return 0;

    FlashLight& flash = FlashLight::getInstance();

    flash.init(id);
    flash.setCallbacks(sCallbacks);
    flash.reserveFlashForCamera(id);

    Camera3HAL* halDev = new Camera3HAL(id, module);

    if (halDev->init() != NO_ERROR) {
        LOGE("HAL initialization fail!");
        delete halDev;
        return -EINVAL;
    }
    camera3_device_t *cam3Device = halDev->getDeviceStruct();

    cam3Device->common.close = hal_dev_close;
    *device = &cam3Device->common;

    sInstanceCount++;
    sInstances[id] = true;

	CallStack stack;
	stack.update();
	stack.log("Camera3HALModule::open");

    return 0;
}
F:\Khadas_Edge_Android_Q\hardware\rockchip\camera\AAL\Camera3HAL.cpp

static camera3_device_ops hal_dev_ops = {
    NAMED_FIELD_INITIALIZER(initialize)                         hal_dev_initialize,
    NAMED_FIELD_INITIALIZER(configure_streams)                  hal_dev_configure_streams,
    NAMED_FIELD_INITIALIZER(register_stream_buffers)            nullptr,
    NAMED_FIELD_INITIALIZER(construct_default_request_settings) hal_dev_construct_default_request_settings,
    NAMED_FIELD_INITIALIZER(process_capture_request)            hal_dev_process_capture_request,
    NAMED_FIELD_INITIALIZER(get_metadata_vendor_tag_ops)        nullptr,
    NAMED_FIELD_INITIALIZER(dump)                               hal_dev_dump,
    NAMED_FIELD_INITIALIZER(flush)                              hal_dev_flush,
    NAMED_FIELD_INITIALIZER(reserved)                           {0},
};

Camera3HAL::Camera3HAL(int cameraId, const hw_module_t* module) :
    mCameraId(cameraId),
    mCameraHw(nullptr),
    mRequestThread(nullptr)
{
    LOGI("@%s", __FUNCTION__);

    struct camera_info info;
    PlatformData::getCameraInfo(cameraId, &info);

    CLEAR(mDevice);
    mDevice.common.tag = HARDWARE_DEVICE_TAG;
    mDevice.common.version = info.device_version;
    mDevice.common.module = (hw_module_t *)(module);
    // hal_dev_close is kept in the module for symmetry with dev_open
    // it will be set there
    mDevice.common.close = nullptr;
    //重要：初始化ops
    mDevice.ops = &hal_dev_ops;
    mDevice.priv = this;
}
status_t Camera3HAL::init(void)
{
    HAL_TRACE_CALL(CAM_GLBL_DBG_HIGH);
    status_t status = UNKNOWN_ERROR;

    mCameraHw = ICameraHw::createCameraHW(mCameraId);

    status = mCameraHw->init();
    if (status != NO_ERROR) {
        LOGE("Error initializing Camera HW");
        goto bail;
    }

    mRequestThread = std::unique_ptr<RequestThread>(new RequestThread(mCameraId, mCameraHw));

    return NO_ERROR;

bail:
    deinit();
    return status;
}

```



#### （7）、openCameraHardware()->Stack Trace

```cpp
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
  00060087  __start_thread+30            
```

![](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android.10.Camera.08/cameraservice_provider.png)
