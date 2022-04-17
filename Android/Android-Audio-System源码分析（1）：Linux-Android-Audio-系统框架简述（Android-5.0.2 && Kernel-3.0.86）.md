---
title: Android Audio System 源码分析（1）：Linux && Android Audio 系统框架简述（Android 5.0.2 && Kernel 3.0.86）
cover: https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/personal.website/post.cover.pictures.00007.jpg
categories:
  - Audio
tags:
  - Android
  - Linux
  - Audio
toc: true
abbrlink: 20200208
date: 2020-02-08 09:25:00
---

注：文章都是通过阅读 Android  && Linux 平台源码、各位前辈总结的资料、加上自己的思考分析总结出来的，其中难免有理不对的地方，欢迎大家批评指正。**文章为个人学习、研究、欣赏之用。图文内容、源码整理自互联网，如有侵权，请联系删除（◕‿◕）**，转载请注明出处（ ©Android @Linux 版权所有），谢谢(๑乛◡乛๑) 、（ ͡° ͜ʖ ͡°）、（ಡωಡ）！！！。
（==**文章基于 Kernel-3.0.86**==）&&（==**文章基于 Android 5.0.2**==）
[【开发板 - 友善之臂 FriendlyARM Cortex-A9 Tiny4412 ADK Exynos4412 （ Android 5.0.2）HD702高清电容屏 扩展套餐】](https://item.taobao.com/item.htm?spm=0.0.0.0.3GsGdQ&id=20131438062)
[【开发板 Android 5.0.2 && Kernel 3.0.86 源码链接： https://pan.baidu.com/s/1jJHm74q 密码：yfih】](https://pan.baidu.com/s/1jJHm74q)

正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢！！！

--------------------------------------------------------------------------------

#### （一）、音频基础知识

理解音频的一些基础知识，对于我们分析整个音频系统是大有裨益的。它可以让我们从实现的层面去思考，音频系统的目的是什么，然后才是怎么样去完成这个目的。

#####（1）声音有哪些重要属性呢？

##### 1.1、响度(Loudness)
响度就是人类可以感知到的各种声音的大小，也就是音量。响度与声波的振幅有直接关系。

##### 1.2、音调(Pitch)
音调与声音的频率有关系，当声音的频率越大时，人耳所感知到的音调就越高，否则就越低。

##### 1.3、音色(Quality)
同一种乐器，使用不同的材质来制作，所表现出来的音色效果是不一样的，这是由物体本身的结构特性所决定的。

如何将各种媒体源数字化呢？

![enter image description here](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/personal.website/Audio-system-1024px-A-D-A_Flow.svg.png) 

将声波波形信号通过ADC转换成计算机支持的二进制的过程叫做音频采样(Audio Sampling)。采样(Sampling)的核心是把连续的模拟信号转换成离散的数字信号。

![enter image description here](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/personal.website/Audio-system-sampling.png) 

##### 1.4、样本(Sample)
这是我们进行采样的初始资料，比如一段连续的声音波形。

##### 1.5、采样器(Sampler)
采样器是将样本转换成终态信号的关键。它可以是一个子系统，也可以指一个操作过程，甚至是一个算法，取决于不同的信号处理场景。理想的采样器要求尽可能不产生信号失真。

##### 1.6、量化(Quantization)
采样后的值还需要通过量化，也就是将连续值近似为某个范围内有限多个离散值的处理过程。因为原始数据是模拟的连续信号，而数字信号则是离散的，它的表达范围是有限的，所以量化是必不可少的一个步骤。

##### 1.7、编码(Coding)
计算机的世界里，所有数值都是用二进制表示的，因而我们还需要把量化值进行二进制编码。这一步通常与量化同时进行。

##### 1.8、采样率（samplerate）

采样就是把模拟信号数字化的过程，不仅仅是音频需要采样，所有的模拟信号都需要通过采样转换为可以用0101来表示的数字信号，示意图如下所示：

![enter image description here](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/personal.website/Audio-system-sampling-rate.png.png) 

蓝色代表模拟音频信号，红色的点代表采样得到的量化数值。

采样频率越高，红色的间隔就越密集，记录这一段音频信号所用的数据量就越大，同时音频质量也就越高。

根据奈奎斯特理论，采样频率只要不低于音频信号最高频率的两倍，就可以无损失地还原原始的声音。

通常人耳能听到频率范围大约在20Hz～20kHz之间的声音，为了保证声音不失真，采样频率应在40kHz以上。常用的音频采样频率有：8kHz、11.025kHz、22.05kHz、16kHz、37.8kHz、44.1kHz、48kHz、96kHz、192kHz等。

##### 1.9、量化精度（位宽）

上图（1.8）中，每一个红色的采样点，都需要用一个数值来表示大小，这个数值的数据类型大小可以是：4bit、8bit、16bit、32bit等等，位数越多，表示得就越精细，声音质量自然就越好，当然，数据量也会成倍增大。

常见的位宽是：8bit 或者 16bit

##### 1.10、 声道数（channels）

由于音频的采集和播放是可以叠加的，因此，可以同时从多个音频源采集声音，并分别输出到不同的扬声器，故声道数一般表示声音录制时的音源数量或回放时相应的扬声器数量。

单声道（Mono）和双声道（Stereo）比较常见，顾名思义，前者的声道数为1，后者为2

##### 1.11、音频帧（frame）

这个概念在应用开发中非常重要，网上很多文章都没有专门介绍这个概念。

音频跟视频很不一样，视频每一帧就是一张图像，而从上面的正玄波可以看出，音频数据是流式的，本身没有明确的一帧帧的概念，在实际的应用中，为了音频算法处理/传输的方便，一般约定俗成取2.5ms~60ms为单位的数据量为一帧音频。

这个时间被称之为“采样时间”，其长度没有特别的标准，它是根据编解码器和具体应用的需求来决定的，我们可以计算一下一帧音频帧的大小：

假设某音频信号是采样率为8kHz、双通道、位宽为16bit，20ms一帧，则一帧音频数据的大小为：

int size = 8000 x 2 x 16bit x 0.02s = 5120 bit = 640 byte

##### 1.12、常见的音频编码方式有哪些？

上面提到过，模拟的音频信号转换为数字信号需要经过采样和量化，量化的过程被称之为编码，根据不同的量化策略，产生了许多不同的编码方式，常见的编码方式有：PCM 和 ADPCM，这些数据代表着无损的原始数字音频信号，添加一些文件头信息，就可以存储为WAV文件了，它是一种由微软和IBM联合开发的用于音频数字存储的标准，可以很容易地被解析和播放。

我们在音频开发过程中，会经常涉及到WAV文件的读写，以验证采集、传输、接收的音频数据的正确性。

##### 1.13、常见的音频压缩格式有哪些？

首先简单介绍一下音频数据压缩的最基本的原理：因为有冗余信息，所以可以压缩。

（1） 频谱掩蔽效应： 人耳所能察觉的声音信号的频率范围为20Hz～20KHz，在这个频率范围以外的音频信号属于冗余信号。

（2） 时域掩蔽效应： 当强音信号和弱音信号同时出现时，弱信号会听不到，因此，弱音信号也属于冗余信号。

下面简单列出常见的音频压缩格式：

MP3，AAC，OGG，WMA，Opus，FLAC，APE，M4A，AMR，等等
##### 1.14、奈奎斯特采样理论
“当对被采样的模拟信号进行还原时，其最高频率只有采样频率的一半”。
换句话说，如果我们要完整重构原始的模拟信号，则采样频率就必须是它的两倍以上。比如人的声音范围是2~ 20kHZ,那么选择的采样频率就应该在40kHZ左右，数值太小则声音将产生失真现象，而数值太大也无法明显提升人耳所能感知的音质。

##### 1.15、总结（音频处理和播放过程）：
![enter image description here](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/personal.website/Audio-system-how-audio-works.png) 


#### （二）、Audio 系统框架
#####  总体Audio框架图
![enter image description here](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/personal.website/Audio-system-Android-Linux-arc.png) 

##### 2.1、APP
音乐播放器软件等等。

##### 2.2、Framework

Android也提供了另两个相似功能的类，即AudioTrack和AudioRecorder，MediaPlayerService内部的实现就是通过它们来完成的,只不过MediaPlayer/MediaRecorder提供了更强大的控制功能，相比前者也更易于使用。除此以外，Android系统还为我们控制音频系统提供了AudioManager、AudioService及AudioSystem类。这些都是framework为便利上层应用开发所设计的。

##### 2.3、Libraries
framework只是向应用程序提供访问Android库的桥梁，具体功能实现放在库中完成。比如上面的AudioTrack、AudioRecorder、MediaPlayer和MediaRecorder等等在库中都能找到相对应的类。

1、frameworks/av/media/libmedia【libmedia.so】
2、frameworks/av/services/audioflinger【libaudioflinger.so】
3、frameworks/av/media/libmediaplayerservice【libmediaplayerservice.so】
##### 2.4、HAL

从设计上来看，硬件抽象层是AudioFlinger直接访问的对象。这说明了两个问题，一方面AudioFlinger并不直接调用底层的驱动程序;另一方面，AudioFlinger上层模块只需要与它进行交互就可以实现音频相关的功能了。因而我们可以认为AudioFlinger是Android音频系统中真正的“隔离板”，无论下面如何变化，上层的实现都可以保持兼容。

音频方面的硬件抽象层主要分为两部分，即AudioFlinger和AudioPolicyService。实际上后者并不是一个真实的设备，只是采用虚拟设备的方式来让厂商可以方便地定制出自己的策略。抽象层的任务是将AudioFlinger/AudioPolicyService真正地与硬件设备关联起来，但又必须提供灵活的结构来应对变化——特别是对于Android这个更新相当频繁的系统。比如以前Android系统中的Audio系统依赖于ALSA-lib，但后期就变为了tinyalsa，这样的转变不应该对上层造成破坏。因而Audio HAL提供了统一的接口来定义它与AudioFlinger/AudioPolicyService之间的通信方式，这就是audio_hw_device、audio_stream_in及audio_stream_out等等存在的目的，这些Struct数据类型内部大多只是函数指针的定义，是一些“壳”。当AudioFlinger/AudioPolicyService初始化时，它们会去寻找系统中最匹配的实现(这些实现驻留在以audio.primary.*,audio.a2dp.*为名的各种库中)来填充这些“壳”。根据产品的不同，音频设备存在很大差异，在Android的音频架构中，这些问题都是由HAL层的audio.primary等等库来解决的，而不需要大规模地修改上层实现。换句话说，厂商在定制时的重点就是如何提供这部分库的高效实现了。
##### 2.5、Tinyalsa
源码在external/tinyalsa目录下
Tinyalsa：tinyplay/tinycap/tinymix，这些用户程序直接调用 alsa 用户库接口来实现放音、录音、控制
##### 2.6、Kernel部分

##### 2.6.1、ALSA 和 ASoC 
Native ALSA Application：tinyplay/tinycap/tinymix，这些用户程序直接调用 alsa 用户库接口来实现放音、录音、控制
ALSA Library API：alsa 用户库接口，常见有 tinyalsa、alsa-lib
ALSA CORE：alsa 核心层，向上提供逻辑设备（PCM/CTL/MIDI/TIMER/…）系统调用，向下驱动硬件设备（Machine/I2S/DMA/CODEC）
ASoC CORE：asoc 是建立在标准 alsa core 基础上，为了更好支持嵌入式系统和应用于移动设备的音频 codec 的一套软件体系
Hardware Driver：音频硬件设备驱动，由三大部分组成，分别是 Machine、Platform、Codec
##### 2.6.2、ASoC 
ASoC被分为Machine、Platform和Codec三大部分。其中的Machine驱动负责Platform和Codec之间的耦合和设备或板子特定的代码。Platform驱动的主要作用是完成音频数据的管理，最终通过CPU的数字音频接口（DAI）把音频数据传送给Codec进行处理，最终由Codec输出驱动耳机或者是喇叭的音信信号。

##### 2.6.2.1、**Machine**
用于描述设备组件信息和特定的控制如耳机/外放等。

> 是指某一款机器，可以是某款设备，某款开发板，又或者是某款智能手机，由此可以看出Machine几乎是不可重用的，每个Machine上的硬件实现可能都不一样，CPU不一样，Codec不一样，音频的输入、输出设备也不一样，Machine为CPU、Codec、输入输出设备提供了一个载体。

这一部分将平台驱动和Codec驱动绑定在一起，描述了板级的硬件特征。主要负责Platform和Codec之间的耦合以及部分和设备或板子特定的代码。Machine驱动负责处理机器特有的一些控件和音频事件（例如，当播放音频时，需要先行打开一个放大器）；单独的Platform和Codec驱动是不能工作的，它必须由Machine驱动把它们结合在一起才能完成整个设备的音频处理工作。ASoC的一切都从Machine驱动开始，包括声卡的注册，绑定Platform和Codec驱动等等

##### 2.6.2.2、**Platform**
用于实现平台相关的DMA驱动和音频接口等。

> 一般是指某一个SoC平台，比如pxaxxx,s3cxxxx,omapxxx等等，与音频相关的通常包含该SoC中的时钟、DMA、I2S、PCM等等，只要指定了SoC，那么我们可以认为它会有一个对应的Platform，它只与SoC相关，与Machine无关，这样我们就可以把Platform抽象出来，使得同一款SoC不用做任何的改动，就可以用在不同的Machine中。实际上，把Platform认为是某个SoC更好理解。

这一部分只关心CPU本身，不关心Codec。主要处理两个问题：DMA引擎和SoC集成的PCM、I2S或AC '97数字接口控制。主要作用是完成音频数据的管理，最终通过CPU的数字音频接口（DAI）把音频数据传送给Codec进行处理，最终由Codec输出驱动耳机或者是喇叭的音信信号。在具体实现上，ASoC有把Platform驱动分为两个部分：snd_soc_platform_driver和snd_soc_dai_driver。其中，platform_driver负责管理音频数据，把音频数据通过dma或其他操作传送至cpu dai中，dai_driver则主要完成cpu一侧的dai的参数配置，同时也会通过一定的途径把必要的dma等参数与snd_soc_platform_driver进行交互。

##### 2.6.2.3、**Codec**
用于实现平台无关的功能，如寄存器读写接口，音频接口，各widgets的控制接口和DAPM的实现等

> 字面上的意思就是编解码器，Codec里面包含了I2S接口、D/A、A/D、Mixer、PA（功放），通常包含多种输入（Mic、Line-in、I2S、PCM）和多个输出（耳机、喇叭、听筒，Line-out），Codec和Platform一样，是可重用的部件，同一个Codec可以被不同的Machine使用。嵌入式Codec通常通过I2C对内部的寄存器进行控制。

这一部分只关心Codec本身，与CPU平台相关的特性不由此部分操作。在移动设备中，Codec的作用可以归结为4种，分别是：

1、对PCM等信号进行D/A转换，把数字的音频信号转换为模拟信号。
2、对Mic、Linein或者其他输入源的模拟信号进行A/D转换，把模拟的声音信号转变CPU能够处理的数字信号。
3、对音频通路进行控制，比如播放音乐，收听调频收音机，又或者接听电话时，音频信号在codec内的流通路线是不一样的。
4、对音频信号做出相应的处理，例如音量控制，功率放大，EQ控制等等。

ASoC对Codec的这些功能都定义好了一些列相应的接口，以方便地对Codec进行控制。ASoC对Codec驱动的一个基本要求是：驱动程序的代码必须要做到平台无关性，以方便同一个Codec的代码不经修改即可用在不同的平台上。

![enter image description here](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/personal.website/Audio-system-asoc-pcm-control.png) 
ASoC对于Alsa来说，就是分别注册PCM/CONTROL类型的snd_device设备，并实现相应的操作方法集。图中DAI是数字音频接口，用于配置音频数据格式等。

☁ Codec驱动向ASoC注册snd_soc_codec和snd_soc_dai设备。
☁ Platform驱动向ASoC注册snd_soc_platform和snd_soc_dai设备。
☁ Machine驱动通过snd_soc_dai_link绑定codec/dai/platform。

Widget是各个组件内部的小单元。处在活动通路上电，不在活动通路下电。ASoC的DAPM正是通过控制这些Widget的上下电达到动态电源管理的效果。

☁ path描述与其它widget的连接关系。
☁ event用于通知该widget的上下电状态。
☁ power指示当前的上电状态。
☁ control实现空间用户接口用于控制widget的音量/通路切换等。

对驱动开者来说，就可以很好的解耦了：

☁ codec驱动的开发者，实现codec的IO读写方法，描述DAI支持的数据格式/操作方法和Widget的连接关系就可以了;
☁ soc芯片的驱动开发者，Platform实现snd_pcm的操作方法集和DAI的配置如操作 DMA，I2S/AC97/PCM的设定等;
☁ 板级的开发者，描述Machine上codec与platform之间的总线连接， earphone/Speaker的布线情况就可以了。

##### 2.6.3、**DAPM**

 DAPM是Dynamic Audio Power Management的缩写，直译过来就是动态音频电源管理的意思，DAPM是为了使基于linux的移动设备上的音频子系统，在任何时候都工作在最小功耗状态下。DAPM对用户空间的应用程序来说是透明的，所有与电源相关的开关都在ASoc core中完成。用户空间的应用程序无需对代码做出修改，也无需重新编译，DAPM根据当前激活的音频流（playback/capture）和声卡中的mixer等的配置来决定那些音频控件的电源开关被打开或关闭。

##### 2.6.4、**DPCM**
[Dynamic PCM](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/plain/Documentation/sound/soc/dpcm.rst?h=v4.16-rc5)

##### 2.7、Audio devices
具体的Audio硬件设备。




#### （三）、参考资料(特别感谢各位前辈的分析和图示)：
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
[Android Media Player 框架分析-Nuplayer - CSDN博客](https://blog.csdn.net/harman_zjc/article/details/53386010)
[Android Media Player 框架分析-AHandler AMessage ALooper - CSDN博客](https://blog.csdn.net/harman_zjc/article/details/53397945)
[Android N Audio播放 start真面目- (六篇) CSDN博客](https://blog.csdn.net/liu1314you/article/category/6344583)
[深入理解Android音视频同步机制（五篇）NuPlayer的avsync逻辑 - CSDN博客](https://blog.csdn.net/nonmarking/article/details/78746671)
[wangyf的专栏 - CSDN博客-MT6737 Android N 平台 Audio系统学习](https://blog.csdn.net/u014310046/article/category/6571854)
[Android 7.0 Audio: Mediaplayer - CSDN博客](https://blog.csdn.net/xiashaohua/article/details/53638780)
[Android 7.0 Audio-相关类浅析- CSDN博客](https://blog.csdn.net/xiashaohua/article/category/6549668)
[Android N Audio播放六：如何读取buffer - CSDN博客](https://blog.csdn.net/liu1314you/article/details/59119144)
[Fuchsia OS中的RPC机制-FIDL - CSDN博客](https://blog.csdn.net/jinzhuojun/article/details/78007568)
[高通Audio中ASOC的codec驱动 - yooooooo - 博客园](http://www.cnblogs.com/linhaostudy/category/1145404.html)
