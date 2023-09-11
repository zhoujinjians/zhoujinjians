---
title: Android N 基础（4）：Android 7.1.2 Activity 启动流程 （AMS）分析
cover: https://raw.githubusercontent.com/zhoujinjiani/PicGo/master/hexo.themes/bing-wallpaper-2018.04.03.jpg
categories: 
  - Android
tags:
  - Android
toc: true
abbrlink: 20171108
date: 2017-11-08 09:25:00
---



Activity启动流程概述：
● 点击桌面App图标，Launcher进程采用Binder IPC向system_server进程发起startActivity请求；
● system_server进程接收到请求后，向zygote进程发送创建进程的请求；
● Zygote进程fork出新的子进程，即App进程；
● App进程，通过Binder IPC向sytem_server进程发起attachApplication请求；
● system_server进程在收到请求后，进行一系列准备工作后，再通过binder IPC向App进程发送scheduleLaunchActivity请求；
● App进程的binder线程（ApplicationThread）在收到请求后，通过handler向主线程发送LAUNCH_ACTIVITY消息；
● 主线程在收到Message后，通过发射机制创建目标Activity，并回调Activity.onCreate()等方法。 <!-- more -->

## 一、概述

基于Android 7.1.2的源码剖析， 分析Android Activity启动流程，相关源码：

--------------------------------------------------------------------------------

**frameworks/base/services/core/java/com/android/server/am/**
● ActivityManagerService.java
● ActivityStackSupervisor.java
● ActivityStack.java
● ActivityRecord.java
● ProcessRecord.java
● TaskRecord.java

**frameworks/base/services/core/java/com/android/server/pm/**
● PackageManagerService.java

**frameworks/base/core/java/android/os/**
● Process.java

**frameworks/base/core/java/android/app/**
● IActivityManager.java
● ActivityManagerNative.java (内含AMP)
● ActivityManager.java
● Activity.java
● ActivityThread.java
● Instrumentation.java
● IApplicationThread.java
● ApplicationThreadNative.java (内含ATP)
● ActivityThread.java (内含ApplicationThread)
● ContextImpl.java

> **主要对象功能介绍：**
● ActivityManagerService，简称AMS，服务端对象，负责系统中所有Activity的生命周期。
● ActivityManagerNative继承Java的Binder类，同时实现了IActivityManager接口，即ActivityManagerNative将作为Binder通信的服务端为用户提供支持。
● ActivityManagerProxy：在ActivityManagerNative类中定义了内部类ActivityManagerProxy，该类同样实现了IActivityManager接口，将作为客户端使用的服务端代理。
● ActivityThread，App的真正入口。当开启App之后，会调用main()开始运行，开启消息循环队列，这就是传说中的UI线程或者叫主线程。与ActivityManagerServices配合，一起完成Activity的管理工作
● ApplicationThread，用来实现ActivityManagerService与ActivityThread之间的交互。在ActivityManagerService需要管理相关Application中的Activity的生命周期时，通过ApplicationThread的代理对象与ActivityThread通讯。
● ApplicationThreadProxy，是ApplicationThread在服务器端的代理，负责和客户端的ApplicationThread通讯。AMS就是通过该代理与ActivityThread进行通信的。
● Instrumentation，每一个应用程序只有一个Instrumentation对象，每个Activity内都有一个对该对象的引用。Instrumentation可以理解为应用进程的管家，ActivityThread要创建或暂停某个Activity时，都需要通过Instrumentation来进行具体的操作。
● ActivityStack，Activity在AMS的栈管理，用来记录已经启动的Activity的先后关系，状态信息等。通过ActivityStack决定是否需要启动新的进程。
● ActivityRecord，ActivityStack的管理对象，每个Activity在AMS对应一个ActivityRecord，来记录Activity的状态以及其他的管理信息。其实就是服务器端的Activity对象的映像。
● TaskRecord，AMS抽象出来的一个"任务"的概念，是记录ActivityRecord的栈，一个"Task"包含若干个ActivityRecord。AMS用TaskRecord确保Activity启动和退出的顺序。如果你清楚Activity的4种launchMode，那么对这个概念应该不陌生。

--------------------------------------------------------------------------------

**相关类的类图：** 
（1）IActivityManager相关类 
![Markdown](https://raw.githubusercontent.com/zhoujinjiani/PicGo/master/android.startactivity/N1-Android-startActivity-IAM-class.png)
（2）IApplicationThread相关类
![Markdown](https://raw.githubusercontent.com/zhoujinjiani/PicGo/master/android.startactivity/N2-Android-startActivity-AMS-class.png)
（3）ActivityManagerService相关类
![Markdown](https://raw.githubusercontent.com/zhoujinjiani/PicGo/master/android.startactivity/N3-Android-startActivity-IAP-class.png)


### 1.1、Task和Stack

Android系统中的每一个Activity都位于一个Task中。一个Task可以包含多个Activity，同一个Activity也可能有多个实例。 在AndroidManifest.xml中，我们可以通过android:launchMode来控制Activity在Task中的实例。

另外，在startActivity的时候，我们也可以通过setFlag 来控制启动的Activity在Task中的实例。

Task管理的意义还在于近期任务列表以及Back栈。 当你通过多任务键（有些设备上是长按Home键，有些设备上是专门提供的多任务键）调出多任务时，其实就是从ActivityManagerService获取了最近启动的Task列表。

Back栈管理了当你在Activity上点击Back键，当前Activity销毁后应该跳转到哪一个Activity的逻辑。关于Task和Back栈，请参见这里：[Tasks and Back Stack](https://developer.android.com/guide/components/tasks-and-back-stack.html)。

其实在ActivityManagerService与WindowManagerService内部管理中，在Task之外，还有一层容器，这个容器应用开发者和用户可能都不会感觉到或者用到，但它却非常重要，那就是Stack。 下文中，我们将看到，Android系统中的多窗口管理，就是建立在Stack的数据结构上的。 一个Stack中包含了多个Task，一个Task中包含了多个Activity（Window），下图描述了它们的关系： 
![Markdown](https://raw.githubusercontent.com/zhoujinjiani/PicGo/master/android.startactivity/Android-_Activity-task_stack.png)
 另外还有一点需要注意的是，ActivityManagerService和WindowManagerService中的Task和Stack结构是一一对应的，对应关系对于如下：

ActivityStack <–> TaskStack TaskRecord <–> Task 即，ActivityManagerService中的每一个ActivityStack或者TaskRecord在WindowManagerService中都有对应的TaskStack和Task，这两类对象都有唯一的id（id是int类型），它们通过id进行关联。

#### 1.2、小结：

> 用户从Launcher程序点击应用图标可启动应用的入口Activity，Activity启动时需要多个进程之间的交互，Android系统中有一个zygote进程专用于孵化Android框架层和应用层程序的进程。还有一个system_server进程，该进程里运行了很多binder service，例如ActivityManagerService，PackageManagerService，WindowManagerService，这些binder service分别运行在不同的线程中，其中ActivityManagerService负责管理Activity栈，应用进程，task。 用户在Launcher程序里点击应用图标时，会通知ActivityManagerService启动应用的入口Activity，ActivityManagerService发现这个应用还未启动，则会通知Zygote进程孵化出应用进程，然后在这个应用进程里执行ActivityThread的main方法。应用进程接下来通知ActivityManagerService应用进程已启动，ActivityManagerService保存应用进程的一个代理对象，这样ActivityManagerService可以通过这个代理对象控制应用进程，然后ActivityManagerService通知应用进程创建入口Activity的实例，并执行它的生命周期方法。

#### **总体启动流程图：**

![Markdown](https://raw.githubusercontent.com/zhoujinjiani/PicGo/master/android.startactivity/N4-Android-startActivity-flow.png)

### 二、 开始请求执行启动Activity

> Activity.startActivity()
Activity.startActivityForResult()
Instrumentation.execStartActivity()
ActivityManagerProxy.startActivity()
ActivityManagerNative.onTransact()
ActivityManagerService.startActivity()

#### 2.1、Activity.startActivity()

从Launcher启动应用的时候，经过调用会执行Activity中的startActivity。

```java
[-> Activity.java]
......
@Override
public void startActivity(Intent intent) {
    this.startActivity(intent, null);
}
......
@Override
public void startActivity(Intent intent, @Nullable Bundle options) {
    if (options != null) {
        startActivityForResult(intent, -1, options);
    } else {
        ......
        startActivityForResult(intent, -1);
    }
}
```

#### 2.2、Activity.startActivityForResult()

```java
[-> Activity.java]
    @Override
public void startActivityForResult(
        String who, Intent intent, int requestCode, @Nullable Bundle options) {
    ......
    Instrumentation.ActivityResult ar =
        mInstrumentation.execStartActivity(
            this, mMainThread.getApplicationThread(), mToken, who,
            intent, requestCode, options);
    ......
}
```

可以发现execStartActivity方法传递的几个参数： this，为启动Activity的对象； contextThread，为Binder对象，是主进程的context对象； token，也是一个Binder对象，指向了服务端一个ActivityRecord对象； target，为启动的Activity； intent，启动的Intent对象； requestCode，请求码； options，参数；

#### 2.3、Instrumentation.execStartActivity()

```java
[-> Instrumentation.java]
    ......
public ActivityResult execStartActivity(
    Context who, IBinder contextThread, IBinder token, String target,
    Intent intent, int requestCode, Bundle options) {
    IApplicationThread whoThread = (IApplicationThread) contextThread;
    ......
    try {
        intent.migrateExtraStreamToClipData();
        intent.prepareToLeaveProcess(who);
        int result = ActivityManagerNative.getDefault()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token, target, requestCode, 0, null, options);
        checkStartActivityResult(result, intent);
    } ......
}
```

关于 ActivityManagerNative.getDefault()返回的是?

```java
 [->ActivityManagerNative.java]
/**
 * Retrieve the system's default/global activity manager.
 */
static public IActivityManager getDefault() {
    return gDefault.get();
}
```

直接返回的是gDefault.get()，那么gDefault又是什么呢？

```java
    [->ActivityManagerNative.java]
    private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
    protected IActivityManager create() {
        IBinder b = ServiceManager.getService("activity");
        if (false) {
            Log.v("ActivityManager", "default service binder = " + b);
        }
        IActivityManager am = asInterface(b);
        if (false) {
            Log.v("ActivityManager", "default service = " + am);
        }
        return am;
    }
};
```

可以发现启动过asInterface()方法创建，然后我们继续看一下asInterface方法的实现：

```java
    static public IActivityManager asInterface(IBinder obj) {
    if (obj == null) {
        return null;
    }
    IActivityManager in =
        (IActivityManager)obj.queryLocalInterface(descriptor);
    if (in != null) {
        return in;
    }

    return new ActivityManagerProxy(obj);
}
```

最后直接返回一个对象， 此处startActivity()的共有10个参数, 下面说说每个参数传递AMP.startActivity()每一项的对应值:

caller: 当前应用的ApplicationThread对象mAppThread; callingPackage: 调用当前ContextImpl.getBasePackageName(),获取当前Activity所在包名; intent: 这便是启动Activity时,传递过来的参数; resolvedType: 调用intent.resolveTypeIfNeeded而获取; resultTo: 来自于当前Activity.mToken resultWho: 来自于当前Activity.mEmbeddedID requestCode = -1; startFlags = 0; profilerInfo = null; options = null;

好吧，最后直接返回一个对象，而继承与IActivityManager，到了这里就引出了我们android系统中很重要的一个概念：Binder机制。我们知道应用进程与SystemServer进程属于两个不同的进程，进程之间需要通讯，android系统采取了自身设计的Binder机制，这里的和ActivityManagerNative都是继承与IActivityManager的而SystemServer进程中的ActivityManagerService对象则继承与ActivityManagerNative。简单的表示： Binder接口 –> ActivityManagerNative/ –> ActivityManagerService；

这样，ActivityManagerNative与相当于一个Binder的客户端而ActivityManagerService相当于Binder的服务端，这样当ActivityManagerNative调用接口方法的时候底层通过Binder driver就会将请求数据与请求传递给server端，并在server端执行具体的接口逻辑。需要注意的是Binder机制是单向的，是异步的，也就是说只能通过client端向server端传递数据与请求而不用等待服务端的返回，也无法返回，那如果SystemServer进程想向应用进程传递数据怎么办？这时候就需要重新定义一个Binder请求以SystemServer为client端，以应用进程为server端，这样就是实现了两个进程之间的双向通讯。

好了，说了这么多我们知道这里的ActivityManagerNative是ActivityManagerService在应用进程的一个client就好了，通过它就可以滴啊用ActivityManagerService的方法了。

#### 2.4、ActivityManagerProxy.startActivity()

ActivityManagerNative.getDefault()方法会返回一个对象，那么我们看一下对象的startActivity方法：

```java
[-> ActivityManagerNative.java :: ActivityManagerProxy]
class  ActivityManagerProxy implements IActivityManager
{
...
    public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
        String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
    ......
    mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
    ......
}
...
}
```

#### 2.5、ActivityManagerNative.onTransact()

```java
[-> ActivityManagerNative.java]
    @Override
public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
        throws RemoteException {
    switch (code) {
    case START_ACTIVITY_TRANSACTION:
    {
        ......
        int result = startActivity(app, callingPackage, intent, resolvedType,
                resultTo, resultWho, requestCode, startFlags, profilerInfo, options);
        ......
    }
    ......
  }
```

这里就涉及到了具体的Binder数据传输机制了，我们不做过多的分析，知道通过数据传输之后就会调用SystemServer进程的ActivityManagerService的startActivity就好了。

以上其实都是发生在应用进程中，下面开始调用的ActivityManagerService的执行时发生在SystemServer进程。

### 三、 ActivityManagerService接收启动Activity的请求

> ActivityManagerService.startActivity()
ActivityStarter.startActivityMayWait()
ActivityStarter.startActivityLocked()
ActivityStarter.startActivityUnchecked()
ActivityStackSupervisor.resumeFocusedStackTopActivityLocked() ActivityStack.resumeTopActivityUncheckedLocked()
ActivityStack.resumeTopActivityInnerLocked()
-->ActivityStackSupervisor.pauseBackStacks() [if (mResumedActivity != null)] -->ActivityStackSupervisor.startSpecificActivityLocked() [if (mResumedActivity == null)]

> ##### 3.1、ActivityManagerService.startActivity()

```java
[-> ActivityManagerService.java]
    public final int startActivity(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
    return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, bOptions,
            UserHandle.getCallingUserId());
}

 @Override
public final int startActivityAsCaller(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, boolean ignoreTargetSecurity,
        int userId) {
        ......
    try {
        int ret = mActivityStarter.startActivityMayWait(null, targetUid, targetPackage, intent,
                resolvedType, null, null, resultTo, resultWho, requestCode, startFlags, null,
                null, null, bOptions, ignoreTargetSecurity, userId, null, null);
        return ret;
    }
        ......
}

final int startActivity(Intent intent, ActivityStackSupervisor.ActivityContainer container) {
    enforceNotIsolatedCaller("ActivityContainer.startActivity");
    final int userId = mUserController.handleIncomingUser(Binder.getCallingPid(),
            Binder.getCallingUid(), mStackSupervisor.mCurrentUser, false,
            ActivityManagerService.ALLOW_FULL_ONLY, "ActivityContainer", null);

    ......
    return mActivityStarter.startActivityMayWait(null, -1, null, intent, mimeType, null, null, null,
            null, 0, 0, null, null, null, null, false, userId, container, null);
}
```

可以看到这里只是进行了一些关于userid的逻辑判断，然后就调用mStackSupervisor.startActivityMayWait方法，此处mStackSupervisor的数据类型为ActivityStackSupervisor。 当程序运行到这里时, ASS.startActivityMayWait的各个参数取值如下:

caller = ApplicationThreadProxy, 用于跟调用者进程ApplicationThread进行通信的binder代理类. callingUid = -1; callingPackage = ContextImpl.getBasePackageName(),获取调用者Activity所在包名 intent: 这是启动Activity时传递过来的参数; resolvedType = intent.resolveTypeIfNeeded voiceSession = null; voiceInteractor = null; resultTo = Activity.mToken, 其中Activity是指调用者所在Activity, mToken对象保存自己所处的ActivityRecord信息 resultWho = Activity.mEmbeddedID, 其中Activity是指调用者所在Activity requestCode = -1; startFlags = 0; profilerInfo = null; outResult = null; config = null; options = null; ignoreTargetSecurity = false; userId = AMS.handleIncomingUser, 当调用者userId跟当前处于同一个userId,则直接返回该userId;当不相等时则根据调用者userId来决定是否需要将callingUserId转换为mCurrentUserId. iContainer = null; inTask = null;

下面我们来看一下这个方法的具体实现：

#### 3.2、ActivityStarter.startActivityMayWait()

```java
[->ActivityStarter.java]
final int startActivityMayWait(IApplicationThread caller, int callingUid,
        String callingPackage, Intent intent, String resolvedType,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        IBinder resultTo, String resultWho, int requestCode, int startFlags,
        ProfilerInfo profilerInfo, IActivityManager.WaitResult outResult, Configuration config,
        Bundle bOptions, boolean ignoreTargetSecurity, int userId,
        IActivityContainer iContainer, TaskRecord inTask) {
    ......
    // Save a copy in case ephemeral needs it
    final Intent ephemeralIntent = new Intent(intent);
    // Don't modify the client's object!
    intent = new Intent(intent);

    ResolveInfo rInfo = mSupervisor.resolveIntent(intent, resolvedType, userId);
    ......
    // Collect information about the target of the Intent.
    ActivityInfo aInfo = mSupervisor.resolveActivity(intent, rInfo, startFlags, profilerInfo);

    ActivityOptions options = ActivityOptions.fromBundle(bOptions);
    ActivityStackSupervisor.ActivityContainer container =
            (ActivityStackSupervisor.ActivityContainer)iContainer;
    ......
        final ActivityRecord[] outRecord = new ActivityRecord[1];
        int res = startActivityLocked(caller, intent, ephemeralIntent, resolvedType,
                aInfo, rInfo, voiceSession, voiceInteractor,
                resultTo, resultWho, requestCode, callingPid,
                callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
                options, ignoreTargetSecurity, componentSpecified, outRecord, container,
                inTask);

       ......
        return res;
    }
}
```

该过程主要功能：通过resolveActivity来获取ActivityInfo信息, 然后再进入ASS.startActivityLocked().先来看看

#### 3.2.1、ActivityStackSupervisor.resolveActivity()

```java
[->ActivityStackSupervisor.java]
        ResolveInfo resolveIntent(Intent intent, String resolvedType, int userId) {
    return resolveIntent(intent, resolvedType, userId, 0);
}

ResolveInfo resolveIntent(Intent intent, String resolvedType, int userId, int flags) {
    try {
        return AppGlobals.getPackageManager().resolveIntent(intent, resolvedType,
                PackageManager.MATCH_DEFAULT_ONLY | flags
                | ActivityManagerService.STOCK_PM_FLAGS, userId);
    } catch (RemoteException e) {
    }
    return null;
}

ActivityInfo resolveActivity(Intent intent, String resolvedType, int startFlags,
        ProfilerInfo profilerInfo, int userId) {
    final ResolveInfo rInfo = resolveIntent(intent, resolvedType, userId);
    return resolveActivity(intent, rInfo, startFlags, profilerInfo);
}
```

#### 3.2.2、PackageManagerService.resolveIntent()

AppGlobals.getPackageManager()经过函数层层调用，获取的是ApplicationPackageManager对象。经过binder IPC调用，最终会调用PackageManagerService对象。故此时调用方法为PMS.resolveIntent().

```java
[-> PackageManagerService.java]
    @Override
public ResolveInfo resolveIntent(Intent intent, String resolvedType,
        int flags, int userId) {
    try {
        ......
        final List<ResolveInfo> query = queryIntentActivitiesInternal(intent, resolvedType,
                flags, userId);
        ......

        final ResolveInfo bestChoice =
                chooseBestActivity(intent, resolvedType, flags, query, userId);
        return bestChoice;
    }
    ......
}
    private @NonNull List<ResolveInfo> queryIntentActivitiesInternal(Intent intent,
        String resolvedType, int flags, int userId) {
    ......
    ComponentName comp = intent.getComponent();
    if (comp == null) {
        if (intent.getSelector() != null) {
            intent = intent.getSelector();
            comp = intent.getComponent();
        }
    }

    if (comp != null) {
        final List<ResolveInfo> list = new ArrayList<ResolveInfo>(1);
        final ActivityInfo ai = getActivityInfo(comp, flags, userId);
        if (ai != null) {
            final ResolveInfo ri = new ResolveInfo();
            ri.activityInfo = ai;
            list.add(ri);
        }
        return list;
    }

    ......
}
```

ActivityStackSupervisor.resolveActivity()方法的核心功能是找到相应的Activity组件，并保存到intent对象。

#### 3.3、ActivityStarter.startActivityLocked()

继续ActivityStarter.startActivityMayWait()个方法中执行了启动Activity的一些其他逻辑判断，在经过判断逻辑之后调用startActivityLocked方法：

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
        mService.mWindowManager.deferSurfaceLayout();
        err = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor, startFlags,
                true, options, inTask);
    }
    ......
    return err;
}
```

这个方法中主要构造了ActivityManagerService端的Activity对象–>ActivityRecord，并根据Activity的启动模式执行了相关逻辑。然后调用了startActivityUncheckedLocked方法：

#### 3.4、ActivityStarter.startActivityUnchecked()

```java
[->ActivityStarter.java]
    private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask) {

    ......
    boolean newTask = false;
    ......
    mTargetStack.startActivityLocked(mStartActivity, newTask, mKeepCurTransition, mOptions);
    if (mDoResume) {
        if (!mLaunchTaskBehind) {
            // ......
            mService.setFocusedActivityLocked(mStartActivity, "startedActivity");
        }
        final ActivityRecord topTaskActivity = mStartActivity.task.topRunningActivityLocked();
        if (!mTargetStack.isFocusable()
                || (topTaskActivity != null && topTaskActivity.mTaskOverlay
                && mStartActivity != topTaskActivity)) {
            // ......
            mTargetStack.ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
            // .......
            mWindowManager.executeAppTransition();
        } else {
            //3.6 ActivityStackSupervisor.resumeFocusedStackTopActivityLocked()
            mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity,
                    mOptions);
        }
    } else {
        mTargetStack.addRecentActivityLocked(mStartActivity);
    }
    mSupervisor.updateUserStackLocked(mStartActivity.userId, mTargetStack);

    mSupervisor.handleNonResizableTaskIfNeeded(
            mStartActivity.task, preferredLaunchStackId, mTargetStack.mStackId);

    return START_SUCCESS;
}
```

找到或创建新的Activit所属于的Task对象，之后调用ActivityStack.startActivityLocked()

#### 3.4.1、ActivityStack.startActivityLocked()

```java
[-> ActivityStack.java]
    final void startActivityLocked(ActivityRecord r, boolean newTask, boolean keepCurTransition,
        ActivityOptions options) {
    ......
    } else {
        // If this is the first activity, don't do any fancy animations,
        // because there is nothing for it to animate on top of.
        addConfigOverride(r, task);
        ......
    }
    ......
}
    void addConfigOverride(ActivityRecord r, TaskRecord task) {
    final Rect bounds = task.updateOverrideConfigurationFromLaunchBounds();
    // TODO: VI deal with activity
    mWindowManager.addAppToken(task.mActivities.indexOf(r), r.appToken,
            r.task.taskId, mStackId, r.info.screenOrientation, r.fullscreen,
            (r.info.flags & FLAG_SHOW_FOR_ALL_USERS) != 0, r.userId, r.info.configChanges,
            task.voiceSession != null, r.mLaunchTaskBehind, bounds, task.mOverrideConfig,
            task.mResizeMode, r.isAlwaysFocusable(), task.isHomeTask(),
            r.appInfo.targetSdkVersion, r.mRotationAnimationHint);
    r.taskConfigOverride = task.mOverrideConfig;
}
```

#### 3.5、ActivityStackSupervisor.resumeFocusedStackTopActivityLocked()

```java
    [->ActivityStackSupervisor.java]
    boolean resumeFocusedStackTopActivityLocked(
        ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {
    if (targetStack != null && isFocusedStack(targetStack)) {
        return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
    }
    final ActivityRecord r = mFocusedStack.topRunningActivityLocked();
    if (r == null || r.state != RESUMED) {
        mFocusedStack.resumeTopActivityUncheckedLocked(null, null);
    }
    return false;
}
```

#### 3.6、ActivityStack.resumeTopActivityUncheckedLocked()

inResumeTopActivity用于保证每次只有一个Activity执行resumeTopActivityLocked()操作.

```java
[->ActivityStack.java]
    boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
    if (mStackSupervisor.inResumeTopActivity) {
        // Don't even start recursing.
        return false;
    }

    boolean result = false;
    try {
        // Protect against recursion.
        mStackSupervisor.inResumeTopActivity = true;
        if (mService.mLockScreenShown == ActivityManagerService.LOCK_SCREEN_LEAVING) {
            mService.mLockScreenShown = ActivityManagerService.LOCK_SCREEN_HIDDEN;
            mService.updateSleepIfNeededLocked();
        }
        result = resumeTopActivityInnerLocked(prev, options);
    } finally {
        mStackSupervisor.inResumeTopActivity = false;
    }
    return result;
}
```

#### 3.7、ActivityStack.resumeTopActivityInnerLocked()

说明：启动一个新Activity时，如果界面还存在其它的Activity，那么必须先中断其它的Activity。 因此，除了第一个启动的Home界面对应的Activity外，其它的Activity均需要进行此操作，当系统启动第一个Activity，即Home时，mResumedActivity的值才会为null。 经过一系列处理逻辑之后最终调用了startPausingLocked方法，这个方法作用就是让系统中栈中的Activity执行onPause方法。

```java
    [->ActivityStack.java]
        private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options)       {
   ......

    // We need to start pausing the current activity so the top one can be resumed...
    final boolean dontWaitForPause = (next.info.flags & FLAG_RESUME_WHILE_PAUSING) != 0;
    boolean pausing = mStackSupervisor.pauseBackStacks(userLeaving, next, dontWaitForPause);
    if (mResumedActivity != null) {
        pausing |= startPausingLocked(userLeaving, false, next, dontWaitForPause);
    }
    ......
     else {
        ......
        mStackSupervisor.startSpecificActivityLocked(next, true, true);
    }
}
```

#### 3.8、ActivityStackSupervisor.pauseBackStacks()

暂停所有处于后台栈的所有Activity。

```java
    [->ActivityStackSupervisor.java]
    boolean pauseBackStacks(boolean userLeaving, ActivityRecord resuming, boolean dontWait) {
    boolean someActivityPaused = false;
    for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
        ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
        for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
            final ActivityStack stack = stacks.get(stackNdx);
            if (!isFocusedStack(stack) && stack.mResumedActivity != null) {
                ......
                someActivityPaused |= stack.startPausingLocked(userLeaving, false, resuming,
                        dontWait);
            }
        }
    }
    return someActivityPaused;
}
```

### 四、执行栈顶Activity的onPause方法

> ActivityStack.startPausingLocked()
ActivityThread.schedulePauseActivity()
ActivityThread.sendMessage()
ActivityThread.H.sendMessage()
ActivityThread.H.handleMessage()
--> ->ActivityThread.handlePauseActivity()
-->ActivityThread.performPauseActivity()
---->ActivityThread.performPauseActivityIfNeeded()
----> Instrumentation.callActivityOnPause()
----> Activity.performPause()
----> Activity.onPause()
-->ActivityManagerService.activityPaused()
---->ActivityStack.activityPausedLocked()
---->ActivityStack.completePauseLocked()
---->ActivityStackSupervisor.resumeFocusedStackTopActivityLocked() ---->ActivityStack.resumeTopActivityUncheckedLocked()
---->ActivityStack.resumeTopActivityInnerLocked()
---->ActivityStackSupervisor.startSpecificActivityLocked()

#### 4.1、ActivityStack.startPausingLocked()

```java
    [->ActivityStack.java]
    final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping,
        ActivityRecord resuming, boolean dontWait) {
    ......
        try {
            ......
            mService.updateUsageStats(prev, false);
            prev.app.thread.schedulePauseActivity(prev.appToken, prev.finishing,
                    userLeaving, prev.configChangeFlags, dontWait);
        }
    ......
}
```

可以看到这里执行了pre.app.thread.schedulePauseActivity方法，通过分析不难发现这里的thread是一个IApplicationThread类型的对象，而在ActivityThread中也定义了一个ApplicationThread的类，其继承了IApplicationThread，并且都是Binder对象，不难看出这里的IAppcation是一个Binder的client端而ActivityThread中的ApplicationThread是一个Binder对象的server端，所以通过这里的thread.schedulePauseActivity实际上调用的就是ApplicationThread的schedulePauseActivity方法。

`这里的ApplicationThread可以和ActivityManagerNative对于一下： 通过ActivityManagerNative –> ActivityManagerService实现了应用进程与SystemServer进程的通讯 通过AppicationThread <– IApplicationThread实现了SystemServer进程与应用进程的通讯`

然后我们继续看一下ActivityThread中schedulePauseActivity的具体实现：

#### 4.2、ActivityThread.schedulePauseActivity()

```java
    [->ActivityThread.java]
    public final void schedulePauseActivity(IBinder token, boolean finished,
        boolean userLeaving, int configChanges, boolean dontReport) {
    int seq = getLifecycleSeq();
    if (DEBUG_ORDER) Slog.d(TAG, "pauseActivity " + ActivityThread.this
            + " operation received seq: " + seq);
    sendMessage(
            finished ? H.PAUSE_ACTIVITY_FINISHING : H.PAUSE_ACTIVITY,
            token,
            (userLeaving ? USER_LEAVING : 0) | (dontReport ? DONT_REPORT : 0),
            configChanges,
            seq);
}
```

#### 4.3、ActivityThread.sendMessage()

```java
    [->ActivityThread.java]
    private void sendMessage(int what, Object obj, int arg1, int arg2, int seq) {
    ......
    mH.sendMessage(msg);
}
```

最终调用了mH的sendMessage方法，mH是在ActivityThread中定义的一个Handler对象，主要处理SystemServer进程的消息，我们看一下其handleMessge方法的实现：

#### 4.4、[ActivityThread.handleMessage() : mH]

```java
   [->ActivityThread.java : mH]
    public void handleMessage(Message msg) {
    switch (msg.what) {
        ......
        case PAUSE_ACTIVITY: {
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityPause");
            SomeArgs args = (SomeArgs) msg.obj;
            handlePauseActivity((IBinder) args.arg1, false,
                    (args.argi1 & USER_LEAVING) != 0, args.argi2,
                    (args.argi1 & DONT_REPORT) != 0, args.argi3);
            maybeSnapshot();
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        } break;
```

可以发现其调用了handlePauseActivity方法：

#### 4.5、ActivityThread.handlePauseActivity()

```java
[->ActivityThread.java]
private void handlePauseActivity(IBinder token, boolean finished,
        boolean userLeaving, int configChanges, boolean dontReport, int seq) {
    ActivityClientRecord r = mActivities.get(token);
    ......
        performPauseActivity(token, finished, r.isPreHoneycomb(), "handlePauseActivity");
        ......
        // Tell the activity manager we have paused.
        if (!dontReport) {
            try {
                ActivityManagerNative.getDefault().activityPaused(token);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
        }
        mSomeActivitiesChanged = true;
    }
}
```

然后在方法体内部通过调用performPauseActivity方法来实现对栈顶Activity的onPause生命周期方法的回调，可以具体看一下他的实现：

#### 4.6、ActivityThread.performPauseActivity()

```java
   [->ActivityThread.java]
    final Bundle performPauseActivity(ActivityClientRecord r, boolean finished,
        boolean saveState, String reason) {
    ......
    performPauseActivityIfNeeded(r, reason);
    ......
}

private void performPauseActivityIfNeeded(ActivityClientRecord r, String reason) {
    ......
    try {
        r.activity.mCalled = false;
        mInstrumentation.callActivityOnPause(r.activity);
    ......
}
```

这样回到了mInstrumentation的callActivityOnPuase方法：

#### 4.7、Instrumentation.callActivityOnPuase()

```java
    [->Instrumentation.java]
public void callActivityOnPause(Activity activity) {
    activity.performPause();
}
```

原来最终回调到了Activity的performPause方法：

#### 4.8、Activity.performPause()

```java
[->Activity.java]
    final void performPause() {
    mDoReportFullyDrawn = false;
    mFragments.dispatchPause();
    mCalled = false;
    onPause();
    mResumed = false;
    if (!mCalled && getApplicationInfo().targetSdkVersion
            >= android.os.Build.VERSION_CODES.GINGERBREAD) {
        throw new SuperNotCalledException(
                "Activity " + mComponent.toShortString() +
                " did not call through to super.onPause()");
    }
    mResumed = false;
}
```

终于，太不容易了，回调到了Activity的onPause方法，哈哈，Activity生命周期中的第一个生命周期方法终于被我们找到了。。。。也就是说我们在启动一个Activity的时候最先被执行的是栈顶的Activity的onPause方法。记住这点吧，面试的时候经常会问到类似的问题。

然后回到我们的handlePauseActivity方法，在该方法的最后面执行了ActivityManagerNative.getDefault().activityPaused(token);方法，这是应用进程告诉服务进程，栈顶Activity已经执行完成onPause方法了，通过前面我们的分析，我们知道这句话最终会被ActivityManagerService的activityPaused方法执行。

```java
  [->ActivityManagerService.java]
    @Override
public final void activityPaused(IBinder token) {
    final long origId = Binder.clearCallingIdentity();
    synchronized(this) {
        ActivityStack stack = ActivityRecord.getStackLocked(token);
        if (stack != null) {
            stack.activityPausedLocked(token, false);
        }
    }
    Binder.restoreCallingIdentity(origId);
}
```

以发现，该方法内部会调用ActivityStack的activityPausedLocked方法，好吧，看一下activityPausedLocked方法的实现，然后执行了completePauseLocked方法：

```java
    private void completePauseLocked(boolean resumeNext, ActivityRecord resuming) {
    ......
    if (resumeNext) {
        final ActivityStack topStack = mStackSupervisor.getFocusedStack();
        if (!mService.isSleepingOrShuttingDownLocked()) {
            mStackSupervisor.resumeFocusedStackTopActivityLocked(topStack, prev, null);
        } else {
            mStackSupervisor.checkReadyForSleepLocked();
            ActivityRecord top = topStack.topRunningActivityLocked();
            if (top == null || (prev != null && top != prev)) {
                // If there are no more activities available to run, do resume anyway to start
                // something. Also if the top activity on the stack is not the just paused
                // activity, we need to go ahead and resume it to ensure we complete an
                // in-flight app switch.
                mStackSupervisor.resumeFocusedStackTopActivityLocked();
            }
        }
    }
    ......
}
```

经过了一系列的逻辑之后，又调用了resumeTopActivitiesLocked方法，又回到了第三步中解析的方法中了，这样经过

> ActivityStackSupervisor.resumeFocusedStackTopActivityLocked()
–> ActivityStack.resumeTopActivityUncheckedLocked()
–> ActivityStack.resumeTopActivityInnerLocked()
–> ActivityStackSupervisor.startSpecificActivityLocked()

好吧，我们看一下startSpecificActivityLocked的具体实现：

```java
    [->ActivityStack.java]
    void startSpecificActivityLocked(ActivityRecord r,
        boolean andResume, boolean checkConfig) {
    // Is this activity's application already running?
    ProcessRecord app = mService.getProcessRecordLocked(r.processName,
            r.info.applicationInfo.uid, true);

    r.task.stack.setLaunchTime(r);

    if (app != null && app.thread != null) {
        try {
            ......
            realStartActivityLocked(r, app, andResume, checkConfig);
            return;
        } catch (RemoteException e) {
            Slog.w(TAG, "Exception when starting activity "
                    + r.intent.getComponent().flattenToShortString(), e);
        }

        // If a dead object exception was thrown -- fall through to
        // restart the application.
    }

    mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
            "activity", r.intent.getComponent(), false, false, true);
}
```

可以发现在这个方法中，首先会判断一下需要启动的Activity所需要的应用进程是否已经启动，若启动的话，则直接调用realStartAtivityLocked方法，否则调用startProcessLocked方法，用于启动应用进程。 这样关于启动Activity时的第三步骤就已经执行完成了，这里主要是实现了对栈顶Activity执行onPause 方法，而这个方法首先判断需要启动的Activity所属的进程是否已经启动，若已经启动则直接调用启动Activity的方法，否则将先启动Activity的应用进程，然后在启动该Activity。

### 五、创建Activity所属的应用进程

> ActivityManagerService.startProcessLocked()
-> Process.start()
-> 创建进程 Process.startViaZygote()
-> 创建进程 Process.zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote)
-> 创建进程 ActivityThread.main()
ActivityThread.attach()
ActivityManagerProxy.attachApplication()
ActivityManagerNative.onTransact()
ActivityManagerService.attachApplication()
ActivityManagerService.attachApplicationLocked() ApplicationThreadNative.ApplicationThreadProxy.bindApplication()
ApplicationThreadNative.onTransact()
ActivityThread.ApplicationThread.bindApplication()
ActivityThread.sendMessage()
ActivityThread.H.sendMessage()
ActivityThread.H.handleMessage()
ActivityThread.handleBindApplication()

#### 5.1、ActivityManagerService.startProcessLocked()

```java
    [->ActivityManagerService.java]
    private final void startProcessLocked(ProcessRecord app,
        String hostingType, String hostingNameStr) {
    startProcessLocked(app, hostingType, hostingNameStr, null /* abiOverride */,
            null /* entryPoint */, null /* entryPointArgs */);
}

private final void startProcessLocked(ProcessRecord app, String hostingType,
        String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
        ......
                    // Start the process.  It will either succeed and return a result containing
        // the PID of the new process, or else throw a RuntimeException.
        boolean isActivityProcess = (entryPoint == null);
        if (entryPoint == null) entryPoint = "android.app.ActivityThread";
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "Start proc: " +
                app.processName);
        checkTime(startTime, "startProcess: asking zygote to start proc");
        Process.ProcessStartResult startResult = Process.start(entryPoint,
                app.processName, uid, uid, gids, debugFlags, mountExternal,
                app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
                app.info.dataDir, entryPointArgs);
        checkTime(startTime, "startProcess: returned from zygote!");
        ......
        }
```

可以发现其经过一系列的初始化操作之后调用了Process.start方法，并且传入了启动的类名"android.app.ActivityThread":

#### 5.2、Process.start()

```java
    [->Process.java]
    public static final ProcessStartResult start(final String processClass,
                              final String niceName,
                              int uid, int gid, int[] gids,
                              int debugFlags, int mountExternal,
                              int targetSdkVersion,
                              String seInfo,
                              String abi,
                              String instructionSet,
                              String appDataDir,
                              String[] zygoteArgs) {
    try {
        return startViaZygote(processClass, niceName, uid, gid, gids,
                debugFlags, mountExternal, targetSdkVersion, seInfo,
                abi, instructionSet, appDataDir, zygoteArgs);
    } catch (ZygoteStartFailedEx ex) {
        Log.e(LOG_TAG,
                "Starting VM process through Zygote failed");
        throw new RuntimeException(
                "Starting VM process through Zygote failed", ex);
    }
}
```

然后调用了startViaZygote方法：

#### 5.3、Process.startViaZygote()

```java
    [->Process.java]
    private static ProcessStartResult startViaZygote(final String processClass,
                              final String niceName,
                              final int uid, final int gid,
                              final int[] gids,
                              int debugFlags, int mountExternal,
                              int targetSdkVersion,
                              String seInfo,
                              String abi,
                              String instructionSet,
                              String appDataDir,
                              String[] extraArgs)
                              throws ZygoteStartFailedEx {
    synchronized(Process.class) {
        ......

        return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
    }
}
```

继续查看一下zygoteSendArgsAndGetResult方法的实现：

#### 5.4、Process.zygoteSendArgsAndGetResult()

```java
    [->Process.java]
    private static ProcessStartResult zygoteSendArgsAndGetResult(
        ZygoteState zygoteState, ArrayList<String> args)
        throws ZygoteStartFailedEx {
    ......

        // Should there be a timeout on this?
        ProcessStartResult result = new ProcessStartResult();

        // Always read the entire result from the input stream to avoid leaving
        // bytes in the stream for future process starts to accidentally stumble
        // upon.
        //等待socket服务端（即zygote）返回新创建的进程pid;
        result.pid = inputStream.readInt();
        result.usingWrapper = inputStream.readBoolean();
        ......
        return result;
    ......
}
```

这个方法的主要功能是通过socket通道向Zygote进程发送一个参数列表，然后进入阻塞等待状态，直到远端的socket服务端发送回来新创建的进程pid才返回。

#### 5.5、Process.openZygoteSocketIfNeeded()

```java
   [->Process.java]
    private static ZygoteState openZygoteSocketIfNeeded(String abi) throws ZygoteStartFailedEx {
    if (primaryZygoteState == null || primaryZygoteState.isClosed()) {
        try {
            //向主zygote发起connect()操作
            primaryZygoteState = ZygoteState.connect(ZYGOTE_SOCKET);
        } catch (IOException ioe) {
            throw new ZygoteStartFailedEx("Error connecting to primary zygote", ioe);
        }
    }

    if (primaryZygoteState.matches(abi)) {
        return primaryZygoteState;
    }

    // The primary zygote didn't match. Try the secondary.
    if (secondaryZygoteState == null || secondaryZygoteState.isClosed()) {
        try {
        //当主zygote没能匹配成功，则采用第二个zygote，发起connect()操作
        secondaryZygoteState = ZygoteState.connect(SECONDARY_ZYGOTE_SOCKET);
        } catch (IOException ioe) {
            throw new ZygoteStartFailedEx("Error connecting to secondary zygote", ioe);
        }
    }

    if (secondaryZygoteState.matches(abi)) {
        return secondaryZygoteState;
    }

    throw new ZygoteStartFailedEx("Unsupported zygote ABI: " + abi);
}
```

openZygoteSocketIfNeeded(abi)方法是根据当前的abi来选择与zygote还是zygote64来进行通信。 既然system_server进程的zygoteSendArgsAndGetResult()方法通过socket向Zygote进程发送消息，这是便会唤醒Zygote进程，来响应socket客户端的请求（即system_server端) 具体详细过程可参考大神博客[理解Android进程创建流程](http://gityuan.com/2016/03/26/app-process-create/) 和 [Android四大组件与进程启动的关系](http://gityuan.com/2016/10/09/app-process-create-2/) 大神是基于Android M，之后我会跟着大神的脚步站在巨人的肩膀上，完成Android N进程创建流程，估计变化不大加深自己理解。 进程创建流程图： 
![Markdown](https://raw.githubusercontent.com/zhoujinjiani/PicGo/master/android.startactivity/Android-_Start_Activity-process-create.jpg)

总结： 可以发现其最终调用了Zygote并通过socket通信的方式让Zygote进程fork除了一个新的进程，并根据我们刚刚传递的"android.app.ActivityThread"字符串，反射出该对象并执行ActivityThread的main方法。这样我们所要启动的应用进程这时候其实已经启动了，但是还没有执行相应的初始化操作。

我们平时App-Crash常见的log就是从ActivityThread.main()抛出异常的,可参考文档：Android 7.1.2(Android N) Android系统启动流程。

> java.lang.RuntimeException: Unable to start activity ComponentInfo{......} at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2665) at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:2726) at android.app.ActivityThread.-wrap12(ActivityThread.java) at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1477)


> ``` java
> at android.os.Handler.dispatchMessage(Handler.java:102)
> at android.os.Looper.loop(Looper.java:154)
> at android.app.ActivityThread.main(ActivityThread.java:6119)
> at java.lang.reflect.Method.invoke(Native Method)
> at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:892)
> at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:782)
> ```

为什么我们平时都将ActivityThread称之为ui线程或者是主线程，这里可以看出，应用进程被创建之后首先执行的是ActivityThread的main方法，所以我们将ActivityThread成为主线程。

好了，这时候我们看一下ActivityThread的main方法的实现逻辑。

#### 5.6、ActivityThread.main()

```java
    [->ActivityThread.java]
    public static void main(String[] args) {
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
    ......
    Process.setArgV0("<pre-initialized>");

    Looper.prepareMainLooper();

    ActivityThread thread = new ActivityThread();
    thread.attach(false);

    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }

    if (false) {
        Looper.myLooper().setMessageLogging(new
                LogPrinter(Log.DEBUG, "ActivityThread"));
    }

    // End of event ActivityThreadMain.
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    Looper.loop();

    throw new RuntimeException("Main thread loop unexpectedly exited");
}
}
```

在main方法中主要执行了一些初始化的逻辑，并且创建了一个UI线程消息队列，这也就是为什么我们可以在主线程中随意的创建Handler而不会报错的原因，这里提出一个问题，大家可以思考一下：子线程可以创建Handler么？可以的话应该怎么做？ 然后执行了ActivityThread的attach方法，这里我们看一下attach方法执行了那些逻辑操作。

#### 5.7、ActivityThread.attach()

```java
    private void attach(boolean system) {
        ......
        //此时进程名还是"<pre-initialized>"
        android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
                                                UserHandle.myUserId());
        RuntimeInit.setApplicationObject(mAppThread.asBinder());
        //创建对象
        final IActivityManager mgr = ActivityManagerNative.getDefault();
        try {
            //调用基于IActivityManager接口的Binder通道
            mgr.attachApplication(mAppThread);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
        ......
}
```

#### 5.8、ActivityManagerProxy .attachApplication()

```java
[-> ActivityManagerNative.java::ActivityManagerProxy]
public void attachApplication(IApplicationThread app) throws RemoteException{
Parcel data = Parcel.obtain();
Parcel reply = Parcel.obtain();
data.writeInterfaceToken(IActivityManager.descriptor);
data.writeStrongBinder(app.asBinder());
mRemote.transact(ATTACH_APPLICATION_TRANSACTION, data, reply, 0);
reply.readException();
data.recycle();
reply.recycle();
}
```

#### 5.9、ActivityManagerNative.onTransact()

```java
[-> ActivityManagerNative.java]
public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
    throws RemoteException {
switch (code) {
...
 case ATTACH_APPLICATION_TRANSACTION: {
    data.enforceInterface(IActivityManager.descriptor);
    //获取ApplicationThread的binder代理类 ApplicationThreadProxy
    IApplicationThread app = ApplicationThreadNative.asInterface(
            data.readStrongBinder());
    if (app != null) {
        attachApplication(app); //此处是ActivityManagerService类中的方法
    }
    reply.writeNoException();
    return true;
}
}
}
```

刚刚我们已经分析过对象是ActivityManagerService的Binder client，所以这里调用了attachApplication实际上就是通过Binder机制调用了ActivityManagerService的attachApplication，具体调用的过程，我们看一下ActivityManagerService是如何实现的：

#### 5.10、ActivityManagerService.attachApplication()

```java
    [->ActivityManagerService.java]
    @Override
public final void attachApplication(IApplicationThread thread) {
    synchronized (this) {
        int callingPid = Binder.getCallingPid();
        final long origId = Binder.clearCallingIdentity();
        //此处的thread便是ApplicationThreadProxy对象,用于跟前面通过Process.start()所创建的进程中ApplicationThread对象进行通信.
        attachApplicationLocked(thread, callingPid);
        Binder.restoreCallingIdentity(origId);
    }
}
    private final boolean attachApplicationLocked(IApplicationThread thread,
        int pid) {

    // 根据pid获取ProcessRecord
    ProcessRecord app;

    if (pid != MY_PID && pid >= 0) {
        synchronized (mPidsSelfLocked) {
            app = mPidsSelfLocked.get(pid);
        }
    }
    ......
    //获取应用appInfo
        ApplicationInfo appInfo = app.instrumentationInfo != null
                ? app.instrumentationInfo : app.info;
        app.compat = compatibilityInfoForPackageLocked(appInfo);
        if (profileFd != null) {
            profileFd = profileFd.dup();
        }
        ......
        //绑定应用
        thread.bindApplication(processName, appInfo, providers, app.instrumentationClass,
                profilerInfo, app.instrumentationArguments, app.instrumentationWatcher,
                app.instrumentationUiAutomationConnection, testMode,
                mBinderTransactionTrackingEnabled, enableTrackAllocation,
                isRestrictedBackupMode || !normalMode, app.persistent,
                new Configuration(mConfiguration), app.compat,
                getCommonServicesLocked(app.isolated),
                               mCoreSettingsObserver.getCoreSettingsLocked());

        updateLruProcessLocked(app, false, null);

    } ......
    // See if the top visible activity is waiting to run in this process...
    if (normalMode) {
        try {
            if (mStackSupervisor.attachApplicationLocked(app)) {
                didSomething = true;
            }
        } catch (Exception e) {
            Slog.wtf(TAG, "Exception thrown launching activities in " + app, e);
            badApp = true;
        }
    }

    ......
    return true;
}
```

下面,再来说说thread.bindApplication的过程.

#### 5.11、ApplicationThreadProxy.bindApplication()

```java
[-> ApplicationThreadNative.java ::ApplicationThreadProxy]
class ApplicationThreadProxy implements IApplicationThread {
...
    @Override
public final void bindApplication(String packageName, ApplicationInfo info,
        List<ProviderInfo> providers, ComponentName testName, ProfilerInfo profilerInfo,
        Bundle testArgs, IInstrumentationWatcher testWatcher,
        IUiAutomationConnection uiAutomationConnection, int debugMode,
        boolean enableBinderTracking, boolean trackAllocation, boolean restrictedBackupMode,
        boolean persistent, Configuration config, CompatibilityInfo compatInfo,
        Map<String, IBinder> services, Bundle coreSettings) throws RemoteException {
    ......
    mRemote.transact(BIND_APPLICATION_TRANSACTION, data, null,
            IBinder.FLAG_ONEWAY);
    data.recycle();
}
...
}
```

#### 5.12、ApplicationThreadNative.onTransact()

```java
[-> ApplicationThreadNative.java]
public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
    throws RemoteException {
switch (code) {
...
       case BIND_APPLICATION_TRANSACTION:
    {
        ......
        bindApplication(packageName, info, providers, testName, profilerInfo, testArgs,
                testWatcher, uiAutomationConnection, testMode, enableBinderTracking,
                trackAllocation, restrictedBackupMode, persistent, config, compatInfo, services,
                coreSettings);
        return true;
    }
...
}
```

#### 5.13、ApplicationThread.bindApplication()

[-> ActivityThread.java ::ApplicationThread]

```java
    public final void bindApplication(String processName, ApplicationInfo appInfo,
        List<ProviderInfo> providers, ComponentName instrumentationName,
        ProfilerInfo profilerInfo, Bundle instrumentationArgs,
        IInstrumentationWatcher instrumentationWatcher,
        IUiAutomationConnection instrumentationUiConnection, int debugMode,
        boolean enableBinderTracking, boolean trackAllocation,
        boolean isRestrictedBackupMode, boolean persistent, Configuration config,
        CompatibilityInfo compatInfo, Map<String, IBinder> services, Bundle coreSettings) {

    if (services != null) {
        // Setup the service cache in the ServiceManager
        ServiceManager.initServiceCache(services);
    }

    setCoreSettings(coreSettings);

    AppBindData data = new AppBindData();
    ......
    sendMessage(H.BIND_APPLICATION, data);
}
```

#### 5.14、ActivityThread.handleBindApplication()

当主线程收到H.BIND_APPLICATION,则调用handleBindApplication

```java
    [-> ActivityThread.java ::H]
    private void handleBindApplication(AppBindData data) {
    ......
    mBoundApplication = data;
    mConfiguration = new Configuration(data.config);
    mCompatConfiguration = new Configuration(data.config);
    //设置进程名, 也就是说进程名是在进程真正创建以后的BIND_APPLICATION过程中才取名
    // send up app name; do this *before* waiting for debugger
    Process.setArgV0(data.processName);
    android.ddm.DdmHandleAppName.setAppName(data.processName,
                                            UserHandle.myUserId());

    ......
    获取LoadedApk对象
    data.info = getPackageInfoNoCheck(data.appInfo, data.compatInfo);


   ......
    //创建ContextImpl上下文
    final ContextImpl appContext = ContextImpl.createAppContext(this, data.info);
    updateLocaleListFromAppContext(appContext,
            mResourcesManager.getConfiguration().getLocales());

    ......
    try {
        // If the app is being launched for full backup or restore, bring it up in
        // a restricted environment with the base application class.
        // 此处data.info是指LoadedApk, 通过反射创建目标应用Application对象
        Application app = data.info.makeApplication(data.restrictedBackupMode, null);
        mInitialApplication = app;

        ......
        // Do this after providers, since instrumentation tests generally start their
        // test thread at this point, and we don't want that racing.
        try {
            mInstrumentation.onCreate(data.instrumentationArgs);
        }
        ......
        try {
            mInstrumentation.callApplicationOnCreate(app);
        }
        ......
}
```

在handleBindApplication()的过程中,会同时设置以下两个值:

LoadedApk.mApplication AT.mInitialApplication

图示总结： 
![Markdown](https://raw.githubusercontent.com/zhoujinjiani/PicGo/master/android.startactivity/N5-Android-startActivity-AMS-AT.png)

### 六、执行启动Acitivity

> ActivityStackSupervisor.attachApplicationLocked()
ActivityStackSupervisor.realStartActivityLocked()
IApplicationThread.scheduleLaunchActivity()
ActivityThread.ApplicationThread.scheduleLaunchActivity()
ActivityThread.sendMessage()
ActivityThread.H.handleMessage()
ActivityThread.handleLauncherActivity()
ActivityThread.performLauncherActivity()
Instrumentation.callActivityOnCreate()
-> Activity.performCreate() Activity.onCreate()

在第五节AMS.startProcessLocked()整个过程，创建完新进程后会在新进程中调用AMP.attachApplication ，该方法经过binder ipc后调用到AMS.attachApplicationLocked。该方法执行了一系列的初始化操作，在执行完bindApplication()之后进入ActivityStackSupervisor.attachApplicationLocked()，这样我们整个应用进程已经启动起来了。终于可以开始activity的启动逻辑了。 关系图： 
![Markdown](https://raw.githubusercontent.com/zhoujinjiani/PicGo/master/android.startactivity/N6-Android-startActivity-arc.jpg)
[ActivityThread简介](http://blog.csdn.net/myarrow/article/details/14223493) 首先看一下attachApplicationLocked方法的实现：

#### 6.1、ActivityStackSupervisor.attachApplicationLocked()

```java
    [->ActivityStackSupervisor.java]
    boolean attachApplicationLocked(ProcessRecord app) throws RemoteException {
    final String processName = app.processName;
    boolean didSomething = false;
    for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
        ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
        for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
            final ActivityStack stack = stacks.get(stackNdx);
            if (!isFocusedStack(stack)) {
                continue;
            }
            ActivityRecord hr = stack.topRunningActivityLocked();
            if (hr != null) {
                if (hr.app == null && app.uid == hr.info.applicationInfo.uid
                        && processName.equals(hr.processName)) {
                    try {
                        if (realStartActivityLocked(hr, app, true, true)) {
                            didSomething = true;
                        }
                    } ......
                }
            }
        }
    }
    if (!didSomething) {
        ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
    }
    return didSomething;
}
```

可以发现其内部调用了realStartActivityLocked方法，通过名字可以知道这个方法应该就是用来启动Activity的，看一下这个方法的实现逻辑：

#### 6.2、ActivityStackSupervisor.realStartActivityLocked()

```java
    [->ActivityStackSupervisor.java]
    final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
        boolean andResume, boolean checkConfig) throws RemoteException {

      ......
        app.forceProcessStateUpTo(mService.mTopProcessState);
        app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                System.identityHashCode(r), r.info, new Configuration(mService.mConfiguration),
                new Configuration(task.mOverrideConfig), r.compat, r.launchedFromPackage,
                task.voiceInteractor, app.repProcState, r.icicle, r.persistentState, results,
                newIntents, !andResume, mService.isNextTransitionForward(), profilerInfo);

       ......

    return true;
}
```

可以发现与第四节执行栈顶Activity onPause时类似，这里也是通过调用IApplicationThread的方法实现的，这里调用的是scheduleLaunchActivity方法，所以真正执行的是ActivityThread中的scheduleLaunchActivity。

#### 6.3、ApplicationThread.scheduleLaunchActivity()

```java
[-> ActivityThread.java :ApplicationThread]
        public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
            ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
            CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
            int procState, Bundle state, PersistableBundle persistentState,
            List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
            boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {

        updateProcessState(procState, false);

        ActivityClientRecord r = new ActivityClientRecord();

        ......
        updatePendingConfiguration(curConfig);

        sendMessage(H.LAUNCH_ACTIVITY, r);
    }
```

#### 6.4、H.handleMessage

```java
[-> ActivityThread.java ::H]
    public void handleMessage(Message msg) {
        if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
        switch (msg.what) {
            case LAUNCH_ACTIVITY: {
                final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

                r.packageInfo = getPackageInfoNoCheck(
                        r.activityInfo.applicationInfo, r.compatInfo);
                handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
            } break;
            ......
     }
```

#### 6.5、ActivityThread.handleLaunchActivity()

ActivityThread接收到SystemServer进程的消息之后会通过其内部的Handler对象分发消息，经过一系列的分发之后调用了ActivityThread的handleLaunchActivity方法：

```java
[-> ActivityThread.java]
    private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
    ......
    // Initialize before creating the activity
    WindowManagerGlobal.initialize();

    Activity a = performLaunchActivity(r, customIntent);

    if (a != null) {
        r.createdConfig = new Configuration(mConfiguration);
        reportSizeConfigurations(r);
        Bundle oldState = r.state;
        handleResumeActivity(r.token, false, r.isForward,
                !r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);
      }
      ......
}
```

可以发现这里调用了performLauncherActivity，看名字应该就是执行Activity的启动操作了......

#### 6.6、ActivityThread.performLaunchActivity()

```java
[-> ActivityThread.java]
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {

    ActivityInfo aInfo = r.activityInfo;
    if (r.packageInfo == null) {
        r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                Context.CONTEXT_INCLUDE_CODE);
    }

    ComponentName component = r.intent.getComponent();
    if (component == null) {
        component = r.intent.resolveActivity(
            mInitialApplication.getPackageManager());
        r.intent.setComponent(component);
    }

    if (r.activityInfo.targetActivity != null) {
        component = new ComponentName(r.activityInfo.packageName,
                r.activityInfo.targetActivity);
    }

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
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);

       ......
        if (activity != null) {
            Context appContext = createBaseContextForActivity(r, activity);
            ......
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window);

            ......

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
            ......
    return activity;
}
```

可以发现这里我们需要的Activity对象终于是创建出来了，然后在代码中其调用Instrumentation的callActivityOnCreate方法。

#### 6.7、Instrumentation.callActivityOnCreate()

```java
    [->Instrumentation.java]
    public void callActivityOnCreate(Activity activity, Bundle icicle,
        PersistableBundle persistentState) {
    prePerformCreate(activity);
    activity.performCreate(icicle, persistentState);
    postPerformCreate(activity);
}
```

然后执行activity的performCreate方法......

#### 6.8、Activity.performCreate()

```java
    final void performCreate(Bundle icicle, PersistableBundle persistentState) {
    restoreHasCurrentPermissionRequest(icicle);
    onCreate(icicle, persistentState);
    mActivityTransitionState.readState(icicle);
    performCreateCommon();
}
```

简要说明剩余生命周期： 回到我们的performLaunchActivity方法，其在调用了mInstrumentation.callActivityOnCreate方法之后又调用了activity.performStart()方法，看一下他的实现方式：

```java
[->Activity.java]

final void performStart() {
    mActivityTransitionState.setEnterActivityOptions(this, getActivityOptions());
    mFragments.noteStateNotSaved();
    mCalled = false;
    mFragments.execPendingActions();
    mInstrumentation.callActivityOnStart(this);
    if (!mCalled) {
        throw new SuperNotCalledException(
            "Activity " + mComponent.toShortString() +
            " did not call through to super.onStart()");
    }
    mFragments.dispatchStart();
    mFragments.reportLoaderStart();
    ......
    mActivityTransitionState.enterReady(this);
}
```

还是通过Instrumentation调用callActivityOnStart方法：

```java
[->Instrumentation.java]

public void callActivityOnStart(Activity activity) {
    activity.onStart();
}
```

然后是直接调用activity的onStart方法，第三个生命周期方法出现了，O(∩_∩)O

还是回到我们刚刚的handleLaunchActivity方法，在调用完performLaunchActivity方法之后，其有吊用了handleResumeActivity方法，好吧，看名字应该是回调Activity的onResume方法的。

```java
[-> ActivityThread.java]

    final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
        ActivityClientRecord r = mActivities.get(token);
        ......
        // TODO Push resumeArgs into the activity for consideration
        r = performResumeActivity(token, clearHide, reason);
        .....
        if (r.window == null && !a.mFinished && willBeVisible) {
                r.window = r.activity.getWindow();
                View decor = r.window.getDecorView();
                decor.setVisibility(View.INVISIBLE);
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                l.softInputMode |= forwardBit;
                if (r.mPreserveWindow) {
                    a.mWindowAdded = true;
                    r.mPreserveWindow = false;
                    // Normally the ViewRoot sets up callbacks with the Activity
                    // in addView->ViewRootImpl#setView. If we are instead reusing
                    // the decor view we have to notify the view root that the
                    // callbacks may have changed.
                    ViewRootImpl impl = decor.getViewRootImpl();
                    if (impl != null) {
                        impl.notifyChildRebuilt();
                    }
                }
                if (a.mVisibleFromClient && !a.mWindowAdded) {
                    a.mWindowAdded = true;
                    wm.addView(decor, l);
                }

            // If the window has already been added, but during resume
            // we started another activity, then don't yet make the
            // window visible.
            } else if (!willBeVisible) {
                if (localLOGV) Slog.v(
                    TAG, "Launch " + r + " mStartedActivity set");
                r.hideForNow = true;
            }
        if (!r.onlyLocalRequest) {
                r.nextIdle = mNewActivities;
                mNewActivities = r;
                if (localLOGV) Slog.v(
                    TAG, "Scheduling idle handler for " + r);
                    //线程空闲，也就是activity创建完毕之后，它会执行queueIdle里面的代码。
                Looper.myQueue().addIdleHandler(new Idler());
        }
    }
```

可以发现其resumeActivity的逻辑调用到了performResumeActivity方法，我们来看一下performResumeActivity是如何实现的。

```java
[-> ActivityThread.java]

    public final ActivityClientRecord performResumeActivity(IBinder token,
            boolean clearHide, String reason) {
        ActivityClientRecord r = mActivities.get(token);
        if (localLOGV) Slog.v(TAG, "Performing resume of " + r
                + " finished=" + r.activity.mFinished);
        if (r != null && !r.activity.mFinished) {
            ...
            try {
                ......
                r.activity.performResume();
                for (int i = mRelaunchingActivities.size() - 1; i >= 0; i--) {
                    final ActivityClientRecord relaunching = mRelaunchingActivities.get(i);
                    if (relaunching.token == r.token
                            && relaunching.onlyLocalRequest && relaunching.startsNotResumed) {
                        relaunching.startsNotResumed = false;
                    }
                }
                EventLog.writeEvent(LOG_AM_ON_RESUME_CALLED, UserHandle.myUserId(),
                        r.activity.getComponentName().getClassName(), reason);
                r.paused = false;
                r.stopped = false;
                r.state = null;
                r.persistentState = null;
            } ......
        }
        return r;
    }
```

在方法体中，最终调用了r.activity.performResume()方法，好吧，这个方法是Activity中定义的方法，我们需要在Activity中查看这个方法的具体实现：

```java
[-> Activity.java]

final void performResume() {
        performRestart();
        ...
        mInstrumentation.callActivityOnResume(this);
        ...
    }
```

可以看到第一个分支走了performRestart()，这个方法即时onRestart()生命周期。

```java
    final void performRestart() {
    ...
    mInstrumentation.callActivityOnRestart(this);
    performStart();
}
    final void performRestart() {
        mFragments.noteStateNotSaved();

        if (mToken != null && mParent == null) {
            // No need to check mStopped, the roots will check if they were actually stopped.
            WindowManagerGlobal.getInstance().setStoppedState(mToken, false /* stopped */);
        }

        if (mStopped) {
            mStopped = false;
            ......
            mCalled = false;
            mInstrumentation.callActivityOnRestart(this);
            ......
            performStart();
        }
    }
```

可以看到首先判断当前activity是否为Stopped状态，是才会走OnRestart()->Onstart()生命周期。

继续看下performResume()第二个分支，又是熟悉的味道，通过Instrumentation来调用了callActivityOnResume方法。。。

```java
[->Instrumentation.java]

public void callActivityOnResume(Activity activity) {
        activity.mResumed = true;
        activity.onResume();

        if (mActivityMonitors != null) {
            synchronized (mSync) {
                final int N = mActivityMonitors.size();
                for (int i=0; i<N; i++) {
                    final ActivityMonitor am = mActivityMonitors.get(i);
                    am.match(activity, activity, activity.getIntent());
                }
            }
        }
    }
```

O(∩_∩)O，第四个生命周期方法出现了，onResume方法。。。

终于回调onResume方法了，这时候我们的界面应该已经展示出来了，照理来说我们的Activity应该已经启动完成了，但是还没有。

有一个问题，Activity a 启动 Activity b 会触发那些生命周期方法？ 你可能会回答？b的onCreate onStart方法，onResume方法 a的onPause方法和onStop方法，onStop方法还没回调，O(∩_∩)O，对了缺少的就是对onStop方法的回调。

### 七、栈顶Activity执行onStop方法

> Looper.myQueue().addIdleHandler(new Idler())
-> Idler.queueIdle()
ActivityManagerNative.getDefault().activityIdle()
ActivityManagerService.activityIdle()
ActivityStackSupervisor.activityIdleInternalLocked()
ActivityStack.stopActivityLocked()
IApplicationThread.scheduleStopActivity()
ActivityThread.scheduleStopActivity()
-> ActivityThread.sendMessage()
ActivityThread.H.sendMessage()
-> ActivityThread.H.handleMessage()
ActivityThread.handleStopActivity()
ActivityThread.performStopActivityInner()
ActivityThread.callCallActivityOnSaveInstanceState()
Instrumentation.callActivityOnSaveInstanceState()
Activity.performSaveInstanceState()
-> Activity.onSaveInstanceState()
Activity.performStop()
-> Instrumentation.callActivityOnStop()
Activity.onStop()

回到我们的handleResumeActivity方法，在方法体最后有这样的一代码：

```java
Looper.myQueue().addIdleHandler(new Idler());
```

这段代码是异步消息机制相关的代码，我们可以看一下Idler对象的具体实现：

```java
private class Idler implements MessageQueue.IdleHandler {
        @Override
        public final boolean queueIdle() {
            ActivityClientRecord a = mNewActivities;
            .....
            if (a != null) {
                mNewActivities = null;
                IActivityManager am = ActivityManagerNative.getDefault();
                ActivityClientRecord prev;
                do {
                    ......
                    if (a.activity != null && !a.activity.mFinished) {
                        try {
                            am.activityIdle(a.token, a.createdConfig, stopProfiling);
                            a.createdConfig = null;
                        } catch (RemoteException ex) {
                            // Ignore
                        }
                    }
                    prev = a;
                    a = a.nextIdle;
                    prev.nextIdle = null;
                } while (a != null);
            }
        }
    }
```

这样当Messagequeue执行add方法之后就会回调其queueIdle()方法，我们可以看到在方法体中其调用了ActivityManagerNative.getDefault().activityIdle()，好吧，熟悉了Binder机制以后我们知道这段代码会执行到ActivityManagerService的activityIdle方法：

```java
@Override
public final void activityIdle(IBinder token, Configuration config, boolean stopProfiling) {
    final long origId = Binder.clearCallingIdentity();
    synchronized (this) {
        ActivityStack stack = ActivityRecord.getStackLocked(token);
        if (stack != null) {
            ActivityRecord r =
                    mStackSupervisor.activityIdleInternalLocked(token, false, config);
            if (stopProfiling) {
                if ((mProfileProc == r.app) && (mProfileFd != null)) {
                    try {
                        mProfileFd.close();
                    } catch (IOException e) {
                    }
                    clearProfilerLocked();
                }
            }
        }
    }
    Binder.restoreCallingIdentity(origId);
}
```

然后在activityIdle方法中又调用了ActivityStackSupervisor.activityIdleInternalLocked方法：

```java
[->ActivityStackSupervisor.java]

final ActivityRecord activityIdleInternalLocked(final IBinder token, boolean fromTimeout,
            Configuration config) {
        ...
        for (int i = 0; i < NS; i++) {
            r = stops.get(i);
            final ActivityStack stack = r.task.stack;
            if (stack != null) {
                if (r.finishing) {
                    stack.finishCurrentActivityLocked(r, ActivityStack.FINISH_IMMEDIATELY, false);
                } else {
                    stack.stopActivityLocked(r);
                }
            }
        }
        ...
        return r;
    }
```

可以发现在其中又调用了ActivityStack.stopActivityLocked方法：

```java
final void stopActivityLocked(ActivityRecord r) {
        if ((r.intent.getFlags()&Intent.FLAG_ACTIVITY_NO_HISTORY) != 0
                || (r.info.flags&ActivityInfo.FLAG_NO_HISTORY) != 0) {
            ...
                r.app.thread.scheduleStopActivity(r.appToken, r.visible, r.configChangeFlags);
             ...
        }
    }
```

好吧，又是相同的逻辑通过IApplicationThread.scheduleStopActivity,最终调用了ActivityThread.scheduleStopActivity()方法。。。。

```java
public final void scheduleStopActivity(IBinder token, boolean showWindow,
                int configChanges) {
           sendMessage(
                showWindow ? H.STOP_ACTIVITY_SHOW : H.STOP_ACTIVITY_HIDE,
                token, 0, configChanges);
        }
```

然后执行sendMessage方法，最终执行H（Handler）的sendMessage方法，并被H的handleMessge方法接收执行handleStopActivity方法。。。

```java

private void handleStopActivity(IBinder token, boolean show, int configChanges) {
        ...
        performStopActivityInner(r, info, show, true);
        ...
    }
```

然后我们看一下performStopActivityInner的实现逻辑：

```java
[->ActivityThread.java]

private void performStopActivityInner(ActivityClientRecord r,
            StopInfo info, boolean keepShown, boolean saveState) {
            ...
            if (!keepShown) {
                try {
                    r.activity.performStop();
                } catch (Exception e) {
                    ......
                }
                r.stopped = true;
            }
        }
    }
```

我们看一下performStopActivityInner中调用到的Activity方法的performStop方法

```java
final void performStop() {
        if (!mStopped) {
            ......
            mFragments.dispatchStop();
            mCalled = false;
            mInstrumentation.callActivityOnStop(this);
            ......
            mStopped = true;
        }
        mResumed = false;
    }
```

还是通过Instrumentation来实现的，调用了它的callActivityOnStop方法。。

```java
public void callActivityOnStop(Activity activity) {
        activity.onStop();
    }
```

生命周期方法onStop()出来了。

我们来看一下Activity 的生命周期：

> **protected void onCreate();**
**protected void onRestart();**
**protected void onStart();**
**protected void onResume();**
**protected void onPause();**
**protected void onStop();**
**protected void onDestory();**

前面我们分析了onCreate()、onStart()、onRestart() 、onResume()、onPause()、onStop()。 Activity 销毁时的 onDestroy() 回调都与前面的过程大同小异，这里就只列举相应的方法栈，不再继续描述。

> Activity.finish()
ActivityManagerNative.getDefault().finishActivity()
ActivityManagerService.finishActivity()
ActivityStack.requestFinishActivityLocked()
ActivityStack.finishActivityLocked()
ActivityStack.startPausingLocked()
参考：[Android源码解析之（十五）-->Activity销毁流程](http://blog.csdn.net/qq_23547831/article/details/51232309)

#### 启动流程：

![Markdown](https://raw.githubusercontent.com/zhoujinjiani/PicGo/master/android.startactivity/Android-start_activity_process.jpg)

1、点击桌面App图标，Launcher进程采用Binder IPC向system_server进程发起startActivity请求；
2、system_server进程接收到请求后，向zygote进程发送创建进程的请求；
3、Zygote进程fork出新的子进程，即App进程；
4、App进程，通过Binder IPC向sytem_server进程发起attachApplication请求；
5、system_server进程在收到请求后，进行一系列准备工作后，再通过binder IPC向App进程发送scheduleLaunchActivity请求；
6、App进程的binder线程（ApplicationThread）在收到请求后，通过handler向主线程发送LAUNCH_ACTIVITY消息；
7、主线程在收到Message后，通过发射机制创建目标Activity，并回调Activity.onCreate()等方法。

到此，App便正式启动，开始进入Activity生命周期，执行完onCreate/onStart/onResume方法，UI渲染结束后便可以看到App的主界面。 启动Activity较为复杂，后续介绍窗口加载渲染过程，可参考文档：【Android 7.1.2 (Android N) Activity-Window加载显示流程分析】

### 参考文档(特别感谢)：

[Android源码解析之（十四）-->Activity启动流程](http://blog.csdn.net/qq_23547831/article/details/51224992)
[Android Activity启动过程分析](http://gityuan.com/2016/03/12/start-activity/)
[Android源码解析之（十五）-->Activity销毁流程](http://blog.csdn.net/qq_23547831/article/details/51232309)
[凯子哥带你学Framework–Activity启动过程全解析](http://blog.csdn.net/zhaokaiqiang1992/article/details/49428287)
[深入理解Activity启动流程(一)–Activity启动的概要流程](http://www.cloudchou.com/android/post-788.html)
[Android 7.0 ActivityManagerService 启动Activity的过程 系列](http://blog.csdn.net/Gaugamela/article/category/6383486)
[Activity生命周期的回调，你应该知道得更多！--Android源码剖析（上）](http://blog.csdn.net/yalinfendou/article/details/46909173)
[Activity生命周期的回调，你应该知道得更多！--Android源码剖析（下）](http://blog.csdn.net/yalinfendou/article/details/46910811)
