---
title: Audio System（3）：Android audio system(音频系统)分析
cover: https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/hexo.themes/bing-wallpaper-2018.04.15.jpg
categories:
  - Audio
tags:
  - Android
  - Linux
  - Audio
toc: true
abbrlink: 20181108
date: 2018-11-08 09:25:00
---

--------------------------------------------------------------------------------
注：文章都是通过阅读各位前辈总结的资料、Android 7.1.2 && Linux（kernel 3.18）Qualcomm平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除，禁止转载（©Qualcomm Technologies, Inc. 版权所有），谢谢。

[【特别感谢 - 林学森的Android专栏】](https://blog.csdn.net/xuesen_lin/article/category/1390680)
[【特别感谢 - Yangwen123 - 深入剖析Android音频系统】](https://blog.csdn.net/yangwen123/article/category/2589087)
[【特别感谢 - Zyuanyun - Android 音频系统：从 AudioTrack 到 AudioFlinger】](https://blog.csdn.net/yangwen123/article/category/2589087)
Google Pixel、Pixel XL 内核代码（Kernel-3.18）：
 [Kernel source for Pixel and Pixel XL - GitHub](https://github.com/matthewdalex/marlin)

AOSP 源码（文章基于 Android 7.1.2）：
 [ Android 系统全套源代码分享 (更新到 8.1.0_r1)](https://testerhome.com/topics/2229)

--------------------------------------------------------------------------------

源码（主要源码路径）：

> User space audio code 源码：

• **/hardware/qcom/audio/hal/ – (Audio 高通HAL 源码)** 

• **/libhardware/modules/audio/ – (Audio 原生HAL 源码)** 

• **/external/tinyalsa/ –  (tinymix, tinyplay, tinycap 源码)**

• **/vendor/qcom/proprietary/mm-audio/ – (QTI OMX  audio encoder and decoders 源码，未公开)**

• **/frameworks/av/media/audioserver/ – (Audioserver 源码)**

• **/hardware/libhardware_legacy/audio – (Audio legacy 源码)**

• **/frameworks/av/media/libstagefright/ – (Google Stagefright 多媒体框架源码)**

• **/frameworks/av/services/audioflinger/ – (Audioflinger 相关源码)**

• **/external/bluetooth/bluedroid/ – (A2DP audio HAL 相关源码)**/

• **/hardware/libhardware/modules/usbaudio/ – (USB HAL 源码)**/

• **/vendor/qcom/proprietary/wfd/mm/source/framework/src/ – (Wi-Fi Display (WFD)、 WFDMMSourceAudioSource.cpp，未公开)**

• **/system/core/include/system/ – (audio.h)**/

#### (一)、深入剖析Android音频之AudioFlinger

##### 1.0、总体框架图

![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/audio.system/31-Audio-system-Android-Linux-arc.png)


系统启动时将执行 /system/etc/init/audioserver.rc ，运行 /system/bin/ 目录下的 audioserver 服务。audioserver.rc 内容如下：

``` cpp
[->\frameworks\av\media\audioserver\audioserver.rc]
service audioserver /system/bin/audioserver
    class main
    user audioserver
    # media gid needed for /dev/fm (radio ) and for  /data/misc/media (tee)
    group audio radio camera drmpc inet media mediarm net_bt net_bt_admin net_bw_acct
    ioprio rt 4
    writepid /dev/cpuset/forground/tasks /dev/stune/foreground/tasks
```
audioserver 是由同目录下main_audioserver编译生成的。
##### 1.1、AudioFlinger
AudioFlinger是整个音频系统的核心与难点。作为Android系统中的音频中枢，它同时也是一个系统服务，启到承上(为上层提供访问接口)启下(通过HAL来管理音频设备)的作用。只有理解了AudioFlinger，才能以此为基础更好地深入到其它模块，并且Audioserver最先启动的也是AudioFlinger，因而我们把它放在前面进行分析。

``` cpp
[->\frameworks\av\media\audioserver\main_audioserver.cpp]
int main(int argc __unused, char **argv)
{
    ......
    if (doLog && (childPid = fork()) != 0) {
    ......
        } else {
        // all other services
        if (doLog) {
            prctl(PR_SET_PDEATHSIG, SIGKILL);   // if parent media.log dies before me, kill me also
            setpgid(0, 0);                      // but if I die first, don't kill my parent
        }
        sp<ProcessState> proc(ProcessState::self());
        sp<IServiceManager> sm = defaultServiceManager();
        ALOGI("ServiceManager: %p", sm.get());
        AudioFlinger::instantiate();
        AudioPolicyService::instantiate();
        RadioService::instantiate();
        SoundTriggerHwService::instantiate();
        ProcessState::self()->startThreadPool();
        IPCThreadState::self()->joinThreadPool();
    }
}

```
AudioFlinger继承了模板类BinderService，该类用于注册native service。
BinderService是一个模板类，该类的publish函数就是完成向ServiceManager注册服务。
AudioFlinger注册名为"media.audio_flinger"的服务。
``` cpp
static const char* getServiceName() ANDROID_API { return "media.audio_flinger"; }
```

##### 1.1.1 AudioFlinger服务的启动和运行
AudioFlinger的构造函数，发现它只是简单地为内部一些变量做了初始化，除此之外就没有任何代码了：

``` cpp
[->\frameworks\av\services\audioflinger\AudioFlinger.cpp]
AudioFlinger::AudioFlinger()
    : BnAudioFlinger(),
      mPrimaryHardwareDev(NULL),
      mAudioHwDevs(NULL),
      mHardwareStatus(AUDIO_HW_IDLE),
      mMasterVolume(1.0f),
      mMasterMute(false),
      // mNextUniqueId(AUDIO_UNIQUE_ID_USE_MAX),
      mMode(AUDIO_MODE_INVALID),
      mBtNrecIsOff(false),
      mIsLowRamDevice(true),
      mIsDeviceTypeKnown(false),
      mGlobalEffectEnableTime(0),
      mSystemReady(false)
{
    // unsigned instead of audio_unique_id_use_t, because ++ operator is unavailable for enum
    for (unsigned use = AUDIO_UNIQUE_ID_USE_UNSPECIFIED; use < AUDIO_UNIQUE_ID_USE_MAX; use++) {
        // zero ID has a special meaning, so unavailable
        mNextUniqueIds[use] = AUDIO_UNIQUE_ID_USE_MAX;
    }
    ......
}
```
BnAudioFlinger是由RefBase层层继承而来的，并且IServiceManager::addService的第二个参数实际上是一个强指针引用(constsp<IBinder>&),因而AudioFlinger具备了强指针被第一次引用时调用onFirstRef的程序逻辑。

``` cpp
[->\frameworks\av\services\audioflinger\AudioFlinger.cpp]
void AudioFlinger::onFirstRef()
{
    Mutex::Autolock _l(mLock);

    /* TODO: move all this work into an Init() function */
    char val_str[PROPERTY_VALUE_MAX] = { 0 };
    if (property_get("ro.audio.flinger_standbytime_ms", val_str, NULL) >= 0) {
        uint32_t int_val;
        if (1 == sscanf(val_str, "%u", &int_val)) {
            mStandbyTimeInNsecs = milliseconds(int_val);
            ALOGI("Using %u mSec as standby time.", int_val);
        } else {
            mStandbyTimeInNsecs = kDefaultStandbyTimeInNsecs;
            ALOGI("Using default %u mSec as standby time.",
                    (uint32_t)(mStandbyTimeInNsecs / 1000000));
        }
    }

    mPatchPanel = new PatchPanel(this);

    mMode = AUDIO_MODE_NORMAL;
}
```
从这时开始，AudioFlinger就是一个“有意义”的实体了

##### 1.2、音频设备的管理
虽然AudioFlinger实体已经成功创建并初始化，但到目前为止它还是一块静态的内存空间，没有涉及到具体的工作。

从职能分布上来讲，AudioPolicyService是策略的制定者，比如什么时候打开音频接口设备、某种Stream类型的音频对应什么设备等等。而AudioFlinger则是策略的执行者，例如具体如何与音频设备通信，如何维护现有系统中的音频设备，以及多个音频流的混音如何处理等等都得由它来完成。

目前Audio系统中支持的音频设备接口(Audio Interface)分为三大类，即：

``` cpp
[->\frameworks\av\services\audioflinger\AudioFlinger.cpp]
static const char * const audio_interfaces[] = {
    AUDIO_HARDWARE_MODULE_ID_PRIMARY,
    AUDIO_HARDWARE_MODULE_ID_A2DP,
    AUDIO_HARDWARE_MODULE_ID_USB,
};
```
每种音频设备接口由一个对应的so库提供支持。那么AudioFlinger怎么会知道当前设备中支持上述的哪些接口，每种接口又支持哪些具体的音频设备呢？这是AudioPolicyService的责任之一，即根据用户配置来指导AudioFlinger加载设备接口。

当AudioPolicyManagerBase(AudioPolicyService中持有的Policy管理者，后面小节有详细介绍)构造时，它会读取厂商关于音频设备的描述文件(audio_policy.conf)，然后据此来打开以上三类音频接口(如果存在的话)。这一过程最终会调用loadHwModule@AudioFlinger，如下所示：
##### 1.2.1、加载设备loadHwModule()
``` cpp
[->\frameworks\av\services\audioflinger\AudioFlinger.cpp]
/*name就是前面audio_interfaces 数组成员中的字符串*/
audio_module_handle_t AudioFlinger::loadHwModule(const char *name)
{
    if (name == NULL) {
        return AUDIO_MODULE_HANDLE_NONE;
    }
    if (!settingsAllowed()) {
        return AUDIO_MODULE_HANDLE_NONE;
    }
    Mutex::Autolock _l(mLock);
    return loadHwModule_l(name);
}
```
这个函数没有做实质性的工作，只是执行了加锁动作，然后接着调用下面的函数：

``` cpp
[->\frameworks\av\services\audioflinger\AudioFlinger.cpp]
// loadHwModule_l() must be called with AudioFlinger::mLock held
audio_module_handle_t AudioFlinger::loadHwModule_l(const char *name)
{

    /* Step 1. 是否已经添加了这个interface ? */
    for (size_t i = 0; i < mAudioHwDevs.size(); i++) {
        if (strncmp(mAudioHwDevs.valueAt(i)->moduleName(), name, strlen(name)) == 0) {
            ALOGW("loadHwModule() module %s already loaded", name);
            return mAudioHwDevs.keyAt(i);
        }
    }

    audio_hw_device_t *dev;
    /* Step 2. 加载audio interface */
    int rc = load_audio_interface(name, &dev);
    ......
    
    /* Step 3. 初始化 */
    mHardwareStatus = AUDIO_HW_INIT;
    rc = dev->init_check(dev);
    mHardwareStatus = AUDIO_HW_IDLE;
    
    ......
    /* Step 4. 添加到全局变量中 */
    audio_module_handle_t handle = (audio_module_handle_t) nextUniqueId(AUDIO_UNIQUE_ID_USE_MODULE);
    mAudioHwDevs.add(handle, new AudioHwDevice(handle, name, dev, flags));

    return handle;

}
```
Step1@ loadHwModule_l. 首先查找mAudioHwDevs是否已经添加了变量name所指示的audio interface，如果是的话直接返回。第一次进入时mAudioHwDevs的size为0，所以还会继续往下执行。

Step2@ loadHwModule_l. 加载指定的audiointerface，比如“primary”、“a2dp”或者“usb”。函数load_audio_interface用来加载设备所需的库文件，然后打开设备并创建一个audio_hw_device_t实例。音频接口设备所对应的库文件名称是有一定格式的，比如a2dp的模块名可能是audio.a2dp.so或者audio.a2dp.default.so等等。查找路径主要有两个，即：

``` cpp
[->\hardware\libhardware\hardware.c]
#define HAL_LIBRARY_PATH1 "/system/lib64/hw"
#define HAL_LIBRARY_PATH2 "/vendor/lib64/hw"
#define HAL_LIBRARY_PATH3 "/odm/lib64/hw"
```
当然，因为Android是完全开源的，各开发商可以根据自己的需要来进行相应的修改，比如下面是Google pixel 设备的音频库：

``` cpp
adb shell && cd system/lib64/hw && ls -l
-rw-r--r-- 1 root root   30440 2009-01-01 00:00 audio.a2dp.default.so
-rw-r--r-- 1 root root   18156 2009-01-01 00:00 audio.primary.default.so
-rw-r--r-- 1 root root  275612 2009-01-01 00:00 audio.primary.msm8996.so
-rw-r--r-- 1 root root   34540 2009-01-01 00:00 audio.r_submix.default.so
-rw-r--r-- 1 root root   22248 2009-01-01 00:00 audio.usb.default.so
-rw-r--r-- 1 root root   96096 2009-01-01 00:00 audio_policy.default.so
-rw-r--r-- 1 root root 1637208 2009-01-01 00:00 bluetooth.default.so
......
```

Step3@ loadHwModule_l，进行初始化操作。其中init_check是为了确定这个audio interface是否已经成功初始化，0是成功，其它值表示失败。接下来如果这个device支持主音量，我们还需要通过set_master_volume进行设置。在每次操作device前，都要先改变mHardwareStatus的状态值，操作结束后将其复原为AUDIO_HW_IDLE(根据源码中的注释，这样做是为了方便dump时正确输出内部状态，这里我们就不去深究了)。

Step4@ loadHwModule_l. 把加载后的设备添加入mAudioHwDevs键值对中，其中key的值是由nextUniqueId生成的，这样做保证了这个audiointerface拥有全局唯一的id号。

完成了audiointerface的模块加载只是万里长征的第一步。因为每一个interface包含的设备通常不止一个，Android系统目前支持的音频设备如下列表所示：
 Android系统支持的音频设备列表(输出)

``` cpp
[->\hardware\libhardware_legacy\audio\audio_hw_hal.cpp]
static uint32_t audio_device_conv_table[][HAL_API_REV_NUM] =
{
    /* output devices */
    { AudioSystem::DEVICE_OUT_EARPIECE, AUDIO_DEVICE_OUT_EARPIECE },//
    { AudioSystem::DEVICE_OUT_SPEAKER, AUDIO_DEVICE_OUT_SPEAKER },//SPEAKER
    { AudioSystem::DEVICE_OUT_WIRED_HEADSET, AUDIO_DEVICE_OUT_WIRED_HEADSET },//HEADSET
    { AudioSystem::DEVICE_OUT_WIRED_HEADPHONE, AUDIO_DEVICE_OUT_WIRED_HEADPHONE },//HEADPHONE
    { AudioSystem::DEVICE_OUT_BLUETOOTH_SCO, AUDIO_DEVICE_OUT_BLUETOOTH_SCO },
    { AudioSystem::DEVICE_OUT_BLUETOOTH_SCO_HEADSET, AUDIO_DEVICE_OUT_BLUETOOTH_SCO_HEADSET },
    { AudioSystem::DEVICE_OUT_BLUETOOTH_SCO_CARKIT, AUDIO_DEVICE_OUT_BLUETOOTH_SCO_CARKIT },
    { AudioSystem::DEVICE_OUT_BLUETOOTH_A2DP, AUDIO_DEVICE_OUT_BLUETOOTH_A2DP },
    { AudioSystem::DEVICE_OUT_BLUETOOTH_A2DP_HEADPHONES, AUDIO_DEVICE_OUT_BLUETOOTH_A2DP_HEADPHONES },
    { AudioSystem::DEVICE_OUT_BLUETOOTH_A2DP_SPEAKER, AUDIO_DEVICE_OUT_BLUETOOTH_A2DP_SPEAKER },
    { AudioSystem::DEVICE_OUT_AUX_DIGITAL, AUDIO_DEVICE_OUT_AUX_DIGITAL },
    { AudioSystem::DEVICE_OUT_ANLG_DOCK_HEADSET, AUDIO_DEVICE_OUT_ANLG_DOCK_HEADSET },
    { AudioSystem::DEVICE_OUT_DGTL_DOCK_HEADSET, AUDIO_DEVICE_OUT_DGTL_DOCK_HEADSET },
    { AudioSystem::DEVICE_OUT_DEFAULT, AUDIO_DEVICE_OUT_DEFAULT },//默认设备
    ......
}
```
大家可能会有疑问：

Ø 这么多的输出设备，那么当我们回放音频流(录音也是类似的情况)时，该选择哪一种呢？

Ø 而且当前系统中audio interface也很可能不止一个，应该如何选择？

显然这些决策工作将由AudioPolicyService来完成，我们会在下一小节做详细阐述。这里先给大家分析下，AudioFlinger是如何打开一个Output通道的(一个audiointerface可能包含若干个output)。
##### 1.2.2、打开音频输出通道openOutput()
打开音频输出通道(output)在AudioFlinger中对应的接口是openOutput()，即：

``` cpp
[->\frameworks\av\services\audioflinger\AudioFlinger.cpp]
status_t AudioFlinger::openOutput(audio_module_handle_t module,
                                  audio_io_handle_t *output,
                                  audio_config_t *config,
                                  audio_devices_t *devices,
                                  const String8& address,
                                  uint32_t *latencyMs,
                                  audio_output_flags_t flags)
{
    ......

    Mutex::Autolock _l(mLock);

    sp<PlaybackThread> thread = openOutput_l(module, output, config, *devices, address, flags);
    if (thread != 0) {
        *latencyMs = thread->latency();

        // notify client processes of the new output creation
        thread->ioConfigChanged(AUDIO_OUTPUT_OPENED);

        // the first primary output opened designates the primary hw device
        if ((mPrimaryHardwareDev == NULL) && (flags & AUDIO_OUTPUT_FLAG_PRIMARY)) {
            ALOGI("Using module %d has the primary audio interface", module);
            mPrimaryHardwareDev = thread->getOutput()->audioHwDev;

            AutoMutex lock(mHardwareLock);
            mHardwareStatus = AUDIO_HW_SET_MODE;
            mPrimaryHardwareDev->hwDevice()->set_mode(mPrimaryHardwareDev->hwDevice(), mMode);
            mHardwareStatus = AUDIO_HW_IDLE;
        }
        return NO_ERROR;
    }

    return NO_INIT;
}
```
继续调用 openOutput_l()函数从处理。
``` cpp
[->\frameworks\av\services\audioflinger\AudioFlinger.cpp]
sp<AudioFlinger::PlaybackThread> AudioFlinger::openOutput_l(audio_module_handle_t module,
                                                            audio_io_handle_t *output,
                                                            audio_config_t *config,
                                                            audio_devices_t devices,
                                                            const String8& address,
                                                            audio_output_flags_t flags)
{
    /*Step 1. 查找相应的audio interface
    AudioHwDevice *outHwDev = findSuitableHwDev_l(module, devices);
    ......
    mHardwareStatus = AUDIO_HW_OUTPUT_OPEN;

     ......

    AudioStreamOut *outputStream = NULL;
     /*Step 2. 为设备打开一个输出流*/
    status_t status = outHwDev->openOutputStream(
            &outputStream,
            *output,
            devices,
            flags,
            config,
            address.string());

    mHardwareStatus = AUDIO_HW_IDLE;
    /*Step 3.创建PlaybackThread*/
    if (status == NO_ERROR) {

        PlaybackThread *thread;
        if (flags & AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD) {
            thread = new OffloadThread(this, outputStream, *output, devices, mSystemReady);
            ALOGV("openOutput_l() created offload output: ID %d thread %p", *output, thread);
        } else if ((flags & AUDIO_OUTPUT_FLAG_DIRECT)
                || !isValidPcmSinkFormat(config->format)
                || !isValidPcmSinkChannelMask(config->channel_mask)) {
            thread = new DirectOutputThread(this, outputStream, *output, devices, mSystemReady);
            ALOGV("openOutput_l() created direct output: ID %d thread %p", *output, thread);
        } else {
            thread = new MixerThread(this, outputStream, *output, devices, mSystemReady);
            ALOGV("openOutput_l() created mixer output: ID %d thread %p", *output, thread);
        }
        mPlaybackThreads.add(*output, thread);
        return thread;
    }

    return 0;
}

```

上面这段代码中，颜色加深的部分是我们接下来分析的重点，主要还是围绕outHwDev这个变量所做的一系列操作，即：

·        查找合适的音频接口设备( findSuitableHwDev_l() )

·        创建音频输出流( 通过openOutputStream()创建AudioStreamOut )

·        创建播放线程( PlaybackThread )

outHwDev用于记录一个打开的音频接口设备，它的数据类型是audio_hw_device_t，是由HAL规定的一个音频接口设备所应具有的属性集合，如下所示：

``` cpp 
[->\frameworks\av\services\audioflinger\AudioHwDevice.h]
class AudioHwDevice {
......
    audio_module_handle_t handle() const { return mHandle; }
    const char *moduleName() const { return mModuleName; }
    audio_hw_device_t *hwDevice() const { return mHwDevice; }
    uint32_t version() const { return mHwDevice->common.version; }
    
    status_t openOutputStream(
            AudioStreamOut **ppStreamOut,
            audio_io_handle_t handle,
            audio_devices_t devices,
            audio_output_flags_t flags,
            struct audio_config *config,
            const char *address);
private:
    const audio_module_handle_t mHandle;
    const char * const          mModuleName;
    audio_hw_device_t * const   mHwDevice;
    const Flags                 mFlags;
};
}
```

其中common代表了HAL层所有设备的共有属性;set_master_volume、set_mode、open_output_stream分别为我们设置audio interface的主音量、设置音频模式类型(比如AUDIO_MODE_RINGTONE、AUDIO_MODE_IN_CALL等等)、打开输出数据流提供了接口。

接下来我们分步来阐述。
##### 1.2.2.1、查找合适的音频接口设备findSuitableHwDev_l()
Step1@ AudioFlinger::openOutput. 在openOutput中，设备outHwDev是通过查找当前系统来得到的，代码如下：

``` cpp
[->\frameworks\av\services\audioflinger\AudioFlinger.cpp]
AudioHwDevice* AudioFlinger::findSuitableHwDev_l(
        audio_module_handle_t module,
        audio_devices_t devices)
{
    // if module is 0, the request comes from an old policy manager and we should load
    // well known modules
    if (module == 0) {
        ALOGW("findSuitableHwDev_l() loading well know audio hw modules");
        for (size_t i = 0; i < ARRAY_SIZE(audio_interfaces); i++) {
            loadHwModule_l(audio_interfaces[i]);
        }
        // then try to find a module supporting the requested device.
        for (size_t i = 0; i < mAudioHwDevs.size(); i++) {
            AudioHwDevice *audioHwDevice = mAudioHwDevs.valueAt(i);
            audio_hw_device_t *dev = audioHwDevice->hwDevice();
            if ((dev->get_supported_devices != NULL) &&
                    (dev->get_supported_devices(dev) & devices) == devices)
                return audioHwDevice;
        }
    } else {
        // check a match for the requested module handle
        AudioHwDevice *audioHwDevice = mAudioHwDevs.valueFor(module);
        if (audioHwDevice != NULL) {
            return audioHwDevice;
        }
    }

    return NULL;
}
```
变量module值为0的情况，是为了兼容之前的Audio Policy而特别做的处理。当module等于0时，首先加载所有已知的音频接口设备，然后再根据devices来确定其中符合要求的。入参devices的值实际上来源于“ Android系统支持的音频设备列表(输出)”所示的设备。可以看到，enum中每个设备类型都对应一个特定的比特位，因而上述代码段中可以通过“与运算”来找到匹配的设备。

当modules为非0值时，说明Audio Policy指定了具体的设备id号，这时就通过查找全局的mAudioHwDevs变量来确认是否存在符合要求的设备。

``` cpp
[->\frameworks\av\services\audioflinger\AudioFlinger.h]
DefaultKeyedVector<audio_module_handle_t, AudioHwDevice*>  mAudioHwDevs;
```

变量mAudioHwDevs是一个Vector，以audio_module_handle_t为key，每一个handle值唯一确定了已经添加的音频设备。那么在什么时候添加设备呢？

一种情况就是前面看到的modules为0时，会load所有潜在设备，另一种情况就是AudioPolicyManagerBase在构造时会预加载所有audio_policy.conf中所描述的output。不管是哪一种情况，最终都会调用loadHwModuleàloadHwModule_l，这个函数我们开头就分析过了。

如果modules为非0，且从mAudioHwDevs中也找不到符合要求的设备，程序并不会就此终结——它会退而求其次，遍历数组中的所有元素寻找支持devices的任何一个audio interface。
##### 1.2.2.2、创建音频输出流openOutputStream()
Step2@ AudioFlinger::openOutput，调用openOutputStream()函数打开一个AudioStreamOut 。源码实现如下：

``` cpp
[->\frameworks\av\services\audioflinger\AudioHwDevice.cpp]
status_t AudioHwDevice::openOutputStream(
        AudioStreamOut **ppStreamOut,
        audio_io_handle_t handle,
        audio_devices_t devices,
        audio_output_flags_t flags,
        struct audio_config *config,
        const char *address)
{

    struct audio_config originalConfig = *config;
    AudioStreamOut *outputStream = new AudioStreamOut(this, flags);
    
    status_t status = outputStream->open(handle, devices, config, address);

    ......
    *ppStreamOut = outputStream;
    return status;
}
```
生成AudioStreamOut对象并赋值给ppStreamOut ，进一步调用了AudioStreamOut->open()函数。

``` cpp
[->\frameworks\av\services\audioflinger\AudioStreamOut.cpp]
status_t AudioStreamOut::open(
        audio_io_handle_t handle,
        audio_devices_t devices,
        struct audio_config *config,
        const char *address)
{
    audio_stream_out_t *outStream;
    .......
    int status = hwDev()->open_output_stream(
            hwDev(),
            handle,
            devices,
            customFlags,
            config,
            &outStream,
            address);
    ......
    return status;
}
```
即会通过audio_hw_device_t->->open_output_stream()创建音频输出流

##### 1.2.2.2.1、audio_hw_device_t->->open_output_stream()
我们先看一下HAL层代码

``` cpp
[->\hardware\qcom\audio\hal\audio_hw.c]
static int adev_open(const hw_module_t *module, const char *name,
                     hw_device_t **device)
{
    ......
    adev = calloc(1, sizeof(struct audio_device));

    pthread_mutex_init(&adev->lock, (const pthread_mutexattr_t *) NULL);

    adev->device.common.tag = HARDWARE_DEVICE_TAG;
    adev->device.common.version = AUDIO_DEVICE_API_VERSION_2_0;
    adev->device.common.module = (struct hw_module_t *)module;
    adev->device.common.close = adev_close;

    adev->device.init_check = adev_init_check;
    adev->device.set_voice_volume = adev_set_voice_volume;
    adev->device.set_master_volume = adev_set_master_volume;
    adev->device.get_master_volume = adev_get_master_volume;
    adev->device.set_master_mute = adev_set_master_mute;
    adev->device.get_master_mute = adev_get_master_mute;
    adev->device.set_mode = adev_set_mode;
    adev->device.set_mic_mute = adev_set_mic_mute;
    adev->device.get_mic_mute = adev_get_mic_mute;
    adev->device.set_parameters = adev_set_parameters;
    adev->device.get_parameters = adev_get_parameters;
    adev->device.get_input_buffer_size = adev_get_input_buffer_size;
    adev->device.open_output_stream = adev_open_output_stream;
    adev->device.close_output_stream = adev_close_output_stream;
    adev->device.open_input_stream = adev_open_input_stream;
    adev->device.close_input_stream = adev_close_input_stream;
    ......
}
```
可以看到当调用open_output_stream 就会调用adev_open_output_stream。

``` cpp
[->\hardware\qcom\audio\hal\audio_hw.c]
static int adev_open_output_stream(struct audio_hw_device *dev,
                                   audio_io_handle_t handle,
                                   audio_devices_t devices,
                                   audio_output_flags_t flags,
                                   struct audio_config *config,
                                   struct audio_stream_out **stream_out,
                                   const char *address __unused)
{
    struct audio_device *adev = (struct audio_device *)dev;
    struct stream_out *out;
    int i, ret;
    
    *stream_out = NULL;
    out = (struct stream_out *)calloc(1, sizeof(struct stream_out));

    ......
    out->stream.common.get_sample_rate = out_get_sample_rate;
    out->stream.common.set_sample_rate = out_set_sample_rate;
    out->stream.common.get_buffer_size = out_get_buffer_size;
    out->stream.common.get_channels = out_get_channels;
    out->stream.common.get_format = out_get_format;
    out->stream.common.set_format = out_set_format;
    out->stream.common.standby = out_standby;
    out->stream.common.dump = out_dump;
    out->stream.common.set_parameters = out_set_parameters;
    out->stream.common.get_parameters = out_get_parameters;
    out->stream.common.add_audio_effect = out_add_audio_effect;
    out->stream.common.remove_audio_effect = out_remove_audio_effect;
    out->stream.get_latency = out_get_latency;
    out->stream.set_volume = out_set_volume;
#ifdef NO_AUDIO_OUT
    out->stream.write = out_write_for_no_output;
#else
    out->stream.write = out_write;
#endif
    out->stream.get_render_position = out_get_render_position;
    out->stream.get_next_write_timestamp = out_get_next_write_timestamp;
    out->stream.get_presentation_position = out_get_presentation_position;

    out->af_period_multiplier  = out->realtime ? af_period_multiplier : 1;
    out->standby = 1;
    ......

    *stream_out = &out->stream;
    ALOGV("%s: exit", __func__);
    return 0;
}
```
根据音频流的熟悉做一系列初始化操作。转了一大圈，继续看看
##### 1.2.2.3、创建播放线程(PlaybackThread)
Step3@ AudioFlinger::openOutput. 既然通道已经打开，那么由谁来往通道里放东西呢？这就是PlaybackThread。这里分三种不同的情况：
·        OffloadThread

·        DirectOutput

如果不需要混音

·        Mixer

需要混音

这三种情况分别对应DirectOutputThread、OffloadThread和MixerThread两种线程。我们以后者为例来分析下PlaybackThread的工作模式，也会后面小节打下基础。回放线程（PlaybackThread 及其派生的子类）和录制线程（RecordThread）进行的，先简单看看回放线程和录制线程类关系：
![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/audio.system/32-Audio-system-mixerthread.jpg)

·       ThreadBase：PlaybackThread 和 RecordThread 的基类
·       RecordThread：录制线程类，由 ThreadBase 派生
·       PlaybackThread：回放线程基类，同由 ThreadBase 派生
·       MixerThread：混音回放线程类，由 PlaybackThread 派生，负责处理标识为 AUDIO_OUTPUT_FLAG_PRIMARY、AUDIO_OUTPUT_FLAG_FAST、AUDIO_OUTPUT_FLAG_DEEP_BUFFER 的音频流，MixerThread 可以把多个音轨的数据混音后再输出
·       DirectOutputThread：直输回放线程类，由 PlaybackThread 派生，负责处理标识为 AUDIO_OUTPUT_FLAG_DIRECT 的音频流，这种音频流数据不需要软件混音，直接输出到音频设备即可
·       DuplicatingThread：复制回放线程类，由 MixerThread 派生，负责复制音频流数据到其他输出设备，使用场景如主声卡设备、蓝牙耳机设备、USB 声卡设备同时输出
·       OffloadThread：硬解回放线程类，由 DirectOutputThread 派生，负责处理标识为 AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD 的音频流，这种音频流未经软件解码的（一般是 MP3、AAC 等格式的数据），需要输出到硬件解码器，由硬件解码器解码成 PCM 数据

PlaybackThread 中有个极为重要的函数 threadLoop()，当 PlaybackThread 被强引用时，threadLoop() 会真正运行起来进入循环主体，处理音频流数据相关事务，threadLoop() 大致流程如下（以 MixerThread 为例）：

``` cpp
[->\frameworks\av\services\audioflinger\Threads.cpp]
bool AudioFlinger::PlaybackThread::threadLoop()
{
    // ......

    while (!exitPending())
    {
        // ......

        { // scope for mLock

            Mutex::Autolock _l(mLock);

            processConfigEvents_l();

            // ......

            if ((!mActiveTracks.size() && systemTime() > mStandbyTimeNs) ||
                                   isSuspended()) {
                // put audio hardware into standby after short delay
                if (shouldStandby_l()) {

                    threadLoop_standby();

                    mStandby = true;
                }

                // ......
            }
            // mMixerStatusIgnoringFastTracks is also updated internally
            mMixerStatus = prepareTracks_l(&tracksToRemove);

            // ......
        } // mLock scope ends

        // ......

        if (mBytesRemaining == 0) {
            mCurrentWriteLength = 0;
            if (mMixerStatus == MIXER_TRACKS_READY) {
                // threadLoop_mix() sets mCurrentWriteLength
                threadLoop_mix();
            }
            // ......
        }

        // ......

        if (!waitingAsyncCallback()) {
            // mSleepTimeUs == 0 means we must write to audio hardware
            if (mSleepTimeUs == 0) {
                // ......
                if (mBytesRemaining) {
                    // FIXME rewrite to reduce number of system calls
                    ret = threadLoop_write();
                    lastWriteFinished = systemTime();
                    delta = lastWriteFinished - mLastWriteTime;
                    if (ret < 0) {
                        mBytesRemaining = 0;
                    } else {
                        mBytesWritten += ret;
                        mBytesRemaining -= ret;
                        mFramesWritten += ret / mFrameSize;
                    }
                }
                // ......
            }
            // ......
        }

        // Finally let go of removed track(s), without the lock held
        // since we can't guarantee the destructors won't acquire that
        // same lock.  This will also mutate and push a new fast mixer state.
        threadLoop_removeTracks(tracksToRemove);
        tracksToRemove.clear();

        // ......
    }

    threadLoop_exit();

    if (!mStandby) {
        threadLoop_standby();
        mStandby = true;
    }

    // ......
    return false;
}
```
threadLoop() 循环的条件是 exitPending() 返回 false，如果想要 PlaybackThread 结束循环，则可以调用 requestExit() 来请求退出；
processConfigEvents_l() ：处理配置事件；当有配置改变的事件发生时，需要调用 sendConfigEvent_l() 来通知 PlaybackThread，这样 PlaybackThread 才能及时处理配置事件；常见的配置事件是切换音频通路；
检查此时此刻是否符合 standby 条件，比如当前并没有 ACTIVE 状态的 Track（mActiveTracks.size() = 0），那么调用 threadLoop_standby() 关闭音频硬件设备以节省能耗；
prepareTracks_l()： 准备音频流和混音器，该函数非常复杂，这里不详细分析了，仅列一下流程要点： 
遍历 mActiveTracks，逐个处理 mActiveTracks 上的 Track，检查该 Track 是否为 ACTIVE 状态；
如果 Track 设置是 ACTIVE 状态，则再检查该 Track 的数据是否准备就绪了；
根据音频流的音量值、格式、声道数、音轨的采样率、硬件设备的采样率，配置好混音器参数；
如果 Track 的状态是 PAUSED 或 STOPPED，则把该 Track 添加到 tracksToRemove 向量中；
threadLoop_mix()：读取所有置了 ACTIVE 状态的音频流数据，混音器开始处理这些数据；
threadLoop_write()： 把混音器处理后的数据写到输出流设备；
threadLoop_removeTracks()： 把 tracksToRemove 上的所有 Track 从 mActiveTracks 中移除出来；这样下一次循环时就不会处理这些 Track 了。
这里说说 PlaybackThread 与输出流设备的关系：PlaybackThread 实例与输出流设备是一一对应的，比方说 OffloadThread 只会将音频数据输出到 compress_offload 设备中，MixerThread(with FastMixer) 只会将音频数据输出到 low_latency 设备中。

从 Audio HAL 中，我们通常看到如下 4 种输出流设备，分别对应着不同的播放场景：

primary_out：主输出流设备，用于铃声类声音输出，对应着标识为 AUDIO_OUTPUT_FLAG_PRIMARY 的音频流和一个 MixerThread 回放线程实例
low_latency：低延迟输出流设备，用于按键音、游戏背景音等对时延要求高的声音输出，对应着标识为 AUDIO_OUTPUT_FLAG_FAST 的音频流和一个 MixerThread 回放线程实例
deep_buffer：音乐音轨输出流设备，用于音乐等对时延要求不高的声音输出，对应着标识为 AUDIO_OUTPUT_FLAG_DEEP_BUFFER 的音频流和一个 MixerThread 回放线程实例
compress_offload：硬解输出流设备，用于需要硬件解码的数据输出，对应着标识为 AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD 的音频流和一个 OffloadThread 回放线程实例
其中 primary_out 设备是必须声明支持的，而且系统启动时就已经打开 primary_out 设备并创建好对应的 MixerThread 实例。其他类型的输出流设备并非必须声明支持的，主要是看硬件上有无这个能力。

可能有人产生这样的疑问：既然 primary_out 设备一直保持打开，那么能耗岂不是很大？这里阐释一个概念：输出流设备属于逻辑设备，并不是硬件设备。所以即使输出流设备一直保持打开，只要硬件设备不工作，那么就不会影响能耗。那么硬件设备什么时候才会打开呢？答案是 PlaybackThread 将音频数据写入到输出流设备时。

下图简单描述 AudioTrack、PlaybackThread、输出流设备三者的对应关系：
![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/audio.system/33-Audio-system-Audio-playback-.png)

我们可以这么说：输出流设备决定了它对应的 PlaybackThread 是什么类型。怎么理解呢？意思是说：只有支持了该类型的输出流设备，那么该类型的 PlaybackThread 才有可能被创建。举个例子：只有硬件上具备硬件解码器，系统才建立 compress_offload 设备，然后播放 mp3 格式的音乐文件时，才会创建 OffloadThread 把数据输出到 compress_offload 设备上；反之，如果硬件上并不具备硬件解码器，系统则不应该建立 compress_offload 设备，那么播放 mp3 格式的音乐文件时，通过 MixerThread 把数据输出到其他输出流设备上。

那么有无可能出现这种情况：底层并不支持 compress_offload 设备，但偏偏有个标识为 AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD 的音频流送到 AudioFlinger 了呢？这是不可能的。系统启动时，会检查并保存输入输出流设备的支持信息；播放器在播放 mp3 文件时，首先看 compress_offload 设备是否支持了，如果支持，那么不进行软件解码，直接把数据标识为 AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD；如果不支持，那么先进行软件解码，然后把解码好的数据标识为 AUDIO_OUTPUT_FLAG_DEEP_BUFFER，前提是 deep_buffer 设备是支持了的；如果 deep_buffer 设备也不支持，那么把数据标识为 AUDIO_OUTPUT_FLAG_PRIMARY。

系统启动时，就已经打开 primary_out、low_latency、deep_buffer 这三种输出流设备，并创建对应的 MixerThread 了；而此时 DirectOutputThread 与 OffloadThread 不会被创建，直到标识为 AUDIO_OUTPUT_FLAG_DIRECT/AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD 的音频流需要输出时，才开始创建 DirectOutputThread/OffloadThread 和打开 direct_out/compress_offload 设备。这一点请参考如下代码，注释非常清晰：

``` cpp
[->\frameworks\av\services\audiopolicy\managerdefault\AudioPolicyManager.cpp]
AudioPolicyManager::AudioPolicyManager(AudioPolicyClientInterface *clientInterface)
    // ......
{
    // ......

    // mAvailableOutputDevices and mAvailableInputDevices now contain all attached devices
    // open all output streams needed to access attached devices
    // ......
    for (size_t i = 0; i < mHwModules.size(); i++) {
        // ......
        // open all output streams needed to access attached devices
        // except for direct output streams that are only opened when they are actually
        // required by an app.
        // This also validates mAvailableOutputDevices list
        for (size_t j = 0; j < mHwModules[i]->mOutputProfiles.size(); j++)
        {
            // ......
            if ((outProfile->getFlags() & AUDIO_OUTPUT_FLAG_DIRECT) != 0) {
                continue;
            }
            // ......
            audio_io_handle_t output = AUDIO_IO_HANDLE_NONE;
            status_t status = mpClientInterface->openOutput(outProfile->getModuleHandle(),
                                                            &output,
                                                            &config,
                                                            &outputDesc->mDevice,
                                                            address,
                                                            &outputDesc->mLatency,
                                                            outputDesc->mFlags);

            // ......
        }
        // open input streams needed to access attached devices to validate
        // mAvailableInputDevices list
        for (size_t j = 0; j < mHwModules[i]->mInputProfiles.size(); j++)
        {
            // ......
            status_t status = mpClientInterface->openInput(inProfile->getModuleHandle(),
                                                           &input,
                                                           &config,
                                                           &inputDesc->mDevice,
                                                           address,
                                                           AUDIO_SOURCE_MIC,
                                                           AUDIO_INPUT_FLAG_NONE);

            // ......
        }
    }
    // ......

    updateDevicesAndOutputs();
}
```
其中 mpClientInterface->openOutput() 最终会调用到 AudioFlinger::openOutput()：打开输出流设备，并创建 PlaybackThread 对象：

``` cpp

status_t AudioFlinger::openOutput(audio_module_handle_t module,
                                  audio_io_handle_t *output,
                                  audio_config_t *config,
                                  audio_devices_t *devices,
                                  const String8& address,
                                  uint32_t *latencyMs,
                                  audio_output_flags_t flags)
{
    // ......

    Mutex::Autolock _l(mLock);

    sp<PlaybackThread> thread = openOutput_l(module, output, config, *devices, address, flags);
    // ......
}

sp<AudioFlinger::PlaybackThread> AudioFlinger::openOutput_l(audio_module_handle_t module,
                                                            audio_io_handle_t *output,
                                                            audio_config_t *config,
                                                            audio_devices_t devices,
                                                            const String8& address,
                                                            audio_output_flags_t flags)
{
    // ......
    // 分配全局唯一的 audio_io_handle_t，可以理解它是回放线程的索引号
    if (*output == AUDIO_IO_HANDLE_NONE) {
        *output = nextUniqueId(); 
    }

    mHardwareStatus = AUDIO_HW_OUTPUT_OPEN;

    //......

    // 打开音频输出流设备，HAL 层根据 flags 选择打开相关类型的输出流设备
    AudioStreamOut *outputStream = NULL;
    status_t status = outHwDev->openOutputStream(
            &outputStream,
            *output,
            devices,
            flags,
            config,
            address.string());

    mHardwareStatus = AUDIO_HW_IDLE;

    if (status == NO_ERROR) {

        PlaybackThread *thread;
        if (flags & AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD) {
            // AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD 音频流，创建 OffloadThread 实例
            thread = new OffloadThread(this, outputStream, *output, devices, mSystemReady);
            ALOGV("openOutput_l() created offload output: ID %d thread %p", *output, thread);
        } else if ((flags & AUDIO_OUTPUT_FLAG_DIRECT)
                || !isValidPcmSinkFormat(config->format)
                || !isValidPcmSinkChannelMask(config->channel_mask)) {
            // AUDIO_OUTPUT_FLAG_DIRECT 音频流，创建 DirectOutputThread 实例
            thread = new DirectOutputThread(this, outputStream, *output, devices, mSystemReady);
            ALOGV("openOutput_l() created direct output: ID %d thread %p", *output, thread);
        } else {
            // 其他标识的音频流，创建 MixerThread 实例
            thread = new MixerThread(this, outputStream, *output, devices, mSystemReady);
            ALOGV("openOutput_l() created mixer output: ID %d thread %p", *output, thread);
        }
        // 把 audio_io_handle_t 和 PlaybackThread 添加到键值对向量 mPlaybackThreads 中
        // 键值对向量 mPlaybackThreads 中，由于 audio_io_handle_t 和 PlaybackThread 是一
        // 一对应的关系，所以拿到一个 audio_io_handle_t，就能找到它对应的 PlaybackThread
        // 所以可以理解 audio_io_handle_t 为 PlaybackThread 的索引号
        mPlaybackThreads.add(*output, thread);
        return thread;
    }

    return 0;
}
```
##### 1.2.3、AudioFlinger 音频流管理

AudioFlinger 音频流管理由 AudioFlinger::PlaybackThread::Track 实现，Track 与 AudioTrack 是一对一的关系，一个 AudioTrack 创建后，那么 AudioFlinger 会创建一个 Track 与之对应；PlaybackThread 与 AudioTrack/Track 是一对多的关系，一个 PlaybackThread 可以挂着多个 Track。

具体来说：AudioTrack 创建后，AudioPolicyManager 根据 AudioTrack 的输出标识和流类型，找到对应的输出流设备和 PlaybackThread（如果没有找到的话，则系统会打开对应的输出流设备并新建一个 PlaybackThread），然后创建一个 Track 并挂到这个 PlaybackThread 下面。

PlaybackThread 有两个私有成员向量与此强相关：

·       mTracks：该 PlaybackThread 创建的所有 Track 均添加保存到这个向量中
·       mActiveTracks：只有需要播放（设置了 ACTIVE 状态）的 Track 会添加到这个向量中；PlaybackThread 会从该向量上找到所有设置了 ACTIVE 状态的 Track，把这些 Track 数据混音后写到输出流设备
音频流控制最常用的三个接口：

AudioFlinger::PlaybackThread::Track::start：开始播放：把该 Track 置 ACTIVE 状态，然后添加到 mActiveTracks 向量中，最后调用 AudioFlinger::PlaybackThread::broadcast_l() 告知 PlaybackThread 情况有变
·       AudioFlinger::PlaybackThread::Track::stop：停止播放：把该 Track 置 STOPPED 状态，最后调用 AudioFlinger::PlaybackThread::broadcast_l() 告知 PlaybackThread 情况有变
·       AudioFlinger::PlaybackThread::Track::pause：暂停播放：把该 Track 置 PAUSING 状态，最后调用 AudioFlinger::PlaybackThread::broadcast_l() 告知 PlaybackThread 情况有变
·       AudioFlinger::PlaybackThread::threadLoop() 得悉情况有变后，调用 prepareTracks_l() 重新准备音频流和混音器：ACTIVE 状态的 Track 会添加到 mActiveTracks，此外的 Track 会从 mActiveTracks 上移除出来，然后重新准备 AudioMixer。

可见这三个音频流控制接口是非常简单的，主要是设置一下 Track 的状态，然后发个事件通知 PlaybackThread 就行，复杂的处理都在 AudioFlinger::PlaybackThread::threadLoop() 中了。

#### (二)、深入剖析Android音频之AudioPolicyService

AudioPolicyService是策略的制定者，比如什么时候打开音频接口设备、某种Stream类型的音频对应什么设备等等。而AudioFlinger则是策略的执行者，例如具体如何与音频设备通信，如何维护现有系统中的音频设备，以及多个音频流的混音如何处理等等都得由它来完成。AudioPolicyService根据用户配置来指导AudioFlinger加载设备接口，起到路由功能

``` cpp
[->\frameworks\av\media\audioserver\main_audioserver.cpp]
int main(int argc __unused, char **argv)
{
    ......
    if (doLog && (childPid = fork()) != 0) {
    ......
        } else {
        // all other services
        if (doLog) {
            prctl(PR_SET_PDEATHSIG, SIGKILL);   // if parent media.log dies before me, kill me also
            setpgid(0, 0);                      // but if I die first, don't kill my parent
        }
        sp<ProcessState> proc(ProcessState::self());
        sp<IServiceManager> sm = defaultServiceManager();
        ALOGI("ServiceManager: %p", sm.get());
        AudioFlinger::instantiate();
        AudioPolicyService::instantiate();
        RadioService::instantiate();
        SoundTriggerHwService::instantiate();
        ProcessState::self()->startThreadPool();
        IPCThreadState::self()->joinThreadPool();
    }
}

```

AudioPolicyService继承了模板类BinderService，该类用于注册native service。
BinderService是一个模板类，该类的publish函数就是完成向ServiceManager注册服务。

``` cpp
[->\frameworks\av\services\audiopolicy\service\AudioPolicyService.h]
static const char *getServiceName() ANDROID_API { return "media.audio_policy"; }
```
AudioPolicyService注册名为"media.audio_policy"的服务。
首先看看AudioPolicyService的onFirstRef()函数

``` cpp
[->\frameworks\av\services\audiopolicy\service\AudioPolicyService.cpp]

AudioPolicyService::AudioPolicyService()
    : BnAudioPolicyService(), mpAudioPolicyDev(NULL), mpAudioPolicy(NULL),
      mAudioPolicyManager(NULL), mAudioPolicyClient(NULL), mPhoneState(AUDIO_MODE_INVALID)
{
}

void AudioPolicyService::onFirstRef()
{
    {
        Mutex::Autolock _l(mLock);
        /* Step 1:创建AudioCommandThread线程 */
        // start tone playback thread
        mTonePlaybackThread = new AudioCommandThread(String8("ApmTone"), this);
        // start audio commands thread
        mAudioCommandThread = new AudioCommandThread(String8("ApmAudio"), this);
        // start output activity command thread
        mOutputCommandThread = new AudioCommandThread(String8("ApmOutput"), this);

#ifdef USE_LEGACY_AUDIO_POLICY
         // 使用老版本的 audio policy 初始化方式
        ....
#else
        // 使用最新的 audio policy 初始化方式
        ALOGI("AudioPolicyService CSTOR in new mode");
        /* Step 2:创建AudioPolicyClient、 AudioPolicyManager */
        mAudioPolicyClient = new AudioPolicyClient(this);
        mAudioPolicyManager = createAudioPolicyManager(mAudioPolicyClient);
#endif
    }
    // load audio processing modules
    sp<AudioPolicyEffects>audioPolicyEffects = new AudioPolicyEffects();
    {
        Mutex::Autolock _l(mLock);
        mAudioPolicyEffects = audioPolicyEffects;
    }
}
```

先看看总体时序图：

##### 2.1、Step 1:创建AudioCommandThread线程

在AudioPolicyService对象构造过程中，分别创建了ApmTone、ApmAudio、ApmOutput三个AudioCommandThread线程：

1、 ApmTone用于播放tone音；

2、 ApmAudio用于执行audio命令；

3、ApmOutput用于执行输出命令；

在第一次强引用AudioCommandThread线程对象时，AudioCommandThread的onFirstRef函数被回调，在此启动线程

``` cpp
[->\frameworks\av\services\audiopolicy\service\AudioPolicyService.cpp]
void AudioPolicyService::AudioCommandThread::onFirstRef()
{
    run(mName.string(), ANDROID_PRIORITY_AUDIO);
}
```

这里采用异步方式来执行audio command，当需要执行上表中的命令时，首先将命令投递到AudioCommandThread的mAudioCommands命令向量表中，然后通过mWaitWorkCV.signal()唤醒AudioCommandThread线程，被唤醒的AudioCommandThread线程执行完command后，又通过mWaitWorkCV.waitRelative(mLock, waitTime)睡眠等待命令到来。
##### 2.2、Step 2: 创建AudioPolicyClient、 AudioPolicyManager
首先创建AudioPolicyClient 

``` cpp
[->\frameworks\av\services\audiopolicy\service\AudioPolicyService.h]
    class AudioPolicyClient : public AudioPolicyClientInterface
    {
     public:
        AudioPolicyClient(AudioPolicyService *service) : mAudioPolicyService(service) {}
        virtual ~AudioPolicyClient() {}
        // loads a HW module.
        virtual audio_module_handle_t loadHwModule(const char *name);
        ......
        virtual status_t openOutput(audio_module_handle_t module,
                                    audio_io_handle_t *output,
                                    audio_config_t *config,
                                    audio_devices_t *devices,
                                    const String8& address,
                                    uint32_t *latencyMs,
                                    audio_output_flags_t flags);
        ......
        // opens an audio input
        virtual audio_io_handle_t openInput(audio_module_handle_t module,
                                            audio_io_handle_t *input,
                                            audio_config_t *config,
                                            audio_devices_t *devices,
                                            const String8& address,
                                            audio_source_t source,
                                            audio_input_flags_t flags);
        ......

     private:
        AudioPolicyService *mAudioPolicyService;
    };

```

createAudioPolicyManager() 函数的实现位于 frameworks/av/services/audiopolicy/manager/AudioPolicyFactory.cpp文件中。查看源码后我们会发现它实际上是直接调用了 AudioPolicyManager 的构造函数。代码如下：

``` cpp
[->\frameworks\av\services\audiopolicy\manager\AudioPolicyFactory.cpp]
extern "C" AudioPolicyInterface* createAudioPolicyManager(
        AudioPolicyClientInterface *clientInterface)
{
    return new AudioPolicyManager(clientInterface);
}

```

##### 2.3、创建AudioPolicyManager()
总体流程图：
![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/audio.system/34-Audio-system-CreateAudioPolicyManager.png)


AudioPolicyManager 的构造函数将解析音频策略配置文件，从而获取到设备所支持的音频设备信息（包括设备是否支持 Offload、Direct 模式输出，各输入输出 profile 所支持的采样率、通道数、数据格式等），加载全部 HwModule，为之创建所有非 direct 输出类型的 outputStream 和所有 inputStream，并创建相应的 playbackThread 或 recordThread 线程。需要注意的是，Android 7.0上的音频策略配置文件开始使用 XML 格式，其文件名为 audio_policy_configuration.xml，

而在之前的版本上音频策略配置文件为 audio_policy.conf。frameworks/av/services/audiopolicy/managerdefault/AudioPolicyManager.cpp 中 AudioPolicyManager 构造函数的关键代码如下：

``` cpp
[->\frameworks\av\services\audiopolicy\managerdefault\AudioPolicyManager.cpp]
AudioPolicyManager::AudioPolicyManager(AudioPolicyClientInterface *clientInterface)
    :
#ifdef AUDIO_POLICY_TEST
    Thread(false),
#endif //AUDIO_POLICY_TEST
    mLimitRingtoneVolume(false), mLastVoiceVolume(-1.0f),
    mA2dpSuspended(false),
    mAudioPortGeneration(1),
    mBeaconMuteRefCount(0),
    mBeaconPlayingRefCount(0),
    mBeaconMuted(false),
    mTtsOutputAvailable(false),
    mMasterMono(false)
{
    ....

#ifdef USE_XML_AUDIO_POLICY_CONF
    // 设备使用的配置文件为 audio_policy_configuration.xml
    mVolumeCurves = new VolumeCurvesCollection();
    AudioPolicyConfig config(mHwModules, mAvailableOutputDevices, mAvailableInputDevices,
                             mDefaultOutputDevice, speakerDrcEnabled,
                             static_cast<VolumeCurvesCollection *>(mVolumeCurves));
    PolicySerializer serializer;
    // 解析 xml 配置文件，将设备支持的音频输出设备保存在 mAvailableOutputDevices 变量中，
    // 将设备支持的音频输入设备保存在 mAvailableInputDevices 变量中，将设备的默认音频输出
    // 设备保存在 mDefaultOutputDevice 变量中。
    if (serializer.deserialize(AUDIO_POLICY_XML_CONFIG_FILE, config) != NO_ERROR) {
#else
    // 设备使用的配置文件为 audio_policy.conf
    mVolumeCurves = new StreamDescriptorCollection();
    AudioPolicyConfig config(mHwModules, mAvailableOutputDevices, mAvailableInputDevices,
                             mDefaultOutputDevice, speakerDrcEnabled);
    // 优先解析 vendor 目录下的 conf 配置文件，然后解析 device 目录下的 conf 配置文件。
    // 将设备支持的音频输出设备保存在 mAvailableOutputDevices 变量中，
    // 将设备支持的音频输入设备保存在 mAvailableInputDevices 变量中，将设备的默认音频输出
    // 设备保存在 mDefaultOutputDevice 变量中。
    if ((ConfigParsingUtils::loadConfig(AUDIO_POLICY_VENDOR_CONFIG_FILE, config) != NO_ERROR) &&
            (ConfigParsingUtils::loadConfig(AUDIO_POLICY_CONFIG_FILE, config) != NO_ERROR)) {
#endif
        ALOGE("could not load audio policy configuration file, setting defaults");
        config.setDefault();
    }
    // must be done after reading the policy (since conditionned by Speaker Drc Enabling)
    mVolumeCurves->initializeVolumeCurves(speakerDrcEnabled);    // 设置音量调节曲线

    ....
    // 依次加载 HwModule 并打开其所含 profile 的 outputStream 及 inputStream
    for (size_t i = 0; i < mHwModules.size(); i++) {
        mHwModules[i]->mHandle = mpClientInterface->loadHwModule(mHwModules[i]->getName());
        if (mHwModules[i]->mHandle == 0) {
            ALOGW("could not open HW module %s", mHwModules[i]->getName());
            continue;
        }
        // open all output streams needed to access attached devices
        // except for direct output streams that are only opened when they are actually
        // required by an app.
        // This also validates mAvailableOutputDevices list
        // 打开当前 module 下所有非 direct 类型 profile 的 outputStream
        for (size_t j = 0; j < mHwModules[i]->mOutputProfiles.size(); j++)
        {
            const sp<IOProfile> outProfile = mHwModules[i]->mOutputProfiles[j];

            ....
            // 如果当前操作的 module.profile 是 direct 类型，则不为其打开 outputStream
            if ((outProfile->getFlags() & AUDIO_OUTPUT_FLAG_DIRECT) != 0) {
                continue;
            }
            ....
            // 获取采样率、通道数、数据格式等各音频参数
            sp<SwAudioOutputDescriptor> outputDesc = new SwAudioOutputDescriptor(outProfile,
                                                                                 mpClientInterface);
            const DeviceVector &supportedDevices = outProfile->getSupportedDevices();
            const DeviceVector &devicesForType = supportedDevices.getDevicesFromType(profileType);
            String8 address = devicesForType.size() > 0 ? devicesForType.itemAt(0)->mAddress
                    : String8("");

            outputDesc->mDevice = profileType;
            audio_config_t config = AUDIO_CONFIG_INITIALIZER;
            config.sample_rate = outputDesc->mSamplingRate;
            config.channel_mask = outputDesc->mChannelMask;
            config.format = outputDesc->mFormat;
            audio_io_handle_t output = AUDIO_IO_HANDLE_NONE;
            // 为当前 module.profile 打开对应的 outputStream 并创建 playbackThread 线程
            status_t status = mpClientInterface->openOutput(outProfile->getModuleHandle(),
                                                            &output,
                                                            &config,
                                                            &outputDesc->mDevice,
                                                            address,
                                                            &outputDesc->mLatency,
                                                            outputDesc->mFlags);

            ....
        }
        // open input streams needed to access attached devices to validate
        // mAvailableInputDevices list
        // 打开当前 module 下所有 profile 的 inputStream
        for (size_t j = 0; j < mHwModules[i]->mInputProfiles.size(); j++)
        {
            const sp<IOProfile> inProfile = mHwModules[i]->mInputProfiles[j];

            ....
            sp<AudioInputDescriptor> inputDesc =
                    new AudioInputDescriptor(inProfile);

            inputDesc->mDevice = profileType;

            // 获取采样率、通道数、数据格式等各音频参数
            // find the address
            DeviceVector inputDevices = mAvailableInputDevices.getDevicesFromType(profileType);
            //   the inputs vector must be of size 1, but we don't want to crash here
            String8 address = inputDevices.size() > 0 ? inputDevices.itemAt(0)->mAddress
                    : String8("");
            ALOGV("  for input device 0x%x using address %s", profileType, address.string());
            ALOGE_IF(inputDevices.size() == 0, "Input device list is empty!");

            audio_config_t config = AUDIO_CONFIG_INITIALIZER;
            config.sample_rate = inputDesc->mSamplingRate;
            config.channel_mask = inputDesc->mChannelMask;
            config.format = inputDesc->mFormat;
            audio_io_handle_t input = AUDIO_IO_HANDLE_NONE;
            // 为当前 module.profile 打开对应的 inputStream 并创建 recordThread 线程
            status_t status = mpClientInterface->openInput(inProfile->getModuleHandle(),
                                                           &input,
                                                           &config,
                                                           &inputDesc->mDevice,
                                                           address,
                                                           AUDIO_SOURCE_MIC,
                                                           AUDIO_INPUT_FLAG_NONE);

            ....
        }
    }
    ....

    updateDevicesAndOutputs();    // 更新系统缓存的音频输出设备信息
    ....
}
```
AudioPolicyManager对象构造过程中主要完成以下几个步骤：

1、  加载audio_policy_configuration.xml或者audio_policy.conf配置文件

2、 初始化音量调节点initializeVolumeCurves(speakerDrcEnabled)

3、  加载audio policy硬件抽象库：mpClientInterface->loadHwModule(mHwModules[i]->mName)

4、  打开对应的outputStream和inputStream  ： mpClientInterface->openOutput()、mpClientInterface->openInput

5、   更新系统缓存的音频输出设备信息updateDevicesAndOutputs()

##### 2.3.1、加载audio_policy_configuration.xml或者audio_policy.conf配置文件
[audio_policy_configuration.xml](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/audio.system/audio_policy_configuration.xml)
audio_policy.conf同时定义了多个audio 接口，每一个audio 接口包含若干output和input，而每个output和input又同时支持多种输入输出模式，每种输入输出模式又支持若干种设备。
![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/audio.system/35-Audio-system-audio_policy.conf.png)


##### 2.3.2、初始化音量调节点initializeVolumeCurves(speakerDrcEnabled)
在AudioPolicyManagerBase中定义了音量调节对应的音频流描述符数组：

``` cpp
//audio_policy_volumes.xml
const AudioPolicyManagerBase::VolumeCurvePoint
            *AudioPolicyManagerBase::sVolumeProfiles[AudioSystem::NUM_STREAM_TYPES]
                                                   [AudioPolicyManagerBase::DEVICE_CATEGORY_CNT] = {
    { // AUDIO_STREAM_VOICE_CALL
        sDefaultVoiceVolumeCurve, // DEVICE_CATEGORY_HEADSET
        sSpeakerVoiceVolumeCurve, // DEVICE_CATEGORY_SPEAKER
        sDefaultVoiceVolumeCurve  // DEVICE_CATEGORY_EARPIECE
    },
    { // AUDIO_STREAM_SYSTEM
        sHeadsetSystemVolumeCurve, // DEVICE_CATEGORY_HEADSET
        sDefaultSystemVolumeCurve, // DEVICE_CATEGORY_SPEAKER
        sDefaultSystemVolumeCurve  // DEVICE_CATEGORY_EARPIECE
    },
    ......
    }
```

initializeVolumeCurves()函数就是初始化该数组元素：

``` cpp
[->]
void AudioPolicyManagerBase::initializeVolumeCurves()
{
    for (int i = 0; i < AudioSystem::NUM_STREAM_TYPES; i++) {
        for (int j = 0; j < DEVICE_CATEGORY_CNT; j++) {
            mStreams[i].mVolumeCurve[j] =
                    sVolumeProfiles[i][j];
        }
    }

    // Check availability of DRC on speaker path: if available, override some of the speaker curves
    if (mSpeakerDrcEnabled) {
        mStreams[AUDIO_STREAM_SYSTEM].mVolumeCurve[DEVICE_CATEGORY_SPEAKER] =
                sDefaultSystemVolumeCurveDrc;
        mStreams[AUDIO_STREAM_RING].mVolumeCurve[DEVICE_CATEGORY_SPEAKER] =
                sSpeakerSonificationVolumeCurveDrc;
        mStreams[AUDIO_STREAM_ALARM].mVolumeCurve[DEVICE_CATEGORY_SPEAKER] =
                sSpeakerSonificationVolumeCurveDrc;
        mStreams[AUDIO_STREAM_NOTIFICATION].mVolumeCurve[DEVICE_CATEGORY_SPEAKER] =
                sSpeakerSonificationVolumeCurveDrc;
    }
}

```

##### 2.3.3、加载audio policy硬件抽象库loadHwModule()
我们直接分析AudioFlinger::loadHwModule_l()中的load_audio_interface()函数
``` cpp
static int load_audio_interface(const char *if_name, audio_hw_device_t **dev)
{
    const hw_module_t *mod;
    int rc;
    //根据名字加载audio_module模块  
    rc = hw_get_module_by_class(AUDIO_HARDWARE_MODULE_ID, if_name, &mod);
    ALOGE_IF(rc, "%s couldn't load audio hw module %s.%s (%s)", __func__,
                 AUDIO_HARDWARE_MODULE_ID, if_name, strerror(-rc));
    //打开audio_device设备
    rc = audio_hw_device_open(mod, dev);
    ALOGE_IF(rc, "%s couldn't open audio hw device in %s.%s (%s)", __func__,
                 AUDIO_HARDWARE_MODULE_ID, if_name, strerror(-rc));
    
    return 0;

}

```
[->\frameworks\av\services\audioflinger\AudioFlinger.cpp]

``` cpp
static inline int audio_hw_device_open(const struct hw_module_t* module,
                                       struct audio_hw_device** device)
{
    return module->methods->open(module, AUDIO_HARDWARE_INTERFACE,
                                 (struct hw_device_t**)device);
}
```
[->\hardware\libhardware_legacy\audio\audio_hw_hal.cpp]

``` cpp
static int legacy_adev_open(const hw_module_t* module, const char* name,
                            hw_device_t** device)
{
    struct legacy_audio_device *ladev;
    int ret;
    ladev = (struct legacy_audio_device *)calloc(1, sizeof(*ladev));

    ladev->device.common.tag = HARDWARE_DEVICE_TAG;
    ladev->device.common.version = AUDIO_DEVICE_API_VERSION_2_0;
    ladev->device.common.module = const_cast<hw_module_t*>(module);
    ladev->device.common.close = legacy_adev_close;

    ladev->device.init_check = adev_init_check;
    ladev->device.set_voice_volume = adev_set_voice_volume;
    ladev->device.set_master_volume = adev_set_master_volume;
    ladev->device.get_master_volume = adev_get_master_volume;
    ladev->device.set_mode = adev_set_mode;
    ladev->device.set_mic_mute = adev_set_mic_mute;
    ladev->device.get_mic_mute = adev_get_mic_mute;
    ladev->device.set_parameters = adev_set_parameters;
    ladev->device.get_parameters = adev_get_parameters;
    ladev->device.get_input_buffer_size = adev_get_input_buffer_size;
    ladev->device.open_output_stream = adev_open_output_stream;
    ladev->device.close_output_stream = adev_close_output_stream;
    ladev->device.open_input_stream = adev_open_input_stream;
    ladev->device.close_input_stream = adev_close_input_stream;
    ladev->device.dump = adev_dump;

    ladev->hwif = createAudioHardware();

    *device = &ladev->device.common;

    return 0;

}

```
到此就加载完系统定义的所有音频接口，并生成相应的数据对象，如下图所示：'
![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/audio.system/36-Audio-system-audiohwdevice.png)

##### 2.3.4、打开对应的outputStream和inputStream
前面一小节已经分析过outputStream，这里不再分析了
![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/audio.system/37-Audio-system-AudioStreamOut.png)

打开音频输出后，在AudioFlinger与AudioPolicyService中的表现形式如下：
![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/audio.system/38-Audio-system-audiohwdevice-openoutput.png)

打开音频输入:
![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/audio.system/39-Audio-system-AudioStreamIn.png)
打开音频输入后，在AudioFlinger与AudioPolicyService中的表现形式如下：
![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/audio.system/310-Audio-system-audiohwdevice-openinput.png)


##### 2.3.5、 更新系统缓存的音频输出设备信息updateDevicesAndOutputs()

``` cpp
[->\frameworks\av\services\audiopolicy\managerdefault\AudioPolicyManager.cpp]
void AudioPolicyManager::updateDevicesAndOutputs()
{
    for (int i = 0; i < NUM_STRATEGIES; i++) {
        mDeviceForStrategy[i] = getDeviceForStrategy((routing_strategy)i, false /*fromCache*/);
    }
    mPreviousOutputs = mOutputs;
}
```

##### 2.4、总结
->打开音频输出时创建一个audio_stream_out通道，并创建AudioStreamOut对象以及新建PlaybackThread播放线程。

-> 打开音频输入时创建一个audio_stream_in通道，并创建AudioStreamIn对象以及创建RecordThread录音线程。
![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/audio.system/311-Audio-system-audio_stream_in-out.png)

#### (三)、深入剖析Android音频之AudioTrack
现在我们开始分析 AudioTrack 的创建过程，特别留意 AudioTrack 与 AudioFlinger 如何建立联系、用于 AudioTrack 与 AudioFlinger 交换数据的匿名共享内存如何分配。

##### 3.1. AudioTrack & AudioFlinger 相关类
时序图：
![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/audio.system/312-Audio-system-create_audiotrack-flow.png)

首先看一下 AudioTrack & AudioFlinger 的类图，理一下 AudioFlinger 的主要类及其关系、AudioTrack 与 AudioFlinger 之间的联系，后面将以该图为脉络展开分析。
![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/audio.system/313-Audio-system-create_audiotrack.jpg)

☯ AudioFlinger::PlaybackThread：回放线程基类，不同输出标识的音频流对应不同类型的 PlaybackThread 实例（分为四种：MixerThread、DirectOutputThread、DuplicatingThread、OffloadThread），具体见 3.4. AudioFlinger 回放录制线程 小节，所有的 PlaybackThread 实例都会添加到 AudioFlinger.mPlaybackThreads 向量中；这个向量的定义： DefaultKeyedVector< audio_io_handle_t, sp<PlaybackThread> > mPlaybackThreads;，可见 audio_io_handle_t 是与 PlaybackThread 是一一对应的，由已知的 audio_io_handle_t 就能找到对应的 PlaybackThread；audio_io_handle_t 在创建 PlaybackThread 时由系统分配，这个值是全局唯一的
☯ AudioFlinger::PlaybackThread::Track：音频流管理类，创建一块匿名共享内存用于 AudioTrack 与 AudioFlinger 之间的数据交换（方便起见，这块匿名共享内存，以后均简单称为 FIFO），同时实现 start()、stop()、pause() 等音频流常用控制手段；注意，多个 Track 对象可能都注册到同一个 PlaybackThread 中（尤其对于 MixerThread 而言，一个 MixerThread 往往挂着多个 Track 对象），这多个 Track 对象都会添加到 PlaybackThread.mTracks 向量中统一管理
☯ AudioFlinger::TrackHandle：Track 对象只负责音频流管理业务，对外并没有提供跨进程的 Binder 调用接口，而应用进程又需要对音频流进行控制，所以需要一个对象来代理 Track 的跨进程通讯，这个角色就是 TrackHandle，AudioTrack 通过它与 Track 交互
☯ AudioTrack：Android 音频系统对外提供的一个 API 类，负责音频流数据输出；每个音频流对应着一个 AudioTrack 实例，不同输出标识的 AudioTrack 会匹配到不同的 AudioFlinger::PlaybackThread；AudioTrack 与 AudioFlinger::PlaybackThread 之间通过 FIFO 来交换音频数据，AudioTrack 是 FIFO 生产者，AudioFlinger::PlaybackThread 是 FIFO 消费者
☯ AudioTrack::AudioTrackThread：数据传输模式为 TRANSFER_CALLBACK 时，需要创建该线程，它通过调用 audioCallback 回调函数主动从用户进程处索取数据并填充到 FIFO 上；数据传输模式为 TRANSFER_SYNC 时，则不需要创建这个线程，因为用户进程会持续调用 AudioTrack.write() 填充数据到 FIFO；数据传输模式为 TRANSFER_SHARED 时，也不需要创建这个线程，因为用户进程会创建一块匿名共享内存，并把要播放的音频数据一次性拷贝到这块匿名共享内存上了
☯ IAudioTrack：IAudioTrack 是链结 AudioTrack 与 AudioFlinger 的桥梁；它在 AudioTrack 端的对象是 BpAudioTrack，在 AudioFlinger 端的对象是 BnAudioTrack，从图中不难看出，AudioFlinger::TrackHandle 继承自 BnAudioTrack，而 AudioFlinger::TrackHandle 恰恰是AudioFlinger::PlaybackThread::Track 的代理对象，所以 AudioTrack 得到 IAudioTrack 实例后，就可以调用 IAudioTrack 的接口与 AudioFlinger::PlaybackThread::Track 交互

**audio_io_handle_t：**

这里再详细说明一下 audio_io_handle_t，它是 AudioTrack/AudioRecord/AudioSystem、AudioFlinger、AudioPolicyManager 之间一个重要的链结点。3.4. AudioFlinger 回放录制线程 小节在 AudioFlinger::openOutput_l() 注释中大致说明了它的来历及其作用，现在回顾下：当打开输出流设备及创建 PlaybackThread 时，系统会分配一个全局唯一的值作为 audio_io_handle_t，并把 audio_io_handle_t 和 PlaybackThread 添加到键值对向量 mPlaybackThreads 中，由于 audio_io_handle_t 和 PlaybackThread 是一一对应的关系，因此拿到一个 audio_io_handle_t，就能遍历键值对向量 mPlaybackThreads 找到它对应的 PlaybackThread，可以简单理解 audio_io_handle_t 为 PlaybackThread 的索引号或线程 id。由于 audio_io_handle_t 具有 PlaybackThread 索引特性，所以应用进程想获取 PlaybackThread 某些信息的话，只需要传入对应的 audio_io_handle_t 即可。例如 AudioFlinger::format(audio_io_handle_t output)，这是 AudioFlinger 的一个服务接口，用户进程可以通过该接口获取某个 PlaybackThread 配置的音频格式：

``` cpp
[->\frameworks\av\services\audioflinger\AudioFlinger.cpp]
audio_format_t AudioFlinger::format(audio_io_handle_t output) const
{
    Mutex::Autolock _l(mLock);
    // checkPlaybackThread_l() 根据传入的 audio_io_handle_t，从键值对向量
    // mPlaybackThreads 中找到它对应的 PlaybackThread
    PlaybackThread *thread = checkPlaybackThread_l(output);
    if (thread == NULL) {
        ALOGW("format() unknown thread %d", output);
        return AUDIO_FORMAT_INVALID;
    }
    return thread->format();
}

AudioFlinger::PlaybackThread *AudioFlinger::checkPlaybackThread_l(audio_io_handle_t output) const
{
    return mPlaybackThreads.valueFor(output).get();
}
```
##### 3.2. AudioTrack 构造过程
当我们构造一个 AudioTrack 实例时（以 MODE_STREAM/TRANSFER_SYNC 模式为例，这也是最常用的模式了，此时 sharedBuffer 为空），系统都发生了什么事？阐述下大致流程：

如果 cbf（audioCallback 回调函数）非空，那么创建 AudioTrackThread 线程处理 audioCallback 回调函数（MODE_STREAM 模式时，cbf 为空）；
根据 streamType（流类型）、flags（输出标识）等参数调用 AudioSystem::getOutputForAttr()；经过一系列的调用，进入 AudioPolicyManager::getOutputForDevice()： 
如果输出标识置了 AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD 或 AUDIO_OUTPUT_FLAG_DIRECT，那么最终调用 AudioFlinger::openOutput() 打开输出标识对应的输出流设备并创建相应的 PlaybackThread，保存该 PlaybackThread 对应的 audio_io_handle_t 给 AudioTrack；
如果输出标识是其他类型，那么根据策略选择一个输出流设备和 PlaybackThread，并保存该 PlaybackThread 对应的 audio_io_handle_t 给 AudioTrack；别忘了在 3.4. AudioFlinger 回放录制线程 小节中提到：系统启动时，就已经打开 primary_out、low_latency、deep_buffer 这三种输出流设备，并创建对应的 PlaybackThread 了；
通过 Binder 机制调用 AudioFlinger::createTrack()（注意 step2 中 AudioTrack 已经拿到一个 audio_io_handle_t 了，此时把这个 audio_io_handle_t 传入给 createTrack()）： 
根据传入的 audio_io_handle_t 找到它对应的 PlaybackThread；
PlaybackThread 新建一个音频流管理对象 Track；Track 构造时会分配一块匿名共享内存用于 AudioFlinger 与 AudioTrack 的数据交换缓冲区（FIFO）及其控制块（audio_track_cblk_t），并创建一个 AudioTrackServerProxy 对象（PlaybackThread 将使用它从 FIFO 上取得可读数据的位置）；
最后新建一个 Track 的通讯代理 TrackHandle，并以 IAudioTrack 作为返回值给 AudioTrack（TrackHandle、BnAudioTrack、BpAudioTrack、IAudioTrack 的关系见上一个小节）；
通过 IAudioTrack 接口，取得 AudioFlinger 中的 FIFO 控制块（audio_track_cblk_t），由此再计算得到 FIFO 的首地址；
创建一个 AudioTrackClientProxy 对象（AudioTrack 将使用它从 FIFO 上取得可用空间的位置）；
AudioTrack 由此建立了和 AudioFlinger 的全部联系工作：

通过 IAudioTrack 接口可以控制该音轨的状态，例如 start、stop、pause
持续写入数据到 FIFO 上，实现音频连续播放
通过 audio_io_handle_t，可以找到它对应的 PlaybackThread，从而查询该 PlaybackThread 的相关信息，如所设置的采样率、格式等等
构造 1 个 AudioTrack 实例时，AudioFlinger 会有 1 个 PlaybackThread 实例、1 个 Track 实例、1 个 TrackHandle 实例、1 个 AudioTrackServerProxy 实例、1 块 FIFO 与之对应。

当同时构造 1 个 AudioTrack with AUDIO_OUTPUT_FLAG_PRIMARY、1 个 AudioTrack with AUDIO_OUTPUT_FLAG_FAST、3 个 AudioTrack with AUDIO_OUTPUT_FLAG_DEEP_BUFFER、1 个 AudioTrack with AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD、1 个 AudioTrack with AUDIO_OUTPUT_FLAG_DIRECT 时（事实上，Android 音频策略不允许出现这种情形的），AudioFlinger 拥有的 PlaybackThread、Track、TrackHandle 实例如下图所示：
![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/audio.system/314-Audio-system-PlaybackThread-Track-TrackHandle.png)

最后附上相关代码的流程分析，我本意是不多贴代码的，但不上代码总觉得缺点什么，这里我尽量把代码精简，提取主干，忽略细节。

``` cpp
[->\frameworks\av\media\libmedia\AudioTrack.cpp]
AudioTrack::AudioTrack(
        audio_stream_type_t streamType,    // 音频流类型：如 Music、Voice-Call、DTMF、Alarm 等等
        uint32_t sampleRate,               // 采样率：如 16KHz、44.1KHz、48KHz 等等
        audio_format_t format,             // 音频格式：如 PCM、MP3、AAC 等等
        audio_channel_mask_t channelMask,  // 声道数：如 Mono（单声道）、Stereo（双声道）
        const sp<IMemory>& sharedBuffer,   // 共享内存缓冲区：数据模式是 MODE_STATIC 时使用，数据模式是 MODE_STREAM 时为空
        audio_output_flags_t flags,        // 输出标识位，详见 AUDIO_OUTPUT_FLAG 描述
        callback_t cbf,                    // 回调函数
        void* user,                        // 回调函数的参数
        uint32_t notificationFrames,
        int sessionId,
        transfer_type transferType,        // 数据传输类型
        const audio_offload_info_t *offloadInfo,
        int uid,
        pid_t pid,
        const audio_attributes_t* pAttributes,
        bool doNotReconnect)
    : mStatus(NO_INIT),
      mIsTimed(false),
      mPreviousPriority(ANDROID_PRIORITY_NORMAL),
      mPreviousSchedulingGroup(SP_DEFAULT),
      mPausedPosition(0),
      mSelectedDeviceId(AUDIO_PORT_HANDLE_NONE)
{
    mStatus = set(streamType, sampleRate, format, channelMask,
            0 /*frameCount*/, flags, cbf, user, notificationFrames,
            sharedBuffer, false /*threadCanCallJava*/, sessionId, transferType, offloadInfo,
            uid, pid, pAttributes, doNotReconnect);
}

status_t AudioTrack::set(
        audio_stream_type_t streamType,
        uint32_t sampleRate,
        audio_format_t format,
        audio_channel_mask_t channelMask,
        size_t frameCount,
        audio_output_flags_t flags,        
        callback_t cbf,
        void* user,
        uint32_t notificationFrames,
        const sp<IMemory>& sharedBuffer,
        bool threadCanCallJava,
        int sessionId,
        transfer_type transferType,
        const audio_offload_info_t *offloadInfo,
        int uid,
        pid_t pid,
        const audio_attributes_t* pAttributes,
        bool doNotReconnect)
{
    // 参数格式合法性检查、音轨音量初始化

    // 如果 cbf 非空，那么创建 AudioTrackThread 线程处理 audioCallback 回调函数
    if (cbf != NULL) {
        mAudioTrackThread = new AudioTrackThread(*this, threadCanCallJava);
        mAudioTrackThread->run("AudioTrack", ANDROID_PRIORITY_AUDIO, 0 /*stack*/);
        // thread begins in paused state, and will not reference us until start()
    }

    // create the IAudioTrack
    status_t status = createTrack_l();

    //......
}

status_t AudioTrack::createTrack_l()
{
    // 获取 IAudioFlinger，通过 binder 请求 AudioFlinger 服务
    const sp<IAudioFlinger>& audioFlinger = AudioSystem::get_audio_flinger();
    if (audioFlinger == 0) {
        ALOGE("Could not get audioflinger");
        return NO_INIT;
    }

    //......

    // AudioSystem::getOutputForAttr() 经过一系列的调用，进入 AudioPolicyManager::getOutputForDevice()
    // 如果输出标识置了 AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD 或 AUDIO_OUTPUT_FLAG_DIRECT，
    // 那么最终调用 AudioFlinger::openOutput() 打开输出标识对应的输出流设备并创建相关的
    // PlaybackThread，保存该 PlaybackThread 对应的 audio_io_handle_t 给 AudioTrack；
    // 如果输出标识是其他类型，那么根据策略选择一个输出流设备和 PlaybackThread，并保存该
    // PlaybackThread 对应的 audio_io_handle_t 给 AudioTrack
    audio_io_handle_t output;
    status = AudioSystem::getOutputForAttr(attr, &output,
                                           (audio_session_t)mSessionId, &streamType, mClientUid,
                                           mSampleRate, mFormat, mChannelMask,
                                           mFlags, mSelectedDeviceId, mOffloadInfo);

    //......

    // 向 AudioFlinger 发出 createTrack 请求
    sp<IAudioTrack> track = audioFlinger->createTrack(streamType,
                                                      mSampleRate,
                                                      mFormat,
                                                      mChannelMask,
                                                      &temp,
                                                      &trackFlags,
                                                      mSharedBuffer,
                                                      output,
                                                      tid,
                                                      &mSessionId,
                                                      mClientUid,
                                                      &status);
    //......

    // AudioFlinger 创建 Track 对象时会分配一个 FIFO，这里获取 FIFO 的控制块
    sp<IMemory> iMem = track->getCblk();
    if (iMem == 0) {
        ALOGE("Could not get control block");
        return NO_INIT;
    }
    // 匿名共享内存首地址
    void *iMemPointer = iMem->pointer();
    if (iMemPointer == NULL) {
        ALOGE("Could not get control block pointer");
        return NO_INIT;
    }
    mAudioTrack = track; // 保存 AudioFlinger::PlaybackThread::Track 的代理对象 IAudioTrack
    mCblkMemory = iMem; // 保存匿名共享内存首地址

    // 控制块位于 AudioFlinger 分配的匿名共享内存的首部
    audio_track_cblk_t* cblk = static_cast<audio_track_cblk_t*>(iMemPointer);
    mCblk = cblk;
    mOutput = output; // 保存返回的 audio_io_handle_t，用它可以找到对应的 PlaybackThread
    //......

    // update proxy
    if (mSharedBuffer == 0) {
        // 当 mSharedBuffer 为空，意味着音轨数据模式为 MODE_STREAM，那么创建 AudioTrackClientProxy 对象
        mStaticProxy.clear();
        mProxy = new AudioTrackClientProxy(cblk, buffers, frameCount, mFrameSize);
    } else {
        // 当 mSharedBuffer 非空，意味着音轨数据模式为 MODE_STATIC，那么创建 StaticAudioTrackClientProxy 对象
        mStaticProxy = new StaticAudioTrackClientProxy(cblk, buffers, frameCount, mFrameSize);
        mProxy = mStaticProxy;
    }

    //......
}
```
AudioFlinger::createTrack()，顾名思义，创建一个 Track 对象，将用于音频流的控制：

``` cpp
[->\frameworks\av\services\audioflinger\AudioFlinger.cpp]
sp<IAudioTrack> AudioFlinger::createTrack(
        audio_stream_type_t streamType,
        uint32_t sampleRate,
        audio_format_t format,
        audio_channel_mask_t channelMask,
        size_t *frameCount,
        IAudioFlinger::track_flags_t *flags,
        const sp<IMemory>& sharedBuffer,
        audio_io_handle_t output,
        pid_t tid,
        int *sessionId,
        int clientUid,
        status_t *status)
{
    sp<PlaybackThread::Track> track;
    sp<TrackHandle> trackHandle;
    sp<Client> client;
    status_t lStatus;
    int lSessionId;

    //......

    {
        Mutex::Autolock _l(mLock);
        // 根据传入来的 audio_io_handle_t，找到对应的 PlaybackThread
        PlaybackThread *thread = checkPlaybackThread_l(output);
        if (thread == NULL) {
            ALOGE("no playback thread found for output handle %d", output);
            lStatus = BAD_VALUE;
            goto Exit;
        }

        //......

        // 在 PlaybackThread 上创建一个音频流管理对象 Track
        track = thread->createTrack_l(client, streamType, sampleRate, format,
                channelMask, frameCount, sharedBuffer, lSessionId, flags, tid, clientUid, &lStatus);
        //......

        setAudioHwSyncForSession_l(thread, (audio_session_t)lSessionId);
    }

    //......

    // 创建 Track 的通讯代理 TrackHandle 并返回它
    trackHandle = new TrackHandle(track);

Exit:
    *status = lStatus;
    return trackHandle;
}

sp<AudioFlinger::PlaybackThread::Track> AudioFlinger::PlaybackThread::createTrack_l(
        const sp<AudioFlinger::Client>& client,
        audio_stream_type_t streamType,
        uint32_t sampleRate,
        audio_format_t format,
        audio_channel_mask_t channelMask,
        size_t *pFrameCount,
        const sp<IMemory>& sharedBuffer,
        int sessionId,
        IAudioFlinger::track_flags_t *flags,
        pid_t tid,
        int uid,
        status_t *status)
{
    size_t frameCount = *pFrameCount;
    sp<Track> track;
    status_t lStatus;

    bool isTimed = (*flags & IAudioFlinger::TRACK_TIMED) != 0;

    // ......

    { // scope for mLock
        Mutex::Autolock _l(mLock);

        // ......

        if (!isTimed) {
            // 创建 Track，等会再看看 Track 构造函数干些啥
            track = new Track(this, client, streamType, sampleRate, format,
                              channelMask, frameCount, NULL, sharedBuffer,
                              sessionId, uid, *flags, TrackBase::TYPE_DEFAULT);
        } else {
            // 创建 TimedTrack，带时间戳的 Track？这里不深究
            track = TimedTrack::create(this, client, streamType, sampleRate, format,
                    channelMask, frameCount, sharedBuffer, sessionId, uid);
        }

        // ......

        // 把创建的 Track 添加到 mTracks 向量中，方便 PlaybackThread 统一管理
        mTracks.add(track);

        // ......
    }

    lStatus = NO_ERROR;

Exit:
    *status = lStatus;
    return track;
}

// ----------------------------------------------------------------------------
// 如下是 TrackHandle 的相关代码，可以看到，TrackHandle 其实就是一个壳子，是 Track 的包装类
// 所有 TrackHandle 接口都是调向 Track 的
// Google 为什么要搞这么一则？Track 是 PlaybackThread 内部使用的，不适宜对外暴露，但应用进程
// 又确实需要控制音频流的状态（start、stop、pause），所以就采取这么一种方式实现

AudioFlinger::TrackHandle::TrackHandle(const sp<AudioFlinger::PlaybackThread::Track>& track)
    : BnAudioTrack(),
      mTrack(track)
{
}

AudioFlinger::TrackHandle::~TrackHandle() {
    // just stop the track on deletion, associated resources
    // will be freed from the main thread once all pending buffers have
    // been played. Unless it's not in the active track list, in which
    // case we free everything now...
    mTrack->destroy();
}

sp<IMemory> AudioFlinger::TrackHandle::getCblk() const {
    return mTrack->getCblk();
}

status_t AudioFlinger::TrackHandle::start() {
    return mTrack->start();
}

void AudioFlinger::TrackHandle::stop() {
    mTrack->stop();
}

void AudioFlinger::TrackHandle::flush() {
    mTrack->flush();
}

void AudioFlinger::TrackHandle::pause() {
    mTrack->pause();
}
// ----------------------------------------------------------------------------
```

最后，我们看看 Track 的构造过程，主要分析数据 FIFO 及它的控制块是如何分配的：

``` cpp
[->\frameworks\av\services\audioflinger\AudioFlinger.cpp]
AudioFlinger::PlaybackThread::Track::Track(
            PlaybackThread *thread,
            const sp<Client>& client,
            audio_stream_type_t streamType,
            uint32_t sampleRate,
            audio_format_t format,
            audio_channel_mask_t channelMask,
            size_t frameCount,
            void *buffer,
            const sp<IMemory>& sharedBuffer,
            int sessionId,
            int uid,
            IAudioFlinger::track_flags_t flags,
            track_type type)
    :   TrackBase(thread, client, sampleRate, format, channelMask, frameCount,
                  (sharedBuffer != 0) ? sharedBuffer->pointer() : buffer,
                  sessionId, uid, flags, true /*isOut*/,
                  (type == TYPE_PATCH) ? ( buffer == NULL ? ALLOC_LOCAL : ALLOC_NONE) : ALLOC_CBLK,
                  type),
    mFillingUpStatus(FS_INVALID),
    // mRetryCount initialized later when needed
    mSharedBuffer(sharedBuffer),
    mStreamType(streamType),
    mName(-1),  // see note below
    mMainBuffer(thread->mixBuffer()),
    mAuxBuffer(NULL),
    mAuxEffectId(0), mHasVolumeController(false),
    mPresentationCompleteFrames(0),
    mFastIndex(-1),
    mCachedVolume(1.0),
    mIsInvalid(false),
    mAudioTrackServerProxy(NULL),
    mResumeToStopping(false),
    mFlushHwPending(false)
{
    // client == 0 implies sharedBuffer == 0
    ALOG_ASSERT(!(client == 0 && sharedBuffer != 0));

    ALOGV_IF(sharedBuffer != 0, "sharedBuffer: %p, size: %d", sharedBuffer->pointer(),
            sharedBuffer->size());

    // 检查 FIFO 控制块（audio_track_cblk_t）是否分配好了，上面代码并未分配 audio_track_cblk_t
    // 因此只可能是构造 TrackBase 时分配的，等下再看看 TrackBase 的构造函数
    if (mCblk == NULL) {
        return;
    }

    if (sharedBuffer == 0) {
        // 数据传输模式为 MODE_STREAM 模式，创建一个 AudioTrackServerProxy 对象
        // PlaybackThread 将持续使用它从 FIFO 上取得可读数据的位置
        mAudioTrackServerProxy = new AudioTrackServerProxy(mCblk, mBuffer, frameCount,
                mFrameSize, !isExternalTrack(), sampleRate);
    } else {
        // 数据传输模式为 MODE_STATIC 模式，创建一个 StaticAudioTrackServerProxy 对象
        mAudioTrackServerProxy = new StaticAudioTrackServerProxy(mCblk, mBuffer, frameCount,
                mFrameSize);
    }
    mServerProxy = mAudioTrackServerProxy;

    // 为 Track 分配一个名称，AudioMixer 会根据 TrackName 找到对应的 Track
    mName = thread->getTrackName_l(channelMask, format, sessionId);
    if (mName < 0) {
        ALOGE("no more track names available");
        return;
    }
    // ......
}

AudioFlinger::ThreadBase::TrackBase::TrackBase(
            ThreadBase *thread,
            const sp<Client>& client,
            uint32_t sampleRate,
            audio_format_t format,
            audio_channel_mask_t channelMask,
            size_t frameCount,
            void *buffer,
            int sessionId,
            int clientUid,
            IAudioFlinger::track_flags_t flags,
            bool isOut,
            alloc_type alloc,
            track_type type)
    :   RefBase(),
        mThread(thread),
        mClient(client),
        mCblk(NULL),
        // mBuffer
        mState(IDLE),
        mSampleRate(sampleRate),
        mFormat(format),
        mChannelMask(channelMask),
        mChannelCount(isOut ?
                audio_channel_count_from_out_mask(channelMask) :
                audio_channel_count_from_in_mask(channelMask)),
        mFrameSize(audio_is_linear_pcm(format) ?
                mChannelCount * audio_bytes_per_sample(format) : sizeof(int8_t)),
        mFrameCount(frameCount),
        mSessionId(sessionId),
        mFlags(flags),
        mIsOut(isOut),
        mServerProxy(NULL),
        mId(android_atomic_inc(&nextTrackId)),
        mTerminated(false),
        mType(type),
        mThreadIoHandle(thread->id())
{
    // ......

    // ALOGD("Creating track with %d buffers @ %d bytes", bufferCount, bufferSize);
    size_t size = sizeof(audio_track_cblk_t);
    size_t bufferSize = (buffer == NULL ? roundup(frameCount) : frameCount) * mFrameSize;
    if (buffer == NULL && alloc == ALLOC_CBLK) {
        // 这个 size 将是分配的匿名共享内存的大小
        // 等于控制块的大小（sizeof(audio_track_cblk_t)加上数据 FIFO的大小（bufferSize）
        // 待会看到这块内存的结构，就明白这样分配的意义了
        size += bufferSize;
    }

    if (client != 0) {
        // 分配一块匿名共享内存
        mCblkMemory = client->heap()->allocate(size);
        if (mCblkMemory == 0 ||
                (mCblk = static_cast<audio_track_cblk_t *>(mCblkMemory->pointer())) == NULL) {
            ALOGE("not enough memory for AudioTrack size=%u", size);
            client->heap()->dump("AudioTrack");
            mCblkMemory.clear();
            return;
        }
    } else {
        // this syntax avoids calling the audio_track_cblk_t constructor twice
        mCblk = (audio_track_cblk_t *) new uint8_t[size];
        // assume mCblk != NULL
    }

    // construct the shared structure in-place.
    if (mCblk != NULL) {
        // 这是 C++ 的 placement new（定位创建对象）语法：new(@BUFFER) @CLASS();
        // 可以在特定内存位置上构造一个对象
        // 这里，在匿名共享内存首地址上构造了一个 audio_track_cblk_t 对象
        // 这样 AudioTrack 与 AudioFlinger 都能访问这个 audio_track_cblk_t 对象了
        new(mCblk) audio_track_cblk_t();

        // 如下分配数据 FIFO，将用于 AudioTrack 与 AudioFlinger 的数据交换
        switch (alloc) {
        // ......
        case ALLOC_CBLK:
            // clear all buffers
            if (buffer == NULL) {
                // 数据传输模式为 MODE_STREAM/TRANSFER_SYNC 时，数据 FIFO 的分配
                // 数据 FIFO 的首地址紧靠控制块（audio_track_cblk_t）之后
                //   |                                                         |
                //   | -------------------> mCblkMemory <--------------------- |
                //   |                                                         |
                //   +--------------------+------------------------------------+
                //   | audio_track_cblk_t |             Buffer                 |
                //   +--------------------+------------------------------------+
                //   ^                    ^
                //   |                    |
                //   mCblk               mBuffer
                mBuffer = (char*)mCblk + sizeof(audio_track_cblk_t);
                memset(mBuffer, 0, bufferSize);
            } else {
                // 数据传输模式为 MODE_STATIC/TRANSFER_SHARED 时，直接指向 sharedBuffer
                // sharedBuffer 是应用进程分配的匿名共享内存，应用进程已经一次性把数据
                // 写到 sharedBuffer 来了，AudioFlinger 可以直接从这里读取
                //   +--------------------+    +-----------------------------------+
                //   | audio_track_cblk_t |    |            sharedBuffer           |
                //   +--------------------+    +-----------------------------------+
                //   ^                         ^
                //   |                         |
                //   mCblk                    mBuffer
                mBuffer = buffer;
            }
            break;
        // ......
        }

        // ......
    }
}
```
##### 3.3. AudioTrack 数据写入
AudioTrack 实例构造后，应用程序接着可以写入音频数据了。如之前所描述：AudioTrack 与 AudioFlinger 是 生产者-消费者 的关系：

☯ AudioTrack：AudioTrack 在 FIFO 中找到一块可用空间，把用户传入的音频数据写入到这块可用空间上，然后更新写位置（对于 AudioFinger 来说，意味 FIFO 上有更多的可读数据了）；如果用户传入的数据量比可用空间要大，那么要把用户传入的数据拆分多次写入到 FIFO 中（AudioTrack 和 AudioFlinger 是不同的进程，AudioFlinger 同时也在不停地读取数据，所以 FIFO 可用空间是在不停变化的）
☯ AudioFlinger：AudioFlinger 在 FIFO 中找到一块可读数据块，把可读数据拷贝到目的缓冲区上，然后更新读位置（对于 AudioTrack 来说，意味着 FIFO 上有更多的可用空间了）；如果FIFO 上可读数据量比预期的要小，那么要进行多次的读取，才能积累到预期的数据量（AudioTrack 和 AudioFlinger 是不同的进程，AudioTrack 同时也在不停地写入数据，所以 FIFO 可读的数据量是在不停变化的）
上面的过程中，如果 AudioTrack 总能及时生产数据，并且 AudioFlinger 总能及时消耗掉这些数据，那么整个过程将是非常和谐的；但系统可能会发生异常，出现如下的状态：

☯ Block：AudioFlinger 长时间不读取 FIFO 上的可读数据，使得 AudioTrack 长时间获取不到可用空间，无法写入数据；这种情况的根本原因大多是底层驱动发生阻塞异常，导致 AudioFlinger 无法继续写数据到硬件设备中，AudioFlinger 本身并没有错
☯ Underrun：AudioTrack 写入数据的速度跟不上 AudioFlinger 读取数据的速度，使得 AudioFlinger 不能及时获取到预期的数据量，反映到现实的后果就是声音断续；这种情况的根本原因大多是应用程序不能及时写入数据或者缓冲区分配过小，AudioTrack 本身并没有错；AudioFlinger 针对这点做了容错处理：当发现 underrun 时，先陷入短时间的睡眠，不急着读取数据，让应用程序准备更多的数据（如果某一天做应用的哥们意识到自己的错误原来由底层的兄弟默默埋单了，会不会感动得哭了^_^）

##### 3.3.1. AudioTrack 写数据流程
我们看一下 AudioTrack 写数据的代码，流程很简单：obtainBuffer() 在 FIFO 中找到一块可用区间，memcpy() 把用户传入的音频数据拷贝到这个可用区间上，releaseBuffer() 更新写位置。

``` cpp
[->\frameworks\av\media\libmedia\AudioTrack.cpp]
ssize_t AudioTrack::write(const void* buffer, size_t userSize, bool blocking)
{
    if (mTransfer != TRANSFER_SYNC) {
        return INVALID_OPERATION;
    }

    if (isDirect()) {
        AutoMutex lock(mLock);
        int32_t flags = android_atomic_and(
                            ~(CBLK_UNDERRUN | CBLK_LOOP_CYCLE | CBLK_LOOP_FINAL | CBLK_BUFFER_END),
                            &mCblk->mFlags);
        if (flags & CBLK_INVALID) {
            return DEAD_OBJECT;
        }
    }

    if (ssize_t(userSize) < 0 || (buffer == NULL && userSize != 0)) {
        // Sanity-check: user is most-likely passing an error code, and it would
        // make the return value ambiguous (actualSize vs error).
        ALOGE("AudioTrack::write(buffer=%p, size=%zu (%zd)", buffer, userSize, userSize);
        return BAD_VALUE;
    }

    size_t written = 0;
    Buffer audioBuffer;

    while (userSize >= mFrameSize) {
        // 单帧数据量 frameSize = channelCount * bytesPerSample
        //   对于双声道，16位采样的音频数据来说，frameSize = 2 * 2 = 4(bytes)
        // 用户传入的数据帧数 frameCount = userSize / frameSize
        audioBuffer.frameCount = userSize / mFrameSize;

        // obtainBuffer() 从 FIFO 上得到一块可用区间
        status_t err = obtainBuffer(&audioBuffer,
                blocking ? &ClientProxy::kForever : &ClientProxy::kNonBlocking);
        if (err < 0) {
            if (written > 0) {
                break;
            }
            if (err == TIMED_OUT || err == -EINTR) {
                err = WOULD_BLOCK;
            }
            return ssize_t(err);
        }

        // toWrite 是 FIFO 可用区间的大小，可能比 userSize（用户传入数据的大小）要小
        //   因此用户传入的数据可能要拆分多次拷贝到 FIFO 上
        // 注意：AudioTrack 和 AudioFlinger 是不同的进程，AudioFlinger 同时也在不停地
        //   消耗数据，所以 FIFO 可用区间是在不停变化的
        size_t toWrite = audioBuffer.size;
        memcpy(audioBuffer.i8, buffer, toWrite); // 把用户数据拷贝到 FIFO 可用区间
        buffer = ((const char *) buffer) + toWrite; // 未拷贝数据的位置
        userSize -= toWrite; // 未拷贝数据的大小
        written += toWrite; // 已拷贝数据的大小

        // releaseBuffer() 更新 FIFO 写位置
        // 对于 AudioFinger 来说，意味 FIFO 上有更多的可读数据
        releaseBuffer(&audioBuffer);
    }

    if (written > 0) {
        mFramesWritten += written / mFrameSize;
    }
    return written;
}
```
##### 3.3.2. AudioFlinger 读数据流程
AudioFlinger 消费数据的流程稍微复杂一点，3.4. AudioFlinger 回放录制线程 小节中描述了 AudioFlinger::PlaybackThread::threadLoop() 工作流程，这里不累述了，我们把焦点放在“如何从 FIFO 读取数据”节点上。

我们以 DirectOutputThread/OffloadThread 为例说明（MixerThread 读数据也是类似的过程，只不过是在 AudioMixer 中进行的）。

``` cpp
[->\frameworks\av\services\audioflinger\AudioFlinger.cpp]
void AudioFlinger::DirectOutputThread::threadLoop_mix()
{
    // mFrameCount 是硬件设备（PCM 设备）处理单个数据块的帧数（周期大小）
    //   上层必须积累了足够多（mFrameCount）的数据，才写入到 PCM 设备
    //   所以 mFrameCount 也就是 AudioFlinger 预期的数据量
    size_t frameCount = mFrameCount;
    // mSinkBuffer 目的缓冲区，threadLoop_write() 会把 mSinkBuffer 上的数据写到 PCM 设备
    int8_t *curBuf = (int8_t *)mSinkBuffer;
    // output audio to hardware
    // FIFO 上可读的数据量可能要比预期的要小，因此可能需要多次读取才能积累足够的数据量
    // 注意：AudioTrack 和 AudioFlinger 是不同的进程，AudioTrack 同时也在不停地生产数据
    //   所以 FIFO 可读的数据量是在不停变化的
    while (frameCount) {
        AudioBufferProvider::Buffer buffer;
        buffer.frameCount = frameCount;
        // getNextBuffer() 从 FIFO 上获取可读数据块
        status_t status = mActiveTrack->getNextBuffer(&buffer);
        if (status != NO_ERROR || buffer.raw == NULL) {
            memset(curBuf, 0, frameCount * mFrameSize);
            break;
        }
        // memcpy() 把 FIFO 可读数据拷贝到 mSinkBuffer 目的缓冲区
        memcpy(curBuf, buffer.raw, buffer.frameCount * mFrameSize);
        frameCount -= buffer.frameCount;
        curBuf += buffer.frameCount * mFrameSize;
        // releaseBuffer() 更新 FIFO 读位置
        // 对于 AudioTrack 来说，意味着 FIFO 上有更多的可用空间
        mActiveTrack->releaseBuffer(&buffer);
    }
    mCurrentWriteLength = curBuf - (int8_t *)mSinkBuffer;
    mSleepTimeUs = 0;
    mStandbyTimeNs = systemTime() + mStandbyDelayNs;
    mActiveTrack.clear();
}
```
##### 3.3.3. 环形 FIFO 管理
在上述过程中，不知大家有无意识到：整个过程中，最难的是如何协调生产者与消费者之间的步调。上文所说的 FIFO 是环形 FIFO，AudioTrack 写指针、AudioFlinger 读指针都是基于 FIFO 当前的读写位置来计算的。

☯AudioTrack 与 AudioFlinger 不在同一个进程上，怎么保证读写指针的线程安全
☯读写指针越过 FIFO 后，怎么处理
☯AudioTrack 写数据完成后，需要同步状态给 AudioFlinger，让 AudioFlinger 知道当前有可读数据了，而 AudioFlinger 读数据完成后，也需要同步状态给 AudioTrack，让 AudioTrack 知道当前有可用空间了；这里采取什么同步机制
我们回顾下创建 AudioTrack 对象时，FIFO 及其控制块的结构如下所示：

``` cpp
MODE_STREAM 模式下的匿名共享内存结构：
  |                                                         |
  | -------------------> mCblkMemory <--------------------- |
  |                                                         |
  +--------------------+------------------------------------+
  | audio_track_cblk_t |               FIFO                 |
  +--------------------+------------------------------------+
  ^                    ^
  |                    |
mCblk               mBuffer

mCblk = static_cast<audio_track_cblk_t *>(mCblkMemory->pointer());
new(mCblk) audio_track_cblk_t();
mBuffer = (char*)mCblk + sizeof(audio_track_cblk_t);
```

☯MODE_STATIC 模式下的匿名共享内存结构：

``` cpp
  +--------------------+    +-----------------------------------+
  | audio_track_cblk_t |    |         FIFO (sharedBuffer)       |
  +--------------------+    +-----------------------------------+
  ^                         ^
  |                         |
mCblk                    mBuffer

mCblk = (audio_track_cblk_t *) new uint8_t[size];
new(mCblk) audio_track_cblk_t();
mBuffer = sharedBuffer->pointer()
```

FIFO 管理相关的类图：
![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/audio.system/315-Audio-system-FIFO.jpg)

☯AudioTrackClientProxy：MODE_STREAM 模式下，生产者 AudioTrack 使用它在 FIFO 中找到可用空间的位置
☯AudioTrackServerProxy：MODE_STREAM 模式下，消费者 AudioFlinger::PlaybackThread 使用它在 FIFO 中找到可读数据的位置
☯StaticAudioTrackClientProxy：MODE_STATIC 模式下，生产者 AudioTrack 使用它在 FIFO 中找到可用空间的位置
☯StaticAudioTrackServerProxy：MODE_STATIC 模式下，消费者 AudioFlinger::PlaybackThread 使用它在 FIFO 中找到可读数据的位置
☯AudioRecordClientProxy：消费者 AudioRecord 使用它在 FIFO 中找到可读数据的位置
☯AudioTrackServerProxy：生产者 AudioFlinger::RecordThread 使用它在 FIFO 中找到可用空间的位置
到这里，我决定结束本文了。环形 FIFO 管理是 Android 音频系统的精髓，一个小节并不足以描述其原理及实现细节；Android 环形 FIFO 的实现可说得上精妙绝伦，其他项目如果要用到环形 FIFO，不妨多借鉴它。

#### (四)、深入剖析MediaPlayer播放音频流程
时序图：
![Alt text](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/audio.system/316-Audio-system-mediaplayer-playback.png)



#### （五）、参考资料(特别感谢各位前辈的分析和图示)：
[Android音频模块启动流程分析](https://zhuanlan.zhihu.com/p/27480328)
[Jhuster的专栏​ Android音频开发](https://zhuanlan.zhihu.com/jhuster)
[高通audio offload学习 | Thinking](http://thinks.me/2016/09/13/audio_qcom_offload/)
[DroidPhone的专栏 - CSDN博客](https://blog.csdn.net/droidphone/article/category/1118446)
[alsa音频架构1-CSDN博客](https://blog.csdn.net/orz415678659/article/details/8866071)
[alsa音频架构2-ASoc - CSDN博客](https://blog.csdn.net/orz415678659/article/details/8982771)
[alsa音频架构3-Pcm - CSDN博客](https://blog.csdn.net/orz415678659/article/details/8995163)
[alsa音频架构4-声卡控制 - CSDN博客](https://blog.csdn.net/orz415678659/article/details/8999364)
[Linux ALSA 音频系统：逻辑设备篇 - CSDN博客](https://blog.csdn.net/zyuanyun/article/details/59180272)
[Linux ALSA 音频系统：物理链路篇 - CSDN博客](https://blog.csdn.net/zyuanyun/article/details/59170418)
[专栏：MultiMedia框架总结(基于6.0源码) - CSDN博客](https://blog.csdn.net/column/details/12812.html)
[Android 音频系统：从 AudioTrack 到 AudioFlinger - CSDN博客](https://blog.csdn.net/zyuanyun/article/details/60890534)
[AZURE - CSDN博客 - ALSA-Android Audio](https://blog.csdn.net/azloong/article/category/778492)
[AZURE - CSDN博客 - ANDROID音频系统](https://blog.csdn.net/azloong/article/category/777239)
[Audio驱动总结--ALSA | Winddoing's Blog](https://winddoing.github.io/2017/07/10/audio_alsa/)
[audio HAL - 牧 天 - 博客园](http://www.cnblogs.com/muhe221/articles/4461543.html)
[林学森的Android专栏 - CSDN博客](https://blog.csdn.net/xuesen_lin/article/category/1390680)
[深入剖析Android音频 - CSDN博客Yangwen123](https://blog.csdn.net/yangwen123/article/category/2589087)
[播放框架 - 标签 - Tocy - 博客园](http://www.cnblogs.com/tocy/tag/%E6%92%AD%E6%94%BE%E6%A1%86%E6%9E%B6/)
[Android-7.0-Nuplayer概述 - CSDN博客](https://blog.csdn.net/miaomiao12345678/article/details/57415505)
[Android-7.0-Nuplayer-启动流程 - CSDN博客](https://blog.csdn.net/miaomiao12345678/article/details/56961370)
[Android Media Player 框架分析-Nuplayer（1） - CSDN博客](https://blog.csdn.net/harman_zjc/article/details/53386010)
[Android Media Player 框架分析-AHandler AMessage ALooper - CSDN博客](https://blog.csdn.net/harman_zjc/article/details/53397945)
[Android N Audio播放 start真面目- (六篇) CSDN博客](https://blog.csdn.net/liu1314you/article/category/6344583)
[深入理解Android音视频同步机制（五篇）NuPlayer的avsync逻辑 - CSDN博客](https://blog.csdn.net/nonmarking/article/details/78746671)
[wangyf的专栏 - CSDN博客-MT6737 Android N 平台 Audio系统学习](https://blog.csdn.net/u014310046/article/category/6571854)
[Android 7.0 Audio: Mediaplayer - CSDN博客](https://blog.csdn.net/xiashaohua/article/details/53638780)
[Android 7.0 Audio-相关类浅析- CSDN博客](https://blog.csdn.net/xiashaohua/article/category/6549668)
[Android N Audio播放六：如何读取buffer - CSDN博客](https://blog.csdn.net/liu1314you/article/details/59119144)
[Fuchsia OS中的RPC机制-FIDL - CSDN博客](https://blog.csdn.net/jinzhuojun/article/details/78007568)
[高通Audio中ASOC的codec驱动 - yooooooo - 博客园](http://www.cnblogs.com/linhaostudy/category/1145404.html)

