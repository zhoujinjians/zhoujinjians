---
title: Android N Display System（4）：Android Display System 系统分析之Gralloc && HWComposer模块分析
cover: https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/hexo.themes/bing-wallpaper-2018.04.22.jpg
categories: 
  - Display
tags:
  - Android
  - Graphics
  - Display
toc: true
abbrlink: 20180708
date: 2018-07-08 09:25:00
---




--------------------------------------------------------------------------------

注：文章都是通过阅读各位前辈总结的资料、Android 7.1.2 && Linux（kernel 3.18）Qualcomm平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除，禁止转载（©Qualcomm Technologies, Inc. 版权所有），谢谢。

[【特别感谢 -  Android研究 Gralloc && HWComposer系列分析】](https://blog.csdn.net/putiancaijunyu/article/category/2558539)
[【特别感谢 -  Android display 系列分析】](https://blog.csdn.net/jinzhuojun/article/details/54234354)
[【特别感谢 -  Android图形显示之硬件抽象层Gralloc】](https://blog.csdn.net/yangwen123/article/details/12192401)

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
\hardware\libhardware\include\hardware
- fb.h

\hardware\libhardware\modules\gralloc
- framebuffer.cpp
- gralloc.cpp
- gralloc_priv.h
- gr.h
- mapper.cpp

\hardware\qcom\display\msm8996\libgralloc
- alloc_controller.cpp
- framebuffer.cpp
- gpu.cpp
- gralloc.cpp
- ionalloc.cpp
- mapper.cpp

\hardware\qcom\display\msm8996\libgralloc1
- gr_adreno_info.cpp
- gr_allocator.cpp
- gr_buf_mgr.cpp
- gr_device_impl.cpp
- gr_ion_alloc.cpp
- gr_utils.cpp

\frameworks\native\services\surfaceflinger
- DisplayDevice.cpp
- SurfaceFlinger.cpp
- MonitoredProducer.cpp
- SurfaceFlingerConsumer.cpp
- SurfaceFlinger_hwc1.cpp
- Client.cpp
- DispSync.cpp
- EventControlThread.cpp
- EventThread.cpp
- Layer.cpp
- MessageQueue.cpp

\frameworks\native\services\surfaceflinger\DisplayHardware
- FramebufferSurface.cpp
- HWC2.cpp
- HWC2On1Adapter.cpp
- HWComposer.cpp
- HWComposer_hwc1.cpp

--------------------------------------------------------------------------------


Linux系统下的显示驱动框架，每个显示屏被抽象为一个帧缓冲区，注册到FrameBuffer模块中，并在/dev/graphics目录下创建对应的fbX设备。Android系统在硬件抽象层中提供了一个Gralloc模块，封装了对帧缓冲区的所有访问操作。用户空间的应用程序在使用帧缓冲区之间，首先要加载Gralloc模块，并且获得一个gralloc设备和一个fb设备。有了gralloc设备之后，用户空间中的应用程序就可以申请分配一块图形缓冲区，并且将这块图形缓冲区映射到应用程序的地址空间来，以便可以向里面写入要绘制的画面的内容。最后，用户空间中的应用程序就通过fb设备来将已经准备好了的图形缓冲区渲染到帧缓冲区中去，即将图形缓冲区的内容绘制到显示屏中去。相应地，当用户空间中的应用程序不再需要使用一块图形缓冲区的时候，就可以通过gralloc设备来释放它，并且将它从地址空间中解除映射。

高通MSM8996 [Gralloc模块](Android 图形系统之gralloc) 实现源码位于：
\hardware\qcom\display\msm8996\libgralloc
每个硬件抽象层模块都必须定义HAL_MODULE_INFO_SYM符号，并且有自己唯一的ID，Gralloc也不例外，Gralloc模块ID定义为：

``` cpp
[->/hardware/libhardware/include/hardware/gralloc.h]
/**
 * The id of this module
 */
#define GRALLOC_HARDWARE_MODULE_ID "gralloc"
```
同时定义了以HAL_MODULE_INFO_SYM为符号的类型为 private_module_t的结构体：

``` cpp
hardware\libhardware\modules\gralloc\gralloc.cpp
// HAL module methods
static struct hw_module_methods_t gralloc_module_methods = {
    .open = gralloc_device_open
};

// HAL module initialize
struct private_module_t HAL_MODULE_INFO_SYM = {
    .base = {
        .common = {
            .tag = HARDWARE_MODULE_TAG,
            .version_major = 1,
            .version_minor = 0,
            .id = GRALLOC_HARDWARE_MODULE_ID,
            .name = "Graphics Memory Allocator Module",
            .author = "The Android Open Source Project",
            .methods = &gralloc_module_methods,
            .dso = 0,
            .reserved = {0},
        },
        .registerBuffer = gralloc_register_buffer,
        .unregisterBuffer = gralloc_unregister_buffer,
        .lock = gralloc_lock,
        .unlock = gralloc_unlock,
        .perform = gralloc_perform,
        .lock_ycbcr = gralloc_lock_ycbcr,
    },
    .framebuffer = 0,
    .fbFormat = 0,
    .flags = 0,
    .numBuffers = 0,
    .bufferMask = 0,
    .lock = PTHREAD_MUTEX_INITIALIZER,
};
```
通过[Gralloc模块加载](Android 图形系统之gralloc) 分析的方法将Gralloc模块加载到内存中来之后，就可以调用函数dlsym来获得它所导出的符号HMI，得到private_module_t的首地址后，由于private_module_t的第一个成员变量的类型为gralloc_module_t，因此也是gralloc_module_t的首地址，由于gralloc_module_t的第一个成员变量类型为hw_module_t，因此也是hw_module_t的首地址，因此只要得到这三种类型中其中一种类型变量的地址，就可以相互转换为其他两种类型的指针。

#### （一）、Gralloc模块 数据结构 
在分析Gralloc模块之前，首先介绍Gralloc模块定义的一些数据结构。private_module_t用于描述Gralloc模块下的系统帧缓冲区信息
``` cpp
[->\hardware\qcom\display\msm8996\libgralloc\fb_priv.h]
struct private_module_t {
    gralloc_module_t base;
    struct private_handle_t* framebuffer;//指向系统帧缓冲区的句柄  
    uint32_t fbFormat;
    uint32_t flags;//用来标志系统帧缓冲区是否支持双缓冲
    uint32_t numBuffers;//表示系统帧缓冲区包含有多少个图形缓冲区  
    uint32_t bufferMask;//记录系统帧缓冲区中的图形缓冲区的使用情况 
    pthread_mutex_t lock;//一个互斥锁，用来保护结构体private_module_t的并行访问
    struct fb_var_screeninfo info;//保存设备显示屏的动态属性信息  
    struct fb_fix_screeninfo finfo;//保存设备显示屏的固定属性信息
    float xdpi;//描述设备显示屏在宽度  
    float ydpi;//描述设备显示屏在高度  
    float fps;//用来描述显示屏的刷新频率  
    uint32_t swapInterval;
};

```
framebuffer_device_t用来描述系统帧缓冲区设备的信息

``` cpp
[->/hardware/libhardware/include/hardware/fb.h]
typedef struct framebuffer_device_t {
    /**
     * Common methods of the framebuffer device.  This *must* be the first member of
     * framebuffer_device_t as users of this structure will cast a hw_device_t to
     * framebuffer_device_t pointer in contexts where it's known the hw_device_t references a
     * framebuffer_device_t.
     */
    struct hw_device_t common;
    //用来记录系统帧缓冲区的标志
    const uint32_t  flags;
    //用来描述设备显示屏的宽度、高度 
    const uint32_t  width;
    const uint32_t  height;
    //用来描述设备显示屏的一行有多少个像素点  
    const int       stride;
    //用来描述系统帧缓冲区的像素格式
    const int       format;
    //用来描述设备显示屏在宽度上的密度、密度
    const float     xdpi;
    const float     ydpi;
    //用来描述设备显示屏的刷新频率  
    const float     fps;

    //用来设置帧缓冲区交换前后两个图形缓冲区的最小和最大时间间隔 
    const int       minSwapInterval;
    const int       maxSwapInterval;

    /* Number of framebuffers supported*/
    const int       numFramebuffers;

    int reserved[7];

    //用来设置帧缓冲区交换前后两个图形缓冲区的最小和最大时间间隔
    int (*setSwapInterval)(struct framebuffer_device_t* window,
            int interval);

    //用来设置帧缓冲区的更新区域
    int (*setUpdateRect)(struct framebuffer_device_t* window,
            int left, int top, int width, int height);

    //用来将图形缓冲区buffer的内容渲染到帧缓冲区中去
    int (*post)(struct framebuffer_device_t* dev, buffer_handle_t buffer);

    //用来通知fb设备，图形缓冲区的组合工作已经完成
    int (*compositionComplete)(struct framebuffer_device_t* dev);

    void (*dump)(struct framebuffer_device_t* dev, char *buff, int buff_len);
    int (*enableScreen)(struct framebuffer_device_t* dev, int enable);

    void* reserved_proc[6];

} framebuffer_device_t;

```
gralloc_module_t用于描述gralloc模块信息

``` cpp
[->\hardware\libhardware\include\hardware\gralloc.h]
typedef struct gralloc_module_t {
    struct hw_module_t common;
    //映射一块图形缓冲区到一个进程的地址空间去 
    int (*registerBuffer)(struct gralloc_module_t const* module,
            buffer_handle_t handle);
    //取消映射一块图形缓冲区到一个进程的地址空间去
    int (*unregisterBuffer)(struct gralloc_module_t const* module,
            buffer_handle_t handle);
    
    int (*lock)(struct gralloc_module_t const* module,
            buffer_handle_t handle, int usage,
            int l, int t, int w, int h,
            void** vaddr);
            
    int (*unlock)(struct gralloc_module_t const* module,
            buffer_handle_t handle);

    ......
    /* reserved for future use */
    void* reserved_proc[3];
} gralloc_module_t;

```
alloc_device_t用于描述gralloc设备的信息

``` cpp
[->\hardware\libhardware\include\hardware\gralloc.h]
typedef struct alloc_device_t {
    struct hw_device_t common;
    
    int (*alloc)(struct alloc_device_t* dev,
            int w, int h, int format, int usage,
            buffer_handle_t* handle, int* stride);
            
    int (*free)(struct alloc_device_t* dev,
            buffer_handle_t handle);
            
    void (*dump)(struct alloc_device_t *dev, char *buff, int buff_len);

    void* reserved_proc[7];
} alloc_device_t;
```

``` cpp
[->/hardware/libhardware/include/hardware/hardware.h]
typedef struct hw_module_t {  
    uint32_t tag;//标签  
　　uint16_t version_major;//模块主设备号  
　　uint16_t version_minor;//模块次设备号  
    const char *id;//模块ID  
    const char *name;//模块名称  
    const char *author;//模块作者  
    struct hw_module_methods_t* methods;//模块操作方法  
    void* dso;//保存模块首地址  
    uint32_t reserved[32-7];//保留位  
} hw_module_t;  
```

| 模块 |     设备|   作用|
| :-------- | --------:| :------: |
| private_module_t |   framebuffer_device_t |  将图形缓冲器映射到帧缓冲区|
| gralloc_module_t |   alloc_module_t |  分配或释放图形缓冲区|
| hw_module_t |   hw_module_t |  关联设备和模块|

硬件抽象层Gralloc模块定义了设备fb和设备gpu：

``` cpp
[->\hardware\libhardware\include\hardware\fb.h]
#define GRALLOC_HARDWARE_FB0 "fb0"
[->/hardware/libhardware/include/hardware/gralloc.h]
#define GRALLOC_HARDWARE_GPU0 "gpu0"
```

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/display.system/DS-04-01-GRALLOC_HARDWARE_.png)

设备gpu用于分配图形缓冲区，而设备fb用于渲染图形缓冲区；hw_module_t用于描述硬件抽象层Gralloc模块，而hw_device_t则用于描述硬件抽象层Gralloc设备，通过硬件抽象层设备可以找到对应的硬件抽象层模块。在Gralloc模块中，无论是定义fb设备还是gpu设备，都是用来处理图形缓冲区，以下是关于缓冲区的数据结构 定义：
private_handle_t用来描述一块缓冲区，Android对缓冲区的定义提供了C和C++两种方式，C++：
``` cpp
[->\hardware\qcom\display\msm8996\libgralloc\gralloc_priv.h]
struct private_handle_t : public native_handle {
#else
    struct private_handle_t {
        native_handle_t nativeHandle;
#endif
        enum {
            PRIV_FLAGS_FRAMEBUFFER        = 0x00000001,
            ......
        };

        // file-descriptors
        int     fd;
        int     fd_metadata;          // fd for the meta-data
        // ints
        int     magic;
        int     flags;
        unsigned int  size;
        unsigned int  offset;
        int     bufferType;
        ......
        int     format;
        int     width;
        int     height;
        ......
```
两种编译器下的private_handle_t定义都继承于native_handle，native_handle的定义如下：

``` cpp
[->/system/core/include/cutils/native_handle.h]
typedef struct native_handle  
{  
    int version; //设置为结构体native_handle_t的大小，用来标识结构体native_handle_t的版本  
    int numFds;  //表示结构体native_handle_t所包含的文件描述符的个数，这些文件描述符保存在成员变量data所指向的一块缓冲区中。  
    int numInts; //表示结构体native_handle_t所包含的整数值的个数，这些整数保存在成员变量data所指向的一块缓冲区中。  
    int data[0]; //指向的一块缓冲区中  
} native_handle_t;  
typedef const native_handle_t* buffer_handle_t;
```

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/display.system/DS-04-02-gralloc-hwt.png)


  下面就分析Gralloc模块中定义了两种设备的打开过程。


![Alt text | center](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/display.system/DS-04-03-GRALLOC_HARDWARE.png)



#### （二）、Fb设备打开过程
Fb设备打开过程是从SurfaceFlinger.init()函数通过HWComposer对象初始化过程中打开的

``` cpp
[->\frameworks\native\services\surfaceflinger\SurfaceFlinger_hwc1.cpp]
void SurfaceFlinger::init() {
    // initialize EGL for the default display
    mEGLDisplay = eglGetDisplay(EGL_DEFAULT_DISPLAY);
    eglInitialize(mEGLDisplay, NULL, NULL);

    // start the EventThread
    sp<VSyncSource> vsyncSrc = new DispSyncSource(&mPrimaryDispSync,
            vsyncPhaseOffsetNs, true, "app");
    mEventThread = new EventThread(vsyncSrc, *this);
    sp<VSyncSource> sfVsyncSrc = new DispSyncSource(&mPrimaryDispSync,
            sfVsyncPhaseOffsetNs, true, "sf");
    mSFEventThread = new EventThread(sfVsyncSrc, *this);
    mEventQueue.setEventThread(mSFEventThread);
    ......
    // Initialize the H/W composer object.  There may or may not be an
    // actual hardware composer underneath.
    mHwc = new HWComposer(this,
            *static_cast<HWComposer::EventHandler *>(this));
    ......
```
看看HWComposer构造函数

``` cpp
HWComposer::HWComposer(
        const sp<SurfaceFlinger>& flinger,
        EventHandler& handler)
    : mFlinger(flinger),
      mFbDev(0), mHwc(0), mNumDisplays(1),
      mCBContext(new cb_context),
      mEventHandler(handler),
      mDebugForceFakeVSync(false)
{
    ......
    // Note: some devices may insist that the FB HAL be opened before HWC.
    int fberr = loadFbHalModule();
    loadHwcModule();
    ......
}
int HWComposer::loadFbHalModule()
{
    hw_module_t const* module;

    int err = hw_get_module(GRALLOC_HARDWARE_MODULE_ID, &module);
    ......
    return framebuffer_open(module, &mFbDev);
}
```
Android系统在硬件抽象层中提供了一个Gralloc模块，封装了对framebuffer的所有访问操作。Gralloc模块符合Android标准的HAL架构设计。Gralloc对应的hardware id为：GRALLOC_HARDWARE_MODULE_ID

``` cpp
[->\hardware\libhardware\include\hardware\fb.h]
static inline int framebuffer_open(const struct hw_module_t* module,
        struct framebuffer_device_t** device) {
    return module->methods->open(module,
            GRALLOC_HARDWARE_FB0, (struct hw_device_t**)device);
}
```
用户空间的应用程序在使用帧缓冲区之前，首先要加载Gralloc模块，并且获得一个gpu0设备(gralloc_device, modulename:GRALLOC_HARDWARE_GPU0)和一个fb0设备(modulename:GRALLOC_HARDWARE_FB0)。

有了alloc设备之后，用户空间中的应用程序就可以申请分配一块图形缓冲区，并且将这块图形缓冲区映射到应用程序的地址空间来，以便可以向里面写入要绘制的画面的内容。最后，用户空间中的应用程序就通过fb0设备来将已经准备好了的图形缓冲区渲染到帧缓冲区中去，即将图形缓冲区的内容绘制到显示屏中去。相应地，当用户空间中的应用程序不再需要使用一块图形缓冲区的时候，就可以通过alloc设备来释放它，并且将它从地址空间中解除映射。

``` cpp
[->\hardware\qcom\display\msm8996\libgralloc\gralloc.cpp]
// Open Gralloc device
int gralloc_device_open(const hw_module_t* module, const char* name,
                        hw_device_t** device)
{
    int status = -EINVAL;
    if (!strcmp(name, GRALLOC_HARDWARE_GPU0)) {
        const private_module_t* m = reinterpret_cast<const private_module_t*>(
            module);
        gpu_context_t *dev;
        IAllocController* alloc_ctrl = IAllocController::getInstance();
        dev = new gpu_context_t(m, alloc_ctrl);
        if(!dev)
            return status;

        *device = &dev->common;
        status = 0;
    } else {
    //GRALLOC_HARDWARE_FB0,
        status = fb_device_open(module, name, device);
    }
    return status;
}

```

#####  2.1、Fb设备打开过程fb_device_open()
看下fb_device_open()函数实现过程：

``` cpp
[->\hardware\qcom\display\msm8996\libgralloc\framebuffer.cpp]

int fb_device_open(hw_module_t const* module, const char* name,
                   hw_device_t** device)
{
    int status = -EINVAL;
    if (!strcmp(name, GRALLOC_HARDWARE_FB0)) {
        alloc_device_t* gralloc_device;
        // 打开gralloc_device设备。GRALLOC_HARDWARE_GPU0
        status = gralloc_open(module, &gralloc_device);
        if (status < 0)
            return status;

        //创建一个fb_context_t对象，用来描述fb设备上下文  
        fb_context_t *dev = (fb_context_t*)malloc(sizeof(*dev));
        ......
        memset(dev, 0, sizeof(*dev));
        //初始化fb_context_t对象  
        /* initialize the procs */
        dev->device.common.tag      = HARDWARE_DEVICE_TAG;
        dev->device.common.version  = 0;
        dev->device.common.module   = const_cast<hw_module_t*>(module);
        //注册fb设备的操作函数
        dev->device.common.close    = fb_close;
        dev->device.setSwapInterval = fb_setSwapInterval;
        dev->device.post            = fb_post;
        dev->device.setUpdateRect   = 0;
        dev->device.compositionComplete = fb_compositionComplete;
        //将fb映射到当前进程地址空间 
        status = mapFrameBuffer((framebuffer_device_t*)dev);
        private_module_t* m = (private_module_t*)dev->device.common.module;
        if (status >= 0) {
            int stride = m->finfo.line_length / (m->info.bits_per_pixel >> 3);
            const_cast<uint32_t&>(dev->device.flags) = 0;
            const_cast<uint32_t&>(dev->device.width) = m->info.xres;
            const_cast<uint32_t&>(dev->device.height) = m->info.yres;
            const_cast<int&>(dev->device.stride) = stride;
            const_cast<int&>(dev->device.format) = m->fbFormat;
            const_cast<float&>(dev->device.xdpi) = m->xdpi;
            const_cast<float&>(dev->device.ydpi) = m->ydpi;
            const_cast<float&>(dev->device.fps) = m->fps;
            const_cast<int&>(dev->device.minSwapInterval) =
                                                        PRIV_MIN_SWAP_INTERVAL;
            const_cast<int&>(dev->device.maxSwapInterval) =
                                                        PRIV_MAX_SWAP_INTERVAL;
            const_cast<int&>(dev->device.numFramebuffers) = m->numBuffers;
            dev->device.setUpdateRect = 0;

            *device = &dev->device.common;
        }

        // Close the gralloc module
        gralloc_close(gralloc_device);
    }
    return status;
}

```
这个函数主要是用来创建一个fb_context_t结构体，并且对它的成员变量device进行初始化。结构体fb_context_t的成员变量device的类型为framebuffer_device_t，它是用来描述fb设备的。fb设备主要是用来渲染图形缓冲区的，这是通过调用它的成员函数post来实现的。函数fb_device_open所打开的fb设备的成员函数post被设置为Gralloc模块中的函数fb_post。函数mapFrameBuffer除了用来获得系统帧缓冲区的信息之外，还会将系统帧缓冲区映射到当前进程的地址空间来。line_length用来描述显示屏一行像素总共所占用的字节数，bits_per_pixel用来描述显示屏每一个像素所占用的位数，bits_per_pixel的值向右移3位，就可以得到显示屏每一个像素所占用的字节数。用显示屏像素总共所占用的字节数line_length除以每一个像素所占用的字节数就可以得到显示屏一行有多少个像素点，并保存在stride中。

#####  2.2、Fb设备地址空间映射mapFrameBuffer()

``` cpp
[->\hardware\qcom\display\msm8996\libgralloc\framebuffer.cpp]
static int mapFrameBuffer(framebuffer_device_t *dev)
{
    int err = -1;
    char property[PROPERTY_VALUE_MAX];
    if((property_get("debug.gralloc.map_fb_memory", property, NULL) > 0) &&
       (!strncmp(property, "1", PROPERTY_VALUE_MAX ) ||
        (!strncasecmp(property,"true", PROPERTY_VALUE_MAX )))) {
        private_module_t* module =
            reinterpret_cast<private_module_t*>(dev->common.module);
        pthread_mutex_lock(&module->lock);
        err = mapFrameBufferLocked(dev);
        pthread_mutex_unlock(&module->lock);
    }
    return err;
}
```
调用mapFrameBufferLocked函数执行映射过程，该函数在线程保护下完成。

``` cpp
[->\hardware\qcom\display\msm8996\libgralloc\framebuffer.cpp]
int mapFrameBufferLocked(framebuffer_device_t *dev)
{
    private_module_t* module =
        reinterpret_cast<private_module_t*>(dev->common.module);
    fb_context_t *ctx = reinterpret_cast<fb_context_t*>(dev);
    // already initialized...
    if (module->framebuffer) {
        return 0;
    }
    char const * const device_template[] = {
        "/dev/graphics/fb%u",
        "/dev/fb%u",
        0 };

    int fd = -1;
    int i=0;
    char name[64];
    char property[PROPERTY_VALUE_MAX];
    //检查是否存在设备文件/dev/graphics/fb0或者/dev/fb0。如果存在的话，那么就调用函数open来打开它，并且将得到的文件描述符保存在变量fd中 
    while ((fd==-1) && device_template[i]) {
        snprintf(name, 64, device_template[i], 0);
        fd = open(name, O_RDWR, 0);
        i++;
    }
    ......
    //通过IO控制命令FBIOGET_FSCREENINFO来获得系统帧缓冲区的固定信息，保存在fb_fix_screeninfo结构体finfo中  
    struct fb_fix_screeninfo finfo;
    if (ioctl(fd, FBIOGET_FSCREENINFO, &finfo) == -1) {
        close(fd);
        return -errno;
    }
    //通过IO控制命令FBIOGET_VSCREENINFO来获得系统帧缓冲区的可变信息，保存在fb_var_screeninfo结构体info中 
    struct fb_var_screeninfo info;
    if (ioctl(fd, FBIOGET_VSCREENINFO, &info) == -1) {
        close(fd);
        return -errno;
    }
    //初始化info  
    info.reserved[0] = 0;
    info.reserved[1] = 0;
    info.reserved[2] = 0;
    info.xoffset = 0;
    info.yoffset = 0;
    info.activate = FB_ACTIVATE_NOW;

    /* Interpretation of offset for color fields: All offsets are from the
     * right, inside a "pixel" value, which is exactly 'bits_per_pixel' wide
     * (means: you can use the offset as right argument to <<). A pixel
     * afterwards is a bit stream and is written to video memory as that
     * unmodified. This implies big-endian byte order if bits_per_pixel is
     * greater than 8.
     */

    if(info.bits_per_pixel == 32) {
        /*
         * Explicitly request RGBA_8888
         */
        info.bits_per_pixel = 32;
        info.red.offset     = 24;
        info.red.length     = 8;
        info.green.offset   = 16;
        info.green.length   = 8;
        info.blue.offset    = 8;
        info.blue.length    = 8;
        info.transp.offset  = 0;
        info.transp.length  = 8;

        /* Note: the GL driver does not have a r=8 g=8 b=8 a=0 config, so if we
         * do not use the MDP for composition (i.e. hw composition == 0), ask
         * for RGBA instead of RGBX. */
        if (property_get("debug.sf.hw", property, NULL) > 0 &&
                                                           atoi(property) == 0)
            module->fbFormat = HAL_PIXEL_FORMAT_RGBX_8888;
        else if(property_get("debug.composition.type", property, NULL) > 0 &&
                (strncmp(property, "mdp", 3) == 0))
            module->fbFormat = HAL_PIXEL_FORMAT_RGBX_8888;
        else
            module->fbFormat = HAL_PIXEL_FORMAT_RGBA_8888;
    } else {
        /*
         * Explicitly request 5/6/5
         */
        info.bits_per_pixel = 16;
        info.red.offset     = 11;
        info.red.length     = 5;
        info.green.offset   = 5;
        info.green.length   = 6;
        info.blue.offset    = 0;
        info.blue.length    = 5;
        info.transp.offset  = 0;
        info.transp.length  = 0;
        module->fbFormat = HAL_PIXEL_FORMAT_RGB_565;
    }

    //adreno needs 4k aligned offsets. Max hole size is 4096-1
    unsigned int size = roundUpToPageSize(info.yres * info.xres *
                                               (info.bits_per_pixel/8));

    /*
     * Request NUM_BUFFERS screens (at least 2 for page flipping)
     */
    int numberOfBuffers = (int)(finfo.smem_len/size);
    ALOGV("num supported framebuffers in kernel = %d", numberOfBuffers);

    if (property_get("debug.gr.numframebuffers", property, NULL) > 0) {
        int num = atoi(property);
        if ((num >= NUM_FRAMEBUFFERS_MIN) && (num <= NUM_FRAMEBUFFERS_MAX)) {
            numberOfBuffers = num;
        }
    }
    if (numberOfBuffers > NUM_FRAMEBUFFERS_MAX)
        numberOfBuffers = NUM_FRAMEBUFFERS_MAX;

    ALOGV("We support %d buffers", numberOfBuffers);

    //consider the included hole by 4k alignment
    uint32_t line_length = (info.xres * info.bits_per_pixel / 8);
    info.yres_virtual = (uint32_t) ((size * numberOfBuffers) / line_length);

    uint32_t flags = PAGE_FLIP;

    if (info.yres_virtual < ((size * 2) / line_length) ) {
        // we need at least 2 for page-flipping
        info.yres_virtual = (int)(size / line_length);
        flags &= ~PAGE_FLIP;
        ......
    }

    if (ioctl(fd, FBIOGET_VSCREENINFO, &info) == -1) {
        close(fd);
        return -errno;
    }

    if (int(info.width) <= 0 || int(info.height) <= 0) {
        info.width  = (uint32_t)(((float)(info.xres) * 25.4f)/160.0f + 0.5f);
        info.height = (uint32_t)(((float)(info.yres) * 25.4f)/160.0f + 0.5f);
    }

    float xdpi = ((float)(info.xres) * 25.4f) / (float)info.width;
    float ydpi = ((float)(info.yres) * 25.4f) / (float)info.height;
    ......

    //通过IO控制命令FBIOGET_VSCREENINFO来重新获得系统帧缓冲区的可变信息  
    if (ioctl(fd, FBIOGET_FSCREENINFO, &finfo) == -1) {
        close(fd);
        return -errno;
    }......
    module->flags = flags;
    module->info = info;
    module->finfo = finfo;
    module->xdpi = xdpi;
    module->ydpi = ydpi;
    module->fps = fps;
    module->swapInterval = 1;

    /*
     * map the framebuffer
     */

    module->numBuffers = info.yres_virtual / info.yres;
    module->bufferMask = 0;
   //整个系统帧缓冲区的大小=虚拟分辨率的高度值info.yres_virtual * 每一行所占用的字节数finfo.line_length,并将整个系统帧缓冲区的大小对齐到页面边界  
    unsigned int fbSize = roundUpToPageSize(finfo.line_length * info.yres)*
                    module->numBuffers;
   //系统帧缓冲区在当前进程的地址空间中的起始地址保存到private_handle_t的域base中  
    void* vaddr = mmap(0, fbSize, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
    ......
    //store the framebuffer fd in the ctx
    ctx->fbFd = fd;
   ......
   //创建一个private_handle_t，用来描述整个系统帧缓冲区的信息
    // Create framebuffer handle using the ION fd
    module->framebuffer = new private_handle_t(fd, fbSize,
                                        private_handle_t::PRIV_FLAGS_USES_ION,
                                        BUFFER_TYPE_UI,
                                        module->fbFormat, info.xres, info.yres);
    //以读写共享方式将帧缓冲区映射到当前进程地址空间中 
    
    module->framebuffer->base = uint64_t(vaddr);
    memset(vaddr, 0, fbSize);
    //Enable vsync
    int enable = 1;
    ioctl(ctx->fbFd, MSMFB_OVERLAY_VSYNC_CTRL, &enable);
    return 0;
}
```

##### 2.3、GPU设备打开过程gralloc_open()

``` cpp
[->\hardware\libhardware\include\hardware\gralloc.h]
/** convenience API for opening and closing a supported device */

static inline int gralloc_open(const struct hw_module_t* module, 
        struct alloc_device_t** device) {
    return module->methods->open(module, 
            GRALLOC_HARDWARE_GPU0, (struct hw_device_t**)device);
}
```
最终会走到gralloc_device_open()函数
``` cpp
[->\hardware\libhardware\include\hardware\gralloc.cpp]

// HAL module methods
static struct hw_module_methods_t gralloc_module_methods = {
    .open = gralloc_device_open
};
// Open Gralloc device
int gralloc_device_open(const hw_module_t* module, const char* name,
                        hw_device_t** device)
{
    int status = -EINVAL;
    if (!strcmp(name, GRALLOC_HARDWARE_GPU0)) {
        const private_module_t* m = reinterpret_cast<const private_module_t*>(
            module);
        gpu_context_t *dev;
        IAllocController* alloc_ctrl = IAllocController::getInstance();
        dev = new gpu_context_t(m, alloc_ctrl);
        ......
        *device = &dev->common;
        status = 0;
    } else {
        status = fb_device_open(module, name, device);
    }
    return status;
}

```

这个函数主要是用来创建一个gpu_context_t 结构体，并且对它的成员变量device进行初始化。gpu_context_t类继承了alloc_device_t，并实现了alloc_device_t中的alloc，free等方法。

``` cpp
[->\hardware\qcom\display\msm8996\libgralloc\gpu.cpp]
gpu_context_t::gpu_context_t(const private_module_t* module,
                             IAllocController* alloc_ctrl ) :
    mAllocCtrl(alloc_ctrl)
{
    // Zero out the alloc_device_t
    memset(static_cast<alloc_device_t*>(this), 0, sizeof(alloc_device_t));

    // Initialize the procs
    common.tag     = HARDWARE_DEVICE_TAG;
    common.version = 0;
    common.module  = const_cast<hw_module_t*>(&module->base.common);
    common.close   = gralloc_close;
    alloc          = gralloc_alloc;
    free           = gralloc_free;

}
```

主要是完成alloc_device_t参数的初始化。其成员函数alloc，free被设置成gralloc_alloc & gralloc_free。自此，alloc设备的打开过程就分析完成了。
接下来，我们重点分析alloc_device_t中提供的几个关键函数。
#### （三）、 Gralloc分配和释放Buffer

##### 3.1、Gralloc分配buffer
先来回忆一下SurfacFlinger图形缓冲区创建过程

``` cpp
GraphicBuffer::GraphicBuffer 
  -> initSize 
    -> GraphicBufferAllocator::alloc 
      -> alloc_device_t::alloc 
        -> gralloc_alloc
```

用户空间的应用程序用到的图形缓冲区是由Gralloc模块中的函数gralloc_alloc来分配的，这个函数实现

``` cpp
[->\hardware\qcom\display\msm8996\libgralloc\gpu.cpp]
int gpu_context_t::gralloc_alloc(alloc_device_t* dev, int w, int h, int format,
                                 int usage, buffer_handle_t* pHandle,
                                 int* pStride)
{
    gpu_context_t* gpu = reinterpret_cast<gpu_context_t*>(dev);
    return gpu->alloc_impl(w, h, format, usage, pHandle, pStride, 0);
}

int gpu_context_t::alloc_impl(int w, int h, int format, int usage,
                              buffer_handle_t* pHandle, int* pStride,
                              unsigned int bufferSize) {
   
    ......
     // 参数format用来描述要分配的图形缓冲区的颜色格式。这些格式定义在system/core/include/system/graphic.h中
    if(format == HAL_PIXEL_FORMAT_IMPLEMENTATION_DEFINED ||
       format == HAL_PIXEL_FORMAT_YCbCr_420_888) {
        if (usage & GRALLOC_USAGE_PRIVATE_ALLOC_UBWC)
            grallocFormat = HAL_PIXEL_FORMAT_YCbCr_420_SP_VENUS_UBWC;
        else if(usage & GRALLOC_USAGE_HW_VIDEO_ENCODER)
            grallocFormat = HAL_PIXEL_FORMAT_NV12_ENCODEABLE; //NV12
        ......
    }
    
    // 设置buffertype，BUFFER_TYPE_UI: RGB formats & HAL_PIXEL_FORMAT_R_8 &HAL_PIXEL_FORMAT_RG_88。其他的都为BUFFER_TYPE_VIDEO
    getGrallocInformationFromFormat(grallocFormat, &bufferType);
    // 根据formate & w，h算出buffersize
    size = getBufferSizeAndDimensions(w, h, grallocFormat, usage, alignedw,
                   alignedh);

    ......
    size = (bufferSize >= size)? bufferSize : size;

    int err = 0;
    if(useFbMem) {
        err = gralloc_alloc_framebuffer(usage, pHandle);
    } else {
        err = gralloc_alloc_buffer(size, usage, pHandle, bufferType,
                                   grallocFormat, alignedw, alignedh);
    }
    ......
    *pStride = alignedw;
    return 0;
}
```

最后根据memory alloc出处，区别调用gralloc_alloc_framebuffer&  gralloc_alloc_buffer函数。

首先来看看 gralloc_alloc_framebuffer的实现：

``` cpp
[->\hardware\qcom\display\msm8996\libgralloc\gpu.cpp]
int gpu_context_t::gralloc_alloc_framebuffer_locked(int usage,
                                                    buffer_handle_t* pHandle)
{
    
    // 变量bufferMask用来描述系统帧缓冲区的使用情况
    // 变量numBuffers用来描述系统帧缓冲区可以划分为多少个图形缓冲区来使用
    // 变量bufferSize用来描述设备显示屏一屏内容所占用的内存的大小,同时高通的硬件要求4K对齐。
    const unsigned int bufferMask = m->bufferMask;
    const uint32_t numBuffers = m->numBuffers;
    unsigned int bufferSize = m->finfo.line_length * m->info.yres;

    //adreno needs FB size to be page aligned
    bufferSize = roundUpToPageSize(bufferSize);

    
    // 假设此时系统帧缓冲区中尚有空闲的图形缓冲区的，接下来函数就会创建一个private_handle_t结构体hnd来描述这个即将要分配出去的图形缓冲区。注意，这个图形缓冲区的标志值等于PRIV_FLAGS_FRAMEBUFFER，即表示这是一块在系统帧缓冲区中分配的图形缓冲区。
    uint64_t vaddr = uint64_t(m->framebuffer->base);
    // As GPU needs ION FD, the private handle is created
    // using ION fd and ION flags are set
    private_handle_t* hnd = new private_handle_t(
        dup(m->framebuffer->fd), bufferSize,
        private_handle_t::PRIV_FLAGS_USES_ION |
        private_handle_t::PRIV_FLAGS_FRAMEBUFFER,
        BUFFER_TYPE_UI, m->fbFormat, m->info.xres,
        m->info.yres);
    //  接下来的for循环从低位到高位检查变量bufferMask的值，并且找到第一个值等于0的位，这样就可以知道在系统帧缓冲区中，第几个图形缓冲区的是空闲的。注意，变量vadrr的值开始的时候指向系统帧缓冲区的基地址，在下面的for循环中，每循环一次它的值都会增加bufferSize。从这里就可以看出，每次从系统帧缓冲区中分配出去的图形缓冲区的大小都是刚好等于显示屏一屏内容大小的。
    // find a free slot
    for (uint32_t i=0 ; i<numBuffers ; i++) {
        if ((bufferMask & (1LU<<i)) == 0) {
            m->bufferMask |= (uint32_t)(1LU<<i);
            break;
        }
        vaddr += bufferSize;
    }
    / 将分配的缓冲区的开始地址保存到变量base中，这样用户控件的应用程序可以直接将需要渲染的图形内容拷贝到这个地址上。这样，就相当于是直接将图形渲染到系统帧缓冲区中去。
// offset表示分配到的图形缓冲区的起始地址正对于系统帧缓冲区基地址的偏移量。
    hnd->base = vaddr;
    hnd->offset = (unsigned int)(vaddr - m->framebuffer->base);
    *pHandle = hnd;
    return 0;
}


int gpu_context_t::gralloc_alloc_framebuffer(int usage,
                                             buffer_handle_t* pHandle)
{
    private_module_t* m = reinterpret_cast<private_module_t*>(common.module);
    pthread_mutex_lock(&m->lock);
    int err = gralloc_alloc_framebuffer_locked(usage, pHandle);
    pthread_mutex_unlock(&m->lock);
    return err;
}

```
上面分析了从framebuffer中分配图形缓冲区的过程。总结下这块buffer的来历。首先在fb设备open的时候，通过mmap从fb0中映射一块内存到用户空间，即一个内存池（module->framebuffer)，通过bufferMask来表示该池中内存的使用情况。而alloc做的事情，就是从这个内存池中找到一个空闲的区块，然后返回该区块的hanlder指针pHandle。

我们现在来看看从内存中分配图形缓冲区的情况。从Android 4.0开始，Android启动新的内存管理方式ION，以取代PMEM。PMEM需要一个连续的物理内存，同时需要在系统启动的时候，就完成分配。

``` cpp
[->\hardware\qcom\display\msm8996\libgralloc\gpu.cpp]
int gpu_context_t::gralloc_alloc_buffer(unsigned int size, int usage,
                                        buffer_handle_t* pHandle, int bufferType,
                                        int format, int width, int height)
{
    int err = 0;
    int flags = 0;
    size = roundUpToPageSize(size);
    // 首先分配一个data区域
    alloc_data data;
    ......
    // 追查代码可以知道mallocCtrl指向IonController对象，关键代码可以参考hardware/qrom/display/msm8974/libgralloc/alloc_controller.cpp。具体怎么从ion中分配buffer
    data.size = size;
    data.pHandle = (uintptr_t) pHandle;
    err = mAllocCtrl->allocate(data, usage);

    if (!err) {
        /* allocate memory for enhancement data */
        alloc_data eData;
        ...
        int eDataErr = mAllocCtrl->allocate(eData, eDataUsage);
        ......
        flags |= data.allocType;
        uint64_t eBaseAddr = (uint64_t)(eData.base) + eData.offset;
        private_handle_t *hnd = new private_handle_t(data.fd, size, flags,
                bufferType, format, width, height, eData.fd, eData.offset,
                eBaseAddr);

        hnd->offset = data.offset;
        hnd->base = (uint64_t)(data.base) + data.offset;
        hnd->gpuaddr = 0;
        ColorSpace_t colorSpace = ITU_R_601;
        setMetaData(hnd, UPDATE_COLOR_SPACE, (void*) &colorSpace);

        *pHandle = hnd;
    }

    ALOGE_IF(err, "gralloc failed err=%s", strerror(-err));

    return err;
}
```
##### 3.2、Gralloc释放buffer
释放buffer本质是调用gralloc_free函数，该函数又调用了free_impl函数。在处理free buffer的时候，也是按照两种情况来分别处理的。如果之前这个buffer是从framebuffer分配的话，就只要把bufferMask中设置成0即可。而对应从内存中申请的，则是调用allocCtrl（ion）中的free_buffer来完成释放。

``` cpp
[->\hardware\qcom\display\msm8996\libgralloc\framebuffer.cpp]
int gpu_context_t::free_impl(private_handle_t const* hnd) {
    private_module_t* m = reinterpret_cast<private_module_t*>(common.module);
    if (hnd->flags & private_handle_t::PRIV_FLAGS_FRAMEBUFFER) {
        const unsigned int bufferSize = m->finfo.line_length * m->info.yres;
        unsigned int index = (unsigned int) ((hnd->base - m->framebuffer->base)
                / bufferSize);
        m->bufferMask &= (uint32_t)~(1LU<<index);
    } else {

        terminateBuffer(&m->base, const_cast<private_handle_t*>(hnd));
        IMemAlloc* memalloc = mAllocCtrl->getAllocator(hnd->flags);
        int err = memalloc->free_buffer((void*)hnd->base, hnd->size,
                                        hnd->offset, hnd->fd);
        if(err)
            return err;
        // free the metadata space
        unsigned int size = ROUND_UP_PAGESIZE(sizeof(MetaData_t));
        err = memalloc->free_buffer((void*)hnd->base_metadata,
                                    size, hnd->offset_metadata,
                                    hnd->fd_metadata);
        if (err)
            return err;
    }
    delete hnd;
    return 0;
} 
```
#### （四）、图形缓冲区映射过程

 图形缓冲区可以从系统帧缓冲区分配也可以从内存中分配，分配一个图形缓冲区后还需要将该图形缓冲区映射到分配该buffer的进程地址空间来，在Android系统中，图形缓冲区的管理由SurfaceFlinger服务来负责。在系统帧缓冲区中分配的图形缓冲区是在SurfaceFlinger服务中使用，而在内存中分配的图形缓冲区既可以在SurfaceFlinger服务中使用，也可以在其它的应用程序中使用。当其它的应用程序需要使用图形缓冲区的时候，它们就会请求SurfaceFlinger服务为它们分配并将SurfaceFlinger服务返回来的图形缓冲区映射到应用程序进程地址空间。在从内存中分配buffer时，已经将分配的buffer映射到了SurfaceFlinger服务进程地址空间，如果该buffer是应用程序请求SurfaceFlinger服务为它们分配的，那么还需要将SurfaceFlinger服务返回来的图形缓冲区映射到应用程序进程地址空间。

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/display.system/DS-04-04-grallocbuffer.jpg)


一个对象要在进程间传输必须继承于Flattenable类，并且实现flatten和unflatten方法，flatten方法用于序列化该对象，unflatten方法用于反序列化对象。

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/display.system/DS-04-05-graphicbuffer-flatten.jpg)


GraphicBuffer类从模板类Flattenable派生，这个派生类可以通过Parcel传递，通常派生类需要重载flatten和unflatten方法，用于对象的序列化和反序列化。

1）将一个对象写入到Parcel中，需要使用flatten函数序列化该对象，我们先来看下flatten函数：

``` cpp
[->\frameworks\native\libs\ui\GraphicBuffer.cpp]
status_t GraphicBuffer::flatten(void*& buffer, size_t& size, int*& fds, size_t& count) const {
    size_t sizeNeeded = GraphicBuffer::getFlattenedSize();
    if (size < sizeNeeded) return NO_MEMORY;

    size_t fdCountNeeded = GraphicBuffer::getFdCount();
    if (count < fdCountNeeded) return NO_MEMORY;

    int32_t* buf = static_cast<int32_t*>(buffer);
    buf[0] = 'GBFR';
    buf[1] = width;
    buf[2] = height;
    buf[3] = stride;
    buf[4] = format;
    buf[5] = usage;
    buf[6] = static_cast<int32_t>(mId >> 32);
    buf[7] = static_cast<int32_t>(mId & 0xFFFFFFFFull);
    buf[8] = 0;
    buf[9] = 0;

    if (handle) {
        buf[8] = handle->numFds;
        buf[9] = handle->numInts;
        native_handle_t const* const h = handle;
        //把handle中的data复制到fds中 
        memcpy(fds,     h->data,             h->numFds*sizeof(int));
        memcpy(&buf[10], h->data + h->numFds, h->numInts*sizeof(int));
    }

    buffer = reinterpret_cast<void*>(static_cast<int*>(buffer) + sizeNeeded);
    size -= sizeNeeded;
    if (handle) {
        fds += handle->numFds;
        count -= handle->numFds;
    }

    return NO_ERROR;
}
```
这个handle类型为native_handle_t ，且typedef成了buffer_handle_t，我们贴一下它的定义：

``` cpp
[->/system/core/include/cutils/native_handle.h]
typedef struct native_handle  
{  
    int version; //设置为结构体native_handle_t的大小，用来标识结构体native_handle_t的版本  
    int numFds;  //表示结构体native_handle_t所包含的文件描述符的个数，这些文件描述符保存在成员变量data所指向的一块缓冲区中。  
    int numInts; //表示结构体native_handle_t所包含的整数值的个数，这些整数保存在成员变量data所指向的一块缓冲区中。  
    int data[0]; //指向的一块缓冲区中  
} native_handle_t;
```
 所以我们回到flatten函数中，fds参数用来传递文件句柄，函数把handle中的表示指向图形缓冲区文件描述符句柄复制到fds中，因此这些句柄就能通过binder传递到目标进程中去。

2）在应用程序读取来自服务进程的GraphicBuffer对象时，也就是result = reply.read(*p)，会调用GraphicBuffer类的unflatten函数进行反序列化过程：

``` cpp
[->\frameworks\native\libs\ui\GraphicBuffer.cpp]
status_t GraphicBuffer::unflatten(
        void const*& buffer, size_t& size, int const*& fds, size_t& count) {
    if (size < 8*sizeof(int)) return NO_MEMORY;

    int const* buf = static_cast<int const*>(buffer);
    if (buf[0] != 'GBFR') return BAD_TYPE;

    const size_t numFds  = buf[8];
    const size_t numInts = buf[9];

    const size_t sizeNeeded = (10 + numInts) * sizeof(int);
    if (size < sizeNeeded) return NO_MEMORY;

    size_t fdCountNeeded = 0;
    if (count < fdCountNeeded) return NO_MEMORY;

    if (handle) {
        // free previous handle if any
        free_handle();
    }

    if (numFds || numInts) {
        width  = buf[1];
        height = buf[2];
        stride = buf[3];
        format = buf[4];
        usage  = buf[5];
        //创建一个native_handle对象
        native_handle* h = native_handle_create(numFds, numInts);
        //将fds复制到native_handle对象的data中，和flatten操作相反
        memcpy(h->data,          fds,     numFds*sizeof(int));
        memcpy(h->data + numFds, &buf[10], numInts*sizeof(int));
        handle = h;
    } else {
        width = height = stride = format = usage = 0;
        handle = NULL;
    }

    mId = static_cast<uint64_t>(buf[6]) << 32;
    mId |= static_cast<uint32_t>(buf[7]);

    mOwner = ownHandle;

    if (handle != 0) {
        //使用GraphicBufferMapper将服务端创建的图形缓冲区映射到当前进程地址空间  
        status_t err = mBufferMapper.registerBuffer(handle);
        if (err != NO_ERROR) {
            width = height = stride = format = usage = 0;
            handle = NULL;
            ......
            return err;
        }
    }

    buffer = reinterpret_cast<void const*>(static_cast<int const*>(buffer) + sizeNeeded);
    size -= sizeNeeded;
    fds += numFds;
    count -= numFds;

    return NO_ERROR;
}
```

调用unflatten函数时，共享区的文件句柄已经准备好了，但是内存还没有进行映射，调用了mBufferMapper.registerBuffer函数来进行内存映射。
##### 4.1、图形缓冲区的注册过程
``` cpp
status_t GraphicBufferMapper::registerBuffer(const GraphicBuffer* buffer)
{
    gralloc1_error_t error = mDevice->retain(buffer);
    return error;
}
```
用了mDevice->retain(buffer)函数，

``` cpp
[->\frameworks\native\libs\ui\Gralloc1On0Adapter.cpp]
gralloc1_error_t Gralloc1On0Adapter::retain(
        const std::shared_ptr<Buffer>& buffer)
{
    std::lock_guard<std::mutex> lock(mBufferMutex);
    buffer->retain();
    return GRALLOC1_ERROR_NONE;
}

gralloc1_error_t Gralloc1On0Adapter::retain(
        const android::GraphicBuffer* graphicBuffer)
{
    
    ......
    buffer_handle_t handle = graphicBuffer->getNativeBuffer()->handle;
    std::lock_guard<std::mutex> lock(mBufferMutex);

    ALOGV("Calling registerBuffer(%p)", handle);
    int result = mModule->registerBuffer(mModule, handle);
    ......
}

```
经过一系列步骤的调用

``` cpp
[->/hardware/qcom/display/msm8996/libgralloc/mapper.cpp]
int gralloc_register_buffer(gralloc_module_t const* module,
                            buffer_handle_t handle)
{
    ......
    int err =  gralloc_map(module, handle);
    ......
    return err;
}

static int gralloc_map(gralloc_module_t const* module,
                       buffer_handle_t handle)
{
    ......
    private_handle_t* hnd = (private_handle_t*)handle;
    unsigned int size = 0;
    int err = 0;
    IMemAlloc* memalloc = getAllocator(hnd->flags) ;
    void *mappedAddress = MAP_FAILED;
    hnd->base = 0;

    // Dont map framebuffer and secure buffers
    if (!(hnd->flags & private_handle_t::PRIV_FLAGS_FRAMEBUFFER) &&
        !(hnd->flags & private_handle_t::PRIV_FLAGS_SECURE_BUFFER)) {
        size = hnd->size;
        err = memalloc->map_buffer(&mappedAddress, size,
                                       hnd->offset, hnd->fd);
        ......

        hnd->base = uint64_t(mappedAddress) + hnd->offset;
    } else {
        err = -EACCES;
    }

    //Allow mapping of metadata for all buffers including secure ones, but not
    //of framebuffer
    int metadata_err = gralloc_map_metadata(handle);
    return err;
}
```
进一步调用

``` cpp
[->/hardware/qcom/display/msm8996/libgralloc/ionalloc.cpp]
int IonAlloc::map_buffer(void **pBase, unsigned int size, unsigned int offset,
        int fd)
{
    ATRACE_CALL();
    int err = 0;
    void *base = 0;
    // It is a (quirky) requirement of ION to have opened the
    // ion fd in the process that is doing the mapping
    err = open_device(); 
    ......
    base = mmap(0, size, PROT_READ| PROT_WRITE,
                MAP_SHARED, fd, 0);
    *pBase = base;
    .......
    return err;
}
```
这个函数就是调用了mmap来进行共享内存的映射。
##### 4.2、图形缓冲区的释放过程
释放过程调用流程类似，最后会调用unmap_buffer()释放图像缓冲区。
##### 4.3、小结

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/display.system/DS-04-06-sf-app-dup-mmap.png)

#### （五）HWComposer模块
前面分析HWComposer构造函数没有分析loadHwcModule()函数，
loadHwcModule()函数用来加载HWC模块，我们继续查看：

``` cpp
[E->\frameworks\native\services\surfaceflinger\DisplayHardware\HWComposer_hwc1.cpp]
HWComposer::HWComposer(
        const sp<SurfaceFlinger>& flinger,
        EventHandler& handler)
    : mFlinger(flinger),
      mFbDev(0), mHwc(0), mNumDisplays(1),
      mCBContext(new cb_context),//这里直接new了一个设备上下文对象
      mEventHandler(handler),
      mDebugForceFakeVSync(false)
{
    ......
    //装载HWComposer的硬件模块,这个函数中会将mHwc置为true
    loadHwcModule();
    ......
    //硬件vsync信号
    if (mHwc) {
        ALOGI("Using %s version %u.%u", HWC_HARDWARE_COMPOSER,
              (hwcApiVersion(mHwc) >> 24) & 0xff,
              (hwcApiVersion(mHwc) >> 16) & 0xff);
        if (mHwc->registerProcs) {
            //HWComposer设备上下文变量mCBContext赋值
            mCBContext->hwc = this;
            //函数指针钩子函数hook_invalidate放入上下文
            mCBContext->procs.invalidate = &hook_invalidate;
            //vsync钩子函数放入上下文
            mCBContext->procs.vsync = &hook_vsync;
            if (hwcHasApiVersion(mHwc, HWC_DEVICE_API_VERSION_1_1))
                //hotplug狗子函数放入上下文
                mCBContext->procs.hotplug = &hook_hotplug;
            else
                mCBContext->procs.hotplug = NULL;
            memset(mCBContext->procs.zero, 0, sizeof(mCBContext->procs.zero));
            //将钩子函数注册进硬件设备，硬件驱动回调这些钩子函数
            mHwc->registerProcs(mHwc, &mCBContext->procs);
        }

        // don't need a vsync thread if we have a hardware composer
        //如果有硬件vsync信号， 则不需要软件vsync实现
        needVSyncThread = false;
        
        ......
    }
}
// Load and prepare the hardware composer module.  Sets mHwc.
void HWComposer::loadHwcModule()
{
    hw_module_t const* module;
    //同样是HAL层封装的函数，参数是HWC_HARDWARE_MODULE_ID，加载hwc模块
    if (hw_get_module(HWC_HARDWARE_MODULE_ID, &module) != 0) {
        return;
    }
    //打开hwc设备
    int err = hwc_open_1(module, &mHwc);
    ......
}
```

如果硬件设备打开成功，则将钩子函数hook_invalidate、hook_vsync和hook_hotplug注册进硬件设备，作为回调函数。这三个都是硬件产生事件信号，通知上层SurfaceFlinger的回调函数，用于处理这个信号。

因为我们本节是Vsync信号相关，所以我们只看看hook_vsync钩子函数。这里指定了vsync的回调函数是hook_vsync，如果硬件中产生了VSync信号，将通过这个函数来通知上层，看看它的代码：

``` cpp
[E->\frameworks\native\services\surfaceflinger\DisplayHardware\HWComposer_hwc1.cpp]
void HWComposer::hook_vsync(const struct hwc_procs* procs, int disp,
        int64_t timestamp) {
    cb_context* ctx = reinterpret_cast<cb_context*>(
            const_cast<hwc_procs_t*>(procs));
    ctx->hwc->vsync(disp, timestamp);
}
```
  hook_vsync钩子函数会调用vsync函数，我们继续看：

``` cpp
[E->\frameworks\native\services\surfaceflinger\DisplayHardware\HWComposer_hwc1.cpp]
void HWComposer::vsync(int disp, int64_t timestamp) {
    if (uint32_t(disp) < HWC_NUM_PHYSICAL_DISPLAY_TYPES) {
        {
            Mutex::Autolock _l(mLock);
            mLastHwVSync[disp] = timestamp;
        }

        char tag[16];
        snprintf(tag, sizeof(tag), "HW_VSYNC_%1u", disp);
        ATRACE_INT(tag, ++mVSyncCounts[disp] & 1);
        //这里调用EventHandler类型变量mEventHandler就是SurfaceFlinger，
        //所以调用了SurfaceFlinger的onVSyncReceived函数
        mEventHandler.onVSyncReceived(disp, timestamp);
    }
}
```
mEventHandler对象类型为EventHandler，我们在SurfaceFlinger的init函数创建HWComposer类实例时候讲SurfaceFlinger强转为EventHandler作为构造函数的参数传入其中。再者SurfaceFlinger继承HWComposer::EventHandler，所以最终会调用SurfaceFlinger的onVSyncReceived函数，这就是硬件vsync信号的产生。

##### 5.1、HWC设备打开过程

``` cpp
[->/hardware/libhardware/include/hardware/hwcomposer.h]
/** convenience API for opening and closing a device */
static inline int hwc_open_1(const struct hw_module_t* module,
        hwc_composer_device_1_t** device) {
    return module->methods->open(module,
            HWC_HARDWARE_COMPOSER, (struct hw_device_t**)device);
}
```
具体实现/hardware/qcom/display/msm8996/sdm/libs/hwc/ or /hardware/qcom/display/msm8996/sdm/libs/hwc2/。

``` cpp
[->/hardware/qcom/display/msm8996/sdm/libs/hwc/hwc_session.h]
  struct HWCModuleMethods : public hw_module_methods_t {
    HWCModuleMethods() {
      hw_module_methods_t::open = HWCSession::Open;
    }
  };
[->/hardware/qcom/display/msm8996/sdm/libs/hwc/hwc_session.cpp]
int HWCSession::Open(const hw_module_t *module, const char *name, hw_device_t **device) {
  ......
  if (!strcmp(name, HWC_HARDWARE_COMPOSER)) {
    HWCSession *hwc_session = new HWCSession(module);
    ......

    int status = hwc_session->Init();
    ......
    hwc_composer_device_1_t *composer_device = hwc_session;
    *device = reinterpret_cast<hw_device_t *>(composer_device);
  }

  return 0;
}

HWCSession::HWCSession(const hw_module_t *module) {
  // By default, drop any events. Calls will be routed to SurfaceFlinger after registerProcs.
  hwc_procs_default_.invalidate = Invalidate;
  hwc_procs_default_.vsync = VSync;
  hwc_procs_default_.hotplug = Hotplug;

  hwc_composer_device_1_t::common.tag = HARDWARE_DEVICE_TAG;
  hwc_composer_device_1_t::common.version = HWC_DEVICE_API_VERSION_1_5;
  hwc_composer_device_1_t::common.module = const_cast<hw_module_t*>(module);
  hwc_composer_device_1_t::common.close = Close;
  hwc_composer_device_1_t::prepare = Prepare;
  hwc_composer_device_1_t::set = Set;
  hwc_composer_device_1_t::eventControl = EventControl;
  hwc_composer_device_1_t::setPowerMode = SetPowerMode;
  hwc_composer_device_1_t::query = Query;
  hwc_composer_device_1_t::registerProcs = RegisterProcs;
  hwc_composer_device_1_t::dump = Dump;
  hwc_composer_device_1_t::getDisplayConfigs = GetDisplayConfigs;
  hwc_composer_device_1_t::getDisplayAttributes = GetDisplayAttributes;
  hwc_composer_device_1_t::getActiveConfig = GetActiveConfig;
  hwc_composer_device_1_t::setActiveConfig = SetActiveConfig;
  hwc_composer_device_1_t::setCursorPositionAsync = SetCursorPositionAsync;
}
```

#### （六）、参考资料(特别感谢各位前辈的分析和图示)：
[ Android研究 Gralloc && HWComposer系列分析](https://blog.csdn.net/putiancaijunyu/article/category/2558539)
[Android Display 系列分析](https://blog.csdn.net/jinzhuojun/article/details/54234354)
[Android display: framebuffer 映射关系](https://blog.csdn.net/honour2sword/article/details/12004289)
[Android display框架与数据流](https://blog.csdn.net/honour2sword/article/details/38265879)
[Android图形显示之硬件抽象层Gralloc](https://blog.csdn.net/yangwen123/article/details/12192401)
[SurfaceFlinger中Buffer的创建与显示](https://www.jianshu.com/p/af5858c06d5d)
[Android 图形系统之gralloc](https://www.wolfcstech.com/2017/09/21/android_graphics_gralloc/)
[深入剖析Android系统 显示模块](https://blog.csdn.net/yangwen123/article/category/1647761)
[Android SurfaceFlinger 学习之路](http://windrunnerlihuan.com/)

