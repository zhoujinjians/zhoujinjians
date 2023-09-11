---
title: Android P Graphics System（五）：GraphicBuffer和Gralloc分析（转载）
cover: https://raw.githubusercontent.com/zhoujinjiani/PicGo/master/hexo.themes/bing-wallpaper-2018.04.40.jpg
categories:
  - Graphics
tags:
  - Android
  - Graphics
toc: true
abbrlink: 20190724
date: 2019-07-24 09:25:00
---




--------------------------------------------------------------------------------

注：文章都是通过阅读各位前辈总结的资料 Android 9.0 && Linux（Kernel 3.18）Qualcomm平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除（◕‿◕），转载请注明出处（©Qualcomm ©Android @Linux 版权所有），谢谢。
（==**文章基于 Kernel-3.18**==）&&（==**文章基于 Android 9.0**==）
 [【开发板 Intrinsyc Open-Q™ 820 µSOM Development Kit】](https://www.intrinsyc.com/snapdragon-embedded-development-kits/open-q-820-usom-development-kit/)
[【开发板 Android 9.0 && Linux（Kernel 3.18）源码链接】](https://gitlab.com/zhoujinjiani/apq8096_la.um.7.5.r1-03100-8x96.0_p_v5.0)

正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢！！！

**注：本博文转载于【[夕月风](https://www.jianshu.com/u/f92447ae8445)】**
[（1）【GraphicBuffer和Gralloc分析】](https://www.jianshu.com/p/dd0b38832346)


--------------------------------------------------------------------------------
==源码（部分）==：
**frameworks/native/libs/ui/**
**drivers/staging/android/ion/msm/**
**hardware/interfaces/graphics/mapper/2.0/**

----------


####  一、GraphicBuffer定义

BufferQueue中的Buffer对象，我们用的都是GraphicBuffer，那么GraphicBuffer是怎么来的呢？接下里我们具体来看这里的流程。
Surface是Andorid窗口的描述，是ANativeWindow的实现；同样GraphicBuffer是Android中图形Buffer的描述，是ANativeWindowBuffer的实现。而一个窗口，可以有几个Buffer。

``` cpp
* frameworks/native/include/ui/GraphicBuffer.h

class GraphicBuffer
    : public ANativeObjectBase<ANativeWindowBuffer, GraphicBuffer, RefBase>,
      public Flattenable<GraphicBuffer>
{
    friend class Flattenable<GraphicBuffer>;
public:
```
其中ANativeObjectBase是一个模板类，定义如下：

``` cpp
* frameworks/native/include/ui/ANativeObjectBase.h

template <typename NATIVE_TYPE, typename TYPE, typename REF,
        typename NATIVE_BASE = android_native_base_t>
class ANativeObjectBase : public NATIVE_TYPE, public REF
{
public:
    // Disambiguate between the incStrong in REF and NATIVE_TYPE
    void incStrong(const void* id) const {
        REF::incStrong(id);
    }
    void decStrong(const void* id) const {
        REF::decStrong(id);
    }
```
这样ANativeObjectBase继承ANativeWindowBuffer和RefBase，GraphicBuffer继承ANativeObjectBase和Flattenable。
这样做的目的：

- RefBase使GraphicBuffer支持引用计数控制
- Flattenable使GraphicBuffer支持序列化。
其中的关键类 ANativeWindowBuffer，它是一个结构体，是对Native Buffer的一个描述，其定义如下：

``` cpp
* frameworks/native/libs/nativebase/include/nativebase/nativebase.h

typedef struct ANativeWindowBuffer
{
#ifdef __cplusplus
    // 构造函数，decStrong和incStrong的实现；得初始化common
#endif

    struct android_native_base_t common;

    int width;
    int height;
    int stride;
    int format;
    int usage_deprecated;
    uintptr_t layerCount;

    void* reserved[1];

    const native_handle_t* handle;
    uint64_t usage;

    void* reserved_proc[8 - (sizeof(uint64_t) / sizeof(void*))];
} ANativeWindowBuffer_t;

typedef struct ANativeWindowBuffer ANativeWindowBuffer;

// Old typedef for backwards compatibility.
typedef ANativeWindowBuffer_t android_native_buffer_t;
```
ANativeWindowBuffer中，很多属性前面我们介绍Surface时，已经介绍过了。这里重点看看这个native_handle_t。

``` cpp
* system/core/libcutils/include/cutils/native_handle.h

typedef struct native_handle
{
    int version;        /* sizeof(native_handle_t) */
    int numFds;         /* number of file-descriptors at &data[0] */
    int numInts;        /* number of ints at &data[numFds] */
    ... ...
    int data[0];        /* numFds + numInts ints */
    ... ...
} native_handle_t;

typedef const native_handle_t* buffer_handle_t;
```
native_handle_t也就是具体Buffer的句柄，根据native_handle_t就能找到护体的Buffer。这里是用文件描述符进行描述的。

GraphicBuffer，很多属性都是继承于父类的，GraphicBuffer自己的属性比较少

``` cpp
* frameworks/native/include/ui/GraphicBuffer.h

    uint8_t mOwner;

    ... ...

    GraphicBufferMapper& mBufferMapper;
    ssize_t mInitCheck;

    // numbers of fds/ints in native_handle_t to flatten
    uint32_t mTransportNumFds;
    uint32_t mTransportNumInts;

    uint64_t mId;

    // Stores the generation number of this buffer. If this number does not
    // match the BufferQueue's internal generation number (set through
    // IGBP::setGenerationNumber), attempts to attach the buffer will fail.
    uint32_t mGenerationNumber;
};
```
- mOwner
表示该GraphicBuffer持有的只是handle，还是持有具体的数据

``` cpp
   enum {
        ownNone   = 0,
        ownHandle = 1,
        ownData   = 2,
    };
```
mOwner不一样，释放时，流程不一样：

``` cpp
void GraphicBuffer::free_handle()
{
    if (mOwner == ownHandle) {
        mBufferMapper.freeBuffer(handle);
    } else if (mOwner == ownData) {
        GraphicBufferAllocator& allocator(GraphicBufferAllocator::get());
        allocator.free(handle);
    }
    handle = NULL;
}
```
- GraphicBufferMapper
GraphicBuffer实现Flattenable，可以将GraphicBuffer进行打包，在Binder中传递，但是传递只是Buffer的描述属性，并不真正去拷贝Buffer的内容。怎么实现的共享的，关键还是这里的handle。GraphicBufferMapper会根据handle去在不同的进程中进map，map到同一块物理内存。这里先埋个伏笔，后续我们会讲到。
- mId
GraphicBuffer的ID，这个ID在不同进程中都是一样的
- mGenerationNumber
可以理解问题这个buffer被用多少次了。如果这个值和BufferQueue中的mGenerationNumber不一直，那么是不能attach的。

余下，GraphicBuffer的相关函数我们接下来具体来看～


####  二、分配一块Buffer

Producer dequeueBuffer的时候，并不是 每一次都会去分配一块Buffer。还记得什么时候回去分配Buffer吗？没错，设置了标识BUFFER_NEEDS_REALLOCATION时。

``` cpp
    if (returnFlags & BUFFER_NEEDS_REALLOCATION) {
        BQ_LOGV("dequeueBuffer: allocating a new buffer for slot %d", *outSlot);
        sp<GraphicBuffer> graphicBuffer = new GraphicBuffer(
                width, height, format, BQ_LAYER_COUNT, usage,
                {mConsumerName.string(), mConsumerName.size()});
```

此时分配的Buffer，参数比较齐全，对应的构造函数为：

``` cpp
* frameworks/native/libs/ui/GraphicBuffer.cpp

GraphicBuffer::GraphicBuffer(uint32_t inWidth, uint32_t inHeight,
        PixelFormat inFormat, uint32_t inLayerCount, uint64_t usage, std::string requestorName)
    : GraphicBuffer()
{
    mInitCheck = initWithSize(inWidth, inHeight, inFormat, inLayerCount,
            usage, std::move(requestorName));
}
```
在默认构造函数中，主要是做变量是初始化：

``` cpp
GraphicBuffer::GraphicBuffer()
    : BASE(), mOwner(ownData), mBufferMapper(GraphicBufferMapper::get()),
      mInitCheck(NO_ERROR), mId(getUniqueId()), mGenerationNumber(0)
{
    width  =
    height =
    stride =
    format =
    usage_deprecated = 0;
    usage  = 0;
    layerCount = 0;
    handle = NULL;
}
```
mOwner默认是ownData。GraphicBufferMapper是一个单例类，mBufferMapper在每个进程中只有一个实际对象。inLayerCount为1，在BufferQueueProducer中是一个常量。

``` cpp
static constexpr uint32_t BQ_LAYER_COUNT = 1;

status_t GraphicBuffer::initWithSize(uint32_t inWidth, uint32_t inHeight,
        PixelFormat inFormat, uint32_t inLayerCount, uint64_t inUsage,
        std::string requestorName)
{
    GraphicBufferAllocator& allocator = GraphicBufferAllocator::get();
    uint32_t outStride = 0;
    status_t err = allocator.allocate(inWidth, inHeight, inFormat, inLayerCount,
            inUsage, &handle, &outStride, mId,
            std::move(requestorName));
    if (err == NO_ERROR) {
        mBufferMapper.getTransportSize(handle, &mTransportNumFds, &mTransportNumInts);

        width = static_cast<int>(inWidth);
        height = static_cast<int>(inHeight);
        format = inFormat;
        layerCount = inLayerCount;
        usage = inUsage;
        usage_deprecated = int(usage);
        stride = static_cast<int>(outStride);
    }
    return err;
}

```
- GraphicBufferAllocator
Buffer管理中，另外一个单例类，GraphicBufferAllocator把Buffer分配出来你，GraphicBufferMapper可以将其map到自己的进程。
需要主要的是，BufferQueueProducer是跑在SurfaceFlinger进程中的，也就是说，绝大部分的应用，使用的Buffer，都是SurfaceFlinger进程分配出来的，所以，如果SurfaceFlinger出现内存泄露，FD泄露等问题，很有可能都是应用没有释放，SurfaceFlinger不会主动释放，它只响应应用的请求。SurfaceFlinger是背锅侠！

GraphicBufferAllocator定义如下：

``` cpp
* frameworks/native/include/ui/GraphicBufferAllocator.h

class GraphicBufferAllocator : public Singleton<GraphicBufferAllocator>
{
public:
    static inline GraphicBufferAllocator& get() { return getInstance(); }

    status_t allocate(uint32_t w, uint32_t h, PixelFormat format,
            uint32_t layerCount, uint64_t usage,
            buffer_handle_t* handle, uint32_t* stride, uint64_t graphicBufferId,
            std::string requestorName);

    status_t free(buffer_handle_t handle);

    void dump(String8& res) const;
    static void dumpToSystemLog();

private:
    struct alloc_rec_t {
        uint32_t width;
        uint32_t height;
        uint32_t stride;
        PixelFormat format;
        uint32_t layerCount;
        uint64_t usage;
        size_t size;
        std::string requestorName;
    };

    static Mutex sLock;
    static KeyedVector<buffer_handle_t, alloc_rec_t> sAllocList;

    friend class Singleton<GraphicBufferAllocator>;
    GraphicBufferAllocator();
    ~GraphicBufferAllocator();

    GraphicBufferMapper& mMapper;
    const std::unique_ptr<const Gralloc2::Allocator> mAllocator;
};

```
- 两个主要的方法，一个allocate用来分配Buffer，一个free用来释放Buffe。
- sAllocList，申请的Buffer，都保存下来，放到sAllocList中，并不是保存具体的Buffer，而是Buffer的描述alloc_rec_t。
- mAllocator，Gralloc登场，gralloc采用版本化管理，用的是Gralloc2。

GraphicBufferAllocator的allocate函数如下：

``` cpp
status_t GraphicBufferAllocator::allocate(uint32_t width, uint32_t height,
        PixelFormat format, uint32_t layerCount, uint64_t usage,
        buffer_handle_t* handle, uint32_t* stride,
        uint64_t /*graphicBufferId*/, std::string requestorName)
{
    ATRACE_CALL();

    // make sure to not allocate a N x 0 or 0 x N buffer, since this is
    // allowed from an API stand-point allocate a 1x1 buffer instead.
    if (!width || !height)
        width = height = 1;

    // Ensure that layerCount is valid.
    if (layerCount < 1)
        layerCount = 1;

    Gralloc2::IMapper::BufferDescriptorInfo info = {};
    info.width = width;
    info.height = height;
    info.layerCount = layerCount;
    info.format = static_cast<Gralloc2::PixelFormat>(format);
    info.usage = usage;

    Gralloc2::Error error = mAllocator->allocate(info, stride, handle);
    if (error == Gralloc2::Error::NONE) {
        Mutex::Autolock _l(sLock);
        KeyedVector<buffer_handle_t, alloc_rec_t>& list(sAllocList);
        uint32_t bpp = bytesPerPixel(format);
        alloc_rec_t rec;
        rec.width = width;
        rec.height = height;
        rec.stride = *stride;
        rec.format = format;
        rec.layerCount = layerCount;
        rec.usage = usage;
        rec.size = static_cast<size_t>(height * (*stride) * bpp);
        rec.requestorName = std::move(requestorName);
        list.add(*handle, rec);

        return NO_ERROR;
    } else {
        ALOGE("Failed to allocate (%u x %u) layerCount %u format %d "
                "usage %" PRIx64 ": %d",
                width, height, layerCount, format, usage,
                error);
        return NO_MEMORY;
    }
}
```
在看allocate函数之前，我们先来看一下GraphicBuffer相关的类：
![Alt text | center](https://raw.githubusercontent.com/zhoujinjiani/PicGo/master/display.system/Android.PG5.GraphicBuffer-Class.png)
GraphicBuffer的左膀右臂，GraphicBufferAllocator和GraphicBufferMapper！从Android 8.0开始，Android 操作系统框架在架构方面的一项重大改变，提出了treble 项目。Vendor的实现和Androd的实现分开，Android和HAL，采用HwBinder进行通信，减少Android对HAL的直接依赖。这里的Allocator和Mapper，就是对HAL结合的包装；IAllocator，IMapper的HAL的接口。V2_1::IMapper是一个对Gralloc HAL的2.1版本。
回到allocate函数～
BufferDescriptorInfo，对Buffer的描述，在HAL层也通用。根据需要，生成BufferDescriptorInfo，再通过Gralloc2的Allocator进行allocate。allocate出来的Buffer 句柄，保存在sAllocList中。
Gralloc2 Allocator的allocate函数提供了很多形态，可以满足我们不同的要求：

``` cpp
* frameworks/native/libs/ui/include/ui/Gralloc2.h

    /*
     * The returned buffers are already imported and must not be imported
     * again.  outBufferHandles must point to a space that can contain at
     * least "count" buffer_handle_t.
     */
    Error allocate(BufferDescriptor descriptor, uint32_t count,
            uint32_t* outStride, buffer_handle_t* outBufferHandles) const;

    Error allocate(BufferDescriptor descriptor,
            uint32_t* outStride, buffer_handle_t* outBufferHandle) const
    {
        return allocate(descriptor, 1, outStride, outBufferHandle);
    }

    Error allocate(const IMapper::BufferDescriptorInfo& descriptorInfo, uint32_t count,
            uint32_t* outStride, buffer_handle_t* outBufferHandles) const
    {
        BufferDescriptor descriptor;
        Error error = mMapper.createDescriptor(descriptorInfo, &descriptor);
        if (error == Error::NONE) {
            error = allocate(descriptor, count, outStride, outBufferHandles);
        }
        return error;
    }

    Error allocate(const IMapper::BufferDescriptorInfo& descriptorInfo,
            uint32_t* outStride, buffer_handle_t* outBufferHandle) const
    {
        return allocate(descriptorInfo, 1, outStride, outBufferHandle);
    }
```
我们传的参数是BufferDescriptorInfo，首先要根据BufferDescriptorInfo，生成一个BufferDescriptor，这个是mapper的HAL层实现的，因为这个BufferDescriptor最后也是要给到HAL层，HAL层根据BufferDescriptor去生成相应描述的Buffer。
最后，allocate的通用实现如下：

``` cpp
* frameworks/native/libs/ui/Gralloc2.cpp

Error Allocator::allocate(BufferDescriptor descriptor, uint32_t count,
        uint32_t* outStride, buffer_handle_t* outBufferHandles) const
{
    Error error;
    auto ret = mAllocator->allocate(descriptor, count,
            [&](const auto& tmpError, const auto& tmpStride,
                const auto& tmpBuffers) {
                error = tmpError;
                if (tmpError != Error::NONE) {
                    return;
                }

                // import buffers
                for (uint32_t i = 0; i < count; i++) {
                    error = mMapper.importBuffer(tmpBuffers[i],
                            &outBufferHandles[i]);
                    if (error != Error::NONE) {
                        for (uint32_t j = 0; j < i; j++) {
                            mMapper.freeBuffer(outBufferHandles[j]);
                            outBufferHandles[j] = nullptr;
                        }
                        return;
                    }
                }

                *outStride = tmpStride;
            });

    // make sure the kernel driver sees BC_FREE_BUFFER and closes the fds now
    hardware::IPCThreadState::self()->flushCommands();

    return (ret.isOk()) ? error : kTransactionError;
}
```
count，表示需要分配的Buffer个数，也就是说我们一次可以分配多个Buffer。

allocator分配完成后，再通过importBuffer函数，import到我们的handle中outBufferHandle。

``` cpp
* frameworks/native/libs/ui/Gralloc2.cpp

Error Mapper::importBuffer(const hardware::hidl_handle& rawHandle,
        buffer_handle_t* outBufferHandle) const
{
    Error error;
    auto ret = mMapper->importBuffer(rawHandle,
            [&](const auto& tmpError, const auto& tmpBuffer)
            {
                error = tmpError;
                if (error != Error::NONE) {
                    return;
                }

                *outBufferHandle = static_cast<buffer_handle_t>(tmpBuffer);
            });

    return (ret.isOk()) ? error : kTransactionError;
}
```
#### Gralloc1.0 接口介绍
Graphic相关的HAL的接口都在定义在hardware/interfaces/graphics/。allocator和mapper也是分开的。

#### IAllocator接口

``` cpp
* hardware/interfaces/graphics/allocator/2.0/IAllocator.hal

package android.hardware.graphics.allocator@2.0;

import android.hardware.graphics.mapper@2.0;

interface IAllocator {
    @entry
    @exit
    @callflow(next="*")
    dumpDebugInfo() generates (string debugInfo);

    @entry
    @exit
    @callflow(next="*")
    allocate(BufferDescriptor descriptor, uint32_t count)
        generates (Error error,
                   uint32_t stride,
                   vec<handle> buffers);
};
```
IAllocator主要两个接口：

- allocate
根据Buffer Descriptor描述的属性，分配对应的Buffer；count，分配的个数；返回值，stride，Buffer 步长，何为步长？我们知道Buffer都有一个宽度，但是Buffer的内存中分配的时候，都是采用对齐后的大小。多少位对齐，每个硬件平台不一样。比如，我们在一个32对齐的平台上，需要申请一块60x60大小的Buffer。因为要做对齐，所以实际分配的大小为64x60。那么对于这块Buffer，stride就是64。这是因为我们读Buffer的时候，基本都是一行一行的读的，我们要读i行j列，也就是base + i*stride + j的位置。在有的场合下，高也会要求做对齐，那么60x60的Buffer，实际分配的大小是64x64的。buffers这是分配的Buffer的handle了。
- dumpDebugInfo
dump函数，主要用来debug用

所以，IAllocator的接口主要就一个allocate。
IAllocator又是怎么跟HAL模块连接上的呢？其实一个hidl的接口，在编译时会生成很多东西～

``` cpp
hidl_interface {
    name: "android.hardware.graphics.allocator@2.0",
    root: "android.hardware",
    vndk: {
        enabled: true,
    },
    srcs: [
        "IAllocator.hal",
    ],
    interfaces: [
        "android.hardware.graphics.common@1.0",
        "android.hardware.graphics.mapper@2.0",
        "android.hidl.base@1.0",
    ],
    gen_java: false,
}
```
IAllocator的目录如下：

``` cpp
out/soong/.intermediates/hardware/interfaces/graphics/allocator/2.0

./android.hardware.graphics.allocator@2.0_genc++_headers/gen/android/hardware/graphics/allocator/2.0/IAllocator.h
./android.hardware.graphics.allocator@2.0_genc++/gen/android/hardware/graphics/allocator/2.0/AllocatorAll.cpp
```
Gralloc2的构造函数中，将首先建立和HAL层的HwBinder服务连接

``` cpp
* frameworks/native/libs/ui/Gralloc2.cpp

Allocator::Allocator(const Mapper& mapper)
    : mMapper(mapper)
{
    mAllocator = IAllocator::getService();
    if (mAllocator == nullptr) {
        LOG_ALWAYS_FATAL("gralloc-alloc is missing");
    }
}
```
IAllocator的getService函数，是.hal文件中是没有定义的，但是编译的中间结果中会生成。

``` cpp
* out/soong/.intermediates/hardware/interfaces/graphics/allocator/2.0/android.hardware.graphics.allocator@2.0_genc++_headers/gen/android/hardware/graphics/allocator/2.0/IAllocator.h

static ::android::sp<IAllocator> getService(const std::string &serviceName="default", bool getStub=false);
```
这里用的是缺省构造函数，这里其实和Binder是类似的：


``` cpp
* out/soong/.intermediates/hardware/interfaces/graphics/allocator/2.0/android.hardware.graphics.allocator@2.0_genc++/gen/android/hardware/graphics/allocator/2.0/AllocatorAll.cpp

// static
::android::sp<IAllocator> IAllocator::getService(const std::string &serviceName, const bool getStub) {
    return ::android::hardware::details::getServiceInternal<BpHwAllocator>(serviceName, true, getStub);
}
```
注册的函数如下：


``` cpp
::android::status_t IAllocator::registerAsService(const std::string &serviceName) {
    ::android::hardware::details::onRegistration("android.hardware.graphics.allocator@2.0", "IAllocator", serviceName);

    const ::android::sp<::android::hidl::manager::V1_0::IServiceManager> sm
            = ::android::hardware::defaultServiceManager();
    if (sm == nullptr) {
        return ::android::INVALID_OPERATION;
    }
    ::android::hardware::Return<bool> ret = sm->add(serviceName.c_str(), this);
    return ret.isOk() && ret ? ::android::OK : ::android::UNKNOWN_ERROR;
}
```
IAllocator HAL服务是谁呢？默认的实现在这里：

``` cpp
hardware/interfaces/graphics/allocator/2.0/default
```

默认服务起来的时候，将通过defaultPassthroughServiceImplementation去注册IAllocator的HAL服务：

``` cpp
#define LOG_TAG "android.hardware.graphics.allocator@2.0-service"

#include <android/hardware/graphics/allocator/2.0/IAllocator.h>

#include <hidl/LegacySupport.h>

using android::hardware::graphics::allocator::V2_0::IAllocator;
using android::hardware::defaultPassthroughServiceImplementation;

int main() {
    return defaultPassthroughServiceImplementation<IAllocator>(4);
}
```
defaultPassthroughServiceImplementation的实现在LegacySupport.h中

``` cpp
* system/libhidl/transport/include/hidl/LegacySupport.h

template<class Interface>
__attribute__((warn_unused_result))
status_t registerPassthroughServiceImplementation(
        std::string name = "default") {
    sp<Interface> service = Interface::getService(name, true /* getStub */);

    if (service == nullptr) {
        ALOGE("Could not get passthrough implementation for %s/%s.",
            Interface::descriptor, name.c_str());
        return EXIT_FAILURE;
    }

    LOG_FATAL_IF(service->isRemote(), "Implementation of %s/%s is remote!",
            Interface::descriptor, name.c_str());

    status_t status = service->registerAsService(name);

    if (status == OK) {
        ALOGI("Registration complete for %s/%s.",
            Interface::descriptor, name.c_str());
    } else {
        ALOGE("Could not register service %s/%s (%d).",
            Interface::descriptor, name.c_str(), status);
    }

    return status;
}

template<class Interface>
__attribute__((warn_unused_result))
status_t defaultPassthroughServiceImplementation(std::string name,
                                            size_t maxThreads = 1) {
    configureRpcThreadpool(maxThreads, true);
    status_t result = registerPassthroughServiceImplementation<Interface>(name);

    if (result != OK) {
        return result;
    }

    joinRpcThreadpool();
    return UNKNOWN_ERROR;
}
template<class Interface>
__attribute__((warn_unused_result))
status_t defaultPassthroughServiceImplementation(size_t maxThreads = 1) {
    return defaultPassthroughServiceImplementation<Interface>("default", maxThreads);
}

```
IAllocator被注册为Passthrough的Service。registerAsService，看看前面的函数，这个service将会被add到IServiceManager中，这这样，get的时候，就能获取到了。
获取到service的时，将会调HIDL_FETCH_***的函数，我们这里就是HIDL_FETCH_IAllocator，中间过程都是在system/libhidl中实现的。这里就不细跟了。
HIDL_FETCH_IAllocator的函数实现如下：

``` cpp
* hardware/interfaces/graphics/allocator/2.0/default/Gralloc.cpp

IAllocator* HIDL_FETCH_IAllocator(const char* /* name */) {
    const hw_module_t* module = nullptr;
    int err = hw_get_module(GRALLOC_HARDWARE_MODULE_ID, &module);
    if (err) {
        ALOGE("failed to get gralloc module");
        return nullptr;
    }

    uint8_t major = (module->module_api_version >> 8) & 0xff;
    switch (major) {
        case 1:
            return new Gralloc1Allocator(module);
        case 0:
            return new Gralloc0Allocator(module);
        default:
            ALOGE("unknown gralloc module major version %d", major);
            return nullptr;
    }
}
```
HIDL_FETCH时，将会加载对应的HAL实现了。gralloc这边的HAL实现GRALLOC_HARDWARE_MODULE_ID。major就是gralloc的API版本。Gralloc1Allocator是对1.0版本的适配，Gralloc0Allocator是对最初版本的适配。
#### Gralloc1 Allocator HAL层接口
大多数Hardware的接口都定义在hardware/libhardware/include/hardware，Gralloc也不例外。
Gralloc1，HAL的描述为gralloc1_device_t

``` cpp
* hardware/libhardware/include/hardware/gralloc1.h

typedef struct gralloc1_device {
    /* Must be the first member of this struct, since a pointer to this struct
     * will be generated by casting from a hw_device_t* */
    struct hw_device_t common;

    // 获取Devices支持的能力
    void (*getCapabilities)(struct gralloc1_device* device, uint32_t* outCount,
            int32_t* /*gralloc1_capability_t*/ outCapabilities);

    // 获取对应功能的函数指针
    gralloc1_function_pointer_t (*getFunction)(struct gralloc1_device* device,
            int32_t /*gralloc1_function_descriptor_t*/ descriptor);
} gralloc1_device_t;

```
Gralloc1和前面的的实现有比较大的差别，接口都通过函数指针实现，不再采用原来的方式。

下面是Gralloc0的定义：

``` cpp
* hardware/libhardware/include/hardware/gralloc.h

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

Gralloc0中，还是采用直接的函数调用。Gralloc1中，只是getCapabilities采用直接的函数调用。
Gralloc1时，走的Gralloc1Allocator，Gralloc0时，走的Gralloc0Allocator。我们主要来看一下Gralloc1Allocator。

``` cpp
* hardware/interfaces/graphics/allocator/2.0/default/Gralloc1Allocator.cpp

Gralloc1Allocator::Gralloc1Allocator(const hw_module_t* module)
    : mDevice(nullptr), mCapabilities(), mDispatch() {
    int result = gralloc1_open(module, &mDevice);
    if (result) {
        LOG_ALWAYS_FATAL("failed to open gralloc1 device: %s",
                         strerror(-result));
    }

    initCapabilities();
    initDispatch();
}
```
gralloc1_open，打开HAL层Gralloc1的具体实现。获取到gralloc1_device_t设备mDevice。

通过initCapabilities函数，将Gralloc1的能力都读出来，放到capabilities中

``` cpp
* hardware/interfaces/graphics/allocator/2.0/default/Gralloc1Allocator.cpp

void Gralloc1Allocator::initCapabilities() {
    uint32_t count = 0;
    mDevice->getCapabilities(mDevice, &count, nullptr);

    std::vector<int32_t> capabilities(count);
    mDevice->getCapabilities(mDevice, &count, capabilities.data());
    capabilities.resize(count);

    for (auto capability : capabilities) {
        if (capability == GRALLOC1_CAPABILITY_LAYERED_BUFFERS) {
            mCapabilities.layeredBuffers = true;
            break;
        }
    }
}
```
mDevice的getCapabilities函数调了两次，这个在HAL实现中经常用到，第一次，主要是获取大小，第二次才去获取具体的值。

initDispatch初始化函数指针，

``` cpp
* hardware/interfaces/graphics/allocator/2.0/default/Gralloc1Allocator.cpp

template <typename T>
void Gralloc1Allocator::initDispatch(gralloc1_function_descriptor_t desc,
                                     T* outPfn) {
    auto pfn = mDevice->getFunction(mDevice, desc);
    if (!pfn) {
        LOG_ALWAYS_FATAL("failed to get gralloc1 function %d", desc);
    }

    *outPfn = reinterpret_cast<T>(pfn);
}

void Gralloc1Allocator::initDispatch() {
    initDispatch(GRALLOC1_FUNCTION_DUMP, &mDispatch.dump);
    initDispatch(GRALLOC1_FUNCTION_CREATE_DESCRIPTOR,
                 &mDispatch.createDescriptor);
    initDispatch(GRALLOC1_FUNCTION_DESTROY_DESCRIPTOR,
                 &mDispatch.destroyDescriptor);
    initDispatch(GRALLOC1_FUNCTION_SET_DIMENSIONS, &mDispatch.setDimensions);
    initDispatch(GRALLOC1_FUNCTION_SET_FORMAT, &mDispatch.setFormat);
    if (mCapabilities.layeredBuffers) {
        initDispatch(GRALLOC1_FUNCTION_SET_LAYER_COUNT,
                     &mDispatch.setLayerCount);
    }
    initDispatch(GRALLOC1_FUNCTION_SET_CONSUMER_USAGE,
                 &mDispatch.setConsumerUsage);
    initDispatch(GRALLOC1_FUNCTION_SET_PRODUCER_USAGE,
                 &mDispatch.setProducerUsage);
    initDispatch(GRALLOC1_FUNCTION_GET_STRIDE, &mDispatch.getStride);
    initDispatch(GRALLOC1_FUNCTION_ALLOCATE, &mDispatch.allocate);
    initDispatch(GRALLOC1_FUNCTION_RELEASE, &mDispatch.release);
}
```

mDevice根据gralloc1_function_descriptor_t，去HAL的实现中去获取对应的函数指针，初始化到mDispatch中。以后我们直接调mDispatch中的函数就访问到HAL的实现。
Gralloc1的gralloc1_function_descriptor_t包括：

``` cpp
* hardware/libhardware/include/hardware/gralloc1.h

typedef enum {
    GRALLOC1_FUNCTION_INVALID = 0,
    GRALLOC1_FUNCTION_DUMP = 1,
    GRALLOC1_FUNCTION_CREATE_DESCRIPTOR = 2,
    GRALLOC1_FUNCTION_DESTROY_DESCRIPTOR = 3,
    GRALLOC1_FUNCTION_SET_CONSUMER_USAGE = 4,
    GRALLOC1_FUNCTION_SET_DIMENSIONS = 5,
    GRALLOC1_FUNCTION_SET_FORMAT = 6,
    GRALLOC1_FUNCTION_SET_PRODUCER_USAGE = 7,
    GRALLOC1_FUNCTION_GET_BACKING_STORE = 8,
    GRALLOC1_FUNCTION_GET_CONSUMER_USAGE = 9,
    GRALLOC1_FUNCTION_GET_DIMENSIONS = 10,
    GRALLOC1_FUNCTION_GET_FORMAT = 11,
    GRALLOC1_FUNCTION_GET_PRODUCER_USAGE = 12,
    GRALLOC1_FUNCTION_GET_STRIDE = 13,
    GRALLOC1_FUNCTION_ALLOCATE = 14,
    GRALLOC1_FUNCTION_RETAIN = 15,
    GRALLOC1_FUNCTION_RELEASE = 16,
    GRALLOC1_FUNCTION_GET_NUM_FLEX_PLANES = 17,
    GRALLOC1_FUNCTION_LOCK = 18,
    GRALLOC1_FUNCTION_LOCK_FLEX = 19,
    GRALLOC1_FUNCTION_UNLOCK = 20,
    GRALLOC1_FUNCTION_SET_LAYER_COUNT = 21,
    GRALLOC1_FUNCTION_GET_LAYER_COUNT = 22,
    GRALLOC1_LAST_FUNCTION = 22,
} gralloc1_function_descriptor_t;
```
IAllocator需要实现的gralloc1_function_descriptor_t包括：

``` cpp
* hardware/interfaces/graphics/allocator/2.0/default/Gralloc1Allocator.h
    struct {
        GRALLOC1_PFN_DUMP dump;
        GRALLOC1_PFN_CREATE_DESCRIPTOR createDescriptor;
        GRALLOC1_PFN_DESTROY_DESCRIPTOR destroyDescriptor;
        GRALLOC1_PFN_SET_DIMENSIONS setDimensions;
        GRALLOC1_PFN_SET_FORMAT setFormat;
        GRALLOC1_PFN_SET_LAYER_COUNT setLayerCount;
        GRALLOC1_PFN_SET_CONSUMER_USAGE setConsumerUsage;
        GRALLOC1_PFN_SET_PRODUCER_USAGE setProducerUsage;
        GRALLOC1_PFN_GET_STRIDE getStride;
        GRALLOC1_PFN_ALLOCATE allocate;
        GRALLOC1_PFN_RELEASE release;
    } mDispatch;
```
我们去实现Gralloc1的HAL时，allocator只去要实现getCapabilities和上面mDispatch中的gralloc1_function_descriptor_t就可以了。

#### IMapper接口
IMapper的接口有两个版本2.0和2.1：

``` cpp
ls hardware/interfaces/graphics/mapper

2.0  2.1
```
2.1是可选的，暂时不严格要求支持：

``` cpp
* frameworks/native/libs/ui/Gralloc2.cpp

void Mapper::preload() {
    android::hardware::preloadPassthroughService<hardware::graphics::mapper::V2_0::IMapper>();
}

Mapper::Mapper()
{
    mMapper = IMapper::getService();
    if (mMapper == nullptr) {
        LOG_ALWAYS_FATAL("gralloc-mapper is missing");
    }
    if (mMapper->isRemote()) {
        LOG_ALWAYS_FATAL("gralloc-mapper must be in passthrough mode");
    }

    // IMapper 2.1 is optional
    mMapperV2_1 = hardware::graphics::mapper::V2_1::IMapper::castFrom(mMapper);
}
```
IMapper也是PassThrough的模式。

#### IMapper2.0的接口

``` cpp
* hardware/interfaces/graphics/mapper/2.0/IMapper.hal

package android.hardware.graphics.mapper@2.0;

import android.hardware.graphics.common@1.0;

interface IMapper {
    struct BufferDescriptorInfo {
        uint32_t width; // 宽，横向的像素点数

        uint32_t height; //高，纵向的像素点数

       /**
        * The number of image layers that must be in the allocated buffer.
        */
        uint32_t layerCount;

        PixelFormat format; //像素点的格式

        bitfield<BufferUsage> usage; //用处
    };

    struct Rect {
        int32_t left;
        int32_t top;
        int32_t width;
        int32_t height;
    };

    /**
     * 创建一个Buffer的描述，这个描述在分配Buffer时使用
     * 如果成功，返回值为NONE, 参数无效或冲突返回BAD_VALUE，没有资源返回NO_RESOURCES，参数不支持返回UNSUPPORTED
     */
    @entry
    @callflow(next="*")
    createDescriptor(BufferDescriptorInfo descriptorInfo)
          generates (Error error,
                     BufferDescriptor descriptor);

    @entry
    @callflow(next="*")
    importBuffer(handle rawHandle) generates (Error error, pointer buffer);

    @exit
    @callflow(next="*")
    freeBuffer(pointer buffer) generates (Error error);

    @callflow(next="unlock")
    lock(pointer buffer,
         bitfield<BufferUsage> cpuUsage,
         Rect accessRegion,
         handle acquireFence)
        generates (Error error,
                   pointer data);

    @callflow(next="unlock")
    lockYCbCr(pointer buffer,
              bitfield<BufferUsage> cpuUsage,
              Rect accessRegion,
              handle acquireFence)
        generates (Error error,
                   YCbCrLayout layout);

    @callflow(next="*")
    unlock(pointer buffer)
        generates (Error error,
                   handle releaseFence);
};
```

IMapper的接口比IAllocator多，具体一些信息我写在代码中。

- createDescriptor
创建一个BufferDescriptor，分配Buffer时，根据Descriptor分配。
- importBuffer
Buffer被冲其他进程或HAL克隆出来时，这个Buffer是RAW状态的Buffer，raw handle是不能直接访问真正的Buffer的，我们需要把它imported到imported的handle中才能访问。创建imported handle时，需要验证raw handle的有效性，且raw handle需要能多少import创建多个imported handle。在passthrough HALs中，从HAL接收到的handle，可能已经被import到进程中，这个时候要能区分，将其当做raw handle处理，而不是返回BAD_BUFFER。
- freeBuffer
释放Buffer handle，通过importBuffer返回的handle必现通过这个接口释放。importBuffer时申请的所有资源必须一起释放。比如 imported handle如果通过native_handle_create创建的，那么必须调用native_handle_close和native_handle_delete
- lock
将Buffer锁住，用来做制定的处理。多线程可以同事lock，但是不能同时写。超出accessRegion区域的Buffer不能写，超出的区域不受保护。Buffer的地址是指针buffer，是从left-top开始的，即使accessRegion不是left-top描述。
- lockYCbCr
这个lock很相似，只是返回值不一样，这里是YCbCrLayout。除非是Codec配置为flexible-YUV-compatible的颜色格式，要不必现是PixelFormat::YCbCr_*_888格式的。
- unlock
表示CPU访问Buffer已经完成

IMapper用到的数据类型定义在types.hal中

``` cpp
* hardware/interfaces/graphics/mapper/2.0/types.hal

package android.hardware.graphics.mapper@2.0;

enum Error : int32_t {
    NONE            = 0, /** no error */
    BAD_DESCRIPTOR  = 1, /** invalid BufferDescriptor */
    BAD_BUFFER      = 2, /** invalid buffer handle */
    BAD_VALUE       = 3, /** invalid width, height, etc. */
    /* 4 is reserved */
    NO_RESOURCES    = 5, /** temporary failure due to resource contention */
    /* 6 is reserved */
    UNSUPPORTED     = 7, /** permanent failure */
};

/**
 * A buffer descriptor is an implementation-defined opaque data returned by
 * createDescriptor. It describes the properties of a buffer and is consumed
 * by the allocator.
 */
typedef vec<uint32_t> BufferDescriptor;

/**
 * Structure for describing YCbCr formats for consumption by applications.
 * This is used with PixelFormat::YCBCR_*_888.
 *
 * Buffer chroma subsampling is defined in the format.
 * e.g. PixelFormat::YCBCR_420_888 has subsampling 4:2:0.
 *
 * Buffers must have a 8 bit depth.
 *
 * y, cb, and cr point to the first byte of their respective planes.
 *
 * Stride describes the distance in bytes from the first value of one row of
 * the image to the first value of the next row. It includes the width of the
 * image plus padding.
 * yStride is the stride of the luma plane.
 * cStride is the stride of the chroma planes.
 *
 * chromaStep is the distance in bytes from one chroma pixel value to the
 * next. This is 2 bytes for semiplanar (because chroma values are interleaved
 * and each chroma value is one byte) and 1 for planar.
 */
struct YCbCrLayout {
    pointer y;
    pointer cb;
    pointer cr;
    uint32_t yStride;
    uint32_t cStride;
    uint32_t chromaStep;
};
```
#### IMapper2.1的接口
IMapper2.1的接口继承IMapper2.0的接口：

``` cpp
* hardware/interfaces/graphics/mapper/2.1/IMapper.hal

package android.hardware.graphics.mapper@2.1;

import android.hardware.graphics.mapper@2.0::Error;
import android.hardware.graphics.mapper@2.0::IMapper;

interface IMapper extends android.hardware.graphics.mapper@2.0::IMapper {

    validateBufferSize(pointer buffer,
                       BufferDescriptorInfo descriptorInfo,
                       uint32_t stride)
            generates (Error error);

    /**
     * Get the transport size of a buffer. An imported buffer handle is a raw
     * buffer handle with the process-local runtime data appended. This
     * function, for example, allows a caller to omit the process-local
     * runtime data at the tail when serializing the imported buffer handle.
     *
     * Note that a client might or might not omit the process-local runtime
     * data when sending an imported buffer handle. The mapper must support
     * both cases on the receiving end.
     *
     * @param buffer is the buffer to get the transport size from.
     * @return error is NONE upon success. Otherwise,
     *                  BAD_BUFFER when the buffer is invalid.
     * @return numFds is the number of file descriptors needed for transport.
     * @return numInts is the number of integers needed for transport.
     */
    getTransportSize(pointer buffer)
            generates (Error error,
                       uint32_t numFds,
                       uint32_t numInts);
};
```
- validateBufferSize
验证，Buffer能不能被制定的描述信息和步长的访问者访问。
- getTransportSize
获取Buffer传输的大小。一个Imported handle是一个raw handle再加上进程本地运行的数据，所以我们可以获取到进程本地的数据。

IMapper的HIDL_FETCH_IMapper函数实现如下：

``` cpp
* hardware/interfaces/graphics/mapper/2.0/default/GrallocMapper.cpp

IMapper* HIDL_FETCH_IMapper(const char* /* name */) {
    const hw_module_t* module = nullptr;
    int err = hw_get_module(GRALLOC_HARDWARE_MODULE_ID, &module);
    if (err) {
        ALOGE("failed to get gralloc module");
        return nullptr;
    }

    uint8_t major = (module->module_api_version >> 8) & 0xff;
    switch (major) {
        case 1:
            return new Gralloc1Mapper(module);
        case 0:
            return new Gralloc0Mapper(module);
        default:
            ALOGE("unknown gralloc module major version %d", major);
            return nullptr;
    }
}
```

IMapper的HAL模块ID为GRALLOC_HARDWARE_MODULE_ID，和IAllocator类似，这里也对应两个mapper。Gralloc1Mapper和Gralloc0Mapper
前面IAllocator的时候，有个main函数，这里为什么没有？那么，IMapper是怎么找到的呢？
夏雨荷已死，还记得Gralloc2中的preload吗？

``` cpp
* frameworks/native/libs/ui/Gralloc2.cpp

void Mapper::preload() {
    android::hardware::preloadPassthroughService<hardware::graphics::mapper::V2_0::IMapper>();
}
```
#### Gralloc1 Mapper HAL层接口
Mapper的接口也定义在gralloc1.h中，Gralloc1对应的Mapper为Gralloc1Mapper


``` cpp
* hardware/interfaces/graphics/allocator/2.0/default/Gralloc1On0Adapter.cpp

Gralloc1Mapper::Gralloc1Mapper(const hw_module_t* module)
    : mDevice(nullptr), mDispatch() {
    int result = gralloc1_open(module, &mDevice);
    if (result) {
        LOG_ALWAYS_FATAL("failed to open gralloc1 device: %s",
                         strerror(-result));
    }

    initCapabilities();
    initDispatch();
}
```
mapper的initCapabilities函数，和allocator的initCapabilities函数类似，都是通过gralloc1_device_t的getCapabilities函数去获取。只是这里但是做了一个封装了mCapabilities。


``` cpp
void Gralloc1Mapper::initCapabilities() {
    mCapabilities.highUsageBits = true;
    mCapabilities.layeredBuffers = false;
    mCapabilities.unregisterImplyDelete = false;

    uint32_t count = 0;
    mDevice->getCapabilities(mDevice, &count, nullptr);

    std::vector<int32_t> capabilities(count);
    mDevice->getCapabilities(mDevice, &count, capabilities.data());
    capabilities.resize(count);

    for (auto capability : capabilities) {
        switch (capability) {
            case GRALLOC1_CAPABILITY_LAYERED_BUFFERS:
                mCapabilities.layeredBuffers = true;
                break;
            case GRALLOC1_CAPABILITY_RELEASE_IMPLY_DELETE:
                mCapabilities.unregisterImplyDelete = true;
                break;
        }
    }
}
```
只是这里但是做了一个封装了mCapabilities。

``` cpp
    struct {
        bool highUsageBits;
        bool layeredBuffers;
        bool unregisterImplyDelete;
    } mCapabilities = {};
```
Gralloc1，中Mapper 对应的gralloc1_function_descriptor_t如下：

``` cpp
template <typename T>
void Gralloc1Mapper::initDispatch(gralloc1_function_descriptor_t desc,
                                  T* outPfn) {
    auto pfn = mDevice->getFunction(mDevice, desc);
    if (!pfn) {
        LOG_ALWAYS_FATAL("failed to get gralloc1 function %d", desc);
    }

    *outPfn = reinterpret_cast<T>(pfn);
}

void Gralloc1Mapper::initDispatch() {
    initDispatch(GRALLOC1_FUNCTION_RETAIN, &mDispatch.retain);
    initDispatch(GRALLOC1_FUNCTION_RELEASE, &mDispatch.release);
    initDispatch(GRALLOC1_FUNCTION_GET_NUM_FLEX_PLANES,
                 &mDispatch.getNumFlexPlanes);
    initDispatch(GRALLOC1_FUNCTION_LOCK, &mDispatch.lock);
    initDispatch(GRALLOC1_FUNCTION_LOCK_FLEX, &mDispatch.lockFlex);
    initDispatch(GRALLOC1_FUNCTION_UNLOCK, &mDispatch.unlock);
}
```
mapper的HAL实现，只要实现上述的gralloc1_function_descriptor_t。

我们来看一下Gralloc1的相关类似关系～


![Alt text | center](https://raw.githubusercontent.com/zhoujinjiani/PicGo/master/display.system/Android.PG5.gralloc1.png)


最终，Gralloc1都是在gralloc1_device_t中去实现的。当然，如果HAL没有实现对应地Gralloc1，而是Gralloc0。Android这边也是提供适配的。对应的代码在：
hardware/interfaces/graphics/allocator/2.0/default/gralloc1-adapter.cpp

下面，我们找一个具体平台的实现来看看，gralloc的HAL层是怎么实现的。



####  三、Qcom高通平台Gralloc HAL实现

我们这里拿到的代码是AOSP的，和vendor从Qcom那里拿到的估计有些区别。我们就看骁龙835吧～，msm8998的displayHAL相关的实现在hardware/qcom/display/msm8998。
##### gralloc1整体架构
高通gralloc HAL的实现在libgralloc1中

``` cpp
* hardware/qcom/display/msm8998/libgralloc1/gr_device_impl.cpp

static struct hw_module_methods_t gralloc_module_methods = {.open = gralloc_device_open};

struct gralloc_module_t HAL_MODULE_INFO_SYM = {
  .common = {
    .tag = HARDWARE_MODULE_TAG,
    .module_api_version = GRALLOC_MODULE_API_VERSION_1_0,
    .hal_api_version    = HARDWARE_HAL_API_VERSION,
    .id = GRALLOC_HARDWARE_MODULE_ID,
    .name = "Graphics Memory Module",
    .author = "Code Aurora Forum",
    .methods = &gralloc_module_methods,
    .dso = 0,
    .reserved = {0},
  },
};

int gralloc_device_open(const struct hw_module_t *module, const char *name, hw_device_t **device) {
  int status = -EINVAL;
  if (!strcmp(name, GRALLOC_HARDWARE_MODULE_ID)) {
    gralloc1::GrallocImpl * /*gralloc1_device_t*/ dev = gralloc1::GrallocImpl::GetInstance(module);
    *device = reinterpret_cast<hw_device_t *>(dev);
    if (dev) {
      status = 0;
    } else {
      ALOGE("Fatal error opening gralloc1 device");
    }
  }
  return status;
}
```

Qcom 的gralloc是1.0版本～采用C++编写，具体实现的类为 GrallocImpl。GrallocImpl继承gralloc1_device_t。

``` cpp
GrallocImpl::GrallocImpl(const hw_module_t *module) {
  common.tag = HARDWARE_DEVICE_TAG;
  common.version = GRALLOC_MODULE_API_VERSION_1_0;
  common.module = const_cast<hw_module_t *>(module);
  common.close = CloseDevice;
  getFunction = GetFunction;
  getCapabilities = GetCapabilities;

  initalized_ = Init();
}
```
gralloc1_device_t的getFunction初始化为GetFunction，getCapabilities初始化为GetCapabilities。
而在Init，申请了一个BufferManager。BufferManager是单例的用法。GrallocImpl也是单例的用法。

``` cpp
bool GrallocImpl::Init() {
  buf_mgr_ = BufferManager::GetInstance();
  return buf_mgr_ != nullptr;
}
```
Qcom的Gralloc1支持 Capabilities 有3种：

``` cpp
void GrallocImpl::GetCapabilities(struct gralloc1_device *device, uint32_t *out_count,
                                  int32_t  /*gralloc1_capability_t*/ *out_capabilities) {
  if (device != nullptr) {
    if (out_capabilities != nullptr && *out_count >= 3) {
      out_capabilities[0] = GRALLOC1_CAPABILITY_TEST_ALLOCATE;
      out_capabilities[1] = GRALLOC1_CAPABILITY_LAYERED_BUFFERS;
      out_capabilities[2] = GRALLOC1_CAPABILITY_RELEASE_IMPLY_DELETE;
    }
    *out_count = 3;
  }
  return;
}
```
从Android对Capabilities的定义来看，Qcom Gralloc1支持Android要求的所有能力。


``` cpp
* hardware/libhardware/include/hardware/gralloc1.h

typedef enum {
    GRALLOC1_CAPABILITY_INVALID = 0,

    /* If this capability is supported, then the outBuffers parameter to
     * allocate may be NULL, which instructs the device to report whether the
     * given allocation is possible or not. */
    GRALLOC1_CAPABILITY_TEST_ALLOCATE = 1,

    /* If this capability is supported, then the implementation supports
     * allocating buffers with more than one image layer. */
    GRALLOC1_CAPABILITY_LAYERED_BUFFERS = 2,

    /* If this capability is supported, then the implementation always closes
     * and deletes a buffer handle whenever the last reference is removed.
     *
     * Supporting this capability is strongly recommended.  It will become
     * mandatory in future releases. */
    GRALLOC1_CAPABILITY_RELEASE_IMPLY_DELETE = 3,

    GRALLOC1_LAST_CAPABILITY = 3,
} gralloc1_capability_t;
```
GetFunction函数，初始化函数指针，gralloc1_function_descriptor_t对应的指针实现如下：


``` cpp
gralloc1_function_pointer_t GrallocImpl::GetFunction(gralloc1_device_t *device, int32_t function) {
  if (!device) {
    return NULL;
  }

  switch (function) {
    case GRALLOC1_FUNCTION_DUMP:
      return reinterpret_cast<gralloc1_function_pointer_t>(Dump);
    case GRALLOC1_FUNCTION_CREATE_DESCRIPTOR:
      return reinterpret_cast<gralloc1_function_pointer_t>(CreateBufferDescriptor);
    case GRALLOC1_FUNCTION_DESTROY_DESCRIPTOR:
      return reinterpret_cast<gralloc1_function_pointer_t>(DestroyBufferDescriptor);
    case GRALLOC1_FUNCTION_SET_CONSUMER_USAGE:
      return reinterpret_cast<gralloc1_function_pointer_t>(SetConsumerUsage);
    case GRALLOC1_FUNCTION_SET_DIMENSIONS:
      return reinterpret_cast<gralloc1_function_pointer_t>(SetBufferDimensions);
    case GRALLOC1_FUNCTION_SET_FORMAT:
      return reinterpret_cast<gralloc1_function_pointer_t>(SetColorFormat);
    case GRALLOC1_FUNCTION_SET_LAYER_COUNT:
      return reinterpret_cast<gralloc1_function_pointer_t>(SetLayerCount);
    case GRALLOC1_FUNCTION_SET_PRODUCER_USAGE:
      return reinterpret_cast<gralloc1_function_pointer_t>(SetProducerUsage);
    case GRALLOC1_FUNCTION_GET_BACKING_STORE:
      return reinterpret_cast<gralloc1_function_pointer_t>(GetBackingStore);
    case GRALLOC1_FUNCTION_GET_CONSUMER_USAGE:
      return reinterpret_cast<gralloc1_function_pointer_t>(GetConsumerUsage);
    case GRALLOC1_FUNCTION_GET_DIMENSIONS:
      return reinterpret_cast<gralloc1_function_pointer_t>(GetBufferDimensions);
    case GRALLOC1_FUNCTION_GET_FORMAT:
      return reinterpret_cast<gralloc1_function_pointer_t>(GetColorFormat);
    case GRALLOC1_FUNCTION_GET_LAYER_COUNT:
      return reinterpret_cast<gralloc1_function_pointer_t>(GetLayerCount);
    case GRALLOC1_FUNCTION_GET_PRODUCER_USAGE:
      return reinterpret_cast<gralloc1_function_pointer_t>(GetProducerUsage);
    case GRALLOC1_FUNCTION_GET_STRIDE:
      return reinterpret_cast<gralloc1_function_pointer_t>(GetBufferStride);
    case GRALLOC1_FUNCTION_ALLOCATE:
      return reinterpret_cast<gralloc1_function_pointer_t>(AllocateBuffers);
    case GRALLOC1_FUNCTION_RETAIN:
      return reinterpret_cast<gralloc1_function_pointer_t>(RetainBuffer);
    case GRALLOC1_FUNCTION_RELEASE:
      return reinterpret_cast<gralloc1_function_pointer_t>(ReleaseBuffer);
    case GRALLOC1_FUNCTION_GET_NUM_FLEX_PLANES:
      return reinterpret_cast<gralloc1_function_pointer_t>(GetNumFlexPlanes);
    case GRALLOC1_FUNCTION_LOCK:
      return reinterpret_cast<gralloc1_function_pointer_t>(LockBuffer);
    case GRALLOC1_FUNCTION_LOCK_FLEX:
      return reinterpret_cast<gralloc1_function_pointer_t>(LockFlex);
    case GRALLOC1_FUNCTION_UNLOCK:
      return reinterpret_cast<gralloc1_function_pointer_t>(UnlockBuffer);
    case GRALLOC1_FUNCTION_PERFORM:
      return reinterpret_cast<gralloc1_function_pointer_t>(Gralloc1Perform);
    default:
      ALOGE("%s:Gralloc Error. Client Requested for unsupported function", __FUNCTION__);
      return NULL;
  }

  return NULL;
}
```
来看看Qcom Gralloc1的整体架构～
![Alt text | center](https://raw.githubusercontent.com/zhoujinjiani/PicGo/master/display.system/Android.PG5.gralloc1-class.png)
- GrallocImpl继承gralloc1_device_t，这个Gralloc1具体的实现！
- GrallocImpl采用一个BufferManager，管理Buffer，自己当领导！
- BufferManager，抽像了一个Allocator，负责具体的Buffer分配！
- Allocator说，我不具体干活，这个活外包给IonAlloc干，IonAlloc好好干，干不好，给就给别人来做了。
- IonAlloc采用ion Buffer，负责具体的Buffer处理

总的来说，设计清晰，扩展方便～

##### allocate相关流程
我们来看下allocate Buffer的流程，代码就不贴了，给个流程图吧～

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiani/PicGo/master/display.system/Android.PG5.gralloc1-allocate.png)

这个流程图中，包括了release的流程～
Android中的importBuffer函数，在default中变为registerBuffer，在HAL中变成retain，高通对应的实现为RetainBuffer，IonAlloc中又变为ImportBuffer。（...不止72变了...比孙悟空还厉害）

``` cpp
* hardware/qcom/display/msm8998/libgralloc1/gr_ion_alloc.cpp

int IonAlloc::ImportBuffer(int fd) {
  struct ion_fd_data fd_data;
  int err = 0;
  fd_data.fd = fd;
  if (ioctl(ion_dev_fd_, INT(ION_IOC_IMPORT), &fd_data)) {
    err = -errno;
    ALOGE("%s: ION_IOC_IMPORT failed with error - %s", __FUNCTION__, strerror(errno));
    return err;
  }
  return fd_data.handle;
}
```
import对应的ioctl为ION_IOC_IMPORT。

ion相关的定义，Android有标准的要求，可以参考：

``` cpp
* system/core/libion/original-kernel-headers/linux/ion.h

#ifndef _UAPI_LINUX_ION_H
#define _UAPI_LINUX_ION_H

#include <linux/ioctl.h>
#include <linux/types.h>

typedef int ion_user_handle_t;

enum ion_heap_type {
    ION_HEAP_TYPE_SYSTEM,
    ION_HEAP_TYPE_SYSTEM_CONTIG,
    ION_HEAP_TYPE_CARVEOUT,
    ION_HEAP_TYPE_CHUNK,
    ION_HEAP_TYPE_DMA,
    ION_HEAP_TYPE_CUSTOM, /* must be last so device specific heaps always
                 are at the end of this enum */
    ION_NUM_HEAPS = 16,
};

#define ION_HEAP_SYSTEM_MASK        (1 << ION_HEAP_TYPE_SYSTEM)
#define ION_HEAP_SYSTEM_CONTIG_MASK (1 << ION_HEAP_TYPE_SYSTEM_CONTIG)
#define ION_HEAP_CARVEOUT_MASK      (1 << ION_HEAP_TYPE_CARVEOUT)
#define ION_HEAP_TYPE_DMA_MASK      (1 << ION_HEAP_TYPE_DMA)

#define ION_NUM_HEAP_IDS        sizeof(unsigned int) * 8

#define ION_FLAG_CACHED 1       /* mappings of this buffer should be
                       cached, ion will do cache
                       maintenance when the buffer is
                       mapped for dma */
#define ION_FLAG_CACHED_NEEDS_SYNC 2    /* mappings of this buffer will created
                       at mmap time, if this is set
                       caches must be managed manually */

struct ion_allocation_data {
    size_t len;
    size_t align;
    unsigned int heap_id_mask;
    unsigned int flags;
    ion_user_handle_t handle;
};

struct ion_fd_data {
    ion_user_handle_t handle;
    int fd;
};

struct ion_handle_data {
    ion_user_handle_t handle;
};

struct ion_custom_data {
    unsigned int cmd;
    unsigned long arg;
};

#define ION_IOC_MAGIC       'I'

#define ION_IOC_ALLOC       _IOWR(ION_IOC_MAGIC, 0, \
                      struct ion_allocation_data)

#define ION_IOC_FREE        _IOWR(ION_IOC_MAGIC, 1, struct ion_handle_data)

#define ION_IOC_MAP     _IOWR(ION_IOC_MAGIC, 2, struct ion_fd_data)

#define ION_IOC_SHARE       _IOWR(ION_IOC_MAGIC, 4, struct ion_fd_data)

#define ION_IOC_IMPORT      _IOWR(ION_IOC_MAGIC, 5, struct ion_fd_data)

#define ION_IOC_SYNC        _IOWR(ION_IOC_MAGIC, 7, struct ion_fd_data)

#define ION_IOC_CUSTOM      _IOWR(ION_IOC_MAGIC, 6, struct ion_custom_data)

#endif /* _UAPI_LINUX_ION_H */

```
Qcom对应的定义在msm_ion.h中，msm_ion.h在这里就不看了。
##### Buffer的usage处理
usage分为两类，一个是Producer的，一个是Consumer的。在GetIonHeapInfo中对usage进行了处理。转换为ion对应的描述ion_heap_id，alloc_type以及ion_flags。这三个属性上面的头文件中有定义。usage的转换如下：

``` cpp
* hardware/qcom/display/msm8998/libgralloc1/gr_allocator.cpp

void Allocator::GetIonHeapInfo(gralloc1_producer_usage_t prod_usage,
                               gralloc1_consumer_usage_t cons_usage, unsigned int *ion_heap_id,
                               unsigned int *alloc_type, unsigned int *ion_flags) {
  unsigned int heap_id = 0;
  unsigned int type = 0;
  uint32_t flags = 0;
  if (prod_usage & GRALLOC1_PRODUCER_USAGE_PROTECTED) {
    if (cons_usage & GRALLOC1_CONSUMER_USAGE_PRIVATE_SECURE_DISPLAY) {
      heap_id = ION_HEAP(SD_HEAP_ID);
      /*
       * There is currently no flag in ION for Secure Display
       * VM. Please add it to the define once available.
       */
      flags |= UINT(ION_SD_FLAGS);
    } else if (prod_usage & GRALLOC1_PRODUCER_USAGE_CAMERA) {
      heap_id = ION_HEAP(SD_HEAP_ID);
      if (cons_usage & GRALLOC1_CONSUMER_USAGE_HWCOMPOSER) {
        flags |= UINT(ION_SC_PREVIEW_FLAGS);
      } else {
        flags |= UINT(ION_SC_FLAGS);
      }
    } else {
      heap_id = ION_HEAP(CP_HEAP_ID);
      flags |= UINT(ION_CP_FLAGS);
    }
  } else if (prod_usage & GRALLOC1_PRODUCER_USAGE_PRIVATE_MM_HEAP) {
    // MM Heap is exclusively a secure heap.
    // If it is used for non secure cases, fallback to IOMMU heap
    ALOGW("MM_HEAP cannot be used as an insecure heap. Using system heap instead!!");
    heap_id |= ION_HEAP(ION_SYSTEM_HEAP_ID);
  }

  if (prod_usage & GRALLOC1_PRODUCER_USAGE_PRIVATE_CAMERA_HEAP) {
    heap_id |= ION_HEAP(ION_CAMERA_HEAP_ID);
  }

  if (prod_usage & GRALLOC1_PRODUCER_USAGE_PRIVATE_ADSP_HEAP ||
      prod_usage & GRALLOC1_PRODUCER_USAGE_SENSOR_DIRECT_DATA) {
    heap_id |= ION_HEAP(ION_ADSP_HEAP_ID);
  }

  if (flags & UINT(ION_SECURE)) {
    type |= private_handle_t::PRIV_FLAGS_SECURE_BUFFER;
  }

  // if no ion heap flags are set, default to system heap
  if (!heap_id) {
    heap_id = ION_HEAP(ION_SYSTEM_HEAP_ID);
  }

  *alloc_type = type;
  *ion_flags = flags;
  *ion_heap_id = heap_id;

  return;
}
```
Qcom的Gralloc1就不多介绍了，需要注意的是Qcom对很多数据结构进行封装，增加了更多的信息，可以借助Qcom的文档进行理解。比如private_handle_t继承native_handle_t，对native_handle_t进行了扩展。定义在下面的头文件中。

``` cpp
hardware/qcom/display/msm8998/libgralloc1/gr_priv_handle.h
```

####  四、ION Buffer

Ion Buffer是一种内存分配器，是Android 4.0版本开始引入的，用以取代被诟病的PMEM，完美解决内存碎片管理。Ion管理着一个或多个内存池，其中有一些会在启动的时候预先分配，供一些特殊的设备使用，比如GPU，Display。
IonAlloc初始化时，将会打开对应的ion驱动设备。

``` cpp
* hardware/qcom/display/msm8998/libgralloc1/gr_ion_alloc.cpp

bool IonAlloc::Init() {
  if (ion_dev_fd_ == FD_INIT) {
    ion_dev_fd_ = open(kIonDevice, O_RDONLY);
  }

  if (ion_dev_fd_ < 0) {
    ALOGE("%s: Failed to open ion device - %s", __FUNCTION__, strerror(errno));
    ion_dev_fd_ = FD_INIT;
    return false;
  }

  return true;
}
```
##### heap的类型
ION的驱动在kernel的驱动中，前面说到的system/core中的ion.h其实是自动生成的。我们基于这个开源的分支来看android-msm-wahoo-4.4-oreo-mr1
kernel/drivers/staging/android/ion

Ion定义了6种不同的heap类似，实现不同的分配策略：

``` cpp
* drivers/staging/android/uapi/ion.h

enum ion_heap_type {
    ION_HEAP_TYPE_SYSTEM,
    ION_HEAP_TYPE_SYSTEM_CONTIG,
    ION_HEAP_TYPE_CARVEOUT,
    ION_HEAP_TYPE_CHUNK,
    ION_HEAP_TYPE_DMA,
    ION_HEAP_TYPE_CUSTOM, 
    ION_NUM_HEAPS = 16,
};
```
- ION_HEAP_TYPE_SYSTEM
通过vmallc分配，vmalloc只保证内存在虚拟地址空间是连续的。
- ION_HEAP_TYPE_SYSTEM_CONTIG
通过kmalloc分配，kmalloc保证物理地址也是连续的。
- ION_HEAP_TYPE_CARVEOUT
从保留的carveout 中分配一个heap，分配的内存是物理连续的。
- ION_HEAP_TYPE_CHUNK
分配一快大内存
- ION_HEAP_TYPE_DMA
通过DMA API分配内存，DMA的Buffer
- ION_HEAP_TYPE_CUSTOM
由用户自己定义，在enum中，必须是最后，这种heap比较特殊

Qcom msm8998中，实现的Ion type如下：

``` cpp
arch/arm/boot/dts/qcom/msm8998-ion.dtsi

&soc {
    qcom,ion {
        compatible = "qcom,msm-ion";
        #address-cells = <1>;
        #size-cells = <0>;

        system_heap: qcom,ion-heap@25 {
            reg = <25>;
            qcom,ion-heap-type = "SYSTEM";
        };

        qcom,ion-heap@22 { /* ADSP HEAP */
            reg = <22>;
            memory-region = <&adsp_mem>;
            qcom,ion-heap-type = "DMA";
        };

        qcom,ion-heap@27 { /* QSEECOM HEAP */
            reg = <27>;
            memory-region = <&qseecom_mem>;
            qcom,ion-heap-type = "DMA";
        };

        qcom,ion-heap@13 { /* SPSS HEAP */
            reg = <13>;
            memory-region = <&sp_mem>;
            qcom,ion-heap-type = "DMA";
        };

        qcom,ion-heap@10 { /* SECURE DISPLAY HEAP */
            reg = <10>;
            memory-region = <&secure_display_memory>;
            qcom,ion-heap-type = "HYP_CMA";
        };

        qcom,ion-heap@9 {
            reg = <9>;
            qcom,ion-heap-type = "SYSTEM_SECURE";
        };
    };
};
```
Qcom定义了更多的heap类型，做特殊之用。dts中的定义在驱动初始化时，将被读出来，构建ion_platform_heap，用ion_platform_heap进行描述。

``` cpp
* drivers/staging/android/ion/msm/msm_ion.c

static struct ion_platform_data *msm_ion_parse_dt(struct platform_device *pdev)
{
    struct ion_platform_data *pdata = 0;
    struct ion_platform_heap *heaps = NULL;
    struct device_node *node;
    struct platform_device *new_dev = NULL;
    const struct device_node *dt_node = pdev->dev.of_node;
    uint32_t val = 0;
    int ret = 0;
    uint32_t num_heaps = 0;
    int idx = 0;

    for_each_available_child_of_node(dt_node, node)
        num_heaps++;

    if (!num_heaps)
        return ERR_PTR(-EINVAL);

    pdata = kzalloc(sizeof(struct ion_platform_data), GFP_KERNEL);
    if (!pdata)
        return ERR_PTR(-ENOMEM);

    heaps = kzalloc(sizeof(struct ion_platform_heap)*num_heaps, GFP_KERNEL);
    if (!heaps) {
        kfree(pdata);
        return ERR_PTR(-ENOMEM);
    }

    pdata->heaps = heaps;
    pdata->nr = num_heaps;

    for_each_available_child_of_node(dt_node, node) {
        new_dev = of_platform_device_create(node, NULL, &pdev->dev);
        if (!new_dev) {
            pr_err("Failed to create device %s\n", node->name);
            goto free_heaps;
        }

        pdata->heaps[idx].priv = &new_dev->dev;
        /**
         * TODO: Replace this with of_get_address() when this patch
         * gets merged: http://
         * permalink.gmane.org/gmane.linux.drivers.devicetree/18614
        */
        ret = of_property_read_u32(node, "reg", &val);
        if (ret) {
            pr_err("%s: Unable to find reg key", __func__);
            goto free_heaps;
        }
        pdata->heaps[idx].id = val;

        ret = msm_ion_populate_heap(node, &pdata->heaps[idx]);
        if (ret)
            goto free_heaps;

        msm_ion_get_heap_dt_data(node, &pdata->heaps[idx]);

        ++idx;
    }
    return pdata;

free_heaps:
    free_pdata(pdata);
    return ERR_PTR(ret);
}
```
ion_platform_heap定定义如下：

``` cpp
* drivers/staging/android/ion/ion.h

/**
 * struct ion_platform_heap - defines a heap in the given platform
 * @type:   type of the heap from ion_heap_type enum
 * @id:     unique identifier for heap.  When allocating higher numbers
 *      will be allocated from first.  At allocation these are passed
 *      as a bit mask and therefore can not exceed ION_NUM_HEAP_IDS.
 * @name:   used for debug purposes
 * @base:   base address of heap in physical memory if applicable
 * @size:   size of the heap in bytes if applicable
 * @has_outer_cache:    set to 1 if outer cache is used, 0 otherwise.
 * @extra_data: Extra data specific to each heap type
 * @priv:   heap private data
 * @align:  required alignment in physical memory if applicable
 * @priv:   private info passed from the board file
 *
 * Provided by the board file.
 */
 
struct ion_platform_heap {
    enum ion_heap_type type;
    unsigned int id;
    const char *name;
    ion_phys_addr_t base;
    size_t size;
    unsigned int has_outer_cache;
    void *extra_data;
    ion_phys_addr_t align;
    void *priv;
};
```
type，就是dts中的ion-heap-type再加上ION_HEAP_TYPE_的前缀。
id，唯一的，是dts中reg的值
base，是物理地址的起始地址
name 是heap的名字，主要用来debug，以对应的id的形式定义在ion_heap_meta中。

``` cpp
* drivers/staging/android/ion/msm/msm_ion.c

static struct ion_heap_desc ion_heap_meta[] = {
    {
        .id = ION_SYSTEM_HEAP_ID,
        .name   = ION_SYSTEM_HEAP_NAME,
    },
    {
        .id = ION_SYSTEM_CONTIG_HEAP_ID,
        .name   = ION_KMALLOC_HEAP_NAME,
    },
    {
        .id = ION_SECURE_HEAP_ID,
        .name   = ION_SECURE_HEAP_NAME,
    },
    {
        .id = ION_CP_MM_HEAP_ID,
        .name   = ION_MM_HEAP_NAME,
        .permission_type = IPT_TYPE_MM_CARVEOUT,
    },
    {
        .id = ION_MM_FIRMWARE_HEAP_ID,
        .name   = ION_MM_FIRMWARE_HEAP_NAME,
    },
    {
        .id = ION_GOOGLE_HEAP_ID,
        .name   = ION_GOOGLE_HEAP_NAME,
    },
    {
        .id = ION_CP_MFC_HEAP_ID,
        .name   = ION_MFC_HEAP_NAME,
        .permission_type = IPT_TYPE_MFC_SHAREDMEM,
    },
    {
        .id = ION_SF_HEAP_ID,
        .name   = ION_SF_HEAP_NAME,
    },
    {
        .id = ION_QSECOM_HEAP_ID,
        .name   = ION_QSECOM_HEAP_NAME,
    },
    {
        .id = ION_SPSS_HEAP_ID,
        .name   = ION_SPSS_HEAP_NAME,
    },
    {
        .id = ION_AUDIO_HEAP_ID,
        .name   = ION_AUDIO_HEAP_NAME,
    },
    {
        .id = ION_PIL1_HEAP_ID,
        .name   = ION_PIL1_HEAP_NAME,
    },
    {
        .id = ION_PIL2_HEAP_ID,
        .name   = ION_PIL2_HEAP_NAME,
    },
    {
        .id = ION_CP_WB_HEAP_ID,
        .name   = ION_WB_HEAP_NAME,
    },
    {
        .id = ION_CAMERA_HEAP_ID,
        .name   = ION_CAMERA_HEAP_NAME,
    },
    {
        .id = ION_ADSP_HEAP_ID,
        .name   = ION_ADSP_HEAP_NAME,
    },
    {
        .id = ION_SECURE_DISPLAY_HEAP_ID,
        .name   = ION_SECURE_DISPLAY_HEAP_NAME,
    }
};
```
解析完dts，ion_platform_heap被放在ion_platform_data的heaps中。
创建ion设备，通过ion_device_create函数：

``` cpp
static int msm_ion_probe(struct platform_device *pdev)
{
    ... ...

    new_dev = ion_device_create(compat_msm_ion_ioctl);
```

创建ion_device成功后，会根据解析出来的ion_platform_heap，通过msm_ion_heap_create创建对应的ion_heap。
msm_ion_allocate 函数如下：

``` cpp
* drivers/staging/android/ion/msm/msm_ion.c

static struct ion_heap *msm_ion_heap_create(struct ion_platform_heap *heap_data)
{
    struct ion_heap *heap = NULL;

    switch ((int)heap_data->type) {
#ifdef CONFIG_CMA
    case ION_HEAP_TYPE_SECURE_DMA:
        heap = ion_secure_cma_heap_create(heap_data);
        break;
#endif
    case ION_HEAP_TYPE_SYSTEM_SECURE:
        heap = ion_system_secure_heap_create(heap_data);
        break;
    case ION_HEAP_TYPE_HYP_CMA:
        heap = ion_cma_secure_heap_create(heap_data);
        break;
    default:
        heap = ion_heap_create(heap_data);
    }

    if (IS_ERR_OR_NULL(heap)) {
        pr_err("%s: error creating heap %s type %d base %pa size %zu\n",
               __func__, heap_data->name, heap_data->type,
               &heap_data->base, heap_data->size);
        return ERR_PTR(-EINVAL);
    }

    heap->name = heap_data->name;
    heap->id = heap_data->id;
    heap->priv = heap_data->priv;
    return heap;
}
```


实际分配的heap有4种：
|类型|函数实现|
|------|-------|-------|-------|
|ION_HEAP_TYPE_SECURE_DMA|ion_secure_cma_heap_create|ion_cma_secure_heap|ion_secure_cma_ops|
|ION_HEAP_TYPE_SYSTEM_SECURE|ion_system_secure_heap_create|ion_system_secure_heap|system_secure_heap_ops|
|ION_HEAP_TYPE_HYP_CMA|ion_cma_secure_heap_create|ion_heap|ion_secure_cma_ops|
|ION_HEAP_TYPE_SYSTEM default|ion_heap_create|ion_system_heap|system_heap_ops|
创建的ion_heap通过ion_device_add_heap中的plist_add(&heap->node, &dev->heaps)，把每个创建的ion_heap->node链接到ion_device->heaps。

``` cpp
* drivers/staging/android/ion/ion.c

void ion_device_add_heap(struct ion_device *dev, struct ion_heap *heap)
{
    struct dentry *debug_file;

    if (!heap->ops->allocate || !heap->ops->free || !heap->ops->map_dma ||
        !heap->ops->unmap_dma)
        pr_err("%s: can not add heap with invalid ops struct.\n",
               __func__);

    spin_lock_init(&heap->free_lock);
    heap->free_list_size = 0;

    if (heap->flags & ION_HEAP_FLAG_DEFER_FREE)
        ion_heap_init_deferred_free(heap);

    if ((heap->flags & ION_HEAP_FLAG_DEFER_FREE) || heap->ops->shrink)
        ion_heap_init_shrinker(heap);

    heap->dev = dev;
    down_write(&dev->lock);
    /*
     * use negative heap->id to reverse the priority -- when traversing
     * the list later attempt higher id numbers first
     */
    plist_node_init(&heap->node, -heap->id);
    plist_add(&heap->node, &dev->heaps);
    debug_file = debugfs_create_file(heap->name, 0664,
                    dev->heaps_debug_root, heap,
                    &debug_heap_fops);

    ... ...

    up_write(&dev->lock);
}
```
- ION_HEAP_TYPE_SECURE_DMA
SECURE_DMA类型相关的实现如下：

``` cpp
* drivers/staging/android/ion/ion_cma_secure_heap.c

struct ion_cma_secure_heap {
    struct device *dev;
    /*
     * Protects against races between threads allocating memory/adding to
     * pool at the same time. (e.g. thread 1 adds to pool, thread 2
     * allocates thread 1's memory before thread 1 knows it needs to
     * allocate more.
     * Admittedly this is fairly coarse grained right now but the chance for
     * contention on this lock is unlikely right now. This can be changed if
     * this ever changes in the future
     */
    struct mutex alloc_lock;
    /*
     * protects the list of memory chunks in this pool
     */
    struct mutex chunk_lock;
    struct ion_heap heap;
    /*
     * Bitmap for allocation. This contains the aggregate of all chunks. */
    unsigned long *bitmap;
    /*
     * List of all allocated chunks
     *
     * This is where things get 'clever'. Individual allocations from
     * dma_alloc_coherent must be allocated and freed in one chunk.
     * We don't just want to limit the allocations to those confined
     * within a single chunk (if clients allocate n small chunks we would
     * never be able to use the combined size). The bitmap allocator is
     * used to find the contiguous region and the parts of the chunks are
     * marked off as used. The chunks won't be freed in the shrinker until
     * the usage is actually zero.
     */
    struct list_head chunks;
    int npages;
    ion_phys_addr_t base;
    struct work_struct work;
    unsigned long last_alloc;
    struct shrinker shrinker;
    atomic_t total_allocated;
    atomic_t total_pool_size;
    atomic_t total_leaked;
    unsigned long heap_size;
    unsigned long default_prefetch_size;
};
```

ion_cma_secure_heap继承ion_heap，又扩展了很多实现

SECURE_DMA对应的Ops如下：


``` cpp
static struct ion_heap_ops ion_secure_cma_ops = {
    .allocate = ion_secure_cma_allocate,
    .free = ion_secure_cma_free,
    .map_dma = ion_secure_cma_heap_map_dma,
    .unmap_dma = ion_secure_cma_heap_unmap_dma,
    .phys = ion_secure_cma_phys,
    .map_user = ion_secure_cma_mmap,
    .map_kernel = ion_secure_cma_map_kernel,
    .unmap_kernel = ion_secure_cma_unmap_kernel,
    .print_debug = ion_secure_cma_print_debug,
};
```

Ops就是ioctl下来的，相关cmd的实现。

- ION_HEAP_TYPE_SYSTEM_SECURE
SYSTEM_SECURE采用ion_system_secure_heap进行 描述，继承ion_heap。

``` cpp
* drivers/staging/android/ion/ion_system_secure_heap.c

struct ion_system_secure_heap {
    struct ion_heap *sys_heap;
    struct ion_heap heap;

    /* Protects prefetch_list */
    spinlock_t work_lock;
    bool destroy_heap;
    struct list_head prefetch_list;
    struct delayed_work prefetch_work;
};
```
对应的接口实现为system_secure_heap_ops

``` cpp
static struct ion_heap_ops system_secure_heap_ops = {
    .allocate = ion_system_secure_heap_allocate,
    .free = ion_system_secure_heap_free,
    .map_dma = ion_system_secure_heap_map_dma,
    .unmap_dma = ion_system_secure_heap_unmap_dma,
    .map_kernel = ion_system_secure_heap_map_kernel,
    .unmap_kernel = ion_system_secure_heap_unmap_kernel,
    .map_user = ion_system_secure_heap_map_user,
    .shrink = ion_system_secure_heap_shrink,
};
```
- ION_HEAP_TYPE_HYP_CMA
HYP_CMA主要用于Secure 显示。HYP_CMA没有做扩展，还是用ion_heap描述。但是采用Ops为ion_secure_cma_ops。

``` cpp
* drivers/staging/android/ion/ion_cma_heap.c

static struct ion_heap_ops ion_secure_cma_ops = {
    .allocate = ion_secure_cma_allocate,
    .free = ion_secure_cma_free,
    .map_dma = ion_cma_heap_map_dma,
    .unmap_dma = ion_cma_heap_unmap_dma,
    .phys = ion_cma_phys,
    .map_user = ion_cma_mmap,
    .map_kernel = ion_cma_map_kernel,
    .unmap_kernel = ion_cma_unmap_kernel,
    .print_debug = ion_cma_print_debug,
};
```
- 除上面3中类型外，其他的类型，通过ion_heap_create分配
具体的对应关系如下：

``` cpp
* drivers/staging/android/ion/ion_heap.c

struct ion_heap *ion_heap_create(struct ion_platform_heap *heap_data)
{
    struct ion_heap *heap = NULL;

    switch (heap_data->type) {
    case ION_HEAP_TYPE_SYSTEM_CONTIG:
        pr_err("%s: Heap type is disabled: %d\n", __func__,
               heap_data->type);
        return ERR_PTR(-EINVAL);
    case ION_HEAP_TYPE_SYSTEM:
        heap = ion_system_heap_create(heap_data);
        break;
    case ION_HEAP_TYPE_CARVEOUT:
        heap = ion_carveout_heap_create(heap_data);
        break;
    case ION_HEAP_TYPE_CHUNK:
        heap = ion_chunk_heap_create(heap_data);
        break;
    case ION_HEAP_TYPE_DMA:
        heap = ion_cma_heap_create(heap_data);
        break;
    default:
        pr_err("%s: Invalid heap type %d\n", __func__,
               heap_data->type);
        return ERR_PTR(-EINVAL);
    }

    if (IS_ERR_OR_NULL(heap)) {
        pr_err("%s: error creating heap %s type %d base %pa size %zu\n",
               __func__, heap_data->name, heap_data->type,
               &heap_data->base, heap_data->size);
        return ERR_PTR(-EINVAL);
    }

    heap->name = heap_data->name;
    heap->id = heap_data->id;
    heap->priv = heap_data->priv;
    return heap;
}
```
比如，ion_system_heap_create类型,采用ion_system_heap描述，继承ion_heap.

``` cpp
* drivers/staging/android/ion/ion_system_heap.c

struct ion_system_heap {
    struct ion_heap heap;
    struct ion_page_pool **uncached_pools;
    struct ion_page_pool **cached_pools;
    struct ion_page_pool **secure_pools[VMID_LAST];
    /* Prevents unnecessary page splitting */
    struct mutex split_page_mutex;
};
```
对应的Ops为system_heap_ops：


``` cpp
static struct ion_heap_ops system_heap_ops = {
    .allocate = ion_system_heap_allocate,
    .free = ion_system_heap_free,
    .map_dma = ion_system_heap_map_dma,
    .unmap_dma = ion_system_heap_unmap_dma,
    .map_kernel = ion_heap_map_kernel,
    .unmap_kernel = ion_heap_unmap_kernel,
    .map_user = ion_heap_map_user,
    .shrink = ion_system_heap_shrink,
};
```


##### Ion API
在用户空间，提供了系统调用ioctl，对应内存空间的API，内核空间的API对应具体类型的Heap的API。Heap的API用ion_heap_ops描述，前面我们已经说过了每种类型的API对应的ion_heap_ops。
除了Ion的标准API外，Qcom又定制了一些自己的ioctl，定制的ioctl实现为compat_msm_ion_ioctl，在下面的代码中

``` cpp
* drivers/staging/android/ion/compat_msm_ion.c
```
Ion标准的ioctl，在ion_ioctl中：

``` cpp
* drivers/staging/android/ion/ion.c

static const struct file_operations ion_fops = {
    .owner          = THIS_MODULE,
    .open           = ion_open,
    .release        = ion_release,
    .unlocked_ioctl = ion_ioctl,
    .compat_ioctl   = compat_ion_ioctl,
};
```

在我们的测试代码中，Producer设置是usage为GRALLOC_USAGE_SW_WRITE_OFTEN，Consumer中设置的usage为USAGE_HW_COMPOSER，Layer创建的时候设置的。

``` cpp
uint32_t Layer::getEffectiveUsage(uint32_t usage) const {
    // TODO: should we do something special if mSecure is set?
    if (mProtectedByApp) {
        // need a hardware-protected path to external video sink
        usage |= GraphicBuffer::USAGE_PROTECTED;
    }
    if (mPotentialCursor) {
        usage |= GraphicBuffer::USAGE_CURSOR;
    }
    usage |= GraphicBuffer::USAGE_HW_COMPOSER;
    return usage;
}
```
GRALLOC_USAGE_SW_WRITE_OFTEN将被转换为 GRALLOC1_PRODUCER_USAGE_CPU_WRITE_OFTEN；USAGE_HW_COMPOSER将被转换为GRALLOC1_CONSUMER_USAGE_HWCOMPOSER；对应的heap id为ION_SYSTEM_HEAP_ID。
下面，以ION_HEAP_TYPE_SYSTEM为例，我们通过一个表格来描述他们直接的关系，和API的用途


 | 用户空间API  | 内核空间API  | Heap API | 作用 |
 |:-------:|:-------:|:-------:|:-------:|
 | open |  ion_client_create  |  null  |  分配一个Ion的客户端，客户端负责和Ion设备进行通信 | 
 | close  | ion_client_destroy  | null  |释放一个Ion的客户端  |
 | ION_IOC_ALLOC |  ion_buffer_create |  ion_system_heap_allocate，map_dma |  申请一块Ion内存，返回Ion Handle | 
 | ION_IOC_FREE |  ion_free ion_free_nolock | ion_system_heap_free | 释放Ion handle | 
 | ION_IOC_SHARE & ION_IOC_MAP |  ion_share_dma_buf_fd |  null |  为制定的Buffer创建DMA映射，返回DMA Buffer的FD | 
 | ION_IOC_IMPORT | ion_import_dma_buf |  null | 通过DMA的FD，返回Ion Buffer的Handle | 
 | ION_IOC_CUSTOM   | mmap  | ion_mmap  | ion_heap_map_user  map内存到user空间 | 



Qcom定制的Ioctl，ION_IOC_CUSTOM还有

``` cpp
ION_IOC_CLEAN_CACHES
ION_IOC_INV_CACHES
ION_IOC_CLEAN_INV_CACHES
ION_IOC_PREFETCH
ION_IOC_DRAIN
```

ION是通过handle而非buffer地址来实现驱动间共享内存，用户空间共享内存也是利用同样原理，所以，map，import都是通过handle来完成。另外，Ion Buffer创建后，映射到 DMA Buffer，后续通过DMA Buffer来处理。
我们我们来看他们之间的关系类图～

![Alt text | center](https://raw.githubusercontent.com/zhoujinjiani/PicGo/master/display.system/Android.PG5.ion.png)



##### Ion Debug
Ion 在/sys/kernel/debug/ion/ 提供一个debugfs 接口。

``` cpp
... /sys/kernel/debug/ion # ls
clients egl heaps 
```

每个heap都有自己的debugfs目录，client内存使用状况显示在/sys/kernel/debug/ion/heaps/<<heap name>>

``` cpp
... /sys/kernel/debug/ion/heaps # ls 
carveout_fb carveout_fb_shrink carveout_overlay carveout_overlay_shrink system system_shrink
```

比如这个system的分配情况：

``` cpp
... /sys/kernel/debug/ion/heaps # cat system
          client              pid             size
----------------------------------------------------
----------------------------------------------------
orphaned allocations (info is from last known client):
          client      pid             user user_pid             size  mcnt  rcnt
 allocator@2.0-s      257  composer@2.1-se      258          7372800     0     1
 allocator@2.0-s      257  composer@2.1-se      258           139264     0     1
 allocator@2.0-s      257  composer@2.1-se      258           139264     0     1
 allocator@2.0-s      257  composer@2.1-se      258          3686400     0     1
 allocator@2.0-s      257  composer@2.1-se      258          3686400     0     1
 allocator@2.0-s      257  composer@2.1-se      258          3686400     0     1
 allocator@2.0-s      257  composer@2.1-se      258           139264     0     1
----------------------------------------------------
  total orphaned         18849792
          total          18849792
   deferred free                0
----------------------------------------------------
4 order 8 highmem pages                     in uncached pool = 4194304 total
2 order 8 lowmem pages              in uncached pool = 2097152 total
14 order 4 highmem pages                    in uncached pool = 917504 total
0 order 4 lowmem pages              in uncached pool = 0 total
0 order 0 highmem pages                     in uncached pool = 0 total
838 order 0 lowmem pages                in uncached pool = 3432448 total
0 order 8 highmem pages                     in cached pool = 0 total
0 order 8 lowmem pages                  in cached pool = 0 total
0 order 4 highmem pages                     in cached pool = 0 total
0 order 4 lowmem pages                  in cached pool = 0 total
0 order 0 highmem pages                     in cached pool = 0 total
0 order 0 lowmem pages                  in cached pool = 0 total
```

前面是ion Client的pid，这里的allocator@2.0-service。然后是使用者pid，这里是composer@2.1-service（大部分Buffer都是这个进分配的，用于显示）。
小结
本章主要讲述GraphicBuffer相关的流程，结合 Qcom的msm8998，讲述了Gralloc1.0的接口实现，介绍了Ion使用及Ion驱动实现。

####  四、参考文档（特别感谢）：
[（1）【GraphicBuffer和Gralloc分析（转载于： 夕月风）】](https://www.jianshu.com/p/dd0b38832346)

