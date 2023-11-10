---
title: Android 11 Display System源码分析（8）：WMS之Window显示流程分析（V1）
cover: https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/post.cover.pictures/bing-wallpaper-2018.04.48.jpg
categories: 
 - Display
tags:
 - Display
toc: true
abbrlink: 20230216
date: 2023-02-16 16:16:16
---
注：文章都是通过阅读各位前辈总结的资料、Android 11 Rockchip平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除（◕‿◕），转载请注明出处（©Rockchip ©Android @Linux 版权所有），谢谢。

（==**文章基于 Android 11.0**==）

[【zhoujinjianin.com博客原图链接】](https://github.com/zhoujinjianin) 

[【开发板】](https://wiki.radxa.com/Rockpi4)

[【开发板 源码链接】](https://wiki.radxa.com/Rockpi4/rockpi-android11)



正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢，由于不喜欢图片水印，去除了水印，敬请谅解！！！

[zhoujinjianin.com](http://zhoujinjianin.com) 



--------------------------------------------------------------------------------

==源码（部分）==：

**xxx**

==源码（部分）==：

--------------------------------------------------------------------------------



## （一）、Window添加过程(App->WMS)

### （1）、Activity.setContentView()

![image-20221018100735157](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221018100735157.png)

![image-20221018101033339](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221018101033339.png)

![image-20221018101232003](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221018101232003.png)

### （2）、ViewRootImpl.init()

![image-20221019105057729](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019105057729.png)

![image-20221019105133929](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019105133929.png)

![image-20221019105726410](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019105726410.png)

![image-20221019110231231](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019110231231.png)

### （3）、ViewRootImpl.setView()![image-20221019110540837](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019110540837.png)

### （4）、WindowManagerGlobal.getWindowSession().addToDisplayAsUser()

![image-20221019110737565](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019110737565.png)

### （5）、WindowManagerService.addWindow()

![image-20221019114118714](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019114118714.png)

![image-20221019114231258](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019114231258.png)

![image-20221019114458689](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019114458689.png)

### （6）、WindowState.attach()

![image-20221019132712267](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019132712267.png)

![image-20221019132734312](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019132734312.png)

![image-20221019132815974](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019132815974.png)

![image-20221019132917859](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019132917859.png)

### （7）、WindowState.assignLayer()

![image-20221019133806222](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019133806222.png)

![image-20221019133847289](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019133847289.png)

![image-20221019134111716](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019134111716.png)

![image-20221019134214399](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019134214399.png)

![image-20221019134248157](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019134248157.png)

![image-20221019134402313](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019134402313.png)

![image-20221019134551758](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019134551758.png)

## （二）、App测量布局与绘制

### （1）、ViewRootImpl.relayoutWindow()

![image-20221019140312543](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019140312543.png)

![image-20221019140445013](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019140445013.png)

```java
10-19 11:32:42.728  1758  1758 V ViewRootImpl[]: relayout: frame=[0,0][1920,952] cutout=DisplayCutout{insets=Rect(0, 0 - 0, 0) waterfall=Insets{left=0, top=0, right=0, bottom=0} boundingRect={Bounds=[Rect(0, 0 - 0, 0), Rect(0, 0 - 0, 0), Rect(0, 0 - 0, 0), Rect(0, 0 - 0, 0)]}} surface=Surface(name=null)/@0x86fcceb
10-19 11:32:42.728  1758  1758 V ViewRootImpl: Relayout returned: frame=Rect(0, 0 - 1920, 952), surface=Surface(name=null)/@0x86fcceb
10-19 11:32:44.076   947  1162 V WindowManager: Relayout Window{6a1de98 u10 com.chinatsp.launcher/com.chinatsp.launcher.view.Launcher}: viewVisibility=0 req=1920x1080 {(0,0)(fillxfill) sim={adjust=nothing forwardNavigation} ty=BASE_APPLICATION wanim=0x1030301
10-19 11:32:44.076   947  1162 W WindowManager: setLayoutNeeded: callers=com.android.server.wm.WindowState.setDisplayLayoutNeeded:2576 com.android.server.wm.WindowManagerService.relayoutWindow:2258 com.android.server.wm.Session.relayout:213 
10-19 11:32:44.076   947  1162 W WindowManager: setLayoutNeeded: callers=com.android.server.wm.WindowState.setDisplayLayoutNeeded:2576 com.android.server.wm.WindowManagerService.relayoutWindow:2258 com.android.server.wm.Session.relayout:213 
10-19 11:32:44.076   947  1162 V RootWindowContainer: performSurfacePlacementInner: entry. Called by com.android.server.wm.RootWindowContainer.performSurfacePlacement:814 com.android.server.wm.WindowSurfacePlacer.performSurfacePlacementLoop:178 com.android.server.wm.WindowSurfacePlacer.performSurfacePlacement:127 com.android.server.wm.WindowManagerService.relayoutWindow:2289 com.android.server.wm.Session.relayout:213 
10-19 11:32:44.076   947  1162 V RootWindowContainer: performSurfacePlacementInner: entry. Called by com.android.server.wm.RootWindowContainer.performSurfacePlacement:814 com.android.server.wm.WindowSurfacePlacer.performSurfacePlacementLoop:178 com.android.server.wm.WindowSurfacePlacer.performSurfacePlacement:127 com.android.server.wm.WindowManagerService.relayoutWindow:2289 com.android.server.wm.Session.relayout:213 
    
    
10-19 11:32:44.079   947  1162 V DisplayPolicy: Compute frame com.chinatsp.launcher/com.chinatsp.launcher.view.Launcher: sim=#130 attach=null type=1 flags=0x81110100 pf=[0,0][1920,1080] df=[0,0][1920,1080] cf=[0,0][1920,1080] vf=[0,0][1920,1080] dcf=[0,0][1920,1080] sf=[0,116][1920,952]
10-19 11:32:44.079   947  1162 I WindowState: com.chinatsp.launcher
10-19 11:32:44.079   947  1162 I WindowState: java.lang.RuntimeException: here
10-19 11:32:44.079   947  1162 I WindowState: 	at com.android.server.wm.WindowState.computeFrameLw(WindowState.java:1267)
10-19 11:32:44.079   947  1162 I WindowState: 	at com.android.server.wm.WindowState.computeFrame(WindowState.java:1048)
10-19 11:32:44.079   947  1162 I WindowState: 	at com.android.server.wm.DisplayPolicy.layoutWindowLw(DisplayPolicy.java:2695)
10-19 11:32:44.079   947  1162 I WindowState: 	at com.android.server.wm.DisplayContent.lambda$new$4$DisplayContent(DisplayContent.java:755)
10-19 11:32:44.079   947  1162 I WindowState: 	at com.android.server.wm.-$$Lambda$DisplayContent$qT01Aq6xt_ZOs86A1yDQe-qmPFQ.accept(Unknown Source:4)
10-19 11:32:44.079   947  1162 I WindowState: 	at com.android.server.wm.WindowContainer$ForAllWindowsConsumerWrapper.apply(WindowContainer.java:1984)
10-19 11:32:44.079   947  1162 I WindowState: 	at com.android.server.wm.WindowContainer$ForAllWindowsConsumerWrapper.apply(WindowContainer.java:1974)
10-19 11:32:44.079   947  1162 I WindowState: 	at com.android.server.wm.WindowState.applyInOrderWithImeWindows(WindowState.java:4668)
10-19 11:32:44.079   947  1162 I WindowState: 	at com.android.server.wm.WindowState.forAllWindows(WindowState.java:4567)
10-19 11:32:44.079   947  1162 I WindowState: 	at com.android.server.wm.WindowContainer.forAllWindows(WindowContainer.java:1302)
10-19 11:32:44.079   947  1162 I WindowState: 	at com.android.server.wm.ActivityRecord.forAllWindowsUnchecked(ActivityRecord.java:3638)
10-19 11:32:44.079   947  1162 I WindowState: 	at com.android.server.wm.ActivityRecord.forAllWindows(ActivityRecord.java:3633)
10-19 11:32:44.079   947  1162 I WindowState: 	at com.android.server.wm.WindowContainer.forAllWindows(WindowContainer.java:1302)
10-19 11:32:44.079   947  1162 I WindowState: 	at com.android.server.wm.WindowContainer.forAllWindows(WindowContainer.java:1302)
10-19 11:32:44.079   947  1162 I WindowState: 	at com.android.server.wm.WindowContainer.forAllWindows(WindowContainer.java:1302)
10-19 11:32:44.079   947  1162 I WindowState: 	at com.android.server.wm.TaskDisplayArea.forAllWindows(TaskDisplayArea.java:488)
10-19 11:32:44.079   947  1162 I WindowState: 	at com.android.server.wm.WindowContainer.forAllWindows(WindowContainer.java:1302)
10-19 11:32:44.079   947  1162 I WindowState: 	at com.android.server.wm.WindowContainer.forAllWindows(WindowContainer.java:1302)
10-19 11:32:44.079   947  1162 I WindowState: 	at com.android.server.wm.WindowContainer.forAllWindows(WindowContainer.java:1302)
10-19 11:32:44.079   947  1162 I WindowState: 	at com.android.server.wm.WindowContainer.forAllWindows(WindowContainer.java:1319)
10-19 11:32:44.079   947  1162 I WindowState: 	at com.android.server.wm.DisplayContent.performLayoutNoTrace(DisplayContent.java:4136)
10-19 11:32:44.079   947  1162 I WindowState: 	at com.android.server.wm.DisplayContent.performLayout(DisplayContent.java:4096)
10-19 11:32:44.079   947  1162 I WindowState: 	at com.android.server.wm.DisplayContent.applySurfaceChangesTransaction(DisplayContent.java:3989)
10-19 11:32:44.079   947  1162 I WindowState: 	at com.android.server.wm.RootWindowContainer.applySurfaceChangesTransaction(RootWindowContainer.java:1077)
10-19 11:32:44.079   947  1162 I WindowState: 	at com.android.server.wm.RootWindowContainer.performSurfacePlacementNoTrace(RootWindowContainer.java:857)
10-19 11:32:44.079   947  1162 I WindowState: 	at com.android.server.wm.RootWindowContainer.performSurfacePlacement(RootWindowContainer.java:814)
10-19 11:32:44.079   947  1162 I WindowState: 	at com.android.server.wm.WindowSurfacePlacer.performSurfacePlacementLoop(WindowSurfacePlacer.java:178)
10-19 11:32:44.079   947  1162 I WindowState: 	at com.android.server.wm.WindowSurfacePlacer.performSurfacePlacement(WindowSurfacePlacer.java:127)
10-19 11:32:44.079   947  1162 I WindowState: 	at com.android.server.wm.WindowManagerService.relayoutWindow(WindowManagerService.java:2289)
10-19 11:32:44.079   947  1162 I WindowState: 	at com.android.server.wm.Session.relayout(Session.java:213)
10-19 11:32:44.079   947  1162 I WindowState: 	at android.view.IWindowSession$Stub.onTransact(IWindowSession.java:840)
10-19 11:32:44.079   947  1162 I WindowState: 	at com.android.server.wm.Session.onTransact(Session.java:139)
10-19 11:32:44.079   947  1162 I WindowState: 	at android.os.Binder.execTransactInternal(Binder.java:1159)
10-19 11:32:44.079   947  1162 I WindowState: 	at android.os.Binder.execTransact(Binder.java:1123)
10-19 11:32:44.079   947  1162 V WindowState: Resolving (mRequestedWidth=1920, mRequestedheight=1080) to (pw=1920, ph=1080): frame=[0,0][1920,1080] ci=[0,0][0,0] vi=[0,0][0,0] si=[0,116][0,128] com.chinatsp.launcher/com.chinatsp.launcher.view.Launcher
    
```

### （2）、DisplayPolicy.layoutWindowLw()

WMS决定App最终显示大小。

![image-20221019141032821](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019141032821.png)

![image-20221019141135178](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019141135178.png)

![image-20221019141308487](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019141308487.png)

### （3）、WindowManagerService.createSurfaceControl()

![image-20221019141626551](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019141626551.png)

![image-20221019141657173](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019141657173.png)

![image-20221019141724115](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019141724115.png)

![image-20221019141758342](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019141758342.png)

![image-20221019141828980](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019141828980.png)

![image-20221019141850769](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019141850769.png)

### （4）、WindowState.prepareSurfaces()

![image-20221019142109914](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019142109914.png)

WindowAnimator属于WMS的。

![image-20221019142334704](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019142334704.png)

![image-20221019142356357](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019142356357.png)

![image-20221019142441710](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019142441710.png)

此时Launcher还未draw完成，未显示。Surface已经准备好。接下来由App测量布局与绘制了。

![image-20221019142701745](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019142701745.png)

### （5）、ViewRootImpl .measureHierarchy()

![image-20221019134934965](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019134934965.png)

![image-20221019135046614](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019135046614.png)

![image-20221019135218716](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019135218716.png)

![image-20221019135351909](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019135351909.png)

![image-20221019135329477](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019135329477.png)

### （6）、ViewRootImpl.performLayout()

![image-20221019135717819](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019135717819.png)

![image-20221019135806495](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019135806495.png)

### （7）、ViewRootImpl.draw()

![image-20221019135919097](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019135919097.png)

底层绘制完成后会回调。

![image-20221019172235905](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019172235905.png)

![image-20221019172404357](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019172404357.png)

![image-20221019170436766](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019170436766.png)

![image-20221019200539561](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019200539561.png)

![image-20221019200621815](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019200621815.png)

![image-20221019165353333](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019165353333.png)

```java
10-19 11:32:44.283  3063  3063 V ViewRootImpl[Launcher]: FINISHED DRAWING: com.chinatsp.launcher/com.chinatsp.launcher.view.Launcher
```

![image-20221019170806109](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019170806109.png)

![image-20221019170957393](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019170957393.png)

![image-20221019171411611](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019171411611.png)

## （三）、WMS显示

### （1）、RootWindowContainer.applySurfaceChangesTransaction()

![image-20221019155136333](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019155136333.png)

![image-20221019161014403](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019161014403.png)



![image-20221019162725964](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019162725964.png)

![image-20221019162908942](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019162908942.png)

![image-20221019163043022](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019163043022.png)

![image-20221019163412145](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019163412145.png)

![image-20221019163458963](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019163458963.png)

![image-20221019163833359](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/Android_Display_System/Android11_Display08/image-20221019163833359.png)

## （四）、参考资料(特别感谢)：

 [（1）xxx](xxx) 