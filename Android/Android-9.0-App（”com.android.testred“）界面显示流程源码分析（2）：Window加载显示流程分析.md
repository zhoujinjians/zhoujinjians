---
title: Android 9.0 App（"com.android.testgreen"）界面显示流程源码分析（2）：Window加载显示流程分析
cover: https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/hexo.themes/bing-wallpaper-2018.04.43.jpg
categories:
  - Graphics
tags:
  - Android
  - Graphics
toc: true
abbrlink: 20190728
date: 2019-07-28 09:25:00
---

--------------------------------------------------------------------------------

注：文章都是通过阅读各位前辈总结的资料 Android 9.0 && Linux（Kernel 3.18）Qualcomm平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除（◕‿◕），转载请注明出处（©Qualcomm ©Android @Linux 版权所有），谢谢。
（==**文章基于 Kernel-3.18**==）&&（==**文章基于 Android 9.0**==）

[【开发板 Intrinsyc Open-Q™ 820 µSOM Development Kit】](https://www.intrinsyc.com/snapdragon-embedded-development-kits/open-q-820-usom-development-kit/)
[【开发板 Android 9.0 && Linux（Kernel 3.18）源码链接】](https://gitlab.com/zhoujinjiancc/apq8096_la.um.7.5.r1-03100-8x96.0_p_v5.0)

正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢！！！

[（1）【Android Display System】](http://charlesvincent.cc/)


--------------------------------------------------------------------------------
==源码（部分）==：
**packages/apps/testViewportGreen && testViewportBlue && testViewportRed/**
**frameworks/base/core/java/android/app/**
**frameworks/base/services/core/java/com/android/server/wm/**
**frameworks/base/services/core/java/com/android/server/am/**
**frameworks/base/services/core/java/com/android/server/policy/**
**frameworks/native/services/surfaceflinger/**
**frameworks/native/libs/gui/**
**frameworks/native/libs/ui/**

--------------------------------------------------------------------------------
![enter image description here](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/personal.website/zjj.display.sys.Android-Graphics-Architecture.png) 

####  （一）、Window添加过程

##### （1）、ActivityThread.performLaunchActivity()
接着上一篇文章开始分析，performLaunchActivity()方法完成了两件事：

``` java
G:\android9.0\frameworks\base\core\java\android\app\ActivityThread.java

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
```


-  Activity窗口对象PhoneWindow的创建，通过attach函数来完成；

``` java
G:\android9.0\frameworks\base\core\java\android\app\Activity.java

    final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback) {
        attachBaseContext(context);

        mFragments.attachHost(null /*parent*/);

        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        mWindow.setWindowControllerCallback(this);
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
            mWindow.setSoftInputMode(info.softInputMode);
        }
        if (info.uiOptions != 0) {
            mWindow.setUiOptions(info.uiOptions);
        }
        mUiThread = Thread.currentThread();

        mMainThread = aThread;
        mInstrumentation = instr;
        mToken = token;
        mIdent = ident;
        mApplication = application;
        mIntent = intent;
        mReferrer = referrer;
        mComponent = intent.getComponent();
        mActivityInfo = info;
        mTitle = title;
        mParent = parent;
        mEmbeddedID = id;
        mLastNonConfigurationInstances = lastNonConfigurationInstances;
        if (voiceInteractor != null) {
            if (lastNonConfigurationInstances != null) {
                mVoiceInteractor = lastNonConfigurationInstances.voiceInteractor;
            } else {
                mVoiceInteractor = new VoiceInteractor(voiceInteractor, this, this,
                        Looper.myLooper());
            }
        }

        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        if (mParent != null) {
            mWindow.setContainer(mParent.getWindow());
        }
        mWindowManager = mWindow.getWindowManager();
        mCurrentConfig = config;

        mWindow.setColorMode(info.colorMode);

        setAutofillCompatibilityEnabled(application.isAutofillCompatibilityEnabled());
        enableAutofillCompatibilityIfNeeded();
    }
```

- Activity视图对象的创建，通过callActivityOnCreate >>> setContentView函数来完成；


##### （2）、Activity.setContentView()

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
G:\android9.0\frameworks\base\core\java\android\app\Activity.java

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
    public void setContentView(int layoutResID) {
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
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

Activity.onCreate()会调用setContentView(),整个过程主要是Activity的布局文件或View添加至窗口里，
详细过程不再赘述，详细加载过程请参考：[ Android应用setContentView与LayoutInflater加载解析机制源码分析](http://blog.csdn.net/yanbober/article/details/45970721)
重点概括为：
``` java
● 创建一个DecorView的对象mDecor，该mDecor对象将作为整个应用窗口的根视图。
● 依据Feature等style theme创建不同的窗口修饰布局文件，并且通过findViewById获取Activity布局文件该存放的地方（窗口修饰布局文件中id为content的FrameLayout）。
● 将Activity的布局文件添加至id为content的FrameLayout内。
● 当setContentView设置显示OK以后会回调Activity的onContentChanged方法。Activity的各种View的findViewById()方法等都可以放到该方法中，系统会帮忙回调。
```
#####（3）、ActivityThread.handleResumeActivity()
在调用完performLaunchActivity()方法之后，其有掉用了handleResumeActivity()法。

``` java
G:\android9.0\frameworks\base\core\java\android\app\ActivityThread.java

    @Override
    public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
            String reason) {
        unscheduleGcIdler();
        mSomeActivitiesChanged = true;

        // TODO Push resumeArgs into the activity for consideration
        final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
        ......
        final Activity a = r.activity;

        if (localLOGV) {
            Slog.v(TAG, "Resume " + r + " started activity: " + a.mStartedActivity
                    + ", hideForNow: " + r.hideForNow + ", finished: " + a.mFinished);
        }

        final int forwardBit = isForward
                ? WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION : 0;

        ......
        if (r.window == null && !a.mFinished && willBeVisible) {
            r.window = r.activity.getWindow();
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
            l.softInputMode |= forwardBit;
            ......
            if (a.mVisibleFromClient) {
                if (!a.mWindowAdded) {
                    a.mWindowAdded = true;
                    wm.addView(decor, l);
                } else {
                    a.onWindowAttributesChanged(l);
                }
            }
        } else if (!willBeVisible) {
            r.hideForNow = true;
        }

        cleanUpPendingRemoveWindows(r, false /* force */);

        if (!r.activity.mFinished && willBeVisible && r.activity.mDecor != null && !r.hideForNow) {
            if (r.newConfig != null) {
                performConfigurationChangedForActivity(r, r.newConfig);
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

我们知道，在前面的performLaunchActivity函数中完成Activity的创建后，会将当前当前创建的Activity在应用程序进程端的描述符ActivityClientRecord以键值对的形式保存到ActivityThread的成员变量mActivities中：mActivities.put(r.token, r);，r.token就是Activity的身份证，即是IApplicationToken.Proxy代理对象，也用于与AMS通信。上面的函数首先通过performResumeActivity从mActivities变量中取出Activity的应用程序端描述符ActivityClientRecord，然后取出前面为Activity创建的视图对象DecorView和窗口管理器WindowManager，最后将视图对象添加到窗口管理器中。

#####（4）、ViewManager.addView()

ViewManager.addView()真正实现的的地方在WindowManagerImpl.java中

``` java
G:\android9.0\frameworks\base\core\java\android\view\ViewManager.java

public interface ViewManager
{
public void addView(View view, ViewGroup.LayoutParams params);
public void updateViewLayout(View view, ViewGroup.LayoutParams params);
public void removeView(View view);
}
```

[->WindowManagerImpl.java]

``` java
G:\android9.0\frameworks\base\core\java\android\view\WindowManagerImpl.java

Override
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    ......
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}
```

[->WindowManagerGlobal.java]

``` java
G:\android9.0\frameworks\base\core\java\android\view\WindowManagerGlobal.java

    public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {
    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
    if (parentWindow != null) {
        parentWindow.adjustLayoutParamsForSubWindow(wparams);
    } else {
        final Context context = view.getContext();
        if (context != null
                && (context.getApplicationInfo().flags
                        & ApplicationInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
            wparams.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
        }
    }
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

#####（5）、ViewRootImpl.setView()

##### 5.1、ViewRootImpl()构造过程
``` java
G:\android9.0\frameworks\base\core\java\android\view\ViewRootImpl.java

    final W mWindow;
    final Surface mSurface = new Surface();
    final ViewRootHandler mHandler = new ViewRootHandler();
    ......

    public ViewRootImpl(Context context, Display display) {
        mContext = context;
        mWindowSession = WindowManagerGlobal.getWindowSession();
        mDisplay = display;
        mBasePackageName = context.getBasePackageName();
        mThread = Thread.currentThread();
        mLocation = new WindowLeaked(null);
        mLocation.fillInStackTrace();
        mWidth = -1;
        mHeight = -1;
        mDirty = new Rect();
        mTempRect = new Rect();
        mVisRect = new Rect();
        mWinFrame = new Rect();
        mWindow = new W(this);
        mTargetSdkVersion = context.getApplicationInfo().targetSdkVersion;
        mViewVisibility = View.GONE;
        mTransparentRegion = new Region();
        mPreviousTransparentRegion = new Region();
        mFirst = true; // true for the first time the view is added
        mAdded = false;
        mAttachInfo = new View.AttachInfo(mWindowSession, mWindow, display, this, mHandler, this,
                context);
        mAccessibilityManager = AccessibilityManager.getInstance(context);
        mAccessibilityManager.addAccessibilityStateChangeListener(
                mAccessibilityInteractionConnectionManager, mHandler);
        mHighContrastTextManager = new HighContrastTextManager();
        mAccessibilityManager.addHighTextContrastStateChangeListener(
                mHighContrastTextManager, mHandler);
        mViewConfiguration = ViewConfiguration.get(context);
        mDensity = context.getResources().getDisplayMetrics().densityDpi;
        mNoncompatDensity = context.getResources().getDisplayMetrics().noncompatDensityDpi;
        mFallbackEventHandler = new PhoneFallbackEventHandler(context);
        mChoreographer = Choreographer.getInstance();
        mDisplayManager = (DisplayManager)context.getSystemService(Context.DISPLAY_SERVICE);
        loadSystemProperties();
        mPerf = new BoostFramework(context);
    }
```

在ViewRootImpl的构造函数中初始化了一些成员变量，ViewRootImpl创建了以下几个主要对象：

(1)        通过WindowManagerGlobal.getWindowSession()得到IWindowSession的代理对象，该对象用于和WMS通信。

(2)        创建了一个W本地Binder对象，用于WMS与APP进程通信。

(3)        采用单例模式创建了一个Choreographer对象，用于统一调度窗口绘图。

(4)        创建ViewRootHandler对象，用于处理当前视图消息。

(5)        构造一个AttachInfo对象；

(6)        创建Surface对象，用于绘制当前视图，当然该Surface对象的真正创建是由WMS来完成的，只不过是WMS传递给应用程序进程的。

##### 5.1.1、IWindowSession代理获取过程

``` java
G:\android9.0\frameworks\base\core\java\android\view\WindowManagerGlobal.java

    public static IWindowSession getWindowSession() {
        synchronized (WindowManagerGlobal.class) {
            if (sWindowSession == null) {
                try {
                    InputMethodManager imm = InputMethodManager.getInstance();
                    IWindowManager windowManager = getWindowManagerService();
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
###### 5.1.1.1、WindowManagerService.openSession()
通过Binder通信WindowManagerService.openSession()
通过WMS的openSession函数创建应用程序与WMS之间的连接通道，即获取IWindowSession代理对象，并将该代理对象保存到ViewRootImpl的静态成员变量sWindowSession中,因此在应用程序进程中有且只有一个IWindowSession代理对象。

``` java
G:\android9.0\frameworks\base\services\core\java\com\android\server\wm\WindowManagerService.java

    @Override
    public IWindowSession openSession(IWindowSessionCallback callback, IInputMethodClient client,
            IInputContext inputContext) {
        if (client == null) throw new IllegalArgumentException("null client");
        if (inputContext == null) throw new IllegalArgumentException("null inputContext");
        Session session = new Session(this, callback, client, inputContext);
        return session;
    }

```
###### 5.1.1.2、new Session()
在WMS服务端构造了一个Session实例对象。ViewRootImpl 是一很重要的类，类似 ActivityThread 负责跟AmS通信一样，ViewRootImpl 的一个重要职责就是跟 WmS 通信，它通静态变量 sWindowSession（IWindowSession实例）与 WmS 进行通信。每个应用进程，仅有一个 sWindowSession 对象，它对应了 WmS 中的 Session 子类，WmS 为每一个应用进程分配一个 Session 对象。WindowState 类有一个 IWindow mClient 参数，是在构造方法中赋值的，是由 Session 调用 addWindow 传递过来了，对应了 ViewRootImpl 中的 W 类的实例。

``` java
G:\android9.0\frameworks\base\services\core\java\com\android\server\wm\Session.java

class Session extends IWindowSession.Stub implements IBinder.DeathRecipient {
    final WindowManagerService mService;
    final IWindowSessionCallback mCallback;
    final IInputMethodClient mClient;
    final int mUid;
    final int mPid;
    private final String mStringName;
    SurfaceSession mSurfaceSession;
    private int mNumWindow = 0;
    // Set of visible application overlay window surfaces connected to this session.
    private final Set<WindowSurfaceController> mAppOverlaySurfaces = new HashSet<>();
    // Set of visible alert window surfaces connected to this session.
    private final Set<WindowSurfaceController> mAlertWindowSurfaces = new HashSet<>();
    private final DragDropController mDragDropController;
    final boolean mCanAddInternalSystemWindow;
    final boolean mCanHideNonSystemOverlayWindows;
    final boolean mCanAcquireSleepToken;
    private AlertWindowNotification mAlertWindowNotification;
    private boolean mShowingAlertWindowNotificationAllowed;
    private boolean mClientDead = false;
    private float mLastReportedAnimatorScale;
    private String mPackageName;
    private String mRelayoutTag;

    public Session(WindowManagerService service, IWindowSessionCallback callback,
            IInputMethodClient client, IInputContext inputContext) {
        mService = service;
        mCallback = callback;
        mClient = client;
        ......
        mDragDropController = mService.mDragDropController;
        ......
        synchronized (mService.mWindowMap) {
            if (mService.mInputMethodManager == null && mService.mHaveInputMethods) {
                IBinder b = ServiceManager.getService(
                        Context.INPUT_METHOD_SERVICE);
                mService.mInputMethodManager = IInputMethodManager.Stub.asInterface(b);
            }
        }
        long ident = Binder.clearCallingIdentity();
        try {
            if (mService.mInputMethodManager != null) {
                mService.mInputMethodManager.addClient(client, inputContext,
                        mUid, mPid);
            } else {
                client.setUsingInputMethod(false);
            }
            client.asBinder().linkToDeath(this, 0);
        } catch (RemoteException e) {
           ......
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
    }

```

##### 5.1.2、创建W本地Binder对象
``` java
G:\android9.0\frameworks\base\core\java\android\view\ViewRootImpl.java

static class W extends IWindow.Stub {
    private final WeakReference<ViewRootImpl> mViewAncestor;
    private final IWindowSession mWindowSession;

    W(ViewRootImpl viewAncestor) {
        mViewAncestor = new WeakReference<ViewRootImpl>(viewAncestor);
        mWindowSession = viewAncestor.mWindowSession;
    }
    ......
    }
```
![enter image description here](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/personal.website/zjj.display.sys.IWindow_session.png) 


##### 5.1.3、Choreographer创建

``` java
G:\android9.0\frameworks\base\core\java\android\view\Choreographer.java

    private Choreographer(Looper looper, int vsyncSource) {
        mLooper = looper;
        mHandler = new FrameHandler(looper);
        mDisplayEventReceiver = USE_VSYNC
                ? new FrameDisplayEventReceiver(looper, vsyncSource)
                : null;
        mLastFrameTimeNanos = Long.MIN_VALUE;

        mFrameIntervalNanos = (long)(1000000000 / getRefreshRate());

        mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];
        for (int i = 0; i <= CALLBACK_LAST; i++) {
            mCallbackQueues[i] = new CallbackQueue();
        }
        // b/68769804: For low FPS experiments.
        setFPSDivisor(SystemProperties.getInt(ThreadedRenderer.DEBUG_FPS_DIVISOR, 1));
    }

G:\android9.0\frameworks\base\core\java\android\view\Choreographer.java
  private final class FrameDisplayEventReceiver extends DisplayEventReceiver
            implements Runnable {
        ......
        public FrameDisplayEventReceiver(Looper looper, int vsyncSource) {
            super(looper, vsyncSource);
        }
        ......
    }
```
FrameDisplayEventReceiver 继承 DisplayEventReceiver

``` java
G:\android9.0\frameworks\base\core\java\android\view\DisplayEventReceiver.java
    public DisplayEventReceiver(Looper looper, int vsyncSource) {
        ......
        mMessageQueue = looper.getQueue();
        mReceiverPtr = nativeInit(new WeakReference<DisplayEventReceiver>(this), mMessageQueue,
                vsyncSource);

        mCloseGuard.open("dispose");
    }
>>>>>>JNI
G:\android9.0\frameworks\base\core\jni\android_view_DisplayEventReceiver.cpp

static jlong nativeInit(JNIEnv* env, jclass clazz, jobject receiverWeak,
        jobject messageQueueObj, jint vsyncSource) {
    sp<MessageQueue> messageQueue = android_os_MessageQueue_getMessageQueue(env, messageQueueObj);
    if (messageQueue == NULL) {
        jniThrowRuntimeException(env, "MessageQueue is not initialized.");
        return 0;
    }

    sp<NativeDisplayEventReceiver> receiver = new NativeDisplayEventReceiver(env,
            receiverWeak, messageQueue, vsyncSource);
    status_t status = receiver->initialize();
    if (status) {
        String8 message;
        message.appendFormat("Failed to initialize display event receiver.  status=%d", status);
        jniThrowRuntimeException(env, message.string());
        return 0;
    }

    receiver->incStrong(gDisplayEventReceiverClassInfo.clazz); // retain a reference for the object
    return reinterpret_cast<jlong>(receiver.get());
}


NativeDisplayEventReceiver::NativeDisplayEventReceiver(JNIEnv* env,
        jobject receiverWeak, const sp<MessageQueue>& messageQueue, jint vsyncSource) :
        DisplayEventDispatcher(messageQueue->getLooper(),
                static_cast<ISurfaceComposer::VsyncSource>(vsyncSource)),
        mReceiverWeakGlobal(env->NewGlobalRef(receiverWeak)),
        mMessageQueue(messageQueue) {
    ALOGV("receiver %p ~ Initializing display event receiver.", this);
}

```
NativeDisplayEventReceiver 继承与DisplayEventDispatcher，所以DisplayEventDispatcher会执行初始化操作。

``` cpp
G:\android9.0\frameworks\base\libs\androidfw\DisplayEventDispatcher.cpp

DisplayEventDispatcher::DisplayEventDispatcher(const sp<Looper>& looper,
        ISurfaceComposer::VsyncSource vsyncSource) :
        mLooper(looper), mReceiver(vsyncSource), mWaitingForVsync(false) {
    ALOGV("dispatcher %p ~ Initializing display event dispatcher.", this);
}

status_t DisplayEventDispatcher::initialize() {
    status_t result = mReceiver.initCheck();
    if (result) {
        ALOGW("Failed to initialize display event receiver, status=%d", result);
        return result;
    }

    int rc = mLooper->addFd(mReceiver.getFd(), 0, Looper::EVENT_INPUT,
            this, NULL);
    if (rc < 0) {
        return UNKNOWN_ERROR;
    }
    return OK;
}
```
 监听mReceiver的所获取的文件句柄，一旦有数据到来，则回调this(此处NativeDisplayEventReceiver)中所复写LooperCallback对象的 handleEvent。

##### 5.1.3.1、Native DisplayEventReceiver初始化
mReceiver为DisplayEventReceiver实例

``` cpp
G:\android9.0\frameworks\native\libs\gui\DisplayEventReceiver.cpp

DisplayEventReceiver::DisplayEventReceiver(ISurfaceComposer::VsyncSource vsyncSource) {
    sp<ISurfaceComposer> sf(ComposerService::getComposerService());
    if (sf != NULL) {
        mEventConnection = sf->createDisplayEventConnection(vsyncSource);
        if (mEventConnection != NULL) {
            mDataChannel = std::make_unique<gui::BitTube>();
            mEventConnection->stealReceiveChannel(mDataChannel.get());
        }
    }
}
```
此处通过binder通信与SurfaceFlinger建立连接，直接看SurfaceFlinger服务端的实现

##### 5.1.3.2、createDisplayEventConnection

``` cpp
G:\android9.0\frameworks\native\services\surfaceflinger\SurfaceFlinger.cpp
// ----------------------------------------------------------------------------

sp<IDisplayEventConnection> SurfaceFlinger::createDisplayEventConnection(
        ISurfaceComposer::VsyncSource vsyncSource) {
    if (vsyncSource == eVsyncSourceSurfaceFlinger) {
        return mSFEventThread->createEventConnection();
    } else {
        return mEventThread->createEventConnection();
    }
}
```
我们这里不是eVsyncSourceSurfaceFlinger，所以会走mEventThread->createEventConnection() 分支，为app进程创建一个Connection，会首先调用Connection::onFirstRef()，然后加入到mDisplayEventConnections中。

``` cpp
G:\android9.0\frameworks\native\services\surfaceflinger\EventThread.cpp

sp<BnDisplayEventConnection> EventThread::createEventConnection() const {
    return new Connection(const_cast<EventThread*>(this));
}

EventThread::Connection::Connection(EventThread* eventThread)
      : count(-1), mEventThread(eventThread), mChannel(gui::BitTube::DefaultSize) {}

void EventThread::Connection::onFirstRef() {
    // NOTE: mEventThread doesn't hold a strong reference on us
    mEventThread->registerDisplayEventConnection(this);
}

status_t EventThread::registerDisplayEventConnection(
        const sp<EventThread::Connection>& connection) {
    std::lock_guard<std::mutex> lock(mMutex);
    mDisplayEventConnections.add(connection);
    mCondition.notify_all();
    return NO_ERROR;
}

```


##### 5.1.3.3、mEventConnection->stealReceiveChannel(mDataChannel.get())
Binder通信直接看服务端实现，将从客户端传来的BitTube设置到接收VSync信号的FD

``` cpp
G:\android9.0\frameworks\native\services\surfaceflinger\EventThread.cpp

status_t EventThread::Connection::stealReceiveChannel(gui::BitTube* outChannel) {
    outChannel->setReceiveFd(mChannel.moveReceiveFd());
    return NO_ERROR;
}

G:\android9.0\frameworks\native\libs\gui\BitTube.cpp
void BitTube::setReceiveFd(base::unique_fd&& receiveFd) {
    mReceiveFd = std::move(receiveFd);
}

```

##### 5.1.4、构造AttachInfo对象

``` java
G:\android9.0\frameworks\base\core\java\android\view\View.java

    final static class AttachInfo {
        ......
        final IWindowSession mSession;
        final IWindow mWindow;
        final IBinder mWindowToken;
        Display mDisplay;
        final Callbacks mRootCallbacks;
        IWindowId mIWindowId;
        WindowId mWindowId;
        View mRootView;
        ......
        final ViewTreeObserver mTreeObserver;
        Canvas mCanvas;
        final ViewRootImpl mViewRootImpl;
        final Handler mHandler;
        ......
        AttachInfo(IWindowSession session, IWindow window, Display display,
                ViewRootImpl viewRootImpl, Handler handler, Callbacks effectPlayer,
                Context context) {
            mSession = session;
            mWindow = window;
            mWindowToken = window.asBinder();
            mDisplay = display;
            mViewRootImpl = viewRootImpl;
            mHandler = handler;
            mRootCallbacks = effectPlayer;
            mTreeObserver = new ViewTreeObserver(context);
        }
```

##### 5.2、视图View添加过程
窗口管理器WindowManagerImpl为当前添加的窗口创建好各种对象后，调用ViewRootImpl的setView函数向WMS服务添加一个窗口对象。
``` java
G:\android9.0\frameworks\base\core\java\android\view\ViewRootImpl.java

    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                mView = view;//将DecorView保存到ViewRootImpl的成员变量mView中

                mAttachInfo.mDisplayState = mDisplay.getState();
                mDisplayManager.registerDisplayListener(mDisplayListener, mHandler);

                mViewLayoutDirectionInitial = mView.getRawLayoutDirection();
                mFallbackEventHandler.setView(view);
                mWindowAttributes.copyFrom(attrs);
                ......
                attrs = mWindowAttributes;
                setTag();

                ......
                mClientWindowLayoutFlags = attrs.flags;

                setAccessibilityFocus(null, null);

                ......

                CompatibilityInfo compatibilityInfo =
                        mDisplay.getDisplayAdjustments().getCompatibilityInfo();
                mTranslator = compatibilityInfo.getTranslator();

                ......
                if (DEBUG_LAYOUT) Log.d(mTag, "WindowLayout in setView:" + attrs);
---------------------------------------------------
对应Log：
	Line 2310: 04-16 13:08:36.961 D/ViewRootImpl[TestActivity]( 2602): WindowLayout in setView:{(0,0)(fillxfill) sim={forwardNavigation} ty=BASE_APPLICATION fmt=TRANSPARENT wanim=0x1030000
	Line 2311: 04-16 13:08:36.961 D/ViewRootImpl[TestActivity]( 2602):   fl=LAYOUT_IN_SCREEN LAYOUT_INSET_DECOR SPLIT_TOUCH HARDWARE_ACCELERATED}
---------------------------------------------------
                if (!compatibilityInfo.supportsScreen()) {
                    attrs.privateFlags |= WindowManager.LayoutParams.PRIVATE_FLAG_COMPATIBLE_WINDOW;
                    mLastInCompatMode = true;
                }

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

                // Schedule the first layout -before- adding to the window
                // manager, to make sure we do the relayout before receiving
                // any other events from the system.
                1）在添加窗口前进行UI布局 
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
                     
                    // 2)将窗口添加到WMS服务中，mWindow为W本地Binder对象，通过Binder传输到WMS服务端后，变为IWindow代理对象  
                    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(), mWinFrame,
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mAttachInfo.mDisplayCutout, mInputChannel);
                } catch (RemoteException e) {
                    mAdded = false;
                    mView = null;
                    mAttachInfo.mRootView = null;
                    mInputChannel = null;
                    mFallbackEventHandler.setView(null);
                    unscheduleTraversals();
                    setAccessibilityFocus(null, null);
                    throw new RuntimeException("Adding window failed", e);
                } finally {
                    if (restore) {
                        attrs.restore();
                    }
                }

                ......

----------------------------------------------------------------
对应Log：
04-16 13:08:36.968 V/ViewRootImpl[TestActivity]( 2602): Added window android.view.ViewRootImpl$W@ad597e9
----------------------------------------------------------------
                if (DEBUG_LAYOUT) Log.v(mTag, "Added window " + mWindow);
                ......
                //3)建立窗口消息通道  
                if (mInputChannel != null) {
                    if (mInputQueueCallback != null) {
                        mInputQueue = new InputQueue();
                        mInputQueueCallback.onInputQueueCreated(mInputQueue);
                    }
                    mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
                            Looper.myLooper());
                }

                view.assignParent(this);
                mAddedTouchMode = (res & WindowManagerGlobal.ADD_FLAG_IN_TOUCH_MODE) != 0;
                mAppVisible = (res & WindowManagerGlobal.ADD_FLAG_APP_VISIBLE) != 0;

                if (mAccessibilityManager.isEnabled()) {
                    mAccessibilityInteractionConnectionManager.ensureConnection();
                }

                if (view.getImportantForAccessibility() == View.IMPORTANT_FOR_ACCESSIBILITY_AUTO) {
                    view.setImportantForAccessibility(View.IMPORTANT_FOR_ACCESSIBILITY_YES);
                }

                // Set up the input pipeline.
                CharSequence counterSuffix = attrs.getTitle();
                mSyntheticInputStage = new SyntheticInputStage();
                InputStage viewPostImeStage = new ViewPostImeInputStage(mSyntheticInputStage);
                InputStage nativePostImeStage = new NativePostImeInputStage(viewPostImeStage,
                        "aq:native-post-ime:" + counterSuffix);
                InputStage earlyPostImeStage = new EarlyPostImeInputStage(nativePostImeStage);
                InputStage imeStage = new ImeInputStage(earlyPostImeStage,
                        "aq:ime:" + counterSuffix);
                InputStage viewPreImeStage = new ViewPreImeInputStage(imeStage);
                InputStage nativePreImeStage = new NativePreImeInputStage(viewPreImeStage,
                        "aq:native-pre-ime:" + counterSuffix);

                mFirstInputStage = nativePreImeStage;
                mFirstPostImeInputStage = earlyPostImeStage;
                mPendingInputEventQueueLengthCounterName = "aq:pending:" + counterSuffix;
            }
        }
    }


```

通过前面的分析可以知道，用户自定义的UI作为一个子View被添加到DecorView中，然后将顶级视图DecorView添加到应用程序进程的窗口管理器中，窗口管理器首先为当前添加的View创建一个ViewRootImpl对象、一个布局参数对象ViewGroup.LayoutParams，然后将这三个对象分别保存到当前应用程序进程的窗口管理器WindowManagerImpl中，最后通过ViewRootImpl对象将当前视图对象注册到WMS服务中。

ViewRootImpl的setView函数向WMS服务添加一个窗口对象过程：

(1)         requestLayout()在应用程序进程中进行窗口UI布局；

(2)         WindowSession.addToDisplay()向WMS服务注册一个窗口对象；

(3)         注册应用程序进程端的消息接收通道；


##### 5.2.1、窗口UI布局过程 ViewRootImpl.requestLayout()

requestLayout函数调用里面使用了Hanlder的一个小手段，那就是利用postSyncBarrier添加了一个Barrier（挡板），这个挡板的作用是阻塞普通的同步消息的执行，在挡板被撤销之前，只会执行异步消息，而requestLayout先添加了一个挡板Barrier，之后自己插入了一个异步任务mTraversalRunnable，其主要作用就是保证mTraversalRunnable在所有同步Message之前被执行，保证View绘制的最高优先级。具体实现如下：

``` java
G:\android9.0\frameworks\base\core\java\android\view\ViewRootImpl.java
Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}

final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}
final TraversalRunnable mTraversalRunnable = new TraversalRunnable();


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
##### 5.2.2、添加回调过程 mChoreographer.postCallback()

``` java
G:\android9.0\frameworks\base\core\java\android\view\Choreographer.java

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
        if (DEBUG_FRAMES) {
            Log.d(TAG, "PostCallback: type=" + callbackType
                    + ", action=" + action + ", token=" + token
                    + ", delayMillis=" + delayMillis);
        }

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


     scheduleFrameLocked(long now) {
        if (!mFrameScheduled) {
            mFrameScheduled = true;
            ......
            if (USE_VSYNC) {
                if (DEBUG_FRAMES) {
                    Log.d(TAG, "Scheduling next frame on vsync.");
                }

                // If running on the Looper thread, then schedule the vsync immediately,
                // otherwise post a message to schedule the vsync from the UI thread
                // as soon as possible.
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
                if (DEBUG_FRAMES) {
                    Log.d(TAG, "Scheduling next frame in " + (nextFrameTime - now) + " ms.");
                }
                Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtTime(msg, nextFrameTime);
            }
        }
    }
```
消息处理：

``` java
G:\android9.0\frameworks\base\core\java\android\view\Choreographer.java
    private final class FrameHandler extends Handler {
        public FrameHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_DO_FRAME:
                    doFrame(System.nanoTime(), 0);
                    break;
                case MSG_DO_SCHEDULE_VSYNC:
                    doScheduleVsync();
                    break;
                case MSG_DO_SCHEDULE_CALLBACK:
                    doScheduleCallback(msg.arg1);
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
        mDisplayEventReceiver.scheduleVsync();
    }

```
在该函数中考虑了两种情况，一种是系统没有使用Vsync机制，在这种情况下，首先根据屏幕刷新频率计算下一次刷新时间，通过异步消息方式延时执行doFrame()函数实现屏幕刷新。如果系统使用了Vsync机制，并且当前线程具备消息循环，则直接请求Vsync信号，否则就通过主线程来请求Vsync信号。

##### 5.2.3、Vsync请求过程 mDisplayEventReceiver.scheduleVsync()
我们知道在Choreographer构造函数中，构造了一个FrameDisplayEventReceiver对象，用于请求并接收Vsync信号，Vsync信号请求过程如下：

``` java
G:\android9.0\frameworks\base\core\java\android\view\DisplayEventReceiver.java
    public void scheduleVsync() {
        if (mReceiverPtr == 0) {
            Log.w(TAG, "Attempted to schedule a vertical sync pulse but the display event "
                    + "receiver has already been disposed.");
        } else {
            nativeScheduleVsync(mReceiverPtr);
        }
    }

```

``` cpp
G:\android9.0\frameworks\base\core\jni\android_view_DisplayEventReceiver.cpp
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
VSync请求过程又转交给了DisplayEventDispatcher： [->DisplayEventDispatcher.cpp]

``` cpp
G:\android9.0\frameworks\base\libs\androidfw\DisplayEventDispatcher.cpp

status_t DisplayEventDispatcher::scheduleVsync() {
    if (!mWaitingForVsync) {
        ALOGV("dispatcher %p ~ Scheduling vsync.", this);

        // Drain all pending events.
        nsecs_t vsyncTimestamp;
        int32_t vsyncDisplayId;
        uint32_t vsyncCount;
        //处理显示设备插入
        if (processPendingEvents(&vsyncTimestamp, &vsyncDisplayId, &vsyncCount)) {
            ALOGE("dispatcher %p ~ last event processed while scheduling was for %" PRId64 "",
                    this, ns2ms(static_cast<nsecs_t>(vsyncTimestamp)));
        }

        status_t status = mReceiver.requestNextVsync();
        if (status) {
            ALOGW("Failed to request next vsync, status=%d", status);
            return status;
        }

        mWaitingForVsync = true;
    }
    return OK;
}
```
VSync请求过程又转交给了DisplayEventReceiver： [->DisplayEventReceiver.cpp]
``` cpp
G:\android9.0\frameworks\native\libs\gui\DisplayEventReceiver.cpp
status_t DisplayEventReceiver::requestNextVsync() {
    if (mEventConnection != NULL) {
        mEventConnection->requestNextVsync();
        return NO_ERROR;
    }
    return NO_INIT;
}
```
看服务端的实现

``` cpp
G:\android9.0\frameworks\native\services\surfaceflinger\EventThread.cpp

void EventThread::requestNextVsync(const sp<EventThread::Connection>& connection) {
    std::lock_guard<std::mutex> lock(mMutex);

    if (mResyncWithRateLimitCallback) {
        mResyncWithRateLimitCallback();
    }

    if (connection->count < 0) {
        connection->count = 0;
        mCondition.notify_all();
    }
}


void EventThread::threadMain() NO_THREAD_SAFETY_ANALYSIS {
    std::unique_lock<std::mutex> lock(mMutex);
    while (mKeepRunning) {
        DisplayEventReceiver::Event event;
        Vector<sp<EventThread::Connection> > signalConnections;
        signalConnections = waitForEventLocked(&lock, &event);

        // dispatch events to listeners...
        const size_t count = signalConnections.size();
        for (size_t i = 0; i < count; i++) {
            const sp<Connection>& conn(signalConnections[i]);
            // now see if we still need to report this event
            status_t err = conn->postEvent(event);
            if (err == -EAGAIN || err == -EWOULDBLOCK) {
                // The destination doesn't accept events anymore, it's probably
                // full. For now, we just drop the events on the floor.
                // FIXME: Note that some events cannot be dropped and would have
                // to be re-sent later.
                // Right-now we don't have the ability to do this.
                ALOGW("EventThread: dropping event (%08x) for connection %p", event.header.type,
                      conn.get());
            } else if (err < 0) {
                // handle any other error on the pipe as fatal. the only
                // reasonable thing to do is to clean-up this connection.
                // The most common error we'll get here is -EPIPE.
                removeDisplayEventConnectionLocked(signalConnections[i]);
            }
        }
    }
}
```
waitForEventLocked()函数中等待下一个VSYNC的trigger.。

需要说明的是

/**************Vsync start**************/
/**************Vsync end**************/
之间的代码此时其实还未执行，call requestNextVsync()来告诉系统我要在下一个VSYNC需要被trigger.

继续ViewRootImpl的setView函数中的WindowSession.addToDisplay()。

##### 5.3、mWindowSession.addToDisplay()向WMS服务注册一个窗口对象

``` java
G:\android9.0\frameworks\base\services\core\java\com\android\server\wm\Session.java

Override
public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
        int viewVisibility, int displayId, Rect outFrame, Rect outContentInsets,
        Rect outStableInsets, Rect outOutsets,
        DisplayCutout.ParcelableWrapper outDisplayCutout, InputChannel outInputChannel) {
    return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId, outFrame,
            outContentInsets, outStableInsets, outOutsets, outDisplayCutout, outInputChannel);
}
```

看看WindowManagerService.java的addWindow()实现
###### 5.3.1、WindowManagerService.addWindow()
``` java
G:\android9.0\frameworks\base\services\core\java\com\android\server\wm\WindowManagerService.java


    public int addWindow(Session session, IWindow client, int seq,
            LayoutParams attrs, int viewVisibility, int displayId, Rect outFrame,
            Rect outContentInsets, Rect outStableInsets, Rect outOutsets,
            DisplayCutout.ParcelableWrapper outDisplayCutout, InputChannel outInputChannel) {
        int[] appOp = new int[1];
        ......

        boolean reportNewConfig = false;
        WindowState parentWindow = null;
        long origId;
        final int callingUid = Binder.getCallingUid();
        final int type = attrs.type;

        mFocusingActivity = attrs.getTitle().toString();

        synchronized(mWindowMap) {
            .......
            final DisplayContent displayContent = getDisplayContentOrCreate(displayId);

            ......

            AppWindowToken atoken = null;
            final boolean hasParent = parentWindow != null;
            // Use existing parent window token for child windows since they go in the same token
            // as there parent window so we can apply the same policy on them.
            WindowToken token = displayContent.getWindowToken(
                    hasParent ? parentWindow.mAttrs.token : attrs.token);
            // If this is a child window, we want to apply the same type checking rules as the
            // parent window type.
            final int rootType = hasParent ? parentWindow.mAttrs.type : type;

            boolean addToastWindowRequiresToken = false;

            if (token == null) {
                .......
                final IBinder binder = attrs.token != null ? attrs.token : client.asBinder();
                final boolean isRoundedCornerOverlay =
                        (attrs.privateFlags & PRIVATE_FLAG_IS_ROUNDED_CORNERS_OVERLAY) != 0;
                token = new WindowToken(this, binder, type, false, displayContent,
                        session.mCanAddInternalSystemWindow, isRoundedCornerOverlay);
            } ......
            } else if (token.asAppWindowToken() != null) {
                Slog.w(TAG_WM, "Non-null appWindowToken for system window of rootType=" + rootType);
                // It is not valid to use an app token with other system types; we will
                // instead make a new token for it (as if null had been passed in for the token).
                attrs.token = null;
                token = new WindowToken(this, client.asBinder(), type, false, displayContent,
                        session.mCanAddInternalSystemWindow);
            }

            final WindowState win = new WindowState(this, session, client, token, parentWindow,
                    appOp[0], seq, attrs, viewVisibility, session.mUid,
                    session.mCanAddInternalSystemWindow);
            ......

            final boolean hasStatusBarServicePermission =
                    mContext.checkCallingOrSelfPermission(permission.STATUS_BAR_SERVICE)
                            == PackageManager.PERMISSION_GRANTED;
            mPolicy.adjustWindowParamsLw(win, win.mAttrs, hasStatusBarServicePermission);
            win.setShowToOwnerOnlyLocked(mPolicy.checkShowToOwnerOnly(attrs));

            res = mPolicy.prepareAddWindowLw(win, attrs);
            ......

            final boolean openInputChannels = (outInputChannel != null
                    && (attrs.inputFeatures & INPUT_FEATURE_NO_INPUT_CHANNEL) == 0);
            if  (openInputChannels) {
                win.openInputChannel(outInputChannel);
            }

            ......

            res = WindowManagerGlobal.ADD_OKAY;
            ......
            origId = Binder.clearCallingIdentity();

            win.attach();
            //将win加入到mWindowMap中
            mWindowMap.put(client.asBinder(), win);

            win.initAppOpsState();

            final boolean suspended = mPmInternal.isPackageSuspended(win.getOwningPackage(),
                    UserHandle.getUserId(win.getOwningUid()));
            win.setHiddenWhileSuspended(suspended);

            final boolean hideSystemAlertWindows = !mHidingNonSystemOverlayWindows.isEmpty();
            win.setForceHideNonSystemOverlayWindowIfNeeded(hideSystemAlertWindows);

            final AppWindowToken aToken = token.asAppWindowToken();
            if (type == TYPE_APPLICATION_STARTING && aToken != null) {
                aToken.startingWindow = win;
                if (DEBUG_STARTING_WINDOW) Slog.v (TAG_WM, "addWindow: " + aToken
                        + " startingWindow=" + win);
            }

            boolean imMayMove = true;
            //将win加入到mToken中
            win.mToken.addWindow(win);

            if (type == TYPE_INPUT_METHOD) {
                win.mGivenInsetsPending = true;
                setInputMethodWindowLocked(win);
                imMayMove = false;
            } else if (type == TYPE_INPUT_METHOD_DIALOG) {
                displayContent.computeImeTarget(true /* updateImeTarget */);
                imMayMove = false;
            } else {
                if (type == TYPE_WALLPAPER) {
                    displayContent.mWallpaperController.clearLastWallpaperTimeoutTime();
                    displayContent.pendingLayoutChanges |= FINISH_LAYOUT_REDO_WALLPAPER;
                } else if ((attrs.flags&FLAG_SHOW_WALLPAPER) != 0) {
                    displayContent.pendingLayoutChanges |= FINISH_LAYOUT_REDO_WALLPAPER;
                } else if (displayContent.mWallpaperController.isBelowWallpaperTarget(win)) {
                    // If there is currently a wallpaper being shown, and
                    // the base layer of the new window is below the current
                    // layer of the target window, then adjust the wallpaper.
                    // This is to avoid a new window being placed between the
                    // wallpaper and its target.
                    displayContent.pendingLayoutChanges |= FINISH_LAYOUT_REDO_WALLPAPER;
                }
            }

            // If the window is being added to a stack that's currently adjusted for IME,
            // make sure to apply the same adjust to this new window.
            win.applyAdjustForImeIfNeeded();

            if (type == TYPE_DOCK_DIVIDER) {
                mRoot.getDisplayContent(displayId).getDockedDividerController().setWindow(win);
            }

            final WindowStateAnimator winAnimator = win.mWinAnimator;
            winAnimator.mEnterAnimationPending = true;
            winAnimator.mEnteringAnimation = true;
            // Check if we need to prepare a transition for replacing window first.
            if (atoken != null && atoken.isVisible()
                    && !prepareWindowReplacementTransition(atoken)) {
                // If not, check if need to set up a dummy transition during display freeze
                // so that the unfreeze wait for the apps to draw. This might be needed if
                // the app is relaunching.
                prepareNoneTransitionForRelaunching(atoken);
            }

            final DisplayFrames displayFrames = displayContent.mDisplayFrames;
            // TODO: Not sure if onDisplayInfoUpdated() call is needed.
            final DisplayInfo displayInfo = displayContent.getDisplayInfo();
            displayFrames.onDisplayInfoUpdated(displayInfo,
                    displayContent.calculateDisplayCutoutForRotation(displayInfo.rotation));
            final Rect taskBounds;
            if (atoken != null && atoken.getTask() != null) {
                taskBounds = mTmpRect;
                atoken.getTask().getBounds(mTmpRect);
            } else {
                taskBounds = null;
            }
            if (mPolicy.getLayoutHintLw(win.mAttrs, taskBounds, displayFrames, outFrame,
                    outContentInsets, outStableInsets, outOutsets, outDisplayCutout)) {
                res |= WindowManagerGlobal.ADD_FLAG_ALWAYS_CONSUME_NAV_BAR;
            }

            if (mInTouchMode) {
                res |= WindowManagerGlobal.ADD_FLAG_IN_TOUCH_MODE;
            }
            if (win.mAppToken == null || !win.mAppToken.isClientHidden()) {
                res |= WindowManagerGlobal.ADD_FLAG_APP_VISIBLE;
            }

            mInputMonitor.setUpdateInputWindowsNeededLw();

            boolean focusChanged = false;
            if (win.canReceiveKeys()) {
                focusChanged = updateFocusedWindowLocked(UPDATE_FOCUS_WILL_ASSIGN_LAYERS,
                        false /*updateInputWindows*/);
                if (focusChanged) {
                    imMayMove = false;
                }
            }

            if (imMayMove) {
                displayContent.computeImeTarget(true /* updateImeTarget */);
            }

            // Don't do layout here, the window must call
            // relayout to be displayed, so we'll do it there.
            win.getParent().assignChildLayers();

            if (focusChanged) {
                mInputMonitor.setInputFocusLw(mCurrentFocus, false /*updateInputWindows*/);
            }
----------------------------------------------------------------
对应Log：
04-16 13:08:36.967 D/WindowManager( 1049): addInputWindowHandle: Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity}, Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity}, layer=0, frame=[0,0,0,0], touchableRegion=SkRegion((0,0,480,854)), visible=false
----------------------------------------------------------------

            mInputMonitor.updateInputWindowsLw(false /*force*/);
----------------------------------------------------------------
对应Log：
04-16 13:08:36.968 V/WindowManager( 1049): addWindow: New client android.os.BinderProxy@b9baf64: window=Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity} Callers=com.android.server.wm.Session.addToDisplay:205 android.view.IWindowSession$Stub.onTransact:129 com.android.server.wm.Session.onTransact:164 android.os.Binder.execTransact:731 <bottom of call stack> 

----------------------------------------------------------------


            if (localLOGV || DEBUG_ADD_REMOVE) Slog.v(TAG_WM, "addWindow: New client "
                    + client.asBinder() + ": window=" + win + " Callers=" + Debug.getCallers(5));

            if (win.isVisibleOrAdding() && updateOrientationFromAppTokensLocked(displayId)) {
                reportNewConfig = true;
            }
        }

        if (reportNewConfig) {
            sendNewConfiguration(displayId);
        }

        Binder.restoreCallingIdentity(origId);

        return res;
    }

```

###### 5.3.1.1、WindowState创建
``` java
G:\android9.0\frameworks\base\services\core\java\com\android\server\wm\WindowState.java

    WindowState(WindowManagerService service, Session s, IWindow c, WindowToken token,
            WindowState parentWindow, int appOp, int seq, WindowManager.LayoutParams a,
            int viewVisibility, int ownerId, boolean ownerCanAddInternalSystemWindow,
            PowerManagerWrapper powerManagerWrapper) {
        super(service);
        mSession = s;
        mClient = c;
        mAppOp = appOp;
        mToken = token;
        mAppToken = mToken.asAppWindowToken();
        mOwnerUid = ownerId;
        mOwnerCanAddInternalSystemWindow = ownerCanAddInternalSystemWindow;
        mWindowId = new WindowId(this);
        mAttrs.copyFrom(a);
        mLastSurfaceInsets.set(mAttrs.surfaceInsets);
        mViewVisibility = viewVisibility;
        mPolicy = mService.mPolicy;
        mContext = mService.mContext;
        DeathRecipient deathRecipient = new DeathRecipient();
        mSeq = seq;
        mEnforceSizeCompat = (mAttrs.privateFlags & PRIVATE_FLAG_COMPATIBLE_WINDOW) != 0;
        mPowerManagerWrapper = powerManagerWrapper;
----------------------------------------------------------------
对应Log：
04-16 13:08:36.964 V/WindowState( 1049): Window Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity} client=android.os.BinderProxy@b9baf64 token=AppWindowToken{1579134 token=Token{ce2ac07 ActivityRecord{65cc87a u0 com.android.testred/.TestActivity t5}}} (Token{ce2ac07 ActivityRecord{65cc87a u0 com.android.testred/.TestActivity t5}}) params={(0,0)(fillxfill) sim={forwardNavigation} ty=BASE_APPLICATION fmt=TRANSPARENT wanim=0x1030000
04-16 13:08:36.964 V/WindowState( 1049):   fl=LAYOUT_IN_SCREEN LAYOUT_INSET_DECOR SPLIT_TOUCH HARDWARE_ACCELERATED}

----------------------------------------------------------------
        if (localLOGV) Slog.v(
            TAG, "Window " + this + " client=" + c.asBinder()
            + " token=" + token + " (" + mAttrs.token + ")" + " params=" + a);
        try {
            c.asBinder().linkToDeath(deathRecipient, 0);
        } catch (RemoteException e) {
            mDeathRecipient = null;
            mIsChildWindow = false;
            mLayoutAttached = false;
            mIsImWindow = false;
            mIsWallpaper = false;
            mIsFloatingLayer = false;
            mBaseLayer = 0;
            mSubLayer = 0;
            mInputWindowHandle = null;
            mWinAnimator = null;
            return;
        }
        mDeathRecipient = deathRecipient;

	//charlesvincent start
	if("com.android.testred".equals(mAttrs.packageName) 
			|| "com.android.testgreen".equals(mAttrs.packageName)
			|| "com.android.testblue".equals(mAttrs.packageName)){
			mBaseLayer = 21010;
			mSubLayer = 0;
			mIsChildWindow = false;
			mLayoutAttached = false;
			mIsImWindow = mAttrs.type == TYPE_INPUT_METHOD
					|| mAttrs.type == TYPE_INPUT_METHOD_DIALOG;
			mIsWallpaper = mAttrs.type == TYPE_WALLPAPER;
	} else{

	        if (mAttrs.type >= FIRST_SUB_WINDOW && mAttrs.type <= LAST_SUB_WINDOW) {
	            // The multiplier here is to reserve space for multiple
	            // windows in the same type layer.
	            mBaseLayer = mPolicy.getWindowLayerLw(parentWindow)
	                    * TYPE_LAYER_MULTIPLIER + TYPE_LAYER_OFFSET;
	            mSubLayer = mPolicy.getSubWindowLayerFromTypeLw(a.type);
	            mIsChildWindow = true;

	            if (DEBUG_ADD_REMOVE) Slog.v(TAG, "Adding " + this + " to " + parentWindow);
	            parentWindow.addChild(this, sWindowSubLayerComparator);

	            mLayoutAttached = mAttrs.type !=
	                    WindowManager.LayoutParams.TYPE_APPLICATION_ATTACHED_DIALOG;
	            mIsImWindow = parentWindow.mAttrs.type == TYPE_INPUT_METHOD
	                    || parentWindow.mAttrs.type == TYPE_INPUT_METHOD_DIALOG;
	            mIsWallpaper = parentWindow.mAttrs.type == TYPE_WALLPAPER;
	        } else {
	            // The multiplier here is to reserve space for multiple
	            // windows in the same type layer.
	            mBaseLayer = mPolicy.getWindowLayerLw(this)
	                    * TYPE_LAYER_MULTIPLIER + TYPE_LAYER_OFFSET;
	            mSubLayer = 0;
	            mIsChildWindow = false;
	            mLayoutAttached = false;
	            mIsImWindow = mAttrs.type == TYPE_INPUT_METHOD
	                    || mAttrs.type == TYPE_INPUT_METHOD_DIALOG;
	            mIsWallpaper = mAttrs.type == TYPE_WALLPAPER;
	        }

	}
 	//charlesvincent end

        mIsFloatingLayer = mIsImWindow || mIsWallpaper;

        if (mAppToken != null && mAppToken.mShowForAllUsers) {
            // Windows for apps that can show for all users should also show when the device is
            // locked.
            mAttrs.flags |= FLAG_SHOW_WHEN_LOCKED;
        }
        //创建WindowStateAnimator
        mWinAnimator = new WindowStateAnimator(this);
        mWinAnimator.mAlpha = a.alpha;

        mRequestedWidth = 0;
        mRequestedHeight = 0;
        mLastRequestedWidth = 0;
        mLastRequestedHeight = 0;
        mLayer = 0;
        mInputWindowHandle = new InputWindowHandle(
                mAppToken != null ? mAppToken.mInputApplicationHandle : null, this, c,
                    getDisplayId());
    }

```

###### 5.3.1.2、WindowState.attach()
在WMS服务端创建了所需对象后，接着调用了WindowState的attach()来进一步完成窗口添加。
``` java
G:\android9.0\frameworks\base\services\core\java\com\android\server\wm\WindowState.java

----------------------------------------------------------------
对应Log：
04-16 13:08:36.964 V/WindowState( 1049): Attaching Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity} token=AppWindowToken{1579134 token=Token{ce2ac07 ActivityRecord{65cc87a u0 com.android.testred/.TestActivity t5}}}
----------------------------------------------------------------

    void attach() {
        if (localLOGV) Slog.v(TAG, "Attaching " + this + " token=" + mToken);
        mSession.windowAddedLocked(mAttrs.packageName);
    }

```
###### 5.3.1.3、mSession.windowAddedLocked(mAttrs.packageName)

``` java
G:\android9.0\frameworks\base\services\core\java\com\android\server\wm\Session.java

    void windowAddedLocked(String packageName) {
        mPackageName = packageName;
        mRelayoutTag = "relayoutWindow: " + mPackageName;
        if (mSurfaceSession == null) {

----------------------------------------------------------------
对应Log：

04-16 13:08:36.964 V/WindowManager( 1049): First window added to Session{e6e78f6 2602:u0a10088}, creating SurfaceSession
04-16 13:08:36.965 I/WindowManager( 1049):   NEW SURFACE SESSION android.view.SurfaceSession@3812e82
----------------------------------------------------------------
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

###### 5.3.1.4、SurfaceSession建立过程
SurfaceSession对象承担了应用程序与SurfaceFlinger之间的通信过程，每一个需要与SurfaceFlinger进程交互的应用程序端都需要创建一个SurfaceSession对象。

客户端请求

``` java
G:\android9.0\frameworks\base\core\java\android\view\SurfaceSession.java

/**
 * An instance of this class represents a connection to the surface
 * flinger, from which you can create one or more Surface instances that will
 * be composited to the screen.
 * {@hide}
 */
public final class SurfaceSession {
    // Note: This field is accessed by native code.
    private long mNativeClient; // SurfaceComposerClient*
    
    /** Create a new connection with the surface flinger. */
    public SurfaceSession() {
        mNativeClient = nativeCreate();
    }
    ......
}
```
Java层的SurfaceSession对象构造过程会通过JNI在native层创建一个SurfaceComposerClient对象。

``` cpp
G:\android9.0\frameworks\base\core\jni\android_view_SurfaceSession.cpp

static jlong nativeCreate(JNIEnv* env, jclass clazz) {
    SurfaceComposerClient* client = new SurfaceComposerClient();
    client->incStrong((void*)nativeCreate);
    return reinterpret_cast<jlong>(client);
}

```

Java层的SurfaceSession对象与C++层的SurfaceComposerClient对象之间是一对一关系。

``` cpp
G:\android9.0\frameworks\native\libs\gui\SurfaceComposerClient.cpp

SurfaceComposerClient::SurfaceComposerClient()
    : mStatus(NO_INIT)
{
}

void SurfaceComposerClient::onFirstRef() {
    sp<ISurfaceComposer> sf(ComposerService::getComposerService());
    if (sf != 0 && mStatus == NO_INIT) {
        auto rootProducer = mParent.promote();
        sp<ISurfaceComposerClient> conn;
        conn = (rootProducer != nullptr) ? sf->createScopedConnection(rootProducer) :
                sf->createConnection();
        if (conn != 0) {
            mClient = conn;
            mStatus = NO_ERROR;
        }
    }
}
```
SurfaceComposerClient继承于RefBase类，当第一次被强引用时，onFirstRef函数被回调，在该函数中SurfaceComposerClient会请求SurfaceFlinger为当前应用程序创建一个Client对象，专门接收该应用程序的请求，在SurfaceFlinger端创建好Client本地Binder对象后，将该Binder代理对象返回给应用程序端，并保存在SurfaceComposerClient的成员变量mClient中。

服务端处理 在SurfaceFlinger服务端为应用程序创建交互的Client对象 [SurfaceFlinger.cpp]

``` cpp
G:\android9.0\frameworks\native\services\surfaceflinger\SurfaceFlinger.cpp


sp<ISurfaceComposerClient> SurfaceFlinger::createConnection() {
    return initClient(new Client(this));
}

sp<ISurfaceComposerClient> SurfaceFlinger::createScopedConnection(
        const sp<IGraphicBufferProducer>& gbp) {
    if (authenticateSurfaceTexture(gbp) == false) {
        return nullptr;
    }
    const auto& layer = (static_cast<MonitoredProducer*>(gbp.get()))->getLayer();
    if (layer == nullptr) {
        return nullptr;
    }

   return initClient(new Client(this, layer));
}

```
![enter image description here](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/personal.website/zjj.display.sys.SurfaceFlinger-Create-Layer.png) 



###### 5.3.1.5、win.mToken.addWindow(win)

``` java
G:\android9.0\frameworks\base\services\core\java\com\android\server\wm\WindowToken.java
----------------------------------------------------------------
对应Log：
04-16 13:08:36.965 D/WindowManager( 1049): addWindow: win=Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity} Callers=com.android.server.wm.AppWindowToken.addWindow:989 com.android.server.wm.WindowManagerService.addWindow:1426 com.android.server.wm.Session.addToDisplay:205 android.view.IWindowSession$Stub.onTransact:129 com.android.server.wm.Session.onTransact:164 
04-16 13:08:36.965 V/WindowManager( 1049): Adding Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity} to AppWindowToken{1579134 token=Token{ce2ac07 ActivityRecord{65cc87a u0 com.android.testred/.TestActivity t5}}}
----------------------------------------------------------------
    void addWindow(final WindowState win) {
        if (DEBUG_FOCUS) Slog.d(TAG_WM,
                "addWindow: win=" + win + " Callers=" + Debug.getCallers(5));

        if (win.isChildWindow()) {
            // Child windows are added to their parent windows.
            return;
        }
        if (!mChildren.contains(win)) {
            if (DEBUG_ADD_REMOVE) Slog.v(TAG_WM, "Adding " + win + " to " + this);
            addChild(win, mWindowComparator);
            mService.mWindowsChanged = true;
            // TODO: Should we also be setting layout needed here and other places?
        }
    }
```

---
/**************Vsync start**************/
---

####  （二）、Vsync trigger
#####（1）、DisplayEventReceiver::sendEvents
``` cpp
G:\android9.0\frameworks\native\services\surfaceflinger\EventThread.cpp

void EventThread::threadMain() NO_THREAD_SAFETY_ANALYSIS {
    std::unique_lock<std::mutex> lock(mMutex);
    while (mKeepRunning) {
        DisplayEventReceiver::Event event;
        Vector<sp<EventThread::Connection> > signalConnections;
        signalConnections = waitForEventLocked(&lock, &event);

        // dispatch events to listeners...
        const size_t count = signalConnections.size();
        for (size_t i = 0; i < count; i++) {
            const sp<Connection>& conn(signalConnections[i]);
            // now see if we still need to report this event
            status_t err = conn->postEvent(event);
            if (err == -EAGAIN || err == -EWOULDBLOCK) {
                // The destination doesn't accept events anymore, it's probably
                // full. For now, we just drop the events on the floor.
                // FIXME: Note that some events cannot be dropped and would have
                // to be re-sent later.
                // Right-now we don't have the ability to do this.
                ALOGW("EventThread: dropping event (%08x) for connection %p", event.header.type,
                      conn.get());
            } else if (err < 0) {
                // handle any other error on the pipe as fatal. the only
                // reasonable thing to do is to clean-up this connection.
                // The most common error we'll get here is -EPIPE.
                removeDisplayEventConnectionLocked(signalConnections[i]);
            }
        }
    }
}


status_t EventThread::Connection::postEvent(const DisplayEventReceiver::Event& event) {
    ssize_t size = DisplayEventReceiver::sendEvents(&mChannel, &event, 1);
    return size < 0 ? status_t(size) : status_t(NO_ERROR);
}

```

当有Vsync信号来到时，会调用conn->postEvent(event)。

``` cpp
G:\android9.0\frameworks\native\libs\gui\DisplayEventReceiver.cpp

ssize_t DisplayEventReceiver::sendEvents(gui::BitTube* dataChannel,
        Event const* events, size_t count)
{
    return gui::BitTube::sendObjects(dataChannel, events, count);
}


G:\android9.0\frameworks\native\libs\sensor\BitTube.cpp

ssize_t BitTube::sendObjects(const sp<BitTube>& tube,
        void const* events, size_t count, size_t objSize)
{
    const char* vaddr = reinterpret_cast<const char*>(events);
    ssize_t size = tube->write(vaddr, count*objSize);

    // should never happen because of SOCK_SEQPACKET
    LOG_ALWAYS_FATAL_IF((size >= 0) && (size % static_cast<ssize_t>(objSize)),
            "BitTube::sendObjects(count=%zu, size=%zu), res=%zd (partial events were sent!)",
            count, objSize, size);

    //ALOGE_IF(size<0, "error %d sending %d events", size, count);
    return size < 0 ? size : size / static_cast<ssize_t>(objSize);
}

ssize_t BitTube::write(void const* vaddr, size_t size)
{
    ssize_t err, len;
    do {
        len = ::send(mSendFd, vaddr, size, MSG_DONTWAIT | MSG_NOSIGNAL);
        // cannot return less than size, since we're using SOCK_SEQPACKET
        err = len < 0 ? errno : 0;
    } while (err == EINTR);
    return err == 0 ? len : -err;
}


```
从BitTube的初始化函数可以看出使用了套接来通讯

``` cpp
G:\android9.0\frameworks\native\libs\sensor\BitTube.cpp

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


``` cpp
G:\android9.0\frameworks\native\libs\sensor\BitTube.cpp
ssize_t BitTube::recvObjects(BitTube* tube, void* events, size_t count, size_t objSize) {
    char* vaddr = reinterpret_cast<char*>(events);
    ssize_t size = tube->read(vaddr, count * objSize);

    // should never happen because of SOCK_SEQPACKET
    LOG_ALWAYS_FATAL_IF((size >= 0) && (size % static_cast<ssize_t>(objSize)),
                        "BitTube::recvObjects(count=%zu, size=%zu), res=%zd (partial events were "
                        "received!)",
                        count, objSize, size);

    // ALOGE_IF(size<0, "error %d receiving %d events", size, count);
    return size < 0 ? size : size / static_cast<ssize_t>(objSize);
}
```


``` cpp
G:\android9.0\frameworks\native\libs\sensor\BitTube.cpp
ssize_t BitTube::read(void* vaddr, size_t size) {
    ssize_t err, len;
    do {
        len = ::recv(mReceiveFd, vaddr, size, MSG_DONTWAIT);
        err = len < 0 ? errno : 0;
    } while (err == EINTR);
    if (err == EAGAIN || err == EWOULDBLOCK) {
        // EAGAIN means that we have non-blocking I/O but there was no data to be read. Nothing the
        // client should care about.
        return 0;
    }
    return err == 0 ? len : -err;
}
```

#####（2）、DisplayEventDispatcher::handleEvent

``` cpp
G:\android9.0\frameworks\base\libs\androidfw\DisplayEventDispatcher.cpp

DisplayEventDispatcher::DisplayEventDispatcher(const sp<Looper>& looper,
        ISurfaceComposer::VsyncSource vsyncSource) :
        mLooper(looper), mReceiver(vsyncSource), mWaitingForVsync(false) {
    ALOGV("dispatcher %p ~ Initializing display event dispatcher.", this);
}

status_t DisplayEventDispatcher::initialize() {
    status_t result = mReceiver.initCheck();
    if (result) {
        ALOGW("Failed to initialize display event receiver, status=%d", result);
        return result;
    }

    int rc = mLooper->addFd(mReceiver.getFd(), 0, Looper::EVENT_INPUT,
            this, NULL);
    if (rc < 0) {
        return UNKNOWN_ERROR;
    }
    return OK;
}
```
而最终在addFd中注册的回调时this，这个我们之前在分析Looper的时候分析过，也就是会调用这个类的handleEvent函数,在这个函数中调用dispatchVsync继续分发VSync信号。
``` cpp
G:\android9.0\frameworks\base\libs\androidfw\DisplayEventDispatcher.cpp

int DisplayEventDispatcher::handleEvent(int, int events, void*) {
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
        ALOGV("dispatcher %p ~ Vsync pulse: timestamp=%" PRId64 ", id=%d, count=%d",
                this, ns2ms(vsyncTimestamp), vsyncDisplayId, vsyncCount);
        mWaitingForVsync = false;
        dispatchVsync(vsyncTimestamp, vsyncDisplayId, vsyncCount);
    }

    return 1; // keep the callback
}

```


``` cpp
G:\android9.0\frameworks\base\core\jni\android_view_DisplayEventReceiver.cpp

void NativeDisplayEventReceiver::dispatchVsync(nsecs_t timestamp, int32_t id, uint32_t count) {
    JNIEnv* env = AndroidRuntime::getJNIEnv();

    ScopedLocalRef<jobject> receiverObj(env, jniGetReferent(env, mReceiverWeakGlobal));
    if (receiverObj.get()) {
        ALOGV("receiver %p ~ Invoking vsync handler.", this);
        env->CallVoidMethod(receiverObj.get(),
                gDisplayEventReceiverClassInfo.dispatchVsync, timestamp, id, count);
        ALOGV("receiver %p ~ Returned from vsync handler.", this);
    }

    mMessageQueue->raiseAndClearException(env, "dispatchVsync");
}
```
这里是利用jni反调了DisplayEventReceiver.java类的dispatchVsync函数。
#####（3）、DisplayEventReceiver.dispatchVsync()（Java层回调）


我们再来看看DisplayEventReceiver java类的dispatchVsync函数，调用了onVsync函数。

``` java
G:\android9.0\frameworks\base\core\java\android\view\DisplayEventReceiver.java

private void dispatchVsync(long timestampNanos, int builtInDisplayId, int frame) {
    onVsync(timestampNanos, builtInDisplayId, frame);
}
```

而这个onVsync是一个空函数，具体在其子类中实现了。

``` java
G:\android9.0\frameworks\base\core\java\android\view\Choreographer.java
    private final class FrameDisplayEventReceiver extends DisplayEventReceiver
            implements Runnable {
        private boolean mHavePendingVsync;
        private long mTimestampNanos;
        private int mFrame;

        public FrameDisplayEventReceiver(Looper looper, int vsyncSource) {
            super(looper, vsyncSource);
        }

        @Override
        public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
            ......
            if (builtInDisplayId != SurfaceControl.BUILT_IN_DISPLAY_ID_MAIN) {
                ......
                scheduleVsync();
                return;
            }

            ......
            long now = System.nanoTime();
            if (timestampNanos > now) {
                ......
                timestampNanos = now;
            }

            if (mHavePendingVsync) {
               ......
            } else {
                mHavePendingVsync = true;
            }

            mTimestampNanos = timestampNanos;
            mFrame = frame;
            Message msg = Message.obtain(mHandler, this);
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
        }

        @Override
        public void run() {
            mHavePendingVsync = false;
            doFrame(mTimestampNanos, mFrame);
        }
    }
```

Message msg = Message.obtain(mHandler, this)最终会调用run()。

#####（4）、 doFrame(mTimestampNanos, mFrame)

``` java
G:\android9.0\frameworks\base\core\java\android\view\Choreographer.java

    void doFrame(long frameTimeNanos, int frame) {
        final long startNanos;
        synchronized (mLock) {
            
            long intendedFrameTimeNanos = frameTimeNanos;
            startNanos = System.nanoTime();
            final long jitterNanos = startNanos - frameTimeNanos;
            if (jitterNanos >= mFrameIntervalNanos) {
                final long skippedFrames = jitterNanos / mFrameIntervalNanos;
                ......
                final long lastFrameOffset = jitterNanos % mFrameIntervalNanos;
                ......
                frameTimeNanos = startNanos - lastFrameOffset;
            }

            if (frameTimeNanos < mLastFrameTimeNanos) {
                ......
                scheduleVsyncLocked();
                return;
            }

            if (mFPSDivisor > 1) {
                long timeSinceVsync = frameTimeNanos - mLastFrameTimeNanos;
                if (timeSinceVsync < (mFrameIntervalNanos * mFPSDivisor) && timeSinceVsync > 0) {
                    scheduleVsyncLocked();
                    return;
                }
            }

            mFrameInfo.setVsync(intendedFrameTimeNanos, frameTimeNanos);
            mFrameScheduled = false;
            mLastFrameTimeNanos = frameTimeNanos;
        }

        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Choreographer#doFrame");
            AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);

            mFrameInfo.markInputHandlingStart();
            doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);

            mFrameInfo.markAnimationsStart();
            doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);

            mFrameInfo.markPerformTraversalsStart();
            doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);

            doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
        } finally {
            AnimationUtils.unlockAnimationClock();
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
       .....
    }
```

Choreographer类中分别定义了CallbackRecord、CallbackQueue内部类，CallbackQueue是一个按时间先后顺序保存CallbackRecord的单向循环链表。

在Choreographer中定义了三个CallbackQueue队列，用数组mCallbackQueues表示，用于分别保存CALLBACK_INPUT、CALLBACK_ANIMATION、CALLBACK_TRAVERSAL这三种类型的Callback，当调用Choreographer类的postCallback()函数时，就是往指定类型的CallbackQueue队列中通过addCallbackLocked()函数添加一个CallbackRecord项：首先构造一个CallbackRecord对象，然后按时间先后顺序插入到CallbackQueue链表中。从代码注释中，我们可以知道CALLBACK_INPUT是指输入回调，该回调优先级最高，首先得到执行，而CALLBACK_TRAVERSAL是指处理布局和绘图的回调，只有在所有异步消息都执行完后才得到执行，CALLBACK_ANIMATION是指动画回调，比CALLBACK_TRAVERSAL优先执行，从doFrame()函数中的doCallbacks调用就能印证这点。 当Vsync事件到来时，顺序执行CALLBACK_INPUT、CALLBACK_ANIMATION、CALLBACK_TRAVERSAL 和CALLBACK_COMMIT 对应CallbackQueue队列中注册的回调。

关于Choreographer的postCallback()用法在前面进行了详细的介绍，当Vsync事件到来时，mTraversalRunnable对象的run()函数将被调用。

mTraversalRunnable对象的类型为TraversalRunnable，该类实现了Runnable接口，在其run()函数中调用了doTraversal()函数来完成窗口布局。


``` java
G:\android9.0\frameworks\base\core\java\android\view\ViewRootImpl.java

final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}
```

#####（5）、 ViewRootImpl.doTraversal()

``` java
G:\android9.0\frameworks\base\core\java\android\view\ViewRootImpl.java

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



####  （三）、Vsync 信号处理

performTraversals函数相当复杂，其主要实现以下几个重要步骤：

- 1.执行窗口测量；

- 2.执行窗口注册；

- 3.执行窗口布局；

- 4.执行窗口绘图；


``` java
G:\android9.0\frameworks\base\core\java\android\view\ViewRootImpl.java

    private void performTraversals() {
        // cache mView since it is used so much below...
        final View host = mView;


----------------------------------------------------------------
对应Log：
04-16 13:08:36.978 I/System.out( 2602): ======================================
04-16 13:08:36.978 I/System.out( 2602): performTraversals
04-16 13:08:36.978 D/View    ( 2602):   + DecorView@515e1ca[TestActivity]
04-16 13:08:36.978 D/View    ( 2602):       frame={0, 0, 0, 0} scroll={0, 0} 
04-16 13:08:36.978 D/View    ( 2602):       mMeasureWidth=0 mMeasureHeight=0
view.java  ===>>> output = mLayoutParams.debug(output);

04-16 13:08:36.991 D/Debug   ( 2602):       Contents of {(0,0)(fillxfill) sim={forwardNavigation} ty=BASE_APPLICATION fmt=TRANSPARENT wanim=0x1030000
04-16 13:08:36.991 D/Debug   ( 2602):   fl=LAYOUT_IN_SCREEN LAYOUT_INSET_DECOR SPLIT_TOUCH HARDWARE_ACCELERATED}:
04-16 13:08:36.991 D/Debug   ( 2602): ViewGroup.LayoutParams={ width=match-parent, height=match-parent }
04-16 13:08:36.991 D/Debug   ( 2602): 
04-16 13:08:36.991 D/Debug   ( 2602): WindowManager.LayoutParams={title=com.android.testred/com.android.testred.TestActivity}
----------------------------------------------------------------
        if (DBG) {
            System.out.println("======================================");
            System.out.println("performTraversals");
            host.debug();
        }

        if (host == null || !mAdded)
            return;

        mIsInTraversal = true;
        mWillDrawSoon = true;
        boolean windowSizeMayChange = false;
        boolean newSurface = false;
        boolean surfaceChanged = false;
        WindowManager.LayoutParams lp = mWindowAttributes;

        int desiredWindowWidth;
        int desiredWindowHeight;

        final int viewVisibility = getHostVisibility();
        ......
        final boolean viewUserVisibilityChanged = !mFirst &&
                ((mViewVisibility == View.VISIBLE) != (viewVisibility == View.VISIBLE));

        WindowManager.LayoutParams params = null;
        if (mWindowAttributesChanged) {
            mWindowAttributesChanged = false;
            surfaceChanged = true;
            params = lp;
        }
        CompatibilityInfo compatibilityInfo =
                mDisplay.getDisplayAdjustments().getCompatibilityInfo();
        if (compatibilityInfo.supportsScreen() == mLastInCompatMode) {
            params = lp;
            mFullRedrawNeeded = true;
            mLayoutRequested = true;
            if (mLastInCompatMode) {
                params.privateFlags &= ~WindowManager.LayoutParams.PRIVATE_FLAG_COMPATIBLE_WINDOW;
                mLastInCompatMode = false;
            } else {
                params.privateFlags |= WindowManager.LayoutParams.PRIVATE_FLAG_COMPATIBLE_WINDOW;
                mLastInCompatMode = true;
            }
        }

        mWindowAttributesChangesFlag = 0;

        Rect frame = mWinFrame;
        if (mFirst) {
            mFullRedrawNeeded = true;
            mLayoutRequested = true;

            final Configuration config = mContext.getResources().getConfiguration();
            if (shouldUseDisplaySize(lp)) {
                // NOTE -- system code, won't try to do compat mode.
                Point size = new Point();
                mDisplay.getRealSize(size);
                desiredWindowWidth = size.x;
                desiredWindowHeight = size.y;
            } else {
                desiredWindowWidth = mWinFrame.width();
                desiredWindowHeight = mWinFrame.height();
            }

            ......
            // Set the layout direction if it has not been set before (inherit is the default)
            if (mViewLayoutDirectionInitial == View.LAYOUT_DIRECTION_INHERIT) {
                host.setLayoutDirection(config.getLayoutDirection());
            }
            host.dispatchAttachedToWindow(mAttachInfo, 0);
            mAttachInfo.mTreeObserver.dispatchOnWindowAttachedChange(true);
            dispatchApplyInsets(host);
        } else {
            desiredWindowWidth = frame.width();
            desiredWindowHeight = frame.height();
            ......
        }

       ......
        boolean insetsChanged = false;

        boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);
        if (layoutRequested) {

            final Resources res = mView.getContext().getResources();

            if (mFirst) {
                // make sure touch mode code executes by setting cached value
                // to opposite of the added touch mode.
                mAttachInfo.mInTouchMode = !mAddedTouchMode;
                ensureTouchModeLocally(mAddedTouchMode);
            } else {
                ......
                if (!mPendingVisibleInsets.equals(mAttachInfo.mVisibleInsets)) {
                    mAttachInfo.mVisibleInsets.set(mPendingVisibleInsets);
                    if (DEBUG_LAYOUT) Log.v(mTag, "Visible insets changing to: "
                            + mAttachInfo.mVisibleInsets);
                }
                ......
                if (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT
                        || lp.height == ViewGroup.LayoutParams.WRAP_CONTENT) {
                    windowSizeMayChange = true;

                    if (shouldUseDisplaySize(lp)) {
                        // NOTE -- system code, won't try to do compat mode.
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
            windowSizeMayChange |= measureHierarchy(host, lp, res,
                    desiredWindowWidth, desiredWindowHeight);
        }

        if (collectViewAttributes()) {
            params = lp;
        }
        if (mAttachInfo.mForceReportNewAttributes) {
            mAttachInfo.mForceReportNewAttributes = false;
            params = lp;
        }
        ......

/****************执行窗口测量******************/  
        if (mApplyInsetsRequested) {
            mApplyInsetsRequested = false;
            mLastOverscanRequested = mAttachInfo.mOverscanRequested;
            dispatchApplyInsets(host);
            if (mLayoutRequested) {
                // Short-circuit catching a new layout request here, so
                // we don't need to go through two layout passes when things
                // change due to fitting system windows, which can happen a lot.
----------------------------------------------------------------
对应Log：
04-16 13:08:36.985 V/ViewRootImpl[TestActivity]( 2602): Measuring DecorView@515e1ca[TestActivity] in display 480x782...
04-16 13:08:36.988 I/System.out( 2602): ======================================
04-16 13:08:36.988 I/System.out( 2602): performTraversals -- after measure
----------------------------------------------------------------
                windowSizeMayChange |= measureHierarchy(host, lp,
                        mView.getContext().getResources(),
                        desiredWindowWidth, desiredWindowHeight);
            }
        }

        if (layoutRequested) {
            // Clear this now, so that if anything requests a layout in the
            // rest of this function we will catch it and re-run a full
            // layout pass.
            mLayoutRequested = false;
        }

        boolean windowShouldResize ...;
        windowShouldResize |= mDragResizing && mResizeMode == RESIZE_MODE_FREEFORM;

        windowShouldResize |= mActivityRelaunched;

        final boolean computesInternalInsets ...;

        boolean insetsPending = false;
        int relayoutResult = 0;
        boolean updatedConfiguration = false;

        final int surfaceGenerationId = mSurface.getGenerationId();

        final boolean isViewVisible = viewVisibility == View.VISIBLE;
        final boolean windowRelayoutWasForced = mForceNextWindowRelayout;
        if (mFirst || windowShouldResize || insetsChanged ||
                viewVisibilityChanged || params != null || mForceNextWindowRelayout) {
            mForceNextWindowRelayout = false;

            if (isViewVisible) {
                insetsPending = computesInternalInsets && (mFirst || viewVisibilityChanged);
            }

            if (mSurfaceHolder != null) {
                mSurfaceHolder.mSurfaceLock.lock();
                mDrawingAllowed = true;
            }

            boolean hwInitialized = false;
            boolean contentInsetsChanged = false;
            boolean hadSurface = mSurface.isValid();

            try {
----------------------------------------------------------------
对应Log：
04-16 13:08:36.995 I/ViewRootImpl[TestActivity]( 2602): host=w:480, h:782, params={(0,0)(fillxfill) sim={adjust=pan forwardNavigation} ty=BASE_APPLICATION fmt=TRANSPARENT wanim=0x1030000
04-16 13:08:36.995 I/ViewRootImpl[TestActivity]( 2602):   fl=LAYOUT_IN_SCREEN LAYOUT_INSET_DECOR SPLIT_TOUCH HARDWARE_ACCELERATED}
----------------------------------------------------------------
                if (DEBUG_LAYOUT) {
                    Log.i(mTag, "host=w:" + host.getMeasuredWidth() + ", h:" +
                            host.getMeasuredHeight() + ", params=" + params);
                }

                if (mAttachInfo.mThreadedRenderer != null) {
                    if (mAttachInfo.mThreadedRenderer.pauseSurface(mSurface)) {
                        mDirty.set(0, 0, mWidth, mHeight);
                    }
                    mChoreographer.mFrameInfo.addFlags(FrameInfo.FLAG_WINDOW_LAYOUT_CHANGED);
                }
                
 /****************请求WMS服务relayout窗口******************/  
----------------------------------------------------------------
对应Log：
04-16 13:08:36.997 D/ViewRootImpl[TestActivity]( 2602): WindowLayout in layoutWindow:{(0,0)(fillxfill) sim={adjust=pan forwardNavigation} ty=BASE_APPLICATION fmt=TRANSPARENT wanim=0x1030000
04-16 13:08:36.997 D/ViewRootImpl[TestActivity]( 2602):   fl=LAYOUT_IN_SCREEN LAYOUT_INSET_DECOR SPLIT_TOUCH HARDWARE_ACCELERATED}
----------------------------------------------------------------

                relayoutResult = relayoutWindow(params, viewVisibility, insetsPending);

```

##### （1）、请求WMS服务relayout窗口mWindowSession.relayout()

``` java
G:\android9.0\frameworks\base\core\java\android\view\ViewRootImpl.java

   private int relayoutWindow(WindowManager.LayoutParams params, int viewVisibility,
            boolean insetsPending) throws RemoteException {

        float appScale = mAttachInfo.mApplicationScale;
        boolean restore = false;
        if (params != null && mTranslator != null) {
            restore = true;
            params.backup();
            mTranslator.translateWindowLayout(params);
        }

        if (params != null) {
            if (DBG) Log.d(mTag, "WindowLayout in layoutWindow:" + params);

            if (mOrigWindowType != params.type) {
                // For compatibility with old apps, don't crash here.
                if (mTargetSdkVersion < Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
                    Slog.w(mTag, "Window type can not be changed after "
                            + "the window is added; ignoring change of " + mView);
                    params.type = mOrigWindowType;
                }
            }
        }

        long frameNumber = -1;
        if (mSurface.isValid()) {
            frameNumber = mSurface.getNextFrameNumber();
        }

        //charlesvincent start
        int relayoutResult = 0;
        if(params != null ){
                        if("com.android.testred".equals(params.packageName) 
				|| "com.android.testgreen".equals(params.packageName)
				|| "com.android.testblue".equals(params.packageName)){
				Point size = new Point();
				mDisplay.getRealSize(size);
				params.width = (int) size.x / 3;
				params.height = (int) size.y;

                relayoutResult = mWindowSession.relayout(mWindow, mSeq, params,
                (int) size.x / 3,
                (int) size.y, viewVisibility,
                insetsPending ? WindowManagerGlobal.RELAYOUT_INSETS_PENDING : 0, frameNumber,
                mWinFrame, mPendingOverscanInsets, mPendingContentInsets, mPendingVisibleInsets,
                mPendingStableInsets, mPendingOutsets, mPendingBackDropFrame, mPendingDisplayCutout,
                mPendingMergedConfiguration, mSurface);

			}else {
                relayoutResult = mWindowSession.relayout(mWindow, mSeq, params,
                (int) (mView.getMeasuredWidth() * appScale + 0.5f),
                (int) (mView.getMeasuredHeight() * appScale + 0.5f), viewVisibility,
                insetsPending ? WindowManagerGlobal.RELAYOUT_INSETS_PENDING : 0, frameNumber,
                mWinFrame, mPendingOverscanInsets, mPendingContentInsets, mPendingVisibleInsets,
                mPendingStableInsets, mPendingOutsets, mPendingBackDropFrame, mPendingDisplayCutout,
                mPendingMergedConfiguration, mSurface);
			}
        }
        //charlesvincent end
        mPendingAlwaysConsumeNavBar =
                (relayoutResult & WindowManagerGlobal.RELAYOUT_RES_CONSUME_ALWAYS_NAV_BAR) != 0;

        if (restore) {
            params.restore();
        }

        if (mTranslator != null) {
            mTranslator.translateRectInScreenToAppWinFrame(mWinFrame);
            mTranslator.translateRectInScreenToAppWindow(mPendingOverscanInsets);
            mTranslator.translateRectInScreenToAppWindow(mPendingContentInsets);
            mTranslator.translateRectInScreenToAppWindow(mPendingVisibleInsets);
            mTranslator.translateRectInScreenToAppWindow(mPendingStableInsets);
        }
        return relayoutResult;
    }

```
这里通过前面获取的IWindowSession代理对象请求WMS服务执行窗口布局，mSurface是ViewRootImpl的成员变量 [->ViewRootImpl.java]

该Surface构造函数仅仅创建了一个空Surface对象，并没有对该Surface进程native层的初始化，到此我们知道应用程序进程为每个窗口对象都创建了一个Surface对象。并且将该Surface通过跨进程方式传输给WMS服务进程，我们知道，在Android系统中，如果一个对象需要在不同进程间传输，必须实现Parcelable接口，Surface类正好实现了Parcelable接口。ViewRootImpl通过IWindowSession接口请求WMS的完整过程如下：


``` java
Android 9.0已改变没有直接提供Java源码，而是.class文件打包为classes-header.jar和classes.jar。
android\out\target\common\obj\JAVA_LIBRARIES\framework_intermediates\
classes-header.jar
classes.jar解压：
classes/android/view/
    IWindowSession.class
    IWindowSession$Stub.class
    IWindowSession$Stub$Proxy.class

[解压后在线反编译](http://www.javadecompilers.com/)
or
[解压后在线反编译](http://javare.cn/)
-----------------------------------------------------------------------------------------------
IWindowSession.java
public interface IWindowSession extends IInterface {
  ......
   int addToDisplay(IWindow var1, int var2, LayoutParams var3, int var4, int var5, Rect var6, Rect var7, Rect var8, Rect var9, ParcelableWrapper var10, InputChannel var11) throws RemoteException;

   ......

   int relayout(IWindow var1, int var2, LayoutParams var3, int var4, int var5, int var6, int var7, long var8, Rect var10, Rect var11, Rect var12, Rect var13, Rect var14, Rect var15, Rect var16, ParcelableWrapper var17, MergedConfiguration var18, Surface var19) throws RemoteException;

   ......
}

-----------------------------------------------------------------------------------------------
IWindowSession$Stub.java

public abstract class IWindowSession$Stub extends Binder implements IWindowSession {

   private static final String DESCRIPTOR = "android.view.IWindowSession";
   static final int TRANSACTION_addToDisplay = 2;
   static final int TRANSACTION_relayout = 6;
   public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
      String descriptor = "android.view.IWindowSession";
      IWindow _arg0;
      ......
      switch(code) {
      case 2:
         data.enforceInterface(descriptor);
         _arg0 = Stub.asInterface(data.readStrongBinder());
         _arg1 = data.readInt();
         if(0 != data.readInt()) {
            _arg25 = (LayoutParams)LayoutParams.CREATOR.createFromParcel(data);
         } else {
            _arg25 = null;
         }

         _arg3 = data.readInt();
         _arg4 = data.readInt();
         _arg53 = new Rect();
         _arg63 = new Rect();
         Rect _result6 = new Rect();
         Rect _arg81 = new Rect();
         ParcelableWrapper _result8 = new ParcelableWrapper();
         InputChannel _arg91 = new InputChannel();
         int _arg101 = this.addToDisplay(_arg0, _arg1, _arg25, _arg3, _arg4, _arg53, _arg63, _result6, _arg81, _result8, _arg91);
         ......

         return true;
     case 6:
         data.enforceInterface(descriptor);
         _arg0 = Stub.asInterface(data.readStrongBinder());
         _arg1 = data.readInt();
         if(0 != data.readInt()) {
            _arg25 = (LayoutParams)LayoutParams.CREATOR.createFromParcel(data);
         } else {
            _arg25 = null;
         }

        ......
         ParcelableWrapper _arg15 = new ParcelableWrapper();
         MergedConfiguration _arg16 = new MergedConfiguration();
         Surface _arg17 = new Surface();
         int _result2 = this.relayout(_arg0, _arg1, _arg25, _arg3, _arg4, _arg5, _arg62, _result4, _result7, _arg9, _arg10, _arg11, _arg12, _arg13, _arg14, _arg15, _arg16, _arg17);
         ......
         if(_arg17 != null) {
            reply.writeInt(1);
            _arg17.writeToParcel(reply, 1);
         } else {
            reply.writeInt(0);
         }
         return true;
}

```

从该函数的实现可以看出，应用程序进程中创建的Surface对象并没有传递到WMS服务进程，只是读取WMS服务进程返回来的Surface。那么WMS服务进程是如何响应应用程序进程布局请求的呢？

``` java
-----------------------------------------------------------------------------------------------
IWindowSession$Stub$Proxy.java

class IWindowSession$Stub$Proxy
  implements IWindowSession
{
  private IBinder mRemote;
  
  IWindowSession$Stub$Proxy(IBinder remote)
  {
    mRemote = remote;
  }
  
  public IBinder asBinder() {
    return mRemote;
  }
  
  public String getInterfaceDescriptor() {
    return "android.view.IWindowSession";
  }
  public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs, int viewVisibility, int layerStackId, Rect outFrame, Rect outContentInsets, Rect outStableInsets, Rect outOutsets, DisplayCutout.ParcelableWrapper displayCutout, InputChannel outInputChannel) throws RemoteException {
    Parcel _data = Parcel.obtain();
    Parcel _reply = Parcel.obtain();
    try
    {
      _data.writeInterfaceToken("android.view.IWindowSession");
      _data.writeStrongBinder(window != null ? window.asBinder() : null);
      _data.writeInt(seq);
      if (attrs != null) {
        _data.writeInt(1);
        attrs.writeToParcel(_data, 0);
      }
      else {
        _data.writeInt(0);
      }
      _data.writeInt(viewVisibility);
      _data.writeInt(layerStackId);
      mRemote.transact(2, _data, _reply, 0);
      _reply.readException();
      int _result = _reply.readInt();
      ......
    }
    finally {
      _reply.recycle();
      _data.recycle(); }
    int _result;
    return _result;
  }

  public int relayout(IWindow window, int seq, WindowManager.LayoutParams attrs, int requestedWidth, int requestedHeight, int viewVisibility, int flags, long frameNumber, Rect outFrame, Rect outOverscanInsets, Rect outContentInsets, Rect outVisibleInsets, Rect outStableInsets, Rect outOutsets, Rect outBackdropFrame, DisplayCutout.ParcelableWrapper displayCutout, MergedConfiguration outMergedConfiguration, Surface outSurface) throws RemoteException {
    Parcel _data = Parcel.obtain();
    Parcel _reply = Parcel.obtain();
    try
    {
      _data.writeInterfaceToken("android.view.IWindowSession");
      _data.writeStrongBinder(window != null ? window.asBinder() : null);
      _data.writeInt(seq);
      if (attrs != null) {
        _data.writeInt(1);
        attrs.writeToParcel(_data, 0);
      }
      else {
        _data.writeInt(0);
      }
      _data.writeInt(requestedWidth);
      _data.writeInt(requestedHeight);
      _data.writeInt(viewVisibility);
      _data.writeInt(flags);
      _data.writeLong(frameNumber);
      mRemote.transact(6, _data, _reply, 0);
      _reply.readException();
      int _result = _reply.readInt();
      ......
      if (0 != _reply.readInt()) {
        outSurface.readFromParcel(_reply);
      }
    }
    finally {
      _reply.recycle();
      _data.recycle(); }
    int _result;
    return _result;
  }
  
}
-----------------------------------------------------------------------------------------------

```
该函数可以看出，WMS服务在响应应用程序进程请求添加窗口时，首先在当前进程空间创建一个Surface对象，然后调用Session的relayout()函数进一步完成窗口添加过程，最后将WMS服务中创建的Surface返回给应用程序进程。

##### 1.1、Session.relayout()

``` java
G:\android9.0\frameworks\base\services\core\java\com\android\server\wm\Session.java

    @Override
    public int relayout(IWindow window, int seq, WindowManager.LayoutParams attrs,
            int requestedWidth, int requestedHeight, int viewFlags, int flags, long frameNumber,
            Rect outFrame, Rect outOverscanInsets, Rect outContentInsets, Rect outVisibleInsets,
            Rect outStableInsets, Rect outsets, Rect outBackdropFrame,
            DisplayCutout.ParcelableWrapper cutout, MergedConfiguration mergedConfiguration,
            Surface outSurface) {
        if (false) Slog.d(TAG_WM, ">>>>>> ENTERED relayout from "
                + Binder.getCallingPid());
        Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, mRelayoutTag);
        int res = mService.relayoutWindow(this, window, seq, attrs,
                requestedWidth, requestedHeight, viewFlags, flags, frameNumber,
                outFrame, outOverscanInsets, outContentInsets, outVisibleInsets,
                outStableInsets, outsets, outBackdropFrame, cutout,
                mergedConfiguration, outSurface);
        Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
        if (false) Slog.d(TAG_WM, "<<<<<< EXITING relayout to "
                + Binder.getCallingPid());
        return res;
    }

```

##### 1.2、WindowManagerService.relayoutWindow()

``` java
G:\android9.0\frameworks\base\services\core\java\com\android\server\wm\WindowManagerService.java
    public int relayoutWindow(Session session, IWindow client, int seq, LayoutParams attrs,
            int requestedWidth, int requestedHeight, int viewVisibility, int flags,
            long frameNumber, Rect outFrame, Rect outOverscanInsets, Rect outContentInsets,
            Rect outVisibleInsets, Rect outStableInsets, Rect outOutsets, Rect outBackdropFrame,
            DisplayCutout.ParcelableWrapper outCutout, MergedConfiguration mergedConfiguration,
            Surface outSurface) {
        int result = 0;
        boolean configChanged;
        ......
        long origId = Binder.clearCallingIdentity();
        final int displayId;
        synchronized(mWindowMap) {
----------------------------------------------------------------
对应Log：
04-16 13:08:36.998 V/WindowManager( 1049): Looking up client android.os.BinderProxy@b9baf64: Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity}
    final WindowState windowForClientLocked(Session session, IBinder client, boolean throwOnError) {
        WindowState win = mWindowMap.get(client);
        if (localLOGV) Slog.v(TAG_WM, "Looking up client " + client + ": " + win);
        return win;
    }
----------------------------------------------------------------
            WindowState win = windowForClientLocked(session, client, false);
            
            displayId = win.getDisplayId();
            //得到WindowStateAnimator
            WindowStateAnimator winAnimator = win.mWinAnimator;
            ......
            win.setFrameNumber(frameNumber);
            int attrChanges = 0;
            int flagChanges = 0;
            if (attrs != null) {
                mPolicy.adjustWindowParamsLw(win, attrs, hasStatusBarServicePermission);
                ......
            }
----------------------------------------------------------------
对应Log：
04-16 13:08:37.000 V/WindowManager( 1049): Relayout Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity}: viewVisibility=0 req=160x854 {(0,0)(160x854) sim={adjust=pan forwardNavigation} ty=BASE_APPLICATION fmt=TRANSPARENT wanim=0x1030000
04-16 13:08:37.000 V/WindowManager( 1049):   fl=LAYOUT_IN_SCREEN LAYOUT_INSET_DECOR SPLIT_TOUCH HARDWARE_ACCELERATED}

----------------------------------------------------------------

            if (DEBUG_LAYOUT) Slog.v(TAG_WM, "Relayout " + win + ": viewVisibility=" + viewVisibility
                    + " req=" + requestedWidth + "x" + requestedHeight + " " + win.mAttrs);
            ......
            win.setWindowScale(win.mRequestedWidth, win.mRequestedHeight);

            ......

            final int oldVisibility = win.mViewVisibility;

            .......
            win.mViewVisibility = viewVisibility;
            if (DEBUG_SCREEN_ON) {
----------------------------------------------------------------
对应Log：
04-16 13:08:37.001 I/WindowManager( 1049): Relayout Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity}: oldVis=4 newVis=0
04-16 13:08:37.001 I/WindowManager( 1049): java.lang.RuntimeException
04-16 13:08:37.001 I/WindowManager( 1049): 	at com.android.server.wm.WindowManagerService.relayoutWindow(WindowManagerService.java:1994)
04-16 13:08:37.001 I/WindowManager( 1049): 	at com.android.server.wm.Session.relayout(Session.java:244)
04-16 13:08:37.001 I/WindowManager( 1049): 	at android.view.IWindowSession$Stub.onTransact(IWindowSession.java:309)
04-16 13:08:37.001 I/WindowManager( 1049): 	at com.android.server.wm.Session.onTransact(Session.java:164)
04-16 13:08:37.001 I/WindowManager( 1049): 	at android.os.Binder.execTransact(Binder.java:731)

----------------------------------------------------------------
                RuntimeException stack = new RuntimeException();
                stack.fillInStackTrace();
                Slog.i(TAG_WM, "Relayout " + win + ": oldVis=" + oldVisibility
                        + " newVis=" + viewVisibility, stack);
            }

----------------------------------------------------------------
对应Log：
04-16 13:08:37.001 W/WindowManager( 1049): setLayoutNeeded: callers=com.android.server.wm.WindowState.setDisplayLayoutNeeded:2250 com.android.server.wm.WindowManagerService.relayoutWindow:1999 com.android.server.wm.Session.relayout:244 


G:\android9.0\frameworks\base\services\core\java\com\android\server\wm\WindowState.java
    void setLayoutNeeded() {
        if (DEBUG_LAYOUT) Slog.w(TAG_WM, "setLayoutNeeded: callers=" + Debug.getCallers(3));
        mLayoutNeeded = true;
    }
----------------------------------------------------------------



            win.setDisplayLayoutNeeded();
            win.mGivenInsetsPending = (flags & WindowManagerGlobal.RELAYOUT_INSETS_PENDING) != 0;

            ......
            // We may be deferring layout passes at the moment, but since the client is interested
            // in the new out values right now we need to force a layout.
            mWindowPlacerLocked.performSurfacePlacement(true /* force */);

            if (shouldRelayout) {
               
                result = win.relayoutVisibleWindow(result, attrChanges, oldVisibility);

                try {
                    result = createSurfaceControl(outSurface, result, win, winAnimator);
                } catch (Exception e) {
                    ......
                    return 0;
                }
            } else {
                Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, "relayoutWindow: viewVisibility_2");

                winAnimator.mEnterAnimationPending = false;
                winAnimator.mEnteringAnimation = false;

                if (viewVisibility == View.VISIBLE && winAnimator.hasSurface()) {
                    winAnimator.mSurfaceController.getSurface(outSurface);
                } else {
                    if (DEBUG_VISIBILITY) Slog.i(TAG_WM, "Releasing surface in: " + win);
                    try {
                        outSurface.release();
                    } finally {
                        Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
                    }
                }

                Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
            }

            if (focusMayChange) {
                //System.out.println("Focus may change: " + win.mAttrs.getTitle());
                //更新Focus窗口
                if (updateFocusedWindowLocked(UPDATE_FOCUS_WILL_PLACE_SURFACES,
                        false /*updateInputWindows*/)) {
                    imMayMove = false;
                }
            }

            // updateFocusedWindowLocked() already assigned layers so we only need to
            // reassign them at this point if the IM window state gets shuffled
            boolean toBeDisplayed = (result & WindowManagerGlobal.RELAYOUT_RES_FIRST_TIME) != 0;
            final DisplayContent dc = win.getDisplayContent();
            if (imMayMove) {
                dc.computeImeTarget(true /* updateImeTarget */);
                if (toBeDisplayed) {
                    dc.assignWindowLayers(false /* setLayoutNeeded */);
                }
            }

           .......
----------------------------------------------------------------
对应Log：
Line 3541: 04-16 13:08:37.071 V/WindowManager( 1049): Relayout given client android.os.BinderProxy@b9baf64, requestedWidth=160, requestedHeight=854, viewVisibility=0
Line 3542: 04-16 13:08:37.071 V/WindowManager( 1049): Relayout returning frame=Rect(0, 0 - 160, 854), surface=Surface(name=null)/@0xc820bda
Line 3543: 04-16 13:08:37.071 V/WindowManager( 1049): Relayout of Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity}: focusMayChange=true
----------------------------------------------------------------
            if (localLOGV) Slog.v(
                TAG_WM, "Relayout given client " + client.asBinder()
                + ", requestedWidth=" + requestedWidth
                + ", requestedHeight=" + requestedHeight
                + ", viewVisibility=" + viewVisibility
                + "\nRelayout returning frame=" + outFrame
                + ", surface=" + outSurface);

            if (localLOGV || DEBUG_FOCUS) Slog.v(
                TAG_WM, "Relayout of " + win + ": focusMayChange=" + focusMayChange);

            result |= mInTouchMode ? WindowManagerGlobal.RELAYOUT_RES_IN_TOUCH_MODE : 0;

            mInputMonitor.updateInputWindowsLw(true /*force*/);

            if (DEBUG_LAYOUT) {
                Slog.v(TAG_WM, "Relayout complete " + win + ": outFrame=" + outFrame.toShortString());
            }
            win.mInRelayout = false;
        }
        ......
        return result;
    }
```

##### 1.2.1、WindowSurfacePlacer.performSurfacePlaRelayout complete cement(true /* force */)
首先看下堆栈信息：
``` cpp

04-16 13:08:37.001 V/RootWindowContainer( 1049): performSurfacePlacementInner: entry. Called by com.android.server.wm.WindowSurfacePlacer.performSurfacePlacementLoop:207 com.android.server.wm.WindowSurfacePlacer.performSurfacePlacement:155 com.android.server.wm.WindowManagerService.relayoutWindow:2031 
```

``` java
G:\android9.0\frameworks\base\services\core\java\com\android\server\wm\WindowSurfacePlacer.java

    final void performSurfacePlacement(boolean force) {
        if (mDeferDepth > 0 && !force) {
            return;
        }
        int loopCount = 6;
        do {
            mTraversalScheduled = false;
            performSurfacePlacementLoop();
            mService.mAnimationHandler.removeCallbacks(mPerformSurfacePlacement);
            loopCount--;
        } while (mTraversalScheduled && loopCount > 0);
        mService.mRoot.mWallpaperActionPending = false;
    }


    private void performSurfacePlacementLoop() {
        ......
        mInLayout = true;

        boolean recoveringMemory = false;
        ......

        try {
            mService.mRoot.performSurfacePlacement(recoveringMemory);

            mInLayout = false;

            if (mService.mRoot.isLayoutNeeded()) {
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
            Slog.wtf(TAG, "Unhandled exception while laying out windows", e);
        }

    }

```

##### 1.2.2、RootWindowContainer.performSurfacePlacement(recoveringMemory)

``` java
G:\android9.0\frameworks\base\services\core\java\com\android\server\wm\RootWindowContainer.java

    void performSurfacePlacement(boolean recoveringMemory) {
        if (DEBUG_WINDOW_TRACE) Slog.v(TAG, "performSurfacePlacementInner: entry. Called by "
                + Debug.getCallers(3));

        int i;
        boolean updateInputWindowsNeeded = false;

        if (mService.mFocusMayChange) {
            mService.mFocusMayChange = false;
            updateInputWindowsNeeded = mService.updateFocusedWindowLocked(
                    UPDATE_FOCUS_WILL_PLACE_SURFACES, false /*updateInputWindows*/);
        }

        // Initialize state of exiting tokens.
        final int numDisplays = mChildren.size();
        for (int displayNdx = 0; displayNdx < numDisplays; ++displayNdx) {
            final DisplayContent displayContent = mChildren.get(displayNdx);
            displayContent.setExitingTokensHasVisible(false);
        }

        mHoldScreen = null;
        mScreenBrightness = -1;
        mUserActivityTimeout = -1;
        mObscureApplicationContentOnSecondaryDisplays = false;
        mSustainedPerformanceModeCurrent = false;
        mService.mTransactionSequence++;

        final DisplayContent defaultDisplay = mService.getDefaultDisplayContentLocked();
        final DisplayInfo defaultInfo = defaultDisplay.getDisplayInfo();
        final int defaultDw = defaultInfo.logicalWidth;
        final int defaultDh = defaultInfo.logicalHeight;
----------------------------------------------------------------
对应Log：
04-16 13:08:37.001 I/RootWindowContainer( 1049): >>> OPEN TRANSACTION performLayoutAndPlaceSurfaces
04-16 13:08:37.001 W/WindowManager( 1049): clearLayoutNeeded: callers=com.android.server.wm.DisplayContent.performLayout:2967 com.android.server.wm.DisplayContent.applySurfaceChangesTransaction:2880 com.android.server.wm.RootWindowContainer.applySurfaceChangesTransaction:852 
......
04-16 13:08:37.009 I/RootWindowContainer( 1049): <<< CLOSE TRANSACTION performLayoutAndPlaceSurfaces
----------------------------------------------------------------


        if (SHOW_LIGHT_TRANSACTIONS) Slog.i(TAG,
                ">>> OPEN TRANSACTION performLayoutAndPlaceSurfaces");
        mService.openSurfaceTransaction();
        try {
            applySurfaceChangesTransaction(recoveringMemory, defaultDw, defaultDh);
        } catch (RuntimeException e) {
            Slog.wtf(TAG, "Unhandled exception in Window Manager", e);
        } finally {
            mService.closeSurfaceTransaction("performLayoutAndPlaceSurfaces");
            if (SHOW_LIGHT_TRANSACTIONS) Slog.i(TAG,
                    "<<< CLOSE TRANSACTION performLayoutAndPlaceSurfaces");
        }

        mService.mAnimator.executeAfterPrepareSurfacesRunnables();

        final WindowSurfacePlacer surfacePlacer = mService.mWindowPlacerLocked;
        // If we are ready to perform an app transition, check through all of the app tokens to be
        // shown and see if they are ready to go.
        if (mService.mAppTransition.isReady()) {
            // This needs to be split into two expressions, as handleAppTransitionReadyLocked may
            // modify dc.pendingLayoutChanges, which would get lost when writing
            // defaultDisplay.pendingLayoutChanges |= handleAppTransitionReadyLocked()
            final int layoutChanges = surfacePlacer.handleAppTransitionReadyLocked();
            defaultDisplay.pendingLayoutChanges |= layoutChanges;
            if (DEBUG_LAYOUT_REPEATS)
                surfacePlacer.debugLayoutRepeats("after handleAppTransitionReadyLocked",
                        defaultDisplay.pendingLayoutChanges);
        }

        ......
        // Check to see if we are now in a state where the screen should
        // be enabled, because the window obscured flags have changed.
        mService.enableScreenIfNeededLocked();

        mService.scheduleAnimationLocked();

        if (DEBUG_WINDOW_TRACE) Slog.e(TAG,
                "performSurfacePlacementInner exit: animating=" + mService.mAnimator.isAnimating());
    }
```


###### 1.2.2.1、applySurfaceChangesTransaction(recoveringMemory, defaultDw, defaultDh)


``` java
G:\android9.0\frameworks\base\services\core\java\com\android\server\wm\RootWindowContainer.java

    private void applySurfaceChangesTransaction(boolean recoveringMemory, int defaultDw,
            int defaultDh) {
        mHoldScreenWindow = null;
        mObscuringWindow = null;

         ......
  
        boolean focusDisplayed = false;

        final int count = mChildren.size();
        for (int j = 0; j < count; ++j) {
            final DisplayContent dc = mChildren.get(j);
            focusDisplayed |= dc.applySurfaceChangesTransaction(recoveringMemory);
        }

        if (focusDisplayed) {
            mService.mH.sendEmptyMessage(REPORT_LOSING_FOCUS);
        }
        mService.mDisplayManagerInternal.performTraversal(mDisplayTransaction);
        SurfaceControl.mergeToGlobalTransaction(mDisplayTransaction);
    }
```

###### 1.2.2.2、DisplayContent.applySurfaceChangesTransaction(recoveringMemory)


``` java
G:\android9.0\frameworks\base\services\core\java\com\android\server\wm\DisplayContent.java

    boolean applySurfaceChangesTransaction(boolean recoveringMemory) {

        final int dw = mDisplayInfo.logicalWidth;
        final int dh = mDisplayInfo.logicalHeight;
        final WindowSurfacePlacer surfacePlacer = mService.mWindowPlacerLocked;

        mTmpUpdateAllDrawn.clear();

        int repeats = 0;
        do {
            ......

            if ((pendingLayoutChanges & FINISH_LAYOUT_REDO_LAYOUT) != 0) {
                setLayoutNeeded();
            }

            // FIRST LOOP: Perform a layout, if needed.
            if (repeats < LAYOUT_REPEAT_THRESHOLD) {
                performLayout(repeats == 1, false /* updateInputWindows */);
            } else {
                Slog.w(TAG, "Layout repeat skipped after too many iterations");
            }

            // FIRST AND ONE HALF LOOP: Make WindowManagerPolicy think it is animating.
            pendingLayoutChanges = 0;

            if (isDefaultDisplay) {
                mService.mPolicy.beginPostLayoutPolicyLw(dw, dh);
----------------------------------------------------------------
对应Log：

04-16 13:08:37.003 I/WindowManager( 1049): Win Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity}: affectsSystemUi=false
......
----------------------------------------------------------------

                forAllWindows(mApplyPostLayoutPolicy, true /* traverseTopToBottom */);
                pendingLayoutChanges |= mService.mPolicy.finishPostLayoutPolicyLw();
                if (DEBUG_LAYOUT_REPEATS) surfacePlacer.debugLayoutRepeats(
                        "after finishPostLayoutPolicyLw", pendingLayoutChanges);
            }
        } while (pendingLayoutChanges != 0);
        mTmpApplySurfaceChangesTransactionState.reset();
        resetDimming();

        mTmpRecoveringMemory = recoveringMemory;
----------------------------------------------------------------
对应Log：
04-16 13:08:37.007 V/AppWindowToken( 1049): Eval win Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity}: isDrawn=false, isAnimationSet=true
04-16 13:08:37.007 V/AppWindowToken( 1049): Not displayed: s=null pv=true mDrawState=NO_SURFACE ph=false th=false a=true
----------------------------------------------------------------
        forAllWindows(mApplySurfaceChangesTransaction, true /* traverseTopToBottom */);
        ......

        return mTmpApplySurfaceChangesTransactionState.focusDisplayed;
    }
```

###### 1.2.2.3、DisplayContent.performLayout()

``` java
G:\android9.0\frameworks\base\services\core\java\com\android\server\wm\DisplayContent.java

    void performLayout(boolean initial, boolean updateInputWindows) {
        if (!isLayoutNeeded()) {
            return;
        }
        clearLayoutNeeded();

        final int dw = mDisplayInfo.logicalWidth;
        final int dh = mDisplayInfo.logicalHeight;
----------------------------------------------------------------
对应Log：
04-16 13:08:37.001 V/DisplayContent( 1049): -------------------------------------
04-16 13:08:37.001 V/DisplayContent( 1049): performLayout: needed=false dw=480 dh=854
----------------------------------------------------------------
        if (DEBUG_LAYOUT) {
            Slog.v(TAG, "-------------------------------------");
            Slog.v(TAG, "performLayout: needed=" + isLayoutNeeded() + " dw=" + dw + " dh=" + dh);
        }

        mDisplayFrames.onDisplayInfoUpdated(mDisplayInfo,
                calculateDisplayCutoutForRotation(mDisplayInfo.rotation));
        // TODO: Not sure if we really need to set the rotation here since we are updating from the
        // display info above...
        mDisplayFrames.mRotation = mRotation;
        //处理NavigationBar StatusBar窗口布局
        mService.mPolicy.beginLayoutLw(mDisplayFrames, getConfiguration().uiMode);
        if (isDefaultDisplay) {
            // Not needed on non-default displays.
            mService.mSystemDecorLayer = mService.mPolicy.getSystemDecorLayerLw();
            mService.mScreenRect.set(0, 0, dw, dh);
        }

        int seq = mLayoutSeq + 1;
        if (seq < 0) seq = 0;
        mLayoutSeq = seq;

        // Used to indicate that we have processed the dream window and all additional windows are
        // behind it.
        mTmpWindow = null;
        mTmpInitial = initial;

        // First perform layout of any root windows (not attached to another window).
        forAllWindows(mPerformLayout, true /* traverseTopToBottom */);

        // Used to indicate that we have processed the dream window and all additional attached
        // windows are behind it.
        mTmpWindow2 = mTmpWindow;
        mTmpWindow = null;

        // Now perform layout of attached windows, which usually depend on the position of the
        // window they are attached to. XXX does not deal with windows that are attached to windows
        // that are themselves attached.
        forAllWindows(mPerformLayoutAttached, true /* traverseTopToBottom */);

        // Window frames may have changed. Tell the input dispatcher about it.
        mService.mInputMonitor.layoutInputConsumers(dw, dh);
        mService.mInputMonitor.setUpdateInputWindowsNeededLw();
        if (updateInputWindows) {
            mService.mInputMonitor.updateInputWindowsLw(false /*force*/);
        }

        mService.mH.sendEmptyMessage(UPDATE_DOCKED_STACK_DIVIDER);
    }
```

###### 1.2.2.4、DisplayContent.forAllWindows(mPerformLayout, ...)

``` java
G:\android9.0\frameworks\base\services\core\java\com\android\server\wm\DisplayContent.java


    private final Consumer<WindowState> mPerformLayout = w -> {
        // Don't do layout of a window if it is not visible, or soon won't be visible, to avoid
        // wasting time and funky changes while a window is animating away.
        final boolean gone = (mTmpWindow != null && mService.mPolicy.canBeHiddenByKeyguardLw(w))
                || w.isGoneForLayoutLw();

----------------------------------------------------------------
对应Log：
......
04-16 13:08:37.002 V/DisplayContent( 1049): 1ST PASS Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity}: gone=false mHaveFrame=false mLayoutAttached=false screen changed=true
04-16 13:08:37.002 V/DisplayContent( 1049):   VIS: mViewVisibility=0 mRelayoutCalled=true hidden=true hiddenRequested=false parentHidden=false
......
----------------------------------------------------------------
        if (DEBUG_LAYOUT && !w.mLayoutAttached) {
            Slog.v(TAG, "1ST PASS " + w + ": gone=" + gone + " mHaveFrame=" + w.mHaveFrame
                    + " mLayoutAttached=" + w.mLayoutAttached
                    + " screen changed=" + w.isConfigChanged());
            final AppWindowToken atoken = w.mAppToken;
            if (gone) Slog.v(TAG, "  GONE: mViewVisibility=" + w.mViewVisibility
                    + " mRelayoutCalled=" + w.mRelayoutCalled + " hidden=" + w.mToken.isHidden()
                    + " hiddenRequested=" + (atoken != null && atoken.hiddenRequested)
                    + " parentHidden=" + w.isParentWindowHidden());
            else Slog.v(TAG, "  VIS: mViewVisibility=" + w.mViewVisibility
                    + " mRelayoutCalled=" + w.mRelayoutCalled + " hidden=" + w.mToken.isHidden()
                    + " hiddenRequested=" + (atoken != null && atoken.hiddenRequested)
                    + " parentHidden=" + w.isParentWindowHidden());
        }

        // If this view is GONE, then skip it -- keep the current frame, and let the caller know
        // so they can ignore it if they want.  (We do the normal layout for INVISIBLE windows,
        // since that means "perform layout as normal, just don't display").
        if (!gone || !w.mHaveFrame || w.mLayoutNeeded
                || ((w.isConfigChanged() || w.setReportResizeHints())
                && !w.isGoneForLayoutLw() &&
                ((w.mAttrs.privateFlags & PRIVATE_FLAG_KEYGUARD) != 0 ||
                        (w.mHasSurface && w.mAppToken != null &&
                                w.mAppToken.layoutConfigChanges)))) {
            if (!w.mLayoutAttached) {
                if (mTmpInitial) {
                    //Slog.i(TAG, "Window " + this + " clearing mContentChanged - initial");
                    w.mContentChanged = false;
                }
                if (w.mAttrs.type == TYPE_DREAM) {
                    // Don't layout windows behind a dream, so that if it does stuff like hide
                    // the status bar we won't get a bad transition when it goes away.
                    mTmpWindow = w;
                }
                w.mLayoutNeeded = false;
                w.prelayout();
                final boolean firstLayout = !w.isLaidOut();
                mService.mPolicy.layoutWindowLw(w, null, mDisplayFrames);
                w.mLayoutSeq = mLayoutSeq;

                // If this is the first layout, we need to initialize the last inset values as
                // otherwise we'd immediately cause an unnecessary resize.
                if (firstLayout) {
                    w.updateLastInsetValues();
                }

                if (w.mAppToken != null) {
                    w.mAppToken.layoutLetterbox(w);
                }
----------------------------------------------------------------
对应Log：
04-16 13:08:37.002 V/DisplayContent( 1049):   LAYOUT: mFrame=Rect(0, 0 - 160, 854) mContainingFrame=Rect(0, 0 - 160, 854) mDisplayFrame=Rect(0, 0 - 160, 854)
----------------------------------------------------------------

                if (DEBUG_LAYOUT) Slog.v(TAG, "  LAYOUT: mFrame=" + w.mFrame
                        + " mContainingFrame=" + w.mContainingFrame
                        + " mDisplayFrame=" + w.mDisplayFrame);
            }
        }
    };
```

###### 1.2.2.5、mService.mPolicy[PhoneWindowManager].layoutWindowLw(w, null, mDisplayFrames)

由之前加入的patch可以知道，我们将App（com.android.testred）的Window宽度强制设置为1/3屏幕宽度，即160。
``` java
    @Override
    public void layoutWindowLw(WindowState win, WindowState attached, DisplayFrames displayFrames) {
        // We've already done the navigation bar, status bar, and all screen decor windows. If the
        // status bar can receive input, we need to layout it again to accommodate for the IME
        // window.
        if ((win == mStatusBar && !canReceiveInput(win)) || win == mNavigationBar
                || mScreenDecorWindows.contains(win)) {
            return;
        }
        final WindowManager.LayoutParams attrs = win.getAttrs();
        final boolean isDefaultDisplay = win.isDefaultDisplay();
        final boolean needsToOffsetInputMethodTarget = isDefaultDisplay &&
                (win == mLastInputMethodTargetWindow && mLastInputMethodWindow != null);
        if (needsToOffsetInputMethodTarget) {
            if (DEBUG_LAYOUT) Slog.i(TAG, "Offset ime target window by the last ime window state");
            offsetInputMethodWindowLw(mLastInputMethodWindow, displayFrames);
        }

        final int type = attrs.type;
        final int fl = PolicyControl.getWindowFlags(win, attrs);
        final int pfl = attrs.privateFlags;
        final int sim = attrs.softInputMode;
        final int requestedSysUiFl = PolicyControl.getSystemUiVisibility(null, attrs);
        final int sysUiFl = requestedSysUiFl | getImpliedSysUiFlagsForLayout(attrs);

        final Rect pf = mTmpParentFrame;
        final Rect df = mTmpDisplayFrame;
        final Rect of = mTmpOverscanFrame;
        final Rect cf = mTmpContentFrame;
        final Rect vf = mTmpVisibleFrame;
        final Rect dcf = mTmpDecorFrame;
        final Rect sf = mTmpStableFrame;
        Rect osf = null;
        dcf.setEmpty();
        .......
        if (layoutInScreen && layoutInsetDecor) {
            if (DEBUG_LAYOUT) Slog.v(TAG, "layoutWindowLw(" + attrs.getTitle()
                        + "): IN_SCREEN, INSET_DECOR");
            ......
        }
        .....
        //charlesvincent
        if ("com.android.testred".equals(win.getAttrs().packageName)) {
		   pf.left = df.left = of.left = cf.left = displayFrames.mOverscan.left;
		   pf.top = df.top = of.top = cf.top = displayFrames.mOverscan.top;
		   pf.right = df.right = of.right = cf.right
				   = displayFrames.mOverscan.left + displayFrames.mOverscan.right / 3;
		   pf.bottom = df.bottom = of.bottom = cf.bottom
				   = displayFrames.mOverscan.top + displayFrames.mOverscan.bottom;
        }

        //charlesvincent
        if ("com.android.testgreen".equals(win.getAttrs().packageName)) {
		   pf.left = df.left = of.left = cf.left = displayFrames.mOverscan.left  + displayFrames.mOverscan.right / 3;
		   pf.top = df.top = of.top = cf.top = displayFrames.mOverscan.top;
		   pf.right = df.right = of.right = cf.right
				   = displayFrames.mOverscan.left  + displayFrames.mOverscan.right * 2 / 3;
		   pf.bottom = df.bottom = of.bottom = cf.bottom
				   = displayFrames.mOverscan.top + displayFrames.mOverscan.bottom;
        }

        //charlesvincent
        if ("com.android.testblue".equals(win.getAttrs().packageName)) {
		   pf.left = df.left = of.left = cf.left = displayFrames.mOverscan.left  + displayFrames.mOverscan.right * 2 / 3;
		   pf.top = df.top = of.top = cf.top = displayFrames.mOverscan.top;
		   pf.right = df.right = of.right = cf.right
				   = displayFrames.mOverscan.left  + displayFrames.mOverscan.right * 3 / 3;
		   pf.bottom = df.bottom = of.bottom = cf.bottom
				   = displayFrames.mOverscan.top + displayFrames.mOverscan.bottom;
        }

----------------------------------------------------------------
对应Log：
......
04-16 13:08:37.002 V/WindowManager( 1049): layoutWindowLw(com.android.testred/com.android.testred.TestActivity): IN_SCREEN, INSET_DECOR
04-16 13:08:37.002 V/WindowManager( 1049): Compute frame com.android.testred/com.android.testred.TestActivity: sim=#120 attach=null type=1 flags=0x01810100 pf=[0,0][160,854] df=[0,0][160,854] of=[0,0][160,854] cf=[0,0][160,854] vf=[0,36][480,782] dcf=[0,36][480,782] sf=[0,36][480,782] osf=null


----------------------------------------------------------------
        if (DEBUG_LAYOUT) Slog.v(TAG, "Compute frame " + attrs.getTitle()
                + ": sim=#" + Integer.toHexString(sim)
                + " attach=" + attached + " type=" + type
                + String.format(" flags=0x%08x", fl)
                + " pf=" + pf.toShortString() + " df=" + df.toShortString()
                + " of=" + of.toShortString()
                + " cf=" + cf.toShortString() + " vf=" + vf.toShortString()
                + " dcf=" + dcf.toShortString()
                + " sf=" + sf.toShortString()
                + " osf=" + (osf == null ? "null" : osf.toShortString()));

        win.computeFrameLw(pf, df, of, cf, vf, dcf, sf, osf, displayFrames.mDisplayCutout,
                parentFrameWasClippedByDisplayCutout);
        // Dock windows carve out the bottom of the screen, so normal windows
        // can't appear underneath them.
        if (type == TYPE_INPUT_METHOD && win.isVisibleLw()
                && !win.getGivenInsetsPendingLw()) {
            setLastInputMethodWindowLw(null, null);
            offsetInputMethodWindowLw(win, displayFrames);
        }
        if (type == TYPE_VOICE_INTERACTION && win.isVisibleLw()
                && !win.getGivenInsetsPendingLw()) {
            offsetVoiceInputWindowLw(win, displayFrames);
        }
    }

```

###### 1.2.2.5、WindowState.computeFrameLw()
WindowState大小复杂计算。
``` java
    @Override
    public void computeFrameLw(Rect parentFrame, Rect displayFrame, Rect overscanFrame,
            Rect contentFrame, Rect visibleFrame, Rect decorFrame, Rect stableFrame,
            Rect outsetFrame, WmDisplayCutout displayCutout,
            boolean parentFrameWasClippedByDisplayCutout) {
        ......
        mHaveFrame = true;
        mParentFrameWasClippedByDisplayCutout = parentFrameWasClippedByDisplayCutout;

        final Task task = getTask();
        final boolean inFullscreenContainer = inFullscreenContainer();
        final boolean windowsAreFloating = task != null && task.isFloating();
        final DisplayContent dc = getDisplayContent();

        ......

        final int pw = mContainingFrame.width();
        final int ph = mContainingFrame.height();
        ......

        mOverscanFrame.set(overscanFrame);
        mContentFrame.set(contentFrame);
        mVisibleFrame.set(visibleFrame);
        mDecorFrame.set(decorFrame);
        mStableFrame.set(stableFrame);
        final boolean hasOutsets = outsetFrame != null;
        if (hasOutsets) {
            mOutsetFrame.set(outsetFrame);
        }

        final int fw = mFrame.width();
        final int fh = mFrame.height();

        applyGravityAndUpdateFrame(layoutContainingFrame, layoutDisplayFrame);
        ......
        if (windowsAreFloating && !mFrame.isEmpty()) {
            ......
        } else {
            mContentFrame.set(Math.max(mContentFrame.left, mFrame.left),
                    Math.max(mContentFrame.top, mFrame.top),
                    Math.min(mContentFrame.right, mFrame.right),
                    Math.min(mContentFrame.bottom, mFrame.bottom));

            mVisibleFrame.set(Math.max(mVisibleFrame.left, mFrame.left),
                    Math.max(mVisibleFrame.top, mFrame.top),
                    Math.min(mVisibleFrame.right, mFrame.right),
                    Math.min(mVisibleFrame.bottom, mFrame.bottom));

            mStableFrame.set(Math.max(mStableFrame.left, mFrame.left),
                    Math.max(mStableFrame.top, mFrame.top),
                    Math.min(mStableFrame.right, mFrame.right),
                    Math.min(mStableFrame.bottom, mFrame.bottom));
        }

        ......
        if (mAttrs.type == TYPE_DOCK_DIVIDER) {
            ......
        } else {
            getDisplayContent().getBounds(mTmpRect);
            // Override right and/or bottom insets in case if the frame doesn't fit the screen in
            // non-fullscreen mode.
            boolean overrideRightInset = !windowsAreFloating && !inFullscreenContainer
                    && mFrame.right > mTmpRect.right;
            boolean overrideBottomInset = !windowsAreFloating && !inFullscreenContainer
                    && mFrame.bottom > mTmpRect.bottom;
            mContentInsets.set(mContentFrame.left - mFrame.left,
                    mContentFrame.top - mFrame.top,
                    overrideRightInset ? mTmpRect.right - mContentFrame.right
                            : mFrame.right - mContentFrame.right,
                    overrideBottomInset ? mTmpRect.bottom - mContentFrame.bottom
                            : mFrame.bottom - mContentFrame.bottom);

            mVisibleInsets.set(mVisibleFrame.left - mFrame.left,
                    mVisibleFrame.top - mFrame.top,
                    overrideRightInset ? mTmpRect.right - mVisibleFrame.right
                            : mFrame.right - mVisibleFrame.right,
                    overrideBottomInset ? mTmpRect.bottom - mVisibleFrame.bottom
                            : mFrame.bottom - mVisibleFrame.bottom);

            mStableInsets.set(Math.max(mStableFrame.left - mFrame.left, 0),
                    Math.max(mStableFrame.top - mFrame.top, 0),
                    overrideRightInset ? Math.max(mTmpRect.right - mStableFrame.right, 0)
                            : Math.max(mFrame.right - mStableFrame.right, 0),
                    overrideBottomInset ? Math.max(mTmpRect.bottom - mStableFrame.bottom, 0)
                            :  Math.max(mFrame.bottom - mStableFrame.bottom, 0));
        }

        mDisplayCutout = displayCutout.calculateRelativeTo(mFrame);

        // Offset the actual frame by the amount layout frame is off.
        mFrame.offset(-layoutXDiff, -layoutYDiff);
        mCompatFrame.offset(-layoutXDiff, -layoutYDiff);
        mContentFrame.offset(-layoutXDiff, -layoutYDiff);
        mVisibleFrame.offset(-layoutXDiff, -layoutYDiff);
        mStableFrame.offset(-layoutXDiff, -layoutYDiff);

        mCompatFrame.set(mFrame);
        ......
----------------------------------------------------------------
对应Log：
04-16 13:08:37.002 V/WindowState( 1049): Resolving (mRequestedWidth=160, mRequestedheight=854) to (pw=160, ph=854): frame=[0,0][160,854] ci=[0,0][0,0] vi=[0,36][0,72] si=[0,36][0,72] of=[0,0][0,0]

----------------------------------------------------------------
        if (DEBUG_LAYOUT || localLOGV) Slog.v(TAG,
                "Resolving (mRequestedWidth="
                + mRequestedWidth + ", mRequestedheight="
                + mRequestedHeight + ") to" + " (pw=" + pw + ", ph=" + ph
                + "): frame=" + mFrame.toShortString()
                + " ci=" + mContentInsets.toShortString()
                + " vi=" + mVisibleInsets.toShortString()
                + " si=" + mStableInsets.toShortString()
                + " of=" + mOutsets.toShortString());
    }

```

继续看看DisplayContent方法中的RootWindowContainer.performSurfacePlacement(recoveringMemory)
##### 1.2.3、surfacePlacer.handleAppTransitionReadyLocked()

``` java
G:\android9.0\frameworks\base\services\core\java\com\android\server\wm\WindowSurfacePlacer.java

    int handleAppTransitionReadyLocked() {
        int appsCount = mService.mOpeningApps.size();
        if (!transitionGoodToGo(appsCount, mTempTransitionReasons)) {
            return 0;
        }

    private boolean transitionGoodToGo(int appsCount, SparseIntArray outReasons) {
        if (DEBUG_APP_TRANSITIONS) Slog.v(TAG,
                "Checking " + appsCount + " opening apps (frozen="
                        + mService.mDisplayFrozen + " timeout="
                        + mService.mAppTransition.isTimeout() + ")...");
        final ScreenRotationAnimation screenRotationAnimation =
            mService.mAnimator.getScreenRotationAnimationLocked(
                    Display.DEFAULT_DISPLAY);

        outReasons.clear();
        if (!mService.mAppTransition.isTimeout()) {
           ......
----------------------------------------------------------------
对应Log：
04-16 13:08:37.009 V/WindowSurfacePlacer( 1049): Checking 1 opening apps (frozen=false timeout=false)...
04-16 13:08:37.009 V/WindowSurfacePlacer( 1049): Check opening app=AppWindowToken{1579134 token=Token{ce2ac07 ActivityRecord{65cc87a u0 com.android.testred/.TestActivity t5}}}: allDrawn=false startingDisplayed=false startingMoved=false isRelaunching()=false startingWindow=null
----------------------------------------------------------------
            for (int i = 0; i < appsCount; i++) {
                AppWindowToken wtoken = mService.mOpeningApps.valueAt(i);
                if (DEBUG_APP_TRANSITIONS) Slog.v(TAG,
                        "Check opening app=" + wtoken + ": allDrawn="
                        + wtoken.allDrawn + " startingDisplayed="
                        + wtoken.startingDisplayed + " startingMoved="
                        + wtoken.startingMoved + " isRelaunching()="
                        + wtoken.isRelaunching() + " startingWindow="
                        + wtoken.startingWindow);


                final boolean allDrawn = wtoken.allDrawn && !wtoken.isRelaunching();
                if (!allDrawn && !wtoken.startingDisplayed && !wtoken.startingMoved) {
                    return false;
                }
                final int windowingMode = wtoken.getWindowingMode();
                if (allDrawn) {
                    outReasons.put(windowingMode,  APP_TRANSITION_WINDOWS_DRAWN);
                } else {
                    outReasons.put(windowingMode,
                            wtoken.startingData instanceof SplashScreenStartingData
                                    ? APP_TRANSITION_SPLASH_SCREEN
                                    : APP_TRANSITION_SNAPSHOT);
                }
            }

           ......
        }
        return true;
    }
```

根据前面分析可知道，现在App（com.android.testred）的大小和布局都处理好了，等待显示，但现在还么有mDrawState=NO_SURFACE ，肯定无法显示出来的，接下来看看Surface创建过程。

``` java
04-16 13:08:37.007 V/AppWindowToken( 1049): Eval win Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity}: isDrawn=false, isAnimationSet=true
04-16 13:08:37.007 V/AppWindowToken( 1049): Not displayed: s=null pv=true mDrawState=NO_SURFACE ph=false th=false a=true
```
继续看看WindowManagerService.relayoutWindow()
##### （2）、Surface创建过程


``` java
G:\android9.0\frameworks\base\services\core\java\com\android\server\wm\WindowManagerService.java

    private int createSurfaceControl(Surface outSurface, int result, WindowState win,
            WindowStateAnimator winAnimator) {
        if (!win.mHasSurface) {
            result |= RELAYOUT_RES_SURFACE_CHANGED;
        }

        WindowSurfaceController surfaceController;
        try {
            Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, "createSurfaceControl");
            surfaceController = winAnimator.createSurfaceLocked(win.mAttrs.type, win.mOwnerUid);
        } finally {
            Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
        }
        if (surfaceController != null) {
            surfaceController.getSurface(outSurface);
            if (SHOW_TRANSACTIONS) Slog.i(TAG_WM, "  OUT SURFACE " + outSurface + ": copied");
        } else {
            // For some reason there isn't a surface.  Clear the
            // caller's object so they see the same state.
            Slog.w(TAG_WM, "Failed to create surface control for " + win);
            outSurface.release();
        }

        return result;
    }
```

##### 2.1、winAnimator.createSurfaceLocked()

``` java
G:\android9.0\frameworks\base\services\core\java\com\android\server\wm\WindowStateAnimator.java

    WindowSurfaceController createSurfaceLocked(int windowType, int ownerUid) {
        final WindowState w = mWin;

        if (mSurfaceController != null) {
            return mSurfaceController;
        }
        mChildrenDetached = false;

        if ((mWin.mAttrs.privateFlags & PRIVATE_FLAG_IS_ROUNDED_CORNERS_OVERLAY) != 0) {
            windowType = SurfaceControl.WINDOW_TYPE_DONT_SCREENSHOT;
        }

        w.setHasSurface(false);

        if (DEBUG_ANIM || DEBUG_ORIENTATION) Slog.i(TAG,
                "createSurface " + this + ": mDrawState=DRAW_PENDING");
----------------------------------------------------------------
对应Log：
04-16 13:08:37.010 I/WindowStateAnimator( 1049): createSurface WindowStateAnimator{4fb81c9 com.android.testred/com.android.testred.TestActivity}: mDrawState=DRAW_PENDING
----------------------------------------------------------------
        resetDrawState();

        mService.makeWindowFreezingScreenIfNeededLocked(w);

        int flags = SurfaceControl.HIDDEN;//设置flags 为 HIDDEN
        final WindowManager.LayoutParams attrs = w.mAttrs;

        if (mService.isSecureLocked(w)) {
            flags |= SurfaceControl.SECURE;
        }

        mTmpSize.set(0, 0, 0, 0);
        //计算Surface大小
        calculateSurfaceBounds(w, attrs);
        final int width = mTmpSize.width();
        final int height = mTmpSize.height();

        if (DEBUG_VISIBILITY) {
            Slog.v(TAG, "Creating surface in session "
                    + mSession.mSurfaceSession + " window " + this
                    + " w=" + width + " h=" + height
                    + " x=" + mTmpSize.left + " y=" + mTmpSize.top
                    + " format=" + attrs.format + " flags=" + flags);
        }
----------------------------------------------------------------
对应Log：
04-16 13:08:37.010 V/WindowStateAnimator( 1049): Creating surface in session android.view.SurfaceSession@3812e82 window WindowStateAnimator{4fb81c9 com.android.testred/com.android.testred.TestActivity} w=160 h=854 x=0 y=0 format=-2 flags=4

----------------------------------------------------------------
        // We may abort, so initialize to defaults.
        mLastClipRect.set(0, 0, 0, 0);

        // Set up surface control with initial size.
        try {

            final boolean isHwAccelerated = (attrs.flags & FLAG_HARDWARE_ACCELERATED) != 0;
            final int format = isHwAccelerated ? PixelFormat.TRANSLUCENT : attrs.format;
            if (!PixelFormat.formatHasAlpha(attrs.format)
                    // Don't make surface with surfaceInsets opaque as they display a
                    // translucent shadow.
                    && attrs.surfaceInsets.left == 0
                    && attrs.surfaceInsets.top == 0
                    && attrs.surfaceInsets.right == 0
                    && attrs.surfaceInsets.bottom == 0
                    // Don't make surface opaque when resizing to reduce the amount of
                    // artifacts shown in areas the app isn't drawing content to.
                    && !w.isDragResizing()) {
                flags |= SurfaceControl.OPAQUE;
            }

            mSurfaceController = new WindowSurfaceController(mSession.mSurfaceSession,
                    attrs.getTitle().toString(), width, height, format, flags, this,
                    windowType, ownerUid);

            setOffsetPositionForStackResize(false);
            mSurfaceFormat = format;

            w.setHasSurface(true);

            if (SHOW_TRANSACTIONS || SHOW_SURFACE_ALLOC) {
                Slog.i(TAG, "  CREATE SURFACE "
                        + mSurfaceController + " IN SESSION "
                        + mSession.mSurfaceSession
                        + ": pid=" + mSession.mPid + " format="
                        + attrs.format + " flags=0x"
                        + Integer.toHexString(flags)
                        + " / " + this);
            }


----------------------------------------------------------------
对应Log：
04-16 13:08:37.012 I/WindowStateAnimator( 1049):   CREATE SURFACE Surface(name=com.android.testred/com.android.testred.TestActivity)/@0xb17b0ce IN SESSION android.view.SurfaceSession@3812e82: pid=2602 format=-2 flags=0x4 / WindowStateAnimator{4fb81c9 com.android.testred/com.android.testred.TestActivity}
----------------------------------------------------------------
        } catch (OutOfResourcesException e) {
            Slog.w(TAG, "OutOfResourcesException creating surface");
            mService.mRoot.reclaimSomeSurfaceMemory(this, "create", true);
            mDrawState = NO_SURFACE;
            return null;
        } catch (Exception e) {
            Slog.e(TAG, "Exception creating surface (parent dead?)", e);
            mDrawState = NO_SURFACE;
            return null;
        }

        if (WindowManagerService.localLOGV) Slog.v(TAG, "Got surface: " + mSurfaceController
                + ", set left=" + w.mFrame.left + " top=" + w.mFrame.top
                + ", animLayer=" + mAnimLayer);
----------------------------------------------------------------
对应Log：
04-16 13:08:37.012 V/WindowStateAnimator( 1049): Got surface: Surface(name=com.android.testred/com.android.testred.TestActivity)/@0xb17b0ce, set left=0 top=0, animLayer=0
----------------------------------------------------------------
        if (SHOW_LIGHT_TRANSACTIONS) {
            Slog.i(TAG, ">>> OPEN TRANSACTION createSurfaceLocked");
            WindowManagerService.logSurface(w, "CREATE pos=("
                    + w.mFrame.left + "," + w.mFrame.top + ") ("
                    + width + "x" + height + "), layer=" + mAnimLayer + " HIDE", false);
        }

        mLastHidden = true;

        if (WindowManagerService.localLOGV) Slog.v(TAG, "Created surface " + this);
----------------------------------------------------------------
对应Log：
04-16 13:08:37.012 I/WindowStateAnimator( 1049): >>> OPEN TRANSACTION createSurfaceLocked
04-16 13:08:37.012 I/WindowManager( 1049):   SURFACE CREATE pos=(0,0) (160x854), layer=0 HIDE: Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity}
04-16 13:08:37.012 V/WindowStateAnimator( 1049): Created surface WindowStateAnimator{4fb81c9 com.android.testred/com.android.testred.TestActivity}
----------------------------------------------------------------
        return mSurfaceController;
    }
```


##### 2.1.1、new WindowSurfaceController()

``` java
G:\android9.0\frameworks\base\services\core\java\com\android\server\wm\WindowSurfaceController.java

    public WindowSurfaceController(SurfaceSession s, String name, int w, int h, int format,
            int flags, WindowStateAnimator animator, int windowType, int ownerUid) {
        mAnimator = animator;

        mSurfaceW = w;
        mSurfaceH = h;

        title = name;

        mService = animator.mService;
        final WindowState win = animator.mWin;
        mWindowType = windowType;
        mWindowSession = win.mSession;

        Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, "new SurfaceControl");
        final SurfaceControl.Builder b = win.makeSurface()
                .setParent(win.getSurfaceControl())
                .setName(name)
                .setSize(w, h)
                .setFormat(format)
                .setFlags(flags)
                .setMetadata(windowType, ownerUid);
        mSurfaceControl = b.build();
        Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
    }
```
###### 2.1.1.1、创建SurfaceControl build()


``` java
G:\android9.0\frameworks\base\core\java\android\view\SurfaceControl.java

        public SurfaceControl build() {
            if (mWidth <= 0 || mHeight <= 0) {
                throw new IllegalArgumentException(
                        "width and height must be set");
            }
            return new SurfaceControl(mSession, mName, mWidth, mHeight, mFormat,
                    mFlags, mParent, mWindowType, mOwnerUid);
        }
```
###### 2.1.1.2、new SurfaceControl()

``` java
    /**
     * Create a surface with a name.
     * <p>
     * The surface creation flags specify what kind of surface to create and
     * certain options such as whether the surface can be assumed to be opaque
     * and whether it should be initially hidden.  Surfaces should always be
     * created with the {@link #HIDDEN} flag set to ensure that they are not
     * made visible prematurely before all of the surface's properties have been
     * configured.
     * <p>
     * Good practice is to first create the surface with the {@link #HIDDEN} flag
     * specified, open a transaction, set the surface layer, layer stack, alpha,
     * and position, call {@link #show} if appropriate, and close the transaction.
     *
     * @param session The surface session, must not be null.
     * @param name The surface name, must not be null.
     * @param w The surface initial width.
     * @param h The surface initial height.
     * @param flags The surface creation flags.  Should always include {@link #HIDDEN}
     * in the creation flags.
     * @param windowType The type of the window as specified in WindowManager.java.
     * @param ownerUid A unique per-app ID.
     *
     * @throws throws OutOfResourcesException If the SurfaceControl cannot be created.
     */
    private SurfaceControl(SurfaceSession session, String name, int w, int h, int format, int flags,
            SurfaceControl parent, int windowType, int ownerUid)
                    throws OutOfResourcesException, IllegalArgumentException {
        ......

        if ((flags & SurfaceControl.HIDDEN) == 0) {
            Log.w(TAG, "Surfaces should always be created with the HIDDEN flag set "
                    + "to ensure that they are not made visible prematurely before "
                    + "all of the surface's properties have been configured.  "
                    + "Set the other properties and make the surface visible within "
                    + "a transaction.  New surface name: " + name,
                    new Throwable());
        }

        mName = name;
        mWidth = w;
        mHeight = h;
        mNativeObject = nativeCreate(session, name, w, h, format, flags,
            parent != null ? parent.mNativeObject : 0, windowType, ownerUid);
        ......
        mCloseGuard.open("release");
    }



G:\android9.0\frameworks\base\core\jni\android_view_SurfaceControl.cpp

static jlong nativeCreate(JNIEnv* env, jclass clazz, jobject sessionObj,
        jstring nameStr, jint w, jint h, jint format, jint flags, jlong parentObject,
        jint windowType, jint ownerUid) {
    ScopedUtfChars name(env, nameStr);
    sp<SurfaceComposerClient> client(android_view_SurfaceSession_getClient(env, sessionObj));
    SurfaceControl *parent = reinterpret_cast<SurfaceControl*>(parentObject);
    sp<SurfaceControl> surface;
    status_t err = client->createSurfaceChecked(
            String8(name.c_str()), w, h, format, &surface, flags, parent, windowType, ownerUid);
    if (err == NAME_NOT_FOUND) {
        jniThrowException(env, "java/lang/IllegalArgumentException", NULL);
        return 0;
    } else if (err != NO_ERROR) {
        jniThrowException(env, OutOfResourcesException, NULL);
        return 0;
    }

    surface->incStrong((void *)nativeCreate);
    return reinterpret_cast<jlong>(surface.get());
}
```

该函数首先得到前面创建好的SurfaceComposerClient对象，通过该对象向SurfaceFlinger端的Client对象发送创建Surface的请求，最后得到一个SurfaceControl对象
看看createSurfaceChecked()函数实现

``` cpp
G:\android9.0\frameworks\native\libs\gui\SurfaceComposerClient.cpp
status_t SurfaceComposerClient::createSurfaceChecked(
        const String8& name,
        uint32_t w,
        uint32_t h,
        PixelFormat format,
        sp<SurfaceControl>* outSurface,
        uint32_t flags,
        SurfaceControl* parent,
        int32_t windowType,
        int32_t ownerUid)
{
    sp<SurfaceControl> sur;
    status_t err = mStatus;

    if (mStatus == NO_ERROR) {
        sp<IBinder> handle;
        sp<IBinder> parentHandle;
        sp<IGraphicBufferProducer> gbp;

        if (parent != nullptr) {
            parentHandle = parent->getHandle();
        }
        err = mClient->createSurface(name, w, h, format, flags, parentHandle,
                windowType, ownerUid, &handle, &gbp);
        ALOGE_IF(err, "SurfaceComposerClient::createSurface error %s", strerror(-err));
        if (err == NO_ERROR) {
            *outSurface = new SurfaceControl(this, handle, gbp, true /* owned */);
        }
    }
    return err;
}


G:\android9.0\frameworks\native\libs\gui\SurfaceControl.cpp
SurfaceControl::SurfaceControl(
        const sp<SurfaceComposerClient>& client,
        const sp<IBinder>& handle,
        const sp<IGraphicBufferProducer>& gbp,
        bool owned)
    : mClient(client), mHandle(handle), mGraphicBufferProducer(gbp), mOwned(owned)
{
}

```
SurfaceComposerClient将Surface创建请求转交给保存在其成员变量中的Bp SurfaceComposerClient对象来完成，在SurfaceFlinger端的Client本地对象会返回一个ISurface代理对象给应用程序，通过该代理对象为应用程序当前创建的Surface创建一个SurfaceControl对象。 [ISurfaceComposerClient.cpp]


``` cpp
G:\android9.0\frameworks\native\libs\gui\ISurfaceComposerClient.cpp
    status_t createSurface(const String8& name, uint32_t width, uint32_t height, PixelFormat format,
                           uint32_t flags, const sp<IBinder>& parent, int32_t windowType,
                           int32_t ownerUid, sp<IBinder>* handle,
                           sp<IGraphicBufferProducer>* gbp) override {
        return callRemote<decltype(&ISurfaceComposerClient::createSurface)>(Tag::CREATE_SURFACE,
                                                                            name, width, height,
                                                                            format, flags, parent,
                                                                            windowType, ownerUid,
                                                                            handle, gbp);
    }

```


###### 2.1.1.3、Client->createSurface()
MessageCreateSurface消息是专门为应用程序请求创建Surface而定义的一种消息类型：
``` cpp
G:\android9.0\frameworks\native\services\surfaceflinger\Client.cpp
status_t Client::createSurface(
        const String8& name,
        uint32_t w, uint32_t h, PixelFormat format, uint32_t flags,
        const sp<IBinder>& parentHandle, int32_t windowType, int32_t ownerUid,
        sp<IBinder>* handle,
        sp<IGraphicBufferProducer>* gbp)
{
    sp<Layer> parent = nullptr;
    if (parentHandle != nullptr) {
        auto layerHandle = reinterpret_cast<Layer::Handle*>(parentHandle.get());
        parent = layerHandle->owner.promote();
        if (parent == nullptr) {
            return NAME_NOT_FOUND;
        }
    }
    if (parent == nullptr) {
        bool parentDied;
        parent = getParentLayer(&parentDied);
        // If we had a parent, but it died, we've lost all
        // our capabilities.
        if (parentDied) {
            return NAME_NOT_FOUND;
        }
    }

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
        sp<Layer>* parent;
        int32_t windowType;
        int32_t ownerUid;
    public:
        MessageCreateLayer(SurfaceFlinger* flinger,
                const String8& name, Client* client,
                uint32_t w, uint32_t h, PixelFormat format, uint32_t flags,
                sp<IBinder>* handle, int32_t windowType, int32_t ownerUid,
                sp<IGraphicBufferProducer>* gbp,
                sp<Layer>* parent)
            : flinger(flinger), client(client),
              handle(handle), gbp(gbp), result(NO_ERROR),
              name(name), w(w), h(h), format(format), flags(flags),
              parent(parent), windowType(windowType), ownerUid(ownerUid) {
        }
        status_t getResult() const { return result; }
        virtual bool handler() {
            result = flinger->createLayer(name, client, w, h, format, flags,
                    windowType, ownerUid, handle, gbp, parent);
            return true;
        }
    };

    sp<MessageBase> msg = new MessageCreateLayer(mFlinger.get(),
            name, this, w, h, format, flags, handle,
            windowType, ownerUid, gbp, &parent);
    mFlinger->postMessageSync(msg);
    return static_cast<MessageCreateLayer*>( msg.get() )->getResult();
}


```
SurfaceFlinger根据标志位创建对应类型的Surface，当前系统定义了2种类型的Layer

``` java
G:\android9.0\frameworks\native\libs\gui\include\gui\ISurfaceComposerClient.h

        eFXSurfaceNormal = 0x00000000,//正常surface
        eFXSurfaceColor = 0x00020000,//只有颜色数据surface
```

``` cpp
G:\android9.0\frameworks\native\services\surfaceflinger\SurfaceFlinger.cpp

status_t SurfaceFlinger::createLayer(
        const String8& name,
        const sp<Client>& client,
        uint32_t w, uint32_t h, PixelFormat format, uint32_t flags,
        int32_t windowType, int32_t ownerUid, sp<IBinder>* handle,
        sp<IGraphicBufferProducer>* gbp, sp<Layer>* parent)
{
    if (int32_t(w|h) < 0) {
        ALOGE("createLayer() failed, w or h is negative (w=%d, h=%d)",
                int(w), int(h));
        return BAD_VALUE;
    }

    status_t result = NO_ERROR;

    sp<Layer> layer;

    String8 uniqueName = getUniqueLayerName(name);

    switch (flags & ISurfaceComposerClient::eFXSurfaceMask) {
        case ISurfaceComposerClient::eFXSurfaceNormal:
            result = createBufferLayer(client,
                    uniqueName, w, h, flags, format,
                    handle, gbp, &layer);

            break;
        case ISurfaceComposerClient::eFXSurfaceColor:
            result = createColorLayer(client,
                    uniqueName, w, h, flags,
                    handle, &layer);
            break;
        default:
            result = BAD_VALUE;
            break;
    }

    if (result != NO_ERROR) {
        return result;
    }

    // window type is WINDOW_TYPE_DONT_SCREENSHOT from SurfaceControl.java
    // TODO b/64227542
    if (windowType == 441731) {
        windowType = 2024; // TYPE_NAVIGATION_BAR_PANEL
        layer->setPrimaryDisplayOnly();
    }

    layer->setInfo(windowType, ownerUid);

    result = addClientLayer(client, *handle, *gbp, layer, *parent);
    if (result != NO_ERROR) {
        return result;
    }
    mInterceptor->saveSurfaceCreation(layer);

    setTransactionFlags(eTransactionNeeded);
    return result;
}

```

有前面java层可知flags = SurfaceControl.HIDDEN，flags & ISurfaceComposerClient::eFXSurfaceMask == eFXSurfaceNormal = 0x00000000，所以此外走createBufferLayer()分支

###### 2.1.1.4、createBufferLayer()

``` java
G:\android9.0\frameworks\native\services\surfaceflinger\SurfaceFlinger.cpp

status_t SurfaceFlinger::createBufferLayer(const sp<Client>& client,
        const String8& name, uint32_t w, uint32_t h, uint32_t flags, PixelFormat& format,
        sp<IBinder>* handle, sp<IGraphicBufferProducer>* gbp, sp<Layer>* outLayer)
{
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

    sp<BufferLayer> layer = DisplayUtils::getInstance()->getBufferLayerInstance(
                            this, client, name, w, h, flags);
    status_t err = layer->setBuffers(w, h, format, flags);
----------------------------------------------------------------
对应Log：

status_t BufferLayer::setBuffers(uint32_t w, uint32_t h, PixelFormat format, uint32_t flags) {
    uint32_t const maxSurfaceDims =
            min(mFlinger->getMaxTextureSize(), mFlinger->getMaxViewportDims());

    // never allow a surface larger than what our underlying GL implementation
    // can handle.
    if ((uint32_t(w) > maxSurfaceDims) || (uint32_t(h) > maxSurfaceDims)) {
        ALOGE("dimensions too large %u x %u", uint32_t(w), uint32_t(h));
        return BAD_VALUE;
    }

    mFormat = format;

    mPotentialCursor = (flags & ISurfaceComposerClient::eCursorWindow) ? true : false;
    mProtectedByApp = (flags & ISurfaceComposerClient::eProtectedByApp) ? true : false;
    mCurrentOpacity = getOpacityForFormat(format);

    mConsumer->setDefaultBufferSize(w, h);
    mConsumer->setDefaultBufferFormat(format);
    mConsumer->setConsumerUsageBits(getEffectiveUsage(0));

    return NO_ERROR;
}

04-16 13:08:37.011 V/BufferQueueConsumer(  739): [com.android.testred/com.android.testred.TestActivity#0] setDefaultBufferSize: width=160 height=854
04-16 13:08:37.011 V/BufferQueueConsumer(  739): [com.android.testred/com.android.testred.TestActivity#0] setDefaultBufferFormat: 1
04-16 13:08:37.011 V/BufferQueueConsumer(  739): [com.android.testred/com.android.testred.TestActivity#0] setConsumerUsageBits: 0x900
----------------------------------------------------------------
    if (err == NO_ERROR) {
        *handle = layer->getHandle();
        *gbp = layer->getProducer();
        *outLayer = layer;
    }

    ALOGE_IF(err, "createBufferLayer() failed (%s)", strerror(-err));
    return err;
}
```

###### 2.1.1.5、DisplayUtils::getInstance()->getBufferLayerInstance()

``` cpp
G:\android9.0\frameworks\native\services\surfaceflinger\DisplayUtils.cpp

BufferLayer* DisplayUtils::getBufferLayerInstance(SurfaceFlinger* flinger,
                            const sp<Client>& client, const String8& name,
                            uint32_t w, uint32_t h, uint32_t flags) {
    if (sUseExtendedImpls) {
        return new ExBufferLayer(flinger, client, name, w, h, flags);
    } else {
        return new BufferLayer(flinger, client, name, w, h, flags);
    }
}
```
###### 2.1.1.6、new BufferLayer()

``` cpp
G:\android9.0\frameworks\native\services\surfaceflinger\BufferLayer.cpp

BufferLayer::BufferLayer(SurfaceFlinger* flinger, const sp<Client>& client, const String8& name,
                         uint32_t w, uint32_t h, uint32_t flags)
      : Layer(flinger, client, name, w, h, flags),
        mConsumer(nullptr),
        mTextureName(UINT32_MAX),
        mFormat(PIXEL_FORMAT_NONE),
        mCurrentScalingMode(NATIVE_WINDOW_SCALING_MODE_FREEZE),
        mBufferLatched(false),
        mPreviousFrameNumber(0),
        mUpdateTexImageFailed(false),
        mRefreshPending(false) {
    ALOGV("Creating Layer %s", name.string());
----------------------------------------------------------------
对应Log：
04-16 13:08:37.011 V/BufferLayer(  739): Creating Layer com.android.testred/com.android.testred.TestActivity#0
----------------------------------------------------------------
    //生成Textures并初始化
    mFlinger->getRenderEngine().genTextures(1, &mTextureName);
    mTexture.init(Texture::TEXTURE_EXTERNAL, mTextureName);

    if (flags & ISurfaceComposerClient::eNonPremultiplied) mPremultipliedAlpha = false;

    mCurrentState.requested = mCurrentState.active;

    // drawing state & current state are identical
    mDrawingState = mCurrentState;
}

```


回到SurfaceComposerClient.cpp中
继续看看，根据回传的 sp<IBinder> handle和 sp<IGraphicBufferProducer> gbp 创建SurfaceControl
*outSurface = new SurfaceControl(this, handle, gbp, true /* owned */)

然后android_view_SurfaceControl.cpp nativeCreate()函数中将 reinterpret_cast<jlong>(surface.get()) 的地址返回给Java层。



###### 2.1.1.7、BufferLayer->onFirstRef()
第一次强引用Layer对象时，onFirstRef()函数被回调 
``` cpp
G:\android9.0\frameworks\native\services\surfaceflinger\BufferLayer.cpp
void BufferLayer::onFirstRef() {
    // Creates a custom BufferQueue for SurfaceFlingerConsumer to use
    sp<IGraphicBufferProducer> producer;
    sp<IGraphicBufferConsumer> consumer;
    BufferQueue::createBufferQueue(&producer, &consumer, true);
    mProducer = new MonitoredProducer(producer, mFlinger, this);
    mConsumer = new BufferLayerConsumer(consumer,
            mFlinger->getRenderEngine(), mTextureName, this);


----------------------------------------------------------------
对应Log：
04-16 13:08:37.011 V/BufferQueueConsumer(  739): [unnamed-739-40] setConsumerUsageBits: 0x900
04-16 13:08:37.011 V/ConsumerBase(  739): [unnamed-739-40] setFrameAvailableListener
04-16 13:08:37.011 V/BufferQueueConsumer(  739): [unnamed-739-40] setConsumerName: 'com.android.testred/com.android.testred.TestActivity#0'
04-16 13:08:37.011 V/BufferQueueProducer(  739): [] setMaxDequeuedBufferCount: maxDequeuedBuffers = 2
----------------------------------------------------------------
    mConsumer->setConsumerUsageBits(getEffectiveUsage(0));
    mConsumer->setContentsChangedListener(this);
    mConsumer->setName(mName);

    if (mFlinger->isLayerTripleBufferingDisabled()) {
        mProducer->setMaxDequeuedBufferCount(2);
    }

    const sp<const DisplayDevice> hw(mFlinger->getDefaultDisplayDevice());
----------------------------------------------------------------
对应Log：
04-16 13:08:37.011 V/BufferQueueConsumer(  739): [com.android.testred/com.android.testred.TestActivity#0] setTransformHint: 0
----------------------------------------------------------------
    updateTransformHint(hw);
}



```


![enter image description here](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/personal.website/zjj.display.sys.SurfaceFlinger-ConsumeLisener-onFrameAvailable.png) 


###### 2.1.1.8、BufferQueue::createBufferQueue()
所以核心都是这个BufferQueueCore，他是管理图形缓冲区的中枢。
``` cpp
G:\android9.0\frameworks\native\libs\gui\BufferQueue.cpp
void BufferQueue::createBufferQueue(sp<IGraphicBufferProducer>* outProducer,
        sp<IGraphicBufferConsumer>* outConsumer,
        bool consumerIsSurfaceFlinger) {
    ......
    sp<BufferQueueCore> core(new BufferQueueCore());
    sp<IGraphicBufferProducer> producer(new BufferQueueProducer(core, consumerIsSurfaceFlinger));
    
    sp<IGraphicBufferConsumer> consumer(new BufferQueueConsumer(core));
    *outProducer = producer;
    *outConsumer = consumer;
}


G:\android9.0\frameworks\native\libs\gui\BufferQueueCore.cpp
BufferQueueCore::BufferQueueCore() :
    ......
    mUniqueId(getUniqueId())
{
    int numStartingBuffers = getMaxBufferCountLocked();
    for (int s = 0; s < numStartingBuffers; s++) {
        mFreeSlots.insert(s);
    }
    //static constexpr int NUM_BUFFER_SLOTS = 64;
    for (int s = numStartingBuffers; s < BufferQueueDefs::NUM_BUFFER_SLOTS;
            s++) {
        mUnusedSlots.push_front(s);
    }
}

```

###### 2.1.1.9、new BufferLayerConsumer()

``` cpp
G:\android9.0\frameworks\native\services\surfaceflinger\BufferLayerConsumer.cpp

BufferLayerConsumer::BufferLayerConsumer(const sp<IGraphicBufferConsumer>& bq,
                                         RE::RenderEngine& engine, uint32_t tex, Layer* layer)
      : ConsumerBase(bq, false),
        mCurrentCrop(Rect::EMPTY_RECT),
        mCurrentTransform(0),
        mCurrentScalingMode(NATIVE_WINDOW_SCALING_MODE_FREEZE),
        mCurrentFence(Fence::NO_FENCE),
        mCurrentTimestamp(0),
        mCurrentDataSpace(ui::Dataspace::UNKNOWN),
        mCurrentFrameNumber(0),
        mCurrentTransformToDisplayInverse(false),
        mCurrentSurfaceDamage(),
        mCurrentApi(0),
        mDefaultWidth(1),
        mDefaultHeight(1),
        mFilteringEnabled(true),
        mRE(engine),
        mTexName(tex),
        mLayer(layer),
        mCurrentTexture(BufferQueue::INVALID_BUFFER_SLOT) {
----------------------------------------------------------------
对应Log：
04-16 13:08:37.011 V/BufferLayerConsumer(  739): [unnamed-739-40] BufferLayerConsumer
----------------------------------------------------------------
    BLC_LOGV("BufferLayerConsumer");

    memcpy(mCurrentTransformMatrix, mtxIdentity.asArray(), sizeof(mCurrentTransformMatrix));
----------------------------------------------------------------
对应Log：
04-16 13:08:37.011 V/BufferQueueConsumer(  739): [unnamed-739-40] setConsumerUsageBits: 0x100
----------------------------------------------------------------
    mConsumer->setConsumerUsageBits(DEFAULT_USAGE_FLAGS);
}

```
BufferLayerConsumer会首先初始化ConsumerBase
###### 2.1.1.10、ConsumerBase()


``` cpp

G:\android9.0\frameworks\native\libs\gui\ConsumerBase.cpp

ConsumerBase::ConsumerBase(const sp<IGraphicBufferConsumer>& bufferQueue, bool controlledByApp) :
        mAbandoned(false),
        mConsumer(bufferQueue),
        mPrevFinalReleaseFence(Fence::NO_FENCE) {
    // Choose a name using the PID and a process-unique ID.
    mName = String8::format("unnamed-%d-%d", getpid(), createProcessUniqueId());

    // Note that we can't create an sp<...>(this) in a ctor that will not keep a
    // reference once the ctor ends, as that would cause the refcount of 'this'
    // dropping to 0 at the end of the ctor.  Since all we need is a wp<...>
    // that's what we create.
    wp<ConsumerListener> listener = static_cast<ConsumerListener*>(this);
    sp<IConsumerListener> proxy = new BufferQueue::ProxyConsumerListener(listener);

----------------------------------------------------------------
对应Log：
04-16 13:08:37.011 V/BufferQueueConsumer(  739): [] connect: controlledByApp=false
04-16 13:08:37.011 V/BufferQueueConsumer(  739): [] setConsumerName: 'unnamed-739-40'
----------------------------------------------------------------

    status_t err = mConsumer->consumerConnect(proxy, controlledByApp);
    if (err != NO_ERROR) {
        CB_LOGE("ConsumerBase: error connecting to BufferQueue: %s (%d)",
                strerror(-err), err);
    } else {
        mConsumer->setConsumerName(mName);
    }
}
```

![enter image description here](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/personal.website/zjj.display.sys.consumerbase.png) 


##### 2.2、winAnimator.createSurfaceLocked()
饶了一大圈，现在Surface创建好了，现象看看如何传回App进程


``` java
G:\android9.0\frameworks\base\services\core\java\com\android\server\wm\WindowManagerService.java

private int createSurfaceControl(Surface outSurface, int result, WindowState win,
        WindowStateAnimator winAnimator) {
    if (!win.mHasSurface) {
        result |= RELAYOUT_RES_SURFACE_CHANGED;
    }

    WindowSurfaceController surfaceController;
    try {
        Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, "createSurfaceControl");
        surfaceController = winAnimator.createSurfaceLocked(win.mAttrs.type, win.mOwnerUid);
    } finally {
        Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
    }
    if (surfaceController != null) {
        surfaceController.getSurface(outSurface);
----------------------------------------------------------------
对应Log：
04-16 13:08:37.012 I/WindowManager( 1049):   OUT SURFACE Surface(name=null)/@0xe493bef: copied
----------------------------------------------------------------
        if (SHOW_TRANSACTIONS) Slog.i(TAG_WM, "  OUT SURFACE " + outSurface + ": copied");
    } else {
        // For some reason there isn't a surface.  Clear the
        // caller's object so they see the same state.
        Slog.w(TAG_WM, "Failed to create surface control for " + win);
        outSurface.release();
    }

    return result;
}
```
###### 2.2.1、surfaceController.getSurface(outSurface)


``` cpp
G:\android9.0\frameworks\base\services\core\java\com\android\server\wm\WindowSurfaceController.java

void getSurface(Surface outSurface) {
    outSurface.copyFrom(mSurfaceControl);
}
```

###### 2.2.2、outSurface.copyFrom(mSurfaceControl)

``` cpp
G:\android9.0\frameworks\base\core\java\android\view\Surface.java

    public void copyFrom(SurfaceControl other) {
        ......
        long surfaceControlPtr = other.mNativeObject;
        .......
        long newNativeObject = nativeGetFromSurfaceControl(surfaceControlPtr);

        synchronized (mLock) {
            if (mNativeObject != 0) {
                nativeRelease(mNativeObject);
            }
            setNativeObjectLocked(newNativeObject);
        }
    }

```
###### 2.2.3、nativeGetFromSurfaceControl()

``` cpp
G:\android9.0\frameworks\base\core\jni\android_view_Surface.cpp

static jlong nativeGetFromSurfaceControl(JNIEnv* env, jclass clazz,
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



G:\android9.0\frameworks\native\libs\gui\SurfaceControl.cpp

sp<Surface> SurfaceControl::generateSurfaceLocked() const
{
    // This surface is always consumed by SurfaceFlinger, so the
    // producerControlledByApp value doesn't matter; using false.
    mSurfaceData = new Surface(mGraphicBufferProducer, false);

    return mSurfaceData;
}

sp<Surface> SurfaceControl::getSurface() const
{
    Mutex::Autolock _l(mLock);
    if (mSurfaceData == 0) {
        return generateSurfaceLocked();
    }
    return mSurfaceData;
}

```


###### 2.2.4、Nativie Surface创建过程
``` cpp
Surface::Surface(const sp<IGraphicBufferProducer>& bufferProducer, bool controlledByApp)
      : mGraphicBufferProducer(bufferProducer),
        mCrop(Rect::EMPTY_RECT),
        mBufferAge(0),
        mGenerationNumber(0),
        mSharedBufferMode(false),
        mAutoRefresh(false),
        mSharedBufferSlot(BufferItem::INVALID_BUFFER_SLOT),
        mSharedBufferHasBeenQueued(false),
        mQueriedSupportedTimestamps(false),
        mFrameTimestampsSupportsPresent(false),
        mEnableFrameTimestamps(false),
        mFrameEventHistory(std::make_unique<ProducerFrameEventHistory>()) {
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
    mDataSpace = Dataspace::UNKNOWN;
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

###### 2.2.5、Surface跨进程传递（ WMS->App）writeToParcel()

得到真正的Surface后，[->IWindowSession$Stub.java]

``` cpp
      case 6:
         ......
         Surface _arg17 = new Surface();
         int _result2 = this.relayout(_arg0, _arg1, _arg25, _arg3, _arg4, _arg5, _arg62, _result4, _result7, _arg9, _arg10, _arg11, _arg12, _arg13, _arg14, _arg15, _arg16, _arg17);
         ......
         if(_arg17 != null) {
            reply.writeInt(1);
            _arg17.writeToParcel(reply, 1);
         } else {
            reply.writeInt(0);
         }
         return true;
```

WMS会首先通过writeToParcel()函数将Surface对象写入App 进程端

``` java
G:\android9.0\frameworks\base\core\java\android\view\Surface.java

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


G:\android9.0\frameworks\base\core\jni\android_view_Surface.cpp

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

###### 2.2.6、Surface跨进程传递（ App->WMS）readFromParcel()

[->IWindowSession$Stub$Proxy.java]
``` java
  public int relayout(IWindow window, int seq, WindowManager.LayoutParams attrs, int requestedWidth, int requestedHeight, int viewVisibility, int flags, long frameNumber, Rect outFrame, Rect outOverscanInsets, Rect outContentInsets, Rect outVisibleInsets, Rect outStableInsets, Rect outOutsets, Rect outBackdropFrame, DisplayCutout.ParcelableWrapper displayCutout, MergedConfiguration outMergedConfiguration, Surface outSurface) throws RemoteException {
    Parcel _data = Parcel.obtain();
    Parcel _reply = Parcel.obtain();
    try
{
   ......
   mRemote.transact(6, _data, _reply, 0);
   _reply.readException();
   int _result = _reply.readInt();
  ......
  if (0 != _reply.readInt()) {
    outSurface.readFromParcel(_reply);
  }
}
```

APP进程通过readFromParcel() 读取WMS写入的数据


``` java
G:\android9.0\frameworks\base\core\java\android\view\Surface.java

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

G:\android9.0\frameworks\base\core\jni\android_view_Surface.cpp

static jlong nativeReadFromParcel(JNIEnv* env, jclass clazz,
        jlong nativeObject, jobject parcelObj) {
    Parcel* parcel = parcelForJavaObject(env, parcelObj);
    if (parcel == NULL) {
        doThrowNPE(env);
        return 0;
    }

    android::view::Surface surfaceShim;

    // Calling code in Surface.java has already read the name of the Surface
    // from the Parcel
    surfaceShim.readFromParcel(parcel, /*nameAlreadyRead*/true);

    sp<Surface> self(reinterpret_cast<Surface *>(nativeObject));

    // update the Surface only if the underlying IGraphicBufferProducer
    // has changed.
    if (self != nullptr
            && (IInterface::asBinder(self->getIGraphicBufferProducer()) ==
                    IInterface::asBinder(surfaceShim.graphicBufferProducer))) {
        // same IGraphicBufferProducer, return ourselves
        return jlong(self.get());
    }

    sp<Surface> sur;
    if (surfaceShim.graphicBufferProducer != nullptr) {
        // we have a new IGraphicBufferProducer, create a new Surface for it
        sur = new Surface(surfaceShim.graphicBufferProducer, true);
        // and keep a reference before passing to java
        sur->incStrong(&sRefBaseOwner);
    }

    if (self != NULL) {
        // and loose the java reference to ourselves
        self->decStrong(&sRefBaseOwner);
    }

    return jlong(sur.get());
}

```
![enter image description here](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/personal.website/zjj.display.sys.Surface-SurfaceControl.png) 


##### （3）、焦点窗口更新updateFocusedWindowLocked()

继续看看relayoutWindow()方法，createSurfaceControl()之后会更新焦点窗口

``` java
G:\android9.0\frameworks\base\services\core\java\com\android\server\wm\WindowManagerService.java

    // TODO: Move to DisplayContent
    boolean updateFocusedWindowLocked(int mode, boolean updateInputWindows) {
        WindowState newFocus = mRoot.computeFocusedWindow();
        if (mCurrentFocus != newFocus) {
            Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, "wmUpdateFocus");
            // This check makes sure that we don't already have the focus
            // change message pending.
----------------------------------------------------------------
对应Log：
04-16 13:08:37.018 I/WindowManager( 1049): Focus moving from null to Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity}
04-16 13:08:37.018 I/WindowManager( 1049): Gaining focus: Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity}
----------------------------------------------------------------
            mH.removeMessages(H.REPORT_FOCUS_CHANGE);
            mH.sendEmptyMessage(H.REPORT_FOCUS_CHANGE);
            // TODO(multidisplay): Focused windows on default display only.
            final DisplayContent displayContent = getDefaultDisplayContentLocked();
            boolean imWindowChanged = false;
            //处理输入法相关
            ......
----------------------------------------------------------------
对应Log：
04-16 13:08:37.012 V/WindowManager( 1049): Changing focus from null to Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity} Callers=com.android.server.wm.WindowManagerService.relayoutWindow:2088 com.android.server.wm.Session.relayout:244 android.view.IWindowSession$Stub.onTransact:309 com.android.server.wm.Session.onTransact:164 

----------------------------------------------------------------
            if (DEBUG_FOCUS_LIGHT || localLOGV) Slog.v(TAG_WM, "Changing focus from " +
                    mCurrentFocus + " to " + newFocus + " Callers=" + Debug.getCallers(4));
            final WindowState oldFocus = mCurrentFocus;
            mCurrentFocus = newFocus;
            mLosingFocus.remove(newFocus);

            if (mCurrentFocus != null) {
                mWinAddedSinceNullFocus.clear();
                mWinRemovedSinceNullFocus.clear();
            }

            int focusChanged = mPolicy.focusChangedLw(oldFocus, newFocus);

            ......
----------------------------------------------------------------
对应Log：
04-16 13:08:37.013 D/WindowManager( 1049): Input focus has changed to Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity}

----------------------------------------------------------------
            if (mode != UPDATE_FOCUS_WILL_ASSIGN_LAYERS) {
                mInputMonitor.setInputFocusLw(mCurrentFocus, updateInputWindows);
            }

            displayContent.adjustForImeIfNeeded();
            displayContent.scheduleToastWindowsTimeoutIfNeededLocked(oldFocus, newFocus);

            Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
            return true;
        }
        return false;
    }
```


##### 3.1、RootWindowContainer.computeFocusedWindow()

``` java
G:\android9.0\frameworks\base\services\core\java\com\android\server\wm\RootWindowContainer.java

    WindowState computeFocusedWindow() {
        // While the keyguard is showing, we must focus anything besides the main display.
        // Otherwise we risk input not going to the keyguard when the user expects it to.
        final boolean forceDefaultDisplay = mService.isKeyguardShowingAndNotOccluded();

        for (int i = mChildren.size() - 1; i >= 0; i--) {
            final DisplayContent dc = mChildren.get(i);
            final WindowState win = dc.findFocusedWindow();
            if (win != null) {
                if (forceDefaultDisplay && !dc.isDefaultDisplay) {
                    EventLog.writeEvent(0x534e4554, "71786287", win.mOwnerUid, "");
                    continue;
                }
                return win;
            }
        }
        return null;
    }
```
###### 3.1.1、DisplayContent.findFocusedWindow()

``` java

G:\android9.0\frameworks\base\services\core\java\com\android\server\wm\DisplayContent.java
    WindowState findFocusedWindow() {
        mTmpWindow = null;
        forAllWindows(mFindFocusedWindow, true /* traverseTopToBottom */);
        return mTmpWindow;
    }

    private final ToBooleanFunction<WindowState> mFindFocusedWindow = w -> {
        final AppWindowToken focusedApp = mService.mFocusedApp;
----------------------------------------------------------------
对应Log：
04-16 13:08:37.012 V/WindowManager( 1049): Looking for focus: Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity}, flags=25231616, canReceive=true
04-16 13:08:37.012 V/WindowManager( 1049): findFocusedWindow: Found new focus @ Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity}
----------------------------------------------------------------
        if (DEBUG_FOCUS) Slog.v(TAG_WM, "Looking for focus: " + w
                + ", flags=" + w.mAttrs.flags + ", canReceive=" + w.canReceiveKeys());
        .......
        final AppWindowToken wtoken = w.mAppToken;
        ......
        if (DEBUG_FOCUS_LIGHT) Slog.v(TAG_WM, "findFocusedWindow: Found new focus @ " + w);
        mTmpWindow = w;
        return true;
    };
```


##### 3.2、Visibility更新AppWindowToken.updateReportedVisibilityLocked()

根据WindowState 的 mDrawState=1  可知，现在WindowState  状态是DRAW_PENDING。

``` java
G:\android9.0\frameworks\base\services\core\java\com\android\server\wm\AppWindowToken.java


----------------------------------------------------------------
对应Log：
08:37.013 V/AppWindowToken( 1049): Update reported visibility: AppWindowToken{1579134 token=Token{ce2ac07 ActivityRecord{65cc87a u0 com.android.testred/.TestActivity t5}}}
04-16 13:08:37.013 V/WindowState( 1049): Win Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity}: isDrawn=false, isAnimationSet=true
04-16 13:08:37.013 V/WindowState( 1049): Not displayed: s=Surface(name=com.android.testred/com.android.testred.TestActivity)/@0xb17b0ce pv=true mDrawState=1 ph=false th=false a=true
04-16 13:08:37.013 V/AppWindowToken( 1049): VIS AppWindowToken{1579134 token=Token{ce2ac07 ActivityRecord{65cc87a u0 com.android.testred/.TestActivity t5}}}: interesting=1 visible=0

----------------------------------------------------------------

    void updateReportedVisibilityLocked() {
        if (appToken == null) {
            return;
        }

        if (DEBUG_VISIBILITY) Slog.v(TAG, "Update reported visibility: " + this);
        final int count = mChildren.size();

        mReportedVisibilityResults.reset();

        for (int i = 0; i < count; i++) {
            final WindowState win = mChildren.get(i);
            win.updateReportedVisibility(mReportedVisibilityResults);
        }

        int numInteresting = mReportedVisibilityResults.numInteresting;
        int numVisible = mReportedVisibilityResults.numVisible;
        int numDrawn = mReportedVisibilityResults.numDrawn;
        boolean nowGone = mReportedVisibilityResults.nowGone;

        boolean nowDrawn = numInteresting > 0 && numDrawn >= numInteresting;
        boolean nowVisible = numInteresting > 0 && numVisible >= numInteresting && !isHidden();
        if (!nowGone) {
            // If the app is not yet gone, then it can only become visible/drawn.
            if (!nowDrawn) {
                nowDrawn = reportedDrawn;
            }
            if (!nowVisible) {
                nowVisible = reportedVisible;
            }
        }
        if (DEBUG_VISIBILITY) Slog.v(TAG, "VIS " + this + ": interesting="
                + numInteresting + " visible=" + numVisible);
        final AppWindowContainerController controller = getController();
        if (nowDrawn != reportedDrawn) {
            if (nowDrawn) {
                if (controller != null) {
                    controller.reportWindowsDrawn();
                }
            }
            reportedDrawn = nowDrawn;
        }
        if (nowVisible != reportedVisible) {
            if (DEBUG_VISIBILITY) Slog.v(TAG,
                    "Visibility changed in " + this + ": vis=" + nowVisible);
            reportedVisible = nowVisible;
            if (controller != null) {
                if (nowVisible) {
                    controller.reportWindowsVisible();
                } else {
                    controller.reportWindowsGone();
                }
            }
        }
    }

```

至此
ViewRootImpl.java 的 performTraversals() 方法中的relayoutWindow(params, viewVisibility, insetsPending)终于完成了。

``` java

G:\android9.0\frameworks\base\core\java\android\view\ViewRootImpl.java

    private void performTraversals() {
......

----------------------------------------------------------------
对应Log：
04-16 13:08:37.015 V/ViewRootImpl[TestActivity]( 2602): relayout: frame=[0,0][160,854] overscan=[0,0][0,0] content=[0,0][0,0] visible=[0,36][0,72] stable=[0,36][0,72] cutout=DisplayCutout{insets=Rect(0, 0 - 0, 0) boundingRect=Rect(0, 0 - 0, 0)} outsets=[0,0][0,0] surface=Surface(name=null)/@0x266c9a5

----------------------------------------------------------------
if (DEBUG_LAYOUT) Log.v(mTag, "relayout: frame=" + frame.toShortString()
                        + " overscan=" + mPendingOverscanInsets.toShortString()
                        + " content=" + mPendingContentInsets.toShortString()
                        + " visible=" + mPendingVisibleInsets.toShortString()
                        + " stable=" + mPendingStableInsets.toShortString()
                        + " cutout=" + mPendingDisplayCutout.get().toString()
                        + " outsets=" + mPendingOutsets.toShortString()
                        + " surface=" + mSurface);

                ......
----------------------------------------------------------------
对应Log：
04-16 13:08:37.015 V/ViewRootImpl[TestActivity]( 2602): Visible with new config: {1.0 ?mcc?mnc [en_US] ldltr sw320dp w320dp h497dp 240dpi nrml port finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 480, 854) mAppBounds=Rect(0, 0 - 480, 782) mWindowingMode=fullscreen mActivityType=standard} s.5}
04-16 13:08:37.015 V/ViewRootImpl[TestActivity]( 2602): Applying new config to window com.android.testred/com.android.testred.TestActivity, globalConfig: {1.0 ?mcc?mnc [en_US] ldltr sw320dp w320dp h497dp 240dpi nrml port finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 480, 782) 
mAppBounds=Rect(0, 0 - 480, 782) mWindowingMode=fullscreen mActivityType=undefined} s.5}, overrideConfig: {1.0 ?mcc?mnc [en_US] ldltr sw320dp w320dp h497dp 240dpi nrml port finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 480, 854) mAppBounds=Rect(0, 0 - 480, 782) mWindowingMode=fullscreen mActivityType=standard} s.5}
----------------------------------------------------------------
                if (!mPendingMergedConfiguration.equals(mLastReportedMergedConfiguration)) {
                    if (DEBUG_CONFIGURATION) Log.v(mTag, "Visible with new config: "
                            + mPendingMergedConfiguration.getMergedConfiguration());
                    performConfigurationChange(mPendingMergedConfiguration, !mFirst,
                            INVALID_DISPLAY /* same display */);
                    updatedConfiguration = true;
                }

                ......
                if (contentInsetsChanged) {
----------------------------------------------------------------
对应Log：
04-16 13:08:37.015 V/ViewRootImpl[TestActivity]( 2602): Content insets changing to: Rect(0, 0 - 0, 0)
----------------------------------------------------------------
                    mAttachInfo.mContentInsets.set(mPendingContentInsets);
                    if (DEBUG_LAYOUT) Log.v(mTag, "Content insets changing to: "
                            + mAttachInfo.mContentInsets);
                }
----------------------------------------------------------------
对应Log：
04-16 13:08:37.015 V/ViewRootImpl[TestActivity]( 2602): Decor insets changing to: Rect(0, 36 - 0, 72)
----------------------------------------------------------------
                if (stableInsetsChanged) {
                    mAttachInfo.mStableInsets.set(mPendingStableInsets);
                    if (DEBUG_LAYOUT) Log.v(mTag, "Decor insets changing to: "
                            + mAttachInfo.mStableInsets);
                    // Need to relayout with content insets.
                    contentInsetsChanged = true;
                }
----------------------------------------------------------------
对应Log：
04-16 13:08:37.015 V/ViewRootImpl[TestActivity]( 2602): Visible insets changing to: Rect(0, 36 - 0, 72)
----------------------------------------------------------------
                if (visibleInsetsChanged) {
                    mAttachInfo.mVisibleInsets.set(mPendingVisibleInsets);
                    if (DEBUG_LAYOUT) Log.v(mTag, "Visible insets changing to: "
                            + mAttachInfo.mVisibleInsets);
                }

                .......

            if (DEBUG_ORIENTATION) Log.v(
                    TAG, "Relayout returned: frame=" + frame + ", surface=" + mSurface);

            mAttachInfo.mWindowLeft = frame.left;
            mAttachInfo.mWindowTop = frame.top;

            // !!FIXME!! This next section handles the case where we did not get the
            // window size we asked for. We should avoid this by getting a maximum size from
            // the window session beforehand.
            if (mWidth != frame.width() || mHeight != frame.height()) {
                mWidth = frame.width();
                mHeight = frame.height();
            }

            ......

            final ThreadedRenderer threadedRenderer = mAttachInfo.mThreadedRenderer;
            if (threadedRenderer != null && threadedRenderer.isEnabled()) {
                if (hwInitialized
                        || mWidth != threadedRenderer.getWidth()
                        || mHeight != threadedRenderer.getHeight()
                        || mNeedsRendererSetup) {
                    threadedRenderer.setup(mWidth, mHeight, mAttachInfo,
                            mWindowAttributes.surfaceInsets);
                    mNeedsRendererSetup = false;
                }
            }

            if (!mStopped || mReportNextDraw) {
                boolean focusChangedDueToTouchMode = ensureTouchModeLocally(
                        (relayoutResult&WindowManagerGlobal.RELAYOUT_RES_IN_TOUCH_MODE) != 0);
                if (focusChangedDueToTouchMode || mWidth != host.getMeasuredWidth()
                        || mHeight != host.getMeasuredHeight() || contentInsetsChanged ||
                        updatedConfiguration) {
                    int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
                    int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
----------------------------------------------------------------
对应Log：

04-16 13:08:37.016 V/ViewRootImpl[TestActivity]( 2602): Ooops, something changed!  mWidth=160 measuredWidth=480 mHeight=854 measuredHeight=782 coveredInsetsChanged=true

----------------------------------------------------------------
                    if (DEBUG_LAYOUT) Log.v(mTag, "Ooops, something changed!  mWidth="
                            + mWidth + " measuredWidth=" + host.getMeasuredWidth()
                            + " mHeight=" + mHeight
                            + " measuredHeight=" + host.getMeasuredHeight()
                            + " coveredInsetsChanged=" + contentInsetsChanged);


                     // Ask host how big it wants to be
/****************执行View控件大小计算******************/  
                    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);

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
                    ......
                    layoutRequested = true;
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

            // By this point all views have been sized and positioned
            // We can compute the transparent area

            if ((host.mPrivateFlags & View.PFLAG_REQUEST_TRANSPARENT_REGIONS) != 0) {
                // start out transparent
                // TODO: AVOID THAT CALL BY CACHING THE RESULT?
                host.getLocationInWindow(mTmpLocation);
                mTransparentRegion.set(mTmpLocation[0], mTmpLocation[1],
                        mTmpLocation[0] + host.mRight - host.mLeft,
                        mTmpLocation[1] + host.mBottom - host.mTop);

                host.gatherTransparentRegion(mTransparentRegion);
                if (mTranslator != null) {
                    mTranslator.translateRegionInWindowToScreen(mTransparentRegion);
                }

                if (!mTransparentRegion.equals(mPreviousTransparentRegion)) {
                    mPreviousTransparentRegion.set(mTransparentRegion);
                    mFullRedrawNeeded = true;
                    // reconfigure window manager
                    try {
                        mWindowSession.setTransparentRegion(mWindow, mTransparentRegion);
                    } catch (RemoteException e) {
                    }
                }
            }

----------------------------------------------------------------
对应Log：

04-16 13:08:37.017 I/System.out( 2602): ======================================
04-16 13:08:37.017 I/System.out( 2602): performTraversals -- after setFrame
----------------------------------------------------------------
            if (DBG) {
                System.out.println("======================================");
                System.out.println("performTraversals -- after setFrame");
                host.debug();

----------------------------------------------------------------
对应Log：
04-16 13:08:37.017 D/View    ( 2602):   + DecorView@515e1ca[TestActivity]
04-16 13:08:37.017 D/View    ( 2602):       frame={0, 0, 160, 854} scroll={0, 0} 
04-16 13:08:37.018 D/View    ( 2602):       mMeasureWidth=160 mMeasureHeight=854
04-16 13:08:37.020 D/Debug   ( 2602):       Contents of {(0,0)(fillxfill) sim={forwardNavigation} ty=BASE_APPLICATION fmt=TRANSPARENT wanim=0x1030000
04-16 13:08:37.020 D/Debug   ( 2602):   fl=LAYOUT_IN_SCREEN LAYOUT_INSET_DECOR SPLIT_TOUCH HARDWARE_ACCELERATED}:
04-16 13:08:37.020 D/Debug   ( 2602): ViewGroup.LayoutParams={ width=match-parent, height=match-parent }
04-16 13:08:37.020 D/Debug   ( 2602): 
04-16 13:08:37.020 D/Debug   ( 2602): WindowManager.LayoutParams={title=com.android.testred/com.android.testred.TestActivity}
04-16 13:08:37.020 D/View    ( 2602): 
04-16 13:08:37.020 D/View    ( 2602):       flags={}
04-16 13:08:37.020 D/View    ( 2602):       privateFlags={IS_ROOT_NAMESPACE HAS_BOUNDS DRAWN}
04-16 13:08:37.020 D/View    ( 2602):       {
04-16 13:08:37.020 D/View    ( 2602):       + android.widget.LinearLayout{c1507be V.ED..... ......ID 0,0-160,854}
04-16 13:08:37.020 D/View    ( 2602):           frame={0, 0, 160, 854} scroll={0, 0} 
04-16 13:08:37.020 D/View    ( 2602):           mMeasureWidth=160 mMeasureHeight=854
04-16 13:08:37.020 D/View    ( 2602):           ViewGroup.LayoutParams={ width=match-parent, height=match-parent }
04-16 13:08:37.020 D/View    ( 2602):           flags={}
04-16 13:08:37.020 D/View    ( 2602):           privateFlags={HAS_BOUNDS}
04-16 13:08:37.020 D/View    ( 2602):           {
04-16 13:08:37.020 D/View    ( 2602):           + android.view.ViewStub{870dc6e G.E...... ......I. 0,0-0,0 #102018a android:id/action_mode_bar_stub} (id=16908682)
04-16 13:08:37.020 D/View    ( 2602):               frame={0, 0, 0, 0} scroll={0, 0} 
04-16 13:08:37.020 D/View    ( 2602):               mMeasureWidth=0 mMeasureHeight=0
04-16 13:08:37.020 D/View    ( 2602):               LinearLayout.LayoutParams={width=match-parent, height=wrap-content weight=0.0}
04-16 13:08:37.020 D/View    ( 2602):               flags={GONE}
04-16 13:08:37.020 D/View    ( 2602):               privateFlags={DRAWN}
04-16 13:08:37.020 D/View    ( 2602):           + android.widget.FrameLayout{b582935 V.ED..... ......ID 0,0-160,38}
04-16 13:08:37.020 D/View    ( 2602):               frame={0, 0, 160, 38} scroll={0, 0} 
04-16 13:08:37.020 D/View    ( 2602):               padding={6, 1, 6, 2}
04-16 13:08:37.020 D/View    ( 2602):               mMeasureWidth=160 mMeasureHeight=38
04-16 13:08:37.020 D/View    ( 2602):               LinearLayout.LayoutParams={width=match-parent, height=38 weight=0.0}
04-16 13:08:37.020 D/View    ( 2602):               flags={}
04-16 13:08:37.020 D/View    ( 2602):               privateFlags={HAS_BOUNDS}
04-16 13:08:37.020 D/View    ( 2602):               {
04-16 13:08:37.020 D/View    ( 2602):               + android.widget.TextView{64b8f0f V.ED..... ......ID 6,1-154,36 #1020016 android:id/title} (id=16908310)
04-16 13:08:37.020 D/View    ( 2602):                   frame={6, 1, 154, 36} scroll={0, 0} 
04-16 13:08:37.020 D/View    ( 2602):                   mMeasureWidth=148 mMeasureHeight=35
04-16 13:08:37.020 D/View    ( 2602):                   ViewGroup.LayoutParams={ width=match-parent, height=match-parent }
04-16 13:08:37.021 D/View    ( 2602):                   flags={}
04-16 13:08:37.021 D/View    ( 2602):                   privateFlags={HAS_BOUNDS}
04-16 13:08:37.021 D/View    ( 2602):                   frame={6, 1, 154, 36} scroll={0, 0} mText="Test Viewport" mLayout width=1048576 height=29
04-16 13:08:37.021 D/View    ( 2602):               }
04-16 13:08:37.021 D/View    ( 2602):           + android.widget.FrameLayout{158df3b V.ED..... ......ID 0,38-160,854 #1020002 android:id/content} (id=16908290)
04-16 13:08:37.021 D/View    ( 2602):               frame={0, 38, 160, 854} scroll={0, 0} 
04-16 13:08:37.021 D/View    ( 2602):               mMeasureWidth=160 mMeasureHeight=816
04-16 13:08:37.021 D/View    ( 2602):               LinearLayout.LayoutParams={width=match-parent, height=0 weight=1.0}
04-16 13:08:37.021 D/View    ( 2602):               flags={}
04-16 13:08:37.021 D/View    ( 2602):               privateFlags={HAS_BOUNDS}
04-16 13:08:37.021 D/View    ( 2602):               {
04-16 13:08:37.021 D/View    ( 2602):               + com.android.testred.TestView{85cab9c VFE...... ......ID 0,0-160,816}
04-16 13:08:37.021 D/View    ( 2602):                   frame={0, 0, 160, 816} scroll={0, 0} 
04-16 13:08:37.021 D/View    ( 2602):                   mMeasureWidth=160 mMeasureHeight=816
04-16 13:08:37.021 D/View    ( 2602):                   ViewGroup.LayoutParams={ width=match-parent, height=match-parent }
04-16 13:08:37.021 D/View    ( 2602):                   flags={TAKES_FOCUS}
04-16 13:08:37.021 D/View    ( 2602):                   privateFlags={HAS_BOUNDS}
04-16 13:08:37.021 D/View    ( 2602):               }
04-16 13:08:37.021 D/View    ( 2602):           }
04-16 13:08:37.021 D/View    ( 2602):       }
----------------------------------------------------------------
            }
        }

        
        ......

        if (mFirst) {
            if (sAlwaysAssignFocus || !isInTouchMode()) {
                // handle first focus request
----------------------------------------------------------------
对应Log：
04-16 13:08:37.021 V/ViewRootImpl[TestActivity]( 2602): First: mView.hasFocus()=false
----------------------------------------------------------------
                if (DEBUG_INPUT_RESIZE) {
                    Log.v(mTag, "First: mView.hasFocus()=" + mView.hasFocus());
                }
                if (mView != null) {
                    if (!mView.hasFocus()) {
----------------------------------------------------------------
mView.restoreDefaultFocus();对应Log：
04-16 13:08:37.021 I/System.out( 2602): DecorView@515e1ca[TestActivity] ViewGroup.requestFocus direction=130
04-16 13:08:37.021 I/System.out( 2602): android.widget.LinearLayout{c1507be V.ED..... ......ID 0,0-160,854} ViewGroup.requestFocus direction=130
04-16 13:08:37.021 I/System.out( 2602): android.widget.FrameLayout{b582935 V.ED..... ......ID 0,0-160,38} ViewGroup.requestFocus direction=130
04-16 13:08:37.021 I/System.out( 2602): android.widget.FrameLayout{158df3b V.ED..... ......ID 0,38-160,854 #1020002 android:id/content} ViewGroup.requestFocus direction=130
04-16 13:08:37.021 I/System.out( 2602): com.android.testred.TestView{85cab9c VFE...... ......ID 0,0-160,816} requestFocus()
04-16 13:08:37.021 I/System.out( 2602): Find focus in DecorView@515e1ca[TestActivity]: flags=false, child=null
04-16 13:08:37.021 I/System.out( 2602): android.widget.FrameLayout{158df3b V.ED..... ......ID 0,38-160,854 #1020002 android:id/content} requestChildFocus()
04-16 13:08:37.021 I/System.out( 2602): android.widget.FrameLayout{158df3b V.ED..... ......ID 0,38-160,854 #1020002 android:id/content} unFocus()
04-16 13:08:37.022 I/System.out( 2602): android.widget.LinearLayout{c1507be V.ED..... ......ID 0,0-160,854} requestChildFocus()
04-16 13:08:37.022 I/System.out( 2602): android.widget.LinearLayout{c1507be V.ED..... ......ID 0,0-160,854} unFocus()
04-16 13:08:37.022 I/System.out( 2602): DecorView@515e1ca[TestActivity] requestChildFocus()
04-16 13:08:37.022 I/System.out( 2602): DecorView@515e1ca[TestActivity] unFocus()
04-16 13:08:37.022 V/ViewRootImpl[TestActivity]( 2602): Request child focus: focus now com.android.testred.TestView{85cab9c VFE...... .F....ID 0,0-160,816}
04-16 13:08:37.022 I/System.out( 2602): Find focus in DecorView@515e1ca[TestActivity]: flags=false, child=android.widget.LinearLayout{c1507be V.ED..... ......ID 0,0-160,854}
04-16 13:08:37.022 I/System.out( 2602): Find focus in android.widget.LinearLayout{c1507be V.ED..... ......ID 0,0-160,854}: flags=false, child=android.widget.FrameLayout{158df3b V.ED..... ......ID 0,38-160,854 #1020002 android:id/content}
04-16 13:08:37.022 I/System.out( 2602): Find focus in android.widget.FrameLayout{158df3b V.ED..... ......ID 0,38-160,854 #1020002 android:id/content}: flags=false, child=com.android.testred.TestView{85cab9c VFE...... .F....ID 0,0-160,816}
04-16 13:08:37.022 V/ViewRootImpl[TestActivity]( 2602): First: requested focused view=com.android.testred.TestView{85cab9c VFE...... .F....ID 0,0-160,816}
----------------------------------------------------------------
                        mView.restoreDefaultFocus();
                        if (DEBUG_INPUT_RESIZE) {
                            Log.v(mTag, "First: requested focused view=" + mView.findFocus());
                        }
                    } else {
                        if (DEBUG_INPUT_RESIZE) {
                            Log.v(mTag, "First: existing focused view=" + mView.findFocus());
                        }
                    }
                }
            } else {
                // Some views (like ScrollView) won't hand focus to descendants that aren't within
                // their viewport. Before layout, there's a good change these views are size 0
                // which means no children can get focus. After layout, this view now has size, but
                // is not guaranteed to hand-off focus to a focusable child (specifically, the edge-
                // case where the child has a size prior to layout and thus won't trigger
                // focusableViewAvailable).
                View focused = mView.findFocus();
                if (focused instanceof ViewGroup
                        && ((ViewGroup) focused).getDescendantFocusability()
                                == ViewGroup.FOCUS_AFTER_DESCENDANTS) {
                    focused.restoreDefaultFocus();
                }
            }
        }
        ......


    /****************执行窗口绘制******************/  
    
        // Remember if we must report the next draw.
        if ((relayoutResult & WindowManagerGlobal.RELAYOUT_RES_FIRST_TIME) != 0) {
            reportNextDraw();
        }

        boolean cancelDraw = mAttachInfo.mTreeObserver.dispatchOnPreDraw() || !isViewVisible;

        if (!cancelDraw && !newSurface) {
            ......
            performDraw();
        } else {
            if (isViewVisible) {
                scheduleTraversals();
            } ......
        }

        mIsInTraversal = false;
    }
```


##### （4）、执行窗口布局performLayout()

``` java
G:\android9.0\frameworks\base\core\java\android\view\ViewRootImpl.java

    private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
            int desiredWindowHeight) {
        mLayoutRequested = false;
        mScrollMayChange = true;
        mInLayout = true;

        final View host = mView;
        ......

----------------------------------------------------------------
对应Log：

04-16 13:08:37.017 V/ViewRootImpl[TestActivity]( 2602): Laying out DecorView@515e1ca[TestActivity] to (160, 854)
----------------------------------------------------------------
        if (DEBUG_ORIENTATION || DEBUG_LAYOUT) {
            Log.v(mTag, "Laying out " + host + " to (" +
                    host.getMeasuredWidth() + ", " + host.getMeasuredHeight() + ")");
        }

        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "layout");
        try {
            host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());

            mInLayout = false;
            int numViewsRequestingLayout = mLayoutRequesters.size();
            ......
        mInLayout = false;
    }

```
###### 4.1、View.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight())

``` java
G:\android9.0\frameworks\base\core\java\android\view\View.java

    public void layout(int l, int t, int r, int b) {
        if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
            onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }

        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;

        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            onLayout(changed, l, t, r, b);
            ......
            mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;

            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnLayoutChangeListeners != null) {
                ArrayList<OnLayoutChangeListener> listenersCopy =
                        (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
                int numListeners = listenersCopy.size();
                for (int i = 0; i < numListeners; ++i) {
                    listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
                }
            }
        }

        final boolean wasLayoutValid = isLayoutValid();

        mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
        mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;

        ......
    }
```


###### 4.1.1、View.setFrame

``` java
G:\android9.0\frameworks\base\core\java\android\view\View.java

    protected boolean setFrame(int left, int top, int right, int bottom) {
        boolean changed = false;

        if (DBG) {
            Log.d(VIEW_LOG_TAG, this + " View.setFrame(" + left + "," + top + ","
                    + right + "," + bottom + ")");
        }
----------------------------------------------------------------
对应Log：

04-16 13:08:37.017 D/View    ( 2602): DecorView@515e1ca[TestActivity] View.setFrame(0,0,160,854)
04-16 13:08:37.017 V/ViewRootImpl[TestActivity]( 2602): Invalidate child: Rect(0, 0 - 160, 854)
04-16 13:08:37.017 D/View    ( 2602): android.widget.LinearLayout{c1507be V.ED..... ......I. 0,0-0,0} View.setFrame(0,0,160,854)
04-16 13:08:37.017 D/View    ( 2602): android.widget.FrameLayout{b582935 V.ED..... ......ID 0,0-0,0} View.setFrame(0,0,160,38)
04-16 13:08:37.017 D/View    ( 2602): android.widget.TextView{64b8f0f V.ED..... ......ID 0,0-0,0 #1020016 android:id/title} View.setFrame(6,1,154,36)
04-16 13:08:37.017 D/View    ( 2602): android.widget.FrameLayout{158df3b V.ED..... ......I. 0,0-0,0 #1020002 android:id/content} View.setFrame(0,38,160,854)
04-16 13:08:37.017 D/View    ( 2602): com.android.testred.TestView{85cab9c VFE...... ......I. 0,0-0,0} View.setFrame(0,0,160,816)
----------------------------------------------------------------
        if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
            changed = true;

            // Remember our drawn bit
            int drawn = mPrivateFlags & PFLAG_DRAWN;

            int oldWidth = mRight - mLeft;
            int oldHeight = mBottom - mTop;
            int newWidth = right - left;
            int newHeight = bottom - top;
            boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);

            // Invalidate our old position
            invalidate(sizeChanged);

            mLeft = left;
            mTop = top;
            mRight = right;
            mBottom = bottom;
            mRenderNode.setLeftTopRightBottom(mLeft, mTop, mRight, mBottom);

            mPrivateFlags |= PFLAG_HAS_BOUNDS;


            if (sizeChanged) {
                sizeChange(newWidth, newHeight, oldWidth, oldHeight);
            }

            if ((mViewFlags & VISIBILITY_MASK) == VISIBLE || mGhostView != null) {

                mPrivateFlags |= PFLAG_DRAWN;
                invalidate(sizeChanged);

                invalidateParentCaches();
            }

            // Reset drawn bit to original value (invalidate turns it off)
            mPrivateFlags |= drawn;

            mBackgroundSizeChanged = true;
            mDefaultFocusHighlightSizeChanged = true;
            if (mForegroundInfo != null) {
                mForegroundInfo.mBoundsChanged = true;
            }

            notifySubtreeAccessibilityStateChangedIfNeeded();
        }
        return changed;
    }

```

###### 4.1.2、SurfaceView.setFrame

``` java
G:\android9.0\frameworks\base\core\java\android\view\SurfaceView.java
Override
protected boolean setFrame(int left, int top, int right, int bottom) {
    boolean result = super.setFrame(left, top, right, bottom);
    updateSurface();
    return result;
}

    protected void updateSurface() {

        ViewRootImpl viewRoot = getViewRootImpl();
        ......

        if (creating || formatChanged || sizeChanged || visibleChanged || windowVisibleChanged) {
            getLocationInWindow(mLocation);

            try {
                final boolean visible = mVisible = mRequestedVisible;
                .......

                if (creating) {
                    mSurfaceSession = new SurfaceSession(viewRoot.mSurface);
                    mDeferredDestroySurfaceControl = mSurfaceControl;

                    updateOpaqueFlag();
                    final String name = "SurfaceView - " + viewRoot.getTitle().toString();

                    mSurfaceControl = new SurfaceControlWithBackground(
                            name,
                            (mSurfaceFlags & SurfaceControl.OPAQUE) != 0,
                            new SurfaceControl.Builder(mSurfaceSession)
                                    .setSize(mSurfaceWidth, mSurfaceHeight)
                                    .setFormat(mFormat)
                                    .setFlags(mSurfaceFlags));
                } else if (mSurfaceControl == null) {
                    return;
                }

                ......
    }


----------------------------------------------------------------
对应Log：
04-16 13:08:37.023 V/BufferLayer(  739): Creating Layer SurfaceView - com.android.testred/com.android.testred.TestActivity#0
04-16 13:08:37.023 V/BufferQueueConsumer(  739): [] connect: controlledByApp=false
04-16 13:08:37.023 V/BufferQueueConsumer(  739): [] setConsumerName: 'unnamed-739-41'
04-16 13:08:37.023 V/BufferLayerConsumer(  739): [unnamed-739-41] BufferLayerConsumer
04-16 13:08:37.023 V/BufferQueueConsumer(  739): [unnamed-739-41] setConsumerUsageBits: 0x100
04-16 13:08:37.023 V/BufferQueueConsumer(  739): [unnamed-739-41] setConsumerUsageBits: 0x900
04-16 13:08:37.023 V/ConsumerBase(  739): [unnamed-739-41] setFrameAvailableListener
04-16 13:08:37.023 V/BufferQueueConsumer(  739): [unnamed-739-41] setConsumerName: 'SurfaceView - com.android.testred/com.android.testred.TestActivity#0'
04-16 13:08:37.023 V/BufferQueueProducer(  739): [] setMaxDequeuedBufferCount: maxDequeuedBuffers = 2
04-16 13:08:37.023 V/BufferQueueConsumer(  739): [SurfaceView - com.android.testred/com.android.testred.TestActivity#0] setTransformHint: 0
04-16 13:08:37.023 V/BufferQueueConsumer(  739): [SurfaceView - com.android.testred/com.android.testred.TestActivity#0] setDefaultBufferSize: width=160 height=816
04-16 13:08:37.023 V/BufferQueueConsumer(  739): [SurfaceView - com.android.testred/com.android.testred.TestActivity#0] setDefaultBufferFormat: 4
04-16 13:08:37.023 V/BufferQueueConsumer(  739): [SurfaceView - com.android.testred/com.android.testred.TestActivity#0] setConsumerUsageBits: 0x900
----------------------------------------------------------------
```
 updateSurface()会请求创建surface，创建过程前面已经分析过了


##### （5）、执行窗口绘制performDraw()

``` java
/frameworks/base/core/java/android/view/ViewRootImpl.java

   private void performDraw() {
        if (mAttachInfo.mDisplayState == Display.STATE_OFF && !mReportNextDraw) {
            return;
        } else if (mView == null) {
            return;
        }

        final boolean fullRedrawNeeded = mFullRedrawNeeded || mReportNextDraw;
        mFullRedrawNeeded = false;

        mIsDrawing = true;
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "draw");

        boolean usingAsyncReport = false;
        if (mReportNextDraw && mAttachInfo.mThreadedRenderer != null
                && mAttachInfo.mThreadedRenderer.isEnabled()) {
            usingAsyncReport = true;
            mAttachInfo.mThreadedRenderer.setFrameCompleteCallback((long frameNr) -> {
                // TODO: Use the frame number
                pendingDrawFinished();
            });
        }

        try {
            boolean canUseAsync = draw(fullRedrawNeeded);
            if (usingAsyncReport && !canUseAsync) {
                mAttachInfo.mThreadedRenderer.setFrameCompleteCallback(null);
                usingAsyncReport = false;
            }
        } finally {
            mIsDrawing = false;
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }

        // For whatever reason we didn't create a HardwareRenderer, end any
        // hardware animations that are now dangling
        if (mAttachInfo.mPendingAnimatingRenderNodes != null) {
            final int count = mAttachInfo.mPendingAnimatingRenderNodes.size();
            for (int i = 0; i < count; i++) {
                mAttachInfo.mPendingAnimatingRenderNodes.get(i).endAllAnimators();
            }
            mAttachInfo.mPendingAnimatingRenderNodes.clear();
        }

        if (mReportNextDraw) {
            mReportNextDraw = false;

            // if we're using multi-thread renderer, wait for the window frame draws
            if (mWindowDrawCountDown != null) {
                try {
                    mWindowDrawCountDown.await();
                } catch (InterruptedException e) {
                    Log.e(mTag, "Window redraw count down interrupted!");
                }
                mWindowDrawCountDown = null;
            }

            if (mAttachInfo.mThreadedRenderer != null) {
                mAttachInfo.mThreadedRenderer.setStopped(mStopped);
            }
---------------------------------------------------------
对应Log：
04-16 13:08:37.113 V/ViewRootImpl[TestActivity]( 2602): FINISHED DRAWING: com.android.testred/com.android.testred.TestActivity
---------------------------------------------------------
            if (LOCAL_LOGV) {
                Log.v(mTag, "FINISHED DRAWING: " + mWindowAttributes.getTitle());
            }
            //绘制完成后会执行回调
            if (mSurfaceHolder != null && mSurface.isValid()) {
                SurfaceCallbackHelper sch = new SurfaceCallbackHelper(this::postDrawFinished);
                SurfaceHolder.Callback callbacks[] = mSurfaceHolder.getCallbacks();

                sch.dispatchSurfaceRedrawNeededAsync(mSurfaceHolder, callbacks);
            } else if (!usingAsyncReport) {
                if (mAttachInfo.mThreadedRenderer != null) {
                    mAttachInfo.mThreadedRenderer.fence();
                }
                pendingDrawFinished();
            }
        }
    }


   private boolean draw(boolean fullRedrawNeeded) {
        Surface surface = mSurface;
        if (!surface.isValid()) {
            return false;
        }

        if (DEBUG_FPS) {
            trackFPS();
        }

        if (!sFirstDrawComplete) {
            synchronized (sFirstDrawHandlers) {
                sFirstDrawComplete = true;
                final int count = sFirstDrawHandlers.size();
                for (int i = 0; i< count; i++) {
                    mHandler.post(sFirstDrawHandlers.get(i));
                }
            }
        }

        scrollToRectOrFocus(null, false);
        .....
        final Rect dirty = mDirty;
        if (mSurfaceHolder != null) {
            // The app owns the surface, we won't draw.
            dirty.setEmpty();
            ......
            return false;
        }

        if (fullRedrawNeeded) {
            mAttachInfo.mIgnoreDirtyState = true;
            dirty.set(0, 0, (int) (mWidth * appScale + 0.5f), (int) (mHeight * appScale + 0.5f));
        }
---------------------------------------------------------
对应Log：
04-16 13:08:37.074 V/ViewRootImpl[TestActivity]( 2602): Draw DecorView@515e1ca[TestActivity]/com.android.testred/com.android.testred.TestActivity: dirty={0,0,160,854} surface=Surface(name=null)/@0x266c9a5 surface.isValid()=true, appScale:1.0, width=160, height=854
---------------------------------------------------------
        if (DEBUG_ORIENTATION || DEBUG_DRAW) {
            Log.v(mTag, "Draw " + mView + "/"
                    + mWindowAttributes.getTitle()
                    + ": dirty={" + dirty.left + "," + dirty.top
                    + "," + dirty.right + "," + dirty.bottom + "} surface="
                    + surface + " surface.isValid()=" + surface.isValid() + ", appScale:" +
                    appScale + ", width=" + mWidth + ", height=" + mHeight);
        }

        mAttachInfo.mTreeObserver.dispatchOnDraw();

        .......
        return useAsyncReport;
    }

```

布局完成后接着Vsync又一次Trigger 。(画图)

GLSurfaceView绘制流程

**GLSurfaceView提供了下列特性：**
> 
> 1> 管理一个surface，这个surface就是一块特殊的内存，能直接排版到android的视图view上。
2> 管理一个EGL display，它能让opengl把内容渲染到上述的surface上。
3> 用户自定义渲染器(render)。
4> 让渲染器在独立的线程里运作，和UI线程分离。
5> 支持按需渲染(on-demand)和连续渲染(continuous)。
6> 一些可选工具，如调试。



**概念：**

> Display(EGLDisplay) 是对实际显示设备的抽象。
Surface（EGLSurface）是对用来存储图像的内存区域FrameBuffer的抽象，包括Color Buffer, Stencil Buffer ,Depth Buffer.
Context (EGLContext) 存储OpenGL ES绘图的一些状态信息。



**步骤：**

>获取EGLDisplay对象
初始化与EGLDisplay 之间的连接。
获取EGLConfig对象
创建EGLContext 实例
创建EGLSurface实例
连接EGLContext和EGLSurface.
使用GL指令绘制图形
断开并释放与EGLSurface关联的EGLContext对象
删除EGLSurface对象
删除EGLContext对象
终止与EGLDisplay之间的连接。



GLSurfaceView的主要绘制过程都是在一个子线程中完成，即整个绘制最终都是guardenRun()中完成。在这个过程中完成了整个EGL绘制的所有步骤。
``` java
/frameworks/base/opengl/java/android/opengl/GLSurfaceView.java
        private void guardedRun() throws InterruptedException {
            mEglHelper = new EglHelper(mGLSurfaceViewWeakRef);
            mHaveEglContext = false;
            mHaveEglSurface = false;
            mWantRenderNotification = false;

            try {
                GL10 gl = null;
                boolean createEglContext = false;
                boolean createEglSurface = false;
                boolean createGlInterface = false;
                boolean lostEglContext = false;
                boolean sizeChanged = false;
                boolean wantRenderNotification = false;
                boolean doRenderNotification = false;
                boolean askedToReleaseEglContext = false;
                int w = 0;
                int h = 0;
                Runnable event = null;
                Runnable finishDrawingRunnable = null;

                while (true) {
                    synchronized (sGLThreadManager) {
                        while (true) {
                            ......

                            ......
                            if (readyToDraw()) {

                                // If we don't have an EGL context, try to acquire one.
                                if (! mHaveEglContext) {
                                    if (askedToReleaseEglContext) {
                                        askedToReleaseEglContext = false;
                                    } else {
                                        try {
                                            mEglHelper.start();
                                        } catch (RuntimeException t) {
                                            ......
                                        }
                                        mHaveEglContext = true;
                                        createEglContext = true;

                                        sGLThreadManager.notifyAll();
                                    }
                                }

                                if (mHaveEglContext && !mHaveEglSurface) {
                                    mHaveEglSurface = true;
                                    createEglSurface = true;
                                    createGlInterface = true;
                                    sizeChanged = true;
                                }

                                if (mHaveEglSurface) {
                                    if (mSizeChanged) {
                                        sizeChanged = true;
                                        w = mWidth;
                                        h = mHeight;
                                        mWantRenderNotification = true;
                                        ......
                                        createEglSurface = true;

                                        mSizeChanged = false;
                                    }
                                    mRequestRender = false;
                                    ......
                                }
                            } else {
                                ......
                            }
                            .......
                            sGLThreadManager.wait();
                        }
                    } 
                    ......

                    if (createEglSurface) {
                        if (LOG_SURFACE) {
                            Log.w("GLThread", "egl createSurface");
                        }
                        if (mEglHelper.createSurface()) {
                            synchronized(sGLThreadManager) {
                                mFinishedCreatingEglSurface = true;
                                sGLThreadManager.notifyAll();
                            }
                        } else {
                            synchronized(sGLThreadManager) {
                                mFinishedCreatingEglSurface = true;
                                mSurfaceIsBad = true;
                                sGLThreadManager.notifyAll();
                            }
                            continue;
                        }
                        createEglSurface = false;
                    }

                    if (createGlInterface) {
                        gl = (GL10) mEglHelper.createGL();

                        createGlInterface = false;
                    }

                    if (createEglContext) {
                        if (LOG_RENDERER) {
                            Log.w("GLThread", "onSurfaceCreated");
                        }
                        GLSurfaceView view = mGLSurfaceViewWeakRef.get();
                        if (view != null) {
                            try {
                                
                                view.mRenderer.onSurfaceCreated(gl, mEglHelper.mEglConfig);
                            } finally {
                                
                            }
                        }
                        createEglContext = false;
                    }

                    if (sizeChanged) {
                        if (LOG_RENDERER) {
                            Log.w("GLThread", "onSurfaceChanged(" + w + ", " + h + ")");
                        }
                        GLSurfaceView view = mGLSurfaceViewWeakRef.get();
                        if (view != null) {
                            try {
                                Trace.traceBegin(Trace.TRACE_TAG_VIEW, "onSurfaceChanged");
                                view.mRenderer.onSurfaceChanged(gl, w, h);
                            } finally {
                                Trace.traceEnd(Trace.TRACE_TAG_VIEW);
                            }
                        }
                        sizeChanged = false;
                    }

                    if (LOG_RENDERER_DRAW_FRAME) {
                        Log.w("GLThread", "onDrawFrame tid=" + getId());
                    }
                    {
                        GLSurfaceView view = mGLSurfaceViewWeakRef.get();
                        if (view != null) {
                            try {
                                Trace.traceBegin(Trace.TRACE_TAG_VIEW, "onDrawFrame");
                                if (mFinishDrawingRunnable != null) {
                                    finishDrawingRunnable = mFinishDrawingRunnable;
                                    mFinishDrawingRunnable = null;
                                }
                                view.mRenderer.onDrawFrame(gl);
                                if (finishDrawingRunnable != null) {
                                    finishDrawingRunnable.run();
                                    finishDrawingRunnable = null;
                                }
                            } finally {
                                Trace.traceEnd(Trace.TRACE_TAG_VIEW);
                            }
                        }
                    }
                    int swapError = mEglHelper.swap();
                    .......
                }

            } finally {
                /*
                 * clean-up everything...
                 */
                synchronized (sGLThreadManager) {
                    stopEglSurfaceLocked();
                    stopEglContextLocked();
                }
            }
        }
```

我们直接看看GLSurfaceView绘制的过程，绘制过程opengl准备工作。
##### 5.1、EglHelper.start()
``` java
//GLSurfaceView 绘制流程分析
/frameworks/base/opengl/java/android/opengl/GLSurfaceView.java

    private static class EglHelper {
        public EglHelper(WeakReference<GLSurfaceView> glSurfaceViewWeakRef) {
            mGLSurfaceViewWeakRef = glSurfaceViewWeakRef;
        }

        public void start() {
            if (LOG_EGL) {
                Log.w("EglHelper", "start() tid=" + Thread.currentThread().getId());
            }

            mEgl = (EGL10) EGLContext.getEGL();

            mEglDisplay = mEgl.eglGetDisplay(EGL10.EGL_DEFAULT_DISPLAY);
            ......
            int[] version = new int[2];
            if(!mEgl.eglInitialize(mEglDisplay, version)) {
                ......
            }
            GLSurfaceView view = mGLSurfaceViewWeakRef.get();
            if (view == null) {
                mEglConfig = null;
                mEglContext = null;
            } else {
                mEglConfig = view.mEGLConfigChooser.chooseConfig(mEgl, mEglDisplay);

                mEglContext = view.mEGLContextFactory.createContext(mEgl, mEglDisplay, mEglConfig);
            }
            if (mEglContext == null || mEglContext == EGL10.EGL_NO_CONTEXT) {
                mEglContext = null;
                throwEglException("createContext");
            }
            ......
            }

            mEglSurface = null;
        }

        .......

        private WeakReference<GLSurfaceView> mGLSurfaceViewWeakRef;
        EGL10 mEgl;
        EGLDisplay mEglDisplay;
        EGLSurface mEglSurface;
        EGLConfig mEglConfig;
        EGLContext mEglContext;

    }

```
mEglHelper.start()就完成了4步：
- 1，获取EGLDisplay对象

- 2，初始化与EGLDisplay 之间的连接。
- 3，获取EGLConfig对象
- 4，创建EGLContext 实例

接下来createSurface()
##### 5.2、EglHelper.createSurface()

``` java

    private static class EglHelper {
        public EglHelper(WeakReference<GLSurfaceView> glSurfaceViewWeakRef) {
            mGLSurfaceViewWeakRef = glSurfaceViewWeakRef;
        }
        ......
        public boolean createSurface() {
            if (LOG_EGL) {
                Log.w("EglHelper", "createSurface()  tid=" + Thread.currentThread().getId());
            }
            ......
            GLSurfaceView view = mGLSurfaceViewWeakRef.get();
            if (view != null) {
                mEglSurface = view.mEGLWindowSurfaceFactory.createWindowSurface(mEgl,
                        mEglDisplay, mEglConfig, view.getHolder());
            } else {
                mEglSurface = null;
            }
            ......
            if (!mEgl.eglMakeCurrent(mEglDisplay, mEglSurface, mEglSurface, mEglContext)) {
                ......
                return false;
            }

            return true;
        }
        ......
        private WeakReference<GLSurfaceView> mGLSurfaceViewWeakRef;
        EGL10 mEgl;
        EGLDisplay mEglDisplay;
        EGLSurface mEglSurface;
        EGLConfig mEglConfig;
        EGLContext mEglContext;

    }

```
最终会调用：

``` cpp
/frameworks/native/opengl/libs/EGL/eglApi.cpp
EGLSurface eglCreateWindowSurface(  EGLDisplay dpy, EGLConfig config,
                                    NativeWindowType window,
                                    const EGLint *attrib_list)
{
    const EGLint *origAttribList = attrib_list;
    clearError();

    egl_connection_t* cnx = NULL;
    egl_display_ptr dp = validate_display_connection(dpy, cnx);
    if (dp) {
        ......
        int value = 0;
        window->query(window, NATIVE_WINDOW_IS_VALID, &value);
        ......

        int result = native_window_api_connect(window, NATIVE_WINDOW_API_EGL);
        ......
        }

        EGLDisplay iDpy = dp->disp.dpy;
        android_pixel_format format;
        getNativePixelFormat(iDpy, cnx, config, &format);

        // now select correct colorspace and dataspace based on user's attribute list
        EGLint colorSpace = EGL_UNKNOWN;
        std::vector<EGLint> strippedAttribList;
        if (!processAttributes(dp, window, format, attrib_list, &colorSpace,
                               &strippedAttribList)) {
           ......
        }
        attrib_list = strippedAttribList.data();

        {
            int err = native_window_set_buffers_format(window, format);
            ......
            }
        }

        android_dataspace dataSpace = dataSpaceFromEGLColorSpace(colorSpace);
        if (dataSpace != HAL_DATASPACE_UNKNOWN) {
            int err = native_window_set_buffers_data_space(window, dataSpace);
            ......
            }
        }

        // the EGL spec requires that a new EGLSurface default to swap interval
        // 1, so explicitly set that on the window here.
        ANativeWindow* anw = reinterpret_cast<ANativeWindow*>(window);
        anw->setSwapInterval(anw, 1);

        EGLSurface surface = cnx->egl.eglCreateWindowSurface(
                iDpy, config, window, attrib_list);
        if (surface != EGL_NO_SURFACE) {
            egl_surface_t* s =
                    new egl_surface_t(dp.get(), config, window, surface,
                                      getReportedColorSpace(colorSpace), cnx);
            return s;
        }

        // EGLSurface creation failed
        native_window_set_buffers_format(window, 0);
        native_window_api_disconnect(window, NATIVE_WINDOW_API_EGL);
    }
    return EGL_NO_SURFACE;
}

---------------------------------------------------------
对应Log：
 I/OpenGLRenderer( 2602): Initialized EGL, version 1.4
04-16 13:08:37.030 D/OpenGLRenderer( 2602): Swap behavior 2
04-16 13:08:37.040 V/Surface ( 2602): Surface::query
04-16 13:08:37.040 V/Surface ( 2602): Surface::connect
04-16 13:08:37.041 V/BufferQueueProducer(  739): [SurfaceView - com.android.testred/com.android.testred.TestActivity#0] connect: api=1 producerControlledByApp=false
04-16 13:08:37.042 V/Surface ( 2602): Surface::setBuffersFormat
04-16 13:08:37.042 V/BufferQueueProducer(  739): [SurfaceView - com.android.testred/com.android.testred.TestActivity#0] setAsyncMode: async = 0
04-16 13:08:37.042 V/Surface ( 2602): Surface::query
04-16 13:08:37.042 V/BufferQueueProducer(  739): [SurfaceView - com.android.testred/com.android.testred.TestActivity#0] query: 10? 2304
04-16 13:08:37.043 V/Surface ( 2602): Surface::setUsage
04-16 13:08:37.043 V/Surface ( 2602): Surface::query
04-16 13:08:37.043 I/chatty  ( 2602): uid=10088(com.android.testred) GLThread 199 identical 1 line
04-16 13:08:37.043 V/Surface ( 2602): Surface::query
04-16 13:08:37.043 V/BufferQueueProducer(  739): [SurfaceView - com.android.testred/com.android.testred.TestActivity#0] query: 10? 2304
04-16 13:08:37.043 V/Surface ( 2602): Surface::query
04-16 13:08:37.044 V/BufferQueueProducer(  739): [com.android.testred/com.android.testred.TestActivity#0] connect: api=1 producerControlledByApp=true
04-16 13:08:37.045 V/Surface ( 2602): Surface::query
04-16 13:08:37.045 V/BufferQueueProducer(  739): [com.android.testred/com.android.testred.TestActivity#0] query: 10? 2304
04-16 13:08:37.046 V/Surface ( 2602): Surface::query
04-16 13:08:37.047 V/Surface ( 2602): Surface::setBuffersDimensions
---------------------------------------------------------

```
##### 5.3、 mEgl.eglMakeCurrent()
现在用于绘制图像的Surface已经创建好啦

``` cpp
  -> eglMakeCurrent 
    -> eglMakeCurrent(OEM EGL) 
      -> egl_window_surface_v2_t::connect()
      -> Surface::hook_dequeueBuffer() 
        -> Surface::dequeueBuffer() 
            -> BpGraphicBufferProducer::dequeueBuffer()
            -> BpGraphicBufferProducer::requestBuffer()

对应log：
---------------------------------------------------------
对应Log：
04-16 13:08:37.047 V/Surface ( 2602): Surface::dequeueBuffer
04-16 13:08:37.047 V/BufferQueueProducer(  739): [SurfaceView - com.android.testred/com.android.testred.TestActivity#0] dequeueBuffer: w=160 h=816 format=0x2, usage=0x10000900
04-16 13:08:37.047 V/BufferQueueProducer(  739): [SurfaceView - com.android.testred/com.android.testred.TestActivity#0] dequeueBuffer: setting buffer age to 0
04-16 13:08:37.047 V/BufferQueueProducer(  739): [SurfaceView - com.android.testred/com.android.testred.TestActivity#0] dequeueBuffer: allocating a new buffer for slot 0

04-16 13:08:37.048 V/BufferQueueProducer(  739): [SurfaceView - com.android.testred/com.android.testred.TestActivity#0] dequeueBuffer: returning slot=0/0 buf=0x7075427380 flags=0x1

04-16 13:08:37.048 V/BufferQueueProducer(  739): [SurfaceView - com.android.testred/com.android.testred.TestActivity#0] requestBuffer: slot 0


04-16 13:08:37.059 V/Surface ( 2602): Surface::setBuffersTransform
04-16 13:08:37.059 V/Surface ( 2602): Surface::queueBuffer
04-16 13:08:37.059 V/Surface ( 2602): Surface::queueBuffer making up timestamp: 83377.24 ms
04-16 13:08:37.060 V/BufferQueueProducer(  739): [SurfaceView - com.android.testred/com.android.testred.TestActivity#0] 
04-16 13:08:37.060 V/ConsumerBase(  739): [SurfaceView - com.android.testred/com.android.testred.TestActivity#0] onFrameAvailable
04-16 13:08:37.060 V/ConsumerBase(  739): [SurfaceView - com.android.testred/com.android.testred.TestActivity#0] actually calling onFrameAvailable
04-16 13:08:37.060 V/BufferQueueProducer(  739): [SurfaceView - com.android.testred/com.android.testred.TestActivity#0] addAndGetFrameTimestamps

04-16 13:08:37.060 V/BufferQueueProducer(  739): [SurfaceView - com.android.testred/com.android.testred.TestActivity#0] query: 11? 0
04-16 13:08:37.060 V/BufferLayerConsumer(  739): [SurfaceView - com.android.testred/com.android.testred.TestActivity#0] updateTexImage
04-16 13:08:37.060 V/BufferQueueConsumer(  739): [SurfaceView - com.android.testred/com.android.testred.TestActivity#0] acquireBuffer: accept desire=83377236737 expect=83393818427 (-16581690) now=83378380070
04-16 13:08:37.060 V/BufferQueueConsumer(  739): [SurfaceView - com.android.testred/com.android.testred.TestActivity#0] acquireBuffer: acquiring { slot=0/1 buffer=0x7075427380 }
04-16 13:08:37.060 V/ConsumerBase(  739): [SurfaceView - com.android.testred/com.android.testred.TestActivity#0] acquireBufferLocked: -> slot=0/1
04-16 13:08:37.060 V/BufferLayerConsumer(  739): [SurfaceView - com.android.testred/com.android.testred.TestActivity#0] syncForReleaseLocked
04-16 13:08:37.060 V/BufferLayerConsumer(  739): [SurfaceView - com.android.testred/com.android.testred.TestActivity#0] updateAndRelease: (slot=-1 buf=0x0) -> (slot=0 buf=0x7075427380)
04-16 13:08:37.060 V/BufferLayerConsumer(  739): [SurfaceView - com.android.testred/com.android.testred.TestActivity#0] computeCurrentTransformMatrixLocked
04-16 13:08:37.061 V/BufferLayerConsumer(  739): [SurfaceView - com.android.testred/com.android.testred.TestActivity#0] getFrameNumber
04-16 13:08:37.061 V/BufferLayerConsumer(  739): [SurfaceView - com.android.testred/com.android.testred.TestActivity#0] getFrameNumber
04-16 13:08:37.061 V/BufferLayerConsumer(  739): [SurfaceView - com.android.testred/com.android.testred.TestActivity#0]
---------------------------------------------------------

```
此时可以看到进行一了一次queueBuffer()操作。此时由于还没有进行绘制操作，所以屏幕肯定是没有显示的。

##### 5.4、Renderer.onDrawFrame()

``` java
/packages/apps/TestViewportRed/src/com/android/testred/TestView.java
class TestView extends GLSurfaceView {

    TestView(Context context) {
        super(context);
        init();
    }

    public TestView(Context context, AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    private void init() {
        setRenderer(new Renderer());
        setRenderMode(RENDERMODE_WHEN_DIRTY);
    }

    private class Renderer implements GLSurfaceView.Renderer {
        private static final String TAG = "Renderer";
			@Override
			public void onSurfaceCreated(GL10 gl, EGLConfig config) {
				gl.glClearColor(1f, 0f, 0f, 0f);
			}
		
			@Override
			public void onSurfaceChanged(GL10 gl, int width, int height) {
				gl.glViewport(0, 0, width, height);
			}
		
			@Override
			public void onDrawFrame(GL10 gl) {
				gl.glClear(GL10.GL_COLOR_BUFFER_BIT);
			}
    }
}

```

此时图像数据已经用OpenGl绘制好了，一块红色的区域（160x854）;


##### 5.5、finishDrawingRunnable.run()

接着会执行ViewRootImpl.java 的 postDrawFinished() 方法

``` java
/frameworks/base/core/java/android/view/ViewRootImpl.java

    void pendingDrawFinished() {
        if (mDrawsNeededToReport == 0) {
            throw new RuntimeException("Unbalanced drawPending/pendingDrawFinished calls");
        }
        mDrawsNeededToReport--;
        if (mDrawsNeededToReport == 0) {
            reportDrawFinished();
        }
    }

    private void postDrawFinished() {
        mHandler.sendEmptyMessage(MSG_DRAW_FINISHED);
    }

    private void reportDrawFinished() {
        try {
            mDrawsNeededToReport = 0;
            mWindowSession.finishDrawing(mWindow);
        } catch (RemoteException e) {
            // Have fun!
        }
    }

```
mWindowSession.finishDrawing(mWindow) 通过binder通信进入

WMS。

``` java
/frameworks/base/services/core/java/com/android/server/wm/Session.java

    @Override
    public void finishDrawing(IWindow window) {
        if (WindowManagerService.localLOGV) Slog.v(
            TAG_WM, "IWindow finishDrawing called for " + window);
        mService.finishDrawingWindow(this, window);
    }

/frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java

    void finishDrawingWindow(Session session, IWindow client) {
        final long origId = Binder.clearCallingIdentity();
        try {
            synchronized (mWindowMap) {
                WindowState win = windowForClientLocked(session, client, false);
                if (DEBUG_ADD_REMOVE) Slog.d(TAG_WM, "finishDrawingWindow: " + win + " mDrawState="
                        + (win != null ? win.mWinAnimator.drawStateToString() : "null"));
                if (win != null && win.mWinAnimator.finishDrawingLocked()) {
                    if ((win.mAttrs.flags & FLAG_SHOW_WALLPAPER) != 0) {
                        win.getDisplayContent().pendingLayoutChanges |=
                                WindowManagerPolicy.FINISH_LAYOUT_REDO_WALLPAPER;
                    }
                    win.setDisplayLayoutNeeded();
                    mWindowPlacerLocked.requestTraversal();
                }
            }
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }


/frameworks/base/services/core/java/com/android/server/wm/WindowState.java
    void setDisplayLayoutNeeded() {
        final DisplayContent dc = getDisplayContent();
        if (dc != null) {
            dc.setLayoutNeeded();
        }
    }

/frameworks/base/services/core/java/com/android/server/wm/DisplayContent.java
    void setLayoutNeeded() {
        if (DEBUG_LAYOUT) Slog.w(TAG_WM, "setLayoutNeeded: callers=" + Debug.getCallers(3));
        mLayoutNeeded = true;
    }


---------------------------------------------------------
对应Log：
04-16 13:08:37.115 V/WindowManager( 1049): IWindow finishDrawing called for android.view.IWindow$Stub$Proxy@44816e8
04-16 13:08:37.115 V/WindowManager( 1049): Looking up client android.os.BinderProxy@b9baf64: Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity}
04-16 13:08:37.115 D/WindowManager( 1049): finishDrawingWindow: Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity} mDrawState=DRAW_PENDING
04-16 13:08:37.115 V/WindowStateAnimator( 1049): finishDrawingLocked: mDrawState=COMMIT_DRAW_PENDING Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity} in Surface(name=com.android.testred/com.android.testred.TestActivity)/@0xb17b0ce
04-16 13:08:37.115 W/WindowManager( 1049): setLayoutNeeded: callers=com.android.server.wm.WindowState.setDisplayLayoutNeeded:2250 com.android.server.wm.WindowManagerService.finishDrawingWindow:2305 com.android.server.wm.Session.finishDrawing:281 
---------------------------------------------------------
```

##### 5.6、mEglHelper.swap()
swap()最终会调用swapBuffers()函数的queueBuffer()通知SurfaceFlinger合成。

``` cpp
[->\frameworks\native\opengl\libagl\egl.cpp]
EGLBoolean egl_window_surface_v2_t::swapBuffers()
{
......
nativeWindow->queueBuffer(nativeWindow, buffer); 
nativeWindow->dequeueBuffer(nativeWindow, &buffer);
......
}


---------------------------------------------------------
看看Log：
04-16 13:08:37.112 V/Surface ( 2602): Surface::queueBuffer
04-16 13:08:37.112 V/Surface ( 2602): Surface::queueBuffer making up timestamp: 83429.66 ms
04-16 13:08:37.112 V/BufferQueueProducer(  739): [com.android.testred/com.android.testred.TestActivity#0] queueBuffer: slot=0/1 time=83429663560 dataSpace=0 validHdrMetadataTypes=0x0 crop=[0,0,0,0] transform=0 scale=FREEZE
04-16 13:08:37.112 V/ConsumerBase(  739): [com.android.testred/com.android.testred.TestActivity#0] onFrameAvailable
04-16 13:08:37.112 V/ConsumerBase(  739): [com.android.testred/com.android.testred.TestActivity#0] actually calling onFrameAvailable
04-16 13:08:37.112 V/BufferQueueProducer(  739): [com.android.testred/com.android.testred.TestActivity#0] addAndGetFrameTimestamps
---------------------------------------------------------
```

##### 5.7、WindowStateAnimator.commitFinishDrawingLocked()

``` java
---------------------------------------------------------
看看Log：
04-16 13:08:37.116 V/RootWindowContainer( 1049): performSurfacePlacementInner: entry. Called by com.android.server.wm.WindowSurfacePlacer.performSurfacePlacementLoop:207 com.android.server.wm.WindowSurfacePlacer.performSurfacePlacement:155 com.android.server.wm.WindowSurfacePlacer.performSurfacePlacement:145 
04-16 13:08:37.116 I/RootWindowContainer( 1049): >>> OPEN TRANSACTION performLayoutAndPlaceSurfaces
04-16 13:08:37.116 W/WindowManager( 1049): clearLayoutNeeded: callers=com.android.server.wm.DisplayContent.performLayout:2967 com.android.server.wm.DisplayContent.applySurfaceChangesTransaction:2880 com.android.server.wm.RootWindowContainer.applySurfaceChangesTransaction:852 
04-16 13:08:37.116 V/DisplayContent( 1049): -------------------------------------
04-16 13:08:37.116 V/DisplayContent( 1049): performLayout: needed=false dw=480 dh=854
---------------------------------------------------------
```

boolean applySurfaceChangesTransaction(boolean recoveringMemory)
方法中
performLayout(repeats == 1, false /* updateInputWindows */);进行布局

``` java
04-16 13:08:37.118 V/DisplayContent( 1049): 1ST PASS Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity}: gone=false mHaveFrame=true mLayoutAttached=false screen changed=false
04-16 13:08:37.118 V/DisplayContent( 1049):   VIS: mViewVisibility=0 mRelayoutCalled=true hidden=true hiddenRequested=false parentHidden=false
04-16 13:08:37.118 V/WindowManager( 1049): layoutWindowLw(com.android.testred/com.android.testred.TestActivity): IN_SCREEN, INSET_DECOR
04-16 13:08:37.118 V/WindowManager( 1049): Compute frame com.android.testred/com.android.testred.TestActivity: sim=#120 attach=null type=1 flags=0x01810100 pf=[0,0][160,854] df=[0,0][160,854] of=[0,0][160,854] cf=[0,0][160,854] vf=[0,36][480,782] dcf=[0,36][480,782] sf=[0,36][480,782] osf=null
```
接着看看boolean applySurfaceChangesTransaction(boolean recoveringMemory)
方法中的
 forAllWindows(mApplySurfaceChangesTransaction, true /* traverseTopToBottom */);

``` java
/frameworks/base/services/core/java/com/android/server/wm/DisplayContent.java

    private final Consumer<WindowState> mApplySurfaceChangesTransaction = w -> {
        final WindowSurfacePlacer surfacePlacer = mService.mWindowPlacerLocked;
        final boolean obscuredChanged = w.mObscured !=
                mTmpApplySurfaceChangesTransactionState.obscured;
        final RootWindowContainer root = mService.mRoot;
        // Only used if default window
        final boolean someoneLosingFocus = !mService.mLosingFocus.isEmpty();

        // Update effect.
        w.mObscured = mTmpApplySurfaceChangesTransactionState.obscured;
        if (!mTmpApplySurfaceChangesTransactionState.obscured) {
            final boolean isDisplayed = w.isDisplayedLw();

            if (isDisplayed && w.isObscuringDisplay()) {
                // This window completely covers everything behind it, so we want to leave all
                // of them as undimmed (for performance reasons).
                root.mObscuringWindow = w;
                mTmpApplySurfaceChangesTransactionState.obscured = true;
            }

            mTmpApplySurfaceChangesTransactionState.displayHasContent |=
                    root.handleNotObscuredLocked(w,
                            mTmpApplySurfaceChangesTransactionState.obscured,
                            mTmpApplySurfaceChangesTransactionState.syswin);

            if (w.mHasSurface && isDisplayed) {
                final int type = w.mAttrs.type;
                if (type == TYPE_SYSTEM_DIALOG || type == TYPE_SYSTEM_ERROR
                        || (w.mAttrs.privateFlags & PRIVATE_FLAG_KEYGUARD) != 0) {
                    mTmpApplySurfaceChangesTransactionState.syswin = true;
                }
                if (mTmpApplySurfaceChangesTransactionState.preferredRefreshRate == 0
                        && w.mAttrs.preferredRefreshRate != 0) {
                    mTmpApplySurfaceChangesTransactionState.preferredRefreshRate
                            = w.mAttrs.preferredRefreshRate;
                }
                if (mTmpApplySurfaceChangesTransactionState.preferredModeId == 0
                        && w.mAttrs.preferredDisplayModeId != 0) {
                    mTmpApplySurfaceChangesTransactionState.preferredModeId
                            = w.mAttrs.preferredDisplayModeId;
                }
            }
        }

        if (isDefaultDisplay && obscuredChanged && w.isVisibleLw()
                && mWallpaperController.isWallpaperTarget(w)) {
            // This is the wallpaper target and its obscured state changed... make sure the
            // current wallpaper's visibility has been updated accordingly.
            mWallpaperController.updateWallpaperVisibility();
        }

        w.handleWindowMovedIfNeeded();

        final WindowStateAnimator winAnimator = w.mWinAnimator;

        //Slog.i(TAG, "Window " + this + " clearing mContentChanged - done placing");
        w.mContentChanged = false;

        // Moved from updateWindowsAndWallpaperLocked().
        if (w.mHasSurface) {
            // Take care of the window being ready to display.
            final boolean committed = winAnimator.commitFinishDrawingLocked();
            if (isDefaultDisplay && committed) {
                if (w.mAttrs.type == TYPE_DREAM) {
                    // HACK: When a dream is shown, it may at that point hide the lock screen.
                    // So we need to redo the layout to let the phone window manager make this
                    // happen.
                    pendingLayoutChanges |= FINISH_LAYOUT_REDO_LAYOUT;
                    if (DEBUG_LAYOUT_REPEATS) {
                        surfacePlacer.debugLayoutRepeats(
                                "dream and commitFinishDrawingLocked true",
                                pendingLayoutChanges);
                    }
                }
                if ((w.mAttrs.flags & FLAG_SHOW_WALLPAPER) != 0) {
                    if (DEBUG_WALLPAPER_LIGHT) Slog.v(TAG,
                            "First draw done in potential wallpaper target " + w);
                    root.mWallpaperMayChange = true;
                    pendingLayoutChanges |= FINISH_LAYOUT_REDO_WALLPAPER;
                    if (DEBUG_LAYOUT_REPEATS) {
                        surfacePlacer.debugLayoutRepeats(
                                "wallpaper and commitFinishDrawingLocked true",
                                pendingLayoutChanges);
                    }
                }
            }
        }

        final AppWindowToken atoken = w.mAppToken;
        if (atoken != null) {
            atoken.updateLetterboxSurface(w);
            final boolean updateAllDrawn = atoken.updateDrawnWindowStates(w);
            if (updateAllDrawn && !mTmpUpdateAllDrawn.contains(atoken)) {
                mTmpUpdateAllDrawn.add(atoken);
            }
        }

        if (isDefaultDisplay && someoneLosingFocus && w == mService.mCurrentFocus
                && w.isDisplayedLw()) {
            mTmpApplySurfaceChangesTransactionState.focusDisplayed = true;
        }

        w.updateResizingWindowIfNeeded();
    };
```

``` java
/frameworks/base/services/core/java/com/android/server/wm/WindowStateAnimator.java

    // This must be called while inside a transaction.
    boolean commitFinishDrawingLocked() {
        if (DEBUG_STARTING_WINDOW_VERBOSE &&
                mWin.mAttrs.type == WindowManager.LayoutParams.TYPE_APPLICATION_STARTING) {
            Slog.i(TAG, "commitFinishDrawingLocked: " + mWin + " cur mDrawState="
                    + drawStateToString());
        }
        if (mDrawState != COMMIT_DRAW_PENDING && mDrawState != READY_TO_SHOW) {
            return false;
        }
---------------------------------------------------------
看看Log：
04-16 13:08:37.123 I/WindowStateAnimator( 1049): commitFinishDrawingLocked: mDrawState=READY_TO_SHOW Surface(name=com.android.testred/com.android.testred.TestActivity)/@0xb17b0ce
---------------------------------------------------------
        if (DEBUG_ANIM) {
            Slog.i(TAG, "commitFinishDrawingLocked: mDrawState=READY_TO_SHOW " + mSurfaceController);
        }
        mDrawState = READY_TO_SHOW;
        boolean result = false;
        final AppWindowToken atoken = mWin.mAppToken;
        if (atoken == null || atoken.allDrawn || mWin.mAttrs.type == TYPE_APPLICATION_STARTING) {
            result = mWin.performShowLocked();
        }
        return result;
    }
```

###### 5.7.1、AppWindowToken.updateDrawnWindowStates()

``` java
/frameworks/base/services/core/java/com/android/server/wm/AppWindowToken.java
---------------------------------------------------------
看看Log：
04-16 13:08:37.124 V/AppWindowToken( 1049): Eval win Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity}: isDrawn=true, isAnimationSet=true
04-16 13:08:37.124 V/AppWindowToken( 1049): tokenMayBeDrawn: AppWindowToken{1579134 token=Token{ce2ac07 ActivityRecord{65cc87a u0 com.android.testred/.TestActivity t5}}} w=Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity} numInteresting=1 freezingScreen=false mAppFreezing=false
---------------------------------------------------------


   boolean updateDrawnWindowStates(WindowState w) {
        w.setDrawnStateEvaluated(true /*evaluated*/);

        if (DEBUG_STARTING_WINDOW_VERBOSE && w == startingWindow) {
            Slog.d(TAG, "updateWindows: starting " + w + " isOnScreen=" + w.isOnScreen()
                    + " allDrawn=" + allDrawn + " freezingScreen=" + mFreezingScreen);
        }

        if (allDrawn && !mFreezingScreen) {
            return false;
        }

        if (mLastTransactionSequence != mService.mTransactionSequence) {
            mLastTransactionSequence = mService.mTransactionSequence;
            mNumDrawnWindows = 0;
            startingDisplayed = false;

            // There is the main base application window, even if it is exiting, wait for it
            mNumInterestingWindows = findMainWindow(false /* includeStartingApp */) != null ? 1 : 0;
        }

        final WindowStateAnimator winAnimator = w.mWinAnimator;

        boolean isInterestingAndDrawn = false;

        if (!allDrawn && w.mightAffectAllDrawn()) {
            if (DEBUG_VISIBILITY || DEBUG_ORIENTATION) {
                Slog.v(TAG, "Eval win " + w + ": isDrawn=" + w.isDrawnLw()
                        + ", isAnimationSet=" + isSelfAnimating());
                if (!w.isDrawnLw()) {
                    Slog.v(TAG, "Not displayed: s=" + winAnimator.mSurfaceController
                            + " pv=" + w.mPolicyVisibility
                            + " mDrawState=" + winAnimator.drawStateToString()
                            + " ph=" + w.isParentWindowHidden() + " th=" + hiddenRequested
                            + " a=" + isSelfAnimating());
                }
            }

            if (w != startingWindow) {
                if (w.isInteresting()) {
                    // Add non-main window as interesting since the main app has already been added
                    if (findMainWindow(false /* includeStartingApp */) != w) {
                        mNumInterestingWindows++;
                    }
                    if (w.isDrawnLw()) {
                        mNumDrawnWindows++;

                        if (DEBUG_VISIBILITY || DEBUG_ORIENTATION) Slog.v(TAG, "tokenMayBeDrawn: "
                                + this + " w=" + w + " numInteresting=" + mNumInterestingWindows
                                + " freezingScreen=" + mFreezingScreen
                                + " mAppFreezing=" + w.mAppFreezing);

                        isInterestingAndDrawn = true;
                    }
                }
            } else if (w.isDrawnLw()) {
                if (getController() != null) {
                    getController().reportStartingWindowDrawn();
                }
                startingDisplayed = true;
            }
        }

        return isInterestingAndDrawn;
    }
```

##### 5.8、DisplayContent.prepareSurfaces()

``` java
/frameworks/base/services/core/java/com/android/server/wm/DisplayContent.java
    @Override
    void prepareSurfaces() {
        ......
        super.prepareSurfaces();
    }

/frameworks/base/services/core/java/com/android/server/wm/WindowContainer.java
    void prepareSurfaces() {
        SurfaceControl.mergeToGlobalTransaction(getPendingTransaction());

        // If a leash has been set when the transaction was committed, then the leash reparent has
        // been committed.
        mCommittedReparentToAnimationLeash = mSurfaceAnimator.hasLeash();
        for (int i = 0; i < mChildren.size(); i++) {
            mChildren.get(i).prepareSurfaces();
        }
    }

/frameworks/base/services/core/java/com/android/server/wm/WindowState.java
    @Override
    void prepareSurfaces() {
        final Dimmer dimmer = getDimmer();
        mIsDimming = false;
        if (dimmer != null) {
            applyDims(dimmer);
        }
        updateSurfacePosition();

        mWinAnimator.prepareSurfaceLocked(true);
        super.prepareSurfaces();
    }

/frameworks/base/services/core/java/com/android/server/wm/WindowStateAnimator.java

    void prepareSurfaceLocked(final boolean recoveringMemory) {
        final WindowState w = mWin;
        if (!hasSurface()) {

            // There is no need to wait for an animation change if our window is gone for layout
            // already as we'll never be visible.
            if (w.getOrientationChanging() && w.isGoneForLayoutLw()) {
                if (DEBUG_ORIENTATION) {
                    Slog.v(TAG, "Orientation change skips hidden " + w);
                }
                w.setOrientationChanging(false);
            }
            return;
        }

        boolean displayed = false;

        computeShownFrameLocked();

        setSurfaceBoundariesLocked(recoveringMemory);

        if (mIsWallpaper && !w.mWallpaperVisible) {
            // Wallpaper is no longer visible and there is no wp target => hide it.
            hide("prepareSurfaceLocked");
        } else if (w.isParentWindowHidden() || !w.isOnScreen()) {
            hide("prepareSurfaceLocked");
            mWallpaperControllerLocked.hideWallpapers(w);

            // If we are waiting for this window to handle an orientation change. If this window is
            // really hidden (gone for layout), there is no point in still waiting for it.
            // Note that this does introduce a potential glitch if the window becomes unhidden
            // before it has drawn for the new orientation.
            if (w.getOrientationChanging() && w.isGoneForLayoutLw()) {
                w.setOrientationChanging(false);
                if (DEBUG_ORIENTATION) Slog.v(TAG,
                        "Orientation change skips hidden " + w);
            }
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

---------------------------------------------------------
看看Log：
04-16 13:08:37.124 I/WindowManager( 1049):   SURFACE controller=Surface(name=com.android.testred/com.android.testred.TestActivity)/@0xb17b0cealpha=1.0 layer=0 matrix=[1.0*1.0,0.0*1.0][0.0*1.0,1.0*1.0]: Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity}
---------------------------------------------------------
            if (SHOW_TRANSACTIONS) WindowManagerService.logSurface(w,
                    "controller=" + mSurfaceController +
                    "alpha=" + mShownAlpha + " layer=" + mAnimLayer
                    + " matrix=[" + mDsDx + "*" + w.mHScale
                    + "," + mDtDx + "*" + w.mVScale
                    + "][" + mDtDy + "*" + w.mHScale
                    + "," + mDsDy + "*" + w.mVScale + "]", false);

            boolean prepared =
                mSurfaceController.prepareToShowInTransaction(mShownAlpha,
                        mDsDx * w.mHScale * mExtraHScale,
                        mDtDx * w.mVScale * mExtraVScale,
                        mDtDy * w.mHScale * mExtraHScale,
                        mDsDy * w.mVScale * mExtraVScale,
                        recoveringMemory);

            if (prepared && mDrawState == HAS_DRAWN) {
                if (mLastHidden) {
                    if (showSurfaceRobustlyLocked()) {
                        markPreservedSurfaceForDestroy();
                        mAnimator.requestRemovalOfReplacedWindows(w);
                        mLastHidden = false;
                        if (mIsWallpaper) {
                            w.dispatchWallpaperVisibility(true);
                        }
                        // This draw means the difference between unique content and mirroring.
                        // Run another pass through performLayout to set mHasContent in the
                        // LogicalDisplay.
                        mAnimator.setPendingLayoutChanges(w.getDisplayId(),
                                FINISH_LAYOUT_REDO_ANIM);
                        if (DEBUG_LAYOUT_REPEATS) {
                            mService.mWindowPlacerLocked.debugLayoutRepeats(
                                    "showSurfaceRobustlyLocked " + w,
                                    mAnimator.getPendingLayoutChanges(w.getDisplayId()));
                        }
                    } else {
                        w.setOrientationChanging(false);
                    }
                }
            }
            if (hasSurface()) {
                w.mToken.hasVisible = true;
            }
        } else {
            if (DEBUG_ANIM && isAnimationSet()) {
                Slog.v(TAG, "prepareSurface: No changes in animation for " + this);
            }
            displayed = true;
        }

        if (w.getOrientationChanging()) {
            if (!w.isDrawnLw()) {
                mAnimator.mBulkUpdateParams &= ~SET_ORIENTATION_CHANGE_COMPLETE;
                mAnimator.mLastWindowFreezeSource = w;
                if (DEBUG_ORIENTATION) Slog.v(TAG,
                        "Orientation continue waiting for draw in " + w);
            } else {
                w.setOrientationChanging(false);
                if (DEBUG_ORIENTATION) Slog.v(TAG, "Orientation change complete in " + w);
            }
        }

        if (displayed) {
            w.mToken.hasVisible = true;
        }
    }
```
##### 5.8.1、WindowStateAnimator.computeShownFrameLocked()


``` java
/frameworks/base/services/core/java/com/android/server/wm/WindowStateAnimator.java

   void computeShownFrameLocked() {
        final int displayId = mWin.getDisplayId();
        final ScreenRotationAnimation screenRotationAnimation =
                mAnimator.getScreenRotationAnimationLocked(displayId);
        final boolean screenAnimation =
                screenRotationAnimation != null && screenRotationAnimation.isAnimating();

        if (screenAnimation) {
            // cache often used attributes locally
            final Rect frame = mWin.mFrame;
            final float tmpFloats[] = mService.mTmpFloats;
            final Matrix tmpMatrix = mWin.mTmpMatrix;

            // Compute the desired transformation.
            if (screenRotationAnimation.isRotating()) {
                final float w = frame.width();
                final float h = frame.height();
                if (w>=1 && h>=1) {
                    tmpMatrix.setScale(1 + 2/w, 1 + 2/h, w/2, h/2);
                } else {
                    tmpMatrix.reset();
                }
            } else {
                tmpMatrix.reset();
            }

            tmpMatrix.postScale(mWin.mGlobalScale, mWin.mGlobalScale);
            ......
         tmpMatrix.postTranslate(mWin.mAttrs.surfaceInsets.left,
                    mWin.mAttrs.surfaceInsets.top);
            ......
            mHaveMatrix = true;
            tmpMatrix.getValues(tmpFloats);
            mDsDx = tmpFloats[Matrix.MSCALE_X];
            mDtDx = tmpFloats[Matrix.MSKEW_Y];
            mDtDy = tmpFloats[Matrix.MSKEW_X];
            mDsDy = tmpFloats[Matrix.MSCALE_Y];
            ......
            mShownAlpha = mAlpha;
            if (!mService.mLimitedAlphaCompositing
                    || (!PixelFormat.formatHasAlpha(mWin.mAttrs.format)
                    || (mWin.isIdentityMatrix(mDsDx, mDtDx, mDtDy, mDsDy)))) {
                //Slog.i(TAG_WM, "Applying alpha transform");
                if (screenAnimation) {
                    mShownAlpha *= screenRotationAnimation.getEnterTransformation().getAlpha();
                }
            } else {
                //Slog.i(TAG_WM, "Not applying alpha transform");
            }

            if ((DEBUG_ANIM || WindowManagerService.localLOGV)
                    && (mShownAlpha == 1.0 || mShownAlpha == 0.0)) Slog.v(
                    TAG, "computeShownFrameLocked: Animating " + this + " mAlpha=" + mAlpha
                    + " screen=" + (screenAnimation ?
                            screenRotationAnimation.getEnterTransformation().getAlpha() : "null"));
            return;
        } else if (mIsWallpaper && mService.mRoot.mWallpaperActionPending) {
            return;
        } else if (mWin.isDragResizeChanged()) {
            return;
        }
---------------------------------------------------------
看看Log：
04-16 13:08:37.124 V/WindowStateAnimator( 1049): computeShownFrameLocked: WindowStateAnimator{4fb81c9 com.android.testred/com.android.testred.TestActivity} not attached, mAlpha=1.0
---------------------------------------------------------

        if (WindowManagerService.localLOGV) Slog.v(
                TAG, "computeShownFrameLocked: " + this +
                " not attached, mAlpha=" + mAlpha);

        mShownAlpha = mAlpha;
        mHaveMatrix = false;
        mDsDx = mWin.mGlobalScale;
        mDtDx = 0;
        mDtDy = 0;
        mDsDy = mWin.mGlobalScale;
    }
```

##### 5.8.2、WindowStateAnimator.setSurfaceBoundariesLocked()

``` java
/frameworks/base/services/core/java/com/android/server/wm/WindowStateAnimator.java

   void setSurfaceBoundariesLocked(final boolean recoveringMemory) {
        ......
        final WindowState w = mWin;
        final LayoutParams attrs = mWin.getAttrs();
        final Task task = w.getTask();

        mTmpSize.set(0, 0, 0, 0);
        calculateSurfaceBounds(w, attrs);

        mExtraHScale = (float) 1.0;
        mExtraVScale = (float) 1.0;

        boolean wasForceScaled = mForceScaleUntilResize;
        boolean wasSeamlesslyRotated = w.mSeamlesslyRotated;

        ......
        final boolean relayout = !w.mRelayoutCalled || w.mInRelayout;
        if (relayout) {
            mSurfaceResized = mSurfaceController.setSizeInTransaction(
                    mTmpSize.width(), mTmpSize.height(), recoveringMemory);
        } else {
            mSurfaceResized = false;
        }
        mForceScaleUntilResize = mForceScaleUntilResize && !mSurfaceResized;
        .......
        mService.markForSeamlessRotation(w, w.mSeamlesslyRotated && !mSurfaceResized);

        Rect clipRect = null;
        if (calculateCrop(mTmpClipRect)) {
            clipRect = mTmpClipRect;
        }

        float surfaceWidth = mSurfaceController.getWidth();
        float surfaceHeight = mSurfaceController.getHeight();

        final Rect insets = attrs.surfaceInsets;

        if (isForceScaled()) {
            int hInsets = insets.left + insets.right;
            int vInsets = insets.top + insets.bottom;
            float surfaceContentWidth = surfaceWidth - hInsets;
            float surfaceContentHeight = surfaceHeight - vInsets;
            if (!mForceScaleUntilResize) {
                mSurfaceController.forceScaleableInTransaction(true);
            }

            int posX = 0;
            int posY = 0;
            task.mStack.getDimBounds(mTmpStackBounds);

            boolean allowStretching = false;
            task.mStack.getFinalAnimationSourceHintBounds(mTmpSourceBounds);
            ......
            if (mTmpSourceBounds.isEmpty() && (mWin.mLastRelayoutContentInsets.width() > 0
                    || mWin.mLastRelayoutContentInsets.height() > 0)
                        && !task.mStack.lastAnimatingBoundsWasToFullscreen()) {
                mTmpSourceBounds.set(task.mStack.mPreAnimationBounds);
                mTmpSourceBounds.inset(mWin.mLastRelayoutContentInsets);
                allowStretching = true;
            }

            ......
            mTmpStackBounds.intersectUnchecked(w.mParentFrame);
            mTmpSourceBounds.intersectUnchecked(w.mParentFrame);
            mTmpAnimatingBounds.intersectUnchecked(w.mParentFrame);

            if (!mTmpSourceBounds.isEmpty()) {
              task.mStack.getFinalAnimationBounds(mTmpAnimatingBounds);
                float finalWidth = mTmpAnimatingBounds.width();
                float initialWidth = mTmpSourceBounds.width();
                float tw = (surfaceContentWidth - mTmpStackBounds.width())
                        / (surfaceContentWidth - mTmpAnimatingBounds.width());
                float th = tw;
                mExtraHScale = (initialWidth + tw * (finalWidth - initialWidth)) / initialWidth;
                if (allowStretching) {
                    float finalHeight = mTmpAnimatingBounds.height();
                    float initialHeight = mTmpSourceBounds.height();
                    th = (surfaceContentHeight - mTmpStackBounds.height())
                        / (surfaceContentHeight - mTmpAnimatingBounds.height());
                    mExtraVScale = (initialHeight + tw * (finalHeight - initialHeight))
                            / initialHeight;
                } else {
                    mExtraVScale = mExtraHScale;
                }

                .......
                clipRect = mTmpClipRect;
                clipRect.set((int)((insets.left + mTmpSourceBounds.left) * tw),
                        (int)((insets.top + mTmpSourceBounds.top) * th),
                        insets.left + (int)(surfaceWidth
                                - (tw* (surfaceWidth - mTmpSourceBounds.right))),
                        insets.top + (int)(surfaceHeight
                                - (th * (surfaceHeight - mTmpSourceBounds.bottom))));
            } else {
                mExtraHScale = mTmpStackBounds.width() / surfaceContentWidth;
                mExtraVScale = mTmpStackBounds.height() / surfaceContentHeight;
                clipRect = null;
            }
            
            posX -= (int) (attrs.x * (1 - mExtraHScale));
            posY -= (int) (attrs.y * (1 - mExtraVScale));
            posX += insets.left * (1 - mExtraHScale);
            posY += insets.top * (1 - mExtraVScale);

            mSurfaceController.setPositionInTransaction((float) Math.floor(posX),
                    (float) Math.floor(posY), recoveringMemory);

            if (mPipAnimationStarted == false) {
                mForceScaleUntilResize = true;
                mPipAnimationStarted = true;
            }
        } else {
            mPipAnimationStarted = false;

            if (!w.mSeamlesslyRotated) {
                // Used to offset the WSA when stack position changes before a resize.
                int xOffset = mXOffset;
                int yOffset = mYOffset;
                if (mOffsetPositionForStackResize) {
                    if (relayout) {
                      setOffsetPositionForStackResize(false);
                        mSurfaceController.deferTransactionUntil(mSurfaceController.getHandle(),
                                mWin.getFrameNumber());
                    } else {
                        final TaskStack stack = mWin.getStack();
                        mTmpPos.x = 0;
                        mTmpPos.y = 0;
                        if (stack != null) {
                            stack.getRelativePosition(mTmpPos);
                        }

                        xOffset = -mTmpPos.x;
                        yOffset = -mTmpPos.y;

                        // Crop also needs to be extended so the bottom isn't cut off when the WSA
                        // position is moved.
                        if (clipRect != null) {
                            clipRect.right += mTmpPos.x;
                            clipRect.bottom += mTmpPos.y;
                        }
                    }
                }
                mSurfaceController.setPositionInTransaction(xOffset, yOffset, recoveringMemory);
            }
        }

setGeometryAppliesWithResizeInTransaction accomplishes this for us.
        if ((wasForceScaled && !mForceScaleUntilResize) ||
                (wasSeamlesslyRotated && !w.mSeamlesslyRotated)) {
            mSurfaceController.setGeometryAppliesWithResizeInTransaction(true);
            mSurfaceController.forceScaleableInTransaction(false);
        }

        if (!w.mSeamlesslyRotated) {
            applyCrop(clipRect, recoveringMemory);
---------------------------------------------------------
看看Log：
04-16 13:08:37.124 I/WindowSurfaceController( 1049):   SURFACE MATRIX [1.0,0.0,0.0,1.0]: com.android.testred/com.android.testred.TestActivity
---------------------------------------------------------mSurfaceController.setMatrixInTransaction(mDsDx * w.mHScale * mExtraHScale,
                    mDtDx * w.mVScale * mExtraVScale,
                    mDtDy * w.mHScale * mExtraHScale,
                    mDsDy * w.mVScale * mExtraVScale, recoveringMemory);
        }

        if (mSurfaceResized) {
            mReportSurfaceResized = true;
            mAnimator.setPendingLayoutChanges(w.getDisplayId(),
                    FINISH_LAYOUT_REDO_WALLPAPER);
        }
    }

```
###### 5.8.2.1、WindowStateAnimator.calculateCrop()
``` java
    private boolean calculateCrop(Rect clipRect) {
        final WindowState w = mWin;
        final DisplayContent displayContent = w.getDisplayContent();
        clipRect.setEmpty();

        if (displayContent == null) {
            return false;
        }

        if (w.inPinnedWindowingMode()) {
            return false;
        }

        // If we're animating, the wallpaper should only
        // be updated at the end of the animation.
        if (w.mAttrs.type == TYPE_WALLPAPER) {
            return false;
        }
---------------------------------------------------------
看看Log：
04-16 13:08:37.124 D/WindowStateAnimator( 1049): Updating crop win=Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity} mLastCrop=Rect(0, 36 - 160, 782)
04-16 13:08:37.124 D/WindowStateAnimator( 1049): Applying decor to crop win=Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity} mDecorFrame=Rect(0, 36 - 480, 782) mSystemDecorRect=Rect(0, 36 - 160, 782)
04-16 13:08:37.124 D/WindowStateAnimator( 1049): win=Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity} Initial clip rect: Rect(0, 36 - 160, 782) fullscreen=true
04-16 13:08:37.124 D/WindowStateAnimator( 1049): win=Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity} Clip rect after stack adjustment=Rect(0, 36 - 160, 782)
04-16 13:08:37.124 D/WindowStateAnimator( 1049): 
---------------------------------------------------------
        if (DEBUG_WINDOW_CROP) Slog.d(TAG,
                "Updating crop win=" + w + " mLastCrop=" + mLastClipRect);

        w.calculatePolicyCrop(mSystemDecorRect);

        if (DEBUG_WINDOW_CROP) Slog.d(TAG, "Applying decor to crop win=" + w + " mDecorFrame="
                + w.mDecorFrame + " mSystemDecorRect=" + mSystemDecorRect);

        final Task task = w.getTask();
        final boolean fullscreen = w.fillsDisplay() || (task != null && task.isFullscreen());
        final boolean isFreeformResizing =
                w.isDragResizing() && w.getResizeMode() == DRAG_RESIZE_MODE_FREEFORM;

        // We use the clip rect as provided by the tranformation for non-fullscreen windows to
        // avoid premature clipping with the system decor rect.
        clipRect.set(mSystemDecorRect);
        if (DEBUG_WINDOW_CROP) Slog.d(TAG, "win=" + w + " Initial clip rect: " + clipRect
                + " fullscreen=" + fullscreen);

        w.expandForSurfaceInsets(clipRect);

        // The clip rect was generated assuming (0,0) as the window origin,
        // so we need to translate to match the actual surface coordinates.
        clipRect.offset(w.mAttrs.surfaceInsets.left, w.mAttrs.surfaceInsets.top);

        if (DEBUG_WINDOW_CROP) Slog.d(TAG,
                "win=" + w + " Clip rect after stack adjustment=" + clipRect);

        w.transformClipRectFromScreenToSurfaceSpace(clipRect);

        return true;
    }
```

###### 5.8.2.2、WindowStateAnimator.applyCrop()

``` java

---------------------------------------------------------
看看Log：
04-16 13:08:37.124 D/WindowStateAnimator( 1049): applyCrop: win=Window{c1255cd u0 com.android.testred/com.android.testred.TestActivity} clipRect=Rect(0, 36 - 160, 782)
04-16 13:08:37.124 I/WindowSurfaceController( 1049):   SURFACE MATRIX [1.0,0.0,0.0,1.0]: com.android.testred/com.android.testred.TestActivity
---------------------------------------------------------
    private void applyCrop(Rect clipRect, boolean recoveringMemory) {
        if (DEBUG_WINDOW_CROP) Slog.d(TAG, "applyCrop: win=" + mWin
                + " clipRect=" + clipRect);
        if (clipRect != null) {
            if (!clipRect.equals(mLastClipRect)) {
                mLastClipRect.set(clipRect);
                mSurfaceController.setCropInTransaction(clipRect, recoveringMemory);
            }
        } else {
            mSurfaceController.clearCropInTransaction(recoveringMemory);
        }
    }
```


##### 5.8.3、mSurfaceController.setPositionInTransaction(......)

``` java
/frameworks/base/services/core/java/com/android/server/wm/WindowSurfaceController.java

    void setPositionInTransaction(float left, float top, boolean recoveringMemory) {
        setPosition(null, left, top, recoveringMemory);
    }

    void setPosition(SurfaceControl.Transaction t, float left, float top,
            boolean recoveringMemory) {
        final boolean surfaceMoved = mSurfaceX != left || mSurfaceY != top;
        if (surfaceMoved) {
            mSurfaceX = left;
            mSurfaceY = top;

            try {
                if (SHOW_TRANSACTIONS) logSurface(
                        "POS (setPositionInTransaction) @ (" + left + "," + top + ")", null);

                if (t == null) {
                    mSurfaceControl.setPosition(left, top);
                } else {
                    t.setPosition(mSurfaceControl, left, top);
                }
            } catch (RuntimeException e) {
                Slog.w(TAG, "Error positioning surface of " + this
                        + " pos=(" + left + "," + top + ")", e);
                if (!recoveringMemory) {
                    mAnimator.reclaimSomeSurfaceMemory("position", true);
                }
            }
        }
    }
```

##### 5.9、AppWindowToken.updateAllDrawn()

``` java
/frameworks/base/services/core/java/com/android/server/wm/AppWindowToken.java
---------------------------------------------------------
看看Log：
04-16 13:08:37.125 V/AppWindowToken( 1049): allDrawn: AppWindowToken{1579134 token=Token{ce2ac07 ActivityRecord{65cc87a u0 com.android.testred/.TestActivity t5}}} interesting=1 drawn=1
---------------------------------------------------------
    void updateAllDrawn() {
        if (!allDrawn) {
            final int numInteresting = mNumInterestingWindows;
            
            if (numInteresting > 0 && allDrawnStatesConsidered()
                    && mNumDrawnWindows >= numInteresting && !isRelaunching()) {
                if (DEBUG_VISIBILITY) Slog.v(TAG, "allDrawn: " + this
                        + " interesting=" + numInteresting + " drawn=" + mNumDrawnWindows);
                allDrawn = true;

                if (mDisplayContent != null) {
                    mDisplayContent.setLayoutNeeded();
                }
                mService.mH.obtainMessage(NOTIFY_ACTIVITY_DRAWN, token).sendToTarget();
                
                final TaskStack pinnedStack = mDisplayContent.getPinnedStack();
                if (pinnedStack != null) {
                    pinnedStack.onAllWindowsDrawn();
                }
            }
        }
    }

```
##### （6）、执行窗口合成显示

##### 6.1、WindowStateAnimator.showSurfaceRobustlyLocked()

``` java
/frameworks/base/services/core/java/com/android/server/wm/WindowStateAnimator.java

    private boolean showSurfaceRobustlyLocked() {
        if (mWin.getWindowConfiguration().windowsAreScaleable()) {
            mSurfaceController.forceScaleableInTransaction(true);
        }

        boolean shown = mSurfaceController.showRobustlyInTransaction();
        if (!shown)
            return false;
        if (mPendingDestroySurface != null && mDestroyPreservedSurfaceUponRedraw) {
            mPendingDestroySurface.mSurfaceControl.hide();
            mPendingDestroySurface.reparentChildrenInTransaction(mSurfaceController);
        }

        return true;
    }
```

###### 6.1.1、WindowSurfaceController.showRobustlyInTransaction()

``` java
WindowSurfaceController.java

---------------------------------------------------------
看看Log：
04-16 13:08:37.153 I/WindowSurfaceController( 1049):   SURFACE SHOW (performLayout): com.android.testred/com.android.testred.TestActivity
04-16 13:08:37.153 V/WindowSurfaceController( 1049): Showing Surface(name=com.android.testred/com.android.testred.TestActivity)/@0xb17b0ce during relayout
---------------------------------------------------------
    boolean showRobustlyInTransaction() {
        if (SHOW_TRANSACTIONS) logSurface(
                "SHOW (performLayout)", null);
        if (DEBUG_VISIBILITY) Slog.v(TAG, "Showing " + this
                + " during relayout");
        mHiddenForOtherReasons = false;
        return updateVisibility();
    }

  private boolean updateVisibility() {
        if (mHiddenForCrop || mHiddenForOtherReasons) {
            if (mSurfaceShown) {
                hideSurface(mTmpTransaction);
                SurfaceControl.mergeToGlobalTransaction(mTmpTransaction);
            }
            return false;
        } else {
            if (!mSurfaceShown) {
                return showSurface();
            } else {
                return true;
            }
        }
    }

    private boolean showSurface() {
        try {
            setShown(true);
            mSurfaceControl.show();
            return true;
        } catch (RuntimeException e) {
            Slog.w(TAG, "Failure showing surface " + mSurfaceControl + " in " + this, e);
        }

        mAnimator.reclaimSomeSurfaceMemory("show", true);

        return false;
    }

```

###### 6.1.2、SurfaceControl.show()

``` java
/frameworks/base/core/java/android/view/SurfaceControl.java

    public void show() {
        checkNotReleased();
        synchronized(SurfaceControl.class) {
            sGlobalTransaction.show(this);
        }
    }
        public Transaction show(SurfaceControl sc) {
            sc.checkNotReleased();
            nativeSetFlags(mNativeObject, sc.mNativeObject, 0, SURFACE_HIDDEN);
            return this;
        }

```


##### 6.2、SurfaceFlinger 合成显示

``` java
---------------------------------------------------------
看看Log：
04-16 13:08:37.161 V/SurfaceFlinger(  739): handlePageFlip
04-16 13:08:37.161 V/SurfaceFlinger(  739): preComposition
04-16 13:08:37.161 V/SurfaceFlinger(  739): rebuildLayerStacks
04-16 13:08:37.161 V/SurfaceFlinger(  739): computeVisibleRegions
04-16 13:08:37.161 V/SurfaceFlinger(  739): setUpHWComposer
04-16 13:08:37.162 V/HWC2    (  739): Created layer 8 on display 0
04-16 13:08:37.162 V/HWC2    (  739): Created layer 9 on display 0
.....
04-16 13:08:37.163 V/BufferLayer(  739): [SurfaceView - com.android.testred/com.android.testred.TestActivity#0] Requesting Device composition
04-16 13:08:37.163 V/Layer   (  739): setCompositionType(8, Device, 1)
04-16 13:08:37.163 V/Layer   (  739):     actually setting
04-16 13:08:37.163 V/BufferLayer(  739): setPerFrameData: dataspace = 0
04-16 13:08:37.163 V/BufferLayerConsumer(  739): [SurfaceView - com.android.testred/com.android.testred.TestActivity#0] getCurrentHdrMetadata
04-16 13:08:37.163 V/BufferLayer(  739): [com.android.testred/com.android.testred.TestActivity#0] Requesting Device composition
04-16 13:08:37.163 V/Layer   (  739): setCompositionType(9, Device, 1)
04-16 13:08:37.163 V/Layer   (  739):     actually setting
04-16 13:08:37.163 V/BufferLayer(  739): setPerFrameData: dataspace = 0
04-16 13:08:37.163 V/BufferLayerConsumer(  739): [com.android.testred/com.android.testred.TestActivity#0] getCurrentHdrMetadata
04-16 13:08:37.163 V/BufferLayer(  739): [StatusBar#0] Requesting Device composition
04-16 13:08:37.163 V/Layer   (  739): setCompositionType(3, Device, 1)
04-16 13:08:37.163 V/BufferLayer(  739): setPerFrameData: dataspace = 0
04-16 13:08:37.163 V/BufferLayerConsumer(  739): [StatusBar#0] getCurrentHdrMetadata
04-16 13:08:37.163 V/BufferLayer(  739): [NavigationBar#0] Requesting Device composition
04-16 13:08:37.163 V/Layer   (  739): setCompositionType(4, Device, 1)
04-16 13:08:37.163 V/BufferLayer(  739): setPerFrameData: dataspace = 0
04-16 13:08:37.163 V/BufferLayerConsumer(  739): [NavigationBar#0] getCurrentHdrMetadata
......
04-16 13:08:37.164 V/HWComposer(  739): SkipValidate failed, Falling back to SLOW validate/present
04-16 13:08:37.164 V/SurfaceFlinger(  739): doComposition
04-16 13:08:37.164 V/SurfaceFlinger(  739): doDisplayComposition
04-16 13:08:37.164 V/SurfaceFlinger(  739): doComposeSurfaces
04-16 13:08:37.164 V/SurfaceFlinger(  739): Rendering client layers
04-16 13:08:37.164 V/SurfaceFlinger(  739): Layer: com.android.systemui.ImageWallpaper#0
04-16 13:08:37.164 V/SurfaceFlinger(  739):   Composition type: Device
04-16 13:08:37.164 V/SurfaceFlinger(  739): Layer: com.android.launcher3/com.android.launcher3.Launcher#0
04-16 13:08:37.164 V/SurfaceFlinger(  739):   Composition type: Device
04-16 13:08:37.164 V/SurfaceFlinger(  739): Layer: SurfaceView - com.android.testred/com.android.testred.TestActivity#0
04-16 13:08:37.164 V/SurfaceFlinger(  739):   Composition type: Device
04-16 13:08:37.164 V/SurfaceFlinger(  739): Layer: com.android.testred/com.android.testred.TestActivity#0
04-16 13:08:37.164 V/SurfaceFlinger(  739):   Composition type: Device
04-16 13:08:37.164 V/SurfaceFlinger(  739): Layer: StatusBar#0
04-16 13:08:37.164 V/SurfaceFlinger(  739):   Composition type: Device
04-16 13:08:37.164 V/SurfaceFlinger(  739): Layer: NavigationBar#0
04-16 13:08:37.164 V/SurfaceFlinger(  739):   Composition type: Device
04-16 13:08:37.164 V/SurfaceFlinger(  739): postFramebuffer
---------------------------------------------------------
```

关于SurfaceFlinger合成显示之前已经分析过很多次啦，这里就不再分析啦。请参考：{ Android P Graphics System（四）：Native Surface 创建 && SurfaceFlinger合成流程分析 }
![enter image description here](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/personal.website/zjj.display.sys.com.android.test_red.png) 


---
/**************Vsync end**************/
---



####  （X）、参考文档：

[（1）【Android Display System】](http://charlesvincent.cc/)



















