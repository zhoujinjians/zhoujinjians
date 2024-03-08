---
title: Android 8.1 Display System源码分析（1）：Display System 精彩世界（RK3399）
cover: https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/personal.website/post.cover.pictures.00010.jpg
categories: 
  - Display
tags:
  - Android
  - Linux
  - Graphics
toc: true
abbrlink: 20200508
date: 2020-05-08 09:25:00
---


注：文章都是通过阅读 Android  && Linux 平台源码、各位前辈总结的资料、加上自己的思考分析总结出来的，其中难免有理不对的地方，欢迎大家批评指正。**文章为个人学习、研究、欣赏之用。图文内容、源码整理自互联网，如有侵权，请联系删除（◕‿◕）**，转载请注明出处（ ©Android @Linux @Rockchip版权所有），谢谢(๑乛◡乛๑) 、（ ͡° ͜ʖ ͡°）、（ಡωಡ）！！！。
（==**文章基于 Kernel-4.4**==）&&（==**文章基于 Android-8.1**==）
[【开发板 - Firefly-RK3399 7.85寸1024x768显示屏模组（MIPI接口）】](
http://wiki.t-firefly.com/zh_CN/Firefly-RK3399/compile_android8.1_firmware.html#)

正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢！！！

[（1）【modetest使用指南】](
https://markyzq.gitbooks.io/rockchip_drm_integration_helper/content/zh/drm_modetest.html) 

[（2）【Ubuntu12.04 编译Android 7.1 mkimage.sh 失败】](https://blog.csdn.net/ansondroider/article/details/85520148) 

--------------------------------------------------------------------------------
==源码（部分）==：

>AMS、WMS、SurfaceFlinger Test App（Android Apk Java层）

- I:\RK3399_Android8.1_MIPI\packages\TestViewportRed\
- I:\RK3399_Android8.1_MIPI\packages\TestViewportGreen\
- I:\RK3399_Android8.1_MIPI\packages\TestViewportBlue\

> SurfaceFlinger Test App（Android Native层）

- I:\RK3399_Android8.1_MIPI\frameworks\native\services\surfaceflinger\tests\SurfaceFlingerTestsRed\
- I:\RK3399_Android8.1_MIPI\frameworks\native\services\surfaceflinger\tests\SurfaceFlingerTestsGreen\
- I:\RK3399_Android8.1_MIPI\frameworks\native\services\surfaceflinger\tests\SurfaceFlingerTestsBlue\

>OpenGLES Test App（Android OpenGLES Native层）

- I:\RK3399_Android8.1_MIPI\frameworks\native\opengl\tests\OpenGLESTexturesRGB\


>ModeTest App （Kernel 用户层）

- I:\RK3399_Android8.1_MIPI\external\libdrm\tests\modetest\

--------------------------------------------------------------------------------
#### 前言

**Ubuntu12.04 编译Android 8.1 mkimage.sh 失败:**
在编译Android8.1过程中, 执行make 完成后, 开始打包img文件时调用:
./mkimage.sh 出现以下错误
>
>in=/work/Android8.1RK3399/out/target/product/rk3399_box/system.img out=/work/Android8.1RK3399/out/target/product/rk3399_box/system.img.out align=1024
>Total of 292968 4096-byte output blocks in 4834 input chunks.
>Exception:unpack requires a string argument of length 12
>Traceback (most recent call last):
>  File "device/rockchip/common/sparse_tool.py", line 183, in <module>
>    main()
>  File "device/rockchip/common/sparse_tool.py", line 175, in main
>    print("i=%u" % (i))
>UnboundLocalError: local variable 'i' referenced before assignment
>————————————————

**解决方案：**

```python
book@book-virtual-machine:/work/Android8.1RK3399/device/rockchip/common$ git diff
diff --git a/device/rockchip/common/sparse_tool.py b/device/rockchip/common/sparse_tool.py
index 47df687..efb1afa 100755
--- a/device/rockchip/common/sparse_tool.py
+++ b/device/rockchip/common/sparse_tool.py
@@ -46,7 +46,7 @@ def main():
                   % (total_blks, blk_sz, total_chunks))
             out_total_chunks = 0
             out_head = bytearray(28)
-            out_chunk = bytearray(12)
+            out_chunk = struct.pack("<2H2I", 0,0,0,0)
             chunk_struct = list(struct.unpack("<2H2I", out_chunk))
             buffer = bytearray(align_unit * 1024)
             offset = 0

```

#### (一)、ModeTest App （Kernel 用户层）

#### （1）、ModeTest App显示图像
##### 1.1、源代码
**modetest.c**

``` cpp
I:\RK3399_Android8.1_MIPI\external\libdrm\tests\modetest\modetest.c

int main(int argc, char **argv)
{
	struct device dev;

	int c;
	int encoders = 0, connectors = 0, crtcs = 0, planes = 0, framebuffers = 0;
	int drop_master = 0;
	int test_vsync = 0;
	int test_cursor = 0;
	char *device = NULL;
	char *module = NULL;
	unsigned int i;
	unsigned int count = 0, plane_count = 0;
	unsigned int prop_count = 0;
	struct pipe_arg *pipe_args = NULL;
	struct plane_arg *plane_args = NULL;
	struct property_arg *prop_args = NULL;
	unsigned int args = 0;
	int ret;

	memset(&dev, 0, sizeof dev);

	......

	if (!args)
		encoders = connectors = crtcs = planes = framebuffers = 1;

	dev.fd = util_open(device, module);
	if (dev.fd < 0)
		return -1;

	if (test_vsync && !page_flipping_supported()) {
		fprintf(stderr, "page flipping not supported by drm.\n");
		return -1;
	}

	if (test_vsync && !count) {
		fprintf(stderr, "page flipping requires at least one -s option.\n");
		return -1;
	}

	if (test_cursor && !cursor_supported()) {
		fprintf(stderr, "hw cursor not supported by drm.\n");
		return -1;
	}

	dev.resources = get_resources(&dev);
	if (!dev.resources) {
		drmClose(dev.fd);
		return 1;
	}

	for (i = 0; i < count; i++) {
		if (pipe_resolve_connectors(&dev, &pipe_args[i]) < 0) {
			free_resources(dev.resources);
			drmClose(dev.fd);
			return 1;
		}
	}

#define dump_resource(dev, res) if (res) dump_##res(dev)

	dump_resource(&dev, encoders);
	dump_resource(&dev, connectors);
	dump_resource(&dev, crtcs);
	dump_resource(&dev, planes);
	dump_resource(&dev, framebuffers);

	for (i = 0; i < prop_count; ++i)
		set_property(&dev, &prop_args[i]);

	if (count || plane_count) {
		uint64_t cap = 0;

		ret = drmGetCap(dev.fd, DRM_CAP_DUMB_BUFFER, &cap);
		if (ret || cap == 0) {
			fprintf(stderr, "driver doesn't support the dumb buffer API\n");
			return 1;
		}

		if (count)
			set_mode(&dev, pipe_args, count);

		if (plane_count)
			set_planes(&dev, plane_args, plane_count);

		if (test_cursor)
			set_cursors(&dev, pipe_args, count);

		if (test_vsync)
			test_page_flip(&dev, pipe_args, count);

		if (drop_master)
			drmDropMaster(dev.fd);

		getchar();

		if (test_cursor)
			clear_cursors(&dev);

		if (plane_count)
			clear_planes(&dev, plane_args, plane_count);

		if (count)
			clear_mode(&dev);
	}

	free_resources(dev.resources);

	return 0;
}



```


**Android.mk**



>LOCAL_PATH := $(call my-dir)
>
>include $(CLEAR_VARS)
>include $(LOCAL_PATH)/Makefile.sources
>
>LOCAL_SRC_FILES := $(filter-out %.h,$(MODETEST_FILES))
>
>LOCAL_MODULE := modetest
>
>LOCAL_SHARED_LIBRARIES := libdrm_platform
>LOCAL_STATIC_LIBRARIES := libdrm_util
>
>LOCAL_C_INCLUDES := $(LOCAL_PATH)/..
>
>include $(LIBDRM_COMMON_MK)
>include $(BUILD_EXECUTABLE)



##### 1.2、modetest使用指南


modetest是libdrm源码自带的调试工具, 可以对drm进行一些基础的调试.
**获取:android平台:**
mmm external/libdrm/tests

**modetest帮助信息:**

>(shell)# modetest -h
>usage: modetest [-cDdefMPpsCvw]
> Query options:
>    -c    list connectors
>    -e    list encoders
>    -f    list framebuffers
>    -p    list CRTCs and planes (pipes)
> Test options:
>    -P <crtc_id>:<w>x<h>[+<x>+<y>][*<scale>][@<format>]    set a plane
>    -s <connector_id>[,<connector_id>][@<crtc_id>]:<mode>[-<vrefresh>][@<format>]    set a mode
>    -C    test hw cursor
>    -v    test vsynced page flipping
>    -w <obj_id>:<prop_name>:<value>    set property
> Generic options:
>    -d    drop master after mode set
>    -M module    use the given driver
>    -D device    use the given device
>    Default is to dump all info.


**使用案例:modetest不带参(有非常多的打印,这边截取部分关键的):**
>
>//由于rockchip driver的一些配置未upstream到libdrm上, 所以从libdrm upstream
>//下载编译的modetest默认不带rockchip支持, 需要在使用的时候加个-M rockchip.
>(shell)# modetest -M rockchip
>Encoders:
>id    crtc    type    possible crtcs    possible clones    
>71    27    TMDS    0x00000001    0x00000000
>73    0    TMDS    0x00000002    0x00000000
>Connectors:
>id    encoder    status        name        size (mm)    modes    encoders
>72    71    connected    HDMI-A-1           410x260        19    71
>  modes:
>    name refresh (Hz) hdisp hss hse htot vdisp vss vse vtot)
>  1440x900 60 1440 1520 1672 1904 900 903 909 934 flags: nhsync, pvsync; type: preferred, driver
>  1280x1024 75 1280 1296 1440 1688 1024 1025 1028 1066 flags: phsync, pvsync; type: driver
>  [...]
>74    0    disconnected    DP-1               0x0        0    73
>CRTCs:
>id    fb    pos    size
>27    0    (0,0)    (1280x1024)
>64    0    (0,0)    (0x0)
>Planes:
>id    crtc    fb    CRTC x,y    x,y    gamma size    possible crtcs
>23    0    0    0,0        0,0    0           0x00000001
>[...]


如上modetest -M rockchip的信息,我们知道当前系统有两个vop, 有两个输出设备:测试显示输出: 测试显示输出时要把当前系统其他的显示关掉, 因为drm只能允许一个显示输出程序.
> android: adb shell stop

##### 1.3、显示输出命令

>(shell)# modetest -M rockchip -s 88@61:768x1024 -v
>
>freq: 60.53Hz
>
>屏幕上即可看到闪烁的彩条显示,
>如需使用dp输出,将命令中的connector的id换成dp的即可.
>如需使用另一个crtc输出, 将命令中的crtc的id换成另一个crtc的id即可
>如需使用别的分辨率输出, 将命令中768x1024换成connectors modes里面别的分辨率即可


##### 1.4、显示输出效果图
![](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/zjj.sys.display.8.1.shot/modetest.shot.jpg)
源代码稍后另外博客再继续分析！


#### (二)、OpenGLES Test App（Android OpenGLES Native层）

#### （2）、使用OpenGL ES绘制图像
##### 2.1、源代码
**OpenGLESTexturesRGB.cpp**

``` cpp
I:\RK3399_Android8.1_MIPI\frameworks\native\opengl\tests\OpenGLESTexturesRGB\OpenGLESTexturesRGB.cpp

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
     /*
     uint8_t t8[]  = { 
             0xFF, 0xFF, 0xFF, 0xFF, 
             0xFF, 0xFF, 0xFF, 0xFF,
             0xFF, 0xFF, 0xFF, 0xFF, 
             0xFF, 0xFF, 0xFF, 0xFF  };
     */

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
  
     /*
     uint16_t t5551[]  = { 
             0xFF00, 0xFFFF, 0x0000, 0xFFFF, 
             0xFFFF, 0x0000, 0xFFFF, 0x0000,
             0x0000, 0xFFFF, 0x0000, 0xFFFF, 
             0xFFFF, 0x0000, 0xFFFF, 0x0000  };

     uint32_t t32[]  = { 
             0xFF000000, 0xFF000000, 0xFF000000, 0xFF000000, 
             0xFF000000, 0xFF000000, 0xFF000000, 0xFF000000, 
             0xFF000000, 0xFF000000, 0xFF000000, 0xFF000000, 
             0xFF000000, 0xFF000000, 0xFF000000, 0xFF000000
     };
     */

     for(int i = 0; i < 10000; i++){
     glClear(GL_COLOR_BUFFER_BIT);
     //glTexImage2D(GL_TEXTURE_2D, 0, GL_LUMINANCE, 4, 4, 0, GL_LUMINANCE, GL_UNSIGNED_BYTE, t8);
	 glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, 4, 4, 0, GL_RGBA, GL_UNSIGNED_SHORT_4_4_4_4, t16R);
     glDrawTexiOES(0, 0, 0, w/3, h);

     //glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, 4, 4, 0, GL_RGB, GL_UNSIGNED_SHORT_5_6_5, t16);
	 glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, 4, 4, 0, GL_RGBA, GL_UNSIGNED_SHORT_4_4_4_4, t16G);
     glDrawTexiOES(w/3, 0, 0, w/3, h);

     glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, 4, 4, 0, GL_RGBA, GL_UNSIGNED_SHORT_4_4_4_4, t16B);
     glDrawTexiOES(w*2/3, 0, 0, w/3, h);

     //glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, 4, 4, 0, GL_RGBA, GL_UNSIGNED_BYTE, t32);
     //glDrawTexiOES(w*2/3, 0, 0, w/3, h);

     eglSwapBuffers(dpy, surface);
     }
     //sleep(2);      // so you have a chance to admire it
     return 0;
}

```

##### 2.2、编译测试
编译会生成OpenGLESTexturesRGB，然后进行测试。

> 1、连接adb 
> 2、adb push OpenGLESTexturesRGB system/bin 
> 3、进入adb shell
> 4、cd system/bin/
> 5、./OpenGLESTexturesRGB

##### 2.3、显示效果
![](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/zjj.sys.display.8.1.shot/OpenGLESTexturesRGB.png)

源代码稍后另外博客再继续分析！

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


![](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/zjj.sys.display.8.1.shot/zjj.display.sys.OpenGLESUse.png)

#### (三)、SurfaceFlinger test App（Android SurfaceFlinger Native层）
#### （3）、SurfaceFlinger合成图像
##### 3.1、源代码
**SurfaceFlingerTestsRed**
```cpp
I:\RK3399_Android8.1_MIPI\frameworks\native\services\surfaceflinger\tests\SurfaceFlingerTestsRed\Transaction_test_red.cpp

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

// A ScreenCapture is a screenshot from SurfaceFlinger that can be used to check
// individual pixel values for testing purposes.
class ScreenCapture : public RefBase {
public:
    static void captureScreen(sp<ScreenCapture>* sc) {
        sp<IGraphicBufferProducer> producer;
        sp<IGraphicBufferConsumer> consumer;
        BufferQueue::createBufferQueue(&producer, &consumer);
        IGraphicBufferProducer::QueueBufferOutput bufferOutput;
        sp<CpuConsumer> cpuConsumer = new CpuConsumer(consumer, 1);
        sp<ISurfaceComposer> sf(ComposerService::getComposerService());
        sp<IBinder> display(sf->getBuiltInDisplay(
                ISurfaceComposer::eDisplayIdMain));
        SurfaceComposerClient::openGlobalTransaction();
        SurfaceComposerClient::closeGlobalTransaction(true);
        ASSERT_EQ(NO_ERROR, sf->captureScreen(display, producer, Rect(), 0, 0,
                0, INT_MAX, false));
        *sc = new ScreenCapture(cpuConsumer);
    }

    void checkPixel(uint32_t x, uint32_t y, uint8_t r, uint8_t g, uint8_t b) {
        ASSERT_EQ(HAL_PIXEL_FORMAT_RGBA_8888, mBuf.format);
        const uint8_t* img = static_cast<const uint8_t*>(mBuf.data);
        const uint8_t* pixel = img + (4 * (y * mBuf.stride + x));
        if (r != pixel[0] || g != pixel[1] || b != pixel[2]) {
            String8 err(String8::format("pixel @ (%3d, %3d): "
                    "expected [%3d, %3d, %3d], got [%3d, %3d, %3d]",
                    x, y, r, g, b, pixel[0], pixel[1], pixel[2]));
            EXPECT_EQ(String8(), err) << err.string();
        }
    }

private:
    ScreenCapture(const sp<CpuConsumer>& cc) :
        mCC(cc) {
        EXPECT_EQ(NO_ERROR, mCC->lockNextBuffer(&mBuf));
    }

    ~ScreenCapture() {
        mCC->unlockBuffer(mBuf);
    }

    sp<CpuConsumer> mCC;
    CpuConsumer::LockedBuffer mBuf;
};

class LayerUpdateTest : public ::testing::Test {
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
        mDisplayWidth = displayWidth;
        mDisplayHeight = displayHeight;

        // Background surface
        mBGSurfaceControl = mComposerClient->createSurface(
                String8("BG Test Surface"), displayWidth/3, displayHeight,
                PIXEL_FORMAT_RGBA_8888, 0);
        ASSERT_TRUE(mBGSurfaceControl != NULL);
        ASSERT_TRUE(mBGSurfaceControl->isValid());
        fillSurfaceRGBA8(mBGSurfaceControl, 255, 0, 0);

        SurfaceComposerClient::openGlobalTransaction();

        mComposerClient->setDisplayLayerStack(display, 0);

        ASSERT_EQ(NO_ERROR, mBGSurfaceControl->setLayer(INT_MAX-2));
        ASSERT_EQ(NO_ERROR, mBGSurfaceControl->setPosition(0, 0));
        ASSERT_EQ(NO_ERROR, mBGSurfaceControl->show());
        SurfaceComposerClient::closeGlobalTransaction(true);
    }

    virtual void TearDown() {
        mComposerClient->dispose();
        mBGSurfaceControl = 0;
        mComposerClient = 0;
    }

    void waitForPostedBuffers() {
    }

    sp<SurfaceComposerClient> mComposerClient;
    sp<SurfaceControl> mBGSurfaceControl;
    int mDisplayWidth;
    int mDisplayHeight;
};


TEST_F(LayerUpdateTest, LayerMoveWorks) {
    for(int i = 0; i< 5000000; i++){
        SurfaceComposerClient::openGlobalTransaction();
        ASSERT_EQ(NO_ERROR, mBGSurfaceControl->setPosition(0, 0));
        SurfaceComposerClient::closeGlobalTransaction(true);
    }
}


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
> 4、cd system/bin/
> 5、SurfaceFlingerTestsRed

##### 3.3、显示效果
**SurfaceFlingerTestsRed**
![](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/zjj.sys.display.8.1.shot/SurfaceFlinger_test_red.png)

**SurfaceFlingerTestsGreen**
![](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/zjj.sys.display.8.1.shot/SurfaceFlinger_test_green.png)

**SurfaceFlingerTestsBlue**
![](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/zjj.sys.display.8.1.shot/SurfaceFlinger_test_blue.png)

同时运行三个bin文件
![](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/zjj.sys.display.8.1.shot/SurfaceFlinger_test_RGB.png)

##### 3.4、SurfaceFlingerTest代码分析
SurfaceFlinger涉及流程复杂，稍后另外博客再继续分析！

#### (四)、AMS、WMS Test App（Android Apk Java层）
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
编译会生成TestViewportRed/Green/Blue apk
TestViewportRed.apk
TestViewportGreen.apk
TestViewportBlue.apk，然后分别进行测试。

> 1、连接adb 
> 2、adb shell am start -n com.android.testred/.TestActivity
> 3、adb shell am start -n com.android.testgreen/.TestActivity
> 4、adb shell am start -n com.android.testblue/.TestActivity

##### 4.3、显示效果
**TestViewportRed.apk**
![](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/zjj.sys.display.8.1.shot/com.android.testred.png)

**TestViewportGreen.apk**
![](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/zjj.sys.display.8.1.shot/com.android.testgreen.png)

**TestViewportBlue.apk**
![](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/zjj.sys.display.8.1.shot/com.android.testblue.png)

同时运行三个APK
![](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/zjj.sys.display.8.1.shot/com.android.testRGB.png)


Android 默认运行Activity是全屏
要实现上述效果，需要修改源码

```java

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

```

##### 4.4、去掉SystemUI
假如去掉SystemUI Apk（adb root / adb remount / rm -rf system/priv-app/SystemUI）如下效果
**TestViewportRed.apk**
![](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/zjj.sys.display.8.1.shot/nobar_com.android.testred.png)

**TestViewportGreen.apk**
![](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/zjj.sys.display.8.1.shot/nobar_com.android.testgreen.png)

**TestViewportBlue.apk**
![](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/zjj.sys.display.8.1.shot/nobar_com.android.testblue.png)

同时运行三个APK
![](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/zjj.sys.display.8.1.shot/nobar_com.android.testRGB.png)

##### 4.5、TestViewportX代码分析
APK涉及AMS,WMS,SurfaceFlinger流程复杂，稍后另外博客再继续分析！

#### (五)、参考资料(特别感谢)：

[（1）【modetest使用指南】](
https://markyzq.gitbooks.io/rockchip_drm_integration_helper/content/zh/drm_modetest.html) 
[（2）【Ubuntu12.04 编译Android 7.1 mkimage.sh 失败】](https://blog.csdn.net/ansondroider/article/details/85520148) 
