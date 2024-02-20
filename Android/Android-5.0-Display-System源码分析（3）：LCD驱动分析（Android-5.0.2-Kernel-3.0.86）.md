---
title: Android L Display System源码分析（3）：LCD驱动分析（Android 5.0.2 && Kernel 3.0.86）
cover: https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/personal.website/post.cover.pictures.00003.jpg
categories: 
  - Display
tags:
  - Android
  - Linux
  - Graphics
toc: true
abbrlink: 20191001
date: 2019-10-01 09:25:00
---


注：文章都是通过阅读 Android  && Linux 平台源码、各位前辈总结的资料、加上自己的思考分析总结出来的，其中难免有理不对的地方，欢迎大家批评指正。**文章为个人学习、研究、欣赏之用。图文内容、源码整理自互联网，如有侵权，请联系删除（◕‿◕）**，转载请注明出处（ ©Android @Linux 版权所有），谢谢(๑乛◡乛๑) 、（ ͡° ͜ʖ ͡°）、（ಡωಡ）！！！。
（==**文章基于 Kernel-3.0.86**==）&&（==**文章基于 Android 5.0.2**==）
[【开发板 - 友善之臂 FriendlyARM Cortex-A9 Tiny4412 ADK Exynos4412 （ Android 5.0.2）HD702高清电容屏 扩展套餐】](https://item.taobao.com/item.htm?spm=0.0.0.0.3GsGdQ&id=20131438062)
[【开发板 Android 5.0.2 && Kernel 3.0.86 源码链接： https://pan.baidu.com/s/1jJHm74q 密码：yfih】](https://pan.baidu.com/s/1jJHm74q)

正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢！！！

[（1）【LCD设备驱动框架分析（数据结构）】](https://blog.csdn.net/z961968549/article/details/78821082)
[（2）【framebuffer之s3cfb_probe分析】](https://blog.csdn.net/Ultraman_hs/article/details/54987874)
[（3）【LCD驱动框架】](https://layty.gitee.io/2018/12/03/Linux/%E8%AE%BE%E5%A4%87%E9%A9%B1%E5%8A%A8/12-lcd%E9%A9%B1%E5%8A%A8%E6%A1%86%E6%9E%B6/)

--------------------------------------------------------------------------------
==源码（部分）==：

> linux-3.0.86（Kernel 层）： FrameBuffer 核心层

- G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\drivers\video\fbmem.c
- G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\include\linux\fb.h

> 设备驱动层：

- G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\drivers\video\samsung\s3cfb_main.c
- G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\drivers\video\samsung\s3cfb_ops.c
- G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\drivers\video\samsung\s3cfb_fimd6x.c
- G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\drivers\video\cfbimgblt.c

> 设备平台相关：

- G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\arch\arm\plat-s5p\include\plat\regs-fb-s5p.h
- G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\arch\arm\plat-s5p\include\plat\fb-s5p.h
- G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\arch\arm\mach-exynos\setup-fb-s5p.c
- G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\arch\arm\mach-exynos\tiny4412-lcds.c
- G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\arch\arm\mach-exynos\mach-tiny4412.c

--------------------------------------------------------------------------------

####  （一）、LCD设备驱动框架图
![enter image description here](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/personal.website/zjj.display.sysfbmem_s3cfb_main.png)

- 核心层是通用的，不需要任何修改。驱动开发者只需要实现硬件设备驱动层
- 帧缓冲设备可以看做是一个完整的子系统，主要由核心层的fbmem.c和硬件设备驱动层构成
- 核心层代码fnmem.c向上提供了完整的字符设备操作接口，也就是实现注册字符设备，提供通用的open,read,write,ioctl,mmap等接口；向下给硬件设备驱动层提供标准的驱动编程接口；
- 在linux系统中，一个硬件控制器（显卡）抽象为一个fb_info结构，要实现一个LCD驱动就是要实现这个结构，并且使用核心层提供的注册函数注册
- fb_info中通过其中的fb_ops结构指针提供了实际硬件操作方法
- fb_info中通过其中的fb_var_screeninfo结构和fb_fix_screeninfo结构提供了具体lcd屏基本信息
- 注册：register_framebuffer
- 注销：unregister_framebuffer

##### （1）、Framebuffer之fbmem.c概括
LCD框架的fbmem.c已经帮我们完成了日常驱动程序的工作，如：注册字符设备、分配设备号、创建类等。
也有了操作函数fb_fops，但它只是一个框架，在具体执行的时候需要知道硬件具体的一些参数，比如分辨率、数据基地址等信息。
因此，我们要利用这一框架，就得构造一个fb_info结构体，完成硬件初始化，设置相关参数等操作，再使用register_framebuffer()将fb_info向上注册。
这样，fbmem.c就可以从registered_fb[]这个数组获得fb_info参数，进行相关的硬件操作。
比如：应用层app想read()，就会调用fbmem.c的fb_read()，在fb_read里面会先尝试使用xxfb.c提供的read()操作函数，如果没有，再根据fb_info
信息得到数据基地址，将基地址开始的数据，返回给应用层，实现读操作。

####  （二）、platform_device && platform_driver分析

##### （1）、platform_device分析
> Log:
> <4>[    0.180000] zjj.android.display.sys tiny4412_get_lcd() march-tiny4412.c 
> <4>[    0.180000] zjj.android.display.sys s3cfb_set_platdata() march-tiny4412.c 

``` cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\arch\arm\mach-exynos\mach-tiny4412.c

static void __init smdk4x12_machine_init(void)
{
	struct s3cfb_lcd *lcd;
	......
	tiny4412_fb_data.lcd = tiny4412_get_lcd();
	s3cfb_set_platdata(&tiny4412_fb_data);
	......
}

G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\arch\arm\mach-exynos\tiny4412-lcds.c

static struct s3cfb_lcd wxga_hd700 = {
	.width = 800,
	.height = 1280,
	.p_width = 94,
	.p_height = 151,
	.bpp = 24,
	.freq = 60,

	.timing = {
		.h_fp = 20,
		.h_bp = 20,
		.h_sw = 24,
		.v_fp =  4,
		.v_fpe = 1,//rgb == 0
		.v_bp =  4,
		.v_bpe = 1,//rgb == 0
		.v_sw =  8,
	},
	.polarity = {
		.rise_vclk = 1,
		.inv_hsync = 0,
		.inv_vsync = 0,
		.inv_vden = 0,
	},
};

static struct {
	char *name;
	struct s3cfb_lcd *lcd;
	int ctp;
} tiny4412_lcd_config[] = {
	{ "HD700",	&wxga_hd700, 1 },
	{ "HD702",	&wxga_hd700, CTP_GOODIX  },
	{ "S70",	&wvga_s70,   1 },
	{ "S702",	&wvga_s70,   1 },
	{ "S70D",	&wvga_s70d,  1 },
	{ "W50",	&wvga_w50,   0 },
	{ "W101",	&wsvga_w101, 1 },
	{ "X710",	&wsvga_x710, CTP_ITE7260 },
	{ "A97",	&xga_a97,    0 },
	{ "LQ150",	&xga_lq150,  1 },
	{ "L80",	&vga_l80,    1 },
	{ "HD101",	&wxga_hd101, 1 },
	{ "HD101B",	&wxga_hd101, CTP_GOODIX  },
	{ "BP101",	&wxga_bp101, 1 },
	{ "HDM",	&hdmi_def,   0 },	/* Pls keep it at last */
};

static int lcd_idx = 0;

static int __init tiny4412_setup_lcd(char *str)
{
	char *delim;
	int i;

	delim = strchr(str, ',');
	if (delim)
		*delim++ = '\0';

	if (!strncasecmp("HDMI", str, 4)) {
		struct hdmi_config *cfg = &tiny4412_hdmi_config[0];
		struct s3cfb_lcd *lcd;

		lcd_idx = ARRAY_SIZE(tiny4412_lcd_config) - 1;
		lcd = tiny4412_lcd_config[lcd_idx].lcd;

		for (i = 0; i < ARRAY_SIZE(tiny4412_hdmi_config); i++, cfg++) {
			if (!strcasecmp(cfg->name, str)) {
				lcd->width = cfg->width;
				lcd->height = cfg->height;
				goto __ret;
			}
		}
	}

	for (i = 0; i < ARRAY_SIZE(tiny4412_lcd_config); i++) {
		if (!strcasecmp(tiny4412_lcd_config[i].name, str)) {
			lcd_idx = i;
			break;
		}
	}

__ret:
	tiny4412_set_ctp(tiny4412_lcd_config[lcd_idx].ctp);

	printk("TINY4412: %s selected\n", tiny4412_lcd_config[lcd_idx].name);
	return 0;
}
early_param("lcd", tiny4412_setup_lcd);


struct s3cfb_lcd *tiny4412_get_lcd(void)
{
	return tiny4412_lcd_config[lcd_idx].lcd;
}

```
根据打印Log，可以知道刚开机选择了LCD HD702

> Log:
> <4>[    0.000000] Machine: TINY4412
> <4>[    0.000000] TINY4412: HD702 selected

通过s3cfb_set_platdata这个函数来单独设置platdata。因为内核中实现了多个LCD设备的platdata，用户可通过相应的宏定义来选择设置。它在启动时通过smdk4x12_machine_init()函数加载，保证platdata在内核启动时就已经设置好。

``` cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\arch\arm\plat-s5p\dev-fimd-s5p.c

void __init s3cfb_set_platdata(struct s3c_platform_fb *pd)
{
	struct s3c_platform_fb *npd;
	int i;

	if (!pd)
		pd = &default_fb_data;

	npd = kmemdup(pd, sizeof(struct s3c_platform_fb), GFP_KERNEL);
	if (!npd)
		printk(KERN_ERR "%s: no memory for platform data\n", __func__);
	else {
		for (i = 0; i < npd->nr_wins; i++)
			npd->nr_buffers[i] = 1;

#if defined(CONFIG_FB_S5P_NR_BUFFERS)
		npd->nr_buffers[npd->default_win] = CONFIG_FB_S5P_NR_BUFFERS;
#else
		npd->nr_buffers[npd->default_win] = 1;
#endif

		s3cfb_get_clk_name(npd->clk_name);
		npd->set_display_path = s3cfb_set_display_path;
		npd->cfg_gpio = s3cfb_cfg_gpio;
		npd->backlight_on = s3cfb_backlight_on;
		npd->backlight_off = s3cfb_backlight_off;
		npd->lcd_on = s3cfb_lcd_on;
		npd->lcd_off = s3cfb_lcd_off;
		npd->clk_on = s3cfb_clk_on;
		npd->clk_off = s3cfb_clk_off;

		s3c_device_fb.dev.platform_data = npd;
	}
}
#endif
```
s3c_device_fb的设备信息如下：

``` cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\arch\arm\plat-s5p\dev-fimd-s5p.c

static struct resource s3cfb_resource[] = {
	[0] = {
		.start	= S5P_PA_FIMD0,
		.end	= S5P_PA_FIMD0 + SZ_32K - 1,
		.flags	= IORESOURCE_MEM,
	},
	[1] = {
		.start	= IRQ_FIMD0_VSYNC,
		.end	= IRQ_FIMD0_VSYNC,
		.flags	= IORESOURCE_IRQ,
	},
	[2] = {
		.start	= IRQ_FIMD0_FIFO,
		.end	= IRQ_FIMD0_FIFO,
		.flags	= IORESOURCE_IRQ,
	},
};

static u64 fb_dma_mask = 0xffffffffUL;

struct platform_device s3c_device_fb = {
	.name		= "s3cfb",
#if defined(CONFIG_ARCH_EXYNOS4)
	.id		= 0,
#else
	.id		= -1,
#endif
	.num_resources	= ARRAY_SIZE(s3cfb_resource),
	.resource	= s3cfb_resource,
	.dev		= {
		.dma_mask		= &fb_dma_mask,
		.coherent_dma_mask	= 0xffffffffUL
	}
};

G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\drivers\video\samsung\s3cfb_main.c	
static struct platform_driver s3cfb_driver = {
	.probe		= s3cfb_probe,
	.remove		= s3cfb_remove,
#ifndef CONFIG_HAS_EARLYSUSPEND
	.suspend	= s3cfb_suspend,
	.resume		= s3cfb_resume,
#endif
	.driver		= {
		.name	= S3CFB_NAME,//s3cfb
		.owner	= THIS_MODULE,
#ifdef CONFIG_EXYNOS_DEV_PD
		.pm	= &s3cfb_pm_ops,
#endif
	},
};
```

##### （2）、platform_driver分析
> tiny4412-lcds.c ：设备参数相关文件 
> s3cfb_fimd6x.c：具体平台硬件相关文件 
> s3cfb_ops.c：fb操作集合 
> s3cfb_main.c ：驱动框架 

s3cfb_main.c中的s3cfb_probe设备探测，是驱动注册的主要函数 


##### 2.1、s3cfb驱动注册

``` cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\drivers\video\samsung\s3cfb_main.c

static struct platform_driver s3cfb_driver = {
	.probe		= s3cfb_probe,
	.remove		= s3cfb_remove,
#ifndef CONFIG_HAS_EARLYSUSPEND
	.suspend	= s3cfb_suspend,
	.resume		= s3cfb_resume,
#endif
	.driver		= {
		.name	= S3CFB_NAME,
		.owner	= THIS_MODULE,
#ifdef CONFIG_EXYNOS_DEV_PD
		.pm	= &s3cfb_pm_ops,
#endif
	},
};

struct fb_ops s3cfb_ops = {
	.owner		= THIS_MODULE,
	.fb_open	= s3cfb_open,
	.fb_release	= s3cfb_release,
	.fb_check_var	= s3cfb_check_var,
	.fb_set_par	= s3cfb_set_par,
	.fb_setcolreg	= s3cfb_setcolreg,
	.fb_blank	= s3cfb_blank,
	.fb_pan_display	= s3cfb_pan_display,
	.fb_fillrect	= cfb_fillrect,
	.fb_copyarea	= cfb_copyarea,
	.fb_imageblit	= cfb_imageblit,
	.fb_cursor	= s3cfb_cursor,
	.fb_ioctl	= s3cfb_ioctl,
};

static int s3cfb_register(void)
{
	return platform_driver_register(&s3cfb_driver);
}

static void s3cfb_unregister(void)
{
	platform_driver_unregister(&s3cfb_driver);
}

module_init(s3cfb_register);
module_exit(s3cfb_unregister);
```
#### （三）、s3cfb_probe函数分析
先通过Log整体看看流程：

> Log：
> 	Line 215: <4>[    0.180000] zjj.android.display.sys tiny4412_get_lcd() march-tiny4412.c 
	Line 217: <4>[    0.180000] zjj.android.display.sys s3cfb_set_platdata() march-tiny4412.c 
	Line 389: <4>[    0.432365] zjj.android.display.sys cfg_gpio() s3cfb_probe s3cfb_main.c 
	Line 391: <6>[    0.432476] s3cfb s3cfb.0: src_clk=800000000, vclk=67184640, div=12(11), rate=72727272
	Line 393: <6>[    0.432488] s3cfb s3cfb.0: fimd sclk rate 66666666, clkdiv 0xfffffb
	Line 395: <4>[    0.432576] zjj.android.display.sys s3cfb_init_global() s3cfb_probe s3cfb_main.c
	Line 397: <4>[    0.432583] zjj.android.display.sys s3cfb_set_output s3cfb_fimd6x.c 
	Line 399: <4>[    0.432589] zjj.android.display.sys s3cfb_set_output s3cfb_fimd6x.c S3C_VIDCON0 = 0
	Line 401: <4>[    0.432595] zjj.android.display.sys s3cfb_set_output s3cfb_fimd6x.c S3C_VIDCON2 = 0
	Line 403: <4>[    0.432600] zjj.android.display.sys s3cfb_set_display_mode s3cfb_fimd6x.c 
	Line 405: <4>[    0.432606] zjj.android.display.sys s3cfb_set_display_mode s3cfb_fimd6x.c S3C_VIDCON0 = 0
	Line 407: <4>[    0.432611] zjj.android.display.sys s3cfb_set_polarity s3cfb_fimd6x.c 
	Line 409: <4>[    0.432617] zjj.android.display.sys s3cfb_set_polarity s3cfb_fimd6x.c S3C_VIDCON1 = 680
	Line 411: <4>[    0.432622] zjj.android.display.sys s3cfb_set_timing s3cfb_fimd6x.c 
	Line 413: <4>[    0.432627] zjj.android.display.sys s3cfb_set_timing s3cfb_fimd6x.c  S3C_VIDTCON0 = 30307 
	Line 415: <4>[    0.432634] zjj.android.display.sys s3cfb_set_timing s3cfb_fimd6x.c  S3C_VIDTCON1 = 131317 
	Line 417: <4>[    0.432639] zjj.android.display.sys s3cfb_set_lcd_size s3cfb_fimd6x.c  S3C_VIDTCON2 = 27fb1f 
	Line 419: <4>[    0.432645] zjj.android.display.sys s3cfb_alloc_framebuffer() s3cfb_probe s3cfb_main.c
	Line 421: <4>[    0.432651] zjj.android.display.sys s3cfb_alloc_framebuffer() s3cfb_ops.c 
	Line 423: <6>[    0.432860] s3cfb s3cfb.0: [fb0] win2: dma: 0x60ad8000, cpu: 0xe4879000, size: 0x00bb8000
	Line 425: <4>[    0.448361] zjj.android.display.sys s3cfb_register_framebuffer() s3cfb_probe s3cfb_main.c
	Line 425: <4>[    0.448361] zjj.android.display.sys s3cfb_register_framebuffer() s3cfb_probe s3cfb_main.c
	Line 427: <4>[    0.448606] zjj.android.display.sys s3cfb_draw_logo() s3cfb_ops.c
	Line 439: <4>[    0.449600] zjj.android.display.sys s3cfb_set_clock() s3cfb_probe s3cfb_main.c
	Line 441: <4>[    0.449607] zjj.android.display.sys s3cfb_set_clock s3cfb_fimd6x.c 
	Line 443: <6>[    0.449616] s3cfb s3cfb.0: FIMD src sclk = 66666666
	Line 445: <4>[    0.449622] zjj.android.display.sys s3cfb_set_clock s3cfb_fimd6x.c S3C_VIDCON0 = 20
	Line 447: <6>[    0.449629] s3cfb s3cfb.0: parent clock: 66666666, vclk: 67186000, vclk div: 1
	Line 449: <4>[    0.449635] zjj.android.display.sys s3cfb_enable_window() s3cfb_probe s3cfb_main.c
	Line 451: <4>[    0.449641] zjj.android.display.sys s3cfb_window_on s3cfb_fimd6x.c
	Line 453: <6>[    0.449647] s3cfb s3cfb.0: [win2] turn on
	Line 455: <4>[    0.449651] zjj.android.display.sys s3cfb_update_power_state() s3cfb_probe s3cfb_main.c
	Line 457: <4>[    0.449657] zjj.android.display.sys s3cfb_display_on s3cfb_fimd6x.c
	Line 459: <4>[    0.449662] zjj.android.display.sys s3cfb_set_display_on s3cfb_fimd6x.c S3C_VIDCON0 = 23
	Line 461: <6>[    0.449668] s3cfb s3cfb.0: global display is on
	Line 463: <6>[    0.449741] s3cfb s3cfb.0: registered successfully
	Line 465: <6>[    0.450044] s3cfb_extdsp s3cfb_extdsp.0: registered successfully
	Line 1959: <4>[    9.365483] zjj.android.display.sys s3cfb_open() s3cfb_ops.c
	Line 1961: <4>[    9.365606] zjj.android.display.sys s3cfb_open() s3cfb_ops.c
	Line 1979: <4>[    9.555995] zjj.android.display.sys s3cfb_open() s3cfb_ops.c


s3cfb_probe()函数如下：
``` cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\drivers\video\samsung\s3cfb_main.c

static int s3cfb_probe(struct platform_device *pdev)
{
	struct s3c_platform_fb *pdata = NULL; /*LCD的platdata，LCD屏配置信息结构体*/
	struct resource *res = NULL;
	struct s3cfb_global *fbdev[2];
	int ret = 0;
	int i = 0;

#ifdef CONFIG_EXYNOS_DEV_PD
	/* to use the runtime PM helper functions */
	pm_runtime_enable(&pdev->dev);
	/* enable the power domain */
	pm_runtime_get_sync(&pdev->dev);
#endif

	fbfimd = kzalloc(sizeof(struct s3cfb_fimd_desc), GFP_KERNEL);
	......

	if (FIMD_MAX == 2)
		fbfimd->dual = 1;
	else
		fbfimd->dual = 0;

	for (i = 0; i < FIMD_MAX; i++) {
		/* global structure */
		fbfimd->fbdev[i] = kzalloc(sizeof(struct s3cfb_global), GFP_KERNEL);
		fbdev[i] = fbfimd->fbdev[i];
		......

		fbdev[i]->dev = &pdev->dev;

#if defined(CONFIG_MACH_SMDK4X12) || defined(CONFIG_FB_S5P_AMS369FG06)
		s3cfb_set_lcd_info(fbdev[i]);
#endif
        /*获取LCD配置信息*/
		/* platform_data*/
		pdata = to_fb_plat(&pdev->dev);

		if (pdata->lcd)
			fbdev[i]->lcd = (struct s3cfb_lcd *)pdata->lcd;
        /*配置GPIO端口*/
		if (pdata->cfg_gpio)
			pdata->cfg_gpio(pdev);
        /*设置时钟参数*/
		if (pdata->clk_on)
			pdata->clk_on(pdev, &fbdev[i]->clock);
        /*获取LCD平台设备所使用的IO端口资源，注意这个IORESOURCE_MEM标志和LCD平台设备定义中的一致*/
		/* io memory */
		res = platform_get_resource(pdev, IORESOURCE_MEM, i);
		......
		/*申请LCD IO端口所占用的IO空间(注意理解IO空间和内存空间的区别),request_mem_region定义在ioport.h中*/
		res = request_mem_region(res->start,
					res->end - res->start + 1, pdev->name);
		......
		 /*将LCD的IO端口占用的这段IO空间映射到内存的虚拟地址，ioremap定义在io.h中
         * 注意：IO空间要映射后才能使用，以后对虚拟地址的操作就是对IO空间的操作*/
		fbdev[i]->regs = ioremap(res->start, res->end - res->start + 1);
		fbdev[i]->regs_org = fbdev[i]->regs;
		......

		spin_lock_init(&fbdev[i]->vsync_slock);

#if defined(CONFIG_FB_S5P_VSYNC_THREAD)
		INIT_LIST_HEAD(&fbdev[i]->update_regs_list);
		mutex_init(&fbdev[i]->update_regs_list_lock);
		init_kthread_worker(&fbdev[i]->update_regs_worker);

		fbdev[i]->update_regs_thread = kthread_run(kthread_worker_fn,
				&fbdev[i]->update_regs_worker, "s3c-fb");
		......
		init_kthread_work(&fbdev[i]->update_regs_work, s3c_fb_update_regs_handler);
		fbdev[i]->timeline = sw_sync_timeline_create("s3c-fb");
		fbdev[i]->timeline_max = 1;
		fbdev[i]->support_fence = FENCE_SUPPORT;
#endif

		/* irq */
		fbdev[i]->irq = platform_get_irq(pdev, 0);
		if (request_irq(fbdev[i]->irq, s3cfb_irq_frame, IRQF_SHARED,
				pdev->name, fbdev[i])) {
			......
		}

#ifdef CONFIG_FB_S5P_TRACE_UNDERRUN
		if (request_irq(platform_get_irq(pdev, 1), s3cfb_irq_fifo,
				IRQF_DISABLED, pdev->name, fbdev[i])) {
			......
		}

		s3cfb_set_fifo_interrupt(fbdev[i], 1);
		dev_info(fbdev[i]->dev, "fifo underrun trace\n");
#endif
#ifdef CONFIG_FB_S5P_MDNIE
		/*  only FIMD0 is supported */
		if (i == 0)
			mdnie_setup();
#endif
		/* hw setting */
		s3cfb_init_global(fbdev[i]);

		fbdev[i]->system_state = POWER_ON;
		mutex_init(&fbdev[i]->output_lock);

		spin_lock_init(&fbdev[i]->slock);
        /*为framebuffer分配空间，进行内存映射，填充fb_info*/
		/* alloc fb_info */
		if (s3cfb_alloc_framebuffer(fbdev[i], i)) {
			......
		}
        /*注册fb设备到系统中*/
		/* register fb_info */
		if (s3cfb_register_framebuffer(fbdev[i])) {
			......
		}

		/* enable display */
		s3cfb_set_clock(fbdev[i]);

		/*  only FIMD0 is supported */
		if (i == 0) {
			if (pdata->set_display_path)
				pdata->set_display_path();

#ifdef CONFIG_FB_S5P_MDNIE
			s3cfb_set_dualrgb(fbdev[i], S3C_DUALRGB_MDNIE);
			mdnie_display_on();
#endif
		}
        /*设置时钟使能window*/
		s3cfb_enable_window(fbdev[0], pdata->default_win);

		s3cfb_update_power_state(fbdev[i], pdata->default_win,
					FB_BLANK_UNBLANK);

		/* Set alpha value width to 8-bit */
		s3cfb_set_alpha_value_width(fbdev[i], i);
        /*使能display*/
		s3cfb_display_on(fbdev[i]);

#if defined(CONFIG_CPU_EXYNOS4212) || defined(CONFIG_CPU_EXYNOS4412)
#ifdef CONFIG_BUSFREQ_OPP
		/* To lock bus frequency in OPP mode */
		fbdev[i]->bus_dev = dev_get("exynos-busfreq");
#endif
#endif

#ifdef CONFIG_HAS_WAKELOCK
#ifdef CONFIG_HAS_EARLYSUSPEND
		fbdev[i]->early_suspend.suspend = s3cfb_early_suspend;
		fbdev[i]->early_suspend.resume = s3cfb_late_resume;
		fbdev[i]->early_suspend.level = EARLY_SUSPEND_LEVEL_DISABLE_FB;

		register_early_suspend(&fbdev[i]->early_suspend);
#endif
#endif
#if defined(CONFIG_FB_S5P_VSYNC_THREAD)
		init_waitqueue_head(&fbdev[i]->vsync_info.wait);

		/* Create vsync thread */
		mutex_init(&fbdev[i]->vsync_info.irq_lock);

		fbdev[i]->vsync_info.thread = kthread_run(
						s3cfb_wait_for_vsync_thread,
						fbdev[i], "s3c-fb-vsync");
		if (fbdev[i]->vsync_info.thread == ERR_PTR(-ENOMEM)) {
			dev_err(fbdev[i]->dev, "failed to run vsync thread\n");
			fbdev[i]->vsync_info.thread = NULL;
		}
#endif
        /*对设备文件系统的支持，创建fb设备文件*/
		ret = device_create_file(fbdev[i]->dev, &dev_attr_fimd_dump);

		ret = device_create_file(fbdev[i]->dev, &dev_attr_ielcd_dump);

		ret = device_create_file(fbdev[i]->dev, &dev_attr_orient);

		ret = device_create_file(fbdev[i]->fb[pdata->default_win]->dev,
					&dev_attr_vsync_event);

		ret = device_create_file(fbdev[i]->fb[pdata->default_win]->dev,
					&dev_attr_vpost_event);

#ifdef CONFIG_FB_S5P_GD2EVF
		ret = device_create_file(fbdev[i]->dev, &dev_attr_lcd_switch);
		......
#endif
		fbdev[i]->ielcd_regs = ioremap(EXYNOS4_PA_LCD_LITE0, SZ_1K);
		fbdev[i]->dsim_regs = ioremap(S5P_PA_DSIM0, SZ_1K);
	}

#ifdef CONFIG_FB_S5P_LCD_INIT
	/* panel control */
	if (pdata->backlight_on)
		pdata->backlight_on(pdev);

	if (pdata->lcd_on)
		pdata->lcd_on(pdev);
#endif

	ret = device_create_file(&(pdev->dev), &dev_attr_win_power);
	.......
#ifdef DISPLAY_BOOT_PROGRESS
	if (!(readl(S5P_INFORM2)))
		s3cfb_start_progress(fbdev[0]->fb[pdata->default_win]);
#endif

#ifdef FEATURE_BUSFREQ_LOCK
	atomic_set(&fbdev[0]->busfreq_lock_cnt, 0);
#endif

	dev_info(fbdev[0]->dev, "registered successfully\n");

	return 0;
   ......
}

```


##### （1）、pdata->cfg_gpio(pdev)

``` cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\arch\arm\plat-s5p\include\plat\fb-s5p.h
struct s3c_platform_fb {
	int		hw_ver;
	char		clk_name[16];
	int		nr_wins;
	int		nr_buffers[5];
	int		default_win;
	int		swap;
	void		*lcd;
	void		(*set_display_path)(void);
	void		(*cfg_gpio)(struct platform_device *dev);
	int		(*backlight_on)(struct platform_device *dev);
	int		(*backlight_off)(struct platform_device *dev);
	int		(*lcd_on)(struct platform_device *dev);
	int		(*lcd_off)(struct platform_device *dev);
	int		(*clk_on)(struct platform_device *pdev, struct clk **s3cfb_clk);
	int		(*clk_off)(struct platform_device *pdev, struct clk **clk);
};

G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\arch\arm\mach-exynos\setup-fb-s5p.c
#elif defined(CONFIG_FB_S5P_TINY4412)
void s3cfb_cfg_gpio(struct platform_device *pdev)
{
	s3cfb_gpio_setup_24bpp(EXYNOS4_GPF0(0), 8, S3C_GPIO_SFN(2), S5P_GPIO_DRVSTR_LV3);
	s3cfb_gpio_setup_24bpp(EXYNOS4_GPF1(0), 8, S3C_GPIO_SFN(2), S5P_GPIO_DRVSTR_LV3);
	s3cfb_gpio_setup_24bpp(EXYNOS4_GPF2(0), 8, S3C_GPIO_SFN(2), S5P_GPIO_DRVSTR_LV3);
	s3cfb_gpio_setup_24bpp(EXYNOS4_GPF3(0), 4, S3C_GPIO_SFN(2), S5P_GPIO_DRVSTR_LV3);

	s5p_gpio_set_drvstr(EXYNOS4_GPF0(0), S5P_GPIO_DRVSTR_LV2);
	s5p_gpio_set_drvstr(EXYNOS4_GPF0(1), S5P_GPIO_DRVSTR_LV2);
	s5p_gpio_set_drvstr(EXYNOS4_GPF0(2), S5P_GPIO_DRVSTR_LV2);
	s5p_gpio_set_drvstr(EXYNOS4_GPF0(3), S5P_GPIO_DRVSTR_LV2);
}
```

##### （2）、pdata->clk_on(pdev, &fbdev[i]->clock)

> Log：
> <6>[    0.432476] s3cfb s3cfb.0: src_clk=800000000, vclk=67184640, div=12(11), rate=72727272

> <6>[    0.432488] s3cfb s3cfb.0: fimd sclk rate 66666666, clkdiv 0xfffffb

```
static unsigned int get_clk_rate(struct platform_device *pdev, struct clk *sclk)
{
	struct s3c_platform_fb *pdata = pdev->dev.platform_data;
	struct s3cfb_lcd *lcd = (struct s3cfb_lcd *)pdata->lcd;
	struct s3cfb_lcd_timing *timing = &lcd->timing;
	u32 src_clk, vclk, div, rate;
	u32 vclk_limit, div_limit, fimd_div;

	src_clk = clk_get_rate(sclk);

	vclk = (lcd->freq *
		(timing->h_bp + timing->h_fp + timing->h_sw + lcd->width) *
		(timing->v_bp + timing->v_fp + timing->v_sw + lcd->height));

	if (!vclk)
		vclk = src_clk;

	div = DIV_ROUND_CLOSEST(src_clk, vclk);

	if (lcd->freq_limit) {
		vclk_limit = (lcd->freq_limit *
			(timing->h_bp + timing->h_fp + timing->h_sw + lcd->width) *
			(timing->v_bp + timing->v_fp + timing->v_sw + lcd->height));

		div_limit = DIV_ROUND_CLOSEST(src_clk, vclk_limit);

		fimd_div = gcd(div, div_limit);

		if (div/fimd_div <= 16)
			div /= fimd_div;
	}

	if (!div) {
		dev_err(&pdev->dev, "div(%d) should be non-zero\n", div);
		div = 1;
	} else if (div > 16) {
		for (fimd_div = 2; fimd_div <= div; fimd_div++) {
			if ((div%fimd_div == 0) && (div/fimd_div <= 16))
				break;
		}
		div /= fimd_div;
	}

	div = (div > 16) ? 16 : div;
	rate = src_clk / div;

	if ((src_clk % rate) && (div != 1)) {
		div--;
		rate = src_clk / div;
		if (!(src_clk % rate))
			rate--;
	}

	dev_info(&pdev->dev, "src_clk=%d, vclk=%d, div=%d(%d), rate=%d\n",
		src_clk, vclk, DIV_ROUND_CLOSEST(src_clk, vclk), div, rate);

	return rate;
}

int s3cfb_clk_on(struct platform_device *pdev, struct clk **s3cfb_clk)
{
	struct clk *sclk = NULL;
	struct clk *mout_mpll = NULL;
	struct clk *lcd_clk = NULL;
	struct clksrc_clk *src_clk = NULL;
	struct s3c_platform_fb *pdata = pdev->dev.platform_data;
	struct s3cfb_lcd *lcd = (struct s3cfb_lcd *)pdata->lcd;

	u32 rate = 0, clkdiv = 0;
	int ret = 0;

	lcd_clk = clk_get(&pdev->dev, "lcd");
	......

	ret = clk_enable(lcd_clk);
	......
	clk_put(lcd_clk);

	sclk = clk_get(&pdev->dev, "sclk_fimd");
	......

	if (soc_is_exynos4210())
		mout_mpll = clk_get(&pdev->dev, "mout_mpll");
	else
		mout_mpll = clk_get(&pdev->dev, "mout_mpll_user");
	......
	ret = clk_set_parent(sclk, mout_mpll);
	......

	if (!lcd->vclk) {
		rate = get_clk_rate(pdev, mout_mpll);
		if (!rate)
			rate = 800 * MHZ;	/* MOUT PLL */
		lcd->vclk = rate;
	} else
		rate = lcd->vclk;

	ret = clk_set_rate(sclk, rate);

	......
	dev_dbg(&pdev->dev, "set fimd sclk rate to %d\n", rate);

	clk_put(mout_mpll);

	ret = clk_enable(sclk);
	......

	*s3cfb_clk = sclk;

#ifdef CONFIG_FB_S5P_MIPI_DSIM
	s3cfb_mipi_clk_enable(1);
#endif
#ifdef CONFIG_FB_S5P_MDNIE
	s3cfb_mdnie_clk_on(rate);
#ifdef CONFIG_FB_MDNIE_PWM
	s3cfb_mdnie_pwm_clk_on();
#endif
#endif

	src_clk = container_of(sclk, struct clksrc_clk, clk);
	clkdiv = __raw_readl(src_clk->reg_div.reg);

	dev_info(&pdev->dev, "fimd sclk rate %ld, clkdiv 0x%x\n",
		clk_get_rate(sclk), clkdiv);

	return 0;

......
}
```
##### （3）、s3cfb_init_global(fbdev[i])

``` cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\drivers\video\samsung\s3cfb_ops.c

int s3cfb_init_global(struct s3cfb_global *fbdev)
{
	fbdev->output = OUTPUT_RGB;
	fbdev->rgb_mode = MODE_RGB_P;

	fbdev->wq_count = 0;
	init_waitqueue_head(&fbdev->wq);
	mutex_init(&fbdev->lock);

	s3cfb_set_output(fbdev);
	s3cfb_set_display_mode(fbdev);
	s3cfb_set_polarity(fbdev);
	s3cfb_set_timing(fbdev);
	s3cfb_set_lcd_size(fbdev);

	return 0;
}

```
#####  3.1、s3cfb_set_output()

> 	fbdev->output = OUTPUT_RGB;// OUTPUT_RGB == 0
	fbdev->rgb_mode = MODE_RGB_P;// MODE_RGB_P == 0
	Line 399: <4>[    0.432589] zjj.android.display.sys s3cfb_set_output s3cfb_fimd6x.c S3C_VIDCON0 = 0
	Line 401: <4>[    0.432595] zjj.android.display.sys s3cfb_set_output s3cfb_fimd6x.c S3C_VIDCON2 = 0


``` cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\drivers\video\samsung\s3cfb_fimd6x.c
int s3cfb_set_output(struct s3cfb_global *ctrl)
{
	u32 cfg;

	cfg = readl(ctrl->regs + S3C_VIDCON0);
	cfg &= ~S3C_VIDCON0_VIDOUT_MASK;

	if (ctrl->output == OUTPUT_RGB)
		cfg |= S3C_VIDCON0_VIDOUT_RGB;
	else if (ctrl->output == OUTPUT_ITU)
		cfg |= S3C_VIDCON0_VIDOUT_ITU;
	else if (ctrl->output == OUTPUT_I80LDI0) {
		cfg |= S3C_VIDCON0_VIDOUT_I80LDI0;
		cfg |= S3C_VIDCON0_DSI_ENABLE;
	} else if (ctrl->output == OUTPUT_I80LDI1)
		cfg |= S3C_VIDCON0_VIDOUT_I80LDI1;
	else if (ctrl->output == OUTPUT_WB_RGB)
		cfg |= S3C_VIDCON0_VIDOUT_WB_RGB;
	else if (ctrl->output == OUTPUT_WB_I80LDI0)
		cfg |= S3C_VIDCON0_VIDOUT_WB_I80LDI0;
	else if (ctrl->output == OUTPUT_WB_I80LDI1)
		cfg |= S3C_VIDCON0_VIDOUT_WB_I80LDI1;
	else {
		dev_err(ctrl->dev, "invalid output type: %d\n", ctrl->output);
		return -EINVAL;
	}

	writel(cfg, ctrl->regs + S3C_VIDCON0);

	cfg = readl(ctrl->regs + S3C_VIDCON2);
	cfg &= ~(S3C_VIDCON2_WB_MASK | S3C_VIDCON2_TVFORMATSEL_MASK | \
					S3C_VIDCON2_TVFORMATSEL_YUV_MASK);

	if (ctrl->output == OUTPUT_RGB)
		cfg |= S3C_VIDCON2_WB_DISABLE;
	else if (ctrl->output == OUTPUT_ITU)
		cfg |= S3C_VIDCON2_WB_DISABLE;
	else if (ctrl->output == OUTPUT_I80LDI0)
		cfg |= S3C_VIDCON2_WB_DISABLE;
	else if (ctrl->output == OUTPUT_I80LDI1)
		cfg |= S3C_VIDCON2_WB_DISABLE;
	else if (ctrl->output == OUTPUT_WB_RGB)
		cfg |= (S3C_VIDCON2_WB_ENABLE | S3C_VIDCON2_TVFORMATSEL_SW | \
					S3C_VIDCON2_TVFORMATSEL_YUV444);
	else if (ctrl->output == OUTPUT_WB_I80LDI0)
		cfg |= (S3C_VIDCON2_WB_ENABLE | S3C_VIDCON2_TVFORMATSEL_SW | \
					S3C_VIDCON2_TVFORMATSEL_YUV444);
	else if (ctrl->output == OUTPUT_WB_I80LDI1)
		cfg |= (S3C_VIDCON2_WB_ENABLE | S3C_VIDCON2_TVFORMATSEL_SW | \
					S3C_VIDCON2_TVFORMATSEL_YUV444);
	else {
		dev_err(ctrl->dev, "invalid output type: %d\n", ctrl->output);
		return -EINVAL;
	}

#if defined(CONFIG_FB_RGBA_ORDER)
	/* Change format to BGR order */
	cfg &= ~(0x3F0000);
	cfg |= 0x240000;
#endif

	writel(cfg, ctrl->regs + S3C_VIDCON2);

	if (ctrl->output == OUTPUT_I80LDI0) {
		cfg = readl(ctrl->regs + S3C_I80IFCONA0);
		cfg |= ((1<<0)|(1<<8));
		writel(cfg, ctrl->regs + S3C_I80IFCONA0);
	}

	return 0;
}
```

#####  3.2、s3cfb_set_display_mode()

> 	Line 403: <4>[    0.432600] zjj.android.display.sys s3cfb_set_display_mode s3cfb_fimd6x.c 

	Line 405: <4>[    0.432606] zjj.android.display.sys s3cfb_set_display_mode s3cfb_fimd6x.c S3C_VIDCON0 = 0
``` cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\drivers\video\samsung\s3cfb_fimd6x.c

int s3cfb_set_display_mode(struct s3cfb_global *ctrl)
{
	u32 cfg;

	cfg = readl(ctrl->regs + S3C_VIDCON0);
	cfg &= ~S3C_VIDCON0_PNRMODE_MASK;
	cfg |= (ctrl->rgb_mode << S3C_VIDCON0_PNRMODE_SHIFT);
	writel(cfg, ctrl->regs + S3C_VIDCON0);

	return 0;
}
```

#####  3.3、s3cfb_set_polarity(fbdev)
> 	Line 407: <4>[    0.432611] zjj.android.display.sys s3cfb_set_polarity s3cfb_fimd6x.c 
>	Line 409: <4>[    0.432617] zjj.android.display.sys s3cfb_set_polarity s3cfb_fimd6x.c S3C_VIDCON1 = 680 //二进制：1101.0000000 
>G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\arch\arm\plat-s5p\include\plat\regs-fb-s5p.h
>	#define S3C_VIDCON1_FIXVCLK_VCLK_RUN_VDEN_DIS	(3 << 9)
>    #define S3C_VIDCON1_IVCLK_RISING_EDGE		(1 << 7)
>	1<<7 || 11<<9 == 1101.0000000 

``` cpp
int s3cfb_set_polarity(struct s3cfb_global *ctrl)
{
	struct s3cfb_lcd_polarity *pol;
	u32 cfg;

	pol = &ctrl->lcd->polarity;
	cfg = 0;

	/* Set VCLK hold scheme */
	cfg &= S3C_VIDCON1_FIXVCLK_MASK;
	cfg |= S3C_VIDCON1_FIXVCLK_VCLK_RUN_VDEN_DIS;

	if (pol->rise_vclk)
		cfg |= S3C_VIDCON1_IVCLK_RISING_EDGE;

	if (pol->inv_hsync)
		cfg |= S3C_VIDCON1_IHSYNC_INVERT;

	if (pol->inv_vsync)
		cfg |= S3C_VIDCON1_IVSYNC_INVERT;

	if (pol->inv_vden)
		cfg |= S3C_VIDCON1_IVDEN_INVERT;

	writel(cfg, ctrl->regs + S3C_VIDCON1);

	return 0;
}

```
#####  3.4、s3cfb_set_timing(fbdev)

``` cpp
int s3cfb_set_timing(struct s3cfb_global *ctrl)
{
	struct s3cfb_lcd_timing *time;
	u32 cfg;

	time = &ctrl->lcd->timing;
	cfg = 0;

	cfg |= S3C_VIDTCON0_VBPDE(time->v_bpe - 1);
	cfg |= S3C_VIDTCON0_VBPD(time->v_bp - 1);
	cfg |= S3C_VIDTCON0_VFPD(time->v_fp - 1);
	cfg |= S3C_VIDTCON0_VSPW(time->v_sw - 1);

	writel(cfg, ctrl->regs + S3C_VIDTCON0);

	cfg = 0;

	cfg |= S3C_VIDTCON1_VFPDE(time->v_fpe - 1);
	cfg |= S3C_VIDTCON1_HBPD(time->h_bp - 1);
	cfg |= S3C_VIDTCON1_HFPD(time->h_fp - 1);
	cfg |= S3C_VIDTCON1_HSPW(time->h_sw - 1);

	writel(cfg, ctrl->regs + S3C_VIDTCON1);

	return 0;
}

```

``` cpp	
/* s3cfb configs for supported LCD */

static struct s3cfb_lcd wxga_hd700 = {
	.width = 800,
	.height = 1280,
	.p_width = 94,
	.p_height = 151,
	.bpp = 24,
	.freq = 60,

	.timing = {
		.h_fp = 20,
		.h_bp = 20,
		.h_sw = 24,
		.v_fp =  4,
		.v_fpe = 1,
		.v_bp =  4,
		.v_bpe = 1,
		.v_sw =  8,
	},
	.polarity = {
		.rise_vclk = 1,
		.inv_hsync = 0,
		.inv_vsync = 0,
		.inv_vden = 0,
	},
};


	Line 411: <4>[    0.432622] zjj.android.display.sys s3cfb_set_timing s3cfb_fimd6x.c 
	Line 413: <4>[    0.432627] zjj.android.display.sys s3cfb_set_timing s3cfb_fimd6x.c  S3C_VIDTCON0 = 30307 //11000000,1100000111

 

	cfg |= S3C_VIDTCON0_VBPDE(time->v_bpe - 1); // 0  0 << 24
	cfg |= S3C_VIDTCON0_VBPD(time->v_bp - 1); // 3   11 << 16 
	cfg |= S3C_VIDTCON0_VFPD(time->v_fp - 1); // 3   11<<8
	cfg |= S3C_VIDTCON0_VSPW(time->v_sw - 1); // 7   111
	
	Line 413: <4>[    0.432544] zjj.android.display.sys s3cfb_set_timing s3cfb_fimd6x.c  S3C_VIDTCON0 = 30307 //100110001001100010111
	cfg |= S3C_VIDTCON1_VFPDE(time->v_fpe - 1); // 0  0 << 24
	cfg |= S3C_VIDTCON1_HBPD(time->h_bp - 1); // 19  10011 <<16
	cfg |= S3C_VIDTCON1_HFPD(time->h_fp - 1); // 19  10011 <<8
	cfg |= S3C_VIDTCON1_HSPW(time->h_sw - 1); // 23  10111 <<0

    ////////////
    /* VIDTCON0 */
    #define S3C_VIDTCON0_VBPDE(x)			(((x) & 0xff) << 24)
    #define S3C_VIDTCON0_VBPD(x)			(((x) & 0xff) << 16)
    #define S3C_VIDTCON0_VFPD(x)			(((x) & 0xff) << 8)
    #define S3C_VIDTCON0_VSPW(x)			(((x) & 0xff) << 0)

    /* VIDTCON1 */
    #define S3C_VIDTCON1_VFPDE(x)			(((x) & 0xff) << 24)
    #define S3C_VIDTCON1_HBPD(x)			(((x) & 0xff) << 16)
    #define S3C_VIDTCON1_HFPD(x)			(((x) & 0xff) << 8)
    #define S3C_VIDTCON1_HSPW(x)			(((x) & 0xff) << 0)
```

#####  3.5、s3cfb_set_lcd_size(fbdev)

``` cpp
Line 417: <4>[    0.432639] zjj.android.display.sys s3cfb_set_lcd_size s3cfb_fimd6x.c  S3C_VIDTCON2 = 27fb1f  //二进制： 1001111111101100011111

#define S3C_VIDTCON2_LINEVAL(x)			(((x) & 0x7ff) << 11)
#define S3C_VIDTCON2_HOZVAL(x)			(((x) & 0x7ff) << 0)

1279+1=1280  HOZVAL [10:0]  10011111111
799+1=800    LINEVAL [21:11] 01100011111
NOTE: HOZVAL = (Horizontal display size) – 1 and LINEVAL = (Vertical display size) – 1
```


##### （4）、s3cfb_alloc_framebuffer(fbdev[i], i)

``` cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\drivers\video\samsung\s3cfb_ops.c
int s3cfb_alloc_framebuffer(struct s3cfb_global *fbdev, int fimd_id)
{
	struct s3c_platform_fb *pdata = to_fb_plat(fbdev->dev);
	int ret = 0;
	int i;
    /* 分配 fb_info */
	fbdev->fb = kmalloc(pdata->nr_wins *
				sizeof(struct fb_info *), GFP_KERNEL);
	......
	for (i = 0; i < pdata->nr_wins; i++) {
		fbdev->fb[i] = framebuffer_alloc(sizeof(struct s3cfb_window),
						fbdev->dev);
		......
		 /* 初始化 fb_info */
		ret = s3cfb_init_fbinfo(fbdev, i);
		......

		if (i == pdata->default_win) {
			if (s3cfb_map_default_video_memory(fbdev,
						fbdev->fb[i], fimd_id)) {
				......
				ret = -ENOMEM;
				goto err_alloc_fb;
			} else {
				memcpy(&(fbdev->initial_fix), &(fbdev->fb[i]->fix), sizeof(struct fb_fix_screeninfo));
				memcpy(&(fbdev->initial_var), &(fbdev->fb[i]->var), sizeof(struct fb_var_screeninfo));
			}
		}
		sec_getlog_supply_fbinfo((void *)fbdev->fb[i]->fix.smem_start,
					 fbdev->fb[i]->var.xres,
					 fbdev->fb[i]->var.yres,
					 fbdev->fb[i]->var.bits_per_pixel, 2);
	}

	return 0;

}

```

##### 4.1、framebuffer_alloc()

``` cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\drivers\video\fbsysfs.c
struct fb_info *framebuffer_alloc(size_t size, struct device *dev)
{
#define BYTES_PER_LONG (BITS_PER_LONG/8)
#define PADDING (BYTES_PER_LONG - (sizeof(struct fb_info) % BYTES_PER_LONG))
	int fb_info_size = sizeof(struct fb_info);
	struct fb_info *info;
	char *p;

	if (size)
		fb_info_size += PADDING;

	p = kzalloc(fb_info_size + size, GFP_KERNEL);


	info = (struct fb_info *) p;

	if (size)
		info->par = p + fb_info_size;

	info->device = dev;

#ifdef CONFIG_FB_BACKLIGHT
	mutex_init(&info->bl_curve_mutex);
#endif

	return info;
#undef PADDING
#undef BYTES_PER_LONG
}
EXPORT_SYMBOL(framebuffer_alloc);
```

##### 4.2、s3cfb_init_fbinfo()

``` cpp
int s3cfb_init_fbinfo(struct s3cfb_global *fbdev, int id)
{
	struct fb_info *fb = fbdev->fb[id];
	struct fb_fix_screeninfo *fix = &fb->fix;
	struct fb_var_screeninfo *var = &fb->var;
	struct s3cfb_window *win = fb->par;
	struct s3cfb_alpha *alpha = &win->alpha;
	struct s3cfb_lcd *lcd = fbdev->lcd;
	struct s3cfb_lcd_timing *timing = &lcd->timing;

	memset(win, 0, sizeof(struct s3cfb_window));
	platform_set_drvdata(to_platform_device(fbdev->dev), fb);
	strcpy(fix->id, S3CFB_NAME);

	/* fimd specific */
	win->id = id;
	win->path = DATA_PATH_DMA;
	win->dma_burst = 16;
	s3cfb_update_power_state(fbdev, win->id, FB_BLANK_POWERDOWN);
	alpha->mode = PLANE_BLENDING;

	/* fbinfo */
	fb->fbops = &s3cfb_ops;
	fb->flags = FBINFO_FLAG_DEFAULT;
	fb->pseudo_palette = &win->pseudo_pal;
    /* 设置 fix 固定的参数 */
#if (CONFIG_FB_S5P_NR_BUFFERS != 1)
#if defined(CONFIG_CPU_EXYNOS4210)
	fix->xpanstep = 1; /*  xpanstep can be 1 if bits_per_pixel is 32 */
#else
	fix->xpanstep = 2;
#endif
	fix->ypanstep = 1;
#else
	fix->xpanstep = 0;
	fix->ypanstep = 0;
#endif
	fix->type = FB_TYPE_PACKED_PIXELS;
	fix->accel = FB_ACCEL_NONE;
	fix->visual = FB_VISUAL_TRUECOLOR;
	 /* 设置 var 可变的参数 */
	var->xres = lcd->width;
	var->yres = lcd->height;

#if defined(CONFIG_FB_S5P_VIRTUAL)
	var->xres_virtual = CONFIG_FB_S5P_X_VRES;
	var->yres_virtual = CONFIG_FB_S5P_Y_VRES * CONFIG_FB_S5P_NR_BUFFERS;
#else
	var->xres_virtual = var->xres;
	var->yres_virtual = var->yres * CONFIG_FB_S5P_NR_BUFFERS;
#endif

	var->bits_per_pixel = 32;
	var->xoffset = 0;
	var->yoffset = 0;
	var->width = lcd->p_width;
	var->height = lcd->p_height;
	var->transp.length = 8;

	fix->line_length = var->xres_virtual * var->bits_per_pixel / 8;
	fix->smem_len = fix->line_length * var->yres_virtual;

	var->nonstd = 0;
	var->activate = FB_ACTIVATE_NOW;
	var->vmode = FB_VMODE_NONINTERLACED;
	var->hsync_len = timing->h_sw;
	var->vsync_len = timing->v_sw;
	var->left_margin = timing->h_bp;
	var->right_margin = timing->h_fp;
	var->upper_margin = timing->v_bp;
	var->lower_margin = timing->v_fp;
	var->pixclock = (lcd->freq *
		(var->left_margin + var->right_margin
		+ var->hsync_len + var->xres) *
		(var->upper_margin + var->lower_margin
		+ var->vsync_len + var->yres));
	var->pixclock = KHZ2PICOS(var->pixclock/1000);

	s3cfb_set_bitfield(var);
	s3cfb_set_alpha_info(var, win);

	return 0;
}
```
##### 4.3、s3cfb_map_default_video_memory()

``` cpp
int s3cfb_map_default_video_memory(struct s3cfb_global *fbdev,
					struct fb_info *fb, int fimd_id)
{
	struct fb_fix_screeninfo *fix = &fb->fix;
	struct s3cfb_window *win = fb->par;

#ifdef CONFIG_CMA
	struct cma_info mem_info;
	int err;
#endif

	if (win->owner == DMA_MEM_OTHER)
		return 0;

#ifdef CONFIG_CMA
	err = cma_info(&mem_info, fbdev->dev, CMA_REGION_FIMD);
	if (err)
		return err;
	fix->smem_start = (dma_addr_t)cma_alloc
		(fbdev->dev, "fimd", (size_t)PAGE_ALIGN(fix->smem_len), 0);
	fb->screen_base = cma_get_virt(fix->smem_start, PAGE_ALIGN(fix->smem_len), 1);
#elif defined(CONFIG_S5P_MEM_BOOTMEM)
	fix->smem_start = s5p_get_media_memory_bank(S5P_MDEV_FIMD, 1);
	fix->smem_len = s5p_get_media_memsize_bank(S5P_MDEV_FIMD, 1);
	fb->screen_base = ioremap_wc(fix->smem_start, fix->smem_len);
#else
	fb->screen_base = dma_alloc_writecombine(fbdev->dev,
						PAGE_ALIGN(fix->smem_len),
						(unsigned int *)
						&fix->smem_start, GFP_KERNEL);
#endif

	if (!fb->screen_base)
		return -ENOMEM;
	else
		dev_info(fbdev->dev, "[fb%d] win%d: dma: 0x%08x, cpu: 0x%08x, "
			"size: 0x%08x\n", fb->node, win->id,
			(unsigned int)fix->smem_start,
			(unsigned int)fb->screen_base, fix->smem_len);

#ifdef CONFIG_FB_S5P_SPLASH_SCREEN
	if (bootloaderfb)
#endif
		memset(fb->screen_base, 0, fix->smem_len);
	win->owner = DMA_MEM_FIMD;

#ifdef CONFIG_FB_S5P_SYSMMU
	fbdev->sysmmu.default_fb_addr = fix->smem_start;
#endif

	return 0;
}

```

##### （5）、s3cfb_register_framebuffer(fbdev[i])

``` cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\drivers\video\samsung\s3cfb_main.c

int s3cfb_register_framebuffer(struct s3cfb_global *fbdev)
{
	struct s3c_platform_fb *pdata = to_fb_plat(fbdev->dev);
	int ret, i, j;

	/* on registering framebuffer, framebuffer of default window is registered at first. */
	for (i = pdata->default_win; i < pdata->nr_wins + pdata->default_win; i++) {
		j = i % pdata->nr_wins;
		ret = register_framebuffer(fbdev->fb[j]);
		......
#ifndef CONFIG_FRAMEBUFFER_CONSOLE
		if (j == pdata->default_win) {
			s3cfb_check_var_window(fbdev, &fbdev->fb[j]->var,
					fbdev->fb[j]);
			ret = s3cfb_set_par_window(fbdev, fbdev->fb[j]);
			if (ret != 0)
				BUG();
			s3cfb_draw_logo(fbdev->fb[j]);
		}
#endif
	}
	return 0;
......
}

G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\drivers\video\fbmem.c
int
register_framebuffer(struct fb_info *fb_info)
{
	int ret;

	mutex_lock(&registration_lock);
	ret = do_register_framebuffer(fb_info);
	mutex_unlock(&registration_lock);

	return ret;
}

static int do_register_framebuffer(struct fb_info *fb_info)
{
	int i;
	struct fb_event event;
	struct fb_videomode mode;

	if (fb_check_foreignness(fb_info))
		return -ENOSYS;

	do_remove_conflicting_framebuffers(fb_info->apertures, fb_info->fix.id,
					 fb_is_primary_device(fb_info));

	if (num_registered_fb == FB_MAX)
		return -ENXIO;

	num_registered_fb++;
	for (i = 0 ; i < FB_MAX; i++)
		if (!registered_fb[i])
			break;
	fb_info->node = i;
	atomic_set(&fb_info->count, 1);
	mutex_init(&fb_info->lock);
	mutex_init(&fb_info->mm_lock);

	fb_info->dev = device_create(fb_class, fb_info->device,
				     MKDEV(FB_MAJOR, i), NULL, "fb%d", i);
	if (IS_ERR(fb_info->dev)) {
		/* Not fatal */
		printk(KERN_WARNING "Unable to create device for framebuffer %d; errno = %ld\n", i, PTR_ERR(fb_info->dev));
		fb_info->dev = NULL;
	} else
		fb_init_device(fb_info);

	if (fb_info->pixmap.addr == NULL) {
		fb_info->pixmap.addr = kmalloc(FBPIXMAPSIZE, GFP_KERNEL);
		if (fb_info->pixmap.addr) {
			fb_info->pixmap.size = FBPIXMAPSIZE;
			fb_info->pixmap.buf_align = 1;
			fb_info->pixmap.scan_align = 1;
			fb_info->pixmap.access_align = 32;
			fb_info->pixmap.flags = FB_PIXMAP_DEFAULT;
		}
	}	
	fb_info->pixmap.offset = 0;

	if (!fb_info->pixmap.blit_x)
		fb_info->pixmap.blit_x = ~(u32)0;

	if (!fb_info->pixmap.blit_y)
		fb_info->pixmap.blit_y = ~(u32)0;

	if (!fb_info->modelist.prev || !fb_info->modelist.next)
		INIT_LIST_HEAD(&fb_info->modelist);

	fb_var_to_videomode(&mode, &fb_info->var);
	fb_add_videomode(&mode, &fb_info->modelist);
	registered_fb[i] = fb_info; //加入到registered_fb[]数组中

	event.info = fb_info;
	if (!lock_fb_info(fb_info))
		return -ENODEV;
	console_lock();
	fb_notifier_call_chain(FB_EVENT_FB_REGISTERED, &event);
	console_unlock();
	unlock_fb_info(fb_info);
	return 0;
}
```
##### （6）、s3cfb_set_clock(fbdev[i])

> 	Line 439: <4>[    0.449600] zjj.android.display.sys s3cfb_set_clock() s3cfb_probe s3cfb_main.c
> 	Line 441: <4>[    0.449607] zjj.android.display.sys s3cfb_set_clock s3cfb_fimd6x.c 
> 	Line 443: <6>[    0.449616] s3cfb s3cfb.0: FIMD src sclk = 66666666
> 	Line 445: <4>[    0.449622] zjj.android.display.sys s3cfb_set_clock s3cfb_fimd6x.c S3C_VIDCON0 = 20 //1000.00 
> S3C_VIDCON0_CLKVAL_F(x)			(((x) & 0xff) << 6)
> ENVID_F 0
> ENVID   0

``` cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\drivers\video\samsung\s3cfb_fimd6x.c
int s3cfb_set_clock(struct s3cfb_global *ctrl)
{
	struct s3c_platform_fb *pdata = to_fb_plat(ctrl->dev);
	u32 cfg, maxclk, src_clk, vclk, div;
#ifdef CONFIG_FB_S5P_GD2EVF
	struct s3cfb_lcd_timing *timing = &ctrl->lcd->timing;
#endif

	/* spec is under 100MHz */
	maxclk = 100 * 1000000;

	cfg = readl(ctrl->regs + S3C_VIDCON0);

	if (pdata->hw_ver == 0x70) {
		cfg &= ~(S3C_VIDCON0_CLKVALUP_MASK |
			S3C_VIDCON0_VCLKEN_MASK);
		cfg |= (S3C_VIDCON0_CLKVALUP_ALWAYS |
			S3C_VIDCON0_VCLKEN_FREERUN);

		src_clk = clk_get_rate(ctrl->clock);
		dev_dbg(ctrl->dev, "FIMD src sclk = %d\n", src_clk);
	} else {
		cfg &= ~(S3C_VIDCON0_CLKSEL_MASK |
			S3C_VIDCON0_CLKVALUP_MASK |
			S3C_VIDCON0_VCLKEN_MASK |
			S3C_VIDCON0_CLKDIR_MASK);
		cfg |= (S3C_VIDCON0_CLKVALUP_ALWAYS |
			S3C_VIDCON0_VCLKEN_NORMAL |
			S3C_VIDCON0_CLKDIR_DIVIDED);

		if (strcmp(pdata->clk_name, "sclk_fimd") == 0) {
			cfg |= S3C_VIDCON0_CLKSEL_SCLK;
			src_clk = clk_get_rate(ctrl->clock);
			dev_dbg(ctrl->dev, "FIMD src sclk = %d\n", src_clk);

		} else {
			cfg |= S3C_VIDCON0_CLKSEL_HCLK;
			src_clk = ctrl->clock->parent->rate;
			dev_dbg(ctrl->dev, "FIMD src hclk = %d\n", src_clk);
		}
	}

#ifdef CONFIG_FB_S5P_GD2EVF
	vclk = (ctrl->lcd->freq *
		(timing->h_bp + timing->h_fp + timing->h_sw + ctrl->lcd->width) *
		(timing->v_bp + timing->v_fp + timing->v_sw + ctrl->lcd->height));
#else
	vclk = PICOS2KHZ(ctrl->fb[pdata->default_win]->var.pixclock) * 1000;
#endif

	if (vclk > maxclk) {
		dev_info(ctrl->dev, "vclk(%d) should be smaller than %d\n",
			vclk, maxclk);
		/* vclk = maxclk; */
	}

	div = DIV_ROUND_CLOSEST(src_clk, vclk);

	......

	if ((src_clk/div) > maxclk)
		dev_info(ctrl->dev, "vclk(%d) should be smaller than %d Hz\n",
			src_clk/div, maxclk);

	cfg &= ~S3C_VIDCON0_CLKVAL_F(0xff);
	cfg |= S3C_VIDCON0_CLKVAL_F(div - 1);
	writel(cfg, ctrl->regs + S3C_VIDCON0);

	dev_info(ctrl->dev, "parent clock: %d, vclk: %d, vclk div: %d\n",
			src_clk, vclk, div);

	return 0;
}
```
##### （7）、s3cfb_enable_window(fbdev[0], pdata->default_win)

``` cpp
	Line 449: <4>[    0.449635] zjj.android.display.sys s3cfb_enable_window() s3cfb_probe s3cfb_main.c
	Line 451: <4>[    0.449641] zjj.android.display.sys s3cfb_window_on s3cfb_fimd6x.c
	Line 453: <6>[    0.449647] s3cfb s3cfb.0: [win2] turn on

G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\drivers\video\samsung\s3cfb_ops.c

int s3cfb_enable_window(struct s3cfb_global *fbdev, int id)
{
	struct s3cfb_window *win = fbdev->fb[id]->par;
#ifdef FEATURE_BUSFREQ_LOCK
	int enabled_win = 0;
#endif
#if defined(CONFIG_CPU_EXYNOS4212) || defined(CONFIG_CPU_EXYNOS4412)
#ifdef CONFIG_BUSFREQ_OPP
	if (CONFIG_FB_S5P_DEFAULT_WINDOW == 3 &&
		id == CONFIG_FB_S5P_DEFAULT_WINDOW-1)
		dev_lock(fbdev->bus_dev, fbdev->dev, 267160);
	else if (id != CONFIG_FB_S5P_DEFAULT_WINDOW) {
		if (id == CONFIG_FB_S5P_DEFAULT_WINDOW-1)
			dev_lock(fbdev->bus_dev, fbdev->dev, 267160);
		else if (id == 3)
			dev_lock(fbdev->bus_dev, fbdev->dev, 267160);
		else
			dev_lock(fbdev->bus_dev, fbdev->dev, 133133);
	}
#endif
#endif

	if (!win->enabled)
		atomic_inc(&fbdev->enabled_win);

#ifdef FEATURE_BUSFREQ_LOCK
	enabled_win = atomic_read(&fbdev->enabled_win);
	if (enabled_win >= 2)
		s3cfb_busfreq_lock(fbdev, 1);
#endif

	if (s3cfb_window_on(fbdev, id)) {
		win->enabled = 0;
		return -EFAULT;
	} else {
		win->enabled = 1;
		return 0;
	}
}
```

##### （8）、s3cfb_display_on(fbdev[i])

``` cpp

	Line 457: <4>[    0.449657] zjj.android.display.sys s3cfb_display_on s3cfb_fimd6x.c
	Line 459: <4>[    0.449662] zjj.android.display.sys s3cfb_set_display_on s3cfb_fimd6x.c S3C_VIDCON0 = 23 //1000.11 
	ENVID_F 1
    ENVID   1
   
	Line 461: <6>[    0.449668] s3cfb s3cfb.0: global display is on
	Line 463: <6>[    0.449741] s3cfb s3cfb.0: registered successfully

G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\drivers\video\samsung\s3cfb_ops.c
int s3cfb_display_on(struct s3cfb_global *ctrl)
{
	u32 cfg;

	cfg = readl(ctrl->regs + S3C_VIDCON0);
	cfg |= (S3C_VIDCON0_ENVID_ENABLE | S3C_VIDCON0_ENVID_F_ENABLE);
	writel(cfg, ctrl->regs + S3C_VIDCON0);

	dev_dbg(ctrl->dev, "global display is on\n");

	return 0;
}
```
然后开启背光，LCD就开始显示画面了。

``` cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\drivers\video\samsung\s3cfb_main.c

/* panel control */
if (pdata->backlight_on)
	pdata->backlight_on(pdev);

if (pdata->lcd_on)
	pdata->lcd_on(pdev);
```

![enter image description here](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/personal.website/zjj.display.sys.s3cfb_probe.png)



####  （四）、参考文档：

[（1）【LCD设备驱动框架分析（数据结构）】](https://blog.csdn.net/z961968549/article/details/78821082)
[（2）【framebuffer之s3cfb_probe分析】](https://blog.csdn.net/Ultraman_hs/article/details/54987874)
[（3）【LCD驱动框架】](https://layty.gitee.io/2018/12/03/Linux/%E8%AE%BE%E5%A4%87%E9%A9%B1%E5%8A%A8/12-lcd%E9%A9%B1%E5%8A%A8%E6%A1%86%E6%9E%B6/)
