---
title: Android Camera System（2）：Camera 系统 startPreview、takePicture、Recorder流程分析
cover: https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/hexo.themes/bing-wallpaper-2018.04.19.jpg
categories:
  - Camera
tags:
  - Android
  - Linux
  - Camera
toc: true
abbrlink: 20190109
date: 2019-01-09 09:25:00
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
#### （一）、Camera System startPreview流程分析
##### 1.1、Camera2 startPreview的应用层(Java)流程分析
preview流程都是从startPreview开始的，所以来看startPreview方法的代码：
``` java
[->/packages/apps/Camera2/src/com/android/camera/one/v2/OneCameraImpl.java]
Override
public void startPreview(Surface previewSurface, CaptureReadyCallback listener) {
    mPreviewSurface = previewSurface;
    //根据Surface以及CaptureReadyCallback回调来建立preview环境
    setupAsync(mPreviewSurface, listener);
}
```
这其中有一个比较重要的回调CaptureReadyCallback，先分析setupAsync方法：
``` java
[->/packages/apps/Camera2/src/com/android/camera/one/v2/OneCameraImpl.java]
private void setupAsync(final Surface previewSurface, final CaptureReadyCallback listener) {
    mCameraHandler.post(new Runnable() {
        @Override
        public void run() {
            //建立preview环境
            setup(previewSurface, listener);
        }
    });
}
```
这里通过CameraHandler来post一个Runnable对象，它只会调用Runnable的run方法，它仍然属于UI线程，并没有创建新的线程。所以，继续分析setup方法：

``` java
[->/packages/apps/Camera2/src/com/android/camera/one/v2/OneCameraImpl.java]
private void setup(Surface previewSurface, final CaptureReadyCallback listener) {
    try {
        if (mCaptureSession != null) {
            mCaptureSession.abortCaptures();
            mCaptureSession = null;
        }
        List<Surface> outputSurfaces = new ArrayList<Surface>(2);
        outputSurfaces.add(previewSurface);
        outputSurfaces.add(mCaptureImageReader.getSurface());
        //创建CaptureSession会话来与Camera Device发送Preview请求
        mDevice.createCaptureSession(outputSurfaces, new CameraCaptureSession.StateCallback() {

            @Override
            public void onConfigureFailed(CameraCaptureSession session) {
                //如果配置失败，则回调CaptureReadyCallback的onSetupFailed方法
                listener.onSetupFailed();
            }

            @Override
            public void onConfigured(CameraCaptureSession session) {
                mCaptureSession = session;
                mAFRegions = ZERO_WEIGHT_3A_REGION;
                mAERegions = ZERO_WEIGHT_3A_REGION;
                mZoomValue = 1f;
                mCropRegion = cropRegionForZoom(mZoomValue);
                //调用repeatingPreview来启动preview
                boolean success = repeatingPreview(null);
                if (success) {
                    //若启动成功，则回调CaptureReadyCallback的onReadyForCapture，表示准备拍照成功
                    listener.onReadyForCapture();
                } else {
                    //若启动失败，则回调CaptureReadyCallback的onSetupFailed，表示preview建立失败
                    listener.onSetupFailed();
                }
            }

            @Override
            public void onClosed(CameraCaptureSession session) {
                super.onClosed(session);
            }
        }, mCameraHandler);
    } catch (CameraAccessException ex) {
        Log.e(TAG, "Could not set up capture session", ex);
        listener.onSetupFailed();
    }
}
```
首先，调用Device的createCaptureSession方法来创建一个会话，并定义了会话的状态回调CameraCaptureSession.StateCallback()，其中，当会话创建成功，则会回调onConfigured()方法,在其中，首先调用repeatingPreview来启动preview，然后处理preview的结果并调用先前定义的CaptureReadyCallback来通知用户进行Capture操作。先分析repeatingPreview方法：

``` java
[->/packages/apps/Camera2/src/com/android/camera/one/v2/OneCameraImpl.java]
private boolean repeatingPreview(Object tag) {
    try {
        //通过CameraDevice对象创建一个CaptureRequest的preview请求
        CaptureRequest.Builder builder = mDevice.createCaptureRequest(
                CameraDevice.TEMPLATE_PREVIEW);
        //添加预览的目标Surface
        builder.addTarget(mPreviewSurface);
        //设置预览模式
        builder.set(CaptureRequest.CONTROL_MODE, CameraMetadata.CONTROL_MODE_AUTO);
        addBaselineCaptureKeysToRequest(builder);
        //利用会话发送请求，mCaptureCallback为
        mCaptureSession.setRepeatingRequest(builder.build(), mCaptureCallback,mCameraHandler);
        Log.v(TAG, String.format("Sent repeating Preview request, zoom = %.2f", mZoomValue));
        return true;
    } catch (CameraAccessException ex) {
        Log.e(TAG, "Could not access camera setting up preview.", ex);
        return false;
    }
}
```
首先调用CameraDeviceImpl的createCaptureRequest方法创建类型为TEMPLATE_PREVIEW 的CaptureRequest，然后调用CameraCaptureSessionImpl的setRepeatingRequest方法将此请求发送出去：

``` java
[->/frameworks/base/core/java/android/hardware/camera2/impl/CameraCaptureSessionImpl.java]
Override
public synchronized int setRepeatingRequest(CaptureRequest request, CaptureCallback callback,
        Handler handler) throws CameraAccessException {
    if (request == null) {
        throw new IllegalArgumentException("request must not be null");
    } else if (request.isReprocess()) {
        throw new IllegalArgumentException("repeating reprocess requests are not supported");
    }

    checkNotClosed();
    handler = checkHandler(handler, callback);
    ...
    //将此请求添加到待处理的序列里
    return addPendingSequence(mDeviceImpl.setRepeatingRequest(request,createCaptureCallbackProxy(
        handler, callback), mDeviceHandler));
}
```
至此应用层的preview的请求流程分析结束，继续分析其结果处理，如果preview开启成功，则会回调CaptureReadyCallback的onReadyForCapture方法，现在分析CaptureReadyCallback回调：

``` java
[->/packages/apps/Camera2/src/com/android/camera/CaptureModule.java]
new CaptureReadyCallback() {
    @Override
    public void onSetupFailed() {
        mCameraOpenCloseLock.release();
        Log.e(TAG, "Could not set up preview.");
        mMainThread.execute(new Runnable() {
            @Override
            public void run() {
                if (mCamera == null) {
                    Log.d(TAG, "Camera closed, aborting.");
                    return;
                }
                mCamera.close();
                mCamera = null;
            }
        });
    }

    @Override
    public void onReadyForCapture() {
        mCameraOpenCloseLock.release();
        mMainThread.execute(new Runnable() {
            @Override
            public void run() {
                Log.d(TAG, "Ready for capture.");
                if (mCamera == null) {
                    Log.d(TAG, "Camera closed, aborting.");
                    return;
                }
                //
                onPreviewStarted();
                onReadyStateChanged(true);
                mCamera.setReadyStateChangedListener(CaptureModule.this);
                mUI.initializeZoom(mCamera.getMaxZoom());
                mCamera.setFocusStateListener(CaptureModule.this);
            }
        });
    }
}
```
根据前面的分析，预览成功后会回调onReadyForCapture方法，它主要是通知主线程的状态改变，并设置Camera的ReadyStateChangedListener的监听，其回调方法如下：

``` java
[->/packages/apps/Camera2/src/com/android/camera/CaptureModule.java]
Override
public void onReadyStateChanged(boolean readyForCapture) {
    if (readyForCapture) {
        mAppController.getCameraAppUI().enableModeOptions();
    }
    mAppController.setShutterEnabled(readyForCapture);
}
```
如代码所示，当其状态变成准备好拍照，则将会调用CameraActivity的setShutterEnabled方法，即使能快门按键，此时也就是说预览成功结束，可以按快门进行拍照了，所以，到这里，应用层的preview的流程基本分析完毕，下图是应用层的关键调用的流程时序图： 

![Alt text](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/camera.system/02-01-preview_java_flow.png)

##### 1.2、Camera2 startPreview的Native层流程分析
分析Preview的Native的代码真是费了九牛二虎之力，若有分析不正确之处，请各位大神指正，在第一小节的后段最后会调用CameraDeviceImpl的setRepeatingRequest方法来提交请求，而在android6.0源码分析之Camera API2.0简介中，分析了Camera2框架Java IPC通信使用了CameraDeviceUser来进行通信，所以看Native层的ICameraDeviceUser的onTransact方法来处理请求的提交：

``` java
[->/frameworks/av/camera/aidl/android/hardware/camera2/ICameraDeviceUser.aidl]
status_t BnCameraDeviceUser::onTransact(uint32_t code, const Parcel& data, Parcel* reply, 
        uint32_t flags){
    switch(code) {
        …
        //请求提交
        case SUBMIT_REQUEST: {
            CHECK_INTERFACE(ICameraDeviceUser, data, reply);

            // arg0 = request
            sp<CaptureRequest> request;
            if (data.readInt32() != 0) {
                request = new CaptureRequest();
                request->readFromParcel(const_cast<Parcel*>(&data));
            }

            // arg1 = streaming (bool)
            bool repeating = data.readInt32();

            // return code: requestId (int32)
            reply->writeNoException();
            int64_t lastFrameNumber = -1;
            //将实现BnCameraDeviceUser的对下岗的submitRequest方法代码写入Binder
            reply->writeInt32(submitRequest(request, repeating, &lastFrameNumber));
            reply->writeInt32(1);
            reply->writeInt64(lastFrameNumber);

            return NO_ERROR;
        } break;
        ...
}
```
CameraDeviceClientBase继承了BnCameraDeviceUser类，所以CameraDeviceClientBase相当于IPC Binder中的client，所以会调用其submitRequest方法，此处，至于IPC Binder通信原理不做分析，其参照其它资料：

``` cpp
[->\frameworks\av\services\camera\libcameraservice\api2\CameraDeviceClient.cpp]
status_t CameraDeviceClient::submitRequest(sp<CaptureRequest> request,bool streaming,
        /*out*/int64_t* lastFrameNumber) {
    List<sp<CaptureRequest> > requestList;
    requestList.push_back(request);
    return submitRequestList(requestList, streaming, lastFrameNumber);
}
```
简单的调用，继续分析submitRequestList：

``` cpp
[->\frameworks\av\services\camera\libcameraservice\api2\CameraDeviceClient.cpp]
status_t CameraDeviceClient::submitRequestList(List<sp<CaptureRequest> > requests,bool streaming, 
        int64_t* lastFrameNumber) {
    ...
    //Metadata链表
    List<const CameraMetadata> metadataRequestList;
    ...
    for (List<sp<CaptureRequest> >::iterator it = requests.begin(); it != requests.end(); ++it) {
        sp<CaptureRequest> request = *it;
        ...
        //初始化Metadata数据
        CameraMetadata metadata(request->mMetadata);
        ...
        //设置Stream的容量
        Vector<int32_t> outputStreamIds;
        outputStreamIds.setCapacity(request->mSurfaceList.size());
        //循环初始化Surface
        for (size_t i = 0; i < request->mSurfaceList.size(); ++i) {
            sp<Surface> surface = request->mSurfaceList[i];
            if (surface == 0) continue;
            sp<IGraphicBufferProducer> gbp = surface->getIGraphicBufferProducer();
            int idx = mStreamMap.indexOfKey(IInterface::asBinder(gbp));
            ...
            int streamId = mStreamMap.valueAt(idx);
            outputStreamIds.push_back(streamId);
        }
        //更新数据
        metadata.update(ANDROID_REQUEST_OUTPUT_STREAMS, &outputStreamIds[0],
                        outputStreamIds.size());
        if (request->mIsReprocess) {
            metadata.update(ANDROID_REQUEST_INPUT_STREAMS, &mInputStream.id, 1);
        }
        metadata.update(ANDROID_REQUEST_ID, &requestId, /*size*/1);
        loopCounter++; // loopCounter starts from 1
        //压栈
        metadataRequestList.push_back(metadata);
    }
    mRequestIdCounter++;

    if (streaming) {
        //预览会走此条通道
        res = mDevice->setStreamingRequestList(metadataRequestList, lastFrameNumber);
        if (res != OK) {
            ...
        } else {
            mStreamingRequestList.push_back(requestId);
        }
    } else {
        //Capture等走此条通道
        res = mDevice->captureList(metadataRequestList, lastFrameNumber);
        if (res != OK) {
            ...
        }
    }
    if (res == OK) {
        return requestId;
    }
    return res;
}
```
setStreamingRequestList和captureList方法都调用了submitRequestsHelper方法，只是他们的repeating参数一个ture,一个为false，而本节分析的preview调用的是setStreamingRequestList方法，并且API2.0下Device的实现为Camera3Device，所以看它的submitRequestsHelper实现：

``` cpp
[->\frameworks\av\services\camera\libcameraservice\device3\Camera3Device.cpp]
status_t Camera3Device::submitRequestsHelper(const List<const CameraMetadata> &requests, 
        bool repeating,/*out*/int64_t *lastFrameNumber) {
    ...
    RequestList requestList;
    //在这里面会进行CaptureRequest的创建，并调用configureStreamLocked进行stream的配置，主要是设置了一个回调captureResultCb，即后面要分析的重要的回调
    res = convertMetadataListToRequestListLocked(requests, /*out*/&requestList);
    ...
    if (repeating) {
        //眼熟不，这个方法名和应用层中CameraDevice的setRepeatingRequests一样
        res = mRequestThread->setRepeatingRequests(requestList, lastFrameNumber);
    } else {
        //不需重复，即repeating为false时，调用此方法来讲请求提交
        res = mRequestThread->queueRequestList(requestList, lastFrameNumber);
    }
    ...
    return res;
}
```

![Alt text](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/camera.system/02-02-preview_native_architecture.png)

从代码可知，在Camera3Device里创建了要给RequestThread线程，调用它的setRepeatingRequests或者queueRequestList方法来将应用层发送过来的Request提交，继续看setRepeatingRequests方法：

``` cpp
[->\frameworks\av\services\camera\libcameraservice\device3\Camera3Device.cpp]
status_t Camera3Device::RequestThread::setRepeatingRequests(const RequestList &requests,
        /*out*/int64_t *lastFrameNumber) {
    Mutex::Autolock l(mRequestLock);
    if (lastFrameNumber != NULL) {
        *lastFrameNumber = mRepeatingLastFrameNumber;
    }
    mRepeatingRequests.clear();
    //将其插入mRepeatingRequest链表
    mRepeatingRequests.insert(mRepeatingRequests.begin(),
            requests.begin(), requests.end());

    unpauseForNewRequests();

    mRepeatingLastFrameNumber = NO_IN_FLIGHT_REPEATING_FRAMES;
    return OK;
}
```
至此，Native层的preview过程基本分析结束，下面的工作将会交给Camera HAL层来处理，先给出Native层的调用时序图： 

![Alt text](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/camera.system/02-03-preview_native_flow.png)

##### 1.3、Camera2 startPreview的HAL层流程分析

本节将不再对Camera的HAL层的初始化以及相关配置进行分析，只对preview等相关流程中的frame metadata的处理流程进行分析，具体的CameraHAL分析请参考前一篇分析，在第二小节的submitRequestsHelper方法中调用convertMetadataListToRequestListLocked的时候会进行CaptureRequest的创建，并调用configureStreamLocked进行stream的配置，主要是设置了一个回调captureResultCb，所以Native层在request提交后，会回调此captureResultCb方法，首先分析captureResultCb：

``` cpp
[->/hardware/qcom/camera/QCamera2/HAL3/QCamera3HWI.cpp]
void QCamera3HardwareInterface::captureResultCb(mm_camera_super_buf_t *metadata_buf,
        camera3_stream_buffer_t *buffer, uint32_t frame_number)
{
    if (metadata_buf) {
        if (mBatchSize) {
            //批处理模式，但代码也是循环调用handleMetadataWithLock方法
            handleBatchMetadata(metadata_buf, true /* free_and_bufdone_meta_buf */);
        } else { /* mBatchSize = 0 */
            pthread_mutex_lock(&mMutex);    
            //处理元数据
            handleMetadataWithLock(metadata_buf, true /* free_and_bufdone_meta_buf */);
            pthread_mutex_unlock(&mMutex);
        }
    } else {
        pthread_mutex_lock(&mMutex);
        handleBufferWithLock(buffer, frame_number);
        pthread_mutex_unlock(&mMutex);
    }
    return;
}
```
一种是通过循环来进行元数据的批处理，另一种是直接进行元数据的处理，但是批处理最终也是循环调用handleMetadataWithLock来处理：

``` cpp
[->/hardware/qcom/camera/QCamera2/HAL3/QCamera3HWI.cpp]
void QCamera3HardwareInterface::handleMetadataWithLock(mm_camera_super_buf_t *metadata_buf, 
        bool free_and_bufdone_meta_buf){
    ...
    //Partial result on process_capture_result for timestamp
    if (urgent_frame_number_valid) {
        ...
        for (List<PendingRequestInfo>::iterator i =mPendingRequestsList.begin(); 
                i != mPendingRequestsList.end(); i++) {
            ...
            if (i->frame_number == urgent_frame_number &&i->bUrgentReceived == 0) {
                camera3_capture_result_t result;
                memset(&result, 0, sizeof(camera3_capture_result_t));
                i->partial_result_cnt++;
                i->bUrgentReceived = 1;
                //提取3A数据
                result.result =translateCbUrgentMetadataToResultMetadata(metadata);
                ...
                //对Capture Result进行处理
                mCallbackOps->process_capture_result(mCallbackOps, &result);
                //释放camera_metadata_t
                free_camera_metadata((camera_metadata_t *)result.result);
                break;
            }
        }
    }
    ...
    for (List<PendingRequestInfo>::iterator i = mPendingRequestsList.begin();
            i != mPendingRequestsList.end() && i->frame_number <= frame_number;) {
        camera3_capture_result_t result;
        memset(&result, 0, sizeof(camera3_capture_result_t));
        ...
        if (i->frame_number < frame_number) {
            //清空数据结构
            camera3_notify_msg_t notify_msg;
            memset(&notify_msg, 0, sizeof(camera3_notify_msg_t));
            //定义消息类型
            notify_msg.type = CAMERA3_MSG_SHUTTER;
            notify_msg.message.shutter.frame_number = i->frame_number;
            notify_msg.message.shutter.timestamp = (uint64_t)capture_time (urgent_frame_number - 
                i->frame_number) * NSEC_PER_33MSEC;
            //调用回调通知应用层发生CAMERA3_MSG_SHUTTER消息
            mCallbackOps->notify(mCallbackOps, &notify_msg);
            ...
            CameraMetadata dummyMetadata;
            //更新元数据
            dummyMetadata.update(ANDROID_SENSOR_TIMESTAMP,
                    &i->timestamp, 1);
            dummyMetadata.update(ANDROID_REQUEST_ID,
                    &(i->request_id), 1);
            //得到元数据释放结果
            result.result = dummyMetadata.release();
        } else {
            camera3_notify_msg_t notify_msg;
            memset(&notify_msg, 0, sizeof(camera3_notify_msg_t));

            // Send shutter notify to frameworks
            notify_msg.type = CAMERA3_MSG_SHUTTER;
            ...
            //从HAL中获得Metadata
            result.result = translateFromHalMetadata(metadata,
                    i->timestamp, i->request_id, i->jpegMetadata, i->pipeline_depth,
                    i->capture_intent);
            saveExifParams(metadata);
            if (i->blob_request) {
                ...
                if (enabled && metadata->is_tuning_params_valid) {
                    //将Metadata复制到文件
                    dumpMetadataToFile(metadata->tuning_params, mMetaFrameCount, enabled,
                        "Snapshot",frame_number);
                }
                mPictureChannel->queueReprocMetadata(metadata_buf);
            } else {
                // Return metadata buffer
                if (free_and_bufdone_meta_buf) {
                    mMetadataChannel->bufDone(metadata_buf);
                    free(metadata_buf);
                }
            }
        }
        ...
    }
}
```
其中，首先会调用回调的process_capture_result方法来对Capture Result进行处理，然后会调用回调的notify方法来发送一个CAMERA3_MSG_SHUTTER消息，而process_capture_result所对应的实现其实就是Camera3Device的processCaptureResult方法，先分析processCaptureResult：

``` cpp
[->\frameworks\av\services\camera\libcameraservice\device3\Camera3Device.cpp]
void Camera3Device::processCaptureResult(const camera3_capture_result *result) {
    ...
    //对于HAL3.2+,如果HAL不支持partial，当metadata被包含在result中时，它必须将partial_result设置为1
    ...
    {
        Mutex::Autolock l(mInFlightLock);
        ssize_t idx = mInFlightMap.indexOfKey(frameNumber);
        ...
        InFlightRequest &request = mInFlightMap.editValueAt(idx);
        if (result->partial_result != 0)
            request.resultExtras.partialResultCount = result->partial_result;
        // 检查结果是否只有partial metadata
        if (mUsePartialResult && result->result != NULL) {
            if (mDeviceVersion >= CAMERA_DEVICE_API_VERSION_3_2) {//HAL版本高于3.2
                if (result->partial_result > mNumPartialResults || result->partial_result < 1) {
                    //Log显示错误
                    return;
                }
                isPartialResult = (result->partial_result < mNumPartialResults);
                if (isPartialResult) {
                    //将结果加入到请求的结果集中
                    request.partialResult.collectedResult.append(result->result);
                }
            } else {//低于3.2
                ...
            }
            if (isPartialResult) {
                // Fire off a 3A-only result if possible
                if (!request.partialResult.haveSent3A) {
                    request.partialResult.haveSent3A =processPartial3AResult(frameNumber,
                        request.partialResult.collectedResult,request.resultExtras);
                }
            }
        }
        ...
        if (result->result != NULL && !isPartialResult) {
            if (shutterTimestamp == 0) {
                request.pendingMetadata = result->result;
                request.partialResult.collectedResult = collectedPartialResult;
            } else {
                CameraMetadata metadata;
                metadata = result->result;
                //发送Capture Result
                sendCaptureResult(metadata, request.resultExtras, collectedPartialResult, 
                    frameNumber, hasInputBufferInRequest,request.aeTriggerCancelOverride);
            }
        }
        //结果处理好了，将请求移除
        removeInFlightRequestIfReadyLocked(idx);
    } // scope for mInFlightLock
    ...
}
```
由代码可知，它会处理局部的或者全部的metadata数据，最后如果result不为空，且得到的是请求处理的全部数据，则会调用sendCaptureResult方法来将请求结果发送出去：

``` cpp
[->\frameworks\av\services\camera\libcameraservice\device3\Camera3Device.cpp]
void Camera3Device::sendCaptureResult(CameraMetadata &pendingMetadata,CaptureResultExtras 
        &resultExtras,CameraMetadata &collectedPartialResult,uint32_t frameNumber,bool reprocess,
        const AeTriggerCancelOverride_t &aeTriggerCancelOverride) {
    if (pendingMetadata.isEmpty())//如果数据为空，直接返回
        return;
    ...
    CaptureResult captureResult;
    captureResult.mResultExtras = resultExtras;
    captureResult.mMetadata = pendingMetadata;
    //更新metadata
    if (captureResult.mMetadata.update(ANDROID_REQUEST_FRAME_COUNT(int32_t*)&frameNumber, 1) 
            != OK) {
        SET_ERR("Failed to set frame# in metadata (%d)",frameNumber);
        return;
    } else {
        ...
    }

    // Append any previous partials to form a complete result
    if (mUsePartialResult && !collectedPartialResult.isEmpty()) {
        captureResult.mMetadata.append(collectedPartialResult);
    }
    //排序
    captureResult.mMetadata.sort();

    // Check that there's a timestamp in the result metadata
    camera_metadata_entry entry = captureResult.mMetadata.find(ANDROID_SENSOR_TIMESTAMP);
    ...
    overrideResultForPrecaptureCancel(&captureResult.mMetadata, aeTriggerCancelOverride);

    // 有效的结果，将其插入Buffer
    List<CaptureResult>::iterator queuedResult =mResultQueue.insert(mResultQueue.end(), 
        CaptureResult(captureResult));
    ...
    mResultSignal.signal();
}
```
最后，它将Capture Result插入了结果队列，并释放了结果的信号量，所以到这里，Capture Result处理成功，下面分析前面的notify发送CAMERA3_MSG_SHUTTER消息：

``` cpp
[->\frameworks\av\services\camera\libcameraservice\device3\Camera3Device.cpp]
void Camera3Device::notify(const camera3_notify_msg *msg) {

    NotificationListener *listener;
    {
        Mutex::Autolock l(mOutputLock);
        listener = mListener;
    }
    ...
    switch (msg->type) {
        case CAMERA3_MSG_ERROR: {
            notifyError(msg->message.error, listener);
            break;
        }
        case CAMERA3_MSG_SHUTTER: {
            notifyShutter(msg->message.shutter, listener);
            break;
        }
        default:
            SET_ERR("Unknown notify message from HAL: %d",
                    msg->type);
    }
}
```
它调用了notifyShutter方法：
``` cpp
[->\frameworks\av\services\camera\libcameraservice\device3\Camera3Device.cpp]
void Camera3Device::notifyShutter(const camera3_shutter_msg_t &msg,
        NotificationListener *listener) {

    ...
    // Set timestamp for the request in the in-flight tracking
    // and get the request ID to send upstream
    {
        Mutex::Autolock l(mInFlightLock);
        idx = mInFlightMap.indexOfKey(msg.frame_number);
        if (idx >= 0) {
            InFlightRequest &r = mInFlightMap.editValueAt(idx);
            // Call listener, if any
            if (listener != NULL) {
                //调用监听的notifyShutter法国法
                listener->notifyShutter(r.resultExtras, msg.timestamp);
            }
            ...
            //将待处理的result发送到Buffer
            sendCaptureResult(r.pendingMetadata, r.resultExtras,
                r.partialResult.collectedResult, msg.frame_number,
                r.hasInputBuffer, r.aeTriggerCancelOverride);
            returnOutputBuffers(r.pendingOutputBuffers.array(),
                r.pendingOutputBuffers.size(), r.shutterTimestamp);
            r.pendingOutputBuffers.clear();
            removeInFlightRequestIfReadyLocked(idx);
        }
    }
    ...
}
```
首先它会通知listener preview成功，最后会调用sendCaptureResult将结果加入到结果队列。它会调用listener的notifyShutter方法，此处的listener其实是CameraDeviceClient类，所以会调用CameraDeviceClient类的notifyShutter方法：

``` cpp
[->\frameworks\av\services\camera\libcameraservice\api2\CameraDeviceClient.cpp]
void CameraDeviceClient::notifyShutter(const CaptureResultExtras& resultExtras,nsecs_t timestamp) {
    // Thread safe. Don't bother locking.
    sp<ICameraDeviceCallbacks> remoteCb = getRemoteCallback();
    if (remoteCb != 0) {
        //调用应用层的回调(CaptureCallback的onCaptureStarted方法)
        remoteCb->onCaptureStarted(resultExtras, timestamp);
    }
}
```
此处的ICameraDeviceCallbacks对应的是Java层的CameraDeviceImpl.java中的内部类CameraDeviceCallbacks，所以会调用它的onCaptureStarted方法：
``` cpp
[->\frameworks\base\core\java\android\hardware\camera2\impl\CameraDeviceImpl.java]
@Override
public void onCaptureStarted(final CaptureResultExtras resultExtras, final long timestamp) {
    int requestId = resultExtras.getRequestId();
    final long frameNumber = resultExtras.getFrameNumber();
    final CaptureCallbackHolder holder;

    synchronized(mInterfaceLock) {
        if (mRemoteDevice == null) return; // Camera already closed
        // Get the callback for this frame ID, if there is one
        holder = CameraDeviceImpl.this.mCaptureCallbackMap.get(requestId);
        ...
        // Dispatch capture start notice
        holder.getHandler().post(new Runnable() {
            @Override
            public void run() {
                if (!CameraDeviceImpl.this.isClosed()) {
                    holder.getCallback().onCaptureStarted(CameraDeviceImpl.this,holder.getRequest(
                        resultExtras.getSubsequenceId()),timestamp, frameNumber);
                }
           }
       });
   }
}
```
它会调用OneCameraImpl.java中的mCaptureCallback的onCaptureStarted方法：
``` cpp
[->\frameworks\base\core\java\android\hardware\camera2\impl\CameraDeviceImpl.java]
//Common listener for preview frame metadata.  
private final CameraCaptureSession.CaptureCallback mCaptureCallback =
    new CameraCaptureSession.CaptureCallback() {
        @Override
        public void onCaptureStarted(CameraCaptureSession session,CaptureRequest request, 
            long timestamp,long frameNumber) {
            if (request.getTag() == RequestTag.CAPTURE&& mLastPictureCallback != null) {
                mLastPictureCallback.onQuickExpose();
            }
        }
        …
}
```

> camera工作时，存在了５中流处理线程和一个专门向hal发送请求的request线程。线程之间通过信号来同步，稍不注意就搞不明白代码是如何运行的了。其中很容易让我们忽视的就是在流发送之前的parent->registerInFlight()该操作将当前的请求保存到一个数组(可以理解成)中。这个数组对象在后续回帧操作中，会将相应帧的shutter,时间戳信息填充到对应的request中，紧接着就把对应帧的信息返回给app。好了先到这吧，下一篇分析Camera recording流程。

![Alt text](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/camera.system/02-04-preview_hal_architecture.png)


注意：Capture,preview以及autoFocus都是使用的这个回调，而Capture调用的时候，其RequestTag为CAPTURE，而autoFocus的时候为TAP_TO_FOCUS,而preview请求时没有对RequestTag进行设置，所以回调到onCaptureStarted方法时，不需要进行处理，但是到此时，preview已经启动成功，可以进行预览了，其数据都在buffer里。所以到此时，preview的流程全部分析结束，下面给出HAL层上的流程时序图 

![Alt text](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/camera.system/02-05-preview_hal_flow.png)

#### （二）、Camera System takePicture流程分析
与TakePicture息息相关的主要有4个线程CaptureSequencer,JpegProcessor,Camera3Device::RequestThread,FrameProcessorBase如下面的代码可以发现，在Camera2client对象初始化后，已经有３个线程已经run起来了，还有有一个RequestThread线程会在Camera3Device初始化时创建的。他们工作非常密切，如下大概画了一个他们的工作机制，４个线程都是通过Conditon条件变量来同步的。 

![Alt text](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/camera.system/02-06-Camera3Device-RequestThread.png)

前面分析preview的时候，当预览成功后，会使能ShutterButton，即可以进行拍照，定位到ShutterButton的监听事件为onShutterButtonClick方法：

``` java
[->/packages/apps/Camera2/src/com/android/camera/CaptureModule.java]
@Override
public void onShutterButtonClick() {
    //Camera未打开
    if (mCamera == null) {
        return;
    }

    int countDownDuration = mSettingsManager.getInteger(SettingsManager
        .SCOPE_GLOBAL,Keys.KEY_COUNTDOWN_DURATION);
    if (countDownDuration > 0) {
        // 开始倒计时
        mAppController.getCameraAppUI().transitionToCancel();
        mAppController.getCameraAppUI().hideModeOptions();
        mUI.setCountdownFinishedListener(this);
        mUI.startCountdown(countDownDuration);
        // Will take picture later via listener callback.
    } else {
        //即刻拍照
        takePictureNow();
    }
}
```
首先，读取Camera的配置，判断配置是否需要延时拍照，此处分析不需延时的情况，即调用takePictureNow方法：

``` java
[->/packages/apps/Camera2/src/com/android/camera/CaptureModule.java]
private void takePictureNow() {
    if (mCamera == null) {
        Log.i(TAG, "Not taking picture since Camera is closed.");
        return;
    }
    //创建Capture会话并开启会话
    CaptureSession session = createAndStartCaptureSession();
    //获取Camera的方向
    int orientation = mAppController.getOrientationManager()
        .getDeviceOrientation().getDegrees();
    //初始化图片参数
    PhotoCaptureParameters params = new PhotoCaptureParameters(
            session.getTitle(), orientation, session.getLocation(),
            mContext.getExternalCacheDir(), this, mPictureSaverCallback,
            mHeadingSensor.getCurrentHeading(), mZoomValue, 0);
    //装配Session
    decorateSessionAtCaptureTime(session);
    //拍照
    mCamera.takePicture(params, session);
}
```
它首先调用createAndStartCaptureSession来创建一个CaptureSession并且启动会话,这里并且会进行初始参数的设置，譬如设置CaptureModule(此处实参为this)为图片处理的回调(后面再分析)：

``` java
[->/packages/apps/Camera2/src/com/android/camera/CaptureModule.java]
private CaptureSession createAndStartCaptureSession() {
    //获取会话时间
    long sessionTime = getSessionTime();
    //当前位置
    Location location = mLocationManager.getCurrentLocation();
    //设置picture name
    String title = CameraUtil.instance().createJpegName(sessionTime);
    //创建会话
    CaptureSession session = getServices().getCaptureSessionManager()
           .createNewSession(title, sessionTime, location);
    //开启会话
    session.startEmpty(new CaptureStats(mHdrPlusEnabled),new Size(
        (int) mPreviewArea.width(), (int) mPreviewArea.height()));
    return session;
}
```
首先，获取会话的相关参数，包括会话时间，拍照的照片名字以及位置信息等，然后调用Session管理来创建CaptureSession，最后将此CaptureSession启动。到这里，会话就创建并启动了，所以接着分析上面的拍照流程，它会调用OneCameraImpl的takePicture方法来进行拍照：

``` java
[->/packages/apps/Camera2/src/com/android/camera/one/v2/OneCameraImpl.java]
@Override
public void takePicture(final PhotoCaptureParameters params, final CaptureSession session) {
    ...
    // 除非拍照已经返回，否则就广播一个未准备好状态的广播，即等待本次拍照结束
    broadcastReadyState(false);
    //创建一个线程
    mTakePictureRunnable = new Runnable() {
        @Override
        public void run() {
            //拍照
            takePictureNow(params, session);
        }
    };
    //设置回调，此回调后面将分析，它其实就是CaptureModule,它实现了PictureCallback
    mLastPictureCallback = params.callback;
    mTakePictureStartMillis = SystemClock.uptimeMillis();

    //如果需要自动聚焦
    if (mLastResultAFState == AutoFocusState.ACTIVE_SCAN) {
        mTakePictureWhenLensIsStopped = true;
    } else {
        //拍照
        takePictureNow(params, session);
    }
}
```
在拍照里，首先广播一个未准备好的状态广播，然后进行拍照的回调设置，并且判断是否有自动聚焦，如果是则将mTakePictureWhenLensIsStopped 设为ture，即即刻拍照被停止了，否则则调用OneCameraImpl的takePictureNow方法来发起拍照请求：

``` java
[->/packages/apps/Camera2/src/com/android/camera/one/v2/OneCameraImpl.java]
public void takePictureNow(PhotoCaptureParameters params, CaptureSession 
        session) {
    long dt = SystemClock.uptimeMillis() - mTakePictureStartMillis;
    try {
        // 构造JPEG图片拍照的请求
        CaptureRequest.Builder builder = mDevice.createCaptureRequest(
            CameraDevice.TEMPLATE_STILL_CAPTURE);
        builder.setTag(RequestTag.CAPTURE);
        addBaselineCaptureKeysToRequest(builder);

        // Enable lens-shading correction for even better DNGs.
        if (sCaptureImageFormat == ImageFormat.RAW_SENSOR) {
            builder.set(CaptureRequest.STATISTICS_LENS_SHADING_MAP_MODE,
                CaptureRequest.STATISTICS_LENS_SHADING_MAP_MODE_ON);
        } else if (sCaptureImageFormat == ImageFormat.JPEG) {
            builder.set(CaptureRequest.JPEG_QUALITY, JPEG_QUALITY);
                .getJpegRotation(params.orientation, mCharacteristics));
        }
        //用于preview的控件
        builder.addTarget(mPreviewSurface);
        //用于图片显示的控件
        builder.addTarget(mCaptureImageReader.getSurface());
        CaptureRequest request = builder.build();

        if (DEBUG_WRITE_CAPTURE_DATA) {
            final String debugDataDir = makeDebugDir(params.debugDataFolder,
                        "normal_capture_debug");
            Log.i(TAG, "Writing capture data to: " + debugDataDir);
            CaptureDataSerializer.toFile("Normal Capture", request, 
                new File(debugDataDir,"capture.txt"));
        }
        //拍照，mCaptureCallback为回调
        mCaptureSession.capture(request, mCaptureCallback, mCameraHandler);
    } catch (CameraAccessException e) {
        Log.e(TAG, "Could not access camera for still image capture.");
        broadcastReadyState(true);
        params.callback.onPictureTakingFailed();
        return;
    }
    synchronized (mCaptureQueue) {
        mCaptureQueue.add(new InFlightCapture(params, session));
    }
}
```
与preview类似，都是通过CaptureRequest来与Camera进行通信的，通过session的capture来进行拍照，并设置拍照的回调函数为mCaptureCallback：

``` java
[->/frameworks/base/core/java/android/hardware/camera2/impl/CameraCaptureSessionImpl.java]
@Override
public synchronized int capture(CaptureRequest request,CaptureCallback callback,Handler handler)throws CameraAccessException{
    ...
    handler = checkHandler(handler,callback);
    return addPendingSequence(mDeviceImpl.capture(request,createCaptureCallbackProxy(
        handler,callback),mDeviceHandler));
}
```
代码与preview中的类似，都是将请求加入到待处理的请求集，现在看CaptureCallback回调：

``` java
[->/packages/apps/Camera2/src/com/android/camera/one/v2/OneCameraImpl.java]
private final CameraCaptureSession.CaptureCallback mCaptureCallback = new CameraCaptureSession.CaptureCallback(){
    @Override
    public void onCaptureStarted(CameraCaptureSession session,CaptureRequest request,long 
            timestamp,long frameNumber){
　　　　　//与preview类似
        if(request.getTag() == RequestTag.CAPTURE&&mLastPictureCallback!=null){
            mLastPictureCallback.onQuickExpose();
        }
    }
    ...
    @Override
    public void onCaptureCompleted(CameraCaptureSession session,CaptureRequest request
            ,TotalCaptureResult result){
        autofocusStateChangeDispatcher(result);
        if(result.get(CaptureResult.CONTROL_AF_STATE) == null){
　　　　　　　//检查自动聚焦的状态
            AutoFocusHelper.checkControlAfState(result);
        }
        ...
        if(request.getTag() == RequestTag.CAPTURE){
            synchronized(mCaptureQueue){
                if(mCaptureQueue.getFirst().setCaptureResult(result).isCaptureComplete()){
                    capture = mCaptureQueue.removeFirst();
                }
            }
            if(capture != null){
　　　　　　　　　//拍照结束
                OneCameraImpl.this.onCaptureCompleted(capture);
            }
        }
        super.onCaptureCompleted(session,request,result);
    }
    ...
}
```
这是Native层在处理请求时，会调用相应的回调，如capture开始时，会回调onCaptureStarted,具体的在preview中有过分析，当拍照结束时，会回调onCaptureCompleted方法，其中会根据CaptureResult来检查自动聚焦的状态，并通过TAG判断其是Capture动作时，再来看它是否是队列中的第一个请求，如果是，则将请求移除，因为请求已经处理成功，最后再调用OneCameraImpl的onCaptureCompleted方法来进行处理：

``` java
[->/packages/apps/Camera2/src/com/android/camera/one/v2/OneCameraImpl.java]
private void onCaptureCompleted(InFlightCapture capture){
    if(isCaptureImageFormat == ImageFormat.RAW_SENSOR){
        ...
        File dngFile = new File(RAW_DIRECTORY,capture.session.getTitle()+".dng");
        writeDngBytesAndClose(capture.image,capture.totalCaptureResult,mCharacteristics,dngFile);
    }else{
        //解析result中的图片数据
        byte[] imageBytes = acquireJpegBytesAndClose(capture.image);
        //保存Jpeg图片
        saveJpegPicture(imageBytes,capture.parameters,capture.session,capture.totalCaptureResult);
    }
    broadcastReadyState(true);
    //调用回调
    capture.parameters.callback.onPictureTaken(capture.session);
}
```
如代码所示，首先，对result中的图片数据进行了解析，然后调用saveJpegPicture方法将解析得到的图片数据进行保存，最后再调用里面的回调(即CaptureModule，前面在初始化Parameters时说明了，它实现了PictureCallbak接口)的onPictureTaken方法，所以，接下来先分析saveJpegPicture方法：

``` java
[->/packages/apps/Camera2/src/com/android/camera/one/v2/OneCameraImpl.java]
private void saveJpegPicture(byte[] jpegData,final PhotoCaptureParameters captureParams,CaptureSession session,CaptureResult result){
    ...
    ListenableFuture<Optional<Uri>> futureUri = session.saveAndFinish(jpegData,width,
            height,rotation,exif);
    Futures.addCallback(futureUri,new FutureCallback<Optional<Uri>>(){
        @Override
        public void onSuccess(Optional<Uri> uriOptional){
            captureParams.callback.onPictureSaved(mOptional.orNull());
        }

        @Override
        public void onFailure(Throwable throwable){
            captureParams.callback.onPictureSaved(null);
        }
    });
}
```
它最后会回调onPictureSaved方法来对图片进行保存，所以需要分析CaptureModule的onPictureSaved方法：

``` java
[->/packages/apps/Camera2/src/com/android/camera/CaptureModule.java]
@Override
public void onPictureSaved(Uri uri){
    mAppController.notifyNewMedia(uri);
}
```
mAppController的实现为CameraActivity，所以分析notifyNewMedia方法：

``` java
[->/packages/apps/Camera2/src/com/android/camera/CameraActivity.java]
@Override
public void notifyNewMedia(Uri uri){
    ...
    if(FilmstripItemUtils.isMimeTypeVideo(mimeType)){
　　　　//如果拍摄的是video
        sendBroadcast(new Intent(CameraUtil.ACTION_NEW_VIDEO,uri));
        newData = mVideoItemFactory.queryContentUri(uri);
        ...
    }else if(FilmstripItemUtils.isMimeTypeImage(mimeType)){
　　　　//如果是拍摄图片
        CameraUtil.broadcastNewPicture(mAppContext,uri);
        newData = mPhotoItemFactory.queryCotentUri(uri);
        ...
    }else{
        return;
    }
    new AsyncTask<FilmstripItem,Void,FilmstripItem>(){
        @Override
        protected FilmstripItem doInBackground(FilmstripItem... Params){
            FilmstripItem data = params[0];
            MetadataLoader.loadMetadata(getAndroidContet(),data);
            return data;
        }
        ...
    }
}
```

由代码可知，这里有两种数据的处理，一种是video，另一种是image。而我们这里分析的是capture图片数据，所以首先会根据在回调函数传入的参数Uri和PhotoItemFactory来查询到相应的拍照数据，然后再开启一个异步的Task来对此数据进行处理，即通过MetadataLoader的loadMetadata来加载数据，并返回。至此，capture的流程就基本分析结束了，下面将给出capture流程的整个过程中的时序图： 

![Alt text](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/camera.system/02-07-capture_over_flow.png)

#### （三）、Camera System takepicture(ZSL)流程分析（本小节基于 Android 5.1 API11源码）
ZSL(zear shutter lag)即零延时，就是在拍照时不停预览就可以拍照.由于有较好的用户体验度，该feature是现在大部分手机都拥有的功能。 
面不再贴出大量代码来描述过程，直接上图。下图是画了2个小时整理出来的Android5.1 Zsl的基本流程，可以看到与ZSL密切相关的有5个线程frameprocessor、captureSequencer、ZslProcessor3、JpegProcessor、Camera3Device:requestThread。其实还有一个主线程用于更新参数。针对Android5.1看代码所得，ZSL过程中大概分成下面7个流程.

![Alt text](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/camera.system/02-08-ZSL_takepicture.png)


更正：图中左上角的FrameProcessor线程起来后会在waitForNextFrame中执行mResultSignal.waitRelative()，图中没有更改过来。

##### 3.0、注册帧监听对象

##### 3.0.1、captureSequence线程注册帧监听对象

##### 3.0.1.1、注册时机 
当上层发出ZSL拍照请求时，底层就会触发拍照捕获状态机，改状态机的基本流程图在上篇笔记中已经整理出来过，这里就不多说了。由于camera2Client与其它处理线程对象基本符合金字塔形的架构，可以看到这里是通过camera2Client的对象将帧可用监听对象注册到FrameProcess对象中的List<RangeListener> mRangeListeners;对象中。

``` cpp
[->/frameworks/av/services/camera/libcameraservice/api1/client2/CaptureSequencer.cpp]
CaptureSequencer::CaptureState CaptureSequencer::manageZslStart(
        sp<Camera2Client> &client) {
    ALOGV("%s", __FUNCTION__);
    status_t res;
    sp<ZslProcessorInterface> processor = mZslProcessor.promote();
    // We don't want to get partial results for ZSL capture.
    client->registerFrameListener(mCaptureId, mCaptureId + 1,
            this,
            /*sendPartials*/false);
    // TODO: Actually select the right thing here.
    res = processor->pushToReprocess(mCaptureId);
    //.......
}
```
特别注意：可以看到在注册帧监听对象时，传入的两个参数是mCaptureId, mCaptureId + 1,为什么会是这样呢,因为这个就是标记我们想抓的是哪一帧,当拍照buffer从hal上来之后,Camera3Device就会回调帧可用监听对象，然后得到拍照帧的时间戳，紧接着根据时间戳从ZSL RingBuffer中找到最理想的inputBuffer，然后下发给hal进行Jpeg编解码。对比下面ZSL线程的CaptureId,应该就理解了.

##### 3.0.1.2、捕获时机

``` cpp
[->/frameworks/av/services/camera/libcameraservice/api1/client2/CaptureSequencer.cpp]
void CaptureSequencer::onResultAvailable(const CaptureResult &result) {
    ATRACE_CALL();
    ALOGV("%s: New result available.", __FUNCTION__);
    Mutex::Autolock l(mInputMutex);
    mNewFrameId = result.mResultExtras.requestId;
    mNewFrame = result.mMetadata;
    if (!mNewFrameReceived) {
        mNewFrameReceived = true;
        .signal();
    }
}
```
上面即是拍照状态机注册的回调函数，其中当ZSL拍照帧上来之后，机会激活正在等待中的CaptureSequencer线程，以进行后续的操作。

#####  3.0.2、ZslProcess3线程注册帧监听对象
#####  3.0.2.1、注册时机

``` cpp
[->/frameworks/av/services/camera/libcameraservice/api1/client2/ZslProcessor.cpp]
status_t ZslProcessor::updateStream(const Parameters &params) {
    if (mZslStreamId == NO_STREAM) {
        // Create stream for HAL production
        // TODO: Sort out better way to select resolution for ZSL

        // Note that format specified internally in Camera3ZslStream
        res = device->createZslStream(
                params.fastInfo.arrayWidth, params.fastInfo.arrayHeight,
                mBufferQueueDepth,
                &mZslStreamId,
                &mZslStream);
        // Only add the camera3 buffer listener when the stream is created.
        mZslStream->addBufferListener(this);//这里是在BufferQueue注册的callback，暂时不用关心。
    }
    client->registerFrameListener(Camera2Client::kPreviewRequestIdStart,
            Camera2Client::kPreviewRequestIdEnd,
            this,
            /*sendPartials*/false);

    return OK;
}
```
上面的即为更新zsl流时调用的函数,可以看到其中使用registerFrameListener注册了RingBuffer可用监听对象，这里我们要特别注意的是下面2个宏。这个是专门为预览预留的requestId，考虑这样也会有录像和拍照的requestId,每次更新参数后，这个requestId会有+1操作，没有参数更新，则不会+1，这个可以在各自的Debug手机上发现。

``` cpp
[->/frameworks/av/services/camera/libcameraservice/api1/Camera2Client.h]
static const int32_t kPreviewRequestIdStart = 10000000;
static const int32_t kPreviewRequestIdEnd   = 20000000;
```

#####  3.0.2.2、捕获时机

``` cpp
[->/frameworks/av/services/camera/libcameraservice/api1/client2/ZslProcessor.cpp]
void ZslProcessor::onResultAvailable(const CaptureResult &result) {
    ATRACE_CALL();
    ALOGV("%s:", __FUNCTION__);
    Mutex::Autolock l(mInputMutex);
    camera_metadata_ro_entry_t entry;
    entry = result.mMetadata.find(ANDROID_SENSOR_TIMESTAMP);
    nsecs_t timestamp = entry.data.i64[0];

    entry = result.mMetadata.find(ANDROID_REQUEST_FRAME_COUNT);
    int32_t frameNumber = entry.data.i32[0];

    // Corresponding buffer has been cleared. No need to push into mFrameList
    if (timestamp <= mLatestClearedBufferTimestamp) return;

    mFrameList.editItemAt(mFrameListHead) = result.mMetadata;
    mFrameListHead = (mFrameListHead + 1) % mFrameListDepth;
}
```
去掉错误检查代码，上面由于CaptureID是下面2个，也就是ZSL的所有预览Buffer可用之后都会回调这个方法,当队列满之后，新buffer会覆盖旧buffer位置。上面可以看到mFrameList中会保存每一帧的metadata数据，mFrameListHead用来标识下一次存放数据的位置。

#####  3.1、查找ZSL拍照最合适的buffer
一开始我以为是是根据想要抓取那帧的captureId来找到zsl拍照buffer的，但是现在看来就是找时间戳最近的那个buffer来进行jpeg编解码(而且google工程师在源码中注释也是这样说的).

``` cpp
[->/frameworks/av/services/camera/libcameraservice/api1/client2/ZslProcessor.cpp]
status_t ZslProcessor::pushToReprocess(int32_t requestId) {
    ALOGV("%s: Send in reprocess request with id %d",
            __FUNCTION__, requestId);
    Mutex::Autolock l(mInputMutex);
    status_t res;
    sp<Camera2Client> client = mClient.promote();
    //下面是就是在mFrameList查找时间戳最近的帧。
    size_t metadataIdx;
    nsecs_t candidateTimestamp = getCandidateTimestampLocked(&metadataIdx);
   //根据上一次查找的时间戳，从ZSL BufferQueue中查找时间最接近的Buffer，并将
   //buffer保存到mInputBufferQueue队列中。
    res = mZslStream->enqueueInputBufferByTimestamp(candidateTimestamp,
                                                    /*actualTimestamp*/NULL);
   //-----------------
    {//获取zsl 编解码的metadataId，稍后会传入给hal编解码。
        CameraMetadata request = mFrameList[metadataIdx];

        // Verify that the frame is reasonable for reprocessing
        camera_metadata_entry_t entry;
        entry = request.find(ANDROID_CONTROL_AE_STATE);

        if (entry.data.u8[0] != ANDROID_CONTROL_AE_STATE_CONVERGED &&
                entry.data.u8[0] != ANDROID_CONTROL_AE_STATE_LOCKED) {
            ALOGV("%s: ZSL queue frame AE state is %d, need full capture",
                    __FUNCTION__, entry.data.u8[0]);
            return NOT_ENOUGH_DATA;
        }
       //这中间会更新输入stream的流ID、更新捕获意图为静态拍照、判断这一帧是否AE稳定、
       //获取jpegStreamID并更新到metadata中、更新请求ID，最后根据更新后的request metadata
       //更新jpeg metadata。最后一步启动Camera3Device抓取图片。
        // Update post-processing settings
        res = updateRequestWithDefaultStillRequest(request);
        mLatestCapturedRequest = request;
        res = client->getCameraDevice()->capture(request);
        mState = LOCKED;
    }
    return OK;
}
```
 还记得在启动状态机器时，注册的帧监听对象吧。这里参数requestId就是我们想要抓拍的图片的请求ID,目前发现该请求ID后面会更新到metadata中。这里只要知道该函数功能就可以了。

☯ 1、从mFrameList中查找时间戳最小的metadata。
☯ 2、根据从第一步获取到时间戳，从ZSL BufferQueue选择时间最接近Buffer.
☯3、将Buffer放到mInputBufferQueue中，更新jpeg编解码metadata，启动Capture功能。

#####  3.2、设置zsl input buffer和 jpeg out buffer
  其实这一步之前已经讨论过，inputBuffer是ZslProcess3线程查找到最合适的用于jpeg编解码的buffer。outputBuffer为JpegProcessor线程更新的buffer用于存放hal编解码之后的jpeg图片。其中准备jpeg OutBuffer的操作就是在下面操作的。可以看到将outputStream的ID，保存到metadata中了。这样就会在Camera3Device中根据这项metadata来添加outputBuffer到hal。

```cpp
status_t ZslProcessor::pushToReprocess(int32_t requestId) {
        // TODO: Shouldn't we also update the latest preview frame?
        int32_t outputStreams[1] =
                { client->getCaptureStreamId() };
        res = request.update(ANDROID_REQUEST_OUTPUT_STREAMS,
                outputStreams, 1);
 }
```
#####  3.3、归还jpeg Buffer干了什么.
  当framework将ZSL inputBuffer和jpeg outputBuffer,传给hal后，hal就会启动STLL_CAPTURE流程，将inputBuffer中的图像数据，进行一系列的后处理流程。当后处理完成后，hal则会将临时Buffer拷贝到outPutBuffer中(注意：这里要记得做flush操作，即刷新Buffer,要不然图片有可能会出现绿条). 
  因为JpegBuffer也是从BufferQueue Dequeue出来的buffer,而且在创建BufferQueue时，也注册了帧监听对象(即：onFrameAvailable()回调).这样的话当帧可用(即：进行了enqueue操作），就会回调onFrameAvailable()方法，这样当hal归还jpegBuffer时就是要进行enqueue()操作。在onFrameAvailable()方法中，会激活jpegproces线程，进行后续的处理，最后激活captureSequeue拍照状态机线程。

#####  3.4、保存ZSLBuffer.
  这里由于ZSL Buffer一直会从hal上来，所以当zslBuffer上来后，就会激活FrameProcesor线程保存这一ZslBuffer，目前FrameWork那边默认是4个buffer，这样的话当队列满之后，就会覆盖之前最老的buffer,如此反复操作。

#####  3.5、获取拍照jpeg Buffer
  当hal上来jpeg帧后，就会激活jpegProcess线程,并从BufferQueue中拿到jpegbuffer，下面可以发现进行lockNextBuffer,unlockBuffer操作。

``` cpp
[->/frameworks/av/services/camera/libcameraservice/api1/client2/JpegProcessor.cpp]
status_t JpegProcessor::processNewCapture() {
    res = mCaptureConsumer->lockNextBuffer(&imgBuffer);
    mCaptureConsumer->unlockBuffer(imgBuffer);
    sp<CaptureSequencer> sequencer = mSequencer.promote();
   //...... 
    if (sequencer != 0) {
        sequencer->onCaptureAvailable(imgBuffer.timestamp, captureBuffer);
    }
}
```
上面可以发现最后回调了captureSequencer线程的onCaptureAvailable()回调方法。该回调方法主要作用就是将时间戳和jpeg buffer的传送到CaptureSequencer线程中,然后激活CaptureSequencer线程。最后将Buffer CallBack到应用层。

``` cpp
[->/frameworks/av/services/camera/libcameraservice/api1/client2/CaptureSequencer.cpp]
void CaptureSequencer::onCaptureAvailable(nsecs_t timestamp,
        sp<MemoryBase> captureBuffer) {
    ATRACE_CALL();
    ALOGV("%s", __FUNCTION__);
    Mutex::Autolock l(mInputMutex);
    mCaptureTimestamp = timestamp;
    mCaptureBuffer = captureBuffer;
    if (!mNewCaptureReceived) {
        mNewCaptureReceived = true;
        mNewCaptureSignal.signal();
    }
}
```
#####  3.6、拍照帧可用回调
当拍照帧回到Framework后，就会回调CaptureSequencer的onResultAvailable()接口，用于设置captureSequencer状态机的标志位和条件激活,如下代码所示。条件变量和标志位的使用可以在状态机方法manageStandardCaptureWait()看到使用。

``` cpp
[->/frameworks/av/services/camera/libcameraservice/api1/client2/CaptureSequencer.cpp]
void CaptureSequencer::onResultAvailable(const CaptureResult &result) {
    ATRACE_CALL();
    ALOGV("%s: New result available.", __FUNCTION__);
    Mutex::Autolock l(mInputMutex);
    mNewFrameId = result.mResultExtras.requestId;
    mNewFrame = result.mMetadata;
    if (!mNewFrameReceived) {
        mNewFrameReceived = true;
        mNewFrameSignal.signal();
    }
}
```
#####  3.7、jpeg buffer回调到app
该callback是应用注册过来的一个代理对象，下面就是通过binder进程间调用将jpeg Buffer传送到APP端，注意这里的msgTyep = CAMERA_MSG_COMPRESSED_IMAGE,就是告诉上层这是一个压缩的图像数据。

``` cpp
[->/frameworks/av/services/camera/libcameraservice/api1/client2/CaptureSequencer.cpp]
CaptureSequencer::CaptureState CaptureSequencer::manageDone(sp<Camera2Client> &client) {
    status_t res = OK;
    ATRACE_CALL();
    mCaptureId++;
        ......
            Camera2Client::SharedCameraCallbacks::Lock
            l(client->mSharedCameraCallbacks);
        ALOGV("%s: Sending still image to client", __FUNCTION__);
        if (l.mRemoteCallback != 0) {
            l.mRemoteCallback->dataCallback(CAMERA_MSG_COMPRESSED_IMAGE,
                    mCaptureBuffer, NULL);
        } else {
            ALOGV("%s: No client!", __FUNCTION__);
        }
        ......
}
```
#### （四）、Camera System Recorder流程分析
camera Video.虽然标题是recording流程分析，但这里很多和preview是相似的(包含更新，创建Stream,创建Request)，这里主要分析MediaRecorder对象创建、video帧监听对象注册、帧可用事件以及一系列callback流程分析。

##### 4.1、认识video(mediaRecorder)状态机

![Alt text](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/camera.system/02-09-mediaRecorder-status.png)

> Used to record audio and video. The recording control is based on a 
simple state machine (see below).状态机请看上面源码中给的流程图。 
A common case of using MediaRecorder to record audio works as follows: 
1.MediaRecorder recorder = new MediaRecorder(); 
2.recorder.setAudioSource(MediaRecorder.AudioSource.MIC); 
3.recorder.setOutputFormat(MediaRecorder.OutputFormat.THREE_GPP); 
4.recorder.setAudioEncoder(MediaRecorder.AudioEncoder.AMR_NB); 
5.recorder.setOutputFile(PATH_NAME); 
6.recorder.prepare(); 
7.recorder.start(); // Recording is now started 
8…. 
9.recorder.stop(); 
10.recorder.reset(); // You can reuse the object by going back to setAudioSource() step 
recorder.release(); // Now the object cannot be reused 
  Applications may want to register for informational and error 
events in order to be informed of some internal update and possible 
runtime errors during recording. Registration for such events is 
done by setting the appropriate listeners (via calls 
(to {@link #setOnInfoListener(OnInfoListener)}setOnInfoListener and/or 
{@link #setOnErrorListener(OnErrorListener)}setOnErrorListener). 
In order to receive the respective callback associated with these listeners, 
applications are required to create MediaRecorder objects on threads with a 
Looper running (the main UI thread by default already has a Looper running).

上面是googole工程师加的注释，最权威的资料。大概意思就是说“使用mediaRecorder记录音视频，需要一个简单的状态机来控制”。上面的1,2,3…就是在操作时需要准守的步骤。算了吧，翻译水平有限，重点还是放到camera这边吧。

##### 4.2、Camera app如何启动录像

``` java
//源码路径:pdk/apps/TestingCamera/src/com/android/testingcamera/TestingCamera.java
 private void startRecording() {
        log("Starting recording");
        logIndent(1);
        log("Configuring MediaRecoder");
        //这里会检查是否打开了录像功能。这里我们省略了，直接不如正题
//上面首先创建了一个MediaRecorder的java对象(注意这里同camera.java类似，java对象中肯定包含了一个mediaRecorder jni本地对象，继续往下看)
        mRecorder = new MediaRecorder();
        //下面就是设置一些callback.
        mRecorder.setOnErrorListener(mRecordingErrorListener);
        mRecorder.setOnInfoListener(mRecordingInfoListener);
        if (!mRecordHandoffCheckBox.isChecked()) {
    //将当前camera java对象设置给了mediaRecorder java对象。
    //这里setCamera是jni接口，后面我们贴代码在分析。
            mRecorder.setCamera(mCamera);
        }
    //将preview surface java对象设置给mediaRecorder java对象，后面贴代码
    //详细说明。
        mRecorder.setPreviewDisplay(mPreviewHolder.getSurface());
　　　　//下面２个是设置音频和视频的资源。
        mRecorder.setAudioSource(MediaRecorder.AudioSource.CAMCORDER);
        mRecorder.setVideoSource(MediaRecorder.VideoSource.CAMERA);
        mRecorder.setProfile(mCamcorderProfiles.get(mCamcorderProfile));
        //从app控件选择录像帧大小，并设置给mediaRecorder
        Camera.Size videoRecordSize = mVideoRecordSizes.get(mVideoRecordSize);
        if (videoRecordSize.width > 0 && videoRecordSize.height > 0) {
            mRecorder.setVideoSize(videoRecordSize.width, videoRecordSize.height);
        }
        //从app控件选择录像帧率，并设置给mediaRecorder.
        if (mVideoFrameRates.get(mVideoFrameRate) > 0) {
            mRecorder.setVideoFrameRate(mVideoFrameRates.get(mVideoFrameRate));
        }
        File outputFile = getOutputMediaFile(MEDIA_TYPE_VIDEO);
        log("File name:" + outputFile.toString());
        mRecorder.setOutputFile(outputFile.toString());

        boolean ready = false;
        log("Preparing MediaRecorder");
        try {
    //准备一下，请看下面google给的使用mediaRecorder标准流程
            mRecorder.prepare();
            ready = true;
        } catch (Exception e) {//------异常处理省略
        }

        if (ready) {
            try {
                log("Starting MediaRecorder");
                mRecorder.start();//启动录像
                mState = CAMERA_RECORD;
                log("Recording active");
                mRecordingFile = outputFile;
            } catch (Exception e) {//-----异常处理省略
        }
//------------
    }
```
可以看到应用启动录像功能是是符合状态机流程的。在应用开发中，也要这样来做。

☯ 1.创建mediaRecorderjava对象，mRecorder = new MediaRecorder();
☯ 2.设置camera java对象到mediaRecorder中，mRecorder.setCamera(mCamera);
☯ 3.将preview surface对象设置给mediaRecorder,mRecorder.setPreviewDisplay(mPreviewHolder.getSurface());
☯ 4.设置音频源，mRecorder.setAudioSource(MediaRecorder.AudioSource.CAMCORDER);
☯ 5.设置视频源，mRecorder.setVideoSource(MediaRecorder.VideoSource.CAMERA);
☯ 6.设置录像帧大小和帧率，以及setOutputFile
☯ 8.准备工作，mRecorder.prepare();
☯ 9.启动mdiaRecorder,mRecorder.start();

##### 4.3、与MediaPlayerService相关的类接口之间的关系简介
##### 4.3.1、mediaRecorder何时与MediaPlayerService发送关系

![Alt text](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/camera.system/02-10-MediaRecorder.png)


``` cpp
/frameworks/base/media/java/android/media/MediaRecorder.java
MediaRecorder::MediaRecorder() : mSurfaceMediaSource(NULL)
{
    ALOGV("constructor");
    const sp<IMediaPlayerService>& service(getMediaPlayerService());
    if (service != NULL) {
        mMediaRecorder = service->createMediaRecorder();
    }
    if (mMediaRecorder != NULL) {
        mCurrentState = MEDIA_RECORDER_IDLE;
    }
    doCleanUp();
}
```
在jni中创建mediaRecorder对象时，其实在构造函数中偷偷的链接了mediaPlayerService，这也是Android习惯用的方法。获取到MediaPlayerService代理对象后，通过匿名binder获取mediaRecorder代理对象。 

##### 4.3.2、mediaPlayerService类和接口之间关系

![Alt text](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/camera.system/02-11-mediaPlayerService.png)


| 接口类型 |     接口说明|  
| :-------- |: --------| 
| virtual sp createMediaRecorder() = 0;	    |   创建mediaRecorder录视频服务对象的接口|  
| virtual sp create(const sp& client, int　audioSessionId = 0) = 0;		    |   创建mediaPlayer播放音乐服务对象的接口，播放音乐都是通过mediaPlayer对象播放的|  
| virtual status_t decode() = 0;	    |   音频解码器|  

##### 4.3.3、MediaRecorder类和接口之间关系

![Alt text](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/camera.system/02-12-IMediaRecorder.png)

mediaRecorder功能就是来录像的。其中MediaRecorder类中，包含了BpMediaRecorder代理对象引用。MediaRecorderClient本地对象驻留在mediaPlayService中。它的接口比较多，这里就列出我们今天关注的几个接口。其它接口查看源码吧 
详细介绍可以参考源码：frameworks/av/include/media/IMediaRecorder.h
| 接口类型 |     接口说明|  
| :-------- |: --------| 
| virtual status_t setCamera(const sp& camera,const sp& proxy) = 0;	    |   这个接口也是非常需要我们关注的，这里获取到了启动录像操作的本地对象(BnCameraRecordingProxy），并通过匿名binder通信方式，第二个参数就是本地对象.然后在startRecording时将帧监听对象注册到camera本地对象中了|  
| virtual status_t setPreviewSurface(const sp& surface) = 0;		    |   将preview预览surface对象设置给medaiRecorder，因为mediaRecorder也有一个camera本地client,所以这个surface对象最终还是会设置到cameraService用于显示。而录像的帧会在CameraService本地创建一个bufferQueue，具体下面会详细说明|  
| virtual status_t setListener(const sp& listener) = 0;	    |   这里一看就是设置监听对象，监听对象是jni中的JNIMediaRecorderListener对象，该对象可以回调MediaRecorder.java类中的postEventFromNative方法，将时间送到java层。其实MediaRecorder实现了BnMediaRecorderClient接口，即实现notify接口，那么这里其实将本地对象传到MediaRecorder本地的客户端对象中（本地对象拿到的就是代理对象了），参考代码片段1|  
| virtual status_t start() = 0;	    |   启动录像功能，函数追究下去和Camera关系不大了，这里就不细说了|  

##### 4.3.3.1、代码片段1

``` cpp
源码路径：frameworks/base/media/jni/android_media_MediaRecorder.cpp
// create new listener and give it to MediaRecorder
sp<JNIMediaRecorderListener> listener = new JNIMediaRecorderListener(env, thiz, weak_this);
mr->setListener(listener);
```

mediaRecorder jni接口回调java方法，通知上层native事件。

##### 4.3.3.2、代码片段2

``` cpp
[->\frameworks\base\media\jni\android_media_MediaRecorder.cpp]
static void android_media_MediaRecorder_setCamera(JNIEnv* env, jobject thiz, jobject camera)
{
// we should not pass a null camera to get_native_camera() call.
//这里检查camera是不是空的，显然不是空的。
    //这个地方需要好好研究一下，其中camera是java层的camera对象(即camera.java)
    //这里由java对象获取到camera应用端本地对象。
    sp<Camera> c = get_native_camera(env, camera, NULL);
    if (c == NULL) {
    // get_native_camera will throw an exception in this case
        return;
    }
    //获取mediaRecorder本地对象
    sp<MediaRecorder> mr = getMediaRecorder(env, thiz);
    //下面要特别注意，这里为什么传入的不是Camera对象而是c->remote()，当时琢磨
    //着，camera.cpp也没实现什么代理类的接口啊，不过后来在cameraBase类中发现
    //重载了remote()方法，该方法返回ICamera代理对象，呵呵。这样的话就会在
    //mediaRecorder中创建一个新的ICamera代理对象。并在mediaPlayerService中
    //创建了一个本地的Camera对象。
    //c->getRecordingProxy():获取camera本地对象实现的Recording本地对象。这里
    //调用setCamera设置到mediaRecorder本地对象中了(见代码片段３)
   process_media_recorder_call(env, mr->setCamera(c->remote(), c->getRecordingProxy()),
            "java/lang/RuntimeException", "setCamera failed.");
}
//camera端
sp<ICameraRecordingProxy> Camera::getRecordingProxy() {
    ALOGV("getProxy");
    return new RecordingProxy(this);
}
//看看下面RecordingProxy实现了BnCameraRecordingProxy接口，
//是个本地对象，水落石出了。
class RecordingProxy : public BnCameraRecordingProxy
    {
    public:
        RecordingProxy(const sp<Camera>& camera);

        // ICameraRecordingProxy interface
        virtual status_t startRecording(const sp<ICameraRecordingProxyListener>& listener);
        virtual void stopRecording();
        virtual void releaseRecordingFrame(const sp<IMemory>& mem);
    private:
    //这里的是mCamera已经不再是之前preview启动时对应的那个本地Camera对象
    //这是mediaRecorder重新创建的camera本地对象。
        sp<Camera>         mCamera;
    };
```
##### 4.3.3.3、代码片段3-setCamera本地实现

``` cpp
[->\frameworks\av\media\libmediaplayerservice\MediaRecorderClient.cpp]
status_t MediaRecorderClient::setCamera(const sp<ICamera>& camera,
                                        const sp<ICameraRecordingProxy>& proxy)
{
    ALOGV("setCamera");
    Mutex::Autolock lock(mLock);
    if (mRecorder == NULL) {
        ALOGE("recorder is not initialized");
        return NO_INIT;
    }
    return mRecorder->setCamera(camera, proxy);
}
//构造函数中可以看到创建了一个StagefrightRecorder对象，后续的其它操作
//都是通过mRecorder对象实现的
MediaRecorderClient::MediaRecorderClient(const sp<MediaPlayerService>& service, pid_t pid)
{
    ALOGV("Client constructor");
    mPid = pid;
    mRecorder = new StagefrightRecorder;
    mMediaPlayerService = service;
}
//StagefrightRecorder::setCamera实现
struct StagefrightRecorder : public MediaRecorderBase {}
status_t StagefrightRecorder::setCamera(const sp<ICamera> &camera,
                                        const sp<ICameraRecordingProxy> &proxy) {
//省去一些错误检查代码
    mCamera = camera;
    mCameraProxy = proxy;
    return OK;
}
```
最终ICamera,ICameraRecordingProxy代理对象都存放到StagefrightRecorder对应的成员变量中，看来猪脚就在这个类中。
##### 4.3.3.4、代码片段4

``` cpp
[->/frameworks/av/media/libstagefright/CameraSource.cpp]
status_t CameraSource::isCameraAvailable(
    const sp<ICamera>& camera, const sp<ICameraRecordingProxy>& proxy,
    int32_t cameraId, const String16& clientName, uid_t clientUid) {

    if (camera == 0) {
        mCamera = Camera::connect(cameraId, clientName, clientUid);
        if (mCamera == 0) return -EBUSY;
        mCameraFlags &= ~FLAGS_HOT_CAMERA;
    } else {
        // We get the proxy from Camera, not ICamera. We need to get the proxy
        // to the remote Camera owned by the application. Here mCamera is a
        // local Camera object created by us. We cannot use the proxy from
        // mCamera here.
        //根据ICamera代理对象重新创建Camera本地对象
        mCamera = Camera::create(camera);
        if (mCamera == 0) return -EBUSY;
        mCameraRecordingProxy = proxy;
        //目前还不清楚是什么标记，权且理解成支持热插拔标记
        mCameraFlags |= FLAGS_HOT_CAMERA;
        //代理对象绑定死亡通知对象
        mDeathNotifier = new DeathNotifier();
        // isBinderAlive needs linkToDeath to work.
        mCameraRecordingProxy->asBinder()->linkToDeath(mDeathNotifier);
    }
    mCamera->lock();
    return OK;
}
```
　由上面的类图之间的关系的，就知道mediaRecorder间接包含了cameaSource对象，这里为了简单直接要害代码。

☯ 1.在创建CameraSource对象时，会去检查一下Camera对象是否可用，可用的话就会根据传进来的代理对象重新创建Camera本地对象（注意这个时候Camera代理对象在mediaRecorder中）
☯ 2.然后保存RecordingProxy代理对象到mCameraRecordingProxy成员中，然后绑定死亡通知对象到RecordingProxy代理对象。

##### 4.3.3.5、代码片段5

``` cpp
[->/frameworks/av/media/libstagefright/CameraSource.cpp]
status_t CameraSource::startCameraRecording() {
    ALOGV("startCameraRecording");
    // Reset the identity to the current thread because media server owns the
    // camera and recording is started by the applications. The applications
    // will connect to the camera in ICameraRecordingProxy::startRecording.
    int64_t token = IPCThreadState::self()->clearCallingIdentity();
    status_t err;
    if (mNumInputBuffers > 0) {
        err = mCamera->sendCommand(
            CAMERA_CMD_SET_VIDEO_BUFFER_COUNT, mNumInputBuffers, 0);
    }
    err = OK;
    if (mCameraFlags & FLAGS_HOT_CAMERA) {//前面已经置位FLAGS_HOT_CAMERA，成立
        mCamera->unlock();
        mCamera.clear();
        //通过recording代理对象，直接启动camera本地端的recording
        if ((err = mCameraRecordingProxy->startRecording(
                new ProxyListener(this))) != OK) {
        }
    } else {
    }
    IPCThreadState::self()->restoreCallingIdentity(token);
    return err;
}
```
上面代码需要我们注意的是在启动startRecording()时，创建的监听对象new ProxyListener(this),该监听对象会传到Camera本地对象中。当帧可用时，用来通知mediaRecorder有帧可以使用了，赶紧编码吧。

##### 4.3.3.6、代码片段6-mediaRecorder注册帧可用监听对象

``` cpp
[->/frameworks/av/include/media/stagefright/CameraSource.h]
class ProxyListener: public BnCameraRecordingProxyListener {
    public:
        ProxyListener(const sp<CameraSource>& source);
        virtual void dataCallbackTimestamp(int64_t timestampUs, int32_t msgType,
                const sp<IMemory> &data);
    private:
        sp<CameraSource> mSource;
    };
//frameworks/av/camera/Camera.cpp
status_t Camera::RecordingProxy::startRecording(const sp<ICameraRecordingProxyListener>& listener)
{
    ALOGV("RecordingProxy::startRecording");
    mCamera->setRecordingProxyListener(listener);
    mCamera->reconnect();
    return mCamera->startRecording();
}
```
注册帧监听对象就是在启动Recording时注册，主要有下面几步：

☯ 1.使用setRecordingProxyListener接口，将监听对象设置给mRecordingProxyListener 成员。
☯ 2.重新和cameraService握手(preview停止时就会断开链接，在切换瞬间就断开了)
☯ 3.使用ICamera代理对象启动录像。
##### 4.4、阶段小结
到这里Camera如何使用medaiRecorder录像的基本流程已经清楚了，这里我画了一个流程图，大概包含下面9个流程。

![Alt text](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/camera.system/02-13-medaiRecorder-java-native.png)

☯ 过程1：上层点击了录像功能，或者录像preview模式下，会创建一个mediaRecorDer Java层对象。
☯ 过程2:java层mediaRecorder对象调用native_jni native_setup方法，创建一个native的mediaRecorder对象。创建的过程中连接mediaPlayerService,并通过匿名binder通信方式获取到一个mediaRecorderClient代理对象，并保存到mediaRecorder对象的成员变量mMediaRecorder中。
☯ 过程3:ava层的Camera对象传给mediaRecorder native层时，可以通过本地方法获取到Camera本地对象和ICamera代理对象。这里是获取ICamera代理对象和RecordingProxy本地对象
☯ 过程4:将ICamera代理对象和RecordingProxy本地对象传给在MedaiService本地端的MediaRecorderClient对象，这时ICamera是重新创建的ICamer代理对象，以及获取到RecordingProxy代理对象。
☯ 过程5：根据过程４获取到的新的ICamera代理对象和RecordingProxy代理对象，创建新的本地Camera对象Camera2，以及注册录像帧监听对象到Camera2中。
☯ 过程6：启动StartRecording
☯ 过程7:当录像帧可用时，通知驻留在MedaiRecorderClient中的Camera2本地对象收帧，于此同时Camera2又是通过注册的帧监听对象告知MediaClientClient对象。MediaClientClient对象拿到帧后进行录像编码。
☯ 过程8,过程９：通过回调函数，将一些消息发送给应用端。

##### 4.5、Camera video创建BufferQueue.

``` cpp
[->/frameworks/av/services/camera/libcameraservice/api1/client2/StreamingProcessor.cpp]
status_t StreamingProcessor::updateRecordingStream(const Parameters &params) {
    ATRACE_CALL();
    status_t res;
    Mutex::Autolock m(mMutex);
    sp<CameraDeviceBase> device = mDevice.promote();
    //----------------
    bool newConsumer = false;
    if (mRecordingConsumer == 0) {
        ALOGV("%s: Camera %d: Creating recording consumer with %zu + 1 "
                "consumer-side buffers", __FUNCTION__, mId, mRecordingHeapCount);
        // Create CPU buffer queue endpoint. We need one more buffer here so that we can
        // always acquire and free a buffer when the heap is full; otherwise the consumer
        // will have buffers in flight we'll never clear out.
        sp<IGraphicBufferProducer> producer;
        sp<IGraphicBufferConsumer> consumer;
        //创建bufferQueue，同时获取到生产者和消费者对象。
        BufferQueue::createBufferQueue(&producer, &consumer);
        //注意下面设置buffer的用处是GRALLOC_USAGE_HW_VIDEO_ENCODER，这个会在
        //mediaRecorder中使用到。
        mRecordingConsumer = new BufferItemConsumer(consumer,
                GRALLOC_USAGE_HW_VIDEO_ENCODER,
                mRecordingHeapCount + 1);
        mRecordingConsumer->setFrameAvailableListener(this);
        mRecordingConsumer->setName(String8("Camera2-RecordingConsumer"));
        mRecordingWindow = new Surface(producer);
        newConsumer = true;
        // Allocate memory later, since we don't know buffer size until receipt
    }
//更新部分代码，就不贴出来了－－－－
//注意下面video 录像buffer的像素格式是CAMERA2_HAL_PIXEL_FORMAT_OPAQUE
    if (mRecordingStreamId == NO_STREAM) {
        mRecordingFrameCount = 0;
        res = device->createStream(mRecordingWindow,
                params.videoWidth, params.videoHeight,
                CAMERA2_HAL_PIXEL_FORMAT_OPAQUE, &mRecordingStreamId);
    }

    return OK;
}
```
主要处理下面几件事情。

☯ 1.由于录像不需要显示，这里创建CameraService BufferQueue本地对象，这个时候获取到的生产者和消费者都是本地的，只有BufferQueue保存的有IGraphicBufferAlloc代理对象mAllocator，专门用来分配buffer。
☯ 2.由于StremingProcess.cpp中实现了FrameAvailableListener监听接口方法onFrameAvailable()。这里会通过setFrameAvailableListener方法注册到BufferQueue中。
☯ 3.根据生产者对象创建surface对象，并传给Camera3Device申请录像buffer.
☯ 4.如果参数有偏差或者之前已经创建过video Stream.这里会删除或者更新videoStream.如果压根没有创建VideoStream,直接创建VideoStream并根据参数更新流信息。

##### 4.6、何时录像帧可用
##### 4.6.1、onFrameAvailable()

``` cpp
[->/frameworks/av/services/camera/libcameraservice/api1/client2/StreamingProcessor.cpp]
void StreamingProcessor::onFrameAvailable(const BufferItem& /*item*/) {
    ATRACE_CALL();
    Mutex::Autolock l(mMutex);
    if (!mRecordingFrameAvailable) {
        mRecordingFrameAvailable = true;
        mRecordingFrameAvailableSignal.signal();
    }

}
```
当video buffer进行enqueue操作后,该函数会被调用。函数中可用发现，激活了StreamingProcessor主线程。

##### 4.6.2、StreamingProcessor线程loop

``` cpp
[->/frameworks/av/services/camera/libcameraservice/api1/client2/StreamingProcessor.cpp]
bool StreamingProcessor::threadLoop() {
    status_t res;
    {
        Mutex::Autolock l(mMutex);
        while (!mRecordingFrameAvailable) {
        //之前是在这里挂起的,现在有帧可用就会从这里唤醒。
            res = mRecordingFrameAvailableSignal.waitRelative(
                mMutex, kWaitDuration);
            if (res == TIMED_OUT) return true;
        }
        mRecordingFrameAvailable = false;
    }
    do {
        res = processRecordingFrame();//进一步处理。
    } while (res == OK);

    return true;
}
```
到这里发现，原来StreamingProcessor主线程只为录像服务，previewStream只是使用了它的几个方法而已。

##### 4.6.3、帧可用消息发送给Camera本地对象

``` cpp
[->\frameworks\av\services\camera\libcameraservice\api1\client2\StreamingProcessor.cpp]
status_t StreamingProcessor::processRecordingFrame() {
    ATRACE_CALL();
    status_t res;
    sp<Camera2Heap> recordingHeap;
    size_t heapIdx = 0;
    nsecs_t timestamp;
    sp<Camera2Client> client = mClient.promote();

    BufferItemConsumer::BufferItem imgBuffer;
    //取出buffer消费，就是拿给mediaRecorder编码
    res = mRecordingConsumer->acquireBuffer(&imgBuffer, 0);
    //----------------------------
    // Call outside locked parameters to allow re-entrancy from notification
    Camera2Client::SharedCameraCallbacks::Lock l(client->mSharedCameraCallbacks);
    if (l.mRemoteCallback != 0) {
    //调用Callback通知Camea本地对象。
        l.mRemoteCallback->dataCallbackTimestamp(timestamp,
                CAMERA_MSG_VIDEO_FRAME,
                recordingHeap->mBuffers[heapIdx]);
    } else {
        ALOGW("%s: Camera %d: Remote callback gone", __FUNCTION__, mId);
    }
    return OK;
```
之前我们已经知道Camera运行时存在类型为ICameraClient的两个对象,其中一个代理对象保存在CameraService中，本地对象保存的Camera本地对象中。这里代理对象通知本地对象取帧了。注意这里消息发送的是“CAMERA_MSG_VIDEO_FRAME”。

##### 4.6.4、Camera本地对象转发消息给mediaRecorder.

``` cpp
[->/frameworks/av/camera/Camera.cpp]
void Camera::dataCallbackTimestamp(nsecs_t timestamp, int32_t msgType, const sp<IMemory>& dataPtr)
{
    // If recording proxy listener is registered, forward the frame and return.
    // The other listener (mListener) is ignored because the receiver needs to
    // call releaseRecordingFrame.
    sp<ICameraRecordingProxyListener> proxylistener;
    {
    //这里mRecordingProxyListener就是mediaRecorder注册过来的监听代理对象
        Mutex::Autolock _l(mLock);
        proxylistener = mRecordingProxyListener;
    }
    if (proxylistener != NULL) {
    //这里就把buffer送到了mediaRecorder中进行编码
        proxylistener->dataCallbackTimestamp(timestamp, msgType, dataPtr);
        return;
    }
  //---------省略代码
}
```
到这里Camera本地对象就会调用mediaRecorder注册来的帧监听对象。前面我们已经做了那么长的铺垫，我想应该可以理解了。好了,mediaRecorder有饭吃了。

##### 4.7、总结
1.一开始我自以为preview和Video使用同一个camera本地对象，看了代码发现，原来是不同的对象。
2.预览的BufferQueue是在CameraService中创建的，和surfaceFlinger没有关系，只是保留了IGraphicBufferAlloc代理对象mAllocator，用于分配buffer.
3.之匿名binder没有理解透彻，以为只有传递本地对象才能使用writeStrongBinder()接口保存binder对象，同时在使用端使用readStrongBinder()就可以获取到代理对象了。其实也可以传递代理对象，只不过代码会走另外一套逻辑，在kernel中重新创建一个binder_ref索引对象返回给另一端。如下mediaRecorder设置camera时就是传递的ICamera代理对象。

``` cpp
[->/frameworks/av/media/libmedia/IMediaRecorder.cpp]
    status_t setCamera(const sp<ICamera>& camera, const sp<ICameraRecordingProxy>& proxy)
    {
        ALOGV("setCamera(%p,%p)", camera.get(), proxy.get());
        Parcel data, reply;
        data.writeInterfaceToken(IMediaRecorder::getInterfaceDescriptor());
        //camera->asBinder()是ICamera代理对象
        data.writeStrongBinder(camera->asBinder());
        data.writeStrongBinder(proxy->asBinder());
        remote()->transact(SET_CAMERA, data, &reply);
        return reply.readInt32();
    }
```

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



