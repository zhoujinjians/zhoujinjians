---
title: Android Video System（6）：Android Multimedia - NuPlayer HLS流媒体协议、RTSP流媒体协议
cover: https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/hexo.themes/bing-wallpaper-2018.04.27.jpg
categories:
  - Multimedia
tags:
  - Android
  - Video
  - Multimedia
toc: true
abbrlink: 20190213
date: 2019-02-13 09:25:00
---

--------------------------------------------------------------------------------

注：文章都是通过阅读各位前辈总结的资料 Android 7.1.2 && Linux（kernel 3.18）Qualcomm平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网（◕‿◕），如有侵权，请联系删除，禁止转载（©Qualcomm Technologies, Inc. 版权所有），谢谢。


首先特别感谢：

[Android 源码分析之基于NuPlayer的HLS流媒体协议](https://tbfungeek.github.io/2016/08/02/Android-%E8%BF%9B%E9%98%B6%E4%B9%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E5%9F%BA%E4%BA%8ENuPlayer%E7%9A%84HLS%E6%B5%81%E5%AA%92%E4%BD%93%E5%8D%8F%E8%AE%AE/)
[Android 源码分析之基于NuPlayer的RTSP流媒体协议](https://tbfungeek.github.io/2016/08/02/Android-%E8%BF%9B%E9%98%B6%E4%B9%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E5%9F%BA%E4%BA%8ENuPlayer%E7%9A%84RTSP%E6%B5%81%E5%AA%92%E4%BD%93%E5%8D%8F%E8%AE%AE/)

正是由于前人的分析和总结，帮助我节约了大量的时间和精力，再次感谢！！！

Google Pixel、Pixel XL 内核代码（文章基于 Kernel-3.18）：
 [Kernel source for Pixel and Pixel XL - GitHub](https://github.com/matthewdalex/marlin)

AOSP 源码（文章基于 Android 7.1.2）：
 [ Android 系统全套源代码分享 (更新到 8.1.0_r1)](https://testerhome.com/topics/2229)

--------------------------------------------------------------------------------

> 注：本文基本转载于上述两篇博客！！！

#### （一）、基于NuPlayer的HLS流媒体协议

##### 1.1、HLS 概述
HTTP Live Streaming（HLS）是苹果公司实现的基于HTTP的流媒体直播和点播协议，主要应用在iOS系统。相对于普通的流媒体，例如RTMP协议、RTSP协议、MMS协议等，HLS最大的优点是可以根据网络状况自动切换到不同码率的视频，如果网络状况较好，则会切换到高码率的视频，若发现网络状况不佳，则会逐渐过渡到低码率的视频，这个我们下面将会结合代码对其进行说明。

##### 1.2、HLS框架介绍
我们接下来看下HLS系统的整体结构图：

![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/video.system/VS-06-01-HLS-playback-architecture.png)

我们首先将要直播的视频送到编码器中，编码器分别对视频和音频进行编码，然后输出到一个MPEG-2格式的传输流中，再由分段器将MPEG-2传输流进行分段，产生一系列等间隔的媒体片段，这些媒体片段一般很小并且保存成后缀为.ts的文件，同时生成一个指向这些媒体文件的索引文件，也就是我们很经常听到的.M3U8文件。完成分段之后将这些索引文件以及媒体文件上传到Web服务器上。客户端读取索引文件，然后按顺序请求下载索引文件中列出的媒体文件。下载后是一个ts文件。需要进行解压获得对应的媒体数据并解码后进行播放。由于在直播过程中服务器端会不断地将最新的直播数据生成新的小文件，并上传所以只要客户端不断地按顺序下载并播放从服务器获取到的文件，从整个过程上看就相当于实现了直播。而且由于分段文件的很短，客户端可以根据实际的带宽情况切换到不同码率的直播源，从而实现多码率的适配的目的。

M3U8 知识请参考：[HLS之m3u8、ts流格式详解](https://blog.csdn.net/Guofengpu/article/details/54922865)

##### 1.3、HLS播放流程
1、获取不同带宽下对应的网络资源URI及音视频编解码，视频分辨率等信息的文件

``` javascript
#EXT-X-STREAM-INF:PROGRAM-ID=1,BANDWIDTH=899152,RESOLUTION=480x270,CODECS="avc1.4d4015,mp4a.40.5"
http://hls.ftdp.com/video1_widld/m3u8/01.m3u8
```

2、根据上述获取的信息初始化对应的编解码器

3、获取第一个网络资源对应的分段索引列表（index文件）

``` javascript
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:10
#EXT-X-MEDIA-SEQUENCE:6532
#EXT-X-KEY:METHOD=AES-128,URI="18319965201.key"
#EXTINF:10,
20125484T125708-01-6533.ts
#EXT-X-KEY:METHOD=AES-128,URI="14319965205.key"
#EXTINF:10,
20125484T125708-01-6534.ts
....
#EXTINF:8,

20140804T125708-01-6593.ts
```
4、获取某一个分片的Key

5、请求下载某一个分片
6、根据当前的带宽决定是否切换视频资源
7、将下载的分片资源解密后送到解码器进行解码

![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/video.system/VS-06-02-HLS-playback-flow.png)



关于NuPlayerDrvier的创建以及SetDataSource的流程和Stagefight Player大体一致，区别在于setDataSource的时候是根据url的不同会创建三种不同的DataSource：HttpLiveSource，RTSPSource，以及GenericSource。这里就不做大篇幅的介绍了，就直接上图吧

![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/video.system/VS-06-03-HLS-nuplayer-setdatasource-java.png)


![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/video.system/VS-06-04-HLS-nuplayer-setdatasource-native.png)

![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/video.system/VS-06-05-HLS-nuplayer-setdatasource-native-2.png)


#### （二）、基于NuPlayer的HLS流媒体播放源码分析

首先看看总体时序图：

![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/video.system/VS-06-06-HttpLiveStream-flow.png)


##### 2.1、HTTPLiveSource::prepareAsync()
我们直接从prepare结合HLS原理开始分析，直接看HttpliveSource的prepareAsync。

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\HTTPLiveSource.cpp]
void NuPlayer::HTTPLiveSource::prepareAsync() {
    //创建并启动一个Looper
    if (mLiveLooper == NULL) {
        mLiveLooper = new ALooper;
        mLiveLooper->setName("http live");
        mLiveLooper->start();
        mLiveLooper->registerHandler(this);
    }
    //创建一个kWhatSessionNotify赋值给LiveSession用于通知
    sp<AMessage> notify = new AMessage(kWhatSessionNotify, this);
    //创建一个LiveSession
    mLiveSession = new LiveSession(
            notify,
            (mFlags & kFlagIncognito) ? LiveSession::kFlagIncognito : 0,
            mHTTPService);
    mLiveLooper->registerHandler(mLiveSession);
    //使用LiveSession进行异步连接
    mLiveSession->connectAsync(mURL.c_str(), mExtraHeaders.isEmpty() ? NULL : &mExtraHeaders);
}
```

``` cpp
[->\frameworks\av\media\libstagefright\httplive\LiveSession.cpp]
void LiveSession::connectAsync(const char *url, const KeyedVector<String8, String8> *headers) {
    //创建一个kWhatConnect并传入url
    sp<AMessage> msg = new AMessage(kWhatConnect, this);
    msg->setString("url", url);
    if (headers != NULL) {
        msg->setPointer("headers",new KeyedVector<String8, String8>(*headers));
    }
    msg->post();
}

void LiveSession::onMessageReceived(const sp<AMessage> &msg) {
	case kWhatConnect:
    {
        //调用onConnect
        onConnect(msg);
        break;
    }
}

void LiveSession::onConnect(const sp<AMessage> &msg) {
    //获取传过来的Uri
    CHECK(msg->findString("url", &mMasterURL));
    KeyedVector<String8, String8> *headers = NULL;
    if (!msg->findPointer("headers", (void **)&headers)) {
        mExtraHeaders.clear();
    } else {
        mExtraHeaders = *headers;
        delete headers;
        headers = NULL;
    }
    //创建一个mFetcherLooper
    if (mFetcherLooper == NULL) {
        mFetcherLooper = new ALooper();
        mFetcherLooper->setName("Fetcher");
        mFetcherLooper->start(false, false);
    }
    //获取不同带宽下对应的网络资源URI及音视频编解码信息
    addFetcher(mMasterURL.c_str())->fetchPlaylistAsync();
}
```
这里就开始获取不同带宽下对应的网络资源URI及音视频编解码信息了

``` cpp
[->\frameworks\av\media\libstagefright\httplive\LiveSession.cpp]
sp<PlaylistFetcher> LiveSession::addFetcher(const char *uri) {

    ssize_t index = mFetcherInfos.indexOfKey(uri);
    sp<AMessage> notify = new AMessage(kWhatFetcherNotify, this);
    notify->setString("uri", uri);
    notify->setInt32("switchGeneration", mSwitchGeneration);
    FetcherInfo info;
    //创建一个PlaylistFetcher并返回
    info.mFetcher = new PlaylistFetcher(notify, this, uri, mCurBandwidthIndex, mSubtitleGeneration);
    info.mDurationUs = -1ll;
    info.mToBeRemoved = false;
    info.mToBeResumed = false;
    mFetcherLooper->registerHandler(info.mFetcher);
    mFetcherInfos.add(uri, info);
    //这里的info.mFetcher是上面new 出来的PlaylistFetcher
    return info.mFetcher;
}
```
我们通过这里返回的PlaylistFetcher调用fetchPlaylistAsync来获取playlists
``` cpp
[->\frameworks\av\media\libstagefright\httplive\PlaylistFetcher.cpp]
void PlaylistFetcher::fetchPlaylistAsync() {
    (new AMessage(kWhatFetchPlaylist, this))->post();
}

void PlaylistFetcher::onMessageReceived(const sp<AMessage> &msg) {
    case kWhatFetchPlaylist:
    {
        bool unchanged;
        //获取一个M3U8Parser
        sp<M3UParser> playlist = mHTTPDownloader->fetchPlaylist(mURI.c_str(), NULL /* curPlaylistHash */, &unchanged);
        sp<AMessage> notify = mNotify->dup();
        notify->setInt32("what", kWhatPlaylistFetched);
        //将playlist返回
        notify->setObject("playlist", playlist);
        notify->post();
        break;
    }
}
```
我们接下来看下fetchFile过程：首先会通过fetchFile从服务器端获取到m3u8 playlist内容存放到buffer缓存区，然后将获取到的缓存数据包装成M3UParser

``` cpp
[->\frameworks\av\media\libstagefright\httplive\HTTPDownloader.cpp]
sp<M3UParser> HTTPDownloader::fetchPlaylist(
        const char *url, uint8_t *curPlaylistHash, bool *unchanged) {

    *unchanged = false;
    sp<ABuffer> buffer;
    String8 actualUrl;
    //调用fetchFile
    ssize_t err = fetchFile(url, &buffer, &actualUrl);
	//断开连接
    mHTTPDataSource->disconnect();
 	//将获取到的缓存数据包装成M3UParser
    sp<M3UParser> playlist = new M3UParser(actualUrl.string(), buffer->data(), buffer->size());
    return playlist;
}
ssize_t HTTPDownloader::fetchFile(
        const char *url, sp<ABuffer> *out, String8 *actualUrl) {
    ssize_t err = fetchBlock(url, out, 0, -1, 0, actualUrl, true /* reconnect */);
    // close off the connection after use
    mHTTPDataSource->disconnect();
    return err;
}
```
我们这里看下M3UParser构造方法：

``` cpp
[->\frameworks\av\media\libstagefright\httplive\M3UParser.cpp]
M3UParser::M3UParser(
        const char *baseURI, const void *data, size_t size)
    : mInitCheck(NO_INIT),
      mBaseURI(baseURI),
      mIsExtM3U(false),
      mIsVariantPlaylist(false),
      mIsComplete(false),
      mIsEvent(false),
      mFirstSeqNumber(-1),
      mLastSeqNumber(-1),
      mTargetDurationUs(-1ll),
      mDiscontinuitySeq(0),
      mDiscontinuityCount(0),
      mSelectedIndex(-1) {
    mInitCheck = parse(data, size);
}
```
在最后的时候会调用parse对缓存数据进行解析：
``` cpp
[->\frameworks\av\media\libstagefright\httplive\M3UParser.cpp]
status_t M3UParser::parse(const void *_data, size_t size) {
    int32_t lineNo = 0;
    sp<AMessage> itemMeta;
    const char *data = (const char *)_data;
    size_t offset = 0;
    uint64_t segmentRangeOffset = 0;
    while (offset < size) {
        size_t offsetLF = offset;
        while (offsetLF < size && data[offsetLF] != '\n') {
            ++offsetLF;
        }
        AString line;
        if (offsetLF > offset && data[offsetLF - 1] == '\r') {
            line.setTo(&data[offset], offsetLF - offset - 1);
        } else {
            line.setTo(&data[offset], offsetLF - offset);
        }

        if (line.empty()) {
            offset = offsetLF + 1;
            continue;
        }
        if (lineNo == 0 && line == "#EXTM3U") {
            mIsExtM3U = true;
        }
        if (mIsExtM3U) {
            status_t err = OK;
            if (line.startsWith("#EXT-X-TARGETDURATION")) {
                if (mIsVariantPlaylist) {
                    return ERROR_MALFORMED;
                }
                err = parseMetaData(line, &mMeta, "target-duration");
            } else if (line.startsWith("#EXT-X-MEDIA-SEQUENCE")) {
                if (mIsVariantPlaylist) {
                    return ERROR_MALFORMED;
                }
                err = parseMetaData(line, &mMeta, "media-sequence");
            } else if (line.startsWith("#EXT-X-KEY")) {
                if (mIsVariantPlaylist) {
                    return ERROR_MALFORMED;
                }
                err = parseCipherInfo(line, &itemMeta, mBaseURI);
            } else if (line.startsWith("#EXT-X-ENDLIST")) {
                mIsComplete = true;
            } else if (line.startsWith("#EXT-X-PLAYLIST-TYPE:EVENT")) {
                mIsEvent = true;
            } else if (line.startsWith("#EXTINF")) {
                if (mIsVariantPlaylist) {
                    return ERROR_MALFORMED;
                }
                err = parseMetaDataDuration(line, &itemMeta, "durationUs");
            } else if (line.startsWith("#EXT-X-DISCONTINUITY")) {
                if (mIsVariantPlaylist) {
                    return ERROR_MALFORMED;
                }
                if (itemMeta == NULL) {
                    itemMeta = new AMessage;
                }
                itemMeta->setInt32("discontinuity", true);
                ++mDiscontinuityCount;
            } else if (line.startsWith("#EXT-X-STREAM-INF")) {
                if (mMeta != NULL) {
                    return ERROR_MALFORMED;
                }
                mIsVariantPlaylist = true;
                err = parseStreamInf(line, &itemMeta);
            } else if (line.startsWith("#EXT-X-BYTERANGE")) {
                if (mIsVariantPlaylist) {
                    return ERROR_MALFORMED;
                }
                uint64_t length, offset;
                err = parseByteRange(line, segmentRangeOffset, &length, &offset);
                if (err == OK) {
                    if (itemMeta == NULL) {
                        itemMeta = new AMessage;
                    }
                    itemMeta->setInt64("range-offset", offset);
                    itemMeta->setInt64("range-length", length);
                    segmentRangeOffset = offset + length;
                }
            } else if (line.startsWith("#EXT-X-MEDIA")) {
                err = parseMedia(line);
            } else if (line.startsWith("#EXT-X-DISCONTINUITY-SEQUENCE")) {
                if (mIsVariantPlaylist) {
                    return ERROR_MALFORMED;
                }
                size_t seq;
                err = parseDiscontinuitySequence(line, &seq);
                if (err == OK) {
                    mDiscontinuitySeq = seq;
                }
            }
            if (err != OK) {
                return err;
            }
        }
        if (!line.startsWith("#")) {
            if (!mIsVariantPlaylist) {
                int64_t durationUs;
                if (itemMeta == NULL
                        || !itemMeta->findInt64("durationUs", &durationUs)) {
                    return ERROR_MALFORMED;
                }
                itemMeta->setInt32("discontinuity-sequence",
                        mDiscontinuitySeq + mDiscontinuityCount);
            }
            mItems.push();
            Item *item = &mItems.editItemAt(mItems.size() - 1);
            CHECK(MakeURL(mBaseURI.c_str(), line.c_str(), &item->mURI));
            item->mMeta = itemMeta;
            itemMeta.clear();
        }
        offset = offsetLF + 1;
        ++lineNo;
    }

    if (!mIsVariantPlaylist) {
        int32_t targetDurationSecs;
        if (mMeta == NULL || !mMeta->findInt32(
                "target-duration", &targetDurationSecs)) {
            ALOGE("Media playlist missing #EXT-X-TARGETDURATION");
            return ERROR_MALFORMED;
        }
        mTargetDurationUs = targetDurationSecs * 1000000ll;
        mFirstSeqNumber = 0;
        if (mMeta != NULL) {
            mMeta->findInt32("media-sequence", &mFirstSeqNumber);
        }
        mLastSeqNumber = mFirstSeqNumber + mItems.size() - 1;
    }
    return OK;
}
```
好了我们现在已经获取到了类型为M3UParser的播放列表文件了，这时候会发送一个kWhatPlaylistFetched，这个在哪里被处理呢？当然是LiveSession啊。

``` cpp
[->\frameworks\av\media\libstagefright\httplive\LiveSession.cpp]
case PlaylistFetcher::kWhatPlaylistFetched:
{
    onMasterPlaylistFetched(msg);
    break;
}
```
获取到播放列表后要干啥呢？我们接下来看：

``` cpp
[->\frameworks\av\media\libstagefright\httplive\LiveSession.cpp]
void LiveSession::onMasterPlaylistFetched(const sp<AMessage> &msg) {

    AString uri;
    CHECK(msg->findString("uri", &uri));
    ssize_t index = mFetcherInfos.indexOfKey(uri);
    // no longer useful, remove
    mFetcherLooper->unregisterHandler(mFetcherInfos[index].mFetcher->id());
    mFetcherInfos.removeItemsAt(index);
    //取走获取到的playlist
    CHECK(msg->findObject("playlist", (sp<RefBase> *)&mPlaylist));

    // We trust the content provider to make a reasonable choice of preferred
    // initial bandwidth by listing it first in the variant playlist.
    // At startup we really don't have a good estimate on the available
    // network bandwidth since we haven't tranferred any data yet. Once
    // we have we can make a better informed choice.
    size_t initialBandwidth = 0;
    size_t initialBandwidthIndex = 0;
    int32_t maxWidth = 0;
    int32_t maxHeight = 0;
    //判断获取到的playlist是否有效，无效就没啥用了，我们这里假设有效
    if (mPlaylist->isVariantPlaylist()) {
        Vector<BandwidthItem> itemsWithVideo;
        for (size_t i = 0; i < mPlaylist->size(); ++i) {
            BandwidthItem item;
            item.mPlaylistIndex = i;
            item.mLastFailureUs = -1ll;
            sp<AMessage> meta;
            AString uri;
            mPlaylist->itemAt(i, &uri, &meta);
            //获取带宽
            CHECK(meta->findInt32("bandwidth", (int32_t *)&item.mBandwidth));
            //获取最大分辨率
            int32_t width, height;
            if (meta->findInt32("width", &width)) {
                maxWidth = max(maxWidth, width);
            }
            if (meta->findInt32("height", &height)) {
                maxHeight = max(maxHeight, height);
            }
            mBandwidthItems.push(item);
            if (mPlaylist->hasType(i, "video")) {
                itemsWithVideo.push(item);
            }
        }
        //移除只有声音的信息
        if (!itemsWithVideo.empty()&& itemsWithVideo.size() < mBandwidthItems.size()) {
            mBandwidthItems.clear();
            for (size_t i = 0; i < itemsWithVideo.size(); ++i) {
                mBandwidthItems.push(itemsWithVideo[i]);
            }
        }
        CHECK_GT(mBandwidthItems.size(), 0u);
        initialBandwidth = mBandwidthItems[0].mBandwidth;
        //按照带宽进行排序
        mBandwidthItems.sort(SortByBandwidth);
        for (size_t i = 0; i < mBandwidthItems.size(); ++i) {
            if (mBandwidthItems.itemAt(i).mBandwidth == initialBandwidth) {
                initialBandwidthIndex = i;
                break;
            }
        }
    } else {
       //......
    }
	//获取到最大的分辨率
    mMaxWidth = maxWidth > 0 ? maxWidth : mMaxWidth;
    mMaxHeight = maxHeight > 0 ? maxHeight : mMaxHeight;
    mPlaylist->pickRandomMediaItems();
    changeConfiguration(0ll /* timeUs */, initialBandwidthIndex, false /* pickTrack */);
}


```

``` cpp
[->\frameworks\av\media\libstagefright\httplive\LiveSession.cpp]
void LiveSession::changeConfiguration(int64_t timeUs, ssize_t bandwidthIndex, bool pickTrack) {

    //取消带宽切换
    cancelBandwidthSwitch();
    mReconfigurationInProgress = true;
    //由mOrigBandwidthIndex切换到mCurBandwidthIndex
    if (bandwidthIndex >= 0) {
        //将当前的带宽设置为当前带宽
        mOrigBandwidthIndex = mCurBandwidthIndex;
        mCurBandwidthIndex = bandwidthIndex;
        if (mOrigBandwidthIndex != mCurBandwidthIndex) {
            //开始切换带宽
            ALOGI("#### Starting Bandwidth Switch: %zd => %zd",mOrigBandwidthIndex, mCurBandwidthIndex);
        }
    }
    CHECK_LT(mCurBandwidthIndex, mBandwidthItems.size());
    //获取当前的BandwidthItem
    const BandwidthItem &item = mBandwidthItems.itemAt(mCurBandwidthIndex);
    uint32_t streamMask = 0; // streams that should be fetched by the new fetcher
    uint32_t resumeMask = 0; // streams that should be fetched by the original fetcher
    AString URIs[kMaxStreams];
    for (size_t i = 0; i < kMaxStreams; ++i) {
        if (mPlaylist->getTypeURI(item.mPlaylistIndex, mStreams[i].mType, &URIs[i])) {
            streamMask |= indexToType(i);
        }
    }

    // 停止我们不需要的，暂停我们将要复用的，第一次的时候这里是没有的所以跳过
    for (size_t i = 0; i < mFetcherInfos.size(); ++i) {
        //.........................
    }

    sp<AMessage> msg;
    if (timeUs < 0ll) {
        // skip onChangeConfiguration2 (decoder destruction) if not seeking.
        msg = new AMessage(kWhatChangeConfiguration3, this);
    } else {
        msg = new AMessage(kWhatChangeConfiguration2, this);
    }
    msg->setInt32("streamMask", streamMask);
    msg->setInt32("resumeMask", resumeMask);
    msg->setInt32("pickTrack", pickTrack);
    msg->setInt64("timeUs", timeUs);
    for (size_t i = 0; i < kMaxStreams; ++i) {
        if ((streamMask | resumeMask) & indexToType(i)) {
            msg->setString(mStreams[i].uriKey().c_str(), URIs[i].c_str());
        }
    }

    // Every time a fetcher acknowledges the stopAsync or pauseAsync request
    // we'll decrement mContinuationCounter, once it reaches zero, i.e. all
    // fetchers have completed their asynchronous operation, we'll post
    // mContinuation, which then is handled below in onChangeConfiguration2.
  	//每次fetcher 调用了stopAsync和pauseAsync mContinuationCounter 数值都会减去1,一旦减到0 那么将会在onChangeConfiguration2处理
    mContinuationCounter = mFetcherInfos.size();
    mContinuation = msg;
    if (mContinuationCounter == 0) {
        msg->post();
    }
}

void LiveSession::onChangeConfiguration2(const sp<AMessage> &msg) {


    int64_t timeUs;
    CHECK(msg->findInt64("timeUs", &timeUs));

    if (timeUs >= 0) {
        mLastSeekTimeUs = timeUs;
        mLastDequeuedTimeUs = timeUs;
        for (size_t i = 0; i < mPacketSources.size(); i++) {
            sp<AnotherPacketSource> packetSource = mPacketSources.editValueAt(i);
            sp<MetaData> format = packetSource->getFormat();
            packetSource->clear();
            packetSource->setFormat(format);
        }
        for (size_t i = 0; i < kMaxStreams; ++i) {
            mStreams[i].reset();
        }
        mDiscontinuityOffsetTimesUs.clear();
        mDiscontinuityAbsStartTimesUs.clear();

        if (mSeekReplyID != NULL) {
            CHECK(mSeekReply != NULL);
            mSeekReply->setInt32("err", OK);
            mSeekReply->postReply(mSeekReplyID);
            mSeekReplyID.clear();
            mSeekReply.clear();
        }
        restartPollBuffering();
    }

    uint32_t streamMask, resumeMask;
    CHECK(msg->findInt32("streamMask", (int32_t *)&streamMask));
    CHECK(msg->findInt32("resumeMask", (int32_t *)&resumeMask));

    streamMask |= resumeMask;

    AString URIs[kMaxStreams];
    for (size_t i = 0; i < kMaxStreams; ++i) {
        if (streamMask & indexToType(i)) {
            const AString &uriKey = mStreams[i].uriKey();
            CHECK(msg->findString(uriKey.c_str(), &URIs[i]));
            ALOGV("%s = '%s'", uriKey.c_str(), URIs[i].c_str());
        }
    }

    uint32_t changedMask = 0;
    for (size_t i = 0; i < kMaxStreams && i != kSubtitleIndex; ++i) {
        // stream URI could change even if onChangeConfiguration2 is only
        // used for seek. Seek could happen during a bw switch, in this
        // case bw switch will be cancelled, but the seekTo position will
        // fetch from the new URI.
        if ((mStreamMask & streamMask & indexToType(i))
                && !mStreams[i].mUri.empty()
                && !(URIs[i] == mStreams[i].mUri)) {
            ALOGV("stream %zu changed: oldURI %s, newURI %s", i,
                    mStreams[i].mUri.c_str(), URIs[i].c_str());
            sp<AnotherPacketSource> source = mPacketSources.valueFor(indexToType(i));
            if (source->getLatestDequeuedMeta() != NULL) {
                source->queueDiscontinuity(ATSParser::DISCONTINUITY_FORMATCHANGE, NULL, true);
            }
        }
        // Determine which decoders to shutdown on the player side,
        // a decoder has to be shutdown if its streamtype was active
        // before but now longer isn't.
        if ((mStreamMask & ~streamMask & indexToType(i))) {
            changedMask |= indexToType(i);
        }
    }

	//这里会触发kWhatStreamsChanged
    sp<AMessage> notify = mNotify->dup();
    notify->setInt32("what", kWhatStreamsChanged);
    notify->setInt32("changedMask", changedMask);
	//将kWhatChangeConfiguration3作为回复消息
    msg->setWhat(kWhatChangeConfiguration3);
    msg->setTarget(this);
    notify->setMessage("reply", msg);
    notify->post();
}
```

``` cpp
[->\frameworks\av\media\libstagefright\httplive\LiveSession.cpp]
void LiveSession::onChangeConfiguration3(const sp<AMessage> &msg) {
    mContinuation.clear();
    
    uint32_t streamMask, resumeMask;
    CHECK(msg->findInt32("streamMask", (int32_t *)&streamMask));
    CHECK(msg->findInt32("resumeMask", (int32_t *)&resumeMask));

    mNewStreamMask = streamMask | resumeMask;

    int64_t timeUs;
    int32_t pickTrack;
    bool switching = false;
    CHECK(msg->findInt64("timeUs", &timeUs));
    CHECK(msg->findInt32("pickTrack", &pickTrack));

    ......
    for (size_t i = 0; i < kMaxStreams; i++) {
        ......
        fetcher->startAsync(
                sources[kAudioIndex],
                sources[kVideoIndex],
                sources[kSubtitleIndex],
                getMetadataSource(sources, mNewStreamMask, switching),
                startTime.mTimeUs < 0 ? mLastSeekTimeUs : startTime.mTimeUs,
                startTime.getSegmentTimeUs(),
                startTime.mSeq,
                seekMode);
    }

    ......
}
```

```cpp
[->\frameworks\av\media\libstagefright\httplive\PlaylistFetcher.cpp]
void PlaylistFetcher::startAsync(
        const sp<AnotherPacketSource> &audioSource,
        const sp<AnotherPacketSource> &videoSource,
        const sp<AnotherPacketSource> &subtitleSource,
        const sp<AnotherPacketSource> &metadataSource,
        int64_t startTimeUs,
        int64_t segmentStartTimeUs,
        int32_t startDiscontinuitySeq,
        LiveSession::SeekMode seekMode) {
    sp<AMessage> msg = new AMessage(kWhatStart, this);
    ......
    msg->setInt32("streamTypeMask", streamTypeMask);
    msg->setInt64("startTimeUs", startTimeUs);
    msg->setInt64("segmentStartTimeUs", segmentStartTimeUs);
    msg->setInt32("startDiscontinuitySeq", startDiscontinuitySeq);
    msg->setInt32("seekMode", seekMode);
    msg->post();
}
status_t PlaylistFetcher::onStart(const sp<AMessage> &msg) {
......

    postMonitorQueue();

    return OK;
}
void PlaylistFetcher::postMonitorQueue(int64_t delayUs, int64_t minDelayUs) {
    int64_t maxDelayUs = delayUsToRefreshPlaylist();
    sp<AMessage> msg = new AMessage(kWhatMonitorQueue, this);
    msg->setInt32("generation", mMonitorQueueGeneration);
    msg->post(delayUs);
}

        case kWhatDownloadNext:
        {
            int32_t generation;
            CHECK(msg->findInt32("generation", &generation));

            if (generation != mMonitorQueueGeneration) {
                // Stale event
                break;
            }

            if (msg->what() == kWhatMonitorQueue) {
                onMonitorQueue();
            } else {
                onDownloadNext();
            }
            break;
        }

void PlaylistFetcher::onDownloadNext() {
    AString uri;
    sp<AMessage> itemMeta;
    sp<ABuffer> buffer;
    sp<ABuffer> tsBuffer;
    int32_t firstSeqNumberInPlaylist = 0;
    int32_t lastSeqNumberInPlaylist = 0;
    bool connectHTTP = true;
    // block-wise download
    bool shouldPause = false;
    ssize_t bytesRead;
    do {
        int64_t startUs = ALooper::GetNowUs();
        bytesRead = mHTTPDownloader->fetchBlock(
                uri.c_str(), &buffer, range_offset, range_length, kDownloadBlockSize,
                NULL /* actualURL */, connectHTTP);
        int64_t delayUs = ALooper::GetNowUs() - startUs;

        ......
        }

        ......
        if (bufferStartsWithTsSyncByte(buffer)) {
            // Incremental extraction is only supported for MPEG2 transport streams.
            if (tsBuffer == NULL) {
                tsBuffer = new ABuffer(buffer->data(), buffer->capacity());
                tsBuffer->setRange(0, 0);
            } else if (tsBuffer->capacity() != buffer->capacity()) {
                size_t tsOff = tsBuffer->offset(), tsSize = tsBuffer->size();
                tsBuffer = new ABuffer(buffer->data(), buffer->capacity());
                tsBuffer->setRange(tsOff, tsSize);
            }
            tsBuffer->setRange(tsBuffer->offset(), tsBuffer->size() + bytesRead);
            err = extractAndQueueAccessUnitsFromTs(tsBuffer);
        }

        ......
    } while (bytesRead != 0);
    ......
    // bulk extract non-ts files
    bool startUp = mStartup;
    if (tsBuffer == NULL) {
        status_t err = extractAndQueueAccessUnits(buffer, itemMeta);
        if (err == -EAGAIN) {
            // starting sequence number too low/high
            postMonitorQueue();
            return;
        } else if (err == ERROR_OUT_OF_RANGE) {
            // reached stopping point
            notifyStopReached();
            return;
        } else if (err != OK) {
            notifyError(err);
            return;
        }
    }
    ......
}


status_t PlaylistFetcher::extractAndQueueAccessUnits(
        const sp<ABuffer> &buffer, const sp<AMessage> &itemMeta) {
    if (bufferStartsWithWebVTTMagicSequence(buffer)) {
        if (mStreamTypeMask != LiveSession::STREAMTYPE_SUBTITLES) {
            ALOGE("This stream only contains subtitles.");
            return ERROR_MALFORMED;
        }

        const sp<AnotherPacketSource> packetSource =
            mPacketSources.valueFor(LiveSession::STREAMTYPE_SUBTITLES);

        int64_t durationUs;
        CHECK(itemMeta->findInt64("durationUs", &durationUs));
        buffer->meta()->setInt64("timeUs", getSegmentStartTimeUs(mSeqNumber));
        buffer->meta()->setInt64("durationUs", durationUs);
        buffer->meta()->setInt64("segmentStartTimeUs", getSegmentStartTimeUs(mSeqNumber));
        buffer->meta()->setInt32("discontinuitySeq", mDiscontinuitySeq);
        buffer->meta()->setInt32("subtitleGeneration", mSubtitleGeneration);
        packetSource->queueAccessUnit(buffer);
        return OK;
    }

    ......

    size_t offset = 0;
    while (offset < buffer->size()) {
        const uint8_t *adtsHeader = buffer->data() + offset;
        CHECK_LT(offset + 5, buffer->size());

        ......
        sp<ABuffer> unit = new ABuffer(aac_frame_length);
        memcpy(unit->data(), adtsHeader, aac_frame_length);

        unit->meta()->setInt64("timeUs", unitTimeUs);
        setAccessUnitProperties(unit, packetSource);
        packetSource->queueAccessUnit(unit);
    }

    return OK;
}


```

``` cpp
[->\frameworks\av\media\libstagefright\mpeg2ts\AnotherPacketSource.cpp]
void AnotherPacketSource::queueAccessUnit(const sp<ABuffer> &buffer) {
    int32_t damaged;
    ......
    Mutex::Autolock autoLock(mLock);
    mBuffers.push_back(buffer);
    mCondition.signal();

    int32_t discontinuity;
    if (buffer->meta()->findInt32("discontinuity", &discontinuity)){
        ALOGV("queueing a discontinuity with queueAccessUnit");

        mLastQueuedTimeUs = 0ll;
        mEOSResult = OK;
        mLatestEnqueuedMeta = NULL;

        mDiscontinuitySegments.push_back(DiscontinuitySegment());
        return;
    }

    ......
}
```

##### 2.2、获取数据进行解码
关于解码初始化NuPlayer::instantiateDecoder过程请参考：[Android Video System（2）：音视频分离MediaExtractor、解码Decoder、渲染Renderer源码分析]()
我们直接看如何获取数据进行解码:

``` cpp
[->\frameworks\av\media\libstagefright\ACodec.cpp]
void ACodec::signalResume() {
    (new AMessage(kWhatResume, this))->post();
}
case kWhatResume:
{
    resume();
    handled = true;
    break;
}
void ACodec::ExecutingState::resume() {

    submitOutputBuffers();
    // Post all available input buffers
    if (mCodec->mBuffers[kPortIndexInput].size() == 0u) {
        ALOGW("[%s] we don't have any input buffers to resume", mCodec->mComponentName.c_str());
    }

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

``` cpp
[->\frameworks\av\media\libstagefright\ACodec.cpp]

case CodecBase::kWhatFillThisBuffer:
{
   
   //..........
    if (mFlags & kFlagIsAsync) {
        if (!mHaveInputSurface) {
            if (mState == FLUSHED) {
                mHavePendingInputBuffers = true;
            } else {
                onInputBufferAvailable();
            }
        }
    } else if (mFlags & kFlagDequeueInputPending) {
        CHECK(handleDequeueInputBuffer(mDequeueInputReplyID));
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

这个mCallback怎么来的？

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayerDecoder.cpp]
void NuPlayer::Decoder::onConfigure(const sp<AMessage> &format) {

	//.................
    sp<AMessage> reply = new AMessage(kWhatCodecNotify, this);
    mCodec->setCallback(reply);
	//..................
}

[->\frameworks\av\media\libstagefright\MediaCodec.cpp]
status_t MediaCodec::setCallback(const sp<AMessage> &callback) {
    sp<AMessage> msg = new AMessage(kWhatSetCallback, this);
    msg->setMessage("callback", callback);

    sp<AMessage> response;
    return PostAndAwaitResponse(msg, &response);
}
case kWhatSetCallback:
{
    sp<AReplyToken> replyID;
    CHECK(msg->senderAwaitsResponse(&replyID));
    sp<AMessage> callback;
    CHECK(msg->findMessage("callback", &callback));

    mCallback = callback;

    if (mCallback != NULL) {
        mFlags |= kFlagIsAsync;
    } else {
        mFlags &= ~kFlagIsAsync;
    }

    sp<AMessage> response = new AMessage;
    response->postReply(replyID);
    break;
}

```
所以根据上面我们可以知道接下来i调用的是kWhatCodecNotify 下的 CB_INPUT_AVAILABLE

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
    if (isDiscontinuityPending()) {
        return false;
    }

    sp<ABuffer> buffer;
    mCodec->getInputBuffer(index, &buffer);

    if (buffer == NULL) {
        handleError(UNKNOWN_ERROR);
        return false;
    }

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

    //CHECK_LT(bufferIx, mInputBuffers.size());

    if (mMediaBuffers[index] != NULL) {
        mMediaBuffers[index]->release();
        mMediaBuffers.editItemAt(index) = NULL;
    }
    mInputBufferIsDequeued.editItemAt(index) = true;

    if (!mCSDsToSubmit.isEmpty()) {
        sp<AMessage> msg = new AMessage();
        msg->setSize("buffer-ix", index);

        sp<ABuffer> buffer = mCSDsToSubmit.itemAt(0);
        ALOGI("[%s] resubmitting CSD", mComponentName.c_str());
        msg->setBuffer("buffer", buffer);
        mCSDsToSubmit.removeAt(0);
        CHECK(onInputBufferFetched(msg));
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

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayerDecoderBase.cpp]
void NuPlayer::DecoderBase::onRequestInputBuffers() {
    if (mRequestInputBuffersPending) {
        return;
    }

    // doRequestBuffers() return true if we should request more data
    if (doRequestBuffers()) {
        mRequestInputBuffersPending = true;

        sp<AMessage> msg = new AMessage(kWhatRequestInputBuffers, this);
        msg->post(10 * 1000ll);
    }
}

[->\frameworks\av\media\libmediaplayerservice\nuplayer\NuPlayerDecoder.cpp]
bool NuPlayer::Decoder::doRequestBuffers() {
    // mRenderer is only NULL if we have a legacy widevine source that
    // is not yet ready. In this case we must not fetch input.
    if (isDiscontinuityPending() || mRenderer == NULL) {
        return false;
    }
    status_t err = OK;
    while (err == OK && !mDequeuedInputBuffers.empty()) {
        size_t bufferIx = *mDequeuedInputBuffers.begin();
        sp<AMessage> msg = new AMessage();
        msg->setSize("buffer-ix", bufferIx);
        err = fetchInputData(msg);
        if (err != OK && err != ERROR_END_OF_STREAM) {
            // if EOS, need to queue EOS buffer
            break;
        }
        mDequeuedInputBuffers.erase(mDequeuedInputBuffers.begin());

        if (!mPendingInputMessages.empty()
                || !onInputBufferFetched(msg)) {
            mPendingInputMessages.push_back(msg);
        }
    }

    return err == -EWOULDBLOCK
            && mSource->feedMoreTSData() == OK;
}
status_t NuPlayer::Decoder::fetchInputData(sp<AMessage> &reply) {
    sp<ABuffer> accessUnit;
    bool dropAccessUnit;
    do {
        status_t err = mSource->dequeueAccessUnit(mIsAudio, &accessUnit);

        if (err == -EWOULDBLOCK) {
            return err;
        } else if (err != OK) {
            if (err == INFO_DISCONTINUITY) {
                int32_t type;
                CHECK(accessUnit->meta()->findInt32("discontinuity", &type));

                bool formatChange =
                    (mIsAudio &&
                     (type & ATSParser::DISCONTINUITY_AUDIO_FORMAT))
                    || (!mIsAudio &&
                            (type & ATSParser::DISCONTINUITY_VIDEO_FORMAT));

                bool timeChange = (type & ATSParser::DISCONTINUITY_TIME) != 0;

                ALOGI("%s discontinuity (format=%d, time=%d)",
                        mIsAudio ? "audio" : "video", formatChange, timeChange);

                bool seamlessFormatChange = false;
                sp<AMessage> newFormat = mSource->getFormat(mIsAudio);
                if (formatChange) {
                    seamlessFormatChange =
                        supportsSeamlessFormatChange(newFormat);
                    // treat seamless format change separately
                    formatChange = !seamlessFormatChange;
                }

                // For format or time change, return EOS to queue EOS input,
                // then wait for EOS on output.
                if (formatChange /* not seamless */) {
                    mFormatChangePending = true;
                    err = ERROR_END_OF_STREAM;
                } else if (timeChange) {
                    rememberCodecSpecificData(newFormat);
                    mTimeChangePending = true;
                    err = ERROR_END_OF_STREAM;
                } else if (seamlessFormatChange) {
                    // reuse existing decoder and don't flush
                    rememberCodecSpecificData(newFormat);
                    continue;
                } else {
                    // This stream is unaffected by the discontinuity
                    return -EWOULDBLOCK;
                }
            }

            // reply should only be returned without a buffer set
            // when there is an error (including EOS)
            CHECK(err != OK);

            reply->setInt32("err", err);
            return ERROR_END_OF_STREAM;
        }

        dropAccessUnit = false;
        if (!mIsAudio
                && !mIsSecure
                && mRenderer->getVideoLateByUs() > 100000ll
                && mIsVideoAVC
                && !IsAVCReferenceFrame(accessUnit)) {
            dropAccessUnit = true;
            ++mNumInputFramesDropped;
        }
    } while (dropAccessUnit);

    // ALOGV("returned a valid buffer of %s data", mIsAudio ? "mIsAudio" : "video");
#if 0
    int64_t mediaTimeUs;
    CHECK(accessUnit->meta()->findInt64("timeUs", &mediaTimeUs));
    ALOGV("[%s] feeding input buffer at media time %.3f",
         mIsAudio ? "audio" : "video",
         mediaTimeUs / 1E6);
#endif

    if (mCCDecoder != NULL) {
        mCCDecoder->decode(accessUnit);
    }

    reply->setBuffer("buffer", accessUnit);

    return OK;
}
```


##### 2.3、循环获取数据HTTPLiveSource::dequeueAccessUnit()

``` cpp
[->\frameworks\av\media\libstagefright\httplive\LiveSession.cpp]
status_t LiveSession::dequeueAccessUnit(
        StreamType stream, sp<ABuffer> *accessUnit) {
    status_t finalResult = OK;
    sp<AnotherPacketSource> packetSource = mPacketSources.valueFor(stream);
    ......

    status_t err = packetSource->dequeueAccessUnit(accessUnit);
    ......
}
```

``` cpp
[->\frameworks\av\media\libstagefright\mpeg2ts\AnotherPacketSource.cpp]
status_t AnotherPacketSource::dequeueAccessUnit(sp<ABuffer> *buffer) {
    buffer->clear();

    Mutex::Autolock autoLock(mLock);
    while (mEOSResult == OK && mBuffers.empty()) {
        mCondition.wait(mLock);
    }

    if (!mBuffers.empty()) {
        *buffer = *mBuffers.begin();
        mBuffers.erase(mBuffers.begin());

        int32_t discontinuity;
        if ((*buffer)->meta()->findInt32("discontinuity", &discontinuity)) {
            if (wasFormatChange(discontinuity)) {
                mFormat.clear();
            }

            mDiscontinuitySegments.erase(mDiscontinuitySegments.begin());
            // CHECK(!mDiscontinuitySegments.empty());
            return INFO_DISCONTINUITY;
        }

        // CHECK(!mDiscontinuitySegments.empty());
        DiscontinuitySegment &seg = *mDiscontinuitySegments.begin();

        int64_t timeUs;
        mLatestDequeuedMeta = (*buffer)->meta()->dup();
        CHECK(mLatestDequeuedMeta->findInt64("timeUs", &timeUs));
        if (timeUs > seg.mMaxDequeTimeUs) {
            seg.mMaxDequeTimeUs = timeUs;
        }

        sp<RefBase> object;
        if ((*buffer)->meta()->findObject("format", &object)) {
            setFormat(static_cast<MetaData*>(object.get()));
        }

        return OK;
    }

    return mEOSResult;
}

```

##### 2.4、解码后音视频播放过程
关于解码后播放过程请参考：[Android Video System（5）：Android Multimedia - NuPlayer音视频同步实现分析]()

#### （三）、基于NuPlayer的RTSP流媒体协议
##### 3.1.1、RTSP 概述：
RTSP 是Real Time Streaming Protocol（实时流媒体协议）的简称。RTSP提供一种可扩展的框架，使得能够提供可控制的，按需传输实时数据，比如音频和视频文件。RTSP对流媒体提供了诸如暂停，快进等控制，而它本身并不传输数据，RTSP作用相当于流媒体服务器的远程控制。传输数据可以通过传输层的TCP，UDP协议，RTSP也提供了基于 RTP传输机制的一些有效的方法。

##### 3.1.2、RTSP 模型：
客户机在向视频服务器请求视频服务之前，首先通过HTTP协议从WEB服务器获取所请求视频服务的演示描述（Presentation description）文件，在RTSP中，每个演示（Presentation）及其所对应的媒体流都由一个RTSP URL标识。整个演示及媒体特性都在一个演示描述（Presentation description）文件中定义，该文件可能包括媒体编码方式、语言、RTSPURLs、目标地址、端口及其它参数。用户在向服务器请求某个连续媒体流的服务之前，必须首先从服务器获得该媒体流的演示描述（Presentation description ）文件以得到必需的参数。利用该文件提供的信息定位视频服务地址（包括视频服务器地址和端口号）及视频服务的编码方式等信息。
客户机根据上述信息向视频服务器请求视频服务。视频服务初始化完毕，视频服务器为该客户建立一个新的视频服务流，客户端与服务器运行实时流控制协议RTSP，以对该流进行各种VCR 控制信号的交换，如播放、暂停、快进、快退等。当服务完毕，客户端提出拆线（TEARDOWN）请求。服务器使用 RTP协议将媒体数据传输给客户端，一旦数据抵达客户端，客户端应用程序即可播放输出。在流式传输中，使用RTP/RTCP和RTSP /TCP两种不同的通信协议在客户端和服务器间建立联系。如下图：

##### 3.1.3、RTSP 协议消息格式：
请求消息格式:

``` cpp
方法   URI  RTSP版本   CR  LF 
消息头 CR   LF   CR  LF 
消息体 CR   LF
```

其中方法包括OPTION回应中所有的命令,URI是接受方的地址,例如
rtsp://192.168.20.136

RTSP版本一般都是 RTSP/1.0.每行后面的CR LF表示回车换行，需要接受端有相应的解析，最后一个消息头需要有两个CR LF

回应消息格式:

``` cpp
RTSP版本  状态码  解释 CR  LF 
消息头 CR  LF  CR  LF 
消息体 CR  LF
```

其中RTSP版本一般都是RTSP/1.0,状态码是一个数值,200表示成功,解释是与状态码对应的文本解释。
##### 3.1.4、简单的RTSP 交互过程:
下面以一次流媒体播放为例介绍整个播放过程的RTSP状态转换的流程：
其中C表示RTSP客户端,S表示RTSP服务端：

``` cpp
C->S:OPTION     request        //询问服务端有哪些方法可用
S->C:OPTION     response       //服务端回应信息中包括提供的所有可用方法 

C->S:DESCRIBE    request        //要求得到服务端提供的媒体初始化描述信息 
S->C:DESCRIBE    response       //服务端回应媒体初始化描述信息，主要是SDP

C->S:SETUP       request        //设置会话的属性，以及传输模式提醒服务端建立会话 
S->C:SETUP       response       //服务端建立会话，返回会话标识符，和会话相关信息 

C->S:PLAY        request         //客户端请求播放 
S->C:PLAY        response       //服务器回应该请求的信息 

S->C:                           //发送流媒体数据 

C->S:TEARDOWN    request        //客户端请求关闭会话 
S->C:TEARDOWN    response //服务端回应该请求
```
其中第SETUP和PLAY这两部是必需的，
OPTION 步骤只要服务器客户端约定好，有哪些方法可用，则option请求可以不要。
如果我们有其他途径得到媒体初始化描述信息，则我们也不需要通过RTSP中的DESCRIPTION请求来完成。
TEARDOWN，可以根据系统需求的设计来决定是否需要。

![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/video.system/VS-06-07-RTSP-TEARDOWN.png)

##### 3.1.5、RTSP的主要命令表：

![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/video.system/VS-06-08-RTSP-option.png)


![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/video.system/VS-06-09-RTSP-describe.png)


![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/video.system/VS-06-10-RTSP-setup.png)


![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/video.system/VS-06-11-RTSP-play.png)


![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/video.system/VS-06-12-RTSP-teardowm.png)


RTSP状态码：

``` cpp
Status-Code =
| "100" ; 				Continue
| "200" ; 				OK
| "201" ; 				Created
| "250" ; 				Low on Storage Space
| "300" ; 				Multiple Choices
| "301" ;				Moved Permanently
| "302" ; 				Moved Temporarily
| "303" ; 				See Other
| "304" ; 				Not Modified
| "305" ; 				Use Proxy
| "400" ; 				Bad Request
| "401" ; 				Unauthorized
| "402" ; 				Payment Required
| "403" ; 				Forbidden
| "404" ; 				Not Found
| "405" ; 				Method Not Allowed
| "406" ; 				Not Acceptable
| "407" ; 				Proxy Authentication Required
| "408" ; 				Request Time-out
| "410" ; 				Gone
| "411" ; 				Length Required
| "412" ; 				Precondition Failed
| "413" ; 				Request Entity Too Large
| "414" ; 				Request-URI Too Large
| "415" ; 				Unsupported Media Type
| "451" ; 				Parameter Not Understood
| "452" ; 				Conference Not Found
| "453" ; 				Not Enough Bandwidth
| "454" ; 				Session Not Found
| "455" ; 				Method Not Valid in This State
| "456" ; 				Header Field Not Valid for Resource
| "457" ; 				Invalid Range
| "458" ; 				Parameter Is Read-Only
| "459" ;				Aggregate operation not allowed
| "460" ; 				Only aggregate operation allowed
| "461" ; 				Unsupported transport
| "462" ; 				Destination unreachable
| "500" ; 				Internal Server Error
| "501" ; 				Not Implemented
| "502" ; 				Bad Gateway
| "503" ; 				Service Unavailable
| "504" ; 				Gateway Time-out
| "505" ; 				RTSP Version not supported
| "551" ; 				Option not supported
```
##### 3.1.6、SDP的格式：

``` cpp
v=<version>                            (协议版本)
o=<username> <session id> <version> <network type> <address type> <address>                              （所有者/创建者和会话标识符）
s=<session name>                       （会话名称）
i=<session description>                （会话信息） 
u=<URI>                                （URI 描述）
e=<email address>                      （Email 地址）
p=<phone number>                       （电话号码）
c=<network type> <address type> <connection address>   （连接信息）
b=<modifier>:<bandwidth-value>         （带宽信息）
t=<start time> <stop time>             （会话活动时间）
r=<repeat interval> <active duration> <list of offsets from start-time> 
                                        （0或多次重复次数）
z=<adjustment time> <offset> <adjustment time> <offset>（时间区域调整）
k=<method>:<encryption key>             （加密密钥）
a=<attribute>:<value>                   （0 个或多个会话属性行）
m=<media> <port> <transport> <fmt list> （媒体名称和传输地址）

时间描述： 
t = （会话活动时间） 
r = * （0或多次重复次数） 
媒体描述： 
m = （媒体名称和传输地址） 
i = * （媒体标题） 
c = * （连接信息 — 如果包含在会话层则该字段可选） 
b = * （带宽信息） 
k = * （加密密钥） 
a = * （0 个或多个媒体属性行）
```
##### 3.1.7、RTP协议：
实时传输协议（Real-time Transport Protocol，RTP）是用来在单播或者多播的情境中传流媒体数据的数据传输协议。通常使用UDP来进行多媒体数据的传输，也不排除使用TCP或者ATM等其它协议作为它的载体，整个RTP 协议由两个密切相关的部分组成：RTP数据协议和RTP控制协议（也就是RTCP协议）。
RTP为Internet上端到端的实时传输提供时间信息和流同步，但它并不保证服务质量，服务质量由RTCP来提供。

- 使用RTP协议进行数据传输的一个简要RTP的会话过程：
当应用程序建立一个RTP会话时，应用程序将确定一对目的传输地址。目的传输地址由一个网络地址和一对端口组成，有两个端口：一个给RTP包，一个给RTCP包，也就是说RTP和RTCP数据包是分开传输的，这样可以使得RTP/RTCP数据能够正确发送。其中RTP数据发向偶数的UDP端口，而对应的控制信号RTCP数据发向相邻的奇数UDP端口，这样就构成一个UDP端口对。
当发送数据的时候RTP协议从上层接收流媒体信息码流，封装成RTP数据包；RTCP从上层接收控制信息，封装成RTCP控制包。RTP将RTP 数据包发往UDP端口对中偶数端口；RTCP将RTCP控制包发往UDP端口对中的接收端口。
如果在一次会议中同时使用了音频和视频会议，这两种媒体将分别在不同的RTP会话中传送，每一个会话使用不同的传输地址（IP地址＋端口）。如果一个用户同时使用了两个会话，则每个会话对应的RTCP包都使用规范化名字CNAME（Canonical Name）。与会者可以根据RTCP包中的CNAME来获取相关联的音频和视频，然后根据RTCP包中的计时信息(Network time protocol)来实现音频和视频的同步。

- 翻译器和混合器
在RTP协议中还引入了翻译器和混合器。翻译器和混合器都是RTP级的中继系统。
混合器的使用情景：
在Internet上举行视频会议时，可能有少数参加者通过低速链路与使用高速网络的多数参加者相连接。为了不强制所有会议参加者都使用低带宽和低质量的数据编码，RTP允许在低带宽区域附近使用混合器作为RTP级中继器。混合器从一个或多个信源接收RTP报文，对到达的数据报文进行重新同步和重新组合，这些重组的数据流被混合成一个数据流，将数据编码转化为在低带宽上可用的类型，并通过低速链路向低带宽区域转发。为了对多个输入信源进行统一的同步，混合器在多个媒体流之间进行定时调整，产生它自己的定时同步，因此所有从混合器输出的报文都把混合器作为同步信源。为了保证接收者能够正确识别混合器处理前的原始报文发送者，混合器在RTP报头中设置了CSRC标识符队列，以标识那些产生混和报文的原始同步信源。
翻译器的使用情景
在Internet环境中，一些会议的参加者可能被隔离在应用级防火墙的外面，这些参加者被禁止直接使用IP组播地址进行访问，虽然他们可能是通过高速链路连接的。在这些情况下，RTP允许使用转换器作为RTP级中继器。在防火墙两端分别安装一个转换器，防火墙之外的转换器过滤所有接收到的组播报文，并通过一条安全的连接传送给防火墙之内的转换器，内部转换器将这些组播报文再转发送给内部网络中的组播组成员
##### 3.1.8、RTP协议报头格式

![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/video.system/VS-06-13-RTSP-baotou1.png)

![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/video.system/VS-06-14-RTSP-baotou2.png)

##### 3.1.9、RTCP协议报头格式

如前面所述RTCP的主要功能是：服务质量的监视与反馈、媒体间的同步，以及多播组中成员的标识。在RTP会话期间，各参与者周期性地传送RTCP包。RTCP包中含有已发送的数据包的数量、丢失的数据包的数量等统计资料，因此，各参与者可以利用这些信息动态地改变传输速率，甚至改变有效载荷类型。RTP和RTCP配合使用，它们能以有效的反馈和最小的开销使传输效率最佳化，因而特别适合传送网上的实时数据。RTCP也是用UDP来传送的，但RTCP封装的仅仅是一些控制信息，因而分组很短，所以可以将多个RTCP分组封装在一个UDP包中。
RTCP有如下五种分组类型：

![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/video.system/VS-06-15-RTCP-baotou1.png)


下面是SR分组的格式：

![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/video.system/VS-06-16-RTCP-baotou2.png)


##### 3.2、基于NuPLayer的RTSP 代码流程
setDataSource 阶段的任务这里就不重复介绍了，它主要完成播放引擎的建立以及根据URL格式创建对应的Source，比如这里将要提到的RTSPSource，然后赋值给mSource。

我们直接来看prepare阶段：

先上图再看代码，结合图看会比较清晰
![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/video.system/VS-06-17-ARTSPConnection.png)
![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/video.system/VS-06-18-ARTSP-Source.png)
![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/video.system/VS-06-19-ARTSP-Source-state.png)
![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/video.system/VS-06-20-ARTSP-Source-state2.png)

##### 3.3、RTSPSource::prepareAsync()流程
在prepare阶段我们首先会判断是否是SDP，mIsSDP这个变量是在初始化RTSPSource时候传入的，我们这里先分析mIsSDP = false的情况。这种情况下首先创建一个MyHandler，并调用connect，与服务器建立连接。

``` cpp
[->\frameworks\av\media\libmediaplayerservice\nuplayer\RTSPSource.cpp]
void NuPlayer::RTSPSource::prepareAsync() {

    //..........
    sp<AMessage> notify = new AMessage(kWhatNotify, this);
    
    //检查当前状态是否为DISCONNECTED
    CHECK_EQ(mState, (int)DISCONNECTED);
    //设置当前状态为CONNECTING
    mState = CONNECTING;

    if (mIsSDP) {
        //如果是SDP那么就需要创建一个SDPLoader 从服务器上加载一个描述文件
        mSDPLoader = new SDPLoader(notify, (mFlags & kFlagIncognito) ? SDPLoader::kFlagIncognito : 0, mHTTPService);
        mSDPLoader->load(mURL.c_str(), mExtraHeaders.isEmpty() ? NULL : &mExtraHeaders);
    } else {
        //如果不是SDP 那么就使用MyHandler 来进行连接
        mHandler = new MyHandler(mURL.c_str(), notify, mUIDValid, mUID);
        mLooper->registerHandler(mHandler);
        mHandler->connect();
    }
    //启动缓存
    startBufferingIfNecessary();
}
```
在介绍connect方法之前需要先了解mConn以及mRTPConn这两个成员变量，mConn是一个ARTSPConnection，它主要与服务器相连，发送和接收请求数据，mRTPConn是一个ARTPConnection 用于发送和接收媒体数据。
在connect方法中会使用mConn向服务器发起连接请求。

``` cpp
[->\frameworks\av\media\libstagefright\rtsp\MyHandler.h]
void connect() {
    //mConn(new ARTSPConnection(mUIDValid, mUID)),
    looper()->registerHandler(mConn);
    //mRTPConn(new ARTPConnection),
    (1 ? mNetLooper : looper())->registerHandler(mRTPConn);
    sp<AMessage> notify = new AMessage('biny', this);
    mConn->observeBinaryData(notify);
    //连接服务
    sp<AMessage> reply = new AMessage('conn', this);
    mConn->connect(mOriginalSessionURL.c_str(), reply);
}
```

``` cpp
[->\frameworks\av\media\libstagefright\rtsp\ARTSPConnection.cpp]
void ARTSPConnection::connect(const char *url, const sp<AMessage> &reply) {
    sp<AMessage> msg = new AMessage(kWhatConnect, this);
    msg->setString("url", url);
    msg->setMessage("reply", reply);
    msg->post();
}
case kWhatConnect:
    onConnect(msg);
    break;
```
在ARTSPConnection::onConnect中将会从传递过来的URl中解析host，port，path，mUser，mPass，并调用::connect 和服务器取得联系，最后调用postReceiveReponseEvent将请求的回复响应暂存起来。

``` cpp
[->\frameworks\av\media\libstagefright\rtsp\ARTSPConnection.cpp]
void ARTSPConnection::onConnect(const sp<AMessage> &msg) {
    ++mConnectionID;
    
    if (mState != DISCONNECTED) {
        if (mUIDValid) {
            HTTPBase::UnRegisterSocketUserTag(mSocket);
            HTTPBase::UnRegisterSocketUserMark(mSocket);
        }
        close(mSocket);
        mSocket = -1;
        flushPendingRequests();
    }

    mState = CONNECTING;
    AString url;
    //从消息中取下Url
    CHECK(msg->findString("url", &url));
    sp<AMessage> reply;
    //从消息中取下replay
    CHECK(msg->findMessage("reply", &reply));

    AString host, path;
    unsigned port;
    //从URl中解析host，port，path，mUser，mPass
    if (!ParseURL(url.c_str(), &host, &port, &path, &mUser, &mPass)
            || (mUser.size() > 0 && mPass.size() == 0)) {

        //有用户名，但是没有密码，返回错误信息
        // If we have a user name but no password we have to give up
        // right here, since we currently have no way of asking the user
        // for this information.
        ALOGE("Malformed rtsp url %s", uriDebugString(url).c_str());
        reply->setInt32("result", ERROR_MALFORMED);
        reply->post();
        mState = DISCONNECTED;
        return;
    }

    if (mUser.size() > 0) {
        ALOGV("user = '%s', pass = '%s'", mUser.c_str(), mPass.c_str());
    }

    struct hostent *ent = gethostbyname(host.c_str());
    if (ent == NULL) {
        ALOGE("Unknown host %s", host.c_str());
        reply->setInt32("result", -ENOENT);
        reply->post();
        mState = DISCONNECTED;
        return;
    }

    mSocket = socket(AF_INET, SOCK_STREAM, 0);

    if (mUIDValid) {
        HTTPBase::RegisterSocketUserTag(mSocket, mUID,(uint32_t)*(uint32_t*) "RTSP");
        HTTPBase::RegisterSocketUserMark(mSocket, mUID);
    }

    MakeSocketBlocking(mSocket, false);

    struct sockaddr_in remote;
    memset(remote.sin_zero, 0, sizeof(remote.sin_zero));
    remote.sin_family = AF_INET;
    remote.sin_addr.s_addr = *(in_addr_t *)ent->h_addr;
    remote.sin_port = htons(port);
    //连接到服务器
    int err = ::connect(mSocket, (const struct sockaddr *)&remote, sizeof(remote));

    //返回服务器ip
    reply->setInt32("server-ip", ntohl(remote.sin_addr.s_addr));

    if (err < 0) {
        if (errno == EINPROGRESS) {
            sp<AMessage> msg = new AMessage(kWhatCompleteConnection, this);
            msg->setMessage("reply", reply);
            msg->setInt32("connection-id", mConnectionID);
            msg->post();
            return;
        }

        reply->setInt32("result", -errno);
        mState = DISCONNECTED;

        if (mUIDValid) {
            HTTPBase::UnRegisterSocketUserTag(mSocket);
            HTTPBase::UnRegisterSocketUserMark(mSocket);
        }
        close(mSocket);
        mSocket = -1;
    } else {
        //成功的花返回result为OK
        reply->setInt32("result", OK);
        //设置状态为CONNECTED
        mState = CONNECTED;
        mNextCSeq = 1;
        //发送等待返回消息
        postReceiveReponseEvent();
    }
    //‘conn’
    reply->post();
}
```

我们接下来看下postReceiveReponseEvent,调用receiveRTSPReponse获得服务器的回复

``` cpp
[->]
void ARTSPConnection::postReceiveReponseEvent() {
    if (mReceiveResponseEventPending) {
        return;
    }
    sp<AMessage> msg = new AMessage(kWhatReceiveResponse, this);
    msg->post();
    mReceiveResponseEventPending = true;
}
void ARTSPConnection::onReceiveResponse() {
    mReceiveResponseEventPending = false;
    if (mState != CONNECTED) {
        return;
    }
    struct timeval tv;
    tv.tv_sec = 0;
    tv.tv_usec = kSelectTimeoutUs;
    fd_set rs;
    FD_ZERO(&rs);
    FD_SET(mSocket, &rs);

    //选择一个返回的连接
    int res = select(mSocket + 1, &rs, NULL, NULL, &tv);

    if (res == 1) {
        MakeSocketBlocking(mSocket, true);
        bool success = receiveRTSPReponse();
        MakeSocketBlocking(mSocket, false);
        if (!success) {
            // Something horrible, irreparable has happened.
            flushPendingRequests();
            return;
        }
    }
    postReceiveReponseEvent();
}
```
注意这里的receiveRTSPReponse是有双重功能的，方面可以接收从服务器发来的请求，另一方面可以处理服务器发来的应答信号。

``` cpp
[->\frameworks\av\media\libstagefright\rtsp\ARTSPConnection.cpp]
bool ARTSPConnection::receiveRTSPReponse() {

    AString statusLine;
    if (!receiveLine(&statusLine)) {
        return false;
    }
    if (statusLine == "$") {
        sp<ABuffer> buffer = receiveBinaryData();
        if (buffer == NULL) {
            return false;
        }
        if (mObserveBinaryMessage != NULL) {
            sp<AMessage> notify = mObserveBinaryMessage->dup();
            notify->setBuffer("buffer", buffer);
            notify->post();
        } else {
            ALOGW("received binary data, but no one cares.");
        }
        return true;
    }

    //RTSP返回对象
    sp<ARTSPResponse> response = new ARTSPResponse;
    response->mStatusLine = statusLine;
    ALOGI("status: %s", response->mStatusLine.c_str());
    ssize_t space1 = response->mStatusLine.find(" ");
    if (space1 < 0) {
        return false;
    }
    ssize_t space2 = response->mStatusLine.find(" ", space1 + 1);
    if (space2 < 0) {
        return false;
    }

    bool isRequest = false;
    //判断返回的RTSP版本是否正确
    if (!IsRTSPVersion(AString(response->mStatusLine, 0, space1))) {
        CHECK(IsRTSPVersion(AString(response->mStatusLine,space2 + 1,response->mStatusLine.size() - space2 - 1)));
        isRequest = true;
        response->mStatusCode = 0;
    } else {
        //判断状态码是否正确
        AString statusCodeStr(response->mStatusLine, space1 + 1, space2 - space1 - 1);
        if (!ParseSingleUnsignedLong(statusCodeStr.c_str(), &response->mStatusCode) || response->mStatusCode < 100 || response->mStatusCode > 999) {
            return false;
        }
    }

    AString line;
    ssize_t lastDictIndex = -1;
    for (;;) {
        if (!receiveLine(&line)) {
            break;
        }
        if (line.empty()) {
            break;
        }
        ALOGV("line: '%s'", line.c_str());
        if (line.c_str()[0] == ' ' || line.c_str()[0] == '\t') {
            // Support for folded header values.
            if (lastDictIndex < 0) {
                // First line cannot be a continuation of the previous one.
                return false;
            }
            AString &value = response->mHeaders.editValueAt(lastDictIndex);
            value.append(line);

            continue;
        }
        ssize_t colonPos = line.find(":");
        if (colonPos < 0) {
            // Malformed header line.
            return false;
        }
        AString key(line, 0, colonPos);
        key.trim();
        key.tolower();
        line.erase(0, colonPos + 1);
        lastDictIndex = response->mHeaders.add(key, line);
    }

    for (size_t i = 0; i < response->mHeaders.size(); ++i) {
        response->mHeaders.editValueAt(i).trim();
    }

    unsigned long contentLength = 0;

    ssize_t i = response->mHeaders.indexOfKey("content-length");

    if (i >= 0) {
        AString value = response->mHeaders.valueAt(i);
        if (!ParseSingleUnsignedLong(value.c_str(), &contentLength)) {
            return false;
        }
    }
    //接收mContent
    if (contentLength > 0) {
        response->mContent = new ABuffer(contentLength);
        if (receive(response->mContent->data(), contentLength) != OK) {
            return false;
        }
    }
    //isRequest 表示是服务器主动发送的请求，那么将调用handleServerRequest，否则表示是服务器被动响应客户端的请求，那么将通知服务器有响应了notifyResponseListener
    return isRequest
        ? handleServerRequest(response)
        : notifyResponseListener(response);
}
```
isRequest 表示是服务器主动发送的请求，那么将调用handleServerRequest，否则表示是服务器被动响应客户端的请求，那么将通知服务器有响应了notifyResponseListener，我们这里先看下这两个方法的实现：

看到handleServerRequest大家可能会有点失望，因为这里尚未实现这个功能所以只是向服务器返回一个“RTSP/1.0 501 Not Implemented”的消息。

``` cpp
[->\frameworks\av\media\libstagefright\rtsp\ARTSPConnection.cpp]
bool ARTSPConnection::handleServerRequest(const sp<ARTSPResponse> &request) {
    // Implementation of server->client requests is optional for all methods
    // but we do need to respond, even if it's just to say that we don't
    // support the method.

    //这里我们不实现任何答复行为只是简单反馈我们尚未实现这个功能
    ssize_t space1 = request->mStatusLine.find(" ");
    CHECK_GE(space1, 0);
    AString response;
    response.append("RTSP/1.0 501 Not Implemented\r\n");
    ssize_t i = request->mHeaders.indexOfKey("cseq");
    if (i >= 0) {
        AString value = request->mHeaders.valueAt(i);
        unsigned long cseq;
        if (!ParseSingleUnsignedLong(value.c_str(), &cseq)) {
            return false;
        }
        response.append("CSeq: ");
        response.append(cseq);
        response.append("\r\n");
    }
    response.append("\r\n");
    size_t numBytesSent = 0;
    while (numBytesSent < response.size()) {
        ssize_t n =
            send(mSocket, response.c_str() + numBytesSent,
                 response.size() - numBytesSent, 0);
        if (n < 0 && errno == EINTR) {
            continue;
        }
        if (n <= 0) {
            if (n == 0) {
                // Server closed the connection.
                ALOGE("Server unexpectedly closed the connection.");
            } else {
                ALOGE("Error sending rtsp response (%s).", strerror(errno));
            }
            performDisconnect();
            return false;
        }
        numBytesSent += (size_t)n;
    }
    return true;
}
```
notifyResponseListener的实现比较清晰，它会根据服务器发来的应答响应，找出响应该应答的Message，然后将response返回给MyHandler，进行处理。

``` cpp
[->\frameworks\av\media\libstagefright\rtsp\ARTSPConnection.cpp]
bool ARTSPConnection::notifyResponseListener(
        const sp<ARTSPResponse> &response) {
    ssize_t i;
    //在队列中查找尚未处理的请求
    status_t err = findPendingRequest(response, &i);
    if (err != OK) {
        return false;
    }
    //发送服务器的回复给它
    sp<AMessage> reply = mPendingRequests.valueAt(i);
    mPendingRequests.removeItemsAt(i);
    reply->setInt32("result", OK);
    reply->setObject("response", response);
    reply->post();
    return true;
}
```
好了我们言归正传，我们看下MyHandler中对conn回复怎么处理：

``` cpp
[->\frameworks\av\media\libstagefright\rtsp\MyHandler.h]
case 'conn':
{
    int32_t result;
    //取出反馈结果
    CHECK(msg->findInt32("result", &result));
    if (result == OK) {
        //发送请求描述符的消息
        AString request;
        request = "DESCRIBE ";
        request.append(mSessionURL);
        request.append(" RTSP/1.0\r\n");
        request.append("Accept: application/sdp\r\n");
        request.append("\r\n");
        sp<AMessage> reply = new AMessage('desc', this);
        mConn->sendRequest(request.c_str(), reply);
    } else {
        (new AMessage('disc', this))->post();
    }
    break;
}
```
这里比较简单就是收到答复之后，直接判断结果是OK还是不OK，如果OK那么就发送一个DESCRIBE的请求。我们重点看下，onSendRequest理解这个很重要：
在onSendRequest中会对请求加工处理下，比如添加Cseq等操作，然后就会调用send向服务器发送请求。并将请求以Cseq为键码，replay为回复消息的待处理请求队列中。

``` cpp
[->\frameworks\av\media\libstagefright\rtsp\ARTSPConnection.cpp]
void ARTSPConnection::onSendRequest(const sp<AMessage> &msg) {
    sp<AMessage> reply;
    CHECK(msg->findMessage("reply", &reply));
    //对请求进行加工处理
    AString request;
    CHECK(msg->findString("request", &request));
    // Just in case we need to re-issue the request with proper authentication
    // later, stash it away.
    reply->setString("original-request", request.c_str(), request.size());
    addAuthentication(&request);
    addUserAgent(&request);
    // Find the boundary between headers and the body.
    ssize_t i = request.find("\r\n\r\n");
    CHECK_GE(i, 0);
    int32_t cseq = mNextCSeq++;
    AString cseqHeader = "CSeq: ";
    cseqHeader.append(cseq);
    cseqHeader.append("\r\n");
    request.insert(cseqHeader, i + 2);
    ALOGV("request: '%s'", request.c_str());

    size_t numBytesSent = 0;
    while (numBytesSent < request.size()) {
        //如果请求还没完全发送结束那么继续发送
        ssize_t n = send(mSocket, request.c_str() + numBytesSent,request.size() - numBytesSent, 0);}
        //忽略错误处理代码
        numBytesSent += (size_t)n;
    }
    //将请求添加到mPendingRequests，等待服务器回复
    mPendingRequests.add(cseq, reply);
}
```
回到上面提到的notifyResponseListener结合onSendRequest以及findPendingRequest是否看出了整个事件处理的流程？

``` cpp
status_t ARTSPConnection::findPendingRequest(
        const sp<ARTSPResponse> &response, ssize_t *index) const {
    *index = 0;

    ssize_t i = response->mHeaders.indexOfKey("cseq");

    if (i < 0) {
        // This is an unsolicited server->client message.
        *index = -1;
        return OK;
    }
    AString value = response->mHeaders.valueAt(i);

    unsigned long cseq;
    if (!ParseSingleUnsignedLong(value.c_str(), &cseq)) {
        return ERROR_MALFORMED;
    }
    i = mPendingRequests.indexOfKey(cseq);
    if (i < 0) {
        return -ENOENT;
    }
    *index = i;
    return OK;
}
```
onSendRequest 会不断将请求放入mPendingRequests中，而每次服务器给出应答的时候会调用notifyResponseListener，notifyResponseListener会从mPendingRequests中取出一个应答消息，并发送消息给MyHandler进行处理，而notifyResponseListener又会阻塞等待下一个服务器的应答信号。

OK我们接下来看下收到‘desc’信号后的处理：

``` cpp
case 'desc':
{
    int32_t result;
    CHECK(msg->findInt32("result", &result));

    if (result == OK) {
        sp<RefBase> obj;
        CHECK(msg->findObject("response", &obj));
        sp<ARTSPResponse> response = static_cast<ARTSPResponse *>(obj.get());
        
        if (response->mStatusCode == 301 || response->mStatusCode == 302) {
            //重定向连接
            //............
        }

        if (response->mStatusCode != 200) {
            result = UNKNOWN_ERROR;
        } else if (response->mContent == NULL) {
            result = ERROR_MALFORMED;
            ALOGE("The response has no content.");
        } else {
            //获得ASessionDescription
            mSessionDesc = new ASessionDescription;
            mSessionDesc->setTo(response->mContent->data(),response->mContent->size());

            if (!mSessionDesc->isValid()) {
                //............
            } else {
                //............
                if (mSessionDesc->countTracks() < 2) {
                    // There's no actual tracks in this session.
                    // The first "track" is merely session meta
                    // data.
                    ALOGW("Session doesn't contain any playable "
                         "tracks. Aborting.");
                    result = ERROR_UNSUPPORTED;
                } else {
                    //这里才是真正要处理的代码
                    setupTrack(1);
                }
            }
        }
    }
    break;
}
```
上面代码很长我们忽略不重要的，直接看setupTrack。

``` cpp
void setupTrack(size_t index) {

    sp<APacketSource> source = new APacketSource(mSessionDesc, index);
    AString url;
    CHECK(mSessionDesc->findAttribute(index, "a=control", &url));
    AString trackURL;
    //获得多媒体文件的Uri
    CHECK(MakeURL(mBaseURL.c_str(), url.c_str(), &trackURL));

    mTracks.push(TrackInfo());
    TrackInfo *info = &mTracks.editItemAt(mTracks.size() - 1);
    //设置uri
    info->mURL = trackURL;
    //设置APacketSource
    info->mPacketSource = source;
    info->mUsingInterleavedTCP = false;
    info->mFirstSeqNumInSegment = 0;
    info->mNewSegment = true;
    info->mRTPSocket = -1;
    info->mRTCPSocket = -1;
    info->mRTPAnchor = 0;
    info->mNTPAnchorUs = -1;
    info->mNormalPlayTimeRTP = 0;
    info->mNormalPlayTimeUs = 0ll;

    unsigned long PT;
    AString formatDesc;
    AString formatParams;
    mSessionDesc->getFormatType(index, &PT, &formatDesc, &formatParams);

    int32_t timescale;
    int32_t numChannels;
    ASessionDescription::ParseFormatDesc(formatDesc.c_str(), &timescale, &numChannels);

    info->mTimeScale = timescale;
    info->mEOSReceived = false;

    ALOGV("track #%zu URL=%s", mTracks.size(), trackURL.c_str());

    //建立SETUP请求
    AString request = "SETUP ";
    request.append(trackURL);
    request.append(" RTSP/1.0\r\n");
    if (mTryTCPInterleaving) {
        size_t interleaveIndex = 2 * (mTracks.size() - 1);
        info->mUsingInterleavedTCP = true;
        info->mRTPSocket = interleaveIndex;
        info->mRTCPSocket = interleaveIndex + 1;
        request.append("Transport: RTP/AVP/TCP;interleaved=");
        request.append(interleaveIndex);
        request.append("-");
        request.append(interleaveIndex + 1);
    } else {
        unsigned rtpPort;
        ARTPConnection::MakePortPair(
                &info->mRTPSocket, &info->mRTCPSocket, &rtpPort);
        if (mUIDValid) {
            HTTPBase::RegisterSocketUserTag(info->mRTPSocket, mUID,
                                            (uint32_t)*(uint32_t*) "RTP_");
            HTTPBase::RegisterSocketUserTag(info->mRTCPSocket, mUID,
                                            (uint32_t)*(uint32_t*) "RTP_");
            HTTPBase::RegisterSocketUserMark(info->mRTPSocket, mUID);
            HTTPBase::RegisterSocketUserMark(info->mRTCPSocket, mUID);
        }
        request.append("Transport: RTP/AVP/UDP;unicast;client_port=");
        request.append(rtpPort);
        request.append("-");
        request.append(rtpPort + 1);
    }
    request.append("\r\n");
    if (index > 1) {
        request.append("Session: ");
        request.append(mSessionID);
        request.append("\r\n");
    }
    request.append("\r\n");
    sp<AMessage> reply = new AMessage('setu', this);
    reply->setSize("index", index);
    reply->setSize("track-index", mTracks.size() - 1);
    mConn->sendRequest(request.c_str(), reply);
}
```
这里的逻辑也很简单就是将要获取到的歌曲信息存放到mTracks，并使用sendRequest发起setu请求，sendRequest就不再作详细介绍了，我们直接看下‘setu’返回后的处理：

``` cpp
case 'setu':
{
    size_t index;
    CHECK(msg->findSize("index", &index));
    TrackInfo *track = NULL;
    size_t trackIndex;
    if (msg->findSize("track-index", &trackIndex)) {
        track = &mTracks.editItemAt(trackIndex);
    }

    int32_t result;
    CHECK(msg->findInt32("result", &result));
    if (result == OK) {
        CHECK(track != NULL);
        sp<RefBase> obj;
        CHECK(msg->findObject("response", &obj));
        sp<ARTSPResponse> response =
            static_cast<ARTSPResponse *>(obj.get());

        if (response->mStatusCode != 200) {

        } else {
            ssize_t i = response->mHeaders.indexOfKey("session");
            CHECK_GE(i, 0);
            //得到SessionID
            mSessionID = response->mHeaders.valueAt(i);
            mKeepAliveTimeoutUs = kDefaultKeepAliveTimeoutUs;
            AString timeoutStr;
            //........................
            sp<AMessage> notify = new AMessage('accu', this);
            notify->setSize("track-index", trackIndex);
            i = response->mHeaders.indexOfKey("transport");
            CHECK_GE(i, 0);
            if (track->mRTPSocket != -1 && track->mRTCPSocket != -1) {
                if (!track->mUsingInterleavedTCP) {
                    AString transport = response->mHeaders.valueAt(i);
                    // We are going to continue even if we were
                    // unable to poke a hole into the firewall...
                    pokeAHole(
                            track->mRTPSocket,
                            track->mRTCPSocket,
                            transport);
                }
                mRTPConn->addStream(
                        track->mRTPSocket, track->mRTCPSocket,
                        mSessionDesc, index,
                        notify, track->mUsingInterleavedTCP);
                mSetupTracksSuccessful = true;
            } else {
                result = BAD_VALUE;
            }
        }
    }
```
上面最重要的就是获取SessionID并调用mRTPConn->addStream完ARTPConnection中添加一个流，我们看下addStream：

``` cpp
void ARTPConnection::addStream(
        int rtpSocket, int rtcpSocket,
        const sp<ASessionDescription> &sessionDesc,
        size_t index,
        const sp<AMessage> &notify,
        bool injected) {
    sp<AMessage> msg = new AMessage(kWhatAddStream, this);
    msg->setInt32("rtp-socket", rtpSocket);
    msg->setInt32("rtcp-socket", rtcpSocket);
    msg->setObject("session-desc", sessionDesc);
    msg->setSize("index", index);
    msg->setMessage("notify", notify);
    msg->setInt32("injected", injected);
    msg->post();
}
case kWhatAddStream:
{
    onAddStream(msg);
    break;
}
void ARTPConnection::onAddStream(const sp<AMessage> &msg) {
    //将Stream信息添加到mStreams
    mStreams.push_back(StreamInfo());
    StreamInfo *info = &*--mStreams.end();
    int32_t s;
    //获得rtp-socket
    CHECK(msg->findInt32("rtp-socket", &s));
    info->mRTPSocket = s;
    //获得rtcp-socket
    CHECK(msg->findInt32("rtcp-socket", &s));
    info->mRTCPSocket = s;

    int32_t injected;
    CHECK(msg->findInt32("injected", &injected));

    info->mIsInjected = injected;
    //获得session-desc
    sp<RefBase> obj;
    CHECK(msg->findObject("session-desc", &obj));
    info->mSessionDesc = static_cast<ASessionDescription *>(obj.get());

    CHECK(msg->findSize("index", &info->mIndex));
    CHECK(msg->findMessage("notify", &info->mNotifyMsg));

    info->mNumRTCPPacketsReceived = 0;
    info->mNumRTPPacketsReceived = 0;
    memset(&info->mRemoteRTCPAddr, 0, sizeof(info->mRemoteRTCPAddr));

    //发送轮询查询事件
    if (!injected) {
        postPollEvent();
    }
}
```
上面代码中重点关注的是postPollEvent：

``` cpp
void ARTPConnection::postPollEvent() {
    if (mPollEventPending) {
        return;
    }
    sp<AMessage> msg = new AMessage(kWhatPollStreams, this);
    msg->post();
    mPollEventPending = true;
}
case kWhatPollStreams:
{
    onPollStreams();
    break;
}
void ARTPConnection::onPollStreams() {
    mPollEventPending = false;

    if (mStreams.empty()) {
        return;
    }

    struct timeval tv;
    tv.tv_sec = 0;
    tv.tv_usec = kSelectTimeoutUs;

    fd_set rs;
    FD_ZERO(&rs);

    int maxSocket = -1;
    for (List<StreamInfo>::iterator it = mStreams.begin();
         it != mStreams.end(); ++it) {
        if ((*it).mIsInjected) {
            continue;
        }
        FD_SET(it->mRTPSocket, &rs);
        FD_SET(it->mRTCPSocket, &rs);
        if (it->mRTPSocket > maxSocket) {
            maxSocket = it->mRTPSocket;
        }
        if (it->mRTCPSocket > maxSocket) {
            maxSocket = it->mRTCPSocket;
        }
    }

    if (maxSocket == -1) {
        return;
    }

    //选择一个网络请求
    int res = select(maxSocket + 1, &rs, NULL, NULL, &tv);

    if (res > 0) {
        //在这里接收服务器发过来的数据
        List<StreamInfo>::iterator it = mStreams.begin();
        while (it != mStreams.end()) {
            if ((*it).mIsInjected) {
                ++it;
                continue;
            }
            status_t err = OK;
            //接受从服务器发来的数据
            if (FD_ISSET(it->mRTPSocket, &rs)) {
                //调用的是status_t ARTPConnection::receive(StreamInfo *s, bool receiveRTP)
                err = receive(&*it, true);
            }
            //接受从服务器发来的数据
            if (err == OK && FD_ISSET(it->mRTCPSocket, &rs)) {
                //调用的是status_t ARTPConnection::receive(StreamInfo *s, bool receiveRTP)
                err = receive(&*it, false);
            }
            ++it;
        }
    }

    int64_t nowUs = ALooper::GetNowUs();
    if (mLastReceiverReportTimeUs <= 0|| mLastReceiverReportTimeUs + 5000000ll <= nowUs) {
        //新建一个缓存区
        sp<ABuffer> buffer = new ABuffer(kMaxUDPSize);
        List<StreamInfo>::iterator it = mStreams.begin();
        while (it != mStreams.end()) {
            StreamInfo *s = &*it;
            if (s->mIsInjected) {
                ++it;
                continue;
            }
            if (s->mNumRTCPPacketsReceived == 0) {
                // We have never received any RTCP packets on this stream,
                // we don't even know where to send a report.
                ++it;
                continue;
            }

            buffer->setRange(0, 0);
            for (size_t i = 0; i < s->mSources.size(); ++i) {
                sp<ARTPSource> source = s->mSources.valueAt(i);
                //填充buffer
                source->addReceiverReport(buffer);
                if (mFlags & kRegularlyRequestFIR) {
                    source->addFIR(buffer);
                }
            }
            if (buffer->size() > 0) {
                ALOGV("Sending RR...");
                ssize_t n;
                do {
                    //通過RTCPSocket發送
                    n = sendto(s->mRTCPSocket, buffer->data(), buffer->size(), 0,(const struct sockaddr *)&s->mRemoteRTCPAddr, sizeof(s->mRemoteRTCPAddr));
                } while (n < 0 && errno == EINTR);
                CHECK_EQ(n, (ssize_t)buffer->size());
                mLastReceiverReportTimeUs = nowUs;
            }
            ++it;
        }
    }

    if (!mStreams.empty()) {
        postPollEvent();
    }
}
```
再onPollStreams中会阻塞监听服务器发过来的媒体数据，并调用receive对其进行处理，并定期发送RTCP消息给服务器。

``` cpp
status_t ARTPConnection::receive(StreamInfo *s, bool receiveRTP) {
    
    ALOGV("receiving %s", receiveRTP ? "RTP" : "RTCP");

    CHECK(!s->mIsInjected);

    sp<ABuffer> buffer = new ABuffer(65536);

    socklen_t remoteAddrLen =
        (!receiveRTP && s->mNumRTCPPacketsReceived == 0)
            ? sizeof(s->mRemoteRTCPAddr) : 0;

    ssize_t nbytes;
    do {
        //从服务器接收数据
        nbytes = recvfrom(
            receiveRTP ? s->mRTPSocket : s->mRTCPSocket,
            buffer->data(),
            buffer->capacity(),
            0,
            remoteAddrLen > 0 ? (struct sockaddr *)&s->mRemoteRTCPAddr : NULL,
            remoteAddrLen > 0 ? &remoteAddrLen : NULL);
    } while (nbytes < 0 && errno == EINTR);

    if (nbytes <= 0) {
        return -ECONNRESET;
    }

    buffer->setRange(0, nbytes);

    // ALOGI("received %d bytes.", buffer->size());

    status_t err;
    //解析RTP 或者 parseRTCP
    if (receiveRTP) {
        err = parseRTP(s, buffer);
    } else {
        err = parseRTCP(s, buffer);
    }

    return err;
}
```
receive方法中会调用recvfrom。将数据从服务器中读取到缓存，并调用parseRTP或者parseRTCP对缓存中的数据进行处理

``` cpp
status_t ARTPConnection::parseRTP(StreamInfo *s, const sp<ABuffer> &buffer) {
    
    const uint8_t *data = buffer->data();

    if ((data[0] >> 6) != 2) {
        // Unsupported version.
        return -1;
    }

    if (data[0] & 0x20) {
        // Padding present.

        size_t paddingLength = data[size - 1];

        if (paddingLength + 12 > size) {
            // If we removed this much padding we'd end up with something
            // that's too short to be a valid RTP header.
            return -1;
        }

        size -= paddingLength;
    }

    int numCSRCs = data[0] & 0x0f;

    size_t payloadOffset = 12 + 4 * numCSRCs;

    if (size < payloadOffset) {
        // Not enough data to fit the basic header and all the CSRC entries.
        return -1;
    }

    if (data[0] & 0x10) {
        // Header eXtension present.

        if (size < payloadOffset + 4) {
            // Not enough data to fit the basic header, all CSRC entries
            // and the first 4 bytes of the extension header.

            return -1;
        }

        const uint8_t *extensionData = &data[payloadOffset];

        size_t extensionLength =
            4 * (extensionData[2] << 8 | extensionData[3]);

        if (size < payloadOffset + 4 + extensionLength) {
            return -1;
        }

        payloadOffset += 4 + extensionLength;
    }

    uint32_t srcId = u32at(&data[8]);

    sp<ARTPSource> source = findSource(s, srcId);

    uint32_t rtpTime = u32at(&data[4]);

    sp<AMessage> meta = buffer->meta();
    meta->setInt32("ssrc", srcId);
    meta->setInt32("rtp-time", rtpTime);
    meta->setInt32("PT", data[1] & 0x7f);
    meta->setInt32("M", data[1] >> 7);

    buffer->setInt32Data(u16at(&data[2]));
    buffer->setRange(payloadOffset, size - payloadOffset);
    //这里十分重要void ARTPSource::processRTPPacket(const sp<ABuffer> &buffer)
    source->processRTPPacket(buffer);
    return OK;
}
```
在parsRTP中根据RTP格式对缓存区中的数据进行解析，最后调用ARTPSource::processRTPPacket进行后续处理。

``` cpp
[->\frameworks\av\media\libstagefright\rtsp\ARTPSource.cpp]
void ARTPSource::processRTPPacket(const sp<ABuffer> &buffer) {
    if (queuePacket(buffer) && mAssembler != NULL) {
        mAssembler->onPacketReceived(this);
    }
}
```
processRTPPacket中调用Assembler来将数据进行重组，这里最重要的方法是assembleMore

``` cpp
[->\frameworks\av\media\libstagefright\rtsp\ARTPAssembler.cpp]
void ARTPAssembler::onPacketReceived(const sp<ARTPSource> &source) {
    AssemblyStatus status;
    for (;;) {
        //assembleMore
        status = assembleMore(source);
        if (status == WRONG_SEQUENCE_NUMBER) {
            if (mFirstFailureTimeUs >= 0) {
                if (ALooper::GetNowUs() - mFirstFailureTimeUs > 10000ll) {
                    mFirstFailureTimeUs = -1;
                    // LOG(VERBOSE) << "waited too long for packet.";
                    packetLost();
                    continue;
                }
            } else {
                mFirstFailureTimeUs = ALooper::GetNowUs();
            }
            break;
        } else {
            mFirstFailureTimeUs = -1;
            if (status == NOT_ENOUGH_DATA) {
                break;
            }
        }
    }
}
ARTPAssembler::AssemblyStatus AMPEG4AudioAssembler::assembleMore(
        const sp<ARTPSource> &source) {
        //调用addPacket
    AssemblyStatus status = addPacket(source);
    if (status == MALFORMED_PACKET) {
        mAccessUnitDamaged = true;
    }
    return status;
}
```
这里实际上是对无序的数据包进行排序，并调用submitAccessUnit提交AU数据。

``` cpp
ARTPAssembler::AssemblyStatus AMPEG4AudioAssembler::addPacket(
        const sp<ARTPSource> &source) {

    List<sp<ABuffer> > *queue = source->queue();
    if (queue->empty()) {
        return NOT_ENOUGH_DATA;
    }
    if (mNextExpectedSeqNoValid) {
        List<sp<ABuffer> >::iterator it = queue->begin();
        while (it != queue->end()) {
            if ((uint32_t)(*it)->int32Data() >= mNextExpectedSeqNo) {
                break;
            }
            it = queue->erase(it);
        }
        if (queue->empty()) {
            return NOT_ENOUGH_DATA;
        }
    }

    sp<ABuffer> buffer = *queue->begin();

    if (!mNextExpectedSeqNoValid) {
        mNextExpectedSeqNoValid = true;
        mNextExpectedSeqNo = (uint32_t)buffer->int32Data();
    } else if ((uint32_t)buffer->int32Data() != mNextExpectedSeqNo) {
#if VERBOSE
        LOG(VERBOSE) << "Not the sequence number I expected";
#endif
        return WRONG_SEQUENCE_NUMBER;
    }

    uint32_t rtpTime;
    CHECK(buffer->meta()->findInt32("rtp-time", (int32_t *)&rtpTime));

    //提交AccessUnit
    if (mPackets.size() > 0 && rtpTime != mAccessUnitRTPTime) {
        submitAccessUnit();
    }
    mAccessUnitRTPTime = rtpTime;
    //将缓存添加到mPackets
    mPackets.push_back(buffer);
    queue->erase(queue->begin());
    ++mNextExpectedSeqNo;
    return OK;
}
```
submitAccessUnit中回调‘accu’，交给MyHandler处理

``` cpp
void AMPEG4AudioAssembler::submitAccessUnit() {
    CHECK(!mPackets.empty());

#if VERBOSE
    LOG(VERBOSE) << "Access unit complete (" << mPackets.size() << " packets)";
#endif

    sp<ABuffer> accessUnit = MakeCompoundFromPackets(mPackets);
    accessUnit = removeLATMFraming(accessUnit);
    CopyTimes(accessUnit, *mPackets.begin());

    if (mAccessUnitDamaged) {
        accessUnit->meta()->setInt32("damaged", true);
    }

    mPackets.clear();
    mAccessUnitDamaged = false;
    //回调‘accu’
    sp<AMessage> msg = mNotifyMsg->dup();
    msg->setBuffer("access-unit", accessUnit);
    msg->post();
}

case 'accu':
{
    int32_t timeUpdate;
    if (msg->findInt32("time-update", &timeUpdate) && timeUpdate) {
        size_t trackIndex;
        CHECK(msg->findSize("track-index", &trackIndex));

        uint32_t rtpTime;
        uint64_t ntpTime;
        CHECK(msg->findInt32("rtp-time", (int32_t *)&rtpTime));
        CHECK(msg->findInt64("ntp-time", (int64_t *)&ntpTime));

        onTimeUpdate(trackIndex, rtpTime, ntpTime);
        break;
    }

    int32_t first;
    if (msg->findInt32("first-rtcp", &first)) {
        mReceivedFirstRTCPPacket = true;
        break;
    }

    if (msg->findInt32("first-rtp", &first)) {
        mReceivedFirstRTPPacket = true;
        break;
    }

    ++mNumAccessUnitsReceived;
    postAccessUnitTimeoutCheck();

    size_t trackIndex;
    CHECK(msg->findSize("track-index", &trackIndex));

    if (trackIndex >= mTracks.size()) {
        ALOGV("late packets ignored.");
        break;
    }

    TrackInfo *track = &mTracks.editItemAt(trackIndex);

    int32_t eos;
    if (msg->findInt32("eos", &eos)) {
        ALOGI("received BYE on track index %zu", trackIndex);
        if (!mAllTracksHaveTime && dataReceivedOnAllChannels()) {
            ALOGI("No time established => fake existing data");

            track->mEOSReceived = true;
            mTryFakeRTCP = true;
            mReceivedFirstRTCPPacket = true;
            fakeTimestamps();
        } else {
            postQueueEOS(trackIndex, ERROR_END_OF_STREAM);
        }
        return;
    }

    sp<ABuffer> accessUnit;
    //取出accessUnit
    CHECK(msg->findBuffer("access-unit", &accessUnit));

    uint32_t seqNum = (uint32_t)accessUnit->int32Data();

    if (mSeekPending) {
        ALOGV("we're seeking, dropping stale packet.");
        break;
    }

    if (seqNum < track->mFirstSeqNumInSegment) {
        ALOGV("dropping stale access-unit (%d < %d)",
             seqNum, track->mFirstSeqNumInSegment);
        break;
    }

    if (track->mNewSegment) {
        track->mNewSegment = false;
    }
    //调用onAccessUnitComplete
    onAccessUnitComplete(trackIndex, accessUnit);
    break;
}
```
‘accu’取出AU数据后调用onAccessUnitComplete进行处理，我们接下来看下这部分逻辑：

``` cpp
void onAccessUnitComplete(
        int32_t trackIndex, const sp<ABuffer> &accessUnit) {
    ALOGV("onAccessUnitComplete track %d", trackIndex);

    if(!mPlayResponseParsed){
        ALOGI("play response is not parsed, storing accessunit");
        TrackInfo *track = &mTracks.editItemAt(trackIndex);
        track->mPackets.push_back(accessUnit);
        return;
    }

    handleFirstAccessUnit();

    TrackInfo *track = &mTracks.editItemAt(trackIndex);

    if (!mAllTracksHaveTime) {
        ALOGV("storing accessUnit, no time established yet");
        track->mPackets.push_back(accessUnit);
        return;
    }

    while (!track->mPackets.empty()) {
        sp<ABuffer> accessUnit = *track->mPackets.begin();
        track->mPackets.erase(track->mPackets.begin());

        if (addMediaTimestamp(trackIndex, track, accessUnit)) {
            //postQueueAccessUnit
            postQueueAccessUnit(trackIndex, accessUnit);
        }
    }

    if (addMediaTimestamp(trackIndex, track, accessUnit)) {
        postQueueAccessUnit(trackIndex, accessUnit);
    }

    if (track->mEOSReceived) {
        postQueueEOS(trackIndex, ERROR_END_OF_STREAM);
        track->mEOSReceived = false;
    }
}
void postQueueAccessUnit(
        size_t trackIndex, const sp<ABuffer> &accessUnit) {
    sp<AMessage> msg = mNotify->dup();
    msg->setInt32("what", kWhatAccessUnit);
    msg->setSize("trackIndex", trackIndex);
    msg->setBuffer("accessUnit", accessUnit);
    msg->post();
}
```
在RTSPSource中调用AnotherPacketSource queueAccessUnit(accessUnit)

``` cpp
case MyHandler::kWhatAccessUnit:
{
    size_t trackIndex;
    CHECK(msg->findSize("trackIndex", &trackIndex));

    if (mTSParser == NULL) {
        CHECK_LT(trackIndex, mTracks.size());
    } else {
        CHECK_EQ(trackIndex, 0u);
    }

    sp<ABuffer> accessUnit;
    CHECK(msg->findBuffer("accessUnit", &accessUnit));

    int32_t damaged;
    if (accessUnit->meta()->findInt32("damaged", &damaged)
            && damaged) {
        ALOGI("dropping damaged access unit.");
        break;
    }

    if (mTSParser != NULL) {
        size_t offset = 0;
        status_t err = OK;
        while (offset + 188 <= accessUnit->size()) {
            err = mTSParser->feedTSPacket(
                    accessUnit->data() + offset, 188);
            if (err != OK) {
                break;
            }

            offset += 188;
        }

        if (offset < accessUnit->size()) {
            err = ERROR_MALFORMED;
        }

        if (err != OK) {
            sp<AnotherPacketSource> source = getSource(false /* audio */);
            if (source != NULL) {
                source->signalEOS(err);
            }

            source = getSource(true /* audio */);
            if (source != NULL) {
                source->signalEOS(err);
            }
        }
        break;
    }

    TrackInfo *info = &mTracks.editItemAt(trackIndex);

    sp<AnotherPacketSource> source = info->mSource;
    if (source != NULL) {
        uint32_t rtpTime;
        CHECK(accessUnit->meta()->findInt32("rtp-time", (int32_t *)&rtpTime));

        if (!info->mNPTMappingValid) {
            // This is a live stream, we didn't receive any normal
            // playtime mapping. We won't map to npt time.
            source->queueAccessUnit(accessUnit);
            break;
        }

        int64_t nptUs =
            ((double)rtpTime - (double)info->mRTPTime)
                / info->mTimeScale
                * 1000000ll
                + info->mNormalPlaytimeUs;

        accessUnit->meta()->setInt64("timeUs", nptUs);
        //......
        source->queueAccessUnit(accessUnit);
    }
    break;
}
```
queueAccessUnit(accessUnit);将AU数据存放到AnotherPacketSource 的mBuffers中供解码器解码播放：

``` cpp
void AnotherPacketSource::queueAccessUnit(const sp<ABuffer> &buffer) {
    int32_t damaged;
    ......
    Mutex::Autolock autoLock(mLock);
    mBuffers.push_back(buffer);
    mCondition.signal();

    int32_t discontinuity;
    if (buffer->meta()->findInt32("discontinuity", &discontinuity)){
        ALOGV("queueing a discontinuity with queueAccessUnit");

        mLastQueuedTimeUs = 0ll;
        mEOSResult = OK;
        mLatestEnqueuedMeta = NULL;

        mDiscontinuitySegments.push_back(DiscontinuitySegment());
        return;
    }

    int64_t lastQueuedTimeUs;
    CHECK(buffer->meta()->findInt64("timeUs", &lastQueuedTimeUs));
    mLastQueuedTimeUs = lastQueuedTimeUs;
    ALOGV("queueAccessUnit timeUs=%" PRIi64 " us (%.2f secs)",
            mLastQueuedTimeUs, mLastQueuedTimeUs / 1E6);

    // CHECK(!mDiscontinuitySegments.empty());
    DiscontinuitySegment &tailSeg = *(--mDiscontinuitySegments.end());
    if (lastQueuedTimeUs > tailSeg.mMaxEnqueTimeUs) {
        tailSeg.mMaxEnqueTimeUs = lastQueuedTimeUs;
    }
    if (tailSeg.mMaxDequeTimeUs == -1) {
        tailSeg.mMaxDequeTimeUs = lastQueuedTimeUs;
    }

    if (mLatestEnqueuedMeta == NULL) {
        mLatestEnqueuedMeta = buffer->meta()->dup();
    } else {
        int64_t latestTimeUs = 0;
        int64_t frameDeltaUs = 0;
        CHECK(mLatestEnqueuedMeta->findInt64("timeUs", &latestTimeUs));
        if (lastQueuedTimeUs > latestTimeUs) {
            mLatestEnqueuedMeta = buffer->meta()->dup();
            frameDeltaUs = lastQueuedTimeUs - latestTimeUs;
            mLatestEnqueuedMeta->setInt64("durationUs", frameDeltaUs);
        } else if (!mLatestEnqueuedMeta->findInt64("durationUs", &frameDeltaUs)) {
            // For B frames
            frameDeltaUs = latestTimeUs - lastQueuedTimeUs;
            mLatestEnqueuedMeta->setInt64("durationUs", frameDeltaUs);
        }
    }
}
```

##### 3.4、获取数据进行编码（请参考前面HLS小节分析）

##### 3.5、播放流程（请参考前面HLS小节分析）

#### （四）、参考资料(特别感谢各位前辈的分析和图示)：
[android ACodec MediaCodec NuPlayer flow - CSDN博客](https://blog.csdn.net/dfhuang09/article/details/54926526)
[android MediaCodec ACodec - CSDN博客](https://blog.csdn.net/dfhuang09/article/details/60132620)
[Android 源码分析之基于NuPlayer的HLS流媒体协议](https://tbfungeek.github.io/2016/08/02/Android-%E8%BF%9B%E9%98%B6%E4%B9%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E5%9F%BA%E4%BA%8ENuPlayer%E7%9A%84HLS%E6%B5%81%E5%AA%92%E4%BD%93%E5%8D%8F%E8%AE%AE/)
[Android 源码分析之基于NuPlayer的RTSP流媒体协议](https://tbfungeek.github.io/2016/08/02/Android-%E8%BF%9B%E9%98%B6%E4%B9%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E5%9F%BA%E4%BA%8ENuPlayer%E7%9A%84RTSP%E6%B5%81%E5%AA%92%E4%BD%93%E5%8D%8F%E8%AE%AE/)

