---
title: Android N Display System（1）：Android Display System 系统分析之Android Graphics 系统分析
cover: https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/hexo.themes/bing-wallpaper-2018.04.07.jpg
categories: 
  - Display
tags:
  - Android
  - Graphics
  - Display
toc: true
abbrlink: 20180408
date: 2018-04-08 09:25:00
---



Android系统图形框架由下往上主要的包括HAL(HWComposer和Gralloc两个moudle)，SurfaceFlinger（BufferQueue的消费者），WindowManagerService（窗口管理者），View（BufferQueue的生产者）四大模块。
● HAL: 包括HWComposer和Gralloc两个moudle，Android N上由SurfaceFlinger打开，因此在同一进程。 gralloc 用于BufferQueue的内存分配，同时也有fb的显示接口，HWComposer作为合成SurfaceFlinger里面的Layer，并显示（通过gralloc的post函数）
● SurfaceFlinger可以叫做LayerFlinger，作为Layer的管理者，同是也是BufferQueue的消费者，当每个Layer的生产者draw完完整的一帧时，会通知SurfaceFlinger，通知的方式采用BufferQueue。
● WindowManagerService: 作为Window的管理者，掌管着计算窗口大小，窗口切换等任务，同时也会将相应的参数设置给SurfaceFlinger，比如Window的在z-order，和窗口的大小。
● View: 作为BufferQueue的生产者，每当执行lockCanvas->draw->unlockCanvas，之后会存入一帧数据进入BufferQueue中。 嘿嘿(_^▽^_),但这也是无可奈何的事情，毕竟世上不可能有那么多十全十美的好事，做人在某些时候总是要有些取舍的。 <!-- more -->

## 源码（部分）：

**/frameworks/native/services/surfaceflinger/**

- tests/Transaction_test.cpp
- tests/vsync/vsync.cpp

**/frameworks/native/include/gui/**

- BitTube.h
- BufferSlot.h
- BufferQueueCore.h
- BufferQueueProducer.h

**/frameworks/base/core/java/android/app/**

- Activity.java
- ActivityThread.java
- Instrumentation.java

**/frameworks/base/core/jni/**

- android_view_DisplayEventReceiver.cpp
- android_view_SurfaceControl.cpp
- android_view_Surface.cpp
- android_view_SurfaceSession.cpp

**/frameworks/native/include/gui/**

- SurfaceComposerClient.h
- IDisplayEventConnection.h
- SurfaceComposerClient.h

**/frameworks/native/services/surfaceflinger/**

- SurfaceFlinger.cpp
- Client.cpp
- main_surfaceflinger.cpp
- DisplayDevice.cpp
- DispSync.cpp
- EventControlThread.cpp
- EventThread.cpp
- Layer.cpp
- MonitoredProducer.cpp

**/frameworks/base/core/java/android/view/**

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

**/frameworks/native/include/ui/**

- GraphicBuffer.h
- GraphicBufferAllocator.h

**/frameworks/base/services/core/java/com/android/server/wm/**

- WindowManagerService.java
- Session.java
- WindowState.java
- WindowStateAnimator.java
- WindowSurfaceController.java

## （一）、Android Graphics 系统框架

（试用限制？？？万恶的亿图(EDraw)强加水印~火~） 
![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.graphics/01-Android-Graphics-Architecture.png)

**App** 基于Android系统的GUI框架开发完整的Apk应用。

**Android Graphics Stack Client(SurfaceFlinger Client)**
Android在客户端的绘图堆栈通常包括： OpenGL ES：使用GPU进行3D和2D的绘图的API EGL：衔接GLES和系统的Native Window系统的适配层 Vulkan：Vulkan为Khronos Group推出的下一代跨平台图形开发接口，用于替代历史悠久的OpenGL。Android从7.0(Nougat)开始加入了对其的支持。Vulkan与OpenGL相比，接口更底层，从而使开发者能更直接地控制GPU。由于更好的并行支持，及更小的开销，性能上也有一定的提升。
**Android Graphics Stack Server（SurfaceFlinger Server）**
SurfaceFlinger是Android用于管理Display和负责Window Composite（窗口混合），把应用的显示窗口输出到Display的系统服务。

**Android Drivers（HAL）**
Android的驱动层，通过Android本身的HAL（硬件抽象层）机制，运行于User Space，跟渲染相关的包括：

Hwcomposer：如果硬件支持，SurfaceFlinger可以请求hwcomposer去做窗口混合而不需要自己来做，这样的效率也会更高，减少对GPU资源的占用 Gralloc：用来管理Graphics Buffer的分配和管理系统的framebuffer OpenGL ES/EGL

**Linux Kernel and Drivers**
除了标准的Linux内核和驱动（例如fb是framebuffer驱动），硬件厂商自己的驱动外，Android自己的一些Patches：

Ashmem：异步共享内存，用于在进程间共享一块内存区域，并允许系统在资源紧张时回收不加锁的内存块 ION：内存管理器 ION是google在Android4.0 为了解决内存碎片管理而引入的通用内存管理器,在面向程序员编程方面，它和ashmem很相似。但ION比ashmem更强大 Binder：高效的进程间通信机制 Vsync：Android 4.1引入了Vsync(Vertical Syncronization)用于渲染同步，使得App UI和SurfaceFlinger可以按硬件产生的VSync节奏来进行工作 **Hardware** Display（显示器）、CPU、GPU、VPU（Video Process Unit）、和内存等等

## （二）、Android Graphics 测试程序（C++）

为了便于观察对原生测试程序显示图像大小做了如下修改：

> frameworks/native/services/surfaceflinger/tests/Transaction_test.cpp

[Disable_HWUI_GPU_HWC.patch](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/others/Android-7.1.2-Qualcomm-MSM89XX-Disable_HWUI_GPU_HWC)

原生SurfaceFlinger测试程序编译：
1、编译Android 7.1.2源码-userdebug版本，烧录重启
2、编译/frameworks/native/services/surfaceflinger/tests/会生成SurfaceFlinger_test
3、连接手机执行命令 adb root、adb remount、adb push SurfaceFlinger_test /system/bin/
4、adb shell setenforce 0(暂时关闭SELinux权限)
5、adb shell、cd system/bin/、chmod 0777 SurfaceFlinger_test
6、运行测试程序：./SurfaceFlinger_test

![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.graphics/02-Android-graphics-SurfaceFlinger-Test-00.gif)

可以看到在Android 显示屏接替绘制了多个图像，并且会变换形状、位置、颜色、透明度等。 我们先看一下主要步骤： 1、 创建SurfaceComposerClient

```c
sp<SurfaceComposerClient> mComposerClient;
mComposerClient = new SurfaceComposerClient;
```

2、 客户端SurfaceComposerClient请求SurfaceFlinger创建Surface 注：App端对应SurfaceControl<--->SurfaceFlinger对应Layer

```c
sp<SurfaceControl> mBGSurfaceControl;
sp<SurfaceControl> mFGSurfaceControl;
       // Background surface
mBGSurfaceControl = mComposerClient->createSurface(
                String8("BG Test Surface"), displayWidth, displayHeight,
                PIXEL_FORMAT_RGBA_8888, 0);

fillSurfaceRGBA8(mBGSurfaceControl, 63, 63, 195);
```

![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.graphics/03-Android-Graphics-SF-test-color-blue.png)

```c
// Foreground surface
mFGSurfaceControl = mComposerClient->createSurface(
        String8("FG Test Surface"), 64, 64, PIXEL_FORMAT_RGBA_8888, 0);
fillSurfaceRGBA8(mFGSurfaceControl, 195, 63, 63);
```

![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.graphics/04-Android-Graphics-SF-test-color-red.png)

3、处理事务，将SurfaceControl（App）的变化更新到Layer（SurfaceFlinger）图层

```c
 SurfaceComposerClient::openGlobalTransaction();

    mComposerClient->setDisplayLayerStack(display, 0);

    ASSERT_EQ(NO_ERROR, mBGSurfaceControl->setLayer(INT_MAX-2));
    ASSERT_EQ(NO_ERROR, mBGSurfaceControl->show());

    ASSERT_EQ(NO_ERROR, mFGSurfaceControl->setLayer(INT_MAX-1));
    ASSERT_EQ(NO_ERROR, mFGSurfaceControl->setPosition(64, 64));
    ASSERT_EQ(NO_ERROR, mFGSurfaceControl->show());

    SurfaceComposerClient::closeGlobalTransaction(true);
```

4、接受Vsync同步信号，渲染合成，推送到显示屏显示 
![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.graphics/05-Android-Graphics-SF-test-frame_01_delay-1s.gif)

接下来开始Android Graphics系统神秘探索之谜。

## （三）、Android Graphics 禁用hwc和GPU

### 3.1、Disable_HWUI_GPU_HWC

注：基于Android 7.1.2 Qualcomm MSM89XX源码

[Disable_HWUI_GPU_HWC.patch](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/others/Android-7.1.2-Qualcomm-MSM89XX-Disable_HWUI_GPU_HWC)

编译userdebug版本，烧录开机: 运行测试程序：./SurfaceFlinger_test 结果跟上述一致，这里不再贴图了。

### 3.2、Vsync测试程序

Vsync(Vertical Syncronization)用于渲染同步，使得App UI和SurfaceFlinger可以按硬件产生的VSync节奏来进行工作。

查看frameworks/native/services/surfaceflinger/tests/下还有vsync测试程序

```c
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

```c
event vsync: count=2631 16.168612 ms (61.848231 Hz)
event vsync: count=2632 16.168613 ms (61.848224 Hz)
event vsync: count=2633 16.168312 ms (61.849378 Hz)
event vsync: count=2634 16.168682 ms (61.847961 Hz)
event vsync: count=2635 16.168596 ms (61.848288 Hz)
event vsync: count=2636 16.168867 ms (61.847255 Hz)
```

## （四）、Android SurfaceFlinger 内部机制

### 4.1、APP与SurfaceFlinger的数据结构

![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.graphics/06-Android-Graphics-App-SurfaceFlinger.png)

#### 4.1.1、BufferQueue介绍

BufferQueue 类是 Android 中所有图形处理操作的核心。它的是将生成图形数据缓冲区的一方（生产者Producer）连接到接受数据以进行显示或进一步处理的一方（消费者Consumer）。几乎所有在系统中移动图形数据缓冲区的内容都依赖于 BufferQueue。 从上图APP与SurfaceFlinger交互中可以看出，BufferQueue内部维持着64个BufferSlot，每一个BufferSlot内部有一个GraphicBuffer指向分配的Graphic Buffer。 
![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.graphics/07-Android-Graphics-SurfaceFlinger-BufferQueue.png.png)
先来看一下图中几个状态代表的含义：

```c
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

**FREE :** FREE表示缓冲区可由生产者（Producer）DEQUEUED出列。 该BufferSlot由BufferQueue"拥有"。 它转换到DEQUEUED 当调用dequeueBuffer时。

**DEQUEUED：** DEQUEUED表示缓冲区已经被生产者（Producer）出列，但是尚未queued 或canceled。生产者（Producer）可以修改缓冲区的内容一旦相关的释放围栏被发信号通知。BufferSlot由Producer"拥有"。 它可以转换到QUEUED（通过 queueBuffer或者attachBuffer）或者返回FREE（通过cancelBuffer或者detachBuffer）。

**QUEUED：** QUEUED表示缓冲区已经被生产者（Producer）填充排队等待消费者（Consumer）使用。 缓冲区内容可能被继续 修改在有限的时间内，所以内容不能被访问，直到关联的栅栏fence发信号。 该BufferSlot由BufferQueue"拥有"。 它 可以转换为ACQUIRED（通过acquireBuffer）或FREE（如果是另一个缓冲区以异步模式排队）。

**ACQUIRED：** ACQUIRED表示缓冲区已被消费者（Consumer）获取。 如与QUEUED，内容不能被消费者访问，直到 获得栅栏fence信号。 BufferSlot由Consumer"拥有"。 它当releaseBuffer（或detachBuffer）被调用时转换为FREE。 一个 分离的缓冲区也可以通过attachBuffer进入ACQUIRED状态。

**SHARED：** SHARED表示此缓冲区正在共享缓冲区中使用模式。 它可以同时在其他State的任何组合， 除了FREE （因为这不包括在任何其他State）。 它可以也可以出列，排队或多次获得。

**简单描述一下状态转换过程：**

1、首先生产者dequeue过来一块Buffer，此时该buffer的状态为DEQUEUED，所有者为PRODUCER，生产者可以填充数据了。在没有dequeue操作时，buffer的状态为free,所有者为BUFFERQUEUE。

2、生产者填充完数据后,进行queue操作，此时buffer的状态由DEQUEUED->QUEUED的转变，buffer所有者也变成了BufferQueue了。

3、上面已经通知消费者去拿buffer了，这个时候消费者就进行acquire操作将buffer拿过来，此时buffer的状态由QUEUED->ACQUIRED转变，buffer的拥有者由BufferQueue变成Consumer。

4、当消费者已经消费了这块buffer(已经合成，已经编码等)，就进行release操作释放buffer,将buffer归还给BufferQueue,buffer状态由ACQUIRED变成FREE.buffer拥有者由Consumer变成BufferQueue.

#### 4.1.2、生产者Producer

生产者Producer实现IGraphicBufferProducer的接口，在实际运作过程中，应用（Client端）存在代理端BpGraphicBufferProducer，SurfaceFlinger（Server端）存在Native端BnGraphicBufferProducer。生产者代理端Bp通过Binder通信，不断的dequeueBuffer和queueBuffer操作，Native端同样响应这些操作请求，这样buffer就转了起来了。 
![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.graphics/08-Android-Graphics-SurfaceFlinger-IGraphicsBufferProducer.png)

这里介绍几个非常重要的函数： **1、requestBuffer** requestBuffer为给定的索引请求一个新的Buffer。 服务器（即IGraphicBufferProducer实现）分配新创建的Buffer到给定的BufferSlot槽索引，并且客户端可以镜像slot->Buffer映射，这样就没有必要传输一个GraphicBuffer用于每个出队操作。

```c
    // requestBuffer requests a new buffer for the given index. The server (i.e.
    // the IGraphicBufferProducer implementation) assigns the newly created
    // buffer to the given slot index, and the client is expected to mirror the
    // slot->buffer mapping so that it's not necessary to transfer a
    // GraphicBuffer for every dequeue operation.
    //
    // The slot must be in the range of [0, NUM_BUFFER_SLOTS).
    virtual status_t requestBuffer(int slot, sp<GraphicBuffer>* buf) = 0;
```

**2、dequeueBuffer** dequeueBuffer请求一个新的Buffer Slot供客户端使用。 插槽的所有权被转移到客户端，这意味着服务器不会使用与该插槽关联的缓冲区的内容。

```c
    // dequeueBuffer requests a new buffer slot for the client to use. Ownership
    // of the slot is transfered to the client, meaning that the server will not
    // use the contents of the buffer associated with that slot.
    //
    virtual status_t dequeueBuffer(int* slot, sp<Fence>* fence, uint32_t w,
            uint32_t h, PixelFormat format, uint32_t usage) = 0;
```

**3、detachBuffer** detachBuffer尝试删除给定buffer 的所有权插槽从buffer queue。 如果这个请求成功，该slot将会被free，并且将无法从这个接口获得缓冲区。释放的插槽将保持未分配状态，直到被选中为止在dequeueBuffer中保存一个新分配的缓冲区，或者附加一个缓冲区到插槽。 缓冲区必须已经被取出，并且调用者必须已经拥有sp

<graphicbuffer>（即必须调用requestBuffer）</graphicbuffer>

```c
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

**4、attachBuffer** attachBuffer尝试将缓冲区的所有权转移给缓冲区队列。 如果这个调用成功，就好像这个缓冲区已经出队一样从返回的插槽号码。 因此，如果连接，这个调用将失败这个缓冲区会导致很多的缓冲区同时出队。

```c
    // attachBuffer attempts to transfer ownership of a buffer to the buffer
    // queue. If this call succeeds, it will be as if this buffer was dequeued
    // from the returned slot number. As such, this call will fail if attaching
    // this buffer would cause too many buffers to be simultaneously dequeued.
    //
    virtual status_t attachBuffer(int* outSlot,
            const sp<GraphicBuffer>& buffer) = 0;
```

#### 4.1.3、消费者Consumer

![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.graphics/09-Android-Graphics-SurfaceFlinger-IGraphicsBufferConsumer.png)
这里介绍几个非常重要的函数： **1、acquireBuffer** acquireBuffer尝试获取下一个未决缓冲区的所有权BufferQueue。 如果没有缓冲区等待，则返回NO_BUFFER_AVAILABLE。 如果缓冲区被成功获取，有关缓冲区的信息将在BufferItem中返回。

```c
    // acquireBuffer attempts to acquire ownership of the next pending buffer in
    // the BufferQueue.  If no buffer is pending then it returns
    // NO_BUFFER_AVAILABLE.  If a buffer is successfully acquired, the
    // information about the buffer is returned in BufferItem.
    //
    virtual status_t acquireBuffer(BufferItem* buffer, nsecs_t presentWhen,
            uint64_t maxFrameNumber = 0) = 0;
```

**2、releaseBuffer** releaseBuffer从消费者释放一个BufferSlot回到BufferQueue。 这可以在缓冲区的内容仍然存在时完成被访问。 栅栏将在缓冲区不再正在使用时发出信号。 frameNumber用于标识返回的确切缓冲区。

```c
    // releaseBuffer releases a buffer slot from the consumer back to the
    // BufferQueue.  This may be done while the buffer's contents are still
    // being accessed.  The fence will signal when the buffer is no longer
    // in use. frameNumber is used to indentify the exact buffer returned.
    //
    virtual status_t releaseBuffer(int buf, uint64_t frameNumber,
            EGLDisplay display, EGLSyncKHR fence,
            const sp<Fence>& releaseFence) = 0;
```

**3、detachBuffer** detachBuffer尝试删除给定缓冲区的所有权插槽从缓冲区队列。 如果这个请求成功，该插槽将会是释放，并且将无法从这个接口获得缓冲区。释放的插槽将保持未分配状态，直到被选中为止在dequeueBuffer中保存一个新分配的缓冲区，或者附加一个缓冲区到slot。 缓冲区必须已被acquired。

```c
    // detachBuffer attempts to remove all ownership of the buffer in the given
    // slot from the buffer queue. If this call succeeds, the slot will be
    // freed, and there will be no way to obtain the buffer from this interface.
    // The freed slot will remain unallocated until either it is selected to
    // hold a freshly allocated buffer in dequeueBuffer or a buffer is attached
    // to the slot. The buffer must have already been acquired.
    //
    virtual status_t detachBuffer(int slot) = 0;
```

**4、attachBuffer** attachBuffer尝试将缓冲区的所有权转移给缓冲区队列。 如果这个调用成功，就好像这个缓冲区被获取了一样从返回的插槽号码。 因此，如果连接，这个调用将失败这个缓冲区会导致太多的缓冲区被同时acquired。

```c
    // attachBuffer attempts to transfer ownership of a buffer to the buffer
    // queue. If this call succeeds, it will be as if this buffer was acquired
    // from the returned slot number. As such, this call will fail if attaching
    // this buffer would cause too many buffers to be simultaneously acquired.
    //
    virtual status_t attachBuffer(int *outSlot,
            const sp<GraphicBuffer>& buffer) = 0;
```

## 4.2、App（Java层）请求创建Surface过程

### 4.2.1、Activity启动流程

Activity创建过程这里不再叙述。 请参考[【Android 7.1.2 (Android N) Activity启动流程分析】]() && [【Android 7.1.2 (Android N) Activity-Window加载显示流程分析】]()

> ● ActivityManagerService接收启动Activity的请求

> Activity.startActivity()
Activity.startActivityForResult()
Instrumentation.execStartActivity()
ActivityManagerProxy.startActivity()
ActivityManagerNative.onTransact()
ActivityManagerService.startActivity()
ActivityStarter.startActivityMayWait()
ActivityStarter.startActivityLocked()
ActivityStarter.startActivityUnchecked()
ActivityStackSupervisor.resumeFocusedStackTopActivityLocked() ActivityStack.resumeTopActivityUncheckedLocked()
ActivityStack.resumeTopActivityInnerLocked()
ActivityStackSupervisor.startSpecificActivityLocked()

> ●● 创建Activity所属的应用进程 ActivityManagerService.startProcessLocked()

> ●●● Zygote通过socket通信fork一个新的进程，并根据"android.app.ActivityThread"字符串
  ●●● 反射出该对象并执行ActivityThread的main方法
  ActivityThread.main()
  ActivityThread.attach()
  ActivityManagerProxy.attachApplication()
  ActivityManagerNative.onTransact()
  ActivityManagerService.attachApplication()
  ActivityManagerService.attachApplicationLocked()
  ActivityStackSupervisor.attachApplicationLocked()
  ActivityStackSupervisor.realStartActivityLocked()

> ●●●● 执行启动Acitivity
IApplicationThread.scheduleLaunchActivity()
ActivityThread.ApplicationThread.scheduleLaunchActivity()
ActivityThread.sendMessage() ActivityThread.H.handleMessage()
ActivityThread.handleLauncherActivity()
ActivityThread.performLauncherActivity()
ActivityThread.handleResumeActivity()

##### 4.2.2、Window加载显示流程

> 画图，需要重新分析一下下，嘿嘿(_^▽^_)~

##### 4.2.2.1、ActivityThread.handleLaunchActivity()

> 接着从ActivityThread的handleLaunchActivity方法：

```java
    [->ActivityThread.java]
    private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason){
    ......
    //创建Activity  
    Activity a = performLaunchActivity(r, customIntent);

    if (a != null) {
        ......
        //启动Activity  
        handleResumeActivity(r.token, false, r.isForward,
                !r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);

        ......
    }
}
```

### 4.2.2.2、ActivityThread.handleResumeActivity()

回到我们刚刚的handleLaunchActivity()方法，在调用完performLaunchActivity()方法之后，其有掉用了handleResumeActivity()法。performLaunchActivity()方法完成了两件事： 1) Activity窗口对象的创建，通过attach函数来完成； 2) Activity视图对象的创建，通过setContentView函数来完成； 这些准备工作完成后，就可以显示该Activity了，应用程序进程通过调用handleResumeActivity函数来启动Activity的显示过程。 [->ActivityThread.java]

```java
    final void handleResumeActivity(IBinder token,
        boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
    ActivityClientRecord r = mActivities.get(token);
    ......
    r = performResumeActivity(token, clearHide, reason);
    ......
        if (r.window == null && !a.mFinished && willBeVisible) {
            //获得为当前Activity创建的窗口PhoneWindow对象
            r.window = r.activity.getWindow();
            //获取为窗口创建的视图DecorView对象
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            //在attach函数中就为当前Activity创建了WindowManager对象  
            ViewManager wm = a.getWindowManager();
            //得到该视图对象的布局参数  
            WindowManager.LayoutParams l = r.window.getAttributes();
            //将视图对象保存到Activity的成员变量mDecor中  
            a.mDecor = decor;
            l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
            l.softInputMode |= forwardBit;
            ......
            if (a.mVisibleFromClient && !a.mWindowAdded) {
                a.mWindowAdded = true;
                //将创建的视图对象DecorView添加到Activity的窗口管理器中  
                wm.addView(decor, l);
            }
        ......
        }
    }
}
```

在前面的performLaunchActivity函数中完成Activity的创建后，会将当前当前创建的Activity在应用程序进程端的描述符ActivityClientRecord以键值对的形式保存到ActivityThread的成员变量mActivities中：mActivities.put(r.token, r)，r.token就是Activity的身份证，即是IApplicationToken.Proxy代理对象，也用于与AMS通信。上面的函数首先通过performResumeActivity从mActivities变量中取出Activity的应用程序端描述符ActivityClientRecord，然后取出前面为Activity创建的视图对象DecorView和窗口管理器WindowManager，最后将视图对象添加到窗口管理器中。 ViewManager.addView()真正实现的的地方在WindowManagerImpl.java中。

```java
public interface ViewManager
{
public void addView(View view, ViewGroup.LayoutParams params);
......
}
```

[->WindowManagerImpl.java]

```java
Override
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    ......
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}
```

[->WindowManagerGlobal.java]

```java
    public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {
    ......
    ViewRootImpl root;
    View panelParentView = null;
    synchronized (mLock) {
        ......
        root = new ViewRootImpl(view.getContext(), display);
        view.setLayoutParams(wparams);
        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);
    }
    try {
        root.setView(view, wparams, panelParentView);
    }
    ......
}
```

### 4.2.2.3、ViewRootImpl()构造过程：

[ViewRootImpl.java # ViewRootImpl()]

```java
    final W mWindow;
    final Surface mSurface = new Surface();
    final ViewRootHandler mHandler = new ViewRootHandler();
    ......
    public ViewRootImpl(Context context, Display display) {
    mContext = context;
    mWindowSession = WindowManagerGlobal.getWindowSession();//IWindowSession的代理对象，该对象用于和WMS通信。
    mDisplay = display;
    ......
    mWindow = new W(this);//创建了一个W本地Binder对象，用于WMS通知应用程序进程
    ......
    mAttachInfo = new View.AttachInfo(mWindowSession, mWindow, display, this, mHandler, this);
    ......
    mViewConfiguration = ViewConfiguration.get(context);
    mDensity = context.getResources().getDisplayMetrics().densityDpi;
    mNoncompatDensity = context.getResources().getDisplayMetrics().noncompatDensityDpi;
    mFallbackEventHandler = new PhoneFallbackEventHandler(context);
    mChoreographer = Choreographer.getInstance();//Choreographer对象
    ......
}
```

在ViewRootImpl的构造函数中初始化了一些成员变量，ViewRootImpl创建了以下几个主要对象：

(1) 通过WindowManagerGlobal.getWindowSession()得到IWindowSession的代理对象，该对象用于和WMS通信。

(2) 创建了一个W本地Binder对象，用于WMS通知应用程序进程。

(3) 采用单例模式创建了一个Choreographer对象，用于统一调度窗口绘图。

(4) 创建ViewRootHandler对象，用于处理当前视图消息。

(5) 构造一个AttachInfo对象；

●●●(6) 创建Surface对象，用于绘制当前视图，当然该Surface对象的真正创建是由WMS来完成的，只不过是WMS传递给应用程序进程的。 
![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.graphics/10-Android-Graphics-App-WMS-Surface.png.png)

### 4.2.2.4、IWindowSession代理获取过程

[->WindowManagerGlobal.java]

```java
   private static IWindowSession sWindowSession;
    public static IWindowSession getWindowSession() {
    synchronized (WindowManagerGlobal.class) {
        if (sWindowSession == null) {
            try {
            ......
                //得到IWindowSession代理对象
                sWindowSession = windowManager.openSession(
                        new IWindowSessionCallback.Stub() {
                            @Override
                            public void onAnimatorScaleChanged(float scale) {
                                ValueAnimator.setDurationScale(scale);
                            }
                        },
                        imm.getClient(), imm.getInputContext());
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        }
        return sWindowSession;
    }
}
```

以上函数通过WMS的openSession函数创建应用程序与WMS之间的连接通道，即获取IWindowSession代理对象，并将该代理对象保存到ViewRootImpl的静态成员变量sWindowSession中,因此在应用程序进程中有且只有一个IWindowSession代理对象。 [->WindowManagerService.java]

```java
Override
public IWindowSession openSession(IWindowSessionCallback callback, IInputMethodClient client,
        IInputContext inputContext) {
    if (client == null) throw new IllegalArgumentException("null client");
    if (inputContext == null) throw new IllegalArgumentException("null inputContext");
    Session session = new Session(this, callback, client, inputContext);
    return session;
}
```

在WMS服务端构造了一个Session实例对象。ViewRootImpl 是一很重要的类，类似 ActivityThread 负责跟AmS通信一样，ViewRootImpl 的一个重要职责就是跟 WmS 通信，它通静态变量 sWindowSession（IWindowSession实例）与 WmS 进行通信。每个应用进程，仅有一个 sWindowSession 对象，它对应了 WmS 中的 Session 子类，WmS 为每一个应用进程分配一个 Session 对象。WindowState 类有一个 IWindow mClient 参数，是在构造方法中赋值的，是由 Session 调用 addWindow 传递过来了，对应了 ViewRootImpl 中的 W 类的实例。

### 4.2.2.5、视图View添加过程ViewRootImpl.setView()

窗口管理器WindowManagerImpl为当前添加的窗口创建好各种对象后，调用ViewRootImpl的setView函数向WMS服务添加一个窗口对象。 [->ViewRootImpl.java]

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
        if (mView == null) {
            ////将DecorView保存到ViewRootImpl的成员变量mView中
            mView = view;
            ......
            //1）在添加窗口前进行UI布局  
            requestLayout();
            ......
            try {
                ......
                 //2)将窗口添加到WMS服务中，mWindow为W本地Binder对象，通过Binder传输到WMS服务端后，变为IWindow代理对象  
                res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                        getHostVisibility(), mDisplay.getDisplayId(),
                        mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                        mAttachInfo.mOutsets, mInputChannel);
            } ......
            //3)建立窗口消息通道  
            if (mInputChannel != null) {
                if (mInputQueueCallback != null) {
                    mInputQueue = new InputQueue();
                    mInputQueueCallback.onInputQueueCreated(mInputQueue);
                }
                mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
                        Looper.myLooper());
            }
            ......
        }
    }
}
```

通过前面的分析可以知道，用户自定义的UI作为一个子View被添加到DecorView中，然后将顶级视图DecorView添加到应用程序进程的窗口管理器中，窗口管理器首先为当前添加的View创建一个ViewRootImpl对象、一个布局参数对象ViewGroup.LayoutParams，然后将这三个对象分别保存到当前应用程序进程的窗口管理器WindowManagerImpl中，最后通过ViewRootImpl对象将当前视图对象注册到WMS服务中。

ViewRootImpl的setView函数向WMS服务添加一个窗口对象过程： (1) requestLayout()在应用程序进程中进行窗口UI布局； (2) WindowSession.addToDisplay()向WMS服务注册一个窗口对象； (3) 注册应用程序进程端的消息接收通道；

### 4.2.2.6、requestLayout()在应用程序进程中进行窗口UI布局；

2.10、窗口UI布局过程 requestLayout函数调用里面使用了Hanlder的一个小手段，那就是利用postSyncBarrier添加了一个Barrier（挡板），这个挡板的作用是阻塞普通的同步消息的执行，在挡板被撤销之前，只会执行异步消息，而requestLayout先添加了一个挡板Barrier，之后自己插入了一个异步任务mTraversalRunnable，其主要作用就是保证mTraversalRunnable在所有同步Message之前被执行，保证View绘制的最高优先级。具体实现如下：

```java
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

这里暂时不讨论Choreographer和Vsync知识，稍后再详细分析。 现在先说出结论：Choreographer构造函数中，构造了一个FrameDisplayEventReceiver对象，用于请求并接收Vsync信号。 此时FrameDisplayEventReceiver会Call requestNextVsync()来告诉系统我要在下一个VSYNC需要被trigger。Vsync信号每隔16ms一次，此时Vsync信号还未来到，继续分析mWindowSession.addToDisplay()。

### 4.2.2.7、mWindowSession.addToDisplay()向WMS服务注册一个窗口对象；

[Session.java]

```java
Override
public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
        int viewVisibility, int displayId, Rect outContentInsets, Rect outStableInsets,
        Rect outOutsets, InputChannel outInputChannel) {
    return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId,
            outContentInsets, outStableInsets, outOutsets, outInputChannel);
}
```

[WindowManagerService.java]

```java
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

构造一个WindowState对象，并将添加的窗口信息记录到mTokenMap和mWindowMap哈希表中。 在WMS服务端创建了所需对象后，接着调用了WindowState的attach()来进一步完成窗口添加。 [WindowState.java]

```java
void attach() {
    if (WindowManagerService.localLOGV) Slog.v(TAG, "Attaching " + this + " token=" + mToken
        + ", list=" + mToken.windows);
    mSession.windowAddedLocked();
}
```

[Session.java]

```java
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

### 4.2.2.8、SurfaceSession建立过程

SurfaceSession对象承担了应用程序与SurfaceFlinger之间的通信过程，每一个需要与SurfaceFlinger进程交互的应用程序端都需要创建一个SurfaceSession对象。

客户端请求 [SurfaceSession.java]

```java
public SurfaceSession() {
    mNativeClient = nativeCreate();
}
```

Java层的SurfaceSession对象构造过程会通过JNI在native层创建一个SurfaceComposerClient对象。 [android_view_SurfaceSession.cpp]

```java
static jlong nativeCreate(JNIEnv* env, jclass clazz) {
SurfaceComposerClient* client = new SurfaceComposerClient();
client->incStrong((void*)nativeCreate);
return reinterpret_cast<jlong>(client);
}
```

Java层的SurfaceSession对象与C++层的SurfaceComposerClient对象之间是一对一关系。 是否似曾相识，就是前面最开始SurfaceFlinger_Test程序第一步：new SurfaceComposerClient的过程。 [SurfaceComposerClient.cpp]

```java
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

服务端处理 在SurfaceFlinger服务端为应用程序创建交互的Client对象 [SurfaceFlinger.cpp]

```java
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

### 4.2.3、App（c层）请求创建SurfaceFlinger客户端(client)的过程

![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.graphics/11-Android-Graphics-App-Ask-SurfaceFlinger-Create-Client.png.png)

继续详细分析AppApp（c层）请求创建SurfaceFlinger客户端(client)的过程

SurfaceComposerClient第一次强引用时，会执行onFirstRef() [SurfaceComposerClient.cpp]

```c
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

第一步：获取"SurfaceFlinger"服务 ComposerService::getComposerService()

```c
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

```c
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

所以前面instance.mComposerService其实返回的是"SurfaceFlinger"服务。 第二步：createConnection() 接下来就会调用"SurfaceFlinger"服务的createConnection()

```c
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

### 4.2.4、APP申请创建Surface过程

前面讲ViewRootImpl.setView时，requestLayout()需要Vsync trigger，加入现在Vsync信号来到，于是会继续执行。App申请创建Surface的过程就在其中。

当Vsync事件到来时，就会通过Choreographer的postCallback()，接着执行mTraversalRunnable对象的run()方法。 mTraversalRunnable对象的类型为TraversalRunnable，该类实现了Runnable接口，在其run()函数中调用了doTraversal()函数来完成窗口布局。 [->ViewRootImpl.java]

```java
final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}
final TraversalRunnable mTraversalRunnable = new TraversalRunnable();
void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
        ......
        performTraversals();
        ......
    }
}
```

performTraversals函数相当复杂，其主要实现以下几个重要步骤：

1.执行窗口测量；

2.执行窗口注册；

3.执行窗口布局；

4.执行窗口绘图；

[->ViewRootImpl.java]

```java
    private void performTraversals() {
    ......
    /****************执行窗口测量******************/  
    if (layoutRequested) {
        windowSizeMayChange |= measureHierarchy(host, lp, res,
                desiredWindowWidth, desiredWindowHeight);
    }
    ......
    /****************向WMS服务添加窗口******************/  
    if (mFirst || windowShouldResize || insetsChanged ||
            viewVisibilityChanged || params != null || mForceNextWindowRelayout) {
        ......
        try {
            ......
            relayoutResult = relayoutWindow(params, viewVisibility, insetsPending);
            ......
        }
        ......
    }
    /****************执行窗口布局******************/
    if (didLayout) {
        performLayout(lp, mWidth, mHeight);
        ......
        }

    /****************执行窗口绘制******************/  
    if (!cancelDraw && !newSurface) {
        ......
        performDraw();
    } ......
}
```

1、执行窗口测量performMeasure() [->ViewRootImpl.java]

```java
    private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
    try {
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    } ......
}
```

#### 4.2.4.1、APP申请创建Surface过程(Java层)

2、执行窗口注册relayoutWindow； [->ViewRootImpl.java]

```java
    private int relayoutWindow(WindowManager.LayoutParams params, int viewVisibility,
        boolean insetsPending) throws RemoteException {

    ......
    int relayoutResult = mWindowSession.relayout(
            mWindow, mSeq, params,
            (int) (mView.getMeasuredWidth() * appScale + 0.5f),
            (int) (mView.getMeasuredHeight() * appScale + 0.5f),
            viewVisibility, insetsPending ? WindowManagerGlobal.RELAYOUT_INSETS_PENDING : 0,
            mWinFrame, mPendingOverscanInsets, mPendingContentInsets, mPendingVisibleInsets,
            mPendingStableInsets, mPendingOutsets, mPendingBackDropFrame, mPendingConfiguration,
            mSurface);

    ......
    return relayoutResult;
}
```

这里通过前面获取的IWindowSession代理对象请求WMS服务执行窗口布局，mSurface是ViewRootImpl的成员变量 [->ViewRootImpl.java]

```java
final Surface mSurface = new Surface();
```

[->Surface.java]

```java
/**
 * Create an empty surface, which will later be filled in by readFromParcel().
 * @hide
 */
public Surface() {
}
```

该Surface构造函数仅仅创建了一个空Surface对象，并没有对该Surface进程native层的初始化，到此我们知道应用程序进程为每个窗口对象都创建了一个Surface对象。并且将该Surface通过跨进程方式传输给WMS服务进程，我们知道，在Android系统中，如果一个对象需要在不同进程间传输，必须实现Parcelable接口，Surface类正好实现了Parcelable接口。ViewRootImpl通过IWindowSession接口请求WMS的完整过程如下： [->IWindowSession.java$ Proxy]

```java
/*
* This file is auto-generated.  DO NOT MODIFY
*  * Original file: frameworks/base/core/java/android/view/IWindowSession.aidl
*/
@Override
  public int relayout(android.view.IWindow window, int seq, android.view.WindowManager.LayoutParams attrs, int requestedWidth, int requestedHeight, int viewVisibility, int flags, android.graphics.Rect outFrame, android.graphics.Rect outOverscanInsets, android.graphics.Rect outContentInsets, android.graphics.Rect outVisibleInsets, android.graphics.Rect outStableInsets, android.graphics.Rect outOutsets, android.graphics.Rect outBackdropFrame, android.content.res.Configuration outConfig, android.view.Surface outSurface) throws android.os.RemoteException {
android.os.Parcel _data = android.os.Parcel.obtain();
android.os.Parcel _reply = android.os.Parcel.obtain();
int _result;
try {
    ......
    mRemote.transact(Stub.TRANSACTION_relayout, _data, _reply, 0);
    ......
    if ((0 != _reply.readInt())) {
        outSurface.readFromParcel(_reply);
    }
} finally {
    ......
}
return _result;
}
```

从该函数的实现可以看出，应用程序进程中创建的Surface对象并没有传递到WMS服务进程，只是读取WMS服务进程返回来的Surface。那么WMS服务进程是如何响应应用程序进程布局请求的呢？

[->IWindowSession.java$ Stub]

```java
@Override public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException
{
switch (code)
{
case TRANSACTION_relayout:
  {
    ......
    android.view.Surface _arg15;
    _arg15 = new android.view.Surface();
    int _result = this.relayout(_arg0, _arg1, _arg2, _arg3, _arg4, _arg5, _arg6, _arg7, _arg8, _arg9, _arg10, _arg11, _arg12, _arg13, _arg14, _arg15);
    reply.writeNoException();
    reply.writeInt(_result);
    ......
    if ((_arg15!=null)) {
    reply.writeInt(1);
    _arg15.writeToParcel(reply, android.os.Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
    }
    return true;
  }
}
```

该函数可以看出，WMS服务在响应应用程序进程请求添加窗口时，首先在当前进程空间创建一个Surface对象

```java
android.view.Surface _arg15;
_arg15 = new android.view.Surface();
```

然后调用Session的relayout()函数进一步完成窗口添加过程，最后将WMS服务中创建的Surface返回给应用程序进程。

到目前为止，在应用程序进程和WMS服务进程分别创建了一个Surface对象，但是他们调用的都是Surface的无参构造函数，在该构造函数中并未真正初始化native层的Surface，那native层的Surface是在那里创建的呢？ [->Session.java]

```java
    public int relayout(IWindow window, int seq, WindowManager.LayoutParams attrs,
        int requestedWidth, int requestedHeight, int viewFlags,
        int flags, Rect outFrame, Rect outOverscanInsets, Rect outContentInsets,
        Rect outVisibleInsets, Rect outStableInsets, Rect outsets, Rect outBackdropFrame,
        Configuration outConfig, Surface outSurface) {
    int res = mService.relayoutWindow(this, window, seq, attrs,
            requestedWidth, requestedHeight, viewFlags, flags,
            outFrame, outOverscanInsets, outContentInsets, outVisibleInsets,
            outStableInsets, outsets, outBackdropFrame, outConfig, outSurface);
    return res;
}
```

[->WindowManagerService.java]

```java
    public int relayoutWindow(Session session, IWindow client, int seq,
        WindowManager.LayoutParams attrs, int requestedWidth,
        int requestedHeight, int viewVisibility, int flags,
        Rect outFrame, Rect outOverscanInsets, Rect outContentInsets,
        Rect outVisibleInsets, Rect outStableInsets, Rect outOutsets, Rect outBackdropFrame,
        Configuration outConfig, Surface outSurface) {
    int result = 0;
    ......
        if (viewVisibility == View.VISIBLE &&
                (win.mAppToken == null || !win.mAppToken.clientHidden)) {
            result = relayoutVisibleWindow(outConfig, result, win, winAnimator, attrChanges,
                    oldVisibility);
            try {
                result = createSurfaceControl(outSurface, result, win, winAnimator);
            } catch (Exception e) {
               ......
                return 0;
            }
            ......
        } else {
           ......
    }

    ......
    return result;
}
```

[->WindowManagerService.java]

```java
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

```java
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

#### 4.2.4.1、APP申请创建Surface过程(c层)

SurfaceControl创建过程 [->SurfaceControl.java]

```java
    public SurfaceControl(SurfaceSession session,
        String name, int w, int h, int format, int flags)
                throws OutOfResourcesException {
    ......
    mNativeObject = nativeCreate(session, name, w, h, format, flags);
    ......
}
```

[->android_view_SurfaceControl.cpp]

```c
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
![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.graphics/12-Android-Graphics-App-Ask-SurfaceFlinger-CreateSurface.png)

[->SurfaceComposerClient.cpp]

```c
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

SurfaceComposerClient将Surface创建请求转交给保存在其成员变量中的Bp SurfaceComposerClient对象来完成，在SurfaceFlinger端的Client本地对象会返回一个ISurface代理对象给应用程序，通过该代理对象为应用程序当前创建的Surface创建一个SurfaceControl对象。 [ISurfaceComposerClient.cpp]

```c
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

[Client.cpp] MessageCreateSurface消息是专门为应用程序请求创建Surface而定义的一种消息类型：

```c
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

SurfaceFlinger根据标志位创建对应类型的Surface，当前系统定义了3种类型的Layer: [->ISurfaceComposerClient.h]

```c
eFXSurfaceNormal    = 0x00000000,
eFXSurfaceDim       = 0x00020000,
eFXSurfaceMask      = 0x000F0000
```

[->SurfaceFlinger.cpp]

```c
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
![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.graphics/13-Android-Graphics-App-Ask-SurfaceFlinger-Create-Layer.png.png.png)
第一次强引用Layer对象时，onFirstRef()函数被回调 [Layer.cpp]

```c
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

#### 4.2.4.2、BufferQueue构造过程

[->BufferQueue.cpp]

```c
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

[->BufferQueueCore.cpp] 所以核心都是这个BufferQueueCore，他是管理图形缓冲区的中枢。

```c
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

![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.graphics/14-Android-Graphics-App-SurfaceFlinger-BufferSlot.png.png)

[->ISurfaceComposer.cpp]

```c
   virtual sp<IGraphicBufferAlloc> createGraphicBufferAlloc()
{
    Parcel data, reply;
    data.writeInterfaceToken(ISurfaceComposer::getInterfaceDescriptor());
    remote()->transact(BnSurfaceComposer::CREATE_GRAPHIC_BUFFER_ALLOC, data, &reply);
    return interface_cast<IGraphicBufferAlloc>(reply.readStrongBinder());
}
```

[->SurfaceFlinger.cpp]

```c
sp<IGraphicBufferAlloc> SurfaceFlinger::createGraphicBufferAlloc()
{
sp<GraphicBufferAlloc> gba(new GraphicBufferAlloc());
return gba;
}
```

**GraphicBufferAlloc构造过程**

[->GraphicBufferAlloc.cpp]

```c
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

**图形缓冲区创建过程** [->GraphicBuffer.cpp]

```c
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

使用GraphicBufferAllocator对象来为图形缓冲区分配内存空间，GraphicBufferAllocator是对Gralloc模块中的gpu设备的封装类。关于GraphicBufferAllocator内存分配过程请以后作分析，图形缓冲区分配完成后，还会映射到SurfaceFlinger服务进程的虚拟地址空间。

**Android图形缓冲区分配过程源码分析**

[->Layer.cpp]

```c
Layer::Layer(SurfaceFlinger* flinger, const sp<Client>& client,
    const String8& name, uint32_t w, uint32_t h, uint32_t flags)
:   contentDirty(false),
    sequence(uint32_t(android_atomic_inc(&sSequence))),
    mFlinger(flinger),
    mTextureName(-1U),
    mPremultipliedAlpha(true),
    mName("unnamed"),
    mFormat(PIXEL_FORMAT_NONE),
    ......{
mCurrentCrop.makeInvalid();
mFlinger->getRenderEngine().genTextures(1, &mTextureName);
mTexture.init(Texture::TEXTURE_EXTERNAL, mTextureName);
......
}
```

到此才算真正创建了一个可用于绘图的Surface (Layer)，从上面的分析我们可以看出，在WMS服务进程端，其实创建了两个Java层的Surface对象，第一个Surface使用了无参构造函数，仅仅构造一个Surface对象而已，而第二个Surface却使用了有参构造函数，参数指定了图象宽高等信息，这个Java层Surface对象还会在native层请求SurfaceFlinger创建一个真正能用于绘制图象的native层Surface。最后通过浅拷贝的方式将第二个Surface复制到第一个Surface中，最后通过writeToParcel方式写回到应用程序进程。 
![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.graphics/15-Android-Graphics-App-WMS-SurfaceFlinger-Surface-SurfaceControl.png)

到目前为止，应用程序和WMS一共创建了3个Java层Surface（SurfaceControl）对象，如上图所示，而真正能用于绘图的Surface只有3号，那么3号Surface与2号Surface之间是什么关系呢？outSurface.copyFrom(surface)

[Surface.java]

```java
    public void copyFrom(SurfaceControl other) {
    ......
    long surfaceControlPtr = other.mNativeObject;
    ......
    long newNativeObject = nativeCreateFromSurfaceControl(surfaceControlPtr);

    synchronized (mLock) {
        if (mNativeObject != 0) {
            nativeRelease(mNativeObject);
        }
        setNativeObjectLocked(newNativeObject);
    }
}
```

[android_view_Surface.cpp]

```c
static jlong nativeCreateFromSurfaceControl(JNIEnv* env, jclass clazz,
    jlong surfaceControlNativeObj) {
/*
 * This is used by the WindowManagerService just after constructing
 * a Surface and is necessary for returning the Surface reference to
 * the caller. At this point, we should only have a SurfaceControl.
 */

sp<SurfaceControl> ctrl(reinterpret_cast<SurfaceControl *>(surfaceControlNativeObj));
sp<Surface> surface(ctrl->getSurface());
if (surface != NULL) {
    surface->incStrong(&sRefBaseOwner);
}
return reinterpret_cast<jlong>(surface.get());
}
```

2号Surface引用到了3号Surface的SurfaceControl对象后，通过writeToParcel()函数写会到应用程序进程。 [Surface.java]

```c
        @Override
public void writeToParcel(Parcel dest, int flags) {
    if (dest == null) {
        throw new IllegalArgumentException("dest must not be null");
    }
    synchronized (mLock) {
        // NOTE: This must be kept synchronized with the native parceling code
        // in frameworks/native/libs/Surface.cpp
        dest.writeString(mName);
        dest.writeInt(mIsSingleBuffered ? 1 : 0);
        nativeWriteToParcel(mNativeObject, dest);
    }
    if ((flags & Parcelable.PARCELABLE_WRITE_RETURN_VALUE) != 0) {
        release();
    }
}
```

[android_view_Surface.cpp]

```c
static void nativeWriteToParcel(JNIEnv* env, jclass clazz,
    jlong nativeObject, jobject parcelObj) {
Parcel* parcel = parcelForJavaObject(env, parcelObj);
if (parcel == NULL) {
    doThrowNPE(env);
    return;
}
sp<Surface> self(reinterpret_cast<Surface *>(nativeObject));
android::view::Surface surfaceShim;
if (self != nullptr) {
    surfaceShim.graphicBufferProducer = self->getIGraphicBufferProducer();
}
// Calling code in Surface.java has already written the name of the Surface
// to the Parcel
surfaceShim.writeToParcel(parcel, /*nameAlreadyWritten*/true);
}
```

应用程序进程中的1号Surface按相反顺序读取WMS服务端返回过来的Binder对象等数据，并构造一个native层的Surface对象。

```c
    public void readFromParcel(Parcel source) {
    if (source == null) {
        throw new IllegalArgumentException("source must not be null");
    }

    synchronized (mLock) {
        // nativeReadFromParcel() will either return mNativeObject, or
        // create a new native Surface and return it after reducing
        // the reference count on mNativeObject.  Either way, it is
        // not necessary to call nativeRelease() here.
        // NOTE: This must be kept synchronized with the native parceling code
        // in frameworks/native/libs/Surface.cpp
        mName = source.readString();
        mIsSingleBuffered = source.readInt() != 0;
        setNativeObjectLocked(nativeReadFromParcel(mNativeObject, source));
    }
}
```

应用程序进程中的1号Surface按相反顺序读取WMS服务端返回过来的Binder对象等数据，并构造一个native层的Surface对象。

```c
static jlong nativeCreateFromSurfaceControl(JNIEnv* env, jclass clazz,
    jlong surfaceControlNativeObj) {
/*
 * This is used by the WindowManagerService just after constructing
 * a Surface and is necessary for returning the Surface reference to
 * the caller. At this point, we should only have a SurfaceControl.
 */

sp<SurfaceControl> ctrl(reinterpret_cast<SurfaceControl *>(surfaceControlNativeObj));
sp<Surface> surface(ctrl->getSurface());
if (surface != NULL) {
    surface->incStrong(&sRefBaseOwner);
}
return reinterpret_cast<jlong>(surface.get());
}
```

每个Activity可以有一个或多个Surface，默认情况下一个Activity只有一个Surface，当Activity中使用SurfaceView时，就存在多个Surface。Activity默认surface是在relayoutWindow过程中由WMS服务创建的，然后回传给应用程序进程，我们知道一个Surface其实就是应用程序端的本地窗口，关于Surface的初始化过程这里就不在介绍。

#### 4.2.4.2、生产者Producer构造过程

```c
sp<IGraphicBufferProducer> producer(new BufferQueueProducer(core));
sp<IGraphicBufferConsumer> consumer(new BufferQueueConsumer(core));
```

实例化BufferQueueProducer，这里初始化了mCore(core) 和 mSlots(core->mSlots)

```c
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

#### 4.2.4.3、消费者Consumer构造过程

```c
sp<IGraphicBufferProducer> producer(new BufferQueueProducer(core));
sp<IGraphicBufferConsumer> consumer(new BufferQueueConsumer(core));
```

实例化BufferQueueConsumer，这里初始化了mCore(core) 和 mSlots(core->mSlots)

```c
BufferQueueConsumer::BufferQueueConsumer(const sp<BufferQueueCore>& core) :
    mCore(core),
    mSlots(core->mSlots),
    mConsumerName() {}
```

#### 4.2.4.4、SurfaceFlinger设置监听

```c
mSurfaceFlingerConsumer = new SurfaceFlingerConsumer(consumer, mTextureName,
        this);
mSurfaceFlingerConsumer->setConsumerUsageBits(getEffectiveUsage(0));
mSurfaceFlingerConsumer->setContentsChangedListener(this);
mSurfaceFlingerConsumer->setName(mName);
```

![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.graphics/16-Android-Graphics-App-Ask-SurfaceFlinger-ConsumeLisener-onFrameAvailable.png)

#### 4.2.4.5、应用程序本地窗口Surface创建过程

从前面分析可知，SurfaceFlinger在处理应用程序请求创建Surface中，在SurfaceFlinger服务端仅仅创建了Layer对象，那么应用程序本地窗口Surface在什么时候、什么地方创建呢？

为应用程序创建好了Layer对象并返回ISurface的代理对象给应用程序，应用程序通过该代理对象创建了一个SurfaceControl对象，Java层Surface需要通过android_view_Surface.cpp中的JNI函数来操作native层的Surface，在操作native层Surface前，首先需要获取到native的Surface，应用程序本地窗口Surface就是在这个时候创建的。 [->SurfaceControl.cpp]

```c
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

```c
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

#### 4.2.4.5、执行窗口布局performLayout()

[->ViewRootImpl.java]

```java
    private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
        int desiredWindowHeight) {
    mLayoutRequested = false;
    mScrollMayChange = true;
    mInLayout = true;
    final View host = mView;
    try {
        host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
        mInLayout = false;
        int numViewsRequestingLayout = mLayoutRequesters.size();
        if (numViewsRequestingLayout > 0) {
                ......
                measureHierarchy(host, lp, mView.getContext().getResources(),
                        desiredWindowWidth, desiredWindowHeight);
                mInLayout = true;
                host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
                ......
    }
    mInLayout = false;
}
```

#### 4.2.4.7、执行窗口绘制performDraw()

```java
[->ViewRootImpl.java]
private void performDraw() {
        ......
        try {
            draw(fullRedrawNeeded);
        }  ......
        }
    }
```

Android是怎样将View画出来的？由于之前我们已经关闭了HWC、GPU、HWUI，这里只关注软件绘制。 [->ViewRootImpl.java]

```java
    private void draw(boolean fullRedrawNeeded) {
    Surface surface = mSurface;
    ......
    if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
        if (mAttachInfo.mHardwareRenderer != null && mAttachInfo.mHardwareRenderer.isEnabled()) {
            ......
        } else {
            ......
            if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset, scalingRequired, dirty)) {
                return;
            }
        }
    }
    .....
}
```

关于渲染这个流程很复杂，我们后续章节再分析。

## 4.3、APP申请(lock)Buffer的过程

![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.graphics/17-Android-Graphics-App-SurfaceFlinger-lock-unlockpost.png)

```java
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

先看看Surface的lockCanvas方法： [->Surface.java]

```java
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

```c
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

这段代码逻辑主要如下： 1）获取java层dirty 的Rect大小和位置信息； 2）调用Surface的lock方法,将申请的图形缓冲区赋给outBuffer； 3）创建一个Skbitmap，填充它用来保存申请的图形缓冲区，并赋值给Java层的Canvas对象； 4）将剪裁位置大小信息赋给java层Canvas对象。

### 4.3.1、Surface管理图形缓冲区-APP申请(lock)Buffer的过程

我们上边分析到了申请图形缓冲区，用到了Surface的lock函数，我们继续查看。 [->Surface.cpp]

```c
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

Surface的lock函数用来申请图形缓冲区和一些操作，方法不长，大概工作有： 1）调用connect函数完成一些初始化； 2）调用dequeueBuffer函数，申请图形缓冲区； 3）计算需要绘制的新的dirty区域，旧的区域原样copy数据。 [->BufferQueueProducer.cpp]

```c
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

```c
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

## 4.4、APP提交(unlockAndPost)Buffer的过程

Surface绘制完毕后，unlockCanvasAndPost操作。 [->android_view_Surface.cpp]

```c
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

```c
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

这里也比较简单，核心也是分两步： 1）解锁图形缓冲区，和前面的lockAsync成对出现； 2）queueBuffer去归还图形缓冲区； 所以我们还是重点分析第二步，查看queueBuffer的实现： [->Surface.cpp]

```c
int Surface::queueBuffer(android_native_buffer_t* buffer, int fenceFd) {
......
status_t err = mGraphicBufferProducer->queueBuffer(i, input, &output);
mLastQueueDuration = systemTime() - now;
......
return err;
}
```

调用BufferQueueProducer的queueBuffer归还缓冲区，将绘制后的图形缓冲区queue回去。 [->BufferQueueProducer.cpp]

```c
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

总结： 1）从传入的QueueBufferInput ，解析填充一些变量； 2）改变入队Slot的状态为QUEUED，每次推进来，mFrameCounter都加1。这里的slot，上一篇讲分配缓冲区返回最老的FREE状态buffer，就是用这个mFrameCounter最小值判断，就是上一篇LRU算法的判断； 3）创建一个BufferItem来描述GraphicBuffer，用mSlots[slot]中的slot填充BufferItem； 4）将BufferItem塞进mCore的mQueue队列，依照指定规则； 5）然后通知SurfaceFlinger去消费。 Folw： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.graphics/18-Android-Graphics-App-WMS-SurfaceFlinger-All-Flow.png)

## （五）、通知SF消费合成

当绘制完毕的GraphicBuffer入队之后，会通知SurfaceFlinger去消费，就是BufferQueueProducer的queueBuffer函数的最后几行，listener->onFrameAvailable()。 listener最终通过回调，会回到Layer当中，所以最终调用Layer的onFrameAvailable接口，我们看看它的实现： [Layer.cpp]

```c
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

这里又调用SurfaceFlinger的signalLayerUpdate函数，继续查看： [SurfaceFlinger.cpp]

```c
void SurfaceFlinger::signalLayerUpdate() {
mEventQueue.invalidate();
}
```

这里又调用MessageQueue的invalidate函数： [MessageQueue.cpp]

```c
void MessageQueue::invalidate() {
mEvents->requestNextVsync();
}
```

贴一下SurfaceFlinger的初始化请求vsync信号流程图： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.graphics/19-Android-Graphics-vsync-surfaceflinger.png)

最终结果会走到SurfaceFlinger的vsync信号接收逻辑，即SurfaceFlinger的onMessageReceived函数： [SurfaceFlinger.cpp]

```c
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

SurfaceFlinger收到了VSync信号后，调用了handleMessageRefresh函数 [SurfaceFlinger.cpp]

```c
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

我们主要看下下面几个函数。 [SurfaceFlinger.cpp]

```c
preComposition();
rebuildLayerStacks();
setUpHWComposer();
doDebugFlashRegions();
doComposition();
postComposition(refreshStartTime);
```

### 一、preComposition()函数

我们先来看第一个函数preComposition() [SurfaceFlinger.cpp]

```c
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

上面函数先是调用了mDrawingState的layersSortedByZ来得到上次绘图的Layer层列表。并不是所有的Layer都会参与屏幕图像的绘制，因此SurfaceFlinger用state对象来记录参与绘制的Layer对象。 记得我们之前分析过createLayer函数来创建Layer，创建之后会调用addClientLayer函数。 [SurfaceFlinger.cpp]

```c
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

我们来看下addClientLayer函数，这里会把Layer对象放在mCurrentState的layersSortedByZ对象中。而mDrawingState和mCurrentState什么关系呢？在后面我们会介绍，mDrawingState代表上一次绘图时的状态，处理完之后会把mCurrentState赋给mDrawingState。 回到preComposition函数，遍历所有的Layer对象，调用其onPreComposition函数来检测Layer层中的图像是否有变化。

## 1.1、每个Layer的onFrameAvailable函数

onPreComposition函数来根据mQueuedFrames来判断图像是否发生了变化，或者是mSidebandStreamChanged、mAutoRefresh。 [Layer.cpp]

```c
bool Layer::onPreComposition() {
mRefreshPending = false;
return mQueuedFrames > 0 || mSidebandStreamChanged || mAutoRefresh;
}
```

当Layer所对应的Surface更新图像后，它所对应的Layer对象的onFrameAvailable函数会被调用来通知这种变化。 在SurfaceFlinger的preComposition函数中当有Layer的图像改变了，最后也会调用SurfaceFlinger的signalLayerUpdate函数。 SurfaceFlinger::signalLayerUpdate是调用了MessageQueue的invalidate函数 最后处理还是调用了SurfaceFlinger的onMessageReceived函数。看看SurfaceFlinger的onMessageReceived函数对NVALIDATE的处理 handleMessageInvalidate函数中调用了handlePageFlip函数，这个函数将会处理Layer中的缓冲区，把更新过的图像缓冲区切换到前台，等待VSync信号更新到FrameBuffer。

## 1.2、绘制流程

用户进程更新Surface图像，将导致SurfaceFlinger中的Layer发送invalidate消息，处理该消息会调用handleTransaction函数和handlePageFilp函数来更新Layer对象。一旦VSync信号到来，再调用rebuildlayerStacks setUpHWComposer doComposition postComposition函数将所有Layer的图像混合后更新到显示设备上去。

### 二、handleTransaction handPageFlip更新Layer对象

在上一节中的绘图的流程中，我们看到了handleTransaction和handPageFlip这两个函数通常是在用户进程更新Surface图像时会调用，来更新Layer对象。这节就主要讲解这两个函数。

## 2.1、handleTransaction函数

handleTransaction函数的参数是transactionFlags，不过函数中没有使用这个参数，而是通过getTransactionFlags(eTransactionMask)来重新对transactionFlags赋值，然后使用它作为参数来调用函数 handleTransactionLocked。 [SurfaceFlinger.cpp]

```c
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

getTransactionFlags函数的参数是eTransactionMask只是屏蔽其他位。 handleTransactionLocked函数会调用每个Layer类的doTransaction函数，在分析handleTransactionLocked函数之前，我们先看看Layer类 的doTransaction函数。

## 2.2、Layer的doTransaction函数

下面是Layer的doTransaction函数代码 [Layer.cpp]

```java
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

Layer类中的两个类型为Layer::State的成员变量mDrawingState、mCurrentState，这里为什么要两个对象呢？Layer对象在绘制图形时，使用的是mDrawingState变量，用户调用接口设置Layer对象属性是，设置的值保存在mCurrentState对象中，这样就不会因为用户的操作而干扰Layer对象的绘制了。 Layer的doTransaction函数据你是比较这两个变量，如果有不同的地方，说明在上次绘制以后，用户改变的Layer的设置，要把这种变化通过flags返回。 State的结构中有两个Geometry字段，active和requested。他们表示layer的尺寸，其中requested保存是用户设置的尺寸，而active保存的值通过计算后的实际尺寸。 State中的z字段的值就是Layer在显示轴的位置，值越小位置越靠下。 layerStack字段是用户指定的一个值，用户可以给DisplayDevice也指定一个layerStack值，只有Layer对象和DisplayDevice对象的layerStack相等，这个Layer才能在这个显示设备上输出，这样的好处是可以让显示设备只显示某个Surface的内容。例如，可以让HDMI显示设备只显示手机上播放视频的Surface窗口，但不显示Activity窗口。 sequence字段是个序列值，每当用户调用了Layer的接口，例如setAlpha、setSize或者setLayer等改变Layer对象属性的哈数，这个值都会加1。因此在doTransaction函数中能通过比较sequence值来判断Layer的属性值有没有变化。 doTransaction函数最后会调用commitTransaction函数，就是把mCurrentState赋值给mDrawingState [Layer.cpp]

```c
void Layer::commitTransaction(const State& stateToCommit) {
mDrawingState = stateToCommit;
}
```

### 2.3、handleTransactionLocked函数

下面我们来分析handleTransactionLocked函数，这个函数比较长，我们分段分析

2.3.1 处理Layer的事务 [SurfaceFlinger.cpp]

```c
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

```c
    struct State {
    LayerVector layersSortedByZ;
    DefaultKeyedVector< wp<IBinder>, DisplayDeviceState> displays;
};
```

结构layersSortedByZ字段保存所有参与绘制的Layer对象，而字段displays保存的是所有输出设备的DisplayDeviceState对象 这里用两个变量的目的是和Layer中使用两个变量是一样的。 上面代码根据eTraversalNeeded标志来决定是否要检查所有的Layer对象。如果某个Layer对象中有eTransactionNeeded标志，将调用它的doTransaction函数。Layer的doTransaction函数返回的flags如果有eVisibleRegion，说明这个Layer需要更新，就把mVisibleRegionsDirty设置为true

### 2.3.2、处理显示设备的变化

```c
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

这段代码的作用是处理显示设备的变化，分成3种情况： 1.显示设备减少了，需要把显示设备对应的DisplayDevice移除 2.显示设备发生了变化，例如用户设置了Surface、重新设置了layerStack、旋转了屏幕等，这就需要重新设置显示对象的属性 3.显示设备增加了，创建新的DisplayDevice加入系统中。

### 2.3.3、设置TransfromHit

```c
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

### 2.3.4、处理Layer增加情况

```c
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

### 2.3.5、设置mDrawingState

```c
commitTransaction();
updateCursorAsync();
```

调用commitTransaction和updateCursorAsync函数 commitTransaction函数作用是把mDrawingState的值设置成mCurrentState的值。而updateCursorAsync函数会更新所有显示设备中光标的位置。

2.3.6 小结 handleTransaction函数的作用的就是处理系统在两次刷新期间的各种变化。SurfaceFlinger模块中不管是SurfaceFlinger类还是Layer类，都采用了双缓冲的方式来保存他们的属性，这样的好处是刚改变SurfaceFlinger对象或者Layer类对象的属性是，不需要上锁，大大的提高了系统效率。只有在最后的图像输出是，才进行一次上锁，并进行内存的属性变化处理。正因此，应用进程必须收到VSync信号才开始改变Surface的内容。

## 2.4、handlePageFlip函数

handlePageFlip函数代码如下： [SurfaceFlinger.cpp]

```c
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

handlePageFlip函数先调用每个Layer对象的hasQueuedFrame函数，确定这个Layer对象是否有需要更新的图层，然后把需要更新的Layer对象放到layersWithQueuedFrames中。 我们先来看Layer的hasQueuedFrame方法就是看其mQueuedFrames是否大于0 和mSidebandStreamChanged。前面小节分析只要Surface有数据写入，就会调用Layer的onFrameAvailable函数，然后mQueuedFrames值加1\. 继续看handlePageFlip函数，接着调用需要更新的Layer对象的latchBuffer函数，然后根据返回的更新区域调用invalidateLayerStack函数来设置更新设备对象的更新区域。 下面我们看看latchBuffer函数

LatchBuffer函数调用updateTextImage来得到需要的图像。这里参数r是Reject对象，其作用是判断在缓冲区的尺寸是否符合要求。调用updateTextImage函数如果得到的结果是PRESENT_LATER,表示推迟处理，然后调用signalLayerUpdate函数来发送invalidate消息，这次绘制过程就不处理这个Surface的图像了。 如果不需要推迟处理，把mQueuedFrames的值减1\. 最后LatchBuffer函数调用mSurfaceFlingerConsumer的getCurrentBuffer来取回当前的图像缓冲区指针，保存在mActiveBuffer中。

### 2.5 小结

这样经过handleTransaction handlePageFlip两个函数处理，SurfaceFlinger中无论是Layer属性的变化还是图像的变化都处理好了，只等VSync信号到来就可以输出了。

## 三、rebuildLayerStacks函数

前面介绍，VSync信号到来后，先是调用了rebuildLayerStacks函数

```c
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

rebuildLayerStacks函数的作用是重建每个显示设备的可见layer对象列表。对于按显示轴（Z轴）排列的Layer对象，排在最前面的当然会优先显示，但是Layer图像可能有透明域，也可能有尺寸没有覆盖整个屏幕，因此下面的layer也有显示的机会。rebuildLayerStacks函数对每个显示设备，先计算和显示设备具有相同layerStack值的Layer对象在该显示设备上的可见区域。然后将可见区域和显示设备的窗口区域有交集的layer组成一个新的列表，最后把这个列表设置到显示设备对象中。 computeVisibleRegions函数首先计算每个Layer在设备上的可见区域visibleRegion。计算方法就是用整个Layer的区域减去上层所有不透明区域aboveOpaqueLayers。而上层所有不透明区域值是一个逐层累计的过程，每层都需要把自己的不透明区域累加到aboveOpaqueLayers中。 而每层的不透明区域的计算方法：如果Layer的alpha的值为255，并且layer的isOpaque函数为true，则本层的不透明区域等于Layer所在区域，否则为0.这样一层层算下来，就很容易得到每层的可见区域大小了。 其次，计算整个显示设备需要更新的区域outDirtyRegion。outDirtyRegion的值也是累计所有层的需要重回的区域得到的。如果Layer中的显示内容发生了变化，则整个可见区域visibleRegion都需要更新，同时还要包括上一次的可见区域，然后在去掉被上层覆盖后的区域得到的就是Layer需要更新的区域。如果Layer显示的内容没有变化，但是考虑到窗口大小的变化或者上层窗口的变化，因此Layer中还是有区域可以需要重绘的地方。这种情况下最简单的算法是用Layer计算出可见区域减去以前的可见区域就可以了。但是在computeVisibleRegions函数还引入了被覆盖区域，通常被覆盖区域和可见区域并不重复，因此函数中计算暴露区域是用可见区域减去被覆盖区域的。

### 四、setUpHWComposer函数

setUpHWComposer函数的作用是更新HWComposer对象中图层对象列表以及图层属性。 [SurfaceFlinger.cpp]

```c
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

HWComposer中有一个类型为DisplayData结构的数组mDisplayData，它维护着每个显示设备的信息。DisplayData结构中有一个类型为hwc_display_contents_l字段list，这个字段又有一个hwc_layer_l类型的数组hwLayers，记录该显示设备所有需要输出的Layer信息。 setUpHWComposer函数调用HWComposer的createWorkList函数就是根据每种显示设备的Layer数量，创建和初始化hwc_display_contents_l对象和hwc_layer_l数组 创建完HWComposer中的列表后，接下来是对每个Layer对象调用它的setPerFrameData函数，参数是HWComposer和HWCLayerInterface。setPerFrameData函数将Layer对象的当前图像缓冲区mActiveBuffer设置到HWCLayerInterface对象对应的hwc_layer_l对象中。 HWComposer类中除了前面介绍的Gralloc还管理着Composer模块，这个模块实现了硬件的图像合成功能。setUpHWComposer函数接下来调用HWComposer类的prepare函数，而prepare函数会调用Composer模块的prepare接口。最后到各个厂家的实现hwc_prepare函数将每种HWComposer中的所有图层的类型都设置为HWC_FRAMEBUFFER就结束了。

### 五、合成所有层的图像 （doComposition()函数）

doComposition函数是合成所有层的图像，代码如下： [SurfaceFlinger.cpp]

```c
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

```c
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

doDisplayComposition函数根据显示设备支持的更新方式，重新设置需要更新区域的大小。 真正的合成工作是在doComposerSurfaces函数中完成，这个函数在layer的类型为HWC_FRAMEBUFFER,或者不支持硬件的composer的情况下，调用layer的draw函数来一层一层低合成最后的图像。 合成完后，doDisplayComposition函数调用了hw的swapBuffers函数，这个函数前面介绍过了，它将在系统不支持硬件的composer情况下调用eglSwapBuffers来输出图像到显示设备。

### 六、postFramebuffer()函数

上一节的doComposition函数最后调用了postFramebuffer函数，代码如下： [SurfaceFlinger.cpp]

```c
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

postFramebuffer先判断系统是否支持composer，如果不支持，我们知道图像已经在doComposition函数时调用hw->swapBuffers输出了，就返回了。如果支持硬件composer，postFramebuffer函数将调用HWComposer的commit函数继续执行。 [HWComposer.cpp]

```c
status_t HWComposer::commit() {
int err = NO_ERROR;
if (mHwc) {
    if (!hwcHasApiVersion(mHwc, HWC_DEVICE_API_VERSION_1_1)) {
        // On version 1.0, the OpenGL ES target surface is communicated
        // by the (dpy, sur) fields and we are guaranteed to have only
        // a single display.
        mLists[0]->dpy = eglGetCurrentDisplay();
        mLists[0]->sur = eglGetCurrentSurface(EGL_DRAW);
    }

    for (size_t i=VIRTUAL_DISPLAY_ID_BASE; i<mNumDisplays; i++) {
        DisplayData& disp(mDisplayData[i]);
        if (disp.outbufHandle) {
            mLists[i]->outbuf = disp.outbufHandle;
            mLists[i]->outbufAcquireFenceFd =
                    disp.outbufAcquireFence->dup();
        }
    }

    err = mHwc->set(mHwc, mNumDisplays, mLists);

    for (size_t i=0 ; i<mNumDisplays ; i++) {
        DisplayData& disp(mDisplayData[i]);
        disp.lastDisplayFence = disp.lastRetireFence;
        disp.lastRetireFence = Fence::NO_FENCE;
        if (disp.list) {
            if (disp.list->retireFenceFd != -1) {
                disp.lastRetireFence = new Fence(disp.list->retireFenceFd);
                disp.list->retireFenceFd = -1;
            }
            disp.list->flags &= ~HWC_GEOMETRY_CHANGED;
        }
    }
}
return (status_t)err;
}
```

合成效果图： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.graphics/20-Android-Graphics-SurfaceFlinger-composition.png)

## （六）、Android SurfaceFlinger - VSync工作原理

### 一、VSYNC 总体概念

### 6.1.1、VSYNC 概念

VSYNC（Vertical Synchronization）是一个相当古老的概念，对于游戏玩家，它有一个更加大名鼎鼎的中文名字---垂直同步。 "垂直同步(vsync)"指的是显卡的输出帧数和屏幕的垂直刷新率相同，这完全是一个CRT显示器上的概念。其实无论是VSYNC还是垂直同步这个名字，因为LCD根本就没有垂直扫描的这种东西，因此这个名字本身已经没有意义。但是基于历史的原因，这个名称在图形图像领域被沿袭下来。 在当下，垂直同步的含义我们可以理解为，使得显卡生成帧的速度和屏幕刷新的速度的保持一致。举例来说，如果屏幕的刷新率为60Hz，那么生成帧的速度就应该被固定在1/60 s。

### 6.1.2、Android VSYNC -- 黄油计划

谷歌为解决Android系统流畅性问题。在4.1版本引入了一个重大的改进--Project Butter黄油计划。 Project Butter对Android Display系统进行了重构，引入了三个核心元素，即VSYNC、Triple Buffer和Choreographer。 VSYNC最重要的作用是防止出现画面撕裂（screentearing）。所谓画面撕裂，就是指一个画面上出现了两帧画面的内容，如下图。 
![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.graphics/21-Android-graphics-view-teaning.png)

为什么会出现这种情况呢？这种情况一般是因为显卡输出帧的速度高于显示器的刷新速度，导致显示器并不能及时处理输出的帧，而最终出现了多个帧的画面都留在了显示器上的问题。这也就是我们所说的画面撕裂。 
![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.graphics/22-Android-Graphics-Draw-whithout-vsync.png)

这个图中有三个元素，Display是显示屏幕，GPU和CPU负责渲染帧数据，每个帧以方框表示，并以数字进行编号，如0、1、2等等。VSync用于指导双缓冲区的交换。 以时间的顺序来看下将会发生的异常： Step1\. Display显示第0帧数据，此时CPU和GPU渲染第1帧画面，而且赶在Display显示下一帧前完成 Step2\. 因为渲染及时，Display在第0帧显示完成后，也就是第1个VSync后，正常显示第1帧 Step3\. 由于某些原因，比如CPU资源被占用，系统没有及时地开始处理第2帧，直到第2个VSync快来前才开始处理 Step4\. 第2个VSync来时，由于第2帧数据还没有准备就绪，显示的还是第1帧。这种情况被Android开发组命名为"Jank"。 Step5\. 当第2帧数据准备完成后，它并不会马上被显示，而是要等待下一个VSync。 所以总的来说，就是屏幕平白无故地多显示了一次第1帧。原因大家应该都看到了，就是CPU没有及时地开始着手处理第2帧的渲染工作，以致"延误军机"。

其实总结上面的这个情况之所以发生，首先的原因就在于第二帧没有及时的绘制（当然即使第二帧及时绘制，也依然可能出现Jank，这就是同时引入三重缓冲的作用。我们将在三重缓冲一节中再讲解这种情况）。那么如何使得第二帧即使被绘制呢？ 这就是我们在Graphic系统中引入VSYNC的原因，考虑下面这张图： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.graphics/23-Android-Graphics-Draw-whit-vsync.png)

如上图所示，一旦VSync出现后，立刻就开始执行下一帧的绘制工作。这样就可以大大降低Jank出现的概率。另外，VSYNC引入后，要求绘制也只能在收到VSYNC消息之后才能进行，因此，也就杜绝了另外一种极端情况的出现---CPU（GPU）一直不停的进行绘制，帧的生成速度高于屏幕的刷新速度，导致生成的帧不能被显示，只能丢弃，这样就出现了丢帧的情况---引入VSYNC后，绘制的速度就和屏幕刷新的速度保持一致了。

### 二、VSync信号产生

那么VSYNC信号是如何生成的呢？ Android系统中VSYNC信号分为两种，一种是硬件生成的信号，一种是软件模拟的信号。 硬件信号是由HardwareComposer提供的，HWC封装了相关的HAL层，如果硬件厂商提供的HAL层实现能定时产生VSYNC中断，则直接使用硬件的VSYNC中断，否则HardwareComposer内部会通过VSyncThread来模拟产生VSYNC中断（其实现很简单，就是sleep固定时间，然后唤醒）。

SurfaceFlinger的启动过程中inti()会创建一个HWComposer对象。

```c
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

我们来看下上面这段代码。 首先mDebugForceFakeVSync是为了调制，可以通过这个变量设置强制使用软件VSYNC模拟。 然后针对不同的屏幕，初始化了他们的mLastHwVSync和mVSyncCounts值。 如果硬件支持，那么就把needVSyncThread设置为false，表示不需要软件模拟。 接着通过eventControl来暂时的关闭了VSYNC信号，这一点将在下面讲解eventControl时一并讲解。 最后，如果需要软件模拟Vsync信号的话，那么我们将通过一个单独的VSyncThread线程来做这个工作(fake VSYNC是这个线程唯一的作用)。我们来看下这个线程。

**软件模拟**

```c
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

```c
mHwc = new HWComposer(this,  
        *static_cast<HWComposer::EventHandler *>(this));  

class SurfaceFlinger : public BnSurfaceComposer,  
                   private IBinder::DeathRecipient,  
                   private HWComposer::EventHandler
```

可以看到这个mEventHandler实际上就是SurfaceFlinger。也就是说，VSYNC信号到来时，SurfaceFlinger的onVSyncReceived函数处理了这个消息。 这里我们暂时先不展开SurfaceFlinger内的逻辑处理，等我们下面分析完硬件实现后，一并进行分析

**硬件实现** 上面我们讲了软件如何模拟一个VSYNC信号并通知SurfaceFlinger,那么硬件又是如何实现这一点的呢？ 我们再一次回到HWC的创建过程中来：

```c
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

来看下上面这段实现。 当HWC有vsync信号生成时，硬件模块会通过procs.vsync来通知软件部分，因此也就是调用了hook_vsync函数。

```c
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

### 三、Surfaceflinger对VSYNC消息的处理

先来直接看下Surfaceflinger的onVSyncReceived函数：

```c
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

mPrimaryDispSync是什么？addResyncSample有什么作用？ 要回答这三个问题，我们首先还是得回到SurfaceFlinger的init函数中来。

### 6.3.1、Surfaceflinger.init()

先看一下总体flow： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.graphics/24-Android-Graphics-Vsync-surfaceflinger.init.png)

```c
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

2个EventThread对象分别是mEventThread，给app用，mSFEventThread，给surfaceflinger自己用。 下面给出这4个Thread关系图。 
![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.graphics/25-Android-Graphics-SurfaceFlinger.init.DispSyncThread.png.png)

这两个DispSyncSource就是KK引入的重大变化。Android 4.4(KitKat)引入了VSync的虚拟化，即把硬件的VSync信号先同步到一个本地VSync模型中，再从中一分为二，引出两条VSync时间与之有固定偏移的线程。示意图如下： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.graphics/26-Android-Graphics-SurfaceFlinger-App-Vsync-offset.png.png)

Google这样修改的目的又是什么呢？ =在当前三重缓冲区的架构下，即对于一帧内容，先等App UI画完了，SurfaceFlinger再出场对其进行合并渲染后放入framebuffer，最后整到屏幕上。而现有的VSync模型是让大家一起开始干活。 这个架构其实会产生一个问题，因为App和SurfaceFlinger被同时唤醒，导致他们二者总是一起工作，必然导致VSync来临的时刻，这二者之间产生了CPU资源的抢占。因此，谷歌给这两个工作都加上一个小小的延迟，让这两个工作并不是同时被唤醒，这样大家就可以错开使用资源的高峰期，提高工作的效率。 
![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.graphics/27-Android-graphics-SurfaceFlinger-Vsync-app-sf.png)

这两个延迟，其实就分别对应上面代码中的vsyncSrc（绘制延迟）和sfVsyncSrc（合成延迟）。 在创建了两个DispSyncSource变量后，我们使用它们来初始化了两个EventThread。下面我们来详细看下EventThread的创建流程：

```c
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

```c
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

上面的函数本身并不复杂，其中调用了一个waitForEvent的函数。这个函数相当之长，为了防止代码展开太多，我们这里暂时不再详细分析这个函数。我们目前只需要知道这个函数的最重要的作用是等待Event的到来，并且查找对event感兴趣的监听者，而在没有event到来时，线程处于休眠状态，等待event的唤醒（我们将下一篇VSYNC的接收和处理中展开分析这个函数）。 这样，EventThread线程就运行起来，处在等待被event唤醒的状态下。 **MessageQueue和EventThread建立连接** 简单说明完EventThread之后，我们再次回到SurfaceFlinger的init过程中来。回到init()函数代码中来： 将SurfaceFlinger的MessageQueue真正和我们刚才创建的EventThread建立起了连接，这样SurfaceFlinger才能真正接收到来自HWC的VSYNC信号。 我们来看下这段代码：

```c
void MessageQueue::setEventThread(const sp<EventThread>& eventThread)  
{  
    mEventThread = eventThread;  
    mEvents = eventThread->createEventConnection();  
    mEventTube = mEvents->getDataChannel();  
    mLooper->addFd(mEventTube->getFd(), 0, ALOOPER_EVENT_INPUT,  
            MessageQueue::cb_eventReceiver, this);  
}
```

这里代码逻辑其实很简单，就是创建了一个到EventThread的连接，得到了发送VSYNC事件通知的BitTube，然后监控这个BitTube中的套接字，并且指定了收到通知后的回调函数，MessageQueue::cb_eventReceiver。这样一旦VSync信号传来，函数cb_eventReceiver将被调用。 **向Eventhread注册一个事件的监听者----createEventConnection** 在SurfaceFlinger的init函数中，我们调用了mEventQueue.setEventThread(mSFEventThread)函数，我们在前面一章中已经提到过，这个函数将SurfaceFlinger的MessageQueue真正和我们刚才创建的EventThread建立起了连接。我们来看下这段代码：

```c
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

这个函数会导致一个Connection类的创建，而这个connection类会被保存在EventThread下的一个容器内。 通过createEventConnection这样一个简单的方法，我们其实就注册了一个事件的监听者，得到了发送VSYNC事件通知的BitTube，然后监控这个BitTube中的套接字，并且指定了收到通知后的回调函数，MessageQueue::cb_eventReceiver。这样一旦VSync信号传来，函数cb_eventReceiver将被调用。

### 6.3.2、VSync信号的处理

我们在前面一章也提到了无论是软件方式还是硬件方式，SurfaceFlinger收到VSync信号后，处理函数都是onVSyncReceived函数：

**VSync消息处理----addResyncSample** 
![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.graphics/28-Android-Graphics-SF-Vsync-addResyncSample.png.png)

```c
bool DispSync::addResyncSample(nsecs_t timestamp) {  
    size_t idx = (mFirstResyncSample + mNumResyncSamples) % MAX_RESYNC_SAMPLES;  
    mResyncSamples[idx] = timestamp;  

    ......
    updateModelLocked();  
    .......
}
```

粗略浏览下这个函数，发现前半部分其实在做一些简单的计数统计，重点实现显然是updateModelLocked函数：

```c
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

```c
void updateModel(nsecs_t period, nsecs_t phase) {  
    Mutex::Autolock lock(mMutex);  
    mPeriod = period;  
    mPhase = phase;  
    mCond.signal();  
}
```

updateModel是DispSyncThread类的函数，这个函数本身代码很短，其实它的主要作用是mCond.signal发送一个信号给等待中的线程。那么究竟是谁在等待这个条件呢？ 其实等待这个条件的正是DispSyncThread的循环函数：

```c
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

```c
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

```c
void fireCallbackInvocations(const Vector<CallbackInvocation>& callbacks) {  
    for (size_t i = 0; i < callbacks.size(); i++) {  
        callbacks[i].mCallback->onDispSyncEvent(callbacks[i].mEventTime);  
    }  
}`
```

我们目前只分析主线的走向,接下来调用了DispSyncSource的onDispSyncEvent在：

```c
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

```c
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

```c
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

![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.graphics/29-Android-Graphics-App-SurfaceFlinger-Vsync-postEvent.png.png)

其实看到这里的BitTube我们就明白了，在本文开始时候我们提到：

通过createEventConnection这样一个简单的方法，我们其实就注册了一个事件的监听者，得到了发送VSYNC事件通知的BitTube，然后监控这个BitTube中的套接字，并且指定了收到通知后的回调函数，MessageQueue::cb_eventReceiver。这样一旦VSync信号传来，函数cb_eventReceiver将被调用。

所以我们这里可以来看看MessageQueue::cb_eventReceiver函数了：

```c
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

```c
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

### 6.4、App向Eventhread注册一个事件的监听者--createEventConnection()

在ViewRootImpl的构造函数中会实例化Choreographer对象

```java
public ViewRootImpl(Context context, Display display) {
                . . . . .
        mChoreographer = Choreographer.getInstance();
 }
```

在mChoreographer 的构造函数中实例化FrameDisplayEventReceiver对象

```java
private Choreographer(Looper looper) {
               . . . . . .
               mDisplayEventReceiver = USE_VSYNC ? new FrameDisplayEventReceiver(looper) : null;
 }
```

在FrameDisplayEventReceiver的父类构造函数中会调用到，android_view_DisplayEventReceiver.cpp中的nativeInit方法,在nativeInit方法中有如下过程

```c
static jlong nativeInit(JNIEnv* env, jclass clazz, jobject receiverWeak,
        jobject messageQueueObj) {
        . . . . . .
    sp<NativeDisplayEventReceiver> receiver = new NativeDisplayEventReceiver(env,
            receiverWeak, messageQueue);
    status_t status = receiver->initialize();
    . . . . . .
```

创建NativeDisplayEventReceiver类 类型指针 在NativeDisplayEventReceiver的构造函数中会调用DisplayEventReceiver类的无参构造函数实例化成员mReceiver；

```c
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

在这段代码中获取Surfaceflinger服务的代理对象，然后通过Binder IPC创建BpDisplayEventConnection对象 该函数经由BnSurfaceComposer.onTransact函数辗转调用到SurfaceFlinger.createDisplayEventConnection函数：

```c
sp<IDisplayEventConnection> SurfaceFlinger::createDisplayEventConnection() {
    return mEventThread->createEventConnection();
}
```

出现了熟悉的面孔mEventThread，该对象是一个EventThread对象，该对象在SurfaceFlinger.init函数里面创建，但是创建运行以后，貌似还没有进行任何的动作，这里调用createEventConnection函数：

```c
sp<EventThread::Connection> EventThread::createEventConnection() const {
    return new Connection(const_cast<EventThread*>(this));
}
```

然后mEventConnection->getDataChannel()方法再次通过Binder IPC创建 BitTube对象mDataChannel ，在Binder IPC创建mDataChannel 过程中会从服务端EventThread::Connection::Connection中（在EventThread类中定义）接收一个socketpair创建的FIFO文件描述符；

EventThread::Connection::Connection创建描述符的代码： Connection构造函数调用BitTube的无参构造函数，在BitTube的构造函数中调用init函数；

```c
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

调用到NativeDisplayEventReceiver类的父类DisplayEventDispatcher中的initialize()方法， 将BpDisplayEventConnection对象获取到的mDataChannel （BitTube类型）中的文件描述符添加到UI主线程Looper的epoll中， 当文件描述符中被写入数据时，该epoll_wait会被唤醒； 直接看代码：

```c
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

这里的主要代码是mMessageQueue->getLooper()->addFd()这一行，其中的参数mReceiver.getFd()返回的是在创建NativeDisplayEventReceiver时从SurfaceFlinger服务端接收回来的socket接收端描述符，前面分析到 mMessageQueue是与当前应用线程关联的java层的MessageQueue对应的native层的MessageQueue对象，下面看一下Looper.addFd这个函数，上面调用时传进来的this指针对应的是一个NativeDisplayEventReceiver对象，该类继承了LooperCallback：

```c
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

首先将上面传进来的NativeDisplayEventReceiver对象封装成一个SimpleLooperCallback对象，调用下面的addFd函数的时候主要步骤如下： （1）创建一个struct epoll_event结构体对象，将对应的内存全部用清0，并作对应的初始化； （2）查询通过addFd方法已经添加到epoll中监听的文件描述符； （3）查询不到的话，则调用epoll_ctl方法设置EPOLL_CTL_ADD属性将对应的文件描述符添加到epoll监听的描述符中； （4）根据前面addFd传入的参数EVENT_INPUT，说明当前应用线程的native层的Looper对象中的epoll机制已经开始监听来自于SurfaceFlinger服务端socket端的写入事件。

### 6.5、App请求Vsync信号

前面讲解ViewRootImpl.setView()的时候，因涉及到Vsync信号知识，requestLayout()没有具体讲解，现在继续。

```java
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

```java
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

```java
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

### 6.5.1、Vsync请求过程

我们知道在Choreographer构造函数中，构造了一个FrameDisplayEventReceiver对象，用于请求并接收Vsync信号，Vsync信号请求过程如下：

```java
private void scheduleVsyncLocked() {  
//申请Vsync信号  
mDisplayEventReceiver.scheduleVsync();  
}
```

[->DisplayEventReceiver.java]

```java
public void scheduleVsync() {
    if (mReceiverPtr == 0) {
        Log.w(TAG, "Attempted to schedule a vertical sync pulse but the display event "
                + "receiver has already been disposed.");
    } else {
        nativeScheduleVsync(mReceiverPtr);
    }
}
```

[->android_view_DisplayEventReceiver.cpp ]

```java
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

VSync请求过程又转交给了DisplayEventReceiver： [->DisplayEventReceiver.cpp]

```java
status_t DisplayEventReceiver::requestNextVsync() {
if (mEventConnection != NULL) {
    mEventConnection->requestNextVsync();
    return NO_ERROR;
}
return NO_INIT;
}
```

这里的mEventConnection也是前面创建native层对象NativeDisplayEventReceiver时创建的，实际对象是一个BpDisplayEventConnection对象，也就是一个Binder客户端，对应的Binder服务端BnDisplayEventConnection是一个EventThread::Connection对象，对应的BpDisplayEventConnection.requestNextVsync函数和BnDisplayEventConnection.onTransact(REQUEST_NEXT_VSYNC)函数没有进行特别的处理，下面就调用到EventThread::Connection.requestNextVsync函数，从BnDisplayEventConnection.onTransact(REQUEST_NEXT_VSYNC)开始已经从用户进程将需要垂直同步信号的请求发送到了SurfaceFlinger进程，下面的函数调用开始进入SF进程：

```c
void EventThread::Connection::requestNextVsync() {
    mEventThread->requestNextVsync(this);
    }
```

辗转调用到EventThread.requestNextVsync函数，注意里面传了参数this，也就是当前的EventThread::Connection对象，需要明确的是，这里的mEventThread对象是创建EventThread::Connection对象的时候保存的，对应的是SurfaceFlinger对象的里面的mEventThread成员，该对象是一个在SurfaceFlinger.init里面创建并启动的线程对象，可见设计的时候就专门用这个SurfaceFlinger.mEventThread线程来接收来自应用进程的同步信号请求，每来一个应用进程同步信号请求，就通过SurfaceFlinger.mEventThread创建一个EventThread::Connection对象，并通过EventThread.registerDisplayEventConnection函数将创建的EventThread::Connection对象保存到EventThread.mDisplayEventConnections里面，上面有调用到了EventThread.requestNextVsync函数：

```c
void EventThread::requestNextVsync(const sp<EventThread::Connection>& connection) {
    Mutex::Autolock _l(mLock);
    if (connection->count < 0) {
        connection->count = 0;
        mCondition.broadcast();
    }
}
```

传进来的是一个前面创建的EventThread::Connection对象，里面判断到了EventThread::Connection.count成员变量，看一下EventThread::Connection构造函数中初始变量的值：

```c
EventThread::Connection::Connection(const sp<EventThread>& eventThread)
    : count(-1), mEventThread(eventThread), mChannel(new BitTube()){
}
```

可以看到初始值是-1，这个值就是前面那个问题的关键，EventThread::Connection.count标示了这次应用进程的垂直同步信号的请求是一次性的，还是多次重复的，看一下注释里面对于这个变量的说明：

```c
// count >= 1 : continuous event. count is the vsync rate
// count == 0 : one-shot event that has not fired
// count ==-1 : one-shot event that fired this round / disabled
int32_t count;
```

很清楚的说明了，count = 0说明当前的垂直同步信号请求是一个一次性的请求，并且还没有被处理。上面EventThread::requestNextVsync里面将count设置成0，同时调用了mCondition.broadcast()唤醒所有正在等待mCondition的线程，这个会触发EventThread.waitForEvent函数从：

```c
mCondition.wait(mLock);
```

中醒来，醒来之后经过一轮do...while循环就会返回，返回以后调用序列如下： （1）EventThread::Connection.postEvent(event) （2）DisplayEventReceiver::sendEvents(mChannel, &event, 1)，mChannel参数就是前面创建DisplayEventReceiver是创建的BitTube对象 （3）BitTube::sendObjects(dataChannel, events, count)，static函数，通过dataChannel指向BitTube对象 最终调用到BitTube::sendObjects函数：

```c
ssize_t BitTube::sendObjects(const sp<BitTube>& tube, void const* events, size_t count, size_t objSize){
    const char* vaddr = reinterpret_cast<const char*>(events);
    ssize_t size = tube->write(vaddr, count*objSize);
    return size < 0 ? size : size / static_cast<ssize_t>(objSize);
}
```

继续调用到BitTube::write函数：

```c
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

这里调用到了::send函数，::是作用域描述符，如果前面没有类名之类的，代表的就是全局的作用域，也就是调用全局函数send，这里很容易就能想到这是一个socket的写入函数，也就是将event事件数据写入到BitTube中互联的socket中，这样在另一端马上就能收到写入的数据，前面分析到这个BitTube的socket的两端连接着SurfaceFlinger进程和应用进程，也就是说通过调用BitTube::write函数，将最初由SurfaceFlinger捕获到的垂直信号事件经由BitTube中互联的socket从SurfaceFlinger进程发送到了应用进程中BitTube的socket接收端。 下面就要分析应用进程是如何接收并使用这个垂直同步信号事件的。

### 6.5.2、应用进程接收VSync

#### 6.5.2.1、解析VSync事件

VSync同步信号事件已经发送到用户进程中的socket接收端，在前面NativeDisplayEventReceiver.initialize中分析到应用进程端的socket接收描述符已经被添加到Choreographer所在线程的native层的Looper机制中，在epoll中监听EPOLLIN事件，当socket收到数据后，epoll会马上返回，下面分步骤看一下Looper.pollInner()数： （1）epoll_wait

```c
struct epoll_event eventItems[EPOLL_MAX_EVENTS];
int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);
```

在监听到描述符对应的事件后，epoll_wait会马上返回，并将产生的具体事件类型写入到参数eventItems里面，最终返回的eventCount是监听到的事件的个数 （2）事件分析

```c
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

Looper目前了解到的主要监听的文件描述符种类有两种： 1）消息事件，epoll_wait监听pipe管道的接收端描述符mWakeReadPipeFd 2）与VSync信号，epoll_wait监听socket接收端描述符，并在addFd的过程中将相关的信息封装在一个Request结构中，并以fd为key存储到了mRequests中，具体可以回过头看3.1.2关于addFd的分析； 因此，上面走的是else的分支，辨别出当前的事件类型后，调用pushResponse：

```c
void Looper::pushResponse(int events, const Request& request) {
    Response response;
    response.events = events;
    response.request = request;  //复制不是引用，调用拷贝构造函数
    mResponses.push(response);
}
```

该函数将Request和events封装在一个Response对象里面，存储到了mResponses里面，也就是mResponses里面放的是"某某fd上接收到了类别为events的时间"记录，继续向下看Looper.pollInner函数 （3）事件分发处理

```c
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

```c
int SimpleLooperCallback::handleEvent(int fd, int events, void* data) {
    return mCallback(fd, events, data);
}
```

这里的mCallback就是当时在addFd的时候传进来的callBack参数，实际上对应的就是NativeDisplayEventReceiver对象本身，因此最终就将垂直同步信号事件分发到了NativeDisplayEventReceiver.handleEvent函数中。

### 6.5.3、**VSync事件分发**

调用到NativeDisplayEventReceiver.handleEvent函数，该函数定义在android_view_DisplayEventReceiver.cpp中，直接列出该函数：

```c
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

```c
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

这里的mReceiver也就是前面创建NativeDisplayEventReceiver对象是创建的成员变量对象DisplayEventReceiver，下面调用到DisplayEventReceiver.getEvents函数，应该是要从出现同步信号事件的socket中读取数据，上面Looper机制中epoll中监听到socket以后，返回到NativeDisplayEventReceiver.handleEvent里面，但是socket里面的数据还没有读取，下面的调用流程为： （1）mReceiver.getEvents(buf, EVENT_BUFFER_SIZE) ---> DisplayEventReceiver::getEvents(DisplayEventReceiver::Event _events, size_t count) （2）BitTube::recvObjects(dataChannel, events, count) ---> BitTube::recvObjects(const sp& tube, void_ events, size_t count, size_t objSize) 看一下这个recvObjects函数：

```c
ssize_t BitTube::recvObjects(const sp<BitTube>& tube, void* events, size_t count, size_t objSize)
{
    char* vaddr = reinterpret_cast<char*>(events);
    ssize_t size = tube->read(vaddr, count*objSize);
    return size < 0 ? size : size / static_cast<ssize_t>(objSize);
}
```

这里在NativeDisplayEventReceiver中创建了一个缓冲区，并在recvObjects中将socket中的Event数据读到这个缓冲区中，这个Event.header.type一般都是DISPLAY_EVENT_VSYNC，因此在上面的processPendingEvents函数中会将Event数据保存在outCount所指向的内存中，并返回true。 接下来返回到NativeDisplayEventReceiver.handleEvent后会调用到dispatchVsync函数：

```c
void NativeDisplayEventReceiver::dispatchVsync(nsecs_t timestamp, int32_t id, uint32_t count) {
    JNIEnv* env = AndroidRuntime::getJNIEnv();
    env->CallVoidMethod(mReceiverObjGlobal, gDisplayEventReceiverClassInfo.dispatchVsync, timestamp, id, count);
    mMessageQueue->raiseAndClearException(env, "dispatchVsync");
}
```

这里的处理很直接，直接调用mReceiverObjGlobal对象在gDisplayEventReceiverClassInfo.dispatchVsync中指定的函数，将后面的timestamp（时间戳） id（设备ID） count（经过的同步信号的数量，一般没有设置采样频率应该都是1），下面分别看一下mReceiverObjGlobal以及gDisplayEventReceiverClassInfo.dispatchVsync代表的是什么？ （1）mReceiverObjGlobal

```c
NativeDisplayEventReceiver::NativeDisplayEventReceiver(JNIEnv* env, jobject receiverObj, const sp<MessageQueue>& messageQueue) :
        mReceiverObjGlobal(env->NewGlobalRef(receiverObj)), mMessageQueue(messageQueue), mWaitingForVsync(false) {
    ALOGV("receiver %p ~ Initializing input event receiver.", this);
}
```

可以看到mReceiverObjGlobal是创建NativeDisplayEventReceiver对象时传进来的第二个参数，该对象是在nativeInit函数中创建：

```c
sp receiver = new NativeDisplayEventReceiver(env, receiverObj, messageQueue);
```

进一步的，receiverObj是调用nativeInit函数时传进来的第一个参数（第一个参数env是系统用于连接虚拟机时自动加上的），nativeInit函数又是在Choreographer中创建FrameDisplayEventReceiver对象时，在基类DisplayEventReceiver构造器中调用的，因此这里的mReceiverObjGlobal对应的就是Choreographer中的FrameDisplayEventReceiver成员mDisplayEventReceiver。 （2）gDisplayEventReceiverClassInfo.dispatchVsync 在JNI中有很多这样的类似的结构体对象，这些对象都是全局结构体对象，这里的gDisplayEventReceiverClassInfo就是这样的一个对象，里面描述了一些在整个文件内可能会调用到的java层的相关类以及成员函数的相关信息，看一下gDisplayEventReceiverClassInfo：

```c
static struct {
    jclass clazz;
    jmethodID dispatchVsync;
    jmethodID dispatchHotplug;
} gDisplayEventReceiverClassInfo;
```

看一下里面的变量名称就能知道大致的含义，clazz成员代表的是某个java层的类的class信息，dispatchVsync和dispatchHotplug代表的是java层类的方法的方法信息，看一下该文件中注册JNI函数的方法：

```c
int register_android_view_DisplayEventReceiver(JNIEnv* env) {
    int res = RegisterMethodsOrDie(env, "android/view/DisplayEventReceiver", gMethods, NELEM(gMethods));
    jclass clazz = FindClassOrDie(env, "android/view/DisplayEventReceiver");
    gDisplayEventReceiverClassInfo.clazz = MakeGlobalRefOrDie(env, clazz);
    gDisplayEventReceiverClassInfo.dispatchVsync = GetMethodIDOrDie(env, gDisplayEventReceiverClassInfo.clazz, "dispatchVsync", "(JII)V");
    gDisplayEventReceiverClassInfo.dispatchHotplug = GetMethodIDOrDie(env, gDisplayEventReceiverClassInfo.clazz, "dispatchHotplug", "(JIZ)V");
    return res;
}
```

RegisterMethodsOrDie调用注册了java层调用native方法时链接到的函数的入口，下面clazz对应的就是java层的"android/view/DisplayEventReceiver.java"类，gDisplayEventReceiverClassInfo.dispatchVsync里面保存的就是clazz类信息中与dispatchVsync方法相关的信息，同样dispatchHotplug也是。 分析到这里，就知道应用进程native接收到同步信号事件后，会调用Choreographer中的FrameDisplayEventReceiver成员mDisplayEventReceiver的dispatchVsync方法。

### 6.5.4、应用接收Vsync

看一下FrameDisplayEventReceiver.dispatchVsync方法，也就是DisplayEventReceiver.dispatchVsync方法(Choreographer.java)：

```java
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

貌似这里的处理只是往Choreographer对象中的mHandler对应的线程Looper中发送一个消息，消息的内容有两个特点： （1）将this，也就是当前的FrameDisplayEventReceiver对象作为参数，后面会回调到FrameDisplayEventReceiver.run方法； （2）为Message设置FLAG_ASYNCHRONOUS属性； 发送这个FLAG_ASYNCHRONOUS消息后，后面会回调到FrameDisplayEventReceiver.run方法，至于为什么，后面再写文章结合View.invalidate方法的过程分析，看一下FrameDisplayEventReceiver.run方法：

```java
@Override
public void run() {
    mHavePendingVsync = false;
    doFrame(mTimestampNanos, mFrame);
}
```

调用Choreographer.doFrame方法，如果是重绘事件doFrame方法会最终调用到ViewRootImpl.performTraversals方法进入实际的绘制流程。经过上面的分析可以知道，调用一次Choreographer.scheduleVsyncLocked只会请求一次同步信号，也就是回调一次FrameDisplayEventReceiver.onVsync方法，在思考一个问题，一个应用进程需要多次请求Vsync同步信号会不会使用同样的一串对象？多个线程又是怎么样的？ 答：一般绘制操作只能在主线程里面进行，因此一般来说只会在主线程里面去请求同步信号，可以认为不会存在同一个应用的多个线程请求SF的Vsync信号，Choreographer是一个线程内的单例模式，存储在了 ThreadLocal sThreadInstance对象里面，所以主线程多次请求使用的是同一个Choreographer对象，所以后面的一串对象应该都是可以复用的。

总体架构： 伐木累:::终于完了，由于Android Graphics系统涉及模块代码纵横交叉复杂，其中代码图示有误的地方请见谅，也没有精力一一核对了，还请海涵~~~主要是分析Android Graphics总体的一个流程思想，有需要再一点点深挖。

![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.graphics/30-Android-Graphics-Arc-flow.png)

## （七）、参考文档(特别感谢各位前辈的分析和图示)：

[Android Vsync 原理](http://www.10tiao.com/html/431/201601/401709603/1.html)
[林学森的Android专栏](http://blog.csdn.net/uiop78uiop78/article/category/1390680)
[Android Graphics](http://coderprof.com/android_tutorials_pdf_examples_PDF.php?q=android+graphics+pdf)
[了解 Systrace](https://source.android.com/devices/tech/debug/systrace)
[图解Android - Android GUI 系统](http://www.cnblogs.com/samchen2009/category/524173.html) 
[Android SurfaceFlinger 学习之路&Android多媒体开发](http://windrunnerlihuan.com/)
[Android7.0 基础业务AMS、数据业务、电源管理业务 源码分析](http://blog.csdn.net/Gaugamela/article/category/6383486) 
[【Android 显示模块】 - 深入剖析Android系统 - CSDN博客](http://blog.csdn.net/yangwen123/article/category/1647761)
[深入理解Android卷一全文-第八章(深入理解Surface系统)](http://blog.csdn.net/innost/article/details/47208337) 
[android系统 - armwind的专栏 - CSDN博客](http://blog.csdn.net/armwind/article/category/6282972)
[android显示系统 - kc58236582的博客 - CSDN博客](http://blog.csdn.net/kc58236582/article/category/6436488) 
[SurfaceView, TextureView, SurfaceTexture等的区别](http://www.cnblogs.com/wytiger/p/5693569.html)
[【Demo】Android graphics 学习－生产者、消费者、BufferQueue介绍](http://blog.csdn.net/armwind/article/details/73436532)
[深入Android Graphics Pipeline：从按钮到帧缓冲（第一部分）](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0421/2765.html)
[深入Android Graphics Pipeline：从按钮到帧缓冲（第二部分）](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0421/2766.html)
[窗口：Profiling 视图（OpenGL/OpenGL ES* 工作负载）](https://software.intel.com/en-us/node/713326)
[Android N中UI硬件渲染（hwui）的HWUI_NEW_OPS(基于Android 7.1)](http://blog.csdn.net/jinzhuojun/article/details/54234354)
[Android's Graphics Buffer Management System (Part I: gralloc)](https://netaz.blogspot.jp/search/label/gralloc)
[Android's Graphics Buffer Management System (Part II: BufferQueue)](https://netaz.blogspot.jp/search/label/Graphics)
[Android GDI之SurfaceFlinger](https://www.kancloud.cn/digest/androidcore/149097)
[Android SurfaceFlinger 学习之路(五)----VSync 工作原理](http://windrunnerlihuan.com/)
[Android 5.1 SurfaceFlinger VSYNC详解](http://blog.csdn.net/michaelcao1980/article/details/43233765) 
[Android 5.1 SurfaceFlinger VSYNC详解](http://blog.csdn.net/newchenxf/article/details/49131167)
[Android消息机制Looper与VSync的传播](http://blog.csdn.net/houliang120/article/details/50958212)
[Android垂直同步信号VSync的产生及传播结构详解](http://blog.csdn.net/houliang120/article/details/50908098) 
[Android 4.4(KitKat)中VSync信号的虚拟化](http://blog.csdn.net/jinzhuojun/article/details/17293325)
[Android 4.4(KitKat)窗口管理子系统 - 体系框架](http://blog.csdn.net/jinzhuojun/article/details/37737439) 
[Android中用OpenGL ES Tracer分析绘制过程](http://blog.csdn.net/ariesjzj/article/category/1087829/2)
[android view的绘制中，View绘制的时间如何和vsync屏幕刷新频率保持同步的？](https://www.zhihu.com/question/30372696?sort=created)
[【深入理解Android卷一全文-第八章】入理解Surface系统](http://blog.csdn.net/innost/article/details/47208337) 
[Android 窗口管理：Z-Order管理](http://blog.csdn.net/april_12345/article/details/52933316)
