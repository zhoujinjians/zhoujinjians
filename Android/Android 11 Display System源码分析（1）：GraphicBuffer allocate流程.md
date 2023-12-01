---
title: Android 11 Display System源码分析（1）：GraphicBuffer allocate流程（V1）
cover: https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/post.cover.pictures/bing-wallpaper-2018.04.41.jpg
categories: 
 - Display
tags:
 - Display
toc: true
abbrlink: 20220616
date: 2022-06-16 16:16:16
---
注：文章都是通过阅读各位前辈总结的资料、Android 11 Rockchip平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除（◕‿◕），转载请注明出处（©Rockchip ©Android @Linux 版权所有），谢谢。

（==**文章基于 Android 11.0**==）

[【zhoujinjian.com博客原图链接】](https://github.com/zhoujinjianmm) 

[【开发板】](https://wiki.radxa.com/Rockpi4)

[【开发板 源码链接】](https://wiki.radxa.com/Rockpi4/rockpi-android11)



正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢，由于不喜欢图片水印，去除了水印，敬请谅解！！！

[zhoujinjian.com](http://zhoujinjian.com) 



--------------------------------------------------------------------------------

==源码（部分）==：

**/lagvm/lagvm/LINUX/android/frameworks/native/opengl/tests/gralloc/gralloc.cpp**

**/lagvm/lagvm/LINUX/android/frameworks/native/libs/ui/**

**Y:\home\zhoujinjian\android11_rockpi4\hardware\rockchip\libgralloc\midgard**

==源码（部分）==：

--------------------------------------------------------------------------------

## 一、GraphicBuffer测试程序
----------------------


## 1.1 、"test-opengl-gralloc"源码

```c++
//android/frameworks/native/opengl/tests/gralloc/gralloc.cpp
#include <stdlib.h>
#include <stdio.h>
#include <utils/StopWatch.h>
#include <utils/Log.h>
#include <ui/GraphicBuffer.h>
#include <ui/GraphicBufferMapper.h>
using namespace android;
......
int main(int /*argc*/, char** /*argv*/)
{
    size_t size = 128*256*4;
    void* temp = malloc(size);
    ......
    memset(temp, 0, size);
    ......

    sp<GraphicBuffer> buffer = new GraphicBuffer(128, 256, HAL_PIXEL_FORMAT_RGBA_8888,
            GRALLOC_USAGE_SW_READ_OFTEN |
            GRALLOC_USAGE_SW_WRITE_OFTEN);

    status_t err = buffer->initCheck();
    ......
    void* vaddr;
    buffer->lock(
            GRALLOC_USAGE_SW_READ_OFTEN | GRALLOC_USAGE_SW_WRITE_OFTEN,
            &vaddr);
    ......
    {
        StopWatch watch("memcpy from gralloc");
        for (int i=0 ; i<10 ; i++)
            memcpy(temp, vaddr, size);
    }

    {
        StopWatch watch("memcpy into gralloc");
        for (int i=0 ; i<10 ; i++)
            memcpy(vaddr, temp, size);
    }
    ......
    buffer->unlock();

    return 0;
}
```

可以看到程序得简单逻辑，new 一个GraphicBuffer，然后执行lock，unlock操作，然后结束运行。

首先看看GraphicBuffer allocate的堆栈信息，然后结合代码查看流程。

## 1.2 、GraphicBuffer allocate流程（Binder Bp端）

```c++
Stack Trace:
00000000000954d0  android::hardware::BpHwBinder::transact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+104                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          system/libhwbinder/BpHwBinder.cpp:111
  000000000000b5b8  android::hardware::graphics::allocator::V4_0::BpHwAllocator::_hidl_allocate(android::hardware::IInterface*, android::hardware::details::HidlInstrumentor*, android::hardware::hidl_vec<unsigned char> const&, unsigned int, std::__1::function<void (android::hardware::graphics::mapper::V4_0::Error, unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> const&)>)+400                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        out/soong/.intermediates/hardware/interfaces/graphics/allocator/4.0/android.hardware.graphics.allocator@4.0_genc++/gen/android/hardware/graphics/allocator/4.0/AllocatorAll.cpp:178
  000000000000b8b8  android::hardware::graphics::allocator::V4_0::BpHwAllocator::allocate(android::hardware::hidl_vec<unsigned char> const&, unsigned int, std::__1::function<void (android::hardware::graphics::mapper::V4_0::Error, unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> const&)>)+168                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             out/soong/.intermediates/hardware/interfaces/graphics/allocator/4.0/android.hardware.graphics.allocator@4.0_genc++/gen/android/hardware/graphics/allocator/4.0/AllocatorAll.cpp:253
  0000000000033958  android::Gralloc4Allocator::allocate(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >, unsigned int, unsigned int, int, unsigned int, unsigned long, unsigned int, unsigned int*, native_handle const**, bool) const+312                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         frameworks/native/libs/ui/Gralloc4.cpp:1082
  00000000000380a8  android::GraphicBufferAllocator::allocateHelper(unsigned int, unsigned int, int, unsigned int, unsigned long, native_handle const**, unsigned int*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >, bool)+312                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  frameworks/native/libs/ui/GraphicBufferAllocator.cpp:140
  00000000000384a0  android::GraphicBufferAllocator::allocate(unsigned int, unsigned int, int, unsigned int, unsigned long, native_handle const**, unsigned int*, unsigned long, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >)+136                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               frameworks/native/libs/ui/GraphicBufferAllocator.cpp:199
  0000000000036374  android::GraphicBuffer::initWithSize(unsigned int, unsigned int, int, unsigned int, unsigned long, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >)+212                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         frameworks/native/libs/ui/GraphicBuffer.cpp:196
  v-------------->  GraphicBuffer                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         frameworks/native/libs/ui/GraphicBuffer.cpp:87
  0000000000036180  android::GraphicBuffer::GraphicBuffer(unsigned int, unsigned int, int, unsigned int, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >)+128                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       frameworks/native/libs/ui/GraphicBuffer.cpp:80
  00000000000010ec  main+164                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              /data/test-opengl-gralloc
  0000000000049a34  __libc_init+108                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       bionic/libc/bionic/libc_init_dynamic.cpp:151

```

## 1.3 、GraphicBuffer allocate流程（Binder Bn端）

```c++
Stack Trace:
  RELADDR           FUNCTION                                                                                                                                                                                                                                                                                                                    FILE:LINE
  000000000000e410  mali_gralloc_ion_allocate(unsigned long const*, unsigned int, native_handle const**, bool*)+136                                                                                                                                                                                                                             hardware/rockchip/libgralloc/midgard/src/allocator/mali_gralloc_ion.cpp:750
  000000000000b9e0  mali_gralloc_buffer_allocate(unsigned long const*, unsigned int, native_handle const**, bool*)+112                                                                                                                                                                                                                          hardware/rockchip/libgralloc/midgard/src/core/mali_gralloc_bufferallocation.cpp:1075
  0000000000008ff0  arm::allocator::common::allocate(buffer_descriptor_t const&, unsigned int, std::__1::function<void (android::hardware::graphics::mapper::V4_0::Error, unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> const&)>, std::__1::function<int (buffer_descriptor_t const*, native_handle const**)>)+184  hardware/rockchip/libgralloc/midgard/src/hidl_common/Allocator.cpp:79
  000000000000848c  arm::allocator::GrallocAllocator::allocate(android::hardware::hidl_vec<unsigned char> const&, unsigned int, std::__1::function<void (android::hardware::graphics::mapper::V4_0::Error, unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> const&)>)+668                                              hardware/rockchip/libgralloc/midgard/src/4.x/GrallocAllocator.cpp:51
  000000000000cb30  android::hardware::graphics::allocator::V4_0::BnHwAllocator::_hidl_allocate(android::hidl::base::V1_0::BnHwBase*, android::hardware::Parcel const&, android::hardware::Parcel*, std::__1::function<void (android::hardware::Parcel&)>)+440                                                                                  out/soong/.intermediates/hardware/interfaces/graphics/allocator/4.0/android.hardware.graphics.allocator@4.0_genc++/gen/android/hardware/graphics/allocator/4.0/AllocatorAll.cpp:395
  000000000000ce0c  android::hardware::graphics::allocator::V4_0::BnHwAllocator::onTransact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+196                                                                                                out/soong/.intermediates/hardware/interfaces/graphics/allocator/4.0/android.hardware.graphics.allocator@4.0_genc++/gen/android/hardware/graphics/allocator/4.0/AllocatorAll.cpp:494
  00000000000947c0  android::hardware::BHwBinder::transact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+72                                                                                                                                  system/libhwbinder/Binder.cpp:116
  v-------------->  android::hardware::IPCThreadState::executeCommand(int)                                                                                                                                                                                                                                                                      system/libhwbinder/IPCThreadState.cpp:1206
  0000000000098874  android::hardware::IPCThreadState::getAndExecuteCommand()+1076                                                                                                                                                                                                                                                              system/libhwbinder/IPCThreadState.cpp:459
  0000000000099ad0  android::hardware::IPCThreadState::joinThreadPool(bool)+96                                                                                                                                                                                                                                                                  system/libhwbinder/IPCThreadState.cpp:559
  v-------------->  int android::hardware::defaultPassthroughServiceImplementation<android::hardware::graphics::allocator::V4_0::IAllocator, android::hardware::graphics::allocator::V4_0::IAllocator>(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, unsigned long)                             system/libhidl/transport/include/hidl/LegacySupport.h:75
  v-------------->  int android::hardware::defaultPassthroughServiceImplementation<android::hardware::graphics::allocator::V4_0::IAllocator, android::hardware::graphics::allocator::V4_0::IAllocator>(unsigned long)                                                                                                                           system/libhidl/transport/include/hidl/LegacySupport.h:81
  00000000000010dc  main+148                                                                                                                                                                                                                                                                                                                    hardware/rockchip/libgralloc/bifrost/service/4.x/service.cpp:27
  0000000000049a34  __libc_init+108                                                                                                                                                                                                                                                                                                             bionic/libc/bionic/libc_init_dynamic.cpp:151

```



二、GraphicBuffer allocate流程源代码分析
----------------



## 1.1 、GraphicBuffer allocate流程（Binder Bp客户端）

Binder Bp客户端就是我们的测试Demo。

> Y:\home\zhoujinjian\android11_rockpi4\frameworks\native\libs\ui\GraphicBuffer.cpp

![image-20220810195307065](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/Android_Display_System/Android11_Display01/image-20220810195307065.png)

![image-20220810195416705](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/Android_Display_System/Android11_Display01/image-20220810195416705.png)

GraphicBuffer::initWithSize首先获取GraphicBufferAllocator实例。初始化GrallocAllocator，系统优先寻找高版本，我们这里是Gralloc4Allocator。

```c++
Y:\home\zhoujinjian\android11_rockpi4\frameworks\native\libs\ui\include\ui\GraphicBufferAllocator.h
static inline GraphicBufferAllocator& get() { return getInstance(); }

Y:\home\zhoujinjian\android11_rockpi4\frameworks\native\libs\ui\GraphicBufferAllocator.cpp
GraphicBufferAllocator::GraphicBufferAllocator() : mMapper(GraphicBufferMapper::getInstance()) {
    mAllocator = std::make_unique<const Gralloc4Allocator>(
            reinterpret_cast<const Gralloc4Mapper&>(mMapper.getGrallocMapper()));
    ......
}

Y:\home\zhoujinjian\android11_rockpi4\frameworks\native\libs\ui\GraphicBufferMapper.cpp
GraphicBufferMapper::GraphicBufferMapper() {
    mMapper = std::make_unique<const Gralloc4Mapper>();
    ......
}

Y:\home\zhoujinjian\android11_rockpi4\frameworks\native\libs\ui\Gralloc4.cpp
void Gralloc4Mapper::preload() {
    android::hardware::preloadPassthroughService<IMapper>();
}

Gralloc4Mapper::Gralloc4Mapper() {
    mMapper = IMapper::getService();
    ......
}
```

接着看看Gralloc4Allocator构造函数，通过hidl获取IAllocator Bp端。

> Y:\home\zhoujinjian\android11_rockpi4\frameworks\native\libs\ui\Gralloc4.cpp

![image-20220810202950919](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/Android_Display_System/Android11_Display01/image-20220810202950919.png)

![image-20220810202125640](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/Android_Display_System/Android11_Display01/image-20220810202125640.png)

![image-20220810202216679](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/Android_Display_System/Android11_Display01/image-20220810202216679.png)

![image-20220810203207268](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/Android_Display_System/Android11_Display01/image-20220810203207268.png)

继续看看堆栈：

```c++
  00000000000954d0  android::hardware::BpHwBinder::transact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+104                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          system/libhwbinder/BpHwBinder.cpp:111
  000000000000b5b8  android::hardware::graphics::allocator::V4_0::BpHwAllocator::_hidl_allocate(android::hardware::IInterface*, android::hardware::details::HidlInstrumentor*, android::hardware::hidl_vec<unsigned char> const&, unsigned int, std::__1::function<void (android::hardware::graphics::mapper::V4_0::Error, unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> const&)>)+400                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        out/soong/.intermediates/hardware/interfaces/graphics/allocator/4.0/android.hardware.graphics.allocator@4.0_genc++/gen/android/hardware/graphics/allocator/4.0/AllocatorAll.cpp:178
  000000000000b8b8  android::hardware::graphics::allocator::V4_0::BpHwAllocator::allocate(android::hardware::hidl_vec<unsigned char> const&, unsigned int, std::__1::function<void (android::hardware::graphics::mapper::V4_0::Error, unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> const&)>)+168                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             out/soong/.intermediates/hardware/interfaces/graphics/allocator/4.0/android.hardware.graphics.allocator@4.0_genc++/gen/android/hardware/graphics/allocator/4.0/AllocatorAll.cpp:253

```

```c++
Y:\home\zhoujinjian\android11_rockpi4\out\soong\.intermediates\hardware\interfaces\graphics\allocator\4.0\android.hardware.graphics.allocator@4.0_genc++\gen\android\hardware\graphics\allocator\4.0\AllocatorAll.cpp
```

![image-20220810204030541](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/Android_Display_System/Android11_Display01/image-20220810204030541.png)

![image-20220810204129098](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/Android_Display_System/Android11_Display01/image-20220810204129098.png)

这里就通过binder通信将allocate 请求发送到服务端了。

## 1.2 、GraphicBuffer allocate流程（Binder Bn服务端）

Binder Bn服务端就是android.hardware.graphics.allocator@4.0-service（Hal Service）。

```c++
system          233      1 10834060 10388 binder_thread_read  0 S android.hardware.graphics.allocator@4.0-service
```

```c++
  000000000000cb30  android::hardware::graphics::allocator::V4_0::BnHwAllocator::_hidl_allocate(android::hidl::base::V1_0::BnHwBase*, android::hardware::Parcel const&, android::hardware::Parcel*, std::__1::function<void (android::hardware::Parcel&)>)+440                                                                                  out/soong/.intermediates/hardware/interfaces/graphics/allocator/4.0/android.hardware.graphics.allocator@4.0_genc++/gen/android/hardware/graphics/allocator/4.0/AllocatorAll.cpp:395
  000000000000ce0c  android::hardware::graphics::allocator::V4_0::BnHwAllocator::onTransact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+196                                                                                                out/soong/.intermediates/hardware/interfaces/graphics/allocator/4.0/android.hardware.graphics.allocator@4.0_genc++/gen/android/hardware/graphics/allocator/4.0/AllocatorAll.cpp:494

```

> Y:\home\zhoujinjian\android11_rockpi4\out\soong\.intermediates\hardware\interfaces\graphics\allocator\4.0\android.hardware.graphics.allocator@4.0_genc++\gen\android\hardware\graphics\allocator\4.0\AllocatorAll.cpp

![image-20220810210407913](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/Android_Display_System/Android11_Display01/image-20220810210407913.png)

![image-20220810210518143](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/Android_Display_System/Android11_Display01/image-20220810210518143.png)



三、 allocate具体实现
----------------------

> hardware/rockchip/libgralloc/midgard/src/4.x/GrallocAllocator.cpp:51


![image-20220811133017826](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/Android_Display_System/Android11_Display01/image-20220811133017826.png)

> hardware/rockchip/libgralloc/midgard/src/hidl_common/Allocator.cpp:79

![image-20220811133411980](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/Android_Display_System/Android11_Display01/image-20220811133411980.png)

具体实现是通过ION分配：

> Y:\home\zhoujinjian\android11_rockpi4\hardware\rockchip\libgralloc\midgard\src\core\mali_gralloc_bufferallocation.cpp

![image-20220811145600644](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/Android_Display_System/Android11_Display01/image-20220811145600644.png)

> Y:\home\zhoujinjian\android11_rockpi4\hardware\rockchip\libgralloc\midgard\src\allocator\mali_gralloc_ion.cpp

![image-20220811145721783](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/Android_Display_System/Android11_Display01/image-20220811145721783.png)

首先来看看allocate这段代码逻辑：

```c++
Y:\home\zhoujinjian\android11_rockpi4\hardware\rockchip\libgralloc\midgard\src\hidl_common\Allocator.cpp

void allocate(const buffer_descriptor_t &bufferDescriptor, uint32_t count, IAllocator::allocate_cb hidl_cb,
              std::function<int(const buffer_descriptor_t *, buffer_handle_t *)> fb_allocator)
{
#if DISABLE_FRAMEBUFFER_HAL
	GRALLOC_UNUSED(fb_allocator);
#endif

	Error error = Error::NONE;
	int stride = 0;
	std::vector<hidl_handle> grallocBuffers;
	gralloc_buffer_descriptor_t grallocBufferDescriptor[1];

	grallocBufferDescriptor[0] = (gralloc_buffer_descriptor_t)(&bufferDescriptor);
	grallocBuffers.reserve(count);

	for (uint32_t i = 0; i < count; i++)
	{
		buffer_handle_t tmpBuffer = nullptr;

		int allocResult;
#if (DISABLE_FRAMEBUFFER_HAL != 1)
		if (((bufferDescriptor.producer_usage & GRALLOC_USAGE_HW_FB) ||
		     (bufferDescriptor.consumer_usage & GRALLOC_USAGE_HW_FB)) &&
		    fb_allocator)
		{
			allocResult = fb_allocator(&bufferDescriptor, &tmpBuffer);
		}
		else
#endif
		{
			allocResult = mali_gralloc_buffer_allocate(grallocBufferDescriptor, 1, &tmpBuffer, nullptr);
			if (allocResult != 0)
			{
				MALI_GRALLOC_LOGE("%s, buffer allocation failed with %d", __func__, allocResult);
				error = Error::NO_RESOURCES;
				break;
			}
			auto hnd = const_cast<private_handle_t *>(reinterpret_cast<const private_handle_t *>(tmpBuffer));
			hnd->imapper_version = HIDL_MAPPER_VERSION_SCALED;

#if GRALLOC_USE_SHARED_METADATA
			hnd->reserved_region_size = bufferDescriptor.reserved_size;
			hnd->attr_size = mapper::common::shared_metadata_size() + hnd->reserved_region_size;
#else
			hnd->attr_size = sizeof(attr_region);
#endif
			std::tie(hnd->share_attr_fd, hnd->attr_base) =
			    gralloc_shared_memory_allocate("gralloc_shared_memory", hnd->attr_size);
			if (hnd->share_attr_fd < 0 || hnd->attr_base == MAP_FAILED)
			{
				MALI_GRALLOC_LOGE("%s, shared memory allocation failed with errno %d", __func__, errno);
				mali_gralloc_buffer_free(tmpBuffer);
				error = Error::UNSUPPORTED;
				break;
			}

#if GRALLOC_USE_SHARED_METADATA
			mapper::common::shared_metadata_init(hnd->attr_base, bufferDescriptor.name);
#else
			new(hnd->attr_base) attr_region;
#endif
			const uint32_t base_format = bufferDescriptor.alloc_format & MALI_GRALLOC_INTFMT_FMT_MASK;
			const uint64_t usage = bufferDescriptor.consumer_usage | bufferDescriptor.producer_usage;
			android_dataspace_t dataspace;
			get_format_dataspace(base_format, usage, hnd->width, hnd->height, &dataspace,
			                     &hnd->yuv_info);

#if GRALLOC_USE_SHARED_METADATA
			mapper::common::set_dataspace(hnd, static_cast<mapper::common::Dataspace>(dataspace));
#else
			int temp_dataspace = static_cast<int>(dataspace);
			gralloc_buffer_attr_write(hnd, GRALLOC_ARM_BUFFER_ATTR_DATASPACE, &temp_dataspace);
#endif
			/*
			 * We need to set attr_base to MAP_FAILED before the HIDL callback
			 * to avoid sending an invalid pointer to the client process.
			 *
			 * hnd->attr_base = mmap(...);
			 * hidl_callback(hnd); // client receives hnd->attr_base = <dangling pointer>
			 */
			munmap(hnd->attr_base, hnd->attr_size);
			hnd->attr_base = MAP_FAILED;

			buffer_descriptor_t * const bufDescriptor = (buffer_descriptor_t *)(grallocBufferDescriptor[0]);
			D("got new private_handle_t instance @%p for buffer '%s'. share_fd : %d, share_attr_fd : %d, "
				"flags : 0x%x, width : %d, height : %d, "
				"req_format : 0x%x, producer_usage : 0x%" PRIx64 ", consumer_usage : 0x%" PRIx64 ", "
				"internal_format : 0x%" PRIx64 ", stride : %d, byte_stride : %d, "
				"internalWidth : %d, internalHeight : %d, "
				"alloc_format : 0x%" PRIx64 ", size : %d, layer_count : %u, backing_store_size : %d, "
				"allocating_pid : %d, ref_count : %d, yuv_info : %d",
				hnd, (bufDescriptor->name).c_str() == nullptr ? "unset" : (bufDescriptor->name).c_str(),
			  hnd->share_fd, hnd->share_attr_fd,
			  hnd->flags, hnd->width, hnd->height,
			  hnd->req_format, hnd->producer_usage, hnd->consumer_usage,
			  hnd->internal_format, hnd->stride, hnd->byte_stride,
			  hnd->internalWidth, hnd->internalHeight,
			  hnd->alloc_format, hnd->size, hnd->layer_count, hnd->backing_store_size,
			  hnd->allocating_pid, hnd->ref_count, hnd->yuv_info);
			ALOGD("plane_info[0]: offset : %u, byte_stride : %u, alloc_width : %u, alloc_height : %u",
					(hnd->plane_info)[0].offset,
					(hnd->plane_info)[0].byte_stride,
					(hnd->plane_info)[0].alloc_width,
					(hnd->plane_info)[0].alloc_height);
			ALOGD("plane_info[1]: offset : %u, byte_stride : %u, alloc_width : %u, alloc_height : %u",
					(hnd->plane_info)[1].offset,
					(hnd->plane_info)[1].byte_stride,
					(hnd->plane_info)[1].alloc_width,
					(hnd->plane_info)[1].alloc_height);
		}

		int tmpStride = 0;
		if (GRALLOC_USE_LEGACY_CALCS)
		{
			const private_handle_t *hnd = static_cast<const private_handle_t *>(tmpBuffer);
			tmpStride = hnd->stride;
		}
		else
		{
			tmpStride = bufferDescriptor.pixel_stride;
		}

		if (stride == 0)
		{
			stride = tmpStride;
		}
		else if (stride != tmpStride)
		{
			/* Stride must be the same for all allocations */
			mali_gralloc_buffer_free(tmpBuffer);
			stride = 0;
			error = Error::UNSUPPORTED;
			break;
		}

		grallocBuffers.emplace_back(hidl_handle(tmpBuffer));
	}

	/* Populate the array of buffers for application consumption */
	hidl_vec<hidl_handle> hidlBuffers;
	if (error == Error::NONE)
	{
		hidlBuffers.setToExternal(grallocBuffers.data(), grallocBuffers.size());
	}
	hidl_cb(error, stride, hidlBuffers);

	/* The application should import the Gralloc buffers using IMapper for
	 * further usage. Free the allocated buffers in IAllocator context
	 */
	for (const auto &buffer : grallocBuffers)
	{
		mali_gralloc_buffer_free(buffer.getNativeHandle());
		native_handle_delete(const_cast<native_handle_t *>(buffer.getNativeHandle()));
	}
}

```

> 1，根据client需求buffer数量，循环allocate ion buffer
>
> 2，buffer_handle_t tmpBuffer转换为hidl_handle放到grallocBuffers数组里面
>
> 3，通过hidl_cb回调给client



![image-20220811150822101](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/Android_Display_System/Android11_Display01/image-20220811150822101.png)

可以看到通过_hidl_reply->writexxx()回调给client进程。

```c++
  Y:\home\zhoujinjian\android11_rockpi4\out\soong\.intermediates\hardware\interfaces\graphics\allocator\4.0\android.hardware.graphics.allocator@4.0_genc++\gen\android\hardware\graphics\allocator\4.0\AllocatorAll.cpp
      
::android::hardware::Return<void> _hidl_ret = static_cast<IAllocator*>(_hidl_this->getImpl().get())->allocate(*descriptor, count, [&](const auto &_hidl_out_error, const auto &_hidl_out_stride, const auto &_hidl_out_buffers) {
        if (_hidl_callbackCalled) {
            LOG_ALWAYS_FATAL("allocate: _hidl_cb called a second time, but must be called once.");
        }
        _hidl_callbackCalled = true;

        ::android::hardware::writeToParcel(::android::hardware::Status::ok(), _hidl_reply);

        _hidl_err = _hidl_reply->writeInt32((int32_t)_hidl_out_error);
        if (_hidl_err != ::android::OK) { goto _hidl_error; }

        _hidl_err = _hidl_reply->writeUint32(_hidl_out_stride);
        if (_hidl_err != ::android::OK) { goto _hidl_error; }

        size_t _hidl__hidl_out_buffers_parent;

        _hidl_err = _hidl_reply->writeBuffer(&_hidl_out_buffers, sizeof(_hidl_out_buffers), &_hidl__hidl_out_buffers_parent);
        if (_hidl_err != ::android::OK) { goto _hidl_error; }

        size_t _hidl__hidl_out_buffers_child;

        _hidl_err = ::android::hardware::writeEmbeddedToParcel(
                _hidl_out_buffers,
                _hidl_reply,
                _hidl__hidl_out_buffers_parent,
                0 /* parentOffset */, &_hidl__hidl_out_buffers_child);

        if (_hidl_err != ::android::OK) { goto _hidl_error; }

        for (size_t _hidl_index_0 = 0; _hidl_index_0 < _hidl_out_buffers.size(); ++_hidl_index_0) {
            _hidl_err = ::android::hardware::writeEmbeddedToParcel(
                    _hidl_out_buffers[_hidl_index_0],
                    _hidl_reply,
                    _hidl__hidl_out_buffers_child,
                    _hidl_index_0 * sizeof(::android::hardware::hidl_handle));

            if (_hidl_err != ::android::OK) { goto _hidl_error; }

        }
    _hidl_error:
        atrace_end(ATRACE_TAG_HAL);
        #ifdef __ANDROID_DEBUGGABLE__
        if (UNLIKELY(mEnableInstrumentation)) {
            std::vector<void *> _hidl_args;
            _hidl_args.push_back((void *)&_hidl_out_error);
            _hidl_args.push_back((void *)&_hidl_out_stride);
            _hidl_args.push_back((void *)&_hidl_out_buffers);
            for (const auto &callback: mInstrumentationCallbacks) {
                callback(InstrumentationEvent::SERVER_API_EXIT, "android.hardware.graphics.allocator", "4.0", "IAllocator", "allocate", &_hidl_args);
            }
        }
        #endif // __ANDROID_DEBUGGABLE__

        if (_hidl_err != ::android::OK) { return; }
        _hidl_cb(*_hidl_reply);
    });

    _hidl_ret.assertOk();
    if (!_hidl_callbackCalled) {
        LOG_ALWAYS_FATAL("allocate: _hidl_cb not called, but must be called once.");
    }
```

四 、Log看看分配信息：
-----------------

```c++
Log:
05-23 09:03:25.892   233   233 D gralloc4: [File] : hardware/rockchip/libgralloc/midgard/src/hidl_common/Allocator.cpp; [Line] : 147; [Func] : allocate;
05-23 09:03:25.892   233   233 D gralloc4: got new private_handle_t instance @0xb400006f5fc99eb0 for buffer '<Unknown>'. share_fd : 8, share_attr_fd : 9, flags : 0x4, width : 128, height : 256, req_format : 0x1, producer_usage : 0x33, consumer_usage : 0x33, internal_format : 0x1, stride : 128, byte_stride : 512, internalWidth : 128, internalHeight : 256, alloc_format : 0x1, size : 131072, layer_count : 1, backing_store_size : 131072, allocating_pid : 233, ref_count : 1, yuv_info : 0
05-23 09:03:25.892   233   233 D gralloc4: plane_info[0]: offset : 0, byte_stride : 512, alloc_width : 128, alloc_height : 256
05-23 09:03:25.892   233   233 D gralloc4: plane_info[1]: offset : 0, byte_stride : 0, alloc_width : 0, alloc_height : 0
```

## 五、importBuffer



```c++
Stack Trace:
  RELADDR           FUNCTION                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              FILE:LINE
  00000000000201c8  mali_gralloc_ion_map(private_handle_t*)+96                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            hardware/rockchip/libgralloc/midgard/src/allocator/mali_gralloc_ion.cpp:987
  000000000001dd2c  mali_gralloc_reference_retain(native_handle const*)+260                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               hardware/rockchip/libgralloc/midgard/src/core/mali_gralloc_reference.cpp:62
  v-------------->  arm::mapper::common::registerBuffer(native_handle const*)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             hardware/rockchip/libgralloc/midgard/src/hidl_common/Mapper.cpp:73
  000000000000fb0c  arm::mapper::common::importBuffer(android::hardware::hidl_handle const&, std::__1::function<void (android::hardware::graphics::mapper::V4_0::Error, void*)>)+124                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      hardware/rockchip/libgralloc/midgard/src/hidl_common/Mapper.cpp:274
  000000000000e450  arm::mapper::GrallocMapper::importBuffer(android::hardware::hidl_handle const&, std::__1::function<void (android::hardware::graphics::mapper::V4_0::Error, void*)>)+120                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               hardware/rockchip/libgralloc/midgard/src/4.x/GrallocMapper.cpp:64
  0000000000018d88  android::hardware::graphics::mapper::V4_0::BsMapper::importBuffer(android::hardware::hidl_handle const&, std::__1::function<void (android::hardware::graphics::mapper::V4_0::Error, void*)>)+152                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      out/soong/.intermediates/hardware/interfaces/graphics/mapper/4.0/android.hardware.graphics.mapper@4.0_genc++_headers/gen/android/hardware/graphics/mapper/4.0/BsMapper.h:73
  000000000002e96c  android::Gralloc4Mapper::importBuffer(android::hardware::hidl_handle const&, native_handle const**) const+100                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         frameworks/native/libs/ui/Gralloc4.cpp:144
  v-------------->  operator()<android::hardware::graphics::mapper::V2_0::Error, unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> >                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              frameworks/native/libs/ui/Gralloc2.cpp:404
  v-------------->  _ZNSt3__18__invokeIRZNK7android17Gralloc2Allocator8allocateENS_12basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEEEjjijmjPjPPK13native_handlebE3$_7JNS1_8hardware8graphics6mapper4V2_05ErrorEjRKNSG_8hidl_vecINSG_11hidl_handleEEEEEEDTclclsr3std3__1E7forwardIT_Efp_Espclsr3std3__1E7forwardIT0_Efp0_EEEOSQ_DpOSR_                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                external/libcxx/include/type_traits:4353
  v-------------->  void std::__1::__invoke_void_return_wrapper<void>::__call<android::Gralloc2Allocator::allocate(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >, unsigned int, unsigned int, int, unsigned int, unsigned long, unsigned int, unsigned int*, native_handle const**, bool) const::$_7&, android::hardware::graphics::mapper::V2_0::Error, unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> const&>(android::Gralloc2Allocator::allocate(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >, unsigned int, unsigned int, int, unsigned int, unsigned long, unsigned int, unsigned int*, native_handle const**, bool) const::$_7&, android::hardware::graphics::mapper::V2_0::Error&&, unsigned int&&, android::hardware::hidl_vec<android::hardware::hidl_handle> const&)                                                               external/libcxx/include/__functional_base:349
  v-------------->  std::__1::__function::__alloc_func<android::Gralloc2Allocator::allocate(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >, unsigned int, unsigned int, int, unsigned int, unsigned long, unsigned int, unsigned int*, native_handle const**, bool) const::$_7, std::__1::allocator<android::Gralloc2Allocator::allocate(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >, unsigned int, unsigned int, int, unsigned int, unsigned long, unsigned int, unsigned int*, native_handle const**, bool) const::$_7>, void (android::hardware::graphics::mapper::V2_0::Error, unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> const&)>::operator()(android::hardware::graphics::mapper::V2_0::Error&&, unsigned int&&, android::hardware::hidl_vec<android::hardware::hidl_handle> const&)                                                external/libcxx/include/functional:1527
  000000000002c474  std::__1::__function::__func<android::Gralloc2Allocator::allocate(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >, unsigned int, unsigned int, int, unsigned int, unsigned long, unsigned int, unsigned int*, native_handle const**, bool) const::$_7, std::__1::allocator<android::Gralloc2Allocator::allocate(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >, unsigned int, unsigned int, int, unsigned int, unsigned long, unsigned int, unsigned int*, native_handle const**, bool) const::$_7>, void (android::hardware::graphics::mapper::V2_0::Error, unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> const&)>::operator()(android::hardware::graphics::mapper::V2_0::Error&&, unsigned int&&, android::hardware::hidl_vec<android::hardware::hidl_handle> const&)+124                                                  external/libcxx/include/functional:1651
  v-------------->  std::__1::__function::__value_func<void (android::hardware::graphics::mapper::V4_0::Error, unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> const&)>::operator()(android::hardware::graphics::mapper::V4_0::Error&&, unsigned int&&, android::hardware::hidl_vec<android::hardware::hidl_handle> const&) const                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               external/libcxx/include/functional:1799
  v-------------->  std::__1::function<void (android::hardware::graphics::mapper::V4_0::Error, unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> const&)>::operator()(android::hardware::graphics::mapper::V4_0::Error, unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> const&) const                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   external/libcxx/include/functional:2347
  v-------------->  operator()                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            out/soong/.intermediates/hardware/interfaces/graphics/allocator/4.0/android.hardware.graphics.allocator@4.0_genc++/gen/android/hardware/graphics/allocator/4.0/AllocatorAll.cpp:222
  v-------------->  _ZNSt3__18__invokeIRZN7android8hardware8graphics9allocator4V4_013BpHwAllocator14_hidl_allocateEPNS2_10IInterfaceEPNS2_7details16HidlInstrumentorERKNS2_8hidl_vecIhEEjNS_8functionIFvNS3_6mapper4V4_05ErrorEjRKNSC_INS2_11hidl_handleEEEEEEE3$_3JRNS2_6ParcelEEEEDTclclsr3std3__1E7forwardIT_Efp_Espclsr3std3__1E7forwardIT0_Efp0_EEEOSU_DpOSV_                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        external/libcxx/include/type_traits:4353
  v-------------->  void std::__1::__invoke_void_return_wrapper<void>::__call<android::hardware::graphics::allocator::V4_0::BpHwAllocator::_hidl_allocate(android::hardware::IInterface*, android::hardware::details::HidlInstrumentor*, android::hardware::hidl_vec<unsigned char> const&, unsigned int, std::__1::function<void (android::hardware::graphics::mapper::V4_0::Error, unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> const&)>)::$_3&, android::hardware::Parcel&>(android::hardware::graphics::allocator::V4_0::BpHwAllocator::_hidl_allocate(android::hardware::IInterface*, android::hardware::details::HidlInstrumentor*, android::hardware::hidl_vec<unsigned char> const&, unsigned int, std::__1::function<void (android::hardware::graphics::mapper::V4_0::Error, unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> const&)>)::$_3&, android::hardware::Parcel&)                 external/libcxx/include/__functional_base:349
  v-------------->  std::__1::__function::__alloc_func<android::hardware::graphics::allocator::V4_0::BpHwAllocator::_hidl_allocate(android::hardware::IInterface*, android::hardware::details::HidlInstrumentor*, android::hardware::hidl_vec<unsigned char> const&, unsigned int, std::__1::function<void (android::hardware::graphics::mapper::V4_0::Error, unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> const&)>)::$_3, std::__1::allocator<android::hardware::graphics::allocator::V4_0::BpHwAllocator::_hidl_allocate(android::hardware::IInterface*, android::hardware::details::HidlInstrumentor*, android::hardware::hidl_vec<unsigned char> const&, unsigned int, std::__1::function<void (android::hardware::graphics::mapper::V4_0::Error, unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> const&)>)::$_3>, void (android::hardware::Parcel&)>::operator()(android::hardware::Parcel&)  external/libcxx/include/functional:1527
  0000000000010450  std::__1::__function::__func<android::hardware::graphics::allocator::V4_0::BpHwAllocator::_hidl_allocate(android::hardware::IInterface*, android::hardware::details::HidlInstrumentor*, android::hardware::hidl_vec<unsigned char> const&, unsigned int, std::__1::function<void (android::hardware::graphics::mapper::V4_0::Error, unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> const&)>)::$_3, std::__1::allocator<android::hardware::graphics::allocator::V4_0::BpHwAllocator::_hidl_allocate(android::hardware::IInterface*, android::hardware::details::HidlInstrumentor*, android::hardware::hidl_vec<unsigned char> const&, unsigned int, std::__1::function<void (android::hardware::graphics::mapper::V4_0::Error, unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> const&)>)::$_3>, void (android::hardware::Parcel&)>::operator()(android::hardware::Parcel&)+392    external/libcxx/include/functional:1651
  v-------------->  std::__1::__function::__value_func<void (android::hardware::Parcel&)>::operator()(android::hardware::Parcel&) const                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   external/libcxx/include/functional:1799
  v-------------->  std::__1::function<void (android::hardware::Parcel&)>::operator()(android::hardware::Parcel&) const                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   external/libcxx/include/functional:2347
  00000000000954d0  android::hardware::BpHwBinder::transact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>)+104                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          system/libhwbinder/BpHwBinder.cpp:111
  000000000000b5b8  android::hardware::graphics::allocator::V4_0::BpHwAllocator::_hidl_allocate(android::hardware::IInterface*, android::hardware::details::HidlInstrumentor*, android::hardware::hidl_vec<unsigned char> const&, unsigned int, std::__1::function<void (android::hardware::graphics::mapper::V4_0::Error, unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> const&)>)+400                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        out/soong/.intermediates/hardware/interfaces/graphics/allocator/4.0/android.hardware.graphics.allocator@4.0_genc++/gen/android/hardware/graphics/allocator/4.0/AllocatorAll.cpp:178
  000000000000b8b8  android::hardware::graphics::allocator::V4_0::BpHwAllocator::allocate(android::hardware::hidl_vec<unsigned char> const&, unsigned int, std::__1::function<void (android::hardware::graphics::mapper::V4_0::Error, unsigned int, android::hardware::hidl_vec<android::hardware::hidl_handle> const&)>)+168                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             out/soong/.intermediates/hardware/interfaces/graphics/allocator/4.0/android.hardware.graphics.allocator@4.0_genc++/gen/android/hardware/graphics/allocator/4.0/AllocatorAll.cpp:253
  0000000000033958  android::Gralloc4Allocator::allocate(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >, unsigned int, unsigned int, int, unsigned int, unsigned long, unsigned int, unsigned int*, native_handle const**, bool) const+312                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         frameworks/native/libs/ui/Gralloc4.cpp:1082
  00000000000380a8  android::GraphicBufferAllocator::allocateHelper(unsigned int, unsigned int, int, unsigned int, unsigned long, native_handle const**, unsigned int*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >, bool)+312                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  frameworks/native/libs/ui/GraphicBufferAllocator.cpp:140
  00000000000384a0  android::GraphicBufferAllocator::allocate(unsigned int, unsigned int, int, unsigned int, unsigned long, native_handle const**, unsigned int*, unsigned long, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >)+136                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               frameworks/native/libs/ui/GraphicBufferAllocator.cpp:199
  0000000000036374  android::GraphicBuffer::initWithSize(unsigned int, unsigned int, int, unsigned int, unsigned long, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >)+212                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         frameworks/native/libs/ui/GraphicBuffer.cpp:196
  v-------------->  GraphicBuffer                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         frameworks/native/libs/ui/GraphicBuffer.cpp:87
  0000000000036180  android::GraphicBuffer::GraphicBuffer(unsigned int, unsigned int, int, unsigned int, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >)+128                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       frameworks/native/libs/ui/GraphicBuffer.cpp:80
  00000000000010ec  main+164                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              /data/test-opengl-gralloc
  0000000000049a34  __libc_init+108                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       bionic/libc/bionic/libc_init_dynamic.cpp:151

```

![image-20220829152943138](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/Android_Display_System/Android11_Display01/image-20220829152943138.png)

> frameworks/native/libs/ui/Gralloc2.cpp:404

![image-20220829153238532](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/Android_Display_System/Android11_Display01/image-20220829153238532.png)

> frameworks/native/libs/ui/.cpp:144

![image-20220829153337875](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/Android_Display_System/Android11_Display01/image-20220829153337875.png)

> out/soong/.intermediates/hardware/interfaces/graphics/mapper/4.0/android.hardware.graphics.mapper@4.0_genc++_headers/gen/android/hardware/graphics/mapper/4.0/BsMapper.h:73

![image-20220829155440839](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/Android_Display_System/Android11_Display01/image-20220829155440839.png)

> hardware/rockchip/libgralloc/midgard/src/4.x/GrallocMapper.cpp:64

![image-20220829155545041](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/Android_Display_System/Android11_Display01/image-20220829155545041.png)

> hardware/rockchip/libgralloc/midgard/src/hidl_common/Mapper.cpp:73                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          hardware/rockchip/libgralloc/midgard/src/hidl_common/Mapper.cpp:274

![image-20220829155807324](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/Android_Display_System/Android11_Display01/image-20220829155807324.png)



![image-20220829155715838](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/Android_Display_System/Android11_Display01/image-20220829155715838.png)

> hardware/rockchip/libgralloc/midgard/src/allocator/mali_gralloc_ion.cpp:987

![image-20220829155932726](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/Android_Display_System/Android11_Display01/image-20220829155932726.png)



## 五、freeBuffer

```c++
Stack Trace:
  RELADDR           FUNCTION                                                                    FILE:LINE
  00000000000202f8  mali_gralloc_ion_unmap(private_handle_t*)+96                                hardware/rockchip/libgralloc/midgard/src/allocator/mali_gralloc_ion.cpp:1013
  000000000001ded0  mali_gralloc_reference_release(native_handle const*, bool)+392              hardware/rockchip/libgralloc/midgard/src/core/mali_gralloc_reference.cpp:120
  v-------------->  arm::mapper::common::unregisterBuffer(native_handle const*)                 hardware/rockchip/libgralloc/midgard/src/hidl_common/Mapper.cpp:97
  000000000000fd60  arm::mapper::common::freeBuffer(void*)+168                                  hardware/rockchip/libgralloc/midgard/src/hidl_common/Mapper.cpp:335
  000000000000e4c4  arm::mapper::GrallocMapper::freeBuffer(void*)+20                            hardware/rockchip/libgralloc/midgard/src/4.x/GrallocMapper.cpp:70
  0000000000018f58  android::hardware::graphics::mapper::V4_0::BsMapper::freeBuffer(void*)+120  out/soong/.intermediates/hardware/interfaces/graphics/mapper/4.0/android.hardware.graphics.mapper@4.0_genc++_headers/gen/android/hardware/graphics/mapper/4.0/BsMapper.h:105
  000000000002ea24  android::Gralloc4Mapper::freeBuffer(native_handle const*) const+60          frameworks/native/libs/ui/Gralloc4.cpp:157
  000000000003959c  android::GraphicBufferMapper::freeBuffer(native_handle const*)+44           frameworks/native/libs/ui/GraphicBufferMapper.cpp:122
  0000000000038528  android::GraphicBufferAllocator::free(native_handle const*)+56              frameworks/native/libs/ui/GraphicBufferAllocator.cpp:209
  0000000000036834  android::GraphicBuffer::free_handle()+132                                   frameworks/native/libs/ui/GraphicBuffer.cpp:126
  0000000000036690  android::GraphicBuffer::~GraphicBuffer()+72                                 frameworks/native/libs/ui/GraphicBuffer.cpp:113
  0000000000036860  android::GraphicBuffer::~GraphicBuffer()+16                                 frameworks/native/libs/ui/GraphicBuffer.cpp:110
  0000000000010a48  android::RefBase::decStrong(void const*) const+112                          system/core/libutils/RefBase.cpp:447
  0000000000001144  main+252                                                                    /data/test-opengl-gralloc
  0000000000049a34  __libc_init+108                                                             bionic/libc/bionic/libc_init_dynamic.cpp:151


```

流程跟importbuffer类似，直接看看具体实现。

> hardware/rockchip/libgralloc/midgard/src/hidl_common/Mapper.cpp:97
> hardware/rockchip/libgralloc/midgard/src/hidl_common/Mapper.cpp:335

![image-20220829161307997](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/Android_Display_System/Android11_Display01/image-20220829161307997.png)

![image-20220829161343993](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/Android_Display_System/Android11_Display01/image-20220829161343993.png)

![image-20220829161432203](https://raw.githubusercontent.com/zhoujinjianmm/PicGo/master/Android_Display_System/Android11_Display01/image-20220829161432203.png)
