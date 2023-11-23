---
title: Android N Display System（5）：Android Display System 系统分析之Display Driver Architecture
cover: https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/hexo.themes/bing-wallpaper-2018.04.23.jpg
categories: 
  - Display
tags:
  - Android
  - Graphics
  - Display
toc: true
abbrlink: 20180808
date: 2018-08-08 09:25:00
---

--------------------------------------------------------------------------------

注：文章都是通过阅读各位前辈总结的资料、Android 7.1.2 && Linux（kernel 3.18）Qualcomm平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除，禁止转载（©Qualcomm Technologies, Inc. 版权所有），谢谢。


[【特别感谢 - 高通Android平台-应用空间操作framebuffer dump LCD总结】](https://blog.csdn.net/eliot_shao/article/details/74926010)
[【特别感谢 -  linux qcom LCD framwork】](https://blog.csdn.net/u012719256/article/details/52096727)
[【特别感谢 -  msm8610 lcd driver code analysis】](https://blog.csdn.net/ic_soc_arm_robin/article/details/12949347)



Google Pixel、Pixel XL 内核代码（文章基于 Kernel-3.18）：
 [Kernel source for Pixel and Pixel XL - GitHub](https://github.com/matthewdalex/marlin)

AOSP 源码（文章基于 Android 7.1.2）：
 [ Android 系统全套源代码分享 (更新到 8.1.0_r1)](https://testerhome.com/topics/2229)

 🌀🌀：专注于Linux && Android Multimedia（Camera、Video、Audio、Display）系统分析与研究

【Android Display System 系统分析系列】：
【Android Display System（1）：Android 7.1.2 (Android N) Android Graphics 系统 分析】
【Android Display System（2）：Android Display System 系统分析之Android EGL && OpenGL】
【Android Display System（3）：Android Display System 系统分析之HardwareRenderer.draw()绘制流程分析】
【Android Display System（4）：Android Display System 系统分析之Gralloc && HWComposer模块分析】
【Android Display System（5）：Android Display System 系统分析之Display Driver Architecture】 

--------------------------------------------------------------------------------

路径：
kernel\msm-3.18\drivers\video\msm\mdss

 MDSS driver software block diagram
- mdss_fb → Top-level IOCTL/native framebufferinterface
- mdss_mdp.c → MDP resources(clocks/irq/bus-bw/power)
- mdss_mdp_overlay → Overlay/DMA top-levelAPI
- mdss_mdp_ctl → Controls the hardware abstraction to club the (LM + DSPP + Ping-pong +
interface)
- mdss_mdp_pipe → SRC pipe related handling
- mdss_mdp_intf_cmd/mdss_mdp_intf_video/mdss_mdp_intf_writeback → MDP panel
interface relatedhandling
- mdss_mdp_pp → Postprocessing related implementation
- mdss_mdp_rotator → Rotator APIs (overlay_set/overlay_playinterface)
- mdss_mdp_pp.c → Postprocessing relatedmaterial

--------------------------------------------------------------------------------

**注：**首先说明，由于博主不是kernel开发方向的，可能理解不够透彻，还请看官见谅，主要是为了理解Kernel Display原理。为了加深对显示屏工作原理的理解，首先先看两个操作LCD显示屏的例子
绪论（总体架构图）：

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/display.system/DS05-01-Display-Architecture.png)


#### （一）、直接操作framebuffer显示图像
##### 1.1、直接操作framebuffer显示图像

##### 1.1.1、源代码
**panel_test.c**

``` cpp
#include <unistd.h>  
#include <stdio.h>  
#include <fcntl.h>  
#include <linux/fb.h>  
#include <sys/mman.h>  
#include <stdlib.h>
#include "yellow_face.zif"
int main()  
{  
    int fbfd = 0;  
    struct fb_var_screeninfo vinfo;  
    struct fb_fix_screeninfo finfo;  
    struct fb_cmap cmapinfo;  
    long int screensize = 0;  
    char *fbp = 0;  
    int x = 0, y = 0;  
    long int location = 0;  
    int b,g,r;  
    // Open the file for reading and writing  
    fbfd = open("/dev/graphics/fb0", O_RDWR,0);                    // 打开Frame Buffer设备  
    if (fbfd < 0) {  
                printf("Error: cannot open framebuffer device.%x\n",fbfd);  
                exit(1);  
    }  
    printf("The framebuffer device was opened successfully.\n");  
  
    // Get fixed screen information  
    if (ioctl(fbfd, FBIOGET_FSCREENINFO, &finfo)) {            // 获取设备固有信息  
                printf("Error reading fixed information.\n");  
                exit(2);  
    }  
    printf("\ntype:0x%x\n", finfo.type );                            // FrameBuffer 类型,如0为象素  
    printf("visual:%d\n", finfo.visual );                        // 视觉类型：如真彩2，伪彩3   
    printf("line_length:%d\n", finfo.line_length );        // 每行长度  
    printf("\nsmem_start:0x%lx,smem_len:%u\n", finfo.smem_start, finfo.smem_len ); // 映象RAM的参数  
    printf("mmio_start:0x%lx ,mmio_len:%u\n", finfo.mmio_start, finfo.mmio_len );  
      
    // Get variable screen information  
    if (ioctl(fbfd, FBIOGET_VSCREENINFO, &vinfo)) {            // 获取设备可变信息  
                printf("Error reading variable information.\n");  
                exit(3);  
    }  
    printf("%dx%d, %dbpp,xres_virtual=%d,yres_virtual=%dvinfo.xoffset=%d,vinfo.yoffset=%d\n", vinfo.xres, vinfo.yres, vinfo.bits_per_pixel,vinfo.xres_virtual,vinfo.yres_virtual,vinfo.xoffset,vinfo.yoffset);  

	screensize = finfo.line_length * vinfo.yres_virtual;
    // Map the device to memory 通过mmap系统调用将framebuffer内存映射到用户空间,并返回映射后的起始地址  
    fbp = (char *)mmap(0, screensize, PROT_READ | PROT_WRITE, MAP_SHARED,fbfd, 0);  
    if ((int)fbp == -1) {  
                printf("Error: failed to map framebuffer device to memory.\n");  
                exit(4);  
    }  
    printf("The framebuffer device was mapped to memory successfully.\n");  
/***************exampel 1**********************/
	b = 10;
	g = 100;
	r = 100;
    for ( y = 0; y < 340; y++ )
        for ( x = 0; x < 420; x++ ) { 
      
         location = (x+100) * (vinfo.bits_per_pixel/8) + 
             (y+100) * finfo.line_length; 
      
         if ( vinfo.bits_per_pixel == 32 ) {        //          
                        *(fbp + location) = b; // Some blue  
                        *(fbp + location + 1) = g;             // A little green  
                        *(fbp + location + 2) = r;             // A lot of red  
                        *(fbp + location + 3) = 0;     // No transparency  
         }
      
        }
/*****************exampel 1********************/
/*****************exampel 2********************/		
 	unsigned char *pTemp = (unsigned char *)fbp;
	int i, j;
	//起始坐标(x,y),终点坐标(right,bottom)
	x = 400;
	y = 400;
	int right = 700;//vinfo.xres;
	int bottom = 1000;//vinfo.yres;
	
	for(i=y; i< bottom; i++)
	{
		for(j=x; j<right; j++)
		{
			unsigned short data = yellow_face_data[(((i-y)  % 128) * 128) + ((j-x) %128)];
			pTemp[i*finfo.line_length + (j*4) + 2] = (unsigned char)((data & 0xF800) >> 11 << 3);
			pTemp[i*finfo.line_length + (j*4) + 1] = (unsigned char)((data & 0x7E0) >> 5 << 2);
			pTemp[i*finfo.line_length + (j*4) + 0] = (unsigned char)((data & 0x1F) << 3);
		}
	} 
/*****************exampel 2********************/	
//note：vinfo.xoffset =0 vinfo.yoffset =0 否则FBIOPAN_DISPLAY不成功
	if (ioctl(fbfd, FBIOPAN_DISPLAY, &vinfo)) {    
                printf("Error FBIOPAN_DISPLAY information.\n");  
                exit(5);  
    }  
	sleep(10);
	munmap(fbp,finfo.smem_len);//finfo.smem_len == screensize == finfo.line_length * vinfo.yres_virtual 
    close(fbfd);  
    return 0;  
}  

```

**Android.mk**

``` cpp
# Copyright 2006-2014 The Android Open Source Project

LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)

LOCAL_SRC_FILES:= panel_test.c


LOCAL_SHARED_LIBRARIES        := $(common_libs) libqdutils libdl liblog libbase libcutils
LOCAL_C_INCLUDES              := $(common_includes) $(kernel_includes)
LOCAL_ADDITIONAL_DEPENDENCIES := $(common_deps) $(kernel_deps)

LOCAL_MODULE := panel_test

LOCAL_CFLAGS := -Werror

include $(BUILD_EXECUTABLE)

include $(call first-makefiles-under,$(LOCAL_PATH))
```
**yellow_face.zif**
[yellow_face.zif](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/display.system/DS05-yellow_face.zif)

##### 1.1.2、编译测试
编译会生成panel_test，然后进行测试。

>注意事项：
1、adb shell stop 杀掉surfaceflinger 之后在测试；
2、设置背光 echo 255 > /sys/class/leds/lcd-backlight/brightness

> 1、连接adb 
> 2、adb push panel_test system/bin 
> 3、进入adb shell
> 4、stop
> 5、echo 255 > /sys/class/leds/lcd-backlight/brightness
> 6、system/bin/panel_test

##### 1.1.3、显示效果

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/display.system/DS05-02-Qcom_FrameBuffer_test_.gif)

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/display.system/DS05-03-yellow_face_zif_test.jpg)

##### 1.1.4、视频（加深理解、自☯备☯梯☯子）
[Android Frame Buffer and Screen Shots Tutorial](https://www.youtube.com/watch?v=BUPPyR6VasI)
[Mplayer on Android Through Chroot and Frame Buffer](https://www.youtube.com/watch?v=7n_hDZ6kHjc)
##### 1.1.5、驱动总体概览图

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/display.system/DS05-04-msm8x25-display-overview.png)


#### （二）、FrameBuffer驱动程序分析

FrameBuffer通常作为LCD控制器或者其他显示设备的驱动，FrameBuffer驱动是一个字符设备，设备节点是/dev/fbX（Android 设备为/dev/graphics/fb0），主设备号为29，次设备号递增，用户可以将Framebuffer看成是显示内存的一个映像，将其映射到进程地址空间之后，就可以直接进行读写操作，而写操作可以立即反应在屏幕上。这种操作是抽象的，统一的。用户不必关心物理显存的位置、换页机制等等具体细节。这些都是由Framebuffer设备驱动来完成的。Framebuffer设备为上层应用程序提供系统调用，也为下一层的特定硬件驱动提供接口；那些底层硬件驱动需要用到这儿的接口来向系统内核注册它们自己。

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/display.system/DS05-05-mdss-driver-architecture.png)


Linux中的PCI设备可以将其控制寄存器映射到物理内存空间，而后，对这些控制寄存器的访问变成了对理内存的访问，因此，这些寄存器又被称为"memio"。一旦被映射到物理内存，Linux的普通进程就可以通过mmap将这些内存I/O映射到进程地址空间，这样就可以直接访问这些寄存器了。

FrameBuffer设备属于字符设备，采用了文件层—驱动层的接口方式，Linux为帧缓冲设备定义了驱动层的接口fb_info结构，在文件层上，用户调用file_operations的函数操作，间接调用fb_info中的fb_ops函数集来操作硬件。
##### 2.1、 Framebuffer数据结构

##### 2.1.1、 fb_info
fb_info是Linux为帧缓冲设备定义的驱动层接口。它不仅包含了底层函数，而且还有记录设备状态的数据。每个帧缓冲设备都与一个fb_info结构相对应。

``` cpp
[->kernel\include\linux\fb.h]
struct fb_info {
	atomic_t count;
	int node;
	int flags;
	struct mutex lock;		/* Lock for open/release/ioctl funcs */
	struct mutex mm_lock;		/* Lock for fb_mmap and smem_* fields */
	struct fb_var_screeninfo var;	/* Current var */
	struct fb_fix_screeninfo fix;	/* Current fix */
	struct fb_monspecs monspecs;	/* Current Monitor specs */
	struct work_struct queue;	/* Framebuffer event queue */
	struct fb_pixmap pixmap;	/* Image hardware mapper */
	struct fb_pixmap sprite;	/* Cursor hardware mapper */
	struct fb_cmap cmap;		/* Current cmap */
	struct list_head modelist;      /* mode list */
	struct fb_videomode *mode;	/* current mode */
	struct file *file;		/* current file node */

#ifdef CONFIG_FB_DEFERRED_IO
	struct delayed_work deferred_work;
	struct fb_deferred_io *fbdefio;
#endif

	struct fb_ops *fbops;
	struct device *device;		/* This is the parent */
	struct device *dev;		/* This is this fb device */
	int class_flag;                    /* private sysfs flags */
#ifdef CONFIG_FB_TILEBLITTING
	struct fb_tile_ops *tileops;    /* Tile Blitting */
#endif
	char __iomem *screen_base;	/* Virtual address */
	unsigned long screen_size;	/* Amount of ioremapped VRAM or 0 */ 
	void *pseudo_palette;		/* Fake palette of 16 colors */ 
#define FBINFO_STATE_RUNNING	0
#define FBINFO_STATE_SUSPENDED	1
	u32 state;			/* Hardware state i.e suspend */
	void *fbcon_par;                /* fbcon use-only private area */
	/* From here on everything is device dependent */
	void *par;
	/* we need the PCI or similar aperture base/size not
	   smem_start/size as smem_start may just be an object
	   allocated inside the aperture so may not actually overlap */
	struct apertures_struct {
		unsigned int count;
		struct aperture {
			resource_size_t base;
			resource_size_t size;
		} ranges[0];
	} *apertures;

	bool skip_vt_switch; /* no VT switch on suspend/resume required */
};
```
##### 2.1.2、fb_var_screeninfo

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/display.system/DS05-06-xres-yres.png)


fb_var_screeninfo：用于记录用户可修改的显示控制器参数，包括屏幕分辨率、每个像素点的比特数等

``` cpp
[->\include\uapi\linux\fb.h]
struct fb_var_screeninfo {  
    __u32 xres;         /* 行可见像素*/  
    __u32 yres;         /* 列可见像素*/  
    __u32 xres_virtual; /* 行虚拟像素*/  
    __u32 yres_virtual; /* 列虚拟像素*/  
    __u32 xoffset;      /* 水平偏移量*/  
    __u32 yoffset;      /* 垂直偏移量*/  
    __u32 bits_per_pixel;/*每个像素所占bit位数*/  
    __u32 grayscale;    /* 灰色刻度*/  
    struct fb_bitfield red; /* bitfield in fb mem if true color, */  
    struct fb_bitfield green;   /* else only length is significant */  
    struct fb_bitfield blue;  
    struct fb_bitfield transp;  /* transparency         */    
    __u32 nonstd;           /* != 0 Non standard pixel format */  
    __u32 activate;         /* see FB_ACTIVATE_*        */  
    __u32 height;           /* 图像高度*/  
    __u32 width;            /* 图像宽度*/  
    __u32 accel_flags;      /* (OBSOLETE) see fb_info.flags */  
    __u32 pixclock;         /* pixel clock in ps (pico seconds) */  
    __u32 left_margin;      /* time from sync to picture    */  
    __u32 right_margin;     /* time from picture to sync    */  
    __u32 upper_margin;     /* time from sync to picture    */  
    __u32 lower_margin;  
    __u32 hsync_len;        /* length of horizontal sync    */  
    __u32 vsync_len;        /* length of vertical sync  */  
    __u32 sync;         /* see FB_SYNC_*        */  
    __u32 vmode;            /* see FB_VMODE_*       */  
    __u32 rotate;           /* angle we rotate counter clockwise */  
    __u32 reserved[5];      /* Reserved for future compatibility */  
};  

```
##### 2.1.3、fb_fix_screeninfo
fb_fix_screeninfo：记录了用户不能修改的显示控制器的参数，这些参数是在驱动初始化时设置的

``` cpp
[->\include\uapi\linux\fb.h]
struct fb_fix_screeninfo {
	char id[16];			/* identification string eg "TT Builtin" */
	unsigned long smem_start;	/* Start of frame buffer mem */
					/* (physical address) */
	__u32 smem_len;			/* Length of frame buffer mem */
	__u32 type;			/* see FB_TYPE_*		*/
	__u32 type_aux;			/* Interleave for interleaved Planes */
	__u32 visual;			/* see FB_VISUAL_*		*/ 
	__u16 xpanstep;			/* zero if no hardware panning  */
	__u16 ypanstep;			/* zero if no hardware panning  */
	__u16 ywrapstep;		/* zero if no hardware ywrap    */
	__u32 line_length;		/* length of a line in bytes    */
	unsigned long mmio_start;	/* Start of Memory Mapped I/O   */
					/* (physical address) */
	__u32 mmio_len;			/* Length of Memory Mapped I/O  */
	__u32 accel;			/* Indicate to driver which	*/
					/*  specific chip/card we have	*/
	__u16 capabilities;		/* see FB_CAP_*			*/
	__u16 reserved[2];		/* Reserved for future compatibility */
};
```
fb_ops是提供给底层设备驱动的一个接口。当我们编写一个FrameBuffer的时候，就要依照Linux FrameBuffer编程的套路，填写fb_ops结构体。

``` cpp
[->\include\linux\fb.h]
struct fb_ops {
	/* open/release and usage marking */
	struct module *owner;
	int (*fb_open)(struct fb_info *info, int user);
	int (*fb_release)(struct fb_info *info, int user);

	/* For framebuffers with strange non linear layouts or that do not
	 * work with normal memory mapped access
	 */
	ssize_t (*fb_read)(struct fb_info *info, char __user *buf,
			   size_t count, loff_t *ppos);
	ssize_t (*fb_write)(struct fb_info *info, const char __user *buf,
			    size_t count, loff_t *ppos);

	/* checks var and eventually tweaks it to something supported,
	 * DO NOT MODIFY PAR */
	int (*fb_check_var)(struct fb_var_screeninfo *var, struct fb_info *info);

	/* set the video mode according to info->var */
	int (*fb_set_par)(struct fb_info *info);

	/* set color register */
	int (*fb_setcolreg)(unsigned regno, unsigned red, unsigned green,
			    unsigned blue, unsigned transp, struct fb_info *info);

	/* set color registers in batch */
	int (*fb_setcmap)(struct fb_cmap *cmap, struct fb_info *info);

	/* blank display */
	int (*fb_blank)(int blank, struct fb_info *info);

	/* pan display */
	int (*fb_pan_display)(struct fb_var_screeninfo *var, struct fb_info *info);

	/* Draws a rectangle */
	void (*fb_fillrect) (struct fb_info *info, const struct fb_fillrect *rect);
	/* Copy data from area to another */
	void (*fb_copyarea) (struct fb_info *info, const struct fb_copyarea *region);
	/* Draws a image to the display */
	void (*fb_imageblit) (struct fb_info *info, const struct fb_image *image);

	/* Draws cursor */
	int (*fb_cursor) (struct fb_info *info, struct fb_cursor *cursor);

	/* Rotates the display */
	void (*fb_rotate)(struct fb_info *info, int angle);

	/* wait for blit idle, optional */
	int (*fb_sync)(struct fb_info *info);

	/* perform fb specific ioctl (optional) */
	int (*fb_ioctl)(struct fb_info *info, unsigned int cmd,
			unsigned long arg);

	/* perform fb specific ioctl v2 (optional) - provides file param */
	int (*fb_ioctl_v2)(struct fb_info *info, unsigned int cmd,
			unsigned long arg, struct file *file);

	/* Handle 32bit compat ioctl (optional) */
	int (*fb_compat_ioctl)(struct fb_info *info, unsigned cmd,
			unsigned long arg);

	/* Handle 32bit compat ioctl (optional) */
	int (*fb_compat_ioctl_v2)(struct fb_info *info, unsigned cmd,
			unsigned long arg, struct file *file);

	/* perform fb specific mmap */
	int (*fb_mmap)(struct fb_info *info, struct vm_area_struct *vma);

	/* get capability given var */
	void (*fb_get_caps)(struct fb_info *info, struct fb_blit_caps *caps,
			    struct fb_var_screeninfo *var);

	/* teardown any resources to do with this framebuffer */
	void (*fb_destroy)(struct fb_info *info);

	/* called at KDB enter and leave time to prepare the console */
	int (*fb_debug_enter)(struct fb_info *info);
	int (*fb_debug_leave)(struct fb_info *info);
};
```
##### 2.2、Framebuffer驱动注册过程
在系统启动时，内核调用所有注册驱动程序的驱动程序初始化函数。 为了帧缓冲区驱动程序，调用mdss_fb_init。 mdss_fb_init注册mdss_fb_driver。驱动在mdss_fb.c文件中注册。

``` cpp
[-\drivers\video\msm\mdss\mdss_fb.c]
static struct platform_driver mdss_fb_driver = {
	.probe = mdss_fb_probe,
	.remove = mdss_fb_remove,
	.suspend = mdss_fb_suspend,
	.resume = mdss_fb_resume,
	.shutdown = mdss_fb_shutdown,
	.driver = {
		.name = "mdss_fb",
		.of_match_table = mdss_fb_dt_match,
		.pm = &mdss_fb_pm_ops,
	},
};
```

在调用init之后，内核调用每个平台驱动程序的探测函数。 在调用mdss_fb_probe时，函数执行资源分配并调用mdss_fb_register。 可以有多个帧缓冲区（fb）设备（节点）。 该驱动程序通过调用mdss_fb_register来注册各个fb设备，后者又调用register_framebuffer。 HDMI和主显示器是各个fb设备的例子。 以下操作已注册：

首先看一下mdss_fb_probe()函数

``` cpp
[->\drivers\video\msm\mdss\mdss_fb.c]
static int mdss_fb_probe(struct platform_device *pdev)
{
	struct msm_fb_data_type *mfd = NULL;
	struct mdss_panel_data *pdata;
	struct fb_info *fbi;
	int rc;

	pdata = dev_get_platdata(&pdev->dev);
	
	/*
	 * alloc framebuffer info + par data
	 */
	fbi = framebuffer_alloc(sizeof(struct msm_fb_data_type), NULL);

	mfd = (struct msm_fb_data_type *)fbi->par;
	mfd->key = MFD_KEY;
	mfd->fbi = fbi;
	mfd->panel_info = &pdata->panel_info;
	mfd->panel.type = pdata->panel_info.type;
	mfd->panel.id = mfd->index;
	mfd->fb_page = MDSS_FB_NUM;
	mfd->index = fbi_list_index;
	mfd->mdp_fb_page_protection = MDP_FB_PAGE_PROTECTION_WRITECOMBINE;

	mfd->ext_ad_ctrl = -1;
	......

	platform_set_drvdata(pdev, mfd);

	rc = mdss_fb_register(mfd);
	
	......

	return rc;
}

```
首先调用framebuffer_alloc()函数返回一个fb_info 结构体，然后调用mdss_fb_register(mfd)
``` cpp
[->\drivers\video\msm\mdss\mdss_fb.c]
static int mdss_fb_register(struct msm_fb_data_type *mfd)
{
	int ret = -ENODEV;
	int bpp;
	char panel_name[20];
	struct mdss_panel_info *panel_info = mfd->panel_info;
	struct fb_info *fbi = mfd->fbi;
	struct fb_fix_screeninfo *fix;
	struct fb_var_screeninfo *var;
	int *id;

	/*
	 * fb info initialization
	 */
	fix = &fbi->fix;
	var = &fbi->var;

	fix->type_aux = 0;	/* if type == FB_TYPE_INTERLEAVED_PLANES */
	fix->visual = FB_VISUAL_TRUECOLOR;	/* True Color */
	fix->ywrapstep = 0;	/* No support */
	fix->mmio_start = 0;	/* No MMIO Address */
	fix->mmio_len = 0;	/* No MMIO Address */
	fix->accel = FB_ACCEL_NONE;/* FB_ACCEL_MSM needes to be added in fb.h */

	var->xoffset = 0,	/* Offset from virtual to visible */
	var->yoffset = 0,	/* resolution */
	var->grayscale = 0,	/* No graylevels */
	var->nonstd = 0,	/* standard pixel format */
	var->activate = FB_ACTIVATE_VBL,	/* activate it at vsync */
	var->height = -1,	/* height of picture in mm */
	var->width = -1,	/* width of picture in mm */
	var->accel_flags = 0,	/* acceleration flags */
	var->sync = 0,	/* see FB_SYNC_* */
	var->rotate = 0,	/* angle we rotate counter clockwise */
	mfd->op_enable = false;

	switch (mfd->fb_imgType) {
	case MDP_RGB_565:
		......
		var->transp.offset = 0;
		var->transp.length = 0;
		bpp = 2;
		break;

	case MDP_RGB_888:
		......
		var->transp.offset = 0;
		var->transp.length = 0;
		bpp = 3;
		break;

	case MDP_ARGB_8888:
		......
		var->transp.offset = 0;
		var->transp.length = 8;
		bpp = 4;
		break;

	case MDP_RGBA_8888:
		......
		var->transp.offset = 24;
		var->transp.length = 8;
		bpp = 4;
		break;

	case MDP_YCRYCB_H2V1:
		......
		var->transp.offset = 0;
		var->transp.length = 0;
		bpp = 2;
		break;

	default:
		return ret;
	}

	mdss_panelinfo_to_fb_var(panel_info, var);

	fix->type = panel_info->is_3d_panel;
	if (mfd->mdp.fb_stride)
		fix->line_length = mfd->mdp.fb_stride(mfd->index, var->xres,
							bpp);
	else
		fix->line_length = var->xres * bpp;

	var->xres_virtual = var->xres;
	var->yres_virtual = panel_info->yres * mfd->fb_page;
	var->bits_per_pixel = bpp * 8;	/* FrameBuffer color depth */

	/*
	 * Populate smem length here for uspace to get the
	 * Framebuffer size when FBIO_FSCREENINFO ioctl is called.
	 */
	fix->smem_len = PAGE_ALIGN(fix->line_length * var->yres) * mfd->fb_page;

	/* id field for fb app  */
	id = (int *)&mfd->panel;

	snprintf(fix->id, sizeof(fix->id), "mdssfb_%x", (u32) *id);

	fbi->fbops = &mdss_fb_ops;
	fbi->flags = FBINFO_FLAG_DEFAULT;
	fbi->pseudo_palette = mdss_fb_pseudo_palette;

	mfd->ref_cnt = 0;
	mfd->panel_power_state = MDSS_PANEL_POWER_OFF;
	mfd->dcm_state = DCM_UNINIT;

	if (mdss_fb_alloc_fbmem(mfd))
		pr_warn("unable to allocate fb memory in fb register\n");

	......

	ret = fb_alloc_cmap(&fbi->cmap, 256, 0);
	if (ret)
		pr_err("fb_alloc_cmap() failed!\n");

	if (register_framebuffer(fbi) < 0) {
		fb_dealloc_cmap(&fbi->cmap);
		mfd->op_enable = false;
		return -EPERM;
	}

	snprintf(panel_name, ARRAY_SIZE(panel_name), "mdss_panel_fb%d",
		mfd->index);
	mdss_panel_debugfs_init(panel_info, panel_name);
	pr_info("FrameBuffer[%d] %dx%d registered successfully!\n", mfd->index,
					fbi->var.xres, fbi->var.yres);

	return 0;
}

```

任何一个特定硬件Framebuffer驱动在初始化时都必须向fbmem.c注册，FrameBuffer模块提供了驱动注册接口函数register_framebuffer：

``` cpp
[->\drivers\video\fbdev\core\fbmem.c]
int
register_framebuffer(struct fb_info *fb_info)
{
	int ret;

	mutex_lock(&registration_lock);
	ret = do_register_framebuffer(fb_info);
	mutex_unlock(&registration_lock);

	return ret;
}
```
参数fb_info描述特定硬件的FrameBuffer驱动信息

``` cpp
[->\drivers\video\fbdev\core\fbmem.c]
static int do_register_framebuffer(struct fb_info *fb_info)  
{  
    int i;  
    struct fb_event event;  
    struct fb_videomode mode;  
    if (fb_check_foreignness(fb_info))  
        return -ENOSYS;  
    //根据当前注册的fb_info的apertures属性从FrameBuffer驱动数组registered_fb中查询是否存在冲突  
    do_remove_conflicting_framebuffers(fb_info->apertures, fb_info->fix.id,  
                     fb_is_primary_device(fb_info));  
    //判断已注册的驱动是否超过32个FrameBuffer驱动  
    if (num_registered_fb == FB_MAX)  
        return -ENXIO;  
    //增加已注册的驱动个数  
    num_registered_fb++;  
    //从数组registered_fb中查找空闲元素，用于存储当前注册的fb_info  
    for (i = 0 ; i < FB_MAX; i++)  
        if (!registered_fb[i])  
            break;  
    //将当前注册的fb_info在数组registered_fb中的索引位置保存到fb_info->node  
    fb_info->node = i;  
    //初始化当前注册的fb_info的成员信息  
    atomic_set(&fb_info->count, 1);  
    mutex_init(&fb_info->lock);  
    mutex_init(&fb_info->mm_lock);  
    //在/dev目录下创建一个fbx的设备文件，次设备号就是该fb_info在数组registered_fb中的索引  
    fb_info->dev = device_create(fb_class, fb_info->device,MKDEV(FB_MAJOR, i), NULL, "fb%d", i);  
    if (IS_ERR(fb_info->dev)) {  
        printk(KERN_WARNING "Unable to create device for framebuffer %d; errno = %ld\n", i, PTR_ERR(fb_info->dev));  
        fb_info->dev = NULL;  
    } else  
        //初始化fb_info  
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
    //将特定硬件对应的fb_info注册到registered_fb数组中  
    registered_fb[i] = fb_info;  
    event.info = fb_info;  
    if (!lock_fb_info(fb_info))  
        return -ENODEV;  
    //使用Linux事件通知机制发送一个FrameBuffer注册事件FB_EVENT_FB_REGISTERED  
    fb_notifier_call_chain(FB_EVENT_FB_REGISTERED, &event);  
    unlock_fb_info(fb_info);  
    return 0;  
}  
```
注册过程就是将指定的设备驱动信息fb_info存放到registered_fb数组中。因此在注册具体的fb_info时，首先要构造一个fb_info数据结构，并初始化该数据结构，该结构用于描述一个特定的FrameBuffer驱动。

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/display.system/DS05-07-fb-device-create.jpg)


#### （三）、MDP driver
MDP也被注册为平台驱动程序。 mdp3_driver_init执行驱动程序init。

``` cpp
[->\drivers\video\msm\mdss\mdp3.c]
static struct platform_driver mdp3_driver = {
	.probe = mdp3_probe,
	.remove = mdp3_remove,
	.suspend = mdp3_suspend,
	.resume = mdp3_resume,
	.shutdown = NULL,
	.driver = {
		.name = "mdp3",
		.of_match_table = mdp3_dt_match,
		.pm             = &mdp3_pm_ops,
	},
};

static int __init mdp3_driver_init(void)
{
	int ret;

	ret = platform_driver_register(&mdp3_driver);
	if (ret) {
		pr_err("register mdp3 driver failed!\n");
		return ret;
	}

	return 0;
}

static int mdp3_probe(struct platform_device *pdev)
{
	int rc;
	static struct msm_mdp_interface mdp3_interface = {
	.init_fnc = mdp3_init,
	.fb_mem_get_iommu_domain = mdp3_fb_mem_get_iommu_domain,
	.panel_register_done = mdp3_panel_register_done,
	.fb_stride = mdp3_fb_stride,
	.check_dsi_status = mdp3_check_dsi_ctrl_status,
	};

	struct mdp3_intr_cb underrun_cb = {
		.cb = mdp3_dma_underrun_intr_handler,
		.data = NULL,
	};

	pr_debug("%s: START\n", __func__);
	if (!pdev->dev.of_node) {
		pr_err("MDP driver only supports device tree probe\n");
		return -ENOTSUPP;
	}

	if (mdp3_res) {
		pr_err("MDP already initialized\n");
		return -EINVAL;
	}

	mdp3_res = devm_kzalloc(&pdev->dev, sizeof(struct mdp3_hw_resource),
				GFP_KERNEL);
	if (mdp3_res == NULL)
		return -ENOMEM;

	pdev->id = 0;
	mdp3_res->pdev = pdev;
	mutex_init(&mdp3_res->res_mutex);
	spin_lock_init(&mdp3_res->irq_lock);
	platform_set_drvdata(pdev, mdp3_res);
	atomic_set(&mdp3_res->active_intf_cnt, 0);
	mutex_init(&mdp3_res->reg_bus_lock);
	INIT_LIST_HEAD(&mdp3_res->reg_bus_clist);

	mdp3_res->mdss_util = mdss_get_util_intf();
	if (mdp3_res->mdss_util == NULL) {
		pr_err("Failed to get mdss utility functions\n");
		rc =  -ENODEV;
		goto get_util_fail;
	}
	mdp3_res->mdss_util->get_iommu_domain = mdp3_get_iommu_domain;
	mdp3_res->mdss_util->iommu_attached = is_mdss_iommu_attached;
	mdp3_res->mdss_util->iommu_ctrl = mdp3_iommu_ctrl;
	mdp3_res->mdss_util->bus_scale_set_quota = mdp3_bus_scale_set_quota;
	mdp3_res->mdss_util->panel_intf_type = mdp3_panel_intf_type;
	mdp3_res->mdss_util->dyn_clk_gating_ctrl =
		mdp3_dynamic_clock_gating_ctrl;
	mdp3_res->mdss_util->panel_intf_type = mdp3_panel_intf_type;
	mdp3_res->mdss_util->panel_intf_status = mdp3_panel_get_intf_status;
	rc = mdp3_parse_dt(pdev);
	if (rc)
		goto probe_done;

	rc = mdp3_res_init();
	if (rc) {
		pr_err("unable to initialize mdp3 resources\n");
		goto probe_done;
	}

	mdp3_res->fs_ena = false;
	mdp3_res->fs = devm_regulator_get(&pdev->dev, "vdd");
	if (IS_ERR_OR_NULL(mdp3_res->fs)) {
		pr_err("unable to get mdss gdsc regulator\n");
		return -EINVAL;
	}

	rc = mdp3_debug_init(pdev);
	if (rc) {
		pr_err("unable to initialize mdp debugging\n");
		goto probe_done;
	}

	pm_runtime_set_autosuspend_delay(&pdev->dev, AUTOSUSPEND_TIMEOUT_MS);
	if (mdp3_res->idle_pc_enabled) {
		pr_debug("%s: Enabling autosuspend\n", __func__);
		pm_runtime_use_autosuspend(&pdev->dev);
	}
	/* Enable PM runtime */
	pm_runtime_set_suspended(&pdev->dev);
	pm_runtime_enable(&pdev->dev);

	if (!pm_runtime_enabled(&pdev->dev)) {
		rc = mdp3_footswitch_ctrl(1);
		if (rc) {
			pr_err("unable to turn on FS\n");
			goto probe_done;
		}
	}

	rc = mdp3_check_version();
	if (rc) {
		pr_err("mdp3 check version failed\n");
		goto probe_done;
	}
	rc = mdp3_register_sysfs(pdev);
	if (rc)
		pr_err("unable to register mdp sysfs nodes\n");

	rc = mdss_fb_register_mdp_instance(&mdp3_interface);
	if (rc)
		pr_err("unable to register mdp instance\n");

	rc = mdp3_set_intr_callback(MDP3_INTR_LCDC_UNDERFLOW,
					&underrun_cb);
	if (rc)
		pr_err("unable to configure interrupt callback\n");

	rc = mdss_smmu_init(mdss_res, &pdev->dev);
	if (rc)
		pr_err("mdss smmu init failed\n");

	mdp3_res->mdss_util->mdp_probe_done = true;
	pr_debug("%s: END\n", __func__);
	return rc;
}
```
内核调用MDP探测函数mdp3_probe。 在探测器上，驱动程序从设备树中获取面板信息，并通过调用mdp3_parse_dt来解析信息。 在mdp3_ctrl_on期间，MDP驱动程序调用mdp3_ctrl_res_req_clk并请求MDP和Vsync时钟。 在mdp3_ctrl_off期间，驱动程序请求关闭MDP和Vsync时钟。

#### （四）、DSI controller driver （lcd驱动 dsi）

msm_dsi_v2_driver_init执行驱动程序初始化。 msm_dsi_v2_driver_init调用 msm_dsi_v2_register_driver注册驱动程序。
总体时序图：

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/display.system/DS05-08-dsi-host-v2.png)


``` cpp
[->\drivers\video\msm\mdss\dsi_host_v2.c]
static struct platform_driver msm_dsi_v2_driver = {
	.probe = msm_dsi_probe,
	.remove = msm_dsi_remove,
	.shutdown = NULL,
	.driver = {
		.name = "msm_dsi_v2",
		.of_match_table = msm_dsi_v2_dt_match,
	},
};

static int msm_dsi_v2_register_driver(void)
{
	return platform_driver_register(&msm_dsi_v2_driver);
}
static int msm_dsi_probe(struct platform_device *pdev)
{
	struct dsi_interface intf;
	char panel_cfg[MDSS_MAX_PANEL_LEN];
	struct mdss_dsi_ctrl_pdata *ctrl_pdata = NULL;
	int rc = 0;
	struct device_node *dsi_pan_node = NULL;
	bool cmd_cfg_cont_splash = false;
	struct resource *mdss_dsi_mres;
	int i;

	pr_debug("%s\n", __func__);

	rc = msm_dsi_init();

	pdev->id = 0;

	ctrl_pdata = platform_get_drvdata(pdev);
	if (!ctrl_pdata) {
		ctrl_pdata = devm_kzalloc(&pdev->dev,
			sizeof(struct mdss_dsi_ctrl_pdata), GFP_KERNEL);
		platform_set_drvdata(pdev, ctrl_pdata);
	}

	ctrl_pdata->mdss_util = mdss_get_util_intf();
	if (mdp3_res->mdss_util == NULL) {
		pr_err("Failed to get mdss utility functions\n");
		return -ENODEV;
	}

	mdss_dsi_mres = platform_get_resource(pdev, IORESOURCE_MEM, 0);
	if (!mdss_dsi_mres) {
	} else {
		dsi_host_private->dsi_reg_size = resource_size(mdss_dsi_mres);
		dsi_host_private->dsi_base = ioremap(mdss_dsi_mres->start,
						dsi_host_private->dsi_reg_size);
	}

	mdss_dsi_mres = platform_get_resource(pdev, IORESOURCE_IRQ, 0);
	
	rc = of_platform_populate(pdev->dev.of_node, NULL, NULL, &pdev->dev);
	

	/* DSI panels can be different between controllers */
	rc = dsi_get_panel_cfg(panel_cfg);
	
	/* find panel device node */
	dsi_pan_node = dsi_find_panel_of_node(pdev, panel_cfg);
	
	cmd_cfg_cont_splash = mdp3_panel_get_boot_cfg() ? true : false;

	rc = mdss_dsi_panel_init(dsi_pan_node, ctrl_pdata, cmd_cfg_cont_splash);
	
	rc = dsi_ctrl_config_init(pdev, ctrl_pdata);
	

	msm_dsi_parse_lane_swap(pdev->dev.of_node, &(ctrl_pdata->dlane_swap));

	for (i = 0;  i < DSI_MAX_PM; i++) {
		rc = msm_dsi_io_init(pdev, &(ctrl_pdata->power_data[i]));
		
	}

	pr_debug("%s: Dsi Ctrl->0 initialized\n", __func__);

	dsi_host_private->dis_dev = pdev->dev;
	intf.on = msm_dsi_on;
	intf.off = msm_dsi_off;
	intf.cont_on = msm_dsi_cont_on;
	intf.clk_ctrl = msm_dsi_clk_ctrl;
	intf.op_mode_config = msm_dsi_op_mode_config;
	intf.index = 0;
	intf.private = NULL;
	dsi_register_interface(&intf);

	msm_dsi_debug_init();

	msm_dsi_ctrl_init(ctrl_pdata);

	rc = msm_dsi_irq_init(&pdev->dev, mdss_dsi_mres->start,
					   ctrl_pdata);
	
	rc = dsi_panel_device_register_v2(pdev, ctrl_pdata);
	
	pr_debug("%s success\n", __func__);
	return 0;
}
```
内核调用msm_dsi_probe。 面板被检测到。 msm_dsi_probe调用mdss_dsi_panel_init函数。 mdss_dsi_panel_init调用mdss_panel_parse_dt来获取面板参数。
MDP驱动程序使用该事件与DSI驱动程序进行通信。 DSI驱动程序具有mdss_dsi_event_handler，这是MDP核心事件的回调处理程序。 mdss_panel.h定义了MDP核心事件。

``` cpp
[->\drivers\video\msm\mdss\mdss_panel.h]
enum mdss_intf_events {
	MDSS_EVENT_RESET = 1,
	MDSS_EVENT_LINK_READY,
	MDSS_EVENT_UNBLANK,
	MDSS_EVENT_PANEL_ON,
	MDSS_EVENT_POST_PANEL_ON,
	MDSS_EVENT_BLANK,
	MDSS_EVENT_PANEL_OFF,
	MDSS_EVENT_CLOSE,
	MDSS_EVENT_SUSPEND,
	MDSS_EVENT_RESUME,
	MDSS_EVENT_CHECK_PARAMS,
	MDSS_EVENT_CONT_SPLASH_BEGIN,
	MDSS_EVENT_CONT_SPLASH_FINISH,
	MDSS_EVENT_PANEL_UPDATE_FPS,
	MDSS_EVENT_FB_REGISTERED,
	MDSS_EVENT_PANEL_CLK_CTRL,
	MDSS_EVENT_DSI_CMDLIST_KOFF,
	MDSS_EVENT_ENABLE_PARTIAL_ROI,
	MDSS_EVENT_DSC_PPS_SEND,
	MDSS_EVENT_DSI_STREAM_SIZE,
	MDSS_EVENT_DSI_UPDATE_PANEL_DATA,
	MDSS_EVENT_REGISTER_RECOVERY_HANDLER,
	MDSS_EVENT_REGISTER_MDP_CALLBACK,
	MDSS_EVENT_DSI_PANEL_STATUS,
	MDSS_EVENT_DSI_DYNAMIC_SWITCH,
	MDSS_EVENT_DSI_RECONFIG_CMD,
	MDSS_EVENT_DSI_RESET_WRITE_PTR,
	MDSS_EVENT_PANEL_TIMING_SWITCH,
	MDSS_EVENT_MAX,
};
```
在msm_dsi_on期间，通过调用msm_dsi_clk_enable打开DSI时钟。 在msm_dsi_off期间，通过调用msm_dsi_clk_disable关闭clks。

#### （五）、 Panel driver （面板 dsi）
MDSS : Multimedia Display sub system 
DSI: Display Serial Interface

> qcom,mdss-dsi-force-clock-lane-hs;          // faulse ：clock每帧回lp11  
> ture: clock不回 qcom,mdss-dsi-hfp-power-mode;               // data 每行回lp11,对应的hfp要修改成300以上

面板信息位于kernel\arch\arm\boot\dts\中的.dtsi文件中。 这包含所有面板特定的命令，例如on，off和reset（mdss-dsi-on-command，mdss-dsi-off-command，mdss-dsi-reset-sequence），BL控制和其他面板 独立参数。
例如：
msm8610-mdss.dtsi （文件名通常为 msmxxx-mdss.dtsi 指定了mdss 的 mdp 和 dsi）
##### 5.1、.dtsi文件解析
``` cpp

mdss_mdp: qcom,mdss_mdp@fd900000 {
            compatible = "qcom,mdss_mdp3";  // 对应mdss驱动 mdss_mdp.c
----------
  mdss_dsi0: qcom,mdss_dsi@fdd00000 {
        compatible = "qcom,msm-dsi-v2";      // 对应dsi解析驱动 dsi_host_v2.c

或者

  mdss_dsi0: qcom,mdss_dsi_ctrl0@1a94000 {
        compatible = "qcom,mdss-dsi-ctrl";  // 对应dsi解析驱动 mdss_dsi.c
```

通过下面函数向 mdss_fb.c 注册了fb_info结构  (包含在mdss_dsi_ctrl_pdata结构中)

``` cpp
drivers\video\msm\mdss\dsi_host_v2.c （lcd驱动 dsi）
dsi_panel_device_register_v2(struct platform_device *dev,struct mdss_dsi_ctrl_pdata *ctrl_pdata)


static const struct of_device_id msm_dsi_v2_dt_match[] = {
    {.compatible = "qcom,msm-dsi-v2"},
    {}
};

或者 
drivers\video\msm\mdss\mdss_dsi.c

```

msm8610-asus.dts （指定mdp中的哪一个配置） 
通常在dts文件的 mdss_dsi0 lab里面通过 qcom,dsi-pref-prim-pan 属性 指定使用哪一个lcd配置

``` cpp
&mdss_dsi0 {
        qcom,dsi-pref-prim-pan = <&dsi_fl10802_fwvga_vid>;
};
```
dsi-panel-fl10802-fwvga-video.dtsi

``` cpp
&mdss_mdp {
    dsi_fl10802_fwvga_vid: qcom,mdss_dsi_fl10802_fwvga_video {
        qcom,mdss-dsi-panel-name = "fl10802 fwvga video mode dsi panel";
        qcom,mdss-dsi-drive-ic = "fl10802";
        qcom,mdss-dsi-panel-controller = <&mdss_dsi0>;
        qcom,mdss-dsi-panel-type = "dsi_video_mode";
        qcom,mdss-dsi-panel-destination = "display_1";
        ...
        }
```


##### 5.2、mdss_mdp 和 mdss_dsi0 的关系
mdss_mdp 相当于一个数组，里面定义了很多不同lcd显示屏的配置项包括分辨率等等

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/display.system/DS05-09-mdss_mdp-mdss_dsi0.jpg)



面板驱动程序可以根据连接的实际面板数量具有多个节点。
mdss_register_panel注册面板驱动程序：
msm_dsi_probe→dsi_panel_device_register_v2→mdss_register_panel→
of_platform_device_create
以下面板控制功能可用并在mdss_dsi_panel_init中初始化：
ctrl_pdata→on = mdss_dsi_panel_on;
ctrl_pdata→off = mdss_dsi_panel_off;
ctrl_pdata→panel_data.set_backlight = mdss_dsi_panel_bl_ctrl;
  例如，在mdp3_ctrl.c中，函数mdp3_ctrl_on（）会调用以下DSI处理程序
MDSS_EVENT_UNBLANK和MDSS_EVENT_PANEL_ON事件如下所示：
rc = panel→event_handler（panel，MDSS_EVENT_UNBLANK，NULL）;
rc | =面板→event_handler（面板，MDSS_EVENT_PANEL_ON，NULL）;

##### 5.3、通过内核接口打开和关闭显示器

##### 5.3.1、启动

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/display.system/DS05-10-display-subsystem-on-bootup.png)


按照上面的部分所述注册设备后：
mdss_fb_open→mdss_fb_blank_sub→pdata→on（面板ON功能）
注意：对于命令模式面板，如果面板在启动时已打开，则会跳过该启动序列 以避免关闭/ -on的文物。
##### 5.3.2、暂停/恢复
挂起/恢复时，调用fb驱动程序挂起/恢复和MDP驱动程序挂起/恢复。 fb驱动程序依次调用面板驱动程序的开/关功能。

##### 5.3.2.1、暂停序列

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/display.system/DS05-11-suspend sequence.png)


Kernelcall→mdss_fb_release_all→mdss_fb_blank→mdss_fb_blank_sub→mdp3_ctrl_off→
mdp3_ctrl_off发送两个事件：
1. MDSS_EVENT_PANEL_OFF - dsi_event_handler接收事件。 当事件发生时
    接收到时，调用小组关闭序列。
2. MDSS_EVENT_BLANK - 该事件由调用的dsi_event_handler处理
mdss_dsi_off。
* Kernelcall→mdp3_suspend - 未使用
##### 5.3.2.2、恢复序列

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/display.system/DS05-12-resume sequence.png)


Kernelcall→mdss_fb_blank→mdss_fb_blank_sub→mdp3_ctrl_on→
mdp3_ctrl_on发送两个事件：
1. MDSS_EVENT_UNBLANK - dsi_event_handler接收事件。 当事件发生时
收到，DSI-on被调用。
2. MDSS_EVENT_PANEL_ON - 事件由dsi_event_handler处理，dsi_event_handler发送
面板上的序列。
Kernelcall→mdp3_resume - 未使用

##### 5.4、图像更新到面板

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianok/PicGo/master/display.system/DS05-13-flow for display subsystem commit.png)


 用户必须确保MSMFB_OVERLAY_SET IOCTL在呼叫之前至少被调用一次 到MSMFB_OVERLAY_PLAY IOCLT（例如ioctl(fd, MSMFB_OVERLAY_PLAY, &od)）。MSMFB_OVERLAY_PLAY队列缓冲区 显示在面板上。 用户使用MSMFB_DISPLAY_COMMIT调用fb IOCLT。这会启动一个呼叫
 mdss_fb_display_commit 并安排工作队列。此工作队列处理程序调用
 msm_fb_pan_display_ex，然后调用mdp3_ctrl_pan_display。
 mdss_fb_ioctl→（MSMFB_DISPLAY_COMMIT）→mdss_fb_display_commit→
 mdss_fb_pan_display_ex→计划工作mdss_fb_commit_wq_handler
 msm_fb_commit_wq_handler→mdp3_ctrl_display_commit_kickoff→mdp3_dmap_update
一次呼叫后，可以有多个对PLAY和COMMIT IOCLT的呼叫 MSMFB_OVERLAY_SET IOCTL。面板更新完成并且设备完成后
需要进入挂起或关闭状态，用户可以调用MSMFB_OVERLAY_UNSET。

注意：工作队列架构正在被即将发布的线程取代，
这些文件未被捕获。视频和命令模式面板的面板更新略有不同。为了在命令模式面板中，mdp3_dmap_update函数会等待，直到前一个图像更新为止完成使用MDP DMA开始新帧更新

if（dma-> output_config.out_sel == MDP3_DMA_OUTPUT_SEL_DSI_CMD）{
cb_type = MDP3_DMA_CALLBACK_TYPE_DMA_DONE;
如果（intf-> active）
wait_for_completion_killable（DMA-> dma_comp）;

对于视频面板，在DMA触发后，mdp3_dmap_update等待vsync。
#### （六）、Msm8610  lcd driver 内核初始化分析

请参考[【msm8610 lcd driver code analysis】](https://blog.csdn.net/ic_soc_arm_robin/article/details/12949347)

#### （七）、参考资料(特别感谢各位前辈的分析和图示)：
**(๑乛◡乛๑) 、（ ͡° ͜ʖ ͡°）、（ಡωಡ）！！！累~~~**
**时至今日，终于完整的分析了Android Display System 总体框架流程，不禁感叹计算机世界的博大精深，在这个系列的分析中历练了如何拆解分析一个庞大复杂的模块、学习收获良多，同时也了解了自身知识的欠缺，由于涉及知识较多较广，博主也未能完全吃透，其中分析有误的地方还请各位见谅。所谓路漫漫其修远兮，吾将上下而求索。**
**Todo：专注于Linux && Android Multimedia（Camera、Video、Audio、Display）系统分析与研究，Multimedia System 任还有许多未解之惑，需恶补Linux内核知识，少年，加油（➽➽➽）**


#### （八）、参考资料(特别感谢各位前辈的分析和图示)：
[Graphics Stack Update](https://s3.amazonaws.com/connect.linaro.org/bkk16/Presentations/Wednesday/BKK16-315.pdf)
[高通Android平台-应用空间操作framebuffer dump LCD总结](https://blog.csdn.net/eliot_shao/article/details/74926010)
[msm8610 lcd driver code analysis](https://blog.csdn.net/ic_soc_arm_robin/article/details/12949347)
[linux qcom LCD framwork](https://blog.csdn.net/u012719256/article/details/52096727)
[Qualcomm平台 display bring up 过程详解](https://blog.csdn.net/weijory/article/details/69391838)
[高通8x25平台display模块总结](https://blog.csdn.net/wlwl0071986/article/details/8247443)
[Android 中的 framebuffer](https://blog.csdn.net/sfrysh/article/details/7305253)
[FrameBuffer驱动程序分析](https://blog.csdn.net/yangwen123/article/details/12096483)
[Android Framebuffer介绍及使用](http://www.wxtlife.com/2017/06/07/Android-framebuffer/)
[Android 图形系统](https://www.wolfcstech.com/categories/Android-%E5%9B%BE%E5%BD%A2%E7%B3%BB%E7%BB%9F/)

