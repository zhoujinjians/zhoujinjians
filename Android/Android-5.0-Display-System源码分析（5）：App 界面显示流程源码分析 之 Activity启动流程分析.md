---
title: Android L Display System源码分析（5）：App 界面显示流程源码分析 之 Activity启动流程分析（Android 9.0 && Kernel 3.18）
cover: https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/personal.website/post.cover.pictures.00005.jpg
categories: 
  - Display
tags:
  - Android
  - Linux
  - Graphics
toc: true
abbrlink: 20191201
date: 2019-12-01 09:25:00
---


--------------------------------------------------------------------------------

注：文章都是通过阅读各位前辈总结的资料 Android 9.0 && Linux（Kernel 3.18）Qualcomm平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除（◕‿◕），转载请注明出处（©Qualcomm ©Android @Linux 版权所有），谢谢。
（==**文章基于 Kernel-3.18**==）&&（==**文章基于 Android 9.0**==）
[【开发板 Intrinsyc Open-Q™ 820 µSOM Development Kit】](https://www.intrinsyc.com/snapdragon-embedded-development-kits/open-q-820-usom-development-kit/)
[【开发板 Android 9.0 && Linux（Kernel 3.18）源码链接】](https://gitlab.com/zhoujinjiana/apq8096_la.um.7.5.r1-03100-8x96.0_p_v5.0)

正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢！！！

[（1）【Android Display System】](http://charlesvincent.cc/)
[（2）【Android 9.0 activity启动源码分析】](https://www.jianshu.com/p/50e4b48ed15c)
[（3）【Android 9.0 Activity启动流程源码分析】](https://blog.csdn.net/lj19851227/article/details/82562115)
[（4）【Android App("com.android.testgreen")源码】](https://gitlab.com/zhoujinjiana/apq8096_la.um.7.5.r1-03100-8x96.0_p_v5.0/commit/714c0071b4bc786d00e7a6bcd08face6a28d6c0c)

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

----------

####  App（"com.android.testred"）分析准备工作
分析学习Android源码，最好有编译源码环境并有开发板。
##### ①、App运行效果图
![enter image description here](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/personal.website/zjj.display.sys.test_red.png) 

![enter image description here](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/personal.website/zjj.display.sys.test_green.png)

![enter image description here](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/personal.website/zjj.display.sys.test_blue.png)

以上时三个App运行效果图，我们这里分析App（"com.android.testred"），流程学习最好的办法就是打开Debug开关跟着log分析。
##### ②、打开Debug开关，抓取Log

``` java
diff --git a/frameworks/base/core/java/android/view/View.java b/frameworks/base/core/java/android/view/View.java
index 991ba74..14adf45 100644
--- a/frameworks/base/core/java/android/view/View.java
+++ b/frameworks/base/core/java/android/view/View.java
 -777,10 +777,10 @@ import java.util.function.Predicate;
 @UiThread
 public class View implements Drawable.Callback, KeyEvent.Callback,
         AccessibilityEventSource {
-    private static final boolean DBG = false;
+    private static final boolean DBG = true;
 
     /** @hide */
-    public static boolean DEBUG_DRAW = false;
+    public static boolean DEBUG_DRAW = true;
 
     /**
      * The logging tag used by this class with android.util.Log.
diff --git a/frameworks/base/core/java/android/view/ViewGroup.java b/frameworks/base/core/java/android/view/ViewGroup.java
index baa38bb..ed31ff8 100644
--- a/frameworks/base/core/java/android/view/ViewGroup.java
+++ b/frameworks/base/core/java/android/view/ViewGroup.java
@ -120,7 +120,7 @@ import java.util.function.Predicate;
 public abstract class ViewGroup extends View implements ViewParent, ViewManager {
     private static final String TAG = "ViewGroup";
 
-    private static final boolean DBG = false;
+    private static final boolean DBG = true;
 
     /**
      * Views which have been hidden or removed which need to be animated on
diff --git a/frameworks/base/core/java/android/view/ViewRootImpl.java b/frameworks/base/core/java/android/view/ViewRootImpl.java
index e7bda30..f3b4b31 100644
--- a/frameworks/base/core/java/android/view/ViewRootImpl.java
+++ b/frameworks/base/core/java/android/view/ViewRootImpl.java
@@ -133,19 +133,19 @@ import java.util.concurrent.CountDownLatch;
 public final class ViewRootImpl implements ViewParent,
         View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
     private static final String TAG = "ViewRootImpl";
-    private static final boolean DBG = false;
-    private static final boolean LOCAL_LOGV = false;
+    private static final boolean DBG = true;
+    private static final boolean LOCAL_LOGV = true;
     /** @noinspection PointlessBooleanExpression*/
-    private static final boolean DEBUG_DRAW = false || LOCAL_LOGV;
-    private static final boolean DEBUG_LAYOUT = false || LOCAL_LOGV;
-    private static final boolean DEBUG_DIALOG = false || LOCAL_LOGV;
-    private static final boolean DEBUG_INPUT_RESIZE = false || LOCAL_LOGV;
-    private static final boolean DEBUG_ORIENTATION = false || LOCAL_LOGV;
-    private static final boolean DEBUG_TRACKBALL = false || LOCAL_LOGV;
-    private static final boolean DEBUG_IMF = false || LOCAL_LOGV;
-    private static final boolean DEBUG_CONFIGURATION = false || LOCAL_LOGV;
-    private static final boolean DEBUG_FPS = false;
-    private static final boolean DEBUG_INPUT_STAGES = false || LOCAL_LOGV;
+    private static final boolean DEBUG_DRAW = true || LOCAL_LOGV;
+    private static final boolean DEBUG_LAYOUT = true || LOCAL_LOGV;
+    private static final boolean DEBUG_DIALOG = true || LOCAL_LOGV;
+    private static final boolean DEBUG_INPUT_RESIZE = true || LOCAL_LOGV;
+    private static final boolean DEBUG_ORIENTATION = true || LOCAL_LOGV;
+    private static final boolean DEBUG_TRACKBALL = true || LOCAL_LOGV;
+    private static final boolean DEBUG_IMF = true || LOCAL_LOGV;
+    private static final boolean DEBUG_CONFIGURATION = true || LOCAL_LOGV;
+    private static final boolean DEBUG_FPS = true;
+    private static final boolean DEBUG_INPUT_STAGES = true || LOCAL_LOGV;
     private static final boolean DEBUG_KEEP_SCREEN_ON = false || LOCAL_LOGV;
 
     /**
diff --git a/frameworks/base/services/core/java/com/android/server/am/ActivityManagerDebugConfig.java b/frameworks/base/services/core/java/com/android/server/am/ActivityManagerDebugConfig.java
index 0a7d3fd..6846966 100644
--- a/frameworks/base/services/core/java/com/android/server/am/ActivityManagerDebugConfig.java
+++ b/frameworks/base/services/core/java/com/android/server/am/ActivityManagerDebugConfig.java
@@ -28,18 +28,18 @@ class ActivityManagerDebugConfig {
     // to figure-out the origin of a log message while debugging the activity manager a little
     // painful. By setting this constant to true, log messages from the activity manager package
     // will be tagged with their class names instead fot the generic tag.
-    static final boolean TAG_WITH_CLASS_NAME = false;
+    static final boolean TAG_WITH_CLASS_NAME = true;
 
     // While debugging it is sometimes useful to have the category name of the log appended to the
     // base log tag to make sifting through logs with the same base tag easier. By setting this
     // constant to true, the category name of the log point will be appended to the log tag.
-    static final boolean APPEND_CATEGORY_NAME = false;
+    static final boolean APPEND_CATEGORY_NAME = true;
 
     // Default log tag for the activity manager package.
     static final String TAG_AM = "ActivityManager";
 
     // Enable all debug log categories.
-    static final boolean DEBUG_ALL = false;
+    static final boolean DEBUG_ALL = true;
 
     // Enable all debug log categories for activities.
     static final boolean DEBUG_ALL_ACTIVITIES = DEBUG_ALL || false;
@@ -56,7 +56,7 @@ class ActivityManagerDebugConfig {
     static final boolean DEBUG_CLEANUP = DEBUG_ALL || false;
     static final boolean DEBUG_CONFIGURATION = DEBUG_ALL || false;
     static final boolean DEBUG_CONTAINERS = DEBUG_ALL_ACTIVITIES || false;
-    static final boolean DEBUG_FOCUS = false;
+    static final boolean DEBUG_FOCUS = true;
     static final boolean DEBUG_IDLE = DEBUG_ALL_ACTIVITIES || false;
     static final boolean DEBUG_IMMERSIVE = DEBUG_ALL || false;
     static final boolean DEBUG_LOCKTASK = DEBUG_ALL || false;
diff --git a/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java b/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
index 04a25ea..18a9226 100644
--- a/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
+++ b/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
@@ -2733,10 +2733,10 @@ public class ActivityManagerService extends IActivityManager.Stub
                             pid = proc.pid;
                         } else {
                             ProcessList.abortNextPssTime(proc.procStateMemTracker);
-                            if (DEBUG_PSS) Slog.d(TAG_PSS, "Skipped pss collection of " + pid +
-                                    ": still need " +
-                                    (lastPssTime+ProcessList.PSS_SAFE_TIME_FROM_STATE_CHANGE-now) +
-                                    "ms until safe");
+                            //if (DEBUG_PSS) Slog.d(TAG_PSS, "Skipped pss collection of " + pid +
+                            //        ": still need " +
+                            //        (lastPssTime+ProcessList.PSS_SAFE_TIME_FROM_STATE_CHANGE-now) +
+                            //        "ms until safe");
                             proc = null;
                             pid = 0;
                         }
diff --git a/frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java b/frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java
index 7f6d6c9..153eb94 100644
--- a/frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java
+++ b/frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java
@@ -306,11 +306,11 @@ import java.util.List;
  */
 public class PhoneWindowManager implements WindowManagerPolicy {
     static final String TAG = "WindowManager";
-    static final boolean DEBUG = false;
-    static final boolean localLOGV = false;
+    static final boolean DEBUG = true;
+    static final boolean localLOGV = true;
     static final boolean DEBUG_INPUT = false;
     static final boolean DEBUG_KEYGUARD = false;
-    static final boolean DEBUG_LAYOUT = false;
+    static final boolean DEBUG_LAYOUT = true;
     static final boolean DEBUG_SPLASH_SCREEN = false;
     static final boolean DEBUG_WAKEUP = false;
     static final boolean SHOW_SPLASH_SCREENS = true;
diff --git a/frameworks/base/services/core/java/com/android/server/wm/WindowManagerDebugConfig.java b/frameworks/base/services/core/java/com/android/server/wm/WindowManagerDebugConfig.java
index c366e4d..8bfb9f1 100644
--- a/frameworks/base/services/core/java/com/android/server/wm/WindowManagerDebugConfig.java
+++ b/frameworks/base/services/core/java/com/android/server/wm/WindowManagerDebugConfig.java
@@ -28,53 +28,53 @@ public class WindowManagerDebugConfig {
     // to figure-out the origin of a log message while debugging the window manager a little
     // painful. By setting this constant to true, log messages from the window manager package
     // will be tagged with their class names instead fot the generic tag.
-    static final boolean TAG_WITH_CLASS_NAME = false;
+    static final boolean TAG_WITH_CLASS_NAME = true;
 
     // Default log tag for the window manager package.
     static final String TAG_WM = "WindowManager";
 
-    static final boolean DEBUG_RESIZE = false;
-    static final boolean DEBUG = false;
-    static final boolean DEBUG_ADD_REMOVE = false;
-    static final boolean DEBUG_FOCUS = false;
+    static final boolean DEBUG_RESIZE = true;
+    static final boolean DEBUG = true;
+    static final boolean DEBUG_ADD_REMOVE = true;
+    static final boolean DEBUG_FOCUS = true;
     static final boolean DEBUG_FOCUS_LIGHT = DEBUG_FOCUS || false;
-    static final boolean DEBUG_ANIM = false;
-    static final boolean DEBUG_KEYGUARD = false;
-    static final boolean DEBUG_LAYOUT = false;
-    static final boolean DEBUG_LAYERS = false;
-    static final boolean DEBUG_INPUT = false;
-    static final boolean DEBUG_INPUT_METHOD = false;
-    static final boolean DEBUG_VISIBILITY = false;
-    static final boolean DEBUG_WINDOW_MOVEMENT = false;
-    static final boolean DEBUG_TOKEN_MOVEMENT = false;
-    static final boolean DEBUG_ORIENTATION = false;
-    static final boolean DEBUG_APP_ORIENTATION = false;
-    static final boolean DEBUG_CONFIGURATION = false;
-    static final boolean DEBUG_APP_TRANSITIONS = false;
-    static final boolean DEBUG_STARTING_WINDOW_VERBOSE = false;
+    static final boolean DEBUG_ANIM = true;
+    static final boolean DEBUG_KEYGUARD = true;
+    static final boolean DEBUG_LAYOUT = true;
+    static final boolean DEBUG_LAYERS = true;
+    static final boolean DEBUG_INPUT = true;
+    static final boolean DEBUG_INPUT_METHOD = true;
+    static final boolean DEBUG_VISIBILITY = true;
+    static final boolean DEBUG_WINDOW_MOVEMENT = true;
+    static final boolean DEBUG_TOKEN_MOVEMENT = true;
+    static final boolean DEBUG_ORIENTATION = true;
+    static final boolean DEBUG_APP_ORIENTATION = true;
+    static final boolean DEBUG_CONFIGURATION = true;
+    static final boolean DEBUG_APP_TRANSITIONS = true;
+    static final boolean DEBUG_STARTING_WINDOW_VERBOSE = true;
     static final boolean DEBUG_STARTING_WINDOW = DEBUG_STARTING_WINDOW_VERBOSE || false;
-    static final boolean DEBUG_WALLPAPER = false;
-    static final boolean DEBUG_WALLPAPER_LIGHT = false || DEBUG_WALLPAPER;
-    static final boolean DEBUG_DRAG = false;
-    static final boolean DEBUG_SCREEN_ON = false;
-    static final boolean DEBUG_SCREENSHOT = false;
-    static final boolean DEBUG_BOOT = false;
-    static final boolean DEBUG_LAYOUT_REPEATS = false;
-    static final boolean DEBUG_WINDOW_TRACE = false;
-    static final boolean DEBUG_TASK_MOVEMENT = false;
-    static final boolean DEBUG_TASK_POSITIONING = false;
-    static final boolean DEBUG_STACK = false;
-    static final boolean DEBUG_DISPLAY = false;
-    static final boolean DEBUG_POWER = false;
-    static final boolean DEBUG_DIM_LAYER = false;
-    static final boolean SHOW_SURFACE_ALLOC = false;
-    static final boolean SHOW_TRANSACTIONS = false;
-    static final boolean SHOW_VERBOSE_TRANSACTIONS = false && SHOW_TRANSACTIONS;
-    static final boolean SHOW_LIGHT_TRANSACTIONS = false || SHOW_TRANSACTIONS;
-    static final boolean SHOW_STACK_CRAWLS = false;
-    static final boolean DEBUG_WINDOW_CROP = false;
-    static final boolean DEBUG_UNKNOWN_APP_VISIBILITY = false;
-    static final boolean DEBUG_RECENTS_ANIMATIONS = false;
+    static final boolean DEBUG_WALLPAPER = true;
+    static final boolean DEBUG_WALLPAPER_LIGHT = true || DEBUG_WALLPAPER;
+    static final boolean DEBUG_DRAG = true;
+    static final boolean DEBUG_SCREEN_ON = true;
+    static final boolean DEBUG_SCREENSHOT = true;
+    static final boolean DEBUG_BOOT = true;
+    static final boolean DEBUG_LAYOUT_REPEATS = true;
+    static final boolean DEBUG_WINDOW_TRACE = true;
+    static final boolean DEBUG_TASK_MOVEMENT = true;
+    static final boolean DEBUG_TASK_POSITIONING = true;
+    static final boolean DEBUG_STACK = true;
+    static final boolean DEBUG_DISPLAY = true;
+    static final boolean DEBUG_POWER = true;
+    static final boolean DEBUG_DIM_LAYER = true;
+    static final boolean SHOW_SURFACE_ALLOC = true;
+    static final boolean SHOW_TRANSACTIONS = true;
+    static final boolean SHOW_VERBOSE_TRANSACTIONS = true && SHOW_TRANSACTIONS;
+    static final boolean SHOW_LIGHT_TRANSACTIONS = true || SHOW_TRANSACTIONS;
+    static final boolean SHOW_STACK_CRAWLS = true;
+    static final boolean DEBUG_WINDOW_CROP = true;
+    static final boolean DEBUG_UNKNOWN_APP_VISIBILITY = true;
+    static final boolean DEBUG_RECENTS_ANIMATIONS = true;
     static final boolean DEBUG_REMOTE_ANIMATIONS = DEBUG_APP_TRANSITIONS || false;
 
     static final String TAG_KEEP_SCREEN_ON = "DebugKeepScreenOn";
diff --git a/frameworks/native/libs/gui/BufferItemConsumer.cpp b/frameworks/native/libs/gui/BufferItemConsumer.cpp
index 89bc0c4..7405b7f 100644
--- a/frameworks/native/libs/gui/BufferItemConsumer.cpp
+++ b/frameworks/native/libs/gui/BufferItemConsumer.cpp
@@ -14,7 +14,7 @@
  * limitations under the License.
  */
 
-//#define LOG_NDEBUG 0
+#define LOG_NDEBUG 0
 #define LOG_TAG "BufferItemConsumer"
 //#define ATRACE_TAG ATRACE_TAG_GRAPHICS
 #include <utils/Log.h>
diff --git a/frameworks/native/libs/gui/BufferQueue.cpp b/frameworks/native/libs/gui/BufferQueue.cpp
index a8da134..70695d0 100644
--- a/frameworks/native/libs/gui/BufferQueue.cpp
+++ b/frameworks/native/libs/gui/BufferQueue.cpp
@@ -16,7 +16,7 @@
 
 #define LOG_TAG "BufferQueue"
 #define ATRACE_TAG ATRACE_TAG_GRAPHICS
-//#define LOG_NDEBUG 0
+#define LOG_NDEBUG 0
 
 #ifndef NO_BUFFERHUB
 #include <gui/BufferHubConsumer.h>
diff --git a/frameworks/native/libs/gui/BufferQueueConsumer.cpp b/frameworks/native/libs/gui/BufferQueueConsumer.cpp
index d70e142..fe08603 100644
--- a/frameworks/native/libs/gui/BufferQueueConsumer.cpp
+++ b/frameworks/native/libs/gui/BufferQueueConsumer.cpp
@@ -20,7 +20,7 @@
 
 #define LOG_TAG "BufferQueueConsumer"
 #define ATRACE_TAG ATRACE_TAG_GRAPHICS
-//#define LOG_NDEBUG 0
+#define LOG_NDEBUG 0
 
 #if DEBUG_ONLY_CODE
 #define VALIDATE_CONSISTENCY() do { mCore->validateConsistencyLocked(); } while (0)
diff --git a/frameworks/native/libs/gui/BufferQueueCore.cpp b/frameworks/native/libs/gui/BufferQueueCore.cpp
index bb703da..20a4c72 100644
--- a/frameworks/native/libs/gui/BufferQueueCore.cpp
+++ b/frameworks/native/libs/gui/BufferQueueCore.cpp
@@ -16,7 +16,7 @@
 
 #define LOG_TAG "BufferQueueCore"
 #define ATRACE_TAG ATRACE_TAG_GRAPHICS
-//#define LOG_NDEBUG 0
+#define LOG_NDEBUG 0
 
 #define EGL_EGLEXT_PROTOTYPES
 
diff --git a/frameworks/native/libs/gui/BufferQueueProducer.cpp b/frameworks/native/libs/gui/BufferQueueProducer.cpp
index c8021e4..73917f2 100644
--- a/frameworks/native/libs/gui/BufferQueueProducer.cpp
+++ b/frameworks/native/libs/gui/BufferQueueProducer.cpp
@@ -18,7 +18,7 @@
 
 #define LOG_TAG "BufferQueueProducer"
 #define ATRACE_TAG ATRACE_TAG_GRAPHICS
-//#define LOG_NDEBUG 0
+#define LOG_NDEBUG 0
 
 #if DEBUG_ONLY_CODE
 #define VALIDATE_CONSISTENCY() do { mCore->validateConsistencyLocked(); } while (0)
diff --git a/frameworks/native/libs/gui/ConsumerBase.cpp b/frameworks/native/libs/gui/ConsumerBase.cpp
index f9e292e..d14c55c 100644
--- a/frameworks/native/libs/gui/ConsumerBase.cpp
+++ b/frameworks/native/libs/gui/ConsumerBase.cpp
@@ -18,7 +18,7 @@
 
 #define LOG_TAG "ConsumerBase"
 #define ATRACE_TAG ATRACE_TAG_GRAPHICS
-//#define LOG_NDEBUG 0
+#define LOG_NDEBUG 0
 
 #define EGL_EGLEXT_PROTOTYPES
 
diff --git a/frameworks/native/libs/gui/CpuConsumer.cpp b/frameworks/native/libs/gui/CpuConsumer.cpp
index 8edf604..8a3a4c7 100644
--- a/frameworks/native/libs/gui/CpuConsumer.cpp
+++ b/frameworks/native/libs/gui/CpuConsumer.cpp
@@ -14,7 +14,7 @@
  * limitations under the License.
  */
 
-//#define LOG_NDEBUG 0
+#define LOG_NDEBUG 0
 #define LOG_TAG "CpuConsumer"
 //#define ATRACE_TAG ATRACE_TAG_GRAPHICS
 
diff --git a/frameworks/native/libs/gui/GLConsumer.cpp b/frameworks/native/libs/gui/GLConsumer.cpp
index 885efec..df894ab 100644
--- a/frameworks/native/libs/gui/GLConsumer.cpp
+++ b/frameworks/native/libs/gui/GLConsumer.cpp
@@ -16,7 +16,7 @@
 
 #define LOG_TAG "GLConsumer"
 #define ATRACE_TAG ATRACE_TAG_GRAPHICS
-//#define LOG_NDEBUG 0
+#define LOG_NDEBUG 0
 
 #define GL_GLEXT_PROTOTYPES
 #define EGL_EGLEXT_PROTOTYPES
diff --git a/frameworks/native/libs/gui/Surface.cpp b/frameworks/native/libs/gui/Surface.cpp
index 339bd0f..40c18d3 100644
--- a/frameworks/native/libs/gui/Surface.cpp
+++ b/frameworks/native/libs/gui/Surface.cpp
@@ -16,7 +16,7 @@
 
 #define LOG_TAG "Surface"
 #define ATRACE_TAG ATRACE_TAG_GRAPHICS
-//#define LOG_NDEBUG 0
+#define LOG_NDEBUG 0
 
 #include <gui/Surface.h>
 
diff --git a/frameworks/native/libs/ui/Fence.cpp b/frameworks/native/libs/ui/Fence.cpp
index d6ee80d..9912ab2 100644
--- a/frameworks/native/libs/ui/Fence.cpp
+++ b/frameworks/native/libs/ui/Fence.cpp
@@ -18,7 +18,7 @@
 
 #define LOG_TAG "Fence"
 #define ATRACE_TAG ATRACE_TAG_GRAPHICS
-//#define LOG_NDEBUG 0
+#define LOG_NDEBUG 0
 
 // We would eliminate the non-conforming zero-length array, but we can't since
 // this is effectively included from the Linux kernel
diff --git a/frameworks/native/libs/ui/GraphicBufferMapper.cpp b/frameworks/native/libs/ui/GraphicBufferMapper.cpp
index 2d8e582..c1e1043 100644
--- a/frameworks/native/libs/ui/GraphicBufferMapper.cpp
+++ b/frameworks/native/libs/ui/GraphicBufferMapper.cpp
@@ -16,7 +16,7 @@
 
 #define LOG_TAG "GraphicBufferMapper"
 #define ATRACE_TAG ATRACE_TAG_GRAPHICS
-//#define LOG_NDEBUG 0
+#define LOG_NDEBUG 0
 
 #include <ui/GraphicBufferMapper.h>
 
diff --git a/frameworks/native/services/surfaceflinger/BufferLayer.cpp b/frameworks/native/services/surfaceflinger/BufferLayer.cpp
index 4d3e6cc..56c608e 100644
--- a/frameworks/native/services/surfaceflinger/BufferLayer.cpp
+++ b/frameworks/native/services/surfaceflinger/BufferLayer.cpp
@@ -14,7 +14,7 @@
  * limitations under the License.
  */
 
-//#define LOG_NDEBUG 0
+#define LOG_NDEBUG 0
 #undef LOG_TAG
 #define LOG_TAG "BufferLayer"
 #define ATRACE_TAG ATRACE_TAG_GRAPHICS
diff --git a/frameworks/native/services/surfaceflinger/BufferLayerConsumer.cpp b/frameworks/native/services/surfaceflinger/BufferLayerConsumer.cpp
index 87333d0..4ebf24e 100644
--- a/frameworks/native/services/surfaceflinger/BufferLayerConsumer.cpp
+++ b/frameworks/native/services/surfaceflinger/BufferLayerConsumer.cpp
@@ -17,7 +17,7 @@
 #undef LOG_TAG
 #define LOG_TAG "BufferLayerConsumer"
 #define ATRACE_TAG ATRACE_TAG_GRAPHICS
-//#define LOG_NDEBUG 0
+#define LOG_NDEBUG 0
 
 #include "BufferLayerConsumer.h"
 
diff --git a/frameworks/native/services/surfaceflinger/ColorLayer.cpp b/frameworks/native/services/surfaceflinger/ColorLayer.cpp
index 1a9021a..e95e89d 100644
--- a/frameworks/native/services/surfaceflinger/ColorLayer.cpp
+++ b/frameworks/native/services/surfaceflinger/ColorLayer.cpp
@@ -14,7 +14,7 @@
  * limitations under the License.
  */
 
-// #define LOG_NDEBUG 0
+#define LOG_NDEBUG 0
 #undef LOG_TAG
 #define LOG_TAG "ColorLayer"
 
diff --git a/frameworks/native/services/surfaceflinger/ContainerLayer.cpp b/frameworks/native/services/surfaceflinger/ContainerLayer.cpp
index f259d93..ae8cc67 100644
--- a/frameworks/native/services/surfaceflinger/ContainerLayer.cpp
+++ b/frameworks/native/services/surfaceflinger/ContainerLayer.cpp
@@ -14,7 +14,7 @@
  * limitations under the License.
  */
 
-// #define LOG_NDEBUG 0
+#define LOG_NDEBUG 0
 #undef LOG_TAG
 #define LOG_TAG "ContainerLayer"
 
diff --git a/frameworks/native/services/surfaceflinger/DispSync.cpp b/frameworks/native/services/surfaceflinger/DispSync.cpp
index 7acbd11..a98268c 100644
--- a/frameworks/native/services/surfaceflinger/DispSync.cpp
+++ b/frameworks/native/services/surfaceflinger/DispSync.cpp
@@ -15,7 +15,7 @@
  */
 
 #define ATRACE_TAG ATRACE_TAG_GRAPHICS
-//#define LOG_NDEBUG 0
+#define LOG_NDEBUG 0
 
 // This is needed for stdint.h to define INT64_MAX in C++
 #define __STDC_LIMIT_MACROS
diff --git a/frameworks/native/services/surfaceflinger/DisplayDevice.cpp b/frameworks/native/services/surfaceflinger/DisplayDevice.cpp
index 4801ba0..ed303d8 100644
--- a/frameworks/native/services/surfaceflinger/DisplayDevice.cpp
+++ b/frameworks/native/services/surfaceflinger/DisplayDevice.cpp
@@ -14,7 +14,7 @@
  * limitations under the License.
  */
 
-// #define LOG_NDEBUG 0
+#define LOG_NDEBUG 0
 #undef LOG_TAG
 #define LOG_TAG "DisplayDevice"
 
diff --git a/frameworks/native/services/surfaceflinger/DisplayHardware/FramebufferSurface.cpp b/frameworks/native/services/surfaceflinger/DisplayHardware/FramebufferSurface.cpp
index daabd3d..4efb1e6 100644
--- a/frameworks/native/services/surfaceflinger/DisplayHardware/FramebufferSurface.cpp
+++ b/frameworks/native/services/surfaceflinger/DisplayHardware/FramebufferSurface.cpp
@@ -15,7 +15,7 @@
  ** limitations under the License.
  */
 
-// #define LOG_NDEBUG 0
+#define LOG_NDEBUG 0
 #undef LOG_TAG
 #define LOG_TAG "FramebufferSurface"
 
diff --git a/frameworks/native/services/surfaceflinger/DisplayHardware/HWC2.cpp b/frameworks/native/services/surfaceflinger/DisplayHardware/HWC2.cpp
index 067257c..06d48c9 100644
--- a/frameworks/native/services/surfaceflinger/DisplayHardware/HWC2.cpp
+++ b/frameworks/native/services/surfaceflinger/DisplayHardware/HWC2.cpp
@@ -14,7 +14,7 @@
  * limitations under the License.
  */
 
-// #define LOG_NDEBUG 0
+#define LOG_NDEBUG 0
 
 #undef LOG_TAG
 #define LOG_TAG "HWC2"
diff --git a/frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp b/frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp
index 89777d0..60f661a 100644
--- a/frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp
+++ b/frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp
@@ -14,7 +14,7 @@
  * limitations under the License.
  */
 
-// #define LOG_NDEBUG 0
+#define LOG_NDEBUG 0
 
 #undef LOG_TAG
 #define LOG_TAG "HWComposer"
diff --git a/frameworks/native/services/surfaceflinger/DisplayHardware/VirtualDisplaySurface.cpp b/frameworks/native/services/surfaceflinger/DisplayHardware/VirtualDisplaySurface.cpp
index ff1482c..25826c3 100644
--- a/frameworks/native/services/surfaceflinger/DisplayHardware/VirtualDisplaySurface.cpp
+++ b/frameworks/native/services/surfaceflinger/DisplayHardware/VirtualDisplaySurface.cpp
@@ -14,7 +14,7 @@
  * limitations under the License.
  */
 
-// #define LOG_NDEBUG 0
+#define LOG_NDEBUG 0
 #include "VirtualDisplaySurface.h"
 
 #include <inttypes.h>
diff --git a/frameworks/native/services/surfaceflinger/Layer.cpp b/frameworks/native/services/surfaceflinger/Layer.cpp
index 07ba345..2651d34 100644
--- a/frameworks/native/services/surfaceflinger/Layer.cpp
+++ b/frameworks/native/services/surfaceflinger/Layer.cpp
@@ -14,7 +14,7 @@
  * limitations under the License.
  */
 
-//#define LOG_NDEBUG 0
+#define LOG_NDEBUG 0
 #undef LOG_TAG
 #define LOG_TAG "Layer"
 #define ATRACE_TAG ATRACE_TAG_GRAPHICS
diff --git a/frameworks/native/services/surfaceflinger/RenderEngine/GLES20RenderEngine.cpp b/frameworks/native/services/surfaceflinger/RenderEngine/GLES20RenderEngine.cpp
index 0048000..c1829c2 100644
--- a/frameworks/native/services/surfaceflinger/RenderEngine/GLES20RenderEngine.cpp
+++ b/frameworks/native/services/surfaceflinger/RenderEngine/GLES20RenderEngine.cpp
@@ -14,7 +14,7 @@
  * limitations under the License.
  */
 
-//#define LOG_NDEBUG 0
+#define LOG_NDEBUG 0
 #undef LOG_TAG
 #define LOG_TAG "RenderEngine"
 #define ATRACE_TAG ATRACE_TAG_GRAPHICS
diff --git a/frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp b/frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
index de7ff36..f2ba5cc 100644
--- a/frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
+++ b/frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
@@ -14,7 +14,7 @@
  * limitations under the License.
  */
 
-// #define LOG_NDEBUG 0
+#define LOG_NDEBUG 0
 #define ATRACE_TAG ATRACE_TAG_GRAPHICS
 
 #include <stdint.h>
@@ -2167,6 +2167,7 @@ void SurfaceFlinger::postFramebuffer()
         }
         const auto hwcId = displayDevice->getHwcDisplayId();
         if (hwcId >= 0) {
+            //if didn't call,the screen will black no pixel
             getBE().mHwc->presentAndGetReleaseFences(hwcId);
         }
         displayDevice->onSwapBuffersCompleted();

```
##### ③完整Log
[mainsystem-ams-wms-red-startActivity.log](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/others/mainsystem-ams-wms-red-startActivity.log)


####  一、Activity概述
基于Android 9.0的源码剖析， 分析Android Activity启动流程：

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

![enter image description here](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/personal.website/zjj.display.sys.ActivityStack_TaskRecord_ActivityRecord.png)


另外还有一点需要注意的是，ActivityManagerService和WindowManagerService中的Task和Stack结构是一一对应的，对应关系对于如下：

> ActivityStack <–> TaskStack
>  TaskRecord   <–> Task

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

![enter image description here](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/personal.website/zjj.display.sys.ams_class_member.png)



##### 1.4、小结：

总体概览图：
![enter image description here](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/personal.website/zjj.display.sys.startActivity_fork_new_process.png)


1、点击桌面App图标，Launcher进程采用Binder IPC向system_server进程发起startActivity请求；
2、system_server进程接收到请求后，向zygote进程发送创建进程的请求；
3、Zygote进程fork出新的子进程，即App进程；
4、App进程，通过Binder IPC向sytem_server进程发起attachApplication请求；
5、system_server进程在收到请求后，进行一系列准备工作后，再通过binder IPC向App进程发送scheduleLaunchActivity请求；
6、App进程的binder线程（ApplicationThread）在收到请求后，通过handler向主线程发送LAUNCH_ACTIVITY消息；
7、主线程在收到Message后，通过发射机制创建目标Activity，并回调Activity.onCreate()等方法。

####  二、APP请求启动Activity
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
        ......
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

##### 2.3、Instrumentation.execStartActivity()

``` java
[-> Instrumentation.java]
    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        android.util.SeempLog.record_str(377, intent.toString());
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        ......
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
```

``` java
Android 9.0已改变没有直接提供Java源码，而是.class文件打包为classes-header.jar和classes.jar。
android\out\target\common\obj\JAVA_LIBRARIES\framework_intermediates\
classes-header.jar
classes.jar解压：
classes/android/app/
    IActivityManager.class
    IActivityManager$Stub.class
    IActivityManager$Stub$Proxy.class

[解压后在线反编译](http://www.javadecompilers.com/)
or
[解压后在线反编译](http://javare.cn/)
-----------------------------------------------------------------------------------------------
IActivityManager$Stub.java
   public static IActivityManager asInterface(IBinder obj) {
      if(obj == null) {
         return null;
      } else {
         IInterface iin = obj.queryLocalInterface("android.app.IActivityManager");
         return (IActivityManager)(iin != null && iin instanceof IActivityManager?(IActivityManager)iin:new Proxy(obj));
      }
   }

  IActivityManager$Stub$Proxy.java

  IActivityManager$Stub$Proxy(IBinder remote)
  {
    mRemote = remote;
  }
-----------------------------------------------------------------------------------------------

```

最后直接返回一个android.app.IActivityManager.Stub.Proxy对象，
此处startActivity()的共有10个参数, 下面说说每个参数传递AMP.startActivity()每一项的对应值。

好吧，最后直接返回一个对象，而继承与IActivityManager，到了这里就引出了我们android系统中很重要的一个概念：Binder机制。我们知道应用进程与SystemServer进程属于两个不同的进程，进程之间需要通讯，android系统采取了自身设计的Binder机制，这里的和ActivityManagerNative都是继承与IActivityManager的而SystemServer进程中的ActivityManagerService对象则继承与ActivityManagerNative。简单的表示： 

> IActivityManager.aidl -> Binder机制 -> ActivityManagerService

这样，IActivityManager.Stub.Proxy与相当于一个Binder的客户端而ActivityManagerService相当于Binder的服务端，这样当IActivityManager.Stub.Proxy调用接口方法的时候底层通过Binder driver就会将请求数据与请求传递给server端，并在server端执行具体的接口逻辑。

好了，说了这么多我们知道这里的IActivityManager.Stub.Proxy是ActivityManagerService在应用进程的一个client就好了，通过它就可以滴啊用ActivityManagerService的方法了。

##### 2.4、IActivityManager.Stub.Proxy.startActivity()
ActivityManager.getService()方法会返回一个对象，那么我们看一下对象的startActivity方法：

``` java
IActivityManager.Stub.Proxy.java

public int startActivity(IApplicationThread caller, String callingPackage, Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode, int flags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
    Parcel _data = Parcel.obtain();
    Parcel _reply = Parcel.obtain();
    try
    {
      _data.writeInterfaceToken("android.app.IActivityManager");
      _data.writeStrongBinder(caller != null ? caller.asBinder() : null);
      _data.writeString(callingPackage);
      if (intent != null) {
        _data.writeInt(1);
        intent.writeToParcel(_data, 0);
      }
      else {
        _data.writeInt(0);
      }
      _data.writeString(resolvedType);
      _data.writeStrongBinder(resultTo);
      _data.writeString(resultWho);
      _data.writeInt(requestCode);
      _data.writeInt(flags);
      if (profilerInfo != null) {
        _data.writeInt(1);
        profilerInfo.writeToParcel(_data, 0);
      }
      else {
        _data.writeInt(0);
      }
      if (options != null) {
        _data.writeInt(1);
        options.writeToParcel(_data, 0);
      }
      else {
        _data.writeInt(0);
      }
      mRemote.transact(6, _data, _reply, 0);
      _reply.readException();
      _result = _reply.readInt();
    } finally {
      int _result;
      _reply.recycle();
      _data.recycle(); }
    int _result;
    return _result;
  }
  
```

##### 2.5、IActivityManager.Stub.onTransact()

``` java
IActivityManager.Stub.java

   public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
      String descriptor = "android.app.IActivityManager";
      ComponentName _arg0;
      boolean _result;
      ......
      switch(code) {
    switch (code) {
      case 6:
         return this.onTransact$startActivity$(data, reply);
         
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
```java
[->ActivityStarter.java]
    /**
     * Starts an activity based on the request parameters provided earlier.
     * @return The starter result.
     */
    int execute() {
        try {
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
---------------------------------------------------------------------------------------
04-16 13:08:36.782 I/ActivityMetricsLogger( 1049): notifyActivityLaunched resultCode=0 launchedActivity=ActivityRecord{65cc87a u0 com.android.testred/.TestActivity t5} processRunning=false processSwitch=true
---------------------------------------------------------------------------------------
            mSupervisor.getActivityMetricsLogger().notifyActivityLaunched(res, outRecord[0]);
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


    private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
            String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
            String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
            SafeActivityOptions options,
            boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity,
            TaskRecord inTask, boolean allowPendingRemoteAnimationRegistryLookup) {
        int err = ActivityManager.START_SUCCESS;
        // Pull the optional Ephemeral Installer-only bundle out of the options early.
        final Bundle verificationBundle
                = options != null ? options.popAppVerificationBundle() : null;

        ProcessRecord callerApp = null;
        if (caller != null) {
            callerApp = mService.getRecordForAppLocked(caller);
            if (callerApp != null) {
                callingPid = callerApp.pid;
                callingUid = callerApp.info.uid;
            } else {
                Slog.w(TAG, "Unable to find app for caller " + caller
                        + " (pid=" + callingPid + ") when starting: "
                        + intent.toString());
                err = ActivityManager.START_PERMISSION_DENIED;
            }
        }

        final int userId = aInfo != null && aInfo.applicationInfo != null
                ? UserHandle.getUserId(aInfo.applicationInfo.uid) : 0;

        if (err == ActivityManager.START_SUCCESS) {
            Slog.i(TAG, "START u" + userId + " {" + intent.toShortString(true, true, true, false)
                    + "} from uid " + callingUid);
        }

        ActivityRecord sourceRecord = null;
        ActivityRecord resultRecord = null;
        if (resultTo != null) {
            sourceRecord = mSupervisor.isInAnyStackLocked(resultTo);
            if (DEBUG_RESULTS) Slog.v(TAG_RESULTS,
                    "Will send result to " + resultTo + " " + sourceRecord);
            if (sourceRecord != null) {
                if (requestCode >= 0 && !sourceRecord.finishing) {
                    resultRecord = sourceRecord;
                }
            }
        }

        final int launchFlags = intent.getFlags();

        if ((launchFlags & Intent.FLAG_ACTIVITY_FORWARD_RESULT) != 0 && sourceRecord != null) {
            // Transfer the result target from the source activity to the new
            // one being started, including any failures.
            if (requestCode >= 0) {
                SafeActivityOptions.abort(options);
                return ActivityManager.START_FORWARD_AND_REQUEST_CONFLICT;
            }
            resultRecord = sourceRecord.resultTo;
            if (resultRecord != null && !resultRecord.isInStackLocked()) {
                resultRecord = null;
            }
            resultWho = sourceRecord.resultWho;
            requestCode = sourceRecord.requestCode;
            sourceRecord.resultTo = null;
            if (resultRecord != null) {
                resultRecord.removeResultsLocked(sourceRecord, resultWho, requestCode);
            }
            if (sourceRecord.launchedFromUid == callingUid) {
                callingPackage = sourceRecord.launchedFromPackage;
            }
        }

        ......

        final ActivityStack resultStack = resultRecord == null ? null : resultRecord.getStack();

        ......

        boolean abort = !mSupervisor.checkStartAnyActivityPermission(intent, aInfo, resultWho,
                requestCode, callingPid, callingUid, callingPackage, ignoreTargetSecurity,
                inTask != null, callerApp, resultRecord, resultStack);
        abort |= !mService.mIntentFirewall.checkStartActivity(intent, callingUid,
                callingPid, resolvedType, aInfo.applicationInfo);

        // Merge the two options bundles, while realCallerOptions takes precedence.
        ActivityOptions checkedOptions = options != null
                ? options.getOptions(intent, aInfo, callerApp, mSupervisor)
                : null;
        if (allowPendingRemoteAnimationRegistryLookup) {
            checkedOptions = mService.getActivityStartController()
                    .getPendingRemoteAnimationRegistry()
                    .overrideOptionsIfNeeded(callingPackage, checkedOptions);
        }
        if (mService.mController != null) {
            try {
                // The Intent we give to the watcher has the extra data
                // stripped off, since it can contain private information.
                Intent watchIntent = intent.cloneFilter();
                abort |= !mService.mController.activityStarting(watchIntent,
                        aInfo.applicationInfo.packageName);
            } catch (RemoteException e) {
                mService.mController = null;
            }
        }

        mInterceptor.setStates(userId, realCallingPid, realCallingUid, startFlags, callingPackage);
        ........
        // If permissions need a review before any of the app components can run, we
        // launch the review activity and pass a pending intent to start the activity
        // we are to launching now after the review is completed.
        if (mService.mPermissionReviewRequired && aInfo != null) {
            if (mService.getPackageManagerInternalLocked().isPermissionsReviewRequired(
                    aInfo.packageName, userId)) {
                IIntentSender target = mService.getIntentSenderLocked(
                        ActivityManager.INTENT_SENDER_ACTIVITY, callingPackage,
                        callingUid, userId, null, null, 0, new Intent[]{intent},
                        new String[]{resolvedType}, PendingIntent.FLAG_CANCEL_CURRENT
                                | PendingIntent.FLAG_ONE_SHOT, null);

                final int flags = intent.getFlags();
                Intent newIntent = new Intent(Intent.ACTION_REVIEW_PERMISSIONS);
                newIntent.setFlags(flags
                        | Intent.FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS);
                newIntent.putExtra(Intent.EXTRA_PACKAGE_NAME, aInfo.packageName);
                newIntent.putExtra(Intent.EXTRA_INTENT, new IntentSender(target));
                if (resultRecord != null) {
                    newIntent.putExtra(Intent.EXTRA_RESULT_NEEDED, true);
                }
                intent = newIntent;

                resolvedType = null;
                callingUid = realCallingUid;
                callingPid = realCallingPid;

                rInfo = mSupervisor.resolveIntent(intent, resolvedType, userId, 0,
                        computeResolveFilterUid(
                                callingUid, realCallingUid, mRequest.filterCallingUid));
                aInfo = mSupervisor.resolveActivity(intent, rInfo, startFlags,
                        null /*profilerInfo*/);

                if (DEBUG_PERMISSIONS_REVIEW) {
                    Slog.i(TAG, "START u" + userId + " {" + intent.toShortString(true, true,
                            true, false) + "} from uid " + callingUid + " on display "
                            + (mSupervisor.mFocusedStack == null
                            ? DEFAULT_DISPLAY : mSupervisor.mFocusedStack.mDisplayId));
                }
            }
        }
        
        if (rInfo != null && rInfo.auxiliaryInfo != null) {
            intent = createLaunchIntent(rInfo.auxiliaryInfo, ephemeralIntent,
                    callingPackage, verificationBundle, resolvedType, userId);
            resolvedType = null;
            callingUid = realCallingUid;
            callingPid = realCallingPid;

            aInfo = mSupervisor.resolveActivity(intent, rInfo, startFlags, null /*profilerInfo*/);
        }
        // 创建ActivityRecord
        ActivityRecord r = new ActivityRecord(mService, callerApp, callingPid, callingUid,
                callingPackage, intent, resolvedType, aInfo, mService.getGlobalConfiguration(),
                resultRecord, resultWho, requestCode, componentSpecified, voiceSession != null,
                mSupervisor, checkedOptions, sourceRecord);
        if (outActivity != null) {
            outActivity[0] = r;
        }

       ......

        final ActivityStack stack = mSupervisor.mFocusedStack;
        ......
        mController.doPendingActivityLaunches(false);

        return startActivity(r, sourceRecord, voiceSession, voiceInteractor, startFlags,
                true /* doResume */, checkedOptions, inTask, outActivity);
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
        boolean newTask = false;
        final TaskRecord taskToAffiliate = (mLaunchTaskBehind && mSourceRecord != null)
                ? mSourceRecord.getTask() : null;

        // Should this be considered a new task?
        int result = START_SUCCESS;
        if (mStartActivity.resultTo == null && mInTask == null && !mAddingToTask
                && (mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) {
            newTask = true;
            String packageName= mService.mContext.getPackageName();
            if (mPerf != null) {
                    mPerf.perfHint(BoostFramework.VENDOR_HINT_FIRST_LAUNCH_BOOST,
                                        packageName, -1, BoostFramework.Launch.BOOST_V1);
            }
            result = setTaskFromReuseOrCreateNewTask(taskToAffiliate, topStack);
        } else if (mSourceRecord != null) {
            result = setTaskFromSourceRecord();
        } else if (mInTask != null) {
            result = setTaskFromInTask();
        } else {
            // This not being started from an existing activity, and not part of a new task...
            // just put it in the top task, though these days this case should never happen.
            setTaskToCurrentTopOrCreateNewTask();
        }
        if (result != START_SUCCESS) {
            return result;
        }
        ......
        mTargetStack.startActivityLocked(mStartActivity, topFocused, newTask, mKeepCurTransition,
                mOptions);
        if (mDoResume) {
            final ActivityRecord topTaskActivity =
                    mStartActivity.getTask().topRunningActivityLocked();
            if (!mTargetStack.isFocusable()
                    || (topTaskActivity != null && topTaskActivity.mTaskOverlay
                    && mStartActivity != topTaskActivity)) {
                mTargetStack.ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
                mService.mWindowManager.executeAppTransition();
            } else {
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

##### 3.4.1、ActivityStarter.setTaskFromReuseOrCreateNewTask()-创建新的Task[AMS端]

在此情况下Activity启动Flag为FLAG_ACTIVITY_NEW_TASK

``` java
[->ActivityStarter.java]

    private int setTaskFromReuseOrCreateNewTask(
            TaskRecord taskToAffiliate, ActivityStack topStack) {
        mTargetStack = computeStackFocus(mStartActivity, true, mLaunchFlags, mOptions);

        // Do no move the target stack to front yet, as we might bail if
        // isLockTaskModeViolation fails below.

        if (mReuseTask == null) {
            final TaskRecord task = mTargetStack.createTaskRecord(
                    mSupervisor.getNextTaskIdForUserLocked(mStartActivity.userId),
                    mNewTaskInfo != null ? mNewTaskInfo : mStartActivity.info,
                    mNewTaskIntent != null ? mNewTaskIntent : mIntent, mVoiceSession,
                    mVoiceInteractor, !mLaunchTaskBehind /* toTop */, mStartActivity, mSourceRecord,
                    mOptions);
            addOrReparentStartingActivity(task, "setTaskFromReuseOrCreateNewTask - mReuseTask");
            updateBounds(mStartActivity.getTask(), mLaunchParams.mBounds);

            if (DEBUG_TASKS) Slog.v(TAG_TASKS, "Starting new activity " + mStartActivity
                    + " in new task " + mStartActivity.getTask());
------------------------------------------------------------------------------------------------
此处对应Log：

04-16 13:08:36.757 D/WindowManager( 1049): positionTask: task={stackId=1 tasks=[{taskId=5 appTokens=[] mdr=false}]} position=2147483647

04-16 13:08:36.758 V/ActivityStackSupervisor_Tasks( 1049): Starting new activity ActivityRecord{65cc87a u0 com.android.testred/.TestActivity t5} in new task TaskRecord{d697e21 #5 A=com.android.testred U=0 StackId=1 sz=1}
------------------------------------------------------------------------------------------------
        } else {
            addOrReparentStartingActivity(mReuseTask, "setTaskFromReuseOrCreateNewTask");
        }

        if (taskToAffiliate != null) {
            mStartActivity.setTaskToAffiliateWith(taskToAffiliate);
        }

        if (mService.getLockTaskController().isLockTaskModeViolation(mStartActivity.getTask())) {
            Slog.e(TAG, "Attempted Lock Task Mode violation mStartActivity=" + mStartActivity);
            return START_RETURN_LOCK_TASK_MODE_VIOLATION;
        }

        if (mDoResume) {
            mTargetStack.moveToFront("reuseOrNewTask");
        }
        return START_SUCCESS;
    }
```


##### 3.4.1.1、TaskRecord创建createTaskRecord()

``` java
[-> ActivityStack.java]
    TaskRecord createTaskRecord(int taskId, ActivityInfo info, Intent intent,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            boolean toTop, ActivityRecord activity, ActivityRecord source,
            ActivityOptions options) {
        //创建TaskRecord
        final TaskRecord task = TaskRecord.create(
                mService, taskId, info, intent, voiceSession, voiceInteractor);
        // add the task to stack first, mTaskPositioner might need the stack association
        
        //将TaskRecord加入到ActivityStack中
        addTask(task, toTop, "createTaskRecord");
        final int displayId = mDisplayId != INVALID_DISPLAY ? mDisplayId : DEFAULT_DISPLAY;
        final boolean isLockscreenShown = mService.mStackSupervisor.getKeyguardController()
                .isKeyguardOrAodShowing(displayId);
        if (!mStackSupervisor.getLaunchParamsController()
                .layoutTask(task, info.windowLayout, activity, source, options)
                && !matchParentBounds() && task.isResizeable() && !isLockscreenShown) {
            task.updateOverrideConfiguration(getOverrideBounds());
        }
        //创建WindowContainer
        task.createWindowContainer(toTop, (info.flags & FLAG_SHOW_FOR_ALL_USERS) != 0);
        return task;
    }
```


#####  3.4.1.1.1、TaskRecord.create()


```java
[-> TaskRecord.java]
    static TaskRecord create(ActivityManagerService service, int taskId, ActivityInfo info,
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
    /**
     * Don't use constructor directly. Use {@link #create(ActivityManagerService, int, ActivityInfo,
     * Intent, TaskDescription)} instead.
     */
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


##### 3.4.1.1.2、将TaskRecord加入到ActivityStack中



``` java
[-> TaskRecord.java]
    void addTask(final TaskRecord task, final boolean toTop, String reason) {
        addTask(task, toTop ? MAX_VALUE : 0, true /* schedulePictureInPictureModeChange */, reason);
        if (toTop) {
            // TODO: figure-out a way to remove this call.
            mWindowContainerController.positionChildAtTop(task.getWindowContainerController(),
                    true /* includingParents */);
        }
    }
    void addTask(final TaskRecord task, int position, boolean schedulePictureInPictureModeChange,
            String reason) {
        // TODO: Is this remove really needed? Need to look into the call path for the other addTask
        mTaskHistory.remove(task);
        position = getAdjustedPositionForTask(task, position, null /* starting */);
        final boolean toTop = position >= mTaskHistory.size();
        final ActivityStack prevStack = preAddTask(task, reason, toTop);
        //将TaskRecord加入到mTaskHistory中
        mTaskHistory.add(position, task);
        //设置对应的setStack
        task.setStack(this);

        updateTaskMovement(task, toTop);

        postAddTask(task, prevStack, schedulePictureInPictureModeChange);
    }
```


##### 3.4.1.1.3、TaskRecord.createWindowContainer()

``` java
[-> TaskRecord.java]
    void createWindowContainer(boolean onTop, boolean showForAllUsers) {
        if (mWindowContainerController != null) {
            throw new IllegalArgumentException("Window container=" + mWindowContainerController
                    + " already created for task=" + this);
        }

        final Rect bounds = updateOverrideConfigurationFromLaunchBounds();
        setWindowContainerController(new TaskWindowContainerController(taskId, this,
                getStack().getWindowContainerController(), userId, bounds,
                mResizeMode, mSupportsPictureInPicture, onTop,
                showForAllUsers, lastTaskDescription));
    }


    public TaskWindowContainerController(int taskId, TaskWindowContainerListener listener,
            StackWindowController stackController, int userId, Rect bounds, int resizeMode,
            boolean supportsPictureInPicture, boolean toTop, boolean showForAllUsers,
            TaskDescription taskDescription) {
        this(taskId, listener, stackController, userId, bounds, resizeMode,
                supportsPictureInPicture, toTop, showForAllUsers, taskDescription,
                WindowManagerService.getInstance());
    }
[-> TaskWindowContainerController.java]
    public TaskWindowContainerController(int taskId, TaskWindowContainerListener listener,
            StackWindowController stackController, int userId, Rect bounds, int resizeMode,
            boolean supportsPictureInPicture, boolean toTop, boolean showForAllUsers,
            TaskDescription taskDescription, WindowManagerService service) {
        super(listener, service);
        mTaskId = taskId;
        mHandler = new H(new WeakReference<>(this), service.mH.getLooper());

        synchronized(mWindowMap) {
            if (DEBUG_STACK) Slog.i(TAG_WM, "TaskWindowContainerController: taskId=" + taskId
                    + " stack=" + stackController + " bounds=" + bounds);
            //TaskStack，对应ActivityStack
            final TaskStack stack = stackController.mContainer;
            if (stack == null) {
                throw new IllegalArgumentException("TaskWindowContainerController: invalid stack="
                        + stackController);
            }
            EventLog.writeEvent(WM_TASK_CREATED, taskId, stack.mStackId);
            //根据AMS端的taskId创建WMS端的Task，对应AMS的TaskRecord
            final Task task = createTask(taskId, stack, userId, resizeMode,
                    supportsPictureInPicture, taskDescription);
            final int position = toTop ? POSITION_TOP : POSITION_BOTTOM;
            // We only want to move the parents to the parents if we are creating this task at the
            // top of its stack.
            stack.addTask(task, position, showForAllUsers, toTop /* moveParents */);
        }
    }

```
##### 3.4.1.1.4、TaskWindowContainerController.createTask()[WMS端的Task]

``` java
[-> TaskWindowContainerController.java]
    @VisibleForTesting
    Task createTask(int taskId, TaskStack stack, int userId, int resizeMode,
            boolean supportsPictureInPicture, TaskDescription taskDescription) {
        return new Task(taskId, stack, userId, mService, resizeMode, supportsPictureInPicture,
                taskDescription, this);
    }
[-> G:\android9.0\frameworks\base\services\core\java\com\android\server\wm\Task.java]
    Task(int taskId, TaskStack stack, int userId, WindowManagerService service, int resizeMode,
            boolean supportsPictureInPicture, TaskDescription taskDescription,
            TaskWindowContainerController controller) {
        super(service);
        mTaskId = taskId;
        mStack = stack;
        mUserId = userId;
        mResizeMode = resizeMode;
        mSupportsPictureInPicture = supportsPictureInPicture;
        setController(controller);
        setBounds(getOverrideBounds());
        mTaskDescription = taskDescription;

        // Tasks have no set orientation value (including SCREEN_ORIENTATION_UNSPECIFIED).
        setOrientation(SCREEN_ORIENTATION_UNSET);
    }
```
##### 3.4.1.1.5、将Task[WMS]加入到TaskStack 

```java
[-> G:\android9.0\frameworks\base\services\core\java\com\android\server\wm\TaskStack.java]
    void addTask(Task task, int position, boolean showForAllUsers, boolean moveParents) {
        final TaskStack currentStack = task.mStack;
        // TODO: We pass stack to task's constructor, but we still need to call this method.
        // This doesn't make sense, mStack will already be set equal to "this" at this point.
        if (currentStack != null && currentStack.mStackId != mStackId) {
            throw new IllegalStateException("Trying to add taskId=" + task.mTaskId
                    + " to stackId=" + mStackId
                    + ", but it is already attached to stackId=" + task.mStack.mStackId);
        }

        // Add child task.
        task.mStack = this;
        addChild(task, null);

        // Move child to a proper position, as some restriction for position might apply.
        positionChildAt(position, task, moveParents /* includingParents */, showForAllUsers);
    }

    @Override
    void positionChildAt(int position, Task child, boolean includingParents) {
        positionChildAt(position, child, includingParents, child.showForAllUsers());
    }

    private void positionChildAt(int position, Task child, boolean includingParents,
            boolean showForAllUsers) {
        final int targetPosition = findPositionForTask(child, position, showForAllUsers,
                false /* addingNew */);
        super.positionChildAt(targetPosition, child, includingParents);

        // Log positioning.
        if (DEBUG_TASK_MOVEMENT)
            Slog.d(TAG_WM, "positionTask: task=" + this + " position=" + position);

        final int toTop = targetPosition == mChildren.size() - 1 ? 1 : 0;
        EventLog.writeEvent(EventLogTags.WM_TASK_MOVED, child.mTaskId, toTop, targetPosition);
    }
```

##### 3.4.2、ActivityStack.startActivityLocked()

``` java
[-> ActivityStack.java]
    void startActivityLocked(ActivityRecord r, ActivityRecord focusedTopActivity,
            boolean newTask, boolean keepCurTransition, ActivityOptions options) {
        TaskRecord rTask = r.getTask();
        final int taskId = rTask.taskId;
        // mLaunchTaskBehind tasks get placed at the back of the task stack.
        if (!r.mLaunchTaskBehind && (taskForIdLocked(taskId) == null || newTask)) {
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
                    if (!startIt) {
                        if (DEBUG_ADD_REMOVE) Slog.i(TAG, "Adding activity " + r + " to task "
                                + task, new RuntimeException("here").fillInStackTrace());
                        r.createWindowContainer();
                        ActivityOptions.abort(options);
                        return;
                    }
                    break;
                } else if (task.numFullscreen > 0) {
                    startIt = false;
                }
            }
        }

        final TaskRecord activityTask = r.getTask();
        if (task == activityTask && mTaskHistory.indexOf(task) != (mTaskHistory.size() - 1)) {
            mStackSupervisor.mUserLeaving = false;
            if (DEBUG_USER_LEAVING) Slog.v(TAG_USER_LEAVING,
                    "startActivity() behind front, mUserLeaving=false");
        }

        task = activityTask;
-----------------------------------------------------------------------------------------------
此处对应Log：
04-16 13:08:36.765 I/ActivityStack( 1049): Adding activity ActivityRecord{65cc87a u0 com.android.testred/.TestActivity t5} to stack to task TaskRecord{d697e21 #5 A=com.android.testred U=0 StackId=1 sz=1}
04-16 13:08:36.765 I/ActivityStack( 1049): java.lang.RuntimeException: here
04-16 13:08:36.765 I/ActivityStack( 1049): 	at com.android.server.am.ActivityStack.startActivityLocked(ActivityStack.java:2963)
04-16 13:08:36.765 I/ActivityStack( 1049): 	at com.android.server.am.ActivityStarter.startActivityUnchecked(ActivityStarter.java:1448)
04-16 13:08:36.765 I/ActivityStack( 1049): 	at com.android.server.am.ActivityStarter.startActivity(ActivityStarter.java:1204)
04-16 13:08:36.765 I/ActivityStack( 1049): 	at com.android.server.am.ActivityStarter.startActivity(ActivityStarter.java:872)
04-16 13:08:36.765 I/ActivityStack( 1049): 	at com.android.server.am.ActivityStarter.startActivity(ActivityStarter.java:548)
04-16 13:08:36.765 I/ActivityStack( 1049): 	at com.android.server.am.ActivityStarter.startActivityMayWait(ActivityStarter.java:1103)
04-16 13:08:36.765 I/ActivityStack( 1049): 	at com.android.server.am.ActivityStarter.execute(ActivityStarter.java:490)
04-16 13:08:36.765 I/ActivityStack( 1049): 	at com.android.server.am.ActivityManagerService.startActivityAsUser(ActivityManagerService.java:5182)
04-16 13:08:36.765 I/ActivityStack( 1049): 	at com.android.server.am.ActivityManagerService.startActivityAsUser(ActivityManagerService.java:5156)
04-16 13:08:36.765 I/ActivityStack( 1049): 	at com.android.server.am.ActivityManagerShellCommand.runStartActivity(ActivityManagerShellCommand.java:479)
04-16 13:08:36.765 I/ActivityStack( 1049): 	at com.android.server.am.ActivityManagerShellCommand.onCommand(ActivityManagerShellCommand.java:161)
04-16 13:08:36.765 I/ActivityStack( 1049): 	at android.os.ShellCommand.exec(ShellCommand.java:103)
04-16 13:08:36.765 I/ActivityStack( 1049): 	at com.android.server.am.ActivityManagerService.onShellCommand(ActivityManagerService.java:16147)
04-16 13:08:36.765 I/ActivityStack( 1049): 	at android.os.Binder.shellCommand(Binder.java:634)
04-16 13:08:36.765 I/ActivityStack( 1049): 	at android.os.Binder.onTransact(Binder.java:532)
04-16 13:08:36.765 I/ActivityStack( 1049): 	at android.app.IActivityManager$Stub.onTransact(IActivityManager.java:3592)
04-16 13:08:36.765 I/ActivityStack( 1049): 	at com.android.server.am.ActivityManagerService.onTransact(ActivityManagerService.java:3340)
04-16 13:08:36.765 I/ActivityStack( 1049): 	at android.os.Binder.execTransact(Binder.java:731)
-----------------------------------------------------------------------------------------------
        // Slot the activity into the history stack and proceed
        if (DEBUG_ADD_REMOVE) Slog.i(TAG, "Adding activity " + r + " to stack to task " + task,
                new RuntimeException("here").fillInStackTrace());

        if (r.getWindowContainerController() == null) {
            r.createWindowContainer();
        }
        //将TaskRecord设置为front task
        task.setFrontOfTask();

        if (mActivityTrigger != null) {
            mActivityTrigger.activityStartTrigger(r.intent, r.info, r.appInfo, r.fullscreen);
        }

        if (mActivityPluginDelegate != null && getWindowingMode() != WINDOWING_MODE_UNDEFINED) {
            mActivityPluginDelegate.activityInvokeNotification
                (r.appInfo.packageName, getWindowingMode() == WINDOWING_MODE_FULLSCREEN);
        }

        if (!isActivityTypeHome() || numActivities() > 0) {
            if (DEBUG_TRANSITION) Slog.v(TAG_TRANSITION,
                    "Prepare open transition: starting " + r);
            if ((r.intent.getFlags() & Intent.FLAG_ACTIVITY_NO_ANIMATION) != 0) {
                mWindowManager.prepareAppTransition(TRANSIT_NONE, keepCurTransition);
                mStackSupervisor.mNoAnimActivities.add(r);
            } else {
                int transit = TRANSIT_ACTIVITY_OPEN;
                if (newTask) {
                    if (r.mLaunchTaskBehind) {
                        transit = TRANSIT_TASK_OPEN_BEHIND;
                    } else {
                        if (canEnterPipOnTaskSwitch(focusedTopActivity,
                                null /* toFrontTask */, r, options)) {
                            focusedTopActivity.supportsEnterPipOnTaskSwitch = true;
                        }
                        transit = TRANSIT_TASK_OPEN;
                    }
                }
                //准备App启动变换
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
                if (prev != null) {
                    if (prev.getTask() != prevTask) {
                        prev = null;
                    }
                    else if (prev.nowVisible) {
                        prev = null;
                    }
                }
                // 显示启动窗口
                r.showStartingWindow(prev, newTask, isTaskSwitch(r, focusedTopActivity));
            }
        } else {
            ActivityOptions.abort(options);
        }
    }



---------------------------------------------------
对应Log：
	Line 1046: 04-16 13:08:36.768 V/ActivityStack_Transition( 1049): Prepare open transition: starting ActivityRecord{65cc87a u0 com.android.testred/.TestActivity t5}
	Line 1051: 04-16 13:08:36.768 V/WindowManager( 1049): setAppStartingWindow: token=Token{ce2ac07 ActivityRecord{65cc87a u0 com.android.testred/.TestActivity t5}} pkg=com.android.testred transferFrom=null newTask=true taskSwitch=true processRunning=false allowTaskSnapshot=true
	Line 1051: 04-16 13:08:36.768 V/WindowManager( 1049): setAppStartingWindow: token=Token{ce2ac07 ActivityRecord{65cc87a u0 com.android.testred/.TestActivity t5}} pkg=com.android.testred transferFrom=null newTask=true taskSwitch=true processRunning=false allowTaskSnapshot=true
	Line 1054: 04-16 13:08:36.769 V/ActivityStack_Switch( 1049): Resuming ActivityRecord{65cc87a u0 com.android.testred/.TestActivity t5}
	Line 1055: 04-16 13:08:36.769 D/ActivityTrigger( 1049): activityResumeTrigger: Activity is not Triggerred in full screen ApplicationInfo{2bedb5d com.android.testred}
---------------------------------------------------
```


分段来看：


``` java
if (r.getWindowContainerController() == null) {
    r.createWindowContainer();
}
task.setFrontOfTask();
```

##### 3.4.2.1、ActivityRecord.createWindowContainer()

``` java

G:\android9.0\frameworks\base\services\core\java\com\android\server\am\ActivityRecord.java
    final IApplicationToken.Stub appToken; // window manager token
    
    void createWindowContainer() {
        ......
        inHistory = true;

        final TaskWindowContainerController taskController = task.getWindowContainerController();
        ......
        mWindowContainerController = new AppWindowContainerController(taskController, appToken,
                this, Integer.MAX_VALUE /* add on top */, info.screenOrientation, fullscreen,
                (info.flags & FLAG_SHOW_FOR_ALL_USERS) != 0, info.configChanges,
                task.voiceSession != null, mLaunchTaskBehind, isAlwaysFocusable(),
                appInfo.targetSdkVersion, mRotationAnimationHint,
                ActivityManagerService.getInputDispatchingTimeoutLocked(this) * 1000000L);
        //将当前ActivityRecord加入到TaskRecord的顶部
        task.addActivityToTop(this);
        ......
    }

G:\android9.0\frameworks\base\services\core\java\com\android\server\wm\AppWindowContainerController.java

    public AppWindowContainerController(TaskWindowContainerController taskController,
            IApplicationToken token, AppWindowContainerListener listener, int index,
            int requestedOrientation, boolean fullscreen, boolean showForAllUsers, int configChanges,
            boolean voiceInteraction, boolean launchTaskBehind, boolean alwaysFocusable,
            int targetSdkVersion, int rotationAnimationHint, long inputDispatchingTimeoutNanos) {
        this(taskController, token, listener, index, requestedOrientation, fullscreen,
                showForAllUsers,
                configChanges, voiceInteraction, launchTaskBehind, alwaysFocusable,
                targetSdkVersion, rotationAnimationHint, inputDispatchingTimeoutNanos,
                WindowManagerService.getInstance());
    }

    public AppWindowContainerController(TaskWindowContainerController taskController,
            IApplicationToken token, AppWindowContainerListener listener, int index,
            int requestedOrientation, boolean fullscreen, boolean showForAllUsers, int configChanges,
            boolean voiceInteraction, boolean launchTaskBehind, boolean alwaysFocusable,
            int targetSdkVersion, int rotationAnimationHint, long inputDispatchingTimeoutNanos,
            WindowManagerService service) {
        super(listener, service);
        mHandler = new H(service.mH.getLooper());
        mToken = token;
        synchronized(mWindowMap) {
            AppWindowToken atoken = mRoot.getAppWindowToken(mToken.asBinder());
            if (atoken != null) {
                // TODO: Should this throw an exception instead?
                Slog.w(TAG_WM, "Attempted to add existing app token: " + mToken);
                return;
            }

            final Task task = taskController.mContainer;
            if (task == null) {
                throw new IllegalArgumentException("AppWindowContainerController: invalid "
                        + " controller=" + taskController);
            }

            atoken = createAppWindow(mService, token, voiceInteraction, task.getDisplayContent(),
                    inputDispatchingTimeoutNanos, fullscreen, showForAllUsers, targetSdkVersion,
                    requestedOrientation, rotationAnimationHint, configChanges, launchTaskBehind,
                    alwaysFocusable, this);

-----------------------------------------------------------------------------------------------
对应Log：
04-16 13:08:36.765 V/WindowManager( 1049): addAppToken: AppWindowToken{1579134 token=Token{ce2ac07 ActivityRecord{65cc87a u0 com.android.testred/.TestActivity t5}}} controller={TaskWindowContainerController taskId=5} at 2147483647

-----------------------------------------------------------------------------------------------

            if (DEBUG_TOKEN_MOVEMENT || DEBUG_ADD_REMOVE) Slog.v(TAG_WM, "addAppToken: " + atoken
                    + " controller=" + taskController + " at " + index);
            task.addChild(atoken, index);
        }
    }

    @VisibleForTesting
    AppWindowToken createAppWindow(WindowManagerService service, IApplicationToken token,
            boolean voiceInteraction, DisplayContent dc, long inputDispatchingTimeoutNanos,
            boolean fullscreen, boolean showForAllUsers, int targetSdk, int orientation,
            int rotationAnimationHint, int configChanges, boolean launchTaskBehind,
            boolean alwaysFocusable, AppWindowContainerController controller) {
        return new AppWindowToken(service, token, voiceInteraction, dc,
                inputDispatchingTimeoutNanos, fullscreen, showForAllUsers, targetSdk, orientation,
                rotationAnimationHint, configChanges, launchTaskBehind, alwaysFocusable,
                controller);
    }

[-> G:\android9.0\frameworks\base\services\core\java\com\android\server\wm\AppWindowToken.java]
    // Non-null only for application tokens.
    final IApplicationToken appToken;
    
    AppWindowToken(WindowManagerService service, IApplicationToken token, boolean voiceInteraction,
            DisplayContent dc, long inputDispatchingTimeoutNanos, boolean fullscreen,
            boolean showForAllUsers, int targetSdk, int orientation, int rotationAnimationHint,
            int configChanges, boolean launchTaskBehind, boolean alwaysFocusable,
            AppWindowContainerController controller) {
        this(service, token, voiceInteraction, dc, fullscreen);
        setController(controller);
        mInputDispatchingTimeoutNanos = inputDispatchingTimeoutNanos;
        mShowForAllUsers = showForAllUsers;
        mTargetSdk = targetSdk;
        mOrientation = orientation;
        layoutConfigChanges = (configChanges & (CONFIG_SCREEN_SIZE | CONFIG_ORIENTATION)) != 0;
        mLaunchTaskBehind = launchTaskBehind;
        mAlwaysFocusable = alwaysFocusable;
        mRotationAnimationHint = rotationAnimationHint;

        // Application tokens start out hidden.
        setHidden(true);
        hiddenRequested = true;
    }


```
主要创建了一个AppWindowToken。


接下来继续看下ActivityStarter.startActivityUnchecked()中的ActivityStackSupervisor.resumeFocusedStackTopActivityLocked()

##### 3.5、ActivityStackSupervisor.resumeFocusedStackTopActivityLocked()

``` java
    [->ActivityStackSupervisor.java]
    boolean resumeFocusedStackTopActivityLocked(
            ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {

        if (!readyToResume()) {
            return false;
        }

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

```java
[->ActivityStack.java]
    @GuardedBy("mService")
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

            // When resuming the top activity, it may be necessary to pause the top activity (for
            // example, returning to the lock screen. We suppress the normal pause logic in
            // {@link #resumeTopActivityUncheckedLocked}, since the top activity is resumed at the
            // end. We call the {@link ActivityStackSupervisor#checkReadyForSleepLocked} again here
            // to ensure any necessary pause logic occurs. In the case where the Activity will be
            // shown regardless of the lock screen, the call to
            // {@link ActivityStackSupervisor#checkReadyForSleepLocked} is skipped.
            final ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);
            if (next == null || !next.canTurnScreenOn()) {
                checkReadyForSleep();
            }
        } finally {
            mStackSupervisor.inResumeTopActivity = false;
        }

        return result;
    }
```

##### 3.7、ActivityStack.resumeTopActivityInnerLocked()

说明：启动一个新Activity时，如果界面还存在其它的Activity，那么必须先zan其它的Activity。 
因此，除了第一个启动的Home界面对应的Activity外，其它的Activity均需要进行此操作，当系统启动第一个Activity，即Home时，mResumedActivity的值才会为null。 
经过一系列处理逻辑之后最终调用了startPausingLocked方法，这个方法作用就是让系统中栈中的Activity执行onPause方法。


``` java
    [->ActivityStack.java]
   @GuardedBy("mService")
    private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
        ......
        // Find the next top-most activity to resume in this stack that is not finishing and is
        // focusable. If it is not focusable, we will fall into the case below to resume the
        // top activity in the next focusable task.
        final ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);

        final boolean hasRunningActivity = next != null;

        ......

        // The activity may be waiting for stop, but that is no longer
        // appropriate for it.
        mStackSupervisor.mStoppingActivities.remove(next);
        mStackSupervisor.mGoingToSleepActivities.remove(next);
        next.sleeping = false;
        mStackSupervisor.mActivitiesWaitingForVisibleActivity.remove(next);
        next.launching = true;

        if (DEBUG_SWITCH) Slog.v(TAG_SWITCH, "Resuming " + next);

        if (mActivityTrigger != null) {
            mActivityTrigger.activityResumeTrigger(next.intent, next.info, next.appInfo,
                    next.fullscreen);
        }
------------------------------------------------------------------------------
对应Log：
04-16 13:08:36.769 V/ActivityStack_Switch( 1049): Resuming ActivityRecord{65cc87a u0 com.android.testred/.TestActivity t5}
04-16 13:08:36.769 D/ActivityTrigger( 1049): activityResumeTrigger: Activity is not Triggerred in full screen ApplicationInfo{2bedb5d com.android.testred}
------------------------------------------------------------------------------
        ......

        mStackSupervisor.setLaunchSource(next.info.applicationInfo.uid);

        boolean lastResumedCanPip = false;
        ActivityRecord lastResumed = null;
        final ActivityStack lastFocusedStack = mStackSupervisor.getLastStack();
        ......
        final boolean resumeWhilePausing = (next.info.flags & FLAG_RESUME_WHILE_PAUSING) != 0
                && !lastResumedCanPip;

        boolean pausing = mStackSupervisor.pauseBackStacks(userLeaving, next, false);
        if (mResumedActivity != null) {
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: Pausing " + mResumedActivity);
            pausing |= startPausingLocked(userLeaving, false, next, false);
        }
        .......

        ActivityStack lastStack = mStackSupervisor.getLastStack();
        if (next.app != null && next.app.thread != null) {
            if (DEBUG_SWITCH) Slog.v(TAG_SWITCH, "Resume running: " + next
                    + " stopped=" + next.stopped + " visible=" + next.visible);

            // If the previous activity is translucent, force a visibility update of
            // the next activity, so that it's added to WM's opening app list, and
            // transition animation can be set up properly.
            // For example, pressing Home button with a translucent activity in focus.
            // Launcher is already visible in this case. If we don't add it to opening
            // apps, maybeUpdateTransitToWallpaper() will fail to identify this as a
            // TRANSIT_WALLPAPER_OPEN animation, and run some funny animation.
            final boolean lastActivityTranslucent = lastStack != null
                    && (lastStack.inMultiWindowMode()
                    || (lastStack.mLastPausedActivity != null
                    && !lastStack.mLastPausedActivity.fullscreen));

            // The contained logic must be synchronized, since we are both changing the visibility
            // and updating the {@link Configuration}. {@link ActivityRecord#setVisibility} will
            // ultimately cause the client code to schedule a layout. Since layouts retrieve the
            // current {@link Configuration}, we must ensure that the below code updates it before
            // the layout can occur.
            synchronized(mWindowManager.getWindowManagerLock()) {
                // This activity is now becoming visible.
                if (!next.visible || next.stopped || lastActivityTranslucent) {
                    next.setVisibility(true);
                }

                // schedule launch ticks to collect information about slow apps.
                next.startLaunchTickingLocked();

                ActivityRecord lastResumedActivity =
                        lastStack == null ? null :lastStack.mResumedActivity;
                final ActivityState lastState = next.getState();

                mService.updateCpuStats();

                if (DEBUG_STATES) Slog.v(TAG_STATES, "Moving to RESUMED: " + next
                        + " (in existing)");

                next.setState(RESUMED, "resumeTopActivityInnerLocked");

                mService.updateLruProcessLocked(next.app, true, null);
                updateLRUListLocked(next);
                mService.updateOomAdjLocked();

                // Have the window manager re-evaluate the orientation of
                // the screen based on the new activity order.
                boolean notUpdated = true;

                if (mStackSupervisor.isFocusedStack(this)) {
                    // We have special rotation behavior when here is some active activity that
                    // requests specific orientation or Keyguard is locked. Make sure all activity
                    // visibilities are set correctly as well as the transition is updated if needed
                    // to get the correct rotation behavior. Otherwise the following call to update
                    // the orientation may cause incorrect configurations delivered to client as a
                    // result of invisible window resize.
                    // TODO: Remove this once visibilities are set correctly immediately when
                    // starting an activity.
                    notUpdated = !mStackSupervisor.ensureVisibilityAndConfig(next, mDisplayId,
                            true /* markFrozenIfConfigChanged */, false /* deferResume */);
                }

                if (notUpdated) {
                    // The configuration update wasn't able to keep the existing
                    // instance of the activity, and instead started a new one.
                    // We should be all done, but let's just make sure our activity
                    // is still at the top and schedule another run if something
                    // weird happened.
                    ActivityRecord nextNext = topRunningActivityLocked();
                    if (DEBUG_SWITCH || DEBUG_STATES) Slog.i(TAG_STATES,
                            "Activity config changed during resume: " + next
                                    + ", new next: " + nextNext);
                    if (nextNext != next) {
                        // Do over!
                        mStackSupervisor.scheduleResumeTopActivities();
                    }
                    if (!next.visible || next.stopped) {
                        next.setVisibility(true);
                    }
                    next.completeResumeLocked();
                    if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
                    return true;
                }

                try {
                    final ClientTransaction transaction = ClientTransaction.obtain(next.app.thread,
                            next.appToken);
                    // Deliver all pending results.
                    ArrayList<ResultInfo> a = next.results;
                    if (a != null) {
                        final int N = a.size();
                        if (!next.finishing && N > 0) {
                            if (DEBUG_RESULTS) Slog.v(TAG_RESULTS,
                                    "Delivering results to " + next + ": " + a);
                            transaction.addCallback(ActivityResultItem.obtain(a));
                        }
                    }

                    if (next.newIntents != null) {
                        transaction.addCallback(NewIntentItem.obtain(next.newIntents,
                                false /* andPause */));
                    }

                    // Well the app will no longer be stopped.
                    // Clear app token stopped state in window manager if needed.
                    next.notifyAppResumed(next.stopped);

                    EventLog.writeEvent(EventLogTags.AM_RESUME_ACTIVITY, next.userId,
                            System.identityHashCode(next), next.getTask().taskId,
                            next.shortComponentName);

                    next.sleeping = false;
                    mService.getAppWarningsLocked().onResumeActivity(next);
                    mService.showAskCompatModeDialogLocked(next);
                    next.app.pendingUiClean = true;
                    next.app.forceProcessStateUpTo(mService.mTopProcessState);
                    next.clearOptionsLocked();
                    transaction.setLifecycleStateRequest(
                            ResumeActivityItem.obtain(next.app.repProcState,
                                    mService.isNextTransitionForward()));
                    mService.getLifecycleManager().scheduleTransaction(transaction);

                    if (DEBUG_STATES) Slog.d(TAG_STATES, "resumeTopActivityLocked: Resumed "
                            + next);
                } catch (Exception e) {
                    // Whoops, need to restart this activity!
                    if (DEBUG_STATES) Slog.v(TAG_STATES, "Resume failed; resetting state to "
                            + lastState + ": " + next);
                    next.setState(lastState, "resumeTopActivityInnerLocked");

                    // lastResumedActivity being non-null implies there is a lastStack present.
                    if (lastResumedActivity != null) {
                        lastResumedActivity.setState(RESUMED, "resumeTopActivityInnerLocked");
                    }

                    Slog.i(TAG, "Restarting because process died: " + next);
                    if (!next.hasBeenLaunched) {
                        next.hasBeenLaunched = true;
                    } else  if (SHOW_APP_STARTING_PREVIEW && lastStack != null
                            && getDisplay() != null && lastStack.isTopStackOnDisplay()) {
                        next.showStartingWindow(null /* prev */, false /* newTask */,
                                false /* taskSwitch */);
                    }
                    mStackSupervisor.startSpecificActivityLocked(next, true, false);
                    if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
                    return true;
                }
            }

            // From this point on, if something goes wrong there is no way
            // to recover the activity.
            try {
                next.completeResumeLocked();
            } catch (Exception e) {
                // If any exception gets thrown, toss away this
                // activity and try the next one.
                Slog.w(TAG, "Exception thrown during resume of " + next, e);
                requestFinishActivityLocked(next.appToken, Activity.RESULT_CANCELED, null,
                        "resume-exception", true);
                if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
                return true;
            }
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
#### 四、执行栈顶Activity的onPause方法

##### 4.1、mStackSupervisor.pauseBackStacks(userLeaving, next, false)
首先将Launcher 的ActivityStack 暂停。
``` java
G:\android9.0\frameworks\base\services\core\java\com\android\server\am\ActivityStackSupervisor.java

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

--------------------------------------------------------------------------------------------------
对应Log：
04-16 13:08:36.769 D/ActivityStackSupervisor_States( 1049): pauseBackStacks: stack=ActivityStack{6db92ec stackId=0 type=home mode=fullscreen visible=true translucent=false, 1 tasks} mResumedActivity=ActivityRecord{5e34200 u0 com.android.launcher3/.Launcher t4}

--------------------------------------------------------------------------------------------------
```

##### 4.2、ActivityStack.startPausingLocked()

``` java
G:\android9.0\frameworks\base\services\core\java\com\android\server\am\ActivityStack.java


    final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping,
            ActivityRecord resuming, boolean pauseImmediately) {
        .......
        ActivityRecord prev = mResumedActivity;

        .......
        if (DEBUG_STATES) Slog.v(TAG_STATES, "Moving to PAUSING: " + prev);
        else if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Start pausing: " + prev);

        if (mActivityTrigger != null) {
            mActivityTrigger.activityPauseTrigger(prev.intent, prev.info, prev.appInfo);
        }

       .......
--------------------------------------------------------------------------------------------------
对应Log：
04-16 13:08:36.769 D/ActivityStackSupervisor_States( 1049): pauseBackStacks: stack=ActivityStack{6db92ec stackId=0 type=home mode=fullscreen visible=true translucent=false, 1 tasks} mResumedActivity=ActivityRecord{5e34200 u0 com.android.launcher3/.Launcher t4}

--------------------------------------------------------------------------------------------------
        mResumedActivity = null;
        mPausingActivity = prev;
        mLastPausedActivity = prev;
        mLastNoHistoryActivity = (prev.intent.getFlags() & Intent.FLAG_ACTIVITY_NO_HISTORY) != 0
                || (prev.info.flags & ActivityInfo.FLAG_NO_HISTORY) != 0 ? prev : null;
-----------------------------------------------------------------------------------------------
对应Log：
04-16 13:08:36.769 V/ActivityRecord_States( 1049): State movement: ActivityRecord{5e34200 u0 com.android.launcher3/.Launcher t4} from:RESUMED to:PAUSING reason:startPausingLocked
-----------------------------------------------------------------------------------------------
        prev.setState(PAUSING, "startPausingLocked");


        prev.getTask().touchActiveTime();
        clearLaunchTime(prev);

        mStackSupervisor.getLaunchTimeTracker().stopFullyDrawnTraceIfNeeded(getWindowingMode());

        mService.updateCpuStats();

        if (prev.app != null && prev.app.thread != null) {
--------------------------------------------------------------------------------------------------
对应Log：
04-16 13:08:36.770 V/ActivityStack_Pause( 1049): Enqueueing pending pause: ActivityRecord{5e34200 u0 com.android.launcher3/.Launcher t4}
--------------------------------------------------------------------------------------------------
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
--------------------------------------------------------------------------------------------------
对应Log：
04-16 13:08:36.770 V/WindowManager( 1049): Pausing WindowToken AppWindowToken{483acfb token=Token{f984e8a ActivityRecord{5e34200 u0 com.android.launcher3/.Launcher t4}}}
 prev.pauseKeyDispatchingLocked();
--------------------------------------------------------------------------------------------------
            if (!uiSleeping) {
                prev.pauseKeyDispatchingLocked();
            } else if (DEBUG_PAUSE) {
                 Slog.v(TAG_PAUSE, "Key dispatch not paused for screen off");
            }

            if (pauseImmediately) {
                // If the caller said they don't want to wait for the pause, then complete
                // the pause now.
                //完成Pause
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


```
看下ActivityRecord的setState 方法。

```java
G:\android9.0\frameworks\base\services\core\java\com\android\server\am\ActivityRecord.java

    void setState(ActivityState state, String reason) {
        if (DEBUG_STATES) Slog.v(TAG_STATES, "State movement: " + this + " from:" + getState()
                        + " to:" + state + " reason:" + reason);
        ......
        mState = state;

        final TaskRecord parent = getTask();

        if (parent != null) {
            parent.onActivityStateChanged(this, state, reason);
        }

        // The WindowManager interprets the app stopping signal as
        // an indication that the Surface will eventually be destroyed.
        // This however isn't necessarily true if we are going to sleep.
        if (state == STOPPING && !isSleeping()) {
            mWindowContainerController.notifyAppStopping();
        }
    }
```
接着看下mService.updateUsageStats(prev, false);

```java
对应Log：
04-16 13:08:36.770 D/ActivityManagerService_Switch( 1049): updateUsageStats: comp=ActivityRecord{5e34200 u0 com.android.launcher3/.Launcher t4}res=false

G:\android9.0\frameworks\base\services\core\java\com\android\server\am\ActivityManagerService.java

    void updateUsageStats(ActivityRecord component, boolean resumed) {
        if (DEBUG_SWITCH) Slog.d(TAG_SWITCH,
                "updateUsageStats: comp=" + component + "res=" + resumed);
        final BatteryStatsImpl stats = mBatteryStatsService.getActiveStatistics();
        StatsLog.write(StatsLog.ACTIVITY_FOREGROUND_STATE_CHANGED,
            component.app.uid, component.realActivity.getPackageName(),
            component.realActivity.getShortClassName(), resumed ?
                        StatsLog.ACTIVITY_FOREGROUND_STATE_CHANGED__STATE__FOREGROUND :
                        StatsLog.ACTIVITY_FOREGROUND_STATE_CHANGED__STATE__BACKGROUND);
        if (resumed) {
            if (mUsageStatsService != null) {
                mUsageStatsService.reportEvent(component.realActivity, component.userId,
                        UsageEvents.Event.MOVE_TO_FOREGROUND);

            }
            synchronized (stats) {
                stats.noteActivityResumedLocked(component.app.uid);
            }
        } else {
            if (mUsageStatsService != null) {
                mUsageStatsService.reportEvent(component.realActivity, component.userId,
                        UsageEvents.Event.MOVE_TO_BACKGROUND);
            }
            synchronized (stats) {
                stats.noteActivityPausedLocked(component.app.uid);
            }
        }
    }
```

##### 4.3、scheduleTransaction(prev.app.thread, prev.appToken,PauseActivityItem.obtain(......));
``` java  
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

Android 9.0已改变没有直接提供Java源码，而是.class文件打包为classes-header.jar和classes.jar。
android\out\target\common\obj\JAVA_LIBRARIES\framework_intermediates\
classes-header.jar
classes.jar解压：
classes/android/app/
    IApplicationThread.class
    IApplicationThread$Stub.class
    IApplicationThread$Stub$Proxy.class
    
解压后在线反编译http://www.javadecompilers.com/
------------------------------------------------------------------------------------------------
IApplicationThread.class
------------------------------------------------------------------------------------------------
package android.app;

.......

public abstract interface IApplicationThread
  extends IInterface
{
  public abstract void scheduleReceiver(Intent paramIntent, ActivityInfo paramActivityInfo, CompatibilityInfo paramCompatibilityInfo, int paramInt1, String paramString, Bundle paramBundle, boolean paramBoolean, int paramInt2, int paramInt3)
    throws RemoteException;
  
  ......
  
  public abstract void scheduleTransaction(ClientTransaction paramClientTransaction)
    throws RemoteException;
}
------------------------------------------------------------------------------------------------
IApplicationThread$Stub.class
------------------------------------------------------------------------------------------------
package android.app;

import android.os.Parcel;

public abstract class IApplicationThread$Stub extends android.os.Binder
  implements IApplicationThread
{
  private static final String DESCRIPTOR = "android.app.IApplicationThread";
  static final int TRANSACTION_scheduleReceiver = 1;
  static final int TRANSACTION_scheduleCreateService = 2;
  static final int TRANSACTION_scheduleStopService = 3;
  static final int TRANSACTION_bindApplication = 4;
  static final int TRANSACTION_runIsolatedEntryPoint = 5;
  static final int TRANSACTION_scheduleExit = 6;
  static final int TRANSACTION_scheduleServiceArgs = 7;
  static final int TRANSACTION_updateTimeZone = 8;
  static final int TRANSACTION_processInBackground = 9;
  static final int TRANSACTION_scheduleBindService = 10;
  static final int TRANSACTION_scheduleUnbindService = 11;
  static final int TRANSACTION_dumpService = 12;
  
  public IApplicationThread$Stub() { attachInterface(this, "android.app.IApplicationThread"); }
  
  static final int TRANSACTION_scheduleRegisteredReceiver = 13;
  static final int TRANSACTION_scheduleLowMemory = 14;
  static final int TRANSACTION_scheduleSleeping = 15;
  static final int TRANSACTION_profilerControl = 16;
  
  public static IApplicationThread asInterface(android.os.IBinder obj) {
    if (obj == null) {
      return null;
    }
    android.os.IInterface iin = obj.queryLocalInterface("android.app.IApplicationThread");
    if ((iin != null) && ((iin instanceof IApplicationThread))) {
      return (IApplicationThread)iin;
    }
    return new IApplicationThread.Stub.Proxy(obj); }
  
  static final int TRANSACTION_setSchedulingGroup = 17;
  
  public android.os.IBinder asBinder() { return this; }
  
  static final int TRANSACTION_scheduleCreateBackupAgent = 18;
  
  public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws android.os.RemoteException { String descriptor = "android.app.IApplicationThread";
    switch (code)
    {

    case 1598968902: 
      reply.writeString(descriptor);
      return true;
    ......
    case 51: 
      data.enforceInterface(descriptor);
      android.app.servertransaction.ClientTransaction _arg0;
      android.app.servertransaction.ClientTransaction _arg0; if (0 != data.readInt()) {
        _arg0 = (android.app.servertransaction.ClientTransaction)android.app.servertransaction.ClientTransaction.CREATOR.createFromParcel(data);
      }
      else {
        _arg0 = null;
      }
      scheduleTransaction(_arg0);
      return true;
    }
    
    
    return super.onTransact(code, data, reply, flags);
  }
  
  ......
  static final int TRANSACTION_scheduleTransaction = 51;
}
------------------------------------------------------------------------------------------------
IApplicationThread$Stub$Proxy.class
------------------------------------------------------------------------------------------------
package android.app;
......

class IApplicationThread$Stub$Proxy
  implements IApplicationThread
{
  private IBinder mRemote;
  
  IApplicationThread$Stub$Proxy(IBinder remote)
  {
    mRemote = remote;
  }
  
  public IBinder asBinder() {
    return mRemote;
  }
  
  public String getInterfaceDescriptor() {
    return "android.app.IApplicationThread";
  }
  
  ......
  
  public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
    Parcel _data = Parcel.obtain();
    try {
      _data.writeInterfaceToken("android.app.IApplicationThread");
      if (transaction != null) {
        _data.writeInt(1);
        transaction.writeToParcel(_data, 0);
      }
      else {
        _data.writeInt(0);
      }
      mRemote.transact(51, _data, null, 1);
      

      _data.recycle(); } finally { _data.recycle();
    }
  }
}
```

继续看ActivityThread.this.scheduleTransaction(transaction);源码:

注意一下,这里由于ActivityThread中没有重写scheduleTransaction,所以会走到ActivityThread的父类ClientTransactionHandler的scheduleTransaction

``` cpp
G:\android9.0\frameworks\base\core\java\android\app\ClientTransactionHandler.java

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

``` cpp

G:\android9.0\frameworks\base\core\java\android\app\ActivityThread.java
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

mTransactionExecutor.execute(transaction)源码


``` cpp
G:\android9.0\frameworks\base\core\java\android\app\servertransaction\TransactionExecutor.java

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

##### 4.4、PauseActivityItem.execute()

G:\android9.0\frameworks\base\core\java\android\app\servertransaction\TransactionExecutor.java


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

##### 4.5、[ActivityThread.handleMessage() : mH]

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
##### 4.6、ActivityThread.handlePauseActivity() 

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
##### 4.7、ActivityThread.performPauseActivity() 

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
##### 4.8、Instrumentation.callActivityOnPuase() 

``` java
    [->Instrumentation.java]
public void callActivityOnPause(Activity activity) {
    activity.performPause();
}
```

原来最终回调到了Activity的performPause方法：
##### 4.9、Activity.performPause() 

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

终于，太不容易了，回调到了Activity的onPause方法，哈哈，Activity生命周期中的第一个生命周期方法终于被我们找到了。。。也就是说我们在启动一个Activity的时候最先被执行的是栈顶的Activity的onPause方法。

> ActivityStackSupervisor.resumeFocusedStackTopActivityLocked()
> –> ActivityStack.resumeTopActivityUncheckedLocked() 
> –> ActivityStack.resumeTopActivityInnerLocked()
> –> ActivityStackSupervisor.startSpecificActivityLocked()
继续分析resumeTopActivityInnerLocked

```java
G:\android9.0\frameworks\base\services\core\java\com\android\server\am\ActivityStack.java

    private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
    ......
    if (next.app != null && next.app.thread != null) {
    else {
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
-----------------------------------------------------------------------
对应Log：
04-16 13:08:36.787 D/ActivityStack_States( 1049): resumeTopActivityLocked: Restarting ActivityRecord{65cc87a u0 com.android.testred/.TestActivity t5}
-----------------------------------------------------------------------
         if (DEBUG_STATES) Slog.d(TAG_STATES, "resumeTopActivityLocked: Restarting " + next);
            mStackSupervisor.startSpecificActivityLocked(next, true, true);
        }

```
由于第一Activity的 ProcessRecord app 肯定为null，所以会走else分支;我们看一下startSpecificActivityLocked的具体实现：

``` java
G:\android9.0\frameworks\base\services\core\java\com\android\server\am\ActivityStackSupervisor.java

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
#### 五、 App（"com.android.testred"）进程启动过程
接下来分析一下应用进程的启动过程，上面分析到ActivityStackSupervisor.startSpecificActivityLocked方法，在这个方法中会去根据进程和线程是否存在判断App是否已经启动，如果已经启动，就会调用realStartActivityLocked方法继续处理。如果没有启动则调用ActivityManagerService.startProcessLocked方法创建新的进程处理。
**准备知识**
本文要介绍的是进程的创建，先简单说说进程与线程的区别。

- 进程：每个App在启动前必须先创建一个进程，该进程是由Zygote fork出来的，进程具有独立的资源空间，用于承载App上运行的各种Activity/Service等组件。进程对于上层应用来说是完全透明的，这也是google有意为之，让App程序都是运行在Android Runtime。大多数情况一个App就运行在一个进程中，除非在AndroidManifest.xml中配置Android:process属性，或通过native代码fork进程。

- 线程：线程对应用开发者来说非常熟悉，比如每次new Thread().start()都会创建一个新的线程，该线程并没有自己独立的地址空间，而是与其所在进程之间资源共享。从Linux角度来说进程与线程都是一个task_struct结构体，除了是否共享资源外，并没有其他本质的区别。

在接下来的文章，会涉及到system_server进程和Zygote进程，下面简要这两个进程：

system_server进程：是用于管理整个Java framework层，包含ActivityManager，PowerManager等各种系统服务;
Zygote进程：是Android系统的首个Java进程，Zygote是所有Java进程的父进程，包括 system_server进程以及所有的App进程都是Zygote的子进程，注意这里说的是子进程，而非子线程。

![enter image description here](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/personal.website/zjj.display.sys.Process.start.png)


图解：

- App发起进程：当从桌面启动应用，则发起进程便是Launcher所在进程；当从某App内启动远程进程，则发送进程便是该App所在进程。发起进程先通过binder发送消息给system_server进程；
system_server进程：调用Process.start()方法，通过socket向zygote进程发送创建新进程的请求；
- zygote进程：在执行ZygoteInit.main()后ZygoteServer便进入runSelectLoop()循环体内，当有客户端连接时便会执行ZygoteConnection.processOneCommand()方法，再经过层层调用后fork出新的应用进程；
- 新进程：执行handleChildProc方法，最后调用ActivityThread.main()方法。
接下来，依次从system_server进程发起请求到Zygote创建进程，再到新进程的运行这3大块展开讲解进程创建是一个怎样的过程。
##### 5.1、system_server发起请求
##### 5.1.1、 ActivityManagerService.startProcessLocked()

``` java
G:\android9.0\frameworks\base\services\core\java\com\android\server\am\ActivityManagerService.java

    @GuardedBy("this")
    final ProcessRecord startProcessLocked(String processName,
            ApplicationInfo info, boolean knownToBeDead, int intentFlags,
            String hostingType, ComponentName hostingName, boolean allowWhileBooting,
            boolean isolated, boolean keepIfLarge) {
        return startProcessLocked(processName, info, knownToBeDead, intentFlags, hostingType,
                hostingName, allowWhileBooting, isolated, 0 /* isolatedUid */, keepIfLarge,
                null /* ABI override */, null /* entryPoint */, null /* entryPointArgs */,
                null /* crashHandler */);
    }

    @GuardedBy("this")
    final ProcessRecord startProcessLocked(String processName, ApplicationInfo info,
            boolean knownToBeDead, int intentFlags, String hostingType, ComponentName hostingName,
            boolean allowWhileBooting, boolean isolated, int isolatedUid, boolean keepIfLarge,
            String abiOverride, String entryPoint, String[] entryPointArgs, Runnable crashHandler) {
        long startTime = SystemClock.elapsedRealtime();
        ProcessRecord app;
        if (!isolated) {
            app = getProcessRecordLocked(processName, info.uid, keepIfLarge);
            checkTime(startTime, "startProcess: after getProcessRecord");

            if ((intentFlags & Intent.FLAG_FROM_BACKGROUND) != 0) {
                ......
            } else {
                if (DEBUG_PROCESSES) Slog.v(TAG, "Clearing bad process: " + info.uid
                        + "/" + info.processName);
                mAppErrors.resetProcessCrashTimeLocked(info);
                if (mAppErrors.isBadProcessLocked(info)) {
                    EventLog.writeEvent(EventLogTags.AM_PROC_GOOD,
                            UserHandle.getUserId(info.uid), info.uid,
                            info.processName);
                    mAppErrors.clearBadProcessLocked(info);
                    if (app != null) {
                        app.bad = false;
                    }
                }
            }
        } else {
            // If this is an isolated process, it can't re-use an existing process.
            app = null;
        }

        // We don't have to do anything more if:
        // (1) There is an existing application record; and
        // (2) The caller doesn't think it is dead, OR there is no thread
        //     object attached to it so we know it couldn't have crashed; and
        // (3) There is a pid assigned to it, so it is either starting or
        //     already running.
        if (DEBUG_PROCESSES) Slog.v(TAG_PROCESSES, "startProcess: name=" + processName
                + " app=" + app + " knownToBeDead=" + knownToBeDead
                + " thread=" + (app != null ? app.thread : null)
                + " pid=" + (app != null ? app.pid : -1));
        .......
-------------------------------------------------------------------------------
	Line 1222: 04-16 13:08:36.788 V/ActivityManagerService( 1049): Clearing bad process: 10088/com.android.testred
	Line 1223: 04-16 13:08:36.788 V/ActivityManagerService_Processes( 1049): startProcess: name=com.android.testred app=null knownToBeDead=true thread=null pid=-1
-------------------------------------------------------------------------------
        String hostingNameStr = hostingName != null
                ? hostingName.flattenToShortString() : null;

        if (app == null) {
            checkTime(startTime, "startProcess: creating new process record");
            app = newProcessRecordLocked(info, processName, isolated, isolatedUid);
            ......
            app.crashHandler = crashHandler;
            app.isolatedEntryPoint = entryPoint;
            app.isolatedEntryPointArgs = entryPointArgs;
            checkTime(startTime, "startProcess: done creating new process record");
        } else {
            // If this is a new package in the process, add the package to the list
            app.addPackage(info.packageName, info.versionCode, mProcessStats);
            checkTime(startTime, "startProcess: added package to existing proc");
        }

        ......
        final boolean success = startProcessLocked(app, hostingType, hostingNameStr, abiOverride);
        ......
        return success ? app : null;
    }
```
##### 5.1.2、 ActivityManagerService.newProcessRecordLocked()
创建ProcessRecord
``` java
G:\android9.0\frameworks\base\services\core\java\com\android\server\am\ActivityManagerService.java

newProcessRecordLocked(ApplicationInfo info, String customProcess,
            boolean isolated, int isolatedUid) {
        String proc = customProcess != null ? customProcess : info.processName;
        BatteryStatsImpl stats = mBatteryStatsService.getActiveStatistics();
        final int userId = UserHandle.getUserId(info.uid);
        int uid = info.uid;
        ......
        final ProcessRecord r = new ProcessRecord(this, stats, info, proc, uid);
        ......
        addProcessNameLocked(r);
        return r;
    }
```

##### 5.1.3、 ActivityManagerService.startProcessLocked()


``` java

G:\android9.0\frameworks\base\services\core\java\com\android\server\am\ActivityManagerService.java

            // Start the process.  It will either succeed and return a result containing
            // the PID of the new process, or else throw a RuntimeException.
            final String entryPoint = "android.app.ActivityThread";

            return startProcessLocked(hostingType, hostingNameStr, entryPoint, app, uid, gids,
                    runtimeFlags, mountExternal, seInfo, requiredAbi, instructionSet, invokeWith,
                    startTime);

    @GuardedBy("this")
    private boolean startProcessLocked(String hostingType, String hostingNameStr, String entryPoint,
            ProcessRecord app, int uid, int[] gids, int runtimeFlags, int mountExternal,
            String seInfo, String requiredAbi, String instructionSet, String invokeWith,
            long startTime) {
        app.pendingStart = true;
        app.killedByAm = false;
        app.removed = false;
        app.killed = false;
        final long startSeq = app.startSeq = ++mProcStartSeqCounter;
        app.setStartParams(uid, hostingType, hostingNameStr, seInfo, startTime);
        if (mConstants.FLAG_PROCESS_START_ASYNC) {
-------------------------------------------------------------------------------
	Line 1225: 04-16 13:08:36.788 I/ActivityManagerService_Processes( 1049): Posting procStart msg for 0:com.android.testred/u0a88
-------------------------------------------------------------------------------
            if (DEBUG_PROCESSES) Slog.i(TAG_PROCESSES,
                    "Posting procStart msg for " + app.toShortString());
            mProcStartHandler.post(() -> {
                try {
                    .......
                    final ProcessStartResult startResult = startProcess(app.hostingType, entryPoint,
                            app, app.startUid, gids, runtimeFlags, mountExternal, app.seInfo,
                            requiredAbi, instructionSet, invokeWith, app.startTime);
                    synchronized (ActivityManagerService.this) {
                        handleProcessStartedLocked(app, startResult, startSeq);
                    }
                } catch (RuntimeException e) {
                    synchronized (ActivityManagerService.this) {
                        Slog.e(TAG, "Failure starting process " + app.processName, e);
                        mPendingStarts.remove(startSeq);
                        app.pendingStart = false;
                        forceStopPackageLocked(app.info.packageName, UserHandle.getAppId(app.uid),
                                false, false, true, false, false,
                                UserHandle.getUserId(app.userId), "start failure");
                    }
                }
            });
            return true;
        } else {
            try {
                final ProcessStartResult startResult = startProcess(hostingType, entryPoint, app,
                        uid, gids, runtimeFlags, mountExternal, seInfo, requiredAbi, instructionSet,
                        invokeWith, startTime);
                handleProcessStartedLocked(app, startResult.pid, startResult.usingWrapper,
                        startSeq, false);
            } catch (RuntimeException e) {
                Slog.e(TAG, "Failure starting process " + app.processName, e);
                app.pendingStart = false;
                forceStopPackageLocked(app.info.packageName, UserHandle.getAppId(app.uid),
                        false, false, true, false, false,
                        UserHandle.getUserId(app.userId), "start failure");
            }
            return app.pid > 0;
        }
    }
```

##### 5.1.4、 ActivityManagerService.startProcess()

``` java

G:\android9.0\frameworks\base\services\core\java\com\android\server\am\ActivityManagerService.java

    private ProcessStartResult startProcess(String hostingType, String entryPoint,
            ProcessRecord app, int uid, int[] gids, int runtimeFlags, int mountExternal,
            String seInfo, String requiredAbi, String instructionSet, String invokeWith,
            long startTime) {
        try {
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "Start proc: " +
                    app.processName);
            checkTime(startTime, "startProcess: asking zygote to start proc");
            final ProcessStartResult startResult;
            if (hostingType.equals("webview_service")) {
                startResult = startWebView(entryPoint,
                        app.processName, uid, uid, gids, runtimeFlags, mountExternal,
                        app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                        app.info.dataDir, null,
                        new String[] {PROC_START_SEQ_IDENT + app.startSeq});
            }
            else {
                startResult = Process.start(entryPoint,
                        app.processName, uid, uid, gids, runtimeFlags, mountExternal,
                        app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                        app.info.dataDir, invokeWith,
                        new String[] {PROC_START_SEQ_IDENT + app.startSeq});
            }

            if (mPerfServiceStartHint == null) {
                mPerfServiceStartHint = new BoostFramework();
            }
            if (mPerfServiceStartHint != null) {
                mPerfServiceStartHint.perfHint(BoostFramework.VENDOR_HINT_FIRST_LAUNCH_BOOST, app.processName, -1, BoostFramework.Launch.TYPE_SERVICE_START);
            }

            checkTime(startTime, "startProcess: returned from zygote!");
            return startResult;
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        }
    }
```
##### 5.1.5、Process.start()

``` java
G:\android9.0\frameworks\base\core\java\android\os\Process.java
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
```
##### 5.1.6、zygoteProcess.start()

``` java

G:\android9.0\frameworks\base\core\java\android\os\ZygoteProcess.java
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
        ArrayList<String> argsForZygote = new ArrayList<String>();

        // --runtime-args, --setuid=, --setgid=,
        // and --setgroups= must go first
        argsForZygote.add("--runtime-args");
        argsForZygote.add("--setuid=" + uid);
        argsForZygote.add("--setgid=" + gid);
        argsForZygote.add("--runtime-flags=" + runtimeFlags);
        if (mountExternal == Zygote.MOUNT_EXTERNAL_DEFAULT) {
            argsForZygote.add("--mount-external-default");
        } else if (mountExternal == Zygote.MOUNT_EXTERNAL_READ) {
            argsForZygote.add("--mount-external-read");
        } else if (mountExternal == Zygote.MOUNT_EXTERNAL_WRITE) {
            argsForZygote.add("--mount-external-write");
        }
        argsForZygote.add("--target-sdk-version=" + targetSdkVersion);

        // --setgroups is a comma-separated list
        if (gids != null && gids.length > 0) {
            StringBuilder sb = new StringBuilder();
            sb.append("--setgroups=");

            int sz = gids.length;
            for (int i = 0; i < sz; i++) {
                if (i != 0) {
                    sb.append(',');
                }
                sb.append(gids[i]);
            }

            argsForZygote.add(sb.toString());
        }

        if (niceName != null) {
            argsForZygote.add("--nice-name=" + niceName);
        }

        if (seInfo != null) {
            argsForZygote.add("--seinfo=" + seInfo);
        }

        if (instructionSet != null) {
            argsForZygote.add("--instruction-set=" + instructionSet);
        }

        if (appDataDir != null) {
            argsForZygote.add("--app-data-dir=" + appDataDir);
        }

        if (invokeWith != null) {
            argsForZygote.add("--invoke-with");
            argsForZygote.add(invokeWith);
        }

        if (startChildZygote) {
            argsForZygote.add("--start-child-zygote");
        }

        argsForZygote.add(processClass);

        if (extraArgs != null) {
            for (String arg : extraArgs) {
                argsForZygote.add(arg);
            }
        }

        synchronized(mLock) {
            return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
        }
    }
```
该过程主要工作是生成argsForZygote数组，该数组保存了进程的uid、gid、groups、target-sdk、nice-name等一系列的参数。
##### 5.1.7、zygoteProcess.openZygoteSocketIfNeeded()

``` java
G:\android9.0\frameworks\base\core\java\android\os\ZygoteProcess.java
    @GuardedBy("mLock")
    private ZygoteState openZygoteSocketIfNeeded(String abi) throws ZygoteStartFailedEx {
        Preconditions.checkState(Thread.holdsLock(mLock), "ZygoteProcess lock not held");

        if (primaryZygoteState == null || primaryZygoteState.isClosed()) {
            try {
                //向主zygote发起connect()操作
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
                //当主zygote没能匹配成功，则采用第二个zygote，发起connect()操作
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


openZygoteSocketIfNeeded(abi)方法是根据当前的abi来选择与zygote还是zygote64来进行通信。

既然system_server进程的zygoteSendArgsAndGetResult()方法通过socket向Zygote进程发送消息，这是便会唤醒Zygote进程，来响应socket客户端的请求（即system_server端），接下来的操作便是在Zygote来创建进程


##### 5.1.8、zygoteSendArgsAndGetResult

``` java
G:\android9.0\frameworks\base\core\java\android\os\ZygoteProcess.java

private static ProcessStartResult zygoteSendArgsAndGetResult( ZygoteState zygoteState, ArrayList<String> args) throws ZygoteStartFailedEx {
    try {
        final BufferedWriter writer = zygoteState.writer;
        final DataInputStream inputStream = zygoteState.inputStream;

        writer.write(Integer.toString(args.size()));
        writer.newLine();

        int sz = args.size();
        for (int i = 0; i < sz; i++) {
            String arg = args.get(i);
            if (arg.indexOf('\n') >= 0) {
                throw new ZygoteStartFailedEx(
                        "embedded newlines not allowed");
            }
            writer.write(arg);
            writer.newLine();
        }

        writer.flush();

        ProcessStartResult result = new ProcessStartResult();
        //等待socket服务端（即zygote）返回新创建的进程pid;
        //对于等待时长问题，Google正在考虑此处是否应该有一个timeout，但目前是没有的。
        result.pid = inputStream.readInt();
        if (result.pid < 0) {
            throw new ZygoteStartFailedEx("fork() failed");
        }
        result.usingWrapper = inputStream.readBoolean();
        return result;
    } catch (IOException ex) {
        zygoteState.close();
        throw new ZygoteStartFailedEx(ex);
    }
}
```
这个方法的主要功能是通过socket通道向Zygote进程发送一个参数列表，然后进入阻塞等待状态，直到远端的socket服务端发送回来新创建的进程pid才返回。

##### 5.2、Zygote创建进程
简单来说就是Zygote进程是由由init进程而创建的，进程启动之后调用ZygoteInit.main()方法，经过创建socket管道，预加载资源后，便进程runSelectLoop()方法。

``` java
G:\android9.0\frameworks\base\core\java\com\android\internal\os\ZygoteInit.java

    public static void main(String argv[]) {
        ZygoteServer zygoteServer = new ZygoteServer();

        // Mark zygote start. This ensures that thread creation will throw
        // an error.
        ZygoteHooks.startZygoteNoThreadCreation();

        // Zygote goes into its own process group.
        try {
            Os.setpgid(0, 0);
        } catch (ErrnoException ex) {
            throw new RuntimeException("Failed to setpgid(0,0)", ex);
        }

        final Runnable caller;
        try {
            // Report Zygote start time to tron unless it is a runtime restart
            if (!"1".equals(SystemProperties.get("sys.boot_completed"))) {
                MetricsLogger.histogram(null, "boot_zygote_init",
                        (int) SystemClock.elapsedRealtime());
            }

            String bootTimeTag = Process.is64Bit() ? "Zygote64Timing" : "Zygote32Timing";
            TimingsTraceLog bootTimingsTraceLog = new TimingsTraceLog(bootTimeTag,
                    Trace.TRACE_TAG_DALVIK);
            bootTimingsTraceLog.traceBegin("ZygoteInit");
            RuntimeInit.enableDdms();

            boolean startSystemServer = false;
            String socketName = "zygote";
            String abiList = null;
            boolean enableLazyPreload = false;
            for (int i = 1; i < argv.length; i++) {
                if ("start-system-server".equals(argv[i])) {
                    startSystemServer = true;
                } else if ("--enable-lazy-preload".equals(argv[i])) {
                    enableLazyPreload = true;
                } else if (argv[i].startsWith(ABI_LIST_ARG)) {
                    abiList = argv[i].substring(ABI_LIST_ARG.length());
                } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                    socketName = argv[i].substring(SOCKET_NAME_ARG.length());
                } else {
                    throw new RuntimeException("Unknown command line argument: " + argv[i]);
                }
            }

            if (abiList == null) {
                throw new RuntimeException("No ABI list supplied.");
            }

            zygoteServer.registerServerSocketFromEnv(socketName);
            // In some configurations, we avoid preloading resources and classes eagerly.
            // In such cases, we will preload things prior to our first fork.
            if (!enableLazyPreload) {
                preload(bootTimingsTraceLog);
            } else {
                Zygote.resetNicePriority();
            }


            gcAndFinalize();
            Zygote.nativeSecurityInit();
            Zygote.nativeUnmountStorageOnInit();

            ZygoteHooks.stopZygoteNoThreadCreation();

            if (startSystemServer) {
                Runnable r = forkSystemServer(abiList, socketName, zygoteServer);

                // {@code r == null} in the parent (zygote) process, and {@code r != null} in the
                // child (system_server) process.
                if (r != null) {
                    r.run();
                    return;
                }
            }

            Log.i(TAG, "Accepting command socket connections");

            // The select loop returns early in the child process after a fork and
            // loops forever in the zygote.
            caller = zygoteServer.runSelectLoop(abiList);
        } catch (Throwable ex) {
            Log.e(TAG, "System zygote died with exception", ex);
            throw ex;
        } finally {
            zygoteServer.closeServerSocket();
        }

        // We're in the child process and have exited the select loop. Proceed to execute the
        // command.
        if (caller != null) {
            caller.run();
        }
    }
G:\android9.0\frameworks\base\core\java\com\android\internal\os\ZygoteServer.java

    Runnable runSelectLoop(String abiList) {
        ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
        ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();

        fds.add(mServerSocket.getFileDescriptor());
        peers.add(null);

        while (true) {
            StructPollfd[] pollFds = new StructPollfd[fds.size()];
            for (int i = 0; i < pollFds.length; ++i) {
                pollFds[i] = new StructPollfd();
                pollFds[i].fd = fds.get(i);
                pollFds[i].events = (short) POLLIN;
            }
            try {
                Os.poll(pollFds, -1);
            } catch (ErrnoException ex) {
                throw new RuntimeException("poll failed", ex);
            }
            for (int i = pollFds.length - 1; i >= 0; --i) {
                if ((pollFds[i].revents & POLLIN) == 0) {
                    continue;
                }

                if (i == 0) {
                    ZygoteConnection newPeer = acceptCommandPeer(abiList);
                    peers.add(newPeer);
                    fds.add(newPeer.getFileDesciptor());
                } else {
                    try {
                        ZygoteConnection connection = peers.get(i);
                        final Runnable command = connection.processOneCommand(this);

                        if (mIsForkChild) {
                            if (command == null) {
                                throw new IllegalStateException("command == null");
                            }

                            return command;
                        } else {
                            if (command != null) {
                                throw new IllegalStateException("command != null");
                            }
                            if (connection.isClosedByPeer()) {
                                connection.closeSocket();
                                peers.remove(i);
                                fds.remove(i);
                            }
                        }
                    } catch (Exception e) {
                        if (!mIsForkChild) {
                            ZygoteConnection conn = peers.remove(i);
                            conn.closeSocket();

                            fds.remove(i);
                        } else {
                            Log.e(TAG, "Caught post-fork exception in child process.", e);
                            throw e;
                        }
                    } finally {
                        mIsForkChild = false;
                    }
                }
            }
        }
    }
}

```
##### 5.2.1、acceptCommandPeer()

```java
G:\android9.0\frameworks\base\core\java\com\android\internal\os\ZygoteServer.java

    private ZygoteConnection acceptCommandPeer(String abiList) {
        try {
            return createNewConnection(mServerSocket.accept(), abiList);
        } catch (IOException ex) {
            throw new RuntimeException(
                    "IOException during accept()", ex);
        }
    }

    protected ZygoteConnection createNewConnection(LocalSocket socket, String abiList)
            throws IOException {
        return new ZygoteConnection(socket, abiList);
    }

```
接收客户端发送过来的connect()操作，Zygote作为服务端执行accept()操作。 再后面客户端调用write()写数据，Zygote进程调用read()读数据。

没有连接请求时会进入休眠状态，当有创建新进程的连接请求时，唤醒Zygote进程，创建Socket通道ZygoteConnection，然后执行ZygoteConnection的processOneCommand()方法。

##### 5.2.2、processOneCommand()

``` java
G:\android9.0\frameworks\base\core\java\com\android\internal\os\ZygoteConnection.java
    Runnable processOneCommand(ZygoteServer zygoteServer) {
        String args[];
        Arguments parsedArgs = null;
        FileDescriptor[] descriptors;

        try {
            args = readArgumentList();
            descriptors = mSocket.getAncillaryFileDescriptors();
        } catch (IOException ex) {
            throw new IllegalStateException("IOException on command socket", ex);
        }

        // readArgumentList returns null only when it has reached EOF with no available
        // data to read. This will only happen when the remote socket has disconnected.
        if (args == null) {
            isEof = true;
            return null;
        }

        int pid = -1;
        FileDescriptor childPipeFd = null;
        FileDescriptor serverPipeFd = null;

        parsedArgs = new Arguments(args);

        if (parsedArgs.abiListQuery) {
            handleAbiListQuery();
            return null;
        }

        if (parsedArgs.preloadDefault) {
            handlePreload();
            return null;
        }

        if (parsedArgs.preloadPackage != null) {
            handlePreloadPackage(parsedArgs.preloadPackage, parsedArgs.preloadPackageLibs,
                    parsedArgs.preloadPackageLibFileName, parsedArgs.preloadPackageCacheKey);
            return null;
        }

        if (parsedArgs.apiBlacklistExemptions != null) {
            handleApiBlacklistExemptions(parsedArgs.apiBlacklistExemptions);
            return null;
        }

        if (parsedArgs.hiddenApiAccessLogSampleRate != -1) {
            handleHiddenApiAccessLogSampleRate(parsedArgs.hiddenApiAccessLogSampleRate);
            return null;
        }

        if (parsedArgs.permittedCapabilities != 0 || parsedArgs.effectiveCapabilities != 0) {
            throw new ZygoteSecurityException("Client may not specify capabilities: " +
                    "permitted=0x" + Long.toHexString(parsedArgs.permittedCapabilities) +
                    ", effective=0x" + Long.toHexString(parsedArgs.effectiveCapabilities));
        }

        applyUidSecurityPolicy(parsedArgs, peer);
        applyInvokeWithSecurityPolicy(parsedArgs, peer);

        applyDebuggerSystemProperty(parsedArgs);
        applyInvokeWithSystemProperty(parsedArgs);

        int[][] rlimits = null;

        if (parsedArgs.rlimits != null) {
            rlimits = parsedArgs.rlimits.toArray(intArray2d);
        }

        int[] fdsToIgnore = null;

        if (parsedArgs.invokeWith != null) {
            try {
                FileDescriptor[] pipeFds = Os.pipe2(O_CLOEXEC);
                childPipeFd = pipeFds[1];
                serverPipeFd = pipeFds[0];
                Os.fcntlInt(childPipeFd, F_SETFD, 0);
                fdsToIgnore = new int[]{childPipeFd.getInt$(), serverPipeFd.getInt$()};
            } catch (ErrnoException errnoEx) {
                throw new IllegalStateException("Unable to set up pipe for invoke-with", errnoEx);
            }
        }

        /**
         * In order to avoid leaking descriptors to the Zygote child,
         * the native code must close the two Zygote socket descriptors
         * in the child process before it switches from Zygote-root to
         * the UID and privileges of the application being launched.
         *
         * In order to avoid "bad file descriptor" errors when the
         * two LocalSocket objects are closed, the Posix file
         * descriptors are released via a dup2() call which closes
         * the socket and substitutes an open descriptor to /dev/null.
         */

        int [] fdsToClose = { -1, -1 };

        FileDescriptor fd = mSocket.getFileDescriptor();

        if (fd != null) {
            fdsToClose[0] = fd.getInt$();
        }

        fd = zygoteServer.getServerSocketFileDescriptor();

        if (fd != null) {
            fdsToClose[1] = fd.getInt$();
        }

        fd = null;

        pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
                parsedArgs.runtimeFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,
                parsedArgs.niceName, fdsToClose, fdsToIgnore, parsedArgs.startChildZygote,
                parsedArgs.instructionSet, parsedArgs.appDataDir);

        try {
            if (pid == 0) {
                // in child
                zygoteServer.setForkChild();

                zygoteServer.closeServerSocket();
                IoUtils.closeQuietly(serverPipeFd);
                serverPipeFd = null;

                return handleChildProc(parsedArgs, descriptors, childPipeFd,
                        parsedArgs.startChildZygote);
            } else {
                // In the parent. A pid < 0 indicates a failure and will be handled in
                // handleParentProc.
                IoUtils.closeQuietly(childPipeFd);
                childPipeFd = null;
                handleParentProc(pid, descriptors, serverPipeFd);
                return null;
            }
        } finally {
            IoUtils.closeQuietly(childPipeFd);
            IoUtils.closeQuietly(serverPipeFd);
        }
    }
```
##### 5.2.3、forkAndSpecialize()

``` java
G:\android9.0\frameworks\base\core\java\com\android\internal\os\Zygote.java

    public static int forkAndSpecialize(int uid, int gid, int[] gids, int runtimeFlags,
          int[][] rlimits, int mountExternal, String seInfo, String niceName, int[] fdsToClose,
          int[] fdsToIgnore, boolean startChildZygote, String instructionSet, String appDataDir) {
        VM_HOOKS.preFork();
        // Resets nice priority for zygote process.
        resetNicePriority();
        int pid = nativeForkAndSpecialize(
                  uid, gid, gids, runtimeFlags, rlimits, mountExternal, seInfo, niceName, fdsToClose,
                  fdsToIgnore, startChildZygote, instructionSet, appDataDir);
        // Enable tracing as soon as possible for the child process.
        if (pid == 0) {
            Trace.setTracingEnabled(true, runtimeFlags);

            // Note that this event ends at the end of handleChildProc,
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "PostFork");
        }
        VM_HOOKS.postForkCommon();
        return pid;
    }

```
VM_HOOKS是Zygote对象的静态成员变量：VM_HOOKS = new ZygoteHooks();


##### 5.2.4、nativeForkAndSpecialize
nativeForkAndSpecialize()通过JNI最终调用调用如下方法：

``` cpp
G:\android9.0\frameworks\base\core\jni\com_android_internal_os_Zygote.cpp
static jint com_android_internal_os_Zygote_nativeForkAndSpecialize(
        JNIEnv* env, jclass, jint uid, jint gid, jintArray gids,
        jint runtime_flags, jobjectArray rlimits,
        jint mount_external, jstring se_info, jstring se_name,
        jintArray fdsToClose, jintArray fdsToIgnore, jboolean is_child_zygote,
        jstring instructionSet, jstring appDataDir) {
    jlong capabilities = 0;

    // Grant CAP_WAKE_ALARM to the Bluetooth process.
    // Additionally, allow bluetooth to open packet sockets so it can start the DHCP client.
    // Grant CAP_SYS_NICE to allow Bluetooth to set RT priority for
    // audio-related threads.
    // TODO: consider making such functionality an RPC to netd.
    if (multiuser_get_app_id(uid) == AID_BLUETOOTH) {
      capabilities |= (1LL << CAP_WAKE_ALARM);
      capabilities |= (1LL << CAP_NET_RAW);
      capabilities |= (1LL << CAP_NET_BIND_SERVICE);
      capabilities |= (1LL << CAP_SYS_NICE);
    }

    // Grant CAP_BLOCK_SUSPEND to processes that belong to GID "wakelock"
    bool gid_wakelock_found = false;
    if (gid == AID_WAKELOCK) {
      gid_wakelock_found = true;
    } else if (gids != NULL) {
      jsize gids_num = env->GetArrayLength(gids);
      ScopedIntArrayRO ar(env, gids);
      if (ar.get() == NULL) {
        RuntimeAbort(env, __LINE__, "Bad gids array");
      }
      for (int i = 0; i < gids_num; i++) {
        if (ar[i] == AID_WAKELOCK) {
          gid_wakelock_found = true;
          break;
        }
      }
    }
    if (gid_wakelock_found) {
      capabilities |= (1LL << CAP_BLOCK_SUSPEND);
    }

    // If forking a child zygote process, that zygote will need to be able to change
    // the UID and GID of processes it forks, as well as drop those capabilities.
    if (is_child_zygote) {
      capabilities |= (1LL << CAP_SETUID);
      capabilities |= (1LL << CAP_SETGID);
      capabilities |= (1LL << CAP_SETPCAP);
    }

    // Containers run without some capabilities, so drop any caps that are not
    // available.
    capabilities &= GetEffectiveCapabilityMask(env);

    return ForkAndSpecializeCommon(env, uid, gid, gids, runtime_flags,
            rlimits, capabilities, capabilities, mount_external, se_info,
            se_name, false, fdsToClose, fdsToIgnore, is_child_zygote == JNI_TRUE,
            instructionSet, appDataDir);
}
```
##### 5.2.5、ForkAndSpecializeCommon

``` cpp
G:\android9.0\frameworks\base\core\jni\com_android_internal_os_Zygote.cpp

static pid_t ForkAndSpecializeCommon(JNIEnv* env, uid_t uid, gid_t gid, jintArray javaGids,
                                     jint runtime_flags, jobjectArray javaRlimits,
                                     jlong permittedCapabilities, jlong effectiveCapabilities,
                                     jint mount_external,
                                     jstring java_se_info, jstring java_se_name,
                                     bool is_system_server, jintArray fdsToClose,
                                     jintArray fdsToIgnore, bool is_child_zygote,
                                     jstring instructionSet, jstring dataDir) {
  SetSignalHandlers();

  sigset_t sigchld;
  sigemptyset(&sigchld);
  sigaddset(&sigchld, SIGCHLD);

  auto fail_fn = [env, java_se_name, is_system_server](const std::string& msg)
      __attribute__ ((noreturn)) {
    const char* se_name_c_str = nullptr;
    std::unique_ptr<ScopedUtfChars> se_name;
    if (java_se_name != nullptr) {
      se_name.reset(new ScopedUtfChars(env, java_se_name));
      se_name_c_str = se_name->c_str();
    }
    if (se_name_c_str == nullptr && is_system_server) {
      se_name_c_str = "system_server";
    }
    const std::string& error_msg = (se_name_c_str == nullptr)
        ? msg
        : StringPrintf("(%s) %s", se_name_c_str, msg.c_str());
    env->FatalError(error_msg.c_str());
    __builtin_unreachable();
  };

  // Temporarily block SIGCHLD during forks. The SIGCHLD handler might
  // log, which would result in the logging FDs we close being reopened.
  // This would cause failures because the FDs are not whitelisted.
  //
  // Note that the zygote process is single threaded at this point.
  if (sigprocmask(SIG_BLOCK, &sigchld, nullptr) == -1) {
    fail_fn(CREATE_ERROR("sigprocmask(SIG_SETMASK, { SIGCHLD }) failed: %s", strerror(errno)));
  }

  // Close any logging related FDs before we start evaluating the list of
  // file descriptors.
  __android_log_close();

  std::string error_msg;

  // If this is the first fork for this zygote, create the open FD table.
  // If it isn't, we just need to check whether the list of open files has
  // changed (and it shouldn't in the normal case).
  std::vector<int> fds_to_ignore;
  if (!FillFileDescriptorVector(env, fdsToIgnore, &fds_to_ignore, &error_msg)) {
    fail_fn(error_msg);
  }
  if (gOpenFdTable == NULL) {
    gOpenFdTable = FileDescriptorTable::Create(fds_to_ignore, &error_msg);
    if (gOpenFdTable == NULL) {
      fail_fn(error_msg);
    }
  } else if (!gOpenFdTable->Restat(fds_to_ignore, &error_msg)) {
    fail_fn(error_msg);
  }

  pid_t pid = fork();

  if (pid == 0) {
    PreApplicationInit();

    // Clean up any descriptors which must be closed immediately
    if (!DetachDescriptors(env, fdsToClose, &error_msg)) {
      fail_fn(error_msg);
    }

    // Re-open all remaining open file descriptors so that they aren't shared
    // with the zygote across a fork.
    if (!gOpenFdTable->ReopenOrDetach(&error_msg)) {
      fail_fn(error_msg);
    }

    if (sigprocmask(SIG_UNBLOCK, &sigchld, nullptr) == -1) {
      fail_fn(CREATE_ERROR("sigprocmask(SIG_SETMASK, { SIGCHLD }) failed: %s", strerror(errno)));
    }

    // Keep capabilities across UID change, unless we're staying root.
    if (uid != 0) {
      if (!EnableKeepCapabilities(&error_msg)) {
        fail_fn(error_msg);
      }
    }

    if (!SetInheritable(permittedCapabilities, &error_msg)) {
      fail_fn(error_msg);
    }
    if (!DropCapabilitiesBoundingSet(&error_msg)) {
      fail_fn(error_msg);
    }

    bool use_native_bridge = !is_system_server && (instructionSet != NULL)
        && android::NativeBridgeAvailable();
    if (use_native_bridge) {
      ScopedUtfChars isa_string(env, instructionSet);
      use_native_bridge = android::NeedsNativeBridge(isa_string.c_str());
    }
    if (use_native_bridge && dataDir == NULL) {
      // dataDir should never be null if we need to use a native bridge.
      // In general, dataDir will never be null for normal applications. It can only happen in
      // special cases (for isolated processes which are not associated with any app). These are
      // launched by the framework and should not be emulated anyway.
      use_native_bridge = false;
      ALOGW("Native bridge will not be used because dataDir == NULL.");
    }

    if (!MountEmulatedStorage(uid, mount_external, use_native_bridge, &error_msg)) {
      ALOGW("Failed to mount emulated storage: %s (%s)", error_msg.c_str(), strerror(errno));
      if (errno == ENOTCONN || errno == EROFS) {
        // When device is actively encrypting, we get ENOTCONN here
        // since FUSE was mounted before the framework restarted.
        // When encrypted device is booting, we get EROFS since
        // FUSE hasn't been created yet by init.
        // In either case, continue without external storage.
      } else {
        fail_fn(error_msg);
      }
    }

    // If this zygote isn't root, it won't be able to create a process group,
    // since the directory is owned by root.
    if (!is_system_server && getuid() == 0) {
        int rc = createProcessGroup(uid, getpid());
        if (rc != 0) {
            if (rc == -EROFS) {
                ALOGW("createProcessGroup failed, kernel missing CONFIG_CGROUP_CPUACCT?");
            } else {
                ALOGE("createProcessGroup(%d, %d) failed: %s", uid, pid, strerror(-rc));
            }
        }
    }

    std::string error_msg;
    if (!SetGids(env, javaGids, &error_msg)) {
      fail_fn(error_msg);
    }

    if (!SetRLimits(env, javaRlimits, &error_msg)) {
      fail_fn(error_msg);
    }

    if (use_native_bridge) {
      ScopedUtfChars isa_string(env, instructionSet);
      ScopedUtfChars data_dir(env, dataDir);
      android::PreInitializeNativeBridge(data_dir.c_str(), isa_string.c_str());
    }

    int rc = setresgid(gid, gid, gid);
    if (rc == -1) {
      fail_fn(CREATE_ERROR("setresgid(%d) failed: %s", gid, strerror(errno)));
    }

    // Must be called when the new process still has CAP_SYS_ADMIN, in this case, before changing
    // uid from 0, which clears capabilities.  The other alternative is to call
    // prctl(PR_SET_NO_NEW_PRIVS, 1) afterward, but that breaks SELinux domain transition (see
    // b/71859146).  As the result, privileged syscalls used below still need to be accessible in
    // app process.
    SetUpSeccompFilter(uid);

    rc = setresuid(uid, uid, uid);
    if (rc == -1) {
      fail_fn(CREATE_ERROR("setresuid(%d) failed: %s", uid, strerror(errno)));
    }

    if (NeedsNoRandomizeWorkaround()) {
        // Work around ARM kernel ASLR lossage (http://b/5817320).
        int old_personality = personality(0xffffffff);
        int new_personality = personality(old_personality | ADDR_NO_RANDOMIZE);
        if (new_personality == -1) {
            ALOGW("personality(%d) failed: %s", new_personality, strerror(errno));
        }
    }

    if (!SetCapabilities(permittedCapabilities, effectiveCapabilities, permittedCapabilities,
                         &error_msg)) {
      fail_fn(error_msg);
    }

    if (!SetSchedulerPolicy(&error_msg)) {
      fail_fn(error_msg);
    }

    const char* se_info_c_str = NULL;
    ScopedUtfChars* se_info = NULL;
    if (java_se_info != NULL) {
        se_info = new ScopedUtfChars(env, java_se_info);
        se_info_c_str = se_info->c_str();
        if (se_info_c_str == NULL) {
          fail_fn("se_info_c_str == NULL");
        }
    }
    const char* se_name_c_str = NULL;
    ScopedUtfChars* se_name = NULL;
    if (java_se_name != NULL) {
        se_name = new ScopedUtfChars(env, java_se_name);
        se_name_c_str = se_name->c_str();
        if (se_name_c_str == NULL) {
          fail_fn("se_name_c_str == NULL");
        }
    }
    rc = selinux_android_setcontext(uid, is_system_server, se_info_c_str, se_name_c_str);
    if (rc == -1) {
      fail_fn(CREATE_ERROR("selinux_android_setcontext(%d, %d, \"%s\", \"%s\") failed", uid,
            is_system_server, se_info_c_str, se_name_c_str));
    }

    // Make it easier to debug audit logs by setting the main thread's name to the
    // nice name rather than "app_process".
    if (se_name_c_str == NULL && is_system_server) {
      se_name_c_str = "system_server";
    }
    if (se_name_c_str != NULL) {
      SetThreadName(se_name_c_str);
    }

    delete se_info;
    delete se_name;

    // Unset the SIGCHLD handler, but keep ignoring SIGHUP (rationale in SetSignalHandlers).
    UnsetChldSignalHandler();

    env->CallStaticVoidMethod(gZygoteClass, gCallPostForkChildHooks, runtime_flags,
                              is_system_server, is_child_zygote, instructionSet);
    if (env->ExceptionCheck()) {
      fail_fn("Error calling post fork hooks.");
    }
  } else if (pid > 0) {
    // the parent process

    // We blocked SIGCHLD prior to a fork, we unblock it here.
    if (sigprocmask(SIG_UNBLOCK, &sigchld, nullptr) == -1) {
      fail_fn(CREATE_ERROR("sigprocmask(SIG_SETMASK, { SIGCHLD }) failed: %s", strerror(errno)));
    }
  }
  return pid;
}
```
##### 5.3、新进程运行
##### 5.3.1、handleChildProc()
processOneCommand()过程中调用forkAndSpecialize()创建完新进程后，返回值pid=0(即运行在子进程)继续开始执行handleChildProc()方法。

``` cpp
G:\android9.0\frameworks\base\core\java\com\android\internal\os\ZygoteConnection.java
    private Runnable handleChildProc(Arguments parsedArgs, FileDescriptor[] descriptors,
            FileDescriptor pipeFd, boolean isZygote) {
        /**
         * By the time we get here, the native code has closed the two actual Zygote
         * socket connections, and substituted /dev/null in their place.  The LocalSocket
         * objects still need to be closed properly.
         */

        closeSocket();
        if (descriptors != null) {
            try {
                Os.dup2(descriptors[0], STDIN_FILENO);
                Os.dup2(descriptors[1], STDOUT_FILENO);
                Os.dup2(descriptors[2], STDERR_FILENO);

                for (FileDescriptor fd: descriptors) {
                    IoUtils.closeQuietly(fd);
                }
            } catch (ErrnoException ex) {
                Log.e(TAG, "Error reopening stdio", ex);
            }
        }

        if (parsedArgs.niceName != null) {
            Process.setArgV0(parsedArgs.niceName);
        }

        // End of the postFork event.
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        if (parsedArgs.invokeWith != null) {
            WrapperInit.execApplication(parsedArgs.invokeWith,
                    parsedArgs.niceName, parsedArgs.targetSdkVersion,
                    VMRuntime.getCurrentInstructionSet(),
                    pipeFd, parsedArgs.remainingArgs);

            // Should not get here.
            throw new IllegalStateException("WrapperInit.execApplication unexpectedly returned");
        } else {
            if (!isZygote) {
                return ZygoteInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs,
                        null /* classLoader */);
            } else {
                return ZygoteInit.childZygoteInit(parsedArgs.targetSdkVersion,
                        parsedArgs.remainingArgs, null /* classLoader */);
            }
        }
    }

```
##### 5.3.2、ZygoteInit.zygoteInit()

``` cpp
G:\android9.0\frameworks\base\core\java\com\android\internal\os\ZygoteInit.java
    public static final Runnable zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader) {
        if (RuntimeInit.DEBUG) {
            Slog.d(RuntimeInit.TAG, "RuntimeInit: Starting application from zygote");
        }

        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ZygoteInit");
        RuntimeInit.redirectLogStreams();

        RuntimeInit.commonInit();
        ZygoteInit.nativeZygoteInit();
        return RuntimeInit.applicationInit(targetSdkVersion, argv, classLoader);
    }
```
##### 5.3.3、RuntimeInit.commonInit()
``` cpp
G:\android9.0\frameworks\base\core\java\com\android\internal\os\RuntimeInit.java
    protected static final void commonInit() {
        if (DEBUG) Slog.d(TAG, "Entered RuntimeInit!");
        LoggingHandler loggingHandler = new LoggingHandler();
        Thread.setUncaughtExceptionPreHandler(loggingHandler);
        Thread.setDefaultUncaughtExceptionHandler(new KillApplicationHandler(loggingHandler));
        
        TimezoneGetter.setInstance(new TimezoneGetter() {
            @Override
            public String getId() {
                return SystemProperties.get("persist.sys.timezone");
            }
        });
        TimeZone.setDefault(null);

        LogManager.getLogManager().reset();
        new AndroidConfig()
        String userAgent = getDefaultUserAgent();
        System.setProperty("http.agent", userAgent);

        NetworkManagementSocketTagger.install();

        String trace = SystemProperties.get("ro.kernel.android.tracing");
        if (trace.equals("1")) {
            Slog.i(TAG, "NOTE: emulator trace profiling enabled");
            Debug.enableEmulatorTraceOutput();
        }

        initialized = true;
    }

```
##### 5.3.4、nativeZygoteInit

``` cpp
G:\android9.0\frameworks\base\core\jni\AndroidRuntime.cpp
static void com_android_internal_os_ZygoteInit_nativeZygoteInit(JNIEnv* env, jobject clazz)
{
    gCurRuntime->onZygoteInit();
}

G:\android9.0\frameworks\base\cmds\app_process\app_main.cpp
    virtual void onZygoteInit()
    {
        sp<ProcessState> proc = ProcessState::self();
        ALOGV("App process: starting thread pool.\n");
        proc->startThreadPool();
    }

```
ProcessState::self():主要工作是调用open()打开/dev/binder驱动设备，再利用mmap()映射内核的地址空间，将Binder驱动的fd赋值ProcessState对象中的变量mDriverFD，用于交互操作。startThreadPool()是创建一个新的binder线程，不断进行talkWithDriver().
startThreadPool(): 启动Binder线程池
##### 5.3.5、RuntimeInit.applicationInit()

``` java
G:\android9.0\frameworks\base\core\java\com\android\internal\os\RuntimeInit.java

    protected static Runnable applicationInit(int targetSdkVersion, String[] argv,
            ClassLoader classLoader) {
        // If the application calls System.exit(), terminate the process
        // immediately without running any shutdown hooks.  It is not possible to
        // shutdown an Android application gracefully.  Among other things, the
        // Android runtime shutdown hooks close the Binder driver, which can cause
        // leftover running threads to crash before the process actually exits.
        nativeSetExitWithoutCleanup(true);

        // We want to be fairly aggressive about heap utilization, to avoid
        // holding on to a lot of memory that isn't needed.
        VMRuntime.getRuntime().setTargetHeapUtilization(0.75f);
        VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);

        final Arguments args = new Arguments(argv);

        // The end of of the RuntimeInit event (see #zygoteInit).
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);

        // Remaining arguments are passed to the start class's static main
        return findStaticMain(args.startClass, args.startArgs, classLoader);
    }

```

此处args.startClass为”android.app.ActivityThread”。
##### 5.3.6、findStaticMain()

``` cpp

    protected static Runnable findStaticMain(String className, String[] argv,
            ClassLoader classLoader) {
        Class<?> cl;

        try {
            cl = Class.forName(className, true, classLoader);
        } catch (ClassNotFoundException ex) {
            throw new RuntimeException(
                    "Missing class when invoking static main " + className,
                    ex);
        }

        Method m;
        try {
            m = cl.getMethod("main", new Class[] { String[].class });
        } catch (NoSuchMethodException ex) {
            throw new RuntimeException(
                    "Missing static main on " + className, ex);
        } catch (SecurityException ex) {
            throw new RuntimeException(
                    "Problem getting static main on " + className, ex);
        }

        int modifiers = m.getModifiers();
        if (! (Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {
            throw new RuntimeException(
                    "Main method is not public and static on " + className);
        }

        /*
         * This throw gets caught in ZygoteInit.main(), which responds
         * by invoking the exception's run() method. This arrangement
         * clears up all the stack frames that were required in setting
         * up the process.
         */
        return new MethodAndArgsCaller(m, argv);
    }

```
##### 5.3.7、MethodAndArgsCaller

``` cpp
    static class MethodAndArgsCaller implements Runnable {
        /** method to call */
        private final Method mMethod;

        /** argument array */
        private final String[] mArgs;

        public MethodAndArgsCaller(Method method, String[] args) {
            mMethod = method;
            mArgs = args;
        }

        public void run() {
            try {
                //根据AMS startProcessLocked传递过来的参数，此处反射调用ActivityThread.main()方法
                mMethod.invoke(null, new Object[] { mArgs });
            } catch (IllegalAccessException ex) {
                throw new RuntimeException(ex);
            } catch (InvocationTargetException ex) {
                Throwable cause = ex.getCause();
                if (cause instanceof RuntimeException) {
                    throw (RuntimeException) cause;
                } else if (cause instanceof Error) {
                    throw (Error) cause;
                }
                throw new RuntimeException(ex);
            }
        }
    }
```
##### 5.3.8、Activity所在应用主线程初始化
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

    @GuardedBy("this")
    private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid, int callingUid, long startSeq) {

        // Find the application record that is being attached...  either via
        // the pid if we are running in multiple processes, or just pull the
        // next app record if we are emulating process with anonymous threads.
        ProcessRecord app;
        long startTime = SystemClock.uptimeMillis();
        if (pid != MY_PID && pid >= 0) {
            synchronized (mPidsSelfLocked) {
                app = mPidsSelfLocked.get(pid);
            }
        } else {
            app = null;
        }

        // It's possible that process called attachApplication before we got a chance to
        // update the internal state.
        if (app == null && startSeq > 0) {
            final ProcessRecord pending = mPendingStarts.get(startSeq);
            if (pending != null && pending.startUid == callingUid
                    && handleProcessStartedLocked(pending, pid, pending.usingWrapper,
                            startSeq, true)) {
                app = pending;
            }
        }

        if (app == null) {
            Slog.w(TAG, "No pending application record for pid " + pid
                    + " (IApplicationThread " + thread + "); dropping process");
            EventLog.writeEvent(EventLogTags.AM_DROP_PROCESS, pid);
            if (pid > 0 && pid != MY_PID) {
                killProcessQuiet(pid);
                //TODO: killProcessGroup(app.info.uid, pid);
            } else {
                try {
                    thread.scheduleExit();
                } catch (Exception e) {
                    // Ignore exceptions.
                }
            }
            return false;
        }

        // If this application record is still attached to a previous
        // process, clean it up now.
        if (app.thread != null) {
            handleAppDiedLocked(app, true, true);
        }

        // Tell the process all about itself.

        if (DEBUG_ALL) Slog.v(
                TAG, "Binding process pid " + pid + " to record " + app);
------------------------------------
对应Log：
04-16 13:08:36.817 I/ActivityManagerService( 1049): Start proc 2602:com.android.testred/u0a88 for activity com.android.testred/.TestActivity
04-16 13:08:36.837 V/ActivityManagerService( 1049): Binding process pid 2602 to record ProcessRecord{d768ca0 2602:com.android.testred/u0a88}
------------------------------------
        final String processName = app.processName;
        try {
            AppDeathRecipient adr = new AppDeathRecipient(
                    app, pid, thread);
            thread.asBinder().linkToDeath(adr, 0);
            app.deathRecipient = adr;
------------------------------------
对应Log：
04-16 13:08:36.837 V/ActivityManagerService( 1049): New death recipient com.android.server.am.ActivityManagerService$AppDeathRecipient@53b14ff for thread android.os.BinderProxy@6c7d2cc
------------------------------------
        } catch (RemoteException e) {
            app.resetPackageList(mProcessStats);
            startProcessLocked(app, "link fail", processName);
            return false;
        }

        EventLog.writeEvent(EventLogTags.AM_PROC_BOUND, app.userId, app.pid, app.processName);

        app.makeActive(thread, mProcessStats);
        app.curAdj = app.setAdj = app.verifiedAdj = ProcessList.INVALID_ADJ;
        app.curSchedGroup = app.setSchedGroup = ProcessList.SCHED_GROUP_DEFAULT;
        app.forcingToImportant = null;
        updateProcessForegroundLocked(app, false, false);
        app.hasShownUi = false;
        app.debugging = false;
        app.cached = false;
        app.killedByAm = false;
        app.killed = false;


        // We carefully use the same state that PackageManager uses for
        // filtering, since we use this flag to decide if we need to install
        // providers when user is unlocked later
        app.unlocked = StorageManager.isUserKeyUnlocked(app.userId);

        mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);

        boolean normalMode = mProcessesReady || isAllowedWhileBooting(app.info);
        List<ProviderInfo> providers = normalMode ? generateApplicationProvidersLocked(app) : null;

        if (providers != null && checkAppInLaunchingProvidersLocked(app)) {
            Message msg = mHandler.obtainMessage(CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG);
            msg.obj = app;
            mHandler.sendMessageDelayed(msg, CONTENT_PROVIDER_PUBLISH_TIMEOUT);
        }

        checkTime(startTime, "attachApplicationLocked: before bindApplication");

        if (!normalMode) {
            Slog.i(TAG, "Launching preboot mode app: " + app);
        }
------------------------------------
对应Log：
04-16 13:08:36.838 V/ActivityManagerService( 1049): New app record ProcessRecord{d768ca0 2602:com.android.testred/u0a88} thread=android.os.BinderProxy@6c7d2cc pid=2602
04-16 13:08:36.838 V/ActivityManagerService_Configuration( 1049): Binding proc com.android.testred with config {1.0 ?mcc?mnc [en_US] ldltr sw320dp w320dp h497dp 240dpi nrml port finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 0, 0) mAppBounds=Rect(0, 0 - 480, 782) mWindowingMode=fullscreen mActivityType=undefined} s.5}
------------------------------------
        if (DEBUG_ALL) Slog.v(
            TAG, "New app record " + app
            + " thread=" + thread.asBinder() + " pid=" + pid);
        try {
            int testMode = ApplicationThreadConstants.DEBUG_OFF;
            if (mDebugApp != null && mDebugApp.equals(processName)) {
                testMode = mWaitForDebugger
                    ? ApplicationThreadConstants.DEBUG_WAIT
                    : ApplicationThreadConstants.DEBUG_ON;
                app.debugging = true;
                if (mDebugTransient) {
                    mDebugApp = mOrigDebugApp;
                    mWaitForDebugger = mOrigWaitForDebugger;
                }
            }

            boolean enableTrackAllocation = false;
            if (mTrackAllocationApp != null && mTrackAllocationApp.equals(processName)) {
                enableTrackAllocation = true;
                mTrackAllocationApp = null;
            }

            // If the app is being launched for restore or full backup, set it up specially
            boolean isRestrictedBackupMode = false;
            if (mBackupTarget != null && mBackupAppName.equals(processName)) {
                isRestrictedBackupMode = mBackupTarget.appInfo.uid >= FIRST_APPLICATION_UID
                        && ((mBackupTarget.backupMode == BackupRecord.RESTORE)
                                || (mBackupTarget.backupMode == BackupRecord.RESTORE_FULL)
                                || (mBackupTarget.backupMode == BackupRecord.BACKUP_FULL));
            }

            if (app.instr != null) {
                notifyPackageUse(app.instr.mClass.getPackageName(),
                                 PackageManager.NOTIFY_PACKAGE_USE_INSTRUMENTATION);
            }
            if (DEBUG_CONFIGURATION) Slog.v(TAG_CONFIGURATION, "Binding proc "
                    + processName + " with config " + getGlobalConfiguration());
            ApplicationInfo appInfo = app.instr != null ? app.instr.mTargetInfo : app.info;
            app.compat = compatibilityInfoForPackageLocked(appInfo);

            ProfilerInfo profilerInfo = null;
            String preBindAgent = null;
            if (mProfileApp != null && mProfileApp.equals(processName)) {
                mProfileProc = app;
                if (mProfilerInfo != null) {
                    // Send a profiler info object to the app if either a file is given, or
                    // an agent should be loaded at bind-time.
                    boolean needsInfo = mProfilerInfo.profileFile != null
                            || mProfilerInfo.attachAgentDuringBind;
                    profilerInfo = needsInfo ? new ProfilerInfo(mProfilerInfo) : null;
                    if (mProfilerInfo.agent != null) {
                        preBindAgent = mProfilerInfo.agent;
                    }
                }
            } else if (app.instr != null && app.instr.mProfileFile != null) {
                profilerInfo = new ProfilerInfo(app.instr.mProfileFile, null, 0, false, false,
                        null, false);
            }
            if (mAppAgentMap != null && mAppAgentMap.containsKey(processName)) {
                // We need to do a debuggable check here. See setAgentApp for why the check is
                // postponed to here.
                if ((app.info.flags & ApplicationInfo.FLAG_DEBUGGABLE) != 0) {
                    String agent = mAppAgentMap.get(processName);
                    // Do not overwrite already requested agent.
                    if (profilerInfo == null) {
                        profilerInfo = new ProfilerInfo(null, null, 0, false, false,
                                mAppAgentMap.get(processName), true);
                    } else if (profilerInfo.agent == null) {
                        profilerInfo = profilerInfo.setAgent(mAppAgentMap.get(processName), true);
                    }
                }
            }

            if (profilerInfo != null && profilerInfo.profileFd != null) {
                profilerInfo.profileFd = profilerInfo.profileFd.dup();
                if (TextUtils.equals(mProfileApp, processName) && mProfilerInfo != null) {
                    clearProfilerLocked();
                }
            }

            // We deprecated Build.SERIAL and it is not accessible to
            // apps that target the v2 security sandbox and to apps that
            // target APIs higher than O MR1. Since access to the serial
            // is now behind a permission we push down the value.
            final String buildSerial = (appInfo.targetSandboxVersion < 2
                    && appInfo.targetSdkVersion < Build.VERSION_CODES.P)
                            ? sTheRealBuildSerial : Build.UNKNOWN;

            // Check if this is a secondary process that should be incorporated into some
            // currently active instrumentation.  (Note we do this AFTER all of the profiling
            // stuff above because profiling can currently happen only in the primary
            // instrumentation process.)
            if (mActiveInstrumentation.size() > 0 && app.instr == null) {
                for (int i = mActiveInstrumentation.size() - 1; i >= 0 && app.instr == null; i--) {
                    ActiveInstrumentation aInstr = mActiveInstrumentation.get(i);
                    if (!aInstr.mFinished && aInstr.mTargetInfo.uid == app.uid) {
                        if (aInstr.mTargetProcesses.length == 0) {
                            // This is the wildcard mode, where every process brought up for
                            // the target instrumentation should be included.
                            if (aInstr.mTargetInfo.packageName.equals(app.info.packageName)) {
                                app.instr = aInstr;
                                aInstr.mRunningProcesses.add(app);
                            }
                        } else {
                            for (String proc : aInstr.mTargetProcesses) {
                                if (proc.equals(app.processName)) {
                                    app.instr = aInstr;
                                    aInstr.mRunningProcesses.add(app);
                                    break;
                                }
                            }
                        }
                    }
                }
            }

            // If we were asked to attach an agent on startup, do so now, before we're binding
            // application code.
            if (preBindAgent != null) {
                thread.attachAgent(preBindAgent);
            }


            // Figure out whether the app needs to run in autofill compat mode.
            boolean isAutofillCompatEnabled = false;
            if (UserHandle.getAppId(app.info.uid) >= Process.FIRST_APPLICATION_UID) {
                final AutofillManagerInternal afm = LocalServices.getService(
                        AutofillManagerInternal.class);
                if (afm != null) {
                    isAutofillCompatEnabled = afm.isCompatibilityModeRequested(
                            app.info.packageName, app.info.versionCode, app.userId);
                }
            }

            checkTime(startTime, "attachApplicationLocked: immediately before bindApplication");
            mStackSupervisor.getActivityMetricsLogger().notifyBindApplication(app);
            if (app.isolatedEntryPoint != null) {
                // This is an isolated process which should just call an entry point instead of
                // being bound to an application.
                thread.runIsolatedEntryPoint(app.isolatedEntryPoint, app.isolatedEntryPointArgs);
            } else if (app.instr != null) {
                thread.bindApplication(processName, appInfo, providers,
                        app.instr.mClass,
                        profilerInfo, app.instr.mArguments,
                        app.instr.mWatcher,
                        app.instr.mUiAutomationConnection, testMode,
                        mBinderTransactionTrackingEnabled, enableTrackAllocation,
                        isRestrictedBackupMode || !normalMode, app.persistent,
                        new Configuration(getGlobalConfiguration()), app.compat,
                        getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked(),
                        buildSerial, isAutofillCompatEnabled);
            } else {
                thread.bindApplication(processName, appInfo, providers, null, profilerInfo,
                        null, null, null, testMode,
                        mBinderTransactionTrackingEnabled, enableTrackAllocation,
                        isRestrictedBackupMode || !normalMode, app.persistent,
                        new Configuration(getGlobalConfiguration()), app.compat,
                        getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked(),
                        buildSerial, isAutofillCompatEnabled);
            }
            if (profilerInfo != null) {
                profilerInfo.closeFd();
                profilerInfo = null;
            }
            checkTime(startTime, "attachApplicationLocked: immediately after bindApplication");
------------------------
对应Log：
04-16 13:08:36.838 D/ActivityManagerService_LRU( 1049): Adding at 45 of LRU list: ProcessRecord{d768ca0 2602:com.android.testred/u0a88}
------------------------
            updateLruProcessLocked(app, false, null);
            checkTime(startTime, "attachApplicationLocked: after updateLruProcessLocked");
            app.lastRequestedGc = app.lastLowMemory = SystemClock.uptimeMillis();
        } catch (Exception e) {
            // todo: Yikes!  What should we do?  For now we will try to
            // start another process, but that could easily get us in
            // an infinite loop of restarting processes...
            Slog.wtf(TAG, "Exception thrown during bind of " + app, e);

            app.resetPackageList(mProcessStats);
            app.unlinkDeathRecipient();
            startProcessLocked(app, "bind fail", processName);
            return false;
        }

        // Remove this record from the list of starting applications.
        mPersistentStartingProcesses.remove(app);
        if (DEBUG_PROCESSES && mProcessesOnHold.contains(app)) Slog.v(TAG_PROCESSES,
                "Attach application locked removing on hold: " + app);
        mProcessesOnHold.remove(app);

        boolean badApp = false;
        boolean didSomething = false;

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

        // Find any services that should be running in this process...
        if (!badApp) {
            try {
                didSomething |= mServices.attachApplicationLocked(app, processName);
                checkTime(startTime, "attachApplicationLocked: after mServices.attachApplicationLocked");
            } catch (Exception e) {
                Slog.wtf(TAG, "Exception thrown starting services in " + app, e);
                badApp = true;
            }
        }

        // Check if a next-broadcast receiver is in this process...
        if (!badApp && isPendingBroadcastProcessLocked(pid)) {
            try {
                didSomething |= sendPendingBroadcastsLocked(app);
                checkTime(startTime, "attachApplicationLocked: after sendPendingBroadcastsLocked");
            } catch (Exception e) {
                // If the app died trying to launch the receiver we declare it 'bad'
                Slog.wtf(TAG, "Exception thrown dispatching broadcasts in " + app, e);
                badApp = true;
            }
        }

        // Check whether the next backup agent is in this process...
        if (!badApp && mBackupTarget != null && mBackupTarget.app == app) {
            if (DEBUG_BACKUP) Slog.v(TAG_BACKUP,
                    "New app is backup target, launching agent for " + app);
            notifyPackageUse(mBackupTarget.appInfo.packageName,
                             PackageManager.NOTIFY_PACKAGE_USE_BACKUP);
            try {
                thread.scheduleCreateBackupAgent(mBackupTarget.appInfo,
                        compatibilityInfoForPackageLocked(mBackupTarget.appInfo),
                        mBackupTarget.backupMode);
            } catch (Exception e) {
                Slog.wtf(TAG, "Exception thrown creating backup agent in " + app, e);
                badApp = true;
            }
        }

        if (badApp) {
            app.kill("error during init", true);
            handleAppDiedLocked(app, false, true);
            return false;
        }

        if (!didSomething) {
            updateOomAdjLocked();
            checkTime(startTime, "attachApplicationLocked: after updateOomAdjLocked");
        }

        return true;
    }
```

``` java

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
   void ensureActivitiesVisibleLocked(ActivityRecord starting, int configChanges,
            boolean preserveWindows) {
        ensureActivitiesVisibleLocked(starting, configChanges, preserveWindows,
                true /* notifyClients */);
    }

    /**
     * @see #ensureActivitiesVisibleLocked(ActivityRecord, int, boolean)
     */
    void ensureActivitiesVisibleLocked(ActivityRecord starting, int configChanges,
            boolean preserveWindows, boolean notifyClients) {
        getKeyguardController().beginActivityVisibilityUpdate();
        try {
            // First the front stacks. In case any are not fullscreen and are in front of home.
            for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
                final ActivityDisplay display = mActivityDisplays.valueAt(displayNdx);
                for (int stackNdx = display.getChildCount() - 1; stackNdx >= 0; --stackNdx) {
                    final ActivityStack stack = display.getChildAt(stackNdx);
                    stack.ensureActivitiesVisibleLocked(starting, configChanges, preserveWindows,
                            notifyClients);
                }
            }
        } finally {
            getKeyguardController().endActivityVisibilityUpdate();
        }
    }

android9.0/frameworks/base/services/core/java/com/android/server/am/ActivityStack.java

 final void ensureActivitiesVisibleLocked(ActivityRecord starting, int configChanges,
            boolean preserveWindows, boolean notifyClients) {
        mTopActivityOccludesKeyguard = false;
        mTopDismissingKeyguardActivity = null;
        mStackSupervisor.getKeyguardController().beginActivityVisibilityUpdate();
        try {
            ActivityRecord top = topRunningActivityLocked();
----------------------------------------------
对应Log：
04-16 13:08:36.840 V/ActivityStack_Visibility( 1049): ensureActivitiesVisible behind ActivityRecord{65cc87a u0 com.android.testred/.TestActivity t5} configChanges=0x0
04-16 13:08:36.840 V/ActivityStack_Visibility( 1049): Make visible? ActivityRecord{65cc87a u0 com.android.testred/.TestActivity t5} finishing=false state=INITIALIZING
04-16 13:08:36.840 V/ActivityStack_Visibility( 1049): Skipping: already visible at ActivityRecord{65cc87a u0 com.android.testred/.TestActivity t5}
----------------------------------------------
            if (DEBUG_VISIBILITY) Slog.v(TAG_VISIBILITY, "ensureActivitiesVisible behind " + top
                    + " configChanges=0x" + Integer.toHexString(configChanges));
            if (top != null) {
                checkTranslucentActivityWaiting(top);
            }

            // If the top activity is not fullscreen, then we need to
            // make sure any activities under it are now visible.
            boolean aboveTop = top != null;
            final boolean stackShouldBeVisible = shouldBeVisible(starting);
            boolean behindFullscreenActivity = !stackShouldBeVisible;
            boolean resumeNextActivity = mStackSupervisor.isFocusedStack(this)
                    && (isInStackLocked(starting) == null);
            final boolean isTopNotPinnedStack =
                    isAttached() && getDisplay().isTopNotPinnedStack(this);
            for (int taskNdx = mTaskHistory.size() - 1; taskNdx >= 0; --taskNdx) {
                final TaskRecord task = mTaskHistory.get(taskNdx);
                final ArrayList<ActivityRecord> activities = task.mActivities;
                for (int activityNdx = activities.size() - 1; activityNdx >= 0; --activityNdx) {
                    final ActivityRecord r = activities.get(activityNdx);
                    if (r.finishing) {
                        continue;
                    }
                    final boolean isTop = r == top;
                    if (aboveTop && !isTop) {
                        continue;
                    }
                    aboveTop = false;

                    // Check whether activity should be visible without Keyguard influence
                    final boolean visibleIgnoringKeyguard = r.shouldBeVisibleIgnoringKeyguard(
                            behindFullscreenActivity);
                    r.visibleIgnoringKeyguard = visibleIgnoringKeyguard;

                    // Now check whether it's really visible depending on Keyguard state.
                    final boolean reallyVisible = checkKeyguardVisibility(r,
                            visibleIgnoringKeyguard, isTop && isTopNotPinnedStack);
                    if (visibleIgnoringKeyguard) {
                        behindFullscreenActivity = updateBehindFullscreen(!stackShouldBeVisible,
                                behindFullscreenActivity, r);
                    }
                    if (reallyVisible) {
                        if (DEBUG_VISIBILITY) Slog.v(TAG_VISIBILITY, "Make visible? " + r
                                + " finishing=" + r.finishing + " state=" + r.getState());
                        // First: if this is not the current activity being started, make
                        // sure it matches the current configuration.
                        if (r != starting && notifyClients) {
                            r.ensureActivityConfiguration(0 /* globalChanges */, preserveWindows,
                                    true /* ignoreStopState */);
                        }

                        if (r.app == null || r.app.thread == null) {
                            if (makeVisibleAndRestartIfNeeded(starting, configChanges, isTop,
                                    resumeNextActivity, r)) {
                                if (activityNdx >= activities.size()) {
                                    // Record may be removed if its process needs to restart.
                                    activityNdx = activities.size() - 1;
                                } else {
                                    resumeNextActivity = false;
                                }
                            }
                        } else if (r.visible) {
                            // If this activity is already visible, then there is nothing to do here.
                            if (DEBUG_VISIBILITY) Slog.v(TAG_VISIBILITY,
                                    "Skipping: already visible at " + r);

                            if (r.mClientVisibilityDeferred && notifyClients) {
                                r.makeClientVisible();
                            }

                            if (r.handleAlreadyVisible()) {
                                resumeNextActivity = false;
                            }
                        } else {
                            r.makeVisibleIfNeeded(starting, notifyClients);
                        }
                        // Aggregate current change flags.
                        configChanges |= r.configChangeFlags;
                    } else {
                        if (DEBUG_VISIBILITY) Slog.v(TAG_VISIBILITY, "Make invisible? " + r
                                + " finishing=" + r.finishing + " state=" + r.getState()
                                + " stackShouldBeVisible=" + stackShouldBeVisible
                                + " behindFullscreenActivity=" + behindFullscreenActivity
                                + " mLaunchTaskBehind=" + r.mLaunchTaskBehind);

			//charlesvincent start
			if("com.android.testred".equals(r.packageName) 
			   || "com.android.testgreen".equals(r.packageName)
			   || "com.android.testblue".equals(r.packageName)){
				//do nothing
			}else{
                            makeInvisible(r);
			}
			//charlesvincent end
                    }
                }
                final int windowingMode = getWindowingMode();
                if (windowingMode == WINDOWING_MODE_FREEFORM) {
                    // The visibility of tasks and the activities they contain in freeform stack are
                    // determined individually unlike other stacks where the visibility or fullscreen
                    // status of an activity in a previous task affects other.
                    behindFullscreenActivity = !stackShouldBeVisible;
                } else if (isActivityTypeHome()) {
                    if (DEBUG_VISIBILITY) Slog.v(TAG_VISIBILITY, "Home task: at " + task
                            + " stackShouldBeVisible=" + stackShouldBeVisible
                            + " behindFullscreenActivity=" + behindFullscreenActivity);
                    // No other task in the home stack should be visible behind the home activity.
                    // Home activities is usually a translucent activity with the wallpaper behind
                    // them. However, when they don't have the wallpaper behind them, we want to
                    // show activities in the next application stack behind them vs. another
                    // task in the home stack like recents.
                    behindFullscreenActivity = true;
                }
            }

            if (mTranslucentActivityWaiting != null &&
                    mUndrawnActivitiesBelowTopTranslucent.isEmpty()) {
                // Nothing is getting drawn or everything was already visible, don't wait for timeout.
                notifyActivityDrawnLocked(null);
            }
        } finally {
            mStackSupervisor.getKeyguardController().endActivityVisibilityUpdate();
        }
    }
```
在ActivityManagerService.attachApplication方法中经过多次跳转执行到ActivityStackSupervisor.realStartActivityLocked方法。看到这个方法有没有一点眼熟？没错，就是之前分析过程中遇到的如果应用进程已经启动的情况下去启动Activity所调用的方法，有点绕，自己体会一下。在ActivityStackSupervisor.realStartActivityLocked方法中为ClientTransaction对象添加LaunchActivityItem的callback，然后设置当前的生命周期状态，最后调用ClientLifecycleManager.scheduleTransaction方法执行。

####  六、执行启动Acitivity

##### 6.1、ActivityStackSupervisor.realStartActivityLocked() 

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
------------------------------
对应Log：

    void setVisibility(boolean visible) {
        mWindowContainerController.setVisibility(visible, mDeferHidingClient);
        mStackSupervisor.getActivityMetricsLogger().notifyVisibilityChanged(this);
    }
04-16 13:08:36.850 V/WindowManager( 1049): setAppVisibility(Token{ce2ac07 ActivityRecord{65cc87a u0 com.android.testred/.TestActivity t5}}, visible=true): mNextAppTransition=TRANSIT_TASK_OPEN hidden=true hiddenRequested=false Callers=com.android.server.am.ActivityRecord.setVisibility:1668 com.android.server.am.ActivityStackSupervisor.realStartActivityLocked:1433 com.android.server.am.ActivityStackSupervisor.attachApplicationLocked:996 com.android.server.am.ActivityManagerService.attachApplicationLocked:7977 com.android.server.am.ActivityManagerService.attachApplication:8045 android.app.IActivityManager$Stub.onTransact:198 
04-16 13:08:36.850 V/WindowManager( 1049): No longer Stopped: AppWindowToken{1579134 token=Token{ce2ac07 ActivityRecord{65cc87a u0 com.android.testred/.TestActivity t5}}}
------------------------------
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
-----------------------------------------------
对应Log：
04-16 13:08:36.850 V/ActivityStackSupervisor( 1049): Launching: ActivityRecord{65cc87a u0 com.android.testred/.TestActivity t5}
04-16 13:08:36.850 D/ActivityManagerService_LRU( 1049): Adding to top of LRU activity list: ProcessRecord{d768ca0 2602:com.android.testred/u0a88}
04-16 13:08:36.850 D/ActivityManagerService_OomAdj( 1049): Making top: ProcessRecord{d768ca0 2602:com.android.testred/u0a88}
-----------------------------------------------

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
------------------------------------------------
对应Log：
04-16 13:08:36.854 V/ActivityStackSupervisor_Switch( 1049): Launching: ActivityRecord{65cc87a u0 com.android.testred/.TestActivity t5} icicle=null with results=null newIntents=null andResume=true
------------------------------------------------
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
                // ResumeActivityItem
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
----------------------------------------------
对应Log：
    04-16 13:08:36.854 V/ActivityStack_States( 1049): Moving to RESUMED: ActivityRecord{65cc87a u0 com.android.testred/.TestActivity t5} (starting new instance) callers=com.android.server.am.ActivityStackSupervisor.realStartActivityLocked:1609 com.android.server.am.ActivityStackSupervisor.attachApplicationLocked:996 com.android.server.am.ActivityManagerService.attachApplicationLocked:7977 com.android.server.am.ActivityManagerService.attachApplication:8045 android.app.IActivityManager$Stub.onTransact:198 
04-16 13:08:36.854 V/ActivityRecord_States( 1049): State movement: ActivityRecord{65cc87a u0 com.android.testred/.TestActivity t5} from:INITIALIZING to:RESUMED reason:minimalResumeActivityLocked
04-16 13:08:36.854 V/ActivityStack_Stack( 1049): set resumed activity to:ActivityRecord{65cc87a u0 com.android.testred/.TestActivity t5} reason:minimalResumeActivityLocked
04-16 13:08:36.855 D/ActivityStack_Stack( 1049): setResumedActivity stack:ActivityStack{1b7b488 stackId=1 type=standard mode=fullscreen visible=true translucent=true, 1 tasks} + from: null to:ActivityRecord{65cc87a u0 com.android.testred/.TestActivity t5} reason:minimalResumeActivityLocked - onActivityStateChanged
----------------------------------------------
            stack.minimalResumeActivityLocked(r);
----------------------------------------------
看下 stack.minimalResumeActivityLocked(r)
最终调用到android9.0/frameworks/base/services/core/java/com/android/server/am/ActivityStack.java

    void onActivityStateChanged(ActivityRecord record, ActivityState state, String reason) {
        if (record == mResumedActivity && state != RESUMED) {
            setResumedActivity(null, reason + " - onActivityStateChanged");
        }

        if (state == RESUMED) {
            if (DEBUG_STACK) Slog.v(TAG_STACK, "set resumed activity to:" + record + " reason:"
                    + reason);
            setResumedActivity(record, reason + " - onActivityStateChanged");
            
 //mWindowManager.setFocusedApp(r.appToken, true);
   ----------------------------------------------
对应Log：
04-16 13:08:36.855 V/WindowManager( 1049): Set focused app to: AppWindowToken{1579134 token=Token{ce2ac07 ActivityRecord{65cc87a u0 com.android.testred/.TestActivity t5}}} old focus=AppWindowToken{483acfb token=Token{f984e8a ActivityRecord{5e34200 u0 com.android.launcher3/.Launcher t4}}} moveFocusNow=true
----------------------------------------------
       mService.setResumedActivityUncheckLocked(record, reason);
            //将TaskRecord加入到mRecentTasks
            mStackSupervisor.mRecentTasks.add(record.getTask());
        }
    }
----------------------------------------------
        }
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



##### 6.2、ActivityThread.handleLaunchActivity()
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


##### 6.3、ActivityThread.performLaunchActivity()
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

```
我们来看一下Activity 的生命周期：

> protected void onCreate(); 
> protected void onStart(); 
> protected void onResume(); 
> protected void onPause(); 
> protected void onStop();
> protected void onDestory();

```


前面我们分析了onCreate()、onStart() 、onResume()、onPause()。 Activity 暂停销毁时的 、onStop() 、onDestroy() 回调都与前面的过程大同小异，这里就只列举相应的方法栈，不再继续描述。

####  七、参考文档

[（1）【Android Display System】](http://charlesvincent.cc/)
[（2）【Android 9.0 activity启动源码分析】](https://www.jianshu.com/p/50e4b48ed15c)
[（3）【Android 9.0 Activity启动流程源码分析】](https://blog.csdn.net/lj19851227/article/details/82562115)
[（4）【Android App("com.android.testgreen")源码】](https://gitlab.com/zhoujinjiana/apq8096_la.um.7.5.r1-03100-8x96.0_p_v5.0/commit/714c0071b4bc786d00e7a6bcd08face6a28d6c0c)
[（5）【理解Android进程创建流程】](http://gityuan.com/2016/03/26/app-process-create/)

