---
title: Android Video System（7）：Android Multimedia Codecs - AAC编解码分析
cover: https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/hexo.themes/bing-wallpaper-2018.04.29.jpg
categories:
  - Multimedia
tags:
  - Android
  - Video
  - Multimedia
toc: true
abbrlink: 20190214
date: 2019-02-14 09:25:00
---

--------------------------------------------------------------------------------

注：文章都是通过阅读各位前辈总结的资料 Android 8.x && Linux（kernel 4.x）Qualcomm平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网（◕‿◕），如有侵权，请联系删除，禁止转载（©Qualcomm ©Android @Linux 版权所有），谢谢。

首先感谢：

[AAC音频格式分析与解码](http://www.cnblogs.com/caosiyang/archive/2012/07/16/2594029.html)
[android ACodec MediaCodec NuPlayer flow](https://blog.csdn.net/dfhuang09/article/details/54926526)

正是由于前人的分析和总结，帮助我节约了大量的时间和精力，再次感谢！！！

Google Pixel、Pixel XL 内核代码（==**文章基于 Kernel-3.x**==）：
 [Kernel source for Pixel 2 (walleye) and Pixel 2 XL (taimen) - GitHub](https://github.com/nathanchance/wahoo)

AOSP 源码（==**文章基于 Android 7.x**==）：
 [ Android 系统全套源代码分享 (更新到 8.1.0_r1)](https://testerhome.com/topics/2229)

--------------------------------------------------------------------------------

``` cpp
\frameworks\av\media\libstagefright\omx\SoftOMXPlugin.cpp
kComponents[] = {
    { "OMX.google.aac.decoder", "aacdec", "audio_decoder.aac" },
    { "OMX.google.aac.encoder", "aacenc", "audio_encoder.aac" },
    ......
};
```

==源码（部分）==：

>PATH : \frameworks\av\media\libstagefright\codecs （Android Codecs）

- aacdec
- aacenc

> \external\aac（libFraunhoferAAC 库）

- libAACdec
- libAACenc
- libFDK
- documentation\aacDecoder.pdf
- documentation\aacEncoder.pdf


> \cts\tests\tests\media\res\raw（资源文件）

- loudsoftaac.aac

Log：
[AAC-DEcorder-Google-Log.md](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/android.codecs/AAC-DEcorder-Google-Log.md)
[AAC-Encorder-Google-Log.md](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/android.codecs/AAC-Encorder-Google-Log.md)

--------------------------------------------------------------------------------

#### （一）、AAC音频格式分析（概览）

##### 1.1、AAC概述
[wiki/進階音訊編碼](https://zh.wikipedia.org/wiki/%E9%80%B2%E9%9A%8E%E9%9F%B3%E8%A8%8A%E7%B7%A8%E7%A2%BC)
高级音频编码（英语：Advanced Audio Coding，AAC），出现于1997年，为一种基于MPEG-2的有损数字音频压缩的专利音频编码标准，由Fraunhofer IIS、杜比实验室、AT&T、Sony、Nokia等公司共同开发。2000年，MPEG-4标准在原本的基础上加上了PNS（Perceptual Noise Substitution）等技术，并提供了多种扩展工具。为了区别于传统的MPEG-2 AAC又称为MPEG-4 AAC。其作为MP3的后继者而被设计出来，在相同的比特率之下，AAC相较于MP3通常可以达到更好的声音品质[2]。

AAC由国际标准化组织及国际电工委员会标准化为MPEG-2及MPEG-4规格的一部分。[3][4]部分的AAC、HE-AAC(AAC+)为MPEG-4音频的一部分，并且被采用在数字声音广播、世界数字广播两个数字广播标准中以及DVB-H、ATSC-M/H两个移动电视标准中。

AAC支持包含一个流中48个最高至96 kHz的全带宽声道，加上16个120 Hz的低频声道(LFE)、不多于16个耦合声道及数据流。在joint stereo模式下，要使立体声的品质达到可接受的程度仅需96 kbps的比特率，若要达到Hi-fi则最少需要在可变码率下128 kbps。

AAC 被YouTube、iPhone、iPod、 iPad、 任天堂DSi、任天堂3DS、iTunes、DivX、PlayStation 3和多款Nokia 40系列手机采用为默认的音频编码格式，并且被PlayStation Vita、Wii、Sony Walkman MP3系列及随后的Android、BlackBerry等移动操作系统支持。

AAC编码的主要扩展名有三种：

.aac - 使用MPEG-2 Audio Transport Stream（ADTS，参见MPEG-2）容器，区别于使用MPEG-4容器的MP4/M4A格式，属于传统的AAC编码（FAAC默认的封装，但FAAC亦可输出MPEG-4封装的AAC）。
.mp4 - 使用了MPEG-4 Part 14（第14部分）的简化版即3GPP Media Release 6 Basic（3gp6，参见3GP）进行封装的AAC编码（Nero AAC编码器仅能输出MPEG-4封装的AAC）。
.m4a - 为了区别纯音频MP4文件和包含视频的MP4文件而由苹果（Apple）公司使用的扩展名，Apple iTunes对纯音频MP4文件采用了".m4a"命名。M4A的本质和音频MP4相同，故音频MP4文件亦可直接更改扩展名为M4A。

##### 1.2、AAC音频格式分析

AAC音频格式有ADIF和ADTS：

ADIF：Audio Data Interchange Format 音频数据交换格式。这种格式的特征是可以确定的找到这个音频数据的开始，不需进行在音频数据流中间开始的解码，即它的解码必须在明确定义的开始处进行。故这种格式常用在磁盘文件中。

ADTS：Audio Data Transport Stream 音频数据传输流。这种格式的特征是它是一个有同步字的比特流，解码可以在这个流中任何位置开始。它的特征类似于mp3数据流格式。

简单说，ADTS可以在任意帧解码，也就是说它每一帧都有头信息。ADIF只有一个统一的头，所以必须得到所有的数据后解码。且这两种的header的格式也是不同的，目前一般编码后的和抽取出的都是ADTS格式的音频流。

语音系统对实时性要求较高，基本是这样一个流程，采集音频数据，本地编码，数据上传，服务器处理，数据下发，本地解码

ADTS是帧序列，本身具备流特征，在音频流的传输与处理方面更加合适。

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/android.codecs/VS7-01-AAC-ADTS.png)

ADTS是AAC音频文件常见的传输格式。当你编码AAC裸流的时候，会遇到写出来的AAC文件并不能在PC和手机上播放，很大的可能就是AAC文件的每一帧里缺少了ADTS头信息文件的包装拼接。只需要加入头文件ADTS即可。一个AAC原始数据块长度是可变的，对原始帧加上ADTS头进行ADTS的封装，就形成了ADTS帧。

 图中表示出了ADTS一帧的简明结构，其两边的空白矩形表示一帧前后的数据。

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/android.codecs/VS7-02-AAC-ADTS-hejunlin.png)


#### （二）、Android AAC音频 解码

前面的文章已经分析过音视频录制和播放流程了，首先回忆一下播放流程：
[android ACodec MediaCodec NuPlayer flow](https://blog.csdn.net/dfhuang09/article/details/54926526)

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/android.codecs/VS7-03-Media-ACodec.png)

AAC播放流程基本一致，所以这里专注于分析下Android  AAC编解码流程，首先看看时序图：

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/android.codecs/VS7-04-AAC-Decord-flow.png)

##### 2.1、initDecoder()
从打印Log[]()可以看出，在初始化解码器的时候是用的Google的aac.decoder解码：
``` cpp

12-25 23:12:00.176: V/ACodec(795): Now uninitialized
12-25 23:12:00.176: V/ACodec(795): onAllocateComponent
12-25 23:12:00.178: I/MediaPlayerService(795): MediaPlayerService::getOMX
12-25 23:12:00.182: I/OMXClient(795): MuxOMX ctor
12-25 23:12:00.182: V/MediaCodecList(795): matching 'OMX.google.aac.decoder'
12-25 23:12:00.183: V/OMXNodeInstance(785): debug level for OMX.google.aac.decoder is 0
12-25 23:12:00.183: I/OMXMaster(785): makeComponentInstance(OMX.google.aac.decoder) in mediacodec process
12-25 23:12:00.184: V/SoftOMXPlugin(785): makeComponentInstance 'OMX.google.aac.decoder'
12-25 23:12:00.188: V/SoftAAC2(785): initDecoder()
```
主要是调用libFraunhoferAAC 库完成一些参数初始化：
``` cpp
Z:\MediaLearing\frameworks\av\media\libstagefright\codecs\aacdec\SoftAAC2.cpp
status_t SoftAAC2::initDecoder() {
    ALOGV("initDecoder()");
    status_t status = UNKNOWN_ERROR;
    // 打开aacDecoder
    mAACDecoder = aacDecoder_Open(TT_MP4_ADIF, /* num layers */ 1);
    .....

    //init DRC wrapper
    mDrcWrap.setDecoderHandle(mAACDecoder);
    mDrcWrap.submitStreamData(mStreamInfo);
    ....
    // By default, the decoder creates a 5.1 channel downmix signal.
    // For seven and eight channel input streams, enable 6.1 and 7.1 channel output
    aacDecoder_SetParam(mAACDecoder, AAC_PCM_MAX_OUTPUT_CHANNELS, -1);

    return status;
}
```
我们看看打开过程，只贴出代码，不做具体分析：

``` cpp
Z:\MediaLearing\external\aac\libAACdec\src\aacdecoder_lib.cpp
LINKSPEC_CPP HANDLE_AACDECODER aacDecoder_Open(TRANSPORT_TYPE transportFmt, UINT nrOfLayers)
{
  AAC_DECODER_INSTANCE *aacDec = NULL;
  HANDLE_TRANSPORTDEC pIn;
  int err = 0;

  /* Allocate transport layer struct. */
  pIn = transportDec_Open(transportFmt, TP_FLAG_MPEG4);
  ......

  transportDec_SetParam(pIn, TPDEC_PARAM_IGNORE_BUFFERFULLNESS, 1);

  /* Allocate AAC decoder core struct. */
  aacDec = CAacDecoder_Open(transportFmt);

 ......
  aacDec->hInput = pIn;

  aacDec->nrOfLayers = nrOfLayers;

  aacDec->channelOutputMapping = channelMappingTableWAV;

  /* Register Config Update callback. */
  transportDec_RegisterAscCallback(pIn, aacDecoder_ConfigCallback, (void*)aacDec);

  /* open SBR decoder */
  if ( SBRDEC_OK != sbrDecoder_Open ( &aacDec->hSbrDecoder )) {
    ......
  }
  aacDec->qmfModeUser = NOT_DEFINED;
  transportDec_RegisterSbrCallback(aacDec->hInput, (cbSbr_t)sbrDecoder_Header, (void*)aacDec->hSbrDecoder);
  ......
  pcmDmx_Open( &aacDec->hPcmUtils );
 ......
  aacDec->hLimiter = createLimiter(TDL_ATTACK_DEFAULT_MS, TDL_RELEASE_DEFAULT_MS, SAMPLE_MAX, (8), 96000);
  ......
  aacDec->limiterEnableUser = (UCHAR)-1;
  aacDec->limiterEnableCurr = 0;
  ......
  return aacDec;
}
```
具体的就不深入分析了，等有具体问题再来研究。

##### 2.2、fillThisBuffer()
使用ACodec填充数据的时候，会一直调用到SoftAAC2::onQueueFilled()函数：

``` cpp
Z:\MediaLearing\frameworks\av\media\libstagefright\codecs\aacdec\SoftAAC2.cpp

void SoftAAC2::onQueueFilled(OMX_U32 /* portIndex */) {
    ......
    UCHAR* inBuffer[FILEREAD_MAX_LAYERS];
    UINT inBufferLength[FILEREAD_MAX_LAYERS] = {0};
    UINT bytesValid[FILEREAD_MAX_LAYERS] = {0};

    List<BufferInfo *> &inQueue = getPortQueue(0);
    List<BufferInfo *> &outQueue = getPortQueue(1);

    while ((!inQueue.empty() || mEndOfInput) && !outQueue.empty()) {
        if (!inQueue.empty()) {
            INT_PCM tmpOutBuffer[2048 * MAX_CHANNEL_COUNT];
            BufferInfo *inInfo = *inQueue.begin();
            OMX_BUFFERHEADERTYPE *inHeader = inInfo->mHeader;

            mEndOfInput = (inHeader->nFlags & OMX_BUFFERFLAG_EOS) != 0;

            ......

           if (mIsADTS) {
                size_t adtsHeaderSize = 0;
                // skip 30 bits, aac_frame_length follows.
                // ssssssss ssssiiip ppffffPc ccohCCll llllllll lll?????

                const uint8_t *adtsHeader = inHeader->pBuffer + inHeader->nOffset;

                bool signalError = false;
         
                ......
                // insert buffer size and time stamp
                mBufferSizes.add(inBufferLength[0]);
                .......
            } else {
                inBuffer[0] = inHeader->pBuffer + inHeader->nOffset;
                inBufferLength[0] = inHeader->nFilledLen;
                mLastInHeader = inHeader;
                updateTimeStamp(inHeader->nTimeStamp);
                mBufferSizes.add(inHeader->nFilledLen);
            }

            // Fill and decode
            bytesValid[0] = inBufferLength[0];

            INT prevSampleRate = mStreamInfo->sampleRate;
            INT prevNumChannels = mStreamInfo->numChannels;

            aacDecoder_Fill(mAACDecoder,
                            inBuffer,
                            inBufferLength,
                            bytesValid);

            // run DRC check
            mDrcWrap.submitStreamData(mStreamInfo);
            mDrcWrap.update();

            UINT inBufferUsedLength = inBufferLength[0] - bytesValid[0];
            inHeader->nFilledLen -= inBufferUsedLength;
            inHeader->nOffset += inBufferUsedLength;

            AAC_DECODER_ERROR decoderErr;
            int numLoops = 0;
            do {
                if (outputDelayRingBufferSpaceLeft() <
                        (mStreamInfo->frameSize * mStreamInfo->numChannels)) {
                    ALOGV("skipping decode: not enough space left in ringbuffer");
                    break;
                }

                int numConsumed = mStreamInfo->numTotalBytes;
                decoderErr = aacDecoder_DecodeFrame(mAACDecoder,
                                           tmpOutBuffer,
                                           2048 * MAX_CHANNEL_COUNT,
                                           0 /* flags */);

                numConsumed = mStreamInfo->numTotalBytes - numConsumed;
                numLoops++;

                mDecodedSizes.add(numConsumed);

                size_t numOutBytes =
                    mStreamInfo->frameSize * sizeof(int16_t) * mStreamInfo->numChannels;

            } while (decoderErr == AAC_DEC_OK);
        }
        while (!outQueue.empty()
                && outputDelayRingBufferSamplesAvailable()
                        >= mStreamInfo->frameSize * mStreamInfo->numChannels) {
            BufferInfo *outInfo = *outQueue.begin();
            OMX_BUFFERHEADERTYPE *outHeader = outInfo->mHeader;

            INT_PCM *outBuffer =
                    reinterpret_cast<INT_PCM *>(outHeader->pBuffer + outHeader->nOffset);
            int samplesize = mStreamInfo->numChannels * sizeof(int16_t);
            ......
            int available = outputDelayRingBufferSamplesAvailable();
            int numSamples = outHeader->nAllocLen / sizeof(int16_t);
            if (numSamples > available) {
                numSamples = available;
            }
            int64_t currentTime = 0;
            if (available) {

                int numFrames = numSamples / (mStreamInfo->frameSize * mStreamInfo->numChannels);
                numSamples = numFrames * (mStreamInfo->frameSize * mStreamInfo->numChannels);

                ALOGV("%d samples available (%d), or %d frames",
                        numSamples, available, numFrames);
                int64_t *nextTimeStamp = &mBufferTimestamps.editItemAt(0);
                currentTime = *nextTimeStamp;
                int32_t *currentBufLeft = &mBufferSizes.editItemAt(0);
                for (int i = 0; i < numFrames; i++) {
                    int32_t decodedSize = mDecodedSizes.itemAt(0);
                    mDecodedSizes.removeAt(0);
                    ALOGV("decoded %d of %d", decodedSize, *currentBufLeft);
                    if (*currentBufLeft > decodedSize) {
                        // adjust/interpolate next time stamp
                        *currentBufLeft -= decodedSize;
                        *nextTimeStamp += mStreamInfo->aacSamplesPerFrame *
                                1000000ll / mStreamInfo->aacSampleRate;
                        mNextOutBufferTimeUs = *nextTimeStamp;
                        ALOGV("adjusted nextTimeStamp/size to %lld/%d",
                                (long long) *nextTimeStamp, *currentBufLeft);
                    } else {
                        // move to next timestamp in list
                        if (mBufferTimestamps.size() > 0) {
                            mBufferTimestamps.removeAt(0);
                            nextTimeStamp = &mBufferTimestamps.editItemAt(0);
                            mNextOutBufferTimeUs = *nextTimeStamp;
                            mBufferSizes.removeAt(0);
                            currentBufLeft = &mBufferSizes.editItemAt(0);
                            ALOGV("moved to next time/size: %lld/%d",
                                    (long long) *nextTimeStamp, *currentBufLeft);
                        }
                        numFrames = i + 1;
                        numSamples = numFrames * mStreamInfo->frameSize * mStreamInfo->numChannels;
                        break;
                    }
                }

                ALOGV("getting %d from ringbuffer", numSamples);
                int32_t ns = outputDelayRingBufferGetSamples(outBuffer, numSamples);
            }

            outHeader->nFilledLen = numSamples * sizeof(int16_t);


            outHeader->nTimeStamp = currentTime;

            mOutputBufferCount++;
            outInfo->mOwnedByUs = false;
            outQueue.erase(outQueue.begin());
            outInfo = NULL;
            ALOGV("out timestamp %lld / %d", outHeader->nTimeStamp, outHeader->nFilledLen);
            notifyFillBufferDone(outHeader);
            outHeader = NULL;
        }

        if (mEndOfInput) {
            int ringBufAvail = outputDelayRingBufferSamplesAvailable();
            if (!outQueue.empty()
                    && ringBufAvail < mStreamInfo->frameSize * mStreamInfo->numChannels) {
                if (!mEndOfOutput) {
                    // send partial or empty block signaling EOS
                    mEndOfOutput = true;
                    BufferInfo *outInfo = *outQueue.begin();
                    OMX_BUFFERHEADERTYPE *outHeader = outInfo->mHeader;

                    INT_PCM *outBuffer = reinterpret_cast<INT_PCM *>(outHeader->pBuffer
                            + outHeader->nOffset);
                    int32_t ns = outputDelayRingBufferGetSamples(outBuffer, ringBufAvail);
                    if (ns < 0) {
                        ns = 0;
                    }
                    outHeader->nFilledLen = ns;
                    outHeader->nFlags = OMX_BUFFERFLAG_EOS;

                    outHeader->nTimeStamp = mBufferTimestamps.itemAt(0);
                    mBufferTimestamps.clear();
                    mBufferSizes.clear();
                    mDecodedSizes.clear();

                    mOutputBufferCount++;
                    outInfo->mOwnedByUs = false;
                    outQueue.erase(outQueue.begin());
                    outInfo = NULL;
                    notifyFillBufferDone(outHeader);
                    outHeader = NULL;
                }
                break; // if outQueue not empty but no more output
            }
        }
    }
}
```


这段代码主要作用是将数据inBuffer送到解码库libFraunhoferAAC解码（aacDecoder_Fill）
从Log可以看出此音频是属于adts 格式的：
``` cpp
12-25 23:12:00.161: V/NuPlayerDecoder(795): [decoder] onMessage: AMessage(what = 'conf', target = 28) = {
12-25 23:12:00.161: V/NuPlayerDecoder(795):   AMessage format = AMessage(what = 0x00000000) = {
12-25 23:12:00.161: V/NuPlayerDecoder(795):                       string mime = "audio/mp4a-latm"
12-25 23:12:00.161: V/NuPlayerDecoder(795):                       int64_t durationUs = 9936000
12-25 23:12:00.161: V/NuPlayerDecoder(795):                       int32_t channel-count = 1
12-25 23:12:00.161: V/NuPlayerDecoder(795):                       int32_t sample-rate = 44100
12-25 23:12:00.161: V/NuPlayerDecoder(795):                       int32_t is-adts = 1
12-25 23:12:00.161: V/NuPlayerDecoder(795):                       int32_t aac-profile = 2
12-25 23:12:00.161: V/NuPlayerDecoder(795):                       int32_t max-input-size = 262144
12-25 23:12:00.161: V/NuPlayerDecoder(795):                       Buffer csd-0 = {
12-25 23:12:00.161: V/NuPlayerDecoder(795):                         00000000:  12 08                                             ..
12-25 23:12:00.161: V/NuPlayerDecoder(795):                       }
12-25 23:12:00.161: V/NuPlayerDecoder(795):                       string file-format = "audio/aac"
12-25 23:12:00.161: V/NuPlayerDecoder(795):                       int32_t aac-format-adif = 0
12-25 23:12:00.161: V/NuPlayerDecoder(795):                       int32_t is-byte-stream-mode = 1
12-25 23:12:00.161: V/NuPlayerDecoder(795):                       int32_t priority = 0
12-25 23:12:00.161: V/NuPlayerDecoder(795):                       int32_t pcm-encoding = 2
12-25 23:12:00.161: V/NuPlayerDecoder(795):                     }
12-25 23:12:00.161: V/NuPlayerDecoder(795): }
```

然后循环解码数据（aacDecoder_DecodeFrame），最后数据解码完成后通知IOMX（notifyFillBufferDone(outHeader)）。
Log:
``` cpp
12-25 23:12:00.420: V/SoftAAC2(785): inHeader->nFilledLen = 50360
12-25 23:12:00.420: V/SoftAAC2(785): 1024 samples available (1387), or 1 frames
12-25 23:12:00.420: V/SoftAAC2(785): decoded 115 of 115
12-25 23:12:00.420: V/SoftAAC2(785): moved to next time/size: 116095/112
12-25 23:12:00.420: V/SoftAAC2(785): getting 1024 from ringbuffer
12-25 23:12:00.420: V/SoftAAC2(785): out timestamp 92876 / 2048
```
最后经过NuPlayerRenderer的onDrainAudioQueue函数经由AudioSink 将数据写入到底层，最后经由硬件播放声音。

``` cpp
12-25 23:12:08.293: V/NuPlayerRenderer(795): onDrainAudioQueue: rendering audio at media time 8.36 secs
12-25 23:12:08.294: V/NuPlayerRenderer(795): AudioSink write short frame count 296 < 2048
```

#### （三）、Android AAC音频 编码
首先看看时序图：
![Alt text | center](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/android.codecs/VS7-05-AAC-Encord-flow.png)

编码：

``` cpp
Z:\MediaLearing\frameworks\av\media\libstagefright\codecs\aacenc\SoftAACEncoder2.cpp

void SoftAACEncoder2::onQueueFilled(OMX_U32 /* portIndex */) {
    .......
    List<BufferInfo *> &inQueue = getPortQueue(0);
    List<BufferInfo *> &outQueue = getPortQueue(1);

    if (!mSentCodecSpecificData) {
        ......
        if (AACENC_OK != aacEncEncode(mAACEncoder, NULL, NULL, NULL, NULL)) {
        }

        OMX_U32 actualBitRate  = aacEncoder_GetParam(mAACEncoder, AACENC_BITRATE);
        ......

        AACENC_InfoStruct encInfo;
        if (AACENC_OK != aacEncInfo(mAACEncoder, &encInfo)) {
        }

        BufferInfo *outInfo = *outQueue.begin();
        OMX_BUFFERHEADERTYPE *outHeader = outInfo->mHeader;

        if (outHeader->nOffset + encInfo.confSize > outHeader->nAllocLen) {
        }

        outHeader->nFilledLen = encInfo.confSize;
        outHeader->nFlags = OMX_BUFFERFLAG_CODECCONFIG;

        uint8_t *out = outHeader->pBuffer + outHeader->nOffset;
        memcpy(out, encInfo.confBuf, encInfo.confSize);

        outQueue.erase(outQueue.begin());
        outInfo->mOwnedByUs = false;
        notifyFillBufferDone(outHeader);

        mSentCodecSpecificData = true;
    }

    size_t numBytesPerInputFrame =
        mNumChannels * kNumSamplesPerFrame * sizeof(int16_t);

    for (;;) {
        // We do the following until we run out of buffers.

        while (mInputSize < numBytesPerInputFrame) {
            ......
            BufferInfo *inInfo = *inQueue.begin();
            OMX_BUFFERHEADERTYPE *inHeader = inInfo->mHeader;

            const void *inData = inHeader->pBuffer + inHeader->nOffset;

            size_t copy = numBytesPerInputFrame - mInputSize;
            if (copy > inHeader->nFilledLen) {
                copy = inHeader->nFilledLen;
            }
            ......

            memcpy((uint8_t *)mInputFrame + mInputSize, inData, copy);
            mInputSize += copy;

            inHeader->nOffset += copy;
            inHeader->nFilledLen -= copy;

            // "Time" on the input buffer has in effect advanced by the
            // number of audio frames we just advanced nOffset by.
            inHeader->nTimeStamp +=
                (copy * 1000000ll / mSampleRate)
                    / (mNumChannels * sizeof(int16_t));

            if (inHeader->nFilledLen == 0) {
                if (inHeader->nFlags & OMX_BUFFERFLAG_EOS) {
                    mSawInputEOS = true;

                    // Pad any remaining data with zeroes.
                    memset((uint8_t *)mInputFrame + mInputSize,
                           0,
                           numBytesPerInputFrame - mInputSize);

                    mInputSize = numBytesPerInputFrame;
                }

                inQueue.erase(inQueue.begin());
                inInfo->mOwnedByUs = false;
                notifyEmptyBufferDone(inHeader);

                inData = NULL;
                inHeader = NULL;
                inInfo = NULL;
            }
        }

        BufferInfo *outInfo = *outQueue.begin();
        OMX_BUFFERHEADERTYPE *outHeader = outInfo->mHeader;

        uint8_t *outPtr = (uint8_t *)outHeader->pBuffer + outHeader->nOffset;
        size_t outAvailable = outHeader->nAllocLen - outHeader->nOffset;

        AACENC_InArgs inargs;
        AACENC_OutArgs outargs;
        memset(&inargs, 0, sizeof(inargs));
        memset(&outargs, 0, sizeof(outargs));
        inargs.numInSamples = numBytesPerInputFrame / sizeof(int16_t);

        void* inBuffer[]        = { (unsigned char *)mInputFrame };
        INT   inBufferIds[]     = { IN_AUDIO_DATA };
        INT   inBufferSize[]    = { (INT)numBytesPerInputFrame };
        INT   inBufferElSize[]  = { sizeof(int16_t) };

        AACENC_BufDesc inBufDesc;
        inBufDesc.numBufs           = sizeof(inBuffer) / sizeof(void*);
        inBufDesc.bufs              = (void**)&inBuffer;
        inBufDesc.bufferIdentifiers = inBufferIds;
        inBufDesc.bufSizes          = inBufferSize;
        inBufDesc.bufElSizes        = inBufferElSize;

        void* outBuffer[]       = { outPtr };
        INT   outBufferIds[]    = { OUT_BITSTREAM_DATA };
        INT   outBufferSize[]   = { 0 };
        INT   outBufferElSize[] = { sizeof(UCHAR) };

        AACENC_BufDesc outBufDesc;
        outBufDesc.numBufs           = sizeof(outBuffer) / sizeof(void*);
        outBufDesc.bufs              = (void**)&outBuffer;
        outBufDesc.bufferIdentifiers = outBufferIds;
        outBufDesc.bufSizes          = outBufferSize;
        outBufDesc.bufElSizes        = outBufferElSize;

        // Encode the mInputFrame, which is treated as a modulo buffer
        AACENC_ERROR encoderErr = AACENC_OK;
        size_t nOutputBytes = 0;

        do {
            memset(&outargs, 0, sizeof(outargs));

            outBuffer[0] = outPtr;
            outBufferSize[0] = outAvailable - nOutputBytes;

            encoderErr = aacEncEncode(mAACEncoder,
                                      &inBufDesc,
                                      &outBufDesc,
                                      &inargs,
                                      &outargs);

            ......
        } while (encoderErr == AACENC_OK && inargs.numInSamples > 0);

        outHeader->nFilledLen = nOutputBytes;

        outHeader->nFlags = OMX_BUFFERFLAG_ENDOFFRAME;

      ......
        outHeader->nTimeStamp = mInputTimeUs;
        outQueue.erase(outQueue.begin());
        outInfo->mOwnedByUs = false;
        notifyFillBufferDone(outHeader);

        outHeader = NULL;
        outInfo = NULL;

        mInputSize = 0;
    }
}

```
这段代码主要使用aacEncEncode函数来对原始音频数据编码：

``` cpp
Z:\MediaLearing\external\aac\libAACenc\src\aacenc_lib.cpp

AACENC_ERROR aacEncEncode(
        const HANDLE_AACENCODER   hAacEncoder,
        const AACENC_BufDesc     *inBufDesc,
        const AACENC_BufDesc     *outBufDesc,
        const AACENC_InArgs      *inargs,
        AACENC_OutArgs           *outargs
        )
{
    AACENC_ERROR err = AACENC_OK;
    INT i, nBsBytes = 0;
    INT  outBytes[(1)];
    int  nExtensions = 0;
    int  ancDataExtIdx = -1;
    ......
    if (hAacEncoder->InitFlags!=0) {

        err = aacEncInit(hAacEncoder,
                         hAacEncoder->InitFlags,
                        &hAacEncoder->extParam);
        hAacEncoder->InitFlags = AACENC_INIT_NONE;
    }
    
    ......
    /*
     * Manage incoming audio samples.
     */
    if ( (inargs->numInSamples > 0) && (getBufDescIdx(inBufDesc,IN_AUDIO_DATA) != -1) )
    {
        /* Fetch data until nSamplesToRead reached */
        INT idx = getBufDescIdx(inBufDesc,IN_AUDIO_DATA);
        INT newSamples = fixMax(0,fixMin(inargs->numInSamples, hAacEncoder->nSamplesToRead-hAacEncoder->nSamplesRead));
        INT_PCM *pIn = hAacEncoder->inputBuffer+hAacEncoder->inputBufferOffset+hAacEncoder->nSamplesRead;

        /* Copy new input samples to internal buffer */
        if (inBufDesc->bufElSizes[idx]==(INT)sizeof(INT_PCM)) {
            FDKmemcpy(pIn, (INT_PCM*)inBufDesc->bufs[idx], newSamples*sizeof(INT_PCM));  /* Fast copy. */
        }
        else if (inBufDesc->bufElSizes[idx]>(INT)sizeof(INT_PCM)) {
            for (i=0; i<newSamples; i++) {
                pIn[i] = (INT_PCM)(((LONG*)inBufDesc->bufs[idx])[i]>>16);                /* Convert 32 to 16 bit. */
            }
        }
        else {
            for (i=0; i<newSamples; i++) {
                pIn[i] = ((INT_PCM)(((SHORT*)inBufDesc->bufs[idx])[i]))<<16;             /* Convert 16 to 32 bit. */
            }
        }
        hAacEncoder->nSamplesRead += newSamples;

        /* Number of fetched input buffer samples. */
        outargs->numInSamples = newSamples;
    }

    /* input buffer completely filled ? */
    if (hAacEncoder->nSamplesRead < hAacEncoder->nSamplesToRead)
    {
        /* - eof reached and flushing enabled, or
           - return to main and wait for further incoming audio samples */
        if (inargs->numInSamples==-1)
        {
            if ( (hAacEncoder->nZerosAppended < hAacEncoder->nDelay)
                )
            {
              int nZeros = hAacEncoder->nSamplesToRead - hAacEncoder->nSamplesRead;

              FDK_ASSERT(nZeros >= 0);

              /* clear out until end-of-buffer */
              if (nZeros) {
                FDKmemclear(hAacEncoder->inputBuffer+hAacEncoder->inputBufferOffset+hAacEncoder->nSamplesRead, sizeof(INT_PCM)*nZeros );
                hAacEncoder->nZerosAppended += nZeros;
                hAacEncoder->nSamplesRead = hAacEncoder->nSamplesToRead;
              }
            }
            else { /* flushing completed */
              err = AACENC_ENCODE_EOF; /* eof reached */
              goto bail;
            }
        }
        else { /* inargs->numInSamples!= -1 */
            goto bail; /* not enough samples in input buffer and no flushing enabled */
        }
    }

    /* init payload */
    FDKmemclear(hAacEncoder->extPayload, sizeof(AACENC_EXT_PAYLOAD) * MAX_TOTAL_EXT_PAYLOADS);
    for (i = 0; i < MAX_TOTAL_EXT_PAYLOADS; i++) {
      hAacEncoder->extPayload[i].associatedChElement = -1;
    }
    FDKmemclear(hAacEncoder->extPayloadData, sizeof(hAacEncoder->extPayloadData));
    FDKmemclear(hAacEncoder->extPayloadSize, sizeof(hAacEncoder->extPayloadSize));


    /*
     * Calculate Meta Data info.
     */
    if ( (hAacEncoder->hMetadataEnc!=NULL) && (hAacEncoder->metaDataAllowed!=0) ) {

        const AACENC_MetaData *pMetaData = NULL;
        AACENC_EXT_PAYLOAD *pMetaDataExtPayload = NULL;
        UINT nMetaDataExtensions = 0;
        INT  matrix_mixdown_idx = 0;

        /* New meta data info available ? */
        if ( getBufDescIdx(inBufDesc,IN_METADATA_SETUP) != -1 ) {
          pMetaData = (AACENC_MetaData*)inBufDesc->bufs[getBufDescIdx(inBufDesc,IN_METADATA_SETUP)];
        }

        FDK_MetadataEnc_Process(hAacEncoder->hMetadataEnc,
                                hAacEncoder->inputBuffer+hAacEncoder->inputBufferOffset,
                                hAacEncoder->nSamplesRead,
                                pMetaData,
                               &pMetaDataExtPayload,
                               &nMetaDataExtensions,
                               &matrix_mixdown_idx
                                );

        for (i=0; i<(INT)nMetaDataExtensions; i++) {  /* Get meta data extension payload. */
            hAacEncoder->extPayload[nExtensions++] = pMetaDataExtPayload[i];
        }

        if ( (matrix_mixdown_idx!=-1)
          && ((hAacEncoder->extParam.userChannelMode==MODE_1_2_2)||(hAacEncoder->extParam.userChannelMode==MODE_1_2_2_1)) )
        {
          /* Set matrix mixdown coefficient. */
          UINT pceValue = (UINT)( (0<<3) | ((matrix_mixdown_idx&0x3)<<1) | 1 );
          if (hAacEncoder->extParam.userPceAdditions != pceValue) {
            hAacEncoder->extParam.userPceAdditions = pceValue;
            hAacEncoder->InitFlags |= AACENC_INIT_TRANSPORT;
          }
        }
    }


    if ( isSbrActive(&hAacEncoder->aacConfig) ) {

        INT nPayload = 0;

        /*
         * Encode SBR data.
         */
        if (sbrEncoder_EncodeFrame(hAacEncoder->hEnvEnc,
                                   hAacEncoder->inputBuffer,
                                   hAacEncoder->extParam.nChannels,
                                   hAacEncoder->extPayloadSize[nPayload],
                                   hAacEncoder->extPayloadData[nPayload]
#if defined(EVAL_PACKAGE_SILENCE) || defined(EVAL_PACKAGE_SBR_SILENCE)
                                  ,hAacEncoder->hAacEnc->clearOutput
#endif
                                  ))
        {
            err = AACENC_ENCODE_ERROR;
            goto bail;
        }
        else {
            /* Add SBR extension payload */
            for (i = 0; i < (8); i++) {
                if (hAacEncoder->extPayloadSize[nPayload][i] > 0) {
                    hAacEncoder->extPayload[nExtensions].pData    = hAacEncoder->extPayloadData[nPayload][i];
                    {
                      hAacEncoder->extPayload[nExtensions].dataSize = hAacEncoder->extPayloadSize[nPayload][i];
                      hAacEncoder->extPayload[nExtensions].associatedChElement = i;
                    }
                    hAacEncoder->extPayload[nExtensions].dataType = EXT_SBR_DATA;  /* Once SBR Encoder supports SBR CRC set EXT_SBR_DATA_CRC */
                    nExtensions++;                                                 /* or EXT_SBR_DATA according to configuration. */
                    FDK_ASSERT(nExtensions<=MAX_TOTAL_EXT_PAYLOADS);
                }
            }
            nPayload++;
        }
    } /* sbrEnabled */

    if ( (inargs->numAncBytes > 0) && ( getBufDescIdx(inBufDesc,IN_ANCILLRY_DATA)!=-1 ) ) {
        INT idx = getBufDescIdx(inBufDesc,IN_ANCILLRY_DATA);
        hAacEncoder->extPayload[nExtensions].dataSize = inargs->numAncBytes * 8;
        hAacEncoder->extPayload[nExtensions].pData    = (UCHAR*)inBufDesc->bufs[idx];
        hAacEncoder->extPayload[nExtensions].dataType = EXT_DATA_ELEMENT;
        hAacEncoder->extPayload[nExtensions].associatedChElement = -1;
        ancDataExtIdx = nExtensions; /* store index */
        nExtensions++;
    }

    /*
     * Encode AAC - Core.
     */
    if ( FDKaacEnc_EncodeFrame( hAacEncoder->hAacEnc,
                                hAacEncoder->hTpEnc,
                                hAacEncoder->inputBuffer,
                                outBytes,
                                hAacEncoder->extPayload
                                ) != AAC_ENC_OK )
    {
        err = AACENC_ENCODE_ERROR;
        goto bail;
    }

    if (ancDataExtIdx >= 0) {
      outargs->numAncBytes = inargs->numAncBytes - (hAacEncoder->extPayload[ancDataExtIdx].dataSize>>3);
    }

    /* samples exhausted */
    hAacEncoder->nSamplesRead -= hAacEncoder->nSamplesToRead;

    /*
     * Delay balancing buffer handling
     */
    if (isSbrActive(&hAacEncoder->aacConfig)) {
        sbrEncoder_UpdateBuffers(hAacEncoder->hEnvEnc, hAacEncoder->inputBuffer);
    }

    /*
     * Make bitstream public
     */
    if (outBufDesc->numBufs>=1) {

        INT bsIdx = getBufDescIdx(outBufDesc,OUT_BITSTREAM_DATA);
        INT auIdx = getBufDescIdx(outBufDesc,OUT_AU_SIZES);

        for (i=0,nBsBytes=0; i<hAacEncoder->aacConfig.nSubFrames; i++) {
          nBsBytes += outBytes[i];

          if (auIdx!=-1) {
           ((INT*)outBufDesc->bufs[auIdx])[i] = outBytes[i];
          }
        }

        if ( (bsIdx!=-1) && (outBufDesc->bufSizes[bsIdx]>=nBsBytes) ) {
          FDKmemcpy(outBufDesc->bufs[bsIdx], hAacEncoder->outBuffer, sizeof(UCHAR)*nBsBytes);
          outargs->numOutBytes = nBsBytes;
        }
        else {
          /* output buffer too small, can't write valid bitstream */
          err = AACENC_ENCODE_ERROR;
          goto bail;
        }
    }
    
    return err;
}
```

进一步调用了sbrEncoder_EncodeFrame  和  FDKaacEnc_EncodeFrame 进行编码。
#### （四）、libFraunhoferAAC库分析
**Todo---**

> \external\aac（libFraunhoferAAC 库）
- libAACdec
- libAACenc
- libFDK
- documentation\aacDecoder.pdf
- documentation\aacEncoder.pdf
#### （五）、参考资料(特别感谢各位前辈的分析和图示)：
[AAC音频格式分析与解码](https://www.cnblogs.com/caosiyang/archive/2012/07/16/2594029.html)
[MediaCodec进行编解码AAC（文件格式转换）](https://blog.csdn.net/Ch97CKd/article/details/79091673)
[AAC 文件解析及解码流程](https://blog.csdn.net/wlsfling/article/details/5876016)
[[原创]桓泽学音频编解码（3）：AAC 系统算法分析](http://www.cnblogs.com/gaozehua/archive/2012/05/03/2479960.html)
