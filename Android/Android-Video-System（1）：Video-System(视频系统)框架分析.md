---
title: Android Video System（1）：Video System(视频系统)框架分析
cover: https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/hexo.themes/bing-wallpaper-2018.04.16.jpg
categories:
  - Multimedia
tags:
  - Android
  - Video
  - Multimedia
toc: true
abbrlink: 20190208
date: 2019-02-08 09:25:00
---

--------------------------------------------------------------------------------

注：文章都是通过阅读各位前辈总结的资料、Android 7.1.2 && Linux（kernel 3.18）Qualcomm平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除，禁止转载（©Qualcomm Technologies, Inc. 版权所有），谢谢。

[【特别感谢 -  Android NuPlayer播放框架】](http://www.cnblogs.com/tocy/tag/%E6%92%AD%E6%94%BE%E6%A1%86%E6%9E%B6/)
[【特别感谢 -  android ACodec MediaCodec NuPlayer flow】](https://blog.csdn.net/dfhuang09/article/details/54926526)

Google Pixel、Pixel XL 内核代码（文章基于 Kernel-3.18）：
 [Kernel source for Pixel and Pixel XL - GitHub](https://github.com/matthewdalex/marlin)

AOSP 源码（文章基于 Android 7.1.2）：
 [ Android 系统全套源代码分享 (更新到 8.1.0_r1)](https://testerhome.com/topics/2229)

--------------------------------------------------------------------------------
☯ V4l2 框架代码
☯ kernel/drivers/media/v4l2-core/（文件前缀为 videobuf2）

☯ MSM 视频驱动程序文件
☯ kernel/drivers/media/platform/msm/vidc/

☯ 设备树
☯ /kernel/arch/arm/boot/dts/qcom（Venus 的寄存器基址，时钟频率）

☯ Stagefright、libmedia、libmediaplayerservice、mediaserver
☯ /frameworks/av/media/

☯ OMX
☯ /hardware/qcom/media/mam8996/mm-video-v4l2/vidc/

☯ OMX 核心
☯ /hardware/qcom/media/mm-core

☯ 软件编解码器路径
☯ /vendor/qcom/proprietary/mm-video/omx_vpp(?)→ 解码器代码
☯ /vendor/qcom/proprietary/mm-video/omx_vpp(?) → 编码器代码

--------------------------------------------------------------------------------
#### (一)、Android Video Overview
基于 OpenMAX 的视频解码 – 数据流
![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/video.system/01-01-OpenMax-Based video decode - data flow.png)

> YUV，是一种颜色编码方法。常使用在各个视频处理组件中。 YUV在对照片或视频编码时，考虑到人类的感知能力，允许降低色度的带宽。[YUV](https://zh.wikipedia.org/wiki/YUV)
> VPU，Video processing unit 

基于 OpenMAX 的视频编码 – 数据流
![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/video.system/01-02-OpenMax-Based video encode - data flow.png.png)

视频框架：
![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/video.system/01-03-Video Architecture Software Stack.png)

组件描述：
![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/video.system/01-04-Q Component Description.png)

总结：
    从视频框架可以了解到。视频文件先经Stagefright传到OMX decoder解码（软解或硬解）、OMX decoder将解码后的YUV数据回传到Stagefright，不断循环播放同时经由SurfaceFlinger渲染到LCD屏幕上。
#### (二)、Android MediaPlayer & Nuplayer 框架分析

##### 2.1、MediaPlayer
Android在Java层中提供了一个MediaPlayer的类来作为播放媒体资源的接口，在使用中我们通常会编写以下的代码：

``` cpp
mMediaPlayer = new MediaPlayer();
mMediaPlayer.setDataSource(Environment.getExternalStorageDirectory()+"/test_video.mp4");
mMediaPlayer.setDisplay(...);
mMediaPlayer.setAudioStreamType(AudioManager.STREAM_MUSIC);
mMediaPlayer.prepareAsync();
mMediaPlayer.start();
mediaPlayer.pause(); 
mediaPlayer.stop();
mediaPlayer.reset();
mediaPlayer.release();
```

通常MediaPlayer的调用逻辑是，构造函数-> setDataSource -> SetVideoSurfaceTexture-> prepare/prepareAsync -> start-> stop-> reset-> 析构函数，按照实际需求还会调用pause、isPlaying、getDuration、getCurrentPosition、setLooping、seekTo等方法。

##### 2.1.1、MediaPlayer状态图:
![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/video.system/01-05-MediaPlayer-status-turn-.png)

☯ Idle状态 
调用new或reset()方法创建MediaPlayer后进入空闲
☯ End状态  
调用release()后就结束
☯ Error状态 
播放控制操作出错或无效状态下调用播放控制操作
☯ Initialized状态

调用setDataSource之后完成初始化
☯ Prepared状态  
同步prepare()或异步prepareAsync()完成准备
☯ Preparing状态  
是一种瞬时状态，调用prepareAsync()时会先进入此状态
☯ Started  状态 
要开始播放必须调用start()
☯ Paused  状态
调用pause()并成功返回后播放可以被暂停
☯ Stopped状态  
调用stop()会停止播放
☯ PlaybackCompleted状态
当播放到达流末端时，播放完成
##### 2.1.2、MediaPlayer和MediaPlayerService

mediaserver 启动后会把media相关一些服务添加到servicemanager中，其中就有mediaPlayerService。这样应用启动前，系统就有了mediaPlayerService这个服务程序。

``` cpp
[->\frameworks\av\media\mediaserver\main_mediaserver.cpp]
```
![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/video.system/01-06-Main_mediaserver.png)


##### 2.1.3、创建MediaPlayer
☯ Java应用程序中创建MediaPlayer对象
MediaPlayer mediaPlayer = new MediaPlayer();
☯ MediaPlayer的构造函数中比较重要的就是本地的native函数：native_setup()其对应的JNI函数为
android_media_MediaPlayer_native_setup()

``` cpp
[->\frameworks\base\media\jni\android_media_MediaPlayer.cpp]
```
![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/video.system/01-07-android_media_MediaPlayer_native_setup.png)

构造Native层的MediaPlayer对象的时候【MediaPlayer.cpp】，也会构造其父类的对象。在MediaPlayer的父类IMediaDeathNotifier中有个很重要的方法getMediaPlayerService()来获取MediaPlayerService，其关系到MediaPlayer和MediaPlayerService之间的通信。

##### 2.1.4、setDataSource()设置播放资源

在整个应用程序的进程中，Mediaplayer.cpp 中 setDataSource会从service manager中获得mediaPlayerService 服务，然后通过服务来创建player，这个player就是播放器的真实实例，同时也使MediaPlayer和MediaPlayerService建立了联系。
在java层MediaPlayer.java中的setDataSource最终会调用_setDataSource方法，对应native层MediaPlayer.cpp中的setDataSource方法。
![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/video.system/01-08-mp_setDataSource.png)

通过 getMediaPlayerService 得到的BpMediaPlayerService类型的service，和mediaPlayerService进程中的BnMediaPlayerService 相对应负责binder通讯。
![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/video.system/01-09-MediaPlayerService_Create.png)

在create函数中创建了一个MediaPlayerService::Client的实例，是MediaPlayerService的内部类，也就是说MediaPlayerService会为每个client应用进程创建一个相应的MediaPlayerService::Client的实例，来实现播放以及播放过程的控制，向MediaPlayer发事件通知。到这里，在Server端的对象就创建完成了。

然后在MediaPlayer.cpp中就得到了一个sever端的player实例，它和本地其他类的实例没什么用法上的区别，而实际上则是通过binder机制运行在另外一个进程中的。获得此实例后继续player->setDataSource操作。

![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/video.system/01-10-player-setDataSource.png)

小结：
Java应用程序中使用MediaPlayer.java的setDataSource()会传递到Native层中MediaPlayer.cpp的setDataSource()去执行，而MediaPlayer.cpp又会把这个方法交给MediaPlayerservice去执行。MediaPlayerService则是使用NuPlayer实现的，最后， setDataSource还是交给了NuPlayer去执行了。这个过程把MediaPlayer和MediaPlayerService之间的联系建立起来，同时又把MediaPlayerService和NuPlayer的关系建立了起来。

![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/video.system/01-11-GenericSource-setDataSource.png)


##### 2.1.5、setDisplay()
 下一步就是java层的setDisplay，依然查看java层MediaPlayer：

``` cpp
[->\frameworks\base\media\java\android\media\MediaPlayer.java]
    public void setDisplay(SurfaceHolder sh) {
        mSurfaceHolder = sh;
        Surface surface;
        if (sh != null) {
            surface = sh.getSurface();
        } else {
            surface = null;
        }
        _setVideoSurface(surface);
        updateSurfaceScreenOn();
    }
```
最后会调用本地方法_setVideoSurface，我们继续找到它的jni实现：

``` cpp
[->\frameworks\base\media\jni\android_media_MediaPlayer.cpp]
static void android_media_MediaPlayer_setVideoSurface(JNIEnv *env, jobject thiz, jobject jsurface)
{
    setVideoSurface(env, thiz, jsurface, true /* mediaPlayerMustBeAlive */);
}

static void setVideoSurface(JNIEnv *env, jobject thiz, jobject jsurface, jboolean mediaPlayerMustBeAlive)
{
    sp<MediaPlayer> mp = getMediaPlayer(env, thiz);//获取C++的MediaPlayer
    if (mp == NULL) {
        if (mediaPlayerMustBeAlive) {
            jniThrowException(env, "java/lang/IllegalStateException", NULL);
        }
        return;
    }
	//将旧的IGraphicBufferProducer的强引用减一
    decVideoSurfaceRef(env, thiz);
	//IGraphicBufferProducer图层缓冲区合成器
    sp<IGraphicBufferProducer> new_st;
    if (jsurface) {
	    //得到java层的surface
        sp<Surface> surface(android_view_Surface_getSurface(env, jsurface));
        if (surface != NULL) {
	        //获取IGraphicBufferProducer
            new_st = surface->getIGraphicBufferProducer();
            if (new_st == NULL) {
                jniThrowException(env, "java/lang/IllegalArgumentException",
                    "The surface does not have a binding SurfaceTexture!");
                return;
            }
            //增加IGraphicBufferProducer的强引用+1
            new_st->incStrong((void*)decVideoSurfaceRef);
        } else {
            jniThrowException(env, "java/lang/IllegalArgumentException",
                    "The surface has been released");
            return;
        }
    }
	//上面我们在native_init方法中将java层mNativeSurfaceTexture查找给了jni层，正好，在这里将IGraphicBufferProducer赋给它
    env->SetLongField(thiz, fields.surface_texture, (jlong)new_st.get());

    // This will fail if the media player has not been initialized yet. This
    // can be the case if setDisplay() on MediaPlayer.java has been called
    // before setDataSource(). The redundant call to setVideoSurfaceTexture()
    // in prepare/prepareAsync covers for this case.
    //如果MediaPlayer没有初始化，这一步会失败。原因可能是setDisplay在setDataSource之前。如果在prepare/prepareAsync 时想规避这个错误而去调用setVideoSurfaceTexture是多余的。
    //最终会调用C++层的setVideoSurfaceTexture方法，下一节在分析
    mp->setVideoSurfaceTexture(new_st);
}
//将旧的IGraphicBufferProducer的强引用减一
static void decVideoSurfaceRef(JNIEnv *env, jobject thiz)
{
    sp<MediaPlayer> mp = getMediaPlayer(env, thiz);
    if (mp == NULL) {
        return;
    }

    sp<IGraphicBufferProducer> old_st = getVideoSurfaceTexture(env, thiz);
    if (old_st != NULL) {
        old_st->decStrong((void*)decVideoSurfaceRef);
    }
}
```
这一步主要是对图像显示的surface的保存，然后将旧的IGraphicBufferProducer强引用减一，再获得新的IGraphicBufferProducer，最后会调用C++的MediaPlayer的setVideoSurfaceTexture将它折纸进去。

IGraphicBufferProducer是SurfaceFlinger的内容，一个UI完全显示到diplay的过程，SurfaceFlinger扮演着重要的角色但是它的职责是“Flinger”，即把系统中所有应用程序的最终的“绘图结果”进行“混合”，然后统一显示到物理屏幕上，而其他方面比如各个程序的绘画过程，就由其他东西来担任了。这个光荣的任务自然而然地落在了BufferQueue的肩膀上，它是每个应用程序“一对一”的辅导老师，指导着UI程序的“画板申请”、“作画流程”等一系列细节。下面的图描述了这三者的关系：

![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/video.system/01-12-IGraphicBufferProducer.png)

   虽说是三者的关系，但是他们所属的层却只有两个，app属于Java层，BufferQueue/SurfaceFlinger属于native层。也就是说BufferQueue也是隶属SurfaceFlinger，所有工作围绕SurfaceFlinger展开。
       这里IGraphicBufferProducer就是app和BufferQueue重要桥梁，GraphicBufferProducer承担着单个应用进程中的UI显示需求，与BufferQueue打交道的就是它。
##### 2.1.6、播放器基本模型
NuPlayer不管有多么神秘，说到底还是个播放器。在播放器的基本模型上，他与VCL、mplayer、ffmpeg等开源的结构是一致的。只是组织实现的方式不同。
深入了解NuPlayer之前，把播放器的基本模型总结一下，然后按照模型的各个部分来深入研究NuPlayer的实现方式。

![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/video.system/01-13-source-demux-decoder-output.jpg)



☯ datasource数据源：数据源，数据的来源不一定都是本地file，也有可能是网路上的各种协议例如：http、rtsp、HLS等。source的任务就是把数据源抽象出来，为下一个demux模块提供它需要的稳定的数据流。demux不用关信数据到底是从什么地方来的。

☯ demuxer解复用：视频文件一般情况下都是把音视频的ES流交织的通过某种规则放在一起。这种规则就是容器规则。现在有很多不同的容器格式。如ts、mp4、flv、mkv、avi、rmvb等等。demux的功能就是把音视频的ES流从容器中剥离出来，然后分别送到不同的解码器中。其实音频和视频本身就是2个独立的系统。容器把它们包在了一起。但是他们都是独立解码的，所以解码之前，需要把它分别 独立出来。demux就是干这活的，他为下一步decoder解码提供了数据流。

☯ decoder解码：解码器—-播放器的核心模块。分为音频和视频解码器。影像在录制后, 原始的音视频都是占用大量空间, 而且是冗余度较高的数据. 因此, 通常会在制作的时候就会进行某种压缩 ( 压缩技术就是将数据中的冗余信息去除数据之间的相关性 ). 这就是我们熟知的音视频编码格式, 包括MPEG1（VCD）\ MPEG2（DVD）\ MPEG4 \ H.264 等等. 音视频解码器的作用就是把这些压缩了的数据还原成原始的音视频数据. 当然, 编码解码过程基本上都是有损的 .解码器的作用就是把编码后的数据还原成原始数据。

☯ output输出：输出部分分为音频和视频输出。解码后的音频（pcm）和视频（yuv）的原始数据需要得到音视频的output模块的支持才能真正的让人的感官系统（眼和耳）辨识到。

所以，播放器大致分成上述4部分。怎么抽象的实现这4大部分、以及找到一种合理的方式将这几部分组织并运动起来。是每个播放器不同的实现方式而已。接下来就围绕这4大部分做深入学习，看看NuPlayer的工作原理。
##### 2.2、NuPlayer分析
##### 2.2.0、NuPlayer简介
Android2.3时引入流媒体框架，而流媒体框架的核心是NuPlayer。在之前的版本中一般认为Local Playback就用Stagefrightplayer+Awesomeplayer，流媒体用NuPlayer。Android4.0之后HttpLive和RTSP协议开始使用NuPlayer播放器，Android5.0（L版本）之后本地播放也开始使用NuPlayer播放器。 Android7.0(N版本)则完全去掉了Awesomeplayer。
通俗点说，NuPlayer是AOSP中提供的多媒体播放框架，能够支持本地文件、HTTP（HLS）、RTSP等协议的播放，通常支持H.264、H.265/HEVC、AAC编码格式，支持MP4、MPEG-TS封装。
在实现上NuPlayer和Awesomeplayer不同，NuPlayer基于StagefrightPlayer的基础类构建，利用了更底层的ALooper/AHandler机制来异步地处理请求，ALooper列队消息请求，AHandler中去处理，所以有更少的Mutex/Lock在NuPlayer中。Awesomeplayer中利用了omxcodec而NuPlayer中利用了Acodec。

##### 2.2.1、NuPlayer整体类关系图

![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/video.system/01-14-NuPlayer-arc.jpg)

NuPlayer由NuPlayerDriver封装，利用了底层的ALooper/AHandler机制来异步地处理请求，ALooper保存消息请求，然后在AHandler中处理。另外，NuPlayer中利用到了Acodec。

![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/video.system/01-15-NuPlayer-class.jpg)

☯ NuPlayer::Source
解析模块（parser，功能类似FFmpeg的avformat）。其接口与MediaExtractor和
MediaSource组合的接口差不多，同时提供了用于快速定位的seekTo接口。

☯ NuPlayer::Decoder
解码模块（decoder，功能类似FFmpeg的avcodec），封装了用于AVC、AAC解码的接口，
通过ACodec实现解码（包含OMX硬解码和软解码）。

☯ NuPlayer::Render
渲染模块（render，功能类似声卡驱动和显卡驱动），主要用于音视频渲染和同步，与
NativeWindow有关。

☯ NuPlayer  是播放框架中连接Source、Decoder、Renderer的纽带

☯ NuPlayerDriver
作为NuPlayer类的封装，直接调用NuPlayer。

NuPlayer框架中最顶层的类是NuPlayerDriver，继承自MediaPlayerInterface，主要提供一个状态转换机制，作为NuPlayer类的Wrapper。NuPlayerDriver类中最重要的成员是以下几个：

``` cpp
> State mState 播放器状体标志 
> sp <ALooper> mLooper 内部消息驱动机制 
> sp <NuPlayer>  mPlayer 真正完成播放器的类
```

NuPlayerDriver主要是 构造函数-> setDataSource -> SetVideoSurfaceTexture-> prepare/prepareAsync -> start-> stop-> reset-> 析构函数，实际需求pause、isPlaying、getDuration、getCurrentPosition、setLooping、seekTo等方法

##### 2.2.2、NuPlayer框架需要关注知识点

``` cpp
NuPlayer的框架，其内部实现逻辑。那么最终就落实到如何从一个类中提取出需要的框架及知识点。那么一个类的对外接口部分通常包括：
--- 构造函数和析构函数
--- 必须调用的接口
--- 可选的调用接口

在多媒体播放中，通过关注的点有：
--- 如何实现解复用，得到音频、视频、字幕等数据
--- 如何实现解码
--- 如何实现音视频同步
--- 如何渲染视频
--- 如何播放音频
--- 如何实现快速定位
```

#### (三)、Android MediaPlayer框架分析 - AHandler AMessage ALooper
前文中提到过NuPlayer基于StagefrightPlayer的基础类构建，利用了更底层的ALooper/AHandler机制来**异步地处理请求**，ALooper保存消息请求，然后调用AHandler接口去处理。
实际上在代码中NuPlayer本身继承自AHandler类，而ALooper对象保存在NuPlayerDriver中。
ALooper/AHandler机制是模拟的消息循环处理方式，通常有三个主要部分：消息（message，通常包含Handler）、消息队列（queue）、消息处理线程（looper thread）。

![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/video.system/01-16-AHandler-ALooper-AMessage.png)

对于handler消息机制，构成就必须包括一个Loop，message。那么对应的AHandler，也应该有对应的ALooper、AMessage。
因此本小节主要涉及到三个类ALooper、AHandler、AMessage。


##### 3.1、AHandler接口分析（消息处理类）
下面代码是AHandler接口:

``` cpp
"./frameworks/av/include/media/stagefright/AHandler.h"

struct AHandler : public RefBase {
    AHandler();

    ALooper::handler_id id() const;
    sp<ALooper> looper() const;
    wp<ALooper> getLooper() const;
    wp<AHandler> getHandler() const;

protected:
    virtual void onMessageReceived(const sp<AMessage> &msg) = 0;

private:
    friend struct AMessage;      // deliverMessage()
    friend struct ALooperRoster; // setID()

    uint32_t mMessageCounter;
    KeyedVector<uint32_t, uint32_t> mMessages;

    void setID(ALooper::handler_id id, wp<ALooper> looper);
    void deliverMessage(const sp<AMessage> &msg);
};
```
看上面接口，初步印象是AHandler没有直接对外的接口（只有获取成员变量的接口），基本上只有一个onMessageReceived用于子类继承，deliverMessage用于给类AMessage使用，setID用于给友元类ALooperRoster使用。从这点来说，真正代码应该在AMessage里边。

##### 3.2、AMessage接口分析（消息载体）
下面代码是AMessage的声明：

``` cpp
"./frameworks/av/include/media/stagefright/AMessage.h"
struct AMessage : public RefBase {
    AMessage();
    AMessage(uint32_t what, const sp<const AHandler> &handler); // 代码中常用的构造函数
    static sp<AMessage> FromParcel(const Parcel &parcel, size_t maxNestingLevel = 255);

    // Write this AMessage to a parcel.
    // All items in the AMessage must have types that are recognized by
    // FromParcel(); otherwise, TRESPASS error will occur.
    void writeToParcel(Parcel *parcel) const;

    void setWhat(uint32_t what);
    uint32_t what() const;

    // 注意这是一个AHandler，通过这个可以获得ALooper对象引用
    void setTarget(const sp<const AHandler> &handler);

    // 清除所有设置的消息属性参数
    void clear();
    
    // 一系列设置/获取 Message 属性的函数。。。
    void setInt32/setInt64/setSize/setFloat/setDouble/setPointer/setPointer/setString/setRect/setObject/setBuffer/setMessage(...);
    bool findInt32/findInt64/findSize/findFloat/findDouble/findPointer/findString/findObject/findBuffer/findMessage/findRect(...) const;

    // 通过这个函数检索下指定名称的消息属性是否存在
    bool contains(const char *name) const;

    // 投递消息的接口，顾名思义直接投递给构造函数的ALooper，注意支持延时消息，但不支持提前消息，delayUS > 0
    status_t post(int64_t delayUs = 0);

    // 投递消息并等待执行结束后发送response消息
    status_t postAndAwaitResponse(sp<AMessage> *response);

    // If this returns true, the sender of this message is synchronously
    // awaiting a response and the reply token is consumed from the message
    // and stored into replyID. The reply token must be used to send the response
    // using "postReply" below.
    bool senderAwaitsResponse(sp<AReplyToken> *replyID);

    // Posts the message as a response to a reply token.  A reply token can
    // only be used once. Returns OK if the response could be posted; otherwise,
    // an error.
    status_t postReply(const sp<AReplyToken> &replyID);

    // 深拷贝
    sp<AMessage> dup() const;

    // 比较两个消息，并返回差异
    sp<AMessage> changesFrom(const sp<const AMessage> &other, bool deep = false) const;

    // 获取消息属性存储的个数及特定索引上的消息属性参数
    size_t countEntries() const;
    const char *getEntryNameAt(size_t index, Type *type) const;

protected:
    virtual ~AMessage();

private:
    friend struct ALooper; // deliver()

    uint32_t mWhat;

    wp<AHandler> mHandler;
    wp<ALooper> mLooper;
    
    // 用于ALooper调用的，发送消息的接口
    void deliver();
};
```
从上面的接口可以看出在使用AMessage是只需要指定消息的id和要处理该消息的AHandler即可，可以通过构造函数，也可以单独调用setWhat和setTarget接口。AMessage构造完成之后，可以调用setXXX设置对应的参数，通过findXXX获取传递的参数。最后通过post即可将消息投递到AHandler的消息队列中。

##### 3.3、ALooper接口分析（消息处理循环及后台线程）
其简化的声明如下：

``` cpp
"./frameworks/av/include/media/stagefright/ALooper.h"
struct ALooper : public RefBase {
    ALooper();

    // Takes effect in a subsequent call to start().
    void setName(const char *name);
    const char *getName() const;

    handler_id registerHandler(const sp<AHandler> &handler);
    void unregisterHandler(handler_id handlerID);

    status_t start(bool runOnCallingThread = false,
            bool canCallJava = false, int32_t priority = PRIORITY_DEFAULT);

    status_t stop();

    static int64_t GetNowUs();    

protected:
    virtual ~ALooper();

private:
    friend struct AMessage;       // post()

    AString mName;

    struct Event {
        int64_t mWhenUs;
        sp<AMessage> mMessage;
    };
    List<Event> mEventQueue;

    struct LooperThread;
    sp<LooperThread> mThread;
    bool mRunningLocally;

    // START --- methods used only by AMessage

    // posts a message on this looper with the given timeout
    void post(const sp<AMessage> &msg, int64_t delayUs);

    // creates a reply token to be used with this looper
    sp<AReplyToken> createReplyToken();
    // waits for a response for the reply token.  If status is OK, the response
    // is stored into the supplied variable.  Otherwise, it is unchanged.
    status_t awaitResponse(const sp<AReplyToken> &replyToken, sp<AMessage> *response);
    // posts a reply for a reply token.  If the reply could be successfully posted,
    // it returns OK. Otherwise, it returns an error value.
    status_t postReply(const sp<AReplyToken> &replyToken, const sp<AMessage> &msg);

    // END --- methods used only by AMessage

    bool loop();
};
```
ALooper对外接口比较简单，通常就是NuPlayerDriver构造函数中的调用逻辑。先创建一个ALooper对象，然后调用setName和start接口，之后调用registerHandler设置一个AHandler，这样就完成了初始化。在析构之前需要调用stop接口。
这里需要说明下，ALooper::start接口会启动一个线程，并调用ALooper::loop函数，该函数主要实现消息的实际执行。代码如下：

``` cpp
bool ALooper::loop() {
    Event event;

    {
        Mutex::Autolock autoLock(mLock);
        if (mThread == NULL && !mRunningLocally) {
            return false;
        }

        // 从mEventQueue取出消息，判断是否需要执行，不需要的话就等待
        // 需要的话就调用handler执行，并删除对应消息
        if (mEventQueue.empty()) {
            mQueueChangedCondition.wait(mLock);
            return true;
        }
        int64_t whenUs = (*mEventQueue.begin()).mWhenUs;
        int64_t nowUs = GetNowUs();

        if (whenUs > nowUs) {
            int64_t delayUs = whenUs - nowUs;
            mQueueChangedCondition.waitRelative(mLock, delayUs * 1000ll);

            return true;
        }

        event = *mEventQueue.begin();
        mEventQueue.erase(mEventQueue.begin());
    }

    event.mMessage->deliver();

    return true;
}
```
那么消息是通过那个函数添加进来的呢？ 这就是友元类AMessage的作用，通过调用ALooper::post接口，将AMessage添加到mEventQueue中。

##### 3.4、一个调用实例
以NuPlayer::setVideoSurfaceTextureAsync为示例分析下ALooper/AHandler机制。
这里不解释ALooper的初始化过程，有兴趣的可以参考资料Android Native层异步消息处理框架的内容。
下面是setVideoSurfaceTextureAsync的代码。

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayer.cpp]
void NuPlayer::setVideoSurfaceTextureAsync(
        const sp<IGraphicBufferProducer> &bufferProducer) {
    sp<AMessage> msg = new AMessage(kWhatSetVideoSurface, this);

    if (bufferProducer == NULL) {
        msg->setObject("surface", NULL);
    } else {
        msg->setObject("surface", new Surface(bufferProducer, true /* controlledByApp */));
    }

    msg->post();
}
```
这段代码功能很简单，创建一个AMessage对象，并设置下参数，参数类型为Object，名称是"surface"，然后通过AMessage::post接口，间接调用ALooper::post接口，将消息发送给ALooper-NuPlayerDriver::mLooper；ALooper的消息循环线程检测到这个消息，在ALooper::loop函数中通过AMessage的deliver接口，调用AHandler::deliverMessage接口，这个函数会调动NuPlayer::onMessageReceived（通过继承机制实现）接口。这样绕了一圈。我们就可以通过ALooper/AHandler机制处理消息了。
具体处理代码如下

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayer.cpp]
void NuPlayer::onMessageReceived(const sp<AMessage> &msg) {
    switch (msg->what()) {
        case kWhatSetVideoSurface:
        {

            sp<RefBase> obj;
            CHECK(msg->findObject("surface", &obj));
            sp<Surface> surface = static_cast<Surface *>(obj.get());

            ALOGD("onSetVideoSurface(%p video decoder)", surface.get());

            // Need to check mStarted before calling mSource->getFormat because NuPlayer might
            // be in preparing state and it could take long time.
            // When mStarted is true, mSource must have been set.
            if (mSource == NULL || !mStarted || mSource->getFormat(false /* audio */) == NULL
                    // NOTE: mVideoDecoder's mSurface is always non-null
                    || (mVideoDecoder != NULL && mVideoDecoder->setVideoSurface(surface) == OK)) {
                performSetSurface(surface);
                break;
            }

        }
        
        // ... 省略其他部分代码
    }
}
```

#### (四)、NuPlayer源码分析
这次我们需要深入分析的是NuPlayer类，相比于NuPlayerDriver的接口功能，NuPlayer继承自AHandler类，是AOSP播放框架中连接Source、Decoder、Render的纽带。
##### 4.1、主要接口和核心的类成员
NuPlayer类被NuPlayerDriver直接调用，其主要接口如下：

``` cpp
// code from NuPlayer.h (~/frameworks/av/media/libmediaplayerservice/nuplayer/)
struct NuPlayer : public AHandler {
    NuPlayer(pid_t pid);
    void setUID(uid_t uid);
    void setDriver(const wp<NuPlayerDriver> &driver);
    void setDataSourceAsync(...);
    void prepareAsync();
    void setVideoSurfaceTextureAsync(const sp<IGraphicBufferProducer> &bufferProducer);
    void start();
    void pause();

    // Will notify the driver through "notifyResetComplete" once finished.
    void resetAsync();

    // Will notify the driver through "notifySeekComplete" once finished
    // and needNotify is true.
    void seekToAsync(int64_t seekTimeUs, bool needNotify = false);

    status_t setVideoScalingMode(int32_t mode);
    status_t getTrackInfo(Parcel* reply) const;
    status_t getSelectedTrack(int32_t type, Parcel* reply) const;
    status_t selectTrack(size_t trackIndex, bool select, int64_t timeUs);
    status_t getCurrentPosition(int64_t *mediaUs);

    sp<MetaData> getFileMeta();
    float getFrameRate();

protected:
    virtual ~NuPlayer();
    virtual void onMessageReceived(const sp<AMessage> &msg);
}
```
接口分类下，无外乎几个分类：

☯ 用于初始化的（比如构造函数、setDriver/setDataSourceAsync/prepareAsync/setVideoSurfaceTextureAsync）
☯ 用于销毁的（比如析构函数、resetAsync）
☯ 用于播放控制的（比如start/pause/seekToAsync）
☯ 用于状态获取的（比如getCurrentPosition/getFileMeta）
下面是主要的类成员部分

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayer.h]
wp<NuPlayerDriver> mDriver; // 接口调用方
sp<Source> mSource; // 相当于FFmpeg中的demuxer
sp<Surface> mSurface; // 显示用的Surface
sp<DecoderBase> mVideoDecoder; // 视频解码器
sp<DecoderBase> mAudioDecoder; // 音频解码器
sp<CCDecoder> mCCDecoder; 
sp<Renderer> mRenderer; // 渲染器
sp<ALooper> mRendererLooper;
```
##### 4.2、setDataSourceAsync()现分析
这个函数有多重不同的重载形式，如下：

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayer.h]
void setDataSourceAsync(const sp<IStreamSource> &source);
void setDataSourceAsync(const sp<IMediaHTTPService> &httpService, const char *url,
            const KeyedVector<String8, String8> *headers);
void setDataSourceAsync(int fd, int64_t offset, int64_t length);
void setDataSourceAsync(const sp<DataSource> &source);
```

需要根据实际情况选择，这里以第三个接口为例，说明下多本地媒体文件是如何处理的。
下面是这个函数的实现代码：

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayer.cpp]
void NuPlayer::setDataSourceAsync(int fd, int64_t offset, int64_t length) {
    sp<AMessage> msg = new AMessage(kWhatSetDataSource, this);

    sp<AMessage> notify = new AMessage(kWhatSourceNotify, this);
    // 创建对象用于读取本地文件
    sp<GenericSource> source =
            new GenericSource(notify, mUIDValid, mUID);
    // 实际干活的的代码
    status_t err = source->setDataSource(fd, offset, length);
    
    msg->setObject("source", source);
    msg->post();
}
```
看实现很简单，创建GenericSource对象，并调用其setDataSource接口，然后发送kWhatSetDataSource消息。
我们看看如何处理然后发送kWhatSetDataSource消息呢？代码如下：
``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayer.cpp]
case kWhatSetDataSource:
{
    CHECK(mSource == NULL);

    status_t err = OK;
    sp<RefBase> obj;
    CHECK(msg->findObject("source", &obj));
    if (obj != NULL) {
        Mutex::Autolock autoLock(mSourceLock);
        mSource = static_cast<Source *>(obj.get());
    } else {
        err = UNKNOWN_ERROR;
    }
    // 通知Driver函数调用完成
    CHECK(mDriver != NULL);
    sp<NuPlayerDriver> driver = mDriver.promote();
    if (driver != NULL) {
        driver->notifySetDataSourceCompleted(err);
    }
    break;
}
```

看到这里发现，其实没做什么就是直接通知NuPlayerDriver。我们还注意到这里构建了一个特殊消息（AMessage）notify，这个消息用于在Source和NuPlayer直接传递。下面这是消息循环中的处理函数：

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayer.cpp]
case kWhatSourceNotify:
{
    onSourceNotify(msg);
    break;
}
```

在后续讨论Source的时候详细说明这个消息通知的意义。
##### 4.3、prepareAsync()
这个函数实现的功能对应于MediaPlayerBase::prepare/prepareAsync接口，实现异步的prepare功能，一般就是做一些额外的初始化工作。那么直接看一下实现：

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayer.cpp]
void NuPlayer::prepareAsync() {
    (new AMessage(kWhatPrepare, this))->post();
}
```
代码就是发了一个kWhatPrepare的消息。接下来是如何处理这个消息。

``` cpp
case kWhatPrepare:
{
    mSource->prepareAsync();
    break;
}
```

最终还是调用了Source::prepareAsync接口。后面会解释其功能。（这里面可能会解析下码流，读取音频、视频、字幕流信息，读取时长、元数据等）。
##### 4.3、setVideoSurfaceTextureAsync()
调用这个接口主要为了设置视频渲染窗口。其实现相对简单，创建一个Surface，然后发送异步的kWhatSetVideoSurface消息。代码如下：

``` cpp
void NuPlayer::setVideoSurfaceTextureAsync( const sp<IGraphicBufferProducer> &bufferProducer) {
    sp<AMessage> msg = new AMessage(kWhatSetVideoSurface, this);

    if (bufferProducer == NULL) {
        msg->setObject("surface", NULL);
    } else {
        msg->setObject("surface", new Surface(bufferProducer, true /* controlledByApp */));
    }

    msg->post();
}
```

那么看看如何处理kWhatSetVideoSurface消息呢？

``` cpp
case kWhatSetVideoSurface: {
    sp<RefBase> obj;
    CHECK(msg->findObject("surface", &obj));
    sp<Surface> surface = static_cast<Surface *>(obj.get());

    // Need to check mStarted before calling mSource->getFormat because NuPlayer might
    // be in preparing state and it could take long time.
    // When mStarted is true, mSource must have been set.
    if (mSource == NULL || !mStarted || mSource->getFormat(false /* audio */) == NULL
            // NOTE: mVideoDecoder's mSurface is always non-null
            || (mVideoDecoder != NULL && mVideoDecoder->setVideoSurface(surface) == OK)) {
        performSetSurface(surface); // 通知NuPlayerDriver设置完成
        break;
    }
    // 清空音频、视频缓冲
    mDeferredActions.push_back(
            new FlushDecoderAction(FLUSH_CMD_FLUSH /* audio */,FLUSH_CMD_SHUTDOWN /* video */));
    // 最终调用NuPlayer::performSetSurface接口
    mDeferredActions.push_back(new SetSurfaceAction(surface));

    if (obj != NULL || mAudioDecoder != NULL) {
        if (mStarted) {
            // Issue a seek to refresh the video screen only if started otherwise
            // the extractor may not yet be started and will assert.
            // If the video decoder is not set (perhaps audio only in this case)
            // do not perform a seek as it is not needed.
            int64_t currentPositionUs = 0;
            if (getCurrentPosition(&currentPositionUs) == OK) {
                mDeferredActions.push_back(
                        new SeekAction(currentPositionUs));
            }
        }

        // 对于新的surface设置，重置下解码器
        mDeferredActions.push_back(new SimpleAction(&NuPlayer::performScanSources));
    }

    // After a flush without shutdown, decoder is paused.
    // Don't resume it until source seek is done, otherwise it could
    // start pulling stale data too soon.
    mDeferredActions.push_back(
            new ResumeDecoderAction(false /* needNotify */));
    // 把上面mDeferredActions中缓存的所有Action处理下，并清空
    processDeferredActions();
    break;
}
```

这里的代码相对复杂点，涉及到很多，其实主要是为了设置Surface之后，可以正常解码显示，因为某些情况下解码器初始化需要依赖于具体的Surface。当然，里边还涉及到NuPlayer状态及初始化判断。
##### 4.4、start()/pause()
start函数实现很简单，实际就发送了kWhatStart消息。

``` cpp
void NuPlayer::start() {
    (new AMessage(kWhatStart, this))->post();
}
```

在消息处理函数中的处理如下：

``` cpp
case kWhatStart:
{
    if (mStarted) {
        // do not resume yet if the source is still buffering
        if (!mPausedForBuffering) {
            onResume();
        }
    } else {
        onStart();
    }
    mPausedByClient = false;
    break;
}
```

直接调用了OnStart/OnResume函数。
pause函数实现类似，只是发送的是kWhatPause消息。在消息处理函数中的代码如下：

``` cpp
case kWhatPause:
{
    onPause();
    mPausedByClient = true;
    break;
}
```

直接调用的onPause函数。下面单独分析下这三个函数。先从简单的函数开始OnPause/onResume

NuPlayer::onPause
这个函数实现暂停功能，总体来说就是把Source和Render暂停就可以了，代码如下：

``` cpp
void NuPlayer::onPause() {
    if (mPaused) {
        return;
    }
    mPaused = true;
    if (mSource != NULL) {
        mSource->pause();
    }
    if (mRenderer != NULL) {
        mRenderer->pause();
    }
}
```

NuPlayer::onResume
这个函数实现恢复功能，代码逻辑跟onPause差不多，把Source和Render恢复，还可能涉及其它操作。代码如下：

``` cpp
void NuPlayer::onResume() {
    if (!mPaused || mResetting) {
        return;
    }
    mPaused = false;
    if (mSource != NULL) {
        mSource->resume();
    }
    // |mAudioDecoder| may have been released due to the pause timeout, so re-create it if
    // needed.
    if (audioDecoderStillNeeded() && mAudioDecoder == NULL) {
        instantiateDecoder(true /* audio */, &mAudioDecoder);
    }
    if (mRenderer != NULL) {
        mRenderer->resume();
    }
}
```

NuPlayer::onStart
这个接口实现启动的操作，相对复杂点，需要初始化解码器、初始化Render、设置Source状态，并将三者关联起来。代码如下：

``` cpp
void NuPlayer::onStart(int64_t startPositionUs) {
    if (!mSourceStarted) {
        mSourceStarted = true;
        mSource->start(); // 设置Source状态
    }
    
    // ... (省略部分代码)

    sp<AMessage> notify = new AMessage(kWhatRendererNotify, this);
    ++mRendererGeneration; // 创建Render和RenderLooper，属性设置、与解码器关联
    notify->setInt32("generation", mRendererGeneration);
    mRenderer = new Renderer(mAudioSink, notify, flags);
    mRendererLooper = new ALooper;
    mRendererLooper->setName("NuPlayerRenderer");
    mRendererLooper->start(false, false, ANDROID_PRIORITY_AUDIO);
    mRendererLooper->registerHandler(mRenderer);

    status_t err = mRenderer->setPlaybackSettings(mPlaybackSettings);

    float rate = getFrameRate();
    if (rate > 0) {
        mRenderer->setVideoFrameRate(rate);
    }

    if (mVideoDecoder != NULL) {
        mVideoDecoder->setRenderer(mRenderer);
    }
    if (mAudioDecoder != NULL) {
        mAudioDecoder->setRenderer(mRenderer);
    }

    postScanSources();
}
```

上面代码中没有解码器的初始化，那只能继续看看postScanSources代码了。看实现发现就是发送了kWhatScanSources消息。那么消息循环里边是怎么处理的呢？

``` cpp
case kWhatScanSources:
{
    int32_t generation;
    CHECK(msg->findInt32("generation", &generation));
    if (generation != mScanSourcesGeneration) {
        // Drop obsolete msg.
        break;
    }

    mScanSourcesPending = false;
    bool mHadAnySourcesBefore = (mAudioDecoder != NULL) || (mVideoDecoder != NULL);
    bool rescan = false;

    // initialize video before audio because successful initialization of
    // video may change deep buffer mode of audio.
    if (mSurface != NULL) { // 初始化视频解码器
        if (instantiateDecoder(false, &mVideoDecoder) == -EWOULDBLOCK) {
            rescan = true;
        }
    }

    // Don't try to re-open audio sink if there's an existing decoder.
    if (mAudioSink != NULL && mAudioDecoder == NULL) { // 初始化音频解码器
        if (instantiateDecoder(true, &mAudioDecoder) == -EWOULDBLOCK) {
            rescan = true;
        }
    }

    if (!mHadAnySourcesBefore && (mAudioDecoder != NULL || mVideoDecoder != NULL)) {
        // This is the first time we've found anything playable.
        // 设置定期查询时长
        if (mSourceFlags & Source::FLAG_DYNAMIC_DURATION) {
            schedulePollDuration();
        }
    }

    status_t err; // 一些异常处理逻辑
    if ((err = mSource->feedMoreTSData()) != OK) {
        if (mAudioDecoder == NULL && mVideoDecoder == NULL) {
            // We're not currently decoding anything (no audio or
            // video tracks found) and we just ran out of input data.

            if (err == ERROR_END_OF_STREAM) {
                notifyListener(MEDIA_PLAYBACK_COMPLETE, 0, 0);
            } else {
                notifyListener(MEDIA_ERROR, MEDIA_ERROR_UNKNOWN, err);
            }
        }
        break;
    }
    // 如果需要的话，重新扫描Source
    if (rescan) {
        msg->post(100000ll);
        mScanSourcesPending = true;
    }
    break;
}
```

此外还有seekToAsync()、resetAsync()、getCurrentPosition()、getFileMeta()。由于实现类似，就不一一介绍了。
##### 4.4、小结结和疑问
到这里，我们已经把NuPlayer主要的函数分析完了，但是问题依旧在。比如下面几个：

> **不同格式的多媒体文件如何探测并解析的？音视频数据缓冲区在哪里？（Source）**
> **视频如何显示的？音频如何播放的？音视频同步在哪里？（Renderer）**
> **音频解码线程、视频解码线程在哪里？ （DecoderBase）**

我想接下来的分析就是解决这些疑问的。

##### 4.5、Codec Encoder 、Decoder列表附录
``` cpp
[AOSP/device/qcom/msm8996/media_codecs.xml(system/etc/media_codecs.xml)]
<!--
 8996 Decoder capabilities
 __________________________________________________________________
 | Codec    | W       H       fps     Mbps    MB/s    | Secure-dec |
 |__________|_________________________________________|____________|
 | h264     | 3840    2160    60      100     1958400 |  Y         |
 |          | (4096)  (2160)  (56)    (100)           |            |
 | hevc     | 3840    2160    60      100     1958400 |  Y         |
 |          | (4096)  (2160)  (56)    (100)           |            |
 | mpeg4    | 1920    1088    60      60      489600  |  N         |
 | vc1      | 1920    1088    60      60      489600  |  Y         |
 | vp8      | 3840    2160    30      100     979200  |  N         |
 | vp9      | 3840    2160    30      100     979200  |  Y         |
 | divx3    | 720     480     30      2       40500   |  N         |
 | div4/5/6 | 1920    1088    30      10      244800  |  N         |
 | h263     | 864     480     30      2       48600   |  N         |
 | mpeg2    | 1920    1088    30      40      244800  |  Y         |
 |__________|_________________________________________|____________|


 8996 Encoder capabilities
 ______________________________________________________
 | Codec    | W       H       fps     Mbps    MB/s    |
 |__________|_________________________________________|
 | h264     | 3840    2160    30      100     979200  |
 | hevc     | 3840    2160    30      100     979200  |
 | mpeg4    | 1920    1088    60      60      489600  |
 | vp8      | 3840    2160    30      100     979200  |
 | h263     | 864     480     30      2       48600   |
 |__________|_________________________________________|
-->

<MediaCodecs>
    <Include href="media_codecs_google_audio.xml" />
    <Include href="media_codecs_google_telephony.xml" />
    <Settings>
        <Setting name="max-video-encoder-input-buffers" value="11" />
    </Settings>
    <Encoders>
        <!-- Audio Hardware  -->
        <!-- Audio Software  -->
        <!-- Video Hardware  -->
        <MediaCodec name="OMX.qcom.video.encoder.avc" type="video/avc" >
            <Quirk name="requires-allocate-on-input-ports" />
            <Quirk name="requires-allocate-on-output-ports" />
            <Quirk name="requires-loaded-to-idle-after-allocation" />
            <Limit name="size" min="96x64" max="4096x2160" />
            <Limit name="alignment" value="2x2" />
            <Limit name="block-size" value="16x16" />
            <Limit name="blocks-per-second" min="1" max="979200" />
            <Limit name="bitrate" range="1-100000000" />
            <Limit name="frame-rate" range="1-240" />
            <Limit name="concurrent-instances" max="16" />
          ......
        </MediaCodec>
        <MediaCodec name="OMX.qcom.video.encoder.mpeg4" type="video/mp4v-es" >
            ......
        </MediaCodec>
        <MediaCodec name="OMX.qcom.video.encoder.h263" type="video/3gpp" >
            ......
        </MediaCodec>
        <MediaCodec name="OMX.qcom.video.encoder.vp8" type="video/x-vnd.on2.vp8" >
            ......
        </MediaCodec>
        <MediaCodec name="OMX.qcom.video.encoder.hevc" type="video/hevc" >
            ......
        </MediaCodec>
    </Encoders>
    <Decoders>
       <!-- Video Hardware  -->
        <MediaCodec name="OMX.qcom.video.decoder.avc" type="video/avc" >
            <Quirk name="requires-allocate-on-input-ports" />
            <Quirk name="requires-allocate-on-output-ports" />
            <Limit name="size" min="64x64" max="4096x2160" />
            <Limit name="alignment" value="2x2" />
            <Limit name="block-size" value="16x16" />
            <Limit name="blocks-per-second" min="1" max="1958400" />
            <Limit name="bitrate" range="1-100000000" />
            <Limit name="frame-rate" range="1-240" />
            <Limit name="vt-version" value="65537" />
            <Limit name="vt-low-latency" value="1" />
            <Limit name="vt-max-macroblock-processing-rate" value="972000" />
            <Limit name="vt-max-level" value="52" />
            <Limit name="vt-max-instances" value="16" />
            <Feature name="adaptive-playback" />
            <Limit name="concurrent-instances" max="16" />
        </MediaCodec>
        <MediaCodec name="OMX.qcom.video.decoder.avc.secure" type="video/avc" >
            ......
        </MediaCodec>
        <MediaCodec name="OMX.qcom.video.decoder.mpeg4" type="video/mp4v-es" >
            ......
        </MediaCodec>
        <MediaCodec name="OMX.qcom.video.decoder.mpeg2" type="video/mpeg2" >
            ......
        </MediaCodec>
        <MediaCodec name="OMX.qcom.video.decoder.mpeg2.secure" type="video/mpeg2" >
            ......
        </MediaCodec>
        <MediaCodec name="OMX.qcom.video.decoder.h263" type="video/3gpp" >
            ......
        </MediaCodec>
        <MediaCodec name="OMX.qcom.video.decoder.vc1" type="video/x-ms-wmv" >
            ......
        </MediaCodec>
        <MediaCodec name="OMX.qcom.video.decoder.vc1.secure" type="video/x-ms-wmv" >
            ......
        </MediaCodec>
        <MediaCodec name="OMX.qcom.video.decoder.divx" type="video/divx" >
            ......
        </MediaCodec>
        <MediaCodec name="OMX.qcom.video.decoder.divx311" type="video/divx311" >
            ......
        </MediaCodec>
        <MediaCodec name="OMX.qcom.video.decoder.divx4" type="video/divx4" >
            ......
        </MediaCodec>
        <MediaCodec name="OMX.qcom.video.decoder.vp8" type="video/x-vnd.on2.vp8" >
            ......
        </MediaCodec>
        <MediaCodec name="OMX.qcom.video.decoder.vp9" type="video/x-vnd.on2.vp9" >
            ......
        </MediaCodec>
        <MediaCodec name="OMX.qcom.video.decoder.vp9.secure" type="video/x-vnd.on2.vp9" >
            ......
        </MediaCodec>
        <MediaCodec name="OMX.qcom.video.decoder.hevc" type="video/hevc" >
            ......
        </MediaCodec>
        <MediaCodec name="OMX.qcom.video.decoder.hevc.secure" type="video/hevc" >
            ......
        </MediaCodec>
        <!-- Audio Software  -->
        <MediaCodec name="OMX.qti.audio.decoder.flac" type="audio/flac" />
    </Decoders>
    <Include href="media_codecs_google_video.xml" />
</MediaCodecs>
```

#### （五）、参考资料(特别感谢各位前辈的分析和图示)：
[Android NuPlayer播放框架 ](http://www.cnblogs.com/tocy/tag/%E6%92%AD%E6%94%BE%E6%A1%86%E6%9E%B6/)
[专栏：MultiMedia框架总结(基于6.0源码) - CSDN博客](https://blog.csdn.net/column/details/12812.html)
[Android多媒体开发-归档 | April is your lie](windrunnerlihuan.com)
[Android-7.0-Nuplayer概述 - CSDN博客](https://blog.csdn.net/miaomiao12345678/article/details/57415505)
[Android-7.0-MediaPlayer状态机 - CSDN博客](https://blog.csdn.net/miaomiao12345678/article/details/57409783)
[Android-7.0-Nuplayer-启动流程 - CSDN博客](https://blog.csdn.net/miaomiao12345678/article/details/56961370)
[YUV - 维基百科，自由的百科全书](https://zh.wikipedia.org/wiki/YUV)
[Android Media Player 框架分析-Nuplayer（1） - CSDN博客](https://blog.csdn.net/harman_zjc/article/details/53386010)
[Android Media Player 框架分析-AHandler AMessage ALooper - CSDN博客](https://blog.csdn.net/harman_zjc/article/details/53397945)
[Android 4.2.2 stagefright架构 - CSDN博客](https://blog.csdn.net/gzzaigcnforever/article/category/2135451/1?)
[android4.2.2的stagefright架构下基于SurfaceFlinger的视频解码输出缓存创建机制 - CSDN博客](https://blog.csdn.net/gzzaigcnforever/article/details/26849549)
[husanlim 的专栏 参考 - CSDN博客](https://blog.csdn.net/dfhuang09/article/category/7610722)
[android ACodec MediaCodec NuPlayer flow - CSDN博客](https://blog.csdn.net/dfhuang09/article/details/54926526)
[android MediaCodec ACodec - CSDN博客](https://blog.csdn.net/dfhuang09/article/details/60132620)
