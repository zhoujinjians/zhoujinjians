---
title: Android P Graphics System（四）：Native Surface创建 && SurfaceFlinger合成流程分析
cover: https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/hexo.themes/bing-wallpaper-2018.04.39.jpg
categories:
  - Graphics
tags:
  - Android
  - Graphics
toc: true
abbrlink: 20190716
date: 2019-07-16 09:25:00
---



--------------------------------------------------------------------------------

注：文章都是通过阅读各位前辈总结的资料 Android 9.0 && Linux（Kernel 3.18）Qualcomm平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除（◕‿◕），转载请注明出处（©Qualcomm ©Android @Linux 版权所有），谢谢。
（==**文章基于 Kernel-3.18**==）&&（==**文章基于 Android 9.0**==）

[【开发板 Intrinsyc Open-Q™ 820 µSOM Development Kit】](https://www.intrinsyc.com/snapdragon-embedded-development-kits/open-q-820-usom-development-kit/)
[【开发板 Android 9.0 && Linux（Kernel 3.18）源码链接】](https://gitlab.com/zhoujinjiancc/apq8096_la.um.7.5.r1-03100-8x96.0_p_v5.0)

正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢！！！

[（1）【Android P 图形显示系统】](https://www.jianshu.com/u/f92447ae8445/) 
[（2）【深入剖析Android系统】](https://blog.csdn.net/yangwen123/) 
[（3）【Android Display System】](http://charlesvincent.cc/)
[（4）【Android SurfaceFlinger 学习之路】](http://windrunnerlihuan.com/)

--------------------------------------------------------------------------------
==源码（部分）==：

>SurfaceFlinger

-  frameworks/native/services/surfaceflinger


--------------------------------------------------------------------------------


#### （一）、Native Surface创建过程

##### 1.1.0 、Native Surface创建步骤

``` cpp
sp<SurfaceComposerClient> client = new SurfaceComposerClient();  
sp<SurfaceControl> surfaceControl = client->createSurface(String8("new_surface"),  480, 854, PIXEL_FORMAT_RGBA_8888, 0);  
sp<Surface> surface = surfaceControl->getSurface();  
SurfaceComposerClient::openGlobalTransaction();  
surfaceControl->setLayer(100000);  
SurfaceComposerClient::closeGlobalTransaction();  
ANativeWindow_Buffer outBuffer; 
surface->lock(&outBuffer, NULL);  
//填充或绘制图像数据
//fillBufferColor(outBuffer, Rect(0, 0, buffer.width, buffer.height), COLOR::RED)
surface->unlockAndPost();  
```
下图是Android 7.1.2分析的流程图，大体流程相同，这里就不重新画流程图了。
![Alt text | center](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/display.system/Android.PG4.ask.SurfaceFlinger.createSurface.png)

[->SurfaceComposerClient.cpp]

``` cpp
sp<SurfaceControl> SurfaceComposerClient::createSurface(
    const String8& name,
    uint32_t w,
    uint32_t h,
    PixelFormat format,
    uint32_t flags)
    {
sp<SurfaceControl> sur;
if (mStatus == NO_ERROR) {
    sp<IBinder> handle;
    sp<IGraphicBufferProducer> gbp;
    status_t err = mClient->createSurface(name, w, h, format, flags,
            &handle, &gbp);
    ALOGE_IF(err, "SurfaceComposerClient::createSurface error %s", strerror(-err));
    if (err == NO_ERROR) {
        sur = new SurfaceControl(this, handle, gbp);
    }
}
return sur;
}
```

SurfaceComposerClient将Surface创建请求转交给保存在其成员变量中的Bp SurfaceComposerClient对象来完成，在SurfaceFlinger端的Client本地对象会返回一个ISurface代理对象给应用程序，通过该代理对象为应用程序当前创建的Surface创建一个SurfaceControl对象。
[ISurfaceComposerClient.cpp]

``` cpp
    virtual status_t createSurface(const String8& name, uint32_t width,
        uint32_t height, PixelFormat format, uint32_t flags,
        sp<IBinder>* handle,
        sp<IGraphicBufferProducer>* gbp) {
    Parcel data, reply;
    ......
    remote()->transact(CREATE_SURFACE, data, &reply);
    *handle = reply.readStrongBinder();
    *gbp = interface_cast<IGraphicBufferProducer>(reply.readStrongBinder());
    return reply.readInt32();
}
```

[Client.cpp]
MessageCreateSurface消息是专门为应用程序请求创建Surface而定义的一种消息类型：

``` cpp
    status_t Client::createSurface(
        const String8& name,
        uint32_t w, uint32_t h, PixelFormat format, uint32_t flags,
        sp<IBinder>* handle,
        sp<IGraphicBufferProducer>* gbp){
    /*
     * createSurface must be called from the GL thread so that it can
     * have access to the GL context.
     */

    class MessageCreateLayer : public MessageBase {
        SurfaceFlinger* flinger;
        Client* client;
        sp<IBinder>* handle;
        sp<IGraphicBufferProducer>* gbp;
        status_t result;
        const String8& name;
        uint32_t w, h;
        PixelFormat format;
        uint32_t flags;
    public:
        MessageCreateLayer(SurfaceFlinger* flinger,
                const String8& name, Client* client,
                uint32_t w, uint32_t h, PixelFormat format, uint32_t flags,
                sp<IBinder>* handle,
                sp<IGraphicBufferProducer>* gbp)
            : flinger(flinger), client(client),
              handle(handle), gbp(gbp), result(NO_ERROR),
              name(name), w(w), h(h), format(format), flags(flags) {
        }
        status_t getResult() const { return result; }
        virtual bool handler() {
            result = flinger->createLayer(name, client, w, h, format, flags,
                    handle, gbp);
            return true;
        }
    };

    sp<MessageBase> msg = new MessageCreateLayer(mFlinger.get(),
            name, this, w, h, format, flags, handle, gbp);
    mFlinger->postMessageSync(msg);
    return static_cast<MessageCreateLayer*>( msg.get() )->getResult();
    }
Client将应用程序创建Surface的请求转换为异步消息投递到SurfaceFlinger的消息队列中，将创建Surface的任务转交给SurfaceFlinger。
[->SurfaceFlinger.cpp]

    status_t SurfaceFlinger::createLayer(
        const String8& name,
        const sp<Client>& client,
        uint32_t w, uint32_t h, PixelFormat format, uint32_t flags,
        sp<IBinder>* handle, sp<IGraphicBufferProducer>* gbp){
    //ALOGD("createLayer for (%d x %d), name=%s", w, h, name.string());
    ......

    status_t result = NO_ERROR;

    sp<Layer> layer;
    ////根据flags创建不同类型的layer
    switch (flags & ISurfaceComposerClient::eFXSurfaceMask) {
        case ISurfaceComposerClient::eFXSurfaceNormal:
            result = createNormalLayer(client,
                    name, w, h, flags, format,
                    handle, gbp, &layer);
            break;
        case ISurfaceComposerClient::eFXSurfaceDim:
            result = createDimLayer(client,
                    name, w, h, flags,
                    handle, gbp, &layer);
            break;
        default:
            result = BAD_VALUE;
            break;
    }

    if (result != NO_ERROR) {
        return result;
    }
    //将创建好的Layer对象保存在Client中  
    result = addClientLayer(client, *handle, *gbp, layer);
    if (result != NO_ERROR) {
        return result;
    }

    setTransactionFlags(eTransactionNeeded);
    return result;
    }
```

  SurfaceFlinger根据标志位创建对应类型的Surface，当前系统定义了3种类型的Layer:
  [->ISurfaceComposerClient.h]

``` cpp
eFXSurfaceNormal    = 0x00000000,
eFXSurfaceDim       = 0x00020000,
eFXSurfaceMask      = 0x000F0000
```

[->SurfaceFlinger.cpp]

``` cpp
status_t SurfaceFlinger::createNormalLayer(const sp<Client>& client,
    const String8& name, uint32_t w, uint32_t h, uint32_t flags, PixelFormat& format,
    sp<IBinder>* handle, sp<IGraphicBufferProducer>* gbp, sp<Layer>* outLayer){
// initialize the surfaces
switch (format) {
case PIXEL_FORMAT_TRANSPARENT:
case PIXEL_FORMAT_TRANSLUCENT:
    format = PIXEL_FORMAT_RGBA_8888;
    break;
case PIXEL_FORMAT_OPAQUE:
    format = PIXEL_FORMAT_RGBX_8888;
    break;
}
//在SurfaceFlinger端为应用程序的Surface创建对应的Layer对象  
*outLayer = new Layer(this, client, name, w, h, flags);
status_t err = (*outLayer)->setBuffers(w, h, format, flags);
if (err == NO_ERROR) {
    *handle = (*outLayer)->getHandle();
    *gbp = (*outLayer)->getProducer();
}

ALOGE_IF(err, "createNormalLayer() failed (%s)", strerror(-err));
return err;
}
```

在SurfaceFlinger服务端为应用程序创建的Surface创建对应的Layer对象。


#### （二）、Android SurfaceFlinger内部机制

##### 2.1.0 、BufferQueue介绍
BufferQueue 类是 Android 中所有图形处理操作的核心。它的是将生成图形数据缓冲区的一方（生产者Producer）连接到接受数据以进行显示或进一步处理的一方（消费者Consumer）。几乎所有在系统中移动图形数据缓冲区的内容都依赖于 BufferQueue。
从上图APP与SurfaceFlinger交互中可以看出，BufferQueue内部维持着64个BufferSlot，每一个BufferSlot内部有一个GraphicBuffer指向分配的Graphic Buffer。
![Alt text | center](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/display.system/Android.PG4.SurfaceFlinger-BufferQueue.png.png)

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
![Alt text | center](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/display.system/Android.PG4.SurfaceFlinger-IGraphicsBufferProducer.png)


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
![Alt text | center](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/display.system/Android.PG4.SurfaceFlinger-IGraphicsBufferConsumer.png)

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
status_t Surface::lock(
    ANativeWindow_Buffer* outBuffer, ARect* inOutDirtyBounds)
    {
......

ANativeWindowBuffer* out;
int fenceFd = -1;
//调用dequeueBuffer函数，申请图形缓冲区
status_t err = dequeueBuffer(&out, &fenceFd);
ALOGE_IF(err, "dequeueBuffer failed (%s)", strerror(-err));
if (err == NO_ERROR) {
    //获取图形缓冲区区域大小,赋给后备缓冲区变量backBuffer
    sp<GraphicBuffer> backBuffer(GraphicBuffer::getSelf(out));
    const Rect bounds(backBuffer->width, backBuffer->height);
    Region newDirtyRegion;
    if (inOutDirtyBounds) {
        //如果上层指定乐刷新脏矩形区域，则用这个区域和缓冲区区域求交集，
        //然后将交集的结果设给需要去刷新的新区域
        newDirtyRegion.set(static_cast<Rect const&>(*inOutDirtyBounds));
        newDirtyRegion.andSelf(bounds);
    } else {
        /如果上层没有指定脏矩形区域，所以刷新整个图形缓冲区
        newDirtyRegion.set(bounds);
    }
     
    // figure out if we can copy the frontbuffer back
    //上一次绘制的信息保存在mPostedBuffer中，而这个mPostedBuffer则要在unLockAndPost函数中设置
    int backBufferSlot(getSlotFromBufferLocked(backBuffer.get()));
    const sp<GraphicBuffer>& frontBuffer(mPostedBuffer);
    const bool canCopyBack = (frontBuffer != 0 &&
            backBuffer->width  == frontBuffer->width &&
            backBuffer->height == frontBuffer->height &&
            backBuffer->format == frontBuffer->format);

    if (canCopyBack) {
        Mutex::Autolock lock(mMutex);
        Region oldDirtyRegion;
        if(mSlots[backBufferSlot].dirtyRegion.isEmpty()) {
            oldDirtyRegion.set(bounds);
        } else {
            for(int i = 0 ; i < NUM_BUFFER_SLOTS; i++ ) {
                if(i != backBufferSlot && !mSlots[i].dirtyRegion.isEmpty())
                    oldDirtyRegion.orSelf(mSlots[i].dirtyRegion);
            }
        }
        const Region copyback(oldDirtyRegion.subtract(newDirtyRegion));
        if (!copyback.isEmpty())
        //这里把mPostedBuffer中的旧数据拷贝到BackBuffer中。
            //后续的绘画只要更新脏区域就可以了，这会节约不少资源
            copyBlt(backBuffer, frontBuffer, copyback);
    } else {
        // if we can't copy-back anything, modify the user's dirty
        // region to make sure they redraw the whole buffer
        //如果两次图形缓冲区大小不一致，我们就要修改用户指定的dirty区域大小为整个缓冲区大小，
        //然后去更新整个缓冲区
        newDirtyRegion.set(bounds);
        Mutex::Autolock lock(mMutex);
        for (size_t i=0 ; i<NUM_BUFFER_SLOTS ; i++) {
            mSlots[i].dirtyRegion.clear();
        }
    }


    { // scope for the lock
        Mutex::Autolock lock(mMutex);
        //将新的dirty赋给这个bufferslot
        mSlots[backBufferSlot].dirtyRegion = newDirtyRegion;
    }

    if (inOutDirtyBounds) {
        *inOutDirtyBounds = newDirtyRegion.getBounds();
    }

    void* vaddr;
     //lock和unlock分别用来锁定和解锁一个指定的图形缓冲区，在访问一块图形缓冲区的时候，
    //例如，向一块图形缓冲写入内容的时候，需要将该图形缓冲区锁定，用来避免访问冲突,
    //锁定之后，就可以获得由参数参数l、t、w和h所圈定的一块缓冲区的起始地址，保存在输出参数vaddr中
    status_t res = backBuffer->lockAsync(
            GRALLOC_USAGE_SW_READ_OFTEN | GRALLOC_USAGE_SW_WRITE_OFTEN,
            newDirtyRegion.bounds(), &vaddr, fenceFd);
    ......
}
return err;
}
```

Surface的lock函数用来申请图形缓冲区和一些操作，方法不长，大概工作有：
       1）调用connect函数完成一些初始化；
       2）调用dequeueBuffer函数，申请图形缓冲区；
       3）计算需要绘制的新的dirty区域，旧的区域原样copy数据。
       [->BufferQueueProducer.cpp]
       
``` cpp
int Surface::dequeueBuffer(android_native_buffer_t** buffer, int* fenceFd) {
uint32_t reqWidth;
uint32_t reqHeight;
PixelFormat reqFormat;
uint32_t reqUsage;
{
 ......
//申请图形缓冲区
status_t result = mGraphicBufferProducer->dequeueBuffer(&buf, &fence,
        reqWidth, reqHeight, reqFormat, reqUsage);
......
//根据index获取缓冲区
sp<GraphicBuffer>& gbuf(mSlots[buf].buffer);
......
if ((result & IGraphicBufferProducer::BUFFER_NEEDS_REALLOCATION) || gbuf == 0) {
    //由于申请的内存是在surfaceflinger进程中，
    //BufferQueue中的图形缓冲区也是通过匿名共享内存和binder传递描述符映射过去的，
    //Surface通过调用requestBuffer将图形缓冲区映射到Surface所在进程
    result = mGraphicBufferProducer->requestBuffer(buf, &gbuf);
    ......
}
......
//获取这个这个buffer对象的指针内容
*buffer = gbuf.get();
......
return OK;
}
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

 这个比较简单，还是很好理解的额，就是根据指定index取出mSlots中的slot中的buffer。

##### 3.1.1、APP提交(unlockAndPost)Buffer的过程
Surface绘制完毕后，unlockCanvasAndPost操作。
[->android_view_Surface.cpp]

``` cpp
static void nativeUnlockCanvasAndPost(JNIEnv* env, jclass clazz,
    jlong nativeObject, jobject canvasObj) {
sp<Surface> surface(reinterpret_cast<Surface *>(nativeObject));
if (!isSurfaceValid(surface)) {
    return;
}

// detach the canvas from the surface
Canvas* nativeCanvas = GraphicsJNI::getNativeCanvas(env, canvasObj);
nativeCanvas->setBitmap(SkBitmap());

// unlock surface
status_t err = surface->unlockAndPost();
if (err < 0) {
    doThrowIAE(env);
}
}
```

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

#### （四）、通知SF消费合成
当绘制完毕的GraphicBuffer入队之后，会通知SurfaceFlinger去消费，就是BufferQueueProducer的queueBuffer函数的最后几行，listener->onFrameAvailable()。
listener最终通过回调，会回到BufferLayer当中，所以最终调用BufferLayer的onFrameAvailable接口，我们看看它的实现：
[Layer.cpp]

``` cpp
void BufferLayer::onFrameAvailable(const BufferItem& item) {
    // Add this buffer from our internal queue tracker
    { // Autolock scope
        ......
        mQueueItems.push_back(item);
        android_atomic_inc(&mQueuedFrames);

        // Wake up any pending callbacks
        mLastFrameNumberReceived = item.mFrameNumber;
        mQueueItemCondition.broadcast();
    }

    mFlinger->signalLayerUpdate();
}
```

这里又调用SurfaceFlinger的signalLayerUpdate函数，继续查看：
[SurfaceFlinger.cpp]

``` cpp
void SurfaceFlinger::signalLayerUpdate() {
mEventQueue.invalidate();
}
```

这里又调用MessageQueue的invalidate函数：
[MessageQueue.cpp]

``` cpp
void MessageQueue::invalidate() {
mEvents->requestNextVsync();
}
```
贴一下SurfaceFlinger的初始化请求vsync信号流程图：
![Alt text | center](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/display.system/Android.PG4.Vsync-surfaceflinger.init.png)

MessageQueue中分发两个消息，一个INVALIDATE，一个REFRESH，SurfaceFlinger对这两个消息的响应过程，就是合成的过程。

##### 4.0、消息INVALIDATE处理
最终结果会走到SurfaceFlinger的vsync信号接收逻辑，即SurfaceFlinger的onMessageReceived函数：
[SurfaceFlinger.cpp]

``` cpp
void SurfaceFlinger::onMessageReceived(int32_t what) {
    ATRACE_CALL();
    switch (what) {
        case MessageQueue::INVALIDATE: {
            bool frameMissed = !mHadClientComposition &&
                    mPreviousPresentFence != Fence::NO_FENCE &&
                    (mPreviousPresentFence->getSignalTime() ==
                            Fence::SIGNAL_TIME_PENDING);
            ATRACE_INT("FrameMissed", static_cast<int>(frameMissed));
            if (mPropagateBackpressure && frameMissed) {
                signalLayerUpdate();
                break;
            }

            updateVrFlinger();

            bool refreshNeeded = handleMessageTransaction();
            refreshNeeded |= handleMessageInvalidate();
            refreshNeeded |= mRepaintEverything;
            if (refreshNeeded) {
                signalRefresh();
            }
            break;
        }
        case MessageQueue::REFRESH: {
            handleMessageRefresh();
            break;
        }
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
    return handlePageFlip();
}
```
主要是调用handlePageFlip，做Page的Flip。

``` cpp
bool SurfaceFlinger::handlePageFlip()
{
    ALOGV("handlePageFlip");

    nsecs_t latchTime = systemTime();

    bool visibleRegions = false;
    bool frameQueued = false;
    bool newDataLatched = false;

    mDrawingState.traverseInZOrder([&](Layer* layer) {
        if (layer->hasQueuedFrame()) {
            frameQueued = true;
            if (layer->shouldPresentNow(mPrimaryDispSync)) {
                mLayersWithQueuedFrames.push_back(layer);
            } else {
                layer->useEmptyDamage();
            }
        } else {
            layer->useEmptyDamage();
        }
    });

    for (auto& layer : mLayersWithQueuedFrames) {
        const Region dirty(layer->latchBuffer(visibleRegions, latchTime));
        layer->useSurfaceDamage();
        invalidateLayerStack(layer, dirty);
        if (layer->isBufferLatched()) {
            newDataLatched = true;
        }
    }

    mVisibleRegionsDirty |= visibleRegions;

    // If we will need to wake up at some time in the future to deal with a
    // queued frame that shouldn't be displayed during this vsync period, wake
    // up during the next vsync period to check again.
    if (frameQueued && (mLayersWithQueuedFrames.empty() || !newDataLatched)) {
        signalLayerUpdate();
    }

    // Only continue with the refresh if there is actually new work to do
    return !mLayersWithQueuedFrames.empty() && newDataLatched;
}
```
mLayersWithQueuedFrames，用于标记那些已经有Frame的Layer，这得从Layer的onFrameAvailable说起。

``` cpp
void BufferLayer::onFrameAvailable(const BufferItem& item) {
    // Add this buffer from our internal queue tracker
    { // Autolock scope
        Mutex::Autolock lock(mQueueItemLock);
        mFlinger->mInterceptor.saveBufferUpdate(this, item.mGraphicBuffer->getWidth(),
                                                item.mGraphicBuffer->getHeight(),
                                                item.mFrameNumber);
        // Reset the frame number tracker when we receive the first buffer after
        // a frame number reset
        if (item.mFrameNumber == 1) {
            mLastFrameNumberReceived = 0;
        }

        // Ensure that callbacks are handled in order
        while (item.mFrameNumber != mLastFrameNumberReceived + 1) {
            status_t result = mQueueItemCondition.waitRelative(mQueueItemLock,
                                                               ms2ns(500));
            if (result != NO_ERROR) {
                ALOGE("[%s] Timed out waiting on callback", mName.string());
            }
        }

        mQueueItems.push_back(item);
        android_atomic_inc(&mQueuedFrames);

        // Wake up any pending callbacks
        mLastFrameNumberReceived = item.mFrameNumber;
        mQueueItemCondition.broadcast();
    }

    mFlinger->signalLayerUpdate();
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

Region BufferLayer::latchBuffer(bool& recomputeVisibleRegions, nsecs_t latchTime) {
    ATRACE_CALL();

    if (android_atomic_acquire_cas(true, false, &mSidebandStreamChanged) == 0) {
        // mSidebandStreamChanged was true
        mSidebandStream = mConsumer->getSidebandStream();
        // replicated in LayerBE until FE/BE is ready to be synchronized
        getBE().compositionInfo.hwc.sidebandStream = mSidebandStream;
        if (getBE().compositionInfo.hwc.sidebandStream != NULL) {
            setTransactionFlags(eTransactionNeeded);
            mFlinger->setTransactionFlags(eTraversalNeeded);
        }
        recomputeVisibleRegions = true;

        const State& s(getDrawingState());
        return getTransform().transform(Region(Rect(s.active.w, s.active.h)));
    }
    ... ...
```
android_atomic_acquire_cas是比较-设置的原子操纵函数，如果变量第三个参数和第一个相等，那么将第二个参数赋值给第三个参数，成功返回0。mSidebandStreamChanged为true，说明Sideband流改变了，这里处理后，就返回了。

``` cpp
* frameworks/native/services/surfaceflinger/BufferLayer.cpp
    ... ...

    Region outDirtyRegion;
    ... ... // 如果条件不满足，直接返回

    const State& s(getDrawingState());
    const bool oldOpacity = isOpaque(s);
    sp<GraphicBuffer> oldBuffer = getBE().compositionInfo.mBuffer;

    if (!allTransactionsSignaled()) {
        mFlinger->signalLayerUpdate();
        return outDirtyRegion;
    }
```
oldBuffer，前一帧的Buffer～
allTransactionsSignaled，确保所有的Fence都已经Signal出来。记住这个点，Fence相关的知识。

``` cpp
* frameworks/native/services/surfaceflinger/BufferLayer.cpp
    ... ...

    bool queuedBuffer = false;
    LayerRejecter r(mDrawingState, getCurrentState(), recomputeVisibleRegions,
                    getProducerStickyTransform() != 0, mName.string(),
                    mOverrideScalingMode, mFreezeGeometryUpdates);
    status_t updateResult =
            mConsumer->updateTexImage(&r, mFlinger->mPrimaryDispSync,
                                                    &mAutoRefresh, &queuedBuffer,
                                                    mLastFrameNumberReceived);
    if (updateResult == BufferQueue::PRESENT_LATER) {
        mFlinger->signalLayerUpdate();
        return outDirtyRegion;
    } else if (updateResult == BufferLayerConsumer::BUFFER_REJECTED) {
        if (queuedBuffer) {
            Mutex::Autolock lock(mQueueItemLock);
            mQueueItems.removeAt(0);
            android_atomic_dec(&mQueuedFrames);
        }
        return outDirtyRegion;
    } else if (updateResult != NO_ERROR || mUpdateTexImageFailed) {
        if (queuedBuffer) {
            Mutex::Autolock lock(mQueueItemLock);
            mQueueItems.clear();
            android_atomic_and(0, &mQueuedFrames);
        }

        mUpdateTexImageFailed = true;

        return outDirtyRegion;
    }
```
LayerRejecter顾名思义，用以决定是否拒绝这个Layer。updateTexImage 很关键，这里去获取的Buffer，将通过acquireBuffer函数去请求Buffer。前面我们已经说过BufferQueue的acquireBuffer流程。
updateTexImage有多种返回结果：
PRESENT_LATER：稍后显示，暂时不显示，触发SurfaceFlinger重新刷新signalLayerUpdate。
BUFFER_REJECTED： Buffer被Reject掉，这一帧数据将不再被显示，从mQueueItems中去掉这一帧的Buffer，mQueuedFrames也-1。
更新失败或出错：处理和BUFFER_REJECTED类似。
updateTexImage的流程稍后再看，我们将这个函数读完。

``` cpp
frameworks/native/services/surfaceflinger/BufferLayer.cpp
... ...
if (queuedBuffer) {
// Autolock scope
auto currentFrameNumber = mConsumer->getFrameNumber();
  Mutex::Autolock lock(mQueueItemLock);

  // 删掉updateTexImage中已经被丢弃的Buffer
  while (mQueueItems[0].mFrameNumber != currentFrameNumber) {
      mQueueItems.removeAt(0);
      android_atomic_dec(&mQueuedFrames);
  }

  mQueueItems.removeAt(0);

}
if ((queuedBuffer && android_atomic_dec(&mQueuedFrames) > 1) ||
mAutoRefresh) {
mFlinger->signalLayerUpdate();
}

```
如果获取Buffer后，队列中还有其他的Buffer，触发SurfaceFlinger去再做一次刷新signalLayerUpdate，在下一个Vsync再处理。

```cpp
* frameworks/native/services/surfaceflinger/BufferLayer.cpp
    ... ...
    
    // update the active buffer
    getBE().compositionInfo.mBuffer =
            mConsumer->getCurrentBuffer(&getBE().compositionInfo.mBufferSlot);
    // replicated in LayerBE until FE/BE is ready to be synchronized
    mActiveBuffer = getBE().compositionInfo.mBuffer;
    if (getBE().compositionInfo.mBuffer == NULL) {
        // this can only happen if the very first buffer was rejected.
        return outDirtyRegion;
    }
```
更新Active的Buffer，mActiveBuffer就是我们这次合成，该Layer的数据。如果没有获取到，返回。

``` cpp
frameworks/native/services/surfaceflinger/BufferLayer.cpp
... ...
mBufferLatched = true;
mPreviousFrameNumber = mCurrentFrameNumber;
mCurrentFrameNumber = mConsumer->getFrameNumber();
{
Mutex::Autolock lock(mFrameEventHistoryMutex);
mFrameEventHistory.addLatch(mCurrentFrameNumber, latchTime);
}

```
mFrameEventHistory，记录Frame的历史，Producer和Consumer对Frame的处理。
``` cpp
* frameworks/native/services/surfaceflinger/BufferLayer.cpp
    ... ...

    mRefreshPending = true;
    mFrameLatencyNeeded = true;
    if (oldBuffer == NULL) {
        // the first time we receive a buffer, we need to trigger a
        // geometry invalidation.
        recomputeVisibleRegions = true;
    }

    setDataSpace(mConsumer->getCurrentDataSpace());

    Rect crop(mConsumer->getCurrentCrop());
    const uint32_t transform(mConsumer->getCurrentTransform());
    const uint32_t scalingMode(mConsumer->getCurrentScalingMode());
    if ((crop != mCurrentCrop) ||
        (transform != mCurrentTransform) ||
        (scalingMode != mCurrentScalingMode)) {
        mCurrentCrop = crop;
        mCurrentTransform = transform;
        mCurrentScalingMode = scalingMode;
        recomputeVisibleRegions = true;
    }

    if (oldBuffer != NULL) {
        uint32_t bufWidth = getBE().compositionInfo.mBuffer->getWidth();
        uint32_t bufHeight = getBE().compositionInfo.mBuffer->getHeight();
        if (bufWidth != uint32_t(oldBuffer->width) ||
            bufHeight != uint32_t(oldBuffer->height)) {
            recomputeVisibleRegions = true;
        }
    }

    mCurrentOpacity = getOpacityForFormat(getBE().compositionInfo.mBuffer->format);
    if (oldOpacity != isOpaque(s)) {
        recomputeVisibleRegions = true;
    }
```
这里主要是根据新的Buffer的属性，和上一帧Buffer的的数据，做比较，看看是否需要重新去计算可见区域。

``` cpp
* frameworks/native/services/surfaceflinger/BufferLayer.cpp
    ... ...

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
                point = mLocalSyncPoints.erase(point);
            } else {
                ++point;
            }
        }
    }

    // FIXME: postedRegion should be dirty & bounds
    Region dirtyRegion(Rect(s.active.w, s.active.h));

    // transform the dirty region to window-manager space
    outDirtyRegion = (getTransform().transform(dirtyRegion));

    return outDirtyRegion;
}
```
最后，是对SyncPoint进行处理，新latch的buffer相关的Syncpoint都删掉。返回的是outDirtyRegion，对dirtyRegion做了transform变换后的区域大小。
我们再回过头看updateTexImage，updateTexImage函数实现如下：

``` cpp
status_t BufferLayerConsumer::updateTexImage(BufferRejecter* rejecter, const DispSync& dispSync,
                                             bool* autoRefresh, bool* queuedBuffer,
                                             uint64_t maxFrameNumber) {
    ... ...

    BufferItem item;

    status_t err = acquireBufferLocked(&item, computeExpectedPresent(dispSync), maxFrameNumber);
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

    if (!SyncFeatures::getInstance().useNativeFenceSync()) {
        err = bindTextureImageLocked();
    }

    return err;
}
```
updateTexImage过程大致如下：
1.拿到一块Buffer，从BufferQueue中，acquireBufferLocked

``` cpp
status_t BufferLayerConsumer::acquireBufferLocked(BufferItem* item, nsecs_t presentWhen,
                                                  uint64_t maxFrameNumber) {
    status_t err = ConsumerBase::acquireBufferLocked(item, presentWhen, maxFrameNumber);
    if (err != NO_ERROR) {
        return err;
    }

    // If item->mGraphicBuffer is not null, this buffer has not been acquired
    // before, so any prior EglImage created is using a stale buffer. This
    // replaces any old EglImage with a new one (using the new buffer).
    if (item->mGraphicBuffer != NULL) {
        mImages[item->mSlot] = new Image(item->mGraphicBuffer, mRE);
    }

    return NO_ERROR;
}
```
acquireBufferLocked通过父类，ConsumerBase的acquireBufferLocked函数去获取Buffer，如果Buffer不为空，创建 Eglimage。
在ConsumerBase的acquireBufferLocked中，正式通过BufferQueue的BufferQueueConsumer去acquireBuffer。代码如下：

``` cpp
status_t ConsumerBase::acquireBufferLocked(BufferItem *item,
        nsecs_t presentWhen, uint64_t maxFrameNumber) {
    if (mAbandoned) {
        CB_LOGE("acquireBufferLocked: ConsumerBase is abandoned!");
        return NO_INIT;
    }

    status_t err = mConsumer->acquireBuffer(item, presentWhen, maxFrameNumber);
    if (err != NO_ERROR) {
        return err;
    }

    if (item->mGraphicBuffer != NULL) {
        if (mSlots[item->mSlot].mGraphicBuffer != NULL) {
            freeBufferLocked(item->mSlot);
        }
        mSlots[item->mSlot].mGraphicBuffer = item->mGraphicBuffer;
    }

    mSlots[item->mSlot].mFrameNumber = item->mFrameNumber;
    mSlots[item->mSlot].mFence = item->mFence;

    CB_LOGV("acquireBufferLocked: -> slot=%d/%" PRIu64,
            item->mSlot, item->mFrameNumber);

    return OK;
}
```

acquireBuffer的流程前面已经说过，拿到Buffer后，将Buffer保存在mSlots[item->mSlot].mGraphicBuffer中。同时更新mFrameNumber和mFence。
2.检测Buffer可用不，不可用就Reject掉，rejecter->reject
相关的逻辑在类LayerRejecter中：

* frameworks/native/services/surfaceflinger/LayerRejecter.cpp

代码这里就不贴了，在reject逻辑中，其一是判断释放需要重新计算可见区域mRecomputeVisibleRegions；其二，看看Buffer的属性和状态描述中的属性释放吻合，不一直就reject掉；

3.更新Buffer，释放上一个Buffer，updateAndReleaseLocked

updateAndReleaseLocked函数实现如下：

``` cpp
status_t BufferLayerConsumer::updateAndReleaseLocked(const BufferItem& item,
                                                     PendingRelease* pendingRelease) {
    status_t err = NO_ERROR;

    int slot = item.mSlot;

    // Do whatever sync ops we need to do before releasing the old slot.
    if (slot != mCurrentTexture) {
        err = syncForReleaseLocked();
        if (err != NO_ERROR) {
            // Release the buffer we just acquired.  It's not safe to
            // release the old buffer, so instead we just drop the new frame.
            // As we are still under lock since acquireBuffer, it is safe to
            // release by slot.
            releaseBufferLocked(slot, mSlots[slot].mGraphicBuffer);
            return err;
        }
    }

    BLC_LOGV("updateAndRelease: (slot=%d buf=%p) -> (slot=%d buf=%p)", mCurrentTexture,
             mCurrentTextureImage != NULL ? mCurrentTextureImage->graphicBufferHandle() : 0, slot,
             mSlots[slot].mGraphicBuffer->handle);

    // Hang onto the pointer so that it isn't freed in the call to
    // releaseBufferLocked() if we're in shared buffer mode and both buffers are
    // the same.
    sp<Image> nextTextureImage = mImages[slot];

    // release old buffer
    if (mCurrentTexture != BufferQueue::INVALID_BUFFER_SLOT) {
        if (pendingRelease == nullptr) {
            status_t status =
                    releaseBufferLocked(mCurrentTexture, mCurrentTextureImage->graphicBuffer());
            if (status < NO_ERROR) {
                BLC_LOGE("updateAndRelease: failed to release buffer: %s (%d)", strerror(-status),
                         status);
                err = status;
                // keep going, with error raised [?]
            }
        } else {
            pendingRelease->currentTexture = mCurrentTexture;
            pendingRelease->graphicBuffer = mCurrentTextureImage->graphicBuffer();
            pendingRelease->isPending = true;
        }
    }

    // Update the BufferLayerConsumer state.
    mCurrentTexture = slot;
    mCurrentTextureImage = nextTextureImage;
    mCurrentCrop = item.mCrop;
    mCurrentTransform = item.mTransform;
    mCurrentScalingMode = item.mScalingMode;
    mCurrentTimestamp = item.mTimestamp;
    mCurrentDataSpace = item.mDataSpace;
    mCurrentHdrMetadata = item.mHdrMetadata;
    mCurrentFence = item.mFence;
    mCurrentFenceTime = item.mFenceTime;
    mCurrentFrameNumber = item.mFrameNumber;
    mCurrentTransformToDisplayInverse = item.mTransformToDisplayInverse;
    mCurrentSurfaceDamage = item.mSurfaceDamage;

    computeCurrentTransformMatrixLocked();

    return err;
}
```
该函数中，处理Fence相关的逻辑比较多，后续我们将用专门的章节来讲述Android中Fence同步机制，这里先不要太关注它。该函数中主要作用如下：

- syncForReleaseLocked，mCurrentTexture是上一个Buffer的序号slot，我们需要给旧Buffer设置ReleaseFence。
- releaseBufferLocked，release掉旧的Buffer，先加到mPendingRelease中，待合成完成后release掉Pending的Buffer。
- 更新BufferLayerConsumer的状态，Buffer的属性都保存到mCurrent**定义的属性中。

到此updateTexImage函数完成。
latchBuffer中，再通过getCurrentBuffer去获取Consumer中已经更了Buffer。代码如下：

``` cpp
sp<GraphicBuffer> BufferLayerConsumer::getCurrentBuffer(int* outSlot) const {
    Mutex::Autolock lock(mMutex);

    if (outSlot != nullptr) {
        *outSlot = mCurrentTexture;
    }

    return (mCurrentTextureImage == nullptr) ? NULL : mCurrentTextureImage->graphicBuffer();
}
```
mCurrentTextureImage，也是按照slot从mImages中获取的，前面acquireBuffer时，Buffer根据slot保存在mImages中。
到此，Layer中已经获取到Buffer的数据。需要注意的是，这不是对单个的Layer，而是所有的mLayersWithQueuedFrames都会走上面的流程，而每个Layer有自己的BufferLayerConsumer和BufferQueue。
我们先来看看看这里遇到的几个类间的相互关系：

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/display.system/Android.PG4.Layer.Buffer.png)
还算比较清晰吧～这里我们只关心Buffer从哪儿来，到哪儿去就行了

拿到Buffer后，更新Layer的Damage，useSurfaceDamage，Damage表示Layer的那些区域被破坏了，被破坏的区域需要重新合成显示。

``` cpp
const Region& BufferLayerConsumer::getSurfaceDamage() const {
    return mCurrentSurfaceDamage;
}
```
surfaceDamage就前面updateTexture时一起更新的mCurrentSurfaceDamage。

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
            if (refreshNeeded) {
                signalRefresh();
            }
```
什么时候需要刷新？

- 有新的Transaction处理
- PageFlip时，有Buffer更新！～
- 有重新合成请求时mRepaintEverything，这是响应HWC的请求时触发的。
刷新消息REFRESH处理
SurfaceFlinger收到了VSync信号后，调用了handleMessageRefresh函数
[SurfaceFlinger.cpp]

``` cpp
void SurfaceFlinger::handleMessageRefresh() {
    ATRACE_CALL();

    mRefreshPending = false;

    nsecs_t refreshStartTime = systemTime(SYSTEM_TIME_MONOTONIC);

    preComposition(refreshStartTime);
    rebuildLayerStacks();
    setUpHWComposer();
    doDebugFlashRegions();
    doTracing("handleRefresh");
    doComposition();
    postComposition(refreshStartTime);

    mPreviousPresentFence = getBE().mHwc->getPresentFence(HWC_DISPLAY_PRIMARY);

    mHadClientComposition = false;
    for (size_t displayId = 0; displayId < mDisplays.size(); ++displayId) {
        const sp<DisplayDevice>& displayDevice = mDisplays[displayId];
        mHadClientComposition = mHadClientComposition ||
                getBE().mHwc->hasClientComposition(displayDevice->getHwcDisplayId());
    }

    mLayersWithQueuedFrames.clear();
}
```
handleMessageRefresh 函数中，包含了刷新一帧显示数据所有的流程。下面我们分别来进行说明。

##### 一、preComposition()函数

``` cpp
void SurfaceFlinger::preComposition(nsecs_t refreshStartTime)
{
    ATRACE_CALL();
    ALOGV("preComposition");

    bool needExtraInvalidate = false;
    mDrawingState.traverseInZOrder([&](Layer* layer) {
        if (layer->onPreComposition(refreshStartTime)) {
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
    if (mBufferLatched) {
        Mutex::Autolock lock(mFrameEventHistoryMutex);
        mFrameEventHistory.addPreComposition(mCurrentFrameNumber,
                                             refreshStartTime);
    }
    mRefreshPending = false;
    return mQueuedFrames > 0 || mSidebandStreamChanged ||
            mAutoRefresh;
}
```

onPreComposition中主要作用为：

- mFrameEventHistory记录PreComposition事件
- 判断是否需要再触发SurfaceFlinger继续接受Vsync进行合成
这3中情况需要：如果mQueuedFrames的值大于0，说明这个时候BufferQueue中还有Buffer，之前我们在acquireBuffer的时候，已经做了-1操纵；SidebandStream改变；或者是自动刷新模式。

如果需要再触发SurfaceFlinger工作，调signalLayerUpdate函数。
重构Layer的Stack rebuildLayerStacks
现在，我们需要合成显示的Layer数据，都保存在mDrawingState的layersSortedByZ中，且是按照z-order的顺序进行存放。那么rebuild Layer又是做什么呢？

``` cpp
void SurfaceFlinger::rebuildLayerStacks() {
    ATRACE_CALL();
    ALOGV("rebuildLayerStacks");

    // rebuild the visible layer list per screen
    if (CC_UNLIKELY(mVisibleRegionsDirty)) {
        ATRACE_NAME("rebuildLayerStacks VR Dirty");
        mVisibleRegionsDirty = false;
        invalidateHwcGeometry();

        for (size_t dpy=0 ; dpy<mDisplays.size() ; dpy++) {
            Region opaqueRegion;
            Region dirtyRegion;
            Vector<sp<Layer>> layersSortedByZ;
            Vector<sp<Layer>> layersNeedingFences;
            const sp<DisplayDevice>& displayDevice(mDisplays[dpy]);
            const Transform& tr(displayDevice->getTransform());
            const Rect bounds(displayDevice->getBounds());
            if (displayDevice->isDisplayOn()) {
                computeVisibleRegions(displayDevice, dirtyRegion, opaqueRegion);

                ... ...
            }
            displayDevice->setVisibleLayersSortedByZ(layersSortedByZ);
            displayDevice->setLayersNeedingFences(layersNeedingFences);
            displayDevice->undefinedRegion.set(bounds);
            displayDevice->undefinedRegion.subtractSelf(
                    tr.transform(opaqueRegion));
            displayDevice->dirtyRegion.orSelf(dirtyRegion);
        }
    }
}
```
rebuild Layer的前提是存在脏区域，mVisibleRegionsDirty为true。invalidateHwcGeometry重置mGeometryInvalid标记，这个标识后面会用到。
Android支持多个屏幕，每个屏幕的显示数据并不是完全一样的，每个Display是分开合成的；也就是说，layersSortedByZ中的layer需要根据显示屏的特性，分别进行合成，合成后的数据，送给各自的显示屏。
mDisplays是当前系统中的显示屏，isDisplayOn判断屏幕是否是打开的。主屏幕是默认支持的，处于打开状态。
computeVisibleRegions，计算可见区域。Layer中有很多个区域，不太好理解。computeVisibleRegions函数实现如下：

``` cpp
void SurfaceFlinger::computeVisibleRegions(const sp<const DisplayDevice>& displayDevice,
        Region& outDirtyRegion, Region& outOpaqueRegion)
{
    ATRACE_CALL();
    ALOGV("computeVisibleRegions");

    Region aboveOpaqueLayers;
    Region aboveCoveredLayers;
    Region dirty;

    outDirtyRegion.clear();

    mDrawingState.traverseInReverseZOrder([&](Layer* layer) {
        // start with the whole surface at its current location
        const Layer::State& s(layer->getDrawingState());

        // only consider the layers on the given layer stack
        if (!layer->belongsToDisplay(displayDevice->getLayerStack(), displayDevice->isPrimary()))
            return;

        Region opaqueRegion;

        Region visibleRegion;

        Region coveredRegion;

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
void SurfaceFlinger::computeVisibleRegions(const sp<const DisplayDevice>& displayDevice,
        Region& outDirtyRegion, Region& outOpaqueRegion)
{
        ... ...
        // handle hidden surfaces by setting the visible region to empty
        if (CC_LIKELY(layer->isVisible())) {
            const bool translucent = !layer->isOpaque(s);
            Rect bounds(layer->computeScreenBounds());
            visibleRegion.set(bounds);
            Transform tr = layer->getTransform();
            if (!visibleRegion.isEmpty()) {
                // 将完全透明区域从可见区域中删掉
                if (translucent) {
                    if (tr.preserveRects()) {
                        // transform 透明区域
                        transparentRegion = tr.transform(s.activeTransparentRegion);
                    } else {
                        // 太复杂了，做不了优化
                        transparentRegion.clear();
                    }
                }

                // compute the opaque region
                const int32_t layerOrientation = tr.getOrientation();
                if (layer->getAlpha() == 1.0f && !translucent &&
                        ((layerOrientation & Transform::ROT_INVALID) == false)) {
                    // the opaque region is the layer's footprint
                    opaqueRegion = visibleRegion;
                }
            }
        }
```

这里用到Layer的几个函数：

- isOpaque 说明Layer是非透明的Layer，这个是上层应用设置的，注意，我们这里说的应用不只说App，也包括Android的Framework，是泛指。
- computeScreenBounds 计算Layer的在屏幕上的大小
computeScreenBounds函数如下：

``` cpp
Rect Layer::computeScreenBounds(bool reduceTransparentRegion) const {
    const Layer::State& s(getDrawingState());
    Rect win(s.active.w, s.active.h);

    if (!s.crop.isEmpty()) {
        win.intersect(s.crop, &win);
    }

    Transform t = getTransform();
    win = t.transform(win);

    if (!s.finalCrop.isEmpty()) {
        win.intersect(s.finalCrop, &win);
    }

    const sp<Layer>& p = mDrawingParent.promote();
    if (p != nullptr) {
        Rect bounds = p->computeScreenBounds(false);
        bounds.intersect(win, &win);
    }

    if (reduceTransparentRegion) {
        auto const screenTransparentRegion = t.transform(s.activeTransparentRegion);
        win = reduce(win, screenTransparentRegion);
    }

    return win;
}
```
s.active.w和s.active.h，是Layer本身的大小，用win表示。
crop是Layer的源剪截区域，由上层设置，表示该Layer只截取crop的区域进行合成显示，这个区域可以能比win大，也可能比win小，所以要和win做一个交集运算，截取两个区域重复的部分。
finalCrop和crop类似，只是这里的finalCrop是处理win做了变换后的，最终的区域。finalCrop也是上层设置的。
Layer本身的crop处理完后，还要和父Layer的区域做一个交集运算，子Layer不让超过父Layer的大小？
默认的需要减掉透明区域的，reduceTransparentRegion默认参数为true。
computeScreenBounds的返回值，就是Layer可见区域的大小，visibleRegion区域后续还会被裁剪。

- getTransform 获取 Layer的变换矩阵
屏幕有旋转，需要做变换，去适配显示屏幕

回到 computeVisibleRegions函数，计算完可见区域，计算非透明区域，一般情况下，如果layer是非透明的，非透明区域就是可见区域。

``` cpp
void SurfaceFlinger::computeVisibleRegions(const sp<const DisplayDevice>& displayDevice,
        Region& outDirtyRegion, Region& outOpaqueRegion)
{
        ... ...
        // 遍历时，第一层时，aboveCoveredLayers为空，coveredRegion也是为空，最上面一层是没有被覆盖的，当然为空。
        coveredRegion = aboveCoveredLayers.intersect(visibleRegion);

        // 更新aboveCoveredLayers，该层之下的Layer都被该层Layer覆盖，所以这里和可见区域做一个或操纵，最下面的区域被覆盖的越大
        aboveCoveredLayers.orSelf(visibleRegion);

        // 可见区域要减掉该层之上的非透明区域。
        visibleRegion.subtractSelf(aboveOpaqueLayers);
```
上面部分的逻辑，都注释在代码中。继续看～

下面是计算Layer的脏区域：

``` cpp
void SurfaceFlinger::computeVisibleRegions(const sp<const DisplayDevice>& displayDevice,
        Region& outDirtyRegion, Region& outOpaqueRegion)
{
        ... ...
        // compute this layer's dirty region
        if (layer->contentDirty) {
            // we need to invalidate the whole region
            dirty = visibleRegion;
            // as well, as the old visible region
            dirty.orSelf(layer->visibleRegion);
            layer->contentDirty = false;
        } else {
            const Region newExposed = visibleRegion - coveredRegion;
            const Region oldVisibleRegion = layer->visibleRegion;
            const Region oldCoveredRegion = layer->coveredRegion;
            const Region oldExposed = oldVisibleRegion - oldCoveredRegion;
            dirty = (visibleRegion&oldCoveredRegion) | (newExposed-oldExposed);
        }
        dirty.subtractSelf(aboveOpaqueLayers);
```
contentDirty表示Layer的可见区域被修改了，这个是需要和layer的visibleRegion做一个与运算。确保可见的区域都能被刷新到。如果contentDirty没有被修改，开始计算暴露出来的区域 exposedRegion。exposedRegion包含两部分，之前被覆盖的区域，现在暴露了，直接暴露的区域，现在也是暴露的区域。dirty的区域，就是暴露的区域，再除去上面非透明的区域。

``` cpp
void SurfaceFlinger::computeVisibleRegions(const sp<const DisplayDevice>& displayDevice,
        Region& outDirtyRegion, Region& outOpaqueRegion)
{
        ... ...
        // accumulate to the screen dirty region
        outDirtyRegion.orSelf(dirty);

        // 更新之上非透明的区域，下面的Layer计算时会用到
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
outDirtyRegion是屏幕的脏区域，它是每个Layer脏区域的合。最后将计算好的区域值设置到Layer中。outOpaqueRegion是屏幕的非透明区域。

- setVisibleRegion 设置可见区域
- setCoveredRegion 设置被覆盖的区域
- setVisibleNonTransparentRegion 设置可见的非透明区域，它是可见区域，减去透明区域。

回到rebuildLayerStacks函数～ computeVisibleRegions结束后，屏幕的脏区域得到了，每个Layer的可见区域，被覆盖的区域，以及可见非透明区域都计算出来了。


#### 二、rebuildLayerStacks函数


``` cpp
void SurfaceFlinger::rebuildLayerStacks() {
    if (CC_UNLIKELY(mVisibleRegionsDirty)) {
        ...

        for (size_t dpy=0 ; dpy<mDisplays.size() ; dpy++) {
            ... ...
            if (displayDevice->isDisplayOn()) {
                computeVisibleRegions(displayDevice, dirtyRegion, opaqueRegion);

                mDrawingState.traverseInZOrder([&](Layer* layer) {
                    bool hwcLayerDestroyed = false;
                    if (layer->belongsToDisplay(displayDevice->getLayerStack(),
                                displayDevice->isPrimary())) {
                        Region drawRegion(tr.transform(
                                layer->visibleNonTransparentRegion));
                        drawRegion.andSelf(bounds);
                        if (!drawRegion.isEmpty()) {
                            layersSortedByZ.add(layer);
                        } else {
                            hwcLayerDestroyed = layer->destroyHwcLayer(
                                    displayDevice->getHwcDisplayId());
                        }
                    } else {
                        hwcLayerDestroyed = layer->destroyHwcLayer(
                                displayDevice->getHwcDisplayId());
                    }

                    if (hwcLayerDestroyed) {
                        auto found = std::find(mLayersWithQueuedFrames.cbegin(),
                                mLayersWithQueuedFrames.cend(), layer);
                        if (found != mLayersWithQueuedFrames.cend()) {
                            layersNeedingFences.add(layer);
                        }
                    }
                });
            }
            ... ...
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

        for (size_t dpy=0 ; dpy<mDisplays.size() ; dpy++) {
            ... ...
            displayDevice->setVisibleLayersSortedByZ(layersSortedByZ);
            displayDevice->setLayersNeedingFences(layersNeedingFences);
            displayDevice->undefinedRegion.set(bounds);
            displayDevice->undefinedRegion.subtractSelf(
                    tr.transform(opaqueRegion));
            displayDevice->dirtyRegion.orSelf(dirtyRegion);
        }
    }
}
```
Display中还有一个区域，叫未定义的区域。也就是屏幕的大小减去屏幕的非透明区域opaqueRegion余下的部分。

创建Layer栈完成，此时需要进行合成显示的数据已经被更新到每个Display各自的layersSortedByZ中。

##### 三、配置硬件合成 setUpHWComposer
回到handleMessageRefresh，继续看Refresh消息的处理。此时需要进行合成显示的数据，在rebuildLayerStacks时，已经被更新到每个Display各自的layersSortedByZ中。Layer栈创建完成后，进行HWC 合成的设置。
setUpHWComposer的代码比较长，我们分段看，在setUpHWComposer中，主要做了以下几件事：
##### 1.DisplayDevice beginFrame


[SurfaceFlinger.cpp]

``` cpp
void SurfaceFlinger::setUpHWComposer() {
    ... ...

    for (size_t dpy=0 ; dpy<mDisplays.size() ; dpy++) {
        bool dirty = !mDisplays[dpy]->getDirtyRegion(false).isEmpty();
        bool empty = mDisplays[dpy]->getVisibleLayersSortedByZ().size() == 0;
        bool wasEmpty = !mDisplays[dpy]->lastCompositionHadVisibleLayers;

        //  判断是否需要重新合成
        bool mustRecompose = dirty && !(empty && wasEmpty);

        ... ...

        mDisplays[dpy]->beginFrame(mustRecompose);

        if (mustRecompose) {
            mDisplays[dpy]->lastCompositionHadVisibleLayers = !empty;
        }
    }
```

Android对每一块显示屏的处理都是分开的。这里主要是调Display的beginFrame函数。

``` cpp
status_t DisplayDevice::beginFrame(bool mustRecompose) const {
    return mDisplaySurface->beginFrame(mustRecompose);
}
```
mDisplaySurface根据屏幕有所不同。
主显和外显用的FramebufferSurface，需显示用的VirtualDisplaySurface，我们这里先不关虚显。
status_t FramebufferSurface::beginFrame(bool /*mustRecompose*/) {
    return NO_ERROR;
}

FramebufferSurface在beginFrame是每一做什么过多的处理。
回到setUpHWComposer函数

##### 2.创建工作列表

``` cpp
void SurfaceFlinger::setUpHWComposer() {
     ... ...

    // build the h/w work list
    if (CC_UNLIKELY(mGeometryInvalid)) {
        mGeometryInvalid = false;
        for (size_t dpy=0 ; dpy<mDisplays.size() ; dpy++) {
            sp<const DisplayDevice> displayDevice(mDisplays[dpy]);
            const auto hwcId = displayDevice->getHwcDisplayId();
            if (hwcId >= 0) {
                const Vector<sp<Layer>>& currentLayers(
                        displayDevice->getVisibleLayersSortedByZ());
                for (size_t i = 0; i < currentLayers.size(); i++) {
                    const auto& layer = currentLayers[i];
                    if (!layer->hasHwcLayer(hwcId)) {
                        if (!layer->createHwcLayer(getBE().mHwc.get(), hwcId)) {
                            layer->forceClientComposition(hwcId);
                            continue;
                        }
                    }

                    layer->setGeometry(displayDevice, i);
                    if (mDebugDisableHWC || mDebugRegion) {
                        layer->forceClientComposition(hwcId);
                    }
                }
            }
        }
    }
```
对每个Display中的每个Layer创建对应的HWC Layer，注意hwcId，只是对hwcId大于零的Layer才会创建HWC Layer。Layer的createHwcLayer函数如下：

``` cpp
bool Layer::createHwcLayer(HWComposer* hwc, int32_t hwcId) {
    LOG_ALWAYS_FATAL_IF(getBE().mHwcLayers.count(hwcId) != 0,
                        "Already have a layer for hwcId %d", hwcId);
    HWC2::Layer* layer = hwc->createLayer(hwcId);
    if (!layer) {
        return false;
    }
    LayerBE::HWCInfo& hwcInfo = getBE().mHwcLayers[hwcId];
    hwcInfo.hwc = hwc;
    hwcInfo.layer = layer;
    layer->setLayerDestroyedListener(
            [this, hwcId](HWC2::Layer* /*layer*/) { getBE().mHwcLayers.erase(hwcId); });
    return true;
}
```
根据hwcId来创建，也就是说，HWC上层（SurfaceFlinger）的每个Layer，都会为每个Display创建一个HWC Layer。HWComposer 根据hwcId找对HWC2的Display，再通过具体的HWC2::Dispaly去创建自己的HWC Layer；
Layer通过HWComposer来创建，

``` cpp
HWC2::Layer* HWComposer::createLayer(int32_t displayId) {
    if (!isValidDisplay(displayId)) {
        ALOGE("Failed to create layer on invalid display %d", displayId);
        return nullptr;
    }
    auto display = mDisplayData[displayId].hwcDisplay;
    HWC2::Layer* layer;
    auto error = display->createLayer(&layer);
    if (error != HWC2::Error::None) {
        ALOGE("Failed to create layer on display %d: %s (%d)", displayId,
                to_string(error).c_str(), static_cast<int32_t>(error));
        return nullptr;
    }
    return layer;
}
```
创建HWC Layer的实现，最终是在Vendor实现的HAL中来完成的。对应的HWC2command为HWC2_FUNCTION_CREATE_LAYER。

``` cpp
Error HwcHal::createLayer(Display display, Layer* outLayer)
{
    int32_t err = mDispatch.createLayer(mDevice, display, outLayer);
    return static_cast<Error>(err);
}
```
创建的HWC Layer在HWC2::Dispaly中也保存了一个引用：

``` cpp
* frameworks/native/services/surfaceflinger/DisplayHardware/HWC2.cpp

Error Display::createLayer(Layer** outLayer)
{
    if (!outLayer) {
        return Error::BadParameter;
    }
    hwc2_layer_t layerId = 0;
    auto intError = mComposer.createLayer(mId, &layerId);
    auto error = static_cast<Error>(intError);
    if (error != Error::None) {
        return error;
    }

    auto layer = std::make_unique<Layer>(
            mComposer, mCapabilities, mId, layerId);
    *outLayer = layer.get();
    mLayers.emplace(layerId, std::move(layer));
    return Error::None;
}
```
如果hwclayer没有创建成功，那么这一层Layer就强制用Client方式合成forceClientComposition。

``` cpp
void Layer::forceClientComposition(int32_t hwcId) {
    if (getBE().mHwcLayers.count(hwcId) == 0) {
        ALOGE("forceClientComposition: no HWC layer found (%d)", hwcId);
        return;
    }

    getBE().mHwcLayers[hwcId].forceClientComposition = true;
}
```

另外，如果是Disable掉HWC合成，或者调试Region，也强制用Client方式合成：
mDebugDisableHWC || mDebugRegion

这两个调试方式可以在系统设置，开发者选项中进行设置。
创建完hwcLayer后，设置Layer的几何尺寸：

``` cpp
void Layer::setGeometry(const sp<const DisplayDevice>& displayDevice, uint32_t z)
{
     ... ... //注意，我们这里的数据都是来源于DrawingState
    const State& s(getDrawingState());
    ... ...
    if (!isOpaque(s) || getAlpha() != 1.0f) {
        blendMode =
                mPremultipliedAlpha ? HWC2::BlendMode::Premultiplied : HWC2::BlendMode::Coverage;
    }
    auto error = hwcLayer->setBlendMode(blendMode);

    // 计算displayFrame
    Rect frame{t.transform(computeBounds(activeTransparentRegion))};
    ... ...
    const Transform& tr(displayDevice->getTransform());
    Rect transformedFrame = tr.transform(frame);
    error = hwcLayer->setDisplayFrame(transformedFrame);
    ... ...

    // 计算sourceCrop
    FloatRect sourceCrop = computeCrop(displayDevice);
    error = hwcLayer->setSourceCrop(sourceCrop);
    ... ...

    // 设置Alpha
    float alpha = static_cast<float>(getAlpha());
    error = hwcLayer->setPlaneAlpha(alpha);
    ... ...

    // 设置z-order
    error = hwcLayer->setZOrder(z);
    ... ...

    int type = s.type;
    int appId = s.appId;
    sp<Layer> parent = mDrawingParent.promote();
    if (parent.get()) {
        auto& parentState = parent->getDrawingState();
        type = parentState.type;
        appId = parentState.appId;
    }

    // 设置Layer的信息
    error = hwcLayer->setInfo(type, appId);
    ALOGE_IF(error != HWC2::Error::None, "[%s] Failed to set info (%d)", mName.string(),
             static_cast<int32_t>(error));

    // 设置transform
    const uint32_t orientation = transform.getOrientation();
    if (orientation & Transform::ROT_INVALID) {
        // we can only handle simple transformation
        hwcInfo.forceClientComposition = true;
    } else {
        auto transform = static_cast<HWC2::Transform>(orientation);
        auto error = hwcLayer->setTransform(transform);
        ... ...
    }
}
```
注意，我们这里的数据都是来源于DrawingState，setGeometry函数中，主要做了以下几件事：

- 确认HWCLayer的混合模式
混合模式，是两个Layer直接的混合方式，主要下面的几种：

``` cpp
/* Blend modes, settable per layer */
typedef enum {
    HWC2_BLEND_MODE_INVALID = 0,

    /* colorOut = colorSrc */
    HWC2_BLEND_MODE_NONE = 1,

    /* colorOut = colorSrc + colorDst * (1 - alphaSrc) */
    HWC2_BLEND_MODE_PREMULTIPLIED = 2,

    /* colorOut = colorSrc * alphaSrc + colorDst * (1 - alphaSrc) */
    HWC2_BLEND_MODE_COVERAGE = 3,
} hwc2_blend_mode_t;
```
HWC2_BLEND_MODE_NONE，不混合，源是什么样，输出就是什么样的。
HWC2_BLEND_MODE_PREMULTIPLIED，预乘，Dst需要做Alpha的处理。
HWC2_BLEND_MODE_COVERAGE，覆盖方式，源和Dst都需要做Alpha的处理。

- 计算displayFrame并设置给hwcLayer
displayFrame通过transform转换过的
- 计算sourceCrop并设置给hwcLayer
sourceCrop是上层传下来的，再和Dispaly，其他Layer的属性进行计算。
- 设置Alpha值
- 设置z-Order
- 设置Layer的信息
type和appId是Android Framework层创建SurfaceControl时设置的，可以搜搜"new SurfaceControl"就出来了。type是类型，比如ScreenshotSurface，Background等；appId是应用的进程号。
- 设置变换信息transform

Layer信息的设置，是通过CommandBuffer的读写来完成的，比如，设置混合模式，最终是调的HwcHal的setLayerBlendMode方法。

``` cpp
Error HwcHal::setLayerBlendMode(Display display, Layer layer, int32_t mode)
{
    int32_t err = mDispatch.setLayerBlendMode(mDevice, display, layer, mode);
    return static_cast<Error>(err);
}
```
回到setUpHWComposer函数

##### 3.设置每层Layer的Frame数据

``` cpp
void SurfaceFlinger::setUpHWComposer() {
     ... ...
    mat4 colorMatrix = mColorMatrix * computeSaturationMatrix() * mDaltonizer();

    // Set the per-frame data
    for (size_t displayId = 0; displayId < mDisplays.size(); ++displayId) {
        auto& displayDevice = mDisplays[displayId];
        const auto hwcId = displayDevice->getHwcDisplayId();

        ... ... // 设置每个Dispaly的颜色矩阵
        if (colorMatrix != mPreviousColorMatrix) {
            status_t result = getBE().mHwc->setColorTransform(hwcId, colorMatrix);
            ... ...
        }
        for (auto& layer : displayDevice->getVisibleLayersSortedByZ()) {
            if (layer->getForceClientComposition(hwcId)) {
                ALOGV("[%s] Requesting Client composition", layer->getName().string());
                layer->setCompositionType(hwcId, HWC2::Composition::Client);
                continue;
            }

            layer->setPerFrameData(displayDevice);
        }

        if (hasWideColorDisplay) {
            android_color_mode newColorMode;
            android_dataspace newDataSpace = HAL_DATASPACE_V0_SRGB;

            for (auto& layer : displayDevice->getVisibleLayersSortedByZ()) {
                newDataSpace = bestTargetDataSpace(layer->getDataSpace(), newDataSpace);
                ALOGV("layer: %s, dataspace: %s (%#x), newDataSpace: %s (%#x)",
                      layer->getName().string(), dataspaceDetails(layer->getDataSpace()).c_str(),
                      layer->getDataSpace(), dataspaceDetails(newDataSpace).c_str(), newDataSpace);
            }
            newColorMode = pickColorMode(newDataSpace);

            setActiveColorModeInternal(displayDevice, newColorMode);
        }
    }

    mPreviousColorMatrix = colorMatrix;
```
设置Frame数据时，主要做了以下几件事：

- 设置每个Dispaly的颜色矩阵
可以在开发这选项中设置，模拟颜色空间。其支持的transform主要有：

``` cpp
typedef enum {
    HAL_COLOR_TRANSFORM_IDENTITY = 0,
    HAL_COLOR_TRANSFORM_ARBITRARY_MATRIX = 1,
    HAL_COLOR_TRANSFORM_VALUE_INVERSE = 2,
    HAL_COLOR_TRANSFORM_GRAYSCALE = 3,
    HAL_COLOR_TRANSFORM_CORRECT_PROTANOPIA = 4,
    HAL_COLOR_TRANSFORM_CORRECT_DEUTERANOPIA = 5,
    HAL_COLOR_TRANSFORM_CORRECT_TRITANOPIA = 6,
} android_color_transform_t;
```

colorTransform主要是用以做颜色变换，以模拟或帮助色盲患者等。

- 设置每一层Layer的显示数据
setPerFrameData BufferLayer和ColorLayer的实现不一样。ColorLayer 的逻辑比较简单：

``` cpp
void ColorLayer::setPerFrameData(const sp<const DisplayDevice>& displayDevice) {
    ... ...
    // 设置可见区域
    auto error = hwcLayer->setVisibleRegion(visible);

    // 制定合成方式
    setCompositionType(hwcId, HWC2::Composition::SolidColor);

    half4 color = getColor();
    // 设置颜色
    error = hwcLayer->setColor({static_cast<uint8_t>(std::round(255.0f * color.r)),
                                static_cast<uint8_t>(std::round(255.0f * color.g)),
                                static_cast<uint8_t>(std::round(255.0f * color.b)), 255});
    ... ...// 去掉变换矩阵
    error = hwcLayer->setTransform(HWC2::Transform::None);
    ... ...
}
```
ColorLayer，主要有4个操纵：设置可见区域，这个前面已经就算好了，但是这里要确保它在Dispaly的视窗里；指定合成方式，默认采用SolidColor方式合成；设置颜色，指定该Layer的颜色，RGBA的格式，Alpha默认为255，全透；最后，ColorLayer不需要transform，去掉。
BufferLayer的setPerFrameData处理如下：

``` cpp
void BufferLayer::setPerFrameData(const sp<const DisplayDevice>& displayDevice) {
    // 设置可见区域
    auto& hwcLayer = hwcInfo.layer;
    auto error = hwcLayer->setVisibleRegion(visible);

    ... ... //设置Damage区域
    error = hwcLayer->setSurfaceDamage(surfaceDamageRegion);
    ... ...

    // Sideband layers处理，默认合成类型为Sideband
    if (getBE().compositionInfo.hwc.sidebandStream.get()) {
        setCompositionType(hwcId, HWC2::Composition::Sideband);
        // 制定Sideband流
        error = hwcLayer->setSidebandStream(getBE().compositionInfo.hwc.sidebandStream->handle());
        ... ...
        return; // Sideband layers处理完成后直接返回了。
    }

    // 如果是Cursor Layer，合成类似为Cursor，其他为Device
    if (mPotentialCursor) {
        ALOGV("[%s] Requesting Cursor composition", mName.string());
        setCompositionType(hwcId, HWC2::Composition::Cursor);
    } else {
        ALOGV("[%s] Requesting Device composition", mName.string());
        setCompositionType(hwcId, HWC2::Composition::Device);
    }

    // 设置数据空间dataspace
    error = hwcLayer->setDataspace(mCurrentState.dataSpace);
    if (error != HWC2::Error::None) {
        ALOGE("[%s] Failed to set dataspace %d: %s (%d)", mName.string(), mCurrentState.dataSpace,
              to_string(error).c_str(), static_cast<int32_t>(error));
    }

    // 获取GraphicBuffer
    uint32_t hwcSlot = 0;
    sp<GraphicBuffer> hwcBuffer;
    hwcInfo.bufferCache.getHwcBuffer(getBE().compositionInfo.mBufferSlot,
                                     getBE().compositionInfo.mBuffer, &hwcSlot, &hwcBuffer);

    //获取Fence
    auto acquireFence = mConsumer->getCurrentFence();
    // 设置Buffer
    error = hwcLayer->setBuffer(hwcSlot, hwcBuffer, acquireFence);
    if (error != HWC2::Error::None) {
        ALOGE("[%s] Failed to set buffer %p: %s (%d)", mName.string(),
              getBE().compositionInfo.mBuffer->handle, to_string(error).c_str(),
              static_cast<int32_t>(error));
    }
}
```
BufferLayer的处理比ColorLayer多，Sideband，Cursor和其他的UI图层都属于BufferLayer，每种类型Layer处理不台一样。这里比较难的是Fence的处理。Buffer也只是将Buffer的handle传给底层的HWC，并没有传Buffer里面的内容。

``` cpp
Error Composer::setLayerBuffer(Display display, Layer layer,
        uint32_t slot, const sp<GraphicBuffer>& buffer, int acquireFence)
{
    mWriter.selectDisplay(display);
    mWriter.selectLayer(layer);
    if (mIsUsingVrComposer && buffer.get()) {
        ... ...//VR
    }

    const native_handle_t* handle = nullptr;
    if (buffer.get()) {
        handle = buffer->getNativeBuffer()->handle;
    }

    mWriter.setLayerBuffer(slot, handle, acquireFence);
    return Error::NONE;
}
```
所以，每层Layer的数据，要么的Buffer，要么是固定的颜色。在处理每一层的数据时，还要处理widecolor。首先通过bestTargetDataSpace找到每层Layer的最佳DataSpace，然后再通过pickColorMode选取颜色模式，最后通过setActiveColorModeInternal函数设置。

``` cpp
void SurfaceFlinger::setActiveColorModeInternal(const sp<DisplayDevice>& hw,
        android_color_mode_t mode) {
    int32_t type = hw->getDisplayType();
    android_color_mode_t currentMode = hw->getActiveColorMode();

    ... ...

    hw->setActiveColorMode(mode);
    getHwComposer().setActiveColorMode(type, mode);
}
```

回到setUpHWComposer函数

##### 4.prepareFrame准备数据
![Alt text | center](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/display.system/Android.PG4.HW2_present.png)

``` cpp
for (size_t displayId = 0; displayId < mDisplays.size(); ++displayId) {
        auto& displayDevice = mDisplays[displayId];
        if (!displayDevice->isDisplayOn()) {
            continue;
        }

        status_t result = displayDevice->prepareFrame(*getBE().mHwc);
        ALOGE_IF(result != NO_ERROR, "prepareFrame for display %zd failed:"
                " %d (%s)", displayId, result, strerror(-result));
    }
}
```
Prepare流程，现在写的有点隐晦，以前都是直接在SurfaceFlinger中调的Prepare。现在通过DisplayDevice来完成，调用每个DisplayDevice的prepareFrame。

prepareFrame函数如下：

``` cpp
status_t DisplayDevice::prepareFrame(HWComposer& hwc) {
    status_t error = hwc.prepare(*this);
    if (error != NO_ERROR) {
        return error;
    }

    DisplaySurface::CompositionType compositionType;
    bool hasClient = hwc.hasClientComposition(mHwcDisplayId);
    bool hasDevice = hwc.hasDeviceComposition(mHwcDisplayId);
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
```
设置Layer数据时，已经指定每一层的合成方式，但是那是SurfaceFlinger的一厢情愿，还得看HWC接受不接受。HWComposer的prepare函数如下：

``` cpp
status_t HWComposer::prepare(DisplayDevice& displayDevice) {
    ATRACE_CALL();

    Mutex::Autolock _l(mDisplayLock);
    auto displayId = displayDevice.getHwcDisplayId();
    ... ...

    auto& displayData = mDisplayData[displayId];
    auto& hwcDisplay = displayData.hwcDisplay;
    if (!hwcDisplay->isConnected()) {
        return NO_ERROR;
    }

    ... ...
    if (!displayData.hasClientComposition) {
        sp<android::Fence> outPresentFence;
        uint32_t state = UINT32_MAX;
        error = hwcDisplay->presentOrValidate(&numTypes, &numRequests, &outPresentFence , &state);
        if (error != HWC2::Error::None && error != HWC2::Error::HasChanges) {
            ALOGV("skipValidate: Failed to Present or Validate");
            return UNKNOWN_ERROR;
        }
        if (state == 1) { //Present Succeeded.
            std::unordered_map<HWC2::Layer*, sp<Fence>> releaseFences;
            error = hwcDisplay->getReleaseFences(&releaseFences);
            displayData.releaseFences = std::move(releaseFences);
            displayData.lastPresentFence = outPresentFence;
            displayData.validateWasSkipped = true;
            displayData.presentError = error;
            return NO_ERROR;
        }
        // Present failed but Validate ran.
    } else {
        error = hwcDisplay->validate(&numTypes, &numRequests);
    }
    ... ...

    std::unordered_map<HWC2::Layer*, HWC2::Composition> changedTypes;
    changedTypes.reserve(numTypes);
    error = hwcDisplay->getChangedCompositionTypes(&changedTypes);
    ... ...

    displayData.displayRequests = static_cast<HWC2::DisplayRequest>(0);
    std::unordered_map<HWC2::Layer*, HWC2::LayerRequest> layerRequests;
    layerRequests.reserve(numRequests);
    error = hwcDisplay->getRequests(&displayData.displayRequests,
            &layerRequests);
    if (error != HWC2::Error::None) {
        ALOGE("prepare: getRequests failed on display %d: %s (%d)", displayId,
                to_string(error).c_str(), static_cast<int32_t>(error));
        return BAD_INDEX;
    }

    displayData.hasClientComposition = false;
    displayData.hasDeviceComposition = false;
    for (auto& layer : displayDevice.getVisibleLayersSortedByZ()) {
        auto hwcLayer = layer->getHwcLayer(displayId);

        if (changedTypes.count(hwcLayer) != 0) {
            // We pass false so we only update our state and don't call back
            // into the HWC device
            validateChange(layer->getCompositionType(displayId),
                    changedTypes[hwcLayer]);
            layer->setCompositionType(displayId, changedTypes[hwcLayer], false);
        }

        switch (layer->getCompositionType(displayId)) {
            ... ...
        }

        if (layerRequests.count(hwcLayer) != 0 &&
                layerRequests[hwcLayer] ==
                        HWC2::LayerRequest::ClearClientTarget) {
            layer->setClearClientTarget(displayId, true);
        } else {
            if (layerRequests.count(hwcLayer) != 0) {
                ALOGE("prepare: Unknown layer request: %s",
                        to_string(layerRequests[hwcLayer]).c_str());
            }
            layer->setClearClientTarget(displayId, false);
        }
    }

    error = hwcDisplay->acceptChanges();
    ... ...

    return NO_ERROR;
}
```

prepare流程如下：

- 首先尝试Prepare和Present一次处理完成
如果SurfaceFlinger没有指定得有Client端合成hasClientComposition为false，首先通过presentOrValidate接口尝试直接present，如果HWC不能直接显示，再执行validate操纵，这时的流程和validate是类似的。如果成功，那么此次数据就显示了，不用再 继续后续的处理。
- validate刷新

``` cpp
android/frameworks/native/services/surfaceflinger/DisplayHardware/HWC2.cpp
Error Display::validate(uint32_t* outNumTypes, uint32_t* outNumRequests)
{
    uint32_t numTypes = 0;
    uint32_t numRequests = 0;
    auto intError = mComposer.validateDisplay(mId, &numTypes, &numRequests);
    auto error = static_cast<Error>(intError);
    if (error != Error::None && error != Error::HasChanges) {
        return error;
    }

    *outNumTypes = numTypes;
    *outNumRequests = numRequests;
    return error;
}
```
注意Composer的 validateDisplay 函数，和其他Composer的区别：

``` cpp
Error Composer::validateDisplay(Display display, uint32_t* outNumTypes,
        uint32_t* outNumRequests)
{
    mWriter.selectDisplay(display);
    mWriter.validateDisplay();

    Error error = execute();
    if (error != Error::NONE) {
        return error;
    }

    mReader.hasChanges(display, outNumTypes, outNumRequests);

    return Error::NONE;
}
```
validateDisplay 也是通过CommandWriter写Buffer的方式调用到HWC中的，但是这里多了一个execute函数。其实，validateDisplay之前的通过，Buffer命令的调用，都还没有真正的调到HWC中，只是将命令写到了Buffer中。这里的execute才真正的调用，这里将触发HWC的服务端去解析Buffer命令，再分别去调HWC中对应的实现函数。
比如设置 z-order的解析如下：

``` cpp
bool ComposerClient::CommandReader::parseSetLayerZOrder(uint16_t length)
{
    if (length != CommandWriterBase::kSetLayerZOrderLength) {
        return false;
    }

    auto err = mHal.setLayerZOrder(mDisplay, mLayer, read());
    if (err != Error::NONE) {
        mWriter.setError(getCommandLoc(), err);
    }

    return true;
}
```


- 获取HWC的validate结果
如果SurfaceFlinger指定的合成方式HWC不能处理，通过getChangedCompositionTypes函数获取到HWC对合成方式的修改，保存在 changedTypes 中。获取LayerRequest，保存在layerRequests中。layerRequests和changedTypes都是以HWC2::Layer作为key的map。
- 修改合成方式
如果合成方式HWC不接受，SurfaceFlinger中修改根据HWC的反馈进行修改，也就是changedTypes中的Layer进行修改。修改的函数为setCompositionType，注意这里的callIntoHwc参数为false。
- 响应layerRequests
layerRequests主要是决定是否需要清楚Client端的Target，也就是Client的合成结果，留意clearClientTarget，看看后续的流程是怎么处理的。

``` cpp
void Layer::setClearClientTarget(int32_t hwcId, bool clear) {
    if (getBE().mHwcLayers.count(hwcId) == 0) {
        ALOGE("setClearClientTarget called without a valid HWC layer");
        return;
    }
    getBE().mHwcLayers[hwcId].clearClientTarget = clear;
}
```
- 最后接受修改
通过HWC，SurfaceFlinger接受修改。

``` cpp
Error Display::acceptChanges()
{
    auto intError = mComposer.acceptDisplayChanges(mId);
    return static_cast<Error>(intError);
}
```
![Alt text | center](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/display.system/Android.PG4.SurfaceFlinger_HWC_negotiation_design.png)

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/display.system/Android.PG4.SurfaceFlinger_HWC_negotiation.png)

到此setUpHWComposer结束，此时，我们需要显示的数据已经送到HWC，且每一层Layer的合成方式已经确定。如果是HWC能支持更新和显示同时完成，那么此时数据已经开始显示。
回到handleMessageRefresh函数，接下来是doDebugFlashRegions。doDebugFlashRegions只是一个debug的功能，其目前就是更新的区域不停的闪烁，收mDebugRegion的控制。

``` cpp
void SurfaceFlinger::doDebugFlashRegions()
{
    // is debugging enabled
    if (CC_LIKELY(!mDebugRegion))
        return;

}
```

接下来的doTracing也是一个debug的辅助功能。

``` cpp
void SurfaceFlinger::doTracing(const char* where) {
    ATRACE_CALL();
    ATRACE_NAME(where);
    if (CC_UNLIKELY(mTracing.isEnabled())) {
        mTracing.traceLayers(where, dumpProtoInfo(LayerVector::StateSet::Drawing));
    }
}
```

##### 四、合成处理 doComposition
如果present和validate没有一起完成，那么此时我们需要显示的数据已经送到HWC，且每一层Layer的合成方式已经确定。接下来的合成处理流程在doComposition中完成。

doComposition函数：
``` cpp
void SurfaceFlinger::doComposition() {
    ATRACE_CALL();
    ALOGV("doComposition");

    const bool repaintEverything = android_atomic_and(0, &mRepaintEverything);
    for (size_t dpy=0 ; dpy<mDisplays.size() ; dpy++) {
        const sp<DisplayDevice>& hw(mDisplays[dpy]);
        if (hw->isDisplayOn()) {
            // transform the dirty region into this screen's coordinate space
            const Region dirtyRegion(hw->getDirtyRegion(repaintEverything));

            // repaint the framebuffer (if needed)
            doDisplayComposition(hw, dirtyRegion);

            hw->dirtyRegion.clear();
            hw->flip();
        }
    }
    postFramebuffer();
}
```

合成处理也是每个Display各自 进行的，合成处理主要步骤如下：

##### 1.获取脏区域
在前面重构Layer时，Display的脏区域dirtyRegion已经计算出来。如果是重画，mRepaintEverything为true，那么脏区域就是整个屏幕的大小。
``` cpp
Region DisplayDevice::getDirtyRegion(bool repaintEverything) const {
    Region dirty;
    if (repaintEverything) {
        dirty.set(getBounds());
    } else {
        const Transform& planeTransform(mGlobalTransform);
        dirty = planeTransform.transform(this->dirtyRegion);
        dirty.andSelf(getBounds());
    }
    return dirty;
}
```
##### 2.Display合成处理
doDisplayComposition函数如下：

``` cpp
void SurfaceFlinger::doDisplayComposition(
        const sp<const DisplayDevice>& displayDevice,
        const Region& inDirtyRegion)
{
    // 需要HWC处理，或者脏区域不为空
    bool isHwcDisplay = displayDevice->getHwcDisplayId() >= 0;
    if (!isHwcDisplay && inDirtyRegion.isEmpty()) {
        ALOGV("Skipping display composition");
        return;
    }

    ALOGV("doDisplayComposition");
    if (!doComposeSurfaces(displayDevice)) return;

    // swap buffers (presentation)
    displayDevice->swapBuffers(getHwComposer());
}
```

合成操纵主要在doComposeSurfaces函数中完成，合成方式，主要就两种，一种Client端用GPU合成；另外一种，Device端合成，用的是HWC硬件。doComposeSurfaces主要是处理Client端合成，Client通过RenderEngine用GPU来进行合成。
doComposeSurfaces 函数如下，我们分段看：

##### RenderEngine 的初始化

``` cpp
bool SurfaceFlinger::doComposeSurfaces(const sp<const DisplayDevice>& displayDevice)
{
    ... ...

    const Region bounds(displayDevice->bounds());
    const DisplayRenderArea renderArea(displayDevice);
    const auto hwcId = displayDevice->getHwcDisplayId();

    mat4 oldColorMatrix;
    const bool applyColorMatrix = !getBE().mHwc->hasDeviceComposition(hwcId) &&
            !getBE().mHwc->hasCapability(HWC2::Capability::SkipClientColorTransform);
    if (applyColorMatrix) {
        mat4 colorMatrix = mColorMatrix * mDaltonizer();
        oldColorMatrix = getRenderEngine().setupColorTransform(colorMatrix);
    }

    bool hasClientComposition = getBE().mHwc->hasClientComposition(hwcId);
    if (hasClientComposition) {
        ALOGV("hasClientComposition");

        getBE().mRenderEngine->setWideColor(
                displayDevice->getWideColorSupport() && !mForceNativeColorMode);
        getBE().mRenderEngine->setColorMode(mForceNativeColorMode ?
                HAL_COLOR_MODE_NATIVE : displayDevice->getActiveColorMode());
        if (!displayDevice->makeCurrent()) {
            ... ...
        }

        const bool hasDeviceComposition = getBE().mHwc->hasDeviceComposition(hwcId);
        if (hasDeviceComposition) {
            getBE().mRenderEngine->clearWithColor(0, 0, 0, 0);
        } else {
            const Region letterbox(bounds.subtract(displayDevice->getScissor()));

            // compute the area to clear
            Region region(displayDevice->undefinedRegion.merge(letterbox));

            // screen is already cleared here
            if (!region.isEmpty()) {
                // can happen with SurfaceView
                drawWormhole(displayDevice, region);
            }
        }

        if (displayDevice->getDisplayType() != DisplayDevice::DISPLAY_PRIMARY) {

            const Rect& bounds(displayDevice->getBounds());
            const Rect& scissor(displayDevice->getScissor());
            if (scissor != bounds) {
                const uint32_t height = displayDevice->getHeight();
                getBE().mRenderEngine->setScissor(scissor.left, height - scissor.bottom,
                        scissor.getWidth(), scissor.getHeight());
            }
        }
    }
```
RenderEngine的初始化包括：

- 指定颜色矩阵 setupColorTransform
- 指定是否用WideColor setWideColor
- 指定颜色模式 setColorMode
- 设置FBTarget Surface，视窗，投影矩阵等
- 这个过程在DisplayDevice的makeCurrent中完成：

``` cpp
bool DisplayDevice::makeCurrent() const {
    bool success = mFlinger->getRenderEngine().setCurrentSurface(mSurface);
    setViewportAndProjection();
    return success;
}
```
- FBTarget 处理背景
如果是混合模式，也就是hasClientComposition和hasDeviceComposition，先清掉FBTarget背景。一般情况，合成很少采用这种方式。基本都是通过drawWormhole将屏幕填充为RGBA_0000。
- 设置剪切区 setScissor
对于非主屏，通过setScissor设置Display的剪切区

到此，初始化完成～

##### 将Client端的Layer渲染到FBTarget

``` cpp
bool SurfaceFlinger::doComposeSurfaces(const sp<const DisplayDevice>& displayDevice)
{
    ... ...

    const Transform& displayTransform = displayDevice->getTransform();
    if (hwcId >= 0) {
        // hwcId >=0 我们使用HWC
        bool firstLayer = true;
        for (auto& layer : displayDevice->getVisibleLayersSortedByZ()) {
            const Region clip(bounds.intersect(
                    displayTransform.transform(layer->visibleRegion)));

            if (!clip.isEmpty()) {
                switch (layer->getCompositionType(hwcId)) {
                    case HWC2::Composition::Cursor:
                    case HWC2::Composition::Device:
                    case HWC2::Composition::Sideband:
                    case HWC2::Composition::SolidColor: {
                        const Layer::State& state(layer->getDrawingState());
                        if (layer->getClearClientTarget(hwcId) && !firstLayer &&
                                layer->isOpaque(state) && (state.color.a == 1.0f)
                                && hasClientComposition) {
                            // never clear the very first layer since we're
                            // guaranteed the FB is already cleared
                            layer->clearWithOpenGL(renderArea);
                        }
                        break;
                    }
                    case HWC2::Composition::Client: {
                        layer->draw(renderArea, clip);
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
    } else {
        // we're not using h/w composer
        for (auto& layer : displayDevice->getVisibleLayersSortedByZ()) {
            const Region clip(bounds.intersect(
                    displayTransform.transform(layer->visibleRegion)));
            if (!clip.isEmpty()) {
                layer->draw(renderArea, clip);
            }
        }
    }

    if (applyColorMatrix) {
        getRenderEngine().setupColorTransform(oldColorMatrix);
    }

    // disable scissor at the end of the frame
    getBE().mRenderEngine->disableScissor();
    return true;
}

```
hwcId >= 0说明我们用到了HWC，大多数情况都会走到这里。对于很多VirtualDisplay的情况，hwcId为-1。
用到hwc时，首先计算每一层Layer的可见区域在Display中的区域clip，如果Layer是Client合成，那么直接调layer->draw，将Layer中clip区域绘制到FBTarget上。如果不是Client合成，但是有其他Layer是Client合成时，需要将Layer在 FBTarget中对应的区域清理掉clearWithOpenGL，清理掉的区域HWC合成。最终，FBTarget的内容和HWC中的内容再合成为最后的显示数据。
没有用到hwc时，直接调layer->draw，将Layer中clip区域绘制到FBTarget上。
到此，doComposeSurfaces完成，这里主要是处理和Client端合成相关的流程～
##### 3.Display 交换Buffer

``` cpp
void DisplayDevice::swapBuffers(HWComposer& hwc) const {
    if (hwc.hasClientComposition(mHwcDisplayId)) {
        mSurface.swapBuffers();
    }

    status_t result = mDisplaySurface->advanceFrame();
    if (result != NO_ERROR) {
        ALOGE("[%s] failed pushing new frame to HWC: %d",
                mDisplayName.string(), result);
    }
}
```

如果有Client合成，调eglSwapBuffers交换Buffer

``` cpp
void Surface::swapBuffers() const {
    if (!eglSwapBuffers(mEGLDisplay, mEGLSurface)) {
        ... ...
    }
}
```

mDisplaySurface的advanceFrame方法，虚显用的VirtualDisplaySurface，非虚显用的FramebufferSurface。advanceFrame获取 FBTarget 的数据，我们看非虚显：

``` cpp
status_t FramebufferSurface::advanceFrame() {
    uint32_t slot = 0;
    sp<GraphicBuffer> buf;
    sp<Fence> acquireFence(Fence::NO_FENCE);
    android_dataspace_t dataspace = HAL_DATASPACE_UNKNOWN;
    status_t result = nextBuffer(slot, buf, acquireFence, dataspace);
    mDataSpace = dataspace;
    if (result != NO_ERROR) {
        ALOGE("error latching next FramebufferSurface buffer: %s (%d)",
                strerror(-result), result);
    }
    return result;
}
```
主要在nextBuffer函数中完成：

``` cpp
status_t FramebufferSurface::nextBuffer(uint32_t& outSlot,
        sp<GraphicBuffer>& outBuffer, sp<Fence>& outFence,
        android_dataspace_t& outDataspace) {
    Mutex::Autolock lock(mMutex);

    BufferItem item;
    status_t err = acquireBufferLocked(&item, 0);
    ... ...
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
    mHwcBufferCache.getHwcBuffer(mCurrentBufferSlot, mCurrentBuffer,
            &outSlot, &outBuffer);
    outDataspace = item.mDataSpace;
    status_t result =
            mHwc.setClientTarget(mDisplayType, outSlot, outFence, outBuffer, outDataspace);
    ... ...
}
```
nextBuffer函数中：

- 获取一个Buffer
如果是Client合成，swapBuffer时，将调用queueBuffer，queue到FrameBufferSurface的BufferQueue中。这里的acquireBufferLocked 将从BufferQueue中获取一个Buffer。
- 替换Buffer
当前Buffer的序号mCurrentBufferSlot，当前Buffer mCurrentBuffer，对应的Fence mCurrentFence；如果新获取到的Buffer不一样，释放旧的。Buffer都被cache到mHwcBufferCache中。
-  将 FBTarget 设置给HWC
关键代码mHwc.setClientTarget。

``` cpp
status_t HWComposer::setClientTarget(int32_t displayId, uint32_t slot,
        const sp<Fence>& acquireFence, const sp<GraphicBuffer>& target,
        android_dataspace_t dataspace) {
    ... ...

    auto& hwcDisplay = mDisplayData[displayId].hwcDisplay;
    auto error = hwcDisplay->setClientTarget(slot, target, acquireFence, dataspace);
    if (error != HWC2::Error::None) {
        ALOGE("Failed to set client target for display %d: %s (%d)", displayId,
                to_string(error).c_str(), static_cast<int32_t>(error));
        return BAD_VALUE;
    }

    return NO_ERROR;
}
```
FBTarget 也是通过Command Buffer的方式传到HWC中的。在hwc1.x的版本中，在创建工作列表时也为FBTarget创建了一个Layer，HWC2版本直接传递FBTarget。
回到doComposition函数中，DisplayDevice的flip函数，将记录flip的次数mPageFlipCount。

``` cpp
void DisplayDevice::flip() const
{
    mFlinger->getRenderEngine().checkErrors();
    mPageFlipCount++;
}
```
##### 4.提交Framebuffer
到此，我们显示的数据成什么样了？需要Client合成的，已经合成完了，合成后的结果FBTarget已传给HWC。需要Device合成的数据之前也提交给HWC了。但是数据还没有最终合成显示出来。postFramebuffer 函数就是告诉HWC开始做最后的合成了。

##### 五、postFramebuffer()函数
![Alt text | center](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/display.system/Android.PG4.HW2_postframebuffer.png)
可以看到，这就跟前面的分析衔接起来了。
[Android P Graphics System（三）：Qualcomm HWC2（Hardware Composer 2.0 ）分析](https://zhoujinjian.com/posts/20190708/)
postFramebuffer函数如下：
[SurfaceFlinger.cpp]

``` cpp
void SurfaceFlinger::postFramebuffer()
{
    ... ...

    for (size_t displayId = 0; displayId < mDisplays.size(); ++displayId) {
        auto& displayDevice = mDisplays[displayId];
        if (!displayDevice->isDisplayOn()) {
            continue;
        }
        const auto hwcId = displayDevice->getHwcDisplayId();
        if (hwcId >= 0) {
            getBE().mHwc->presentAndGetReleaseFences(hwcId);
        }
        displayDevice->onSwapBuffersCompleted();
        displayDevice->makeCurrent();
        for (auto& layer : displayDevice->getVisibleLayersSortedByZ()) {
            auto hwcLayer = layer->getHwcLayer(hwcId);
            sp<Fence> releaseFence = getBE().mHwc->getLayerReleaseFence(hwcId, hwcLayer);

            if (layer->getCompositionType(hwcId) == HWC2::Composition::Client) {
                releaseFence = Fence::merge("LayerRelease", releaseFence,
                        displayDevice->getClientTargetAcquireFence());
            }

            layer->onLayerDisplayed(releaseFence);
        }

        if (!displayDevice->getLayersNeedingFences().isEmpty()) {
            sp<Fence> presentFence = getBE().mHwc->getPresentFence(hwcId);
            for (auto& layer : displayDevice->getLayersNeedingFences()) {
                layer->onLayerDisplayed(presentFence);
            }
        }

        if (hwcId >= 0) {
            getBE().mHwc->clearReleaseFences(hwcId);
        }
    }

    mLastSwapBufferTime = systemTime() - now;
    mDebugInSwapBuffers = 0;

    // |mStateLock| not needed as we are on the main thread
    uint32_t flipCount = getDefaultDisplayDeviceLocked()->getPageFlipCount();
    if (flipCount % LOG_FRAME_STATS_PERIOD == 0) {
        logFrameStats();
    }
}
```

postFramebuffer的流程如下：

通过presentAndGetReleaseFences显示获取releaseFence


``` cpp
status_t HWComposer::presentAndGetReleaseFences(int32_t displayId) {
    ... ...

    auto& displayData = mDisplayData[displayId];
    auto& hwcDisplay = displayData.hwcDisplay;

    if (displayData.validateWasSkipped) {
        // explicitly flush all pending commands
        auto error = mHwcDevice->flushCommands();
        ... ...
    }

    auto error = hwcDisplay->present(&displayData.lastPresentFence);
    if (error != HWC2::Error::None) {
        ... ...
    }

    std::unordered_map<HWC2::Layer*, sp<Fence>> releaseFences;
    error = hwcDisplay->getReleaseFences(&releaseFences);
    if (error != HWC2::Error::None) {
        ... ...
    }

    displayData.releaseFences = std::move(releaseFences);

    return NO_ERROR;
}
```
如果之前Prepare过程中，presentOrValidate成功，validateWasSkipped为 true，那么直接刷掉command Buffer中的命令，让你执行。就没有后续的处理了，presentOrValidate成功，是没有Client合成的，也就没有所谓的FBTarget。
如果之前presentOrValidate没有成功，很有可能是需要Client端做合成的，也就是present没有完成，那么这里需要走present的流程。
present操纵也是先写到Command Buffer中。最后调的execute。

``` cpp
Error Composer::presentDisplay(Display display, int* outPresentFence)
{
    mWriter.selectDisplay(display);
    mWriter.presentDisplay();

    Error error = execute();
    if (error != Error::NONE) {
        return error;
    }

    mReader.takePresentFence(display, outPresentFence);

    return Error::NONE;
}
```
设置FBTarget时，只是写到Buffer中，execute时，设置FBTarget的操纵才一起生效，设置到了Server端。Server再进行最后的合成。
present完成后，通过getReleaseFences获取releaseFence，保存在displayData中。注意这里的release是每个Layer的release fence，这是8.0之前的版本没有的流程，之前的releasefence只有一个，所以Layer只有一个。而present时的lastPresentFence就是FBTarget的releasefence。
回到postFramebuffer函数。

- DisplayDevice处理FBTarget
释放上一帧的Buffer

``` cpp
void DisplayDevice::onSwapBuffersCompleted() const {
    mDisplaySurface->onFrameCommitted();
}
```
主要在onFrameCommitted函数中完成：

``` cpp
void FramebufferSurface::onFrameCommitted() {
    if (mHasPendingRelease) {
        sp<Fence> fence = mHwc.getPresentFence(mDisplayType);
        if (fence->isValid()) {
            status_t result = addReleaseFence(mPreviousBufferSlot,
                    mPreviousBuffer, fence);
            ... ...
        }
        status_t result = releaseBufferLocked(mPreviousBufferSlot, mPreviousBuffer);
        ... ...

        mPreviousBuffer.clear();
        mHasPendingRelease = false;
    }
}
```
makeCurrent为新的合成做准备

``` cpp
bool DisplayDevice::makeCurrent() const {
    bool success = mFlinger->getRenderEngine().setCurrentSurface(mSurface);
    setViewportAndProjection();
    return success;
}
```
- 给每一层Layer设置releaseFence

``` cpp
void BufferLayer::onLayerDisplayed(const sp<Fence>& releaseFence) {
    mConsumer->setReleaseFence(releaseFence);
}
```
如果Layer需要Fence，给它presentFence，也就是FBTarget的Fence。最后清除HWComposer的mDisplayData中的releaseFence，因为他们已经传给Layer去了。

到此合成处理结束～

##### 六、合成后处理 postComposition

``` cpp
void SurfaceFlinger::postComposition(nsecs_t refreshStartTime)
{
    // Release any buffers which were replaced this frame
    nsecs_t dequeueReadyTime = systemTime();
    for (auto& layer : mLayersWithQueuedFrames) {
        layer->releasePendingBuffer(dequeueReadyTime);
    }

    // 处理Timeline
    ... ...

    // 记录Buffer状态
    mDrawingState.traverseInZOrder([&](Layer* layer) {
        bool frameLatched = layer->onPostComposition(glCompositionDoneFenceTime,
                presentFenceTime, compositorTiming);
        if (frameLatched) {
            recordBufferingStats(layer->getName().string(),
                    layer->getOccupancyHistory(false));
        }
    });

    // Vsync的同步
    if (presentFenceTime->isValid()) {
        if (mPrimaryDispSync.addPresentFence(presentFenceTime)) {
            enableHardwareVsync();
        } else {
            disableHardwareVsync(false);
        }
    }

    if (!hasSyncFramework) {
        if (hw->isDisplayOn()) {
            enableHardwareVsync();
        }
    }

    // 动画合成处理
    if (mAnimCompositionPending) {
        mAnimCompositionPending = false;

        if (presentFenceTime->isValid()) {
            mAnimFrameTracker.setActualPresentFence(
                    std::move(presentFenceTime));
        } else {
            // The HWC doesn't support present fences, so use the refresh
            // timestamp instead.
            nsecs_t presentTime =
                    getBE().mHwc->getRefreshTimestamp(HWC_DISPLAY_PRIMARY);
            mAnimFrameTracker.setActualPresentTime(presentTime);
        }
        mAnimFrameTracker.advanceFrame();
    }

    // 时间记录
}
```
中主要做了下列几件事：

- 释放待释放的Buffer
这一帧合成完成后，将被替代的Buffer释放掉～

``` cpp
void BufferLayer::releasePendingBuffer(nsecs_t dequeueReadyTime) {
    if (!mConsumer->releasePendingBuffer()) {
        return;
    }

    auto releaseFenceTime =
            std::make_shared<FenceTime>(mConsumer->getPrevFinalReleaseFence());
    mReleaseTimeline.updateSignalTimes();
    mReleaseTimeline.push(releaseFenceTime);

    Mutex::Autolock lock(mFrameEventHistoryMutex);
    if (mPreviousFrameNumber != 0) {
        mFrameEventHistory.addRelease(mPreviousFrameNumber, dequeueReadyTime,
                                      std::move(releaseFenceTime));
    }
}
```
- 处理Timeline

- 记录Buffer状态

``` cpp
void SurfaceFlinger::recordBufferingStats(const char* layerName,
        std::vector<OccupancyTracker::Segment>&& history) {
    Mutex::Autolock lock(getBE().mBufferingStatsMutex);
    auto& stats = getBE().mBufferingStats[layerName];
    for (const auto& segment : history) {
        if (!segment.usedThirdBuffer) {
            stats.twoBufferTime += segment.totalTime;
        }
        if (segment.occupancyAverage < 1.0f) {
            stats.doubleBufferedTime += segment.totalTime;
        } else if (segment.occupancyAverage < 2.0f) {
            stats.tripleBufferedTime += segment.totalTime;
        }
        ++stats.numSegments;
        stats.totalTime += segment.totalTime;
    }
}
```
- Vsync的同步
平常我们用的Vsync都是mPrimaryDispSync分发出来的，并不是每一次都是从底层硬件上报的，所以mPrimaryDispSync需要和- - 底层的硬件Vsync保持同步
- 动画合成处理
- 处理时间的记录

到此一次合成处理完成，REFRESH处理完成。下一个Vsync到来时，新的一次合成又将开始。

#### (五)、Client合成

硬件HWC合成是Vendor实现的，各个Vendor不一样。而Client合成是Android自带的，我们接下来就来看看Android的Client端的合成。
Client端合成，本质是采用GPU进程合成，SurfaceFlinger中封装了RenderEngine进行具体的实现，相关的代码在如下位置：

> frameworks/native/services/surfaceflinger/RenderEngine
我们来看看看相关的类：
![Alt text | center](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/display.system/Android.PG4.RenderEngine.png)
- RenderEngine 是对GPU渲染的封装，包括了 EGLDisplay，EGLContext， EGLConfig，EGLSurface。注意每个Display的EGLSurface不是同一个，各自有各自的EGLSurface。
- GLES20RenderEngine 继承RenderEngine，是GELS的2.0版本实现。其Program采用ProgramCache进行cache。状态用Description进描述。
- 每个BufferLayer 都有专门的Texture进行纹理的描述，GLES20RenderEngine 支持纹理贴图。合成时，将GraphicBuffer转换为纹理，进行混合。

下面我们来看具体的流程，Client端GPU合成相关的流程如下：
##### 1.创建 RenderEngine
RenderEngine 是在SurfaceFlinger初始化时，创建的。

``` cpp
void SurfaceFlinger::init() {
    ... ...
    getBE().mRenderEngine = RenderEngine::create(HAL_PIXEL_FORMAT_RGBA_8888,
            hasWideColorDisplay ? RenderEngine::WIDE_COLOR_SUPPORT : 0);
```
create函数如下：

``` cpp
std::unique_ptr<RenderEngine> RenderEngine::create(int hwcFormat, uint32_t featureFlags) {
    // 初始化EGLDisplay
    EGLDisplay display = eglGetDisplay(EGL_DEFAULT_DISPLAY);
    if (!eglInitialize(display, NULL, NULL)) {
        LOG_ALWAYS_FATAL("failed to initialize EGL");
    }

    // GLExtensions处理

    EGLint renderableType = 0;
    if (config == EGL_NO_CONFIG) {
        renderableType = EGL_OPENGL_ES2_BIT;
    } else if (!eglGetConfigAttrib(display, config, EGL_RENDERABLE_TYPE, &renderableType)) {
        LOG_ALWAYS_FATAL("can't query EGLConfig RENDERABLE_TYPE");
    }
    EGLint contextClientVersion = 0;
    if (renderableType & EGL_OPENGL_ES2_BIT) {
        contextClientVersion = 2;
    } else if (renderableType & EGL_OPENGL_ES_BIT) {
        contextClientVersion = 1;
    } else {
        LOG_ALWAYS_FATAL("no supported EGL_RENDERABLE_TYPEs");
    }

    // 初始化Attributes
    std::vector<EGLint> contextAttributes;
    ... ...

    // 创建EGLContext
    EGLContext ctxt = eglCreateContext(display, config, NULL, contextAttributes.data());

    ... ...

    // 创建PBuffer
    EGLint attribs[] = {EGL_WIDTH, 1, EGL_HEIGHT, 1, EGL_NONE, EGL_NONE};
    EGLSurface dummy = eglCreatePbufferSurface(display, dummyConfig, attribs);

    // 
    EGLBoolean success = eglMakeCurrent(display, dummy, dummy, ctxt);
    LOG_ALWAYS_FATAL_IF(!success, "can't make dummy pbuffer current");

    ... ...

    std::unique_ptr<RenderEngine> engine;
    switch (version) {
        ... ...
        case GLES_VERSION_3_0:
            engine = std::make_unique<GLES20RenderEngine>(featureFlags);
            break;
    }
    // 设置EGL信息
    engine->setEGLHandles(display, config, ctxt);

    ... ...

    eglMakeCurrent(display, EGL_NO_SURFACE, EGL_NO_SURFACE, EGL_NO_CONTEXT);
    eglDestroySurface(display, dummy);

    return engine;
}
```
RenderEngine的初始化过程，就是GPU渲染初始化的过程，做过OpenGL编程的同学来说，小case。其大概的流程如下：
- 创建 EGLDisplay
eglGetDisplay
- 初始化 EGLDisplay
eglInitialize
- 选择 EGLConfig
chooseEglConfig
- 获取renderableType
eglGetConfigAttrib
- 初始化Context属性
contextAttributes
- 创建EGLContext
eglCreateContext
- 创建 PBuffer
eglCreatePbufferSurface
- MakeCurrent
eglMakeCurrent这是为虚拟的PBuffercheck状态。
- 创建RenderEngine
这里，目前值支持GELS2.0，对应的Render GLES20RenderEngine
- 设置设置EGL信息
将创建的EGL对象设置到我们创建的GLES20RenderEngine中。

``` cpp
void RenderEngine::setEGLHandles(EGLDisplay display, EGLConfig config, EGLContext ctxt) {
    mEGLDisplay = display;
    mEGLConfig = config;
    mEGLContext = ctxt;
}
```
##### 2.创建Surface FBTarget
在RenderEngine创建时，初始化了EGLDisplaym，EGLConfig，EGLContext。这些都是所有Display共用的，但是Surface每个Display的是自己的。
在DisplayDevice创建时，创建对应的Surface

``` cpp
isplayDevice::DisplayDevice(
       ... ...
      mSurface{flinger->getRenderEngine()},
      ... ...
{
    // clang-format on
    Surface* surface;
    mNativeWindow = surface = new Surface(producer, false);
    ANativeWindow* const window = mNativeWindow.get();

    ... ...
    mSurface.setCritical(mType == DisplayDevice::DISPLAY_PRIMARY);
    mSurface.setAsync(mType >= DisplayDevice::DISPLAY_VIRTUAL);
    mSurface.setNativeWindow(window);
    mDisplayWidth = mSurface.queryWidth();
    mDisplayHeight = mSurface.queryHeight();

    ... ...

    if (useTripleFramebuffer) {
        surface->allocateBuffers();
    }
}
```
注意mSurface.setNativeWindow，通过ANativeWindow，Surface就和DisplayDevice的BufferQueue建立了联系。

``` cpp
void Surface::setNativeWindow(ANativeWindow* window) {
    if (mEGLSurface != EGL_NO_SURFACE) {
        eglDestroySurface(mEGLDisplay, mEGLSurface);
        mEGLSurface = EGL_NO_SURFACE;
    }

    mWindow = window;
    if (mWindow) {
        mEGLSurface = eglCreateWindowSurface(mEGLDisplay, mEGLConfig, mWindow, nullptr);
    }
}
```
创建的EGLSurface mEGLSurface和nativewindow mWindow 关联。这个GPU就可以通过nativewindow，从BufferQueue中dequeue Buffer进行渲染，swapBuffer时，也queue到Bufferqueu中。这里的ANativeWindow，本质就是 FBTarget。

##### 3.创建Texture
BufferLayer创建时，创建Texture：

``` cpp
BufferLayer::BufferLayer(SurfaceFlinger* flinger, const sp<Client>& client, const String8& name,
                         uint32_t w, uint32_t h, uint32_t flags)
      : Layer(flinger, client, name, w, h, flags),
        ... ...

    mFlinger->getRenderEngine().genTextures(1, &mTextureName);
    mTexture.init(Texture::TEXTURE_EXTERNAL, mTextureName);
}
```
通过glGenTextures函数创建Texture。

``` cpp
void RenderEngine::genTextures(size_t count, uint32_t* names) {
    glGenTextures(count, names);
}
```
且在创建BufferLayerConsumer时，传到了Consumer中，对应的值为mTexName。

glGenTextures生成的Texture，在BufferLayer中，保存在mTexture中。
##### 4.开始合成 doComposeSurfaces

合成是在SurfaceFlinger的doComposeSurfaces中进的，首先先makeCurrent。每个Display有自己的Surface，所以，每个Display做具体合成时，需要给RenderEngine指定Surface，视窗，投影矩阵等，告诉RenderEngine合成到哪个Surface上。

``` cpp
bool DisplayDevice::makeCurrent() const {
    bool success = mFlinger->getRenderEngine().setCurrentSurface(mSurface);
    setViewportAndProjection();
    return success;
}
```
setCurrentSurface函数如下：

``` cpp
bool RenderEngine::setCurrentSurface(const RE::Surface& surface) {
    bool success = true;
    EGLSurface eglSurface = surface.getEGLSurface();
    if (eglSurface != eglGetCurrentSurface(EGL_DRAW)) {
        success = eglMakeCurrent(mEGLDisplay, eglSurface, eglSurface, mEGLContext) == EGL_TRUE;
        if (success && surface.getAsync()) {
            eglSwapInterval(mEGLDisplay, 0);
        }
    }

    return success;
}
```
GPU不支持多线程，所以需要通过eglMakeCurrent切换GPU的工作线程，eglMakeCurrent后，GPU将处理我们当前线程的OpenGL绘图操纵。
##### 5.Layer的合成
合成时，每个Display的每个Layer都合成到Display对应的Surface上。主要是在Layer的draw方法中完成：

``` cpp
void Layer::draw(const RenderArea& renderArea, const Region& clip) const {
    onDraw(renderArea, clip, false);
}
```
BufferLayer和ColorLayer实现各自的onDraw方法。我们先来看BufferLayer，BufferLayer比较复杂。

BufferLayer的合成onDraw处理流程如下：

- 绑定Texture

``` cpp
void BufferLayer::onDraw(const RenderArea& renderArea, const Region& clip,
                         bool useIdentityTransform) const {
    ATRACE_CALL();

    if (CC_UNLIKELY(getBE().compositionInfo.mBuffer == 0)) {
        ... ...
        return;
    }

    // 绑定Texture
    status_t err = mConsumer->bindTextureImage();
    ... ...
```

``` cpp
status_t BufferLayerConsumer::bindTextureImage() {
    Mutex::Autolock lock(mMutex);
    return bindTextureImageLocked();
}
```
绑定Texture主要在bindTextureImageLocked中完成：

``` cpp
status_t BufferLayerConsumer::bindTextureImageLocked() {
    mRE.checkErrors();

    if (mCurrentTexture == BufferQueue::INVALID_BUFFER_SLOT && mCurrentTextureImage == NULL) {
        ... ...
        return NO_INIT;
    }

    const Rect& imageCrop = canUseImageCrop(mCurrentCrop) ? mCurrentCrop : Rect::EMPTY_RECT;
    status_t err = mCurrentTextureImage->createIfNeeded(imageCrop);
    if (err != NO_ERROR) {
        ... ...
        return UNKNOWN_ERROR;
    }

    mRE.bindExternalTextureImage(mTexName, mCurrentTextureImage->image());

    return doFenceWaitLocked();
}
```
mCurrentTextureImage是合成开始时，acquireBuffer时更新的。通过createIfNeeded创建Image。

``` cpp
status_t BufferLayerConsumer::Image::createIfNeeded(const Rect& imageCrop) {
    const int32_t cropWidth = imageCrop.width();
    const int32_t cropHeight = imageCrop.height();
    if (mCreated && mCropWidth == cropWidth && mCropHeight == cropHeight) {
        return OK;
    }

    mCreated = mImage.setNativeWindowBuffer(mGraphicBuffer->getNativeBuffer(),
                                            mGraphicBuffer->getUsage() & GRALLOC_USAGE_PROTECTED,
                                            cropWidth, cropHeight);
    if (mCreated) {
        ... ...

    return mCreated ? OK : UNKNOWN_ERROR;
}
```
image的创建在setNativeWindowBuffer函数中完成：

``` cpp
bool Image::setNativeWindowBuffer(ANativeWindowBuffer* buffer, bool isProtected, int32_t cropWidth,
                                  int32_t cropHeight) {
    if (mEGLImage != EGL_NO_IMAGE_KHR) {
        ... //release pre mEGLImage
    }

    if (buffer) {
        std::vector<EGLint> attrs = buildAttributeList(isProtected, cropWidth, cropHeight);
        mEGLImage = eglCreateImageKHR(mEGLDisplay, EGL_NO_CONTEXT, EGL_NATIVE_BUFFER_ANDROID,
                                      static_cast<EGLClientBuffer>(buffer), attrs.data());
        if (mEGLImage == EGL_NO_IMAGE_KHR) {
            ALOGE("failed to create EGLImage: %#x", eglGetError());
            return false;
        }
    }

    return true;
}
```
setNativeWindowBuffer时，先释放掉旧的mEGLImage。再创建新的mEGLImage。注意eglCreateImageKHR的参数。这里的buffer就是我们acquireBuffer时，获取到的GraphicBuffer。eglCreateImageKHR函数根据GraphicBuffer创建了一个mEGLImage。
回到bindTextureImageLocked函数，创建的EglImage通过bindExternalTextureImage函数绑定。

``` cpp
void RenderEngine::bindExternalTextureImage(uint32_t texName, const RE::Image& image) {
    const GLenum target = GL_TEXTURE_EXTERNAL_OES;

    glBindTexture(target, texName);
    if (image.getEGLImage() != EGL_NO_IMAGE_KHR) {
        glEGLImageTargetTexture2DOES(target, static_cast<GLeglImageOES>(image.getEGLImage()));
    }
}
```
最终通过glEGLImageTargetTexture2DOES函数，将创建的EglImage和Texture mTexName进行绑定。这样我们的Layer数据就送到了GPU进行处理。
回到onDraw方法：

- DRM处理
如果是受保护的内容，或者是Secure的内容想显示在非安全的Display上，都是不允许的。这个时候，相关的区域显示为黑色

``` cpp
void GLES20RenderEngine::setupLayerBlackedOut() {
    glBindTexture(GL_TEXTURE_2D, mProtectedTexName);
    Texture texture(Texture::TEXTURE_2D, mProtectedTexName);
    texture.setDimensions(1, 1); // FIXME: we should get that from somewhere
    mState.setTexture(texture);
}
```
- 获取textureMatrix

``` cpp
void BufferLayer::onDraw(const RenderArea& renderArea, const Region& clip,
                         bool useIdentityTransform) const {
    ... ...
    bool blackOutLayer = isProtected() || (isSecure() && !renderArea.isSecure());

    RenderEngine& engine(mFlinger->getRenderEngine());

    if (!blackOutLayer) {
        const bool useFiltering = getFiltering() || needsFiltering(renderArea) || isFixedSize();

        // Query the texture matrix given our current filtering mode.
        float textureMatrix[16];
        mConsumer->setFilteringEnabled(useFiltering);
        mConsumer->getTransformMatrix(textureMatrix);

        if (getTransformToDisplayInverse()) {
            // 处理Inverse翻转
        }

        // Set things up for texturing.
        mTexture.setDimensions(getBE().compositionInfo.mBuffer->getWidth(),
                               getBE().compositionInfo.mBuffer->getHeight());
        mTexture.setFiltering(useFiltering);
        mTexture.setMatrix(textureMatrix);

        engine.setupLayerTexturing(mTexture);
    } else {
        engine.setupLayerBlackedOut();
    }
    drawWithOpenGL(renderArea, useIdentityTransform);
    engine.disableTexturing();
}
```
textureMatrix是在GLConsumer::computeTransformMatrix中计算的，感兴趣的可以去看看。

- 用OpenGL绘制
主要通过drawWithOpenGL函数完成：

``` cpp
void BufferLayer::drawWithOpenGL(const RenderArea& renderArea, bool useIdentityTransform) const {
    const State& s(getDrawingState());

     //计算区域边界，获取Mesh
    computeGeometry(renderArea, getBE().mMesh, useIdentityTransform);

    const Rect bounds{computeBounds()}; // Rounds from FloatRect

    Transform t = getTransform();
    Rect win = bounds;
    if (!s.finalCrop.isEmpty()) {
         ... ... //处理finalCrop
    }

    float left = float(win.left) / float(s.active.w);
    float top = float(win.top) / float(s.active.h);
    float right = float(win.right) / float(s.active.w);
    float bottom = float(win.bottom) / float(s.active.h);

    // 计算Texture的坐标顶点
    Mesh::VertexArray<vec2> texCoords(getBE().mMesh.getTexCoordArray<vec2>());
    texCoords[0] = vec2(left, 1.0f - top);
    texCoords[1] = vec2(left, 1.0f - bottom);
    texCoords[2] = vec2(right, 1.0f - bottom);
    texCoords[3] = vec2(right, 1.0f - top);

    RenderEngine& engine(mFlinger->getRenderEngine());
    engine.setupLayerBlending(mPremultipliedAlpha, isOpaque(s), false /* disableTexture */,
                              getColor());
    engine.setSourceDataSpace(mCurrentState.dataSpace);
    engine.drawMesh(getBE().mMesh);
    engine.disableBlending();
}
```
setupLayerBlending处理Alpha的Blend：

``` cpp
void GLES20RenderEngine::setupLayerBlending(bool premultipliedAlpha, bool opaque,
                                            bool disableTexture, const half4& color) {
    mState.setPremultipliedAlpha(premultipliedAlpha);
    mState.setOpaque(opaque);
    mState.setColor(color);

    if (disableTexture) {
        mState.disableTexture();
    }

    if (color.a < 1.0f || !opaque) {
        glEnable(GL_BLEND);
        glBlendFunc(premultipliedAlpha ? GL_ONE : GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);
    } else {
        glDisable(GL_BLEND);
    }
}
```
drawMesh绘制内容：

``` cpp
void GLES20RenderEngine::drawMesh(const Mesh& mesh) {
    if (mesh.getTexCoordsSize()) {
        glEnableVertexAttribArray(Program::texCoords);
        glVertexAttribPointer(Program::texCoords, mesh.getTexCoordsSize(), GL_FLOAT, GL_FALSE,
                              mesh.getByteStride(), mesh.getTexCoords());
    }

    glVertexAttribPointer(Program::position, mesh.getVertexSize(), GL_FLOAT, GL_FALSE,
                          mesh.getByteStride(), mesh.getPositions());

    if (usesWideColor()) {
        Description wideColorState = mState;
        if (mDataSpace != HAL_DATASPACE_DISPLAY_P3) {
            ... ...
        }
        ProgramCache::getInstance().useProgram(wideColorState);

        glDrawArrays(mesh.getPrimitive(), 0, mesh.getVertexCount());

        if (outputDebugPPMs) {
            ... ...
        }
    } else {
        ProgramCache::getInstance().useProgram(mState);

        glDrawArrays(mesh.getPrimitive(), 0, mesh.getVertexCount());
    }

    if (mesh.getTexCoordsSize()) {
        glDisableVertexAttribArray(Program::texCoords);
    }
}
```
glDrawArrays绘制～

所有Layer都绘制完成后，swapBuffer

##### 6.交换Buffer
Surface交换Buffer

``` cpp
void Surface::swapBuffers() const {
    if (!eglSwapBuffers(mEGLDisplay, mEGLSurface)) {
        ... ...
    }
}
```

eglSwapBuffers 将交换GPU处理的Buffer，处理完的Buffer，也就是包含Layer合成数据后的Buffer将被queue到BufferQueue中。
前面已经说过advanceFrame时，将acquireBuffer，通过setClientTarget给HWC设置Client端的合成结果，传给底层进行显示。
以上就是Client端的合成处理。



#### (六)、参考资料(特别感谢)：

[（1）【Android P 图形显示系统】](https://www.jianshu.com/u/f92447ae8445/) 
[（2）【深入剖析Android系统】](https://blog.csdn.net/yangwen123/) 
[（3）【Android Display System】](http://charlesvincent.cc/)
[（4）【Android SurfaceFlinger 学习之路】](http://windrunnerlihuan.com/)











