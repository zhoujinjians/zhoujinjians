---
title: Android 11 Display System源码分析（3）：App && Native Surface创建流程、BufferQueue机制（V1）
cover: https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/post.cover.pictures/bing-wallpaper-2018.04.43.jpg
categories: 
 - Display
tags:
 - Display
toc: true
abbrlink: 20220816
date: 2022-08-16 16:16:16
---
注：文章都是通过阅读各位前辈总结的资料、Android 11 Rockchip平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除（◕‿◕），转载请注明出处（©Rockchip ©Android @Linux 版权所有），谢谢。

（==**文章基于 Android 11.0**==）

[【zhoujinjianx.com博客原图链接】](https://github.com/zhoujinjianx) 

[【开发板】](https://wiki.radxa.com/Rockpi4)

[【开发板 源码链接】](https://wiki.radxa.com/Rockpi4/rockpi-android11)



正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢，由于不喜欢图片水印，去除了水印，敬请谅解！！！

 [zhoujinjianx.com](http://zhoujinjianx.com) 




--------------------------------------------------------------------------------

==源码（部分）==：

**xxx**



==源码（部分）==：

--------------------------------------------------------------------------------

## 一、Native Surface和App Surface创建流程

### （1）、App Surface创建步骤

app中Activity回调onCreate()，setContentView最终会走到ViewRootImpl的setView中，然后接收到vSync信号后，会执行relayoutWindow

进一步会请求system_server创建Surface（SurfaceControl）。

![image-20220830135743625](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830135743625.png)

![image-20220830135838170](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830135838170.png)

![image-20220830135913055](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830135913055.png)

![image-20220830135945276](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830135945276.png)

![image-20220830140054180](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830140054180.png)

![image-20220830140124481](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830140124481.png)

![image-20220830140158063](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830140158063.png)

![image-20220830140228359](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830140228359.png)

![image-20220830140444916](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830140444916.png)

![image-20220830140800891](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830140800891.png)

走到这里已经到Native 创建Surface了，接下来看看Native Surface创建步骤。

### （2）、Native Surface创建步骤

我们这里直接找hwui的一个测试Demo。

> Y:\home\zhoujinjian\android11_rockpi4\frameworks\base\libs\hwui\tests\common\TestContext.cpp

![image-20220830112758948](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830112758948.png)

> Y:\home\zhoujinjian\android11_rockpi4\frameworks\native\libs\gui\SurfaceComposerClient.cpp

![image-20220830111100693](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830111100693.png)

![image-20220830111125226](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830111125226.png)

SurfaceComposerClient将Surface创建请求转交给保存在其成员变量中的Bp SurfaceComposerClient对象来完成。

> Y:\home\zhoujinjian\android11_rockpi4\frameworks\native\libs\gui\ISurfaceComposerClient.cpp

![image-20220830111201710](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830111201710.png)

![image-20220830111235170](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830111235170.png)

> Y:\home\zhoujinjian\android11_rockpi4\frameworks\native\services\surfaceflinger\Client.cpp

![image-20220830111339855](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830111339855.png)

![image-20220830111827145](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830111827145.png)

通过Binder通信进入SurfaceFlinger创建Layer。

> Y:\home\zhoujinjian\android11_rockpi4\frameworks\native\services\surfaceflinger\SurfaceFlinger.cpp

![image-20220830112130154](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830112130154.png)

  SurfaceFlinger根据标志位创建对应类型的Surface，当前系统定义了4种类型的Layer:

> Y:\home\zhoujinjian\android11_rockpi4\frameworks\native\libs\gui\include\gui\ISurfaceComposerClient.h

![image-20220830112219815](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830112219815.png)

> Y:\home\zhoujinjian\android11_rockpi4\frameworks\native\services\surfaceflinger\SurfaceFlinger.cpp
>

![image-20220830112340263](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830112340263.png)

在SurfaceFlinger服务端为应用程序创建的Surface创建对应的Layer对象。

![image-20220830112416247](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830112416247.png)



创建BufferQueueLayer 会首先调用onFirstRef()函数，创建IGraphicBufferProducer，IGraphicBufferConsumer，BufferQueue对象。设置最大setMaxDequeuedBufferCount为2。BufferQueueLayer继承BufferLayer，看看BufferLayer构造函数。

获取CompositionEngine创造对应layer，这个后面用于渲染作用。
看看Log：

![image-20220830112456785](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830112456785.png)

> Y:\home\zhoujinjian\android11_rockpi4\frameworks\native\services\surfaceflinger\BufferLayer.cpp

![image-20220830112644132](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830112644132.png)

上面已经创建好Layer，初始化好了GraphicBufferProducer、GraphicBufferConsumer、

## 二、BufferQueue内部机制

#### （1）、BufferQueue介绍

BufferQueue 类是 Android 中所有图形处理操作的核心。它的是将生成图形数据缓冲区的一方（生产者Producer）连接到接受数据以进行显示或进一步处理的一方（消费者Consumer）。几乎所有在系统中移动图形数据缓冲区的内容都依赖于 BufferQueue。
从上图APP与SurfaceFlinger交互中可以看出，BufferQueue内部维持着64个BufferSlot，每一个BufferSlot内部有一个GraphicBuffer指向分配的Graphic Buffer。
![Alt text](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/Android-Graphics-SurfaceFlinger-BufferQueue.png)


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

#### （2）、生产者Producer

生产者Producer实现IGraphicBufferProducer的接口，在实际运作过程中，应用（Client端）存在代理端BpGraphicBufferProducer，SurfaceFlinger（Server端）存在Native端BnGraphicBufferProducer。生产者代理端Bp通过Binder通信，不断的dequeueBuffer和queueBuffer操作，Native端同样响应这些操作请求，这样buffer就转了起来了。 
![Alt text](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/Android-Graphics-SurfaceFlinger-IGraphicsBufferProducer.png)


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

#### （3）、消费者Consumer

![Alt text](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/Android-Graphics-SurfaceFlinger-IGraphicsBufferConsumer.png)


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



## 三、Surface::dequeueBuffer、Surface::queueBuffer实例

### （1）、Surface::dequeueBuffer介绍

```c++
Stack Trace:
  RELADDR           FUNCTION                                                                                                                                                                                          FILE:LINE
  00000000000201c8  mali_gralloc_ion_map(private_handle_t*)+96                                                                                                                                                        hardware/rockchip/libgralloc/midgard/src/allocator/mali_gralloc_ion.cpp:987
  000000000001dd2c  mali_gralloc_reference_retain(native_handle const*)+260                                                                                                                                           hardware/rockchip/libgralloc/midgard/src/core/mali_gralloc_reference.cpp:62
  v-------------->  arm::mapper::common::registerBuffer(native_handle const*)                                                                                                                                         hardware/rockchip/libgralloc/midgard/src/hidl_common/Mapper.cpp:73
  000000000000fb0c  arm::mapper::common::importBuffer(android::hardware::hidl_handle const&, std::__1::function<void (android::hardware::graphics::mapper::V4_0::Error, void*)>)+124                                  hardware/rockchip/libgralloc/midgard/src/hidl_common/Mapper.cpp:274
  000000000000e450  arm::mapper::GrallocMapper::importBuffer(android::hardware::hidl_handle const&, std::__1::function<void (android::hardware::graphics::mapper::V4_0::Error, void*)>)+120                           hardware/rockchip/libgralloc/midgard/src/4.x/GrallocMapper.cpp:64
  0000000000018d88  android::hardware::graphics::mapper::V4_0::BsMapper::importBuffer(android::hardware::hidl_handle const&, std::__1::function<void (android::hardware::graphics::mapper::V4_0::Error, void*)>)+152  out/soong/.intermediates/hardware/interfaces/graphics/mapper/4.0/android.hardware.graphics.mapper@4.0_genc++_headers/gen/android/hardware/graphics/mapper/4.0/BsMapper.h:73
  000000000002e96c  android::Gralloc4Mapper::importBuffer(android::hardware::hidl_handle const&, native_handle const**) const+100                                                                                     frameworks/native/libs/ui/Gralloc4.cpp:144
  000000000003943c  android::GraphicBufferMapper::importBuffer(native_handle const*, unsigned int, unsigned int, unsigned int, int, unsigned long, unsigned int, native_handle const**)+124                           frameworks/native/libs/ui/GraphicBufferMapper.cpp:93
  0000000000037460  android::GraphicBuffer::unflatten(void const*&, unsigned long&, int const*&, unsigned long&)+688                                                                                                  frameworks/native/libs/ui/GraphicBuffer.cpp:509
  v-------------->  android::Flattenable<android::GraphicBuffer>::unflatten(void const*&, unsigned long&, int const*&, unsigned long&)                                                                                system/core/libutils/include/utils/Flattenable.h:143
  0000000000095424  android::Parcel::FlattenableHelper<android::GraphicBuffer>::unflatten(void const*, unsigned long, int const*, unsigned long)+60                                                                   frameworks/native/libs/binder/include/binder/Parcel.h:600
  000000000006527c  android::Parcel::read(android::Parcel::FlattenableHelperInterface&) const+572                                                                                                                     frameworks/native/libs/binder/Parcel.cpp:2198
  v-------------->  int android::Parcel::read<android::GraphicBuffer>(android::Flattenable<android::GraphicBuffer>&) const                                                                                            frameworks/native/libs/binder/include/binder/Parcel.h:661
  0000000000097abc  android::BpGraphicBufferProducer::requestBuffer(int, android::sp<android::GraphicBuffer>*)+300                                                                                                    frameworks/native/libs/gui/IGraphicBufferProducer.cpp:100
  00000000000b6494  android::Surface::dequeueBuffer(ANativeWindowBuffer**, int*)+716                                                                                                                                  frameworks/native/libs/gui/Surface.cpp:705
  00000000009079a0  egl_winsys_get_implementation@@Base+2712                                                                                                                                                          /vendor/lib64/egl/libGLES_mali.so
  00000000009181a4  eglGetCurrentDisplay@@Base+444                                                                                                                                                                    /vendor/lib64/egl/libGLES_mali.so
  000000000091805c  eglGetCurrentDisplay@@Base+116                                                                                                                                                                    /vendor/lib64/egl/libGLES_mali.so
  000000000099132c  glViewport@@Base+77196                                                                                                                                                                            /vendor/lib64/egl/libGLES_mali.so
  0000000000988a28  glViewport@@Base+42120                                                                                                                                                                            /vendor/lib64/egl/libGLES_mali.so
  000000000098fecc  glViewport@@Base+71980                                                                                                                                                                            /vendor/lib64/egl/libGLES_mali.so
  000000000098e81c  glViewport@@Base+66172                                                                                                                                                                            /vendor/lib64/egl/libGLES_mali.so
  000000000000d4ac  android::BootAnimation::android()+300                                                                                                                                                             frameworks/base/cmds/bootanimation/BootAnimation.cpp:634
  000000000000ccb8  android::BootAnimation::threadLoop()+80                                                                                                                                                           frameworks/base/cmds/bootanimation/BootAnimation.cpp:604
  000000000001567c  android::Thread::_threadLoop(void*)+260                                                                                                                                                           system/core/libutils/Threads.cpp:760
  0000000000014f14  thread_data_t::trampoline(thread_data_t const*)+412                                                                                                                                               system/core/libutils/Threads.cpp:97
  00000000000b0c08  __pthread_start(void*)+64                                                                                                                                                                         bionic/libc/bionic/pthread_create.cpp:347
  00000000000505d0  __start_thread+64                                                                                                                                                                                 bionic/libc/bionic/clone.cpp:53

```

![image-20220830142844255](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830142844255.png)

![image-20220830143243343](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830143243343.png)

![image-20220830143404217](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830143404217.png)

![image-20220830143427085](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830143427085.png)

![image-20220830143542207](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830143542207.png)

这里就衔接到《Android 11 Display System源码分析（1）：GraphicBuffer allocate流程》第五步：importBuffer了。

## （2）、Surface::queueBuffer介绍

![image-20220830143952610](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830143952610.png)

![image-20220830144136110](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830144136110.png)

![image-20220830144212163](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830144212163.png)

![image-20220830144610675](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display03/image-20220830144610675.png)

frameAvailableListener->onFrameAvailable(item)：通知SurfaceFlinger去消费
