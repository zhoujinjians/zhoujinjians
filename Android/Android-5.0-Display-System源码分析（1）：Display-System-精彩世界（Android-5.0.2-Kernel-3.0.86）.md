---
title: Android L Display System源码分析（1）：Display System 精彩世界（Android 5.0.2 && Kernel 3.0.86）
cover: https://raw.githubusercontent.com/zhoujinjiankim/PicGo/master/personal.website/post.cover.pictures.00001.jpg
categories: 
  - Display
tags:
  - Android
  - Linux
  - Graphics
toc: true
abbrlink: 20190801
date: 2019-08-01 09:25:00
---



注：文章都是通过阅读 Android  && Linux 平台源码、各位前辈总结的资料、加上自己的思考分析总结出来的，其中难免有理不对的地方，欢迎大家批评指正。**文章为个人学习、研究、欣赏之用。图文内容、源码整理自互联网，如有侵权，请联系删除（◕‿◕）**，转载请注明出处（ ©Android @Linux 版权所有），谢谢(๑乛◡乛๑) 、（ ͡° ͜ʖ ͡°）、（ಡωಡ）！！！。
（==**文章基于 Kernel-3.0.86**==）&&（==**文章基于 Android 5.0.2**==）
[【开发板 - 友善之臂 FriendlyARM Cortex-A9 Tiny4412 ADK Exynos4412 （ Android 5.0.2）HD702高清电容屏 扩展套餐】](https://item.taobao.com/item.htm?spm=0.0.0.0.3GsGdQ&id=20131438062)
[【开发板 Android 5.0.2 && Kernel 3.0.86 源码链接： https://pan.baidu.com/s/1jJHm74q 密码：yfih】](https://pan.baidu.com/s/1jJHm74q)

正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢！！！

[（1）【高通Android平台-应用空间操作framebuffer dump LCD总结】](https://blog.csdn.net/eliot_shao/article/details/74926010) 


--------------------------------------------------------------------------------
==源码（部分）==：

>AMS、WMS、SurfaceFlinger test App（Android Apk Java层）

- \android-5.0.2\vendor\friendly-arm\tiny4412\TestViewportRed
- \android-5.0.2\vendor\friendly-arm\tiny4412\TestViewportGreen
- \android-5.0.2\vendor\friendly-arm\tiny4412\TestViewportBlue

> SurfaceFlinger test App（Android Native层）

- \android-5.0.2\vendor\friendly-arm\tiny4412\SurfaceflingerTestsRed
- \android-5.0.2\vendor\friendly-arm\tiny4412\SurfaceflingerTestsGreen
- \android-5.0.2\vendor\friendly-arm\tiny4412\SurfaceflingerTestsBlue

>OpenGLES test App（Android OpenGLES Native层）

- \android-5.0.2\vendor\friendly-arm\tiny4412\OpenGLESTexturesRGB


>FrameBuffer test App （Kernel 用户层）

- \android-5.0.2\vendor\friendly-arm\tiny4412\FrameBufferTest

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
    printf("%dx%d, %dbpp,xres_virtual=%d,yres_virtual=%d,vinfo.xoffset=%d,vinfo.yoffset=%d\n", vinfo.xres, vinfo.yres, vinfo.bits_per_pixel,vinfo.xres_virtual,vinfo.yres_virtual,vinfo.xoffset,vinfo.yoffset);  
 
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



> LOCAL_PATH:= $(call my-dir)
> include $(CLEAR_VARS)
> 
> LOCAL_SRC_FILES:= FrameBufferTest.cpp
> 
> LOCAL_SHARED_LIBRARIES := liblog libcutils
> 
> LOCAL_MODULE := FrameBufferTest
> 
> LOCAL_MODULE_TAGS := tests
> 
> include $(BUILD_EXECUTABLE)



##### 1.2、编译测试

编译会生成FrameBufferTest，然后进行测试。

> 1、进入adb shell
> 2、cd system/bin/
> 3、./FrameBufferTest

##### 1.3、显示效果
![enter image description here](https://raw.githubusercontent.com/zhoujinjiankim/PicGo/master/personal.website/zjj.display.sys.FBTest.png)
**FrameBuffer 使用**
![enter image description here](https://raw.githubusercontent.com/zhoujinjiankim/PicGo/master/personal.website/zjj.display.sys.FrameBuferrTest.JPG)

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

![enter image description here](https://raw.githubusercontent.com/zhoujinjiankim/PicGo/master/personal.website/zjj.display.sys.fb_var_screeninfo.jpg)

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
![enter image description here](https://raw.githubusercontent.com/zhoujinjiankim/PicGo/master/personal.website/zjj.display.sys.OpenGLESTexturesRGB.JPG)


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
![enter image description here](https://raw.githubusercontent.com/zhoujinjiankim/PicGo/master/personal.website/zjj.display.sys.OpenGLESUse.png)

#### (三)、SurfaceFlinger test App（Android SurfaceFlinger Native层）
#### （3）、SurfaceFlinger合成图像
##### 3.1、源代码
**SurfaceFlingerTestsRed**
```cpp


#include <cutils/memory.h>

#include <utils/Log.h>

#include <binder/IPCThreadState.h>
#include <binder/ProcessState.h>
#include <binder/IServiceManager.h>
#include <android/native_window.h>
#include <gui/Surface.h>
#include <gui/SurfaceComposerClient.h>

using namespace android;

//namespace android {

int main(int argc, char** argv)
{
    // set up the thread-pool
    sp<ProcessState> proc(ProcessState::self());
    ProcessState::self()->startThreadPool();

    // create a client to surfaceflinger
    sp<SurfaceComposerClient> client = new SurfaceComposerClient();
    
    sp<SurfaceControl> surfaceControl = client->createSurface(String8("resize"),
            800 / 3, 1280, PIXEL_FORMAT_RGB_565, 0);

    sp<Surface> surface = surfaceControl->getSurface();

    SurfaceComposerClient::openGlobalTransaction();
    surfaceControl->setLayer(100000);
    surfaceControl->setSize(800 / 3, 1280);
    surfaceControl->setPosition(0, 0);
    SurfaceComposerClient::closeGlobalTransaction();

    ANativeWindow_Buffer outBuffer;
    surface->lock(&outBuffer, NULL);
    ssize_t bpr = outBuffer.stride * bytesPerPixel(outBuffer.format);
    android_memset16((uint16_t*)outBuffer.bits, 0xF800, bpr*outBuffer.height);
    surface->unlockAndPost();

    IPCThreadState::self()->joinThreadPool();
    
    return 0;
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
![enter image description here](https://raw.githubusercontent.com/zhoujinjiankim/PicGo/master/personal.website/zjj.display.sys.SurfaceFlingerTestsRed.JPG)

**SurfaceFlingerTestsGreen**
![enter image description here](https://raw.githubusercontent.com/zhoujinjiankim/PicGo/master/personal.website/zjj.display.sys.SurfaceFlingerTestsGreen.JPG)

**SurfaceFlingerTestsBlue**
![enter image description here](https://raw.githubusercontent.com/zhoujinjiankim/PicGo/master/personal.website/zjj.display.sys.SurfaceFlingerTestsBlue.JPG)

同时运行三个bin文件
![enter image description here](https://raw.githubusercontent.com/zhoujinjiankim/PicGo/master/personal.website/zjj.display.sys.SurfaceFlingerTestsX.JPG)

##### 3.4、SurfaceFlingerTest代码分析
SurfaceFlinger涉及流程复杂，稍后另外博客再继续分析！

#### (四)、AMS、WMS test App（Android Apk Java层）
#### （4）、Apk测试AMS,WMS,SurfaceFlinger
##### 4.1、源代码
**TestActivity.java &&TestView.java **
```java
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
![enter image description here](https://raw.githubusercontent.com/zhoujinjiankim/PicGo/master/personal.website/zjj.display.sys.TestViewportRed.JPG)


**TestViewportGreen.apk**
![enter image description here](https://raw.githubusercontent.com/zhoujinjiankim/PicGo/master/personal.website/zjj.display.sys.TestViewportGreen.JPG)



**TestViewportBlue.apk**
![enter image description here](https://raw.githubusercontent.com/zhoujinjiankim/PicGo/master/personal.website/zjj.display.sys.TestViewportBlue.JPG)



Android 默认运行Activity是全屏
要实现上述效果，需要修改源码

```javascript
/frameworks/base/core/java/android/view/ViewRootImpl.java
        /*int relayoutResult = mWindowSession.relayout(
                mWindow, mSeq, params,
                (int) (mView.getMeasuredWidth() * appScale + 0.5f),
                (int) (mView.getMeasuredHeight() * appScale + 0.5f),
                viewVisibility, insetsPending ? WindowManagerGlobal.RELAYOUT_INSETS_PENDING : 0,
                mWinFrame, mPendingOverscanInsets, mPendingContentInsets, mPendingVisibleInsets,
                mPendingStableInsets, mPendingConfiguration, mSurface);
        */

		//charlesvincent start
		int relayoutResult = 0;
		if(params != null ){
		if("com.android.testred".equals(params.packageName)
		|| "com.android.testgreen".equals(params.packageName)
		|| "com.android.testblue".equals(params.packageName)){
		    Point size = new Point();
		    mDisplay.getRealSize(size);
		    params.width = (int) size.x / 3;
		    params.height = (int) size.y;
		
		    relayoutResult = mWindowSession.relayout(
                    mWindow, mSeq, params,
		    (int) size.x / 3,
		     (int) size.y, 
                    viewVisibility, insetsPending ? WindowManagerGlobal.RELAYOUT_INSETS_PENDING : 0,
                    mWinFrame, mPendingOverscanInsets, mPendingContentInsets, mPendingVisibleInsets,
                    mPendingStableInsets, mPendingConfiguration, mSurface);
		}else {
		    relayoutResult = mWindowSession.relayout(
                    mWindow, mSeq, params,
                    (int) (mView.getMeasuredWidth() * appScale + 0.5f),
                    (int) (mView.getMeasuredHeight() * appScale + 0.5f),
                    viewVisibility, insetsPending ? WindowManagerGlobal.RELAYOUT_INSETS_PENDING : 0,
                    mWinFrame, mPendingOverscanInsets, mPendingContentInsets, mPendingVisibleInsets,
                    mPendingStableInsets, mPendingConfiguration, mSurface);
		    }
		}
		//charlesvincent end
        //Log.d(TAG, "<<<<<< BACK FROM relayout");

/frameworks/base/services/core/java/com/android/server/am/ActivityStack.java

                    // Now for any activities that aren't visible to the user, make
                    // sure they no longer are keeping the screen frozen.
                    if (r.visible) {
                        if (DEBUG_VISBILITY) Slog.v(TAG, "Making invisible: " + r);
                        try {
			//charlesvincent start
			if("com.android.testred".equals(r.packageName) 
			   || "com.android.testgreen".equals(r.packageName)
			   || "com.android.testblue".equals(r.packageName)){
				//do nothing
			}else{
                            setVisibile(r, false);
			}
			//charlesvincent end

                            
                            switch (r.state) {


/frameworks/base/policy/src/com/android/internal/policy/impl/PhoneWindowManager.java 




        //charlesvincent
        if ("com.android.testred".equals(win.getAttrs().packageName)) {
		   pf.left = df.left = of.left = cf.left = mOverscanScreenLeft;
		   pf.top = df.top = of.top = cf.top = mOverscanScreenTop;
		   pf.right = df.right = of.right = cf.right
				   = mOverscanScreenLeft + mOverscanScreenWidth / 3;
		   pf.bottom = df.bottom = of.bottom = cf.bottom
				   = mOverscanScreenTop + mOverscanScreenHeight;
        }

        //charlesvincent
        if ("com.android.testgreen".equals(win.getAttrs().packageName)) {
		   pf.left = df.left = of.left = cf.left = mOverscanScreenLeft  + mOverscanScreenWidth / 3;
		   pf.top = df.top = of.top = cf.top = mOverscanScreenTop;
		   pf.right = df.right = of.right = cf.right
				   = mOverscanScreenLeft  + mOverscanScreenWidth * 2 / 3;
		   pf.bottom = df.bottom = of.bottom = cf.bottom
				   = mOverscanScreenTop + mOverscanScreenHeight;
        }

        //charlesvincent
        if ("com.android.testblue".equals(win.getAttrs().packageName)) {
		   pf.left = df.left = of.left = cf.left = mOverscanScreenLeft  + mOverscanScreenWidth * 2 / 3;
		   pf.top = df.top = of.top = cf.top = mOverscanScreenTop;
		   pf.right = df.right = of.right = cf.right
				   = mOverscanScreenLeft  + mOverscanScreenWidth * 3 / 3;
		   pf.bottom = df.bottom = of.bottom = cf.bottom
				   = mOverscanScreenTop + mOverscanScreenHeight;
        }


        if (DEBUG_LAYOUT) Slog.v(TAG, "Compute frame " + attrs.getTitle()
                + ": sim=#" + Integer.toHexString(sim)
                + " attach=" + attached + " type=" + attrs.type
                + String.format(" flags=0x%08x", fl)
                + " pf=" + pf.toShortString() + " df=" + df.toShortString()
                + " of=" + of.toShortString()
                + " cf=" + cf.toShortString() + " vf=" + vf.toShortString()
                + " dcf=" + dcf.toShortString()
                + " sf=" + sf.toShortString());


/frameworks/base/services/core/java/com/android/server/wm/WindowState.java

       mDeathRecipient = deathRecipient;
 //charlesvincent start
if("com.android.testred".equals(mAttrs.packageName)
		|| "com.android.testgreen".equals(mAttrs.packageName)
		|| "com.android.testblue".equals(mAttrs.packageName)){
		mBaseLayer = 21010;
		mSubLayer = 0;
		//mIsChildWindow = false;
		mLayoutAttached = false;
		mIsImWindow = mAttrs.type == TYPE_INPUT_METHOD
		|| mAttrs.type == TYPE_INPUT_METHOD_DIALOG;
		mIsWallpaper = mAttrs.type == TYPE_WALLPAPER;
                mAttachedWindow = attachedWindow;
                mIsFloatingLayer = mIsImWindow || mIsWallpaper;
} else{
		
        if ((mAttrs.type >= FIRST_SUB_WINDOW &&
                mAttrs.type <= LAST_SUB_WINDOW)) {
            // The multiplier here is to reserve space for multiple
            // windows in the same type layer.
            mBaseLayer = mPolicy.windowTypeToLayerLw(
                    attachedWindow.mAttrs.type) * WindowManagerService.TYPE_LAYER_MULTIPLIER
                    + WindowManagerService.TYPE_LAYER_OFFSET;
            mSubLayer = mPolicy.subWindowTypeToLayerLw(a.type);
            mAttachedWindow = attachedWindow;
            if (WindowManagerService.DEBUG_ADD_REMOVE) Slog.v(TAG, "Adding " + this + " to " + mAttachedWindow);

            int children_size = mAttachedWindow.mChildWindows.size();
            if (children_size == 0) {
                mAttachedWindow.mChildWindows.add(this);
            } else {
                for (int i = 0; i < children_size; i++) {
                    WindowState child = (WindowState)mAttachedWindow.mChildWindows.get(i);
                    if (this.mSubLayer < child.mSubLayer) {
                        mAttachedWindow.mChildWindows.add(i, this);
                        break;
                    } else if (this.mSubLayer > child.mSubLayer) {
                        continue;
                    }

                    if (this.mBaseLayer <= child.mBaseLayer) {
                        mAttachedWindow.mChildWindows.add(i, this);
                        break;
                    } else {
                        continue;
                    }
                }
                if (children_size == mAttachedWindow.mChildWindows.size()) {
                    mAttachedWindow.mChildWindows.add(this);
                }
            }

            mLayoutAttached = mAttrs.type !=
                    WindowManager.LayoutParams.TYPE_APPLICATION_ATTACHED_DIALOG;
            mIsImWindow = attachedWindow.mAttrs.type == TYPE_INPUT_METHOD
                    || attachedWindow.mAttrs.type == TYPE_INPUT_METHOD_DIALOG;
            mIsWallpaper = attachedWindow.mAttrs.type == TYPE_WALLPAPER;
            mIsFloatingLayer = mIsImWindow || mIsWallpaper;
        } else {
            // The multiplier here is to reserve space for multiple
            // windows in the same type layer.
            mBaseLayer = mPolicy.windowTypeToLayerLw(a.type)
                    * WindowManagerService.TYPE_LAYER_MULTIPLIER
                    + WindowManagerService.TYPE_LAYER_OFFSET;
            mSubLayer = 0;
            mAttachedWindow = null;
            mLayoutAttached = false;
            mIsImWindow = mAttrs.type == TYPE_INPUT_METHOD
                    || mAttrs.type == TYPE_INPUT_METHOD_DIALOG;
            mIsWallpaper = mAttrs.type == TYPE_WALLPAPER;
            mIsFloatingLayer = mIsImWindow || mIsWallpaper;
        }

}//zjj
//charlesvincent end

```

##### 4.4、TestViewportX代码分析
APK涉及AMS,WMS,SurfaceFlinger流程复杂，稍后另外博客再继续分析！

#### (五)、参考资料(特别感谢)：

[（1）【高通Android平台-应用空间操作framebuffer dump LCD总结】](https://blog.csdn.net/eliot_shao/article/details/74926010) 
[（2）【Linux 下framebuffer 帧缓冲的使用】](https://blog.csdn.net/eliot_shao/article/details/74926010) 
[（3）【Android Framebuffer介绍及使用】](https://www.jianshu.com/p/df1213e5a0ed) 
