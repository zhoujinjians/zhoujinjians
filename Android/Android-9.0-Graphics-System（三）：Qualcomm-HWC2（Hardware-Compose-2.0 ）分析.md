---
title: Android P Graphics System（三）：Qualcomm HWC2（Hardware Composer 2.0 ）分析
cover: https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/hexo.themes/bing-wallpaper-2018.04.38.jpg
categories:
  - Graphics
tags:
  - Android
  - Graphics
toc: true
abbrlink: 20190708
date: 2019-07-08 09:25:00
---


--------------------------------------------------------------------------------

注：文章都是通过阅读各位前辈总结的资料 Android 9.0 && Linux（Kernel 3.18）Qualcomm平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除（◕‿◕），转载请注明出处（©Qualcomm ©Android @Linux 版权所有），谢谢。
（==**文章基于 Kernel-3.18**==）&&（==**文章基于 Android 9.0**==）

[【开发板 Intrinsyc Open-Q™ 820 µSOM Development Kit】](https://www.intrinsyc.com/snapdragon-embedded-development-kits/open-q-820-usom-development-kit/)
[【开发板 Android 9.0 && Linux（Kernel 3.18）源码链接】](https://gitlab.com/iizhoujinjian/apq8096_la.um.7.5.r1-03100-8x96.0_p_v5.0)

正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢！！！

[（1）【Android P 图形显示系统（一）硬件合成HWC2】](https://www.jianshu.com/p/824a9ddf68b9) 
[（2）【AndroidO Treble架构下Hal进程启动及HIDL服务注册过程】](https://blog.csdn.net/yangwen123/article/details/79854267) 
[（3）【Android O Treble 架构 - HIDL源代码分析】](http://charlesvincent.cc/2018/09/28/Android%20O%20Treble%20架构%20-%20HIDL源代码分析/)
[（4）【AndroidO Treble架构下HIDL IComposer服务查询过程】](https://blog.csdn.net/yangwen123/article/details/79868548)

--------------------------------------------------------------------------------
==源码（部分）==：

>SurfaceFlinger

-  frameworks/native/services/surfaceflinger


>HWC2

- hardware/qcom/display/sdm/libs/hwc2/


--------------------------------------------------------------------------------


#### （一）、HWC2介绍

##### 1.1.0 、HWC2概述

Android 7.0 包含新版本的 HWC (HWC2)，Android需要自行配置，到Android 8.0，HWC2正式开启，且版本升级为2.1版本。HWC2是 SurfaceFlinger 用来与专门的窗口合成硬件进行通信。SurfaceFlinger 包含使用 3D 图形处理器 (GPU) 执行窗口合成任务的备用路径，但由于以下几个原因，此路径并不理想：

- 通常，GPU 未针对此用例进行过优化，因此能耗可能要大于执行合成所需的能耗。
- 每次 SurfaceFlinger 使用 GPU 进行合成时，应用都无法使用处理器进行自我渲染，因此应尽可能使用专门的硬件而不是 GPU 进行合成。

下面是GPU和HWC两种方式的优劣对比：

| 合成类型 |    耗电情况 | 性能情况  | Alpha处理 | DRM内容处理 | 其他限制 |
| :--------: | :--------:| :-------: | :-------: | :-------: | :-------: |
| Device合成（HWC） |耗电低 |  性能高  |很多Vendor的HWC不支持Alpha的处理和合成|基本都能访问DRM内容|能合成的Surface层数有限，对每种Surface类型处理层数有限|
| Client合成（GPU） |  耗电高 |  性能低  |能处理每个像素的Alpha及每个Layear的Alpha | 早期版本GPU不能访问DRM的内容 |   目前的处理层数没有限制 |

所以，HWC的设计最好遵循一些基本的规则～

##### 1.2.0 、HWC常规准则
由于 Hardware Composer 抽象层后的物理显示设备硬件可因设备而异，因此很难就具体功能提供建议。一般来说，请遵循以下准则：

- HWC 应至少支持 4 个叠加层（状态栏、系统栏、应用和壁纸/背景）。
- 层可以大于屏幕，因此 HWC 应能处理大于显示屏的层（例如壁纸）。
- 应同时支持预乘每像素 Alpha 混合和每平面 Alpha 混合。
- HWC 应能处理 GPU、相机和视频解码器生成的相同缓冲区，因此支持以下某些属性很有帮助：

   - RGBA 打包顺序
   - YUV 格式
   - Tiling, swizzling和步幅属性

- 为了支持受保护的内容，必须提供受保护视频播放的硬件路径。

Tiling，翻译过来就没有原文的意味了，说白了，就是将image进行切割，切成MxN的小块，最后用的时候，再将这些小块拼接起来，就像铺瓷砖一样。
swizzling，比Tiling难理解点，它是一种拌和技术，这是向量的单元可以被任意地重排或重复，见过的hwc代码写的都比较隐蔽，没有见多处理的地方。
HWC专注于优化，智能地选择要发送到叠加硬件的 Surface，以最大限度减轻 GPU 的负载。另一种优化是检测屏幕是否正在更新；如果不是，则将合成委托给 OpenGL 而不是 HWC，以节省电量。当屏幕再次更新时，继续将合成分载到 HWC。
为常见用例做准备，如：

- 纵向和横向模式下的全屏游戏
- 带有字幕和播放控件的全屏视频
- 主屏幕（合成状态栏、系统栏、应用窗口和动态壁纸）
- 受保护的视频播放
- 多显示设备支持


##### 1.3.0 、HWC2 框架
从Android 8.0开始的Treble项目，对Android的架构做了重大的调整，让制造商以更低的成本更轻松、更快速地将设备更新到新版 Android 系统。这就对 HAL 层有了很大的调整，利用提供给Vendor的接口，将Vendor的实现和Android上层分离开来。
这样的架构让HWC的架构也变的复杂，HWC属于Binderized的HAL类型。Binderized类型的HAL，将上层Androd和底层HAL分别采用两个不用的进程实现，中间采用Binder进行通信，为了和前面的Binder进行区别，这里采用HwBinder。
因此，我们可以将HWC再进行划分，可以分为下面这几个部分，如下图：
![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/display.system/Android.PG3.HW2_Pipeline.png)

![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/display.system/Android.PG3.HW2_Arc.png)

- **Client端**
Client也就是SurfaceFlinger，不过SurfaceFlinger采用前后端的设计，以后和HWC相关的逻辑应该都会放到SurfaceFlinger后端也就是SurfaceFlingerBE中。代码位置：

> frameworks/native/services/surfaceflinger


- **HWC2 Client端**
这一部分属于SurfaceFlinger进程，其直接通过Binder通信，和HWC2的HAL Server交互。这部分的代码也在SurfaceFlinger进程中，但是采用Hwc2的命名空间。
- **HWC Server端**
这一部分还是属于Android的系统，这里将建立一个进程，实现HWC的服务端，Server端再调底层Vendor的具体实现。并且，对于底层合成的实现不同，这里会做一些适配，适配HWC1.x，和FrameBuffer的实现。这部分包含三部分：接口，实现和服务，以动态库的形式存在：

> 代码位置： hardware/interfaces/graphics/composer/2.1/default
> 动态库：
> android.hardware.graphics.composer@2.1.so
> android.hardware.graphics.composer@2.1-impl.so
> android.hardware.graphics.composer@2.1-service.so

- **HWC Vendor的实现**
这部分是HWC的具体实现，这部分的实现由硬件厂商完成。比如高通平台，代码位置一般为：

> G:\android9.0\hardware\qcom\display\sdm\libs
> > core
> hwc
> hwc2
> utils


需要注意的是，HWC必须采用Binderized HAL模式，但是，并没有要求一定要实现HWC2的HAL版本。HWC2的实现需要配置，以Android 9.0为例：

- 添加宏定义 TARGET_USES_HWC2
- 编译打包HWC2相关的so库
- SeLinux相关的权限添加
- 配置manifest.xml

``` cpp
     <hal format="hidl">
        <name>android.hardware.graphics.composer</name>
        <transport>hwbinder</transport>
        <version>2.1</version>
        <interface>
            <name>IComposer</name>
            <instance>default</instance>
        </interface>
    </hal>
```


#### （二）、HWC2合成服务(Composer Service)启动流程
Android Framework进程和Hal分离，每个Hal独立运行在自己的进程地址空间，那么这些Hal进程是如何启动的呢？本文以composer hal为例展开分析。

在以下路径有composer hal的rc启动脚本：

``` cpp
hardware/interfaces/graphics/composer/2.1/default/android.hardware.graphics.composer@2.1-service.rc

service hwcomposer-2-1 /vendor/bin/hw/android.hardware.graphics.composer@2.1-service
    class hal animation
    user system
    group graphics drmrpc
    capabilities SYS_NICE
    onrestart restart surfaceflinger
```

编译后，会将该脚本文件copy到vendor/etc/init目录，在开机时，init进程会读取并解析这个脚本，然后启动android.hardware.graphics.composer@2.1-service进程：

``` cpp
msm8996:/ $ ps -A | grep "composer"
system         345     1   68600   5964 0                   0 S android.hardware.graphics.composer@2.1-service
```

该进程的可执行文件是：vendor/bin/hw/android.hardware.graphics.composer@2.1-service，

该可执行文件对应的源码为：hardware/interfaces/graphics/composer/2.1/default/service.cpp

##### 2.1.0 、Composer Hal启动过程

``` cpp
hardware/interfaces/graphics/composer/2.1/default/service.cpp
int main() {
    // the conventional HAL might start binder services
    android::ProcessState::initWithDriver("/dev/vndbinder");
    android::ProcessState::self()->setThreadPoolMaxThreadCount(4);
    android::ProcessState::self()->startThreadPool();

    // same as SF main thread
    struct sched_param param = {0};
    param.sched_priority = 2;
    if (sched_setscheduler(0, SCHED_FIFO | SCHED_RESET_ON_FORK,
                &param) != 0) {
        ALOGE("Couldn't set SCHED_FIFO: %d", errno);
    }
#ifdef ARCH_ARM_32
    android::hardware::ProcessState::initWithMmapSize((size_t)(32768));
#endif
    return defaultPassthroughServiceImplementation<IComposer>(4);
}

```
在Treble架构下，存在了3个binder设备，分别是/dev/binder、/dev/vndbinder、/dev/hwbinder，上层需要通过binder库来访问这些binder设备，而/dev/binder和/dev/vndbinder都是由libbinder来访问，因此需要指定打开的binder设备。
android::ProcessState::initWithDriver("/dev/vndbinder");
这句说明composer hal通过vndbinder来通信的，接下来就是设置binder线程个数为4，并启动binder线程池，然后调用
defaultPassthroughServiceImplementation<IComposer>(4)
完成composer hal的启动。

``` cpp
system\libhidl\transport\include\hidl\LegacySupport.h

/**
 * Registers passthrough service implementation.
 */
template<class Interface>
__attribute__((warn_unused_result))
status_t registerPassthroughServiceImplementation(
        std::string name = "default") {
    sp<Interface> service = Interface::getService(name, true /* getStub */);//从当前进程空间中拿到IComposer接口类对象

    if (service == nullptr) {
        ALOGE("Could not get passthrough implementation for %s/%s.",
            Interface::descriptor, name.c_str());
        return EXIT_FAILURE;
    }

    LOG_FATAL_IF(service->isRemote(), "Implementation of %s/%s is remote!",
            Interface::descriptor, name.c_str());

    status_t status = service->registerAsService(name);//将IComposer注册到hwservicemanager中

    if (status == OK) {
        ALOGI("Registration complete for %s/%s.",
            Interface::descriptor, name.c_str());
    } else {
        ALOGE("Could not register service %s/%s (%d).",
            Interface::descriptor, name.c_str(), status);
    }

    return status;
}

/**
 * Creates default passthrough service implementation. This method never returns.
 *
 * Return value is exit status.
 */
template<class Interface>
__attribute__((warn_unused_result))
status_t defaultPassthroughServiceImplementation(std::string name,
                                            size_t maxThreads = 1) {
    configureRpcThreadpool(maxThreads, true);
    status_t result = registerPassthroughServiceImplementation<Interface>(name);
    ......
    joinRpcThreadpool();
    return UNKNOWN_ERROR;
}
```
![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/display.system/Android.PG3.HW2_Composer_registerAsService.png)


##### 2.2.0 、Hal进程获取IComposer类对象
在composer hal进程启动时，首先调用IComposer 的getService(“default”,true)来获取IComposer的类对象。

``` cpp
android\out\soong\.intermediates\hardware\interfaces\graphics\composer\2.1
android.hardware.graphics.composer@2.1_genc++\gen\android\hardware\graphics\composer\2.1\ComposerAll.cpp

// static
::android::sp<IComposer> IComposer::getService(const std::string &serviceName, const bool getStub) {
    using ::android::hardware::defaultServiceManager;
    using ::android::hardware::details::waitForHwService;
    using ::android::hardware::getPassthroughServiceManager;
    using ::android::hardware::Return;
    using ::android::sp;
    using Transport = ::android::hidl::manager::V1_0::IServiceManager::Transport;

    sp<IComposer> iface = nullptr;
     //获取hwservicemanager的代理
    const sp<::android::hidl::manager::V1_0::IServiceManager> sm = defaultServiceManager();
    ......
    //查询IComposer的Transport
    Return<Transport> transportRet = sm->getTransport(IComposer::descriptor, serviceName);

    ......
    Transport transport = transportRet;
    const bool vintfHwbinder = (transport == Transport::HWBINDER);
    const bool vintfPassthru = (transport == Transport::PASSTHROUGH);

    #ifdef __ANDROID_TREBLE__

    #ifdef __ANDROID_DEBUGGABLE__
    const char* env = std::getenv("TREBLE_TESTING_OVERRIDE");
    const bool trebleTestingOverride =  env && !strcmp(env, "true");
    const bool vintfLegacy = (transport == Transport::EMPTY) && trebleTestingOverride;
    #else // __ANDROID_TREBLE__ but not __ANDROID_DEBUGGABLE__
    const bool trebleTestingOverride = false;
    const bool vintfLegacy = false;
    #endif // __ANDROID_DEBUGGABLE__

    #else // not __ANDROID_TREBLE__
    const char* env = std::getenv("TREBLE_TESTING_OVERRIDE");
    const bool trebleTestingOverride =  env && !strcmp(env, "true");
    const bool vintfLegacy = (transport == Transport::EMPTY);

    #endif // __ANDROID_TREBLE__
    //hwbinder方式下获取IComposer对象
    for (int tries = 0; !getStub && (vintfHwbinder || (vintfLegacy && tries == 0)); tries++) {
        ......
        Return<sp<::android::hidl::base::V1_0::IBase>> ret = 
                sm->get(IComposer::descriptor, serviceName);
        ......
        sp<::android::hidl::base::V1_0::IBase> base = ret;
        ......
        Return<sp<IComposer>> castRet = IComposer::castFrom(base, true /* emitError */);
        ......
        iface = castRet;
        ......
        return iface;
    }
     //passthrough方式下获取IComposer对象
    if (getStub || vintfPassthru || vintfLegacy) {
        const sp<::android::hidl::manager::V1_0::IServiceManager> pm = getPassthroughServiceManager();
        if (pm != nullptr) {
            Return<sp<::android::hidl::base::V1_0::IBase>> ret = 
                    pm->get(IComposer::descriptor, serviceName);
            if (ret.isOk()) {
                sp<::android::hidl::base::V1_0::IBase> baseInterface = ret;
                if (baseInterface != nullptr) {
                    iface = IComposer::castFrom(baseInterface);
                    if (!getStub || trebleTestingOverride) {
                        iface = new BsComposer(iface);
                    }
                }
            }
        }
    }
    return iface;
}

```
这里通过hwservicemanager获取当前服务的Tranport类型，Treble中定义的Tranport包括passthrough和binderized，每个hidl服务都在/system/manifest.xml或者/vendor/manifest.xml中指定了对应的Tranport类型：
``` cpp
     <hal format="hidl">
        <name>android.hardware.graphics.composer</name>
        <transport>hwbinder</transport>
        <version>2.1</version>
        <interface>
            <name>IComposer</name>
            <instance>default</instance>
        </interface>
    </hal>
```
manifest.xml文件的读取和解析都是由hwservicemanager来完成的，此时android.hardware.graphics.composer@2.1-service作为hwservicemanager的client端，通过hwservicemanager的binder代理对象来请求hwservicemanager进程查询IComposer的Transport类型，从上图可以看出IComposer的Transport被定义为hwbinder，因此：

vintfHwbinder=true
vintfPassthru=false
vintfLegacy=false

hidl服务对象获取方式包括2中：

1. 通过查询hwservicemanager来获取；
2. 通过PassthroughServiceManager从本进程地址空间中获取；


那如何选择获取方式呢？ 其实就是vintfHwbinder、vintfPassthru、vintfLegacy、getStub这4个变量值来决定hidl服务的获取方式。

 1. 当getStub为true时，不管hal属于什么传输模式，都采用PassthroughServiceManager获取接口对象；
 2. 当getStub为false时，则根据hal传输模式来选择接口获取方式；

>    《1》 当hal模式为Hwbinder时，则从hwservicemanager中查询；
> 
>    《2》当hal传输模式为Passthru或Legacy时，则采用PassthroughServiceManager来获取；

![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/display.system/Android.PG3.HW2_HIDL_arc.png)

sp<Interface> service = Interface::getService(name, true /* getStub */)所以getStub=true. 这里通过PassthroughServiceManager来获取IComposer对象。其实所有的Hal 进程都是通过PassthroughServiceManager来得到hidl服务对象的，而作为Hal进程的Client端Framework进程在获取hidl服务对象时，需要通过hal的Transport类型来选择获取方式。

``` cpp
system\libhidl\transport\ServiceManagement.cpp

sp<IServiceManager> getPassthroughServiceManager() {
    static sp<PassthroughServiceManager> manager(new PassthroughServiceManager());
    return manager;
}

sp<IServiceManager1_1> getPassthroughServiceManager1_1() {
    static sp<PassthroughServiceManager> manager(new PassthroughServiceManager());
    return manager;
}

struct PassthroughServiceManager : IServiceManager1_1 {
    static void openLibs(
        const std::string& fqName,
        const std::function<bool /* continue */ (void* /* handle */, const std::string& /* lib */,
                                                 const std::string& /* sym */)>& eachLib) {
        //fqName looks like android.hardware.foo@1.0::IFoo
        size_t idx = fqName.find("::");
        ......
        std::string packageAndVersion = fqName.substr(0, idx);
        std::string ifaceName = fqName.substr(idx + strlen("::"));

        const std::string prefix = packageAndVersion + "-impl";
        const std::string sym = "HIDL_FETCH_" + ifaceName;

        constexpr int dlMode = RTLD_LAZY;
        void* handle = nullptr;

        dlerror(); // clear

        static std::string halLibPathVndkSp = android::base::StringPrintf(
            HAL_LIBRARY_PATH_VNDK_SP_FOR_VERSION, details::getVndkVersionStr().c_str());
        std::vector<std::string> paths = {HAL_LIBRARY_PATH_ODM, HAL_LIBRARY_PATH_VENDOR,
                                          halLibPathVndkSp, HAL_LIBRARY_PATH_SYSTEM};

        ......

        for (const std::string& path : paths) {
            std::vector<std::string> libs = search(path, prefix, ".so");

            for (const std::string &lib : libs) {
                const std::string fullPath = path + lib;

                if (path == HAL_LIBRARY_PATH_SYSTEM) {
                    handle = dlopen(fullPath.c_str(), dlMode);
                } else {
                    handle = android_load_sphal_library(fullPath.c_str(), dlMode);
                }

                ......
            }
        }
    }
        .....
        Return<sp<IBase>> get(const hidl_string& fqName,
                          const hidl_string& name) override {
        sp<IBase> ret = nullptr;

        openLibs(fqName, [&](void* handle, const std::string &lib, const std::string &sym) {
            IBase* (*generator)(const char* name);
            *(void **)(&generator) = dlsym(handle, sym.c_str());
            ......
            ret = (*generator)(name.c_str());
            .......
            // Actual fqname might be a subclass.
            // This assumption is tested in vts_treble_vintf_test
            using ::android::hardware::details::getDescriptor;
            std::string actualFqName = getDescriptor(ret.get());
            CHECK(actualFqName.size() > 0);
            registerReference(actualFqName, name);
            return false;
        });

        return ret;
    }
    ......
    }
```

这里只是简单的创建了一个PassthroughServiceManager对象。PassthroughServiceManager也实现了IServiceManager接口。然后通过PassthroughServiceManager询服务：

![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/display.system/Android.PG3.HW2_HIDL_registerReference.png)


根据传入的fqName=(android.hardware.graphics.composer@2.1::IComposer")获取当前的接口名IComposer，拼接出后面需要查找的函数名HIDL_FETCH_IComposer和库名字android.hardware.graphics.composer@2.1-impl.so,然后查找"/system/lib64/hw/"、"/vendor/lib64/hw/"、"/odm/lib64/hw/"下是否有对应的so库。接着通过dlopen载入/vendor/lib/hw/android.hardware.graphics.composer@2.1-impl.so，然后通过dlsym查找并调用HwcLoader::load()函数，最后调用registerReference(fqName, name)向hwservicemanager注册。

``` cpp
hardware/interfaces/graphics/composer/2.1/default/Android.bp
cc_library_shared {
    name: "android.hardware.graphics.composer@2.1-impl",
    defaults: ["hidl_defaults"],
    vendor: true,
    relative_install_path: "hw",
    srcs: ["passthrough.cpp"],
    header_libs: [
        "android.hardware.graphics.composer@2.1-passthrough",
    ],
    shared_libs: [
        "android.hardware.graphics.composer@2.1",
        "android.hardware.graphics.mapper@2.0",
        "libbase",
        "libcutils",
        "libfmq",
        "libhardware",
        "libhidlbase",
        "libhidltransport",
        "liblog",
        "libsync",
        "libutils",
        "libhwc2on1adapter",
        "libhwc2onfbadapter",
    ],
    cflags: [
        "-DLOG_TAG=\"ComposerHal\""
    ],
}

```
从上面的编译脚本可知，android.hardware.graphics.composer@2.1-impl.so的源码文件为passthrough.cpp：

``` cpp
hardware/interfaces/graphics/composer/2.1/default/passthrough.cpp
extern "C" IComposer* HIDL_FETCH_IComposer(const char* /* name */) {
    return HwcLoader::load();
}

......
G:\android9.0\hardware\interfaces\graphics\composer\2.1\utils\passthrough\include\composer-passthrough\2.1\HwcLoader.h
class HwcLoader {
   public:
    static IComposer* load() {
        const hw_module_t* module = loadModule();
        ......
        auto hal = createHalWithAdapter(module);
        ......
        return createComposer(std::move(hal));
    }

    // load hwcomposer2 module
    static const hw_module_t* loadModule() {
        const hw_module_t* module;
        int error = hw_get_module(HWC_HARDWARE_MODULE_ID, &module);
        ......
        return module;
    }

    // create a ComposerHal instance
    static std::unique_ptr<hal::ComposerHal> createHal(const hw_module_t* module) {
        auto hal = std::make_unique<HwcHal>();
        return hal->initWithModule(module) ? std::move(hal) : nullptr;
    }

    // create a ComposerHal instance, insert an adapter if necessary
    static std::unique_ptr<hal::ComposerHal> createHalWithAdapter(const hw_module_t* module) {
        bool adapted;
        hwc2_device_t* device = openDeviceWithAdapter(module, &adapted);
        ......
        auto hal = std::make_unique<HwcHal>();
        return hal->initWithDevice(std::move(device), !adapted) ? std::move(hal) : nullptr;
    }

    // create an IComposer instance
    static IComposer* createComposer(std::unique_ptr<hal::ComposerHal> hal) {
        return hal::Composer::create(std::move(hal)).release();
    }

   protected:
    // open hwcomposer2 device, install an adapter if necessary
    static hwc2_device_t* openDeviceWithAdapter(const hw_module_t* module, bool* outAdapted) {
        if (module->id && std::string(module->id) == GRALLOC_HARDWARE_MODULE_ID) {
            *outAdapted = true;
            return adaptGrallocModule(module);
        }

        hw_device_t* device;
        int error = module->methods->open(module, HWC_HARDWARE_COMPOSER, &device);
        ......

        int major = (device->version >> 24) & 0xf;
        if (major != 2) {
            *outAdapted = true;
            return adaptHwc1Device(std::move(reinterpret_cast<hwc_composer_device_1*>(device)));
        }

        *outAdapted = false;
        return reinterpret_cast<hwc2_device_t*>(device);
    }

   private:
    static hwc2_device_t* adaptGrallocModule(const hw_module_t* module) {
        framebuffer_device_t* device;
        int error = framebuffer_open(module, &device);
        ......
        return new HWC2OnFbAdapter(device);
    }

    static hwc2_device_t* adaptHwc1Device(hwc_composer_device_1* device) {
        int minor = (device->common.version >> 16) & 0xf;
        ......
        return new HWC2On1Adapter(device);
    }
};

......
```

##### 2.3.0 、registerPassthroughClient
得到IComposer接口对象后，需要注册相关信息到hwservicemanager中。

``` cpp
system\libhidl\transport\ServiceManagement.cpp
static void registerReference(const hidl_string &interfaceName, const hidl_string &instanceName) {
    sp<IServiceManager1_0> binderizedManager = defaultServiceManager();
   ......
    auto ret = binderizedManager->registerPassthroughClient(interfaceName, instanceName);
   ......
    LOG(VERBOSE) << "Successfully registerReference for "
                 << interfaceName << "/" << instanceName;
}
```

这里通过hwservicemanager的代理对象跨进程调用registerPassthroughClient。

``` cpp
android\out\soong\.intermediates\system\libhidl\transport\manager\1.1\android.hidl.manager@1.1_genc++\gen\android\hidl\manager\1.1\ServiceManagerAll.cpp

::android::hardware::Return<void> BpHwServiceManager::registerPassthroughClient(const ::android::hardware::hidl_string& fqName, const ::android::hardware::hidl_string& name){
    ::android::hardware::Return<void>  _hidl_out = ::android::hidl::manager::V1_0::BpHwServiceManager::_hidl_registerPassthroughClient(this, this, fqName, name);

    return _hidl_out;
}
```

``` cpp
\android\out\soong\.intermediates\system\libhidl\transport\manager\1.0\android.hidl.manager@1.0_genc++\gen\android\hidl\manager\1.0\ServiceManagerAll.cpp

::android::hardware::Return<void> BpHwServiceManager::_hidl_registerPassthroughClient(::android::hardware::IInterface *_hidl_this, ::android::hardware::details::HidlInstrumentor *_hidl_this_instrumentor, const ::android::hardware::hidl_string& fqName, const ::android::hardware::hidl_string& name) {
    #ifdef __ANDROID_DEBUGGABLE__
    bool mEnableInstrumentation = _hidl_this_instrumentor->isInstrumentationEnabled();
    const auto &mInstrumentationCallbacks = _hidl_this_instrumentor->getInstrumentationCallbacks();
    #else
    (void) _hidl_this_instrumentor;
    #endif // __ANDROID_DEBUGGABLE__
    atrace_begin(ATRACE_TAG_HAL, "HIDL::IServiceManager::registerPassthroughClient::client");
    #ifdef __ANDROID_DEBUGGABLE__
    if (UNLIKELY(mEnableInstrumentation)) {
        std::vector<void *> _hidl_args;
        _hidl_args.push_back((void *)&fqName);
        _hidl_args.push_back((void *)&name);
        for (const auto &callback: mInstrumentationCallbacks) {
            callback(InstrumentationEvent::CLIENT_API_ENTRY, "android.hidl.manager", "1.0", "IServiceManager", "registerPassthroughClient", &_hidl_args);
        }
    }
    #endif // __ANDROID_DEBUGGABLE__

    ::android::hardware::Parcel _hidl_data;
    ::android::hardware::Parcel _hidl_reply;
    ::android::status_t _hidl_err;
    ::android::hardware::Status _hidl_status;

    _hidl_err = _hidl_data.writeInterfaceToken(BpHwServiceManager::descriptor);
    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    size_t _hidl_fqName_parent;

    _hidl_err = _hidl_data.writeBuffer(&fqName, sizeof(fqName), &_hidl_fqName_parent);
    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    _hidl_err = ::android::hardware::writeEmbeddedToParcel(
            fqName,
            &_hidl_data,
            _hidl_fqName_parent,
            0 /* parentOffset */);

    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    size_t _hidl_name_parent;

    _hidl_err = _hidl_data.writeBuffer(&name, sizeof(name), &_hidl_name_parent);
    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    _hidl_err = ::android::hardware::writeEmbeddedToParcel(
            name,
            &_hidl_data,
            _hidl_name_parent,
            0 /* parentOffset */);

    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    _hidl_err = ::android::hardware::IInterface::asBinder(_hidl_this)->transact(8 /* registerPassthroughClient */, _hidl_data, &_hidl_reply);
    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    _hidl_err = ::android::hardware::readFromParcel(&_hidl_status, _hidl_reply);
    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    if (!_hidl_status.isOk()) { return _hidl_status; }

    atrace_end(ATRACE_TAG_HAL);
    #ifdef __ANDROID_DEBUGGABLE__
    if (UNLIKELY(mEnableInstrumentation)) {
        std::vector<void *> _hidl_args;
        for (const auto &callback: mInstrumentationCallbacks) {
            callback(InstrumentationEvent::CLIENT_API_EXIT, "android.hidl.manager", "1.0", "IServiceManager", "registerPassthroughClient", &_hidl_args);
        }
    }
    #endif // __ANDROID_DEBUGGABLE__

    _hidl_status.setFromStatusT(_hidl_err);
    return ::android::hardware::Return<void>();

......
}

```


这里和普通binder通信相同，先就需要传输的函数参数打包到Parcel对象中，然后调用binder代理对象的transact函数将函数参数，函数调用码发送到Server端进程，这里的_hidl_this其实指向的是BpHwServiceManager，这个是与业务相关的代理对象，通过asBinder函数得到与传输相关的binder代理，那这个binder代理是什么类型呢？ 其实就是BpHwBinder，关于hwservicemanager代理对象的获取，asBinder函数的实现，在后续的章节中进行分析。经过BpHwServiceManager的请求，最终位于hwservicemanager进程中的BnHwServiceManager将接收函数调用请求：

``` cpp
android\out\soong\.intermediates\system\libhidl\transport\manager\1.1\android.hidl.manager@1.1_genc++\gen\android\hidl\manager\1.1\ServiceManagerAll.cpp
::android::status_t BnHwServiceManager::onTransact(
	uint32_t _hidl_code,
	const ::android::hardware::Parcel &_hidl_data,
	::android::hardware::Parcel *_hidl_reply,
	uint32_t _hidl_flags,
	TransactCallback _hidl_cb) {
::android::status_t _hidl_err = ::android::OK;
 
switch (_hidl_code) {
	case 8 /* registerPassthroughClient */:
	{
		_hidl_err = ::android::hidl::manager::V1_0::BnHwServiceManager::_hidl_registerPassthroughClient(this, _hidl_data, _hidl_reply, _hidl_cb);
		break;
	}
	default:
	{
		return ::android::hidl::base::V1_0::BnHwBase::onTransact(
				_hidl_code, _hidl_data, _hidl_reply, _hidl_flags, _hidl_cb);
	}
}
```
BnHwServiceManager将调用_hidl_registerPassthroughClient来执行Client端的注册。


``` cpp
\android\out\soong\.intermediates\system\libhidl\transport\manager\1.0\android.hidl.manager@1.0_genc++\gen\android\hidl\manager\1.0\ServiceManagerAll.cpp

::android::status_t BnHwServiceManager::_hidl_registerPassthroughClient(
        ::android::hidl::base::V1_0::BnHwBase* _hidl_this,
        const ::android::hardware::Parcel &_hidl_data,
        ::android::hardware::Parcel *_hidl_reply,
        TransactCallback _hidl_cb) {
    #ifdef __ANDROID_DEBUGGABLE__
    bool mEnableInstrumentation = _hidl_this->isInstrumentationEnabled();
    const auto &mInstrumentationCallbacks = _hidl_this->getInstrumentationCallbacks();
    #endif // __ANDROID_DEBUGGABLE__

    ::android::status_t _hidl_err = ::android::OK;
    if (!_hidl_data.enforceInterface(BnHwServiceManager::Pure::descriptor)) {
        _hidl_err = ::android::BAD_TYPE;
        return _hidl_err;
    }

    const ::android::hardware::hidl_string* fqName;
    const ::android::hardware::hidl_string* name;

    size_t _hidl_fqName_parent;

    _hidl_err = _hidl_data.readBuffer(sizeof(*fqName), &_hidl_fqName_parent,  reinterpret_cast<const void **>(&fqName));

    if (_hidl_err != ::android::OK) { return _hidl_err; }

    _hidl_err = ::android::hardware::readEmbeddedFromParcel(
            const_cast<::android::hardware::hidl_string &>(*fqName),
            _hidl_data,
            _hidl_fqName_parent,
            0 /* parentOffset */);

    if (_hidl_err != ::android::OK) { return _hidl_err; }

    size_t _hidl_name_parent;

    _hidl_err = _hidl_data.readBuffer(sizeof(*name), &_hidl_name_parent,  reinterpret_cast<const void **>(&name));

    if (_hidl_err != ::android::OK) { return _hidl_err; }

    _hidl_err = ::android::hardware::readEmbeddedFromParcel(
            const_cast<::android::hardware::hidl_string &>(*name),
            _hidl_data,
            _hidl_name_parent,
            0 /* parentOffset */);

    if (_hidl_err != ::android::OK) { return _hidl_err; }

    atrace_begin(ATRACE_TAG_HAL, "HIDL::IServiceManager::registerPassthroughClient::server");
    #ifdef __ANDROID_DEBUGGABLE__
    if (UNLIKELY(mEnableInstrumentation)) {
        std::vector<void *> _hidl_args;
        _hidl_args.push_back((void *)fqName);
        _hidl_args.push_back((void *)name);
        for (const auto &callback: mInstrumentationCallbacks) {
            callback(InstrumentationEvent::SERVER_API_ENTRY, "android.hidl.manager", "1.0", "IServiceManager", "registerPassthroughClient", &_hidl_args);
        }
    }
    #endif // __ANDROID_DEBUGGABLE__

    static_cast<BnHwServiceManager*>(_hidl_this)->_hidl_mImpl->registerPassthroughClient(*fqName, *name);

    (void) _hidl_cb;

    atrace_end(ATRACE_TAG_HAL);
    #ifdef __ANDROID_DEBUGGABLE__
    if (UNLIKELY(mEnableInstrumentation)) {
        std::vector<void *> _hidl_args;
        for (const auto &callback: mInstrumentationCallbacks) {
            callback(InstrumentationEvent::SERVER_API_EXIT, "android.hidl.manager", "1.0", "IServiceManager", "registerPassthroughClient", &_hidl_args);
        }
    }
    #endif // __ANDROID_DEBUGGABLE__

    ::android::hardware::writeToParcel(::android::hardware::Status::ok(), _hidl_reply);

    return _hidl_err;
}
```
BnHwServiceManager首先读取BpHwServiceManager发送过来的函数参数，然后将registerPassthroughClient的执行转交个其成员变量的_hidl_mImpl对象，然后将执行结果返回给BpHwServiceManager，那么_hidl_mImpl保存的是什么对象呢？ 其实_hidl_mImpl指向的是ServiceManager对象，这个是在构造BnHwServiceManager对象时传入的，在后续分析hwservicemanager启动过程时，会进行详细分析。

``` cpp
system\hwservicemanager\ServiceManager.cpp
Return<void> ServiceManager::registerPassthroughClient(const hidl_string &fqName,
        const hidl_string &name) {
    pid_t pid = IPCThreadState::self()->getCallingPid();
    ......

    PackageInterfaceMap &ifaceMap = mServiceMap[fqName];

    ......

    HidlService *service = ifaceMap.lookup(name);

    if (service == nullptr) {
        auto adding = std::make_unique<HidlService>(fqName, name);
        adding->registerPassthroughClient(pid);
        ifaceMap.insertService(std::move(adding));
    } else {
        service->registerPassthroughClient(pid);
    }
    return Void();
}
```
首先根据fqName从mServiceMap中查找对应的PackageInterfaceMap，然后根据name从PackageInterfaceMap中查找HidlService，如果找不到对应的HidlService对象，那么就调用std::make_unique<HidlService>(fqName,name)创建一个新的HidlService对象，并ifaceMap.insertService(std::move(adding))添加到PackageInterfaceMap中。如果查找到了HidlService对象，那么仅仅将Client进程的pid保存到HidlService的mPassthroughClients变量中。

``` cpp
android/system/hwservicemanager/HidlService.cpp
void HidlService::registerPassthroughClient(pid_t pid) {
    mPassthroughClients.insert(pid);
}
```
因此registerPassthroughClient在hwservicemanager中插入一个HidlService对象而已，并没有注册对应的IBase对象。getService最后将HwcHal对象返回给registerPassthroughServiceImplementation()函数，然后再次调用registerAsService注册该IBase对象。

![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/display.system/Android.PG3.HW2_HIDL_registerPassthroughClient.png)


##### 2.4.0 、registerAsService注册
registerAsService用于向hwservicemanager注册IBase对象，由于前面通过PassthroughServiceManager得到的HwcHal继承于IBase，因此可以调用registerAsService函数来注册。

``` cpp
android\out\soong\.intermediates\system\libhidl\transport\manager\1.0\android.hidl.manager@1.0_genc++\gen\android\hidl\manager\1.0\ServiceManagerAll.cpp


::android::status_t IServiceManager::registerAsService(const std::string &serviceName) {
    ::android::hardware::details::onRegistration("android.hidl.manager@1.0", "IServiceManager", serviceName);

    const ::android::sp<IServiceManager> sm
            = ::android::hardware::defaultServiceManager();
    if (sm == nullptr) {
        return ::android::INVALID_OPERATION;
    }
    ::android::hardware::Return<bool> ret = sm->add(serviceName.c_str(), this);
    return ret.isOk() && ret ? ::android::OK : ::android::UNKNOWN_ERROR;
}

```
首先执行onRegistration函数，然后调用hwservicemanager的代理对象的add函数。

``` cpp
system\libhidl\transport\ServiceManagement.cpp

void onRegistration(const std::string &packageName,
                    const std::string& /* interfaceName */,
                    const std::string& /* instanceName */) {
    tryShortenProcessName(packageName);
}

void tryShortenProcessName(const std::string &packageName) {
    std::string processName = binaryName();
    ......
    // e.x. android.hardware.module.foo@1.0 -> foo@1.0
    size_t lastDot = packageName.rfind('.');
    size_t secondDot = packageName.rfind('.', lastDot - 1);

    .....
    std::string newName = processName.substr(secondDot + 1,
            16 /* TASK_COMM_LEN */ - 1);
    ......

    int rc = pthread_setname_np(pthread_self(), newName.c_str());
    ......
}
```
这里只是简单的修改了当前进程的名称。

``` cpp
android\out\soong\.intermediates\system\libhidl\transport\manager\1.0\android.hidl.manager@1.0_genc++\gen\android\hidl\manager\1.0\ServiceManagerAll.cpp

::android::hardware::Return<bool> BpHwServiceManager::add(const ::android::hardware::hidl_string& name, const ::android::sp<::android::hidl::base::V1_0::IBase>& service){
    ::android::hardware::Return<bool>  _hidl_out = ::android::hidl::manager::V1_0::BpHwServiceManager::_hidl_add(this, this, name, service);

    return _hidl_out;
}


::android::hardware::Return<bool> BpHwServiceManager::_hidl_add(::android::hardware::IInterface *_hidl_this, ::android::hardware::details::HidlInstrumentor *_hidl_this_instrumentor, const ::android::hardware::hidl_string& name, const ::android::sp<::android::hidl::base::V1_0::IBase>& service) {
    bool mEnableInstrumentation = _hidl_this_instrumentor->isInstrumentationEnabled();
    const auto &mInstrumentationCallbacks = _hidl_this_instrumentor->getInstrumentationCallbacks();
    #else
    (void) _hidl_this_instrumentor;
    ......

    ::android::hardware::Parcel _hidl_data;
    ::android::hardware::Parcel _hidl_reply;
    ::android::status_t _hidl_err;
    ::android::hardware::Status _hidl_status;

    bool _hidl_out_success;

    _hidl_err = _hidl_data.writeInterfaceToken(BpHwServiceManager::descriptor);

    size_t _hidl_name_parent;

    _hidl_err = _hidl_data.writeBuffer(&name, sizeof(name), &_hidl_name_parent);

    _hidl_err = ::android::hardware::writeEmbeddedToParcel(
            name,
            &_hidl_data,
            _hidl_name_parent,
            0 /* parentOffset */);

    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    if (service == nullptr) {
        _hidl_err = _hidl_data.writeStrongBinder(nullptr);
    } else {
        ::android::sp<::android::hardware::IBinder> _hidl_binder = ::android::hardware::toBinder<
                ::android::hidl::base::V1_0::IBase>(service);
        if (_hidl_binder.get() != nullptr) {
            _hidl_err = _hidl_data.writeStrongBinder(_hidl_binder);
        } else {
            _hidl_err = ::android::UNKNOWN_ERROR;
        }
    }
}
    ::android::hardware::ProcessState::self()->startThreadPool();
    _hidl_err = ::android::hardware::IInterface::asBinder(_hidl_this)->transact(2 /* add */, _hidl_data, &_hidl_reply);

    _hidl_err = ::android::hardware::readFromParcel(&_hidl_status, _hidl_reply);

    if (!_hidl_status.isOk()) { return _hidl_status; }

    _hidl_err = _hidl_reply.readBool(&_hidl_out_success);

    _hidl_status.setFromStatusT(_hidl_err);
    return ::android::hardware::Return<bool>(_hidl_out_success);
......
    return ::android::hardware::Return<bool>(_hidl_status);
}

```
这里的步骤和前面的registerPassthroughClient基本一致，唯一不同的是，此时需要向Server端hwservicemanager传输一个IBase对象。

``` cpp
::android::sp<::android::hardware::IBinder> _hidl_binder = ::android::hardware::toBinder<
		::android::hidl::base::V1_0::IBase>(service);
if (_hidl_binder.get() != nullptr) {
	_hidl_err = _hidl_data.writeStrongBinder(_hidl_binder);
}
```
这里首先通过toBinder函数将IBase对象，其实就是HwcHal对象转换为IBinder对象，然后通过writeStrongBinder将IBinder对象序列化到Parcel中，toBinder函数在后续进行分析，我们这里只需要知道经过toBinder函数后，在Hal进程端会创建一个BnHwComposer本地binder对象，然后通过IPC调用发送给hwservicemanager。

``` cpp
android\out\soong\.intermediates\system\libhidl\transport\manager\1.0\android.hidl.manager@1.0_genc++\gen\android\hidl\manager\1.0\ServiceManagerAll.cpp

::android::status_t BnHwServiceManager::onTransact(
        uint32_t _hidl_code,
        const ::android::hardware::Parcel &_hidl_data,
        ::android::hardware::Parcel *_hidl_reply,
        uint32_t _hidl_flags,
        TransactCallback _hidl_cb) {
    ::android::status_t _hidl_err = ::android::OK;
    switch (_hidl_code) {
        case 2 /* add */:
        {
            _hidl_err = ::android::hidl::manager::V1_0::BnHwServiceManager::_hidl_add(this, _hidl_data, _hidl_reply, _hidl_cb);
            break;
        }
        default:
        {
            return ::android::hidl::base::V1_0::BnHwBase::onTransact(
                    _hidl_code, _hidl_data, _hidl_reply, _hidl_flags, _hidl_cb);
        }
    }
    if (_hidl_err == ::android::UNEXPECTED_NULL) {
        _hidl_err = ::android::hardware::writeToParcel(
                ::android::hardware::Status::fromExceptionCode(::android::hardware::Status::EX_NULL_POINTER),
                _hidl_reply);
    }return _hidl_err;
}

--------------------------------------------------------------------------------------------

::android::status_t BnHwServiceManager::_hidl_add(
        ::android::hidl::base::V1_0::BnHwBase* _hidl_this,
        const ::android::hardware::Parcel &_hidl_data,
        ::android::hardware::Parcel *_hidl_reply,
        TransactCallback _hidl_cb) {

    ::android::status_t _hidl_err = ::android::OK;

    const ::android::hardware::hidl_string* name;
    ::android::sp<::android::hidl::base::V1_0::IBase> service;

    size_t _hidl_name_parent;

    _hidl_err = _hidl_data.readBuffer(sizeof(*name), &_hidl_name_parent,  reinterpret_cast<const void **>(&name));


    _hidl_err = ::android::hardware::readEmbeddedFromParcel(
            const_cast<::android::hardware::hidl_string &>(*name),
            _hidl_data,
            _hidl_name_parent,
            0 /* parentOffset */);

    if (_hidl_err != ::android::OK) { return _hidl_err; }

    {
        ::android::sp<::android::hardware::IBinder> _hidl_service_binder;
        _hidl_err = _hidl_data.readNullableStrongBinder(&_hidl_service_binder);
        if (_hidl_err != ::android::OK) { return _hidl_err; }

        service = ::android::hardware::fromBinder<::android::hidl::base::V1_0::IBase,::android::hidl::base::V1_0::BpHwBase,::android::hidl::base::V1_0::BnHwBase>(_hidl_service_binder);
    }

    bool _hidl_out_success = static_cast<BnHwServiceManager*>(_hidl_this)->_hidl_mImpl->add(*name, service);

    ::android::hardware::writeToParcel(::android::hardware::Status::ok(), _hidl_reply);

    _hidl_err = _hidl_reply->writeBool(_hidl_out_success);

    _hidl_cb(*_hidl_reply);
    return _hidl_err;
}
```
wservicemanager进程通过_hidl_err = _hidl_data.readNullableStrongBinder(&_hidl_service_binder);拿到client进程发送过来的BnHwComposer对象，binder实体到达目的端进程将变为binder代理对象，然后通过fromBinder函数将binder代理对象转换为业务代理对象BpHwBase，这个过程在后续进行详细分析，接下来继续调用_hidl_mImpl的add函数，而我们知道_hidl_mImpl其实就是ServiceManager：
![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/display.system/Android.PG3.HW2_HIDL_add.png)


``` cpp
android\system\hwservicemanager\ServiceManager.cpp
Return<bool> ServiceManager::add(const hidl_string& name, const sp<IBase>& service) {
    bool isValidService = false;

    ......
    pid_t pid = IPCThreadState::self()->getCallingPid();
    auto context = mAcl.getContext(pid);

    auto ret = service->interfaceChain([&](const auto &interfaceChain) {
        ......
        // First, verify you're allowed to add() the whole interface hierarchy
        for(size_t i = 0; i < interfaceChain.size(); i++) {
            std::string fqName = interfaceChain[i];

            if (!mAcl.canAdd(fqName, context, pid)) {
                return;
            }
        }

        for(size_t i = 0; i < interfaceChain.size(); i++) {
            std::string fqName = interfaceChain[i];

            PackageInterfaceMap &ifaceMap = mServiceMap[fqName];
            HidlService *hidlService = ifaceMap.lookup(name);

            if (hidlService == nullptr) {
                ifaceMap.insertService(
                    std::make_unique<HidlService>(fqName, name, service, pid));
            } else {
                if (hidlService->getService() != nullptr) {
                    auto ret = hidlService->getService()->unlinkToDeath(this);
                    ret.isOk(); // ignore
                }
                hidlService->setService(service, pid);
            }

            ifaceMap.sendPackageRegistrationNotification(fqName, name);
        }

        auto linkRet = service->linkToDeath(this, 0 /*cookie*/);
        linkRet.isOk(); // ignore

        isValidService = true;
    });


    return isValidService;
}

```
##### 2.5.0 、IComposer服务注册
如果服务注册进程有权限向hwservicemanager注册服务，接下来将完成服务添加。
![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/display.system/Android.PG3.HW2_HIDL_mServiceMap.png)
每个服务接口对应多个实例，比如android.hidl.manager@1.1::IServiceManager可以注册多个实例，每个实例名称不同。
![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/display.system/Android.PG3.HW2_HIDL_mInstanceMap.png)

``` cpp
PackageInterfaceMap &ifaceMap = mServiceMap[fqName];
HidlService *hidlService = ifaceMap.lookup(name);
const HidlService *ServiceManager::PackageInterfaceMap::lookup(
        const std::string &name) const {
    auto it = mInstanceMap.find(name);
 
    if (it == mInstanceMap.end()) {
        return nullptr;
    }
    return it->second.get();
}
```

根据名称查找HidlService，如果找不到，则新增一个HidlService，如果已经存在，则更新service。

``` cpp
if (hidlService == nullptr) {
	LOG(INFO) << "insertService " << name << " of " << fgName ;
	ifaceMap.insertService(
		std::make_unique<HidlService>(fqName, name, service, pid));
} else {
	if (hidlService->getService() != nullptr) {
		auto ret = hidlService->getService()->unlinkToDeath(this);
		ret.isOk(); // ignore
	}
	LOG(INFO) << "setService " << " of " << fgName ;
	hidlService->setService(service, pid);
}
```
![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/display.system/Android.PG3.HW2_HIDL_registerSuccess.png)

##### 2.6.0 、IComposer服务查询
通过前面的分析我们知道，Hal进程启动时，会向hwservicemanager进程注册hidl服务，那么当Framework Server需要通过hal访问硬件设备时，首先需要查询对应的hidl服务，那么Client进程是如何查询hidl服务的呢？这篇文章将展开分析，这里再次以IComposer为例进行展开。

![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/display.system/Android.PG3.HW2_HIDL_BpHwComposer.png)

``` cpp
frameworks\native\services\surfaceflinger\DisplayHardware\ComposerHal.cpp

Composer::Composer(bool useVrComposer)
    : mWriter(kWriterInitialSize),
      mIsUsingVrComposer(useVrComposer)
{
    if (mIsUsingVrComposer) {
        mComposer = IComposer::getService("vr");
    } else {
        mComposer = IComposer::getService(); // use default name
    }

    if (mComposer == nullptr) {
        LOG_ALWAYS_FATAL("failed to get hwcomposer service");
    }

    mComposer->createClient(
            [&](const auto& tmpError, const auto& tmpClient)
            {
                if (tmpError == Error::NONE) {
                    mClient = tmpClient;
                }
            });
    if (mClient == nullptr) {
        LOG_ALWAYS_FATAL("failed to create composer client");
    }

    if (mIsUsingVrComposer) {
        sp<IVrComposerClient> vrClient = IVrComposerClient::castFrom(mClient);
        if (vrClient == nullptr) {
            LOG_ALWAYS_FATAL("failed to create vr composer client");
        }
    }
}

```
这里通过IComposer::getService()函数来查询IComposer这个HIDL服务，由于这里没有传递任何参数，因此函数最终会调用：

``` cpp
android\out\soong\.intermediates\hardware\interfaces\graphics\composer\2.1\android.hardware.graphics.composer@2.1_genc++_headers\gen\android\hardware\graphics\composer\2.1
    static ::android::sp<IComposer> getService(const std::string &serviceName="default", bool getStub=false);
```
注意，这里的getStub为false，说明加载hidl服务方式是由当前hidl服务的transport类型决定。
由于IComposer的transport是hwbinder类型，那么将从hwservicemanager中查询hidl服务。


``` cpp
android\out\soong\.intermediates\hardware\interfaces\graphics\composer\2.1\android.hardware.graphics.composer@2.1_genc++\gen\android\hardware\graphics\composer\2.1\ComposerAll.cpp
// static
::android::sp<IComposer> IComposer::getService(const std::string &serviceName, const bool getStub) {
    using ::android::hardware::defaultServiceManager;
    using ::android::hardware::details::waitForHwService;
    using ::android::hardware::getPassthroughServiceManager;
    using ::android::hardware::Return;
    using ::android::sp;
    using Transport = ::android::hidl::manager::V1_0::IServiceManager::Transport;

    sp<IComposer> iface = nullptr;

    const sp<::android::hidl::manager::V1_0::IServiceManager> sm = defaultServiceManager();
    .......
    Return<Transport> transportRet = sm->getTransport(IComposer::descriptor, serviceName);

    ......
    Transport transport = transportRet;
    const bool vintfHwbinder = (transport == Transport::HWBINDER);
    const bool vintfPassthru = (transport == Transport::PASSTHROUGH);

    #ifdef __ANDROID_TREBLE__

    ......

    for (int tries = 0; !getStub && (vintfHwbinder || (vintfLegacy && tries == 0)); tries++) {
        ......
        if (vintfHwbinder && tries > 0) {
            waitForHwService(IComposer::descriptor, serviceName);
        }
        Return<sp<::android::hidl::base::V1_0::IBase>> ret = 
                sm->get(IComposer::descriptor, serviceName);
        ......
        sp<::android::hidl::base::V1_0::IBase> base = ret;
        if (base == nullptr) {
            if (tries > 0) {
                ALOGW("IComposer: found null hwbinder interface");
            }continue;
        }
        Return<sp<IComposer>> castRet = IComposer::castFrom(base, true /* emitError */);
        if (!castRet.isOk()) {
            if (castRet.isDeadObject()) {
                ALOGW("IComposer: found dead hwbinder service");
                continue;
            } else {
                ALOGW("IComposer: cannot call into hwbinder service: %s; No permission? Check for selinux denials.", castRet.description().c_str());
                break;
            }
        }
        iface = castRet;
        if (iface == nullptr) {
            ALOGW("IComposer: received incompatible service; bug in hwservicemanager?");
            break;
        }
        return iface;
    }
    if (getStub || vintfPassthru || vintfLegacy) {
        const sp<::android::hidl::manager::V1_0::IServiceManager> pm = getPassthroughServiceManager();
        if (pm != nullptr) {
            Return<sp<::android::hidl::base::V1_0::IBase>> ret = 
                    pm->get(IComposer::descriptor, serviceName);
            if (ret.isOk()) {
                sp<::android::hidl::base::V1_0::IBase> baseInterface = ret;
                if (baseInterface != nullptr) {
                    iface = IComposer::castFrom(baseInterface);
                    if (!getStub || trebleTestingOverride) {
                        iface = new BsComposer(iface);
                    }
                }
            }
        }
    }
    return iface;
}
```
这里通过sm->get(IComposer::descriptor, serviceName)查询IComposer这个hidl服务，得到IBase对象后，在通过IComposer::castFrom转换为IComposer对象。
![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/display.system/Android.PG3.HW2_HIDL_castFrom.png)

**服务查询**

``` cpp
android\out\soong\.intermediates\system\libhidl\transport\manager\1.0\android.hidl.manager@1.0_genc++\gen\android\hidl\manager\1.0\ServiceManagerAll.cpp

// Methods from IServiceManager follow.
::android::hardware::Return<::android::sp<::android::hidl::base::V1_0::IBase>> BpHwServiceManager::get(const ::android::hardware::hidl_string& fqName, const ::android::hardware::hidl_string& name){
    ::android::hardware::Return<::android::sp<::android::hidl::base::V1_0::IBase>>  _hidl_out = ::android::hidl::manager::V1_0::BpHwServiceManager::_hidl_get(this, this, fqName, name);

    return _hidl_out;
}




// Methods from IServiceManager follow.
::android::hardware::Return<::android::sp<::android::hidl::base::V1_0::IBase>> BpHwServiceManager::_hidl_get(::android::hardware::IInterface *_hidl_this, ::android::hardware::details::HidlInstrumentor *_hidl_this_instrumentor, const ::android::hardware::hidl_string& fqName, const ::android::hardware::hidl_string& name) {
    #ifdef __ANDROID_DEBUGGABLE__
    bool mEnableInstrumentation = _hidl_this_instrumentor->isInstrumentationEnabled();
    const auto &mInstrumentationCallbacks = _hidl_this_instrumentor->getInstrumentationCallbacks();
    #else
    (void) _hidl_this_instrumentor;
    #endif // __ANDROID_DEBUGGABLE__
    atrace_begin(ATRACE_TAG_HAL, "HIDL::IServiceManager::get::client");
    #ifdef __ANDROID_DEBUGGABLE__
    if (UNLIKELY(mEnableInstrumentation)) {
        std::vector<void *> _hidl_args;
        _hidl_args.push_back((void *)&fqName);
        _hidl_args.push_back((void *)&name);
        for (const auto &callback: mInstrumentationCallbacks) {
            callback(InstrumentationEvent::CLIENT_API_ENTRY, "android.hidl.manager", "1.0", "IServiceManager", "get", &_hidl_args);
        }
    }
    #endif // __ANDROID_DEBUGGABLE__

    ::android::hardware::Parcel _hidl_data;
    ::android::hardware::Parcel _hidl_reply;
    ::android::status_t _hidl_err;
    ::android::hardware::Status _hidl_status;

    ::android::sp<::android::hidl::base::V1_0::IBase> _hidl_out_service;

    _hidl_err = _hidl_data.writeInterfaceToken(BpHwServiceManager::descriptor);
    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    size_t _hidl_fqName_parent;

    _hidl_err = _hidl_data.writeBuffer(&fqName, sizeof(fqName), &_hidl_fqName_parent);
    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    _hidl_err = ::android::hardware::writeEmbeddedToParcel(
            fqName,
            &_hidl_data,
            _hidl_fqName_parent,
            0 /* parentOffset */);

    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    size_t _hidl_name_parent;

    _hidl_err = _hidl_data.writeBuffer(&name, sizeof(name), &_hidl_name_parent);
    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    _hidl_err = ::android::hardware::writeEmbeddedToParcel(
            name,
            &_hidl_data,
            _hidl_name_parent,
            0 /* parentOffset */);

    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    _hidl_err = ::android::hardware::IInterface::asBinder(_hidl_this)->transact(1 /* get */, _hidl_data, &_hidl_reply);
    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    _hidl_err = ::android::hardware::readFromParcel(&_hidl_status, _hidl_reply);
    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    if (!_hidl_status.isOk()) { return _hidl_status; }

    {
        ::android::sp<::android::hardware::IBinder> _hidl__hidl_out_service_binder;
        _hidl_err = _hidl_reply.readNullableStrongBinder(&_hidl__hidl_out_service_binder);
        if (_hidl_err != ::android::OK) { goto _hidl_error; }

        _hidl_out_service = ::android::hardware::fromBinder<::android::hidl::base::V1_0::IBase,::android::hidl::base::V1_0::BpHwBase,::android::hidl::base::V1_0::BnHwBase>(_hidl__hidl_out_service_binder);
    }

    atrace_end(ATRACE_TAG_HAL);
    #ifdef __ANDROID_DEBUGGABLE__
    if (UNLIKELY(mEnableInstrumentation)) {
        std::vector<void *> _hidl_args;
        _hidl_args.push_back((void *)&_hidl_out_service);
        for (const auto &callback: mInstrumentationCallbacks) {
            callback(InstrumentationEvent::CLIENT_API_EXIT, "android.hidl.manager", "1.0", "IServiceManager", "get", &_hidl_args);
        }
    }
    #endif // __ANDROID_DEBUGGABLE__

    _hidl_status.setFromStatusT(_hidl_err);
    return ::android::hardware::Return<::android::sp<::android::hidl::base::V1_0::IBase>>(_hidl_out_service);

_hidl_error:
    _hidl_status.setFromStatusT(_hidl_err);
    return ::android::hardware::Return<::android::sp<::android::hidl::base::V1_0::IBase>>(_hidl_status);
}
```
整个调用过程和hidl服务过程完全一致，就是一个从BpHwServiceManager --> BnHwServiceManager --> ServiceManager的过程。但需要注意，BpHwServiceManager得到BnHwServiceManager返回过来的binder代理后，会通过fromBinder函数进行对象转换：

::android::hardware::fromBinder<::android::hidl::base::V1_0::IBase,::android::hidl::base::V1_0::BpHwBase,::android::hidl::base::V1_0::BnHwBase>(_hidl__hidl_out_service_binder)

hwservicemanager将IComposer的binder代理BpHwBinder发给Framework Server进程，Framework Server进程拿到的依然是IComposer的binder代理BpHwBinder对象，因此在fromBinder函数中将创建BpHwBase对象来封装BpHwBinder。


``` cpp
android\out\soong\.intermediates\system\libhidl\transport\manager\1.0\android.hidl.manager@1.0_genc++\gen\android\hidl\manager\1.0\ServiceManagerAll.cpp

// Methods from IServiceManager follow.
::android::status_t BnHwServiceManager::_hidl_get(
        ::android::hidl::base::V1_0::BnHwBase* _hidl_this,
        const ::android::hardware::Parcel &_hidl_data,
        ::android::hardware::Parcel *_hidl_reply,
        TransactCallback _hidl_cb) {
    #ifdef __ANDROID_DEBUGGABLE__
    bool mEnableInstrumentation = _hidl_this->isInstrumentationEnabled();
    const auto &mInstrumentationCallbacks = _hidl_this->getInstrumentationCallbacks();
    #endif // __ANDROID_DEBUGGABLE__

    ::android::status_t _hidl_err = ::android::OK;
    if (!_hidl_data.enforceInterface(BnHwServiceManager::Pure::descriptor)) {
        _hidl_err = ::android::BAD_TYPE;
        return _hidl_err;
    }

    const ::android::hardware::hidl_string* fqName;
    const ::android::hardware::hidl_string* name;

    size_t _hidl_fqName_parent;

    _hidl_err = _hidl_data.readBuffer(sizeof(*fqName), &_hidl_fqName_parent,  reinterpret_cast<const void **>(&fqName));

    if (_hidl_err != ::android::OK) { return _hidl_err; }

    _hidl_err = ::android::hardware::readEmbeddedFromParcel(
            const_cast<::android::hardware::hidl_string &>(*fqName),
            _hidl_data,
            _hidl_fqName_parent,
            0 /* parentOffset */);

    if (_hidl_err != ::android::OK) { return _hidl_err; }

    size_t _hidl_name_parent;

    _hidl_err = _hidl_data.readBuffer(sizeof(*name), &_hidl_name_parent,  reinterpret_cast<const void **>(&name));

    if (_hidl_err != ::android::OK) { return _hidl_err; }

    _hidl_err = ::android::hardware::readEmbeddedFromParcel(
            const_cast<::android::hardware::hidl_string &>(*name),
            _hidl_data,
            _hidl_name_parent,
            0 /* parentOffset */);

    if (_hidl_err != ::android::OK) { return _hidl_err; }

    atrace_begin(ATRACE_TAG_HAL, "HIDL::IServiceManager::get::server");
    #ifdef __ANDROID_DEBUGGABLE__
    if (UNLIKELY(mEnableInstrumentation)) {
        std::vector<void *> _hidl_args;
        _hidl_args.push_back((void *)fqName);
        _hidl_args.push_back((void *)name);
        for (const auto &callback: mInstrumentationCallbacks) {
            callback(InstrumentationEvent::SERVER_API_ENTRY, "android.hidl.manager", "1.0", "IServiceManager", "get", &_hidl_args);
        }
    }
    #endif // __ANDROID_DEBUGGABLE__

    ::android::sp<::android::hidl::base::V1_0::IBase> _hidl_out_service = static_cast<BnHwServiceManager*>(_hidl_this)->_hidl_mImpl->get(*fqName, *name);

    ::android::hardware::writeToParcel(::android::hardware::Status::ok(), _hidl_reply);

    if (_hidl_out_service == nullptr) {
        _hidl_err = _hidl_reply->writeStrongBinder(nullptr);
    } else {
        ::android::sp<::android::hardware::IBinder> _hidl_binder = ::android::hardware::toBinder<
                ::android::hidl::base::V1_0::IBase>(_hidl_out_service);
        if (_hidl_binder.get() != nullptr) {
            _hidl_err = _hidl_reply->writeStrongBinder(_hidl_binder);
        } else {
            _hidl_err = ::android::UNKNOWN_ERROR;
        }
    }
    /* _hidl_err ignored! */

    atrace_end(ATRACE_TAG_HAL);
    #ifdef __ANDROID_DEBUGGABLE__
    if (UNLIKELY(mEnableInstrumentation)) {
        std::vector<void *> _hidl_args;
        _hidl_args.push_back((void *)&_hidl_out_service);
        for (const auto &callback: mInstrumentationCallbacks) {
            callback(InstrumentationEvent::SERVER_API_EXIT, "android.hidl.manager", "1.0", "IServiceManager", "get", &_hidl_args);
        }
    }
    #endif // __ANDROID_DEBUGGABLE__

    _hidl_cb(*_hidl_reply);
    return _hidl_err;
}
```

BnHwServiceManager通过ServiceManager对象查询到对应的hidl服务，返回IBase对象后，会调用toBinder函数转换为IBinder类型对象：

``` cpp
::android::hardware::toBinder< ::android::hidl::base::V1_0::IBase>(_hidl_out_service)
```

由于在hwservicemanager这边，保存的是IComposer的BpHwBase对象，因此在toBinder函数中将调用IInterface::asBinder来得到BpHwBase的成员变量中的BpHwBinder对象。
fromBinder和toBinder函数在之前的文章中已经详细分析了，这里就不再展开。
![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/display.system/Android.PG3.HW2_HIDL_hwbinder.png)

服务查询过程其实就是根据接口包名及服务名称，从hwservicemanager管理的表中查询对应的IBase服务对象，然后在Client进程空间分别创建BpHwBinder和BpHwBase对象。
**接口转换**
Framework Server进程通过上述hidl服务查询，得到了BpHwBase对象后，需要将其转换为与业务相关的代理对象，这就是通过：**Return<sp<IComposer>> castRet = IComposer::castFrom(base, true /* emitError */)**;

``` cpp
android\out\soong\.intermediates\system\libhidl\transport\manager\1.0\android.hidl.manager@1.0_genc++\gen\android\hidl\manager\1.0\ServiceManagerAll.cpp

// static 
::android::hardware::Return<::android::sp<IComposer>> IComposer::castFrom(const ::android::sp<::android::hidl::base::V1_0::IBase>& parent, bool emitError) {
    return ::android::hardware::details::castInterface<IComposer, ::android::hidl::base::V1_0::IBase, BpHwComposer>(
            parent, "android.hardware.graphics.composer@2.1::IComposer", emitError);
}
```


``` cpp
system\libhidl\transport\include\hidl\HidlTransportSupport.h
// Return nullptr if:
// 1. parent is null
// 2. cast failed because IChild is not a child type of IParent.
// 3. !emitError, calling into parent fails.
// Return an error Return object if:
// 1. emitError, calling into parent fails.
template <typename IChild, typename IParent, typename BpChild>
Return<sp<IChild>> castInterface(sp<IParent> parent, const char* childIndicator, bool emitError) {
    if (parent.get() == nullptr) {
        // casts always succeed with nullptrs.
        return nullptr;
    }
    Return<bool> canCastRet = details::canCastInterface(parent.get(), childIndicator, emitError);
    if (!canCastRet.isOk()) {
        // call fails, propagate the error if emitError
        return emitError
                ? details::StatusOf<bool, sp<IChild>>(canCastRet)
                : Return<sp<IChild>>(sp<IChild>(nullptr));
    }

    if (!canCastRet) {
        return sp<IChild>(nullptr); // cast failed.
    }
    // TODO b/32001926 Needs to be fixed for socket mode.
    if (parent->isRemote()) {
        // binderized mode. Got BpChild. grab the remote and wrap it.
        return sp<IChild>(new BpChild(toBinder<IParent>(parent)));
    }
    // Passthrough mode. Got BnChild and BsChild.
    return sp<IChild>(static_cast<IChild *>(parent.get()));
}

```
这个模板函数展开后如下：

``` cpp
Return<sp<IComposer>> castInterface(sp<IBase> parent, const char *childIndicator, bool emitError) {
    if (parent.get() == nullptr) {
        // casts always succeed with nullptrs.
        return nullptr;
    }
    Return<bool> canCastRet = details::canCastInterface(parent.get(), childIndicator, emitError);
    if (!canCastRet.isOk()) {
        // call fails, propagate the error if emitError
        return emitError
                ? details::StatusOf<bool, sp<IComposer>>(canCastRet)
                : Return<sp<IComposer>>(sp<IComposer>(nullptr));
    }
 
    if (!canCastRet) {
        return sp<IComposer>(nullptr); // cast failed.
    }
    // TODO b/32001926 Needs to be fixed for socket mode.
    if (parent->isRemote()) {
        // binderized mode. Got BpChild. grab the remote and wrap it.
        return sp<IComposer>(new BpHwComposer(toBinder<IBase, BpParent>(parent)));
    }
    // Passthrough mode. Got BnChild and BsChild.
    return sp<IComposer>(static_cast<IComposer *>(parent.get()));
}
```
因此最终会创建一个BpHwComposer对象。

new BpHwComposer(toBinder<IBase, BpParent>(parent))
![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/display.system/Android.PG3.HW2_HIDL_Get_BpHwComposer.png)


#### （三）、SurfaceFlinger 与 HWC2 通信

我们首先看看Qualcomm HWC2的一些初始化过程堆栈信息，这里不着重分析此过程，主要分析SurfaceFlinger 与 HWC2 通信。
``` cpp
sdm::StrategyImpl::StrategyImpl()                                                                                       /vendor/lib64/libsdmextension.so
sdm::CreateStrategy()                                                                                                   /vendor/lib64/libsdmextension.so
sdm::ExtensionImpl::CreateStrategyExtn()                                                                                /vendor/lib64/libsdmextension.so
sdm::Strategy::Init()                                                                                                   /hardware/qcom/msm8996/sdm/libs/core/strategy.cpp
sdm::CompManager::RegisterDisplay()                                                                                     /hardware/qcom/msm8996/sdm/libs/core/comp_manager.cpp
sdm::DisplayBase::Init()                                                                                                /hardware/qcom/msm8996/sdm/libs/core/display_base.cpp
sdm::DisplayPrimary::Init()                                                                                             /hardware/qcom/msm8996/sdm/libs/core/display_primary.cpp
sdm::CoreImpl::CreateDisplay()                                                                                          /hardware/qcom/msm8996/sdm/libs/core/core_impl.cpp
sdm::HWCDisplay::Init()                                                                                                 /hardware/qcom/msm8996/sdm/libs/hwc2/hwc_display.cpp
sdm::HWCDisplayPrimary::Init()                                                                                          /hardware/qcom/msm8996/sdm/libs/hwc2/hwc_display_primary.cpp
sdm::HWCDisplayPrimary::Create()                                                                                        /hardware/qcom/msm8996/sdm/libs/hwc2/hwc_display_primary.cpp
sdm::HWCSession::Init()                                                                                                 /hardware/qcom/msm8996/sdm/libs/hwc2/hwc_session.cpp:200
sdm::HWCSession::Open(hw_module_t const*, char const*, hw_device_t**)                                                   /hardware/qcom/msm8996/sdm/libs/hwc2/hwc_session.cpp:259
android::hardware::graphics::composer::V2_1::passthrough::HwcLoader::openDeviceWithAdapter(hw_module_t const*, bool*)   /hardware/interfaces/graphics/composer/2.1/utils/passthrough/include/composer-passthrough/2.1/HwcLoader.h
android::hardware::graphics::composer::V2_1::passthrough::HwcLoader::createHalWithAdapter(hw_module_t const*)           /hardware/interfaces/graphics/composer/2.1/utils/passthrough/include/composer-passthrough/2.1/HwcLoader.h
android::hardware::graphics::composer::V2_1::passthrough::HwcLoader::load()                                             /hardware/interfaces/graphics/composer/2.1/utils/passthrough/include/composer-passthrough/2.1/HwcLoader.h
android::hardware::PassthroughServiceManager::get() const                                                               /system/libhidl/transport/ServiceManagement.cpp
android::hardware::PassthroughServiceManager::openLibs()> const&)                                                       /system/libhidl/transport/ServiceManagement.cpp
android::hardware::PassthroughServiceManager::get()                                                                     /system/libhidl/transport/ServiceManagement.cpp
android::hardware::details::getRawServiceInternal()                                                                     /system/libhidl/transport/ServiceManagement.cpp
android::sp<android::hardware::graphics::composer::V2_1::IComposer> android::hardware::details::getServiceInternal<a>   /system/libhidl/transport/include/hidl/HidlTransportSupport.h
int android::hardware::registerPassthroughServiceImplementation<android::hardware::graphics::composer::V2_1::IComposer> /system/libhidl/transport/include/hidl/LegacySupport.h
int android::hardware::defaultPassthroughServiceImplementation<android::hardware::graphics::composer::V2_1::IComposer>()/system/libhidl/transport/include/hidl/LegacySupport.h
int android::hardware::defaultPassthroughServiceImplementation<android::hardware::graphics::composer::V2_1::IComposer>()/system/libhidl/transport/include/hidl/LegacySupport.h
main                                                                                                                    /hardware/interfaces/graphics/composer/2.1/default/service.cpp
```

##### 3.1.0 、SurfaceFlinger 与 HWC2 通信框架图
![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/display.system/Android.PG3.HW2_SurfaceFlinger.png)

##### 3.1.1 、SurfaceFlinger 合成图层
通过Log来初略看下SurfaceFlinger合并图层流程。
``` cpp
02-11 15:35:25.546 V/SurfaceFlinger(  355): handlePageFlip
02-11 15:35:25.546 V/SurfaceFlinger(  355): preComposition
02-11 15:35:25.546 V/SurfaceFlinger(  355): rebuildLayerStacks
02-11 15:35:25.546 V/SurfaceFlinger(  355): setUpHWComposer
02-11 15:35:25.546 V/SurfaceFlinger(  355): doComposition
02-11 15:35:25.546 V/SurfaceFlinger(  355): doDisplayComposition
02-11 15:35:25.546 V/SurfaceFlinger(  355): doComposeSurfaces
02-11 15:35:25.547 V/SurfaceFlinger(  355): Rendering client layers
02-11 15:35:25.547 V/SurfaceFlinger(  355): postFramebuffer
02-11 15:35:25.548 V/SurfaceFlinger(  355): postComposition
```
##### 3.2.0 、SurfaceFlinger 与 HWC2 通信流程

![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/display.system/Android.PG3.HW2_present.png)



#### （四）、HWC2 处理 HWC Layers
首先根据Log来看下Qualcomm  HWC2工作流程。

``` cpp
02-22 18:54:00.084 V/SDM     (  486): HWScale::DumpScaleData: Fetch=[0 0 0 0]  Repeat=[0 0 0 0]  roi_width = 480
02-22 18:54:00.084 D/SDM     (  486): HWDevice::Validate: *************************************************************
02-22 18:54:00.084 D/SDM     (  486): HWDevice::Validate: ******************* Layer[2] Left pipe Input ******************
02-22 18:54:00.084 D/SDM     (  486): HWDevice::Validate: in_w 512, in_h 864, in_f 45
02-22 18:54:00.084 D/SDM     (  486): HWDevice::Validate: plane_alpha 255, zorder 2, blending 2, horz_deci 0, vert_deci 0, pipe_id = 0x200, mdp_flags 0x4
02-22 18:54:00.084 V/SDM     (  486): HWDevice::Validate: src_rect [0, 0, 480, 36]
02-22 18:54:00.085 V/SDM     (  486): HWDevice::Validate: dst_rect [0, 0, 480, 36]
......
02-22 18:54:00.085 D/SDM     (  486): HWDevice::Validate: *************************************************************
02-22 18:54:00.085 D/SDM     (  486): HWDevice::Validate: ******************* Layer[3] Left pipe Input ******************
02-22 18:54:00.085 D/SDM     (  486): HWDevice::Validate: in_w 512, in_h 48, in_f 45
02-22 18:54:00.085 D/SDM     (  486): HWDevice::Validate: plane_alpha 255, zorder 3, blending 2, horz_deci 0, vert_deci 0, pipe_id = 0x20, mdp_flags 0x4
02-22 18:54:00.085 V/SDM     (  486): HWDevice::Validate: src_rect [0, 0, 480, 36]
02-22 18:54:00.085 V/SDM     (  486): HWDevice::Validate: dst_rect [0, 0, 480, 36]
02-22 18:54:00.085 D/SDM     (  486): HWScale::DumpScaleData: Scale Enable = 1
02-22 18:54:00.085 V/SDM     (  486): HWScale::DumpScaleData: Scale Data[0] : Phase=[0 200000 0 200000] Pixel_Ext=[0 0 0 0]
02-22 18:54:00.086 V/SDM     (  486): HWScale::DumpScaleData: Fetch=[0 0 0 0]  Repeat=[0 0 0 0]  roi_width = 480
......
02-22 18:54:00.086 D/SDM     (  486): HWDevice::Validate: *************************************************************
02-22 18:54:00.086 D/SDM     (  486): HWDevice::Validate: ioctl_ MSMFB_ATOMIC_COMMIT
02-22 18:54:00.087 D/SDM     (  486): StrategyImpl::Stop: 0x2000 SUCCEEDED for display = 0
02-22 18:54:00.087 I/SDM     (  486): DisplayBase::Prepare: Exiting Prepare for display type : 0
02-22 18:54:00.087 I/SDM     (  486): DisplayBase::Commit: Entering commit for display type : 0
02-22 18:54:00.087 D/SDM     (  486): HWDevice::Commit: *************************** Primary Display Device Commit Input ************************
02-22 18:54:00.087 D/SDM     (  486): HWDevice::Commit: SDE layer count is 4
02-22 18:54:00.087 D/SDM     (  486): HWDevice::Commit: ****************** Layer[0] Left pipe Input *******************
02-22 18:54:00.087 D/SDM     (  486): HWDevice::Commit: in_w 512, in_h 864, in_f 45, horz_deci 0, vert_deci 0
02-22 18:54:00.087 V/SDM     (  486): HWDevice::Commit: in_buf_fd 22, in_buf_offset 0, in_buf_stride 512, in_plane_count 1, in_fence -1, layer count 4
02-22 18:54:00.087 D/SDM     (  486): HWDevice::Commit: *************************************************************
02-22 18:54:00.087 D/SDM     (  486): HWDevice::Commit: ****************** Layer[1] Left pipe Input *******************
02-22 18:54:00.087 D/SDM     (  486): HWDevice::Commit: in_w 1280, in_h 1024, in_f 45, horz_deci 0, vert_deci 0
02-22 18:54:00.087 V/SDM     (  486): HWDevice::Commit: in_buf_fd 25, in_buf_offset 0, in_buf_stride 1280, in_plane_count 1, in_fence 26, layer count 4
02-22 18:54:00.087 D/SDM     (  486): HWDevice::Commit: *************************************************************
02-22 18:54:00.087 D/SDM     (  486): HWDevice::Commit: ****************** Layer[2] Left pipe Input *******************
02-22 18:54:00.087 D/SDM     (  486): HWDevice::Commit: in_w 512, in_h 864, in_f 45, horz_deci 0, vert_deci 0
02-22 18:54:00.087 V/SDM     (  486): HWDevice::Commit: in_buf_fd 32, in_buf_offset 0, in_buf_stride 512, in_plane_count 1, in_fence 29, layer count 4
02-22 18:54:00.087 D/SDM     (  486): HWDevice::Commit: *************************************************************
02-22 18:54:00.087 D/SDM     (  486): HWDevice::Commit: ****************** Layer[3] Left pipe Input *******************
02-22 18:54:00.087 D/SDM     (  486): HWDevice::Commit: in_w 512, in_h 48, in_f 45, horz_deci 0, vert_deci 0
02-22 18:54:00.087 V/SDM     (  486): HWDevice::Commit: in_buf_fd 28, in_buf_offset 0, in_buf_stride 512, in_plane_count 1, in_fence 34, layer count 4
02-22 18:54:00.087 D/SDM     (  486): HWDevice::Commit: *************************************************************
02-22 18:54:00.087 I/SDM     (  486): HWDevice::Commit: ioctl_ MSMFB_ATOMIC_COMMIT
02-22 18:54:00.088 I/SDM     (  486): HWDevice::Commit: *************************** Primary Display Device Commit Input ************************
02-22 18:54:00.088 I/SDM     (  486): HWDevice::Commit: retire_fence_fd 27
02-22 18:54:00.088 I/SDM     (  486): HWDevice::Commit: *******************************************************************
02-22 18:54:00.088 V/SDM     (  486): ResourceImpl::PostCommit: Resource for hw_block = 0
02-22 18:54:00.088 V/SDM     (  486): PipeAlloc::PostCommit: Pipe acquired index = 6, type = 2, pipe_id = 20
02-22 18:54:00.088 V/SDM     (  486): PipeAlloc::PostCommit: Pipe acquired index = 7, type = 2, pipe_id = 200
02-22 18:54:00.088 V/SDM     (  486): PipeAlloc::PostCommit: Pipe acquired index = 8, type = 3, pipe_id = 40
02-22 18:54:00.088 V/SDM     (  486): PipeAlloc::PostCommit: Pipe acquired index = 9, type = 3, pipe_id = 80
02-22 18:54:00.088 V/SDM     (  486): CompManager::PostCommit: registered display bit mask 0x1, configured display bit mask 0x1, display type 0
02-22 18:54:00.088 I/SDM     (  486): DisplayBase::Commit: Exiting commit for display type : 0
02-22 18:54:00.088 I/SDM     (  486): HWPrimary::SetIdleTimeoutMs: Setting idle timeout to = 70 ms

```

#### 4.1.0 、HWCDisplayPrimary::Validate

``` cpp
android/hardware/qcom/display/sdm/libs/hwc2/hwc_display_primary.cpp

HWC2::Error HWCDisplayPrimary::Validate(uint32_t *out_num_types, uint32_t *out_num_requests) {
  auto status = HWC2::Error::None;
  DisplayError error = kErrorNone;

  if (default_mode_status_ && !boot_animation_completed_) {
    ProcessBootAnimCompleted();
  }

  if (display_paused_) {
    MarkLayersForGPUBypass();
    return status;
  }

  if (color_tranform_failed_) {
    // Must fall back to client composition
    MarkLayersForClientComposition();
  }

  // Fill in the remaining blanks in the layers and add them to the SDM layerstack
  BuildLayerStack();
  // Checks and replaces layer stack for solid fill
  SolidFillPrepare();

  bool pending_output_dump = dump_frame_count_ && dump_output_to_file_;

  if (frame_capture_buffer_queued_ || pending_output_dump) {
    // RHS values were set in FrameCaptureAsync() called from a binder thread. They are picked up
    // here in a subsequent draw round.
    layer_stack_.output_buffer = &output_buffer_;
    layer_stack_.flags.post_processed_output = post_processed_output_;
  }

  bool one_updating_layer = SingleLayerUpdating();
  ToggleCPUHint(one_updating_layer);

  uint32_t refresh_rate = GetOptimalRefreshRate(one_updating_layer);
  if (current_refresh_rate_ != refresh_rate) {
    error = display_intf_->SetRefreshRate(refresh_rate);
  }

  if (error == kErrorNone) {
    // On success, set current refresh rate to new refresh rate
    current_refresh_rate_ = refresh_rate;
  }

  if (handle_idle_timeout_) {
    handle_idle_timeout_ = false;
  }

  if (layer_set_.empty()) {
    validated_ = true;
    flush_ = true;
    return status;
  }

  status = PrepareLayerStack(out_num_types, out_num_requests);
  return status;
}
```

##### 4.1.1 、HWCDisplay::BuildLayerStack()

``` cpp
android/hardware/qcom/display/sdm/libs/hwc2/hwc_display.cpp

void HWCDisplay::BuildLayerStack() {
  layer_stack_ = LayerStack();
  display_rect_ = LayerRect();
  metadata_refresh_rate_ = 0;
  auto working_primaries = ColorPrimaries_BT709_5;

  // Add one layer for fb target
  // TODO(user): Add blit target layers
  for (auto hwc_layer : layer_set_) {
    Layer *layer = hwc_layer->GetSDMLayer();
    layer->flags = {};   // Reset earlier flags
    if (hwc_layer->GetClientRequestedCompositionType() == HWC2::Composition::Client) {
      layer->flags.skip = true;
      layer_stack_.flags.client_composited_layer_present = true;
    } else if (hwc_layer->GetClientRequestedCompositionType() == HWC2::Composition::SolidColor) {
      layer->flags.solid_fill = true;
    } else if (hwc_layer->GetClientRequestedCompositionType() == HWC2::Composition::Sideband) {
      layer->flags.sideband = true;
    }

    if (!hwc_layer->ValidateAndSetCSC()) {
#ifdef FEATURE_WIDE_COLOR
      layer->flags.skip = true;
#endif
    }

    working_primaries = WidestPrimaries(working_primaries,
                                        layer->input_buffer.color_metadata.colorPrimaries);

    // set default composition as GPU for SDM
    layer->composition = kCompositionGPU;

    if (swap_interval_zero_) {
      if (layer->input_buffer.acquire_fence_fd >= 0) {
        close(layer->input_buffer.acquire_fence_fd);
        layer->input_buffer.acquire_fence_fd = -1;
      }
    }

    const private_handle_t *handle =
        reinterpret_cast<const private_handle_t *>(layer->input_buffer.buffer_id);
    if (handle) {
#ifdef USE_GRALLOC1
      if (handle->buffer_type == BUFFER_TYPE_VIDEO) {
#else
      if (handle->bufferType == BUFFER_TYPE_VIDEO) {
#endif
        layer_stack_.flags.video_present = true;
      }
      // TZ Protected Buffer - L1
      if (handle->flags & private_handle_t::PRIV_FLAGS_SECURE_BUFFER) {
        layer_stack_.flags.secure_present = true;
      }
      // Gralloc Usage Protected Buffer - L3 - which needs to be treated as Secure & avoid fallback
      if (handle->flags & private_handle_t::PRIV_FLAGS_PROTECTED_BUFFER) {
        layer_stack_.flags.secure_present = true;
      }
    }

    if (layer->flags.skip) {
      layer_stack_.flags.skip_present = true;
    }

    if (layer->flags.sideband) {
      layer_stack_.flags.sideband_present = true;
    }

    if (hwc_layer->GetClientRequestedCompositionType() == HWC2::Composition::Cursor) {
      // Currently we support only one HWCursor & only at top most z-order
      if ((*layer_set_.rbegin())->GetId() == hwc_layer->GetId()) {
        layer->flags.cursor = true;
        layer_stack_.flags.cursor_present = true;
      }
    }

    // TODO(user): Move to a getter if this is needed at other places
    hwc_rect_t scaled_display_frame = {INT(layer->dst_rect.left), INT(layer->dst_rect.top),
                                       INT(layer->dst_rect.right), INT(layer->dst_rect.bottom)};
    ApplyScanAdjustment(&scaled_display_frame);
    hwc_layer->SetLayerDisplayFrame(scaled_display_frame);
    ApplyDeInterlaceAdjustment(layer);
    // SDM requires these details even for solid fill
    if (layer->flags.solid_fill) {
      LayerBuffer *layer_buffer = &layer->input_buffer;
      layer_buffer->width = UINT32(layer->dst_rect.right - layer->dst_rect.left);
      layer_buffer->height = UINT32(layer->dst_rect.bottom - layer->dst_rect.top);
      layer_buffer->unaligned_width = layer_buffer->width;
      layer_buffer->unaligned_height = layer_buffer->height;
      layer_buffer->acquire_fence_fd = -1;
      layer_buffer->release_fence_fd = -1;
      layer->src_rect.left = 0;
      layer->src_rect.top = 0;
      layer->src_rect.right = layer_buffer->width;
      layer->src_rect.bottom = layer_buffer->height;
      // if bottom layer has the same color as border color, skip it
      if (hwc_layer != *layer_set_.begin() || (layer->solid_fill_color & 0xFFFFFF) != 0) {
        layer->flags.skip = true;
        layer_stack_.flags.skip_present = true;
      }
    }

    if (layer->frame_rate > metadata_refresh_rate_) {
      metadata_refresh_rate_ = SanitizeRefreshRate(layer->frame_rate);
    } else {
      layer->frame_rate = current_refresh_rate_;
    }
    display_rect_ = Union(display_rect_, layer->dst_rect);
    geometry_changes_ |= hwc_layer->GetGeometryChanges();

    layer->flags.updating = true;
    if (layer_set_.size() <= kMaxLayerCount) {
      layer->flags.updating = IsLayerUpdating(layer);
    }

    layer_stack_.layers.push_back(layer);
  }
```
细节暂时不分析，就是循环将layers处理后重新push_back到layer_stack_。


##### 4.1.2 、HWCDisplay::PrepareLayerStack()
接着调用了PrepareLayerStack()。
``` cpp
android/hardware/qcom/display/sdm/libs/hwc2/hwc_display.cpp

HWC2::Error HWCDisplay::PrepareLayerStack(uint32_t *out_num_types, uint32_t *out_num_requests) {
  layer_changes_.clear();
  layer_requests_.clear();
  ......
  if (!skip_prepare_) {
    DisplayError error = display_intf_->Prepare(&layer_stack_);
    if (error != kErrorNone) {
      if (error == kErrorShutDown) {
        shutdown_pending_ = true;
      } else if (error != kErrorPermission) {
        DLOGE("Prepare failed. Error = %d", error);
        // To prevent surfaceflinger infinite wait, flush the previous frame during Commit()
        // so that previous buffer and fences are released, and override the error.
        flush_ = true;
      }
      return HWC2::Error::BadDisplay;
    }
  } else {
    // Skip is not set
    MarkLayersForGPUBypass();
    skip_prepare_ = false;
    DLOGI("SecureDisplay %s, Skip Prepare/Commit and Flush",
          secure_display_active_ ? "Starting" : "Stopping");
    flush_ = true;
  }

  for (auto hwc_layer : layer_set_) {
    Layer *layer = hwc_layer->GetSDMLayer();
    LayerComposition &composition = layer->composition;

    if ((composition == kCompositionSDE) || (composition == kCompositionHybrid) ||
        (composition == kCompositionBlit) || (composition == kCompositionSideband)) {
      layer_requests_[hwc_layer->GetId()] = HWC2::LayerRequest::ClearClientTarget;
    }

    HWC2::Composition requested_composition = hwc_layer->GetClientRequestedCompositionType();
    // Set SDM composition to HWC2 type in HWCLayer
    hwc_layer->SetComposition(composition);
    HWC2::Composition device_composition  = hwc_layer->GetDeviceSelectedCompositionType();
    // Update the changes list only if the requested composition is different from SDM comp type
    // TODO(user): Take Care of other comptypes(BLIT)
    if (requested_composition != device_composition) {
      layer_changes_[hwc_layer->GetId()] = device_composition;
    }
  }
  *out_num_types = UINT32(layer_changes_.size());
  *out_num_requests = UINT32(layer_requests_.size());
  validated_ = true;
  if (*out_num_types > 0) {
    return HWC2::Error::HasChanges;
  } else {
    return HWC2::Error::None;
  }
}
```
##### 4.1.3 、display_intf_->Prepare(&layer_stack_)

``` cpp
\android\hardware\qcom\display\sdm\libs\core\display_base.cpp

DisplayError DisplayBase::Prepare(LayerStack *layer_stack) {
  lock_guard<recursive_mutex> obj(recursive_mutex_);
  DisplayError error = kErrorNone;

  ......
  error = BuildLayerStackStats(layer_stack);
  ......
  for (auto &layer : layer_stack->layers) {
    if (layer->buffer_map == nullptr) {
      layer->buffer_map = std::make_shared<LayerBufferMap>();
    }
  }
  error = HandleHDR(layer_stack);
  ......
  if (color_mgr_ && color_mgr_->NeedsPartialUpdateDisable()) {
    DisablePartialUpdateOneFrame();
  }

  if (partial_update_control_ == false || disable_pu_one_frame_) {
    comp_manager_->ControlPartialUpdate(display_comp_ctx_, false /* enable */);
    disable_pu_one_frame_ = false;
  }

  comp_manager_->PrePrepare(display_comp_ctx_, &hw_layers_);
  while (true) {
    error = comp_manager_->Prepare(display_comp_ctx_, &hw_layers_);
    ......

    error = hw_intf_->Validate(&hw_layers_);
    ......
    if (error == kErrorShutDown) {
      comp_manager_->PostPrepare(display_comp_ctx_, &hw_layers_);
      return error;
    }
  }

  comp_manager_->PostPrepare(display_comp_ctx_, &hw_layers_);

  return error;
}
```
##### 4.1.4 、hw_intf_->Validate(&hw_layers_)
调用HWDevice的Validate()，这正是log打印处来的。
``` cpp
\android\hardware\qcom\display\sdm\libs\core\fb\hw_device.cpp

DisplayError HWDevice::Validate(HWLayers *hw_layers) {
  DTRACE_SCOPED();

  DisplayError error = kErrorNone;

  HWLayersInfo &hw_layer_info = hw_layers->info;
  uint32_t hw_layer_count = UINT32(hw_layer_info.hw_layers.size());

  DLOGV_IF(kTagDriverConfig, "************************** %s Validate Input ***********************",
           device_name_);
  DLOGV_IF(kTagDriverConfig, "SDE layer count is %d", hw_layer_count);

  mdp_layer_commit_v1 &mdp_commit = mdp_disp_commit_.commit_v1;
  uint32_t &mdp_layer_count = mdp_commit.input_layer_cnt;

  DLOGI_IF(kTagDriverConfig, "left_roi: x = %d, y = %d, w = %d, h = %d", mdp_commit.left_roi.x,
    mdp_commit.left_roi.y, mdp_commit.left_roi.w, mdp_commit.left_roi.h);
  DLOGI_IF(kTagDriverConfig, "right_roi: x = %d, y = %d, w = %d, h = %d", mdp_commit.right_roi.x,
    mdp_commit.right_roi.y, mdp_commit.right_roi.w, mdp_commit.right_roi.h);

  for (uint32_t i = 0; i < hw_layer_count; i++) {
    const Layer &layer = hw_layer_info.hw_layers.at(i);
    LayerBuffer input_buffer = layer.input_buffer;
    HWPipeInfo *left_pipe = &hw_layers->config[i].left_pipe;
    HWPipeInfo *right_pipe = &hw_layers->config[i].right_pipe;
    HWRotatorSession *hw_rotator_session = &hw_layers->config[i].hw_rotator_session;
    bool is_rotator_used = (hw_rotator_session->hw_block_count != 0);
    bool is_cursor_pipe_used = (hw_layer_info.use_hw_cursor & layer.flags.cursor);

    for (uint32_t count = 0; count < 2; count++) {
      HWPipeInfo *pipe_info = (count == 0) ? left_pipe : right_pipe;
      HWRotateInfo *hw_rotate_info = &hw_rotator_session->hw_rotate_info[count];

      if (hw_rotate_info->valid) {
        input_buffer = hw_rotator_session->output_buffer;
      }

      if (pipe_info->valid) {
        mdp_input_layer &mdp_layer = mdp_in_layers_[mdp_layer_count];
        mdp_layer_buffer &mdp_buffer = mdp_layer.buffer;

        mdp_buffer.width = input_buffer.width;
        mdp_buffer.height = input_buffer.height;
        mdp_buffer.comp_ratio.denom = 1000;
        mdp_buffer.comp_ratio.numer = UINT32(hw_layers->config[i].compression * 1000);

        if (layer.flags.solid_fill) {
          mdp_buffer.format = MDP_ARGB_8888;
        } else {
          error = SetFormat(input_buffer.format, &mdp_buffer.format);
          if (error != kErrorNone) {
            return error;
          }
        }
        mdp_layer.alpha = layer.plane_alpha;
        mdp_layer.z_order = UINT16(pipe_info->z_order);
        mdp_layer.transp_mask = 0xffffffff;
        SetBlending(layer.blending, &mdp_layer.blend_op);
        mdp_layer.pipe_ndx = pipe_info->pipe_id;
        mdp_layer.horz_deci = pipe_info->horizontal_decimation;
        mdp_layer.vert_deci = pipe_info->vertical_decimation;
#ifdef MDP_COMMIT_RECT_NUM
        mdp_layer.rect_num = pipe_info->rect;
#endif
        SetRect(pipe_info->src_roi, &mdp_layer.src_rect);
        SetRect(pipe_info->dst_roi, &mdp_layer.dst_rect);
        SetMDPFlags(&layer, is_rotator_used, is_cursor_pipe_used, &mdp_layer.flags);
        SetCSC(layer.input_buffer.color_metadata, &mdp_layer.color_space);
        if (pipe_info->flags & kIGC) {
          SetIGC(&layer.input_buffer, mdp_layer_count);
        }
        if (pipe_info->flags & kMultiRect) {
          mdp_layer.flags |= MDP_LAYER_MULTIRECT_ENABLE;
          if (pipe_info->flags & kMultiRectParallelMode) {
            mdp_layer.flags |= MDP_LAYER_MULTIRECT_PARALLEL_MODE;
          }
        }
        mdp_layer.bg_color = layer.solid_fill_color;

        // HWScaleData to MDP driver
        hw_scale_->SetHWScaleData(pipe_info->scale_data, mdp_layer_count, &mdp_commit,
                                  pipe_info->sub_block_type);
        mdp_layer.scale = hw_scale_->GetScaleDataRef(mdp_layer_count, pipe_info->sub_block_type);

        mdp_layer_count++;

        DLOGV_IF(kTagDriverConfig, "******************* Layer[%d] %s pipe Input ******************",
                 i, count ? "Right" : "Left");
        DLOGV_IF(kTagDriverConfig, "in_w %d, in_h %d, in_f %d", mdp_buffer.width, mdp_buffer.height,
                 mdp_buffer.format);
        DLOGV_IF(kTagDriverConfig, "plane_alpha %d, zorder %d, blending %d, horz_deci %d, "
                 "vert_deci %d, pipe_id = 0x%x, mdp_flags 0x%x", mdp_layer.alpha, mdp_layer.z_order,
                 mdp_layer.blend_op, mdp_layer.horz_deci, mdp_layer.vert_deci, mdp_layer.pipe_ndx,
                 mdp_layer.flags);
        DLOGV_IF(kTagDriverConfig, "src_rect [%d, %d, %d, %d]", mdp_layer.src_rect.x,
                 mdp_layer.src_rect.y, mdp_layer.src_rect.w, mdp_layer.src_rect.h);
        DLOGV_IF(kTagDriverConfig, "dst_rect [%d, %d, %d, %d]", mdp_layer.dst_rect.x,
                 mdp_layer.dst_rect.y, mdp_layer.dst_rect.w, mdp_layer.dst_rect.h);
        hw_scale_->DumpScaleData(mdp_layer.scale);
        DLOGV_IF(kTagDriverConfig, "*************************************************************");
      }
    }
  }

  // TODO(user): This block should move to the derived class
  if (device_type_ == kDeviceVirtual) {
    LayerBuffer *output_buffer = hw_layers->info.stack->output_buffer;
    mdp_out_layer_.writeback_ndx = hw_resource_.writeback_index;
    mdp_out_layer_.buffer.width = output_buffer->width;
    mdp_out_layer_.buffer.height = output_buffer->height;
    if (output_buffer->flags.secure) {
      mdp_out_layer_.flags |= MDP_LAYER_SECURE_SESSION;
    }
    mdp_out_layer_.buffer.comp_ratio.denom = 1000;
    mdp_out_layer_.buffer.comp_ratio.numer = UINT32(hw_layers->output_compression * 1000);
#ifdef OUT_LAYER_COLOR_SPACE
    SetCSC(output_buffer->color_metadata, &mdp_out_layer_.color_space);
#endif
    SetFormat(output_buffer->format, &mdp_out_layer_.buffer.format);

    DLOGI_IF(kTagDriverConfig, "********************* Output buffer Info ************************");
    DLOGI_IF(kTagDriverConfig, "out_w %d, out_h %d, out_f %d, wb_id %d",
             mdp_out_layer_.buffer.width, mdp_out_layer_.buffer.height,
             mdp_out_layer_.buffer.format, mdp_out_layer_.writeback_ndx);
    DLOGI_IF(kTagDriverConfig, "*****************************************************************");
  }

  uint32_t index = 0;
  for (uint32_t i = 0; i < hw_resource_.hw_dest_scalar_info.count; i++) {
    DestScaleInfoMap::iterator it = hw_layer_info.dest_scale_info_map.find(i);

    if (it == hw_layer_info.dest_scale_info_map.end()) {
      continue;
    }

    HWDestScaleInfo *dest_scale_info = it->second;

    mdp_destination_scaler_data *dest_scalar_data = &mdp_dest_scalar_data_[index];
    hw_scale_->SetHWScaleData(dest_scale_info->scale_data, index, &mdp_commit,
                              kHWDestinationScalar);

    if (dest_scale_info->scale_update) {
      dest_scalar_data->flags |= MDP_DESTSCALER_SCALE_UPDATE;
    }

    dest_scalar_data->dest_scaler_ndx = i;
    dest_scalar_data->lm_width = dest_scale_info->mixer_width;
    dest_scalar_data->lm_height = dest_scale_info->mixer_height;
#ifdef MDP_DESTSCALER_ROI_ENABLE
    SetRect(dest_scale_info->panel_roi, &dest_scalar_data->panel_roi);
    dest_scalar_data->flags |= MDP_DESTSCALER_ROI_ENABLE;
#endif
    dest_scalar_data->scale = reinterpret_cast <uint64_t>
      (hw_scale_->GetScaleDataRef(index, kHWDestinationScalar));

    index++;

    DLOGV_IF(kTagDriverConfig, "************************ DestScalar[%d] **************************",
             dest_scalar_data->dest_scaler_ndx);
    DLOGV_IF(kTagDriverConfig, "Mixer WxH %dx%d flags %x", dest_scalar_data->lm_width,
             dest_scalar_data->lm_height, dest_scalar_data->flags);
#ifdef MDP_DESTSCALER_ROI_ENABLE
    DLOGV_IF(kTagDriverConfig, "Panel ROI [%d, %d, %d, %d]", dest_scalar_data->panel_roi.x,
             dest_scalar_data->panel_roi.y, dest_scalar_data->panel_roi.w,
             dest_scalar_data->panel_roi.h);
#endif
    DLOGV_IF(kTagDriverConfig, "*****************************************************************");
  }
  mdp_commit.dest_scaler_cnt = UINT32(hw_layer_info.dest_scale_info_map.size());

  mdp_commit.flags |= MDP_VALIDATE_LAYER;
#ifdef MDP_COMMIT_RECT_NUM
  mdp_commit.flags |= MDP_COMMIT_RECT_NUM;
#endif
  if (Sys::ioctl_(device_fd_, INT(MSMFB_ATOMIC_COMMIT), &mdp_disp_commit_) < 0) {
    if (errno == ESHUTDOWN) {
      DLOGI_IF(kTagDriverConfig, "Driver is processing shutdown sequence");
      return kErrorShutDown;
    }
    IOCTL_LOGE(MSMFB_ATOMIC_COMMIT, device_type_);
    DumpLayerCommit(mdp_disp_commit_);
    return kErrorHardware;
  }

  return kErrorNone;
}

```


#### 4.2.0 、HWCDisplayPrimary::Present

``` cpp
android/hardware/qcom/display/sdm/libs/hwc2/hwc_display_primary.cpp

HWC2::Error HWCDisplayPrimary::Present(int32_t *out_retire_fence) {
  auto status = HWC2::Error::None;
  if (display_paused_) {
    // TODO(user): From old HWC implementation
    // If we do not handle the frame set retireFenceFd to outbufAcquireFenceFd
    // Revisit this when validating display_paused
    DisplayError error = display_intf_->Flush();
    if (error != kErrorNone) {
      DLOGE("Flush failed. Error = %d", error);
    }
  } else {
    status = HWCDisplay::CommitLayerStack();
    if (status == HWC2::Error::None) {
      HandleFrameOutput();
      SolidFillCommit();
      status = HWCDisplay::PostCommitLayerStack(out_retire_fence);
    }
  }

  CloseAcquireFds();
  return status;
}

```
接着看看HWCDisplay::CommitLayerStack()函数。

##### 4.2.1 、HWCDisplay::CommitLayerStack()

``` cpp
\android\hardware\qcom\display\sdm\libs\hwc2\hwc_display.cpp
WC2::Error HWCDisplay::CommitLayerStack(void) {

  if (!validated_) {
    DLOGW("Display is not validated");
    return HWC2::Error::NotValidated;
  }

  if (shutdown_pending_ || layer_set_.empty()) {
    return HWC2::Error::None;
  }

  DumpInputBuffers();

  if (!flush_) {
    DisplayError error = kErrorUndefined;
    error = display_intf_->Commit(&layer_stack_);
    validated_ = false;

    if (error == kErrorNone) {
      // A commit is successfully submitted, start flushing on failure now onwards.
      flush_on_error_ = true;
    } else {
      if (error == kErrorShutDown) {
        shutdown_pending_ = true;
        return HWC2::Error::Unsupported;
      } else if (error != kErrorPermission) {
        DLOGE("Commit failed. Error = %d", error);
        // To prevent surfaceflinger infinite wait, flush the previous frame during Commit()
        // so that previous buffer and fences are released, and override the error.
        flush_ = true;
      }
    }
  }

  return HWC2::Error::None;
}

```
接着看看display_intf_->Commit(&layer_stack_) 函数。

##### 4.2.2 、HWCDisplay::CommitLayerStack()

``` cpp
\android\hardware\qcom\display\sdm\libs\core\display_base.cpp

DisplayError DisplayBase::Commit(LayerStack *layer_stack) {
  lock_guard<recursive_mutex> obj(recursive_mutex_);
  DisplayError error = kErrorNone;

  if (!active_) {
    pending_commit_ = false;
    return kErrorPermission;
  }

  if (!layer_stack) {
    return kErrorParameters;
  }

  if (!pending_commit_) {
    DLOGE("Commit: Corresponding Prepare() is not called for display = %d", display_type_);
    return kErrorUndefined;
  }

  pending_commit_ = false;

  // Layer stack attributes has changed, need to Reconfigure, currently in use for Hybrid Comp
  if (layer_stack->flags.attributes_changed) {
    error = comp_manager_->ReConfigure(display_comp_ctx_, &hw_layers_);
    if (error != kErrorNone) {
      return error;
    }

    error = hw_intf_->Validate(&hw_layers_);
    if (error != kErrorNone) {
        return error;
    }
  }

  CommitLayerParams(layer_stack);

  if (comp_manager_->Commit(display_comp_ctx_, &hw_layers_)) {
    if (error != kErrorNone) {
      return error;
    }
  }

  // check if feature list cache is dirty and pending.
  // If dirty, need program to hardware blocks.
  if (color_mgr_)
    error = color_mgr_->Commit();
  if (error != kErrorNone) {  // won't affect this execution path.
    DLOGW("ColorManager::Commit(...) isn't working");
  }

  error = hw_intf_->Commit(&hw_layers_);
  if (error != kErrorNone) {
    return error;
  }

  PostCommitLayerParams(layer_stack);

  if (partial_update_control_) {
    comp_manager_->ControlPartialUpdate(display_comp_ctx_, true /* enable */);
  }

  error = comp_manager_->PostCommit(display_comp_ctx_, &hw_layers_);
  if (error != kErrorNone) {
    return error;
  }

  return kErrorNone;
}
```

##### 4.2.3 、hw_intf_->Commit(&hw_layers_)

``` cpp
\android\hardware\qcom\display\sdm\libs\core\fb\hw_device.cpp
DisplayError HWDevice::Commit(HWLayers *hw_layers) {
  DTRACE_SCOPED();

  HWLayersInfo &hw_layer_info = hw_layers->info;
  uint32_t hw_layer_count = UINT32(hw_layer_info.hw_layers.size());

  DLOGV_IF(kTagDriverConfig, "*************************** %s Commit Input ************************",
           device_name_);
  DLOGV_IF(kTagDriverConfig, "SDE layer count is %d", hw_layer_count);

  mdp_layer_commit_v1 &mdp_commit = mdp_disp_commit_.commit_v1;
  uint32_t mdp_layer_index = 0;

  for (uint32_t i = 0; i < hw_layer_count; i++) {
    const Layer &layer = hw_layer_info.hw_layers.at(i);
    LayerBuffer *input_buffer = const_cast<LayerBuffer *>(&layer.input_buffer);
    HWPipeInfo *left_pipe = &hw_layers->config[i].left_pipe;
    HWPipeInfo *right_pipe = &hw_layers->config[i].right_pipe;
    HWRotatorSession *hw_rotator_session = &hw_layers->config[i].hw_rotator_session;

    for (uint32_t count = 0; count < 2; count++) {
      HWPipeInfo *pipe_info = (count == 0) ? left_pipe : right_pipe;
      HWRotateInfo *hw_rotate_info = &hw_rotator_session->hw_rotate_info[count];

      if (hw_rotate_info->valid) {
        input_buffer = &hw_rotator_session->output_buffer;
      }

      if (pipe_info->valid) {
        mdp_layer_buffer &mdp_buffer = mdp_in_layers_[mdp_layer_index].buffer;
        mdp_input_layer &mdp_layer = mdp_in_layers_[mdp_layer_index];
        if (input_buffer->planes[0].fd >= 0) {
          mdp_buffer.plane_count = 1;
          mdp_buffer.planes[0].fd = input_buffer->planes[0].fd;
          mdp_buffer.planes[0].offset = input_buffer->planes[0].offset;
          SetStride(device_type_, input_buffer->format, input_buffer->planes[0].stride,
                    &mdp_buffer.planes[0].stride);
        } else {
          mdp_buffer.plane_count = 0;
        }

        mdp_buffer.fence = input_buffer->acquire_fence_fd;
        mdp_layer_index++;

        DLOGV_IF(kTagDriverConfig, "****************** Layer[%d] %s pipe Input *******************",
                 i, count ? "Right" : "Left");
        DLOGI_IF(kTagDriverConfig, "in_w %d, in_h %d, in_f %d, horz_deci %d, vert_deci %d",
                 mdp_buffer.width, mdp_buffer.height, mdp_buffer.format, mdp_layer.horz_deci,
                 mdp_layer.vert_deci);
        DLOGI_IF(kTagDriverConfig, "in_buf_fd %d, in_buf_offset %d, in_buf_stride %d, " \
                 "in_plane_count %d, in_fence %d, layer count %d", mdp_buffer.planes[0].fd,
                 mdp_buffer.planes[0].offset, mdp_buffer.planes[0].stride, mdp_buffer.plane_count,
                 mdp_buffer.fence, mdp_commit.input_layer_cnt);
        DLOGV_IF(kTagDriverConfig, "*************************************************************");
      }
    }
  }

  // TODO(user): Move to derived class
  if (device_type_ == kDeviceVirtual) {
    LayerBuffer *output_buffer = hw_layers->info.stack->output_buffer;

    if (output_buffer->planes[0].fd >= 0) {
      mdp_out_layer_.buffer.planes[0].fd = output_buffer->planes[0].fd;
      mdp_out_layer_.buffer.planes[0].offset = output_buffer->planes[0].offset;
      SetStride(device_type_, output_buffer->format, output_buffer->planes[0].stride,
                &mdp_out_layer_.buffer.planes[0].stride);
      mdp_out_layer_.buffer.plane_count = 1;
    } else {
      DLOGE("Invalid output buffer fd");
      return kErrorParameters;
    }

    mdp_out_layer_.buffer.fence = output_buffer->acquire_fence_fd;

    DLOGI_IF(kTagDriverConfig, "********************** Output buffer Info ***********************");
    DLOGI_IF(kTagDriverConfig, "out_fd %d, out_offset %d, out_stride %d, acquire_fence %d",
             mdp_out_layer_.buffer.planes[0].fd, mdp_out_layer_.buffer.planes[0].offset,
             mdp_out_layer_.buffer.planes[0].stride,  mdp_out_layer_.buffer.fence);
    DLOGI_IF(kTagDriverConfig, "*****************************************************************");
  }

  mdp_commit.release_fence = -1;
  mdp_commit.flags &= UINT32(~MDP_VALIDATE_LAYER);
  if (synchronous_commit_) {
    mdp_commit.flags |= MDP_COMMIT_WAIT_FOR_FINISH;
  }
  if (bl_update_commit && bl_level_update_commit >= 0) {
#ifdef MDP_COMMIT_UPDATE_BRIGHTNESS
    mdp_commit.bl_level = (uint32_t)bl_level_update_commit;
    mdp_commit.flags |= MDP_COMMIT_UPDATE_BRIGHTNESS;
#endif
  }
  if (Sys::ioctl_(device_fd_, INT(MSMFB_ATOMIC_COMMIT), &mdp_disp_commit_) < 0) {
    if (errno == ESHUTDOWN) {
      DLOGI_IF(kTagDriverConfig, "Driver is processing shutdown sequence");
      return kErrorShutDown;
    }
    IOCTL_LOGE(MSMFB_ATOMIC_COMMIT, device_type_);
    DumpLayerCommit(mdp_disp_commit_);
    synchronous_commit_ = false;
    return kErrorHardware;
  }

  LayerStack *stack = hw_layer_info.stack;
  stack->retire_fence_fd = mdp_commit.retire_fence;
#ifdef VIDEO_MODE_DEFER_RETIRE_FENCE
  if (hw_panel_info_.mode == kModeVideo) {
    stack->retire_fence_fd = stored_retire_fence;
    stored_retire_fence =  mdp_commit.retire_fence;
  }
#endif
  // MDP returns only one release fence for the entire layer stack. Duplicate this fence into all
  // layers being composed by MDP.

  for (uint32_t i = 0; i < hw_layer_count; i++) {
    const Layer &layer = hw_layer_info.hw_layers.at(i);
    LayerBuffer *input_buffer = const_cast<LayerBuffer *>(&layer.input_buffer);
    HWRotatorSession *hw_rotator_session = &hw_layers->config[i].hw_rotator_session;

    if (hw_rotator_session->hw_block_count) {
      input_buffer = &hw_rotator_session->output_buffer;
      input_buffer->release_fence_fd = Sys::dup_(mdp_commit.release_fence);
      continue;
    }

    input_buffer->release_fence_fd = Sys::dup_(mdp_commit.release_fence);
  }

  hw_layer_info.sync_handle = Sys::dup_(mdp_commit.release_fence);

  DLOGI_IF(kTagDriverConfig, "*************************** %s Commit Input ************************",
           device_name_);
  DLOGI_IF(kTagDriverConfig, "retire_fence_fd %d", stack->retire_fence_fd);
  DLOGI_IF(kTagDriverConfig, "*******************************************************************");

  if (mdp_commit.release_fence >= 0) {
    Sys::close_(mdp_commit.release_fence);
  }

  if (synchronous_commit_) {
    // A synchronous commit can be requested when changing the display mode so we need to update
    // panel info.
    PopulateHWPanelInfo();
    synchronous_commit_ = false;
  }

  if (bl_update_commit)
    bl_update_commit = false;

  return kErrorNone;
}

```
可以看到log跟SDM打印出来的Log一致，说明分析大体无问题。
此时已经通过ioctl将图像数据传到Kernel驱动了，也就是上一篇分析的内容。
[Android P Graphics System（二）：Qualcomm MDSS - MDP、DSI、FrameBuffer分析]()

#### （五）、**Sys::ioctl_(device_fd_, INT(MSMFB_ATOMIC_COMMIT), &mdp_disp_commit_)**
Sys::ioctl_是Qualcomm 对原生ioctl做的一层封装，本质就是ioctl。我们来看看device_fd_是啥冬冬，从Init()函数来看。
device_fd_就是通过Sys::open_("/dev/graphics/fb0", O_RDWR);得到的，是不是比较熟悉，此处就与驱动连接起来了。
[Android P Graphics System（一）：Android Graphics精彩世界]()
``` cpp
android\hardware\qcom\display\sdm\libs\core\fb\hw_device.cpp

DisplayError HWDevice::Init() {
  // Read the fb node index
  fb_node_index_ = GetFBNodeIndex(device_type_);
  if (fb_node_index_ == -1) {
    DLOGE("device type = %d should be present", device_type_);
    return kErrorHardware;
  }

  const char *dev_name = NULL;
  vector<string> dev_paths = {"/dev/graphics/fb", "/dev/fb"};
  for (size_t i = 0; i < dev_paths.size(); i++) {
    dev_paths[i] += to_string(fb_node_index_);
    if (Sys::access_(dev_paths[i].c_str(), F_OK) >= 0) {
      dev_name = dev_paths[i].c_str();
      DLOGI("access(%s) successful", dev_name);
      break;
    }

    DLOGI("access(%s), errno = %d, error = %s", dev_paths[i].c_str(), errno, strerror(errno));
  }

  if (!dev_name) {
    DLOGE("access() failed for all possible paths");
    return kErrorHardware;
  }

  // Populate Panel Info (Used for Partial Update)
  PopulateHWPanelInfo();
  // Populate HW Capabilities
  hw_resource_ = HWResourceInfo();
  hw_info_intf_->GetHWResourceInfo(&hw_resource_);

  device_fd_ = Sys::open_(dev_name, O_RDWR);
  if (device_fd_ < 0) {
    DLOGE("open %s failed errno = %d, error = %s", dev_name, errno, strerror(errno));
    return kErrorResources;
  }

  return HWScale::Create(&hw_scale_, hw_resource_.has_qseed3);
}

```

#### （六）、Hardware Composer Sync Fences
![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/display.system/Android.PG3.HWC_1.x_sync_fences.png)

![Alt text | center](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/display.system/Android.PG3.HWC_2.x_sync_fences.png)


 Sync Fences暂不分析，以后有时间再做分析。

#### （七）、参考资料(特别感谢)：
[（1）【Android P 图形显示系统（一）硬件合成HWC2】](https://www.jianshu.com/p/824a9ddf68b9) 
[（2）【AndroidO Treble架构下Hal进程启动及HIDL服务注册过程】](https://blog.csdn.net/yangwen123/article/details/79854267) 
[（3）【Android O Treble 架构 - HIDL源代码分析】](http://charlesvincent.cc/2018/09/28/Android%20O%20Treble%20架构%20-%20HIDL源代码分析/)
 [（4）【AndroidO Treble架构下HIDL IComposer服务查询过程】](https://blog.csdn.net/yangwen123/article/details/79868548)







