---
title: Android P Graphics System（一）：Android Graphics精彩世界
cover: https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/hexo.themes/bing-wallpaper-2018.04.36.jpg
categories:
  - Graphics
tags:
  - Android
  - Graphics
toc: true
abbrlink: 20190508
date: 2019-05-08 09:25:00
---

--------------------------------------------------------------------------------

注：文章都是通过阅读各位前辈总结的资料 Android 9.0 && Linux（Kernel 3.18）Qualcomm平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除（◕‿◕），转载请注明出处（©Qualcomm ©Android @Linux 版权所有），谢谢。
（==**文章基于 Kernel-3.18**==）&&（==**文章基于 Android 9.0**==）
 [【开发板 Intrinsyc Open-Q™ 820 µSOM Development Kit】](https://www.intrinsyc.com/snapdragon-embedded-development-kits/open-q-820-usom-development-kit/)
[【开发板 Android 9.0 && Linux（Kernel 3.18）源码链接】](https://gitlab.com/zhoujinjiann/apq8096_la.um.7.5.r1-03100-8x96.0_p_v5.0)

正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢！！！

[（1）【高通Android平台-应用空间操作framebuffer dump LCD总结】](https://blog.csdn.net/eliot_shao/article/details/74926010) 


--------------------------------------------------------------------------------
==源码（部分）==：
[【测试程序源码提交记录】](https://gitlab.com/zhoujinjiann/apq8096_la.um.7.5.r1-03100-8x96.0_p_v5.0/commit/714c0071b4bc786d00e7a6bcd08face6a28d6c0c) 

>AMS、WMS、SurfaceFlinger test App（Android Apk Java层）

- G:\android9.0\packages\apps\TestViewportRed
- G:\android9.0\packages\apps\TestViewportGreen
- G:\android9.0\packages\apps\TestViewportBlue

> SurfaceFlinger test App（Android Native层）

- G:\android9.0\frameworks\native\services\surfaceflinger\SurfaceflingerTestsRed
- G:\android9.0\frameworks\native\services\surfaceflinger\SurfaceflingerTestsGreen
- G:\android9.0\frameworks\native\services\surfaceflinger\SurfaceflingerTestsBlue

>OpenGLES test App（Android OpenGLES Native层）

- G:\android9.0\frameworks\native\opengl\tests\OpenGLESTexturesRGB


>FrameBuffer test App （Kernel 用户层）

- G:\android9.0\hardware\qcom\display\FrameBufferTest

--------------------------------------------------------------------------------

#### (一)、FrameBuffer test App （Kernel 用户层）

#### （1）、直接操作framebuffer显示图像
##### 1.1、源代码
**FrameBufferTest.c**

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
    fbfd = open("/dev/graphics/fb0", O_RDWR,0); // 打开Frame Buffer设备  
    if (fbfd < 0) {  
        printf("Error: cannot open framebuffer device.%x\n",fbfd);  
        exit(1);  
    }  
    printf("The framebuffer device was opened successfully.\n"); 
    // Get fixed screen information  
    if (ioctl(fbfd, FBIOGET_FSCREENINFO, &finfo)) { // 获取设备固有信息  
        printf("Error reading fixed information.\n");  
        exit(2);  
    }  
    printf("\ntype:0x%x\n", finfo.type ); // FrameBuffer 类型,如0为象素  
    printf("visual:%d\n", finfo.visual ); // 视觉类型：如真彩2，伪彩3   
    printf("line_length:%d\n", finfo.line_length ); // 每行长度  
    printf("\nsmem_start:0x%lx,smem_len:%u\n", finfo.smem_start, finfo.smem_len ); // 映象RAM的参数  
    printf("mmio_start:0x%lx ,mmio_len:%u\n", finfo.mmio_start, finfo.mmio_len );  
      
    // Get variable screen information  
    if (ioctl(fbfd, FBIOGET_VSCREENINFO, &vinfo)) { // 获取设备可变信息  
        printf("Error reading variable information.\n");  
        exit(3);  
    }  
    printf("%dx%d, %dbpp,xres_virtual=%d,yres_virtual=%dvinfo.xoffset=%d,vinfo.yoffset=%d\n", 
    vinfo.xres, vinfo.yres, vinfo.bits_per_pixel,
    vinfo.xres_virtual,vinfo.yres_virtual,vinfo.xoffset,vinfo.yoffset);  

	screensize = finfo.line_length * vinfo.yres_virtual;
    // Map the device to memory 通过mmap系统调用将framebuffer内存映射到用户空间,并返回映射后的起始地址  
    fbp = (char *)mmap(0, screensize, PROT_READ | PROT_WRITE, MAP_SHARED,fbfd, 0);  
    if ((int)fbp == -1) {  
        printf("Error: failed to map framebuffer device to memory.\n");  
        exit(4);  
    }  
    printf("The framebuffer device was mapped to memory successfully.\n");  

    /*****************exampel********************/		
 	unsigned char *pTemp = (unsigned char *)fbp;
	int i, j;
	//起始坐标(x,y),终点坐标(right,bottom)
	x = 0;
	y = 0;
	int right = 480;//vinfo.xres;
	int bottom = 1708;//vinfo.yres;
	//计算偏移量.例如在（x，y）位置写入颜色 pixel值.
	//offset = (x + y * screen_width) * 4; // (4个字节) 
	for(i=y; i< bottom; i++)
	{
		for(j=x; j<right; j++)
		{
			unsigned short data = yellow_face_data[(((i-y)  % 128) * 128) + ((j-x) %128)];
			// *((uint32_t *)(fbp + offset)) = pixel;
			pTemp[i*finfo.line_length + (j*4) + 2] = (unsigned char)((data & 0xF800) >> 11 << 3);
			pTemp[i*finfo.line_length + (j*4) + 1] = (unsigned char)((data & 0x7E0) >> 5 << 2);
			pTemp[i*finfo.line_length + (j*4) + 0] = (unsigned char)((data & 0x1F) << 3);
		}
	} 
    /*****************FBIOPAN_DISPLAY********************/	
    //note：vinfo.xoffset =0 vinfo.yoffset =0 否则FBIOPAN_DISPLAY不成功
	if (ioctl(fbfd, FBIOPAN_DISPLAY, &vinfo)) {    
        printf("Error FBIOPAN_DISPLAY information.\n");  
        exit(5);  
    }  
	sleep(10);
	//finfo.smem_len == screensize == finfo.line_length * vinfo.yres_virtual 
	munmap(fbp,finfo.smem_len);
    close(fbfd);  
    return 0;  
}
```

##### 1.2、编译测试
编译会生成FrameBufferTest，然后进行测试。

>注意事项：
1、adb shell stop 杀掉surfaceflinger 之后在测试；
2、设置背光 echo 255 > /sys/class/leds/lcd-backlight/brightness

> 1、连接adb 
> 2、adb push FrameBufferTest system/bin 
> 3、进入adb shell
> 4、stop
> 5、echo 255 > /sys/class/leds/lcd-backlight/brightness
> 6、system/bin/FrameBufferTest

##### 1.3、显示效果
![Alt text | center](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/display.system/Android.PG1.FrameBufferTest.jpg)

**FrameBuffer 使用**
![Alt text | center](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/display.system/Android.PG1.FBTest.png)

#### 1.4、FrameBuffer 代码分析
FrameBuffer通常作为LCD控制器或者其他显示设备的驱动，FrameBuffer驱动是一个字符设备，设备节点是/dev/fbX（Android 设备为/dev/graphics/fb0），主设备号为29，次设备号递增，用户可以将Framebuffer看成是显示内存的一个映像，将其映射到进程地址空间之后，就可以直接进行读写操作，而写操作可以立即反应在屏幕上。这种操作是抽象的，统一的。用户不必关心物理显存的位置、换页机制等等具体细节。这些都是由Framebuffer设备驱动来完成的。Framebuffer设备为上层应用程序提供系统调用，也为下一层的特定硬件驱动提供接口；那些底层硬件驱动需要用到这儿的接口来向系统内核注册它们自己。

##### 1.4.1、 Framebuffer数据结构

##### 1.4.1.1、 fb_info
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
##### 1.4.1.2、fb_var_screeninfo

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/display.system/Android.PG1.fb_var_screeninfo.jpg)

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
##### 1.4.1.3、fb_fix_screeninfo
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


#### (二)、OpenGLES test App（Android OpenGLES Native层）

#### （2）、使用OpenGL ES绘制图像
##### 2.1、源代码
**OpenGLESTexturesRGB.cpp**

``` cpp
#include <stdlib.h>
#include <stdio.h>

#include <EGL/egl.h>
#include <GLES/gl.h>
#include <GLES/glext.h>

#include <WindowSurface.h>
#include <EGLUtils.h>

using namespace android;

int main(int /*argc*/, char** /*argv*/)
{
    EGLint configAttribs[] = {
         EGL_DEPTH_SIZE, 0,
         EGL_NONE
     };
     
     EGLint majorVersion;
     EGLint minorVersion;
     EGLContext context;
     EGLConfig config;
     EGLSurface surface;
     EGLint w, h;
     EGLDisplay dpy;

     WindowSurface windowSurface;
     EGLNativeWindowType window = windowSurface.getSurface();
     
     dpy = eglGetDisplay(EGL_DEFAULT_DISPLAY);
     eglInitialize(dpy, &majorVersion, &minorVersion);
          
     status_t err = EGLUtils::selectConfigForNativeWindow(
             dpy, configAttribs, window, &config);
     if (err) {
         fprintf(stderr, "couldn't find an EGLConfig matching the screen format\n");
         return 0;
     }

     surface = eglCreateWindowSurface(dpy, config, window, NULL);
     context = eglCreateContext(dpy, config, NULL, NULL);
     eglMakeCurrent(dpy, surface, surface, context);   
     eglQuerySurface(dpy, surface, EGL_WIDTH, &w);
     eglQuerySurface(dpy, surface, EGL_HEIGHT, &h);
     //GLint dim = w<h ? w : h;


     GLint crop[4] = { 0, 4, 4, -4 };
     glBindTexture(GL_TEXTURE_2D, 0);
     glTexParameteriv(GL_TEXTURE_2D, GL_TEXTURE_CROP_RECT_OES, crop);
     glTexParameterx(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
     glTexParameterx(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
     glTexParameterx(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
     glTexParameterx(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
     glTexEnvx(GL_TEXTURE_ENV, GL_TEXTURE_ENV_MODE, GL_REPLACE);
     glEnable(GL_TEXTURE_2D);
     glColor4f(1,1,1,1);

     // packing is always 4
     uint16_t t16R[]  = { 
             0xF000, 0xF000, 0xF000, 0xF000, 
             0xF000, 0xF000, 0xF000, 0xF000, 
             0xF000, 0xF000, 0xF000, 0xF000,  
             0xF000, 0xF000, 0xF000, 0xF000,  };

     uint16_t t16G[]  = { 
             0x0F00, 0x0F00, 0x0F00, 0x0F00, 
             0x0F00, 0x0F00, 0x0F00, 0x0F00, 
             0x0F00, 0x0F00, 0x0F00, 0x0F00,  
             0x0F00, 0x0F00, 0x0F00, 0x0F00,  };

     uint16_t t16B[]  = { 
             0x00F0, 0x00F0, 0x00F0, 0x00F0, 
             0x00F0, 0x00F0, 0x00F0, 0x00F0,
             0x00F0, 0x00F0, 0x00F0, 0x00F0, 
             0x00F0, 0x00F0, 0x00F0, 0x00F0,  };

     for(int i = 0; i < 10000; i++){
     glClear(GL_COLOR_BUFFER_BIT);
	 glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, 4, 4, 0, GL_RGBA, GL_UNSIGNED_SHORT_4_4_4_4, t16R);
     glDrawTexiOES(0, 0, 0, w/3, h);

	 glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, 4, 4, 0, GL_RGBA, GL_UNSIGNED_SHORT_4_4_4_4, t16G);
     glDrawTexiOES(w/3, 0, 0, w/3, h);

     glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, 4, 4, 0, GL_RGBA, GL_UNSIGNED_SHORT_4_4_4_4, t16B);
     glDrawTexiOES(w*2/3, 0, 0, w/3, h);

     eglSwapBuffers(dpy, surface);
     }
     //sleep(2); // so you have a chance to admire it
     return 0;
}
```

##### 2.2、编译测试
编译会生成OpenGLESTexturesRGB，然后进行测试。

> 1、连接adb 
> 2、adb push OpenGLESTexturesRGB system/bin 
> 3、进入adb shell
> 4、system/bin/OpenGLESTexturesRGB

##### 2.3、显示效果
![Alt text | center](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/display.system/Android.PG1.OpenGLESTexturesRGB.png)

##### 2.4、OpenGLESTexturesRGB代码分析

> 使用OpenGLES的绘图的一般步骤：

>1、获取 EGL Display 对象：eglGetDisplay()
>2、初始化与 EGLDisplay 之间的连接：eglInitialize()
>3、获取 EGLConfig 对象：eglChooseConfig()
>4、创建 EGLContext 实例：eglCreateContext()
>5、创建 EGLSurface 实例：eglCreateWindowSurface()
>6、连接 EGLContext 和 EGLSurface：eglMakeCurrent()
>7、使用 OpenGL ES API 绘制图形：Draw()
>8、切换 front buffer 和 back buffer 送显：eglSwapBuffer()
>9、断开并释放与 EGLSurface 关联的 EGLContext 对象：eglRelease()
>10、删除 EGLSurface 对象
>11、删除 EGLContext 对象
>12、终止与 EGLDisplay 之间的连接
![Alt text | center](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/display.system/Android.PG1.OpenGLESUse.png)

#### (三)、SurfaceFlinger test App（Android SurfaceFlinger Native层）
#### （3）、SurfaceFlinger合成图像
##### 3.1、源代码
**Transaction_test_red.cpp**
``` cpp

#include <algorithm>
#include <functional>
#include <limits>
#include <ostream>

#include <gtest/gtest.h>

#include <android/native_window.h>

#include <gui/ISurfaceComposer.h>
#include <gui/LayerState.h>

#include <gui/Surface.h>
#include <gui/SurfaceComposerClient.h>
#include <private/gui/ComposerService.h>

#include <ui/DisplayInfo.h>
#include <ui/Rect.h>
#include <utils/String8.h>

#include <math.h>
#include <math/vec3.h>

namespace android {

namespace {

struct Color {
    uint8_t r;
    uint8_t g;
    uint8_t b;
    uint8_t a;

    static const Color RED;
//    static const Color GREEN;
//    static const Color BLUE;
//    static const Color WHITE;
//    static const Color BLACK;
//    static const Color TRANSPARENT;
};

const Color Color::RED{255, 0, 0, 255};
//const Color Color::GREEN{0, 255, 0, 255};
//const Color Color::BLUE{0, 0, 255, 255};
//const Color Color::WHITE{255, 255, 255, 255};
//const Color Color::BLACK{0, 0, 0, 255};
//const Color Color::TRANSPARENT{0, 0, 0, 0};

// Fill a region with the specified color.
void fillBufferColor(const ANativeWindow_Buffer& buffer, const Rect& rect, const Color& color) {
    int32_t x = rect.left;
    int32_t y = rect.top;
    int32_t width = rect.right - rect.left;
    int32_t height = rect.bottom - rect.top;

    if (x < 0) {
        width += x;
        x = 0;
    }
    if (y < 0) {
        height += y;
        y = 0;
    }
    if (x + width > buffer.width) {
        x = std::min(x, buffer.width);
        width = buffer.width - x;
    }
    if (y + height > buffer.height) {
        y = std::min(y, buffer.height);
        height = buffer.height - y;
    }

    for (int32_t j = 0; j < height; j++) {
        uint8_t* dst = static_cast<uint8_t*>(buffer.bits) + (buffer.stride * (y + j) + x) * 4;
        for (int32_t i = 0; i < width; i++) {
            dst[0] = color.r;
            dst[1] = color.g;
            dst[2] = color.b;
            dst[3] = color.a;
            dst += 4;
        }
    }
}


} // anonymous namespace

using Transaction = SurfaceComposerClient::Transaction;


class LayerTransactionTest : public ::testing::Test {
protected:
    void SetUp() override {
        mClient = new SurfaceComposerClient;
        ASSERT_EQ(NO_ERROR, mClient->initCheck()) << "failed to create SurfaceComposerClient";

        ASSERT_NO_FATAL_FAILURE(SetUpDisplay());
    }

    sp<SurfaceControl> createLayer(const char* name, uint32_t width, uint32_t height,
                                   uint32_t flags = 0) {
        auto layer =
                mClient->createSurface(String8(name), width, height, PIXEL_FORMAT_RGBA_8888, flags);
        EXPECT_NE(nullptr, layer.get()) << "failed to create SurfaceControl";

        status_t error = Transaction()
                                 .setLayerStack(layer, mDisplayLayerStack)
                                 .setLayer(layer, mLayerZBase)
                                 .apply();
        if (error != NO_ERROR) {
            ADD_FAILURE() << "failed to initialize SurfaceControl";
            layer.clear();
        }

        return layer;
    }

    ANativeWindow_Buffer getLayerBuffer(const sp<SurfaceControl>& layer) {
        // wait for previous transactions (such as setSize) to complete
        Transaction().apply(true);

        ANativeWindow_Buffer buffer = {};
        EXPECT_EQ(NO_ERROR, layer->getSurface()->lock(&buffer, nullptr));

        return buffer;
    }

    void postLayerBuffer(const sp<SurfaceControl>& layer) {
        ASSERT_EQ(NO_ERROR, layer->getSurface()->unlockAndPost());

        // wait for the newly posted buffer to be latched
        waitForLayerBuffers();
    }

    void fillLayerColor(const sp<SurfaceControl>& layer, const Color& color) {
        ANativeWindow_Buffer buffer;
        ASSERT_NO_FATAL_FAILURE(buffer = getLayerBuffer(layer));
        fillBufferColor(buffer, Rect(0, 0, buffer.width, buffer.height), color);
        postLayerBuffer(layer);
    }

    void fillLayerQuadrant(const sp<SurfaceControl>& layer, const Color& topLeft,
                           const Color& topRight, const Color& bottomLeft,
                           const Color& bottomRight) {
        ANativeWindow_Buffer buffer;
        ASSERT_NO_FATAL_FAILURE(buffer = getLayerBuffer(layer));
        ASSERT_TRUE(buffer.width % 2 == 0 && buffer.height % 2 == 0);

        const int32_t halfW = buffer.width / 2;
        const int32_t halfH = buffer.height / 2;
        fillBufferColor(buffer, Rect(0, 0, halfW, halfH), topLeft);
        fillBufferColor(buffer, Rect(halfW, 0, buffer.width, halfH), topRight);
        fillBufferColor(buffer, Rect(0, halfH, halfW, buffer.height), bottomLeft);
        fillBufferColor(buffer, Rect(halfW, halfH, buffer.width, buffer.height), bottomRight);

        postLayerBuffer(layer);
    }

    sp<SurfaceComposerClient> mClient;

    sp<IBinder> mDisplay;
    uint32_t mDisplayWidth;
    uint32_t mDisplayHeight;
    uint32_t mDisplayLayerStack;

    // leave room for ~256 layers
    const int32_t mLayerZBase = std::numeric_limits<int32_t>::max() - 256;

private:
    void SetUpDisplay() {
        mDisplay = mClient->getBuiltInDisplay(ISurfaceComposer::eDisplayIdMain);
        ASSERT_NE(nullptr, mDisplay.get()) << "failed to get built-in display";

        // get display width/height
        DisplayInfo info;
        SurfaceComposerClient::getDisplayInfo(mDisplay, &info);
        mDisplayWidth = info.w;
        mDisplayHeight = info.h;

        // After a new buffer is queued, SurfaceFlinger is notified and will
        // latch the new buffer on next vsync.  Let's heuristically wait for 3
        // vsyncs.
        mBufferPostDelay = int32_t(1e6 / info.fps) * 3;

        mDisplayLayerStack = 0;
        // set layer stack (b/68888219)
        Transaction t;
        t.setDisplayLayerStack(mDisplay, mDisplayLayerStack);
        t.apply();
    }

    void waitForLayerBuffers() { usleep(mBufferPostDelay); }

    int32_t mBufferPostDelay;
};


TEST_F(LayerTransactionTest, SetPositionBasic) {
    sp<SurfaceControl> layer;
    ASSERT_NO_FATAL_FAILURE(layer = createLayer("test", mDisplayWidth / 3, mDisplayHeight));
    ASSERT_NO_FATAL_FAILURE(fillLayerColor(layer, Color::RED));
    for(int i = 0; i< 10000; i++){
        Transaction().setPosition(layer, 0, 0).apply();
    }
}

//end
}
```
绿色和蓝色的源码类似就不贴出啦，请直接看源码。
##### 3.2、编译测试
编译会SurfaceFlingerTestsRed/Green/Blue生成
SurfaceFlingerTestsRed
SurfaceFlingerTestsGreen
SurfaceFlingerTestsBlue，然后分别进行测试。

> 1、连接adb 
> 2、adb push SurfaceFlingerTestsRed system/bin 
> 3、进入adb shell
> 4、system/bin/SurfaceFlingerTestsRed

##### 3.3、显示效果
**SurfaceFlingerTestsRed**
![Alt text | center](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/display.system/Android.PG1.SurfaceFlingerTestRed.png)

**SurfaceFlingerTestsGreen**
![Alt text | center](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/display.system/Android.PG1.SurfaceFlingerTestGreen.png)

**SurfaceFlingerTestsBlue**
![Alt text | center](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/display.system/Android.PG1.SurfaceFlingerTestBlue.png)

同时运行三个bin文件
![Alt text | center](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/display.system/Android.PG1.SurfaceFlingerTestRedGreenBlue.png)

##### 3.4、SurfaceFlingerTest代码分析
SurfaceFlinger涉及流程复杂，稍后另外博客再继续分析！

#### (四)、AMS、WMS test App（Android Apk Java层）
#### （4）、Apk测试AMS,WMS,SurfaceFlinger
##### 4.1、源代码
**TestActivity.java &&TestView.java **
``` java
public class TestActivity extends Activity {
    private final static String TAG = "TestActivity";
    TestView mView;

    @Override
    protected void onCreate(Bundle icicle) {
        super.onCreate(icicle);
        mView = new TestView(getApplication());
	    mView.setFocusableInTouchMode(true);
	    setContentView(mView);
    }

    @Override
    protected void onPause() {
        super.onPause();
        mView.onPause();
    }

    @Override
    protected void onResume() {
        super.onResume();
        mView.onResume();
    }
}

class TestView extends GLSurfaceView {
    TestView(Context context) {
        super(context);
        init();
    }

    public TestView(Context context, AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    private void init() {
        setRenderer(new Renderer());
        setRenderMode(RENDERMODE_WHEN_DIRTY);
    }

    private class Renderer implements GLSurfaceView.Renderer {
        private static final String TAG = "Renderer";
			@Override
			public void onSurfaceCreated(GL10 gl, EGLConfig config) {
				gl.glClearColor(0f, 1f, 0f, 0f);
			}
		
			@Override
			public void onSurfaceChanged(GL10 gl, int width, int height) {
				gl.glViewport(0, 0, width, height);
			}
		
			@Override
			public void onDrawFrame(GL10 gl) {
				gl.glClear(GL10.GL_COLOR_BUFFER_BIT);
			}
    }
}
```
绿色和蓝色的源码类似就不贴出啦，请直接看源码。

##### 4.2、编译测试
编译会TestViewportRed/Green/Blue apk生成
TestViewportRed.apk
TestViewportGreen.apk
TestViewportBlue.apk，然后分别进行测试。

> 1、连接adb 
> 2、adb shell am start -n com.android.testred/.TestActivity
> 3、adb shell am start -n com.android.testgreen/.TestActivity
> 4、adb shell am start -n com.android.testblue/.TestActivity

##### 4.3、显示效果
**TestViewportRed.apk**

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/display.system/Android.PG1.TestViewportRed.png)

**TestViewportGreen.apk**
![Alt text | center](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/display.system/Android.PG1.TestViewportGreen.png)


**TestViewportBlue.apk**
![Alt text | center](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/display.system/Android.PG1.TestViewportBlue.png)


同时运行三个apk
![Alt text | center](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/display.system/Android.PG1.TestViewportRGB.png)

Android 默认运行Activity是全屏
要实现上述效果，需要修改源码

``` java
diff --git a/frameworks/base/core/java/android/view/ViewRootImpl.java b/frameworks/base/core/java/android/view/ViewRootImpl.java
index ab63be2..e7bda30 100644
--- a/frameworks/base/core/java/android/view/ViewRootImpl.java
+++ b/frameworks/base/core/java/android/view/ViewRootImpl.java
 -6568,14 +6568,36 @@ public final class ViewRootImpl implements ViewParent,
             frameNumber = mSurface.getNextFrameNumber();
         }
 
-        int relayoutResult = mWindowSession.relayout(mWindow, mSeq, params,
+        //charlesvincent start
+        int relayoutResult = 0;
+        if(params != null ){
+               if("com.android.testred".equals(params.packageName) 
+				|| "com.android.testgreen".equals(params.packageName)
+				|| "com.android.testblue".equals(params.packageName)){
+				Point size = new Point();
+				mDisplay.getRealSize(size);
+				params.width = (int) size.x / 3;
+				params.height = (int) size.y;
+
+                relayoutResult = mWindowSession.relayout(mWindow, mSeq, params,
+                (int) size.x / 3,
+                (int) size.y, viewVisibility,
+                insetsPending ? WindowManagerGlobal.RELAYOUT_INSETS_PENDING : 0, frameNumber,
+                mWinFrame, mPendingOverscanInsets, mPendingContentInsets, mPendingVisibleInsets,
+                mPendingStableInsets, mPendingOutsets, mPendingBackDropFrame, mPendingDisplayCutout,
+                mPendingMergedConfiguration, mSurface);
+
+			}else {
+                relayoutResult = mWindowSession.relayout(mWindow, mSeq, params,
                 (int) (mView.getMeasuredWidth() * appScale + 0.5f),
                 (int) (mView.getMeasuredHeight() * appScale + 0.5f), viewVisibility,
                 insetsPending ? WindowManagerGlobal.RELAYOUT_INSETS_PENDING : 0, frameNumber,
                 mWinFrame, mPendingOverscanInsets, mPendingContentInsets, mPendingVisibleInsets,
                 mPendingStableInsets, mPendingOutsets, mPendingBackDropFrame, mPendingDisplayCutout,
                 mPendingMergedConfiguration, mSurface);
-
+			}
+        }
+        //charlesvincent end
         mPendingAlwaysConsumeNavBar =
                 (relayoutResult & WindowManagerGlobal.RELAYOUT_RES_CONSUME_ALWAYS_NAV_BAR) != 0;
 
diff --git a/frameworks/base/services/core/java/com/android/server/am/ActivityStack.java b/frameworks/base/services/core/java/com/android/server/am/ActivityStack.java
index 0a3ace7..923fea4 100644
--- a/frameworks/base/services/core/java/com/android/server/am/ActivityStack.java
+++ b/frameworks/base/services/core/java/com/android/server/am/ActivityStack.java
 -1957,7 +1957,16 @@ class ActivityStack<T extends StackWindowController> extends ConfigurationContai
                                 + " stackShouldBeVisible=" + stackShouldBeVisible
                                 + " behindFullscreenActivity=" + behindFullscreenActivity
                                 + " mLaunchTaskBehind=" + r.mLaunchTaskBehind);
-                        makeInvisible(r);
+
+			//charlesvincent start
+			if("com.android.testred".equals(r.packageName) 
+			   || "com.android.testgreen".equals(r.packageName)
+			   || "com.android.testblue".equals(r.packageName)){
+				//do nothing
+			}else{
+               makeInvisible(r);
+			}
+			//charlesvincent end
                     }
                 }
                 final int windowingMode = getWindowingMode();
diff --git a/frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java b/frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java
index e56e5d0..7f6d6c9 100644
--- a/frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java
+++ b/frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java
 -5629,6 +5629,37 @@ public class PhoneWindowManager implements WindowManagerPolicy {
             }
         }
 
+
+        //charlesvincent
+        if ("com.android.testred".equals(win.getAttrs().packageName)) {
+		   pf.left = df.left = of.left = cf.left = displayFrames.mOverscan.left;
+		   pf.top = df.top = of.top = cf.top = displayFrames.mOverscan.top;
+		   pf.right = df.right = of.right = cf.right
+				   = displayFrames.mOverscan.left + displayFrames.mOverscan.right / 3;
+		   pf.bottom = df.bottom = of.bottom = cf.bottom
+				   = displayFrames.mOverscan.top + displayFrames.mOverscan.bottom;
+        }
+
+        //charlesvincent
+        if ("com.android.testgreen".equals(win.getAttrs().packageName)) {
+		   pf.left = df.left = of.left = cf.left = displayFrames.mOverscan.left  + displayFrames.mOverscan.right / 3;
+		   pf.top = df.top = of.top = cf.top = displayFrames.mOverscan.top;
+		   pf.right = df.right = of.right = cf.right
+				   = displayFrames.mOverscan.left  + displayFrames.mOverscan.right * 2 / 3;
+		   pf.bottom = df.bottom = of.bottom = cf.bottom
+				   = displayFrames.mOverscan.top + displayFrames.mOverscan.bottom;
+        }
+
+        //charlesvincent
+        if ("com.android.testblue".equals(win.getAttrs().packageName)) {
+		   pf.left = df.left = of.left = cf.left = displayFrames.mOverscan.left  + displayFrames.mOverscan.right * 2 / 3;
+		   pf.top = df.top = of.top = cf.top = displayFrames.mOverscan.top;
+		   pf.right = df.right = of.right = cf.right
+				   = displayFrames.mOverscan.left  + displayFrames.mOverscan.right * 3 / 3;
+		   pf.bottom = df.bottom = of.bottom = cf.bottom
+				   = displayFrames.mOverscan.top + displayFrames.mOverscan.bottom;
+        }
+
         if (DEBUG_LAYOUT) Slog.v(TAG, "Compute frame " + attrs.getTitle()
                 + ": sim=#" + Integer.toHexString(sim)
                 + " attach=" + attached + " type=" + type
diff --git a/frameworks/base/services/core/java/com/android/server/wm/WindowState.java b/frameworks/base/services/core/java/com/android/server/wm/WindowState.java
index 118b01c..7559383 100644
--- a/frameworks/base/services/core/java/com/android/server/wm/WindowState.java
+++ b/frameworks/base/services/core/java/com/android/server/wm/WindowState.java
@@ -734,34 +734,51 @@ class WindowState extends WindowContainer<WindowState> implements WindowManagerP
         }
         mDeathRecipient = deathRecipient;
 
-        if (mAttrs.type >= FIRST_SUB_WINDOW && mAttrs.type <= LAST_SUB_WINDOW) {
-            // The multiplier here is to reserve space for multiple
-            // windows in the same type layer.
-            mBaseLayer = mPolicy.getWindowLayerLw(parentWindow)
-                    * TYPE_LAYER_MULTIPLIER + TYPE_LAYER_OFFSET;
-            mSubLayer = mPolicy.getSubWindowLayerFromTypeLw(a.type);
-            mIsChildWindow = true;
-
-            if (DEBUG_ADD_REMOVE) Slog.v(TAG, "Adding " + this + " to " + parentWindow);
-            parentWindow.addChild(this, sWindowSubLayerComparator);
-
-            mLayoutAttached = mAttrs.type !=
-                    WindowManager.LayoutParams.TYPE_APPLICATION_ATTACHED_DIALOG;
-            mIsImWindow = parentWindow.mAttrs.type == TYPE_INPUT_METHOD
-                    || parentWindow.mAttrs.type == TYPE_INPUT_METHOD_DIALOG;
-            mIsWallpaper = parentWindow.mAttrs.type == TYPE_WALLPAPER;
-        } else {
-            // The multiplier here is to reserve space for multiple
-            // windows in the same type layer.
-            mBaseLayer = mPolicy.getWindowLayerLw(this)
-                    * TYPE_LAYER_MULTIPLIER + TYPE_LAYER_OFFSET;
-            mSubLayer = 0;
-            mIsChildWindow = false;
-            mLayoutAttached = false;
-            mIsImWindow = mAttrs.type == TYPE_INPUT_METHOD
-                    || mAttrs.type == TYPE_INPUT_METHOD_DIALOG;
-            mIsWallpaper = mAttrs.type == TYPE_WALLPAPER;
-        }
+	//charlesvincent start
+	if("com.android.testred".equals(mAttrs.packageName) 
+			|| "com.android.testgreen".equals(mAttrs.packageName)
+			|| "com.android.testblue".equals(mAttrs.packageName)){
+			mBaseLayer = 21010;
+			mSubLayer = 0;
+			mIsChildWindow = false;
+			mLayoutAttached = false;
+			mIsImWindow = mAttrs.type == TYPE_INPUT_METHOD
+					|| mAttrs.type == TYPE_INPUT_METHOD_DIALOG;
+			mIsWallpaper = mAttrs.type == TYPE_WALLPAPER;
+	} else{
+
+	        if (mAttrs.type >= FIRST_SUB_WINDOW && mAttrs.type <= LAST_SUB_WINDOW) {
+	            // The multiplier here is to reserve space for multiple
+	            // windows in the same type layer.
+	            mBaseLayer = mPolicy.getWindowLayerLw(parentWindow)
+	                    * TYPE_LAYER_MULTIPLIER + TYPE_LAYER_OFFSET;
+	            mSubLayer = mPolicy.getSubWindowLayerFromTypeLw(a.type);
+	            mIsChildWindow = true;
+
+	            if (DEBUG_ADD_REMOVE) Slog.v(TAG, "Adding " + this + " to " + parentWindow);
+	            parentWindow.addChild(this, sWindowSubLayerComparator);
+
+	            mLayoutAttached = mAttrs.type !=
+	                    WindowManager.LayoutParams.TYPE_APPLICATION_ATTACHED_DIALOG;
+	            mIsImWindow = parentWindow.mAttrs.type == TYPE_INPUT_METHOD
+	                    || parentWindow.mAttrs.type == TYPE_INPUT_METHOD_DIALOG;
+	            mIsWallpaper = parentWindow.mAttrs.type == TYPE_WALLPAPER;
+	        } else {
+	            // The multiplier here is to reserve space for multiple
+	            // windows in the same type layer.
+	            mBaseLayer = mPolicy.getWindowLayerLw(this)
+	                    * TYPE_LAYER_MULTIPLIER + TYPE_LAYER_OFFSET;
+	            mSubLayer = 0;
+	            mIsChildWindow = false;
+	            mLayoutAttached = false;
+	            mIsImWindow = mAttrs.type == TYPE_INPUT_METHOD
+	                    || mAttrs.type == TYPE_INPUT_METHOD_DIALOG;
+	            mIsWallpaper = mAttrs.type == TYPE_WALLPAPER;
+	        }
+
+	}
+ 	//charlesvincent end
+
         mIsFloatingLayer = mIsImWindow || mIsWallpaper;
 
         if (mAppToken != null && mAppToken.mShowForAllUsers) {
```

说明：假如没有状态栏StatusBar和导航栏NavigationBar，测试结果就和之前完全一样啦，我们通过adb shell dumpsys SurfaceFlinger来查看大小可以验证。

``` java
+ BufferLayer (com.android.testred/com.android.testred.TestActivity#0)
  pos=(0,0), size=( 160, 854), 

+ BufferLayer (com.android.testgreen/com.android.testgreen.TestActivity#0)
  pos=(160,0), size=( 160, 854),
  
+ BufferLayer (com.android.testblue/com.android.testblue.TestActivity#0)
  pos=(320,0), size=( 160, 854),


Display 0 HWC layers:
-------------------------------------------------------------------------------
 Layer name
           Z |  Comp Type |   Disp Frame (LTRB) |          Source Crop (LTRB)
-------------------------------------------------------------------------------
 SurfaceView - com.android.testred/com.android.testred.TestActivity#0
  rel     -2 |     Client |    0    0  160  782 |    0.0    0.0  160.0  782.0
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
 SurfaceView - com.android.testgreen/com.android.testgreen.TestActivity#0
  rel     -2 |     Client |  160    0  320  782 |    0.0    0.0  160.0  782.0
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
 SurfaceView - com.android.testblue/com.android.testblue.TestActivity#0
  rel     -2 |     Client |  320    0  480  782 |    0.0    0.0  160.0  782.0
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

##### 2.4、TestViewportX代码分析
APK涉及AMS,WMS,SurfaceFlinger流程复杂，稍后另外博客再继续分析！

#### (五)、参考资料(特别感谢)：

[（1）【高通Android平台-应用空间操作framebuffer dump LCD总结】](https://blog.csdn.net/eliot_shao/article/details/74926010) 
[（2）【Linux 下framebuffer 帧缓冲的使用】](https://blog.csdn.net/eliot_shao/article/details/74926010) 
[（3）【Android Framebuffer介绍及使用】](https://www.jianshu.com/p/df1213e5a0ed) 

