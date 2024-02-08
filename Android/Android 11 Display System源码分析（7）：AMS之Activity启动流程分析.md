---
title: Android 11 Display System源码分析（7）：AMS之Activity启动流程分析（V1）
cover: https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/post.cover.pictures/bing-wallpaper-2018.04.47.jpg
categories: 
 - Display
tags:
 - Display
toc: true
abbrlink: 20221216
date: 2022-12-16 16:16:16
---
注：文章都是通过阅读各位前辈总结的资料、Android 11 Rockchip平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除（◕‿◕），转载请注明出处（©Rockchip ©Android @Linux 版权所有），谢谢。

（==**文章基于 Android 11.0**==）

[【zhoujinjian.com博客原图链接】](https://github.com/zjjzhoujinjian) 

[【开发板】](https://wiki.radxa.com/Rockpi4)

[【开发板 源码链接】](https://wiki.radxa.com/Rockpi4/rockpi-android11)



正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢，由于不喜欢图片水印，去除了水印，敬请谅解！！！

[zhoujinjian.com](http://zhoujinjian.com) 



--------------------------------------------------------------------------------

==源码（部分）==：

**xxx**

==源码（部分）==：

--------------------------------------------------------------------------------

我们这里分析以Launcher为实例。

## （一）、FallbackHome到Launcher

![image-20221011104643904](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011104643904.png)

### （1）、FallbackHome到Launcher

系统开机首先启动FallbackHome，FallbackHome其实是一个优先级更高的Luancher，然后其实没做什么事情，后面就finish了。

![image-20221011104835086](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011104835086.png)

![image-20221011104959978](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011104959978.png)

FallbackHome生命周期走finish后，系统会去resume 下一个top activity。此时系统没有其他Activity启动。就会回到桌面。

![image-20221011105203171](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011105203171.png)

通过PackageManager().resolveIntent(）寻找合适的launcher（存在多launcher）的ActivityInfo。

![image-20221011105239435](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011105239435.png)

![image-20221011110050571](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011110050571.png)

![image-20221011110340325](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011110340325.png)

寻找Home ActivityInfo。

![image-20221011105350290](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011105350290.png)

这里找到了Launcher包名（com.chinatsp.launcher）。

![image-20221011105620408](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011105620408.png)

### （2）、ActivityStartController().startHomeActivity()

![image-20221011110522580](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011110522580.png)

这里就开始启动Launcher之旅了。

```java
01-01 08:00:14.976   879  1846 I ActivityStarter: START u10 {act=android.intent.action.MAIN cat=[android.intent.category.HOME] flg=0x10000100 cmp=com.chinatsp.launcher/.view.Launcher} from uid 0

```

### （3）、ActivityStarter.startHomeActivity()

![image-20221011110751929](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011110751929.png)

看看堆栈信息。

![image-20221011110934838](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011110934838.png)

系统会首先为Luancher创建一个ActivityRecord。这个时候framework里面的生命周期从from:null to:INITIALIZING。

```java
01-01 08:00:14.978   879  1846 V ActivityRecord_States: State movement: ActivityRecord{1d62093 u10 com.chinatsp.launcher/.view.Launcher from:null to:INITIALIZING reason:ActivityRecord ctor
```

![image-20221011111058491](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011111058491.png)

### （4）、ActivityStarter.startActivityUnchecked()

![image-20221011111325206](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011111325206.png)

![image-20221011111419993](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011111419993.png)

看看堆栈先：

![image-20221011111550667](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011111550667.png)

startActivityInner()首先会去寻找是否存在可以重复使用的Task（AMS的Task），我们这里Launcher第一次启动会新建Task。

### （5）、ActivityStarter.getReusableTask()

![image-20221011111800677](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011111800677.png)

未找到可Reuse的Task。

![image-20221011112111691](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011112111691.png)

### （6）、ActivityStarter.getLaunchStack()

getLaunchStack会为Luancher创建ActivityStack（如果不存在的话）。

![image-20221011113416697](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011113416697.png)

![image-20221011113508151](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011113508151.png)

![image-20221011112658712](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011112658712.png)

![image-20221011112739019](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011112739019.png)

![image-20221011112957814](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011112957814.png)

![image-20221011112924614](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011112924614.png)

这里创建好了ActivityStack，但不是对于Launcher的。

### （7）、ActivityStarter.setNewTask()

![image-20221011112431644](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011112431644.png)

![image-20221011112508287](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011112508287.png)

![image-20221011113741587](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011113741587.png)

>  ActivityStack extends Task ，ActivityStack 继承于Task ，new ActivityStack会走Task初始化方法。

![image-20221011114029542](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011114029542.png)

这里Task也创建好了。看看堆栈。

![image-20221011114211041](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011114211041.png)

### （8）、ActivityStack.startActivityLocked()

![image-20221011114427994](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011114427994.png)

![image-20221011114544106](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011114544106.png)

![image-20221011133747889](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011133747889.png)

![image-20221011133914558](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011133914558.png)

![image-20221011133929185](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011133929185.png)

![image-20221011140137515](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011140137515.png)

## （二）、Launcher进程创建

### （1）、ActivityTaskManagerService.startProcessAsync()

## ![image-20221011140325794](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011140325794.png)

![image-20221011140423965](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011140423965.png)

进程创建好，首先会回调ActivityThread.main()方法（具体如何回调这里不做分析）。

### （2）、ActivityThread.main()

![image-20221011140843178](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011140843178.png)

![image-20221011140908486](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011140908486.png)

Binder通信进入System_Sever进程。

![image-20221011141058283](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011141058283.png)

### （3）、AMS.attachApplicationLocked()

![image-20221011141151943](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011141151943.png)

#### ![image-20221011142419407](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011142419407.png)

![image-20221011142541709](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011142541709.png)

![image-20221011143005218](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011143005218.png)

![image-20221011142755104](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011142755104.png)

前面都是创建进程和一系列资源绑定。

```java
10-11 14:20:26.699  2692  2692 V ActivityThread: <<< done: BIND_APPLICATION
```



## （三）、Launcher生命周期

### （1）、ActivityStackSupervisor.realStartActivityLocked()

![image-20221011144017919](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011144017919.png)

### （2）、ActivityThread.handleLaunchActivity()

![image-20221011144326968](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011144326968.png)

![image-20221011154115394](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011154115394.png)

这里生命周期会从onCreate() 到 onResume()

### （3）、ActivityThread.performLaunchActivity()

![image-20221011144426651](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011144426651.png)

![image-20221011144530621](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011144530621.png)

### （4）、Activity.attach()

![image-20221011144640509](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011144640509.png)

### （5）、Activity.performCreate()

![image-20221011144933462](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011144933462.png)

![image-20221011145032477](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011145032477.png)

### （6）、Activity.onCreate()

![image-20221011145111546](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011145111546.png)

### （7）、Activity.handleStartActivity()

![image-20221011145735815](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011145735815.png)

![image-20221011152946359](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011152946359.png)



![image-20221011153043498](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011153043498.png)

### （8）、Activity.performStart()

![image-20221011153154349](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011153154349.png)

### （9）、Activity.onStart()

![image-20221011162230472](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011162230472.png)

### （10）、ActivityThread.handleResumeActivity()

![image-20221011154319419](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011154319419.png)

![image-20221011154426393](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011154426393.png)

![image-20221011154447360](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011154447360.png)

### （11）、Activity.performResume()

![image-20221011154522347](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011154522347.png)

![image-20221011154613555](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011154613555.png)

### （12）、Activity.onResume()

![image-20221011162336393](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011162336393.png)

### （13）、ActivityThread.handleTopResumedActivityChanged()

![image-20221011155531560](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011155531560.png)

![image-20221011155052203](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011155052203.png)

![image-20221011155620942](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011155620942.png)

当启动其他进程Activity时，收降Luancher pause；

### （14）、ActivityThread.handlePauseActivity()

![image-20221011193418140](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011193418140.png)

![image-20221011193451022](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011193451022.png)

![image-20221011160026341](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011160026341.png)

![image-20221011160525590](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011160525590.png)

![image-20221011160743678](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011160743678.png)

![image-20221011160806358](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011160806358.png)

![image-20221011193659276](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011193659276.png)

### （15）、ActivityThread.onPause()

![image-20221011160902219](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011160902219.png)

![image-20221011162543535](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011162543535.png)

### （16）、ActivityThread.handlePauseActivity()

![image-20221011163931198](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011163931198.png)

![image-20221011164006731](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011164006731.png)

### （17）、Activity.onPause()

![image-20221011164052114](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011164052114.png)

### （18）、Activity.handleStopActivity()

![image-20221011193915376](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011193915376.png)

![image-20221011193938949](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011193938949.png)

![image-20221011165936039](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011165936039.png)

![image-20221011170024978](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011170024978.png)

### （19）、Activity.onStop()

![image-20221011170114143](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/Android_Display_System/Android11_Display07/image-20221011170114143.png)

## （四）、参考资料(特别感谢)：

 [（1）xxx](xxx) 