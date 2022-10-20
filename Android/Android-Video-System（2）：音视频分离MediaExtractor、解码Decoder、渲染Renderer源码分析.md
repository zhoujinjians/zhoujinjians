---
title: Android Video System（2）：音视频分离MediaExtractor、解码Decoder、渲染Renderer源码分析
cover: https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/hexo.themes/bing-wallpaper-2018.04.17.jpg
categories:
  - Multimedia
tags:
  - Android
  - Video
  - Multimedia
toc: true
abbrlink: 20190209
date: 2019-02-09 09:25:00
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

前面第一章节已经分析过mMediaPlayer.setDataSource()、mMediaPlayer.setDisplay()下来的分析尝试分析解答如下疑问：
> **不同格式的多媒体文件如何探测并解析的？音视频数据缓冲区在哪里？（Source）**
> **音频解码线程、视频解码线程在哪里？ （DecoderBase）**
> **视频如何显示的？音频如何播放的？音视频同步在哪里？（Renderer）**
#### （一）、多媒体文件解析 - MediaExtractor分离音视频
接下来继续分析mMediaPlayer.prepareAsync()
##### 1.1、mMediaPlayer.prepareAsync()

``` java
[->\frameworks\base\media\java\android\media\MediaPlayer.java]
public native void prepareAsync() throws IllegalStateException;
```
通过JNI调用

``` cpp
[->\frameworks\base\media\jni\android_media_MediaPlayer.cpp]
static void
android_media_MediaPlayer_prepareAsync(JNIEnv *env, jobject thiz)
{
    sp<MediaPlayer> mp = getMediaPlayer(env, thiz);
    if (mp == NULL ) {
        jniThrowException(env, "java/lang/IllegalStateException", NULL);
        return;
    }

    // Handle the case where the display surface was set before the mp was
    // initialized. We try again to make it stick.
    sp<IGraphicBufferProducer> st = getVideoSurfaceTexture(env, thiz);
    mp->setVideoSurfaceTexture(st);

    process_media_player_call( env, thiz, mp->prepareAsync(), "java/io/IOException", "Prepare Async failed." );
}
```
首先设置视频的 display surface（关于IGraphicBufferProducer相关知识请参考：Android 7.1.2 (Android N) Android Graphics 系统 分析 [i.wonder~]），

##### 1.1.1、MediaPlayer.setVideoSurfaceTexture()
``` cpp
[->\frameworks\av\media\libmedia\mediaplayer.cpp]
status_t MediaPlayer::setVideoSurfaceTexture(
        const sp<IGraphicBufferProducer>& bufferProducer)
{
    ALOGV("setVideoSurfaceTexture");
    Mutex::Autolock _l(mLock);
    if (mPlayer == 0) return NO_INIT;
    return mPlayer->setVideoSurfaceTexture(bufferProducer);
}
```

前面setDataSource()分析过，此处会调用NuPlayer的setVideoSurfaceTexture()函数

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
此处首先构造了一个AMessage消息，然后new Surface()，接下来看看消息处理过程。
``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayer.cpp]
case kWhatSetVideoSurface:
        {
            sp<RefBase> obj;
            CHECK(msg->findObject("surface", &obj));
            sp<Surface> surface = static_cast<Surface *>(obj.get());

            if (mSource == NULL || !mStarted || mSource->getFormat(false /* audio */) == NULL
                    // NOTE: mVideoDecoder's mSurface is always non-null
                    || (mVideoDecoder != NULL && mVideoDecoder->setVideoSurface(surface) == OK)) {
                performSetSurface(surface);
                break;
            }

            mDeferredActions.push_back(
                    new FlushDecoderAction(FLUSH_CMD_FLUSH /* audio */,
                                           FLUSH_CMD_SHUTDOWN /* video */));

            mDeferredActions.push_back(new SetSurfaceAction(surface));

            if (obj != NULL || mAudioDecoder != NULL) {
                if (mStarted) {
                    int64_t currentPositionUs = 0;
                    if (getCurrentPosition(&currentPositionUs) == OK) {
                        mDeferredActions.push_back(
                                new SeekAction(currentPositionUs));
                    }
                }
                mDeferredActions.push_back(
                        new SimpleAction(&NuPlayer::performScanSources));
            }

            mDeferredActions.push_back(
                    new ResumeDecoderAction(false /* needNotify */));

            processDeferredActions();
            break;
        }

void NuPlayer::performSetSurface(const sp<Surface> &surface) {
    ALOGV("performSetSurface");

    mSurface = surface;

    // XXX - ignore error from setVideoScalingMode for now
    setVideoScalingMode(mVideoScalingMode);

    if (mDriver != NULL) {
        sp<NuPlayerDriver> driver = mDriver.promote();
        if (driver != NULL) {
            driver->notifySetSurfaceComplete();
        }
    }
}
```
可以看到将surface 赋值给NuPlayer的mSurface ，待视频解码后就可以在此surface 上渲染画面了，
这个稍后再作分析。

##### 1.1.2、MediaPlayer.prepareAsync()
然后接着调用MediaPlayer prepareAsync()函数。
``` cpp
[->\frameworks\av\media\libmedia\mediaplayer.cpp]
status_t MediaPlayer::prepareAsync()
{
    ALOGV("prepareAsync");
    Mutex::Autolock _l(mLock);
    return prepareAsync_l();
}

status_t MediaPlayer::prepareAsync_l()
{
    if ( (mPlayer != 0) && ( mCurrentState & (MEDIA_PLAYER_INITIALIZED | MEDIA_PLAYER_STOPPED) ) ) {
        if (mAudioAttributesParcel != NULL) {
            mPlayer->setParameter(KEY_PARAMETER_AUDIO_ATTRIBUTES, *mAudioAttributesParcel);
        } else {
            mPlayer->setAudioStreamType(mStreamType);
        }
        mCurrentState = MEDIA_PLAYER_PREPARING;
        return mPlayer->prepareAsync();
    }
    ALOGE("prepareAsync called in state %d, mPlayer(%p)", mCurrentState, mPlayer.get());
    return INVALID_OPERATION;
}
```
此处会调用NuPlayer的prepareAsync()函数，prepareAsync()发送了一个kWhatPrepare的AMessage，我们直接看看消息处理。

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayer.cpp]
void NuPlayer::prepareAsync() {
    (new AMessage(kWhatPrepare, this))->post();
}        
case kWhatPrepare:
{
    mSource->prepareAsync();
    break;
}
```
此处又调用了GenericSource的prepareAsync()函数，发送了一个kWhatPrepareAsync消息。直接看看GenericSource如何处理的

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\GenericSource.cpp]
void NuPlayer::GenericSource::prepareAsync() {
    if (mLooper == NULL) {
        mLooper = new ALooper;
        mLooper->setName("generic");
        mLooper->start();

        mLooper->registerHandler(this);
    }

    sp<AMessage> msg = new AMessage(kWhatPrepareAsync, this);
    msg->post();
}
    switch (msg->what()) {
      case kWhatPrepareAsync:
      {
          onPrepareAsync();
          break;
      }
void NuPlayer::GenericSource::onPrepareAsync() {
    // delayed data source creation
    if (mDataSource == NULL) {
        // set to false first, if the extractor
        // comes back as secure, set it to true then.
        mIsSecure = false;

        if (!mUri.empty()) {
            const char* uri = mUri.c_str();
            String8 contentType;
            mIsWidevine = !strncasecmp(uri, "widevine://", 11);

            if (!strncasecmp("http://", uri, 7)
                    || !strncasecmp("https://", uri, 8)
                    || mIsWidevine) {
                mHttpSource = DataSource::CreateMediaHTTP(mHTTPService);
                ......
            }

            mDataSource = DataSource::CreateFromURI(
                   mHTTPService, uri, &mUriHeaders, &contentType,
                   static_cast<HTTPBase *>(mHttpSource.get()));
        } else {
            mIsWidevine = false;

            mDataSource = new FileSource(mFd, mOffset, mLength);
            mFd = -1;
        }

       ......
    }

    if (mDataSource->flags() & DataSource::kIsCachingDataSource) {
        mCachedSource = static_cast<NuCachedSource2 *>(mDataSource.get());
    }

    mIsStreaming = (mIsWidevine || mCachedSource != NULL);

    // init extractor from data source
    status_t err = initFromDataSource();

    if (mVideoTrack.mSource != NULL) {
        sp<MetaData> meta = doGetFormatMeta(false /* audio */);
        sp<AMessage> msg = new AMessage;
        err = convertMetaDataToMessage(meta, &msg);
        ......
        notifyVideoSizeChanged(msg);
    }

    ......
    if (mIsSecure) {
        // secure decoders must be instantiated before starting widevine source
        sp<AMessage> reply = new AMessage(kWhatSecureDecodersInstantiated, this);
        notifyInstantiateSecureDecoders(reply);
    } else {
        finishPrepareAsync();
    }
}
```
首先构造了  mDataSource = new FileSource，然后调用了initFromDataSource()，这里面包含多媒体文件格式探测，。

##### 1.1.3、GenericSource.initFromDataSource()

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\GenericSource.cpp]
status_t NuPlayer::GenericSource::initFromDataSource() {
    sp<IMediaExtractor> extractor;
    String8 mimeType;
    float confidence;
    sp<AMessage> dummy;
    bool isWidevineStreaming = false;

    CHECK(mDataSource != NULL);

    if (mIsWidevine) {
        ......
    } else if (mIsStreaming) {
        if (!mDataSource->sniff(&mimeType, &confidence, &dummy)) {
            return UNKNOWN_ERROR;
        }
        isWidevineStreaming = !strcasecmp(
                mimeType.string(), MEDIA_MIMETYPE_CONTAINER_WVM);
    }

    if (isWidevineStreaming) {
        ......
    } else {
        extractor = MediaExtractor::Create(mDataSource,
                mimeType.isEmpty() ? NULL : mimeType.string());
    }

    ......
    if (extractor->getDrmFlag()) {
        checkDrmStatus(mDataSource);
    }

    mFileMeta = extractor->getMetaData();
    if (mFileMeta != NULL) {
        int64_t duration;
        if (mFileMeta->findInt64(kKeyDuration, &duration)) {
            mDurationUs = duration;
        }......
    }

    int32_t totalBitrate = 0;

    size_t numtracks = extractor->countTracks();
    
    for (size_t i = 0; i < numtracks; ++i) {
        sp<IMediaSource> track = extractor->getTrack(i);
        
        sp<MetaData> meta = extractor->getTrackMetaData(i);
       
        const char *mime;
        CHECK(meta->findCString(kKeyMIMEType, &mime));
        
        // Do the string compare immediately with "mime",
        // we can't assume "mime" would stay valid after another
        // extractor operation, some extractors might modify meta
        // during getTrack() and make it invalid.
        if (!strncasecmp(mime, "audio/", 6)) {
            if (mAudioTrack.mSource == NULL) {
                mAudioTrack.mIndex = i;
                mAudioTrack.mSource = track;
                mAudioTrack.mPackets =
                    new AnotherPacketSource(mAudioTrack.mSource->getFormat());

                if (!strcasecmp(mime, MEDIA_MIMETYPE_AUDIO_VORBIS)) {
                    mAudioIsVorbis = true;
                } else {
                    mAudioIsVorbis = false;
                }
            }
        } else if (!strncasecmp(mime, "video/", 6)) {
            if (mVideoTrack.mSource == NULL) {
                mVideoTrack.mIndex = i;
                mVideoTrack.mSource = track;
                mVideoTrack.mPackets =
                    new AnotherPacketSource(mVideoTrack.mSource->getFormat());

                // check if the source requires secure buffers
                int32_t secure;
                if (meta->findInt32(kKeyRequiresSecureBuffers, &secure)
                        && secure) {
                    mIsSecure = true;
                    if (mUIDValid) {
                        extractor->setUID(mUID);
                    }
                }
            }
        }

        mSources.push(track);
        int64_t durationUs;
        if (meta->findInt64(kKeyDuration, &durationUs)) {
            if (durationUs > mDurationUs) {
                mDurationUs = durationUs;
            }
        }

        int32_t bitrate;
        if (totalBitrate >= 0 && meta->findInt32(kKeyBitRate, &bitrate)) {
            totalBitrate += bitrate;
        } else {
            totalBitrate = -1;
        }
    }
   ......
    mBitrate = totalBitrate;
    return OK;
}
```
可以看到通过MediaExtractor::Create()得到MediaExtractor，然后将数据解析成track 赋值给mAudioTrack.mSource、mVideoTrack.mSource。
##### 1.1.4、MediaExtractor::Create()

``` cpp
[->\frameworks\av\media\libstagefright\MediaExtractor.cpp]
sp<IMediaExtractor> MediaExtractor::Create(
        const sp<DataSource> &source, const char *mime) {
    ALOGV("MediaExtractor::Create %s", mime);

    char value[PROPERTY_VALUE_MAX];
    if (property_get("media.stagefright.extractremote", value, NULL)
            && (!strcmp("0", value) || !strcasecmp("false", value))) {
        // local extractor
        ALOGW("creating media extractor in calling process");
        return CreateFromService(source, mime);
    } else {
        // Check if it's WVM, since WVMExtractor needs to be created in the media server process,
        // not the extractor process.
        String8 mime8;
        float confidence;
        sp<AMessage> meta;
        if (SniffWVM(source, &mime8, &confidence, &meta) &&
                !strcasecmp(mime8, MEDIA_MIMETYPE_CONTAINER_WVM)) {
            return new WVMExtractor(source);
        }
        ......
        if (SniffDRM(source, &mime8, &confidence, &meta)) {
            const char *drmMime = mime8.string();
            ALOGV("Detected media content as '%s' with confidence %.2f", drmMime, confidence);
            if (!strncmp(drmMime, "drm+es_based+", 13)) {
                // DRMExtractor sets container metadata kKeyIsDRM to 1
                return new DRMExtractor(source, drmMime + 14);
            }
        }

        // remote extractor
        ALOGV("get service manager");
        sp<IBinder> binder = defaultServiceManager()->getService(String16("media.extractor"));

        if (binder != 0) {
            sp<IMediaExtractorService> mediaExService(interface_cast<IMediaExtractorService>(binder));
            sp<IMediaExtractor> ex = mediaExService->makeExtractor(RemoteDataSource::wrap(source), mime);
            return ex;
        } else {
            ......
        }
    }
    return NULL;
}
```
可以看到通过Binder通信获取"media.extractor"服务得到一个Extractor。

##### 1.1.5、IMediaExtractor->getTrack()
根据不同类别解析出不同的Track

``` cpp
[->\frameworks\av\media\libstagefright\]

AACExtractor.cpp sp<IMediaSource> AACExtractor::getTrack(size_t index)
AMRExtractor.cpp sp<IMediaSource> AMRExtractor::getTrack(size_t index)
MP3Extractor.cpp sp<IMediaSource> MP3Extractor::getTrack(size_t index)
NuMediaExtractor.cpp sp<IMediaSource> source = mImpl->getTrack(index)
WAVExtractor.cpp sp<IMediaSource> WAVExtractor::getTrack(size_t index)
FLACExtractor.cpp sp<IMediaSource> FLACExtractor::getTrack(size_t index) 
StagefrightMetadataRetriever.cpp sp<IMediaSource> source = mExtractor->getTrack(i)
AVIExtractor.cpp sp<MediaSource> AVIExtractor::getTrack(size_t index)
OggExtractor.cpp sp<IMediaSource> OggExtractor::getTrack(size_t index)
MPEG4Extractor.cpp sp<IMediaSource> MPEG4Extractor::getTrack(size_t index)

//MP3
sp<IMediaSource> MP3Extractor::getTrack(size_t index) {
    return new MP3Source(
            mMeta, mDataSource, mFirstFramePos, mFixedHeader,
            mSeeker);
}
//MPEG4
sp<IMediaSource> MPEG4Extractor::getTrack(size_t index) {
    status_t err;
    ......
    Track *track = mFirstTrack;
    while (index > 0) {
        if (track == NULL) {
            return NULL;
        }

        track = track->next;
        --index;
    }
    ......
    Trex *trex = NULL;
    int32_t trackId;
    if (track->meta->findInt32(kKeyTrackID, &trackId)) {
        for (size_t i = 0; i < mTrex.size(); i++) {
            Trex *t = &mTrex.editItemAt(i);
            if (t->track_ID == (uint32_t) trackId) {
                trex = t;
                break;
            }
        }
    } else {
        ......
    }
    const char *mime;
    .......
    if (!strcasecmp(mime, MEDIA_MIMETYPE_VIDEO_AVC)) {
        uint32_t type;
        const void *data;
        size_t size;
        if (!track->meta->findData(kKeyAVCC, &type, &data, &size)) {
            return NULL;
        }
        const uint8_t *ptr = (const uint8_t *)data;
        ......
    } else if (!strcasecmp(mime, MEDIA_MIMETYPE_VIDEO_HEVC)) {
        uint32_t type;
        const void *data;
        size_t size;
        if (!track->meta->findData(kKeyHVCC, &type, &data, &size)) {
            return NULL;
        }

        const uint8_t *ptr = (const uint8_t *)data;
       ......
    }
    return new MPEG4Source(this,
            track->meta, mDataSource, track->timescale, track->sampleTable,
            mSidxEntries, trex, mMoofOffset);
}
```
得到不同格式的 MP3Extractor、MPEG4Source ......

还记的前面提出的第一点疑问吗，现在我们知道了如何分离音视频了并且得到了相应的文件Source了。
图示（红线部分）：

![Alt text](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/video.system/02-01-source-demux-decoder-output-MediaExtractor.jpg)

#### （二）、多媒体文件 - 音视频解码（Decoder）
音频解码、视频解码在何处，答案就在mMediaPlayer.start()流程当中，先看看start()总体时序图，然后一步步分析

![Alt text](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/video.system/02-02-NuPlayer-Start-instantiateDecoder.png)

由于从Java层到JNI前面已多次分析，这里直接从NuPlayer::start()开始分析

##### 2.1、NuPlayer::start()

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayer.cpp]
void NuPlayer::start() {
    (new AMessage(kWhatStart, this))->post();
}
        case kWhatStart:
        {
            ALOGV("kWhatStart");
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
void NuPlayer::onStart(int64_t startPositionUs) {
    if (!mSourceStarted) {
        mSourceStarted = true;
        mSource->start();
    }
    if (startPositionUs > 0) {
        performSeek(startPositionUs);
        if (mSource->getFormat(false /* audio */) == NULL) {
            return;
        }
    }

    mOffloadAudio = false;
    mAudioEOS = false;
    mVideoEOS = false;
    mStarted = true;
    mPaused = false;

    uint32_t flags = 0;

    if (mSource->isRealTime()) {
        flags |= Renderer::FLAG_REAL_TIME;
    }

    sp<MetaData> audioMeta = mSource->getFormatMeta(true /* audio */);
    sp<MetaData> videoMeta = mSource->getFormatMeta(false /* audio */);
    ......
    ALOGV_IF(audioMeta == NULL, "no metadata for audio source");  // video only stream

    audio_stream_type_t streamType = AUDIO_STREAM_MUSIC;
    if (mAudioSink != NULL) {
        streamType = mAudioSink->getAudioStreamType();
    }

    sp<AMessage> videoFormat = mSource->getFormat(false /* audio */);

    mOffloadAudio =
        canOffloadStream(audioMeta, (videoFormat != NULL), mSource->isStreaming(), streamType)
                && (mPlaybackSettings.mSpeed == 1.f && mPlaybackSettings.mPitch == 1.f);
    if (mOffloadAudio) {
        flags |= Renderer::FLAG_OFFLOAD_AUDIO;
    }

    sp<AMessage> notify = new AMessage(kWhatRendererNotify, this);
    ++mRendererGeneration;
    notify->setInt32("generation", mRendererGeneration);
    mRenderer = new Renderer(mAudioSink, notify, flags);
    mRendererLooper = new ALooper;
    mRendererLooper->setName("NuPlayerRenderer");
    mRendererLooper->start(false, false, ANDROID_PRIORITY_AUDIO);
    mRendererLooper->registerHandler(mRenderer);

    status_t err = mRenderer->setPlaybackSettings(mPlaybackSettings);
    ......
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
这里创建了名为NuPlayerRenderer的Renderer对象，然后启动循环，看看初始化

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayerRenderer.cpp]
NuPlayer::Renderer::Renderer(
        const sp<MediaPlayerBase::AudioSink> &sink,
        const sp<AMessage> &notify,
        uint32_t flags)
    : mAudioSink(sink),
      mUseVirtualAudioSink(false),
      mNotify(notify),
      mFlags(flags),
      mNumFramesWritten(0),
      mDrainAudioQueuePending(false),
      mDrainVideoQueuePending(false),
      mAudioQueueGeneration(0),
      mVideoQueueGeneration(0),
      mAudioDrainGeneration(0),
      mVideoDrainGeneration(0),
      mAudioEOSGeneration(0),
      mPlaybackSettings(AUDIO_PLAYBACK_RATE_DEFAULT),
      ......
      mWakeLock(new AWakeLock()) {
    mMediaClock = new MediaClock;
    mPlaybackRate = mPlaybackSettings.mSpeed;
    mMediaClock->setPlaybackRate(mPlaybackRate);
}
```
##### 2.2、postScanSources()

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayer.cpp]
void NuPlayer::postScanSources() {
    sp<AMessage> msg = new AMessage(kWhatScanSources, this);
    msg->setInt32("generation", mScanSourcesGeneration);
    msg->post();

    mScanSourcesPending = true;
}

        case kWhatScanSources:
        {
            int32_t generation;
            mScanSourcesPending = false;

            bool mHadAnySourcesBefore =
                (mAudioDecoder != NULL) || (mVideoDecoder != NULL);
            bool rescan = false;

            // initialize video before audio because successful initialization of
            // video may change deep buffer mode of audio.
            if (mSurface != NULL) {
                if (instantiateDecoder(false, &mVideoDecoder) == -EWOULDBLOCK) {
                    rescan = true;
                }
            }

            // Don't try to re-open audio sink if there's an existing decoder.
            if (mAudioSink != NULL && mAudioDecoder == NULL) {
                if (instantiateDecoder(true, &mAudioDecoder) == -EWOULDBLOCK) {
                    rescan = true;
                }
            }

            ......
        }
```
此处调用了instantiateDecoder()来初始化音视频解码器Decoder
##### 2.3、instantiateDecoder()

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayer.cpp]
status_t NuPlayer::instantiateDecoder(
        bool audio, sp<DecoderBase> *decoder, bool checkAudioModeChange) {
    ......
    sp<AMessage> format = mSource->getFormat(audio);

    format->setInt32("priority", 0 /* realtime */);

    ......

    if (audio) {
        sp<AMessage> notify = new AMessage(kWhatAudioNotify, this);
        ++mAudioDecoderGeneration;
        notify->setInt32("generation", mAudioDecoderGeneration);

        if (checkAudioModeChange) {
            determineAudioModeChange(format);
        }
        if (mOffloadAudio) {
            mSource->setOffloadAudio(true /* offload */);

            const bool hasVideo = (mSource->getFormat(false /*audio */) != NULL);
            format->setInt32("has-video", hasVideo);
            *decoder = new DecoderPassThrough(notify, mSource, mRenderer);
        } else {
            mSource->setOffloadAudio(false /* offload */);

            *decoder = new Decoder(notify, mSource, mPID, mRenderer);
        }
    } else {
        sp<AMessage> notify = new AMessage(kWhatVideoNotify, this);
        ++mVideoDecoderGeneration;
        notify->setInt32("generation", mVideoDecoderGeneration);

        *decoder = new Decoder(
                notify, mSource, mPID, mRenderer, mSurface, mCCDecoder);

        // enable FRC if high-quality AV sync is requested, even if not
        // directly queuing to display, as this will even improve textureview
        // playback.
        {
            char value[PROPERTY_VALUE_MAX];
            if (property_get("persist.sys.media.avsync", value, NULL) &&
                    (!strcmp("1", value) || !strcasecmp("true", value))) {
                format->setInt32("auto-frc", 1);
            }
        }
    }
    (*decoder)->init();
    (*decoder)->configure(format);

    ......
    return OK;
}
```
##### 2.3.1、创建音视频解码器new Decoder()

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayerDecoder.cpp]
NuPlayer::Decoder::Decoder(
        const sp<AMessage> &notify,
        const sp<Source> &source,
        pid_t pid,
        const sp<Renderer> &renderer,
        const sp<Surface> &surface,
        const sp<CCDecoder> &ccDecoder)
    : DecoderBase(notify),
      mSurface(surface),
      mSource(source),
      mRenderer(renderer),
      mCCDecoder(ccDecoder),
      ......
      mVideoWidth(0),
      mVideoHeight(0),
      mIsAudio(true),
      ......
      mComponentName("decoder") {
    mCodecLooper = new ALooper;
    mCodecLooper->setName("NPDecoder-CL");
    mCodecLooper->start(false, false, ANDROID_PRIORITY_AUDIO);
    mVideoTemporalLayerAggregateFps[0] = mFrameRateTotal;
}
```
创建音视频解码器（NuPlayer::Decoder），为其创建名为NPDecoder-CL的mCodecLooper 【其父类NuPlayer::DecoderBase的构造中则会创建NPDecoder】

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayerDecoderBase.cpp]
NuPlayer::DecoderBase::DecoderBase(const sp<AMessage> &notify)
    :  mNotify(notify),
       mBufferGeneration(0),
       mPaused(false),
       mStats(new AMessage),
       mRequestInputBuffersPending(false) {
    // Every decoder has its own looper because MediaCodec operations
    // are blocking, but NuPlayer needs asynchronous operations.
    mDecoderLooper = new ALooper;
    mDecoderLooper->setName("NPDecoder");
    mDecoderLooper->start(false, false, ANDROID_PRIORITY_AUDIO);
}
```

##### 2.3.2、初始化Decoder->init()
对该解码器进行init()操作，调用NuPlayer::DecoderBase::init()为mDecoderLooper注册handler【init()和configure()都是NuPlayerDecoder继承自NuPlayer::DecoderBase的方法】

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayerDecoderBase.cpp]
void NuPlayer::DecoderBase::configure(const sp<AMessage> &format) {
    sp<AMessage> msg = new AMessage(kWhatConfigure, this);
    msg->setMessage("format", format);
    msg->post();
}

void NuPlayer::DecoderBase::init() {
    mDecoderLooper->registerHandler(this);
}

        case kWhatConfigure:
        {
            sp<AMessage> format;
            CHECK(msg->findMessage("format", &format));
            onConfigure(format);
            break;
        }
```
对该解码器进行configure(format)操作，调用NuPlayer::DecoderBase::configure(…)产生一个kWhatConfigure消息，然后消息处理中调用NuPlayer::Decoder::onConfigure(…)

##### 2.3.3、配置Decoder->configure()

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayerDecoder.cpp]
void NuPlayer::Decoder::onConfigure(const sp<AMessage> &format) {
  
    mFormatChangePending = false;
    mTimeChangePending = false;

    ++mBufferGeneration;

    AString mime;

    mIsAudio = !strncasecmp("audio/", mime.c_str(), 6);
    mIsVideoAVC = !strcasecmp(MEDIA_MIMETYPE_VIDEO_AVC, mime.c_str());

    mComponentName = mime;
    mComponentName.append(" decoder");
    ALOGV("[%s] onConfigure (surface=%p)", mComponentName.c_str(), mSurface.get());

    mCodec = MediaCodec::CreateByType(
            mCodecLooper, mime.c_str(), false /* encoder */, NULL /* err */, mPid);
    int32_t secure = 0;
    if (format->findInt32("secure", &secure) && secure != 0) {
        if (mCodec != NULL) {
            mCodec->getName(&mComponentName);
            mComponentName.append(".secure");
            mCodec->release();
            ALOGI("[%s] creating", mComponentName.c_str());
            mCodec = MediaCodec::CreateByComponentName(
                    mCodecLooper, mComponentName.c_str(), NULL /* err */, mPid);
        }
    }
    ......
    mIsSecure = secure;

    mCodec->getName(&mComponentName);

    status_t err;
    if (mSurface != NULL) {
        // disconnect from surface as MediaCodec will reconnect
        err = native_window_api_disconnect(
                mSurface.get(), NATIVE_WINDOW_API_MEDIA);
        // We treat this as a warning, as this is a preparatory step.
        // Codec will try to connect to the surface, which is where
        // any error signaling will occur.
        ALOGW_IF(err != OK, "failed to disconnect from surface: %d", err);
    }
    err = mCodec->configure(
            format, mSurface, NULL /* crypto */, 0 /* flags */);
    ......
    rememberCodecSpecificData(format);
    mStats->setString("mime", mime.c_str());
    mStats->setString("component-name", mComponentName.c_str());

    if (!mIsAudio) {
        int32_t width, height;
        if (mOutputFormat->findInt32("width", &width)
                && mOutputFormat->findInt32("height", &height)) {
            mStats->setInt32("width", width);
            mStats->setInt32("height", height);
        }
    }

    sp<AMessage> reply = new AMessage(kWhatCodecNotify, this);
    mCodec->setCallback(reply);

    err = mCodec->start();


    releaseAndResetMediaBuffers();

    mPaused = false;
    mResumePending = false;
}
```
在onConfigure中，首先会调用MediaCodec::CreateByType(…)或者MediaCodec::CreateByComponentName(…)根据情况创建MediaCodec，接着调用MediaCodec::init(…)，随后调用MediaCodec::configure(…)对MediaCodec进行配置使其转入Configured状态;然后又调用MediaCodec::start()使MediaCodec转入Executing状态。
##### 2.3.4、MediaCodec::init(…)

``` cpp
[->\frameworks\av\media\libstagefright\MediaCodec.cpp]
sp<MediaCodec> MediaCodec::CreateByType(
        const sp<ALooper> &looper, const AString &mime, bool encoder, status_t *err, pid_t pid) {
    sp<MediaCodec> codec = new MediaCodec(looper, pid);

    const status_t ret = codec->init(mime, true /* nameIsType */, encoder);
    return ret == OK ? codec : NULL; // NULL deallocates codec.
}

sp<MediaCodec> MediaCodec::CreateByComponentName(
        const sp<ALooper> &looper, const AString &name, status_t *err, pid_t pid) {
    sp<MediaCodec> codec = new MediaCodec(looper, pid);

    const status_t ret = codec->init(name, false /* nameIsType */, false /* encoder */);
    return ret == OK ? codec : NULL; // NULL deallocates codec.
}

sp<MediaCodec> MediaCodec::CreateByType(
        const sp<ALooper> &looper, const AString &mime, bool encoder, status_t *err, pid_t pid) {
    sp<MediaCodec> codec = new MediaCodec(looper, pid);

    const status_t ret = codec->init(mime, true /* nameIsType */, encoder);
    if (err != NULL) {
        *err = ret;
    }
    return ret == OK ? codec : NULL; // NULL deallocates codec.
}
status_t MediaCodec::init(const AString &name, bool nameIsType, bool encoder) {
    mResourceManagerService->init();

    // save init parameters for reset
    mInitName = name;
    mInitNameIsType = nameIsType;
    mInitIsEncoder = encoder;

    // Current video decoders do not return from OMX_FillThisBuffer
    // quickly, violating the OpenMAX specs, until that is remedied
    // we need to invest in an extra looper to free the main event
    // queue.

    mCodec = GetCodecBase(name, nameIsType);
    ......

    bool secureCodec = false;
    if (nameIsType && !strncasecmp(name.c_str(), "video/", 6)) {
        mIsVideo = true;
    } else {
        AString tmp = name;
        if (tmp.endsWith(".secure")) {
            secureCodec = true;
            tmp.erase(tmp.size() - 7, 7);
        }
        const sp<IMediaCodecList> mcl = MediaCodecList::getInstance();
        ......
        ssize_t codecIdx = mcl->findCodecByName(tmp.c_str());
        if (codecIdx >= 0) {
            const sp<MediaCodecInfo> info = mcl->getCodecInfo(codecIdx);
            Vector<AString> mimes;
            info->getSupportedMimes(&mimes);
            for (size_t i = 0; i < mimes.size(); i++) {
                if (mimes[i].startsWith("video/")) {
                    mIsVideo = true;
                    break;
                }
            }
        }
    }

    if (mIsVideo) {
        // video codec needs dedicated looper
        if (mCodecLooper == NULL) {
            mCodecLooper = new ALooper;
            mCodecLooper->setName("CodecLooper");
            mCodecLooper->start(false, false, ANDROID_PRIORITY_AUDIO);
        }

        mCodecLooper->registerHandler(mCodec);
    } else {
        mLooper->registerHandler(mCodec);
    }

    mLooper->registerHandler(this);

    mCodec->setNotificationMessage(new AMessage(kWhatCodecNotify, this));

    sp<AMessage> msg = new AMessage(kWhatInit, this);
    msg->setString("name", name);
    msg->setInt32("nameIsType", nameIsType);

    if (nameIsType) {
        msg->setInt32("encoder", encoder);
    }

    status_t err;
    Vector<MediaResource> resources;
    MediaResource::Type type =
            secureCodec ? MediaResource::kSecureCodec : MediaResource::kNonSecureCodec;
    MediaResource::SubType subtype =
            mIsVideo ? MediaResource::kVideoCodec : MediaResource::kAudioCodec;
    resources.push_back(MediaResource(type, subtype, 1));
    for (int i = 0; i <= kMaxRetry; ++i) {
        
        sp<AMessage> response;
        err = PostAndAwaitResponse(msg, &response);
  
    }
    return err;
}
```
##### 2.3.4.1、GetCodecBase
当编解码以"omx."开头则创建ACodec对象。
``` cpp
[->\frameworks\av\media\libstagefright\MediaCodec.cpp]
sp<CodecBase> MediaCodec::GetCodecBase(const AString &name, bool nameIsType) {
    // at this time only ACodec specifies a mime type.
    if (nameIsType || name.startsWithIgnoreCase("omx.")) {
        return new ACodec;
    } else if (name.startsWithIgnoreCase("android.filter.")) {
        return new MediaFilter;
    } else {
        return NULL;
    }
}
```

##### 2.3.4.2、MediaCodecList::getInstance()

``` cpp
[->\frameworks\av\media\libstagefright\MediaCodecList.cpp]
sp<IMediaCodecList> MediaCodecList::getInstance() {
    Mutex::Autolock _l(sRemoteInitMutex);
    if (sRemoteList == NULL) {
        sp<IBinder> binder =
            defaultServiceManager()->getService(String16("media.player"));
        sp<IMediaPlayerService> service =
            interface_cast<IMediaPlayerService>(binder);
        if (service.get() != NULL) {
            sRemoteList = service->getCodecList();
        }
    }
    return sRemoteList;
}

```
通过Binder通信获取MediaCodec列表。getCodecList()函数实现在

``` cpp
[->\frameworks\av\media\libmediaplayerservice\MediaPlayerService.cpp]
sp<IMediaCodecList> MediaPlayerService::getCodecList() const {
    return MediaCodecList::getLocalInstance();
}

```
``` cpp
[->\frameworks\av\media\libstagefright\MediaCodecList.cpp]
sp<IMediaCodecList> MediaCodecList::getLocalInstance() {
    Mutex::Autolock autoLock(sInitMutex);
    if (sCodecList == NULL) {
        MediaCodecList *codecList = new MediaCodecList;
        ......
    }
    return sCodecList;
}
MediaCodecList::MediaCodecList()
    : mInitCheck(NO_INIT),
      mUpdate(false),
      mGlobalSettings(new AMessage()) {
    parseTopLevelXMLFile("/etc/media_codecs.xml");
    parseTopLevelXMLFile("/etc/media_codecs_performance.xml", true/* ignore_errors */);
    parseTopLevelXMLFile(kProfilingResults, true/* ignore_errors */);
}
```
O(∩_∩)O哈哈~，终于分析到Codecs加载的地方了。还记得第一章节分析的附录吗，高通的音视频硬解码，这里再贴一下。

``` cpp
[AOSP/device/qcom/msm8996/media_codecs.xml(system/etc/media_codecs.xml)]

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
        <MediaCodec name="OMX.qcom.video.decoder.mpeg4" type="video/mp4v-es" >
        <MediaCodec name="OMX.qcom.video.decoder.h263" type="video/3gpp" >
        <MediaCodec name="OMX.qcom.video.decoder.vc1" type="video/x-ms-wmv" >
        <MediaCodec name="OMX.qcom.video.decoder.vc1.secure" type="video/x-ms-wmv" >
        <MediaCodec name="OMX.qcom.video.decoder.divx" type="video/divx" >
        <MediaCodec name="OMX.qcom.video.decoder.divx311" type="video/divx311" >
        <MediaCodec name="OMX.qcom.video.decoder.divx4" type="video/divx4" >
        <MediaCodec name="OMX.qcom.video.decoder.vp8" type="video/x-vnd.on2.vp8" >
        <MediaCodec name="OMX.qcom.video.decoder.vp9" type="video/x-vnd.on2.vp9" >
        <MediaCodec name="OMX.qcom.video.decoder.vp9.secure" type="video/x-vnd.on2.vp9" >
        <MediaCodec name="OMX.qcom.video.decoder.hevc" type="video/hevc" >
        <MediaCodec name="OMX.qcom.video.decoder.hevc.secure" type="video/hevc" >
        <!-- Audio Software  -->
        <MediaCodec name="OMX.qti.audio.decoder.flac" type="audio/flac" />
    </Decoders>
    <Include href="media_codecs_google_video.xml" />
```

##### 2.3.5、MediaCodec->configure()
产生kWhatConfigure消息，在消息处理中调用ACodec::initiateConfigureComponent(…)又产生消息kWhatConfigureComponent，然后该消息处理中又调用了ACodec::LoadedState::onConfigureComponent(…)。然后在其中又会先调用ACodec::configureCodec(…)，在configureCodec中会对IOMX进行一系列的设置以及配置操作，通过Binder通信就对OMXNodeInstance进行相应的设置和配置操作，最终就对OMX组件进行了相应的设置和配置。然后向MediaCodec发送kWhatComponentConfigured消息，在消息处理中将MediaCodec状态设为CONFIGURED；

``` cpp
[->\frameworks\av\media\libstagefright\MediaCodec.cpp]
status_t MediaCodec::configure(
        const sp<AMessage> &format,
        const sp<Surface> &surface,
        const sp<ICrypto> &crypto,
        uint32_t flags) {
    sp<AMessage> msg = new AMessage(kWhatConfigure, this);
        case kWhatConfigure:
        {
            sp<AReplyToken> replyID;
            ......
            sp<RefBase> obj;
            sp<AMessage> format;
            ......
            if (obj != NULL) {
                format->setObject("native-window", obj);
                status_t err = handleSetSurface(static_cast<Surface *>(obj.get()));
            } else {
                handleSetSurface(NULL);
            }

            mReplyID = replyID;
            setState(CONFIGURING);
            ......
            extractCSD(format);

            mCodec->initiateConfigureComponent(format);
            break;
        }

```

``` cpp
[->\frameworks\av\media\libstagefright\ACodec.cpp]
void ACodec::initiateConfigureComponent(const sp<AMessage> &msg) {
    msg->setWhat(kWhatConfigureComponent);
    msg->setTarget(this);
    msg->post();
}

        case ACodec::kWhatConfigureComponent:
        {
            onConfigureComponent(msg);
            handled = true;
            break;
        }
bool ACodec::LoadedState::onConfigureComponent(
        const sp<AMessage> &msg) {
    status_t err = OK;
    AString mime;
    if (!msg->findString("mime", &mime)) {
        err = BAD_VALUE;
    } else {
        err = mCodec->configureCodec(mime.c_str(), msg);
    }
   ......
    {
        sp<AMessage> notify = mCodec->mNotify->dup();
        notify->setInt32("what", CodecBase::kWhatComponentConfigured);
        notify->setMessage("input-format", mCodec->mInputFormat);
        notify->setMessage("output-format", mCodec->mOutputFormat);
        notify->post();
    }
    return true;
}
```

##### 2.3.6、MediaCodec->start()

产生kWhatStart消息，消息处理中先将MediaCodec状态设为STARTING，然后调用ACodec::initiateStart()产生kWhatStart消息，在其消息处理中又调用ACodec::LoadedState::onStart()，然后在其中首先向IOMX发送状态转换命令，经过OMXNodeInstance最终对将OMX组件状态转换成Idle（转换完成时OMX会发送OMX_EventCmdComplete事件），接着对ACodec进行changeState至LoadedToIdleState。而在changeState过程中会调用ACodec::LoadedToIdleState::stateEntered() =>  ACodec::LoadedToIdleState::allocateBuffers() => ACodec::allocateBuffersOnPort(…)，其中会为OMX组件端口分配缓冲，并向MediaCodec发送消息kWhatBuffersAllocated，消息处理中将MediaCodec状态设为STARTED而若allocateBuffers失败则由IOMX经OMXNodeInstance将OMX组件转换回Loaded状态，同时把ACodec状态转换回LoadedState

``` cpp
[->\frameworks\av\media\libstagefright\MediaCodec.cpp]
status_t MediaCodec::start() {
    sp<AMessage> msg = new AMessage(kWhatStart, this);
......
}
        case kWhatStart:
        {
            sp<AReplyToken> replyID;
            ......
            setState(STARTING);

            mCodec->initiateStart();
            break;
        }
```

``` cpp
[->\frameworks\av\media\libstagefright\ACodec.cpp]
void ACodec::initiateStart() {
    (new AMessage(kWhatStart, this))->post();
}

        case ACodec::kWhatStart:
        {
            onStart();
            handled = true;
            break;
        }

void ACodec::LoadedState::onStart() {
    status_t err = mCodec->mOMX->sendCommand(mCodec->mNode, OMX_CommandStateSet, OMX_StateIdle);
    if (err != OK) {
        mCodec->signalError(OMX_ErrorUndefined, makeNoSideEffectStatus(err));
    } else {
        mCodec->changeState(mCodec->mLoadedToIdleState);
    }
}
```

一旦缓冲区成功分配到输入和输出端口，OMX组件（编解码）会为Loaded-to-Idle状态生成OMX_EventCmdComplete事件转换并使用EventHandlerCallback将其发送给客户端。

#### （三）、音视频解码数据处理
##### 3.1、音视频解码数据处理-emptyBuffer
还是老样子，先看看时序图，然后一步步分析

![Alt text](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/video.system/02-03-acodec-emptyBuffer.png)

1、	MediaCodec::start()之后ACodec是在LoadedToIdleState状态，此时若ACodec::LoadedToIdleState::onOMXEvent(…)接收到组件转换至Idle状态后的OMX_EventCmdComplete事件，会向IOMX发送状态转换命令，经过OMXNodeInstance最终对将OMX组件状态转换成Executing状态（这里OMX会发送OMX_EventCmdComplete事件），然后ACodec进行changeState至IdleToExecutingState。
2、	此时ACodec::IdleToExecutingState::onOMXEvent(…)检测到上面的OMX_EventCmdComplete事件后，会首先调用函数ACodec::ExecutingState::resume()，然后对ACodec进行changeState至ExecutingState。
##### 3.1.1、ACodec::ExecutingState::resume()

``` cpp
[->\frameworks\av\media\libstagefright\ACodec.cpp]
void ACodec::ExecutingState::resume() {
    submitOutputBuffers();
    ......
    for (size_t i = 0; i < mCodec->mBuffers[kPortIndexInput].size(); i++) {
        BufferInfo *info = &mCodec->mBuffers[kPortIndexInput].editItemAt(i);
        if (info->mStatus == BufferInfo::OWNED_BY_US) {
            postFillThisBuffer(info);
        }
    }

    mActive = true;
}
void ACodec::BaseState::postFillThisBuffer(BufferInfo *info) {
    if (mCodec->mPortEOS[kPortIndexInput]) {
        return;
    }

    CHECK_EQ((int)info->mStatus, (int)BufferInfo::OWNED_BY_US);

    sp<AMessage> notify = mCodec->mNotify->dup();
    notify->setInt32("what", CodecBase::kWhatFillThisBuffer);
    notify->setInt32("buffer-id", info->mBufferID);

    info->mData->meta()->clear();
    notify->setBuffer("buffer", info->mData);

    sp<AMessage> reply = new AMessage(kWhatInputBufferFilled, mCodec);
    reply->setInt32("buffer-id", info->mBufferID);

    notify->setMessage("reply", reply);

    notify->post();

    info->mStatus = BufferInfo::OWNED_BY_UPSTREAM;
}

```
在函数ACodec::ExecutingState::resume()中会调用ACodec::BaseState::postFillThisBuffer(…)，然后其中会先向MediaCodec发送kWhatFillThisBuffer消息，消息处理中在满足相应的条件下就会去调用函数MediaCodec::onInputBufferAvailable()来通知NuPlayer::Decoder有可用的inputbuffer；然后再生成kWhatInputBufferFilled消息，消息处理中调用ACodec::BaseState::onInputBufferFilled(…)。
【产生两个消息，一个向上(MediaCodec)处理，一个向下(OMX)处理】
##### 3.1.1.1、kWhatFillThisBuffer消息处理
``` cpp
[->\frameworks\av\media\libstagefright\MediaCodec.cpp]
                case CodecBase::kWhatFillThisBuffer:
                {
                    /* size_t index = */updateBuffers(kPortIndexInput, msg);
                    ......
                    if (mFlags & kFlagIsAsync) {
                        if (!mHaveInputSurface) {
                            if (mState == FLUSHED) {
                                mHavePendingInputBuffers = true;
                            } else {
                                onInputBufferAvailable();
                            }
                        }
                    } else if (mFlags & kFlagDequeueInputPending) {
                        ++mDequeueInputTimeoutGeneration;
                        mFlags &= ~kFlagDequeueInputPending;
                        mDequeueInputReplyID = 0;
                    } else {
                        postActivityNotificationIfPossible();
                    }
                    break;
                }
void MediaCodec::onInputBufferAvailable() {
    int32_t index;
    while ((index = dequeuePortBuffer(kPortIndexInput)) >= 0) {
        sp<AMessage> msg = mCallback->dup();
        msg->setInt32("callbackID", CB_INPUT_AVAILABLE);
        msg->setInt32("index", index);
        msg->post();
    }
}
```
P.S. 1：MediaCodec::onInputBufferAvailable()的调用：
其中会先调用函数MediaCodec::dequeuePortBuffer(…)获取buffer的索引，然后将一个新消息发送给NuPlayer::Decoder，并设置消息的callbackID为CB_INPUT_AVAILABLE，同时设置index，接着NuPlayer::Decoder接收到该CB_INPUT_AVAILABLE消息，在消息处理中调用NuPlayer::Decoder::handleAnInputBuffer(…)，其会：

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayerDecoder.cpp]
case MediaCodec::CB_INPUT_AVAILABLE:
{
    int32_t index;
    CHECK(msg->findInt32("index", &index));

    handleAnInputBuffer(index);
    break;
}
bool NuPlayer::Decoder::handleAnInputBuffer(size_t index) {
    sp<ABuffer> buffer;
    mCodec->getInputBuffer(index, &buffer);

    if (index >= mInputBuffers.size()) {
        for (size_t i = mInputBuffers.size(); i <= index; ++i) {
            mInputBuffers.add();
            mMediaBuffers.add();
            mInputBufferIsDequeued.add();
            mMediaBuffers.editItemAt(i) = NULL;
            mInputBufferIsDequeued.editItemAt(i) = false;
        }
    }
    mInputBuffers.editItemAt(index) = buffer;

    if (mMediaBuffers[index] != NULL) {
        mMediaBuffers[index]->release();
        mMediaBuffers.editItemAt(index) = NULL;
    }
    mInputBufferIsDequeued.editItemAt(index) = true;

    if (!mCSDsToSubmit.isEmpty()) {
        sp<AMessage> msg = new AMessage();
        msg->setSize("buffer-ix", index);

        sp<ABuffer> buffer = mCSDsToSubmit.itemAt(0);
        msg->setBuffer("buffer", buffer);
        mCSDsToSubmit.removeAt(0);
        return true;
    }

    while (!mPendingInputMessages.empty()) {
        sp<AMessage> msg = *mPendingInputMessages.begin();
        if (!onInputBufferFetched(msg)) {
            break;
        }
        mPendingInputMessages.erase(mPendingInputMessages.begin());
    }

    if (!mInputBufferIsDequeued.editItemAt(index)) {
        return true;
    }

    mDequeuedInputBuffers.push_back(index);

    onRequestInputBuffers();
    return true;
}
```

○1、先通过MediaCodec::getInputBuffer(…) -> MediaCodec::getBufferAndFormat(…)获取该buffer


○2、然后调用NuPlayer::Decoder::onInputBufferFetched(…)执行内存拷贝将buffer拷贝到编解码器，然后又调用了MediaCodec::queueInputBuffer(…)将buffer提交给解码器，其会产生消息kWhatQueueInputBuffer，消息处理中调用MediaCodec::onQueueInputBuffer(…)

○3、之后调用函数NuPlayer::DecoderBase::onRequestInputBuffers()，处理是否需要更多的数据。其中会调用NuPlayer::Decoder::doRequestBuffers，若返回true则需要更多的数据，则会产生新消息kWhatRequestInputBuffers，消息处理中又将调用onRequestInputBuffers。（实际获取更多缓冲的操作在下面ACodec部分完成）

##### 3.1.1.2、kWhatInputBufferFilled消息处理

``` cpp
[->\frameworks\av\media\libstagefright\ACodec.cpp]
        case kWhatInputBufferFilled:
        {
            onInputBufferFilled(msg);
            break;
        }
        
void ACodec::BaseState::onInputBufferFilled(const sp<AMessage> &msg) {
    IOMX::buffer_id bufferID;
    CHECK(msg->findInt32("buffer-id", (int32_t*)&bufferID));
    sp<ABuffer> buffer;
    int32_t err = OK;
    bool eos = false;
    PortMode mode = getPortMode(kPortIndexInput);
    int32_t tmp;
    BufferInfo *info = mCodec->findBufferByID(kPortIndexInput, bufferID);
    BufferInfo::Status status = BufferInfo::getSafeStatus(info);
    info->mStatus = BufferInfo::OWNED_BY_US;

    switch (mode) {
        case KEEP_BUFFERS:
        {
            if (eos) {
                if (!mCodec->mPortEOS[kPortIndexInput]) {
                    mCodec->mPortEOS[kPortIndexInput] = true;
                    mCodec->mInputEOSResult = err;
                }
            }
            break;
        }

        case RESUBMIT_BUFFERS:
        {
            if (buffer != NULL && !mCodec->mPortEOS[kPortIndexInput]) {
                int64_t timeUs;
                CHECK(buffer->meta()->findInt64("timeUs", &timeUs));

                OMX_U32 flags = OMX_BUFFERFLAG_ENDOFFRAME;

                MetadataBufferType metaType = mCodec->mInputMetadataType;
                int32_t isCSD = 0;
                if (buffer->meta()->findInt32("csd", &isCSD) && isCSD != 0) {
                    if (mCodec->mIsLegacyVP9Decoder) {
                        postFillThisBuffer(info);
                        break;
                    }
                    flags |= OMX_BUFFERFLAG_CODECCONFIG;
                    metaType = kMetadataBufferTypeInvalid;
                }

               ......
                if (buffer != info->mCodecData) {
                    sp<DataConverter> converter = mCodec->mConverter[kPortIndexInput];
                    status_t err = converter->convert(buffer, info->mCodecData);
                }
                ......
                info->checkReadFence("onInputBufferFilled");

                status_t err2 = OK;
                switch (metaType) {
                case kMetadataBufferTypeInvalid:
                    break;
#ifndef OMX_ANDROID_COMPILE_AS_32BIT_ON_64BIT_PLATFORMS
                case kMetadataBufferTypeNativeHandleSource:
                    if (info->mCodecData->size() >= sizeof(VideoNativeHandleMetadata)) {
                        VideoNativeHandleMetadata *vnhmd =
                            (VideoNativeHandleMetadata*)info->mCodecData->base();
                        err2 = mCodec->mOMX->updateNativeHandleInMeta(
                                mCodec->mNode, kPortIndexInput,
                                NativeHandle::create(vnhmd->pHandle, false /* ownsHandle */),
                                bufferID);
                    }
                    break;
                case kMetadataBufferTypeANWBuffer:
                    if (info->mCodecData->size() >= sizeof(VideoNativeMetadata)) {
                        VideoNativeMetadata *vnmd = (VideoNativeMetadata*)info->mCodecData->base();
                        err2 = mCodec->mOMX->updateGraphicBufferInMeta(
                                mCodec->mNode, kPortIndexInput,
                                new GraphicBuffer(vnmd->pBuffer, false /* keepOwnership */),
                                bufferID);
                    }
                    break;
#endif
                default:
                    err2 = ERROR_UNSUPPORTED;
                    break;
                }

                if (err2 == OK) {
                    err2 = mCodec->mOMX->emptyBuffer(
                        mCodec->mNode,
                        bufferID,
                        0,
                        info->mCodecData->size(),
                        flags,
                        timeUs,
                        info->mFenceFd);
                }
                info->mFenceFd = -1;
                ......
                info->mStatus = BufferInfo::OWNED_BY_COMPONENT;

                if (!eos && err == OK) {
                    getMoreInputDataIfPossible();
                } else {
                    ALOGV("[%s] Signalled EOS (%d) on the input port",
                         mCodec->mComponentName.c_str(), err);

                    mCodec->mPortEOS[kPortIndexInput] = true;
                    mCodec->mInputEOSResult = err;
                }
            } else if (!mCodec->mPortEOS[kPortIndexInput]) {
                ......

                info->checkReadFence("onInputBufferFilled");
                status_t err2 = mCodec->mOMX->emptyBuffer(
                        mCodec->mNode,
                        bufferID,
                        0,
                        0,
                        OMX_BUFFERFLAG_EOS,
                        0,
                        info->mFenceFd);
                info->mFenceFd = -1;
                ......
                info->mStatus = BufferInfo::OWNED_BY_COMPONENT;

                mCodec->mPortEOS[kPortIndexInput] = true;
                mCodec->mInputEOSResult = err;
            }
            break;
        }
        ......
    }
}
```
P.S. 2：ACodec::BaseState::onInputBufferFilled(…)的调用：
因为当前ACodec在ExecutingState，所以PortMode为RESUBMIT_BUFFERS，故会调用IOMX的emptyBuffer(…)方法，经过进程间通信调用到OMX::emptyBuffer(…)，并最终调用OMXNodeInstance::emptyBuffer(…)，其中又会调用到函数OMXNodeInstance::emptyBuffer_l(…)，其则会调用OMX_EmptyThisBuffer宏对OMX组件进行相关的操作（根据需要选择相应的软解组件或者硬解组件）。对于软解组件SoftOMXComponent

○1、其的构造函数的初始化列表中有mComponent->EmptyThisBuffer = EmptyThisBufferWrapper;故实际会调用其EmptyThisBufferWrapper(…)函数，而其中调用SoftOMXComponent的虚函数emptyThisBuffer。

○2、所以调用子类的emptyThisBuffer即SimpleSoftOMXComponent::emptyThisBuffer(…)产生kWhatEmptyThisBuffer消息，消息处理中实际的解码器就要调用onQueueFilled(…)函数【实际组件继承自SimpleSoftOMXComponent】

○3、接着会调用SoftOMXComponent::notifyEmptyBufferDone(…)使用OMX的回调机制，闭环发送消息到OMX客户端ACodec。

○4、调用到OMXNodeInstance::OnEmptyBufferDone(…)，其又会调用OMX::OnEmptyBufferDone(…)，然后在其中会发送omx_message::EMPTY_BUFFER_DONE消息，ACodec中收到该消息【CodecObserver中先收到，但只设置消息】调用ACodec::BaseState::onOMXEmptyBufferDone(…)

○5、在onOMXEmptyBufferDone中获取PortMode，为RESUBMIT_BUFFERS则ACodec::BaseState::postFillThisBuffer(…)被调用，从而又从3中的postFillThisBuffer开始循环执行相关操作以处理更多的输入缓冲。

##### 3.2、音视频解码数据处理-fillBuffer
还是老样子，先看看时序图，然后一步步分析

![Alt text](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/video.system/02-04-acodec-fillBuffer.png)


##### 3.2.1、ACodec::ExecutingState::resume()
``` cpp
[->\frameworks\av\media\libstagefright\ACodec.cpp]
void ACodec::ExecutingState::resume() {
    submitOutputBuffers();
    for (size_t i = 0; i < mCodec->mBuffers[kPortIndexInput].size(); i++) {
        BufferInfo *info = &mCodec->mBuffers[kPortIndexInput].editItemAt(i);
        if (info->mStatus == BufferInfo::OWNED_BY_US) {
            postFillThisBuffer(info);
        }
    }
    mActive = true;
}
void ACodec::ExecutingState::submitOutputBuffers() {
    submitRegularOutputBuffers();
    if (mCodec->storingMetadataInDecodedBuffers()) {
        submitOutputMetaBuffers();
    }
}

void ACodec::ExecutingState::submitRegularOutputBuffers() {
    bool failed = false;
    for (size_t i = 0; i < mCodec->mBuffers[kPortIndexOutput].size(); ++i) {
        BufferInfo *info = &mCodec->mBuffers[kPortIndexOutput].editItemAt(i);

        if (mCodec->mNativeWindow != NULL) {
            ......
        } else {
            ......
        }
        ......
        info->checkWriteFence("submitRegularOutputBuffers");
        status_t err = mCodec->mOMX->fillBuffer(mCodec->mNode, info->mBufferID, info->mFenceFd);
        info->mFenceFd = -1;
        ......
        info->mStatus = BufferInfo::OWNED_BY_COMPONENT;
    }
    ......
}

```
1、ACodec::ExecutingState::resume()函数，在resume()中调用ACodec::BaseState::postFillThisBuffer(…)前会先调用函数ACodec::ExecutingState::submitOutputBuffers()，即在获取输入数据前会先把输出端的数据提交出去。

2、在submitOutputBuffers()中调用ACodec::ExecutingState::submitRegularOutputBuffers()，其中又会调用到IOMX的fillBuffer (…)方法，经过进程间通信调用到OMX:: fillBuffer (…)，并最终调用OMXNodeInstance:: fillBuffer (…)，其中又会调用到OMX_FillThisBuffer宏对OMX组件进行相关的操作（同样根据需要选择相应的软解组件或者硬解组件）。对于软解组件SoftOMXComponent：（下面的操作与emptyBuffer时类似）

○1、在其构造函数的初始化列表中有mComponent->FillThisBuffer = FillThisBufferWrapper;所以实际会调用到其FillThisBufferWrapper (…)函数
○2、然后调用SimpleSoftOMXComponent::fillThisBuffer(…)产生kWhatFillThisBuffer消息，消息处理中实际的组件就要调用onQueueFilled(…)函数【实际组件继承自SimpleSoftOMXComponent】

○3、接着会调用SoftOMXComponent::notifyFillBufferDone(…)使用OMX的回调机制，闭环发送消息到OMX客户端ACodec。
○4之后调用到OMXNodeInstance:: OnFillBufferDone (…)函数，其又会调用OMX:: OnFillBufferDone (…)，然后在其中会发送omx_message:: FILL_BUFFER_DONE消息，ACodec中收到该消息【CodecObserver中先收到，但只设置消息】调用ACodec::BaseState:: onOMXFillBufferDone (…)
``` cpp
[->\frameworks\av\media\libstagefright\ACodec.cpp]
bool ACodec::BaseState::onOMXFillBufferDone(
        IOMX::buffer_id bufferID,
        size_t rangeOffset, size_t rangeLength,
        OMX_U32 flags,
        int64_t timeUs,
        int fenceFd) {
    ALOGV("[%s] onOMXFillBufferDone %u time %" PRId64 " us, flags = 0x%08x",
         mCodec->mComponentName.c_str(), bufferID, timeUs, flags);

    ssize_t index;
    status_t err= OK;

#if TRACK_BUFFER_TIMING
    index = mCodec->mBufferStats.indexOfKey(timeUs);
    if (index >= 0) {
        ACodec::BufferStats *stats = &mCodec->mBufferStats.editValueAt(index);
        stats->mFillBufferDoneTimeUs = ALooper::GetNowUs();

        ALOGI("frame PTS %lld: %lld",
                timeUs,
                stats->mFillBufferDoneTimeUs - stats->mEmptyBufferTimeUs);

        mCodec->mBufferStats.removeItemsAt(index);
        stats = NULL;
    }
#endif

    BufferInfo *info =
        mCodec->findBufferByID(kPortIndexOutput, bufferID, &index);
    BufferInfo::Status status = BufferInfo::getSafeStatus(info);
    if (status != BufferInfo::OWNED_BY_COMPONENT) {
        ALOGE("Wrong ownership in FBD: %s(%d) buffer #%u", _asString(status), status, bufferID);
        mCodec->dumpBuffers(kPortIndexOutput);
        mCodec->signalError(OMX_ErrorUndefined, FAILED_TRANSACTION);
        if (fenceFd >= 0) {
            ::close(fenceFd);
        }
        return true;
    }

    info->mDequeuedAt = ++mCodec->mDequeueCounter;
    info->mStatus = BufferInfo::OWNED_BY_US;

    if (info->mRenderInfo != NULL) {
        // The fence for an emptied buffer must have signaled, but there still could be queued
        // or out-of-order dequeued buffers in the render queue prior to this buffer. Drop these,
        // as we will soon requeue this buffer to the surface. While in theory we could still keep
        // track of buffers that are requeued to the surface, it is better to add support to the
        // buffer-queue to notify us of released buffers and their fences (in the future).
        mCodec->notifyOfRenderedFrames(true /* dropIncomplete */);
    }

    // byte buffers cannot take fences, so wait for any fence now
    if (mCodec->mNativeWindow == NULL) {
        (void)mCodec->waitForFence(fenceFd, "onOMXFillBufferDone");
        fenceFd = -1;
    }
    info->setReadFence(fenceFd, "onOMXFillBufferDone");

    PortMode mode = getPortMode(kPortIndexOutput);

    switch (mode) {
        case KEEP_BUFFERS:
            break;

        case RESUBMIT_BUFFERS:
        {
            if (rangeLength == 0 && (!(flags & OMX_BUFFERFLAG_EOS)
                    || mCodec->mPortEOS[kPortIndexOutput])) {
                ......
                err = mCodec->mOMX->fillBuffer(mCodec->mNode, info->mBufferID, info->mFenceFd);
                info->mFenceFd = -1;
                ......
                info->mStatus = BufferInfo::OWNED_BY_COMPONENT;
                break;
            }

            sp<AMessage> reply =
                new AMessage(kWhatOutputBufferDrained, mCodec);

            if (mCodec->mOutputFormat != mCodec->mLastOutputFormat && rangeLength > 0) {
                // pretend that output format has changed on the first frame (we used to do this)
                if (mCodec->mBaseOutputFormat == mCodec->mOutputFormat) {
                    mCodec->onOutputFormatChanged(mCodec->mOutputFormat);
                }
                mCodec->addKeyFormatChangesToRenderBufferNotification(reply);
                mCodec->sendFormatChange();
            } else if (rangeLength > 0 && mCodec->mNativeWindow != NULL) {
                // If potentially rendering onto a surface, always save key format data (crop &
                // data space) so that we can set it if and once the buffer is rendered.
                mCodec->addKeyFormatChangesToRenderBufferNotification(reply);
            }

            if (mCodec->usingMetadataOnEncoderOutput()) {
                native_handle_t *handle = NULL;
                VideoNativeHandleMetadata &nativeMeta =
                    *(VideoNativeHandleMetadata *)info->mData->data();
                if (info->mData->size() >= sizeof(nativeMeta)
                        && nativeMeta.eType == kMetadataBufferTypeNativeHandleSource) {
#ifdef OMX_ANDROID_COMPILE_AS_32BIT_ON_64BIT_PLATFORMS
                    // handle is only valid on 32-bit/mediaserver process
                    handle = NULL;
#else
                    handle = (native_handle_t *)nativeMeta.pHandle;
#endif
                }
                info->mData->meta()->setPointer("handle", handle);
                info->mData->meta()->setInt32("rangeOffset", rangeOffset);
                info->mData->meta()->setInt32("rangeLength", rangeLength);
            } else if (info->mData == info->mCodecData) {
                info->mData->setRange(rangeOffset, rangeLength);
            } else {
                info->mCodecData->setRange(rangeOffset, rangeLength);
                // in this case we know that mConverter is not null
                status_t err = mCodec->mConverter[kPortIndexOutput]->convert(
                        info->mCodecData, info->mData);
                if (err != OK) {
                    mCodec->signalError(OMX_ErrorUndefined, makeNoSideEffectStatus(err));
                    return true;
                }
            }

            if (mCodec->mSkipCutBuffer != NULL) {
                mCodec->mSkipCutBuffer->submit(info->mData);
            }
            info->mData->meta()->setInt64("timeUs", timeUs);

            sp<AMessage> notify = mCodec->mNotify->dup();
            notify->setInt32("what", CodecBase::kWhatDrainThisBuffer);
            notify->setInt32("buffer-id", info->mBufferID);
            notify->setBuffer("buffer", info->mData);
            notify->setInt32("flags", flags);

            reply->setInt32("buffer-id", info->mBufferID);

            notify->setMessage("reply", reply);

            notify->post();

            info->mStatus = BufferInfo::OWNED_BY_DOWNSTREAM;

            if (flags & OMX_BUFFERFLAG_EOS) {
                ALOGV("[%s] saw output EOS", mCodec->mComponentName.c_str());

                sp<AMessage> notify = mCodec->mNotify->dup();
                notify->setInt32("what", CodecBase::kWhatEOS);
                notify->setInt32("err", mCodec->mInputEOSResult);
                notify->post();

                mCodec->mPortEOS[kPortIndexOutput] = true;
            }
            break;
        }

        case FREE_BUFFERS:
            err = mCodec->freeBuffer(kPortIndexOutput, index);
            if (err != OK) {
                mCodec->signalError(OMX_ErrorUndefined, makeNoSideEffectStatus(err));
                return true;
            }
            break;

        default:
            ALOGE("Invalid port mode: %d", mode);
            return false;
    }

    return true;
}

```

○5、在onOMXFillBufferDone中获取PortMode，为RESUBMIT_BUFFERS则首先如果需要继续调用到IOMX的fillBuffer (…)填充输出缓冲重复做相关操作，接着ACodec又会生成一个kWhatOutputBufferDrained消息存在reply中，作为kWhatDrainThisBuffer消息的返回消息【notify->setMessage("reply", reply);】，然后向MediaCodec发送消息kWhatDrainThisBuffer，消息处理中调用函数MediaCodec::onOutputBufferAvailable()通知NuPlayer::Decoder有可用的output buffer，其中会设置消息的callbackID为CB_OUTPUT_AVAILABLE，同时设置index，接着NuPlayer::Decoder接收到该CB_OUTPUT_AVAILABLE消息，在消息处理中调用NuPlayer::Decoder::handleAnOutputBuffer(…)，在其中会进行如下处理：

``` cpp
[->\frameworks\av\media\libstagefright\MediaCodec.cpp]
void MediaCodec::onOutputBufferAvailable() {
    int32_t index;
    while ((index = dequeuePortBuffer(kPortIndexOutput)) >= 0) {
        const sp<ABuffer> &buffer =
            mPortBuffers[kPortIndexOutput].itemAt(index).mData;
        sp<AMessage> msg = mCallback->dup();
        msg->setInt32("callbackID", CB_OUTPUT_AVAILABLE);
        msg->setInt32("index", index);
        msg->setSize("offset", buffer->offset());
        msg->setSize("size", buffer->size());

        int64_t timeUs;
        CHECK(buffer->meta()->findInt64("timeUs", &timeUs));

        msg->setInt64("timeUs", timeUs);

        int32_t omxFlags;
        CHECK(buffer->meta()->findInt32("omxFlags", &omxFlags));

        uint32_t flags = 0;
        if (omxFlags & OMX_BUFFERFLAG_SYNCFRAME) {
            flags |= BUFFER_FLAG_SYNCFRAME;
        }
        if (omxFlags & OMX_BUFFERFLAG_CODECCONFIG) {
            flags |= BUFFER_FLAG_CODECCONFIG;
        }
        if (omxFlags & OMX_BUFFERFLAG_EOS) {
            flags |= BUFFER_FLAG_EOS;
        }

        msg->setInt32("flags", flags);

        msg->post();
    }
}
```
#### （四）、多媒体文件 - 音视频渲染（Renderer）
``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayerDecoder.cpp]
bool NuPlayer::Decoder::handleAnOutputBuffer(
        size_t index,
        size_t offset,
        size_t size,
        int64_t timeUs,
        int32_t flags) {
    sp<ABuffer> buffer;
    mCodec->getOutputBuffer(index, &buffer);

    if (index >= mOutputBuffers.size()) {
        for (size_t i = mOutputBuffers.size(); i <= index; ++i) {
            mOutputBuffers.add();
        }
    }

    mOutputBuffers.editItemAt(index) = buffer;

    buffer->setRange(offset, size);
    buffer->meta()->clear();
    buffer->meta()->setInt64("timeUs", timeUs);

    bool eos = flags & MediaCodec::BUFFER_FLAG_EOS;
    // we do not expect CODECCONFIG or SYNCFRAME for decoder

    sp<AMessage> reply = new AMessage(kWhatRenderBuffer, this);
    reply->setSize("buffer-ix", index);
    reply->setInt32("generation", mBufferGeneration);

    if (eos) {
        buffer->meta()->setInt32("eos", true);
        reply->setInt32("eos", true);
    } else if (mSkipRenderingUntilMediaTimeUs >= 0) {
        if (timeUs < mSkipRenderingUntilMediaTimeUs) {
            reply->post();
            return true;
        }

        mSkipRenderingUntilMediaTimeUs = -1;
    }

    mNumFramesTotal += !mIsAudio;

    // wait until 1st frame comes out to signal resume complete
    notifyResumeCompleteIfNecessary();

    if (mRenderer != NULL) {
        // send the buffer to renderer.
        mRenderer->queueBuffer(mIsAudio, buffer, reply);
        if (eos && !isDiscontinuityPending()) {
            mRenderer->queueEOS(mIsAudio, ERROR_END_OF_STREAM);
        }
    }

    return true;
}
```
a.	在kWhatRenderBuffer消息处理中会调用NuPlayer::Decoder::onRenderBuffer(…)，在其中根据情况调用函数MediaCodec::renderOutputBufferAndRelease(..)渲染并释放，或者调用MediaCodec::releaseOutputBuffer(…)不渲染直接释放，两中情况都会产生kWhatReleaseOutputBuffer消息，该消息处理中调用函数MediaCodec::onReleaseOutputBuffer(…)，其中判断若SoftRenderer非空则进行软件渲染，不然就会通过○5中的reply让ACodec去硬件渲染，在kWhatOutputBufferDrained消息处理就会中调用到函数ACodec::BaseState::onOutputBufferDrained(…)进行真正的硬件渲染。

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayerDecoder.cpp]
void NuPlayer::Decoder::onRenderBuffer(const sp<AMessage> &msg) {
    status_t err;
    int32_t render;
    size_t bufferIx;
    int32_t eos;
    CHECK(msg->findSize("buffer-ix", &bufferIx));

    if (!mIsAudio) {
        int64_t timeUs;
        sp<ABuffer> buffer = mOutputBuffers[bufferIx];
        buffer->meta()->findInt64("timeUs", &timeUs);

        if (mCCDecoder != NULL && mCCDecoder->isSelected()) {
            mCCDecoder->display(timeUs);
        }
    }

    if (msg->findInt32("render", &render) && render) {
        int64_t timestampNs;
        CHECK(msg->findInt64("timestampNs", &timestampNs));
        err = mCodec->renderOutputBufferAndRelease(bufferIx, timestampNs);
    } else {
        mNumOutputFramesDropped += !mIsAudio;
        err = mCodec->releaseOutputBuffer(bufferIx);
    }
    ......
}
```

b.	MediaCodec:: getOutputBuffer (…) -> MediaCodec::getBufferAndFormat(…)获取该buffer的信息
c.	若Renderer非空则会调用NuPlayer::Renderer::queueBuffer(…)进行Renderer的相关处理同时消耗产生的kWhatRenderBuffer消息。queueBuffer()会产生kWhatQueueBuffer消息，消息处理中会调用函数NuPlayer::Renderer::onQueueBuffer(…) –> NuPlayer::Renderer::postDrainVideoQueue() 【另外有audio的相关处理】，其中产生kWhatDrainVideoQueue消息，消息处理中调用先NuPlayer::Renderer::onDrainVideoQueue()在VideoQueue中取相关数据，再调用NuPlayer::Renderer::postDrainVideoQueue()循环取video数据，接着还会发送kWhatRenderBuffer消息。


``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayerRenderer.cpp]
void NuPlayer::Renderer::onQueueBuffer(const sp<AMessage> &msg) {
    int32_t audio;
    if (audio) {
        mHasAudio = true;
    } else {
        mHasVideo = true;
    }

    if (mHasVideo) {
        if (mVideoScheduler == NULL) {
            mVideoScheduler = new VideoFrameScheduler();
            mVideoScheduler->init();
        }
    }

    sp<ABuffer> buffer;
    CHECK(msg->findBuffer("buffer", &buffer));

    sp<AMessage> notifyConsumed;
    CHECK(msg->findMessage("notifyConsumed", &notifyConsumed));

    QueueEntry entry;
    entry.mBuffer = buffer;
    entry.mNotifyConsumed = notifyConsumed;
    entry.mOffset = 0;
    entry.mFinalResult = OK;
    entry.mBufferOrdinal = ++mTotalBuffersQueued;

    if (audio) {
        Mutex::Autolock autoLock(mLock);
        mAudioQueue.push_back(entry);
        postDrainAudioQueue_l();
    } else {
        mVideoQueue.push_back(entry);
        postDrainVideoQueue();
    }

    Mutex::Autolock autoLock(mLock);
    if (!mSyncQueues || mAudioQueue.empty() || mVideoQueue.empty()) {
        return;
    }

    sp<ABuffer> firstAudioBuffer = (*mAudioQueue.begin()).mBuffer;
    sp<ABuffer> firstVideoBuffer = (*mVideoQueue.begin()).mBuffer;

    if (firstAudioBuffer == NULL || firstVideoBuffer == NULL) {
        syncQueuesDone_l();
        return;
    }

    int64_t firstAudioTimeUs;
    int64_t firstVideoTimeUs;

    int64_t diff = firstVideoTimeUs - firstAudioTimeUs;

    if (diff > 100000ll) {
        (*mAudioQueue.begin()).mNotifyConsumed->post();
        mAudioQueue.erase(mAudioQueue.begin());
        return;
    }

    syncQueuesDone_l();
}
```


![Alt text](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/video.system/02-05-source-demux-decoder-output-render.jpg)


#### （五）、视频解码输出到SurfaceFlinger

``` cpp
[->\frameworks\av\media\libstagefright\ACodec.cpp]
void ACodec::BaseState::onOutputBufferDrained(const sp<AMessage> &msg) {
    IOMX::buffer_id bufferID;
    CHECK(msg->findInt32("buffer-id", (int32_t*)&bufferID));
    ssize_t index;
    BufferInfo *info = mCodec->findBufferByID(kPortIndexOutput, bufferID, &index);
    BufferInfo::Status status = BufferInfo::getSafeStatus(info);
    if (status != BufferInfo::OWNED_BY_DOWNSTREAM) {
        ALOGE("Wrong ownership in OBD: %s(%d) buffer #%u", _asString(status), status, bufferID);
        mCodec->dumpBuffers(kPortIndexOutput);
        mCodec->signalError(OMX_ErrorUndefined, FAILED_TRANSACTION);
        return;
    }

    android_native_rect_t crop;
    if (msg->findRect("crop", &crop.left, &crop.top, &crop.right, &crop.bottom)
            && memcmp(&crop, &mCodec->mLastNativeWindowCrop, sizeof(crop)) != 0) {
        mCodec->mLastNativeWindowCrop = crop;
        status_t err = native_window_set_crop(mCodec->mNativeWindow.get(), &crop);
        ALOGW_IF(err != NO_ERROR, "failed to set crop: %d", err);
    }

    int32_t dataSpace;
    if (msg->findInt32("dataspace", &dataSpace)
            && dataSpace != mCodec->mLastNativeWindowDataSpace) {
        status_t err = native_window_set_buffers_data_space(
                mCodec->mNativeWindow.get(), (android_dataspace)dataSpace);
        mCodec->mLastNativeWindowDataSpace = dataSpace;
        ALOGW_IF(err != NO_ERROR, "failed to set dataspace: %d", err);
    }

    int32_t render;
    if (mCodec->mNativeWindow != NULL
            && msg->findInt32("render", &render) && render != 0
            && info->mData != NULL && info->mData->size() != 0) {
        ATRACE_NAME("render");
        // The client wants this buffer to be rendered.

        // save buffers sent to the surface so we can get render time when they return
        int64_t mediaTimeUs = -1;
        info->mData->meta()->findInt64("timeUs", &mediaTimeUs);
        if (mediaTimeUs >= 0) {
            mCodec->mRenderTracker.onFrameQueued(
                    mediaTimeUs, info->mGraphicBuffer, new Fence(::dup(info->mFenceFd)));
        }

        int64_t timestampNs = 0;
        if (!msg->findInt64("timestampNs", &timestampNs)) {
            // use media timestamp if client did not request a specific render timestamp
            if (info->mData->meta()->findInt64("timeUs", &timestampNs)) {
                ALOGV("using buffer PTS of %lld", (long long)timestampNs);
                timestampNs *= 1000;
            }
        }

        status_t err;
        err = native_window_set_buffers_timestamp(mCodec->mNativeWindow.get(), timestampNs);
        ALOGW_IF(err != NO_ERROR, "failed to set buffer timestamp: %d", err);

        info->checkReadFence("onOutputBufferDrained before queueBuffer");
        err = mCodec->mNativeWindow->queueBuffer(
                    mCodec->mNativeWindow.get(), info->mGraphicBuffer.get(), info->mFenceFd);
        info->mFenceFd = -1;
        if (err == OK) {
            info->mStatus = BufferInfo::OWNED_BY_NATIVE_WINDOW;
        } else {
            ALOGE("queueBuffer failed in onOutputBufferDrained: %d", err);
            mCodec->signalError(OMX_ErrorUndefined, makeNoSideEffectStatus(err));
            info->mStatus = BufferInfo::OWNED_BY_US;
            // keeping read fence as write fence to avoid clobbering
            info->mIsReadFence = false;
        }
    } else {
        if (mCodec->mNativeWindow != NULL &&
            (info->mData == NULL || info->mData->size() != 0)) {
            // move read fence into write fence to avoid clobbering
            info->mIsReadFence = false;
            ATRACE_NAME("frame-drop");
        }
        info->mStatus = BufferInfo::OWNED_BY_US;
    }

    ......
}
```
##### 5.1、Surfaceflinger 视频解码缓存申请
前面2.3.6、MediaCodec->start()分析过：
产生kWhatStart消息，消息处理中先将MediaCodec状态设为STARTING，然后调用ACodec::initiateStart()产生kWhatStart消息，在其消息处理中又调用ACodec::LoadedState::onStart()，然后在其中首先向IOMX发送状态转换命令，经过OMXNodeInstance最终对将OMX组件状态转换成Idle（转换完成时OMX会发送OMX_EventCmdComplete事件），接着对ACodec进行changeState至LoadedToIdleState。而在changeState过程中会调用ACodec::LoadedToIdleState::stateEntered() =>  ACodec::LoadedToIdleState::allocateBuffers() => ACodec::allocateBuffersOnPort(…)，其中会为OMX组件端口分配缓冲，并向MediaCodec发送消息kWhatBuffersAllocated，消息处理中将MediaCodec状态设为STARTED而若allocateBuffers失败则由IOMX经OMXNodeInstance将OMX组件转换回Loaded状态，同时把ACodec状态转换回LoadedState
``` cpp
[->\frameworks\av\media\libstagefright\ACodec.cpp]
//使用surface渲染，为输出分配图形缓存GraphicBuffer  
status_t ACodec::LoadedToIdleState::allocateBuffers() {
    status_t err = mCodec->allocateBuffersOnPort(kPortIndexInput);
    if (err != OK) {
        return err;
    }
    return mCodec->allocateBuffersOnPort(kPortIndexOutput);
}
status_t ACodec::allocateBuffersOnPort(OMX_U32 portIndex) {
    CHECK(portIndex == kPortIndexInput || portIndex == kPortIndexOutput);

    CHECK(mDealer[portIndex] == NULL);
    CHECK(mBuffers[portIndex].isEmpty());

    status_t err;
    if (mNativeWindow != NULL && portIndex == kPortIndexOutput) {
        if (storingMetadataInDecodedBuffers()) {
            err = allocateOutputMetadataBuffers();
        } else {
            err = allocateOutputBuffersFromNativeWindow();
        }
    } 
    ......
}
```
##### 5.1.1、allocateOutputBuffersFromNativeWindow()的实现

``` cpp
[->\frameworks\av\media\libstagefright\ACodec.cpp]
status_t ACodec::allocateOutputBuffersFromNativeWindow() {
    OMX_U32 bufferCount, bufferSize, minUndequeuedBuffers;
    status_t err = configureOutputBuffersFromNativeWindow(
            &bufferCount, &bufferSize, &minUndequeuedBuffers, true /* preregister */);
    if (err != 0)
        return err;
    mNumUndequeuedBuffers = minUndequeuedBuffers;

    if (!storingMetadataInDecodedBuffers()) {
        static_cast<Surface*>(mNativeWindow.get())
                ->getIGraphicBufferProducer()->allowAllocation(true);
    }
    ......
    // Dequeue buffers and send them to OMX
    for (OMX_U32 i = 0; i < bufferCount; i++) {
        ANativeWindowBuffer *buf;
        int fenceFd;
        err = mNativeWindow->dequeueBuffer(mNativeWindow.get(), &buf, &fenceFd);
        ......
        sp<GraphicBuffer> graphicBuffer(new GraphicBuffer(buf, false));
        BufferInfo info;
        info.mStatus = BufferInfo::OWNED_BY_US;
        info.mFenceFd = fenceFd;
        info.mIsReadFence = false;
        info.mRenderInfo = NULL;
        info.mData = new ABuffer(NULL /* data */, bufferSize /* capacity */);
        info.mCodecData = info.mData;
        info.mGraphicBuffer = graphicBuffer;
        mBuffers[kPortIndexOutput].push(info);

        IOMX::buffer_id bufferId;
        err = mOMX->useGraphicBuffer(mNode, kPortIndexOutput, graphicBuffer,
                &bufferId);
        ......
        mBuffers[kPortIndexOutput].editItemAt(i).mBufferID = bufferId;

        ......
    }

    ......
    return err;
}
```
##### 5.1.1.1、首先为视频编码输出准备Surface
此处通过Binder通信使用IGraphicBufferProducer请求分配一个Native Surface
``` cpp
static_cast<Surface*>(mNativeWindow.get())
        ->getIGraphicBufferProducer()->allowAllocation(true);
```
![Alt text](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/video.system/02-06-Surface-ANativeWindow.png)


##### 5.1.1.2、Surface->dequeueBuffer
为Surface分配Buffer，提供给视频解码后数据使用
``` cpp
[->\frameworks\av\media\libstagefright\ACodec.cpp]
status_t ACodec::allocateOutputBuffersFromNativeWindow() {
    for (OMX_U32 i = 0; i < bufferCount; i++) {
        ANativeWindowBuffer *buf;
        int fenceFd;
        err = mNativeWindow->dequeueBuffer(mNativeWindow.get(), &buf, &fenceFd);
        ......
}

```

##### 5.2、Surface->queueBuffer()
待视频解码后，使用queueBuffer()交给SurfaceFlinger渲染，就可以在屏幕上看到视频画面了。
``` cpp
[->\frameworks\av\media\libstagefright\ACodec.cpp]
void ACodec::BaseState::onOutputBufferDrained(const sp<AMessage> &msg) {
        err = mCodec->mNativeWindow->queueBuffer(
                    mCodec->mNativeWindow.get(), info->mGraphicBuffer.get(), info->mFenceFd);
        ...
}
```
![Alt text](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/video.system/02-07-OpenMax-Based video decode-surfaceflinger.png)

关于SurfaceFlinger的知识请参考：【Android 7.1.2 (Android N) Android Graphics 系统分析】

**（ ͡° ͜ʖ ͡°）、（ಡωಡ）累~~~，有时间再继续Todo的分析吧，(๑乛◡乛๑) ！！！**
**Todo：Android OpenMax机制 实现分析**
**Todo：Android 音视频同步机制 源码分析**
**Todo：Android 音视频录制（Recoder）、编码（Encode）、混合（MediaMuxer）源码分析**

#### （六）、参考资料(特别感谢各位前辈的分析和图示)：
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
[ffmpeg开发之旅(1)-(7)（总共七篇）](https://blog.csdn.net/column/details/16222.html)
[深入理解Android音视频同步机制（总共五篇）](https://blog.csdn.net/nonmarking/article/category/6746500)
[Android硬编码——音频编码、视频编码及音视频混合](https://blog.csdn.net/junzia/article/details/54018671)

