---
title: Android 10 Display System源码分析（2）：Display System 精彩世界（Android 10.0 && Kernel 4.15）
cover: https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/post.cover.pictures/bing-wallpaper-2018.04.23.jpg
categories: 
  - Display
tags:
  - Android
  - Linux
  - Graphics
toc: true
abbrlink: 20210410
date: 2021-04-10 09:25:00
---

**源码路径：**
**AMS、WMS、SurfaceFlinger test App（Android Apk Java层）**
- \zhoujinjian\kahas_edge_v_android10_rk3399\packages\apps\TestViewportRed
- \zhoujinjian\kahas_edge_v_android10_rk3399\packages\apps\TestViewportGreen
- \zhoujinjian\kahas_edge_v_android10_rk3399\packages\apps\TestViewportBlue

**SurfaceFlinger test App（Android Native层）**
- \zhoujinjian\kahas_edge_v_android10_rk3399\packages\apps\SurfaceflingerTestsRed
- \zhoujinjian\kahas_edge_v_android10_rk3399\packages\apps\SurfaceflingerTestsGreen
- \zhoujinjian\kahas_edge_v_android10_rk3399\packages\apps\SurfaceflingerTestsBlue

**OpenGLES test App（Android OpenGLES Native层）**
- \zhoujinjian\kahas_edge_v_android10_rk3399\packages\apps\OpenGLESTexturesRGB

**FrameBuffer test App （Kernel 用户层）**
- \zhoujinjian\kahas_edge_v_android10_rk3399\packages\apps\FrameBufferTest

## (一)、FrameBuffer test App （Kernel 用户层）
### （1）、直接操作framebuffer显示图像
####  1.1、源代码
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
    long int screensize = 0;
    char *fbp = 0;
    int x = 0, y = 0;
    long int location = 0;
​
    // Open the file for reading and writing
    fbfd = open("/dev/graphics/fb0", O_RDWR,0);
    if (fbfd < 0) {
        printf("Error: cannot open framebuffer device.%x\n",fbfd);
        exit(1);
    }
    printf("The framebuffer device was opened successfully.\n");
​
    // Get fixed screen information
    if (ioctl(fbfd, FBIOGET_FSCREENINFO, &finfo)) {
        printf("Error reading fixed information.\n");
        exit(2);
    }
    printf("\ntype:0x%x\n", finfo.type );
    printf("visual:%d\n", finfo.visual );
    printf("line_length:%d\n", finfo.line_length );
    printf("\nsmem_start:0x%lx,smem_len:%u\n", finfo.smem_start, finfo.smem_len );
    printf("mmio_start:0x%lx ,mmio_len:%u\n", finfo.mmio_start, finfo.mmio_len );
​
    // Get variable screen information
    if (ioctl(fbfd, FBIOGET_VSCREENINFO, &vinfo)) {
        printf("Error reading variable information.\n");
        exit(3);
    }
    printf("%dx%d, %dbpp,\n xres_virtual=%d,\n yres_virtual=%d,\n vinfo.xoffset=%d,\n vinfo.yoffset=%d, \n", vinfo.xres, vinfo.yres, vinfo.bits_per_pixel,vinfo.xres_virtual,vinfo.yres_virtual,vinfo.xoffset,vinfo.yoffset);
​
    screensize = finfo.line_length * vinfo.yres_virtual;
    // Map the device to memory
    fbp = (char *)mmap(0, screensize, PROT_READ | PROT_WRITE, MAP_SHARED,fbfd, 0);
    if ((long)fbp == -1) {
        printf("Error: failed to map framebuffer device to memory.\n");
        exit(4);
    }
    printf("The framebuffer device was mapped to memory successfully.\n");
    //red
    for ( y = 0; y < 1920; y++ ) {
        for ( x = 0; x < 360; x++ ) {
            location = (x) * (vinfo.bits_per_pixel / 8) + (y) * finfo.line_length;
            if ( vinfo.bits_per_pixel == 32 ) {
                *(fbp + location) = 255;
                *(fbp + location + 1) = 0;
                *(fbp + location + 2) = 0;
                *(fbp + location + 3) = 255;
            }
        }
    }
    //green
    for ( y = 0; y < 1920; y++ ) {
        for ( x = 360; x < 720; x++ ) {
            location = (x) * (vinfo.bits_per_pixel / 8) + (y) * finfo.line_length;
            if ( vinfo.bits_per_pixel == 32 ) {
                *(fbp + location) = 0;
                *(fbp + location + 1) = 255;
                *(fbp + location + 2) = 0;
                *(fbp + location + 3) = 255;
            }
        }
    }
    //blue
    for ( y = 0; y < 1920; y++ ) {
        for ( x = 720; x < 1080; x++ ) {
            location = (x) * (vinfo.bits_per_pixel / 8) + (y) * finfo.line_length;
            if ( vinfo.bits_per_pixel == 32 ) {
                *(fbp + location) = 0;
                *(fbp + location + 1) = 0;
                *(fbp + location + 2) = 255;
                *(fbp + location + 3) = 255;
            }
        }
    }
    unsigned char *pTemp = (unsigned char *)fbp;
    int i, j;
    //Start coordinates (x, y), end coordinates (right, bottom)
    x = 0;
    y = 640;
    int right = 1080;//vinfo.xres;
    int bottom = 1280;//vinfo.yres;
​
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
    //note：vinfo.xoffset =0 vinfo.yoffset =0 Otherwise FBIOPAN_DISPLAY is unsuccessful
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
​
```

**Android.mk**

> ​​​​LOCAL_PATH:= $(call my-dir)
> 
>​​​​​include $(CLEAR_VARS)

> ​​​​​LOCAL_MODULE:= FrameBufferTest
> ​​​​​​​
> ​​​​​​​LOCAL_SDK_VERSION := 21
> ​​​​​​​LOCAL_NDK_STL_VARIANT := c++_static
> ​​​​​​​
> ​​​​​​​LOCAL_SRC_FILES:= \
> ​​​​​​​    FrameBufferTest.cpp \
> ​​​​​​​
> ​​​​​​​OCAL_SHARED_LIBRARIES := \
> ​​​​​​​    liblog \
> ​​​​​​​    libutils \
> ​​​​​​​
> ​​​​​​​include $(BUILD_EXECUTABLE)

#### **1.2、编译测试**
编译会生成FrameBufferTest，然后进行测试。

```
1、进入adb shell
2、cd system/bin/
3、./FrameBufferTest
```

#### **1.3、显示效果**

![](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/Android10.Display.2/Android10.0.Dispaly.Sys.FrameBufferTest.jpg)


## (二)、OpenGLES test App（Android OpenGLES Native层）
### （2）、使用OpenGL ES绘制图像
#### 2.1、源代码
**OpenGLESTexturesRGB.cpp**

``` cpp
#include <stdlib.h>
#include <stdio.h>
​
#include <EGL/egl.h>
#include <GLES/gl.h>
#include <GLES/glext.h>
​
#include <WindowSurface.h>
#include <EGLUtils.h>
​
using namespace android;
​
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
​
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
​
    surface = eglCreateWindowSurface(dpy, config, window, NULL);
    context = eglCreateContext(dpy, config, NULL, NULL);
    eglMakeCurrent(dpy, surface, surface, context);   
    eglQuerySurface(dpy, surface, EGL_WIDTH, &w);
    eglQuerySurface(dpy, surface, EGL_HEIGHT, &h);
​
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
​
    uint16_t t16R[]  = { 
            0xF000, 0xF000, 0xF000, 0xF000, 
            0xF000, 0xF000, 0xF000, 0xF000, 
            0xF000, 0xF000, 0xF000, 0xF000,  
            0xF000, 0xF000, 0xF000, 0xF000,  };
​
    uint16_t t16G[]  = { 
            0x0F00, 0x0F00, 0x0F00, 0x0F00, 
            0x0F00, 0x0F00, 0x0F00, 0x0F00, 
            0x0F00, 0x0F00, 0x0F00, 0x0F00,  
            0x0F00, 0x0F00, 0x0F00, 0x0F00,  };
​
    uint16_t t16B[]  = { 
            0x00F0, 0x00F0, 0x00F0, 0x00F0, 
            0x00F0, 0x00F0, 0x00F0, 0x00F0,
            0x00F0, 0x00F0, 0x00F0, 0x00F0, 
            0x00F0, 0x00F0, 0x00F0, 0x00F0,  };
  
    for(int i = 0; i < 10000; i++){
    glClear(GL_COLOR_BUFFER_BIT);
    glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, 4, 4, 0, GL_RGBA, GL_UNSIGNED_SHORT_4_4_4_4, t16R);
    glDrawTexiOES(0, 0, 0, w/3, h);
​
    glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, 4, 4, 0, GL_RGBA, GL_UNSIGNED_SHORT_4_4_4_4, t16G);
    glDrawTexiOES(w/3, 0, 0, w/3, h);
​
    glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, 4, 4, 0, GL_RGBA, GL_UNSIGNED_SHORT_4_4_4_4, t16B);
    glDrawTexiOES(w*2/3, 0, 0, w/3, h);
​
    eglSwapBuffers(dpy, surface);
    }
    //sleep(2);// so you have a chance to admire it
    return 0;
}
​
```

#### 2.2、编译测试
编译会生成OpenGLESTestRGB，然后进行测试。

```
1、连接adb
2、adb push OpenGLESTestRGB system/bin
3、进入adb shell
4、system/bin/OpenGLESTestRGB
```

#### 2.3、显示效果
![](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/Android10.Display.2/Android10.0.Dispaly.Sys.OpenGLESTestRGB.jpg)

## (三)、SurfaceFlinger test App（Android SurfaceFlinger Native层）
### （3）、SurfaceFlinger合成图像
#### 3.1、源代码
**SurfaceFlingerTestsRed**

``` cpp
#include <algorithm>
#include <functional>
#include <limits>
#include <ostream>
​
#include <gtest/gtest.h>
​
#include <android/native_window.h>
​
#include <gui/ISurfaceComposer.h>
#include <gui/LayerState.h>
​
#include <gui/Surface.h>
#include <gui/SurfaceComposerClient.h>
#include <private/gui/ComposerService.h>
​
#include <ui/DisplayInfo.h>
#include <ui/Rect.h>
#include <utils/String8.h>
​
#include <math.h>
#include <math/vec3.h>
​
namespace android {
​
namespace {
​
struct Color {
    uint8_t r;
    uint8_t g;
    uint8_t b;
    uint8_t a;
​
    static const Color RED;
};
​
const Color Color::RED{255, 0, 0, 255};
​
// Fill a region with the specified color.
void fillBufferColor(const ANativeWindow_Buffer& buffer, const Rect& rect, const Color& color) {
    int32_t x = rect.left;
    int32_t y = rect.top;
    int32_t width = rect.right - rect.left;
    int32_t height = rect.bottom - rect.top;
​
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
​
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
​
using Transaction = SurfaceComposerClient::Transaction;
​
class LayerTransactionTest : public ::testing::Test {
protected:
    void SetUp() override {
        mClient = new SurfaceComposerClient;
        ASSERT_EQ(NO_ERROR, mClient->initCheck()) << "failed to create SurfaceComposerClient";
​
        ASSERT_NO_FATAL_FAILURE(SetUpDisplay());
    }
​
    sp<SurfaceControl> createLayer(const char* name, uint32_t width, uint32_t height,
                                   uint32_t flags = 0) {
        auto layer =
                mClient->createSurface(String8(name), width, height, PIXEL_FORMAT_RGBA_8888, flags);
        EXPECT_NE(nullptr, layer.get()) << "failed to create SurfaceControl";
​
        status_t error = Transaction()
                                 .setLayerStack(layer, mDisplayLayerStack)
                                 .setLayer(layer, mLayerZBase)
                                 .apply();
        if (error != NO_ERROR) {
            ADD_FAILURE() << "failed to initialize SurfaceControl";
            layer.clear();
        }
​
        return layer;
    }
​
    ANativeWindow_Buffer getLayerBuffer(const sp<SurfaceControl>& layer) {
        // wait for previous transactions (such as setSize) to complete
        Transaction().apply(true);
​
        ANativeWindow_Buffer buffer = {};
        EXPECT_EQ(NO_ERROR, layer->getSurface()->lock(&buffer, nullptr));
​
        return buffer;
    }
​
    void postLayerBuffer(const sp<SurfaceControl>& layer) {
        ASSERT_EQ(NO_ERROR, layer->getSurface()->unlockAndPost());
​
        // wait for the newly posted buffer to be latched
        waitForLayerBuffers();
    }
​
    void fillLayerColor(const sp<SurfaceControl>& layer, const Color& color) {
        ANativeWindow_Buffer buffer;
        ASSERT_NO_FATAL_FAILURE(buffer = getLayerBuffer(layer));
        fillBufferColor(buffer, Rect(0, 0, buffer.width, buffer.height), color);
        postLayerBuffer(layer);
    }
​
    void fillLayerQuadrant(const sp<SurfaceControl>& layer, const Color& topLeft,
                           const Color& topRight, const Color& bottomLeft,
                           const Color& bottomRight) {
        ANativeWindow_Buffer buffer;
        ASSERT_NO_FATAL_FAILURE(buffer = getLayerBuffer(layer));
        ASSERT_TRUE(buffer.width % 2 == 0 && buffer.height % 2 == 0);
​
        const int32_t halfW = buffer.width / 2;
        const int32_t halfH = buffer.height / 2;
        fillBufferColor(buffer, Rect(0, 0, halfW, halfH), topLeft);
        fillBufferColor(buffer, Rect(halfW, 0, buffer.width, halfH), topRight);
        fillBufferColor(buffer, Rect(0, halfH, halfW, buffer.height), bottomLeft);
        fillBufferColor(buffer, Rect(halfW, halfH, buffer.width, buffer.height), bottomRight);
​
        postLayerBuffer(layer);
    }
​
    sp<SurfaceComposerClient> mClient;
​
    sp<IBinder> mDisplay;
    uint32_t mDisplayWidth;
    uint32_t mDisplayHeight;
    uint32_t mDisplayLayerStack;
​
    // leave room for ~256 layers
    const int32_t mLayerZBase = std::numeric_limits<int32_t>::max() - 256;
​
private:
    void SetUpDisplay() {
        mDisplay = mClient->getInternalDisplayToken();
        ASSERT_NE(nullptr, mDisplay.get()) << "failed to get built-in display";
​
        // get display width/height
        DisplayInfo info;
        SurfaceComposerClient::getDisplayInfo(mDisplay, &info);
        mDisplayWidth = info.w;
        mDisplayHeight = info.h;
​
        // After a new buffer is queued, SurfaceFlinger is notified and will
        // latch the new buffer on next vsync.  Let's heuristically wait for 3
        // vsyncs.
        mBufferPostDelay = int32_t(1e6 / info.fps) * 3;
​
        mDisplayLayerStack = 0;
        // set layer stack (b/68888219)
        Transaction t;
        t.setDisplayLayerStack(mDisplay, mDisplayLayerStack);
        t.apply();
    }
​
    void waitForLayerBuffers() { usleep(mBufferPostDelay); }
​
    int32_t mBufferPostDelay;
};
​
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
​
```

绿色和蓝色的源码类似就不贴出啦，请直接看源码。

#### 3.2、编译测试
编译会SurfaceFlingerTestsRed/Green/Blue生成
SurfaceFlingerTestsRed
SurfaceFlingerTestsGreen
SurfaceFlingerTestsBlue，然后分别进行测试。

```
1、连接adb
2、adb push SurfaceFlingerTestsRed system/bin
3、进入adb shell
4、system/bin/SurfaceFlingerTestsRed
```

#### 3.3、显示效果
**SurfaceFlingerTestsRed​​​​​​​**
![](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/Android10.Display.2/Android10.0.Dispaly.Sys.SurfaceFlingerTestsRed.jpg)

**SurfaceFlingerTestsGreen​​​​​​​**
![](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/Android10.Display.2/Android10.0.Dispaly.Sys.SurfaceFlingerTestsGreen.jpg)

**SurfaceFlingerTestsBlue**
![](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/Android10.Display.2/Android10.0.Dispaly.Sys.SurfaceFlingerTestsBlue.jpg)

**同时运行三个bin文件**
![](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/Android10.Display.2/Android10.0.Dispaly.Sys.OpenGLESTestRGB.jpg)

## (四)、AMS、WMS test App（Android Apk Java层）
### （4）、Apk测试AMS,WMS,SurfaceFlinger
#### 4.1、源代码
TestActivity.java &&TestView.java

``` java
public class TestActivity extends Activity {
    private final static String TAG = "TestActivity";
    TestView mView;
​
    @Override
    protected void onCreate(Bundle icicle) {
        super.onCreate(icicle);
        mView = new TestView(getApplication());
        mView.setFocusableInTouchMode(true);
        setContentView(mView);
    }
​
    @Override
    protected void onPause() {
        super.onPause();
        mView.onPause();
    }
​
    @Override
    protected void onResume() {
        super.onResume();
        mView.onResume();
    }
}
​
class TestView extends GLSurfaceView {
    TestView(Context context) {
        super(context);
        init();
    }
​
    public TestView(Context context, AttributeSet attrs) {
        super(context, attrs);
        init();
    }
​
    private void init() {
        setRenderer(new Renderer());
        setRenderMode(RENDERMODE_WHEN_DIRTY);
    }
​
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

#### 4.2、编译测试
编译会TestViewportRed/Green/Blue apk生成
TestViewportRed.apk
TestViewportGreen.apk
TestViewportBlue.apk，然后分别进行测试。

```
1、连接adb
2、adb shell am start -n com.android.testred/.TestActivity
3、adb shell am start -n com.android.testgreen/.TestActivity
4、adb shell am start -n com.android.testblue/.TestActivity
```

#### 4.3、显示效果


**TestViewportRed.apk**
![](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/Android10.Display.2/Android10.0.Dispaly.Sys.TestViewortRed.jpg)

**​​​​​​​TestViewportGreen.apk**
![](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/Android10.Display.2/Android10.0.Dispaly.Sys.TestViewortGreen.jpg)

**TestViewportBlue.apk**
![](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/Android10.Display.2/Android10.0.Dispaly.Sys.TestViewortBlue.jpg)