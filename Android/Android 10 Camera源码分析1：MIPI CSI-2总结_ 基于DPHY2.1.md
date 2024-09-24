---
title:  Android 10 Camera源码分析1：MIPI CSI2总结基于DPHY2.1(转载)
cover: https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/post.cover.pictures/bing-wallpaper-2018.04.31.jpg
categories:
  - Camera
tags:
  - Camera
toc: true
abbrlink: 20220101
date: 2022-01-01 01:01:00
---

注：文章都是通过阅读各位前辈总结的资料 Android 10.0 && Linux（Kernel 4.19）Rockchip平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除（◕‿◕），转载请注明出处（©Rockchip ©Android @Linux 版权所有），谢谢。
（==**文章基于 Kernel-4.19**==）&&（==**文章基于 Android 10.0**==）
 [【zhoujinjian.com博客原图链接】](https://github.com/zhoujinjianz) 
 [【开发板 Khadas Edge V】](https://www.khadas.cn/edge-v)
[【开发板 Khadas Edge V Android 10.0 && Linux（Kernel 4.19）源码链接】](https://github.com/khadas/linux/tree/khadas-edge-Qt)

正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢！！！

[**[Gustav杂物间]:MIPI CSI-2总结: 基于DPHY2.1**](https://www.ifreehub.com/archives/45/)

--------------------------------------------------------------------------------
==源码（部分）==：
--------------------------------------------------------------------------------


CSI（Camera Serial Interface）定义了摄像头外设与主机控制器之间的接口，旨在确定摄像头与主机控制器在移动应用中的标准。

* * *

**关键词描述**
---------

| 缩写 | 解释 |
| --- | --- |
| CCI | Camera Control Interface（物理层组件，通常使用I2C或I3C进行通信） |
| CIL | Control and Interface Logic |
| DT | Data Type（数据格式，YUV422、RGB888等） |
| SoT | Start of Transmission（传输启动信号） |
| EoT | End of Transmission（传输停止信号） |
| FS | Frame Start（一帧画面开始标志） |
| FE | Frame End（一帧画面结束标志） |
| LS | Line Start（一行像素开始标志） |
| LE | Line End（一行像素结束标志） |
| PH | Packet Header（包头） |
| PF | Packet Footer（包尾） |
| HS | High SPeed（DPHY的传输模式之一） |
| LP | Low Power（DPHY的传输模式之一） |
| LP-RX | Low-Power Receiver（DPHY LP接收器） |
| LP-TX | Low-Power Transmitter（DPHY LP发送器） |
| HS-RX | High-Speed Receiver（DPHY HS接收器） |
| HS-TX | High-Speed Transmitter（DPHY HS发送器） |
| Lane | 单向、点对点的信号传输通道，对于DPHY由2线差分接口构成 |
| Virtual Channel | 用于标识多路独立数据流，DPHY最高支持16个虚拟通道 |
| UI | 单位间隔，等于Clock Lane上任意HS状态（HS0或HS1）的持续时间 |

* * *

**概述**
------

CSI-2定义了摄像头应用中发送方（camera）与接收方（soc）之间的数据与控制传输标准，其物理层支持DPHY与CPHY两种，这里以DPHY为例。

* * *

**CSI-2层定义**
------------

与网络标准的多层协议相似，CSI-2标准也对camera数据处理的过程进行了分层，简单来说分为应用层、协议层与物理层。协议层又进行了细分：像素字节转换层、低级协议层、Lane管理层。

![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android.10.Camera.01/csi2_layer.png)

*   应用层（Application Layer）  
    该层主要用于不同场景对数据的处理过程，对于发送方，多为camera生成数据，对于接收方，多为SOC对数据进行处理。
*   协议层（Protocol Layer）  
    CSI-2协议可以使用SOC上的一个物理接口实现多条数据流的传输。协议层规定了如何对多条数据流进行标记和交织，从而使每条数据流能够正确地重建。
    
    *   像素字节转换层（Pixel/Byte Packing/Unpacking Layer）:CSI-2能够支持多种多样的像素格式，对于发送方，在数据发送之前，需要根据像素格式，将像素数据转换为对应的字节流；对于接收方，在将数据提供给应用层之前，需要将字节流数据转换为像素数据。
    *   低级协议层（Low Level Protocol）：LLP指的是SoT与EoT之间的数据包字节流协议，LLP的最小单元为字节。
    *   Lane管理器（Lane Management）：为了适应不同场景下对带宽的要求，CSI-2规定了Lane的数量是可拓展的。因此，在面临多Lane同时传输时，发送方需要对字节流进行公平分流（distributor），接收方则需要对多Lane数据进行合并（merger）。
*   物理层（PHY Layer）  
    PHY层指定了传输媒介，在电气层面从串行bit流中捕捉“0”与“1”，同时生成SoT与EoT等信号。

* * *

**物理层 DPHY**
------------

DPHY在物理上采用2线差分接口，由1对的差分clock lane与1对或多对的差分data lane组成。

![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android.10.Camera.01/dphy.png)

上图表明使用DPHY作为物理层时，Camera与SOC之间的硬件关系。SOC的CCI组件通过I2C完成对Camera的配置，使其输出mipi信号，其中一对Clock+/-则由Clock Lane标示，一对DataNBA+/-则由Data Lane标示。

DPHY工作于两种工作模式：

*   HS（High Speed Mode），这种模式用于传输高速的数据信号，如视频流；高速模式下，每对Lane都是工作在低电压摆幅的差分状态下，数据速率为80Mbps到1500Mbps。
*   LP（Low Power Mode），这种模式则可以用来传输控制信号；低速模式下，每对lane的2根导线都转变为单端状态，数据速率为10Mbps。

![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android.10.Camera.01/Lane_Module.png)

上图为单个Lane模块的内部组成，包含了CIL（Lane控制器与借口逻辑器），LP驱动器，HS驱动器，LP冲突检测。CIL负责控制各个驱动器的工作状态，使得Dp、Dn的工作状态可以在HS与LP之间进行切换。

![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android.10.Camera.01/Line_Level.png)

处于HS模式下，差分信号电平摆幅约为200mV；处于LP模式下，单端信号电平摆幅约为1.2V。  
在LP模式下，根据各个Line的电平可以确定当前Lane的State。

### **Data Lane**

![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android.10.Camera.01/Lane_State.png)

Data Lane差分线电平的高低表明了当前处于何种状态，发送方通过驱动差分线一系列的状态变化，进入不同的工作模式。

*   Burst Mode：High-Speed下的唯一模式，高速数据传输模式，此时各个Lane的Line工作在差分模式
*   Control Mode：Low Power下的一种模式，可以通过变化不同的state进入其他模式。
*   Escape Mode：Low Power下的特殊模式，在这种模式下可以使用一些特别的功能，详见下文。

协议中规定了进入不同模式时的state变化状态：

*   HS模式：
    
    *   进入HS模式：LP11->LP01->LP00

通常进入HS模式也就伴随着高速数据的传输，因此SoT（启动传输）信号也就产生（由于手头没有差分探针，示波器抓到的波形只能看出大致形状）。

![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android.10.Camera.01/SoT.png)

![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android.10.Camera.01/SoT_Scop.png)

上图为完整的SoT时序，可以看出SoT由“进入HS模式（A）”和“同步序列（B）”组成。

SoT流程如下：

| 发送方 | 接收方 |
| --- | --- |
| 处于stop状态 | 监测stop状态 |
| 进入HS-Rqst状态（LP01）并保持TLPX时间 | 监测到LP11至LP01的状态变化 |
| 进入Bridge状态（LP00）并保持THS-PREPARE时间 | 监测到LP01至LP00的状态变化 |
| 打开HS驱动器，同时关闭LP驱动器 | \_ |
| 进入HS0状态并保持THS-ZERO时间 | 使能HS的接收，并等待THS\_SETTLE时间，进入稳定期 |
| \_ | 寻找Sync\_Sequence |
| 发出Sync\_Sequenc'00011101' | \_ |
| \_ | 接收到Sync\_Sequence |
| 发送负载数据 |
| \_ | 接收负载数据 |

*   退出HS模式：LP11

![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android.10.Camera.01/EoT.png)

![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android.10.Camera.01/EoT_Scop.png)

EoT流程如下：

| 发送方 | 接收方 |
| --- | --- |
| 完成负载数据发送 | 接收负载数据 |
| 发送完最后一个bit后立刻翻转差分线状态，并保持THS-TRAIL时间 | \_ |
| 关闭HS-TX，使能LPTX，并进入LP11状态 | \_ |

*   进入Escape模式：LP11->LP10->LP00->LP01->LP00  
    进入Escape模式后，则可以通过发送8bit命令执行相应的功能，下表列出了可用的Escape命令：

| 功能 | 命令类型 | 命令 |
| --- | --- | --- |
| Low-Power Data Transmission | mode | 11100001 |
| Ultra-Low Power State | mode | 00011110 |
| Undefined-1 | mode | 10011111 |
| Undefined-2 | mode | 11011110 |
| Reset-Trigger | Trigger | 01100010 |
| Unknown-3 | Trigger | 01011101 |
| Unknown-4 | Trigger | 00100001 |
| Unknown-5 | Trigger | 10100000 |

1.  **Low-Power Data Transmission**

    LPDT功能下，数据能够在LP模式下进行低速传输，此时Clock的时钟需要小于20MHz。
    ![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android.10.Camera.01/LPDT.png)
    上图描述了LPDT下传输2Byte数据的时序。


1.  **Reset-Trigger**
    
    Reset-Trigger是发送方与接收方相互对应的通信形式，如果输入命令模式与Reset-Trigger命令相匹配，则在接收端通过逻辑PPI向协议标记一个

2.  **Ultra-Low-Power**
    
    发送方发送ULPS码后，接收方则可以根据这个信号进行自己的LowPower处理。在
    ULPS期间，Line始终处于LP00状态。若出现持续T<sub>WAKEUP</sub>的Mark-1状态后紧接着Stop状态则退出ULPS。

### **Clock Lane**

与Data Lane一样，Clock Lane也有两种模式，高速传输模式与低功耗模式。  
![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android.10.Camera.01/Clock_Lane.png)

很明显，Clock Lane进入与退出高速模式与Data Lane基本一致。  
下面两个表格描述了进入LowPower模式与HS模式的过程(结合上面的时序图与Lane模块的内部组成图一起看)

进入LP模式：

| 发送方 | 接收方 |
| --- | --- |
| 驱动HS Clock信号（HS-0与HS-1交替切换的这种状态） | 接收HS Clock信号 |
| 最后一个Data Lane进入LP模式 | \_ |
| 继续驱动HS Clock信号保持TCLK-POST时间并以HS-0结束 | \_ |
| 保持HS-0状态约TCLK-TRAIL时长 | 在TCLK-MISS检测时钟是否消失，并关闭HS-RX单元 |
| 关闭HS-TX单元，使能LP-TX单元，转变为停止状态LP-11并保持THS-EXIT | \_ |
| \_ | 检测到LP-11状态，关闭HS终端，进入停止模式 |

进入HS模式

| 发送方 | 接收方 |
| --- | --- |
| 驱动至Stop状态（LP-11） | 侦测Stop状态 |
| 驱动至HS-Req（LP-01）并保持TLPX | 侦测导线上是否出现LP-11到LP-01的变化 |
| 驱动至LP-00并保持TCLK-PREPARE | 侦测LP-01到LP-00的变化并在TCLK-TERM-EN后使能HS终端 |
| 使能高速驱动器同时关闭LP驱动器，保持HS-0状态TCLK-ZERO | 使能HS-RX并等待TCLK-SETTLE |
| \_ | 接收HS信号 |
| 在Data Lane启动之前，驱动HS Clock保持TCLK-PRE | 接收HS Clock信号 |

### 时序参数

接收方的DPHY能否成功捕捉到有效的数据取决于发送与接收两方之间的时序，下表列出了收发双方的所有时序参数，其中接收方的时序应该能够兼容发送方的时序参数。  
![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android.10.Camera.01/timing_param1.png)

![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android.10.Camera.01/timing_param2.png)

![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android.10.Camera.01/notes.png)

* * *

**多Lane的分发与合并**
---------------

当某些应用场景需要的带宽超过了单Lane所能提供的带宽或者为了降低Clock频率的情况下，则可以通过拓展Data Lane来满足要求。

下图展示了数据流在单lane与多lane的情况下，发送方内部的Lane Distribution Function（LDF）对来自LLP层的字节数据流的分发过程。

![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android.10.Camera.01/lane_distribution.png)

对于接收方，对应的拥有一个Lane Merging Function（LMF），用于将PHY层的多lane的数据合并为LLP层所需的字节流。

![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android.10.Camera.01/lane_merging.png)

在多Lane（定为N）传输的情况下，对于一帧数据，其数据长度不是Lane数量（N）的整数倍的情况下，则倒数第二组数据发出后会剩下少于N的字节需要发送，此时LDF会将无数据分配的Lane置为“Invalid Data”直接进入EoT。

下图展示了分发后的数据在2个Lane上的传输情况（包含两种情况）

![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android.10.Camera.01/lane_dis_dphy.png)

多Lane合并就是多Lane分发的相反过程，这里就不在赘述。

**低级协议（Low Level Protocol）**
----------------------------

LLP是基于字节，以包为单元的协议，支持长短两种包格式。

*   传输任意数据（负载独立）
*   以字节为数据元
*   对于DPHY，支持16路虚拟通道；对于CPHY，支持32路虚拟通道
*   支持独立的帧起始，帧结束，行起始，行结束等数据包
*   包含对数据类型，像素深度与格式的描述
*   16-bit Checksum错误侦测
*   6-bit 错误发现与纠正

LLP支持两种包结构，分别为长包与短包，包格式与长短取决于具体的物理层。无论何种类型的包，均由SoT信号开始，EoT信号结束。

![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android.10.Camera.01/llp_overview.png)

以DPHY为例来分析具体的协议格式，DPHY长包主要由包头、包负载、包尾三部分组成，具体如下图：  
![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android.10.Camera.01/llp.png)

短包协议格式如下图：

![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android.10.Camera.01/llp_short.png)

其中Data Type取值为0x10-0x37（见下图），虚拟通道由4bit构成，高2bit来自VCX，低2bit来自Data ID的bit6、7。

![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android.10.Camera.01/data_type_class.png)

上图可以发现，短包有两类Data Type

*   同步短包：用于对帧或行起停进行标识，起到同步作用。

![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android.10.Camera.01/sync_short.png)

*   通用短包：通用短包数据类型的目的是提供一种在数据流中打开/关闭快门、触发闪光灯等的机制。16bit短包数据域用于传输约定好的信息，实现对一些自定义功能的控制。

![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android.10.Camera.01/generic_short.png)

### **虚拟通道（virtual Channel ID）**

虚拟通道的作用是在交织传输的不同数据流中，区分出各个数据流所属的逻辑上的通道，以max9286为例，来自4路同轴线的相机数据可以设置为不同的虚拟通道，这样，在SOC侧CSI模块处理时，可以根据不同的虚拟通道ID将每个摄像头的数据转发至各自的内存区域，这样就能从4个地址获得单独的4个图像。若不使用虚拟通道，则4路数据就无法区分了（当然max9286内部能够将4个图像拼接为一个大图输出）。  
虚拟通道由包头中的VC与VCX构成，对于DPHY来说，由4bit组成，最高16路虚拟通道，高2bit来自VCX，低2bit来自VC。

![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android.10.Camera.01/vc.png)

### **数据类型（Data Type）**

数据类型表明了负载数据的格式和内容，上文提到，根据长短包的不同，数据类型共有8种不同的分类。短包数据类型的详细信息在上文已经介绍了，这里说明下长包的5种数据类型，详见下表：

![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android.10.Camera.01/data_format.png)

**像素字节转换**
----------

图像数据在LLP传输时需要以字节流的形式进行，应用层来的像素数据需要根据具体的像素格式进行转换，以YUV422 8bit为例：  
YUV422 8bit是以UYVY的顺序进行传输的，见下图：

![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android.10.Camera.01/yuv422_8bit.png)

YUV422 8bit的最小包传输单元如下表，其他长度的包长度必须为最小单元的整数倍：  
![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android.10.Camera.01/yuv422_packet.png)

为什么最小包单元是2pixels、4bytes、32bit而不是1pixels、2bytes、16bit呢？这是因为YUV422 8bit中两个Y分量共用一个UV分量，因此一Packet至少需要携带2个pixels的信息。

像素至字节的转换过程如下，从CSI-2标准文档提供的示意图可以看出，相对与原数据其实也没做啥转换，就统一了数据在LLP传输时高底位的发送顺序。

![](https://raw.githubusercontent.com/zhoujinjianz/zhoujinjian.com.images/master/Android.10.Camera.01/pixel_byte.png)