---
title: Android L Display System源码分析（4）：Android Graphics 系统分析（Android 5.0.2 && Kernel 3.0.86）
cover: https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/personal.website/post.cover.pictures.00004.jpg
categories: 
  - Display
tags:
  - Android
  - Linux
  - Graphics
toc: true
abbrlink: 20191101
date: 2019-11-01 09:25:00
---


注：文章都是通过阅读 Android  && Linux 平台源码、各位前辈总结的资料、加上自己的思考分析总结出来的，其中难免有理不对的地方，欢迎大家批评指正。**文章为个人学习、研究、欣赏之用。图文内容、源码整理自互联网，如有侵权，请联系删除（◕‿◕）**，转载请注明出处（ ©Android @Linux 版权所有），谢谢(๑乛◡乛๑) 、（ ͡° ͜ʖ ͡°）、（ಡωಡ）！！！。
（==**文章基于 Kernel-3.0.86**==）&&（==**文章基于 Android 5.0.2**==）
[【开发板 - 友善之臂 FriendlyARM Cortex-A9 Tiny4412 ADK Exynos4412 （ Android 5.0.2）HD702高清电容屏 扩展套餐】](https://item.taobao.com/item.htm?spm=0.0.0.0.3GsGdQ&id=20131438062)
[【开发板 Android 5.0.2 && Kernel 3.0.86 源码链接： https://pan.baidu.com/s/1jJHm74q 密码：yfih】](https://pan.baidu.com/s/1jJHm74q)

正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢！！！

[【Android 7.1.2 (Android N) Android Graphics 系统 分析】](http://charlesvincent.cc/2018/02/01/Android-7-1-2-Android-N-Android-Graphics-%E7%B3%BB%E7%BB%9F%E5%88%86%E6%9E%90/)
[【Android图形显示之硬件抽象层Gralloc】](https://blog.csdn.net/yangwen123/article/details/12192401)

--------------------------------------------------------------------------------
==源码（部分）==：

**/frameworks/native/services/surfaceflinger/**
 - tests/Transaction_test.cpp	
 - tests/vsync/vsync.cpp

**/frameworks/native/include/gui/**
 - BitTube.h
 - BufferSlot.h
 - BufferQueueCore.h
 - BufferQueueProducer.h

**frameworks/base/core/java/android/app/**
 - Activity.java
 - ActivityThread.java
 - Instrumentation.java

**frameworks/base/core/jni/**
 - android_view_DisplayEventReceiver.cpp	
 - android_view_SurfaceControl.cpp
 - android_view_Surface.cpp
 - android_view_SurfaceSession.cpp

**frameworks/native/include/gui/**
 - SurfaceComposerClient.h
 - IDisplayEventConnection.h
 - SurfaceComposerClient.h
   

**frameworks/native/services/surfaceflinger/**
 - SurfaceFlinger.cpp
 - Client.cpp
 - main_surfaceflinger.cpp
 - DisplayDevice.cpp
 - DispSync.cpp
 - EventControlThread.cpp
 - EventThread.cpp
 - Layer.cpp
 - MonitoredProducer.cpp
   

**frameworks/base/core/java/android/view/**
 - WindowManagerImpl.java
 - ViewManager.java
 - WindowManagerGlobal.java
 - ViewRootImpl.java
 - Choreographer.java
 - IWindowSession.aidl
 - DisplayEventReceiver.java
 - SurfaceControl.java
 - Surface.java
 - SurfaceSession.java

**frameworks/native/include/ui**
 - GraphicBuffer.h
 - GraphicBufferAllocator.h

**frameworks/base/services/core/java/com/android/server/wm/**
 - WindowManagerService.java
 - Session.java
 - WindowState.java
 - WindowStateAnimator.java
 - WindowSurfaceController.java


--------------------------------------------------------------------------------

####  （一）、Android Graphics 系统框架

Android系统图形框架由下往上主要的包括HAL(HWComposer和Gralloc两个moudle)，SurfaceFlinger（BufferQueue的消费者），WindowManagerService（窗口管理者），View（BufferQueue的生产者）四大模块。
● HAL: 包括HWComposer和Gralloc两个moudle，Android N上由SurfaceFlinger打开，因此在同一进程。 gralloc 用于BufferQueue的内存分配，同时也有fb的显示接口，HWComposer作为合成SurfaceFlinger里面的Layer，并显示（通过gralloc的post函数）
● SurfaceFlinger可以叫做LayerFlinger，作为Layer的管理者，同是也是BufferQueue的消费者，当每个Layer的生产者draw完完整的一帧时，会通知SurfaceFlinger，通知的方式采用BufferQueue。
● WindowManagerService: 作为Window的管理者，掌管着计算窗口大小，窗口切换等任务，同时也会将相应的参数设置给SurfaceFlinger，比如Window的在z-order，和窗口的大小。
● View: 作为BufferQueue的生产者，每当执行lockCanvas->draw->unlockCanvas，之后会存入一帧数据进入BufferQueue中。

![enter image description here](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/personal.website/zjj.display.sys.android.graphics.arc.png)

**App**
基于Android系统的GUI框架开发完整的Apk应用。

**Android Graphics Stack Client(SurfaceFlinger  Client)**
Android在客户端的绘图堆栈通常包括：
OpenGL ES：使用GPU进行3D和2D的绘图的API
EGL：衔接GLES和系统的Native Window系统的适配层
Vulkan：Vulkan为Khronos Group推出的下一代跨平台图形开发接口，用于替代历史悠久的OpenGL。Android从7.0(Nougat)开始加入了对其的支持。Vulkan与OpenGL相比，接口更底层，从而使开发者能更直接地控制GPU。由于更好的并行支持，及更小的开销，性能上也有一定的提升。

**Android Graphics Stack Server（SurfaceFlinger  Server）**
SurfaceFlinger是Android用于管理Display和负责Window Composite（窗口混合），把应用的显示窗口输出到Display的系统服务。

**Android Drivers（HAL）**
Android的驱动层，通过Android本身的HAL（硬件抽象层）机制，运行于User Space，跟渲染相关的包括：

Hwcomposer：如果硬件支持，SurfaceFlinger可以请求hwcomposer去做窗口混合而不需要自己来做，这样的效率也会更高，减少对GPU资源的占用
Gralloc：用来管理Graphics Buffer的分配和管理系统的framebuffer
OpenGL ES/EGL

**Linux Kernel and Drivers**
除了标准的Linux内核和驱动（例如fb是framebuffer驱动），硬件厂商自己的驱动外，Android自己的一些Patches：

Ashmem：异步共享内存，用于在进程间共享一块内存区域，并允许系统在资源紧张时回收不加锁的内存块
ION：内存管理器 ION是google在Android4.0 为了解决内存碎片管理而引入的通用内存管理器,在面向程序员编程方面，它和ashmem很相似。但ION比ashmem更强大
Binder：高效的进程间通信机制
Vsync：Android 4.1引入了Vsync(Vertical Syncronization)用于渲染同步，使得App UI和SurfaceFlinger可以按硬件产生的VSync节奏来进行工作
**Hardware**
Display（显示器）、CPU、GPU、VPU（Video Process Unit）、和内存等等

#### （二）、Android Graphics 测试程序（C++）
为了便于观察对原生测试程序显示图像大小做了如下修改：

> android-5.0.2\vendor\friendly-arm\tiny4412\SurfaceFlingerTestsRed\SurfaceFlingerTestsRed.cpp

![enter image description here](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/personal.website/zjj.display.sys.SurfaceFlingerTestsRed.JPG)

我们先看一下主要步骤：
1、 创建SurfaceComposerClient

``` cpp
sp<SurfaceComposerClient> mComposerClient;
mComposerClient = new SurfaceComposerClient;
```

2、 客户端SurfaceComposerClient请求SurfaceFlinger创建Surface
注：App端对应SurfaceControl<--->SurfaceFlinger对应Layer

``` cpp
    sp<SurfaceControl> surfaceControl = client->createSurface(String8("resize"),
            800 / 3, 1280, PIXEL_FORMAT_RGB_565, 0);
```


3、处理事务，将SurfaceControl（App）的变化更新到Layer（SurfaceFlinger）图层
``` cpp
    sp<Surface> surface = surfaceControl->getSurface();

    SurfaceComposerClient::openGlobalTransaction();
    surfaceControl->setLayer(100000);
    surfaceControl->setSize(800 / 3, 1280);
    surfaceControl->setPosition(0, 0);
    SurfaceComposerClient::closeGlobalTransaction();

    ANativeWindow_Buffer outBuffer;
    surface->lock(&outBuffer, NULL);
    ssize_t bpr = outBuffer.stride * bytesPerPixel(outBuffer.format);
    android_memset16((uint16_t*)outBuffer.bits, 0xF800, bpr*outBuffer.height);
    surface->unlockAndPost();

```

接下来开始Android Graphics系统神秘探索之谜。

#### （三）、Android Graphics 禁用hwc和GPU
##### 3.1、Disable_HWUI_GPU_HWC
注：基于Android 5.0.2 Tiny4412源码，由于代码段较长，已放到GitHub
[Disable_HWUI_GPU_HWC.patch](https://github.com/weidongshan/SYS_0003_Patch_Disable_HWC_GPU_tiny4412/blob/master/android-5.0.2_no_hwc_no_gpu.patch)

##### 3.2、Vsync测试程序
Vsync(Vertical Syncronization)用于渲染同步，使得App UI和SurfaceFlinger可以按硬件产生的VSync节奏来进行工作。

查看frameworks/native/services/surfaceflinger/tests/下还有vsync测试程序

``` cpp
#include <android/looper.h>
#include <gui/DisplayEventReceiver.h>
#include <utils/Looper.h>

using namespace android;

int receiver(int fd, int events, void* data)
{
    DisplayEventReceiver* q = (DisplayEventReceiver*)data;
    ssize_t n;
    DisplayEventReceiver::Event buffer[1];
    static nsecs_t oldTimeStamp = 0;
    while ((n = q->getEvents(buffer, 1)) > 0) {
        for (int i=0 ; i<n ; i++) {
            if (buffer[i].header.type == DisplayEventReceiver::DISPLAY_EVENT_VSYNC) {
                printf("event vsync: count=%d\t", buffer[i].vsync.count);
            }
            if (oldTimeStamp) {
                float t = float(buffer[i].header.timestamp - oldTimeStamp) / s2ns(1);
                printf("%f ms (%f Hz)\n", t*1000, 1.0/t);
            }
            oldTimeStamp = buffer[i].header.timestamp;
        }
    }
    if (n<0) {printf("error reading events (%s)\n", strerror(-n));}
    return 1;
}

int main(int argc, char** argv)
{
    DisplayEventReceiver myDisplayEvent;
    sp<Looper> loop = new Looper(false);
    loop->addFd(myDisplayEvent.getFd(), 0, ALOOPER_EVENT_INPUT, receiver,
            &myDisplayEvent);
    myDisplayEvent.setVsyncRate(1);
    do {
        //printf("about to poll...\n");
        int32_t ret = loop->pollOnce(-1);
        switch (ret) {
            case ALOOPER_POLL_WAKE:
                //("ALOOPER_POLL_WAKE\n");
                break;
            case ALOOPER_POLL_CALLBACK:
                //("ALOOPER_POLL_CALLBACK\n");
                break;
            case ALOOPER_POLL_TIMEOUT:
                printf("ALOOPER_POLL_TIMEOUT\n");
                break;
            case ALOOPER_POLL_ERROR:
                printf("ALOOPER_POLL_TIMEOUT\n");
                break;
            default:
                printf("ugh? poll returned %d\n", ret);
                break;
        }
    } while (1);

    return 0;
}
```

编译运行看看：可以看到vsync信号每隔16 ms一次，关于vsync知识稍后再分析。


``` cpp
event vsync: count=2631 16.168612 ms (61.848231 Hz)
event vsync: count=2632 16.168613 ms (61.848224 Hz)
event vsync: count=2633 16.168312 ms (61.849378 Hz)
event vsync: count=2634 16.168682 ms (61.847961 Hz)
event vsync: count=2635 16.168596 ms (61.848288 Hz)
event vsync: count=2636 16.168867 ms (61.847255 Hz)
```

#### （四）、Android SurfaceFlinger 内部机制

##### 4.1、APP与SurfaceFlinger的数据结构
![enter image description here](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/personal.website/zjj.display.sys.App.SurfaceFlinger.png)

###### 4.1.1、BufferQueue介绍
BufferQueue 类是 Android 中所有图形处理操作的核心。它的是将生成图形数据缓冲区的一方（生产者Producer）连接到接受数据以进行显示或进一步处理的一方（消费者Consumer）。几乎所有在系统中移动图形数据缓冲区的内容都依赖于 BufferQueue。
从上图APP与SurfaceFlinger交互中可以看出，BufferQueue内部维持着64个BufferSlot，每一个BufferSlot内部有一个GraphicBuffer指向分配的Graphic Buffer。

![enter image description here](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/personal.website/zjj.display.sys.SurfaceFlinger.BufferQueue.png)

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
###### 4.1.2、生产者Producer
生产者Producer实现IGraphicBufferProducer的接口，在实际运作过程中，应用（Client端）存在代理端BpGraphicBufferProducer，SurfaceFlinger（Server端）存在Native端BnGraphicBufferProducer。生产者代理端Bp通过Binder通信，不断的dequeueBuffer和queueBuffer操作，Native端同样响应这些操作请求，这样buffer就转了起来了。 
![enter image description here](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/personal.website/zjj.display.sys.SurfaceFlinger.IGraphicsBufferProducer.png)


这里介绍几个非常重要的函数：
**1、requestBuffer**
requestBuffer为给定的索引请求一个新的Buffer。 服务器（即IGraphicBufferProducer实现）分配新创建的Buffer到给定的BufferSlot槽索引，并且客户端可以镜像slot->Buffer映射，这样就没有必要传输一个GraphicBuffer用于每个出队操作。

```
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
```
    // dequeueBuffer requests a new buffer slot for the client to use. Ownership
    // of the slot is transfered to the client, meaning that the server will not
    // use the contents of the buffer associated with that slot.
    //
    virtual status_t dequeueBuffer(int* slot, sp<Fence>* fence, uint32_t w,
            uint32_t h, PixelFormat format, uint32_t usage) = 0;
```
**3、detachBuffer**
detachBuffer尝试删除给定buffer 的所有权插槽从buffer queue。 如果这个请求成功，该slot将会被free，并且将无法从这个接口获得缓冲区。释放的插槽将保持未分配状态，直到被选中为止在dequeueBuffer中保存一个新分配的缓冲区，或者附加一个缓冲区到插槽。 缓冲区必须已经被取出，并且调用者必须已经拥有sp <GraphicBuffer>（即必须调用requestBuffer）
```
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
```
    // attachBuffer attempts to transfer ownership of a buffer to the buffer
    // queue. If this call succeeds, it will be as if this buffer was dequeued
    // from the returned slot number. As such, this call will fail if attaching
    // this buffer would cause too many buffers to be simultaneously dequeued.
    //
    virtual status_t attachBuffer(int* outSlot,
            const sp<GraphicBuffer>& buffer) = 0;
```

###### 4.1.3、消费者Consumer

![enter image description here](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/personal.website/zjj.display.sys.SurfaceFlinger.IGraphicsBufferConsumer.png)


这里介绍几个非常重要的函数：
**1、acquireBuffer**
acquireBuffer尝试获取下一个未决缓冲区的所有权BufferQueue。 如果没有缓冲区等待，则返回NO_BUFFER_AVAILABLE。 如果缓冲区被成功获取，有关缓冲区的信息将在BufferItem中返回。

```
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
```
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
#### 4.2、App（Java层）请求创建Surface过程
Activity创建过程以后再详细分析。直接从addToDisplay()分析。
##### 4.2.1、Session.addToDisplay()向WMS服务注册一个窗口对象；
[Session.java]
```
    @Override
    public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
            int viewVisibility, int displayId, Rect outContentInsets,
            InputChannel outInputChannel) {
        return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId,
                outContentInsets, outInputChannel);
    }

```
[WindowManagerService.java]

``` java
public int addWindow(Session session, IWindow client, int seq,
        WindowManager.LayoutParams attrs, int viewVisibility, int displayId,
        Rect outContentInsets, Rect outStableInsets, Rect outOutsets,
        InputChannel outInputChannel) {
    ......
    synchronized(mWindowMap) {
        ......
        WindowState win = new WindowState(this, session, client, token,
                attachedWindow, appOp[0], seq, attrs, viewVisibility, displayContent);

            return WindowManagerGlobal.ADD_APP_EXITING;
        }
        ......
        if (addToken) {
            mTokenMap.put(attrs.token, token);
        }
        win.attach();
        mWindowMap.put(client.asBinder(), win);
        ......
    return res;
}
```
构造一个WindowState对象，并将添加的窗口信息记录到mTokenMap和mWindowMap哈希表中。
在WMS服务端创建了所需对象后，接着调用了WindowState的attach()来进一步完成窗口添加。
[WindowState.java]
``` java
void attach() {
    if (WindowManagerService.localLOGV) Slog.v(TAG, "Attaching " + this + " token=" + mToken
        + ", list=" + mToken.windows);
    mSession.windowAddedLocked();
}
```

[Session.java]

``` java
    void windowAddedLocked() {
    if (mSurfaceSession == null) {
        if (WindowManagerService.localLOGV) Slog.v(
            TAG_WM, "First window added to " + this + ", creating SurfaceSession");
        mSurfaceSession = new SurfaceSession();
        if (SHOW_TRANSACTIONS) Slog.i(TAG_WM, "  NEW SURFACE SESSION " + mSurfaceSession);
        mService.mSessions.add(this);
        if (mLastReportedAnimatorScale != mService.getCurrentAnimatorScale()) {
            mService.dispatchNewAnimatorScaleLocked(this);
        }
    }
    mNumWindow++;
}
```
##### 4.2.2、SurfaceSession建立过程
SurfaceSession对象承担了应用程序与SurfaceFlinger之间的通信过程，每一个需要与SurfaceFlinger进程交互的应用程序端都需要创建一个SurfaceSession对象。

客户端请求
[SurfaceSession.java]

``` java
public SurfaceSession() {
    mNativeClient = nativeCreate();
}
```

Java层的SurfaceSession对象构造过程会通过JNI在native层创建一个SurfaceComposerClient对象。
[android_view_SurfaceSession.cpp]
``` java
static jlong nativeCreate(JNIEnv* env, jclass clazz) {
SurfaceComposerClient* client = new SurfaceComposerClient();
client->incStrong((void*)nativeCreate);
return reinterpret_cast<jlong>(client);
}
```

Java层的SurfaceSession对象与C++层的SurfaceComposerClient对象之间是一对一关系。
是否似曾相识，就是前面最开始SurfaceFlinger_Test程序第一步：new SurfaceComposerClient的过程。
[SurfaceComposerClient.cpp]

``` java
SurfaceComposerClient::SurfaceComposerClient()
: mStatus(NO_INIT), mComposer(Composer::getInstance()){}
void SurfaceComposerClient::onFirstRef() {
//得到SurfaceFlinger的代理对象BpSurfaceComposer  
sp<ISurfaceComposer> sm(ComposerService::getComposerService());
if (sm != 0) {
    sp<ISurfaceComposerClient> conn = sm->createConnection();
    if (conn != 0) {
        mClient = conn;
        mStatus = NO_ERROR;
    }
}
}
```

SurfaceComposerClient继承于RefBase类，当第一次被强引用时，onFirstRef函数被回调，在该函数中SurfaceComposerClient会请求SurfaceFlinger为当前应用程序创建一个Client对象，专门接收该应用程序的请求，在SurfaceFlinger端创建好Client本地Binder对象后，将该Binder代理对象返回给应用程序端，并保存在SurfaceComposerClient的成员变量mClient中。

服务端处理
在SurfaceFlinger服务端为应用程序创建交互的Client对象
[SurfaceFlinger.cpp]

``` java
sp<ISurfaceComposerClient> SurfaceFlinger::createConnection()
{
sp<ISurfaceComposerClient> bclient;
sp<Client> client(new Client(this));
status_t err = client->initCheck();
if (err == NO_ERROR) {
    bclient = client;
}
return bclient;
}
```

##### 4.2.3、App（C++层）请求创建SurfaceFlinger客户端(client)的过程
![enter image description here](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/personal.website/zjj.display.sys.App.Ask.SurfaceFlinger.Create.Client.png)


继续详细分析AppApp（C++层）请求创建SurfaceFlinger客户端(client)的过程

SurfaceComposerClient第一次强引用时，会执行onFirstRef()
[SurfaceComposerClient.cpp]

``` cpp
SurfaceComposerClient::SurfaceComposerClient()
: mStatus(NO_INIT), mComposer(Composer::getInstance()){}
void SurfaceComposerClient::onFirstRef() {
//得到SurfaceFlinger的代理对象BpSurfaceComposer  
sp<ISurfaceComposer> sm(ComposerService::getComposerService());
if (sm != 0) {
    sp<ISurfaceComposerClient> conn = sm->createConnection();
    if (conn != 0) {
        mClient = conn;
        mStatus = NO_ERROR;
    }
}
}
```

第一步：获取"SurfaceFlinger"服务
ComposerService::getComposerService()
``` cpp
/*static*/ sp<ISurfaceComposer> ComposerService::getComposerService() {
    ComposerService& instance = ComposerService::getInstance();
    Mutex::Autolock _l(instance.mLock);
    if (instance.mComposerService == NULL) {
        ComposerService::getInstance().connectLocked();
        assert(instance.mComposerService != NULL);
        ALOGD("ComposerService reconnected");
    }
    return instance.mComposerService;
}
```
 ComposerService::getInstance()会调用connectLocked()获取"SurfaceFlinger"服务。
```
ComposerService::ComposerService()
: Singleton<ComposerService>() {
    Mutex::Autolock _l(mLock);
    connectLocked();
}

void ComposerService::connectLocked() {
    const String16 name("SurfaceFlinger");
    while (getService(name, &mComposerService) != NO_ERROR) {
        usleep(250000);
    }
    ......
}
```
所以前面instance.mComposerService其实返回的是"SurfaceFlinger"服务。
第二步：createConnection()
接下来就会调用"SurfaceFlinger"服务的createConnection()

```cpp
sp<ISurfaceComposerClient> SurfaceFlinger::createConnection()
{
    sp<ISurfaceComposerClient> bclient;
    sp<Client> client(new Client(this));
    status_t err = client->initCheck();
    if (err == NO_ERROR) {
        bclient = client;
    }
    return bclient;
}
```
##### 4.2.4、APP申请创建Surface过程(Java层)


[->WindowManagerService.java]
``` java
    private int createSurfaceControl(Surface outSurface, int result, WindowState win,
        WindowStateAnimator winAnimator) {
    if (!win.mHasSurface) {
        result |= RELAYOUT_RES_SURFACE_CHANGED;
    }
    WindowSurfaceController surfaceController = winAnimator.createSurfaceLocked();
    if (surfaceController != null) {
        surfaceController.getSurface(outSurface);
    } else {
        outSurface.release();
    }
    return result;
}
```

[->WindowSurfaceController.java]

```java
    void getSurface(Surface outSurface) {
    outSurface.copyFrom(mSurfaceControl);
}
```

[->WindowStateAnimator.java]


```java
    WindowSurfaceController createSurfaceLocked() {
    ......
    try {
        ......
        mSurfaceController = new WindowSurfaceController(mSession.mSurfaceSession,
                attrs.getTitle().toString(),
                width, height, format, flags, this);
        w.setHasSurface(true);
    } 
    ......
    return mSurfaceController;
}
```

[->WindowSurfaceController.java]

``` java
public WindowSurfaceController(SurfaceSession s,
        String name, int w, int h, int format, int flags, WindowStateAnimator animator) {
    mAnimator = animator;
    mSurfaceW = w;
    mSurfaceH = h;
    ......
    if (animator.mWin.isChildWindow() &&
            animator.mWin.mSubLayer < 0 &&
            animator.mWin.mAppToken != null) {
        ......
    } else {
        mSurfaceControl = new SurfaceControl(
                s, name, w, h, format, flags);
    }
}
```
###### 4.2.4.1、APP申请创建Surface过程(C++层)
SurfaceControl创建过程
[->SurfaceControl.java]

``` java
    public SurfaceControl(SurfaceSession session,
        String name, int w, int h, int format, int flags)
                throws OutOfResourcesException {
    ......
    mNativeObject = nativeCreate(session, name, w, h, format, flags);
    ......
}
```

[->android_view_SurfaceControl.cpp]

``` cpp
static jlong nativeCreate(JNIEnv* env, jclass clazz, jobject sessionObj,
    jstring nameStr, jint w, jint h, jint format, jint flags) {
ScopedUtfChars name(env, nameStr);
sp<SurfaceComposerClient> client(android_view_SurfaceSession_getClient(env, sessionObj));
sp<SurfaceControl> surface = client->createSurface(
        String8(name.c_str()), w, h, format, flags);
if (surface == NULL) {
    jniThrowException(env, OutOfResourcesException, NULL);
    return 0;
}
surface->incStrong((void *)nativeCreate);
return reinterpret_cast<jlong>(surface.get());
}
```

该函数首先得到前面创建好的SurfaceComposerClient对象，通过该对象向SurfaceFlinger端的Client对象发送创建Surface的请求，最后得到一个SurfaceControl对象。

![enter image description here](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/personal.website/zjj.display.sys.App.Ask.SurfaceFlinger.CreateSurface.png)


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

在SurfaceFlinger服务端为应用程序创建的Surface创建对应的Layer对象。应用程序请求创建Surface过程如下：


![enter image description here](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/personal.website/zjj.display.sys.App.Ask.SurfaceFlinger.Create.Layer.png)


第一次强引用Layer对象时，onFirstRef()函数被回调
[Layer.cpp]

``` cpp
void Layer::onFirstRef() {
// Creates a custom BufferQueue for SurfaceFlingerConsumer to use
sp<IGraphicBufferProducer> producer;
sp<IGraphicBufferConsumer> consumer;
//创建BufferQueue对象
BufferQueue::createBufferQueue(&producer, &consumer);
mProducer = new MonitoredProducer(producer, mFlinger);
mSurfaceFlingerConsumer = new SurfaceFlingerConsumer(consumer, mTextureName,
        this);
mSurfaceFlingerConsumer->setConsumerUsageBits(getEffectiveUsage(0));
mSurfaceFlingerConsumer->setContentsChangedListener(this);
mSurfaceFlingerConsumer->setName(mName);
#ifdef TARGET_DISABLE_TRIPLE_BUFFERING
#warning "disabling triple buffering"
#else
mProducer->setMaxDequeuedBufferCount(2);
#endif

const sp<const DisplayDevice> hw(mFlinger->getDefaultDisplayDevice());
updateTransformHint(hw);
}
```

根据buffer可用监听器的注册过程，我们知道，当生产者也就是应用程序填充好图形buffer数据后，通过回调方式通知消费者的

###### 4.2.4.2、BufferQueue构造过程
[->BufferQueue.cpp]

``` cpp
void BufferQueue::createBufferQueue(sp<IGraphicBufferProducer>* outProducer,
    sp<IGraphicBufferConsumer>* outConsumer,
    const sp<IGraphicBufferAlloc>& allocator) {
......

sp<BufferQueueCore> core(new BufferQueueCore(allocator));
sp<IGraphicBufferProducer> producer(new BufferQueueProducer(core));
sp<IGraphicBufferConsumer> consumer(new BufferQueueConsumer(core));
*outProducer = producer;
*outConsumer = consumer;
}
```

[->BufferQueueCore.cpp]
所以核心都是这个BufferQueueCore，他是管理图形缓冲区的中枢。

``` cpp
BufferQueueCore::BufferQueueCore(const sp<IGraphicBufferAlloc>& allocator) :
mAllocator(allocator),
......
{
if (allocator == NULL) {
    sp<ISurfaceComposer> composer(ComposerService::getComposerService());
    mAllocator = composer->createGraphicBufferAlloc();
    if (mAllocator == NULL) {
        BQ_LOGE("createGraphicBufferAlloc failed");
    }
}

int numStartingBuffers = getMaxBufferCountLocked();
for (int s = 0; s < numStartingBuffers; s++) {
    mFreeSlots.insert(s);
}
for (int s = numStartingBuffers; s < BufferQueueDefs::NUM_BUFFER_SLOTS;
        s++) {
    mUnusedSlots.push_front(s);
}
}
```

BufferQueueCore类中定义了一个64项的数据mSlots，是一个容量大小为64的数组，因此BufferQueueCore可以管理最多64块的GraphicBuffer。

![enter image description here](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/personal.website/zjj.display.App.SurfaceFlinger.BufferSlot.png)

[->ISurfaceComposer.cpp]

``` cpp
   virtual sp<IGraphicBufferAlloc> createGraphicBufferAlloc()
{
    Parcel data, reply;
    data.writeInterfaceToken(ISurfaceComposer::getInterfaceDescriptor());
    remote()->transact(BnSurfaceComposer::CREATE_GRAPHIC_BUFFER_ALLOC, data, &reply);
    return interface_cast<IGraphicBufferAlloc>(reply.readStrongBinder());
}
```

[->SurfaceFlinger.cpp]

``` cpp 
sp<IGraphicBufferAlloc> SurfaceFlinger::createGraphicBufferAlloc()
{
sp<GraphicBufferAlloc> gba(new GraphicBufferAlloc());
return gba;
}
```

##### 4.2.5、**GraphicBufferAlloc构造过程**

``` cpp
android-5.0.2\frameworks\native\libs\gui\BufferQueueProducer.cpp
status_t BufferQueueProducer::dequeueBuffer(int *outSlot,
        sp<android::Fence> *outFence, bool async,
        uint32_t width, uint32_t height, uint32_t format, uint32_t usage) {
    ATRACE_CALL();
    { // Autolock scope
        Mutex::Autolock lock(mCore->mMutex);
        mConsumerName = mCore->mConsumerName;
    } // Autolock scope

    BQ_LOGV("dequeueBuffer: async=%s w=%u h=%u format=%#x, usage=%#x",
            async ? "true" : "false", width, height, format, usage);

   ......

    if (returnFlags & BUFFER_NEEDS_REALLOCATION) {
        status_t error;
        BQ_LOGV("dequeueBuffer: allocating a new buffer for slot %d", *outSlot);
        sp<GraphicBuffer> graphicBuffer(mCore->mAllocator->createGraphicBuffer(
                    width, height, format, usage, &error));
     ......
    BQ_LOGV("dequeueBuffer: returning slot=%d/%" PRIu64 " buf=%p flags=%#x",
            *outSlot,
            mSlots[*outSlot].mFrameNumber,
            mSlots[*outSlot].mGraphicBuffer->handle, returnFlags);

    return returnFlags;
}
```

[->GraphicBufferAlloc.cpp]

``` cpp
sp<GraphicBuffer> GraphicBufferAlloc::createGraphicBuffer(uint32_t width,
    uint32_t height, PixelFormat format, uint32_t usage,
    std::string requestorName, status_t* error) {
sp<GraphicBuffer> graphicBuffer(new GraphicBuffer(
        width, height, format, usage, std::move(requestorName)));
status_t err = graphicBuffer->initCheck();
......
return graphicBuffer;
}
```

##### 4.2.6、**Gralloc模块打开过程 && 图形缓冲区创建过程**
[->GraphicBuffer.cpp]

``` cpp
GraphicBuffer::GraphicBuffer(uint32_t inWidth, uint32_t inHeight,
    PixelFormat inFormat, uint32_t inUsage, std::string requestorName)
: BASE(), mOwner(ownData), mBufferMapper(GraphicBufferMapper::get()),
  mInitCheck(NO_ERROR), mId(getUniqueId()), mGenerationNumber(0)
  {
width  =
height =
stride =
format =
usage  = 0;
handle = NULL;
mInitCheck = initSize(inWidth, inHeight, inFormat, inUsage,
        std::move(requestorName));
        }
```

根据图形buffer的宽高、格式等信息为图形缓冲区分配存储空间。

``` cpp
\android-5.0.2\frameworks\native\libs\ui\GraphicBuffer.cpp

status_t GraphicBuffer::initSize(uint32_t w, uint32_t h, PixelFormat format,
        uint32_t reqUsage)
{
    GraphicBufferAllocator& allocator = GraphicBufferAllocator::get();
    status_t err = allocator.alloc(w, h, format, reqUsage, &handle, &stride);
    if (err == NO_ERROR) {
        this->width  = w;
        this->height = h;
        this->format = format;
        this->usage  = reqUsage;
    }
    return err;
}

\android-5.0.2\frameworks\native\include\ui\GraphicBufferAllocator.h
    static inline GraphicBufferAllocator& get() { return getInstance(); }

\android-5.0.2\frameworks\native\libs\ui\GraphicBufferAllocator.cpp
GraphicBufferAllocator::GraphicBufferAllocator()
    : mAllocDev(0)
{
    hw_module_t const* module;
    int err = hw_get_module(GRALLOC_HARDWARE_MODULE_ID, &module);
    ALOGE_IF(err, "FATAL: can't find the %s module", GRALLOC_HARDWARE_MODULE_ID);
    if (err == 0) {
        gralloc_open(module, &mAllocDev);
    }
}

```
##### 4.2.7、Android图形显示之硬件抽象层Gralloc
FrameBuffer驱动程序分析文中介绍了Linux系统下的显示驱动框架，每个显示屏被抽象为一个帧缓冲区，注册到FrameBuffer模块中，并在/dev/graphics目录下创建对应的fbX设备。Android系统在硬件抽象层中提供了一个Gralloc模块，封装了对帧缓冲区的所有访问操作。用户空间的应用程序在使用帧缓冲区之间，首先要加载Gralloc模块，并且获得一个gralloc设备和一个fb设备。有了gralloc设备之后，用户空间中的应用程序就可以申请分配一块图形缓冲区，并且将这块图形缓冲区映射到应用程序的地址空间来，以便可以向里面写入要绘制的画面的内容。最后，用户空间中的应用程序就通过fb设备来将已经准备好了的图形缓冲区渲染到帧缓冲区中去，即将图形缓冲区的内容绘制到显示屏中去。相应地，当用户空间中的应用程序不再需要使用一块图形缓冲区的时候，就可以通过gralloc设备来释放它，并且将它从地址空间中解除映射。

Gralloc模块实现源码位于：hardware/libhardware/modules/gralloc


├── Android.mk
├── framebuffer.cpp
├── gralloc.cpp
├── gralloc_priv.h
├── gr.h
└── mapper.cpp
Gralloc模块ID定义为：

>  GRALLOC_HARDWARE_MODULE_ID "gralloc"

同时定义了以HAL_MODULE_INFO_SYM为符号的类型为private_module_t的结构体：

``` cpp
hardware\libhardware\modules\gralloc\gralloc.cpp

static struct hw_module_methods_t gralloc_module_methods = {
        open: gralloc_device_open
};
struct private_module_t HAL_MODULE_INFO_SYM = {
    base: {
        common: {
            tag: HARDWARE_MODULE_TAG,
            version_major: 1,
            version_minor: 0,
            id: GRALLOC_HARDWARE_MODULE_ID,
            name: "Graphics Memory Allocator Module",
            author: "The Android Open Source Project",
            methods: &gralloc_module_methods
        },
        registerBuffer: gralloc_register_buffer,
        unregisterBuffer: gralloc_unregister_buffer,
        lock: gralloc_lock,
        unlock: gralloc_unlock,
    },
    framebuffer: 0,
    flags: 0,
    numBuffers: 0,
    bufferMask: 0,
    lock: PTHREAD_MUTEX_INITIALIZER,
    currentBuffer: 0,
};
```
###### 4.2.7.1、数据结构定义
在分析Gralloc模块之前，首先介绍Gralloc模块定义的一些数据结构。private_module_t用于描述Gralloc模块下的系统帧缓冲区信息

``` cpp
/hardware/libhardware/include/hardware/fb.h
/hardware/libhardware/modules/gralloc/gralloc_priv.h
struct private_module_t {
    gralloc_module_t base;
    private_handle_t* framebuffer; //指向系统帧缓冲区的句柄
    uint32_t flags; //用来标志系统帧缓冲区是否支持双缓冲
    uint32_t numBuffers;//表示系统帧缓冲区包含有多少个图形缓冲区
    uint32_t bufferMask; //记录系统帧缓冲区中的图形缓冲区的使用情况
    pthread_mutex_t lock; //一个互斥锁，用来保护结构体private_module_t的并行访问
    buffer_handle_t currentBuffer; //用来描述当前正在被渲染的图形缓冲区
    int pmem_master;
    void* pmem_master_base;
    struct fb_var_screeninfo info; //保存设备显示屏的动态属性信息
    struct fb_fix_screeninfo finfo; ////保存设备显示屏的固定属性信息
    float xdpi; //描述设备显示屏在宽度
    float ydpi; //描述设备显示屏在高度
    float fps; //用来描述显示屏的刷新频率
};
```
framebuffer_device_t用来描述系统帧缓冲区设备的信息
``` cpp
/hardware/libhardware/include/hardware/fb.h

typedef struct framebuffer_device_t {
    struct hw_device_t common;
    const uint32_t  flags;//用来记录系统帧缓冲区的标志
    const uint32_t  width;//用来描述设备显示屏的宽度
    const uint32_t  height;//用来描述设备显示屏的高度
    const int       stride;//用来描述设备显示屏的一行有多少个像素点
    const int       format;//用来描述系统帧缓冲区的像素格式
    const float     xdpi;//用来描述设备显示屏在宽度上的密度
    const float     ydpi;//用来描述设备显示屏在高度上的密度
    const float     fps;//用来描述设备显示屏的刷新频率
    const int       minSwapInterval;//用来描述帧缓冲区交换前后两个图形缓冲区的最小时间间隔
    const int       maxSwapInterval;//用来描述帧缓冲区交换前后两个图形缓冲区的最大时间间隔
    int reserved[8];//保留
	//用来设置帧缓冲区交换前后两个图形缓冲区的最小和最大时间间隔
    int (*setSwapInterval)(struct framebuffer_device_t* window,int interval);
	//用来设置帧缓冲区的更新区域
    int (*setUpdateRect)(struct framebuffer_device_t* window,int left, int top, int width, int height);
	//用来将图形缓冲区buffer的内容渲染到帧缓冲区中去
    int (*post)(struct framebuffer_device_t* dev, buffer_handle_t buffer);
	//用来通知fb设备，图形缓冲区的组合工作已经完成
    int (*compositionComplete)(struct framebuffer_device_t* dev);
    void (*dump)(struct framebuffer_device_t* dev, char *buff, int buff_len);
    int (*enableScreen)(struct framebuffer_device_t* dev, int enable);
	//保留
    void* reserved_proc[6];
} framebuffer_device_t;
```

gralloc_module_t用于描述gralloc模块信息
alloc_device_t用于描述gralloc设备的信息
``` cpp
\android-5.0.2\hardware\libhardware\include\hardware\gralloc.h
typedef struct gralloc_module_t {
　　struct hw_module_t common;
　　//映射一块图形缓冲区到一个进程的地址空间去
　　int (*registerBuffer)(struct gralloc_module_t const* module,buffer_handle_t handle);
　　//取消映射一块图形缓冲区到一个进程的地址空间去
　　int (*unregisterBuffer)(struct gralloc_module_t const* module,buffer_handle_t handle);
　　//锁定一个指定的图形缓冲区
    int (*lock)(struct gralloc_module_t const* module,buffer_handle_t handle, int usage,
            int l, int t, int w, int h,void** vaddr);
    //解锁一个指定的图形缓冲区
　　int (*unlock)(struct gralloc_module_t const* module,buffer_handle_t handle);
    int (*perform)(struct gralloc_module_t const* module,int operation, ... );
    void* reserved_proc[7];
} gralloc_module_t;

//alloc_device_t用于描述gralloc设备的信息

typedef struct alloc_device_t {
    struct hw_device_t common;
	//用于分配一块图形缓冲区
    int (*alloc)(struct alloc_device_t* dev,int w, int h, int format, int usage,buffer_handle_t* handle, int* stride);
	//用于释放指定的图形缓冲区
    int (*free)(struct alloc_device_t* dev,buffer_handle_t handle);
    void (*dump)(struct alloc_device_t *dev, char *buff, int buff_len);
    void* reserved_proc[7];
} alloc_device_t;

```
###### 4.2.7.2、Fb设备打开过程
``` cpp
/frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp
// Load and prepare the FB HAL, which uses the gralloc module.  Sets mFbDev.
int HWComposer::loadFbHalModule()
{
    hw_module_t const* module;

    int err = hw_get_module(GRALLOC_HARDWARE_MODULE_ID, &module);
    if (err != 0) {
        ALOGE("%s module not found", GRALLOC_HARDWARE_MODULE_ID);
        return err;
    }

    return framebuffer_open(module, &mFbDev);
}

```
由于我们禁用了HWComposer，所以看看framebuffer_open

``` cpp
hardware\libhardware\include\hardware\fb.h

static inline int framebuffer_open(const struct hw_module_t* module,
        struct framebuffer_device_t** device) {
    return module->methods->open(module,GRALLOC_HARDWARE_FB0, (struct hw_device_t**)device);
}
```
module指向的是一个用来描述Gralloc模块的hw_module_t结构体，前面提到，它的成员变量methods所指向的一个hw_module_methods_t结构体的成员函数open指向了Gralloc模块中的函数gralloc_device_open

``` cpp
hardware\libhardware\modules\gralloc\gralloc.cpp
int gralloc_device_open(const hw_module_t* module, const char* name,
        hw_device_t** device)
{
    int status = -EINVAL;
    if (!strcmp(name, GRALLOC_HARDWARE_GPU0)) {
       ...
    } else {
        status = fb_device_open(module, name, device);
    }
    return status;
}
```
gralloc_device_open函数即可以打开fb设备，也可以用于打开gpu设备，这里根据设备名来区分打开的设备，对应fb设备，则调用fb_device_open函数来完成设备打开操作。

``` cpp
hardware\libhardware\modules\gralloc\framebuffer.cpp
int fb_device_open(hw_module_t const* module, const char* name,
        hw_device_t** device)
{
    int status = -EINVAL;
	//判断打开的是fb设备
    if (!strcmp(name, GRALLOC_HARDWARE_FB0)) {
        alloc_device_t* gralloc_device;
		//打开gpu设备
        status = gralloc_open(module, &gralloc_device);
        if (status < 0)
            return status;
        //创建一个fb_context_t对象，用来描述fb设备上下文
        fb_context_t *dev = (fb_context_t*)malloc(sizeof(*dev));
        memset(dev, 0, sizeof(*dev));
        //初始化fb_context_t对象
        dev->device.common.tag = HARDWARE_DEVICE_TAG;
        dev->device.common.version = 0;
        dev->device.common.module = const_cast<hw_module_t*>(module);
		//注册fb设备的操作函数
        dev->device.common.close = fb_close; 
        dev->device.setSwapInterval = fb_setSwapInterval;
        dev->device.post            = fb_post;
        dev->device.setUpdateRect = 0;
        
        private_module_t* m = (private_module_t*)module;
		//将fb映射到当前进程地址空间
        status = mapFrameBuffer(m);
        if (status >= 0) {
            int stride = m->finfo.line_length / (m->info.bits_per_pixel >> 3);
            int format = (m->info.bits_per_pixel == 32)
                         ? HAL_PIXEL_FORMAT_RGBX_8888
                         : HAL_PIXEL_FORMAT_RGB_565;
            const_cast<uint32_t&>(dev->device.flags) = 0;
            const_cast<uint32_t&>(dev->device.width) = m->info.xres;
            const_cast<uint32_t&>(dev->device.height) = m->info.yres;
            const_cast<int&>(dev->device.stride) = stride;
            const_cast<int&>(dev->device.format) = format;
            const_cast<float&>(dev->device.xdpi) = m->xdpi;
            const_cast<float&>(dev->device.ydpi) = m->ydpi;
            const_cast<float&>(dev->device.fps) = m->fps;
            const_cast<int&>(dev->device.minSwapInterval) = 1;
            const_cast<int&>(dev->device.maxSwapInterval) = 1;
            *device = &dev->device.common;
        }
    }
    return status;
}

static int mapFrameBuffer(struct private_module_t* module)
{
    pthread_mutex_lock(&module->lock);
    int err = mapFrameBufferLocked(module);
    pthread_mutex_unlock(&module->lock);
    return err;
}

```

##### 4.2.8、GraphicBuffer图形缓冲区创建过程

``` cpp
frameworks\native\libs\ui\GraphicBufferAllocator.cpp

status_t GraphicBufferAllocator::alloc(uint32_t w, uint32_t h, PixelFormat format,
        int usage, buffer_handle_t* handle, int32_t* stride)
{
    ATRACE_CALL();
    // make sure to not allocate a N x 0 or 0 x N buffer, since this is
    // allowed from an API stand-point allocate a 1x1 buffer instead.
    if (!w || !h)
        w = h = 1;

    // we have a h/w allocator and h/w buffer is requested
    status_t err; 
    
    err = mAllocDev->alloc(mAllocDev, w, h, format, usage, handle, stride);

    ALOGW_IF(err, "alloc(%u, %u, %d, %08x, ...) failed %d (%s)",
            w, h, format, usage, err, strerror(-err));
    
    if (err == NO_ERROR) {
        Mutex::Autolock _l(sLock);
        KeyedVector<buffer_handle_t, alloc_rec_t>& list(sAllocList);
        int bpp = bytesPerPixel(format);
        if (bpp < 0) {
            // probably a HAL custom format. in any case, we don't know
            // what its pixel size is.
            bpp = 0;
        }
        alloc_rec_t rec;
        rec.w = w;
        rec.h = h;
        rec.s = *stride;
        rec.format = format;
        rec.usage = usage;
        rec.size = h * stride[0] * bpp;
        list.add(*handle, rec);
    }

    return err;
}

```
hardware\libhardware\modules\gralloc\gralloc.cpp

``` cpp
static int gralloc_alloc(alloc_device_t* dev,//gpu设备描述符
        int w, //图像宽度
		int h, //图像高度
		int format, //图形格式
		int usage, //图形缓冲区的使用类型
        buffer_handle_t* pHandle, //即将被分配的图形缓冲区
		int* pStride)//分配的图形缓冲区一行包含有多少个像素点
{
    if (!pHandle || !pStride)
        return -EINVAL;
    size_t size, stride;
    int align = 4;
    int bpp = 0;
    switch (format) {
        case HAL_PIXEL_FORMAT_RGBA_8888:
        case HAL_PIXEL_FORMAT_RGBX_8888:
        case HAL_PIXEL_FORMAT_BGRA_8888:
		//一个像素需要使用32位来表示，即4个字节
            bpp = 4;
            break;
        case HAL_PIXEL_FORMAT_RGB_888:
		//一个像素需要使用24位来描述，即3个字节
            bpp = 3;
            break;
        case HAL_PIXEL_FORMAT_RGB_565:
        case HAL_PIXEL_FORMAT_RGBA_5551:
        case HAL_PIXEL_FORMAT_RGBA_4444:
		//一个像需要使用16位来描述，即2个字节
            bpp = 2;
            break;
        default:
            return -EINVAL;
    }
	//w表示要分配的图形缓冲区所保存的图像的宽度，w*bpp就可以得到保存一行像素所需要使用的字节数，并且对齐到4个字节边界
    size_t bpr = (w*bpp + (align-1)) & ~(align-1);
	//h表示要分配的图形缓冲区所保存的图像的高度，bpr* h就可以得到保存整个图像所需要使用的字节数
    size = bpr * h;
	//要分配的图形缓冲区一行包含有多少个像素点
    stride = bpr / bpp;
    int err;
	//要分配的图形缓冲区一行包含有多少个像素点
    if (usage & GRALLOC_USAGE_HW_FB) {
		//系统帧缓冲区中分配图形缓冲区
        err = gralloc_alloc_framebuffer(dev, size, usage, pHandle);
    } else {
		//从内存中分配图形缓冲区
        err = gralloc_alloc_buffer(dev, size, usage, pHandle);
    }
 
    if (err < 0) {
        return err;
    }
    *pStride = stride;
    return 0;
}
```
###### 4.2.8.1、系统帧缓冲区分配Buffer
gralloc_alloc_framebuffer函数用于从FrameBuffer中分配图形缓冲区。

``` cpp
android-5.0.2\hardware\libhardware\modules\gralloc\gralloc.cpp
static int gralloc_alloc_framebuffer(alloc_device_t* dev,
        size_t size, int usage, buffer_handle_t* pHandle)
{
    private_module_t* m = reinterpret_cast<private_module_t*>(dev->common.module);
    pthread_mutex_lock(&m->lock);
    int err = gralloc_alloc_framebuffer_locked(dev, size, usage, pHandle);
    pthread_mutex_unlock(&m->lock);
    return err;
}
为了保证分配过程中线程安全，这里使用了private_module_t中的lock来同步多线程。
static int gralloc_alloc_framebuffer_locked(alloc_device_t* dev,
        size_t size, int usage, buffer_handle_t* pHandle)
{
	//根据alloc_device_t查找到系统帧缓冲区描述体
    private_module_t* m = reinterpret_cast<private_module_t*>(dev->common.module);
    /** *从系统帧缓冲区分配图形缓冲区之前，首先要对系统帧缓冲区进行过初始化。mapFrameBufferLocked函数用于系统帧缓冲区的初始化，在系统帧缓冲区初始化时，用private_ha*ndle_t来描述整个系统帧缓冲区信息，并保持到private_module_t的成员framebuffer中，如果该成员变量framebuffer为空，说明系统帧缓冲区还为被初始化。
	**/
    if (m->framebuffer == NULL) {//系统帧缓冲区还未初始化
		//初始化系统帧缓冲区，映射到当前进程的虚拟地址空间来
        int err = mapFrameBufferLocked(m);
        if (err < 0) {
            return err;
        }
    }
	//得到系统帧缓冲区的使用情况
    const uint32_t bufferMask = m->bufferMask;
	//系统帧缓冲区可以划分为多少个图形缓冲区来使用
    const uint32_t numBuffers = m->numBuffers;
	//设备显示屏一屏内容所占用的内存的大小
    const size_t bufferSize = m->finfo.line_length * m->info.yres;
	//如果系统帧缓冲区只有一个图形缓冲区大小，即变量numBuffers的值等于1，那么这个图形缓冲区就始终用作系统主图形缓冲区来使用
    if (numBuffers == 1) {
		//不能够在系统帧缓冲区中分配图形缓冲区，从内存中来分配图形缓冲区
        int newUsage = (usage & ~GRALLOC_USAGE_HW_FB) | GRALLOC_USAGE_HW_2D;
        return gralloc_alloc_buffer(dev, bufferSize, newUsage, pHandle);
    }
	//系统帧缓冲区中的图形缓冲区全部都分配出去了
    if (bufferMask >= ((1LU<<numBuffers)-1)) {
        return -ENOMEM;
    }
    //指向系统帧缓冲区的基地址
    intptr_t vaddr = intptr_t(m->framebuffer->base);
	//创建一个private_handle_t结构体hnd来描述这个即将要分配出去的图形缓冲区
    private_handle_t* hnd = new private_handle_t(dup(m->framebuffer->fd), size,private_handle_t::PRIV_FLAGS_FRAMEBUFFER);
    //从bufferMask中查找空闲的图形缓冲区
    for (uint32_t i=0 ; i<numBuffers ; i++) {
        if ((bufferMask & (1LU<<i)) == 0) {
            m->bufferMask |= (1LU<<i);
            break;
        }
		//每次从系统帧缓冲区中分配出去的图形缓冲区的大小都是刚好等于显示屏一屏内容大小的
        vaddr += bufferSize;
    }
	//分配出去的图形缓冲区的开始地址保存在创建的private_handle_t结构体hnd的成员变量base中
    hnd->base = vaddr;
	//分配出去的图形缓冲区的起始地址相对于系统帧缓冲区的基地址的偏移量
    hnd->offset = vaddr - intptr_t(m->framebuffer->base);
    *pHandle = hnd;
    return 0;
}
```

###### 4.2.8.2、内存中分配Buffer

该函数首先创建一块名为"gralloc-buffer"的匿名共享内存，并根据匿名共享内存的信息来构造一个private_handle_t对象，该对象用来描述分配的图形缓冲区，然后将创建的匿名共享内存映射到当前进程的虚拟地址空间。在系统帧缓冲区分配图形buffer时并没有执行地址空间映射，而这里却需要执行映射过程，为什么呢？这是因为在执行mapFrameBufferLocked函数初始化系统帧缓冲区时，已经将系统的整个帧缓冲区映射到当前进程地址空间中了，在系统帧缓冲区中分配buffer就不在需要重复映射了，而在内存中分配buffer就不一样，内存中分配的buffer是一块匿名共享内存，该匿名共享内存并没有映射到当前进程地址空间，因此这里就需要完成这一映射过程，从而为以后应用程序进程直接访问这块buffer做好准备工作。

``` cpp
int mapBuffer(gralloc_module_t const* module,private_handle_t* hnd)
{
    void* vaddr;
    return gralloc_map(module, hnd, &vaddr);
}
gralloc_map将参数hnd所描述的一个图形缓冲区映射到当前进程的地址空间来。
static int gralloc_map(gralloc_module_t const* module,buffer_handle_t handle,void** vaddr)
{
    private_handle_t* hnd = (private_handle_t*)handle;
	//如果当前buffer不是从系统帧缓冲区中分配的
    if (!(hnd->flags & private_handle_t::PRIV_FLAGS_FRAMEBUFFER)) {
        size_t size = hnd->size;
        void* mappedAddress = mmap(0, size,PROT_READ|PROT_WRITE, MAP_SHARED, hnd->fd, 0);
        if (mappedAddress == MAP_FAILED) {
            ALOGE("Could not mmap %s", strerror(errno));
            return -errno;
        }
        hnd->base = intptr_t(mappedAddress) + hnd->offset;
    }
    *vaddr = (void*)hnd->base;
    return 0;
}
```
在初始化系统帧缓冲区的时候，已经将系统帧缓冲区映射到进程地址空间了，因此如果发现要注册的图形缓冲区是在系统帧缓冲区分配的，那么就不需要再执行映射图形缓冲区的操作了。


##### 4.2.9、生产者Producer构造过程

```cpp
sp<IGraphicBufferProducer> producer(new BufferQueueProducer(core));
sp<IGraphicBufferConsumer> consumer(new BufferQueueConsumer(core));
```
实例化BufferQueueProducer，这里初始化了mCore(core) 和 mSlots(core->mSlots)
```cpp
BufferQueueProducer::BufferQueueProducer(const sp<BufferQueueCore>& core) :
    mCore(core),
    mSlots(core->mSlots),
    mConsumerName(),
    mStickyTransform(0),
    mLastQueueBufferFence(Fence::NO_FENCE),
    mCallbackMutex(),
    mNextCallbackTicket(0),
    mCurrentCallbackTicket(0),
    mCallbackCondition(),
    mDequeueTimeout(-1) {}
```


##### 4.2.10、消费者Consumer构造过程

```cpp
sp<IGraphicBufferProducer> producer(new BufferQueueProducer(core));
sp<IGraphicBufferConsumer> consumer(new BufferQueueConsumer(core));
```
实例化BufferQueueConsumer，这里初始化了mCore(core) 和 mSlots(core->mSlots)
```cpp
BufferQueueConsumer::BufferQueueConsumer(const sp<BufferQueueCore>& core) :
    mCore(core),
    mSlots(core->mSlots),
    mConsumerName() {}
```

###### SurfaceFlinger设置监听

```cpp
mSurfaceFlingerConsumer = new SurfaceFlingerConsumer(consumer, mTextureName,
        this);
mSurfaceFlingerConsumer->setConsumerUsageBits(getEffectiveUsage(0));
mSurfaceFlingerConsumer->setContentsChangedListener(this);
mSurfaceFlingerConsumer->setName(mName);
```
![enter image description here](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/personal.website/zjj.display.sys.App.Ask.SurfaceFlinger.ConsumeLisener.onFrameAvailable.png)

##### 4.2.11、应用程序本地窗口Surface创建过程

从前面分析可知，SurfaceFlinger在处理应用程序请求创建Surface中，在SurfaceFlinger服务端仅仅创建了Layer对象，那么应用程序本地窗口Surface在什么时候、什么地方创建呢？

为应用程序创建好了Layer对象并返回ISurface的代理对象给应用程序，应用程序通过该代理对象创建了一个SurfaceControl对象，Java层Surface需要通过android_view_Surface.cpp中的JNI函数来操作native层的Surface，在操作native层Surface前，首先需要获取到native的Surface，应用程序本地窗口Surface就是在这个时候创建的。
[->SurfaceControl.cpp]

``` cpp
sp<Surface> SurfaceControl::getSurface() const
{
Mutex::Autolock _l(mLock);
if (mSurfaceData == 0) {
    // This surface is always consumed by SurfaceFlinger, so the
    // producerControlledByApp value doesn't matter; using false.
    mSurfaceData = new Surface(mGraphicBufferProducer, false);
}
return mSurfaceData;
}
```

[Surface.cpp]

``` cpp
Surface::Surface(
    const sp<IGraphicBufferProducer>& bufferProducer,
    bool controlledByApp)
: mGraphicBufferProducer(bufferProducer),
  mCrop(Rect::EMPTY_RECT),
  mGenerationNumber(0),
  mSharedBufferMode(false),
  mAutoRefresh(false),
  mSharedBufferSlot(BufferItem::INVALID_BUFFER_SLOT),
  mSharedBufferHasBeenQueued(false),
  mNextFrameNumber(1)
  {
// Initialize the ANativeWindow function pointers.
ANativeWindow::setSwapInterval  = hook_setSwapInterval;
ANativeWindow::dequeueBuffer    = hook_dequeueBuffer;
ANativeWindow::cancelBuffer     = hook_cancelBuffer;
ANativeWindow::queueBuffer      = hook_queueBuffer;
ANativeWindow::query            = hook_query;
ANativeWindow::perform          = hook_perform;

ANativeWindow::dequeueBuffer_DEPRECATED = hook_dequeueBuffer_DEPRECATED;
ANativeWindow::cancelBuffer_DEPRECATED  = hook_cancelBuffer_DEPRECATED;
ANativeWindow::lockBuffer_DEPRECATED    = hook_lockBuffer_DEPRECATED;
ANativeWindow::queueBuffer_DEPRECATED   = hook_queueBuffer_DEPRECATED;

const_cast<int&>(ANativeWindow::minSwapInterval) = 0;
const_cast<int&>(ANativeWindow::maxSwapInterval) = 1;

mReqWidth = 0;
mReqHeight = 0;
mReqFormat = 0;
mReqUsage = 0;
......
mSwapIntervalZero = false;
}
```

在创建完应用程序本地窗口Surface后，想要在该Surface上绘图，首先需要为该Surface分配图形buffer。我们前面介绍了Android应用程序图形缓冲区的分配都是由SurfaceFlinger服务进程来完成，在请求创建Surface时，在服务端创建了一个BufferQueue本地Binder对象，该对象负责管理应用程序一个本地窗口Surface的图形缓冲区。

#### 4.3、APP申请(lock)Buffer的过程

![enter image description here](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/personal.website/zjj.display.sys.App.SurfaceFlinger.lock.unlockpost.png)

``` java
   private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
        boolean scalingRequired, Rect dirty) {

    // Draw with software renderer.
    final Canvas canvas;
    try {
        ......
        canvas = mSurface.lockCanvas(dirty);
        ......
    } ......
        try {
            canvas.translate(-xoff, -yoff);
            if (mTranslator != null) {
                mTranslator.translateCanvas(canvas);
            }
            canvas.setScreenDensity(scalingRequired ? mNoncompatDensity : 0);
            attachInfo.mSetIgnoreDirtyState = false;

            mView.draw(canvas);

            drawAccessibilityFocusedDrawableIfNeeded(canvas);
        }......
    } finally {
        try {
            surface.unlockCanvasAndPost(canvas);
        } catch (IllegalArgumentException e) {
           ......
            return false;
        }
    }
    return true;
}
```

先看看Surface的lockCanvas方法：
[->Surface.java]

``` java
//mCanvas 变量直接赋值
private final Canvas mCanvas = new CompatibleCanvas();
public Canvas lockCanvas(Rect inOutDirty)
    throws Surface.OutOfResourcesException, IllegalArgumentException {
synchronized (mLock) {
    checkNotReleasedLocked();
    ......
    mLockedObject = nativeLockCanvas(mNativeObject, mCanvas, inOutDirty);
    return mCanvas;
}
}
```

[->android_view_Surface.cpp]

``` cpp
static jlong nativeLockCanvas(JNIEnv* env, jclass clazz,
    jlong nativeObject, jobject canvasObj, jobject dirtyRectObj) {
    //获取java层的Surface保存的long型句柄
sp<Surface> surface(reinterpret_cast<Surface *>(nativeObject));

if (!isSurfaceValid(surface)) {
    doThrowIAE(env);
    return 0;
}

Rect dirtyRect(Rect::EMPTY_RECT);
Rect* dirtyRectPtr = NULL;
//获取java层dirty Rect的位置大小信息
if (dirtyRectObj) {
    dirtyRect.left   = env->GetIntField(dirtyRectObj, gRectClassInfo.left);
    dirtyRect.top    = env->GetIntField(dirtyRectObj, gRectClassInfo.top);
    dirtyRect.right  = env->GetIntField(dirtyRectObj, gRectClassInfo.right);
    dirtyRect.bottom = env->GetIntField(dirtyRectObj, gRectClassInfo.bottom);
    dirtyRectPtr = &dirtyRect;
}

ANativeWindow_Buffer outBuffer;
 //调用Surface的lock方法,将申请的图形缓冲区赋给outBuffer
status_t err = surface->lock(&outBuffer, dirtyRectPtr);
......

SkImageInfo info = SkImageInfo::Make(outBuffer.width, outBuffer.height,
                                     convertPixelFormat(outBuffer.format),
                                     outBuffer.format == PIXEL_FORMAT_RGBX_8888 ?
                                     kOpaque_SkAlphaType : kPremul_SkAlphaType);

SkBitmap bitmap;
//创建一个SkBitmap 
//图形缓冲区每一行像素大小
ssize_t bpr = outBuffer.stride * bytesPerPixel(outBuffer.format);
bitmap.setInfo(info, bpr);
if (outBuffer.width > 0 && outBuffer.height > 0) {
    bitmap.setPixels(outBuffer.bits);
} else {
    // be safe with an empty bitmap.
    bitmap.setPixels(NULL);
}

Canvas* nativeCanvas = GraphicsJNI::getNativeCanvas(env, canvasObj);
nativeCanvas->setBitmap(bitmap);

if (dirtyRectPtr) {
    nativeCanvas->clipRect(dirtyRect.left, dirtyRect.top,
            dirtyRect.right, dirtyRect.bottom);
}

if (dirtyRectObj) {
    env->SetIntField(dirtyRectObj, gRectClassInfo.left,   dirtyRect.left);
    env->SetIntField(dirtyRectObj, gRectClassInfo.top,    dirtyRect.top);
    env->SetIntField(dirtyRectObj, gRectClassInfo.right,  dirtyRect.right);
    env->SetIntField(dirtyRectObj, gRectClassInfo.bottom, dirtyRect.bottom);
}

......
sp<Surface> lockedSurface(surface);
lockedSurface->incStrong(&sRefBaseOwner);
return (jlong) lockedSurface.get();
}
```

这段代码逻辑主要如下：
       1）获取java层dirty 的Rect大小和位置信息；
       2）调用Surface的lock方法,将申请的图形缓冲区赋给outBuffer；
       3）创建一个Skbitmap，填充它用来保存申请的图形缓冲区，并赋值给Java层的Canvas对象；
       4）将剪裁位置大小信息赋给java层Canvas对象。
##### 4.3.1、Surface管理图形缓冲区-APP申请(lock)Buffer的过程
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

#### 4.4、APP提交(unlockAndPost)Buffer的过程
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
Folw：
![enter image description here](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/personal.website/zjj.display.sys.App.WMS.SurfaceFlinger.All.Flow.png)


#### （五）、通知SF消费合成
当绘制完毕的GraphicBuffer入队之后，会通知SurfaceFlinger去消费，就是BufferQueueProducer的queueBuffer函数的最后几行，listener->onFrameAvailable()。
listener最终通过回调，会回到Layer当中，所以最终调用Layer的onFrameAvailable接口，我们看看它的实现：
[Layer.cpp]

``` cpp
void Layer::onFrameAvailable(const BufferItem& item) {
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
![enter image description here](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/personal.website/zjj.display.sys.vsync.surfaceflinger.png)


![enter image description here](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/personal.website/zjj.display.sys.sf_use_vsync.png)



最终结果会走到SurfaceFlinger的vsync信号接收逻辑，即SurfaceFlinger的onMessageReceived函数：
[SurfaceFlinger.cpp]

``` cpp
void SurfaceFlinger::onMessageReceived(int32_t what) {
ATRACE_CALL();
switch (what) {
    case MessageQueue::INVALIDATE: {
        bool frameMissed = !mHadClientComposition &&
                mPreviousPresentFence != Fence::NO_FENCE &&
                mPreviousPresentFence->getSignalTime() == INT64_MAX;
        ATRACE_INT("FrameMissed", static_cast<int>(frameMissed));
        if (mPropagateBackpressure && frameMissed) {
            signalLayerUpdate();
            break;
        }

        bool refreshNeeded = handleMessageTransaction();
        refreshNeeded |= handleMessageInvalidate();
        refreshNeeded |= mRepaintEverything;
        if (refreshNeeded) {
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
}
```

SurfaceFlinger收到了VSync信号后，调用了handleMessageRefresh函数

![enter image description here](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/personal.website/zjj.display.sys.handle_vsync.png)

[SurfaceFlinger.cpp]

``` cpp
void SurfaceFlinger::handleMessageRefresh() {
ATRACE_CALL();

nsecs_t refreshStartTime = systemTime(SYSTEM_TIME_MONOTONIC);

preComposition();
rebuildLayerStacks();
setUpHWComposer();
doDebugFlashRegions();
doComposition();
postComposition(refreshStartTime);

mPreviousPresentFence = mHwc->getRetireFence(HWC_DISPLAY_PRIMARY);

mHadClientComposition = false;
for (size_t displayId = 0; displayId < mDisplays.size(); ++displayId) {
    const sp<DisplayDevice>& displayDevice = mDisplays[displayId];
    mHadClientComposition = mHadClientComposition ||
            mHwc->hasClientComposition(displayDevice->getHwcDisplayId());
}

// Release any buffers which were replaced this frame
for (auto& layer : mLayersWithQueuedFrames) {
    layer->releasePendingBuffer();
}
mLayersWithQueuedFrames.clear();
}
```

我们主要看下下面几个函数。
[SurfaceFlinger.cpp]

``` cpp
preComposition();
rebuildLayerStacks();
setUpHWComposer();
doDebugFlashRegions();
doComposition();
postComposition(refreshStartTime);
```

##### 一、preComposition()函数
我们先来看第一个函数preComposition()
[SurfaceFlinger.cpp]

``` cpp
void SurfaceFlinger::preComposition()
{
bool needExtraInvalidate = false;
const LayerVector& layers(mDrawingState.layersSortedByZ);
const size_t count = layers.size();
for (size_t i=0 ; i<count ; i++) {
    if (layers[i]->onPreComposition()) {
        needExtraInvalidate = true;
    }
}
if (needExtraInvalidate) {
    signalLayerUpdate();
}
}
```

上面函数先是调用了mDrawingState的layersSortedByZ来得到上次绘图的Layer层列表。并不是所有的Layer都会参与屏幕图像的绘制，因此SurfaceFlinger用state对象来记录参与绘制的Layer对象。
记得我们之前分析过createLayer函数来创建Layer，创建之后会调用addClientLayer函数。
[SurfaceFlinger.cpp]

``` cpp
status_t SurfaceFlinger::addClientLayer(const sp<Client>& client,
    const sp<IBinder>& handle,
    const sp<IGraphicBufferProducer>& gbc,
    const sp<Layer>& lbc)
    {
// add this layer to the current state list
{
    Mutex::Autolock _l(mStateLock);
    if (mCurrentState.layersSortedByZ.size() >= MAX_LAYERS) {
        return NO_MEMORY;
    }
    mCurrentState.layersSortedByZ.add(lbc);
    mGraphicBufferProducerList.add(IInterface::asBinder(gbc));
}

// attach this layer to the client
client->attachLayer(handle, lbc);

return NO_ERROR;
}
```

我们来看下addClientLayer函数，这里会把Layer对象放在mCurrentState的layersSortedByZ对象中。而mDrawingState和mCurrentState什么关系呢？在后面我们会介绍，mDrawingState代表上一次绘图时的状态，处理完之后会把mCurrentState赋给mDrawingState。
回到preComposition函数，遍历所有的Layer对象，调用其onPreComposition函数来检测Layer层中的图像是否有变化。

#### 1.1、每个Layer的onFrameAvailable函数
onPreComposition函数来根据mQueuedFrames来判断图像是否发生了变化，或者是mSidebandStreamChanged、mAutoRefresh。
[Layer.cpp]

``` cpp
bool Layer::onPreComposition() {
mRefreshPending = false;
return mQueuedFrames > 0 || mSidebandStreamChanged || mAutoRefresh;
}
```

当Layer所对应的Surface更新图像后，它所对应的Layer对象的onFrameAvailable函数会被调用来通知这种变化。
在SurfaceFlinger的preComposition函数中当有Layer的图像改变了，最后也会调用SurfaceFlinger的signalLayerUpdate函数。
SurfaceFlinger::signalLayerUpdate是调用了MessageQueue的invalidate函数
最后处理还是调用了SurfaceFlinger的onMessageReceived函数。看看SurfaceFlinger的onMessageReceived函数对NVALIDATE的处理
handleMessageInvalidate函数中调用了handlePageFlip函数，这个函数将会处理Layer中的缓冲区，把更新过的图像缓冲区切换到前台，等待VSync信号更新到FrameBuffer。

#### 1.2、绘制流程

用户进程更新Surface图像，将导致SurfaceFlinger中的Layer发送invalidate消息，处理该消息会调用handleTransaction函数和handlePageFilp函数来更新Layer对象。一旦VSync信号到来，再调用rebuildlayerStacks setUpHWComposer doComposition postComposition函数将所有Layer的图像混合后更新到显示设备上去。

##### 二、handleTransaction handPageFlip更新Layer对象
在上一节中的绘图的流程中，我们看到了handleTransaction和handPageFlip这两个函数通常是在用户进程更新Surface图像时会调用，来更新Layer对象。这节就主要讲解这两个函数。

#### 2.1、handleTransaction函数
handleTransaction函数的参数是transactionFlags，不过函数中没有使用这个参数，而是通过getTransactionFlags(eTransactionMask)来重新对transactionFlags赋值，然后使用它作为参数来调用函数
handleTransactionLocked。
[SurfaceFlinger.cpp]

``` cpp
void SurfaceFlinger::handleTransaction(uint32_t transactionFlags)
{
ATRACE_CALL();

Mutex::Autolock _l(mStateLock);
const nsecs_t now = systemTime();
mDebugInTransaction = now;

transactionFlags = getTransactionFlags(eTransactionMask);
handleTransactionLocked(transactionFlags);

mLastTransactionTime = systemTime() - now;
mDebugInTransaction = 0;
invalidateHwcGeometry();
}
```

getTransactionFlags函数的参数是eTransactionMask只是屏蔽其他位。
handleTransactionLocked函数会调用每个Layer类的doTransaction函数，在分析handleTransactionLocked函数之前，我们先看看Layer类 的doTransaction函数。
#### 2.2、Layer的doTransaction函数
下面是Layer的doTransaction函数代码
[Layer.cpp]

``` java
uint32_t Layer::doTransaction(uint32_t flags) {
ATRACE_CALL();

pushPendingState();//上次绘制的State对象  
Layer::State c = getCurrentState();//当前使用的State对象 

const Layer::State& s(getDrawingState());

const bool sizeChanged = (c.requested.w != s.requested.w) ||
                         (c.requested.h != s.requested.h);

if (sizeChanged) {
    // the size changed, we need to ask our client to request a new buffer
    //如果Layer的尺寸发生变化，就要改变Surface的缓冲区的尺寸
    // record the new size, form this point on, when the client request
    // a buffer, it'll get the new size.
    mSurfaceFlingerConsumer->setDefaultBufferSize(
            c.requested.w, c.requested.h);
}

const bool resizePending = (c.requested.w != c.active.w) ||
        (c.requested.h != c.active.h);
if (!isFixedSize()) {
    if (resizePending && mSidebandStream == NULL) {
    //如果Layer不是固定尺寸的类型，比较它的实际大小和要求的改变大小  
        flags |= eDontUpdateGeometryState;
    }
}
//如果没有eDontUpdateGeometryState标志，更新active的值为request  
if (flags & eDontUpdateGeometryState)  {
} else {
    Layer::State& editCurrentState(getCurrentState());
    if (mFreezePositionUpdates) {
        float tx = c.active.transform.tx();
        float ty = c.active.transform.ty();
        c.active = c.requested;
        c.active.transform.set(tx, ty);
        editCurrentState.active = c.active;
    } else {
        editCurrentState.active = editCurrentState.requested;
        c.active = c.requested;
    }
}
// 如果当前state的active和以前的State的active不等，设置更新标志  
if (s.active != c.active) {
    // invalidate and recompute the visible regions if needed
    flags |= Layer::eVisibleRegion;
}
//如果当前state的sequence和以前state的sequence不等，设置更新标志
if (c.sequence != s.sequence) {
    // invalidate and recompute the visible regions if needed
    flags |= eVisibleRegion;
    this->contentDirty = true;

    // we may use linear filtering, if the matrix scales us
    const uint8_t type = c.active.transform.getType();
    mNeedsFiltering = (!c.active.transform.preserveRects() ||
            (type >= Transform::SCALE));
}

// If the layer is hidden, signal and clear out all local sync points so
// that transactions for layers depending on this layer's frames becoming
// visible are not blocked
if (c.flags & layer_state_t::eLayerHidden) {
    Mutex::Autolock lock(mLocalSyncPointMutex);
    for (auto& point : mLocalSyncPoints) {
        point->setFrameAvailable();
    }
    mLocalSyncPoints.clear();
}

// Commit the transaction
commitTransaction(c);
return flags;
}
```

Layer类中的两个类型为Layer::State的成员变量mDrawingState、mCurrentState，这里为什么要两个对象呢？Layer对象在绘制图形时，使用的是mDrawingState变量，用户调用接口设置Layer对象属性是，设置的值保存在mCurrentState对象中，这样就不会因为用户的操作而干扰Layer对象的绘制了。
Layer的doTransaction函数据你是比较这两个变量，如果有不同的地方，说明在上次绘制以后，用户改变的Layer的设置，要把这种变化通过flags返回。
State的结构中有两个Geometry字段，active和requested。他们表示layer的尺寸，其中requested保存是用户设置的尺寸，而active保存的值通过计算后的实际尺寸。
State中的z字段的值就是Layer在显示轴的位置，值越小位置越靠下。
layerStack字段是用户指定的一个值，用户可以给DisplayDevice也指定一个layerStack值，只有Layer对象和DisplayDevice对象的layerStack相等，这个Layer才能在这个显示设备上输出，这样的好处是可以让显示设备只显示某个Surface的内容。例如，可以让HDMI显示设备只显示手机上播放视频的Surface窗口，但不显示Activity窗口。
sequence字段是个序列值，每当用户调用了Layer的接口，例如setAlpha、setSize或者setLayer等改变Layer对象属性的哈数，这个值都会加1。因此在doTransaction函数中能通过比较sequence值来判断Layer的属性值有没有变化。
doTransaction函数最后会调用commitTransaction函数，就是把mCurrentState赋值给mDrawingState
[Layer.cpp]

``` cpp
void Layer::commitTransaction(const State& stateToCommit) {
mDrawingState = stateToCommit;
}
```
##### 2.3、handleTransactionLocked函数
下面我们来分析handleTransactionLocked函数，这个函数比较长，我们分段分析

2.3.1 处理Layer的事务
[SurfaceFlinger.cpp]

``` cpp
void SurfaceFlinger::handleTransactionLocked(uint32_t transactionFlags)
{
const LayerVector& currentLayers(mCurrentState.layersSortedByZ);
const size_t count = currentLayers.size();

// Notify all layers of available frames
for (size_t i = 0; i < count; ++i) {
    currentLayers[i]->notifyAvailableFrames();
}

if (transactionFlags & eTraversalNeeded) {
    for (size_t i=0 ; i<count ; i++) {
        const sp<Layer>& layer(currentLayers[i]);
        uint32_t trFlags = layer->getTransactionFlags(eTransactionNeeded);
        if (!trFlags) continue;

        const uint32_t flags = layer->doTransaction(0);
        if (flags & Layer::eVisibleRegion)
            mVisibleRegionsDirty = true;
    }
}
```

在SurfaceFlinger中也有两个类型为State的变量mCurrentState和mDrawingState，但是和Layer中的不要混起来。它的名字相同而已

``` cpp
    struct State {
    LayerVector layersSortedByZ;
    DefaultKeyedVector< wp<IBinder>, DisplayDeviceState> displays;
};
```

结构layersSortedByZ字段保存所有参与绘制的Layer对象，而字段displays保存的是所有输出设备的DisplayDeviceState对象
这里用两个变量的目的是和Layer中使用两个变量是一样的。
上面代码根据eTraversalNeeded标志来决定是否要检查所有的Layer对象。如果某个Layer对象中有eTransactionNeeded标志，将调用它的doTransaction函数。Layer的doTransaction函数返回的flags如果有eVisibleRegion，说明这个Layer需要更新，就把mVisibleRegionsDirty设置为true

##### 2.3.2、处理显示设备的变化

``` cpp
    if (transactionFlags & eDisplayTransactionNeeded) {
    // here we take advantage of Vector's copy-on-write semantics to
    // improve performance by skipping the transaction entirely when
    // know that the lists are identical
    const KeyedVector<  wp<IBinder>, DisplayDeviceState>& curr(mCurrentState.displays);
    const KeyedVector<  wp<IBinder>, DisplayDeviceState>& draw(mDrawingState.displays);
    if (!curr.isIdenticalTo(draw)) {
        mVisibleRegionsDirty = true;
        const size_t cc = curr.size();
              size_t dc = draw.size();

        // find the displays that were removed
        // (ie: in drawing state but not in current state)
        // also handle displays that changed
        // (ie: displays that are in both lists)
        for (size_t i=0 ; i<dc ; i++) {
            const ssize_t j = curr.indexOfKey(draw.keyAt(i));
            if (j < 0) {
                // in drawing state but not in current state
                if (!draw[i].isMainDisplay()) {
                    // Call makeCurrent() on the primary display so we can
                    // be sure that nothing associated with this display
                    // is current.
                    const sp<const DisplayDevice> defaultDisplay(getDefaultDisplayDevice());
                    defaultDisplay->makeCurrent(mEGLDisplay, mEGLContext);
                    sp<DisplayDevice> hw(getDisplayDevice(draw.keyAt(i)));
                    if (hw != NULL)
                        hw->disconnect(getHwComposer());
                    if (draw[i].type < DisplayDevice::NUM_BUILTIN_DISPLAY_TYPES)
                        mEventThread->onHotplugReceived(draw[i].type, false);
                    mDisplays.removeItem(draw.keyAt(i));
                } else {
                    ALOGW("trying to remove the main display");
                }
            } else {
                // this display is in both lists. see if something changed.
                const DisplayDeviceState& state(curr[j]);
                const wp<IBinder>& display(curr.keyAt(j));
                const sp<IBinder> state_binder = IInterface::asBinder(state.surface);
                const sp<IBinder> draw_binder = IInterface::asBinder(draw[i].surface);
                if (state_binder != draw_binder) {
                    // changing the surface is like destroying and
                    // recreating the DisplayDevice, so we just remove it
                    // from the drawing state, so that it get re-added
                    // below.
                    sp<DisplayDevice> hw(getDisplayDevice(display));
                    if (hw != NULL)
                        hw->disconnect(getHwComposer());
                    mDisplays.removeItem(display);
                    mDrawingState.displays.removeItemsAt(i);
                    dc--; i--;
                    // at this point we must loop to the next item
                    continue;
                }

                const sp<DisplayDevice> disp(getDisplayDevice(display));
                if (disp != NULL) {
                    if (state.layerStack != draw[i].layerStack) {
                        disp->setLayerStack(state.layerStack);
                    }
                    if ((state.orientation != draw[i].orientation)
                            || (state.viewport != draw[i].viewport)
                            || (state.frame != draw[i].frame))
                    {
                        disp->setProjection(state.orientation,
                                state.viewport, state.frame);
                    }
                    if (state.width != draw[i].width || state.height != draw[i].height) {
                        disp->setDisplaySize(state.width, state.height);
                    }
                }
            }
        }

        // find displays that were added
        // (ie: in current state but not in drawing state)
        for (size_t i=0 ; i<cc ; i++) {
            if (draw.indexOfKey(curr.keyAt(i)) < 0) {
                const DisplayDeviceState& state(curr[i]);

                sp<DisplaySurface> dispSurface;
                sp<IGraphicBufferProducer> producer;
                sp<IGraphicBufferProducer> bqProducer;
                sp<IGraphicBufferConsumer> bqConsumer;
                BufferQueue::createBufferQueue(&bqProducer, &bqConsumer,
                        new GraphicBufferAlloc());

                int32_t hwcDisplayId = -1;
                if (state.isVirtualDisplay()) {
                    // Virtual displays without a surface are dormant:
                    // they have external state (layer stack, projection,
                    // etc.) but no internal state (i.e. a DisplayDevice).
                    if (state.surface != NULL) {

                        int width = 0;
                        DisplayUtils* displayUtils = DisplayUtils::getInstance();
                        int status = state.surface->query(
                                NATIVE_WINDOW_WIDTH, &width);
                        ALOGE_IF(status != NO_ERROR,
                                "Unable to query width (%d)", status);
                        int height = 0;
                        status = state.surface->query(
                            NATIVE_WINDOW_HEIGHT, &height);
                        ALOGE_IF(status != NO_ERROR,
                            "Unable to query height (%d)", status);
                        if (MAX_VIRTUAL_DISPLAY_DIMENSION == 0 ||
                            (width <= MAX_VIRTUAL_DISPLAY_DIMENSION &&
                             height <= MAX_VIRTUAL_DISPLAY_DIMENSION)) {
                            int usage = 0;
                            status = state.surface->query(
                                NATIVE_WINDOW_CONSUMER_USAGE_BITS, &usage);
                            ALOGW_IF(status != NO_ERROR,
                                "Unable to query usage (%d)", status);
                            if ( (status == NO_ERROR) &&
                                  displayUtils->canAllocateHwcDisplayIdForVDS(usage)) {
                                hwcDisplayId = allocateHwcDisplayId(state.type);
                            }
                        }

                        displayUtils->initVDSInstance(mHwc, hwcDisplayId, state.surface,
                                dispSurface, producer, bqProducer, bqConsumer,
                                state.displayName, state.isSecure, state.type);
                    }
                } else {
                    ALOGE_IF(state.surface!=NULL,
                            "adding a supported display, but rendering "
                            "surface is provided (%p), ignoring it",
                            state.surface.get());
                    hwcDisplayId = allocateHwcDisplayId(state.type);
                    // for supported (by hwc) displays we provide our
                    // own rendering surface
                    dispSurface = new FramebufferSurface(*mHwc, state.type,
                            bqConsumer);
                    producer = bqProducer;
                }

                const wp<IBinder>& display(curr.keyAt(i));
                if (dispSurface != NULL && producer != NULL) {
                    sp<DisplayDevice> hw = new DisplayDevice(this,
                            state.type, hwcDisplayId,
                            mHwc->getFormat(hwcDisplayId), state.isSecure,
                            display, dispSurface, producer,
                            mRenderEngine->getEGLConfig());
                    hw->setLayerStack(state.layerStack);
                    hw->setProjection(state.orientation,
                            state.viewport, state.frame);
                    hw->setDisplayName(state.displayName);
                    // When a new display device is added update the active
                    // config by querying HWC otherwise the default config
                    // (config 0) will be used.
                    if (hwcDisplayId >= DisplayDevice::DISPLAY_PRIMARY &&
                            hwcDisplayId < DisplayDevice::NUM_BUILTIN_DISPLAY_TYPES) {
                        int activeConfig = mHwc->getActiveConfig(hwcDisplayId);
                        if (activeConfig >= 0) {
                            hw->setActiveConfig(activeConfig);
                        }
                    }
                    mDisplays.add(display, hw);
                    if (state.isVirtualDisplay()) {
                        if (hwcDisplayId >= 0) {
                            mHwc->setVirtualDisplayProperties(hwcDisplayId,
                                    hw->getWidth(), hw->getHeight(),
                                    hw->getFormat());
                        }
                    } else {
                        mEventThread->onHotplugReceived(state.type, true);
                    }
                }
            }
        }
    }
}
```

这段代码的作用是处理显示设备的变化，分成3种情况：
1.显示设备减少了，需要把显示设备对应的DisplayDevice移除
2.显示设备发生了变化，例如用户设置了Surface、重新设置了layerStack、旋转了屏幕等，这就需要重新设置显示对象的属性
3.显示设备增加了，创建新的DisplayDevice加入系统中。

##### 2.3.3、设置TransfromHit

``` cpp
    if (transactionFlags & (eTraversalNeeded|eDisplayTransactionNeeded)) {
    ......
    sp<const DisplayDevice> disp;
    uint32_t currentlayerStack = 0;
    for (size_t i=0; i<count; i++) {
        // NOTE: we rely on the fact that layers are sorted by
        // layerStack first (so we don't have to traverse the list
        // of displays for every layer).
        const sp<Layer>& layer(currentLayers[i]);
        uint32_t layerStack = layer->getDrawingState().layerStack;
        if (i==0 || currentlayerStack != layerStack) {
            currentlayerStack = layerStack;
            // figure out if this layerstack is mirrored
            // (more than one display) if so, pick the default display,
            // if not, pick the only display it's on.
            disp.clear();
            for (size_t dpy=0 ; dpy<mDisplays.size() ; dpy++) {
                sp<const DisplayDevice> hw(mDisplays[dpy]);
                if (hw->getLayerStack() == currentlayerStack) {
                    if (disp == NULL) {
                        disp = hw;
                    } else {
                        disp = NULL;
                        break;
                    }
                }
            }
        }
        if (disp == NULL) {
            // NOTE: TEMPORARY FIX ONLY. Real fix should cause layers to
            // redraw after transform hint changes. See bug 8508397.

            // could be null when this layer is using a layerStack
            // that is not visible on any display. Also can occur at
            // screen off/on times.
            disp = getDefaultDisplayDevice();
        }
        layer->updateTransformHint(disp);
    }
}
```

这段代码的作用是根据每种显示设备的不同，设置和显示设备关联在一起的Layer（主要看Layer的layerStack是否和DisplayDevice的layerStack）的TransformHint（主要指设备的显示方向orientation）。

##### 2.3.4、处理Layer增加情况

``` cpp
/*
 * Perform our own transaction if needed
 */

const LayerVector& layers(mDrawingState.layersSortedByZ);
if (currentLayers.size() > layers.size()) {
    // layers have been added
    mVisibleRegionsDirty = true;
}

// some layers might have been removed, so
// we need to update the regions they're exposing.
if (mLayersRemoved) {
    mLayersRemoved = false;
    mVisibleRegionsDirty = true;
    const size_t count = layers.size();
    for (size_t i=0 ; i<count ; i++) {
        const sp<Layer>& layer(layers[i]);
        if (currentLayers.indexOf(layer) < 0) {
            // this layer is not visible anymore
            // TODO: we could traverse the tree from front to back and
            //       compute the actual visible region
            // TODO: we could cache the transformed region
            const Layer::State& s(layer->getDrawingState());
            Region visibleReg = s.active.transform.transform(
                    Region(Rect(s.active.w, s.active.h)));
            invalidateLayerStack(s.layerStack, visibleReg);
        }
    }
}
```

这段代码处理Layer的增加情况，如果Layer增加了，需要重新计算设备的更新区域，因此把mVisibleRegionsDirty设为true，如果Layer删除了，需要把Layer的可见区域加入到系统需要更新的区域中。
##### 2.3.5、设置mDrawingState

``` cpp
commitTransaction();
updateCursorAsync();
```

调用commitTransaction和updateCursorAsync函数 commitTransaction函数作用是把mDrawingState的值设置成mCurrentState的值。而updateCursorAsync函数会更新所有显示设备中光标的位置。

##### 2.3.6 小结
handleTransaction函数的作用的就是处理系统在两次刷新期间的各种变化。SurfaceFlinger模块中不管是SurfaceFlinger类还是Layer类，都采用了双缓冲的方式来保存他们的属性，这样的好处是刚改变SurfaceFlinger对象或者Layer类对象的属性是，不需要上锁，大大的提高了系统效率。只有在最后的图像输出是，才进行一次上锁，并进行内存的属性变化处理。正因此，应用进程必须收到VSync信号才开始改变Surface的内容。

#### 2.4、handlePageFlip函数
handlePageFlip函数代码如下：
[SurfaceFlinger.cpp]

``` cpp
bool SurfaceFlinger::handlePageFlip()
{
Region dirtyRegion;

bool visibleRegions = false;
const LayerVector& layers(mDrawingState.layersSortedByZ);
bool frameQueued = false;

// Store the set of layers that need updates. This set must not change as
// buffers are being latched, as this could result in a deadlock.
// Example: Two producers share the same command stream and:
// 1.) Layer 0 is latched
// 2.) Layer 0 gets a new frame
// 2.) Layer 1 gets a new frame
// 3.) Layer 1 is latched.
// Display is now waiting on Layer 1's frame, which is behind layer 0's
// second frame. But layer 0's second frame could be waiting on display.
Vector<Layer*> layersWithQueuedFrames;
for (size_t i = 0, count = layers.size(); i<count ; i++) {
    const sp<Layer>& layer(layers[i]);
    if (layer->hasQueuedFrame()) {
        frameQueued = true;
        if (layer->shouldPresentNow(mPrimaryDispSync)) {
            layersWithQueuedFrames.push_back(layer.get());
        } else {
            layer->useEmptyDamage();
        }
    } else {
        layer->useEmptyDamage();
    }
}
for (size_t i = 0, count = layersWithQueuedFrames.size() ; i<count ; i++) {
    Layer* layer = layersWithQueuedFrames[i];
    const Region dirty(layer->latchBuffer(visibleRegions));
    layer->useSurfaceDamage();
    const Layer::State& s(layer->getDrawingState());
    invalidateLayerStack(s.layerStack, dirty);
}

mVisibleRegionsDirty |= visibleRegions;

// If we will need to wake up at some time in the future to deal with a
// queued frame that shouldn't be displayed during this vsync period, wake
// up during the next vsync period to check again.
if (frameQueued && layersWithQueuedFrames.empty()) {
    signalLayerUpdate();
}

// Only continue with the refresh if there is actually new work to do
return !layersWithQueuedFrames.empty();
}
```

handlePageFlip函数先调用每个Layer对象的hasQueuedFrame函数，确定这个Layer对象是否有需要更新的图层，然后把需要更新的Layer对象放到layersWithQueuedFrames中。
我们先来看Layer的hasQueuedFrame方法就是看其mQueuedFrames是否大于0 和mSidebandStreamChanged。前面小节分析只要Surface有数据写入，就会调用Layer的onFrameAvailable函数，然后mQueuedFrames值加1.
继续看handlePageFlip函数，接着调用需要更新的Layer对象的latchBuffer函数，然后根据返回的更新区域调用invalidateLayerStack函数来设置更新设备对象的更新区域。
下面我们看看latchBuffer函数

LatchBuffer函数调用updateTextImage来得到需要的图像。这里参数r是Reject对象，其作用是判断在缓冲区的尺寸是否符合要求。调用updateTextImage函数如果得到的结果是PRESENT_LATER,表示推迟处理，然后调用signalLayerUpdate函数来发送invalidate消息，这次绘制过程就不处理这个Surface的图像了。
如果不需要推迟处理，把mQueuedFrames的值减1.
最后LatchBuffer函数调用mSurfaceFlingerConsumer的getCurrentBuffer来取回当前的图像缓冲区指针，保存在mActiveBuffer中。

##### 2.5 小结
这样经过handleTransaction handlePageFlip两个函数处理，SurfaceFlinger中无论是Layer属性的变化还是图像的变化都处理好了，只等VSync信号到来就可以输出了。


##### 三、rebuildLayerStacks函数
前面介绍，VSync信号到来后，先是调用了rebuildLayerStacks函数


``` cpp
void SurfaceFlinger::rebuildLayerStacks() {
updateExtendedMode();
// rebuild the visible layer list per screen
if (CC_UNLIKELY(mVisibleRegionsDirty)) {
    ATRACE_CALL();
    mVisibleRegionsDirty = false;
    invalidateHwcGeometry();
    //计算每个显示设备上可见的Layer  
    const LayerVector& layers(mDrawingState.layersSortedByZ);
    for (size_t dpy=0 ; dpy<mDisplays.size() ; dpy++) {
        Region opaqueRegion;
        Region dirtyRegion;
        Vector< sp<Layer> > layersSortedByZ;
        const sp<DisplayDevice>& hw(mDisplays[dpy]);
        const Transform& tr(hw->getTransform());
        const Rect bounds(hw->getBounds());
        if (hw->isDisplayOn()) {
            //计算每个layer的可见区域，确定设备需要重新绘制的区域  
            computeVisibleRegions(hw->getHwcDisplayId(), layers,
                    hw->getLayerStack(), dirtyRegion, opaqueRegion);

            const size_t count = layers.size();
            for (size_t i=0 ; i<count ; i++) {
                const sp<Layer>& layer(layers[i]);
                {
                    //只需要和显示设备的LayerStack相同的layer  
                    Region drawRegion(tr.transform(
                            layer->visibleNonTransparentRegion));
                    drawRegion.andSelf(bounds);
                    if (!drawRegion.isEmpty()) {
                    //如果Layer的显示区域和显示设备的窗口有交集  
                        //把Layer加入列表中 
                        layersSortedByZ.add(layer);
                    }
                }
            }
        }
        //设置显示设备的可见Layer列表  
        hw->setVisibleLayersSortedByZ(layersSortedByZ);
        hw->undefinedRegion.set(bounds);
        hw->undefinedRegion.subtractSelf(tr.transform(opaqueRegion));
        hw->dirtyRegion.orSelf(dirtyRegion);
    }
}
}
```

rebuildLayerStacks函数的作用是重建每个显示设备的可见layer对象列表。对于按显示轴（Z轴）排列的Layer对象，排在最前面的当然会优先显示，但是Layer图像可能有透明域，也可能有尺寸没有覆盖整个屏幕，因此下面的layer也有显示的机会。rebuildLayerStacks函数对每个显示设备，先计算和显示设备具有相同layerStack值的Layer对象在该显示设备上的可见区域。然后将可见区域和显示设备的窗口区域有交集的layer组成一个新的列表，最后把这个列表设置到显示设备对象中。
computeVisibleRegions函数首先计算每个Layer在设备上的可见区域visibleRegion。计算方法就是用整个Layer的区域减去上层所有不透明区域aboveOpaqueLayers。而上层所有不透明区域值是一个逐层累计的过程，每层都需要把自己的不透明区域累加到aboveOpaqueLayers中。
而每层的不透明区域的计算方法：如果Layer的alpha的值为255，并且layer的isOpaque函数为true，则本层的不透明区域等于Layer所在区域，否则为0.这样一层层算下来，就很容易得到每层的可见区域大小了。
其次，计算整个显示设备需要更新的区域outDirtyRegion。outDirtyRegion的值也是累计所有层的需要重回的区域得到的。如果Layer中的显示内容发生了变化，则整个可见区域visibleRegion都需要更新，同时还要包括上一次的可见区域，然后在去掉被上层覆盖后的区域得到的就是Layer需要更新的区域。如果Layer显示的内容没有变化，但是考虑到窗口大小的变化或者上层窗口的变化，因此Layer中还是有区域可以需要重绘的地方。这种情况下最简单的算法是用Layer计算出可见区域减去以前的可见区域就可以了。但是在computeVisibleRegions函数还引入了被覆盖区域，通常被覆盖区域和可见区域并不重复，因此函数中计算暴露区域是用可见区域减去被覆盖区域的。

##### 四、setUpHWComposer函数
setUpHWComposer函数的作用是更新HWComposer对象中图层对象列表以及图层属性。
[SurfaceFlinger.cpp]

``` cpp
void SurfaceFlinger::setUpHWComposer() {
for (size_t dpy=0 ; dpy<mDisplays.size() ; dpy++) {
    bool dirty = !mDisplays[dpy]->getDirtyRegion(false).isEmpty();
    bool empty = mDisplays[dpy]->getVisibleLayersSortedByZ().size() == 0;
    bool wasEmpty = !mDisplays[dpy]->lastCompositionHadVisibleLayers;

    ......
    bool mustRecompose = dirty && !(empty && wasEmpty);

    ......

    mDisplays[dpy]->beginFrame(mustRecompose);

    if (mustRecompose) {
        mDisplays[dpy]->lastCompositionHadVisibleLayers = !empty;
    }
}
//得到系统HWComposer对象  
HWComposer& hwc(getHwComposer());
if (hwc.initCheck() == NO_ERROR) {
    // build the h/w work list
    if (CC_UNLIKELY(mHwWorkListDirty)) {
        mHwWorkListDirty = false;
        for (size_t dpy=0 ; dpy<mDisplays.size() ; dpy++) {
            sp<const DisplayDevice> hw(mDisplays[dpy]);
            const int32_t id = hw->getHwcDisplayId();
            if (id >= 0) {
                const Vector< sp<Layer> >& currentLayers(
                    hw->getVisibleLayersSortedByZ());
                const size_t count = currentLayers.size();
                //根据Layer数量在HWComposer中创建hwc_layer_list_t列表  
                if (hwc.createWorkList(id, count) == NO_ERROR) {
                    HWComposer::LayerListIterator cur = hwc.begin(id);
                    const HWComposer::LayerListIterator end = hwc.end(id);
                    for (size_t i=0 ; cur!=end && i<count ; ++i, ++cur) {
                        const sp<Layer>& layer(currentLayers[i]);
                        layer->setGeometry(hw, *cur);
                        if (mDebugDisableHWC || mDebugRegion || mDaltonize || mHasColorMatrix) {
                            cur->setSkip(true);
                        }
                    }
                }
            }
        }
    }

    // set the per-frame data
    for (size_t dpy=0 ; dpy<mDisplays.size() ; dpy++) {
        sp<const DisplayDevice> hw(mDisplays[dpy]);
        const int32_t id = hw->getHwcDisplayId();
        if (id >= 0) {
            bool freezeSurfacePresent = false;
            isfreezeSurfacePresent(freezeSurfacePresent, hw, id);
            const Vector< sp<Layer> >& currentLayers(
                hw->getVisibleLayersSortedByZ());
            const size_t count = currentLayers.size();
            HWComposer::LayerListIterator cur = hwc.begin(id);
            const HWComposer::LayerListIterator end = hwc.end(id);
            for (size_t i=0 ; cur!=end && i<count ; ++i, ++cur) {
                /*
                 * update the per-frame h/w composer data for each layer
                 * and build the transparent region of the FB
                 */
                const sp<Layer>& layer(currentLayers[i]);
                //将Layer的mActiveBuffer设置到HWComposer中 
                layer->setPerFrameData(hw, *cur);
                setOrientationEventControl(freezeSurfacePresent,id);
            }
        }
    }

    // If possible, attempt to use the cursor overlay on each display.
    for (size_t dpy=0 ; dpy<mDisplays.size() ; dpy++) {
        sp<const DisplayDevice> hw(mDisplays[dpy]);
        const int32_t id = hw->getHwcDisplayId();
        if (id >= 0) {
            const Vector< sp<Layer> >& currentLayers(
                hw->getVisibleLayersSortedByZ());
            const size_t count = currentLayers.size();
            HWComposer::LayerListIterator cur = hwc.begin(id);
            const HWComposer::LayerListIterator end = hwc.end(id);
            for (size_t i=0 ; cur!=end && i<count ; ++i, ++cur) {
                const sp<Layer>& layer(currentLayers[i]);
                if (layer->isPotentialCursor()) {
                    cur->setIsCursorLayerHint();
                    break;
                }
            }
        }
    }

    dumpDrawCycle(true);

    status_t err = hwc.prepare();
    ALOGE_IF(err, "HWComposer::prepare failed (%s)", strerror(-err));

    for (size_t dpy=0 ; dpy<mDisplays.size() ; dpy++) {
        sp<const DisplayDevice> hw(mDisplays[dpy]);
        hw->prepareFrame(hwc);
    }
}
}
```

HWComposer中有一个类型为DisplayData结构的数组mDisplayData，它维护着每个显示设备的信息。DisplayData结构中有一个类型为hwc_display_contents_l字段list，这个字段又有一个hwc_layer_l类型的数组hwLayers，记录该显示设备所有需要输出的Layer信息。
setUpHWComposer函数调用HWComposer的createWorkList函数就是根据每种显示设备的Layer数量，创建和初始化hwc_display_contents_l对象和hwc_layer_l数组
创建完HWComposer中的列表后，接下来是对每个Layer对象调用它的setPerFrameData函数，参数是HWComposer和HWCLayerInterface。setPerFrameData函数将Layer对象的当前图像缓冲区mActiveBuffer设置到HWCLayerInterface对象对应的hwc_layer_l对象中。
HWComposer类中除了前面介绍的Gralloc还管理着Composer模块，这个模块实现了硬件的图像合成功能。setUpHWComposer函数接下来调用HWComposer类的prepare函数，而prepare函数会调用Composer模块的prepare接口。最后到各个厂家的实现hwc_prepare函数将每种HWComposer中的所有图层的类型都设置为HWC_FRAMEBUFFER就结束了。
##### 五、合成所有层的图像 （doComposition()函数）
doComposition函数是合成所有层的图像，代码如下：
[SurfaceFlinger.cpp]

``` cpp
void SurfaceFlinger::doComposition() {
ATRACE_CALL();
const bool repaintEverything = android_atomic_and(0, &mRepaintEverything);
for (size_t dpy=0 ; dpy<mDisplays.size() ; dpy++) {
    const sp<DisplayDevice>& hw(mDisplays[dpy]);
    if (hw->isDisplayOn()) {
        // transform the dirty region into this screen's coordinate space
        const Region dirtyRegion(hw->getDirtyRegion(repaintEverything));

        // repaint the framebuffer (if needed)
        doDisplayComposition(hw, dirtyRegion);

        hw->dirtyRegion.clear();
        hw->flip(hw->swapRegion);
        hw->swapRegion.clear();
    }
    // inform the h/w that we're done compositing
    hw->compositionComplete();
}
postFramebuffer();
}
```

doComposition函数针对每种显示设备调用doDisplayComposition函数来合成，合成后调用postFramebuffer函数，我们先来看看doDisplayComposition函数

``` cpp
void SurfaceFlinger::doDisplayComposition(const sp<const DisplayDevice>& hw,
    const Region& inDirtyRegion)
    {
// We only need to actually compose the display if:
// 1) It is being handled by hardware composer, which may need this to
//    keep its virtual display state machine in sync, or
// 2) There is work to be done (the dirty region isn't empty)
bool isHwcDisplay = hw->getHwcDisplayId() >= 0;
if (!isHwcDisplay && inDirtyRegion.isEmpty()) {
    ALOGV("Skipping display composition");
    return;
}

ALOGV("doDisplayComposition");

Region dirtyRegion(inDirtyRegion);

// compute the invalid region
//swapRegion设置为需要更新的区域  
hw->swapRegion.orSelf(dirtyRegion);

uint32_t flags = hw->getFlags();//获得显示设备支持的更新方式标志  
if (flags & DisplayDevice::SWAP_RECTANGLE) {
    // we can redraw only what's dirty, but since SWAP_RECTANGLE only
    // takes a rectangle, we must make sure to update that whole
    // rectangle in that case
    dirtyRegion.set(hw->swapRegion.bounds());
} else {
    if (flags & DisplayDevice::PARTIAL_UPDATES) {//支持部分更新  
        // We need to redraw the rectangle that will be updated
        // (pushed to the framebuffer).
        // This is needed because PARTIAL_UPDATES only takes one
        // rectangle instead of a region (see DisplayDevice::flip())
        //将更新区域调整为整个窗口大小  
        dirtyRegion.set(hw->swapRegion.bounds());
    } else {
        // we need to redraw everything (the whole screen)
        dirtyRegion.set(hw->bounds());
        hw->swapRegion = dirtyRegion;
    }
}
//合成  
if (!doComposeSurfaces(hw, dirtyRegion)) return;

// update the swap region and clear the dirty region
hw->swapRegion.orSelf(dirtyRegion);
//没有硬件composer的情况，输出图像 
// swap buffers (presentation)
hw->swapBuffers(getHwComposer());
}
```

doDisplayComposition函数根据显示设备支持的更新方式，重新设置需要更新区域的大小。
真正的合成工作是在doComposerSurfaces函数中完成，这个函数在layer的类型为HWC_FRAMEBUFFER,或者不支持硬件的composer的情况下，调用layer的draw函数来一层一层低合成最后的图像。
合成完后，doDisplayComposition函数调用了hw的swapBuffers函数，这个函数前面介绍过了，它将在系统不支持硬件的composer情况下调用eglSwapBuffers来输出图像到显示设备。

![enter image description here](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/personal.website/zjj.display.sys.draw_whith_openGL_queuebuffer.png)



``` cpp
android-5.0.2\frameworks\native\services\surfaceflinger\DisplayHardware\FramebufferSurface.cpp

// Overrides ConsumerBase::onFrameAvailable(), does not call base class impl.
void FramebufferSurface::onFrameAvailable() {
    sp<GraphicBuffer> buf;
    sp<Fence> acquireFence;
    status_t err = nextBuffer(buf, acquireFence);
    if (err != NO_ERROR) {
        ALOGE("error latching nnext FramebufferSurface buffer: %s (%d)",
                strerror(-err), err);
        return;
    }
    err = mHwc.fbPost(mDisplayType, acquireFence, buf);
    if (err != NO_ERROR) {
        ALOGE("error posting framebuffer: %d", err);
    }
}

android-5.0.2\frameworks\native\services\surfaceflinger\DisplayHardware\HWComposer.cpp
int HWComposer::fbPost(int32_t id,
        const sp<Fence>& acquireFence, const sp<GraphicBuffer>& buffer) {
    if (mHwc && hwcHasApiVersion(mHwc, HWC_DEVICE_API_VERSION_1_1)) {
        return setFramebufferTarget(id, acquireFence, buffer);
    } else {
        acquireFence->waitForever("HWComposer::fbPost");
        return mFbDev->post(mFbDev, buffer->handle);
    }
}

```

> zjj.display.sys ioctl **FBIOPAN_DISPLAY** fb_post() framebuffer.cpp

``` cpp
zjj.display.sys ioctl FBIOPAN_DISPLAY fb_post() framebuffer.cpp

static int fb_post(struct framebuffer_device_t* dev, buffer_handle_t buffer)
{
    if (private_handle_t::validate(buffer) < 0)
        return -EINVAL;

    fb_context_t* ctx = (fb_context_t*)dev;

    private_handle_t const* hnd = reinterpret_cast<private_handle_t const*>(buffer);
    private_module_t* m = reinterpret_cast<private_module_t*>(
            dev->common.module);

    if (hnd->flags & private_handle_t::PRIV_FLAGS_FRAMEBUFFER) {
        const size_t offset = hnd->base - m->framebuffer->base;
        m->info.activate = FB_ACTIVATE_VBL;
        m->info.yoffset = offset / m->finfo.line_length;
#if 0

        if (ioctl(m->framebuffer->fd, FBIOPUT_VSCREENINFO, &m->info) == -1) {
            ALOGE("FBIOPUT_VSCREENINFO failed");
            m->base.unlock(&m->base, buffer); 
            return -errno;
        }
#else
       if (ioctl(m->framebuffer->fd, FBIOPAN_DISPLAY, &m->info) == -1) {
           ALOGE("FBIOPAN_DISPLAY failed");
           m->base.unlock(&m->base, buffer); 
           return -errno;
       }
#endif     

        m->currentBuffer = buffer;
        
    } 
    ......
    return 0;
}

```
LCD驱动前面已经分析过了，我们直接看看s3cfb_pan_display()函数
``` cpp
\linux-3.0.86\drivers\video\samsung\s3cfb_ops.c

int s3cfb_pan_display(struct fb_var_screeninfo *var, struct fb_info *fb)
{
	struct s3cfb_window *win = fb->par;
	struct s3cfb_global *fbdev = get_fimd_global(win->id);
	struct s3c_platform_fb *pdata = to_fb_plat(fbdev->dev);

	if (win->id == pdata->default_win)
		spin_lock(&fbdev->slock);

#ifdef CONFIG_EXYNOS_DEV_PD
	if (unlikely(fbdev->system_state == POWER_OFF) || fbdev->regs == 0) {
		dev_err(fbdev->dev, "%s::system_state is POWER_OFF, fb%d, win%d\n", __func__, fb->node, win->id);
		if (win->id == pdata->default_win)
			spin_unlock(&fbdev->slock);
		return -EINVAL;
	}
#endif

	if (var->yoffset + var->yres > var->yres_virtual) {
		dev_err(fbdev->dev, "invalid yoffset value\n");
		if (win->id == pdata->default_win)
			spin_unlock(&fbdev->slock);
		return -EINVAL;
	}

#if defined(CONFIG_CPU_EXYNOS4210)
	if (unlikely(var->xoffset + var->xres > var->xres_virtual)) {
		dev_err(fbdev->dev, "invalid xoffset value\n");
		if (win->id == pdata->default_win)
			spin_unlock(&fbdev->slock);
		return -EINVAL;
	}
	fb->var.xoffset = var->xoffset;
#endif

	fb->var.yoffset = var->yoffset;

	dev_dbg(fbdev->dev, "[fb%d] win%d: yoffset for pan display: %d\n",
		fb->node, win->id, var->yoffset);

	s3cfb_set_buffer_address(fbdev, win->id);

	if (win->id == pdata->default_win)
		spin_unlock(&fbdev->slock);
	return 0;
}

int s3cfb_set_buffer_address(struct s3cfb_global *ctrl, int id)
{
	struct fb_fix_screeninfo *fix = &ctrl->fb[id]->fix;
	struct fb_var_screeninfo *var = &ctrl->fb[id]->var;
	struct s3c_platform_fb *pdata = to_fb_plat(ctrl->dev);
	dma_addr_t start_addr = 0, end_addr = 0;
	u32 shw;

	if (fix->smem_start) {
		start_addr = fix->smem_start + ((var->xres_virtual *
				var->yoffset + var->xoffset) *
				(var->bits_per_pixel / 8));

		end_addr = start_addr + fix->line_length * var->yres;
	}

	if ((pdata->hw_ver == 0x62) || (pdata->hw_ver == 0x70)) {
		shw = readl(ctrl->regs + S3C_WINSHMAP);
		shw |= S3C_WINSHMAP_PROTECT(id);
		writel(shw, ctrl->regs + S3C_WINSHMAP);
	}

	writel(start_addr, ctrl->regs + S3C_VIDADDR_START0(id));
	writel(end_addr, ctrl->regs + S3C_VIDADDR_END0(id));

	if ((pdata->hw_ver == 0x62) || (pdata->hw_ver == 0x70)) {
		shw = readl(ctrl->regs + S3C_WINSHMAP);
		shw &= ~(S3C_WINSHMAP_PROTECT(id));
		writel(shw, ctrl->regs + S3C_WINSHMAP);
	}

	dev_dbg(ctrl->dev, "[win%d] start_addr: 0x%08x, end_addr: 0x%08x\n",
		id, start_addr, end_addr);

	return 0;
}
```
可以看到，spin_lock首先把framebuffer锁住，然后更新framebuffer的S3C_VIDADDR_START0 和 S3C_VIDADDR_END0，释放锁spin_unlock就将具体的画面数据刷新到LCD屏幕上了。

##### 六、postFramebuffer()函数
上一节的doComposition函数最后调用了postFramebuffer函数，代码如下：
[SurfaceFlinger.cpp]

``` cpp
void SurfaceFlinger::postFramebuffer()
{
ATRACE_CALL();

const nsecs_t now = systemTime();
mDebugInSwapBuffers = now;

HWComposer& hwc(getHwComposer());
if (hwc.initCheck() == NO_ERROR) {
    if (!hwc.supportsFramebufferTarget()) {
        // EGL spec says:
        //   "surface must be bound to the calling thread's current context,
        //    for the current rendering API."
        getDefaultDisplayDevice()->makeCurrent(mEGLDisplay, mEGLContext);
    }
    hwc.commit();
}

// make the default display current because the VirtualDisplayDevice code cannot
// deal with dequeueBuffer() being called outside of the composition loop; however
// the code below can call glFlush() which is allowed (and does in some case) call
// dequeueBuffer().
getDefaultDisplayDevice()->makeCurrent(mEGLDisplay, mEGLContext);

for (size_t dpy=0 ; dpy<mDisplays.size() ; dpy++) {
    sp<const DisplayDevice> hw(mDisplays[dpy]);
    const Vector< sp<Layer> >& currentLayers(hw->getVisibleLayersSortedByZ());
    hw->onSwapBuffersCompleted(hwc);
    const size_t count = currentLayers.size();
    int32_t id = hw->getHwcDisplayId();
    if (id >=0 && hwc.initCheck() == NO_ERROR) {
        HWComposer::LayerListIterator cur = hwc.begin(id);
        const HWComposer::LayerListIterator end = hwc.end(id);
        for (size_t i = 0; cur != end && i < count; ++i, ++cur) {
            currentLayers[i]->onLayerDisplayed(hw, &*cur);
        }
    } else {
        for (size_t i = 0; i < count; i++) {
            currentLayers[i]->onLayerDisplayed(hw, NULL);
        }
    }
}

mLastSwapBufferTime = systemTime() - now;
mDebugInSwapBuffers = 0;

uint32_t flipCount = getDefaultDisplayDevice()->getPageFlipCount();
if (flipCount % LOG_FRAME_STATS_PERIOD == 0) {
    logFrameStats();
}
}
```

postFramebuffer先判断系统是否支持composer，如果不支持，我们知道图像已经在doComposition函数时调用hw->swapBuffers输出了，就返回了。
#### （六）、Android SurfaceFlinger - VSync工作原理 


##### 一、VSYNC 总体概念

##### 6.1.1、VSYNC 概念
VSYNC（Vertical Synchronization）是一个相当古老的概念，对于游戏玩家，它有一个更加大名鼎鼎的中文名字—-垂直同步。
“垂直同步(vsync)”指的是显卡的输出帧数和屏幕的垂直刷新率相同，这完全是一个CRT显示器上的概念。其实无论是VSYNC还是垂直同步这个名字，因为LCD根本就没有垂直扫描的这种东西，因此这个名字本身已经没有意义。但是基于历史的原因，这个名称在图形图像领域被沿袭下来。
在当下，垂直同步的含义我们可以理解为，使得显卡生成帧的速度和屏幕刷新的速度的保持一致。举例来说，如果屏幕的刷新率为60Hz，那么生成帧的速度就应该被固定在1/60 s。
##### 6.1.2、Android VSYNC — 黄油计划
谷歌为解决Android系统流畅性问题。在4.1版本引入了一个重大的改进—Project Butter黄油计划。
Project Butter对Android Display系统进行了重构，引入了三个核心元素，即VSYNC、Triple Buffer和Choreographer。
VSYNC最重要的作用是防止出现画面撕裂（screentearing）。所谓画面撕裂，就是指一个画面上出现了两帧画面的内容，如下图。
![enter image description here](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/personal.website/zjj.display.sys.view-teaning.png)

为什么会出现这种情况呢？这种情况一般是因为显卡输出帧的速度高于显示器的刷新速度，导致显示器并不能及时处理输出的帧，而最终出现了多个帧的画面都留在了显示器上的问题。这也就是我们所说的画面撕裂。

![enter image description here](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/personal.website/zjj.display.sys.Draw-whithout-vsync.png)

这个图中有三个元素，Display是显示屏幕，GPU和CPU负责渲染帧数据，每个帧以方框表示，并以数字进行编号，如0、1、2等等。VSync用于指导双缓冲区的交换。
以时间的顺序来看下将会发生的异常：
Step1. Display显示第0帧数据，此时CPU和GPU渲染第1帧画面，而且赶在Display显示下一帧前完成
Step2. 因为渲染及时，Display在第0帧显示完成后，也就是第1个VSync后，正常显示第1帧
Step3. 由于某些原因，比如CPU资源被占用，系统没有及时地开始处理第2帧，直到第2个VSync快来前才开始处理
Step4. 第2个VSync来时，由于第2帧数据还没有准备就绪，显示的还是第1帧。这种情况被Android开发组命名为“Jank”。
Step5. 当第2帧数据准备完成后，它并不会马上被显示，而是要等待下一个VSync。
所以总的来说，就是屏幕平白无故地多显示了一次第1帧。原因大家应该都看到了，就是CPU没有及时地开始着手处理第2帧的渲染工作，以致“延误军机”。

其实总结上面的这个情况之所以发生，首先的原因就在于第二帧没有及时的绘制（当然即使第二帧及时绘制，也依然可能出现Jank，这就是同时引入三重缓冲的作用。我们将在三重缓冲一节中再讲解这种情况）。那么如何使得第二帧即使被绘制呢？
这就是我们在Graphic系统中引入VSYNC的原因，考虑下面这张图：
![enter image description here](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/personal.website/zjj.display.sys.Draw-whit-vsync.png)


如上图所示，一旦VSync出现后，立刻就开始执行下一帧的绘制工作。这样就可以大大降低Jank出现的概率。另外，VSYNC引入后，要求绘制也只能在收到VSYNC消息之后才能进行，因此，也就杜绝了另外一种极端情况的出现—-CPU（GPU）一直不停的进行绘制，帧的生成速度高于屏幕的刷新速度，导致生成的帧不能被显示，只能丢弃，这样就出现了丢帧的情况—-引入VSYNC后，绘制的速度就和屏幕刷新的速度保持一致了。
##### 二、VSync信号产生
那么VSYNC信号是如何生成的呢？
Android系统中VSYNC信号分为两种，一种是硬件生成的信号，一种是软件模拟的信号。
硬件信号是由HardwareComposer提供的，HWC封装了相关的HAL层，如果硬件厂商提供的HAL层实现能定时产生VSYNC中断，则直接使用硬件的VSYNC中断，否则HardwareComposer内部会通过VSyncThread来模拟产生VSYNC中断（其实现很简单，就是sleep固定时间，然后唤醒）。

SurfaceFlinger的启动过程中inti()会创建一个HWComposer对象。

``` cpp
HWComposer::HWComposer(  
        const sp<SurfaceFlinger>& flinger,  
        EventHandler& handler)  
    : mFlinger(flinger),  
      mFbDev(0), mHwc(0), mNumDisplays(1),  
      mCBContext(new cb_context),  
      mEventHandler(handler),  
      mDebugForceFakeVSync(false)  
{  
...  
    //首先是一些和VSYNC有关的信息的初始化  
    //因为在硬件支持的情况下，VSYNC的功能就是由HWC提供的  
    for (size_t i=0 ; i<HWC_NUM_PHYSICAL_DISPLAY_TYPES ; i++) {  
        mLastHwVSync[i] = 0;  
        mVSyncCounts[i] = 0;  
    }  
    //根据配置来看是否需要模拟VSYNC消息  
    char value[PROPERTY_VALUE_MAX];  
    property_get("debug.sf.no_hw_vsync", value, "0");  
    mDebugForceFakeVSync = atoi(value);  
    ...  
    // don't need a vsync thread if we have a hardware composer  
    needVSyncThread = false;  
    // always turn vsync off when we start,只是暂时关闭信号，后面会再开启  
    eventControl(HWC_DISPLAY_PRIMARY, HWC_EVENT_VSYNC, 0);      
  
    //显然，如果需要模拟VSync信号的话，我们需要线程来做这个工作  
    if (needVSyncThread) {  
        // we don't have VSYNC support, we need to fake it  
        //VSyncThread类的实现很简单，无非就是一个计时器而已，定时发送消息而已  
        //TODO VSYNC专题  
        mVSyncThread = new VSyncThread(*this);  
    }  
...  
}  
  
HWComposer::HWComposer(  
        const sp<SurfaceFlinger>& flinger,  
        EventHandler& handler)  
    : mFlinger(flinger),  
      mFbDev(0), mHwc(0), mNumDisplays(1),  
      mCBContext(new cb_context),  
      mEventHandler(handler),  
      mDebugForceFakeVSync(false)  
{  
...  
    //首先是一些和VSYNC有关的信息的初始化  
    //因为在硬件支持的情况下，VSYNC的功能就是由HWC提供的  
    for (size_t i=0 ; i<HWC_NUM_PHYSICAL_DISPLAY_TYPES ; i++) {  
        mLastHwVSync[i] = 0;  
        mVSyncCounts[i] = 0;  
    }  
    //根据配置来看是否需要模拟VSYNC消息  
    char value[PROPERTY_VALUE_MAX];  
    property_get("debug.sf.no_hw_vsync", value, "0");  
    mDebugForceFakeVSync = atoi(value);  
    ...  
    // don't need a vsync thread if we have a hardware composer  
    needVSyncThread = false;  
    // always turn vsync off when we start,只是暂时关闭信号，后面会再开启  
    eventControl(HWC_DISPLAY_PRIMARY, HWC_EVENT_VSYNC, 0);      
  
    //显然，如果需要模拟VSync信号的话，我们需要线程来做这个工作  
    if (needVSyncThread) {  
        // we don't have VSYNC support, we need to fake it  
        //VSyncThread类的实现很简单，无非就是一个计时器而已，定时发送消息而已  
        //TODO VSYNC专题  
        mVSyncThread = new VSyncThread(*this);  
    }  
...  
}  
```

我们来看下上面这段代码。
首先mDebugForceFakeVSync是为了调制，可以通过这个变量设置强制使用软件VSYNC模拟。
然后针对不同的屏幕，初始化了他们的mLastHwVSync和mVSyncCounts值。
如果硬件支持，那么就把needVSyncThread设置为false，表示不需要软件模拟。
接着通过eventControl来暂时的关闭了VSYNC信号，这一点将在下面讲解eventControl时一并讲解。
最后，如果需要软件模拟Vsync信号的话，那么我们将通过一个单独的VSyncThread线程来做这个工作(fake VSYNC是这个线程唯一的作用)。我们来看下这个线程。

**软件模拟**

``` cpp
bool HWComposer::VSyncThread::threadLoop() {  
    const nsecs_t period = mRefreshPeriod;  
    //当前的时间  
    const nsecs_t now = systemTime(CLOCK_MONOTONIC);  
    //下一次VSYNC到来的时间  
    nsecs_t next_vsync = mNextFakeVSync;  
    //为了等待下个时间到来应该休眠的时间  
    nsecs_t sleep = next_vsync - now;  
    //错过了VSYNC的时间  
    if (sleep < 0) {  
        // we missed, find where the next vsync should be  
        //重新计算下应该休息的时间  
        sleep = (period - ((now - next_vsync) % period));  
        //更新下次VSYNC的时间  
        next_vsync = now + sleep;  
    }  
    //更新下下次VSYNC的时间  
    mNextFakeVSync = next_vsync + period;  
  
    struct timespec spec;  
    spec.tv_sec  = next_vsync / 1000000000;  
    spec.tv_nsec = next_vsync % 1000000000;  
  
    int err;  
    do {  
        //纳秒精度级的休眠  
        err = clock_nanosleep(CLOCK_MONOTONIC, TIMER_ABSTIME, &spec, NULL);  
    } while (err<0 && errno == EINTR);  
  
    if (err == 0) {  
        //休眠之后，到了该发生VSYNC的时间了  
        mHwc.mEventHandler.onVSyncReceived(0, next_vsync);  
    }  
    return true;  
}  
```

这个函数其实很简单，无非就是一个简单的时间计算，计算过程我已经写在了程序注释里面。总之到了应该发生VSYNC信号的时候，就调用了mHwc.mEventHandler.onVSyncReceived(0, next_vsync)函数来通知VSYNC的到来。

我们注意到mEventHandler实际上是在HWC创建时被传入的，我们来看下HWC创建时的代码.

```
mHwc = new HWComposer(this,  
        *static_cast<HWComposer::EventHandler *>(this));  
        
class SurfaceFlinger : public BnSurfaceComposer,  
                   private IBinder::DeathRecipient,  
                   private HWComposer::EventHandler  
```

可以看到这个mEventHandler实际上就是SurfaceFlinger。也就是说，VSYNC信号到来时，SurfaceFlinger的onVSyncReceived函数处理了这个消息。
这里我们暂时先不展开SurfaceFlinger内的逻辑处理，等我们下面分析完硬件实现后，一并进行分析

**硬件实现**
上面我们讲了软件如何模拟一个VSYNC信号并通知SurfaceFlinger,那么硬件又是如何实现这一点的呢？
我们再一次回到HWC的创建过程中来：

```cpp
if (mHwc) {  
       ALOGE("Lee Using %s version %u.%u", HWC_HARDWARE_COMPOSER,  
             (hwcApiVersion(mHwc) >> 24) & 0xff,  
             (hwcApiVersion(mHwc) >> 16) & 0xff);  
       if (mHwc->registerProcs) {  
           mCBContext->hwc = this;  
           mCBContext->procs.invalidate = &hook_invalidate;  
           mCBContext->procs.vsync = &hook_vsync;  
           if (hwcHasApiVersion(mHwc, HWC_DEVICE_API_VERSION_1_1))  
               mCBContext->procs.hotplug = &hook_hotplug;  
           else  
               mCBContext->procs.hotplug = NULL;  
           memset(mCBContext->procs.zero, 0, sizeof(mCBContext->procs.zero));  
           mHwc->registerProcs(mHwc, &mCBContext->procs);  
       }  
```

来看下上面这段实现。
当HWC有vsync信号生成时，硬件模块会通过procs.vsync来通知软件部分，因此也就是调用了hook_vsync函数。

``` cpp
void HWComposer::hook_vsync(const struct hwc_procs* procs, int disp,  
        int64_t timestamp) {  
    cb_context* ctx = reinterpret_cast<cb_context*>(  
            const_cast<hwc_procs_t*>(procs));  
    ctx->hwc->vsync(disp, timestamp);  
}  
  
void HWComposer::vsync(int disp, int64_t timestamp) {  
    //只有真实的硬件设备才会产生VSYNC  
    if (uint32_t(disp) < HWC_NUM_PHYSICAL_DISPLAY_TYPES) {  
        {  
            mLastHwVSync[disp] = timestamp;  
        }  
        mEventHandler.onVSyncReceived(disp, timestamp);  
}  
```

我们发现最后殊途同归，硬件信号最终也通过onVSyncReceived函数通知到了SurfaceFlinger了。下面我们来分析下SurfaceFlinger的处理过程。
##### 三、Surfaceflinger对VSYNC消息的处理


先来直接看下Surfaceflinger的onVSyncReceived函数：

```cpp
void SurfaceFlinger::onVSyncReceived(int32_t type, nsecs_t timestamp) {
    bool needsHwVsync = false;

    { // Scope for the lock
        Mutex::Autolock _l(mHWVsyncLock);
        if (type == 0 && mPrimaryHWVsyncEnabled) {
            needsHwVsync = mPrimaryDispSync.addResyncSample(timestamp);
        }
    }

    if (needsHwVsync) {
        enableHardwareVsync();
    } else {
        disableHardwareVsync(false);
    }
}
```
mPrimaryDispSync是什么？addResyncSample有什么作用？
要回答这三个问题，我们首先还是得回到SurfaceFlinger的init函数中来。

##### 6.3.1、Surfaceflinger.init()
先看一下总体flow：
![enter image description here](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/personal.website/zjj.display.sys.Vsync-surfaceflinger.init.png)

```
void SurfaceFlinger::init() {
    ALOGI(  "SurfaceFlinger's main thread ready to run. "
            "Initializing graphics H/W...");

    { 
        ......
        // start the EventThread
        sp<VSyncSource> vsyncSrc = new DispSyncSource(&mPrimaryDispSync,
                vsyncPhaseOffsetNs, true, "app");
        mEventThread = new EventThread(vsyncSrc, *this);
        sp<VSyncSource> sfVsyncSrc = new DispSyncSource(&mPrimaryDispSync,
                sfVsyncPhaseOffsetNs, true, "sf");
        mSFEventThread = new EventThread(sfVsyncSrc, *this);
        mEventQueue.setEventThread(mSFEventThread);
        ......
    }
    ......
    mEventControlThread = new EventControlThread(this);
    mEventControlThread->run("EventControl", PRIORITY_URGENT_DISPLAY);
    ......
}
```
2个EventThread对象分别是mEventThread，给app用，mSFEventThread，给surfaceflinger自己用。
下面给出这4个Thread关系图。

![enter image description here](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/personal.website/zjj.display.sys.SurfaceFlinger.init.DispSyncThread.png)


这两个DispSyncSource就是KK引入的重大变化。Android 4.4(KitKat)引入了VSync的虚拟化，即把硬件的VSync信号先同步到一个本地VSync模型中，再从中一分为二，引出两条VSync时间与之有固定偏移的线程。示意图如下：
![enter image description here](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/personal.website/zjj.display.sys.SurfaceFlinger-App-Vsync-offset.png)

Google这样修改的目的又是什么呢？
=在当前三重缓冲区的架构下，即对于一帧内容，先等App UI画完了，SurfaceFlinger再出场对其进行合并渲染后放入framebuffer，最后整到屏幕上。而现有的VSync模型是让大家一起开始干活。
这个架构其实会产生一个问题，因为App和SurfaceFlinger被同时唤醒，导致他们二者总是一起工作，必然导致VSync来临的时刻，这二者之间产生了CPU资源的抢占。因此，谷歌给这两个工作都加上一个小小的延迟，让这两个工作并不是同时被唤醒，这样大家就可以错开使用资源的高峰期，提高工作的效率。
![enter image description here](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/personal.website/zjj.display.sys.SurfaceFlinger-Vsync-app-sf.png)

这两个延迟，其实就分别对应上面代码中的vsyncSrc（绘制延迟）和sfVsyncSrc（合成延迟）。
在创建了两个DispSyncSource变量后，我们使用它们来初始化了两个EventThread。下面我们来详细看下EventThread的创建流程：

``` cpp
EventThread::EventThread(const sp<VSyncSource>& src, SurfaceFlinger& flinger)
    : mVSyncSource(src),
      mFlinger(flinger),
      mUseSoftwareVSync(false),
      mVsyncEnabled(false),
      mDebugVsyncEnabled(false),
      mVsyncHintSent(false) {

    for (int32_t i=0 ; i<DisplayDevice::NUM_BUILTIN_DISPLAY_TYPES ; i++) {
        mVSyncEvent[i].header.type = DisplayEventReceiver::DISPLAY_EVENT_VSYNC;
        mVSyncEvent[i].header.id = 0;
        mVSyncEvent[i].header.timestamp = 0;
        mVSyncEvent[i].vsync.count =  0;
    }
    struct sigevent se;
    se.sigev_notify = SIGEV_THREAD;
    se.sigev_value.sival_ptr = this;
    se.sigev_notify_function = vsyncOffCallback;
    se.sigev_notify_attributes = NULL;
    timer_create(CLOCK_MONOTONIC, &se, &mTimerId);
}
void EventThread::onFirstRef() {
    run("EventThread", PRIORITY_URGENT_DISPLAY + PRIORITY_MORE_FAVORABLE);
}
```
EventThread的构造函数很简单。重点是它的onFirstRef函数启动了一个EventThread线程，于是下面的代码才是重点：

``` cpp
bool EventThread::threadLoop() {
    DisplayEventReceiver::Event event;
    Vector< sp<EventThread::Connection> > signalConnections;
    signalConnections = waitForEvent(&event);

    // dispatch events to listeners...
    const size_t count = signalConnections.size();
    for (size_t i=0 ; i<count ; i++) {
        const sp<Connection>& conn(signalConnections[i]);
        // now see if we still need to report this event
        status_t err = conn->postEvent(event);
        ......
    }
    return true;
}
```
上面的函数本身并不复杂，其中调用了一个waitForEvent的函数。这个函数相当之长，为了防止代码展开太多，我们这里暂时不再详细分析这个函数。我们目前只需要知道这个函数的最重要的作用是等待Event的到来，并且查找对event感兴趣的监听者，而在没有event到来时，线程处于休眠状态，等待event的唤醒（我们将下一篇VSYNC的接收和处理中展开分析这个函数）。
这样，EventThread线程就运行起来，处在等待被event唤醒的状态下。
**MessageQueue和EventThread建立连接**
简单说明完EventThread之后，我们再次回到SurfaceFlinger的init过程中来。回到init()函数代码中来：
将SurfaceFlinger的MessageQueue真正和我们刚才创建的EventThread建立起了连接，这样SurfaceFlinger才能真正接收到来自HWC的VSYNC信号。
我们来看下这段代码：

``` cpp
void MessageQueue::setEventThread(const sp<EventThread>& eventThread)  
{  
    mEventThread = eventThread;  
    mEvents = eventThread->createEventConnection();  
    mEventTube = mEvents->getDataChannel();  
    mLooper->addFd(mEventTube->getFd(), 0, ALOOPER_EVENT_INPUT,  
            MessageQueue::cb_eventReceiver, this);  
}  
```

这里代码逻辑其实很简单，就是创建了一个到EventThread的连接，得到了发送VSYNC事件通知的BitTube，然后监控这个BitTube中的套接字，并且指定了收到通知后的回调函数，MessageQueue::cb_eventReceiver。这样一旦VSync信号传来，函数cb_eventReceiver将被调用。
**向Eventhread注册一个事件的监听者——createEventConnection**
在SurfaceFlinger的init函数中，我们调用了mEventQueue.setEventThread(mSFEventThread)函数，我们在前面一章中已经提到过，这个函数将SurfaceFlinger的MessageQueue真正和我们刚才创建的EventThread建立起了连接。我们来看下这段代码：

``` cpp
sp<EventThread::Connection> EventThread::createEventConnection() const {  
    return new Connection(const_cast<EventThread*>(this));  
}  
EventThread::Connection::Connection(  
        const sp<EventThread>& eventThread)  
    : count(-1), mEventThread(eventThread), mChannel(new BitTube())  
{  
}  
void EventThread::Connection::onFirstRef() {  
    mEventThread->registerDisplayEventConnection(this);  
}  
status_t EventThread::registerDisplayEventConnection(  
        const sp<EventThread::Connection>& connection) {  
    mDisplayEventConnections.add(connection);  
    mCondition.broadcast();  
}
```
这个函数会导致一个Connection类的创建，而这个connection类会被保存在EventThread下的一个容器内。
通过createEventConnection这样一个简单的方法，我们其实就注册了一个事件的监听者，得到了发送VSYNC事件通知的BitTube，然后监控这个BitTube中的套接字，并且指定了收到通知后的回调函数，MessageQueue::cb_eventReceiver。这样一旦VSync信号传来，函数cb_eventReceiver将被调用。

##### 6.3.2、VSync信号的处理
我们在前面一章也提到了无论是软件方式还是硬件方式，SurfaceFlinger收到VSync信号后，处理函数都是onVSyncReceived函数：

**VSync消息处理——addResyncSample**
![enter image description here](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/personal.website/zjj.display.sys.SF-Vsync-addResyncSample.png)

``` cpp
bool DispSync::addResyncSample(nsecs_t timestamp) {  
    size_t idx = (mFirstResyncSample + mNumResyncSamples) % MAX_RESYNC_SAMPLES;  
    mResyncSamples[idx] = timestamp;  
  
    ......
    updateModelLocked();  
    .......
}  
```

粗略浏览下这个函数，发现前半部分其实在做一些简单的计数统计，重点实现显然是updateModelLocked函数：

```
void DispSync::updateModelLocked() {  
    if (mNumResyncSamples >= MIN_RESYNC_SAMPLES_FOR_UPDATE) {  
        nsecs_t durationSum = 0;  
        for (size_t i = 1; i < mNumResyncSamples; i++) {  
            size_t idx = (mFirstResyncSample + i) % MAX_RESYNC_SAMPLES;  
            size_t prev = (idx + MAX_RESYNC_SAMPLES - 1) % MAX_RESYNC_SAMPLES;  
            durationSum += mResyncSamples[idx] - mResyncSamples[prev];  
        }  
  
        mPeriod = durationSum / (mNumResyncSamples - 1);  
  
        double sampleAvgX = 0;  
        double sampleAvgY = 0;  
        double scale = 2.0 * M_PI / double(mPeriod);  
        for (size_t i = 0; i < mNumResyncSamples; i++) {  
            size_t idx = (mFirstResyncSample + i) % MAX_RESYNC_SAMPLES;  
            nsecs_t sample = mResyncSamples[idx];  
            double samplePhase = double(sample % mPeriod) * scale;  
            sampleAvgX += cos(samplePhase);  
            sampleAvgY += sin(samplePhase);  
        }  
  
        sampleAvgX /= double(mNumResyncSamples);  
        sampleAvgY /= double(mNumResyncSamples);  
  
        mPhase = nsecs_t(atan2(sampleAvgY, sampleAvgX) / scale);  
        ...... 
        mThread->updateModel(mPeriod, mPhase);  
    }  
}  
```

不得不说，前面大段的数学计算让人有些困惑，我们暂且跳过，先分析下主线流程，也就是mThread->updateModel(mPeriod, mPhase)这个调用：

**DispSyncThread.updateModel的用途**

``` cpp
void updateModel(nsecs_t period, nsecs_t phase) {  
    Mutex::Autolock lock(mMutex);  
    mPeriod = period;  
    mPhase = phase;  
    mCond.signal();  
}  
```

updateModel是DispSyncThread类的函数，这个函数本身代码很短，其实它的主要作用是mCond.signal发送一个信号给等待中的线程。那么究竟是谁在等待这个条件呢？
其实等待这个条件的正是DispSyncThread的循环函数：


``` cpp
virtual bool threadLoop() {  
      status_t err;  
      nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);  
      nsecs_t nextEventTime = 0;  
      while (true) {  
          Vector<CallbackInvocation> callbackInvocations;  
          nsecs_t targetTime = 0;  
          { // Scope for lock  
              Mutex::Autolock lock(mMutex);  
              ......
              if (mPeriod == 0) {  
                  err = mCond.wait(mMutex);  
                  ...... 
              }  
              nextEventTime = computeNextEventTimeLocked(now);  
              targetTime = nextEventTime;  
              ...... 
              }  
              now = systemTime(SYSTEM_TIME_MONOTONIC);  
              ......
              callbackInvocations = gatherCallbackInvocationsLocked(now);  
          }  
          if (callbackInvocations.size() > 0) {  
              fireCallbackInvocations(callbackInvocations);  
          }  
      }  
      return false;  
  }  
```

大量的时间相关的计算和状态的转变我们不再深入研究，我们来看下这个线程被通知唤醒之后做的两个主要的函数的处理，gatherCallbackInvocationsLocked()和fireCallbackInvocations()。

gatherCallbackInvocationsLocked()的代码其实很简单：

```cpp
Vector<CallbackInvocation> gatherCallbackInvocationsLocked(nsecs_t now) {  
    Vector<CallbackInvocation> callbackInvocations;  
    nsecs_t ref = now - mPeriod;  
    for (size_t i = 0; i < mEventListeners.size(); i++) {  
        nsecs_t t = computeListenerNextEventTimeLocked(mEventListeners[i],  
                ref);  
        if (t < now) {  
            CallbackInvocation ci;  
            ci.mCallback = mEventListeners[i].mCallback;  
            ci.mEventTime = t;  
            callbackInvocations.push(ci);  
            mEventListeners.editItemAt(i).mLastEventTime = t;  
        }  
    }  
    return callbackInvocations;  
}  
```

其实就是从mEventListeners取出之前注册的事件监听者，放入callbackInvocations中，等待后面的调用。至于监听者从何处而来？在waitforevent时通过enableVSyncLocked注册的。

继续看下fireCallbackInvocations()函数：

``` cpp
void fireCallbackInvocations(const Vector<CallbackInvocation>& callbacks) {  
    for (size_t i = 0; i < callbacks.size(); i++) {  
        callbacks[i].mCallback->onDispSyncEvent(callbacks[i].mEventTime);  
    }  
}`  
```

我们目前只分析主线的走向,接下来调用了DispSyncSource的onDispSyncEvent在：

```cpp
virtual void onDispSyncEvent(nsecs_t when) {  
        sp<VSyncSource::Callback> callback;  
        {  
            callback = mCallback;  
        }  
        if (callback != NULL) {  
            callback->onVSyncEvent(when);  
        }  
    }  
void EventThread::onVSyncEvent(nsecs_t timestamp) {  
    Mutex::Autolock _l(mLock);  
    mVSyncEvent[0].header.type = DisplayEventReceiver::DISPLAY_EVENT_VSYNC;  
    mVSyncEvent[0].header.id = 0;  
    mVSyncEvent[0].header.timestamp = timestamp;  
    mVSyncEvent[0].vsync.count++;  
    mCondition.broadcast();  
}  
```

我们看到这里mCondition.broadcas发出了命令，那么EventThread中waitforEvent的等待就会被唤醒。而一旦唤醒，我们就回到了EventThread的loop中，我们来看下代码：

```
bool EventThread::threadLoop() {  
    DisplayEventReceiver::Event event;  
    Vector< sp<EventThread::Connection> > signalConnections;  
    signalConnections = waitForEvent(&event);  
  
    // dispatch events to listeners...  
    const size_t count = signalConnections.size();  
    for (size_t i=0 ; i<count ; i++) {  
        const sp<Connection>& conn(signalConnections[i]);  
        // now see if we still need to report this event  
        status_t err = conn->postEvent(event);  
        ......
    }  
    return true;  
}  
```

这里主要就是通过conn->postEvent来分发事件：

``` cpp
status_t EventThread::Connection::postEvent(  
        const DisplayEventReceiver::Event& event) {  
    ssize_t size = DisplayEventReceiver::sendEvents(mChannel, &event, 1);  
    return size < 0 ? status_t(size) : status_t(NO_ERROR);  
}  
ssize_t DisplayEventReceiver::sendEvents(const sp<BitTube>& dataChannel,  
        Event const* events, size_t count)  
{  
    return BitTube::sendObjects(dataChannel, events, count);  
}  
```
![enter image description here](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/personal.website/zjj.display.sys.App-SurfaceFlinger-Vsync-postEvent.png)

其实看到这里的BitTube我们就明白了，在本文开始时候我们提到：

通过createEventConnection这样一个简单的方法，我们其实就注册了一个事件的监听者，得到了发送VSYNC事件通知的BitTube，然后监控这个BitTube中的套接字，并且指定了收到通知后的回调函数，MessageQueue::cb_eventReceiver。这样一旦VSync信号传来，函数cb_eventReceiver将被调用。

所以我们这里可以来看看MessageQueue::cb_eventReceiver函数了：

``` cpp
int MessageQueue::cb_eventReceiver(int fd, int events, void* data) {  
    MessageQueue* queue = reinterpret_cast<MessageQueue *>(data);  
    return queue->eventReceiver(fd, events);  
}  
  
int MessageQueue::eventReceiver(int fd, int events) {  
    ssize_t n;  
    DisplayEventReceiver::Event buffer[8];  
    while ((n = DisplayEventReceiver::getEvents(mEventTube, buffer, 8)) > 0) {  
        for (int i=0 ; i<n ; i++) {  
            if (buffer[i].header.type == DisplayEventReceiver::DISPLAY_EVENT_VSYNC) {  
                mHandler->dispatchInvalidate();  
                break;  
            }  
        }  
    }  
    return 1;  
}  
```

我们看到收到消息之后MessageQueue对消息进行了分发，我们目前走的是dispatchInvalidate()。

``` cpp
void MessageQueue::Handler::dispatchInvalidate() {  
    if ((android_atomic_or(eventMaskInvalidate, &mEventMask) & eventMaskInvalidate) == 0) {  
        mQueue.mLooper->sendMessage(this, Message(MessageQueue::INVALIDATE));  
    }  
}  
  
void MessageQueue::Handler::handleMessage(const Message& message) {  
    switch (message.what) {  
        case INVALIDATE:  
            android_atomic_and(~eventMaskInvalidate, &mEventMask);  
            mQueue.mFlinger->onMessageReceived(message.what);  
            break;  
        case REFRESH:  
            android_atomic_and(~eventMaskRefresh, &mEventMask);  
            mQueue.mFlinger->onMessageReceived(message.what);  
            break;  
        case TRANSACTION:  
            android_atomic_and(~eventMaskTransaction, &mEventMask);  
            mQueue.mFlinger->onMessageReceived(message.what);  
            break;  
    }  
}  
  
void SurfaceFlinger::onMessageReceived(int32_t what) {  
    ATRACE_CALL();  
    switch (what) {  
    case MessageQueue::TRANSACTION:  
        handleMessageTransaction();  
        break;  
    case MessageQueue::INVALIDATE:  
        handleMessageTransaction();  
        handleMessageInvalidate();  
        signalRefresh();  
        break;  
    case MessageQueue::REFRESH:  
        handleMessageRefresh();  
        break;  
    }  
}  
```

到了这里，就进入了SurfaceFlinger的处理流程，我们看到对于INVALIDATE的消息，实际上系统在处理过程中实际还是会发送一个Refresh消息。

##### 6.4、App向Eventhread注册一个事件的监听者—createEventConnection()

在ViewRootImpl的构造函数中会实例化Choreographer对象

``` cpp
public ViewRootImpl(Context context, Display display) {
                . . . . . 
        mChoreographer = Choreographer.getInstance();
 }
```

在mChoreographer 的构造函数中实例化FrameDisplayEventReceiver对象

```
private Choreographer(Looper looper) {
               . . . . . .
               mDisplayEventReceiver = USE_VSYNC ? new FrameDisplayEventReceiver(looper) : null;
 }
```

在FrameDisplayEventReceiver的父类构造函数中会调用到，android_view_DisplayEventReceiver.cpp中的nativeInit方法,在nativeInit方法中有如下过程

``` cpp
static jlong nativeInit(JNIEnv* env, jclass clazz, jobject receiverWeak,
        jobject messageQueueObj) {
        . . . . . .
    sp<NativeDisplayEventReceiver> receiver = new NativeDisplayEventReceiver(env,
            receiverWeak, messageQueue);
    status_t status = receiver->initialize();
    . . . . . .
```

创建NativeDisplayEventReceiver类 类型指针
在NativeDisplayEventReceiver的构造函数中会调用DisplayEventReceiver类的无参构造函数实例化成员mReceiver；

``` cpp
DisplayEventReceiver::DisplayEventReceiver() {
    sp<ISurfaceComposer> sf(ComposerService::getComposerService());
    if (sf != NULL) {
        mEventConnection = sf->createDisplayEventConnection();
        if (mEventConnection != NULL) {
            mDataChannel = mEventConnection->getDataChannel();
        }
    }
}
```

在这段代码中获取Surfaceflinger服务的代理对象，然后通过Binder IPC创建BpDisplayEventConnection对象
该函数经由BnSurfaceComposer.onTransact函数辗转调用到SurfaceFlinger.createDisplayEventConnection函数：

``` cpp
sp<IDisplayEventConnection> SurfaceFlinger::createDisplayEventConnection() {
    return mEventThread->createEventConnection();
}
```

出现了熟悉的面孔mEventThread，该对象是一个EventThread对象，该对象在SurfaceFlinger.init函数里面创建，但是创建运行以后，貌似还没有进行任何的动作，这里调用createEventConnection函数：

``` cpp
sp<EventThread::Connection> EventThread::createEventConnection() const {
    return new Connection(const_cast<EventThread*>(this));
}
```

然后mEventConnection->getDataChannel()方法再次通过Binder IPC创建 BitTube对象mDataChannel ，在Binder IPC创建mDataChannel 过程中会从服务端EventThread::Connection::Connection中（在EventThread类中定义）接收一个socketpair创建的FIFO文件描述符；

EventThread::Connection::Connection创建描述符的代码： 
Connection构造函数调用BitTube的无参构造函数，在BitTube的构造函数中调用init函数；

``` cpp
void BitTube::init(size_t rcvbuf, size_t sndbuf) {
    int sockets[2];
    if (socketpair(AF_UNIX, SOCK_SEQPACKET, 0, sockets) == 0) {
        size_t size = DEFAULT_SOCKET_BUFFER_SIZE;
        setsockopt(sockets[0], SOL_SOCKET, SO_RCVBUF, &rcvbuf, sizeof(rcvbuf));
        setsockopt(sockets[1], SOL_SOCKET, SO_SNDBUF, &sndbuf, sizeof(sndbuf));
        // sine we don't use the "return channel", we keep it small...
        setsockopt(sockets[0], SOL_SOCKET, SO_SNDBUF, &size, sizeof(size));
        setsockopt(sockets[1], SOL_SOCKET, SO_RCVBUF, &size, sizeof(size));
        fcntl(sockets[0], F_SETFL, O_NONBLOCK);
        fcntl(sockets[1], F_SETFL, O_NONBLOCK);
        mReceiveFd = sockets[0];
        mSendFd = sockets[1];
    } else {
        mReceiveFd = -errno;
        ALOGE("BitTube: pipe creation failed (%s)", strerror(-mReceiveFd));
    }
}
```

调用到NativeDisplayEventReceiver类的父类DisplayEventDispatcher中的initialize()方法，
  将BpDisplayEventConnection对象获取到的mDataChannel （BitTube类型）中的文件描述符添加到UI主线程Looper的epoll中，
  当文件描述符中被写入数据时，该epoll_wait会被唤醒；
  直接看代码：

```cpp
status_t NativeDisplayEventReceiver::initialize() {
    status_t result = mReceiver.initCheck();
    if (result) {
        ALOGW("Failed to initialize display event receiver, status=%d", result);
        return result;
    }
    int rc = mMessageQueue->getLooper()->addFd(mReceiver.getFd(), 0, Looper::EVENT_INPUT, this, NULL);
    if (rc < 0) {
        return UNKNOWN_ERROR;
    }
    return OK;
}
```

这里的主要代码是mMessageQueue->getLooper()->addFd()这一行，其中的参数mReceiver.getFd()返回的是在创建NativeDisplayEventReceiver时从SurfaceFlinger服务端接收回来的socket接收端描述符，前面分析到 
mMessageQueue是与当前应用线程关联的java层的MessageQueue对应的native层的MessageQueue对象，下面看一下Looper.addFd这个函数，上面调用时传进来的this指针对应的是一个NativeDisplayEventReceiver对象，该类继承了LooperCallback：

```cpp
int Looper::addFd(int fd, int ident, int events, Looper_callbackFunc callback, void* data) {
    return addFd(fd, ident, events, callback ? new SimpleLooperCallback(callback) : NULL, data);
}
int Looper::addFd(int fd, int ident, int events, const sp<LooperCallback>& callback, void* data) { 
    int epollEvents = 0;
    if (events & EVENT_INPUT) epollEvents |= EPOLLIN;
    if (events & EVENT_OUTPUT) epollEvents |= EPOLLOUT;

    { // acquire lock
        AutoMutex _l(mLock);

        Request request;
        request.fd = fd;
        request.ident = ident;
        request.callback = callback;
        request.data = data;

        struct epoll_event eventItem;
        memset(& eventItem, 0, sizeof(epoll_event)); // zero out unused members of data field union
        eventItem.events = epollEvents;
        eventItem.data.fd = fd;

        ssize_t requestIndex = mRequests.indexOfKey(fd);
        if (requestIndex < 0) {
            int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, fd, & eventItem);
            if (epollResult < 0) {
            }
            mRequests.add(fd, request);
        } else {
            int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_MOD, fd, & eventItem);
            if (epollResult < 0) {
                return -1;
            }
            mRequests.replaceValueAt(requestIndex, request);
        }
    } // release lock
    return 1;
}
```

首先将上面传进来的NativeDisplayEventReceiver对象封装成一个SimpleLooperCallback对象，调用下面的addFd函数的时候主要步骤如下： 
（1）创建一个struct epoll_event结构体对象，将对应的内存全部用清0，并作对应的初始化； 
（2）查询通过addFd方法已经添加到epoll中监听的文件描述符； 
（3）查询不到的话，则调用epoll_ctl方法设置EPOLL_CTL_ADD属性将对应的文件描述符添加到epoll监听的描述符中； 
（4）根据前面addFd传入的参数EVENT_INPUT，说明当前应用线程的native层的Looper对象中的epoll机制已经开始监听来自于SurfaceFlinger服务端socket端的写入事件。 

##### 6.5、App请求Vsync信号
前面讲解ViewRootImpl.setView()的时候，因涉及到Vsync信号知识，requestLayout()没有具体讲解，现在继续。
``` java
Override
public void requestLayout() {
        scheduleTraversals();
}
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        ......
    }
}
```
[->Choreographer.java]

``` java
public void postCallback(int callbackType, Runnable action, Object token) {
    postCallbackDelayed(callbackType, action, token, 0);
}

public void postCallbackDelayed(int callbackType,
        Runnable action, Object token, long delayMillis) {
    ......
    postCallbackDelayedInternal(callbackType, action, token, delayMillis);
}

private void postCallbackDelayedInternal(int callbackType,
        Object action, Object token, long delayMillis) {
    ......

    synchronized (mLock) {
        final long now = SystemClock.uptimeMillis();
        final long dueTime = now + delayMillis;
        //将要执行的回调封装成CallbackRecord对象，保存到mCallbackQueues数组中
        mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);
        
        if (dueTime <= now) {
            scheduleFrameLocked(now);
        } else {
            Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
            msg.arg1 = callbackType;
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, dueTime);
        }
    }
}
private void scheduleFrameLocked(long now) {
    if (!mFrameScheduled) {
        mFrameScheduled = true;
        if (USE_VSYNC) {
            if (isRunningOnLooperThreadLocked()) {
                scheduleVsyncLocked();
            } else {
                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtFrontOfQueue(msg);
            }
        } else {
            final long nextFrameTime = Math.max(
                    mLastFrameTimeNanos / TimeUtils.NANOS_PER_MS + sFrameDelay, now);
            Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, nextFrameTime);
        }
    }
}
```

消息处理：

``` java
    private final class FrameHandler extends Handler {
    public FrameHandler(Looper looper) {
        super(looper);
    }

    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case MSG_DO_SCHEDULE_VSYNC:
                doScheduleVsync();
                break;
        }
    }
}
    void doScheduleVsync() {
    synchronized (mLock) {
        if (mFrameScheduled) {
            scheduleVsyncLocked();
        }
    }
}
private void scheduleVsyncLocked() {  
//申请Vsync信号  
mDisplayEventReceiver.scheduleVsync();  
}  
```


在该函数中考虑了两种情况，一种是系统没有使用Vsync机制，在这种情况下，首先根据屏幕刷新频率计算下一次刷新时间，通过异步消息方式延时执行doFrame()函数实现屏幕刷新。如果系统使用了Vsync机制，并且当前线程具备消息循环，则直接请求Vsync信号，否则就通过主线程来请求Vsync信号。

##### 6.5.1、Vsync请求过程

我们知道在Choreographer构造函数中，构造了一个FrameDisplayEventReceiver对象，用于请求并接收Vsync信号，Vsync信号请求过程如下：

``` java
private void scheduleVsyncLocked() {  
//申请Vsync信号  
mDisplayEventReceiver.scheduleVsync();  
}  
```

[->DisplayEventReceiver.java]

``` java
public void scheduleVsync() {
    if (mReceiverPtr == 0) {
        Log.w(TAG, "Attempted to schedule a vertical sync pulse but the display event "
                + "receiver has already been disposed.");
    } else {
        nativeScheduleVsync(mReceiverPtr);
    }
}
```

[->android_view_DisplayEventReceiver.cpp	]

``` java
static void nativeScheduleVsync(JNIEnv* env, jclass clazz, jlong receiverPtr) {
sp<NativeDisplayEventReceiver> receiver =
        reinterpret_cast<NativeDisplayEventReceiver*>(receiverPtr);
status_t status = receiver->scheduleVsync();
if (status) {
    String8 message;
    message.appendFormat("Failed to schedule next vertical sync pulse.  status=%d", status);
    jniThrowRuntimeException(env, message.string());
}
}
```

VSync请求过程又转交给了DisplayEventReceiver：
[->DisplayEventReceiver.cpp]

``` java
status_t DisplayEventReceiver::requestNextVsync() {
if (mEventConnection != NULL) {
    mEventConnection->requestNextVsync();
    return NO_ERROR;
}
return NO_INIT;
}
```
这里的mEventConnection也是前面创建native层对象NativeDisplayEventReceiver时创建的，实际对象是一个BpDisplayEventConnection对象，也就是一个Binder客户端，对应的Binder服务端BnDisplayEventConnection是一个EventThread::Connection对象，对应的BpDisplayEventConnection.requestNextVsync函数和BnDisplayEventConnection.onTransact(REQUEST_NEXT_VSYNC)函数没有进行特别的处理，下面就调用到EventThread::Connection.requestNextVsync函数，从BnDisplayEventConnection.onTransact(REQUEST_NEXT_VSYNC)开始已经从用户进程将需要垂直同步信号的请求发送到了SurfaceFlinger进程，下面的函数调用开始进入SF进程：

``` cpp
void EventThread::Connection::requestNextVsync() {
    mEventThread->requestNextVsync(this);
    }
```



辗转调用到EventThread.requestNextVsync函数，注意里面传了参数this，也就是当前的EventThread::Connection对象，需要明确的是，这里的mEventThread对象是创建EventThread::Connection对象的时候保存的，对应的是SurfaceFlinger对象的里面的mEventThread成员，该对象是一个在SurfaceFlinger.init里面创建并启动的线程对象，可见设计的时候就专门用这个SurfaceFlinger.mEventThread线程来接收来自应用进程的同步信号请求，每来一个应用进程同步信号请求，就通过SurfaceFlinger.mEventThread创建一个EventThread::Connection对象，并通过EventThread.registerDisplayEventConnection函数将创建的EventThread::Connection对象保存到EventThread.mDisplayEventConnections里面，上面有调用到了EventThread.requestNextVsync函数：

``` cpp
void EventThread::requestNextVsync(const sp<EventThread::Connection>& connection) {
    Mutex::Autolock _l(mLock);
    if (connection->count < 0) {
        connection->count = 0;
        mCondition.broadcast();
    }
}
```

传进来的是一个前面创建的EventThread::Connection对象，里面判断到了EventThread::Connection.count成员变量，看一下EventThread::Connection构造函数中初始变量的值：

``` cpp
EventThread::Connection::Connection(const sp<EventThread>& eventThread)
    : count(-1), mEventThread(eventThread), mChannel(new BitTube()){
}
```

可以看到初始值是-1，这个值就是前面那个问题的关键，EventThread::Connection.count标示了这次应用进程的垂直同步信号的请求是一次性的，还是多次重复的，看一下注释里面对于这个变量的说明：

``` cpp
// count >= 1 : continuous event. count is the vsync rate
// count == 0 : one-shot event that has not fired
// count ==-1 : one-shot event that fired this round / disabled
int32_t count;
```

很清楚的说明了，count = 0说明当前的垂直同步信号请求是一个一次性的请求，并且还没有被处理。上面EventThread::requestNextVsync里面将count设置成0，同时调用了mCondition.broadcast()唤醒所有正在等待mCondition的线程，这个会触发EventThread.waitForEvent函数从：

```
mCondition.wait(mLock);
```

中醒来，醒来之后经过一轮do…while循环就会返回，返回以后调用序列如下： 
（1）EventThread::Connection.postEvent(event) 
（2）DisplayEventReceiver::sendEvents(mChannel, &event, 1)，mChannel参数就是前面创建DisplayEventReceiver是创建的BitTube对象 
（3）BitTube::sendObjects(dataChannel, events, count)，static函数，通过dataChannel指向BitTube对象 
最终调用到BitTube::sendObjects函数：

``` cpp
ssize_t BitTube::sendObjects(const sp<BitTube>& tube, void const* events, size_t count, size_t objSize){
    const char* vaddr = reinterpret_cast<const char*>(events);
    ssize_t size = tube->write(vaddr, count*objSize);
    return size < 0 ? size : size / static_cast<ssize_t>(objSize);
}
```

继续调用到BitTube::write函数：

``` cpp
ssize_t BitTube::write(void const* vaddr, size_t size){
    ssize_t err, len;
    do {
        len = ::send(mSendFd, vaddr, size, MSG_DONTWAIT | MSG_NOSIGNAL);
        // cannot return less than size, since we're using SOCK_SEQPACKET
        err = len < 0 ? errno : 0;
    } while (err == EINTR);
    return err == 0 ? len : -err;
}
```

这里调用到了::send函数，::是作用域描述符，如果前面没有类名之类的，代表的就是全局的作用域，也就是调用全局函数send，这里很容易就能想到这是一个socket的写入函数，也就是将event事件数据写入到BitTube中互联的socket中，这样在另一端马上就能收到写入的数据，前面分析到这个BitTube的socket的两端连接着SurfaceFlinger进程和应用进程，也就是说通过调用BitTube::write函数，将最初由SurfaceFlinger捕获到的垂直信号事件经由BitTube中互联的socket从SurfaceFlinger进程发送到了应用进程中BitTube的socket接收端。 
下面就要分析应用进程是如何接收并使用这个垂直同步信号事件的。

##### 6.5.2、应用进程接收VSync
###### 6.5.2.1、解析VSync事件
VSync同步信号事件已经发送到用户进程中的socket接收端，在前面NativeDisplayEventReceiver.initialize中分析到应用进程端的socket接收描述符已经被添加到Choreographer所在线程的native层的Looper机制中，在epoll中监听EPOLLIN事件，当socket收到数据后，epoll会马上返回，下面分步骤看一下Looper.pollInner()数： 
（1）epoll_wait

``` cpp
struct epoll_event eventItems[EPOLL_MAX_EVENTS];
int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);
```

在监听到描述符对应的事件后，epoll_wait会马上返回，并将产生的具体事件类型写入到参数eventItems里面，最终返回的eventCount是监听到的事件的个数 
（2）事件分析

``` cpp
   for (int i = 0; i < eventCount; i++) {
        int fd = eventItems[i].data.fd;
        uint32_t epollEvents = eventItems[i].events;
        if (fd == mWakeReadPipeFd) {   //判断是不是pipe读管道的事件   这里如果是EventThread,这里就是一个socket的描述符,而不是mWakeReadPipeFd
            if (epollEvents & EPOLLIN) {
                awoken(); // 清空读管道中的数据
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on wake read pipe.", epollEvents);
            }
        } else {
           //EventThread接收到同步信号走的这里
            ssize_t requestIndex = mRequests.indexOfKey(fd);
            if (requestIndex >= 0) {
                int events = 0;
                if (epollEvents & EPOLLIN) events |= EVENT_INPUT;
                if (epollEvents & EPOLLOUT) events |= EVENT_OUTPUT;
                if (epollEvents & EPOLLERR) events |= EVENT_ERROR;
                if (epollEvents & EPOLLHUP) events |= EVENT_HANGUP;
                pushResponse(events, mRequests.valueAt(requestIndex));
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on fd %d that is "
                        "no longer registered.", epollEvents, fd);
            }
        }
    }
```

Looper目前了解到的主要监听的文件描述符种类有两种： 
1）消息事件，epoll_wait监听pipe管道的接收端描述符mWakeReadPipeFd 
2）与VSync信号，epoll_wait监听socket接收端描述符，并在addFd的过程中将相关的信息封装在一个Request结构中，并以fd为key存储到了mRequests中，具体可以回过头看3.1.2关于addFd的分析； 
因此，上面走的是else的分支，辨别出当前的事件类型后，调用pushResponse：

``` cpp
void Looper::pushResponse(int events, const Request& request) {
    Response response;
    response.events = events;
    response.request = request;  //复制不是引用，调用拷贝构造函数
    mResponses.push(response);
}
```

该函数将Request和events封装在一个Response对象里面，存储到了mResponses里面，也就是mResponses里面放的是“某某fd上接收到了类别为events的时间”记录，继续向下看Looper.pollInner函数 
（3）事件分发处理

``` cpp
// Invoke all response callbacks.
for (size_t i = 0; i < mResponses.size(); i++) {
    Response& response = mResponses.editItemAt(i);
    if (response.request.ident == POLL_CALLBACK) {
        int fd = response.request.fd;
        int events = response.events;
        void* data = response.request.data;
        int callbackResult = response.request.callback->handleEvent(fd, events, data);
        if (callbackResult == 0) {
            removeFd(fd);
        }
        // Clear the callback reference in the response structure promptly because we
        // will not clear the response vector itself until the next poll.
        response.request.callback.clear();
        result = POLL_CALLBACK;
    }
}
```

这里的response.request是从pushResponse里面复制过来的，里面的request对应的Request对象是在addFd的时候创建的，ident成员就是POLL_CALLBACK，所以继续走到response.request.callback->handleEvent这个函数，回忆一下3.1.2里面的addFd函数，这里的callback实际上是一个SimpleLooperCallback（定义在Looper.cpp中）对象，看一下里面的handleEvent函数：

``` cpp
int SimpleLooperCallback::handleEvent(int fd, int events, void* data) {
    return mCallback(fd, events, data);
}
```

这里的mCallback就是当时在addFd的时候传进来的callBack参数，实际上对应的就是NativeDisplayEventReceiver对象本身，因此最终就将垂直同步信号事件分发到了NativeDisplayEventReceiver.handleEvent函数中。

##### 6.5.3、**VSync事件分发**
调用到NativeDisplayEventReceiver.handleEvent函数，该函数定义在android_view_DisplayEventReceiver.cpp中，直接列出该函数：

``` cpp
int NativeDisplayEventReceiver::handleEvent(int receiveFd, int events, void* data) {
    if (events & (Looper::EVENT_ERROR | Looper::EVENT_HANGUP)) {
        ALOGE("Display event receiver pipe was closed or an error occurred.  "
                "events=0x%x", events);
        return 0; // remove the callback
    } 
    if (!(events & Looper::EVENT_INPUT)) {
        ALOGW("Received spurious callback for unhandled poll event.  "
                "events=0x%x", events);
        return 1; // keep the callback
    } 
    // Drain all pending events, keep the last vsync.
    nsecs_t vsyncTimestamp;
    int32_t vsyncDisplayId;
    uint32_t vsyncCount;
    if (processPendingEvents(&vsyncTimestamp, &vsyncDisplayId, &vsyncCount)) {
        ALOGV("receiver %p ~ Vsync pulse: timestamp=%" PRId64 ", id=%d, count=%d",
                this, vsyncTimestamp, vsyncDisplayId, vsyncCount);
        mWaitingForVsync = false;
        dispatchVsync(vsyncTimestamp, vsyncDisplayId, vsyncCount);
    }
    return 1; // keep the callback
}
```

首先判断事件是不是正确的Looper::EVENT_INPUT事件，然后调用到NativeDisplayEventReceiver.processPendingEvents函数：

``` cpp
bool NativeDisplayEventReceiver::processPendingEvents(nsecs_t* outTimestamp, int32_t* outId, uint32_t* outCount) {
    bool gotVsync = false;
    DisplayEventReceiver::Event buf[EVENT_BUFFER_SIZE];
    ssize_t n;
    while ((n = mReceiver.getEvents(buf, EVENT_BUFFER_SIZE)) > 0) {
        for (ssize_t i = 0; i < n; i++) {
            const DisplayEventReceiver::Event& ev = buf[i];
            switch (ev.header.type) {
            case DisplayEventReceiver::DISPLAY_EVENT_VSYNC:
                // Later vsync events will just overwrite the info from earlier
                // ones. That's fine, we only care about the most recent.
                gotVsync = true;
                *outTimestamp = ev.header.timestamp;
                *outId = ev.header.id;
                *outCount = ev.vsync.count;
                break;
            case DisplayEventReceiver::DISPLAY_EVENT_HOTPLUG:
                dispatchHotplug(ev.header.timestamp, ev.header.id, ev.hotplug.connected);
                break;
            default:
                ALOGW("receiver %p ~ ignoring unknown event type %#x", this, ev.header.type);
                break;
            }
        }
    }
    if (n < 0) {
        ALOGW("Failed to get events from display event receiver, status=%d", status_t(n));
    }
    return gotVsync;
}
```

这里的mReceiver也就是前面创建NativeDisplayEventReceiver对象是创建的成员变量对象DisplayEventReceiver，下面调用到DisplayEventReceiver.getEvents函数，应该是要从出现同步信号事件的socket中读取数据，上面Looper机制中epoll中监听到socket以后，返回到NativeDisplayEventReceiver.handleEvent里面，但是socket里面的数据还没有读取，下面的调用流程为： 
（1）mReceiver.getEvents(buf, EVENT_BUFFER_SIZE) —-> DisplayEventReceiver::getEvents(DisplayEventReceiver::Event* events, size_t count) 
（2）BitTube::recvObjects(dataChannel, events, count) —-> BitTube::recvObjects(const sp& tube, void* events, size_t count, size_t objSize) 
看一下这个recvObjects函数：

```cpp
ssize_t BitTube::recvObjects(const sp<BitTube>& tube, void* events, size_t count, size_t objSize)
{
    char* vaddr = reinterpret_cast<char*>(events);
    ssize_t size = tube->read(vaddr, count*objSize);
    return size < 0 ? size : size / static_cast<ssize_t>(objSize);
}
```

这里在NativeDisplayEventReceiver中创建了一个缓冲区，并在recvObjects中将socket中的Event数据读到这个缓冲区中，这个Event.header.type一般都是DISPLAY_EVENT_VSYNC，因此在上面的processPendingEvents函数中会将Event数据保存在outCount所指向的内存中，并返回true。 接下来返回到NativeDisplayEventReceiver.handleEvent后会调用到dispatchVsync函数：

``` cpp
void NativeDisplayEventReceiver::dispatchVsync(nsecs_t timestamp, int32_t id, uint32_t count) {
    JNIEnv* env = AndroidRuntime::getJNIEnv();
    env->CallVoidMethod(mReceiverObjGlobal, gDisplayEventReceiverClassInfo.dispatchVsync, timestamp, id, count);
    mMessageQueue->raiseAndClearException(env, "dispatchVsync");
}
```

这里的处理很直接，直接调用mReceiverObjGlobal对象在gDisplayEventReceiverClassInfo.dispatchVsync中指定的函数，将后面的timestamp（时间戳） id（设备ID） count（经过的同步信号的数量，一般没有设置采样频率应该都是1），下面分别看一下mReceiverObjGlobal以及gDisplayEventReceiverClassInfo.dispatchVsync代表的是什么？ 
（1）mReceiverObjGlobal

``` cpp
NativeDisplayEventReceiver::NativeDisplayEventReceiver(JNIEnv* env, jobject receiverObj, const sp<MessageQueue>& messageQueue) :
        mReceiverObjGlobal(env->NewGlobalRef(receiverObj)), mMessageQueue(messageQueue), mWaitingForVsync(false) {
    ALOGV("receiver %p ~ Initializing input event receiver.", this);
}
```

可以看到mReceiverObjGlobal是创建NativeDisplayEventReceiver对象时传进来的第二个参数，该对象是在nativeInit函数中创建： 

``` cpp
sp receiver = new NativeDisplayEventReceiver(env, receiverObj, messageQueue); 
```

进一步的，receiverObj是调用nativeInit函数时传进来的第一个参数（第一个参数env是系统用于连接虚拟机时自动加上的），nativeInit函数又是在Choreographer中创建FrameDisplayEventReceiver对象时，在基类DisplayEventReceiver构造器中调用的，因此这里的mReceiverObjGlobal对应的就是Choreographer中的FrameDisplayEventReceiver成员mDisplayEventReceiver。 
（2）gDisplayEventReceiverClassInfo.dispatchVsync 
在JNI中有很多这样的类似的结构体对象，这些对象都是全局结构体对象，这里的gDisplayEventReceiverClassInfo就是这样的一个对象，里面描述了一些在整个文件内可能会调用到的java层的相关类以及成员函数的相关信息，看一下gDisplayEventReceiverClassInfo：

``` cpp
static struct {
    jclass clazz; 
    jmethodID dispatchVsync;
    jmethodID dispatchHotplug;
} gDisplayEventReceiverClassInfo;
```

看一下里面的变量名称就能知道大致的含义，clazz成员代表的是某个java层的类的class信息，dispatchVsync和dispatchHotplug代表的是java层类的方法的方法信息，看一下该文件中注册JNI函数的方法：

```cpp
int register_android_view_DisplayEventReceiver(JNIEnv* env) {
    int res = RegisterMethodsOrDie(env, "android/view/DisplayEventReceiver", gMethods, NELEM(gMethods)); 
    jclass clazz = FindClassOrDie(env, "android/view/DisplayEventReceiver");
    gDisplayEventReceiverClassInfo.clazz = MakeGlobalRefOrDie(env, clazz); 
    gDisplayEventReceiverClassInfo.dispatchVsync = GetMethodIDOrDie(env, gDisplayEventReceiverClassInfo.clazz, "dispatchVsync", "(JII)V");
    gDisplayEventReceiverClassInfo.dispatchHotplug = GetMethodIDOrDie(env, gDisplayEventReceiverClassInfo.clazz, "dispatchHotplug", "(JIZ)V");
    return res;
}
```

RegisterMethodsOrDie调用注册了java层调用native方法时链接到的函数的入口，下面clazz对应的就是java层的“android/view/DisplayEventReceiver.java”类，gDisplayEventReceiverClassInfo.dispatchVsync里面保存的就是clazz类信息中与dispatchVsync方法相关的信息，同样dispatchHotplug也是。 
分析到这里，就知道应用进程native接收到同步信号事件后，会调用Choreographer中的FrameDisplayEventReceiver成员mDisplayEventReceiver的dispatchVsync方法。

##### 6.5.4、应用接收Vsync
看一下FrameDisplayEventReceiver.dispatchVsync方法，也就是DisplayEventReceiver.dispatchVsync方法(Choreographer.java)：



```
   // Called from native code.   
    @SuppressWarnings("unused")
    private void dispatchVsync(long timestampNanos, int builtInDisplayId, int frame) {
        onVsync(timestampNanos, builtInDisplayId, frame);
    }
    注释表明这个方法是从native代码调用的，该函数然后会调用FrameDisplayEventReceiver.onVsync方法：
        @Override
        public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
            // Ignore vsync from secondary display.
            // This can be problematic because the call to scheduleVsync() is a one-shot.
            // We need to ensure that we will still receive the vsync from the primary
            // display which is the one we really care about.  Ideally we should schedule
            // vsync for a particular display.
            // At this time Surface Flinger won't send us vsyncs for secondary displays
            // but that could change in the future so let's log a message to help us remember
            // that we need to fix this.
            //注释：忽略来自非主显示器的Vsync信号，但是我们前面调用的scheduleVsync函数只能请求到一次Vsync信号，因此需要重新调用scheduleVsync函数
            //请求来自主显示设备的Vsync信号
            if (builtInDisplayId != SurfaceControl.BUILT_IN_DISPLAY_ID_MAIN) {
                Log.d(TAG, "Received vsync from secondary display, but we don't support "
                        + "this case yet.  Choreographer needs a way to explicitly request "
                        + "vsync for a specific display to ensure it doesn't lose track "
                        + "of its scheduled vsync.");
                scheduleVsync();
                return;
            }

            // Post the vsync event to the Handler.
            // The idea is to prevent incoming vsync events from completely starving
            // the message queue.  If there are no messages in the queue with timestamps
            // earlier than the frame time, then the vsync event will be processed immediately.
            // Otherwise, messages that predate the vsync event will be handled first.
            long now = System.nanoTime();
            if (timestampNanos > now) {
                Log.w(TAG, "Frame time is " + ((timestampNanos - now) * 0.000001f)
                        + " ms in the future!  Check that graphics HAL is generating vsync "
                        + "timestamps using the correct timebase.");
                timestampNanos = now;
            }

            if (mHavePendingVsync) {
                Log.w(TAG, "Already have a pending vsync event.  There should only be "
                        + "one at a time.");
            } else {
                mHavePendingVsync = true;
            }

            mTimestampNanos = timestampNanos;  //同步信号时间戳
            mFrame = frame;                //同步信号的个数，理解就是从调用scheduleVsync到onVsync接收到信号之间经历的同步信号的个数，一般都是1
            Message msg = Message.obtain(mHandler, this);
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
        }
```

貌似这里的处理只是往Choreographer对象中的mHandler对应的线程Looper中发送一个消息，消息的内容有两个特点： 
（1）将this，也就是当前的FrameDisplayEventReceiver对象作为参数，后面会回调到FrameDisplayEventReceiver.run方法； 
（2）为Message设置FLAG_ASYNCHRONOUS属性； 
发送这个FLAG_ASYNCHRONOUS消息后，后面会回调到FrameDisplayEventReceiver.run方法，至于为什么，后面再写文章结合View.invalidate方法的过程分析，看一下FrameDisplayEventReceiver.run方法：

```
@Override
public void run() {
    mHavePendingVsync = false;
    doFrame(mTimestampNanos, mFrame);
}
```

调用Choreographer.doFrame方法，如果是重绘事件doFrame方法会最终调用到ViewRootImpl.performTraversals方法进入实际的绘制流程。经过上面的分析可以知道，调用一次Choreographer.scheduleVsyncLocked只会请求一次同步信号，也就是回调一次FrameDisplayEventReceiver.onVsync方法，在思考一个问题，一个应用进程需要多次请求Vsync同步信号会不会使用同样的一串对象？多个线程又是怎么样的？ 
答：一般绘制操作只能在主线程里面进行，因此一般来说只会在主线程里面去请求同步信号，可以认为不会存在同一个应用的多个线程请求SF的Vsync信号，Choreographer是一个线程内的单例模式，存储在了 ThreadLocal sThreadInstance对象里面，所以主线程多次请求使用的是同一个Choreographer对象，所以后面的一串对象应该都是可以复用的。

#### （七）、参考文档(特别感谢各位前辈的分析和图示)：

[【Android 7.1.2 (Android N) Android Graphics 系统 分析】](http://charlesvincent.cc/2018/02/01/Android-7-1-2-Android-N-Android-Graphics-%E7%B3%BB%E7%BB%9F%E5%88%86%E6%9E%90/)
[【Android图形显示之硬件抽象层Gralloc】](https://blog.csdn.net/yangwen123/article/details/12192401)

