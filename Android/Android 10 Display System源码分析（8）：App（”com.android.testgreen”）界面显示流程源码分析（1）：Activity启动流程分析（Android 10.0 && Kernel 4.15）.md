---
title:  Android 10 Display System源码分析（8）：App（"com.android.testgreen"）界面显示流程源码分析（1）：Activity启动流程分析（Android 10.0 && Kernel 4.15）
cover: https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/post.cover.pictures/bing-wallpaper-2018.04.29.jpg
categories: 
  - Display
tags:
  - Android
  - Linux
  - Graphics
toc: true
abbrlink: 20211010
date: 2021-10-10 09:25:00
---


###### 准备
我们首先去除掉系统的：
``` cpp
adb root
adb remount
adb shell
rk3399_Android10:/system/bin # rm -rf bootanimation
rk3399_Android10:/product/priv-app # rm -rf SystemUI/
rk3399_Android10:/product/priv-app # rm -rf Launcher3QuickStep/
```
然后等待开机完成App（"com.android.testgreen"），抓取Log。setprop vendor.dump true。抓取的帧会按数字排列，还带分辨率参数。adb pull /data/dump/。抓到的bin文件可以用软件7yuv打开查看，格式设定为RGBA8888
App（"com.android.testgreen"）渲染图：
![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android10.Display.8/dmlayer_fb_com.android.testred.png)
FrameBuffer渲染图：
![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android10.Display.8/dmlayer_fb_com.android.testred.png)

效果图：
![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android10.Display.8/Android10.com.android.test.takepic.jpg)

----------
==源码（部分）==：
**packages/apps/testViewportGreen && testViewportBlue && testViewportRed/**
**frameworks/base/core/java/android/app/**
**frameworks/base/services/core/java/com/android/server/wm/**
**frameworks/base/services/core/java/com/android/server/am/**
**frameworks/base/services/core/java/com/android/server/policy/**
**frameworks/native/services/surfaceflinger/**
**frameworks/native/libs/gui/**
**frameworks/native/libs/ui/**

----------
####  一、Activity概述
基于Android 10.0的源码剖析， 分析Android Activity启动流程：

----------

##### 1.1、Activity的管理机制

Android的管理主要是通过Activity栈来进行的。当一个Activity启动时，系统根据其配置或调用的方式，将Activity压入一个特定的栈中，系统处 于运行（Running or Resumed）状态。当按Back键或触发finish()方法时，Activity会从栈中被压出，进而被销毁，当有新的Activity压入栈时， 如果原Activity仍然可见，则原Activity的状态将转变为暂停（Paused）状态，如果原Activity完全被遮挡，那么其状态将转变为 停止（Stopped）。

###### 1.1.1、Task和Stack
Android系统中的每一个Activity都位于一个Task中。一个Task可以包含多个Activity，同一个Activity也可能有多个实例。 在AndroidManifest.xml中，我们可以通过android:launchMode来控制Activity在Task中的实例。

###### 1.1.2、启动模式的作用
Activity启动模式就是属于Activity配置属性之一，叫它具有四种启动模式，分别是：1.standard ，2.singleTop，3.singleTask，4.singleInstance，一般如果不显示声明，默认为standard模式。launchMode 在多个Activity跳转的过程中扮演着重要的角色，它可以决定是否生成新的Activity实例，是否重用已存在的Activity实例，是否和其他 Activity实例公用一个task里。
###### 1.1.3、启动模式的作用
**模式说明**
- standard模式：这是系统默认的启动模式，这种模式就是创建一个Activity压入Task容器栈中，当当前Activity激活，并处在和用户交互时，此Activity弹出栈顶，当finish()时候，在任务栈中销毁该实例。
- singleTop模式：这种模式首先会判断要激活的Activity是否在栈顶，如果在则不重新创建新的实例，复用当前的实例，如果不在栈顶，则在任务栈中创建实例。条件是是否在栈顶，而不是是否在栈中。
- singleTask模式： 这种模式启 动的目标Activity实例如果已经存在task容器栈中，不管当前实例处于栈的任何位置，是栈顶也好，栈底也好，还是处于栈中间，只要目标 Activity实例处于task容器栈中，都可以重用该Activity实例对象，然后，把处于该Activity实例对象上面全部Activity实 例清除掉，并且，task容器栈中永远只有唯一实例对象，不会存在两个相同的实例对象。所以，如果你想你的应用不管怎么启动目标Activity，都只有 唯一一个实例对象，就使用这种启动模式。
- singleInstance模式：当该模式Activity实例在任务栈中创建后，只要该实例还在任务栈中，即只要激活的是该类型的Activity，都会通过调用实例的newInstance()方法重用该Activity，此时使用的都是同一个Activity实例，它都会处于任务栈的栈顶。此模式一般用于加载较慢的，比较耗性能且不需要每次都重新创建的Activity。

###### 1.1.4、ActivityStack & TaskRecord & ActivityRecord 关系
Back栈管理了当你在Activity上点击Back键，当前Activity销毁后应该跳转到哪一个Activity的逻辑。

其实在ActivityManagerService与WindowManagerService内部管理中，在Task之外，还有一层容器，这个容器应用开发者和用户可能都不会感觉到或者用到，但它却非常重要，那就是ActivityStack。 下文中，我们将看到，Android系统中的多窗口管理，就是建立在Stack的数据结构上的。 一个ActivityStack中包含了多个TaskRecord，一个TaskRecord中包含了多个ActivityRecord，下图描述了它们的关系：
![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android10.Display.8/com.android.testred.1.ActivityStack_TaskRecord_ActivityRecord.png)


另外还有一点需要注意的是，ActivityManagerService和WindowManagerService中的Task和Stack结构是一一对应的，对应关系对于如下：

> ActivityStack <–> TaskStack
> TaskRecord   <–> Task

即，ActivityManagerService中的每一个ActivityStack或者TaskRecord在WindowManagerService中都有对应的TaskStack和Task，这两类对象都有唯一的id（id是int类型），它们通过id进行关联。


##### 1.2、ActivityManagerService相关类简介

- Instrumentation
用于实现应用程序测试代码的基类。当在打开仪器的情况下运行时，这个类将在任何应用程序代码之前为您实例化，允许您监视系统与应用程序的所有交互。可以通过AndroidManifest.xml的标签描述该类的实现。

- ActivityManager
该类提供与Activity、Service和Process相关的信息以及交互方法， 可以被看作是ActivityManagerService的辅助类。

- ActivityManagerService
Android中最核心的服务，主要负责系统中四大组件的启动、切换、调度及应用程序的管理和调度等工作。

- ActivityThread
管理应用程序进程中主线程的执行，根据Activity管理者的请求调度和执行activities、broadcasts及其相关的操作。

- ActivityStack
负责单个Activity栈的状态和管理。

- ActivityStackSupervisor
负责所有Activity栈的管理。内部管理了mHomeStack、mFocusedStack和mLastFocusedStack三个Activity栈。其中，mHomeStack管理的是Launcher相关的Activity栈；mFocusedStack管理的是当前显示在前台Activity的Activity栈；mLastFocusedStack管理的是上一次显示在前台Activity的Activity栈。

- ClientLifecycleManager
用来管理多个客户端生命周期执行请求的管理类。

- Activity
用来管理客户端视图、点击、输入等功能的抽象，具有完整的生命周期

- ViewRootImpl
 APP进程与system_server进程进行视图相关通信的桥梁，同时也管理着APP进程视图方面操作的主要逻辑

- ActivityRecord
Activity的信息记录在ActivityRecord对象, 并通过通过成员变量task指向TaskRecord

>ProcessRecord app //跑在哪个进程
TaskRecord task //跑在哪个task
ActivityInfo info // Activity信息
ActivityState state //Activity状态
ApplicationInfo appInfo //跑在哪个app
ComponentName realActivity //组件名
String packageName //包名
String processName //进程名
int launchMode //启动模式
int userId // 该Activity运行在哪个用户id

- ActivityState：

>INITIALIZING
RESUMED：已恢复
PAUSING
PAUSED：已暂停
STOPPING
STOPPED：已停止
FINISHING
DESTROYING
DESTROYED：已销毁


- TaskRecord
Task的信息记录在TaskRecord对象.

>ActivityStack stack; //当前所属的stack
ArrayList<ActivityRecord> mActivities;; // 当前task的所有Activity列表
int taskId
String affinity； 是指root activity的affinity，即该Task中第一个Activity;
int mCallingUid;
String mCallingPackage； //调用者的包名

- ActivityStack

>ArrayList<TaskRecord> mTaskHistory //保存所有的Task列表
final int mStackId;
int mDisplayId;
ActivityRecord mPausingActivity //正在pause
ActivityRecord mLastPausedActivity
ActivityRecord mResumedActivity //已经resumed
ActivityRecord mLastStartedActivity

所有前台stack的mResumedActivity的state == RESUMED, 则表示allResumedActivitiesComplete, 此时mLastFocusedStack = mFocusedStack;

- ActivityStackSupervisor

>ActivityStack mHomeStack //桌面的stack
ActivityStack mFocusedStack //当前聚焦stack
ActivityStack mLastFocusedStack //正在切换
SparseArray mActivityDisplays //displayId为key
SparseArray mActivityContainers // mStackId为key

##### 1.3、相关类重要成员变量
![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android10.Display.8/com.android.testred.1.ams_class_member.png)



##### 1.4、小结：

总体概览图：
![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android10.Display.8/com.android.testred.1.startActivity_fork_new_process.png)

1、点击桌面App图标，Launcher进程采用Binder IPC向system_server进程发起startActivity请求；
2、system_server进程接收到请求后，向zygote进程发送创建进程的请求；
3、Zygote进程fork出新的子进程，即App进程；
4、App进程，通过Binder IPC向sytem_server进程发起attachApplication请求；
5、system_server进程在收到请求后，进行一系列准备工作后，再通过binder IPC向App进程发送scheduleLaunchActivity请求；
6、App进程的binder线程（ApplicationThread）在收到请求后，通过handler向主线程发送LAUNCH_ACTIVITY消息；
7、主线程在收到Message后，通过发射机制创建目标Activity，并回调Activity.onCreate()等方法。


####  二、APP请求启动Activity（普通app启动流程）
##### 2.1、Activity.startActivity()（普通app启动流程）
从Launcher启动应用的时候，经过调用会执行Activity中的startActivity。(我们这里testred就是launcher)
``` java
X:\frameworks\base\core\java\android\app\Activity.java
    @Override
    public void startActivity(Intent intent) {
        this.startActivity(intent, null);
    }
    @Override
    public void startActivity(Intent intent, @Nullable Bundle options) {
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            startActivityForResult(intent, -1);
        }
    }
```
##### 2.2、Activity.startActivityForResult()（普通app启动流程）

``` java
X:\frameworks\base\core\java\android\app\Activity.java
    @Override
    @UnsupportedAppUsage
    public void startActivityForResult(
            String who, Intent intent, int requestCode, @Nullable Bundle options) {
        Uri referrer = onProvideReferrer();
        if (referrer != null) {
            intent.putExtra(Intent.EXTRA_REFERRER, referrer);
        }
        options = transferSpringboardActivityOptions(options);
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, who,
                intent, requestCode, options);
        if (ar != null) {
            mMainThread.sendActivityResult(
                mToken, who, requestCode,
                ar.getResultCode(), ar.getResultData());
        }
        cancelInputsAndStartExitTransition(options);
    }
```
##### 2.3、Instrumentation.execStartActivity()（普通app启动流程）

``` java
X:\frameworks\base\core\java\android\app\Instrumentation.java
    @UnsupportedAppUsage
    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, String resultWho,
            Intent intent, int requestCode, Bundle options, UserHandle user) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        ......
        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess(who);
            int result = ActivityTaskManager.getService()
                .startActivityAsUser(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, resultWho,
                        requestCode, 0, null, options, user.getIdentifier());
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }

```
ActivityTaskManager.getService()返回一个android.app.IActivityManager.Stub.Proxy对象(请参考Android9.0activity启动分析)
> IActivityManager.aidl -> Binder机制 -> ActivityManagerService

#### 三、 ActivityManagerService接收启动Activity的请求

##### 3.1、ActivityManagerService.startActivity()（普通app启动流程）

``` java
X:\frameworks\base\services\core\java\com\android\server\am\ActivityManagerService.java
    @Override
    public int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
        return mActivityTaskManager.startActivity(caller, callingPackage, intent, resolvedType,
                resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions);
    }

    @Override
    public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {

            return mActivityTaskManager.startActivityAsUser(caller, callingPackage, intent,
                    resolvedType, resultTo, resultWho, requestCode, startFlags, profilerInfo,
                    bOptions, userId);
    }

    WaitResult startActivityAndWait(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
            return mActivityTaskManager.startActivityAndWait(caller, callingPackage, intent,
                    resolvedType, resultTo, resultWho, requestCode, startFlags, profilerInfo,
                    bOptions, userId);
    }

```

##### 3.2、ActivityTaskManagerService.startActivityAndWait()（普通app启动流程）

``` java
X:\frameworks\base\services\core\java\com\android\server\wm\ActivityTaskManagerService.java
    @Override
    public final WaitResult startActivityAndWait(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
        final WaitResult res = new WaitResult();
        synchronized (mGlobalLock) {
            enforceNotIsolatedCaller("startActivityAndWait");
            userId = handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(),
                    userId, "startActivityAndWait");
            // TODO: Switch to user app stacks here.
            getActivityStartController().obtainStarter(intent, "startActivityAndWait")
                    .setCaller(caller)
                    .setCallingPackage(callingPackage)
                    .setResolvedType(resolvedType)
                    .setResultTo(resultTo)
                    .setResultWho(resultWho)
                    .setRequestCode(requestCode)
                    .setStartFlags(startFlags)
                    .setActivityOptions(bOptions)
                    .setMayWait(userId)
                    .setProfilerInfo(profilerInfo)
                    .setWaitResult(res)
                    .execute();
        }
        return res;
    }

X:\frameworks\base\services\core\java\com\android\server\wm\ActivityStartController.java
    ActivityStarter obtainStarter(Intent intent, String reason) {
        return mFactory.obtain().setIntent(intent).setReason(reason);
    }
```


##### 3.3、ActivityStarter.execute()（普通app启动流程）

``` java
X:\frameworks\base\services\core\java\com\android\server\wm\ActivityStarter.java
int execute() {
	try {
		// TODO(b/64750076): Look into passing request directly to these methods to allow
		// for transactional diffs and preprocessing.
		if (mRequest.mayWait) {
			return startActivityMayWait(mRequest.caller, mRequest.callingUid,
					mRequest.callingPackage, mRequest.realCallingPid, mRequest.realCallingUid,
					mRequest.intent, mRequest.resolvedType,
					mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
					mRequest.resultWho, mRequest.requestCode, mRequest.startFlags,
					mRequest.profilerInfo, mRequest.waitResult, mRequest.globalConfig,
					mRequest.activityOptions, mRequest.ignoreTargetSecurity, mRequest.userId,
					mRequest.inTask, mRequest.reason,
					mRequest.allowPendingRemoteAnimationRegistryLookup,
					mRequest.originatingPendingIntent, mRequest.allowBackgroundActivityStart);
		} else {
			return startActivity(mRequest.caller, mRequest.intent, mRequest.ephemeralIntent,
					mRequest.resolvedType, mRequest.activityInfo, mRequest.resolveInfo,
					mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
					mRequest.resultWho, mRequest.requestCode, mRequest.callingPid,
					mRequest.callingUid, mRequest.callingPackage, mRequest.realCallingPid,
					mRequest.realCallingUid, mRequest.startFlags, mRequest.activityOptions,
					mRequest.ignoreTargetSecurity, mRequest.componentSpecified,
					mRequest.outActivity, mRequest.inTask, mRequest.reason,
					mRequest.allowPendingRemoteAnimationRegistryLookup,
					mRequest.originatingPendingIntent, mRequest.allowBackgroundActivityStart);
		}
	} finally {
		onExecutionComplete();
	}
}
```


>这个方法是基于初始的参数来启动activity,可以看到有两种启动activity的代码块,startActivityMayWait最终会调用startActivity所以:我们来看startActivityMayWait源码


##### 3.4、App（"com.android.testgreen"）（Launcher App启动流程）

Android 开机会启动一个com.android.settings/.FallbackHome，当FallbackHome Paused的时候，此时系统已经没有Activity，ActivityStack 就会调用方法resumeNextFocusableActivityWhenStackIsEmpty。

``` java
11-12 03:15:29.679   433   916 V ActivityStack_Pause: Activity paused: token=Token{484d3e0 ActivityRecord{fd3e474 u0 com.android.settings/.FallbackHome t1 f}}, timeout=false
11-12 03:15:29.679   433   916 V ActivityStack_States: Moving to PAUSED: ActivityRecord{fd3e474 u0 com.android.settings/.FallbackHome t1 f} (pause complete)
11-12 03:15:29.679   433   916 V ActivityStack_Pause: Complete pause: ActivityRecord{fd3e474 u0 com.android.settings/.FallbackHome t1 f}
11-12 03:15:29.679   433   916 V ActivityRecord_States: State movement: ActivityRecord{fd3e474 u0 com.android.settings/.FallbackHome t1 f} from:PAUSING to:PAUSED reason:completePausedLocked
11-12 03:15:29.679   433   916 V ActivityStack_Pause: Executing finish of activity: ActivityRecord{fd3e474 u0 com.android.settings/.FallbackHome t1 f}

11-12 03:15:29.679   433   916 V ActivityStack_States: Moving to FINISHING: ActivityRecord{fd3e474 u0 com.android.settings/.FallbackHome t1 f}
11-12 03:15:29.679   433   916 V ActivityRecord_States: State movement: ActivityRecord{fd3e474 u0 com.android.settings/.FallbackHome t1 f} from:PAUSED to:FINISHING reason:finishCurrentActivityLocked
11-12 03:15:29.679   433   916 V ActivityStack: Enqueueing pending finish: ActivityRecord{fd3e474 u0 com.android.settings/.FallbackHome t1 f}

11-12 03:15:29.680   433   916 D ActivityStack_States: resumeNextFocusableActivityWhenStackIsEmpty: noMoreActivities, go home
```

此时就是我们的主脚上场，App（"com.android.testgreen"）启动，我们看看调用Stack。


``` java
11-12 03:15:29.701   433   916 I ActivityStack: java.lang.RuntimeException: here
11-12 03:15:29.701   433   916 I ActivityStack: 	at com.android.server.wm.ActivityStack.startActivityLocked(ActivityStack.java:3254)
11-12 03:15:29.701   433   916 I ActivityStack: 	at com.android.server.wm.ActivityStarter.startActivityUnchecked(ActivityStarter.java:1718)
11-12 03:15:29.701   433   916 I ActivityStack: 	at com.android.server.wm.ActivityStarter.startActivity(ActivityStarter.java:1404)
11-12 03:15:29.701   433   916 I ActivityStack: 	at com.android.server.wm.ActivityStarter.startActivity(ActivityStarter.java:935)
11-12 03:15:29.701   433   916 I ActivityStack: 	at com.android.server.wm.ActivityStarter.startActivity(ActivityStarter.java:585)
11-12 03:15:29.701   433   916 I ActivityStack: 	at com.android.server.wm.ActivityStarter.execute(ActivityStarter.java:527)
11-12 03:15:29.701   433   916 I ActivityStack: 	at com.android.server.wm.ActivityStartController.startHomeActivity(ActivityStartController.java:186)
11-12 03:15:29.701   433   916 I ActivityStack: 	at com.android.server.wm.RootActivityContainer.startHomeOnDisplay(RootActivityContainer.java:403)
11-12 03:15:29.701   433   916 I ActivityStack: 	at com.android.server.wm.RootActivityContainer.startHomeOnDisplay(RootActivityContainer.java:352)
11-12 03:15:29.701   433   916 I ActivityStack: 	at com.android.server.wm.RootActivityContainer.resumeHomeActivity(RootActivityContainer.java:533)
11-12 03:15:29.701   433   916 I ActivityStack: 	at com.android.server.wm.ActivityStack.resumeNextFocusableActivityWhenStackIsEmpty(ActivityStack.java:3131)
11-12 03:15:29.701   433   916 I ActivityStack: 	at com.android.server.wm.ActivityStack.resumeTopActivityInnerLocked(ActivityStack.java:2700)
11-12 03:15:29.701   433   916 I ActivityStack: 	at com.android.server.wm.ActivityStack.resumeTopActivityUncheckedLocked(ActivityStack.java:2603)
11-12 03:15:29.701   433   916 I ActivityStack: 	at com.android.server.wm.RootActivityContainer.resumeFocusedStacksTopActivities(RootActivityContainer.java:1193)
11-12 03:15:29.701   433   916 I ActivityStack: 	at com.android.server.wm.RootActivityContainer.resumeFocusedStacksTopActivities(RootActivityContainer.java:1146)
11-12 03:15:29.701   433   916 I ActivityStack: 	at com.android.server.wm.ActivityStack.finishCurrentActivityLocked(ActivityStack.java:4240)
11-12 03:15:29.701   433   916 I ActivityStack: 	at com.android.server.wm.ActivityStack.completePauseLocked(ActivityStack.java:1806)
11-12 03:15:29.701   433   916 I ActivityStack: 	at com.android.server.wm.ActivityStack.activityPausedLocked(ActivityStack.java:1772)
11-12 03:15:29.701   433   916 I ActivityStack: 	at com.android.server.wm.ActivityTaskManagerService.activityPaused(ActivityTaskManagerService.java:1725)
11-12 03:15:29.701   433   916 I ActivityStack: 	at android.app.IActivityTaskManager$Stub.onTransact(IActivityTaskManager.java:1981)
11-12 03:15:29.701   433   916 I ActivityStack: 	at android.os.Binder.execTransactInternal(Binder.java:1021)
11-12 03:15:29.701   433   916 I ActivityStack: 	at android.os.Binder.execTransact(Binder.java:994)

```

可以看到也会执行ActivityStarter.execute()来继续执行。所以我们继续分析代码。

##### 3.5、ActivityStarter.startActivity()

``` java
X:\frameworks\base\services\core\java\com\android\server\wm\ActivityStarter.java
    private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
            String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
            String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
            SafeActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
            ActivityRecord[] outActivity, TaskRecord inTask, String reason,
            boolean allowPendingRemoteAnimationRegistryLookup,
            PendingIntentRecord originatingPendingIntent, boolean allowBackgroundActivityStart) {
        ......
        mLastStartReason = reason;
        mLastStartActivityTimeMs = System.currentTimeMillis();
        mLastStartActivityRecord[0] = null;

        mLastStartActivityResult = startActivity(caller, intent, ephemeralIntent, resolvedType,
                aInfo, rInfo, voiceSession, voiceInteractor, resultTo, resultWho, requestCode,
                callingPid, callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
                options, ignoreTargetSecurity, componentSpecified, mLastStartActivityRecord,
                inTask, allowPendingRemoteAnimationRegistryLookup, originatingPendingIntent,
                allowBackgroundActivityStart);

        if (outActivity != null) {
            // mLastStartActivityRecord[0] is set in the call to startActivity above.
            outActivity[0] = mLastStartActivityRecord[0];
        }

        return getExternalResult(mLastStartActivityResult);
    }

    private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
            String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
            String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
            SafeActivityOptions options,
            boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity,
            TaskRecord inTask, boolean allowPendingRemoteAnimationRegistryLookup,
            PendingIntentRecord originatingPendingIntent, boolean allowBackgroundActivityStart) {
        mSupervisor.getActivityMetricsLogger().notifyActivityLaunching(intent);
        int err = ActivityManager.START_SUCCESS;
        ......
        if (err == ActivityManager.START_SUCCESS) {
            Slog.i(TAG, "START u" + userId + " {" + intent.toShortString(true, true, true, false)
                    + "} from uid " + callingUid);
------------------------------------------------------------------------------------------------------
对应Log：11-12 03:15:29.681   433   916 I ActivityStarter: START u0 {act=android.intent.action.MAIN cat=[android.intent.category.HOME] flg=0x10000100 cmp=com.android.testred/.TestActivity} from uid 0
------------------------------------------------------------------------------------------------------
        }
        //创建ActivityRecord
        ActivityRecord r = new ActivityRecord(mService, callerApp, callingPid, callingUid,
                callingPackage, intent, resolvedType, aInfo, mService.getGlobalConfiguration(),
                resultRecord, resultWho, requestCode, componentSpecified, voiceSession != null,
                mSupervisor, checkedOptions, sourceRecord);
        ......

        final int res = startActivity(r, sourceRecord, voiceSession, voiceInteractor, startFlags,
                true /* doResume */, checkedOptions, inTask, outActivity, restrictedBgActivity);
        mSupervisor.getActivityMetricsLogger().notifyActivityLaunched(res, outActivity[0]);
        return res;
    }

    private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
                IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
                ActivityRecord[] outActivity, boolean restrictedBgActivity) {
        int result = START_CANCELED;
        final ActivityStack startedActivityStack;
        try {
            mService.mWindowManager.deferSurfaceLayout();
            result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
                    startFlags, doResume, options, inTask, outActivity, restrictedBgActivity);
        } finally {
        ......
        }

        postStartActivityProcessing(r, result, startedActivityStack);

        return result;
    }

```


##### 3.6、ActivityStarter.startActivityUnchecked()

``` java
// Note: This method should only be called from {@link startActivity}.
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
		IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
		int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
		ActivityRecord[] outActivity, boolean restrictedBgActivity) {
    ......

	boolean newTask = false;
	final TaskRecord taskToAffiliate = (mLaunchTaskBehind && mSourceRecord != null)
			? mSourceRecord.getTaskRecord() : null;

	// Should this be considered a new task?
	int result = START_SUCCESS;
	if (mStartActivity.resultTo == null && mInTask == null && !mAddingToTask
			&& (mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) {
		newTask = true;
		result = setTaskFromReuseOrCreateNewTask(taskToAffiliate);
	} else if (mSourceRecord != null) {
		result = setTaskFromSourceRecord();
	} else if (mInTask != null) {
		result = setTaskFromInTask();
	} else {
		// This not being started from an existing activity, and not part of a new task...
		// just put it in the top task, though these days this case should never happen.
		result = setTaskToCurrentTopOrCreateNewTask();
	}
	if (result != START_SUCCESS) {
		return result;
	}

	mService.mUgmInternal.grantUriPermissionFromIntent(mCallingUid, mStartActivity.packageName,
			mIntent, mStartActivity.getUriPermissionsLocked(), mStartActivity.mUserId);
	mService.getPackageManagerInternalLocked().grantEphemeralAccess(
			mStartActivity.mUserId, mIntent, UserHandle.getAppId(mStartActivity.appInfo.uid),
			UserHandle.getAppId(mCallingUid));
	if (newTask) {
		EventLog.writeEvent(EventLogTags.AM_CREATE_TASK, mStartActivity.mUserId,
				mStartActivity.getTaskRecord().taskId);
	}
	ActivityStack.logStartActivity(
			EventLogTags.AM_CREATE_ACTIVITY, mStartActivity, mStartActivity.getTaskRecord());
	mTargetStack.mLastPausedActivity = null;

	mRootActivityContainer.sendPowerHintForLaunchStartIfNeeded(
			false /* forceSend */, mStartActivity);

	mTargetStack.startActivityLocked(mStartActivity, topFocused, newTask, mKeepCurTransition,
			mOptions);
	if (mDoResume) {
		final ActivityRecord topTaskActivity =
				mStartActivity.getTaskRecord().topRunningActivityLocked();
		if (!mTargetStack.isFocusable()
				|| (topTaskActivity != null && topTaskActivity.mTaskOverlay
				&& mStartActivity != topTaskActivity)) {
			// If the activity is not focusable, we can't resume it, but still would like to
			// make sure it becomes visible as it starts (this will also trigger entry
			// animation). An example of this are PIP activities.
			// Also, we don't want to resume activities in a task that currently has an overlay
			// as the starting activity just needs to be in the visible paused state until the
			// over is removed.
			mTargetStack.ensureActivitiesVisibleLocked(mStartActivity, 0, !PRESERVE_WINDOWS);
			// Go ahead and tell window manager to execute app transition for this activity
			// since the app transition will not be triggered through the resume channel.
			mTargetStack.getDisplay().mDisplayContent.executeAppTransition();
		} else {
			// If the target stack was not previously focusable (previous top running activity
			// on that stack was not visible) then any prior calls to move the stack to the
			// will not update the focused stack.  If starting the new activity now allows the
			// task stack to be focusable, then ensure that we now update the focused stack
			// accordingly.
			if (mTargetStack.isFocusable()
					&& !mRootActivityContainer.isTopDisplayFocusedStack(mTargetStack)) {
				mTargetStack.moveToFront("startActivityUnchecked");
			}
			mRootActivityContainer.resumeFocusedStacksTopActivities(
					mTargetStack, mStartActivity, mOptions);
		}
	} else if (mStartActivity != null) {
		mSupervisor.mRecentTasks.add(mStartActivity.getTaskRecord());
	}
	mRootActivityContainer.updateUserStack(mStartActivity.mUserId, mTargetStack);

	mSupervisor.handleNonResizableTaskIfNeeded(mStartActivity.getTaskRecord(),
			preferredWindowingMode, mPreferredDisplayId, mTargetStack);

	return START_SUCCESS;
}


```

##### 3.6.1、ActivityStarter.setTaskFromReuseOrCreateNewTask()-创建新的Task[AMS端]

``` java
X:\frameworks\base\services\core\java\com\android\server\wm\ActivityStarter.java
    private int setTaskFromReuseOrCreateNewTask(TaskRecord taskToAffiliate) {
        if (mRestrictedBgActivity && (mReuseTask == null || !mReuseTask.containsAppUid(mCallingUid))
                && handleBackgroundActivityAbort(mStartActivity)) {
            return START_ABORTED;
        }

        mTargetStack = computeStackFocus(mStartActivity, true, mLaunchFlags, mOptions);

        // Do no move the target stack to front yet, as we might bail if
        // isLockTaskModeViolation fails below.

        if (mReuseTask == null) {
            final boolean toTop = !mLaunchTaskBehind && !mAvoidMoveToFront;
            final TaskRecord task = mTargetStack.createTaskRecord(
                    mSupervisor.getNextTaskIdForUserLocked(mStartActivity.mUserId),
                    mNewTaskInfo != null ? mNewTaskInfo : mStartActivity.info,
                    mNewTaskIntent != null ? mNewTaskIntent : mIntent, mVoiceSession,
                    mVoiceInteractor, toTop, mStartActivity, mSourceRecord, mOptions);
            addOrReparentStartingActivity(task, "setTaskFromReuseOrCreateNewTask - mReuseTask");
            updateBounds(mStartActivity.getTaskRecord(), mLaunchParams.mBounds);

            if (DEBUG_TASKS) Slog.v(TAG_TASKS, "Starting new activity " + mStartActivity
                    + " in new task " + mStartActivity.getTaskRecord());
------------------------------------------------------------------------------------------------------
对应Log：11-12 03:15:29.698   433   916 V ActivityStackSupervisor_Tasks: Starting new activity ActivityRecord{563ed0d u0 com.android.testred/.TestActivity t2} in new task TaskRecord{c645b40 #2 A=com.android.testred U=0 StackId=0 sz=1}

------------------------------------------------------------------------------------------------------
 
        } else {
            addOrReparentStartingActivity(mReuseTask, "setTaskFromReuseOrCreateNewTask");
        }

        if (taskToAffiliate != null) {
            mStartActivity.setTaskToAffiliateWith(taskToAffiliate);
        }

        if (mService.getLockTaskController().isLockTaskModeViolation(
                mStartActivity.getTaskRecord())) {
            Slog.e(TAG, "Attempted Lock Task Mode violation mStartActivity=" + mStartActivity);
            return START_RETURN_LOCK_TASK_MODE_VIOLATION;
        }

        if (mDoResume) {
            mTargetStack.moveToFront("reuseOrNewTask");
        }
        return START_SUCCESS;
    }

```

##### 3.6.1.1、TaskRecord创建createTaskRecord()

``` java
X:\frameworks\base\services\core\java\com\android\server\wm\ActivityStack.java
TaskRecord createTaskRecord(int taskId, ActivityInfo info, Intent intent,
		IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
		boolean toTop, ActivityRecord activity, ActivityRecord source,
		ActivityOptions options) {
	final TaskRecord task = TaskRecord.create(
			mService, taskId, info, intent, voiceSession, voiceInteractor);
	// add the task to stack first, mTaskPositioner might need the stack association
	addTask(task, toTop, "createTaskRecord");
	final int displayId = mDisplayId != INVALID_DISPLAY ? mDisplayId : DEFAULT_DISPLAY;
	final boolean isLockscreenShown = mService.mStackSupervisor.getKeyguardController()
			.isKeyguardOrAodShowing(displayId);
	if (!mStackSupervisor.getLaunchParamsController()
			.layoutTask(task, info.windowLayout, activity, source, options)
			&& !matchParentBounds() && task.isResizeable() && !isLockscreenShown) {
		task.updateOverrideConfiguration(getRequestedOverrideBounds());
	}
	task.createTask(toTop, (info.flags & FLAG_SHOW_FOR_ALL_USERS) != 0);
	return task;
}
```

##### 3.6.1.1.1、TaskRecord.create()

``` java
X:\frameworks\base\services\core\java\com\android\server\wm\TaskRecord.java
    static TaskRecord create(ActivityTaskManagerService service, int taskId, ActivityInfo info,
            Intent intent, IVoiceInteractionSession voiceSession,
            IVoiceInteractor voiceInteractor) {
        return getTaskRecordFactory().create(
                service, taskId, info, intent, voiceSession, voiceInteractor);
    }

    static class TaskRecordFactory {

        TaskRecord create(ActivityManagerService service, int taskId, ActivityInfo info,
                Intent intent, IVoiceInteractionSession voiceSession,
                IVoiceInteractor voiceInteractor) {
            return new TaskRecord(
                    service, taskId, info, intent, voiceSession, voiceInteractor);
        }
        ......
     }

    TaskRecord(ActivityManagerService service, int _taskId, ActivityInfo info, Intent _intent,
            IVoiceInteractionSession _voiceSession, IVoiceInteractor _voiceInteractor) {
        mService = service;
        userId = UserHandle.getUserId(info.applicationInfo.uid);
        taskId = _taskId;
        lastActiveTime = SystemClock.elapsedRealtime();
        mAffiliatedTaskId = _taskId;
        voiceSession = _voiceSession;
        voiceInteractor = _voiceInteractor;
        isAvailable = true;
        mActivities = new ArrayList<>();
        mCallingUid = info.applicationInfo.uid;
        mCallingPackage = info.packageName;
        setIntent(_intent, info);
        setMinDimensions(info);
        touchActiveTime();
        mService.mTaskChangeNotificationController.notifyTaskCreated(_taskId, realActivity);
    }
```

##### 3.6.1.1.2、将TaskRecord加入到ActivityStack中


``` java
X:\frameworks\base\services\core\java\com\android\server\wm\ActivityStack.java
   void addTask(final TaskRecord task, final boolean toTop, String reason) {
        addTask(task, toTop ? MAX_VALUE : 0, true /* schedulePictureInPictureModeChange */, reason);
        if (toTop) {
            // TODO: figure-out a way to remove this call.
            positionChildWindowContainerAtTop(task);
        }
    }

    // TODO: This shouldn't allow automatic reparenting. Remove the call to preAddTask and deal
    // with the fall-out...
    void addTask(final TaskRecord task, int position, boolean schedulePictureInPictureModeChange,
            String reason) {
        // TODO: Is this remove really needed? Need to look into the call path for the other addTask
        mTaskHistory.remove(task);
        if (isSingleTaskInstance() && !mTaskHistory.isEmpty()) {
            throw new IllegalStateException("Can only have one child on stack=" + this);
        }

        position = getAdjustedPositionForTask(task, position, null /* starting */);
        final boolean toTop = position >= mTaskHistory.size();
        final ActivityStack prevStack = preAddTask(task, reason, toTop);

        mTaskHistory.add(position, task);
        task.setStack(this);

        updateTaskMovement(task, toTop);

        postAddTask(task, prevStack, schedulePictureInPictureModeChange);
    }
```
##### 3.6.1.1.3、TaskRecord..createTask()[WMS端的Task]

``` java
X:\frameworks\base\services\core\java\com\android\server\wm\TaskRecord.java
    void createTask(boolean onTop, boolean showForAllUsers) {
        
        final Rect bounds = updateOverrideConfigurationFromLaunchBounds();
        final TaskStack stack = getStack().getTaskStack();

        EventLog.writeEvent(WM_TASK_CREATED, taskId, stack.mStackId);
        mTask = new Task(taskId, stack, userId, mService.mWindowManager, mResizeMode,
                mSupportsPictureInPicture, lastTaskDescription, this);
        final int position = onTop ? POSITION_TOP : POSITION_BOTTOM;

        if (!mDisplayedBounds.isEmpty()) {
            mTask.setOverrideDisplayedBounds(mDisplayedBounds);
        }
        // We only want to move the parents to the parents if we are creating this task at the
        // top of its stack.
        stack.addTask(mTask, position, showForAllUsers, onTop /* moveParents */);
    }
X:\frameworks\base\services\core\java\com\android\server\wm\TaskRecord.java
    Task(int taskId, TaskStack stack, int userId, WindowManagerService service, int resizeMode,
            boolean supportsPictureInPicture, TaskDescription taskDescription,
            TaskRecord taskRecord) {
        super(service);
        mTaskId = taskId;
        mStack = stack;
        mUserId = userId;
        mResizeMode = resizeMode;
        mSupportsPictureInPicture = supportsPictureInPicture;
        mTaskRecord = taskRecord;
        if (mTaskRecord != null) {
            mTaskRecord.registerConfigurationChangeListener(this);
        }
        setBounds(getRequestedOverrideBounds());
        mTaskDescription = taskDescription;

        // Tasks have no set orientation value (including SCREEN_ORIENTATION_UNSPECIFIED).
        setOrientation(SCREEN_ORIENTATION_UNSET);
    }


```

##### 3.6.1.1.4、将Task[WMS]加入到TaskStack 

``` java
X:\frameworks\base\services\core\java\com\android\server\wm\TaskStack.java
    void addTask(Task task, int position, boolean showForAllUsers, boolean moveParents) {
        final TaskStack currentStack = task.mStack;

        // Add child task.
        task.mStack = this;
        addChild(task, null);

        // Move child to a proper position, as some restriction for position might apply.
        positionChildAt(position, task, moveParents /* includingParents */, showForAllUsers);
    }

```


##### 3.6.2、ActivityStack.startActivityLocked()


``` java
X:\frameworks\base\services\core\java\com\android\server\wm\ActivityStack.java

    void startActivityLocked(ActivityRecord r, ActivityRecord focusedTopActivity,
            boolean newTask, boolean keepCurTransition, ActivityOptions options) {
        TaskRecord rTask = r.getTaskRecord();
        final int taskId = rTask.taskId;
        final boolean allowMoveToFront = options == null || !options.getAvoidMoveToFront();
        // mLaunchTaskBehind tasks get placed at the back of the task stack.
        if (!r.mLaunchTaskBehind && allowMoveToFront
                && (taskForIdLocked(taskId) == null || newTask)) {
            insertTaskAtTop(rTask, r);
        }
        TaskRecord task = null;
        if (!newTask) {
            // If starting in an existing task, find where that is...
            boolean startIt = true;
            for (int taskNdx = mTaskHistory.size() - 1; taskNdx >= 0; --taskNdx) {
                task = mTaskHistory.get(taskNdx);
                if (task.getTopActivity() == null) {
                    // All activities in task are finishing.
                    continue;
                }
                if (task == rTask) {
                    // Here it is!  Now, if this is not yet visible to the
                    // user, then just add it without starting; it will
                    // get started when the user navigates back to it.
                    if (!startIt) {
                    ------------------------------
                    11-12 03:15:29.701   433   916 I ActivityStack: Adding activity ActivityRecord{563ed0d u0 com.android.testred/.TestActivity t2} to stack to task TaskRecord{c645b40 #2 A=com.android.testred U=0 StackId=0 sz=1}
                    ------------------------------
                        if (DEBUG_ADD_REMOVE) Slog.i(TAG, "Adding activity " + r + " to task "
                                + task, new RuntimeException("here").fillInStackTrace());
                        r.createAppWindowToken();
                        ActivityOptions.abort(options);
                        return;
                    }
                    break;
                } else if (task.numFullscreen > 0) {
                    startIt = false;
                }
            }
        }

        // Place a new activity at top of stack, so it is next to interact with the user.

        task = activityTask;

        // Slot the activity into the history stack and proceed
        if (DEBUG_ADD_REMOVE) Slog.i(TAG, "Adding activity " + r + " to stack to task " + task,
                new RuntimeException("here").fillInStackTrace());
        // TODO: Need to investigate if it is okay for the controller to already be created by the
        // time we get to this point. I think it is, but need to double check.
        // Use test in b/34179495 to trace the call path.
        if (r.mAppWindowToken == null) {
            r.createAppWindowToken();
        }

        task.setFrontOfTask();

        // The transition animation and starting window are not needed if {@code allowMoveToFront}
        // is false, because the activity won't be visible.
        if ((!isHomeOrRecentsStack() || numActivities() > 0) && allowMoveToFront) {
            final DisplayContent dc = getDisplay().mDisplayContent;
            ----------------------------------------------------
11-12 03:15:29.703   433   916 V ActivityStack_Transition: Prepare open transition: starting ActivityRecord{563ed0d u0 com.android.testred/.TestActivity t2}
            ----------------------------------------------------
            if (DEBUG_TRANSITION) Slog.v(TAG_TRANSITION,
                    "Prepare open transition: starting " + r);
            if ((r.intent.getFlags() & Intent.FLAG_ACTIVITY_NO_ANIMATION) != 0) {
                dc.prepareAppTransition(TRANSIT_NONE, keepCurTransition);
                mStackSupervisor.mNoAnimActivities.add(r);
            } else {
                int transit = TRANSIT_ACTIVITY_OPEN;
                if (newTask) {
                    if (r.mLaunchTaskBehind) {
                        transit = TRANSIT_TASK_OPEN_BEHIND;
                    } else if (getDisplay().isSingleTaskInstance()) {
                        transit = TRANSIT_SHOW_SINGLE_TASK_DISPLAY;
                    } else {
                        // If a new task is being launched, then mark the existing top activity as
                        // supporting picture-in-picture while pausing only if the starting activity
                        // would not be considered an overlay on top of the current activity
                        // (eg. not fullscreen, or the assistant)
                        if (canEnterPipOnTaskSwitch(focusedTopActivity,
                                null /* toFrontTask */, r, options)) {
                            focusedTopActivity.supportsEnterPipOnTaskSwitch = true;
                        }
                        transit = TRANSIT_TASK_OPEN;
                    }
                }
                dc.prepareAppTransition(transit, keepCurTransition);
                mStackSupervisor.mNoAnimActivities.remove(r);
            }
            boolean doShow = true;
            if (newTask) {
                // Even though this activity is starting fresh, we still need
                // to reset it to make sure we apply affinities to move any
                // existing activities from other tasks in to it.
                // If the caller has requested that the target task be
                // reset, then do so.
                if ((r.intent.getFlags() & Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) != 0) {
                    resetTaskIfNeededLocked(r, r);
                    doShow = topRunningNonDelayedActivityLocked(null) == r;
                }
            } else if (options != null && options.getAnimationType()
                    == ActivityOptions.ANIM_SCENE_TRANSITION) {
                doShow = false;
            }
            if (r.mLaunchTaskBehind) {
                // Don't do a starting window for mLaunchTaskBehind. More importantly make sure we
                // tell WindowManager that r is visible even though it is at the back of the stack.
                r.setVisibility(true);
                ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
            } else if (SHOW_APP_STARTING_PREVIEW && doShow) {
                ......
                r.showStartingWindow(prev, newTask, isTaskSwitch(r, focusedTopActivity));
            }
        } else {
            // If this is the first activity, don't do any fancy animations,
            // because there is nothing for it to animate on top of.
            ActivityOptions.abort(options);
        }
    }

```


##### 3.6.3、ActivityStack.ensureActivitiesVisibleLocked()
由于现在App进程还未创建，会走此分支
``` java
    final void ensureActivitiesVisibleLocked(ActivityRecord starting, int configChanges,

                        if (!r.attachedToProcess()) {
                            if (makeVisibleAndRestartIfNeeded(starting, configChanges, isTop,
                                    resumeNextActivity, r)) {
                                if (activityNdx >= activities.size()) {
                                    // Record may be removed if its process needs to restart.
                                    activityNdx = activities.size() - 1;
                                } else {
                                    resumeNextActivity = false;
                                }
                            }
                        } 

        }
    }

```


##### 3.6.3.1、ActivityStack.makeVisibleAndRestartIfNeeded()

此时ActivityRecord 状态为state=INITIALIZING
> 11-12 03:15:29.708   433   916 V ActivityStack_Visibility: Make visible? ActivityRecord{563ed0d u0 com.android.testred/.TestActivity t2} finishing=false state=INITIALIZING


``` java
    private boolean makeVisibleAndRestartIfNeeded(ActivityRecord starting, int configChanges,
            boolean isTop, boolean andResume, ActivityRecord r) {
        // We need to make sure the app is running if it's the top, or it is just made visible from
        // invisible. If the app is already visible, it must have died while it was visible. In this
        // case, we'll show the dead window but will not restart the app. Otherwise we could end up
        // thrashing.
        if (isTop || !r.visible) {
            // This activity needs to be visible, but isn't even running...
            // get it started and resume if no other stack in this stack is resumed.
            if (DEBUG_VISIBILITY) Slog.v(TAG_VISIBILITY, "Start and freeze screen for " + r);
            if (r != starting) {
                r.startFreezingScreenLocked(r.app, configChanges);
            }
            if (!r.visible || r.mLaunchTaskBehind) {
                if (DEBUG_VISIBILITY) Slog.v(TAG_VISIBILITY, "Starting and making visible: " + r);
                r.setVisible(true);
            }
            if (r != starting) {
                // We should not resume activities that being launched behind because these
                // activities are actually behind other fullscreen activities, but still required
                // to be visible (such as performing Recents animation).
                mStackSupervisor.startSpecificActivityLocked(r, andResume && !r.mLaunchTaskBehind,
                        true /* checkConfig */);
                return true;
            }
        }
        return false;
    }

```

##### 3.6.4、StackSupervisor.startSpecificActivityLocked()

``` java
    void startSpecificActivityLocked(ActivityRecord r, boolean andResume, boolean checkConfig) {
        // Is this activity's application already running?
        final WindowProcessController wpc =
                mService.getProcessController(r.processName, r.info.applicationInfo.uid);

        boolean knownToBeDead = false;
        if (wpc != null && wpc.hasThread()) {
            try {
                realStartActivityLocked(r, wpc, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting activity "
                        + r.intent.getComponent().flattenToShortString(), e);
            }
            ......
        }
        ......

        try {
            ......
            final Message msg = PooledLambda.obtainMessage(
                    ActivityManagerInternal::startProcess, mService.mAmInternal, r.processName,
                    r.info.applicationInfo, knownToBeDead, "activity", r.intent.getComponent());
            mService.mH.sendMessage(msg);
        } finally {
            Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
        }
    }

11-12 03:18:19.103   427   449 I ActivityManagerService: java.lang.RuntimeException: here
11-12 03:18:19.103   427   449 I ActivityManagerService: 	at com.android.server.am.ActivityManagerService$LocalService.startProcess(ActivityManagerService.java:18542)
11-12 03:18:19.103   427   449 I ActivityManagerService: 	at com.android.server.wm.-$$Lambda$3W4Y_XVQUddVKzLjibuHW7h0R1g.accept(Unknown Source:22)
11-12 03:18:19.103   427   449 I ActivityManagerService: 	at com.android.internal.util.function.pooled.PooledLambdaImpl.doInvoke(PooledLambdaImpl.java:335)
11-12 03:18:19.103   427   449 I ActivityManagerService: 	at com.android.internal.util.function.pooled.PooledLambdaImpl.invoke(PooledLambdaImpl.java:195)
11-12 03:18:19.103   427   449 I ActivityManagerService: 	at com.android.internal.util.function.pooled.OmniFunction.run(OmniFunction.java:86)
11-12 03:18:19.103   427   449 I ActivityManagerService: 	at android.os.Handler.handleCallback(Handler.java:883)
11-12 03:18:19.103   427   449 I ActivityManagerService: 	at android.os.Handler.dispatchMessage(Handler.java:100)
11-12 03:18:19.103   427   449 I ActivityManagerService: 	at android.os.Looper.loop(Looper.java:214)
11-12 03:18:19.103   427   449 I ActivityManagerService: 	at android.os.HandlerThread.run(HandlerThread.java:67)
11-12 03:18:19.103   427   449 I ActivityManagerService: 	at com.android.server.ServiceThread.run(ServiceThread.java:44)


```

##### 3.6.5、StackSupervisor.startSpecificActivityLocked()
ActivityManagerInternal::startProcess -> ActivityManagerService::startProcess
``` java
frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java

        @Override
        public void startProcess(String processName, ApplicationInfo info,
                boolean knownToBeDead, String hostingType, ComponentName hostingName) {
            try {
                if (Trace.isTagEnabled(Trace.TRACE_TAG_ACTIVITY_MANAGER)) {
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "startProcess:"
                            + processName);
                }
 -------------------------------------------------
 11-12 03:15:29.918   433   455 V xzj     : startProcess: name=com.android.testred app=ProcessRecord{f5e79a0 1069:com.android.testred/u0a56} knownToBeDead=false hostingRecord=activity intentFlags=0 thread=android.app.IApplicationThread$Stub$Proxy@cbb8cb pid=1069
 -------------------------------------------------
                synchronized (ActivityManagerService.this) {
                    startProcessLocked(processName, info, knownToBeDead, 0 /* intentFlags */,
                            new HostingRecord(hostingType, hostingName),
                            false /* allowWhileBooting */, false /* isolated */,
                            true /* keepIfLarge */);
                }
            } finally {
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            }
        }

```

进程的创建跟之前分析的一致，就不再分析了。创建完成后回调用ActivityThread的main方法。


#### 3.7、Activity所在应用主线程初始化 ActivityThread.main()
在ActivityThread.main方法中对ActivityThread进行了初始化，创建了主线程的Looper对象并调用Looper.loop()方法启动Looper，把自定义Handler类H的对象作为主线程的handler。接下来跳转到ActivityThread.attach方法，看都做了什么。

``` java
   public static void main(String[] args) {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");

        // Install selective syscall interception
        AndroidOs.install();

        Environment.initForCurrentUser();

        // Make sure TrustedCertificateStore looks in the right place for CA certificates
        final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
        TrustedCertificateStore.setDefaultUserDirectory(configDir);

        Process.setArgV0("<pre-initialized>");

        Looper.prepareMainLooper();
        ......
        ActivityThread thread = new ActivityThread();
        thread.attach(false, startSeq);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }
        // End of event ActivityThreadMain.
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```


Activity所在的进程创建完了，主线程也初始化了，接下来就该真正的启动Activity了。在ActivityThread.attach方法中，首先会通过ActivityManagerService为这个应用绑定一个Application，然后添加一个垃圾回收观察者，每当系统触发垃圾回收的时候就会在run方法里面去计算应用使用了多少内存，如果超过总量的四分之三就会尝试释放内存。最后，为根View添加config回调接收config变化相关的信息。

``` java
   @UnsupportedAppUsage
    private void attach(boolean system, long startSeq) {
        sCurrentActivityThread = this;
        mSystemThread = system;
        if (!system) {
            android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
                                                    UserHandle.myUserId());
            RuntimeInit.setApplicationObject(mAppThread.asBinder());
            final IActivityManager mgr = ActivityManager.getService();
            try {
                mgr.attachApplication(mAppThread, startSeq);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
            ......
        } else {
            // Don't set application object here -- if the system crashes,
            // we can't display an alert, we just want to die die die.
            android.ddm.DdmHandleAppName.setAppName("system_process",
                    UserHandle.myUserId());
            try {
                mInstrumentation = new Instrumentation();
                mInstrumentation.basicInit(this);
                ContextImpl context = ContextImpl.createAppContext(
                        this, getSystemContext().mPackageInfo);
                mInitialApplication = context.mPackageInfo.makeApplication(true, null);
                mInitialApplication.onCreate();
            } catch (Exception e) {
                throw new RuntimeException(
                        "Unable to instantiate Application():" + e.toString(), e);
            }
        }

        ViewRootImpl.ConfigChangedCallback configChangedCallback
                = (Configuration globalConfig) -> {
            synchronized (mResourcesManager) {
                // We need to apply this change to the resources immediately, because upon returning
                // the view hierarchy will be informed about it.
                if (mResourcesManager.applyConfigurationToResourcesLocked(globalConfig,
                        null /* compat */)) {
                    updateLocaleListFromAppContext(mInitialApplication.getApplicationContext(),
                            mResourcesManager.getConfiguration().getLocales());

                    // This actually changed the resources! Tell everyone about it.
                    if (mPendingConfiguration == null
                            || mPendingConfiguration.isOtherSeqNewer(globalConfig)) {
                        mPendingConfiguration = globalConfig;
                        sendMessage(H.CONFIGURATION_CHANGED, globalConfig);
                    }
                }
            }
        };
        ViewRootImpl.addConfigCallback(configChangedCallback);
    }
```

#### 3.7.1、ActivityManagerService.attachApplication()

``` java
X:\frameworks\base\services\core\java\com\android\server\am\ActivityManagerService.java
    @GuardedBy("this")
    private boolean attachApplicationLocked(@NonNull IApplicationThread thread,
            int pid, int callingUid, long startSeq) {

        // Find the application record that is being attached...  either via
        // the pid if we are running in multiple processes, or just pull the
        // next app record if we are emulating process with anonymous threads.
        ProcessRecord app;
        long startTime = SystemClock.uptimeMillis();
        long bindApplicationTimeMillis;
        ......
        ------------------------------------------------------
11-12 03:15:29.897   433   451 V ActivityManagerService: Binding process pid 1069 to record ProcessRecord{f5e79a0 1069:com.android.testred/u0a56}
11-12 03:15:29.897   433   451 V ActivityManagerService: New death recipient com.android.server.am.ActivityManagerService$AppDeathRecipient@c19c0af for thread android.os.BinderProxy@6068cbc
11-12 03:15:29.897   433   451 I am_proc_bound: [0,1069,com.android.testred]
11-12 03:15:29.897   433   451 V ActivityManagerService: New app record ProcessRecord{f5e79a0 1069:com.android.testred/u0a56} thread=android.os.BinderProxy@6068cbc pid=1069
        ------------------------------------------------------
        if (DEBUG_ALL) Slog.v(
                TAG, "Binding process pid " + pid + " to record " + app);
        if (DEBUG_ALL) Slog.v(
            TAG, "New app record " + app
            + " thread=" + thread.asBinder() + " pid=" + pid);
        final BackupRecord backupTarget = mBackupTargets.get(app.userId);
     
            -------------------------------------------------
11-12 03:15:29.897   433   451 V ActivityManagerService_Configuration: Binding proc com.android.testred with config {1.0 ?mcc?mnc [en_US] ldltr sw621dp w1097dp h549dp 280dpi lrg long land finger -keyb/v/h dpad/v winConfig={ mBounds=Rect(0, 0 - 1920, 1088) mAppBounds=Rect(0, 0 - 1920, 1004) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=undefined mAlwaysOnTop=undefined mRotation=ROTATION_90} s.6}
            -------------------------------------------------
            if (DEBUG_CONFIGURATION) Slog.v(TAG_CONFIGURATION, "Binding proc "
                    + processName + " with config "
                    + app.getWindowProcessController().getConfiguration());
            ApplicationInfo appInfo = instr != null ? instr.mTargetInfo : app.info;
            app.compat = compatibilityInfoForPackage(appInfo);

            ProfilerInfo profilerInfo = null;
            String preBindAgent = null;
            ......
            final ActiveInstrumentation instr2 = app.getActiveInstrumentation();
            if (app.isolatedEntryPoint != null) {
                // This is an isolated process which should just call an entry point instead of
                // being bound to an application.
                thread.runIsolatedEntryPoint(app.isolatedEntryPoint, app.isolatedEntryPointArgs);
            } else if (instr2 != null) {
                thread.bindApplication(processName, appInfo, providers,
                        instr2.mClass,
                        profilerInfo, instr2.mArguments,
                        instr2.mWatcher,
                        instr2.mUiAutomationConnection, testMode,
                        mBinderTransactionTrackingEnabled, enableTrackAllocation,
                        isRestrictedBackupMode || !normalMode, app.isPersistent(),
                        new Configuration(app.getWindowProcessController().getConfiguration()),
                        app.compat, getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked(),
                        buildSerial, autofillOptions, contentCaptureOptions);
            } else {
                thread.bindApplication(processName, appInfo, providers, null, profilerInfo,
                        null, null, null, testMode,
                        mBinderTransactionTrackingEnabled, enableTrackAllocation,
                        isRestrictedBackupMode || !normalMode, app.isPersistent(),
                        new Configuration(app.getWindowProcessController().getConfiguration()),
                        app.compat, getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked(),
                        buildSerial, autofillOptions, contentCaptureOptions);
            }

        // See if the top visible activity is waiting to run in this process...
        if (normalMode) {
            try {
                didSomething = mAtmInternal.attachApplication(app.getWindowProcessController());
            } catch (Exception e) {
                Slog.wtf(TAG, "Exception thrown launching activities in " + app, e);
                badApp = true;
            }
        }
        ......
        return true;
    }

```
这里又调用App进程自己的bindApplication()方法。接着

``` java
callers=com.android.server.wm.ActivityStackSupervisor.realStartActivityLocked:917 com.android.server.wm.RootActivityContainer.attachApplication:784 com.android.server.wm.ActivityTaskManagerService$LocalService.attachApplication:6949 com.android.server.am.ActivityManagerService.attachApplicationLocked:5213 com.android.server.am.ActivityManagerService.attachApplication:5293 
```

#### 3.7.2、ActivityTaskManagerService$LocalService.attachApplication()

``` java
X:\frameworks\base\services\core\java\com\android\server\wm\ActivityTaskManagerService.java
        @HotPath(caller = HotPath.PROCESS_CHANGE)
        @Override
        public boolean attachApplication(WindowProcessController wpc) throws RemoteException {
            synchronized (mGlobalLockWithoutBoost) {
                return mRootActivityContainer.attachApplication(wpc);
            }
        }
```
#### 3.7.3、RootActivityContainer.attachApplication()

``` java
X:\frameworks\base\services\core\java\com\android\server\wm\RootActivityContainer.java
    boolean attachApplication(WindowProcessController app) throws RemoteException {
        final String processName = app.mName;
        boolean didSomething = false;
        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            final ActivityDisplay display = mActivityDisplays.get(displayNdx);
            final ActivityStack stack = display.getFocusedStack();
            if (stack != null) {
                stack.getAllRunningVisibleActivitiesLocked(mTmpActivityList);
                final ActivityRecord top = stack.topRunningActivityLocked();
                final int size = mTmpActivityList.size();
                for (int i = 0; i < size; i++) {
                    final ActivityRecord activity = mTmpActivityList.get(i);
                    if (activity.app == null && app.mUid == activity.info.applicationInfo.uid
                            && processName.equals(activity.processName)) {
                        try {
                            if (mStackSupervisor.realStartActivityLocked(activity, app,
                                    top == activity /* andResume */, true /* checkConfig */)) {
                                didSomething = true;
                            }
                        } catch (RemoteException e) {
                            Slog.w(TAG, "Exception in new application when starting activity "
                                    + top.intent.getComponent().flattenToShortString(), e);
                            throw e;
                        }
                    }
                }
            }
        }
        if (!didSomething) {
            ensureActivitiesVisible(null, 0, false /* preserve_windows */);
        }
        return didSomething;
    }
```

#### 3.7.4、ActivityStackSupervisor.realStartActivityLocked()
又进入熟悉的realStartActivityLocked方法了，之前因为app进程没有创建，startSpecificActivityLocked走的另外一个分支，还记着吗

``` java
boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
            boolean andResume, boolean checkConfig) throws RemoteException {

        ......
        final TaskRecord task = r.getTaskRecord();
        final ActivityStack stack = task.getStack();

        beginDeferResume();

        try {
            r.startFreezingScreenLocked(proc, 0);

            // schedule launch ticks to collect information about slow apps.
            r.startLaunchTickingLocked();

            r.setProcess(proc);
            ......
            r.launchCount++;
            r.lastLaunchTime = SystemClock.uptimeMillis();

            if (DEBUG_ALL) Slog.v(TAG, "Launching: " + r);

            proc.addActivityIfNeeded(r);

            final LockTaskController lockTaskController = mService.getLockTaskController();
            if (task.mLockTaskAuth == LOCK_TASK_AUTH_LAUNCHABLE
                    || task.mLockTaskAuth == LOCK_TASK_AUTH_LAUNCHABLE_PRIV
                    || (task.mLockTaskAuth == LOCK_TASK_AUTH_WHITELISTED
                            && lockTaskController.getLockTaskModeState()
                                    == LOCK_TASK_MODE_LOCKED)) {
                lockTaskController.startLockTaskMode(task, false, 0 /* blank UID */);
            }

            try {
                if (!proc.hasThread()) {
                    throw new RemoteException();
                }
                List<ResultInfo> results = null;
                List<ReferrerIntent> newIntents = null;
                if (andResume) {
                    // We don't need to deliver new intents and/or set results if activity is going
                    // to pause immediately after launch.
                    results = r.results;
                    newIntents = r.newIntents;
                }
    -------------------------------------------------------------
	Line 80647: 11-12 03:15:29.909   433   451 V ActivityStackSupervisor: Launching: ActivityRecord{563ed0d u0 com.android.testred/.TestActivity t2}
	Line 80652: 11-12 03:15:29.909   433   451 V ActivityStackSupervisor_Switch: Launching: ActivityRecord{563ed0d u0 com.android.testred/.TestActivity t2} icicle=null with results=null newIntents=null andResume=true
	Line 80656: 11-12 03:15:29.909   433   451 I am_restart_activity: [0,90434829,2,com.android.testred/.TestActivity]
  --------------------------------------------------------------------------
                if (DEBUG_SWITCH) Slog.v(TAG_SWITCH,
                        "Launching: " + r + " icicle=" + r.icicle + " with results=" + results
                                + " newIntents=" + newIntents + " andResume=" + andResume);
                EventLog.writeEvent(EventLogTags.AM_RESTART_ACTIVITY, r.mUserId,
                        System.identityHashCode(r), task.taskId, r.shortComponentName);
                if (r.isActivityTypeHome()) {
                    // Home process is the root process of the task.
                    updateHomeProcess(task.mActivities.get(0).app);
                }
                mService.getPackageManagerInternalLocked().notifyPackageUse(
                        r.intent.getComponent().getPackageName(), NOTIFY_PACKAGE_USE_ACTIVITY);
                r.sleeping = false;
                r.forceNewConfig = false;
                mService.getAppWarningsLocked().onStartActivity(r);
                r.compat = mService.compatibilityInfoForPackageLocked(r.info.applicationInfo);

                // Because we could be starting an Activity in the system process this may not go
                // across a Binder interface which would create a new Configuration. Consequently
                // we have to always create a new Configuration here.

                final MergedConfiguration mergedConfiguration = new MergedConfiguration(
                        proc.getConfiguration(), r.getMergedOverrideConfiguration());
                r.setLastReportedConfiguration(mergedConfiguration);

                logIfTransactionTooLarge(r.intent, r.icicle);


                // Create activity launch transaction.
                final ClientTransaction clientTransaction = ClientTransaction.obtain(
                        proc.getThread(), r.appToken);

                final DisplayContent dc = r.getDisplay().mDisplayContent;
                clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                        System.identityHashCode(r), r.info,
                        // TODO: Have this take the merged configuration instead of separate global
                        // and override configs.
                        mergedConfiguration.getGlobalConfiguration(),
                        mergedConfiguration.getOverrideConfiguration(), r.compat,
                        r.launchedFromPackage, task.voiceInteractor, proc.getReportedProcState(),
                        r.icicle, r.persistentState, results, newIntents,
                        dc.isNextTransitionForward(), proc.createProfilerInfoIfNeeded(),
                                r.assistToken));

                // Set desired final state.
                final ActivityLifecycleItem lifecycleItem;
                if (andResume) {
                    lifecycleItem = ResumeActivityItem.obtain(dc.isNextTransitionForward());
                } else {
                    lifecycleItem = PauseActivityItem.obtain();
                }
                clientTransaction.setLifecycleStateRequest(lifecycleItem);

                // Schedule transaction.
                mService.getLifecycleManager().scheduleTransaction(clientTransaction);

                if ((proc.mInfo.privateFlags & ApplicationInfo.PRIVATE_FLAG_CANT_SAVE_STATE) != 0
                        && mService.mHasHeavyWeightFeature) {
                    // This may be a heavy-weight process! Note that the package manager will ensure
                    // that only activity can run in the main process of the .apk, which is the only
                    // thing that will be considered heavy-weight.
                    if (proc.mName.equals(proc.mInfo.packageName)) {
                        if (mService.mHeavyWeightProcess != null
                                && mService.mHeavyWeightProcess != proc) {
                            Slog.w(TAG, "Starting new heavy weight process " + proc
                                    + " when already running "
                                    + mService.mHeavyWeightProcess);
                        }
                        mService.setHeavyWeightProcess(r);
                    }
                }

            } catch (RemoteException e) {
                if (r.launchFailed) {
                    // This is the second time we failed -- finish activity and give up.
                    Slog.e(TAG, "Second failure launching "
                            + r.intent.getComponent().flattenToShortString() + ", giving up", e);
                    proc.appDied();
                    stack.requestFinishActivityLocked(r.appToken, Activity.RESULT_CANCELED, null,
                            "2nd-crash", false);
                    return false;
                }

                // This is the first time we failed -- restart process and
                // retry.
                r.launchFailed = true;
                proc.removeActivity(r);
                throw e;
            }
        } finally {
            endDeferResume();
        }

        r.launchFailed = false;
        if (stack.updateLRUListLocked(r)) {
            Slog.w(TAG, "Activity " + r + " being launched, but already in LRU list");
        }

        // TODO(lifecycler): Resume or pause requests are done as part of launch transaction,
        // so updating the state should be done accordingly.
        if (andResume && readyToResume()) {
            // As part of the process of launching, ActivityThread also performs
            // a resume.
            stack.minimalResumeActivityLocked(r);
        } else {
            // This activity is not starting in the resumed state... which should look like we asked
            // it to pause+stop (but remain visible), and it has done so and reported back the
            // current icicle and other state.
            if (DEBUG_STATES) Slog.v(TAG_STATES,
                    "Moving to PAUSED: " + r + " (starting in paused state)");
            r.setState(PAUSED, "realStartActivityLocked");
        }
        // Perform OOM scoring after the activity state is set, so the process can be updated with
        // the latest state.
        proc.onStartActivity(mService.mTopProcessState, r.info);

        // Launch the new version setup screen if needed.  We do this -after-
        // launching the initial activity (that is, home), so that it can have
        // a chance to initialize itself while in the background, making the
        // switch back to it faster and look better.
        if (mRootActivityContainer.isTopDisplayFocusedStack(stack)) {
            mService.getActivityStartController().startSetupActivity();
        }

        // Update any services we are bound to that might care about whether
        // their client may have activities.
        if (r.app != null) {
            r.app.updateServiceConnectionActivities();
        }

        return true;
    }
```
#### 3.7.5、LaunchActivityItem.obtain()
ClientTransaction对象添加LaunchActivityItem的callback，然后设置当前的生命周期状态，最后调用ClientLifecycleManager.scheduleTransaction方法执行。
``` java
X:\frameworks\base\services\core\java\com\android\server\wm\ActivityStackSupervisor.java
                // Create activity launch transaction.
                final ClientTransaction clientTransaction = ClientTransaction.obtain(
                        proc.getThread(), r.appToken);

                final DisplayContent dc = r.getDisplay().mDisplayContent;
                clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                        System.identityHashCode(r), r.info,
                        // TODO: Have this take the merged configuration instead of separate global
                        // and override configs.
                        mergedConfiguration.getGlobalConfiguration(),
                        mergedConfiguration.getOverrideConfiguration(), r.compat,
                        r.launchedFromPackage, task.voiceInteractor, proc.getReportedProcState(),
                        r.icicle, r.persistentState, results, newIntents,
                        dc.isNextTransitionForward(), proc.createProfilerInfoIfNeeded(),
                                r.assistToken));
                // Set desired final state.
                final ActivityLifecycleItem lifecycleItem;
                if (andResume) {
                    lifecycleItem = ResumeActivityItem.obtain(dc.isNextTransitionForward());
                } else {
                    lifecycleItem = PauseActivityItem.obtain();
                }
                clientTransaction.setLifecycleStateRequest(lifecycleItem);

                // Schedule transaction.
                mService.getLifecycleManager().scheduleTransaction(clientTransaction);
```


#### 3.7.6、ActivityStack.minimalResumeActivityLocked(r)

``` java
X:\frameworks\base\services\core\java\com\android\server\wm\ActivityStack.java
    void minimalResumeActivityLocked(ActivityRecord r) {
        if (DEBUG_STATES) Slog.v(TAG_STATES, "Moving to RESUMED: " + r + " (starting new instance)"
                + " callers=" + Debug.getCallers(5));
        r.setState(RESUMED, "minimalResumeActivityLocked");
        r.completeResumeLocked();
        if (DEBUG_SAVED_STATE) Slog.i(TAG_SAVED_STATE,
                "Launch completed; removing icicle of " + r.icicle);
    }
--------------------------------------------------------------
11-12 03:15:29.910   433   451 V ActivityStack_Stack: set resumed activity to:ActivityRecord{563ed0d u0 com.android.testred/.TestActivity t2} reason:minimalResumeActivityLocked

11-12 03:15:29.911   433   451 D ActivityStack_Stack: setResumedActivity stack:ActivityStack{6c17ba0 stackId=0 type=home mode=fullscreen visible=true translucent=true, 2 tasks} + from: null to:ActivityRecord{563ed0d u0 com.android.testred/.TestActivity t2} reason:minimalResumeActivityLocked - onActivityStateChanged

-------------------------------------------------------------
X:\frameworks\base\services\core\java\com\android\server\wm\ActivityRecord.java
    void setState(ActivityState state, String reason) {
        if (DEBUG_STATES) Slog.v(TAG_STATES, "State movement: " + this + " from:" + getState()
                        + " to:" + state + " reason:" + reason);

        if (state == mState) {
            // No need to do anything if state doesn't change.
            if (DEBUG_STATES) Slog.v(TAG_STATES, "State unchanged from:" + state);
            return;
        }

        mState = state;

        final TaskRecord parent = getTaskRecord();

        if (parent != null) {
            parent.onActivityStateChanged(this, state, reason);
        }

        // The WindowManager interprets the app stopping signal as
        // an indication that the Surface will eventually be destroyed.
        // This however isn't necessarily true if we are going to sleep.
        if (state == STOPPING && !isSleeping()) {
            if (mAppWindowToken == null) {
                Slog.w(TAG_WM, "Attempted to notify stopping on non-existing app token: "
                        + appToken);
                return;
            }
            mAppWindowToken.detachChildren();
        }

        if (state == RESUMED) {
            mAtmService.updateBatteryStats(this, true);
            mAtmService.updateActivityUsageStats(this, Event.ACTIVITY_RESUMED);
        } else if (state == PAUSED) {
            mAtmService.updateBatteryStats(this, false);
            mAtmService.updateActivityUsageStats(this, Event.ACTIVITY_PAUSED);
        } else if (state == STOPPED) {
            mAtmService.updateActivityUsageStats(this, Event.ACTIVITY_STOPPED);
        } else if (state == DESTROYED) {
            mAtmService.updateActivityUsageStats(this, Event.ACTIVITY_DESTROYED);
        }
    }

```

####  四、执行启动Acitivity
##### 4.1、LaunchActivityItem.execute()
``` java
X:\frameworks\base\services\core\java\com\android\server\wm\ActivityStackSupervisor.java
                clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                        System.identityHashCode(r), r.info,
                        // TODO: Have this take the merged configuration instead of separate global
                        // and override configs.
                        mergedConfiguration.getGlobalConfiguration(),
                        mergedConfiguration.getOverrideConfiguration(), r.compat,
                        r.launchedFromPackage, task.voiceInteractor, proc.getReportedProcState(),
                        r.icicle, r.persistentState, results, newIntents,
                        dc.isNextTransitionForward(), proc.createProfilerInfoIfNeeded(),
                                r.assistToken));
```

这里的clientTransaction.addCallback(LaunchActivityItem.obtain(arg...))就是我们目前用到的lifecycleItem,也就是说 这个lifecycleItem就是LaunchActivityItem
继续看lifecycleItem.execute(mTransactionHandler, token, mPendingActions);源码:

``` java
X:\frameworks\base\core\java\android\app\servertransaction\LaunchActivityItem.java
    @Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
        ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
                mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
                mPendingResults, mPendingNewIntents, mIsForward,
                mProfilerInfo, client, mAssistToken);
        client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
        Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
    }
```

##### 4.2、ActivityThread.handleLaunchActivity()
client.handleLaunchActivity就是:控制activity的生命周期  真正的实现类是ClientTransactionHandler的子类 Activityhread;

继续看client.handleLaunchActivity源码:

``` java

   /**
     * Extended implementation of activity launch. Used when server requests a launch or relaunch.
     */
    @Override
    public Activity handleLaunchActivity(ActivityClientRecord r,
            PendingTransactionActions pendingActions, Intent customIntent) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();
        mSomeActivitiesChanged = true;

        if (r.profilerInfo != null) {
            mProfiler.setProfiler(r.profilerInfo);
            mProfiler.startProfiling();
        }

        // Make sure we are running with the most recent config.
        handleConfigurationChanged(null, null);

        if (localLOGV) Slog.v(
            TAG, "Handling launch of " + r);
        if (localLOGV) Slog.i(TAG, "handleLaunchActivity",new RuntimeException("here").fillInStackTrace());;

        // Initialize before creating the activity
        if (!ThreadedRenderer.sRendererDisabled
                && (r.activityInfo.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
            HardwareRenderer.preload();
        }
        WindowManagerGlobal.initialize();

        // Hint the GraphicsEnvironment that an activity is launching on the process.
        GraphicsEnvironment.hintActivityLaunch();

        final Activity a = performLaunchActivity(r, customIntent);

        if (a != null) {
            r.createdConfig = new Configuration(mConfiguration);
            reportSizeConfigurations(r);
            if (!r.activity.mFinished && pendingActions != null) {
                pendingActions.setOldState(r.state);
                pendingActions.setRestoreInstanceState(true);
                pendingActions.setCallOnPostCreate(true);
            }
        } else {
            // If there was an error, for any reason, tell the activity manager to stop us.
            try {
                ActivityTaskManager.getService()
                        .finishActivity(r.token, Activity.RESULT_CANCELED, null,
                                Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
        }

        return a;
    }
----------------------------------------------------------------------------

11-12 03:15:29.978  1069  1069 I ActivityThread: handleLaunchActivity
11-12 03:15:29.978  1069  1069 I ActivityThread: java.lang.RuntimeException: here
11-12 03:15:29.978  1069  1069 I ActivityThread: 	at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:3401)
11-12 03:15:29.978  1069  1069 I ActivityThread: 	at android.app.servertransaction.LaunchActivityItem.execute(LaunchActivityItem.java:83)
11-12 03:15:29.978  1069  1069 I ActivityThread: 	at android.app.servertransaction.TransactionExecutor.executeCallbacks(TransactionExecutor.java:135)
11-12 03:15:29.978  1069  1069 I ActivityThread: 	at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:95)
11-12 03:15:29.978  1069  1069 I ActivityThread: 	at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2016)
11-12 03:15:29.978  1069  1069 I ActivityThread: 	at android.os.Handler.dispatchMessage(Handler.java:107)
11-12 03:15:29.978  1069  1069 I ActivityThread: 	at android.os.Looper.loop(Looper.java:214)
11-12 03:15:29.978  1069  1069 I ActivityThread: 	at android.app.ActivityThread.main(ActivityThread.java:7368)
11-12 03:15:29.978  1069  1069 I ActivityThread: 	at java.lang.reflect.Method.invoke(Native Method)
11-12 03:15:29.978  1069  1069 I ActivityThread: 	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:492)
11-12 03:15:29.978  1069  1069 I ActivityThread: 	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:930)
----------------------------------------------------------------------------
```


##### 4.3、ActivityThread.performLaunchActivity()

``` java
   /**  Core implementation of activity launch. */
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        if (localLOGV) Slog.i(TAG, "performLaunchActivity",new RuntimeException("here").fillInStackTrace());;

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

        ContextImpl appContext = createBaseContextForActivity(r);
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
            }
        }

        try {
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);

            if (localLOGV) Slog.v(TAG, "Performing launch of " + r);
            if (localLOGV) Slog.v(
                    TAG, r + ": app=" + app
                    + ", appName=" + app.getPackageName()
                    + ", pkg=" + r.packageInfo.getPackageName()
                    + ", comp=" + r.intent.getComponent().toShortString()
                    + ", dir=" + r.packageInfo.getAppDir());

            if (activity != null) {
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                if (r.overrideConfig != null) {
                    config.updateFrom(r.overrideConfig);
                }
                if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                        + r.activityInfo.name + " with config " + config);
                Window window = null;
                if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                    window = r.mPendingRemoveWindow;
                    r.mPendingRemoveWindow = null;
                    r.mPendingRemoveWindowManager = null;
                }
                appContext.setOuterContext(activity);
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback,
                        r.assistToken);

                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }
                r.lastNonConfigurationInstances = null;
                checkAndBlockForNetworkAccess();
                activity.mStartedActivity = false;
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }

                activity.mCalled = false;
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onCreate()");
                }
                r.activity = activity;
            }
            r.setState(ON_CREATE);

            // updatePendingActivityConfiguration() reads from mActivities to update
            // ActivityClientRecord which runs in a different thread. Protect modifications to
            // mActivities to avoid race.
            synchronized (mResourcesManager) {
                mActivities.put(r.token, r);
            }

        } catch (SuperNotCalledException e) {
            throw e;

        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to start activity " + component
                    + ": " + e.toString(), e);
            }
        }

        return activity;
    }
------------------------------------------------------
11-12 03:15:29.983  1069  1069 I ActivityThread: performLaunchActivity
11-12 03:15:29.983  1069  1069 I ActivityThread: java.lang.RuntimeException: here
11-12 03:15:29.983  1069  1069 I ActivityThread: 	at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:3160)
11-12 03:15:29.983  1069  1069 I ActivityThread: 	at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:3413)
11-12 03:15:29.983  1069  1069 I ActivityThread: 	at android.app.servertransaction.LaunchActivityItem.execute(LaunchActivityItem.java:83)
11-12 03:15:29.983  1069  1069 I ActivityThread: 	at android.app.servertransaction.TransactionExecutor.executeCallbacks(TransactionExecutor.java:135)
11-12 03:15:29.983  1069  1069 I ActivityThread: 	at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:95)
11-12 03:15:29.983  1069  1069 I ActivityThread: 	at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2016)
11-12 03:15:29.983  1069  1069 I ActivityThread: 	at android.os.Handler.dispatchMessage(Handler.java:107)
11-12 03:15:29.983  1069  1069 I ActivityThread: 	at android.os.Looper.loop(Looper.java:214)
11-12 03:15:29.983  1069  1069 I ActivityThread: 	at android.app.ActivityThread.main(ActivityThread.java:7368)
11-12 03:15:29.983  1069  1069 I ActivityThread: 	at java.lang.reflect.Method.invoke(Native Method)
11-12 03:15:29.983  1069  1069 I ActivityThread: 	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:492)
11-12 03:15:29.983  1069  1069 I ActivityThread: 	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:930)
-------------------------------------------------------
```

- 初始化activity的context
- 利用classloader机制反射获取activity实例
- 创建app的入口application
- 执行 activity的attach方法   初始化window/父parent-DecorView/windowManage ,windowManager的子类为 - WindowManagerimpl 是view和window的操作类
- 执行activity的onCreate方法

至此executeCallbacks执行完毕，开始执行executeLifecycleState方法。先执行cycleToPath方法，生命周期状态是从ON_CREATE状态到ON_RESUME状态，中间有一个ON_START状态，所以会执行ActivityThread.handleStartActivity方法。

``` java
X:\frameworks\base\core\java\android\app\servertransaction\TransactionExecutor.java
    /**
     * Transition the client between states with an option not to perform the last hop in the
     * sequence. This is used when resolving lifecycle state request, when the last transition must
     * be performed with some specific parameters.
     */
    private void cycleToPath(ActivityClientRecord r, int finish, boolean excludeLastState,
            ClientTransaction transaction) {
        final int start = r.getLifecycleState();
        if (DEBUG_RESOLVER) {
            Slog.d(TAG, tId(transaction) + "Cycle activity: "
                    + getShortActivityName(r.token, mTransactionHandler)
                    + " from: " + getStateName(start) + " to: " + getStateName(finish)
                    + " excludeLastState: " + excludeLastState);
        }
        final IntArray path = mHelper.getLifecyclePath(start, finish, excludeLastState);
        performLifecycleSequence(r, path, transaction);
    }

    /** Transition the client through previously initialized state sequence. */
    private void performLifecycleSequence(ActivityClientRecord r, IntArray path,
            ClientTransaction transaction) {
        final int size = path.size();
        for (int i = 0, state; i < size; i++) {
            state = path.get(i);
            if (DEBUG_RESOLVER) {
                Slog.d(TAG, tId(transaction) + "Transitioning activity: "
                        + getShortActivityName(r.token, mTransactionHandler)
                        + " to state: " + getStateName(state));
            }
            switch (state) {
                case ON_CREATE:
                    mTransactionHandler.handleLaunchActivity(r, mPendingActions,
                            null /* customIntent */);
                    break;
                case ON_START:
                    mTransactionHandler.handleStartActivity(r, mPendingActions);
                    break;
                case ON_RESUME:
                    mTransactionHandler.handleResumeActivity(r.token, false /* finalStateRequest */,
                            r.isForward, "LIFECYCLER_RESUME_ACTIVITY");
                    break;
                case ON_PAUSE:
                    mTransactionHandler.handlePauseActivity(r.token, false /* finished */,
                            false /* userLeaving */, 0 /* configChanges */, mPendingActions,
                            "LIFECYCLER_PAUSE_ACTIVITY");
                    break;
                case ON_STOP:
                    mTransactionHandler.handleStopActivity(r.token, false /* show */,
                            0 /* configChanges */, mPendingActions, false /* finalStateRequest */,
                            "LIFECYCLER_STOP_ACTIVITY");
                    break;
                case ON_DESTROY:
                    mTransactionHandler.handleDestroyActivity(r.token, false /* finishing */,
                            0 /* configChanges */, false /* getNonConfigInstance */,
                            "performLifecycleSequence. cycling to:" + path.get(size - 1));
                    break;
                case ON_RESTART:
                    mTransactionHandler.performRestartActivity(r.token, false /* start */);
                    break;
                default:
                    throw new IllegalArgumentException("Unexpected lifecycle state: " + state);
            }
        }
    }
```
从ActivityThread.handleStartActivity方法经过多次跳转最后会调用Activity.onStart方法，至此cycleToPath方法执行完毕。

``` java
    frameworks/base/core/java/android/app/ActivityThread.java
    public void handleStartActivity(ActivityClientRecord r,
            PendingTransactionActions pendingActions) {
        ...
        // Start
        activity.performStart("handleStartActivity");
        r.setState(ON_START);
        ...
    }

    frameworks/base/core/java/android/app/Activity.java
        final void performStart(String reason) {
        ...
        mInstrumentation.callActivityOnStart(this);
        ...
    }

    frameworks/base/core/java/android/app/Instrumentation.java
    public void callActivityOnStart(Activity activity) {
        activity.onStart();
    }
----------------------------------------------------------

11-12 03:15:30.031  1069  1069 I ActivityThread: handleStartActivity
11-12 03:15:30.031  1069  1069 I ActivityThread: java.lang.RuntimeException: here
11-12 03:15:30.031  1069  1069 I ActivityThread: 	at android.app.ActivityThread.handleStartActivity(ActivityThread.java:3294)
11-12 03:15:30.031  1069  1069 I ActivityThread: 	at android.app.servertransaction.TransactionExecutor.performLifecycleSequence(TransactionExecutor.java:221)
11-12 03:15:30.031  1069  1069 I ActivityThread: 	at android.app.servertransaction.TransactionExecutor.cycleToPath(TransactionExecutor.java:201)
11-12 03:15:30.031  1069  1069 I ActivityThread: 	at android.app.servertransaction.TransactionExecutor.executeLifecycleState(TransactionExecutor.java:173)
11-12 03:15:30.031  1069  1069 I ActivityThread: 	at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:97)
11-12 03:15:30.031  1069  1069 I ActivityThread: 	at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2016)
11-12 03:15:30.031  1069  1069 I ActivityThread: 	at android.os.Handler.dispatchMessage(Handler.java:107)
11-12 03:15:30.031  1069  1069 I ActivityThread: 	at android.os.Looper.loop(Looper.java:214)
11-12 03:15:30.031  1069  1069 I ActivityThread: 	at android.app.ActivityThread.main(ActivityThread.java:7368)
11-12 03:15:30.031  1069  1069 I ActivityThread: 	at java.lang.reflect.Method.invoke(Native Method)
11-12 03:15:30.031  1069  1069 I ActivityThread: 	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:492)
11-12 03:15:30.031  1069  1069 I ActivityThread: 	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:930)
----------------------------------------------------------
```
执行完毕cycleToPath，开始执行ResumeActivityItem.execute方法。

``` java
   frameworks/base/core/java/android/app/servertransaction/ResumeActivityItem.java
    @Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        ...
        client.handleResumeActivity(token, true /* finalStateRequest */, mIsForward,
                "RESUME_ACTIVITY");
        ...
    }

    frameworks/base/core/java/android/app/ActivityThread.java
    @Override
    public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
            String reason) {
        ...
        // TODO Push resumeArgs into the activity for consideration
        final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
        ...
        Looper.myQueue().addIdleHandler(new Idler());
        ...
    }

    public ActivityClientRecord performResumeActivity(IBinder token, boolean finalStateRequest,
            String reason) {
        ...
        try {
            r.activity.onStateNotSaved();
            r.activity.mFragments.noteStateNotSaved();
            checkAndBlockForNetworkAccess();
            if (r.pendingIntents != null) {
                deliverNewIntents(r, r.pendingIntents);
                r.pendingIntents = null;
            }
            if (r.pendingResults != null) {
                deliverResults(r, r.pendingResults, reason);
                r.pendingResults = null;
            }
            r.activity.performResume(r.startsNotResumed, reason);

            r.state = null;
            r.persistentState = null;
            r.setState(ON_RESUME);
        } catch (Exception e) {
            if (!mInstrumentation.onException(r.activity, e)) {
                throw new RuntimeException("Unable to resume activity "
                        + r.intent.getComponent().toShortString() + ": " + e.toString(), e);
            }
        }
        ...
        return r;
    }

    frameworks/base/core/java/android/app/Activity.java
    final void performResume(boolean followedByPause, String reason) {
        performRestart(true /* start */, reason);
        ...
        // mResumed is set by the instrumentation
        mInstrumentation.callActivityOnResume(this);
        ...
    }

    frameworks/base/core/java/android/app/Instrumentation.java
    public void callActivityOnResume(Activity activity) {
        activity.mResumed = true;
        activity.onResume();
        ...
    }
------------------------------------------------------------------------------

11-12 03:15:30.043  1069  1069 I ActivityThread: handleResumeActivity
11-12 03:15:30.043  1069  1069 I ActivityThread: java.lang.RuntimeException: here
11-12 03:15:30.043  1069  1069 I ActivityThread: 	at android.app.ActivityThread.handleResumeActivity(ActivityThread.java:4241)
11-12 03:15:30.043  1069  1069 I ActivityThread: 	at android.app.servertransaction.ResumeActivityItem.execute(ResumeActivityItem.java:52)
11-12 03:15:30.043  1069  1069 I ActivityThread: 	at android.app.servertransaction.TransactionExecutor.executeLifecycleState(TransactionExecutor.java:176)
11-12 03:15:30.043  1069  1069 I ActivityThread: 	at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:97)
11-12 03:15:30.043  1069  1069 I ActivityThread: 	at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2016)
11-12 03:15:30.043  1069  1069 I ActivityThread: 	at android.os.Handler.dispatchMessage(Handler.java:107)
11-12 03:15:30.043  1069  1069 I ActivityThread: 	at android.os.Looper.loop(Looper.java:214)
11-12 03:15:30.043  1069  1069 I ActivityThread: 	at android.app.ActivityThread.main(ActivityThread.java:7368)
11-12 03:15:30.043  1069  1069 I ActivityThread: 	at java.lang.reflect.Method.invoke(Native Method)
11-12 03:15:30.043  1069  1069 I ActivityThread: 	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:492)
11-12 03:15:30.043  1069  1069 I ActivityThread: 	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:930)]


-----------------------------------------------------------------------------
```

经过上面的多次跳转最终调用到Activity.onResume方法，Activity启动完毕。

``` java
我们来看一下Activity 的生命周期：

> protected void onCreate(); 
> protected void onStart(); 
> protected void onResume(); 
> protected void onPause(); 
> protected void onStop();
> protected void onDestory();
```
- performPauseActivityIfNeeded
``` java
performPauseActivityIfNeeded
11-12 03:16:44.970   728   728 I ActivityThread: performPauseActivityIfNeeded
11-12 03:16:44.970   728   728 I ActivityThread: java.lang.RuntimeException: here
11-12 03:16:44.970   728   728 I ActivityThread: 	at android.app.ActivityThread.performPauseActivityIfNeeded(ActivityThread.java:4493)
11-12 03:16:44.970   728   728 I ActivityThread: 	at android.app.ActivityThread.performPauseActivity(ActivityThread.java:4461)
11-12 03:16:44.970   728   728 I ActivityThread: 	at android.app.ActivityThread.handleRelaunchActivityInner(ActivityThread.java:5261)
11-12 03:16:44.970   728   728 I ActivityThread: 	at android.app.ActivityThread.handleRelaunchActivity(ActivityThread.java:5199)
11-12 03:16:44.970   728   728 I ActivityThread: 	at android.app.servertransaction.ActivityRelaunchItem.execute(ActivityRelaunchItem.java:69)
11-12 03:16:44.970   728   728 I ActivityThread: 	at android.app.servertransaction.TransactionExecutor.executeCallbacks(TransactionExecutor.java:135)
11-12 03:16:44.970   728   728 I ActivityThread: 	at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:95)
11-12 03:16:44.970   728   728 I ActivityThread: 	at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2016)
11-12 03:16:44.970   728   728 I ActivityThread: 	at android.os.Handler.dispatchMessage(Handler.java:107)
11-12 03:16:44.970   728   728 I ActivityThread: 	at android.os.Looper.loop(Looper.java:214)
11-12 03:16:44.970   728   728 I ActivityThread: 	at android.app.ActivityThread.main(ActivityThread.java:7368)
11-12 03:16:44.970   728   728 I ActivityThread: 	at java.lang.reflect.Method.invoke(Native Method)
11-12 03:16:44.970   728   728 I ActivityThread: 	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:492)
11-12 03:16:44.970   728   728 I ActivityThread: 	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:930
```

- performStopActivityInner
``` java
11-12 03:15:31.339   729   729 I ActivityThread: performStopActivityInner
11-12 03:15:31.339   729   729 I ActivityThread: java.lang.RuntimeException: here
11-12 03:15:31.339   729   729 I ActivityThread: 	at android.app.ActivityThread.performStopActivityInner(ActivityThread.java:4565)
11-12 03:15:31.339   729   729 I ActivityThread: 	at android.app.ActivityThread.handleStopActivity(ActivityThread.java:4679)
11-12 03:15:31.339   729   729 I ActivityThread: 	at android.app.servertransaction.TransactionExecutor.performLifecycleSequence(TransactionExecutor.java:233)
11-12 03:15:31.339   729   729 I ActivityThread: 	at android.app.servertransaction.TransactionExecutor.cycleToPath(TransactionExecutor.java:201)
11-12 03:15:31.339   729   729 I ActivityThread: 	at android.app.servertransaction.TransactionExecutor.executeLifecycleState(TransactionExecutor.java:173)
11-12 03:15:31.339   729   729 I ActivityThread: 	at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:97)
11-12 03:15:31.339   729   729 I ActivityThread: 	at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2016)
11-12 03:15:31.339   729   729 I ActivityThread: 	at android.os.Handler.dispatchMessage(Handler.java:107)
11-12 03:15:31.339   729   729 I ActivityThread: 	at android.os.Looper.loop(Looper.java:214)
11-12 03:15:31.339   729   729 I ActivityThread: 	at android.app.ActivityThread.main(ActivityThread.java:7368)
11-12 03:15:31.339   729   729 I ActivityThread: 	at java.lang.reflect.Method.invoke(Native Method)
11-12 03:15:31.339   729   729 I ActivityThread: 	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:492)
11-12 03:15:31.339   729   729 I ActivityThread: 	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:930)
```

- performDestroyActivity 
``` java
11-12 03:15:31.341   729   729 I ActivityThread: performDestroyActivity
11-12 03:15:31.341   729   729 I ActivityThread: java.lang.RuntimeException: here
11-12 03:15:31.341   729   729 I ActivityThread: 	at android.app.ActivityThread.performDestroyActivity(ActivityThread.java:4909)
11-12 03:15:31.341   729   729 I ActivityThread: 	at android.app.ActivityThread.handleDestroyActivity(ActivityThread.java:4982)
11-12 03:15:31.341   729   729 I ActivityThread: 	at android.app.servertransaction.DestroyActivityItem.execute(DestroyActivityItem.java:44)
11-12 03:15:31.341   729   729 I ActivityThread: 	at android.app.servertransaction.TransactionExecutor.executeLifecycleState(TransactionExecutor.java:176)
11-12 03:15:31.341   729   729 I ActivityThread: 	at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:97)
11-12 03:15:31.341   729   729 I ActivityThread: 	at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2016)
11-12 03:15:31.341   729   729 I ActivityThread: 	at android.os.Handler.dispatchMessage(Handler.java:107)
11-12 03:15:31.341   729   729 I ActivityThread: 	at android.os.Looper.loop(Looper.java:214)
11-12 03:15:31.341   729   729 I ActivityThread: 	at android.app.ActivityThread.main(ActivityThread.java:7368)
11-12 03:15:31.341   729   729 I ActivityThread: 	at java.lang.reflect.Method.invoke(Native Method)
11-12 03:15:31.341   729   729 I ActivityThread: 	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:492)
11-12 03:15:31.341   729   729 I ActivityThread: 	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:930)
```

前面我们分析了onCreate()、onStart() 、onResume()、onPause()。 Activity 暂停销毁时的 、onStop() 、onDestroy() 回调都与前面的过程大同小异，这里就只列举相应的方法栈，不再继续描述。

``` java
----------------------------------------
Line 81035: 11-12 03:15:30.004  1069  1069 V Activity: onCreate com.android.testred.TestActivity@e170766: null
Line 81056: 11-12 03:15:30.023  1069  1069 I am_on_create_called: [0,com.android.testred.TestActivity,performCreate]
Line 81078: 11-12 03:15:30.029   433   451 I ActivityRecord: Taking options for ActivityRecord{563ed0d u0 com.android.testred/.TestActivity t2} callers=com.android.server.wm.ActivityTaskManagerService.getActivityOptions:2985 android.app.IActivityTaskManager$Stub.onTransact:2569 android.os.Binder.execTransactInternal:1021 android.os.Binder.execTransact:994 <bottom of call stack> <bottom of call stack> 
Line 81079: 11-12 03:15:30.030   433   451 V ActivityTaskManagerService: Report configuration: Token{b3bac1f ActivityRecord{563ed0d u0 com.android.testred/.TestActivity t2}} null null
Line 81106: 11-12 03:15:30.037   433   714 I ActivityRecord: Taking options for ActivityRecord{563ed0d u0 com.android.testred/.TestActivity t2} callers=com.android.server.wm.ActivityTaskManagerService.getActivityOptions:2985 android.app.IActivityTaskManager$Stub.onTransact:2569 android.os.Binder.execTransactInternal:1021 android.os.Binder.execTransact:994 <bottom of call stack> <bottom of call stack> 
Line 81107: 11-12 03:15:30.038  1069  1069 V Activity: onStart com.android.testred.TestActivity@e170766
Line 81108: 11-12 03:15:30.039  1069  1069 I am_on_start_called: [0,com.android.testred.TestActivity,handleStartActivity]
Line 81138: 11-12 03:15:30.043  1069  1069 V ActivityThread: Performing resume of ActivityRecord{9cb94af token=android.os.BinderProxy@fbcf48e {com.android.testred/com.android.testred.TestActivity}} finished=false
Line 81156: 11-12 03:15:30.044  1069  1069 V Activity: onResume com.android.testred.TestActivity@e170766
Line 81157: 11-12 03:15:30.045  1069  1069 I am_on_resume_called: [0,com.android.testred.TestActivity,RESUME_ACTIVITY]
---------------------------------------
```