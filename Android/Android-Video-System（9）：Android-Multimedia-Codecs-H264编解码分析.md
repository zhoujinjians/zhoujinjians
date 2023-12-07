---
title: Android Video System（9）：Android Multimedia Codecs - H264编解码分析
cover: https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/hexo.themes/bing-wallpaper-2018.04.31.jpg
categories:
  - Multimedia
tags:
  - Android
  - Video
  - Multimedia
toc: true
abbrlink: 20190216
date: 2019-02-16 09:25:00
---

--------------------------------------------------------------------------------

注：文章都是通过阅读各位前辈总结的资料 Android 8.x && Linux（kernel 4.x）Qualcomm平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网（◕‿◕），如有侵权，请联系删除，禁止转载（©Qualcomm ©Android @Linux 版权所有），谢谢。


首先感谢：

[【从零了解H264结构】](http://www.iosxxx.com/blog/2017-08-09-%E4%BB%8E%E9%9B%B6%E4%BA%86%E8%A7%A3H264%E7%BB%93%E6%9E%84.html)

正是由于前人的分析和总结，帮助我节约了大量的时间和精力，再次感谢！！！

Google Pixel、Pixel XL 内核代码（==**文章基于 Kernel-3.x**==）：
 [Kernel source for Pixel 2 (walleye) and Pixel 2 XL (taimen) - GitHub](https://github.com/nathanchance/wahoo)

AOSP 源码（==**文章基于 Android 7.x**==）：
 [ Android 系统全套源代码分享 (更新到 8.1.0_r1)](https://testerhome.com/topics/2229)

--------------------------------------------------------------------------------

``` cpp
\frameworks\av\media\libstagefright\omx\SoftOMXPlugin.cpp
kComponents[] = {
    ......
    { "OMX.google.h264.decoder", "avcdec", "video_decoder.avc" },
    { "OMX.google.h264.encoder", "avcenc", "video_encoder.avc" },
    ......
```

==源码（部分）==：

>PATH : \frameworks\av\media\libstagefright\codecs （Android Codecs）

- avc
- avcdec
- avcenc

>\external\libavc（libavc 库）

- encoder
- decoder
- common

>hardware\qcom\media（Qcom）

- mm-core
- libstagefrighthw
- mm-video-v4l2

Log：
[H264-Decorder-Google-log.md](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/android.codecs/H264-Decorder-Google-log.md)
[H264-Encorder-QCom-log.md](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/android.codecs/H264-Encorder-QCom-log.md)


--------------------------------------------------------------------------------

#### （一）、从零了解H264结构（概览）
##### 1.0、前言
为什么需要编码呢？比如当前屏幕是1280*720.一秒24张图片.那么我们一秒的视频数据是

> 1280*720(位像素)*24(张) / 8(1字节8位)(结果:B) / 1024(结果:KB) / 1024 (结果:MB) = 2.64MB

一秒的数据有2.64MB数据量。1分钟就会有100多MB。这对用户来说真心是灾难。所以现在我们需要一种压缩方式减小数据的大小.在更低 比特率(bps)的情况下依然提供清晰的视频。
H264: H264/AVC是广泛采用的一种编码方式。我们这边会带大家了解。从大到小排序依次是 序列，图像，片组，片，NALU，宏块，亚宏块，块，像素。
##### 1.1、原理
H.264原始码流(裸流)是由一个接一个NALU组成，它的功能分为两层，VCL(视频编码层)和 NAL(网络提取层).

> VCL(Video Coding Layer) + NAL(Network Abstraction Layer).

1.VCL：包括核心压缩引擎和块，宏块和片的语法级别定义，设计目标是尽可能地独立于网络进行高效的编码；
2.NAL：负责将VCL产生的比特字符串适配到各种各样的网络和多元环境中，覆盖了所有片级以上的语法级别。

在VCL进行数据传输或存储之前，这些编码的VCL数据，被映射或封装进NAL单元。（NALU）。

> 一个NALU = 一组对应于视频编码的NALU头部信息 + 一个原始字节序列负荷(RBSP,Raw Byte Sequence Payload).

如图所示，下图中的NALU的头 + RBSP 就相当于一个NALU(Nal Unit),每个单元都按独立的NALU传送。H.264的结构全部都是以NALU为主，理解了NALU，就理解了H.264的结构。
一个原始的H.264 NALU 单元常由 [StartCode] [NALU Header] [NALU Payload] 三部分组成，其中 Start Code 用于标示这是一个NALU 单元的开始，必须是”00 00 00 01” 或”00 00 01”
![Alt text | center](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/android.codecs/VS9-01-H264 NALU headerRBSP.png)

##### 1.1.1. NAL Header
由三部分组成，forbidden_bit(1bit)，nal_reference_bit(2bits)（优先级），nal_unit_type(5bits)（类型）。

![Alt text | center](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/android.codecs/VS9-02-H264.NAL Header.png)

举例来说：


> 00 00 00 01 06:  SEI信息    
> 00 00 00 01 67:  0x67&0x1f = 0x07 :SPS 
> 00 00 00 01 68:  0x68&0x1f = 0x08 :PPS 
> 00 00 00 01 65:  0x65&0x1f = 0x05: IDR Slice

##### 1.1.2. RBSP
![Alt text | center](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/android.codecs/VS9-03-H264.RBSP.png)

图 6.69 RBSP 序列举例

![Alt text | center](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/android.codecs/VS9-04-H264.RBSP.describle.png)


**SODB与RBSP**
SODB 数据比特串 -> 是编码后的原始数据.
RBSP 原始字节序列载荷 -> 在原始编码数据的后面添加了 结尾比特。一个 bit“1”若干比特“0”，以便字节对齐。
![Alt text | center](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/android.codecs/VS9-05-H264.RBSP.SODB.png)


##### 1.2、从NALU出发了解H.264里面的专业词语

![Alt text | center](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/android.codecs/VS9-06-H264.Slice.Layer.png)


> 1帧 = n个片 
> 1片 = n个宏块 
> 1宏块 = 16x16yuv数据

##### 1.2.1. Slice(片)
如图所示，NALU的主体中包含了Slice(片).

> 一个片 = Slice Header + Slice Data

片是H.264提出的新概念，通过编码图片后切分通过高效的方式整合出来的概念。一张图片有一个或者多个片，而片由NALU装载并进行网络传输的。但是NALU不一定是切片，这是充分不必要条件，因为 NALU 还有可能装载着其他用作描述视频的信息.

那么为什么要设置片呢?
设置片的目的是为了限制误码的扩散和传输，应使编码片相互间是独立的。某片的预测不能以其他片中的宏块为参考图像，这样某一片中的预测误差才不会传播到其他片中。

可以看到上图中，每个图像中，若干宏块(Macroblock)被排列成片。一个视频图像可编程一个或更多个片，每片包含整数个宏块 (MB),每片至少包含一个宏块。
片有一下五种类型:


![Alt text | center](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/android.codecs/VS9-07-H264.I.P.B..png)


##### 1.2.2. 宏块(Macroblock)
刚才在片中提到了宏块.那么什么是宏块呢？
宏块是视频信息的主要承载者。一个编码图像通常划分为多个宏块组成.包含着每一个像素的亮度和色度信息。视频解码最主要的工作则是提供高效的方式从码流中获得宏块中像素阵列。

> 一个宏块 = 一个16*16的亮度像素 + 一个8×8Cb + 一个8×8Cr彩色像素块组成。(YCbCr 是属于 YUV
> 家族的一员,在YCbCr 中 Y 是指亮度分量，Cb 指蓝色色度分量，而 Cr 指红色色度分量)

![Alt text | center](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/android.codecs/VS9-08-H264.Macroblock.png)

分层结构,在 H.264 中，句法元素共被组织成 序列、图像、片、宏块、子宏块五个层次。
句法元素的分层结构有助于更有效地节省码流。例如，再一个图像中，经常会在各个片之间有相同的数据，如果每个片都同时携带这些数据，势必会造成码流的浪费。更为有效的做法是将该图像的公共信息抽取出来，形成图像一级的句法元素，而在片级只携带该片自身独有的句法元素。

![Alt text | center](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/android.codecs/VS9-09-H264.slice.header.data.png)

![Alt text | center](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/android.codecs/VS9-10-H264.slice.header.data-detail.png)

![Alt text | center](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/android.codecs/VS9-11-H264.Macroblock_type.png.png)


##### 1.2.3. 图像,场和帧
图像是个集合概念，顶 场、底场、帧都可以称为图像。对于H.264 协议来说，我们平常所熟悉的那些称呼，例如： I 帧、P 帧、B帧等等，实际上都是我们把图像这个概念具体化和细小化了。我们 在 H.264里提到的“帧”通常就是指不分场的图像；

视频的一场或一帧可用来产生一个编码图像。一帧通常是一个完整的图像。当采集视频信号时，如果采用隔行扫描(奇.偶数行),则扫描下来的一帧图像就被分为了两个部分,这每一部分就被称为 [场],根据次序氛围: [顶场] 和 [底场]。

![Alt text | center](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/android.codecs/VS9-12-H264.zuoyongyu.png)
##### 1.2.4. I,P,B帧与pts/dts

![Alt text | center](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/android.codecs/VS9-13-H264.PTS.DTS.png)

DTS与PTS的不同:
DTS主要用户视频的解码，在解码阶段使用。PTS主要用于视频的同步和输出，在display的时候使用。再没有B frame的时候输出顺序一样。

##### 1.2.5. GOP
![Alt text | center](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/android.codecs/VS9-14-H264.GOP.png)

GOP是画面组，一个GOP是一组连续的画面。
GOP一般有两个数字，如M=3，N=12.M制定I帧与P帧之间的距离，N指定两个I帧之间的距离。那么现在的GOP结构是

> I BBP BBP BBP BB I

增大图片组能有效的减少编码后的视频体积，但是也会降低视频质量，至于怎么取舍，得看需求了

##### 1.2.6 . IDR
一个序列的第一个图像叫做 IDR 图像（立即刷新图像），IDR 图像都是 I 帧图像。
I和IDR帧都使用帧内预测。I帧不用参考任何帧，但是之后的P帧和B帧是有可能参考这个I帧之前的帧的。IDR就不允许这样。
比如这种情况:
IDR1 P4 B2 B3 P7 B5 B6 I10 B8 B9 P13 B11 B12 P16 B14 B15 这里的B8可以跨过I10去参考P7

核心作用：
H.264 引入 IDR 图像是为了解码的重同步，当解码器解码到 IDR 图像时，立即将参考帧队列清空，将已解码的数据全部输出或抛弃，重新查找参数集，开始一个新的序列。这样，如果前一个序列出现重大错误，在这里可以获得重新同步的机会。IDR图像之后的图像永远不会使用IDR之前的图像的数据来解码。


##### 1.3. 帧内预测和帧间预测

##### 1.3. 1. 帧内预测（也叫帧内压缩）

![Alt text | center](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/android.codecs/VS9-15-H264.zenjianyuce.png)

我们可以通过第 1、2、3、4、5 块的编码来推测和计算第 6 块的编码，因此就不需要对第 6 块进行编码了，从而压缩了第 6 块，节省了空间

##### 1.3. 2. 帧间预测（也叫帧间压缩）

![Alt text | center](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/android.codecs/VS9-16-H264.zenjianyasuo.png)

可以看到前后两帧的差异其实是很小的，这时候用帧间压缩就很有意义。
这里涉及到几个重要的概念：块匹配，残差，运动搜索(运动估计),运动补偿.

帧间压缩最常用的方式就是块匹配(Block Matching)。找找看前面已经编码的几帧里面，和我当前这个块最类似的一个块，这样我不用编码当前块的内容了，只需要编码当前块和我找到的快的差异(残差)即可。找最想的块的过程叫运动搜索(Motion Search),又叫运动估计。用残差和原来的块就能推算出当前块的过程叫运动补偿(Motion Compensation).


#### （二）、Android  H264编码

![Alt text | center](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/android.codecs/VS9-17-H264-Encoder-google.png)

由于博主的手机为Qcom平台，支持h264编解码，博主保留了Qcom h264 Encoder ，将Decoder使用Google的，但分析主要还是以Google的H264 Encoder/Decoder 为主。
``` xml
[/system/etc/media_codecs.xml]
<!--
  Decoder capabilities for thorium
   _________________________________________________________________________
  | Codec    | W       H       fps     Mbps    MB/s    | Encode Secure-dec |
  |__________|_________________________________________|___________________|
  | h264     | 1920    1088    30      20      244800  |  Y       N        |
  | hevc     | 1920    1088    30      20      244800  |  N       N        |
  | mpeg4    | 1920    1088    30      6       244800  |  Y       N        |
  | vp8      | 1920    1088    30      20      244800  |  N       N        |
  | div4/5/6 | 1920    1088    30      6       244800  |  N       N        |
  | h263     |  864     480    30      2       48600   |  Y       N        |
  |__________|_________________________________________|___________________|

-->

<!--
  Encoder capabilities for thorium
   ____________________________________________________
  | Codec    | W       H       fps     Mbps    MB/s    |
  |__________|_________________________________________|
  | h264     | 1920    1088    30      20      244800  |
  | mpeg4    | 864      480    30       2       48600  |
  | h263     | 864      480    30       2       48600  |
  |____________________________________________________|
-->

<MediaCodecs>
    <Include href="media_codecs_google_audio.xml" />
    <Include href="media_codecs_google_telephony.xml" />
    <Settings>
        <Setting name="max-video-encoder-input-buffers" value="9" />
    </Settings>
    <Encoders>
        <!-- Video Hardware  -->
        <MediaCodec name="OMX.qcom.video.encoder.avc" type="video/avc" >
            <Quirk name="requires-allocate-on-input-ports" />
            <Quirk name="requires-allocate-on-output-ports" />
            <Quirk name="requires-loaded-to-idle-after-allocation" />
            <Limit name="size" min="96x96" max="1920x1088" />
            <Limit name="alignment" value="2x2" />
            <Limit name="block-size" value="16x16" />
            <Limit name="blocks-per-second" min="1" max="244800" />
            <Limit name="bitrate" range="1-20000000" />
            <Limit name="concurrent-instances" max="8" />
            <Feature name="can-swap-width-height" />
        </MediaCodec>
    </Encoders>
    <Decoders>
        <!-- Audio Software  -->
        <MediaCodec name="OMX.qti.audio.decoder.flac" type="audio/flac" />
    </Decoders>
    <Include href="media_codecs_google_video.xml" />
</MediaCodecs>

```
以下分析以'OMX.google.h264.decoder'为主：
##### 2.1、SoftAVC::initEncoder()

``` cpp
[->Z:\MediaLearing\frameworks\av\media\libstagefright\codecs\avcenc\SoftAVCEnc.cpp]
OMX_ERRORTYPE SoftAVC::initEncoder() {
    IV_STATUS_T status;
    WORD32 level;
    uint32_t displaySizeY;

    CHECK(!mStarted);

    OMX_ERRORTYPE errType = OMX_ErrorNone;

    displaySizeY = mWidth * mHeight;
    if (displaySizeY > (1920 * 1088)) {
        level = 50;
    } else if (displaySizeY > (1280 * 720)) {
        level = 40;
    } else if (displaySizeY > (720 * 576)) {
        level = 31;
    } else if (displaySizeY > (624 * 320)) {
        level = 30;
    } else if (displaySizeY > (352 * 288)) {
        level = 21;
    } else {
        level = 20;
    }
    mAVCEncLevel = MAX(level, mAVCEncLevel);

    mStride = mWidth;

    if (mInputDataIsMeta) {
        for (size_t i = 0; i < MAX_CONVERSION_BUFFERS; i++) {
            if (mConversionBuffers[i] != NULL) {
                free(mConversionBuffers[i]);
                mConversionBuffers[i] = 0;
            }

            if (((uint64_t)mStride * mHeight) > ((uint64_t)INT32_MAX / 3)) {
                ALOGE("Buffer size is too big.");
                return OMX_ErrorUndefined;
            }
            mConversionBuffers[i] = (uint8_t *)malloc(mStride * mHeight * 3 / 2);

            if (mConversionBuffers[i] == NULL) {
                ALOGE("Allocating conversion buffer failed.");
                return OMX_ErrorUndefined;
            }

            mConversionBuffersFree[i] = 1;
        }
    }

    switch (mColorFormat) {
        case OMX_COLOR_FormatYUV420SemiPlanar:
            mIvVideoColorFormat = IV_YUV_420SP_UV;
            ALOGV("colorFormat YUV_420SP");
            break;
        default:
        case OMX_COLOR_FormatYUV420Planar:
            mIvVideoColorFormat = IV_YUV_420P;
            ALOGV("colorFormat YUV_420P");
            break;
    }

    ALOGD("Params width %d height %d level %d colorFormat %d", mWidth,
            mHeight, mAVCEncLevel, mIvVideoColorFormat);

    /* Getting Number of MemRecords */
    {
        iv_num_mem_rec_ip_t s_num_mem_rec_ip;
        iv_num_mem_rec_op_t s_num_mem_rec_op;

        s_num_mem_rec_ip.u4_size = sizeof(iv_num_mem_rec_ip_t);
        s_num_mem_rec_op.u4_size = sizeof(iv_num_mem_rec_op_t);

        s_num_mem_rec_ip.e_cmd = IV_CMD_GET_NUM_MEM_REC;

        status = ive_api_function(0, &s_num_mem_rec_ip, &s_num_mem_rec_op);

        if (status != IV_SUCCESS) {
            ALOGE("Get number of memory records failed = 0x%x\n",
                    s_num_mem_rec_op.u4_error_code);
            return OMX_ErrorUndefined;
        }

        mNumMemRecords = s_num_mem_rec_op.u4_num_mem_rec;
    }

    /* Allocate array to hold memory records */
    if (mNumMemRecords > SIZE_MAX / sizeof(iv_mem_rec_t)) {
        ALOGE("requested memory size is too big.");
        return OMX_ErrorUndefined;
    }
    mMemRecords = (iv_mem_rec_t *)malloc(mNumMemRecords * sizeof(iv_mem_rec_t));
    if (NULL == mMemRecords) {
        ALOGE("Unable to allocate memory for hold memory records: Size %zu",
                mNumMemRecords * sizeof(iv_mem_rec_t));
        mSignalledError = true;
        notify(OMX_EventError, OMX_ErrorUndefined, 0, 0);
        return OMX_ErrorUndefined;
    }

    {
        iv_mem_rec_t *ps_mem_rec;
        ps_mem_rec = mMemRecords;
        for (size_t i = 0; i < mNumMemRecords; i++) {
            ps_mem_rec->u4_size = sizeof(iv_mem_rec_t);
            ps_mem_rec->pv_base = NULL;
            ps_mem_rec->u4_mem_size = 0;
            ps_mem_rec->u4_mem_alignment = 0;
            ps_mem_rec->e_mem_type = IV_NA_MEM_TYPE;

            ps_mem_rec++;
        }
    }

    /* Getting MemRecords Attributes */
    {
        iv_fill_mem_rec_ip_t s_fill_mem_rec_ip;
        iv_fill_mem_rec_op_t s_fill_mem_rec_op;

        s_fill_mem_rec_ip.u4_size = sizeof(iv_fill_mem_rec_ip_t);
        s_fill_mem_rec_op.u4_size = sizeof(iv_fill_mem_rec_op_t);

        s_fill_mem_rec_ip.e_cmd = IV_CMD_FILL_NUM_MEM_REC;
        s_fill_mem_rec_ip.ps_mem_rec = mMemRecords;
        s_fill_mem_rec_ip.u4_num_mem_rec = mNumMemRecords;
        s_fill_mem_rec_ip.u4_max_wd = mWidth;
        s_fill_mem_rec_ip.u4_max_ht = mHeight;
        s_fill_mem_rec_ip.u4_max_level = mAVCEncLevel;
        s_fill_mem_rec_ip.e_color_format = DEFAULT_INP_COLOR_FORMAT;
        s_fill_mem_rec_ip.u4_max_ref_cnt = DEFAULT_MAX_REF_FRM;
        s_fill_mem_rec_ip.u4_max_reorder_cnt = DEFAULT_MAX_REORDER_FRM;
        s_fill_mem_rec_ip.u4_max_srch_rng_x = DEFAULT_MAX_SRCH_RANGE_X;
        s_fill_mem_rec_ip.u4_max_srch_rng_y = DEFAULT_MAX_SRCH_RANGE_Y;

        status = ive_api_function(0, &s_fill_mem_rec_ip, &s_fill_mem_rec_op);

        if (status != IV_SUCCESS) {
            ALOGE("Fill memory records failed = 0x%x\n",
                    s_fill_mem_rec_op.u4_error_code);
            mSignalledError = true;
            notify(OMX_EventError, OMX_ErrorUndefined, 0, 0);
            return OMX_ErrorUndefined;
        }
    }

    /* Allocating Memory for Mem Records */
    {
        WORD32 total_size;
        iv_mem_rec_t *ps_mem_rec;
        total_size = 0;
        ps_mem_rec = mMemRecords;

        for (size_t i = 0; i < mNumMemRecords; i++) {
            ps_mem_rec->pv_base = ive_aligned_malloc(
                    ps_mem_rec->u4_mem_alignment, ps_mem_rec->u4_mem_size);
            if (ps_mem_rec->pv_base == NULL) {
                ALOGE("Allocation failure for mem record id %zu size %u\n", i,
                        ps_mem_rec->u4_mem_size);
                mSignalledError = true;
                notify(OMX_EventError, OMX_ErrorUndefined, 0, 0);
                return OMX_ErrorUndefined;

            }
            total_size += ps_mem_rec->u4_mem_size;

            ps_mem_rec++;
        }
    }

    /* Codec Instance Creation */
    {
        ive_init_ip_t s_init_ip;
        ive_init_op_t s_init_op;

        mCodecCtx = (iv_obj_t *)mMemRecords[0].pv_base;
        mCodecCtx->u4_size = sizeof(iv_obj_t);
        mCodecCtx->pv_fxns = (void *)ive_api_function;

        s_init_ip.u4_size = sizeof(ive_init_ip_t);
        s_init_op.u4_size = sizeof(ive_init_op_t);

        s_init_ip.e_cmd = IV_CMD_INIT;
        s_init_ip.u4_num_mem_rec = mNumMemRecords;
        s_init_ip.ps_mem_rec = mMemRecords;
        s_init_ip.u4_max_wd = mWidth;
        s_init_ip.u4_max_ht = mHeight;
        s_init_ip.u4_max_ref_cnt = DEFAULT_MAX_REF_FRM;
        s_init_ip.u4_max_reorder_cnt = DEFAULT_MAX_REORDER_FRM;
        s_init_ip.u4_max_level = mAVCEncLevel;
        s_init_ip.e_inp_color_fmt = mIvVideoColorFormat;

        if (mReconEnable || mPSNREnable) {
            s_init_ip.u4_enable_recon = 1;
        } else {
            s_init_ip.u4_enable_recon = 0;
        }
        s_init_ip.e_recon_color_fmt = DEFAULT_RECON_COLOR_FORMAT;
        s_init_ip.e_rc_mode = DEFAULT_RC_MODE;
        s_init_ip.u4_max_framerate = DEFAULT_MAX_FRAMERATE;
        s_init_ip.u4_max_bitrate = DEFAULT_MAX_BITRATE;
        s_init_ip.u4_num_bframes = mBframes;
        s_init_ip.e_content_type = IV_PROGRESSIVE;
        s_init_ip.u4_max_srch_rng_x = DEFAULT_MAX_SRCH_RANGE_X;
        s_init_ip.u4_max_srch_rng_y = DEFAULT_MAX_SRCH_RANGE_Y;
        s_init_ip.e_slice_mode = mSliceMode;
        s_init_ip.u4_slice_param = mSliceParam;
        s_init_ip.e_arch = mArch;
        s_init_ip.e_soc = DEFAULT_SOC;

        status = ive_api_function(mCodecCtx, &s_init_ip, &s_init_op);

        ......
    }

    /* Get Codec Version */
    logVersion();

    /* set processor details */
    setNumCores();

    /* Video control Set Frame dimensions */
    setDimensions();

    /* Video control Set Frame rates */
    setFrameRate();

    /* Video control Set IPE Params */
    setIpeParams();

    /* Video control Set Bitrate */
    setBitRate();

    /* Video control Set QP */
    setQp();

    /* Video control Set AIR params */
    setAirParams();

    /* Video control Set VBV params */
    setVbvParams();

    /* Video control Set Motion estimation params */
    setMeParams();

    /* Video control Set GOP params */
    setGopParams();

    /* Video control Set Deblock params */
    setDeblockParams();

    /* Video control Set Profile params */
    setProfileParams();

    /* Video control Set in Encode header mode */
    setEncMode(IVE_ENC_MODE_HEADER);

    ALOGV("init_codec successfull");

    mSpsPpsHeaderReceived = false;
    mStarted = true;

    return OMX_ErrorNone;
}
```
##### 2.2、SoftAVC::onQueueFilled()

``` cpp
[->Z:\MediaLearing\frameworks\av\media\libstagefright\codecs\avcenc\SoftAVCEnc.cpp]
void SoftAVC::onQueueFilled(OMX_U32 portIndex) {
    IV_STATUS_T status;
    WORD32 timeDelay, timeTaken;

    UNUSED(portIndex);

    // Initialize encoder if not already initialized
    if (mCodecCtx == NULL) {
        if (OMX_ErrorNone != initEncoder()) {
            ALOGE("Failed to initialize encoder");
            notify(OMX_EventError, OMX_ErrorUndefined, 0 /* arg2 */, NULL /* data */);
            return;
        }
    }
    if (mSignalledError) {
        return;
    }

    List<BufferInfo *> &inQueue = getPortQueue(0);
    List<BufferInfo *> &outQueue = getPortQueue(1);

    while (!mSawOutputEOS && !outQueue.empty()) {

        OMX_ERRORTYPE error;
        ive_video_encode_ip_t s_encode_ip;
        ive_video_encode_op_t s_encode_op;
        BufferInfo *outputBufferInfo = *outQueue.begin();
        OMX_BUFFERHEADERTYPE *outputBufferHeader = outputBufferInfo->mHeader;

        BufferInfo *inputBufferInfo;
        OMX_BUFFERHEADERTYPE *inputBufferHeader;

        if (mSawInputEOS) {
            inputBufferHeader = NULL;
            inputBufferInfo = NULL;
        } else if (!inQueue.empty()) {
            inputBufferInfo = *inQueue.begin();
            inputBufferHeader = inputBufferInfo->mHeader;
        } else {
            return;
        }

        outputBufferHeader->nTimeStamp = 0;
        outputBufferHeader->nFlags = 0;
        outputBufferHeader->nOffset = 0;
        outputBufferHeader->nFilledLen = 0;
        outputBufferHeader->nOffset = 0;

        if (inputBufferHeader != NULL) {
            outputBufferHeader->nFlags = inputBufferHeader->nFlags;
        }

        uint8_t *outPtr = (uint8_t *)outputBufferHeader->pBuffer;

        if (!mSpsPpsHeaderReceived) {
            error = setEncodeArgs(&s_encode_ip, &s_encode_op, NULL, outputBufferHeader);
            if (error != OMX_ErrorNone) {
                mSignalledError = true;
                notify(OMX_EventError, OMX_ErrorUndefined, 0, 0);
                return;
            }
            status = ive_api_function(mCodecCtx, &s_encode_ip, &s_encode_op);

            if (IV_SUCCESS != status) {
                ALOGE("Encode Frame failed = 0x%x\n",
                        s_encode_op.u4_error_code);
            } else {
                ALOGV("Bytes Generated in header %d\n",
                        s_encode_op.s_out_buf.u4_bytes);
            }

            mSpsPpsHeaderReceived = true;

            outputBufferHeader->nFlags = OMX_BUFFERFLAG_CODECCONFIG;
            outputBufferHeader->nFilledLen = s_encode_op.s_out_buf.u4_bytes;
            if (inputBufferHeader != NULL) {
                outputBufferHeader->nTimeStamp = inputBufferHeader->nTimeStamp;
            }

            outQueue.erase(outQueue.begin());
            outputBufferInfo->mOwnedByUs = false;

            DUMP_TO_FILE(
                    mOutFile, outputBufferHeader->pBuffer,
                    outputBufferHeader->nFilledLen);
            notifyFillBufferDone(outputBufferHeader);

            setEncMode(IVE_ENC_MODE_PICTURE);
            return;
        }

        if (mUpdateFlag) {
            if (mUpdateFlag & kUpdateBitrate) {
                setBitRate();
            }
            if (mUpdateFlag & kRequestKeyFrame) {
                setFrameType(IV_IDR_FRAME);
            }
            if (mUpdateFlag & kUpdateAIRMode) {
                setAirParams();
                notify(OMX_EventPortSettingsChanged, kOutputPortIndex,
                        OMX_IndexConfigAndroidIntraRefresh, NULL);
            }
            mUpdateFlag = 0;
        }

        if ((inputBufferHeader != NULL)
                && (inputBufferHeader->nFlags & OMX_BUFFERFLAG_EOS)) {
            mSawInputEOS = true;
        }

        /* In normal mode, store inputBufferInfo and this will be returned
           when encoder consumes this input */
        if (!mInputDataIsMeta && (inputBufferInfo != NULL)) {
            for (size_t i = 0; i < MAX_INPUT_BUFFER_HEADERS; i++) {
                if (NULL == mInputBufferInfo[i]) {
                    mInputBufferInfo[i] = inputBufferInfo;
                    break;
                }
            }
        }
        error = setEncodeArgs(
                &s_encode_ip, &s_encode_op, inputBufferHeader, outputBufferHeader);

        if (error != OMX_ErrorNone) {
            mSignalledError = true;
            notify(OMX_EventError, OMX_ErrorUndefined, 0, 0);
            return;
        }

        DUMP_TO_FILE(
                mInFile, s_encode_ip.s_inp_buf.apv_bufs[0],
                (mHeight * mStride * 3 / 2));

        GETTIME(&mTimeStart, NULL);
        /* Compute time elapsed between end of previous decode()
         * to start of current decode() */
        TIME_DIFF(mTimeEnd, mTimeStart, timeDelay);
        status = ive_api_function(mCodecCtx, &s_encode_ip, &s_encode_op);

        if (IV_SUCCESS != status) {
            ALOGE("Encode Frame failed = 0x%x\n",
                    s_encode_op.u4_error_code);
            mSignalledError = true;
            notify(OMX_EventError, OMX_ErrorUndefined, 0, 0);
            return;
        }

        GETTIME(&mTimeEnd, NULL);
        /* Compute time taken for decode() */
        TIME_DIFF(mTimeStart, mTimeEnd, timeTaken);

        ALOGV("timeTaken=%6d delay=%6d numBytes=%6d", timeTaken, timeDelay,
                s_encode_op.s_out_buf.u4_bytes);

        /* In encoder frees up an input buffer, mark it as free */
        if (s_encode_op.s_inp_buf.apv_bufs[0] != NULL) {
            if (mInputDataIsMeta) {
                for (size_t i = 0; i < MAX_CONVERSION_BUFFERS; i++) {
                    if (mConversionBuffers[i] == s_encode_op.s_inp_buf.apv_bufs[0]) {
                        mConversionBuffersFree[i] = 1;
                        break;
                    }
                }
            } else {
                /* In normal mode, call EBD on inBuffeHeader that is freed by the codec */
                for (size_t i = 0; i < MAX_INPUT_BUFFER_HEADERS; i++) {
                    uint8_t *buf = NULL;
                    OMX_BUFFERHEADERTYPE *bufHdr = NULL;
                    if (mInputBufferInfo[i] != NULL) {
                        bufHdr = mInputBufferInfo[i]->mHeader;
                        buf = bufHdr->pBuffer + bufHdr->nOffset;
                    }
                    if (s_encode_op.s_inp_buf.apv_bufs[0] == buf) {
                        mInputBufferInfo[i]->mOwnedByUs = false;
                        notifyEmptyBufferDone(bufHdr);
                        mInputBufferInfo[i] = NULL;
                        break;
                    }
                }
            }
        }

        outputBufferHeader->nFilledLen = s_encode_op.s_out_buf.u4_bytes;

        if (IV_IDR_FRAME == s_encode_op.u4_encoded_frame_type) {
            outputBufferHeader->nFlags |= OMX_BUFFERFLAG_SYNCFRAME;
        }

        if (inputBufferHeader != NULL) {
            inQueue.erase(inQueue.begin());

            /* If in meta data, call EBD on input */
            /* In case of normal mode, EBD will be done once encoder
            releases the input buffer */
            if (mInputDataIsMeta) {
                inputBufferInfo->mOwnedByUs = false;
                notifyEmptyBufferDone(inputBufferHeader);
            }
        }

        if (s_encode_op.u4_is_last) {
            outputBufferHeader->nFlags |= OMX_BUFFERFLAG_EOS;
            mSawOutputEOS = true;
        } else {
            outputBufferHeader->nFlags &= ~OMX_BUFFERFLAG_EOS;
        }

        if (outputBufferHeader->nFilledLen || s_encode_op.u4_is_last) {
            outputBufferHeader->nTimeStamp = s_encode_op.u4_timestamp_high;
            outputBufferHeader->nTimeStamp <<= 32;
            outputBufferHeader->nTimeStamp |= s_encode_op.u4_timestamp_low;
            outputBufferInfo->mOwnedByUs = false;
            outQueue.erase(outQueue.begin());
            DUMP_TO_FILE(mOutFile, outputBufferHeader->pBuffer,
                    outputBufferHeader->nFilledLen);
            notifyFillBufferDone(outputBufferHeader);
        }

        if (s_encode_op.u4_is_last == 1) {
            return;
        }
    }
    return;
}
```
##### 2.3、ih264e_encode()

``` cpp
[->Z:\Ph55c74\PH55C74\android\external\libavc\encoder\ih264e_encode.c]
WORD32 ih264e_encode(iv_obj_t *ps_codec_obj, void *pv_api_ip, void *pv_api_op)
{
    /* error status */
    IH264E_ERROR_T error_status = IH264E_SUCCESS;

    /* codec ctxt */
    codec_t *ps_codec = (codec_t *)ps_codec_obj->pv_codec_handle;

    /* input frame to encode */
    ih264e_video_encode_ip_t *ps_video_encode_ip = pv_api_ip;

    /* output buffer to write stream */
    ih264e_video_encode_op_t *ps_video_encode_op = pv_api_op;

    /* i/o structures */
    inp_buf_t s_inp_buf;
    out_buf_t s_out_buf;

    /* temp var */
    WORD32 ctxt_sel = 0, i, i4_rc_pre_enc_skip;

    /********************************************************************/
    /*                            BEGIN INIT                            */
    /********************************************************************/
    /* reset output structure */
    ps_video_encode_op->s_ive_op.u4_error_code = IV_SUCCESS;
    ps_video_encode_op->s_ive_op.output_present  = 0;
    ps_video_encode_op->s_ive_op.dump_recon = 0;
    ps_video_encode_op->s_ive_op.u4_encoded_frame_type = IV_NA_FRAME;

    /* Check for output memory allocation size */
    if (ps_video_encode_ip->s_ive_ip.s_out_buf.u4_bufsize < MIN_STREAM_SIZE)
    {
        error_status |= IH264E_INSUFFICIENT_OUTPUT_BUFFER;
        SET_ERROR_ON_RETURN(error_status,
                            IVE_UNSUPPORTEDPARAM,
                            ps_video_encode_op->s_ive_op.u4_error_code,
                            IV_FAIL);
    }

    /* copy output info. to internal structure */
    s_out_buf.s_bits_buf = ps_video_encode_ip->s_ive_ip.s_out_buf;
    s_out_buf.u4_is_last = 0;
    s_out_buf.u4_timestamp_low = ps_video_encode_ip->s_ive_ip.u4_timestamp_low;
    s_out_buf.u4_timestamp_high = ps_video_encode_ip->s_ive_ip.u4_timestamp_high;

    /* api call cnt */
    ps_codec->i4_encode_api_call_cnt += 1;

    /* codec context selector */
    ctxt_sel = ps_codec->i4_encode_api_call_cnt % MAX_CTXT_SETS;

    /* reset status flags */
    ps_codec->ai4_pic_cnt[ctxt_sel] = -1;
    ps_codec->s_rate_control.post_encode_skip[ctxt_sel] = 0;
    ps_codec->s_rate_control.pre_encode_skip[ctxt_sel] = 0;

    /* pass output buffer to codec */
    ps_codec->as_out_buf[ctxt_sel] = s_out_buf;

    /* initialize codec ctxt with default params for the first encode api call */
    if (ps_codec->i4_encode_api_call_cnt == 0)
    {
        ih264e_codec_init(ps_codec);
    }

    /* parse configuration params */
    for (i = 0; i < MAX_ACTIVE_CONFIG_PARAMS; i++)
    {
        cfg_params_t *ps_cfg = &ps_codec->as_cfg[i];

        if (1 == ps_cfg->u4_is_valid)
        {
            if ( ((ps_cfg->u4_timestamp_high == ps_video_encode_ip->s_ive_ip.u4_timestamp_high) &&
                            (ps_cfg->u4_timestamp_low == ps_video_encode_ip->s_ive_ip.u4_timestamp_low)) ||
                            ((WORD32)ps_cfg->u4_timestamp_high == -1) ||
                            ((WORD32)ps_cfg->u4_timestamp_low == -1) )
            {
                error_status |= ih264e_codec_update_config(ps_codec, ps_cfg);
                SET_ERROR_ON_RETURN(error_status,
                                    IVE_UNSUPPORTEDPARAM,
                                    ps_video_encode_op->s_ive_op.u4_error_code,
                                    IV_FAIL);

                ps_cfg->u4_is_valid = 0;
            }
        }
    }
    ......
    /* Send the input to encoder so that it can free it if possible */
    ps_video_encode_op->s_ive_op.s_out_buf = s_out_buf.s_bits_buf;
    ps_video_encode_op->s_ive_op.s_inp_buf = s_inp_buf.s_raw_buf;


    if (1 == s_inp_buf.u4_is_last)
    {
        ps_video_encode_op->s_ive_op.output_present = 0;
        ps_video_encode_op->s_ive_op.dump_recon = 0;
    }

    return IV_SUCCESS;
}

```
可以看到编码过程极其复杂...精髓在libavc库里面，第四节详细分析（Todo）。
#### （三）、Android  H264解码
##### 3.1、SoftAVC::initDecoder()
![Alt text | center](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/android.codecs/VS9-18-H264-Decoder-google.png)


``` cpp
[->Z:\MediaLearing\frameworks\av\media\libstagefright\codecs\avcdec\SoftAVCDec.cpp]
status_t SoftAVC::initDecoder() {
    IV_API_CALL_STATUS_T status;

    mNumCores = GetCPUCoreCount();
    mCodecCtx = NULL;

    mStride = outputBufferWidth();

    /* Initialize the decoder */
    {
        ivdext_create_ip_t s_create_ip;
        ivdext_create_op_t s_create_op;

        void *dec_fxns = (void *)ivdec_api_function;

        s_create_ip.s_ivd_create_ip_t.u4_size = sizeof(ivdext_create_ip_t);
        s_create_ip.s_ivd_create_ip_t.e_cmd = IVD_CMD_CREATE;
        s_create_ip.s_ivd_create_ip_t.u4_share_disp_buf = 0;
        s_create_op.s_ivd_create_op_t.u4_size = sizeof(ivdext_create_op_t);
        s_create_ip.s_ivd_create_ip_t.e_output_format = mIvColorFormat;
        s_create_ip.s_ivd_create_ip_t.pf_aligned_alloc = ivd_aligned_malloc;
        s_create_ip.s_ivd_create_ip_t.pf_aligned_free = ivd_aligned_free;
        s_create_ip.s_ivd_create_ip_t.pv_mem_ctxt = NULL;

        status = ivdec_api_function(mCodecCtx, (void *)&s_create_ip, (void *)&s_create_op);

        if (status != IV_SUCCESS) {
            ALOGE("Error in create: 0x%x",
                    s_create_op.s_ivd_create_op_t.u4_error_code);
            deInitDecoder();
            mCodecCtx = NULL;
            return UNKNOWN_ERROR;
        }

        mCodecCtx = (iv_obj_t*)s_create_op.s_ivd_create_op_t.pv_handle;
        mCodecCtx->pv_fxns = dec_fxns;
        mCodecCtx->u4_size = sizeof(iv_obj_t);
    }

    /* Reset the plugin state */
    resetPlugin();

    /* Set the run time (dynamic) parameters */
    setParams(mStride);

    /* Set number of cores/threads to be used by the codec */
    setNumCores();

    /* Get codec version */
    logVersion();

    mFlushNeeded = false;
    return OK;
}

```
##### 3.2、SoftAVC::onQueueFilled()

``` cpp
[->Z:\MediaLearing\frameworks\av\media\libstagefright\codecs\avcdec\SoftAVCDec.cpp]
void SoftAVC::onQueueFilled(OMX_U32 portIndex) {
    UNUSED(portIndex);

   ......
    if (outputBufferWidth() != mStride) {
        /* Set the run-time (dynamic) parameters */
        mStride = outputBufferWidth();
        setParams(mStride);
    }

    List<BufferInfo *> &inQueue = getPortQueue(kInputPortIndex);
    List<BufferInfo *> &outQueue = getPortQueue(kOutputPortIndex);

    while (!outQueue.empty()) {
        BufferInfo *inInfo;
        OMX_BUFFERHEADERTYPE *inHeader;

        BufferInfo *outInfo;
        OMX_BUFFERHEADERTYPE *outHeader;
        size_t timeStampIx;

        inInfo = NULL;
        inHeader = NULL;

        ......
        outInfo = *outQueue.begin();
        outHeader = outInfo->mHeader;
        outHeader->nFlags = 0;
        outHeader->nTimeStamp = 0;
        outHeader->nOffset = 0;

        ......

        /* Get a free slot in timestamp array to hold input timestamp */
        {
            size_t i;
            timeStampIx = 0;
            for (i = 0; i < MAX_TIME_STAMPS; i++) {
                if (!mTimeStampsValid[i]) {
                    timeStampIx = i;
                    break;
                }
            }
            if (inHeader != NULL) {
                mTimeStampsValid[timeStampIx] = true;
                mTimeStamps[timeStampIx] = inHeader->nTimeStamp;
            }
        }

        {
            ivd_video_decode_ip_t s_dec_ip;
            ivd_video_decode_op_t s_dec_op;
            WORD32 timeDelay, timeTaken;
            size_t sizeY, sizeUV;

            if (!setDecodeArgs(&s_dec_ip, &s_dec_op, inHeader, outHeader, timeStampIx)) {
                ALOGE("Decoder arg setup failed");
                notify(OMX_EventError, OMX_ErrorUndefined, 0, NULL);
                mSignalledError = true;
                return;
            }
            // If input dump is enabled, then write to file
            DUMP_TO_FILE(mInFile, s_dec_ip.pv_stream_buffer, s_dec_ip.u4_num_Bytes);

            GETTIME(&mTimeStart, NULL);
            /* Compute time elapsed between end of previous decode()
             * to start of current decode() */
            TIME_DIFF(mTimeEnd, mTimeStart, timeDelay);

            IV_API_CALL_STATUS_T status;
            status = ivdec_api_function(mCodecCtx, (void *)&s_dec_ip, (void *)&s_dec_op);

            bool unsupportedResolution =
                (IVD_STREAM_WIDTH_HEIGHT_NOT_SUPPORTED == (s_dec_op.u4_error_code & 0xFF));

            /* Check for unsupported dimensions */
            if (unsupportedResolution) {
                ALOGE("Unsupported resolution : %dx%d", mWidth, mHeight);
                notify(OMX_EventError, OMX_ErrorUnsupportedSetting, 0, NULL);
                mSignalledError = true;
                return;
            }

            bool allocationFailed = (IVD_MEM_ALLOC_FAILED == (s_dec_op.u4_error_code & 0xFF));
            if (allocationFailed) {
                ALOGE("Allocation failure in decoder");
                notify(OMX_EventError, OMX_ErrorUnsupportedSetting, 0, NULL);
                mSignalledError = true;
                return;
            }

            bool resChanged = (IVD_RES_CHANGED == (s_dec_op.u4_error_code & 0xFF));

            getVUIParams();

            GETTIME(&mTimeEnd, NULL);
            /* Compute time taken for decode() */
            TIME_DIFF(mTimeStart, mTimeEnd, timeTaken);

            PRINT_TIME("timeTaken=%6d delay=%6d numBytes=%6d", timeTaken, timeDelay,
                   s_dec_op.u4_num_bytes_consumed);
            if (s_dec_op.u4_frame_decoded_flag && !mFlushNeeded) {
                mFlushNeeded = true;
            }

            if ((inHeader != NULL) && (1 != s_dec_op.u4_frame_decoded_flag)) {
                /* If the input did not contain picture data, then ignore
                 * the associated timestamp */
                mTimeStampsValid[timeStampIx] = false;
            }

            // If the decoder is in the changing resolution mode and there is no output present,
            // that means the switching is done and it's ready to reset the decoder and the plugin.
            if (mChangingResolution && !s_dec_op.u4_output_present) {
                mChangingResolution = false;
                resetDecoder();
                resetPlugin();
                mStride = outputBufferWidth();
                setParams(mStride);
                continue;
            }

            if (resChanged) {
                mChangingResolution = true;
                if (mFlushNeeded) {
                    setFlushMode();
                }
                continue;
            }

            // Combine the resolution change and coloraspects change in one PortSettingChange event
            // if necessary.
            if ((0 < s_dec_op.u4_pic_wd) && (0 < s_dec_op.u4_pic_ht)) {
                uint32_t width = s_dec_op.u4_pic_wd;
                uint32_t height = s_dec_op.u4_pic_ht;
                bool portWillReset = false;
                handlePortSettingsChange(&portWillReset, width, height);
                if (portWillReset) {
                    resetDecoder();
                    return;
                }
            } else if (mUpdateColorAspects) {
                notify(OMX_EventPortSettingsChanged, kOutputPortIndex,
                    kDescribeColorAspectsIndex, NULL);
                mUpdateColorAspects = false;
                return;
            }

            if (s_dec_op.u4_output_present) {
                outHeader->nFilledLen = (outputBufferWidth() * outputBufferHeight() * 3) / 2;

                outHeader->nTimeStamp = mTimeStamps[s_dec_op.u4_ts];
                mTimeStampsValid[s_dec_op.u4_ts] = false;

                outInfo->mOwnedByUs = false;
                outQueue.erase(outQueue.begin());
                outInfo = NULL;
                notifyFillBufferDone(outHeader);
                outHeader = NULL;
            } else if (mIsInFlush) {
                /* If in flush mode and no output is returned by the codec,
                 * then come out of flush mode */
                mIsInFlush = false;

                /* If EOS was recieved on input port and there is no output
                 * from the codec, then signal EOS on output port */
                if (mReceivedEOS) {
                    outHeader->nFilledLen = 0;
                    outHeader->nFlags |= OMX_BUFFERFLAG_EOS;

                    outInfo->mOwnedByUs = false;
                    outQueue.erase(outQueue.begin());
                    outInfo = NULL;
                    notifyFillBufferDone(outHeader);
                    outHeader = NULL;
                    resetPlugin();
                }
            }
        }

        ......
    }
}
```
##### 3.3、WORD32 ih264d_video_decode()

``` cpp
[->Z:\Ph55c74\PH55C74\android\external\libavc\decoder\ih264d_api.c]
WORD32 ih264d_video_decode(iv_obj_t *dec_hdl, void *pv_api_ip, void *pv_api_op)
{
    /* ! */

    dec_struct_t * ps_dec = (dec_struct_t *)(dec_hdl->pv_codec_handle);

    WORD32 i4_err_status = 0;
    UWORD8 *pu1_buf = NULL;
    WORD32 buflen;
    UWORD32 u4_max_ofst, u4_length_of_start_code = 0;

    UWORD32 bytes_consumed = 0;
    UWORD32 cur_slice_is_nonref = 0;
    UWORD32 u4_next_is_aud;
    UWORD32 u4_first_start_code_found = 0;
    WORD32 ret = 0,api_ret_value = IV_SUCCESS;
    WORD32 header_data_left = 0,frame_data_left = 0;
    UWORD8 *pu1_bitstrm_buf;
    ivd_video_decode_ip_t *ps_dec_ip;
    ivd_video_decode_op_t *ps_dec_op;

    ithread_set_name((void*)"Parse_thread");

    ps_dec_ip = (ivd_video_decode_ip_t *)pv_api_ip;
    ps_dec_op = (ivd_video_decode_op_t *)pv_api_op;

    {
        UWORD32 u4_size;
        u4_size = ps_dec_op->u4_size;
        memset(ps_dec_op, 0, sizeof(ivd_video_decode_op_t));
        ps_dec_op->u4_size = u4_size;
    }

    ps_dec->pv_dec_out = ps_dec_op;
    if(ps_dec->init_done != 1)
    {
        return IV_FAIL;
    }

    /*Data memory barries instruction,so that bitstream write by the application is complete*/
    DATA_SYNC();

    if(0 == ps_dec->u1_flushfrm)
    {
        if(ps_dec_ip->pv_stream_buffer == NULL)
        {
            ps_dec_op->u4_error_code |= 1 << IVD_UNSUPPORTEDPARAM;
            ps_dec_op->u4_error_code |= IVD_DEC_FRM_BS_BUF_NULL;
            return IV_FAIL;
        }
        if(ps_dec_ip->u4_num_Bytes <= 0)
        {
            ps_dec_op->u4_error_code |= 1 << IVD_UNSUPPORTEDPARAM;
            ps_dec_op->u4_error_code |= IVD_DEC_NUMBYTES_INV;
            return IV_FAIL;

        }
    }
    ps_dec->u1_pic_decode_done = 0;

    ps_dec_op->u4_num_bytes_consumed = 0;

    ps_dec->ps_out_buffer = NULL;

    if(ps_dec_ip->u4_size
                    >= offsetof(ivd_video_decode_ip_t, s_out_buffer))
        ps_dec->ps_out_buffer = &ps_dec_ip->s_out_buffer;

    ps_dec->u4_fmt_conv_cur_row = 0;

    ps_dec->u4_output_present = 0;
    ps_dec->s_disp_op.u4_error_code = 1;
    ps_dec->u4_fmt_conv_num_rows = FMT_CONV_NUM_ROWS;
    if(0 == ps_dec->u4_share_disp_buf
                    && ps_dec->i4_decode_header == 0)
    {
        UWORD32 i;
        if((ps_dec->ps_out_buffer->u4_num_bufs == 0) ||
           (ps_dec->ps_out_buffer->u4_num_bufs > IVD_VIDDEC_MAX_IO_BUFFERS))
        {
            ps_dec_op->u4_error_code |= 1 << IVD_UNSUPPORTEDPARAM;
            ps_dec_op->u4_error_code |= IVD_DISP_FRM_ZERO_OP_BUFS;
            return IV_FAIL;
        }

        for(i = 0; i < ps_dec->ps_out_buffer->u4_num_bufs; i++)
        {
            if(ps_dec->ps_out_buffer->pu1_bufs[i] == NULL)
            {
                ps_dec_op->u4_error_code |= 1 << IVD_UNSUPPORTEDPARAM;
                ps_dec_op->u4_error_code |= IVD_DISP_FRM_OP_BUF_NULL;
                return IV_FAIL;
            }

            if(ps_dec->ps_out_buffer->u4_min_out_buf_size[i] == 0)
            {
                ps_dec_op->u4_error_code |= 1 << IVD_UNSUPPORTEDPARAM;
                ps_dec_op->u4_error_code |=
                                IVD_DISP_FRM_ZERO_OP_BUF_SIZE;
                return IV_FAIL;
            }
        }
    }

    if(ps_dec->u4_total_frames_decoded >= NUM_FRAMES_LIMIT)
    {
        ps_dec_op->u4_error_code = ERROR_FRAME_LIMIT_OVER;
        return IV_FAIL;
    }

    /* ! */
    ps_dec->u4_ts = ps_dec_ip->u4_ts;

    ps_dec_op->u4_error_code = 0;
    ps_dec_op->e_pic_type = -1;
    ps_dec_op->u4_output_present = 0;
    ps_dec_op->u4_frame_decoded_flag = 0;

    ps_dec->i4_frametype = -1;
    ps_dec->i4_content_type = -1;

    ps_dec->u4_slice_start_code_found = 0;

    /* In case the deocder is not in flush mode(in shared mode),
     then decoder has to pick up a buffer to write current frame.
     Check if a frame is available in such cases */

    if(ps_dec->u1_init_dec_flag == 1 && ps_dec->u4_share_disp_buf == 1
                    && ps_dec->u1_flushfrm == 0)
    {
        UWORD32 i;

        WORD32 disp_avail = 0, free_id;

        /* Check if at least one buffer is available with the codec */
        /* If not then return to application with error */
        for(i = 0; i < ps_dec->u1_pic_bufs; i++)
        {
            if(0 == ps_dec->u4_disp_buf_mapping[i]
                            || 1 == ps_dec->u4_disp_buf_to_be_freed[i])
            {
                disp_avail = 1;
                break;
            }

        }

        if(0 == disp_avail)
        {
            /* If something is queued for display wait for that buffer to be returned */

            ps_dec_op->u4_error_code = IVD_DEC_REF_BUF_NULL;
            ps_dec_op->u4_error_code |= (1 << IVD_UNSUPPORTEDPARAM);
            return (IV_FAIL);
        }

        while(1)
        {
            pic_buffer_t *ps_pic_buf;
            ps_pic_buf = (pic_buffer_t *)ih264_buf_mgr_get_next_free(
                            (buf_mgr_t *)ps_dec->pv_pic_buf_mgr, &free_id);

            if(ps_pic_buf == NULL)
            {
                UWORD32 i, display_queued = 0;

                /* check if any buffer was given for display which is not returned yet */
                for(i = 0; i < (MAX_DISP_BUFS_NEW); i++)
                {
                    if(0 != ps_dec->u4_disp_buf_mapping[i])
                    {
                        display_queued = 1;
                        break;
                    }
                }
                /* If some buffer is queued for display, then codec has to singal an error and wait
                 for that buffer to be returned.
                 If nothing is queued for display then codec has ownership of all display buffers
                 and it can reuse any of the existing buffers and continue decoding */

                if(1 == display_queued)
                {
                    /* If something is queued for display wait for that buffer to be returned */
                    ps_dec_op->u4_error_code = IVD_DEC_REF_BUF_NULL;
                    ps_dec_op->u4_error_code |= (1
                                    << IVD_UNSUPPORTEDPARAM);
                    return (IV_FAIL);
                }
            }
            else
            {
                /* If the buffer is with display, then mark it as in use and then look for a buffer again */
                if(1 == ps_dec->u4_disp_buf_mapping[free_id])
                {
                    ih264_buf_mgr_set_status(
                                    (buf_mgr_t *)ps_dec->pv_pic_buf_mgr,
                                    free_id,
                                    BUF_MGR_IO);
                }
                else
                {
                    /**
                     *  Found a free buffer for present call. Release it now.
                     *  Will be again obtained later.
                     */
                    ih264_buf_mgr_release((buf_mgr_t *)ps_dec->pv_pic_buf_mgr,
                                          free_id,
                                          BUF_MGR_IO);
                    break;
                }
            }
        }

    }

    if(ps_dec->u1_flushfrm)
    {
        if(ps_dec->u1_init_dec_flag == 0)
        {
            /*Come out of flush mode and return*/
            ps_dec->u1_flushfrm = 0;
            return (IV_FAIL);
        }



        ih264d_get_next_display_field(ps_dec, ps_dec->ps_out_buffer,
                                      &(ps_dec->s_disp_op));
        if(0 == ps_dec->s_disp_op.u4_error_code)
        {
            /* check output buffer size given by the application */
            if(check_app_out_buf_size(ps_dec) != IV_SUCCESS)
            {
                ps_dec_op->u4_error_code= IVD_DISP_FRM_ZERO_OP_BUF_SIZE;
                return (IV_FAIL);
            }

            ps_dec->u4_fmt_conv_cur_row = 0;
            ps_dec->u4_fmt_conv_num_rows = ps_dec->s_disp_frame_info.u4_y_ht;
            ih264d_format_convert(ps_dec, &(ps_dec->s_disp_op),
                                  ps_dec->u4_fmt_conv_cur_row,
                                  ps_dec->u4_fmt_conv_num_rows);
            ps_dec->u4_fmt_conv_cur_row += ps_dec->u4_fmt_conv_num_rows;
            ps_dec->u4_output_present = 1;

        }
        ih264d_release_display_field(ps_dec, &(ps_dec->s_disp_op));

        ps_dec_op->u4_pic_wd = (UWORD32)ps_dec->u2_disp_width;
        ps_dec_op->u4_pic_ht = (UWORD32)ps_dec->u2_disp_height;

        ps_dec_op->u4_new_seq = 0;

        ps_dec_op->u4_output_present = ps_dec->u4_output_present;
        ps_dec_op->u4_progressive_frame_flag =
                        ps_dec->s_disp_op.u4_progressive_frame_flag;
        ps_dec_op->e_output_format =
                        ps_dec->s_disp_op.e_output_format;
        ps_dec_op->s_disp_frm_buf = ps_dec->s_disp_op.s_disp_frm_buf;
        ps_dec_op->e4_fld_type = ps_dec->s_disp_op.e4_fld_type;
        ps_dec_op->u4_ts = ps_dec->s_disp_op.u4_ts;
        ps_dec_op->u4_disp_buf_id = ps_dec->s_disp_op.u4_disp_buf_id;

        /*In the case of flush ,since no frame is decoded set pic type as invalid*/
        ps_dec_op->u4_is_ref_flag = -1;
        ps_dec_op->e_pic_type = IV_NA_FRAME;
        ps_dec_op->u4_frame_decoded_flag = 0;

        if(0 == ps_dec->s_disp_op.u4_error_code)
        {
            return (IV_SUCCESS);
        }
        else
            return (IV_FAIL);

    }
    if(ps_dec->u1_res_changed == 1)
    {
        /*if resolution has changed and all buffers have been flushed, reset decoder*/
        ih264d_init_decoder(ps_dec);
    }

    ps_dec->u4_prev_nal_skipped = 0;

    ps_dec->u2_cur_mb_addr = 0;
    ps_dec->u2_total_mbs_coded = 0;
    ps_dec->u2_cur_slice_num = 0;
    ps_dec->cur_dec_mb_num = 0;
    ps_dec->cur_recon_mb_num = 0;
    ps_dec->u4_first_slice_in_pic = 1;
    ps_dec->u1_slice_header_done = 0;
    ps_dec->u1_dangling_field = 0;

    ps_dec->u4_dec_thread_created = 0;
    ps_dec->u4_bs_deblk_thread_created = 0;
    ps_dec->u4_cur_bs_mb_num = 0;
    ps_dec->u4_start_recon_deblk  = 0;
    ps_dec->u4_sps_cnt_in_process = 0;

    DEBUG_THREADS_PRINTF(" Starting process call\n");


    ps_dec->u4_pic_buf_got = 0;

    do
    {
        WORD32 buf_size;

        pu1_buf = (UWORD8*)ps_dec_ip->pv_stream_buffer
                        + ps_dec_op->u4_num_bytes_consumed;

        u4_max_ofst = ps_dec_ip->u4_num_Bytes
                        - ps_dec_op->u4_num_bytes_consumed;

        /* If dynamic bitstream buffer is not allocated and
         * header decode is done, then allocate dynamic bitstream buffer
         */
        if((NULL == ps_dec->pu1_bits_buf_dynamic) &&
           (ps_dec->i4_header_decoded & 1))
        {
            WORD32 size;

            void *pv_buf;
            void *pv_mem_ctxt = ps_dec->pv_mem_ctxt;
            size = MAX(256000, ps_dec->u2_pic_wd * ps_dec->u2_pic_ht * 3 / 2);
            pv_buf = ps_dec->pf_aligned_alloc(pv_mem_ctxt, 128,
                                              size + EXTRA_BS_OFFSET);
            RETURN_IF((NULL == pv_buf), IV_FAIL);
            ps_dec->pu1_bits_buf_dynamic = pv_buf;
            ps_dec->u4_dynamic_bits_buf_size = size;
        }

        if(ps_dec->pu1_bits_buf_dynamic)
        {
            pu1_bitstrm_buf = ps_dec->pu1_bits_buf_dynamic;
            buf_size = ps_dec->u4_dynamic_bits_buf_size;
        }
        else
        {
            pu1_bitstrm_buf = ps_dec->pu1_bits_buf_static;
            buf_size = ps_dec->u4_static_bits_buf_size;
        }

        u4_next_is_aud = 0;

        buflen = ih264d_find_start_code(pu1_buf, 0, u4_max_ofst,
                                               &u4_length_of_start_code,
                                               &u4_next_is_aud);

        if(buflen == -1)
            buflen = 0;
        /* Ignore bytes beyond the allocated size of intermediate buffer */
        /* Since 8 bytes are read ahead, ensure 8 bytes are free at the
        end of the buffer, which will be memset to 0 after emulation prevention */
        buflen = MIN(buflen, buf_size - 8);

        bytes_consumed = buflen + u4_length_of_start_code;
        ps_dec_op->u4_num_bytes_consumed += bytes_consumed;

        {
            UWORD8 u1_firstbyte, u1_nal_ref_idc;

            if(ps_dec->i4_app_skip_mode == IVD_SKIP_B)
            {
                u1_firstbyte = *(pu1_buf + u4_length_of_start_code);
                u1_nal_ref_idc = (UWORD8)(NAL_REF_IDC(u1_firstbyte));
                if(u1_nal_ref_idc == 0)
                {
                    /*skip non reference frames*/
                    cur_slice_is_nonref = 1;
                    continue;
                }
                else
                {
                    if(1 == cur_slice_is_nonref)
                    {
                        /*We have encountered a referenced frame,return to app*/
                        ps_dec_op->u4_num_bytes_consumed -=
                                        bytes_consumed;
                        ps_dec_op->e_pic_type = IV_B_FRAME;
                        ps_dec_op->u4_error_code =
                                        IVD_DEC_FRM_SKIPPED;
                        ps_dec_op->u4_error_code |= (1
                                        << IVD_UNSUPPORTEDPARAM);
                        ps_dec_op->u4_frame_decoded_flag = 0;
                        ps_dec_op->u4_size =
                                        sizeof(ivd_video_decode_op_t);
                        /*signal the decode thread*/
                        ih264d_signal_decode_thread(ps_dec);
                        /* close deblock thread if it is not closed yet*/
                        if(ps_dec->u4_num_cores == 3)
                        {
                            ih264d_signal_bs_deblk_thread(ps_dec);
                        }

                        return (IV_FAIL);
                    }
                }

            }

        }


        if(buflen)
        {
            memcpy(pu1_bitstrm_buf, pu1_buf + u4_length_of_start_code,
                   buflen);
            /* Decoder may read extra 8 bytes near end of the frame */
            if((buflen + 8) < buf_size)
            {
                memset(pu1_bitstrm_buf + buflen, 0, 8);
            }
            u4_first_start_code_found = 1;

        }
        else
        {
            /*start code not found*/

            if(u4_first_start_code_found == 0)
            {
                /*no start codes found in current process call*/

                ps_dec->i4_error_code = ERROR_START_CODE_NOT_FOUND;
                ps_dec_op->u4_error_code |= 1 << IVD_INSUFFICIENTDATA;

                if(ps_dec->u4_pic_buf_got == 0)
                {

                    ih264d_fill_output_struct_from_context(ps_dec,
                                                           ps_dec_op);

                    ps_dec_op->u4_error_code = ps_dec->i4_error_code;
                    ps_dec_op->u4_frame_decoded_flag = 0;

                    return (IV_FAIL);
                }
                else
                {
                    ps_dec->u1_pic_decode_done = 1;
                    continue;
                }
            }
            else
            {
                /* a start code has already been found earlier in the same process call*/
                frame_data_left = 0;
                header_data_left = 0;
                continue;
            }

        }

        ps_dec->u4_return_to_app = 0;
        ret = ih264d_parse_nal_unit(dec_hdl, ps_dec_op,
                              pu1_bitstrm_buf, buflen);
        if(ret != OK)
        {
            UWORD32 error =  ih264d_map_error(ret);
            ps_dec_op->u4_error_code = error | ret;
            api_ret_value = IV_FAIL;

            if((ret == IVD_RES_CHANGED)
                            || (ret == IVD_MEM_ALLOC_FAILED)
                            || (ret == ERROR_UNAVAIL_PICBUF_T)
                            || (ret == ERROR_UNAVAIL_MVBUF_T)
                            || (ret == ERROR_INV_SPS_PPS_T)
                            || (ret == IVD_DISP_FRM_ZERO_OP_BUF_SIZE))
            {
                ps_dec->u4_slice_start_code_found = 0;
                break;
            }

            if((ret == ERROR_INCOMPLETE_FRAME) || (ret == ERROR_DANGLING_FIELD_IN_PIC))
            {
                ps_dec_op->u4_num_bytes_consumed -= bytes_consumed;
                api_ret_value = IV_FAIL;
                break;
            }

            if(ret == ERROR_IN_LAST_SLICE_OF_PIC)
            {
                api_ret_value = IV_FAIL;
                break;
            }

        }

        if(ps_dec->u4_return_to_app)
        {
            /*We have encountered a referenced frame,return to app*/
            ps_dec_op->u4_num_bytes_consumed -= bytes_consumed;
            ps_dec_op->u4_error_code = IVD_DEC_FRM_SKIPPED;
            ps_dec_op->u4_error_code |= (1 << IVD_UNSUPPORTEDPARAM);
            ps_dec_op->u4_frame_decoded_flag = 0;
            ps_dec_op->u4_size = sizeof(ivd_video_decode_op_t);
            /*signal the decode thread*/
            ih264d_signal_decode_thread(ps_dec);
            /* close deblock thread if it is not closed yet*/
            if(ps_dec->u4_num_cores == 3)
            {
                ih264d_signal_bs_deblk_thread(ps_dec);
            }
            return (IV_FAIL);

        }



        header_data_left = ((ps_dec->i4_decode_header == 1)
                        && (ps_dec->i4_header_decoded != 3)
                        && (ps_dec_op->u4_num_bytes_consumed
                                        < ps_dec_ip->u4_num_Bytes));
        frame_data_left = (((ps_dec->i4_decode_header == 0)
                        && ((ps_dec->u1_pic_decode_done == 0)
                                        || (u4_next_is_aud == 1)))
                        && (ps_dec_op->u4_num_bytes_consumed
                                        < ps_dec_ip->u4_num_Bytes));
    }
    while(( header_data_left == 1)||(frame_data_left == 1));
   ......
}
```
代码还是超级多，就不详细分析了，，，编解码知识需要长时间积累再做第四节分析了。
#### （四）、libavc库分析
**Todo---**
>\external\libavc（libavc 库）
- encoder
- decoder
- common

#### （五）、参考资料(特别感谢各位前辈的分析和图示)：
[从零了解H264结构](http://www.iosxxx.com/blog/2017-08-09-%E4%BB%8E%E9%9B%B6%E4%BA%86%E8%A7%A3H264%E7%BB%93%E6%9E%84.html)
[新一代视频压缩编码标准H.264](http://read.pudn.com/downloads147/ebook/635957/%E6%96%B0%E4%B8%80%E4%BB%A3%E8%A7%86%E9%A2%91%E5%8E%8B%E7%BC%A9%E7%BC%96%E7%A0%81%E6%A0%87%E5%87%86H.264.pdf)

