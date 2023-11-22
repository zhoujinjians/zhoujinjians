---
title: Android Video System（4）：Android Multimedia - OpenMax实现分析
cover: https://raw.githubusercontent.com/zhoujinjian777/PicGo/master/hexo.themes/bing-wallpaper-2018.04.25.jpg
categories:
  - Multimedia
tags:
  - Android
  - Video
  - Multimedia
toc: true
abbrlink: 20190211
date: 2019-02-11 09:25:00
---



--------------------------------------------------------------------------------

注：文章都是通过阅读各位前辈总结的资料 Android 7.1.2 && Linux（kernel 3.18）Qualcomm平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除，禁止转载（©Qualcomm Technologies, Inc. 版权所有），谢谢。

[【特别感谢 -  Copy Windrunnerlihuan（OpenMax分析）】](http://windrunnerlihuan.com/categories/Android%E6%8A%80%E6%9C%AF%E7%82%B9/page/2/)
[【特别感谢 -  Android MediaCodec ACodec】](https://blog.csdn.net/dfhuang09/article/details/60132620)
[【特别感谢 -  android ACodec MediaCodec NuPlayer flow】](https://blog.csdn.net/dfhuang09/article/details/54926526)


Google Pixel、Pixel XL 内核代码（文章基于 Kernel-3.18）：
 [Kernel source for Pixel and Pixel XL - GitHub](https://github.com/matthewdalex/marlin)

AOSP 源码（文章基于 Android 7.1.2）：
 [ Android 系统全套源代码分享 (更新到 8.1.0_r1)](https://testerhome.com/topics/2229)

--------------------------------------------------------------------------------
☯ \hardware\qcom\media\msm8996\libstagefrighthw
- QComOMXPlugin.cpp
- QComOMXMetadata.h
- QComOMXPlugin.h

☯  \hardware\qcom\media\msm8996\mm-video-v4l2\vidc

☯ \hardware\qcom\media\msm8996\mm-core

--------------------------------------------------------------------------------
#### （一）、OpenMax简介
  android中的 NuPlayer就是用OpenMax来做(codec)编解码的，本节就主要科普一下OpenMax和它在Android系统中扮演的角色。
##### 1.1、 OpenMax系统的结构
###### 1.1.1、OpenMax总体层次结构
 OpenMax是一个多媒体应用程序的框架标准，由NVIDIA公司和Khronos在2006年推出。

 OpenMax是无授权费的，跨平台的应用程序接口API，通过使媒体加速组件能够在开发、集成和编程环节中实现跨多操作系统和处理器硬件平台，提供全面的流媒体编解码器和应用程序便携化。

OpenMax的官方网站如下所示：
       http://www.khronos.org/openmax/
OpenMax实际上分成三个层次，自上而下分别是，OpenMax DL（开发层），OpenMax IL（集成层）和OpenMax AL（应用层）。三个层次的内容分别如下所示：

> 第一层：OpenMax DL（Development Layer，开发层）
>         OpenMax DL定义了一个API，它是音频、视频和图像功能的集合。供应商能够在一个新的处理器上实现并优化，然后编解码供应商使用它来编写更广泛的编解码器功能。它包括音频信号的处理功能，如FFT和filter，图像原始处理，如颜色空间转换、视频原始处理，以实现例如MPEG-4、H.264、MP3、AAC和JPEG等编解码器的优化。
> 
> 
> 第二层：OpenMax IL（Integration Layer，集成层）
>        OpenMax IL作为音频、视频和图像编解码器能与多媒体编解码器交互，并以统一的行为支持组件（例如，资源和皮肤）。这些编解码器或许是软硬件的混合体，对用户是透明的底层接口应用于嵌入式、移动设备。它提供了应用程序和媒体框架，透明的。编解码器供应商必须写私有的或者封闭的接口，集成进移动设备。IL的主要目的是使用特征集合为编解码器提供一个系统抽象，为解决多个不同媒体系统之间轻便性的问题。
> 
> 第三层：OpenMax AL（Appliction Layer，应用层）
>        OpenMax AL API在应用程序和多媒体中间件之间提供了一个标准化接口，多媒体中间件提供服务以实现被期待的API功能。

![Alt text | center](https://raw.githubusercontent.com/zhoujinjian777/PicGo/master/video.system/VS-04-01-openmaxsystem.png)

    OpenMax API将会与处理器一同提供，以**使库和编解码器开发者能够高速有效地利用新器件的完整加速潜能，无须担心其底层的硬件结构。**该标准是针对嵌入式设备和移动设备的多媒体软件架构。在架构底层上为多媒体的编解码和数据处理定义了一套统一的编程接口，对多媒体数据的处理功能进行系统级抽象，为用户屏蔽了底层的细节。因此，**多媒体应用程序和多媒体框架通过OpenMax IL可以以一种统一的方式来使用编解码和其他多媒体数据处理功能，具有了跨越软硬件平台的移植性。**

注：在实际的应用中，OpenMax的三个层次中使用较多的是OpenMax IL集成层，由于操作系统到硬件的差异和多媒体应用的差异，OpenMax的DL和AL层使用相对较少。

###### 1.1.2、OpenMax IL简介
OpenMax IL 处在中间层的位置，OpenMAX IL 作为音频，视频和图像编解码器 能与多媒体编解码器交互，并以统一的行为支持组件（例如资源和皮肤）。这些编解码器或许是软硬件的混合体，对用户是 的底层接口应用于嵌入式或 / 和移动设备。它提供了应用程序和媒体框架， 透明的。本质上不存在这种标准化的接口，编解码器供 应商必须写私有的或者封闭的接口，集成进移动设备。 IL 的主要目的 是使用特征集合为编解码器提供一个系统抽象，为解决多个不同媒体系统之间轻便性的问题。
       OpenMax IL 的目的就是为硬件平台的图形及音视频提供一个抽象层，可以为上层的应用提供一个可跨平台的支撑。这一点对于跨平台的媒体应用来说十分重要。本人也接触过几家高清解码芯片，这些芯片底层的音视频接口虽然功能上大致相同，但是接口设计及用法上各有不同，而且相差很多。你要想让自己开发的媒体应用完美的运行在不同的硬件厂商平台上，就得适应不同芯片的底层解码接口。这个对于应用开发来说十分繁琐。所以就需要类似于OpenMax IL 这种接口规范。应用假如涉及到音视频相关功能时，只需调用这些标准的接口，而不需要关心接口下方硬件相关的实现。假如换了硬件平台时，只需要把接口层与硬件适配好了就行了。上层应用不需要频繁改动。
       你可以把OpenMax IL 看作是中间件中的porting层接口，但是现在中间件大部分都是自家定义自己的。

OpenMax 想做的就是定义一个这样的行业标准，这样媒体应用、硬件厂商都遵循这种标准。硬件厂商将OpenMax 与处理器一并提供，上层的多媒体框架想要用到硬件音视频加速功能时，只需遵循openmax的接口就可以扩平台运行。
       可喜的，现在越来越多的多媒体框架及多媒体应用正在遵循openmax标准，包括各种知名的媒体开源软件。越来越多的芯片厂商也在遵循openmax的标准。对于现在的音视频编解码来说，分辨率越来越高，需要芯片提供硬件加速功能是个大的趋势。我相信 接口的标准化是一定要走的。如下图所示， openmax IL在多媒体框架中的应用：

![Alt text | center](https://raw.githubusercontent.com/zhoujinjian777/PicGo/master/video.system/VS-04-02-openmaxuse.png)



**OpenMax IL目前已经成为了事实上的多媒体框架标准。**嵌入式处理器或者多媒体编解码模块的硬件生产者，通常提供标准的OpenMax IL层的软件接口，这样软件的开发者就可以基于这个层次的标准化接口进行多媒体程序的开发。

 OpenMax IL的接口层次结构适中，既不是硬件编解码的接口，也不是应用程序层的接口，因此比较容易实现标准化。OpenMax IL的层次结构如下：

![Alt text | center](https://raw.githubusercontent.com/zhoujinjian777/PicGo/master/video.system/VS-04-03-openmaxIL.png)


图中的虚线中的内容是OpenMax IL层的内容，其主要实现了OpenMax IL中的各个组件（Component）。**对下层，OpenMax IL可以调用OpenMax DL层的接口，也可以直接调用各种Codec实现。对上层，OpenMax IL可以给OpenMax AL 层等框架层（Middleware）调用，也可以给应用程序直接调用。**

###### 1.1.3、OpenMax IL结构
OpenMax IL主要内容如下所示。

☯ 客户端（Client）：OpenMax IL的调用者
☯ 组件（Component）：OpenMax IL的单元，每一个组件实现一种功能
☯ 端口（Port）：组件的输入输出接口
☯ 隧道化（Tunneled）：让两个组件直接连接的方式
       OpenMax IL的基本运作过程如图所示：
	   
![Alt text | center](https://raw.githubusercontent.com/zhoujinjian777/PicGo/master/video.system/VS-04-04-openmaxilbase.png)

OpenMAL IL的客户端，通过调用四个OpenMAL IL组件，实现了一个功能。四个组件分别是Source组件、Host组件、Accelerator组件和Sink组件。Source组件只有一个输出端口；而Host组件有一个输入端口和一个输出端口；Accelerator组件具有一个输入端口，调用了硬件的编解码器，加速主要体现在这个环节上。Accelerator组件和Sink组件通过私有通讯方式在内部进行连接，没有经过明确的组件端口。
       OpenMAL IL在使用的时候，其数据流也有不同的处理方式：既可以经由客户端，也可以不经由客户端。图中Source组件到Host组件的数据流就是经过客户端的；而Host组件到Accelerator组件的数据流就没有经过客户端，使用了隧道化的方式；Accelerator组件和Sink组件甚至可以使用私有的通讯方式。

   OpenMax Core是辅助各个组件运行的部分，它通常需要完成各个组件的初始化等工作，在真正运行过程中，重点是各个OpenMax IL的组件，OpenMax Core不是重点，也不是标准。

 OpenMAL IL的组件是OpenMax IL实现的核心内容，一个组件以输入、输出端口为接口，端口可以被连接到另一个组件上。外部对组件可以发送命令，还进行设置/获取参数、配置等内容。组件的端口可以包含缓冲区（Buffer）的队列。

组件的处理的核心内容是：通过输入端口消耗Buffer，通过输出端口填充Buffer，由此多组件相联接可以构成流式的处理。OpenMAL IL中一个组件的结构如下图所示：

![Alt text | center](https://raw.githubusercontent.com/zhoujinjian777/PicGo/master/video.system/VS-04-05-openilinternal.jpg)


       组件的功能和其定义的端口类型密切相关，通常情况下：只有一个输出端口的，为Source组件；只有一个输入端口的，为Sink组件；有多个输入端口，一个输出端口的为Mux组件；有一个输入端口，多个输出端口的为DeMux组件；输入输出端口各一个组件的为中间处理环节，这是最常见的组件。

  端口具体支持的数据也有不同的类型。例如，对于一个输入、输出端口各一个组件，其输入端口使用MP3格式的数据，输出端口使用PCM格式的数据，那么这个组件就是一个MP3解码组件。

  隧道化（Tunneled）是一个关于组件连接方式的概念。通过隧道化可以将不同的组件的一个输入端口和一个输出端口连接到一起，在这种情况下，两个组件的处理过程合并，共同处理。尤其对于单输入和单输出的组件，两个组件将作为类似一个使用。
##### 1.2、Android中的OpenMax

###### 1.2.1、OpenMax在Android中的使用情况
  在Android中，OpenMax IL层，**通常可以用于多媒体引擎的插件**，Android的多媒体引擎OpenCore和StageFright都可以使用OpenMax作为插件，主要用于**编解码（Codec）**处理。
  在Android的框架层，也定义了由Android封装的OpenMax接口，和标准的接口概念基本相同，但是使用C++类型的接口，并且使用了Android的Binder IPC机制。Android封装OpenMax的接口被StageFright使用，OpenCore没有使用这个接口，而是使用其他形式对OpenMax IL层接口进行封装。Android OpenMax的基本层次结构如图：

![Alt text | center](https://raw.githubusercontent.com/zhoujinjian777/PicGo/master/video.system/VS-04-06-openmaxinandroid.jpg)


       Android系统的一些部分对OpenMax IL层进行使用，基本使用的是标准OpenMax IL层的接口，只是进行了简单的封装。标准的OpenMax IL实现很容易以插件的形式加入到Android系统中。
       Android的多媒体引擎OpenCore和StageFright都可以使用OpenMax作为多媒体编解码的插件，只是没有直接使用OpenMax IL层提供的纯C接口，而是对其进行了一定的封装(C++封装)。
       在Android2.x版本之后，Android的框架层也对OpenMax IL层的接口进行了封装定义，甚至使用Android中的Binder IPC机制。Stagefright使用了这个层次的接口，OpenCore没有使用。
       注：OpenCore使用OpenMax IL层作为编解码插件在前，Android框架层封装OpenMax接口在后面的版本中才引入。

###### 1.2.2、Android OpenMax实现的内容
   android中的 AwesomePlayer就是用openmax来做(Codec)编解码,其实在openmax接口设计中，他不光能用来当编解码。通过他的组件可以组成一个完整的播放器，包括sourc、demux、decode、output。但是为什么android只用他来做code呢？应该有如下方面：

☯    1.在整个播放器中，解码器不得不说是最重要的一部分，而且也是最耗资源的一块。如果全靠软解，直接通过cpu来运算，特别是高清视频。别的事你就可以啥都不干了。所以解码器是最需要硬件提供加速的部分。现在的高清解码芯片都是主芯片+DSP结构，解码的工作都是通过DSP来做，不会在过多的占用主芯片。所有将芯片中DSP硬件编解码的能力通过openmax标准接口呈现出来，提供上层播放器来用。我认为这块是openmax最重要的意义。
☯    2.source 主要是和协议打交道，demux 分解容器部分，大多数的容器格式的分解是不需要通过硬件来支持。只是ts流这种格式最可能用到硬件的支持。因为ts格式比较特殊，单包的大小太小了，只有188字节。所以也是为什么现在常见的解码芯片都会提供硬件ts demux 的支持。
☯   3.音视频输出部分video\audio output 这块和操作系统关系十分紧密。可以看看著名开源播放器vlc。vlc 在mac、linux、Windows都有，功能上差别也不大。所以说他是跨平台的，他跨平台跨在哪？主要的工作量还是在音视频解码完之后的输出模块。因为各个系统的图像渲染和音频输出实现方法不同，所以vlc需要针对每个平台实现不同的output。这部分内容放在openmax来显然不合适。
       Android中使用的主要是OpenMax的编解码功能。虽然OpenMax也可以生成输入、输出、文件解析-构建等组件，但是在各个系统（不仅是Android）中使用的最多的还是编解码组件。媒体的输入、输出环节和系统的关系很大，引入OpenMax标准比较麻烦；文件解析-构建环节一般不需要使用硬件加速。编解码组件也是最能体现硬件加速的环节，因此最常使用。

##### 1.3、初窥适配层接口
在Android中实现OpenMax IL层和标准的OpenMax IL层的方式基本，一般需要实现以下两个环节：

☯    编解码驱动程序：位于Linux内核空间，需要通过Linux内核调用驱动程序，通常使用非标准的驱动程序。
☯   OpenMax IL层：根据OpenMax IL层的标准头文件实现不同功能的组件。
 **Android中还提供了OpenMax的适配层接口（对OpenMax IL的标准组件进行封装适配）**，它作为Android本地层的接口，可以被Android的多媒体引擎调用。上一篇文章末尾，初始化解码器核心调用的两个方法就是适配层的接口。

![Alt text | center](https://raw.githubusercontent.com/zhoujinjian777/PicGo/master/video.system/VS-04-07-openmaxsult.jpg)


☯    1.上面已经说过了，android系统中只用openmax来做Codec，所以android向上抽象了一层OMXCodec，提供给上层播放器用。播放器中音视频解码器mVideosource、mAudiosource都是OMXCodec的实例。
☯    2.OMXCodec通过IOMX 依赖binder机制 获得 OMX服务，OMX服务 才是openmax 在android中 实现。
☯    3.OMX把软编解码和硬件编解码统一看作插件的形式管理起来。

#### （二）、Android中OpenMax的实现(preview)
##### 2.1、OpenMax的接口与实现

在Android中实现OpenMax IL层和标准的OpenMax IL层的方式基本，一般需要实现以下两个环节。

编解码驱动程序：位于Linux内核空间，需要通过Linux内核调用驱动程序，通常使用非标准的驱动程序。
OpenMax IL层：根据OpenMax IL层的标准头文件实现不同功能的组件。
       Android中还提供了OpenMax的适配层接口（对OpenMax IL的标准组件进行封装适配），它作为Android本地层的接口，可以被Android的多媒体引擎调用。

OpenMax IL层接口
       OpenMax IL层的接口定义由若干个头文件组成，这也是实现它需要实现的内容，位于frameworks/native/include/media/openmax下，它们的基本描述如下所示：

> OMX_Types.h：OpenMax Il的数据类型定义
OMX_Core.h：OpenMax IL核心的API
OMX_Component.h：OpenMax IL 组件相关的 API
OMX_Audio.h：音频相关的常量和数据结构
OMX_IVCommon.h：图像和视频公共的常量和数据结构
OMX_Image.h：图像相关的常量和数据结构
OMX_Video.h：视频相关的常量和数据结构
OMX_Other.h：其他数据结构（包括A/V 同步）
OMX_Index.h：OpenMax IL定义的数据结构索引
OMX_ContentPipe.h：内容的管道定义

   **提示：OpenMax标准只有头文件，没有标准的库，设置没有定义函数接口。对于实现者，需要实现的主要是包含函数指针的结构体。**

其中，OMX_Component.h中定义的OMX_COMPONENTTYPE结构体是OpenMax IL层的核心内容，表示一个组件，其内容如下所示：

``` cpp
[->frameworks/native/include/media/openmax/OMX_Component.h]
typedef struct OMX_COMPONENTTYPE    
{  
    OMX_U32 nSize;                          /* 这个结构体的大小 */    
    OMX_VERSIONTYPE nVersion;           /* 版本号 */    
    OMX_PTR pComponentPrivate;          /* 这个组件的私有数据指针. */    
  
    /* 调用者（IL client）设置的指针，用于保存它的私有数据，传回给所有的回调函数 */    
    OMX_PTR pApplicationPrivate;    
    /* 以下的函数指针返回OMX_core.h中的对应内容 */    
    OMX_ERRORTYPE (*GetComponentVersion)(/* 获得组件的版本*/    
        OMX_IN  OMX_HANDLETYPE hComponent,    
        OMX_OUT OMX_STRING pComponentName,    
        OMX_OUT OMX_VERSIONTYPE* pComponentVersion,    
        OMX_OUT OMX_VERSIONTYPE* pSpecVersion,    
        OMX_OUT OMX_UUIDTYPE* pComponentUUID);    
    OMX_ERRORTYPE (*SendCommand)(/* 发送命令 */    
        OMX_IN  OMX_HANDLETYPE hComponent,    
        OMX_IN  OMX_COMMANDTYPE Cmd,    
        OMX_IN  OMX_U32 nParam1,    
        OMX_IN  OMX_PTR pCmdData);    
    OMX_ERRORTYPE (*GetParameter)(/* 获得参数 */    
        OMX_IN  OMX_HANDLETYPE hComponent,    
        OMX_IN  OMX_INDEXTYPE nParamIndex,    
        OMX_INOUT OMX_PTR pComponentParameterStructure);    
    OMX_ERRORTYPE (*SetParameter)(/* 设置参数 */    
        OMX_IN  OMX_HANDLETYPE hComponent,    
        OMX_IN  OMX_INDEXTYPE nIndex,    
        OMX_IN  OMX_PTR pComponentParameterStructure);    
    OMX_ERRORTYPE (*GetConfig)(/* 获得配置 */    
        OMX_IN  OMX_HANDLETYPE hComponent,    
        OMX_IN  OMX_INDEXTYPE nIndex,    
        OMX_INOUT OMX_PTR pComponentConfigStructure);    
    OMX_ERRORTYPE (*SetConfig)(/* 设置配置 */    
        OMX_IN  OMX_HANDLETYPE hComponent,    
        OMX_IN  OMX_INDEXTYPE nIndex,    
        OMX_IN  OMX_PTR pComponentConfigStructure);    
    OMX_ERRORTYPE (*GetExtensionIndex)(/* 转换成OMX结构的索引 */    
        OMX_IN  OMX_HANDLETYPE hComponent,    
        OMX_IN  OMX_STRING cParameterName,    
        OMX_OUT OMX_INDEXTYPE* pIndexType);    
    OMX_ERRORTYPE (*GetState)(/* 获得组件当前的状态 */    
        OMX_IN  OMX_HANDLETYPE hComponent,    
        OMX_OUT OMX_STATETYPE* pState);    
    OMX_ERRORTYPE (*ComponentTunnelRequest)(/* 用于连接到另一个组件*/    
        OMX_IN  OMX_HANDLETYPE hComp,    
        OMX_IN  OMX_U32 nPort,    
        OMX_IN  OMX_HANDLETYPE hTunneledComp,    
        OMX_IN  OMX_U32 nTunneledPort,    
        OMX_INOUT  OMX_TUNNELSETUPTYPE* pTunnelSetup);    
    OMX_ERRORTYPE (*UseBuffer)(/* 为某个端口使用Buffer */    
        OMX_IN OMX_HANDLETYPE hComponent,    
        OMX_INOUT OMX_BUFFERHEADERTYPE** ppBufferHdr,    
        OMX_IN OMX_U32 nPortIndex,    
        OMX_IN OMX_PTR pAppPrivate,    
        OMX_IN OMX_U32 nSizeBytes,    
        OMX_IN OMX_U8* pBuffer);    
    OMX_ERRORTYPE (*AllocateBuffer)(/* 在某个端口分配Buffer */    
        OMX_IN OMX_HANDLETYPE hComponent,    
        OMX_INOUT OMX_BUFFERHEADERTYPE** ppBuffer,    
        OMX_IN OMX_U32 nPortIndex,    
        OMX_IN OMX_PTR pAppPrivate,    
        OMX_IN OMX_U32 nSizeBytes);    
    OMX_ERRORTYPE (*FreeBuffer)(/*将某个端口Buffer释放*/    
        OMX_IN  OMX_HANDLETYPE hComponent,    
        OMX_IN  OMX_U32 nPortIndex,    
        OMX_IN  OMX_BUFFERHEADERTYPE* pBuffer);    
    OMX_ERRORTYPE (*EmptyThisBuffer)(/* 让组件消耗这个Buffer */    
        OMX_IN  OMX_HANDLETYPE hComponent,    
        OMX_IN  OMX_BUFFERHEADERTYPE* pBuffer);    
    OMX_ERRORTYPE (*FillThisBuffer)(/* 让组件填充这个Buffer */    
        OMX_IN  OMX_HANDLETYPE hComponent,    
        OMX_IN  OMX_BUFFERHEADERTYPE* pBuffer);    
    OMX_ERRORTYPE (*SetCallbacks)(/* 设置回调函数 */    
        OMX_IN  OMX_HANDLETYPE hComponent,    
        OMX_IN  OMX_CALLBACKTYPE* pCallbacks,    
        OMX_IN  OMX_PTR pAppData);    
    OMX_ERRORTYPE (*ComponentDeInit)(/* 反初始化组件 */    
        OMX_IN  OMX_HANDLETYPE hComponent);    
    OMX_ERRORTYPE (*UseEGLImage)(    
        OMX_IN OMX_HANDLETYPE hComponent,    
        OMX_INOUT OMX_BUFFERHEADERTYPE** ppBufferHdr,    
        OMX_IN OMX_U32 nPortIndex,    
        OMX_IN OMX_PTR pAppPrivate,    
        OMX_IN void* eglImage);    
    OMX_ERRORTYPE (*ComponentRoleEnum)(    
        OMX_IN OMX_HANDLETYPE hComponent,    
        OMX_OUT OMX_U8 *cRole,    
        OMX_IN OMX_U32 nIndex);    
} OMX_COMPONENTTYPE;
```

☯    1）EmptyThisBuffer和FillThisBuffer是驱动组件运行的基本的机制，前者表示让组件消耗缓冲区，表示对应组件输入的内容；后者表示让组件填充缓冲区，表示对应组件输出的内容。
☯    2）UseBuffer，AllocateBuffer，FreeBuffer为和端口相关的缓冲区管理函数，对于组件的端口有些可以自己分配缓冲区，有些可以使用外部的缓冲区，因此有不同的接口对其进行操作。
☯    3）SendCommand表示向组件发送控制类的命令。GetParameter，SetParameter，GetConfig，SetConfig几个接口用于辅助的参数和配置的设置和获取。
☯     4）ComponentTunnelRequest用于组件之间的隧道化连接，其中需要制定两个组件及其相连的端口。
☯     5）ComponentDeInit用于组件的反初始化。

OMX_COMPONENTTYPE结构体实现后，其中的各个函数指针就是调用者可以使用的内容。各个函数指针和OMX_core.h中定义的内容相对应。

提示：OpenMax函数的参数中，经常包含OMX_IN和OMX_OUT等宏，它们的实际内容为空，只是为了标记参数的方向是输入还是输出。

OMX_Component.h中端口类型的定义为OMX_PORTDOMAINTYPE枚举类型，内容如下所示：

``` cpp
typedef enum OMX_PORTDOMAINTYPE {   
    OMX_PortDomainAudio,        /* 音频类型端口 */   
    OMX_PortDomainVideo,        /* 视频类型端口 */   
    OMX_PortDomainImage,        /* 图像类型端口 */   
    OMX_PortDomainOther,        /* 其他类型端口 */   
    OMX_PortDomainKhronosExtensions = 0x6F000000,   //为Khronos标准预留宽展
    OMX_PortDomainVendorStartUnused = 0x7F000000    //为厂商预留扩展
    OMX_PortDomainMax = 0x7ffffff  
} OMX_PORTDOMAINTY
```
音频类型，视频类型，图像类型，其他类型是OpenMax IL层此所定义的四种端口的类型。

端口具体内容的定义使用OMX_PARAM_PORTDEFINITIONTYPE类（也在OMX_Component.h中定义）来表示，其内容如下所示：

``` cpp
typedef struct OMX_PARAM_PORTDEFINITIONTYPE {   
    OMX_U32 nSize;                      /* 结构体大小 */   
    OMX_VERSIONTYPE nVersion;           /* 版本*/   
    OMX_U32 nPortIndex;             /* 端口号 */   
    OMX_DIRTYPE eDir;                   /* 端口的方向 */   
    OMX_U32 nBufferCountActual;         /* 为这个端口实际分配的Buffer的数目 */   
    OMX_U32 nBufferCountMin;            /* 这个端口最小Buffer的数目*/   
    OMX_U32 nBufferSize;                /* 缓冲区的字节数 */   
    OMX_BOOL bEnabled;                  /* 是否使能 */   
    OMX_BOOL bPopulated;                /* 是否在填充 */   
    OMX_PORTDOMAINTYPE eDomain;         /* 端口的类型 */   
    union {                         /* 端口实际的内容，由类型确定具体结构 */   
        OMX_AUDIO_PORTDEFINITIONTYPE audio;   
        OMX_VIDEO_PORTDEFINITIONTYPE video;   
        OMX_IMAGE_PORTDEFINITIONTYPE image;   
        OMX_OTHER_PORTDEFINITIONTYPE other;   
    } format;   
    OMX_BOOL bBuffersContiguous;   
    OMX_U32 nBufferAlignment;   
} OMX_PARAM_PORTDEFINITIONTYPE;
```
对于一个端口，其重点的内容如下:

☯    端口的方向（OMX_DIRTYPE）：包含OMX_DirInput（输入）和OMX_DirOutput（输出）两种
☯    端口分配的缓冲区数目和最小缓冲区数目
☯    端口的类型（OMX_PORTDOMAINTYPE）：可以是四种类型
☯    端口格式的数据结构：使用format联合体来表示，具体由四种不同类型来表示，与端口的类型相对应
☯    OMX_AUDIO_PORTDEFINITIONTYPE，OMX_VIDEO_PORTDEFINITIONTYPE，OMX_IMAGE_PORTDEFINITIONTYPE和OMX_OTHER_PORTDEFINITIONTYPE等几个具体的格式类型，分别在OMX_Audio.h，OMX_Video.h，OMX_Image.h和OMX_Other.h这四个头文件中定义。
       OMX_Core.h中定义的枚举类型OMX_STATETYPE命令表示OpenMax的状态机，内容如下所示：
       
``` cpp
typedef enum OMX_STATETYPE   
{   
    OMX_StateInvalid,                   /* 组件监测到内部的数据结构被破坏 */   
    OMX_StateLoaded,                    /* 组件被加载但是没有完成初始化 */   
    OMX_StateIdle,                      /* 组件初始化完成，准备开始 */   
    OMX_StateExecuting,             /* 组件接受了开始命令，正在树立数据 */   
    OMX_StatePause,                     /* 组件接受暂停命令*/   
    OMX_StateWaitForResources,      /* 组件正在等待资源 */   
    OMX_StateKhronosExtensions = 0x6F000000, /* 保留for Khronos */   
    OMX_StateVendorStartUnused = 0x7F000000, /* 保留for厂商 */   
    OMX_StateMax = 0X7FFFFFFF  
} OMX_STATETYPE;
```

OpenMax组件的状态机可以由外部的命令改变，也可以由内部发生的情况改变。OpenMax IL组件的状态机的迁移关系如图所示：

![Alt text | center](https://raw.githubusercontent.com/zhoujinjian777/PicGo/master/video.system/VS-04-08-OMXstate.png)


 OMX_Core.h中定义的枚举类型OMX_COMMANDTYPE表示对组件的命令类型，内容如下所示：

``` cpp
typedef enum OMX_COMMANDTYPE   
{   
    OMX_CommandStateSet,                /* 改变状态机器 */   
    OMX_CommandFlush,                   /* 刷新数据队列 */   
    OMX_CommandPortDisable,             /* 禁止端口 */   
    OMX_CommandPortEnable,              /* 使能端口 */   
    OMX_CommandMarkBuffer,              /* 标记组件或Buffer用于观察 */   
    OMX_CommandKhronosExtensions = 0x6F000000, /* 保留for Khronos */   
    OMX_CommandVendorStartUnused = 0x7F000000, /* 保留for厂商 */   
    OMX_CommandMax = 0X7FFFFFFF  
} OMX_COMMANDTYPE;
```

OMX_COMMANDTYPE类型在SendCommand调用中作为参数被使用，其中OMX_CommandStateSet就是改变状态机的命令。

###### 2.1.2、OpenMax IL实现的内容
  对于OpenMax IL层的实现，一般的方式并不调用OpenMax DL层。具体实现的内容就是各个不同的组件。
       OpenMax IL组件的实现包含以下两个步骤：

☯    组件的初始化函数：硬件和OpenMax数据结构的初始化，一般分成函数指针初始化、私有数据结构的初始化、端口的初始化等几个步骤，使用OMX_Component.h其中的pComponentPrivate成员保留本组件的私有数据为上下文，最后获得填充完成OMX_COMPONENTTYPE类型的结构体。
☯    OMX_COMPONENTTYPE类型结构体的各个指针：实现其中的各个函数指针，需要使用私有数据的时候，从其中的pComponentPrivate得到指针，转化成实际的数据结构使用。

端口的定义是OpenMax IL组件对外部的接口。OpenMax IL常用的组件大都是输入和输出端口各一个。对于最常用的编解码（Codec）组件，通常需要在每个组件的实现过程中，调用硬件的编解码接口来实现。在组件的内部处理中，可以建立线程来处理。OpenMax的组件的端口有默认参数，但也可以在运行时设置，因此一个端口也可以支持不同的编码格式。音频编码组件的输出和音频编码组件的输入通常是原始数据格式（PCM格式），视频编码组件的输出和视频编码组件的输入通常是原始数据格式（YUV格式）。
       提示：在一种特定的硬件实现中，编解码部分具有相似性，因此通常可以构建一个OpenMax组件的”基类”或者公共函数，来完成公共性的操作。


##### 2.2、Android中OpenMax的适配层
  Android中的OpenMax适配层的接口在frameworks/av/include/media/IOMX.h文件定义，其内容如下所示：

``` cpp
class IOMX : public IInterface {    
public:    
    DECLARE_META_INTERFACE(OMX);    
    typedef void *buffer_id;    
    typedef void *node_id;    
    virtual bool livesLocally(pid_t pid) = 0;    
    struct ComponentInfo {// 组件的信息    
        String8 mName;    
        List<String8> mRoles;    
    };    
    virtual status_t listNodes(List<ComponentInfo> *list) = 0;  // 节点列表    
    virtual status_t allocateNode(    
        const char *name, const sp<IOMXObserver> &observer,  // 分配节点    
        node_id *node) = 0;    
    virtual status_t freeNode(node_id node) = 0; // 找到节点    
    virtual status_t sendCommand(// 发送命令    
        node_id node, OMX_COMMANDTYPE cmd, OMX_S32 param) = 0;    
    virtual status_t getParameter(// 获得参数    
        node_id node, OMX_INDEXTYPE index,    
        void *params, size_t size) = 0;    
    virtual status_t setParameter(// 设置参数    
        node_id node, OMX_INDEXTYPE index,    
        const void *params, size_t size) = 0;    
    virtual status_t getConfig(// 获得配置    
        node_id node, OMX_INDEXTYPE index,    
        void *params, size_t size) = 0;    
    virtual status_t setConfig(// 设置配置    
        node_id node, OMX_INDEXTYPE index,    
        const void *params, size_t size) = 0;   
    virtual status_t useBuffer(// 使用缓冲区    
        node_id node, OMX_U32 port_index, const sp<IMemory> ¶ms,    
        buffer_id *buffer) = 0;    
    virtual status_t allocateBuffer(// 分配缓冲区    
        node_id node, OMX_U32 port_index, size_t size,    
        buffer_id *buffer, void **buffer_data) = 0;    
    virtual status_t allocateBufferWithBackup(// 分配带后备缓冲区    
        node_id node, OMX_U32 port_index, const sp<IMemory> ¶ms,    
        buffer_id *buffer) = 0;    
    virtual status_t freeBuffer(// 释放缓冲区    
        node_id node, OMX_U32 port_index, buffer_id buffer) = 0;    
    virtual status_t fillBuffer(node_id node, buffer_id buffer) = 0; // 填充缓冲区    
    virtual status_t emptyBuffer(// 消耗缓冲区    
        node_id node,    
        buffer_id buffer,    
        OMX_U32 range_offset, OMX_U32 range_length,    
        OMX_U32 flags, OMX_TICKS timestamp) = 0;    
    virtual status_t getExtensionIndex(    
        node_id node,    
        const char *parameter_name,    
        OMX_INDEXTYPE *index) = 0;    
    virtual sp<IOMXRenderer> createRenderer(// 创建渲染器（从ISurface）    
        const sp<ISurface> &surface,    
        const char *componentName,    
        OMX_COLOR_FORMATTYPE colorFormat,    
        size_t encodedWidth, size_t encodedHeight,    
        size_t displayWidth, size_t displayHeight) = 0;    
    
    ......   
};
```
 IOMX表示的是OpenMax的一个组件，根据Android的Binder IPC机制，BnOMX继承IOMX，实现者需要继承实现BnOMX。IOMX类中，有标准的OpenMax的GetParameter，SetParameter，GetConfig，SetConfig，SendCommand，UseBuffer，AllocateBuffer，FreeBuffer，FillThisBuffer和EmptyThisBuffer等接口。
       在IOMX.h文件中，另有表示观察器类的IOMXObserver，这个类表示OpenMax的观察者，其中只包含一个onMessage()函数，其参数为omx_message接口体，其中包含Event事件类型、FillThisBuffer完成和EmptyThisBuffer完成几种类型。
       提示：Android中OpenMax的适配层是OpenMAX IL层至上的封装层，在Android系统中被StageFright调用，也可以被其他部分调用。

#### 2.3、TI(Texas Instruments 德州仪器) OpenMax IL的硬件实现
##### 2.3.1、TI OpenMax IL实现的结构和机制
 Android的开源代码中，已经包含了TI的OpenMax IL层的实现代码，其路径如hardware/ti/omap3/omx下。其中包含的主要目录如下所示：

☯    system：OpenMax核心和公共部分
☯    audio：音频处理部分的OpenMax IL组件
☯    video：视频处理部分OpenMax IL组件
☯    image：图像处理部分OpenMax IL组件
       TI OpenMax IL实现的结构如图所示:

![Alt text | center](https://raw.githubusercontent.com/zhoujinjian777/PicGo/master/video.system/VS-04-09-tiopenmaxil.png)


 在TI OpenMax IL实现中，最上面的内容是OpenMax的管理者用于管理和初始化，中间层是各个编解码单元的OpenMax IL标准组件，下层是LCML层，供各个OpenMax IL标准组件所调用。
       （1）TI OpenMax IL实现的公共部分在system/src/openmax_il/目录中，主要的内容如下所示。

☯ omx_core/src：OpenMax IL的核心，生成动态库libOMX_Core.so
☯ lcml/：LCML的工具库，生成动态库libLCML.so
       （2）I OpenMax IL的视频（Video）相关的组件在video/src/openmax_il/目录中，主要的内容如下所示。

☯ prepost_processor：Video数据的前处理和后处理，生成动态库libOMX.TI.VPP.so
☯ video_decode：Video解码器，生成动态库libOMX.TI.Video.Decoder.so
☯ video_encode：Video编码器，生成动态库libOMX.TI.Video.encoder.so
       （3）TI OpenMax IL的音频（Audio）相关的组件在audio/src/openmax_il/目录中，主要的内容如下所示。

☯ g711_dec：G711解码器，生成动态库libOMX.TI.G711.decode.so
☯ g711_enc：G711编码器，生成动态库libOMX.TI.G711.encode.so
☯ g722_dec：G722解码器，生成动态库libOMX.TI.G722.decode.so
☯ g722_enc：G722编码器，生成动态库libOMX.TI.G722.encode.so
☯ g726_dec：G726解码器，生成动态库libOMX.TI.G726.decode.so
☯ g726_enc：G726编码器，生成动态库libOMX.TI.G726.encode.so
☯ g729_dec：G729解码器，生成动态库libOMX.TI.G729.decode.so
☯ g729_enc：G720编码器，生成动态库libOMX.TI.G729.encode.so
☯ nbamr_dec：AMR窄带解码器，生成动态库libOMX.TI.AMR.decode.so
☯ nbamr_enc：AMR窄带编码器，生成动态库libOMX.TI.AMR.encode.so
☯ wbamr_dec：AMR宽带解码器，生成动态库libOMX.TI.WBAMR.decode.so
☯ wbamr_enc：AMR宽带编码器，生成动态库libOMX.TI.WBAMR.encode.so
☯ mp3_dec：MP3解码器，生成动态库libOMX.TI.MP3.decode.so
☯ aac_dec：AAC解码器，生成动态库libOMX.TI.AAC.decode.so
☯ aac_enc：AAC编码器，生成动态库libOMX.TI.AAC.encode.so
☯ wma_dec：WMA解码器，生成动态库libOMX.TI.WMA.decode.so
       （4）TI OpenMax IL的图像（Image）相关的组件在image/src/openmax_il/目录中，主要的内容如下所示。

☯ jpeg_enc：JPEG编码器，生成动态库libOMX.TI.JPEG.Encoder.so
☯ jpeg_dec：JPEG解码器，生成动态库libOMX.TI.JPEG.decoder.so

##### 2.3.2、TI OpenMax IL的核心和公共内容
 LCML的全称是”Linux Common Multimedia Layer“，是TI的Linux公共多媒体层。在OpenMax IL的实现中，这个内容在system/src/openmax_il/lcml/目录中，主要文件是子目录src中的LCML_DspCodec.c文件。通过调用DSPBridge的内容， 让ARM和DSP进行通信，然DSP进行编解码方面的处理。DSP的运行还需要固件的支持。

 TI OpenMax IL的核心实现在system/src/openmax_il/omx_core/目录中，生成TI OpenMax IL的核心库libOMX_Core.so。

其中子目录src中的OMX_Core.c为主要文件，其中定义了编解码器的名称等，其片断如下所示：

``` cpp
char *tComponentName[MAXCOMP][2] = {    
    {"OMX.TI.JPEG.decoder", "image_decoder.jpeg"},/* 图像和视频编解码器 */    
    {"OMX.TI.JPEG.Encoder", "image_encoder.jpeg"},    
    {"OMX.TI.Video.Decoder", "video_decoder.avc"},    
    {"OMX.TI.Video.Decoder", "video_decoder.mpeg4"},    
    {"OMX.TI.Video.Decoder", "video_decoder.wmv"},    
    {"OMX.TI.Video.encoder", "video_encoder.mpeg4"},    
    {"OMX.TI.Video.encoder", "video_encoder.h263"},    
    {"OMX.TI.Video.encoder", "video_encoder.avc"},    
     /* ......省略 ，语音相关组件*/    
#ifdef BUILD_WITH_TI_AUDIO /* 音频编解码器 */    
    {"OMX.TI.MP3.decode", "audio_decoder.mp3"},    
    {"OMX.TI.AAC.encode", "audio_encoder.aac"},    
    {"OMX.TI.AAC.decode", "audio_decoder.aac"},    
    {"OMX.TI.WMA.decode", "audio_decoder.wma"},    
    {"OMX.TI.WBAMR.decode", "audio_decoder.amrwb"},    
    {"OMX.TI.AMR.decode", "audio_decoder.amrnb"},    
    {"OMX.TI.AMR.encode", "audio_encoder.amrnb"},    
    {"OMX.TI.WBAMR.encode", "audio_encoder.amrwb"},    
#endif    
    {NULL, NULL},    
};
```
  tComponentName数组的各个项中，第一个表示**编解码库内容**，第二个表示**库所实现的功能**。
       其中，TIOMX_GetHandle()函数用于获得各个组件的句柄，其实现的主要片断如下所示：
``` cpp
OMX_ERRORTYPE TIOMX_GetHandle( OMX_HANDLETYPE* pHandle, OMX_STRING cComponentName,    
    OMX_PTR pAppData, OMX_CALLBACKTYPE* pCallBacks)   
{    
    static const char prefix[] = "lib";    
    static const char postfix[] = ".so";    
    OMX_ERRORTYPE (*pComponentInit)(OMX_HANDLETYPE*);    
    OMX_ERRORTYPE err = OMX_ErrorNone;    
    OMX_COMPONENTTYPE *componentType;    
    const char* pErr = dlerror();    
    // ...... 省略错误处理内容    
    int i = 0;    
    for(i=0; i< COUNTOF(pModules); i++) {       // 循环查找    
        if(pModules[i] == NULL) break;    
    }    
    // ...... 省略错误处理内容    
    int refIndex = 0;    
    for (refIndex=0; refIndex < MAX_TABLE_SIZE; refIndex++) {    
    // 循环查找组件列表    
        if (strcmp(componentTable[refIndex].name, cComponentName) == 0) {    
            if (componentTable[refIndex].refCount>= MAX_CONCURRENT_INSTANCES) {    
            // ...... 省略错误处理内容    
            } else {    
                char buf[sizeof(prefix) + MAXNAMESIZE+ sizeof(postfix)];    
                strcpy(buf, prefix);    
                strcat(buf, cComponentName);    
                strcat(buf, postfix);    
                pModules[i] = dlopen(buf, RTLD_LAZY | RTLD_GLOBAL);    
                // ...... 省略错误处理内容    
                // 动态取出初始化的符号    
                pComponentInit = dlsym(pModules[i], "OMX_ComponentInit");    
                pErr = dlerror();    
                // ...... 省略错误处理内容    
                *pHandle = malloc(sizeof(OMX_COMPONENTTYPE));    
                // ...... 省略错误处理内容    
                pComponents[i] = *pHandle;    
                componentType = (OMX_COMPONENTTYPE*) *pHandle;    
                componentType->nSize = sizeof(OMX_COMPONENTTYPE);    
                err = (*pComponentInit)(*pHandle);   // 执行初始化工作    
                // ...... 省略部分内容    
            }    
        }    
    }    
    err = OMX_ErrorComponentNotFound;    
    goto UNLOCK_MUTEX;    
    // ...... 省略部分内容    
     return (err);    
}
```
 在TIOMX_GetHandle()函数中，根据tComponentName数组中动态库的名称，动态打开各个编解码实现的动态库，取出其中的**OMX_ComponentInit**符号来执行各个组件的初始化。

##### 2.3.3、一个TI OpenMax IL组件的实现

  TI OpenMax IL中各个组件都是通过调用LCML来实现的，实现的方式基本类似。主要都是实现了名称为OMX_ComponentInit的初始化函数，实现OMX_COMPONENTTYPE类型的结构体中的各个成员。各个组件其目录结构和文件结构也类似。

以MP3解码器的实现为例，在audio/src/openmax_il/mp3_dec/src目录中，主要包含以下文件：

☯ OMX_Mp3Decoder.c：MP3解码器组件实现
☯ OMX_Mp3Dec_CompThread.c：MP3解码器组件的线程循环
☯ OMX_Mp3Dec_Utils.c：MP3解码器的相关工具，调用LCML实现真正的MP3解码的功能
       OMX_Mp3Decoder.c中的OMX_ComponentInit()函数负责组件的初始化，返回的内容再从参数中得到，这个函数的主要片断如下所示：

``` cpp
OMX_ERRORTYPE OMX_ComponentInit (OMX_HANDLETYPE hComp)    
{    
    OMX_ERRORTYPE eError = OMX_ErrorNone;    
    OMX_COMPONENTTYPE *pHandle = (OMX_COMPONENTTYPE*) hComp;    
    OMX_PARAM_PORTDEFINITIONTYPE *pPortDef_ip = NULL, *pPortDef_op = NULL;    
    OMX_AUDIO_PARAM_PORTFORMATTYPE *pPortFormat = NULL;    
    OMX_AUDIO_PARAM_MP3TYPE *mp3_ip = NULL;    
    OMX_AUDIO_PARAM_PCMMODETYPE *mp3_op = NULL;    
    MP3DEC_COMPONENT_PRIVATE *pComponentPrivate = NULL;    
    MP3D_AUDIODEC_PORT_TYPE *pCompPort = NULL;    
    MP3D_BUFFERLIST *pTemp = NULL;    
    int i=0;    
  
    MP3D_OMX_CONF_CHECK_CMD(pHandle,1,1);    
    /* ......省略，初始化OMX_COMPONENTTYPE类型的指针pHandle */    
    OMX_MALLOC_GENERIC(pHandle->pComponentPrivate, MP3DEC_COMPONENT_PRIVATE);    
    pComponentPrivate = pHandle->pComponentPrivate; /* 私有指针互相指向 */    
    pComponentPrivate->pHandlepHandle = pHandle;    
    /* ......略，初始化似有数据指针pComponentPrivate */    
    /* 设置输入端口（OMX_PARAM_PORTDEFINITIONTYPE类型）的默认值 */    
    pPortDef_ip->nSize                   = sizeof(OMX_PARAM_PORTDEFINITIONTYPE);    
    pPortDef_ip->nPortIndex             = MP3D_INPUT_PORT;    
    pPortDef_ip->eDir                    = OMX_DirInput;    
    pPortDef_ip->nBufferCountActual    = MP3D_NUM_INPUT_BUFFERS;    
    pPortDef_ip->nBufferCountMin        = MP3D_NUM_INPUT_BUFFERS;    
    pPortDef_ip->nBufferSize             = MP3D_INPUT_BUFFER_SIZE;    
    pPortDef_ip->nBufferAlignment       = DSP_CACHE_ALIGNMENT;    
    pPortDef_ip->bEnabled                 = OMX_TRUE;    
    pPortDef_ip->bPopulated               = OMX_FALSE;    
    pPortDef_ip->eDomain                   = OMX_PortDomainAudio;    
    pPortDef_ip->format.audio.eEncoding = OMX_AUDIO_CodingMP3;    
    pPortDef_ip->format.audio.cMIMEType = NULL;    
    pPortDef_ip->format.audio.pNativeRender           = NULL;    
    pPortDef_ip->format.audio.bFlagErrorConcealment = OMX_FALSE;    
    /* 设置输出端口（OMX_PARAM_PORTDEFINITIONTYPE类型）的默认值 */    
    pPortDef_op->nSize                 = sizeof(OMX_PARAM_PORTDEFINITIONTYPE);    
    pPortDef_op->nPortIndex           = MP3D_OUTPUT_PORT;    
    pPortDef_op->eDir                  = OMX_DirOutput;    
    pPortDef_op->nBufferCountMin     = MP3D_NUM_OUTPUT_BUFFERS;    
    pPortDef_op->nBufferCountActual  = MP3D_NUM_OUTPUT_BUFFERS;    
    pPortDef_op->nBufferSize          = MP3D_OUTPUT_BUFFER_SIZE;    
    pPortDef_op->nBufferAlignment    = DSP_CACHE_ALIGNMENT;    
    pPortDef_op->bEnabled              = OMX_TRUE;    
    pPortDef_op->bPopulated            = OMX_FALSE;    
    pPortDef_op->eDomain               = OMX_PortDomainAudio;    
    pPortDef_op->format.audio.eEncoding      = OMX_AUDIO_CodingPCM;    
    pPortDef_op->format.audio.cMIMEType      = NULL;    
    pPortDef_op->format.audio.pNativeRender = NULL;    
    pPortDef_op->format.audio.bFlagErrorConcealment = OMX_FALSE;    
    /* ......省略，分配端口 */    
    /* 设置输入端口的默认格式 */    
    pPortFormat = pComponentPrivate->pCompPort[MP3D_INPUT_PORT]->pPortFormat;    
    OMX_CONF_INIT_STRUCT(pPortFormat, OMX_AUDIO_PARAM_PORTFORMATTYPE);    
    pPortFormat->nPortIndex         = MP3D_INPUT_PORT;    
    pPortFormat->nIndex             = OMX_IndexParamAudioMp3;    
    pPortFormat->eEncoding          = OMX_AUDIO_CodingMP3;    
    /* 设置输出端口的默认格式 */    
    pPortFormat = pComponentPrivate->pCompPort[MP3D_OUTPUT_PORT]->pPortFormat;    
    OMX_CONF_INIT_STRUCT(pPortFormat, OMX_AUDIO_PARAM_PORTFORMATTYPE);    
        pPortFormat->nPortIndex         = MP3D_OUTPUT_PORT;    
        pPortFormat->nIndex             = OMX_IndexParamAudioPcm;    
        pPortFormat->eEncoding          = OMX_AUDIO_CodingPCM;    
    /* ......省略部分内容 */    
    eError = Mp3Dec_StartCompThread(pHandle);   // 启动MP3解码线程    
    /* ......省略部分内容 */    
    return eError;    
}
```
 这个组件是OpenMax的标准实现方式，对外的接口的内容只有一个初始化函数。完成OMX_COMPONENTTYPE类型的初始化。输入端口的编号为MP3D_INPUT_PORT（==0），类型为OMX_PortDomainAudio，格式为OMX_AUDIO_CodingMP3。输出端口的编号是MP3D_OUTPUT_PORT（==1），类型为OMX_PortDomainAudio，格式为OMXAUDIO CodingPCM。

  OMX_Mp3Dec_CompThread.c中定义了MP3DEC_ComponentThread()函数，用于创建MP3解码的线程的执行函数。
  OMX_Mp3Dec_Utils.c中的Mp3Dec_StartCompThread()函数，调用了POSIX的线程库建立MP3解码的线程，如下所示：

``` cpp
nRet = pthread_create (&(pComponentPrivate->ComponentThread), NULL,    
    MP3DEC_ComponentThread, pComponentPrivate);
```

 Mp3Dec_StartCompThread()函数就是在组件初始化函数OMX_ComponentInit()最后调用的内容。MP3线程的开始并不表示解码过程开始，线程需要等待通过pipe机制获得命令和数据（cmdPipe和dataPipe），在适当的时候开始工作。这个pipe在MP3解码组件的SendCommand等实现写操作，在线程中读取其内容。


##### 2.4、Qualcomm(高通) OpenMax IL的硬件实现
###### 2.4.1、qcom OpenMax IL实现的结构和机制

（1）在AOSP中依然有对高通平台的OpenMax IL层实现代码，位于hardware/qcom/media/mm-core下。这一部分是OpenMax核心和公共部分，主要编译为libOmxCore.so。

（2）e.g. 继续在hardware/qcom/media下，选取mm-video-v4l2目录。即Video4linux2（简称V4L2),是linux中关于视频设备的内核驱动。再次进入vidc，（DivxDrmDecrypt为DRM数字版权相关）主要目录如下：

☯ vdec：视频解码处理，编译成libOmxVdec.so/libOmxVdecHevc.so
☯ venc：视频编码处理，编译成libOmxVenc.so
qcom OpenMax IL的核心和公共内容
       类似于前面介绍的TI，高通平台在OpenMax IL实现也是大同小异，位于hardware/qcom/media/mm-core，生成libOmxCore.so库。
       其中qc_omx_core为主要文件，位于hardware/qcom/media/mm-core/omxcore/src/common/下面。和TI的差不多，OMX_GetHandle()函数用户获取各个组件的句柄，其实现的主要片断如下所示：

``` cpp
//编解码器组件集合数组
extern omx_core_cb_type core[];

 OMX_API OMX_ERRORTYPE OMX_APIENTRY
OMX_GetHandle(OMX_OUT OMX_HANDLETYPE*     handle,
              OMX_IN OMX_STRING    componentName,
              OMX_IN OMX_PTR             appData,
              OMX_IN OMX_CALLBACKTYPE* callBacks)
{
  OMX_ERRORTYPE  eRet = OMX_ErrorNone;
  int cmp_index = -1;
  int hnd_index = -1;

  DEBUG_PRINT("OMXCORE API :  Get Handle %p %s %p\n", handle,
                                                     componentName,
                                                     appData);
  pthread_mutex_lock(&lock_core);
  if(handle)
  {
    struct stat sd;
	//组件句柄
    *handle = NULL;
	//获取根据组件名获取相应index
    cmp_index = get_cmp_index(componentName);

    if(cmp_index >= 0)
    {
       DEBUG_PRINT("getting fn pointer\n");

      // dynamically load the so 动态加载组件的so库
      core[cmp_index].fn_ptr =
        omx_core_load_cmp_library(core[cmp_index].so_lib_name,
                                  &core[cmp_index].so_lib_handle);

      if(core[cmp_index].fn_ptr)
      {
        // Construct the component requested
        // Function returns the opaque handle
        //根据获取的组件句柄初始化它
        void* pThis = (*(core[cmp_index].fn_ptr))();
        if(pThis)
        {
	      //包装一层，忽略
          void *hComp = NULL;
          hComp = qc_omx_create_component_wrapper((OMX_PTR)pThis);
          if((eRet = qc_omx_component_init(hComp, core[cmp_index].name)) !=
                           OMX_ErrorNone)
          {
              DEBUG_PRINT("Component not created succesfully\n");
              pthread_mutex_unlock(&lock_core);
              return eRet;

          }
          //设置回调
          qc_omx_component_set_callbacks(hComp,callBacks,appData);
          hnd_index = get_comp_handle_index(componentName);
 
          if(hnd_index >= 0)
          {
            //保存这个组件句柄
            core[cmp_index].inst[hnd_index]= *handle = (OMX_HANDLETYPE) hComp;
          }
          else
          {
          /*-------下面全是错误处理，忽略------*/
            DEBUG_PRINT("OMX_GetHandle:NO free slot available to store Component Handle\n");
            pthread_mutex_unlock(&lock_core);
            return OMX_ErrorInsufficientResources;
          }
          DEBUG_PRINT("Component %p Successfully created\n",*handle);
        }
        else
        {
          eRet = OMX_ErrorInsufficientResources;
          DEBUG_PRINT("Component Creation failed\n");
        }
      }
      else
      {
        eRet = OMX_ErrorNotImplemented;
        DEBUG_PRINT("library couldnt return create instance fn\n");
      }

    }
    else
    {
      eRet = OMX_ErrorNotImplemented;
      DEBUG_PRINT("ERROR: Already another instance active  ;rejecting \n");
    }
  }
  else
  {
    eRet =  OMX_ErrorBadParameter;
    DEBUG_PRINT("\n OMX_GetHandle: NULL handle \n");
  }
  pthread_mutex_unlock(&lock_core);
  return eRet;
}
```


  这里的有个数组：extern omx_core_cb_type core[]，是从别的文件中声明过来的全局变量，其中包含了各种编解码器的名称和一些属性的结构体。结构体定义位于hardware/qcom/media/mm-core/omxcore/src/common/qc_omx_core.h：

``` cpp
typedef struct _omx_core_cb_type
{
  char*                         name;// Component name 组件名
  create_qc_omx_component     fn_ptr;// create instance fn ptr 创建实例函数指针
  void*                         inst[OMX_COMP_MAX_INST];// Instance handle 实例句柄
  void*                so_lib_handle;// So Library handle so库句柄
  char*                  so_lib_name;// so directory so名
  char* roles[OMX_CORE_MAX_CMP_ROLES];// roles played 组件扮演的角色
}omx_core_cb_type;
```
  但是给omx_core_cb_type core[]这个结构体数组复制的地方要根据不同型号进行选取，我们进入hardware/qcom/media/mm-core/src下面，会看到有许多型号，7627A、7630、8084、8226、8610、8660等等。比如这个8974的，位于hardware/qcom/media/mm-core/src/8974/qc_registry_table_android.c中：

``` cpp
omx_core_cb_type core[] =
{
  //avc/h264解码器
  {
    "OMX.qcom.video.decoder.avc",
    NULL, // Create instance function
    // Unique instance handle
    {
      NULL
    },
    NULL,   // Shared object library handle
    "libOmxVdec.so",
    {
      "video_decoder.avc"
    }
  },
  //mpeg4解码器
{
    "OMX.qcom.video.decoder.mpeg4",
    NULL,   // Create instance function
    // Unique instance handle
    {
      NULL
    },
    NULL,   // Shared object library handle
    "libOmxVdec.so",
    {
      "video_decoder.mpeg4"
    }
  },
    //wmv解码器
  {
    "OMX.qcom.video.decoder.wmv",
    NULL, // Create instance function
    // Unique instance handle
    {
      NULL
    },
    NULL,   // Shared object library handle
    "libOmxVdec.so",
    {
      "video_decoder.vc1"
    }
  },
    //hevc/h265编码器
   {
    "OMX.qcom.video.encoder.hevc",
    NULL,   // Create instance function
    // Unique instance handle
    {
      NULL
    },
    NULL,   // Shared object library handle
    "libOmxVencHevc.so",
    {
      "video_encoder.hevc"
    }
  },
  
  ...太多了，省略...
  
  }
```
上面就是对编解码器相关信息的注册。

在OMX_GetHandle()函数中，根据omx_core_cb_type core[]数组中动态库的名称，动态打开各个编解码实现的动态库,然后进行初始化。
###### 2.4.1、一个qcom OpenMax IL组件的实现
 高通平台对于编解码组件的处理都比较集中，不像TI那么分散和细致。一个组件实现都要包含Qc_omx_component.h头文件，位于很多地方，如hardware/qcom/media/mm-core/inc，要实现里面相关纯虚函数。当一个组件被创建后要初始化，就要实现component_init(OMX_IN OMX_STRING componentName)方法。


举个例子，依然以Video4linux2平台，进入hardware/qcom/media/mm-video-v4l2/vidc/vdec/src查看视频解码相关组件。比如我们看看解码组件omx_vdec_hevc.cpp，查看component_init方法：

``` cpp
#ifdef VENUS_HEVC
#define DEVICE_NAME "/dev/video/venus_dec"
#else
#define DEVICE_NAME "/dev/video/q6_dec"
#endif

/* ======================================================================
   FUNCTION
   omx_vdec::ComponentInit

   DESCRIPTION
   Initialize the component.

   PARAMETERS
   ctxt -- Context information related to the self.
   id   -- Event identifier. This could be any of the following:
   1. Command completion event
   2. Buffer done callback event
   3. Frame done callback event

   RETURN VALUE
   None.

   ========================================================================== */
OMX_ERRORTYPE omx_vdec::component_init(OMX_STRING role)
{

    OMX_ERRORTYPE eRet = OMX_ErrorNone;
    struct v4l2_fmtdesc fdesc;
    struct v4l2_format fmt;
    struct v4l2_requestbuffers bufreq;
    struct v4l2_control control;
    unsigned int   alignment = 0,buffer_size = 0;
    int fds[2];
    int r,ret=0;
    bool codec_ambiguous = false;
    //打开设备文件"/dev/video/venus_dec"或"/dev/video/q6_dec"
    OMX_STRING device_name = (OMX_STRING)DEVICE_NAME;
    ......
    drv_ctx.video_driver_fd = open(device_name, O_RDWR);

	......
	//如果是一个打开成功后，为什么要再次打开？？excuse me ？
    if (drv_ctx.video_driver_fd == 0) {
        drv_ctx.video_driver_fd = open(device_name, O_RDWR);
    }
	//打开设备文件失败
    if (drv_ctx.video_driver_fd < 0) {
        DEBUG_PRINT_ERROR("Omx_vdec::Comp Init Returning failure, errno %d", errno);
        return OMX_ErrorInsufficientResources;
    }
    drv_ctx.frame_rate.fps_numerator = DEFAULT_FPS;//帧率分子
    drv_ctx.frame_rate.fps_denominator = 1;//帧率分母
	//创建一个异步线程，执行async_message_thread函数，对输入端进行设置
    ret = pthread_create(&async_thread_id,0,async_message_thread,this);
    //创建线程失败，则关闭设备文件
    if (ret < 0) {
        close(drv_ctx.video_driver_fd);
        DEBUG_PRINT_ERROR("Failed to create async_message_thread");
        return OMX_ErrorInsufficientResources;
    }

	......

    // Copy the role information which provides the decoder kind
    //将组建角色名字copy进设备驱动上下文结构体的kind属性
    strlcpy(drv_ctx.kind,role,128);
	//如果是mpeg4解码组件
    if (!strncmp(drv_ctx.kind,"OMX.qcom.video.decoder.mpeg4",\
                OMX_MAX_STRINGNAME_SIZE)) {
        strlcpy((char *)m_cRole, "video_decoder.mpeg4",\
                OMX_MAX_STRINGNAME_SIZE);
        drv_ctx.timestamp_adjust = true;
        drv_ctx.decoder_format = VDEC_CODECTYPE_MPEG4;
        eCompressionFormat = OMX_VIDEO_CodingMPEG4;
        output_capability=V4L2_PIX_FMT_MPEG4;
        /*Initialize Start Code for MPEG4*/
        codec_type_parse = CODEC_TYPE_MPEG4;
        m_frame_parser.init_start_codes (codec_type_parse);
	......
	//如果是mpeg2解码组件
    } else if (!strncmp(drv_ctx.kind,"OMX.qcom.video.decoder.mpeg2",\
                OMX_MAX_STRINGNAME_SIZE)) {
        strlcpy((char *)m_cRole, "video_decoder.mpeg2",\
                OMX_MAX_STRINGNAME_SIZE);
        drv_ctx.decoder_format = VDEC_CODECTYPE_MPEG2;
        output_capability = V4L2_PIX_FMT_MPEG2;
        eCompressionFormat = OMX_VIDEO_CodingMPEG2;
        /*Initialize Start Code for MPEG2*/
        codec_type_parse = CODEC_TYPE_MPEG2;
        m_frame_parser.init_start_codes (codec_type_parse);
	......
	//如果是h263解码组件
    } else if (!strncmp(drv_ctx.kind, "OMX.qcom.video.decoder.h263",\
                OMX_MAX_STRINGNAME_SIZE)) {
        strlcpy((char *)m_cRole, "video_decoder.h263",OMX_MAX_STRINGNAME_SIZE);
        DEBUG_PRINT_LOW("H263 Decoder selected");
        drv_ctx.decoder_format = VDEC_CODECTYPE_H263;
        eCompressionFormat = OMX_VIDEO_CodingH263;
        output_capability = V4L2_PIX_FMT_H263;
        codec_type_parse = CODEC_TYPE_H263;
        m_frame_parser.init_start_codes (codec_type_parse);
	......
	//如果是divx311...
    } else if (!strncmp(drv_ctx.kind, "OMX.qcom.video.decoder.divx311",\
                OMX_MAX_STRINGNAME_SIZE)) {
        strlcpy((char *)m_cRole, "video_decoder.divx",OMX_MAX_STRINGNAME_SIZE);
        DEBUG_PRINT_LOW ("DIVX 311 Decoder selected");
        drv_ctx.decoder_format = VDEC_CODECTYPE_DIVX_3;
        output_capability = V4L2_PIX_FMT_DIVX_311;
        eCompressionFormat = (OMX_VIDEO_CODINGTYPE)QOMX_VIDEO_CodingDivx;
        codec_type_parse = CODEC_TYPE_DIVX;
        m_frame_parser.init_start_codes (codec_type_parse);

        eRet = createDivxDrmContext();
        if (eRet != OMX_ErrorNone) {
            DEBUG_PRINT_ERROR("createDivxDrmContext Failed");
            return eRet;
        }
        //如果是divx4...
    } else if (!strncmp(drv_ctx.kind, "OMX.qcom.video.decoder.divx4",\
                OMX_MAX_STRINGNAME_SIZE)) {
        strlcpy((char *)m_cRole, "video_decoder.divx",OMX_MAX_STRINGNAME_SIZE);
        DEBUG_PRINT_ERROR ("DIVX 4 Decoder selected");
        drv_ctx.decoder_format = VDEC_CODECTYPE_DIVX_4;
        output_capability = V4L2_PIX_FMT_DIVX;
        eCompressionFormat = (OMX_VIDEO_CODINGTYPE)QOMX_VIDEO_CodingDivx;
        codec_type_parse = CODEC_TYPE_DIVX;
        codec_ambiguous = true;
        m_frame_parser.init_start_codes (codec_type_parse);

        eRet = createDivxDrmContext();
        if (eRet != OMX_ErrorNone) {
            DEBUG_PRINT_ERROR("createDivxDrmContext Failed");
            return eRet;
        }
        //如果是divx...
    } else if (!strncmp(drv_ctx.kind, "OMX.qcom.video.decoder.divx",\
                OMX_MAX_STRINGNAME_SIZE)) {
        strlcpy((char *)m_cRole, "video_decoder.divx",OMX_MAX_STRINGNAME_SIZE);
        DEBUG_PRINT_ERROR ("DIVX 5/6 Decoder selected");
        drv_ctx.decoder_format = VDEC_CODECTYPE_DIVX_6;
        output_capability = V4L2_PIX_FMT_DIVX;
        eCompressionFormat = (OMX_VIDEO_CODINGTYPE)QOMX_VIDEO_CodingDivx;
        codec_type_parse = CODEC_TYPE_DIVX;
        codec_ambiguous = true;
        m_frame_parser.init_start_codes (codec_type_parse);

        eRet = createDivxDrmContext();
        if (eRet != OMX_ErrorNone) {
            DEBUG_PRINT_ERROR("createDivxDrmContext Failed");
            return eRet;
        }
		//如果是avc/h264...
    } else if (!strncmp(drv_ctx.kind, "OMX.qcom.video.decoder.avc",\
                OMX_MAX_STRINGNAME_SIZE)) {
        strlcpy((char *)m_cRole, "video_decoder.avc",OMX_MAX_STRINGNAME_SIZE);
        drv_ctx.decoder_format = VDEC_CODECTYPE_H264;
        output_capability=V4L2_PIX_FMT_H264;
        eCompressionFormat = OMX_VIDEO_CodingAVC;
        codec_type_parse = CODEC_TYPE_H264;
        m_frame_parser.init_start_codes (codec_type_parse);
        m_frame_parser.init_nal_length(nal_length);
	......
	//如果是hevc/h265...
    } else if (!strncmp(drv_ctx.kind, "OMX.qcom.video.decoder.hevc",\
                OMX_MAX_STRINGNAME_SIZE)) {
        strlcpy((char *)m_cRole, "video_decoder.hevc",OMX_MAX_STRINGNAME_SIZE);
        drv_ctx.decoder_format = VDEC_CODECTYPE_HEVC;
        output_capability=V4L2_PIX_FMT_HEVC;
        eCompressionFormat = (OMX_VIDEO_CODINGTYPE)QOMX_VIDEO_CodingHevc;
        codec_type_parse = CODEC_TYPE_HEVC;
        m_frame_parser.init_start_codes (codec_type_parse);
        m_frame_parser.init_nal_length(nal_length);
	...
	//如果是vc1...
    } else if (!strncmp(drv_ctx.kind, "OMX.qcom.video.decoder.vc1",\
                OMX_MAX_STRINGNAME_SIZE)) {
        strlcpy((char *)m_cRole, "video_decoder.vc1",OMX_MAX_STRINGNAME_SIZE);
        drv_ctx.decoder_format = VDEC_CODECTYPE_VC1;
        eCompressionFormat = OMX_VIDEO_CodingWMV;
        codec_type_parse = CODEC_TYPE_VC1;
        output_capability = V4L2_PIX_FMT_VC1_ANNEX_G;
        m_frame_parser.init_start_codes (codec_type_parse);
	//如果是wmv...
    } else if (!strncmp(drv_ctx.kind, "OMX.qcom.video.decoder.wmv",\
                OMX_MAX_STRINGNAME_SIZE)) {
        strlcpy((char *)m_cRole, "video_decoder.vc1",OMX_MAX_STRINGNAME_SIZE);
        drv_ctx.decoder_format = VDEC_CODECTYPE_VC1_RCV;
        eCompressionFormat = OMX_VIDEO_CodingWMV;
        codec_type_parse = CODEC_TYPE_VC1;
        output_capability = V4L2_PIX_FMT_VC1_ANNEX_L;
        m_frame_parser.init_start_codes (codec_type_parse);
	//如果是vp8...
    } else if (!strncmp(drv_ctx.kind, "OMX.qcom.video.decoder.vp8",\
                OMX_MAX_STRINGNAME_SIZE)) {
        strlcpy((char *)m_cRole, "video_decoder.vp8",OMX_MAX_STRINGNAME_SIZE);
        output_capability=V4L2_PIX_FMT_VP8;
        eCompressionFormat = OMX_VIDEO_CodingVPX;
        codec_type_parse = CODEC_TYPE_VP8;
        arbitrary_bytes = false;
        
    // 如果是不认识的解码组件，则报错
    } else {
        DEBUG_PRINT_ERROR("ERROR:Unknown Component");
        eRet = OMX_ErrorInvalidComponentName;
    }
	//如果错误
    if (eRet == OMX_ErrorNone) {
		//设置视频输出编码格式为YUV的一种
        drv_ctx.output_format = VDEC_YUV_FORMAT_NV12;
        //设置颜色编码
        OMX_COLOR_FORMATTYPE dest_color_format = (OMX_COLOR_FORMATTYPE)
            QOMX_COLOR_FORMATYUV420PackedSemiPlanar32m;
        if (!client_buffers.set_color_format(dest_color_format)) {
            DEBUG_PRINT_ERROR("Setting color format failed");
            eRet = OMX_ErrorInsufficientResources;
        }
		//订阅事件
        capture_capability= V4L2_PIX_FMT_NV12;
        ret = subscribe_to_events(drv_ctx.video_driver_fd);
        if (ret) {
            DEBUG_PRINT_ERROR("Subscribe Event Failed");
            return OMX_ErrorInsufficientResources;
        }
		
        struct v4l2_capability cap;
        //设置查询能力标志位
        ret = ioctl(drv_ctx.video_driver_fd, VIDIOC_QUERYCAP, &cap);
        if (ret) {
            DEBUG_PRINT_ERROR("Failed to query capabilities");
            /*TODO: How to handle this case */
        } else {
            DEBUG_PRINT_HIGH("Capabilities: driver_name = %s, card = %s, bus_info = %s,"
                    " version = %d, capabilities = %x", cap.driver, cap.card,
                    cap.bus_info, cap.version, cap.capabilities);
        }
        ret=0;
        fdesc.type=V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;
        fdesc.index=0;
        while (ioctl(drv_ctx.video_driver_fd, VIDIOC_ENUM_FMT, &fdesc) == 0) {
            DEBUG_PRINT_HIGH("fmt: description: %s, fmt: %x, flags = %x", fdesc.description,
                    fdesc.pixelformat, fdesc.flags);
            fdesc.index++;
        }
        fdesc.type=V4L2_BUF_TYPE_VIDEO_OUTPUT_MPLANE;
        fdesc.index=0;
        while (ioctl(drv_ctx.video_driver_fd, VIDIOC_ENUM_FMT, &fdesc) == 0) {

            DEBUG_PRINT_HIGH("fmt: description: %s, fmt: %x, flags = %x", fdesc.description,
                    fdesc.pixelformat, fdesc.flags);
            fdesc.index++;
        }
        update_resolution(320, 240);
        fmt.type = V4L2_BUF_TYPE_VIDEO_OUTPUT_MPLANE;
        fmt.fmt.pix_mp.height = drv_ctx.video_resolution.frame_height;
        fmt.fmt.pix_mp.width = drv_ctx.video_resolution.frame_width;
        fmt.fmt.pix_mp.pixelformat = output_capability;
        ret = ioctl(drv_ctx.video_driver_fd, VIDIOC_S_FMT, &fmt);
        if (ret) {
            /*TODO: How to handle this case */
            DEBUG_PRINT_ERROR("Failed to set format on output port");
        }
        DEBUG_PRINT_HIGH("Set Format was successful");
        //如果有歧义的解码组件
        if (codec_ambiguous) {
            if (output_capability == V4L2_PIX_FMT_DIVX) {
                struct v4l2_control divx_ctrl;

                if (drv_ctx.decoder_format == VDEC_CODECTYPE_DIVX_4) {
                    divx_ctrl.value = V4L2_MPEG_VIDC_VIDEO_DIVX_FORMAT_4;
                } else if (drv_ctx.decoder_format == VDEC_CODECTYPE_DIVX_5) {
                    divx_ctrl.value = V4L2_MPEG_VIDC_VIDEO_DIVX_FORMAT_5;
                } else {
                    divx_ctrl.value = V4L2_MPEG_VIDC_VIDEO_DIVX_FORMAT_6;
                }

                divx_ctrl.id = V4L2_CID_MPEG_VIDC_VIDEO_DIVX_FORMAT;
                ret = ioctl(drv_ctx.video_driver_fd, VIDIOC_S_CTRL, &divx_ctrl);
                if (ret) {
                    DEBUG_PRINT_ERROR("Failed to set divx version");
                }
            } else {
                DEBUG_PRINT_ERROR("Codec should not be ambiguous");
            }
        }
		//解码相关参数设置
        fmt.type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;
        fmt.fmt.pix_mp.height = drv_ctx.video_resolution.frame_height;
        fmt.fmt.pix_mp.width = drv_ctx.video_resolution.frame_width;
        fmt.fmt.pix_mp.pixelformat = capture_capability;
        ret = ioctl(drv_ctx.video_driver_fd, VIDIOC_S_FMT, &fmt);
        if (ret) {
            /*TODO: How to handle this case */
            DEBUG_PRINT_ERROR("Failed to set format on capture port");
        }
        DEBUG_PRINT_HIGH("Set Format was successful");
        if (secure_mode) {
            control.id = V4L2_CID_MPEG_VIDC_VIDEO_SECURE;
            control.value = 1;
            DEBUG_PRINT_LOW("Omx_vdec:: calling to open secure device %d", ret);
            ret=ioctl(drv_ctx.video_driver_fd, VIDIOC_S_CTRL,&control);
            if (ret) {
                DEBUG_PRINT_ERROR("Omx_vdec:: Unable to open secure device %d", ret);
                close(drv_ctx.video_driver_fd);
                return OMX_ErrorInsufficientResources;
            }
        }

        /*Get the Buffer requirements for input and output ports*/
        //获得输入和输出的缓冲条件
        drv_ctx.ip_buf.buffer_type = VDEC_BUFFER_TYPE_INPUT;
        drv_ctx.op_buf.buffer_type = VDEC_BUFFER_TYPE_OUTPUT;
        if (secure_mode) {
            drv_ctx.op_buf.alignment=SZ_1M;
            drv_ctx.ip_buf.alignment=SZ_1M;
        } else {
            drv_ctx.op_buf.alignment=SZ_4K;
            drv_ctx.ip_buf.alignment=SZ_4K;
        }
        drv_ctx.interlace = VDEC_InterlaceFrameProgressive;
        drv_ctx.extradata = 0;
        drv_ctx.picture_order = VDEC_ORDER_DISPLAY;
        control.id = V4L2_CID_MPEG_VIDC_VIDEO_OUTPUT_ORDER;
        control.value = V4L2_MPEG_VIDC_VIDEO_OUTPUT_ORDER_DISPLAY;
        ret = ioctl(drv_ctx.video_driver_fd, VIDIOC_S_CTRL, &control);
        drv_ctx.idr_only_decoding = 0;

        m_state = OMX_StateLoaded;
#ifdef DEFAULT_EXTRADATA
        if (eRet == OMX_ErrorNone && !secure_mode)
            enable_extradata(DEFAULT_EXTRADATA, true, true);
#endif
        eRet=get_buffer_req(&drv_ctx.ip_buf);
        DEBUG_PRINT_HIGH("Input Buffer Size =%d",drv_ctx.ip_buf.buffer_size);
        get_buffer_req(&drv_ctx.op_buf);
        //如果解码器格式是h264或者hevc/h265
        if (drv_ctx.decoder_format == VDEC_CODECTYPE_H264 ||
                drv_ctx.decoder_format == VDEC_CODECTYPE_HEVC) {
            h264_scratch.nAllocLen = drv_ctx.ip_buf.buffer_size;
            h264_scratch.pBuffer = (OMX_U8 *)malloc (drv_ctx.ip_buf.buffer_size);
            h264_scratch.nFilledLen = 0;
            h264_scratch.nOffset = 0;

            if (h264_scratch.pBuffer == NULL) {
                DEBUG_PRINT_ERROR("h264_scratch.pBuffer Allocation failed ");
                return OMX_ErrorInsufficientResources;
            }
        }
		//如果解码器格式是h264
        if (drv_ctx.decoder_format == VDEC_CODECTYPE_H264) {
            if (m_frame_parser.mutils == NULL) {
                m_frame_parser.mutils = new H264_Utils();

                if (m_frame_parser.mutils == NULL) {
                    DEBUG_PRINT_ERROR("parser utils Allocation failed ");
                    eRet = OMX_ErrorInsufficientResources;
                } else {
                    m_frame_parser.mutils->initialize_frame_checking_environment();
                    m_frame_parser.mutils->allocate_rbsp_buffer (drv_ctx.ip_buf.buffer_size);
                }
            }
			//创建一个h264流的解析器
            h264_parser = new h264_stream_parser();
            if (!h264_parser) {
                DEBUG_PRINT_ERROR("ERROR: H264 parser allocation failed!");
                eRet = OMX_ErrorInsufficientResources;
            }
        }
		//打开一个管道，读写端保存进fds数组
        if (pipe(fds)) {
            DEBUG_PRINT_ERROR("pipe creation failed");
            eRet = OMX_ErrorInsufficientResources;
        } else {
            int temp1[2];
            if (fds[0] == 0 || fds[1] == 0) {
                if (pipe (temp1)) {
                    DEBUG_PRINT_ERROR("pipe creation failed");
                    return OMX_ErrorInsufficientResources;
                }
                //close (fds[0]);
                //close (fds[1]);
                fds[0] = temp1 [0];
                fds[1] = temp1 [1];
            }
            //输入/读
            m_pipe_in = fds[0];
            //输出/写
            m_pipe_out = fds[1];
            //创建一个工作线程，调用omx开始处理解码，并进行i/o操作
            r = pthread_create(&msg_thread_id,0,message_thread,this);

            if (r < 0) {
                DEBUG_PRINT_ERROR("component_init(): message_thread creation failed");
                eRet = OMX_ErrorInsufficientResources;
            }
        }
    }
	//没有错误，然后收尾
    if (eRet != OMX_ErrorNone) {
        DEBUG_PRINT_ERROR("Component Init Failed");
        DEBUG_PRINT_HIGH("Calling VDEC_IOCTL_STOP_NEXT_MSG");
        (void)ioctl(drv_ctx.video_driver_fd, VDEC_IOCTL_STOP_NEXT_MSG,
                NULL);
        DEBUG_PRINT_HIGH("Calling close() on Video Driver");
        close (drv_ctx.video_driver_fd);
        drv_ctx.video_driver_fd = -1;
    } else {
        DEBUG_PRINT_HIGH("omx_vdec::component_init() success");
    }
    //memset(&h264_mv_buff,0,sizeof(struct h264_mv_buffer));
    return eRet;
}
```

 这个是解码组件的初始化实现。我们能够看出和TI的差距挺大的。步骤大概如下（maybe wrong）：

☯ 打开media相关设备文件
☯ 创建一个异步线程，执行async_message_thread函数，对输入端进行设置
☯ 根据解码器role名配置相关属性
☯ 对视频解码相关基本配置进行设置
☯ 创建一个管道，然后再开一个个工作线程，调用omx开始处理解码，并进行i/o操作


#### （三）、Android中OpenMax的实现

##### 3.1、android MediaCodec ACodec
##### 3.1.1、MediaCodec

MediaCodec类可用于访问Android底层的媒体编解码器，例如，编码/解码组件。它是Android为多媒体支持提供的底层接口的一部分（通常与MediaExtractor, MediaSync, MediaMuxer, MediaCrypto, MediaDrm, Image, Surface, 以及AudioTrack一起使用）。
从广义上讲，一个编解码器通过处理输入数据来产生输出数据。它通过异步方式处理数据，并且使用了一组输入输出buffers。在简单层面，请求（或接收）到一个空的输入buffer，向里面填满数据并将它传递给编解码器处理。这个编解码器将使用完这些数据并向所有空的输出buffer中的一个填充这些数据。最终，请求（或接受）到一个填充了数据的buffer,可以使用其中的数据内容，并且在使用完后将其释放回编解码器。


1、	Data Types
编解码器处理三种类型的数据：压缩数据，原始音频数据，原始视频数据。上述三种数据都可以通过ByteBuffers进行处理，但需要为原始视频数据提供一个Surface来提高编解码性能。
	压缩缓存（Compressed Buffers）：输入缓冲区(解码器)和输出缓冲区(编码器)包含压缩数据格式的类型。
	原始音频缓存（Raw Audio Buffers）：原始音频缓冲区包含整个PCM音频帧数据。
	原始视频缓存（Raw Video Buffers）：ByteBuffer模式视频缓冲区根据他们的颜色格式布局。视频编解码器可能支持三种类型的色彩格式：1)、native raw video format：被COLOR_FormatSurface标记，其可与输入或输出Surface一起使用；2)、flexible YUV buffers（如COLOR_FormatYUV420Flexible），可以与输入/输出Surface一起使用, ByteBuffer模式下可以通过调用getInput/OutputImage(int)方法进行使用；3)、通常只在ByteBuffer模式下被支持。由供应商指定，可以使用 getInput/OutputImage(int)方法。


2、States
在编解码器的生命周期内有三种理论状态：停止态-Stopped、执行态-Executing、释放态-Released。停止状态（Stopped）包括了三种子状态：未初始化（Uninitialized）、配置（Configured）、错误（Error）。执行状态（Executing）在概念上会经历三种子状态：刷新（Flushed）、运行（Running）、流结束（End-of-Stream）。

![Alt text | center](https://raw.githubusercontent.com/zhoujinjian777/PicGo/master/video.system/VS-04-10-MediaCodec-states.png)



使用工厂方法之一创建一个编解码器的时候，是处于Uninitialized状态。首先，需要通过configure(…)方法配置它，以此进入Configured 状态。然后，通过调用start()方法转入Executing 状态。在这个状态下可以通过上述buffer队列操作过程数据。
调用start()方法后立即进入Flushed状态，此时编解码器拥有所有的缓存。一旦第一个输入缓存被移出队列，编解码器就转入运行子状态，这种状态占据了编解码器的大部分生命周期。当将一个带有end-of-stream marker标记的输入缓存入队列时，编解码器将转入流结束子状态。可在Executing状态的任何时候通过调用flush()。
调用stop()方法返回编解码器的Uninitialized 状态，因此这个编解码器需要再次configured 。当使用完编解码器后，必须调用release()方法释放其资源。


3、Data Processing
每一个编解码器包含一组输入和输出缓存，这些缓存在API调用中通过buffer-ID进行引用。当成功调用start()方法后客户端将不会拥有输入或输出buffers。在同步模式下，通过调用dequeueInput/OutputBuffer(…) 方法从编解码器获得一个输入或输出buffer；在异步模式下，可以通过MediaCodec.Callback.onInput/OutputBufferAvailable(…)的回调方法自动地获得可用的buffers。
在获得一个输入buffe后，填充数据，利用queueInputBuffer/queueSecureInputBuffer方法将其提交给编解码器，不要提交多个具有相同时间戳的输入bufers（除非它是也被同样标记的codec-specific data，因为codec-specific data缓冲的时间戳无意义）。
a.	Asynchronous Processing using Buffers
    从Android 5.0开始，首选的方法是调用configure之前通过设置回调异步地处理数据。异步模式稍微改变了状态转换方式，因为必须在调用flush()方法后再调用start()方法才能使编解码器的状态转换为Running子状态并开始接收输入buffers。同样，初始调用start方法将编解码器的状态直接变化为Running 子状态并通过回调方法开始传递可用的输入buufers。
b.	Synchronous Processing using Buffers
从Android 5.0开始，即使在同步模式下使用编解码器，也应该通过getInput/OutputBuffer(int) 和/或 getInput/OutputImage(int) 方法检索输入和输出buffers。

##### 3.1.2、ACodec

1、ACodec消息机制：
	ACodec有一个BaseState和派生出来的其他State，如 UninitializedState, LoadedToIdleState, ExecutingState等。当有消息过来时，如果派生类有重写的方法，则会调到重写的方法，如果没有，则会调到BaseState的
	ACodec继承自AHierarchicalStateMachine类，该类用于将收到的消息传递给哪个state。
	ACodec收到的消息分两种，一种是MediaCodec传过来的，对应onMessageReceived方法；另一种是OMX Component传过来的，对应onOMXMessage方法。而onOMXMessage里面又分了4种情况来调用不同的方法。（EVENT、EMPTY_BUFFER_DONE、FILL_BUFFER_DONE和FRAME_RENDERED）


2、MediaCodec与ACodec的通知：
	OMX的组件解码之后，ACodec::BaseState:: onOMXFillBufferDone (…)会被回调，去取得解码后的数据。然后会在onOMXFillBufferDone中调用notify通知MediaCodec，发给MediaCodec的消息形如notify->setInt32("what", CodecBase::kWhatDrainThisBuffer);
	MediaCodec收到ACodec发的消息之后会updateBuffers(kPortIndexOutput, msg) 进行更新，同时调用onOutputBufferAvailable()中通知NuPlayer::Decoder有可用的output buffer。


3、ACodec有三种端口模式状态，其会根据当前处于哪个状态来决定buffer如何处理：
	KEEP_BUFFERS：当ACodec处于BaseState或者收到OnInputBufferFilled消息但是buffer里面没有填充有效的数据时，ACodec握有的buffer不会送到OMX 组件；
	RESUBMIT_BUFFERS：当ACodec处于ExecutingState或者处于OutputPortSettingChangedState但是当前是input口的buffer时，ACodec将握有的buffer送给OMX 组件；
	FREE_BUFFERS：当ACodec处于OutputPortSettingChangedState并且当前是output口的buffer时，ACodec将握有的buffer free。


4、stagefright类的调用关系：
	OMXNodeInstance负责创建并维护不同的实例，这些实例以node作为唯一标识。这样播放器中每个ACodec在OMX服务端都对应有了自己的OMXNodeInstance实例。
	OMXMaster用来维护底层软硬件解码库，根据OMXNodeInstance中想要的解码器来创建解码实体组件。
	OMXPluginBase负责加载组件库，创建组件实例，由OMXMaster管理。Android原生提供的组件都是由SoftOMXPlugin类来管理，这个类就是继承自OMXPluginBase。（Android源码提供了一些软件解码和编码的组件，它们被抽象为SoftOMXComponent）
	OMXClient是客户端用来与OMX IL进行通信的。
	内部结构CallbackDispatcher作用是用来接收回调函数的消息
	OMXNodeInstance + CallbackDispatcher = 合作完成不同实例的消息处理任务


5、ACodec同OMXNodeInstance的消息传递：
	ACodec将CodecObserver observer对象通过omx->allocateNode()传递到OMXNodeInstance。
	OMXNodeInstance将kCallbacks(OnEvent,OnEmptyBufferDone,OnFillBufferDone)传递给OMX Component
	当OMX Component有消息notify上来时，OMXNodeInstance最先收到，然后调用OMX.cpp。将消息在OMX.cpp里面将OMX Component thread转换到CallbackDispatcher线程中处理。CallbackDispatcher又将消息反调到OMXNodeInstance. 最后调用CodecObserver 的onMessage()回到ACodec中


6、	ACodec与OMX组件的关系
	ACodec ，CodecObserver和OMXNodeInstance是一一对应的，简单的可以理解它们3个构成了OpenMAX IL的一个Component，每一个node就是一个codec在OMX服务端的标识。当然还有CallbackDispatcher，用于处理codec过来的消息，通过它的post/loop/dispatch来发起接收，从OMX.cpp发送消息，最终通过OMXNodeInstance::onMessage -> CodecObserver::onMessage -> ACodec::onMessage一路往上，当然消息的来源是因为我们有向codec注册OMXNodeInstance::kCallbacks。
	而在OMXPluginBase创建组件实例的时候，需要传递一个callback给组件，这个callback用于接收组件的消息，它的实现是在OMXNodeInstance.cpp中。而kcallbacks是OMXNodeInstance的静态成员变量，它内部的三个函数指针分别指向了OMXNodeInstance的三个静态方法，也即是这三个方法与组件进行着消息传递

![Alt text | center](https://raw.githubusercontent.com/zhoujinjian777/PicGo/master/video.system/VS-04-11-OMX-allocateNode.png)




	对于NuPlayer来说，它并不直接接触解码组件，而是通过创建ACodec来和组件交互。ACode内部有一个id，这个id对应于一个OMXNodeInstance。OMX对象中会对产生的每一个OMXNodeInstance分配一个唯一的node_id。每一个OMXNodeInstance内部又保存着组件实例的指针【OMX_HANDLETYPE mHandle;】，通过这个指针就可以和组件进行交互。交互的流程为：ACodec → OMX → OMXNodeInstance → COMPONENT。

![Alt text | center](https://raw.githubusercontent.com/zhoujinjian777/PicGo/master/video.system/VS-04-12-NuPlayer-libstagefrighthw.png)




8、组件的管理
	对组件的管理可以总结为：通过OMXMaster加载libstagefrighthw.so库文件，创建OMXPluginBase【即创建继承此类的组件对象】，通过这个类来管理组件。
	Android源码提供了一些软件解码和编码的组件，它们被抽象为SoftOMXComponent。OMXPluginBase扮演者组件的管理者。它负责加载组件库，创建组件实例。而OMXMaster则管理着OMXPluginBase，Android原生提供的组件都是由SoftOMXPlugin类来管理，这个类就是继承自OMXPluginBase。
	对于厂商来说，如果要实现自己的组件管理模块，需要通过继承实现OMXPluginBase，并将之编译为libstagefrighthw.so。在OMXMaster中会加载这个库文件，然后调用其createOMXPlugin方法获得一个OMXPluginBase指针，然后将其加入OMXPluginBase列表以及与组件名相关的map 【mPluginByComponentName】中，后续都会通过OMXPluginBase来管理组件。

![Alt text | center](https://raw.githubusercontent.com/zhoujinjian777/PicGo/master/video.system/VS-04-13-OMXMaster-addPlugin.png)



##### 3.2、音视频解码数据处理
数据处理请参考： [音视频解码数据处理](http://charlesvincent.cc/2018/06/06/Android%20Video%20System%EF%BC%882%EF%BC%89%EF%BC%9A%E9%9F%B3%E8%A7%86%E9%A2%91%E5%88%86%E7%A6%BBMediaExtractor%E3%80%81%E8%A7%A3%E7%A0%81Decoder%E3%80%81%E6%B8%B2%E6%9F%93Renderer%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/#%EF%BC%88%E4%B8%89%EF%BC%89%E3%80%81%E9%9F%B3%E8%A7%86%E9%A2%91%E8%A7%A3%E7%A0%81%E6%95%B0%E6%8D%AE%E5%A4%84%E7%90%86)

##### 3.3、高通实现（参考高通文档）
[OpenMAX Integration Layer Video Encoder for Linux Android（高通文档）](https://createpoint.qti.qualcomm.com)
[OpenMAX Integration Layer Video Decoder for Linux Android（高通文档）](https://createpoint.qti.qualcomm.com)

#### （四）、参考资料(特别感谢各位前辈的分析和图示)：
[Android多媒体开发(五)----OpenMax简介](http://windrunnerlihuan.com/2016/12/15/Android%E5%A4%9A%E5%AA%92%E4%BD%93%E5%BC%80%E5%8F%91-%E4%BA%94-OpenMax%E7%AE%80%E4%BB%8B/)
[Android多媒体开发(六)----Android中OpenMax的实现(preview)](http://windrunnerlihuan.com/2016/12/26/Android%E5%A4%9A%E5%AA%92%E4%BD%93%E5%BC%80%E5%8F%91-%E5%85%AD-Android%E4%B8%ADOpenMax%E7%9A%84%E5%AE%9E%E7%8E%B0-preview/)
[Android多媒体开发(七)----Android中OpenMax的实现](http://windrunnerlihuan.com/2016/12/29/Android%E5%A4%9A%E5%AA%92%E4%BD%93%E5%BC%80%E5%8F%91-%E4%B8%83-Android%E4%B8%ADOpenMax%E7%9A%84%E5%AE%9E%E7%8E%B0/)
[android ACodec MediaCodec NuPlayer flow - CSDN博客](https://blog.csdn.net/dfhuang09/article/details/54926526)
[android MediaCodec ACodec - CSDN博客](https://blog.csdn.net/dfhuang09/article/details/60132620)