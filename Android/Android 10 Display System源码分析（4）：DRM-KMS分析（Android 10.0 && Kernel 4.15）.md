---
title: Android 10 Display System源码分析（4）：DRM/KMS分析（Android 10.0 && Kernel 4.15）
cover: https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/post.cover.pictures/bing-wallpaper-2018.04.25.jpg
categories: 
  - Display
tags:
  - Android
  - Linux
  - Graphics
toc: true
abbrlink: 20210610
date: 2021-06-10 09:25:00
---


## (一)、DRM/KMS简介
### （1）、DRM（Direct Rendering Manager）简介
DRM 是 Linux 目前主流的图形显示框架，相比 FB 架构，DRM 更能适应当前日益更新的显示硬件。比如 FB 原生不支持多层合成，不支持 VSYNC，不支持 DMA-BUF，不支持异步更新，不支持 fence 机制等等，而这些功能 DRM 原生都支持。同时 DRM 可以统一管理 GPU 和 Display 驱动，使得软件架构更为统一，方便管理和维护。

DRM 从模块上划分，可以简单分为 3 部分：**libdrm、KMS、GEM**

![](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/Android10.Display.4/drm_user_kernel_space.svg)

#### 1.1、libdrm
对底层接口进行封装，向上层提供通用的 API 接口，主要是对各种 IOCTL 接口进行封装。

#### 1.2、GEM
Graphic Execution Manager，主要负责显示 buffer 的分配和释放，也是 GPU 唯一用到 DRM 的地方。

#### 1.3、基本元素
DRM 框架涉及到的元素很多，大致如下：
KMS：**CRTC，ENCODER，CONNECTOR，PLANE，FB，VBLANK，property**
GEM：**DUMB、PRIME、fence**

![](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/Android10.Display.4/drm_architecture.jpg)

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

### （2）、KMS
Kernel Mode Setting，所谓 Mode setting，其实说白了就两件事：更新画面和设置显示参数。
更新画面：显示 buffer 的切换，多图层的合成方式，以及每个图层的显示位置。
设置显示参数：包括分辨率、刷新率、电源状态（休眠唤醒）等。

## (二)、DRM/KMS驱动分析
首先看看RK3399 DRM Kernel启动框图：
![](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/Android10.Display.4/rockchip_drm_kernel_boot_up.png)




首先看看RK3399DRM Kernel启动UML流程图：

![](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/Android10.Display.4/Android10.Display.4.Kernel.Display.Flow.png)



### （1）、Drm device创建注册


``` c
X:\kernel\drivers\gpu\drm\rockchip\rockchip_drm_drv.c
static int rockchip_drm_bind(struct device *dev)
{
    struct drm_device *drm_dev;
    struct rockchip_drm_private *private;
    int ret;
    struct device_node *np = dev->of_node;
    struct device_node *parent_np;
    struct drm_crtc *crtc;
    printk("zjj.rk3399.kernel %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
​
    drm_dev = drm_dev_alloc(&rockchip_drm_driver, dev);
​
    dev_set_drvdata(dev, drm_dev);
​
    private = devm_kzalloc(drm_dev->dev, sizeof(*private), GFP_KERNEL);
    
    mutex_init(&private->commit_lock);
    INIT_WORK(&private->commit_work, rockchip_drm_atomic_work);
    drm_dev->dev_private = private;
​
    private->dmc_support = false;
    private->devfreq = devfreq_get_devfreq_by_phandle(dev, 0);
    ......
​
    INIT_LIST_HEAD(&private->psr_list);
    mutex_init(&private->psr_list_lock);
​
    ret = rockchip_drm_init_iommu(drm_dev);
​
    drm_mode_config_init(drm_dev);
​
    rockchip_drm_mode_config_init(drm_dev);
    rockchip_drm_create_properties(drm_dev);
    /* Try to bind all sub drivers. */
    ret = component_bind_all(dev, drm_dev);
​
    rockchip_attach_connector_property(drm_dev);
    ret = drm_vblank_init(drm_dev, drm_dev->mode_config.num_crtc);
​
    drm_mode_config_reset(drm_dev);
    rockchip_drm_set_property_default(drm_dev);
​
    /*
     * enable drm irq mode.
     * - with irq_enabled = true, we can use the vblank feature.
     */
    drm_dev->irq_enabled = true;
​
    /* init kms poll for handling hpd */
    drm_kms_helper_poll_init(drm_dev);
​
    rockchip_gem_pool_init(drm_dev);
#ifndef MODULE
    show_loader_logo(drm_dev);
#endif
    ret = of_reserved_mem_device_init(drm_dev->dev);
​
    ret = rockchip_drm_fbdev_init(drm_dev);
    
    drm_for_each_crtc(crtc, drm_dev) {
        struct drm_fb_helper *helper = private->fbdev_helper;
        struct rockchip_crtc_state *s = NULL;
​
        s = to_rockchip_crtc_state(crtc->state);
        if (is_support_hotplug(s->output_type))
            drm_framebuffer_get(helper->fb);
    }
​
    drm_dev->mode_config.allow_fb_modifiers = true;
​
    ret = drm_dev_register(drm_dev, 0);
​
    return 0;
}
```

### （2）、Drm driver信息


``` c
X:\kernel\drivers\gpu\drm\rockchip\rockchip_drm_drv.c
static struct drm_driver rockchip_drm_driver = {
    .driver_features    = DRIVER_MODESET | DRIVER_GEM |
                  DRIVER_PRIME | DRIVER_ATOMIC |
                  DRIVER_RENDER,
    .postclose      = rockchip_drm_postclose,
    .lastclose      = rockchip_drm_lastclose,
    .open           = rockchip_drm_open,
    .gem_vm_ops     = &drm_gem_cma_vm_ops,
    .gem_free_object_unlocked = rockchip_gem_free_object,
    .dumb_create        = rockchip_gem_dumb_create,
    .dumb_map_offset    = rockchip_gem_dumb_map_offset,
    .dumb_destroy       = drm_gem_dumb_destroy,
    .prime_handle_to_fd = drm_gem_prime_handle_to_fd,
    .prime_fd_to_handle = drm_gem_prime_fd_to_handle,
    .gem_prime_import   = drm_gem_prime_import,
    .gem_prime_export   = drm_gem_prime_export,
    .gem_prime_get_sg_table = rockchip_gem_prime_get_sg_table,
    .gem_prime_import_sg_table  = rockchip_gem_prime_import_sg_table,
    .gem_prime_vmap     = rockchip_gem_prime_vmap,
    .gem_prime_vunmap   = rockchip_gem_prime_vunmap,
    .gem_prime_mmap     = rockchip_gem_mmap_buf,
    .gem_prime_begin_cpu_access = rockchip_gem_prime_begin_cpu_access,
    .gem_prime_end_cpu_access = rockchip_gem_prime_end_cpu_access,
#ifdef CONFIG_DEBUG_FS
    .debugfs_init       = rockchip_drm_debugfs_init,
#endif
    .ioctls         = rockchip_ioctls,
    .num_ioctls     = ARRAY_SIZE(rockchip_ioctls),
    .fops           = &rockchip_drm_driver_fops,
    .name   = DRIVER_NAME,
    .desc   = DRIVER_DESC,
    .date   = DRIVER_DATE,
    .major  = DRIVER_MAJOR,
    .minor  = DRIVER_MINOR,
    .patchlevel = DRIVER_PATCH,
};
​
```

### （3）、Drm driver file operations


``` c
X:\kernel\drivers\gpu\drm\rockchip\rockchip_drm_drv.c
static const struct file_operations rockchip_drm_driver_fops = {
    .owner = THIS_MODULE,
    .open = drm_open,
    .mmap = rockchip_gem_mmap,
    .poll = drm_poll,
    .read = drm_read,
    .unlocked_ioctl = drm_ioctl,
    .compat_ioctl = drm_compat_ioctl,
    .release = drm_release,
};
```

### （4）、Mode Config


``` c
X:\kernel\drivers\gpu\drm\rockchip\rockchip_drm_drv.c
static int rockchip_drm_bind(struct device *dev)
{
drm_mode_config_init(drm_dev);
rockchip_drm_mode_config_init(drm_dev);
}
X:\kernel\drivers\gpu\drm\rockchip\rockchip_drm_fb.c
static const struct drm_mode_config_helper_funcs rockchip_mode_config_helpers = {
    .atomic_commit_tail = rockchip_atomic_helper_commit_tail_rpm,
};
​
static const struct drm_mode_config_funcs rockchip_drm_mode_config_funcs = {
    .fb_create = rockchip_user_fb_create,
    .output_poll_changed = rockchip_drm_output_poll_changed,
    .atomic_check = drm_atomic_helper_check,
    .atomic_commit = rockchip_drm_atomic_commit,
};
​
void rockchip_drm_mode_config_init(struct drm_device *dev)
{
    printk("zjj.rk3399.kernel %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
​
    dev->mode_config.min_width = 0;
    dev->mode_config.min_height = 0;
​
    /*
     * set max width and height as default value(4096x4096).
     * this value would be used to check framebuffer size limitation
     * at drm_mode_addfb().
     */
    dev->mode_config.max_width = 8192;
    dev->mode_config.max_height = 8192;
    dev->mode_config.async_page_flip = true;
​
    dev->mode_config.funcs = &rockchip_drm_mode_config_funcs;
    dev->mode_config.helper_private = &rockchip_mode_config_helpers;
}
​
```

### （5）、GEM

``` c
X:\kernel\drivers\gpu\drm\rockchip\rockchip_drm_drv.c
static struct drm_driver rockchip_drm_driver = {
    .driver_features    = DRIVER_MODESET | DRIVER_GEM |
                  DRIVER_PRIME | DRIVER_ATOMIC |
                  DRIVER_RENDER,
    .postclose      = rockchip_drm_postclose,
    .lastclose      = rockchip_drm_lastclose,
    .open           = rockchip_drm_open,
    .gem_vm_ops     = &drm_gem_cma_vm_ops,
    .gem_free_object_unlocked = rockchip_gem_free_object,
    .dumb_create        = rockchip_gem_dumb_create,
    .dumb_map_offset    = rockchip_gem_dumb_map_offset,
    .dumb_destroy       = drm_gem_dumb_destroy,
    .prime_handle_to_fd = drm_gem_prime_handle_to_fd,
    .prime_fd_to_handle = drm_gem_prime_fd_to_handle,
    .gem_prime_import   = drm_gem_prime_import,
    .gem_prime_export   = drm_gem_prime_export,
    .gem_prime_get_sg_table = rockchip_gem_prime_get_sg_table,
    .gem_prime_import_sg_table  = rockchip_gem_prime_import_sg_table,
    .gem_prime_vmap     = rockchip_gem_prime_vmap,
    .gem_prime_vunmap   = rockchip_gem_prime_vunmap,
    .gem_prime_mmap     = rockchip_gem_mmap_buf,
    .gem_prime_begin_cpu_access = rockchip_gem_prime_begin_cpu_access,
    .gem_prime_end_cpu_access = rockchip_gem_prime_end_cpu_access,
    ......
    }
    
X:\kernel\drivers\gpu\drm\drm_gem_cma_helper.c
const struct vm_operations_struct drm_gem_cma_vm_ops = {
    .open = drm_gem_vm_open,
    .close = drm_gem_vm_close,
};
```

### （6）、Frame Buffer

``` c
X:\kernel\drivers\gpu\drm\rockchip\rockchip_drm_fbdev.c
static const struct drm_fb_helper_funcs rockchip_drm_fb_helper_funcs = {
    .fb_probe = rockchip_drm_fbdev_create,
};
static struct fb_ops rockchip_drm_fbdev_ops = {
    .owner      = THIS_MODULE,
    .fb_check_var   = drm_fb_helper_check_var,
    .fb_set_par = drm_fb_helper_set_par,
    .fb_setcmap = drm_fb_helper_setcmap,
    .fb_pan_display = drm_fb_helper_pan_display,
    .fb_debug_enter = drm_fb_helper_debug_enter,
    .fb_debug_leave = drm_fb_helper_debug_leave,
    .fb_ioctl   = drm_fb_helper_ioctl,
    .fb_fillrect    = drm_fb_helper_cfb_fillrect,
    .fb_copyarea    = drm_fb_helper_cfb_copyarea,
    .fb_imageblit   = drm_fb_helper_cfb_imageblit,
    .fb_read    = drm_fb_helper_sys_read,
    .fb_write   = drm_fb_helper_sys_write,
    .fb_mmap        = rockchip_fbdev_mmap,
    .fb_blank       = rockchip_fbdev_blank,
    .fb_dmabuf_export   = rockchip_fbdev_get_dma_buf,
};
​
int rockchip_drm_fbdev_init(struct drm_device *dev)
{
    struct rockchip_drm_private *private = dev->dev_private;
    struct drm_fb_helper *helper;
    int ret;
​
    if (!dev->mode_config.num_crtc || !dev->mode_config.num_connector)
        return -EINVAL;
​
    helper = devm_kzalloc(dev->dev, sizeof(*helper), GFP_KERNEL);
    if (!helper)
        return -ENOMEM;
    private->fbdev_helper = helper;
​
    drm_fb_helper_prepare(dev, helper, &rockchip_drm_fb_helper_funcs);
​
    ret = drm_fb_helper_init(dev, helper, ROCKCHIP_MAX_CONNECTOR);
    if (ret < 0) {
        DRM_DEV_ERROR(dev->dev,
                  "Failed to initialize drm fb helper - %d.\n",
                  ret);
        return ret;
    }
​
    ret = drm_fb_helper_single_add_all_connectors(helper);
    if (ret < 0) {
        DRM_DEV_ERROR(dev->dev,
                  "Failed to add connectors - %d.\n", ret);
        goto err_drm_fb_helper_fini;
    }
​
    ret = drm_fb_helper_initial_config(helper, PREFERRED_BPP);
    if (ret < 0) {
        DRM_DEV_ERROR(dev->dev,
                  "Failed to set initial hw config - %d.\n",
                  ret);
        goto err_drm_fb_helper_fini;
    }
​
    return 0;
​
err_drm_fb_helper_fini:
    drm_fb_helper_fini(helper);
    return ret;
}
​
```

### （7）、Initialization – CRTC

``` c
X:\kernel\drivers\gpu\drm\rockchip\rockchip_drm_vop.c
static int vop_bind(struct device *dev, struct device *master, void *data)
{
    struct platform_device *pdev = to_platform_device(dev);
    const struct vop_data *vop_data;
    struct drm_device *drm_dev = data;
    struct vop *vop;
    struct resource *res;
    size_t alloc_size;
    int ret, irq, i;
    int num_wins = 0;
    bool dual_channel_swap = false;
    struct device_node *mcu = NULL;
    printk("zjj.rk3399.kernel %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
​
    vop_data = of_device_get_match_data(dev);
    ......
    ret = vop_create_crtc(vop);
    ......
​
    return 0;
}
​
static int vop_create_crtc(struct vop *vop)
{
    struct device *dev = vop->dev;
    const struct vop_data *vop_data = vop->data;
    struct drm_device *drm_dev = vop->drm_dev;
    struct rockchip_drm_private *private = drm_dev->dev_private;
    struct drm_plane *primary = NULL, *cursor = NULL, *plane, *tmp;
    struct drm_crtc *crtc = &vop->crtc;
    struct device_node *port;
    uint64_t feature = 0;
    int ret = 0;
    int i;
    printk("zjj.rk3399.kernel %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
​
    ret = drm_crtc_init_with_planes(drm_dev, crtc, primary, cursor,
                    &vop_crtc_funcs, NULL);
    
    drm_crtc_helper_add(crtc, &vop_crtc_helper_funcs);
​
   ......
}
​
static const struct drm_crtc_funcs vop_crtc_funcs = {
    .gamma_set = vop_crtc_gamma_set,
    .set_config = drm_atomic_helper_set_config,
    .page_flip = drm_atomic_helper_page_flip,
    .destroy = vop_crtc_destroy,
    .reset = vop_crtc_reset,
    .atomic_get_property = vop_crtc_atomic_get_property,
    .atomic_set_property = vop_crtc_atomic_set_property,
    .atomic_duplicate_state = vop_crtc_duplicate_state,
    .atomic_destroy_state = vop_crtc_destroy_state,
    .enable_vblank = vop_crtc_enable_vblank,
    .disable_vblank = vop_crtc_disable_vblank,
    .set_crc_source = vop_crtc_set_crc_source,
    .verify_crc_source = vop_crtc_verify_crc_source,
};
​
static const struct drm_crtc_helper_funcs vop_crtc_helper_funcs = {
    .mode_fixup = vop_crtc_mode_fixup,
    .atomic_check = vop_crtc_atomic_check,
    .atomic_flush = vop_crtc_atomic_flush,
    .atomic_enable = vop_crtc_atomic_enable,
    .atomic_disable = vop_crtc_atomic_disable,
};
```

### （8）、Initialization – Plane


``` c
X:\kernel\drivers\gpu\drm\rockchip\rockchip_drm_vop.c
​
static const struct drm_plane_funcs vop_plane_funcs = {
    .update_plane   = rockchip_atomic_helper_update_plane,
    .disable_plane  = rockchip_atomic_helper_disable_plane,
    .destroy = vop_plane_destroy,
    .reset = vop_atomic_plane_reset,
    .atomic_duplicate_state = vop_atomic_plane_duplicate_state,
    .atomic_destroy_state = vop_atomic_plane_destroy_state,
    .atomic_set_property = vop_atomic_plane_set_property,
    .atomic_get_property = vop_atomic_plane_get_property,
};
​
static const struct drm_plane_helper_funcs plane_helper_funcs = {
    .prepare_fb = vop_plane_prepare_fb,
    .cleanup_fb = vop_plane_cleanup_fb,
    .atomic_check = vop_plane_atomic_check,
    .atomic_update = vop_plane_atomic_update,
    .atomic_disable = vop_plane_atomic_disable,
};
​
static int vop_plane_init(struct vop *vop, struct vop_win *win,
              unsigned long possible_crtcs)
{
    struct rockchip_drm_private *private = vop->drm_dev->dev_private;
    struct drm_plane *share = NULL;
    uint64_t feature = 0;
    int ret;
    printk("zjj.rk3399.kernel %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
​
    if (win->parent)
        share = &win->parent->base;
​
    ret = drm_share_plane_init(vop->drm_dev, &win->base, share,
                   possible_crtcs, &vop_plane_funcs,
                   win->data_formats, win->nformats, win->type);
    
    drm_plane_helper_add(&win->base, &plane_helper_funcs);
    drm_object_attach_property(&win->base.base,
                   vop->plane_zpos_prop, win->win_id);
    ......
    return 0;
}
​
```

### （9）、Initialization – Encoder && Connector


``` c
X:\kernel\drivers\gpu\drm\rockchip\dw-mipi-dsi.c
static const struct drm_encoder_funcs dw_mipi_dsi_encoder_funcs = {
    .destroy = drm_encoder_cleanup,
};
static const struct drm_connector_funcs dw_mipi_dsi_atomic_connector_funcs = {
    .fill_modes = drm_helper_probe_single_connector_modes,
    .destroy = dw_mipi_dsi_drm_connector_destroy,
    .reset = drm_atomic_helper_connector_reset,
    .atomic_duplicate_state = drm_atomic_helper_connector_duplicate_state,
    .atomic_destroy_state = drm_atomic_helper_connector_destroy_state,
};
static struct drm_connector_helper_funcs dw_mipi_dsi_connector_helper_funcs = {
    .get_modes = dw_mipi_dsi_connector_get_modes,
};
static const struct drm_encoder_helper_funcs
dw_mipi_dsi_encoder_helper_funcs = {
    .enable = dw_mipi_dsi_encoder_enable,
    .disable = dw_mipi_dsi_encoder_disable,
    .atomic_check = dw_mipi_dsi_encoder_atomic_check,
    .atomic_mode_set = dw_mipi_dsi_encoder_atomic_mode_set,
    .loader_protect = dw_mipi_dsi_encoder_loader_protect,
};
​
static int dw_mipi_dsi_bind(struct device *dev, struct device *master,
                void *data)
{
    struct dw_mipi_dsi *dsi = dev_get_drvdata(dev);
    struct drm_device *drm = data;
    struct drm_encoder *encoder = &dsi->encoder;
    struct drm_connector *connector = &dsi->connector;
    int ret;
    printk("zjj.rk3399.kernel %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
​
    ret = dw_mipi_dsi_dual_channel_probe(dsi);
    if (ret)
        return ret;
​
    if (dsi->master)
        return 0;
​
    dsi->panel = of_drm_find_panel(dsi->client);
    if (IS_ERR(dsi->panel))
        return -EPROBE_DEFER;
​
    encoder->port = dev->of_node;
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
​
    ret = drm_encoder_init(drm, encoder, &dw_mipi_dsi_encoder_funcs,
                   DRM_MODE_ENCODER_DSI, NULL);
    if (ret) {
        DRM_DEV_ERROR(dev, "Failed to initialize encoder with drm\n");
        return ret;
    }
​
    drm_encoder_helper_add(encoder, &dw_mipi_dsi_encoder_helper_funcs);
​
    connector->port = dev->of_node;
​
    drm_connector_init(drm, connector, &dw_mipi_dsi_atomic_connector_funcs,
               DRM_MODE_CONNECTOR_DSI);
    drm_connector_helper_add(connector,
                 &dw_mipi_dsi_connector_helper_funcs);
    drm_connector_attach_encoder(connector, encoder);
​
    ret = drm_panel_attach(dsi->panel, connector);
    if (ret) {
        DRM_DEV_ERROR(dsi->dev, "Failed to attach panel: %d\n", ret);
        goto err_cleanup;
    }
​
    pm_runtime_enable(dsi->dev);
    if (dsi->slave)
        pm_runtime_enable(dsi->slave->dev);
​
    return 0;
​
err_cleanup:
    connector->funcs->destroy(connector);
    encoder->funcs->destroy(encoder);
    return ret;
}
```

### （10）、Initialization – Panel

``` c
X:\kernel\drivers\gpu\drm\panel\panel-simple.c
static const struct drm_panel_funcs panel_simple_funcs = {
    .loader_protect = panel_simple_loader_protect,
    .disable = panel_simple_disable,
    .unprepare = panel_simple_unprepare,
    .prepare = panel_simple_prepare,
    .enable = panel_simple_enable,
    .get_modes = panel_simple_get_modes,
    .get_timings = panel_simple_get_timings,
};
​
static int panel_simple_probe(struct device *dev, const struct panel_desc *desc)
{
    struct device_node *backlight, *ddc;
    struct panel_simple *panel;
    const char *cmd_type;
    int err;
    printk("zjj.rk3399.kernel %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
​
    panel = devm_kzalloc(dev, sizeof(*panel), GFP_KERNEL);
    ......
    drm_panel_init(&panel->base);
    panel->base.dev = dev;
    panel->base.funcs = &panel_simple_funcs;
​
    err = drm_panel_add(&panel->base);
​
    dev_set_drvdata(dev, panel);
​
    return 0;
}
```

## （三）、Log：


[2020/10/16 11:06:31] [    0.735426] zjj.rk3399.kernel drivers/gpu/drm/drm_drv.c drm_core_init 1003 
[2020/10/16 11:06:31] [    0.735426] zjj.rk3399.kernel drivers/gpu/drm/drm_drv.c drm_core_init 1003 
[2020/10/16 11:06:31] [    0.735426] zjj.rk3399.kernel drivers/gpu/drm/drm_drv.c drm_core_init 1003 
[2020/10/16 11:06:31] [    0.738303] zjj.rk3399.kernel drivers/gpu/drm/rockchip/dw-mipi-dsi.c dw_mipi_dsi_probe 1650 
[2020/10/16 11:06:31] [    0.738434] zjj.rk3399.kernel drivers/gpu/drm/rockchip/dw-mipi-dsi.c mipi_dphy_attach 659 
[2020/10/16 11:06:31] [    0.738537] zjj.rk3399.kernel drivers/gpu/drm/drm_mipi_dsi.c mipi_dsi_host_register 288 
[2020/10/16 11:06:31] [    0.738537] zjj.rk3399.kernel drivers/gpu/drm/drm_mipi_dsi.c mipi_dsi_host_register 288 
[2020/10/16 11:06:31] [    0.739472] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c rockchip_drm_platform_probe 1912 
[2020/10/16 11:06:31] [    0.739472] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c rockchip_drm_platform_probe 1912 
[2020/10/16 11:06:31] [    0.739472] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c rockchip_drm_platform_probe 1912 
[2020/10/16 11:06:31] [    0.739486] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c rockchip_drm_platform_of_probe 1858 
[2020/10/16 11:06:31] [    0.739486] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c rockchip_drm_platform_of_probe 1858 
[2020/10/16 11:06:31] [    0.739486] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c rockchip_drm_platform_of_probe 1858 
[2020/10/16 11:06:31] [    0.739504] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c rockchip_drm_match_add 1821 
[2020/10/16 11:06:31] [    0.739504] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c rockchip_drm_match_add 1821 
[2020/10/16 11:06:31] [    0.739504] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c rockchip_drm_match_add 1821 
[2020/10/16 11:06:31] [    0.739889] rockchip-drm display-subsystem: Linked as a consumer to ff8f0000.vop
[2020/10/16 11:06:31] [    0.739915] rockchip-drm display-subsystem: Linked as a consumer to ff900000.vop
[2020/10/16 11:06:31] [    0.740552] rockchip-drm display-subsystem: Linked as a consumer to fec00000.dp
[2020/10/16 11:06:31] [    0.740894] rockchip-drm display-subsystem: Linked as a consumer to ff940000.hdmi
[2020/10/16 11:06:31] [    0.741197] rockchip-drm display-subsystem: Linked as a consumer to ff960000.dsi
[2020/10/16 11:06:31] [    0.744650] zjj.rk3399.kernel drivers/gpu/drm/panel/panel-simple.c panel_simple_dsi_probe 3401 
[2020/10/16 11:06:31] [    0.744762] zjj.rk3399.kernel drivers/gpu/drm/panel/panel-simple.c panel_simple_probe 681 
[2020/10/16 11:06:31] [    0.745053] zjj.rk3399.kernel drivers/gpu/drm/rockchip/dw-mipi-dsi.c dw_mipi_dsi_host_attach 716 
[2020/10/16 11:06:32] [    1.721618] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c rockchip_drm_bind 1475 
[2020/10/16 11:06:32] [    1.721618] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c rockchip_drm_bind 1475 
[2020/10/16 11:06:32] [    1.721618] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c rockchip_drm_bind 1475 
[2020/10/16 11:06:32] [    1.721828] rockchip-drm display-subsystem: defer getting devfreq
[2020/10/16 11:06:32] [    1.721857] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c rockchip_drm_init_iommu 1186 
[2020/10/16 11:06:32] [    1.721857] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c rockchip_drm_init_iommu 1186 
[2020/10/16 11:06:32] [    1.721857] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c rockchip_drm_init_iommu 1186 
[2020/10/16 11:06:32] [    1.721947] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_drm_mode_config_init 542 
[2020/10/16 11:06:32] [    1.721947] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_drm_mode_config_init 542 
[2020/10/16 11:06:32] [    1.721947] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_drm_mode_config_init 542 
[2020/10/16 11:06:32] [    1.721963] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c rockchip_drm_create_properties 1299 
[2020/10/16 11:06:32] [    1.721963] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c rockchip_drm_create_properties 1299 
[2020/10/16 11:06:32] [    1.721963] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c rockchip_drm_create_properties 1299 
[2020/10/16 11:06:32] [    1.722009] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_bind 4498 
[2020/10/16 11:06:32] [    1.722009] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_bind 4498 
[2020/10/16 11:06:32] [    1.722034] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_win_init 4361 
[2020/10/16 11:06:32] [    1.722034] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_win_init 4361 
[2020/10/16 11:06:32] [    1.722204] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_create_crtc 4170 
[2020/10/16 11:06:32] [    1.722204] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_create_crtc 4170 
[2020/10/16 11:06:32] [    1.722219] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_init 4055 
[2020/10/16 11:06:32] [    1.722219] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_init 4055 
[2020/10/16 11:06:32] [    1.722241] zjj.rk3399.kernel drivers/gpu/drm/drm_crtc.c drm_crtc_init_with_planes 274 
[2020/10/16 11:06:32] [    1.722241] zjj.rk3399.kernel drivers/gpu/drm/drm_crtc.c drm_crtc_init_with_planes 274 
[2020/10/16 11:06:32] [    1.722241] zjj.rk3399.kernel drivers/gpu/drm/drm_crtc.c drm_crtc_init_with_planes 274 
[2020/10/16 11:06:32] [    1.722260] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_init 4055 
[2020/10/16 11:06:32] [    1.722260] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_init 4055 
[2020/10/16 11:06:32] [    1.722283] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_init 4055 
[2020/10/16 11:06:32] [    1.722283] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_init 4055 
[2020/10/16 11:06:32] [    1.722302] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_init 4055 
[2020/10/16 11:06:32] [    1.722302] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_init 4055 
[2020/10/16 11:06:32] [    1.722319] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_init 4055 
[2020/10/16 11:06:32] [    1.722319] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_init 4055 
[2020/10/16 11:06:32] [    1.722338] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c rockchip_register_crtc_funcs 1137 
[2020/10/16 11:06:32] [    1.722338] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c rockchip_register_crtc_funcs 1137 
[2020/10/16 11:06:32] [    1.722381] rockchip-drm display-subsystem: bound ff8f0000.vop (ops vop_component_ops)
[2020/10/16 11:06:32] [    1.722390] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_bind 4498 
[2020/10/16 11:06:32] [    1.722390] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_bind 4498 
[2020/10/16 11:06:32] [    1.722409] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_win_init 4361 
[2020/10/16 11:06:32] [    1.722409] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_win_init 4361 
[2020/10/16 11:06:32] [    1.722524] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_create_crtc 4170 
[2020/10/16 11:06:32] [    1.722524] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_create_crtc 4170 
[2020/10/16 11:06:32] [    1.722538] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_init 4055 
[2020/10/16 11:06:32] [    1.722538] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_init 4055 
[2020/10/16 11:06:32] [    1.722556] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_init 4055 
[2020/10/16 11:06:32] [    1.722556] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_init 4055 
[2020/10/16 11:06:32] [    1.722576] zjj.rk3399.kernel drivers/gpu/drm/drm_crtc.c drm_crtc_init_with_planes 274 
[2020/10/16 11:06:32] [    1.722576] zjj.rk3399.kernel drivers/gpu/drm/drm_crtc.c drm_crtc_init_with_planes 274 
[2020/10/16 11:06:32] [    1.722576] zjj.rk3399.kernel drivers/gpu/drm/drm_crtc.c drm_crtc_init_with_planes 274 
[2020/10/16 11:06:32] [    1.722594] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_init 4055 
[2020/10/16 11:06:32] [    1.722594] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_init 4055 
[2020/10/16 11:06:32] [    1.722614] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_init 4055 
[2020/10/16 11:06:32] [    1.722614] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_init 4055 
[2020/10/16 11:06:32] [    1.722634] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_init 4055 
[2020/10/16 11:06:32] [    1.722634] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_init 4055 
[2020/10/16 11:06:32] [    1.722645] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_init 4055 
[2020/10/16 11:06:32] [    1.722645] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_init 4055 
[2020/10/16 11:06:32] [    1.722662] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_init 4055 
[2020/10/16 11:06:32] [    1.722662] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_init 4055 
[2020/10/16 11:06:32] [    1.722678] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_init 4055 
[2020/10/16 11:06:32] [    1.722678] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_init 4055 
[2020/10/16 11:06:32] [    1.722694] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_init 4055 
[2020/10/16 11:06:32] [    1.722694] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_init 4055 
[2020/10/16 11:06:32] [    1.722710] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_init 4055 
[2020/10/16 11:06:32] [    1.722710] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_init 4055 
[2020/10/16 11:06:32] [    1.722727] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c rockchip_register_crtc_funcs 1137 
[2020/10/16 11:06:32] [    1.722727] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c rockchip_register_crtc_funcs 1137 
[2020/10/16 11:06:32] [    1.722817] rockchip-drm display-subsystem: bound ff900000.vop (ops vop_component_ops)
[2020/10/16 11:06:32] [    1.723409] zjj.rk3399.kernel drivers/gpu/drm/drm_encoder.c drm_encoder_init 113 
[2020/10/16 11:06:32] [    1.723409] zjj.rk3399.kernel drivers/gpu/drm/drm_encoder.c drm_encoder_init 113 
[2020/10/16 11:06:32] [    1.723409] zjj.rk3399.kernel drivers/gpu/drm/drm_encoder.c drm_encoder_init 113 
[2020/10/16 11:06:32] [    1.723428] zjj.rk3399.kernel drivers/gpu/drm/drm_connector.c drm_connector_init 201 
[2020/10/16 11:06:32] [    1.723428] zjj.rk3399.kernel drivers/gpu/drm/drm_connector.c drm_connector_init 201 
[2020/10/16 11:06:32] [    1.723428] zjj.rk3399.kernel drivers/gpu/drm/drm_connector.c drm_connector_init 201 
[2020/10/16 11:06:32] [    1.723453] rockchip-drm display-subsystem: bound fec00000.dp (ops cdn_dp_component_ops)
[2020/10/16 11:06:32] [    1.723511] zjj.rk3399.kernel drivers/gpu/drm/drm_encoder.c drm_encoder_init 113 
[2020/10/16 11:06:32] [    1.723511] zjj.rk3399.kernel drivers/gpu/drm/drm_encoder.c drm_encoder_init 113 
[2020/10/16 11:06:32] [    1.723511] zjj.rk3399.kernel drivers/gpu/drm/drm_encoder.c drm_encoder_init 113 
[2020/10/16 11:06:32] [    1.725840] zjj.rk3399.kernel drivers/gpu/drm/drm_connector.c drm_connector_init 201 
[2020/10/16 11:06:32] [    1.725840] zjj.rk3399.kernel drivers/gpu/drm/drm_connector.c drm_connector_init 201 
[2020/10/16 11:06:32] [    1.725840] zjj.rk3399.kernel drivers/gpu/drm/drm_connector.c drm_connector_init 201 
[2020/10/16 11:06:32] [    1.725917] rockchip-drm display-subsystem: bound ff940000.hdmi (ops dw_hdmi_rockchip_ops)
[2020/10/16 11:06:32] [    1.725940] zjj.rk3399.kernel drivers/gpu/drm/rockchip/dw-mipi-dsi.c dw_mipi_dsi_bind 1485 
[2020/10/16 11:06:32] [    1.725963] zjj.rk3399.kernel drivers/gpu/drm/drm_encoder.c drm_encoder_init 113 
[2020/10/16 11:06:32] [    1.725963] zjj.rk3399.kernel drivers/gpu/drm/drm_encoder.c drm_encoder_init 113 
[2020/10/16 11:06:32] [    1.725963] zjj.rk3399.kernel drivers/gpu/drm/drm_encoder.c drm_encoder_init 113 
[2020/10/16 11:06:32] [    1.725976] zjj.rk3399.kernel drivers/gpu/drm/drm_connector.c drm_connector_init 201 
[2020/10/16 11:06:32] [    1.725976] zjj.rk3399.kernel drivers/gpu/drm/drm_connector.c drm_connector_init 201 
[2020/10/16 11:06:32] [    1.725976] zjj.rk3399.kernel drivers/gpu/drm/drm_connector.c drm_connector_init 201 
[2020/10/16 11:06:32] [    1.726002] rockchip-drm display-subsystem: bound ff960000.dsi (ops dw_mipi_dsi_ops)
[2020/10/16 11:06:32] [    1.726026] [drm] Supports vblank timestamp caching Rev 2 (21.10.2013).
[2020/10/16 11:06:32] [    1.726033] [drm] No driver support for vblank timestamp query.
[2020/10/16 11:06:32] [    1.726079] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c rockchip_drm_set_property_default 1418 
[2020/10/16 11:06:32] [    1.726079] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c rockchip_drm_set_property_default 1418 
[2020/10/16 11:06:32] [    1.726079] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c rockchip_drm_set_property_default 1418 
[2020/10/16 11:06:32] [    1.726103] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c drm_atomic_helper_duplicate_state 3924 
[2020/10/16 11:06:32] [    1.726103] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c drm_atomic_helper_duplicate_state 3924 
[2020/10/16 11:06:32] [    1.726103] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c drm_atomic_helper_duplicate_state 3924 
[2020/10/16 11:06:32] [    1.726124] drm_atomic.c Allocated atomic state 0000000095d54fe3
[2020/10/16 11:06:32] [    1.726136] drm_atomic.c Added [CRTC:56:crtc-0] 0000000096d7e41a state to 0000000095d54fe3
[2020/10/16 11:06:32] [    1.726153] drm_atomic.c Added [CRTC:67:crtc-1] 00000000c8c67c28 state to 0000000095d54fe3
[2020/10/16 11:06:32] [    1.726163] drm_atomic.c Added [PLANE:55:plane-0] 0000000093318889 state to 0000000095d54fe3
[2020/10/16 11:06:32] [    1.726180] drm_atomic.c Added [PLANE:57:plane-1] 000000003de3cd07 state to 0000000095d54fe3
[2020/10/16 11:06:32] [    1.726189] drm_atomic.c Added [PLANE:59:plane-2] 00000000921d0fba state to 0000000095d54fe3
[2020/10/16 11:06:32] [    1.726198] drm_atomic.c Added [PLANE:60:plane-3] 0000000093e0dae3 state to 0000000095d54fe3
[2020/10/16 11:06:32] [    1.726206] drm_atomic.c Added [PLANE:61:plane-4] 0000000080e2c4f7 state to 0000000095d54fe3
[2020/10/16 11:06:32] [    1.726214] drm_atomic.c Added [PLANE:65:plane-5] 000000006f806d14 state to 0000000095d54fe3
[2020/10/16 11:06:32] [    1.726226] drm_atomic.c Added [PLANE:66:plane-6] 00000000b5f94e98 state to 0000000095d54fe3
[2020/10/16 11:06:32] [    1.726242] drm_atomic.c Added [PLANE:68:plane-7] 0000000008eb7b42 state to 0000000095d54fe3
[2020/10/16 11:06:32] [    1.726251] drm_atomic.c Added [PLANE:70:plane-8] 000000000316616c state to 0000000095d54fe3
[2020/10/16 11:06:32] [    1.726271] drm_atomic.c Added [PLANE:72:plane-9] 000000003e149226 state to 0000000095d54fe3
[2020/10/16 11:06:32] [    1.726280] drm_atomic.c Added [PLANE:73:plane-10] 000000002a4fb1ba state to 0000000095d54fe3
[2020/10/16 11:06:32] [    1.726301] drm_atomic.c Added [PLANE:74:plane-11] 00000000d7d155b4 state to 0000000095d54fe3
[2020/10/16 11:06:32] [    1.726312] drm_atomic.c Added [PLANE:75:plane-12] 000000004153e0be state to 0000000095d54fe3
[2020/10/16 11:06:32] [    1.726329] drm_atomic.c Added [PLANE:76:plane-13] 000000002cdebae9 state to 0000000095d54fe3
[2020/10/16 11:06:32] [    1.726338] drm_atomic.c Added [PLANE:77:plane-14] 0000000051580220 state to 0000000095d54fe3
[2020/10/16 11:06:32] [    1.726357] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726357] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726357] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726376] drm_atomic.c Added [CONNECTOR:79:DP-1] 000000002977e92b state to 0000000095d54fe3
[2020/10/16 11:06:32] [    1.726391] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726391] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726391] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726403] drm_atomic.c Added [CONNECTOR:81:HDMI-A-1] 000000007fda130a state to 0000000095d54fe3
[2020/10/16 11:06:32] [    1.726418] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726418] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726418] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726431] drm_atomic.c Added [CONNECTOR:89:DSI-1] 00000000da14bfe5 state to 0000000095d54fe3
[2020/10/16 11:06:32] [    1.726439] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726439] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726439] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726446] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726446] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726446] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726454] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726454] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726454] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726464] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726464] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726464] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726481] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726481] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726481] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726488] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726488] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726488] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726498] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726498] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726498] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726516] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726516] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726516] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726523] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726523] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726523] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726532] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726532] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726532] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726550] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726550] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726550] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726557] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726557] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726557] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726566] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726566] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726566] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726584] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726584] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726584] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726591] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726591] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726591] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.726601] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_commit 2019 
[2020/10/16 11:06:32] [    1.726601] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_commit 2019 
[2020/10/16 11:06:32] [    1.726601] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_commit 2019 
[2020/10/16 11:06:32] [    1.726617] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_check_only 1946 
[2020/10/16 11:06:32] [    1.726617] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_check_only 1946 
[2020/10/16 11:06:32] [    1.726617] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_check_only 1946 
[2020/10/16 11:06:32] [    1.726631] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c drm_atomic_helper_check 916 
[2020/10/16 11:06:32] [    1.726631] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c drm_atomic_helper_check 916 
[2020/10/16 11:06:32] [    1.726631] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c drm_atomic_helper_check 916 
[2020/10/16 11:06:32] [    1.726646] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c drm_atomic_helper_check_modeset 597 
[2020/10/16 11:06:32] [    1.726646] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c drm_atomic_helper_check_modeset 597 
[2020/10/16 11:06:32] [    1.726646] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c drm_atomic_helper_check_modeset 597 
[2020/10/16 11:06:32] [    1.726655] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c update_connector_routing 287 
[2020/10/16 11:06:32] [    1.726655] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c update_connector_routing 287 
[2020/10/16 11:06:32] [    1.726686] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c update_connector_routing 287 
[2020/10/16 11:06:32] [    1.726686] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c update_connector_routing 287 
[2020/10/16 11:06:32] [    1.726704] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c update_connector_routing 287 
[2020/10/16 11:06:32] [    1.726704] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c update_connector_routing 287 
[2020/10/16 11:06:32] [    1.726761] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.726761] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.726777] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.726777] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.726785] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.726785] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.726795] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.726795] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.726813] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.726813] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.726820] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.726820] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.726830] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.726830] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.726847] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.726847] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.726854] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.726854] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.726861] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.726861] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.726875] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.726875] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.726882] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.726882] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.726888] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.726888] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.726903] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.726903] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.726910] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.726910] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.726920] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_crtc_atomic_check 3212 
[2020/10/16 11:06:32] [    1.726920] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_crtc_atomic_check 3212 
[2020/10/16 11:06:32] [    1.726944] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_crtc_atomic_check 3212 
[2020/10/16 11:06:32] [    1.726944] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_crtc_atomic_check 3212 
[2020/10/16 11:06:32] [    1.726961] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_drm_atomic_commit 470 
[2020/10/16 11:06:32] [    1.726961] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_drm_atomic_commit 470 
[2020/10/16 11:06:32] [    1.726961] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_drm_atomic_commit 470 
[2020/10/16 11:06:32] [    1.726988] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_atomic_commit_complete 400 
[2020/10/16 11:06:32] [    1.726988] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_atomic_commit_complete 400 
[2020/10/16 11:06:32] [    1.727017] drm_atomic.c Clearing atomic state 0000000095d54fe3
[2020/10/16 11:06:32] [    1.727053] drm_atomic.c Freeing atomic state 0000000095d54fe3
[2020/10/16 11:06:32] [    1.727072] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c rockchip_gem_pool_init 1354 
[2020/10/16 11:06:32] [    1.727072] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c rockchip_gem_pool_init 1354 
[2020/10/16 11:06:32] [    1.727139] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c show_loader_logo 863 
[2020/10/16 11:06:32] [    1.727139] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c show_loader_logo 863 
[2020/10/16 11:06:32] [    1.727155] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c init_loader_memory 246 
[2020/10/16 11:06:32] [    1.727155] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c init_loader_memory 246 
[2020/10/16 11:06:32] [    1.727213] drm_atomic.c Allocated atomic state 00000000b87b3dfb
[2020/10/16 11:06:32] [    1.727230] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_fb_alloc 125 
[2020/10/16 11:06:32] [    1.727230] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_fb_alloc 125 
[2020/10/16 11:06:32] [    1.727243] zjj.rk3399.kernel drivers/gpu/drm/drm_framebuffer.c drm_framebuffer_init 691 
[2020/10/16 11:06:32] [    1.727243] zjj.rk3399.kernel drivers/gpu/drm/drm_framebuffer.c drm_framebuffer_init 691 
[2020/10/16 11:06:32] [    1.727243] zjj.rk3399.kernel drivers/gpu/drm/drm_framebuffer.c drm_framebuffer_init 691 
[2020/10/16 11:06:32] [    1.727277] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.727277] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.727277] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.727300] drm_atomic.c Added [CONNECTOR:81:HDMI-A-1] 0000000018aa6012 state to 00000000b87b3dfb
[2020/10/16 11:06:32] [    1.727340] rockchip-drm display-subsystem: connector[HDMI-A-1] can't found any modes
[2020/10/16 11:06:32] [    1.727369] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_fb_alloc 125 
[2020/10/16 11:06:32] [    1.727369] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_fb_alloc 125 
[2020/10/16 11:06:32] [    1.727384] zjj.rk3399.kernel drivers/gpu/drm/drm_framebuffer.c drm_framebuffer_init 691 
[2020/10/16 11:06:32] [    1.727384] zjj.rk3399.kernel drivers/gpu/drm/drm_framebuffer.c drm_framebuffer_init 691 
[2020/10/16 11:06:32] [    1.727384] zjj.rk3399.kernel drivers/gpu/drm/drm_framebuffer.c drm_framebuffer_init 691 
[2020/10/16 11:06:32] [    1.727406] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.727406] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.727406] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.727426] drm_atomic.c Added [CONNECTOR:89:DSI-1] 000000008c315a3e state to 00000000b87b3dfb
[2020/10/16 11:06:32] [    1.727435] zjj.rk3399.kernel drivers/gpu/drm/panel/panel-simple.c panel_simple_regulator_enable 433 
[2020/10/16 11:06:32] [    1.727480] zjj.rk3399.kernel drivers/gpu/drm/panel/panel-simple.c panel_simple_get_modes 629 
[2020/10/16 11:06:32] [    1.727488] zjj.rk3399.kernel drivers/gpu/drm/panel/panel-simple.c panel_simple_get_fixed_modes 373 
[2020/10/16 11:06:32] [    1.727549] drm_atomic.c Added [CRTC:56:crtc-0] 00000000adfe7ba0 state to 00000000b87b3dfb
[2020/10/16 11:06:32] [    1.727582] drm_atomic.c Set [MODE:1088x1920] for [CRTC:56:crtc-0] state 00000000adfe7ba0
[2020/10/16 11:06:32] [    1.727668] drm_atomic.c Added [PLANE:55:plane-0] 000000001d64f6c8 state to 00000000b87b3dfb
[2020/10/16 11:06:32] [    1.727688] drm_atomic.c Clearing atomic state 00000000b87b3dfb
[2020/10/16 11:06:32] [    1.727705] drm_atomic.c Freeing atomic state 00000000b87b3dfb
[2020/10/16 11:06:32] [    1.727720] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c drm_atomic_helper_duplicate_state 3924 
[2020/10/16 11:06:32] [    1.727720] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c drm_atomic_helper_duplicate_state 3924 
[2020/10/16 11:06:32] [    1.727720] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c drm_atomic_helper_duplicate_state 3924 
[2020/10/16 11:06:32] [    1.727727] drm_atomic.c Allocated atomic state 00000000b87b3dfb
[2020/10/16 11:06:32] [    1.727757] drm_atomic.c Added [CRTC:56:crtc-0] 0000000096d7e41a state to 00000000b87b3dfb
[2020/10/16 11:06:32] [    1.727766] drm_atomic.c Added [CRTC:67:crtc-1] 00000000861a7111 state to 00000000b87b3dfb
[2020/10/16 11:06:32] [    1.727774] drm_atomic.c Added [PLANE:55:plane-0] 000000002aa7d4eb state to 00000000b87b3dfb
[2020/10/16 11:06:32] [    1.727792] drm_atomic.c Added [PLANE:57:plane-1] 0000000004d5e0b6 state to 00000000b87b3dfb
[2020/10/16 11:06:32] [    1.727800] drm_atomic.c Added [PLANE:59:plane-2] 00000000582a25a3 state to 00000000b87b3dfb
[2020/10/16 11:06:32] [    1.727811] drm_atomic.c Added [PLANE:60:plane-3] 00000000a9a4f5c8 state to 00000000b87b3dfb
[2020/10/16 11:06:32] [    1.727829] drm_atomic.c Added [PLANE:61:plane-4] 00000000a4cf8359 state to 00000000b87b3dfb
[2020/10/16 11:06:32] [    1.727837] drm_atomic.c Added [PLANE:65:plane-5] 00000000d34b8709 state to 00000000b87b3dfb
[2020/10/16 11:06:32] [    1.727858] drm_atomic.c Added [PLANE:66:plane-6] 00000000b6029f00 state to 00000000b87b3dfb
[2020/10/16 11:06:32] [    1.727866] drm_atomic.c Added [PLANE:68:plane-7] 0000000093f43ff1 state to 00000000b87b3dfb
[2020/10/16 11:06:32] [    1.727887] drm_atomic.c Added [PLANE:70:plane-8] 0000000096f71a41 state to 00000000b87b3dfb
[2020/10/16 11:06:32] [    1.727895] drm_atomic.c Added [PLANE:72:plane-9] 00000000ab393d6f state to 00000000b87b3dfb
[2020/10/16 11:06:32] [    1.727916] drm_atomic.c Added [PLANE:73:plane-10] 0000000093318889 state to 00000000b87b3dfb
[2020/10/16 11:06:32] [    1.727924] drm_atomic.c Added [PLANE:74:plane-11] 00000000061a9b60 state to 00000000b87b3dfb
[2020/10/16 11:06:32] [    1.727944] drm_atomic.c Added [PLANE:75:plane-12] 00000000d6caa386 state to 00000000b87b3dfb
[2020/10/16 11:06:32] [    1.727952] drm_atomic.c Added [PLANE:76:plane-13] 00000000c4021e88 state to 00000000b87b3dfb
[2020/10/16 11:06:32] [    1.727973] drm_atomic.c Added [PLANE:77:plane-14] 000000004cf69062 state to 00000000b87b3dfb
[2020/10/16 11:06:32] [    1.727980] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.727980] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.727980] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.727993] drm_atomic.c Added [CONNECTOR:79:DP-1] 0000000026eec589 state to 00000000b87b3dfb
[2020/10/16 11:06:32] [    1.728000] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.728000] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.728000] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.728011] drm_atomic.c Added [CONNECTOR:81:HDMI-A-1] 00000000de264a03 state to 00000000b87b3dfb
[2020/10/16 11:06:32] [    1.728029] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.728029] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.728029] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.728044] drm_atomic.c Added [CONNECTOR:89:DSI-1] 0000000084587423 state to 00000000b87b3dfb
[2020/10/16 11:06:32] [    1.728060] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c drm_atomic_helper_duplicate_state 3924 
[2020/10/16 11:06:32] [    1.728060] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c drm_atomic_helper_duplicate_state 3924 
[2020/10/16 11:06:32] [    1.728060] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c drm_atomic_helper_duplicate_state 3924 
[2020/10/16 11:06:32] [    1.728067] drm_atomic.c Allocated atomic state 00000000c88c6186
[2020/10/16 11:06:32] [    1.728082] drm_atomic.c Added [CRTC:56:crtc-0] 0000000008121677 state to 00000000c88c6186
[2020/10/16 11:06:32] [    1.728098] drm_atomic.c Added [CRTC:67:crtc-1] 000000002b0c8bf1 state to 00000000c88c6186
[2020/10/16 11:06:32] [    1.728106] drm_atomic.c Added [PLANE:55:plane-0] 0000000039f71813 state to 00000000c88c6186
[2020/10/16 11:06:32] [    1.728126] drm_atomic.c Added [PLANE:57:plane-1] 0000000070a9f9db state to 00000000c88c6186
[2020/10/16 11:06:32] [    1.728134] drm_atomic.c Added [PLANE:59:plane-2] 000000006a92117f state to 00000000c88c6186
[2020/10/16 11:06:32] [    1.728155] drm_atomic.c Added [PLANE:60:plane-3] 000000006e1d5fc6 state to 00000000c88c6186
[2020/10/16 11:06:32] [    1.728163] drm_atomic.c Added [PLANE:61:plane-4] 00000000ea58a29a state to 00000000c88c6186
[2020/10/16 11:06:32] [    1.728173] drm_atomic.c Added [PLANE:65:plane-5] 00000000a68c96ce state to 00000000c88c6186
[2020/10/16 11:06:32] [    1.728191] drm_atomic.c Added [PLANE:66:plane-6] 0000000087db4117 state to 00000000c88c6186
[2020/10/16 11:06:32] [    1.728199] drm_atomic.c Added [PLANE:68:plane-7] 0000000020b2e27e state to 00000000c88c6186
[2020/10/16 11:06:32] [    1.728209] drm_atomic.c Added [PLANE:70:plane-8] 00000000b4bb618e state to 00000000c88c6186
[2020/10/16 11:06:32] [    1.728227] drm_atomic.c Added [PLANE:72:plane-9] 0000000091127c4f state to 00000000c88c6186
[2020/10/16 11:06:32] [    1.728236] drm_atomic.c Added [PLANE:73:plane-10] 00000000c7db7054 state to 00000000c88c6186
[2020/10/16 11:06:32] [    1.728247] drm_atomic.c Added [PLANE:74:plane-11] 000000003230cbca state to 00000000c88c6186
[2020/10/16 11:06:32] [    1.728254] drm_atomic.c Added [PLANE:75:plane-12] 00000000af1e62b6 state to 00000000c88c6186
[2020/10/16 11:06:32] [    1.728262] drm_atomic.c Added [PLANE:76:plane-13] 0000000024a4b36e state to 00000000c88c6186
[2020/10/16 11:06:32] [    1.728283] drm_atomic.c Added [PLANE:77:plane-14] 000000001f0bb2f1 state to 00000000c88c6186
[2020/10/16 11:06:32] [    1.728290] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.728290] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.728290] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.728305] drm_atomic.c Added [CONNECTOR:79:DP-1] 00000000ab46bd15 state to 00000000c88c6186
[2020/10/16 11:06:32] [    1.728320] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.728320] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.728320] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.728331] drm_atomic.c Added [CONNECTOR:81:HDMI-A-1] 000000004b6a594d state to 00000000c88c6186
[2020/10/16 11:06:32] [    1.728349] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.728349] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.728349] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.728364] drm_atomic.c Added [CONNECTOR:89:DSI-1] 0000000030c8c428 state to 00000000c88c6186
[2020/10/16 11:06:32] [    1.728379] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.728379] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.728379] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.728387] zjj.rk3399.kernel drivers/gpu/drm/rockchip/dw-mipi-dsi.c dw_mipi_dsi_encoder_atomic_check 1340 
[2020/10/16 11:06:32] [    1.728408] zjj.rk3399.kernel drivers/gpu/drm/rockchip/dw-mipi-dsi.c dw_mipi_dsi_encoder_atomic_mode_set 1382 
[2020/10/16 11:06:32] [    1.728443] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_commit 2019 
[2020/10/16 11:06:32] [    1.728443] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_commit 2019 
[2020/10/16 11:06:32] [    1.728443] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_commit 2019 
[2020/10/16 11:06:32] [    1.728452] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_check_only 1946 
[2020/10/16 11:06:32] [    1.728452] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_check_only 1946 
[2020/10/16 11:06:32] [    1.728452] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_check_only 1946 
[2020/10/16 11:06:32] [    1.728478] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c drm_atomic_helper_check 916 
[2020/10/16 11:06:32] [    1.728478] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c drm_atomic_helper_check 916 
[2020/10/16 11:06:32] [    1.728478] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c drm_atomic_helper_check 916 
[2020/10/16 11:06:32] [    1.728485] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c drm_atomic_helper_check_modeset 597 
[2020/10/16 11:06:32] [    1.728485] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c drm_atomic_helper_check_modeset 597 
[2020/10/16 11:06:32] [    1.728485] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c drm_atomic_helper_check_modeset 597 
[2020/10/16 11:06:32] [    1.728494] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c update_connector_routing 287 
[2020/10/16 11:06:32] [    1.728494] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c update_connector_routing 287 
[2020/10/16 11:06:32] [    1.728515] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c update_connector_routing 287 
[2020/10/16 11:06:32] [    1.728515] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c update_connector_routing 287 
[2020/10/16 11:06:32] [    1.728544] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c update_connector_routing 287 
[2020/10/16 11:06:32] [    1.728544] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c update_connector_routing 287 
[2020/10/16 11:06:32] [    1.728580] zjj.rk3399.kernel drivers/gpu/drm/rockchip/dw-mipi-dsi.c dw_mipi_dsi_encoder_atomic_check 1340 
[2020/10/16 11:06:32] [    1.728597] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.728597] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.728608] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_fb_is_logo 34 
[2020/10/16 11:06:32] [    1.728608] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_fb_is_logo 34 
[2020/10/16 11:06:32] [    1.728624] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_fb_get_dma_addr 43 
[2020/10/16 11:06:32] [    1.728624] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_fb_get_dma_addr 43 
[2020/10/16 11:06:32] [    1.728631] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_fb_get_kvaddr 54 
[2020/10/16 11:06:32] [    1.728631] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_fb_get_kvaddr 54 
[2020/10/16 11:06:32] [    1.728644] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.728644] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.728663] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.728663] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.728670] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.728670] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.728687] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.728687] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.728694] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.728694] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.728700] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.728700] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.728715] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.728715] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.728721] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.728721] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.728728] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.728728] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.728748] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.728748] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.728754] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.728754] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.728761] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.728761] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.728767] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.728767] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.728774] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.728774] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/16 11:06:32] [    1.728795] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_crtc_atomic_check 3212 
[2020/10/16 11:06:32] [    1.728795] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_crtc_atomic_check 3212 
[2020/10/16 11:06:32] [    1.728806] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_crtc_atomic_check 3212 
[2020/10/16 11:06:32] [    1.728806] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_crtc_atomic_check 3212 
[2020/10/16 11:06:32] [    1.728835] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_drm_atomic_commit 470 
[2020/10/16 11:06:32] [    1.728835] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_drm_atomic_commit 470 
[2020/10/16 11:06:32] [    1.728835] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_drm_atomic_commit 470 
[2020/10/16 11:06:32] [    1.728868] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_atomic_commit_complete 400 
[2020/10/16 11:06:32] [    1.728868] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_atomic_commit_complete 400 
[2020/10/16 11:06:32] [    1.728905] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_update 1758 
[2020/10/16 11:06:32] [    1.728905] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_update 1758 
[2020/10/16 11:06:32] [    1.728933] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_crtc_atomic_flush 3558 
[2020/10/16 11:06:32] [    1.728933] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_crtc_atomic_flush 3558 
[2020/10/16 11:06:32] [    1.728949] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_cfg_update 3485 
[2020/10/16 11:06:32] [    1.728949] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_cfg_update 3485 
[2020/10/16 11:06:32] [    1.728961] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_tv_config_update 3397 
[2020/10/16 11:06:32] [    1.728961] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_tv_config_update 3397 
[2020/10/16 11:06:32] [    1.766290] drm_atomic.c Clearing atomic state 00000000b87b3dfb
[2020/10/16 11:06:32] [    1.766333] drm_atomic.c Freeing atomic state 00000000b87b3dfb
[2020/10/16 11:06:32] [    1.766339] drm_atomic.c Clearing atomic state 00000000c88c6186
[2020/10/16 11:06:32] [    1.766366] drm_atomic.c Freeing atomic state 00000000c88c6186
[2020/10/16 11:06:32] [    1.766383] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fbdev.c rockchip_drm_fbdev_init 173 
[2020/10/16 11:06:32] [    1.766383] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fbdev.c rockchip_drm_fbdev_init 173 
[2020/10/16 11:06:32] [    1.766383] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fbdev.c rockchip_drm_fbdev_init 173 
[2020/10/16 11:06:32] [    1.766417] zjj.rk3399.kernel drivers/gpu/drm/drm_fb_helper.c drm_fb_helper_init 864 
[2020/10/16 11:06:32] [    1.766417] zjj.rk3399.kernel drivers/gpu/drm/drm_fb_helper.c drm_fb_helper_init 864 
[2020/10/16 11:06:32] [    1.766417] zjj.rk3399.kernel drivers/gpu/drm/drm_fb_helper.c drm_fb_helper_init 864 
[2020/10/16 11:06:32] [    1.766486] zjj.rk3399.kernel drivers/gpu/drm/panel/panel-simple.c panel_simple_get_modes 629 
[2020/10/16 11:06:32] [    1.766498] zjj.rk3399.kernel drivers/gpu/drm/panel/panel-simple.c panel_simple_get_fixed_modes 373 
[2020/10/16 11:06:32] [    1.766590] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fbdev.c rockchip_drm_fbdev_create 97 
[2020/10/16 11:06:32] [    1.766590] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fbdev.c rockchip_drm_fbdev_create 97 
[2020/10/16 11:06:32] [    1.766590] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fbdev.c rockchip_drm_fbdev_create 97 
[2020/10/16 11:06:32] [    1.773850] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_drm_framebuffer_init 531 
[2020/10/16 11:06:32] [    1.773850] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_drm_framebuffer_init 531 
[2020/10/16 11:06:32] [    1.773850] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_drm_framebuffer_init 531 
[2020/10/16 11:06:32] [    1.773870] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_fb_alloc 125 
[2020/10/16 11:06:32] [    1.773870] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_fb_alloc 125 
[2020/10/16 11:06:32] [    1.773878] zjj.rk3399.kernel drivers/gpu/drm/drm_framebuffer.c drm_framebuffer_init 691 
[2020/10/16 11:06:32] [    1.773878] zjj.rk3399.kernel drivers/gpu/drm/drm_framebuffer.c drm_framebuffer_init 691 
[2020/10/16 11:06:32] [    1.773878] zjj.rk3399.kernel drivers/gpu/drm/drm_framebuffer.c drm_framebuffer_init 691 
[2020/10/16 11:06:32] [    1.774080] zjj.rk3399.kernel drivers/gpu/drm/panel/panel-simple.c panel_simple_get_modes 629 
[2020/10/16 11:06:32] [    1.774087] zjj.rk3399.kernel drivers/gpu/drm/panel/panel-simple.c panel_simple_get_fixed_modes 373 
[2020/10/16 11:06:32] [    1.774205] zjj.rk3399.kernel drivers/gpu/drm/drm_fb_helper.c restore_fbdev_mode_atomic 398 
[2020/10/16 11:06:32] [    1.774205] zjj.rk3399.kernel drivers/gpu/drm/drm_fb_helper.c restore_fbdev_mode_atomic 398 
[2020/10/16 11:06:32] [    1.774214] drm_atomic.c Allocated atomic state 0000000042af48a7
[2020/10/16 11:06:32] [    1.774220] zjj.rk3399.kernel retry drivers/gpu/drm/drm_fb_helper.c restore_fbdev_mode_atomic 410 
[2020/10/16 11:06:32] [    1.774220] zjj.rk3399.kernel retry drivers/gpu/drm/drm_fb_helper.c restore_fbdev_mode_atomic 410 
[2020/10/16 11:06:32] [    1.774250] drm_atomic.c Added [PLANE:55:plane-0] 000000002d911d74 state to 0000000042af48a7
[2020/10/16 11:06:32] [    1.774263] drm_atomic.c Added [CRTC:56:crtc-0] 00000000861a7111 state to 0000000042af48a7
[2020/10/16 11:06:32] [    1.774279] drm_atomic.c Added [PLANE:57:plane-1] 000000006df95373 state to 0000000042af48a7
[2020/10/16 11:06:32] [    1.774293] drm_atomic.c Added [PLANE:59:plane-2] 00000000122a95dd state to 0000000042af48a7
[2020/10/16 11:06:32] [    1.774305] drm_atomic.c Added [PLANE:60:plane-3] 00000000c5ac98a6 state to 0000000042af48a7
[2020/10/16 11:06:32] [    1.774318] drm_atomic.c Added [PLANE:61:plane-4] 0000000056e69e01 state to 0000000042af48a7
[2020/10/16 11:06:32] [    1.774344] drm_atomic.c Added [PLANE:65:plane-5] 000000005156e610 state to 0000000042af48a7
[2020/10/16 11:06:32] [    1.774352] drm_atomic.c Added [PLANE:66:plane-6] 00000000d7d155b4 state to 0000000042af48a7
[2020/10/16 11:06:32] [    1.774377] drm_atomic.c Added [PLANE:68:plane-7] 000000002a4fb1ba state to 0000000042af48a7
[2020/10/16 11:06:32] [    1.774390] drm_atomic.c Added [PLANE:70:plane-8] 000000003e149226 state to 0000000042af48a7
[2020/10/16 11:06:32] [    1.774410] drm_atomic.c Added [PLANE:72:plane-9] 000000000316616c state to 0000000042af48a7
[2020/10/16 11:06:32] [    1.774433] drm_atomic.c Added [PLANE:73:plane-10] 0000000008eb7b42 state to 0000000042af48a7
[2020/10/16 11:06:32] [    1.774447] drm_atomic.c Added [PLANE:74:plane-11] 00000000b5f94e98 state to 0000000042af48a7
[2020/10/16 11:06:32] [    1.774460] drm_atomic.c Added [PLANE:75:plane-12] 000000006f806d14 state to 0000000042af48a7
[2020/10/16 11:06:32] [    1.774472] drm_atomic.c Added [PLANE:76:plane-13] 0000000080e2c4f7 state to 0000000042af48a7
[2020/10/16 11:06:32] [    1.774484] drm_atomic.c Added [PLANE:77:plane-14] 0000000093e0dae3 state to 0000000042af48a7
[2020/10/16 11:06:32] [    1.774505] drm_atomic.c Set [MODE:1088x1920] for [CRTC:56:crtc-0] state 00000000861a7111
[2020/10/16 11:06:32] [    1.774535] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_add_affected_connectors 1848 
[2020/10/16 11:06:32] [    1.774535] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_add_affected_connectors 1848 
[2020/10/16 11:06:32] [    1.774535] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_add_affected_connectors 1848 
[2020/10/16 11:06:32] [    1.774557] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.774557] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.774557] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.774573] drm_atomic.c Added [CONNECTOR:89:DSI-1] 0000000026eec589 state to 0000000042af48a7
[2020/10/16 11:06:32] [    1.774594] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.774594] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.774594] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/16 11:06:32] [    1.774613] drm_atomic.c Added [CRTC:67:crtc-1] 0000000096d7e41a state to 0000000042af48a7
[2020/10/16 11:06:32] [    1.774619] drm_atomic.c Set [NOMODE] for [CRTC:67:crtc-1] state 0000000096d7e41a
[2020/10/16 11:06:32] [    1.774630] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_add_affected_connectors 1848 
[2020/10/16 11:06:32] [    1.774630] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_add_affected_connectors 1848 
[2020/10/16 11:06:32] [    1.774630] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_add_affected_connectors 1848 
[2020/10/16 11:06:32] [    1.774642] drm_atomic.c Clearing atomic state 0000000042af48a7
[2020/10/16 11:06:32] [    1.774669] drm_atomic.c Freeing atomic state 0000000042af48a7
[2020/10/16 11:06:32] [    1.774729] rockchip-drm display-subsystem: fb0:  frame buffer device
[2020/10/16 11:06:32] [    1.774827] zjj.rk3399.kernel drivers/gpu/drm/drm_drv.c drm_dev_register 820 
[2020/10/16 11:06:32] [    1.774827] zjj.rk3399.kernel drivers/gpu/drm/drm_drv.c drm_dev_register 820 
[2020/10/16 11:06:32] [    1.774827] zjj.rk3399.kernel drivers/gpu/drm/drm_drv.c drm_dev_register 820 
[2020/10/16 11:06:32] [    1.775481] zjj.rk3399.kernel drivers/gpu/drm/drm_encoder.c drm_encoder_register_all 70 
[2020/10/16 11:06:32] [    1.775481] zjj.rk3399.kernel drivers/gpu/drm/drm_encoder.c drm_encoder_register_all 70 
[2020/10/16 11:06:32] [    1.775481] zjj.rk3399.kernel drivers/gpu/drm/drm_encoder.c drm_encoder_register_all 70 
[2020/10/16 11:06:33] [    2.729943] cdn-dp fec00000.dp: [drm:cdn_dp_pd_event_work] Not connected. Disabling cdn



## （四）、参考资料(特别感谢)：
[（1）【DRM（Direct Rendering Manager）学习简介】](https://blog.csdn.net/hexiaolong2009/article/details/83720940)

[（2）【The DRM/KMS subsystem from a newbie’s point of view】](https://events.static.linuxfound.org/sites/events/files/slides/brezillon-drm-kms.pdf)

[（3）【Anatomy of an Atomic KMS Driver】](https://elinux.orghttps://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/4/45/Atomic_kms_driver_pinchart.pdf)

[（4）【Why and How to use KMS】](https://events.static.linuxfound.org/sites/events/files/lcjpcojp13_pinchart.pdf)