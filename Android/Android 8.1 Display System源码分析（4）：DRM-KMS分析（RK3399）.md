---
title: Android 8.1 Display System源码分析（4）：DRM/KMS分析（RK3399）
cover: https://raw.githubusercontent.com/zhoujinjian777/PicGo/master/personal.website/post.cover.pictures.00013.jpg
categories: 
  - Display
tags:
  - Android
  - Linux
  - Graphics
toc: true
abbrlink: 20200705
date: 2020-07-05 09:25:00
---

注：文章都是通过阅读 Android  && Linux 平台源码、各位前辈总结的资料、加上自己的思考分析总结出来的，其中难免有理不对的地方，欢迎大家批评指正。**文章为个人学习、研究、欣赏之用。图文内容、源码整理自互联网，如有侵权，请联系删除（◕‿◕）**，转载请注明出处（ ©Android @Linux @Rockchip版权所有），谢谢(๑乛◡乛๑) 、（ ͡° ͜ʖ ͡°）、（ಡωಡ）！！！。
（==**文章基于 Kernel-4.4**==）&&（==**文章基于 Android-8.1**==）
[【开发板 - Firefly-RK3399 7.85寸1024x768显示屏模组（MIPI接口）】](
http://wiki.t-firefly.com/zh_CN/Firefly-RK3399/compile_android8.1_firmware.html#)

正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢！！！

[（1）【DRM（Direct Rendering Manager）学习简介】](
https://blog.csdn.net/hexiaolong2009/article/details/83720940) 

[（2）【The DRM/KMS subsystem from a newbie’s point of view】](
https://events.static.linuxfound.org/sites/events/files/slides/brezillon-drm-kms.pdf) 

[（3）【Anatomy of an Atomic KMS Driver】](
https://elinux.orghttps://raw.githubusercontent.com/zhoujinjian777/PicGo/master/4/45/Atomic_kms_driver_pinchart.pdf) 

[（4）【Why and How to use KMS】](
https://events.static.linuxfound.org/sites/events/files/lcjpcojp13_pinchart.pdf) 

--------------------------------------------------------------------------------
==源码（部分）==

>DRM

- I:\RK3399_Android8.1_MIPI\kernel\drivers\gpu\drm\rockchip\*
- I:\RK3399_Android8.1_MIPI\kernel\drivers\gpu\drm\*
- I:\RK3399_Android8.1_MIPI\kernel\drivers\phy\rockchip\*



--------------------------------------------------------------------------------

#### (一)、DRM/KMS简介 

#### （1）、DRM（Direct Rendering Manager）简介

DRM 是 Linux 目前主流的图形显示框架，相比 FB 架构，DRM 更能适应当前日益更新的显示硬件。比如 FB 原生不支持多层合成，不支持 VSYNC，不支持 DMA-BUF，不支持异步更新，不支持 fence 机制等等，而这些功能 DRM 原生都支持。同时 DRM 可以统一管理 GPU 和 Display 驱动，使得软件架构更为统一，方便管理和维护。

DRM 从模块上划分，可以简单分为 3 部分：**`libdrm`**、**`KMS`**、**`GEM`**

![](https://raw.githubusercontent.com/zhoujinjian777/PicGo/master/zjj.sys.display.8.1.drmkms/drm_user_kernel_space.svg)

##### 1.1、libdrm

对底层接口进行封装，向上层提供通用的 API 接口，主要是对各种 IOCTL 接口进行封装。

##### 1.2、GEM

Graphic Execution Manager，主要负责显示 buffer 的分配和释放，也是 GPU 唯一用到 DRM 的地方。

##### 1.3、基本元素

----

DRM 框架涉及到的元素很多，大致如下：  
KMS：**`CRTC`**，**`ENCODER`**，**`CONNECTOR`**，**`PLANE`**，**`FB`**，**`VBLANK`**，**`property`**  
GEM：**`DUMB`**、**`PRIME`**、**`fence`**

![](https://raw.githubusercontent.com/zhoujinjian777/PicGo/master/zjj.sys.display.8.1.drmkms/drm_architecture.jpg)


| 元素 | 说明 |
| --- | --- |
| CRTC | 对显示 buffer 进行扫描，并产生时序信号的硬件模块，通常指 Display Controller |
| ENCODER | 负责将 CRTC 输出的 timing 时序转换成外部设备所需要的信号的模块，如 HDMI 转换器或 DSI Controller |
| CONNECTOR | 连接物理显示设备的连接器，如 HDMI、DisplayPort、DSI 总线，通常和 Encoder 驱动绑定在一起 |
| PLANE | 硬件图层，有的 Display 硬件支持多层合成显示，但所有的 Display Controller 至少要有 1 个 plane |
| FB | Framebuffer，单个图层的显示内容，唯一一个和硬件无关的基本元素 |
| VBLANK | 软件和硬件的同步机制，RGB 时序中的垂直消影区，软件通常使用硬件 VSYNC 来实现 |
| property | 任何你想设置的参数，都可以做成 property，是 DRM 驱动中最灵活、最方便的 Mode setting 机制 |
| DUMB | 只支持连续物理内存，基于 kernel 中通用 CMA API 实现，多用于小分辨率简单场景 |
| PRIME | 连续、非连续物理内存都支持，基于 DMA-BUF 机制，可以实现 buffer 共享，多用于大内存复杂场景 |
| fence | buffer 同步机制，基于内核 dma_fence 机制实现，用于防止显示内容出现异步问题 |


#### （2）、KMS

Kernel Mode Setting，所谓 Mode setting，其实说白了就两件事：`更新画面`和`设置显示参数`。  
**更新画面**：显示 buffer 的切换，多图层的合成方式，以及每个图层的显示位置。  
**设置显示参数**：包括分辨率、刷新率、电源状态（休眠唤醒）等。

#### (二)、DRM/KMS驱动分析

首先看看RK3399 DRM Kernel启动框图：
![](https://raw.githubusercontent.com/zhoujinjian777/PicGo/master/zjj.sys.display.8.1.drmkms/rockchip_drm_kernel_boot_up.png)
首先看看RK3399DRM Kernel启动UML流程图：
![](https://raw.githubusercontent.com/zhoujinjian777/PicGo/master/zjj.sys.display.8.1.drmkms/FireFly-Rk3399-DRM-KMS-Kernel-Boot-Up.png)


#### （1）、Drm device创建注册
``` c
I:\RK3399_Android8.1_MIPI\kernel\drivers\gpu\drm\rockchip\rockchip_drm_drv.c
static int rockchip_drm_bind(struct device *dev)
{
	struct drm_device *drm_dev;
	struct rockchip_drm_private *private;
	int ret;
	struct device_node *np = dev->of_node;
	struct device_node *parent_np;
	struct drm_crtc *crtc;
    printk("zjj.sye.display rockchip_drm_drv.c rockchip_drm_bind \n");
    //drm_dev_alloc 分配
	drm_dev = drm_dev_alloc(&rockchip_drm_driver, dev);

	ret = drm_dev_set_unique(drm_dev, "%s", dev_name(dev));

	dev_set_drvdata(dev, drm_dev);

	private = devm_kzalloc(drm_dev->dev, sizeof(*private), GFP_KERNEL);

	mutex_init(&private->commit_lock);
	INIT_WORK(&private->commit_work, rockchip_drm_atomic_work);

	drm_dev->dev_private = private;

	private->dmc_support = false;
	private->devfreq = devfreq_get_devfreq_by_phandle(dev, 0);

	private->hdmi_pll.pll = devm_clk_get(dev, "hdmi-tmds-pll");

	private->default_pll.pll = devm_clk_get(dev, "default-vop-pll");

#ifdef CONFIG_DRM_DMA_SYNC
	private->cpu_fence_context = fence_context_alloc(1);
	atomic_set(&private->cpu_fence_seqno, 0);
#endif

	ret = rockchip_drm_init_iommu(drm_dev);

	drm_mode_config_init(drm_dev);

	rockchip_drm_mode_config_init(drm_dev);
	rockchip_drm_create_properties(drm_dev);

	/* Try to bind all sub drivers. */
	ret = component_bind_all(dev, drm_dev);

	rockchip_attach_connector_property(drm_dev);

	ret = drm_vblank_init(drm_dev, drm_dev->mode_config.num_crtc);

	drm_mode_config_reset(drm_dev);

	rockchip_drm_set_property_default(drm_dev);
	drm_dev->irq_enabled = true;
	drm_kms_helper_poll_init(drm_dev);
	drm_dev->vblank_disable_allowed = true;

	rockchip_gem_pool_init(drm_dev);
#ifndef MODULE
	show_loader_logo(drm_dev);
#endif
	ret = of_reserved_mem_device_init(drm_dev->dev);

	ret = rockchip_drm_fbdev_init(drm_dev);

	drm_for_each_crtc(crtc, drm_dev) {
		struct drm_fb_helper *helper = private->fbdev_helper;
		struct rockchip_crtc_state *s = NULL;

		s = to_rockchip_crtc_state(crtc->state);
		if (is_support_hotplug(s->output_type)) {
			s->crtc_primary_fb = crtc->primary->fb;
			crtc->primary->fb = helper->fb;
			drm_framebuffer_reference(helper->fb);
		}
	}
	drm_dev->mode_config.allow_fb_modifiers = true;
    //drm_dev注册
	ret = drm_dev_register(drm_dev, 0);
 }
```


#### （2）、Drm driver信息

``` c
I:\RK3399_Android8.1_MIPI\kernel\drivers\gpu\drm\rockchip\rockchip_drm_drv.c
static struct drm_driver rockchip_drm_driver = {
	.driver_features	= DRIVER_MODESET | DRIVER_GEM |
				  DRIVER_PRIME | DRIVER_ATOMIC |
				  DRIVER_RENDER,
	.preclose		= rockchip_drm_preclose,
	.lastclose		= rockchip_drm_lastclose,
	.get_vblank_counter	= drm_vblank_no_hw_counter,
	.open			= rockchip_drm_open,
	.postclose		= rockchip_drm_postclose,
	.enable_vblank		= rockchip_drm_crtc_enable_vblank,
	.disable_vblank		= rockchip_drm_crtc_disable_vblank,
	.gem_vm_ops		= &rockchip_drm_vm_ops,
	.gem_free_object	= rockchip_gem_free_object,
	.dumb_create		= rockchip_gem_dumb_create,
	.dumb_map_offset	= rockchip_gem_dumb_map_offset,
	.dumb_destroy		= drm_gem_dumb_destroy,
	.prime_handle_to_fd	= drm_gem_prime_handle_to_fd,
	.prime_fd_to_handle	= drm_gem_prime_fd_to_handle,
	.gem_prime_import	= drm_gem_prime_import,
	.gem_prime_export	= drm_gem_prime_export,
	.gem_prime_get_sg_table	= rockchip_gem_prime_get_sg_table,
	.gem_prime_import_sg_table	= rockchip_gem_prime_import_sg_table,
	.gem_prime_vmap		= rockchip_gem_prime_vmap,
	.gem_prime_vunmap	= rockchip_gem_prime_vunmap,
	.gem_prime_mmap		= rockchip_gem_mmap_buf,
	.gem_prime_begin_cpu_access = rockchip_gem_prime_begin_cpu_access,
	.gem_prime_end_cpu_access = rockchip_gem_prime_end_cpu_access,
#ifdef CONFIG_DEBUG_FS
	.debugfs_init		= rockchip_drm_debugfs_init,
	.debugfs_cleanup	= rockchip_drm_debugfs_cleanup,
#endif
	.ioctls			= rockchip_ioctls,
	.num_ioctls		= ARRAY_SIZE(rockchip_ioctls),
	.fops			= &rockchip_drm_driver_fops,
	.name	= DRIVER_NAME,
	.desc	= DRIVER_DESC,
	.date	= DRIVER_DATE,
	.major	= DRIVER_MAJOR,
	.minor	= DRIVER_MINOR,
};

```

#### （3）、Drm driver file operations

``` c
I:\RK3399_Android8.1_MIPI\kernel\drivers\gpu\drm\rockchip\rockchip_drm_drv.c
static const struct file_operations rockchip_drm_driver_fops = {
	.owner = THIS_MODULE,
	.open = drm_open,
	.mmap = rockchip_gem_mmap,
	.poll = drm_poll,
	.read = drm_read,
	.unlocked_ioctl = drm_ioctl,
#ifdef CONFIG_COMPAT
	.compat_ioctl = drm_compat_ioctl,
#endif
	.release = drm_release,
};

```

#### （4）、Mode Config

``` c
I:\RK3399_Android8.1_MIPI\kernel\drivers\gpu\drm\rockchip\rockchip_drm_drv.c
static int rockchip_drm_bind(struct device *dev)
{
drm_mode_config_init(drm_dev);
rockchip_drm_mode_config_init(drm_dev);
}

I:\RK3399_Android8.1_MIPI\kernel\arch\arm64\boot\dts\rockchip\rk3399-firefly-mipi.dts

void rockchip_drm_mode_config_init(struct drm_device *dev)
{
	dev->mode_config.min_width = 0;
	dev->mode_config.min_height = 0;
	dev->mode_config.max_width = 8192;
	dev->mode_config.max_height = 8192;

	dev->mode_config.funcs = &rockchip_drm_mode_config_funcs;
}

static const struct drm_mode_config_funcs rockchip_drm_mode_config_funcs = {
	.fb_create = rockchip_user_fb_create,
	.output_poll_changed = rockchip_drm_output_poll_changed,
	.atomic_check = drm_atomic_helper_check,
	.atomic_commit = rockchip_drm_atomic_commit,
};

```


#### （5）、GEM

``` c
I:\RK3399_Android8.1_MIPI\kernel\drivers\gpu\drm\rockchip\rockchip_drm_drv.c
const struct vm_operations_struct rockchip_drm_vm_ops = {
	.open = drm_gem_vm_open,
	.close = drm_gem_vm_close,
};
```
#### （6）、GEM – Dumb 
``` c
I:\RK3399_Android8.1_MIPI\kernel\drivers\gpu\drm\rockchip\rockchip_drm_drv.c

static struct drm_driver rockchip_drm_driver = {
	.driver_features	= DRIVER_MODESET | DRIVER_GEM |
				  DRIVER_PRIME | DRIVER_ATOMIC |
				  DRIVER_RENDER,
	......
	.dumb_create		= rockchip_gem_dumb_create,
	.dumb_map_offset	= rockchip_gem_dumb_map_offset,
	.dumb_destroy		= drm_gem_dumb_destroy,
	......
};
```
#### （7）、GEM – PRIME


``` c
I:\RK3399_Android8.1_MIPI\kernel\drivers\gpu\drm\rockchip\rockchip_drm_drv.c

static struct drm_driver rockchip_drm_driver = {
	.driver_features	= DRIVER_MODESET | DRIVER_GEM |
				  DRIVER_PRIME | DRIVER_ATOMIC |
				  DRIVER_RENDER,
	......
	.prime_handle_to_fd	= drm_gem_prime_handle_to_fd,
	.prime_fd_to_handle	= drm_gem_prime_fd_to_handle,
	.gem_prime_import	= drm_gem_prime_import,
	.gem_prime_export	= drm_gem_prime_export,
	.gem_prime_get_sg_table	= rockchip_gem_prime_get_sg_table,
	.gem_prime_import_sg_table	= rockchip_gem_prime_import_sg_table,
	.gem_prime_vmap		= rockchip_gem_prime_vmap,
	.gem_prime_vunmap	= rockchip_gem_prime_vunmap,
	.gem_prime_mmap		= rockchip_gem_mmap_buf,
	.gem_prime_begin_cpu_access = rockchip_gem_prime_begin_cpu_access,
	.gem_prime_end_cpu_access = rockchip_gem_prime_end_cpu_access,
	......
};
```
#### （8）、Frame Buffer
``` c
I:\RK3399_Android8.1_MIPI\kernel\drivers\gpu\drm\rockchip\rockchip_drm_fb.c

static const struct drm_mode_config_funcs rockchip_drm_mode_config_funcs = {
	.fb_create = rockchip_user_fb_create,
	.output_poll_changed = rockchip_drm_output_poll_changed,
	.atomic_check = drm_atomic_helper_check,
	.atomic_commit = rockchip_drm_atomic_commit,
};

static struct drm_framebuffer *
rockchip_user_fb_create(struct drm_device *dev, struct drm_file *file_priv,
			struct drm_mode_fb_cmd2 *mode_cmd)
{
	struct drm_framebuffer *fb;
	struct drm_gem_object *objs[ROCKCHIP_MAX_FB_BUFFER];
	struct drm_gem_object *obj;
	unsigned int hsub;
	unsigned int vsub;
	int num_planes;
	int ret;
	int i;

	hsub = drm_format_horz_chroma_subsampling(mode_cmd->pixel_format);
	vsub = drm_format_vert_chroma_subsampling(mode_cmd->pixel_format);
	num_planes = min(drm_format_num_planes(mode_cmd->pixel_format),
			 ROCKCHIP_MAX_FB_BUFFER);

	for (i = 0; i < num_planes; i++) {
		unsigned int width = mode_cmd->width / (i ? hsub : 1);
		unsigned int height = mode_cmd->height / (i ? vsub : 1);
		unsigned int min_size;
		unsigned int bpp =
			drm_format_plane_bpp(mode_cmd->pixel_format, i);

		obj = drm_gem_object_lookup(dev, file_priv,
					    mode_cmd->handles[i]);
		if (!obj) {
			dev_err(dev->dev, "Failed to lookup GEM object\n");
			ret = -ENXIO;
			goto err_gem_object_unreference;
		}

		min_size = (height - 1) * mode_cmd->pitches[i] +
			mode_cmd->offsets[i] + roundup(width * bpp, 8) / 8;
		if (obj->size < min_size) {
			DRM_ERROR("Invalid Gem size on plane[%d]: %zd < %d\n",
				  i, obj->size, min_size);
			drm_gem_object_unreference_unlocked(obj);
			ret = -EINVAL;
			goto err_gem_object_unreference;
		}
		objs[i] = obj;
	}

	fb = rockchip_fb_alloc(dev, mode_cmd, objs, NULL, i);
	......

	return fb;
}
```
#### （9）、Initialization – CRTC

``` c
static int vop_bind(struct device *dev, struct device *master, void *data)
{
	struct platform_device *pdev = to_platform_device(dev);
	const struct vop_data *vop_data;
	struct drm_device *drm_dev = data;
	struct vop *vop;
	struct resource *res;
	......

	ret = vop_create_crtc(vop);
	......
	of_rockchip_drm_sub_backlight_register(dev, &vop->crtc,
					       &rockchip_sub_backlight_ops);

	return 0;
}


static int vop_create_crtc(struct vop *vop)
{
	struct device *dev = vop->dev;
	const struct vop_data *vop_data = vop->data;
	struct drm_device *drm_dev = vop->drm_dev;
	struct rockchip_drm_private *private = drm_dev->dev_private;
	struct drm_plane *primary = NULL, *cursor = NULL, *plane, *tmp;
	struct drm_crtc *crtc = &vop->crtc;
	struct device_node *port;
	
	/*
	 * Create drm_plane for primary and cursor planes first, since we need
	 * to pass them to drm_crtc_init_with_planes, which sets the
	 * "possible_crtcs" to the newly initialized crtc.
	 */
	for (i = 0; i < vop->num_wins; i++) {
		struct vop_win *win = &vop->win[i];
       ......

		ret = vop_plane_init(vop, win, 0);
		......

	}

	ret = drm_crtc_init_with_planes(drm_dev, crtc, primary, cursor,
					&vop_crtc_funcs, NULL);
	.......
	drm_crtc_helper_add(crtc, &vop_crtc_helper_funcs);

	......

	init_completion(&vop->dsp_hold_completion);
	init_completion(&vop->line_flag_completion);
	crtc->port = port;
	rockchip_register_crtc_funcs(crtc, &private_crtc_funcs);
    .......
}

```
#### （10）、Initialization – Encoder

``` c
I:\RK3399_Android8.1_MIPI\kernel\drivers\gpu\drm\rockchip\dw-mipi-dsi.c

static int dw_mipi_dsi_register(struct drm_device *drm,
				      struct dw_mipi_dsi *dsi)
{
	struct drm_encoder *encoder = &dsi->encoder;
	struct drm_connector *connector = &dsi->connector;
	struct device *dev = dsi->dev;
	int ret;
   printk("zjj.sye.display dw_mipi_dsi.c dw_mipi_dsi_register\n");
	encoder->possible_crtcs = drm_of_find_possible_crtcs(drm,
							     dev->of_node);
	/*
	 * If we failed to find the CRTC(s) which this encoder is
	 * supposed to be connected to, it's because the CRTC has
	 * not been registered yet.  Defer probing, and hope that
	 * the required CRTC is added later.
	 */
	if (encoder->possible_crtcs == 0)
		return -EPROBE_DEFER;

	drm_encoder_helper_add(&dsi->encoder,
			       &dw_mipi_dsi_encoder_helper_funcs);
	ret = drm_encoder_init(drm, &dsi->encoder, &dw_mipi_dsi_encoder_funcs,
			 DRM_MODE_ENCODER_DSI, NULL);
	......
	return 0;
}
```

#### （11）、Initialization – Connector
``` c
I:\RK3399_Android8.1_MIPI\kernel\drivers\gpu\drm\rockchip\dw-mipi-dsi.c

static int dw_mipi_dsi_register(struct drm_device *drm,
				      struct dw_mipi_dsi *dsi)
{
	struct drm_encoder *encoder = &dsi->encoder;
	struct drm_connector *connector = &dsi->connector;
	struct device *dev = dsi->dev;
	int ret;

	......

	/* If there's a bridge, attach to it and let it create the connector. */
	if (dsi->bridge) {
		......
	/* Otherwise create our own connector and attach to a panel */
	} else {
		dsi->connector.port = dev->of_node;
		ret = drm_connector_init(drm, &dsi->connector,
					 &dw_mipi_dsi_atomic_connector_funcs,
					 DRM_MODE_CONNECTOR_DSI);
		......
		drm_connector_helper_add(connector,
					 &dw_mipi_dsi_connector_helper_funcs);
		drm_mode_connector_attach_encoder(connector, encoder);

		ret = drm_panel_attach(dsi->panel, &dsi->connector);
		......
	}

	return 0;
}
```

#### （12）、Initialization – Plane

``` c
I:\RK3399_Android8.1_MIPI\kernel\drivers\gpu\drm\panel\panel-simple.c

static int panel_simple_probe(struct device *dev, const struct panel_desc *desc)
{
	struct device_node *backlight, *ddc;
	struct panel_simple *panel;
	struct panel_desc *of_desc;
	const char *cmd_type;
	u32 val;
	......
	panel = devm_kzalloc(dev, sizeof(*panel), GFP_KERNEL);
	......

	drm_panel_init(&panel->base);
	panel->base.dev = dev;
	panel->base.funcs = &panel_simple_funcs;

	err = drm_panel_add(&panel->base);
	if (err < 0)
		goto free_ddc;

	dev_set_drvdata(dev, panel);

	return 0;
}

```

#### （13）、Modes Discovery


``` c
I:\RK3399_Android8.1_MIPI\kernel\drivers\gpu\drm\rockchip\dw-mipi-dsi.c

static const struct drm_connector_funcs dw_mipi_dsi_atomic_connector_funcs = {
	.dpms = drm_atomic_helper_connector_dpms,
	.fill_modes = drm_helper_probe_single_connector_modes,
	.detect = dw_mipi_dsi_detect,
	.destroy = dw_mipi_dsi_drm_connector_destroy,
	.reset = drm_atomic_helper_connector_reset,
	.atomic_duplicate_state = drm_atomic_helper_connector_duplicate_state,
	.atomic_destroy_state = drm_atomic_helper_connector_destroy_state,
};

static const struct drm_connector_helper_funcs
dw_mipi_dsi_connector_helper_funcs = {
	.loader_protect = dw_mipi_loader_protect,
	.get_modes = dw_mipi_dsi_connector_get_modes,
	.best_encoder = dw_mipi_dsi_connector_best_encoder,
};
```

#### （14）、State – CRTC

``` c
I:\RK3399_Android8.1_MIPI\kernel\drivers\gpu\drm\rockchip\rockchip_drm_vop.c

static const struct drm_crtc_funcs vop_crtc_funcs = {
	......
	.atomic_duplicate_state = vop_crtc_duplicate_state,
	.atomic_destroy_state = vop_crtc_destroy_state,
};


```

#### （15）、State – Connector


``` c
I:\RK3399_Android8.1_MIPI\kernel\drivers\gpu\drm\rockchip\dw-mipi-dsi.c

static const struct drm_connector_funcs dw_mipi_dsi_atomic_connector_funcs = {
	......
	.atomic_duplicate_state = drm_atomic_helper_connector_duplicate_state,
	.atomic_destroy_state = drm_atomic_helper_connector_destroy_state,
};

```

#### （16）、Atomic Update – KMS




``` c
I:\RK3399_Android8.1_MIPI\kernel\drivers\gpu\drm\rockchip\dw-mipi-dsi.c
static const struct drm_mode_config_funcs rockchip_drm_mode_config_funcs = {
	.......
	.atomic_check = drm_atomic_helper_check,
	.atomic_commit = rockchip_drm_atomic_commit,
};


```


#### （17）、Atomic Update – CRTC


``` c
I:\RK3399_Android8.1_MIPI\kernel\drivers\gpu\drm\rockchip\rockchip_drm_vop.c
static const struct drm_crtc_helper_funcs vop_crtc_helper_funcs = {
	.load_lut = vop_crtc_load_lut,
	.enable = vop_crtc_enable,
	.disable = vop_crtc_disable,
	.mode_fixup = vop_crtc_mode_fixup,
	.atomic_check = vop_crtc_atomic_check,
	.atomic_flush = vop_crtc_atomic_flush,
	.atomic_begin = vop_crtc_atomic_begin,
};


```


#### （18）、Atomic Update – Encoder


``` c
I:\RK3399_Android8.1_MIPI\kernel\drivers\gpu\drm\rockchip\dw-mipi-dsi.c

static const struct drm_encoder_helper_funcs
dw_mipi_dsi_encoder_helper_funcs = {
	.mode_fixup = dw_mipi_dsi_encoder_mode_fixup,
	.mode_set = dw_mipi_dsi_encoder_mode_set,
	.enable = dw_mipi_dsi_encoder_enable,
	.disable = dw_mipi_dsi_encoder_disable,
	.atomic_check = dw_mipi_dsi_encoder_atomic_check,
};

```

#### （19）、Atomic Update – CRTC+Plane

``` c
I:\RK3399_Android8.1_MIPI\kernel\drivers\gpu\drm\rockchip\rockchip_drm_vop.c


static const struct drm_plane_helper_funcs plane_helper_funcs = {
	.prepare_fb = vop_plane_prepare_fb,
	.cleanup_fb = vop_plane_cleanup_fb,
	.atomic_check = vop_plane_atomic_check,
	.atomic_update = vop_plane_atomic_update,
	.atomic_disable = vop_plane_atomic_disable,
};

```
#### （20）、Vertical Blanking

``` c
I:\RK3399_Android8.1_MIPI\kernel\drivers\gpu\drm\rockchip\rockchip_drm_drv.c

static struct drm_driver rockchip_drm_driver = {
	.driver_features	= DRIVER_MODESET | DRIVER_GEM |
				  DRIVER_PRIME | DRIVER_ATOMIC |
				  DRIVER_RENDER,
	......
	.enable_vblank		= rockchip_drm_crtc_enable_vblank,
	.disable_vblank		= rockchip_drm_crtc_disable_vblank,
 }
 
 I:\RK3399_Android8.1_MIPI\kernel\drivers\gpu\drm\rockchip\rockchip_drm_vop.c
 static int vop_crtc_enable_vblank(struct drm_crtc *crtc)
{
	struct vop *vop = to_vop(crtc);
	unsigned long flags;

	......

	spin_lock_irqsave(&vop->irq_lock, flags);

	if (VOP_MAJOR(vop->version) == 3 && VOP_MINOR(vop->version) >= 7) {
		VOP_INTR_SET_TYPE(vop, clear, FS_FIELD_INTR, 1);
		VOP_INTR_SET_TYPE(vop, enable, FS_FIELD_INTR, 1);
	} else {
		VOP_INTR_SET_TYPE(vop, clear, FS_INTR, 1);
		VOP_INTR_SET_TYPE(vop, enable, FS_INTR, 1);
	}

	spin_unlock_irqrestore(&vop->irq_lock, flags);

	return 0;
}

```

#### （三）、参考资料(特别感谢)：
[（1）【DRM（Direct Rendering Manager）学习简介】](
https://blog.csdn.net/hexiaolong2009/article/details/83720940) 

[（2）【The DRM/KMS subsystem from a newbie’s point of view】](
https://events.static.linuxfound.org/sites/events/files/slides/brezillon-drm-kms.pdf) 

[（3）【Anatomy of an Atomic KMS Driver】](
https://elinux.orghttps://raw.githubusercontent.com/zhoujinjian777/PicGo/master/4/45/Atomic_kms_driver_pinchart.pdf) 

[（4）【Why and How to use KMS】](
https://events.static.linuxfound.org/sites/events/files/lcjpcojp13_pinchart.pdf) 
