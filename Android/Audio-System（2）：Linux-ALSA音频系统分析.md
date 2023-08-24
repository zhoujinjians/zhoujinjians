---
title: Audio System（2）：Linux ALSA音频系统分析
cover: https://raw.githubusercontent.com/zhoujinjiankim/PicGo/master/hexo.themes/bing-wallpaper-2018.04.14.jpg
categories:
  - Audio
tags:
  - Android
  - Linux
  - Audio
toc: true
abbrlink: 20181008
date: 2018-10-08 09:25:00
password:
---

--------------------------------------------------------------------------------
注：文章都是通过阅读各位前辈总结的资料、Android 7.1.2 && Linux（kernel 3.18）Qualcomm平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除，禁止转载（©Qualcomm Technologies, Inc. 版权所有），谢谢。

[【特别感谢 - 雲和山的彼端 - 音频系统分析】](https://blog.csdn.net/zyuanyun)

Google Pixel、Pixel XL 内核代码（Kernel-3.18）：
 [Kernel source for Pixel and Pixel XL - GitHub](https://github.com/matthewdalex/marlin)

AOSP 源码（Android 7.1.2）：
 [ Android 系统全套源代码分享 (更新到 8.1.0_r1)](https://testerhome.com/topics/2229)

--------------------------------------------------------------------------------

源码（主要源码路径）：

> User space audio code 源码：

• **/hardware/qcom/audio/hal/ – (Audio HAL 源码)** 

• **/external/tinyalsa/ –  (tinymix, tinyplay, tinycap 源码)**

• **/vendor/qcom/proprietary/mm-audio/ – (QTI OMX  audio encoder and decoders 源码，未公开)**

• **/frameworks/av/media/audioserver/ – (Audioserver 源码)**

• **/frameworks/av/media/libstagefright/ – (Google Stagefright 多媒体框架源码)**

• **/frameworks/av/services/audioflinger/ – (Audioflinger 相关源码)**

• **/external/bluetooth/bluedroid/ – (A2DP audio HAL 相关源码)**/

• **/hardware/libhardware/modules/usbaudio/ – (USB HAL 源码)**/

• **/vendor/qcom/proprietary/wfd/mm/source/framework/src/ – (Wi-Fi Display (WFD)、 WFDMMSourceAudioSource.cpp，未公开)**

• **/system/core/include/system/ – (audio.h)**/

> Kernel space audio code 源码：

• **/kernel/sound/soc/msm/ –  msm8996.c machine driver 源码 **

• **/kernel/sound/soc/msm/qdsp6v2 –  platform drivers, front end (FE), and back-end (BE) DAI driver, Hexagon DSP drivers for AFE, ADM, and ASM, voice driver 相关源码**

• **kernel/sound/soc/soc-*.c – All the SoC-*.c  ALSA SoCs framework 源码**

• **kernel/drivers/slimbus/ –  SLIMbus driver 源码 **

• **kernel/arch/arm/mach-msm/ – 包含比如 acpuclock-8996.c, board-8996-gpiomux.c, board-8996.c, and clock-8996.c related to the GPIO, clock, and board-specific information on the MSM8996 相关源码**

• **/kernel/arch/arm/mach-msm/qdsp6v2/ – Contains the drivers for DSP-based encoders and decoders, code for the aDSP loader, APR driver, Ion memory driver, and other utility files**

• **/kernel/arch/arm/boot/dts – Contains MSM8996-*.its and MSM8996-*.Dtsi files that contain MSM8996-specific information; audio-related customization is available in files such as MSM8996.dtsi, msm8996-mtp.dtsi, and msm8996-cdp.dtsi**

• **/kernel/sound/soc/codecs/ – Contains the source code for the codec driver for WCD9335; codec driver-related source files are wcd9335.c, wcd9xxx-mbhc.c, wcd9xxx-resmgr.c, wcd9xxx-common.c, and so on.**

• **/kernel/drivers/mfd/ – Contains the source code for the codec driver; wcd9xxx-core.c, wcd9xxx-slimslave.c, and wcd9xxx-irq.c are the codec driverrelated files**


#### （一） Overview

硬件平台及软件版本：
☁ Kernel - 3.18
☁ SoC - Qualcomm snapdragon
☁ CODEC - WCD9335
☁ Machine - msm8996
☁ Userspace - tinyalsa

Linux ALSA 音频系统架构大致如下：
![Alt text](https://raw.githubusercontent.com/zhoujinjiankim/PicGo/master/audio.system/21-Audio-system-Android-Linux-ASoc-arc.png.png)

• Native ALSA Application：tinyplay/tinycap/tinymix，这些用户程序直接调用 alsa 用户库接口来实现放音、录音、控制
• ALSA Library API：alsa 用户库接口，常见有 tinyalsa、alsa-lib
• ALSA CORE：alsa 核心层，向上提供逻辑设备（PCM/CTL/MIDI/TIMER/…）系统调用，向下驱动硬件设备（Machine/I2S/DMA/CODEC）
• ASoC CORE：asoc 是建立在标准 alsa core 基础上，为了更好支持嵌入式系统和应用于移动设备的音频 codec 的一套软件体系
• Hardware Driver：音频硬件设备驱动，由三大部分组成，分别是 Machine、Platform、Codec

Platform：指某款 SoC 平台的音频模块，如 exynos、omap、qcom 等等。Platform 又可细分两部分：

  • cpu dai：在嵌入式系统里面通常指 SoC 的 I2S、PCM 总线控制器，负责把音频数据从 I2S tx FIFO 搬运到 CODEC（这是回放的情形，录制则方向相反）。cpu_dai 通过 **snd_soc_register_dai()** 来注册。注：DAI 是 Digital Audio Interface 的简称，分为 cpu_dai 和 codec_dai，这两者通过 I2S/PCM 总线连接；AIF 是 Audio Interface 的简称，嵌入式系统中一般是 I2S 和 PCM 接口。
  • pcm dma：负责把 dma buffer 中的音频数据搬运到 I2S tx FIFO。值得留意的是：某些情形下是不需要 dma 操作的，比如 Modem 和 CODEC 直连，因为 Modem 本身已经把数据送到 FIFO 了，这时只需启动 codec_dai 接收数据即可；该情形下，Machine 驱动 dai_link 中需要设定 .platform_name = "msm-pcm-xxx"。

Codec：对于回放来说，userspace 送过来的音频数据是经过采样量化的数字信号，在 codec 经过 DAC 转换成模拟信号然后输出到外放或耳机，这样我们就可以听到声音了。Codec 字面意思是编解码器，但芯片里面的功能部件很多，常见的有 AIF、DAC、ADC、Mixer、PGA、Line-in、Line-out，有些高端的 codec 芯片还有 EQ、DSP、SRC、DRC、AGC、Echo-Canceller、Noise-Suppression 等部件。

Machine：指某款机器，通过配置 dai_link 把 cpu_dai、codec_dai、modem_dai 各个音频接口给链结成一条条音频链路，然后注册 snd_soc_card。和上面两个不一样，Platform 和 CODEC 驱动一般是可以重用的，而 Machine 有它特定的硬件特性，几乎是不可重用的。所谓的硬件特性指：SoC Platform 与 Codec 的差异；DAIs 之间的链结方式；通过某个 GPIO 打开 Amplifier；通过某个 GPIO 检测耳机插拔；使用某个时钟如 MCLK/External-OSC 作为 I2S、CODEC 的时钟源等等。

从上面的描述来看，对于回放的情形，PCM 数据流向大致是：

``` cpp
        copy_from_user           DMA                 I2S           DAC
              ^                   ^                   ^             ^
+---------+   |    +----------+   |   +-----------+   |   +-----+   |   +------+
|userspace+-------->DMA Buffer+------->I2S TX FIFO+------->CODEC+------->SPK/HP|
+---------+        +----------+       +-----------+       +-----+       +------+
```

几个音频物理链路的概念：

dai_link：machine 驱动中定义的音频数据链路，它指定链路用到的 codec、codec_dai、cpu_dai、platform。比如对于 WCD9335 平台的 media 链路：.codec_dai_name = "snd-soc-dummy-dai", .codec_name = "snd-soc-dummy", .cpu_dai_name = "MultiMediaX", .platform_name = "msm-pcm-dsp.0"，这四者就构成了一条音频数据链路用于多媒体声音的回放和录制。一个系统可能有多个音频数据链路，比如 media 和 voice，因此可以定义多个 dai_link 。
代码如下：

``` cpp
[->/sound/soc/msm/msm8996.c]
/* Digital audio interface glue - connects codec <---> CPU */
static struct snd_soc_dai_link msm8996_common_dai_links[] = {
	/* FrontEnd DAI Links */
	{
		.name = "MSM8996 Media1",
		.stream_name = "MultiMedia1",
		.cpu_dai_name = "MultiMedia1",
		.platform_name = "msm-pcm-dsp.0",
		.dynamic = 1,
		.async_ops = ASYNC_DPCM_SND_SOC_PREPARE,
		.dpcm_playback = 1,
		.dpcm_capture = 1,
		.trigger = {SND_SOC_DPCM_TRIGGER_POST,
			SND_SOC_DPCM_TRIGGER_POST},
		.codec_dai_name = "snd-soc-dummy-dai",
		.codec_name = "snd-soc-dummy",
		.ignore_suspend = 1,
		/* this dainlink has playback support */
		.ignore_pmdown_time = 1,
		.be_id = MSM_FRONTEND_DAI_MULTIMEDIA1
	},
	......
}
```
![Alt text](https://raw.githubusercontent.com/zhoujinjiankim/PicGo/master/audio.system/22-MSM8996-Linux-Android-Audio-ASoc-Architectre.png)

高通平台因DSP而存在特殊性，如上图，Frontend 链接 "Platform"，经由 "Platform"->Backend链接到Codec。
Front-end DAI：

``` cpp
[->/sound/soc/msm/msm-dai-fe.c]
static struct snd_soc_dai_driver msm_fe_dais[] = {
	{
		.playback = {
			.stream_name = "MultiMedia1 Playback",
			.aif_name = "MM_DL1",
			.rates = (SNDRV_PCM_RATE_8000_192000|
					SNDRV_PCM_RATE_KNOT),
			.formats = (SNDRV_PCM_FMTBIT_S16_LE |
						SNDRV_PCM_FMTBIT_S24_LE |
						SNDRV_PCM_FMTBIT_S24_3LE),
			.channels_min = 1,
			.channels_max = 8,
			.rate_min =     8000,
			.rate_max =	192000,
		},
		.capture = {
			.stream_name = "MultiMedia1 Capture",
			.aif_name = "MM_UL1",
			.rates = (SNDRV_PCM_RATE_8000_192000|
					SNDRV_PCM_RATE_KNOT),
			.formats = (SNDRV_PCM_FMTBIT_S16_LE |
				    SNDRV_PCM_FMTBIT_S24_LE|
				    SNDRV_PCM_FMTBIT_S24_3LE),
			.channels_min = 1,
			.channels_max = 4,
			.rate_min =     8000,
			.rate_max =	48000,
		},
		.ops = &msm_fe_Multimedia_dai_ops,
		.name = "MultiMedia1",
		.probe = fe_dai_probe,
	},
	......
}
```
Back-end DAI：

``` cpp
[->sound/soc/msm/msm8996.c]
static struct snd_soc_dai_link msm8996_tasha_be_dai_links[] = {
	/* Backend DAI Links */
	{
		.name = LPASS_BE_SLIMBUS_0_RX,
		.stream_name = "Slimbus Playback",
		.cpu_dai_name = "msm-dai-q6-dev.16384",
		.platform_name = "msm-pcm-routing",
		.codec_name = "tasha_codec",
		.codec_dai_name = "tasha_mix_rx1",
		.no_pcm = 1,
		.dpcm_playback = 1,
		.be_id = MSM_BACKEND_DAI_SLIMBUS_0_RX,
		.init = &msm_audrx_init,
		.be_hw_params_fixup = msm_slim_0_rx_be_hw_params_fixup,
		/* this dainlink has playback support */
		.ignore_pmdown_time = 1,
		.ignore_suspend = 1,
		.ops = &msm8996_be_ops,
	},
	......
}
static struct snd_soc_dai_link msm8996_tasha_fe_dai_links[] = {
	{
		.name = LPASS_BE_SLIMBUS_4_TX,
		.stream_name = "Slimbus4 Capture",
		.cpu_dai_name = "msm-dai-q6-dev.16393",
		.platform_name = "msm-pcm-hostless",
		.codec_name = "tasha_codec",
		.codec_dai_name = "tasha_vifeedback",
		.be_id = MSM_BACKEND_DAI_SLIMBUS_4_TX,
		.be_hw_params_fixup = msm_slim_4_tx_be_hw_params_fixup,
		.ops = &msm8996_be_ops,
		.no_host_mode = SND_SOC_DAI_LINK_NO_HOST,
		.ignore_suspend = 1,
	},
	......
}
```

hw constraints：指平台本身的硬件限制，如所能支持的通道数/采样率/数据格式、DMA 支持的数据周期大小（period size）、周期次数（period count）等，通过 snd_pcm_hardware 结构体描述：
``` cpp
[->sound/soc/msm/qdsp6v2/msm-pcm-q6-v2.c]
static struct snd_pcm_hardware msm_pcm_hardware_capture = {
	.info =                 (SNDRV_PCM_INFO_MMAP |
				SNDRV_PCM_INFO_BLOCK_TRANSFER |
				SNDRV_PCM_INFO_MMAP_VALID |
				SNDRV_PCM_INFO_INTERLEAVED |
				SNDRV_PCM_INFO_PAUSE | SNDRV_PCM_INFO_RESUME),
	.formats =              (SNDRV_PCM_FMTBIT_S16_LE |
				SNDRV_PCM_FMTBIT_S24_LE |
				SNDRV_PCM_FMTBIT_S24_3LE),
	.rates =                SNDRV_PCM_RATE_8000_48000,
	.rate_min =             8000,
	.rate_max =             48000,
	.channels_min =         1,
	.channels_max =         4,
	.buffer_bytes_max =     CAPTURE_MAX_NUM_PERIODS *
				CAPTURE_MAX_PERIOD_SIZE,
	.period_bytes_min =	CAPTURE_MIN_PERIOD_SIZE,
	.period_bytes_max =     CAPTURE_MAX_PERIOD_SIZE,
	.periods_min =          CAPTURE_MIN_NUM_PERIODS,
	.periods_max =          CAPTURE_MAX_NUM_PERIODS,
	.fifo_size =            0,
};

```
hw params：用户层设置的硬件参数，如 channels、sample rate、pcm format、period size、period count；这些参数受 hw constraints 约束。

sw params：用户层设置的软件参数，如 start threshold、stop threshold、silence threshold。
#### （二）ASoC Core
ASoC：ALSA System on Chip，是建立在标准 ALSA 驱动之上，为了更好支持嵌入式系统和应用于移动设备的音频 codec 的一套软件体系，它依赖于标准 ALSA 驱动框架。内核文档 Documentation/alsa/soc/overview.txt 中详细介绍了 ASoC 的设计初衷，这里不一一引用，简单陈述如下：

• 独立的 codec 驱动，标准的 ALSA 驱动框架里面 codec 驱动往往与 SoC/CPU 耦合过于紧密，不利于在多样化的平台/机器上移植复用
• 方便 codec 与 SoC 通过 PCM/I2S 总线建立链接
• 动态音频电源管理 DAPM，使得 codec 任何时候都工作在最低功耗状态，同时负责音频路由的创建
• POPs 和 click 音抑制弱化处理，在 ASoC 中通过正确的音频部件上下电次序来实现
• Machine 驱动的特定控制，比如耳机、麦克风的插拔检测，外放功放的开关
在概述中已经介绍了 ASoC 硬件设备驱动的三大构成：Codec、Platform 和 Machine，下面列举各驱动的功能构成：

ASoC Codec Driver：

• Codec DAI 和 PCM 的配置信息
• Codec 的控制接口，如 I2C/SPI
• Mixer 和其他音频控件
• Codec 的音频接口函数，见 snd_soc_dai_ops 结构体定义
• DAPM 描述信息
• DAPM 事件处理句柄
• DAC 数字静音控制

ASoC Platform Driver： 包括 dma 和 cpu_dai 两部分：

• dma 驱动实现音频 dma 操作，具体见 snd_pcm_ops 结构体定义
• cpu_dai 驱动实现音频数字接口控制器的描述和配置
• ASoC Machine Driver：

作为链结 Platform 和 Codec 的载体，它必须配置 dai_link 为音频数据链路指定 Platform 和 Codec
处理机器特有的音频控件和音频事件，例如回放时打开外放功放
硬件设备驱动相关结构体：

• snd_soc_codec_driver：音频编解码芯片描述及操作函数，如控件/微件/音频路由的描述信息、时钟配置、IO 控制等
• snd_soc_dai_driver：音频数据接口描述及操作函数，根据 codec 端和 soc 端，分为 codec_dai 和 cpu_dai
• snd_soc_platform_driver：音频 dma 设备描述及操作函数
• snd_soc_dai_link：音频链路描述及板级操作函数

#### （三）Codec Driver
基本是以内核文档 Documentation/sound/alsa/soc/codec.txt 中的内容为脉络来分析的。Codec 的作用，之前已有描述，本章主要罗列下 Codec driver 中重要的数据结构及注册流程。
其中有着各种功能部件，包括但不限于 ：

>ADC	把麦克风拾取的模拟信号转换成数字信号
DAC	把音频接口过来的数字信号转换成模拟信号
MIXER	混音器，把多路输入信号混合成单路输出

##### 3.1. Codec DAI and PCM configuration
codec_dai 和 pcm 配置信息通过结构体 snd_soc_dai_driver 描述，包括 dai 的能力描述和操作接口，snd_soc_dai_driver 最终会被注册到 soc-core 中。

``` cpp
[->include/sound/soc-dai.h]
/*
 * Digital Audio Interface Driver.
 *
 * Describes the Digital Audio Interface in terms of its ALSA, DAI and AC97
 * operations and capabilities. Codec and platform drivers will register this
 * structure for every DAI they have.
 * This structure covers the clocking, formating and ALSA operations for each
 * interface.
 */
struct snd_soc_dai_driver {
    /* DAI description */
    const char *name;
    unsigned int id;
    int ac97_control;

    /* DAI driver callbacks */
    int (*probe)(struct snd_soc_dai *dai);
    int (*remove)(struct snd_soc_dai *dai);
    int (*suspend)(struct snd_soc_dai *dai);
    int (*resume)(struct snd_soc_dai *dai);

    /* ops */
    const struct snd_soc_dai_ops *ops;

    /* DAI capabilities */
    struct snd_soc_pcm_stream capture;
    struct snd_soc_pcm_stream playback;
    unsigned int symmetric_rates:1;

    /* probe ordering - for components with runtime dependencies */
    int probe_order;
    int remove_order;
};
```
name：codec_dai 的名称标识，dai_link 通过配置 codec_dai_name 来找到对应的 codec_dai；
probe：codec_dai 的初始化函数，注册声卡时回调；
playback：回放能力描述，如回放设备所支持的声道数、采样率、音频格式；
capture：录制能力描述，如录制设备所支持声道数、采样率、音频格式；
ops：codec_dai 的操作函数集，这些函数集非常重要，用于 dai 的时钟配置、格式配置、硬件参数配置。

codec_dai：
``` cpp
[->sound/soc/codecs/wcd9335.c]
static struct snd_soc_dai_driver tasha_i2s_dai[] = {
	{
		.name = "tasha_i2s_rx1",
		.id = AIF1_PB,
		.playback = {
			.stream_name = "AIF1 Playback",
			.rates = WCD9335_RATES_MASK,
			.formats = TASHA_FORMATS_S16_S24_LE,
			.rate_max = 192000,
			.rate_min = 8000,
			.channels_min = 1,
			.channels_max = 2,
		},
		.ops = &tasha_dai_ops,
	},
	{
		.name = "tasha_i2s_tx1",
		.id = AIF1_CAP,
		.capture = {
			.stream_name = "AIF1 Capture",
			.rates = WCD9335_RATES_MASK,
			.formats = TASHA_FORMATS,
			.rate_max = 192000,
			.rate_min = 8000,
			.channels_min = 1,
			.channels_max = 4,
		},
		.ops = &tasha_dai_ops,
	},
	......
}
```
##### 3.2. Codec control IO
移动设备的音频 Codec，其控制接口一般是 I2C 或 SPI，控制接口用于读写 codec 的寄存器。在 snd_soc_codec_driver 结构体中，有如下字段描述 Codec 的控制接口：

``` cpp
[->include/sound/soc.h]
/* codec driver */
struct snd_soc_codec_driver {

	......

	/* codec IO */
	struct regmap *(*get_regmap)(struct device *);
	unsigned int (*read)(struct snd_soc_codec *, unsigned int);
	int (*write)(struct snd_soc_codec *, unsigned int, unsigned int);
	int (*display_register)(struct snd_soc_codec *, char *,
				size_t, unsigned int);
	int (*volatile_register)(struct snd_soc_codec *, unsigned int);
	int (*readable_register)(struct snd_soc_codec *, unsigned int);
	int (*writable_register)(struct snd_soc_codec *, unsigned int);
	unsigned int reg_cache_size;
	short reg_cache_step;
	short reg_word_size;
	const void *reg_cache_default;

	......
};
```
• read：读寄存器；
• write：写寄存器；
• volatile_register：判断指定的寄存器是否是 volatile 属性；假如是，则读取寄存器时不是读 cache，而直接访问硬件；
• readable_register：判断指定的寄存器是否可读；
• reg_cache_default：寄存器的缺省值；
• reg_cache_size：缺省的寄存器值数组大小；
• reg_word_size：寄存器宽度。
在 Linux-3.4.5 中，很多 codec 的控制接口都改用 regmap 了。soc-core 中判断是否用的是 regmap，如果是，则调用 regmap 接口。
##### 3.3. Mixers and audio controls
音频控件多用于部件开关和音量的设定，音频控件可通过 soc.h 中的宏来定义，例如单一型控件：

``` cpp
[->include/sound/soc.h]
#define SOC_SINGLE(xname, reg, shift, max, invert) \
{   .iface = SNDRV_CTL_ELEM_IFACE_MIXER, .name = xname, \
    .info = snd_soc_info_volsw, .get = snd_soc_get_volsw,\
    .put = snd_soc_put_volsw, \
    .private_value =  SOC_SINGLE_VALUE(reg, shift, max, invert) }
```
这种控件只有一个设置量，一般用于部件开关。宏定义的参数说明：

• xname：控件的名称标识；
• reg：控件对应的寄存器地址；
• shift：控件控制位在寄存器中的偏移；
• max：控件设置值范围；
• invert：设定值是否取反。
其他类型控件类似，不一一介绍了。

上述只是宏定义，音频控件真正的结构是 snd_kcontrol_new：

``` cpp
[->/include/sound/control.h]
struct snd_kcontrol_new {
    snd_ctl_elem_iface_t iface; /* interface identifier */
    unsigned int device;        /* device/client number */
    unsigned int subdevice;     /* subdevice (substream) number */
    const unsigned char *name;  /* ASCII name of item */
    unsigned int index;     /* index of item */
    unsigned int access;        /* access rights */
    unsigned int count;     /* count of same elements */
    snd_kcontrol_info_t *info;
    snd_kcontrol_get_t *get;
    snd_kcontrol_put_t *put;
    union {
        snd_kcontrol_tlv_rw_t *c;
        const unsigned int *p;
    } tlv;
    unsigned long private_value;
};
```

Codec 初始化时，通过 snd_soc_add_codec_controls() 把所有定义好的音频控件注册到 alsa-core ，上层可以通过 tinymix、alsa_amixer 等工具查看修改这些控件的设定。

##### 3.4. Codec audio operations
Codec 音频操作接口通过结构体 snd_soc_dai_ops 描述：

``` cpp
[->include/sound/soc-dai.h]
struct snd_soc_dai_ops {
	/*
	 * DAI clocking configuration, all optional.
	 * Called by soc_card drivers, normally in their hw_params.
	 */
	int (*set_sysclk)(struct snd_soc_dai *dai,
		int clk_id, unsigned int freq, int dir);
	int (*set_pll)(struct snd_soc_dai *dai, int pll_id, int source,
		unsigned int freq_in, unsigned int freq_out);
	int (*set_clkdiv)(struct snd_soc_dai *dai, int div_id, int div);
	int (*set_bclk_ratio)(struct snd_soc_dai *dai, unsigned int ratio);

	/*
	 * DAI format configuration
	 * Called by soc_card drivers, normally in their hw_params.
	 */
	int (*set_fmt)(struct snd_soc_dai *dai, unsigned int fmt);
	int (*xlate_tdm_slot_mask)(unsigned int slots,
		unsigned int *tx_mask, unsigned int *rx_mask);
	int (*set_tdm_slot)(struct snd_soc_dai *dai,
		unsigned int tx_mask, unsigned int rx_mask,
		int slots, int slot_width);
	int (*set_channel_map)(struct snd_soc_dai *dai,
		unsigned int tx_num, unsigned int *tx_slot,
		unsigned int rx_num, unsigned int *rx_slot);
	int (*set_tristate)(struct snd_soc_dai *dai, int tristate);
	int (*get_channel_map)(struct snd_soc_dai *dai,
		unsigned int *tx_num, unsigned int *tx_slot,
		unsigned int *rx_num, unsigned int *rx_slot);

	......
};

```
注释比较详细的了，Codec 音频操作接口分为 5 大部分：时钟配置、格式配置、数字静音、PCM 音频接口、FIFO 延迟。着重说下时钟配置及格式配置接口：

• set_sysclk：codec_dai 系统时钟设置，当上层打开 pcm 设备时，需要回调该接口设置 Codec 的系统时钟，Codec 才能正常工作；
• set_pll：Codec FLL 设置，Codec 一般接了一个 MCLK 输入时钟，回调该接口基于 MCLK 来产生 Codec FLL 时钟，接着 codec_dai 的 sysclk、bclk、lrclk 均可从 FLL 分频出来（假设 Codec 作为 master）；
• set_fmt：codec_dai 格式设置，具体见 soc-dai.h； 
  • SND_SOC_DAIFMT_I2S：音频数据是 I2S 格式，常用于多媒体音频；
  • SND_SOC_DAIFMT_DSP_A：音频数据是 PCM 格式，常用于通话语音；
  • SND_SOC_DAIFMT_CBM_CFM：Codec 作为 master，BCLK 和 LRCLK 由 Codec 提供；
  • SND_SOC_DAIFMT_CBS_CFS：Codec 作为 slave，BCLK 和 LRCLK 由 SoC/CPU 提供；
• hw_params：codec_dai 硬件参数设置，根据上层设定的声道数、采样率、数据格式，来配置 codec_dai 相关寄存器。

WCD9335的snd_soc_dai_ops ：
``` cpp
[->/sound/soc/codecs/wcd9335.c]
static struct snd_soc_dai_ops tasha_dai_ops = {
	.startup = tasha_startup,
	.shutdown = tasha_shutdown,
	.hw_params = tasha_hw_params,
	.prepare = tasha_prepare,
	.set_sysclk = tasha_set_dai_sysclk,
	.set_fmt = tasha_set_dai_fmt,
	.set_channel_map = tasha_set_channel_map,
	.get_channel_map = tasha_get_channel_map,
};
```
##### 3.5. Codec register
当 platform_driver：

``` cpp
[->/sound/soc/codecs/wcd9335.c]
static struct platform_driver tasha_codec_driver = {
	.probe = tasha_probe,
	.remove = tasha_remove,
	.driver = {
		.name = "tasha_codec",
		.owner = THIS_MODULE,
#ifdef CONFIG_PM
		.pm = &tasha_pm_ops,
#endif
	},
};

```
与.name = "tasha_codec"  的 platform_device（该 platform_device 在 drivers/mfd/wcd9xxx-core.c 中注册wcd9xxx_device_init->wcd9xxx_check_codec_type->tasha_devs）匹配后，

``` cpp
[->drivers/mfd/wcd9xxx-core.c]
static struct mfd_cell tasha_devs[] = {
	{
		.name = "tasha_codec",
	},
};
```

立即回调 tasha_probe() 注册 Codec：

``` cpp
[->/sound/soc/codecs/wcd9335.c]
static int tasha_probe(struct platform_device *pdev)
{
	int ret = 0;
	struct tasha_priv *tasha;
	struct clk *wcd_ext_clk, *wcd_native_clk;
	struct wcd9xxx_resmgr_v2 *resmgr;
	struct wcd9xxx_power_region *cdc_pwr;
	......
	tasha = devm_kzalloc(&pdev->dev, sizeof(struct tasha_priv),
			    GFP_KERNEL);
	......
    tasha->resmgr = resmgr;
	tasha->swr_plat_data.handle = (void *) tasha;
	tasha->swr_plat_data.read = tasha_swrm_read;
	tasha->swr_plat_data.write = tasha_swrm_write;
	tasha->swr_plat_data.bulk_write = tasha_swrm_bulk_write;
	tasha->swr_plat_data.clk = tasha_swrm_clock;
	tasha->swr_plat_data.handle_irq = tasha_swrm_handle_irq;

	/* Register for Clock */
	wcd_ext_clk = clk_get(tasha->wcd9xxx->dev, "wcd_clk");
	if (IS_ERR(wcd_ext_clk)) {
		dev_err(tasha->wcd9xxx->dev, "%s: clk get %s failed\n",
			__func__, "wcd_ext_clk");
		goto err_clk;
	}
	tasha->wcd_ext_clk = wcd_ext_clk;
	tasha->sido_voltage = SIDO_VOLTAGE_NOMINAL_MV;
	set_bit(AUDIO_NOMINAL, &tasha->status_mask);
	tasha->sido_ccl_cnt = 0;
	......
	if (wcd9xxx_get_intf_type() == WCD9XXX_INTERFACE_TYPE_SLIMBUS)
		ret = snd_soc_register_codec(&pdev->dev, &soc_codec_dev_tasha,
					     tasha_dai, ARRAY_SIZE(tasha_dai));
	else if (wcd9xxx_get_intf_type() == WCD9XXX_INTERFACE_TYPE_I2C)
		ret = snd_soc_register_codec(&pdev->dev, &soc_codec_dev_tasha,
					     tasha_i2s_dai,
					     ARRAY_SIZE(tasha_i2s_dai));
	else
		ret = -EINVAL;
	......
}
```
snd_soc_register_codec：将 codec_driver 和 codec_dai_driver 注册到 soc-core。

``` cpp
[->]
/**
 * snd_soc_register_codec - Register a codec with the ASoC core
 *
 * @codec: codec to register
 */
int snd_soc_register_codec(struct device *dev,
               const struct snd_soc_codec_driver *codec_drv,
               struct snd_soc_dai_driver *dai_drv,
               int num_dai)
```
创建一个 snd_soc_codec 实例，包含 codec_drv（snd_soc_dai_driver）相关信息，封装给 soc-core 使用，相关代码段如下：

``` cpp
[sound/soc/soc-core.c: snd_soc_register_codec]
    struct snd_soc_codec *codec;

    dev_dbg(dev, "codec register %s\n", dev_name(dev));

    codec = kzalloc(sizeof(struct snd_soc_codec), GFP_KERNEL);
    if (codec == NULL)
        return -ENOMEM;

    /* create CODEC component name */
    codec->name = fmt_single_name(dev, &codec->id);
    if (codec->name == NULL) {
        kfree(codec);
        return -ENOMEM;
    }

    // 初始化 Codec 的寄存器缓存配置及读写接口
    codec->write = codec_drv->write;
    codec->read = codec_drv->read;
    codec->volatile_register = codec_drv->volatile_register;
    codec->readable_register = codec_drv->readable_register;
    codec->writable_register = codec_drv->writable_register;
    codec->ignore_pmdown_time = codec_drv->ignore_pmdown_time;
    codec->dapm.bias_level = SND_SOC_BIAS_OFF;
    codec->dapm.dev = dev;
    codec->dapm.codec = codec;
    codec->dapm.seq_notifier = codec_drv->seq_notifier;
    codec->dapm.stream_event = codec_drv->stream_event;
    codec->dev = dev;
    codec->driver = codec_drv;
    codec->num_dai = num_dai;
    mutex_init(&codec->mutex);
```
把以上 codec 实例插入到 codec_list链表中（声卡注册时会遍历该链表，找到 dai_link 声明的 codec 并绑定）：

``` cpp
[sound/soc/soc-core.c: snd_soc_register_codec]
list_add(&codec->list, &codec_list);
```
把 codec_drv 中的 snd_soc_dai_driver（tasha_dai 或者tasha_i2s_dai ）注册到 soc-core：

``` cpp
[sound/soc/soc-core.c: snd_soc_register_codec]
snd_soc_register_dais(&codec->component, dai_drv, num_dai, false);
```
snd_soc_register_dais() 会把 dai 插入到 dai_list 链表中（声卡注册时会遍历该链表，找到 dai_link 声明的 codec_dai 并绑定）：

``` cpp
[sound/soc/soc-core.c: snd_soc_register_codec]
list_add(&dai->list, &dai_list);
```
最后顺便提下 codec 和 codec_dai 的区别：codec 指音频芯片共有的部分，包括 codec 初始化函数、控制接口、寄存器缓存、控件、dapm 部件、音频路由、偏置电压设置函数等描述信息；而 codec_dai 指 codec 上的音频接口驱动描述，包括时钟配置、格式配置、能力描述等等，各个接口的描述信息不一定都是一致的，所以每个音频接口都有着各自的驱动描述。
#### （四）Platform Driver
概述中提到音频 Platform 驱动主要用于音频数据传输，这里又细分为两步：

启动 dma 设备，把音频数据从 dma buffer 搬运到 cpu_dai FIFO，这部分驱动用 snd_soc_platform_driver 描述，后面分析用 pcm_dma 指代它。
启动数字音频接口控制器（I2S/PCM/AC97），把音频数据从 cpu_dai FIFO 传送到 codec_dai（高通平台会将数据传送到ADSP）这部分驱动用 snd_soc_dai_driver 描述，后面分析用 cpu_dai 指代它。

> MSM8996 包含三个 Hexagon DSP ：application, modem, and sensor。 
> Application  DSP：不仅可以处理语音和音频，还可以处理计算机 视觉、视频、图像和Camera。
>  Sensor DSP：也叫做SLPI，所有的sensor都链接到SLPI上面，它管理所有的Sensor及相关算法。

对于 cpu_dai 驱动，从上面的类图我们可知，主要工作有：

实现 dai 操作函数，见 snd_soc_dai_ops 定义，用于配置和操作音频数字接口控制器，如时钟配置 set_sysclk()、格式配置 set_fmt()、硬件参数配置 hw_params()、启动/停止数据传输 trigger() 等；
实现 probe 函数（初始化）、remove 函数（卸载）、suspend/resume 函数（电源管理）；
初始化 snd_soc_dai_driver 实例，包括回放和录制的能力描述、dai 操作函数集、probe/remove 回调、电源管理相关的 suspend/resume 回调；
通过 snd_soc_register_dai() 把初始化完成的 snd_soc_dai_driver 注册到 soc-core：首先创建一个 snd_soc_dai 实例，然后把该 snd_soc_dai 实例插入到 dai_list 链表（声卡注册时会遍历该链表，找到 dai_link 声明的 cpu_dai 并绑定）。

``` cpp
[sound/soc/soc-core.c]
static int snd_soc_register_dais(struct snd_soc_component *component,
	struct snd_soc_dai_driver *dai_drv, size_t count,
	bool legacy_dai_naming)
{
	struct device *dev = component->dev;
	struct snd_soc_dai *dai;
	unsigned int i;
	int ret;

	dev_dbg(dev, "ASoC: dai register %s #%Zu\n", dev_name(dev), count);

	component->dai_drv = dai_drv;
	component->num_dai = count;

	for (i = 0; i < count; i++) {

		dai = kzalloc(sizeof(struct snd_soc_dai), GFP_KERNEL);
		......
		if (count == 1 && legacy_dai_naming) {
			dai->name = fmt_single_name(dev, &dai->id);
		} else {
			dai->name = fmt_multiple_name(dev, &dai_drv[i]);
			if (dai_drv[i].id)
				dai->id = dai_drv[i].id;
			else
				dai->id = i;
		}
		......
		dai->component = component;
		dai->dev = dev;
		dai->driver = &dai_drv[i];
		if (!dai->driver->ops)
			dai->driver->ops = &null_dai_ops;

		list_add(&dai->list, &component->dai_list);
	}

	return 0;

	return ret;
}

```
dai 操作函数的实现是 cpu_dai 驱动的主体，需要配置好相关寄存器让 I2S/PCM 总线控制器正常运转，snd_soc_dai_ops 字段的详细说明见 3.6. Codec audio operations 章节。

cpu_dai 驱动应该算是这个系列中最简单的一环，因此不多花费笔墨在这里了。倒是某些平台上，dma 设备信息（总线地址、通道号、传输单元大小）是在这里初始化的，这点要留意，这些 dma 设备信息在 pcm_dma 驱动中用到。

##### 4.1. pcm operations
操作函数的实现是本模块的主体，见 snd_pcm_ops 结构体描述：

``` cpp
[->include/sound/pcm.h]
struct snd_pcm_ops {
	int (*open)(struct snd_pcm_substream *substream);
	int (*close)(struct snd_pcm_substream *substream);
	int (*ioctl)(struct snd_pcm_substream * substream,
		     unsigned int cmd, void *arg);
	int (*compat_ioctl)(struct snd_pcm_substream *substream,
		     unsigned int cmd, void *arg);
	int (*hw_params)(struct snd_pcm_substream *substream,
			 struct snd_pcm_hw_params *params);
	int (*hw_free)(struct snd_pcm_substream *substream);
	int (*prepare)(struct snd_pcm_substream *substream);
	int (*trigger)(struct snd_pcm_substream *substream, int cmd);
	snd_pcm_uframes_t (*pointer)(struct snd_pcm_substream *substream);
	int (*delay_blk)(struct snd_pcm_substream *substream);
	int (*wall_clock)(struct snd_pcm_substream *substream,
			  struct timespec *audio_ts);
	int (*copy)(struct snd_pcm_substream *substream, int channel,
		    snd_pcm_uframes_t pos,
		    void __user *buf, snd_pcm_uframes_t count);
	int (*silence)(struct snd_pcm_substream *substream, int channel,
		       snd_pcm_uframes_t pos, snd_pcm_uframes_t count);
	struct page *(*page)(struct snd_pcm_substream *substream,
			     unsigned long offset);
	int (*mmap)(struct snd_pcm_substream *substream, struct vm_area_struct *vma);
	int (*ack)(struct snd_pcm_substream *substream);
	int (*restart)(struct snd_pcm_substream *substream);
};
```
##### 4.1. platform_driver 注册
当 platform_driver：

``` cpp
[->sound/soc/msm/qdsp6v2/msm-pcm-q6-v2.c]
static struct platform_driver msm_pcm_driver = {
	.driver = {
		.name = "msm-pcm-dsp",
		.owner = THIS_MODULE,
		.of_match_table = msm_pcm_dt_match,
	},
	.probe = msm_pcm_probe,
	.remove = msm_pcm_remove,
};
```
与 .name = "msm-pcm-dsp" 的 platform_device 注册 匹配后，系统会回调 msm_pcm_probe() 注册 platform：
``` cpp
[->sound/soc/msm/qdsp6v2/msm-pcm-q6-v2.c]
static int msm_pcm_probe(struct platform_device *pdev)
{
	int rc;
	int id;
	struct msm_plat_data *pdata;
	const char *latency_level;

	rc = of_property_read_u32(pdev->dev.of_node,
				"qcom,msm-pcm-dsp-id", &id);
	......
	pdata = kzalloc(sizeof(struct msm_plat_data), GFP_KERNEL);
	......

	if (of_property_read_bool(pdev->dev.of_node,
				"qcom,msm-pcm-low-latency")) {
		pdata->perf_mode = LOW_LATENCY_PCM_MODE;
		rc = of_property_read_string(pdev->dev.of_node,
			"qcom,latency-level", &latency_level);
		if (!rc) {
			if (!strcmp(latency_level, "ultra"))
				pdata->perf_mode = ULTRA_LOW_LATENCY_PCM_MODE;
			else if (!strcmp(latency_level, "ull-pp"))
				pdata->perf_mode =
					ULL_POST_PROCESSING_PCM_MODE;
		}
	}
	else
		pdata->perf_mode = LEGACY_PCM_MODE;

	dev_set_drvdata(&pdev->dev, pdata);

	return snd_soc_register_platform(&pdev->dev,
				   &msm_soc_platform);
}
```
snd_soc_register_platform：将 platform_drv 注册到 soc-core。
创建一个 snd_soc_platform 实例，包含 platform_drv（snd_soc_platform_driver）的相关信息，封装给 soc-core 使用；
把以上创建的 platform 实例插入到 platform_list 链表上（声卡注册时会遍历该链表，找到 dai_link 声明的 platform 并绑定）。
代码实现：

``` cpp
int snd_soc_register_platform(struct device *dev,
		const struct snd_soc_platform_driver *platform_drv)
{
	struct snd_soc_platform *platform;
	int ret;

	platform = kzalloc(sizeof(struct snd_soc_platform), GFP_KERNEL);

	ret = snd_soc_add_platform(dev, platform, platform_drv);

	return ret;
}
```
至此，完成了 Platform 驱动的实现。回放情形下，pcm_dma 设备负责把 dma buffer 中的数据搬运到 I2S tx FIFO，I2S 总线控制器负责把 I2S tx FIFO 中的数据传送DSP，DSP经处理后传送到到 Codec。
#### （五）  Machine Driver
章节 3. Codec 和 4. Platform 介绍了 Codec、Platform 驱动，但仅有 Codec、Platform 驱动是不能工作的，需要一个角色把 codec、codec_dai、cpu_dai、platform 给链结起来才能构成一个完整的音频链路，这个角色就由 machine_drv 承担了。

snd_soc_dai_link 结构体：

``` cpp
[->/include/sound/soc.h]
struct snd_soc_dai_link {
	const char *name;			/* Codec name */
	const char *stream_name;		/* Stream name */
	const char *cpu_name;
	struct device_node *cpu_of_node;
	const char *cpu_dai_name;
	const char *codec_name;
	struct device_node *codec_of_node;
	const char *codec_dai_name;
	struct snd_soc_dai_link_component *codecs;
	unsigned int num_codecs;
	const char *platform_name;
	struct device_node *platform_of_node;
	int be_id;	/* optional ID for machine driver BE identification */
	const struct snd_soc_pcm_stream *params;
	unsigned int dai_fmt;           /* format to set on init */
	enum snd_soc_dpcm_trigger trigger[2]; /* trigger type for DPCM */
	unsigned int ignore_suspend:1;
	unsigned int symmetric_rates:1;
	unsigned int symmetric_channels:1;
	unsigned int symmetric_samplebits:1;
	unsigned int no_pcm:1;
	unsigned int dynamic:1;
	unsigned int no_host_mode:2;
	unsigned int dpcm_capture:1;
	unsigned int dpcm_playback:1;
	unsigned int ignore_pmdown_time:1;
	int (*init)(struct snd_soc_pcm_runtime *rtd);
	int (*be_hw_params_fixup)(struct snd_soc_pcm_runtime *rtd,
			struct snd_pcm_hw_params *params);
	const struct snd_soc_ops *ops;
	const struct snd_soc_compr_ops *compr_ops;
	bool playback_only;
	bool capture_only;
	enum snd_soc_async_ops async_ops;
}
```
重点介绍如下几个字段：

• codec_name：音频链路需要绑定的 codec 名称，声卡注册时会遍历 codec_list，找到同名的 codec 并绑定；
• platform_name：音频链路需要绑定的 platform 名称，声卡注册时会遍历 platform_list，找到同名的 platform 并绑定；
• cpu_dai_name：音频链路需要绑定的 cpu_dai 名称，声卡注册时会遍历 dai_list，找到同名的 dai 并绑定；
• codec_dai_name：音频链路需要绑定的 codec_dai 名称，声卡注册时会遍历 dai_list，找到同名的 dai 并绑定；
ops：重点留意 hw_params() 回调，一般来说这个回调是要实现的，用于配置 codec、codec_dai、cpu_dai 的数据格式和系统时钟。在 3.6. Codec audio operations 小节中有描述。
/sound/soc/msm/msm8996.c 中的 dai_link 定义，两个音频链路分别用于 Media和 Voice：

``` cpp 
[->/sound/soc/msm/msm8996.c]
/* Digital audio interface glue - connects codec <---> CPU */
static struct snd_soc_dai_link msm8996_common_dai_links[] = {
	/* FrontEnd DAI Links */
	{
		.name = "MSM8996 Media1",
		.stream_name = "MultiMedia1",
		.cpu_dai_name = "MultiMedia1",
		.platform_name = "msm-pcm-dsp.0",
		.dynamic = 1,
		.async_ops = ASYNC_DPCM_SND_SOC_PREPARE,
		.dpcm_playback = 1,
		.dpcm_capture = 1,
		.trigger = {SND_SOC_DPCM_TRIGGER_POST,
			SND_SOC_DPCM_TRIGGER_POST},
		.codec_dai_name = "snd-soc-dummy-dai",
		.codec_name = "snd-soc-dummy",
		.ignore_suspend = 1,
		/* this dainlink has playback support */
		.ignore_pmdown_time = 1,
		.be_id = MSM_FRONTEND_DAI_MULTIMEDIA1
	},
	......
	{
		.name = "VoiceMMode1",
		.stream_name = "VoiceMMode1",
		.cpu_dai_name = "VoiceMMode1",
		.platform_name = "msm-pcm-voice",
		.dynamic = 1,
		.dpcm_playback = 1,
		.dpcm_capture = 1,
		.trigger = {SND_SOC_DPCM_TRIGGER_POST,
			    SND_SOC_DPCM_TRIGGER_POST},
		.no_host_mode = SND_SOC_DAI_LINK_NO_HOST,
		.ignore_suspend = 1,
		.ignore_pmdown_time = 1,
		.codec_dai_name = "snd-soc-dummy-dai",
		.codec_name = "snd-soc-dummy",
		.be_id = MSM_FRONTEND_DAI_VOICEMMODE1,
	},
	
}
```
除了 dai_link，机器中一些特定的音频控件和音频事件也可以在 machine_drv 定义，如耳机插拔检测、外部功放打开关闭等。

我们再分析 machine_drv 初始化过程：

``` cpp
[->/sound/soc/msm/msm8996.c]
static struct platform_driver msm8996_asoc_machine_driver = {
	.driver = {
		.name = DRV_NAME,
		.owner = THIS_MODULE,
		.pm = &snd_soc_pm_ops,
		.of_match_table = msm8996_asoc_machine_of_match,
	},
	.probe = msm8996_asoc_machine_probe,
	.remove = msm8996_asoc_machine_remove,
};
```

``` cpp
[->/sound/soc/msm/msm8996.c]
static int msm8996_asoc_machine_probe(struct platform_device *pdev)
{
	struct snd_soc_card *card;
	struct msm8996_asoc_mach_data *pdata;
	const char *mbhc_audio_jack_type = NULL;
	char *mclk_freq_prop_name;
	const struct of_device_id *match;
	int ret;
    ......
	pdata = devm_kzalloc(&pdev->dev,
			sizeof(struct msm8996_asoc_mach_data), GFP_KERNEL);


	card = populate_snd_card_dailinks(&pdev->dev);
	......
	match = of_match_node(msm8996_asoc_machine_of_match,
			pdev->dev.of_node);
	ret = msm8996_populate_dai_link_component_of_node(card);
	......
	
	ret = snd_soc_register_card(card);
	}
```
设置dailinks后，继而调用 snd_soc_register_card() 注册声卡。由于该过程很冗长，这里不一一贴代码分析了，但整个流程是比较简单的，流程图如下：

![Alt text](https://raw.githubusercontent.com/zhoujinjiankim/PicGo/master/audio.system/23-Audio-system-msm8996-probe-snd-card-register.png)

• 取出 platform_device 的私有数据，该私有数据就是 snd_soc_card ；
• snd_soc_register_card() 为每个 dai_link 分配一个 snd_soc_pcm_runtime 实例，别忘了之前提过 snd_soc_pcm_runtime 是 ASoC 的桥梁，保存着 codec、codec_dai、cpu_dai、platform 等硬件设备实例。
• 随后的工作都在 snd_soc_instantiate_card() 进行：
• 遍历 dai_list、codec_list、platform_list 链表，为每个音频链路找到对应的 cpu_dai、codec_dai、codec、platform；找到的 cpu_dai、codec_dai、codec、platform 保存到 snd_soc_pcm_runtime ，完成音频链路的设备绑定；
• 调用 snd_card_create() 创建声卡；
• soc_probe_dai_link() 依次回调 cpu_dai、codec、platform、codec_dai 的 probe() 函数，完成各音频设备的初始化，随后调用  
• soc_new_pcm() 创建 pcm 逻辑设备（因为涉及到本系列的重点内容，后面具体分析这个函数）；
最后调用 snd_card_register() 注册声卡。

[->sound/soc/soc-core.c]

![Alt text](https://raw.githubusercontent.com/zhoujinjiankim/PicGo/master/audio.system/24-Audio-system-msm8996-probe-snd-card.png)

soc_new_pcm 源码分析：

``` cpp
[->/sound/soc/soc-pcm.c]
/* create a new pcm */
int soc_new_pcm(struct snd_soc_pcm_runtime *rtd, int num)
{
	struct snd_soc_platform *platform = rtd->platform;
	struct snd_soc_dai *codec_dai;
	struct snd_soc_dai *cpu_dai = rtd->cpu_dai;
	struct snd_pcm *pcm;
	char new_name[64];
	int ret = 0, playback = 0, capture = 0;
	int i;

	if (rtd->dai_link->dynamic || rtd->dai_link->no_pcm) {
		playback = rtd->dai_link->dpcm_playback;
		capture = rtd->dai_link->dpcm_capture;
	} else {
		for (i = 0; i < rtd->num_codecs; i++) {
			codec_dai = rtd->codec_dais[i];
			if (codec_dai->driver->playback.channels_min)
				playback = 1;
			if (codec_dai->driver->capture.channels_min)
				capture = 1;
		}

		capture = capture && cpu_dai->driver->capture.channels_min;
		playback = playback && cpu_dai->driver->playback.channels_min;
	}

	if (rtd->dai_link->playback_only) {
		playback = 1;
		capture = 0;
	}

	if (rtd->dai_link->capture_only) {
		playback = 0;
		capture = 1;
	}

	/* create the PCM */
	if (rtd->dai_link->no_pcm) {
		snprintf(new_name, sizeof(new_name), "(%s)",
			rtd->dai_link->stream_name);

		ret = snd_pcm_new_internal(rtd->card->snd_card, new_name, num,
				playback, capture, &pcm);
	} else {
		if (rtd->dai_link->dynamic)
			snprintf(new_name, sizeof(new_name), "%s (*)",
				rtd->dai_link->stream_name);
		else
			snprintf(new_name, sizeof(new_name), "%s %s-%d",
				rtd->dai_link->stream_name,
				(rtd->num_codecs > 1) ?
				"multicodec" : rtd->codec_dai->name, num);

		ret = snd_pcm_new(rtd->card->snd_card, new_name, num, playback,
			capture, &pcm);
	}
	if (ret < 0) {
		dev_err(rtd->card->dev, "ASoC: can't create pcm for %s\n",
			rtd->dai_link->name);
		return ret;
	}
	dev_dbg(rtd->card->dev, "ASoC: registered pcm #%d %s\n",num, new_name);

	/* DAPM dai link stream work */
	INIT_DELAYED_WORK(&rtd->delayed_work, close_delayed_work);

	rtd->pcm = pcm;
	pcm->private_data = rtd;

	if (rtd->dai_link->no_pcm) {
		if (playback)
			pcm->streams[SNDRV_PCM_STREAM_PLAYBACK].substream->private_data = rtd;
		if (capture)
			pcm->streams[SNDRV_PCM_STREAM_CAPTURE].substream->private_data = rtd;
		if (platform->driver->pcm_new)
			rtd->platform->driver->pcm_new(rtd);
		goto out;
	}

	/* setup any hostless PCMs - i.e. no host IO is performed */
	if (rtd->dai_link->no_host_mode) {
		if (pcm->streams[SNDRV_PCM_STREAM_PLAYBACK].substream) {
			pcm->streams[SNDRV_PCM_STREAM_PLAYBACK].substream->hw_no_buffer = 1;
			snd_soc_set_runtime_hwparams(
				pcm->streams[SNDRV_PCM_STREAM_PLAYBACK].substream,
				&no_host_hardware);
		}
		if (pcm->streams[SNDRV_PCM_STREAM_CAPTURE].substream) {
			pcm->streams[SNDRV_PCM_STREAM_CAPTURE].substream->hw_no_buffer = 1;
			snd_soc_set_runtime_hwparams(
				pcm->streams[SNDRV_PCM_STREAM_CAPTURE].substream,
				&no_host_hardware);
		}
	}

	/* ASoC PCM operations */
	if (rtd->dai_link->dynamic) {
		rtd->ops.open		= dpcm_fe_dai_open;
		rtd->ops.hw_params	= dpcm_fe_dai_hw_params;
		rtd->ops.prepare	= dpcm_fe_dai_prepare;
		rtd->ops.trigger	= dpcm_fe_dai_trigger;
		rtd->ops.hw_free	= dpcm_fe_dai_hw_free;
		rtd->ops.close		= dpcm_fe_dai_close;
		rtd->ops.pointer	= soc_pcm_pointer;
		rtd->ops.delay_blk	= soc_pcm_delay_blk;
		rtd->ops.ioctl		= soc_pcm_ioctl;
		rtd->ops.compat_ioctl   = soc_pcm_compat_ioctl;
	} else {
		rtd->ops.open		= soc_pcm_open;
		rtd->ops.hw_params	= soc_pcm_hw_params;
		rtd->ops.prepare	= soc_pcm_prepare;
		rtd->ops.trigger	= soc_pcm_trigger;
		rtd->ops.hw_free	= soc_pcm_hw_free;
		rtd->ops.close		= soc_pcm_close;
		rtd->ops.pointer	= soc_pcm_pointer;
		rtd->ops.delay_blk	= soc_pcm_delay_blk;
		rtd->ops.ioctl		= soc_pcm_ioctl;
		rtd->ops.compat_ioctl   = soc_pcm_compat_ioctl;
	}

	if (platform->driver->ops) {
		rtd->ops.ack		= platform->driver->ops->ack;
		rtd->ops.copy		= platform->driver->ops->copy;
		rtd->ops.silence	= platform->driver->ops->silence;
		rtd->ops.page		= platform->driver->ops->page;
		rtd->ops.mmap		= platform->driver->ops->mmap;
		rtd->ops.restart	= platform->driver->ops->restart;
	}

	if (playback)
		snd_pcm_set_ops(pcm, SNDRV_PCM_STREAM_PLAYBACK, &rtd->ops);

	if (capture)
		snd_pcm_set_ops(pcm, SNDRV_PCM_STREAM_CAPTURE, &rtd->ops);

	if (platform->driver->pcm_new) {
		ret = platform->driver->pcm_new(rtd);
		if (ret < 0) {
			dev_err(platform->dev,
				"ASoC: pcm constructor failed: %d\n",
				ret);
			return ret;
		}
	}

	pcm->private_free = platform->driver->pcm_free;
	
	return ret;
}
```
可见 soc_new_pcm() 最主要的工作是创建 pcm 逻辑设备，创建回放子流和录制子流实例，并初始化回放子流和录制子流的 pcm 操作函数（数据搬运时，需要调用这些函数来驱动 codec、codec_dai、cpu_dai、dma 设备工作）。
#### （六）、声卡和 PCM 设备的建立过程
前面几章分析了 Codec、Platform、Machine 驱动的组成部分及其注册过程，这三者都是物理设备相关的，大家应该对音频物理链路有了一定的认知。接着分析音频驱动的中间层，由于这些并不是真正的物理设备，故我们称之为逻辑设备。

PCM 逻辑设备，我们又习惯称之为 PCM 中间层或 pcm native，起着承上启下的作用：往上是与用户态接口的交互，实现音频数据在用户态和内核态之间的拷贝；往下是触发 codec、platform、machine 的操作函数，实现音频数据在 dma_buffer <-> cpu_dai <-> codec 之间的传输。后面章节将会详细分析这个过程，这里还是先从声卡的注册谈起。
声卡驱动中，一般挂载着多个逻辑设备，看看我们计算机的声卡驱动有几个逻辑设备：

``` cpp
adb shell cat /proc/asound/devices 
  2: [ 0]   : control
  3: [ 0- 0]: digital audio playback
  4: [ 0- 0]: digital audio capture
  5: [ 0- 1]: digital audio playback
  6: [ 0- 1]: digital audio capture
 ......
 27: [ 0-16]: digital audio playback
 28: [ 0-16]: digital audio capture
 29: [ 0-17]: digital audio playback
 30: [ 0-17]: digital audio capture
 33:        : timer
```

> digital audio playback	用于回放的 PCM 设备 
> digital audio capture	用于录制的 PCM 设备
> control	用于声卡控制的 CTL 设备，如通路控制、音量调整等
> timer	定时器设备
手机系统中，通常我们更关心 PCM 和 CTL 这两种设备。

设备节点如下：

``` cpp
adb shell ls -l /dev/snd
crw-rw---- 1 system audio 116,  51 1970-06-19 02:07 comprC0D24
crw-rw---- 1 system audio 116,  52 1970-06-19 02:07 comprC0D27
crw-rw---- 1 system audio 116,  53 1970-06-19 02:07 comprC0D28
......
crw-rw---- 1 system audio 116,   2 1970-06-19 02:07 controlC0
crw-rw---- 1 system audio 116,  59 1970-06-19 02:07 hwC0D1000
crw-rw---- 1 system audio 116,  66 1970-06-19 02:07 hwC0D11
crw-rw---- 1 system audio 116,  67 1970-06-19 02:07 hwC0D12
crw-rw---- 1 system audio 116,  76 1970-06-19 02:07 hwC0D13
......
crw-rw---- 1 system audio 116,  13 1970-06-19 02:07 pcmC0D6c
crw-rw---- 1 system audio 116,  14 1970-06-19 02:07 pcmC0D7p
crw-rw---- 1 system audio 116,  15 1970-06-19 02:07 pcmC0D8c
crw-rw---- 1 system audio 116,  33 1970-06-19 02:07 timer
```

可以看到这些设备节点的 Major=116，Minor 则与 /proc/asound/devices 所列的对应起来，都是字符设备。上层可以通过 open/close/read/write/ioctl 等系统调用来操作声卡设备，这和其他字符设备类似，但一般情况下我们会使用已封装好的用户接口库如 tinyalsa、alsa-lib。

##### 6.1. 声卡结构概述
回顾下 ASoC 是如何注册声卡的，详细请参考章节 5. ASoC machine driver，这里仅简单陈述下：

• Machine 驱动初始化时，.name = "soc-audio" 的 platform_device 与 platform_driver 匹配成功，触发 soc_probe() 调用；
• 继而调用 snd_soc_register_card()： 
  ﹋• 为每个音频物理链路找到对应的 codec、codec_dai、cpu_dai、platform 设备实例，完成 dai_link 的绑定；
  ﹋ • 调用 snd_card_create() 创建声卡；
  ﹋ • 依次回调 cpu_dai、codec、platform 的 probe() 函数，完成物理设备的初始化；
• 随后调用 soc_new_pcm()： 
  ﹋ • 设置 pcm native 中要使用的 pcm 操作函数，这些函数用于驱动音频物理设备，包括 machine、codec_dai、cpu_dai、platform；
  ﹋ • 调用 snd_pcm_new() 创建 pcm 逻辑设备，回放子流和录制子流都在这里创建；
  ﹋ • 回调 platform 驱动的 pcm_new()，完成音频 dma 设备初始化和 dma buffer 内存分配；
• 最后调用 snd_card_register() 注册声卡。
关于音频物理设备部分（Codec/Platform/Machine）不再累述，下面详细分析声卡和 PCM 逻辑设备的注册过程。

上面提到声卡驱动上挂着多个逻辑子设备，有 pcm 音频数据流、control 混音器、midi 迷笛、timer 定时器等。

``` cpp
                  +-----------+
                  | snd_card  |
                  +-----------+
                    |   |   |
        +-----------+   |   +------------+
        |               |                |
+-----------+    +-----------+    +-----------+
 |  snd_pcm  |    |snd_control|    | snd_timer |    ...
 +-----------+    +-----------+    +-----------+
```
这些与声音相关的逻辑设备都在结构体 snd_card 管理之下，可以说 snd_card 是 alsa 中最顶层的结构。我们再看看 alsa 声卡驱动的大致结构图（不是严格的 UML 类图，有结构体定义、模块关系、函数调用，方便标示结构模块的层次及关系）：
![Alt text](https://raw.githubusercontent.com/zhoujinjiankim/PicGo/master/audio.system/25-Audio-system-snd-card-uml.png)


snd_cards：记录着所注册的声卡实例，每个声卡实例有着各自的逻辑设备，如 PCM 设备、CTL 设备、MIDI 设备等，并一一记录到 snd_card 的 devices 链表上 
snd_minors：记录着所有逻辑设备的上下文信息，它是声卡逻辑设备与系统调用 API 之间的桥梁；每个 snd_minor 在逻辑设备注册时被填充，在逻辑设备使用时就可以从该结构体中得到相应的信息（主要是系统调用函数集 file_operations）

##### 6.2. 声卡的创建snd_card_create()

``` cpp
[->sound/core/init.c]
/**
 *  snd_card_new - create and initialize a soundcard structure
 *  @parent: the parent device object
 *  @idx: card index (address) [0 ... (SNDRV_CARDS-1)]
 *  @xid: card identification (ASCII string)
 *  @module: top level module for locking
 *  @extra_size: allocate this extra size after the main soundcard structure
 *  @card_ret: the pointer to store the created card instance
 *
 *  Creates and initializes a soundcard structure.
 *
 *  The function allocates snd_card instance via kzalloc with the given
 *  space for the driver to use freely.  The allocated struct is stored
 *  in the given card_ret pointer.
 *
 *  Return: Zero if successful or a negative error code.
 */
int snd_card_new(struct device *parent, int idx, const char *xid,
		    struct module *module, int extra_size,
		    struct snd_card **card_ret)
```

注释非常详细，简单说下：
idx：声卡的编号，如为 -1，则由系统自动分配
xid：声卡标识符，如为 NULL，则以 snd_card 的 shortname 或 longname 代替
card_ret：返回所创建的声卡实例的指针
如下是Google Pixel手机的声卡信息：
``` cpp
adb shell
sailfish:/ $ cat /proc/asound/cards
 0 [msm8996tashamar]: msm8996-tasha-m - msm8996-tasha-marlin-snd-card
                      msm8996-tasha-marlin-snd-card
```

##### 6.3. 逻辑设备的创建
当声卡实例建立后，接着可以创建声卡下面的各个逻辑设备了。每个逻辑设备创建时，都会调用 snd_device_new() 生成一个 snd_device 实例，并把该实例挂到声卡 snd_card 的 devices 链表上。alsa 驱动为各种逻辑设备提供了创建接口，如下：

> PCM	snd_pcm_new() 
> CONTROL	snd_ctl_create()
> MIDI	snd_rawmidi_new() 
> TIMER	snd_timer_new()
> SEQUENCER	snd_seq_device_new() 
> JACK	snd_jack_new()

这些接口的一般过程如下：

``` cpp
int snd_xxx_new()
{
    // 这些接口供逻辑设备注册时回调
    static struct snd_device_ops ops = {
        .dev_free = snd_xxx_dev_free,
        .dev_register = snd_xxx_dev_register,
        .dev_disconnect = snd_xxx_dev_disconnect,
    };

    // 逻辑设备实例初始化

    // 新建一个设备实例 snd_device，挂到 snd_card 的 devices 链表上，把该逻辑设备纳入声卡的管理当中，SNDRV_DEV_xxx 是逻辑设备的类型
    return snd_device_new(card, SNDRV_DEV_xxx, card, &ops);
}
```
其中 snd_device_ops 是声卡逻辑设备的注册函数集，dev_register() 回调尤其重要，它在声卡注册时被调用，用于建立系统的设备节点，/dev/snd/ 目录的设备节点都是在这里创建的，通过这些设备节点可系统调用 open/release/read/write/ioctl… 访问操作该逻辑设备。

例如 snd_ctl_dev_register()：

``` cpp
[->/sound/core/control.c]
static const struct file_operations snd_ctl_f_ops =
{
	.owner =	THIS_MODULE,
	.read =		snd_ctl_read,
	.open =		snd_ctl_open,
	.release =	snd_ctl_release,
	.llseek =	no_llseek,
	.poll =		snd_ctl_poll,
	.unlocked_ioctl =	snd_ctl_ioctl,
	.compat_ioctl =	snd_ctl_ioctl_compat,
	.fasync =	snd_ctl_fasync,
};
/*
 * registration of the control device
 */
static int snd_ctl_dev_register(struct snd_device *device)
{
	struct snd_card *card = device->device_data;
	int err, cardnum;
	char name[16];

	if (snd_BUG_ON(!card))
		return -ENXIO;
	cardnum = card->number;
	if (snd_BUG_ON(cardnum < 0 || cardnum >= SNDRV_CARDS))
		return -ENXIO;
	sprintf(name, "controlC%i", cardnum);
	if ((err = snd_register_device(SNDRV_DEVICE_TYPE_CONTROL, card, -1,
				       &snd_ctl_f_ops, card, name)) < 0)
		return err;
	return 0;
}

```
事实是调用 snd_register_device_for_dev ()：

``` cpp
[->/sound/core/sound.c]
int snd_register_device_for_dev(int type, struct snd_card *card, int dev,
				const struct file_operations *f_ops,
				void *private_data,
				const char *name, struct device *device)
{
	int minor;
	struct snd_minor *preg;
	preg = kmalloc(sizeof *preg, GFP_KERNEL);
	
	preg->type = type;
	preg->card = card ? card->number : -1;
	preg->device = dev;
	preg->f_ops = f_ops;
	preg->private_data = private_data;
	preg->card_ptr = card;
	mutex_lock(&sound_mutex);
#ifdef CONFIG_SND_DYNAMIC_MINORS
	minor = snd_find_free_minor(type);
#else
	minor = snd_kernel_minor(type, card, dev);
#endif
	......
	snd_minors[minor] = preg;
	preg->dev = device_create(sound_class, device, MKDEV(major, minor),
				  private_data, "%s", name);
	......

	mutex_unlock(&sound_mutex);
	return 0;
}
```

分配并初始化一个 snd_minor 实例；
保存该 snd_minor 实例到 snd_minors 数组中；
调用 device_create() 生成设备文件节点。

上面过程是声卡注册时才被回调的。

##### 6.4. 声卡的注册
当声卡下的所有逻辑设备都已经准备就绪后，就可以调用 snd_card_register() 注册声卡了：
![Alt text](https://raw.githubusercontent.com/zhoujinjiankim/PicGo/master/audio.system/26-Audio-system-snd-card-register.png.png)

• 创建声卡的 sysfs 设备；
• 调用 snd_device_register_all() 注册所有挂在该声卡下的逻辑设备；
• 建立 proc 信息文件和 sysfs 属性文件。

#### （七）、DAPM分析
##### 7.1、DAPM简介
 DAPM是Dynamic Audio Power Management的缩写，直译过来就是动态音频电源管理的意思，DAPM是为了使基于linux的移动设备上的音频子系统，在任何时候都工作在最小功耗状态下。DAPM对用户空间的应用程序来说是透明的，所有与电源相关的开关都在ASoc core中完成。用户空间的应用程序无需对代码做出修改，也无需重新编译，DAPM根据当前激活的音频流（playback/capture）和声卡中的mixer等的配置来决定那些音频控件的电源开关被打开或关闭。

DAPM是基于kcontrol改进过后的相应框架，增加了相应的电源管理机制，其电源管理机制其实就是按照相应的音频路径，完美的对各种部件的电源进行控制，而且按照某种顺序进行。
##### 7.1、kcontrol
通常，一个kcontrol代表着一个mixer（混音器），或者是一个mux（多路开关），又或者是一个音量控制器等等。 从上述文章中我们知道，定义一个kcontrol主要就是定义一个snd_kcontrol_new 结构，

``` cpp
[->/include/sound/control.h]
struct snd_kcontrol_new {
	snd_ctl_elem_iface_t iface;	/* interface identifier */
	unsigned int device;		/* device/client number */
	unsigned int subdevice;		/* subdevice (substream) number */
	const unsigned char *name;	/* ASCII name of item */
	unsigned int index;		/* index of item */
	unsigned int access;		/* access rights */
	unsigned int count;		/* count of same elements */
	snd_kcontrol_info_t *info;
	snd_kcontrol_get_t *get;
	snd_kcontrol_put_t *put;
	union {
		snd_kcontrol_tlv_rw_t *c;
		const unsigned int *p;
	} tlv;
	unsigned long private_value;
};
```
对于每个控件，我们需要定义一个和他对应的snd_kcontrol_new结构，这些snd_kcontrol_new结构会在声卡的初始化阶段，通过snd_soc_dapm_new_controls()函数注册到系统中，用户空间就可以通过tinymix查看和设定这些控件的状态。
编译/external/tinyalsa/得到tinymix, tinyplay, tinycap，Push到手机执行tinymix可得到如下类似信息。

``` cpp
......
990	BOOL	1	QUAT_MI2S_RX Audio Mixer MultiMedia10    Off
991	BOOL	1	QUAT_MI2S_RX Audio Mixer MultiMedia11    Off
992	BOOL	1	QUAT_MI2S_RX Audio Mixer MultiMedia12    Off
993	BOOL	1	QUAT_MI2S_RX Audio Mixer MultiMedia13    Off
994	BOOL	1	QUAT_MI2S_RX Audio Mixer MultiMedia14    Off
995	BOOL	1	QUAT_MI2S_RX Audio Mixer MultiMedia15    Off
996	BOOL	1	QUAT_MI2S_RX Audio Mixer MultiMedia16    Off
997	BOOL	1	MI2S_RX Audio Mixer MultiMedia1          Off
998	BOOL	1	MI2S_RX Audio Mixer MultiMedia2          Off
999	BOOL	1	MI2S_RX Audio Mixer MultiMedia3          Off
1000	BOOL	1	MI2S_RX Audio Mixer MultiMedia4          Off
1001	BOOL	1	MI2S_RX Audio Mixer MultiMedia5          Off
1002	BOOL	1	MI2S_RX Audio Mixer MultiMedia6          Off
......
```
snd_kcontrol_new结构中，几个主要的字段是get，put，private_value，get回调函数用于获取该控件当前的状态值，而put回调函数则用于设置控件的状态值，而private_value字段则根据不同的控件类型有不同的意义，比如对于普通的控件，private_value字段可以用来定义该控件所对应的寄存器的地址以及对应的控制位在寄存器中的位置信息。值得庆幸的是，ASoc系统已经为我们准备了大量的宏定义，用于定义常用的控件，这些宏定义位于include/sound/soc.h中。下面我们分别讨论一下如何用这些预设的宏定义来定义一些常用的控件。
##### 7.1.1、简单型的控件
SOC_SINGLE    SOC_SINGLE应该算是最简单的控件了，这种控件只有一个控制量，比如一个开关，或者是一个数值变量（比如Codec中某个频率，FIFO大小等等）。我们看看这个宏是如何定义的：
``` cpp
[->/include/sound/soc.h]
#define SOC_SINGLE(xname, reg, shift, max, invert) \
{	.iface = SNDRV_CTL_ELEM_IFACE_MIXER, .name = xname, \
	.info = snd_soc_info_volsw, .get = snd_soc_get_volsw,\
	.put = snd_soc_put_volsw, \
	.private_value = SOC_SINGLE_VALUE(reg, shift, max, invert, 0) }
```
宏定义的参数分别是：xname（该控件的名字），reg（该控件对应的寄存器的地址），shift（控制位在寄存器中的位移），max（控件可设置的最大值），invert（设定值是否逻辑取反）。这里又使用了一个宏来定义private_value字段：SOC_SINGLE_VALUE，我们看看它的定义：

``` cpp
[->/include/sound/soc.h]
#define SOC_DOUBLE_VALUE(xreg, shift_left, shift_right, xmax, xinvert, xautodisable) \
	((unsigned long)&(struct soc_mixer_control) \
	{.reg = xreg, .rreg = xreg, .shift = shift_left, \
	.rshift = shift_right, .max = xmax, .platform_max = xmax, \
	.invert = xinvert, .autodisable = xautodisable})
#define SOC_SINGLE_VALUE(xreg, xshift, xmax, xinvert, xautodisable) \
	SOC_DOUBLE_VALUE(xreg, xshift, xshift, xmax, xinvert, xautodisable)
```
这里实际上是定义了一个soc_mixer_control结构，然后把该结构的地址赋值给了private_value字段，soc_mixer_control结构是这样的：

``` cpp
[->/include/sound/soc.h]
/* mixer control */
struct soc_mixer_control {
	int min, max, platform_max;
	int reg, rreg;
	unsigned int shift, rshift;
	unsigned int sign_bit;
	unsigned int invert:1;
	unsigned int autodisable:1;
};

```
看来soc_mixer_control是控件特征的真正描述者，它确定了该控件对应寄存器的地址，位移值，最大值和是否逻辑取反等特性，控件的put回调函数和get回调函数需要借助该结构来访问实际的寄存器。
**SOC_SINGLE_TLV**    **SOC_SINGLE_TLV**是SOC_SINGLE的一种扩展，主要用于定义那些有增益控制的控件，例如音量控制器，EQ均衡器等等。

``` cpp
[->/include/sound/soc.h]
#define SOC_SINGLE_TLV(xname, reg, shift, max, invert, tlv_array) \
{	.iface = SNDRV_CTL_ELEM_IFACE_MIXER, .name = xname, \
	.access = SNDRV_CTL_ELEM_ACCESS_TLV_READ |\
		 SNDRV_CTL_ELEM_ACCESS_READWRITE,\
	.tlv.p = (tlv_array), \
	.info = snd_soc_info_volsw, .get = snd_soc_get_volsw,\
	.put = snd_soc_put_volsw, \
	.private_value = SOC_SINGLE_VALUE(reg, shift, max, invert, 0) }
```
从他的定义可以看出，用于设定寄存器信息的private_value字段的定义和SOC_SINGLE是一样的，甚至put、get回调函数也是使用同一套，唯一不同的是增加了一个tlv_array参数，并把它赋值给了tlv.p字段。用户空间可以通过对声卡的control设备发起以下两种ioctl来访问tlv字段所指向的数组：
  •  SNDRV_CTL_IOCTL_TLV_READ
  •  SNDRV_CTL_IOCTL_TLV_WRITE
  •  SNDRV_CTL_IOCTL_TLV_COMMAND

SOC_DOUBLE    与SOC_SINGLE相对应，区别是SOC_SINGLE只控制一个变量，而SOC_DOUBLE则可以同时在一个寄存器中控制两个相似的变量，最常用的就是用于一些立体声的控件，我们需要同时对左右声道进行控制，因为多了一个声道，参数也就相应地多了一个shift位移值

SOC_DOUBLE_R    与SOC_DOUBLE类似，对于左右声道的控制寄存器不一样的情况，使用SOC_DOUBLE_R来定义，参数中需要指定两个寄存器地址。
SOC_DOUBLE_TLV    与SOC_SINGLE_TLV对应的立体声版本，通常用于立体声音量控件的定义。

SOC_DOUBLE_R_TLV    左右声道有独立寄存器控制的SOC_DOUBLE_TLV版本

##### 7.1.2、Mixer控件
Mixer控件用于音频通道的路由控制，由多个输入和一个输出组成，多个输入可以自由地混合在一起，形成混合后的输出：
![Alt text](https://raw.githubusercontent.com/zhoujinjiankim/PicGo/master/audio.system/27-Audio-system-mixer-1525417497024.png)

对于Mixer控件，我们可以认为是多个简单控件的组合，通常，我们会为mixer的每个输入端都单独定义一个简单控件来控制该路输入的开启和关闭，反应在代码上，就是定义一个soc_kcontrol_new数组：

``` cpp
[->/sound/soc/codecs/wcd9335.c]
static const struct snd_kcontrol_new aif4_vi_mixer[] = {
	SOC_SINGLE_EXT("SPKR_VI_1", SND_SOC_NOPM, TASHA_TX14, 1, 0,
			tasha_vi_feed_mixer_get, tasha_vi_feed_mixer_put),
	SOC_SINGLE_EXT("SPKR_VI_2", SND_SOC_NOPM, TASHA_TX15, 1, 0,
			tasha_vi_feed_mixer_get, tasha_vi_feed_mixer_put),
};

```
##### 7.1.3、Mux控件
mux控件与mixer控件类似，也是多个输入端和一个输出端的组合控件，与mixer控件不同的是，mux控件的多个输入端同时只能有一个被选中。因此，mux控件所对应的寄存器，通常可以设定一段连续的数值，每个不同的数值对应不同的输入端被打开，与上述的mixer控件不同，ASoc用soc_enum结构来描述mux控件的寄存器信息：

``` cpp
[->/include/sound/soc.h]
/* enumerated kcontrol */
struct soc_enum {
	int reg;
	unsigned char shift_l;
	unsigned char shift_r;
	unsigned int items;
	unsigned int mask;
	const char * const *texts;
	const unsigned int *values;
};
```

两个寄存器地址和位移字段：reg，reg2，shift_l，shift_r，用于描述左右声道的控制寄存器信息。字符串数组指针用于描述每个输入端对应的名字，value字段则指向一个数组，该数组定义了寄存器可以选择的值，每个值对应一个输入端，如果value是一组连续的值，通常我们可以忽略values参数。
##### 7.2、widget、path、route
前面一节中，我们介绍了音频驱动中对基本控制单元的封装：kcontrol。利用kcontrol，我们可以完成对音频系统中的mixer，mux，音量控制，音效控制，以及各种开关量的控制，通过对各种kcontrol的控制，使得音频硬件能够按照我们预想的结果进行工作。同时我们可以看到，kcontrol还是有以下几点不足：
只能描述自身，无法描述各个kcontrol之间的连接关系；
没有相应的电源管理机制；
没有相应的时间处理机制来响应播放、停止、上电、下电等音频事件；
为了防止pop-pop声，需要用户程序关注各个kcontrol上电和下电的顺序；
当一个音频路径不再有效时，不能自动关闭该路径上的所有的kcontrol；
为此，DAPM框架正是为了要解决以上这些问题而诞生的，DAPM目前已经是ASoc中的重要组成部分，让我们先从DAPM的数据结构开始，了解它的设计思想和工作原理。

##### 7.2.1、DAPM的基本单元：widget
文章的开头，我们说明了一下目前kcontrol的一些不足，而DAPM框架为了解决这些问题，引入了widget这一概念，所谓widget，其实可以理解为是kcontrol的进一步升级和封装，她同样是指音频系统中的某个部件，比如mixer，mux，输入输出引脚，电源供应器等等，甚至，我们可以定义虚拟的widget，例如playback stream widget。widget把kcontrol和动态电源管理进行了有机的结合，同时还具备音频路径的连结功能，一个widget可以与它相邻的widget有某种动态的连结关系。在DAPM框架中，widget用结构体snd_soc_dapm_widget来描述：

``` cpp
[->/include/sound/soc-dapm.h]
/* dapm widget */
struct snd_soc_dapm_widget {
	enum snd_soc_dapm_type id;
	const char *name;		/* widget name */
	const char *sname;	/* stream name */
	struct snd_soc_codec *codec;
	struct list_head list;
	struct snd_soc_dapm_context *dapm;

	void *priv;				/* widget specific data */
	struct regulator *regulator;		/* attached regulator */
	const struct snd_soc_pcm_stream *params; /* params for dai links */

	/* dapm control */
	int reg;				/* negative reg = no direct dapm */
	unsigned char shift;			/* bits to shift */
	unsigned int mask;			/* non-shifted mask */
	unsigned int on_val;			/* on state value */
	unsigned int off_val;			/* off state value */
	unsigned char power:1;			/* block power status */
	unsigned char active:1;			/* active stream on DAC, ADC's */
	unsigned char connected:1;		/* connected codec pin */
	unsigned char new:1;			/* cnew complete */
	unsigned char ext:1;			/* has external widgets */
	unsigned char force:1;			/* force state */
	unsigned char ignore_suspend:1;         /* kept enabled over suspend */
	unsigned char new_power:1;		/* power from this run */
	unsigned char power_checked:1;		/* power checked this run */
	int subseq;				/* sort within widget type */
	......
	/* widget input and outputs */
	struct list_head sources;
	struct list_head sinks;
	......
};
```
snd_soc_dapm_widget结构比较大，为了简洁一些，这里我没有列出该结构体的完整字段，不过不用担心，下面我会说明每个字段的意义：
id    该widget的类型值，比如snd_soc_dapm_output，snd_soc_dapm_mixer等等。

*name    该widget的名字

*sname    代表该widget所在stream的名字，比如对于snd_soc_dapm_dai_in类型的widget，会使用该字段。

*codec *platform    指向该widget所属的codec和platform。

list    所有注册到系统中的widget都会通过该list，链接到代表声卡的snd_soc_card结构的widgets链表头字段中。

*dapm    snd_soc_dapm_context结构指针，ASoc把系统划分为多个dapm域，每个widget属于某个dapm域，同一个域代表着同样的偏置电压供电策略，比如，同一个codec中的widget通常位于同一个dapm域，而平台上的widget可能又会位于另外一个platform域中。

*priv    有些widget可能需要一些专有的数据，可以使用该字段来保存，像snd_soc_dapm_dai_in类型的widget，会使用该字段来记住与之相关联的snd_soc_dai结构指针。

*regulator    对于snd_soc_dapm_regulator_supply类型的widget，该字段指向与之相关的regulator结构指针。

*params    目前对于snd_soc_dapm_dai_link类型的widget，指向该dai的配置信息的snd_soc_pcm_stream结构。

reg shift mask     这3个字段用来控制该widget的电源状态，分别对应控制信息所在的寄存器地址，位移值和屏蔽值。

value  on_val  off_val    电源状态的当前只，开启时和关闭时所对应的值。

power invert    用于指示该widget当前是否处于上电状态，invert则用于表明power字段是否需要逻辑反转。

active connected    分别表示该widget是否处于激活状态和连接状态，当和相邻的widget有连接关系时，connected位会被置1，否则置0。

new   我们定义好的widget（snd_soc_dapm_widget结构），在注册到声卡中时需要进行实例化，该字段用来表示该widget是否已经被实例化。

ext    表示该widget当前是否有外部连接，比如连接mic，耳机，喇叭等等。

force    该位被设置后，将会不管widget当前的状态，强制更新至新的电源状态。

ignore_suspend new_power power_checked    这些电源管理相关的字段。

subseq    该widget目前在上电或下电队列中的排序编号，为了防止在上下电的过程中出现pop-pop声，DAPM会给每个widget分配合理的上下电顺序。

*power_check    用于检查该widget是否应该上电或下电的回调函数指针。
event_flags    该字段是一个位或字段，每个位代表该widget会关注某个DAPM事件通知。只有被关注的通知事件会被发送到widget的事件处理回调函数中。

*event    DAPM事件处理回调函数指针。

num_kcontrols *kcontrol_news **kcontrols    这3个字段用来描述与该widget所包含的kcontrol控件，例如一个mixer控件或者是一个mux控件。

sources sinks    两个链表字段，两个widget如果有连接关系，会通过一个snd_soc_dapm_path结构进行连接，sources链表用于链接所有的输入path，sinks链表用于链接所有的输出path。

power_list    每次更新整个dapm的电源状态时，会根据一定的算法扫描所有的widget，然后把需要变更电源状态的widget利用该字段链接到一个上电或下电的链表中，扫描完毕后，dapm系统会遍历这两个链表执行相应的上电或下电操作。

dirty    链表字段，widget的状态变更后，dapm系统会利用该字段，把该widget加入到一个dirty链表中，稍后会对dirty链表进行扫描，以执行整个路径的更新。

inputs    该widget的所有有效路径中，连接到输入端的路径数量。

outputs    该widget的所有有效路径中，连接到输出端的路径数量。

*clk    对于snd_soc_dapm_clock_supply类型的widget，指向相关联的clk结构指针。

以上我们对snd_soc_dapm_widget结构的各个字段所代表的意义一一做出了说明，这里只是让大家现有个概念

##### 7.2.2、widget的种类

在DAPM框架中，把各种不同的widget划分为不同的种类，snd_soc_dapm_widget结构中的id字段用来表示该widget的种类，可选的种类都定义在一个枚举中：

``` cpp
[->/include/sound/soc-dapm.h]
/* dapm widget types */
enum snd_soc_dapm_type {
	snd_soc_dapm_input = 0,		/* input pin */
	snd_soc_dapm_output,		/* output pin */
	......
```
下面我们逐个解释一下这些widget的种类：
snd_soc_dapm_input     该widget对应一个输入引脚。
snd_soc_dapm_output    该widget对应一个输出引脚。
snd_soc_dapm_mux    该widget对应一个mux控件。
snd_soc_dapm_virt_mux    该widget对应一个虚拟的mux控件。
snd_soc_dapm_value_mux    该widget对应一个value类型的mux控件。
snd_soc_dapm_mixer    该widget对应一个mixer控件。
snd_soc_dapm_mixer_named_ctl    该widget对应一个mixer控件，但是对应的kcontrol的名字不会加入widget的名字作为前缀。
snd_soc_dapm_pga    该widget对应一个pga控件（可编程增益控件）。
snd_soc_dapm_out_drv    该widget对应一个输出驱动控件
snd_soc_dapm_adc    该widget对应一个ADC 
snd_soc_dapm_dac    该widget对应一个DAC 
snd_soc_dapm_micbias    该widget对应一个麦克风偏置电压控件
snd_soc_dapm_mic    该widget对应一个麦克风。
snd_soc_dapm_hp    该widget对应一个耳机。
snd_soc_dapm_spk    该widget对应一个扬声器。
snd_soc_dapm_line     该widget对应一个线路输入。
snd_soc_dapm_switch       该widget对应一个模拟开关。
snd_soc_dapm_vmid      该widget对应一个codec的vmid偏置电压。
snd_soc_dapm_pre      machine级别的专用widget，会先于其它widget执行检查操作。
snd_soc_dapm_post    machine级别的专用widget，会后于其它widget执行检查操作。
snd_soc_dapm_supply           对应一个电源或是时钟源。
snd_soc_dapm_regulator_supply  对应一个外部regulator稳压器。
snd_soc_dapm_clock_supply      对应一个外部时钟源。
snd_soc_dapm_aif_in            对应一个数字音频输入接口，比如I2S接口的输入端。
snd_soc_dapm_aif_out          对应一个数字音频输出接口，比如I2S接口的输出端。
snd_soc_dapm_siggen            对应一个信号发生器。
snd_soc_dapm_dai_in           对应一个platform或codec域的输入DAI结构。
snd_soc_dapm_dai_out        对应一个platform或codec域的输出DAI结构。
snd_soc_dapm_dai_link         用于链接一对输入/输出DAI结构。

##### 7.2.3、widget之间的连接器：path
之前已经提到，一个widget是有输入和输出的，而且widget之间是可以动态地进行连接的，那它们是用什么来连接两个widget的呢？DAPM为我们提出了path这一概念，path相当于电路中的一根跳线，它把一个widget的输出端和另一个widget的输入端连接在一起，path用snd_soc_dapm_path结构来描述：


``` cpp
[->/include/sound/soc-dapm.h]
/* dapm audio path between two widgets */
struct snd_soc_dapm_path {
	const char *name;

	/* source (input) and sink (output) widgets */
	struct snd_soc_dapm_widget *source;
	struct snd_soc_dapm_widget *sink;

	/* status */
	u32 connect:1;	/* source and sink widgets are connected */
	u32 walked:1;	/* path has been walked */
	u32 walking:1;  /* path is in the process of being walked */
	u32 weak:1;	/* path ignored for power management */

	int (*connected)(struct snd_soc_dapm_widget *source,
			 struct snd_soc_dapm_widget *sink);

	struct list_head list_source;
	struct list_head list_sink;
	struct list_head list_kcontrol;
	struct list_head list;
};
```
当widget之间发生连接关系时，snd_soc_dapm_path作为连接者，它的source字段会指向该连接的起始端widget，而它的sink字段会指向该连接的到达端widget，还记得前面snd_soc_dapm_widget结构中的两个链表头字段：sources和sinks么？widget的输入端和输出端可能连接着多个path，所有输入端的snd_soc_dapm_path结构通过list_sink字段挂在widget的souces链表中，同样，所有输出端的snd_soc_dapm_path结构通过list_source字段挂在widget的sinks链表中。这里可能大家会被搞得晕呼呼的，一会source，一会sink，不要紧，只要记住，连接的路径是这样的：起始端widget的输出-->path的输入-->path的输出-->到达端widget输入。
![Alt text](https://raw.githubusercontent.com/zhoujinjiankim/PicGo/master/audio.system/28-Audio-system-snd_soc_dapm_path.png)
另外，snd_soc_dapm_path结构的list字段用于把所有的path注册到声卡中，其实就是挂在snd_soc_card结构的paths链表头字段中。如果你要自己定义方法来检查path的当前连接状态，你可以提供自己的connected回调函数指针。

connect，walked，walking，weak是几个辅助字段，用于帮助所有path的遍历。

##### 7.2.4、widget的连接关系：route
通过上一节的内容，我们知道，一个路径的连接至少包含以下几个元素：起始端widget，跳线path，到达端widget，在DAPM中，用snd_soc_dapm_route结构来描述这样一个连接关系：

``` cpp
[->/include/sound/soc-dapm.h]
struct snd_soc_dapm_route {
	const char *sink;
	const char *control;
	const char *source;

	/* Note: currently only supported for links where source is a supply */
	int (*connected)(struct snd_soc_dapm_widget *source,
			 struct snd_soc_dapm_widget *sink);
};
```
sink指向到达端widget的名字字符串，source指向起始端widget的名字字符串，control指向负责控制该连接所对应的kcontrol名字字符串，connected回调则定义了上一节所提到的自定义连接检查回调函数。该结构的意义很明显就是：source通过一个kcontrol，和sink连接在一起，现在是否处于连接状态，请调用connected回调函数检查。
这里直接使用名字字符串来描述连接关系，所有定义好的route，最后都要注册到dapm系统中，dapm会根据这些名字找出相应的widget，并动态地生成所需要的snd_soc_dapm_path结构，正确地处理各个链表和指针的关系，实现两个widget之间的连接

##### 7.3、建立widget之间的连接关系

前面我们主要着重于codec、platform、machine驱动程序中如何使用和建立dapm所需要的widget，route，这些是音频驱动开发人员必须要了解的内容，经过前几章的介绍，我们应该知道如何在alsa音频驱动的3大部分（codec、platform、machine）中，按照所使用的音频硬件结构，定义出相应的widget，kcontrol，以及必要的音频路径，而在本节中，我们将会深入dapm的核心部分，看看各个widget之间是如何建立连接关系，形成一条完整的音频路径。

前面我们已经简单地介绍过，驱动程序需要使用以下api函数创建widget：

• snd_soc_dapm_new_controls()
实际上，这个函数只是创建widget的第一步，它为每个widget分配内存，初始化必要的字段，然后把这些widget挂在代表声卡的snd_soc_card的widgets链表字段中。要使widget之间具备连接能力，我们还需要第二个函数：
• snd_soc_dapm_new_widgets()
这个函数会根据widget的信息，创建widget所需要的dapm kcontrol，这些dapm kcontol的状态变化，代表着音频路径的变化，从而影响着各个widget的电源状态。看到函数的名称可能会迷惑一下，实际上，snd_soc_dapm_new_controls的作用更多地是创建widget，而snd_soc_dapm_new_widget的作用则更多地是创建widget所包含的kcontrol，所以在我看来，这两个函数名称应该换过来叫更好！下面我们分别介绍一下这两个函数是如何工作的。
##### 7.3.1、创建widget
snd_soc_dapm_new_controls()函数完成widget的创建工作，并把这些创建好的widget注册在声卡的widgets链表中，我们看看他的定义：

``` cpp
[->/sound/soc/soc-dapm.c]
int snd_soc_dapm_new_controls(struct snd_soc_dapm_context *dapm,
	const struct snd_soc_dapm_widget *widget,
	int num)
{
	struct snd_soc_dapm_widget *w;
	int i;
	int ret = 0;

	mutex_lock_nested(&dapm->card->dapm_mutex, SND_SOC_DAPM_CLASS_INIT);
	for (i = 0; i < num; i++) {
		w = snd_soc_dapm_new_control(dapm, widget);
		if (!w) {
			dev_err(dapm->dev,
				"ASoC: Failed to create DAPM control %s\n",
				widget->name);
			ret = -ENOMEM;
			break;
		}
		widget++;
	}
	mutex_unlock(&dapm->card->dapm_mutex);
	return ret;
}
```
该函数只是简单的一个循环，为传入的widget模板数组依次调用snd_soc_dapm_new_control函数，实际的工作由snd_soc_dapm_new_control完成，继续进入该函数，看看它做了那些工作。
我们之前已经说过，驱动中定义的snd_soc_dapm_widget数组，只是作为一个模板，所以，snd_soc_dapm_new_control所做的第一件事，就是为该widget重新分配内存，并把模板的内容拷贝过来：

``` cpp
[->/sound/soc/soc-dapm.c]
static struct snd_soc_dapm_widget *
snd_soc_dapm_new_control(struct snd_soc_dapm_context *dapm,
			 const struct snd_soc_dapm_widget *widget)
{
	struct snd_soc_dapm_widget *w;
	const char *prefix;
	int ret;

	if ((w = dapm_cnew_widget(widget)) == NULL)
		return NULL;
    //由dapm_cnew_widget完成内存申请和拷贝模板的动作。接下来，根据widget的类型做不同的处理：
	switch (w->id) {
	case snd_soc_dapm_regulator_supply:
	......
    }
	prefix = soc_dapm_prefix(dapm);
	//对于snd_soc_dapm_regulator_supply类型的widget，根据widget的名称获取对应的regulator结构，对于snd_soc_dapm_clock_supply类型的widget，根据widget的名称，获取对应的clock结构。接下来，根据需要，在widget的名称前加入必要的前缀：
	if (prefix) {
		w->name = kasprintf(GFP_KERNEL, "%s %s", prefix, widget->name);
		if (widget->sname)
			w->sname = kasprintf(GFP_KERNEL, "%s %s", prefix,
					     widget->sname);
	} else {
		w->name = kasprintf(GFP_KERNEL, "%s", widget->name);
		if (widget->sname)
			w->sname = kasprintf(GFP_KERNEL, "%s", widget->sname);
	}
	......
```

![Alt text](https://raw.githubusercontent.com/zhoujinjiankim/PicGo/master/audio.system/29-Audio-system-widget-1525420992983.png)
当音频路径发生变化时，power_check回调会被调用，用于检查该widget的电源状态是否需要更新。power_check设置完成后，需要设置widget所属的codec、platform和dapm context，几个用于音频路径的链表也需要初始化，然后，把该widget加入到声卡的widgets链表中：

``` cpp
[->/sound/soc/soc-dapm.c：snd_soc_dapm_new_control()
w->dapm = dapm;  
w->codec = dapm->codec;  
w->platform = dapm->platform;  
INIT_LIST_HEAD(&w->sources);  
INIT_LIST_HEAD(&w->sinks);  
INIT_LIST_HEAD(&w->list);  
INIT_LIST_HEAD(&w->dirty);  
list_add(&w->list, &dapm->card->widgets);  
```

几个链表的作用如下：
sources    用于链接所有连接到该widget输入端的snd_soc_path结构
sinks    用于链接所有连接到该widget输出端的snd_soc_path结构
list    用于链接到声卡的widgets链表
dirty    用于链接到声卡的dapm_dirty链表
最后，把widget设置为connect状态：

``` cpp
[->/sound/soc/soc-dapm.c：snd_soc_dapm_new_control()
/* machine layer set ups unconnected pins and insertions */  
w->connected = 1;  
return w;  
```
connected字段代表着引脚的连接状态，目前，只有以下这些widget使用connected字段：
snd_soc_dapm_output
snd_soc_dapm_input
snd_soc_dapm_hp
snd_soc_dapm_spk
snd_soc_dapm_line
snd_soc_dapm_vmid
snd_soc_dapm_mic
snd_soc_dapm_siggen
驱动程序可以使用以下这些api来设置引脚的连接状态：
snd_soc_dapm_enable_pin
snd_soc_dapm_force_enable_pin
snd_soc_dapm_disable_pin
snd_soc_dapm_nc_pin
到此，widget已经被正确地创建并初始化，而且被挂在声卡的widgets链表中，以后我们就可以通过声卡的widgets链表来遍历所有的widget，再次强调一下snd_soc_dapm_new_controls函数所完成的主要功能：
为widget分配内存，并拷贝参数中传入的在驱动中定义好的模板
设置power_check回调函数
把widget挂在声卡的widgets链表中

##### 7.3.2、为widget建立dapm kcontrol
定义一个widget，我们需要指定两个很重要的内容：一个是用于控制widget的电源状态的reg/shift等寄存器信息，另一个是用于控制音频路径切换的dapm kcontrol信息，这些dapm kcontrol有它们自己的reg/shift寄存器信息用于切换widget的路径连接方式。前一节的内容中，我们只是创建了widget的实例，并把它们注册到声卡的widgts链表中，但是到目前为止，包含在widget中的dapm kcontrol并没有建立起来，dapm框架在声卡的初始化阶段，等所有的widget（包括machine、platform、codec）都创建好之后，通过snd_soc_dapm_new_widgets函数，创建widget内包含的dapm kcontrol，并初始化widget的初始电源状态和音频路径的初始连接状态。我们看看声卡的初始化函数，都有那些初始化与dapm有关：

``` cpp
[->/sound/soc/soc-dapm.c]
static int snd_soc_instantiate_card(struct snd_soc_card *card)  
{  
        ......  
        /* card bind complete so register a sound card */  
        ret = snd_card_create(SNDRV_DEFAULT_IDX1, SNDRV_DEFAULT_STR1,  
                        card->owner, 0, &card->snd_card);  
        ......  
   
        card->dapm.bias_level = SND_SOC_BIAS_OFF;  
        card->dapm.dev = card->dev;  
        card->dapm.card = card;  
        list_add(&card->dapm.list, &card->dapm_list);  
        ......  
        if (card->dapm_widgets)    /* 创建machine级别的widget  */  
                snd_soc_dapm_new_controls(&card->dapm, card->dapm_widgets,  
                                          card->num_dapm_widgets);  
        ......  
        snd_soc_dapm_link_dai_widgets(card);  /*  连接dai widget  */  
  
        if (card->controls)    /*  建立machine级别的普通kcontrol控件  */  
                snd_soc_add_card_controls(card, card->controls, card->num_controls);  
  
        if (card->dapm_routes)    /*  注册machine级别的路径连接信息  */  
                snd_soc_dapm_add_routes(&card->dapm, card->dapm_routes,  
                                        card->num_dapm_routes);  
        ......  
  
        if (card->fully_routed)    /*  如果该标志被置位，自动把codec中没有路径连接信息的引脚设置为无用widget  */  
                list_for_each_entry(codec, &card->codec_dev_list, card_list)  
                        snd_soc_dapm_auto_nc_codec_pins(codec);  
  
        snd_soc_dapm_new_widgets(card);    /*初始化widget包含的dapm kcontrol、电源状态和连接状态*/  
  
        ret = snd_card_register(card->snd_card);  
        ......  
        card->instantiated = 1;  
        snd_soc_dapm_sync(&card->dapm);  
        ......  
        return 0;  
}   
```
正如我添加的注释中所示，在完成machine级别的widget和route处理之后，调用的snd_soc_dapm_new_widgets函数，来为所有已经注册的widget初始化他们所包含的dapm kcontrol，并初始化widget的电源状态和路径连接状态。下面我们看看snd_soc_dapm_new_widgets函数的工作过程。
##### 7.3.2.1、snd_soc_dapm_new_widgets函数   
该函数通过声卡的widgets链表，遍历所有已经注册了的widget，其中的new字段用于判断该widget是否已经执行过snd_soc_dapm_new_widgets函数，如果num_kcontrols字段有数值，表明该widget包含有若干个dapm kcontrol，那么就需要为这些kcontrol分配一个指针数组，并把数组的首地址赋值给widget的kcontrols字段，该数组存放着指向这些kcontrol的指针，当然现在这些都是空指针，因为实际的kcontrol现在还没有被创建：

``` cpp
[->/sound/soc/soc-dapm.c]
int snd_soc_dapm_new_widgets(struct snd_soc_card *card)  
{  
        ......  
        list_for_each_entry(w, &card->widgets, list)  
        {                 
                if (w->new) continue;  
                                  
                if (w->num_kcontrols) {  
                        w->kcontrols = kzalloc(w->num_kcontrols *  
                                                sizeof(struct snd_kcontrol *),  
                                                GFP_KERNEL);  
                        ......  
                }  
```
接着，对几种能影响音频路径的widget，创建并初始化它们所包含的dapm kcontrol：

``` cpp
[->/sound/soc/soc-dapm.c:snd_soc_dapm_new_widgets()]
switch(w->id) {  
case snd_soc_dapm_switch:  
case snd_soc_dapm_mixer:  
case snd_soc_dapm_mixer_named_ctl:  
        dapm_new_mixer(w);  
        break;  
case snd_soc_dapm_mux:  
case snd_soc_dapm_virt_mux:  
case snd_soc_dapm_value_mux:  
        dapm_new_mux(w);  
        break;  
case snd_soc_dapm_pga:  
case snd_soc_dapm_out_drv:  
        dapm_new_pga(w);  
        break;  
default:  
        break;  
}  
```
需要用到的创建函数分别是：
dapm_new_mixer()    对于mixer类型，用该函数创建dapm kcontrol；
dapm_new_mux()   对于mux类型，用该函数创建dapm kcontrol；
dapm_new_pga()   对于pga类型，用该函数创建dapm kcontrol；
然后，根据widget寄存器的当前值，初始化widget的电源状态，并设置到power字段中：

``` cpp
[->/sound/soc/soc-dapm.c:snd_soc_dapm_new_widgets()]
/* Read the initial power state from the device */  
if (w->reg >= 0) {  
        val = soc_widget_read(w, w->reg) >> w->shift;  
        val &= w->mask;  
        if (val == w->on_val)  
                w->power = 1;  
}  
```

接着，设置new字段，表明该widget已经初始化完成，我们还要吧该widget加入到声卡的dapm_dirty链表中，表明该widget的状态发生了变化，稍后在合适的时刻，dapm框架会扫描dapm_dirty链表，统一处理所有已经变化的widget。为什么要统一处理？因为dapm要控制各种widget的上下电顺序，同时也是为了减少寄存器的读写次数（多个widget可能使用同一个寄存器）：

``` cpp
[->/sound/soc/soc-dapm.c:snd_soc_dapm_new_widgets()]
w->new = 1;  
  
dapm_mark_dirty(w, "new widget");  
dapm_debugfs_add_widget(w);  
```

最后，通过dapm_power_widgets函数，统一处理所有位于dapm_dirty链表上的widget的状态改变：

``` cpp
[->/sound/soc/soc-dapm.c:snd_soc_dapm_new_widgets()]
dapm_power_widgets(card, SND_SOC_DAPM_STREAM_NOP);  
......  
return 0;  
```
##### 7.3.2.2、dapm mixer kcontrol
上一节中，我们提到，对于mixer类型的dapm kcontrol，我们会使用dapm_new_mixer来完成具体的创建工作，先看代码后分析：

``` cpp
[->/sound/soc/soc-dapm.c]
static int dapm_new_mixer(struct snd_soc_dapm_widget *w)  
{  
        int i, ret;  
        struct snd_soc_dapm_path *path;  
  
        /* add kcontrol */  
（1）        for (i = 0; i < w->num_kcontrols; i++) {                                  
                /* match name */  
（2）                list_for_each_entry(path, &w->sources, list_sink) {               
                        /* mixer/mux paths name must match control name */  
（3）                        if (path->name != (char *)w->kcontrol_news[i].name)       
                                continue;  
  
（4）                        if (w->kcontrols[i]) {                                   
                                dapm_kcontrol_add_path(w->kcontrols[i], path);  
                                continue;  
                        }  
  
（5）                        ret = dapm_create_or_share_mixmux_kcontrol(w, i);        
                        if (ret < 0)  
                                return ret;  
  
（6）                        dapm_kcontrol_add_path(w->kcontrols[i], path);           
                }  
        }  
  
        return 0;  
}  
```
（1）  因为一个mixer是由多个kcontrol组成的，每个kcontrol控制着mixer的一个输入端的开启和关闭，所以，该函数会根据kcontrol的数量做循环，逐个建立对应的kcontrol。
（2）（3）  之前多次提到，widget之间使用snd_soc_path进行连接，widget的sources链表保存着所有和输入端连接的snd_soc_path结构，所以我们可以用kcontrol模板中指定的名字来匹配对应的snd_soc_path结构。
（4）  因为一个输入脚可能会连接多个输入源，所以可能在上一个输入源的path关联时已经创建了这个kcontrol，所以这里判断kcontrols指针数组中对应索引中的指针值，如果已经赋值，说明kcontrol已经在之前创建好了，所以我们只要简单地把连接该输入端的path加入到kcontrol的path_list链表中，并且增加一个虚拟的影子widget，该影子widget连接和输入端对应的源widget，因为使用了kcontrol本身的reg/shift等寄存器信息，所以实际上控制的是该kcontrol的开和关，这个影子widget只有在kcontrol的autodisable字段被设置的情况下才会被创建，该特性使得source的关闭时，与之连接的mixer的输入端也可以自动关闭，这个特性通过dapm_kcontrol_add_path来实现这一点：

``` cpp
[->/sound/soc/soc-dapm.c]
static void dapm_kcontrol_add_path(const struct snd_kcontrol *kcontrol,  
        struct snd_soc_dapm_path *path)  
{  
        struct dapm_kcontrol_data *data = snd_kcontrol_chip(kcontrol);  
        /*  把kcontrol连接的path加入到paths链表中  */  
        /*  paths链表所在的dapm_kcontrol_data结构会保存在kcontrol的private_data字段中  */  
        list_add_tail(&path->list_kcontrol, &data->paths);  
  
        if (data->widget) {  
                snd_soc_dapm_add_path(data->widget->dapm, data->widget,  
                    path->source, NULL, NULL);  
        }  
}  
```
（5）  如果kcontrol之前没有被创建，则通过dapm_create_or_share_mixmux_kcontrol创建这个输入端的kcontrol，同理，kcontrol对应的影子widget也会通过dapm_kcontrol_add_path判断是否需要创建。

##### 7.3.2.3、dapm mux kcontrol
因为一个widget最多只会包含一个mux类型的damp kcontrol，所以他的创建方法稍有不同，dapm框架使用dapm_new_mux函数来创建mux类型的dapm kcontrol：

``` cpp
[->/sound/soc/soc-dapm.c]
static int dapm_new_mux(struct snd_soc_dapm_widget *w)  
{         
        struct snd_soc_dapm_context *dapm = w->dapm;  
        struct snd_soc_dapm_path *path;  
        int ret;  
          
(1)     if (w->num_kcontrols != 1) {  
                dev_err(dapm->dev,  
                        "ASoC: mux %s has incorrect number of controls\n",  
                        w->name);  
                return -EINVAL;  
        }  
  
        if (list_empty(&w->sources)) {  
                dev_err(dapm->dev, "ASoC: mux %s has no paths\n", w->name);  
                return -EINVAL;  
        }  
  
(2)     ret = dapm_create_or_share_mixmux_kcontrol(w, 0);  
        if (ret < 0)  
                return ret;  
(3)       list_for_each_entry(path, &w->sources, list_sink)  
                dapm_kcontrol_add_path(w->kcontrols[0], path);  
        return 0;  
}  
```
（1）  对于mux类型的widget，因为只会有一个kcontrol，所以在这里做一下判断。
（2）  同样地，和mixer类型一样，也使用dapm_create_or_share_mixmux_kcontrol来创建这个kcontrol。
（3）  对每个输入端所连接的path都加入dapm_kcontrol_data结构的paths链表中，并且创建一个影子widget，用于支持autodisable特性。

##### 7.3.2.4、dapm pga kcontrol
目前对于pga类型的widget，kcontrol的创建函数是个空函数，所以我们不用太关注它：

``` cpp
[->/sound/soc/soc-dapm.c]
static int dapm_new_pga(struct snd_soc_dapm_widget *w)
{
	if (w->num_kcontrols)
		dev_err(w->dapm->dev,
			"ASoC: PGA controls not supported: '%s'\n", w->name);

	return 0;
}
```
dapm_create_or_share_mixmux_kcontrol函数
上面所说的mixer类型和mux类型的widget，在创建他们所包含的dapm kcontrol时，最后其实都是使用了dapm_create_or_share_mixmux_kcontrol函数来完成创建工作的，所以在这里我们有必要分析一下这个函数的工作原理。这个函数中有很大一部分代码实在处理kcontrol的名字是否要加入codec的前缀，我们会忽略这部分的代码，感兴趣的读者可以自己查看内核的代码，路径在：sound/soc/soc-dapm.c中，简化后的代码如下：

``` cpp
[->/sound/soc/soc-dapm.c]
static int dapm_create_or_share_mixmux_kcontrol(struct snd_soc_dapm_widget *w,  
        int kci)  
{  
          ......  
(1)       shared = dapm_is_shared_kcontrol(dapm, w, &w->kcontrol_news[kci],  
                                         &kcontrol);  
     
(2)       if (!kcontrol) {  
(3)            kcontrol = snd_soc_cnew(&w->kcontrol_news[kci], NULL, name,prefix）;  
               ......  
               kcontrol->private_free = dapm_kcontrol_free;  
(4)            ret = dapm_kcontrol_data_alloc(w, kcontrol);  
                ......  
(5)            ret = snd_ctl_add(card, kcontrol);  
                ......  
        }  
(6)     ret = dapm_kcontrol_add_widget(kcontrol, w);  
        ......  
(7)     w->kcontrols[kci] = kcontrol;  
        return 0;  
}  
```

（1）  为了节省内存，通过kcontrol名字的匹配查找，如果这个kcontrol已经在其他widget中已经创建好了，那我们不再创建，dapm_is_shared_kcontrol的参数kcontrol会返回已经创建好的kcontrol的指针。
（2）  如果kcontrol指针被赋值，说明在（1）中查找到了其他widget中同名的kcontrol，我们不用再次创建，只要共享该kcontrol即可。
（3）  标准的kcontrol创建函数，
（4）  如果widget支持autodisable特性，创建与该kcontrol所对应的影子widget，该影子widget的类型是：snd_soc_dapm_kcontrol。
（5）  标准的kcontrol创建函数，
（6）  把所有共享该kcontrol的影子widget（snd_soc_dapm_kcontrol），加入到kcontrol的private_data字段所指向的dapm_kcontrol_data结构中。
（7）  把创建好的kcontrol指针赋值到widget的kcontrols数组中。
需要注意的是，如果kcontol支持autodisable特性，一旦kcontrol由于source的关闭而被自动关闭，则用户空间只能操作该kcontrol的cache值，只有该kcontrol再次打开时，该cache值才会被真正地更新到寄存器中。
现在。我们总结一下，创建一个widget所包含的kcontrol所做的工作：
• 循环每一个输入端，为每个输入端依次执行下面的一系列操作
• 为每个输入端创建一个kcontrol，能共享的则直接使用创建好的kcontrol
• kcontrol的private_data字段保存着这些共享widget的信息
• 如果支持autodisable特性，每个输入端还要额外地创建一个虚拟的snd_soc_dapm_kcontrol类型的影子widget，该影子widget也记录在private_data字段中
• 创建好的kcontrol会依次存放在widget的kcontrols数组中，供路径的控制和匹配之用。

##### 7.3.2.5、为widget建立连接关系

如果widget之间没有连接关系，dapm就无法实现动态的电源管理工作，正是widget之间有了连结关系，这些连接关系形成了一条所谓的完成的音频路径，dapm可以顺着这条路径，统一控制路径上所有widget的电源状态，前面我们已经知道，widget之间是使用snd_soc_path结构进行连接的，驱动要做的是定义一个snd_soc_route结构数组，该数组的每个条目描述了目的widget的和源widget的名称，以及控制这个连接的kcontrol的名称，最终，驱动程序使用api函数snd_soc_dapm_add_routes来注册这些连接信息，接下来我们就是要分析该函数的具体实现方式：

``` cpp
[->/sound/soc/soc-dapm.c]
int snd_soc_dapm_add_routes(struct snd_soc_dapm_context *dapm,  
                            const struct snd_soc_dapm_route *route, int num)  
{  
        int i, r, ret = 0;  
  
        mutex_lock_nested(&dapm->card->dapm_mutex, SND_SOC_DAPM_CLASS_INIT);  
        for (i = 0; i < num; i++) {  
                r = snd_soc_dapm_add_route(dapm, route);  
                ......  
                route++;  
        }  
        mutex_unlock(&dapm->card->dapm_mutex);  
  
        return ret;  
}  
```
该函数只是一个循环，依次对参数传入的数组调用snd_soc_dapm_add_route，主要的工作由snd_soc_dapm_add_route完成。我们进入snd_soc_dapm_add_route函数看看：

``` cpp
[->/sound/soc/soc-dapm.c]
static int snd_soc_dapm_add_route(struct snd_soc_dapm_context *dapm,  
                                  const struct snd_soc_dapm_route *route)  
{  
        struct snd_soc_dapm_widget *wsource = NULL, *wsink = NULL, *w;  
        struct snd_soc_dapm_widget *wtsource = NULL, *wtsink = NULL;  
        const char *sink;  
        const char *source;  
        ......  
        list_for_each_entry(w, &dapm->card->widgets, list) {  
                if (!wsink && !(strcmp(w->name, sink))) {  
                        wtsink = w;  
                        if (w->dapm == dapm)  
                                wsink = w;  
                        continue;  
                }  
                if (!wsource && !(strcmp(w->name, source))) {  
                        wtsource = w;  
                        if (w->dapm == dapm)  
                                wsource = w;  
                }  
        }  
```
上面的代码我再次省略了关于名称前缀的处理部分。我们可以看到，用widget的名字来比较，遍历声卡的widgets链表，找出源widget和目的widget的指针，这段代码虽然正确，但我总感觉少了一个判断退出循环的条件，如果链表的开头就找到了两个widget，还是要遍历整个链表才结束循环，好浪费时间。
下面，如果在本dapm context中没有找到，则使用别的dapm context中找到的widget：

``` cpp
[->/sound/soc/soc-dapm.c:snd_soc_dapm_add_route()]
if (!wsink)  
        wsink = wtsink;  
if (!wsource)  
        wsource = wtsource; 
```
最后，使用来增加一条连接信息：

``` cpp
[->/sound/soc/soc-dapm.c:snd_soc_dapm_add_route()]
        ret = snd_soc_dapm_add_path(dapm, wsource, wsink, route->control,  
                route->connected);  
        ......  
  
        return 0;  
}  
```
snd_soc_dapm_add_path函数是整个调用链条中的关键，我们来分析一下：

``` cpp
[->/sound/soc/soc-dapm.c]
static int snd_soc_dapm_add_path(struct snd_soc_dapm_context *dapm,  
        struct snd_soc_dapm_widget *wsource, struct snd_soc_dapm_widget *wsink,  
        const char *control,  
        int (*connected)(struct snd_soc_dapm_widget *source,  
                         struct snd_soc_dapm_widget *sink))  
{  
        struct snd_soc_dapm_path *path;  
        int ret;  
  
        path = kzalloc(sizeof(struct snd_soc_dapm_path), GFP_KERNEL);  
        if (!path)  
                return -ENOMEM;  
  
        path->source = wsource;  
        path->sink = wsink;  
        path->connected = connected;  
        INIT_LIST_HEAD(&path->list);  
        INIT_LIST_HEAD(&path->list_kcontrol);  
        INIT_LIST_HEAD(&path->list_source);  
        INIT_LIST_HEAD(&path->list_sink);  
```
函数的一开始，首先为这个连接分配了一个snd_soc_path结构，path的source和sink字段分别指向源widget和目的widget，connected字段保存connected回调函数，初始化几个snd_soc_path结构中的几个链表。

``` cpp
[->/sound/soc/soc-dapm.c:snd_soc_dapm_add_path()]
/* check for external widgets */  
        if (wsink->id == snd_soc_dapm_input) {  
                if (wsource->id == snd_soc_dapm_micbias ||  
                        wsource->id == snd_soc_dapm_mic ||  
                        wsource->id == snd_soc_dapm_line ||  
                        wsource->id == snd_soc_dapm_output)  
                        wsink->ext = 1;  
        }  
        if (wsource->id == snd_soc_dapm_output) {  
                if (wsink->id == snd_soc_dapm_spk ||  
                        wsink->id == snd_soc_dapm_hp ||  
                        wsink->id == snd_soc_dapm_line ||  
                        wsink->id == snd_soc_dapm_input)  
                        wsource->ext = 1;  
        }  
```
这段代码用于判断是否有外部连接关系，如果有，置位widget的ext字段。判断方法从代码中可以方便地看出：
目的widget是一个输入脚，如果源widget是mic、line、micbias或output，则认为目的widget具有外部连接关系。
源widget是一个输出脚，如果目的widget是spk、hp、line或input，则认为源widget具有外部连接关系。

``` cpp
[->/sound/soc/soc-dapm.c:snd_soc_dapm_add_path()]
dapm_mark_dirty(wsource, "Route added");  
dapm_mark_dirty(wsink, "Route added");  
  
/* connect static paths */  
if (control == NULL) {  
        list_add(&path->list, &dapm->card->paths);  
        list_add(&path->list_sink, &wsink->sources);  
        list_add(&path->list_source, &wsource->sinks);  
        path->connect = 1;  
        return 0;  
}  
```
因为增加了连结关系，所以把源widget和目的widget加入到dapm_dirty链表中。如果没有kcontrol来控制该连接关系，则这是一个静态连接，直接用path把它们连接在一起。在接着往下看：

``` cpp
[->/sound/soc/soc-dapm.c:snd_soc_dapm_add_path()]
/* connect dynamic paths */  
switch (wsink->id) {  
case snd_soc_dapm_adc:  
case snd_soc_dapm_dac:  
case snd_soc_dapm_pga:  
case snd_soc_dapm_out_drv:  
case snd_soc_dapm_input:  
case snd_soc_dapm_output:  
case snd_soc_dapm_siggen:  
case snd_soc_dapm_micbias:  
case snd_soc_dapm_vmid:  
case snd_soc_dapm_pre:  
case snd_soc_dapm_post:  
case snd_soc_dapm_supply:  
case snd_soc_dapm_regulator_supply:  
case snd_soc_dapm_clock_supply:  
case snd_soc_dapm_aif_in:  
case snd_soc_dapm_aif_out:  
case snd_soc_dapm_dai_in:  
case snd_soc_dapm_dai_out:  
case snd_soc_dapm_dai_link:  
case snd_soc_dapm_kcontrol:  
        list_add(&path->list, &dapm->card->paths);  
        list_add(&path->list_sink, &wsink->sources);  
        list_add(&path->list_source, &wsource->sinks);  
        path->connect = 1;  
        return 0;  
```
按照目的widget来判断，如果属于以上这些类型，直接把它们连接在一起即可，这段感觉有点多余，因为通常以上这些类型的widget本来也没有kcontrol，直接用上一段代码就可以了，也许是dapm的作者们想着以后可能会有所扩展吧。

``` cpp
[->/sound/soc/soc-dapm.c:snd_soc_dapm_add_path()]
case snd_soc_dapm_mux:  
case snd_soc_dapm_virt_mux:  
case snd_soc_dapm_value_mux:  
        ret = dapm_connect_mux(dapm, wsource, wsink, path, control,  
                &wsink->kcontrol_news[0]);  
        if (ret != 0)  
                goto err;  
        break;  
case snd_soc_dapm_switch:  
case snd_soc_dapm_mixer:  
case snd_soc_dapm_mixer_named_ctl:  
        ret = dapm_connect_mixer(dapm, wsource, wsink, path, control);  
        if (ret != 0)  
                goto err;  
        break;  
```
目的widget如果是mixer和mux类型，分别用dapm_connect_mixer和dapm_connect_mux函数完成连接工作，这两个函数我们后面再讲。

``` cpp
[->/sound/soc/soc-dapm.c:snd_soc_dapm_add_path()]
        case snd_soc_dapm_hp:  
        case snd_soc_dapm_mic:  
        case snd_soc_dapm_line:  
        case snd_soc_dapm_spk:  
                list_add(&path->list, &dapm->card->paths);  
                list_add(&path->list_sink, &wsink->sources);  
                list_add(&path->list_source, &wsource->sinks);  
                path->connect = 0;  
                return 0;  
        }  
  
        return 0;  
err:  
        kfree(path);  
        return ret;  
}  
```
hp、mic、line和spk这几种widget属于外部器件，也只是简单地连接在一起，不过connect字段默认为是未连接状态。
现在，我们回过头来看看目的widget是mixer和mux这两种类型时的连接方式：
dapm_connect_mixer  用该函数连接一个目的widget为mixer类型的所有输入端：

``` cpp
[->/sound/soc/soc-dapm.c]
static int dapm_connect_mixer(struct snd_soc_dapm_context *dapm,  
        struct snd_soc_dapm_widget *src, struct snd_soc_dapm_widget *dest,  
        struct snd_soc_dapm_path *path, const char *control_name)  
{  
        int i;  
  
        /* search for mixer kcontrol */  
        for (i = 0; i < dest->num_kcontrols; i++) {  
                if (!strcmp(control_name, dest->kcontrol_news[i].name)) {  
                        list_add(&path->list, &dapm->card->paths);  
                        list_add(&path->list_sink, &dest->sources);  
                        list_add(&path->list_source, &src->sinks);  
                        path->name = dest->kcontrol_news[i].name;  
                        dapm_set_path_status(dest, path, i);  
                        return 0;  
                }  
        }  
        return -ENODEV;  
}  
```
用需要用来连接的kcontrol的名字，和目的widget中的kcontrol模板数组中的名字相比较，找出该kcontrol在widget中的编号，path的名字设置为该kcontrol的名字，然后用dapm_set_path_status函数来初始化该输入端的连接状态。连接两个widget的链表操作和其他widget是一样的。

dapm_connect_mux 用该函数连接一个目的widget是mux类型的所有输入端：

``` cpp
[->/sound/soc/soc-dapm.c]
static int dapm_connect_mux(struct snd_soc_dapm_context *dapm,  
        struct snd_soc_dapm_widget *src, struct snd_soc_dapm_widget *dest,  
        struct snd_soc_dapm_path *path, const char *control_name,  
        const struct snd_kcontrol_new *kcontrol)  
{  
        struct soc_enum *e = (struct soc_enum *)kcontrol->private_value;  
        int i;  
  
        for (i = 0; i < e->max; i++) {  
                if (!(strcmp(control_name, e->texts[i]))) {  
                        list_add(&path->list, &dapm->card->paths);  
                        list_add(&path->list_sink, &dest->sources);  
                        list_add(&path->list_source, &src->sinks);  
                        path->name = (char*)e->texts[i];  
                        dapm_set_path_status(dest, path, 0);  
                        return 0;  
                }  
        }  
  
        return -ENODEV;  
}  
```
和mixer类型一样用名字进行匹配，只不过mux类型的kcontrol只需一个，所以要通过private_value字段所指向的soc_enum结构找出匹配的输入脚编号，最后也是通过dapm_set_path_status函数来初始化该输入端的连接状态，因为只有一个kcontrol，所以第三个参数是0。连接两个widget的链表操作和其他widget也是一样的。
dapm_set_path_status    该函数根据传入widget中的kcontrol编号，读取实际寄存器的值，根据寄存器的值来初始化这个path是否处于连接状态，详细的代码这里就不贴了。
当widget之间通过path进行连接之后，他们之间的关系就如下图所示：
![Alt text](https://raw.githubusercontent.com/zhoujinjiankim/PicGo/master/audio.system/210-Audio-system-snd_soc_dapm_path.png)

到这里为止，我们为声卡创建并初始化好了所需的widget，各个widget也通过path连接在了一起，接下来，dapm等待用户的指令，一旦某个dapm kcontrol被用户空间改变，利用这些连接关系，dapm会重新创建音频路径，脱离音频路径的widget会被下电，加入音频路径的widget会被上电，所有的上下电动作都会自动完成，用户空间的应用程序无需关注这些变化，它只管按需要改变某个dapm kcontrol即可。

#### （八）、tinyplay playback、capture
##### 8.1、tinyplay playback
![Alt text](https://raw.githubusercontent.com/zhoujinjiankim/PicGo/master/audio.system/211-tiny-capture.png)


有时序图可知：主要涉及pcm_open()、pcm_write()、pcm_prepare()、pcm_start()。
##### 8.1.1、使用耳机播放
1. 启动音频播放
2. 启用 Rx codec 路径
tinymix 'RX1 MIX1 INP1' 'RX1'
tinymix 'RX2 MIX1 INP1' 'RX2'
tinymix 'RDAC2 MUX' 'RX2'
tinymix 'HPHL' 'Switch'
tinymix 'HPHR' 'Switch'
tinymix 'MI2S_RX Channels' 'Two
3. 启用用于通过 MI2S 接口进行播放的 DSP AFE
tinymix 'PRI_MI2S_RX Audio Mixer MultiMedia1' 1
4. 播放 PCM 音频
tinyplay <filename.wav> 
5. 停止音频播放
6. 禁用接收 Rx codec 路径
tinymix 'RX1 MIX1 INP1' 'ZERO'
tinymix 'RX2 MIX1 INP1' 'ZERO'
tinymix 'RDAC2 MUX' 'ZERO'
tinymix 'HPHL' 'ZERO'
tinymix 'HPHR' 'ZERO'
tinymix 'MI2S_RX Channels' 'One'
7. 禁用用于通过 I2S 接口进行音频播放的 DSP AFE
tinymix 'PRI_MI2S_RX Audio Mixer MultiMedia1' 0


##### 8.2、tinyplay capture
![Alt text](https://raw.githubusercontent.com/zhoujinjiankim/PicGo/master/audio.system/212-tiny-playback.png)


有时序图可知：主要涉及pcm_open()、pcm_read()、pcm_start()。
##### 8.2.1、使用音频录制
1. 输入以下命令：
//Enable DSP AFE for Audio Recording over I2S
tinymix 'MultiMedia1 Mixer TERT_MI2S_TX' 1
//Enable Codec TX Path
tinymix 'DEC1 MUX' 'ADC2'
tinymix 'ADC2 MUX' 'INP2’
2. 启动录音功能：
tinycap /data/rec.wav
3. 禁用 HeadsetX 设备 (AMIC2)：
tinymix 'MultiMedia1 Mixer TERT_MI2S_TX' 0
tinymix 'DEC1 MUX' 'ZERO'
tinymix 'ADC2 MUX' 'ZERO'

#### （九）、参考资料(特别感谢各位前辈的分析和图示)：
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

