---
title: Android 10 Camera源码分析10：Camera Android架构(基于Q)转载.md
cover: https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/post.cover.pictures/bing-wallpaper-2018.04.40.jpg
categories: 
 - Camera
tags:
 - Camera
toc: true
abbrlink: 20220515
date: 2022-05-15 05:15:00
---
注：文章都是通过阅读各位前辈总结的资料 Android 10.0 && Linux（Kernel 4.19）Rockchip平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除（◕‿◕），转载请注明出处（©Rockchip ©Android @Linux 版权所有），谢谢。

（==**文章基于 Kernel-4.19**==）&&（==**文章基于 Android 10.0**==）

[【zhoujinjian.com博客原图链接】](https://github.com/zzhoujinjian) 

[【开发板 Khadas Edge V】](https://www.khadas.cn/edge-v)

[【开发板 Khadas Edge V Android 10.0 && Linux（Kernel 4.19）源码链接】](https://github.com/khadas/linux/tree/khadas-edge-Qt)



正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢，由于不喜欢图片水印，去除了水印，敬请谅解！！！

 [Android Camera简单整理(一)-Camera Android架构(基于Q)](https://blog.csdn.net/TaylorPotter/article/details/105387109) 


--------------------------------------------------------------------------------

==源码（部分）==：

**F:\Khadas_Edge_Android_Q\frameworks\av\services\camera\libcameraservice\**

*F:\Khadas_Edge_Android_Q\frameworks\base\core\java\android\hardware\camera2\*

==源码（部分）==：

--------------------------------------------------------------------------------

## 一.Android Camera整体架构简述
----------------------

自Android8.0之后大多机型采用Camera API2 HAL3架构,  
先盗改谷歌的一张图,读完整部代码后再看这张图,真的是很清晰,很简洁,很到位.  
原图:https://source.android.google.cn/devices/camera

### 1.1 Android Camera 基本分层

![](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/Android.10.Camera.10/Android_Camera_basic_layering01.png)

从上图得知,Android手机中Camera软件主要有大体上有4层:

**1.应用层:** 应用开发者调用AOSP提供的接口即可,AOSP的接口即Android提供的相机应用的通用接口,这些接口将通过Binder与Framework层的相机服务进行操作与数据传递;

**2.Framework层:** 位于 frameworks/av/services/camera/libcameraservice/CameraService.cpp ,相机Framework服务是承上启下的作用,上与应用交互,下与HAL曾交互。

**3.Hal层:** 硬件抽象层,Android 定义好了Framework服务与HAL层通信的协议及接口,HAL层如何实现有各个Vendor自己实现,如Qcom的老架构mm-Camera,新架构Camx架构,Mtk的P之后的Hal3架构.

**4.Driver层:** 驱动层,数据由硬件到驱动层处理,驱动层接收HAL层数据以及传递Sensor数据给到HAL层,这里当然是各个Sensor芯片不同驱动也不同.

说到这为什么要分这么多层,大体上还是为了分清界限,升级方便, [AndroidO Treble架构分析](https://blog.csdn.net/yangwen123/article/details/79835965).

Android要适应各个手机组装厂商的不同配置,不同sensor,不管怎么换芯片,从Framework及之上都不需要改变,App也不需要改变就可以在各种配置机器上顺利运行,HAL层对上的接口也由Android定义死,各个平台厂商只需要按照接口实现适合自己平台的HAL层即可.

### 1.2 Android Camera工作大体流程

![](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/Android.10.Camera.10/The_general_process_of_Android_Camera_work02.png)

绿色框中是应用开发者需要做的操作,蓝色为AOSP提供的API,黄色为Native Framework Service,紫色为HAL层Service.  
描述一下步骤:  
1 App一般在MainActivity中使用SurfaceView或者SurfaceTexture+TextureView或者GLSurfaceView等控件作为显示预览界面的控件,共同点都是包含了一个单独的Surface作为取相机数据的容器.

2 在MainActivity onCreate的时候调用API 去通知Framework Native Service CameraServer去connect HAL继而打开Camera硬件sensor.

3 openCamera成功会有回调从CameraServer通知到App,在onOpenedCamera或类似回调中去调用类似startPreview的操作.此时会创建CameraCaptureSession,创建过程中会向CameraServer调用ConfigureStream的操作,ConfigureStream的参数中包含了第一步中空间中的Surface的引用,相当于App将Surface容器给到了CameraServer,CameraServer包装了下该Surface容器为stream,通过HIDL传递给HAL,继而HAL也做configureStream操作

4 ConfigureStream成功后CameraServer会给App回调通知ConfigStream成功,接下来App便会调用AOSP setRepeatingRequest接口给到CameraServer,CameraServer初始化时便起来了一个死循环线程等待来接收Request.

5 CameraServer将request交到Hal层去处理,得到HAL处理结果后取出该Request的处理Result中的Buffer填到App给到的容器中,SetRepeatingRequest为了预览,则交给Preview的Surface容器,如果是Capture Request则将收到的Buffer交给ImageReader的Surface容器.

6 Surface本质上是BufferQueue的使用者和封装者,当CameraServer中App设置来的Surface容器被填满了BufferQueue机制将会通知到应用,此时App中控件取出各自容器中的内容消费掉,Preview控件中的Surface中的内容将通过View提供到SurfaceFlinger中进行合成最终显示出来,即预览;而ImageReader中的Surface被填了,则App将会取出保存成图片文件消费掉.参考

7 录制视频可以参考该篇,这里不再赘述:[\[Android\]\[MediaRecorder\] Android MediaRecorder框架简洁梳理](https://blog.csdn.net/TaylorPotter/article/details/104878815)

再简单一张图如下:  
![](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/Android.10.Camera.10/The_general_process_of_Android_Camera_work03.png)

二. Camera App层简述
----------------

应用层即应用开发者关注的地方,主要就是利用AOSP提供的应用可用的组件实现用户可见可用的相机应用,主要的接口及要点在这  
[Android 开发者/文档/指南/相机](https://developer.android.google.cn/training/camera)

应用层开发者需要做的就是按照AOSP的API规定提供的接口,打开相机,做基本的相机参数的设置,发送request指令,将收到的数据显示在应用界面或保存到存储中.

应用层开发者不需要关注有手机有几个摄像头他们是什么牌子的,他们是怎么组合的,特定模式下哪个摄像头是开或者是关的,他们利用AOSP提供的接口通过AIDL binder调用向Framework层的CameraServer进程下指令,从CameraServer进程中取的数据.

基本过程都如下:

1.  openCamera:Sensor上电
2.  configureStream: 该步就是将控件如GLSurfaceView,ImageReader等中的Surface容器给到CameraServer.
3.  request: 预览使用SetRepeatingRequest,拍一张可以使用Capture,本质都是setRequest给到CameraServer
4.  CameraServer将Request的处理结果Buffer数据填到对应的Surface容器中,填完后由BufferQueue机制回调到引用层对应的Surface控件的CallBack处理函数,接下来要显示预览或保图片App中对应的Surface中都有数据了.

主要一个预览控件和拍照保存控件,视频录制见[\[Android\]\[MediaRecorder\] Android MediaRecorder框架简洁梳理](https://blog.csdn.net/TaylorPotter/article/details/104878815)

三. Camera Framework层简述
----------------------

Camera Framework层即CameraServer服务实现.CameraServer是Native Service,代码在  
frameworks/av/services/camera/libcameraservice/  
CameraServer承上启下,上对应用提供Aosp的接口服务,下和Hal直接交互.一般而言,CamerServer出现问题的概率极低,大部分还是App层及HAL层出现的问题居多.  
CameraServer架构主要架构也如第一张图所示,主要还是Android自己的事.

### 3.1 CameraServer初始化

```
frameworks/av/camera/cameraserver/cameraserver.rc
service cameraserver /system/bin/cameraserver
    class main
    user cameraserver
    group audio camera input drmrpc
    ioprio rt 4
    writepid /dev/cpuset/camera-daemon/tasks /dev/stune/top-app/tasks
    rlimit rtprio 10 10

```

CameraServer由init启动,简单过程如下:  
![](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/Android.10.Camera.10/CameraProvider04.png)

详细过程如下:  
![](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/Android.10.Camera.10/CameraService_init05.png)

### 3.2 App调用CameraServer的相关操作

简单过程如下:  
![](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/Android.10.Camera.10/App_CameraServer06.png)

详细过程如下:

#### 3.2.1 open Camera:

![](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/Android.10.Camera.10/OpenCamera07.png)

#### 3.2.2 configurestream

![](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/Android.10.Camera.10/Configurestream08.png)

#### 3.2.3 preview and capture request:

![](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/Android.10.Camera.10/preview_and_capture_request09.png)

#### 3.2.4 flush and close

![](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/Android.10.Camera.10/flush_and_close10.png)

四 Camera Hal3 子系统
-----------------

Android 官方讲解 [HAL 子系统](https://source.android.google.cn/devices/camera/camera3_requests_hal)  
Android 的相机硬件抽象层 (HAL) 可将 android.hardware.camera2 中较高级别的相机框架 API 连接到底层的相机驱动程序和硬件。  
Android 8.0 引入了 Treble，用于将 CameraHal API 切换到由 HAL 接口描述语言 (HIDL) 定义的稳定接口。  
盗图一张:  
![](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/Android.10.Camera.10/Camera_Hal3_Subsystem11.png)

1.应用向相机子系统发出request,一个request对应一组结果.request中包含所有配置信息。其中包括分辨率和像素格式；手动传感器、镜头和闪光灯控件；3A 操作模式；RAW 到 YUV 处理控件；以及统计信息的生成等.一次可发起多个请求，而且提交请求时不会出现阻塞。请求始终按照接收的顺序进行处理。

2.图中看到request中携带了数据容器Surface,交到framework cameraserver中,打包成Camera3OutputStream实例,在一次CameraCaptureSession中包装成Hal request交给HAL层处理. Hal层获取到处理数据后返回給CameraServer,即CaptureResult通知到Framework,Framework cameraserver则得到HAL层传来的数据给他放进Stream中的容器Surface中.而这些Surface正是来自应用层封装了Surface的控件,这样App就得到了相机子系统传来的数据.

3.HAL3 基于captureRequest和CaptureResult来实现事件和数据的传递,一个Request会对应一个Result.

4.当然这些是Android原生的HAL3定义,接口放在那,各个芯片厂商实现当然不一样,其中接触的便是高通的mm-camera,camx,联发科的mtkcam hal3,后面继续整理实现过程.

HAL3接口定义在  
android_root/hardware/interfaces/camera/
