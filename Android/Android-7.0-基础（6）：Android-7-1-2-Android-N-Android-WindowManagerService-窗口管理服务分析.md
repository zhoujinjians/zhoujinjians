---
title: Android N 基础（6）：Android 7.1.2 Android WindowManagerService 窗口管理服务分析
cover: https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/hexo.themes/bing-wallpaper-2018.04.08.jpg
categories: 
  - Android
tags:
  - Android
toc: true
abbrlink: 20180108
date: 2018-01-08 09:25:00
---
窗口管理系统WMS是Android中的主要子系统之一，它涉及到App中组件的管理，系统和应用窗口的管理和绘制等工作。由于其涉及模块众多，且与用户体验密切相关，所以它也是Android当中最为复杂的子系统之一。一个App从启动到主窗口显示出来，需要App，ActivityManagerService（AMS），WindowManagerService（WMS），SurfaceFlinger（SF）等几个模块相互合作。App负责业务逻辑，绘制自己的视图；AMS管理组件、进程信息和Activity的堆栈及状态等等；WMS管理Activity对应的窗口及子窗口，还有系统窗口等；SF用于管理图形缓冲区，将App绘制的东西合成渲染在屏幕上。

<!-- more -->



## 源码（部分）：

**/frameworks/base/services/core/java/com/android/server/am/**

- ActivityStack.java
- ActivityManagerService.java
- ActivityStackSupervisor.java
- ActivityStarter.java
- ActivityRecord.java

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

**/frameworks/base/services/core/java/com/android/server/wm/**

- WindowManagerService.java
- AppWindowAnimator.java
- AppTransition.java
- AppWindowToken.java
- Session.java
- WindowState.java
- WindowAnimator.java
- WindowStateAnimator.java
- WindowSurfacePlacer.java
- WindowSurfaceController.java


我们先看一下窗口启动、退出过程动态图，之后再详细分析： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/android.wms/01-Android-WMS-ezgif.com-gif-maker-WindowManagerService-resize.gif)


## （一）、Window 组织方式

ActivityManagerService（AMS），WindowManagerService（WMS），SurfaceFlinger（SF）等几个模块相互合作。App负责业务逻辑，绘制自己的视图；AMS管理组件、进程信息和Activity的堆栈及状态等等；WMS管理Activity对应的窗口及子窗口，还有系统窗口等；SF用于管理图形缓冲区，将App绘制的东西合成渲染在屏幕上。 窗口管理系统主要框架： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/android.wms/02-Android-WMS-AMS-SurfaceFlinger-Conn.png)

> 主要对象功能介绍：

> WindowManagerService负责完成窗口的管理工作

> WindowState和应用端窗口一一对应，应用调用WMS添加窗口时，最终会在WindowManagerService.addWindow()创建一个WindowState与之一一对应

> WindowToken是一个句柄，保存了所有具有同一个token的WindowState。应用请求WindowManagerService添加窗口的时候，提供了一个token，该token标识了被添加窗口的归属，WindowManagerService为该token生成一个WindowToken对象，所有token相同的WindowState被关联到同一个WindowToken，如输入法添加窗口时，会传递一个IBinder mCurToken，墙纸服务添加窗口时，会传递一个WallpaperConnection::final Binder mToken。

> AppWindowToken继承于WindowToken，专门用于标识一个Activity。AppWindowToken里的token实际上就是指向了一个Activity。ActivityManagerService通知应用启动的时候，在服务端生成一个token用于标识该Activity，并且把该token传递到应用客户端，客户端的Activity在申请添加窗口时，以该token作为标识传递到WindowManagerService。同一个Activity中的主窗口、对话框窗口、菜单窗口都关联到同一个AppWindowToken。

> Session表示一个客户端和服务端的交互会话。一般来说不同的应用通过不同的会话来和WindowManagerService交互，但是处于同一个进程的不同应用通过同一个Session来交互。

### 1.1、Android Token介绍

Token是ActivityRecord的内部静态类，我们先来看下Token的继承关系，Token extends IApplicationToken.Stub，从IApplicationToken.Stub类进行继承，根据Binder的机制可以知道Token是一个匿名Binder实体类，这个匿名Binder实体会传递给其他进程，其他进程会拿到Token的代理端。 我们知道匿名Binder有两个比较重要的用途，一个是拿到Binder代理端后可跨Binder调用实体端的函数接口，另一个作用便是在多个进程中标识同一个对象。往往这两个作用是同时存在的，比如我们这里研究的Token就同时存在这两个作用，但最重要的便是后者，Token标识了一个ActivityRecord对象，即间接标识了一个Activity。

Token梳理： 分析源码，我们发现，大多数 token 的对象，都表示一个 IBinder 对象。提到 IBinder，大家一点也不陌生，就是 Android 的 IPC 通信机制。在创建窗口过程中，涉及到的 IPC 通信，无非包含两方面，一个是 WmS 用来跟应用所在的进程进行通信的 ViewRootImpl.W 类的对象，另一个是指向一个 ActivityRecord 的对象，自然应该是WMS用来跟 AMS进行通信的了。我们梳理了一下，token 以下几处的定义，分别来讲讲这里的 token 代表什么。

![Markdown](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/android.wms/03-Android-WMS-Token-Detail.png)

分析一下 View 的 AttachInfo 的赋值。ViewRootImpl 在构建方法里，会初始化一个 AttachInfo 实例，把它的 Session，以及 W类对象赋值给 AttachInfo。分析可以看到，AttachInfo 中的 mWindowToken，与mWindow 都是指向 ViewRootImpl 中的 mWindow(W类实例)。当一个 View attach 到窗口后，ViewRootImpl会执行performTraversals，如果发现是首次调用会，会把自己的 mAttachInfo 传递给根 View（通过dispatchAttachedToWindow），告诉 View 树现在已经 attch to Window 了，马上可以显示了。根 View（一般是 ViewGroup）会把这个信息，遍历地传递给 View 树中的每一个子 View，这样每个 View 的 mAttachInfo 都被赋值为 ViewRootImp 的 mAttachInfo了。

### 1.1.1、Token对象的创建

下面这个图是Token的传递，首先会传递到WMS中，接着会传递到应用进程ActivityThread中，下面来具体分析这个传递流程。 总体流程图： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/android.wms/04-Android-WMS-Token.png)

我们之前分析：【Android 7.1.2 (Android N) Activity启动流程分析】 在启动Activity过程中会调用ActivityStarter.startActivityLocked()

```java
[->ActivityStarter.java]
    final int startActivityLocked(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
        String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
        String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
        ActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
        ActivityRecord[] outActivity, ActivityStackSupervisor.ActivityContainer container,
        TaskRecord inTask) {
    ......
    ActivityRecord sourceRecord = null;
    ActivityRecord resultRecord = null;
    ......
    ActivityRecord r = new ActivityRecord(mService, callerApp, callingUid, callingPackage,
            intent, resolvedType, aInfo, mService.mConfiguration, resultRecord, resultWho,
            requestCode, componentSpecified, voiceSession != null, mSupervisor, container,
            options, sourceRecord);
    ......
    try {
        err = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor, startFlags,
                true, options, inTask);
    }
    return err;
}
```

可以看到在startActivityLocked()中创建了一个ActivityRecord对象

```java
[->ActivityRecord.java]
final IApplicationToken.Stub appToken; // window manager token

ActivityRecord(ActivityManagerService _service, ProcessRecord _caller,
        int _launchedFromUid, String _launchedFromPackage, Intent _intent, String _resolvedType,
        ActivityInfo aInfo, Configuration _configuration,
        ActivityRecord _resultTo, String _resultWho, int _reqCode,
        boolean _componentSpecified, boolean _rootVoiceInteraction,
        ActivityStackSupervisor supervisor,
        ActivityContainer container, ActivityOptions options, ActivityRecord sourceRecord) {
    service = _service;
    appToken = new Token(this, service);
    ......
    }
```

在ActivityRecord的构造函数中创建，标识着当前这个ActivityRecord，即间接代表着一个Activity。

### 1.1.2、AMS调用WMS的addAPPToken()接口

在启动一个Activity时，会调用startActivityLocked()来在WMS中添加一个AppWindowToken对象 startActivityLocked()创建ActivityRecord对象后会继续调用startActivityUnchecked()方法。

```java
[->ActivityStarter.java]
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask) {
        ......
        mTargetStack.startActivityLocked(mStartActivity, newTask, mKeepCurTransition, mOptions);
        ......
        }
```

```java
[->ActivityStack.java]
final void startActivityLocked(ActivityRecord r, boolean newTask, boolean keepCurTransition,
            ActivityOptions options) {
            ......
            addConfigOverride(r, task);
            ......
     }

    void addConfigOverride(ActivityRecord r, TaskRecord task) {
        final Rect bounds = task.updateOverrideConfigurationFromLaunchBounds();
        // 跳转到WMS
        mWindowManager.addAppToken(task.mActivities.indexOf(r), r.appToken,
                r.task.taskId, mStackId, r.info.screenOrientation, r.fullscreen,
                (r.info.flags & FLAG_SHOW_FOR_ALL_USERS) != 0, r.userId, r.info.configChanges,
                task.voiceSession != null, r.mLaunchTaskBehind, bounds, task.mOverrideConfig,
                task.mResizeMode, r.isAlwaysFocusable(), task.isHomeTask(),
                r.appInfo.targetSdkVersion, r.mRotationAnimationHint);
        r.taskConfigOverride = task.mOverrideConfig;
    }
```

我们继续看下WindowManager.addAppToken()方法

```java
[->WindowManagerService.java]
    @Override
    public void addAppToken(int addPos, IApplicationToken token, int taskId, int stackId,
            int requestedOrientation, boolean fullscreen, boolean showForAllUsers, int userId,
            int configChanges, boolean voiceInteraction, boolean launchTaskBehind,
            Rect taskBounds, Configuration config, int taskResizeMode, boolean alwaysFocusable,
            boolean homeTask, int targetSdkVersion, int rotationAnimationHint) {
      ......

        synchronized(mWindowMap) {
            AppWindowToken atoken = findAppWindowToken(token.asBinder());
            if (atoken != null) {
                Slog.w(TAG_WM, "Attempted to add existing app token: " + token);
                return;
            }
            //根据ActivityRecord中IApplicationToken.Stub的代理，创建AppWindowToken
            atoken = new AppWindowToken(this, token, voiceInteraction);
            atoken.inputDispatchingTimeoutNanos = inputDispatchingTimeoutNanos;
            atoken.appFullscreen = fullscreen;
            atoken.showForAllUsers = showForAllUsers;
            atoken.targetSdk = targetSdkVersion;
            atoken.requestedOrientation = requestedOrientation;
            atoken.layoutConfigChanges = (configChanges &
                    (ActivityInfo.CONFIG_SCREEN_SIZE | ActivityInfo.CONFIG_ORIENTATION)) != 0;
            atoken.mLaunchTaskBehind = launchTaskBehind;
            atoken.mAlwaysFocusable = alwaysFocusable;
            atoken.mRotationAnimationHint = rotationAnimationHint;

            Task task = mTaskIdToTask.get(taskId);
            if (task == null) {
                task = createTaskLocked(taskId, stackId, userId, atoken, taskBounds, config);
            }
            task.addAppToken(addPos, atoken, taskResizeMode, homeTask);
            //将atoken放入到mTokenMap中，等应用程序addWindow时，进行身份验证
            //其中token.asBinder()是IApplicationToken.Stub的代理，atoken就是根据代理，得到对应AppWindowToken
            mTokenMap.put(token.asBinder(), atoken);

            // Application tokens start out hidden.
            atoken.hidden = true;
            atoken.hiddenRequested = true;
        }
    }
```

### 1.1.3、AMS跨Binder调用应用进程的scheduleLaunchActivity()将Token传递给上层应用进程

当框架通过ApplicationThread的代理回调到ActivityThread的时候，将对应的步骤一种生成的token代理传入。 ActivityStackSupervisor.realStartActivityLocked()

```java
    [->ActivityStackSupervisor.java]
    final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
        boolean andResume, boolean checkConfig) throws RemoteException {

      ......
        app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                System.identityHashCode(r), r.info, new Configuration(mService.mConfiguration),
                new Configuration(task.mOverrideConfig), r.compat, r.launchedFromPackage,
                task.voiceInteractor, app.repProcState, r.icicle, r.persistentState, results,
                newIntents, !andResume, mService.isNextTransitionForward(), profilerInfo);

       ......
    return true;
}
```

这里通过调用IApplicationThread的方法实现的，这里调用的是scheduleLaunchActivity()方法，所以真正执行的是ActivityThread中的scheduleLaunchActivity()。

```java
[-> ActivityThread.java :ApplicationThread]
        @Override
        public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
                ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
                CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
                int procState, Bundle state, PersistableBundle persistentState,
                List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
                boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {

            updateProcessState(procState, false);

            ActivityClientRecord r = new ActivityClientRecord();
            //传递给了ActivityThread的token，这个token就是IApplicationToken.Stub的代理
            r.token = token;
            ......
            updatePendingConfiguration(curConfig);

            sendMessage(H.LAUNCH_ACTIVITY, r);
        }
```

### 1.1.4、Activity窗口添加过程

详细过程请查看：【Android 7.1.2 (Android N) Activity-Window加载显示流程分析】

```java
[->ActivityThread.java]
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
            if (r.activity.mVisibleFromClient) {
                r.activity.makeVisible();
            }
        }
        ......
    }
}
```

进而层层调用到：ViewRootImpl.setView()

```java
[->ViewRootImpl.java]
WindowManager.LayoutParams l = r.window.getAttributes();
```

ViewRootImpl.setView()函数中添加Activity窗口时在参数mWindowAttributes中携带Token代理对象

```java
[->ViewManager.java]
public int addWindow(Session session, IWindow client, int seq,  
        WindowManager.LayoutParams attrs, int viewVisibility, int displayId,  
        Rect outContentInsets, Rect outStableInsets, InputChannel outInputChannel) {  
        ......  
        boolean addToken = false;  
        //attrs这个是应用程序ActivityClientRecord中传递过来的参数，其中的attrs.token就是步骤三种的r.token
        WindowToken token = mTokenMap.get(attrs.token);  
        ......  
        win = new WindowState(this, session, client, token,  
                attachedWindow, appOp[0], seq, attrs, viewVisibility, displayContent);  

        mWindowMap.put(client.asBinder(), win);  
        ......  
}
```

根据Binder机制可以知道从上层应用传递过来的Token代理对象会转换成SystemServer进程中的Token本地对象，后者与第2步中从Token对象是同一个对象，所以上面调用mTokenMap.get(attrs.token)时便能返回正确返回一个WindowToken（这个WindowToken其实是一个APPWindowToken），这样添加的窗口也就跟Activity关联上了。

### 1.2、WMS组织方式

Activity管理服务ActivityManagerService中每一个ActivityRecord对象在Window管理服务WindowManagerService中都对应有一个AppWindowToken对象。

此外，在输入法管理服务InputMethodManagerService中，每一个输入法窗口都对应有一个Binder对象，这个Binder对象在Window管理服务WindowManagerService又对应有一个WindowToken对象。

与输入法窗口类似，在壁纸管理服务WallpaperManagerService中，每一个壁纸窗口都对应有一个Binder对象，这个Binder对象在Window管理服务WindowManagerService也对应有一个WindowToken对象。

在Window管理服务WindowManagerService中，无论是AppWindowToken对象，还是WindowToken对象，它们都是用来描述一组有着相同令牌的窗口的，每一个窗口都是通过一个WindowState对象来描述的。例如，一个Activity组件窗口可能有一个启动窗口（Starting Window），还有若干个子窗口，那么这些窗口就会组成一组，并且都是以Activity组件在Window管理服务WindowManagerService中所对应的AppWindowToken对象为令牌的。从抽象的角度来看，就是在Window管理服务WindowManagerService中，每一个令牌（AppWindowToken或者WindowToken）都是用来描述一组窗口（WindowState）的，并且每一个窗口的子窗口也是与它同属于一个组，即都有着相同的令牌。

![Markdown](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/android.wms/05-Android-WMS-AppWindowToken.png)
其中，Activity Stack是在ActivityManagerService服务中创建的，Token List和Window Stack是在WindowManagerService中创建的，而Binder for IM和Binder for WP分别是在InputMethodManagerService服务和WallpaperManagerService服务中创建的，用来描述一个输入法窗口和一个壁纸窗口。

### 1.3、WMS窗口类型

添加一个窗口是通过 WindowManagerGlobal.addView()来完成的，分析 addView 方法的参数，有三个参数是必不可少的，view，params，以及 display。而 display 一般直接取 WindowMnagerImpl 中的 mDisplay，表示要输出的显示设备。view 自然表示要显示的 View，而 params 是 WindowManager.LayoutParams，用来描述这个 view 的些窗口属性，其中一个重要的参数 type，用来描述窗口的类型。

````java
[->WindowManagerGlobal]
 public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }
        if (display == null) {
            throw new IllegalArgumentException("display must not be null");
        }
        if (!(params instanceof WindowManager.LayoutParams)) {
            throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
        }
}

````
打开WindowManager类，看到静态内部类。

``` java
[->WindowManager]
public static class LayoutParams extends ViewGroup.LayoutParams implements Parcelable {
......
}
````

可以看到在LayoutParams中，有2个比较重要的参数: flags,type。 我们简要的分析一下flags,该参数表示Window的属性，它有很多选项，通过这些选项可以控制Window的显示特性，这里主要介绍几个比较常用的选项。

FLAG_NOT_FOCUSABLE 表示Window不需要获取焦点，也不需要接收各种输入事件，此标记会同时启用FLAG_NOT_TOUCH_MODAL，最终事件会直接传递给下层具有焦点的Window。

FLAG_NOT_TOUCH_MODAL 系统会将当前Window区域以外的单击事件传递给底层的Window，当前Window区域以内的单击事件则自己处理，这个标记很重要，一般来说都需要开启此标记，否则其他Window将无法接收到单击事件。

FLAG_SHOW_WHEN_LOCKED 开启此模式可以让Window显示在锁屏的界面上。

> Type参数表示Window的类型，Window有三种类型，分别是应用Window、子Window、系统Window。应用类Window对应着一个Activity。子Window不能单独存在，它需要附属在特定的父Window之中，比如常见的PopupWindow就是一个子Window。有些系统Window是需要声明权限才能创建的Window，比如Toast和系统状态栏这些都是系统Window。

### 1.3.1、应用窗口

Activity 对应的窗口类型是应用窗口， 所有 Activity 默认的窗口类型是 TYPE_BASE_APPLICATION。 WindowManager 的 LayoutParams 的默认类型是 TYPE_APPLICATION。 Dialog 并没有设置type，所以也是默认的窗口类型即 TYPE_APPLICATION。

```java
[->WindowManager.LayoutParams]

public LayoutParams() {
    super(LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT);
    type = TYPE_APPLICATION;
    format = PixelFormat.OPAQUE;
}
```

type层级                      | 类型
:-------------------------- | :----------------------------------------
FIRST_APPLICATION_WINDOW=1  | 开始应用程序窗口，第一个普通应用窗口
TYPE_BASE_APPLICATION=1     | 所有程序窗口的base窗口，其他应用程序窗口都显示在它上面
TYPE_APPLICATION=2          | 普通应用程序窗口，token必须设置为Activity的token来指定窗口属于谁
TYPE_APPLICATION_STARTING=3 | 应用程序启动时先显示此窗口，当真正的窗口配置完成后，关闭此窗口
LAST_APPLICATION_WINDOW=99  | 最后一个应用窗口

### 1.3.2、子窗口

子窗口不能单独存在，它需要附属在特定的父Window之中，例如开篇第一张图，绿色框框即为popupWindow，它就是子窗口，类型一般为TYPE_APPLICATION_PANEL。之所以称为子窗口，即它的父窗口显示时，子窗口才显示。父窗口不显示，它也不显示。追随父窗口。

type层级                                | 类型
:------------------------------------ | :----------------------------------
FIRST_SUB_WINDOW=1000                 | 第一个子窗口
TYPE_APPLICATION_PANEL=1000           | 应用窗口的子窗口,popupWindow的默认类型
TYPE_APPLICATION_MEDIA=1001           | 媒体窗口
TYPE_APPLICATION_SUB_PANEL=1002       | TYPE_APPLICATION_PANE的子窗口
TYPE_APPLICATION_ATTACHED_DIALOG=1003 | 对话框，类似于面板窗口(OptionMenu,ContextMenu)
TYPE_APPLICATION_MEDIA_OVERLAY=1004   | 媒体信息，显示在媒体层和程序窗口之间，需要实现半透明效果
LAST_SUB_WINDOW=1999                  | 最后一个子窗口

### 1.3.3、系统窗口

系统窗口跟应用窗口不同，不需要对应 Activity。跟子窗口不同，不需要有父窗口。一般来讲，系统窗口应该由系统来创建的，例如发生异常，ANR时的提示框，又如系统状态栏，屏保等。但是，Framework 还是定义了一些，可以被应用所创建的系统窗口，如 TYPE _TOAST，TYPE _INPUT_ METHOD，TYPE _WALLPAPTER 等等。

type层级                          | 类型
:------------------------------ | :----------------------------------
FIRST_SYSTEM_WINDOW=2000        | 第一个系统窗口
TYPE_STATUS_BAR=2000            | 状态栏，只能有一个状态栏，位于屏幕顶端
TYPE_SEARCH_BAR =2001           | 搜索栏
TYPE_PHONE=2002                 | 电话窗口，它用于电话交互
TYPE_SYSTEM_ALERT=2003          | 系统警告，出现在应用程序窗口之上
TYPE_KEYGUARD=2004              | 锁屏窗口
TYPE_TOAST=2005                 | 信息窗口，用于显示Toast
TYPE_SYSTEM_OVERLAY=2006        | 系统顶层窗口，显示在其他内容之上，此窗口不能获得输入焦点，否则影响锁屏
TYPE_PRIORITY_PHONE=2007        | 当锁屏时显示的来电显示窗口
TYPE_SYSTEM_DIALOG=2008         | 系统对话框
TYPE_KEYGUARD_DIALOG=2009       | 锁屏时显示的对话框
TYPE_SYSTEM_ERROR=2010          | 系统内部错误提示
TYPE_INPUT_METHOD=2011          | 输入法窗口，显示于普通应用/子窗口之上
TYPE_INPUT_METHOD_DIALOG=2012   | 输入法中备选框对应的窗口
TYPE_WALLPAPER=2013             | 墙纸窗口
TYPE_STATUS_BAR_PANEL=2014      | 滑动状态条后出现的窗口
TYPE_SECURE_SYSTEM_OVERLAY=2015 | 安全系统覆盖窗口
......                          | ......
LAST_SYSTEM_WINDOW=2999         | 最后一个系统窗口

那么，这个type层级到底有什么作用呢？ Window是分层的，每个Window都有对应的z-ordered，（z轴，从1层层叠加到2999，你可以将屏幕想成三维坐标模式）层级大的会覆盖在层级小的Window上面。

在三类Window中，应用Window的层级范围是1~99。子Window的层级范围是1000~1999，系统Window的层级范围是2000~2999，这些层级范围对应着WindowManager.LayoutParams的type参数。如果想要Window位于所有Window的最顶层，那么采用较大的层级即可。另外有些系统层级的使用是需要声明权限的。

## （二）、Window Size（大小）和 Window Position（位置） 计算过程

之前在【Android 7.1.2 (Android N) Android Graphics 系统分析 [i.wonder~]】分析过，当Vsync事件到来时，就会通过Choreographer的postCallback()，接着执行mTraversalRunnable对象的run()方法。 mTraversalRunnable对象的类型为TraversalRunnable，该类实现了Runnable接口，在其run()函数中调用了doTraversal()函数来完成窗口布局。为了分析的连贯性，这里重新贴一下源码：

```java
[->ViewRootImpl.java]
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

performTraversals()函数相当复杂，其主要实现以下几个重要步骤：

1.执行窗口测量；

2.执行窗口注册；

3.执行窗口布局；

4.执行窗口绘图；

```java
[->ViewRootImpl.java]
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

### 2.1、 Android 屏幕区域介绍

首先来看relayoutWindow()。relayoutWindow() 是Window Manager Service 重要工作之一，它的流程如下图所示：

![Markdown](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/android.wms/06-Android-WMS-relayoutWindow-flow.png.png)

每个View将期望窗口尺寸交给WMS（WindowManager Service). WMS 将所有的窗口大小以及当前的Overscan区域传给WPM （WindowManager Policy). WPM根据用户配置确定每个Window在最终Display输出上的位置以及需要分配的Surface大小。 返回这些信息给每个View，他们将在给会的区域空间里绘图。 Android里定义了很多区域,如下图所示 
![Markdown](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/android.wms/07-Android-WMS-OverSan-area.png)

**Overscan:** Overscan 是电视特有的概念，上图中黄色部分就是Overscan区域，指的是电视机屏幕四周某些不可见的区域（因为电视特性，这部分区域的buffer内容显示时被丢弃），也意味着如果窗口的某些内容画在这个区域里，它在某些电视上就会看不到。为了避免这种情况发生，通常要求UI不要画在屏幕的边角上，而是预留一定的空间。因为Overscan的区域大小随着电视不 同而不同，它一般由终端用户通过UI指定，（比如说GoogleTV里就有确定Overscan大小的应用）。

**OverscanScreen, Screen:** OverscanScreen 是包含Overscan区域的屏幕大小,而Screen则为去除Overscan区域后的屏幕区域, OverscanScreen > Screen.

**Restricted and Unrestricted:** 某些区域是被系统保留的，比如说手机屏幕上方的状态栏(如图纸绿色区域）和下方的导航栏，根据是否包括这些预留的区域，Android把区域分为Unrestricted Area 和 Resctrited Aread, 前者包括这部分预留区域，后者则不包含, Unrestricted area > Rectricted area。

**mFrame, mDisplayFrame, mContainingFrame** Frame指的是一片内存区域, 对应于屏幕上的一块矩形区域. mFrame的大小就是Surface的大小, 如上上图中的蓝色区域. mDisplayFrame 和 mContainingFrame 一般和mFrame 大小一致. mXXX 是Window(ViewRootImpl, Windowstate) 里面定义的成员变量.

**mContentFrame, mVisibleFrame** 一个Surface的所有内容不一定在屏幕上都得到显示, 与Overscan重叠的部分会被截掉, 系统的其他窗口也会遮挡掉部分区域 (比如短信窗口，ContentFrame是800x600(没有Status Bar), 但当输入法窗口弹出是，变成了800x352), 剩下的区域称为Visible Frame, UI内容只有画在这个区域里才能确保可见. 所以也称为Content Frame. mXXX也是Window(ViewRootImpl, WindowState) 里面定义的成员变量. 
![Markdown](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/android.wms/08-Android-WMS-Content-Insets.png)

**Insets** insets的定义如上图所示, 用了表示某个Frame的边缘大小.

### 2.2、 Window 大小位置计算过程

在Android系统中，Activity窗口的大小是由WindowManagerService服务来计算的。WindowManagerService服务会根据屏幕及其装饰区的大小来决定Activity窗口的大小。一个Activity窗口只有知道自己的大小之后，才能对它里面的UI元素进行测量、布局以及绘制。 一般来说，Activity窗口的大小等于整个屏幕的大小，但是它并不占据着整块屏幕。为了理解这一点，我们首先分析一下Activity窗口的区域是如何划分的。 我们知道，Activity窗口的上方一般会有一个状态栏，用来显示3G信号、电量使用等图标，如图 
![Markdown](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/android.wms/09-Android-WMS-Content-Visible-Frame.png)

从Activity窗口剔除掉状态栏所占用的区域之后，所得到的区域就称为内容区域（Content Region）。顾名思义，内容区域就是用来显示Activity窗口的内容的。我们再抽象一下，假设Activity窗口的四周都有一块类似状态栏的区域，那么将这些区域剔除之后，得到中间的那一块区域就称为内容区域，而被剔除出来的区域所组成的区域就称为内容边衬区域（Content Insets）。Activity窗口的内容边衬区域可以用一个四元组（content-left, content-top, content-right, content-bottom）来描述，其中，content-left、content-right、content-top、content-bottom分别用来描述内容区域与窗口区域的左右上下边界距离。 我们还知道，Activity窗口有时候需要显示输入法窗口，如图。 
![Markdown](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/android.wms/10-Android-WMS-InputMethod-Content-Frame.png)

这时候Activity窗口的内容区域的大小有可能没有发生变化，这取决于它的Soft Input Mode。我们假设Activity窗口的内容区域没有发生变化，但是它在底部的一些区域被输入法窗口遮挡了，即它在底部的一些内容是不可见的。从Activity窗口剔除掉状态栏和输入法窗口所占用的区域之后，所得到的区域就称为可见区域（Visible Region）。同样，我们再抽象一下，假设Activity窗口的四周都有一块类似状态栏和输入法窗口的区域，那么将这些区域剔除之后，得到中间的那一块区域就称为可见区域，而被剔除出来的区域所组成的区域就称为可见边衬区域（Visible Insets）。Activity窗口的可见边衬区域可以用一个四元组（visible-left, visible-top, visible-right, visible-bottom）来描述，其中，visible-left、visible-right、visible-top、visible-bottom分别用来描述可见区域与窗口区域的左右上下边界距离。

在大多数情况下，Activity窗口的内容区域和可见区域的大小是一致的，而状态栏和输入法窗口所占用的区域又称为屏幕装饰区。理解了这些概念之后，我们就可以推断，WindowManagerService服务实际上就是需要根据屏幕以及可能出现的状态栏和输入法窗口的大小来计算出Activity窗口的整体大小及其内容区域边衬和可见区域边衬的大小。有了这三个数据之后，Activity窗口就可以对它里面的UI元素进行测量、布局以及绘制等操作了。 总体流程图： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/android.wms/11-Android-WMS-relayoutWindow-time-diagram.png)

这个过程可以分为13个步骤，接下来我们就详细分析每一个步骤。

### 2.2.1、ViewRootImpl.performTraversals()

```java
[->ViewRootImpl.java]
private void performTraversals() {
    // cache mView since it is used so much below...
    final View host = mView;
    ......
    WindowManager.LayoutParams lp = mWindowAttributes;

    int desiredWindowWidth;
    int desiredWindowHeight;
    ......
    Rect frame = mWinFrame;
    if (mFirst) {
        mFullRedrawNeeded = true;
        mLayoutRequested = true;
        //第一次被请求执行测量、布局和绘制操作，desiredWindowWidth和desiredWindowHeight等于Display Size，否则mWinFrame保存的宽度和高度值。
        if (shouldUseDisplaySize(lp)) {
            Point size = new Point();
            mDisplay.getRealSize(size);
            desiredWindowWidth = size.x;
            desiredWindowHeight = size.y;
        } else {
            Configuration config = mContext.getResources().getConfiguration();
            desiredWindowWidth = dipToPx(config.screenWidthDp);
            desiredWindowHeight = dipToPx(config.screenHeightDp);
        }
        ......
        host.dispatchAttachedToWindow(mAttachInfo, 0);
        mAttachInfo.mTreeObserver.dispatchOnWindowAttachedChange(true);
        dispatchApplyInsets(host);
    } else {
        //不是第一次请求，当desiredWindowWidth != mWidth || desiredWindowHeight != mHeight，说明Activity窗口的大小发生了变化，这时候windowSizeMayChange = true，以便接下来对Activity窗口大小变化进行处理
        desiredWindowWidth = frame.width();
        desiredWindowHeight = frame.height();
        if (desiredWindowWidth != mWidth || desiredWindowHeight != mHeight) {
            mFullRedrawNeeded = true;
            mLayoutRequested = true;
            windowSizeMayChange = true;
        }
    }
```

这段代码用来获得Activity窗口的当前宽度desiredWindowWidth和当前高度desiredWindowHeight。

继续阅读代码：

```java
[->ViewRootImpl.java::performTraversals()]
boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);
if (layoutRequested) {

    final Resources res = mView.getContext().getResources();

    if (mFirst) {
        ......
    } else {
        //AttachInfo对象用来描述Activity窗口的属性,mContentInsets和mVisibleInsets分别用来描述Activity窗口的当前内容边衬大小和可见边衬大小。
        //判断Activity窗口的OverscanInsets、ContentInsets、StableInsets、VisibleInsets大小是否发生了变化
        if (!mPendingOverscanInsets.equals(mAttachInfo.mOverscanInsets)) {
            insetsChanged = true;
        }
        ......
        //WRAP_CONTENT表明Activity窗口的大小要等于内容区域的大小，同时等于Display size
        if (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT
                || lp.height == ViewGroup.LayoutParams.WRAP_CONTENT) {
            windowSizeMayChange = true;

            if (shouldUseDisplaySize(lp)) {
                Point size = new Point();
                mDisplay.getRealSize(size);
                desiredWindowWidth = size.x;
                desiredWindowHeight = size.y;
            } else {
                Configuration config = res.getConfiguration();
                desiredWindowWidth = dipToPx(config.screenWidthDp);
                desiredWindowHeight = dipToPx(config.screenHeightDp);
            }
        }
    }
    // Ask host how big it wants to be
    //知道了顶层Activity窗口大小从而计算Activity内各个子View的大小
    windowSizeMayChange |= measureHierarchy(host, lp, res,
            desiredWindowWidth, desiredWindowHeight);
}
```

这段代码用来在Activity窗口主动请求WindowManagerService服务计算大小之前，对它的顶层视图进行一次测量操作。

继续阅读代码：

```java
[->ViewRootImpl.java::performTraversals()]
 if (mFirst || windowShouldResize || insetsChanged ||
        viewVisibilityChanged || params != null || mForceNextWindowRelayout) {
    ......
    try {
        ......
        //relayoutWindow来请求WMS计算Activity窗口的大小以及xxxInsets大小，并保存在PendingxxxInsets中
        relayoutResult = relayoutWindow(params, viewVisibility, insetsPending);
        ......
        if (contentInsetsChanged) {
            mAttachInfo.mContentInsets.set(mPendingContentInsets);
        }
        ......
        if (visibleInsetsChanged) {
            mAttachInfo.mVisibleInsets.set(mPendingVisibleInsets);
        }
        ......
    } catch (RemoteException e) {
    }
    ......
    //将计算得到的Activity窗口的左上角坐标保存在变量attachInfo所指向的一个AttachInfo对象的成员变量mWindowLeft和mWindowTop
    mAttachInfo.mWindowLeft = frame.left;
    mAttachInfo.mWindowTop = frame.top;
    //将计算得到的Activity窗口的宽度和高度保存在ViewRootImpl类的成员变量mWidth和mHeight中
    if (mWidth != frame.width() || mHeight != frame.height()) {
        mWidth = frame.width();
        mHeight = frame.height();
    }
```

这段代码主要调用relayoutWindow()来请求WMS计算Activity窗口的大小以及边忖xxxInsets大小。计算完毕之后，分别保存在mPendingXXXInsets中。

继续阅读代码：

```java
[->ViewRootImpl.java::performTraversals()]
 if (!mStopped || mReportNextDraw) {
            boolean focusChangedDueToTouchMode = ensureTouchModeLocally(
                    (relayoutResult&WindowManagerGlobal.RELAYOUT_RES_IN_TOUCH_MODE) != 0);
            if (focusChangedDueToTouchMode || mWidth != host.getMeasuredWidth()
                    || mHeight != host.getMeasuredHeight() || contentInsetsChanged ||
                    updatedConfiguration) {
                int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
                int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
                ......
                 // Ask host how big it wants to be
                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);

                ......
                int width = host.getMeasuredWidth();
                int height = host.getMeasuredHeight();
                boolean measureAgain = false;

                if (lp.horizontalWeight > 0.0f) {
                    width += (int) ((mWidth - width) * lp.horizontalWeight);
                    childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(width,
                            MeasureSpec.EXACTLY);
                    measureAgain = true;
                }
                if (lp.verticalWeight > 0.0f) {
                    height += (int) ((mHeight - height) * lp.verticalWeight);
                    childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(height,
                            MeasureSpec.EXACTLY);
                    measureAgain = true;
                }

                if (measureAgain) {
                    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                }

                layoutRequested = true;
            }
        }
    } else {
    }
```

这段代码用来检查是否需要重新测量Activity窗口的大小。

经过前面漫长的操作后，Activity窗口的大小测量工作终于尘埃落定，这时候就可以对Activity窗口的内容进行布局 performLayout(lp, mWidth, mHeight)和进行绘画了，performDraw()，由于主要关注Activity窗口大小计算过程，在此不做继续分析。

### 2.2.2、ViewRootImpl.relayoutWindow()

通过调用这个Session对象的成员函数relayout()来请求WindowManagerService服务计算Activity窗口的大小。

```java
[->ViewRootImpl.java]
private int relayoutWindow(WindowManager.LayoutParams params, int viewVisibility,
        boolean insetsPending) throws RemoteException {
    .....
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

参数说明： 1、mWindow 用来标志要计算的是哪一个Activity窗口的大小p 2、Activity窗口的顶层视图经过测量后得到的宽度和高度 3、Activity窗口的可见状态，即viewVisibility 4、Activity窗口是否有额外的内容区域边衬和可见区域边衬等待告诉给WindowManagerService服务，即参数insetsPending 5、mWinFrame，这是一个输出参数，用来保存WindowManagerService服务计算后得到的Activity窗口的大小 6、mPendingOverscanInsets用来保存Overscan边衬，mPendingContentInsets用来保存内容区域边衬，mPendingVisibleInsets用来保存可见区域边衬，mPendingStableInsets用来保存可能被系统UI元素部分或完全遮蔽的全屏窗口区域 7、mPendingConfiguration，这是一个输出参数，用来保存WindowManagerService服务返回来的Activity窗口的配置信息 8、mSurface，这是一个输出参数，用来保存WindowManagerService服务返回来的Activity窗口的绘图表面

### 2.2.3、Session.relayout()

```java
[->Session.java]
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

只是调用了WindowManagerService类的成员函数relayoutWindow来进一步计算参数window所描述的一个Activity窗品的大小

### 2.2.4、WindowManagerService.relayoutWindow()

```java
[->WindowManagerService.java]
public int relayoutWindow(Session session, IWindow client, int seq,
        WindowManager.LayoutParams attrs, int requestedWidth,
        int requestedHeight, int viewVisibility, int flags,
        Rect outFrame, Rect outOverscanInsets, Rect outContentInsets,
        Rect outVisibleInsets, Rect outStableInsets, Rect outOutsets, Rect outBackdropFrame,
        Configuration outConfig, Surface outSurface) {
    int result = 0;
    ......
    synchronized(mWindowMap) {
        WindowState win = windowForClientLocked(session, client, false);
        WindowStateAnimator winAnimator = win.mWinAnimator;
        if (viewVisibility != View.GONE) {
            win.setRequestedSize(requestedWidth, requestedHeight);
        }
        .....
        if (attrs != null) {
                mPolicy.adjustWindowParamsLw(attrs);
                ......
        }
        win.setWindowScale(win.mRequestedWidth, win.mRequestedHeight);
        ......
        if (viewVisibility == View.VISIBLE &&
                (win.mAppToken == null || !win.mAppToken.clientHidden)) {
            result = relayoutVisibleWindow(outConfig, result, win, winAnimator, attrChanges,
                    oldVisibility);
            try {
                result = createSurfaceControl(outSurface, result, win, winAnimator);
            } catch (Exception e) {
                ......
            }
            ......
            win.adjustStartingWindowFlags();
        } else {
        ......
        }
        ......
        mWindowPlacerLocked.performSurfacePlacement();
        ......
        outFrame.set(win.mCompatFrame);
        outOverscanInsets.set(win.mOverscanInsets);
        outContentInsets.set(win.mContentInsets);
        outVisibleInsets.set(win.mVisibleInsets);
        outStableInsets.set(win.mStableInsets);
        outOutsets.set(win.mOutsets);
        outBackdropFrame.set(win.getBackdropFrame(win.mFrame));
        ......
    return result;
}
```

只关注relayoutWindow中与窗口大小计算有关的逻辑，计算过程如下所示： 1、参数requestedWidth和requestedHeight描述的是应用程序进程请求设置Activity窗口中的宽度和高度，它们会被记录在WindowState对象win的成员变量mRequestedWidth和mRequestedHeight中 2、WindowState对象win的成员变量mAttrs，它指向的是一个WindowManager.LayoutParams对象，用来描述Activity窗口的布局参数 3、调用WindowSurfacePlacer.performSurfacePlacement()来计算Activity窗口的大小。计算完成之后，参数client所描述的Activity窗口的大小、内容区域边衬大小和可见区域边边衬大小就会分别保存在WindowState对象win的成员变量mCompatFrame、mOverscanInsets、mContentInsets、mVisibleInsets、mStableInsets、mOutsets中 4、 将WindowState对象win的成员变量mCompatFrame、mOverscanInsets、mContentInsets、mVisibleInsets、mStableInsets、mOutsets拷贝赋值对应变量中，以便可以返回给应用程序进程

经过上述4个操作后，Activity窗口的大小计算过程就完成了，接下来我们继续分析WindowSurfacePlacer.performSurfacePlacement()的实现，以便可以详细了解Activity窗口的大小计算过程

### 2.2.5、WindowSurfacePlacer.performSurfacePlacement()

```java
[->WindowSurfacePlacer.java]
final void performSurfacePlacement() {
    if (mDeferDepth > 0) {
        return;
    }
    int loopCount = 6;
    do {
        mTraversalScheduled = false;
        performSurfacePlacementLoop();
        mService.mH.removeMessages(DO_TRAVERSAL);
        loopCount--;
    } while (mTraversalScheduled && loopCount > 0);
    mWallpaperActionPending = false;
}
```

### 2.2.6.WindowSurfacePlacer.performSurfacePlacementLoop()

```java
[->WindowSurfacePlacer.java]
private void performSurfacePlacementLoop() {
       mInLayout = true;
        boolean recoveringMemory = false;
        if (!mService.mForceRemoves.isEmpty()) {
            recoveringMemory = true;

            while (!mService.mForceRemoves.isEmpty()) {
                WindowState ws = mService.mForceRemoves.remove(0);
                mService.removeWindowInnerLocked(ws);
            }
            ......
        }
    ......
    try {
        performSurfacePlacementInner(recoveringMemory);

        mInLayout = false;

        if (mService.needsLayout()) {
            if (++mLayoutRepeatCount < 6) {
                requestTraversal();
            } else {
                Slog.e(TAG, "Performed 6 layouts in a row. Skipping");
                mLayoutRepeatCount = 0;
            }
        } else {
            mLayoutRepeatCount = 0;
        }
        if (mService.mWindowsChanged && !mService.mWindowChangeListeners.isEmpty()) {
            mService.mH.removeMessages(REPORT_WINDOWS_CHANGE);
            mService.mH.sendEmptyMessage(REPORT_WINDOWS_CHANGE);
        }
    } catch (RuntimeException e) {
        mInLayout = false;
    }
}
```

在调用成员函数performSurfacePlacementInner()刷新系统UI的前后 1、检查系统中是否存在强制删除的窗口 2、检查系统中是否有窗口需要移除

### 2.2.7、WindowSurfacePlacer.performSurfacePlacementInner()

继续分析的performSurfacePlacementInner()实现，以便可以了解Activity窗口的大小计算过程。

```java
[->WindowSurfacePlacer.java]
private void performSurfacePlacementInner(boolean recoveringMemory) {
    if (DEBUG_WINDOW_TRACE) Slog.v(TAG, "performSurfacePlacementInner: entry. Called by "
            + Debug.getCallers(3));

    ......

    final DisplayContent defaultDisplay = mService.getDefaultDisplayContentLocked();
    final DisplayInfo defaultInfo = defaultDisplay.getDisplayInfo();
    final int defaultDw = defaultInfo.logicalWidth;
    final int defaultDh = defaultInfo.logicalHeight;
    SurfaceControl.openTransaction();
    try {
        applySurfaceChangesTransaction(recoveringMemory, numDisplays, defaultDw, defaultDh);
    } catch (RuntimeException e) {
        Slog.wtf(TAG, "Unhandled exception in Window Manager", e);
    } finally {
        SurfaceControl.closeTransaction();
    }

    final WindowList defaultWindows = defaultDisplay.getWindowList();
    ......
        final int N = mService.mPendingRemove.size();
        if (N > 0) {
            if (mService.mPendingRemoveTmp.length < N) {
                mService.mPendingRemoveTmp = new WindowState[N+10];
            }
            mService.mPendingRemove.toArray(mService.mPendingRemoveTmp);
            mService.mPendingRemove.clear();
            DisplayContentList displayList = new DisplayContentList();
            for (i = 0; i < N; i++) {
                WindowState w = mService.mPendingRemoveTmp[i];
                mService.removeWindowInnerLocked(w);
                final DisplayContent displayContent = w.getDisplayContent();
                if (displayContent != null && !displayList.contains(displayContent)) {
                    displayList.add(displayContent);
                }
            }

            for (DisplayContent displayContent : displayList) {
                mService.mLayersController.assignLayersLocked(displayContent.getWindowList());
                displayContent.layoutNeeded = true;
            }
        }
        ......
        mService.scheduleAnimationLocked();
        ......
}
```

可以看到进一步调用applySurfaceChangesTransaction()方法进行进一步计算

### 2.2.8、WindowSurfacePlacer.applySurfaceChangesTransaction()

```java
[->WindowSurfacePlacer.java]
private void applySurfaceChangesTransaction(boolean recoveringMemory, int numDisplays,
        int defaultDw, int defaultDh) {
    ......
    boolean focusDisplayed = false;

    for (int displayNdx = 0; displayNdx < numDisplays; ++displayNdx) {
        final DisplayContent displayContent = mService.mDisplayContents.valueAt(displayNdx);
        boolean updateAllDrawn = false;
        WindowList windows = displayContent.getWindowList();
        DisplayInfo displayInfo = displayContent.getDisplayInfo();
        final int displayId = displayContent.getDisplayId();
        final int dw = displayInfo.logicalWidth;
        final int dh = displayInfo.logicalHeight;
        final int innerDw = displayInfo.appWidth;
        final int innerDh = displayInfo.appHeight;
        final boolean isDefaultDisplay = (displayId == Display.DEFAULT_DISPLAY);

        // Reset for each display.
        mDisplayHasContent = false;
        mPreferredRefreshRate = 0;
        mPreferredModeId = 0;

        int repeats = 0;
        do {
            repeats++;
            if (repeats > 6) {//最多执行7次的while循环
                displayContent.layoutNeeded = false;
                break;
            }
            //通知SurfaceFlinger服务了，也就是让SurfaceFlinger服务更新它里面的各个Layer的属性值，以便可以对这些Layer执行可见性计算、合成等操作，最后渲染到硬件帧缓冲区中去
            if ((displayContent.pendingLayoutChanges & FINISH_LAYOUT_REDO_WALLPAPER) != 0 &&
                    mWallpaperControllerLocked.adjustWallpaperWindows()) {
                mService.mLayersController.assignLayersLocked(windows);
                displayContent.layoutNeeded = true;
            }
           ......
            if ((displayContent.pendingLayoutChanges & FINISH_LAYOUT_REDO_LAYOUT) != 0) {
                displayContent.layoutNeeded = true;
            }

            // FIRST LOOP: Perform a layout, if needed.
            //计算各个窗品的大小
            if (repeats < LAYOUT_REPEAT_THRESHOLD) {
                performLayoutLockedInner(displayContent, repeats == 1,
                        false /* updateInputWindows */);
            } else {
            }

            // FIRST AND ONE HALF LOOP: Make WindowManagerPolicy think
            // it is animating.
            displayContent.pendingLayoutChanges = 0;

            if (isDefaultDisplay) {
                mService.mPolicy.beginPostLayoutPolicyLw(dw, dh);
                for (int i = windows.size() - 1; i >= 0; i--) {
                    WindowState w = windows.get(i);
                    if (w.mHasSurface) {
                        mService.mPolicy.applyPostLayoutPolicyLw(w, w.mAttrs,
                                w.mAttachedWindow);
                    }
                }
                displayContent.pendingLayoutChanges |=
                        mService.mPolicy.finishPostLayoutPolicyLw();
            }
        } while (displayContent.pendingLayoutChanges != 0);

        ......
}
```

### 2.2.9、WindowSurfacePlacer.applySurfaceChangesTransaction()

```java
[->WindowSurfacePlacer.java]
final void performLayoutLockedInner(final DisplayContent displayContent,
        boolean initial, boolean updateInputWindows) {

    displayContent.layoutNeeded = false;
    WindowList windows = displayContent.getWindowList();
    boolean isDefaultDisplay = displayContent.isDefaultDisplay;

    DisplayInfo displayInfo = displayContent.getDisplayInfo();
    final int dw = displayInfo.logicalWidth;
    final int dh = displayInfo.logicalHeight;
    ......
    final int N = windows.size();
    int i;
    ///
    mService.mPolicy.beginLayoutLw(isDefaultDisplay, dw, dh, mService.mRotation,
            mService.mCurConfiguration.uiMode);
    ......
    displayContent.resize(mTmpContentRect);

    int seq = mService.mLayoutSeq+1;
    if (seq < 0) seq = 0;
    mService.mLayoutSeq = seq;

    boolean behindDream = false;

    int topAttached = -1;
    for (i = N-1; i >= 0; i--) {
        final WindowState win = windows.get(i);
        ......
        final boolean gone = (behindDream && mService.mPolicy.canBeForceHidden(win, win.mAttrs))
                || win.isGoneForLayoutLw();

        ......
        if (!gone || !win.mHaveFrame || win.mLayoutNeeded
                || ((win.isConfigChanged() || win.setReportResizeHints())
                        && !win.isGoneForLayoutLw() &&
                        ((win.mAttrs.privateFlags & PRIVATE_FLAG_KEYGUARD) != 0 ||
                        (win.mHasSurface && win.mAppToken != null &&
                        win.mAppToken.layoutConfigChanges)))) {
            if (!win.mLayoutAttached) {
                ......
                win.mLayoutNeeded = false;
                win.prelayout();
                mService.mPolicy.layoutWindowLw(win, null);
                win.mLayoutSeq = seq;

                // Window frames may have changed. Update dim layer with the new bounds.
                final Task task = win.getTask();
                if (task != null) {
                    displayContent.mDimLayerController.updateDimLayer(task);
                }
                ......
            } else {
            }
        }
    }

    boolean attachedBehindDream = false;
    ......
    for (i = topAttached; i >= 0; i--) {
        final WindowState win = windows.get(i);
        if (win.mLayoutAttached) {
            ......
            if ((win.mViewVisibility != View.GONE && win.mRelayoutCalled)
                    || !win.mHaveFrame || win.mLayoutNeeded) {
                ......
                win.mLayoutNeeded = false;
                win.prelayout();
                mService.mPolicy.layoutWindowLw(win, win.mAttachedWindow);
                win.mLayoutSeq = seq;
            }
        } else if (win.mAttrs.type == TYPE_DREAM) {
            attachedBehindDream = behindDream;
        }
    }

    // Window frames may have changed. Tell the input dispatcher about it.
    mService.mInputMonitor.setUpdateInputWindowsNeededLw();
    if (updateInputWindows) {
        mService.mInputMonitor.updateInputWindowsLw(false /*force*/);
    }

    mService.mPolicy.finishLayoutLw();
    mService.mH.sendEmptyMessage(UPDATE_DOCKED_STACK_DIVIDER);
}
```

1、mPolicy指向的是一个窗口管理策略类，即PhoneWindowManager对象，主要是用来制定窗口的大小计算策略 2、准备阶段：调用PhoneWindowManager.beginLayoutLw()来设置屏幕的大小。包括NavigationBar、StatusBar大小计算 3、计算阶段：调用PhoneWindowManager.layoutWindowLw()来计算各个窗口的大小、内容区域边衬大小以及可见区域边衬大小。 4、结束阶段：调用PhoneWindowManager类的成员函数finishLayoutLw()来执行一些清理工作。

### 2.2.10、PhoneWindowManager.beginLayoutLw()

```java
[->PhoneWindowManager.java]
public void beginLayoutLw(boolean isDefaultDisplay, int displayWidth, int displayHeight,
                          int displayRotation, int uiMode) {
    mDisplayRotation = displayRotation;
    final int overscanLeft, overscanTop, overscanRight, overscanBottom;
    if (isDefaultDisplay) {
        switch (displayRotation) {
            case Surface.ROTATION_90:
                overscanLeft = mOverscanTop;
                overscanTop = mOverscanRight;
                overscanRight = mOverscanBottom;
                overscanBottom = mOverscanLeft;
                break;
            ......
        }
    } else {
        overscanLeft = 0;
        overscanTop = 0;
        overscanRight = 0;
        overscanBottom = 0;
    }
    mOverscanScreenLeft = mRestrictedOverscanScreenLeft = 0;
    mOverscanScreenTop = mRestrictedOverscanScreenTop = 0;
    mOverscanScreenWidth = mRestrictedOverscanScreenWidth = displayWidth;
    mOverscanScreenHeight = mRestrictedOverscanScreenHeight = displayHeight;
    mSystemLeft = 0;
    mSystemTop = 0;
    mSystemRight = displayWidth;
    mSystemBottom = displayHeight;
    mUnrestrictedScreenLeft = overscanLeft;
    mUnrestrictedScreenTop = overscanTop;
    mUnrestrictedScreenWidth = displayWidth - overscanLeft - overscanRight;
    mUnrestrictedScreenHeight = displayHeight - overscanTop - overscanBottom;
    mRestrictedScreenLeft = mUnrestrictedScreenLeft;
    mRestrictedScreenTop = mUnrestrictedScreenTop;
    mRestrictedScreenWidth = mSystemGestures.screenWidth = mUnrestrictedScreenWidth;
    mRestrictedScreenHeight = mSystemGestures.screenHeight = mUnrestrictedScreenHeight;
    mDockLeft = mContentLeft = mVoiceContentLeft = mStableLeft = mStableFullscreenLeft
            = mCurLeft = mUnrestrictedScreenLeft;
    mDockTop = mContentTop = mVoiceContentTop = mStableTop = mStableFullscreenTop
            = mCurTop = mUnrestrictedScreenTop;
    mDockRight = mContentRight = mVoiceContentRight = mStableRight = mStableFullscreenRight
            = mCurRight = displayWidth - overscanRight;
    mDockBottom = mContentBottom = mVoiceContentBottom = mStableBottom = mStableFullscreenBottom
            = mCurBottom = displayHeight - overscanBottom;
    mDockLayer = 0x10000000;
    mStatusBarLayer = -1;

    // start with the current dock rect, which will be (0,0,displayWidth,displayHeight)
    final Rect pf = mTmpParentFrame;
    final Rect df = mTmpDisplayFrame;
    final Rect of = mTmpOverscanFrame;
    final Rect vf = mTmpVisibleFrame;
    final Rect dcf = mTmpDecorFrame;
    pf.left = df.left = of.left = vf.left = mDockLeft;
    pf.top = df.top = of.top = vf.top = mDockTop;
    pf.right = df.right = of.right = vf.right = mDockRight;
    pf.bottom = df.bottom = of.bottom = vf.bottom = mDockBottom;
    dcf.setEmpty();  // Decor frame N/A for system bars.
        .......
        if (isDefaultDisplay) {
        navVisible |= !canHideNavigationBar();

        boolean updateSysUiVisibility = layoutNavigationBar(displayWidth, displayHeight,
                displayRotation, uiMode, overscanLeft, overscanRight, overscanBottom, dcf, navVisible, navTranslucent,navAllowedHidden, statusBarExpandedNotKeyguard);
        updateSysUiVisibility |= layoutStatusBar(pf, df, of, vf, dcf, sysui, isKeyguardShowing);
        ......
        if (updateSysUiVisibility) {
            updateSystemUiVisibilityLw();
        }
    }
}
```

1、初始化Overscan、UnrestrictedScreen、RestrictedScreen等屏幕区域变量 2、计算NavigationBar和StatusBar大小

### 2.2.11、PhoneWindowManager.layoutWindowLw()

```java
[->PhoneWindowManager.java]
Override
public void layoutWindowLw(WindowState win, WindowState attached) {
    ......
    final int fl = PolicyControl.getWindowFlags(win, attrs);
    final int pfl = attrs.privateFlags;
    final int sim = attrs.softInputMode;
    final int sysUiFl = PolicyControl.getSystemUiVisibility(win, null);

    final Rect pf = mTmpParentFrame;
    final Rect df = mTmpDisplayFrame;
    final Rect of = mTmpOverscanFrame;
    final Rect cf = mTmpContentFrame;
    final Rect vf = mTmpVisibleFrame;
    final Rect dcf = mTmpDecorFrame;
    final Rect sf = mTmpStableFrame;
    Rect osf = null;
    dcf.setEmpty();

    final boolean hasNavBar = (isDefaultDisplay && mHasNavigationBar
            && mNavigationBar != null && mNavigationBar.isVisibleLw());

    final int adjust = sim & SOFT_INPUT_MASK_ADJUST;

    if (isDefaultDisplay) {
        sf.set(mStableLeft, mStableTop, mStableRight, mStableBottom);
    } else {
        sf.set(mOverscanLeft, mOverscanTop, mOverscanRight, mOverscanBottom);
    }

    if (!isDefaultDisplay) {
        if (attached != null) {
            // If this window is attached to another, our display
            // frame is the same as the one we are attached to.
            setAttachedWindowFrames(win, fl, adjust, attached, true, pf, df, of, cf, vf);
        } else {
            // Give the window full screen.
            pf.left = df.left = of.left = cf.left = mOverscanScreenLeft;
            pf.top = df.top = of.top = cf.top = mOverscanScreenTop;
            pf.right = df.right = of.right = cf.right
                    = mOverscanScreenLeft + mOverscanScreenWidth;
            pf.bottom = df.bottom = of.bottom = cf.bottom
                    = mOverscanScreenTop + mOverscanScreenHeight;
        }
    } ......


    win.computeFrameLw(pf, df, of, cf, vf, dcf, sf, osf);

    ......
}
```

然后调用WindowState.computeFrameLw来计算窗口win的具体大小了。计算的结果便得到了窗口win的大小，以及它的内容区域边衬大小和可见区域边衬大小。

### 2.2.12、WindowState.computeFrameLw()

```java
[->WindowState.java]
public void computeFrameLw(Rect pf, Rect df, Rect of, Rect cf, Rect vf, Rect dcf, Rect sf,
        Rect osf) {
    ......
    final Task task = getTask();
    final boolean fullscreenTask = !isInMultiWindowMode();
    final boolean windowsAreFloating = task != null && task.isFloating();
    ......
    final Rect layoutContainingFrame;
    final Rect layoutDisplayFrame;

    // The offset from the layout containing frame to the actual containing frame.
    final int layoutXDiff;
    final int layoutYDiff;
    if (fullscreenTask || layoutInParentFrame()) {
        // We use the parent frame as the containing frame for fullscreen and child windows
        mContainingFrame.set(pf);
        mDisplayFrame.set(df);
        layoutDisplayFrame = df;
        layoutContainingFrame = pf;
        layoutXDiff = 0;
        layoutYDiff = 0;
    } else {
        task.getBounds(mContainingFrame);
        ......
        mDisplayFrame.set(mContainingFrame);
        layoutXDiff = !mInsetFrame.isEmpty() ? mInsetFrame.left - mContainingFrame.left : 0;
        layoutYDiff = !mInsetFrame.isEmpty() ? mInsetFrame.top - mContainingFrame.top : 0;
        layoutContainingFrame = !mInsetFrame.isEmpty() ? mInsetFrame : mContainingFrame;
        mTmpRect.set(0, 0, mDisplayContent.getDisplayInfo().logicalWidth,
                mDisplayContent.getDisplayInfo().logicalHeight);
        subtractInsets(mDisplayFrame, layoutContainingFrame, df, mTmpRect);
        ......
        layoutDisplayFrame = df;
        layoutDisplayFrame.intersect(layoutContainingFrame);
    }

    final int pw = mContainingFrame.width();
    final int ph = mContainingFrame.height();
    ......
    mOverscanFrame.set(of);
    mContentFrame.set(cf);
    mVisibleFrame.set(vf);
    mDecorFrame.set(dcf);
    mStableFrame.set(sf);
    final boolean hasOutsets = osf != null;
    final int fw = mFrame.width();
    final int fh = mFrame.height();
    ......
    if (windowsAreFloating && !mFrame.isEmpty()) {
        final int height = Math.min(mFrame.height(), mContentFrame.height());
        final int width = Math.min(mContentFrame.width(), mFrame.width());
        final DisplayMetrics displayMetrics = getDisplayContent().getDisplayMetrics();
        final int minVisibleHeight = Math.min(height, WindowManagerService.dipToPixel(
                MINIMUM_VISIBLE_HEIGHT_IN_DP, displayMetrics));
        final int minVisibleWidth = Math.min(width, WindowManagerService.dipToPixel(
                MINIMUM_VISIBLE_WIDTH_IN_DP, displayMetrics));
        final int top = Math.max(mContentFrame.top,
                Math.min(mFrame.top, mContentFrame.bottom - minVisibleHeight));
        final int left = Math.max(mContentFrame.left + minVisibleWidth - width,
                Math.min(mFrame.left, mContentFrame.right - minVisibleWidth));
        mFrame.set(left, top, left + width, top + height);
        mContentFrame.set(mFrame);
        mVisibleFrame.set(mContentFrame);
        mStableFrame.set(mContentFrame);
    } else if (mAttrs.type == TYPE_DOCK_DIVIDER) {
        mDisplayContent.getDockedDividerController().positionDockedStackedDivider(mFrame);
        mContentFrame.set(mFrame);
    } else {
        mContentFrame.set(......);

        mVisibleFrame.set(......);

        mStableFrame.set(......);
    }

    if (fullscreenTask && !windowsAreFloating) {
        // Windows that are not fullscreen can be positioned outside of the display frame,
        mOverscanInsets.set(......);
    }

    if (mAttrs.type == TYPE_DOCK_DIVIDER) {
        mStableInsets.set(......);
        mContentInsets.setEmpty();
        mVisibleInsets.setEmpty();
    } else {
        getDisplayContent().getLogicalDisplayRect(mTmpRect);
        boolean overrideRightInset = !fullscreenTask && mFrame.right > mTmpRect.right;
        boolean overrideBottomInset = !fullscreenTask && mFrame.bottom > mTmpRect.bottom;
        mContentInsets.set(......);

        mVisibleInsets.set(......);

        mStableInsets.set(......);
    }

    // Offset the actual frame by the amount layout frame is off.
    mFrame.offset(-layoutXDiff, -layoutYDiff);
    mCompatFrame.offset(-layoutXDiff, -layoutYDiff);
    mContentFrame.offset(-layoutXDiff, -layoutYDiff);
    mVisibleFrame.offset(-layoutXDiff, -layoutYDiff);
    mStableFrame.offset(-layoutXDiff, -layoutYDiff);

    mCompatFrame.set(mFrame);
    if (mEnforceSizeCompat) {
        mOverscanInsets.scale(mInvGlobalScale);
        mContentInsets.scale(mInvGlobalScale);
        mVisibleInsets.scale(mInvGlobalScale);
        mStableInsets.scale(mInvGlobalScale);
        mOutsets.scale(mInvGlobalScale);
        mCompatFrame.scale(mInvGlobalScale);
    }

    ......
}
```

整个窗口大小保存在WindowState类的成员变量mFrame中，而窗品的内容区域边衬大小和可见区域边衬大小分别保在WindowState类的成员变量mContentInsets和mVisibleInsets中

### 2.2.13、PhoneWindowManager.finishLayoutLw()

```java
[WindowState.java]
@Override
public void finishLayoutLw() {
    return;
}
```

## （三）、Window Z-Order 计算和调整过程

口的UI最终是需要通过SurfaceFlinger服务来统一渲染的，而SurfaceFlinger服务在渲染窗口的UI之前，需要计算基于各个窗口的Z轴位置来计算它们的可见区域。因此，WindowManagerService服务计算好每一个窗口的Z轴位置之后，还需要将它们设置到SurfaceFlinger服务中去，以便SurfaceFlinger服务可以正确地渲染每一个窗口的UI。 
![Markdown](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/android.wms/12-Android-WMS-Z-Order.png)

### 3.1、需要重新计算窗口Z轴位置的情景

在【Android 7.1.2 (Android N) Activity-Window加载显示流程分析】中已经详细介绍Window添加过程，这里直接从 WMS.addWindow开始分析。

```java
[->WindowManagerService.java]
public int addWindow(Session session, IWindow client, int seq,
        WindowManager.LayoutParams attrs, int viewVisibility, int displayId,
        Rect outContentInsets, Rect outStableInsets, Rect outOutsets,
        InputChannel outInputChannel) {
    .......
    synchronized(mWindowMap) {
        ......

        boolean addToken = false;
        WindowToken token = mTokenMap.get(attrs.token);
        AppWindowToken atoken = null;
        boolean addToastWindowRequiresToken = false;

        ......
        //
        WindowState win = new WindowState(this, session, client, token,
                attachedWindow, appOp[0], seq, attrs, viewVisibility, displayContent);
        ......

        origId = Binder.clearCallingIdentity();

        if (addToken) {
            mTokenMap.put(attrs.token, token);
        }
        win.attach();
        mWindowMap.put(client.asBinder(), win);
        ......

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
            ......
        }

        ......
        //重新计算系统中所有窗口的Z轴位置
        mLayersController.assignLayersLocked(displayContent.getWindowList());
        ......
    }
    ......

    return res;
}
```

WMS.relayoutWindow()也会调用WindowLayersController.assignLayersLocked()重新计算、调整系统中所有窗口的Z轴位置，由于原理类似这里不做解释。

### 3.2、计算系统中所有窗口的Z轴位置

接下来我们就通过WindowState类的构造函数来分析一个窗口的BaseLayer值是如何确定

```java
[WindowState.java]
WindowState(WindowManagerService service, Session s, IWindow c, WindowToken token,
       WindowState attachedWindow, int appOp, int seq, WindowManager.LayoutParams a,
       int viewVisibility, final DisplayContent displayContent) {
    ......

    if ((mAttrs.type >= FIRST_SUB_WINDOW &&
            mAttrs.type <= LAST_SUB_WINDOW)) {
        ......
        //windowTypeToLayerLw的返回值并且不是一个窗口的最终的BaseLayer值，而是要将它的返回值乘以一个常量TYPE_LAYER_MULTIPLIER，再加上另外一个常量TYPE_LAYER_OFFSET之后，才得到最终的BaseLayer值
        mBaseLayer = mPolicy.windowTypeToLayerLw(
                attachedWindow.mAttrs.type) * WindowManagerService.TYPE_LAYER_MULTIPLIER
                + WindowManagerService.TYPE_LAYER_OFFSET;
        mSubLayer = mPolicy.subWindowTypeToLayerLw(a.type);
        ......
}
```

一个窗口除了有一个BaseLayer值之外，还有一个SubLayer值，分别保存在一个对应的WindowState对象的成员变量mBaseLayer和mSubLayer。SubLayer值是用来描述一个窗口是否是另外一个窗口的子窗口的。 在继续分析WindowLayersController.assignLayersLocked()之前，我们首先分析PhoneWindowManager.windowTypeToLayerLw()和subWindowTypeToLayerLw()的实现，以便可以了解一个窗口的BaseLayer值和SubLayer值是如何确定的。

```java
[->PhoneWindowManager.java]
@Override
public int windowTypeToLayerLw(int type) {
    if (type >= FIRST_APPLICATION_WINDOW && type <= LAST_APPLICATION_WINDOW) {
        return 2;
    }
    switch (type) {
   ......
    case TYPE_SYSTEM_DIALOG:
        return 7;
    case TYPE_TOAST:
        // toasts and the plugged-in battery thing
        return 8;
   ......
    case TYPE_SYSTEM_ALERT:
        // like the ANR / app crashed dialogs
        return 11;
    ......
    case TYPE_STATUS_BAR_SUB_PANEL:
        return 15;
    case TYPE_STATUS_BAR:
        return 16;
    case TYPE_STATUS_BAR_PANEL:
        return 17;
    case TYPE_KEYGUARD_DIALOG:
        return 18;
    ......
    case TYPE_NAVIGATION_BAR:
        // the navigation bar, if available, shows atop most things
        return 21;
    case TYPE_NAVIGATION_BAR_PANEL:
        // some panels (e.g. search) need to show on top of the navigation bar
        return 22;
    ......
    }
    return 2;
}

/** {@inheritDoc} */
@Override
public int subWindowTypeToLayerLw(int type) {
    switch (type) {
    case TYPE_APPLICATION_PANEL:
    case TYPE_APPLICATION_ATTACHED_DIALOG:
        return APPLICATION_PANEL_SUBLAYER;
    ......
    case TYPE_APPLICATION_ABOVE_SUB_PANEL:
        return APPLICATION_ABOVE_SUB_PANEL_SUBLAYER;
    }
    return 0;
}
```

主要根据不同的Window Type返回不一样的数值。

```java
[->WindowManagerService.java]
static final int TYPE_LAYER_MULTIPLIER = 10000;
static final int TYPE_LAYER_OFFSET = 1000;
```

我们可以用adb shell dumpsys window -a 命令查看一下Layer的数值，可以看到StatusBar的数值计算： mBaseLayer = 16 _WindowManagerService.TYPE_LAYER_MULTIPLIER+ WindowManagerService.TYPE_LAYER_OFFSET StatusBar.mBaseLayer = 16_ 10000 + 1000 = 161000 。

```java
  Window #4 Window{3c1f1fb u0 StatusBar}:
    mBaseLayer=161000 mSubLayer=0 mAnimLayer=161000+0=161000 mLastLayer=161000
  Window #3 Window{fe4aae2 u0 KeyguardScrim}:
    mBaseLayer=141000 mSubLayer=0 mAnimLayer=141000+0=141000 mLastLayer=141000
  Window #2 Window{173ee76 u0 DockedStackDivider}:
    mBaseLayer=21000 mSubLayer=0 mAnimLayer=21010+0=21010 mLastLayer=0
  Window #1 Window{3a83a4a u0 com.android.launcher}:
    mBaseLayer=21000 mSubLayer=0 mAnimLayer=21005+0=21005 mLastLayer=21005
  Window #0 Window{aefb7bc u0 com.android.systemui.ImageWallpaper}:
    mBaseLayer=21000 mSubLayer=0 mAnimLayer=21000+0=21000 mLastLayer=21000
```

理解了窗口的BaseLayer值和SubLayer值的计算过程之外，接下来我们就可以分析WindowManagerService类的成员函数assignLayersLocked()的实现了

### 3.2.1、WindowLayersController.assignLayersLocked()

```java
[->WindowLayersController.java]
final void assignLayersLocked(WindowList windows) {
    if (DEBUG_LAYERS) Slog.v(TAG_WM, "Assigning layers based on windows=" + windows,
            new RuntimeException("here").fillInStackTrace());
    clear();
    int curBaseLayer = 0;
    int curLayer = 0;
    boolean anyLayerChanged = false;
    for (int i = 0, windowCount = windows.size(); i < windowCount; i++) {
        final WindowState w = windows.get(i);
        boolean layerChanged = false;

        int oldLayer = w.mLayer;
        if (w.mBaseLayer == curBaseLayer || w.mIsImWindow || (i > 0 && w.mIsWallpaper)) {
            curLayer += WINDOW_LAYER_MULTIPLIER;
        } else {
            curBaseLayer = curLayer = w.mBaseLayer;
        }
        assignAnimLayer(w, curLayer);

        // TODO: Preserved old behavior of code here but not sure comparing
        // oldLayer to mAnimLayer and mLayer makes sense...though the
        // worst case would be unintentionalp layer reassignment.
        if (w.mLayer != oldLayer || w.mWinAnimator.mAnimLayer != oldLayer) {
            layerChanged = true;
            anyLayerChanged = true;
        }

        if (w.mAppToken != null) {
            mHighestApplicationLayer = Math.max(mHighestApplicationLayer,
                    w.mWinAnimator.mAnimLayer);
        }
        collectSpecialWindows(w);

        if (layerChanged) {
            w.scheduleAnimationIfDimming();
        }
    }

    adjustSpecialWindows();
    ......
}
```

调用 assignAnimLayer() 进行Layer调整：

```java
[->WindowLayersController.java]
  private void assignAnimLayer(WindowState w, int layer) {
        w.mLayer = layer;
        w.mWinAnimator.mAnimLayer = w.mLayer + w.getAnimLayerAdjustment() +
                    getSpecialWindowAnimLayerAdjustment(w);
        if (w.mAppToken != null && w.mAppToken.mAppAnimator.thumbnailForceAboveLayer > 0
                && w.mWinAnimator.mAnimLayer > w.mAppToken.mAppAnimator.thumbnailForceAboveLayer) {
            w.mAppToken.mAppAnimator.thumbnailForceAboveLayer = w.mWinAnimator.mAnimLayer;
        }
    }
```

### 3.3、设置窗口的Z轴位置到SurfaceFlinger服务中去

WindowManagerService服务在刷新系统的UI的时候，就会将系统中已经计算好了的窗口Z轴位置设置到SurfaceFlinger服务中去，以便SurfaceFlinger服务可以对系统中的窗口进行可见性计算以及合成和渲染等操作 首先看一下堆栈信息：

```java
WindowSurfaceController.showSurface()添加打印Log
Slog.i("charlesvincent", "charlesvincent",new RuntimeException("here").fillInStackTrace());
03-01 19:06:45.229: I/charlesvincent(1424): charlesvincent
03-01 19:06:45.229: I/charlesvincent(1424): java.lang.RuntimeException: here
03-01 19:06:45.229: I/charlesvincent(1424):     at com.android.server.wm.WindowSurfaceController.showSurface(WindowSurfaceController.java:414)
03-01 19:06:45.229: I/charlesvincent(1424):     at com.android.server.wm.WindowSurfaceController.updateVisibility(WindowSurfaceController.java:402)
03-01 19:06:45.229: I/charlesvincent(1424):     at com.android.server.wm.WindowSurfaceController.showRobustlyInTransaction(WindowSurfaceController.java:391)
03-01 19:06:45.229: I/charlesvincent(1424):     at com.android.server.wm.WindowStateAnimator.showSurfaceRobustlyLocked(WindowStateAnimator.java:1814)
03-01 19:06:45.229: I/charlesvincent(1424):     at com.android.server.wm.WindowStateAnimator.prepareSurfaceLocked(WindowStateAnimator.java:1609)
03-01 19:06:45.229: I/charlesvincent(1424):     at com.android.server.wm.WindowAnimator.animateLocked(WindowAnimator.java:791)
03-01 19:06:45.229: I/charlesvincent(1424):     at com.android.server.wm.WindowAnimator.-wrap0(WindowAnimator.java)
03-01 19:06:45.229: I/charlesvincent(1424):     at com.android.server.wm.WindowAnimator$1.doFrame(WindowAnimator.java:166)
03-01 19:06:45.229: I/charlesvincent(1424):     at android.view.Choreographer$CallbackRecord.run(Choreographer.java:879)
03-01 19:06:45.229: I/charlesvincent(1424):     at android.view.Choreographer.doCallbacks(Choreographer.java:693)
03-01 19:06:45.229: I/charlesvincent(1424):     at android.view.Choreographer.doFrame(Choreographer.java:625)
03-01 19:06:45.229: I/charlesvincent(1424):     at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:867)
03-01 19:06:45.229: I/charlesvincent(1424):     at android.os.Handler.handleCallback(Handler.java:751)
03-01 19:06:45.229: I/charlesvincent(1424):     at android.os.Handler.dispatchMessage(Handler.java:95)
03-01 19:06:45.229: I/charlesvincent(1424):     at android.os.Looper.loop(Looper.java:154)
03-01 19:06:45.229: I/charlesvincent(1424):     at android.os.HandlerThread.run(HandlerThread.java:61)
03-01 19:06:45.229: I/charlesvincent(1424):     at com.android.server.ServiceThread.run(ServiceThread.java:46)
```

为了了解WMS是如何将Z轴位置设置到SurfaceFlinger服务中去，首先看一下WMS构造方法中关键对象WindowAnimator的创建

### 3.3.1、Vsync刷新UI回调过程

开机启动时会初始化WMS。

```java
[->WindowManagerService.java]
private WindowManagerService(Context context, InputManagerService inputManager,
        boolean haveInputMethods, boolean showBootMsgs, boolean onlyCore) {
        ......
        mAnimator = new WindowAnimator(this);
        ......
        }
```

调用WindowAnimator构造方法

```java
[->WindowAnimator.java]
WindowAnimator(final WindowManagerService service) {
    mService = service;
    mContext = service.mContext;
    mPolicy = service.mPolicy;
    mWindowPlacerLocked = service.mWindowPlacerLocked;

    mAnimationFrameCallback = new Choreographer.FrameCallback() {
        public void doFrame(long frameTimeNs) {
            synchronized (mService.mWindowMap) {
                mService.mAnimationScheduled = false;
                animateLocked(frameTimeNs);
            }
        }
    };
}
```

可以看到创建了Choreographer.FrameCallback()，前面在【Android-7-1-2-Android-N-Activity-Window加载显示流程】分析过，FrameDisplayEventReceiver（在Choreographer构造方法中初始化）对象用于请求并接收Vsync信号，当Vsync信号到来时，系统会自动调用其onVsync()函数，后面会回调到FrameDisplayEventReceiver.run()方法，再回调函数中执行doFrame()实现屏幕刷新，doFrame()会顺序执行CALLBACK_INPUT、CALLBACK_ANIMATION、CALLBACK_TRAVERSAL 和CALLBACK_COMMIT 对应CallbackQueue队列中注册的回调，从而会执行CallbackRecord.run()，在执行其回调函数时，就需要区别这两种对象类型，如果注册的是Runnable对象，则调用其run()函数，如果注册的是FrameCallback对象，则调用它的doFrame()函数。

```java
[->Choreographer.java]
private static final class CallbackRecord {
    public CallbackRecord next;
    public long dueTime;
    public Object action; // Runnable or FrameCallback
    public Object token;

    public void run(long frameTimeNanos) {
        if (token == FRAME_CALLBACK_TOKEN) {
            ((FrameCallback)action).doFrame(frameTimeNanos);
        } else {
            ((Runnable)action).run();
        }
    }
}
```

在此种情况下会执行FrameCallback对象的doFrame()函数（原因稍后再分析动画时详细分析），由WindowAnimator构造函数中可知接着就会执行WindowAnimator.animateLocked()

### 3.3.2、准备刷新UI

```java
[->WindowAnimator.java]
    private void animateLocked(long frameTimeNs) {
        ......
        if (SHOW_TRANSACTIONS) Slog.i(
                TAG, ">>> OPEN TRANSACTION animateLocked");
        SurfaceControl.openTransaction();
        SurfaceControl.setAnimationTransaction();
        try {
            final int numDisplays = mDisplayContentsAnimators.size();
            for (int i = 0; i < numDisplays; i++) {
                final int displayId = mDisplayContentsAnimators.keyAt(i);
                updateAppWindowsLocked(displayId);
                DisplayContentsAnimator displayAnimator = mDisplayContentsAnimators.valueAt(i);
                ......
                updateWindowsLocked(displayId);
                updateWallpaperLocked(displayId);

                final WindowList windows = mService.getWindowListLocked(displayId);
                final int N = windows.size();
                //通过一个for循环来遍历保存在窗口堆栈的每一个WindowState对象，以便可以对系统中的每一个窗口的绘图表面进行更新
                for (int j = 0; j < N; j++) {
                    windows.get(j).mWinAnimator.prepareSurfaceLocked(true);
                }
            }
            ......
        catch (RuntimeException e) {
            ......
        } finally {
            SurfaceControl.closeTransaction();
            if (SHOW_TRANSACTIONS) Slog.i(
                    TAG, "<<< CLOSE TRANSACTION animateLocked");
        }
}
```

首先获取windows列表，然后循环调用windows.get(j).mWinAnimator.prepareSurfaceLocked(true)。

```java
[->WindowStateAnimator.java]
void prepareSurfaceLocked(final boolean recoveringMemory) {
    final WindowState w = mWin;
    ......

    boolean displayed = false;
    //确定该窗口实际要显示的大小、位置、Alpha通道和变换矩阵等信息
    computeShownFrameLocked();

    setSurfaceBoundariesLocked(recoveringMemory);

    if (mIsWallpaper && !mWin.mWallpaperVisible) {
        ......
    } else if (w.mAttachedHidden || !w.isOnScreen()) {
        ......
    } else if (mLastLayer != mAnimLayer
            || mLastAlpha != mShownAlpha
            || mLastDsDx != mDsDx
            || mLastDtDx != mDtDx
            || mLastDsDy != mDsDy
            || mLastDtDy != mDtDy
            || w.mLastHScale != w.mHScale
            || w.mLastVScale != w.mVScale
            || mLastHidden) {
        displayed = true;
        mLastAlpha = mShownAlpha;
        mLastLayer = mAnimLayer;
        mLastDsDx = mDsDx;
        mLastDtDx = mDtDx;
        mLastDsDy = mDsDy;
        mLastDtDy = mDtDy;
        w.mLastHScale = w.mHScale;
        w.mLastVScale = w.mVScale;
        ......
        boolean prepared =
            mSurfaceController.prepareToShowInTransaction(mShownAlpha, mAnimLayer,
                    mDsDx * w.mHScale * mExtraHScale,
                    mDtDx * w.mVScale * mExtraVScale,
                    mDsDy * w.mHScale * mExtraHScale,
                    mDtDy * w.mVScale * mExtraVScale,
                    recoveringMemory);

        if (prepared && mLastHidden && mDrawState == HAS_DRAWN) {
            if (showSurfaceRobustlyLocked()) {
                markPreservedSurfaceForDestroy();
                mAnimator.requestRemovalOfReplacedWindows(w);
                mLastHidden = false;
                if (mIsWallpaper) {
                    mWallpaperControllerLocked.dispatchWallpaperVisibility(w, true);
                }
                ......
                mAnimator.setPendingLayoutChanges(w.getDisplayId(),
                        WindowManagerPolicy.FINISH_LAYOUT_REDO_ANIM);
            } else {
                w.mOrientationChanging = false;
            }
        }
        if (hasSurface()) {
            w.mToken.hasVisible = true;
        }
    } else {
        displayed = true;
    }

    ......
}
```

调用prepareToShowInTransaction()将alph、alayer、setMatrix设置到mSurfaceControl中。

```java
 boolean prepareToShowInTransaction(float alpha, int layer, float dsdx, float dtdx, float dsdy,
            float dtdy, boolean recoveringMemory) {
        if (mSurfaceControl != null) {
            try {
                mSurfaceAlpha = alpha;
                mSurfaceControl.setAlpha(alpha);
                mSurfaceLayer = layer;
                mSurfaceControl.setLayer(layer);
                mSurfaceControl.setMatrix(
                        dsdx, dtdx, dsdy, dtdy);

            } catch (RuntimeException e) {
                .......
            }
        }
        return true;
    }
```

setSurfaceBoundariesLocked()方法中会调用SurfaceControl.setPosition()等等方法将计算好的数值设置到mSurfaceControl中。 说明：一个窗口的显示和隐藏，以及大小、X轴和Y轴位置、Z轴位置、Alpha通道和变换矩阵设置，是通过调用Java层的SurfaceControl类的成员函数show、hide、setSize、setPosition、setLayer、setAlpha和setMatrix来实现的，它们最终都是通过调用JNI方法实现的

```java
SurfaceControl.java
......
public void setAlpha(float alpha) {
    checkNotReleased();
    nativeSetAlpha(mNativeObject, alpha);
}

public void setMatrix(float dsdx, float dtdx, float dsdy, float dtdy) {
    checkNotReleased();
    nativeSetMatrix(mNativeObject, dsdx, dtdx, dsdy, dtdy);
}

public void setWindowCrop(Rect crop) {
    checkNotReleased();
    if (crop != null) {
        nativeSetWindowCrop(mNativeObject,
            crop.left, crop.top, crop.right, crop.bottom);
    } else {
        nativeSetWindowCrop(mNativeObject, 0, 0, 0, 0);
    }
}

public void setFinalCrop(Rect crop) {
    checkNotReleased();
    if (crop != null) {
        nativeSetFinalCrop(mNativeObject,
            crop.left, crop.top, crop.right, crop.bottom);
    } else {
        nativeSetFinalCrop(mNativeObject, 0, 0, 0, 0);
    }
}
....
```

### 3.3.3、告知SurfaceFlinger显示UI

如果WindowState对象w所描述的窗口满足条件：prepared && mLastHidden && mDrawState == HAS_DRAWN 那么就说明现在是时候要将WindowState对象w所描述的窗口显示出来了，通过调用showSurfaceRobustlyLocked实现

```java
[->WindowStateAnimator.java]
private boolean showSurfaceRobustlyLocked() {
    final Task task = mWin.getTask();
    if (task != null && StackId.windowsAreScaleable(task.mStack.mStackId)) {
        mSurfaceController.forceScaleableInTransaction(true);
    }
    boolean shown = mSurfaceController.showRobustlyInTransaction();
    ......
    if (mWin.mTurnOnScreen) {
        ......
        mWin.mTurnOnScreen = false;
        mAnimator.mBulkUpdateParams |= SET_TURN_ON_SCREEN;
    }
    return true;
}
```

直接调用WindowSurfaceController.showRobustlyInTransaction() --> updateVisibility()-->showSurface()->SurfaceControl.show()

```java
[->WindowSurfaceController.java]
    boolean showRobustlyInTransaction() {
        ......
        mHiddenForOtherReasons = false;
        return updateVisibility();
    }
   private boolean updateVisibility() {
        if (mHiddenForCrop || mHiddenForOtherReasons) {
            if (mSurfaceShown) {hideSurface();}
            return false;
        } else {
            if (!mSurfaceShown) {return showSurface();
            } else {
                return true;
            }
        }
    }
    private boolean showSurface() {
        try {
            mSurfaceShown = true;
            mSurfaceControl.show();
             Slog.i("charlesvincent", "charlesvincent",new RuntimeException("here").fillInStackTrace());
            return true;
        } catch (RuntimeException e) {
            ......
        }
        //出现异常,回收系统内存资源
        mAnimator.reclaimSomeSurfaceMemory("show", true);
        return false;
    }
```

```java
[->SurfaceControl.java]
public void show() {
    checkNotReleased();
    nativeSetFlags(mNativeObject, 0, SURFACE_HIDDEN);
}
```

通过JNI调用android_view_SurfaceControl.cpp的nativeSetFlags函数，可以看到flags == 0；

```c
[->android_view_SurfaceControl.cpp]
static void nativeSetFlags(JNIEnv* env, jclass clazz, jlong nativeObject, jint flags, jint mask) {
    SurfaceControl* const ctrl = reinterpret_cast<SurfaceControl *>(nativeObject);
    status_t err = ctrl->setFlags(flags, mask);
    if (err < 0 && err != NO_INIT) {
        doThrowIAE(env);
    }
}
```

调用SurfaceControl.cpp的setFlags()函数

```c
[->SurfaceControl.cpp]
status_t SurfaceControl::setFlags(uint32_t flags, uint32_t mask) {
    status_t err = validate();
    if (err < 0) return err;
    return mClient->setFlags(mHandle, flags, mask);
}
```

进一步通过Binder IPC机制，SurfaceComposerClient.cpp->Composer::setFlags()

```c
[->SurfaceComposerClient.cpp]
status_t Composer::setFlags(const sp<SurfaceComposerClient>& client,
        const sp<IBinder>& id, uint32_t flags,
        uint32_t mask) {
    Mutex::Autolock _l(mLock);
    layer_state_t* s = getLayerStateLocked(client, id);
    ......
    if ((mask & layer_state_t::eLayerOpaque) ||
            (mask & layer_state_t::eLayerHidden) ||
            (mask & layer_state_t::eLayerSecure)) {
        s->what |= layer_state_t::eFlagsChanged;
    }
    s->flags &= ~mask;
    s->flags |= (flags & mask);
    s->mask |= mask;
    return NO_ERROR;
}
```

具体数值就不详细计算了，前面分析【Android 7.1.2 (Android N) Android Graphics 系统分析 [i.wonder~]】可知，SurfaceFlinger接收Vsync信号与App有一个offset间隔时间，当SurfaceFlinger接收Vsync信号时，就可以根据flags是否显示 和 上面设置的一系列数值进行渲染合成，最终显示到屏幕上。

## （四）、Activity启动窗口(Starting Window)添加过程

### 4.1、Activity组件的启动窗口(Starting Window)的添加过程

时序图： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/android.wms/13-Android-WMS-Starting-Window-addview.png)

### 4.1.1、 ActivityStack.startActivityLocked()

```java
[->ActivityStack.java]
   final void startActivityLocked(ActivityRecord r, boolean newTask, boolean keepCurTransition,
            ActivityOptions options) {
            ......
                    if (!isHomeStack() || numActivities() > 0) {
            ......
            if ((r.intent.getFlags() & Intent.FLAG_ACTIVITY_NO_ANIMATION) != 0) {
                mWindowManager.prepareAppTransition(TRANSIT_NONE, keepCurTransition);
                mNoAnimActivities.add(r);
            } else {
                mWindowManager.prepareAppTransition(newTask
                        ? r.mLaunchTaskBehind
                                ? TRANSIT_TASK_OPEN_BEHIND
                                : TRANSIT_TASK_OPEN
                        : TRANSIT_ACTIVITY_OPEN, keepCurTransition);
                mNoAnimActivities.remove(r);
            }
            addConfigOverride(r, task);
            boolean doShow = true;
            ......
            if (r.mLaunchTaskBehind) {
                .......
            } else if (SHOW_APP_STARTING_PREVIEW && doShow) {
                ActivityRecord prev = r.task.topRunningActivityWithStartingWindowLocked();
                if (prev != null) {
                    // We don't want to reuse the previous starting preview if:
                    // (1) The current activity is in a different task.
                    if (prev.task != r.task) {
                        prev = null;
                    }
                    // (2) The current activity is already displayed.
                    else if (prev.nowVisible) {
                        prev = null;
                    }
                }
                r.showStartingWindow(prev, showStartingIcon);
            }
        } ......
    }
```

可以看到直接调用ActivityRecord.showStartingWindow()进一步添加启动窗口

### 4.1.2、 ActivityRecord.showStartingWindow()

```java
[->ActivityRecord.java]
void showStartingWindow(ActivityRecord prev, boolean createIfNeeded) {
    final CompatibilityInfo compatInfo =
            service.compatibilityInfoForPackageLocked(info.applicationInfo);
    final boolean shown = service.mWindowManager.setAppStartingWindow(
            appToken, packageName, theme, compatInfo, nonLocalizedLabel, labelRes, icon,
            logo, windowFlags, prev != null ? prev.appToken : null, createIfNeeded);
    if (shown) {
        mStartingWindowState = STARTING_WINDOW_SHOWN;
    }
}
```

### 4.1.2、ActivityRecord.setAppStartingWindow()

```java
[->ActivityRecord.java]
public boolean setAppStartingWindow(IBinder token, String pkg,
        int theme, CompatibilityInfo compatInfo,
        CharSequence nonLocalizedLabel, int labelRes, int icon, int logo,
        int windowFlags, IBinder transferFrom, boolean createIfNeeded) {
   ......
  synchronized(mWindowMap) {
   AppWindowToken wtoken = findAppWindowToken(token);
        ......
        if (transferStartingWindow(transferFrom, wtoken)) {
            return true;
        }
        ......
        wtoken.startingData = new StartingData(pkg, theme, compatInfo, nonLocalizedLabel,
                labelRes, icon, logo, windowFlags);
        Message m = mH.obtainMessage(H.ADD_STARTING, wtoken);
        mH.sendMessageAtFrontOfQueue(m);
    }
    return true;
}
```

如果参数transferFrom所描述的Activity组件没有启动窗口或者启动窗口数据转移给参数token所描述的Activity组件，那么接下来就可能需要为参数token所描述的Activity组件创建一个新的启动窗口

### 4.1.3、 H.handleMessage()

```java
[->WindowManagerService.java::H]
case ADD_STARTING: {
    final AppWindowToken wtoken = (AppWindowToken)msg.obj;
    final StartingData sd = wtoken.startingData;
    ......
    View view = null;
    try {
        final Configuration overrideConfig = wtoken != null && wtoken.mTask != null
                ? wtoken.mTask.mOverrideConfig : null;
        view = mPolicy.addStartingWindow(wtoken.token, sd.pkg, sd.theme,
            sd.compatInfo, sd.nonLocalizedLabel, sd.labelRes, sd.icon, sd.logo,
            sd.windowFlags, overrideConfig);
    } catch (Exception e) {

    }

    if (view != null) {
        boolean abort = false;
        synchronized(mWindowMap) {
            if (wtoken.removed || wtoken.startingData == null) {
                    wtoken.startingWindow = null;
                    wtoken.startingData = null;
                    abort = true;
                }
            } else {
                wtoken.startingView = view;
            }
        }
        if (abort) {
            try {
                mPolicy.removeStartingWindow(wtoken.token, view);
            } catch (Exception e) {

            }
        }
    }
} break;
```

PhoneWindowManager实现WindowManagerPolicy，所以会调用PhoneWindowManager中的方法 继续分析PhoneWindowManager.addStartingWindow()

### 4.1.4、PhoneWindowManager.addStartingWindow()

```java
[->PhoneWindowManager.java]
public View addStartingWindow(IBinder appToken, String packageName, int theme,
        CompatibilityInfo compatInfo, CharSequence nonLocalizedLabel, int labelRes,
        int icon, int logo, int windowFlags, Configuration overrideConfig) {
    ......
    WindowManager wm = null;
    View view = null;
    try {
        Context context = mContext;
        ......
        final PhoneWindow win = new PhoneWindow(context);
        win.setIsStartingWindow(true);
        ......
        win.setType(
            WindowManager.LayoutParams.TYPE_APPLICATION_STARTING);
        ......
        win.setLayout(WindowManager.LayoutParams.MATCH_PARENT,
                WindowManager.LayoutParams.MATCH_PARENT);

        final WindowManager.LayoutParams params = win.getAttributes();
        params.token = appToken;
        params.packageName = packageName;
        params.windowAnimations = win.getWindowStyle().getResourceId(
                com.android.internal.R.styleable.Window_windowAnimationStyle, 0);
        ......
        params.setTitle("Starting " + packageName);
        wm = (WindowManager)context.getSystemService(Context.WINDOW_SERVICE);
        view = win.getDecorView();
       ......
        wm.addView(view, params);
        return view.getParent() != null ? view : null;
    } catch (WindowManager.BadTokenException e) {
        ......
    } catch (RuntimeException e) {
        ......
    } finally {
        if (view != null && view.getParent() == null) {
            wm.removeViewImmediate(view);
        }
    }

    return null;
}
```

创建PhoneWindow对象，接下来继续设置所创建的窗口win的以下属性： 1、窗口类型：设置为WindowManager.LayoutParams.TYPE_APPLICATION_STARTING，即设置为启动窗口类型；

2、窗口标题：由参数labelRes、nonLocalizedLabel，以及窗口的运行上下文context来确定；

3、窗口标志：分别将indowManager.LayoutParams.FLAG_NOT_TOUCHABLE、WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE和WindowManager.LayoutParams.FLAG_ALT_FOCUSABLE_IM位设置为1，即不可接受触摸事件和不可获得焦点，但是可以接受输入法窗口；

4、窗口大小：设置为WindowManager.LayoutParams.MATCH_PARENT，即与父窗口一样大，但是由于这是一个顶层窗口，因此实际上是指与屏幕一样大；

5、布局参数：包括窗口所对应的窗口令牌（token）和包名（packageName），以及窗口所使用的动画类型（windowAnimations）和标题（title）。

wm.addView(view, params)，一个新创建的Activity组件的启动窗口就增加到WindowManagerService服务中去了，这样，WindowManagerService服务就可以下次刷新系统UI时，将该启动窗口显示出来

## （五）、WMS切换Activity窗口（App Transition）过程

WindowManagerService服务在执行Activity窗口的切换操作的时候，会给参与切换操作的Activity组件的设置一个动画，以便可以向用户展现一个Activity组件切换效果，从而提高用户体验。 首先看一下App Transition动态图： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/android.wms/14-Android-WMS-ezgif.com-video-to-gif-WMS-App-Transition.gif)

时序图： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/android.wms/15-Android-WMS-executeAppTransition.png)

我们直接分析App Transition过程的prepareAppTransition、executeAppTransition 关于Activity启动过程请参考：【Android-7-1-2-Android-N-Activity启动流程分析】

### 5.1、prepareAppTransition()过程

### 5.1.1、ActivityStack.startActivityLocked()

```java
[->ActivityStack.java]
......
mWindowManager.prepareAppTransition(newTask
        ? r.mLaunchTaskBehind
                ? TRANSIT_TASK_OPEN_BEHIND
                : TRANSIT_TASK_OPEN
        : TRANSIT_ACTIVITY_OPEN, keepCurTransition);
......
```

### 5.1.2、WindowManagerService.prepareAppTransition()

```java
[->WindowManagerService.java]
@Override
public void prepareAppTransition(int transit, boolean alwaysKeepCurrent) {
    ......
    synchronized(mWindowMap) {
        boolean prepared = mAppTransition.prepareAppTransitionLocked(
                transit, alwaysKeepCurrent);
        if (prepared && okToDisplay()) {
            mSkipAppTransitionAnimation = false;
        }
    }
}
```

发现是直接调用AppTransition.prepareAppTransitionLocked()实现的。

### 5.1.3、AppTransition.prepareAppTransitionLocked()

进一步调用setAppTransition()

```java

private void setAppTransition(int transit) {
    mNextAppTransition = transit;
    setLastAppTransition(TRANSIT_UNSET, null, null);
}
```

发现只是将transit（即AppTransition动画类型）赋值给变量mNextAppTransition

### 5.2、AppTransition animation设置过程

继续分析ActivityStackSupervisor.realStartActivityLocked()

```java
[->ActivityStackSupervisor.java]
final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
        boolean andResume, boolean checkConfig) throws RemoteException {

  ......
    if (andResume) {
        r.startFreezingScreenLocked(app, 0);
        mWindowManager.setAppVisibility(r.appToken, true);

        // schedule launch ticks to collect information about slow apps.
        r.startLaunchTickingLocked();
    }
```

首先会通知WindowManagerService服务将参数r.appToken所描述的Activity组件的可见性设置为true

```java
[->WindowManagerService.java]
@Override
public void setAppVisibility(IBinder token, boolean visible) {
    ......
    AppWindowToken wtoken;

    synchronized(mWindowMap) {
        wtoken = findAppWindowToken(token);
        ......
        mOpeningApps.remove(wtoken);
        mClosingApps.remove(wtoken);
        ......
            final long origId = Binder.clearCallingIdentity();
            wtoken.inPendingTransaction = false;
            setTokenVisibilityLocked(wtoken, null, visible, AppTransition.TRANSIT_UNSET,
                    true, wtoken.voiceInteraction);
            wtoken.updateReportedVisibilityLocked();
        }
    }
```

调用WindowManagerService.setTokenVisibilityLocked()

```java
[->WindowManagerService.java]
    boolean setTokenVisibilityLocked(AppWindowToken wtoken, WindowManager.LayoutParams lp,
            boolean visible, int transit, boolean performLayout, boolean isVoiceInteraction) {
        ......
        if (wtoken.hidden == visible || (wtoken.hidden && wtoken.mIsExiting) ||
                (visible && wtoken.waitingForReplacement())) {
           ......
            if (transit != AppTransition.TRANSIT_UNSET) {
                ......
                if (applyAnimationLocked(wtoken, lp, transit, visible, isVoiceInteraction)) {
                    delayed = runningAppAnimation = true;
                }
                ......
                changed = true;
            }
         }
    }
```

调用WindowManagerService.applyAnimationLocked()设置AppTransition动画

```java
[->WindowManagerService.java]
    private boolean applyAnimationLocked(AppWindowToken atoken, WindowManager.LayoutParams lp,
            int transit, boolean enter, boolean isVoiceInteraction) {
        ......
        if (okToDisplay()) {
        Animation a = mAppTransition.loadAnimation(lp, transit, enter, mCurConfiguration.uiMode,
        mCurConfiguration.orientation, frame, displayFrame, insets, surfaceInsets,
        isVoiceInteraction, freeform, atoken.mTask.mTaskId);
        }
}
```

终于到了AppTransition真正设置过程了

```java
[->AppTransition.java]
Animation loadAnimation(WindowManager.LayoutParams lp, int transit, boolean enter, int uiMode,
            int orientation, Rect frame, Rect displayFrame, Rect insets,
            @Nullable Rect surfaceInsets, boolean isVoiceInteraction, boolean freeform,
            int taskId) {
        Animation a;
      if(){}
      ......
      } else if(){
      ......
      }else {
        int animAttr = 0;
        switch (transit) {
            case TRANSIT_ACTIVITY_OPEN:
                animAttr = enter
                        ? WindowAnimation_activityOpenEnterAnimation
                        : WindowAnimation_activityOpenExitAnimation;
                break;
            case TRANSIT_ACTIVITY_CLOSE:
                animAttr = enter
                        ? WindowAnimation_activityCloseEnterAnimation
                        : WindowAnimation_activityCloseExitAnimation;
                break;
            case TRANSIT_DOCK_TASK_FROM_RECENTS:
            case TRANSIT_TASK_OPEN:
                animAttr = enter
                        ? WindowAnimation_taskOpenEnterAnimation
                        : WindowAnimation_taskOpenExitAnimation;
                break;
            case TRANSIT_TASK_CLOSE:
                animAttr = enter
                        ? WindowAnimation_taskCloseEnterAnimation
                        : WindowAnimation_taskCloseExitAnimation;
                break;
            case TRANSIT_TASK_TO_FRONT:
                animAttr = enter
                        ? WindowAnimation_taskToFrontEnterAnimation
                        : WindowAnimation_taskToFrontExitAnimation;
                break;
            case TRANSIT_TASK_TO_BACK:
                animAttr = enter
                        ? WindowAnimation_taskToBackEnterAnimation
                        : WindowAnimation_taskToBackExitAnimation;
                break;
            ......
        }
        a = animAttr != 0 ? loadAnimationAttr(lp, animAttr) : null;
    }
    return a;
}
```

最后通过**loadAnimationAttr**加载xml文件加载动画，动画xml文件的存放路径（/frameworks/base/core/res/res/anim/） 
![Markdown](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/android.wms/16-Android-WMS-executeAppTransition-xml.png)

### 5.3、executeAppTransition过程

> ActivityStackSupervisor.realStartActivityLocked()

> > ActivityStack.minimalResumeActivityLocked() ActivityStack.completeResumeLocked() ActivityStackSupervisor.reportResumedActivityLocked()

继续分析ActivityStackSupervisor.reportResumedActivityLocked()

```java
[->ActivityStackSupervisor.java]
boolean reportResumedActivityLocked(ActivityRecord r) {
    final ActivityStack stack = r.task.stack;
    ......
    if (allResumedActivitiesComplete()) {
        ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
        mWindowManager.executeAppTransition();
        return true;
    }
    return false;
}
```

首先是Activity组件的可见性设置，然后执行executeAppTransition()

### 5.3.1.WindowManagerService.executeAppTransition()

```java
[->WindowManagerService.java]
    public void executeAppTransition() {
       ......
        synchronized(mWindowMap) {
            if (mAppTransition.isTransitionSet()) {
                mAppTransition.setReady();
                ......
                try {
                    mWindowPlacerLocked.performSurfacePlacement();
                } finally {
                    ......
                }
            }
        }
    }
```

而WindowSurfacePlacer.performSurfacePlacement()请看前面第二节分析

## （六）、Activity Window 动画显示过程

### 6.1、动画的设置过程

在Android系统中，窗口动画的本质就是对原始窗口施加一个变换（Transformation）。在线性数学中，对物体的形状进行变换是通过乘以一个矩阵（Matrix）来实现，目的就是对物体进行偏移、旋转、缩放、切变、反射和投影等。因此，给窗口设置动画实际上就给窗口设置一个变换矩阵（Transformation Matrix）。 
![Markdown](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/android.wms/17-Android-WMS-Transformation-matrix.png)

窗口被设置的动画虽然可以达到三个，但是这三个动画可以归结为两类，一类是普通动画，例如，窗口在打开过程中被设置的进入动画和在关闭过程中被设置的退出动画，另一类是切换动画。其中，Self Transformation和Attached Transformation都是属于普通动画，而App Transformation属于切换动画。前面已经分析过App Transformation的设置过程 接下来分析普通动画的设置过程。

### 6.1、普通动画的设置过程

普通动画的设置过程也是通过setTokenVisibilityLocked()设置的

### 6.1.1、WindowManagerService.setTokenVisibilityLocked()

```java
[->WindowManagerService.java]
    boolean setTokenVisibilityLocked(AppWindowToken wtoken, WindowManager.LayoutParams lp,
            boolean visible, int transit, boolean performLayout, boolean isVoiceInteraction) {
        ......
        if (wtoken.hidden == visible || (wtoken.hidden && wtoken.mIsExiting) ||
                (visible && wtoken.waitingForReplacement())) {
           ......
            if (transit != AppTransition.TRANSIT_UNSET) {
                ......
                if (applyAnimationLocked(wtoken, lp, transit, visible, isVoiceInteraction)) {
                    delayed = runningAppAnimation = true;
                }
                ......
                changed = true;
            }
            final int windowsCount = wtoken.allAppWindows.size();
            for (int i = 0; i < windowsCount; i++) {
                WindowState win = wtoken.allAppWindows.get(i);
                ......
                if (visible) {
                    if (!win.isVisibleNow()) {
                        if (!runningAppAnimation) {
                            win.mWinAnimator.applyAnimationLocked(
                                    WindowManagerPolicy.TRANSIT_ENTER, true);
                            ......
                        }
                        changed = true;
                        win.setDisplayLayoutNeeded();
                    }
                } else if (win.isVisibleNow()) {
                    if (!runningAppAnimation) {
                        win.mWinAnimator.applyAnimationLocked(
                                WindowManagerPolicy.TRANSIT_EXIT, false);
                        ......
                    }
                    changed = true;
                    win.setDisplayLayoutNeeded();
                }
            }

            wtoken.hidden = wtoken.hiddenRequested = !visible;
            visibilityChanged = true;
            ......
            if (changed) {
                mInputMonitor.setUpdateInputWindowsNeededLw();
                if (performLayout) {
                    updateFocusedWindowLocked(UPDATE_FOCUS_WILL_PLACE_SURFACES,
                            false /*updateInputWindows*/);
                    mWindowPlacerLocked.performSurfacePlacement();
                }
                mInputMonitor.updateInputWindowsLw(false /*force*/);
            }
         }
    }
```

可以看到普通动画是通过win.mWinAnimator.applyAnimationLocked(WindowManagerPolicy.TRANSIT_ENTER, true)

### 6.1.1.WindowStateAnimator.applyAnimationLocked()

```java
[->WindowStateAnimator.java]
boolean applyAnimationLocked(int transit, boolean isEntrance) {
    ......
    if (mService.okToDisplay()) {
        int anim = mPolicy.selectAnimationLw(mWin, transit);
        int attr = -1;
        Animation a = null;
        if (anim != 0) {
            a = anim != -1 ? AnimationUtils.loadAnimation(mContext, anim) : null;
        } else {
            switch (transit) {
                case WindowManagerPolicy.TRANSIT_ENTER:
                    attr = com.android.internal.R.styleable.WindowAnimation_windowEnterAnimation;
                    break;
                case WindowManagerPolicy.TRANSIT_EXIT:
                    attr = com.android.internal.R.styleable.WindowAnimation_windowExitAnimation;
                    break;
                case WindowManagerPolicy.TRANSIT_SHOW:
                    attr = com.android.internal.R.styleable.WindowAnimation_windowShowAnimation;
                    break;
                case WindowManagerPolicy.TRANSIT_HIDE:
                    attr = com.android.internal.R.styleable.WindowAnimation_windowHideAnimation;
                    break;
            }
            if (attr >= 0) {
                a = mService.mAppTransition.loadAnimationAttr(mWin.mAttrs, attr);
            }
        }
        if (a != null) {
            setAnimation(a);
            mAnimationIsEntrance = isEntrance;
        }
    } else {
        clearAnimation();
    }
    ......
    return mAnimation != null;
}
```

可以看到也是根据动画类型从而通过AppTransition.loadAnimationAttr(mWin.mAttrs, attr)加载不同的anim xml文件。

### 6.2、窗口动画的显示框架

![Markdown](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/android.wms/19-Android-WMS-animationLocked.png)
通过堆栈信息可以看到，由Vsync信号驱动，然后调用Choreographer.doFrame完成动画的相关操作，关于Vsync这部分之前文章已经分析过，这里不再分析了。

```java
[->WindowAnimator.java]
/** Locked on mService.mWindowMap. */
private void animateLocked(long frameTimeNs) {
    ......
    mCurrentTime = frameTimeNs / TimeUtils.NANOS_PER_MS;
    mBulkUpdateParams = SET_ORIENTATION_CHANGE_COMPLETE;
    boolean wasAnimating = mAnimating;
    setAnimating(false);
    mAppWindowAnimating = false;
    ......
    if (SHOW_TRANSACTIONS) Slog.i(
            TAG, ">>> OPEN TRANSACTION animateLocked");
    SurfaceControl.openTransaction();
    SurfaceControl.setAnimationTransaction();
    try {
        final int numDisplays = mDisplayContentsAnimators.size();
        for (int i = 0; i < numDisplays; i++) {
            final int displayId = mDisplayContentsAnimators.keyAt(i);
            //1、Activity组件切换动画的推进过程
            updateAppWindowsLocked(displayId);
            DisplayContentsAnimator displayAnimator = mDisplayContentsAnimators.valueAt(i);

            final ScreenRotationAnimation screenRotationAnimation =
                    displayAnimator.mScreenRotationAnimation;
            if (screenRotationAnimation != null && screenRotationAnimation.isAnimating()) {
                if (screenRotationAnimation.stepAnimationLocked(mCurrentTime)) {
                    setAnimating(true);
                } else {
                    mBulkUpdateParams |= SET_UPDATE_ROTATION;
                    screenRotationAnimation.kill();
                    displayAnimator.mScreenRotationAnimation = null;

                    //TODO (multidisplay): Accessibility supported only for the default display.
                    if (mService.mAccessibilityController != null
                            && displayId == Display.DEFAULT_DISPLAY) {
                        // We just finished rotation animation which means we did not
                        // anounce the rotation and waited for it to end, announce now.
                        mService.mAccessibilityController.onRotationChangedLocked(
                                mService.getDefaultDisplayContentLocked(), mService.mRotation);
                    }
                }
            }

            // Update animations of all applications, including those
            // associated with exiting/removed apps
            //2、窗口动画的推进过程
            updateWindowsLocked(displayId);
            //壁纸动画的推进过程
            updateWallpaperLocked(displayId);

            final WindowList windows = mService.getWindowListLocked(displayId);
            final int N = windows.size();
            //3、通过一个for循环来遍历保存在窗口堆栈的每一个WindowState对象，以便可以对系统中的每一个窗口的绘图表面进行更新
            //确定该窗口实际要显示的大小、位置、Alpha通道和变换矩阵等信息
            for (int j = 0; j < N; j++) {
                windows.get(j).mWinAnimator.prepareSurfaceLocked(true);
            }
        }

        for (int i = 0; i < numDisplays; i++) {
            final int displayId = mDisplayContentsAnimators.keyAt(i);

            testTokenMayBeDrawnLocked(displayId);

            final ScreenRotationAnimation screenRotationAnimation =
                    mDisplayContentsAnimators.valueAt(i).mScreenRotationAnimation;
            if (screenRotationAnimation != null) {
                screenRotationAnimation.updateSurfacesInTransaction();
            }

            orAnimating(mService.getDisplayContentLocked(displayId).animateDimLayers());
            orAnimating(mService.getDisplayContentLocked(displayId).getDockedDividerController()
                    .animate(mCurrentTime));
            //TODO (multidisplay): Magnification is supported only for the default display.
            if (mService.mAccessibilityController != null
                    && displayId == Display.DEFAULT_DISPLAY) {
                mService.mAccessibilityController.drawMagnifiedRegionBorderIfNeededLocked();
            }
        }

        ......
        //4、触发下一帧动画逻辑
        if (mAnimating) {
            mService.scheduleAnimationLocked();
        }

        if (mService.mWatermark != null) {
            mService.mWatermark.drawIfNeeded();
        }
    } catch (RuntimeException e) {
        ......
    } finally {
        SurfaceControl.closeTransaction();
        if (SHOW_TRANSACTIONS) Slog.i(
                TAG, "<<< CLOSE TRANSACTION animateLocked");
    }

    boolean hasPendingLayoutChanges = false;
    final int numDisplays = mService.mDisplayContents.size();
    for (int displayNdx = 0; displayNdx < numDisplays; ++displayNdx) {
        final DisplayContent displayContent = mService.mDisplayContents.valueAt(displayNdx);
        final int pendingChanges = getPendingLayoutChanges(displayContent.getDisplayId());
        if ((pendingChanges & WindowManagerPolicy.FINISH_LAYOUT_REDO_WALLPAPER) != 0) {
            mBulkUpdateParams |= SET_WALLPAPER_ACTION_PENDING;
        }
        if (pendingChanges != 0) {
            hasPendingLayoutChanges = true;
        }
    }

    boolean doRequest = false;
    if (mBulkUpdateParams != 0) {
        doRequest = mWindowPlacerLocked.copyAnimToLayoutParamsLocked();
    }
    //5、刷新系统UI
    if (hasPendingLayoutChanges || doRequest) {
        mWindowPlacerLocked.requestTraversal();
    }

    ......
    if (mRemoveReplacedWindows) {
        removeReplacedWindowsLocked();
    }

    mService.stopUsingSavedSurfaceLocked();
    mService.destroyPreservedSurfaceLocked();
    mService.mWindowPlacerLocked.destroyPendingSurfaces();

    ......
}
```

1、Activity组件切换动画的推进、 2、窗口动画的推进、壁纸动画推进 3、循环遍历保存在窗口堆栈的每一个WindowState对象，以便可以对系统中的每一个窗口的绘图表面进行更新 确定该窗口实际要显示的大小、位置、Alpha通道和变换矩阵等信息 4、触发下一帧动画逻辑 5、刷新系统UI 其中第3点 prepareSurfaceLocked在3.3.小节已经分析过了、第5点最终会调用mWindowPlacerLocked.performSurfacePlacement来刷新UI，也已经分析过了。 接下来分析Activity组件切换动画、窗口动画的推进过程。

### 6.3、Activity组件切换动画

AppWindowAnimator:属于AppWindowToken，它的成员变量mAppAnimator代表了此应用程序所属的AppWindowAnimator WindowStateAnimator:WMS记录了所有窗口的WindowState，其中WindowState.mWinAnimator是一个WindowStateAnimator对象，它和上面AppWindowAnimator一样可以由开发人员定制

WindowAnimator.updateAppWindowsLocked()

```java
[->WindowAnimator.java]
private void updateAppWindowsLocked(int displayId) {
    ArrayList<TaskStack> stacks = mService.getDisplayContentLocked(displayId).getStacks();
    for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
        final TaskStack stack = stacks.get(stackNdx);
        final ArrayList<Task> tasks = stack.getTasks();
        for (int taskNdx = tasks.size() - 1; taskNdx >= 0; --taskNdx) {
            final AppTokenList tokens = tasks.get(taskNdx).mAppTokens;
            for (int tokenNdx = tokens.size() - 1; tokenNdx >= 0; --tokenNdx) {
                final AppWindowAnimator appAnimator = tokens.get(tokenNdx).mAppAnimator;
                appAnimator.wasAnimating = appAnimator.animating;
                if (appAnimator.stepAnimationLocked(mCurrentTime, displayId)) {
                    appAnimator.animating = true;
                    setAnimating(true);
                    mAppWindowAnimating = true;
                } else if (appAnimator.wasAnimating) {
                    // stopped animating, do one more pass through the layout
                    setAppLayoutChanges(appAnimator,
                            WindowManagerPolicy.FINISH_LAYOUT_REDO_WALLPAPER,
                            "appToken " + appAnimator.mAppToken + " done", displayId);
                    ......
                }
            }
        }
```

调用stepAnimationLocked()

```java
[->AppWindowAnimator.java]
    // This must be called while inside a transaction.
    boolean stepAnimationLocked(long currentTime, final int displayId) {
        if (mService.okToDisplay()) {
            ......

            if ((mAppToken.allDrawn || animating || mAppToken.startingDisplayed)
                    && animation != null) {
                ......
                if (stepAnimation(currentTime)) {
                    // animation isn't over, step any thumbnail and that's
                    // it for now.
                    if (thumbnail != null) {
                        stepThumbnailAnimation(currentTime);
                    }
                    return true;
                }
            }
        } else if (animation != null) {
            animating = true;
            animation = null;
        }
        ......
    }
}
```

调用stepAnimation()

```java
[->AppWindowAnimator.java]
private boolean stepAnimation(long currentTime) {
    if (animation == null) {
        return false;
    }
    //1.
    transformation.clear();
    final long animationFrameTime = getAnimationFrameTime(animation, currentTime);
    //2.
    boolean hasMoreFrames = animation.getTransformation(animationFrameTime, transformation);
    if (!hasMoreFrames) {
        if (deferThumbnailDestruction && !deferFinalFrameCleanup) {
            deferFinalFrameCleanup = true;
            hasMoreFrames = true;
        } else {
            deferFinalFrameCleanup = false;
            if (mProlongAnimation == PROLONG_ANIMATION_AT_END) {
                hasMoreFrames = true;
            } else {
                setNullAnimation();
                clearThumbnail();
            }
        }
    }
    hasTransformation = hasMoreFrames;
    return hasMoreFrames;
}
```

1.将成员变量transformation所描述的变换矩阵的数据清空 2.调用Animation.getTransformation()来计算Activity组件切换动画下一步所对应的变换矩阵，并且将这个变换矩阵的数据保存在成员变量transformation

### 6.4、窗口动画的推进过程

继续分析WindowAnimator.animateLocked()的updateWindowsLocked()

### 6.4.1、WindowAnimator.updateWindowsLocked()

```java
[->WindowAnimator.java]
private void updateWindowsLocked(final int displayId) {
    ++mAnimTransactionSequence;

    final WindowList windows = mService.getWindowListLocked(displayId);
    ......
    for (int i = windows.size() - 1; i >= 0; i--) {
        WindowState win = windows.get(i);
        WindowStateAnimator winAnimator = win.mWinAnimator;
        final int flags = win.mAttrs.flags;
        boolean canBeForceHidden = mPolicy.canBeForceHidden(win, win.mAttrs);
        boolean shouldBeForceHidden = shouldForceHide(win);
        if (winAnimator.hasSurface()) {
            final boolean wasAnimating = winAnimator.mWasAnimating;
            final boolean nowAnimating = winAnimator.stepAnimationLocked(mCurrentTime);
            winAnimator.mWasAnimating = nowAnimating;
            orAnimating(nowAnimating);
            }
        }
    ......
    }
    ......
}
```

WindowStateAnimator.stepAnimationLocked() 如果窗口的动画尚未结束显示，那么stepAnimationLocked()会返回一个true值给调用者，否则的话，就会返回一个false值给调用者

```java
[->WindowStateAnimator.java]
    boolean stepAnimationLocked(long currentTime) {
        // Save the animation state as it was before this step so WindowManagerService can tell if
        // we just started or just stopped animating by comparing mWasAnimating with isAnimationSet().
        mWasAnimating = mAnimating;
        final DisplayContent displayContent = mWin.getDisplayContent();
        if (displayContent != null && mService.okToDisplay()) {
            // We will run animations as long as the display isn't frozen.
            if (mWin.isDrawnLw() && mAnimation != null) {
                mHasTransformation = true;
                mHasLocalTransformation = true;
                if (!mLocalAnimating) {
                    final DisplayInfo displayInfo = displayContent.getDisplayInfo();
                    if (mAnimateMove) {
                        mAnimateMove = false;
                        mAnimation.initialize(mWin.mFrame.width(), mWin.mFrame.height(),
                                mAnimDx, mAnimDy);
                    } else {
                        mAnimation.initialize(mWin.mFrame.width(), mWin.mFrame.height(),
                                displayInfo.appWidth, displayInfo.appHeight);
                    }
                    mAnimDx = displayInfo.appWidth;
                    mAnimDy = displayInfo.appHeight;
                    mAnimation.setStartTime(mAnimationStartTime != -1
                            ? mAnimationStartTime
                            : currentTime);
                    mLocalAnimating = true;
                    mAnimating = true;
                }
                if ((mAnimation != null) && mLocalAnimating) {
                    mLastAnimationTime = currentTime;
                    if (stepAnimation(currentTime)) {
                        return true;
                    }
                }
                ......
            }
        ......
        return false;
    }
```

```java
[->WindowStateAnimator.java]
private boolean stepAnimation(long currentTime) {
    ......
    currentTime = getAnimationFrameTime(mAnimation, currentTime);
    //1.
    mTransformation.clear();
    //2.
    final boolean more = mAnimation.getTransformation(currentTime, mTransformation);
    if (mAnimationStartDelayed && mAnimationIsEntrance) {
        mTransformation.setAlpha(0f);
    }
    return more;
}
```

1、将成员变量mTransformation所描述的变换矩阵的数据清空。 2、调用mAnimation.getTransformation()来计算窗口动画下一步所对应的变换矩阵，并且将这个变换矩阵的数据保存在成员变量mTransformation。

然后就是动画过后，窗口大小计算、渲染合成等等显示步骤了，由于之前已经分析过了，不再分析了： 3、循环遍历保存在窗口堆栈的每一个WindowState对象，以便可以对系统中的每一个窗口的绘图表面进行更新 确定该窗口实际要显示的大小、位置、Alpha通道和变换矩阵等信息

```java
[->WindowAnimator.java::animateLocked()]
//确定该窗口实际要显示的大小、位置、Alpha通道和变换矩阵等信息
for (int j = 0; j < N; j++) {
    windows.get(j).mWinAnimator.prepareSurfaceLocked(true);
}
```

4、触发下一帧动画逻辑

```java
[->WindowAnimator.java::animateLocked()]
if (mAnimating) {
    mService.scheduleAnimationLocked();
}
```

5、刷新系统UI

```java
[->WindowAnimator.java::animateLocked()]
if (hasPendingLayoutChanges || doRequest) {
    mWindowPlacerLocked.requestTraversal();
}
```

最终经过SurfaceFlinger合成显示到屏幕上。 总体流程图(...)： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/android.wms/20-Android-WMS-animation_Locked-time-diagram.png)

## （七）、参考文档(特别感谢各位前辈的分析和图示)：

[浅析 Android 的窗口](https://www.bbsmax.com/A/n2d9gjgJDv/)
[WMS:窗口大小的计算](https://www.jianshu.com/p/2faedc664d11)
[Android 窗口的计算过程](http://blog.csdn.net/luozirong/article/details/70256809)
[Android Window 机制探索](http://blog.csdn.net/qian520ao/article/details/78555397)
[Android 窗口管理 - 且听风吟](https://calvinlee.github.io/blog/2012/04/21/android-window-management-architecture/)
[Android 关于Window Overscan](http://www.cnblogs.com/all-for-fiona/p/4054527.html)
[WindowManagerService动画分析](http://blog.csdn.net/guoqifa29/article/details/49273065)
[深入理解Activity----Token之旅 - CSDN博客](http://blog.csdn.net/guoqifa29/article/details/46819377)
[Android 4.4(KitKat)窗口管理子系统 - 体系框架](http://blog.csdn.net/jinzhuojun/article/details/37737439) 
[Android窗口系统第四篇---Activity动画的设置过程](https://www.jianshu.com/p/c2e48b3e33a0)
[Android 7.1 GUI系统-窗口管理WMS-Surface管理（四）](http://blog.csdn.net/lin20044140410/article/details/78798048)
[Android 的窗口管理系统 (View, Canvas, WindowManager)](http://www.cnblogs.com/samchen2009/p/3367496.html) 
[WMS--启动窗口(StartingWindow) - Gityuan博客 | 袁辉辉博客](http://gityuan.com/2017/01/15/wms_starting_window/)
[View绘制流程及源码解析(一)----performTraversals()源码分析](https://www.jianshu.com/p/a65861e946cb)
[Android窗口系统第三篇---WindowManagerService中窗口的组织方式](http://blog.csdn.net/u013263323/article/details/78482141)
[google 进入分屏后在横屏模式按 home 键界面错乱 (二) - Android - 掘金](https://juejin.im/entry/58c899bea22b9d006411241c)
[Android应用Activity、Dialog、PopWindow、Toast窗口添加机制及源码分析](http://blog.csdn.net/yanbober/article/details/46361191)
[Android窗口管理服务WindowManagerService的简要介绍和学习计划 - CSDN博客](http://blog.csdn.net/luoshengyang/article/details/8462738)
[Android窗口管理分析（2）：WindowManagerService窗口管理之Window添加流程](https://www.jianshu.com/p/40776c123adb) 
[Android窗口管理服务WindowManagerService显示窗口动画的原理分析 - CSDN博客](http://blog.csdn.net/luoshengyang/article/details/8611754)
[Android6.0 WMS（五） WMS计算Activity窗口大小的过程分析（二）WMS的relayoutWindow](http://blog.csdn.net/kc58236582/article/details/53782138)
