---
title: Android Video System（5）：Android Multimedia - NuPlayer音视频同步实现分析
cover: https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/hexo.themes/bing-wallpaper-2018.04.26.jpg
categories:
  - Multimedia
tags:
  - Android
  - Video
  - Multimedia
toc: true
abbrlink: 20190212
date: 2019-02-12 09:25:00
---

--------------------------------------------------------------------------------

注：文章都是通过阅读各位前辈总结的资料 Android 7.1.2 && Linux（kernel 3.18）Qualcomm平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除，禁止转载（©Qualcomm Technologies, Inc. 版权所有），谢谢。

[【特别感谢 -  Android MediaCodec ACodec】](https://blog.csdn.net/dfhuang09/article/details/60132620)
[【特别感谢 -  android ACodec MediaCodec NuPlayer flow】](https://blog.csdn.net/dfhuang09/article/details/54926526)


Google Pixel、Pixel XL 内核代码（文章基于 Kernel-3.18）：
 [Kernel source for Pixel and Pixel XL - GitHub](https://github.com/matthewdalex/marlin)

AOSP 源码（文章基于 Android 7.1.2）：
 [ Android 系统全套源代码分享 (更新到 8.1.0_r1)](https://testerhome.com/topics/2229)

--------------------------------------------------------------------------------
#### （一）、音视频同步时如何实现的？
从Renderer接口层来看，没有任何关于同步处理的接口，仅有有限的几个控制接口flush/pause/resume，以及queueBuffer/queueEOS接口。同步问题的核心就在于ALooper-AHandler机制。其实真正的同步都是在消息循环的响应函数里实现的。先看音频。

##### 1.1、Renderer中的音频同步机制
起始位置从音频PCM数据进入开始，处理在Renderer::queueBuffer()中，最终发送了kWhatQueueBuffer消息。这个消息的实际处理函数是Renderer::onQueueBuffer()。实际代码在“音视频原始数据输入——queueBuffer”中有，这里仅针对音频流程解释下。 基本逻辑很简单，保存传入的buffer参数，并通知输出下AudioQueue。

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayerRenderer.cpp]
void NuPlayer::Renderer::onQueueBuffer(const sp<AMessage> &msg) {
......
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
......
}
```

下面看看postDrainAudioQueue_l的实现，内部实现逻辑基本上就是边界判断加上发送kWhatDrainAudioQueue消息。

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayerRenderer.cpp]
void NuPlayer::Renderer::postDrainAudioQueue_l(int64_t delayUs) {
    if (mAudioQueue.empty()) return;

    mDrainAudioQueuePending = true;
    sp<AMessage> msg = new AMessage(kWhatDrainAudioQueue, this);
    msg->setInt32("drainGeneration", mAudioDrainGeneration);
    msg->post(delayUs);
}
```

那就继续查看下这个消息如何处理的。

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayerRenderer.cpp]
        case kWhatDrainAudioQueue:
        {
            mDrainAudioQueuePending = false;
            if (onDrainAudioQueue()) {
                uint32_t numFramesPlayed;
                uint32_t numFramesPendingPlayout = mNumFramesWritten - numFramesPlayed;

                // 这里是audio sink中缓存了多长的可用于播放的数据
                int64_t delayUs = mAudioSink->msecsPerFrame() * numFramesPendingPlayout * 1000ll;
                if (mPlaybackRate > 1.0f) {
                    delayUs /= mPlaybackRate;
                }

                // 利用一半的延时来保证下次刷新时间（注意时间上有重叠）
                delayUs /= 2;
                // 参考buffer大小来估计最大的延时时间
                const int64_t maxDrainDelayUs = std::max(
                        mAudioSink->getBufferDurationInUs(), (int64_t)500000 /* half second */);
                ALOGD_IF(delayUs > maxDrainDelayUs, "postDrainAudioQueue long delay: %lld > %lld",
                        (long long)delayUs, (long long)maxDrainDelayUs);
                Mutex::Autolock autoLock(mLock);
                postDrainAudioQueue_l(delayUs); // 这里同一个消息重发了
            }
            break;
        }
```
到这里，貌似还是没有同步的机制，不过我们已经知道这个音频播放消息的触发机制了，在queueBuffer和消息处理函数中都会触发，基本上就是定时器。还有最后一个函数onDrainAudioQueue()。下面是代码：

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayerRenderer.cpp]
bool NuPlayer::Renderer::onDrainAudioQueue() {
    uint32_t numFramesPlayed;
    if (mAudioSink->getPosition(&numFramesPlayed) != OK) {      
        drainAudioQueueUntilLastEOS();
        ALOGW("onDrainAudioQueue(): audio sink is not ready");
        return false;
    }

    uint32_t prevFramesWritten = mNumFramesWritten;
    while (!mAudioQueue.empty()) {
        QueueEntry *entry = &*mAudioQueue.begin();

        mLastAudioBufferDrained = entry->mBufferOrdinal;

        if (entry->mBuffer == NULL) {
            // 删除针对EOS的处理代码            
        }

        // ignore 0-sized buffer which could be EOS marker with no data
        if (entry->mOffset == 0 && entry->mBuffer->size() > 0) {
            int64_t mediaTimeUs;
            CHECK(entry->mBuffer->meta()->findInt64("timeUs", &mediaTimeUs));
            ALOGV("onDrainAudioQueue: rendering audio at media time %.2f secs",
                    mediaTimeUs / 1E6);
            onNewAudioMediaTime(mediaTimeUs);
        }

        size_t copy = entry->mBuffer->size() - entry->mOffset;
        ssize_t written = mAudioSink->write(entry->mBuffer->data() + entry->mOffset,
                                            copy, false /* blocking */);
        if (written < 0) {/* ...忽略异常处理部分代码 */}

        entry->mOffset += written;
        size_t remainder = entry->mBuffer->size() - entry->mOffset;
        if ((ssize_t)remainder < mAudioSink->frameSize()) {
            if (remainder > 0) {// 这是直接凑成完整的一帧音频
                ALOGW("Corrupted audio buffer has fractional frames, discarding %zu bytes.", remainder);
                entry->mOffset += remainder;
                copy -= remainder;
            }

            entry->mNotifyConsumed->post();
            mAudioQueue.erase(mAudioQueue.begin());
            entry = NULL;
        }

        size_t copiedFrames = written / mAudioSink->frameSize();
        mNumFramesWritten += copiedFrames;

        {
            Mutex::Autolock autoLock(mLock);
            int64_t maxTimeMedia;
            maxTimeMedia = mAnchorTimeMediaUs +
                        (int64_t)(max((long long)mNumFramesWritten - mAnchorNumFramesWritten, 0LL)
                                * 1000LL * mAudioSink->msecsPerFrame());
            mMediaClock->updateMaxTimeMedia(maxTimeMedia);

            notifyIfMediaRenderingStarted_l();
        }

        if (written != (ssize_t)copy) {
            // A short count was received from AudioSink::write()
            //
            // AudioSink write is called in non-blocking mode.
            // It may return with a short count when:
            //
            // 1) Size to be copied is not a multiple of the frame size. Fractional frames are
            //    discarded.
            // 2) The data to be copied exceeds the available buffer in AudioSink.
            // 3) An error occurs and data has been partially copied to the buffer in AudioSink.
            // 4) AudioSink is an AudioCache for data retrieval, and the AudioCache is exceeded.

            // (Case 1)
            // Must be a multiple of the frame size.  If it is not a multiple of a frame size, it
            // needs to fail, as we should not carry over fractional frames between calls.
            CHECK_EQ(copy % mAudioSink->frameSize(), 0);

            // (Case 2, 3, 4)
            // Return early to the caller.
            // Beware of calling immediately again as this may busy-loop if you are not careful.
            ALOGV("AudioSink write short frame count %zd < %zu", written, copy);
            break;
        }
    }

    // calculate whether we need to reschedule another write.
    bool reschedule = !mAudioQueue.empty()
            && (!mPaused
                || prevFramesWritten != mNumFramesWritten); // permit pause to fill buffers
    //ALOGD("reschedule:%d  empty:%d  mPaused:%d  prevFramesWritten:%u  mNumFramesWritten:%u",
    //        reschedule, mAudioQueue.empty(), mPaused, prevFramesWritten, mNumFramesWritten);
    return reschedule;
}
```
这里面比较主要的更新是onNewAudioMediaTime和mNumFramesWritten字段。
剩下的一部分代码是关于异常边界情况下的音视频处理逻辑：

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayerRenderer.cpp]
sp<ABuffer> firstAudioBuffer = (*mAudioQueue.begin()).mBuffer;
    sp<ABuffer> firstVideoBuffer = (*mVideoQueue.begin()).mBuffer;

    if (firstAudioBuffer == NULL || firstVideoBuffer == NULL) {
        // 对于一个队列为空的情况，通知另个一队列EOS
        syncQueuesDone_l();
        return;
    }

    int64_t firstAudioTimeUs;
    int64_t firstVideoTimeUs;
    CHECK(firstAudioBuffer->meta()
            ->findInt64("timeUs", &firstAudioTimeUs));
    CHECK(firstVideoBuffer->meta()
            ->findInt64("timeUs", &firstVideoTimeUs));

    int64_t diff = firstVideoTimeUs - firstAudioTimeUs;
    if (diff > 100000ll) {
        // 音频数据时间戳比视频数据早0.1s，

        (*mAudioQueue.begin()).mNotifyConsumed->post();
        mAudioQueue.erase(mAudioQueue.begin());
        return;
    }

    syncQueuesDone_l();
```
##### 1.2、Renderer中的视频同步部分
和音频同步类似，入口在在Renderer::queueBuffer()，主要区分在Renderer::onQueueBuffer()中，代码如下：

``` cpp
// 如果是视频，则将数据存放到视频队列，然后安排刷新
mVideoQueue.push_back(entry);
postDrainVideoQueue();
```
下面按照之前的思路继续分析，接下来是postDrainVideoQueue实现，主要音视频同步逻辑位于这里。

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayerRenderer.cpp]
void NuPlayer::Renderer::postDrainVideoQueue() {
    if (mVideoQueue.empty()) {
        return;
    }

    QueueEntry &entry = *mVideoQueue.begin();

    sp<AMessage> msg = new AMessage(kWhatDrainVideoQueue, this); //这是实际处理视频缓冲区和显示的消息
    msg->setInt32("drainGeneration", getDrainGeneration(false /* audio */));

    if (entry.mBuffer == NULL) {
        // EOS doesn't carry a timestamp.
        msg->post();
        mDrainVideoQueuePending = true;
        return;
    }

    bool needRepostDrainVideoQueue = false;
    int64_t delayUs;
    int64_t nowUs = ALooper::GetNowUs();
    int64_t realTimeUs;
    int64_t mediaTimeUs;
    CHECK(entry.mBuffer->meta()->findInt64("timeUs", &mediaTimeUs));
    if (mFlags & FLAG_REAL_TIME) {        
        realTimeUs = mediaTimeUs;
    } else {
        {
            Mutex::Autolock autoLock(mLock);
            if (mAnchorTimeMediaUs < 0) { // 同步基准未设置的情况下，直接显示
                mMediaClock->updateAnchor(mediaTimeUs, nowUs, mediaTimeUs);
                mAnchorTimeMediaUs = mediaTimeUs;
                realTimeUs = nowUs;
            } else if (!mVideoSampleReceived) { // 第一帧未显示前，直接显示
                // Always render the first video frame.
                realTimeUs = nowUs;
            } else if (mAudioFirstAnchorTimeMediaUs < 0 // 音频未播放之前，以视频为准
                || mMediaClock->getRealTimeFor(mediaTimeUs, &realTimeUs) == OK) {
                realTimeUs = getRealTimeUs(mediaTimeUs, nowUs);
            } else if (mediaTimeUs - mAudioFirstAnchorTimeMediaUs >= 0) { // 视频超前的情况下，等待
                needRepostDrainVideoQueue = true; 
                realTimeUs = nowUs;
            } else {
                realTimeUs = nowUs;
            }
        }

        // Heuristics to handle situation when media time changed without a
        // discontinuity. If we have not drained an audio buffer that was
        // received after this buffer, repost in 10 msec. Otherwise repost
        // in 500 msec.
        delayUs = realTimeUs - nowUs;
        int64_t postDelayUs = -1;
        if (delayUs > 500000) {
            postDelayUs = 500000;
            if (mHasAudio && (mLastAudioBufferDrained - entry.mBufferOrdinal) <= 0) {
                postDelayUs = 10000;
            }
        } else if (needRepostDrainVideoQueue) {
            // CHECK(mPlaybackRate > 0);
            // CHECK(mAudioFirstAnchorTimeMediaUs >= 0);
            // CHECK(mediaTimeUs - mAudioFirstAnchorTimeMediaUs >= 0);
            postDelayUs = mediaTimeUs - mAudioFirstAnchorTimeMediaUs;
            postDelayUs /= mPlaybackRate;
        }

        if (postDelayUs >= 0) {
            msg->setWhat(kWhatPostDrainVideoQueue);
            msg->post(postDelayUs);
            mVideoScheduler->restart();
            ALOGI("possible video time jump of %dms or uninitialized media clock, retrying in %dms",
                    (int)(delayUs / 1000), (int)(postDelayUs / 1000));
            mDrainVideoQueuePending = true;
            return;
        }
    }

    realTimeUs = mVideoScheduler->schedule(realTimeUs * 1000) / 1000;
    int64_t twoVsyncsUs = 2 * (mVideoScheduler->getVsyncPeriod() / 1000);

    delayUs = realTimeUs - nowUs;
    // 上面代码的主要目的是计算这个延时
    ALOGW_IF(delayUs > 500000, "unusually high delayUs: %" PRId64, delayUs);
    // post 2 display refreshes before rendering is due
    msg->post(delayUs > twoVsyncsUs ? delayUs - twoVsyncsUs : 0);

    mDrainVideoQueuePending = true;
}
```
这里主要的是发送了一个延时消息kWhatDrainVideoQueue，下面是如何处理的代码：

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayerRenderer.cpp]
case kWhatDrainVideoQueue:
{
    int32_t generation;
    CHECK(msg->findInt32("drainGeneration", &generation));
    if (generation != getDrainGeneration(false /* audio */)) {
        break;
    }

    mDrainVideoQueuePending = false;
    onDrainVideoQueue();
    postDrainVideoQueue(); // 注意这里相当于定时器的实现了
    break;
}
```

直接调用onDrainVideoQueue函数，看看如何实现的：

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayerRenderer.cpp]
void NuPlayer::Renderer::onDrainVideoQueue() {
    if (mVideoQueue.empty()) {
        return;
    }

    QueueEntry *entry = &*mVideoQueue.begin();
    if (entry->mBuffer == NULL) {
        // ...省略针对EOS 处理
    }

    int64_t nowUs = ALooper::GetNowUs();
    int64_t realTimeUs;
    int64_t mediaTimeUs = -1;
    if (mFlags & FLAG_REAL_TIME) {
        CHECK(entry->mBuffer->meta()->findInt64("timeUs", &realTimeUs));
    } else {
        CHECK(entry->mBuffer->meta()->findInt64("timeUs", &mediaTimeUs));
        realTimeUs = getRealTimeUs(mediaTimeUs, nowUs);
    }

    bool tooLate = false;
    if (!mPaused) {
        setVideoLateByUs(nowUs - realTimeUs);
        tooLate = (mVideoLateByUs > 40000);

        if (tooLate) {
            ALOGV("video late by %lld us (%.2f secs)",
                 (long long)mVideoLateByUs, mVideoLateByUs / 1E6);
        } else {
            int64_t mediaUs = 0;
            mMediaClock->getMediaTime(realTimeUs, &mediaUs);
            ALOGV("rendering video at media time %.2f secs",
                    (mFlags & FLAG_REAL_TIME ? realTimeUs :
                    mediaUs) / 1E6);

            if (!(mFlags & FLAG_REAL_TIME)
                    && mLastAudioMediaTimeUs != -1
                    && mediaTimeUs > mLastAudioMediaTimeUs) {
                // If audio ends before video, video continues to drive media clock.
                // Also smooth out videos >= 10fps.
                mMediaClock->updateMaxTimeMedia(mediaTimeUs + 100000);
            }
        }
    } else {
        setVideoLateByUs(0);
        if (!mVideoSampleReceived && !mHasAudio) {
            // This will ensure that the first frame after a flush won't be used as anchor
            // when renderer is in paused state, because resume can happen any time after seek.
            Mutex::Autolock autoLock(mLock);
            clearAnchorTime_l();
        }
    }

    // Always render the first video frame while keeping stats on A/V sync.
    if (!mVideoSampleReceived) {
        realTimeUs = nowUs;
        tooLate = false;
    }

    entry->mNotifyConsumed->setInt64("timestampNs", realTimeUs * 1000ll); // 上面所有计算的参数在这里使用了
    entry->mNotifyConsumed->setInt32("render", !tooLate);
    entry->mNotifyConsumed->post(); // 注意这里，实际是向解码器发送消息，用于显示
    mVideoQueue.erase(mVideoQueue.begin());
    entry = NULL;

    mVideoSampleReceived = true;

    if (!mPaused) { // 这里是通知NuPlayer层渲染开始
        if (!mVideoRenderingStarted) {
            mVideoRenderingStarted = true;
            notifyVideoRenderingStart();
        }
        Mutex::Autolock autoLock(mLock);
        notifyIfMediaRenderingStarted_l();
    }
}
```

到这里，小结下，读完这部分代码发现，NuPlayer::Renderer使用的以视频为基准的同步机制，音频晚了直接丢包，视频需要显示。同步主要位于视频缓冲区处理部分onDrainVideoQueue和音频缓冲区处理部分onDrainVideoQueue中。
![Alt text | center](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/video.system/VS-05-02-avsync.jpg)


#### （二）、音视频同步时序图

![Alt text | center](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/video.system/VS-05-01-AudioVideoSync.png)

1：OMX component 集成在ACodec中，ACodec（A/V）解完数据后，通知Nulayer；
2：NuPlayer通知Render，Render需要A/V的时间同步（另，如果是JPEG的话就不需要这个同步，直接render即可）；
3：对于Audio，直接通过AudioSink播放；
4：对于Video，通过通知ACodec，让ACodec通过(NativeWindow/Render)发送到界面

Step1:
1.1 omx_message::FILL_BUFFER_DONE ===>>>ACodec::onOMXFillBufferDone()
  OMX component send msg FILL_BUFFER_DONE
  ACodec call onOMXFillBufferDone to handle
1.2 ACodec::onOMXFillBufferDone()::ACodec::kWhatDrainThisBuffer setMessage ===>>> NuPlayer::onMessageReceived()
Step2:
2.1 NuPlayer::renderBuffer()
2.2 NuPlayer::Renderer::queueBuffer() ===>>> send msg kWhatQueueBuffer
2.3 NuPlayer::Renderer::onMessageReceived() ===>>> onQueueBuffer()
2.4 postDrainAudioQueue() or postDrainVideoQueue() ===>>> send msg kWhatDrainVideoQueue
2.5 onMessageReceived() ===>>> onDrainVideoQueue(); postDrainVideoQueue();
onDrainVideoQueue():A/V的时间同步,如果慢0.4s,标记too_late
postDrainVideoQueue():A/V的时间同步,如果解码时间快，决定等待的时间，并把消息给render
Step3：
3.1 postDrainAudioQueue()===>>>onDrainAudioQueue()===>>>mAudioSink->write()
Step4:
4.1 Renderer::onDrainVideoQueue(): entry->mNotifyConsumed->setInt32("render", !tooLate);
4.2 Renderer::postDrainVideoQueue()===>>> send msg kWhatDrainVideoQueue
4.2 ACodec::BaseState::onMessageReceived() ===>>> onOutputBufferDrained(msg);


``` cpp
Log:
...
07-03 11:32:58.675   741  5075 V NuPlayerRenderer: rendering video at media time 0.36 secs
07-03 11:32:58.676   725  1139 V AudioFlinger: releaseWakeLock_l() AudioOut_25
07-03 11:32:58.676   725  1139 V AudioFlinger: thread 0xef883380 type 0 TID 1139 going to sleep
07-03 11:32:58.679   725  1131 V AudioFlinger: releaseWakeLock_l() AudioOut_D
07-03 11:32:58.682   741  5075 V NuPlayerRenderer: onDrainAudioQueue: rendering audio at media time 0.70 secs
07-03 11:32:58.683   741  5075 V AudioTrack: frame adjustment:2400  timestamp:BOOTTIME offset 0
07-03 11:32:58.683   741  5075 V AudioTrack: ExtendedTimestamp[0]  position: 0  time: -1
07-03 11:32:58.683   741  5075 V AudioTrack: ExtendedTimestamp[1]  position: 17280  time: 96476185950
07-03 11:32:58.683   741  5075 V AudioTrack: ExtendedTimestamp[2]  position: 0  time: -1
07-03 11:32:58.683   741  5075 V AudioTrack: ExtendedTimestamp[3]  position: 0  time: -1
07-03 11:32:58.683   741  5075 V AudioTrack: ExtendedTimestamp[4]  position: 0  time: -1
07-03 11:32:58.683   741  5075 V AudioSink: getPlayedOutDurationUs(370130) nowUs(96536315) frames(14880) framesAt(96476185)
07-03 11:32:58.684   741  5075 V NuPlayerRenderer: onDrainAudioQueue: rendering audio at media time 0.73 secs
07-03 11:32:58.694   725  1131 V AudioFlinger: thread 0xf0208880 type 0 TID 1131 going to sleep
07-03 11:32:58.701   741  5075 V NuPlayerRenderer: onDrainAudioQueue: rendering audio at media time 0.75 secs
07-03 11:32:58.717   741  5075 V NuPlayerRenderer: rendering video at media time 0.41 secs
07-03 11:32:58.733   741  5075 V NuPlayerRenderer: onDrainAudioQueue: rendering audio at media time 0.77 secs
...
```
#### （三）、参考资料(特别感谢各位前辈的分析和图示)：
[④NuPlayer播放框架之Renderer源码分析：音视频同步时如何实现的？](http://windrunnerlihuan.com/2016/12/15/Android%E5%A4%9A%E5%AA%92%E4%BD%93%E5%BC%80%E5%8F%91-%E4%BA%94-OpenMax%E7%AE%80%E4%BB%8B/)
[android ACodec MediaCodec NuPlayer flow - CSDN博客](https://blog.csdn.net/dfhuang09/article/details/54926526)
[android MediaCodec ACodec - CSDN博客](https://blog.csdn.net/dfhuang09/article/details/60132620)

