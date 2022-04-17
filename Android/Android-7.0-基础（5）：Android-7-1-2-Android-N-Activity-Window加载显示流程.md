---
title: Android N 基础（5）：Android 7.1.2 Activity - Window 加载显示流程（AMS && WMS）分析
cover: https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/hexo.themes/bing-wallpaper-2018.04.04.jpg
categories: 
  - Android
tags:
  - Android
toc: true
abbrlink: 20171208
date: 2017-12-08 09:25:00
---



Activity-Window加载显示流程概述： Android系统中图形系统是相当复杂的，包括WindowManager，SurfaceFlinger,Open GL,GPU等模块。 其中SurfaceFlinger作为负责绘制应用UI的核心，从名字可以看出其功能是将所有Surface合成工作。 不论使用什么渲染API, 所有的东西最终都是渲染到"surface". surface代表BufferQueue的生产者端, 并且 由SurfaceFlinger所消费, 这便是基本的生产者-消费者模式. Android平台所创建的Window都由surface所支持, 所有可见的surface渲染到显示设备都是通过SurfaceFlinger来完成的. 本文详细分析Android Window加载显示流程，并通过SurfaceFlinger渲染合成输出到屏幕的过程。 <!-- more -->

## 一、Activity启动流程概述

基于Android 7.1.2的源码剖析， 分析Activity-Window加载显示流程，相关源码：

--------------------------------------------------------------------------------

**frameworks/base/core/java/android/app/**
● Activity.java
● ActivityThread.java
● Instrumentation.java

**frameworks/base/core/jni/**
● android_view_DisplayEventReceiver.cpp<br>
● android_view_SurfaceControl.cpp
● android_view_Surface.cpp
● android_view_SurfaceSession.cpp

**frameworks/native/include/gui/**
● SurfaceComposerClient.cpp<br>
● SurfaceComposerClient.h

**frameworks/native/services/surfaceflinger/**
● SurfaceFlinger.cpp
● Client.cpp

**frameworks/base/core/java/android/view/**
● WindowManagerImpl.java
● ViewManager.java
● WindowManagerGlobal.java
● ViewRootImpl.java
● Choreographer.java
● IWindowSession.aidl
● DisplayEventReceiver.java
● SurfaceControl.java
● Surface.java
● SurfaceSession.java

**frameworks/base/core/java/com/android/internal/policy/**
● PhoneWindow.java

**frameworks/base/services/core/java/com/android/server/wm/**
● WindowManagerService.java
● Session.java
● WindowState.java
● WindowStateAnimator.java
● WindowSurfaceController.java


在前面文章（Android 7.1.2(Android N) Activity启动流程分析）中详细分析了Activity启动流程，这里回顾一下总体流程。 Activity启动流程图： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.addwindow/N1-Android-addWindow-start_activity_process.jpg)

> 启动流程：

> ● 点击桌面App图标，Launcher进程采用Binder IPC向system_server进程发起startActivity请求；
● system_server进程接收到请求后，向zygote进程发送创建进程的请求；
● Zygote进程fork出新的子进程，即App进程；
● App进程，通过Binder IPC向sytem_server进程发起attachApplication请求；
● system_server进程在收到请求后，进行一系列准备工作后，再通过binder IPC向App进程发送scheduleLaunchActivity请求；
● App进程的binder线程（ApplicationThread）在收到请求后，通过handler向主线程发送LAUNCH_ACTIVITY消息；
● 主线程在收到Message后，通过发射机制创建目标Activity，并回调Activity.onCreate()等方法。

App正式启动后，开始进入Activity生命周期，执行完onCreate/onStart/onResume方法，UI渲染结束后便可以看到App的主界面， 接下来分析UI渲染流程。

### 二、Window加载显示流程

#### 2.1、ActivityThread.handleLaunchActivity()

接着从ActivityThread的handleLaunchActivity方法： [->ActivityThread.java]

```java
    private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason){
    ......
    // Initialize before creating the activity
    WindowManagerGlobal.initialize();
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

应用程序进程通过performLaunchActivity函数将即将要启动的Activity加载到当前进程空间来，同时为启动Activity做准备。 [ActivityThread.java #performLaunchActivity()]

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    //通过Activity所在的应用程序信息及该Activity对应的CompatibilityInfo信息从PMS服务中查询当前Activity的包信息  
    ActivityInfo aInfo = r.activityInfo;
    if (r.packageInfo == null) {
        r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                Context.CONTEXT_INCLUDE_CODE);
    }
    //获取当前Activity的组件信息  
    ComponentName component = r.intent.getComponent();
    if (component == null) {
        component = r.intent.resolveActivity(
            mInitialApplication.getPackageManager());
        r.intent.setComponent(component);
    }
    //packageName为启动Activity的包名，targetActivity为Activity的类名  
    if (r.activityInfo.targetActivity != null) {
        component = new ComponentName(r.activityInfo.packageName,
                r.activityInfo.targetActivity);
    }
    //通过类反射方式加载即将启动的Activity
    Activity activity = null;
    try {
        java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
        StrictMode.incrementExpectedActivityCount(activity.getClass());
        r.intent.setExtrasClassLoader(cl);
        r.intent.prepareToEnterProcess();
        if (r.state != null) {
            r.state.setClassLoader(cl);
        }
    } ......
    try {
        //通过单例模式为应用程序进程创建Application对象  
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);

       ......
        if (activity != null) {
            //为当前Activity创建上下文对象ContextImpl  
            Context appContext = createBaseContextForActivity(r, activity);
            ......
            //将当前启动的Activity和上下文ContextImpl、Application绑定
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window);

            ......
            //将Activity保存到ActivityClientRecord中，ActivityClientRecord为Activity在应用程序进程中的描述符  
            r.activity = activity;
            activity.mCalled = false;
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
            ......
            //生命周期onStart、onresume
            if (!r.activity.mFinished) {
                activity.performStart();
                r.stopped = false;
            }

            //ActivityThread的成员变量mActivities保存了当前应用程序进程中的所有Activity的描述符
            mActivities.put(r.token, r);
            ......
    return activity;
}
```

在该函数中，首先通过PMS服务查找到即将启动的Activity的包名信息，然后通过类反射方式创建一个该Activity实例，同时为应用程序启动的每一个Activity创建一个LoadedApk实例对象，应用程序进程中创建的所有LoadedApk对象保存在ActivityThread的成员变量mPackages中。接着通过LoadedApk对象的makeApplication函数，使用单例模式创建Application对象，因此在android应用程序进程中有且只有一个Application实例。然后为当前启动的Activity创建一个ContextImpl上下文对象，并初始化该上下文，到此我们可以知道，启动一个Activity需要以下对象：

1) Activity对象，需要启动的Activity；

2) LoadedApk对象，每个启动的Activity都拥有属于自身的LoadedApk对象；

3) ContextImpl对象，每个启动的Activity都拥有属于自身的ContextImpl对象；

4) Application对象，应用程序进程中有且只有一个实例，和Activity是一对多的关系；

#### 2.2、Activity对象Attach过程

Activity所需要的对象都创建好了，就需要将Activity和Application对象、ContextImpl对象绑定在一起。

参数： context：Activity的上下文对象，就是前面创建的ContextImpl对象；

aThread：Activity运行所在的主线程描述符ActivityThread；

instr：用于监控Activity运行状态的Instrumentation对象；

token：用于和AMS服务通信的IApplicationToken.Proxy代理对象；

application：Activity运行所在进程的Application对象；

parent：启动当前Activity的Activity；

[->Activity.java]

```java
    final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor,
        Window window) {
        //将上下文对象ContextImpl保存到Activity的成员变量中  
    attachBaseContext(context);
    ......

    mWindow = new PhoneWindow(this, window);
    ......
    //记录应用程序的UI线程  
    mUiThread = Thread.currentThread();
    //记录应用程序的ActivityThread对象
    mMainThread = aThread;
    ......
    //为Activity所在的窗口创建窗口管理器
    mWindow.setWindowManager(
            (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
            mToken, mComponent.flattenToString(),
            (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    if (mParent != null) {
        mWindow.setContainer(mParent.getWindow());
    }
    mWindowManager = mWindow.getWindowManager();
    mCurrentConfig = config;
}
```

在该attach函数中主要做了以下几件事：

1) 根据参数初始化Activity的成员变量；

2) 为Activity创建窗口Window对象；

3) 为Window创建窗口管理器；

#### 2.3、Activity视图对象的创建过程-Activity.setContentView()

> ActivityThread.performLaunchActivity()
--> Instrumentation.callActivityOnCreate()
------>Activity.performCreate()
------>Activity.onCreate()
-->Activity.performStart()
------>Instrumentation.callActivityOnStart()
------>Activity.onStart()

```java
    public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
```

getWindow()函数得到前面创建的窗口对象PhoneWindow，通过PhoneWindow来设置Activity的视图。 [->PhoneWindow.java]

```java
    public void setContentView(View view, ViewGroup.LayoutParams params) {
    ......
    //如果窗口顶级视图对象为空，则创建窗口视图对象  
    if (mContentParent == null) {
        installDecor();
    }
    ......

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
    ......
    } else {//加载布局文件，并将布局文件中的所有视图对象添加到mContentParent容器中,PhoneWindow的成员变量mContentParent的类型为ViewGroup，是窗口内容存放的地方
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    mContentParent.requestApplyInsets();
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        cb.onContentChanged();
    }
    mContentParentExplicitlySet = true;
}
```

Activity.onCreate()会调用setContentView(),整个过程主要是Activity的布局文件或View添加至窗口里， 详细过程不再赘述，详细加载过程请参考： [Android应用setContentView与LayoutInflater加载解析机制源码分析](http://blog.csdn.net/yanbober/article/details/45970721) 重点概括为：

```java
 ● 创建一个DecorView的对象mDecor，该mDecor对象将作为整个应用窗口的根视图。
 ● 依据Feature等style theme创建不同的窗口修饰布局文件，并且通过findViewById获取Activity布局文件该存放的地方（窗口修饰布局文件中id为content的FrameLayout）。
 ● 将Activity的布局文件添加至id为content的FrameLayout内。
 ● 当setContentView设置显示OK以后会回调Activity的onContentChanged方法。Activity的各种View的findViewById()方法等都可以放到该方法中，系统会帮忙回调。
```

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.addwindow/N2-Android-addWindow-DecorView.jpg)

#### 2.4、ActivityThread.handleResumeActivity()

回到我们刚刚的handleLaunchActivity()方法，在调用完performLaunchActivity()方法之后，其有掉用了handleResumeActivity()法。

performLaunchActivity()方法完成了两件事：

1) Activity窗口对象的创建，通过attach函数来完成；

2) Activity视图对象的创建，通过setContentView函数来完成；

这些准备工作完成后，就可以显示该Activity了，应用程序进程通过调用handleResumeActivity函数来启动Activity的显示过程。

[->ActivityThread.java]

```java
    final void handleResumeActivity(IBinder token,
        boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
    ActivityClientRecord r = mActivities.get(token);
    ......
    //
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
            if (r.activity.mVisibleFromClient) {
                r.activity.makeVisible();
            }
        }

        if (!r.onlyLocalRequest) {
            //onStop()......
            Looper.myQueue().addIdleHandler(new Idler());
        }
        r.onlyLocalRequest = false;
        if (reallyResume) {
            try {
                ActivityManagerNative.getDefault().activityResumed(token);
            }
        }
         ......
    }
}
```

我们知道，在前面的performLaunchActivity函数中完成Activity的创建后，会将当前当前创建的Activity在应用程序进程端的描述符ActivityClientRecord以键值对的形式保存到ActivityThread的成员变量mActivities中：mActivities.put(r.token, r)，r.token就是Activity的身份证，即是IApplicationToken.Proxy代理对象，也用于与AMS通信。上面的函数首先通过performResumeActivity从mActivities变量中取出Activity的应用程序端描述符ActivityClientRecord，然后取出前面为Activity创建的视图对象DecorView和窗口管理器WindowManager，最后将视图对象添加到窗口管理器中。

ViewManager.addView()真正实现的的地方在WindowManagerImpl.java中。

```java
public interface ViewManager
{
public void addView(View view, ViewGroup.LayoutParams params);
public void updateViewLayout(View view, ViewGroup.LayoutParams params);
public void removeView(View view);
}
```

[->WindowManagerImpl.java]

```java
@Override
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

#### 2.5、ViewRootImpl()构造过程：

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

(6) 创建Surface对象，用于绘制当前视图，当然该Surface对象的真正创建是由WMS来完成的，只不过是WMS传递给应用程序进程的。 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.addwindow/N3-Android-addWindow-ViewRootImpl.png)

#### 2.6、IWindowSession代理获取过程

[->WindowManagerGlobal.java]

```java
   private static IWindowSession sWindowSession;
    public static IWindowSession getWindowSession() {
    synchronized (WindowManagerGlobal.class) {
        if (sWindowSession == null) {
            try {
                //获取输入法管理器
                InputMethodManager imm = InputMethodManager.getInstance();
                //获取窗口管理器  
                IWindowManager windowManager = getWindowManagerService();
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
@Override
public IWindowSession openSession(IWindowSessionCallback callback, IInputMethodClient client,
        IInputContext inputContext) {
    if (client == null) throw new IllegalArgumentException("null client");
    if (inputContext == null) throw new IllegalArgumentException("null inputContext");
    Session session = new Session(this, callback, client, inputContext);
    return session;
}
```

在WMS服务端构造了一个Session实例对象。ViewRootImpl 是一很重要的类，类似 ActivityThread 负责跟AmS通信一样，ViewRootImpl 的一个重要职责就是跟 WmS 通信，它通静态变量 sWindowSession（IWindowSession实例）与 WmS 进行通信。每个应用进程，仅有一个 sWindowSession 对象，它对应了 WmS 中的 Session 子类，WmS 为每一个应用进程分配一个 Session 对象。WindowState 类有一个 IWindow mClient 参数，是在构造方法中赋值的，是由 Session 调用 addWindow 传递过来了，对应了 ViewRootImpl 中的 W 类的实例。 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.addwindow/N4-Android-addWindow-ViewRootImpl-WMS.png)

#### 2.7、AttachInfo构造过程

```java
    AttachInfo(IWindowSession session, IWindow window, Display display,
        ViewRootImpl viewRootImpl, Handler handler, Callbacks effectPlayer) {
    mSession = session;//IWindowSession代理对象，用于与WMS通信
    mWindow = window;//W对象  
    mWindowToken = window.asBinder();//W本地Binder对象
    mDisplay = display;
    mViewRootImpl = viewRootImpl;//ViewRootImpl实例  
    mHandler = handler;//ViewRootHandler对象  
    mRootCallbacks = effectPlayer;
}
```

#### 2.8、创建Choreographer对象

[->Choreographer.java]

```java
        // Thread local storage for the choreographer.
    private static final ThreadLocal<Choreographer> sThreadInstance =
            new ThreadLocal<Choreographer>() {
        @Override
        protected Choreographer initialValue() {
            Looper looper = Looper.myLooper();
            if (looper == null) {
                throw new IllegalStateException("The current thread must have a looper!");
            }
            return new Choreographer(looper);
        }
    };
    public static Choreographer getInstance() {
        return sThreadInstance.get();
    }
```

为调用线程创建一个Choreographer实例，调用线程必须具备消息循环功能，因为ViewRootImpl对象的构造是在应用程序进程的UI主线程中执行的，因此创建的Choreographer对象将使用UI线程消息队列。 [->Choreographer.java]

```java
    private Choreographer(Looper looper) {
    mLooper = looper;
    //创建消息处理Handler
    mHandler = new FrameHandler(looper);
    //如果系统使用了Vsync机制，则注册一个FrameDisplayEventReceiver接收器
    mDisplayEventReceiver = USE_VSYNC ? new FrameDisplayEventReceiver(looper) : null;
    mLastFrameTimeNanos = Long.MIN_VALUE;
    //屏幕刷新周期
    mFrameIntervalNanos = (long)(1000000000 / getRefreshRate());
    //创建回调数组  
    mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];
    //初始化数组
    for (int i = 0; i <= CALLBACK_LAST; i++) {
        mCallbackQueues[i] = new CallbackQueue();
    }
}
```

FrameDisplayEventReceiver详细过程以后再Android 7.1.2(Android N) Choreographer机制实现过程分析。

#### 2.9、视图View添加过程

窗口管理器WindowManagerImpl为当前添加的窗口创建好各种对象后，调用ViewRootImpl的setView函数向WMS服务添加一个窗口对象。 [->ViewRootImpl.java]

```java
    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
        if (mView == null) {
            ////将DecorView保存到ViewRootImpl的成员变量mView中
            mView = view;

            ......

            mSoftInputMode = attrs.softInputMode;
            mWindowAttributesChanged = true;
            mWindowAttributesChangesFlag = WindowManager.LayoutParams.EVERYTHING_CHANGED;
            //同时将DecorView保存到mAttachInfo中  
            mAttachInfo.mRootView = view;
            mAttachInfo.mScalingRequired = mTranslator != null;
            mAttachInfo.mApplicationScale =
                    mTranslator == null ? 1.0f : mTranslator.applicationScale;
            if (panelParentView != null) {
                mAttachInfo.mPanelParentWindowToken
                        = panelParentView.getApplicationWindowToken();
            }
            mAdded = true;
            int res; /* = WindowManagerImpl.ADD_OKAY; */

            //1）在添加窗口前进行UI布局  
            requestLayout();
            if ((mWindowAttributes.inputFeatures
                    & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
                mInputChannel = new InputChannel();
            }
            mForceDecorViewVisibility = (mWindowAttributes.privateFlags
                    & PRIVATE_FLAG_FORCE_DECOR_VIEW_VISIBILITY) != 0;
            try {
                mOrigWindowType = mWindowAttributes.type;
                mAttachInfo.mRecomputeGlobalAttributes = true;
                collectViewAttributes();
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

ViewRootImpl的setView函数向WMS服务添加一个窗口对象过程：

(1) requestLayout()在应用程序进程中进行窗口UI布局；

(2) WindowSession.addToDisplay()向WMS服务注册一个窗口对象；

(3) 注册应用程序进程端的消息接收通道；

### (1)、requestLayout()在应用程序进程中进行窗口UI布局；

#### 2.10、窗口UI布局过程

requestLayout函数调用里面使用了Hanlder的一个小手段，那就是利用postSyncBarrier添加了一个Barrier（挡板），这个挡板的作用是阻塞普通的同步消息的执行，在挡板被撤销之前，只会执行异步消息，而requestLayout先添加了一个挡板Barrier，之后自己插入了一个异步任务mTraversalRunnable，其主要作用就是保证mTraversalRunnable在所有同步Message之前被执行，保证View绘制的最高优先级。具体实现如下：

```java
@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}
    void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        if (!mUnbufferedInputDispatch) {
            scheduleConsumeBatchedInput();
        }
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}
```

#### 2.10.1、添加回调过程

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

#### 2.10.2、Vsync请求过程

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

这里又通过IDisplayEventConnection接口来请求Vsync信号，IDisplayEventConnection实现了Binder通信框架，可以跨进程调用，因为Vsync信号请求进程和Vsync产生进程有可能不在同一个进程空间，因此这里就借助IDisplayEventConnection接口来实现。下面通过图来梳理Vsync请求的调用流程： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.addwindow/N5-Android-addWindow-requestNextVsync.png)

需要说明的是/_Vsync_/之间的代码此时其实还未执行，call requestNextVsync()来告诉系统我要在下一个VSYNC需要被trigger.

继续ViewRootImpl的setView函数中的WindowSession.addToDisplay()。

#### (2) 、mWindowSession.addToDisplay()向WMS服务注册一个窗口对象；

[Session.java]

```java
@Override
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
    boolean reportNewConfig = false;
    WindowState attachedWindow = null;
    long origId;
    final int callingUid = Binder.getCallingUid();
    final int type = attrs.type;
    synchronized(mWindowMap) {
        ......
        final DisplayContent displayContent = getDisplayContentLocked(displayId);
        ......
        boolean addToken = false;
        WindowToken token = mTokenMap.get(attrs.token);
        AppWindowToken atoken = null;
        boolean addToastWindowRequiresToken = false;
        ......
        WindowState win = new WindowState(this, session, client, token,
                attachedWindow, appOp[0], seq, attrs, viewVisibility, displayContent);

            return WindowManagerGlobal.ADD_APP_EXITING;
        }
        ......
        mPolicy.adjustWindowParamsLw(win.mAttrs);
        win.setShowToOwnerOnlyLocked(mPolicy.checkShowToOwnerOnly(attrs));
        res = mPolicy.prepareAddWindowLw(win, attrs);
        ......
        final boolean openInputChannels = (outInputChannel != null
                && (attrs.inputFeatures & INPUT_FEATURE_NO_INPUT_CHANNEL) == 0);
        if  (openInputChannels) {
            win.openInputChannel(outInputChannel);
        }
        ......
        if (addToken) {
            mTokenMap.put(attrs.token, token);
        }
        win.attach();
        mWindowMap.put(client.asBinder(), win);
        ......
        }
        boolean imMayMove = true;
        if (type == TYPE_INPUT_METHOD) {
            win.mGivenInsetsPending = true;
            mInputMethodWindow = win;
            addInputMethodWindowToListLocked(win);
            imMayMove = false;
        } else if (type == TYPE_INPUT_METHOD_DIALOG) {
            mInputMethodDialogs.add(win);
            addWindowToListInOrderLocked(win, true);
            moveInputMethodDialogsLocked(findDesiredInputMethodWindowIndexLocked(true));
            imMayMove = false;
        } else {
            addWindowToListInOrderLocked(win, true);
            if (type == TYPE_WALLPAPER) {
                mWallpaperControllerLocked.clearLastWallpaperTimeoutTime();
                displayContent.pendingLayoutChanges |= FINISH_LAYOUT_REDO_WALLPAPER;
            } else if ((attrs.flags&FLAG_SHOW_WALLPAPER) != 0) {
                displayContent.pendingLayoutChanges |= FINISH_LAYOUT_REDO_WALLPAPER;
            } else if (mWallpaperControllerLocked.isBelowWallpaperTarget(win)) {
                displayContent.pendingLayoutChanges |= FINISH_LAYOUT_REDO_WALLPAPER;
            }
        }
        ......
        mInputMonitor.setUpdateInputWindowsNeededLw();

        boolean focusChanged = false;
        if (win.canReceiveKeys()) {
            focusChanged = updateFocusedWindowLocked(UPDATE_FOCUS_WILL_ASSIGN_LAYERS,
                    false /*updateInputWindows*/);
            if (focusChanged) {
                imMayMove = false;
            }
        }
        ......
        mLayersController.assignLayersLocked(displayContent.getWindowList());
        // Don't do layout here, the window must call
        // relayout to be displayed, so we'll do it there.
        if (focusChanged) {
            mInputMonitor.setInputFocusLw(mCurrentFocus, false /*updateInputWindows*/);
        }
        mInputMonitor.updateInputWindowsLw(false /*force*/);
        ......
    }
    if (reportNewConfig) {
        sendNewConfiguration();
    }
    return res;
}
```

我们知道当应用程序进程添加一个DecorView到窗口管理器时，会为当前添加的窗口创建ViewRootImpl对象，同时构造了一个W本地Binder对象，无论是窗口视图对象DecorView还是ViewRootImpl对象，都只是存在于应用程序进程中，在添加窗口过程中仅仅将该窗口的W对象传递给WMS服务，经过Binder传输后，到达WMS服务端进程后变为IWindow.Proxy代理对象，因此该函数的参数client的类型为IWindow.Proxy。参数attrs的类型为WindowManager.LayoutParams，在应用程序进程启动Activity时，handleResumeActivity()函数通过WindowManager.LayoutParams l = r.window.getAttributes();来得到应用程序窗口布局参数，由于WindowManager.LayoutParams实现了Parcelable接口，因此WindowManager.LayoutParams对象可以跨进程传输，WMS服务的addWindow函数中的attrs参数就是应用程序进程发送过来的窗口布局参数。在WindowManagerImpl的addView函数中为窗口布局参数设置了相应的token，如果是应用程序窗口，则布局参数的token设为W本地Binder对象。如果不是应用程序窗口，同时当前窗口没有父窗口，则设置token为当前窗口的IApplicationToken.Proxy代理对象，否则设置为父窗口的IApplicationToken.Proxy代理对象，由于应用程序和WMS分属于两个不同的进程空间，因此经过Binder传输后，布局参数的令牌attrs.token就转变为IWindow.Proxy或者Token。以上函数首先根据布局参数的token等信息构造一个WindowToken对象，然后在构造一个WindowState对象，并将添加的窗口信息记录到mTokenMap和mWindowMap哈希表中。

在WMS服务端创建了所需对象后，接着调用了WindowState的attach()来进一步完成窗口添加。 [WindowState.java]

```java
void attach() {
    if (WindowManagerService.localLOGV) Slog.v(
        TAG, "Attaching " + this + " token=" + mToken
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
        if (SHOW_TRANSACTIONS) Slog.i(
                TAG_WM, "  NEW SURFACE SESSION " + mSurfaceSession);
        mService.mSessions.add(this);
        if (mLastReportedAnimatorScale != mService.getCurrentAnimatorScale()) {
            mService.dispatchNewAnimatorScaleLocked(this);
        }
    }
    mNumWindow++;
}
```

##### SurfaceSession建立过程

SurfaceSession对象承担了应用程序与SurfaceFlinger之间的通信过程，每一个需要与SurfaceFlinger进程交互的应用程序端都需要创建一个SurfaceSession对象。

客户端请求 [SurfaceSession.java]

```java
public SurfaceSession() {
    mNativeClient = nativeCreate();
}
```

Java层的SurfaceSession对象构造过程会通过JNI在native层创建一个SurfaceComposerClient对象。

[android_view_SurfaceSession.cpp]

```java
static jlong nativeCreate(JNIEnv* env, jclass clazz) {
SurfaceComposerClient* client = new SurfaceComposerClient();
client->incStrong((void*)nativeCreate);
return reinterpret_cast<jlong>(client);
}
```

Java层的SurfaceSession对象与C++层的SurfaceComposerClient对象之间是一对一关系。 [SurfaceComposerClient.cpp]

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

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.addwindow/N6-Android-addWindow-SF-Client.png)

/**********_**_**********_**_**********_**_**********Vsync**********_**_**********_**_**********_**_**********/

### Vsync信号处理

以上是请求过程，FrameDisplayEventReceiver对象用于请求并接收Vsync信号，当Vsync信号到来时，系统会自动调用其onVsync()函数，后面会回调到FrameDisplayEventReceiver.run方法（Why "同步分割栏"？[再分析](http://xfenglin.com/a/12007076930.html)），再回调函数中执行doFrame()实现屏幕刷新。 当VSYNC信号到达时，Choreographer doFrame()函数被调用 [->Choreographer.java]

```java
void doFrame(long frameTimeNanos, int frame) {
    final long startNanos;
    ......

        long intendedFrameTimeNanos = frameTimeNanos;
        //保存起始时间
        startNanos = System.nanoTime();
        //由于Vsync事件处理采用的是异步方式，因此这里计算消息发送与函数调用开始之间所花费的时间
        final long jitterNanos = startNanos - frameTimeNanos;
        //计算函数调用期间所错过的帧数  
        if (jitterNanos >= mFrameIntervalNanos) {
            final long skippedFrames = jitterNanos / mFrameIntervalNanos;
            if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
                Log.i(TAG, "Skipped " + skippedFrames + " frames!  "
                        + "The application may be doing too much work on its main thread.");
            }
            final long lastFrameOffset = jitterNanos % mFrameIntervalNanos;
            frameTimeNanos = startNanos - lastFrameOffset;
        }
        //如果frameTimeNanos小于一个屏幕刷新周期，则重新请求VSync信号
        if (frameTimeNanos < mLastFrameTimeNanos) {
            scheduleVsyncLocked();
            return;
        }

        mFrameInfo.setVsync(intendedFrameTimeNanos, frameTimeNanos);
        mFrameScheduled = false;
        mLastFrameTimeNanos = frameTimeNanos;
    }

    try {
       //分别回调CALLBACK_INPUT、CALLBACK_ANIMATION、CALLBACK_TRAVERSAL、CALLBACK_COMMIT事件  
        doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);
        doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);
        doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);
        doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
    }
}
```

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.addwindow/N7-Android-addWindow-Vsync-time.png)
Choreographer类中分别定义了CallbackRecord、CallbackQueue内部类，CallbackQueue是一个按时间先后顺序保存CallbackRecord的单向循环链表。 

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.addwindow/N8-Android-addWindow-CallBackRecord.png)
在Choreographer中定义了三个CallbackQueue队列，用数组mCallbackQueues表示，用于分别保存CALLBACK_INPUT、CALLBACK_ANIMATION、CALLBACK_TRAVERSAL这三种类型的Callback，当调用Choreographer类的postCallback()函数时，就是往指定类型的CallbackQueue队列中通过addCallbackLocked()函数添加一个CallbackRecord项：首先构造一个CallbackRecord对象，然后按时间先后顺序插入到CallbackQueue链表中。从代码注释中，我们可以知道CALLBACK_INPUT是指输入回调，该回调优先级最高，首先得到执行，而CALLBACK_TRAVERSAL是指处理布局和绘图的回调，只有在所有异步消息都执行完后才得到执行，CALLBACK_ANIMATION是指动画回调，比CALLBACK_TRAVERSAL优先执行，从doFrame()函数中的doCallbacks调用就能印证这点。 当Vsync事件到来时，顺序执行CALLBACK_INPUT、CALLBACK_ANIMATION、CALLBACK_TRAVERSAL 和CALLBACK_COMMIT 对应CallbackQueue队列中注册的回调。 [->Choreographer.java]

```java
    void doCallbacks(int callbackType, long frameTimeNanos) {
    CallbackRecord callbacks;
    synchronized (mLock) {
        final long now = System.nanoTime();
        //从指定类型的CallbackQueue队列中查找执行时间到的CallbackRecord
        callbacks = mCallbackQueues[callbackType].extractDueCallbacksLocked(
                now / TimeUtils.NANOS_PER_MS);
        ......

        if (callbackType == Choreographer.CALLBACK_COMMIT) {
            final long jitterNanos = now - frameTimeNanos;
            if (jitterNanos >= 2 * mFrameIntervalNanos) {
                final long lastFrameOffset = jitterNanos % mFrameIntervalNanos
                        + mFrameIntervalNanos;
                    mDebugPrintNextFrameTimeDelta = true;
                }
                frameTimeNanos = now - lastFrameOffset;
                mLastFrameTimeNanos = frameTimeNanos;
            }
        }
    }
    try {
        for (CallbackRecord c = callbacks; c != null; c = c.next) {
          //由于CallbackQueues是按时间先后顺序排序的，因此遍历执行所有时间到的CallbackRecord  
            c.run(frameTimeNanos);
        }
    } finally {
        synchronized (mLock) {
            mCallbacksRunning = false;
            do {
                final CallbackRecord next = callbacks.next;
                recycleCallbackLocked(callbacks);
                callbacks = next;
            } while (callbacks != null);
        }
    }
}
```

我们知道Choreographer对外提供了两个接口函数用于注册指定的Callback，postCallback()用于注册Runnable对象，而postFrameCallback()函数用于注册FrameCallback对象，无论注册的是Runnable对象还是FrameCallback对象，在CallbackRecord对象中统一装箱为Object类型。在执行其回调函数时，就需要区别这两种对象类型，如果注册的是Runnable对象，则调用其run()函数，如果注册的是FrameCallback对象，则调用它的doFrame()函数。 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.addwindow/N9-Android-addWindow-Choreographer.png)

#### 2.11、视图View添加过程

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.addwindow/N10-Android-addWindow-AT-WMS.png)

关于Choreographer的postCallback()用法在前面进行了详细的介绍，当Vsync事件到来时，mTraversalRunnable对象的run()函数将被调用。

mTraversalRunnable对象的类型为TraversalRunnable，该类实现了Runnable接口，在其run()函数中调用了doTraversal()函数来完成窗口布局。 [->ViewRootImpl.java]

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

        if (mProfile) {
            Debug.startMethodTracing("ViewAncestor");
        }

        performTraversals();

        if (mProfile) {
            Debug.stopMethodTracing();
            mProfile = false;
        }
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
    final View host = mView;
    mIsInTraversal = true;
    mWillDrawSoon = true;
    boolean windowSizeMayChange = false;
    boolean newSurface = false;
    boolean surfaceChanged = false;
    WindowManager.LayoutParams lp = mWindowAttributes;

    int desiredWindowWidth;
    int desiredWindowHeight;

    final int viewVisibility = getHostVisibility();
    final boolean viewVisibilityChanged = !mFirst
            && (mViewVisibility != viewVisibility || mNewSurfaceNeeded);
    final boolean viewUserVisibilityChanged = !mFirst &&
            ((mViewVisibility == View.VISIBLE) != (viewVisibility == View.VISIBLE));

    WindowManager.LayoutParams params = null;
    if (mWindowAttributesChanged) {
        mWindowAttributesChanged = false;
        surfaceChanged = true;
        params = lp;
    }
    ......
    /****************执行窗口测量******************/  
    boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);
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
        } catch (RemoteException e) {
        }
        ......
        if (!mStopped || mReportNextDraw) {
            ......
                 // Ask host how big it wants to be
                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                ......
            }
        }
    } else {
        ......
    }
    /****************执行窗口布局******************/
    final boolean didLayout = layoutRequested && (!mStopped || mReportNextDraw);
    boolean triggerGlobalLayoutListener = didLayout
            || mAttachInfo.mRecomputeGlobalAttributes;
    if (didLayout) {
        performLayout(lp, mWidth, mHeight);
        ......
        }
    /****************查找窗口焦点******************/
    if (mFirst) {
    ......
        if (mView != null) {
            if (!mView.hasFocus()) {
                mView.requestFocus(View.FOCUS_FORWARD);
            } else {}
        }
    }

    final boolean changedVisibility = (viewVisibilityChanged || mFirst) && isViewVisible;
    final boolean hasWindowFocus = mAttachInfo.mHasWindowFocus && isViewVisible;
    final boolean regainedFocus = hasWindowFocus && mLostWindowFocus;
    if (regainedFocus) {
        mLostWindowFocus = false;
    } else if (!hasWindowFocus && mHadWindowFocus) {
        mLostWindowFocus = true;
    }

    if (changedVisibility || regainedFocus) {
        // Toasts are presented as notifications - don't present them as windows as well
        boolean isToast = (mWindowAttributes == null) ? false
                : (mWindowAttributes.type == WindowManager.LayoutParams.TYPE_TOAST);
        if (!isToast) {
            host.sendAccessibilityEvent(AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED);
        }
    }

    mFirst = false;
    mWillDrawSoon = false;
    mNewSurfaceNeeded = false;
    mActivityRelaunched = false;
    mViewVisibility = viewVisibility;
    mHadWindowFocus = hasWindowFocus;

    if (hasWindowFocus && !isInLocalFocusMode()) {
        final boolean imTarget = WindowManager.LayoutParams
                .mayUseInputMethod(mWindowAttributes.flags);
        if (imTarget != mLastWasImTarget) {
            mLastWasImTarget = imTarget;
            InputMethodManager imm = InputMethodManager.peekInstance();
            if (imm != null && imTarget) {
                imm.onPreWindowFocus(mView, hasWindowFocus);
                imm.onPostWindowFocus(mView, mView.findFocus(),
                        mWindowAttributes.softInputMode,
                        !mHasHadWindowFocus, mWindowAttributes.flags);
            }
        }
    }

    // Remember if we must report the next draw.
    if ((relayoutResult & WindowManagerGlobal.RELAYOUT_RES_FIRST_TIME) != 0) {
        mReportNextDraw = true;
    }
    /****************执行窗口绘制******************/  
    boolean cancelDraw = mAttachInfo.mTreeObserver.dispatchOnPreDraw() || !isViewVisible;

    if (!cancelDraw && !newSurface) {
        if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
            for (int i = 0; i < mPendingTransitions.size(); ++i) {
                mPendingTransitions.get(i).startChangingAnimations();
            }
            mPendingTransitions.clear();
        }

        performDraw();
    } else {
        if (isViewVisible) {
            // Try again
            scheduleTraversals();
        } else if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
            for (int i = 0; i < mPendingTransitions.size(); ++i) {
                mPendingTransitions.get(i).endChangingAnimations();
            }
            mPendingTransitions.clear();
        }
    }

    mIsInTraversal = false;
}
```

#### 1、执行窗口测量performMeasure()

[->ViewRootImpl.java]

```java
    private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
    try {
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

#### 2、执行窗口注册relayoutWindow；

[->ViewRootImpl.java]

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

该Surface构造函数仅仅创建了一个空Surface对象，并没有对该Surface进程native层的初始化，到此我们知道应用程序进程为每个窗口对象都创建了一个Surface对象。并且将该Surface通过跨进程方式传输给WMS服务进程，我们知道，在Android系统中，如果一个对象需要在不同进程间传输，必须实现Parcelable接口，Surface类正好实现了Parcelable接口。ViewRootImpl通过IWindowSession接口请求WMS的完整过程如下：

[->IWindowSession.java$ Proxy]

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

该函数可以看出，WMS服务在响应应用程序进程请求添加窗口时，首先在当前进程空间创建一个Surface对象，然后调用Session的relayout()函数进一步完成窗口添加过程，最后将WMS服务中创建的Surface返回给应用程序进程。

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.addwindow/N11-Android-addWindow-App-WMS-Surface.png)

到目前为止，在应用程序进程和WMS服务进程分别创建了一个Surface对象，但是他们调用的都是Surface的无参构造函数，在该构造函数中并未真正初始化native层的Surface，那native层的Surface是在那里创建的呢？

[->Session.java]

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

#### 2.12、Surface创建过程

[->SurfaceControl.java]

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

该函数首先得到前面创建好的SurfaceComposerClient对象，通过该对象向SurfaceFlinger端的Client对象发送创建Surface的请求，最后得到一个SurfaceControl对象。 [->SurfaceComposerClient.cpp]

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

SurfaceFlinger根据标志位创建对应类型的Surface，当前系统定义了2种类型的Layer: [->ISurfaceComposerClient.h]

```c
eFXSurfaceNormal    = 0x00000000,
eFXSurfaceDim       = 0x00020000,
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
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.addwindow/N12-Android-addWindow-createSurface.png)

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

#### BufferQueue构造过程

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

[->BufferQueueCore.cpp] 所以核心都是这个BufferQueueCore，他是管理图形缓冲区的中枢。这里举一个SurfaceTexture的例子，来看看他们之间的关系： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.addwindow/N13-Android-addWindow-BufferQueue.jpg)

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
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.addwindow/N14-Android-addWindow-slot.jpg)

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

#### GraphicBufferAlloc构造过程

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

#### 图形缓冲区创建过程

[->GraphicBuffer.cpp]

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

使用GraphicBufferAllocator对象来为图形缓冲区分配内存空间，GraphicBufferAllocator是对Gralloc模块中的gpu设备的封装类。关于GraphicBufferAllocator内存分配过程请查看[Android图形缓冲区分配过程源码分析](http://blog.csdn.net/yangwen123/article/details/12231687)图形缓冲区分配完成后，还会映射到SurfaceFlinger服务进程的虚拟地址空间。

#### Android图形缓冲区分配过程源码分析

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
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.addwindow/N15-Android-addWindow-createSurface-Layer.png)

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
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.addwindow/N16-Android-addWindow-surface-create-client.png)

#### 应用程序本地窗口Surface创建过程

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
mTimestamp = NATIVE_WINDOW_TIMESTAMP_AUTO;
mDataSpace = HAL_DATASPACE_UNKNOWN;
mScalingMode = NATIVE_WINDOW_SCALING_MODE_FREEZE;
mTransform = 0;
mStickyTransform = 0;
mDefaultWidth = 0;
mDefaultHeight = 0;
mUserWidth = 0;
mUserHeight = 0;
mTransformHint = 0;
mConsumerRunningBehind = false;
mConnectedToCpu = false;
mProducerControlledByApp = controlledByApp;
mSwapIntervalZero = false;
}
```

在创建完应用程序本地窗口Surface后，想要在该Surface上绘图，首先需要为该Surface分配图形buffer。我们前面介绍了Android应用程序图形缓冲区的分配都是由SurfaceFlinger服务进程来完成，在请求创建Surface时，在服务端创建了一个BufferQueue本地Binder对象，该对象负责管理应用程序一个本地窗口Surface的图形缓冲区。

##### 3、执行窗口布局performLayout()

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
            // requestLayout() was called during layout.
            // If no layout-request flags are set on the requesting views, there is no problem.
            // If some requests are still pending, then we need to clear those flags and do
            // a full request/measure/layout pass to handle this situation.
            ArrayList<View> validLayoutRequesters = getValidLayoutRequesters(mLayoutRequesters,
                    false);
            if (validLayoutRequesters != null) {
                // Set this flag to indicate that any further requests are happening during
                // the second pass, which may result in posting those requests to the next
                // frame instead
                mHandlingLayoutInLayoutRequest = true;

                // Process fresh layout requests, then measure and layout
                int numValidRequests = validLayoutRequesters.size();
                for (int i = 0; i < numValidRequests; ++i) {
                    final View view = validLayoutRequesters.get(i);
                    Log.w("View", "requestLayout() improperly called by " + view +
                            " during layout: running second layout pass");
                    view.requestLayout();
                }
                measureHierarchy(host, lp, mView.getContext().getResources(),
                        desiredWindowWidth, desiredWindowHeight);
                mInLayout = true;
                host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());

                mHandlingLayoutInLayoutRequest = false;

                // Check the valid requests again, this time without checking/clearing the
                // layout flags, since requests happening during the second pass get noop'd
                validLayoutRequesters = getValidLayoutRequesters(mLayoutRequesters, true);
                if (validLayoutRequesters != null) {
                    final ArrayList<View> finalRequesters = validLayoutRequesters;
                    // Post second-pass requests to the next frame
                    getRunQueue().post(new Runnable() {
                        @Override
                        public void run() {
                            int numValidRequests = finalRequesters.size();
                            for (int i = 0; i < numValidRequests; ++i) {
                                final View view = finalRequesters.get(i);
                                view.requestLayout();
                            }
                        } });} } }
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
    mInLayout = false;
}
```

##### 4.执行窗口绘制performDraw()

```java
  [->ViewRootImpl.java]

        private void performDraw() {
        ......
        try {
            draw(fullRedrawNeeded);
        } finally {
            mIsDrawing = false;
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
        ......
        }
    }
```

Android是怎样将View画出来的？ [->ViewRootImpl.java]

```java
    private void draw(boolean fullRedrawNeeded) {
    Surface surface = mSurface;
    ......

    mAttachInfo.mTreeObserver.dispatchOnDraw();

    int xOffset = -mCanvasOffsetX;
    int yOffset = -mCanvasOffsetY + curScrollY;
    final WindowManager.LayoutParams params = mWindowAttributes;
    final Rect surfaceInsets = params != null ? params.surfaceInsets : null;
    if (surfaceInsets != null) {
        xOffset -= surfaceInsets.left;
        yOffset -= surfaceInsets.top;

        // Offset dirty rect for surface insets.
        dirty.offset(surfaceInsets.left, surfaceInsets.right);
    }

    boolean accessibilityFocusDirty = false;
    final Drawable drawable = mAttachInfo.mAccessibilityFocusDrawable;
    if (drawable != null) {
        final Rect bounds = mAttachInfo.mTmpInvalRect;
        final boolean hasFocus = getAccessibilityFocusedRect(bounds);
        if (!hasFocus) {
            bounds.setEmpty();
        }
        if (!bounds.equals(drawable.getBounds())) {
            accessibilityFocusDirty = true;
        }
    }

    mAttachInfo.mDrawingTime =
            mChoreographer.getFrameTimeNanos() / TimeUtils.NANOS_PER_MS;

    if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
        if (mAttachInfo.mHardwareRenderer != null && mAttachInfo.mHardwareRenderer.isEnabled()) {
            // If accessibility focus moved, always invalidate the root.
            boolean invalidateRoot = accessibilityFocusDirty || mInvalidateRootRequested;
            mInvalidateRootRequested = false;

            // Draw with hardware renderer.
            mIsAnimating = false;

            if (mHardwareYOffset != yOffset || mHardwareXOffset != xOffset) {
                mHardwareYOffset = yOffset;
                mHardwareXOffset = xOffset;
                invalidateRoot = true;
            }

            if (invalidateRoot) {
                mAttachInfo.mHardwareRenderer.invalidateRoot();
            }
            ......
            if (updated) {
                requestDrawWindow();
            }
            mAttachInfo.mHardwareRenderer.draw(mView, mAttachInfo, this);
        } else {
            if (mAttachInfo.mHardwareRenderer != null &&
                    !mAttachInfo.mHardwareRenderer.isEnabled() &&
                    mAttachInfo.mHardwareRenderer.isRequested()) {
                try {
                    mAttachInfo.mHardwareRenderer.initializeIfNeeded(
                            mWidth, mHeight, mAttachInfo, mSurface, surfaceInsets);
                } catch (OutOfResourcesException e) {
                    handleOutOfResourcesException(e);
                    return;
                }
                mFullRedrawNeeded = true;
                scheduleTraversals();
                return;
            }

            if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset, scalingRequired, dirty)) {
                return;
            }
        }
    }

    if (animating) {
        mFullRedrawNeeded = true;
        scheduleTraversals();
    }
}
```

关于绘制这个流程很复杂，我们后续章节再分析。 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.addwindow/N17-Android-addWindow-Render.JPG)

参考博客：[Android应用程序UI硬件加速渲染技术简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/45601143)

这里我们因为要分析Surface机制，所以只分析ViewRootImpl的draw流程。（如果开启了硬件加速功能，则会使用hwui硬件绘制功能，这里我们忽略这个，使用默认的软件绘制流程drawSoftware）。 [->ViewRootImpl.java]

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

```c
这段代码逻辑主要如下：
   1）获取java层dirty 的Rect大小和位置信息；
   2）调用Surface的lock方法,将申请的图形缓冲区赋给outBuffer；
   3）创建一个Skbitmap，填充它用来保存申请的图形缓冲区，并赋值给Java层的Canvas对象；
   4）将剪裁位置大小信息赋给java层Canvas对象。
```

#### unlockCanvasAndPost()

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

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.addwindow/N18-Android-addWindow-LockUnlock-Surfacecycle.jpg)

#### Surface管理图形缓冲区

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

#### 图形缓冲区入队

我们前面讲了，省略了第二步绘制流程，因此我们这里分析第三部，绘制完毕后再queueBuffer。 同样，调用了Surface的unlockCanvasAndPost函数，我们查看它的实现： [->Surface.cpp]

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

总结： 1）从传入的QueueBufferInput ，解析填充一些变量； 2）改变入队Slot的状态为QUEUED，每次推进来，mFrameCounter都加1。这里的slot，上一篇讲分配缓冲区返回最老的FREE状态buffer，就是用这个mFrameCounter最小值判断，就是上一篇LRU算法的判断； 3）创建一个BufferItem来描述GraphicBuffer，用mSlots[slot]中的slot填充BufferItem； 4）将BufferItem塞进mCore的mQueue队列，依照指定规则； 5）然后通知SurfaceFlinger去消费。

上述lockCanvas和unlockCanvasAndPost可以用下图来总结一下： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.addwindow/N19-Android-addWindow-LockUnlock.jpg)

### 通知SF消费合成

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.addwindow/N20-Android-addWindow-SF-Composition.png)

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
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.addwindow/N21-Android-addWindow-vsync-surfaceflinger.png)

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

#### 一、preComposition()函数

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

#### 1.1、每个Layer的onFrameAvailable函数

onPreComposition函数来根据mQueuedFrames来判断图像是否发生了变化，或者是mSidebandStreamChanged、mAutoRefresh。 [Layer.cpp]

```c
bool Layer::onPreComposition() {
mRefreshPending = false;
return mQueuedFrames > 0 || mSidebandStreamChanged || mAutoRefresh;
}
```

当Layer所对应的Surface更新图像后，它所对应的Layer对象的onFrameAvailable函数会被调用来通知这种变化。 在SurfaceFlinger的preComposition函数中当有Layer的图像改变了，最后也会调用SurfaceFlinger的signalLayerUpdate函数。 SurfaceFlinger::signalLayerUpdate是调用了MessageQueue的invalidate函数 最后处理还是调用了SurfaceFlinger的onMessageReceived函数。看看SurfaceFlinger的onMessageReceived函数对NVALIDATE的处理 handleMessageInvalidate函数中调用了handlePageFlip函数，这个函数将会处理Layer中的缓冲区，把更新过的图像缓冲区切换到前台，等待VSync信号更新到FrameBuffer。

#### 1.2、绘制流程

具体完整的绘制流程如图。 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.addwindow/N22-Android-addWindow-SF-Composition.jpg)

##### 二、handleTransaction handPageFlip更新Layer对象

在上一节中的绘图的流程中，我们看到了handleTransaction和handPageFlip这两个函数通常是在用户进程更新Surface图像时会调用，来更新Layer对象。这节就主要讲解这两个函数。

#### 2.1、handleTransaction函数

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

#### 2.2、Layer的doTransaction函数

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

##### 2.3、handleTransactionLocked函数

下面我们来分析handleTransactionLocked函数，这个函数比较长，我们分段分析

###### 2.3.1 处理Layer的事务 [SurfaceFlinger.cpp]

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

##### 2.3.2、处理显示设备的变化

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

##### 2.3.3、设置TransfromHit

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

##### 2.3.4、处理Layer增加情况

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

##### 2.3.5、设置mDrawingState

```c
    commitTransaction();
updateCursorAsync();
}
```

调用commitTransaction和updateCursorAsync函数 commitTransaction函数作用是把mDrawingState的值设置成mCurrentState的值。而updateCursorAsync函数会更新所有显示设备中光标的位置。

2.3.6 小结 handleTransaction函数的作用的就是处理系统在两次刷新期间的各种变化。SurfaceFlinger模块中不管是SurfaceFlinger类还是Layer类，都采用了双缓冲的方式来保存他们的属性，这样的好处是刚改变SurfaceFlinger对象或者Layer类对象的属性是，不需要上锁，大大的提高了系统效率。只有在最后的图像输出是，才进行一次上锁，并进行内存的属性变化处理。正因此，应用进程必须收到VSync信号才开始改变Surface的内容。

#### 2.4、handlePageFlip函数

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

##### 2.5 小结

这样经过handleTransaction handlePageFlip两个函数处理，SurfaceFlinger中无论是Layer属性的变化还是图像的变化都处理好了，只等VSync信号到来就可以输出了。

#### 三、rebuildLayerStacks函数

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

##### 四、setUpHWComposer函数

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

##### 五、合成所有层的图像 （doComposition()函数）

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

##### 六、postFramebuffer函数

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

/**********_**_**********_**_**********_**_**********Vsync**********_**_**********_**_**********_**_**********/

## 参考文档（特别感谢）：

[Android6.0 显示系统（六） 图像的输出过程 - kc58236582的博客 - CSDN博客](http://blog.csdn.net/kc58236582/article/details/52778333)
[startActivity启动过程分析](http://gityuan.com/2016/03/12/start-activity/)
[浅析Android的窗口 - DEV CLUB](https://dev.qq.com/topic/5923ef85bdc9739041a4a798)
[Android源码解析之（十四）-->Activity启动流程](http://blog.csdn.net/qq_23547831/article/details/51224992)
[Android应用setContentView与LayoutInflater加载解析机制源码分析](http://blog.csdn.net/yanbober/article/details/45970721)
[Android应用程序窗口设计框架介绍 - 深入剖析Android系统 - CSDN博客](http://blog.csdn.net/yangwen123/article/details/35987609)
[Android显示系统设计框架介绍 - 深入剖析Android系统 - CSDN博客](http://blog.csdn.net/yangwen123/article/details/22647255)
[图解Android - Android GUI 系统 (2) - 窗口管理 (View, Canvas, Window Manager) - 漫天尘沙 - 博客园](http://www.cnblogs.com/samchen2009/p/3367496.html)
[Android SurfaceFlinger 学习之路(五)----VSync 工作原理 | April is your lie](http://windrunnerlihuan.com/2017/05/25/Android-SurfaceFlinger-%E5%AD%A6%E4%B9%A0%E4%B9%8B%E8%B7%AF-%E4%BA%94-VSync-%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86/)
[Android graphics 学习－生产者、消费者、BufferQueue介绍 - armwind的专栏 - CSDN博客 Graphics DEMO Bingo](http://blog.csdn.net/armwind/article/details/73436532)
[Android 图形系统概述](https://toutiao.io/posts/n0wr4e/preview)
