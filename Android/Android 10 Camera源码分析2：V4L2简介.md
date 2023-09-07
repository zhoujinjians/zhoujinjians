---
title:  Android 10 Camera源码分析2：V4L2简介
cover: https://raw.githubusercontent.com/zhoujinjianOS/PicGo/master/post.cover.pictures/bing-wallpaper-2018.04.32.jpg
categories: 
  - Camera
tags:
  - Camera
toc: true
abbrlink: 20220115
date: 2022-01-15 01:15:00
---

注：文章都是通过阅读各位前辈总结的资料 Android 10.0 && Linux（Kernel 4.19）Rockchip平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除（◕‿◕），转载请注明出处（©Rockchip ©Android @Linux 版权所有），谢谢。
（==**文章基于 Kernel-4.19**==）&&（==**文章基于 Android 10.0**==）
 [【blog.zhoujinjian.cn博客原图链接】](https://github.com/zhoujinjianOS) 
 [【开发板 Khadas Edge V】](https://www.khadas.cn/edge-v)
[【开发板 Khadas Edge V Android 10.0 && Linux（Kernel 4.19）源码链接】](https://github.com/khadas/linux/tree/khadas-edge-Qt)

正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢！！！

 [第一章 V4L2简介](https://work-blog.readthedocs.io/en/latest/v4l2%20intro.html) 
 [linux v4l2学习之-消息机制](https://blog.csdn.net/armwind/article/details/88765108) 
 [linux v4l2学习之-v4l2设备注册过程及各个设备之间的联系](https://blog.csdn.net/armwind/article/details/88781335) 
 [V4L2框架解析](https://deepinout.com/v4l2-tutorials/linux-v4l2-architecture.html) 

--------------------------------------------------------------------------------
==源码（部分）==：
**F:\Khadas_Edge_Android_Q\kernel\include\media\v4l2-dev.h**
**F:\Khadas_Edge_Android_Q\kernel\include\media\v4l2-device.h**
**F:\Khadas_Edge_Android_Q\kernel\include\media\v4l2-subdev.h**
**F:\Khadas_Edge_Android_Q\kernel\drivers\media\platform\rockchip\isp1\capture.c**
**F:\Khadas_Edge_Android_Q\kernel\drivers\phy\rockchip\phy-rockchip-mipi-rx.c**
**F:\Khadas_Edge_Android_Q\kernel\drivers\media\platform\rockchip\isp1\dev.c**
**F:\Khadas_Edge_Android_Q\kernel\drivers\media\v4l2-core\v4l2-event.c**
**F:\Khadas_Edge_Android_Q\kernel\drivers\media\v4l2-core\v4l2-device.c**
**F:\Khadas_Edge_Android_Q\kernel\include\uapi\linux\videodev2.h**
***F:\Khadas_Edge_Android_Q\kernel\include\media\v4l2-event.h**

-------------------------------------------------------------------------------


### （一）、什么是v4l2


V4L2（Video4Linux的缩写）是Linux下关于视频采集相关设备的驱动框架，为驱动和应用程序提供了一套统一的接口规范。

V4L2支持的设备十分广泛，但是其中只有很少一部分在本质上是真正的视频设备：

*   **Video capture device** ： 从摄像头等设备上获取视频数据。对很多人来讲，video capture是V4L2的基本应用。设备名称为/dev/video,主设备号81，子设备号0~63
*   **Video output device** ： 将视频数据编码为模拟信号输出。与video capture设备名相同。
*   **Video overlay device** ： 将同步锁相视频数据（如TV）转换为VGA信号，或者将抓取的视频数据直接存放到视频卡的显存中。
*   **Video output overlay device** ：也被称为OSD(On-Screen Display)
*   **VBI device** ： 提供对VBI（Vertical Blanking Interval）数据的控制，发送VBI数据或抓取VBI数据。设备名/dev/vbi0~vbi31,主设备号81,子设备号224~255
*   **Radio device** ： FM/AM发送和接收设备。设备名/dev/radio0~radio63,主设备号81，子设备号64~127

V4L2在Linux系统中的结构图如下：
![](https://raw.githubusercontent.com/zhoujinjianOS/PicGo/master/Android.10.Camera.02/CAMERAOverview.png)


### （二）、从应用层看V4L2

从**==V4L2简单框图==**以看出,V4L2是一个字符设备，而V4L2的大部分功能都是通过设备文件的ioctl导出的。

**可以将这些ioctl分类如下**：

##### 1、 Query Capability:查询设备支持的功能，只有VIDIOC_QUERY_CAP一个。
##### 2、优先级相关：包括VIDIOC_G_PRIORITY,VIDIOC_S_PRIORITY,设置优先级。
##### 3、capture相关：视频捕获相关Ioctl。
> **capture ioctl list**
> | ID | 描述 |
> | --- | --- |
> | VIDIOC_ENUM_FMT | 枚举设备所支持的所有数据格式 |
> | VIDIOC_S_FMT | 设置数据格式 |
> | VIDIOC_G_FMT | 获取数据格式 |
> | VIDIOC_TRY_FMT | 与VIDIOC_S_FMT一样，但不会改变设备的状态 |
> | VIDIOC_REQBUFS | 向设备请求视频缓冲区，即初始化视频缓冲区 |
> | VIDIOC_QUERYBUF | 查询缓冲区的状态 |
> | VIDIOC_QBUF | 从设备获取一帧视频数据 |
> | VIDIOC_DQBUF | 将视频缓冲区归回给设备， |
> | VIDIOC_OVERLAY | 开始或者停止overlay |
> | VIDIOC_G_FBUF | 获取video overlay设备或OSD设备的framebuffer参数 |
> | VIDIOC_S_FBUF | 设置framebuffer参数 |
> | VIDIOC_STREAMON | 开始流I/O操作，capture or output device |
> | VIDIOC_STREAMOFF | 关闭流I/O操作 |

##### 4、TV视频标准：

> **TV Standard**
> | ID | 描述 |
> | --- | --- |
> | VIDIOC_ENUMSTD | 枚举设备支持的所有标准 |
> | VIDIOC_G_STD | 获取当前正在使用的标准 |
> | VIDIOC_S_STD | 设置视频标准 |
> | VIDIOC_QUERYSTD | 有的设备支持自动侦测输入源的视频标准，此时使用此ioctl查询侦测到的视频标准 |

##### 5、input/output：

> **Input / Output**
> | ID | 描述 |
> | --- | --- |
> | VIDIOC_ENUMINPUT | 枚举所有input端口 |
> | VIDIOC_G_INPUT | 获取当前正在使用的input端口 |
> | VIDIOC_S_INPUT | 设置将要使用的input端口 |
> | VIDIOC_ENUMOUTPUT | 枚举所有output端口 |
> | VIDIOC_G_OUTPUT | 获取当前正在使用的output端口 |
> | VIDIOC_S_OUTPUT | 设置将要使用的output端口 |
> | VIDIOC_ENUMAUDIO | 枚举所有audio input端口 |
> | VIDIOC_G_AUDIO | 获取当前正在使用的audio input端口 |
> | VIDIOC_S_AUDIO | 设置将要使用的audio input端口 |
> | VIDIOC_ENUMAUDOUT | 枚举所有audio output端口 |
> | VIDIOC_G_AUDOUT | 获取当前正在使用的audio output端口 |
> | VIDIOC_S_AUDOUT | 设置将要使用的audio output端口 |

##### 6、controls：设备特定的控制，例如设置对比度，亮度

> **controls**
> | ID | 描述 |
> | --- | --- |
> | VIDIOC_QUERYCTRL | 查询指定control的详细信息 |
> | VIDIOC_G_CTRL | 获取指定control的值 |
> | VIDIOC_S_CTRL | 设置指定control的值 |
> | VIDIOC_G_EXT_CTRLS | 获取多个control的值 |
> | VIDIOC_S_EXT_CTRLS | 设置多个control的值 |
> | VIDIOC_TRY_EXT_CTRLS | 与VIDIOC_S_EXT_CTRLS相同，但是不改变设备状态 |
> | VIDIOC_QUERYMENU | 查询menu |

##### 7、其他杂项：

> **controls**
> | ID | 描述 |
> | --- | --- |
> | VIDIOC_G_MODULATOR |   |
> | VIDIOC_S_MODULATOR |   |
> | VIDIOC_G_CROP |   |
> | VIDIOC_S_CROP |   |
> | VIDIOC_G_SELECTION |   |
> | VIDIOC_S_SELECTION |   |
> | VIDIOC_CROPCAP |   |
> | VIDIOC_G_ENC_INDEX |   |
> | VIDIOC_ENCODER_CMD |   |
> | VIDIOC_TRY_ENCODER_CMD |   |
> | VIDIOC_DECODER_CMD |   |
> | VIDIOC_TRY_DECODER_CMD |   |
> | VIDIOC_G_PARM |   |
> | VIDIOC_S_PARM |   |
> | VIDIOC_G_TUNER |   |
> | VIDIOC_S_TUNER |   |
> | VIDIOC_G_FREQUENCY |   |
> | VIDIOC_S_FREQUENCY |   |
> | VIDIOC_G_SLICED_VBI_CAP |   |
> | VIDIOC_LOG_STATUS |   |
> | VIDIOC_DBG_G_CHIP_IDENT |   |
> | VIDIOC_S_HW_FREQ_SEEK |   |
> | VIDIOC_ENUM_FRAMESIZES |   |
> | VIDIOC_ENUM_FRAMEINTERVALS |   |
> | VIDIOC_ENUM_DV_PRESETS |   |
> | VIDIOC_S_DV_PRESET |   |
> | VIDIOC_G_DV_PRESET |   |
> | VIDIOC_QUERY_DV_PRESET |   |
> | VIDIOC_S_DV_TIMINGS |   |
> | VIDIOC_G_DV_TIMINGS |   |
> | VIDIOC_DQEVENT |   |
> | VIDIOC_SUBSCRIBE_EVENT |   |
> | VIDIOC_UNSUBSCRIBE_EVENT |   |
> | VIDIOC_CREATE_BUFS |   |
> | VIDIOC_PREPARE_BUF |   |

v4l2设备的基本操作流程如下：

1.  **打开设备**，例如 `fd = open("/dev/video0",0)`
2.  **查询设备能力**. 例如:

> struct capability cap;
> ioctl(fd,VIDIOC_QUERYCAP,&cap)

3.  **设置优先级(可选)**.
4.  **配置设备**。包括：

> *   视频输入源的视频标准，VIDIOC_\*_STD
> *   视频数据的格式 , VIDIOC_\*_FMT
> *   视频输入端口, VIDIOC_\*_INPUT
> *   视频输出端口，VIDIOC_\*_OUTPUT

5.  **启动设备开始I/O操作**。V4L2支持一下三种I/O方式：
    
    *   **Read/Write**：通过调用设备节点文件的Read/Write函数，与设备交互数据。打开设备后，默认使用的是此方法。
    *   **Stream I/O**：流操作，只传递数据缓冲区指针，不拷贝数据。使用此方法，需要调用VIDIOC_REQBUFS ioctl来通知设备。流操作I/O有两种方式memory map和user buffer。（具体区别后面章节介绍）
    *   **overlay** ： 也可以理解为memory to memory 传输。将数据从内存拷贝到显存中。overlay设备独有的。
    
    对于Capture device可以以如下方式启动设备：
    
    *   调用VIDIOC_REQBUFS ioctl来分配缓冲区队列；
    *   调用VIDIOC_STREAMON ioctl通知设备开始stream IO
    *   调用VIDIOC_QBUF ioctl从设备获取一帧视频数据；
    *   使用完数据后，调用VIDIOC_DQBUF将缓冲区还给设备，以便设备填充下一帧数据。
6.  **释放资源并关闭设备。** 

### （三）、从驱动层看V4L2

在驱动层，V4L2为驱动编写者做了很多工作。只需要实现硬件相关的代码，并且注册相关设备即可。

硬件相关代码的编写，除了编写具体硬件的控制代码外，最主要的就是将代码与V4L2框架绑定。绑定主要分为以下两个部分：

*   关系绑定：也就是要将我们自己的结构体，与V4L2框架中相关连的结构体绑定在一起。
*   IOCTL等函数绑定：将V4L2所定义的空的函数指针，与自己的函数绑定在一起。

#### 3.1、关系绑定

提到关系绑定，就必须介绍下V4L2几个重要结构体。

*   struct video_device：主要的任务就是负责向内核注册字符设备
*   struct v4l2_device：一个硬件设备可能包含多个子设备，比如一个电视卡除了有capture设备，可能还有VBI设备或者FM tunner。而v4l2_device就是所有这些设备的根节点，负责管理所有的子设备。
*   struct v4l2_subdev：子设备，负责实现具体的功能。
##### 3.1.1、video_device
**简介**
video_device是指向v4l2具体的设备，名字同样有些不够准确，事实上，根据注册时传入type的不同，可以分为视频输入，视频输出，VBI，Radio等。video_device结构体用于在/dev目录下生成设备节点文件，把操作设备的接口暴露给用户空间。

``` c
F:\Khadas_Edge_Android_Q\kernel\include\media\v4l2-dev.h
/*
 * Newer version of video_device, handled by videodev2.c
 *	This version moves redundant code from video device code to
 *	the common handler
 */

/**
 * struct video_device - Structure used to create and manage the V4L2 device
 *	nodes.
 *
 * @entity: &struct media_entity
 * @intf_devnode: pointer to &struct media_intf_devnode
 * @pipe: &struct media_pipeline
 * @fops: pointer to &struct v4l2_file_operations for the video device
 * @device_caps: device capabilities as used in v4l2_capabilities
 * @dev: &struct device for the video device
 * @cdev: character device
 * @v4l2_dev: pointer to &struct v4l2_device parent
 * @dev_parent: pointer to &struct device parent
 * @ctrl_handler: Control handler associated with this device node.
 *	 May be NULL.
 * @queue: &struct vb2_queue associated with this device node. May be NULL.
 * @prio: pointer to &struct v4l2_prio_state with device's Priority state.
 *	 If NULL, then v4l2_dev->prio will be used.
 * @name: video device name
 * @vfl_type: V4L device type, as defined by &enum vfl_devnode_type
 * @vfl_dir: V4L receiver, transmitter or m2m
 * @minor: device node 'minor'. It is set to -1 if the registration failed
 * @num: number of the video device node
 * @flags: video device flags. Use bitops to set/clear/test flags.
 *	   Contains a set of &enum v4l2_video_device_flags.
 * @index: attribute to differentiate multiple indices on one physical device
 * @fh_lock: Lock for all v4l2_fhs
 * @fh_list: List of &struct v4l2_fh
 * @dev_debug: Internal device debug flags, not for use by drivers
 * @tvnorms: Supported tv norms
 *
 * @release: video device release() callback
 * @ioctl_ops: pointer to &struct v4l2_ioctl_ops with ioctl callbacks
 *
 * @valid_ioctls: bitmap with the valid ioctls for this device
 * @lock: pointer to &struct mutex serialization lock
 *
 * .. note::
 *	Only set @dev_parent if that can't be deduced from @v4l2_dev.
 */

struct video_device
{
#if defined(CONFIG_MEDIA_CONTROLLER)
	struct media_entity entity;
	struct media_intf_devnode *intf_devnode;
	struct media_pipeline pipe;
#endif
    //pointer to &struct v4l2_file_operations for the video device
	const struct v4l2_file_operations *fops;
    
    //evice capabilities as used in v4l2_capabilities
	u32 device_caps;

	/* sysfs */
	struct device dev;
	struct cdev *cdev;

   /* Set either parent or v4l2_dev if your driver uses v4l2_device */
	struct v4l2_device *v4l2_dev; /* v4l2_device parent */
	struct device *dev_parent;    /* device parent */

    //Control handler associated with this device node. May be NULL.
	struct v4l2_ctrl_handler *ctrl_handler;

    //&struct vb2_queue associated with this device node. May be NULL.
	struct vb2_queue *queue;

    ///* Priority state. If NULL, then v4l2_dev->prio will be used. */
	struct v4l2_prio_state *prio;

	/* device info */
	char name[32]; //video device name
	enum vfl_devnode_type vfl_type; //V4L device type, as defined by &enum vfl_devnode_type
	enum vfl_devnode_direction vfl_dir;
	int minor;
	u16 num;
	unsigned long flags;
	int index;

	/* V4L2 file handles */
	spinlock_t		fh_lock;
	struct list_head	fh_list;

	int dev_debug;

	v4l2_std_id tvnorms; //Supported tv norms

	/* callbacks */
	void (*release)(struct video_device *vdev);
	const struct v4l2_ioctl_ops *ioctl_ops;
	DECLARE_BITMAP(valid_ioctls, BASE_VIDIOC_PRIVATE);

	struct mutex *lock;
};
```

##### 3.1.2、v4l2_device

``` c
F:\Khadas_Edge_Android_Q\kernel\include\media\v4l2-device.h
/**
 * struct v4l2_device - main struct to for V4L2 device drivers
 *
 * @dev: pointer to struct device.
 * @mdev: pointer to struct media_device, may be NULL.
 * @subdevs: used to keep track of the registered subdevs
 * @lock: lock this struct; can be used by the driver as well
 *	if this struct is embedded into a larger struct.
 * @name: unique device name, by default the driver name + bus ID
 * @notify: notify operation called by some sub-devices.
 * @ctrl_handler: The control handler. May be %NULL.
 * @prio: Device's priority state
 * @ref: Keep track of the references to this struct.
 * @release: Release function that is called when the ref count
 *	goes to 0.
 *
 * Each instance of a V4L2 device should create the v4l2_device struct,
 * either stand-alone or embedded in a larger struct.
 *
 * It allows easy access to sub-devices (see v4l2-subdev.h) and provides
 * basic V4L2 device-level support.
 *
 * .. note::
 *
 *    #) @dev->driver_data points to this struct.
 *    #) @dev might be %NULL if there is no parent device
 */
struct v4l2_device {
	struct device *dev;
	struct media_device *mdev;
	struct list_head subdevs;
	spinlock_t lock;
	char name[V4L2_DEVICE_NAME_SIZE];
	void (*notify)(struct v4l2_subdev *sd,
			unsigned int notification, void *arg);
	struct v4l2_ctrl_handler *ctrl_handler;
	struct v4l2_prio_state prio;
	struct kref ref;
	void (*release)(struct v4l2_device *v4l2_dev);
};
```

##### 3.1.3、v4l2_subdev

``` c
F:\Khadas_Edge_Android_Q\kernel\include\media\v4l2-subdev.h
/**
 * struct v4l2_subdev - describes a V4L2 sub-device
 *
 * @entity: pointer to &struct media_entity
 * @list: List of sub-devices
 * @owner: The owner is the same as the driver's &struct device owner.
 * @owner_v4l2_dev: true if the &sd->owner matches the owner of @v4l2_dev->dev
 *	owner. Initialized by v4l2_device_register_subdev().
 * @flags: subdev flags. Can be:
 *   %V4L2_SUBDEV_FL_IS_I2C - Set this flag if this subdev is a i2c device;
 *   %V4L2_SUBDEV_FL_IS_SPI - Set this flag if this subdev is a spi device;
 *   %V4L2_SUBDEV_FL_HAS_DEVNODE - Set this flag if this subdev needs a
 *   device node;
 *   %V4L2_SUBDEV_FL_HAS_EVENTS -  Set this flag if this subdev generates
 *   events.
 *
 * @v4l2_dev: pointer to struct &v4l2_device
 * @ops: pointer to struct &v4l2_subdev_ops
 * @internal_ops: pointer to struct &v4l2_subdev_internal_ops.
 *	Never call these internal ops from within a driver!
 * @ctrl_handler: The control handler of this subdev. May be NULL.
 * @name: Name of the sub-device. Please notice that the name must be unique.
 * @grp_id: can be used to group similar subdevs. Value is driver-specific
 * @dev_priv: pointer to private data
 * @host_priv: pointer to private data used by the device where the subdev
 *	is attached.
 * @devnode: subdev device node
 * @dev: pointer to the physical device, if any
 * @fwnode: The fwnode_handle of the subdev, usually the same as
 *	    either dev->of_node->fwnode or dev->fwnode (whichever is non-NULL).
 * @async_list: Links this subdev to a global subdev_list or @notifier->done
 *	list.
 * @asd: Pointer to respective &struct v4l2_async_subdev.
 * @notifier: Pointer to the managing notifier.
 * @subdev_notifier: A sub-device notifier implicitly registered for the sub-
 *		     device using v4l2_device_register_sensor_subdev().
 * @pdata: common part of subdevice platform data
 *
 * Each instance of a subdev driver should create this struct, either
 * stand-alone or embedded in a larger struct.
 *
 * This structure should be initialized by v4l2_subdev_init() or one of
 * its variants: v4l2_spi_subdev_init(), v4l2_i2c_subdev_init().
 */
struct v4l2_subdev {
#if defined(CONFIG_MEDIA_CONTROLLER)
	struct media_entity entity;
#endif
	struct list_head list;
	struct module *owner;
	bool owner_v4l2_dev;
	u32 flags;
	struct v4l2_device *v4l2_dev;
	const struct v4l2_subdev_ops *ops;
	const struct v4l2_subdev_internal_ops *internal_ops;
	struct v4l2_ctrl_handler *ctrl_handler;
	char name[V4L2_SUBDEV_NAME_SIZE];
	u32 grp_id;
	void *dev_priv;
	void *host_priv;
	struct video_device *devnode;
	struct device *dev;
	struct fwnode_handle *fwnode;
	struct list_head async_list;
	struct v4l2_async_subdev *asd;
	struct v4l2_async_notifier *notifier;
	struct v4l2_async_notifier *subdev_notifier;
	struct v4l2_subdev_platform_data *pdata;
};
```

v4l2_device,v4l2_subdev可以看作所有设备和子设备的基类。我们在编写自己的驱动时，往往需要继承这些设备基类，添加一些自己的数据成员。


**绑定的基本流程**

*   根据需要”重载”v4l2_device或v4l2_subdev结构体，添加需要的结构体成员。例如 ：

``` c
F:\Khadas_Edge_Android_Q\kernel\drivers\media\platform\rockchip\isp1\dev.h
/*
 * struct rkisp1_device - ISP platform device
 * @base_addr: base register address
 * @active_sensor: sensor in-use, set when streaming on
 * @isp_sdev: ISP sub-device
 * @rkisp1_stream: capture video device
 * @stats_vdev: ISP statistics output device
 * @params_vdev: ISP input parameters device
 */
struct rkisp1_device {
	struct list_head list;
	struct regmap *grf;
	void __iomem *base_addr;
	int irq;
	struct device *dev;
	struct clk *clks[RKISP1_MAX_BUS_CLK];
	int num_clks;
	struct v4l2_device v4l2_dev;
	struct v4l2_ctrl_handler ctrl_handler;
	struct media_device media_dev;
	struct v4l2_async_notifier notifier;
	struct v4l2_subdev *subdevs[RKISP1_SD_MAX];
	struct rkisp1_sensor_info *active_sensor;
	struct rkisp1_sensor_info sensors[RKISP1_MAX_SENSOR];
	int num_sensors;
	struct rkisp1_isp_subdev isp_sdev;
	struct rkisp1_stream stream[RKISP1_MAX_STREAM];
	struct rkisp1_isp_stats_vdev stats_vdev;
	struct rkisp1_isp_params_vdev params_vdev;
	struct rkisp1_dmarx_device dmarx_dev;
	struct rkisp1_pipeline pipe;
	struct iommu_domain *domain;
	enum rkisp1_isp_ver isp_ver;
	const unsigned int *clk_rate_tbl;
	int num_clk_rate_tbl;
	atomic_t open_cnt;
	struct rkisp1_emd_data emd_data_fifo[RKISP1_EMDDATA_FIFO_MAX];
	unsigned int emd_data_idx;
	unsigned int emd_vc;
	unsigned int emd_dt;
	int vs_irq;
	int mipi_irq;
	struct gpio_desc *vs_irq_gpio;
	struct v4l2_subdev *hdr_sensor;
	enum rkisp1_isp_state isp_state;
	unsigned int isp_err_cnt;
	enum rkisp1_isp_inp isp_inp;
	struct mutex apilock; /* mutex to serialize the calls of stream */
	struct mutex iqlock; /* mutex to serialize the calls of iq */
	wait_queue_head_t sync_onoff;
};
```

rkisp1_sensor_info 中”重载”了v4l2_subdev：

``` c
F:\Khadas_Edge_Android_Q\kernel\drivers\media\platform\rockchip\isp1\dev.h
/*
 * struct rkisp1_sensor_info - Sensor infomations
 * @mbus: media bus configuration
 */
struct rkisp1_sensor_info {
	struct v4l2_subdev *sd;
	struct v4l2_mbus_config mbus;
	struct v4l2_subdev_format fmt;
	struct v4l2_subdev_pad_config cfg;
};

```

*   v4l2_device与V4L2框架的绑定：通过调用v4l2_device_register函数实现。

``` c
F:\Khadas_Edge_Android_Q\kernel\drivers\media\platform\rockchip\isp1\dev.c
static int rkisp1_plat_probe(struct platform_device *pdev)
{
	printk("zjj.android10.kernel.camera || %s %s %d \n",
		__FUNCTION__, __FILE__, __LINE__); //zhoujinjian
	......
	ret = v4l2_device_register(isp_dev->dev, &isp_dev->v4l2_dev);
	......
}
```

*   v4l2_subdev与v4l2_device的绑定：通过v4l2_device_register_subdev函数，将subdev注册到根节点上。例如：
``` c
F:\Khadas_Edge_Android_Q\kernel\drivers\phy\rockchip\phy-rockchip-mipi-rx.c
static int rockchip_mipidphy_media_init(struct mipidphy_priv *priv)
{
	......
	return v4l2_async_register_subdev(&priv->sd);
}
```
*   video_device与v4l2_device的绑定：将v4l2_device的地址赋值给video_device的v4l2_dev即可。
此步不一定必要。只要有办法通过文件节点file(struct file)找到v4l2_device即可。

#### 3.2、函数绑定

其中v4l2_file_operations和v4l2_ioctl_ops是必须实现的。而v4l2_subdev_ops下的八类ops中，v4l2_subdev_core_ops是必须实现的，其余需要根据设备类型选择实现的。比如video capture类设备需要实现v4l2_subdev_core_ops, v4l2_subdev_video_ops。

*   v4l2_file_operations：实现文件类操作，比如open,close,read,write,mmap等。但是ioctl是不需要实现的，一般都是用video_ioctl2代替。

``` c
F:\Khadas_Edge_Android_Q\kernel\drivers\media\platform\rockchip\isp1\capture.c
static struct vb2_ops rkisp1_vb2_ops = {
	.queue_setup = rkisp1_queue_setup,
	.buf_queue = rkisp1_buf_queue,
	.wait_prepare = vb2_ops_wait_prepare,
	.wait_finish = vb2_ops_wait_finish,
	.stop_streaming = rkisp1_stop_streaming,
	.start_streaming = rkisp1_start_streaming,
};
static const struct v4l2_ioctl_ops rkisp1_v4l2_ioctl_ops = {
	.vidioc_reqbufs = vb2_ioctl_reqbufs,
	.vidioc_querybuf = vb2_ioctl_querybuf,
	.vidioc_create_bufs = vb2_ioctl_create_bufs,
	.vidioc_qbuf = vb2_ioctl_qbuf,
	.vidioc_expbuf = vb2_ioctl_expbuf,
	.vidioc_dqbuf = vb2_ioctl_dqbuf,
	.vidioc_prepare_buf = vb2_ioctl_prepare_buf,
	.vidioc_streamon = vb2_ioctl_streamon,
	.vidioc_streamoff = vb2_ioctl_streamoff,
	.vidioc_enum_input = rkisp1_enum_input,
	.vidioc_try_fmt_vid_cap_mplane = rkisp1_try_fmt_vid_cap_mplane,
	.vidioc_enum_fmt_vid_cap_mplane = rkisp1_enum_fmt_vid_cap_mplane,
	.vidioc_s_fmt_vid_cap_mplane = rkisp1_s_fmt_vid_cap_mplane,
	.vidioc_g_fmt_vid_cap_mplane = rkisp1_g_fmt_vid_cap_mplane,
	.vidioc_s_selection = rkisp1_s_selection,
	.vidioc_g_selection = rkisp1_g_selection,
	.vidioc_querycap = rkisp1_querycap,
	.vidioc_enum_frameintervals = rkisp_enum_frameintervals,
	.vidioc_enum_framesizes = rkisp_enum_framesizes,
};
```

*   v4l2_subdev_ops：v4l2_subdev有可能需要实现的ops的总合。分为8类，core,pad,video......等。例如，

``` c
F:\Khadas_Edge_Android_Q\kernel\drivers\phy\rockchip\phy-rockchip-mipi-rx.c
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
```
函数绑定只是将驱动所实现的函数赋值给相关的变量即可。


### （四）、V4L2设备注册过程及各个设备之间的联系
#### 1、v4l2_device_register
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

#### 2、video_register_device注册过程

##### 2.1、注册过程

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

##### 2.2、video在系统中的位置

```
	struct v4l2_device ------|                      struct video_device   
		|                    |------------引用----------> |----*v4l2_dev
        |                                                |
		|-->ctrlhandler<------------挂载------------------|----ctrl_handler
		|--->media->entry-list<------------挂载-----------|----->entry


```

#### 3、subdev注册过程

##### 3.1、v4l2_device_register_subdev()注册过程

```c
F:\Khadas_Edge_Android_Q\kernel\drivers\media\v4l2-core\v4l2-device.c
int v4l2_device_register_subdev(struct v4l2_device *v4l2_dev,
				struct v4l2_subdev *sd)
{
#if defined(CONFIG_MEDIA_CONTROLLER)
//获取到子设备的入口对象，方便后面注册到media_device上面。
	struct media_entity *entity = &sd->entity;
#endif
	int err;
　　 //此处省略一些错误检查代码
	/*
	 * The reason to acquire the module here is to avoid unloading
	 * a module of sub-device which is registered to a media
	 * device. To make it possible to unload modules for media
	 * devices that also register sub-devices, do not
	 * try_module_get() such sub-device owners.
	 */
	sd->owner_v4l2_dev = v4l2_dev->dev && v4l2_dev->dev->driver &&
		sd->owner == v4l2_dev->dev->driver->owner;

	if (!sd->owner_v4l2_dev && !try_module_get(sd->owner))
		return -ENODEV;
　　//下面v4l2_dev一般为系统根v4l2_device设备。
	sd->v4l2_dev = v4l2_dev;
	//这里对应具体的子设备，可以发现调用了registered()回调，如果有需要可以在对应的设备驱动中实现。
	if (sd->internal_ops && sd->internal_ops->registered) {
		err = sd->internal_ops->registered(sd);
		if (err)
			goto error_module;
	}
　　//印证了之前说的，子设备的ctrl_handler都会挂载到根设备v4l2_device的ctrl_handler上面。
	/* This just returns 0 if either of the two args is NULL */
	err = v4l2_ctrl_add_handler(v4l2_dev->ctrl_handler, sd->ctrl_handler, NULL);
	if (err)
		goto error_unregister;

#if defined(CONFIG_MEDIA_CONTROLLER)
	/* Register the entity. */
	if (v4l2_dev->mdev) {
	//上面将
		err = media_device_register_entity(v4l2_dev->mdev, entity);
		if (err < 0)
			goto error_unregister;
	}
#endif

	spin_lock(&v4l2_dev->lock);
	//将子设备链接到跟设备的,subdevs链表上。
	list_add_tail(&sd->list, &v4l2_dev->subdevs);
	spin_unlock(&v4l2_dev->lock);

	return 0;

error_unregister:
	if (sd->internal_ops && sd->internal_ops->unregistered)
		sd->internal_ops->unregistered(sd);
error_module:
	if (!sd->owner_v4l2_dev)
		module_put(sd->owner);
	sd->v4l2_dev = NULL;
	return err;
}
EXPORT_SYMBOL_GPL(v4l2_device_register_subdev);

```

*   获取到子设备的entity对象，并将当前根设备对象赋值给subdev->v4l2_dev域。
*   将子设备的ctrl_handler对象，添加到根设备的`v4l2_dev->ctrl_handler`域。
*   将子设备的entity注册到根设备上的media_device设备中。

这里只是将子设备挂接到v4l2_device的subdevs链表上，还没有真正的注册设备节点。真正注册设备节点的是`v4l2_device_register_subdev_nodes`方法。下面会介绍。

##### 3.2、v4l2_device_register_subdev_nodes()注册设备节点

```c
F:\Khadas_Edge_Android_Q\kernel\drivers\media\v4l2-core\v4l2-device.c
int v4l2_device_register_subdev_nodes(struct v4l2_device *v4l2_dev)
{
	struct video_device *vdev;
	struct v4l2_subdev *sd;
	int err;

	/* Register a device node for every subdev marked with the
	 * V4L2_SUBDEV_FL_HAS_DEVNODE flag.
	 */
	list_for_each_entry(sd, &v4l2_dev->subdevs, list) {
		if (!(sd->flags & V4L2_SUBDEV_FL_HAS_DEVNODE))
			continue;

		vdev = kzalloc(sizeof(*vdev), GFP_KERNEL);
		if (!vdev) {
			err = -ENOMEM;
			goto clean_up;
		}

		video_set_drvdata(vdev, sd);
		strlcpy(vdev->name, sd->name, sizeof(vdev->name));
		vdev->v4l2_dev = v4l2_dev;
		vdev->fops = &v4l2_subdev_fops;
		vdev->release = v4l2_device_release_subdev_node;
		vdev->ctrl_handler = sd->ctrl_handler;
		err = __video_register_device(vdev, VFL_TYPE_SUBDEV, -1, 1,
					      sd->owner);
		if (err < 0) {
			kfree(vdev);
			goto clean_up;
		}
#if defined(CONFIG_MEDIA_CONTROLLER)
		sd->entity.info.v4l.major = VIDEO_MAJOR;
		sd->entity.info.v4l.minor = vdev->minor;
#endif
		sd->devnode = vdev;
	}
	return 0;

clean_up:
	list_for_each_entry(sd, &v4l2_dev->subdevs, list) {
		if (!sd->devnode)
			break;
		video_unregister_device(sd->devnode);
	}

	return err;
}
EXPORT_SYMBOL_GPL(v4l2_device_register_subdev_nodes);

```

上面这个方法才是真正注册设备节点的地方，不过一般请看下不这样用，一般稍微修改一下`v4l2_device_register_subdev`方法，或者直接重新写一个注册方法，在注册设备到v4l2_device根设备上的同时注册设备文件到系统中。

##### 3.3、subdev在系统中的位置

``` c
 struct v4l2_device
 		|
 		|-------->subdevs <-----> list <-----> list <-----> list
 		|       					|           |
 						|---------------|   |--------------|
 						|struct sub_dev |   |struct sub_dev|
 						|---------------|   |--------------|

```


子设备就这样一个一个挂在v4l2_device设备上了。

#### 4、media_device_register()多媒体设备注册

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

#### 5、总结

上面media_device在v4l2中只用于遍历子设备，就没有描述出media_device在系统中的位置，等后面需要时在分析吧。总结到目前了解到v4l2需要一个根节点v4l2_device来管理子设备，然后上层用户空间，根据根节点，来查找所有子设备。针对高通平台，双摄camera来说。系统中存在3个media设备，分别对应3个物理camera.每一个camera对应的子设备都是根据media设备查找获取到的。


### （五）、V4L2学习之消息机制

v4l2的消息同大部分的消息组织形式类似，可以理解成都是以队列的形式，有人往里面push,有人去get。看了大半天才把消息队列的机制了解清楚。在了解之前需要重温一下下面几个结构体。  
#### 1、v4l2消息队列理解准备条件
v4l2消息结构体了解之前，仍然需要了解几个结构体。结构体中的数据域最好结合代码分析一下作用。

##### 1.1、struct v4l2_event
```c
F:\Khadas_Edge_Android_Q\kernel\include\uapi\linux\videodev2.h
struct v4l2_event {
	__u32				type;
	union {
		struct v4l2_event_vsync		vsync;
		struct v4l2_event_ctrl		ctrl;
		struct v4l2_event_frame_sync	frame_sync;
		struct v4l2_event_src_change	src_change;
		struct v4l2_event_motion_det	motion_det;
		__u8				data[64];
	} u;
	__u32				pending;
	__u32				sequence;
	struct timespec			timestamp;
	__u32				id;
	__u32				reserved[8];
};
```

这里主要说明下面几个数据域，其它用得少，暂时不记录。

*   type:毋庸置疑这是消息的类型,目前发现v4l2原生的类型包含下面这些，这些用户可以根据需要自己定制，最后一个私有的`V4L2_EVENT_PRIVATE_START`就是为扩展做准备的。

```c
/*
 *	E V E N T S
 */
#define V4L2_EVENT_ALL				0
#define V4L2_EVENT_VSYNC			1
#define V4L2_EVENT_EOS				2
#define V4L2_EVENT_CTRL				3
#define V4L2_EVENT_FRAME_SYNC			4
#define V4L2_EVENT_SOURCE_CHANGE		5
#define V4L2_EVENT_MOTION_DET			6
#define V4L2_EVENT_PRIVATE_START		0x08000000
```

*   data\[64\]:这里用户可以根据实际情况需要，用这64个字节定义自己的消息实体。
    
*   sequence:这个用来记录该消息的序列号(并不是index)，其和`struct v4l2_fh`中的sequence是一致的，后面会结合代码分析。
    
*   timestamp：消息发送的时间戳，代码中也没发现具体使用的地方，应该是用来判断消息生命周期准备吧。暂且不关心
    
*   pending:用来记录消息队列中尚未处理的消息数量。
    
*   id：命令id.

##### 1.2、struct v4l2_event_subscription

订阅消息发起时，订阅的消息用此结构体描述。此订阅发起者既可以是kernel模块，也可以是用户空间的管理者。这里第一次情调一下**消息只有订阅后，别的模块才能把消息放到模块的消息队列中，没有定义的消息是无法queue进去的**

```c
F:\Khadas_Edge_Android_Q\kernel\include\uapi\linux\videodev2.h
struct v4l2_event_subscription {
	__u32				type;
	__u32				id;
	__u32				flags;
	__u32				reserved[5];
};
```
##### 1.3、struct v4l2_fh

``` c
F:\Khadas_Edge_Android_Q\kernel\include\media\v4l2-fh.h
/**
 * struct v4l2_fh - Describes a V4L2 file handler
 *
 * @list: list of file handlers
 * @vdev: pointer to &struct video_device
 * @ctrl_handler: pointer to &struct v4l2_ctrl_handler
 * @prio: priority of the file handler, as defined by &enum v4l2_priority
 *
 * @wait: event' s wait queue
 * @subscribe_lock: serialise changes to the subscribed list; guarantee that
 *		    the add and del event callbacks are orderly called
 * @subscribed: list of subscribed events
 * @available: list of events waiting to be dequeued
 * @navailable: number of available events at @available list
 * @sequence: event sequence number
 *
 * @m2m_ctx: pointer to &struct v4l2_m2m_ctx
 */
struct v4l2_fh {
	struct list_head	list;
	struct video_device	*vdev;
	struct v4l2_ctrl_handler *ctrl_handler;
	enum v4l2_priority	prio;

	/* Events */
	wait_queue_head_t	wait;
	struct mutex		subscribe_lock;
	struct list_head	subscribed;
	struct list_head	available;
	unsigned int		navailable;
	u32			sequence;

#if IS_ENABLED(CONFIG_V4L2_MEM2MEM_DEV)
	struct v4l2_m2m_ctx	*m2m_ctx;
#endif
};
```

消息队列是用此结构体描述的。

*   list:链表入口，因为一个设备可能有好几条消息队列。(一般情况下打开一次设备会创建一个消息队列，不过一般请看下只会打开一次)
*   vdev：拥有此消息队列的设备
*   ctrl_handler：这其中也牵扯很多其它结构体，只需要了解这与控制元素相关就可以了。
*   prio：消息队列优先级。
*   wait：等待队列，由于可能有几个线程同时访问消息队列，所有等待消息队列可用的线程都会记录在这个等待列表中。
*   subscribed：这个是订阅消息链表。只有订阅的消息才能够queue到消息队列中，反之是无法queue到队列中的。
*   available：**消息队列，可用的消息都以链表的形式保存在这里。** 
*   navailable:消息队列中可用消息的数量。
*   sequence:序列号，此数据域会**一直自加下去**，直到内核对象消亡。

写到这里消息队列、订阅消息、就绪消息的存在形式和依赖关系应该初步了解了，他们的组织结构大体如下所示：  
![](https://raw.githubusercontent.com/zhoujinjianOS/PicGo/master/Android.10.Camera.02/v4l2_fh.png)


**注意：** 

*   available表示的是还没处理的消息，依然挂在链表中。
*   subscribed表示的是订阅的消息，单个订阅消息仍然会保存几个子消息(毕竟一个消息可以发送多次)。这里要特别注意**消息queue进来时，先会保存到subscribed对应的订阅消息列表，然后才会链接到available链表上**，上面**蓝色和橙色之间的箭头不一定是一一对应的**。具体为什么会保存多个自消息，可以继续了解一下订阅消息结构体。
##### 1.4、struct v4l2_subscribed_event

订阅消息是以`struct v4l2_subscribed_event`结构存在，其成员详细分布如下所示：

``` c
F:\Khadas_Edge_Android_Q\kernel\include\uapi\linux\videodev2.h
/**
 * struct v4l2_subscribed_event - Internal struct representing a subscribed
 *		event.
 *
 * @list:	List node for the v4l2_fh->subscribed list.
 * @type:	Event type.
 * @id:	Associated object ID (e.g. control ID). 0 if there isn't any.
 * @flags:	Copy of v4l2_event_subscription->flags.
 * @fh:	Filehandle that subscribed to this event.
 * @node:	List node that hooks into the object's event list
 *		(if there is one).
 * @ops:	v4l2_subscribed_event_ops
 * @elems:	The number of elements in the events array.
 * @first:	The index of the events containing the oldest available event.
 * @in_use:	The number of queued events.
 * @events:	An array of @elems events.
 */
struct v4l2_subscribed_event {
	struct list_head	list;
	u32			type;
	u32			id;
	u32			flags;
	struct v4l2_fh		*fh;
	struct list_head	node;
	const struct v4l2_subscribed_event_ops *ops;
	unsigned int		elems;
	unsigned int		first;
	unsigned int		in_use;
	struct v4l2_kevent	events[];
};

```
这了简单介绍几个重要变量

*   list:将订阅消息插入到消息队列里的链接指针
*   flags：一些标志位记录
*   fh:可以理解成，该消息挂在哪个消息队列之上。
*   replace:替换消息的方法，当订阅消息队列只能存放１个消息时，才会触发此替换方法。（**此方法在订阅消息队列满以及使用`struct v4l2_ctrl`控制**）
*   merge:当订阅消息队列有多余２个容量时，会触发merge方法（**此方法在订阅消息队列满以及使用`struct v4l2_ctrl`控制**）
*   elems:订阅消息队列中最多能存放消息的数量，此变量在订阅消息下发过来时确定，然后会由kernel分配对应的内存。
*   first：这个可以理解成游标指针，记录当前订阅消息队列中**入队时间最久且需要处理的消息**。
*   in_use：入队的消息数量
*   events:这里存放的只是一个指针，其大小由发起订阅时确定。

小结：此订阅消息，**可以理解成一类消息的组合**

#### 2、Enqueue消息

``` c
F:\Khadas_Edge_Android_Q\kernel\drivers\media\v4l2-core\v4l2-event.c
static void __v4l2_event_queue_fh(struct v4l2_fh *fh, const struct v4l2_event *ev,
		const struct timespec *ts)
{
	struct v4l2_subscribed_event *sev;
	struct v4l2_kevent *kev;
	bool copy_payload = true;

	/* Are we subscribed? */
	//这里特别要注意了，前面说过消息必须先订阅，才能queue进来，这了可以看到
	//当检测到订阅消息列表中，没有当前消息，则直接return.
	sev = v4l2_event_subscribed(fh, ev->type, ev->id);
	if (sev == NULL)
		return;

	/* Increase event sequence number on fh. */
	//消息序列号自动＋１．
	fh->sequence++;

	/* Do we have any free events? */
	//显然当in_use和elems相等时，此时消息数组已经满员了，此时需要将最旧的
	//没有处理的消息移除订阅消息队列。
	if (sev->in_use == sev->elems) {
		/* no, remove the oldest one */
		//队列中已经慢了，将第一个消息移除队列，腾出空间。
		kev = sev->events + sev_pos(sev, 0);
		list_del(&kev->list);
		sev->in_use--;　//如队列消息数目-1.
		sev->first = sev_pos(sev, 1); //将可用指针指向第二个消息实体
		fh->navailable--;//消息队列消息数量-1.
		if (sev->elems == 1) {
		//如果子消息数量只有１个，则替换该消息，注意由于都是同类消息，只需要改变状态即可。
			if (sev->replace) {
				sev->replace(&kev->event, ev);
				copy_payload = false;
			}
		} else if (sev->merge) {
			struct v4l2_kevent *second_oldest =
				sev->events + sev_pos(sev, 0);
			sev->merge(&kev->event, &second_oldest->event);
		}
	}

	/* Take one and fill it. */
	//这了取出in_use所对应的消息实体，来存放具体的消息内容。
	kev = sev->events + sev_pos(sev, sev->in_use);
	kev->event.type = ev->type;
	if (copy_payload)//拷贝数据内容
		kev->event.u = ev->u;
	kev->event.id = ev->id;
	kev->event.timestamp = *ts;
	kev->event.sequence = fh->sequence;
	sev->in_use++;//可用消息索引往后移动，相当于记录指针。
	//上面已经把消息添加到订阅消息子消息列表中了，种类把消息再次插入具体设备的消息队列中。
	list_add_tail(&kev->list, &fh->available);

	fh->navailable++;//需处理消息数量+1.

	wake_up_all(&fh->wait);//唤醒所有等待消息的线程
}

```

这里就是如队列消息的最重要的函数，此函数大概有下面几个流程。

*   1.先检查消息是否存在订阅消息列表中，不存在的话不允许该消息入队列，反之允许消息入队列。
*   2.当该消息对应的**订阅消息对象子消息满员之后**，则将子消息队列中最老的那个替换掉，以留给新的消息。
*   3.将新消息挂接到v4l2_fh可用链表中，以进行后续其它处理

注意：下图是`struct v4l2_subscribed_event`结构体中的`struct v4l2_kevent`子消息在订阅消息结构体中的存放形态。  
![](https://raw.githubusercontent.com/zhoujinjianOS/PicGo/master/Android.10.Camera.02/v4l2_subscribed_event.png)

上面子消息队列中已经入队了２个消息，此时in_use=2,而此时记录指针指向1,则first=1,剩下的红色都是空闲子消息缓存。子消息的结构如下所示：

```c
F:\Khadas_Edge_Android_Q\kernel\include\media\v4l2-event.h
struct v4l2_kevent {
	struct list_head	list;
	struct v4l2_subscribed_event *sev;
	struct v4l2_event	event;
};

```

#### 3、Dequeue消息

Dequeue操作就是取消息的操作(类似于取buffer时用`dequeue_buffer`），相比较消息的enqueue操作，dequeue操作简单多了。这里仍然根据代码流程来分析。

```c
//从参数中可以看到，v4l2_fh即为消息队列，该队列被一个设备对象所拥有
static int __v4l2_event_dequeue(struct v4l2_fh *fh, struct v4l2_event *event)
{
	struct v4l2_kevent *kev;
	unsigned long flags;

	spin_lock_irqsave(&fh->vdev->fh_lock, flags);
	//判断队列中是否有可用的消息，没有消息直接返回。
	if (list_empty(&fh->available)) {
		spin_unlock_irqrestore(&fh->vdev->fh_lock, flags);
		return -ENOENT;
	}

	WARN_ON(fh->navailable == 0);
	//从可用列表中取出第一个消息的入口，并将该消息从消息列表中删除。
	kev = list_first_entry(&fh->available, struct v4l2_kevent, list);
	list_del(&kev->list);
	fh->navailable--;//消息数量-1,
	//将当前还需处理的消息列表，复制给刚才取出来的消息，以便它做其它操作。
	kev->event.pending = fh->navailable;
	*event = kev->event;
	//将消息对应的订阅消息对象中的记录指针想后移动１个
	kev->sev->first = sev_pos(kev->sev, 1);
	kev->sev->in_use--;//入队列的消息数量-1.

	spin_unlock_irqrestore(&fh->vdev->fh_lock, flags);

	return 0;
}


```

取消息相对简单多了，大概就下面几步

*   1.检查消息列表是否是为空，不为空则从中取出子消息。
*   2.将上面子消息，从上面的消息列表中删除，并减少消息队列中消息的数量
*   3.将消息对应的订阅消息对象的记录指针后移，以及如队列消息-1.
#### 4、总结

看了半天代码就总结一句话“消息必须先订阅，才允许入队列”。同时也需要知道下面几点。

*   .订阅消息对象默认元素数量为１，如果设置的话就按设置的来分配内存。
![](https://raw.githubusercontent.com/zhoujinjianOS/PicGo/master/Android.10.Camera.02/V4L2_dev_node.png)

 ![](https://raw.githubusercontent.com/zhoujinjianOS/PicGo/master/Android.10.Camera.02/v4l2-framework.png)



### （六）、V4L2学习之-media_device
 > 本文对 V4L2 的运行时数据流设备管理做一个详细的介绍，包括什么叫「运行时设备管理」，它是干什么用的，怎么使用等等。本文的目标是掌握 media device 的编码使用方法以及功能运用。

#### 1、media framework
---------------

*   简介  
    相关的控制 API 在 `Documentation/DocBook/media/v4l/media-controller.xml`，本文档聚焦于内核测的media框架实现。**注意：直接查看是看不出啥的，在内核的根目录下 `make htmldocs` 或者其它格式的都行，易于查看。** 
    
*   运行时设备控制  
    也就是设备启动之后的数据流线路控制，就像一个工厂流水线一样，流水线上面的一个个节点（贴商标、喷丝印、打包）就形同于输入设备中的一个个子设备，运行时设备控制就是要达到能够控制节点的效果，比如贴商标的机器有好几台，应该选择哪一台进行此次流水线处理，要不要把喷丝印加上去，加哪一个机子等等。
    
*   作用  
    提供实时的 pipeline 管理，pipeline 就理解为管道，想象一下水管，里面的水就是数据流，输入设备中的 csi->isp->video 就组成了一个 pipeline 线路。media framework 提供 pipeline 的开启、关停、效果控制、节点控制等功能。
    
*   如何使用  
    内核当中主要利用四个结构体把众多的节点组织起来：`media_device`,`media_entity`,`media_link`,`media_pad`。整个 media framework 都是围绕这四个结构体来进行使用的，下文会对这些进行详细介绍。
    
*   抽象设备模型  
    media framework 其中一个目的是：在运行时状态下发现设备拓扑并对其进行配置。为了达到这个目的，media framework将硬件设备抽象为一个个的entity，它们之间通过links连接。
    

1.  entity：硬件设备模块抽象（类比电路板上面的各个元器件、芯片）
2.  pad：硬件设备端口抽象（类比元器件、芯片上面的管脚）
3.  link：硬件设备的连线抽象，link的两端是pad（类比元器件管脚之间的连线）

```
#------------#                #------------#
|          __|__            __|__          |
|         |  |  |   link   |  |  |         |
|         | pad |<-------->| pad |         |
|         |__|__|          |__|__|         |
|            |                |            |
|   entity   |                |   entity   |
#------------#                #------------# 

```

可以想象一下，如果各个 entity 之间需要建立连接的话，就需要在 pad 中存储 link 以及 entity 信息，link 中需要存储 pad 与 entity 信息，entity 里面需要存储 link 与 pad 信息，属于你中有我，我中有你的情况。

#### 2、media_device
--------

一个 media 设备用一个 `media_device` 结构体来表示，通常情况下该结构体要嵌入到一个更大的设备自定义的结构体里面，并且大多数时候 `media_device` 与 `v4l2_device` 是处于并列的级别，以rkisp1_device的代码为例：

``` c
F:\Khadas_Edge_Android_Q\kernel\drivers\media\platform\rockchip\isp1\dev.h
/*
 * struct rkisp1_device - ISP platform device
 * @base_addr: base register address
 * @active_sensor: sensor in-use, set when streaming on
 * @isp_sdev: ISP sub-device
 * @rkisp1_stream: capture video device
 * @stats_vdev: ISP statistics output device
 * @params_vdev: ISP input parameters device
 */
struct rkisp1_device {
	struct list_head list;
	struct regmap *grf;
	void __iomem *base_addr;
	int irq;
	struct device *dev;
	struct clk *clks[RKISP1_MAX_BUS_CLK];
	int num_clks;
	struct v4l2_device v4l2_dev;
	struct v4l2_ctrl_handler ctrl_handler;
	struct media_device media_dev;
	struct v4l2_async_notifier notifier;
	struct v4l2_subdev *subdevs[RKISP1_SD_MAX];
	struct rkisp1_sensor_info *active_sensor;
	struct rkisp1_sensor_info sensors[RKISP1_MAX_SENSOR];
	int num_sensors;
	struct rkisp1_isp_subdev isp_sdev;
	struct rkisp1_stream stream[RKISP1_MAX_STREAM];
	struct rkisp1_isp_stats_vdev stats_vdev;
	struct rkisp1_isp_params_vdev params_vdev;
	struct rkisp1_dmarx_device dmarx_dev;
	struct rkisp1_pipeline pipe;
	struct iommu_domain *domain;
	enum rkisp1_isp_ver isp_ver;
	const unsigned int *clk_rate_tbl;
	int num_clk_rate_tbl;
	atomic_t open_cnt;
	struct rkisp1_emd_data emd_data_fifo[RKISP1_EMDDATA_FIFO_MAX];
	unsigned int emd_data_idx;
	unsigned int emd_vc;
	unsigned int emd_dt;
	int vs_irq;
	int mipi_irq;
	struct gpio_desc *vs_irq_gpio;
	struct v4l2_subdev *hdr_sensor;
	enum rkisp1_isp_state isp_state;
	unsigned int isp_err_cnt;
	enum rkisp1_isp_inp isp_inp;
	struct mutex apilock; /* mutex to serialize the calls of stream */
	struct mutex iqlock; /* mutex to serialize the calls of iq */
	wait_queue_head_t sync_onoff;
};
```

使用以下函数进行 meida 设备的注册：`media_device_register(struct media_device *mdev);`  
函数的调用者需要在注册之前设置以下结构体成员（提前初始化该结构体是调用者的责任）：

``` c
F:\Khadas_Edge_Android_Q\kernel\include\media\media-device.h

/**
 * struct media_device - Media device
 * @dev:	Parent device
 * @devnode:	Media device node
 * @driver_name: Optional device driver name. If not set, calls to
 *		%MEDIA_IOC_DEVICE_INFO will return ``dev->driver->name``.
 *		This is needed for USB drivers for example, as otherwise
 *		they'll all appear as if the driver name was "usb".
 * @model:	Device model name
 * @serial:	Device serial number (optional)
 * @bus_info:	Unique and stable device location identifier
 * @hw_revision: Hardware device revision
 * @topology_version: Monotonic counter for storing the version of the graph
 *		topology. Should be incremented each time the topology changes.
 * @id:		Unique ID used on the last registered graph object
 * @entity_internal_idx: Unique internal entity ID used by the graph traversal
 *		algorithms
 * @entity_internal_idx_max: Allocated internal entity indices
 * @entities:	List of registered entities
 * @interfaces:	List of registered interfaces
 * @pads:	List of registered pads
 * @links:	List of registered links
 * @entity_notify: List of registered entity_notify callbacks
 * @graph_mutex: Protects access to struct media_device data
 * @pm_count_walk: Graph walk for power state walk. Access serialised using
 *		   graph_mutex.
 *
 * @source_priv: Driver Private data for enable/disable source handlers
 * @enable_source: Enable Source Handler function pointer
 * @disable_source: Disable Source Handler function pointer
 *
 * @ops:	Operation handler callbacks
 *
 * This structure represents an abstract high-level media device. It allows easy
 * access to entities and provides basic media device-level support. The
 * structure can be allocated directly or embedded in a larger structure.
 *
 * The parent @dev is a physical device. It must be set before registering the
 * media device.
 *
 * @model is a descriptive model name exported through sysfs. It doesn't have to
 * be unique.
 *
 * @enable_source is a handler to find source entity for the
 * sink entity  and activate the link between them if source
 * entity is free. Drivers should call this handler before
 * accessing the source.
 *
 * @disable_source is a handler to find source entity for the
 * sink entity  and deactivate the link between them. Drivers
 * should call this handler to release the source.
 *
 * Use-case: find tuner entity connected to the decoder
 * entity and check if it is available, and activate the
 * the link between them from @enable_source and deactivate
 * from @disable_source.
 *
 * .. note::
 *
 *    Bridge driver is expected to implement and set the
 *    handler when &media_device is registered or when
 *    bridge driver finds the media_device during probe.
 *    Bridge driver sets source_priv with information
 *    necessary to run @enable_source and @disable_source handlers.
 *    Callers should hold graph_mutex to access and call @enable_source
 *    and @disable_source handlers.
 */
struct media_device {
	/* dev->driver_data points to this struct. */
	struct device *dev;
	struct media_devnode *devnode;

	char model[32];
	char driver_name[32];
	char serial[40];
	char bus_info[32];
	u32 hw_revision;

	u64 topology_version;

	u32 id;
	struct ida entity_internal_idx;
	int entity_internal_idx_max;

	struct list_head entities;
	struct list_head interfaces;
	struct list_head pads;
	struct list_head links;

	/* notify callback list invoked when a new entity is registered */
	struct list_head entity_notify;

	/* Serializes graph operations. */
	struct mutex graph_mutex;
	struct media_graph pm_count_walk;

	void *source_priv;
	int (*enable_source)(struct media_entity *entity,
			     struct media_pipeline *pipe);
	void (*disable_source)(struct media_entity *entity);

	const struct media_device_ops *ops;
};
```

*   dev：必须指向一个父设备，通常是平台设备的device成员。
*   model：模型名字。  
    以下的成员是可选的：
*   serial：序列号，必须是唯一的
*   bus_info：总线信息，如果是PCI设备的话就可以设置成"PCI:"
*   hw_revision：硬件版本。可以的话，应该用KERNEL_VERSION宏定义进行格式化
*   driver_version：驱动版本。最终生成的设备节点的名称是media\[0-9\]，节点号由内核自动生成。

使用以下函数进行设备卸载：`media_device_unregister(struct media_device *mdev);`  
需要注意的是，卸载一个并没有注册过的设备是\*\*\*不安全的\*\*\*。**个人查看代码猜想不安全的原因主要有几个：1. 如果没有被注册，那么 `media_device` 内部的 entity 成员就有可能没有被初始化，如果其值为一个不确定的值，那么就会引起非法访问；2. 如果没有注册，内部的 `devnode` 成员就没有初始化，卸载时就会出现问题。** 

#### 3、「entities、pads、links」
---------------------
* struct video_device：视频设备结构体，通过video_register_device注册到系统后会在/dev/下生成videoX节点。在capture驱动中，为videoX节点配置了ops如v4l2_file_operations和v4l2_ioctl_ops。这样应用层就可以open、mmap和ioctl视频设备节点了。
* struct v4l2_device：v4l2设备结构体，在fimc的media_dev中创建的结构体。在注册这个设备的同时也注册了异步的
* subdev_notifier，有了这个subdev_notifier之后，只要subdev通过v4l2_async_register_subdev以异步的方式注册到系统，那么就会匹配到这个v4l2_device然后建立链接关系。
* struct v4l2_subdev：v4l2子设备结构体，在这里我有6个子设备：

``` c
rk3399_Android10:/ # grep '' /sys/class/video4linux/v4l-subdev*/name
/sys/class/video4linux/v4l-subdev0/name:rkisp1-isp-subdev
/sys/class/video4linux/v4l-subdev1/name:rockchip-mipi-dphy-rx
/sys/class/video4linux/v4l-subdev2/name:m00_b_imx214 6-0010
/sys/class/video4linux/v4l-subdev3/name:m00_b_vm149c 6-000c
/sys/class/video4linux/v4l-subdev4/name:rkisp1-isp-subdev
/sys/class/video4linux/v4l-subdev5/name:rockchip-mipi-dphy-rx
```

这里的imx214以异步的方式注册，所以会跟上面的v4l2_device建立联系。
* struct media_entity：属于media framework的概念，可以类比成电子元件。这个结构体代表一个media实体，它一般嵌入到更高级的一个数据结构中，比如video_device、v4l2_subdev中。
* struct media_pad：属于media framework的概念，可以类比成电子元件上面的引脚。比如我ov7740定义了一个source pad、fimc.0定义了属于video_device的一个sink pad、属于subdev的一个sink pad和src pad。
* struct media_link：用于链接的结构体，在调用media_create_pad_link的时候会生成一个link和一个back link，然后建立指定pad，以及对应的entity之间的链接。
通过对这些结构体的初始化和注册，结合media框架中提供的media_create_pad_link就会产生图中的链接图。media各个结构体的关系是你中有我，我中有你。这种密切的关系为media pipeline的运行时控制提供了基础。

#### 4、相关函数
----

建立media_entity与media_pad之间的链接：

```c
F:\Khadas_Edge_Android_Q\kernel\drivers\media\media-entity.c
int media_entity_pads_init(struct media_entity *entity, u16 num_pads,struct media_pad *pads){
	struct media_device *mdev = entity->graph_obj.mdev;
	.....
	entity->num_pads = num_pads;	
	entity->pads = pads;	//保存pad到entity
	for (i = 0; i < num_pads; i++) {
		pads[i].entity = entity;	//保存entity到pad
		pads[i].index = i;
		if (mdev)
			media_gobj_create(mdev, MEDIA_GRAPH_PAD,
					&entity->pads[i].graph_obj);
	}
}

```

建立media_device与media_entity之间的链接：

```c
video_register_device
	__video_register_device
		video_register_media_controller(vdev)
			media_device_register_entity(vdev->v4l2_dev->mdev, &vdev->entity);
				entity->graph_obj.mdev = mdev;

```

建立两个entity之间的链接：

```c
F:\Khadas_Edge_Android_Q\kernel\drivers\media\media-entity.c
int media_create_pad_link(struct media_entity *source, u16 source_pad,
			 struct media_entity *sink, u16 sink_pad, u32 flags){
	struct media_link *link;
	struct media_link *backlink;
	link = media_add_link(&source->links);
		link = kzalloc(sizeof(*link), GFP_KERNEL);
		list_add_tail(&link->list, head);
	link->source = &source->pads[source_pad];
	link->sink = &sink->pads[sink_pad];
	link->flags = flags & ~MEDIA_LNK_FL_INTERFACE_LINK;

	backlink = media_add_link(&sink->links);
	backlink->source = &source->pads[source_pad];
	backlink->sink = &sink->pads[sink_pad];
	backlink->flags = flags;
	backlink->is_backlink = true;

	link->reverse = backlink;
	backlink->reverse = link;
}

```

找到远程的pad：  
假如我这个pad是一个sink，那么我会通过link去找到它的远程source；假如我这个pad是一个source，那么我会通过link去找到它的远程sink。

```c
F:\Khadas_Edge_Android_Q\kernel\drivers\media\media-entity.c
struct media_pad *media_entity_remote_pad(const struct media_pad *pad){
	struct media_link *link;
	list_for_each_entry(link, &pad->entity->links, list) {
		if (!(link->flags & MEDIA_LNK_FL_ENABLED))//必须是已经使能的链接
			continue;
		if (link->source == pad)	
			return link->sink;
		if (link->sink == pad)
			return link->source;
	}
	return NULL;
}

```

启动一个pipeline。在启动流传输的时候需要调用media_pipeline_start去告知每一个entity，并设置对应的pad状态，然后调用entity->ops->link_validate(link)去验证每一个entity是否准备好了。这里传入的media_pipeline充当一个管理者角色，保存一些数据，保障后面的stop操作。

```c
F:\Khadas_Edge_Android_Q\kernel\drivers\media\media-entity.c
__must_check int media_pipeline_start(struct media_entity *entity,
				      struct media_pipeline *pipe){
	ret = __media_pipeline_start(entity, pipe);
	while ((entity = media_graph_walk_next(graph))) {
		entity->pipe = pipe;
		....
		list_for_each_entry(link, &entity->links, list) {
			struct media_pad *pad = link->sink->entity == entity
						? link->sink : link->source;
			....
			ret = entity->ops->link_validate(link);
		}
	}
}

```
#### 5、Rockchip-isp1-v4l2：
![](https://raw.githubusercontent.com/zhoujinjianOS/PicGo/master/Android.10.Camera.02/Rockchip-isp1-v4l2.png)
