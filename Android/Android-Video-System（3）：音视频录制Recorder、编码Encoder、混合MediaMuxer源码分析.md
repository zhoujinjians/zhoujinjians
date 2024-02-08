---
title: Android Video System（3）：音视频录制Recorder、编码Encoder、混合MediaMuxer源码分析
cover: https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/hexo.themes/bing-wallpaper-2018.04.24.jpg
categories:
  - Multimedia
tags:
  - Android
  - Video
  - Multimedia
toc: true
abbrlink: 20190210
date: 2019-02-10 09:25:00
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
☯ /vendor/qcom/proprietary/mm-video/omx_vpp(?) → 解码器代码
☯ /vendor/qcom/proprietary/mm-video/omx_vpp(?) → 编码器代码

--------------------------------------------------------------------------------
首先看一下使用MediaRecorder 录制音频的Java实例：

``` java
 MediaRecorder recorder=newMediaRecorder();
 recorder.setAudioSource(MediaRecorder.AudioSource.MIC);
 recorder.setOutputFormat(MediaRecorder.OutputFormat.THREE_GPP);
 recorder.setAudioEncoder(MediaRecorder.AudioEncoder.AMR_NB);
 recorder.setOutputFile(PATH_NAME);
 recorder.prepare();
 recorder.start();  // Recording is now started
 ...
 recorder.stop();  //
 recorder.reset();  // You can reuse the object by going back to setAudioSource() step
 recorder.release();// Now the object cannot be reused
```

之前在[Audio System（2）：Linux ALSA音频系统分析]() 第（八）节画过tinyplay capture录音时序图，但当时没有仔细分析，今天来分析Audio录音如何从Java层一步步最终到达tinyalsa层的pcm_open()、pcm_read()函数的

``` cpp
[->E:\android-7.1.2_r1\external\tinyalsa\tinycap.c]
unsigned int capture_sample(FILE *file, unsigned int card, unsigned int device,
                            unsigned int channels, unsigned int rate,
                            enum pcm_format format, unsigned int period_size,
                            unsigned int period_count)
{
    struct pcm_config config;
    struct pcm *pcm;
    char *buffer;
    unsigned int size;
    unsigned int bytes_read = 0;

    memset(&config, 0, sizeof(config));
    config.channels = channels;
    config.rate = rate;
    config.period_size = period_size;
    config.period_count = period_count;
    config.format = format;
    config.start_threshold = 0;
    config.stop_threshold = 0;
    config.silence_threshold = 0;
    //打开录制节点
    pcm = pcm_open(card, device, PCM_IN, &config);
    ......
}
```
并读取从Kernel内核传过来的数据最终合成音频文件的过程
``` cpp
[->E:\android-7.1.2_r1\external\tinyalsa\tinycap.c]
unsigned int capture_sample(FILE *file, unsigned int card, unsigned int device,
                            unsigned int channels, unsigned int rate,
                            enum pcm_format format, unsigned int period_size,
                            unsigned int period_count)
{
    ......
    //循环读取音频数据
    size = pcm_frames_to_bytes(pcm, pcm_get_buffer_size(pcm));
    buffer = malloc(size);
        while (capturing && !pcm_read(pcm, buffer, size)) {
        if (fwrite(buffer, 1, size, file) != size) {
            fprintf(stderr,"Error capturing sample\n");
            break;
        }
        bytes_read += size;
    }
    
```
开始分析音频录制编码合成之旅，之后分析视频录制编码合成过程。

#### （一）、Audio Recorder 音频录制源码分析
![Alt text](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/video.system/VS-03-01-MediaRecorder-flow.png)

#### （二）、Media Recorder 视频录制源码分析

``` java
		mCamera = getCameraInstance();
		mCamera .open()
		mCamera.startPreview() 

        mMediaRecorder = new MediaRecorder();
        mMediaRecorder.setCamera(mCamera);
 
        mMediaRecorder.setPreviewDisplay(android.view.SurfaceHolder);
        mMediaRecorder.setAudioSource(MediaRecorder.AudioSource.CAMCORDER);
        mMediaRecorder.setVideoSource(MediaRecorder.VideoSource.CAMERA);

        mMediaRecorder.setVideoSize(videoRecordSize.width, videoRecordSize.height);
		mMediaRecorder.setProfile(CamcorderProfile.get(CamcorderProfile.QUALITY_HIGH));
        mMediaRecorder.setOutputFile(outputFile.toString());
        //MediaRecorder.OutputFormat.MPEG_4.

        mMediaRecorder.setAudioEncoder(MediaRecorder.AudioEncoder.AMR_NB)
        mMediaRecorder.setVideoEncoder(MediaRecorder.VideoEncoder.MPEG_4_SP)
        
        mMediaRecorder.prepare();
        mMediaRecorder.start();

		mMediaRecorder.stop()
		mMediaRecorder.release()
		mCamera.stopPreview()
```


![Alt text](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/video.system/VS-03-02-MediaRecorder-video-flow.png)

#### （三）、音频Recorder编码Encoder

``` cpp
	Line 3036: 07-02 17:00:07.175  8430  8479 V ACodec  : Now uninitialized
	Line 3674: 07-02 17:00:35.717   745  1949 V ACodec  : Now uninitialized
	Line 3675: 07-02 17:00:35.718   745  8649 V ACodec  : onAllocateComponent
	Line 3678: 07-02 17:00:35.721   731   921 I OMXMaster: makeComponentInstance(OMX.google.amrnb.encoder) in mediacodec process
	Line 3679: 07-02 17:00:35.742   745  8649 V ACodec  : [OMX.google.amrnb.encoder] Now Loaded
	Line 3680: 07-02 17:00:35.742   745  8649 I MediaCodec: MediaCodec will operate in async mode
	Line 3680: 07-02 17:00:35.742   745  8649 I MediaCodec: MediaCodec will operate in async mode
	Line 3681: 07-02 17:00:35.742   745  8649 V MediaCodec: Found 0 pieces of codec specific data.
	Line 3682: 07-02 17:00:35.742   745  8649 V ACodec  : onConfigureComponent
	Line 3684: 07-02 17:00:35.744   745  8649 I ACodec  : codec does not support config priority (err -2147483648)
	Line 3686: 07-02 17:00:35.751   745  8649 I ACodec  : codec does not support config priority (err -2147483648)
	Line 3687: 07-02 17:00:35.755   745  8649 V MediaCodec: [OMX.google.amrnb.encoder] configured as input format: AMessage(what = 0x00000000) = {
	Line 3688: 07-02 17:00:35.755   745  8649 V MediaCodec:       string mime = "audio/raw"
	Line 3689: 07-02 17:00:35.755   745  8649 V MediaCodec:       int32_t channel-count = 1
	Line 3690: 07-02 17:00:35.755   745  8649 V MediaCodec:       int32_t sample-rate = 8000
	Line 3691: 07-02 17:00:35.755   745  8649 V MediaCodec:       int32_t pcm-encoding = 2
	Line 3692: 07-02 17:00:35.755   745  8649 V MediaCodec:     }, output format: AMessage(what = 0x00000000) = {
	Line 3693: 07-02 17:00:35.755   745  8649 V MediaCodec:       int32_t bitrate = 12200
	Line 3694: 07-02 17:00:35.755   745  8649 V MediaCodec:       int32_t max-bitrate = 12200
	Line 3695: 07-02 17:00:35.755   745  8649 V MediaCodec:       int32_t channel-count = 1
	Line 3696: 07-02 17:00:35.755   745  8649 V MediaCodec:       string mime = "audio/3gpp"
	Line 3697: 07-02 17:00:35.755   745  8649 V MediaCodec:       int32_t sample-rate = 8000
	Line 3698: 07-02 17:00:35.755   745  8649 V MediaCodec:     }
	Line 3699: 07-02 17:00:35.757   745  8649 V ACodec  : onStart
	Line 3700: 07-02 17:00:35.758   745  8649 V ACodec  : [OMX.google.amrnb.encoder] Now Loaded->Idle
	Line 3701: 07-02 17:00:35.758   745  8649 V ACodec  : [OMX.google.amrnb.encoder] Allocating 4 buffers of size 2048/2048 (from 2048 using Invalid) on input port
	Line 3702: 07-02 17:00:35.766   745  8649 V ACodec  : [OMX.google.amrnb.encoder] Allocating 4 buffers of size 8192/8192 (from 8192 using Invalid) on output port
	Line 3703: 07-02 17:00:35.771   745  8649 V MediaCodec: input buffers allocated
	Line 3704: 07-02 17:00:35.771   745  8649 V MediaCodec: output buffers allocated
	Line 3705: 07-02 17:00:35.772   745  8646 I MediaCodecSource: MediaCodecSource (audio) starting
    Line 3706: 07-02 17:00:35.772   745  8649 V ACodec  : [OMX.google.amrnb.encoder] Now Idle->Executing
	Line 3701: 07-02 17:00:35.758   745  8649 V ACodec  : [OMX.google.amrnb.encoder] Allocating 4 buffers of size 2048/2048 (from 2048 using Invalid) on input port
	Line 3702: 07-02 17:00:35.766   745  8649 V ACodec  : [OMX.google.amrnb.encoder] Allocating 4 buffers of size 8192/8192 (from 8192 using Invalid) on output port
	Line 3706: 07-02 17:00:35.772   745  8649 V ACodec  : [OMX.google.amrnb.encoder] Now Idle->Executing
	Line 3707: 07-02 17:00:35.773   745  8649 V ACodec  : [OMX.google.amrnb.encoder] calling fillBuffer 5
	Line 3708: 07-02 17:00:35.773   745  8649 V ACodec  : [OMX.google.amrnb.encoder] calling fillBuffer 6
	Line 3711: 07-02 17:00:35.775   745  8649 V ACodec  : [OMX.google.amrnb.encoder] calling fillBuffer 7
	Line 3715: 07-02 17:00:35.775   745  8649 V ACodec  : [OMX.google.amrnb.encoder] calling fillBuffer 8
	Line 3717: 07-02 17:00:35.776   745  8649 V ACodec  : [OMX.google.amrnb.encoder] Now Executing
	Line 3833: 07-02 17:00:35.902   745  8649 V ACodec  : [OMX.google.amrnb.encoder] calling emptyBuffer 1 w/ time 20000 us
	Line 3834: 07-02 17:00:35.904   745  8649 V ACodec  : [OMX.google.amrnb.encoder] onOMXEmptyBufferDone 1
	Line 3835: 07-02 17:00:35.907   745  8649 V ACodec  : [OMX.google.amrnb.encoder] onOMXFillBufferDone 5 time 20000 us, flags = 0x00000010
	Line 3836: 07-02 17:00:35.911   745  8649 V MediaCodec: [OMX.google.amrnb.encoder] output format changed to: AMessage(what = 0x00000000) = {
	Line 3843: 07-02 17:00:35.913   745  8649 V ACodec  : [OMX.google.amrnb.encoder] calling fillBuffer 5
	Line 3844: 07-02 17:00:35.925   745  8649 V ACodec  : [OMX.google.amrnb.encoder] calling emptyBuffer 2 w/ time 40000 us
	Line 3845: 07-02 17:00:35.926   745  8649 V ACodec  : [OMX.google.amrnb.encoder] onOMXEmptyBufferDone 2
	Line 3846: 07-02 17:00:35.927   745  8649 V ACodec  : [OMX.google.amrnb.encoder] onOMXFillBufferDone 6 time 40000 us, flags = 0x00000010
	Line 3847: 07-02 17:00:35.927   745  8649 V ACodec  : [OMX.google.amrnb.encoder] calling fillBuffer 6
	Line 3848: 07-02 17:00:35.942   745  8649 V ACodec  : [OMX.google.amrnb.encoder] calling emptyBuffer 3 w/ time 60000 us
	Line 3849: 07-02 17:00:35.945   745  8649 V ACodec  : [OMX.google.amrnb.encoder] onOMXEmptyBufferDone 3
	Line 3850: 07-02 17:00:35.945   745  8649 V ACodec  : [OMX.google.amrnb.encoder] onOMXFillBufferDone 7 time 60000 us, flags = 0x00000010
	Line 3851: 07-02 17:00:35.946   745  8649 V ACodec  : [OMX.google.amrnb.encoder] calling fillBuffer 7
	Line 3852: 07-02 17:00:35.963   745  8649 V ACodec  : [OMX.google.amrnb.encoder] calling emptyBuffer 4 w/ time 80000 us
	Line 3853: 07-02 17:00:35.965   745  8649 V ACodec  : [OMX.google.amrnb.encoder] onOMXEmptyBufferDone 4
	Line 3854: 07-02 17:00:35.966   745  8649 V ACodec  : [OMX.google.amrnb.encoder] onOMXFillBufferDone 8 time 80000 us, flags = 0x00000010
	Line 3855: 07-02 17:00:35.967   745  8649 V ACodec  : [OMX.google.amrnb.encoder] calling fillBuffer 8
	Line 3856: 07-02 17:00:35.982   745  8649 V ACodec  : [OMX.google.amrnb.encoder] calling emptyBuffer 1 w/ time 100000 us
	Line 3857: 07-02 17:00:35.983   745  8649 V ACodec  : [OMX.google.amrnb.encoder] onOMXEmptyBufferDone 1
	Line 3858: 07-02 17:00:35.985   745  8649 V ACodec  : [OMX.google.amrnb.encoder] onOMXFillBufferDone 5 time 100000 us, flags = 0x00000010
......
	Line 8217: 07-02 17:00:56.446   745  8649 V ACodec  : [OMX.google.amrnb.encoder] calling fillBuffer 8
	Line 8218: 07-02 17:00:56.462   745  8649 V ACodec  : [OMX.google.amrnb.encoder] calling emptyBuffer 1 w/ time 20580000 us
	Line 8219: 07-02 17:00:56.464   745  8649 V ACodec  : [OMX.google.amrnb.encoder] onOMXEmptyBufferDone 1
	Line 8220: 07-02 17:00:56.466   745  8649 V ACodec  : [OMX.google.amrnb.encoder] onOMXFillBufferDone 5 time 20580000 us, flags = 0x00000010
	Line 8221: 07-02 17:00:56.467   745  8649 V ACodec  : [OMX.google.amrnb.encoder] calling fillBuffer 5
	Line 8222: 07-02 17:00:56.468   745  8646 I MediaCodecSource: encoder (audio) stopping
	Line 8223: 07-02 17:00:56.483   745  8649 V ACodec  : [OMX.google.amrnb.encoder] Now Executing->Idle
	Line 8224: 07-02 17:00:56.484   745  8649 V ACodec  : [OMX.google.amrnb.encoder] onOMXFillBufferDone 5 time 20580000 us, flags = 0x00000000
	Line 8225: 07-02 17:00:56.484   745  8649 V ACodec  : [OMX.google.amrnb.encoder] onOMXFillBufferDone 6 time 20520000 us, flags = 0x00000000
	Line 8226: 07-02 17:00:56.484   745  8649 V ACodec  : [OMX.google.amrnb.encoder] onOMXFillBufferDone 7 time 20540000 us, flags = 0x00000000
	Line 8227: 07-02 17:00:56.484   745  8649 V ACodec  : [OMX.google.amrnb.encoder] onOMXFillBufferDone 8 time 20560000 us, flags = 0x00000000
	Line 8228: 07-02 17:00:56.496   745  8649 V ACodec  : [OMX.google.amrnb.encoder] Now Idle->Loaded
	Line 8229: 07-02 17:00:56.496   745  8649 V ACodec  : [OMX.google.amrnb.encoder] Now Loaded
	Line 8231: 07-02 17:00:56.499   745  8646 I MediaCodecSource: encoder (audio) stopped
```
#### （四）、视频Recorder编码Encoder

``` cpp
07-03 11:43:44.947  6238  6238 V CAM_VideoModule: startVideoRecording
07-03 11:43:44.974  6238  6238 D CameraStorage: External storage state=mounted
07-03 11:43:44.995  6238  6238 D CameraStorage: External storage state=mounted
07-03 11:43:44.998  6238  6238 V CAM_VideoModule: initializeRecorder
07-03 11:43:45.003   741  1966 V MediaPlayerService: Create new media recorder client from pid 6238
07-03 11:43:45.009  6238  6238 I CAM_VideoModule: NOTE: hfr = off : hsr = off
07-03 11:43:45.024   802  6282 E mm-camera: <ISP   ><ERROR> 378: tintless40_algo_process_be: failed: update_func rc -4
07-03 11:43:45.024   802  6282 E mm-camera: <ISP   ><ERROR> 851: tintless40_algo_execute: failed: tintless40_trigger_algo
07-03 11:43:45.024   802  6282 E mm-camera: <ISP   ><ERROR> 98: isp_algo_execute_internal_algo: failed to run algo tintless
07-03 11:43:45.024   802  6282 E mm-camera: <ISP   ><ERROR> 710: isp_parser_thread_func: failed: isp_parser_process
07-03 11:43:45.031  6238  6238 D LocationManager: No location received yet.
07-03 11:43:45.033  6238  6238 D LocationManager: No location received yet.
07-03 11:43:45.033  6238  6238 V CAM_VideoModule: New video filename: /storage/emulated/0/DCIM/Camera/VID_20180703_114345.mp4
07-03 11:43:45.059  1391  2526 I MediaFocusControl:  AudioFocus  requestAudioFocus() from uid/pid 10025/6238 clientId=android.media.AudioManager@11f939d req=2 flags=0x0
07-03 11:43:45.062  4966  4966 D AudioManager: AudioManager dispatching onAudioFocusChange(-2) for android.media.AudioManager@e6fb066com.android.music.MediaPlaybackService$4@2315fa7
07-03 11:43:45.063  4966  4966 V MediaPlaybackService: AudioFocus: received AUDIOFOCUS_LOSS_TRANSIENT
07-03 11:43:45.079   726  6266 E QCamera : <HAL><ERROR> status_t qcamera::QCameraParameters::setSkinBeautify(const qcamera::QCameraParameters &): 15184: gpw status_t qcamera::QCameraParameters::setSkinBeautify(const qcamera::QCameraParameters &): str=off , prev_str=off
07-03 11:43:45.079   726  6266 E QCamera : <HAL><ERROR> int32_t qcamera::QCameraParameters::setAjustLevel(const qcamera::QCameraParameters &): 15302: gpw   -1 -1 -1 -1 -1
07-03 11:43:45.079   726  6266 E QCamera : <HAL><ERROR> int32_t qcamera::QCameraParameters::setAjustLevel(int, int, int, int, int): 15232: ggw3 -1 -1  
07-03 11:43:45.091   726  2209 E CameraClient: setVideoBufferMode: 535: videoBufferMode 2 is not supported.
07-03 11:43:45.100   802  6282 E mm-camera: <ISP   ><ERROR> 378: tintless40_algo_process_be: failed: update_func rc -4
07-03 11:43:45.101   802  6282 E mm-camera: <ISP   ><ERROR> 851: tintless40_algo_execute: failed: tintless40_trigger_algo
07-03 11:43:45.101   802  6282 E mm-camera: <ISP   ><ERROR> 98: isp_algo_execute_internal_algo: failed to run algo tintless
07-03 11:43:45.101   802  6282 E mm-camera: <ISP   ><ERROR> 710: isp_parser_thread_func: failed: isp_parser_process
07-03 11:43:45.105   802  6296 E mm-camera: <IMGLIB><ERROR> 318: faceproc_comp_set_param: Error param=523
07-03 11:43:45.111   726  6266 E QCamera : <HAL><ERROR> status_t qcamera::QCameraParameters::setSkinBeautify(const qcamera::QCameraParameters &): 15184: gpw status_t qcamera::QCameraParameters::setSkinBeautify(const qcamera::QCameraParameters &): str=off , prev_str=off
07-03 11:43:45.111   726  6266 E QCamera : <HAL><ERROR> int32_t qcamera::QCameraParameters::setAjustLevel(const qcamera::QCameraParameters &): 15302: gpw   -1 -1 -1 -1 -1
07-03 11:43:45.111   726  6266 E QCamera : <HAL><ERROR> int32_t qcamera::QCameraParameters::setAjustLevel(int, int, int, int, int): 15232: ggw3 -1 -1  
07-03 11:43:45.121   741  6349 I MediaPlayerService: MediaPlayerService::getOMX
07-03 11:43:45.122   741  6349 I OMXClient: MuxOMX ctor

	Line 5362: 07-03 11:43:41.712  6238  6238 V CAM_VideoModule: Video Encoder selected = 2
	Line 5363: 07-03 11:43:41.712  6238  6238 V CAM_VideoModule: Audio Encoder selected = 3
	Line 5449: 07-03 11:43:41.944  6238  6238 V CAM_VideoModule: Video Encoder selected = 2
	Line 5450: 07-03 11:43:41.944  6238  6238 V CAM_VideoModule: Audio Encoder selected = 3
	Line 6011: 07-03 11:43:45.123   732  2029 I OMXMaster: makeComponentInstance(OMX.qcom.video.encoder.avc) in mediacodec process
	Line 6014: 07-03 11:43:45.170   732  2029 I OMX-VENC: Component_init : OMX.qcom.video.encoder.avc : return = 0x0
	Line 6017: 07-03 11:43:45.180   732  2168 E OMXNodeInstance: getParameter(2dc0040:qcom.encoder.avc, ParamConsumerUsageBits(0x6f800004)) ERROR: UnsupportedIndex(0x8000101a)
	Line 6019: 07-03 11:43:45.181   732  1863 W OMXNodeInstance: [2dc0040:qcom.encoder.avc] component does not support metadata mode; using fallback
	Line 6020: 07-03 11:43:45.181   741  6349 E ACodec  : [OMX.qcom.video.encoder.avc] storeMetaDataInBuffers (output) failed w/ err -1010
	Line 6021: 07-03 11:43:45.182   741  6349 I ExtendedACodec: setupVideoEncoder()
	Line 6026: 07-03 11:43:45.231   741  6349 I ACodec  : setupAVCEncoderParameters with [profile: Baseline] [level: Level1]
	Line 6028: 07-03 11:43:45.241   732  1863 E OMXNodeInstance: getConfig(2dc0040:qcom.encoder.avc, ??(0x7f000062)) ERROR: UnsupportedSetting(0x80001019)
	Line 6029: 07-03 11:43:45.245   741  6349 I ACodec  : [OMX.qcom.video.encoder.avc] cannot encode HDR static metadata. Ignoring.
	Line 6030: 07-03 11:43:45.245   741  6349 I ACodec  : setupVideoEncoder succeeded
	Line 6031: 07-03 11:43:45.245   741  6349 I ExtendedACodec: [OMX.qcom.video.encoder.avc] configure, AMessage : AMessage(what = 'conf', target = 75) = {
	Line 6044: 07-03 11:43:45.245   741  6349 I ExtendedACodec:   int32_t encoder = 1
	Line 6174: 07-03 11:43:45.455   732  2169 I OMXMaster: makeComponentInstance(OMX.qcom.audio.encoder.aac) in mediacodec process
	Line 6180: 07-03 11:43:45.460   732  2169 E QC_AACENC:  component init: role = OMX.qcom.audio.encoder.aac
	Line 6182: 07-03 11:43:45.491   732   732 E OMXNodeInstance: setConfig(2dc0041:qcom.encoder.aac, ConfigPriority(0x6f800002)) ERROR: UnsupportedIndex(0x8000101a)
	Line 6184: 07-03 11:43:45.502   732  2169 E OMXNodeInstance: setConfig(2dc0041:qcom.encoder.aac, ConfigPriority(0x6f800002)) ERROR: UnsupportedIndex(0x8000101a)
	Line 6193: 07-03 11:43:45.537   741  6363 I CameraSource: Using encoder format: 0x22
	Line 6194: 07-03 11:43:45.537   741  6363 I CameraSource: Using encoder data space: 0x104
	Line 7431: 07-03 11:43:56.472   741  6347 I MediaCodecSource: encoder (video) stopping
	Line 7456: 07-03 11:43:56.611   741  6347 I MediaCodecSource: encoder (video) stopped
	Line 7494: 07-03 11:43:56.677   741  6347 I MediaCodecSource: encoder (audio) stopping
	Line 7534: 07-03 11:43:56.800   741  6347 I MediaCodecSource: encoder (audio) stopped
```

#### （五）、音视频混合MediaMuxer源码分析

``` cpp
sp<MediaWriter> mWriter;
[->\android\frameworks\av\media\libstagefright\MediaMuxer.cpp]
status_t MediaMuxer::start() {
    Mutex::Autolock autoLock(mMuxerLock);
    if (mState == INITIALIZED) {
        mState = STARTED;
        mFileMeta->setInt32(kKeyRealTimeRecording, false);
        return mWriter->start(mFileMeta.get());
    } else {
        ALOGE("start() is called in invalid state %d", mState);
        return INVALID_OPERATION;
    }
}
```
可以看到具体还是调用的MediaWriter实现的。我们来分析具体实例：MPEG4Writer.cpp，即MP4文件的格式封装过程。

1)       录制开始时，写入文件头部。

2)       录制进行时，实时写入音视频轨迹的数据块。

3)       录制结束时，写入索引信息并更新头部参数。

  索引负责描述音视频轨迹的特征，会随着音视频轨迹的存储而变化，所以通常做法会将录像文件索引信息放在音视频轨迹流后面，在媒体流数据写完（录像结束）后才能写入。可以看到，存放音视频数据的mdat box是位于第二位的，而负责检索音视频的moov box是位于最后的，这与通常的MP4封装的排列顺序不同，当然这是为了符合录制而产生的结果。因为 moov的大小是随着 mdat 变化的，而我们录制视频的时间预先是不知道的，所以需要先将mdat 数据写入，最后再写入moov，完成封装。 

  现有Android系统上录像都是录制是MP4或3GP格式，底层就是使用MPEG4Writer组合器类来完成的，它将编码后的音视频轨迹按照MPEG4规范进行封装，填入各个参数，就组合成完整的MP4格式文件。MPEG4Writer的组合功能主要由两种线程完成，一种是负责音视频数据写入封装文件的写线程（WriterThread），一种是音视频数据读取处理的轨迹线程（TrackThread）。轨迹线程一般有两个：视频轨迹数据读取线程和音频轨迹数据读取线程，而写线程只有一个，负责将轨迹线程中打包成Chunk的数据写入封装文件。

  如下图所示，轨迹线程是以帧为单位获取数据帧（Sample），并将每帧中的信息及系统环境信息提取汇总存储在内存的trak表中，其中需要维持的信息有Chunk写入文件的偏移地址Stco（Chunk Offset）、Sample与Chunk的映射关系Stsc（Sample-to-Chunk）、关键帧Stss（Sync Sample）、每一帧的持续时间Stts（Time-to-Sample）等，这些信息是跟每一帧的信息密切相关的，由图可以看出trak表由各自的线程维护，当录像结束时trak表会就会写入封装文件。而每一帧的数据流会先存入一个链表缓存中，当帧的数量达到一定值时，轨迹线程会将这些帧数据打包成块（Chunk）并通知写线程写入到封装文件。写线程接到Chunk已准备好的通知后就马上搜索Chunk链表（链表个数与轨迹线程个数相关，一般有两个，音视频轨迹线程各有一个），将找到的第一个Chunk后便写入封装文件，并会将写入的偏移地址更新到相应的trak表的Stco项（但trak表中其它数据是由轨迹线程更新）。音视频的Chunk数据是存储于同一mdat box中，按添加到Chunk链表时间先后顺序排列。等到录像结束时，录像应用会调用MPEG4Writer的stop方法，此时就会将音视频的trak表分别写入moov。

  

  ![Alt text](https://raw.githubusercontent.com/zjjzhoujinjian/zhoujinjian.com.images/master/video.system/VS-03-03-MPEG4Writer.png)


 其实看完上面的内容，应该对Android录制视频过程中，录制的视频的封装过程有一个大体了解，我们平时所说的视频后缀名.mp4/.mkv等等就是视频封装的各种格式。
先看看构造函数：在这里将实现一些参数的初始化，fd是传进来的录制文件的文件描述符。

``` cpp
[->\android\frameworks\av\media\libstagefright\MPEG4Writer.cpp]
MPEG4Writer::MPEG4Writer(int fd)
    : mFd(dup(fd)),
      mInitCheck(mFd < 0? NO_INIT: OK),
      mIsRealTimeRecording(true),
      mUse4ByteNalLength(true),
      mUse32BitOffset(true),
      mIsFileSizeLimitExplicitlyRequested(false),
      mPaused(false),
      mStarted(false),
      mWriterThreadStarted(false),
      mOffset(0),
      mMdatOffset(0),
      mMoovBoxBuffer(NULL),
      mMoovBoxBufferOffset(0),
      mWriteMoovBoxToMemory(false),
      mFreeBoxOffset(0),
      mStreamableFile(false),
      mEstimatedMoovBoxSize(0),
      mMoovExtraSize(0),
      mInterleaveDurationUs(1000000),
      mTimeScale(-1),
      mStartTimestampUs(-1ll),
      mLatitudex10000(0),
      mLongitudex10000(0),
      mAreGeoTagsAvailable(false),
      mStartTimeOffsetMs(-1),
      mMetaKeys(new AMessage()),
      mIsAudioAMR(false) {
    addDeviceMeta();

    // Verify mFd is seekable
    off64_t off = lseek64(mFd, 0, SEEK_SET);
    if (off < 0) {
        ALOGE("cannot seek mFd: %s (%d)", strerror(errno), errno);
        release();
    }
}

```

接着从 MPEG4Writer.cpp 的start()函数开始：
在start部分，我们看到在这一部分，writeFtypBox(param) 将实现录制文件文件头部信息的相关信息的写入操作；startWriterThread() 开启封装视频文件的写线程；startTracks(param) 开启视频数据的读线程，也就是前面文件部分所说的轨迹线程。

``` cpp
status_t MPEG4Writer::start(MetaData *param) {
    ......

    if (mStarted) {
        if (mPaused) {
            mPaused = false;
            return startTracks(param);
        }
        return OK;
    }
    ......
    mWriteMoovBoxToMemory = false;
    mMoovBoxBuffer = NULL;
    mMoovBoxBufferOffset = 0;

    writeFtypBox(param);

    mFreeBoxOffset = mOffset;

    if (mEstimatedMoovBoxSize == 0) {
        int32_t bitRate = -1;
        if (param) {
            param->findInt32(kKeyBitRate, &bitRate);
        }
        mEstimatedMoovBoxSize = estimateMoovBoxSize(bitRate);
    }
    CHECK_GE(mEstimatedMoovBoxSize, 8);
    if (mStreamableFile) {
        // Reserve a 'free' box only for streamable file
        lseek64(mFd, mFreeBoxOffset, SEEK_SET);
        writeInt32(mEstimatedMoovBoxSize);
        write("free", 4);
        mMdatOffset = mFreeBoxOffset + mEstimatedMoovBoxSize;
    } else {
        mMdatOffset = mOffset;
    }

    mOffset = mMdatOffset;
    lseek64(mFd, mMdatOffset, SEEK_SET);
   ......

    status_t err = startWriterThread();
   ......

    err = startTracks(param);
   ......
    mStarted = true;
    return OK;
}

```
  继续看下 startWriterThread（）部分，在startWriterThread（）函数中，将真正建立新的子线程，并在子线程中执行ThreadWrappe函数中的操作。

``` cpp
status_t MPEG4Writer::startWriterThread() {
    ALOGV("startWriterThread");

    mDone = false;
    mIsFirstChunk = true;
    mDriftTimeUs = 0;
    for (List<Track *>::iterator it = mTracks.begin();
         it != mTracks.end(); ++it) {
        ChunkInfo info;
        info.mTrack = *it;
        info.mPrevChunkTimestampUs = 0;
        info.mMaxInterChunkDurUs = 0;
        mChunkInfos.push_back(info);
    }

    pthread_attr_t attr;
    pthread_attr_init(&attr);
    pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_JOINABLE);
    pthread_create(&mThread, &attr, ThreadWrapper, this);
    pthread_attr_destroy(&attr);
    mWriterThreadStarted = true;
    return OK;
}

```
接着继续看 ThreadWrapper（）函数,在这里new 了一个MPEGWriter对象，真正的操作在threadFunc()中体现

``` cpp
void *MPEG4Writer::ThreadWrapper(void *me) {
    ALOGV("ThreadWrapper: %p", me);
    MPEG4Writer *writer = static_cast<MPEG4Writer *>(me);
    writer->threadFunc();
    return NULL;
}
```
下面看下threadFun()。在这个函数中，将根据变量mDone 进行while循环，一直检测是否有数据块Chunk可写。轨迹线程是一直将读数据的数据往buffer中写入，buffer到了一定量后，就是chunk,这时就会通过信号量 mChunkReadyCondition来通知封装文件的写线程去检测链表，然后将检索到的Chunk数据写入文件的数据区，当然写之前，肯定会去判断下是否真的有数据可写。

``` cpp
void MPEG4Writer::threadFunc() {
    ALOGV("threadFunc");

    prctl(PR_SET_NAME, (unsigned long)"MPEG4Writer", 0, 0, 0);

    Mutex::Autolock autoLock(mLock);
    while (!mDone) {
        Chunk chunk;
        bool chunkFound = false;

        while (!mDone && !(chunkFound = findChunkToWrite(&chunk))) {
            mChunkReadyCondition.wait(mLock);
        }
        if (chunkFound) {
            if (mIsRealTimeRecording) {
                mLock.unlock();
            }
            writeChunkToFile(&chunk);
            if (mIsRealTimeRecording) {
                mLock.lock();
            }
        }
    }

    writeAllChunks();
}

```
 下面看下writerChunkToFile(&chunk);轨迹线程读数据时是以数据帧Sample为单位，所以这里将Chunk写入封装文件，也是以Sample为单位，遍历整个链表，将数据写入封装文件，真正的写入操作是addSamole_l(*it);

``` cpp
void MPEG4Writer::writeAllChunks() {
    ALOGV("writeAllChunks");
    size_t outstandingChunks = 0;
    Chunk chunk;
    while (findChunkToWrite(&chunk)) {
        writeChunkToFile(&chunk);
        ++outstandingChunks;
    }

    sendSessionSummary();

    mChunkInfos.clear();
    ALOGD("%zu chunks are written in the last batch", outstandingChunks);
}

void MPEG4Writer::writeChunkToFile(Chunk* chunk) {
    ALOGV("writeChunkToFile: %" PRId64 " from %s track",
        chunk->mTimeStampUs, chunk->mTrack->isAudio()? "audio": "video");

    int32_t isFirstSample = true;
    while (!chunk->mSamples.empty()) {
        List<MediaBuffer *>::iterator it = chunk->mSamples.begin();

        off64_t offset = (chunk->mTrack->isAvc() || chunk->mTrack->isHevc())
                                ? addMultipleLengthPrefixedSamples_l(*it)
                                : addSample_l(*it);

        if (isFirstSample) {
            chunk->mTrack->addChunkOffset(offset);
            isFirstSample = false;
        }

        (*it)->release();
        (*it) = NULL;
        chunk->mSamples.erase(it);
    }
    chunk->mSamples.clear();
}

```
  下面看下addSamole_l(*it) 函数，wirte写入操作，mFd 是上层设置录制的文件路径传下来的文件描述符

``` cpp
off64_t MPEG4Writer::addSample_l(MediaBuffer *buffer) {
    off64_t old_offset = mOffset;

    ::write(mFd,
          (const uint8_t *)buffer->data() + buffer->range_offset(),
          buffer->range_length());

    mOffset += buffer->range_length();

    return old_offset;
}
```
  到此，封装文件的写入线程的操作大体走完，下面看轨迹线程的操作。

---------------------------------------------------------------------------------------------------------------------------------

   startTracks(param) 轨迹线程的开启。文件的录制过程中是有2条轨迹线程，一个是视频的轨迹线程，另一条则是音频的轨迹线程，在starTrack（param）中是在for 循环中start了两条轨迹线程。

``` cpp
status_t MPEG4Writer::startTracks(MetaData *params) {
    if (mTracks.empty()) {
        ALOGE("No source added");
        return INVALID_OPERATION;
    }

    for (List<Track *>::iterator it = mTracks.begin();
         it != mTracks.end(); ++it) {
        status_t err = (*it)->start(params);

        if (err != OK) {
            for (List<Track *>::iterator it2 = mTracks.begin();
                 it2 != it; ++it2) {
                (*it2)->stop();
            }

            return err;
        }
    }
    return OK;
}

```
（*it)->start(params) 将会执行status_t MPEG4Writer::Track::start(MetaData *params) {} 。在这边也是同样新建子线程，在子线程中执行轨迹线程的相应操作。

```cpp
status_t MPEG4Writer::Track::start(MetaData *params) {

    int64_t startTimeUs;
    ......
    mStartTimeRealUs = startTimeUs;

    int32_t rotationDegrees;
    ......
    initTrackingProgressStatus(params);

    sp<MetaData> meta = new MetaData;
    if (mOwner->isRealTimeRecording() && mOwner->numTracks() > 1) {
        int64_t startTimeOffsetUs = mOwner->getStartTimeOffsetMs() * 1000LL;
        if (startTimeOffsetUs < 0) {  // Start time offset was not set
            startTimeOffsetUs = kInitialDelayTimeUs;
        }
        startTimeUs += startTimeOffsetUs;
        ALOGI("Start time offset: %" PRId64 " us", startTimeOffsetUs);
    }

    meta->setInt64(kKeyTime, startTimeUs);

    status_t err = mSource->start(meta.get());
    ......

    pthread_attr_t attr;
    pthread_attr_init(&attr);
    pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_JOINABLE);

    mDone = false;
    mStarted = true;
    mTrackDurationUs = 0;
    mReachedEOS = false;
    mEstimatedTrackSizeBytes = 0;
    mMdatSizeBytes = 0;
    mMaxChunkDurationUs = 0;
    mLastDecodingTimeUs = -1;

    pthread_create(&mThread, &attr, ThreadWrapper, this);
    pthread_attr_destroy(&attr);

    return OK;
}
```
下面看下上面ThreadWrapper函数,真正的操作又是放到了threadEntry()中去执行

``` cpp
void *MPEG4Writer::Track::ThreadWrapper(void *me) {
 
    Track *track = static_cast<Track *>(me);
 
    status_t err = track->threadEntry();
    return (void *) err;
}
status_t MPEG4Writer::Track::threadEntry() {
    int32_t count = 0;
    const int64_t interleaveDurationUs = mOwner->interleaveDuration();
    const bool hasMultipleTracks = (mOwner->numTracks() > 1);
    int64_t chunkTimestampUs = 0;
    int32_t nChunks = 0;
    int32_t nActualFrames = 0;        // frames containing non-CSD data (non-0 length)
    int32_t nZeroLengthFrames = 0;
    int64_t lastTimestampUs = 0;      // Previous sample time stamp
    int64_t lastDurationUs = 0;       // Between the previous two samples
    int64_t currDurationTicks = 0;    // Timescale based ticks
    int64_t lastDurationTicks = 0;    // Timescale based ticks
    int32_t sampleCount = 1;          // Sample count in the current stts table entry
    uint32_t previousSampleSize = 0;  // Size of the previous sample
    int64_t previousPausedDurationUs = 0;
    int64_t timestampUs = 0;
    int64_t cttsOffsetTimeUs = 0;
    int64_t currCttsOffsetTimeTicks = 0;   // Timescale based ticks
    int64_t lastCttsOffsetTimeTicks = -1;  // Timescale based ticks
    int32_t cttsSampleCount = 0;           // Sample count in the current ctts table entry
    uint32_t lastSamplesPerChunk = 0;

    if (mIsAudio) {
        prctl(PR_SET_NAME, (unsigned long)"AudioTrackEncoding", 0, 0, 0);
    } else {
        prctl(PR_SET_NAME, (unsigned long)"VideoTrackEncoding", 0, 0, 0);
    }

    if (mOwner->isRealTimeRecording()) {
        androidSetThreadPriority(0, ANDROID_PRIORITY_AUDIO);
    }

    sp<MetaData> meta_data;

    status_t err = OK;
    MediaBuffer *buffer;
    const char *trackName = mIsAudio ? "Audio" : "Video";
    while (!mDone && (err = mSource->read(&buffer)) == OK) {
        if (buffer->range_length() == 0) {
            buffer->release();
            buffer = NULL;
            ++nZeroLengthFrames;
            continue;
        }

        // If the codec specific data has not been received yet, delay pause.
        // After the codec specific data is received, discard what we received
        // when the track is to be paused.
        if (mPaused && !mResumed) {
            buffer->release();
            buffer = NULL;
            continue;
        }

        ++count;

        int32_t isCodecConfig;
        if (buffer->meta_data()->findInt32(kKeyIsCodecConfig, &isCodecConfig)
                && isCodecConfig) {
            // if config format (at track addition) already had CSD, keep that
            // UNLESS we have not received any frames yet.
            // TODO: for now the entire CSD has to come in one frame for encoders, even though
            // they need to be spread out for decoders.
            if (mGotAllCodecSpecificData && nActualFrames > 0) {
                ALOGI("ignoring additional CSD for video track after first frame");
            } else {
                mMeta = mSource->getFormat(); // get output format after format change

                if (mIsAvc) {
                    status_t err = makeAVCCodecSpecificData(
                            (const uint8_t *)buffer->data()
                                + buffer->range_offset(),
                            buffer->range_length());
                    CHECK_EQ((status_t)OK, err);
                } else if (mIsHevc) {
                    status_t err = makeHEVCCodecSpecificData(
                            (const uint8_t *)buffer->data()
                                + buffer->range_offset(),
                            buffer->range_length());
                    CHECK_EQ((status_t)OK, err);
                } else if (mIsMPEG4) {
                    copyCodecSpecificData((const uint8_t *)buffer->data() + buffer->range_offset(),
                            buffer->range_length());
                }
            }

            if (!mIsAudio) {
                int32_t fps;
                mMeta->findInt32(kKeyFrameRate, &fps);
                int64_t cttsOffsetTimeUs = 1000000LL/fps;
                mCttsOffsetTimeUs = cttsOffsetTimeUs + kMinCttsOffsetTimeUs; //delta factor
            }
            buffer->release();
            buffer = NULL;

            mGotAllCodecSpecificData = true;
            continue;
        }

        ++nActualFrames;

        MediaBuffer *copy = NULL;
        // Check if the upstream source hints it is OK to hold on to the
        // buffer without releasing immediately and avoid cloning the buffer
        if (AVUtils::get()->canDeferRelease(buffer->meta_data())) {
            copy = buffer;
            meta_data = new MetaData(*buffer->meta_data().get());
        } else {
            // Make a deep copy of the MediaBuffer and Metadata and release
            // the original as soon as we can
            copy = new MediaBuffer(buffer->range_length());
            memcpy(copy->data(), (uint8_t *)buffer->data() + buffer->range_offset(),
                    buffer->range_length());
            copy->set_range(0, buffer->range_length());
            meta_data = new MetaData(*buffer->meta_data().get());
            buffer->release();
            buffer = NULL;
        }

        if (mIsAvc || mIsHevc) StripStartcode(copy);

        size_t sampleSize = copy->range_length();
        if (mIsAvc || mIsHevc) {
            if (mOwner->useNalLengthFour()) {
                sampleSize += 4;
            } else {
                sampleSize += 2;
            }
        }

        // Max file size or duration handling
        mMdatSizeBytes += sampleSize;
        updateTrackSizeEstimate();

        if (mOwner->exceedsFileSizeLimit()) {
            ALOGW("Recorded file size exceeds limit %" PRId64 "bytes",
                    mOwner->mMaxFileSizeLimitBytes);
            mOwner->notify(MEDIA_RECORDER_EVENT_INFO, MEDIA_RECORDER_INFO_MAX_FILESIZE_REACHED, 0);
            copy->release();
            mSource->stop();
            break;
        }
        if (mOwner->exceedsFileDurationLimit()) {
            ALOGW("Recorded file duration exceeds limit %" PRId64 "microseconds",
                    mOwner->mMaxFileDurationLimitUs);
            mOwner->notify(MEDIA_RECORDER_EVENT_INFO, MEDIA_RECORDER_INFO_MAX_DURATION_REACHED, 0);
            copy->release();
            mSource->stop();
            break;
        }


        int32_t isSync = false;
        meta_data->findInt32(kKeyIsSyncFrame, &isSync);
        CHECK(meta_data->findInt64(kKeyTime, &timestampUs));

////////////////////////////////////////////////////////////////////////////////
        if (mStszTableEntries->count() == 0) {
            mFirstSampleTimeRealUs = systemTime() / 1000;
            mStartTimestampUs = timestampUs;
            mOwner->setStartTimestampUs(mStartTimestampUs);
            previousPausedDurationUs = mStartTimestampUs;
        }

        if (mResumed) {
            int64_t durExcludingEarlierPausesUs = timestampUs - previousPausedDurationUs;
            if (WARN_UNLESS(durExcludingEarlierPausesUs >= 0ll, "for %s track", trackName)) {
                copy->release();
                mSource->stop();
                mIsMalformed = true;
                break;
            }

            int64_t pausedDurationUs = durExcludingEarlierPausesUs - mTrackDurationUs;
            if (WARN_UNLESS(pausedDurationUs >= lastDurationUs, "for %s track", trackName)) {
                copy->release();
                mSource->stop();
                mIsMalformed = true;
                break;
            }

            previousPausedDurationUs += pausedDurationUs - lastDurationUs;
            mResumed = false;
        }

        timestampUs -= previousPausedDurationUs;
        if (WARN_UNLESS(timestampUs >= 0ll, "for %s track", trackName)) {
            copy->release();
            mSource->stop();
            mIsMalformed = true;
            break;
        }

        if (!mIsAudio) {
            /*
             * Composition time: timestampUs
             * Decoding time: decodingTimeUs
             * Composition time offset = composition time - decoding time
             */
            int64_t decodingTimeUs;
            CHECK(meta_data->findInt64(kKeyDecodingTime, &decodingTimeUs));
            decodingTimeUs -= previousPausedDurationUs;

            // ensure non-negative, monotonic decoding time
            if (mLastDecodingTimeUs < 0) {
                decodingTimeUs = std::max((int64_t)0, decodingTimeUs);
            } else {
                // increase decoding time by at least 1 tick
                decodingTimeUs = std::max(
                        mLastDecodingTimeUs + divUp(1000000, mTimeScale), decodingTimeUs);
            }

            mLastDecodingTimeUs = decodingTimeUs;
            cttsOffsetTimeUs =
                    timestampUs + mCttsOffsetTimeUs - decodingTimeUs;
            if (cttsOffsetTimeUs < 0) {
                cttsOffsetTimeUs = 0;
            }
            if (WARN_UNLESS(cttsOffsetTimeUs >= 0ll, "for %s track", trackName)) {
                copy->release();
                mSource->stop();
                mIsMalformed = true;
                break;
            }

            timestampUs = decodingTimeUs;
            ALOGV("decoding time: %" PRId64 " and ctts offset time: %" PRId64,
                timestampUs, cttsOffsetTimeUs);

            // Update ctts box table if necessary
            currCttsOffsetTimeTicks =
                    (cttsOffsetTimeUs * mTimeScale + 500000LL) / 1000000LL;
            if (WARN_UNLESS(currCttsOffsetTimeTicks <= 0x0FFFFFFFFLL, "for %s track", trackName)) {
                copy->release();
                mSource->stop();
                mIsMalformed = true;
                break;
            }

            if (mStszTableEntries->count() == 0) {
                lastCttsOffsetTimeTicks = currCttsOffsetTimeTicks;
                //addOneCttsTableEntry(1, currCttsOffsetTimeTicks);
                //cttsSampleCount = 0;      // No sample in ctts box is pending
                cttsSampleCount = 1;
            } else {
                if (currCttsOffsetTimeTicks != lastCttsOffsetTimeTicks) {
                    addOneCttsTableEntry(cttsSampleCount, lastCttsOffsetTimeTicks);
                    lastCttsOffsetTimeTicks = currCttsOffsetTimeTicks;
                    cttsSampleCount = 1;  // One sample in ctts box is pending
                } else {
                    ++cttsSampleCount;
                }
            }

            // Update ctts time offset range
            if (mStszTableEntries->count() == 0) {
                mMinCttsOffsetTimeUs = currCttsOffsetTimeTicks;
                mMaxCttsOffsetTimeUs = currCttsOffsetTimeTicks;
            } else {
                if (currCttsOffsetTimeTicks > mMaxCttsOffsetTimeUs) {
                    mMaxCttsOffsetTimeUs = currCttsOffsetTimeTicks;
                } else if (currCttsOffsetTimeTicks < mMinCttsOffsetTimeUs) {
                    mMinCttsOffsetTimeUs = currCttsOffsetTimeTicks;
                }
            }

        }

        if (mOwner->isRealTimeRecording()) {
            if (mIsAudio) {
                updateDriftTime(meta_data);
            }
        }

        if (WARN_UNLESS(timestampUs >= 0ll, "for %s track", trackName)) {
            copy->release();
            mSource->stop();
            mIsMalformed = true;
            break;
        }

        ALOGV("%s media time stamp: %" PRId64 " and previous paused duration %" PRId64,
                trackName, timestampUs, previousPausedDurationUs);
        if (timestampUs > mTrackDurationUs) {
            mTrackDurationUs = timestampUs;
        }

        // We need to use the time scale based ticks, rather than the
        // timestamp itself to determine whether we have to use a new
        // stts entry, since we may have rounding errors.
        // The calculation is intended to reduce the accumulated
        // rounding errors.
        currDurationTicks =
            ((timestampUs * mTimeScale + 500000LL) / 1000000LL -
                (lastTimestampUs * mTimeScale + 500000LL) / 1000000LL);
        if (currDurationTicks < 0ll) {
            ALOGE("do not support out of order frames (timestamp: %lld < last: %lld for %s track",
                    (long long)timestampUs, (long long)lastTimestampUs, trackName);
            copy->release();
            mSource->stop();
            mIsMalformed = true;
            break;
        }

        // if the duration is different for this sample, see if it is close enough to the previous
        // duration that we can fudge it and use the same value, to avoid filling the stts table
        // with lots of near-identical entries.
        // "close enough" here means that the current duration needs to be adjusted by less
        // than 0.1 milliseconds
        if (lastDurationTicks && (currDurationTicks != lastDurationTicks)) {
            int64_t deltaUs = ((lastDurationTicks - currDurationTicks) * 1000000LL
                    + (mTimeScale / 2)) / mTimeScale;
            if (deltaUs > -100 && deltaUs < 100) {
                // use previous ticks, and adjust timestamp as if it was actually that number
                // of ticks
                currDurationTicks = lastDurationTicks;
                timestampUs += deltaUs;
            }
        }

        mStszTableEntries->add(htonl(sampleSize));
        if (mStszTableEntries->count() > 2) {

            // Force the first sample to have its own stts entry so that
            // we can adjust its value later to maintain the A/V sync.
            if (mStszTableEntries->count() == 3 || currDurationTicks != lastDurationTicks) {
                addOneSttsTableEntry(sampleCount, lastDurationTicks);
                sampleCount = 1;
            } else {
                ++sampleCount;
            }

        }
        if (mSamplesHaveSameSize) {
            if (mStszTableEntries->count() >= 2 && previousSampleSize != sampleSize) {
                mSamplesHaveSameSize = false;
            }
            previousSampleSize = sampleSize;
        }
        ALOGV("%s timestampUs/lastTimestampUs: %" PRId64 "/%" PRId64,
                trackName, timestampUs, lastTimestampUs);
        lastDurationUs = timestampUs - lastTimestampUs;
        lastDurationTicks = currDurationTicks;
        lastTimestampUs = timestampUs;

        if (isSync != 0) {
            addOneStssTableEntry(mStszTableEntries->count());
        }

        if (mTrackingProgressStatus) {
            if (mPreviousTrackTimeUs <= 0) {
                mPreviousTrackTimeUs = mStartTimestampUs;
            }
            trackProgressStatus(timestampUs);
        }
        if (!hasMultipleTracks) {
            off64_t offset = (mIsAvc || mIsHevc) ? mOwner->addMultipleLengthPrefixedSamples_l(copy)
                                 : mOwner->addSample_l(copy);

            uint32_t count = (mOwner->use32BitFileOffset()
                        ? mStcoTableEntries->count()
                        : mCo64TableEntries->count());

            if (count == 0) {
                addChunkOffset(offset);
            }
            copy->release();
            copy = NULL;
            continue;
        }

        mChunkSamples.push_back(copy);
        if (interleaveDurationUs == 0) {
            addOneStscTableEntry(++nChunks, 1);
            bufferChunk(timestampUs);
        } else {
            if (chunkTimestampUs == 0) {
                chunkTimestampUs = timestampUs;
            } else {
                int64_t chunkDurationUs = timestampUs - chunkTimestampUs;
                if (chunkDurationUs > interleaveDurationUs) {
                    if (chunkDurationUs > mMaxChunkDurationUs) {
                        mMaxChunkDurationUs = chunkDurationUs;
                    }
                    ++nChunks;
                    if (nChunks == 1 ||  // First chunk
                        lastSamplesPerChunk != mChunkSamples.size()) {
                        lastSamplesPerChunk = mChunkSamples.size();
                        addOneStscTableEntry(nChunks, lastSamplesPerChunk);
                    }
                    bufferChunk(timestampUs);
                    chunkTimestampUs = timestampUs;
                }
            }
        }

    }

    if (isTrackMalFormed()) {
        err = ERROR_MALFORMED;
    }

    mOwner->trackProgressStatus(mTrackId, -1, err);

    // Last chunk
    if (!hasMultipleTracks) {
        addOneStscTableEntry(1, mStszTableEntries->count());
    } else if (!mChunkSamples.empty()) {
        addOneStscTableEntry(++nChunks, mChunkSamples.size());
        bufferChunk(timestampUs);
    }

    // We don't really know how long the last frame lasts, since
    // there is no frame time after it, just repeat the previous
    // frame's duration.
    if (mStszTableEntries->count() == 1) {
        lastDurationUs = 0;  // A single sample's duration
        lastDurationTicks = 0;
    } else {
        ++sampleCount;  // Count for the last sample
    }

    if (mStszTableEntries->count() <= 2) {
        addOneSttsTableEntry(1, lastDurationTicks);
        if (sampleCount - 1 > 0) {
            addOneSttsTableEntry(sampleCount - 1, lastDurationTicks);
        }
    } else {
        addOneSttsTableEntry(sampleCount, lastDurationTicks);
    }

    // The last ctts box may not have been written yet, and this
    // is to make sure that we write out the last ctts box.
    if (currCttsOffsetTimeTicks == lastCttsOffsetTimeTicks) {
        if (cttsSampleCount > 0) {
            addOneCttsTableEntry(cttsSampleCount, lastCttsOffsetTimeTicks);
        }
    }

    mTrackDurationUs += lastDurationUs;
    mReachedEOS = true;

    sendTrackSummary(hasMultipleTracks);

    ALOGI("Received total/0-length (%d/%d) buffers and encoded %d frames. - %s",
            count, nZeroLengthFrames, mStszTableEntries->count(), trackName);
    if (mIsAudio) {
        ALOGI("Audio track drift time: %" PRId64 " us", mOwner->getDriftTimeUs());
    }
    // if err is ERROR_IO (ex: during SSR), return OK to save the
    // recorded file successfully. Session tear down will happen as part of
    // client callback
    if ((err == ERROR_IO) || (err == ERROR_END_OF_STREAM)) {
        return OK;
    }
    return err;
}
```
------------------------------------------------------------------------------------------------------------

 下面看下，录制文件结束时的一些操作。录制文件结束时，上层应用分别是调用 MediaRecorder的stop()、reset()和release()法，下面看下MPEG4Writer.cpp中相对应的操作。

``` cpp
status_t MPEG4Writer::Track::stop() {
    ALOGD("%s track stopping", mIsAudio? "Audio": "Video");
    if (!mStarted) {
        ALOGE("Stop() called but track is not started");
        return ERROR_END_OF_STREAM;
    }

    if (mDone) {
        return OK;
    }
    mDone = true;

    ALOGD("%s track source stopping", mIsAudio? "Audio": "Video");
    mSource->stop();
    ALOGD("%s track source stopped", mIsAudio? "Audio": "Video");

    void *dummy;
    pthread_join(mThread, &dummy);
    status_t err = static_cast<status_t>(reinterpret_cast<uintptr_t>(dummy));

    ALOGD("%s track stopped", mIsAudio? "Audio": "Video");
    return err;
}
void MPEG4Writer::release() {
    close(mFd);
    mFd = -1;
    mInitCheck = NO_INIT;
    mStarted = false;
    free(mMoovBoxBuffer);
    mMoovBoxBuffer = NULL;
}
```

#### （六）、参考资料(特别感谢各位前辈的分析和图示)：
[Camera#capture-video](https://developer.android.com/guide/topics/media/camera#capture-video)
[Android NuPlayer播放框架 ](http://www.cnblogs.com/tocy/tag/%E6%92%AD%E6%94%BE%E6%A1%86%E6%9E%B6/)
[android ACodec MediaCodec NuPlayer flow - CSDN博客](https://blog.csdn.net/dfhuang09/article/details/54926526)
[android MediaCodec ACodec - CSDN博客](https://blog.csdn.net/dfhuang09/article/details/60132620)
[ffmpeg开发之旅(1)-(7)（总共七篇）](https://blog.csdn.net/column/details/16222.html)
[深入理解Android音视频同步机制（总共五篇）](https://blog.csdn.net/nonmarking/article/category/6746500)
[Android硬编码——音频编码、视频编码及音视频混合](https://blog.csdn.net/junzia/article/details/54018671)
[Android 高通平台Camera录制--MPEG4Writer.cpp 简单跟读](https://blog.csdn.net/mr_zjc/article/details/46822833)



