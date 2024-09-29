---
title: Android N Display System（2）：Android Display System 系统分析之Android EGL && OpenGL
cover: https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/hexo.themes/bing-wallpaper-2018.04.20.jpg
categories: 
  - Display
tags:
  - Android
  - Graphics
  - Display
toc: true
abbrlink: 20180508
date: 2018-05-08 09:25:00
---



--------------------------------------------------------------------------------

注：文章都是通过阅读各位前辈总结的资料、Android 7.1.2 && Linux（kernel 3.18）Qualcomm平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除，禁止转载（©Qualcomm Technologies, Inc. 版权所有），谢谢。

[【特别感谢 - Android Graphics and Android EGL】](http://www.kandroid.org/board/data/board/conference/file_in_body/1/3AndroidGraphicsAndAndroidEGL.pdf)
[【特别感谢 - Android 4.4 (KitKat) Design Pattern-Graphics Subsystem】](https://blog.csdn.net/jinzhuojun/article/details/17427491)
[【特别感谢 - Android 4.4 (KitKat) in virtualization VSync signal】](https://blog.csdn.net/jinzhuojun/article/details/17293325)
[【特别感谢 - Android显示系统设计框架介绍】](https://blog.csdn.net/yangwen123/article/details/22647255)
[【特别感谢 - 图解Android - Android GUI 系统 (2) - 窗口管理 (View, Canvas, Window Manager)】](http://www.cnblogs.com/samchen2009/p/3367496.html)

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

> **Android EGL、GLES_CM、GLES2：**

\frameworks\native\opengl\libs\EGL
- egl.cpp
- egl_display.cpp
- eglApi.cpp
- Loader.cpp

\frameworks\native\opengl\libs\GLES_CM
- gl.cpp
- gl_api.in
- glext_api.in

\frameworks\native\opengl\libs\GLES2
- gl2.cpp
- gl2_api.in
- gl2ext_api.in

> **OpenGL Native && JNI ：**

\frameworks\base\core\jni\
- android_opengl_GLES10.cpp
- android_opengl_GLES10Ext.cpp
- android_opengl_GLES11.cpp
- android_opengl_GLES11Ext.cpp
- android_opengl_GLES20.cpp
- android_opengl_GLES30.cpp
- android_opengl_GLES31.cpp
- android_opengl_GLES31Ext.cpp
- android_opengl_GLES32.cpp
- com_google_android_gles_jni_EGLImpl.cpp
- com_google_android_gles_jni_GLImpl.cpp

> **Opengl Java：**

\frameworks\base\opengl\java\android\opengl
- GLES10.java
- GLES10Ext.java
- GLLogWrapper.java
- GLSurfaceView.java
- EGLDisplay.java
- EGLConfig.java
- EGLContext.java
- EGLSurface.java

\frameworks\base\opengl\java\javax\microedition\khronos\opengles
- GL10.java
- GL11.java

\frameworks\base\opengl\java\com\google\android\gles_jni
- GLImpl.java
- EGLImpl.java
- EGLConfigImpl.java
- EGLContextImpl.java
- EGLDisplayImpl.java

--------------------------------------------------------------------------------


总体架构：

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/display.system/DS-02-01-Android-Graphics-Architecture-EGL.png)

[【Android Display System（1）- Android Graphics 系统 分析】](http://charlesvincent.cc/2018/02/01/Android-7-1-2-Android-N-Android-Graphics-%E7%B3%BB%E7%BB%9F%E5%88%86%E6%9E%90/)
####（一）、 Android EGL 应用实例
##### 1.1、Android Graphics 测试程序
首先看一下Android测试程序：

``` cpp 
参考\frameworks\native\services\surfaceflinger\tests\Transaction_test.cpp
拷贝同目录下.mk文件push到手机运行即可看到效果。

#include <gtest/gtest.h>

#include <android/native_window.h>

#include <binder/IMemory.h>

#include <gui/ISurfaceComposer.h>
#include <gui/Surface.h>
#include <gui/SurfaceComposerClient.h>
#include <private/gui/ComposerService.h>
#include <private/gui/LayerState.h>

#include <utils/String8.h>
#include <ui/DisplayInfo.h>

#include <math.h>

namespace android {

// Fill an RGBA_8888 formatted surface with a single color.
static void fillSurfaceRGBA8(const sp<SurfaceControl>& sc,
        uint8_t r, uint8_t g, uint8_t b) {
    ANativeWindow_Buffer outBuffer;
    sp<Surface> s = sc->getSurface();
    ASSERT_TRUE(s != NULL);
    ASSERT_EQ(NO_ERROR, s->lock(&outBuffer, NULL));
    uint8_t* img = reinterpret_cast<uint8_t*>(outBuffer.bits);
    for (int y = 0; y < outBuffer.height; y++) {
        for (int x = 0; x < outBuffer.width; x++) {
            uint8_t* pixel = img + (4 * (y*outBuffer.stride + x));
            pixel[0] = r;
            pixel[1] = g;
            pixel[2] = b;
            pixel[3] = 255;
        }
    }
    ASSERT_EQ(NO_ERROR, s->unlockAndPost());
}

class LayerTest : public ::testing::Test {
protected:
    virtual void SetUp() {
        mComposerClient = new SurfaceComposerClient;
        ASSERT_EQ(NO_ERROR, mComposerClient->initCheck());

        sp<IBinder> display(SurfaceComposerClient::getBuiltInDisplay(
                ISurfaceComposer::eDisplayIdMain));
        DisplayInfo info;
        SurfaceComposerClient::getDisplayInfo(display, &info);

        ssize_t displayWidth = info.w;
        ssize_t displayHeight = info.h;

        // Background surface black
        mBGSurfaceControl = mComposerClient->createSurface(
                String8("BG Test Surface"), displayWidth, displayHeight-720,
                PIXEL_FORMAT_RGBA_8888, 0);
        ASSERT_TRUE(mBGSurfaceControl != NULL);
        ASSERT_TRUE(mBGSurfaceControl->isValid());
        fillSurfaceRGBA8(mBGSurfaceControl, 0, 0, 0);

        // Foreground surface red
        mFGSurfaceControl = mComposerClient->createSurface(
                String8("FG Test Surface"), 360, 360, PIXEL_FORMAT_RGBA_8888, 0);
        ASSERT_TRUE(mFGSurfaceControl != NULL);
        ASSERT_TRUE(mFGSurfaceControl->isValid());

        fillSurfaceRGBA8(mFGSurfaceControl, 255, 0, 0);

        // Foreground surface blue
        mFGSurfaceControlBlue = mComposerClient->createSurface(
                String8("FG Test Surface"), 360, 360, PIXEL_FORMAT_RGBA_8888, 0);
        ASSERT_TRUE(mFGSurfaceControlBlue != NULL);
        ASSERT_TRUE(mFGSurfaceControlBlue->isValid());

        fillSurfaceRGBA8(mFGSurfaceControl, 0, 255, 0);


        // Foreground surface green
        mFGSurfaceControlGreen = mComposerClient->createSurface(
                String8("FG Test Surface"), 360, 360, PIXEL_FORMAT_RGBA_8888, 0);
        ASSERT_TRUE(mFGSurfaceControlGreen != NULL);
        ASSERT_TRUE(mFGSurfaceControlGreen->isValid());

        fillSurfaceRGBA8(mFGSurfaceControl, 0, 0, 255);



        // Synchronization surface
        mSyncSurfaceControl = mComposerClient->createSurface(
                String8("Sync Test Surface"), 1, 1, PIXEL_FORMAT_RGBA_8888, 0);
        ASSERT_TRUE(mSyncSurfaceControl != NULL);
        ASSERT_TRUE(mSyncSurfaceControl->isValid());

        fillSurfaceRGBA8(mSyncSurfaceControl, 31, 31, 31);

	    //SurfaceComposerClient::openGlobalTransaction()

        SurfaceComposerClient::openGlobalTransaction();

        mComposerClient->setDisplayLayerStack(display, 0);
        //black
        ASSERT_EQ(NO_ERROR, mBGSurfaceControl->setLayer(INT_MAX-4));
        ASSERT_EQ(NO_ERROR, mBGSurfaceControl->show());
        //red
        ASSERT_EQ(NO_ERROR, mFGSurfaceControl->setLayer(INT_MAX-3));
        ASSERT_EQ(NO_ERROR, mFGSurfaceControl->setPosition(0, 0));
        ASSERT_EQ(NO_ERROR, mFGSurfaceControl->show());
		//blue
		ASSERT_EQ(NO_ERROR, mFGSurfaceControlBlue->setLayer(INT_MAX-2));
        ASSERT_EQ(NO_ERROR, mFGSurfaceControlBlue->setPosition(360, 360));
        ASSERT_EQ(NO_ERROR, mFGSurfaceControlBlue->show());
		//green
		ASSERT_EQ(NO_ERROR, mFGSurfaceControlGreen->setLayer(INT_MAX-1));
        ASSERT_EQ(NO_ERROR, mFGSurfaceControlGreen->setPosition(720, 720));
        ASSERT_EQ(NO_ERROR, mFGSurfaceControlGreen->show());

        ASSERT_EQ(NO_ERROR, mSyncSurfaceControl->setLayer(INT_MAX-1));
        ASSERT_EQ(NO_ERROR, mSyncSurfaceControl->setPosition(displayWidth-2,
                displayHeight-2));
        ASSERT_EQ(NO_ERROR, mSyncSurfaceControl->show());

        SurfaceComposerClient::closeGlobalTransaction(true);

		waitForPostedBuffers();
    }

    virtual void TearDown() {
        mComposerClient->dispose();
        mBGSurfaceControl = 0;
        mFGSurfaceControl = 0;
        mSyncSurfaceControl = 0;
        mComposerClient = 0;
    }

    void waitForPostedBuffers() {
        // Since the sync surface is in synchronous mode (i.e. double buffered)
        // posting three buffers to it should ensure that at least two
        // SurfaceFlinger::handlePageFlip calls have been made, which should
        // guaranteed that a buffer posted to another Surface has been retired.
        fillSurfaceRGBA8(mFGSurfaceControl, 255, 0, 0);
        fillSurfaceRGBA8(mFGSurfaceControlBlue, 0, 255, 0);
        fillSurfaceRGBA8(mFGSurfaceControlGreen, 0, 0, 255);
	    fillSurfaceRGBA8(mSyncSurfaceControl, 0, 0, 0);
    }

    sp<SurfaceComposerClient> mComposerClient;
    sp<SurfaceControl> mBGSurfaceControl;
    sp<SurfaceControl> mFGSurfaceControl;//red
    sp<SurfaceControl> mFGSurfaceControlBlue;
    sp<SurfaceControl> mFGSurfaceControlGreen;

    // This surface is used to ensure that the buffers posted to
    // mFGSurfaceControl have been picked up by SurfaceFlinger.
    sp<SurfaceControl> mSyncSurfaceControl;
};

TEST_F(LayerTest, LayerWorks) {

    SurfaceComposerClient::openGlobalTransaction();
    ASSERT_EQ(NO_ERROR, mFGSurfaceControl->setPosition(0, 0));

	
    SurfaceComposerClient::closeGlobalTransaction(true);

	for(;;){
	sp<IBinder> display(SurfaceComposerClient::getBuiltInDisplay(
			ISurfaceComposer::eDisplayIdMain));
	DisplayInfo info;
	SurfaceComposerClient::getDisplayInfo(display, &info);
	
	ssize_t displayWidth = info.w;
	ssize_t displayHeight = info.h;

	SurfaceComposerClient::openGlobalTransaction();
	
	mComposerClient->setDisplayLayerStack(display, 0);
	
	ASSERT_EQ(NO_ERROR, mBGSurfaceControl->setLayer(INT_MAX-4));
	ASSERT_EQ(NO_ERROR, mBGSurfaceControl->show());
	//red
	ASSERT_EQ(NO_ERROR, mFGSurfaceControl->setLayer(INT_MAX-3));
	ASSERT_EQ(NO_ERROR, mFGSurfaceControl->setPosition(0, 0));
	ASSERT_EQ(NO_ERROR, mFGSurfaceControl->show());
	//blue
	ASSERT_EQ(NO_ERROR, mFGSurfaceControlBlue->setLayer(INT_MAX-2));
	ASSERT_EQ(NO_ERROR, mFGSurfaceControlBlue->setPosition(360, 360));
	ASSERT_EQ(NO_ERROR, mFGSurfaceControlBlue->show());
	//green
	ASSERT_EQ(NO_ERROR, mFGSurfaceControlGreen->setLayer(INT_MAX-1));
	ASSERT_EQ(NO_ERROR, mFGSurfaceControlGreen->setPosition(720, 720));
	ASSERT_EQ(NO_ERROR, mFGSurfaceControlGreen->show());
	//Sync
	ASSERT_EQ(NO_ERROR, mSyncSurfaceControl->setLayer(INT_MAX-1));
	ASSERT_EQ(NO_ERROR, mSyncSurfaceControl->setPosition(displayWidth-2,
			displayHeight-2));
	ASSERT_EQ(NO_ERROR, mSyncSurfaceControl->show());


	SurfaceComposerClient::closeGlobalTransaction(true);

	}

}
```

实现效果（保持运行，可以看到界面最顶层会绘制黑色背景和红绿蓝三个色块）：

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/display.system/DS-02-02-Android-graphics-surface-test.gif)

可以看到比较关键的代码：

``` cpp
        //创建SurfaceComposerClient
        mComposerClient = new SurfaceComposerClient;
        ASSERT_EQ(NO_ERROR, mComposerClient->initCheck());
        //获取display信息
        sp<IBinder> display(SurfaceComposerClient::getBuiltInDisplay(
                ISurfaceComposer::eDisplayIdMain));
        DisplayInfo info;
        SurfaceComposerClient::getDisplayInfo(display, &info);

        ssize_t displayWidth = info.w;
        ssize_t displayHeight = info.h;

        // Background surface black
        //请求SurfaceFlinger创建Surface
        mBGSurfaceControl = mComposerClient->createSurface(
                String8("BG Test Surface"), displayWidth, displayHeight-720,
                PIXEL_FORMAT_RGBA_8888, 0);
        //填充Surface
        fillSurfaceRGBA8(mBGSurfaceControl, 0, 0, 0);
        //设置Layer层级
        ASSERT_EQ(NO_ERROR, mBGSurfaceControl->setLayer(INT_MAX-4));
        //SurfaceControl->show()显示surface
        ASSERT_EQ(NO_ERROR, mBGSurfaceControl->show());
```
这部分的分析请参考[【Android Display System（1）- Android Graphics 系统 分析】](http://charlesvincent.cc/2018/02/01/Android-7-1-2-Android-N-Android-Graphics-%E7%B3%BB%E7%BB%9F%E5%88%86%E6%9E%90/)
我们这里主要为了引出Android底层如何利用EGL绘图。
##### 1.2、Android BootAnimation 开机动画(EGL在Android中应用)

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/display.system/DS-02-03-Android-boot-egl.png)


关键代码：

``` cpp
[->\frameworks\base\cmds\bootanimation\BootAnimation.cpp]
status_t BootAnimation::readyToRun() {
    mAssets.addDefaultAssets();
    sp<IBinder> dtoken(SurfaceComposerClient::getBuiltInDisplay(
            ISurfaceComposer::eDisplayIdMain));
    DisplayInfo dinfo;
    status_t status = SurfaceComposerClient::getDisplayInfo(dtoken, &dinfo);
  
    // create the native surface
    sp<SurfaceControl> control = session()->createSurface(String8("BootAnimation"),
            dinfo.w, dinfo.h, PIXEL_FORMAT_RGB_565);

    SurfaceComposerClient::openGlobalTransaction();
    control->setLayer(0x40000000);
    SurfaceComposerClient::closeGlobalTransaction();

    sp<Surface> s = control->getSurface();

    // initialize opengl and egl
    ......
    // (1)、获得 EGLDisplay 对象。
    EGLDisplay display = eglGetDisplay(EGL_DEFAULT_DISPLAY);
    // (2)、初始化 EGLDisplay 对象
    eglInitialize(display, 0, 0);
    // (3)、选择 EGLConfig
    eglChooseConfig(display, attribs, &config, 1, &numConfigs);
    // (4)、创建 Windows Surface
    surface = eglCreateWindowSurface(display, config, s.get(), NULL);
    // (5)、创建 EGL context
    context = eglCreateContext(display, config, NULL, NULL);
    eglQuerySurface(display, surface, EGL_WIDTH, &w);
    eglQuerySurface(display, surface, EGL_HEIGHT, &h);
    // (6)、启用前面创建的 EGL context
    if (eglMakeCurrent(display, surface, surface, context) == EGL_FALSE)
        return NO_INIT;
    }
bool BootAnimation::playAnimation(const Animation& animation)
{
    ......
                // (7)、OpenGL ES API 绘制图形：gl_*()
                glDrawTexiOES(xc, mHeight - (yc + frame.trimHeight),
                              0, frame.trimWidth, frame.trimHeight);
                ......
                // (8)、SwapBuffers显示
                eglSwapBuffers(mDisplay, mSurface);
                ......

    }
```
使用EGL一般会经历以上几个步骤。
##### 1.2、Understanding Android Graphics Internals
要深入了解Android Graphics机制，需要了解熟悉以下知识（包括但不限于）。

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/display.system/DS-02-04-understand-android-graphics-internals.png)

####（二）、OpenGL ES 2.0 知识串讲
在了解EGL之前，先来看看前人总结的知识[OPENGL ES 2.0 知识串讲](http://geekfaner.com/shineengine/index.html)：
##### 2.1、写在前面的话
##### 2.1.1、电脑是做什么用的?

电脑又被称为计算机,那么最重要的工作就是计算。看过三体的同学都知道, 电脑中有无数纳米级别的计算单元,通过 0 和 1 的转换,完成加减乘除的操作。

##### 2.1.2、是什么使得电脑工作?

驱动,驱使着硬件完成工作。

##### 2.1.3、谁来写驱动?

制造电脑的公司自己来写驱动,因为他们对自己的底层硬件架构最熟悉。

##### 2.1.4、谁会使用驱动?

所有的软件工程师都会直接或者间接的使用到驱动。

那么问题来了,如果说不同的电脑公司,制造出来不同的硬件,使用不同的 驱动,提供出来不同的接口供软件工程师进行使用,那么软件工程师就要崩溃了。

所以,一定是需要一个标准,来统一一下。

##### 2.1.5、那么在哪里进行统一?

硬件没有办法统一,每个电脑公司为了优化自己电脑性能和功耗,制造出来 不同的硬件架构,这是需要无数的心血完成的,如果统一了,那么就不需要那么 多电脑公司了。

所以只能统一驱动的接口。

电脑组件大致分为:CPU、GPU、内存、总线等。而 OpenGL 就是 GPU 驱动 的一套标准接口(OpenGL ES 为嵌入式设备 GPU 驱动的标准接口,比如手机, OpenGL ES 全称:OpenGL for Embedded Systems)。

所以综上所述,我使用了 5 个问题,引出了 OpenGL 的用处:就是将复杂的、 各种各样的 GPU 硬件包装起来,各个电脑公司编写自家的驱动,然后提供出来 一套统一的接口,供上层软件工程师调用。这样,世界就和平了。

##### 2.1.6、谁这么牛,定义了 OpenGL 这套标准?

Khronos。每当我打这几个字母的时候,都会抱有一种敬畏的心理,因为它 不是一家公司,它是一个组织,它是由众多大公司联合组建而来,比如 Apple、 Intel、AMD、Google、ARM、Qualcomm、Nvidia 等等等等。各个大公司投入了大 量的人力、资金等创建了这个组织。对电脑 GPU 定义了统一的接口 OpenGL,对 手机 GPU 定义了统一的接口 OpenGL ES(我也非常有幸,在 Intel 工作期间,跟 Intel 驻 Khronos 的 3D 负责人共事了一段时间,每周一次的跨洋电话,都会让我受益匪浅)

这个组织除了定义了 OpenGL 接口之外,还定义了很多其他接口。目前针对 GPU 又提出了另外一套更底层的接口 Vulkan,这是一套比 OpenGL 更底层的接口, 使用其可以更容易优化,不过目前硬件厂商的驱动还有待开发,可能普及 Vulkan 还需要很多年。就好比 OpenGL ES 已经发展到了 3.1,而市面上的手机很多还是 只能支持 OpenGL ES 2.0 一样。所以新的科技从提出,到实现,到量产,到使用, 到普及,是一段很长的路。

所以,我们现在学习 OpenGL ES 2.0 是适时的,且是非常必要的(不懂 2.0, 想直接学习更难的 3.0、3.1、Vulkan,很难)。

事先预告一下,OpenGL ES 2.0 会分十三个课程,结束之后,我会立即奉上 OpenGL ES 3.0 在 OpenGL ES 2.0 基础上的改变。

##### 2.1.7、OpenGL 和我们游戏（Android ）开发者有什么关系?

电脑/手机屏幕上显示的东西,要么是 2D 的,要么是 3D 的,那么如果是 3D 的,不管是 App 也好,游戏也好,简单的图片界面也好,底层都是通过 GPU、 通过 OpenGL(ES)绘制出来的。

开发 App 的时候,是通过创建控件的方式,而控件已经对底层进行了一层封装,所以 App 开发者很少会接触到 OpenGL(ES)。

游戏的开发是通过游戏引擎,而游戏引擎的最底层,是直接调用了 OpenGL(ES),直接对 GPU 进行控制。

所以说游戏引擎工程师必须懂 OpenGL(ES),而游戏开发者,想要更好的对游戏进行更好的理解和优化,也建议学一些 OpenGL(ES)。

##### 2.1.8、DirectX 是什么?

最后一个问题。我们发现 Khronos 组织的成员中,我没有提到大名鼎鼎的微 软,因为微软不在组织中,而它提出了自己的 GPU 驱动标准,DirectX。

所以目前手机,不管是 iOS 还是 Android,都是支持 OpenGL ES。电脑,Windows 系统支持 DirectX 和 OpenGL,Linux/Mac(Unix)系统支持 OpenGL。

#### 2.2、OpenGL ES 的两个小伙伴

虽然,我们教程的标题是 OpenGL ES,但是我们的内容将不仅限于 OpenGL ES。 OpenGL ES 是负责 GPU 工作的,目的是通过 GPU 计算,得到一张图片,这张图 片在内存中其实就是一块 buffer,存储有每个点的颜色信息等。而这张图片最终是要显示到屏幕上,所以还需要具体的窗口系统来操作,OpenGL ES 并没有相关的函数。所以,OpenGL ES 有一个好搭档 EGL。

EGL,全称:embedded Graphic Interface,是 OpenGL ES 和底层 Native 平台 视窗系统之间的接口。所以大概流程是这样的:首先,通过 EGL 获取到手机屏幕 的 handle,获取到手机支持的配置(RGBA8888/RGB565 之类,表示每个像素中包 含的颜色等信息的存储空间是多少位),然后根据这个配置创建一块包含默认 buffer 的 surface(buffer 的大小是根据屏幕分辨率乘以每个像素信息所占大小计 算而得)和用于存放 OpenGL ES 状态集的 context,并将它们 enable 起来。然后, 通过 OpenGL ES 操作 GPU 进行计算,将计算的结果保存在 surface 的 buffer 中。 最后,使用 EGL,将绘制的图片显示到手机屏幕上。

而在 OpenGL ES 操作 GPU 计算的时候,还需要介绍 OpenGL ES 的另外一个好搭档 GLSL。

GLSL,全称:OpenGL Shading Language,是 OpenGL ES 中使用到的着色器的 语言,用这个语言可以编写小程序运行在 GPU 上。

在这里需要先提到 CPU 和 GPU 的区别,它们的功能都是用于计算,也都是由很多核组成,区别在于 CPU 的核比较少,但是单个核的计算能力比较强,而 GPU 的核很多,但是每个核的计算能力都不算特别强。目前 GPU 的主要工作是用于生成图片(现在也有通过 GPU 进行高性能运算_并行运算,但是在这里不属于讨论的范围),原因就是图片是由很多像素组成,每个像素都包含有颜色、深度等信息,而为了得到这些信息数据,针对每个像素点的计算,是可以通过统一的算法来完成。GPU 就擅长处理针对这种大规模数据,使用同一个算法进行计算。而这个算法,就是使用 GLSL 写成 Shader,供 GPU 运算使用。

在图形学的视角中,所有的图片都是由三角形构成的。所以通过 OpenGL ES 绘制图片的时候,我们需要通过 OpenGL ES API 创建用于在 GPU 上运行的 shader, 然后将通过 CPU 获取到的图片顶点信息,传入 GPU 中的 Shader 中。在 Vertex Shader 中通过矩阵变换,将顶点坐标从模型坐标系转换到世界坐标系,再到观察坐标系,到裁剪坐标系,最后投影到屏幕坐标系中,计算出在屏幕上各个顶点的坐标。然后,通过光栅化,以插值的方法得到所有像素点的信息,并在 Fragment shader 中计算出所有像素点的颜色。最后,通过 OpenGL ES 的 API 设定的状态,将得到的像素信息进行 depth/stencil test、blend,得到最终的图片。

#### 2.3、屏幕图片的本质和产生过程
当我们买一个手机的时候,我们会非常关注这个手机的分辨率。分辨率代表着像素的多少,比如我们熟知的 iphone6 的分辨率为 1334×750,而 iphone6 plus 的分辨率是1920×1080。

手机屏幕上的图片,是由一个一个的像素组成,那么可以计算出来,一个屏幕上的图片,是由上百万个像素点组成。而每个像素点都有自己的颜色,每种颜色都是由 RGB 三原色组成。三原色按照不同的比例混合,组成了手机所能显示出来的颜色。

每个像素的颜色信息都保存在 buffer 中,这块 buffer 可以分给 RGB 每个通 道各 8bit 进行信息保存,也可以分给 RGB 每个通道不同的空间进行信息保存, 比如由于人眼对绿色最敏感,那么可以分配给 G 通道 6 位,R 和 B 通道各 5 位。这些都是常见的手机配置。假如使用 RGB888 的手机配置,也就是每种颜色的取值从 0 到 255,0 最小,255 最大。那么红绿蓝都为 0 的时候,这个像素点的颜色就是黑色,红绿蓝都为 255 的时候,这个像素点的颜色就是白色。当红为 255, 绿蓝都为 0 的时候,这个像素点的颜色就是红色。当红绿为 255,蓝为 0 的时候, 这个像素点的颜色就是黄色。当然不是只取 0 或者 255,可以取 0-255 中间的值, 100,200,任意在 0 和 255 中间的值都没有问题。那么我们可以算一下,按照红绿蓝不同比例进行搭配,每个像素点,可以显示的颜色有 255*255*255=16581375 种,这个数字是非常恐怖,所以我们的手机可以显示出来各种各样的颜色。 这里在延伸的科普一下,我们看到手机可以显示那么多种颜色了,但是是不是说我们的手机在颜色上就已经发展到极致了呢?其实是远远没有的,在这个手机配置下,三原色中每一种的取值可以从 0 到 255,而在现实生活中,它们的取 值可以从 0 到 1 亿,而我们人类的眼睛所能看到的范围是,从 0 到 10 万。所以手机硬件还存在很大的提升空间。而在手机硬件提升之前,我们也可以通过 HDR 等技术尽量的在手机中多显示一些颜色。所以,讲到这里,我们知道了,手机屏幕上显示的图片,是由这上百万个像素点,以及这上百万个像素点对应的颜色组成的。

用程序员的角度来看,就是手机屏幕对应着一块 buffer,这块 buffer 对应上百万个像素点,每个像素点需要一定的空间来存储其颜色。如果使用更加形象的例子来比喻,手机屏幕对应的 buffer 就好像一块巨大的棋盘,棋盘上有上百万个格子,每个格子都有自己的颜色,那么从远处整体的看这个棋盘,就是我们看手机的时候显示的样子。这就是手机屏幕上图片的本质。

通过我们对 EGL、GLSL、OpenGL ES 的理解,借助一张图片,从专业的角度来解释一下手机屏幕上的图片是如何生成的。

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/display.system/DS-02-05-OpenGL-Picture-Generate.png)


首先,通过 EGL 获取手机屏幕,进而获取到手机屏幕对应的这个棋盘,同时, 在手机的 GPU 中根据手机的配置信息,生成另外一个的棋盘和一个本子,本子是用于记录这个棋盘初始颜色等信息。

然后,OpenGL ES 就好像程序员的画笔,程序员需要知道自己想画什么东西,比如想画一个苹果,那么就需要通过为数不多的基本几何图元(如点、直线、三 角形)来创建所需要的模型。比如用几个三角形和点和线来近似的组成这个苹果 (图形学的根本就是点、线和三角形,所有的图形,都可以由这些基本图形组成, 比如正方形或者长方形,就可以由两个三角形组成,圆形可以由无数个三角形组成,只是三角形的数量越多,圆形看上去越圆润)。

根据这些几何图元,建立数学描述,比如每个三角形或者线的顶点坐标位置、每个顶点的颜色。得到这些信息之后,可以先通过 OpenGL ES 将 EGL 生成的棋盘 (buffer)进行颜色初始化,一般会被初始化为黑色。然后将刚才我们获取到的顶点坐标位置,通过矩阵变化的方式,进行模型变换、观察变换、投影变换,最后映射到屏幕上,得到屏幕上的坐标。这个步骤可以在 CPU 中完成,也就是在 OpenGL ES 把坐标信息传给 Shader 之前,在 CPU 中通过矩阵相乘等方式进行更新,或者是直接把坐标信息通过 OpenGL ES 传给 Shader,同时也把矩阵信息传给 Shader,通过 Shader 在 GPU 端进行坐标更新,更新的算法通过 GLSL 写在 Shader 中。这个进行坐标更新的 Shader 被称为 vertex shader,简称 VS,是 OpenGL ES2.0, 也是 GLSL130 版本对应的最重要两个 shader 之一,作用是完成顶点操作阶段中的所有操作。经过矩阵变换后的像素坐标信息,为屏幕坐标系中的坐标信息。在 VS 中,最重要的输入为顶点坐标、矩阵(还可以传入顶点的颜色、法线、纹理 坐标等信息),而最重要的运算结果,就是这个将要显示在屏幕上的坐标信息。 VS 会针对传入的所有顶点进行运算,比如在 OpenGL ES 中只想绘制一个三角形 和一条线,这两个图元不共享顶点,那么在 VS 中,也就传入了 5 个顶点信息, 根据矩阵变换,这 5 个顶点的坐标转换成了屏幕上的顶点坐标信息,从图上显示, 也就是从左上角的图一,更新成了中上图的图二。

再然后,当图二生成之后,我们知道了图元在屏幕上的顶点位置,而顶点的颜色在 VS 中没有发生变化,所以图元的顶点颜色我们也是知道的。下面就是根据 OpenGL ES 中设置的状态,表明哪些点连成线,哪些点组成三角形,进行图元装配,也就是我们在右上角的图三中看到的样子。这个样子在 GPU 中不会显示, 那几条线也是虚拟的线,是不会显示在棋盘 buffer 中的,而 GPU 做的是光珊化,这一步是发生在从 VS 出来,进入另外一个Shader (Pixel shader,也称 fragment shader)之前,在 GPU 中进行的。作用是把线上,或者三角形内部所有的像素点找到,并根据插值或者其他方式计算出其颜色等信息(如果不通过插值,可以使用其他的方法,这些在 OpenGL ES 和 GLSL 中都可以进行设置)。也就生成了下面一行的图四和图五。

我们大概可以看到在图 4 和图 5 种出现了大量的顶点,大概数一下估计有 40 个点左右,这些点全部都会进入 PS 进行操作,在 PS 中可以对这些点的颜色进行操作,比如可以只显示这些点的红色通道,其他的绿蓝通道的值设置为 0, 比如之前某个点的 RGB 为 200,100,100。在 PS 中可以将其通过计算,更新为 200,0,0。这样做的结果就是所显示的图片均为红色,只是深浅不同。这也就好像戴上了一层红色的滤镜,其他颜色均为滤掉了。所以用 PS 来做滤镜是非常方便的。再比如,假如一盏红色的灯照到了苹果上,那么显示出来的颜色就是在苹果原本的颜色基础上,红色值进行一定的增值。

所以,总结一下,经过 VS 和 PS 之后,程序员想要画的东西,就已经被画出来了。想要绘制的东西,也就是左下角图五的样子。然后再根据 OpenGL ES 的设置,对新绘制出来的东西进行 Depth/Stencil Test,剔除掉被遮挡的部分,将剩余部分与原图片进行 Blend,生成新的图片。 最后,通过 EGL,把这个生成的棋盘 buffer 和手机屏幕上对应的棋盘 buffer 进行调换,让手机屏幕显示这个新生成的棋盘,旧的那个棋盘再去绘制新的图片信息。周而复始,不停的把棋盘进行切换,也就像过去看连环画一样,动画就是由一幅幅的图片组成,当每秒切换的图片数量超过 30 张的时候,我们的手机也就看到了动态的效果。这就是屏幕上图片的产生过程。

在这里再进行一下延伸,这个例子中,VS 计算了 5 个顶点的数据,PS 计算 了大概 40 个顶点的数据,而我们刚才说过,手机中存在上百万个像素点,这上百万个像素点都可以是顶点,那么这个计算量是非常大的。而这也是为什么要将 shader 运算放在 GPU 中的原因,因为 GPU 擅长进行这种运算。

我们知道 CPU 现在一般都是双核或者 4 核,多的也就是 8 核或者 16 核,但是 GPU 动辄就是 72 核,多的还有上千核,这么多核的目的就是进行并行运算, 虽然单个的 GPU 核不如 CPU 核,但是单个的 GPU 核足够进行加减乘除运算,所以大量的 GPU 核用在图形学像素点运算上,是非常有效的。而 CPU 虽然单个很强大,而且也可以通过多级流水来提高吞吐率,但是终究还是不如 GPU 的多核来得快。但是在通过 GPU 进行多核运算的时候,需要注意的是:如果 shader 中存放判断语句,就会对 GPU 造成比较大的负荷,不同 GPU 的实现方式不同,多数 GPU 会对判断语句的两种情况都进行运算,然后根据判断结果取其中一个。

我们通过这个例子再次清楚了 OpenGL ES 绘制的整个流程,而这个例子也是最简单的一个例子,其中有很多 OpenGL ES 的其他操作没有被涉及到。比如,我们绘制物体的颜色大多是从纹理中采样出来,那么设计到通过 OpenGL ES 对纹理 进行操作。而 OpenGL ES 的这些功能,我们会在下面一点一点进行学习。

#### 2.4.2、OpenGL 流水线（pipeline）

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/display.system/DS-02-06-OpenGL-ES-pipeline.png)


EGL 是用于与手机设备打交道,比如获取绘制 buffer,将绘制 buffer 展现到手机屏幕中。那么抛开 EGL 不说,OpenGL ES 与 GLSL 的主要功能,就是往这块 buffer 上绘制图片。

所以,我们可以把OpenGL ES和GLSL的流程单独拿出来进行归纳总结,而这幅流程图就是著名的 OpenGL ES2.0 pipeline。

首先,最左边的 API 指的就是 OpenGL ES 的 API,OpenGL ES 其实是一个图形学库,由 109 个 API 组成,只要明白了这 109 个 API 的意义和用途,就掌握了OpenGL ES 2.0。

然后,我们通过 API 先设定了顶点的信息,顶点的坐标、索引、颜色等信息,将这些信息传入 VS。

在 VS 中进行运算,得到最终的顶点坐标。再把算出来的顶点坐标进行图元装配,构建成虚拟的线和三角形。再进行光珊化(在光珊化的时候,把顶点连接起来形成直线,或者填充多边形的时候,需要考虑直线和多边形的直线宽度、点的大小、渐变算法以及是否使用支持抗锯齿处理的覆盖算法。最终的每个像素点,都具有各自的颜色和深度值)。

将光珊化的结果传入 PS,进行最终的颜色计算。

然后,这所谓最终的结果在被实际存储到绘制 buffer 之前,还需要进行一系列的操作。这些操作可能会修改甚至丢弃这些像素点。

这些操作主要为 alpha test、Depth/Stencil test、Blend、Dither。

Alpha Test 采用一种很霸道极端的机制,只要一个像素的 alpha 不满足条件, 那么它就会被 fragment shader 舍弃,被舍弃的 fragments 不会对后面的各种 Tests 产生影响;否则,就会按正常方式继续下面的检验。Alpha Test 产生的效果也很极端,要么完全透明,即看不到,要么完全不透明。

Depth/stencil test 比较容易理解。由于我们绘制的是 3D 图形,那么坐标为 XYZ,而 Z 一般就是深度值,OpenGL ES 可以对深度测试进行设定,比如设定深度值大的被抛弃,那么假如绘制 buffer 上某个像素点的深度值为 0,而 PS 输出的 像素点的深度值为 1,那么 PS 输出的像素点就被抛弃了。而 stencil 测试更加简单,其又被称为蒙版测试,比如可以通过 OpenGL ES 设定不同 stencil 值的配抛弃, 那么假如绘制 buffer 上某个像素点的 stencil 值为 0,而 PS 输出的像素点的 stencil 值为 1,那么 PS 输出的像素点就被抛弃了。

既然说到了 Depth/stencil,那么就在这里说一下绘制 buffer 到底有多大,存 储了多少信息。按照我们刚才的说法,手机可以支持一百万个像素,那么生成的 绘制 buffer 就需要存储这一百万个像素所包含的信息,而每个像素包含的信息, 与手机配置有关,假如手机支持 Depth/stencil。那么通过 EGL 获取的绘制 buffer 中,每个像素点就包含了 RGBA 的颜色值,depth 值和 stencil 值,其中 RGBA 每个分量一般占据 8 位,也就是 8bit,也就是 1byte,而 depth 大多数占 24 位,stencil 占 8 位。所以每个像素占 64bit,也就是 8byte。那么 iphone6 plus 的绘制 buffer 的尺寸为 1920×1080×8=16588800byte=16200KB=15.8MB。

下面还有 blend,通过 OpenGL ES 可以设置 blend 混合模式。由于绘制 buffer 中原本每个像素点已经有颜色了,那么 PS 输出的颜色与绘制 buffer 中的颜色如何混合,生成新的颜色存储在绘制 buffer 中,就是通过 blend 来进行设定。

最后的 dither,dither 是一种图像处理技术,是故意造成的噪音,用以随机化量化误差,阻止大幅度拉升图像时,导致的像 banding(色带)这样的问题。也 是通过OpenGL ES 可以开启或者关闭。

经过了这一系列的运算和测试,也就得到了最终的像素点信息,将其存储到绘制 buffer 上之后,OpenGL ES 的 pipeline 也就结束了。

整个pipeline中，纵向按照流水线作业，横线按照独立作业，多级并行、提高渲染性能


####（三）、 Android EGL Overview： OpenGL ES 和 EGL 介绍

#### 3.1.0、OpenGL ES
OpenGL ES（OpenGL for Embedded Systems）是 OpenGL 三维图形API的子集，针对手机、PDA和游戏主机等嵌入式设备而设计，各显卡制造商和系统制造商来实现这组 API。
#### 3.1.1、OpenGL 基本概念
OpenGL 的结构可以从逻辑上划分为下面 3 个部分：

☯ 图元（Primitives）
☯ 缓冲区（Buffers）
☯ 光栅化（Rasterize）
 **图元（Primitives）**
在 OpenGL 的世界里，我们只能画点、线、三角形这三种基本图形，而其它复杂的图形都可以通过三角形来组成。所以这里的图元指的就是这三种基础图形：

☯ 点：点存在于三维空间，坐标用（x,y,z）表示。
☯ 线：由两个三维空间中的点组成。
☯ 三角形：由三个三维空间的点组成。
**缓冲区（Buffers）**
OpenGL 中主要有 3 种 Buffer：

**帧缓冲区（Frame Buffers）** 帧缓冲区：**这个是存储OpenGL 最终渲染输出结果的地方**，它是一个包含多个图像的集合，例如颜色图像、深度图像、模板图像等。

**渲染缓冲区（Render Buffers）** 渲染缓冲区：渲染缓冲区就是一个图像，它是 Frame Buffer 的一个子集。

**缓冲区对象（Buffer Objects）** 缓冲区对象就是程序员输入到 OpenGL 的数据，分为结构类和索引类的。前者被称为“数组缓冲区对象”或“顶点缓冲区对象”（“Array Buffer Object”或“Vertex Buff er Object”），即用来描述模型的数组，如顶点数组、纹理数组等； 后者被称为“索引缓冲区对象”（“Index Buffer Object”），是对上述数组的索引。

**光栅化（Rasterize）**
在介绍光栅化之前，首先来补充 OpenGL 中的两个非常重要的概念：

Vertex Vertex 就是图形中顶点，一系列的顶点就围成了一个图形。
Fragment Fragment 是三维空间的点、线、三角形这些基本图元映射到二维平面上的映射区域，通常一个 Fragment 对应于屏幕上的一个像素，但高分辨率的屏幕可能会用多个像素点映射到一个 Fragment，以减少 GPU 的工作。
而光栅化是把点、线、三角形映射到屏幕上的像素点的过程。

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/display.system/DS-02-07-OpenGL-guangshanhua.png)



着色器程序（Shader）
Shader 用来描述如何绘制（渲染），GLSL 是 OpenGL 的编程语言，全称 OpenGL Shader Language，它的语法类似于 C 语言。OpenGL 渲染需要两种 Shader：Vertex Shader 和 Fragment Shader。

Vertex Shader Vertex Shader 对于3D模型网格的每个顶点执行一次，主要是确定该顶点的最终位置。
Fragment Shader Fragment Shader对光栅化之后2D图像中的每个像素处理一次。3D物体的表面最终显示成什么样将由它决定，例如为模型的可见表面添加纹理，处理光照、阴影的影响等等。

#### 3.2、EGL Overview

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/display.system/DS-02-08-OpenGL-EGL-Overview.png.png)


What is the Direction?
SW : Standard API (Java, NDK Stable API)
HW : OpenGLES, OpenSLES, OpenMAX
EGL™ is an interface between Khronos rendering APIs such as OpenGL ES or OpenVG and the underlying native platform window system
#### 3.2.1、什么是 EGL？
EGL 是 OpenGL ES 渲染 API 和本地窗口系统(native platform window system)之间的一个中间接口层，它主要由系统制造商实现。

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/display.system/DS-02-09-eglCreateWindowSurface.png)

EGL提供如下机制：
与设备的原生窗口系统通信
查询绘图表面的可用类型和配置
创建绘图表面
在OpenGL ES 和其他图形渲染API之间同步渲染
管理纹理贴图等渲染资源
为了让OpenGL ES能够绘制在当前设备上，我们需要EGL作为OpenGL ES与设备的桥梁。

#### 3.2.2、使用 EGL 绘图的基本步骤

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/display.system/DS-02-10-egl-draw-surface.png.png)

☯  Display(EGLDisplay) 是对实际显示设备的抽象。
☯  Surface（EGLSurface）是对用来存储图像的内存区域
☯  FrameBuffer 的抽象，包括 Color Buffer， Stencil Buffer ，Depth Buffer。Context (EGLContext) 存储 OpenGL ES绘图的一些状态信息。
使用EGL的绘图的一般步骤：

1、获取 EGL Display 对象：eglGetDisplay()
2、初始化与 EGLDisplay 之间的连接：eglInitialize()
3、获取 EGLConfig 对象：eglChooseConfig()
4、创建 EGLContext 实例：eglCreateContext()
5、创建 EGLSurface 实例：eglCreateWindowSurface()
6、连接 EGLContext 和 EGLSurface：eglMakeCurrent()
7、使用 OpenGL ES API 绘制图形：gl_*()
8、切换 front buffer 和 back buffer 送显：eglSwapBuffer()
9、断开并释放与 EGLSurface 关联的 EGLContext 对象：eglRelease()
10、删除 EGLSurface 对象
11、删除 EGLContext 对象
12、终止与 EGLDisplay 之间的连接


#### 3.3、EGLSurface and ANativeWindow 关系



OpenGL ES 定义了一个渲染图形的 API，但没有定义窗口系统。为了让 GLES 能够适合各种平台，GLES 将与知道如何通过操作系统创建和访问窗口的库结合使用。用于 Android 的库称为 EGL。如果要绘制纹理多边形，应使用 GLES 调用；如果要在屏幕上进行渲染，应使用 EGL 调用。

在使用 GLES 进行任何操作之前，需要创建一个 GL 上下文。在 EGL 中，这意味着要创建一个 EGLContext 和一个 EGLSurface。GLES 操作适用于当前上下文，该上下文通过线程局部存储访问，而不是作为参数进行传递。这意味着您必须注意渲染代码在哪个线程上执行，以及该线程上的当前上下文。


#### 3.3.1、EGLSurface

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/display.system/DS-02-11-eglsurface-anativewindwo.png)


EGLSurface 可以是由 EGL 分配的离屏缓冲区（称为“pbuffer”），或由操作系统分配的窗口。EGL 窗口 Surface 通过 eglCreateWindowSurface() 调用被创建。该调用将“窗口对象”作为参数，在 Android 上，该对象可以是 SurfaceView、SurfaceTexture、SurfaceHolder 或 Surface，所有这些对象下面都有一个 BufferQueue。当您进行此调用时，EGL 将创建一个新的 EGLSurface 对象，并将其连接到窗口对象的 BufferQueue 的生产方接口。此后，渲染到该 EGLSurface 会导致一个缓冲区离开队列、进行渲染，然后排队等待消耗方使用。（术语“窗口”表示预期用途，但请注意，输出内容不一定会显示在显示屏上。）

EGL 不提供锁定/解锁调用，而是由您发出绘制命令，然后调用 eglSwapBuffers() 来提交当前帧。方法名称来自传统的前后缓冲区交换，但实际实现可能会有很大的不同。

一个 Surface 一次只能与一个 EGLSurface 关联（您只能将一个生产方连接到一个 BufferQueue），但是如果您销毁该 EGLSurface，它将与该 BufferQueue 断开连接，并允许其他内容连接到该 BufferQueue。

通过更改“当前”EGLSurface，指定线程可在多个 EGLSurface 之间进行切换。一个 EGLSurface 一次只能在一个线程上处于当前状态。

关于 EGLSurface 最常见的一个错误理解就是假设它只是 Surface 的另一方面（如 SurfaceHolder）。它是一个相关但独立的概念。您可以在没有 Surface 作为支持的 EGLSurface 上绘制，也可以在没有 EGL 的情况下使用 Surface。EGLSurface 仅为 GLES 提供一个绘制的地方。
#### 3.3.2、ANativeWindow
公开的 Surface 类以 Java 编程语言实现。C/C++ 中的同等项是 ANATIONWindow 类，由 Android NDK 半公开。您可以使用 ANativeWindow_fromSurface() 调用从 Surface 获取 ANativeWindow。就像它的 Java 语言同等项一样，您可以对 ANativeWindow 进行锁定、在软件中进行渲染，以及解锁并发布。

要从原生代码创建 EGL 窗口 Surface，可将 EGLNativeWindowType 的实例传递到 eglCreateWindowSurface()。EGLNativeWindowType 是 ANativeWindow 的同义词，您可以自由地在它们之间转换。

基本的“原生窗口”类型只是封装 BufferQueue 的生产方，这一点并不足为奇。

#### 3.3.3、egl_surface_t 关系图

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/display.system/DS-02-12-egl_surface_t.png)


``` cpp
[->\frameworks\native\opengl\libagl\egl.cpp]
static EGLSurface createWindowSurface(EGLDisplay dpy, EGLConfig config,
        NativeWindowType window, const EGLint* /*attrib_list*/)
{
    ......
    EGLint surfaceType;
    if (!(surfaceType & EGL_WINDOW_BIT))
        return setError(EGL_BAD_MATCH, EGL_NO_SURFACE);
        
    if (static_cast<ANativeWindow*>(window)->common.magic !=
            ANDROID_NATIVE_WINDOW_MAGIC) {
        return setError(EGL_BAD_NATIVE_WINDOW, EGL_NO_SURFACE);
    }
        
    EGLint configID;
    if (getConfigAttrib(dpy, config, EGL_CONFIG_ID, &configID) == EGL_FALSE)
        return EGL_FALSE;

    int32_t depthFormat;
    int32_t pixelFormat;
    if (getConfigFormatInfo(configID, pixelFormat, depthFormat) != NO_ERROR) {
        return setError(EGL_BAD_MATCH, EGL_NO_SURFACE);
    }

   ......
    egl_surface_t* surface;
    surface = new egl_window_surface_v2_t(dpy, config, depthFormat,
            static_cast<ANativeWindow*>(window));

    ......
    return surface;
}

```

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/display.system/DS-02-13-framebufferwindow-surface.png)


#### 3.3.4、EGLContext and Thread Local Storage

#### 3.3.4.1、EGLContext

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/display.system/DS-02-14-eglcontect-TLS.png)

#### 3.3.4.2、Thread Local Storage

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/display.system/DS-02-15-eglcontect-Thread-Loacal.png.png)



#### 3.3.5、EGLImplementation : HWCompser and SurfaceFlinger

#### 3.3.5.1、HWCompser 

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/display.system/DS-02-16-Android-graphics-components.png)


#### 3.3.5.2、SurfaceFlinger

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/display.system/DS-02-17-eglimp-hwrenderer-surfaceflinger.png)


####（四）、Android EGL：OpenGL ES 库和 EGL 库加载过程
在详细分析 EGL 绘图基本步骤 前，先来看看OpenGL ES 库和 EGL 库加载过程。
##### 4.1、OpenGL ES 和 OpenGL ES 库的区别

**OpenGL ES ：** 它本身只是一个协议规范，定义了一套可以供上层应用程序进行调用的 API，它抽象了 GPU 的功能，使应用开发者不必关心底层的 GPU 类型和具体实现。
**OpenGL ES 库：**OpenGL ES 库就是上面 OpenGL ES 中定义的 API 的具体实现。由于每个显卡制造厂商的 GPU 硬件结构不同，从而导致各个厂商的OpenGL ES 库也各不相同，所以 Android 系统中的 OpenGL ES 库通常是由硬件厂商提供的，通常存放在 Android 系统中的 /system/lib64/（/system/lib/） 。
**OpenGL ES Wrapper 库：**OpenGL ES Wrapper 库是一个对 OpenGL ES API 进行封装的一个包裹库，它向上为应用程序提供了标准的 OpenGL ES API，向下可以和不同厂商实现的 OpenGL ES 库进行绑定，将 OpenGL ES API 和对应的实现函数一一绑定在一起。
并且，OpenGL ES 库的实现分为：
**软件模拟实现**
**硬件加速实现**
现在，因为我们 Android 手机中的 Soc 片上芯片中都集成了 GPU 模块，所以这里使用的就是硬件加速实现的 OpenGL ES 库。但是，像 Android Emulator 中的 Android 系统，如果不支持将 OpenGL ES API 指令重定向到主机系统的 GPU 加速执行的话，它所采用的 OpenGL ES 库就是软件模拟实现的。

补充：如前面小节【OpenGL ES 和 EGL 介绍】中介绍的，EGL 也是一套 API，它的实现也需要系统厂商来提供。系统厂商通常会将这两套 API 的实现封装在一个共享链接库中，但是根据最新的标准，OpenGL ES API 实现的共享链接库和 EGL API 实现的共享链接库是独立分开的，例如  Nexus 9 平板设备中 OpenGL ES 和 EGL API 实现库就是独立分开的。

##### 4.2、Android 中 OpenGL ES 软件层次栈
按照分层理念的设计，Android 中的 OpenGL ES 实现也是层次设计的，形成一个软件层次栈。最上面的是 Java 层，接着下面是 JNI 层，再调用下面的 wrapper 层，wrapper 层下面则是 OpenGL ES API 的具体软件实或者硬件实现了。整个 OpenGL 软件层次栈的调用关系如下所示：

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/display.system/DS-02-18-OpenGL_ES_call_graph_so.png)


##### 4.3、OpenGL ES/EGL Wrapper 库
前面我们已经介绍过 OpenGL ES/EGL Wrapper 库是一个将 OpenGL ES API 和 OpenGL ES API 具体实现绑定在一起的库，它对应的源码路径是：/frameworks/native/opengl/libs/，其中:

``` cpp
libGLESv1_CM.so：OpenGL ES 1.x API 的 Wrapper 库
libGLESv2.so：OpenGL ES 2.0 的 Wrapper 库
libGLESv3.so：OpenGL ES 3.0 的 Wrapper 库
```

其中因为 OpenGL ES 3.0 API 是兼容 OpenGL ES 2.0 API 的，所以 libGLESv2.so 库本质上和 libGLESv3.so 库是一样的。

##### 4.3.1、OpenGL ES/EGL 实现库
如果Android系统平台支持 OpenGL ES 硬件加速渲染，那么 OpenGL ES/EGL 实现库由系统厂商以.so的共享链接库的形式提供，例如，高通的实现：system\vendor\lib\egl

``` cpp
libEGL_adreno.so 
libGLESv1_CM_adreno.so
libGLESv2_adreno.so
```

如果Android系统平台不支持 OpenGL ES 硬件加速渲染，那么它就会默认启用软件模拟渲染，这时 OpenGL ES/EGL 实现库就是由 AOSP 提供，链接库的存在的路径为： /system/lib64/egl/libGLES_android.so。而 libGLES_android.so 库在 Android 7.1 系统对应的实现源码路径为：/frameworks/native/opengl/libagl/ 。

##### 4.3.2、Android 7.1 中加载 OpenGL ES 库的过程
Android 中图形渲染所采用的方式（硬件 or 软件）是在系统启动之后动态确定的，而确定渲染方式的这个源码文件就是 /frameworks/native/opengl/libs/EGL/Loader.cpp 。

##### 4.3.2.1、 Android 7.1 OpenGL ES 库和 EGL 库加载说明
[How Android finds OpenGL libraries, and the death of egl.cfg](http://www.2net.co.uk/tutorial/android-egl-cgf-is-dead) 这篇文章中提到了非常关键的一点，就是从 Android Kitkat 4.4 之后，Android 中加载 OpenGL ES/EGL 库的方法发生了变化了（但是整个加载过程都是由 /frameworks/native/opengl/libs/EGL/Loader.cpp 程序所决定的，也就是说 Loader.cpp 文件发生了变化）。

在 Android 4.4 之前，加载 OpenGL ES 库是由 /system/lib/egl/egl.cfg 文件所决定的，通过读取这个配置文件来确定是加载 OpenGL ES 软件模拟实现的库，还是OpenGL ES 硬件加速实现的库。

但是，在Android 4.4 之后，Android 不再通过读取 egl.cfg 配置文件的方式来加载 OpenGL ES 库，新的加载 OpenGL ES 库的规则，如下所示：

从 /system/lib/egl 或者 /system/vendor/lib/egl/ 目录下加载 libGLES.so 库文件或者 libEGL_vendor.so，libGLESv1_CM_vendor.so，libGLESv2_vendor.so 库文件。
为了向下兼容旧的库的命名方式，同样也会加载 /system/lib/egl 或者 /vendor/lib/egl/ 目录下的 libGLES_*.so 或者 libEGL_*.so，libGLESv1CM*.so，libGLESv2_*.so 库文件。
##### 4.3.2.2、硬件加速渲染 or 软件模拟渲染？
前面我们提到 OpenGL ES 库的实现方式有两种，一种是硬件加速实现，一种是软件模拟实现，那么系统是怎么确定加载那一种 OpenGL ES 库的呢？

Android 7.1 源码中负责加载 OpenGL ES/EGL 库部分的代码位于：/frameworks/native/opengl/libs/EGL/Loader.cpp 文件中，这个文件中代码的主要入口函数是 Loader::open() 函数，而决定加载硬件加速渲染库还是软件模拟渲染库主要涉及到下面两个函数：

``` cpp
setEmulatorGlesValue()
checkGlesEmulationStatus()
```

下面就来简要的分析一下 Android 系统是如何选择加载硬件加速渲染库还是软件模拟渲染库：

首先，Loader::open() 入口函数会调用 setEmulatorGlesValue() 从 property 属性系统中获取一些属性值来判断当前 Android 系统是否在 Emulator 环境中运行，并根据读取出来的信息来重新设置新的属性键值对，setEmulatorGlesValue() 函数的代码如下所示：

``` cpp
[->/frameworks/native/opengl/libs/EGL/Loader.cpp]
static void setEmulatorGlesValue(void) {
     char prop[PROPERTY_VALUE_MAX];
     property_get("ro.kernel.qemu", prop, "0"); //读取 ro.kernel.qemu 属性值，判断Android系统是否运行在 qemu 中
     if (atoi(prop) != 1) return;
    
     property_get("ro.kernel.qemu.gles", prop, "0"); //读取 ro.kernel.qemu.gles 属性值，判断 qemu 中 OpenGL ES 库的实现方式
     if (atoi(prop) == 1) {
         ALOGD("Emulator has host GPU support, qemu.gles is set to 1.");
         property_set("qemu.gles", "1");
         return;
     }
    
     // for now, checking the following
     // directory is good enough for emulator system images
     const char* vendor_lib_path =
 #if defined(__LP64__)
         "/vendor/lib64/egl";
 #else
         "/vendor/lib/egl";
 #endif
    
     const bool has_vendor_lib = (access(vendor_lib_path, R_OK) == 0);
     //如果存在 vendor_lib_path 这个路径，那么就说明厂商提供了 OpenGL ES库自己的软件模拟渲染库，而不是 Android 系统自己编译得到的软件模拟渲染库
     if (has_vendor_lib) {
         ALOGD("Emulator has vendor provided software renderer, qemu.gles is set to 2.");
         property_set("qemu.gles", "2");
     } else {
         ALOGD("Emulator without GPU support detected. "
               "Fallback to legacy software renderer, qemu.gles is set to 0.");
         property_set("qemu.gles", "0"); //最后，默认采取的是方案就是调用传统的Android系统自己编译得到软件模拟渲染库
     }
 }
```
在 load_system_driver() 函数中，内部类 MatchFile 类中会调用 checkGlesEmulationStatus() 函数来检查 Android 系统是否运行在模拟器中，以及在模拟器中是否启用了主机硬件加速的功能，然后根据 checkGlesEmulationStatus() 函数的返回状态值来确定要加载共享链接库的文件绝对路径。load_system_driver() 和 checkGlesEmulationStatus() 函数代码如下所示：

``` cpp
[->/frameworks/native/opengl/libs/EGL/Loader.cpp]
static void* load_system_driver(const char* kind) {
     ATRACE_CALL();
     class MatchFile {
     public:
         //这个函数作用是返回需要加载打开的 OpenGL ES 和 EGL API 实现库文件的绝对路径
         static String8 find(const char* kind) {
             String8 result;
             int emulationStatus = checkGlesEmulationStatus(); //检查 Android 系统是否运行在模拟器中，以及在模拟器中是否启用了主机硬件加速的功能
             switch (emulationStatus) {
             case 0: //Android 运行在模拟器中，使用系统软件模拟实现的 OpenGL ES API 库 libGLES_android.so
 #if defined(__LP64__)
                 result.setTo("/system/lib64/egl/libGLES_android.so");
 #else
                 result.setTo("/system/lib/egl/libGLES_android.so");
 #endif
                 return result;
             case 1: // Android 运行在模拟器中，通过主机系统中实现 OpenGL ES 加速渲染，通过 libGLES_emulation.so 库将  OpenGL ES API 指令重定向到 host 中执行
                 // Use host-side OpenGL through the "emulation" library
 #if defined(__LP64__)
                 result.appendFormat("/system/lib64/egl/lib%s_emulation.so", kind);
 #else
                 result.appendFormat("/system/lib/egl/lib%s_emulation.so", kind);
 #endif
                 return result;
             default:
                 // Not in emulator, or use other guest-side implementation
                 break;
             }
    
             // 如果不是上面两种情况，就根据库的命名规则去找到厂商实现库文件的绝对路径
             String8 pattern;
             pattern.appendFormat("lib%s", kind);
             const char* const searchPaths[] = {
 #if defined(__LP64__)
                 "/vendor/lib64/egl",
                 "/system/lib64/egl"
 #else
                 "/vendor/lib/egl",
                 "/system/lib/egl"
 #endif
             };
                
             ......
     }
        
 }
```
总结一下上面代码的功能就是，首先判断 Android 是否在 qemu 虚拟机中运行，如果不是，那么就直接去加载厂商存放库的路径中去加载 OpenGL ES 实现库（不管是硬件加速实现的，还是软件模拟实现的）；如果是在 qemu 中运行，那么就要根据返回的 emulationStatus 值 来确定是加软件模拟实现的 OpenGL ES API 库 libGLES_android.so，还是加载 libGLES_emulation.so库将 OpenGL ES 指令重定向到 Host 系统中去执行。

##### 4.3.3、OpenGL ES/EGL 库加载和解析过程
正如前面分析，在进行 OpenGL 编程时，最先开始需要获取 Display，这将调用 eglgGetDisplay() 函数被调用。在 eglGetDisplay() 里则会调用 egl_init_drivers() 初始化驱动：装载各个库进行解析，将 OpenGL ES/EGL API 函数接口和具体的实现绑定在一起，并将结果保存在 egl_connection_t 类型的全局变量 gEGLImpl 的结构体的成员变量中。

下面以 SurfaceFlinger 进程init()为例进行分析，整个 OpenGL ES/EGL 库的加载和解析流程如下所示：

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/display.system/DS-02-19-SF_init.png)

这里通过调用 EGL 库的 eglGetDisplay() 获得 Display。

``` cpp
[->\frameworks\native\opengl\libs\EGL\eglApi.cpp]
EGLDisplay eglGetDisplay(EGLNativeDisplayType display)
{
    ......

    if (egl_init_drivers() == EGL_FALSE) {
        return setError(EGL_BAD_PARAMETER, EGL_NO_DISPLAY);
    }

    EGLDisplay dpy = egl_display_t::getFromNativeDisplay(display);
    return dpy;
}

```
函数EGLBoolean egl_init_drivers()就是负责OpenGL库的加载。
```cpp
[->\frameworks\native\opengl\libs\EGL\egl.cpp]
EGLBoolean egl_init_drivers() {
    EGLBoolean res;
    pthread_mutex_lock(&sInitDriverMutex);
    res = egl_init_drivers_locked();
    pthread_mutex_unlock(&sInitDriverMutex);
    return res;
}
```
为保证多线程访问的安全性，使用线程锁来放完另一个接口函数egl_init_drivers_locked()
```cpp
[->\frameworks\native\opengl\libs\EGL\egl.cpp]
//在该文件起始位置定义的全局变量
egl_connection_t gEGLImpl; // 描述EGL实现内容的结构体对象
gl_hooks_t gHooks[2]; // gl_hooks_t 是包含 OpenGL ES API 函数声明对应的函数指针结构体
gl_hooks_t gHooksNoContext;
pthread_key_t gGLWrapperKey = -1;

static EGLBoolean egl_init_drivers_locked() {
    if (sEarlyInitState) {
        // initialized by static ctor. should be set here.
        return EGL_FALSE;
    }

    // 得到 Loader 对象单例
    // get our driver loader
    Loader& loader(Loader::getInstance());

    //  gEGLImple 是一个全局变量，数据类型为 egl_connection_t 结构体类型
    // dynamically load our EGL implementation
    egl_connection_t* cnx = &gEGLImpl;

    // cnx->dso 本质上是一个 (void *)类型的指针，它指向的对象是 EGL 共享库打开之后的句柄
    if (cnx->dso == 0) { 
        // >= 将cnx中的 hooks 数组中指向OpenGL ES API 函数指针结构体指的数组成员，用 gHooks 中的成员的地址去初始化
        //也就是说 gEGLImpl 中 hook 数组指向 gHooks 数组，最终指向同一个 OpenGL ES API 函数指针的实现
        cnx->hooks[egl_connection_t::GLESv1_INDEX] =
            &gHooks[egl_connection_t::GLESv1_INDEX];
        cnx->hooks[egl_connection_t::GLESv2_INDEX] =
            &gHooks[egl_connection_t::GLESv2_INDEX];

        // >= 最后通过loader对象的open函数开始加载 OpenGL ES 和 EGL wrapper 库
        cnx->dso = loader.open(cnx);
    }

    return cnx->dso ? EGL_TRUE : EGL_FALSE;
}
```
在这个函数中，有一个非常关键的 egl_connection_t 指针指向一个全局变量 gEGLImpl，当第一次初始化加载 OpenGL ES 实现库和 EGL 实现库时，还需要将 gEGLImpl 中的 hooks 数组中的两个指针指向一个全局的 gl_hooks_t 数组 gHooks（这就是两个指针钩子，最终初始化完成后将分别勾住 OpenGL ES 1.0 和 OpenGL ES 2.0 的实现库），接着调用 Loader 类的实例的 open() 函数完成从 OpenGL ES 实现库中完成符号解析工作。

Loader::open() 函数的代码如下所示：

``` cpp
[/frameworks/native/opengl/libs/EGL/Loader.cpp]
// >= Loader 类对象构造完成后，就在 /EGL/egl.cpp 文件中的 egl_init_drivers_locked() 中被调用
void* Loader::open(egl_connection_t* cnx)
{
    void* dso;
    driver_t* hnd = 0;
    setEmulatorGlesValue();
    dso = load_driver("GLES", cnx, EGL | GLESv1_CM | GLESv2);
    if (dso) {
        hnd = new driver_t(dso);
    } else {
        // Always load EGL first
        dso = load_driver("EGL", cnx, EGL);
        if (dso) {
            hnd = new driver_t(dso);
            hnd->set( load_driver("GLESv1_CM", cnx, GLESv1_CM), GLESv1_CM );
            hnd->set( load_driver("GLESv2",    cnx, GLESv2),    GLESv2 );
        }
    }
    ......
    cnx->libEgl   = load_wrapper(EGL_WRAPPER_DIR "/libEGL.so");
    cnx->libGles2 = load_wrapper(EGL_WRAPPER_DIR "/libGLESv2.so");
    cnx->libGles1 = load_wrapper(EGL_WRAPPER_DIR "/libGLESv1_CM.so");
    ......
    return (void*)hnd;
}
```
open() 函数主要负责 OpenGL ES 库加载前的准备工作，具体的加载细节，则是通过调用 load_driver() 去完成的。
``` cpp
[/frameworks/native/opengl/libs/EGL/Loader.cpp]
oid *Loader::load_driver(const char* kind,
                          egl_connection_t* cnx, uint32_t mask)
{
    void* dso = nullptr;
    if (mGetDriverNamespace) {
        android_namespace_t* ns = mGetDriverNamespace();
        if (ns) {
            dso = load_updated_driver(kind, ns); //加载 OpenGL ES 实现库，放回打开的共享链接库的句柄
        }
    }
    if (!dso) {
        dso = load_system_driver(kind);
        ......
    }

    // 解析 EGL 库，并将wrapper 库 libEGL.so 中的函数 API 指针和具体的实现绑定在一起
    if (mask & EGL) {
        getProcAddress = (getProcAddressType)dlsym(dso, "eglGetProcAddress");
        ......
        egl_t* egl = &cnx->egl; //将 egl 指针指向描述当前系统支持 OpenGL ES和 EGL 全局变量的 gEGLImpl
        __eglMustCastToProperFunctionPointerType* curr =
            (__eglMustCastToProperFunctionPointerType*)egl;
        char const * const * api = egl_names; //egl_names 是定义在 egl.cpp 文件中的一个数组，数组中的元素是 EGL API 函数指针
        while (*api) {
            char const * name = *api;
            __eglMustCastToProperFunctionPointerType f =
                (__eglMustCastToProperFunctionPointerType)dlsym(dso, name);
            if (f == NULL) {
                // couldn't find the entry-point, use eglGetProcAddress()
                f = getProcAddress(name);
                if (f == NULL) {
                    f = (__eglMustCastToProperFunctionPointerType)0;
                }
            }
            *curr++ = f; //这一步就是最关键的将共享链接库中的 EGL API 的实现和上层调用的 API 函数指针绑定在一起
            api++; //指向下一个需要绑定的 api 函数
        }
    }

    // 解析 OpenGL ES 库中的 OpenGL ES 1.x API 符号
    if (mask & GLESv1_CM) {
        // 调用 init_api 实现 OpenGL API 和对应实现函数的绑定
        init_api(dso, gl_names, // gl_names 是定义在 egl.cpp 文件中的一个数组，数组中的元素是 OpenGL ES API 函数指针
                 (__eglMustCastToProperFunctionPointerType*)
                 &cnx->hooks[egl_connection_t::GLESv1_INDEX]->gl, //gl成员变量是一个结构体变量，结构体中的是 OpenGL ES API 函数指针
                 getProcAddress);
    }

    // 解析 OpenGL ES 库中的 OpenGL ES 2.0 API 符号
    if (mask & GLESv2) {
        init_api(dso, gl_names,
                 (__eglMustCastToProperFunctionPointerType*)
                 &cnx->hooks[egl_connection_t::GLESv2_INDEX]->gl,
                 getProcAddress);
    }

    return dso;
}
```
Loader::load_driver() 它主要实现了两个功能：

通过 load_system_driver()  函数查找 OpenGL ES/EGL 实现库，并在指定的存放路径中找到共享链接库文件并打开它。
调用 init_api()解析打开的 OpenGL ES/EGL 共享链接库，将 OpenGL ES/EGL API 函数指针和共享链接库中实现的对应的函数符号绑定在一起，这样调用 OpenGL ES/EGL API 就会调用到具体实现的OpenGL ES/EGL 共享链接库中对应函数。
##### 4.4、小结
Android OpenGL ES 图形库结构
Android 的 OpenGL ES 图形系统涉及多个库，根据设备类型的不同，这些库有着不同的结构。

对于模拟器，没有开启 OpenGL ES 的 GPU 硬件模拟的情况，Android OpenGL ES 图形库结构如下：

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/display.system/DS-02-20-opencl-egl-imp.png)



当为模拟器开启了 OpenGL ES 的 GPU 硬件模拟，实际的 EGL 和 OpenGL ES 实现库会采用由 android-7.1.1_r22/device/generic/goldfish-opengl 下的源码编译出来的几个库文件，即 libGLESv2_emulation.so、libGLESv1_CM_emulation.so 和 libEGL_emulation.so。此时，OpenGL ES 图形库结构如下：

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/display.system/DS-02-21-opencl-egl-imp-emulate.png)


对于真实的物理 Android 设备，OpenGL ES 图形库结构如下，例如高通实现（libEGL_adreno.so 
libGLESv1_CM_adreno.so libGLESv2_adreno.so [\system\vendor\lib64\egl]）：

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/display.system/DS-02-22-opencl-egl-imp-qcom-adreno.png)

####（五）、OpenGL ES：EGL接口解析与理解

由前面的分析知道EGL的绘图的一般步骤如下，接下来分析主要的1-8个小步骤：

> 使用EGL的绘图的一般步骤：
> 1、获取 EGL Display 对象：eglGetDisplay() 
> 2、初始化与 EGLDisplay 之间的连接：eglInitialize() 
> 3、获取 EGLConfig 对象：eglChooseConfig() 
> 4、创建 EGLContext 实例：eglCreateContext() 
> 5、创建 EGLSurface 实例：eglCreateWindowSurface() 
> 6、连接 EGLContext 和 EGLSurface：eglMakeCurrent() 
> 7、使用 OpenGL ES API 绘制图形：gl_*() 
> 8、切换 front buffer 和 back buffer 送显：eglSwapBuffer() 
> 9、断开并释放与 EGLSurface 关联的 EGLContext 对象：eglRelease() 
> 10、删除 EGLSurface 对象 
> 11、删除 EGLContext 对象 
> 12、终止与 EGLDisplay 之间的连接


标准 EGL 数据类型如下所示：

EGLBoolean ——EGL_TRUE =1, EGL_FALSE=0
EGLint ——int 数据类型
EGLDisplay ——系统显示 ID 或句柄，可以理解为一个前端的显示窗口
EGLConfig ——Surface的EGL配置，可以理解为绘制目标framebuffer的配置属性
EGLSurface ——系统窗口或 frame buffer 句柄 ，可以理解为一个后端的渲染目标窗口。
EGLContext ——OpenGL ES 图形上下文，它代表了OpenGL状态机；如果没有它，OpenGL指令就没有执行的环境。

下面几个类型比较复杂，通过例子可以更深入的理解。这里要说明的是这几个类型在不同平台其实现是不同的，EGL只提供抽象标准。

NativeDisplayType——Native 系统显示类型，标识你所开发设备的物理屏幕
NativeWindowType ——Native 系统窗口缓存类型，标识系统窗口
NativePixmapType ——Native 系统 frame buffer，可以作为 Framebuffer 的系统图像（内存）数据类型，该类型只用于离屏渲染.

##### 5.1、eglGetDisplay() 
EGLDisplay 是一个关联系统物理屏幕的通用数据类型，表示显示设备句柄，也可以认为是一个前端显示窗。为了使用系统的显示设备， EGL 提供了 EGLDisplay 数据类型，以及一组操作设备显示的 API 。
下面的函数原型用于获取 Native Display ：
```cpp
[->\frameworks\native\opengl\libagl\egl.cpp]
EGLDisplay eglGetDisplay(NativeDisplayType display)
{
......
    if (display == EGL_DEFAULT_DISPLAY) {
        EGLDisplay dpy = (EGLDisplay)1;
        egl_display_t& d = egl_display_t::get_display(dpy);
        d.type = display;
        return dpy;
    }
    return EGL_NO_DISPLAY;
}

egl_display_t& egl_display_t::get_display(EGLDisplay dpy) {
    return gDisplays[uintptr_t(dpy)-1U];
}
```
其 中 display 参数是 native 系统的窗口显示 ID 值。如果你只是想得到一个系统默认的 Display ，你可以使用 EGL_DEFAULT_DISPLAY 参数。如果系统中没有一个可用的 native display ID 与给定的 display 参数匹配，函数将返回 EGL_NO_DISPLAY ，而没有任何 Error 状态被设置。

##### 5.2、eglInitialize() 
每个 EGLDisplay 在使用前都需要初始化。初始化 EGLDisplay 的同时，你可以得到系统中 EGL 的实现版本号。了解当前的版本号在向后兼容性方面是非常有价值的。在移动设备上，通过动态查询 EGL 版本号，你可以为新旧版本的 EGL 附加额外的特性或运行环境。基于平台配置，软件开发可用清楚知道哪些 API 可用访问，这将会为你的代码提供最大限度的可移植性。
``` cpp
[->\frameworks\native\opengl\libagl\egl.cpp]
EGLBoolean eglInitialize(EGLDisplay dpy, EGLint *major, EGLint *minor)
{
    if (egl_display_t::is_valid(dpy) == EGL_FALSE)
        return setError(EGL_BAD_DISPLAY, EGL_FALSE);

    EGLBoolean res = EGL_TRUE;
    egl_display_t& d = egl_display_t::get_display(dpy);

    if (d.initialized.fetch_add(1, std::memory_order_acquire) == 0) {
        ......
    }

    if (res == EGL_TRUE) {
        if (major != NULL) *major = VERSION_MAJOR;
        if (minor != NULL) *minor = VERSION_MINOR;
    }
    return res;
}
```

其中 dpy 应该是一个有效的 EGLDisplay 。函数返回时， major 和 minor 将被赋予当前 EGL 版本号。比如 EGL1.0 ， major 返回 1 ， minor 则返回 0 。给 major 和 minor 传 NULL 是有效的，如果你不关心版本号。
eglQueryString() 函数是另外一个获取版本信息和其他信息的途径。通过 eglQueryString() 获取版本信息需要解析版本字符串，所以通过传递一个指针给 eglInitializ() 函数比较容易获得这个信息。注意在调用 eglQueryString() 必须先使用 eglInitialize() 初始化 EGLDisplay ，否则将得到 EGL_NOT_INITIALIZED 错误信息。

##### 5.3、eglChooseConfig() 
基 于 EGL 的属性，可以得到一个和需求接近的Config，但也可以选择自己需要的Config，只要平台支持。不是所有的Config都是有效的，也就是不是所有Config都会支持。 eglChooseConfig() 函数将适配一个所期望的配置，并且尽可能接近一个有效的系统配置。

``` cpp
[->\frameworks\native\opengl\libagl\egl.cpp]
EGLBoolean eglChooseConfig( EGLDisplay dpy, const EGLint *attrib_list,
                            EGLConfig *configs, EGLint config_size,
                            EGLint *num_config)
{
    if (egl_display_t::is_valid(dpy) == EGL_FALSE)
        return setError(EGL_BAD_DISPLAY, EGL_FALSE);
    
    if (ggl_unlikely(num_config==0)) {
        return setError(EGL_BAD_PARAMETER, EGL_FALSE);
    }

    if (ggl_unlikely(attrib_list==0)) {
        /*
         * A NULL attrib_list should be treated as though it was an empty
         * one (terminated with EGL_NONE) as defined in
         * section 3.4.1 "Querying Configurations" in the EGL specification.
         */
        static const EGLint dummy = EGL_NONE;
        attrib_list = &dummy;
    }

    int numAttributes = 0;
    int numConfigs =  NELEM(gConfigs);
    uint32_t possibleMatch = (1<<numConfigs)-1;
    while(possibleMatch && *attrib_list != EGL_NONE) {
        numAttributes++;
        EGLint attr = *attrib_list++;
        EGLint val  = *attrib_list++;
        for (int i=0 ; possibleMatch && i<numConfigs ; i++) {
            if (!(possibleMatch & (1<<i)))
                continue;
            if (isAttributeMatching(i, attr, val) == 0) {
                possibleMatch &= ~(1<<i);
            }
        }
    }

    // now, handle the attributes which have a useful default value
    for (size_t j=0 ; possibleMatch && j<NELEM(config_defaults) ; j++) {
        // see if this attribute was specified, if not, apply its
        // default value
        if (binarySearch<config_pair_t>(
                (config_pair_t const*)attrib_list,
                0, numAttributes-1,
                config_defaults[j].key) < 0)
        {
            for (int i=0 ; possibleMatch && i<numConfigs ; i++) {
                if (!(possibleMatch & (1<<i)))
                    continue;
                if (isAttributeMatching(i,
                        config_defaults[j].key,
                        config_defaults[j].value) == 0)
                {
                    possibleMatch &= ~(1<<i);
                }
            }
        }
    }

    // return the configurations found
    int n=0;
    if (possibleMatch) {
        if (configs) {
            for (int i=0 ; config_size && i<numConfigs ; i++) {
                if (possibleMatch & (1<<i)) {
                    *configs++ = (EGLConfig)(uintptr_t)i;
                    config_size--;
                    n++;
                }
            }
        } else {
            for (int i=0 ; i<numConfigs ; i++) {
                if (possibleMatch & (1<<i)) {
                    n++;
                }
            }
        }
    }
    *num_config = n;
     return EGL_TRUE;
}
```
参数 attrib_list 指定了选择配置时需要参照的属性。参数 configs 将返回一个按照 attrib_list 排序的平台有效的所有 EGL framebuffer 配置列表。参数 config_size 指定了可以返回到 configs 的总配置个数。参数 num_config 返回了实际匹配的配置总数。

##### 5.4、eglCreateContext() 
OpenGL ES的pipeline从程序的角度看就是一个状态机，有当前的颜色、纹理坐标、变换矩阵、绚染模式等一大堆状态，这些状态作用于OpenGL API程序提交的顶点坐标等图元从而形成帧缓冲内的像素。在OpenGL的编程接口中，Context就代表这个状态机，OpenGL API程序的主要工作就是向Context提供图元、设置状态，偶尔也从Context里获取一些信息。
``` cpp
[->\frameworks\native\opengl\libagl\egl.cpp]
EGLContext eglCreateContext(EGLDisplay dpy, EGLConfig config,
                            EGLContext /*share_list*/, const EGLint* /*attrib_list*/)
{
    if (egl_display_t::is_valid(dpy) == EGL_FALSE)
        return setError(EGL_BAD_DISPLAY, EGL_NO_SURFACE);

    ogles_context_t* gl = ogles_init(sizeof(egl_context_t));
    if (!gl) return setError(EGL_BAD_ALLOC, EGL_NO_CONTEXT);

    egl_context_t* c = static_cast<egl_context_t*>(gl->rasterizer.base);
    c->flags = egl_context_t::NEVER_CURRENT;
    c->dpy = dpy;
    c->config = config;
    c->read = 0;
    c->draw = 0;
    return (EGLContext)gl;
}
```

##### 5.5、eglCreateWindowSurface()
Surface实际上就是一个FrameBuffer，也就是渲染目的地，

``` cpp
[->\frameworks\native\opengl\libagl\egl.cpp]
EGLSurface eglCreateWindowSurface(  EGLDisplay dpy, EGLConfig config,
                                    NativeWindowType window,
                                    const EGLint *attrib_list)
{
    return createWindowSurface(dpy, config, window, attrib_list);
}

static EGLSurface createWindowSurface(EGLDisplay dpy, EGLConfig config,
        NativeWindowType window, const EGLint* /*attrib_list*/)
{
    if (egl_display_t::is_valid(dpy) == EGL_FALSE)
        return setError(EGL_BAD_DISPLAY, EGL_NO_SURFACE);
    if (window == 0)
        return setError(EGL_BAD_MATCH, EGL_NO_SURFACE);

    EGLint surfaceType;
    if (getConfigAttrib(dpy, config, EGL_SURFACE_TYPE, &surfaceType) == EGL_FALSE)
        return EGL_FALSE;

    if (!(surfaceType & EGL_WINDOW_BIT))
        return setError(EGL_BAD_MATCH, EGL_NO_SURFACE);

    if (static_cast<ANativeWindow*>(window)->common.magic !=
            ANDROID_NATIVE_WINDOW_MAGIC) {
        return setError(EGL_BAD_NATIVE_WINDOW, EGL_NO_SURFACE);
    }
        
    EGLint configID;
    if (getConfigAttrib(dpy, config, EGL_CONFIG_ID, &configID) == EGL_FALSE)
        return EGL_FALSE;

    int32_t depthFormat;
    int32_t pixelFormat;
    if (getConfigFormatInfo(configID, pixelFormat, depthFormat) != NO_ERROR) {
        return setError(EGL_BAD_MATCH, EGL_NO_SURFACE);
    }

    ......

    egl_surface_t* surface;
    surface = new egl_window_surface_v2_t(dpy, config, depthFormat,
            static_cast<ANativeWindow*>(window));

    if (!surface->initCheck()) {
        delete surface;
        surface = 0;
    }
    return surface;
}

```
来创建一个可实际显示的Surface。

系统通常还支持另外两种Surface：PixmapSurface和PBufferSurface，这两种都不是可显示的Surface，PixmapSurface是保存在系统内存中的位图，PBuffer则是保存在显存中的帧。

对于这两种surface，Android系统中，支持PBufferSurface。

##### 5.6、eglMakeCurrent()
该接口将申请到的display，draw（surface）和 context进行了绑定。也就是说，在context下的OpenGLAPI指令将draw（surface）作为其渲染最终目的地。而display作为draw（surface）的前端显示。调用后，当前线程使用的EGLContex为context。
``` cpp
[->\frameworks\native\opengl\libagl\egl.cpp]
EGLBoolean eglMakeCurrent(  EGLDisplay dpy, EGLSurface draw,
                            EGLSurface read, EGLContext ctx)
{
    if (egl_display_t::is_valid(dpy) == EGL_FALSE)
        return setError(EGL_BAD_DISPLAY, EGL_FALSE);
    if (draw) {
        egl_surface_t* s = (egl_surface_t*)draw;
        if (!s->isValid())
            return setError(EGL_BAD_SURFACE, EGL_FALSE);
        if (s->dpy != dpy)
            return setError(EGL_BAD_DISPLAY, EGL_FALSE);
        // TODO: check that draw is compatible with the context
    }
    if (read && read!=draw) {
        egl_surface_t* s = (egl_surface_t*)read;
        if (!s->isValid())
            return setError(EGL_BAD_SURFACE, EGL_FALSE);
        if (s->dpy != dpy)
            return setError(EGL_BAD_DISPLAY, EGL_FALSE);
        // TODO: check that read is compatible with the context
    }

    EGLContext current_ctx = EGL_NO_CONTEXT;

    if ((read == EGL_NO_SURFACE && draw == EGL_NO_SURFACE) && (ctx != EGL_NO_CONTEXT))
        return setError(EGL_BAD_MATCH, EGL_FALSE);

    if ((read != EGL_NO_SURFACE || draw != EGL_NO_SURFACE) && (ctx == EGL_NO_CONTEXT))
        return setError(EGL_BAD_MATCH, EGL_FALSE);

    if (ctx == EGL_NO_CONTEXT) {
        // if we're detaching, we need the current context
        current_ctx = (EGLContext)getGlThreadSpecific();
    } else {
        egl_context_t* c = egl_context_t::context(ctx);
        egl_surface_t* d = (egl_surface_t*)draw;
        egl_surface_t* r = (egl_surface_t*)read;
        if ((d && d->ctx && d->ctx != ctx) ||
            (r && r->ctx && r->ctx != ctx)) {
            // one of the surface is bound to a context in another thread
            return setError(EGL_BAD_ACCESS, EGL_FALSE);
        }
    }

    ogles_context_t* gl = (ogles_context_t*)ctx;
    if (makeCurrent(gl) == 0) {
        if (ctx) {
            egl_context_t* c = egl_context_t::context(ctx);
            egl_surface_t* d = (egl_surface_t*)draw;
            egl_surface_t* r = (egl_surface_t*)read;
            
            if (c->draw) {
                egl_surface_t* s = reinterpret_cast<egl_surface_t*>(c->draw);
                s->disconnect();
                s->ctx = EGL_NO_CONTEXT;
                if (s->zombie)
                    delete s;
            }
            if (c->read) {
                // FIXME: unlock/disconnect the read surface too 
            }
            
            c->draw = draw;
            c->read = read;

            if (c->flags & egl_context_t::NEVER_CURRENT) {
                c->flags &= ~egl_context_t::NEVER_CURRENT;
                GLint w = 0;
                GLint h = 0;
                if (draw) {
                    w = d->getWidth();
                    h = d->getHeight();
                }
                ogles_surfaceport(gl, 0, 0);
                ogles_viewport(gl, 0, 0, w, h);
                ogles_scissor(gl, 0, 0, w, h);
            }
            if (d) {
                if (d->connect() == EGL_FALSE) {
                    return EGL_FALSE;
                }
                d->ctx = ctx;
                d->bindDrawSurface(gl);
            }
            if (r) {
                // FIXME: lock/connect the read surface too 
                r->ctx = ctx;
                r->bindReadSurface(gl);
            }
        } else {
            // if surfaces were bound to the context bound to this thread
            // mark then as unbound.
            if (current_ctx) {
                egl_context_t* c = egl_context_t::context(current_ctx);
                egl_surface_t* d = (egl_surface_t*)c->draw;
                egl_surface_t* r = (egl_surface_t*)c->read;
                if (d) {
                    c->draw = 0;
                    d->disconnect();
                    d->ctx = EGL_NO_CONTEXT;
                    if (d->zombie)
                        delete d;
                }
                if (r) {
                    c->read = 0;
                    r->ctx = EGL_NO_CONTEXT;
                    // FIXME: unlock/disconnect the read surface too 
                }
            }
        }
        return EGL_TRUE;
    }
    return setError(EGL_BAD_ACCESS, EGL_FALSE);
}

```

##### 5.7、绘制gl_*() 
应用程序通过OpenGL API进行绘制，一帧完成之后，调用eglSwapBuffers(EGLDisplay dpy, EGLContext ctx)来显示。

##### 5.8、eglSwapBuffers接口实现说明
Android平台：

为了实现eglSwapBuffers， eglSurface其实代表了一个从NativeWindow 申请到的一个Buffer（Dequeue操作）。当调用eglSwapBuffers时，对于一般应用窗口而言，NativeWindow将该Surface的Buffer 提交回去给SurfaceFlinger（Queue操作)，

``` cpp
[->\frameworks\native\opengl\libagl\egl.cpp]
EGLBoolean egl_window_surface_v2_t::swapBuffers()
{
......
nativeWindow->queueBuffer(nativeWindow, buffer); 
nativeWindow->dequeueBuffer(nativeWindow, &buffer);
......
}
```

然后又重新从NativeWindow中重新Dequeue出来一个新的Buffer给eglSurface。而eglDisplay并不代表实际的意义。我们只是从接口上感觉是，surface和display进行了交换。（注：现在是Triple Buffer）


> **总结：从前面关于Android EGL、OpenGL ES的分析知道，现在我们可以通过SurfaceFlinger申请一块Surface（Buffer），然后可以利用OpenGL ES接口在Native 层绘制相关的图片、文字；那么疑问来了，Android上层绚丽多彩的App界面是如何绘制而成的呢、App层如何通过底层的OpenGL ES接口来完成绘制呢？？？**

#### （六）、参考资料(特别感谢各位前辈的分析和图示)：
[Khronos Group](https://www.khronos.org/)
[Android图形架构 官方文档](https://source.android.com/devices/graphics/architecture)
[OPENGL ES 2.0 知识串讲](http://geekfaner.com/shineengine/index.html)
[OpenGL ES EGL Spec&APIs](https://www.slideshare.net/namjungsoo/egl-31239467)
[Understaing-Android-Egl](https://www.slideshare.net/SuhanLee2/understaing-android-egl)
[Android 系统图形栈(1) &&(2)： OpenGL ES 和 EGL](https://woshijpf.github.io/category/android/)
[Android L 的开机动画流程 - CSDN博客](https://blog.csdn.net/hovan/article/details/43198399)



