---
title: Android 8.1 Display System源码分析（5）：ModeTest分析（RK3399）
cover: https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/personal.website/post.cover.pictures.00014.jpg
categories: 
  - Display
tags:
  - Android
  - Linux
  - Graphics
toc: true
abbrlink: 20200808
date: 2020-08-08 09:25:00
---

注：文章都是通过阅读 Android  && Linux 平台源码、各位前辈总结的资料、加上自己的思考分析总结出来的，其中难免有理不对的地方，欢迎大家批评指正。**文章为个人学习、研究、欣赏之用。图文内容、源码整理自互联网，如有侵权，请联系删除（◕‿◕）**，转载请注明出处（ ©Android @Linux @Rockchip版权所有），谢谢(๑乛◡乛๑) 、（ ͡° ͜ʖ ͡°）、（ಡωಡ）！！！。
（==**文章基于 Kernel-4.4**==）&&（==**文章基于 Android-8.1**==）
[【开发板 - Firefly-RK3399 7.85寸1024x768显示屏模组（MIPI接口）】](
http://wiki.t-firefly.com/zh_CN/Firefly-RK3399/compile_android8.1_firmware.html#)

正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢！！！
[（1）【DRM（Direct Rendering Manager）】](https://docs.nvidia.com/drive/nvvib_docs/NVIDIA%20DRIVE%20Linux%20SDK%20Development%20Guide/baggage/group__direct__rendering__manager.html) 
[（2）【DRM（Direct Rendering Manager）学习简介】](
https://blog.csdn.net/hexiaolong2009/article/details/83720940) 
[（3）【LCD显示异常分析——撕裂(tear effect)】](https://blog.csdn.net/hexiaolong2009/article/details/79319512) 

#### （一）、最简单的DRM应用程序 （single-buffer）

在学习 DRM 驱动之前，应该首先了解如何使用 DRM 驱动。以下使用伪代码的方式，简单介绍如何编写一个最简单的 DRM 应用程序。

伪代码：

``` c
int main(int argc, char **argv)
{
	/* open the drm device */
	open("/dev/dri/card0");

	/* get crtc/encoder/connector id */
	drmModeGetResources(...);

	/* get connector for display mode */
	drmModeGetConnector(...);

	/* create a dumb-buffer */
	drmIoctl(DRM_IOCTL_MODE_CREATE_DUMB);

	/* bind the dumb-buffer to an FB object */
	drmModeAddFB(...);

	/* map the dumb buffer for userspace drawing */
	drmIoctl(DRM_IOCTL_MODE_MAP_DUMB);
	mmap(...);

	/* start display */
	drmModeSetCrtc(crtc_id, fb_id, connector_id, mode);
}
```

当执行完`mmap`之后，我们就可以直接在应用层对 framebuffer 进行绘图操作了。

* * *

详细参考代码如下：

**modeset-single-buffer.c**

``` c
#define _GNU_SOURCE
#include <errno.h>
#include <fcntl.h>
#include <stdbool.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>
#include <time.h>
#include <unistd.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

struct buffer_object {
	uint32_t width;
	uint32_t height;
	uint32_t pitch;
	uint32_t handle;
	uint32_t size;
	uint8_t *vaddr;
	uint32_t fb_id;
};

struct buffer_object buf;

static int modeset_create_fb(int fd, struct buffer_object *bo)
{
	struct drm_mode_create_dumb create = {};
 	struct drm_mode_map_dumb map = {};

	/* create a dumb-buffer, the pixel format is XRGB888 */
	create.width = bo->width;
	create.height = bo->height;
	create.bpp = 32;
	drmIoctl(fd, DRM_IOCTL_MODE_CREATE_DUMB, &create);

	/* bind the dumb-buffer to an FB object */
	bo->pitch = create.pitch;
	bo->size = create.size;
	bo->handle = create.handle;
	drmModeAddFB(fd, bo->width, bo->height, 24, 32, bo->pitch,
			   bo->handle, &bo->fb_id);

	/* map the dumb-buffer to userspace */
	map.handle = create.handle;
	drmIoctl(fd, DRM_IOCTL_MODE_MAP_DUMB, &map);

	bo->vaddr = mmap(0, create.size, PROT_READ | PROT_WRITE,
			MAP_SHARED, fd, map.offset);

	/* initialize the dumb-buffer with white-color */
	memset(bo->vaddr, 0xff, bo->size);

	return 0;
}

static void modeset_destroy_fb(int fd, struct buffer_object *bo)
{
	struct drm_mode_destroy_dumb destroy = {};

	drmModeRmFB(fd, bo->fb_id);

	munmap(bo->vaddr, bo->size);

	destroy.handle = bo->handle;
	drmIoctl(fd, DRM_IOCTL_MODE_DESTROY_DUMB, &destroy);
}

int main(int argc, char **argv)
{
	int fd;
	drmModeConnector *conn;
	drmModeRes *res;
	uint32_t conn_id;
	uint32_t crtc_id;

	fd = open("/dev/dri/card0", O_RDWR | O_CLOEXEC);

	res = drmModeGetResources(fd);
	crtc_id = res->crtcs[0];
	conn_id = res->connectors[0];

	conn = drmModeGetConnector(fd, conn_id);
	buf.width = conn->modes[0].hdisplay;
	buf.height = conn->modes[0].vdisplay;

	modeset_create_fb(fd, &buf);

	drmModeSetCrtc(fd, crtc_id, buf.fb_id,
			0, 0, &conn_id, 1, &conn->modes[0]);

	getchar();

	modeset_destroy_fb(fd, &buf);

	drmModeFreeConnector(conn);
	drmModeFreeResources(res);

	close(fd);

	return 0;
}
```

上面代码有一个关键的函数，它就是`drmModeSetCrtc()`，该函数需要 _crtc_id_、_connector_id_、_fb_id_、_drm_mode_ 这几个参数。可以看到，几乎所有的代码都是为了该函数能够顺利传参而编写的：

> 为了获取 _crtc_id_ 和 _connector_id_，需要调用 `drmModeGetResources()`  
> 为了获取 _fb_id_，需要调用 `drmModeAddFB()`  
> 为了获取 _drm_mode_，需要调用 `drmModeGetConnector`

通过调用`drmModeSetCrtc()`，整个底层显示 pipeline 硬件就都初始化好了，并且还在屏幕上显示出了 FB 的内容，非常简单。

以上代码其实是基于 kernel DRM maintainer **David Herrmann** 所写的 [_drm-howto/modeset.c_](https://github.com/dvdhrm/docs/blob/master/drm-howto/modeset.c) 文件修改的，**需要注意的是，以上参考代码删除了许多异常错误处理，且只有在以下条件都满足时，才能正常运行：**

> 1.  DRM 驱动支持 MODESET；
> 2.  DRM 驱动支持 dumb-buffer(即连续物理内存)；
> 3.  DRM 驱动至少支持 1 个 CRTC，1 个 Encoder，1 个 Connector；
> 4.  DRM 驱动的 Connector 至少包含 1 个有效的 drm_display_mode。

**运行结果：**
![](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/zjj.sys.display.8.1.modetest/modetest.shot.single.buffer.jpg)
描述：程序运行后，显示全屏白色，等待用户输入按键；当用户按下任意按键后，程序退出，显示黑屏。

> 注意：程序运行之前，请确保没有其它应用或服务占用 / dev/dri/card0 节点，否则将出现 **Permission Denied** 错误。

**源码下载：**[modeset-plane-test](https://github.com/zhoujinjianok/sample-code)

#### （二）、最简单的DRM应用程序 （double-buffer）
使用上一节中的 modeset-single-buffer 程序，如果用户想要修改画面内容，就只能对`mmap()`后的 buffer 进行修改，这就会导致用户能很明显的在屏幕上看到软件修改 buffer 的过程，用户体验大大降低。而双 buffer 机制则能很好的避免这种问题，双 buffer 的概念无需过多赘述，大家听名字就知道什么意思了，即前后台 buffer 切换机制。

伪代码：

``` c
int main(void)
{
    ...
    while(1) {
        drmModeSetCrtc(fb0);
        ...
        drmModeSetCrtc(fb1);
        ...
    }
    ...
}
```

* * *

详细参考代码如下：

**modeset-double-buffer.c**

``` c
#define _GNU_SOURCE
#include <errno.h>
#include <fcntl.h>
#include <stdbool.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>
#include <time.h>
#include <unistd.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

struct buffer_object {
	uint32_t width;
	uint32_t height;
	uint32_t pitch;
	uint32_t handle;
	uint32_t size;
	uint32_t *vaddr;
	uint32_t fb_id;
};

struct buffer_object buf[2];

static int modeset_create_fb(int fd, struct buffer_object *bo, uint32_t color)
{
	struct drm_mode_create_dumb create = {};
 	struct drm_mode_map_dumb map = {};
	uint32_t i;

	create.width = bo->width;
	create.height = bo->height;
	create.bpp = 32;
	drmIoctl(fd, DRM_IOCTL_MODE_CREATE_DUMB, &create);

	bo->pitch = create.pitch;
	bo->size = create.size;
	bo->handle = create.handle;
	drmModeAddFB(fd, bo->width, bo->height, 24, 32, bo->pitch,
			   bo->handle, &bo->fb_id);

	map.handle = create.handle;
	drmIoctl(fd, DRM_IOCTL_MODE_MAP_DUMB, &map);

	bo->vaddr = mmap(0, create.size, PROT_READ | PROT_WRITE,
			MAP_SHARED, fd, map.offset);

	for (i = 0; i < (bo->size / 4); i++)
		bo->vaddr[i] = color;

	return 0;
}

static void modeset_destroy_fb(int fd, struct buffer_object *bo)
{
	struct drm_mode_destroy_dumb destroy = {};

	drmModeRmFB(fd, bo->fb_id);

	munmap(bo->vaddr, bo->size);

	destroy.handle = bo->handle;
	drmIoctl(fd, DRM_IOCTL_MODE_DESTROY_DUMB, &destroy);
}

int main(int argc, char **argv)
{
	int fd;
	drmModeConnector *conn;
	drmModeRes *res;
	uint32_t conn_id;
	uint32_t crtc_id;

	fd = open("/dev/dri/card0", O_RDWR | O_CLOEXEC);

	res = drmModeGetResources(fd);
	crtc_id = res->crtcs[0];
	conn_id = res->connectors[0];

	conn = drmModeGetConnector(fd, conn_id);
	buf[0].width = conn->modes[0].hdisplay;
	buf[0].height = conn->modes[0].vdisplay;
	buf[1].width = conn->modes[0].hdisplay;
	buf[1].height = conn->modes[0].vdisplay;

	modeset_create_fb(fd, &buf[0], 0xff0000);
	modeset_create_fb(fd, &buf[1], 0x0000ff);

	drmModeSetCrtc(fd, crtc_id, buf[0].fb_id,
			0, 0, &conn_id, 1, &conn->modes[0]);

	getchar();

	drmModeSetCrtc(fd, crtc_id, buf[1].fb_id,
			0, 0, &conn_id, 1, &conn->modes[0]);

	getchar();

	modeset_destroy_fb(fd, &buf[1]);
	modeset_destroy_fb(fd, &buf[0]);

	drmModeFreeConnector(conn);
	drmModeFreeResources(res);

	close(fd);

	return 0;
}
```

以上代码是基于 **David Herrmann** 所写的 [_drm-howto/modeset-double-buffered.c_](https://github.com/dvdhrm/docs/blob/master/drm-howto/modeset-double-buffered.c) 文件修改的。和 modeset-single-buffer 案例一样，为了方便大家关注重点，以上代码删除了许多异常错误处理，并对程序功能做了简化。

从上面的代码我们可以看出，`drmModeSetCrtc()` 的功能除了可以初始化整条显示 pipeline，建立 crtc 到 connector 之间的连接关系外，**它还可以更新屏幕显示内容**，即通过修改 fb_id，来完成显示 buffer 的切换。

> 有的同学可能会担心，重复调用`drmModeSetCrtc()`会导致硬件链路被重复初始化。其实不必担心，因为 DRM 驱动框架会对传入的参数进行检查，只要 display mode 和 pipeline 链路连接关系没有发生变化，就不会重新初始化硬件。

**运行结果：**
![](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/zjj.sys.display.8.1.modetest/modetest.shot.double.buffer.jpg)

![](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/zjj.sys.display.8.1.modetest/modetest.shot.double.buffer2.jpg)

描述：程序运行后，屏幕显示红色；输入回车后，屏幕显示蓝色(Firefly-RK3399 Android 8.1异常，显示成绿色了)；再次输入回车后，程序退出。

> 注意：程序运行之前，请确保没有其它应用或服务占用 / dev/dri/card0 节点，否则将出现 **Permission Denied** 错误。

**源码下载：**[modeset-plane-test](https://github.com/zhoujinjianok/sample-code)
#### （三）、最简单的DRM应用程序 （page-flip）

在上一节 最简单的 DRM 应用程序 （double-buffer）中，我们了解了 DRM 更新图像的一个重要接口`drmModeSetCrtc()`。在本篇文章中，我们将一起来学习 DRM 另一个重要的刷图接口：`drmModePageFlip()`。

`drmModePageFlip()`的功能也是用于更新显示内容的，但是它和`drmModeSetCrtc()`最大的区别在于，`drmModePageFlip()`只会等到 VSYNC 到来后才会真正执行 framebuffer 切换动作，而`drmModeSetCrtc()`则会立即执行 framebuffer 切换动作。很明显，`drmModeSetCrtc()`对于某些硬件来说，很容易造成 [_**撕裂（tear effect）**_见参考文档]问题，而`drmModePageFlip()`则不会造成这种问题。

由于`drmModePageFlip()`本身是基于 VSYNC 事件机制的，**因此底层 DRM 驱动必须支持 VBLANK 事件**。

伪代码：

``` c
void my_page_flip_handler(...)
{
	drmModePageFlip(DRM_MODE_PAGE_FLIP_EVENT);
	...
}

int main(void)
{
	drmEventContext ev = {};

	ev.version = DRM_EVENT_CONTEXT_VERSION;
	ev.page_flip_handler = my_page_flip_handler;
	...

	drmModePageFlip(DRM_MODE_PAGE_FLIP_EVENT);
	
	while (1) {
		drmHandleEvent(&ev);
	}
}
```

* * *

详细参考代码如下：

**modeset-page-flip.c**

``` c
#define _GNU_SOURCE
#include <errno.h>
#include <fcntl.h>
#include <stdbool.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>
#include <time.h>
#include <unistd.h>
#include <signal.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

struct buffer_object {
	uint32_t width;
	uint32_t height;
	uint32_t pitch;
	uint32_t handle;
	uint32_t size;
	uint32_t *vaddr;
	uint32_t fb_id;
};

struct buffer_object buf[2];
static int terminate;

static int modeset_create_fb(int fd, struct buffer_object *bo, uint32_t color)
{
	struct drm_mode_create_dumb create = {};
 	struct drm_mode_map_dumb map = {};
	uint32_t i;

	create.width = bo->width;
	create.height = bo->height;
	create.bpp = 32;
	drmIoctl(fd, DRM_IOCTL_MODE_CREATE_DUMB, &create);

	bo->pitch = create.pitch;
	bo->size = create.size;
	bo->handle = create.handle;
	drmModeAddFB(fd, bo->width, bo->height, 24, 32, bo->pitch,
			   bo->handle, &bo->fb_id);

	map.handle = create.handle;
	drmIoctl(fd, DRM_IOCTL_MODE_MAP_DUMB, &map);

	bo->vaddr = mmap(0, create.size, PROT_READ | PROT_WRITE,
			MAP_SHARED, fd, map.offset);

	for (i = 0; i < (bo->size / 4); i++)
		bo->vaddr[i] = color;

	return 0;
}

static void modeset_destroy_fb(int fd, struct buffer_object *bo)
{
	struct drm_mode_destroy_dumb destroy = {};

	drmModeRmFB(fd, bo->fb_id);

	munmap(bo->vaddr, bo->size);

	destroy.handle = bo->handle;
	drmIoctl(fd, DRM_IOCTL_MODE_DESTROY_DUMB, &destroy);
}

static void modeset_page_flip_handler(int fd, uint32_t frame,
				    uint32_t sec, uint32_t usec,
				    void *data)
{
	static int i = 0;
	uint32_t crtc_id = *(uint32_t *)data;

	i ^= 1;

	drmModePageFlip(fd, crtc_id, buf[i].fb_id,
			DRM_MODE_PAGE_FLIP_EVENT, data);

	usleep(500000);
}

static void sigint_handler(int arg)
{
	terminate = 1;
}

int main(int argc, char **argv)
{
	int fd;
	drmEventContext ev = {};
	drmModeConnector *conn;
	drmModeRes *res;
	uint32_t conn_id;
	uint32_t crtc_id;

	/* register CTRL+C terminate interrupt */
	signal(SIGINT, sigint_handler);

	ev.version = DRM_EVENT_CONTEXT_VERSION;
	ev.page_flip_handler = modeset_page_flip_handler;

	fd = open("/dev/dri/card0", O_RDWR | O_CLOEXEC);

	res = drmModeGetResources(fd);
	crtc_id = res->crtcs[0];
	conn_id = res->connectors[0];

	conn = drmModeGetConnector(fd, conn_id);
	buf[0].width = conn->modes[0].hdisplay;
	buf[0].height = conn->modes[0].vdisplay;
	buf[1].width = conn->modes[0].hdisplay;
	buf[1].height = conn->modes[0].vdisplay;

	modeset_create_fb(fd, &buf[0], 0xff0000);
	modeset_create_fb(fd, &buf[1], 0x0000ff);

	drmModeSetCrtc(fd, crtc_id, buf[0].fb_id,
			0, 0, &conn_id, 1, &conn->modes[0]);

	drmModePageFlip(fd, crtc_id, buf[0].fb_id,
			DRM_MODE_PAGE_FLIP_EVENT, &crtc_id);

	while (!terminate) {
		drmHandleEvent(fd, &ev);
	}

	modeset_destroy_fb(fd, &buf[1]);
	modeset_destroy_fb(fd, &buf[0]);

	drmModeFreeConnector(conn);
	drmModeFreeResources(res);

	close(fd);

	return 0;
}
```

以上代码是基于 **David Herrmann** 所写的 [_drm-howto/modeset-vsync.c_](https://github.com/dvdhrm/docs/blob/master/drm-howto/modeset-vsync.c) 文件修改的。和前两篇的案例一样，为了方便大家关注重点，以上代码删除了许多异常错误处理，并对程序功能做了简化。

从上面的代码可以看出，要使用`drmModePageFlip()`，就必须依赖`drmHandleEvent()`函数，该函数内部以阻塞的形式等待底层驱动返回相应的 vblank 事件，以确保和 VSYNC 同步。**需要注意的是，`drmModePageFlip()`不允许在 1 个 VSYNC 周期内被调用多次，否则只有第一次调用有效，后面几次调用都会返回`-EBUSY`错误（-16）**。

**运行结果：**
![](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/zjj.sys.display.8.1.modetest/modtest.shot.page.flip.gif)
描述：程序运行后，屏幕在红色和蓝色(Firefly-RK3399 Android 8.1异常，显示成绿色了)之间来回切换；当输入`CTRL+C`后，程序退出。

> 注意：程序运行之前，请确保没有其它应用或服务占用 / dev/dri/card0 节点，否则将出现 **Permission Denied** 错误。

**源码下载：**[modeset-plane-test](https://github.com/zhoujinjianok/sample-code)

#### （四）、最简单的DRM应用程序 （plane-test）
在上一篇 最简单的 DRM 应用程序 （page-flip）中，我们学习了`drmModePageFlip()`的用法。而在更早的两篇文章中，我们还学习了`drmModeSetCrtc()`的使用方法。但是这两个接口都只能全屏显示 framebuffer 的内容，如何才能在屏幕上只显示 framebuffer 的一部分内容呢？本篇我们将一起来学习 DRM 另一个重要的刷图接口：`drmModeSetPlane()`。

在学习该函数之前，我们首先来了解一下，什么是 Plane？

DRM 中的 Plane 和我们常说的 YUV/YCbCr 图形格式中的 plane 完全是两个不同的概念。YUV 图形格式中的 plane 指的是图像数据在内存中的排列形式，一般 Y 通道占一段连续的内存块，UV 通道占另一段连续的内存块，我们称之为 YUV-2plane (也叫 YUV 2 平面)，属于软件层面。而 DRM 中的 Plane 指的是 Display Controller 中用于多层合成的单个硬件图层模块，属于硬件层面。二者概念上不要混淆。

> Plane 的历史
> 
> 随着软件技术的不断更新，对硬件的性能要求越来越高，在满足功能正常使用的前提下，对功耗的要求也越来越苛刻。本来 GPU 可以处理所有图形任务，但是由于它运行时的功耗实在太高，设计者们决定将一部分简单的任务交给 Display Controller 去处理（比如合成），而让 GPU 专注于绘图（即渲染）这一主要任务，减轻 GPU 的负担，从而达到降低功耗提升性能的目的。于是，Plane（硬件图层单元）就诞生了。

**Plane 是连接 FB 与 CRTC 的纽带，是内存的搬运工。**

伪代码：

``` c
int main(void)
{
	...
	drmSetClientCap(DRM_CLIENT_CAP_UNIVERSAL_PLANES);
	drmModeGetPlaneResources();

	drmModeSetPlane(plane_id, crtc_id, fb_id, 0,
			crtc_x, crtc_y, crtc_w, crtc_h,
			src_x << 16, src_y << 16, src_w << 16, src_h << 16);
	...
}
```

先来了解一下`drmModeSetPlane()`参数含义：

![](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/zjj.sys.display.8.1.modetest/modetest.shot.plane-draft.png)
（上图实现了裁剪、平移和放大的效果）

> 当 SRC 与 CRTC 的 X/Y 不相等时，则实现了平移的效果；  
> 当 SRC 与 CRTC 的 W/H 不相等时，则实现了缩放的效果；  
> 当 SRC 与 FrameBuffer 的 W/H 不相等时，则实现了裁剪的效果；

一个高级的 Plane，通常具有如下功能：

| 功能 | 说明 |
| :-- | :-- |
| Crop | 裁剪，如上图 |
| Scaling | 缩放，放大或缩小 |
| Rotation | 旋转，90° 180° 270° X/Y 镜像 |
| Z-Order | Z - 顺序，调整当前层在总图层中的 Z 轴顺序 |
| Blending | 合成，pixel alpha / global alpha |
| Format | 颜色格式，ARGB888 XRGB888 YUV420 等 |

**再次强调，以上这些功能都是由硬件直接完成的，而非软件实现。**

在 DRM 框架中，Plane 又分为如下 3 种类型：

| 类型 | 说明 |
| :-- | :-- |
| Cursor | 光标图层，一般用于 PC 系统，用于显示鼠标 |
| Overlay | 叠加图层，通常用于 YUV 格式的视频图层 |
| Primary | 主要图层，通常用于仅支持 RGB 格式的简单图层 |

> 其实随着现代半导体技术的飞速发展，`Overlay Plane`和`Primary Plane`之间已经没有明显的界限了，许多芯片的图层处理能力已经非常强大，不仅仅可以处理简单的 RGB 格式，也可以处理 YUV 视频格式，甚至 FBC 压缩格式。针对这类硬件图层，它既可以是`Overlay Plane`，也可以是`Primary Plane`，至于驱动如何定义，就要看工程师的喜好了。  
> 而对于一些早期处理能力比较弱的硬件，为了节约成本，每个图层支持的格式并不一样，比如将平常使用格式最多的 RGB 图层作为`Primary Plane`，而将平时用不多的 YUV 视频图层作为`Overlay Plane`，那么这个时候上层应用程序在使用这两种 plane 的时候就需要区别对待了。

需要注意的是，并不是所有的 Display Controller 都支持 Plane，从前面 [_single-buffer_] 案例中的`drmModeSetCrtc()`函数也能看出，即使没有 plane_id，屏幕也能正常显示。比如 s3c2440 这种骨灰级 ARM9 SoC，它的 LCDC 就没有 Plane 的概念。但是 **DRM 框架规定，任何一个 CRTC，必须要有 1 个**`Primary Plane`。 即使像 S3C2440 这种不带真实 Plane 硬件的 Display Controller，我们也认为它的 Primary Plane 就是 LCDC 本身，因为它实现了从 Framebuffer 到 CRTC 的数据搬运工作，而这正是一个 Plane 最基本的功能。

**为什么要设置`DRM_CLIENT_CAP_UNIVERSAL_PLANES` ？**

> 因为如果不设置，`drmModeGetPlaneResources()`就只会返回 Overlay Plane，其他 Plane 都不会返回。而如果设置了，DRM 驱动则会返回所有支持的 Plane 资源，包括 cursor、overlay 和 primary。

* * *

详细参考代码如下：

**modeset-plane-test.c**

``` c
#define _GNU_SOURCE
#include <errno.h>
#include <fcntl.h>
#include <stdbool.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>
#include <time.h>
#include <unistd.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

struct buffer_object {
	uint32_t width;
	uint32_t height;
	uint32_t pitch;
	uint32_t handle;
	uint32_t size;
	uint8_t *vaddr;
	uint32_t fb_id;
};

struct buffer_object buf;

static int modeset_create_fb(int fd, struct buffer_object *bo)
{
	struct drm_mode_create_dumb create = {};
 	struct drm_mode_map_dumb map = {};

	create.width = bo->width;
	create.height = bo->height;
	create.bpp = 32;
	drmIoctl(fd, DRM_IOCTL_MODE_CREATE_DUMB, &create);

	bo->pitch = create.pitch;
	bo->size = create.size;
	bo->handle = create.handle;
	drmModeAddFB(fd, bo->width, bo->height, 24, 32, bo->pitch,
			   bo->handle, &bo->fb_id);

	map.handle = create.handle;
	drmIoctl(fd, DRM_IOCTL_MODE_MAP_DUMB, &map);

	bo->vaddr = mmap(0, create.size, PROT_READ | PROT_WRITE,
			MAP_SHARED, fd, map.offset);

	memset(bo->vaddr, 0xff, bo->size);

	return 0;
}

static void modeset_destroy_fb(int fd, struct buffer_object *bo)
{
	struct drm_mode_destroy_dumb destroy = {};

	drmModeRmFB(fd, bo->fb_id);

	munmap(bo->vaddr, bo->size);

	destroy.handle = bo->handle;
	drmIoctl(fd, DRM_IOCTL_MODE_DESTROY_DUMB, &destroy);
}

int main(int argc, char **argv)
{
	int fd;
	drmModeConnector *conn;
	drmModeRes *res;
	drmModePlaneRes *plane_res;
	uint32_t conn_id;
	uint32_t crtc_id;
	uint32_t plane_id;

	fd = open("/dev/dri/card0", O_RDWR | O_CLOEXEC);

	res = drmModeGetResources(fd);
	crtc_id = res->crtcs[0];
	conn_id = res->connectors[0];

	drmSetClientCap(fd, DRM_CLIENT_CAP_UNIVERSAL_PLANES, 1);
	plane_res = drmModeGetPlaneResources(fd);
	plane_id = plane_res->planes[0];

	conn = drmModeGetConnector(fd, conn_id);
	buf.width = conn->modes[0].hdisplay;
	buf.height = conn->modes[0].vdisplay;

	modeset_create_fb(fd, &buf);

	drmModeSetCrtc(fd, crtc_id, buf.fb_id,
			0, 0, &conn_id, 1, &conn->modes[0]);

	getchar();

	/* crop the rect from framebuffer(100, 150) to crtc(50, 50) */
	drmModeSetPlane(fd, plane_id, crtc_id, buf.fb_id, 0,
			50, 50, 320, 320,
			100 << 16, 150 << 16, 320 << 16, 320 << 16);

	getchar();

	modeset_destroy_fb(fd, &buf);

	drmModeFreeConnector(conn);
	drmModeFreePlaneResources(plane_res);
	drmModeFreeResources(res);

	close(fd);

	return 0;
}
```

以上代码参考 Google Android 工程中 [external/libdrm/tests/planetest/planetest.c](https://github.com/zhoujinjianok/libdrm/blob/master/tests/planetest/planetest.c) 文件，为了演示方便，仅仅实现了一个最简单的`drmModeSetPlane()`调用。**需要注意的是，该函数调用之前，必须先通过`drmModeSetCrtc()`初始化整个显示链路，否则 Plane 设置将无效。**

**运行结果：**（模拟效果）  
![](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/zjj.sys.display.8.1.modetest/modetest.shot.plane.test.jpg)
![](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/zjj.sys.display.8.1.modetest/modetest.shot.plane.test2.jpg)

描述：程序运行后，屏幕显示全屏白色；当输入回车后，屏幕将 framebuffer 中的（100，150）的矩形，显示到屏幕的（50，50）位置；再次输入回车后，程序退出。

> 注意：程序运行之前，请确保没有其它应用或服务占用 / dev/dri/card0 节点，否则将出现 **Permission Denied** 错误。

**源码下载：**[modeset-plane-test](https://github.com/zhoujinjianok/sample-code)
#### （五）、DRM应用程序进阶 （Property）
通过前面几节[《最简单的 DRM 应用程序》]系列文章，我们学习了如何编写一个最基本的 DRM 应用程序。但是，这些程序所使用的接口，在如今的 DRM 架构中其实早已经被标记为 **Legacy（过时的）** 接口了，而目前 DRM 主要推荐使用的是 **Atomic（原子的）** 接口。Atomic 接口我会在下篇文章中重点介绍，本篇主要介绍 Atomic 操作必须依赖的基本元素，**Property（属性）**。


所谓 Property，其实就是把前几篇的 legacy 接口传入的参数单独抽出来，做成一个个独立的全局属性。通过设置这些属性参数，即可完成对显示参数的设置。

Property 的结构简单概括主要由 3 部分组成：`name`、`id` 和 `value`。其中`id`为该 property 在 DRM 框架中全局唯一的标识符。

采用 property 机制的好处是：

> 1.  减少上层应用接口的维护工作量。当开发者有新的功能需要添加时，无需增加新的函数名和 IOCTL，只需在底层驱动中新增一个 property，然后在自己的应用程序中获取 / 操作该 property 的值即可。
> 2.  增强了参数设置的灵活性。一次 IOCTL 可以同时设置多个 property，减少了 user space 与 kernel space 切换的次数，同时最大限度的满足了不同硬件对于参数设置的要求，提高了软件效率。

DRM 中的 property 大多以功能进行划分，并且还定义了一组 _**Standard Properties**_，这些标准 properties 在任何平台上都会被创建。

下表列出了应用程序开发中，常用的 property：

* * *

######  CRTC

| name | desctription |
| :-- | :-- |
| ACTIVE | CRTC 当前的使能状态，一般用于控制 CRTC 上下电 |
| MODE_ID | CRTC 当前所使用的 display mode ID，通过该 ID 可以找到具体的 display mode 配置参数 |
| OUT_FENCE_PTR | 输出 fence 指针，指向当前正在显示的 buffer 所对应的 fence fd，该 fence 由 DRM 驱动创建，供上层应用程序使用，用来表示当前 buffer CRTC 是否还在占用 |

(optional)

| name | desctription |
| :-- | :-- |
| DEGAMMA_LUT | de-gamma 查找表参数 |
| DEGAMMA_LUT_SIZE | de-gamma 查找表参数长度 |
| CTM | Color Transformation Matrix，颜色矩阵转换参数，3x3 的矩阵 |
| GAMMA_LUT | gamma 查找表参数 |
| GAMMA_LUT_SIZE | gamma 查找表参数长度 |

* * *

######  PLANE

| name | desctription |
| :-- | :-- |
| type | plane 的类型，CURSOR、PRIMARY 或者 OVERLAY |
| FB_ID | 与当前 plane 绑定的 framebuffer object ID |
| IN_FENCE_FD | 与当前 plane 相关联的 input fence fd，由 buffer 的生产者创建，供 DRM 底层驱动使用，用来标识当前传下来的 buffer 是否可以开始访问 |
| CRTC_ID | 当前 plane 所关联的 CRTC object ID，与 CONNECTOR 中的 CRTC_ID 属性是同一个 property |
| SRC_X | 当前 framebuffer crop 区域的起始偏移 x 坐标 |
| SRC_Y | 当前 framebuffer crop 区域的起始偏移 y 坐标 |
| SRC_W | 当前 framebuffer crop 区域的宽度 |
| SRC_H | 当前 framebuffer crop 区域的高度 |
| CRTC_X | 屏幕显示区域的起始偏移 x 坐标 |
| CRTC_Y | 屏幕显示区域的起始偏移 y 坐标 |
| CRTC_W | 屏幕显示区域的宽度 |
| CRTC_H | 屏幕显示区域的高度 |

(optional)

| name | desctription |
| :-- | :-- |
| IN_FORMATS | 用于标识特殊的颜色存储格式，如 AFBC、IFBC 存储格式，该属性为只读 |
| rotation | 当前图层的旋转角度 |
| zposition | 当前图层在所有图层中的 Z 轴顺序 |
| alpha | 当前图层的 global alpha（非 pixel alpha），用于多层合成 |
| pixel blend mode | 当前图层的合成方式，如 Pre-multiplied/Coverage 等 |

* * *

######  CONNECTOR

| name | desctription |
| :-- | :-- |
| EDID | Extended Display Identification Data，标识显示器的参数信息，是一种 VESA 标准数据格式 |
| DPMS | Display Power Management Signaling，用于控制显示器的电源状态，如休眠唤醒。也是一种 VESA 标准 |
| link-status | 用于标识当前 connector 的连接状态，如 Good/Bad |
| CRTC_ID | 当前 connector 所连接的 CRTC object ID，与 PLANE 中 CRTC_ID 属性是同一个 property |

(optional)

| name | desctription |
| :-- | :-- |
| PATH | DisplayPort 专用的属性，主要用于 Multi-Stream Transport (MST) 功能，即多路显示应用场景 |
| TILE | 用于标识当前 connector 是否应用于多屏拼接场景，如平时随处可见的多屏拼接显示的广告大屏幕 |

* * *

###### Property Type

Property 的类型分为如下几种：

> *   enum
> *   bitmask
> *   range
> *   signed range
> *   object
> *   blob

以上类型中需要着重介绍的是 object 和 blob 类型，其它类型看名字就知道什么意思，所以就不做介绍了。

###### Object Property

Object 类型的 property，它的值用 _drm_mode_object_ ID 来表示。目前的 DRM 架构中仅用到 2 个 Object Property，它们分别是 _**"FB_ID"**_ 和 _**"CRTC_ID"**_ ，它们的 property 值分别表示 framebuffer object ID 和 crtc object ID。

###### Blob Property

Blob 类型的 property，它的值用 blob object ID 来表示。所谓 blob，说白了就是一个自定义长度的内存块，用来存放自定义的结构体数据。典型的 Blob Property，如 _**"MODE_ID"**_ ，它的值为 blob object ID，drm 驱动可以根据该 ID 找到对应的 _drm_property_blob_ 结构体，该结构体中存放着 modeinfo 的相关信息。

> 在 DRM 的 property type 中，还有 2 种特殊的 type，它们分别是 `IMMUTABLE TYPE` 和 `ATOMIC TYPE`。这两种 type 的特殊性在于，它们可以和上面任意一种 property 进行组合使用，用来修饰上面的 property。
> 
> *   IMMUTABLE TYPE：表示该 property 为只读，应用程序无法修改它的值，如 "IN_FORMATS"。
> *   ATOMIC TYPE：表示该 property 只有在 drm 应用程序（drm client）支持 ATOMIC 操作时才可见。

概念图，还是习惯性的上一张图吧：

![](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/zjj.sys.display.8.1.modetest/modetest.shot.property.png)

###### 总结

DRM 的 Property，其实有点类似于 kernel 中的 sysfs 属性节点。DRM 驱动将 kernel 层的重要参数通过 property 机制导出给上层，使得上层应用可以采用统一的接口形式来修改 property 的值，从而实现参数的传递，而无需新增额外的 IOCTL 接口。

通过本篇的学习，我们了解了 DRM Property 的基本概念，这有助于我们学习后面的 Atomic 编程。在下一篇文章中，我们将一起来学习如何使用 libdrm 的 Atomic 接口进行编程，敬请期待！

#### （六）、DRM应用程序进阶 （atomic-crtc）

在上一篇《DRM 应用程序进阶（Property）》中，我们学习了 Property 的基本概念及作用。在本篇中，我们将一起来学习如何操作这些 Property，即 libdrm Atomic 接口的用法。

**Atomic**

**为什么叫 “Atomic Commit”？**  
初学者第一次接触到 DRM 时，总会好奇当初开发者为什么要起 _Atomic_ 这个名字。Wiki 上关于该名词有较详细的解释，如果大家感兴趣可以通过本篇结尾的参考资料获取链接查看。我这里用白话简单概括就是：**本次 commit 操作，要么成功，要么保持原来的状态不变。** 即如果中途操作失败了，那些已经生效的配置需要恢复成之前的状态，就像没发生过 commit 操作似的，这就是 Atomic 的含义。

而用 _Commit_ 一词，是因为本次操作可能会修改到多个参数，等修改好这些参数后，再一次性发起操作请求，有点类似与填表后 “提交” 材料的意思。

**如何操作 property？**  
在上一篇我们了解了 Property 的基本组成结构，即`name`、`id` 和 `value`。因此操作 property 就变得非常简单，通过 _name_ 来获取 property，通过 _id_ 来操作 property，通过 _value_ 来修改 property 的值。而完成这些操作的应用接口，就是 libdrm 提供的 Atomic 接口。

> 需要记住一点，在 libdrm 中，所有的操作都是以 Object ID 来进行访问的，因此要操作 property，首先需要获取该 property 的 Object ID。

**伪代码：**

```
int main(void)
{
    ...
	drmSetClientCap(DRM_CLIENT_CAP_ATOMIC);

	drmModeObjectGetProperties(...);
	drmModeGetProperty(property_id)
	...
	drmModeAtomicAlloc();
	drmModeAtomicAddProperty(..., property_id, property_value);
	drmModeAtomicCommit(...);
	drmModeAtomicFree();
    ...
}
```

首先通过`drmModeGetProperty()`来获取 property 的相关信息，然后通过`drmModeAtomicAddProperty()`来修改 property 的值，最后通过`drmModeAtomicCommit()`来发起真正的修改请求。

**为什么要设置 _DRM_CLIENT_CAP_ATOMIC_ ？**  
在《DRM 应用程序进阶（Property）》中有介绍过，凡是被`DRM_MODE_PROP_ATOMIC`修饰过的 property，只有在 drm 应用程序支持 Atomic 操作时才可见，否则该 property 对应用程序不可见。因此通过设置`DRM_CLIENT_CAP_ATOMIC`这个 flag，来告知 DRM 驱动该应用程序支持 Atomic 操作。


基于之前的《最简单的 DRM 应用程序（plane-test）》的参考代码，我们使用 Atomic 接口来替代原来的`drmModeSetCrtc()`接口，从而通过差异对比来学些 Atomic 接口的操作。

**modeset-atomic-crtc.c**

``` c
#define _GNU_SOURCE
#include <errno.h>
#include <fcntl.h>
#include <stdbool.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>
#include <time.h>
#include <unistd.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

struct buffer_object {
	uint32_t width;
	uint32_t height;
	uint32_t pitch;
	uint32_t handle;
	uint32_t size;
	uint8_t *vaddr;
	uint32_t fb_id;
};

struct buffer_object buf;

static int modeset_create_fb(int fd, struct buffer_object *bo)
{
	struct drm_mode_create_dumb create = {};
 	struct drm_mode_map_dumb map = {};

	create.width = bo->width;
	create.height = bo->height;
	create.bpp = 32;
	drmIoctl(fd, DRM_IOCTL_MODE_CREATE_DUMB, &create);

	bo->pitch = create.pitch;
	bo->size = create.size;
	bo->handle = create.handle;
	drmModeAddFB(fd, bo->width, bo->height, 24, 32, bo->pitch,
			   bo->handle, &bo->fb_id);

	map.handle = create.handle;
	drmIoctl(fd, DRM_IOCTL_MODE_MAP_DUMB, &map);

	bo->vaddr = mmap(0, create.size, PROT_READ | PROT_WRITE,
			MAP_SHARED, fd, map.offset);

	memset(bo->vaddr, 0xff, bo->size);

	return 0;
}

static void modeset_destroy_fb(int fd, struct buffer_object *bo)
{
	struct drm_mode_destroy_dumb destroy = {};

	drmModeRmFB(fd, bo->fb_id);

	munmap(bo->vaddr, bo->size);

	destroy.handle = bo->handle;
	drmIoctl(fd, DRM_IOCTL_MODE_DESTROY_DUMB, &destroy);
}

static uint32_t get_property_id(int fd, drmModeObjectProperties *props,
				const char *name)
{
	drmModePropertyPtr property;
	uint32_t i, id = 0;

	/* find property according to the name */
	for (i = 0; i < props->count_props; i++) {
		property = drmModeGetProperty(fd, props->props[i]);
		if (!strcmp(property->name, name))
			id = property->prop_id;
		drmModeFreeProperty(property);

		if (id)
			break;
	}

	return id;
}

int main(int argc, char **argv)
{
	int fd;
	drmModeConnector *conn;
	drmModeRes *res;
	drmModePlaneRes *plane_res;
	drmModeObjectProperties *props;
	drmModeAtomicReq *req;
	uint32_t conn_id;
	uint32_t crtc_id;
	uint32_t plane_id;
	uint32_t blob_id;
	uint32_t property_crtc_id;
	uint32_t property_mode_id;
	uint32_t property_active;

	fd = open("/dev/dri/card0", O_RDWR | O_CLOEXEC);

	res = drmModeGetResources(fd);
	crtc_id = res->crtcs[0];
	conn_id = res->connectors[0];

	drmSetClientCap(fd, DRM_CLIENT_CAP_UNIVERSAL_PLANES, 1);
	plane_res = drmModeGetPlaneResources(fd);
	plane_id = plane_res->planes[0];

	conn = drmModeGetConnector(fd, conn_id);
	buf.width = conn->modes[0].hdisplay;
	buf.height = conn->modes[0].vdisplay;

	modeset_create_fb(fd, &buf);

	drmSetClientCap(fd, DRM_CLIENT_CAP_ATOMIC, 1);

	/* get connector properties */
	props = drmModeObjectGetProperties(fd, conn_id,	DRM_MODE_OBJECT_CONNECTOR);
	property_crtc_id = get_property_id(fd, props, "CRTC_ID");
	drmModeFreeObjectProperties(props);

	/* get crtc properties */
	props = drmModeObjectGetProperties(fd, crtc_id, DRM_MODE_OBJECT_CRTC);
	property_active = get_property_id(fd, props, "ACTIVE");
	property_mode_id = get_property_id(fd, props, "MODE_ID");
	drmModeFreeObjectProperties(props);

	/* create blob to store current mode, and retun the blob id */
	drmModeCreatePropertyBlob(fd, &conn->modes[0],
				sizeof(conn->modes[0]), &blob_id);

	/* start modeseting */
	req = drmModeAtomicAlloc();
	drmModeAtomicAddProperty(req, crtc_id, property_active, 1);
	drmModeAtomicAddProperty(req, crtc_id, property_mode_id, blob_id);
	drmModeAtomicAddProperty(req, conn_id, property_crtc_id, crtc_id);
	drmModeAtomicCommit(fd, req, DRM_MODE_ATOMIC_ALLOW_MODESET, NULL);
	drmModeAtomicFree(req);

	printf("drmModeAtomicCommit SetCrtc\n");
	getchar();

	drmModeSetPlane(fd, plane_id, crtc_id, buf.fb_id, 0,
			50, 50, 320, 320,
			0, 0, 320 << 16, 320 << 16);

	printf("drmModeSetPlane\n");
	getchar();

	modeset_destroy_fb(fd, &buf);

	drmModeFreeConnector(conn);
	drmModeFreePlaneResources(plane_res);
	drmModeFreeResources(res);

	close(fd);

	return 0;
}
```

通过上面的代码我们可以看出，原来的 _**drmModeSetCrtc(crtc_id, fb_id, conn_id, &mode)**_ 被下面这部分代码取代了：

``` c
req = drmModeAtomicAlloc();
	drmModeAtomicAddProperty(req, crtc_id, property_active, 1);
	drmModeAtomicAddProperty(req, crtc_id, property_mode_id, blob_id);
	drmModeAtomicAddProperty(req, conn_id, property_crtc_id, crtc_id);
	drmModeAtomicCommit(fd, req, DRM_MODE_ATOMIC_ALLOW_MODESET, NULL);
	drmModeAtomicFree(req);
```

虽然代码量增加了，但是应用程序的灵活性和可扩展性也增强了。

由于以上代码没有添加对 fb_id 的操作，因此它的作用只是初始化 CRTC/ENCODER/CONNECTOR 硬件，以及建立硬件链路的连接关系，并不会显示 framebuffer 的内容，即保持黑屏状态。framebuffer 的显示将由后面的 _drmModeSetPlane()_ 操作来完成。


以上代码运行的效果如下：  
![](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/zjj.sys.display.8.1.modetest/modetest.shot.atomic.crtc.jpg)
> 注意：程序运行之前，请确保没有其它应用或服务占用 / dev/dri/card0 节点，否则将出现 **Permission Denied** 错误。

描述：

1.  程序运行后，屏幕显示黑色，终端打印 “drmModeAtomicCommit” 信息，表明当前已经初始化好 CRTC/ENCODER/CONNECTOR 硬件；
2.  输入回车后，屏幕显示 framebuffer 的 crop 区域，同时终端打印 “drmModeSetPlane” 信息；
3.  再次输入回车，显示黑屏，程序退出。

#### （七）、DRM应用程序进阶 （atomic-plane）

在上一篇《DRM 应用程序进阶（atomic-crtc）》文章中，我们学习了如何通过 libdrm 的 atomic 接口实现 modeseting 的操作。本篇，我们将一起来学习如何通过 atomic 接口，来实现 `drmModeSetPlane()` 相同的操作。

###### Atomic Commit

plane update 的 atomic 操作如下：

``` c
drmModeAtomicAddProperty(req, plane_id, property_crtc_id, crtc_id);
drmModeAtomicAddProperty(req, plane_id, property_fb_id, fb_id);
drmModeAtomicAddProperty(req, plane_id, property_crtc_x, crtc_x);
drmModeAtomicAddProperty(req, plane_id, property_crtc_y, crtc_y);
drmModeAtomicAddProperty(req, plane_id, property_crtc_w, crtc_w);
drmModeAtomicAddProperty(req, plane_id, property_crtc_h, crtc_h);
drmModeAtomicAddProperty(req, plane_id, property_src_x, src_x);
drmModeAtomicAddProperty(req, plane_id, property_src_y, src_y);
drmModeAtomicAddProperty(req, plane_id, property_src_w, src_w << 16);
drmModeAtomicAddProperty(req, plane_id, property_src_h, src_h << 16);
drmModeAtomicCommit(fd, req, flags, NULL);
```

以上各项参数与 _drmModeSetPlane()_ 的参数一一对应：

``` c
drmModeSetPlane(fd, plane_id, crtc_id, fb_id, flags,
                crtc_x, crtc_y, crtc_w, crtc_h,
                src_x << 16, src_y << 16, src_w << 16, src_h << 16)
```

其中，参数 _flags_ 可以是如下值的组合：

*   `DRM_MODE_PAGE_FLIP_EVENT`: 请求底层驱动发送 PAGE_FLIP 事件，上层应用需要调用 _drmHandleEvent()_ 来接收并处理相应事件，详见《最简单的 DRM 应用程序 （page-flip）》
*   `DRM_MODE_ATOMIC_TEST_ONLY`: 仅用于试探本次 commit 操作是否能成功，不会操作真正的硬件寄存器。**不能和 _DRM_MODE_PAGE_FLIP_EVENT_ 同时使用**。
*   `DRM_MODE_ATOMIC_NONBLOCK`: 允许本次 commit 操作异步执行，即无需等待上一次 commit 操作彻底执行完成，就可以发起本次操作。**_drmModeAtomicCommit()_ 默认以 BLOCK（同步）方式执行**。
*   `DRM_MODE_ATOMIC_ALLOW_MODESET`: 告诉底层驱动，本次 commit 操作修改到了 modeseting 相关的参数，需要执行一次 full modeset 动作。

当然，_flags_ 参数为 0 也是可以的。

_drmModeAtomicCommit()_ 与 _drmModeSetPlane()_ 对比，具有如下优势：

| drmModeAtomicCommit | drmModeSetPlane |
| :-- | :-- |
| 支持异步调用 | 只能同步调用 |
| 一次调用可以更新多个 plane | 每次调用只能更新 1 个 plane |
| 支持 test 功能，提前预知调用结果 | 不支持 |

###### Property Commit

如果你从事过 Android 底层开发，你会发现在 Android _external/libdrm_ 的某些 [release 分支](https://github.com/zhoujinjianok/libdrm/blob/oreo-m4-s2-release/xf86drmMode.c)上，会多出 `drmModePropertySetAdd()`、`drmModePropertySetCommit()` 等这类函数接口，而在 libdrm 的社区主线仓库 [mesa/drm](https://cgit.freedesktop.org/mesa/drm) 中，其实是找不到这样的接口的。该接口最初由 Ville Syrjälä 和 Sean Paul 于 2012 年提交到 libdrm 的一个临时分支上，后来又被 cherry-pick 到 Android 的 libdrm 仓库中，[点此链接](https://github.com/atseanpaul/libdrm/commit/c2df6962ee17181f73799fe4b81cdcbbc816c31e#diff-c2cd4bcd4ad8113f21389f4caa348e0a)查看提交记录。

**伪代码**

```
pset = drmModePropertySetAlloc();
drmModePropertySetAdd(pset, object_id, property_id, property_value);
drmModePropertySetCommit(fd, flags, data, pset);
drmModePropertySetFree(pset);
```

由于这些接口的用法同主线中的 Atomic 接口大同小异，因此不做详细描述。

参考代码：基于上一篇 [modeset-atomic-crtc]程序，我们使用 atomic 接口替换 _drmModeSetPlane()_ 函数，具体如下：

**modeset-atomic-plane.c**

``` c
#define _GNU_SOURCE
#include <errno.h>
#include <fcntl.h>
#include <stdbool.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>
#include <time.h>
#include <unistd.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

struct buffer_object {
	uint32_t width;
	uint32_t height;
	uint32_t pitch;
	uint32_t handle;
	uint32_t size;
	uint8_t *vaddr;
	uint32_t fb_id;
};

struct buffer_object buf;

static int modeset_create_fb(int fd, struct buffer_object *bo)
{
	struct drm_mode_create_dumb create = {};
 	struct drm_mode_map_dumb map = {};

	create.width = bo->width;
	create.height = bo->height;
	create.bpp = 32;
	drmIoctl(fd, DRM_IOCTL_MODE_CREATE_DUMB, &create);

	bo->pitch = create.pitch;
	bo->size = create.size;
	bo->handle = create.handle;
	drmModeAddFB(fd, bo->width, bo->height, 24, 32, bo->pitch,
			   bo->handle, &bo->fb_id);

	map.handle = create.handle;
	drmIoctl(fd, DRM_IOCTL_MODE_MAP_DUMB, &map);

	bo->vaddr = mmap(0, create.size, PROT_READ | PROT_WRITE,
			MAP_SHARED, fd, map.offset);

	memset(bo->vaddr, 0xff, bo->size);

	return 0;
}

static void modeset_destroy_fb(int fd, struct buffer_object *bo)
{
	struct drm_mode_destroy_dumb destroy = {};

	drmModeRmFB(fd, bo->fb_id);

	munmap(bo->vaddr, bo->size);

	destroy.handle = bo->handle;
	drmIoctl(fd, DRM_IOCTL_MODE_DESTROY_DUMB, &destroy);
}

static uint32_t get_property_id(int fd, drmModeObjectProperties *props,
				const char *name)
{
	drmModePropertyPtr property;
	uint32_t i, id = 0;

	for (i = 0; i < props->count_props; i++) {
		property = drmModeGetProperty(fd, props->props[i]);
		if (!strcmp(property->name, name))
			id = property->prop_id;
		drmModeFreeProperty(property);

		if (id)
			break;
	}

	return id;
}

int main(int argc, char **argv)
{
	int fd;
	drmModeConnector *conn;
	drmModeRes *res;
	drmModePlaneRes *plane_res;
	drmModeObjectProperties *props;
	drmModeAtomicReq *req;
	uint32_t conn_id;
	uint32_t crtc_id;
	uint32_t plane_id;
	uint32_t blob_id;
	uint32_t property_crtc_id;
	uint32_t property_mode_id;
	uint32_t property_active;
	uint32_t property_fb_id;
	uint32_t property_crtc_x;
	uint32_t property_crtc_y;
	uint32_t property_crtc_w;
	uint32_t property_crtc_h;
	uint32_t property_src_x;
	uint32_t property_src_y;
	uint32_t property_src_w;
	uint32_t property_src_h;

	fd = open("/dev/dri/card0", O_RDWR | O_CLOEXEC);

	res = drmModeGetResources(fd);
	crtc_id = res->crtcs[0];
	conn_id = res->connectors[0];

	drmSetClientCap(fd, DRM_CLIENT_CAP_UNIVERSAL_PLANES, 1);
	plane_res = drmModeGetPlaneResources(fd);
	plane_id = plane_res->planes[0];

	conn = drmModeGetConnector(fd, conn_id);
	buf.width = conn->modes[0].hdisplay;
	buf.height = conn->modes[0].vdisplay;

	modeset_create_fb(fd, &buf);

	drmSetClientCap(fd, DRM_CLIENT_CAP_ATOMIC, 1);

	props = drmModeObjectGetProperties(fd, conn_id,	DRM_MODE_OBJECT_CONNECTOR);
	property_crtc_id = get_property_id(fd, props, "CRTC_ID");
	drmModeFreeObjectProperties(props);

	props = drmModeObjectGetProperties(fd, crtc_id, DRM_MODE_OBJECT_CRTC);
	property_active = get_property_id(fd, props, "ACTIVE");
	property_mode_id = get_property_id(fd, props, "MODE_ID");
	drmModeFreeObjectProperties(props);

	drmModeCreatePropertyBlob(fd, &conn->modes[0],
				sizeof(conn->modes[0]), &blob_id);

	req = drmModeAtomicAlloc();
	drmModeAtomicAddProperty(req, crtc_id, property_active, 1);
	drmModeAtomicAddProperty(req, crtc_id, property_mode_id, blob_id);
	drmModeAtomicAddProperty(req, conn_id, property_crtc_id, crtc_id);
	drmModeAtomicCommit(fd, req, DRM_MODE_ATOMIC_ALLOW_MODESET, NULL);
	drmModeAtomicFree(req);

	printf("drmModeAtomicCommit SetCrtc\n");
	getchar();

    /* get plane properties */
	props = drmModeObjectGetProperties(fd, plane_id, DRM_MODE_OBJECT_PLANE);
	property_crtc_id = get_property_id(fd, props, "CRTC_ID");
	property_fb_id = get_property_id(fd, props, "FB_ID");
	property_crtc_x = get_property_id(fd, props, "CRTC_X");
	property_crtc_y = get_property_id(fd, props, "CRTC_Y");
	property_crtc_w = get_property_id(fd, props, "CRTC_W");
	property_crtc_h = get_property_id(fd, props, "CRTC_H");
	property_src_x = get_property_id(fd, props, "SRC_X");
	property_src_y = get_property_id(fd, props, "SRC_Y");
	property_src_w = get_property_id(fd, props, "SRC_W");
	property_src_h = get_property_id(fd, props, "SRC_H");
	drmModeFreeObjectProperties(props);

    /* atomic plane update */
	req = drmModeAtomicAlloc();
	drmModeAtomicAddProperty(req, plane_id, property_crtc_id, crtc_id);
	drmModeAtomicAddProperty(req, plane_id, property_fb_id, buf.fb_id);
	drmModeAtomicAddProperty(req, plane_id, property_crtc_x, 50);
	drmModeAtomicAddProperty(req, plane_id, property_crtc_y, 50);
	drmModeAtomicAddProperty(req, plane_id, property_crtc_w, 320);
	drmModeAtomicAddProperty(req, plane_id, property_crtc_h, 320);
	drmModeAtomicAddProperty(req, plane_id, property_src_x, 0);
	drmModeAtomicAddProperty(req, plane_id, property_src_y, 0);
	drmModeAtomicAddProperty(req, plane_id, property_src_w, 320 << 16);
	drmModeAtomicAddProperty(req, plane_id, property_src_h, 320 << 16);
	drmModeAtomicCommit(fd, req, 0, NULL);
	drmModeAtomicFree(req);

	printf("drmModeAtomicCommit SetPlane\n");
	getchar();

	modeset_destroy_fb(fd, &buf);

	drmModeFreeConnector(conn);
	drmModeFreePlaneResources(plane_res);
	drmModeFreeResources(res);

	close(fd);

	return 0;
}
```

提示：

> 1.  上面的两次 _drmModeAtomicCommit()_ 操作可以合并成一次；
> 2.  无需频繁调用 _drmModeAtomicAlloc()_ 、_drmModeAtomicFree()_ ，可以在第一次 commit 之前 Alloc，在最后一次 commit 之后 free，也没问题；
> 3.  plane update 操作时，可以只 add 发生变化的 property，其它未发生变化的 properties 即使没有被 add，在 commit 时底层驱动仍然会取上一次的值来配置硬件寄存器。

运行结果，以上代码运行的效果如下：  
![](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/zjj.sys.display.8.1.modetest/modetest.shot.atomic.plane.jpg)

> 注意：程序运行之前，请确保没有其它应用或服务占用 / dev/dri/card0 节点，否则将出现 **Permission Denied** 错误。

**描述：**

1.  程序运行后，屏幕显示黑色，终端打印 “drmModeAtomicCommit SetCrtc” 信息，表明当前已经初始化好 CRTC/ENCODER/CONNECTOR 硬件；
2.  输入回车后，屏幕显示 framebuffer 的 crop 区域，同时终端打印 “drmModeAtomicCommit SetPlane”；
3.  再次输入回车，显示黑屏，程序退出

**源码下载：**[modeset-plane-test](https://github.com/zhoujinjianok/sample-code)
#### （八）、ModeTest显示流程图Flow

![](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/zjj.sys.display.8.1.modetest/FireFly-Rk3399-DRM-KMS-ModeTest.png)
#### （九）、ModeTest显示图像

>//modetest
>adb shell
>stop
>               //connector@crts
>modetest -M rockchip -s 88@61:768x1024 -v

##### 8.1、显示图像（模式为UTIL_PATTERN_SMPTE时）：
``` c
I:\RK3399_Android8.1_MIPI\external\libdrm\tests\modetest\modetest.c
static void set_mode(struct device *dev, struct pipe_arg *pipes, unsigned int count)
{
......
	bo = bo_create(dev->fd, pipes[0].fourcc, dev->mode.width,
		       dev->mode.height, handles, pitches, offsets,
		       UTIL_PATTERN_SMPTE);
}
```
![](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/zjj.sys.display.8.1.modetest/modetest.shot.pattern_smpte.jpg)
##### 8.2、显示图像（模式为UTIL_PATTERN_TILES时）：
``` c
I:\RK3399_Android8.1_MIPI\external\libdrm\tests\modetest\modetest.c
static void set_mode(struct device *dev, struct pipe_arg *pipes, unsigned int count)
{
......
	bo = bo_create(dev->fd, pipes[0].fourcc, dev->mode.width,
		       dev->mode.height, handles, pitches, offsets,
		       UTIL_PATTERN_TILES);
}
```

![](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/zjj.sys.display.8.1.modetest/modetest.shot.pattern_tiles.jpg)

#### （十）、参考资料(特别感谢)：
[（1）【DRM（Direct Rendering Manager）】](https://docs.nvidia.com/drive/nvvib_docs/NVIDIA%20DRIVE%20Linux%20SDK%20Development%20Guide/baggage/group__direct__rendering__manager.html) 
[（2）【DRM（Direct Rendering Manager）学习简介】](https://blog.csdn.net/hexiaolong2009/article/details/83720940) 
[（3）【LCD显示异常分析——撕裂(tear effect)】](https://blog.csdn.net/hexiaolong2009/article/details/79319512)