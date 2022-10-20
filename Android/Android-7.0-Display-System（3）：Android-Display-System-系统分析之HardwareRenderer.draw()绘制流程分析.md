---
title: Android N Display System（3）：Android Display System 系统分析之HardwareRenderer.draw()绘制流程分析
cover: https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/hexo.themes/bing-wallpaper-2018.04.21.jpg
categories: 
  - Display
tags:
  - Android
  - Graphics
  - Display
toc: true
abbrlink: 20180608
date: 2018-06-08 09:25:00
---


--------------------------------------------------------------------------------

注：文章都是通过阅读各位前辈总结的资料、Android 7.1.2 && Linux（kernel 3.18）Qualcomm平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除，禁止转载（©Qualcomm Technologies, Inc. 版权所有），谢谢。

[【特别感谢 - Android应用程序UI硬件加速渲染技术简要介绍和学习计划】](https://blog.csdn.net/luoshengyang/article/details/45601143)
[【特别感谢 - Android N中UI硬件渲染（hwui）的HWUI_NEW_OPS(基于Android 7.1)】](https://blog.csdn.net/jinzhuojun/article/details/54234354)
[【特别感谢 - Android DisplayList 构建过程】](https://www.jianshu.com/p/7bf306c09c7e)

Google Pixel、Pixel XL 内核代码（文章基于 Kernel-3.18）：
 [Kernel source for Pixel and Pixel XL - GitHub](https://github.com/matthewdalex/marlin)

AOSP 源码（文章基于 Android 7.1.2）：
 [ Android 系统全套源代码分享 (更新到 8.1.0_r1)](https://testerhome.com/topics/2229)

 🌀🌀：专注于Linux && Android Multimedia（Camera、Video、Audio、Display）系统分析与研究

【Android Display System 系统分析系列】：
【Android Display System（1）：Android 7.1.2 (Android N) Android Graphics 系统 分析】
【Android Display System（2）：Android Display System 系统分析之Android EGL && OpenGL】
【Android Display System（3）：Android Display System 系统分析之HardwareRenderer.draw()绘制流程分析】
【Android Display System（4）：Android Display System 系统分析之Gralloc && HWComposer模块分析】
【Android Display System（5）：Android Display System 系统分析之Display Driver Architecture】 

--------------------------------------------------------------------------------

\frameworks\base\core\java\android\view\
- ViewRootImpl.java
- RenderNode.java
- View.java
- DisplayListCanvas.java

\frameworks\base\core\jni\
- android_view_DisplayListCanvas.cpp
- android_view_RenderNode.cpp
- android_view_ThreadedRenderer.cpp
- android_view_Surface.cpp
- android_server_AssetAtlasService.cpp

\frameworks\base\core\jni\android\graphics\
- Graphics.cpp
- Bitmap.cpp

\frameworks\base\libs\hwui\
- AssetAtlas.cpp
- BakedOpDispatcher.cpp
- BakedOpRenderer.cpp
- DeferredDisplayList.cpp
- DeferredLayerUpdater.cpp
- DisplayList.cpp
- DisplayListCanvas.cpp
- LayerBuilder.cpp
- LayerRenderer.cpp
- LayerUpdateQueue.cpp
- Patch.cpp
- RecordingCanvas.cpp
- RenderNode.cpp

\frameworks\base\libs\hwui\hwui\
- Canvas.cpp

\frameworks\base\libs\hwui\renderthread\
- EglManager.cpp
- CanvasContext.cpp
- DrawFrameTask.cpp
- RenderProxy.cpp
- RenderTask.cpp
- RenderThread.cpp

--------------------------------------------------------------------------------

UI作为用户体验的核心之一，始终是Android每次升级中的重点。从Androd 3.0(Honeycomb)开始，Android开始支持hwui（UI硬件加速）。到Android 4.0（ICS）时，硬件加速被默认开启。同时ICS还引入了DisplayList的概念（不是OpenGL里的那个），它相当于是从View的绘制命令到GL命令之间的“中间语言”。它记录了绘制该View所需的全部信息，之后只要重放（replay）即可完成内容的绘制。这样如果View没有改动或只部分改动，便可重用或修改DisplayList，从而避免调用了一些上层代码，提高了效率。Android 4.3（JB）中引入了DisplayList的defer操作，它主要用于对DisplayList中命令进行Batch（批次）和Merge（合并）。这样可以减少GL draw call和context切换以提高效率。之后，在Android 5.0（Lollipop）中又引入了RenderNode（渲染节点）的概念，它是对DisplayList及一些View显示属性的进一步封装。代码上，一个View对应一个RenderNode（Native层对应同名类），其中管理着对应的DisplayList和OffscreenBuffer（如果该View为硬件绘制层）。每个向WindowManagerService注册的窗口对应一个RootRenderNode，通过它可以找到View层次结构中所有View的DisplayList信息。在Java层的DisplayListCanvas用于生成DisplayList，其在native层的对应类为RecordingCanvas（在Android N前为DisplayListCanvas）。另外Android L中还引入了RenderThread（渲染线程）。所有的GL命令执行都放到这个线程上。渲染线程在RenderNode中存有渲染帧的所有信息，且还监听VSync信号，因此可以独立做一些属性动画。这样即便主线程block也可以保证动画流畅。引入渲染线程后ThreadedRenderer替代了Gl20Renderer，作为proxy用于主线程（UI线程）把渲染任务交给渲染线程。近期，在Android 7.0（Nougat）中又对hwui进行了小规模重构，引入了BakedOpRenderer, FrameBuilder, LayerBuilder, RecordingCanvas等类，用宏HWUI_NEW_OPS管理。下面简单介绍下这些新成员：

☯ **RecordingCanvas**: 之前Java层的DisplayListCanvas对应native层的DisplayListCanvas。引入RecordingCanvas后，其在native层的对应物就变成了RecordingCanvas。和DisplayListCanvas类似，画在RecordingCanvas上的内容都会被记录在RenderNode的DisplayList中。

☯ **BakedOpRenderer**: 顾名思义，就是用于绘制batch/merge好的操作。用于替代之前的OpenGLRenderer。它是真正用GL绘制到on-screen surface上的。

☯ **BakedOpDispatcher**: 提供一系列onXXX（如onBitmapOp）和onMergedXXX（如onMergedBitmapOps）静态函数供replay时调用。这些dispatch函数最后一般都会通过GlopBuilder来构造Glop然后通过BakedOpRenderer的renderGlop()函数来用OpenGL绘制。

☯ **LayerBuilder**: 用于存储绘制某一层的操作和状态。替代了部分原DeferredDisplayList的工作。对于所有View通用，即如果View有render layer，它对应一个FBO；如果对于普通View，它对应的是SurfaceFlinger提供的surface。 其中的mBatches存储了当前层defer后（即batch/merge好）的绘制操作。

☯ **FrameBuilder**: 管理某一帧的构建，用于处理，优化和存储从RenderNode和LayerUpdateQueue中来的渲染命令，同时它的replayBakedOps()方法还用于该帧的绘制命令重放。一帧中可能需要绘制多个层，每一层的上下文都会存在相应的LayerBuilder中。在FrameBuilder中通过mLayerBuilders和mLayerStack存储一个layer stack。它替代了原Snapshot类的一部分功能。

☯ **OffscreenBuffer**: 用于替代Layer类，但是设计上更轻量，而且自带内存池（通过OffscreenBufferPool）。

☯ **LayerUpdateQueue**：用于记录类型为硬件绘制层的RenderNode的更新操作。之后会通过FrameBuilder将该layer对应的RenderNode通过deferNodeOps()方法进行处理。

☯ **RecordedOp**: 由RecordedCanvas将View中的绘制命令转化为RecordedOp。RecordedOp也是DisplayList中的基本元素，用于替代Android N之前的DisplayListOp。它有一坨各式各样的继承类代表各种各样的绘制操作。BakedOpState是RecordedOp和相应的状态的自包含封装（封装的过程称为bake）。

☯ **BatchBase**: LayerBuilder中对DisplayList进行batch/merge处理后的结果以BatchBase形式保存在LayerBuilder的mBatches成员中。它有两个继承类分别为OpBatch和MergingOpBatch，分别用于不可合并和可合并操作。

概括下它们和相关类的关系图如下，接下来从DisplayList(RenderNode)的构建和绘制两个阶段分析下具体有哪些改动。

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/display.system/DS-03-01-android-hw-ops.png)

首先看看总体时序图：然后一步一步分析：

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/display.system/DS-03-02-HW.Draw.png)


#### （一）、Android硬件渲染环境初始化ViewRootImpl.enableHardwareAcceleration()
在ViewRootImpl.java的setView里，会去enable硬件加速功能。如果当前创建的窗口支持硬件加速渲染，那么就会创建一个HardwareRenderer对象，这个HardwareRenderer对象以后将负责执行窗口硬件加速渲染的相关操作。
``` java
[->\frameworks\base\core\java\android\view\ViewRootImpl.java]
private void enableHardwareAcceleration(WindowManager.LayoutParams attrs) {
        ......
        if (hardwareAccelerated) {
            ......
            if (fakeHwAccelerated) {
                ......
            } else if (!ThreadedRenderer.sRendererDisabled
                    || (ThreadedRenderer.sSystemRendererDisabled && forceHwAccelerated)) {
                ......
                mAttachInfo.mHardwareRenderer = ThreadedRenderer.create(mContext, translucent);
                ......
            }
        }
    }
```
ThreadedRenderer类的静态成员函数create的实现如下所示：
##### 1.1、ThreadedRenderer创建过程
``` java
[->frameworks\base\core\java\android\view\ThreadedRenderer.java]
    public static ThreadedRenderer create(Context context, boolean translucent) {
        ThreadedRenderer renderer = null;
        if (DisplayListCanvas.isAvailable()) {
            renderer = new ThreadedRenderer(context, translucent);
        }
        return renderer;
    }
    ThreadedRenderer(Context context, boolean translucent) {
        ......
        //1、nCreateRootRenderNode在Native层创建了一个Render Node
        long rootNodePtr = nCreateRootRenderNode();
        mRootNode = RenderNode.adopt(rootNodePtr);
        ......
        //2、nCreateProxy在Native层创建了一个Render Proxy对象
        mNativeProxy = nCreateProxy(translucent, rootNodePtr);
        //3、初始化一个系统预加载资源的地图集,优化资源的内存使用
        ProcessInitializer.sInstance.init(context, mNativeProxy);

        loadSystemProperties();
    }
```
##### 1.1.1、创建RenderNode
nCreateRootRenderNode在Native层创建了一个Render Node。 从这里就可以看出，窗口在Native层的Root Render Node实际上是一个RootRenderNode对象。
``` cpp
[->\frameworks\base\core\jni\android_view_ThreadedRenderer.cpp]
class RootRenderNode : public RenderNode, ErrorHandler {
public:
    RootRenderNode(JNIEnv* env) : RenderNode() {
        mLooper = Looper::getForThread();
        env->GetJavaVM(&mVm);
    }
    ......
}

static jlong android_view_ThreadedRenderer_createRootRenderNode(JNIEnv* env, jobject clazz) {
    RootRenderNode* node = new RootRenderNode(env);
    node->incStrong(0);
    node->setName("RootRenderNode");
    return reinterpret_cast<jlong>(node);
}
```
可以看出会创建一个C++层的RootRenderNode对象，RootRenderNode继承自RenderNode，去看看RenderNode构造函数。

``` cpp
[->\frameworks\base\libs\hwui\RenderNode.cpp]
RenderNode::RenderNode()
        : mDirtyPropertyFields(0)
        , mNeedsDisplayListSync(false)
        , mDisplayList(nullptr)
        , mStagingDisplayList(nullptr)
        , mAnimatorManager(*this)
        , mParentCount(0) {
}
```
有了这个RootRenderNode对象之后，函数android_view_ThreadedRenderer_createProxy就创建了一个RenderProxy对象。
##### 1.1.2、创建RenderProxy

``` cpp
[->\frameworks\base\core\jni\android_view_ThreadedRenderer.cpp]
static jlong android_view_ThreadedRenderer_createProxy(JNIEnv* env, jobject clazz,
        jboolean translucent, jlong rootRenderNodePtr) {
    RootRenderNode* rootRenderNode = reinterpret_cast<RootRenderNode*>(rootRenderNodePtr);
    ContextFactoryImpl factory(rootRenderNode);
    return (jlong) new RenderProxy(translucent, rootRenderNode, &factory);
}
```
RenderProxy对象的创建过程如下所示

``` cpp
 [->\frameworks\base\libs\hwui\renderthread\RenderProxy.cpp]
 RenderProxy::RenderProxy(bool translucent, RenderNode* rootRenderNode, IContextFactory* contextFactory)
        : mRenderThread(RenderThread::getInstance())
        , mContext(nullptr) {
    SETUP_TASK(createContext);
    args->translucent = translucent;
    args->rootRenderNode = rootRenderNode;
    args->thread = &mRenderThread;
    args->contextFactory = contextFactory;
    mContext = (CanvasContext*) postAndWait(task);
    mDrawFrameTask.setContext(&mRenderThread, mContext, rootRenderNode);
}
```
RenderProxy类有三个重要的成员变量mRenderThread、mContext和mDrawFrameTask，mRenderThread描述的就是Render Thread，mContext描述的是一个画布上下文，mDrawFrameTask描述的是一个用来执行渲染任务的Task

##### 1.1.2.1、RenderThread 实例化
来看看RenderThread::getInstance()实例化过程

``` cpp
[->\frameworks\base\libs\hwui\renderthread\RenderThread.cpp]
RenderThread& RenderThread::getInstance() {
    static RenderThread* sInstance = new RenderThread();
    gHasRenderThreadInstance = true;
    return *sInstance;
}
RenderThread::RenderThread() : Thread(true)
        , mNextWakeup(LLONG_MAX)
        , mDisplayEventReceiver(nullptr)
        , mVsyncRequested(false)
        , mFrameCallbackTaskPending(false)
        , mFrameCallbackTask(nullptr)
        , mRenderState(nullptr)
        , mEglManager(nullptr) {
    Properties::load();
    mFrameCallbackTask = new DispatchFrameCallbacks(this);
    mLooper = new Looper(false);
    run("RenderThread");
}

//RenderThread类的成员变量mFrameCallbackTask描述的Task是用来做什么的呢？原来就是用来显示动画的。当Java层注册一个动画
//类型的Render Node到Render Thread时，一个类型为IFrameCallback的回调接口就会通过RenderThread类的成员函数
//postFrameCallback注册到Render Thread的一个Pending Registration Frame Callbacks列表中

class DispatchFrameCallbacks : public RenderTask {
private:
    RenderThread* mRenderThread;
public:
    DispatchFrameCallbacks(RenderThread* rt) : mRenderThread(rt) {}

    virtual void run() override {
        mRenderThread->dispatchFrameCallbacks();
    }
};

void RenderThread::dispatchFrameCallbacks() {
   ......
    if (callbacks.size()) {
        ......
        requestVsync();
        for (std::set<IFrameCallback*>::iterator it = callbacks.begin(); it != callbacks.end(); it++) {
            (*it)->doFrame();
        }
    }
}
```
1、mFrameCallbackTask指向一个DispatchFrameCallbacks对象，用来描述一个帧绘制任务
2、mLooper指向一个Looper对象，消息驱动模型
3、RenderThread类是从Thread类继承下来的，当我们调用它的成员函数run的时候，就会创建一个新的线程。这个新的线程的入口点函数为RenderThread类的成员函数threadLoop，它的实现如下所示：

``` cpp
[->\frameworks\base\libs\hwui\renderthread\RenderThread.cpp]
bool RenderThread::threadLoop() {
    setpriority(PRIO_PROCESS, 0, PRIORITY_DISPLAY);
    initThreadLocals();

    int timeoutMillis = -1;
    for (;;) {
        int result = mLooper->pollOnce(timeoutMillis);

        nsecs_t nextWakeup;
        // Process our queue, if we have anything
        while (RenderTask* task = nextTask(&nextWakeup)) {
            task->run();
            // task may have deleted itself, do not reference it again
        }
        ......

        if (mPendingRegistrationFrameCallbacks.size() && !mFrameCallbackTaskPending) {
            drainDisplayEventQueue();
            mFrameCallbacks.insert(
                    mPendingRegistrationFrameCallbacks.begin(), mPendingRegistrationFrameCallbacks.end());
            mPendingRegistrationFrameCallbacks.clear();
            requestVsync();
        }

        if (!mFrameCallbackTaskPending && !mVsyncRequested && mFrameCallbacks.size()) {
            requestVsync();
        }
    }

    return false;
}
```
这里我们就可以看到Render Thread的运行模型：
1、空闲的时候，Render Thread就睡眠在成员变量mLooper指向的一个Looper对象的成员函数pollOnce中。
2、当其它线程需要调度Render Thread，就会向它的任务队列增加一个任务，然后唤醒Render Thread进行处理。Render Thread通过成员函数nextTask获得需要处理的任务，并且调用它的成员函数run进行处理。
RenderThread类的成员函数nextTask的实现如下所示：

``` cpp
[->\frameworks\base\libs\hwui\renderthread\RenderThread.cpp]
RenderTask* RenderThread::nextTask(nsecs_t* nextWakeup) {
    AutoMutex _lock(mLock);
    RenderTask* next = mQueue.peek();
    if (!next) {
        mNextWakeup = LLONG_MAX;
    } else {
        ......
        if (next->mRunAt <= 0 || next->mRunAt <= systemTime(SYSTEM_TIME_MONOTONIC)) {
            next = mQueue.next();
        } else {
            next = nullptr;
        }
    }
    ......
    return next;
}
```
 注意，如果没有下一个任务可以执行，那么RenderThread类的成员函数nextTask通过参数nextWakeup返回的值为LLONG_MAX，表示Render Thread接下来无限期进入睡眠状态，直到被其它线程唤醒为止。
 RenderThread类提供了queue、queueAndWait、queueAtFront和queueAt四个成员函数向Task Queue增加一个Task，它们的实现如下所示：

``` cpp
[->\frameworks\base\libs\hwui\renderthread\RenderThread.cpp]
void RenderThread::queue(RenderTask* task) {
    AutoMutex _lock(mLock);
    mQueue.queue(task);
    if (mNextWakeup && task->mRunAt < mNextWakeup) {
        mNextWakeup = 0;
        mLooper->wake();
    }
}

void RenderThread::queueAndWait(RenderTask* task) {
    Mutex mutex;
    Condition condition;
    SignalingRenderTask syncTask(task, &mutex, &condition);

    AutoMutex _lock(mutex);
    queue(&syncTask);
    condition.wait(mutex);
}

void RenderThread::queueAtFront(RenderTask* task) {
    AutoMutex _lock(mLock);
    mQueue.queueAtFront(task);
    mLooper->wake();
}

void RenderThread::queueAt(RenderTask* task, nsecs_t runAtNs) {
    task->mRunAt = runAtNs;
    queue(task);
}
```
再来看Render Thread在进入无限循环之前调用的initThreadLocals()函数

``` cpp
[->\frameworks\base\libs\hwui\renderthread\RenderThread.cpp]
void RenderThread::initThreadLocals() {
    sp<IBinder> dtoken(SurfaceComposerClient::getBuiltInDisplay(
            ISurfaceComposer::eDisplayIdMain));
    status_t status = SurfaceComposerClient::getDisplayInfo(dtoken, &mDisplayInfo);
    ......
    initializeDisplayEventReceiver();
    mEglManager = new EglManager(*this);
    mRenderState = new RenderState(*this);
    mJankTracker = new JankTracker(mDisplayInfo);
}

```
1、initializeDisplayEventReceiver创建和初始化一个DisplayEventReceiver对象，用来接收Vsync信号。
产生Vsync信号时，SurfaceFlinger服务（Vsync信号由SurfaceFlinger服务进行管理和分发）会通过上述文件描述符号唤醒Render Thread。这时候Render Thread就会调用RenderThread类的静态成员函数displayEventReceiverCallback()->drainDisplayEventQueue()->
2、创建一个EglManager对象，之后会调用EglManager::initialize() Open GL ES环境初始化
3、创建一个RenderState对象（记录Render Thread当前的一些渲染状态）

##### 1.1.2.2、CanvasContext(画布上下文)的初始化过程
了解了Render Thread的创建过程之后，回到RenderProxy类的构造函数中，接下来我们继续分析它的成员变量mContext的初始化过程，也就是画布上下文的初始化过程。这是通过向Render Thread发送一个createContext命令来完成的。为了方便描述，我们将相关的代码列出来，如下所示：

``` cpp
[->\frameworks\base\libs\hwui\renderthread\RenderProxy.cpp]
#define ARGS(method) method ## Args  
  
#define CREATE_BRIDGE4(name, a1, a2, a3, a4) CREATE_BRIDGE(name, a1,a2,a3,a4,,,,)  
  
#define CREATE_BRIDGE(name, a1, a2, a3, a4, a5, a6, a7, a8) \  
    typedef struct { \  
        a1; a2; a3; a4; a5; a6; a7; a8; \  
    } ARGS(name); \  
    static void* Bridge_ ## name(ARGS(name)* args)  
  
#define SETUP_TASK(method) \  
    .......  
    MethodInvokeRenderTask* task = new MethodInvokeRenderTask((RunnableMethod) Bridge_ ## method); \  
    ARGS(method) *args = (ARGS(method) *) task->payload()  
  
  
CREATE_BRIDGE4(createContext, RenderThread* thread, bool translucent,  
        RenderNode* rootRenderNode, IContextFactory* contextFactory) {  
    return new CanvasContext(*args->thread, args->translucent,  
            args->rootRenderNode, args->contextFactory);  
}  
  
```
 我们首先看宏SETUP_TASK，它需要一个函数作为参数。这个函数通过CREATE_BRIDGEX来声明，其中X是一个数字，数字的大小就等于函数需要的参数的个数。例如，通过CREATE_BRIDGE4声明的函数有4个参数。在上面的代码段中，我们通过CREATE_BRIDGE4宏声明了一个createContext函数。

宏SETUP_TASK的作用创建一个类型MethodInvokeRenderTask的Task。这个Task关联有一个由CREATE_BRIDGEX宏声明的函数。例如，SETUP_TASK(createContext)创建的MethodInvokeRenderTask关联的函数是由CREATE_BRIDGE4声明的函数createContext。这个Task最终会通过RenderProxy类的成员函数postAndWait添加到Render Thread的Task Queue中，如下所示：

``` cpp
[->\frameworks\base\libs\hwui\renderthread\RenderProxy.cpp]
void* RenderProxy::postAndWait(MethodInvokeRenderTask* task) {
    void* retval;
    task->setReturnPtr(&retval);
    SignalingRenderTask syncTask(task, &mSyncMutex, &mSyncCondition);
    AutoMutex _lock(mSyncMutex);
    mRenderThread.queue(&syncTask);
    mSyncCondition.wait(mSyncMutex);
    return retval;
}
```
RenderProxy对象的创建过程就分析完成了，从中我们也看到Render Thread的创建过程和运行模型，以及Render Proxy与Render Thread的交互模型，总结来说：

1、RenderProxy内部有一个成员变量mRenderThread，它指向的是一个RenderThread对象，通过它可以向Render Thread线程发送命令。

2、 RenderProxy内部有一个成员变量mContext，它指向的是一个CanvasContext对象，Render Thread的渲染工作就是通过它来完成的。

3、RenderProxy内部有一个成员变量mDrawFrameTask，它指向的是一个DrawFrameTask对象，Main Thread通过它向Render Thread线程发送渲染下一帧的命令。

#### （二）、Open GL ES环境初始化 (绑定窗口到Render Thread中)

Activity窗口的绘制流程是在ViewRoot(Impl)类的成员函数performTraversals发起的。在绘制之前，首先要获得一个Surface。这个Surface描述的就是一个窗口。因此，一旦获得了对应的Surface，就需要将它绑定到Render Thread中，如下所示：

``` java
[->/frameworks/base/core/java/android/view/ViewRootImpl.java]
public final class ViewRootImpl implements ViewParent,  
        View.AttachInfo.Callbacks, HardwareRenderer.HardwareDrawCallbacks {  
    ......  
    private void performTraversals() {  
        ......
        hwInitialized = mAttachInfo.mHardwareRenderer.initialize(  
                                        mSurface);  
    }
}  
```
mHardwareRenderer对象，它的成员函数initialize的实现如下所示：

``` java
[->\frameworks\base\core\java\android\view\ThreadedRenderer.java]
    boolean initialize(Surface surface) throws OutOfResourcesException {
        boolean status = !mInitialized;
        mInitialized = true;
        updateEnabledState(surface);
        nInitialize(mNativeProxy, surface);
        return status;
    }
```
nInitialize是一个JNI函数，由Native层的函数android_view_ThreadedRenderer_initialize实现

``` cpp
[->\frameworks\base\core\jni\android_view_ThreadedRenderer.cpp]
static void android_view_ThreadedRenderer_initialize(JNIEnv* env, jobject clazz,
        jlong proxyPtr, jobject jsurface) {
    RenderProxy* proxy = reinterpret_cast<RenderProxy*>(proxyPtr);
    sp<Surface> surface = android_view_Surface_getSurface(env, jsurface);
    proxy->initialize(surface);
}
```
Java层的Surface在Native层对应的是一个ANativeWindow。我们可以通过函数android_view_Surface_getNativeWindow来获得一个Java层的Surface在Native层对应的ANativeWindow
RenderProxy->initialize()函数实现

``` cpp
CREATE_BRIDGE2(initialize, CanvasContext* context, Surface* surface) {
    args->context->initialize(args->surface);
    return nullptr;
}

void RenderProxy::initialize(const sp<Surface>& surface) {
    SETUP_TASK(initialize);
    args->context = mContext;
    args->surface = surface.get();
    post(task);
}

```
当这个Task在Render Thread中执行时，由宏CREATE_BRIDGE2声明的函数initialize就会被执行。

在由宏CREATE_BRIDGE2声明的函数initialize中，参数context指向的是RenderProxy类的成员变量mContext指向的一个CanvasContext对象，而参数window指向的ANativeWindow就是要绑定到Render Thread的ANativeWindow。

由宏CREATE_BRIDGE2声明的函数initialize通过调用参数context指向的CanvasContext对象的成员函数initialize来绑定参数window指向的ANativeWindow，如下所示：

``` cpp
[->\frameworks\base\libs\hwui\renderthread\CanvasContext.cpp]
void CanvasContext::initialize(Surface* surface) {
    setSurface(surface);
#if !HWUI_NEW_OPS
    if (mCanvas) return;
    mCanvas = new OpenGLRenderer(mRenderThread.renderState());
    mCanvas->initProperties();
#endif
}

void CanvasContext::setSurface(Surface* surface) {
    mNativeSurface = surface;
    ......
    if (surface) {
        mEglSurface = mEglManager.createSurface(surface);
    }

    if (mEglSurface != EGL_NO_SURFACE) {
        ......
        mHaveNewSurface = true;
        .....
    } else {
        mRenderThread.removeFrameCallback(this);
    }
}
```
每一个Open GL渲染上下文都需要关联有一个EGL Surface。这个EGL Surface描述的是一个绘图表面，它封装的实际上是一个ANativeWindow。有了这个EGL Surface之后，我们在执行Open GL命令的时候，才能确定这些命令是作用在哪个窗口上。

``` cpp
[->\frameworks\base\libs\hwui\renderthread\EglManager.cpp]
EGLSurface EglManager::createSurface(EGLNativeWindowType window) {
    initialize();
    EGLSurface surface = eglCreateWindowSurface(mEglDisplay, mEglConfig, window, nullptr);
    ......
    }

    return surface;
}
void EglManager::initialize() {
    ......
    loadConfig();//
    createContext();//
    createPBufferSurface();//
    makeCurrent(mPBufferSurface);//
    DeviceInfo::initialize();
    mRenderThread.renderState().onGLContextCreated();
    initAtlas();
}
```
> 使用EGL的绘图的一般步骤：
> 1、获取 EGL Display 对象：eglGetDisplay() 
> 2、初始化与 EGLDisplay 之间的连接：eglInitialize() 
> 3、获取 EGLConfig 对象：eglChooseConfig() 
> 4、创建 EGLContext 实例：eglCreateContext() 
> 5、创建 EGLSurface 实例：eglCreateWindowSurface() 
> 6、连接 EGLContext 和 EGLSurface：eglMakeCurrent() 
> 7、使用 OpenGL ES API 绘制图形：gl_*() 
> 8、切换 front buffer 和 back buffer 送显：eglSwapBuffer() 
> 9、断开并释放与 EGLSurface 关联的 EGLContext 对象：eglRelease() 
> 10、删除 EGLSurface 对象 
> 11、删除 EGLContext 对象 
> 12、终止与 EGLDisplay 之间的连接

Android EGL && OpenGL分析请参考【Android Display System（2）：Android Display System 系统分析 之 Android EGL && OpenGL】

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/display.system/DS-03-03-UIThread-ThreadRenderer.png)


至此，将当前窗口绑定到Render Thread的过程就分析完成了，整个Android应用程序UI硬件加速渲染环境的初始化过程也分析完成了。
#### （三）、RenderNode构建

我们知道，通常情况下，App要完成一帧渲染，是通过ViewRootImpl的performTraversals()函数来实现。而它又可分为measure, layout, draw三个阶段。上面这些改动主要影响的是最后这步，因此我们就主要focus在draw这个阶段的流程。首先看DisplayList是怎么录制的。在ViewRootImpl::performDraw()中会调用draw()函数。当判断需要进行绘制时（比如有脏区域，或在动画中时），又如果硬件加速可用（通过ThreadedRenderer的isEnabled()），会进行下面的重绘动作。接下来根据是否有相关请求（如resize时）或offset是否有变化来判断是否要调用ThreadedRenderer的invalidRoot()来标记更新RootRenderNode。

扯个题外话。和Android M相比，N中UI子系统中加入了不少对用户进行窗口resize的处理，主要应该是为了Android N新增加的多窗口分屏模式。比如当用户拖拽分屏窗口边缘时，onWindowDragResizeStart()被调用。它其中会创建BackdropFrameRenderer。BackdropFrameRenderer本身运行单独的线程，它负责在resize窗口而窗口绘制来不及的情况下填充背景。它会通过addRenderNode()加入专用的RenderNode。同时，Android N中将DecorView从PhoneWindow中分离成一个单独的文件，并实现新加的WindowCallbacks接口。它主要用于当用户变化窗口大小时ViewRootImpl对DecorView的回调。因为ViewRootImpl和WindowManagerService通信，它会被通知到窗口变化，然后回调到DecorView中。而DecorView中的相应回调会和BackupdropFrameRenderer交互。如updateContentDrawBounds()中最后会调用到了BackupdropFrmeRenderer的onContentDrawn()函数，其返回值代表在下面的内容绘制后是否需要再发起一次绘制。如果需要，之后会调用requestDrawWindow()。

回到ViewRootImpl::performDraw()函数，接下来，最重要的就是通过ThreadedRenderer的draw()来进行绘制。在这个draw()函数中，比较重要的一步是通过updateRootDisplayList()函数来更新根结点的DisplayList。

``` java
[->\frameworks\base\core\java\android\view\ThreadedRenderer.java]
    private void updateViewTreeDisplayList(View view) {
        view.mPrivateFlags |= View.PFLAG_DRAWN;
        view.mRecreateDisplayList = (view.mPrivateFlags & View.PFLAG_INVALIDATED)
                == View.PFLAG_INVALIDATED;
        view.mPrivateFlags &= ~View.PFLAG_INVALIDATED;
        view.updateDisplayListIfDirty();
        view.mRecreateDisplayList = false;
    }
    private void updateRootDisplayList(View view, HardwareDrawCallbacks callbacks) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Record View#draw()");
        updateViewTreeDisplayList(view);

        if (mRootNodeNeedsUpdate || !mRootNode.isValid()) {
            DisplayListCanvas canvas = mRootNode.start(mSurfaceWidth, mSurfaceHeight);
            try {
                final int saveCount = canvas.save();
                canvas.translate(mInsetLeft, mInsetTop);
                callbacks.onHardwarePreDraw(canvas);

                canvas.insertReorderBarrier();
                canvas.drawRenderNode(view.updateDisplayListIfDirty());
                canvas.insertInorderBarrier();

                callbacks.onHardwarePostDraw(canvas);
                canvas.restoreToCount(saveCount);
                mRootNodeNeedsUpdate = false;
            } finally {
                mRootNode.end(canvas);
            }
        }
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
......
```
函数updateRootDisplayList()中的updateViewTreeDisplayList()会调到DecorView的updateDisplayListIfDirty()函数。这个函数主要功能是更新DecorView对应的RenderNode中的DisplayList。它返回的RenderNode会通过RecordingCanvas::drawRenderNode()函数将之作为RenderNodeOp加入到RootRenderNode的DisplayList中。函数updateDisplayListIfDirty()中首先判断当前View是否需要更新。如果不需要就调用dispatchGetDisplayList()让子View更新，然后直接返回。否则就是当前View的DisplayList需要更新。这里我们假设是第一次绘制，更新DisplayList的流程首先通过RenderNode的start()来获得一个用于记录绘制操作的Canvas，即DisplayListCanvas（在Android M中Java层由GLES20RecordingCanvas改为DisplayListCanvas，native层中的DisplayListRenderer改为DisplayListCanvas，Android N中native层中的DisplayListCanvas改为RecordingCanvas）。

接下去就是比较关键的步骤了。这里就要分几种情况了，一个View可以为三种类型（LAYER_TYPE_NONE, LAYER_TYPE_SOFTWARE, LAYER_TYPE_HARDWARE）中的一种。LAYER_TYPE_NONE为默认值，代表没有layer。LAYER_TYPE_SOFTWARE代表该View有软件层，以bitmap为back，内容用软件渲染。LAYER_TYPE_HARDWARE和LAYER_TYPE_SOFTWARE类似，区别在于其有硬件层，以FBO（Framebuffer object）为back，内容使用硬件渲染。如果硬件加速没有打开，它的行为和LAYER_TYPE_SOFTWARE是一样的。

``` cpp
[->\frameworks\base\core\java\android\view\ThreadedRenderer.java]
    public RenderNode updateDisplayListIfDirty() {
        final RenderNode renderNode = mRenderNode;
       .......
        if ((mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == 0
                || !renderNode.isValid()
                || (mRecreateDisplayList)) {
            // Don't need to recreate the display list, just need to tell our
            // children to restore/recreate theirs
            if (renderNode.isValid()
                    && !mRecreateDisplayList) {
                mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
                mPrivateFlags &= ~PFLAG_DIRTY_MASK;
                dispatchGetDisplayList();

                return renderNode; // no work needed
            }

            // If we got here, we're recreating it. Mark it as such to ensure that
            // we copy in child display lists into ours in drawChild()
            mRecreateDisplayList = true;

            int width = mRight - mLeft;
            int height = mBottom - mTop;
            int layerType = getLayerType();

            final DisplayListCanvas canvas = renderNode.start(width, height);
            canvas.setHighContrastText(mAttachInfo.mHighContrastText);

            try {
                if (layerType == LAYER_TYPE_SOFTWARE) {
                    buildDrawingCache(true);
                    Bitmap cache = getDrawingCache(true);
                    if (cache != null) {
                        canvas.drawBitmap(cache, 0, 0, mLayerPaint);
                    }
                } else {
                    computeScroll();

                    canvas.translate(-mScrollX, -mScrollY);
                    mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
                    mPrivateFlags &= ~PFLAG_DIRTY_MASK;

                    // Fast path for layouts with no backgrounds
                    if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
                        dispatchDraw(canvas);
                        if (mOverlay != null && !mOverlay.isEmpty()) {
                            mOverlay.getOverlayView().draw(canvas);
                        }
                    } else {
                        draw(canvas);
                    }
                }
            } finally {
                renderNode.end(canvas);
                setDisplayListProperties(renderNode);
            }
        } else {
            mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
            mPrivateFlags &= ~PFLAG_DIRTY_MASK;
        }
        return renderNode;
    }

```
如果当前View是软件渲染层（类型为LAYER_TYPE_SOFTWARE）的话，则调用buildDrawingCache()获得Bitmap后调用drawBitmap()将该Bitmap记录到DisplayListCanvas中。现在Android中都默认硬件渲染了，为什么还要考虑软件渲染层呢?一方面有些平台不支持硬件渲染，或app不启用硬件加速，另一方面有些UI控件不支持硬件渲染 。在复杂的View（及子View）在动画过程中，可以被绘制成纹理，这样只需要画一次。显然，在View经常更新的情况下并不适用。因为这样每次都需要重新用软件渲染，如果硬件渲染打开时还要上传成硬件纹理（上传纹理是个比较慢的操作）。类似的，硬件渲染层（LAYER_TYPE_HARDWARE）也是适用于类似的复杂View结构进行属性动画的场景，但它与LAYER_TYPE_SOFTWARE的层的区别为它对应FBO，可以直接硬件渲染生成纹理。因此渲染的过程中不需要先生成Bitmap，从而省去了上传成硬件纹理的这一步操作。

如果当前View对应LAYER_TYPE_NONE或者LAYER_TYPE_HARDWARE，下面会考查是否为没有背景的Layout。这种情况下当前View没什么好画的，会走快速路径。即通过dispatchDraw()直接让子View重绘。否则就调draw()来绘制当前View及其子View。注意View中的draw()有两个重载同名函数。一个参数的版本用于直接调用。三个参数的版本用于ViewGroup中drawChild()时调用。这里调的是一个参数的版本。这个draw()函数中会按下面的顺序进行绘制（DisplayList的更新）：

``` java
    @CallSuper
    public void draw(Canvas canvas) {
        final int privateFlags = mPrivateFlags;
        final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
                (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
        mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

        /*
         * Draw traversal performs several drawing steps which must be executed
         * in the appropriate order:
         *
         *      1. Draw the background
         *      2. If necessary, save the canvas' layers to prepare for fading
         *      3. Draw view's content
         *      4. Draw children
         *      5. If necessary, draw the fading edges and restore layers
         *      6. Draw decorations (scrollbars for instance)
         */

        // Step 1, draw the background, if needed
        ......
        drawBackground(canvas);

        // skip step 2 & 5 if possible (common case)
        final int viewFlags = mViewFlags;
        boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
        boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
        
        // Step 3, draw the content
        if (!dirtyOpaque) onDraw(canvas);

        // Step 4, draw the children
        dispatchDraw(canvas);

        // Step 5, draw the fade effect and restore layers
        final Paint p = scrollabilityCache.paint;
        final Matrix matrix = scrollabilityCache.matrix;
        final Shader fade = scrollabilityCache.shader;

        if (drawTop) {
            matrix.setScale(1, fadeHeight * topFadeStrength);
            matrix.postTranslate(left, top);
            fade.setLocalMatrix(matrix);
            p.setShader(fade);
            canvas.drawRect(left, top, right, top + length, p);
        }
        ......

        canvas.restoreToCount(saveCount);

        // Overlay is part of the content and draws beneath Foreground
        if (mOverlay != null && !mOverlay.isEmpty()) {
            mOverlay.getOverlayView().dispatchDraw(canvas);
        }

        // Step 6, draw decorations (foreground, scrollbars)
        onDrawForeground(canvas);
    }

```
这些是通过drawBackground(), onDraw(), dispatchDraw()和onDrawForeground()等函数实现。这些函数本质上就是将相应内容绘制到提供的DisplayListCanvas上。由于View是以树形层次结构组织的，draw()中会通过dispatchDraw()来更新子View的DisplayList。dispatchDraw()为对每个子View调用drawChild()。然后调用子View的draw()函数（这次就是上面说的draw()的三个参数的版本了）。这个版本的draw()函数里会更新其View的DisplayList，然后调用DisplayListCanvas的drawRenderNode()将该子view对应的RenderNode记录到其父view的DisplayList中去。这样便根据View的树型结构生成了DisplayList的树型结构。

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/display.system/DS-03-04-DisplayList-RootRenderNode.png)

其中onDraw()用于绘制当前View的自定义UI，它是每个View需要自定义的成员函数。比较典型地，在View的绘制函数中会调用canvas的drawXXX函数。比如canvas.drawLine()->drawLines(android_graphics_Canvas.cpp)，它会通过JNI最后调到RecordingCanvas.cpp中的RecordingCanvas::drawLines()：

``` cpp
[->\frameworks\base\libs\hwui\RecordingCanvas.cpp]
void RecordingCanvas::drawLines(const float* points, int floatCount, const SkPaint& paint) {
    if (CC_UNLIKELY(floatCount < 4 || PaintUtils::paintWillNotDraw(paint))) return;
    floatCount &= ~0x3; // round down to nearest four

    addOp(alloc().create_trivial<LinesOp>(
            calcBoundsOfPoints(points, floatCount),
            *mState.currentSnapshot()->transform,
            getRecordedClip(),
            refPaint(&paint), refBuffer<float>(points, floatCount), floatCount));
}
```
RecordingCanvas中绝大多数的drawXXX系函数都是类似于这样，通过addOp()将一个RecordedOp的继承类存到其成员mDisplayList中。RecordedOp家庭成员很多，有不少继承类，每个对应一种操作。操作的种类可以参照下这个表：

``` cpp
[->\frameworks\base\libs\hwui\RecordedOp.h]
#define MAP_OPS_BASED_ON_TYPE(PRE_RENDER_OP_FN, RENDER_ONLY_OP_FN, UNMERGEABLE_OP_FN, MERGEABLE_OP_FN) \
        PRE_RENDER_OP_FN(RenderNodeOp) \
        PRE_RENDER_OP_FN(CirclePropsOp) \
        PRE_RENDER_OP_FN(RoundRectPropsOp) \
        PRE_RENDER_OP_FN(BeginLayerOp) \
        PRE_RENDER_OP_FN(EndLayerOp) \
        PRE_RENDER_OP_FN(BeginUnclippedLayerOp) \
        PRE_RENDER_OP_FN(EndUnclippedLayerOp) \
        PRE_RENDER_OP_FN(VectorDrawableOp) \
        \
        RENDER_ONLY_OP_FN(ShadowOp) \
        RENDER_ONLY_OP_FN(LayerOp) \
        RENDER_ONLY_OP_FN(CopyToLayerOp) \
        RENDER_ONLY_OP_FN(CopyFromLayerOp) \
        \
        UNMERGEABLE_OP_FN(ArcOp) \
        UNMERGEABLE_OP_FN(BitmapMeshOp) \
        UNMERGEABLE_OP_FN(BitmapRectOp) \
        UNMERGEABLE_OP_FN(ColorOp) \
        UNMERGEABLE_OP_FN(FunctorOp) \
        UNMERGEABLE_OP_FN(LinesOp) \
        UNMERGEABLE_OP_FN(OvalOp) \
        UNMERGEABLE_OP_FN(PathOp) \
        UNMERGEABLE_OP_FN(PointsOp) \
        UNMERGEABLE_OP_FN(RectOp) \
        UNMERGEABLE_OP_FN(RoundRectOp) \
        UNMERGEABLE_OP_FN(SimpleRectsOp) \
        UNMERGEABLE_OP_FN(TextOnPathOp) \
        UNMERGEABLE_OP_FN(TextureLayerOp) \
        \
        MERGEABLE_OP_FN(BitmapOp) \
        MERGEABLE_OP_FN(PatchOp) \
        MERGEABLE_OP_FN(TextOp)
```
各个View的DisplayList更新好后，回到udpateRootDisplayList()。如果发现RootRenderNode也需要更新，则先通过Java层的RenderNode::start()获得DisplayListCanvas，在这个Canvas上的动作都会被记录到DisplayList中，直到调用RenderNode.end()。然后为了防止对上下文状态的影响，用Canvas::save()和Canvas::restoreToCount()来生成临时的画布状态。再接下来就是通过drawRenderNode()将DecorView的RenderNode以RenderNodeOp的形式记录到RootRenderNode。

**DisplayList构建实例：**
[Android DisplayList 构建过程](https://www.jianshu.com/p/7bf306c09c7e)
**activity_main.xml**

```
链接：https://www.jianshu.com/p/7bf306c09c7e
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <TextView
        android:id="@+id/sample_text"
        android:layout_width="match_parent"
        android:layout_height="60dp"
        android:gravity="center"
        android:textSize="20sp"
        android:text="Hello World!" />
    
    <cc.bobby.debugapp.MyView
        android:layout_width="match_parent"
        android:layout_height="100dp" />
</LinearLayout>
```

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/display.system/DS-03-05-activity_main.xml.png)


**udpateRootDisplayList()**

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/display.system/DS-03-06-updateRootDisplayList.jpg)


**父View与子View的DisplayList**

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/display.system/DS-03-07-rootview-childview-displaylist.png)


#### （四）、RenderNode绘制
在ThreadedRenderer的draw()函数中构建完DisplayList后，接下来需要准备渲染了。首先通过JNI调用nSyncAndDrawFrame()调用到native层的android_view_ThreadedRenderer_syncAndDrawFrame()。其中将参数中的FrameInfo数组传到RenderProxy的mFrameInfo成员中。它是Android M开始加入用来细化hwui性能统计的。同时调用RenderProxy的syncAndDrawFrame()函数，并将创建的TreeObserver作为参数。函数syncAndDrawFrame()中即调用DrawFrameTask（这是RenderThread的TaskQueue中的特殊Task实例）的drawFrame()函数。继而通过postAndWait()往RenderThread的TaskQueue里插入自身（即DrawFrameTask）来申请新一帧的渲染。在RenderThread的queue()函数中会按Task的运行时间将之插入到适当的位置。接着postAndWait()函数中会block UI线程等待渲染线程将之unblock。渲染线程在N中的改动不大，这里就不花太多文字介绍了，需要的时候把它当作跨线程调用即可。

另一边，渲染线程处理这个DrawFrameTask时会调用到其run()函数：

``` cpp
[->\frameworks\base\libs\hwui\renderthread\DrawFrameTask.cpp]
void DrawFrameTask::run() {
    ATRACE_NAME("DrawFrame");

    bool canUnblockUiThread;
    bool canDrawThisFrame;
    {
        TreeInfo info(TreeInfo::MODE_FULL, *mContext);
        info.observer = mObserver;
        canUnblockUiThread = syncFrameState(info);
        canDrawThisFrame = info.out.canDrawThisFrame;
    }

    // Grab a copy of everything we need
    CanvasContext* context = mContext;

    // From this point on anything in "this" is *UNSAFE TO ACCESS*
    if (canUnblockUiThread) {
        unblockUiThread();
    }

    if (CC_LIKELY(canDrawThisFrame)) {
        context->draw();
    } else {
        // wait on fences so tasks don't overlap next frame
        context->waitOnFences();
    }

    if (!canUnblockUiThread) {
        unblockUiThread();
    }
}
```
其中首先通过DrawFrameTask::syncFrameState()函数将主线程的渲染信息（如DisplayList，Property和Bitmap等）同步到渲染线程。

``` cpp
syncFrameState()
  -> eglMakeCurrent 
    -> eglMakeCurrent(OEM EGL) 
      -> egl_window_surface_v2_t::connect()
      -> Surface::hook_dequeueBuffer() 
        -> Surface::dequeueBuffer() 
            -> BpGraphicBufferProducer::dequeueBuffer()
            -> BpGraphicBufferProducer::requestBuffer()

```

这个函数中首先会处理DrawFrameTask中的mLayers。它是DeferredLayerUpdater的vector，顾名思义，就是延迟处理的layer更新任务。这主要用于TextureView。TextureView是比较特殊的类。它通常用于显示内容流，生产者端可以是另一个进程。中间通过BufferQueue进行buffer的传输和交换。当有新的buffer来到（或者有属性变化，如visibility等）是，会通过回调设置标志位(mUpdateLayer)并通过invalidate()调度下一次重绘。当下一次draw()被调用时，先通过applyUpdate()->updateSurfaceTexture()->ThreadedRenderer::pushLayerUpdate()，再调到渲染线程中的 DrawFrameTask::pushLayerUpdate()，将本次更新记录在DrawFrameTask的mLayers中。这样，在后面调用DrawFrameTask::syncFrameState()是会依次调用mLayers中的apply()进行真正的更新。这里调用它的apply()函数就会取新可用buffer（通过doUpdateTexImage()函数），并将相关纹理信息更新到mLayer。在syncFrameState()函数中，接下来，通过CanvasContext的prepareTree()继而调用RenderNode的prepareTree()同步渲染信息。最后会输出TreeInfo结构，其中的prepareTextures代表纹理上传是否成功。如果为false，说明texture cache用完了。这样为了防止渲染线程在渲染过程中使用的资源和主线程竞争，在渲染线程绘制当前帧时就不能让主线程继续往下跑了，也就不能做到真正并行。在sync完数据后，DrawFrameTask::run()最后会调用CanvasContext::draw()来进行接下来的渲染。这部分的大体流程如下：

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/display.system/DS-03-08-ThreadRender-draw.png)


接下来瞄下CanvasContext::draw()里做了什么。先要小小准备下EGL环境，比如通过EglManager的beginFrame()函数，

``` cpp
beginFrame()
  -> eglMakeCurrent 
    -> eglMakeCurrent(OEM EGL) 
      -> Surface::hook_dequeueBuffer() 
        -> Surface::dequeueBuffer() 
            -> BpGraphicBufferProducer::dequeueBuffer()
            -> BpGraphicBufferProducer::requestBuffer()

[->\frameworks\native\opengl\libagl\egl.cpp]
Frame EglManager::beginFrame(EGLSurface surface) {
    .....
    makeCurrent(surface);
    ....
}
bool EglManager::makeCurrent(EGLSurface surface, EGLint* errOut) {

    if (!eglMakeCurrent(mEglDisplay, surface, surface, mEglContext)) {
        ....
    }
    .....
}

EGLBoolean eglMakeCurrent(  EGLDisplay dpy, EGLSurface draw,
                            EGLSurface read, EGLContext ctx)
{

    if (ctx == EGL_NO_CONTEXT) {
        // if we're detaching, we need the current context
        current_ctx = (EGLContext)getGlThreadSpecific();
    } else {
       
        egl_surface_t* d = (egl_surface_t*)draw;
    }

    if (d) {
        if (d->connect() == EGL_FALSE) {
            return EGL_FALSE;
        }
    }
    return setError(EGL_BAD_ACCESS, EGL_FALSE);
}

EGLBoolean egl_window_surface_v2_t::connect()
{

    // dequeue a buffer
    int fenceFd = -1;
    if (nativeWindow->dequeueBuffer(nativeWindow, &buffer,
            &fenceFd) != NO_ERROR) {
        return setError(EGL_BAD_ALLOC, EGL_FALSE);
    }

    // wait for the buffer
    sp<Fence> fence(new Fence(fenceFd));
    if (fence->wait(Fence::TIMEOUT_NEVER) != NO_ERROR) {
        nativeWindow->cancelBuffer(nativeWindow, buffer, fenceFd);
        return setError(EGL_BAD_ALLOC, EGL_FALSE);
    }

    return EGL_TRUE;
}

```

继而用eglMakeCurrent()将渲染context切换到相应的surface。然后EglManager的damageFrame()设定当前帧的脏区域（如果gfx平台支持局部更新的话）。接下来就是绘制的主体部分了。这也是N中改动比较大的部分。

``` cpp
[->\frameworks\base\libs\hwui\renderthread\CanvasContext.cpp]
void CanvasContext::draw() {
    SkRect dirty;
    mDamageAccumulator.finish(&dirty);

    mCurrentFrameInfo->markIssueDrawCommandsStart();

    Frame frame = mEglManager.beginFrame(mEglSurface);
    ......
    SkRect screenDirty(dirty);
    ......
    mEglManager.damageFrame(frame, dirty);

#if HWUI_NEW_OPS
    auto& caches = Caches::getInstance();
    FrameBuilder frameBuilder(dirty, frame.width(), frame.height(), mLightGeometry, caches);

    frameBuilder.deferLayers(mLayerUpdateQueue);
    mLayerUpdateQueue.clear();

    frameBuilder.deferRenderNodeScene(mRenderNodes, mContentDrawBounds);

    BakedOpRenderer renderer(caches, mRenderThread.renderState(),
            mOpaque, mLightInfo);
    frameBuilder.replayBakedOps<BakedOpDispatcher>(renderer);
    profiler().draw(&renderer);
    bool drew = renderer.didDraw();

    // post frame cleanup
    caches.clearGarbage();
    caches.pathCache.trim();
    caches.tessellationCache.trim();
    ......
    mCanvas->prepareDirty(frame.width(), frame.height(),
            dirty.fLeft, dirty.fTop, dirty.fRight, dirty.fBottom, mOpaque);

    Rect outBounds;
    // It there are multiple render nodes, they are laid out as follows:
    // #0 - backdrop (content + caption)
    // #1 - content (positioned at (0,0) and clipped to - its bounds mContentDrawBounds)
    // #2 - additional overlay nodes
    ......
    // Draw all render nodes. Note that
    for (const sp<RenderNode>& node : mRenderNodes) {
        if (layer == 0) { // Backdrop.
            .......
            // Check if we have to draw something on the left side ...
            if (targetBounds.left < contentBounds.left) {
                mCanvas->save(SaveFlags::Clip);
                if (mCanvas->clipRect(targetBounds.left, targetBounds.top,
                                      contentBounds.left, targetBounds.bottom,
                                      SkRegion::kIntersect_Op)) {
                    mCanvas->drawRenderNode(node.get(), outBounds);
                }
                // Reduce the target area by the area we have just painted.
                targetBounds.left = std::min(contentBounds.left, targetBounds.right);
                mCanvas->restore();
            }
            // ... or on the right side ...
            // ... or at the top ...
            // ... or at the bottom.
        } else if (layer == 1) { // Content
            // It gets cropped against the bounds of the backdrop to stay inside.
            mCanvas->save(SaveFlags::MatrixClip);
            ......
            mCanvas->translate(dx, dy);
            if (mCanvas->clipRect(left, top, left + width, top + height, SkRegion::kIntersect_Op)) {
                mCanvas->drawRenderNode(node.get(), outBounds);
            }
            mCanvas->restore();
        } else { // draw the rest on top at will!
            mCanvas->drawRenderNode(node.get(), outBounds);
        }
        layer++;
    }

    profiler().draw(mCanvas);

    bool drew = mCanvas->finish();
    ......
```
先得到Caches的实例。它是一个单例类，包含了各种绘制资源的cache。然后创建FrameBuilder。该类用于当前帧的构建。FrameBuilder的构造函数中又会创建对应fbo0的LayerBuilder。fbo0即对应通过SurfaceFlinger申请来的on-screen surface，然后将之放入layer stack（通过mLayerBuilders和mLayerStack两个成员维护）。同时还会在initializeSaveStack()函数中创建和初始化Snapshot。就像名字一样，它保存了渲染surface的当前状态的一个“快照”。每个Snapshot有一个指向前继的Snapshot，从而形成一个"栈"。每次调用save()和restore()就相当于压栈和弹栈。

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/display.system/DS-03-09-frameBuilder.deferLayers.png)

接下来deferLayers()函数处理LayerUpdateQueue中的元素。之前在渲染线程每画一帧前同步信息时调用RenderNode::prepareTree()会遍历DisplayList的树形结构，对于子节点递归调用prepareTreeImpl()，如果是render layer，在RenderNode::pushLayerUpdate()中会将该layer的更新操作记录到LayerUpdateQueue中。至于哪些节点是render layer。主要是根据之前提到的view类型（LAYER_TYPE_NONE/SOFTWARE/HARDWARE）。但会有一个优化，如果一个普通view满足promotedToLayer()定义的条件，它会被当做render layer处理。

``` cpp
[->\frameworks\base\libs\hwui\FrameBuilder.cpp]
void FrameBuilder::deferLayers(const LayerUpdateQueue& layers) {
    // Render all layers to be updated, in order. Defer in reverse order, so that they'll be
    // updated in the order they're passed in (mLayerBuilders are issued to Renderer in reverse)
    for (int i = layers.entries().size() - 1; i >= 0; i--) {
        RenderNode* layerNode = layers.entries()[i].renderNode;
        // only schedule repaint if node still on layer - possible it may have been
        // removed during a dropped frame, but layers may still remain scheduled so
        // as not to lose info on what portion is damaged
        OffscreenBuffer* layer = layerNode->getLayer();
        if (CC_LIKELY(layer)) {
            ATRACE_FORMAT("Optimize HW Layer DisplayList %s %ux%u",
                    layerNode->getName(), layerNode->getWidth(), layerNode->getHeight());

            Rect layerDamage = layers.entries()[i].damage;
            // TODO: ensure layer damage can't be larger than layer
            layerDamage.doIntersect(0, 0, layer->viewportWidth, layer->viewportHeight);
            layerNode->computeOrdering();

            // map current light center into RenderNode's coordinate space
            Vector3 lightCenter = mCanvasState.currentSnapshot()->getRelativeLightCenter();
            layer->inverseTransformInWindow.mapPoint3d(lightCenter);

            saveForLayer(layerNode->getWidth(), layerNode->getHeight(), 0, 0,
                    layerDamage, lightCenter, nullptr, layerNode);

            if (layerNode->getDisplayList()) {
                deferNodeOps(*layerNode);
            }
            restoreForLayer();
        }
    }
}
```
回到deferLayers()函数。这里就是把LayerUpdateQueue里的元素按逆序拿出来，依次调用saveForLayer()，deferNodeOps()和restoreForLayer()。saveForLayer()为该render laye创建Snapshot和LayerBuilder并放进mLayerStack和mLayerBuilders。而restoreForLayer()则是它的逆操作。Layer stack和canvas state是栈的结构。saveForLayer() 和restoreForLayer()就相当于一个push stack，一个pop stack。这里核心的deferNodeOps()函数处理该layer对应的DisplayList，将它们按以下类型以batch的形式组织存放在LayerBuilder的mBatches成员中。其中同一类型中能合并的操作还会进行合并（目前只支持Bitmap, Text和Patch三种类型的操作合并）。Batch的类型有以下几种：

``` cpp
[->\frameworks\base\libs\hwui\LayerBuilder.h]
namespace OpBatchType {
    enum {
        Bitmap,
        MergedPatch,
        AlphaVertices,
        Vertices,
        AlphaMaskTexture,
        Text,
        ColorText,
        Shadow,
        TextureLayer,
        Functor,
        CopyToLayer,
        CopyFromLayer,

        Count // must be last
    };
}
```
下面看下deferNodeOps()函数里是怎么处理RenderNode的。

``` cpp
[->\frameworks\base\libs\hwui\FrameBuilder.cpp]
/**
 * Used to define a list of lambdas referencing private FrameBuilder::onXX::defer() methods.
 *
 * This allows opIds embedded in the RecordedOps to be used for dispatching to these lambdas.
 * E.g. a BitmapOp op then would be dispatched to FrameBuilder::onBitmapOp(const BitmapOp&)
 */
#define OP_RECEIVER(Type) \
        [](FrameBuilder& frameBuilder, const RecordedOp& op) { frameBuilder.defer##Type(static_cast<const Type&>(op)); },
void FrameBuilder::deferNodeOps(const RenderNode& renderNode) {
    typedef void (*OpDispatcher) (FrameBuilder& frameBuilder, const RecordedOp& op);
    static OpDispatcher receivers[] = BUILD_DEFERRABLE_OP_LUT(OP_RECEIVER);

    // can't be null, since DL=null node rejection happens before deferNodePropsAndOps
    const DisplayList& displayList = *(renderNode.getDisplayList());
    for (auto& chunk : displayList.getChunks()) {
        FatVector<ZRenderNodeOpPair, 16> zTranslatedNodes;
        buildZSortedChildList(&zTranslatedNodes, displayList, chunk);

        defer3dChildren(chunk.reorderClip, ChildrenSelectMode::Negative, zTranslatedNodes);
        for (size_t opIndex = chunk.beginOpIndex; opIndex < chunk.endOpIndex; opIndex++) {
            const RecordedOp* op = displayList.getOps()[opIndex];
            receivers[op->opId](*this, *op);

            if (CC_UNLIKELY(!renderNode.mProjectedNodes.empty()
                    && displayList.projectionReceiveIndex >= 0
                    && static_cast<int>(opIndex) == displayList.projectionReceiveIndex)) {
                deferProjectedChildren(renderNode);
            }
        }
        defer3dChildren(chunk.reorderClip, ChildrenSelectMode::Positive, zTranslatedNodes);
    }
}

```
DisplayList以chunk为单位组合RecordedOp。这些RecordedOp的opId代表它们的类型。根据这个类型调用receivers这个查找表（通过BUILD_DEFERABLE_OP_LUT构造）中的函数。它会调用FrameBuilder中相应的deferXXX函数（比如deferArcOp, deferBitmapOp, deferRenderNodeOp等）。这些deferXXX系函数一般会将RecordedOp用BakedOpState封装一下，然后会调用LayerBuilder的deferUnmergeableOp()和deferMergeableOp()函数将BakedOpState组织进mBatches成员。同时还有两个查找表mBatchLookup和mMergingBatchLookup分别用于不能合并的batch（OpBatch）和能合并的batch（MergingOpBatch）。它们分别用于查找特定类型的最近一个OpBatch或者MergingOpBatch。

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/display.system/DS-03-10-DisplayList-chunk.png)


先看下deferUnmergeableOp()函数。它会将BakedOpState按batch类型放进mBatches中。mBatches是指向BatchBase对象的vector，每个处理好的BakedOpState都会按类型放进来。如果还未有该类型的batch则创建OpBatch，并把它插入到mBatches的末尾。同时插入mBatchLookup这个查找表（batchId到最近一个该类型的OpBatch对象的映射）。这样之后处理同类型的BakedOpState时候，就会先搜索这个查找表。假如找到了，则进一步在mBatches数组中找到相应的OpBatch并通过它的batchOp()将该BakedOpState加入。

``` cpp
[->\frameworks\base\libs\hwui\LayerBuilder.cpp]
void LayerBuilder::deferUnmergeableOp(LinearAllocator& allocator,
        BakedOpState* op, batchid_t batchId) {
    onDeferOp(allocator, op);
    OpBatch* targetBatch = mBatchLookup[batchId];

    size_t insertBatchIndex = mBatches.size();
    if (targetBatch) {
        locateInsertIndex(batchId, op->computedState.clippedBounds,
                (BatchBase**)(&targetBatch), &insertBatchIndex);
    }

    if (targetBatch) {
        targetBatch->batchOp(op);
    } else  {
        // new non-merging batch
        targetBatch = allocator.create<OpBatch>(batchId, op);
        mBatchLookup[batchId] = targetBatch;
        mBatches.insert(mBatches.begin() + insertBatchIndex, targetBatch);
    }
}
```
接下来看用于合并操作的deferMergeableOp()函数。它也是类似的，当没有可以合并的MergingOpBatch时会创建新的，并且插入到mBatches。因为可能存在情况这个batchId在mBatches中有但是mMergingBatchLookup中没找到（说明还没有可合并的MergingOpBatch对象）或者通过MergingOpBatch::canMergeWidth()判断不满足合并条件。这时候就要插入到mBatches中该类型所在位置。如果很顺利的情况下，前面已经有MergingOpBatch在mMergingBatchLookup中而且又满足合并条件，就通过MergingOpBatch::mergeOp()将该BakedOpState和已有的进行合并。
``` cpp
[->\frameworks\base\libs\hwui\LayerBuilder.cpp]
void LayerBuilder::deferMergeableOp(LinearAllocator& allocator,
        BakedOpState* op, batchid_t batchId, mergeid_t mergeId) {
    onDeferOp(allocator, op);
    MergingOpBatch* targetBatch = nullptr;

    // Try to merge with any existing batch with same mergeId
    auto getResult = mMergingBatchLookup[batchId].find(mergeId);
    if (getResult != mMergingBatchLookup[batchId].end()) {
        targetBatch = getResult->second;
        if (!targetBatch->canMergeWith(op)) {
            targetBatch = nullptr;
        }
    }

    size_t insertBatchIndex = mBatches.size();
    locateInsertIndex(batchId, op->computedState.clippedBounds,
            (BatchBase**)(&targetBatch), &insertBatchIndex);

    if (targetBatch) {
        targetBatch->mergeOp(op);
    } else  {
        // new merging batch
        targetBatch = allocator.create<MergingOpBatch>(batchId, op);
        mMergingBatchLookup[batchId].insert(std::make_pair(mergeId, targetBatch));

        mBatches.insert(mBatches.begin() + insertBatchIndex, targetBatch);
    }
}
```

回到CanvasContext::draw()函数，处理好layer后，下面得就是通过FrameBuilder::deferRenderNodeScene()函数处理FrameBuilder成员mRenderNodes中的RenderNode，其中包含了RootRenderNode（也可能有其它的RenderNode，比如backdrop和overlay nodes）。对于每个RenderNode，如果需要绘制则调用FrameBuilder的deferRenderNode()函数：

``` cpp
[->\frameworks\base\libs\hwui\FrameBuilder.cpp]
void FrameBuilder::deferRenderNode(RenderNode& renderNode) {
    renderNode.computeOrdering();

    mCanvasState.save(SaveFlags::MatrixClip);
    deferNodePropsAndOps(renderNode);
    mCanvasState.restore();
}
```
这里和前面类似，会为之创建独立的Snapshot（Canvas渲染状态），deferNodePropsAndOps()根据RenderNode中的RenderProperties通过CanvasState设置一堆状态。如果该RenderNode对应是一个render layer，则将它封装为LayerOp（绘制offscreen buffer）并通过deferUnmergeableOp()加入batch。如果该RenderNode对应RenderProperties有半透明效果且不是render layer，则可以将该RenderNode绘制到一个临时的layer（称为save layer）。这是通过BeginLayerOp和EndLayerOp来记录的。正常情况下，还是通过deferNodeOps()来将RenderNode进行batch/merge。这个函数前面已有说明。

再次回到CanvasContext::draw()函数，下面终于要真得进行渲染了。首先创建BakedOpRenderer，然后调用FrameBuilder::replayBakedOps()函数并将BakedOpRenderer作为参数传进去。注意这是个模板函数，这里模板参数为BakedOpDispatcher。在replayBakedOps()函数中会构造两个用于处理BakedOpState的函数查找表。它们将BakedOpState按操作类型分发到BakedOpDispatcher的相应静态处理函数（onXXX或者onMergedXXX，分别用于非合并和合并的操作） 。

``` cpp
[->\frameworks\base\libs\hwui\FrameBuilder.h]
template <typename StaticDispatcher, typename Renderer>
    void replayBakedOps(Renderer& renderer) {
        std::vector<OffscreenBuffer*> temporaryLayers;
        finishDefer();
        /**
         * Defines a LUT of lambdas which allow a recorded BakedOpState to use state->op->opId to
         * dispatch the op via a method on a static dispatcher when the op is replayed.
         *
         * For example a BitmapOp would resolve, via the lambda lookup, to calling:
         *
         * StaticDispatcher::onBitmapOp(Renderer& renderer, const BitmapOp& op, const BakedOpState& state);
         */
        #define X(Type) \
                [](void* renderer, const BakedOpState& state) { \
                    StaticDispatcher::on##Type(*(static_cast<Renderer*>(renderer)), \
                            static_cast<const Type&>(*(state.op)), state); \
                },
        static BakedOpReceiver unmergedReceivers[] = BUILD_RENDERABLE_OP_LUT(X);
        #undef X

        /**
         * Defines a LUT of lambdas which allow merged arrays of BakedOpState* to be passed to a
         * static dispatcher when the group of merged ops is replayed.
         */
        #define X(Type) \
                [](void* renderer, const MergedBakedOpList& opList) { \
                    StaticDispatcher::onMerged##Type##s(*(static_cast<Renderer*>(renderer)), opList); \
                },
        static MergedOpReceiver mergedReceivers[] = BUILD_MERGEABLE_OP_LUT(X);
        #undef X
```
 如前面如述，之前已经在FrameBuilder中构造了LayerBuilder的stack。接下来，这儿就是按push时的逆序（z-order高到底）对其中的BakedOpState进行replay，因为下面的layer可能会依赖的上面layer的渲染结果。比如要把上面layer画在FBO上的东西当成纹理画到下一层layer上。对于layer（persistent或者temporary的），先在BakedOpRenderer::startRepaintLayer()中初始化相关GL环境，比如创建FBO，绑定layer对应OffscreenBuffer中的纹理，设置viewport，清color buffer等等。对应地，BakedOpRenderer::endLayer()中最相应的销毁和清理工作。中间调用LayerBuilder::replayBakedOpsImpl()函数做真正的replay动作。对于fbo0（即on-screen surface），也是类似的，只是把startRepaintLayer()和endLayer()换成BakedOpRenderer::startFrame()和BakedOpRenderer::endFrame()。它们的功能也是初始化和销毁GL环境。

``` cpp
[->\frameworks\base\libs\hwui\FrameBuilder.h]
template <typename StaticDispatcher, typename Renderer>
    void replayBakedOps(Renderer& renderer) {
        .....
       // Relay through layers in reverse order, since layers
        // later in the list will be drawn by earlier ones
        for (int i = mLayerBuilders.size() - 1; i >= 1; i--) {
            GL_CHECKPOINT(MODERATE);
            LayerBuilder& layer = *(mLayerBuilders[i]);
            if (layer.renderNode) {
                // cached HW layer - can't skip layer if empty
                renderer.startRepaintLayer(layer.offscreenBuffer, layer.repaintRect);
                GL_CHECKPOINT(MODERATE);
                layer.replayBakedOpsImpl((void*)&renderer, unmergedReceivers, mergedReceivers);
                GL_CHECKPOINT(MODERATE);
                renderer.endLayer();
            } else if (!layer.empty()) {
                // save layer - skip entire layer if empty (in which case, LayerOp has null layer).
                layer.offscreenBuffer = renderer.startTemporaryLayer(layer.width, layer.height);
                temporaryLayers.push_back(layer.offscreenBuffer);
                GL_CHECKPOINT(MODERATE);
                layer.replayBakedOpsImpl((void*)&renderer, unmergedReceivers, mergedReceivers);
                GL_CHECKPOINT(MODERATE);
                renderer.endLayer();
            }
        }

        GL_CHECKPOINT(MODERATE);
        if (CC_LIKELY(mDrawFbo0)) {
            const LayerBuilder& fbo0 = *(mLayerBuilders[0]);
            renderer.startFrame(fbo0.width, fbo0.height, fbo0.repaintRect);
            GL_CHECKPOINT(MODERATE);
            fbo0.replayBakedOpsImpl((void*)&renderer, unmergedReceivers, mergedReceivers);
            GL_CHECKPOINT(MODERATE);
            renderer.endFrame(fbo0.repaintRect);
        }

        for (auto& temporaryLayer : temporaryLayers) {
            renderer.recycleTemporaryLayer(temporaryLayer);
        }
    }
```
在replayBakedOpsImpl()函数中，会根据操作的类型调用前面生成的unmergedReceivers和mergedReceivers两个函数分发表中的对应处理函数。它们实质指向BakedOpDispatcher中的静态函数。这些函数onXXXOp()和onMergedXXXOps()函数大同小异，基本都是通过GlopBuilder将BakedOpState和相关的信息封装成Glop对象，然后调用BakedOpRenderer::renderGlop()，接着通过DefaultGlopReceiver()调用BakedOpRenderer::renderGlopImpl()函数，最后在RenderState::render()中通过GL命令将Glop渲染出来。大功告成。

**结语**
合并之后，DeferredDisplayList Vector<Batch * > mBatches 包含全部整合后的绘制命令，之后渲染即可，需要注意的是这里的合并并不是多个变一个，只是做了一个集合，主要是方便使用各资源纹理等，比如绘制文字的时候，需要根据文字的纹理进行渲染，而这个时候就需要查询文字的纹理坐标系，合并到一起方便统一处理，一次渲染，减少资源加载的浪费，当然对于理解硬件加速的整体流程，这个合并操作可以完全无视，甚至可以直观认为，构建完之后，就可以直接渲染，它的主要特点是在另一个Render线程使用OpenGL进行绘制，这个是它最重要的特点。而mBatches中所有的DrawOp都会通过OpenGL被绘制到GraphicBuffer中，最后通过swapBuffers函数调用queueBuffer()通知SurfaceFlinger合成。

``` cpp
[->\frameworks\native\opengl\libagl\egl.cpp]
EGLBoolean egl_window_surface_v2_t::swapBuffers()
{
......
nativeWindow->queueBuffer(nativeWindow, buffer); 
nativeWindow->dequeueBuffer(nativeWindow, &buffer);
......
}
```

总的来说，可以看到一个View上的东西要绘制出来，要经过多步的转化。

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/display.system/DS-03-11-View-draw-Surface.png)


这样做有几个好处：第一、对绘制操作进行batch/merge可以减少GL的draw call，从而减少渲染状态切换，提高了性能。第二、因为将View层次结构要绘制的东西转化为DisplayList这种“中间语言”的形式，当需要绘制时才转化为GL命令。因此在View中内容没有更改或只有部分属性更改时只要修改中间表示（即RenderNode和RenderProperties）即可，从而避免很多重复劳动。第三、由于DisplayList中包含了要绘制的所有信息，一些属性动画可以由渲染线程全权处理，无需主线程介入，主线程卡住也不会让界面卡住。另一方面，也可以看到一些潜力可挖。比如当前可以合并的操作类型有限。另外主线程和渲染线程间的很多调用还是同步的，并行度或许可以进一步提高。另外Vulkan的引入也可以帮助进一步榨干GPU的能力。
**例如：**

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/display.system/DS-03-12-one_button_draw.png)


绘制的批次按文本、图片资源、几何图形等进行分类，分批绘制的效果如下图所示:

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/display.system/DS-03-13-BakedOpDispatcher-onMergedBitmapOps.gif)



#### （五）、参考资料(特别感谢各位前辈的分析和图示)：
[Android N中UI硬件渲染（hwui）的HWUI_NEW_OPS(基于Android 7.1) ](https://blog.csdn.net/jinzhuojun/article/details/54234354)
[Android应用程序UI硬件加速渲染技术简要介绍和学习计划](https://blog.csdn.net/luoshengyang/article/details/45601143)
[【特别感谢 - Android N中UI硬件渲染（hwui）的HWUI_NEW_OPS(基于Android 7.1)】](https://blog.csdn.net/jinzhuojun/article/details/54234354)
[Android DisplayList 构建过程](https://www.jianshu.com/p/7bf306c09c7e)
[Android5.0中 hwui 中 RenderThread 工作流程](http://www.androidperformance.com/2015/08/12/AndroidL-hwui-RenderThread-workflow/)

