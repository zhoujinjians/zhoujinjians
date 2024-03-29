---
title: Android 11 Input System源码分析（1）：Input流程分析
cover: https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/post.cover.pictures/bing-wallpaper-2018.04.49.jpg
categories: 
 - Display
tags:
 - Display
toc: true
abbrlink: 20230316
date: 2023-03-16 16:16:16
---
注：文章都是通过阅读各位前辈总结的资料、Android 11 Rockchip平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除（◕‿◕），转载请注明出处（©Rockchip ©Android @Linux 版权所有），谢谢。

（==**文章基于 Android 11.0**==）

[【zhoujinjian.com博客原图链接】](https://github.com/zhoujinjianzz) 

[【开发板】](https://wiki.radxa.com/Rockpi4)

[【开发板 源码链接】](https://wiki.radxa.com/Rockpi4/rockpi-android11)



正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢，由于不喜欢图片水印，去除了水印，敬请谅解！！！

[zhoujinjian.com](http://zhoujinjian.com) 



--------------------------------------------------------------------------------

==源码（部分）==：

**xxx**

==源码（部分）==：

--------------------------------------------------------------------------------

## （一）、InputManagerService初始化

![image-20221027094950105](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027094950105.png)

![image-20221027095512362](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027095512362.png)

![image-20221027095548641](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027095548641.png)

![image-20221027095626369](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027095626369.png)

![image-20221027095652842](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027095652842.png)

### （1）、InputDispatcher初始化

![image-20221027095740500](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027095740500.png)



### （2）、InputReader初始化

![image-20221027095844947](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027095844947.png)

### （3）、EventHub初始化

![image-20221027100319791](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027100319791.png)

### （4）、InputManagerService.start()

![image-20221027100656839](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027100656839.png)

![image-20221027100836913](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027100836913.png)

![image-20221027100858489](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027100858489.png)

![image-20221027100926276](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027100926276.png)

### （5）、InputDispatcher.start()

![image-20221027101106916](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027101106916.png)

![image-20221027113354714](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027113354714.png)

### （6）、InputReader.start()

![image-20221027101147654](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027101147654.png)

![image-20221027113531171](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027113531171.png)

### （7）、EventHub::getEvents()

![image-20221027132600483](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027132600483.png)

## （二）、InputChannel 服务端（System_Server）register

![image-20221026113350606](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221026113350606.png)

![image-20221026192540644](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221026192540644.png)

### （1）、WindowState.openInputChannel()

![image-20221026192704803](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221026192704803.png)

![image-20221026192752789](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221026192752789.png)

![image-20221026192815403](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221026192815403.png)

![image-20221026192926747](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221026192926747.png)

![image-20221026193014900](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221026193014900.png)

![image-20221027092859857](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027092859857.png)

### （2）、InputManagerService.registerInputChannel()

![image-20221027093126783](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027093126783.png)

![image-20221027093224596](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027093224596.png)

![image-20221027093252326](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027093252326.png)

![image-20221027093449771](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027093449771.png)

![image-20221027093715359](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027093715359.png)

![image-20221027093813544](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027093813544.png)



## （三）、InputChannel 客户端（App-Launcher）register

![image-20221027094005740](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027094005740.png)

![image-20221027094505483](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027094505483.png)

![image-20221027094532853](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027094532853.png)

![image-20221027094607077](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027094607077.png)

![image-20221027094321718](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027094321718.png)

## （四）、Input Motion Event事件 Server分发

![image-20221027143408175](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027143408175.png)

![image-20221027143519671](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027143519671.png)

### （1）、InputReader::processEventsLocked()

![image-20221027143653894](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027143653894.png)

![image-20221027143935200](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027143935200.png)

### （2）、InputDevice::process()

![image-20221027144139620](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027144139620.png)

### （3）、TouchInputMapper::processRawTouches()

![image-20221027144621694](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027144621694.png)

Log:

![image-20221027145357518](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027145357518.png)

### （4）、TouchInputMapper::processRawTouches()

![image-20221027145446258](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027145446258.png)

![image-20221027145533749](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027145533749.png)

### （5）、TouchInputMapper::dispatchTouches()

![image-20221027145639238](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027145639238.png)

![image-20221027145709304](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027145709304.png)

### （6）、InputDispatcher::notifyMotion()

![image-20221027150101292](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027150101292.png)

![image-20221027150214826](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027150214826.png)

### （7）、InputDispatcher::notifyMotion()

![image-20221027150650263](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027150650263.png)

![image-20221027150816563](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027150816563.png)

### （8）、InputDispatcher::dispatchOnceInnerLocked()

![image-20221027152145125](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027152145125.png)

![image-20221027152228877](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027152228877.png)

### （9）、InputDispatcher::dispatchEventLocked()

![image-20221027152405656](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027152405656.png)

![image-20221027152544526](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027152544526.png)

![image-20221027153049512](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027153049512.png)

![image-20221027153202922](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027153202922.png)

### （10）、InputDispatcher::startDispatchCycleLocked()

![image-20221027153340854](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027153340854.png)



### （11）、InputPublisher::publishMotionEvent()

![image-20221027153617457](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027153617457.png)

![image-20221027153754750](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027153754750.png)

![image-20221027153812565](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027153812565.png)

![image-20221027162237787](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027162237787.png)

## （五）、Input Motion Event事件 Client接收

### （1）、NativeInputEventReceiver::handleEvent()

![image-20221027162447073](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027162447073.png)

![image-20221027162400118](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027162400118.png)

### （2）、InputConsumer::consume()

![image-20221027162600337](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027162600337.png)

![image-20221027162710884](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027162710884.png)

![image-20221027162732280](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027162732280.png)

### （3）、NativeInputEventReceiver::consumeEvents()

![image-20221027162918422](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027162918422.png)

![image-20221027162830945](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027162830945.png)

### （4）、InputEventReceiver::dispatchInputEvent()

![image-20221027163134650](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027163134650.png)

![image-20221027163310911](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221027163310911.png)

### （5）、View.dispatchTouchEvent()

![image-20221028134050016](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221028134050016.png)

![image-20221028134140542](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221028134140542.png)

### （6）、MotionEvent { action=ACTION_DOWN...}

![image-20221028134316074](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221028134316074.png)

![image-20221028134408948](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221028134408948.png)

### （7）、MotionEvent { action=ACTION_UP...}

![image-20221028134532535](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221028134532535.png)

![image-20221028134556342](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/Android_Input_System/Android11_Input01/image-20221028134556342.png)

### （8）、Pointer 0: id=0, toolType=1, x=1294.325806, y=401.628113

```c++
	行 52806: 10-21 15:00:37.848   950  1094 D InputDispatcher:   Pointer 0: id=0, toolType=1, x=1294.325806, y=401.628113, pressure=1.000000, size=0.000000, touchMajor=0.000000, touchMinor=0.000000, toolMajor=0.000000, toolMinor=0.000000, orientation=0.000000
	行 52823: 10-21 15:00:37.850   950  1093 D InputDispatcher:   Pointer 0: id=0, toolType=1, x=1294.325806, y=401.628113, pressure=1.000000, size=0.000000, touchMajor=0.000000, touchMinor=0.000000, toolMajor=0.000000, toolMinor=0.000000, orientation=0.000000
	行 53173: 10-21 15:00:37.880   950  1094 D InputDispatcher:   Pointer 0: id=0, toolType=1, x=1294.325806, y=401.628113, pressure=1.000000, size=0.000000, touchMajor=0.000000, touchMinor=0.000000, toolMajor=0.000000, toolMinor=0.000000, orientation=0.000000
	行 53193: 10-21 15:00:37.882   950  1093 D InputDispatcher:   Pointer 0: id=0, toolType=1, x=1294.325806, y=401.628113, pressure=1.000000, size=0.000000, touchMajor=0.000000, touchMinor=0.000000, toolMajor=0.000000, toolMinor=0.000000, orientation=0.000000
	行 53532: 10-21 15:00:37.953  4443  4443 V ViewRootImpl[Launcher]: Done with EarlyPostImeInputStage. QueuedInputEvent{flags=0, hasNextQueuedEvent=true, hasInputEventReceiver=true, mEvent=MotionEvent { action=ACTION_DOWN, actionButton=0, id[0]=0, x[0]=1294.3258, y[0]=401.6281, toolType[0]=TOOL_TYPE_FINGER, buttonState=0, classification=NONE, metaState=0, flags=0x2, edgeFlags=0x0, pointerCount=1, historySize=0, eventTime=98717, downTime=98717, deviceId=-1, source=0x1002, displayId=0 }}
	行 53533: 10-21 15:00:37.954  4443  4443 V ViewRootImpl[Launcher]: Done with NativePostImeInputStage. QueuedInputEvent{flags=0, hasNextQueuedEvent=true, hasInputEventReceiver=true, mEvent=MotionEvent { action=ACTION_DOWN, actionButton=0, id[0]=0, x[0]=1294.3258, y[0]=401.6281, toolType[0]=TOOL_TYPE_FINGER, buttonState=0, classification=NONE, metaState=0, flags=0x2, edgeFlags=0x0, pointerCount=1, historySize=0, eventTime=98717, downTime=98717, deviceId=-1, source=0x1002, displayId=0 }}
	行 53535: 10-21 15:00:37.955  4443  4443 D HMI_Launcher: DragLayer: onInterceptTouchEvent: mDragging=false, ev=MotionEvent { action=ACTION_DOWN, actionButton=0, id[0]=0, x[0]=1294.3258, y[0]=401.6281, toolType[0]=TOOL_TYPE_FINGER, buttonState=0, classification=NONE, metaState=0, flags=0x2, edgeFlags=0x0, pointerCount=1, historySize=0, eventTime=98717, downTime=98717, deviceId=-1, source=0x1002, displayId=0 }
	行 53549: 10-21 15:00:37.955  4443  4443 D HMI_Launcher: Workspace: onInterceptTouchEvent: 1294.3258, 401.6281
	行 54150: 10-21 15:00:37.967  4443  4443 D ViewRootImpl: processPointerEvent src=0x1002 eventTimeNano=98717995000 id=0x3e5d3449 MotionEvent:MotionEvent { action=ACTION_DOWN, actionButton=0, id[0]=0, x[0]=1294.3258, y[0]=401.6281, toolType[0]=TOOL_TYPE_FINGER, buttonState=0, classification=NONE, metaState=0, flags=0x2, edgeFlags=0x0, pointerCount=1, historySize=0, eventTime=98717, downTime=98717, deviceId=-1, source=0x1002, displayId=0 } handled:true
	行 54178: 10-21 15:00:37.967  4443  4443 V ViewRootImpl[Launcher]: Done with ViewPostImeInputStage. QueuedInputEvent{flags=FINISHED|FINISHED_HANDLED, hasNextQueuedEvent=true, hasInputEventReceiver=true, mEvent=MotionEvent { action=ACTION_DOWN, actionButton=0, id[0]=0, x[0]=1294.3258, y[0]=401.6281, toolType[0]=TOOL_TYPE_FINGER, buttonState=0, classification=NONE, metaState=0, flags=0x2, edgeFlags=0x0, pointerCount=1, historySize=0, eventTime=98717, downTime=98717, deviceId=-1, source=0x1002, displayId=0 }}
	行 54179: 10-21 15:00:37.968  4443  4443 V ViewRootImpl[Launcher]: Done with SyntheticInputStage. QueuedInputEvent{flags=FINISHED|FINISHED_HANDLED, hasNextQueuedEvent=true, hasInputEventReceiver=true, mEvent=MotionEvent { action=ACTION_DOWN, actionButton=0, id[0]=0, x[0]=1294.3258, y[0]=401.6281, toolType[0]=TOOL_TYPE_FINGER, buttonState=0, classification=NONE, metaState=0, flags=0x2, edgeFlags=0x0, pointerCount=1, historySize=0, eventTime=98717, downTime=98717, deviceId=-1, source=0x1002, displayId=0 }}
	行 54224: 10-21 15:00:37.968  4443  4443 V ViewRootImpl[Launcher]: Done with EarlyPostImeInputStage. QueuedInputEvent{flags=0, hasNextQueuedEvent=true, hasInputEventReceiver=true, mEvent=MotionEvent { action=ACTION_UP, actionButton=0, id[0]=0, x[0]=1294.3258, y[0]=401.6281, toolType[0]=TOOL_TYPE_FINGER, buttonState=0, classification=NONE, metaState=0, flags=0x2, edgeFlags=0x0, pointerCount=1, historySize=0, eventTime=98749, downTime=98717, deviceId=-1, source=0x1002, displayId=0 }}
	行 54225: 10-21 15:00:37.969  4443  4443 V ViewRootImpl[Launcher]: Done with NativePostImeInputStage. QueuedInputEvent{flags=0, hasNextQueuedEvent=true, hasInputEventReceiver=true, mEvent=MotionEvent { action=ACTION_UP, actionButton=0, id[0]=0, x[0]=1294.3258, y[0]=401.6281, toolType[0]=TOOL_TYPE_FINGER, buttonState=0, classification=NONE, metaState=0, flags=0x2, edgeFlags=0x0, pointerCount=1, historySize=0, eventTime=98749, downTime=98717, deviceId=-1, source=0x1002, displayId=0 }}
	行 54226: 10-21 15:00:37.969  4443  4443 D HMI_Launcher: DragLayer: onInterceptTouchEvent: mDragging=false, ev=MotionEvent { action=ACTION_UP, actionButton=0, id[0]=0, x[0]=1294.3258, y[0]=401.6281, toolType[0]=TOOL_TYPE_FINGER, buttonState=0, classification=NONE, metaState=0, flags=0x2, edgeFlags=0x0, pointerCount=1, historySize=0, eventTime=98749, downTime=98717, deviceId=-1, source=0x1002, displayId=0 }
	行 54228: 10-21 15:00:37.969  4443  4443 D HMI_Launcher: Workspace: onInterceptTouchEvent: 1294.3258, 401.6281
	行 54368: 10-21 15:00:37.974  4443  4443 D ViewRootImpl: processPointerEvent src=0x1002 eventTimeNano=98749755000 id=0x1e902588 MotionEvent:MotionEvent { action=ACTION_UP, actionButton=0, id[0]=0, x[0]=1294.3258, y[0]=401.6281, toolType[0]=TOOL_TYPE_FINGER, buttonState=0, classification=NONE, metaState=0, flags=0x2, edgeFlags=0x0, pointerCount=1, historySize=0, eventTime=98749, downTime=98717, deviceId=-1, source=0x1002, displayId=0 } handled:true
	行 54396: 10-21 15:00:37.974  4443  4443 V ViewRootImpl[Launcher]: Done with ViewPostImeInputStage. QueuedInputEvent{flags=FINISHED|FINISHED_HANDLED, hasNextQueuedEvent=true, hasInputEventReceiver=true, mEvent=MotionEvent { action=ACTION_UP, actionButton=0, id[0]=0, x[0]=1294.3258, y[0]=401.6281, toolType[0]=TOOL_TYPE_FINGER, buttonState=0, classification=NONE, metaState=0, flags=0x2, edgeFlags=0x0, pointerCount=1, historySize=0, eventTime=98749, downTime=98717, deviceId=-1, source=0x1002, displayId=0 }}
	行 54397: 10-21 15:00:37.974  4443  4443 V ViewRootImpl[Launcher]: Done with SyntheticInputStage. QueuedInputEvent{flags=FINISHED|FINISHED_HANDLED, hasNextQueuedEvent=true, hasInputEventReceiver=true, mEvent=MotionEvent { action=ACTION_UP, actionButton=0, id[0]=0, x[0]=1294.3258, y[0]=401.6281, toolType[0]=TOOL_TYPE_FINGER, buttonState=0, classification=NONE, metaState=0, flags=0x2, edgeFlags=0x0, pointerCount=1, historySize=0, eventTime=98749, downTime=98717, deviceId=-1, source=0x1002, displayId=0 }}
```



## （六）、参考资料(特别感谢)：

 [（1）xxx](xxx) 