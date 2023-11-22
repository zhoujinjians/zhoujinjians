---
title: Android Graphics Tests 程序学习（1）： 使用OpenGL EGL一般顺序（configdump & tritex & textures & fillrate & filter& finish）
cover:  https://raw.githubusercontent.com/zhoujinjian777/PicGo/master/hexo.themes/bing-wallpaper-2018.04.33.jpg
categories: 
  - OpenGL
tags:
  - Android
  - OpenGL
toc: true
abbrlink: 20190408
date: 2019-04-08 09:25:00
---



--------------------------------------------------------------------------------

注：文章都是通过阅读各位前辈总结的资料 Android 8.x && Linux（kernel 4.x）Qualcomm平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网（◕‿◕），如有侵权，请联系删除，禁止转载（©Qualcomm ©Android @Linux 版权所有），谢谢。

首先感谢：

[【EGL函数API文档】](https://www.zybuluo.com/cxm-2016/note/572030)
[【OpenGL ES EGL介绍】](https://blog.csdn.net/cauchyweierstrass/article/details/53189449)

正是由于前人的分析和总结，帮助我节约了大量的时间和精力，再次感谢！！！

Google Pixel、Pixel XL 内核代码（==**文章基于 Kernel-4.x**==）：
 [Kernel source for Pixel 2 (walleye) and Pixel 2 XL (taimen) - GitHub](https://github.com/nathanchance/wahoo)

AOSP 源码（==**文章基于 Android 8.x**==）：
 [ Android 系统全套源代码分享 (更新到 8.1.0_r1)](https://testerhome.com/topics/2229)

--------------------------------------------------------------------------------
==源码（部分）==：

>opengl

- android/frameworks/native/opengl/tests/configdump
- android/frameworks/native/opengl/tests/tritex
- android/frameworks/native/opengl/tests/testViewport
- android/frameworks/native/opengl/tests/textures
- android/frameworks/native/opengl/tests/filter
- android/frameworks/native/opengl/tests/fillrate 
- android/frameworks/native/opengl/tests/finish

--------------------------------------------------------------------------------

#### (一)、configdump运行结果

##### 1、预备条件
为了便于观察学习现象，我将Android系统SystemUI、Launcher3统统移除了，将桌面换成了一整个空白的画面。

``` cpp
adb root
adb remount
adb shell
cd system/priv-app/xxx
rm -rf *
```

简易Launcher源码（android/frameworks/native/opengl/tests/testViewport）：
``` cpp
adb root
adb remount
adb shell
mkdir testViewport
adb push TestViewport /system/app/testViewport
```
然后重启系统，等开机动画完成后，运行命令

``` cpp
adb shell am start -n com.android.test/.TestActivity
```
整个桌面就像是一张白色的纸张了。

![Alt text | center](https://raw.githubusercontent.com/zhoujinjian777/PicGo/master/others/OpenGL-EGL-all-white-screen.png)


##### 2、运行test-opengl-configdump

``` cpp
adb push test-opengl-configdump/test-opengl-configdump  /data/nativetest64/
adb shell
cd /data/nativetest64/
chmod 0777 test-opengl-configdump
./test-opengl-configdump

EGLConfig[0]
	EGL_BUFFER_SIZE                 :         16 (0x00000010)
	EGL_ALPHA_SIZE                  :          0 (0x00000000)
	EGL_BLUE_SIZE                   :          5 (0x00000005)
	EGL_GREEN_SIZE                  :          6 (0x00000006)
	EGL_RED_SIZE                    :          5 (0x00000005)
	EGL_DEPTH_SIZE                  :          0 (0x00000000)
	EGL_STENCIL_SIZE                :          0 (0x00000000)
	EGL_CONFIG_CAVEAT               :      12344 (0x00003038)
	EGL_CONFIG_ID                   :          1 (0x00000001)
	EGL_LEVEL                       :          0 (0x00000000)
	EGL_MAX_PBUFFER_HEIGHT          :      16384 (0x00004000)
	EGL_MAX_PBUFFER_WIDTH           :      16384 (0x00004000)
	EGL_MAX_PBUFFER_PIXELS          :  268435456 (0x10000000)
	EGL_NATIVE_RENDERABLE           :          1 (0x00000001)
	EGL_NATIVE_VISUAL_ID            :          4 (0x00000004)
	EGL_NATIVE_VISUAL_TYPE          :         -1 (0xffffffff)
	EGL_SAMPLES                     :          0 (0x00000000)
	EGL_SAMPLE_BUFFERS              :          0 (0x00000000)
	EGL_SURFACE_TYPE                :       5541 (0x000015a5)
	EGL_TRANSPARENT_TYPE            :      12344 (0x00003038)
	EGL_TRANSPARENT_BLUE_VALUE      :         -1 (0xffffffff)
	EGL_TRANSPARENT_GREEN_VALUE     :         -1 (0xffffffff)
	EGL_TRANSPARENT_RED_VALUE       :         -1 (0xffffffff)
	EGL_BIND_TO_TEXTURE_RGB         :          1 (0x00000001)
	EGL_BIND_TO_TEXTURE_RGBA        :          0 (0x00000000)
	EGL_MIN_SWAP_INTERVAL           :          0 (0x00000000)
	EGL_MAX_SWAP_INTERVAL           :          1 (0x00000001)
	EGL_LUMINANCE_SIZE              :          0 (0x00000000)
	EGL_ALPHA_MASK_SIZE             :          0 (0x00000000)
	EGL_COLOR_BUFFER_TYPE           :      12430 (0x0000308e)
	EGL_RENDERABLE_TYPE             :         69 (0x00000045)
	EGL_MATCH_NATIVE_PIXMAP         :          0 (0x00000000)
	EGL_CONFORMANT                  :         69 (0x00000045)
	EGL_COLOR_COMPONENT_TYPE_EXT    :      13114 (0x0000333a)
```



##### 3、configdump源码

``` cpp
#include <stdlib.h>
#include <stdio.h>

#include <EGL/egl.h>
#include <EGL/eglext.h>

#define ATTRIBUTE(_attr) { _attr, #_attr }

struct Attribute {
    EGLint attribute;
    char const* name;
};

// clang-format off
Attribute attributes[] = {
        ATTRIBUTE( EGL_BUFFER_SIZE ),
        ATTRIBUTE( EGL_ALPHA_SIZE ),
        ATTRIBUTE( EGL_BLUE_SIZE ),
        ATTRIBUTE( EGL_GREEN_SIZE ),
        ATTRIBUTE( EGL_RED_SIZE ),
        ATTRIBUTE( EGL_DEPTH_SIZE ),
        ATTRIBUTE( EGL_STENCIL_SIZE ),
        ATTRIBUTE( EGL_CONFIG_CAVEAT ),
        ATTRIBUTE( EGL_CONFIG_ID ),
        ATTRIBUTE( EGL_LEVEL ),
        ATTRIBUTE( EGL_MAX_PBUFFER_HEIGHT ),
        ATTRIBUTE( EGL_MAX_PBUFFER_WIDTH ),
        ATTRIBUTE( EGL_MAX_PBUFFER_PIXELS ),
        ATTRIBUTE( EGL_NATIVE_RENDERABLE ),
        ATTRIBUTE( EGL_NATIVE_VISUAL_ID ),
        ATTRIBUTE( EGL_NATIVE_VISUAL_TYPE ),
        ATTRIBUTE( EGL_SAMPLES ),
        ATTRIBUTE( EGL_SAMPLE_BUFFERS ),
        ATTRIBUTE( EGL_SURFACE_TYPE ),
        ATTRIBUTE( EGL_TRANSPARENT_TYPE ),
        ATTRIBUTE( EGL_TRANSPARENT_BLUE_VALUE ),
        ATTRIBUTE( EGL_TRANSPARENT_GREEN_VALUE ),
        ATTRIBUTE( EGL_TRANSPARENT_RED_VALUE ),
        ATTRIBUTE( EGL_BIND_TO_TEXTURE_RGB ),
        ATTRIBUTE( EGL_BIND_TO_TEXTURE_RGBA ),
        ATTRIBUTE( EGL_MIN_SWAP_INTERVAL ),
        ATTRIBUTE( EGL_MAX_SWAP_INTERVAL ),
        ATTRIBUTE( EGL_LUMINANCE_SIZE ),
        ATTRIBUTE( EGL_ALPHA_MASK_SIZE ),
        ATTRIBUTE( EGL_COLOR_BUFFER_TYPE ),
        ATTRIBUTE( EGL_RENDERABLE_TYPE ),
        ATTRIBUTE( EGL_MATCH_NATIVE_PIXMAP ),
        ATTRIBUTE( EGL_CONFORMANT ),
        ATTRIBUTE( EGL_COLOR_COMPONENT_TYPE_EXT ),
};
// clang-format on

int main(int /*argc*/, char** /*argv*/) {
    EGLConfig* configs;
    EGLint n;
    EGLDisplay dpy = eglGetDisplay(EGL_DEFAULT_DISPLAY);
    eglInitialize(dpy, 0, 0);
    eglGetConfigs(dpy, NULL, 0, &n);
    configs = new EGLConfig[n];
    eglGetConfigs(dpy, configs, n, &n);

    for (EGLint i=0 ; i<n ; i++) {
        printf("EGLConfig[%d]\n", i);
        for (unsigned attr = 0 ; attr<sizeof(attributes)/sizeof(Attribute) ; attr++) {
            EGLint value;
            eglGetConfigAttrib(dpy, configs[i], attributes[attr].attribute, &value);
            printf("\t%-32s: %10d (0x%08x)\n", attributes[attr].name, value, value);
        }
    }

    delete [] configs;
    eglTerminate(dpy);
    return 0;
}
```
#### (二)、初识EGL函数API文档
##### 1、EGL介绍
EGL 是 OpenGL ES 和底层 Native 平台视窗系统之间的接口。OpenGL ES 本质上是一个图形渲染管线的状态机，而 EGL 则是用于监控这些状态以及维护 Frame buffer 和其他渲染 Surface 的外部层。EGL提供如下机制：

- 与设备的原生窗口系统通信
- 查询绘图表面的可用类型和配置
- 创建绘图表面
- 在OpenGL ES 和其他图形渲染API之间同步渲染
- 管理纹理贴图等渲染资源

##### EGL类型
EGLBoolean

> EGL中的布尔类型。
> 
> typedef unsigned int EGLBoolean;

##### EGLDisplay

> 不透明类型，封装了与底层系统的交互，用于充当与原生窗口之间的接口。
> 
> typedef void * EGLDisplay;

##### EGLint

> EGL整数类型。
> 
> typedef int32_t EGLint;

##### EGLNativeDisplayType

> 用于匹配原生窗口系统的显示类型。
> 
> typedef void * EGLNativeDisplayType;

##### EGL常量
###### 布尔值

> EGL_FALSE：条件为假 
> EGL_TRUE：条件为真

###### 创建窗口的属性

> EGL_RENDER_BUFFER：指定渲染所用的缓冲区

##### 错误代码

``` cpp
EGL_SUCCESS：没有错误
EGL_NOT_INITIALIZED：没有初始化
EGL_BAD_ACCESS：数据访问失败
EGL_BAD_ALLOC：内存分配失败
EGL_BAD_ATTRIBUTE：错误的属性
EGL_BAD_CONFIG：错误的配置
EGL_BAD_CONTEXT：错误的上下文
EGL_BAD_CURRENT_SURFACE：当前Surface对象错误
EGL_BAD_DISPLAY：错误的设备对象
EGL_BAD_MATCH：无法匹配
EGL_BAD_NATIVE_PIXMAP：错误的像素图
EGL_BAD_NATIVE_WINDOW：错误的本地窗口对象
EGL_BAD_PARAMETER：错误的参数
EGL_BAD_SURFACE：错误的Surface对象
EGL_CONTEXT_LOST：上下文丢失
```

##### 配置属性

``` cpp
EGL_ALPHA_SIZE：颜色缓冲区中的透明度分量的位数
EGL_ALPHA_MASK_SIZE：透明度掩码位数
EGL_BUFFER_SIZE：颜色缓冲区中颜色分量的位数
EGL_BLUE_SIZE：颜色缓冲区中的蓝色分量的位数
EGL_BIND_TO_TEXTURE_RGB：是否可以绑定RGB纹理
EGL_BIND_TO_TEXTURE_RGBA：是否可以绑定EGBA纹理
EGL_CONFIG_CAVEAT：注意事项
EGL_COLOR_BUFFER_TYPE：颜色缓冲区类型
EGL_CONFORMANT：创建的上下文是否兼容
EGL_CONFIG_ID：配置信息ID
EGL_DEPTH_SIZE：深度缓冲区位数
EGL_GREEN_SIZE：颜色缓冲区中的绿色分量的位数
EGL_LEVEL：帧缓冲区级别
EGL_LUMINANCE_SIZE：颜色缓冲区亮度位数
EGL_MAX_PBUFFER_HEIGHT：Pbuffer的最大高度
EGL_MAX_PBUFFER_PIXELS：Pbuffer的最大尺寸
EGL_MAX_PBUFFER_WIDTH：Pbuffer的最大宽度
EGL_MATCH_NATIVE_PIXMAP
EGL_MIN_SWAP_INTERVAL：最小缓冲区交换间隔
EGL_MAX_SWAP_INTERVAL：最大缓冲区交换间隔
EGL_NATIVE_RENDERABLE：是否可用原生渲染库渲染
EGL_NONE
EGL_NATIVE_VISUAL_ID：原生窗口系统的可视ID
EGL_NATIVE_VISUAL_TYPE：原生窗口系统的可视类型
EGL_RED_SIZE：颜色缓冲区中的红色分量的位数
EGL_RENDERABLE_TYPE：可渲染接口类型
EGL_STENCIL_SIZE：模板缓冲区位数
EGL_SAMPLES：每个像素的样本数量
EGL_SAMPLE_BUFFERS：可用多重采样缓冲区数量
EGL_SURFACE_TYPE：EGL表面类型
EGL_TRANSPARENT_TYPE：透明度类型
EGL_TRANSPARENT_BLUE_VALUE：透明的蓝色值
EGL_TRANSPARENT_GREEN_VALUE：透明的绿色值
EGL_TRANSPARENT_RED_VALUE：透明的红色值
显示设备对象
EGL_NO_DISPLAY：当前无可用设备
显示设备类型
EGL_DEFAULT_DISPLAY：默认为当前使用设备
```

#### 2、EGL介绍
##### 2.1、 EGL函数
在学习具体的实例之前，先来学习 EGL函数。
#### eglGetDisplay
> 获得并与可用设备进行连接。
> 
> EGLDisplay eglGetDisplay(EGLNativeDisplayType display_id);
> 
> display_id：当前需要连接的设备类型 return：已经连接上的设备对象 EGLDisplay eglDisplay =
> eglGetDisplay(EGL_DEFAULT_DISPLAY);

####  eglChooseConfig

> 在初始化EGL时，我们需要列出并让EGL选择最合适的配置。
>
>EGLBoolean eglChooseConfig(EGLDisplay dpy, const EGLint *attrib_list, EGLConfig *configs, EGLint config_size, EGLint *num_config);
> 
>dpy：已连接的设备
>attrib_list：传入的配置信息数组
>configs：保存返回的配置信息的数组
>config_size：传入的配置数组的长度
>num_config：保存返回的配置信息的数组的长度

``` cpp
EGLConfig config;
EGLint numConfigs = 0;
EGLint attribList[] =
        {
                EGL_RENDERABLE_TYPE, EGL_OPENGL_ES_BIT,
                EGL_RED_SIZE, 8,
                EGL_GREEN_SIZE, 8,
                EGL_BLUE_SIZE, 8,
                EGL_ALPHA_SIZE, 8,
                EGL_DEPTH_SIZE, 16,
                EGL_NONE
        };
if (!eglChooseConfig(context->eglDisplay, attribList, &config, 1, &numConfigs)) {
    return GL_FALSE;
}
```

#### eglInitialize
> 一般在成功打开设备连接之后需要初始化EGL。初始化过程将会对EGL内部的数据结构进行设置，然后返回EGL的主次版本号。
> 
> EGLBoolean eglInitialize(EGLDisplay dpy, EGLint *major, EGLint
> *minor);
> 
>  - dpy：指定EGL设备对象。
>  - major：设备主版本号。
>  - minor：设备次版本号。
>  
>     GLint majorVersion;
>     GLint minorVersion;
>     if (!eglInitialize(eglDisplay, &majorVersion, &minorVersion)) {
>         return EGL_FALSE;
>     }

#### eglCreateContext

> 渲染上下文是OpenGL ES的内部数据结构，包含操作所需的所有状态信息。例如程序中使用的顶点着色器或者片元着色器的引用。OpenGL ES必须有一个可用的上下文才能绘图。使用下面的函数可以创建一个上下文：
> 
> EGLContext eglCreateContext(EGLDisplay display, EGLConfig config, EGLContext shareContext, const EGLint *attribList);
> 
>  - display：指定显示连接
>  - config：指定配置对象
>  - shareContext：允许多个EGL上下文共享特定的数据，EGL_NO_CONTEXT参数表示没有共享
>  - attribList：指定创建上下文使用的属性列表
>  - return：创建的上下文对象

``` cpp
    EGLint contextAttribs[] = {EGL_CONTEXT_CLIENT_VERSION, 3, EGL_NONE};
    context->eglContext = eglCreateContext(context->eglDisplay, config, EGL_NO_CONTEXT, contextAttribs);
    if (context->eglContext == EGL_NO_CONTEXT) {
        return GL_FALSE;
    }
```

#### eglCreateWindowSurface

> 一旦我们有了符合渲染需求的EGLConfig，就为窗口创建做好了准备。调用如下函数可以创建一个窗口。
>
>EGLSurface eglCreateWindowSurface(EGLDisplay display, EGLConfig config, EGLNativeWindowType window,const EGLint *attribList);
>
> - display：已连接的设备对象
> - config：指定的配置对象
> - window：指定原生窗口对象
> - attriList：指定窗口的属性列表
> - return：EGL渲染区域对象
> 
>这个函数以我们到原生显示管理器的连接和前一步获得的EGLConfig为参数。此外，它需要原生窗口系统事先创建一个窗口。因为EGL是许多不同窗口系统和OpenGL ES之间的软件接口层。最后这个函数需要一个属性列表；但是，这个列表中的属性与参数属性不完全相同，并且额外使用到了创建窗口的属性。该函数在多种情况下都有可能失败。

``` cpp
 context->eglSurface = eglCreateWindowSurface(context->eglDisplay, config, context->nativeWindow,
                                                 NULL);
    if (context->eglSurface == EGL_NO_SURFACE) {
        EGLint error;
        while((error = eglGetError()) != EGL_SUCCESS){
            switch(error) {
                case EGL_BAD_MATCH:{
                    //提供的原生窗口不匹配或者不支持渲染
                }
                case EGL_BAD_CONFIG:{
                    //系统不支持该配置
                }
                case EGL_BAD_NATIVE_WINDOW:{
                    //提供的原生窗口无效
                }
                case EGL_BAD_ALLOC:{
                    //无法为新的EGL分配资源或者该窗口已经被关联
                }
            }
        }
    }
```

#### eglGetConfigAttrib

> 如果我们获获取了一个EGL配置对象，我们可以通过下列函数查询该对象中指定属性的值。

> EGLBoolean eglGetConfigAttrib(EGLDisplay dpy, EGLConfig config, EGLint attribute, EGLint *value);

> -  dpy：已连接的设备对象
> -  config：待查询的配置对象
> -  attribute：需要查询的参数属性
> - value：返回的查询结果
> -  return：查询结果，如果attribute不是有效属性，则产生一个EGL_BAD_ATTRIBUTE错误。

#### eglGetConfigs

> 在初始化EGL之后，我们需要给EGL选择一组配置。
> 
> EGLBoolean eglGetConfigs(EGLDisplay dpy, EGLConfig *configs, EGLint
> config_size, EGLint *num_config);
> 
> - dpy：已连接设备对象 
> - configs：保存配置信息的列表 
> - size：configs的长度 
> - num_config：EGL返回的配置信息数量
> -  return：查询结果状态
> 
> 通常情况下，我们有两种方式使用该函数。首先，我们指定configs参数为NULL，此时EGL会查询所有可用的EGLConfigs数量并赋值给num_config，但此时不会有任何其他信息返回。
> 
> 另外，我们也可以创建一个未初始化的EGLConfig，并作为函数的参数传入。此时，EGL将会查询不超过config_size数量的配置信息存入到configs，并通过num_config返回保存数据的数量。

#### eglGetError
> EGL中大部分函数在成功时都会返回EGL_TRUE，否则返回EGL_FALSE。但是，我们仅从这个返回值上并不能看出错误原因是什么。如果想要明确的知道EGL的错误代码，应该调用下列函数。
> 
> EGLint eglGetError(void);
> 
> return：见 [EGL常量-错误代码]

#### eglMakeCurrent
> 
> 因为一个应用程序可能创建多个EGLContext用作不同的用途，所以我们需要指定关联特定的EGLContext和渲染表面——这一过程被称为“指定当前上下文”。
> 
> EGLBoolean eglMakeCurrent(EGLDisplay display, EGLSurface draw,
> EGLSurface read, EGLContext context);
> 
>  - display：指定EGL显示设备 draw：指定EGL绘图表面 read：指定EGL读取表面 context：指定连接到该表面的渲染上下文
> - return：函数时候执行成功

#### (三)、使用EGL一般顺序：

##### 3.1、使用EGL首先必须创建，建立本地窗口系统和OpenGL ES的连接。---==eglDisplay()==

> EGLDisplay eglDisplay(EGLNativeDisplayType displayId)

EGL提供了平台无关类型EGLDisplay表示窗口。定义EGLNativeDisplayType是为了匹配原生窗口系统的显示类型，对于Windows，EGLNativeDisplayType被定义为HDC，对于Linux系统，被定义为Display*类型，对于Android系统，定义为ANativeWindow *类型，为了方便的将代码转移到不同的操作系统上，应该传入EGL_DEFAULT_DISPLAY，返回与默认原生窗口的连接。如果连接不可用，则返回EGL_NO_DISPLAY。

##### 3.2、初始化EGL ---==eglInitialize()==

创建与本地原生窗口的连接后需要初始化EGL，使用函数eglInitialize进行初始化操作。如果 EGL 不能初始化，它将返回EGL_FALSE，并将EGL错误码设置为EGL_BAD_DISPLAY表示制定了不合法的EGLDisplay，或者EGL_NOT_INITIALIZED表示EGL不能初始化。使用函数eglGetError用来获取最近一次调用EGL函数出错的错误代码

> EGLBoolean eglInitialize(EGLDisplay display, // 创建的EGL连接
>                          EGLint *majorVersion, // 返回EGL主板版本号
>                          EGLint *minorVersion); // 返回EGL次版本号

初始化EGL示例

``` cpp
EGLint majorVersion;
EGLint minorVersion;
EGLDisplay display;
display = eglGetDisplay(EGL_DEFAULT_DISPLAY);
if(display == EGL_NO_DISPLAY)
{
    // Unable to open connection to local windowing system
}
if(!eglInitialize(display, &majorVersion, &minorVersion))
{
    // Unable to initialize EGL. Handle and recover
}
```

##### 3.3、确定可用的渲染表面（Surface）的配置。一旦初始化了EGL，就可以确定可用渲染表面的类型和配置了。---==eglChooseChofig()==

一种方式是使用eglGetConfigs函数获取底层窗口系统支持的所有EGL表面配置，然后再使用eglGetConfigAttrib依次查询每个EGLConfig相关的信息，EGLConfig包含了渲染表面的所有信息，包括可用颜色、缓冲区等其他特性。

> EGLBoolean eglGetConfigs(EGLDisplay display, EGLConfig *configs,
> EGLint maxReturnConfigs,EGLint *numConfigs);
> 
> EGLBoolean eglGetConfigAttrib(EGLDisplay display, EGLConfig config,
> EGLint attribute, EGLint *value)

另一种方式是指定我们需要的渲染表面配置，让EGL自己选择一个符合条件的EGLConfig配置。[eglChooseChofig](https://www.khronos.org/registry/egl/sdk/docs/man/html/eglChooseConfig.xhtml)调用成功返回EGL_TRUE，失败时返回EGL_FALSE，如果attribList包含了未定义的EGL属性，或者属性值不合法，EGL代码被设置为EGL_BAD_ATTRIBUTR

> EGLBoolean eglChooseChofig(EGLDispay display, // 创建的和本地窗口系统的连接
>                            const EGLint *attribList, // 指定渲染表面的参数列表，可以为null
>                            EGLConfig *config,   // 调用成功，返会符合条件的EGLConfig列表
>                            EGLint maxReturnConfigs, //最多返回的符合条件的EGLConfig个数
>                            ELGint *numConfigs );  // 实际返回的符合条件的EGLConfig个数

attribList参数在EGL函数中可以为null或者指向一组以EGL_NONE结尾的键对值

``` cpp
// we need this config
EGLint attribList[] ={
EGL_RENDERABLE_TYPE, EGL_OPENGL_ES2_BIT,
EGL_RED_SIZE, 5,
EGL_GREEN_SIZE, 6,
EGL_BLUE_SIZE, 5,
EGL_DEPTH_SIZE, 1,
EGL_NONE
};
const EGLint MaxConfigs = 10;
EGLConfig configs[MaxConfigs]; // We'll only accept 10 configs
EGLint numConfigs;
if(!eglChooseConfig(dpy, attribList, configs, MaxConfigs,
&numConfigs))
{
// Something didn't work … handle error situation
}
else
{
// Everything's okay. Continue to create a rendering surface
}
```
##### 3.4、创建渲染表面---==eglCreateWindowSurface()==
有了符合条件的EGLConfig后，就可以通过eglCreateWindowSurface函数创建渲染表面。使用这个函数的前提是要使用原生窗口系统提供的API创建一个窗口。eglCreateWindowSurface中attribList一般可以使用null即可。函数调用失败会返回EGL_NO_SURFACE，并设置对应的错误码。

> EGLSurface eglCreateWindowSurface(EGLDisplay display,
>                                   EGLConfig config, // 前面选好的可用EGLConfig
>                                   EGLNatvieWindowType window, // 指定原生窗口
>                                   const EGLint *attribList) // 指定窗口属性列表，可以为null，一般指定渲染所用的缓冲区使用但缓冲或者后台缓冲，默认为后者。

创建EGL渲染表面示例

``` cpp
EGLRenderSurface window;
EGLint attribList[] =
{
  EGL_RENDER_BUFFER, EGL_BACK_BUFFER,
  EGL_NONE
);
window = eglCreateWindowSurface(dpy, config, window, attribList);
if(window == EGL_NO_SURFACE)
{
switch(eglGetError())
{
  case EGL_BAD_MATCH:
    // Check window and EGLConfig attributes to determine
    // compatibility, or verify that the EGLConfig
    // supports rendering to a window,
  break;
  case EGL_BAD_CONFIG:
      // Verify that provided EGLConfig is valid
  break;
  case EGL_BAD_NATIVE_WINDOW:
      // Verify that provided EGLNativeWindow is valid
  break;
  case EGL_BAD_ALLOC:
      // Not enough resources available. Handle and recover
  break;
}
}
```
使用eglCreateWindowSurface函数创建在窗口上的渲染表面，此外还可以使用[eglCreatePbufferSurface](https://www.khronos.org/registry/egl/sdk/docs/man/html/eglCreatePbufferSurface.xhtml)创建屏幕外渲染表面（Pixel Buffer 像素缓冲区）。使用Pbuffer一般用于生成纹理贴图，不过该功能已经被FrameBuffer替代了，使用帧缓冲对象的好处是所有的操作都由OpenGL ES来控制。使用Pbuffer的方法和前面创建窗口渲染表面一样，需要改动的地方是在选取EGLConfig时，增加EGL_SURFACE_TYPE参数使其值包含EGL_PBUFFER_BIT。而该参数默认值为EGL_WINDOW_BIT。

> EGLSurface eglCreatePbufferSurface( EGLDisplay display,
>                                    EGLConfig config,
>                                    EGLint const * attrib_list // 指定像素缓冲区属性列表
>                                   );


##### 3.5、创建渲染上下文---==eglCreateContext()==

使用[eglCreateContext](https://www.khronos.org/registry/egl/sdk/docs/man/html/eglCreateContext.xhtml)为当前的渲染API创建EGL渲染上下文，返回一个上下文，当前的渲染API是由函数eglBindAPI设置的。OpenGL ES是一个状态机，用一系列变量描述OpenGL ES当前的状态如何运行，我们通常使用如下途径去更改OpenGL状态：设置选项，操作缓冲。最后，我们使用当前OpenGL上下文来渲染。比如我想告诉OpenGL ES接下来要绘制三角形，可以通过一些上下文变量来改变OpenGL ES的状态，一旦改变了OpenGL ES的状态为绘制三角形，下一个命令就会画出三角形。通过这些状态设置函数就会改变上下文，接下来的操作总会根据当前上下文的状态来执行，除非再次重新改变状态。
> // 设置当前的渲染API
> EGLBoolean eglBindAPI(   EGLenum api //可选 EGL_OPENGL_API,
> EGL_OPENGL_ES_API, or EGL_OPENVG_API );
> 
> EGLContext eglCreateContext(EGLDisplay display, 
>                             EGLConfig config, // 前面选好的可用EGLConfig
>                             EGLContext shareContext, // 允许多个EGLContext共享特定类型的数据，传递EGL_NO_CONTEXT表示不与其他上下文共享资源
>                             const EGLint* attribList // 指定操作的属性列表，只能接受一个属性EGL_CONTEXT_CLIENT_VERSION用来表示使用的OpenGL ES版本
>                            );

创建上下文示例

``` cpp
const ELGint attribList[] = {
EGL_CONTEXT_CLIENT_VERSION, 2,
    EGL_NONE
};
EGLContext context;
context = eglCreateContext(dpy, config, EGL_NO_CONTEXT, attribList);
if(context == EGL_NO_CONTEXT)
{
  EGLError error = eglGetError();
  if(error == EGL_BAD_CONFIG)
  {
    // Handle error and recover
  }
}
```
##### 3.6、指定某个EGLContext为当前上下文---==eglMakeCurrent()==

用[eglMakeCurrent](https://www.khronos.org/registry/egl/sdk/docs/man/html/eglMakeCurrent.xhtml)函数进行当前上下文的绑定。一个程序可能创建多个EGLContext，所以需要关联特定的EGLContext和渲染表面，一般情况下两个EGLSurface参数设置成一样的。

> EGLBoolean eglMakeCurrent(EGLDisplay display,
>                           EGLSurface draw, // EGL绘图表面
>                           EGLSurface read, // EGL读取表面
>                           EGLContext context // 指定连接到该表面的渲染上下文
>                          );

##### 3.7、使用OpenGL相关的API进行==绘制==操作。

##### 3.8、交换EGL的Surface的内部缓冲和EGL创建的和平台无关的窗口diaplay。EGL实际上维护了两个buffer，前台buffer显示的时候，绘制操作会在后台buffer上进行。

> EGLBoolean ==eglSwapBuffers==(EGLDisplay display, // 指定的EGL和本地窗口的连接
>                           EGLSurface surface  // 指定要交换缓冲的EGL绘制表面
>                          );

#### (四)、tritex 运行结果
首先我们来看一下tritex的运行效果，左下角有一个类似矩阵的东东。
![Alt text | center](https://raw.githubusercontent.com/zhoujinjian777/PicGo/master/others/OpenGL-EGL-test-opengl-tritex.png)

##### 4.1、test-opengl-tritex源码分析
首先从main()函数分析，

``` cpp
EGLDisplay eglDisplay;

EGLSurface eglSurface;

EGLContext eglContext;

GLuint texture;

int main(int argc, char **argv)

{
    int q;
    int start, end;
    printf("Initializing EGL...\n");
    WindowSurface windowSurface;
    if(!init_gl_surface(windowSurface))
    {
        printf("GL initialisation failed - exiting\n");
        return 0;
    }
    init_scene();
    create_texture();
    printf("Start test...\n");
    render(argc==2 ? atoi(argv[1]) : ITERATIONS);
    free_gl_surface();
    return 0;
}
```

##### 4.1.1、EGL初始化init_gl_surface()
首先看看函数init_gl_surface()是如何初始化化得。

``` cpp
int init_gl_surface(const WindowSurface& windowSurface)
{
    EGLint numConfigs = 1;
    EGLConfig myConfig = {0};
    EGLint attrib[] =
    {
            EGL_SURFACE_TYPE, EGL_WINDOW_BIT,
            EGL_DEPTH_SIZE,     16,
            EGL_NONE
    };
    //建立本地窗口系统和OpenGL ES的连接
    if ( (eglDisplay = eglGetDisplay(EGL_DEFAULT_DISPLAY)) == EGL_NO_DISPLAY )
    {
        printf("eglGetDisplay failed\n");
        return 0;
    }
    //使用函数eglInitialize进行初始化操作
    if ( eglInitialize(eglDisplay, NULL, NULL) != EGL_TRUE )
    {
        printf("eglInitialize failed\n");
        return 0;
    }
    //根据EGLNativeWindowType 选择Config
    EGLNativeWindowType window = windowSurface.getSurface();
    EGLUtils::selectConfigForNativeWindow(eglDisplay, attrib, window, &myConfig);
    //创建渲染表面
    if ( (eglSurface = eglCreateWindowSurface(eglDisplay, myConfig,
            window, 0)) == EGL_NO_SURFACE )
    {
        printf("eglCreateWindowSurface failed\n");
        return 0;
    }
    //创建渲染上下文
    if ( (eglContext = eglCreateContext(eglDisplay, myConfig, 0, 0)) == EGL_NO_CONTEXT )
    {
        printf("eglCreateContext failed\n");
        return 0;
    }
    //进行当前上下文的绑定
    if ( eglMakeCurrent(eglDisplay, eglSurface, eglSurface, eglContext) != EGL_TRUE )
    {
        printf("eglMakeCurrent failed\n");
        return 0;
    }

    return 1;
```

是不是跟之前使用OpenGL的顺序基本完全一致。

##### 4.1.2、init_scene()

``` cpp
void init_scene(void)
{
    glDisable(GL_DITHER);
    glEnable(GL_CULL_FACE);
    float ratio = 320.0f / 480.0f;
    glViewport(0, 0, 320, 480);//设置视区尺寸
    glMatrixMode(GL_PROJECTION);
    glLoadIdentity();
    glFrustumf(-ratio, ratio, -1, 1, 1, 10);//使用一个透视矩阵乘以当前矩阵
    glMatrixMode(GL_MODELVIEW);
    glLoadIdentity();//重置投影矩阵
    gluLookAt(
            0, 0, 3,  // eye
            0, 0, 0,  // center
            0, 1, 0); // up
    glEnable(GL_TEXTURE_2D);
    glEnableClientState(GL_VERTEX_ARRAY);
    glEnableClientState(GL_TEXTURE_COORD_ARRAY);
}
```
[【Viewport 变换】](http://wiki.jikexueyuan.com/project/opengl-es-guide/viewport.html)
[【投影变换 Projection】](http://wiki.jikexueyuan.com/project/opengl-es-guide/projection.html)
[【OpenGL ES 投影变换 Projection】](https://www.cnblogs.com/ghj1976/archive/2012/04/27/2473624.html)
##### 4.1.3、create_texture()

``` cpp
void create_texture(void)
{
    const unsigned int on = 0xff0000ff;
    const unsigned int off = 0xffffffff;
    const unsigned int pixels[] =
    {
            on, off, on, off, on, off, on, off,
            off, on, off, on, off, on, off, on,
            on, off, on, off, on, off, on, off,
            off, on, off, on, off, on, off, on,
            on, off, on, off, on, off, on, off,
            off, on, off, on, off, on, off, on,
            on, off, on, off, on, off, on, off,
            off, on, off, on, off, on, off, on,
    };
    glGenTextures(1, &texture);
    glBindTexture(GL_TEXTURE_2D, texture);
    glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, 8, 8, 0, GL_RGBA, GL_UNSIGNED_BYTE, pixels);
    glTexParameterx(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
    glTexParameterx(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
    glTexEnvx(GL_TEXTURE_ENV, GL_TEXTURE_ENV_MODE, GL_REPLACE);
}

```
[【OpenGL纹理显示】](https://zhuanlan.zhihu.com/p/24714986)
##### 4.1.4、绘制渲染render()
首先通过glDrawElements()绘制，然后通过==eglSwapBuffers(eglDisplay, eglSurface)==显示到屏幕上。
``` cpp
void render(int quads)
{
    int i, j;

    const GLfloat vertices[] = {
            -1,  -1,  0,
             1,  -1,  0,
             1,   1,  0,
            -1,   1,  0
    };

    const GLfixed texCoords[] = {
            0,            0,
            FIXED_ONE,    0,
            FIXED_ONE,    FIXED_ONE,
            0,            FIXED_ONE
    };

    const GLushort quadIndices[] = { 0, 1, 2,  0, 2, 3 };

    GLushort* indices = (GLushort*)malloc(quads*sizeof(quadIndices));
    for (i=0 ; i<quads ; i++)
        memcpy(indices+(sizeof(quadIndices)/sizeof(indices[0]))*i, quadIndices, sizeof(quadIndices));

    glVertexPointer(3, GL_FLOAT, 0, vertices);
    glTexCoordPointer(2, GL_FIXED, 0, texCoords);

    // make sure to do a couple eglSwapBuffers to make sure there are
    // no problems with the very first ones (who knows)
    glClearColor(0.4, 0.4, 0.4, 0.4);
    glClear(GL_DEPTH_BUFFER_BIT | GL_COLOR_BUFFER_BIT);
    eglSwapBuffers(eglDisplay, eglSurface);
    glClearColor(0.6, 0.6, 0.6, 0.6);
    glClear(GL_DEPTH_BUFFER_BIT | GL_COLOR_BUFFER_BIT);
    eglSwapBuffers(eglDisplay, eglSurface);
    glClearColor(1.0, 1.0, 1.0, 1.0);

    for (j=0 ; j<10 ; j++) {
        printf("loop %d / 10 (%d quads / loop)\n", j, quads);

        int nelem = sizeof(quadIndices)/sizeof(quadIndices[0]);
        glClear(GL_DEPTH_BUFFER_BIT | GL_COLOR_BUFFER_BIT);
        glDrawElements(GL_TRIANGLES, nelem*quads, GL_UNSIGNED_SHORT, indices);
        eglSwapBuffers(eglDisplay, eglSurface);
    }

    free(indices);
}
```

#### (五)、textures实例 test-opengl-textures
这个纹理的例子比较明确，可以先看一下效果图，使用基本一致，就不分析了。
![Alt text | center](https://raw.githubusercontent.com/zhoujinjian777/PicGo/master/others/OpenGL-EGL-test-opengl-textures.png)

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
     GLint dim = w<h ? w : h;


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
     uint8_t t8[]  = { 
             0x00, 0x55, 0x00, 0x55, 
             0xAA, 0xFF, 0xAA, 0xFF,
             0x00, 0x55, 0x00, 0x55, 
             0xAA, 0xFF, 0xAA, 0xFF  };

     uint16_t t16[]  = { 
             0x0000, 0x5555, 0x0000, 0x5555, 
             0xAAAA, 0xFFFF, 0xAAAA, 0xFFFF,
             0x0000, 0x5555, 0x0000, 0x5555, 
             0xAAAA, 0xFFFF, 0xAAAA, 0xFFFF  };

     uint16_t t5551[]  = { 
             0x0000, 0xFFFF, 0x0000, 0xFFFF, 
             0xFFFF, 0x0000, 0xFFFF, 0x0000,
             0x0000, 0xFFFF, 0x0000, 0xFFFF, 
             0xFFFF, 0x0000, 0xFFFF, 0x0000  };

     uint32_t t32[]  = { 
             0xFF000000, 0xFF0000FF, 0xFF00FF00, 0xFFFF0000, 
             0xFF00FF00, 0xFFFF0000, 0xFF000000, 0xFF0000FF, 
             0xFF00FFFF, 0xFF00FF00, 0x00FF00FF, 0xFFFFFF00, 
             0xFF000000, 0xFFFF00FF, 0xFF00FFFF, 0xFFFFFFFF
     };


     glClear(GL_COLOR_BUFFER_BIT);
     glTexImage2D(GL_TEXTURE_2D, 0, GL_LUMINANCE, 4, 4, 0, GL_LUMINANCE, GL_UNSIGNED_BYTE, t8);
     glDrawTexiOES(0, 0, 0, dim/2, dim/2);

     glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, 4, 4, 0, GL_RGB, GL_UNSIGNED_SHORT_5_6_5, t16);
     glDrawTexiOES(dim/2, 0, 0, dim/2, dim/2);

     glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, 4, 4, 0, GL_RGBA, GL_UNSIGNED_SHORT_4_4_4_4, t16);
     glDrawTexiOES(0, dim/2, 0, dim/2, dim/2);

     glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, 4, 4, 0, GL_RGBA, GL_UNSIGNED_BYTE, t32);
     glDrawTexiOES(dim/2, dim/2, 0, dim/2, dim/2);

     eglSwapBuffers(dpy, surface);

     sleep(2);      // so you have a chance to admire it
     return 0;
}

```

#### (六)、fillrate 运行结果
![Alt text | center](https://raw.githubusercontent.com/zhoujinjian777/PicGo/master/others/OpenGL-EGL-test-opengl-fillrate.png)

##### 6.1、fillrate 源码

``` cpp
#define LOG_TAG "fillrate"

#include <stdlib.h>
#include <stdio.h>

#include <EGL/egl.h>
#include <GLES/gl.h>
#include <GLES/glext.h>

#include <utils/StopWatch.h>
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
     
     printf("w=%d, h=%d\n", w, h);
     
     glBindTexture(GL_TEXTURE_2D, 0);
     glTexParameterx(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
     glTexParameterx(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
     glTexParameterx(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT);
     glTexParameterx(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_REPEAT);
     glTexEnvx(GL_TEXTURE_ENV, GL_TEXTURE_ENV_MODE, GL_REPLACE);
     glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);
     glDisable(GL_DITHER);
     glEnable(GL_BLEND);
     glEnable(GL_TEXTURE_2D);
     glColor4f(1,1,1,1);

     uint32_t* t32 = (uint32_t*)malloc(512*512*4); 
     for (int y=0 ; y<512 ; y++) {
         for (int x=0 ; x<512 ; x++) {
             int u = x-256;
             int v = y-256;
             if (u*u+v*v < 256*256) {
                 t32[x+y*512] = 0x10FFFFFF;
             } else {
                 t32[x+y*512] = 0x20FF0000;
             }
         }
     }

     const GLfloat fh = h;
     const GLfloat fw = w;
     const GLfloat vertices[4][2] = {
             { 0,   0  },
             { 0,   fh },
             { fw,  fh },
             { fw,  0  }
     };

     const GLfloat texCoords[4][2] = {
             { 0,  0 },
             { 0,  1 },
             { 1,  1 },
             { 1,  0 }
     };

     glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, 512, 512, 0, GL_RGBA, GL_UNSIGNED_BYTE, t32);

     glViewport(0, 0, w, h);
     glMatrixMode(GL_PROJECTION);
     glLoadIdentity();
     glOrthof(0, w, 0, h, 0, 1);

     glEnableClientState(GL_VERTEX_ARRAY);
     glEnableClientState(GL_TEXTURE_COORD_ARRAY);
     glVertexPointer(2, GL_FLOAT, 0, vertices);
     glTexCoordPointer(2, GL_FLOAT, 0, texCoords);

     eglSwapInterval(dpy, 1);

     glClearColor(1,0,0,0);
     glClear(GL_COLOR_BUFFER_BIT);
     glDrawArrays(GL_TRIANGLE_FAN, 0, 4); 
     eglSwapBuffers(dpy, surface);
     

     nsecs_t times[32];

     for (int c=1 ; c<32 ; c++) {
         glClear(GL_COLOR_BUFFER_BIT);
         for (int i=0 ; i<c ; i++) {
             glDrawArrays(GL_TRIANGLE_FAN, 0, 4); 
         }
         eglSwapBuffers(dpy, surface);
     }


     //     for (int c=31 ; c>=1 ; c--) {
     int j=0;
     for (int c=1 ; c<32 ; c++) {
         glClear(GL_COLOR_BUFFER_BIT);
         nsecs_t now = systemTime();
         for (int i=0 ; i<c ; i++) {
             glDrawArrays(GL_TRIANGLE_FAN, 0, 4); 
         }
         eglSwapBuffers(dpy, surface);
         nsecs_t t = systemTime() - now;
         times[j++] = t;
     }

     for (int c=1, j=0 ; c<32 ; c++, j++) {
         nsecs_t t = times[j];
         printf("%ld\t%d\t%f\n", t, c, (double(t)/c)/1000000.0);
     }


       
     eglTerminate(dpy);
     
     return 0;
}

```

#### (七)、filter运行结果
![Alt text | center](https://raw.githubusercontent.com/zhoujinjian777/PicGo/master/others/OpenGL-EGL-test-opengl-filter.png)


##### 7.1、filter源码

``` cpp
#include <stdlib.h>
#include <stdio.h>

#include <EGL/egl.h>
#include <GLES/gl.h>
#include <GLES/glext.h>

#include <WindowSurface.h>
#include <EGLUtils.h>

using namespace android;

#define USE_DRAW_TEXTURE 1

int main(int argc, char** argv)
{
    if (argc!=2 && argc!=3) {
        printf("usage: %s <0-6> [pbuffer]\n", argv[0]);
        return 0;
    }
    
    const int test = atoi(argv[1]);
    int usePbuffer = argc==3 && !strcmp(argv[2], "pbuffer");
    EGLint s_configAttribs[] = {
         EGL_SURFACE_TYPE, EGL_PBUFFER_BIT|EGL_WINDOW_BIT,
         EGL_RED_SIZE,       5,
         EGL_GREEN_SIZE,     6,
         EGL_BLUE_SIZE,      5,
         EGL_NONE
     };
     
     EGLint numConfigs = -1;
     EGLint majorVersion;
     EGLint minorVersion;
     EGLConfig config;
     EGLContext context;
     EGLSurface surface;
     EGLint w, h;
     
     EGLDisplay dpy;

     EGLNativeWindowType window = 0;
     WindowSurface* windowSurface = NULL;
     if (!usePbuffer) {
         windowSurface = new WindowSurface();
         window = windowSurface->getSurface();
     }
     
     dpy = eglGetDisplay(EGL_DEFAULT_DISPLAY);
     eglInitialize(dpy, &majorVersion, &minorVersion);
     if (!usePbuffer) {
         EGLUtils::selectConfigForNativeWindow(
                 dpy, s_configAttribs, window, &config);
         surface = eglCreateWindowSurface(dpy, config, window, NULL);
     } else {
         printf("using pbuffer\n");
         eglChooseConfig(dpy, s_configAttribs, &config, 1, &numConfigs);
         EGLint attribs[] = { EGL_WIDTH, 320, EGL_HEIGHT, 480, EGL_NONE };
         surface = eglCreatePbufferSurface(dpy, config, attribs);
         if (surface == EGL_NO_SURFACE) {
             printf("eglCreatePbufferSurface error %x\n", eglGetError());
         }
     }
     context = eglCreateContext(dpy, config, NULL, NULL);
     eglMakeCurrent(dpy, surface, surface, context);   
     eglQuerySurface(dpy, surface, EGL_WIDTH, &w);
     eglQuerySurface(dpy, surface, EGL_HEIGHT, &h);
     GLint dim = w<h ? w : h;

     glViewport(0, 0, w, h);
     glMatrixMode(GL_PROJECTION);
     glLoadIdentity();
     glOrthof(0, w, 0, h, 0, 1);

     glClearColor(0,0,0,0);
     glClear(GL_COLOR_BUFFER_BIT);

     GLint crop[4] = { 0, 4, 4, -4 };
     glBindTexture(GL_TEXTURE_2D, 0);
     glTexParameteriv(GL_TEXTURE_2D, GL_TEXTURE_CROP_RECT_OES, crop);
     glTexParameterx(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
     glTexParameterx(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
     glTexEnvx(GL_TEXTURE_ENV, GL_TEXTURE_ENV_MODE, GL_REPLACE);
     glEnable(GL_TEXTURE_2D);
     glColor4f(1,1,1,1);

     // packing is always 4
     uint8_t t8[]  = { 
             0x00, 0x55, 0x00, 0x55, 
             0xAA, 0xFF, 0xAA, 0xFF,
             0x00, 0x55, 0x00, 0x55, 
             0xAA, 0xFF, 0xAA, 0xFF  };

     uint16_t t16[]  = { 
             0x0000, 0x5555, 0x0000, 0x5555, 
             0xAAAA, 0xFFFF, 0xAAAA, 0xFFFF,
             0x0000, 0x5555, 0x0000, 0x5555, 
             0xAAAA, 0xFFFF, 0xAAAA, 0xFFFF  };

     uint16_t t5551[]  = { 
             0x0000, 0xFFFF, 0x0000, 0xFFFF, 
             0xFFFF, 0x0000, 0xFFFF, 0x0000,
             0x0000, 0xFFFF, 0x0000, 0xFFFF, 
             0xFFFF, 0x0000, 0xFFFF, 0x0000  };

     uint32_t t32[]  = { 
             0xFF000000, 0xFF0000FF, 0xFF00FF00, 0xFFFF0000, 
             0xFF00FF00, 0xFFFF0000, 0xFF000000, 0xFF0000FF, 
             0xFF00FFFF, 0xFF00FF00, 0x00FF00FF, 0xFFFFFF00, 
             0xFF000000, 0xFFFF00FF, 0xFF00FFFF, 0xFFFFFFFF
     };

     switch(test) 
     {
     case 1:
         glTexImage2D(GL_TEXTURE_2D, 0, GL_LUMINANCE,
                 4, 4, 0, GL_LUMINANCE, GL_UNSIGNED_BYTE, t8);
         break;
     case 2:
         glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB,
                 4, 4, 0, GL_RGB, GL_UNSIGNED_SHORT_5_6_5, t16);
         break;
     case 3:
         glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA,
                 4, 4, 0, GL_RGBA, GL_UNSIGNED_SHORT_4_4_4_4, t16);
         break;
     case 4:
         glTexImage2D(GL_TEXTURE_2D, 0, GL_LUMINANCE_ALPHA,
                 4, 4, 0, GL_LUMINANCE_ALPHA, GL_UNSIGNED_BYTE, t16);
         break;
     case 5:
         glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA,
                 4, 4, 0, GL_RGBA, GL_UNSIGNED_SHORT_5_5_5_1, t5551);
         break;
     case 6:
         glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA,
                 4, 4, 0, GL_RGBA, GL_UNSIGNED_BYTE, t32);
         break;
     }

     //glDrawTexiOES(0, 0, 0, dim, dim);
     
     const GLfloat fdim = dim;
     const GLfloat vertices[4][2] = {
             { 0,     0    },
             { 0,     fdim },
             { fdim,  fdim },
             { fdim,  0    }
     };

     const GLfloat texCoords[4][2] = {
             { 0,  0 },
             { 0,  1 },
             { 1,  1 },
             { 1,  0 }
     };
     
     if (!usePbuffer) {
         eglSwapBuffers(dpy, surface);
     }
     
     glMatrixMode(GL_MODELVIEW);
     glScissor(0,dim,dim,h-dim);
     glDisable(GL_SCISSOR_TEST);
     
     for (int y=0 ; y<dim ; y++) {
         //glDisable(GL_SCISSOR_TEST);
         glClear(GL_COLOR_BUFFER_BIT);

         //glEnable(GL_SCISSOR_TEST);

#if USE_DRAW_TEXTURE && GL_OES_draw_texture
         glDrawTexiOES(0, y, 1, dim, dim);
#else
         glLoadIdentity();
         glTranslatef(0, y, 0);
         glEnableClientState(GL_VERTEX_ARRAY);
         glEnableClientState(GL_TEXTURE_COORD_ARRAY);
         glVertexPointer(2, GL_FLOAT, 0, vertices);
         glTexCoordPointer(2, GL_FLOAT, 0, texCoords);
         glDrawArrays(GL_TRIANGLE_FAN, 0, 4);
#endif

         if (!usePbuffer) {
             eglSwapBuffers(dpy, surface);
         } else {
             glFinish();
         }
     }

     eglTerminate(dpy);
     delete windowSurface;
     return 0;
}

```

#### (八)、finish运行结果
![Alt text | center](https://raw.githubusercontent.com/zhoujinjian777/PicGo/master/others/OpenGL-EGL-test-opengl-finish.png)


##### 8.1、finish源码
``` cpp
#include <stdlib.h>
#include <stdio.h>
#include <time.h>
#include <sched.h>
#include <sys/resource.h>

#include <EGL/egl.h>
#include <GLES/gl.h>
#include <GLES/glext.h>

#include <utils/Timers.h>

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
     GLint dim = w<h ? w : h;

     glBindTexture(GL_TEXTURE_2D, 0);
     glTexParameterx(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
     glTexParameterx(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
     glTexEnvx(GL_TEXTURE_ENV, GL_TEXTURE_ENV_MODE, GL_REPLACE);
     glEnable(GL_TEXTURE_2D);
     glColor4f(1,1,1,1);
     glDisable(GL_DITHER);
     glShadeModel(GL_FLAT);

     long long now, t;
     int i;

     char* texels = (char*)malloc(512*512*2);
     memset(texels,0xFF,512*512*2);
     
     glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB,
             512, 512, 0, GL_RGB, GL_UNSIGNED_SHORT_5_6_5, texels);

     char* dst = (char*)malloc(320*480*2);
     memset(dst, 0, 320*480*2);
     printf("307200 bytes memcpy\n");
     for (i=0 ; i<4 ; i++) {
         now = systemTime();
         memcpy(dst, texels, 320*480*2);
         t = systemTime();
         printf("memcpy() time = %llu us\n", (t-now)/1000);
         fflush(stdout);
     }
     free(dst);

     free(texels);

     setpriority(PRIO_PROCESS, 0, -20);
     
     printf("512x512 unmodified texture, 512x512 blit:\n");
     glClear(GL_COLOR_BUFFER_BIT);
     for (i=0 ; i<4 ; i++) {
         GLint crop[4] = { 0, 512, 512, -512 };
         glTexParameteriv(GL_TEXTURE_2D, GL_TEXTURE_CROP_RECT_OES, crop);
         now = systemTime();
         glDrawTexiOES(0, 0, 0, 512, 512);
         glFinish();
         t = systemTime();
         printf("glFinish() time = %llu us\n", (t-now)/1000);
         fflush(stdout);
         eglSwapBuffers(dpy, surface);
     }
     
     printf("512x512 unmodified texture, 1x1 blit:\n");
     glClear(GL_COLOR_BUFFER_BIT);
     for (i=0 ; i<4 ; i++) {
         GLint crop[4] = { 0, 1, 1, -1 };
         glTexParameteriv(GL_TEXTURE_2D, GL_TEXTURE_CROP_RECT_OES, crop);
         now = systemTime();
         glDrawTexiOES(0, 0, 0, 1, 1);
         glFinish();
         t = systemTime();
         printf("glFinish() time = %llu us\n", (t-now)/1000);
         fflush(stdout);
         eglSwapBuffers(dpy, surface);
     }
     
     printf("512x512 unmodified texture, 512x512 blit (x2):\n");
     glClear(GL_COLOR_BUFFER_BIT);
     for (i=0 ; i<4 ; i++) {
         GLint crop[4] = { 0, 512, 512, -512 };
         glTexParameteriv(GL_TEXTURE_2D, GL_TEXTURE_CROP_RECT_OES, crop);
         now = systemTime();
         glDrawTexiOES(0, 0, 0, 512, 512);
         glDrawTexiOES(0, 0, 0, 512, 512);
         glFinish();
         t = systemTime();
         printf("glFinish() time = %llu us\n", (t-now)/1000);
         fflush(stdout);
         eglSwapBuffers(dpy, surface);
     }

     printf("512x512 unmodified texture, 1x1 blit (x2):\n");
     glClear(GL_COLOR_BUFFER_BIT);
     for (i=0 ; i<4 ; i++) {
         GLint crop[4] = { 0, 1, 1, -1 };
         glTexParameteriv(GL_TEXTURE_2D, GL_TEXTURE_CROP_RECT_OES, crop);
         now = systemTime();
         glDrawTexiOES(0, 0, 0, 1, 1);
         glDrawTexiOES(0, 0, 0, 1, 1);
         glFinish();
         t = systemTime();
         printf("glFinish() time = %llu us\n", (t-now)/1000);
         fflush(stdout);
         eglSwapBuffers(dpy, surface);
     }

     
     printf("512x512 (1x1 texel MODIFIED texture), 512x512 blit:\n");
     glClear(GL_COLOR_BUFFER_BIT);
     for (i=0 ; i<4 ; i++) {
         uint16_t green = 0x7E0;
         GLint crop[4] = { 0, 512, 512, -512 };
         glTexParameteriv(GL_TEXTURE_2D, GL_TEXTURE_CROP_RECT_OES, crop);
         glTexSubImage2D(GL_TEXTURE_2D, 0, 0, 0, 1, 1, GL_RGB, GL_UNSIGNED_SHORT_5_6_5, &green);
         now = systemTime();
         glDrawTexiOES(0, 0, 0, 512, 512);
         glFinish();
         t = systemTime();
         printf("glFinish() time = %llu us\n", (t-now)/1000);
         fflush(stdout);
         eglSwapBuffers(dpy, surface);
     }


     int16_t texel = 0xF800;
     glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB,
             1, 1, 0, GL_RGB, GL_UNSIGNED_SHORT_5_6_5, &texel);

     printf("1x1 unmodified texture, 1x1 blit:\n");
     glClear(GL_COLOR_BUFFER_BIT);
     for (i=0 ; i<4 ; i++) {
         GLint crop[4] = { 0, 1, 1, -1 };
         glTexParameteriv(GL_TEXTURE_2D, GL_TEXTURE_CROP_RECT_OES, crop);
         now = systemTime();
         glDrawTexiOES(0, 0, 0, 1, 1);
         glFinish();
         t = systemTime();
         printf("glFinish() time = %llu us\n", (t-now)/1000);
         eglSwapBuffers(dpy, surface);
     }

     printf("1x1 unmodified texture, 512x512 blit:\n");
     glClear(GL_COLOR_BUFFER_BIT);
     for (i=0 ; i<4 ; i++) {
         GLint crop[4] = { 0, 1, 1, -1 };
         glTexParameteriv(GL_TEXTURE_2D, GL_TEXTURE_CROP_RECT_OES, crop);
         now = systemTime();
         glDrawTexiOES(0, 0, 0, 512, 512);
         glFinish();
         t = systemTime();
         printf("glFinish() time = %llu us\n", (t-now)/1000);
         fflush(stdout);
         eglSwapBuffers(dpy, surface);
     }

     printf("1x1 (1x1 texel MODIFIED texture), 512x512 blit:\n");
     glClear(GL_COLOR_BUFFER_BIT);
     for (i=0 ; i<4 ; i++) {
         uint16_t green = 0x7E0;
         GLint crop[4] = { 0, 1, 1, -1 };
         glTexParameteriv(GL_TEXTURE_2D, GL_TEXTURE_CROP_RECT_OES, crop);
         glTexSubImage2D(GL_TEXTURE_2D, 0, 0, 0, 1, 1, GL_RGB, GL_UNSIGNED_SHORT_5_6_5, &green);
         now = systemTime();
         glDrawTexiOES(0, 0, 0, 1, 1);
         glFinish();
         t = systemTime();
         printf("glFinish() time = %llu us\n", (t-now)/1000);
         fflush(stdout);
         eglSwapBuffers(dpy, surface);
     }

     return 0;
}

```

#### (九)、参考文档：
[【EGL函数API文档】](https://www.zybuluo.com/cxm-2016/note/572030)
[【OpenGL ES EGL介绍】](https://blog.csdn.net/cauchyweierstrass/article/details/53189449)





