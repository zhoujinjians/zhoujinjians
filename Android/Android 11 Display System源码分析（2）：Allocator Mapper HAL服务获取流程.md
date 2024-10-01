---
title: Android 11 Display System源码分析（2）：IAllocator IMapper HAL服务获取流程（V1）
cover: https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/post.cover.pictures/bing-wallpaper-2018.04.42.jpg
categories: 
 - Display
tags:
 - Display
toc: true
abbrlink: 20220716
date: 2022-07-16 16:16:16
---
注：文章都是通过阅读各位前辈总结的资料、Android 11 Rockchip平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除（◕‿◕），转载请注明出处（©Rockchip ©Android @Linux 版权所有），谢谢。

（==**文章基于 Android 11.0**==）

[【zhoujinjian.com博客原图链接】](https://github.com/zhoujinjianm) 

[【开发板】](https://wiki.radxa.com/Rockpi4)

[【开发板 源码链接】](https://wiki.radxa.com/Rockpi4/rockpi-android11)



正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢，由于不喜欢图片水印，去除了水印，敬请谅解！！！

 [Native层HIDL服务的注册原理-Android10.0 HwBinder通信原理(六)](https://blog.csdn.net/yiranfeng/article/details/107966454) 

 [Native层HIDL服务的获取原理-Android10.0 HwBinder通信原理(七)](https://blog.csdn.net/yiranfeng/article/details/108037473) 


--------------------------------------------------------------------------------

==源码（部分）==：

**/lagvm/lagvm/LINUX/android/frameworks/native/opengl/tests/gralloc/gralloc.cpp**

**/lagvm/lagvm/LINUX/android/frameworks/native/libs/ui/**

**Y:\home\zhoujinjian\android11_rockpi4\hardware\rockchip\libgralloc\midgard**

**Y:\home\zhoujinjian\android11_rockpi4\hardware\rockchip\libgralloc\bifrost\service\4.x**

==源码（部分）==：

--------------------------------------------------------------------------------

## 一、IAllocator服务启动注册流程
----------------------

还记得第一篇章说的android.hardware.graphics.allocator@4.0-service服务吗。

Binder Bn服务端就是android.hardware.graphics.allocator@4.0-service（Hal Service）。

```c++
system          233      1 10834060 10388 binder_thread_read  0 S android.hardware.graphics.allocator@4.0-service
```

```
service vendor.gralloc-4-0 /vendor/bin/hw/android.hardware.graphics.allocator@4.0-service
    class hal animation
    interface android.hardware.graphics.allocator@4.0::IAllocator default
    user system
    group graphics drmrpc
    capabilities SYS_NICE
    onrestart restart surfaceflinger
```

系统通过android.hardware.graphics.allocator@4.0-service.rc启动服务，继续跟踪一下源码。

只有简单的一句：

```c++
#define LOG_TAG "android.hardware.graphics.allocator@4.0-service"

#include <android/hardware/graphics/allocator/4.0/IAllocator.h>

#include <hidl/LegacySupport.h>

using android::hardware::defaultPassthroughServiceImplementation;
using android::hardware::graphics::allocator::V4_0::IAllocator;

int main() {
    return defaultPassthroughServiceImplementation<IAllocator>(/*maxThreads*/ 4);
}
```

> IAllocator的服务注册流程分为以下几步：
>
> 1.获得ProcessState实例对象，与"/dev/hwbinder"进行通信，设置最大的线程个数为4
>
> 2.获取IAllocator服务对象
>
> 3.调用registerAsServiceInternal注册IAllocator服务
>
> 4.把当前线程加入到线程池
>
> 服务注册，主要的入口就是 registerAsServiceInternal()，下面我们从registerAsServiceInternal()进一步分析
>
>  

> Y:\home\zhoujinjian\android11_rockpi4\system\libhidl\transport\include\hidl\LegacySupport.h

![image-20220811203557089](https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/Android_Display_System/Android11_Display02/image-20220811203557089.png)



![image-20220811204440115](https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/Android_Display_System/Android11_Display02/image-20220811204440115.png)

> Y:\home\zhoujinjian\android11_rockpi4\system\libhidl\transport\LegacySupport.cpp

![image-20220811211030918](https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/Android_Display_System/Android11_Display02/image-20220811211030918.png)

![image-20220811205128732](https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/Android_Display_System/Android11_Display02/image-20220811205128732.png)

registerServiceCb即为registerAsServiceInternal

> Y:\home\zhoujinjian\android11_rockpi4\system\libhidl\transport\ServiceManagement.cpp

![image-20220811211423210](https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/Android_Display_System/Android11_Display02/image-20220811211423210.png)

> ## BpHwServiceManager::addWithChain()
>
> Y:\home\zhoujinjian\android11_rockpi4\out\soong\.intermediates\system\libhidl\transport\manager\1.2\android.hidl.manager@1.2_genc++\gen\android\hidl\manager\1.2\ServiceManagerAll.cpp

![image-20220829102350669](https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/Android_Display_System/Android11_Display02/image-20220829102350669.png)

![image-20220829103006883](https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/Android_Display_System/Android11_Display02/image-20220829103006883.png)

> ## ServiceManager::addWithChain()
>
> Y:\home\zhoujinjian\android11_rockpi4\system\hwservicemanager\ServiceManager.cpp

![image-20220829103242379](https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/Android_Display_System/Android11_Display02/image-20220829103242379.png)

![image-20220829103355828](https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/Android_Display_System/Android11_Display02/image-20220829103355828.png)

> ## ServiceManager::insertService()

![image-20220829103507527](https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/Android_Display_System/Android11_Display02/image-20220829103507527.png)

至此，我们的IAllocator服务注册完成，注册步骤分为以下几个步骤：

> 1.服务进程先获得一个 ProcessState()的对象
>
> 2.获取HwServiceManager的代理对象HwBpServiceManager，主要通过new BpHwBinder(0)得到。
>
> 3.调用HwBpServiceManager的addWithChain，组装一个Parce数据，传入服务名称、服务实体对象--BHwBinder、执行12 /* addWithChain */
>
> 4.先通过IPCThreadThread的writeTransactionData()把上面的Parcel数据写入mOut，用来进行发送
>
> 5.通过IPCThreadThread的talkWithDriver()与HwBinder驱动通信，传递语义BINDER_WRITE_READ
>
> 6.HwServiceManager有个for循环，调用epoll_wait()监控hwbinder的变化
>
> 7.waitForResponse()收到BR_TRANSACTION响应码后，最终后调用executeCommand()进行命令执行，根据逻辑处理，最终会调用到BnHwServiceManager::onTransact()，根据传入的code=12,调用addWithChain，最终把服务名称和实体对象插入到mServiceMap这个map中
>
> 8.注册成功后，把注册结果写入reply，返回reply信息
>
> 9.最终完成服务的注册流程

二、IAllocator服务获取流程
----------------

> ## getService()

![image-20220829133030627](https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/Android_Display_System/Android11_Display02/image-20220829133030627.png)

getService的调用栈如下：

![img](https://img-blog.csdnimg.cn/20200816160045466.jpg)

> ## getRawServiceInternal()

![image-20220829133159318](https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/Android_Display_System/Android11_Display02/image-20220829133159318.png)

getRawServiceInternal()步骤如下：

1.获取HwServiceManager的代理对象

2.获取IDemo 这个hidl服务的transport，来确认是绑定式服务，还是直通式服务

3.根据transport类型，调用不同的接口来获取服务对象 (我们设计的IDemo是绑定式hidl服务)

> Y:\home\zhoujinjian\android11_rockpi4\out\soong\.intermediates\system\libhidl\transport\manager\1.2\android.hidl.manager@1.2_genc++\gen\android\hidl\manager\1.2\ServiceManagerAll.cpp

![image-20220829133519664](https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/Android_Display_System/Android11_Display02/image-20220829133519664.png)

> Y:\home\zhoujinjian\android11_rockpi4\out\soong\.intermediates\system\libhidl\transport\manager\1.0\android.hidl.manager@1.0_genc++\gen\android\hidl\manager\1.0\ServiceManagerAll.cpp

![image-20220829133839642](https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/Android_Display_System/Android11_Display02/image-20220829133839642.png)

> _hidl_get流程如下：
>
>   1.获取hidl服务的fqName
>
>    2.获取hidl服务的name
>
>    3.根据hidl服务的fqName和name，从HwServiceManager的hidl service的map中拿到服务对象
>
>    4.把reply信息写入Parcel
>
>    5.把获取的hidl服务转成IBinder对象
>
>    6.把转换后的IBinder对象，写入reply的Parcel数据中
>
>    7.通过回调，把reply的Parcel数据给发给client



> ## ServiceManager::get()

![image-20220829141027593](https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/Android_Display_System/Android11_Display02/image-20220829141027593.png)

```c++
[/system/hwservicemanager/HidlService.cpp]
sp<IBase> HidlService::getService() const {
    return mService;
}
```





