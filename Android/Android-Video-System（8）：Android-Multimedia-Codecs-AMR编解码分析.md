---
title: Android Video System（8）：Android Multimedia Codecs - AMR编解码分析
cover: https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/hexo.themes/bing-wallpaper-2018.04.30.jpg
categories:
  - Multimedia
tags:
  - Android
  - Video
  - Multimedia
toc: true
abbrlink: 20190215
date: 2019-02-15 09:25:00
---

--------------------------------------------------------------------------------

注：文章都是通过阅读各位前辈总结的资料 Android 8.x && Linux（kernel 4.x）Qualcomm平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网（◕‿◕），如有侵权，请联系删除，禁止转载（©Qualcomm ©Android @Linux 版权所有），谢谢。


首先感谢：

[【AMR音频编码器概述及文件格式分析】](https://blog.csdn.net/lusonglin121/article/details/9332823)

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
    { "OMX.google.amrnb.decoder", "amrdec", "audio_decoder.amrnb" },
    { "OMX.google.amrnb.encoder", "amrnbenc", "audio_encoder.amrnb" },
    { "OMX.google.amrwb.decoder", "amrdec", "audio_decoder.amrwb" },
    { "OMX.google.amrwb.encoder", "amrwbenc", "audio_encoder.amrwb" },
    ......
};
```

==源码（部分）==：

>PATH : \frameworks\av\media\libstagefright\codecs （Android Codecs）

- amrnb
- amrwb
- amrwbenc

Log：
[AMR-DEcorder-Google-Log.md](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/android.codecs/AMR-DEcorder-Google-Log.md)
[AMR-Encorder-Google-Log.md](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/android.codecs/AMR-Encorder-Google-Log.md)
--------------------------------------------------------------------------------

#### （一）、AMR音频编码器概述及文件格式分析
全称Adaptive Multi-Rate，自适应多速率编码，主要用于移动设备的音频，压缩比比较大，但相对其他的压缩格式质量比较差，由于多用于人声，通话，效果还是很不错的。
##### 一、分类
1. AMR: 又称为AMR-NB，相对于下面的WB而言，
语音带宽范围：300－3400Hz，
8KHz抽样
2. AMR-WB:AMR WideBand，
语音带宽范围： 50－7000Hz
16KHz 抽样
“AMR-WB”全称为“Adaptive Multi-rate - Wideband”，即“自适应多速率宽带编码”，采样频率为16kHz，是一种同时被国际标准化组织ITU-T和3GPP采用的宽带语音编码标准，也称 为G722.2标准。AMR-WB提供语音带宽范围达到50～7000Hz，用户可主观感受到话音比以前更加自然、舒适和易于分辨。
与之作比较，现在GSM用的EFR(Enhenced Full Rate，增强型全速率编码)采样频率为8kHz，语音带宽为200～3400Hz。
AMR-WB应用于窄带GSM(全速信道16k，GMSK)的优势在于其可采用从6.6kb/s, 8.85kb/s和12.65kb/s三种编码，当网络繁忙时C/I恶化，编码器可以自动调整编码模式，从而增强QoS。在这种应用中，AMR-WB抗扰 度优于AMR-NB。
AMR-WB应用于EDGE、3G可充分体现其优势。足够的传输带宽保证AMR-WB可采用从6.6kb/s到23.85kb/s共九种编码，语音质量超越PSTN固定电话。

##### 三、编码方式
1. AMR-NB:
AMR 一共有16种编码方式， 0-7对应8种不同的编码方式， 8-15 用于噪音或者保留用。
![Alt text | center](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/android.codecs/VS8-01-AMR-NB-Codecs-16.png)

2. AMR-WB:

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/android.codecs/VS8-02-AMR-WB-Codecs-16.png.png)


二、AMR 帧格式：
AMR 有两种类型的帧格式：AMR IF1 和 AMR IF2
1. AMR IF1:
 IF1 的帧格式如下图所示：
  ![Alt text | center](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/android.codecs/VS8-03-AMR IF1.jpg)

FrameType, Mode Indication, Mode Request 对应上面两个表格里的数。从上面的表格里我们可以看出，这三个域的值是相同的。所以在IF2中省略了Mode Indication, Mode Request 两个域。
Frame Quality Indicator: 0表示bad frame 或者corrupted frame； 1表示 good frame
每一帧的数据有分为三个部分：Class A/B/C
Class A：一帧中最敏感、最重要的数据。一旦这一部份数据有损坏，整个帧就无法解码，就损坏了。所以，一般在无线传输的时候要使用各种冗余的方式对这部分数据加以保护。
Class B：相对于Class A不那么重要的数据。
Class C：比Class B还不重要的数据。

2. AMR IF2:
 IF2的帧格式如下图所示：
  ![Alt text | center](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/android.codecs/VS8-04-AMR IF2.jpg)


相对于IF1, IF2 省去了Frame Quality Indicator, Mode Indication, Mode Request 和CRC 校验。但是增加了bit 填充。因为AMR帧中数据的长度并不是字节（8bit）的整数倍，所以在有些帧的末尾需要增加bit填充，以使整个帧的长度达到字节的整数倍。
有关IF2帧中各个域的信息请参考下面的帧大小节的表格。

##### 四、帧大小
1. AMR-NB
![Alt text | center](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/android.codecs/VS8-05-AMR-NB-Frame.png)


Number of bits in Classes A, B, and C for each AMR codec mode
![Alt text | center](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/android.codecs/VS8-06-AMR-NB-Frame-Number.png)

2. AMR-WB:
Composition of AMR-WB IF2 Frames for all Frame Types
![Alt text | center](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/android.codecs/VS8-07-AMR-WB-Frame.png.png)

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/android.codecs/VS8-08-AMR-WB-Frame-number.png.png)


##### 五、PCM16和AMR之间的转换
Amr 一帧为20毫秒
以AMR 4.75Kbits/s为例:

每秒产生的声音位数 = 4750bits/s
每20ms帧占用的位数 = 4750bits/s / 50frames/s = 95bits
每20ms帧占用的字节数 = 95bits / 8bits/byte = 11.875bytes - 圆整到12字节，不足的补0
加上一个字节的帧头，所以，20ms一帧的AMR: 12-bytes + 1-byte = 13-bytes

相反，转换回来就成了
13-bytes * 50frames/s * 8bits/byte = 5200bits/s

注意，这里两个数值并不对应，是由于圆整的原因


##### 六、AMR 文件的存储格式（RFC 3267）：
AMR IF1, IF2定义了 AMR的帧格式， 用于无线传输用。 RFC 3267定义了把AMR数据存成文件的文件格式。
AMR的文件格式如下图1所示：
它包含一个文件头，然后就是一帧一帧的AMR数据了。

<!--[if !supportLists]-->1.       <!--[endif]-->文件头格式：
 AMR 文件支持单声道和多声道。单声道和多声道的文件头是不同的。
 单声道：
 AMR-NB文件头： "#!AMR\n" (or 0x2321414d520a in hexadecimal)(引号内的部分)
 AMR-WB 文件头："#!AMR-WB\n" (or 0x2321414d522d57420a in hexadecimal).（引号内）
多声道：
多声道的文件头包含一个magic number和32bit channle description域。

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/android.codecs/VS8-09-AMR-magic number.jpg)


AMR-NB 的magic number："#!AMR_MC1.0\n"
(or 0x2321414d525F4D43312E300a in hexadecimal).
AMR-WB的magic number："#!AMR-WB_MC1.0\n"
                         (or 0x2321414d522d57425F4D43312E300a in hexadecimal).

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/android.codecs/VS8-10-AMR-WB-magic number.jpg)


32bit的channel description域的定义如下：

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/android.codecs/VS8-11-AMR-channel description-.jpg)


其中 reserved bits必须为0， CHAN:表示当前文件中含有几个声道。

帧头的格式：
帧头的格式如图2 所示， 它占1个字节（8个bit）
P为填充为设置为0
FT为编码模式， 即上面提到的16中编码模式。
Q为帧质量指示器，如果为0表明帧被损坏。


图3 列举了AMR-NB 5.9Kbit的一个帧的格式，
对于5.9kbit一帧的有118bit的数据，15*8=120=118+2, 所以在最后有2个bit的填充位。


#### （二）、Android AMRNB编码
我们首先看看Log：
``` cpp
	Line 12089: 07-26 14:30:22.666: V/MediaCodecList(729): matching 'OMX.google.amrnb.encoder'
	Line 12095: 07-26 14:30:22.671: V/OMXNodeInstance(702): debug level for OMX.google.amrnb.encoder is 0
	Line 12096: 07-26 14:30:22.671: I/OMXMaster(702): makeComponentInstance(OMX.google.amrnb.encoder) in mediacodec process
	Line 12097: 07-26 14:30:22.671: V/SoftOMXPlugin(702): makeComponentInstance 'OMX.google.amrnb.encoder'
	Line 12099: 07-26 14:30:22.675: V/ACodec(729): [OMX.google.amrnb.encoder] Now Loaded
	Line 12111: 07-26 14:30:22.675: E/OMXNodeInstance(702): setConfig(2be0030:google.amrnb.encoder, ConfigPriority(0x6f800002)) ERROR: Undefined(0x80001001)
	Line 12123: 07-26 14:30:22.680: E/OMXNodeInstance(702): setConfig(2be0030:google.amrnb.encoder, ConfigPriority(0x6f800002)) ERROR: Undefined(0x80001001)
	Line 12131: 07-26 14:30:22.683: V/MediaCodec(729): [OMX.google.amrnb.encoder] configured as input format: AMessage(what = 0x00000000) = {

07-26 14:30:22.683: V/MediaCodec(729): [OMX.google.amrnb.encoder] configured as input format: AMessage(what = 0x00000000) = {
07-26 14:30:22.683: V/MediaCodec(729):       string mime = "audio/raw"
07-26 14:30:22.683: V/MediaCodec(729):       int32_t channel-count = 1
07-26 14:30:22.683: V/MediaCodec(729):       int32_t sample-rate = 8000
07-26 14:30:22.683: V/MediaCodec(729):       int32_t pcm-encoding = 2
07-26 14:30:22.683: V/MediaCodec(729):     }, output format: AMessage(what = 0x00000000) = {
07-26 14:30:22.683: V/MediaCodec(729):       int32_t bitrate = 12200
07-26 14:30:22.683: V/MediaCodec(729):       int32_t max-bitrate = 12200
07-26 14:30:22.683: V/MediaCodec(729):       int32_t channel-count = 1
07-26 14:30:22.683: V/MediaCodec(729):       string mime = "audio/3gpp"
07-26 14:30:22.683: V/MediaCodec(729):       int32_t sample-rate = 8000
07-26 14:30:22.683: V/MediaCodec(729):     }


	Line 12148: 07-26 14:30:22.685: V/ACodec(729): [OMX.google.amrnb.encoder] Now Loaded->Idle
	Line 12150: 07-26 14:30:22.686: V/ACodec(729): [OMX.google.amrnb.encoder] Allocating 4 buffers of size 2048/2048 (from 2048 using Invalid) on input port
	Line 12153: 07-26 14:30:22.689: V/ACodec(729): [OMX.google.amrnb.encoder] Allocating 4 buffers of size 8192/8192 (from 8192 using Invalid) on output port
	Line 12165: 07-26 14:30:22.694: V/ACodec(729): [OMX.google.amrnb.encoder] Now Idle->Executing
	Line 12175: 07-26 14:30:22.696: V/ACodec(729): [OMX.google.amrnb.encoder] calling fillBuffer 5
	Line 12177: 07-26 14:30:22.696: V/ACodec(729): [OMX.google.amrnb.encoder] calling fillBuffer 6
	Line 12179: 07-26 14:30:22.696: V/ACodec(729): [OMX.google.amrnb.encoder] calling fillBuffer 7
	Line 12193: 07-26 14:30:22.707: V/ACodec(729): [OMX.google.amrnb.encoder] calling fillBuffer 8
	Line 12196: 07-26 14:30:22.712: V/ACodec(729): [OMX.google.amrnb.encoder] Now Executing
	Line 12311: 07-26 14:30:22.846: V/ACodec(729): [OMX.google.amrnb.encoder] calling emptyBuffer 1 w/ time 130895 us
	Line 12324: 07-26 14:30:22.866: V/ACodec(729): [OMX.google.amrnb.encoder] onOMXEmptyBufferDone 1
	Line 12325: 07-26 14:30:22.866: V/ACodec(729): [OMX.google.amrnb.encoder] calling emptyBuffer 2 w/ time 150895 us
	Line 12336: 07-26 14:30:22.868: V/ACodec(729): [OMX.google.amrnb.encoder] onOMXFillBufferDone 5 time 130895 us, flags = 0x00000010
	Line 12340: 07-26 14:30:22.869: V/ACodec(729): [OMX.google.amrnb.encoder] onOMXEmptyBufferDone 2
	Line 12341: 07-26 14:30:22.869: V/ACodec(729): [OMX.google.amrnb.encoder] onOMXFillBufferDone 6 time 150895 us, flags = 0x00000010
	Line 12342: 07-26 14:30:22.869: V/MediaCodec(729): [OMX.google.amrnb.encoder] output format changed to: AMessage(what = 0x00000000) = {
	Line 12350: 07-26 14:30:22.870: V/ACodec(729): [OMX.google.amrnb.encoder] calling fillBuffer 5
	Line 12357: 07-26 14:30:22.871: V/ACodec(729): [OMX.google.amrnb.encoder] calling fillBuffer 6
	Line 12363: 07-26 14:30:22.886: V/ACodec(729): [OMX.google.amrnb.encoder] calling emptyBuffer 3 w/ time 170895 us
	Line 12371: 07-26 14:30:22.888: V/ACodec(729): [OMX.google.amrnb.encoder] onOMXEmptyBufferDone 3
	Line 12372: 07-26 14:30:22.888: V/ACodec(729): [OMX.google.amrnb.encoder] onOMXFillBufferDone 7 time 170895 us, flags = 0x00000010
	Line 12374: 07-26 14:30:22.889: V/ACodec(729): [OMX.google.amrnb.encoder] calling fillBuffer 7
	Line 12380: 07-26 14:30:22.906: V/ACodec(729): [OMX.google.amrnb.encoder] calling emptyBuffer 4 w/ time 190895 us
	Line 12388: 07-26 14:30:22.907: V/ACodec(729): [OMX.google.amrnb.encoder] onOMXEmptyBufferDone 4
```
可以看到博主当前手机录制的编码格式为amrnb。先看看流程图。
![Alt text | center](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/android.codecs/VS8-12-AMRNB-Encorder-flow.png)


##### 1.1、 初始化AMRNB编码器 SoftAMRNBEncoder::initEncoder()

``` cpp
[->Z:\MediaLearing\frameworks\av\media\libstagefright\codecs\amrnb\enc\SoftAMRNBEncoder.cpp]
status_t SoftAMRNBEncoder::initEncoder() {
    if (AMREncodeInit(&mEncState, &mSidState, false /* dtx_enable */) != 0) {
        return UNKNOWN_ERROR;
    }

    return OK;
}

Z:\MediaLearing\frameworks\av\media\libstagefright\codecs\amrnb\enc\src\amrencode.cpp
Word16 AMREncodeInit(
    void **pEncStructure,
    void **pSidSyncStructure,
    Flag dtx_enable)
{
    Word16 enc_init_status = 0;
    Word16 sid_sync_init_status = 0;
    Word16 init_status = 0;

    /* Initialize GSM AMR Encoder */
#ifdef CONSOLE_ENCODER_REF
    /* Change to original ETS input types */
    Speech_Encode_FrameState **speech_encode_frame =
        (Speech_Encode_FrameState **)(pEncStructure);

    sid_syncState **sid_sync_state = (sid_syncState **)(pSidSyncStructure);

    /* Use ETS version of sp_enc.c */
    enc_init_status = Speech_Encode_Frame_init(speech_encode_frame,
                      dtx_enable,
                      (Word8*)"encoder");
    /* Initialize SID synchronization */
    sid_sync_init_status = sid_sync_init(sid_sync_state);

#else
    /* Use PV version of sp_enc.c */
    enc_init_status = GSMInitEncode(pEncStructure,
                                    dtx_enable,
                                    (Word8*)"encoder");
    /* Initialize SID synchronization */
    sid_sync_init_status = sid_sync_init(pSidSyncStructure);
#endif

    if ((enc_init_status != 0) || (sid_sync_init_status != 0))
    {
        init_status = -1;
    }

    return(init_status);
}
```

##### 1.2、编码AMREncode()


``` cpp
[->Z:\MediaLearing\frameworks\av\media\libstagefright\codecs\amrnb\enc\src\amrencode.cpp]
Word16 AMREncode(
    void *pEncState,
    void *pSidSyncState,
    enum Mode mode,
    Word16 *pEncInput,
    UWord8 *pEncOutput,
    enum Frame_Type_3GPP *p3gpp_frame_type,
    Word16 output_format
)
{
    Word16 ets_output_bfr[MAX_SERIAL_SIZE+2];
    UWord8 *ets_output_ptr;
    Word16 num_enc_bytes = -1;
    Word16 i;
    enum TXFrameType tx_frame_type;
    enum Mode usedMode = MR475;

    /* Encode WMF or IF2 frames */
    if ((output_format == AMR_TX_WMF) | (output_format == AMR_TX_IF2))
    {
        /* Encode one speech frame (20 ms) */

#ifndef CONSOLE_ENCODER_REF

        /* Use PV version of sp_enc.c */
        GSMEncodeFrame(pEncState, mode, pEncInput, ets_output_bfr, &usedMode);

#else
        /* Use ETS version of sp_enc.c */
        Speech_Encode_Frame(pEncState, mode, pEncInput, ets_output_bfr, &usedMode);

#endif

        /* Determine transmit frame type */
        sid_sync(pSidSyncState, usedMode, &tx_frame_type);

        if (tx_frame_type != TX_NO_DATA)
        {
            /* There is data to transmit */
            *p3gpp_frame_type = (enum Frame_Type_3GPP) usedMode;

            /* Add SID type and mode info for SID frames */
            if (*p3gpp_frame_type == AMR_SID)
            {
                /* Add SID type to encoder output buffer */
                if (tx_frame_type == TX_SID_FIRST)
                {
                    ets_output_bfr[AMRSID_TXTYPE_BIT_OFFSET] &= 0x0000;
                }
                else if (tx_frame_type == TX_SID_UPDATE)
                {
                    ets_output_bfr[AMRSID_TXTYPE_BIT_OFFSET] |= 0x0001;
                }

                /* Add mode information bits */
                for (i = 0; i < NUM_AMRSID_TXMODE_BITS; i++)
                {
                    ets_output_bfr[AMRSID_TXMODE_BIT_OFFSET+i] =
                        (mode >> i) & 0x0001;
                }
            }
        }
        else
        {
            /* This is no data to transmit */
            *p3gpp_frame_type = (enum Frame_Type_3GPP)AMR_NO_DATA;
        }

        /* At this point, output format is ETS */
        /* Determine the output format to use */
        if (output_format == AMR_TX_WMF)
        {
            /* Change output data format to WMF */
            ets_to_wmf(*p3gpp_frame_type, ets_output_bfr, pEncOutput);

            /* Set up the number of encoded WMF bytes */
            num_enc_bytes = WmfEncBytesPerFrame[(Word16) *p3gpp_frame_type];

        }
        else if (output_format == AMR_TX_IF2)
        {
            /* Change output data format to IF2 */
            ets_to_if2(*p3gpp_frame_type, ets_output_bfr, pEncOutput);

            /* Set up the number of encoded IF2 bytes */
            num_enc_bytes = If2EncBytesPerFrame[(Word16) *p3gpp_frame_type];

        }
    }

    /* Encode ETS frames */
    else if (output_format == AMR_TX_ETS)
    {
        /* Encode one speech frame (20 ms) */

#ifndef CONSOLE_ENCODER_REF

        /* Use PV version of sp_enc.c */
        GSMEncodeFrame(pEncState, mode, pEncInput, &ets_output_bfr[1], &usedMode);

#else
        /* Use ETS version of sp_enc.c */
        Speech_Encode_Frame(pEncState, mode, pEncInput, &ets_output_bfr[1], &usedMode);

#endif

        /* Save used mode */
        *p3gpp_frame_type = (enum Frame_Type_3GPP) usedMode;

        /* Determine transmit frame type */
        sid_sync(pSidSyncState, usedMode, &tx_frame_type);

        /* Put TX frame type in output buffer */
        ets_output_bfr[0] = tx_frame_type;

        /* Put mode information after the encoded speech parameters */
        if (tx_frame_type != TX_NO_DATA)
        {
            ets_output_bfr[1+MAX_SERIAL_SIZE] = (Word16) mode;
        }
        else
        {
            ets_output_bfr[1+MAX_SERIAL_SIZE] = -1;
        }

        /* Copy output of encoder to pEncOutput buffer */
        ets_output_ptr = (UWord8 *) & ets_output_bfr[0];

        /* Copy 16-bit data in 8-bit chunks  */
        /* using Little Endian configuration */
        for (i = 0; i < 2*(MAX_SERIAL_SIZE + 2); i++)
        {
            *(pEncOutput + i) = *ets_output_ptr;
            ets_output_ptr += 1;
        }

        /* Set up the number of encoded bytes */
        num_enc_bytes = 2 * (MAX_SERIAL_SIZE + 2);

    }

    /* Invalid frame format */
    else
    {
        /* Invalid output format, set up error code */
        num_enc_bytes = -1;
    }

    return(num_enc_bytes);
}

```

#### （三）、Android AMRNB 解码
``` cpp
07-26 14:52:55.610: V/MediaCodec(701): [OMX.google.amrnb.decoder] configured as input format: AMessage(what = 0x00000000) = {
07-26 14:52:55.610: V/MediaCodec(701):       int32_t channel-count = 1
07-26 14:52:55.610: V/MediaCodec(701):       string mime = "audio/3gpp"
07-26 14:52:55.610: V/MediaCodec(701):       int32_t sample-rate = 8000
07-26 14:52:55.610: V/MediaCodec(701):     }, output format: AMessage(what = 0x00000000) = {
07-26 14:52:55.610: V/MediaCodec(701):       string mime = "audio/raw"
07-26 14:52:55.610: V/MediaCodec(701):       int32_t channel-count = 1
07-26 14:52:55.610: V/MediaCodec(701):       int32_t sample-rate = 8000
07-26 14:52:55.610: V/MediaCodec(701):       int32_t pcm-encoding = 2
07-26 14:52:55.610: V/MediaCodec(701):     }
07-26 14:52:55.611: I/MediaCodec(701): MediaCodec will operate in async mode
```

以上是解码初始化信息，先看看流程图
![Alt text | center](https://raw.githubusercontent.com/zhoujinjiancc/zhoujinjian.com.images/master/android.codecs/VS8-13-AMRNB-Decorder-flow.png)



``` cpp
Z:\MediaLearing\frameworks\av\media\libstagefright\codecs\amrnb\dec\src\amrdecode.cpp

Word16 AMRDecode(
    void                      *state_data,
    enum Frame_Type_3GPP      frame_type,
    UWord8                    *speech_bits_ptr,
    Word16                    *raw_pcm_buffer,
    bitstream_format          input_format
)
{
    Word16 *ets_word_ptr;
    enum Mode mode = (enum Mode)MR475;
    int modeStore;
    int tempInt;
    enum RXFrameType rx_type = RX_NO_DATA;
    Word16 dec_ets_input_bfr[MAX_SERIAL_SIZE];
    Word16 i;
    Word16 byte_offset = -1;

    /* Type cast state_data to Speech_Decode_FrameState rather than passing
     * that structure type to this function so the structure make up can't
     * be viewed from higher level functions than this.
     */
    Speech_Decode_FrameState *decoder_state
    = (Speech_Decode_FrameState *) state_data;

    /* Determine type of de-formatting */
    /* WMF or IF2 frames */
    if ((input_format == MIME_IETF) | (input_format == IF2))
    {
        if (input_format == MIME_IETF)
        {
            /* Convert incoming packetized raw WMF data to ETS format */
            wmf_to_ets(frame_type, speech_bits_ptr, dec_ets_input_bfr);

            /* Address offset of the start of next frame */
            byte_offset = WmfDecBytesPerFrame[frame_type];
        }
        else   /* else has to be input_format  IF2 */
        {
            /* Convert incoming packetized raw IF2 data to ETS format */
            if2_to_ets(frame_type, speech_bits_ptr, dec_ets_input_bfr);

            /* Address offset of the start of next frame */
            byte_offset = If2DecBytesPerFrame[frame_type];
        }

        /* At this point, input data is in ETS format     */
        /* Determine AMR codec mode and AMR RX frame type */
        if (frame_type <= AMR_122)
        {
            mode = (enum Mode) frame_type;
            rx_type = RX_SPEECH_GOOD;
        }
        else if (frame_type == AMR_SID)
        {
            /* Clear mode store prior to reading mode info from input buffer */
            modeStore = 0;

            for (i = 0; i < NUM_AMRSID_RXMODE_BITS; i++)
            {
                tempInt = dec_ets_input_bfr[AMRSID_RXMODE_BIT_OFFSET+i] << i;
                modeStore |= tempInt;
            }
            mode = (enum Mode) modeStore;

            /* Get RX frame type */
            if (dec_ets_input_bfr[AMRSID_RXTYPE_BIT_OFFSET] == 0)
            {
                rx_type = RX_SID_FIRST;
            }
            else
            {
                rx_type = RX_SID_UPDATE;
            }
        }
        else if (frame_type < AMR_NO_DATA)
        {
            /* Invalid frame_type, return error code */
            byte_offset = -1;   /*  !!! */
        }
        else
        {
            mode = decoder_state->prev_mode;

            /*
             * RX_NO_DATA, generate exponential decay from latest valid frame for the first 6 frames
             * after that, create silent frames
             */
            rx_type = RX_NO_DATA;

        }

    }

    /* ETS frames */
    else if (input_format == ETS)
    {
        /* Change type of pointer to incoming raw ETS data */
        ets_word_ptr = (Word16 *) speech_bits_ptr;

        /* Get RX frame type */
        rx_type = (enum RXFrameType) * ets_word_ptr;
        ets_word_ptr++;

        /* Copy incoming raw ETS data to dec_ets_input_bfr */
        for (i = 0; i < MAX_SERIAL_SIZE; i++)
        {
            dec_ets_input_bfr[i] = *ets_word_ptr;
            ets_word_ptr++;
        }

        /* Get codec mode */
        if (rx_type != RX_NO_DATA)
        {
            /* Get mode from input bitstream */
            mode = (enum Mode) * ets_word_ptr;
        }
        else
        {
            /* Use previous mode if no received data */
            mode = decoder_state->prev_mode;
        }

        /* Set up byte_offset */
        byte_offset = 2 * (MAX_SERIAL_SIZE + 2);
    }
    else
    {
        /* Invalid input format, return error code */
        byte_offset = -1;
    }

    /* Proceed with decoding frame, if there are no errors */
    if (byte_offset != -1)
    {
        /* Decode a 20 ms frame */

#ifndef CONSOLE_DECODER_REF
        /* Use PV version of sp_dec.c */
        GSMFrameDecode(decoder_state, mode, dec_ets_input_bfr, rx_type,
                       raw_pcm_buffer);

#else
        /* Use ETS version of sp_dec.c */
        Speech_Decode_Frame(decoder_state, mode, dec_ets_input_bfr, rx_type,
                            raw_pcm_buffer);

#endif

        /* Save mode for next frame */
        decoder_state->prev_mode = mode;
    }

    return (byte_offset);
}
```


``` cpp
Z:\MediaLearing\frameworks\av\media\libstagefright\codecs\amrnb\dec\src\sp_dec.cpp

void GSMFrameDecode(
    Speech_Decode_FrameState *st, /* io: post filter states                */
    enum Mode mode,               /* i : AMR mode                          */
    Word16 *serial,               /* i : serial bit stream                 */
    enum RXFrameType frame_type,  /* i : Frame type                        */
    Word16 *synth)                /* o : synthesis speech (postfiltered    */
/*     output)                           */

{
    Word16 parm[MAX_PRM_SIZE + 1];  /* Synthesis parameters                */
    Word16 Az_dec[AZ_SIZE];         /* Decoded Az for post-filter          */
    /* in 4 subframes                      */
    Flag *pOverflow = &(st->decoder_amrState.overflow);  /* Overflow flag  */

#if !defined(NO13BIT)
    Word16 i;
#endif

    /* Serial to parameters   */
    if ((frame_type == RX_SID_BAD) ||
            (frame_type == RX_SID_UPDATE))
    {
        /* Override mode to MRDTX */
        Bits2prm(MRDTX, serial, parm);
    }
    else
    {
        Bits2prm(mode, serial, parm);
    }

    /* Synthesis */
    Decoder_amr(
        &(st->decoder_amrState),
        mode,
        parm,
        frame_type,
        synth,
        Az_dec);

    /* Post-filter */
    Post_Filter(
        &(st->post_state),
        mode,
        synth,
        Az_dec,
        pOverflow);

    /* post HP filter, and 15->16 bits */
    Post_Process(
        &(st->postHP_state),
        synth,
        L_FRAME,
        pOverflow);

#if !defined(NO13BIT)
    /* Truncate to 13 bits */
    for (i = 0; i < L_FRAME; i++)
    {
        synth[i] = synth[i] & 0xfff8;
    }
#endif

    return;
}

```

#### （四）、AMRNB、AMRWB编解码库分析
**Todo---**

>PATH : \frameworks\av\media\libstagefright\codecs （Android Codecs）
- amrnb
- amrwb
- amrwbenc
#### （五）、参考资料(特别感谢各位前辈的分析和图示)：
[AMR音频编码器概述及文件格式分析](https://blog.csdn.net/lusonglin121/article/details/9332823)

