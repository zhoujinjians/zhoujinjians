---
title: Android P Graphics System（六）：Activity启动流程 && Surface创建分析
cover: https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/hexo.themes/bing-wallpaper-2018.04.41.jpg
categories:
  - Graphics
tags:
  - Android
  - Graphics
toc: true
abbrlink: 20190725
date: 2019-07-25 09:25:00
---
--------------------------------------------------------------------------------

注：文章都是通过阅读各位前辈总结的资料 Android 9.0 && Linux（Kernel 3.18）Qualcomm平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除（◕‿◕），转载请注明出处（©Qualcomm ©Android @Linux 版权所有），谢谢。
（==**文章基于 Kernel-3.18**==）&&（==**文章基于 Android 9.0**==）
[【开发板 Intrinsyc Open-Q™ 820 µSOM Development Kit】](https://www.intrinsyc.com/snapdragon-embedded-development-kits/open-q-820-usom-development-kit/)
[【开发板 Android 9.0 && Linux（Kernel 3.18）源码链接】](https://gitlab.com/zhoujinjianz/apq8096_la.um.7.5.r1-03100-8x96.0_p_v5.0)

正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢！！！

[（1）【Android Display System】](http://charlesvincent.cc/)
[（2）【android 9.0 activity启动源码分析】](https://www.jianshu.com/p/50e4b48ed15c)
[（3）【（Android 9.0）Activity启动流程源码分析】](https://blog.csdn.net/lj19851227/article/details/82562115)


--------------------------------------------------------------------------------
==源码（部分）==：
**frameworks/base/services/core/java/com/android/server/am/**
**frameworks/base/core/java/android/app/**

----------


####  一、概述
基于Android 9.0的源码剖析， 分析Android Activity启动流程：

----------


#### 1.1、Task和Stack
Android系统中的每一个Activity都位于一个Task中。一个Task可以包含多个Activity，同一个Activity也可能有多个实例。 在AndroidManifest.xml中，我们可以通过android:launchMode来控制Activity在Task中的实例。

另外，在startActivity的时候，我们也可以通过setFlag 来控制启动的Activity在Task中的实例。

Task管理的意义还在于近期任务列表以及Back栈。 当你通过多任务键（有些设备上是长按Home键，有些设备上是专门提供的多任务键）调出多任务时，其实就是从ActivityManagerService获取了最近启动的Task列表。

Back栈管理了当你在Activity上点击Back键，当前Activity销毁后应该跳转到哪一个Activity的逻辑。关于Task和Back栈，请参见这里：[Tasks and Back Stack](https://developer.android.com/guide/components/tasks-and-back-stack.html)。

其实在ActivityManagerService与WindowManagerService内部管理中，在Task之外，还有一层容器，这个容器应用开发者和用户可能都不会感觉到或者用到，但它却非常重要，那就是Stack。 下文中，我们将看到，Android系统中的多窗口管理，就是建立在Stack的数据结构上的。 一个Stack中包含了多个Task，一个Task中包含了多个Activity（Window），下图描述了它们的关系：

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/display.system/Android.PG6.Stack_Task.png)

另外还有一点需要注意的是，ActivityManagerService和WindowManagerService中的Task和Stack结构是一一对应的，对应关系对于如下：

ActivityStack <–> TaskStack
TaskRecord <–> Task
即，ActivityManagerService中的每一个ActivityStack或者TaskRecord在WindowManagerService中都有对应的TaskStack和Task，这两类对象都有唯一的id（id是int类型），它们通过id进行关联。


##### 1.2、相关类简介

- Instrumentation
用于实现应用程序测试代码的基类。当在打开仪器的情况下运行时，这个类将在任何应用程序代码之前为您实例化，允许您监视系统与应用程序的所有交互。可以通过AndroidManifest.xml的标签描述该类的实现。

- ActivityManager
该类提供与Activity、Service和Process相关的信息以及交互方法， 可以被看作是ActivityManagerService的辅助类。

- ActivityManagerService
Android中最核心的服务，主要负责系统中四大组件的启动、切换、调度及应用程序的管理和调度等工作。

- ActivityThread
管理应用程序进程中主线程的执行，根据Activity管理者的请求调度和执行activities、broadcasts及其相关的操作。

ActivityStack
负责单个Activity栈的状态和管理。

- ActivityStackSupervisor
负责所有Activity栈的管理。内部管理了mHomeStack、mFocusedStack和mLastFocusedStack三个Activity栈。其中，mHomeStack管理的是Launcher相关的Activity栈；mFocusedStack管理的是当前显示在前台Activity的Activity栈；mLastFocusedStack管理的是上一次显示在前台Activity的Activity栈。

- ClientLifecycleManager
用来管理多个客户端生命周期执行请求的管理类。

##### 1.3、小结：

> 用户从Launcher程序点击应用图标可启动应用的入口Activity，Activity启动时需要多个进程之间的交互，Android系统中有一个zygote进程专用于孵化Android框架层和应用层程序的进程。还有一个system_server进程，该进程里运行了很多binder
> service，例如ActivityManagerService，PackageManagerService，WindowManagerService，这些binder
> service分别运行在不同的线程中，其中ActivityManagerService负责管理Activity栈，应用进程，task。
> 用户在Launcher程序里点击应用图标时，会通知ActivityManagerService启动应用的入口Activity，ActivityManagerService发现这个应用还未启动，则会通知Zygote进程孵化出应用进程，然后在这个应用进程里执行ActivityThread的main方法。应用进程接下来通知ActivityManagerService应用进程已启动，ActivityManagerService保存应用进程的一个代理对象，这样ActivityManagerService可以通过这个代理对象控制应用进程，然后ActivityManagerService通知应用进程创建入口Activity的实例，并执行它的生命周期方法。



#### 二、 开始请求执行启动Activity

##### 2.1、Activity.startActivity()
从Launcher启动应用的时候，经过调用会执行Activity中的startActivity。

``` java
[-> Activity.java]
......
    @Override
    public void startActivity(Intent intent) {
        this.startActivity(intent, null);
    }

......
[-> Activity.java]
    @Override
    public void startActivity(Intent intent, @Nullable Bundle options) {
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            // Note we want to go through this call for compatibility with
            // applications that may have overridden the method.
            startActivityForResult(intent, -1);
        }
    }

```

##### 2.2、Activity.startActivityForResult()

``` java
[-> Activity.java]
    @Override
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

可以发现execStartActivity方法传递的几个参数： 
this，为启动Activity的对象； 
mMainThread.getApplicationThread()，为Binder对象，是主进程的context对象； 
mToken，也是一个Binder对象，指向了服务端一个ActivityRecord对象； 
who，为启动的Activity； 
intent，启动的Intent对象； 
requestCode，请求码； 
options，参数；
##### 2.3、Instrumentation.execStartActivity()

``` java
[-> Instrumentation.java]
    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        android.util.SeempLog.record_str(377, intent.toString());
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        Uri referrer = target != null ? target.onProvideReferrer() : null;
        if (referrer != null) {
            intent.putExtra(Intent.EXTRA_REFERRER, referrer);
        }
        if (mActivityMonitors != null) {
            synchronized (mSync) {
                final int N = mActivityMonitors.size();
                for (int i=0; i<N; i++) {
                    final ActivityMonitor am = mActivityMonitors.get(i);
                    ActivityResult result = null;
                    if (am.ignoreMatchingSpecificIntents()) {
                        result = am.onStartActivity(intent);
                    }
                    if (result != null) {
                        am.mHits++;
                        return result;
                    } else if (am.match(who, null, intent)) {
                        am.mHits++;
                        if (am.isBlocking()) {
                            return requestCode >= 0 ? am.getResult() : null;
                        }
                        break;
                    }
                }
            }
        }
        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess(who);
            int result = ActivityManager.getService()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }

```

关于 ActivityManager.getService()返回的是? 

``` java
 [->ActivityManagerNative.java]
    public static IActivityManager getService() {
        return IActivityManagerSingleton.get();
    }

```

直接返回的是gDefault.get()，那么gDefault又是什么呢？

``` java
    [->IActivityManagerSingleton.java]
    private static final Singleton<IActivityManager> IActivityManagerSingleton =
            new Singleton<IActivityManager>() {
                @Override
                protected IActivityManager create() {
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                    final IActivityManager am = IActivityManager.Stub.asInterface(b);
                    return am;
                }
            };
   
   \android\out\target\common\obj\JAVA_LIBRARIES\framework_intermediates\core\java\android\app\IActivityManager.java


public static android.app.IActivityManager asInterface(android.os.IBinder obj)
{
if ((obj==null)) {
return null;
}
android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
if (((iin!=null)&&(iin instanceof android.app.IActivityManager))) {
return ((android.app.IActivityManager)iin);
}
return new android.app.IActivityManager.Stub.Proxy(obj);
}
```


最后直接返回一个android.app.IActivityManager.Stub.Proxy对象，
此处startActivity()的共有10个参数, 下面说说每个参数传递AMP.startActivity()每一项的对应值。

好吧，最后直接返回一个对象，而继承与IActivityManager，到了这里就引出了我们android系统中很重要的一个概念：Binder机制。我们知道应用进程与SystemServer进程属于两个不同的进程，进程之间需要通讯，android系统采取了自身设计的Binder机制，这里的和ActivityManagerNative都是继承与IActivityManager的而SystemServer进程中的ActivityManagerService对象则继承与ActivityManagerNative。简单的表示： 
Binder接口 –> IActivityManager.aidl / –> ActivityManagerService

这样，IActivityManager.Stub与相当于一个Binder的客户端而ActivityManagerService相当于Binder的服务端，这样当IActivityManager.Stub.Proxy调用接口方法的时候底层通过Binder driver就会将请求数据与请求传递给server端，并在server端执行具体的接口逻辑。

好了，说了这么多我们知道这里的IActivityManager.Stub.Proxy是ActivityManagerService在应用进程的一个client就好了，通过它就可以滴啊用ActivityManagerService的方法了。

##### 2.4、IActivityManager.Stub.Proxy.startActivity()
ActivityManager.getService()方法会返回一个对象，那么我们看一下对象的startActivity方法：

``` java
\android\out\target\common\obj\JAVA_LIBRARIES\framework_intermediates\core\java\android\app\IActivityManager.java


Override public int startActivity( android.app.IApplicationThread caller, java.lang.String callingPackage, android.content.Intent intent, java.lang.String resolvedType, android.os.IBinder resultTo, java.lang.String resultWho, int requestCode, int flags, android.app.ProfilerInfo profilerInfo, android.os.Bundle options ) throws android.os.RemoteException
{
	android.os.Parcel	_data	= android.os.Parcel.obtain();
	android.os.Parcel	_reply	= android.os.Parcel.obtain();
	int			_result;
	try {
		_data.writeInterfaceToken( DESCRIPTOR );
		_data.writeStrongBinder( ( ( (caller != null) ) ? (caller.asBinder() ) : (null) ) );
		_data.writeString( callingPackage );
		if ( (intent != null) )
		{
			_data.writeInt( 1 );
			intent.writeToParcel( _data, 0 );
		}else  {
			_data.writeInt( 0 );
		}
		_data.writeString( resolvedType );
		_data.writeStrongBinder( resultTo );
		_data.writeString( resultWho );
		_data.writeInt( requestCode );
		_data.writeInt( flags );
		if ( (profilerInfo != null) )
		{
			_data.writeInt( 1 );
			profilerInfo.writeToParcel( _data, 0 );
		}else  {
			_data.writeInt( 0 );
		}
		if ( (options != null) )
		{
			_data.writeInt( 1 );
			options.writeToParcel( _data, 0 );
		}else  {
			_data.writeInt( 0 );
		}
		mRemote.transact( Stub.TRANSACTION_startActivity, _data, _reply, 0 );
		_reply.readException();
		_result = _reply.readInt();
	}
	finally {
		_reply.recycle();
		_data.recycle();
	}
	return(_result);
}

```

##### 2.5、IActivityManager.Stub..onTransact()

``` java
\android\out\target\common\obj\JAVA_LIBRARIES\framework_intermediates\core\java\android\app\IActivityManager.java

    @Override
public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
        throws RemoteException {
    switch (code) {
case TRANSACTION_startActivity :
{
	data.enforceInterface( DESCRIPTOR );
	android.app.IApplicationThread _arg0;
	_arg0 = android.app.IApplicationThread.Stub.asInterface( data.readStrongBinder() );
	java.lang.String _arg1;
	_arg1 = data.readString();
	android.content.Intent _arg2;
	if ( (0 != data.readInt() ) )
	{
		_arg2 = android.content.Intent.CREATOR.createFromParcel( data );
	}else  {
		_arg2 = null;
	}
	java.lang.String _arg3;
	_arg3 = data.readString();
	android.os.IBinder _arg4;
	_arg4 = data.readStrongBinder();
	java.lang.String _arg5;
	_arg5 = data.readString();
	int _arg6;
	_arg6 = data.readInt();
	int _arg7;
	_arg7 = data.readInt();
	android.app.ProfilerInfo _arg8;
	if ( (0 != data.readInt() ) )
	{
		_arg8 = android.app.ProfilerInfo.CREATOR.createFromParcel( data );
	}else  {
		_arg8 = null;
	}
	android.os.Bundle _arg9;
	if ( (0 != data.readInt() ) )
	{
		_arg9 = android.os.Bundle.CREATOR.createFromParcel( data );
	}else  {
		_arg9 = null;
	}
	int _result = this.startActivity( _arg0, _arg1, _arg2, _arg3, _arg4, _arg5, _arg6, _arg7, _arg8, _arg9 );
	reply.writeNoException();
	reply.writeInt( _result );
	return(true);
}
    ......
  }
```

这里就涉及到了具体的Binder数据传输机制了，我们不做过多的分析，知道通过数据传输之后就会调用SystemServer进程的ActivityManagerService的startActivity就好了。

以上其实都是发生在应用进程中，下面开始调用的ActivityManagerService的执行时发生在SystemServer进程。

#### 三、 ActivityManagerService接收启动Activity的请求


##### 3.1、ActivityManagerService.startActivity()

``` java
[-> ActivityManagerService.java]
    @Override
    public final int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                resultWho, requestCode, startFlags, profilerInfo, bOptions,
                UserHandle.getCallingUserId());
    }
    
    @Override
    public final int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                resultWho, requestCode, startFlags, profilerInfo, bOptions,
                UserHandle.getCallingUserId());
    }

    public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId,
            boolean validateIncomingUser) {
        enforceNotIsolatedCaller("startActivity");

        userId = mActivityStartController.checkTargetUser(userId, validateIncomingUser,
                Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

        // TODO: Switch to user app stacks here.
        return mActivityStartController.obtainStarter(intent, "startActivityAsUser")
                .setCaller(caller)
                .setCallingPackage(callingPackage)
                .setResolvedType(resolvedType)
                .setResultTo(resultTo)
                .setResultWho(resultWho)
                .setRequestCode(requestCode)
                .setStartFlags(startFlags)
                .setProfilerInfo(profilerInfo)
                .setActivityOptions(bOptions)
                .setMayWait(userId)
                .execute();

    }

```

##### 3.2、ActivityStarter.execute()
这个方法是基于初始的参数来启动activity,可以看到有两种启动activity的代码块,startActivityMayWait最终会调用startActivity所以:我们来看startActivityMayWait源码
``` java
[->ActivityStarter.java]
 /**
     * Starts an activity based on the request parameters provided earlier.
     * @return The starter result.
     */
    int execute() {
        try {
            // TODO(b/64750076): Look into passing request directly to these methods to allow
            // for transactional diffs and preprocessing.
            if (mRequest.mayWait) {
                return startActivityMayWait(mRequest.caller, mRequest.callingUid,
                        mRequest.callingPackage, mRequest.intent, mRequest.resolvedType,
                        mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                        mRequest.resultWho, mRequest.requestCode, mRequest.startFlags,
                        mRequest.profilerInfo, mRequest.waitResult, mRequest.globalConfig,
                        mRequest.activityOptions, mRequest.ignoreTargetSecurity, mRequest.userId,
                        mRequest.inTask, mRequest.reason,
                        mRequest.allowPendingRemoteAnimationRegistryLookup);
            } else {
                return startActivity(mRequest.caller, mRequest.intent, mRequest.ephemeralIntent,
                        mRequest.resolvedType, mRequest.activityInfo, mRequest.resolveInfo,
                        mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                        mRequest.resultWho, mRequest.requestCode, mRequest.callingPid,
                        mRequest.callingUid, mRequest.callingPackage, mRequest.realCallingPid,
                        mRequest.realCallingUid, mRequest.startFlags, mRequest.activityOptions,
                        mRequest.ignoreTargetSecurity, mRequest.componentSpecified,
                        mRequest.outActivity, mRequest.inTask, mRequest.reason,
                        mRequest.allowPendingRemoteAnimationRegistryLookup);
            }
        } finally {
            onExecutionComplete();
        }
    }


   private int startActivityMayWait(IApplicationThread caller, int callingUid,
            String callingPackage, Intent intent, String resolvedType,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, WaitResult outResult,
            Configuration globalConfig, SafeActivityOptions options, boolean ignoreTargetSecurity,
            int userId, TaskRecord inTask, String reason,
            boolean allowPendingRemoteAnimationRegistryLookup) {
        .......
        ResolveInfo rInfo = mSupervisor.resolveIntent(intent, resolvedType, userId,
                0 /* matchFlags */,
                        computeResolveFilterUid(
                                callingUid, realCallingUid, mRequest.filterCallingUid));
        ......
        // Collect information about the target of the Intent.
        ActivityInfo aInfo = mSupervisor.resolveActivity(intent, rInfo, startFlags, profilerInfo);

        synchronized (mService) {
            final ActivityStack stack = mSupervisor.mFocusedStack;
            stack.mConfigWillChange = globalConfig != null
                    && mService.getGlobalConfiguration().diff(globalConfig) != 0;
            if (DEBUG_CONFIGURATION) Slog.v(TAG_CONFIGURATION,
                    "Starting activity when config will change = " + stack.mConfigWillChange);

            final long origId = Binder.clearCallingIdentity();

            if (aInfo != null &&
                    (aInfo.applicationInfo.privateFlags
                            & ApplicationInfo.PRIVATE_FLAG_CANT_SAVE_STATE) != 0 &&
                    mService.mHasHeavyWeightFeature) {
                // This may be a heavy-weight process!  Check to see if we already
                // have another, different heavy-weight process running.
                if (aInfo.processName.equals(aInfo.applicationInfo.packageName)) {
                    ......
                        Intent newIntent = new Intent();
                        ......
                        newIntent.putExtra(HeavyWeightSwitcherActivity.KEY_INTENT,
                                new IntentSender(target));
                        ......
                        newIntent.putExtra(HeavyWeightSwitcherActivity.KEY_NEW_APP,
                                aInfo.packageName);
                        newIntent.setFlags(intent.getFlags());
                        newIntent.setClassName("android",
                                HeavyWeightSwitcherActivity.class.getName());
                        intent = newIntent;
                        resolvedType = null;
                        caller = null;
                        callingUid = Binder.getCallingUid();
                        callingPid = Binder.getCallingPid();
                        componentSpecified = true;
                        rInfo = mSupervisor.resolveIntent(intent, null /*resolvedType*/, userId,
                                0 /* matchFlags */, computeResolveFilterUid(
                                        callingUid, realCallingUid, mRequest.filterCallingUid));
                        aInfo = rInfo != null ? rInfo.activityInfo : null;
                        if (aInfo != null) {
                            aInfo = mService.getActivityInfoForUser(aInfo, userId);
                        }
                    }
                }
            }

            final ActivityRecord[] outRecord = new ActivityRecord[1];
            int res = startActivity(caller, intent, ephemeralIntent, resolvedType, aInfo, rInfo,
                    voiceSession, voiceInteractor, resultTo, resultWho, requestCode, callingPid,
                    callingUid, callingPackage, realCallingPid, realCallingUid, startFlags, options,
                    ignoreTargetSecurity, componentSpecified, outRecord, inTask, reason,
                    allowPendingRemoteAnimationRegistryLookup);

            Binder.restoreCallingIdentity(origId);

            ......
            return res;
        }
    }


```


##### 3.3、ActivityStarter.startActivity()


``` java
[->ActivityStarter.java]
    private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
            String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
            String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
            SafeActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
            ActivityRecord[] outActivity, TaskRecord inTask, String reason,
            boolean allowPendingRemoteAnimationRegistryLookup) {

        if (TextUtils.isEmpty(reason)) {
            throw new IllegalArgumentException("Need to specify a reason.");
        }
        mLastStartReason = reason;
        mLastStartActivityTimeMs = System.currentTimeMillis();
        mLastStartActivityRecord[0] = null;

        mLastStartActivityResult = startActivity(caller, intent, ephemeralIntent, resolvedType,
                aInfo, rInfo, voiceSession, voiceInteractor, resultTo, resultWho, requestCode,
                callingPid, callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
                options, ignoreTargetSecurity, componentSpecified, mLastStartActivityRecord,
                inTask, allowPendingRemoteAnimationRegistryLookup);

        if (outActivity != null) {
            // mLastStartActivityRecord[0] is set in the call to startActivity above.
            outActivity[0] = mLastStartActivityRecord[0];
        }

        return getExternalResult(mLastStartActivityResult);
    }

    private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
                IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
                ActivityRecord[] outActivity) {
        int result = START_CANCELED;
        try {
            mService.mWindowManager.deferSurfaceLayout();
            result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
                    startFlags, doResume, options, inTask, outActivity);
        } finally {
            // If we are not able to proceed, disassociate the activity from the task. Leaving an
            // activity in an incomplete state can lead to issues, such as performing operations
            // without a window container.
            final ActivityStack stack = mStartActivity.getStack();
            if (!ActivityManager.isStartResultSuccessful(result) && stack != null) {
                stack.finishActivityLocked(mStartActivity, RESULT_CANCELED,
                        null /* intentResultData */, "startActivity", true /* oomAdj */);
            }
            mService.mWindowManager.continueSurfaceLayout();
        }

        postStartActivityProcessing(r, result, mTargetStack);

        return result;
    }
```


##### 3.4、ActivityStarter.startActivityUnchecked()

``` java
[->ActivityStarter.java]
   private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
            ActivityRecord[] outActivity) {

        .......

        mTargetStack.startActivityLocked(mStartActivity, topFocused, newTask, mKeepCurTransition,
                mOptions);
        if (mDoResume) {
            final ActivityRecord topTaskActivity =
                    mStartActivity.getTask().topRunningActivityLocked();
            if (!mTargetStack.isFocusable()
                    || (topTaskActivity != null && topTaskActivity.mTaskOverlay
                    && mStartActivity != topTaskActivity)) {
                // If the activity is not focusable, we can't resume it, but still would like to
                // make sure it becomes visible as it starts (this will also trigger entry
                // animation). An example of this are PIP activities.
                // Also, we don't want to resume activities in a task that currently has an overlay
                // as the starting activity just needs to be in the visible paused state until the
                // over is removed.
                mTargetStack.ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
                // Go ahead and tell window manager to execute app transition for this activity
                // since the app transition will not be triggered through the resume channel.
                mService.mWindowManager.executeAppTransition();
            } else {
                // If the target stack was not previously focusable (previous top running activity
                // on that stack was not visible) then any prior calls to move the stack to the
                // will not update the focused stack.  If starting the new activity now allows the
                // task stack to be focusable, then ensure that we now update the focused stack
                // accordingly.
                if (mTargetStack.isFocusable() && !mSupervisor.isFocusedStack(mTargetStack)) {
                    mTargetStack.moveToFront("startActivityUnchecked");
                }
                mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity,
                        mOptions);
            }
        } else if (mStartActivity != null) {
            mSupervisor.mRecentTasks.add(mStartActivity.getTask());
        }
        mSupervisor.updateUserStackLocked(mStartActivity.userId, mTargetStack);

        mSupervisor.handleNonResizableTaskIfNeeded(mStartActivity.getTask(), preferredWindowingMode,
                preferredLaunchDisplayId, mTargetStack);

        return START_SUCCESS;
    }
```



##### 3.4.1、ActivityStack.startActivityLocked()

``` java
[-> ActivityStack.java]
    void startActivityLocked(ActivityRecord r, ActivityRecord focusedTopActivity,
            boolean newTask, boolean keepCurTransition, ActivityOptions options) {
        TaskRecord rTask = r.getTask();
        final int taskId = rTask.taskId;

        if (!r.mLaunchTaskBehind && (taskForIdLocked(taskId) == null || newTask)) {
            insertTaskAtTop(rTask, r);
        }
        TaskRecord task = null;
        ......
        task = activityTask;        task.setFrontOfTask();

        if (mActivityTrigger != null) {
            mActivityTrigger.activityStartTrigger(r.intent, r.info, r.appInfo, r.fullscreen);
        }

        if (mActivityPluginDelegate != null && getWindowingMode() != WINDOWING_MODE_UNDEFINED) {
            mActivityPluginDelegate.activityInvokeNotification
                (r.appInfo.packageName, getWindowingMode() == WINDOWING_MODE_FULLSCREEN);
        }

        if (!isActivityTypeHome() || numActivities() > 0) {
            if ((r.intent.getFlags() & Intent.FLAG_ACTIVITY_NO_ANIMATION) != 0) {
                mWindowManager.prepareAppTransition(TRANSIT_NONE, keepCurTransition);
                mStackSupervisor.mNoAnimActivities.add(r);
            } else {
                int transit = TRANSIT_ACTIVITY_OPEN;
                if (newTask) {
                    if (r.mLaunchTaskBehind) {
                        transit = TRANSIT_TASK_OPEN_BEHIND;
                    } else {
                        ......
                        transit = TRANSIT_TASK_OPEN;
                    }
                }
                mWindowManager.prepareAppTransition(transit, keepCurTransition);
                mStackSupervisor.mNoAnimActivities.remove(r);
            }
            boolean doShow = true;
            if (newTask) {

                if ((r.intent.getFlags() & Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) != 0) {
                    resetTaskIfNeededLocked(r, r);
                    doShow = topRunningNonDelayedActivityLocked(null) == r;
                }
            } else if (options != null && options.getAnimationType()
                    == ActivityOptions.ANIM_SCENE_TRANSITION) {
                doShow = false;
            }
            if (r.mLaunchTaskBehind) {

                r.setVisibility(true);
                ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
            } else if (SHOW_APP_STARTING_PREVIEW && doShow) {
                TaskRecord prevTask = r.getTask();
                ActivityRecord prev = prevTask.topRunningActivityWithStartingWindowLocked();
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

##### 3.5、ActivityStackSupervisor.resumeFocusedStackTopActivityLocked()

``` java
    [->ActivityStackSupervisor.java]
    boolean resumeFocusedStackTopActivityLocked(
            ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {

        ......

        if (targetStack != null && isFocusedStack(targetStack)) {
            return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
        }

        final ActivityRecord r = mFocusedStack.topRunningActivityLocked();
        if (r == null || !r.isState(RESUMED)) {
            mFocusedStack.resumeTopActivityUncheckedLocked(null, null);
        } else if (r.isState(RESUMED)) {
            // Kick off any lingering app transitions form the MoveTaskToFront operation.
            mFocusedStack.executeAppTransition(targetOptions);
        }

        return false;
    }

```

##### 3.6、ActivityStack.resumeTopActivityUncheckedLocked()

 inResumeTopActivity用于保证每次只有一个Activity执行resumeTopActivityLocked()操作.

``` java
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
            result = resumeTopActivityInnerLocked(prev, options);

            ......
            final ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);
            .......
        } finally {
            mStackSupervisor.inResumeTopActivity = false;
        }

        return result;
    }

```

##### 3.7、ActivityStack.resumeTopActivityInnerLocked()

说明：启动一个新Activity时，如果界面还存在其它的Activity，那么必须先中断其它的Activity。 
因此，除了第一个启动的Home界面对应的Activity外，其它的Activity均需要进行此操作，当系统启动第一个Activity，即Home时，mResumedActivity的值才会为null。 
经过一系列处理逻辑之后最终调用了startPausingLocked方法，这个方法作用就是让系统中栈中的Activity执行onPause方法。


``` java
    [->ActivityStack.java]
   @GuardedBy("mService")
    private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
        ......
        final ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);

        boolean pausing = mStackSupervisor.pauseBackStacks(userLeaving, next, false);
        ......

        ActivityStack lastStack = mStackSupervisor.getLastStack();
        if (next.app != null && next.app.thread != null) {
            ......
        } else {
            // Whoops, need to restart this activity!
            if (!next.hasBeenLaunched) {
                next.hasBeenLaunched = true;
            } else {
                if (SHOW_APP_STARTING_PREVIEW) {
                    next.showStartingWindow(null /* prev */, false /* newTask */,
                            false /* taskSwich */);
                }
                if (DEBUG_SWITCH) Slog.v(TAG_SWITCH, "Restarting: " + next);
            }
            if (DEBUG_STATES) Slog.d(TAG_STATES, "resumeTopActivityLocked: Restarting " + next);
            mStackSupervisor.startSpecificActivityLocked(next, true, true);
        }

        if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
        return true;
    }
```

##### 3.8、ActivityStackSupervisor.pauseBackStacks()
暂停所有处于后台栈的所有Activity。

``` java
    [->ActivityStackSupervisor.java]
    boolean pauseBackStacks(boolean userLeaving, ActivityRecord resuming, boolean dontWait) {
        boolean someActivityPaused = false;
        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            final ActivityDisplay display = mActivityDisplays.valueAt(displayNdx);
            for (int stackNdx = display.getChildCount() - 1; stackNdx >= 0; --stackNdx) {
                final ActivityStack stack = display.getChildAt(stackNdx);
                if (!isFocusedStack(stack) && stack.getResumedActivity() != null) {
                    if (DEBUG_STATES) Slog.d(TAG_STATES, "pauseBackStacks: stack=" + stack +
                            " mResumedActivity=" + stack.getResumedActivity());
                    someActivityPaused |= stack.startPausingLocked(userLeaving, false, resuming,
                            dontWait);
                }
            }
        }
        return someActivityPaused;
    }

```

#### 四、执行栈顶Activity的onPause方法


##### 4.1、ActivityStack.startPausingLocked()

``` java
    [->ActivityStack.java]
    final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping,
            ActivityRecord resuming, boolean pauseImmediately) {
        if (mPausingActivity != null) {
            Slog.wtf(TAG, "Going to pause when pause is already pending for " + mPausingActivity
                    + " state=" + mPausingActivity.getState());
            if (!shouldSleepActivities()) {
                // Avoid recursion among check for sleep and complete pause during sleeping.
                // Because activity will be paused immediately after resume, just let pause
                // be completed by the order of activity paused from clients.
                completePauseLocked(false, resuming);
            }
        }
        ActivityRecord prev = mResumedActivity;

        if (prev == null) {
            if (resuming == null) {
                Slog.wtf(TAG, "Trying to pause when nothing is resumed");
                mStackSupervisor.resumeFocusedStackTopActivityLocked();
            }
            return false;
        }

        if (prev == resuming) {
            Slog.wtf(TAG, "Trying to pause activity that is in process of being resumed");
            return false;
        }

        if (DEBUG_STATES) Slog.v(TAG_STATES, "Moving to PAUSING: " + prev);
        else if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Start pausing: " + prev);

        if (mActivityTrigger != null) {
            mActivityTrigger.activityPauseTrigger(prev.intent, prev.info, prev.appInfo);
        }

        if (mActivityPluginDelegate != null && getWindowingMode() != WINDOWING_MODE_UNDEFINED) {
            mActivityPluginDelegate.activitySuspendNotification
                (prev.appInfo.packageName, getWindowingMode() == WINDOWING_MODE_FULLSCREEN, true);
        }

        mResumedActivity = null;
        mPausingActivity = prev;
        mLastPausedActivity = prev;
        mLastNoHistoryActivity = (prev.intent.getFlags() & Intent.FLAG_ACTIVITY_NO_HISTORY) != 0
                || (prev.info.flags & ActivityInfo.FLAG_NO_HISTORY) != 0 ? prev : null;
        prev.setState(PAUSING, "startPausingLocked");
        prev.getTask().touchActiveTime();
        clearLaunchTime(prev);

        mStackSupervisor.getLaunchTimeTracker().stopFullyDrawnTraceIfNeeded(getWindowingMode());

        mService.updateCpuStats();

        if (prev.app != null && prev.app.thread != null) {
            if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Enqueueing pending pause: " + prev);
            try {
                EventLogTags.writeAmPauseActivity(prev.userId, System.identityHashCode(prev),
                        prev.shortComponentName, "userLeaving=" + userLeaving);
                mService.updateUsageStats(prev, false);

                mService.getLifecycleManager().scheduleTransaction(prev.app.thread, prev.appToken,
                        PauseActivityItem.obtain(prev.finishing, userLeaving,
                                prev.configChangeFlags, pauseImmediately));
            } catch (Exception e) {
                // Ignore exception, if process died other code will cleanup.
                Slog.w(TAG, "Exception thrown during pause", e);
                mPausingActivity = null;
                mLastPausedActivity = null;
                mLastNoHistoryActivity = null;
            }
        } else {
            mPausingActivity = null;
            mLastPausedActivity = null;
            mLastNoHistoryActivity = null;
        }

        // If we are not going to sleep, we want to ensure the device is
        // awake until the next activity is started.
        if (!uiSleeping && !mService.isSleepingOrShuttingDownLocked()) {
            mStackSupervisor.acquireLaunchWakelock();
        }

        if (mPausingActivity != null) {
            // Have the window manager pause its key dispatching until the new
            // activity has started.  If we're pausing the activity just because
            // the screen is being turned off and the UI is sleeping, don't interrupt
            // key dispatch; the same activity will pick it up again on wakeup.
            if (!uiSleeping) {
                prev.pauseKeyDispatchingLocked();
            } else if (DEBUG_PAUSE) {
                 Slog.v(TAG_PAUSE, "Key dispatch not paused for screen off");
            }

            if (pauseImmediately) {
                // If the caller said they don't want to wait for the pause, then complete
                // the pause now.
                completePauseLocked(false, resuming);
                return false;

            } else {
                schedulePauseTimeout(prev);
                return true;
            }

        } else {
            // This activity failed to schedule the
            // pause, so just treat it as being paused now.
            if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Activity not running, resuming next.");
            if (resuming == null) {
                mStackSupervisor.resumeFocusedStackTopActivityLocked();
            }
            return false;
        }
    }



/frameworks/base/services/core/java/com/android/server/am/ClientLifecycleManager.java
    void scheduleTransaction(@NonNull IApplicationThread client, @NonNull IBinder activityToken,
            @NonNull ClientTransactionItem callback) throws RemoteException {
        final ClientTransaction clientTransaction = transactionWithCallback(client, activityToken,
                callback);
        scheduleTransaction(clientTransaction);
    }

    void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
        final IApplicationThread client = transaction.getClient();
        transaction.schedule();
        if (!(client instanceof Binder)) {
            // If client is not an instance of Binder - it's a remote call and at this point it is
            // safe to recycle the object. All objects used for local calls will be recycled after
            // the transaction is executed on client in ActivityThread.
            transaction.recycle();
        }
    }
    
G:\android9.0\frameworks\base\core\java\android\app\servertransaction\ClientTransaction.java
        /** Target client. */
    private IApplicationThread mClient;
    public void schedule() throws RemoteException {
        mClient.scheduleTransaction(this);
    }
```
那么这个mClient是什么呢,根据code可以看到private IApplicationThread mClient;其实mClient就是右上一步  clientTransaction.addCallback(arg...);传入的IApplication的实例,前面也说过,这个IApplication实例是一个binder对象,用于System_service与app端进行交互;

``` cpp
    private class ApplicationThread extends IApplicationThread.Stub {
        ......
        @Override
        public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
            ActivityThread.this.scheduleTransaction(transaction);
        }
        ......
}
```

AMS--通过ApplicationThread:scheduleTransaction回调至app,注意ApplicationThread 就是一个binder
到此为止AMS的回调完成回到app的进程中;

``` cpp
android\out\target\common\obj\JAVA_LIBRARIES\framework_intermediates\core\java\android\app\IApplicationThread.java
```

继续看ActivityThread.this.scheduleTransaction(transaction);源码:
//ClientTransactionHandler
注意一下,这里由于ActivityThread中没有重写scheduleTransaction,所以会走到ActivityThread的父类ClientTransactionHandler的scheduleTransaction

``` cpp
 void scheduleTransaction(ClientTransaction transaction) {
        transaction.preExecute(this);
        // 通过handler 发消息通知ActivityThread
        sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
    }

      //ActivityThread
     void sendMessage(int what, Object obj) {
        sendMessage(what, obj, 0, 0, false);
    }
```
scheduleTransaction然后走到ActivityThread的sendMessage,进而会掉到ActivityThread的H类

H类源码:
//ActivityThread

``` cpp
 class H extends Handler {
        ......
        public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
            switch (msg.what) {
                case BIND_APPLICATION:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
                    AppBindData data = (AppBindData)msg.obj;
                    handleBindApplication(data);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
                case EXIT_APPLICATION:
                    if (mInitialApplication != null) {
                        mInitialApplication.onTerminate();
                    }
                    Looper.myLooper().quit();
                    break;
                ......
                case EXECUTE_TRANSACTION:
                    final ClientTransaction transaction = (ClientTransaction) msg.obj;
                    // 注释1
                    mTransactionExecutor.execute(transaction);
                    if (isSystem()) {
                        // Client transactions inside system process are recycled on the client side
                        // instead of ClientLifecycleManager to avoid being cleared before this
                        // message is handled.
                        transaction.recycle();
                    }
                    // TODO(lifecycler): Recycle locally scheduled transactions.
                    break;
                case RELAUNCH_ACTIVITY:
                    handleRelaunchActivityLocally((IBinder) msg.obj);
                    break;
            }
            Object obj = msg.obj;
            if (obj instanceof SomeArgs) {
                ((SomeArgs) obj).recycle();
            }
            if (DEBUG_MESSAGES) Slog.v(TAG, "<<< done: " + codeToString(msg.what));
        }
    }
```
毫无疑问,四大组件的生命周期控制的AMS回调入口的实现逻辑就在这里了,ok,我们继续回到activity的创建逻辑线,继续分析;
代码太多,所以从上面摘出一部分分析;

``` cpp
case EXECUTE_TRANSACTION:
                    final ClientTransaction transaction = (ClientTransaction) msg.obj;
                    // 注释1
                    mTransactionExecutor.execute(transaction);
                    if (isSystem()) {
                        // Client transactions inside system process are recycled on the client side
                        // instead of ClientLifecycleManager to avoid being cleared before this
                        // message is handled.
                        transaction.recycle();
                    }
                    // TODO(lifecycler): Recycle locally scheduled transactions.
                    break;
```

继续看注释1mTransactionExecutor.execute(transaction);源码:
//TransactionExecutor

``` cpp
 public void execute(ClientTransaction transaction) {
        final IBinder token = transaction.getActivityToken();
        log("Start resolving transaction for client: " + mTransactionHandler + ", token: " + token);

        executeCallbacks(transaction);

        executeLifecycleState(transaction);
        mPendingActions.clear();
        log("End resolving transaction");
    }


/** Transition to the final state if requested by the transaction. */
    private void executeLifecycleState(ClientTransaction transaction) {
        final ActivityLifecycleItem lifecycleItem = transaction.getLifecycleStateRequest();
        if (lifecycleItem == null) {
            // No lifecycle request, return early.
            return;
        }
        log("Resolving lifecycle state: " + lifecycleItem);

        final IBinder token = transaction.getActivityToken();
        final ActivityClientRecord r = mTransactionHandler.getActivityClient(token);

        if (r == null) {
            // Ignore requests for non-existent client records for now.
            return;
        }

        // Cycle to the state right before the final requested state.
        cycleToPath(r, lifecycleItem.getTargetState(), true /* excludeLastState */);

        // Execute the final transition with proper parameters.
        lifecycleItem.execute(mTransactionHandler, token, mPendingActions);
        lifecycleItem.postExecute(mTransactionHandler, token, mPendingActions);
    }
```
execute(ClientTransaction transaction)中调用 executeLifecycleState(ClientTransaction transaction),然后调用lifecycleItem.execute(mTransactionHandler, token, mPendingActions);
这个PauseActivityItem是在System_service时候,传入的,下面摘入一段前面的代码:

``` cpp
                mService.getLifecycleManager().scheduleTransaction(prev.app.thread, prev.appToken,
                        PauseActivityItem.obtain(prev.finishing, userLeaving,
                                prev.configChangeFlags, pauseImmediately));
```

这里的clientTransaction.addCallback(PauseActivityItem.obtain(arg...))就是我们目前用到的lifecycleItem,也就是说 这个lifecycleItem就是PauseActivityItem

继续看lifecycleItem.execute(mTransactionHandler, token, mPendingActions);源码:

//PauseActivityItem
##### 4.2、PauseActivityItem.execute()

``` java
G:\android9.0\frameworks\base\core\java\android\app\servertransaction\PauseActivityItem.java

    @Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityPause");
        client.handlePauseActivity(token, mFinished, mUserLeaving, mConfigChanges, pendingActions,
                "PAUSE_ACTIVITY_ITEM");
        Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
    }

```

##### 4.3、[ActivityThread.handleMessage() : mH]

``` java
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
##### 4.4、ActivityThread.handlePauseActivity() 

``` java
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
##### 4.5、ActivityThread.performPauseActivity() 

``` java
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
##### 4.6、Instrumentation.callActivityOnPuase() 

``` java
    [->Instrumentation.java]
public void callActivityOnPause(Activity activity) {
    activity.performPause();
}
```

原来最终回调到了Activity的performPause方法：
##### 4.7、Activity.performPause() 

``` java
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

终于，太不容易了，回调到了Activity的onPause方法，哈哈，Activity生命周期中的第一个生命周期方法终于被我们找到了。。。。也就是说我们在启动一个Activity的时候最先被执行的是栈顶的Activity的onPause方法。

> ActivityStackSupervisor.resumeFocusedStackTopActivityLocked()–> 
> ActivityStack.resumeTopActivityUncheckedLocked() –> 
> ActivityStack.resumeTopActivityInnerLocked() –> 
> ActivityStackSupervisor.startSpecificActivityLocked()
继续分析resumeTopActivityInnerLocked


好吧，我们看一下startSpecificActivityLocked的具体实现：

``` java
    [->ActivityStack.java]
   void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {
        // Is this activity's application already running?
        ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                r.info.applicationInfo.uid, true);

        getLaunchTimeTracker().setLaunchTime(r);

        if (app != null && app.thread != null) {
            try {
                if ((r.info.flags&ActivityInfo.FLAG_MULTIPROCESS) == 0
                        || !"android".equals(r.info.packageName)) {
                    // Don't add this if it is a platform component that is marked
                    // to run in multiple processes, because this is actually
                    // part of the framework so doesn't make sense to track as a
                    // separate apk in the process.
                    app.addPackage(r.info.packageName, r.info.applicationInfo.longVersionCode,
                            mService.mProcessStats);
                }
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

可以发现在这个方法中，首先会判断一下需要启动的Activity所需要的应用进程是否已经启动，若启动的话，则直接调用realStartAtivityLocked方法，否则调用startProcessLocked方法，用于启动应用进程。 
##### 4.8、Activity所在的应用进程启动过程
接下来分析一下应用进程的启动过程，上面分析到ActivityStackSupervisor.startSpecificActivityLocked方法，在这个方法中会去根据进程和线程是否存在判断App是否已经启动，如果已经启动，就会调用realStartActivityLocked方法继续处理。如果没有启动则调用ActivityManagerService.startProcessLocked方法创建新的进程处理。接下来跟踪一下一个新的Activity是如何一步步启动的。

``` cpp
    frameworks/base/services/core/java/com/android/server/am/ActivityStackSupervisor.java
    void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {
        // Is this activity's application already running?
        ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                r.info.applicationInfo.uid, true);
            ...
            if (app != null && app.thread != null) {
                ...
                realStartActivityLocked(r, app, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting activity "
                        + r.intent.getComponent().flattenToShortString(), e);
            }
        }

        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                "activity", r.intent.getComponent(), false, false, true);
    }

```
ActivityManagerService.startProcessLocked方法经过多次跳转最终会通过Process.start方法来为应用创建进程。

``` cpp
    frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
    private ProcessStartResult startProcess(String hostingType, String entryPoint,
            ProcessRecord app, int uid, int[] gids, int runtimeFlags, int mountExternal,
            String seInfo, String requiredAbi, String instructionSet, String invokeWith,
            long startTime) {
        ...
        startResult = Process.start(entryPoint,
                        app.processName, uid, uid, gids, runtimeFlags, mountExternal,
                        app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                        app.info.dataDir, invokeWith,
                        new String[] {PROC_START_SEQ_IDENT + app.startSeq});
        ...
    }

    frameworks/base/core/java/android/os/Process.java
    public static final ProcessStartResult start(final String processClass,
                                  final String niceName,
                                  int uid, int gid, int[] gids,
                                  int runtimeFlags, int mountExternal,
                                  int targetSdkVersion,
                                  String seInfo,
                                  String abi,
                                  String instructionSet,
                                  String appDataDir,
                                  String invokeWith,
                                  String[] zygoteArgs) {
        return zygoteProcess.start(processClass, niceName, uid, gid, gids,
                    runtimeFlags, mountExternal, targetSdkVersion, seInfo,
                    abi, instructionSet, appDataDir, invokeWith, zygoteArgs);
    }

    frameworks/base/core/java/android/os/ZygoteProcess.java
    public final Process.ProcessStartResult start(final String processClass,
                                                  final String niceName,
                                                  int uid, int gid, int[] gids,
                                                  int runtimeFlags, int mountExternal,
                                                  int targetSdkVersion,
                                                  String seInfo,
                                                  String abi,
                                                  String instructionSet,
                                                  String appDataDir,
                                                  String invokeWith,
                                                  String[] zygoteArgs) {
        try {
            return startViaZygote(processClass, niceName, uid, gid, gids,
                    runtimeFlags, mountExternal, targetSdkVersion, seInfo,
                    abi, instructionSet, appDataDir, invokeWith, false /* startChildZygote */,
                    zygoteArgs);
        } catch (ZygoteStartFailedEx ex) {
            Log.e(LOG_TAG,
                    "Starting VM process through Zygote failed");
            throw new RuntimeException(
                    "Starting VM process through Zygote failed", ex);
        }
    }

    private Process.ProcessStartResult startViaZygote(final String processClass,
                                                      final String niceName,
                                                      final int uid, final int gid,
                                                      final int[] gids,
                                                      int runtimeFlags, int mountExternal,
                                                      int targetSdkVersion,
                                                      String seInfo,
                                                      String abi,
                                                      String instructionSet,
                                                      String appDataDir,
                                                      String invokeWith,
                                                      boolean startChildZygote,
                                                      String[] extraArgs)
                                                      throws ZygoteStartFailedEx {
        ...
        synchronized(mLock) {
            return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
        }
    }

    private ZygoteState openZygoteSocketIfNeeded(String abi) throws ZygoteStartFailedEx {
        Preconditions.checkState(Thread.holdsLock(mLock), "ZygoteProcess lock not held");

        if (primaryZygoteState == null || primaryZygoteState.isClosed()) {
            try {
                primaryZygoteState = ZygoteState.connect(mSocket);
            } catch (IOException ioe) {
                throw new ZygoteStartFailedEx("Error connecting to primary zygote", ioe);
            }
            maybeSetApiBlacklistExemptions(primaryZygoteState, false);
            maybeSetHiddenApiAccessLogSampleRate(primaryZygoteState);
        }
        if (primaryZygoteState.matches(abi)) {
            return primaryZygoteState;
        }

        // The primary zygote didn't match. Try the secondary.
        if (secondaryZygoteState == null || secondaryZygoteState.isClosed()) {
            try {
                secondaryZygoteState = ZygoteState.connect(mSecondarySocket);
            } catch (IOException ioe) {
                throw new ZygoteStartFailedEx("Error connecting to secondary zygote", ioe);
            }
            maybeSetApiBlacklistExemptions(secondaryZygoteState, false);
            maybeSetHiddenApiAccessLogSampleRate(secondaryZygoteState);
        }

        if (secondaryZygoteState.matches(abi)) {
            return secondaryZygoteState;
        }

        throw new ZygoteStartFailedEx("Unsupported zygote ABI: " + abi);
    }

```
经过一步步调用，可以发现其最终调用了Zygote并通过socket通信的方式让Zygote进程fork出一个新的进程，并根据传递的”android.app.ActivityThread”字符串，反射出该对象并执行ActivityThread的main方法对其进行初始化。

##### 4.9、Activity所在应用主线程初始化
在ActivityThread.main方法中对ActivityThread进行了初始化，创建了主线程的Looper对象并调用Looper.loop()方法启动Looper，把自定义Handler类H的对象作为主线程的handler。接下来跳转到ActivityThread.attach方法，看都做了什么。

``` cpp
   frameworks/base/core/java/android/app/ActivityThread.java
    public static void main(String[] args) {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
        ...
        Looper.prepareMainLooper();

        // Find the value for {@link #PROC_START_SEQ_IDENT} if provided on the command line.
        // It will be in the format "seq=114"
        long startSeq = 0;
        if (args != null) {
            for (int i = args.length - 1; i >= 0; --i) {
                if (args[i] != null && args[i].startsWith(PROC_START_SEQ_IDENT)) {
                    startSeq = Long.parseLong(
                            args[i].substring(PROC_START_SEQ_IDENT.length()));
                }
            }
        }
        ActivityThread thread = new ActivityThread();
        thread.attach(false, startSeq);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }
        ...
        Looper.loop();
        ...
    }

```
Activity所在的进程创建完了，主线程也初始化了，接下来就该真正的启动Activity了。在ActivityThread.attach方法中，首先会通过ActivityManagerService为这个应用绑定一个Application，然后添加一个垃圾回收观察者，每当系统触发垃圾回收的时候就会在run方法里面去计算应用使用了多少内存，如果超过总量的四分之三就会尝试释放内存。最后，为根View添加config回调接收config变化相关的信息。

``` java
    frameworks/base/core/java/android/app/ActivityThread.java
    private void attach(boolean system, long startSeq) {
        sCurrentActivityThread = this;
        mSystemThread = system;
        if (!system) {
            ...
            final IActivityManager mgr = ActivityManager.getService();
            try {
                mgr.attachApplication(mAppThread, startSeq);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
            // Watch for getting close to heap limit.
            BinderInternal.addGcWatcher(new Runnable() {
                @Override public void run() {
                    if (!mSomeActivitiesChanged) {
                        return;
                    }
                    Runtime runtime = Runtime.getRuntime();
                    long dalvikMax = runtime.maxMemory();
                    long dalvikUsed = runtime.totalMemory() - runtime.freeMemory();
                    if (dalvikUsed > ((3*dalvikMax)/4)) {
                        mSomeActivitiesChanged = false;
                        try {
                            mgr.releaseSomeActivities(mAppThread);
                        } catch (RemoteException e) {
                            throw e.rethrowFromSystemServer();
                        }
                    }
                }
            });
        }
        ...
        ViewRootImpl.addConfigCallback(configChangedCallback);
    }

    frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
    @Override
    public final void attachApplication(IApplicationThread thread, long startSeq) {
        synchronized (this) {
            int callingPid = Binder.getCallingPid();
            final int callingUid = Binder.getCallingUid();
            final long origId = Binder.clearCallingIdentity();
            attachApplicationLocked(thread, callingPid, callingUid, startSeq);
            Binder.restoreCallingIdentity(origId);
        }
    }

    private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid, int callingUid, long startSeq) {
        ...
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
        ...
    }

    frameworks/base/services/core/java/com/android/server/am/ActivityStackSupervisor.java
    boolean attachApplicationLocked(ProcessRecord app) throws RemoteException {
        final String processName = app.processName;
        boolean didSomething = false;
        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            final ActivityDisplay display = mActivityDisplays.valueAt(displayNdx);
            for (int stackNdx = display.getChildCount() - 1; stackNdx >= 0; --stackNdx) {
                final ActivityStack stack = display.getChildAt(stackNdx);
                if (!isFocusedStack(stack)) {
                    continue;
                }
                stack.getAllRunningVisibleActivitiesLocked(mTmpActivityList);
                final ActivityRecord top = stack.topRunningActivityLocked();
                final int size = mTmpActivityList.size();
                for (int i = 0; i < size; i++) {
                    final ActivityRecord activity = mTmpActivityList.get(i);
                    if (activity.app == null && app.uid == activity.info.applicationInfo.uid
                            && processName.equals(activity.processName)) {
                        try {
                            if (realStartActivityLocked(activity, app,
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
            ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
        }
        return didSomething;
    }

```
在ActivityManagerService.attachApplication方法中经过多次跳转执行到ActivityStackSupervisor.realStartActivityLocked方法。看到这个方法有没有一点眼熟？没错，就是之前分析过程中遇到的如果应用进程已经启动的情况下去启动Activity所调用的方法，有点绕，自己体会一下。在ActivityStackSupervisor.realStartActivityLocked方法中为ClientTransaction对象添加LaunchActivityItem的callback，然后设置当前的生命周期状态，最后调用ClientLifecycleManager.scheduleTransaction方法执行。

#### 五、执行启动Acitivity

##### 5.1、ActivityStackSupervisor.realStartActivityLocked() 

``` java
    [->ActivityStackSupervisor.java]
    final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
            boolean andResume, boolean checkConfig) throws RemoteException {

        if (!allPausedActivitiesComplete()) {
            // While there are activities pausing we skipping starting any new activities until
            // pauses are complete. NOTE: that we also do this for activities that are starting in
            // the paused state because they will first be resumed then paused on the client side.
            if (DEBUG_SWITCH || DEBUG_PAUSE || DEBUG_STATES) Slog.v(TAG_PAUSE,
                    "realStartActivityLocked: Skipping start of r=" + r
                    + " some activities pausing...");
            return false;
        }

        final TaskRecord task = r.getTask();
        final ActivityStack stack = task.getStack();

        beginDeferResume();

        try {
            r.startFreezingScreenLocked(app, 0);

            // schedule launch ticks to collect information about slow apps.
            r.startLaunchTickingLocked();

            r.setProcess(app);

            if (getKeyguardController().isKeyguardLocked()) {
                r.notifyUnknownVisibilityLaunched();
            }

            // Have the window manager re-evaluate the orientation of the screen based on the new
            // activity order.  Note that as a result of this, it can call back into the activity
            // manager with a new orientation.  We don't care about that, because the activity is
            // not currently running so we are just restarting it anyway.
            if (checkConfig) {
                // Deferring resume here because we're going to launch new activity shortly.
                // We don't want to perform a redundant launch of the same record while ensuring
                // configurations and trying to resume top activity of focused stack.
                ensureVisibilityAndConfig(r, r.getDisplayId(),
                        false /* markFrozenIfConfigChanged */, true /* deferResume */);
            }

            if (r.getStack().checkKeyguardVisibility(r, true /* shouldBeVisible */,
                    true /* isTop */)) {
                // We only set the visibility to true if the activity is allowed to be visible
                // based on
                // keyguard state. This avoids setting this into motion in window manager that is
                // later cancelled due to later calls to ensure visible activities that set
                // visibility back to false.
                r.setVisibility(true);
            }

            final int applicationInfoUid =
                    (r.info.applicationInfo != null) ? r.info.applicationInfo.uid : -1;
            if ((r.userId != app.userId) || (r.appInfo.uid != applicationInfoUid)) {
                Slog.wtf(TAG,
                        "User ID for activity changing for " + r
                                + " appInfo.uid=" + r.appInfo.uid
                                + " info.ai.uid=" + applicationInfoUid
                                + " old=" + r.app + " new=" + app);
            }

            app.waitingToKill = null;
            r.launchCount++;
            r.lastLaunchTime = SystemClock.uptimeMillis();

            if (DEBUG_ALL) Slog.v(TAG, "Launching: " + r);

            int idx = app.activities.indexOf(r);
            if (idx < 0) {
                app.activities.add(r);
            }
            mService.updateLruProcessLocked(app, true, null);
            mService.updateOomAdjLocked();

            final LockTaskController lockTaskController = mService.getLockTaskController();
            if (task.mLockTaskAuth == LOCK_TASK_AUTH_LAUNCHABLE
                    || task.mLockTaskAuth == LOCK_TASK_AUTH_LAUNCHABLE_PRIV
                    || (task.mLockTaskAuth == LOCK_TASK_AUTH_WHITELISTED
                            && lockTaskController.getLockTaskModeState()
                                    == LOCK_TASK_MODE_LOCKED)) {
                lockTaskController.startLockTaskMode(task, false, 0 /* blank UID */);
            }

            try {
                if (app.thread == null) {
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
                if (DEBUG_SWITCH) Slog.v(TAG_SWITCH,
                        "Launching: " + r + " icicle=" + r.icicle + " with results=" + results
                                + " newIntents=" + newIntents + " andResume=" + andResume);
                EventLog.writeEvent(EventLogTags.AM_RESTART_ACTIVITY, r.userId,
                        System.identityHashCode(r), task.taskId, r.shortComponentName);
                if (r.isActivityTypeHome()) {
                    // Home process is the root process of the task.
                    mService.mHomeProcess = task.mActivities.get(0).app;
                }
                mService.notifyPackageUse(r.intent.getComponent().getPackageName(),
                        PackageManager.NOTIFY_PACKAGE_USE_ACTIVITY);
                r.sleeping = false;
                r.forceNewConfig = false;
                mService.getAppWarningsLocked().onStartActivity(r);
                mService.showAskCompatModeDialogLocked(r);
                r.compat = mService.compatibilityInfoForPackageLocked(r.info.applicationInfo);
                ProfilerInfo profilerInfo = null;
                if (mService.mProfileApp != null && mService.mProfileApp.equals(app.processName)) {
                    if (mService.mProfileProc == null || mService.mProfileProc == app) {
                        mService.mProfileProc = app;
                        ProfilerInfo profilerInfoSvc = mService.mProfilerInfo;
                        if (profilerInfoSvc != null && profilerInfoSvc.profileFile != null) {
                            if (profilerInfoSvc.profileFd != null) {
                                try {
                                    profilerInfoSvc.profileFd = profilerInfoSvc.profileFd.dup();
                                } catch (IOException e) {
                                    profilerInfoSvc.closeFd();
                                }
                            }

                            profilerInfo = new ProfilerInfo(profilerInfoSvc);
                        }
                    }
                }

                app.hasShownUi = true;
                app.pendingUiClean = true;
                app.forceProcessStateUpTo(mService.mTopProcessState);
                // Because we could be starting an Activity in the system process this may not go
                // across a Binder interface which would create a new Configuration. Consequently
                // we have to always create a new Configuration here.

                final MergedConfiguration mergedConfiguration = new MergedConfiguration(
                        mService.getGlobalConfiguration(), r.getMergedOverrideConfiguration());
                r.setLastReportedConfiguration(mergedConfiguration);

                logIfTransactionTooLarge(r.intent, r.icicle);


                // Create activity launch transaction.
                final ClientTransaction clientTransaction = ClientTransaction.obtain(app.thread,
                        r.appToken);
                clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                        System.identityHashCode(r), r.info,
                        // TODO: Have this take the merged configuration instead of separate global
                        // and override configs.
                        mergedConfiguration.getGlobalConfiguration(),
                        mergedConfiguration.getOverrideConfiguration(), r.compat,
                        r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
                        r.persistentState, results, newIntents, mService.isNextTransitionForward(),
                        profilerInfo));

                // Set desired final state.
                final ActivityLifecycleItem lifecycleItem;
                if (andResume) {
                    lifecycleItem = ResumeActivityItem.obtain(mService.isNextTransitionForward());
                } else {
                    lifecycleItem = PauseActivityItem.obtain();
                }
                clientTransaction.setLifecycleStateRequest(lifecycleItem);

                // Schedule transaction.
                mService.getLifecycleManager().scheduleTransaction(clientTransaction);


                if ((app.info.privateFlags & ApplicationInfo.PRIVATE_FLAG_CANT_SAVE_STATE) != 0
                        && mService.mHasHeavyWeightFeature) {
                    // This may be a heavy-weight process!  Note that the package
                    // manager will ensure that only activity can run in the main
                    // process of the .apk, which is the only thing that will be
                    // considered heavy-weight.
                    if (app.processName.equals(app.info.packageName)) {
                        if (mService.mHeavyWeightProcess != null
                                && mService.mHeavyWeightProcess != app) {
                            Slog.w(TAG, "Starting new heavy weight process " + app
                                    + " when already running "
                                    + mService.mHeavyWeightProcess);
                        }
                        mService.mHeavyWeightProcess = app;
                        Message msg = mService.mHandler.obtainMessage(
                                ActivityManagerService.POST_HEAVY_NOTIFICATION_MSG);
                        msg.obj = r;
                        mService.mHandler.sendMessage(msg);
                    }
                }

            } catch (RemoteException e) {
                if (r.launchFailed) {
                    // This is the second time we failed -- finish activity
                    // and give up.
                    Slog.e(TAG, "Second failure launching "
                            + r.intent.getComponent().flattenToShortString()
                            + ", giving up", e);
                    mService.appDiedLocked(app);
                    stack.requestFinishActivityLocked(r.appToken, Activity.RESULT_CANCELED, null,
                            "2nd-crash", false);
                    return false;
                }

                // This is the first time we failed -- restart process and
                // retry.
                r.launchFailed = true;
                app.activities.remove(r);
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

        // Launch the new version setup screen if needed.  We do this -after-
        // launching the initial activity (that is, home), so that it can have
        // a chance to initialize itself while in the background, making the
        // switch back to it faster and look better.
        if (isFocusedStack(stack)) {
            mService.getActivityStartController().startSetupActivity();
        }

        // Update any services we are bound to that might care about whether
        // their client may have activities.
        if (r.app != null) {
            mService.mServices.updateServiceConnectionActivitiesLocked(r.app);
        }

        return true;
    }
```

``` java
clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
        System.identityHashCode(r), r.info,
        // TODO: Have this take the merged configuration instead of separate global
        // and override configs.
        mergedConfiguration.getGlobalConfiguration(),
        mergedConfiguration.getOverrideConfiguration(), r.compat,
        r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
        r.persistentState, results, newIntents, mService.isNextTransitionForward(),
        profilerInfo));
```

这里的clientTransaction.addCallback(LaunchActivityItem.obtain(arg...))就是我们目前用到的lifecycleItem,也就是说 这个lifecycleItem就是LaunchActivityItem
继续看lifecycleItem.execute(mTransactionHandler, token, mPendingActions);源码:
//LaunchActivityItem

``` cpp
  @Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
        ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
                mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
                mPendingResults, mPendingNewIntents, mIsForward,
                mProfilerInfo, client);
        //注释1
        client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
        Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
    }
```



##### 5.2、ActivityThread.handleLaunchActivity()
client.handleLaunchActivity就是:控制activity的生命周期  真正的实现类是ClientTransactionHandler的子类 Activityhread;

继续看client.handleLaunchActivity源码:

``` java
[-> ActivityThread.java]
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

        // Initialize before creating the activity
        if (!ThreadedRenderer.sRendererDisabled) {
            GraphicsEnvironment.earlyInitEGL();
        }
        WindowManagerGlobal.initialize();

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
                ActivityManager.getService()
                        .finishActivity(r.token, Activity.RESULT_CANCELED, null,
                                Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
        }

        return a;
    }
```


##### 5.3、ActivityThread.performLaunchActivity()
performLaunchActivity(r, customIntent)源码:


``` java
[-> ActivityThread.java]
    /**  Core implementation of activity launch. */
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
                        r.referrer, r.voiceInteractor, window, r.configCallback);

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

            mActivities.put(r.token, r);

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
```
初始化activity的context
利用classloader机制反射获取activity实例
创建app的入口application
执行 activity的attach方法   初始化window/父parent-DecorView/windowManage ,windowManager的子类为 WindowManagerimpl 是view和window的操作类
执行activity的onCreate方法

至此executeCallbacks执行完毕，开始执行executeLifecycleState方法。先执行cycleToPath方法，生命周期状态是从ON_CREATE状态到ON_RESUME状态，中间有一个ON_START状态，所以会执行ActivityThread.handleStartActivity方法。

``` cpp
    frameworks/base/core/java/android/app/servertransaction/TransactionExecutor.java
    private void cycleToPath(ActivityClientRecord r, int finish,
            boolean excludeLastState) {
        ...
        final IntArray path = mHelper.getLifecyclePath(start, finish, excludeLastState);
        performLifecycleSequence(r, path);
    }

    /** Transition the client through previously initialized state sequence. */
    private void performLifecycleSequence(ActivityClientRecord r, IntArray path) {
        final int size = path.size();
        for (int i = 0, state; i < size; i++) {
            state = path.get(i);
            log("Transitioning to state: " + state);
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

``` cpp
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

```
经过上面的多次跳转最终调用到Activity.onResume方法，Activity启动完毕。

##### 5.4、栈顶Activity执行onStop过程
至此Activity的启动流程基本上就分析完了，但是熟悉Activity生命周期的同学不难发现，栈顶Activity执行了onPause方法之后怎么没有看到哪里执行了onStop方法呢？之前在ActivityThread.handleResumeActivity方法里面有这么一行不起眼的代码，当MessageQueue空闲的时候就会执行这个Handler。那什么时候空闲呢？从这行代码所在的位置不难分析出是当前Activity执行完onResume的时候执行。

``` java
    frameworks/base/core/java/android/app/ActivityThread.java
    @Override
    public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
            String reason) {
        ...
        Looper.myQueue().addIdleHandler(new Idler());
        ...
    }

```
接下来看一下Idler都做了什么。当MessageQueue空闲的时候就会回调Idler.queueIdle方法，经过层层调用跳转到ActivityStack.stopActivityLocked方法。

``` java
    private class Idler implements MessageQueue.IdleHandler {
        @Override
        public final boolean queueIdle() {
            ...
            if (a.activity != null && !a.activity.mFinished) {
                try {
                    am.activityIdle(a.token, a.createdConfig, stopProfiling);
                        a.createdConfig = null;
                } catch (RemoteException ex) {
                    throw ex.rethrowFromSystemServer();
                }
            }
            ...
            return false;
        }
    }

    frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
    @Override
    public final void activityIdle(IBinder token, Configuration config, boolean stopProfiling) {
        final long origId = Binder.clearCallingIdentity();
        synchronized (this) {
            ActivityStack stack = ActivityRecord.getStackLocked(token);
            if (stack != null) {
                ActivityRecord r =
                        mStackSupervisor.activityIdleInternalLocked(token, false /* fromTimeout */,
                                false /* processPausingActivities */, config);
                if (stopProfiling) {
                    if ((mProfileProc == r.app) && mProfilerInfo != null) {
                        clearProfilerLocked();
                    }
                }
            }
        }
        Binder.restoreCallingIdentity(origId);
    }

    frameworks/base/services/core/java/com/android/server/am/ActivityStackSupervisor.java
    final ActivityRecord activityIdleInternalLocked(final IBinder token, boolean fromTimeout,
            boolean processPausingActivities, Configuration config) {
        ...
        // Stop any activities that are scheduled to do so but have been
        // waiting for the next one to start.
        for (int i = 0; i < NS; i++) {
            r = stops.get(i);
            final ActivityStack stack = r.getStack();
            if (stack != null) {
                if (r.finishing) {
                    stack.finishCurrentActivityLocked(r, ActivityStack.FINISH_IMMEDIATELY, false,
                            "activityIdleInternalLocked");
                } else {
                    stack.stopActivityLocked(r);
                }
            }
        }
        ...

        return r;
    }

    frameworks/base/services/core/java/com/android/server/am/ActivityStack.java
    final void stopActivityLocked(ActivityRecord r) {
        ...
        mService.getLifecycleManager().scheduleTransaction(r.app.thread, r.appToken,
                StopActivityItem.obtain(r.visible, r.configChangeFlags));
        ...
    }

```
在ActivityStack.stopActivityLocked方法中，又见到了ClientLifecycleManager.scheduleTransaction方法，前面已经分析过多次，会去执行StopActivityItem.execute方法，然后经过多次跳转，最终执行了Activity.onStop方法。至此，栈顶Activity的onStop过程分析完毕。

``` java
    frameworks/base/core/java/android/app/servertransaction/StopActivityItem.java
    @Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        ...
        client.handleStopActivity(token, mShowWindow, mConfigChanges, pendingActions,
                true /* finalStateRequest */, "STOP_ACTIVITY_ITEM");
        ...
    }

    frameworks/base/core/java/android/app/ActivityThread.java
    @Override
    public void handleStopActivity(IBinder token, boolean show, int configChanges,
            PendingTransactionActions pendingActions, boolean finalStateRequest, String reason) {
        ...
        performStopActivityInner(r, stopInfo, show, true /* saveState */, finalStateRequest,
                reason);
        ...
    }

    private void performStopActivityInner(ActivityClientRecord r, StopInfo info, boolean keepShown,
            boolean saveState, boolean finalStateRequest, String reason) {
            ...
            if (!keepShown) {
                callActivityOnStop(r, saveState, reason);
            }
        }
    }

    private void callActivityOnStop(ActivityClientRecord r, boolean saveState, String reason) {
        ...
        try {
            r.activity.performStop(false /*preserveWindow*/, reason);
        } catch (SuperNotCalledException e) {
            throw e;
        } catch (Exception e) {
            if (!mInstrumentation.onException(r.activity, e)) {
                throw new RuntimeException(
                        "Unable to stop activity "
                                + r.intent.getComponent().toShortString()
                                + ": " + e.toString(), e);
            }
        }
        ...
    }

    frameworks/base/core/java/android/app/Activity.java
    final void performStop(boolean preserveWindow, String reason) {
        ...
        mInstrumentation.callActivityOnStop(this);
        ...
    }

    frameworks/base/core/java/android/app/Instrumentation.java
    public void callActivityOnStop(Activity activity) {
        activity.onStop();
    }

```
我们来看一下Activity 的生命周期：

> protected void onCreate(); 
> protected void onStart(); 
> protected void onResume(); 
> protected void onPause(); 
> protected void onStop();
> protected void onDestory();

前面我们分析了onCreate()、onStart() 、onResume()、onPause()、onStop()。 Activity 销毁时的 onDestroy() 回调都与前面的过程大同小异，这里就只列举相应的方法栈，不再继续描述。

#### （六）、Surface创建过程分析

``` java
G:\android9.0\frameworks\base\core\java\android\app\Activity.java

final void performCreate(Bundle icicle, PersistableBundle persistentState) {
    mCanEnterPictureInPicture = true;
    restoreHasCurrentPermissionRequest(icicle);
    if (persistentState != null) {
        onCreate(icicle, persistentState);
    } else {
        onCreate(icicle);
    }
    writeEventLog(LOG_AM_ON_CREATE_CALLED, "performCreate");
    mActivityTransitionState.readState(icicle);

    mVisibleFromClient = !mWindow.getWindowStyle().getBoolean(
            com.android.internal.R.styleable.Window_windowNoDisplay, false);
    mFragments.dispatchActivityCreated();
    mActivityTransitionState.setEnterActivityOptions(this, getActivityOptions());
}
```

``` java
Activity.java

    public void setContentView(@LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }
```

getWindow()函数得到前面创建的窗口对象PhoneWindow，通过PhoneWindow来设置Activity的视图。
[->PhoneWindow.java]

``` java
G:\android9.0\frameworks\base\core\java\com\android\internal\policy\PhoneWindow.java
    @Override
    public void setContentView(View view) {
        setContentView(view, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
    }


    // This is the view in which the window contents are placed. It is either
    // mDecor itself, or a child of mDecor where the contents go.
    ViewGroup mContentParent;
    @Override
    public void setContentView(View view, ViewGroup.LayoutParams params) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            view.setLayoutParams(params);
            final Scene newScene = new Scene(mContentParent, view);
            transitionTo(newScene);
        } else {
            mContentParent.addView(view, params);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
    }
```

Activity.onCreate()会调用setContentView(),整个过程主要是Activity的布局文件或View添加至窗口里，
详细过程不再赘述，详细加载过程请参考：[ Android应用setContentView与LayoutInflater加载解析机制源码分析](http://blog.csdn.net/yanbober/article/details/45970721)
重点概括为：

     ● 创建一个DecorView的对象mDecor，该mDecor对象将作为整个应用窗口的根视图。
     ● 依据Feature等style theme创建不同的窗口修饰布局文件，并且通过findViewById获取Activity布局文件该存放的地方（窗口修饰布局文件中id为content的FrameLayout）。
     ● 将Activity的布局文件添加至id为content的FrameLayout内。
     ● 当setContentView设置显示OK以后会回调Activity的onContentChanged方法。Activity的各种View的findViewById()方法等都可以放到该方法中，系统会帮忙回调。



##### 6.1、ActivityThread.handleResumeActivity()
回到我们刚刚的handleLaunchActivity()方法，在调用完performLaunchActivity()方法之后，其有掉用了handleResumeActivity()法。

performLaunchActivity()方法完成了两件事：

1)        Activity窗口对象的创建，通过attach函数来完成；

2)        Activity视图对象的创建，通过setContentView函数来完成；

这些准备工作完成后，就可以显示该Activity了，应用程序进程通过调用handleResumeActivity函数来启动Activity的显示过程。

 [->ActivityThread.java]

``` java
    @Override
    public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
            String reason) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();
        mSomeActivitiesChanged = true;

        // TODO Push resumeArgs into the activity for consideration
        final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
        if (r == null) {
            // We didn't actually resume the activity, so skipping any follow-up actions.
            return;
        }

        final Activity a = r.activity;

        if (localLOGV) {
            Slog.v(TAG, "Resume " + r + " started activity: " + a.mStartedActivity
                    + ", hideForNow: " + r.hideForNow + ", finished: " + a.mFinished);
        }

        final int forwardBit = isForward
                ? WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION : 0;

        // If the window hasn't yet been added to the window manager,
        // and this guy didn't finish itself or start another activity,
        // then go ahead and add the window.
        boolean willBeVisible = !a.mStartedActivity;
        if (!willBeVisible) {
            try {
                willBeVisible = ActivityManager.getService().willActivityBeVisible(
                        a.getActivityToken());
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        }
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
            if (a.mVisibleFromClient) {
                if (!a.mWindowAdded) {
                    a.mWindowAdded = true;
                    wm.addView(decor, l);
                } else {
                    // The activity will get a callback for this {@link LayoutParams} change
                    // earlier. However, at that time the decor will not be set (this is set
                    // in this method), so no action will be taken. This call ensures the
                    // callback occurs with the decor set.
                    a.onWindowAttributesChanged(l);
                }
            }

            // If the window has already been added, but during resume
            // we started another activity, then don't yet make the
            // window visible.
        } else if (!willBeVisible) {
            if (localLOGV) Slog.v(TAG, "Launch " + r + " mStartedActivity set");
            r.hideForNow = true;
        }

        // Get rid of anything left hanging around.
        cleanUpPendingRemoveWindows(r, false /* force */);

        // The window is now visible if it has been added, we are not
        // simply finishing, and we are not starting another activity.
        if (!r.activity.mFinished && willBeVisible && r.activity.mDecor != null && !r.hideForNow) {
            if (r.newConfig != null) {
                performConfigurationChangedForActivity(r, r.newConfig);
                if (DEBUG_CONFIGURATION) {
                    Slog.v(TAG, "Resuming activity " + r.activityInfo.name + " with newConfig "
                            + r.activity.mCurrentConfig);
                }
                r.newConfig = null;
            }
            if (localLOGV) Slog.v(TAG, "Resuming " + r + " with isForward=" + isForward);
            WindowManager.LayoutParams l = r.window.getAttributes();
            if ((l.softInputMode
                    & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION)
                    != forwardBit) {
                l.softInputMode = (l.softInputMode
                        & (~WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION))
                        | forwardBit;
                if (r.activity.mVisibleFromClient) {
                    ViewManager wm = a.getWindowManager();
                    View decor = r.window.getDecorView();
                    wm.updateViewLayout(decor, l);
                }
            }

            r.activity.mVisibleFromServer = true;
            mNumVisibleActivities++;
            if (r.activity.mVisibleFromClient) {
                r.activity.makeVisible();
            }
        }

        r.nextIdle = mNewActivities;
        mNewActivities = r;
        if (localLOGV) Slog.v(TAG, "Scheduling idle handler for " + r);
        Looper.myQueue().addIdleHandler(new Idler());
    }

```

我们知道，在前面的performLaunchActivity函数中完成Activity的创建后，会将当前当前创建的Activity在应用程序进程端的描述符ActivityClientRecord以键值对的形式保存到ActivityThread的成员变量mActivities中：mActivities.put(r.token, r)，r.token就是Activity的身份证，即是IApplicationToken.Proxy代理对象，也用于与AMS通信。上面的函数首先通过performResumeActivity从mActivities变量中取出Activity的应用程序端描述符ActivityClientRecord，然后取出前面为Activity创建的视图对象DecorView和窗口管理器WindowManager，最后将视图对象添加到窗口管理器中。

ViewManager.addView()真正实现的的地方在WindowManagerImpl.java中。

``` java
public interface ViewManager
{
public void addView(View view, ViewGroup.LayoutParams params);
public void updateViewLayout(View view, ViewGroup.LayoutParams params);
public void removeView(View view);
}
```

[->WindowManagerImpl.java]

``` java
Override
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    ......
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}
```

[->WindowManagerGlobal.java]

``` java
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

##### 6.2、ViewRootImpl()构造过程：
[ViewRootImpl.java # ViewRootImpl()]

``` java
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

(1)        通过WindowManagerGlobal.getWindowSession()得到IWindowSession的代理对象，该对象用于和WMS通信。

(2)        创建了一个W本地Binder对象，用于WMS通知应用程序进程。

(3)        采用单例模式创建了一个Choreographer对象，用于统一调度窗口绘图。

(4)        创建ViewRootHandler对象，用于处理当前视图消息。

(5)        构造一个AttachInfo对象；

(6)        创建Surface对象，用于绘制当前视图，当然该Surface对象的真正创建是由WMS来完成的，只不过是WMS传递给应用程序进程的。


##### 6.3、IWindowSession代理获取过程
[->WindowManagerGlobal.java]

``` java
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

以上函数通过WMS的openSession函数创建应用程序与WMS之间的连接通道，即获取IWindowSession代理对象，并将该代理对象保存到ViewRootImpl的静态成员变量sWindowSession中,因此在应用程序进程中有且只有一个IWindowSession代理对象。
[->WindowManagerService.java]

``` java
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

##### 6.4、视图View添加过程
窗口管理器WindowManagerImpl为当前添加的窗口创建好各种对象后，调用ViewRootImpl的setView函数向WMS服务添加一个窗口对象。
[->ViewRootImpl.java]

``` java
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

(1)         requestLayout()在应用程序进程中进行窗口UI布局；

(2)         WindowSession.addToDisplay()向WMS服务注册一个窗口对象；

(3)         注册应用程序进程端的消息接收通道；

**Vsync信号处理来到时，会执行performTraversals()，从而执行relayoutWindow()方法。**

##### 6.5、执行窗口注册relayoutWindow；
[->ViewRootImpl.java]

``` java
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

这里通过前面获取的IWindowSession代理对象请求WMS服务执行窗口布局，mSurface是ViewRootImpl的成员变量
[->ViewRootImpl.java]

``` java
final Surface mSurface = new Surface();
```

[->Surface.java]

``` java
    /**
     * Create an empty surface, which will later be filled in by readFromParcel().
     * @hide
     */
    public Surface() {
    }

```

该Surface构造函数仅仅创建了一个空Surface对象，并没有对该Surface进程native层的初始化，到此我们知道应用程序进程为每个窗口对象都创建了一个Surface对象。并且将该Surface通过跨进程方式传输给WMS服务进程，我们知道，在Android系统中，如果一个对象需要在不同进程间传输，必须实现Parcelable接口，Surface类正好实现了Parcelable接口。ViewRootImpl通过IWindowSession接口请求WMS的完整过程如下：

[->IWindowSession.java$ Proxy]

``` java
/*
* This file is auto-generated.  DO NOT MODIFY
*  * Original file: frameworks/base/core/java/android/view/IWindowSession.aidl
*/
Override
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

``` java
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

到目前为止，在应用程序进程和WMS服务进程分别创建了一个Surface对象，但是他们调用的都是Surface的无参构造函数，在该构造函数中并未真正初始化native层的Surface，那native层的Surface是在那里创建的呢？

[->Session.java]

``` java
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

``` java
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

``` java
    void getSurface(Surface outSurface) {
    outSurface.copyFrom(mSurfaceControl);
}
```

[->WindowStateAnimator.java]

``` java
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

##### 6.6、Surface创建过程
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

  SurfaceFlinger根据标志位创建对应类型的Surface，当前系统定义了2种类型的Layer:
  [->ISurfaceComposerClient.h]

``` cpp
eFXSurfaceNormal    = 0x00000000,
eFXSurfaceDim       = 0x00020000,
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

在SurfaceFlinger服务端为应用程序创建的Surface创建对应的Layer对象

#### （X）、参考文档(特别感谢)：
[（1）【Android Display System】](http://charlesvincent.cc/)
[（2）【android 9.0 activity启动源码分析<一>】](https://www.jianshu.com/p/50e4b48ed15c)
[（3）【（Android 9.0）Activity启动流程源码分析】](https://blog.csdn.net/lj19851227/article/details/82562115)



