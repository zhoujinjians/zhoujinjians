---
title:  Android 10 Camera源码分析4：Rockchip ISP1 Driver 分析
cover: https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/post.cover.pictures/bing-wallpaper-2018.04.34.jpg
categories: 
  - Camera
tags:
  - Camera
toc: true
abbrlink: 20220215
date: 2022-02-15 02:15:00
---

注：文章都是通过阅读各位前辈总结的资料 Android 10.0 && Linux（Kernel 4.19）Rockchip平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除（◕‿◕），转载请注明出处（©Rockchip ©Android @Linux 版权所有），谢谢。
（==**文章基于 Kernel-4.19**==）&&（==**文章基于 Android 10.0**==）
 [【zhoujinjian.com博客原图链接】](https://github.com/zhoujinjiana) 
 [【开发板 Khadas Edge V】](https://www.khadas.cn/edge-v)
[【开发板 Khadas Edge V Android 10.0 && Linux（Kernel 4.19）源码链接】](https://github.com/khadas/linux/tree/khadas-edge-Qt)

正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢！！！

 [V4L2框架解析](https://deepinout.com/v4l2-tutorials/linux-v4l2-architecture.html) 
 [camera linux v4l2相关接口](https://blog.csdn.net/weixin_43503508/category_10257454.html) 
 [Rockchip ISP1 Driver](https://patchwork.ozlabs.org/project/devicetree-bindings/cover/1514533978-20408-1-git-send-email-zhengsq@rock-chips.com/) 


--------------------------------------------------------------------------------
==源码（部分）==：
**F:\Khadas_Edge_Android_Q\kernel\include\uapi\linux\**
**F:\Khadas_Edge_Android_Q\kernel\drivers\phy\rockchip\**
**F:\Khadas_Edge_Android_Q\kernel\drivers\media\platform\rockchip\isp1\**
**F:\Khadas_Edge_Android_Q\kernel\drivers\media\v4l2-core\**
**F:\Khadas_Edge_Android_Q\kernel\drivers\media\common\videobuf2\**

-------------------------------------------------------------------------------

## （一）、Rockchip-isp1介绍
#### 1、Overview

The document below provide basical informations about the rockchip-isp1 driver driver and Image Signal Processing block on Rockchip SoC with examples and details.

#### 2、Hardware

More detailed information could be found in TRM chapter "Image Signal Processing", "MIPI D-PHY" , but they are only available under NDA.

#### 3、ISP Details

ISP comprises with:

*   MIPI serial camera interface
*   Image Signal Processing
*   Many Image Enhancement Blocks
*   Crop
*   Resize

#### 4、Block diagram

The completed block diagram can not be pasted from datasheet to here, below diagram is an abstract version:

![](http://opensource.rock-chips.comhttps://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/thumb/4/48/Block.png/1200px-Block.png)
](/wiki_File:Block.png "Blocks.png")

#### 5、MIPI Details

There are three D-PHY instances in rockchip SoC, their connection are shown as following figure:
![](http://opensource.rock-chips.comhttps://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/thumb/a/ac/Phy.png/600px-Phy.png)

#### 6、Software


#### 7、Driver

rockchip-isp1 is a V4L2 based driver for Image Signal Processing block on Rockchip SoC. Compared to the earlier driver rk-isp10, it use Media Controller framework, which it's different from plain v4L2. While the plain v4L2 had a view of the device as a plain DMA based image drabber which connects the input data to the host memory, the Media Controller takes into consideration the fact that a typical video device might consist of multiple sub-devices.

#### 8、Media Controller Basics

Please read below link carefully, especially if you don't have use it before.

[https://linuxtv.org/downloads/v4l-dvb-apis/uapi/mediactl/media-controller-intro.html](https://linuxtv.org/downloads/v4l-dvb-apis/uapi/mediactl/media-controller-intro.html)

#### 9、Block diagram

#### 10、File view

![](http://opensource.rock-chips.comhttps://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/thumb/3/38/Code.png/1200px-Code.png)

#### 11、V4l2 view

![](http://opensource.rock-chips.comhttps://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/thumb/7/74/Rockchip-isp1-v4l2.png/800px-Rockchip-isp1-v4l2.png)
| Name | Type | Description |
| --- | --- | --- |
| rkisp1_mainpath | v4l2_vdev, capture | Format: YUV, RAW Bayer; Max resolution: 4416\*3312; Support: Crop  |
| rkisp1_selfpath | v4l2_vdev, capture | Format: YUV, RGB; Max resolution: 1920\*1080; Support: Crop |
| rkisp1-isp-subdev | v4l2_subdev |  Internal isp blocks; Support: source/sink pad crop. The format on sink pad should be equal to sensor input format, the size should be equal/less than sensor input size. The format on source pad should be equal to vdev output format if output format is raw bayer, otherwise it should be  YUYV2X8. The size should be equal/less than sink pad size.  |
| rockchip-sy-mipi-dphy | v4l2_subdev |  MIPI-DPHY Configure  |
| rkisp1-statistics | v4l2_vdev, capture |  Provide Image color Statistics information.  |
| rkisp1-input-params | v4l2_vdev, output |  Accept params for AWB, BLC...... Image enhancement blcoks  |

#### 12、Sensor Driver Requirement

The sensor driver should implement controls in the following table. 

| Controls | Description | R/W | Needed by |
| --- | --- | --- | --- |
| V4L2_CID_VBLANK | Vertical blanking. The idle period after every frame during which no image data is produced. The unit of vertical blanking is a line. Every line has length of the image width plus horizontal blanking at the pixel rate defined by `V4L2_CID_PIXEL_RATE` control in the same sub-device. | R | Ae |
| V4L2_CID_HBLANK | Horizontal blanking. The idle period after every line of image data during which no image data is produced. The unit of horizontal blanking is pixels. | R | Ae |
| V4L2_CID_EXPOSURE | Determines the exposure time of the camera sensor. The exposure time is limited by the frame interval. Drivers should interpret the values as 100 µs units, where the value 1 stands for 1/10000th of a second, 10000 for 1 second and 100000 for 10 seconds. | R/W | Ae |
| V4L2_CID_ANALOGUE_GAIN | Analogue gain is gain affecting all colour components in the pixel matrix. The gain operation is performed in the analogue domain before A/D conversion. | R/W | Ae |
| V4L2_CID_DIGITAL_GAIN | Digital gain is the value by which all colour components are multiplied by. Typically the digital gain applied is the control value divided by e.g. 0x100, meaning that to get no digital gain the control value needs to be 0x100. The no-gain configuration is also typically the default. | R/W | Ae |
| V4L2_CID_PIXEL_RATE | Pixel rate in the source pads of the subdev. This control is read-only and its unit is pixels / second. | R | Ae |
| V4L2_CID_LINK_FREQ | Data bus frequency. Together with the media bus pixel code, bus type (clock cycles per sample), the data bus frequency defines the pixel rate (`V4L2_CID_PIXEL_RATE`) in the pixel array (or possibly elsewhere, if the device is not an image sensor). The frame rate can be calculated from the pixel clock, image width and height and horizontal and vertical blanking. While the pixel rate control may be defined elsewhere than in the subdev containing the pixel array, the frame rate cannot be obtained from that information. This is because only on the pixel array it can be assumed that the vertical and horizontal blanking information is exact: no other blanking is allowed in the pixel array. The selection of frame rate is performed by selecting the desired horizontal and vertical blanking. The unit of this control is Hz. | R | mipi-dphy |

#### 13、Devicetree Bindings

[https://github.com/torvalds/linux/tree/master/Documentation/devicetree/bindings/media/rockchip-isp1.txt](https://github.com/wzyy2/linux/blob/rkisp1/Documentation/devicetree/bindings/media/rockchip-isp1.txt)

[https://github.com/torvalds/linux/tree/master/Documentation/devicetree/bindings/media/rockchip-mipi-dphy.txt](https://github.com/wzyy2/linux/blob/rkisp1/Documentation/devicetree/bindings/media/rockchip-mipi-dphy.txt)

  

#### 14、Linux

#### 15、User applications 

The v4l-utils tool and applications   
The v4l-utils tool is a V4L2 development kit maintained by Linuxtv\[1\] . It provides a set   
of V4L2 and media framework related tools for configuring V4L2 sub-device properties,   
testing V4L2 devices, and providing development libraries such as libv4l2.so and so on.   
This chapter mainly introduces two command-line tools in v4l-utils: media-ctl and v4l2-   
ctl   
 media-ctl, used to view and configure topology   
 v4l2-ctl, used to configure v4l2 controls, capture frames, set cif, isp, sensor   
parameters   
The format code of different versions of v4l-utils will be different, especially mbus-fmt   
part. The version used in this document is v4l-utils-1.14.1 integrated in Linux SDK. 

#### 16、Android


#### 17、Camera Hal3

[https://source.android.com/devices/camera/camera3](https://source.android.com/devices/camera/camera3?hl=zh-cn)


#### 18、Topology

``` c
media-ctl --device /dev/media0 --print-topology

Opening media device /dev/media0
Enumerating entities
Found 9 entities
Enumerating pads and links
Media controller API version 0.0.111

Media device information
------------------------
driver          rkisp1
model           rkisp1
serial
bus info
hw revision     0x0
driver version  0.0.111

Device topology
- entity 1: rkisp1-isp-subdev (4 pads, 6 links)
            type V4L2 subdev subtype Unknown
            device node name /dev/v4l-subdev0
        pad0: Sink
                [fmt:SBGGR10/4208x3120
                 crop.bounds:(0,0)/4208x3120
                 crop:(0,0)/4208x3120]
                <- "rkisp1_dmapath":0 []
                <- "rockchip-mipi-dphy-rx":1 [ENABLED]
        pad1: Sink
                <- "rkisp1-input-params":0 [ENABLED]
        pad2: Source
                [fmt:YUYV2X8/4208x3120
                 crop.bounds:(0,0)/4208x3120
                 crop:(0,0)/4208x3120]
                -> "rkisp1_selfpath":0 [ENABLED]
                -> "rkisp1_mainpath":0 [ENABLED]
        pad3: Source
                -> "rkisp1-statistics":0 [ENABLED]

- entity 6: rkisp1_mainpath (1 pad, 1 link)
            type Node subtype V4L
            device node name /dev/video0
        pad0: Sink
                <- "rkisp1-isp-subdev":2 [ENABLED]

- entity 10: rkisp1_selfpath (1 pad, 1 link)
             type Node subtype V4L
             device node name /dev/video1
        pad0: Sink
                <- "rkisp1-isp-subdev":2 [ENABLED]

- entity 14: rkisp1_dmapath (1 pad, 1 link)
             type Node subtype V4L
             device node name /dev/video2
        pad0: Source
                -> "rkisp1-isp-subdev":0 []

- entity 20: rkisp1-statistics (1 pad, 1 link)
             type Node subtype V4L
             device node name /dev/video3
        pad0: Sink
                <- "rkisp1-isp-subdev":3 [ENABLED]

- entity 24: rkisp1-input-params (1 pad, 1 link)
             type Node subtype V4L
             device node name /dev/video4
        pad0: Source
                -> "rkisp1-isp-subdev":1 [ENABLED]

- entity 28: rockchip-mipi-dphy-rx (2 pads, 2 links)
             type V4L2 subdev subtype Unknown
             device node name /dev/v4l-subdev1
        pad0: Sink
                [fmt:SBGGR10/4208x3120]
                <- "m00_b_imx214 6-0010":0 [ENABLED]
        pad1: Source
                [fmt:SBGGR10/4208x3120]
                -> "rkisp1-isp-subdev":0 [ENABLED]

- entity 31: m00_b_imx214 6-0010 (1 pad, 1 link)
             type V4L2 subdev subtype Sensor
             device node name /dev/v4l-subdev2
        pad0: Source
                [fmt:SBGGR10/4208x3120]
                -> "rockchip-mipi-dphy-rx":0 [ENABLED]

- entity 35: m00_b_vm149c 6-000c (0 pad, 0 link)
             type V4L2 subdev subtype Lens
             device node name /dev/v4l-subdev3

```



## （二）、Rockchip-isp1 driver分析

本章主要介绍 RKisp1 的驱动代码结构、dts 配置、debug 方法等。ISP 驱动符合 media controller，v4l2，vb2 框架，与 mipi-dphy，Sensor 相互独立且通过异步注册。

### （1）、Camera 软件驱动目录说明

![](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/Android.10.Camera.04/isp1code_source.png)

#### 1、框架简要说明

RKISP 驱动主要是依据 v4l2 / media framework 实现硬件的配置、中断处理、控制 buffer轮转，以及控制 subdevice(如 mipi dphy 及 sensor)的上下电等功能。
下面的框图描述了 RKISP1 驱动的拓扑结构。
![](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/Android.10.Camera.04/1624498395236.png)

| Name | Type | Description |
| --- | --- | --- |
| rkisp1_mainpath | v4l2_vdev, capture | Format: YUV, RAW Bayer; Max resolution: 4416\*3312; Support: Crop  |
| rkisp1_selfpath | v4l2_vdev, capture | Format: YUV, RGB; Max resolution: 1920\*1080; Support: Crop |
| rkisp1-isp-subdev | v4l2_subdev |  Internal isp blocks; Support: source/sink pad crop. The format on sink pad should be equal to sensor input format, the size should be equal/less than sensor input size. The format on source pad should be equal to vdev output format if output format is raw bayer, otherwise it should be  YUYV2X8. The size should be equal/less than sink pad size.  |
| rockchip-sy-mipi-dphy | v4l2_subdev |  MIPI-DPHY Configure  |
| rkisp1-statistics | v4l2_vdev, capture |  Provide Image color Statistics information.  |
| rkisp1-input-params | v4l2_vdev, output |  Accept params for AWB, BLC...... Image enhancement blcoks  |

#### 2、驱动版本号获取方式
·从 kernel 启动 log 中获取
``` c
rkisp1 ff910000.rkisp1: rkisp1 driver version: v00.01.05
```
·由以下命令获取
``` c
cat /sys/module/video_rkisp1/parameters/version
```

#### 3、RKisp1 DTS配置
RKisp1 的 DTS binding 也有完整的 Document 位于Documentation/devicetree/bindings/media/rockchip-isp1.txt。请首先参考该文档，它会跟驱动保持同步更新。在 RK Linux SDK 发布时， 若芯片支持 ISP，其 dtsi 中已经有定义好 rkisp1
节点，如rk3399.dtsi 中的 rkisp1_0，rkisp1_1 节点。下表描述各芯片 ISP 的信息。
![](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/Android.10.Camera.04/RK3399_rkisp1_0.png)

注意：
-  RK3399.dtsi 中也注册了多个 isp 节点，请注意本章所描述的是 rkisp1_0 及 rkisp1_1
-  Iommu 需要和 isp 一同 enable 或 disable
-  如有多个 isp（比如 rk3399），请注意 iommu 节点要选

##### 3.1、 RKisp1 DTS的板级配置
isp、mipi-dphy、sensor 是单独定义的节点， 三者之间通过
remote endpoint 相互建立连接。在结构框图上，mipi-dphy 连接了 isp 及 sensor。以下示例是
基于 RK3399 edge-v 上的 dts 板级配置，可以参考文件F:\Khadas_Edge_Android_Q\kernel\arch\arm64\boot\dts\rockchip\rk3399-khadas-edge-android.dts。
在该板子上， RK3399 的两个 ISP 都启用了，这里仅举例 rkisp1_0。
> 首先， 将 rkisp1_0 节点设置为”okay”状态。
``` c
F:\Khadas_Edge_Android_Q\kernel\arch\arm64\boot\dts\rockchip\rk3399.dtsi
	rkisp1_0: rkisp1@ff910000 {
		compatible = "rockchip,rk3399-rkisp1";
		reg = <0x0 0xff910000 0x0 0x4000>;
		interrupts = <GIC_SPI 43 IRQ_TYPE_LEVEL_HIGH 0>;
		interrupt-names = "isp_irq";
		clocks = <&cru SCLK_ISP0>,
			 <&cru ACLK_ISP0>, <&cru HCLK_ISP0>,
			 <&cru ACLK_ISP0_WRAPPER>, <&cru HCLK_ISP0_WRAPPER>;
		clock-names = "clk_isp",
			 "aclk_isp", "hclk_isp",
			 "aclk_isp_wrap", "hclk_isp_wrap";
		devfreq = <&dmc>;
		power-domains = <&power RK3399_PD_ISP0>;
		iommus = <&isp0_mmu>;
		status = "disabled";
	};
F:\Khadas_Edge_Android_Q\kernel\arch\arm64\boot\dts\rockchip\rk3399-khadas-edge-android.dts
&rkisp1_0 {
	status = "okay";

	port {
		#address-cells = <1>;
		#size-cells = <0>;

		isp0_mipi_in: endpoint@0 {
			reg = <0>;
			remote-endpoint = <&dphy_rx0_out>;
		};
	};
};
```
注意，
- 其它版本的 isp 驱动的节点状态需要 disabled。否则两套不同的驱动会冲突
- Port 节点中定义了该 rkisp1_0 是与 dphy_rx0_out 

> 第二，rkisp1_0 对应的 iommu 也需要是”okay”状态。

``` c
&isp0_mmu {
	status = "okay";
};
```
> 第三，该 sensor 是 mipi 接口，因此需要把 mipi dph也启用。

``` c
F:\Khadas_Edge_Android_Q\kernel\arch\arm64\boot\dts\rockchip\rk3399-khadas-edge-android.dts
&mipi_dphy_rx0 {
	status = "okay";

	ports {
		#address-cells = <1>;
		#size-cells = <0>;

		port@0 {
			reg = <0>;
			#address-cells = <1>;
			#size-cells = <0>;

			mipi_in_ucam0: endpoint@1 {
				reg = <1>;
				remote-endpoint = <&ucam_out0>;
				data-lanes = <1 2>;
			};
		};

		port@1 {
			reg = <1>;
			#address-cells = <1>;
			#size-cells = <0>;

			dphy_rx0_out: endpoint@0 {
				reg = <0>;
				remote-endpoint = <&isp0_mipi_in>;
			};
		};
	};
};

```
注意：
- Dphy 既连接 isp，也连接sensor。因此它有两个endpoint。mipi_in_ucam0是与Sensor
相连接，dphy_rx0_out 与 isp 相连接
- mipi_in_ucam0 中声明了 data-lanes，指明了 sensor 使用了两个 lane。如果有 4 个
lanes，这里就应该定义成<1 2 3 4>。以此类推。

最后板级的sensor节点定义好，并且也需要声明port子节点，与mipi dphy的 dphy_rx0_out
相连接。
#### 4、rkisp1_plat_probe

``` c
F:\Khadas_Edge_Android_Q\kernel\drivers\media\platform\rockchip\isp1\dev.c
static int rkisp1_plat_probe(struct platform_device *pdev)
{
......
//初始化isp_dev rkisp1_pipeline 函数
	isp_dev->pipe.open = rkisp1_pipeline_open;
	isp_dev->pipe.close = rkisp1_pipeline_close;
	isp_dev->pipe.set_stream = rkisp1_pipeline_set_stream;
//初始化stream
	rkisp1_stream_init(isp_dev, RKISP1_STREAM_SP);
	rkisp1_stream_init(isp_dev, RKISP1_STREAM_MP);
	rkisp1_stream_init(isp_dev, RKISP1_STREAM_RAW);
//media_dev.ops
	isp_dev->media_dev.dev = &pdev->dev;
	isp_dev->media_dev.ops = &rkisp1_media_ops;
	v4l2_dev = &isp_dev->v4l2_dev;
	v4l2_dev->mdev = &isp_dev->media_dev;
//初始化ctrl_handler
	strlcpy(v4l2_dev->name, "rkisp1", sizeof(v4l2_dev->name));
	v4l2_ctrl_handler_init(&isp_dev->ctrl_handler, 5);
	v4l2_dev->ctrl_handler = &isp_dev->ctrl_handler;
//v4l2_device_register
	ret = v4l2_device_register(isp_dev->dev, &isp_dev->v4l2_dev);
//media_device_register,此处会生成/dev/media0
	media_device_init(&isp_dev->media_dev);
	ret = media_device_register(&isp_dev->media_dev);
//rkisp1_register_platform_subdevs
	/* create & register platefom subdev (from of_node) */
	ret = rkisp1_register_platform_subdevs(isp_dev);	
}	
```
#### 4.1、v4l2_device_register
v4l2的设备注册其实没有注册设备或者设备驱动，只是将v4l2的大结构体与其他设备进行捆绑。如下面是v4l2注册设备的过程。

``` c
F:\Khadas_Edge_Android_Q\kernel\drivers\media\v4l2-core\v4l2-device.c
int v4l2_device_register(struct device *dev, struct v4l2_device *v4l2_dev)
{
	if (v4l2_dev == NULL)
		return -EINVAL;

	INIT_LIST_HEAD(&v4l2_dev->subdevs);
	spin_lock_init(&v4l2_dev->lock);
	mutex_init(&v4l2_dev->ioctl_lock);
	v4l2_prio_init(&v4l2_dev->prio);
	kref_init(&v4l2_dev->ref);
	//上面都是做一些初始化工作，
	get_device(dev);
	//下面将当前设备的对象，赋值给v4l2_dev->dev中，这样的话当前的设备就成了v4l2设备，由于当前设备已经注册过了设备文件，所以后续不需要在重新注册设备文件。
	v4l2_dev->dev = dev;
	if (dev == NULL) {
		/* If dev == NULL, then name must be filled in by the caller */
		if (WARN_ON(!v4l2_dev->name[0]))
			return -EINVAL;
		return 0;
	}

	/* Set name to driver name + device name if it is empty. */
	/* 如果v4l2设备的名字为空，则会将当前设备的名字拷贝为v4l2设备中。*/
	if (!v4l2_dev->name[0])
		snprintf(v4l2_dev->name, sizeof(v4l2_dev->name), "%s %s",
			dev->driver->name, dev_name(dev));
	//下面是非常重要的一步，即将v4l2设备对象，保存到dev设备中，这个dev可以是platform、char、block等设，我们可以使用dev_get_drvdata(dev)获取到v4l2对象。
	if (!dev_get_drvdata(dev))
		dev_set_drvdata(dev, v4l2_dev);
	return 0;
}
```
  这里上面代码比较简单，这需要知道此函数功能是将当前设备和v4l2设备捆绑，并保存v4l2设备对象到当前设备的driverdata中。**此函数的作用就是将当前设备驱动和v4l2对象进行捆绑，便于挂接子设备**

#### 4.2、video_register_device注册过程

##### 4.2.1、注册过程

``` c
F:\Khadas_Edge_Android_Q\kernel\include\media\v4l2-dev.h
/**
 *  video_register_device - register video4linux devices
 *
 * @vdev: struct video_device to register
 * @type: type of device to register, as defined by &enum vfl_devnode_type
 * @nr:   which device node number is desired:
 *	(0 == /dev/video0, 1 == /dev/video1, ..., -1 == first free)
 *
 * Internally, it calls __video_register_device(). Please see its
 * documentation for more details.
 *
 * .. note::
 *	if video_register_device fails, the release() callback of
 *	&struct video_device structure is *not* called, so the caller
 *	is responsible for freeing any data. Usually that means that
 *	you video_device_release() should be called on failure.
 */
static inline int __must_check video_register_device(struct video_device *vdev,
						     enum vfl_devnode_type type,
						     int nr)
{
	return __video_register_device(vdev, type, nr, 1, vdev->fops->owner);
}
```

上面方法又封装了一层，转而调用下面的方法

``` c
F:\Khadas_Edge_Android_Q\kernel\drivers\media\v4l2-core\v4l2-dev.c
int __video_register_device(struct video_device *vdev,
			    enum vfl_devnode_type type,
			    int nr, int warn_if_nr_in_use,
			    struct module *owner)
{
	int i = 0;
	int ret;
	int minor_offset = 0;
	int minor_cnt = VIDEO_NUM_DEVICES;
	const char *name_base;

	/* A minor value of -1 marks this video device as never
	   having been registered */
	vdev->minor = -1;

	/* the release callback MUST be present */
	if (WARN_ON(!vdev->release))
		return -EINVAL;
	/* the v4l2_dev pointer MUST be present */
	if (WARN_ON(!vdev->v4l2_dev))
		return -EINVAL;

	/* v4l2_fh support */
	spin_lock_init(&vdev->fh_lock);
	INIT_LIST_HEAD(&vdev->fh_list);

	/* Part 1: check device type */
	switch (type) {
	case VFL_TYPE_GRABBER:
		name_base = "video";
		break;
	case VFL_TYPE_VBI:
		name_base = "vbi";
		break;
	case VFL_TYPE_RADIO:
		name_base = "radio";
		break;
	case VFL_TYPE_SUBDEV:
		name_base = "v4l-subdev";
		break;
	case VFL_TYPE_SDR:
		/* Use device name 'swradio' because 'sdr' was already taken. */
		name_base = "swradio";
		break;
	case VFL_TYPE_TOUCH:
		name_base = "v4l-touch";
		break;
	default:
		pr_err("%s called with unknown type: %d\n",
		       __func__, type);
		return -EINVAL;
	}

	vdev->vfl_type = type;
	vdev->cdev = NULL;
	if (vdev->dev_parent == NULL)
		vdev->dev_parent = vdev->v4l2_dev->dev;
	if (vdev->ctrl_handler == NULL)
		vdev->ctrl_handler = vdev->v4l2_dev->ctrl_handler;
	/* If the prio state pointer is NULL, then use the v4l2_device
	   prio state. */
	if (vdev->prio == NULL)
		vdev->prio = &vdev->v4l2_dev->prio;

	/* Part 2: find a free minor, device node number and device index. */
#ifdef CONFIG_VIDEO_FIXED_MINOR_RANGES
	/* Keep the ranges for the first four types for historical
	 * reasons.
	 * Newer devices (not yet in place) should use the range
	 * of 128-191 and just pick the first free minor there
	 * (new style). */
	switch (type) {
	case VFL_TYPE_GRABBER:
		minor_offset = 0;
		minor_cnt = 64;
		break;
	case VFL_TYPE_RADIO:
		minor_offset = 64;
		minor_cnt = 64;
		break;
	case VFL_TYPE_VBI:
		minor_offset = 224;
		minor_cnt = 32;
		break;
	default:
		minor_offset = 128;
		minor_cnt = 64;
		break;
	}
#endif

	/* Pick a device node number */
	mutex_lock(&videodev_lock);
	nr = devnode_find(vdev, nr == -1 ? 0 : nr, minor_cnt);
	if (nr == minor_cnt)
		nr = devnode_find(vdev, 0, minor_cnt);
	if (nr == minor_cnt) {
		pr_err("could not get a free device node number\n");
		mutex_unlock(&videodev_lock);
		return -ENFILE;
	}
#ifdef CONFIG_VIDEO_FIXED_MINOR_RANGES
	/* 1-on-1 mapping of device node number to minor number */
	i = nr;//这里保存下空闲的次设备号
#else
	/* The device node number and minor numbers are independent, so
	   we just find the first free minor number. */
	/* 查找到第一个空闲的minor号,可以理解成是次设备号*/   
	for (i = 0; i < VIDEO_NUM_DEVICES; i++)
		if (video_devices[i] == NULL)//所有注册的video_device都会保存到这个数组中
			break;
	if (i == VIDEO_NUM_DEVICES) {
		mutex_unlock(&videodev_lock);
		pr_err("could not get a free minor\n");
		return -ENFILE;
	}
#endif
//minor_offset一般是0，i就是查找到的空闲次设备号，这里总的次设备支持到256个
	vdev->minor = i + minor_offset;
//这里num会保存到session_id中，唯一表示一个任务。一般情况下nr == minor	
	vdev->num = nr;//将标准位置为已用。

	/* Should not happen since we thought this minor was free */
	if (WARN_ON(video_devices[vdev->minor])) {
		mutex_unlock(&videodev_lock);
		pr_err("video_device not empty!\n");
		return -ENFILE;
	}
	devnode_set(vdev);
	/* 下面这个方法字面意思看起来是获取一个index，但是查看源代码会发现
	   “这里会去查找不是直系的设备空闲号”，就是说只有video_device不为空，而且v4l2 parent
	   对象相同都会认为是同类，直接跳过相应的index号*/	
	vdev->index = get_index(vdev);
/* 下面要重点了，这里根据次设备号将当前video_device保存到video_device[]数组中*/	
	video_devices[vdev->minor] = vdev;
	mutex_unlock(&videodev_lock);

	if (vdev->ioctl_ops)
		determine_valid_ioctls(vdev);
/* 分配字符设备文件*/
	/* Part 3: Initialize the character device */
	vdev->cdev = cdev_alloc();
	if (vdev->cdev == NULL) {
		ret = -ENOMEM;
		goto cleanup;
	}//务必留意这个ioctl,后面子设备中的ioctl都是通过这里查找的。
	vdev->cdev->ops = &v4l2_fops;
	vdev->cdev->owner = owner;
	ret = cdev_add(vdev->cdev, MKDEV(VIDEO_MAJOR, vdev->minor), 1);
	if (ret < 0) {
		pr_err("%s: cdev_add failed\n", __func__);
		kfree(vdev->cdev);
		vdev->cdev = NULL;
		goto cleanup;
	}

	/* Part 4: register the device with sysfs */
	vdev->dev.class = &video_class;//根目录sys中有对应的属性可操作。
	vdev->dev.devt = MKDEV(VIDEO_MAJOR, vdev->minor);//主设备号为81
	vdev->dev.parent = vdev->dev_parent;//这里一般是v4l2_device对象
	dev_set_name(&vdev->dev, "%s%d", name_base, vdev->num);
	ret = device_register(&vdev->dev);//注册该video_device到kernel中。
	if (ret < 0) {
		pr_err("%s: device_register failed\n", __func__);
		goto cleanup;
	}
	/* Register the release callback that will be called when the last
	   reference to the device goes away. */
	vdev->dev.release = v4l2_device_release;

	if (nr != -1 && nr != vdev->num && warn_if_nr_in_use)
		pr_warn("%s: requested %s%d, got %s\n", __func__,
			name_base, nr, video_device_node_name(vdev));
/* 下面这一步很关键，增加v4l2设备对象引用计数*/
	/* Increase v4l2_device refcount */
	v4l2_device_get(vdev->v4l2_dev);

	/* Part 5: Register the entity. */
	ret = video_register_media_controller(vdev);

	/* Part 6: Activate this minor. The char device can now be used. */
	set_bit(V4L2_FL_REGISTERED, &vdev->flags);

	return 0;

cleanup:
	mutex_lock(&videodev_lock);
	if (vdev->cdev)
		cdev_del(vdev->cdev);
	video_devices[vdev->minor] = NULL;
	devnode_clear(vdev);
	mutex_unlock(&videodev_lock);
	/* Mark this video device as never having been registered. */
	vdev->minor = -1;
	return ret;
}
```

  由这里可以发现，创建video_device时，也创建了一个字符设备。并将该设备的parent节点指定为v4l2_device所依附的那个节点。主要需要注意下面几点。
  *   1.根据设备类型确定设备名字和次设备数量  
    由于系统可能包含很多媒体设备，所以v4l2核心将0～255次设备编号划分了区域如下所示：其中VFL_TYPE_GRABBER一般表示提供数据的设备如camera.

| 类型 | 次设备号区间 | 设备基名称 |
| --- | --- | --- |
| VFL_TYPE_GRABBER | 0~63 | “video” |
| VFL_TYPE_VBI | 224～255 | “vbi” |
| VFL_TYPE_RADIO | 64~127 | “radio” |
| 其它(含VFL_TYPE_SUBDEV) | 128~223 | 含“v4l-subdev” |

*   2.确定设备编号，注册字符设备驱动。这里我曾今迷糊了,我以为下面这会注册２个设备到kernel中，后来才发现我错了。这是注册字符设备的一种方式。

``` c
	ret = cdev_add(vdev->cdev, MKDEV(VIDEO_MAJOR, vdev->minor), 1);
	dev_set_name(&vdev->dev, "%s%d", name_base, vdev->num);
	ret = device_register(&vdev->dev);
```

*   3.确定设备的入口

``` c
	/* Part 5: Register the entity. */
	ret = video_register_media_controller(vdev);
```

##### 4.2.2、video在系统中的位置

```
	struct v4l2_device ------|                      struct video_device   
		|                    |------------引用----------> |----*v4l2_dev
        |                                                |
		|-->ctrlhandler<------------挂载------------------|----ctrl_handler
		|--->media->entry-list<------------挂载-----------|----->entry


```

#### 4.3、media_device_register()多媒体设备注册

其实这里又用宏转了一下,THIS_MODULE就是代表当前模块的struct module对象，也就代表当前驱动的实例对象。

```c
F:\Khadas_Edge_Android_Q\kernel\include\media\media-device.h
#define media_device_register(mdev) __media_device_register(mdev, THIS_MODULE)
__media_device_register(mdev, THIS_MODULE)

```

```c
F:\Khadas_Edge_Android_Q\kernel\drivers\media\media-device.c
/**
 * media_device_register - register a media device
 * @mdev:	The media device
 *
 * The caller is responsible for initializing the media device before
 * registration. The following fields must be set:
 *
 * - dev must point to the parent device
 * - model must be filled with the device model name
 */
int __must_check __media_device_register(struct media_device *mdev,
					 struct module *owner)
{
	int ret;  if (WARN_ON(mdev->dev == NULL || mdev->model[0] == 0))
		return -EINVAL;

	mdev->entity_id = 1;
	INIT_LIST_HEAD(&mdev->entities);
	spin_lock_init(&mdev->lock);
	mutex_init(&mdev->graph_mutex);
    /*上面同样是一些变量的初始化*/
	/* Register the device node. */
	mdev->devnode.fops = &media_device_fops;
	/*下面是绑定父设备，同上面的v4l2_devcie绑定的是同一个struct devie对象*/
	mdev->devnode.parent = mdev->dev;
	mdev->devnode.release = media_device_release;
	ret = media_devnode_register(&mdev->devnode, owner);
    //下面可以发现创建了属性文件，我们可以直接通过属性文件来操作这个文件节点。
    //但是可惜的是，在属性接口中只发现了下面这样的代码，只打印了media_device的描述信息。
    //“sprintf(buf, "%.*s\n", (int)sizeof(mdev->model), mdev->model);”
	ret = device_create_file(&mdev->devnode.dev, &dev_attr_model);
	//省略一些错误检测
	return 0;
}

```
上面初始化了一些media_devnode的设备域。便于后续标准接口的调用。紧接着重量级的操作来了，注释部分写的也很精彩。

```c
F:\Khadas_Edge_Android_Q\kernel\drivers\media\media-devnode.c
/**
 * media_devnode_register - register a media device node
 * @mdev: media device node structure we want to register
 *
 * The registration code assigns minor numbers and registers the new device node
 * with the kernel. An error is returned if no free minor number can be found,
 * or if the registration of the device node fails.
 *
 * Zero is returned on success.
 * * Note that if the media_devnode_register call fails, the release() callback of
 * the media_devnode structure is *not* called, so the caller is responsible for
 * freeing any data.
 */
int __must_check media_devnode_register(struct media_devnode *mdev,
					struct module *owner)
{
	int minor;
	int ret;

	/* Part 1: Find a free minor number */
	mutex_lock(&media_devnode_lock);
	//获取可用的次设备编号。
	minor = find_next_zero_bit(media_devnode_nums, MEDIA_NUM_DEVICES, 0);
	if (minor == MEDIA_NUM_DEVICES) {
		mutex_unlock(&media_devnode_lock);
		pr_err("could not get a free minor\n");
		return -ENFILE;
	}

	set_bit(minor, media_devnode_nums);
	mutex_unlock(&media_devnode_lock);
    //注意下面保存了次设备号，便于后续打开设备节点。 
	mdev->minor = minor;

	/* Part 2: Initialize and register the character device */
	cdev_init(&mdev->cdev, &media_devnode_fops);
	mdev->cdev.owner = owner;

	ret = cdev_add(&mdev->cdev, MKDEV(MAJOR(media_dev_t), mdev->minor), 1);
	if (ret < 0) {
		pr_err("%s: cdev_add failed\n", __func__);
		goto error;
	}

	/* Part 3: Register the media device */
	mdev->dev.bus = &media_bus_type;
	mdev->dev.devt = MKDEV(MAJOR(media_dev_t), mdev->minor);
	mdev->dev.release = media_devnode_release;
	if (mdev->parent)
		mdev->dev.parent = mdev->parent;
	dev_set_name(&mdev->dev, "media%d", mdev->minor);
	ret = device_register(&mdev->dev);
	if (ret < 0) {
		pr_err("%s: device_register failed\n", __func__);
		goto error;
	}

	/* Part 4: Activate this minor. The char device can now be used. */
	set_bit(MEDIA_FLAG_REGISTERED, &mdev->flags);

	return 0;

error:
	cdev_del(&mdev->cdev);
	clear_bit(mdev->minor, media_devnode_nums);
	return ret;
}

```

上面代码注册了一个字符设备驱动，上面代码注释可以发现函数分成4部分功能.

*   part1:这一部分功能就是找一个空闲的次设备号。  
    1.MEDIA_NUM_DEVICES:该宏表示系统最大支持256个子设备.  
    2.media_devnode_nums:该全局变量用来表示所有位号是否已经使用了，这里可以理解成这个变量有256位，每一位都是一个标志。例如：我们得到一个次设备号是5，则该变量的第5位就会被置位。
*   part2:这里会根据主设备号media_dev_t和次设备号，创建字符设备。并将该字符设备的owner赋值为当前模块，然后添加到字符设备集合中。
*   part3：注册设备
*   part4：置标志变量，表示该字符设备可以使用了。
#### 4.4、rkisp1_register_platform_subdevs

``` c
F:\Khadas_Edge_Android_Q\kernel\drivers\media\platform\rockchip\isp1\dev.c
static int rkisp1_register_platform_subdevs(struct rkisp1_device *dev)
{
	int ret;
	printk("zjj.android10.kernel.camera || %s %s %d \n",
		__FUNCTION__, __FILE__, __LINE__); //zhoujinjian
    //此处注册rkisp1-isp-subdev
	ret = rkisp1_register_isp_subdev(dev, &dev->v4l2_dev);
	if (ret < 0)
		return ret;
    //此处注册rkisp1_mainpath rkisp1_selfpath
	ret = rkisp1_register_stream_vdevs(dev);
	if (ret < 0)
		goto err_unreg_isp_subdev;
    //此处注册rkisp1_dmapath
	ret = rkisp1_register_dmarx_vdev(dev);
	if (ret < 0)
		goto err_unreg_stream_vdev;
    //此处注册rkisp1-statistics
	ret = rkisp1_register_stats_vdev(&dev->stats_vdev, &dev->v4l2_dev, dev);
	if (ret < 0)
		goto err_unreg_dmarx_vdev;
    //此处注册rkisp1-input-params
	ret = rkisp1_register_params_vdev(&dev->params_vdev, &dev->v4l2_dev,
					  dev);
	if (ret < 0)
		goto err_unreg_stats_vdev;

	ret = isp_subdev_notifier(dev);
	if (ret < 0) {
		v4l2_err(&dev->v4l2_dev,
			 "Failed to register subdev notifier(%d)\n", ret);
		goto err_unreg_params_vdev;
	}

	return 0;
}

```

#### 4.5、rkisp1_register_stream_vdev

``` c
F:\Khadas_Edge_Android_Q\kernel\drivers\media\platform\rockchip\isp1\capture.c
static int rkisp1_register_stream_vdev(struct rkisp1_stream *stream)
{
	struct rkisp1_device *dev = stream->ispdev;
	struct v4l2_device *v4l2_dev = &dev->v4l2_dev;
	struct video_device *vdev = &stream->vnode.vdev;
	struct rkisp1_vdev_node *node;
	int ret = 0;
	char *vdev_name;
	printk("zjj.android10.kernel.camera || %s %s %d \n",
		__FUNCTION__, __FILE__, __LINE__); //zhoujinjian

	switch (stream->id) {
	case RKISP1_STREAM_SP:
		vdev_name = SP_VDEV_NAME;
		if (dev->isp_ver == ISP_V10_1)
			return 0;
		break;
	case RKISP1_STREAM_MP:
		vdev_name = MP_VDEV_NAME;
		break;
	case RKISP1_STREAM_RAW:
		vdev_name = RAW_VDEV_NAME;
		break;
	}
	strlcpy(vdev->name, vdev_name, sizeof(vdev->name));
	node = vdev_to_node(vdev);

	vdev->ioctl_ops = &rkisp1_v4l2_ioctl_ops;
	vdev->release = video_device_release_empty;
	vdev->fops = &rkisp1_fops;
	vdev->minor = -1;
	vdev->v4l2_dev = v4l2_dev;
	vdev->lock = &dev->apilock;
	//初始化device_caps = V4L2_CAP_VIDEO_CAPTURE_MPLANE | V4L2_CAP_STREAMING;
	//打开/dev/video0，可通过ioctl(fd, VIDIOC_QUERYCAP, &cap)查询
	vdev->device_caps = V4L2_CAP_VIDEO_CAPTURE_MPLANE |
				V4L2_CAP_STREAMING;
	video_set_drvdata(vdev, stream);
	vdev->vfl_dir = VFL_DIR_RX;
	node->pad.flags = MEDIA_PAD_FL_SINK;

	rkisp_init_vb2_queue(&node->buf_queue, stream,
			     V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE);
	vdev->queue = &node->buf_queue;
    //根据类型注册/dev/video*
rk3399_Android10:/ # grep '' /sys/class/video4linux/video*/name
/sys/class/video4linux/video0/name:rkisp1_mainpath
/sys/class/video4linux/video1/name:rkisp1_selfpath

	ret = video_register_device(vdev, VFL_TYPE_GRABBER, -1);

	ret = media_entity_pads_init(&vdev->entity, 1, &node->pad);

	return 0;
unreg:
	video_unregister_device(vdev);
	return ret;
}
```

#### 4.6、rkisp_init_vb2_queue

> 创建并初始化一个vb2_queue结构体 ，

``` c
F:\Khadas_Edge_Android_Q\kernel\drivers\media\platform\rockchip\isp1\common.h
/* One structure per video node */
struct rkisp1_vdev_node {
	struct vb2_queue buf_queue;
	struct video_device vdev;
	struct media_pad pad;
};
F:\Khadas_Edge_Android_Q\kernel\drivers\media\platform\rockchip\isp1\capture.c
static int rkisp_init_vb2_queue(struct vb2_queue *q,
				struct rkisp1_stream *stream,
				enum v4l2_buf_type buf_type)
{
	q->type = buf_type;  // 类型
	q->io_modes = VB2_MMAP | VB2_USERPTR | VB2_DMABUF; //该队列支持的模式
	q->drv_priv = stream;  // 自定义模式
	q->ops = &rkisp1_vb2_ops;
	q->mem_ops = &vb2_dma_contig_memops;
	q->buf_struct_size = sizeof(struct rkisp1_buffer);// 将vb2_buffer结构体封装到我们自己的buffer中，此为我们自己的buffer的size
	q->min_buffers_needed = CIF_ISP_REQ_BUFS_MIN;
	q->timestamp_flags = V4L2_BUF_FLAG_TIMESTAMP_MONOTONIC;
	q->lock = &stream->ispdev->apilock;
	q->dev = stream->ispdev->dev;
	printk("zjj.android10.kernel.camera || %s %s %d \n",
		__FUNCTION__, __FILE__, __LINE__); //zhoujinjian

	return vb2_queue_init(q);
}
其中rkisp1_vb2_ops 是队列的操作函数，以后的REQBUFS 等请求会调用到此中的函数，其结构如下
static struct vb2_ops rkisp1_vb2_ops = {
	.queue_setup = rkisp1_queue_setup,// 必须有，vb2_queue_init中会判断
	.buf_queue = rkisp1_buf_queue, // 必须有，vb2_queue_init中会判断
	.wait_prepare = vb2_ops_wait_prepare,
	.wait_finish = vb2_ops_wait_finish,
	.stop_streaming = rkisp1_stop_streaming,
	.start_streaming = rkisp1_start_streaming,
};

其中vb2_vmalloc_memops用于有关此队列中mem分配问题，其中的函数一般不需要我们自己写，使用默认
F:\Khadas_Edge_Android_Q\kernel\drivers\media\common\videobuf2\videobuf2-dma-contig.c
const struct vb2_mem_ops vb2_dma_contig_memops = {
	.alloc		= vb2_dc_alloc,
	.put		= vb2_dc_put,
	.get_dmabuf	= vb2_dc_get_dmabuf,
	.cookie		= vb2_dc_cookie,
	.vaddr		= vb2_dc_vaddr,
	.mmap		= vb2_dc_mmap,
	.get_userptr	= vb2_dc_get_userptr,
	.put_userptr	= vb2_dc_put_userptr,
	.prepare	= vb2_dc_prepare,
	.finish		= vb2_dc_finish,
	.map_dmabuf	= vb2_dc_map_dmabuf,
	.unmap_dmabuf	= vb2_dc_unmap_dmabuf,
	.attach_dmabuf	= vb2_dc_attach_dmabuf,
	.detach_dmabuf	= vb2_dc_detach_dmabuf,
	.num_users	= vb2_dc_num_users,
};
 
```

----

#### 4.7、判断驱动 probe 状态
RKISP1 如果 probe 成功，会有 video 及 media 设备存在于/dev/目录下。系统中可能存在多
个/dev/video 设备，通过/sys/ 可以查询到 RKISP1 注册的 video 节点

RKISP1 驱动会注册 4 个设备，分别为 selfpath，mainpath，statistics，input-params。
其中前两个是用于帧输出， 后两个是用于 3A 的参数设置与统计。通过查找 video4linux 下的 sys
节点，我们得到的RKISP1 信息如下表

``` c
rk3399_Android10:/ # grep '' /sys/class/video4linux/video*/name
/sys/class/video4linux/video0/name:rkisp1_mainpath
/sys/class/video4linux/video1/name:rkisp1_selfpath
/sys/class/video4linux/video2/name:rkisp1_dmapath
/sys/class/video4linux/video3/name:rkisp1-statistics
/sys/class/video4linux/video4/name:rkisp1-input-params
/sys/class/video4linux/video5/name:rkisp1_mainpath
/sys/class/video4linux/video6/name:rkisp1_selfpath
/sys/class/video4linux/video7/name:rkisp1_dmapath
/sys/class/video4linux/video8/name:rkisp1-statistics
/sys/class/video4linux/video9/name:rkisp1-input-params
```

![](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/Android.10.Camera.04/RK3399_dev_video.png)
Rkisp1 驱动会注册 4 个/dev/video 设备，编号连续。
- 如 RK3399 平台的两个 isp 同时启用，共会注册 8 个/dev/video 设备。前 4 个属于一个
isp，后 4 个属于另外一个 isp。软件上尚无法区分两个 isp

用户还可以通过 media-ctl，打印拓扑结构查看 pipeline 是否正常。请参考 media-ctl 及拓扑
结构。前面已经打印过Topology。

### （2）、mipi-dphy 驱动分析
#### 1、 mipi-dphy 的板级配置

``` c
F:\Khadas_Edge_Android_Q\kernel\arch\arm64\boot\dts\rockchip\rk3399.dtsi
		mipi_dphy_rx0: mipi-dphy-rx0 {
			compatible = "rockchip,rk3399-mipi-dphy";
			clocks = <&cru SCLK_MIPIDPHY_REF>,
				<&cru SCLK_DPHY_RX0_CFG>,
				<&cru PCLK_VIO_GRF>;
			clock-names = "dphy-ref", "dphy-cfg", "grf";
			power-domains = <&power RK3399_PD_VIO>;
			status = "disabled";
		};
F:\Khadas_Edge_Android_Q\kernel\arch\arm64\boot\dts\rockchip\rk3399-khadas-edge-android.dts
&mipi_dphy_rx0 {
	status = "okay";

	ports {
		#address-cells = <1>;
		#size-cells = <0>;

		port@0 {
			reg = <0>;
			#address-cells = <1>;
			#size-cells = <0>;

			mipi_in_ucam0: endpoint@1 {
				reg = <1>;
				remote-endpoint = <&ucam_out0>;
				data-lanes = <1 2>;
			};
		};

		port@1 {
			reg = <1>;
			#address-cells = <1>;
			#size-cells = <0>;

			dphy_rx0_out: endpoint@0 {
				reg = <0>;
				remote-endpoint = <&isp0_mipi_in>;
			};
		};
	};
};

&rkisp1_0 {
	status = "okay";

	port {
		#address-cells = <1>;
		#size-cells = <0>;

		isp0_mipi_in: endpoint@0 {
			reg = <0>;
			remote-endpoint = <&dphy_rx0_out>;
		};
	};
};

	imx214b: imx214b@10 {
		compatible = "sony,imx214";
		status = "okay";
		......
		lens-focus = <&vm149c>;
		port {
			ucam_out0: endpoint {
				remote-endpoint = <&mipi_in_ucam0>;
				//remote-endpoint = <&mipi_in_ucam1>;
				data-lanes = <1 2>;
			};
		};
	};	
```
#### 2、 mipi-dphy 驱动

``` cpp
F:\Khadas_Edge_Android_Q\kernel\drivers\phy\rockchip\phy-rockchip-mipi-rx.c

static const struct of_device_id rockchip_mipidphy_match_id[] = {
	......
	{
		.compatible = "rockchip,rk3399-mipi-dphy",
		.data = &rk3399_mipidphy_drv_data,
	},
	......
};

static const struct v4l2_subdev_pad_ops mipidphy_subdev_pad_ops = {
	.set_fmt = mipidphy_get_set_fmt,
	.get_fmt = mipidphy_get_set_fmt,
	.get_selection = mipidphy_get_selection,
};

static const struct v4l2_subdev_core_ops mipidphy_core_ops = {
	.s_power = mipidphy_s_power,
};

static const struct v4l2_subdev_video_ops mipidphy_video_ops = {
	.g_frame_interval = mipidphy_g_frame_interval,
	.g_mbus_config = mipidphy_g_mbus_config,
	.s_stream = mipidphy_s_stream,
};

static const struct v4l2_subdev_ops mipidphy_subdev_ops = {
	.core = &mipidphy_core_ops,
	.video = &mipidphy_video_ops,
	.pad = &mipidphy_subdev_pad_ops,
};

static int rockchip_mipidphy_probe(struct platform_device *pdev)
{
	struct device *dev = &pdev->dev;
	struct v4l2_subdev *sd;
	struct mipidphy_priv *priv;
	struct regmap *grf;
	struct resource *res;
	const struct of_device_id *of_id;
	const struct dphy_drv_data *drv_data;
	int i, ret;
	printk("zjj.android10.kernel.camera || %s %s %d \n",
		__FUNCTION__, __FILE__, __LINE__); //zhoujinjian

	priv = devm_kzalloc(dev, sizeof(*priv), GFP_KERNEL);
	of_id = of_match_device(rockchip_mipidphy_match_id, dev);
	grf = syscon_node_to_regmap(dev->parent->of_node);
	priv->regmap_grf = grf;

	priv->phy_index = of_alias_get_id(dev->of_node, "dphy");
	if (priv->phy_index < 0)
		priv->phy_index = 0;

	drv_data = of_id->data;
	for (i = 0; i < drv_data->num_clks; i++) {
		priv->clks[i] = devm_clk_get(dev, drv_data->clks[i]);
	}

	priv->grf_regs = drv_data->grf_regs;
	priv->txrx_regs = drv_data->txrx_regs;
	priv->csiphy_regs = drv_data->csiphy_regs;
	priv->drv_data = drv_data;
	if (drv_data->ctl_type == MIPI_DPHY_CTL_CSI_HOST) {
		......
	} else {
	//设置stream_on函数
		priv->stream_on = mipidphy_txrx_stream_on;
		priv->txrx_base_addr = NULL;
		res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
		priv->txrx_base_addr = devm_ioremap_resource(dev, res);
		if (IS_ERR(priv->txrx_base_addr))
			priv->stream_on = mipidphy_rx_stream_on;
		priv->stream_off = NULL;
	}

	sd = &priv->sd;
	mutex_init(&priv->mutex);
	//设置v4l2_subdev_ops 
	v4l2_subdev_init(sd, &mipidphy_subdev_ops);
	sd->flags |= V4L2_SUBDEV_FL_HAS_DEVNODE;
	snprintf(sd->name, sizeof(sd->name), "rockchip-mipi-dphy-rx");
	sd->dev = dev;

	platform_set_drvdata(pdev, &sd->entity);
    //media_entity_pads_init
    //v4l2_async_register_subdev
	ret = rockchip_mipidphy_media_init(priv);
	pm_runtime_enable(&pdev->dev);
	drv_data->individual_init(priv);
	return 0;
}
```

### （3）、VCM驱动

#### 1、VCM 设备注册(DTS)

``` c
F:\Khadas_Edge_Android_Q\kernel\arch\arm64\boot\dts\rockchip\rk3399-khadas-edge-android.dts
&i2c6 {
	status = "okay";

	vm149c: vm149c@0c {// vcm 驱动设置，支持 AF 时需要有这个设置
		compatible = "silicon touch,vm149c";
		status = "okay";
		reg = <0x0c>;
		rockchip,camera-module-index = <0>;// 模组编号
		rockchip,camera-module-facing = "back";// 模组朝向，有"back"和"front"
	};
	
	imx214b: imx214b@10 {
		compatible = "sony,imx214";
		status = "okay";
		reg = <0x10>;
		clocks = <&cru SCLK_CIF_OUT>;
		clock-names = "xvclk";
		/* avdd-supply = <>; */
		/* dvdd-supply = <>; */
		/* dovdd-supply = <>; */
		/* reset-gpios = <>; */
		pwdn-gpios = <&gpio2 5 GPIO_ACTIVE_HIGH>;
		reset-gpios = <&gpio2 12 GPIO_ACTIVE_HIGH>;
		pinctrl-names = "rockchip,camera_default";
		pinctrl-0 = <&cif_clkout>;
		rockchip,camera-module-index = <0>;
		rockchip,camera-module-facing = "back";
		rockchip,camera-module-name = "YG9626-S600Y7-C";
		rockchip,camera-module-lens-name = "LG-50013A7";
		lens-focus = <&vm149c>;// vcm 驱动设置，支持 AF 时需要有这个设置
		port {
			ucam_out0: endpoint {
				remote-endpoint = <&mipi_in_ucam0>;
				//remote-endpoint = <&mipi_in_ucam1>;
				data-lanes = <1 2>;
			};
		};
	};	
```
#### 2、VCM 驱动说明

``` c
F:\Khadas_Edge_Android_Q\kernel\drivers\media\i2c\vm149c.c
//vm149c_get_ctrl 和 vm149c_set_ctrl 对下面的 control 进行了支持
//V4L2_CID_FOCUS_ABSOLUTE
static const struct v4l2_ctrl_ops vm149c_vcm_ctrl_ops = {
	.g_volatile_ctrl = vm149c_get_ctrl,
	.s_ctrl = vm149c_set_ctrl,
};
//目前使用了如下的私有 ioctl 实现马达移动时间信息的查询。
//RK_VIDIOC_VCM_TIMEINFO
static const struct v4l2_subdev_core_ops vm149c_core_ops = {
	.ioctl = vm149c_ioctl,
#ifdef CONFIG_COMPAT
	.compat_ioctl32 = vm149c_compat_ioctl32
#endif
};

static const struct v4l2_subdev_ops vm149c_ops = {
	.core = &vm149c_core_ops,
};


static int vm149c_probe(struct i2c_client *client,
			const struct i2c_device_id *id)
{
	struct device_node *np = of_node_get(client->dev.of_node);
	struct vm149c_device *vm149c_dev;
	int ret;
	int current_distance;
	unsigned int start_current;
	unsigned int rated_current;
	unsigned int step_mode;
	struct v4l2_subdev *sd;
	char facing[2];

	dev_info(&client->dev, "probing...\n");
	dev_info(&client->dev, "driver version: %02x.%02x.%02x",
		DRIVER_VERSION >> 16,
		(DRIVER_VERSION & 0xff00) >> 8,
		DRIVER_VERSION & 0x00ff);
	......
//v4l2_i2c_subdev_init，RK VCM 驱动要求 subdev 拥有自己的设
//备节点供用户态 camera_engine 访问，通过该设备节点实现调焦控制；
	
	v4l2_i2c_subdev_init(&vm149c_dev->sd, client, &vm149c_ops);
	vm149c_dev->sd.flags |= V4L2_SUBDEV_FL_HAS_DEVNODE;
	vm149c_dev->sd.internal_ops = &vm149c_int_ops;

	ret = vm149c_init_controls(vm149c_dev);
	ret = media_entity_pads_init(&vm149c_dev->sd.entity, 0, NULL);

	sd = &vm149c_dev->sd;
	sd->entity.function = MEDIA_ENT_F_LENS;
	......
	snprintf(sd->name, sizeof(sd->name), "m%02d_%s_%s %s",
		 vm149c_dev->module_index, facing,
		 VM149C_NAME, dev_name(sd->dev));
	ret = v4l2_async_register_subdev(sd);
	......
	dev_info(&client->dev, "probing successful\n");

	return 0;

}
static int vm149c_init_controls(struct vm149c_device *dev_vcm)
{
	struct v4l2_ctrl_handler *hdl = &dev_vcm->ctrls_vcm;
	const struct v4l2_ctrl_ops *ops = &vm149c_vcm_ctrl_ops;

	v4l2_ctrl_handler_init(hdl, 1);

	v4l2_ctrl_new_std(hdl, ops, V4L2_CID_FOCUS_ABSOLUTE,
			  0, VCMDRV_MAX_LOG, 1, VCMDRV_MAX_LOG);

	if (hdl->error)
		dev_err(dev_vcm->sd.dev, "%s fail error: 0x%x\n",
			__func__, hdl->error);
	dev_vcm->sd.ctrl_handler = hdl;
	return hdl->error;
}

static int rockchip_mipidphy_media_init(struct mipidphy_priv *priv)
{
	int ret;
	printk("zjj.android10.kernel.camera || %s %s %d \n",
		__FUNCTION__, __FILE__, __LINE__); //zhoujinjian

	priv->pads[MIPI_DPHY_RX_PAD_SOURCE].flags =
		MEDIA_PAD_FL_SOURCE | MEDIA_PAD_FL_MUST_CONNECT;
	priv->pads[MIPI_DPHY_RX_PAD_SINK].flags =
		MEDIA_PAD_FL_SINK | MEDIA_PAD_FL_MUST_CONNECT;
	priv->sd.entity.function = MEDIA_ENT_F_VID_IF_BRIDGE;
	ret = media_entity_pads_init(&priv->sd.entity,
				MIPI_DPHY_RX_PADS_NUM, priv->pads);
	......
	ret = v4l2_async_notifier_parse_fwnode_endpoints_by_port(
		priv->dev, &priv->notifier,
		sizeof(struct sensor_async_subdev), 0,
		rockchip_mipidphy_fwnode_parse);
	......
	priv->sd.subdev_notifier = &priv->notifier;
	priv->notifier.ops = &rockchip_mipidphy_async_ops;
	ret = v4l2_async_subdev_notifier_register(&priv->sd, &priv->notifier);
	......

	return v4l2_async_register_subdev(&priv->sd);
}

```

### （4）、imx214驱动分析
Camera Sensor 采用 I2C 与主控进行交互，目前 sensor driver 按照 I2C 设备驱动方式实现，sensor driver 同时采用 v4l2 subdev 的方式实现与 host driver 之间的交互。

Sensor 驱动位于 drivers/media/i2c 目录下，注意到本章节所描述的是具有 mediacontroller 属性的 sensor 驱动， 故 drivers/media/i2c/soc_camera 目录下的驱动并不适用。在 Media Controller 结构下， Sensor 一般作为 sub-device 并通过 pad 与 cif、isp 或者mipi phy 链接在一起。本章主要介绍 Sensor 驱动的代码[1]， dts 配置，及如何验证 sensor 驱动的正确性
``` c
F:\Khadas_Edge_Android_Q\kernel\arch\arm64\boot\dts\rockchip\rk3399-khadas-edge-android.dts
	imx214b: imx214b@10 {
		compatible = "sony,imx214";
		status = "okay";
		reg = <0x10>;
		clocks = <&cru SCLK_CIF_OUT>;
		clock-names = "xvclk";
		/* avdd-supply = <>; */
		/* dvdd-supply = <>; */
		/* dovdd-supply = <>; */
		/* reset-gpios = <>; */
		pwdn-gpios = <&gpio2 5 GPIO_ACTIVE_HIGH>;
		reset-gpios = <&gpio2 12 GPIO_ACTIVE_HIGH>;
		pinctrl-names = "rockchip,camera_default";
		pinctrl-0 = <&cif_clkout>;
		rockchip,camera-module-index = <0>;
		rockchip,camera-module-facing = "back";
		rockchip,camera-module-name = "YG9626-S600Y7-C";
		rockchip,camera-module-lens-name = "LG-50013A7";
		lens-focus = <&vm149c>;
		port {
			ucam_out0: endpoint {
				remote-endpoint = <&mipi_in_ucam0>;
				//remote-endpoint = <&mipi_in_ucam1>;
				data-lanes = <1 2>;
			};
		};
	};	

```
Pinctrl，声明了必要的 pinctrl， 该例子中包括了 reset pin 初始化和 clk iomux
- Clock，指定名称为 xvclk（驱动会讯取名为 xvclk 的 clock），即 24M 时钟
- Vdd supply
- Port 子节点，定义了一个 endpoint，声明需要与 mipi_in_wcam 建立连接。同样地 mipi
dphy 会引用 ucam_out0
- Data-lanes 指定了 imx214 使用两个 lane


本章将 Sensor 驱动的开发移植概括为 5 个部分，
- 按照 datasheet 编写上电时序， 主要包括 vdd，reset，powerdown，clk 等。
- 配置 sensor 的寄存器以输出所需的分辨率、格式。
- 编写 struct v4l2_subdev_ops 所需要的回调函数，一般包括 set_fmt，get_fmt，imx214_s_stream
- 增加 v4l2 controller 用来设置如 fps，exposure，gain，test pattern
- 编写.probe()函数，并添加 media control 及 v4l2 sub device 初始化代码作为良好的习惯，完成驱动编码后，也需要增加相应的 Documentation，可以参考Documentation/devicetree/bindings/media/i2c/。这样板级 dts 可以根据该文档快速配置。在板级 dts 中，引用 Sensor 驱动，一般需要
- 配置正确的 clk，io mux
- 根据原理图设置上电时序所需要的 regulator 及 gpio
- 增加 port 子节点，与 cif 或者 isp 建立连接

本章以im214为例，简单分析 Sensor 驱动。
#### 1、上电时序
不同 Sensor 对上电时序要求不同，例如 OV Camera。可能很大部分的 OV Camera 对时序要求不严格，只要 mclk，vdd，reset 或 powerdown 状态是对的，就能正确进行 I2C 通讯并输出图片。但还是有小部分 Sensor 对上电要求非常严格。在 Sensor 厂家提供的 DataSheet 中，一般会有上电时序图，只需要按顺序配置即可。

```
F:\Khadas_Edge_Android_Q\kernel\drivers\media\i2c\imx214.c
static int __imx214_power_on(struct imx214 *imx214)
{
	int ret;
	u32 delay_us;
	struct device *dev = &imx214->client->dev;

	dev_info(&imx214->client->dev, "%s(%d) enter!\n", __func__, __LINE__);
	printk("zjj.android10.kernel.camera || %s %s %d \n",
		__FUNCTION__, __FILE__, __LINE__); //zhoujinjian
......
	ret = clk_set_rate(imx214->xvclk, IMX214_XVCLK_FREQ);
......
	ret = clk_prepare_enable(imx214->xvclk);
......
	ret = regulator_bulk_enable(IMX214_NUM_SUPPLIES, imx214->supplies);
......
	if (!IS_ERR(imx214->reset_gpio))
		gpiod_set_value_cansleep(imx214->reset_gpio, 0);

	usleep_range(500, 1000);
	if (!IS_ERR(imx214->pwdn_gpio))
		gpiod_set_value_cansleep(imx214->pwdn_gpio, 1);
	usleep_range(500, 1000);
......
	delay_us = imx214_cal_delay(8192);
	usleep_range(delay_us, delay_us * 2);

	return 0;
}
```
imx214 的上电时序简要说明如下，
- 首先提供 xvclk(即 mclk)
- 紧接着 reset pin 使能
- 各路的 vdd 上电。这里使用了 regulator_bulk，因为 vdd, vodd, avdd 三者无严格顺序。如果 vdd 之间有严格的要求，需要分开处理，可参考 OV2685 驱动代码
- Vdd 上电后， 取消 Sensor Reset，powerdown 状态。Reset, powerdown 可能只需要一个，Sensor 封装不同， 可能有差异
- 最后按时序要求，需要 8192 个 clk cycle 之后，上电才算完成。
注意，虽然不按 datasheet 要求上电许多 Sensor 也能正常工作，但按原厂建议的时序操作，无疑是最可靠的。同样，datasheet 中还会有下电时序(Power Down Sequence)，也同样按要求即可。

在.probe()阶段会去尝试读取 chip id，如 imx214的imx214_check_sensor_id()，够正确读取到 chip id，一般就认为上电时序正确，Sensor 能够正常进行 i2c 通信。

#### 2、Sensor 初始化寄存器列表
在imx214 中，各定义了struct imx214_mode ，

``` c
F:\Khadas_Edge_Android_Q\kernel\drivers\media\i2c\imx214.c
static const struct imx214_mode supported_modes[] = {
	{
		.width = 4208,
		.height = 3120,
		.max_fps = {
			.numerator = 10000,
			.denominator = 150000,
		},
		.exp_def = 0x0c70,
		.hts_def = 0x1390,
		.vts_def = 0x0c7a,
		.reg_list = imx214_4208x3120_regs,
	},
	{
		.width = 2104,
		.height = 1560,
		.max_fps = {
			.numerator = 10000,
			.denominator = 300000,
		},
		.exp_def = 0x0630,
		.hts_def = 0x1390,
		.vts_def = 0x0640,
		.reg_list = imx214_2104x1560_regs,
	},
};
```

用来表示 sensor 不同的初始化 mode。Mode 可以包括如分辨率，mbus code，寄存器初始化列表等。寄存器初始化列表，请按厂家提供的直接填入即可。需要注意的是， 列表最后用了 REG_NULL表示结束

#### 3、 V4l2_subdev_ops 回调函数

V4l2_subdev_ops 回调函数是 Sensor 驱动中逻辑控制的核心。回调函数包括非常多的功能，具体可以查看 kernel 代码 include/media/v4l2-subdev.h。建议 Sensor 驱动至少包括如下回调函数。

``` c
F:\Khadas_Edge_Android_Q\kernel\drivers\media\i2c\imx214.c
static const struct dev_pm_ops imx214_pm_ops = {
	SET_RUNTIME_PM_OPS(imx214_runtime_suspend,
		imx214_runtime_resume, NULL)
};

#ifdef CONFIG_VIDEO_V4L2_SUBDEV_API
static const struct v4l2_subdev_internal_ops imx214_internal_ops = {
	.open = imx214_open,
};
#endif

static const struct v4l2_subdev_core_ops imx214_core_ops = {
	.s_power = imx214_s_power,
	.ioctl = imx214_ioctl,
#ifdef CONFIG_COMPAT
	.compat_ioctl32 = imx214_compat_ioctl32,
#endif
};

static const struct v4l2_subdev_video_ops imx214_video_ops = {
	.s_stream = imx214_s_stream,
	.g_frame_interval = imx214_g_frame_interval,
};

static const struct v4l2_subdev_pad_ops imx214_pad_ops = {
	.enum_mbus_code = imx214_enum_mbus_code,
	.enum_frame_size = imx214_enum_frame_sizes,
	.enum_frame_interval = imx214_enum_frame_interval,
	.get_fmt = imx214_get_fmt,
	.set_fmt = imx214_set_fmt,
};

static const struct v4l2_subdev_ops imx214_subdev_ops = {
	.core	= &imx214_core_ops,
	.video	= &imx214_video_ops,
	.pad	= &imx214_pad_ops,
};

```

- .open，这样上层才可以打开/dev/v4l-subdev?节点。在上层需要单独对 sensor 设置 v4l control 时，.open()是必须实现的
- .s_stream，即 set stream，包括 stream on 和 stream off，一般在这里配置寄存器，使其输出图像
- .enum_mbus_code，枚举驱动支持的 mbus_code
- .enum_frame_size，枚举驱动支持的分辨率
- .get_fmt，返回当前 Sensor 选中的 format/size。如果.get_fmt 缺失，media-ctl 工具无法查看 sensor entity 当前配置的 fmt
- .set_fmt，设置 Sensor 的 format/size以上回调中， 
- .s_stream stream_on 会比较复杂些。在 imx214驱动代码中，它包括pm_runtime 使能（即唤醒并上电），配置 control 信息（v4l2 control 可能会在 sensor 下电时配置）即 v4l2_ctrl_handler_setup()，并最终写入寄存器 stream on。

#### 4、 V4l2 controller
对于需要配置 fps，exposure, gain, blanking 的场景，v4l2 controller 部分是必要的。

``` c
F:\Khadas_Edge_Android_Q\kernel\drivers\media\i2c\imx214.c
static const struct v4l2_ctrl_ops imx214_ctrl_ops = {
	.s_ctrl = imx214_set_ctrl,
};

static int imx214_initialize_controls(struct imx214 *imx214)
{
	const struct imx214_mode *mode;
	struct v4l2_ctrl_handler *handler;
	s64 exposure_max, vblank_def;
	u32 h_blank;
	int ret;
	printk("zjj.android10.kernel.camera || %s %s %d \n",
		__FUNCTION__, __FILE__, __LINE__); //zhoujinjian

	handler = &imx214->ctrl_handler;
	mode = imx214->cur_mode;
	ret = v4l2_ctrl_handler_init(handler, 8);
	......
	handler->lock = &imx214->mutex;

	imx214->link_freq = v4l2_ctrl_new_int_menu(handler, NULL,
		V4L2_CID_LINK_FREQ, 1, 0,
		link_freq_menu_items);

	imx214->pixel_rate_ctrl = v4l2_ctrl_new_std(handler, NULL,
		V4L2_CID_PIXEL_RATE, 0, IMX214_PIXEL_RATE_FULL_SIZE,
		1, IMX214_PIXEL_RATE_FULL_SIZE);

	h_blank = mode->hts_def - mode->width;
	imx214->hblank = v4l2_ctrl_new_std(handler, NULL,
		V4L2_CID_HBLANK, h_blank, h_blank, 1, h_blank);
	if (imx214->hblank)
		imx214->hblank->flags |= V4L2_CTRL_FLAG_READ_ONLY;

	vblank_def = mode->vts_def - mode->height;
	imx214->vblank = v4l2_ctrl_new_std(handler, &imx214_ctrl_ops,
		V4L2_CID_VBLANK, vblank_def,
		IMX214_VTS_MAX - mode->height,
		1, vblank_def);

	exposure_max = mode->vts_def - 4;
	imx214->exposure = v4l2_ctrl_new_std(handler, &imx214_ctrl_ops,
		V4L2_CID_EXPOSURE, IMX214_EXPOSURE_MIN,
		exposure_max, IMX214_EXPOSURE_STEP,
		mode->exp_def);

	imx214->anal_gain = v4l2_ctrl_new_std(handler, &imx214_ctrl_ops,
		V4L2_CID_ANALOGUE_GAIN, IMX214_GAIN_MIN,
		IMX214_GAIN_MAX, IMX214_GAIN_STEP,
		IMX214_GAIN_DEFAULT);

	imx214->test_pattern = v4l2_ctrl_new_std_menu_items(handler,
		&imx214_ctrl_ops, V4L2_CID_TEST_PATTERN,
		ARRAY_SIZE(imx214_test_pattern_menu) - 1,
		0, 0, imx214_test_pattern_menu);

	if (handler->error) {
		ret = handler->error;
		dev_err(&imx214->client->dev,
			"Failed to init controls(%d)\n", ret);
		goto err_free_handler;
	}

	imx214->subdev.ctrl_handler = handler;

	return 0;

```

- imx214_initialize_controls()，用来声明支持哪些 control，并设置最大最小值等信息
- Struct v4l2_ctrl_ops，包含了 imx214_set_ctrl()回调函数，用以响应上层的设置。

#### 5、Probe 函数及注册 media entity, v4l2 subdev
Probe 函数中， 首先对 dts 进行解析，获取 regulator, gpio, clk 等信息用以对 sensor 上下电。其次注册 media entity, v4l2 subdev，及 v4l2 controller 信息。注意到 v4l2 subdev 的注册是异步。如下几个关键的函数调用。
- v4l2_i2c_subdev_init(), 注册为一个 v4l2 subdev，参数中提供回调函数
- imx214_initialize_controls()，初始化 v4l2 controls
- media_entity_pads_init()，注册成为一个 media entity，imx214 仅有一个输出，即 SourcePad
- v4l2_async_register_subdev()，声明 sensor 需要异步注册。因为 RKISP1 及 CIF 都采用异步注册 


随 Linux SDK 发布的，还有一个 rkisp-demo， 位于 F:\Khadas_Edge_Android_Q\hardware\rockchip\camera_engine_rkisp\tests\rkisp_demo\rkisp_demo.cpp。该 demo 极简单
地使用了 v4l2 接口配置设备。请直接参考源码即可。

## （三）、Camera Buffer Flow
当ISP1准备好数据后，就会发生中断调用mi_frame_end 。调用vb2_buffer_done，应用app就可以dqbuf获取camera数据了，
并且update_mi更新下一个buffer 的地址给memory interface，如此反复循环。


``` c
[2021/6/17 11:13:50] [  127.289285] Hardware name: Khadas,edge (DT)
[2021/6/17 11:13:50] [  127.289335] Call trace:
[2021/6/17 11:13:50] [  127.289373]  dump_backtrace+0x0/0x178
[2021/6/17 11:13:50] [  127.289415]  show_stack+0x14/0x20
[2021/6/17 11:13:50] [  127.289463]  dump_stack+0x94/0xb4
[2021/6/17 11:13:50] [  127.289508]  vb2_buffer_done+0x25c/0x2c0
[2021/6/17 11:13:50] [  127.289560]  mi_frame_end+0x118/0x248
[2021/6/17 11:13:50] [  127.289618]  rkisp1_mi_isr+0xe4/0x118
[2021/6/17 11:13:50] [  127.289664]  rkisp1_irq_handler+0xa4/0xd8
[2021/6/17 11:13:50] [  127.289713]  __handle_irq_event_percpu+0x6c/0x2a0
[2021/6/17 11:13:50] [  127.289766]  handle_irq_event_percpu+0x34/0x88
[2021/6/17 11:13:50] [  127.289811]  handle_irq_event+0x48/0x78
[2021/6/17 11:13:50] [  127.289862]  handle_fasteoi_irq+0xac/0x188
[2021/6/17 11:13:50] [  127.289916]  generic_handle_irq+0x24/0x38
[2021/6/17 11:13:50] [  127.289957]  __handle_domain_irq+0x5c/0xb8
[2021/6/17 11:13:50] [  127.289978]  gic_handle_irq+0xc4/0x178
[2021/6/17 11:13:50] [  127.290011]  el1_irq+0xe8/0x198
[2021/6/17 11:13:50] [  127.290064]  __do_softirq+0x9c/0x340
[2021/6/17 11:13:50] [  127.290111]  irq_exit+0xcc/0xd8
[2021/6/17 11:13:50] [  127.290165]  __handle_domain_irq+0x60/0xb8
[2021/6/17 11:13:50] [  127.290204]  gic_handle_irq+0xc4/0x178
[2021/6/17 11:13:50] [  127.290236]  el1_irq+0xe8/0x198
[2021/6/17 11:13:50] [  127.290273]  cpuidle_enter_state+0xb4/0x3c8
[2021/6/17 11:13:50] [  127.290308]  cpuidle_enter+0x18/0x20
[2021/6/17 11:13:50] [  127.290343]  call_cpuidle+0x18/0x30
[2021/6/17 11:13:50] [  127.290366]  do_idle+0x214/0x278
[2021/6/17 11:13:50] [  127.290401]  cpu_startup_entry+0x20/0x28
[2021/6/17 11:13:50] [  127.290436]  rest_init+0xcc/0xd8
[2021/6/17 11:13:50] [  127.290474]  start_kernel+0x4c0/0x4ec
```

``` c
F:\Khadas_Edge_Android_Q\kernel\drivers\media\platform\rockchip\isp1\capture.c

static int mi_frame_end(struct rkisp1_stream *stream)
{
	struct rkisp1_device *isp_dev = stream->ispdev;
	struct rkisp1_isp_subdev *isp_sd = &isp_dev->isp_sdev;
	struct capture_fmt *isp_fmt = &stream->out_isp_fmt;
	bool interlaced = stream->interlaced;
	unsigned long lock_flags = 0;
	int i = 0;
	printk("zjj.android10.kernel.camera || %s %s %d \n",
		__FUNCTION__, __FILE__, __LINE__); //zhoujinjian

	if (stream->curr_buf &&
		(!interlaced ||
		(stream->u.sp.field_rec == RKISP_FIELD_ODD &&
		stream->u.sp.field == RKISP_FIELD_EVEN))) {
		u64 ns = ktime_get_ns();

		/* Dequeue a filled buffer */
		for (i = 0; i < isp_fmt->mplanes; i++) {
			u32 payload_size =
				stream->out_fmt.plane_fmt[i].sizeimage;
			vb2_set_plane_payload(
				&stream->curr_buf->vb.vb2_buf, i,
				payload_size);
		}
		stream->curr_buf->vb.sequence =
				atomic_read(&isp_sd->frm_sync_seq) - 1;
		stream->curr_buf->vb.vb2_buf.timestamp = ns;
		vb2_buffer_done(&stream->curr_buf->vb.vb2_buf,
				VB2_BUF_STATE_DONE);
		stream->curr_buf = NULL;
	}

	if (!interlaced ||
		(stream->curr_buf == stream->next_buf &&
		stream->u.sp.field == RKISP_FIELD_ODD)) {
		/* Next frame is writing to it
		 * Interlaced: odd field next buffer address
		 */
		stream->curr_buf = stream->next_buf;
		stream->next_buf = NULL;

		/* Set up an empty buffer for the next-next frame */
		spin_lock_irqsave(&stream->vbq_lock, lock_flags);
		if (!list_empty(&stream->buf_queue)) {
			stream->next_buf =
				list_first_entry(&stream->buf_queue,
						 struct rkisp1_buffer,
						 queue);
			list_del(&stream->next_buf->queue);
		}
		spin_unlock_irqrestore(&stream->vbq_lock, lock_flags);
	} else if (stream->u.sp.field_rec == RKISP_FIELD_ODD &&
		stream->u.sp.field == RKISP_FIELD_EVEN) {
		/* Interlaced: event field next buffer address */
		if (stream->next_buf) {
			stream->next_buf->buff_addr[RKISP1_PLANE_Y] +=
				stream->u.sp.vir_offs;
			stream->next_buf->buff_addr[RKISP1_PLANE_CB] +=
				stream->u.sp.vir_offs;
			stream->next_buf->buff_addr[RKISP1_PLANE_CR] +=
				stream->u.sp.vir_offs;
		}
		stream->curr_buf = stream->next_buf;
	}

	stream->ops->update_mi(stream);

	if (interlaced)
		stream->u.sp.field_rec = stream->u.sp.field;

	return 0;
}


/* Update buffer info to memory interface, it's called in interrupt */
static void update_mi(struct rkisp1_stream *stream)
{
	struct rkisp1_dummy_buffer *dummy_buf = &stream->dummy_buf;
	printk("zjj.android10.kernel.camera || %s %s %d \n",
		__FUNCTION__, __FILE__, __LINE__); //zhoujinjian

	/* The dummy space allocated by dma_alloc_coherent is used, we can
	 * throw data to it if there is no available buffer.
	 */
	if (stream->next_buf) {
		//NV12中,U和V就交错排布的。看到内存中的排布很清楚，先开始都是Y，之后的都是U1V1U2V2的交错式排布。
		//https://blog.csdn.net/byhook/article/details/84037338
		//以 4 X 4 图片为例子，占用内存为 4 X 4 X 3 / 2 = 24 个字节,Y存储, mWidth:1280, mHeight:960
		printk("zjj.android10.kernel.camera RKISP1_PLANE_Y : 0x%08lx \n", (unsigned long)stream->next_buf->buff_addr[RKISP1_PLANE_Y]);	
		printk("zjj.android10.kernel.camera RKISP1_PLANE_CB: 0x%08lx \n", (unsigned long)stream->next_buf->buff_addr[RKISP1_PLANE_CB]);	
		printk("zjj.android10.kernel.camera RKISP1_PLANE_CR: 0x%08lx \n", (unsigned long)stream->next_buf->buff_addr[RKISP1_PLANE_CR]);	

		mi_set_y_addr(stream,
			stream->next_buf->buff_addr[RKISP1_PLANE_Y]);
		mi_set_cb_addr(stream,
			stream->next_buf->buff_addr[RKISP1_PLANE_CB]);
		mi_set_cr_addr(stream,
			stream->next_buf->buff_addr[RKISP1_PLANE_CR]);
	} else {
		v4l2_dbg(1, rkisp1_debug, &stream->ispdev->v4l2_dev,
			 "stream %d: to dummy buf\n", stream->id);	
		mi_set_y_addr(stream, dummy_buf->dma_addr);
		mi_set_cb_addr(stream, dummy_buf->dma_addr);
		mi_set_cr_addr(stream, dummy_buf->dma_addr);
	}

	mi_set_y_offset(stream, 0);
	mi_set_cb_offset(stream, 0);
	mi_set_cr_offset(stream, 0);
}

```

