---
title:  Android 10 Display System源码分析（7）：Native Surface创建 && SurfaceFlinger合成流程分析（Android 10.0 && Kernel 4.15）
cover: https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/post.cover.pictures/bing-wallpaper-2018.04.28.jpg
categories: 
  - Display
tags:
  - Android
  - Linux
  - Graphics
toc: true
abbrlink: 20210910
date: 2021-09-10 09:25:00
---

我们首先去除掉系统的：
``` cpp
rk3399_Android10:/system/bin # rm -rf bootanimation
rk3399_Android10:/product/priv-app # rm -rf Settings/
rk3399_Android10:/product/priv-app # rm -rf SystemUI/
rk3399_Android10:/product/priv-app # rm -rf Launcher3QuickStep/
```
然后等待开机完成SurfaceFlinger_test_red，抓取Log。setprop vendor.dump true。抓取的帧会按数字排列，还带分辨率参数。adb pull /data/dump/。抓到的bin文件可以用软件7yuv打开查看，格式设定为RGBA8888
Layer test#0渲染图：
![](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/Android10.Display.7/SurfaceFlinger_test_red_take_picture_Test.jpg)
FrameBuffer渲染图：
![](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/Android10.Display.7/SurfaceFlinger_test_red_take_picture_FB.jpg)
效果图：
![](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/Android10.Display.7/SurfaceFlinger_test_red_take_picture.jpg)
#### （一）、Native Surface创建过程

##### 1.1.0 、Native Surface创建步骤

``` cpp
X:\packages\apps\SurfaceFlingerTestsRed\Transaction_test_red.cpp
    void SetUp() override {
        mClient = new SurfaceComposerClient;
        ASSERT_EQ(NO_ERROR, mClient->initCheck()) << "failed to create SurfaceComposerClient";

        ASSERT_NO_FATAL_FAILURE(SetUpDisplay());
    }

    sp<SurfaceControl> createLayer(const char* name, uint32_t width, uint32_t height,
                                   uint32_t flags = 0) {
        auto layer =
                mClient->createSurface(String8(name), width, height, PIXEL_FORMAT_RGBA_8888, flags);
        EXPECT_NE(nullptr, layer.get()) << "failed to create SurfaceControl";

        status_t error = Transaction()
                                 .setLayerStack(layer, mDisplayLayerStack)
                                 .setLayer(layer, mLayerZBase)
                                 .apply();
        if (error != NO_ERROR) {
            ADD_FAILURE() << "failed to initialize SurfaceControl";
            layer.clear();
        }

        return layer;
    }
 
```


[->SurfaceComposerClient.cpp]

``` cpp
X:\frameworks\native\libs\gui\SurfaceComposerClient.cpp
sp<SurfaceControl> SurfaceComposerClient::createSurface(const String8& name, uint32_t w, uint32_t h,
                                                        PixelFormat format, uint32_t flags,
                                                        SurfaceControl* parent,
                                                        LayerMetadata metadata) {
    sp<SurfaceControl> s;
    createSurfaceChecked(name, w, h, format, &s, flags, parent, std::move(metadata));
    return s;
}
status_t SurfaceComposerClient::createSurfaceChecked(const String8& name, uint32_t w, uint32_t h,
                                                     PixelFormat format,
                                                     sp<SurfaceControl>* outSurface, uint32_t flags,
                                                     SurfaceControl* parent,
                                                     LayerMetadata metadata) {
    sp<SurfaceControl> sur;
    status_t err = mStatus;

    if (mStatus == NO_ERROR) {
        sp<IBinder> handle;
        sp<IBinder> parentHandle;
        sp<IGraphicBufferProducer> gbp;

        if (parent != nullptr) {
            parentHandle = parent->getHandle();
        }

        err = mClient->createSurface(name, w, h, format, flags, parentHandle, std::move(metadata),
                                     &handle, &gbp);
        ALOGE_IF(err, "SurfaceComposerClient::createSurface error %s", strerror(-err));
        if (err == NO_ERROR) {
            *outSurface = new SurfaceControl(this, handle, gbp, true /* owned */);
        }
    }
    return err;
}
```

SurfaceComposerClient将Surface创建请求转交给保存在其成员变量中的Bp SurfaceComposerClient对象来完成。

``` cpp
X:\frameworks\native\libs\gui\ISurfaceComposerClient.cpp
    status_t createSurface(const String8& name, uint32_t width, uint32_t height, PixelFormat format,
                           uint32_t flags, const sp<IBinder>& parent, LayerMetadata metadata,
                           sp<IBinder>* handle, sp<IGraphicBufferProducer>* gbp) override {
        return callRemote<decltype(&ISurfaceComposerClient::createSurface)>(Tag::CREATE_SURFACE,
                                                                            name, width, height,
                                                                            format, flags, parent,
                                                                            std::move(metadata),
                                                                            handle, gbp);
    }


IMPLEMENT_META_INTERFACE(SurfaceComposerClient, "android.ui.ISurfaceComposerClient");

// ----------------------------------------------------------------------

status_t BnSurfaceComposerClient::onTransact(uint32_t code, const Parcel& data, Parcel* reply,
                                             uint32_t flags) {
    if (code < IBinder::FIRST_CALL_TRANSACTION || code > static_cast<uint32_t>(Tag::LAST)) {
        return BBinder::onTransact(code, data, reply, flags);
    }
    auto tag = static_cast<Tag>(code);
    switch (tag) {
        case Tag::CREATE_SURFACE:
            return callLocal(data, reply, &ISurfaceComposerClient::createSurface);
        case Tag::CREATE_WITH_SURFACE_PARENT:
            return callLocal(data, reply, &ISurfaceComposerClient::createWithSurfaceParent);
        case Tag::CLEAR_LAYER_FRAME_STATS:
            return callLocal(data, reply, &ISurfaceComposerClient::clearLayerFrameStats);
        case Tag::GET_LAYER_FRAME_STATS:
            return callLocal(data, reply, &ISurfaceComposerClient::getLayerFrameStats);
    }
}

```

``` cpp
X:\frameworks\native\services\surfaceflinger\Client.cpp
status_t Client::createSurface(const String8& name, uint32_t w, uint32_t h, PixelFormat format,
                               uint32_t flags, const sp<IBinder>& parentHandle,
                               LayerMetadata metadata, sp<IBinder>* handle,
                               sp<IGraphicBufferProducer>* gbp) {
    // We rely on createLayer to check permissions.
	ALOGI("zjj.rk3399.SF createSurface %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

    return mFlinger->createLayer(name, this, w, h, format, flags, std::move(metadata), handle, gbp,
                                 parentHandle);
}

```
通过Binder通信进入SurfaceFlinger创建Layer。


``` cpp
status_t SurfaceFlinger::createLayer(const String8& name, const sp<Client>& client, uint32_t w,
                                     uint32_t h, PixelFormat format, uint32_t flags,
                                     LayerMetadata metadata, sp<IBinder>* handle,
                                     sp<IGraphicBufferProducer>* gbp,
                                     const sp<IBinder>& parentHandle,
                                     const sp<Layer>& parentLayer) {
	ALOGI("zjj.rk3399.SF createLayer %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
    ......
    status_t result = NO_ERROR;

    sp<Layer> layer;

    String8 uniqueName = getUniqueLayerName(name);

    bool primaryDisplayOnly = false;
    ......
    switch (flags & ISurfaceComposerClient::eFXSurfaceMask) {
        case ISurfaceComposerClient::eFXSurfaceBufferQueue:
            result = createBufferQueueLayer(client, uniqueName, w, h, flags, std::move(metadata),
                                            format, handle, gbp, &layer);

            break;
        case ISurfaceComposerClient::eFXSurfaceBufferState:
            result = createBufferStateLayer(client, uniqueName, w, h, flags, std::move(metadata),
                                            handle, &layer);
            break;
        case ISurfaceComposerClient::eFXSurfaceColor:
            ......
            result = createColorLayer(client, uniqueName, w, h, flags, std::move(metadata), handle,
                                      &layer);
            break;
        case ISurfaceComposerClient::eFXSurfaceContainer:
            ......
            result = createContainerLayer(client, uniqueName, w, h, flags, std::move(metadata),
                                          handle, &layer);
            break;
        default:
            result = BAD_VALUE;
            break;
    }
    ......
    bool addToCurrentState = callingThreadHasUnscopedSurfaceFlingerAccess();
    result = addClientLayer(client, *handle, *gbp, layer, parentHandle, parentLayer,
                            addToCurrentState);
    ......
    mInterceptor->saveSurfaceCreation(layer);

    setTransactionFlags(eTransactionNeeded);
    return result;
}

```

  SurfaceFlinger根据标志位创建对应类型的Surface，当前系统定义了4种类型的Layer:
  [->ISurfaceComposerClient.h]

``` cpp
   // flags for createSurface()
    enum { // (keep in sync with Surface.java)
        ......

        eFXSurfaceBufferQueue = 0x00000000,
        eFXSurfaceColor = 0x00020000,
        eFXSurfaceBufferState = 0x00040000,
        eFXSurfaceContainer = 0x00080000,
        ......
    };
```

[->SurfaceFlinger.cpp]

``` cpp
status_t SurfaceFlinger::createBufferQueueLayer(const sp<Client>& client, const String8& name,
                                                uint32_t w, uint32_t h, uint32_t flags,
                                                LayerMetadata metadata, PixelFormat& format,
                                                sp<IBinder>* handle,
                                                sp<IGraphicBufferProducer>* gbp,
                                                sp<Layer>* outLayer) {
    // initialize the surfaces
	ALOGI("zjj.rk3399.SF createBufferQueueLayer %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

    switch (format) {
    case PIXEL_FORMAT_TRANSPARENT:
    case PIXEL_FORMAT_TRANSLUCENT:
        format = PIXEL_FORMAT_RGBA_8888;
        break;
    case PIXEL_FORMAT_OPAQUE:
        format = PIXEL_FORMAT_RGBX_8888;
        break;
    }

    sp<BufferQueueLayer> layer = getFactory().createBufferQueueLayer(
            LayerCreationArgs(this, client, name, w, h, flags, std::move(metadata)));
    status_t err = layer->setDefaultBufferProperties(w, h, format);
    if (err == NO_ERROR) {
        *handle = layer->getHandle();
        *gbp = layer->getProducer();
        *outLayer = layer;
    }

    ALOGE_IF(err, "createBufferQueueLayer() failed (%s)", strerror(-err));
    return err;
}
```

在SurfaceFlinger服务端为应用程序创建的Surface创建对应的Layer对象。

``` cpp
frameworks/native/services/surfaceflinger/SurfaceFlingerFactory.cpp
        sp<BufferQueueLayer> createBufferQueueLayer(const LayerCreationArgs& args) override {
			ALOGI("zjj.rk3399.SF createBufferQueueLayer %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

            return new BufferQueueLayer(args);
        }
X:\frameworks\native\services\surfaceflinger\BufferQueueLayer.cpp
BufferQueueLayer::BufferQueueLayer(const LayerCreationArgs& args) : BufferLayer(args) {}
void BufferQueueLayer::onFirstRef() {
    BufferLayer::onFirstRef();

    // Creates a custom BufferQueue for SurfaceFlingerConsumer to use
    sp<IGraphicBufferProducer> producer;
    sp<IGraphicBufferConsumer> consumer;
    BufferQueue::createBufferQueue(&producer, &consumer, true);
    mProducer = new MonitoredProducer(producer, mFlinger, this);
    {
        // Grab the SF state lock during this since it's the only safe way to access RenderEngine
        Mutex::Autolock lock(mFlinger->mStateLock);
        mConsumer =
                new BufferLayerConsumer(consumer, mFlinger->getRenderEngine(), mTextureName, this);
    }
    mConsumer->setConsumerUsageBits(getEffectiveUsage(0));
    mConsumer->setContentsChangedListener(this);
    mConsumer->setName(mName);

    // BufferQueueCore::mMaxDequeuedBufferCount is default to 1
    if (!mFlinger->isLayerTripleBufferingDisabled()) {
        mProducer->setMaxDequeuedBufferCount(2);
    }

    if (const auto display = mFlinger->getDefaultDisplayDevice()) {
        updateTransformHint(display);
    }
}
```
创建BufferQueueLayer 会首先调用onFirstRef()函数，创建IGraphicBufferProducer，IGraphicBufferConsumer，BufferQueue对象。设置最大setMaxDequeuedBufferCount为2。BufferQueueLayer继承BufferLayer，看看BufferLayer构造函数。

获取CompositionEngine创造对应layer，这个后面用于渲染作用。
看看Log：

```bash
10-28 09:41:00.086   179   523 V BufferLayerConsumer: [unnamed-179-2] BufferLayerConsumer
10-28 09:41:00.086   179   523 V ConsumerBase: [unnamed-179-2] setFrameAvailableListener
10-28 09:41:00.086   179   523 V BufferQueueProducer: [] setMaxDequeuedBufferCount: maxDequeuedBuffers = 2
10-28 09:41:00.086   179   523 I BufferLayerConsumer: zjj.rk3399.SF setDefaultBufferSize frameworks/native/services/surfaceflinger/BufferLayerConsumer.cpp setDefaultBufferSize 86 

10-28 09:41:00.086   179   523 I SurfaceFlinger: zjj.rk3399.SF addClientLayer frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp addClientLayer 3685 
```

``` cpp
X:\frameworks\native\services\surfaceflinger\BufferLayer.cpp
BufferLayer::BufferLayer(const LayerCreationArgs& args)
      : Layer(args),
        mTextureName(args.flinger->getNewTexture()),
        mCompositionLayer{mFlinger->getCompositionEngine().createLayer(
                compositionengine::LayerCreationArgs{this})} {
    ALOGV("Creating Layer %s", args.name.string());
	ALOGD_CALLSTACK("BufferLayer");

    mPremultipliedAlpha = !(args.flags & ISurfaceComposerClient::eNonPremultiplied);

    mPotentialCursor = args.flags & ISurfaceComposerClient::eCursorWindow;
    mProtectedByApp = args.flags & ISurfaceComposerClient::eProtectedByApp;

    // Use rk ashmem -------
    if ( NULL == g_gralloc )
    {
        int ret = hw_get_module(GRALLOC_HARDWARE_MODULE_ID,
                      (const hw_module_t **)&g_gralloc);
        if (ret) {
            ALOGE("Failed to open gralloc module %d", ret);
        }
    }
    // Use rk ashmem -------
}

```

看看Log：

``` bash
10-28 09:41:00.066   179   523 I SurfaceFlinger: zjj.rk3399.SF createSurface frameworks/native/services/surfaceflinger/Client.cpp createSurface 80 
10-28 09:41:00.066   179   523 I SurfaceFlinger: zjj.rk3399.SF createLayer frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp createLayer 4355 
10-28 09:41:00.066   179   523 I SurfaceFlinger: zjj.rk3399.SF createBufferQueueLayer frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp createBufferQueueLayer 4476 
10-28 09:41:00.066   179   523 I SurfaceFlinger: zjj.rk3399.SF createBufferQueueLayer frameworks/native/services/surfaceflinger/SurfaceFlingerFactory.cpp createBufferQueueLayer 132 
10-28 09:41:00.066   179   523 I CompositionEngine: zjj.rk3399.SF createLayer frameworks/native/services/surfaceflinger/CompositionEngine/src/CompositionEngine.cpp createLayer 43 
10-28 09:41:00.066   179   523 V BufferLayer: Creating Layer test#0
10-28 09:41:00.066   179   523 D BufferLayer: BufferLayer

10-28 09:41:00.086   179   523 D BufferLayer:   #00 pc 000000000007d14c  /system/lib64/libsurfaceflinger.so (android::BufferLayer::BufferLayer(android::LayerCreationArgs const&)+300)
10-28 09:41:00.086   179   523 D BufferLayer:   #01 pc 00000000000830b8  /system/lib64/libsurfaceflinger.so (android::BufferQueueLayer::BufferQueueLayer(android::LayerCreationArgs const&)+88)
10-28 09:41:00.086   179   523 D BufferLayer:   #02 pc 00000000001160ac  /system/lib64/libsurfaceflinger.so (_ZZN7android14surfaceflinger20createSurfaceFlingerEvEN7Factory22createBufferQueueLayerERKNS_17LayerCreationArgsE$f8e1ddd5c1a01af33e02be699775c0a6+84)
10-28 09:41:00.086   179   523 D BufferLayer:   #03 pc 00000000000e9314  /system/lib64/libsurfaceflinger.so (android::SurfaceFlinger::createLayer(android::String8 const&, android::sp<android::Client> const&, unsigned int, unsigned int, int, unsigned int, android::LayerMetadata, android::sp<android::IBinder>*, android::sp<android::IGraphicBufferProducer>*, android::sp<android::IBinder> const&, android::sp<android::Layer> const&)+828)
10-28 09:41:00.086   179   523 D BufferLayer:   #04 pc 000000000008bf88  /system/lib64/libsurfaceflinger.so (android::Client::createSurface(android::String8 const&, unsigned int, unsigned int, int, unsigned int, android::sp<android::IBinder> const&, android::LayerMetadata, android::sp<android::IBinder>*, android::sp<android::IGraphicBufferProducer>*)+232)
10-28 09:41:00.086   179   523 D BufferLayer:   #05 pc 00000000000ac2a8  /system/lib64/libgui.so (_ZN7android15SafeBnInterfaceINS_22ISurfaceComposerClientEE12MethodCallerIJNSt3__15tupleIJRKNS_7String8EjjijRKNS_2spINS_22IGraphicBufferProducerEEENS_13LayerMetadataEPNS9_INS_7IBinderEEEPSB_EEEEE10callHelperIS2_MS1_FiS8_jjijSD_SE_SH_SI_ENS5_IJS6_jjijSB_SE_SG_SB_EEEJLm0ELm1ELm2ELm3ELm4ELm5ELm6ELm7ELm8EEEEiPT_T0_PT1_NS4_16integer_sequenceImJXspT2_EEEE+136)
10-28 09:41:00.086   179   523 D BufferLayer:   #06 pc 00000000000aa7ac  /system/lib64/libgui.so (_ZN7android15SafeBnInterfaceINS_22ISurfaceComposerClientEE9callLocalIMS1_FiRKNS_7String8EjjijRKNS_2spINS_7IBinderEEENS_13LayerMetadataEPS9_PNS7_INS_22IGraphicBufferProducerEEEEEEiRKNS_6ParcelEPSJ_T_+172)
10-28 09:41:00.086   179   523 D BufferLayer:   #07 pc 000000000004c6b8  /system/lib64/libbinder.so (android::BBinder::transact(unsigned int, android::Parcel const&, android::Parcel*, unsigned int)+136)
10-28 09:41:00.086   179   523 D BufferLayer:   #08 pc 0000000000058d08  /system/lib64/libbinder.so (android::IPCThreadState::executeCommand(int)+984)
10-28 09:41:00.086   179   523 D BufferLayer:   #09 pc 000000000005887c  /system/lib64/libbinder.so (android::IPCThreadState::getAndExecuteCommand()+156)
10-28 09:41:00.086   179   523 D BufferLayer:   #10 pc 0000000000058fc4  /system/lib64/libbinder.so (android::IPCThreadState::joinThreadPool(bool)+60)
10-28 09:41:00.086   179   523 D BufferLayer:   #11 pc 000000000007f578  /system/lib64/libbinder.so (android::PoolThread::threadLoop()+24)
10-28 09:41:00.086   179   523 D BufferLayer:   #12 pc 00000000000137a4  /system/lib64/libutils.so (android::Thread::_threadLoop(void*)+284)
10-28 09:41:00.086   179   523 D BufferLayer:   #13 pc 00000000000e230c  /apex/com.android.runtime/lib64/bionic/libc.so (__pthread_start(void*)+36)
10-28 09:41:00.086   179   523 D BufferLayer:   #14 pc 0000000000083d98  /apex/com.android.runtime/lib64/bionic/libc.so (__start_thread+64)


```

#### （二）、Android SurfaceFlinger内部机制

##### 2.1.0 、BufferQueue介绍
BufferQueue 类是 Android 中所有图形处理操作的核心。它的是将生成图形数据缓冲区的一方（生产者Producer）连接到接受数据以进行显示或进一步处理的一方（消费者Consumer）。几乎所有在系统中移动图形数据缓冲区的内容都依赖于 BufferQueue。
从上图APP与SurfaceFlinger交互中可以看出，BufferQueue内部维持着64个BufferSlot，每一个BufferSlot内部有一个GraphicBuffer指向分配的Graphic Buffer。
![](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/Android10.Display.7/Android-Graphics-SurfaceFlinger-BufferQueue.png)


先来看一下图中几个状态代表的含义：

``` cpp
    frameworks/native/include/gui/BufferSlot.h

    // A buffer can be in one of five states, represented as below:
    //
    //         | mShared | mDequeueCount | mQueueCount | mAcquireCount |
    // --------|---------|---------------|-------------|---------------|
    // FREE    |  false  |       0       |      0      |       0       |
    // DEQUEUED|  false  |       1       |      0      |       0       |
    // QUEUED  |  false  |       0       |      1      |       0       |
    // ACQUIRED|  false  |       0       |      0      |       1       |
    // SHARED  |  true   |      any      |     any     |      any      |
```
**FREE :** 
FREE表示缓冲区可由生产者（Producer）DEQUEUED出列。 该BufferSlot由BufferQueue“拥有”。 它转换到DEQUEUED
当调用dequeueBuffer时。

**DEQUEUED：**
DEQUEUED表示缓冲区已经被生产者（Producer）出列，但是尚未queued 或canceled。生产者（Producer）可以修改缓冲区的内容一旦相关的释放围栏被发信号通知。BufferSlot由Producer“拥有”。 它可以转换到QUEUED（通过
queueBuffer或者attachBuffer）或者返回FREE（通过cancelBuffer或者detachBuffer）。

**QUEUED：**
QUEUED表示缓冲区已经被生产者（Producer）填充排队等待消费者（Consumer）使用。 缓冲区内容可能被继续
  修改在有限的时间内，所以内容不能被访问，直到关联的栅栏fence发信号。 该BufferSlot由BufferQueue“拥有”。 它
  可以转换为ACQUIRED（通过acquireBuffer）或FREE（如果是另一个缓冲区以异步模式排队）。

**ACQUIRED：**
ACQUIRED表示缓冲区已被消费者（Consumer）获取。 如与QUEUED，内容不能被消费者访问，直到
获得栅栏fence信号。 BufferSlot由Consumer“拥有”。 它当releaseBuffer（或detachBuffer）被调用时转换为FREE。 一个
分离的缓冲区也可以通过attachBuffer进入ACQUIRED状态。

**SHARED：** 
 SHARED表示此缓冲区正在共享缓冲区中使用模式。 它可以同时在其他State的任何组合，
除了FREE （因为这不包括在任何其他State）。 它可以也可以出列，排队或多次获得。

**简单描述一下状态转换过程：**

1、首先生产者dequeue过来一块Buffer，此时该buffer的状态为DEQUEUED，所有者为PRODUCER，生产者可以填充数据了。在没有dequeue操作时，buffer的状态为free,所有者为BUFFERQUEUE。

2、生产者填充完数据后,进行queue操作，此时buffer的状态由DEQUEUED->QUEUED的转变，buffer所有者也变成了BufferQueue了。 

3、上面已经通知消费者去拿buffer了，这个时候消费者就进行acquire操作将buffer拿过来，此时buffer的状态由QUEUED->ACQUIRED转变，buffer的拥有者由BufferQueue变成Consumer。

4、当消费者已经消费了这块buffer(已经合成，已经编码等)，就进行release操作释放buffer,将buffer归还给BufferQueue,buffer状态由ACQUIRED变成FREE.buffer拥有者由Consumer变成BufferQueue.
###### 2.1.1、生产者Producer
生产者Producer实现IGraphicBufferProducer的接口，在实际运作过程中，应用（Client端）存在代理端BpGraphicBufferProducer，SurfaceFlinger（Server端）存在Native端BnGraphicBufferProducer。生产者代理端Bp通过Binder通信，不断的dequeueBuffer和queueBuffer操作，Native端同样响应这些操作请求，这样buffer就转了起来了。 
![](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/Android10.Display.7/Android-Graphics-SurfaceFlinger-IGraphicsBufferProducer.png)


这里介绍几个非常重要的函数：
**1、requestBuffer**
requestBuffer为给定的索引请求一个新的Buffer。 服务器（即IGraphicBufferProducer实现）分配新创建的Buffer到给定的BufferSlot槽索引，并且客户端可以镜像slot->Buffer映射，这样就没有必要传输一个GraphicBuffer用于每个出队操作。

``` cpp
    // requestBuffer requests a new buffer for the given index. The server (i.e.
    // the IGraphicBufferProducer implementation) assigns the newly created
    // buffer to the given slot index, and the client is expected to mirror the
    // slot->buffer mapping so that it's not necessary to transfer a
    // GraphicBuffer for every dequeue operation.
    //
    // The slot must be in the range of [0, NUM_BUFFER_SLOTS).
    virtual status_t requestBuffer(int slot, sp<GraphicBuffer>* buf) = 0;
```
**2、dequeueBuffer**
dequeueBuffer请求一个新的Buffer Slot供客户端使用。 插槽的所有权被转移到客户端，这意味着服务器不会使用与该插槽关联的缓冲区的内容。
``` cpp
    // dequeueBuffer requests a new buffer slot for the client to use. Ownership
    // of the slot is transfered to the client, meaning that the server will not
    // use the contents of the buffer associated with that slot.
    //
    virtual status_t dequeueBuffer(int* slot, sp<Fence>* fence, uint32_t w,
            uint32_t h, PixelFormat format, uint32_t usage) = 0;
```
**3、detachBuffer**
detachBuffer尝试删除给定buffer 的所有权插槽从buffer queue。 如果这个请求成功，该slot将会被free，并且将无法从这个接口获得缓冲区。释放的插槽将保持未分配状态，直到被选中为止在dequeueBuffer中保存一个新分配的缓冲区，或者附加一个缓冲区到插槽。 缓冲区必须已经被取出，并且调用者必须已经拥有sp <GraphicBuffer>（即必须调用requestBuffer）
``` cpp
    // detachBuffer attempts to remove all ownership of the buffer in the given
    // slot from the buffer queue. If this call succeeds, the slot will be
    // freed, and there will be no way to obtain the buffer from this interface.
    // The freed slot will remain unallocated until either it is selected to
    // hold a freshly allocated buffer in dequeueBuffer or a buffer is attached
    // to the slot. The buffer must have already been dequeued, and the caller
    // must already possesses the sp<GraphicBuffer> (i.e., must have called
    // requestBuffer).
    //
    virtual status_t detachBuffer(int slot) = 0;
```
**4、attachBuffer**
attachBuffer尝试将缓冲区的所有权转移给缓冲区队列。 如果这个调用成功，就好像这个缓冲区已经出队一样从返回的插槽号码。 因此，如果连接，这个调用将失败这个缓冲区会导致很多的缓冲区同时出队。
``` cpp
    // attachBuffer attempts to transfer ownership of a buffer to the buffer
    // queue. If this call succeeds, it will be as if this buffer was dequeued
    // from the returned slot number. As such, this call will fail if attaching
    // this buffer would cause too many buffers to be simultaneously dequeued.
    //
    virtual status_t attachBuffer(int* outSlot,
            const sp<GraphicBuffer>& buffer) = 0;
```

###### 2.1.2、消费者Consumer
![](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/Android10.Display.7/Android-Graphics-SurfaceFlinger-IGraphicsBufferConsumer.png)


这里介绍几个非常重要的函数：
**1、acquireBuffer**
acquireBuffer尝试获取下一个未决缓冲区的所有权BufferQueue。 如果没有缓冲区等待，则返回NO_BUFFER_AVAILABLE。 如果缓冲区被成功获取，有关缓冲区的信息将在BufferItem中返回。

``` cpp
    // acquireBuffer attempts to acquire ownership of the next pending buffer in
    // the BufferQueue.  If no buffer is pending then it returns
    // NO_BUFFER_AVAILABLE.  If a buffer is successfully acquired, the
    // information about the buffer is returned in BufferItem.
    //
    virtual status_t acquireBuffer(BufferItem* buffer, nsecs_t presentWhen,
            uint64_t maxFrameNumber = 0) = 0;
```
**2、releaseBuffer**
releaseBuffer从消费者释放一个BufferSlot回到BufferQueue。 这可以在缓冲区的内容仍然存在时完成被访问。 栅栏将在缓冲区不再正在使用时发出信号。 frameNumber用于标识返回的确切缓冲区。
``` cpp
    // releaseBuffer releases a buffer slot from the consumer back to the
    // BufferQueue.  This may be done while the buffer's contents are still
    // being accessed.  The fence will signal when the buffer is no longer
    // in use. frameNumber is used to indentify the exact buffer returned.
    //
    virtual status_t releaseBuffer(int buf, uint64_t frameNumber,
            EGLDisplay display, EGLSyncKHR fence,
            const sp<Fence>& releaseFence) = 0;
```
**3、detachBuffer**
detachBuffer尝试删除给定缓冲区的所有权插槽从缓冲区队列。 如果这个请求成功，该插槽将会是释放，并且将无法从这个接口获得缓冲区。释放的插槽将保持未分配状态，直到被选中为止在dequeueBuffer中保存一个新分配的缓冲区，或者附加一个缓冲区到slot。 缓冲区必须已被acquired。
``` cpp
    // detachBuffer attempts to remove all ownership of the buffer in the given
    // slot from the buffer queue. If this call succeeds, the slot will be
    // freed, and there will be no way to obtain the buffer from this interface.
    // The freed slot will remain unallocated until either it is selected to
    // hold a freshly allocated buffer in dequeueBuffer or a buffer is attached
    // to the slot. The buffer must have already been acquired.
    //
    virtual status_t detachBuffer(int slot) = 0;
```
**4、attachBuffer**
attachBuffer尝试将缓冲区的所有权转移给缓冲区队列。 如果这个调用成功，就好像这个缓冲区被获取了一样从返回的插槽号码。 因此，如果连接，这个调用将失败这个缓冲区会导致太多的缓冲区被同时acquired。
``` cpp
    // attachBuffer attempts to transfer ownership of a buffer to the buffer
    // queue. If this call succeeds, it will be as if this buffer was acquired
    // from the returned slot number. As such, this call will fail if attaching
    // this buffer would cause too many buffers to be simultaneously acquired.
    //
    virtual status_t attachBuffer(int *outSlot,
            const sp<GraphicBuffer>& buffer) = 0;
```

#### （三）、Surface管理图形缓冲区- (lock) Buffer && (unlockAndPost) Buffer的过程

##### 3.1.0、APP申请(lock)Buffer的过程
我们上边分析到了申请图形缓冲区，用到了Surface的lock函数，我们继续查看。
[->Surface.cpp]

``` cpp
// ----------------------------------------------------------------------------
X:\frameworks\native\libs\gui\Surface.cpp
status_t Surface::lock(
        ANativeWindow_Buffer* outBuffer, ARect* inOutDirtyBounds)
{
    if (mLockedBuffer != nullptr) {
        ALOGE("Surface::lock failed, already locked");
        return INVALID_OPERATION;
    }

    if (!mConnectedToCpu) {
        int err = Surface::connect(NATIVE_WINDOW_API_CPU);
        ......
        setUsage(GRALLOC_USAGE_SW_READ_OFTEN | GRALLOC_USAGE_SW_WRITE_OFTEN);
    }

    ANativeWindowBuffer* out;
    int fenceFd = -1;
    status_t err = dequeueBuffer(&out, &fenceFd);
    ALOGE_IF(err, "dequeueBuffer failed (%s)", strerror(-err));
    if (err == NO_ERROR) {
        sp<GraphicBuffer> backBuffer(GraphicBuffer::getSelf(out));
        const Rect bounds(backBuffer->width, backBuffer->height);

        Region newDirtyRegion;
        if (inOutDirtyBounds) {
            newDirtyRegion.set(static_cast<Rect const&>(*inOutDirtyBounds));
            newDirtyRegion.andSelf(bounds);
        } else {
            newDirtyRegion.set(bounds);
        }

        // figure out if we can copy the frontbuffer back
        const sp<GraphicBuffer>& frontBuffer(mPostedBuffer);
        const bool canCopyBack = (frontBuffer != nullptr &&
                backBuffer->width  == frontBuffer->width &&
                backBuffer->height == frontBuffer->height &&
                backBuffer->format == frontBuffer->format);

        if (canCopyBack) {
            // copy the area that is invalid and not repainted this round
            const Region copyback(mDirtyRegion.subtract(newDirtyRegion));
            if (!copyback.isEmpty()) {
                copyBlt(backBuffer, frontBuffer, copyback, &fenceFd);
            }
        } else {
            // if we can't copy-back anything, modify the user's dirty
            // region to make sure they redraw the whole buffer
            newDirtyRegion.set(bounds);
            mDirtyRegion.clear();
            Mutex::Autolock lock(mMutex);
            for (size_t i=0 ; i<NUM_BUFFER_SLOTS ; i++) {
                mSlots[i].dirtyRegion.clear();
            }
        }


        { // scope for the lock
            Mutex::Autolock lock(mMutex);
            int backBufferSlot(getSlotFromBufferLocked(backBuffer.get()));
            if (backBufferSlot >= 0) {
                Region& dirtyRegion(mSlots[backBufferSlot].dirtyRegion);
                mDirtyRegion.subtract(dirtyRegion);
                dirtyRegion = newDirtyRegion;
            }
        }

        mDirtyRegion.orSelf(newDirtyRegion);
        if (inOutDirtyBounds) {
            *inOutDirtyBounds = newDirtyRegion.getBounds();
        }

        void* vaddr;
        status_t res = backBuffer->lockAsync(
                GRALLOC_USAGE_SW_READ_OFTEN | GRALLOC_USAGE_SW_WRITE_OFTEN,
                newDirtyRegion.bounds(), &vaddr, fenceFd);

        ALOGW_IF(res, "failed locking buffer (handle = %p)",
                backBuffer->handle);

        if (res != 0) {
            err = INVALID_OPERATION;
        } else {
            mLockedBuffer = backBuffer;
            outBuffer->width  = backBuffer->width;
            outBuffer->height = backBuffer->height;
            outBuffer->stride = backBuffer->stride;
            outBuffer->format = backBuffer->format;
            outBuffer->bits   = vaddr;
        }
    }
    return err;
}

```
看看Log：

``` cpp
Stack Trace:
  RELADDR           FUNCTION                                                                                                                                        FILE:LINE
  000000000001fc18  android::Gralloc2Mapper::lock(native_handle const*, unsigned long, android::Rect const&, int, void**, int*, int*) const+224                     frameworks/native/libs/ui/Gralloc2.cpp:276
  0000000000025310  android::GraphicBufferMapper::lockAsync(native_handle const*, unsigned long, unsigned long, android::Rect const&, void**, int, int*, int*)+152  frameworks/native/libs/ui/GraphicBufferMapper.cpp:146
  v-------------->  android::GraphicBuffer::lockAsync(unsigned long, unsigned long, android::Rect const&, void**, int, int*, int*)                                  frameworks/native/libs/ui/GraphicBuffer.cpp:336
  0000000000022e0c  android::GraphicBuffer::lockAsync(unsigned int, android::Rect const&, void**, int, int*, int*)+164                                              frameworks/native/libs/ui/GraphicBuffer.cpp:322
  00000000000bc11c  android::Surface::lock(ANativeWindow_Buffer*, ARect*)+1172                                                                                      frameworks/native/libs/gui/Surface.cpp:1878
  0000000000011048  android::LayerTransactionTest::getLayerBuffer(android::sp<android::SurfaceControl> const&)+248                                                  /system/bin/SurfaceFlinger_test_red
  000000000000f1e4  android::LayerTransactionTest_SetPositionBasic_Test::TestBody()+404                                                                             /system/bin/SurfaceFlinger_test_red
  0000000000017b8c  testing::Test::Run()+668                                                                                                                        /system/bin/SurfaceFlinger_test_red
  00000000000188dc  testing::TestInfo::Run()+676                                                                                                                    /system/bin/SurfaceFlinger_test_red
  0000000000018d70  testing::TestSuite::Run()+272                                                                                                                   /system/bin/SurfaceFlinger_test_red
  0000000000025678  testing::internal::UnitTestImpl::RunAllTests()+1112                                                                                             /system/bin/SurfaceFlinger_test_red
  00000000000251cc  testing::UnitTest::Run()+340                                                                                                                    /system/bin/SurfaceFlinger_test_red
  0000000000012210  main+72                                                                                                                                         /system/bin/SurfaceFlinger_test_red
  000000000007d844  __libc_init+108                                                                                                                                 bionic/libc/bionic/libc_init_dynamic.cpp:136

```

Surface的lock函数用来申请图形缓冲区和一些操作，方法不长，大概工作有：
       1）调用connect函数完成一些初始化；
       2）调用dequeueBuffer函数，申请图形缓冲区；
       3）计算需要绘制的新的dirty区域，旧的区域原样copy数据。
       [->BufferQueueProducer.cpp]
       
``` cpp
X:\frameworks\native\libs\gui\BufferQueueProducer.cpp
status_t BufferQueueProducer::dequeueBuffer(int* outSlot, sp<android::Fence>* outFence,
                                            uint32_t width, uint32_t height, PixelFormat format,
                                            uint64_t usage, uint64_t* outBufferAge,
                                            FrameEventHistoryDelta* outTimestamps) {
    ATRACE_CALL();
    { // Autolock scope
        std::lock_guard<std::mutex> lock(mCore->mMutex);
        mConsumerName = mCore->mConsumerName;

        if (mCore->mIsAbandoned) {
            BQ_LOGE("dequeueBuffer: BufferQueue has been abandoned");
            return NO_INIT;
        }

        if (mCore->mConnectedApi == BufferQueueCore::NO_CONNECTED_API) {
            BQ_LOGE("dequeueBuffer: BufferQueue has no connected producer");
            return NO_INIT;
        }
    } // Autolock scope

    BQ_LOGV("dequeueBuffer: w=%u h=%u format=%#x, usage=%#" PRIx64, width, height, format, usage);

    if ((width && !height) || (!width && height)) {
        BQ_LOGE("dequeueBuffer: invalid size: w=%u h=%u", width, height);
        return BAD_VALUE;
    }

    status_t returnFlags = NO_ERROR;
    EGLDisplay eglDisplay = EGL_NO_DISPLAY;
    EGLSyncKHR eglFence = EGL_NO_SYNC_KHR;
    bool attachedByConsumer = false;

    { // Autolock scope
        std::unique_lock<std::mutex> lock(mCore->mMutex);

        // If we don't have a free buffer, but we are currently allocating, we wait until allocation
        // is finished such that we don't allocate in parallel.
        if (mCore->mFreeBuffers.empty() && mCore->mIsAllocating) {
            mDequeueWaitingForAllocation = true;
            mCore->waitWhileAllocatingLocked(lock);
            mDequeueWaitingForAllocation = false;
            mDequeueWaitingForAllocationCondition.notify_all();
        }

        if (format == 0) {
            format = mCore->mDefaultBufferFormat;
        }

        // Enable the usage bits the consumer requested
        usage |= mCore->mConsumerUsageBits;

        const bool useDefaultSize = !width && !height;
        if (useDefaultSize) {
            width = mCore->mDefaultWidth;
            height = mCore->mDefaultHeight;
        }

        int found = BufferItem::INVALID_BUFFER_SLOT;
        while (found == BufferItem::INVALID_BUFFER_SLOT) {
            status_t status = waitForFreeSlotThenRelock(FreeSlotCaller::Dequeue, lock, &found);
            if (status != NO_ERROR) {
                return status;
            }

            // This should not happen
            if (found == BufferQueueCore::INVALID_BUFFER_SLOT) {
                BQ_LOGE("dequeueBuffer: no available buffer slots");
                return -EBUSY;
            }

            const sp<GraphicBuffer>& buffer(mSlots[found].mGraphicBuffer);

            // If we are not allowed to allocate new buffers,
            // waitForFreeSlotThenRelock must have returned a slot containing a
            // buffer. If this buffer would require reallocation to meet the
            // requested attributes, we free it and attempt to get another one.
            if (!mCore->mAllowAllocation) {
                if (buffer->needsReallocation(width, height, format, BQ_LAYER_COUNT, usage)) {
                    if (mCore->mSharedBufferSlot == found) {
                        BQ_LOGE("dequeueBuffer: cannot re-allocate a sharedbuffer");
                        return BAD_VALUE;
                    }
                    mCore->mFreeSlots.insert(found);
                    mCore->clearBufferSlotLocked(found);
                    found = BufferItem::INVALID_BUFFER_SLOT;
                    continue;
                }
            }
        }

        const sp<GraphicBuffer>& buffer(mSlots[found].mGraphicBuffer);
        if (mCore->mSharedBufferSlot == found &&
                buffer->needsReallocation(width, height, format, BQ_LAYER_COUNT, usage)) {
            BQ_LOGE("dequeueBuffer: cannot re-allocate a shared"
                    "buffer");

            return BAD_VALUE;
        }

        if (mCore->mSharedBufferSlot != found) {
            mCore->mActiveBuffers.insert(found);
        }
        *outSlot = found;
        ATRACE_BUFFER_INDEX(found);

        attachedByConsumer = mSlots[found].mNeedsReallocation;
        mSlots[found].mNeedsReallocation = false;

        mSlots[found].mBufferState.dequeue();

        if ((buffer == nullptr) ||
                buffer->needsReallocation(width, height, format, BQ_LAYER_COUNT, usage))
        {
            mSlots[found].mAcquireCalled = false;
            mSlots[found].mGraphicBuffer = nullptr;
            mSlots[found].mRequestBufferCalled = false;
            mSlots[found].mEglDisplay = EGL_NO_DISPLAY;
            mSlots[found].mEglFence = EGL_NO_SYNC_KHR;
            mSlots[found].mFence = Fence::NO_FENCE;
            mCore->mBufferAge = 0;
            mCore->mIsAllocating = true;

            returnFlags |= BUFFER_NEEDS_REALLOCATION;
        } else {
            // We add 1 because that will be the frame number when this buffer
            // is queued
            mCore->mBufferAge = mCore->mFrameCounter + 1 - mSlots[found].mFrameNumber;
        }

        BQ_LOGV("dequeueBuffer: setting buffer age to %" PRIu64,
                mCore->mBufferAge);

        if (CC_UNLIKELY(mSlots[found].mFence == nullptr)) {
            BQ_LOGE("dequeueBuffer: about to return a NULL fence - "
                    "slot=%d w=%d h=%d format=%u",
                    found, buffer->width, buffer->height, buffer->format);
        }

        eglDisplay = mSlots[found].mEglDisplay;
        eglFence = mSlots[found].mEglFence;
        // Don't return a fence in shared buffer mode, except for the first
        // frame.
        *outFence = (mCore->mSharedBufferMode &&
                mCore->mSharedBufferSlot == found) ?
                Fence::NO_FENCE : mSlots[found].mFence;
        mSlots[found].mEglFence = EGL_NO_SYNC_KHR;
        mSlots[found].mFence = Fence::NO_FENCE;

        // If shared buffer mode has just been enabled, cache the slot of the
        // first buffer that is dequeued and mark it as the shared buffer.
        if (mCore->mSharedBufferMode && mCore->mSharedBufferSlot ==
                BufferQueueCore::INVALID_BUFFER_SLOT) {
            mCore->mSharedBufferSlot = found;
            mSlots[found].mBufferState.mShared = true;
        }
    } // Autolock scope

    if (returnFlags & BUFFER_NEEDS_REALLOCATION) {
        BQ_LOGV("dequeueBuffer: allocating a new buffer for slot %d", *outSlot);
        sp<GraphicBuffer> graphicBuffer = new GraphicBuffer(
                width, height, format, BQ_LAYER_COUNT, usage,
                {mConsumerName.string(), mConsumerName.size()});

        status_t error = graphicBuffer->initCheck();

        { // Autolock scope
            std::lock_guard<std::mutex> lock(mCore->mMutex);

            if (error == NO_ERROR && !mCore->mIsAbandoned) {
                graphicBuffer->setGenerationNumber(mCore->mGenerationNumber);
                mSlots[*outSlot].mGraphicBuffer = graphicBuffer;
            }

            mCore->mIsAllocating = false;
            mCore->mIsAllocatingCondition.notify_all();

            if (error != NO_ERROR) {
                mCore->mFreeSlots.insert(*outSlot);
                mCore->clearBufferSlotLocked(*outSlot);
                BQ_LOGE("dequeueBuffer: createGraphicBuffer failed");
                return error;
            }

            if (mCore->mIsAbandoned) {
                mCore->mFreeSlots.insert(*outSlot);
                mCore->clearBufferSlotLocked(*outSlot);
                BQ_LOGE("dequeueBuffer: BufferQueue has been abandoned");
                return NO_INIT;
            }

            VALIDATE_CONSISTENCY();
        } // Autolock scope
    }

    if (attachedByConsumer) {
        returnFlags |= BUFFER_NEEDS_REALLOCATION;
    }

    if (eglFence != EGL_NO_SYNC_KHR) {
        EGLint result = eglClientWaitSyncKHR(eglDisplay, eglFence, 0,
                1000000000);
        // If something goes wrong, log the error, but return the buffer without
        // synchronizing access to it. It's too late at this point to abort the
        // dequeue operation.
        if (result == EGL_FALSE) {
            BQ_LOGE("dequeueBuffer: error %#x waiting for fence",
                    eglGetError());
        } else if (result == EGL_TIMEOUT_EXPIRED_KHR) {
            BQ_LOGE("dequeueBuffer: timeout waiting for fence");
        }
        eglDestroySyncKHR(eglDisplay, eglFence);
    }

    BQ_LOGV("dequeueBuffer: returning slot=%d/%" PRIu64 " buf=%p flags=%#x",
            *outSlot,
            mSlots[*outSlot].mFrameNumber,
            mSlots[*outSlot].mGraphicBuffer->handle, returnFlags);

    if (outBufferAge) {
        *outBufferAge = mCore->mBufferAge;
    }
    addAndGetFrameTimestamps(nullptr, outTimestamps);

    return returnFlags;
}
```
看看Log：

``` prolog
10-28 09:41:00.093   884   884 V Surface : Surface::connect
10-28 09:41:00.093   884   884 V Surface : Surface::setUsage
10-28 09:41:00.093   884   884 V Surface : Surface::dequeueBuffer
10-28 09:41:00.093   179   523 V BufferQueueProducer: [test#0] dequeueBuffer: w=0 h=0 format=0, usage=0x33

Stack Trace:
  RELADDR           FUNCTION                                                                                                                                                                                                                                                 FILE:LINE
  00000000000203cc  android::Gralloc2Allocator::allocate(unsigned int, unsigned int, int, unsigned int, unsigned long, unsigned int, unsigned int*, native_handle const**) const+172                                                                                         frameworks/native/libs/ui/Gralloc2.cpp:444
  00000000000243f4  android::GraphicBufferAllocator::allocate(unsigned int, unsigned int, int, unsigned int, unsigned long, native_handle const**, unsigned int*, unsigned long, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >)+284  frameworks/native/libs/ui/GraphicBufferAllocator.cpp:134
  0000000000022234  android::GraphicBuffer::initWithSize(unsigned int, unsigned int, int, unsigned int, unsigned long, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >)+212                                                            frameworks/native/libs/ui/GraphicBuffer.cpp:207
  0000000000022114  android::GraphicBuffer::GraphicBuffer(unsigned int, unsigned int, int, unsigned int, unsigned long, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >)+124                                                           frameworks/native/libs/ui/GraphicBuffer.cpp:92
  0000000000070a8c  android::BufferQueueProducer::dequeueBuffer(int*, android::sp<android::Fence>*, unsigned int, unsigned int, int, unsigned long, unsigned long*, android::FrameEventHistoryDelta*)+1516                                                                   frameworks/native/libs/gui/BufferQueueProducer.cpp:515
  000000000007ef34  android::BnGraphicBufferProducer::onTransact(unsigned int, android::Parcel const&, android::Parcel*, unsigned int)+1300                                                                                                                                  frameworks/native/libs/gui/IGraphicBufferProducer.cpp:806
  000000000004c6b8  android::BBinder::transact(unsigned int, android::Parcel const&, android::Parcel*, unsigned int)+136                                                                                                                                                     frameworks/native/libs/binder/Binder.cpp:134
  0000000000058d08  android::IPCThreadState::executeCommand(int)+984                                                                                                                                                                                                         frameworks/native/libs/binder/IPCThreadState.cpp:1213
  000000000005887c  android::IPCThreadState::getAndExecuteCommand()+156                                                                                                                                                                                                      frameworks/native/libs/binder/IPCThreadState.cpp:514
  0000000000058fc4  android::IPCThreadState::joinThreadPool(bool)+60                                                                                                                                                                                                         frameworks/native/libs/binder/IPCThreadState.cpp:594
  000000000007f578  android::PoolThread::threadLoop()+24                                                                                                                                                                                                                     frameworks/native/libs/binder/ProcessState.cpp:67
  00000000000137a4  android::Thread::_threadLoop(void*)+284                                                                                                                                                                                                                  system/core/libutils/Threads.cpp:746
  00000000000e230c  __pthread_start(void*)+36                                                                                                                                                                                                                                bionic/libc/bionic/pthread_create.cpp:338
  0000000000083d98  __start_thread+64                                                                                                                                                                                                                                        bionic/libc/bionic/clone.cpp:53


```
接着就会创建GraphicBuffer对象

``` cpp
    if (returnFlags & BUFFER_NEEDS_REALLOCATION) {
        BQ_LOGV("dequeueBuffer: allocating a new buffer for slot %d", *outSlot);
        sp<GraphicBuffer> graphicBuffer = new GraphicBuffer(
                width, height, format, BQ_LAYER_COUNT, usage,
                {mConsumerName.string(), mConsumerName.size()});
X:\frameworks\native\libs\ui\GraphicBuffer.cpp
GraphicBuffer::GraphicBuffer(uint32_t inWidth, uint32_t inHeight, PixelFormat inFormat,
                             uint32_t inLayerCount, uint64_t inUsage, std::string requestorName)
      : GraphicBuffer() {
    mInitCheck = initWithSize(inWidth, inHeight, inFormat, inLayerCount, inUsage,
                              std::move(requestorName));
}
status_t GraphicBuffer::initWithSize(uint32_t inWidth, uint32_t inHeight,
        PixelFormat inFormat, uint32_t inLayerCount, uint64_t inUsage,
        std::string requestorName)
{
    GraphicBufferAllocator& allocator = GraphicBufferAllocator::get();
    uint32_t outStride = 0;
    status_t err = allocator.allocate(inWidth, inHeight, inFormat, inLayerCount,
            inUsage, &handle, &outStride, mId,
            std::move(requestorName));
    if (err == NO_ERROR) {
        mBufferMapper.getTransportSize(handle, &mTransportNumFds, &mTransportNumInts);

        width = static_cast<int>(inWidth);
        height = static_cast<int>(inHeight);
        format = inFormat;
        layerCount = inLayerCount;
        usage = inUsage;
        usage_deprecated = int(usage);
        stride = static_cast<int>(outStride);
    }
    return err;
}

```
接着请求GraphicBufferAllocator 做一个allocate的动作。

``` cpp
X:\frameworks\native\libs\ui\GraphicBufferAllocator.cpp
status_t GraphicBufferAllocator::allocate(uint32_t width, uint32_t height,
        PixelFormat format, uint32_t layerCount, uint64_t usage,
        buffer_handle_t* handle, uint32_t* stride,
        uint64_t /*graphicBufferId*/, std::string requestorName)
{
    ATRACE_CALL();

    // make sure to not allocate a N x 0 or 0 x N buffer, since this is
    // allowed from an API stand-point allocate a 1x1 buffer instead.
    if (!width || !height)
        width = height = 1;

    const uint32_t bpp = bytesPerPixel(format);
    if (std::numeric_limits<size_t>::max() / width / height < static_cast<size_t>(bpp)) {
        ALOGE("Failed to allocate (%u x %u) layerCount %u format %d "
              "usage %" PRIx64 ": Requesting too large a buffer size",
              width, height, layerCount, format, usage);
        return BAD_VALUE;
    }

    // Ensure that layerCount is valid.
    if (layerCount < 1)
        layerCount = 1;

    // TODO(b/72323293, b/72703005): Remove these invalid bits from callers
    usage &= ~static_cast<uint64_t>((1 << 10) | (1 << 13));

    status_t error =
            mAllocator->allocate(width, height, format, layerCount, usage, 1, stride, handle);
    size_t bufSize;

    // if stride has no meaning or is too large,
    // approximate size with the input width instead
    if ((*stride) != 0 &&
        std::numeric_limits<size_t>::max() / height / (*stride) < static_cast<size_t>(bpp)) {
        bufSize = static_cast<size_t>(width) * height * bpp;
    } else {
        bufSize = static_cast<size_t>((*stride)) * height * bpp;
    }

    if (error == NO_ERROR) {
        Mutex::Autolock _l(sLock);
        KeyedVector<buffer_handle_t, alloc_rec_t>& list(sAllocList);
        alloc_rec_t rec;
        rec.width = width;
        rec.height = height;
        rec.stride = *stride;
        rec.format = format;
        rec.layerCount = layerCount;
        rec.usage = usage;
        rec.size = bufSize;
        rec.requestorName = std::move(requestorName);
        list.add(*handle, rec);

        return NO_ERROR;
    } else {
        ALOGE("Failed to allocate (%u x %u) layerCount %u format %d "
                "usage %" PRIx64 ": %d",
                width, height, layerCount, format, usage,
                error);
        return NO_MEMORY;
    }
}


X:\frameworks\native\libs\ui\Gralloc2.cpp
status_t Gralloc2Allocator::allocate(uint32_t width, uint32_t height, PixelFormat format,
                                     uint32_t layerCount, uint64_t usage, uint32_t bufferCount,
                                     uint32_t* outStride, buffer_handle_t* outBufferHandles) const {
    IMapper::BufferDescriptorInfo descriptorInfo = {};
    descriptorInfo.width = width;
    descriptorInfo.height = height;
    descriptorInfo.layerCount = layerCount;
    descriptorInfo.format = static_cast<hardware::graphics::common::V1_1::PixelFormat>(format);
    descriptorInfo.usage = usage;

    BufferDescriptor descriptor;
    status_t error = mMapper.createDescriptor(static_cast<void*>(&descriptorInfo),
                                              static_cast<void*>(&descriptor));
    if (error != NO_ERROR) {
        return error;
    }
    if(printTrace > 0){
        ALOGD_CALLSTACK("allocate");
	    printTrace--;
	}

    auto ret = mAllocator->allocate(descriptor, bufferCount,
                                    [&](const auto& tmpError, const auto& tmpStride,
                                        const auto& tmpBuffers) {
                                        error = static_cast<status_t>(tmpError);
                                        if (tmpError != Error::NONE) {
                                            return;
                                        }

                                        // import buffers
                                        for (uint32_t i = 0; i < bufferCount; i++) {
                                            error = mMapper.importBuffer(tmpBuffers[i],
                                                                         &outBufferHandles[i]);
                                            if (error != NO_ERROR) {
                                                for (uint32_t j = 0; j < i; j++) {
                                                    mMapper.freeBuffer(outBufferHandles[j]);
                                                    outBufferHandles[j] = nullptr;
                                                }
                                                return;
                                            }
                                        }

                                        *outStride = tmpStride;
                                    });

    // make sure the kernel driver sees BC_FREE_BUFFER and closes the fds now
    hardware::IPCThreadState::self()->flushCommands();

    return (ret.isOk()) ? error : static_cast<status_t>(kTransactionError);
}
```
看看Log：

```
Stack Trace:
  RELADDR           FUNCTION                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      FILE:LINE
  000000000001f7e0  android::Gralloc2Mapper::importBuffer(android::hardware::hidl_handle const&, native_handle const**) const+112                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 frameworks/native/libs/ui/Gralloc2.cpp:180
  v-------------->  operator()<android::hardware::graphics::mapper::V2_0::Error, unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> >                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      frameworks/native/libs/ui/Gralloc2.cpp:458
  v-------------->  _ZNSt3__18__invokeIRZNK7android17Gralloc2Allocator8allocateEjjijmjPjPPK13native_handleE3$_7JNS1_8hardware8graphics6mapper4V2_05ErrorEjRKNSA_8hidl_vecINSA_11hidl_handleEEEEEEDTclclsr3std3__1E7forwardIT_Efp_Espclsr3std3__1E7forwardIT0_Efp0_EEEOSK_DpOSL_                                                                                                                                                                                                                                                                                                                                                                                                                                   external/libcxx/include/type_traits:4353
  v-------------->  void std::__1::__invoke_void_return_wrapper<void>::__call<android::Gralloc2Allocator::allocate(unsigned int, unsigned int, int, unsigned int, unsigned long, unsigned int, unsigned int*, native_handle const**) const::$_7&, android::hardware::graphics::mapper::V2_0::Error, unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> const&>(android::Gralloc2Allocator::allocate(unsigned int, unsigned int, int, unsigned int, unsigned long, unsigned int, unsigned int*, native_handle const**) const::$_7&, android::hardware::graphics::mapper::V2_0::Error&&, unsigned int&&, android::hardware::hidl_vec<android::hardware::hidl_handle> const&)                 external/libcxx/include/__functional_base:349
  v-------------->  std::__1::__function::__alloc_func<android::Gralloc2Allocator::allocate(unsigned int, unsigned int, int, unsigned int, unsigned long, unsigned int, unsigned int*, native_handle const**) const::$_7, std::__1::allocator<android::Gralloc2Allocator::allocate(unsigned int, unsigned int, int, unsigned int, unsigned long, unsigned int, unsigned int*, native_handle const**) const::$_7>, void (android::hardware::graphics::mapper::V2_0::Error, unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> const&)>::operator()(android::hardware::graphics::mapper::V2_0::Error&&, unsigned int&&, android::hardware::hidl_vec<android::hardware::hidl_handle> const&)  external/libcxx/include/functional:1527
  0000000000020bcc  std::__1::__function::__func<android::Gralloc2Allocator::allocate(unsigned int, unsigned int, int, unsigned int, unsigned long, unsigned int, unsigned int*, native_handle const**) const::$_7, std::__1::allocator<android::Gralloc2Allocator::allocate(unsigned int, unsigned int, int, unsigned int, unsigned long, unsigned int, unsigned int*, native_handle const**) const::$_7>, void (android::hardware::graphics::mapper::V2_0::Error, unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> const&)>::operator()(android::hardware::graphics::mapper::V2_0::Error&&, unsigned int&&, android::hardware::hidl_vec<android::hardware::hidl_handle> const&)+116    external/libcxx/include/functional:1651
  v-------------->  std::__1::__function::__value_func<void (android::hardware::graphics::mapper::V2_0::Error, unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> const&)>::operator()(android::hardware::graphics::mapper::V2_0::Error&&, unsigned int&&, android::hardware::hidl_vec<android::hardware::hidl_handle> const&) const                                                                                                                                                                                                                                                                                                                                                       external/libcxx/include/functional:1799
  v-------------->  std::__1::function<void (android::hardware::graphics::mapper::V2_0::Error, unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> const&)>::operator()(android::hardware::graphics::mapper::V2_0::Error, unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> const&) const                                                                                                                                                                                                                                                                                                                                                                           external/libcxx/include/functional:2347
  000000000000b9c8  android::hardware::graphics::allocator::V2_0::BpHwAllocator::_hidl_allocate(android::hardware::IInterface*, android::hardware::details::HidlInstrumentor*, android::hardware::hidl_vec<unsigned int> const&, unsigned int, std::__1::function<void (android::hardware::graphics::mapper::V2_0::Error, unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> const&)>)+864                                                                                                                                                                                                                                                                                                 out/soong/.intermediates/hardware/interfaces/graphics/allocator/2.0/android.hardware.graphics.allocator@2.0_genc++/gen/android/hardware/graphics/allocator/2.0/AllocatorAll.cpp:295
  000000000000bdac  android::hardware::graphics::allocator::V2_0::BpHwAllocator::allocate(android::hardware::hidl_vec<unsigned int> const&, unsigned int, std::__1::function<void (android::hardware::graphics::mapper::V2_0::Error, unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> const&)>)+156                                                                                                                                                                                                                                                                                                                                                                                      out/soong/.intermediates/hardware/interfaces/graphics/allocator/2.0/android.hardware.graphics.allocator@2.0_genc++/gen/android/hardware/graphics/allocator/2.0/AllocatorAll.cpp:326
  000000000002045c  android::Gralloc2Allocator::allocate(unsigned int, unsigned int, int, unsigned int, unsigned long, unsigned int, unsigned int*, native_handle const**) const+316                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              frameworks/native/libs/ui/Gralloc2.cpp:448
  00000000000243f4  android::GraphicBufferAllocator::allocate(unsigned int, unsigned int, int, unsigned int, unsigned long, native_handle const**, unsigned int*, unsigned long, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >)+284                                                                                                                                                                                                                                                                                                                                                                                                                                       frameworks/native/libs/ui/GraphicBufferAllocator.cpp:134
  0000000000022234  android::GraphicBuffer::initWithSize(unsigned int, unsigned int, int, unsigned int, unsigned long, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >)+212                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 frameworks/native/libs/ui/GraphicBuffer.cpp:207
  0000000000022114  android::GraphicBuffer::GraphicBuffer(unsigned int, unsigned int, int, unsigned int, unsigned long, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >)+124                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                frameworks/native/libs/ui/GraphicBuffer.cpp:92
  0000000000070b50  android::BufferQueueProducer::dequeueBuffer(int*, android::sp<android::Fence>*, unsigned int, unsigned int, int, unsigned long, unsigned long*, android::FrameEventHistoryDelta*)+1576                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        frameworks/native/libs/gui/BufferQueueProducer.cpp:515
  000000000007f47c  android::BnGraphicBufferProducer::onTransact(unsigned int, android::Parcel const&, android::Parcel*, unsigned int)+1300                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       frameworks/native/libs/gui/IGraphicBufferProducer.cpp:806
  000000000004c6b8  android::BBinder::transact(unsigned int, android::Parcel const&, android::Parcel*, unsigned int)+136                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          frameworks/native/libs/binder/Binder.cpp:134
  0000000000058d08  android::IPCThreadState::executeCommand(int)+984                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              frameworks/native/libs/binder/IPCThreadState.cpp:1213
  000000000005887c  android::IPCThreadState::getAndExecuteCommand()+156                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           frameworks/native/libs/binder/IPCThreadState.cpp:514
  0000000000058fc4  android::IPCThreadState::joinThreadPool(bool)+60                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              frameworks/native/libs/binder/IPCThreadState.cpp:594
  000000000007f578  android::PoolThread::threadLoop()+24                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          frameworks/native/libs/binder/ProcessState.cpp:67
  00000000000137a4  android::Thread::_threadLoop(void*)+284                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       system/core/libutils/Threads.cpp:746
  00000000000e230c  __pthread_start(void*)+36                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     bionic/libc/bionic/pthread_create.cpp:338
  0000000000083d98  __start_thread+64                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             bionic/libc/bionic/clone.cpp:53

```

[->BufferQueueProducer.cpp]

``` cpp
status_t BufferQueueProducer::requestBuffer(int slot, sp<GraphicBuffer>* buf) {
ATRACE_CALL();
Mutex::Autolock lock(mCore->mMutex);

......

mSlots[slot].mRequestBufferCalled = true;
*buf = mSlots[slot].mGraphicBuffer;
return NO_ERROR;
}
```

 这个比较简单，还是很好理解的额，就是根据指定index取出mSlots中的slot中的buffer。这里只有一个buffer，slot为0。


看看Log：

```
10-28 09:41:00.121   179   523 D GRALLOC-DRM: handle: name=0 pfd=46
10-28 09:41:00.121   179   523 D GRALLOC-ROCKCHIP: [File] : hardware/rockchip/libgralloc/midgard/gralloc_drm_rockchip.cpp; [Line] : 1766; [Func] : drm_gem_rockchip_alloc;
10-28 09:41:00.121   179   523 D GRALLOC-ROCKCHIP: drm_gem_rockchip_alloc enter, w : 362, h : 1920, format : 0x1, usage : 0x933.
10-28 09:41:00.121   179   523 D GRALLOC-ROCKCHIP: [File] : hardware/rockchip/libgralloc/midgard/gralloc_drm_rockchip.cpp; [Line] : 2168; [Func] : drm_gem_rockchip_alloc;
10-28 09:41:00.121   179   523 D GRALLOC-ROCKCHIP: to ask for cachable buffer for CPU read, usage : 0x933
10-28 09:41:00.121   179   523 D GRALLOC-ROCKCHIP: [File] : hardware/rockchip/libgralloc/midgard/gralloc_drm_rockchip.cpp; [Line] : 2664; [Func] : rk_drm_adapter_import_dma_buf;
10-28 09:41:00.121   179   523 D GRALLOC-ROCKCHIP: imported a dma_buf as a gem_obj with handle 4
10-28 09:41:00.121   179   523 D GRALLOC-ROCKCHIP: [File] : hardware/rockchip/libgralloc/midgard/gralloc_drm_rockchip.cpp; [Line] : 2375; [Func] : drm_gem_rockchip_alloc;
10-28 09:41:00.121   179   523 D GRALLOC-ROCKCHIP: leave, w : 362, h : 1920, format : 0x1,internal_format : 0x1, usage : 0x933. size=2826240,pixel_stride=368,byte_stride=1472
10-28 09:41:00.121   179   523 D GRALLOC-ROCKCHIP: [File] : hardware/rockchip/libgralloc/midgard/gralloc_drm_rockchip.cpp; [Line] : 2376; [Func] : drm_gem_rockchip_alloc;
10-28 09:41:00.121   179   523 D GRALLOC-ROCKCHIP: leave: prime_fd=46,share_attr_fd=48
10-28 09:41:00.121   179   523 V BufferQueueProducer: [test#0] dequeueBuffer: returning slot=0/0 buf=0x70f185f180 flags=0x1

10-28 09:41:00.122   179   189 V BufferQueueProducer: [test#0] requestBuffer: slot 0
```


##### 3.1.1、SurfaceFlinger_test_red填充数据
我们这里将Layer填充为屏幕三分之一大小的红色。
``` cpp
// Fill a region with the specified color.
void fillBufferColor(const ANativeWindow_Buffer& buffer, const Rect& rect, const Color& color) {
    int32_t x = rect.left;
    int32_t y = rect.top;
    int32_t width = rect.right - rect.left;
    int32_t height = rect.bottom - rect.top;

    if (x < 0) {
        width += x;
        x = 0;
    }
    if (y < 0) {
        height += y;
        y = 0;
    }
    if (x + width > buffer.width) {
        x = std::min(x, buffer.width);
        width = buffer.width - x;
    }
    if (y + height > buffer.height) {
        y = std::min(y, buffer.height);
        height = buffer.height - y;
    }

    for (int32_t j = 0; j < height; j++) {
        uint8_t* dst = static_cast<uint8_t*>(buffer.bits) + (buffer.stride * (y + j) + x) * 4;
        for (int32_t i = 0; i < width; i++) {
            dst[0] = color.r;
            dst[1] = color.g;
            dst[2] = color.b;
            dst[3] = color.a;
            dst += 4;
        }
    }
}
} // anonymous namespace

    void fillLayerColor(const sp<SurfaceControl>& layer, const Color& color) {
        ANativeWindow_Buffer buffer;
        ASSERT_NO_FATAL_FAILURE(buffer = getLayerBuffer(layer));
        fillBufferColor(buffer, Rect(0, 0, buffer.width, buffer.height), color);
        postLayerBuffer(layer);
    }
    void postLayerBuffer(const sp<SurfaceControl>& layer) {
        ASSERT_EQ(NO_ERROR, layer->getSurface()->unlockAndPost());

        // wait for the newly posted buffer to be latched
        waitForLayerBuffers();
    }

```


##### 3.1.2、APP提交(unlockAndPost)Buffer的过程
 [->Surface.cpp]
       
``` cpp
status_t Surface::unlockAndPost()
{
......

int fd = -1;
//解锁图形缓冲区，和前面的lockAsync成对出现
status_t err = mLockedBuffer->unlockAsync(&fd);
//queueBuffer去归还图形缓冲区
err = queueBuffer(mLockedBuffer.get(), fd);


mPostedBuffer = mLockedBuffer;
mLockedBuffer = 0;
return err;
}
```

``` bash

Stack Trace:
  RELADDR           FUNCTION                                                                                        FILE:LINE
  000000000001ffe4  android::Gralloc2Mapper::unlock(native_handle const*) const+116                                 frameworks/native/libs/ui/Gralloc2.cpp:361
  0000000000025218  android::GraphicBufferMapper::unlockAsync(native_handle const*, int*)+80                        frameworks/native/libs/ui/GraphicBufferMapper.cpp:162
  00000000000bc2f0  android::Surface::unlockAndPost()+64                                                            frameworks/native/libs/gui/Surface.cpp:1907
  000000000001118c  android::LayerTransactionTest::postLayerBuffer(android::sp<android::SurfaceControl> const&)+68  /system/bin/SurfaceFlinger_test_red
  000000000000f310  android::LayerTransactionTest_SetPositionBasic_Test::TestBody()+704                             /system/bin/SurfaceFlinger_test_red
  0000000000017b8c  testing::Test::Run()+668                                                                        /system/bin/SurfaceFlinger_test_red
  00000000000188dc  testing::TestInfo::Run()+676                                                                    /system/bin/SurfaceFlinger_test_red
  0000000000018d70  testing::TestSuite::Run()+272                                                                   /system/bin/SurfaceFlinger_test_red
  0000000000025678  testing::internal::UnitTestImpl::RunAllTests()+1112                                             /system/bin/SurfaceFlinger_test_red
  00000000000251cc  testing::UnitTest::Run()+340                                                                    /system/bin/SurfaceFlinger_test_red
  0000000000012210  main+72                                                                                         /system/bin/SurfaceFlinger_test_red
  000000000007d844  __libc_init+108                                                                                 bionic/libc/bionic/libc_init_dynamic.cpp:136



10-28 09:41:00.183   884   884 V Surface : Surface::queueBuffer
10-28 09:41:00.183   884   884 V Surface : Surface::queueBuffer making up timestamp: 43112.59 ms
10-28 09:41:00.183   179   523 V BufferQueueProducer: [test#0] queueBuffer: slot=0/1 time=43112591811 dataSpace=0 validHdrMetadataTypes=0x0 crop=[0,0,0,0] transform=0 scale=FREEZE
10-28 09:41:00.184   179   523 V ConsumerBase: [test#0] onFrameAvailable
10-28 09:41:00.184   179   523 V ConsumerBase: [test#0] actually calling onFrameAvailable
10-28 09:41:00.184   179   523 I BufferQueueLayer: zjj.rk3399.SF onFrameAvailable frameworks/native/services/surfaceflinger/BufferQueueLayer.cpp onFrameAvailable 456 
10-28 09:41:00.184   179   523 I BufferLayerConsumer: zjj.rk3399.SF onBufferAvailable frameworks/native/services/surfaceflinger/BufferLayerConsumer.cpp onBufferAvailable 506 
10-28 09:41:00.184   179   523 I BufferLayerConsumer: zjj.rk3399.SF Image frameworks/native/services/surfaceflinger/BufferLayerConsumer.cpp Image 553 
10-28 09:41:00.184   179   523 V BufferQueueProducer: [test#0] addAndGetFrameTimestamps
```

   这里也比较简单，核心也是分两步：
       1）解锁图形缓冲区，和前面的lockAsync成对出现；
       2）queueBuffer去归还图形缓冲区；
        所以我们还是重点分析第二步，查看queueBuffer的实现：
       [->Surface.cpp]
       
``` cpp
int Surface::queueBuffer(android_native_buffer_t* buffer, int fenceFd) {
......
status_t err = mGraphicBufferProducer->queueBuffer(i, input, &output);
mLastQueueDuration = systemTime() - now;
......
return err;
}
```

调用BufferQueueProducer的queueBuffer归还缓冲区，将绘制后的图形缓冲区queue回去。
[->BufferQueueProducer.cpp]

``` cpp
status_t BufferQueueProducer::queueBuffer(int slot,
    const QueueBufferInput &input, QueueBufferOutput *output) {
......

{ // scope for the lock
    Mutex::Autolock lock(mCallbackMutex);
    while (callbackTicket != mCurrentCallbackTicket) {
        mCallbackCondition.wait(mCallbackMutex);
    }

    if (frameAvailableListener != NULL) {
        frameAvailableListener->onFrameAvailable(item);
    } else if (frameReplacedListener != NULL) {
        frameReplacedListener->onFrameReplaced(item);
    }
    ......
}
......
return NO_ERROR;
}
```

总结：
       1）从传入的QueueBufferInput ，解析填充一些变量；
       2）改变入队Slot的状态为QUEUED，每次推进来，mFrameCounter都加1。这里的slot，上一篇讲分配缓冲区返回最老的FREE状态buffer，就是用这个mFrameCounter最小值判断，就是上一篇LRU算法的判断；
       3）创建一个BufferItem来描述GraphicBuffer，用mSlots[slot]中的slot填充BufferItem；
       4）将BufferItem塞进mCore的mQueue队列，依照指定规则；
       5）然后通知SurfaceFlinger去消费。
看看Log：

#### （四）、通知SF消费合成
当绘制完毕的GraphicBuffer入队之后，会通知SurfaceFlinger去消费，就是BufferQueueProducer的queueBuffer函数的最后几行，listener->onFrameAvailable()。
listener最终通过回调，会回到BufferLayer当中，所以最终调用BufferQueueLayer的onFrameAvailable接口，我们看看它的实现：
[Layer.cpp]

``` cpp
void BufferQueueLayer::onFrameAvailable(const BufferItem& item) {
    ATRACE_CALL();
    // Add this buffer from our internal queue tracker
    { // Autolock scope
		ALOGI("zjj.rk3399.SF onFrameAvailable %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

		if (mFlinger->mUseSmart90ForVideo) {
            const nsecs_t presentTime = item.mIsAutoTimestamp ? 0 : item.mTimestamp;
            mFlinger->mScheduler->addLayerPresentTimeAndHDR(mSchedulerLayerHandle, presentTime,
                                                            item.mHdrMetadata.validTypes != 0);
        }

        Mutex::Autolock lock(mQueueItemLock);
        // Reset the frame number tracker when we receive the first buffer after
        // a frame number reset
        if (item.mFrameNumber == 1) {
            mLastFrameNumberReceived = 0;
        }

        // Ensure that callbacks are handled in order
        while (item.mFrameNumber != mLastFrameNumberReceived + 1) {
            status_t result = mQueueItemCondition.waitRelative(mQueueItemLock, ms2ns(500));
            if (result != NO_ERROR) {
                ALOGE("[%s] Timed out waiting on callback", mName.string());
            }
        }

        mQueueItems.push_back(item);
        mQueuedFrames++;

        // Wake up any pending callbacks
        mLastFrameNumberReceived = item.mFrameNumber;
        mQueueItemCondition.broadcast();
    }

    mFlinger->mInterceptor->saveBufferUpdate(this, item.mGraphicBuffer->getWidth(),
                                             item.mGraphicBuffer->getHeight(), item.mFrameNumber);

    // If this layer is orphaned, then we run a fake vsync pulse so that
    // dequeueBuffer doesn't block indefinitely.
    if (isRemovedFromCurrentState()) {
        fakeVsync();
    } else {
        mFlinger->signalLayerUpdate();
    }
    mConsumer->onBufferAvailable(item);
}

```

这里又调用SurfaceFlinger的signalLayerUpdate函数，继续查看：
[SurfaceFlinger.cpp]

``` cpp

void SurfaceFlinger::signalLayerUpdate() {
    mScheduler->resetIdleTimer();
    mEventQueue->invalidate();
}

```

这里又调用MessageQueue的invalidate函数：
[MessageQueue.cpp]

``` cpp
void MessageQueue::invalidate() {
mEvents->requestNextVsync();
}
```


MessageQueue中分发两个消息，一个INVALIDATE，一个REFRESH，SurfaceFlinger对这两个消息的响应过程，就是合成的过程。

##### 4.0、消息INVALIDATE处理
最终结果会走到SurfaceFlinger的vsync信号接收逻辑，即SurfaceFlinger的onMessageReceived函数：
[SurfaceFlinger.cpp]

``` cpp
void SurfaceFlinger::onMessageReceived(int32_t what) NO_THREAD_SAFETY_ANALYSIS {
    ATRACE_CALL();
    switch (what) {
        case MessageQueue::INVALIDATE: {
            // calculate the expected present time once and use the cached
            // value throughout this frame to make sure all layers are
            // seeing this same value.
            populateExpectedPresentTime();

            // When Backpressure propagation is enabled we want to give a small grace period
            // for the present fence to fire instead of just giving up on this frame to handle cases
            // where present fence is just about to get signaled.
            const int graceTimeForPresentFenceMs =
                    (mPropagateBackpressure &&
                     (mPropagateBackpressureClientComposition || !mHadClientComposition))
                    ? 1
                    : 0;
            bool frameMissed = previousFrameMissed(graceTimeForPresentFenceMs);
            bool hwcFrameMissed = mHadDeviceComposition && frameMissed;
            bool gpuFrameMissed = mHadClientComposition && frameMissed;
            ATRACE_INT("FrameMissed", static_cast<int>(frameMissed));
            ATRACE_INT("HwcFrameMissed", static_cast<int>(hwcFrameMissed));
            ATRACE_INT("GpuFrameMissed", static_cast<int>(gpuFrameMissed));
            if (frameMissed) {
                mFrameMissedCount++;
                mTimeStats->incrementMissedFrames();
            }

            if (hwcFrameMissed) {
                mHwcFrameMissedCount++;
            }

            if (gpuFrameMissed) {
                mGpuFrameMissedCount++;
            }

            if (mUseSmart90ForVideo) {
                // This call is made each time SF wakes up and creates a new frame. It is part
                // of video detection feature.
                mScheduler->updateFpsBasedOnContent();
            }

            if (performSetActiveConfig()) {
                break;
            }

            if (frameMissed && mPropagateBackpressure) {
                if ((hwcFrameMissed && !gpuFrameMissed) ||
                    mPropagateBackpressureClientComposition) {
                    signalLayerUpdate();
                    break;
                }
            }

            // Now that we're going to make it to the handleMessageTransaction()
            // call below it's safe to call updateVrFlinger(), which will
            // potentially trigger a display handoff.
            updateVrFlinger();

            bool refreshNeeded = handleMessageTransaction();
            refreshNeeded |= handleMessageInvalidate();

            updateCursorAsync();
            updateInputFlinger();

            refreshNeeded |= mRepaintEverything;
            if (refreshNeeded && CC_LIKELY(mBootStage != BootStage::BOOTLOADER)) {
                // Signal a refresh if a transaction modified the window state,
                // a new buffer was latched, or if HWC has requested a full
                // repaint
                signalRefresh();
            }
            break;
        }
        case MessageQueue::REFRESH: {
            handleMessageRefresh();
            break;
        }
    }
```
在INVALIDATE过程中，主要做以下处理：
- 对丢帧的处理

如果丢帧，且mPropagateBackpressure为true，mPropagateBackpressure表示显示给压力了。显示说，太慢了，都丢帧了，给点压力，上层赶紧处理。mPropagateBackpressure是在SurfaceFlinger的构造函数中初始化的，受debug.sf.disable_backpressure属性的控制。
- 更新VR updateVrFlinger

这个只有在VR模式下才会起作用，我们这里先不管VR的事。

- 处理Transition

Transition的处理，前面我们已经说过，只是当时不清楚是什么时候触发的，现在清楚了。Vsync到来后，触发INVALIDATE消息时先去处理Transition。处理的过程就是前面已经说过的handleMessageTransaction，有需要可以回头去看看。这个过程就是处理应用传过来的各种Transition，需要记住的是在commit Transition时，又个状态的更替，mCurrentState赋值给了mDrawingState。

``` cpp
void SurfaceFlinger::commitTransaction()
{
    ... ...

    mDrawingState = mCurrentState;
    mDrawingState.traverseInZOrder([](Layer* layer) {
        layer->commitChildList();
    });
    ... ...
}
```
所以SurfaceFlinger两个状态：
mCurrentState状态， 准备数据，应用传过来的数据保存在mCurrentState中。
mDrawingState状态，进程合成状态，需要进行合成的数据保存在mDrawingState中。
也就是说，每次合成时，先更新一下状态数据。每一层Layer也需要去更新状态数据。

- 处理Invalidate
这是一个重要的流程，handleMessageInvalidate函数如下：

``` cpp
bool SurfaceFlinger::handleMessageInvalidate() {
    ATRACE_CALL();
    bool refreshNeeded = handlePageFlip();
	ALOGI("zjj.rk3399.SF handleMessageInvalidate %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

    if (mVisibleRegionsDirty) {
        computeLayerBounds();
        if (mTracingEnabled) {
            mTracing.notify("visibleRegionsDirty");
        }
    }

    for (auto& layer : mLayersPendingRefresh) {
        Region visibleReg;
        visibleReg.set(layer->getScreenBounds());
        invalidateLayerStack(layer, visibleReg);
    }
    mLayersPendingRefresh.clear();
    return refreshNeeded;
}
```
主要是调用handlePageFlip，做Page的Flip。

``` cpp
bool SurfaceFlinger::handlePageFlip()
{
    ATRACE_CALL();
    //ALOGV("handlePageFlip");
	ALOGI("zjj.rk3399.SF handlePageFlip %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
    nsecs_t latchTime = systemTime();

    bool visibleRegions = false;
    bool frameQueued = false;
    bool newDataLatched = false;

    // Store the set of layers that need updates. This set must not change as
    // buffers are being latched, as this could result in a deadlock.
    // Example: Two producers share the same command stream and:
    // 1.) Layer 0 is latched
    // 2.) Layer 0 gets a new frame
    // 2.) Layer 1 gets a new frame
    // 3.) Layer 1 is latched.
    // Display is now waiting on Layer 1's frame, which is behind layer 0's
    // second frame. But layer 0's second frame could be waiting on display.
    mDrawingState.traverseInZOrder([&](Layer* layer) {
        if (layer->hasReadyFrame()) {
            frameQueued = true;
            const nsecs_t expectedPresentTime = getExpectedPresentTime();
            if (layer->shouldPresentNow(expectedPresentTime)) {
                mLayersWithQueuedFrames.push_back(layer);
            } else {
                ATRACE_NAME("!layer->shouldPresentNow()");
                layer->useEmptyDamage();
            }
        } else {
            layer->useEmptyDamage();
        }
    });

    if (!mLayersWithQueuedFrames.empty()) {
        // mStateLock is needed for latchBuffer as LayerRejecter::reject()
        // writes to Layer current state. See also b/119481871
        Mutex::Autolock lock(mStateLock);

        for (auto& layer : mLayersWithQueuedFrames) {
            if (layer->latchBuffer(visibleRegions, latchTime)) {
                mLayersPendingRefresh.push_back(layer);
            }
            layer->useSurfaceDamage();
            if (layer->isBufferLatched()) {
                newDataLatched = true;
            }
        }
    }

    mVisibleRegionsDirty |= visibleRegions;

    // If we will need to wake up at some time in the future to deal with a
    // queued frame that shouldn't be displayed during this vsync period, wake
    // up during the next vsync period to check again.
    if (frameQueued && (mLayersWithQueuedFrames.empty() || !newDataLatched)) {
        signalLayerUpdate();
    }

    // enter boot animation on first buffer latch
    if (CC_UNLIKELY(mBootStage == BootStage::BOOTLOADER && newDataLatched)) {
        ALOGI("Enter boot animation");
        mBootStage = BootStage::BOOTANIMATION;
    }

    // Only continue with the refresh if there is actually new work to do
    return !mLayersWithQueuedFrames.empty() && newDataLatched;
}
```
mLayersWithQueuedFrames，用于标记那些已经有Frame的Layer，这得从Layer的onFrameAvailable说起。

``` cpp
void BufferQueueLayer::onFrameAvailable(const BufferItem& item) {
    ATRACE_CALL();
    // Add this buffer from our internal queue tracker
    { // Autolock scope
		ALOGI("zjj.rk3399.SF onFrameAvailable %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

		if (mFlinger->mUseSmart90ForVideo) {
            const nsecs_t presentTime = item.mIsAutoTimestamp ? 0 : item.mTimestamp;
            mFlinger->mScheduler->addLayerPresentTimeAndHDR(mSchedulerLayerHandle, presentTime,
                                                            item.mHdrMetadata.validTypes != 0);
        }

        Mutex::Autolock lock(mQueueItemLock);
        // Reset the frame number tracker when we receive the first buffer after
        // a frame number reset
        if (item.mFrameNumber == 1) {
            mLastFrameNumberReceived = 0;
        }

        // Ensure that callbacks are handled in order
        while (item.mFrameNumber != mLastFrameNumberReceived + 1) {
            status_t result = mQueueItemCondition.waitRelative(mQueueItemLock, ms2ns(500));
            if (result != NO_ERROR) {
                ALOGE("[%s] Timed out waiting on callback", mName.string());
            }
        }

        mQueueItems.push_back(item);
        mQueuedFrames++;

        // Wake up any pending callbacks
        mLastFrameNumberReceived = item.mFrameNumber;
        mQueueItemCondition.broadcast();
    }

    mFlinger->mInterceptor->saveBufferUpdate(this, item.mGraphicBuffer->getWidth(),
                                             item.mGraphicBuffer->getHeight(), item.mFrameNumber);

    // If this layer is orphaned, then we run a fake vsync pulse so that
    // dequeueBuffer doesn't block indefinitely.
    if (isRemovedFromCurrentState()) {
        fakeVsync();
    } else {
        mFlinger->signalLayerUpdate();
    }
    mConsumer->onBufferAvailable(item);
}

```

onFrameAvailable时，先将Buffer的窗口属性保存在mInterceptor中，这里我们暂时不看，记得标记一下。然后对FrameNumber进行处理，一是确保FrameNumber被重置时，重置mLastFrameNumberReceived，二时，确保FrameNumber的顺序。之后，将新过来的BufferItem，push到mQueueItems中，对mQueuedFrames数进行+1。最后才触发SurfaceFlinger进行signalLayerUpdate。
回到handlePageFlip。所以，对于触发SurfaceFlinger进行signalLayerUpdate的Layer，hasQueuedFrame为true，是有Queued的Frame的。
但是mLayersWithQueuedFrames还要一个条件，shouldPresentNow。

``` cpp
bool BufferLayer::shouldPresentNow(const DispSync& dispSync) const {
    if (mSidebandStreamChanged || mAutoRefresh) {
        return true;
    }

    Mutex::Autolock lock(mQueueItemLock);
    if (mQueueItems.empty()) {
        return false;
    }
    auto timestamp = mQueueItems[0].mTimestamp;
    nsecs_t expectedPresent = mConsumer->computeExpectedPresent(dispSync);

    // Ignore timestamps more than a second in the future
    bool isPlausible = timestamp < (expectedPresent + s2ns(1));
    ALOGW_IF(!isPlausible,
             "[%s] Timestamp %" PRId64 " seems implausible "
             "relative to expectedPresent %" PRId64,
             mName.string(), timestamp, expectedPresent);

    bool isDue = timestamp < expectedPresent;
    return isDue || !isPlausible;
}
```
在shouldPresentNow的判断逻辑中，首先根据DispSync，去计算期望显示的时间。再看看Buffer的时间戳和期望显示的时间，如果Buffer的时间还没有到，且和期望显示的时间之间差不到1秒，那么shouldPresentNow成立。该Layer标记为mLayersWithQueuedFrames；否则，Layer使用空的DamageRegion。记住这个**DamageRegion**。

``` cpp
void BufferLayer::useEmptyDamage() {
    surfaceDamageRegion.clear();
}
```

继续handlePageFlip函数分析。

对mLayersWithQueuedFrames标记的Layer进行处理

首先，通过latchBuffer获取Layer的Buffer；再更新Surface的Damage；再通过invalidateLayerStack去刷新脏区域，验证LayerStack。记住LayerStack这个概念。处理这块稍后继续～～～
注意这里重新signalLayerUpdate的逻辑。

``` cpp
if (frameQueued && (mLayersWithQueuedFrames.empty() || !newDataLatched)) {
    signalLayerUpdate();
}
```
有BufferQueue过来，但是还没有到显示时间（mLayersWithQueuedFrames为空），或者没有获取到Buffer。重新触发一次更新～
注意handlePageFlip的返回值，有Layer要显示，且获取到Buffer时，才返回true。注意这里的**mVisibleRegionsDirty**，mVisibleRegionsDirty，脏区域，表示可见区域有更新。
##### 4.1、handlePageFlip获取Buffer
继续前面的Buffer的处理

``` cpp
    for (auto& layer : mLayersWithQueuedFrames) {
        const Region dirty(layer->latchBuffer(visibleRegions, latchTime));
        layer->useSurfaceDamage();
        invalidateLayerStack(layer, dirty);
        if (layer->isBufferLatched()) {
            newDataLatched = true;
        }
    }
```
- 获取 Buffer
Layer的latchBuffer函数比较长，这里将去获取Producer Queue过来的数据。我们分段来看：

``` cpp
* frameworks/native/services/surfaceflinger/BufferLayer.cpp
bool BufferLayer::latchBuffer(bool& recomputeVisibleRegions, nsecs_t latchTime) {
    ATRACE_CALL();
	ALOGI("zjj.rk3399.SF latchBuffer %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

    bool refreshRequired = latchSidebandStream(recomputeVisibleRegions);
   ....... 
    // we'll trigger an update in onPreComposition().
    if (mRefreshPending) {
        return false;
    }
   ....... 
    if (!fenceHasSignaled()) {
        ATRACE_NAME("!fenceHasSignaled()");
        mFlinger->signalLayerUpdate();
        return false;
    }

    // Capture the old state of the layer for comparisons later
    const State& s(getDrawingState());
    const bool oldOpacity = isOpaque(s);
    sp<GraphicBuffer> oldBuffer = mActiveBuffer;

    if (!allTransactionsSignaled()) {
        mFlinger->setTransactionFlags(eTraversalNeeded);
        return false;
    }

    status_t err = updateTexImage(recomputeVisibleRegions, latchTime);
    ....... 

    err = updateActiveBuffer();
   ....... 
    mBufferLatched = true;

    err = updateFrameNumber(latchTime);
   ....... 

    mRefreshPending = true;
    mFrameLatencyNeeded = true;
    if (oldBuffer == nullptr) {
        // the first time we receive a buffer, we need to trigger a
        // geometry invalidation.
        recomputeVisibleRegions = true;
    }

    ui::Dataspace dataSpace = getDrawingDataSpace();
    // translate legacy dataspaces to modern dataspaces
    switch (dataSpace) {
        case ui::Dataspace::SRGB:
            dataSpace = ui::Dataspace::V0_SRGB;
            break;
        case ui::Dataspace::SRGB_LINEAR:
            dataSpace = ui::Dataspace::V0_SRGB_LINEAR;
            break;
        case ui::Dataspace::JFIF:
            dataSpace = ui::Dataspace::V0_JFIF;
            break;
        case ui::Dataspace::BT601_625:
            dataSpace = ui::Dataspace::V0_BT601_625;
            break;
        case ui::Dataspace::BT601_525:
            dataSpace = ui::Dataspace::V0_BT601_525;
            break;
        case ui::Dataspace::BT709:
            dataSpace = ui::Dataspace::V0_BT709;
            break;
        default:
            break;
    }
    mCurrentDataSpace = dataSpace;

    Rect crop(getDrawingCrop());
    const uint32_t transform(getDrawingTransform());
    const uint32_t scalingMode(getDrawingScalingMode());
    const bool transformToDisplayInverse(getTransformToDisplayInverse());
    if ((crop != mCurrentCrop) || (transform != mCurrentTransform) ||
        (scalingMode != mCurrentScalingMode) ||
        (transformToDisplayInverse != mTransformToDisplayInverse)) {
        mCurrentCrop = crop;
        mCurrentTransform = transform;
        mCurrentScalingMode = scalingMode;
        mTransformToDisplayInverse = transformToDisplayInverse;
        recomputeVisibleRegions = true;
    }

    if (oldBuffer != nullptr) {
        uint32_t bufWidth = mActiveBuffer->getWidth();
        uint32_t bufHeight = mActiveBuffer->getHeight();
        if (bufWidth != uint32_t(oldBuffer->width) || bufHeight != uint32_t(oldBuffer->height)) {
            recomputeVisibleRegions = true;
        }
    }

    if (oldOpacity != isOpaque(s)) {
        recomputeVisibleRegions = true;
    }

    // Remove any sync points corresponding to the buffer which was just
    // latched
    {
        Mutex::Autolock lock(mLocalSyncPointMutex);
        auto point = mLocalSyncPoints.begin();
        while (point != mLocalSyncPoints.end()) {
            if (!(*point)->frameIsAvailable() || !(*point)->transactionIsApplied()) {
                // This sync point must have been added since we started
                // latching. Don't drop it yet.
                ++point;
                continue;
            }

            if ((*point)->getFrameNumber() <= mCurrentFrameNumber) {
                std::stringstream ss;
                ss << "Dropping sync point " << (*point)->getFrameNumber();
                ATRACE_NAME(ss.str().c_str());
                point = mLocalSyncPoints.erase(point);
            } else {
                ++point;
            }
        }
    }

    return true;
}
```
updateActiveBuffer()更新当前激活buffer。
看看Log：

``` cpp
10-28 09:41:00.211   179   179 I BufferLayer: zjj.rk3399.SF latchBuffer frameworks/native/services/surfaceflinger/BufferLayer.cpp latchBuffer 629 
10-28 09:41:00.211   179   179 I BufferLayer: zjj.rk3399.SF hasReadyFrame frameworks/native/services/surfaceflinger/BufferLayer.cpp hasReadyFrame 795 
10-28 09:41:00.211   179   179 I BufferQueueLayer: zjj.rk3399.SF updateTexImage frameworks/native/services/surfaceflinger/BufferQueueLayer.cpp updateTexImage 278 
10-28 09:41:00.211   179   179 V BufferQueueProducer: [test#0] query: 11? 0
10-28 09:41:00.211   179   179 V BufferLayerConsumer: [test#0] updateTexImage
10-28 09:41:00.211   179   179 I BufferLayerConsumer: zjj.rk3399.SF updateTexImage frameworks/native/services/surfaceflinger/BufferLayerConsumer.cpp updateTexImage 108 
10-28 09:41:00.211   179   179 I BufferLayerConsumer: zjj.rk3399.SF acquireBufferLocked frameworks/native/services/surfaceflinger/BufferLayerConsumer.cpp acquireBufferLocked 218 
10-28 09:41:00.211   179   179 V ConsumerBase: [test#0] acquireBufferLocked: -> slot=0/1
10-28 09:41:00.211   179   179 I BufferLayerConsumer: zjj.rk3399.SF updateAndReleaseLocked frameworks/native/services/surfaceflinger/BufferLayerConsumer.cpp updateAndReleaseLocked 243 
10-28 09:41:00.211   179   179 V BufferLayerConsumer: [test#0] updateAndRelease: (slot=-1 buf=0x0) -> (slot=0 buf=0x70f185f180)
10-28 09:41:00.211   179   179 V BufferLayerConsumer: [test#0] computeCurrentTransformMatrixLocked
10-28 09:41:00.211   179   179 I BufferLayerConsumer: zjj.rk3399.SF computeCurrentTransformMatrixLocked frameworks/native/services/surfaceflinger/BufferLayerConsumer.cpp computeCurrentTransformMatrixLocked 343 
10-28 09:41:00.211   179   179 V BufferLayerConsumer: [test#0] getFrameNumber
10-28 09:41:00.211   179   179 I BufferQueueLayer: zjj.rk3399.SF updateActiveBuffer frameworks/native/services/surfaceflinger/BufferQueueLayer.cpp updateActiveBuffer 383 
10-28 09:41:00.211   179   179 I BufferLayerConsumer: zjj.rk3399.SF getCurrentBuffer frameworks/native/services/surfaceflinger/BufferLayerConsumer.cpp getCurrentBuffer 402 
10-28 09:41:00.211   179   179 I BufferLayer: zjj.rk3399.SF getCompositionLayer frameworks/native/services/surfaceflinger/BufferLayer.cpp getCompositionLayer 939 
10-28 09:41:00.211   179   179 I BufferQueueLayer: zjj.rk3399.SF updateFrameNumber frameworks/native/services/surfaceflinger/BufferQueueLayer.cpp updateFrameNumber 398 
10-28 09:41:00.211   179   179 V BufferLayerConsumer: [test#0] getFrameNumber
10-28 09:41:00.211   179   179 V BufferLayerConsumer: [test#0] getCurrentDataSpace
```

接着调用BufferQueueLayer 的 updateTexImage()函数：

``` cpp
X:\frameworks\native\services\surfaceflinger\BufferQueueLayer.cpp
status_t BufferQueueLayer::updateTexImage(bool& recomputeVisibleRegions, nsecs_t latchTime) {
    // This boolean is used to make sure that SurfaceFlinger's shadow copy
    // of the buffer queue isn't modified when the buffer queue is returning
    // BufferItem's that weren't actually queued. This can happen in shared
    // buffer mode.
	ALOGI("zjj.rk3399.SF updateTexImage %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

    bool queuedBuffer = false;
    const int32_t layerID = getSequence();
    LayerRejecter r(mDrawingState, getCurrentState(), recomputeVisibleRegions,
                    getProducerStickyTransform() != 0, mName.string(), mOverrideScalingMode,
                    getTransformToDisplayInverse(), mFreezeGeometryUpdates);

    nsecs_t expectedPresentTime = mFlinger->getExpectedPresentTime();

    if (isRemovedFromCurrentState()) {
        expectedPresentTime = 0;
    }

    // updateTexImage() below might drop the some buffers at the head of the queue if there is a
    // buffer behind them which is timely to be presented. However this buffer may not be signaled
    // yet. The code below makes sure that this wouldn't happen by setting maxFrameNumber to the
    // last buffer that was signaled.
    uint64_t lastSignaledFrameNumber = mLastFrameNumberReceived;
    {
        Mutex::Autolock lock(mQueueItemLock);
        for (int i = 0; i < mQueueItems.size(); i++) {
            bool fenceSignaled =
                    mQueueItems[i].mFenceTime->getSignalTime() != Fence::SIGNAL_TIME_PENDING;
            if (!fenceSignaled) {
                break;
            }
            lastSignaledFrameNumber = mQueueItems[i].mFrameNumber;
        }
    }
    const uint64_t maxFrameNumberToAcquire =
            std::min(mLastFrameNumberReceived.load(), lastSignaledFrameNumber);

    status_t updateResult = mConsumer->updateTexImage(&r, expectedPresentTime, &mAutoRefresh,
                                                      &queuedBuffer, maxFrameNumberToAcquire);
    if (updateResult == BufferQueue::PRESENT_LATER) {
        // Producer doesn't want buffer to be displayed yet.  Signal a
        // layer update so we check again at the next opportunity.
        mFlinger->signalLayerUpdate();
        return BAD_VALUE;
    } else if (updateResult == BufferLayerConsumer::BUFFER_REJECTED) {
        // If the buffer has been rejected, remove it from the shadow queue
        // and return early
        if (queuedBuffer) {
            Mutex::Autolock lock(mQueueItemLock);
            mConsumer->mergeSurfaceDamage(mQueueItems[0].mSurfaceDamage);
            mFlinger->mTimeStats->removeTimeRecord(layerID, mQueueItems[0].mFrameNumber);
            mQueueItems.removeAt(0);
            mQueuedFrames--;
        }
        return BAD_VALUE;
    } else if (updateResult != NO_ERROR || mUpdateTexImageFailed) {
        // This can occur if something goes wrong when trying to create the
        // EGLImage for this buffer. If this happens, the buffer has already
        // been released, so we need to clean up the queue and bug out
        // early.
        if (queuedBuffer) {
            Mutex::Autolock lock(mQueueItemLock);
            mQueueItems.clear();
            mQueuedFrames = 0;
            mFlinger->mTimeStats->onDestroy(layerID);
        }

        // Once we have hit this state, the shadow queue may no longer
        // correctly reflect the incoming BufferQueue's contents, so even if
        // updateTexImage starts working, the only safe course of action is
        // to continue to ignore updates.
        mUpdateTexImageFailed = true;

        return BAD_VALUE;
    }

    if (queuedBuffer) {
        // Autolock scope
        auto currentFrameNumber = mConsumer->getFrameNumber();

        Mutex::Autolock lock(mQueueItemLock);

        // Remove any stale buffers that have been dropped during
        // updateTexImage
        while (mQueueItems[0].mFrameNumber != currentFrameNumber) {
            mConsumer->mergeSurfaceDamage(mQueueItems[0].mSurfaceDamage);
            mFlinger->mTimeStats->removeTimeRecord(layerID, mQueueItems[0].mFrameNumber);
            mQueueItems.removeAt(0);
            mQueuedFrames--;
        }

        mFlinger->mTimeStats->setAcquireFence(layerID, currentFrameNumber,
                                              mQueueItems[0].mFenceTime);
        mFlinger->mTimeStats->setLatchTime(layerID, currentFrameNumber, latchTime);

        mQueueItems.removeAt(0);
    }

    // Decrement the queued-frames count.  Signal another event if we
    // have more frames pending.
    if ((queuedBuffer && mQueuedFrames.fetch_sub(1) > 1) || mAutoRefresh) {
        mFlinger->signalLayerUpdate();
    }

    return NO_ERROR;
}
```

看看BufferLayerConsumer::updateTexImage

``` cpp
X:\frameworks\native\services\surfaceflinger\BufferLayerConsumer.cpp
status_t BufferLayerConsumer::updateTexImage(BufferRejecter* rejecter, nsecs_t expectedPresentTime,
                                             bool* autoRefresh, bool* queuedBuffer,
                                             uint64_t maxFrameNumber) {
    ATRACE_CALL();
    BLC_LOGV("updateTexImage");
	ALOGI("zjj.rk3399.SF updateTexImage %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

	Mutex::Autolock lock(mMutex);

    if (mAbandoned) {
        BLC_LOGE("updateTexImage: BufferLayerConsumer is abandoned!");
        return NO_INIT;
    }

    BufferItem item;

    // Acquire the next buffer.
    // In asynchronous mode the list is guaranteed to be one buffer
    // deep, while in synchronous mode we use the oldest buffer.
    status_t err = acquireBufferLocked(&item, expectedPresentTime, maxFrameNumber);
    if (err != NO_ERROR) {
        if (err == BufferQueue::NO_BUFFER_AVAILABLE) {
            err = NO_ERROR;
        } else if (err == BufferQueue::PRESENT_LATER) {
            // return the error, without logging
        } else {
            BLC_LOGE("updateTexImage: acquire failed: %s (%d)", strerror(-err), err);
        }
        return err;
    }

    if (autoRefresh) {
        *autoRefresh = item.mAutoRefresh;
    }

    if (queuedBuffer) {
        *queuedBuffer = item.mQueuedBuffer;
    }

    // We call the rejecter here, in case the caller has a reason to
    // not accept this buffer.  This is used by SurfaceFlinger to
    // reject buffers which have the wrong size
    int slot = item.mSlot;
    if (rejecter && rejecter->reject(mSlots[slot].mGraphicBuffer, item)) {
        releaseBufferLocked(slot, mSlots[slot].mGraphicBuffer);
        return BUFFER_REJECTED;
    }

    // Release the previous buffer.
    err = updateAndReleaseLocked(item, &mPendingRelease);
    if (err != NO_ERROR) {
        return err;
    }

    if (!mRE.useNativeFenceSync()) {
        // Bind the new buffer to the GL texture.
        //
        // Older devices require the "implicit" synchronization provided
        // by glEGLImageTargetTexture2DOES, which this method calls.  Newer
        // devices will either call this in Layer::onDraw, or (if it's not
        // a GL-composited layer) not at all.
        err = bindTextureImageLocked();
    }

    return err;
}

status_t BufferLayerConsumer::updateAndReleaseLocked(const BufferItem& item,
                                                     PendingRelease* pendingRelease) {
    status_t err = NO_ERROR;

    int slot = item.mSlot;
	ALOGI("zjj.rk3399.SF updateAndReleaseLocked %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

    BLC_LOGV("updateAndRelease: (slot=%d buf=%p) -> (slot=%d buf=%p)", mCurrentTexture,
             (mCurrentTextureBuffer != nullptr && mCurrentTextureBuffer->graphicBuffer() != nullptr)
                     ? mCurrentTextureBuffer->graphicBuffer()->handle
                     : 0,
             slot, mSlots[slot].mGraphicBuffer->handle);

    // Hang onto the pointer so that it isn't freed in the call to
    // releaseBufferLocked() if we're in shared buffer mode and both buffers are
    // the same.

    std::shared_ptr<Image> nextTextureBuffer;
    {
        std::lock_guard<std::mutex> lock(mImagesMutex);
        nextTextureBuffer = mImages[slot];
    }

    // release old buffer
    if (mCurrentTexture != BufferQueue::INVALID_BUFFER_SLOT) {
        if (pendingRelease == nullptr) {
            status_t status =
                    releaseBufferLocked(mCurrentTexture, mCurrentTextureBuffer->graphicBuffer());
            if (status < NO_ERROR) {
                BLC_LOGE("updateAndRelease: failed to release buffer: %s (%d)", strerror(-status),
                         status);
                err = status;
                // keep going, with error raised [?]
            }
        } else {
            pendingRelease->currentTexture = mCurrentTexture;
            pendingRelease->graphicBuffer = mCurrentTextureBuffer->graphicBuffer();
            pendingRelease->isPending = true;
        }
    }

    // Update the BufferLayerConsumer state.
    mCurrentTexture = slot;
    mCurrentTextureBuffer = nextTextureBuffer;
    mCurrentCrop = item.mCrop;
    mCurrentTransform = item.mTransform;
    mCurrentScalingMode = item.mScalingMode;
    mCurrentTimestamp = item.mTimestamp;
    mCurrentDataSpace = static_cast<ui::Dataspace>(item.mDataSpace);
    mCurrentHdrMetadata = item.mHdrMetadata;
    mCurrentFence = item.mFence;
    mCurrentFenceTime = item.mFenceTime;
    mCurrentFrameNumber = item.mFrameNumber;
    mCurrentTransformToDisplayInverse = item.mTransformToDisplayInverse;
    mCurrentSurfaceDamage = item.mSurfaceDamage;
    mCurrentApi = item.mApi;

    computeCurrentTransformMatrixLocked();

    return err;
}

status_t BufferLayerConsumer::acquireBufferLocked(BufferItem* item, nsecs_t presentWhen,
                                                  uint64_t maxFrameNumber) {
	ALOGI("zjj.rk3399.SF acquireBufferLocked %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

    status_t err = ConsumerBase::acquireBufferLocked(item, presentWhen, maxFrameNumber);
    if (err != NO_ERROR) {
        return err;
    }

    // If item->mGraphicBuffer is not null, this buffer has not been acquired
    // before, so we need to clean up old references.
    if (item->mGraphicBuffer != nullptr) {
        std::lock_guard<std::mutex> lock(mImagesMutex);
        if (mImages[item->mSlot] == nullptr || mImages[item->mSlot]->graphicBuffer() == nullptr ||
            mImages[item->mSlot]->graphicBuffer()->getId() != item->mGraphicBuffer->getId()) {
            mImages[item->mSlot] = std::make_shared<Image>(item->mGraphicBuffer, mRE);
        }
    }

    return NO_ERROR;
}

status_t BufferLayerConsumer::bindTextureImageLocked() {
    ATRACE_CALL();
	ALOGI("zjj.rk3399.SF bindTextureImageLocked %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

    if (mCurrentTextureBuffer != nullptr && mCurrentTextureBuffer->graphicBuffer() != nullptr) {
        return mRE.bindExternalTextureBuffer(mTexName, mCurrentTextureBuffer->graphicBuffer(),
                                             mCurrentFence);
    }

    return NO_INIT;
}


```



invalidateLayerStack的处理如下：

``` cpp
void SurfaceFlinger::invalidateLayerStack(const sp<const Layer>& layer, const Region& dirty) {
    for (size_t dpy=0 ; dpy<mDisplays.size() ; dpy++) {
        const sp<DisplayDevice>& hw(mDisplays[dpy]);
        if (layer->belongsToDisplay(hw->getLayerStack(), hw->isPrimary())) {
            hw->dirtyRegion.orSelf(dirty);
        }
    }
}
```
layerStack，Layer的栈，Android支持多个屏幕，layer可以定制化的只显示到某个显示屏幕上。其中就是靠layerStack来实现的。Layer的stack值如果和DisplayDevice的stack值一样，那说明这个layer是属于这个显示屏幕的。
INVALIDATE消息处理，基本完成。如果需要刷新，触发刷新的消息：

``` cpp
X:\frameworks\native\services\surfaceflinger\SurfaceFlinger.cpp
            bool refreshNeeded = handleMessageTransaction();
            refreshNeeded |= handleMessageInvalidate();

            updateCursorAsync();
            updateInputFlinger();

            refreshNeeded |= mRepaintEverything;
            if (refreshNeeded && CC_LIKELY(mBootStage != BootStage::BOOTLOADER)) {
                // Signal a refresh if a transaction modified the window state,
                // a new buffer was latched, or if HWC has requested a full
                // repaint
                signalRefresh();
            }
            break;
```
什么时候需要刷新？

- 有新的Transaction处理
- PageFlip时，有Buffer更新！～
- 有重新合成请求时mRepaintEverything，这是响应HWC的请求时触发的。
刷新消息REFRESH处理
SurfaceFlinger收到了VSync信号后，调用了handleMessageRefresh函数

``` cpp
[SurfaceFlinger.cpp]
void SurfaceFlinger::handleMessageRefresh() {
    ATRACE_CALL();
	ALOGI("zjj.rk3399.SF handleMessageRefresh %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

    mRefreshPending = false;

    const bool repaintEverything = mRepaintEverything.exchange(false);
    preComposition();
    rebuildLayerStacks();
    calculateWorkingSet();
    for (const auto& [token, display] : mDisplays) {
        beginFrame(display);
        prepareFrame(display);
        doDebugFlashRegions(display, repaintEverything);
        doComposition(display, repaintEverything);
    }

    // For rk:
    // Adapt to HWC1,all displays call postFramebuffer after prepareFrame
    for (const auto& [token, display] : mDisplays) {
        postFramebuffer(display);
    }

    logLayerStats();

    postFrame();
    postComposition();

    mHadClientComposition = false;
    mHadDeviceComposition = false;
    for (const auto& [token, displayDevice] : mDisplays) {
        auto display = displayDevice->getCompositionDisplay();
        const auto displayId = display->getId();
        mHadClientComposition =
                mHadClientComposition || getHwComposer().hasClientComposition(displayId);
        mHadDeviceComposition =
                mHadDeviceComposition || getHwComposer().hasDeviceComposition(displayId);
    }

    mVsyncModulator.onRefreshed(mHadClientComposition);

    mLayersWithQueuedFrames.clear();
}
```

handleMessageRefresh 函数中，包含了刷新一帧显示数据所有的流程。下面我们分别来进行说明。

###  一、preComposition()函数

``` cpp
void SurfaceFlinger::preComposition()
{
    ATRACE_CALL();
	ALOGI("zjj.rk3399.SF preComposition %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

    mRefreshStartTime = systemTime(SYSTEM_TIME_MONOTONIC);

    bool needExtraInvalidate = false;
    mDrawingState.traverseInZOrder([&](Layer* layer) {
        if (layer->onPreComposition(mRefreshStartTime)) {
            needExtraInvalidate = true;
        }
    });

    if (needExtraInvalidate) {
        signalLayerUpdate();
    }
}
```

在合成前，先遍历所有需要进行合成的Layer，调Layer的onPreComposition方法。

ColorLayer的onPreComposition，返回值是固定的，为true；BufferLayer的onPreComposition如下：


``` cpp
bool BufferLayer::onPreComposition(nsecs_t refreshStartTime) {
	ALOGI("zjj.rk3399.SF onPreComposition %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

    if (mBufferLatched) {
        Mutex::Autolock lock(mFrameEventHistoryMutex);
        mFrameEventHistory.addPreComposition(mCurrentFrameNumber, refreshStartTime);
    }
    mRefreshPending = false;
    return hasReadyFrame();
}
bool BufferLayer::hasReadyFrame() const {
	ALOGI("zjj.rk3399.SF hasReadyFrame %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

    return hasFrameUpdate() || getSidebandStreamChanged() || getAutoRefresh();
}
```
重构Layer的Stack rebuildLayerStacks
现在，我们需要合成显示的Layer数据，都保存在mDrawingState的layersSortedByZ中，且是按照z-order的顺序进行存放。那么rebuild Layer又是做什么呢？
### 二、rebuildLayerStacks()函数

``` cpp
void SurfaceFlinger::rebuildLayerStacks() {
    ATRACE_CALL();
	ALOGI("zjj.rk3399.SF rebuildLayerStacks %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

    // rebuild the visible layer list per screen
    if (CC_UNLIKELY(mVisibleRegionsDirty)) {
        ATRACE_NAME("rebuildLayerStacks VR Dirty");
        mVisibleRegionsDirty = false;
        invalidateHwcGeometry();

        for (const auto& pair : mDisplays) {
            const auto& displayDevice = pair.second;
            auto display = displayDevice->getCompositionDisplay();
            const auto& displayState = display->getState();
            Region opaqueRegion;
            Region dirtyRegion;
            compositionengine::Output::OutputLayers layersSortedByZ;
            Vector<sp<Layer>> deprecated_layersSortedByZ;
            Vector<sp<Layer>> layersNeedingFences;
            const ui::Transform& tr = displayState.transform;
            const Rect bounds = displayState.bounds;
            if (displayState.isEnabled) {
                computeVisibleRegions(displayDevice, dirtyRegion, opaqueRegion);

               ......
    }
}

```


``` cpp
void SurfaceFlinger::computeVisibleRegions(const sp<const DisplayDevice>& displayDevice,
                                           Region& outDirtyRegion, Region& outOpaqueRegion) {
    ATRACE_CALL();
    //ALOGV("computeVisibleRegions");
	ALOGI("zjj.rk3399.SF computeVisibleRegions %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

    auto display = displayDevice->getCompositionDisplay();

    Region aboveOpaqueLayers;
    Region aboveCoveredLayers;
    Region dirty;

    outDirtyRegion.clear();

    mDrawingState.traverseInReverseZOrder([&](Layer* layer) {
        // start with the whole surface at its current location
        const Layer::State& s(layer->getDrawingState());

        // only consider the layers on the given layer stack
        if (!display->belongsInOutput(layer->getLayerStack(), layer->getPrimaryDisplayOnly())) {
            return;
        }

        /*
         * opaqueRegion: area of a surface that is fully opaque.
         */
        Region opaqueRegion;

        /*
         * visibleRegion: area of a surface that is visible on screen
         * and not fully transparent. This is essentially the layer's
         * footprint minus the opaque regions above it.
         * Areas covered by a translucent surface are considered visible.
         */
        Region visibleRegion;

        /*
         * coveredRegion: area of a surface that is covered by all
         * visible regions above it (which includes the translucent areas).
         */
        Region coveredRegion;

        /*
         * transparentRegion: area of a surface that is hinted to be completely
         * transparent. This is only used to tell when the layer has no visible
         * non-transparent regions and can be removed from the layer list. It
         * does not affect the visibleRegion of this layer or any layers
         * beneath it. The hint may not be correct if apps don't respect the
         * SurfaceView restrictions (which, sadly, some don't).
         */
        Region transparentRegion;
```
我们先来看Display的几个关于区域的概念：

- 脏区域 dirtyRegion
计算脏区域时，outDirtyRegion先清空～  然后遍历mDrawingState中的Layer，如果Layer不属于Display，那么就返回了，outDirtyRegion为空。
- 非透明区域 opaqueRegion
Surface(Layer)完全不透明的区域
可见区域 visibleRegion
Layer可以被看见的区域，包括不完全透明的区域。原则上，这就是整个Surface减去非透明区域。
- 被覆盖区域 coveredRegion
Surface被上面的Surface覆盖的区域，包括被透明区域覆盖的区域。
- 透明区域 transparentRegion
Surface完全透明的部分，如果没有可见的非透明区域，这个Layer就可以从Layer列表中删掉。它并不影响该Layer本身或其下方Layer的可见区域大小。这个区域可能不太准，如果App不遵守SurfaceView的限制，可悲的是，确实有不遵守的。

回到computeVisibleRegions函数，其是按照z-order进行反序号遍历的，所以从最上面开始遍历。

``` cpp
        // handle hidden surfaces by setting the visible region to empty
        if (CC_LIKELY(layer->isVisible())) {
            const bool translucent = !layer->isOpaque(s);
            Rect bounds(layer->getScreenBounds());

            visibleRegion.set(bounds);
            ui::Transform tr = layer->getTransform();
            if (!visibleRegion.isEmpty()) {
                // Remove the transparent area from the visible region
                if (translucent) {
                    if (tr.preserveRects()) {
                        // transform the transparent region
                        transparentRegion = tr.transform(layer->getActiveTransparentRegion(s));
                    } else {
                        // transformation too complex, can't do the
                        // transparent region optimization.
                        transparentRegion.clear();
                    }
                }

                // compute the opaque region
                const int32_t layerOrientation = tr.getOrientation();
                if (layer->getAlpha() == 1.0f && !translucent &&
                        layer->getRoundedCornerState().radius == 0.0f &&
                        ((layerOrientation & ui::Transform::ROT_INVALID) == false)) {
                    // the opaque region is the layer's footprint
                    opaqueRegion = visibleRegion;
                }
            }
        }

        if (visibleRegion.isEmpty()) {
            layer->clearVisibilityRegions();
            return;
        }

        // Clip the covered region to the visible region
        coveredRegion = aboveCoveredLayers.intersect(visibleRegion);

        // Update aboveCoveredLayers for next (lower) layer
        aboveCoveredLayers.orSelf(visibleRegion);

        // subtract the opaque region covered by the layers above us
        visibleRegion.subtractSelf(aboveOpaqueLayers);

        // compute this layer's dirty region
        if (layer->contentDirty) {
            // we need to invalidate the whole region
            dirty = visibleRegion;
            // as well, as the old visible region
            dirty.orSelf(layer->visibleRegion);
            layer->contentDirty = false;
        } else {
            /* compute the exposed region:
             *   the exposed region consists of two components:
             *   1) what's VISIBLE now and was COVERED before
             *   2) what's EXPOSED now less what was EXPOSED before
             *
             * note that (1) is conservative, we start with the whole
             * visible region but only keep what used to be covered by
             * something -- which mean it may have been exposed.
             *
             * (2) handles areas that were not covered by anything but got
             * exposed because of a resize.
             */
            const Region newExposed = visibleRegion - coveredRegion;
            const Region oldVisibleRegion = layer->visibleRegion;
            const Region oldCoveredRegion = layer->coveredRegion;
            const Region oldExposed = oldVisibleRegion - oldCoveredRegion;
            dirty = (visibleRegion&oldCoveredRegion) | (newExposed-oldExposed);
        }
        dirty.subtractSelf(aboveOpaqueLayers);

        // accumulate to the screen dirty region
        outDirtyRegion.orSelf(dirty);

        // Update aboveOpaqueLayers for next (lower) layer
        aboveOpaqueLayers.orSelf(opaqueRegion);

        // Store the visible region in screen space
        layer->setVisibleRegion(visibleRegion);
        layer->setCoveredRegion(coveredRegion);
        layer->setVisibleNonTransparentRegion(
                visibleRegion.subtract(transparentRegion));
    });

    outOpaqueRegion = aboveOpaqueLayers;
}
```
 getTransform 获取 Layer的变换矩阵
屏幕有旋转，需要做变换，去适配显示屏幕

回到 computeVisibleRegions函数，计算完可见区域，计算非透明区域，一般情况下，如果layer是非透明的，非透明区域就是可见区域
outDirtyRegion 是屏幕的脏区域，它是每个Layer脏区域的合。最后将计算好的区域值设置到Layer中。outOpaqueRegion是屏幕的非透明区域。

- setVisibleRegion 设置可见区域
- setCoveredRegion 设置被覆盖的区域
- setVisibleNonTransparentRegion 设置可见的非透明区域，它是可见区域，减去透明区域。

回到rebuildLayerStacks函数～ computeVisibleRegions结束后，屏幕的脏区域得到了，每个Layer的可见区域，被覆盖的区域，以及可见非透明区域都计算出来了。



``` cpp
void SurfaceFlinger::rebuildLayerStacks() {
    ATRACE_CALL();
	ALOGI("zjj.rk3399.SF rebuildLayerStacks %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

    // rebuild the visible layer list per screen
    if (CC_UNLIKELY(mVisibleRegionsDirty)) {
        ATRACE_NAME("rebuildLayerStacks VR Dirty");
        mVisibleRegionsDirty = false;
        invalidateHwcGeometry();

        for (const auto& pair : mDisplays) {
            const auto& displayDevice = pair.second;
            auto display = displayDevice->getCompositionDisplay();
            const auto& displayState = display->getState();
            Region opaqueRegion;
            Region dirtyRegion;
            compositionengine::Output::OutputLayers layersSortedByZ;
            Vector<sp<Layer>> deprecated_layersSortedByZ;
            Vector<sp<Layer>> layersNeedingFences;
            const ui::Transform& tr = displayState.transform;
            const Rect bounds = displayState.bounds;
            if (displayState.isEnabled) {
                computeVisibleRegions(displayDevice, dirtyRegion, opaqueRegion);

                mDrawingState.traverseInZOrder([&](Layer* layer) {
                    auto compositionLayer = layer->getCompositionLayer();
                    if (compositionLayer == nullptr) {
                        return;
                    }

                    const auto displayId = displayDevice->getId();
                    sp<compositionengine::LayerFE> layerFE = compositionLayer->getLayerFE();
                    LOG_ALWAYS_FATAL_IF(layerFE.get() == nullptr);

                    bool needsOutputLayer = false;

                    if (display->belongsInOutput(layer->getLayerStack(),
                                                 layer->getPrimaryDisplayOnly())) {
                        Region drawRegion(tr.transform(
                                layer->visibleNonTransparentRegion));
                        drawRegion.andSelf(bounds);
                        if (!drawRegion.isEmpty()) {
                            needsOutputLayer = true;
                        }
                    }

                    if (needsOutputLayer) {
                        layersSortedByZ.emplace_back(
                                display->getOrCreateOutputLayer(displayId, compositionLayer,
                                                                layerFE));
                        deprecated_layersSortedByZ.add(layer);

                        auto& outputLayerState = layersSortedByZ.back()->editState();
                        outputLayerState.visibleRegion =
                                tr.transform(layer->visibleRegion.intersect(displayState.viewport));
                    } else if (displayId) {
                        // For layers that are being removed from a HWC display,
                        // and that have queued frames, add them to a a list of
                        // released layers so we can properly set a fence.
                        bool hasExistingOutputLayer =
                                display->getOutputLayerForLayer(compositionLayer.get()) != nullptr;
                        bool hasQueuedFrames = std::find(mLayersWithQueuedFrames.cbegin(),
                                                         mLayersWithQueuedFrames.cend(),
                                                         layer) != mLayersWithQueuedFrames.cend();

                        if (hasExistingOutputLayer && hasQueuedFrames) {
                            layersNeedingFences.add(layer);
                        }
                    }
                });
            }

            display->setOutputLayersOrderedByZ(std::move(layersSortedByZ));

            displayDevice->setVisibleLayersSortedByZ(deprecated_layersSortedByZ);
            displayDevice->setLayersNeedingFences(layersNeedingFences);

            Region undefinedRegion{bounds};
            undefinedRegion.subtractSelf(tr.transform(opaqueRegion));

            display->editState().undefinedRegion = undefinedRegion;
            display->editState().dirtyRegion.orSelf(dirtyRegion);
        }
    }
}
```

rebuildLayerStacks函数中对Layer再遍历一次，这次是正序，也就是从下往上。遍历时，主要做了一下处理：

- 计算Layer需要绘制的区域drawRegion，将Layer的可见区域和Display的大小做交集而得到
- 如果drawRegion不为空，将该Layer加到当前Display的Layer列表中，也是按照z-order进行存放layersSortedByZ。每个Display有自己的layersSortedByZ。
- 如果之前Layer是可见的，现在不可见，销毁掉hwc Layer。销毁的Layer放到layersNeedingFences中，它虽然不需要releaseFence，但是还是需要fence去释放旧的Buffer。

rebuildLayerStacks的最后，将数据更新到Display中。

``` cpp
void SurfaceFlinger::rebuildLayerStacks() {
    ... ...

        for (const auto& pair : mDisplays) {
            ... ...
            display->setOutputLayersOrderedByZ(std::move(layersSortedByZ));

            displayDevice->setVisibleLayersSortedByZ(deprecated_layersSortedByZ);
            displayDevice->setLayersNeedingFences(layersNeedingFences);

            Region undefinedRegion{bounds};
            undefinedRegion.subtractSelf(tr.transform(opaqueRegion));

            display->editState().undefinedRegion = undefinedRegion;
            display->editState().dirtyRegion.orSelf(dirtyRegion);
        }
    }
}
```
Display中还有一个区域，叫未定义的区域。也就是屏幕的大小减去屏幕的非透明区域opaqueRegion余下的部分。

创建Layer栈完成，此时需要进行合成显示的数据已经被更新到每个Display各自的layersSortedByZ中。

### 三、calculateWorkingSet()函数

``` cpp
void SurfaceFlinger::calculateWorkingSet() {
    ATRACE_CALL();
    ALOGV(__FUNCTION__);

    // 建立HWC中的Layer列表
    if (CC_UNLIKELY(mGeometryInvalid)) {
        mGeometryInvalid = false;
        // 同样需要针对各个Display做处理
        for (const auto& [token, displayDevice] : mDisplays) {
            auto display = displayDevice->getCompositionDisplay();

            uint32_t zOrder = 0;

            // 按照Z轴的顺序，依次给可见OutputLayer在HWC中建立映射
            for (auto& layer : display->getOutputLayersOrderedByZ()) {
                auto& compositionState = layer->editState();
                compositionState.forceClientComposition = false;
                if (!compositionState.hwc || mDebugDisableHWC || mDebugRegion) {
                    compositionState.forceClientComposition = true;
                }

                // Z轴顺序依次递增
                compositionState.z = zOrder++;

                // 更新与显示无关的合成状态，其实就是将Layer的状态信息放在CompositionState中了。
                // 也就是frontEnd（LayerFECompositionState）中
                layer->getLayerFE().latchCompositionState(layer->getLayer().editState().frontEnd,
                                                          true);

                // 重新计算OutputLayer的几何状态
                // 比如根据显示屏全局矩阵调整该Layer的DisplayFrame、
                // 变换窗口裁剪以匹配缓冲区坐标系等等。
                layer->updateCompositionState(true);

                // 将Layer更新完毕的几何状态写入HWC
                layer->writeStateToHWC(true);
            }
        }
    }

    // 设置每帧的数据
    for (const auto& [token, displayDevice] : mDisplays) {
        auto display = displayDevice->getCompositionDisplay();
        const auto displayId = display->getId();
        if (!displayId) {
            continue;
        }
        auto* profile = display->getDisplayColorProfile();

        if (mDrawingState.colorMatrixChanged) {
            display->setColorTransform(mDrawingState.colorMatrix);
        }
        Dataspace targetDataspace = Dataspace::UNKNOWN;
        if (useColorManagement) {
            ColorMode colorMode;
            RenderIntent renderIntent;
            pickColorMode(displayDevice, &colorMode, &targetDataspace, &renderIntent);
            display->setColorMode(colorMode, targetDataspace, renderIntent);
        }
        for (auto& layer : displayDevice->getVisibleLayersSortedByZ()) {
            ......
            const auto& displayState = display->getState();

            // 将Layer的mActiveBuffer设置到HWComposer中
            layer->setPerFrameData(displayDevice, displayState.transform, displayState.viewport,
                                   displayDevice->getSupportedPerFrameMetadata(),
                                   isHdrColorMode(displayState.colorMode) ? Dataspace::UNKNOWN
                                                                          : targetDataspace);
        }
    }

    mDrawingState.colorMatrixChanged = false;

    for (const auto& [token, displayDevice] : mDisplays) {
        auto display = displayDevice->getCompositionDisplay();
        for (auto& layer : displayDevice->getVisibleLayersSortedByZ()) {
            auto& layerState = layer->getCompositionLayer()->editState().frontEnd;
            layerState.compositionType = static_cast<Hwc2::IComposerClient::Composition>(
                    layer->getCompositionType(displayDevice));
        }
    }
}

```
这里建立HWC中的Layer列表：

按照Z轴的顺序，依次给可见OutputLayer在HWC中建立映射也就是将Layer的状态信息放在CompositionState中，并重新计算OutputLayer的几何状态，写入HWC将Layer的mActiveBuffer设置到HWComposer中
#### 3.1、setPerFrameData()

``` cpp
X:\frameworks\native\services\surfaceflinger\BufferLayer.cpp
void BufferLayer::setPerFrameData(const sp<const DisplayDevice>& displayDevice,
                                  const ui::Transform& transform, const Rect& viewport,
                                  int32_t supportedPerFrameMetadata,
                                  const ui::Dataspace targetDataspace) {
    RETURN_IF_NO_HWC_LAYER(displayDevice);
	ALOGI("zjj.rk3399.SF setPerFrameData %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

    // Apply this display's projection's viewport to the visible region
    // before giving it to the HWC HAL.
    Region visible = transform.transform(visibleRegion.intersect(viewport));

    const auto outputLayer = findOutputLayerForDisplay(displayDevice);
    LOG_FATAL_IF(!outputLayer || !outputLayer->getState().hwc);

    auto& hwcLayer = (*outputLayer->getState().hwc).hwcLayer;
    auto error = hwcLayer->setVisibleRegion(visible);
    if (error != HWC2::Error::None) {
        ALOGE("[%s] Failed to set visible region: %s (%d)", mName.string(),
              to_string(error).c_str(), static_cast<int32_t>(error));
        visible.dump(LOG_TAG);
    }
    outputLayer->editState().visibleRegion = visible;

    auto& layerCompositionState = getCompositionLayer()->editState().frontEnd;

    error = hwcLayer->setSurfaceDamage(surfaceDamageRegion);
    if (error != HWC2::Error::None) {
        ALOGE("[%s] Failed to set surface damage: %s (%d)", mName.string(),
              to_string(error).c_str(), static_cast<int32_t>(error));
        surfaceDamageRegion.dump(LOG_TAG);
    }
    layerCompositionState.surfaceDamage = surfaceDamageRegion;

    // Sideband layers
    if (layerCompositionState.sidebandStream.get()) {
        setCompositionType(displayDevice, Hwc2::IComposerClient::Composition::SIDEBAND);
        ALOGV("[%s] Requesting Sideband composition", mName.string());
        error = hwcLayer->setSidebandStream(layerCompositionState.sidebandStream->handle());
        if (error != HWC2::Error::None) {
            ALOGE("[%s] Failed to set sideband stream %p: %s (%d)", mName.string(),
                  layerCompositionState.sidebandStream->handle(), to_string(error).c_str(),
                  static_cast<int32_t>(error));
        }
        layerCompositionState.compositionType = Hwc2::IComposerClient::Composition::SIDEBAND;
        return;
    }

    // Device or Cursor layers
    if (mPotentialCursor) {
        ALOGV("[%s] Requesting Cursor composition", mName.string());
        setCompositionType(displayDevice, Hwc2::IComposerClient::Composition::CURSOR);
    } else {
        ALOGV("[%s] Requesting Device composition", mName.string());
        setCompositionType(displayDevice, Hwc2::IComposerClient::Composition::DEVICE);
    }

    ui::Dataspace dataspace = isColorSpaceAgnostic() && targetDataspace != ui::Dataspace::UNKNOWN
            ? targetDataspace
            : mCurrentDataSpace;
    error = hwcLayer->setDataspace(dataspace);
    if (error != HWC2::Error::None) {
        ALOGE("[%s] Failed to set dataspace %d: %s (%d)", mName.string(), dataspace,
              to_string(error).c_str(), static_cast<int32_t>(error));
    }

    const HdrMetadata& metadata = getDrawingHdrMetadata();
    error = hwcLayer->setPerFrameMetadata(supportedPerFrameMetadata, metadata);
    if (error != HWC2::Error::None && error != HWC2::Error::Unsupported) {
        ALOGE("[%s] Failed to set hdrMetadata: %s (%d)", mName.string(),
              to_string(error).c_str(), static_cast<int32_t>(error));
    }

    error = hwcLayer->setColorTransform(getColorTransform());
    if (error == HWC2::Error::Unsupported) {
        // If per layer color transform is not supported, we use GPU composition.
        setCompositionType(displayDevice, Hwc2::IComposerClient::Composition::CLIENT);
    } else if (error != HWC2::Error::None) {
        ALOGE("[%s] Failed to setColorTransform: %s (%d)", mName.string(),
                to_string(error).c_str(), static_cast<int32_t>(error));
    }
    layerCompositionState.dataspace = mCurrentDataSpace;
    layerCompositionState.colorTransform = getColorTransform();
    layerCompositionState.hdrMetadata = metadata;

    setHwcLayerBuffer(displayDevice);
    // Use rk ashmem -------
    if(mActiveBuffer != nullptr)
        set_handle_layername(mActiveBuffer->handle, mName.string());
    // Use rk ashmem -------
}

X:\frameworks\native\services\surfaceflinger\BufferQueueLayer.cpp
void BufferQueueLayer::setHwcLayerBuffer(const sp<const DisplayDevice>& display) {
	ALOGI("zjj.rk3399.SF setHwcLayerBuffer %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

    const auto outputLayer = findOutputLayerForDisplay(display);
    LOG_FATAL_IF(!outputLayer);
    LOG_FATAL_IF(!outputLayer->getState.hwc);
    auto& hwcLayer = (*outputLayer->getState().hwc).hwcLayer;

    uint32_t hwcSlot = 0;
    sp<GraphicBuffer> hwcBuffer;

    // INVALID_BUFFER_SLOT is used to identify BufferStateLayers.  Default to 0
    // for BufferQueueLayers
    int slot = (mActiveBufferSlot == BufferQueue::INVALID_BUFFER_SLOT) ? 0 : mActiveBufferSlot;
    (*outputLayer->editState().hwc)
            .hwcBufferCache.getHwcBuffer(slot, mActiveBuffer, &hwcSlot, &hwcBuffer);

    auto acquireFence = mConsumer->getCurrentFence();
    auto error = hwcLayer->setBuffer(hwcSlot, hwcBuffer, acquireFence);
    if (error != HWC2::Error::None) {
        ALOGE("[%s] Failed to set buffer %p: %s (%d)", mName.string(), mActiveBuffer->handle,
              to_string(error).c_str(), static_cast<int32_t>(error));
    }

    auto& layerCompositionState = getCompositionLayer()->editState().frontEnd;
    layerCompositionState.bufferSlot = mActiveBufferSlot;
    layerCompositionState.buffer = mActiveBuffer;
    layerCompositionState.acquireFence = acquireFence;
}

```

### 四、beginFrame()函数
开始合成前的准备。

``` cpp
void SurfaceFlinger::beginFrame(const sp<DisplayDevice>& displayDevice) {
    auto display = displayDevice->getCompositionDisplay();
    const auto& displayState = display->getState();

    // 是否有待更新的区域
    bool dirty = !display->getDirtyRegion(false).isEmpty();
    // 可见Layer数量是否为0
    bool empty = displayDevice->getVisibleLayersSortedByZ().size() == 0;
    // 上次合成是否有可见Layer
    bool wasEmpty = !displayState.lastCompositionHadVisibleLayers;

    // 没有变化时或者有变化但此时没有可见Layer且上次合成时也没有就跳过
    bool mustRecompose = dirty && !(empty && wasEmpty);

    const char flagPrefix[] = {'-', '+'};
    static_cast<void>(flagPrefix);
    ALOGV_IF(displayDevice->isVirtual(), "%s: %s composition for %s (%cdirty %cempty %cwasEmpty)",
             __FUNCTION__, mustRecompose ? "doing" : "skipping",
             displayDevice->getDebugName().c_str(), flagPrefix[dirty], flagPrefix[empty],
             flagPrefix[wasEmpty]);

    // 这里面其实没有做什么特殊的操作，我们看一下DisplayDevice相关的类
    display->getRenderSurface()->beginFrame(mustRecompose);

    if (mustRecompose) {
        display->editState().lastCompositionHadVisibleLayers = !empty;
    }
}


status_t RenderSurface::beginFrame(bool mustRecompose) {
	ALOGI("zjj.rk3399.SF beginFrame %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

    return mDisplaySurface->beginFrame(mustRecompose);
}

status_t FramebufferSurface::beginFrame(bool /*mustRecompose*/) {
	ALOGI("zjj.rk3399.SF beginFrame %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

    return NO_ERROR;
}

```

### 五、prepareFrame()函数

``` cpp
void SurfaceFlinger::prepareFrame(const sp<DisplayDevice>& displayDevice) {
    auto display = displayDevice->getCompositionDisplay();
    const auto& displayState = display->getState();
	ALOGI("zjj.rk3399.SF prepareFrame %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

    if (!displayState.isEnabled) {
        return;
    }

    status_t result = display->getRenderSurface()->prepareFrame();
    ALOGE_IF(result != NO_ERROR, "prepareFrame failed for %s: %d (%s)",
             displayDevice->getDebugName().c_str(), result, strerror(-result));
}

status_t RenderSurface::prepareFrame() {
	ALOGI("zjj.rk3399.SF prepareFrame %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

    auto& hwc = mCompositionEngine.getHwComposer();
    const auto id = mDisplay.getId();
    if (id) {
        status_t error = hwc.prepare(*id, mDisplay);
        if (error != NO_ERROR) {
            return error;
        }
    }

    DisplaySurface::CompositionType compositionType;
    const bool hasClient = hwc.hasClientComposition(id);
    const bool hasDevice = hwc.hasDeviceComposition(id);
    if (hasClient && hasDevice) {
        compositionType = DisplaySurface::COMPOSITION_MIXED;
    } else if (hasClient) {
        compositionType = DisplaySurface::COMPOSITION_GLES;
    } else if (hasDevice) {
        compositionType = DisplaySurface::COMPOSITION_HWC;
    } else {
        // Nothing to do -- when turning the screen off we get a frame like
        // this. Call it a HWC frame since we won't be doing any GLES work but
        // will do a prepare/set cycle.
        compositionType = DisplaySurface::COMPOSITION_HWC;
    }
    return mDisplaySurface->prepareFrame(compositionType);
}

status_t FramebufferSurface::prepareFrame(CompositionType /*compositionType*/) {
	ALOGI("zjj.rk3399.SF prepareFrame %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

    return NO_ERROR;
}
```

### 六、doComposition(display, repaintEverything)函数

``` cpp
void SurfaceFlinger::doComposition(const sp<DisplayDevice>& displayDevice, bool repaintEverything) {
    ATRACE_CALL();
	ALOGI("zjj.rk3399.SF doComposition %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

    auto display = displayDevice->getCompositionDisplay();
    const auto& displayState = display->getState();

    if (displayState.isEnabled) {
        // transform the dirty region into this screen's coordinate space
        const Region dirtyRegion = display->getDirtyRegion(repaintEverything);

        // repaint the framebuffer (if needed)
        doDisplayComposition(displayDevice, dirtyRegion);

        display->editState().dirtyRegion.clear();
        display->getRenderSurface()->flip();
    }
}

void SurfaceFlinger::doDisplayComposition(const sp<DisplayDevice>& displayDevice,
                                          const Region& inDirtyRegion) {
    auto display = displayDevice->getCompositionDisplay();
    // We only need to actually compose the display if:
    // 1) It is being handled by hardware composer, which may need this to
    //    keep its virtual display state machine in sync, or
    // 2) There is work to be done (the dirty region isn't empty)
    if (!displayDevice->getId() && inDirtyRegion.isEmpty()) {
        ALOGV("Skipping display composition");
        return;
    }
	ALOGI("zjj.rk3399.SF doDisplayComposition %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

    //ALOGV("doDisplayComposition");
    base::unique_fd readyFence;
    if (!doComposeSurfaces(displayDevice, Region::INVALID_REGION, &readyFence)) return;

    // swap buffers (presentation)
    display->getRenderSurface()->queueBuffer(std::move(readyFence));
}

bool SurfaceFlinger::doComposeSurfaces(const sp<DisplayDevice>& displayDevice,
                                       const Region& debugRegion, base::unique_fd* readyFence) {
    ATRACE_CALL();
	ALOGI("zjj.rk3399.SF doComposeSurfaces %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

    auto display = displayDevice->getCompositionDisplay();
    const auto& displayState = display->getState();
    const auto displayId = display->getId();
    auto& renderEngine = getRenderEngine();
    const bool supportProtectedContent = renderEngine.supportsProtectedContent();

    const Region bounds(displayState.bounds);
    const DisplayRenderArea renderArea(displayDevice);
    const bool hasClientComposition = getHwComposer().hasClientComposition(displayId);
    ATRACE_INT("hasClientComposition", hasClientComposition);

    bool applyColorMatrix = false;

    renderengine::DisplaySettings clientCompositionDisplay;
    std::vector<renderengine::LayerSettings> clientCompositionLayers;
    sp<GraphicBuffer> buf;
    base::unique_fd fd;

    if (hasClientComposition) {
        ALOGV("hasClientComposition");

        if (displayDevice->isPrimary() && supportProtectedContent) {
            bool needsProtected = false;
            for (auto& layer : displayDevice->getVisibleLayersSortedByZ()) {
                // If the layer is a protected layer, mark protected context is needed.
                if (layer->isProtected()) {
                    needsProtected = true;
                    break;
                }
            }
            if (needsProtected != renderEngine.isProtected()) {
                renderEngine.useProtectedContext(needsProtected);
            }
            if (needsProtected != display->getRenderSurface()->isProtected() &&
                needsProtected == renderEngine.isProtected()) {
                display->getRenderSurface()->setProtected(needsProtected);
            }
        }

        buf = display->getRenderSurface()->dequeueBuffer(&fd);

        if (buf == nullptr) {
            ALOGW("Dequeuing buffer for display [%s] failed, bailing out of "
                  "client composition for this frame",
                  displayDevice->getDisplayName().c_str());
            return false;
        }

        clientCompositionDisplay.physicalDisplay = displayState.scissor;
        clientCompositionDisplay.clip = displayState.scissor;
        const ui::Transform& displayTransform = displayState.transform;
        clientCompositionDisplay.globalTransform = displayTransform.asMatrix4();
        clientCompositionDisplay.orientation = displayState.orientation;

        const auto* profile = display->getDisplayColorProfile();
        Dataspace outputDataspace = Dataspace::UNKNOWN;
        if (profile->hasWideColorGamut()) {
            outputDataspace = displayState.dataspace;
        }
        clientCompositionDisplay.outputDataspace = outputDataspace;
        clientCompositionDisplay.maxLuminance =
                profile->getHdrCapabilities().getDesiredMaxLuminance();

        const bool hasDeviceComposition = getHwComposer().hasDeviceComposition(displayId);
        const bool skipClientColorTransform =
                getHwComposer()
                        .hasDisplayCapability(displayId,
                                              HWC2::DisplayCapability::SkipClientColorTransform);

        // Compute the global color transform matrix.
        applyColorMatrix = !hasDeviceComposition && !skipClientColorTransform;
        if (applyColorMatrix) {
            clientCompositionDisplay.colorTransform = displayState.colorTransformMat;
        }
    }

    /*
     * and then, render the layers targeted at the framebuffer
     */

    ALOGV("Rendering client layers");
    bool firstLayer = true;
    Region clearRegion = Region::INVALID_REGION;
    for (auto& layer : displayDevice->getVisibleLayersSortedByZ()) {
        const Region viewportRegion(displayState.viewport);
        const Region clip(viewportRegion.intersect(layer->visibleRegion));
        ALOGV("Layer: %s", layer->getName().string());
        ALOGV("  Composition type: %s", toString(layer->getCompositionType(displayDevice)).c_str());
        if (!clip.isEmpty()) {
            switch (layer->getCompositionType(displayDevice)) {
                case Hwc2::IComposerClient::Composition::CURSOR:
                case Hwc2::IComposerClient::Composition::DEVICE:
                case Hwc2::IComposerClient::Composition::SIDEBAND:
                case Hwc2::IComposerClient::Composition::SOLID_COLOR: {
                    LOG_ALWAYS_FATAL_IF(!displayId);
                    const Layer::State& state(layer->getDrawingState());
                    if (layer->getClearClientTarget(displayDevice) && !firstLayer &&
                        layer->isOpaque(state) && (layer->getAlpha() == 1.0f) &&
                        layer->getRoundedCornerState().radius == 0.0f && hasClientComposition) {
                        // never clear the very first layer since we're
                        // guaranteed the FB is already cleared
                        renderengine::LayerSettings layerSettings;
                        Region dummyRegion;
                        bool prepared =
                                layer->prepareClientLayer(renderArea, clip, dummyRegion,
                                                          supportProtectedContent, layerSettings);

                        if (prepared) {
                            layerSettings.source.buffer.buffer = nullptr;
                            layerSettings.source.solidColor = half3(0.0, 0.0, 0.0);
                            layerSettings.alpha = half(0.0);
                            layerSettings.disableBlending = true;
                            clientCompositionLayers.push_back(layerSettings);
                        }
                    }
                    break;
                }
                case Hwc2::IComposerClient::Composition::CLIENT: {
                    renderengine::LayerSettings layerSettings;
                    bool prepared =
                            layer->prepareClientLayer(renderArea, clip, clearRegion,
                                                      supportProtectedContent, layerSettings);
                    if (prepared) {
                        clientCompositionLayers.push_back(layerSettings);
                    }
                    break;
                }
                default:
                    break;
            }
        } else {
            ALOGV("  Skipping for empty clip");
        }
        firstLayer = false;
    }

    // Perform some cleanup steps if we used client composition.
    if (hasClientComposition) {
        clientCompositionDisplay.clearRegion = clearRegion;

        // We boost GPU frequency here because there will be color spaces conversion
        // and it's expensive. We boost the GPU frequency so that GPU composition can
        // finish in time. We must reset GPU frequency afterwards, because high frequency
        // consumes extra battery.
        const bool expensiveRenderingExpected =
                clientCompositionDisplay.outputDataspace == Dataspace::DISPLAY_P3;
        if (expensiveRenderingExpected && displayId) {
            mPowerAdvisor.setExpensiveRenderingExpected(*displayId, true);
        }
        if (!debugRegion.isEmpty()) {
            Region::const_iterator it = debugRegion.begin();
            Region::const_iterator end = debugRegion.end();
            while (it != end) {
                const Rect& rect = *it++;
                renderengine::LayerSettings layerSettings;
                layerSettings.source.buffer.buffer = nullptr;
                layerSettings.source.solidColor = half3(1.0, 0.0, 1.0);
                layerSettings.geometry.boundaries = rect.toFloatRect();
                layerSettings.alpha = half(1.0);
                clientCompositionLayers.push_back(layerSettings);
            }
        }
		ALOGD_CALLSTACK("drawLayers");

        renderEngine.drawLayers(clientCompositionDisplay, clientCompositionLayers,
                                buf->getNativeBuffer(), /*useFramebufferCache=*/true, std::move(fd),
                                readyFence);
    } else if (displayId) {
        mPowerAdvisor.setExpensiveRenderingExpected(*displayId, false);
    }
    return true;
}

```
#### 6.1、RenderSurface()->dequeueBuffer(&fd)
看看Log：

``` cpp
10-28 09:40:54.275   179   179 V SurfaceFlinger: hasClientComposition
10-28 09:40:54.275   179   179 I CompositionEngine: zjj.rk3399.SF dequeueBuffer frameworks/native/services/surfaceflinger/CompositionEngine/src/RenderSurface.cpp dequeueBuffer 166 
10-28 09:40:54.275   179   179 D CompositionEngine: queueBuffer
10-28 09:40:54.297   179   179 D CompositionEngine:   #00 pc 000000000011ed98  /system/lib64/libsurfaceflinger.so (android::compositionengine::impl::RenderSurface::dequeueBuffer(android::base::unique_fd_impl<android::base::DefaultCloser>*)+184)
10-28 09:40:54.297   179   179 D CompositionEngine:   #01 pc 00000000000e3998  /system/lib64/libsurfaceflinger.so (android::SurfaceFlinger::doComposeSurfaces(android::sp<android::DisplayDevice> const&, android::Region const&, android::base::unique_fd_impl<android::base::DefaultCloser>*)+1024)
10-28 09:40:54.297   179   179 D CompositionEngine:   #02 pc 00000000000e0868  /system/lib64/libsurfaceflinger.so (android::SurfaceFlinger::handleMessageRefresh()+3528)
10-28 09:40:54.297   179   179 D CompositionEngine:   #03 pc 00000000000df6bc  /system/lib64/libsurfaceflinger.so (android::SurfaceFlinger::onMessageReceived(int)+9484)
10-28 09:40:54.297   179   179 D CompositionEngine:   #04 pc 0000000000017dfc  /system/lib64/libutils.so (android::Looper::pollInner(int)+332)
10-28 09:40:54.297   179   179 D CompositionEngine:   #05 pc 0000000000017c10  /system/lib64/libutils.so (android::Looper::pollOnce(int, int*, int*, void**)+56)
10-28 09:40:54.297   179   179 D CompositionEngine:   #06 pc 00000000000ced04  /system/lib64/libsurfaceflinger.so (android::impl::MessageQueue::waitMessage()+92)
10-28 09:40:54.297   179   179 D CompositionEngine:   #07 pc 00000000000dc69c  /system/lib64/libsurfaceflinger.so (android::SurfaceFlinger::run()+20)
10-28 09:40:54.297   179   179 D CompositionEngine:   #08 pc 0000000000003370  /system/bin/surfaceflinger (main+800)
10-28 09:40:54.297   179   179 D CompositionEngine:   #09 pc 000000000007d844  /apex/com.android.runtime/lib64/bionic/libc.so (__libc_init+108)
10-28 09:40:54.297   179   179 V Surface : Surface::dequeueBuffer
10-28 09:40:54.297   179   179 V BufferQueueProducer: [FramebufferSurface] dequeueBuffer: w=0 h=0 format=0x1, usage=0x200
10-28 09:40:54.297   179   179 V BufferQueueProducer: [FramebufferSurface] dequeueBuffer: setting buffer age to 1
10-28 09:40:54.297   179   179 V BufferQueueProducer: [FramebufferSurface] dequeueBuffer: returning slot=2/0 buf=0x70de65bb40 flags=0x1
10-28 09:40:54.297   179   179 V BufferQueueProducer: [FramebufferSurface] requestBuffer: slot 2
10-28 09:40:54.297   179   179 V SurfaceFlinger: Rendering client layers
```

dequeueBuffer用于RenderSurface。
``` cpp
sp<GraphicBuffer> RenderSurface::dequeueBuffer(base::unique_fd* bufferFence) {
    ATRACE_CALL();
    int fd = -1;
    ANativeWindowBuffer* buffer = nullptr;
	ALOGI("zjj.rk3399.SF dequeueBuffer %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
	ALOGD_CALLSTACK("queueBuffer");

    status_t result = mNativeWindow->dequeueBuffer(mNativeWindow.get(), &buffer, &fd);

    if (result != NO_ERROR) {
        ALOGE("ANativeWindow::dequeueBuffer failed for display [%s] with error: %d",
              mDisplay.getName().c_str(), result);
        // Return fast here as we can't do much more - any rendering we do
        // now will just be wrong.
        return mGraphicBuffer;
    }

    ALOGW_IF(mGraphicBuffer != nullptr, "Clobbering a non-null pointer to a buffer [%p].",
             mGraphicBuffer->getNativeBuffer()->handle);
    mGraphicBuffer = GraphicBuffer::from(buffer);

    *bufferFence = base::unique_fd(fd);

    return mGraphicBuffer;
}
```

#### 6.2、prepareClientLayer()

``` cpp
bool BufferLayer::prepareClientLayer(const RenderArea& renderArea, const Region& clip,
                                     bool useIdentityTransform, Region& clearRegion,
                                     const bool supportProtectedContent,
                                     renderengine::LayerSettings& layer) {
    ATRACE_CALL();
	ALOGI("zjj.rk3399.SF prepareClientLayer %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

    Layer::prepareClientLayer(renderArea, clip, useIdentityTransform, clearRegion,
                              supportProtectedContent, layer);
    if (CC_UNLIKELY(mActiveBuffer == 0)) {
        // the texture has not been created yet, this Layer has
        // in fact never been drawn into. This happens frequently with
        // SurfaceView because the WindowManager can't know when the client
        // has drawn the first time.

        // If there is nothing under us, we paint the screen in black, otherwise
        // we just skip this update.

        // figure out if there is something below us
        Region under;
        bool finished = false;
        mFlinger->mDrawingState.traverseInZOrder([&](Layer* layer) {
            if (finished || layer == static_cast<BufferLayer const*>(this)) {
                finished = true;
                return;
            }
            under.orSelf(layer->visibleRegion);
        });
        // if not everything below us is covered, we plug the holes!
        Region holes(clip.subtract(under));
        if (!holes.isEmpty()) {
            clearRegion.orSelf(holes);
        }
        return false;
    }

    //rk_ext: Deliver to GLESRenderEngine for 10bit to 8bit
    layer.source.buffer.currentcrop = mCurrentCrop;

    bool blackOutLayer =
            (isProtected() && !supportProtectedContent) || (isSecure() && !renderArea.isSecure());
    const State& s(getDrawingState());
    if (!blackOutLayer) {
        layer.source.buffer.buffer = mActiveBuffer;
        layer.source.buffer.isOpaque = isOpaque(s);
        layer.source.buffer.fence = mActiveBufferFence;
        layer.source.buffer.textureName = mTextureName;
        layer.source.buffer.usePremultipliedAlpha = getPremultipledAlpha();
        layer.source.buffer.isY410BT2020 = isHdrY410();
        // TODO: we could be more subtle with isFixedSize()
        const bool useFiltering = needsFiltering(renderArea.getDisplayDevice()) ||
                renderArea.needsFiltering() || isFixedSize();

        // Query the texture matrix given our current filtering mode.
        float textureMatrix[16];
        setFilteringEnabled(useFiltering);
        getDrawingTransformMatrix(textureMatrix);

        if (getTransformToDisplayInverse()) {
            /*
             * the code below applies the primary display's inverse transform to
             * the texture transform
             */
            uint32_t transform = DisplayDevice::getPrimaryDisplayOrientationTransform();
            mat4 tr = inverseOrientation(transform);

            /**
             * TODO(b/36727915): This is basically a hack.
             *
             * Ensure that regardless of the parent transformation,
             * this buffer is always transformed from native display
             * orientation to display orientation. For example, in the case
             * of a camera where the buffer remains in native orientation,
             * we want the pixels to always be upright.
             */
            sp<Layer> p = mDrawingParent.promote();
            if (p != nullptr) {
                const auto parentTransform = p->getTransform();
                tr = tr * inverseOrientation(parentTransform.getOrientation());
            }

            // and finally apply it to the original texture matrix
            const mat4 texTransform(mat4(static_cast<const float*>(textureMatrix)) * tr);
            memcpy(textureMatrix, texTransform.asArray(), sizeof(textureMatrix));
        }

        const Rect win{getBounds()};
        float bufferWidth = getBufferSize(s).getWidth();
        float bufferHeight = getBufferSize(s).getHeight();

        // BufferStateLayers can have a "buffer size" of [0, 0, -1, -1] when no display frame has
        // been set and there is no parent layer bounds. In that case, the scale is meaningless so
        // ignore them.
        if (!getBufferSize(s).isValid()) {
            bufferWidth = float(win.right) - float(win.left);
            bufferHeight = float(win.bottom) - float(win.top);
        }

        const float scaleHeight = (float(win.bottom) - float(win.top)) / bufferHeight;
        const float scaleWidth = (float(win.right) - float(win.left)) / bufferWidth;
        const float translateY = float(win.top) / bufferHeight;
        const float translateX = float(win.left) / bufferWidth;

        // Flip y-coordinates because GLConsumer expects OpenGL convention.
        mat4 tr = mat4::translate(vec4(.5, .5, 0, 1)) * mat4::scale(vec4(1, -1, 1, 1)) *
                mat4::translate(vec4(-.5, -.5, 0, 1)) *
                mat4::translate(vec4(translateX, translateY, 0, 1)) *
                mat4::scale(vec4(scaleWidth, scaleHeight, 1.0, 1.0));

        layer.source.buffer.useTextureFiltering = useFiltering;
        layer.source.buffer.textureTransform = mat4(static_cast<const float*>(textureMatrix)) * tr;
    } else {
        // If layer is blacked out, force alpha to 1 so that we draw a black color
        // layer.
        layer.source.buffer.buffer = nullptr;
        layer.alpha = 1.0;
    }

    return true;
}


bool Layer::prepareClientLayer(const RenderArea& /*renderArea*/, const Region& /*clip*/,
                               bool useIdentityTransform, Region& /*clearRegion*/,
                               const bool /*supportProtectedContent*/,
                               renderengine::LayerSettings& layer) {
	ALOGI("zjj.rk3399.SF prepareClientLayer %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

    FloatRect bounds = getBounds();
    half alpha = getAlpha();
    layer.geometry.boundaries = bounds;
    if (useIdentityTransform) {
        layer.geometry.positionTransform = mat4();
    } else {
        const ui::Transform transform = getTransform();
        mat4 m;
        m[0][0] = transform[0][0];
        m[0][1] = transform[0][1];
        m[0][3] = transform[0][2];
        m[1][0] = transform[1][0];
        m[1][1] = transform[1][1];
        m[1][3] = transform[1][2];
        m[3][0] = transform[2][0];
        m[3][1] = transform[2][1];
        m[3][3] = transform[2][2];
        layer.geometry.positionTransform = m;
    }

    if (hasColorTransform()) {
        layer.colorTransform = getColorTransform();
    }

    const auto roundedCornerState = getRoundedCornerState();
    layer.geometry.roundedCornersRadius = roundedCornerState.radius;
    layer.geometry.roundedCornersCrop = roundedCornerState.cropRect;

    layer.alpha = alpha;
    layer.sourceDataspace = mCurrentDataSpace;
    return true;
}
```


#### 6.3、drawLayers()
使用渲染引擎GLESRenderEngine合成所有Layer
``` cpp
status_t GLESRenderEngine::drawLayers(const DisplaySettings& display,
                                      const std::vector<LayerSettings>& layers,
                                      ANativeWindowBuffer* const buffer,
                                      const bool useFramebufferCache, base::unique_fd&& bufferFence,
                                      base::unique_fd* drawFence) {
    ATRACE_CALL();
    if (layers.empty()) {
        ALOGV("Drawing empty layer stack");
        return NO_ERROR;
    }
		ALOGI("zjj.rk3399.SF drawLayers %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
		//ALOGD_CALLSTACK("drawLayers");

#if MALI_PRODUCT_ID_450 || MALI_PRODUCT_ID_400
        if(bufferFence.get() >= 0){     //.RK ext: don't use waitFence before solve the problem(eglWaitSyncKHR didn't wait release fence) 
#else
        if (bufferFence.get() >= 0 && !waitFence(std::move(bufferFence))) {
#endif
        ATRACE_NAME("Waiting before draw");
        sync_wait(bufferFence.get(), -1);
    }

    if (buffer == nullptr) {
        ALOGE("No output buffer provided. Aborting GPU composition.");
        return BAD_VALUE;
    }

    BindNativeBufferAsFramebuffer fbo(*this, buffer, useFramebufferCache);

    if (fbo.getStatus() != NO_ERROR) {
        ALOGE("Failed to bind framebuffer! Aborting GPU composition for buffer (%p).",
              buffer->handle);
        checkErrors();
        return fbo.getStatus();
    }

    // clear the entire buffer, sometimes when we reuse buffers we'd persist
    // ghost images otherwise.
    // we also require a full transparent framebuffer for overlays. This is
    // probably not quite efficient on all GPUs, since we could filter out
    // opaque layers.
    clearWithColor(0.0, 0.0, 0.0, 0.0);

    setViewportAndProjection(display.physicalDisplay, display.clip);

    setOutputDataSpace(display.outputDataspace);
    setDisplayMaxLuminance(display.maxLuminance);

    mat4 projectionMatrix = mState.projectionMatrix * display.globalTransform;
    mState.projectionMatrix = projectionMatrix;
    if (!display.clearRegion.isEmpty()) {
        glDisable(GL_BLEND);
        fillRegionWithColor(display.clearRegion, 0.0, 0.0, 0.0, 1.0);
    }

    Mesh mesh(Mesh::TRIANGLE_FAN, 4, 2, 2);
    for (auto layer : layers) {
        mState.projectionMatrix = projectionMatrix * layer.geometry.positionTransform;

        const FloatRect bounds = layer.geometry.boundaries;
        Mesh::VertexArray<vec2> position(mesh.getPositionArray<vec2>());
        position[0] = vec2(bounds.left, bounds.top);
        position[1] = vec2(bounds.left, bounds.bottom);
        position[2] = vec2(bounds.right, bounds.bottom);
        position[3] = vec2(bounds.right, bounds.top);

        setupLayerCropping(layer, mesh);
        setColorTransform(display.colorTransform * layer.colorTransform);

        bool usePremultipliedAlpha = true;
        bool disableTexture = true;
        bool isOpaque = false;

        if (layer.source.buffer.buffer != nullptr) {
            disableTexture = false;
            isOpaque = layer.source.buffer.isOpaque;

            sp<GraphicBuffer> gBuf = layer.source.buffer.buffer;
#if (RK_NV12_10_TO_NV12_BY_RGA | RK_NV12_10_TO_NV12_BY_NENO | RK_HDR)
            if(gBuf != NULL &&
               gBuf->getPixelFormat() == HAL_PIXEL_FORMAT_YCrCb_NV12_10 )
            {
                //Rect CurrentCrop(0,0,3840,2160);
                Rect CurrentCrop(layer.source.buffer.currentcrop);
#if RK_HDR
                const int yuvTexUsage = GraphicBuffer::USAGE_HW_TEXTURE | GRALLOC_USAGE_TO_USE_ARM_P010;
                const int yuvTexFormat = HAL_PIXEL_FORMAT_YCrCb_NV12_10;
#elif (RK_NV12_10_TO_NV12_BY_NENO | RK_NV12_10_TO_NV12_BY_RGA)
                const int yuvTexUsage = GraphicBuffer::USAGE_HW_TEXTURE /*| HDRUSAGE*/;
                //GraphicBuffer::USAGE_SW_WRITE_RARELY;
                const int yuvTexFormat = HAL_PIXEL_FORMAT_YCrCb_NV12;
#endif
                static int yuvcnt;
                int yuvIndex ;

                yuvcnt ++;
                yuvIndex = yuvcnt%2;
#if (RK_HDR | RK_NV12_10_TO_NV12_BY_NENO)
                int src_l,src_t,src_r,src_b,src_stride;
                void *src_vaddr;
                void *dst_vaddr;
                src_l = CurrentCrop.left;
                src_t = CurrentCrop.top;
                src_r = CurrentCrop.right;
                src_b = CurrentCrop.bottom;
                src_stride = gBuf->getStride();
                uint32_t w = src_r - src_l;
#elif RK_NV12_10_TO_NV12_BY_RGA
                //Since rga cann't support scalet to bigger than 4096 limit to 4096
                uint32_t w = (CurrentCrop.getWidth() + 31) & (~31);
                //ALOGD("10to8[%s %d] f:%x w:%d gH:%d\n",__FUNCTION__,__LINE__,gBuf->getPixelFormat(),w,gBuf->getHeight());
#endif
                if((yuvTeximg[yuvIndex].yuvTexBuffer != NULL) &&
                   (yuvTeximg[yuvIndex].yuvTexBuffer->getWidth() != w ||
                    yuvTeximg[yuvIndex].yuvTexBuffer->getHeight() != gBuf->getHeight()))
                {
                    yuvTeximg[yuvIndex].yuvTexBuffer = NULL;
                }
                if(yuvTeximg[yuvIndex].yuvTexBuffer == NULL)
                {
                    yuvTeximg[yuvIndex].yuvTexBuffer = new GraphicBuffer(w, gBuf->getHeight(),yuvTexFormat, yuvTexUsage);
                }

#if (RK_HDR | RK_NV12_10_TO_NV12_BY_NENO)
                gBuf->lock(GRALLOC_USAGE_SW_READ_OFTEN,&src_vaddr);
                yuvTeximg[yuvIndex].yuvTexBuffer->lock(GRALLOC_USAGE_SW_WRITE_OFTEN,&dst_vaddr);

                //PRINT_TIME_START
                if(dso == NULL)
                    dso = dlopen(RK_XXX_PATH, RTLD_NOW | RTLD_LOCAL);

                if (dso == 0) {
                    ALOGE("rk_debug can't not find /system/lib64/librockchipxxx.so ! error=%s \n",
                        dlerror());
                    return BAD_VALUE;
                }
#if RK_HDR
                if(rockchipxxx == NULL)
                    rockchipxxx = (__rockchipxxx)dlsym(dso, "_Z11rockchipxxxPhS_iiiii");
                if(rockchipxxx == NULL)
                {
                    ALOGE("rk_debug can't not find target function in /system/lib64/librockchipxxx.so ! \n");
                    dlclose(dso);
                    return BAD_VALUE;
                }
                /* align w to 64 */
                w = ALIGN(w, 64);
                ALOGD("DEBUG_lb Stride=%d",yuvTeximg[yuvIndex].yuvTexBuffer->getStride());
                if(w <= yuvTeximg[yuvIndex].yuvTexBuffer->getStride()/2)
                {
                    rockchipxxx((u8*)src_vaddr, (u8*)dst_vaddr, w, src_b - src_t, src_stride, yuvTeximg[yuvIndex].yuvTexBuffer->getStride(), 0);
                }else
                    ALOGE("%s(%d):unsupport resolution for 4k", __FUNCTION__, __LINE__);
#elif RK_NV12_10_TO_NV12_BY_NENO
                if(rockchipxxx3288 == NULL)
                    rockchipxxx3288 = (__rockchipxxx3288)dlsym(dso, "_Z15rockchipxxx3288PhS_iiiii");
                if(rockchipxxx3288 == NULL)
                {
                    ALOGE("rk_debug can't not find target function in /system/lib64/librockchipxxx.so ! \n");
                    dlclose(dso);
                    return BAD_VALUE;
                }
                rockchipxxx3288((u8*)src_vaddr, (u8*)dst_vaddr, src_r - src_l, src_b - src_t, src_stride, (src_r - src_l), 0);
#endif
                //PRINT_TIME_END("convert10to16_highbit_arm64_neon")
                ALOGD("src_vaddr=%p,dst_vaddr=%p,crop_w=%d,crop_h=%d,stride=%f, src_stride=%d,raw_w=%d,raw_h=%d",
                        src_vaddr, dst_vaddr, src_r - src_l,src_b - src_t,
                        (src_r - src_l)*1.25+64,src_stride,gBuf->getWidth(),gBuf->getHeight());
                //dump data
                static int i =0;
                char pro_value[PROPERTY_VALUE_MAX];

                property_get("sys.dump_out_neon",pro_value,0);
                if(i<10 && !strcmp(pro_value,"true"))
                {
                    char data_name[100];

                    sprintf(data_name,"/data/dump/glesdmlayer%d_%d_%d.bin", i,
                            yuvTeximg[yuvIndex].yuvTexBuffer->getWidth(),yuvTeximg[yuvIndex].yuvTexBuffer->getHeight());
#if RK_HDR
                    int n = yuvTeximg[yuvIndex].yuvTexBuffer->getHeight() * yuvTeximg[yuvIndex].yuvTexBuffer->getStride();
#else
                    int n = yuvTeximg[yuvIndex].yuvTexBuffer->getHeight() * yuvTeximg[yuvIndex].yuvTexBuffer->getStride() * 1.5;
#endif
                    ALOGD("dump %s size=%d", data_name, n );
                    FILE *fp;
                    if ((fp = fopen(data_name, "w+")) == NULL)
                    {
                        printf("can't open output.bin!!!!!\n");
                    }
                    fwrite(dst_vaddr, n, 1, fp);
                    fclose(fp);
                    i++;
                }

#elif RK_NV12_10_TO_NV12_BY_RGA
                rgaCopyBit(gBuf, yuvTeximg[yuvIndex].yuvTexBuffer, CurrentCrop);
#endif
                bindExternalTextureBuffer(layer.source.buffer.textureName,
                                yuvTeximg[yuvIndex].yuvTexBuffer, layer.source.buffer.fence);

            }
            else
#endif
            {
                bindExternalTextureBuffer(layer.source.buffer.textureName, gBuf, layer.source.buffer.fence);
            }

            usePremultipliedAlpha = layer.source.buffer.usePremultipliedAlpha;
            Texture texture(Texture::TEXTURE_EXTERNAL, layer.source.buffer.textureName);
            mat4 texMatrix = layer.source.buffer.textureTransform;

            texture.setMatrix(texMatrix.asArray());
            texture.setFiltering(layer.source.buffer.useTextureFiltering);

            texture.setDimensions(gBuf->getWidth(), gBuf->getHeight());
            setSourceY410BT2020(layer.source.buffer.isY410BT2020);

            renderengine::Mesh::VertexArray<vec2> texCoords(mesh.getTexCoordArray<vec2>());
            texCoords[0] = vec2(0.0, 0.0);
            texCoords[1] = vec2(0.0, 1.0);
            texCoords[2] = vec2(1.0, 1.0);
            texCoords[3] = vec2(1.0, 0.0);
            setupLayerTexturing(texture);
        }

        const half3 solidColor = layer.source.solidColor;
        const half4 color = half4(solidColor.r, solidColor.g, solidColor.b, layer.alpha);
        // Buffer sources will have a black solid color ignored in the shader,
        // so in that scenario the solid color passed here is arbitrary.
        setupLayerBlending(usePremultipliedAlpha, isOpaque, disableTexture, color,
                           layer.geometry.roundedCornersRadius);
        if (layer.disableBlending) {
            glDisable(GL_BLEND);
        }
        setSourceDataSpace(layer.sourceDataspace);

        // We only want to do a special handling for rounded corners when having rounded corners
        // is the only reason it needs to turn on blending, otherwise, we handle it like the
        // usual way since it needs to turn on blending anyway.
        if (layer.geometry.roundedCornersRadius > 0.0 && color.a >= 1.0f && isOpaque) {
            handleRoundedCorners(display, layer, mesh);
        } else {
            drawMesh(mesh);
        }

        // Cleanup if there's a buffer source
        if (layer.source.buffer.buffer != nullptr) {
            disableBlending();
            setSourceY410BT2020(false);
            disableTexturing();
        }
    }

    if (drawFence != nullptr) {
		ALOGI("zjj.rk3399.SF drawFence %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

        *drawFence = flush();
    }
    // If flush failed or we don't support native fences, we need to force the
    // gl command stream to be executed.
    if (drawFence == nullptr || drawFence->get() < 0) {
		ALOGI("zjj.rk3399.SF drawFence %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

        bool success = finish();
        if (!success) {
            ALOGE("Failed to flush RenderEngine commands");
            checkErrors();
            // Chances are, something illegal happened (either the caller passed
            // us bad parameters, or we messed up our shader generation).
            return INVALID_OPERATION;
        }
    }

    checkErrors();
    return NO_ERROR;
}

```
看看Log：

``` cpp
10-28 09:40:54.319   179   179 I RenderEngine: zjj.rk3399.SF drawLayers frameworks/native/libs/renderengine/gl/GLESRenderEngine.cpp drawLayers 1089 
10-28 09:40:54.319   179   179 I RenderEngine: zjj.rk3399.SF getFramebufferForDrawing frameworks/native/libs/renderengine/gl/GLESRenderEngine.cpp getFramebufferForDrawing 484 
10-28 09:40:54.319   179   179 I RenderEngine: zjj.rk3399.SF setNativeWindowBuffer frameworks/native/libs/renderengine/gl/GLFramebuffer.cpp setNativeWindowBuffer 48 
10-28 09:40:54.319   179   179 I RenderEngine: zjj.rk3399.SF createFramebufferImageIfNeeded frameworks/native/libs/renderengine/gl/GLESRenderEngine.cpp createFramebufferImageIfNeeded 944 
10-28 09:40:54.320   179   179 I RenderEngine: zjj.rk3399.SF bindFrameBuffer frameworks/native/libs/renderengine/gl/GLESRenderEngine.cpp bindFrameBuffer 883 
10-28 09:40:54.320   179   179 I RenderEngine: zjj.rk3399.SF bindExternalTextureBuffer frameworks/native/libs/renderengine/gl/GLESRenderEngine.cpp bindExternalTextureBuffer 667 
10-28 09:40:54.320   179   179 I RenderEngine: zjj.rk3399.SF bindExternalTextureImage frameworks/native/libs/renderengine/gl/GLESRenderEngine.cpp bindExternalTextureImage 648 
10-28 09:40:54.321   179   179 I RenderEngine: zjj.rk3399.SF flush frameworks/native/libs/renderengine/gl/GLESRenderEngine.cpp flush 503
```
#### 6.4、RenderSurface()->queueBuffer()

``` cpp
// swap buffers (presentation)
display->getRenderSurface()->queueBuffer(std::move(readyFence));

X:\frameworks\native\services\surfaceflinger\CompositionEngine\src\RenderSurface.cpp
void RenderSurface::queueBuffer(base::unique_fd&& readyFence) {
    auto& hwc = mCompositionEngine.getHwComposer();
    const auto id = mDisplay.getId();
	ALOGI("zjj.rk3399.SF queueBuffer %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
	ALOGD_CALLSTACK("queueBuffer");

    if (hwc.hasClientComposition(id) || hwc.hasFlipClientTargetRequest(id)) {
        // hasFlipClientTargetRequest could return true even if we haven't
        // dequeued a buffer before. Try dequeueing one if we don't have a
        // buffer ready.
        if (mGraphicBuffer == nullptr) {
            ALOGI("Attempting to queue a client composited buffer without one "
                  "previously dequeued for display [%s]. Attempting to dequeue "
                  "a scratch buffer now",
                  mDisplay.getName().c_str());
            // We shouldn't deadlock here, since mGraphicBuffer == nullptr only
            // after a successful call to queueBuffer, or if dequeueBuffer has
            // never been called.
            base::unique_fd unused;
            dequeueBuffer(&unused);
        }

        if (mGraphicBuffer == nullptr) {
            ALOGE("No buffer is ready for display [%s]", mDisplay.getName().c_str());
        } else {
            status_t result =
                    mNativeWindow->queueBuffer(mNativeWindow.get(),
                                               mGraphicBuffer->getNativeBuffer(), dup(readyFence));
            if (result != NO_ERROR) {
                ALOGE("Error when queueing buffer for display [%s]: %d", mDisplay.getName().c_str(),
                      result);
                // We risk blocking on dequeueBuffer if the primary display failed
                // to queue up its buffer, so crash here.
                if (!mDisplay.isVirtual()) {
                    LOG_ALWAYS_FATAL("ANativeWindow::queueBuffer failed with error: %d", result);
                } else {
                    mNativeWindow->cancelBuffer(mNativeWindow.get(),
                                                mGraphicBuffer->getNativeBuffer(), dup(readyFence));
                }
            }

            mGraphicBuffer = nullptr;
        }
    }

    status_t result = mDisplaySurface->advanceFrame();
    if (result != NO_ERROR) {
        ALOGE("[%s] failed pushing new frame to HWC: %d", mDisplay.getName().c_str(), result);
    }
}


X:\frameworks\native\services\surfaceflinger\DisplayHardware\FramebufferSurface.cpp

status_t FramebufferSurface::advanceFrame() {
    uint32_t slot = 0;
	ALOGI("zjj.rk3399.SF advanceFrame %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

	sp<GraphicBuffer> buf;
    sp<Fence> acquireFence(Fence::NO_FENCE);
    Dataspace dataspace = Dataspace::UNKNOWN;
    status_t result = nextBuffer(slot, buf, acquireFence, dataspace);
    mDataSpace = dataspace;
    if (result != NO_ERROR) {
        ALOGE("error latching next FramebufferSurface buffer: %s (%d)",
                strerror(-result), result);
    }
    return result;
}

status_t FramebufferSurface::nextBuffer(uint32_t& outSlot,
        sp<GraphicBuffer>& outBuffer, sp<Fence>& outFence,
        Dataspace& outDataspace) {
    Mutex::Autolock lock(mMutex);
	ALOGI("zjj.rk3399.SF nextBuffer %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
	ALOGD_CALLSTACK("nextBuffer");

    BufferItem item;
    status_t err = acquireBufferLocked(&item, 0);
    if (err == BufferQueue::NO_BUFFER_AVAILABLE) {
        mHwcBufferCache.getHwcBuffer(mCurrentBufferSlot, mCurrentBuffer, &outSlot, &outBuffer);
        return NO_ERROR;
    } else if (err != NO_ERROR) {
        ALOGE("error acquiring buffer: %s (%d)", strerror(-err), err);
        return err;
    }

    // If the BufferQueue has freed and reallocated a buffer in mCurrentSlot
    // then we may have acquired the slot we already own.  If we had released
    // our current buffer before we call acquireBuffer then that release call
    // would have returned STALE_BUFFER_SLOT, and we would have called
    // freeBufferLocked on that slot.  Because the buffer slot has already
    // been overwritten with the new buffer all we have to do is skip the
    // releaseBuffer call and we should be in the same state we'd be in if we
    // had released the old buffer first.
    if (mCurrentBufferSlot != BufferQueue::INVALID_BUFFER_SLOT &&
        item.mSlot != mCurrentBufferSlot) {
        mHasPendingRelease = true;
        mPreviousBufferSlot = mCurrentBufferSlot;
        mPreviousBuffer = mCurrentBuffer;
    }
    mCurrentBufferSlot = item.mSlot;
    mCurrentBuffer = mSlots[mCurrentBufferSlot].mGraphicBuffer;
    mCurrentFence = item.mFence;

    outFence = item.mFence;
    mHwcBufferCache.getHwcBuffer(mCurrentBufferSlot, mCurrentBuffer, &outSlot, &outBuffer);
    outDataspace = static_cast<Dataspace>(item.mDataSpace);
    status_t result = mHwc.setClientTarget(mDisplayId, outSlot, outFence, outBuffer, outDataspace);
    if (result != NO_ERROR) {
        ALOGE("error posting framebuffer: %d", result);
        return result;
    }

    return NO_ERROR;
}
```

### 七、postFramebuffer(display)函数

``` cpp
void SurfaceFlinger::postFramebuffer(const sp<DisplayDevice>& displayDevice) {
    ATRACE_CALL();
	ALOGI("zjj.rk3399.SF postFramebuffer %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

    auto display = displayDevice->getCompositionDisplay();
    const auto& displayState = display->getState();
    const auto displayId = display->getId();

    if (displayState.isEnabled) {
        if (displayId) {
            getHwComposer().presentAndGetReleaseFences(*displayId);
        }
        display->getRenderSurface()->onPresentDisplayCompleted();
        for (auto& layer : display->getOutputLayersOrderedByZ()) {
            sp<Fence> releaseFence = Fence::NO_FENCE;
            bool usedClientComposition = true;

            // The layer buffer from the previous frame (if any) is released
            // by HWC only when the release fence from this frame (if any) is
            // signaled.  Always get the release fence from HWC first.
            if (layer->getState().hwc) {
                const auto& hwcState = *layer->getState().hwc;
                releaseFence =
                        getHwComposer().getLayerReleaseFence(*displayId, hwcState.hwcLayer.get());
                usedClientComposition =
                        hwcState.hwcCompositionType == Hwc2::IComposerClient::Composition::CLIENT;
            }

            // If the layer was client composited in the previous frame, we
            // need to merge with the previous client target acquire fence.
            // Since we do not track that, always merge with the current
            // client target acquire fence when it is available, even though
            // this is suboptimal.
            if (usedClientComposition) {
                releaseFence =
                        Fence::merge("LayerRelease", releaseFence,
                                     display->getRenderSurface()->getClientTargetAcquireFence());
            }

            layer->getLayerFE().onLayerDisplayed(releaseFence);
        }

        // We've got a list of layers needing fences, that are disjoint with
        // display->getVisibleLayersSortedByZ.  The best we can do is to
        // supply them with the present fence.
        if (!displayDevice->getLayersNeedingFences().isEmpty()) {
            sp<Fence> presentFence =
                    displayId ? getHwComposer().getPresentFence(*displayId) : Fence::NO_FENCE;
            for (auto& layer : displayDevice->getLayersNeedingFences()) {
                layer->getCompositionLayer()->getLayerFE()->onLayerDisplayed(presentFence);
            }
        }

        if (displayId) {
            getHwComposer().clearReleaseFences(*displayId);
        }
    }
}

X:\frameworks\native\services\surfaceflinger\DisplayHardware\HWComposer.cpp
status_t HWComposer::presentAndGetReleaseFences(DisplayId displayId) {
    ATRACE_CALL();

    RETURN_IF_INVALID_DISPLAY(displayId, BAD_INDEX);

    auto& displayData = mDisplayData[displayId];
    auto& hwcDisplay = displayData.hwcDisplay;

    if (displayData.validateWasSkipped) {
        // explicitly flush all pending commands
        auto error = mHwcDevice->flushCommands();
        RETURN_IF_HWC_ERROR_FOR("flushCommands", error, displayId, UNKNOWN_ERROR);
        RETURN_IF_HWC_ERROR_FOR("present", displayData.presentError, displayId, UNKNOWN_ERROR);
        return NO_ERROR;
    }

    auto error = hwcDisplay->present(&displayData.lastPresentFence);
    RETURN_IF_HWC_ERROR_FOR("present", error, displayId, UNKNOWN_ERROR);

    std::unordered_map<HWC2::Layer*, sp<Fence>> releaseFences;
    error = hwcDisplay->getReleaseFences(&releaseFences);
    RETURN_IF_HWC_ERROR_FOR("getReleaseFences", error, displayId, UNKNOWN_ERROR);

    displayData.releaseFences = std::move(releaseFences);

    return NO_ERROR;
}

X:\frameworks\native\services\surfaceflinger\CompositionEngine\src\RenderSurface.cpp
void RenderSurface::onPresentDisplayCompleted() {
    mDisplaySurface->onFrameCommitted();
}
X:\frameworks\native\services\surfaceflinger\DisplayHardware\FramebufferSurface.cpp
void FramebufferSurface::onFrameCommitted() {
	ALOGI("zjj.rk3399.SF onFrameCommitted %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
	ALOGD_CALLSTACK("onFrameCommitted");

    if (mHasPendingRelease) {
        sp<Fence> fence = mHwc.getPresentFence(mDisplayId);
        if (fence->isValid()) {
            status_t result = addReleaseFence(mPreviousBufferSlot,
                    mPreviousBuffer, fence);
            ALOGE_IF(result != NO_ERROR, "onFrameCommitted: failed to add the"
                    " fence: %s (%d)", strerror(-result), result);
        }
        status_t result = releaseBufferLocked(mPreviousBufferSlot, mPreviousBuffer);
        ALOGE_IF(result != NO_ERROR, "onFrameCommitted: error releasing buffer:"
                " %s (%d)", strerror(-result), result);

        mPreviousBuffer.clear();
        mHasPendingRelease = false;
    }
}

X:\frameworks\native\libs\gui\ConsumerBase.cpp
status_t ConsumerBase::releaseBufferLocked(
        int slot, const sp<GraphicBuffer> graphicBuffer,
        EGLDisplay display, EGLSyncKHR eglFence) {
    if (mAbandoned) {
        CB_LOGE("releaseBufferLocked: ConsumerBase is abandoned!");
        return NO_INIT;
    }
    // If consumer no longer tracks this graphicBuffer (we received a new
    // buffer on the same slot), the buffer producer is definitely no longer
    // tracking it.
    if (!stillTracking(slot, graphicBuffer)) {
        return OK;
    }

    CB_LOGV("releaseBufferLocked: slot=%d/%" PRIu64,
            slot, mSlots[slot].mFrameNumber);
    status_t err = mConsumer->releaseBuffer(slot, mSlots[slot].mFrameNumber,
            display, eglFence, mSlots[slot].mFence);
    if (err == IGraphicBufferConsumer::STALE_BUFFER_SLOT) {
        freeBufferLocked(slot);
    }

    mPrevFinalReleaseFence = mSlots[slot].mFence;
    mSlots[slot].mFence = Fence::NO_FENCE;

    return err;
}

```
hwcDisplay->present看看log:

``` bash
Stack Trace:
  RELADDR           FUNCTION                                                                                  FILE:LINE
  000000000009f0a0  HWC2::impl::Display::present(android::sp<android::Fence>*)+144                            frameworks/native/services/surfaceflinger/DisplayHardware/HWC2.cpp:602
  00000000000a79e0  android::impl::HWComposer::presentAndGetReleaseFences(android::DisplayId)+520             frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp:578
  00000000000e2f40  android::SurfaceFlinger::postFramebuffer(android::sp<android::DisplayDevice> const&)+232  frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:2613
  00000000000e0b9c  android::SurfaceFlinger::handleMessageRefresh()+4348                                      frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:1886
  00000000000df6bc  android::SurfaceFlinger::onMessageReceived(int)+9484                                      frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:1834
  0000000000017dfc  android::Looper::pollInner(int)+332                                                       system/core/libutils/Looper.cpp:323
  0000000000017c10  android::Looper::pollOnce(int, int*, int*, void**)+56                                     system/core/libutils/Looper.cpp:205
  v-------------->  android::Looper::pollOnce(int)                                                            system/core/libutils/include/utils/Looper.h:267
  00000000000ced04  android::impl::MessageQueue::waitMessage()+92                                             frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp:120
  v-------------->  android::SurfaceFlinger::waitForEvent()                                                   frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:1470
  00000000000dc69c  android::SurfaceFlinger::run()+20                                                         frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:1504
  0000000000003370  main+800                                                                                  frameworks/native/services/surfaceflinger/main_surfaceflinger.cpp:120
  000000000007d844  __libc_init+108                                                                           bionic/libc/bionic/libc_init_dynamic.cpp:136





Stack Trace:
  RELADDR           FUNCTION                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          FILE:LINE
  0000000000002440  drm_mod_perform(gralloc_module_t const*, int, ...)+120                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            hardware/rockchip/libgralloc/midgard/gralloc.cpp:95
  v-------------->  android::hwc_get_handle_alreadyStereo(gralloc_module_t const*, native_handle const*)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              hardware/rockchip/hwcomposer/hwc_rockchip.cpp:494
  0000000000060958  android::detect_3d_mode(android::hwc_drm_display*, hwc_display_contents_1*, int)+176                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              hardware/rockchip/hwcomposer/hwc_rockchip.cpp:200
  00000000000536b4  android::hwc_prepare(hwc_composer_device_1*, unsigned long, hwc_display_contents_1**)+12044                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       hardware/rockchip/hwcomposer/hwcomposer.cpp:2564
  00000000000151cc  android::HWC2On1Adapter::prepareAllDisplays()+1156                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                hardware/interfaces/graphics/composer/2.1/utils/hwc2on1adapter/HWC2On1Adapter.cpp:2395
  0000000000014cdc  android::HWC2On1Adapter::Display::validate(unsigned int*, unsigned int*)+92                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       hardware/interfaces/graphics/composer/2.1/utils/hwc2on1adapter/HWC2On1Adapter.cpp:984
  00000000000092f8  android::hardware::graphics::composer::V2_1::passthrough::detail::HwcHalImpl<android::hardware::graphics::composer::V2_1::hal::ComposerHal>::validateDisplay(unsigned long, std::__1::vector<unsigned long, std::__1::allocator<unsigned long> >*, std::__1::vector<android::hardware::graphics::composer::V2_1::IComposerClient::Composition, std::__1::allocator<android::hardware::graphics::composer::V2_1::IComposerClient::Composition> >*, unsigned int*, std::__1::vector<unsigned long, std::__1::allocator<unsigned long> >*, std::__1::vector<unsigned int, std::__1::allocator<unsigned int> >*)+112  hardware/interfaces/graphics/composer/2.1/utils/passthrough/include/composer-passthrough/2.1/HwcHal.h:334
  0000000000011510  android::hardware::graphics::composer::V2_1::hal::ComposerCommandEngine::executePresentOrValidateDisplay(unsigned short)+424                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      hardware/interfaces/graphics/composer/2.1/utils/hal/include/composer-hal/2.1/ComposerCommandEngine.h:300
  0000000000010adc  android::hardware::graphics::composer::V2_1::hal::ComposerCommandEngine::executeCommand(android::hardware::graphics::composer::V2_1::IComposerClient::Command, unsigned short)+1540                                                                                                                                                                                                                                                                                                                                                                                                                               hardware/interfaces/graphics/composer/2.1/utils/hal/include/composer-hal/2.1/ComposerCommandEngine.h:105
  000000000000f168  android::hardware::graphics::composer::V2_1::hal::ComposerCommandEngine::execute(unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> const&, bool*, unsigned int*, android::hardware::hidl_vec<android::hardware::hidl_handle>*)+128                                                                                                                                                                                                                                                                                                                                                        hardware/interfaces/graphics/composer/2.1/utils/hal/include/composer-hal/2.1/ComposerCommandEngine.h:64
  000000000000c604  android::hardware::graphics::composer::V2_1::hal::detail::ComposerClientImpl<android::hardware::graphics::composer::V2_1::IComposerClient, android::hardware::graphics::composer::V2_1::hal::ComposerHal>::executeCommands(unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> const&, std::__1::function<void (android::hardware::graphics::composer::V2_1::Error, bool, unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> const&)>)+116                                                                                                                           hardware/interfaces/graphics/composer/2.1/utils/hal/include/composer-hal/2.1/ComposerClient.h:305
  00000000000388e4  android::hardware::graphics::composer::V2_1::BnHwComposerClient::_hidl_executeCommands(android::hidl::base::V1_0::BnHwBase*, android::hardware::Parcel const&, android::hardware::Parcel*, std::__1::function<void (android::hardware::Parcel&)>)+524                                                                                                                                                                                                                                                                                                                                                             out/soong/.intermediates/hardware/interfaces/graphics/composer/2.1/android.hardware.graphics.composer@2.1_genc++/gen/android/hardware/graphics/composer/2.1/ComposerClientAll.cpp:3710
  0000000000039590  android::hardware::graphics::composer::V2_1::BnHwComposerClient::onTransact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+2672                                                                                                                                                                                                                                                                                                                                                                                 out/soong/.intermediates/hardware/interfaces/graphics/composer/2.1/android.hardware.graphics.composer@2.1_genc++/gen/android/hardware/graphics/composer/2.1/ComposerClientAll.cpp:4058
  0000000000096604  android::hardware::BHwBinder::transact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+68                                                                                                                                                                                                                                                                                                                                                                                                                        system/libhwbinder/Binder.cpp:116
  v-------------->  android::hardware::IPCThreadState::executeCommand(int)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            system/libhwbinder/IPCThreadState.cpp:1204
  0000000000099fcc  android::hardware::IPCThreadState::getAndExecuteCommand()+1036                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    system/libhwbinder/IPCThreadState.cpp:461
  000000000009b1e0  android::hardware::IPCThreadState::joinThreadPool(bool)+96                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        system/libhwbinder/IPCThreadState.cpp:561
  00000000000a9df0  android::hardware::PoolThread::threadLoop()+24                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    system/libhwbinder/ProcessState.cpp:61
  00000000000137a4  android::Thread::_threadLoop(void*)+284                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           system/core/libutils/Threads.cpp:746
  00000000000e230c  __pthread_start(void*)+36                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         bionic/libc/bionic/pthread_create.cpp:338
  0000000000083d98  __start_thread+64             

```

### 八、postFrame()函数

``` cpp
void SurfaceFlinger::postFrame()
{
    // |mStateLock| not needed as we are on the main thread
	ALOGI("zjj.rk3399.SF postFrame %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

	const auto display = getDefaultDisplayDeviceLocked();
    if (display && getHwComposer().isConnected(*display->getId())) {
        uint32_t flipCount = display->getPageFlipCount();
        if (flipCount % LOG_FRAME_STATS_PERIOD == 0) {
            logFrameStats();
        }
    }
}
```

### 九、postComposition()函数

``` cpp
void SurfaceFlinger::postComposition()
{
    ATRACE_CALL();
	ALOGI("zjj.rk3399.SF postComposition %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

    // Release any buffers which were replaced this frame
    nsecs_t dequeueReadyTime = systemTime();
    for (auto& layer : mLayersWithQueuedFrames) {
        layer->releasePendingBuffer(dequeueReadyTime);
    }

    // |mStateLock| not needed as we are on the main thread
    const auto displayDevice = getDefaultDisplayDeviceLocked();

    getBE().mGlCompositionDoneTimeline.updateSignalTimes();
    std::shared_ptr<FenceTime> glCompositionDoneFenceTime;
    if (displayDevice && getHwComposer().hasClientComposition(displayDevice->getId())) {
        glCompositionDoneFenceTime =
                std::make_shared<FenceTime>(displayDevice->getCompositionDisplay()
                                                    ->getRenderSurface()
                                                    ->getClientTargetAcquireFence());
        getBE().mGlCompositionDoneTimeline.push(glCompositionDoneFenceTime);
    } else {
        glCompositionDoneFenceTime = FenceTime::NO_FENCE;
    }

    getBE().mDisplayTimeline.updateSignalTimes();
    mPreviousPresentFences[1] = mPreviousPresentFences[0];
    mPreviousPresentFences[0] = displayDevice
            ? getHwComposer().getPresentFence(*displayDevice->getId())
            : Fence::NO_FENCE;
    auto presentFenceTime = std::make_shared<FenceTime>(mPreviousPresentFences[0]);
    getBE().mDisplayTimeline.push(presentFenceTime);

    DisplayStatInfo stats;
    mScheduler->getDisplayStatInfo(&stats);

    // We use the mRefreshStartTime which might be sampled a little later than
    // when we started doing work for this frame, but that should be okay
    // since updateCompositorTiming has snapping logic.
    updateCompositorTiming(stats, mRefreshStartTime, presentFenceTime);
    CompositorTiming compositorTiming;
    {
        std::lock_guard<std::mutex> lock(getBE().mCompositorTimingLock);
        compositorTiming = getBE().mCompositorTiming;
    }

    mDrawingState.traverseInZOrder([&](Layer* layer) {
        bool frameLatched =
                layer->onPostComposition(displayDevice->getId(), glCompositionDoneFenceTime,
                                         presentFenceTime, compositorTiming);
        if (frameLatched) {
            recordBufferingStats(layer->getName().string(),
                    layer->getOccupancyHistory(false));
        }
    });

    if (presentFenceTime->isValid()) {
        mScheduler->addPresentFence(presentFenceTime);
    }

    if (!hasSyncFramework) {
        if (displayDevice && getHwComposer().isConnected(*displayDevice->getId()) &&
            displayDevice->isPoweredOn()) {
            mScheduler->enableHardwareVsync();
        }
    }

    if (mAnimCompositionPending) {
        mAnimCompositionPending = false;

        if (presentFenceTime->isValid()) {
            mAnimFrameTracker.setActualPresentFence(
                    std::move(presentFenceTime));
        } else if (displayDevice && getHwComposer().isConnected(*displayDevice->getId())) {
            // The HWC doesn't support present fences, so use the refresh
            // timestamp instead.
            const nsecs_t presentTime =
                    getHwComposer().getRefreshTimestamp(*displayDevice->getId());
            mAnimFrameTracker.setActualPresentTime(presentTime);
        }
        mAnimFrameTracker.advanceFrame();
    }

    mTimeStats->incrementTotalFrames();
    if (mHadClientComposition) {
        mTimeStats->incrementClientCompositionFrames();
    }

    mTimeStats->setPresentFenceGlobal(presentFenceTime);

    if (displayDevice && getHwComposer().isConnected(*displayDevice->getId()) &&
        !displayDevice->isPoweredOn()) {
        return;
    }

    nsecs_t currentTime = systemTime();
    if (mHasPoweredOff) {
        mHasPoweredOff = false;
    } else {
        nsecs_t elapsedTime = currentTime - getBE().mLastSwapTime;
        size_t numPeriods = static_cast<size_t>(elapsedTime / stats.vsyncPeriod);
        if (numPeriods < SurfaceFlingerBE::NUM_BUCKETS - 1) {
            getBE().mFrameBuckets[numPeriods] += elapsedTime;
        } else {
            getBE().mFrameBuckets[SurfaceFlingerBE::NUM_BUCKETS - 1] += elapsedTime;
        }
        getBE().mTotalTime += elapsedTime;
    }
    getBE().mLastSwapTime = currentTime;

    {
        std::lock_guard lock(mTexturePoolMutex);
        if (mTexturePool.size() < mTexturePoolSize) {
            const size_t refillCount = mTexturePoolSize - mTexturePool.size();
            const size_t offset = mTexturePool.size();
            mTexturePool.resize(mTexturePoolSize);
            getRenderEngine().genTextures(refillCount, mTexturePool.data() + offset);
            ATRACE_INT("TexturePoolSize", mTexturePool.size());
        } else if (mTexturePool.size() > mTexturePoolSize) {
            const size_t deleteCount = mTexturePool.size() - mTexturePoolSize;
            const size_t offset = mTexturePoolSize;
            getRenderEngine().deleteTextures(deleteCount, mTexturePool.data() + offset);
            mTexturePool.resize(mTexturePoolSize);
            ATRACE_INT("TexturePoolSize", mTexturePool.size());
        }
    }

    mTransactionCompletedThread.addPresentFence(mPreviousPresentFences[0]);

    // Lock the mStateLock in case SurfaceFlinger is in the middle of applying a transaction.
    // If we do not lock here, a callback could be sent without all of its SurfaceControls and
    // metrics.
    {
        Mutex::Autolock _l(mStateLock);
        mTransactionCompletedThread.sendCallbacks();
    }

    if (mLumaSampling && mRegionSamplingThread) {
        mRegionSamplingThread->notifyNewContent();
    }

    // Even though ATRACE_INT64 already checks if tracing is enabled, it doesn't prevent the
    // side-effect of getTotalSize(), so we check that again here
    if (ATRACE_ENABLED()) {
        ATRACE_INT64("Total Buffer Size", GraphicBufferAllocator::get().getTotalSize());
    }
}


```
