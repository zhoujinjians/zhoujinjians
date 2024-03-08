---
title: Android O Treble 架构 - HIDL源代码分析➺➺➺(๑乛◡乛๑)➺➺➺
cover: https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/hexo.themes/bing-wallpaper-2018.04.28.jpg
categories: 
  - HIDL
tags:
  - Android
  - Treble
  - HIDL
toc: true
abbrlink: 20190308
date: 2019-03-08 09:25:00
---


--------------------------------------------------------------------------------

注：文章都是通过阅读各位前辈总结的资料 Android 8.x && Linux（kernel 4.x）Qualcomm平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网（◕‿◕），如有侵权，请联系删除，禁止转载（©Qualcomm ©Android @Linux 版权所有），谢谢。

首先感谢：

[【YANGWEN123】Android O Treble架构（系列分析文章）](https://blog.csdn.net/yangwen123)
[【Karim Yaghmour】Android's HIDL: Treble in the HAL - SlideShare](https://www.slideshare.net/opersys/androids-hidl-treble-in-the-hal)

正是由于前人的分析和总结，帮助我节约了大量的时间和精力，再次感谢！！！

Google Pixel、Pixel XL 内核代码（==**文章基于 Kernel-4.x**==）：
 [Kernel source for Pixel 2 (walleye) and Pixel 2 XL (taimen) - GitHub](https://github.com/nathanchance/wahoo)

AOSP 源码（==**文章基于 Android 8.x**==）：
 [ Android 系统全套源代码分享 (更新到 8.1.0_r1)](https://testerhome.com/topics/2229)

--------------------------------------------------------------------------------
==源码（部分）==：

> Java框架

/frameworks/base/core/java/android/os/（JAVA）
- IHwInterface.java
- HwBinder.java
- HwRemoteBinder.java
- IHwBinder.java
- HwParcel.java

/frameworks/base/core/jni/（JNI）
- android_os_HwRemoteBinder.cpp
- android_os_HwBinder.cpp
- android_os_HwParcel.cpp

> Native框架

/system/libhwbinder/
- Binder.cpp
- BpHwBinder.cpp
- IInterface.cpp
- IPCThreadState.cpp
- Parcel.cpp
- ProcessState.cpp

/system/libhidl/transport/
- HidlBinderSupport.cpp
- HidlTransportSupport.cpp
- ServiceManagement.cpp
- /manager/1.0/IServiceManager.hal
- /manager/1.0/IServiceNotification.hal

/system/hwservicemanager/
- HidlService.cpp
- hwservicemanager.rc
- ServiceManager.cpp
- service.cpp


> Binder Driver（Kernel）

/drivers/android/
- binder.c
- binder_alloc.c

> 源码编译生成路径（Java）：

\out\target\common\gen\JAVA_LIBRARIES
- android.hardware.light-V2.0-java_intermediates
- android.hidl.manager-V1.0-java_intermediates
- ......

> 源码编译生成路径（hardware/interfaces/）：

\out\soong\.intermediates\system\
- libhidl\transport\manager\1.1\android.hidl.manager@1.1_genc++

\out\soong\.intermediates\hardware\interfaces\*
- light\2.0\android.hardware.light@2.0_genc++
- audio\2.0\android.hardware.audio@2.0_genc++
- camera\*
- graphics\*
- media\*
- wifi\*
- ......


--------------------------------------------------------------------------------
> 注：文中图文许多参考[快乐安卓](https://blog.csdn.net/yangwen123/)，由于博主有点小小的强迫症，把图片水印都去掉了，请各位见谅！

#### （一）、Android O Treble 架构介绍
###### 1.0、Android整体架构变化（VNDK、VINTF、HIDL）

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-01-HIDL-treble-after.png.png)

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-02-HIDL-treble-architecture.png.png)

AndroidO引入Treble架构后的变化:

1. 增加了2个服务管家，Android O 之前版本有且只有一个servicemanager，现在增加到3个(servicemanager、hwservicemanager、vndservicemanager)，他们分管不同的服务。

2. 增加了binder通信库，这是为了适配binder域的扩展。

3. 增加了binder域，系统定义了3个binder设备节点，binder驱动分别处理这3个binder设备节点上的binder通信事件。

###### 1.1、Binder通信域变化

Treble架构的引入足以说明Binder通信的重要性，之前APP和Framework之间通过binder实现跨进程调用，当然这个调用对开发者来说是透明的，相当于函数本地调用。Treble引入后，Framework和HAL又实现了进程分离，Framework和HAL之间依然使用binder通信，通过HIDL来定义通信接口。那binder通信有什么变化呢？ 在Treble中，引入了多个binder域，主要是增加了多个binder设备，binder驱动实现原理基本没变，变化了一些细节。增加binder设备应该是为了实现更换的权限控制，使用不同binder设备的主体和客体之间的selinux权限有所不同，同时，Android 框架和 HAL 现在使用 Binder 互相通信。由于这种通信方式极大地增加了 Binder 流量。

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-03-HIDL-Binder-enhancement.png)


为了明确地拆分框架（与设备无关）和供应商（与具体设备相关）代码之间的 Binder 流量，Android O 引入了“Binder 上下文”这一概念。每个 Binder 上下文都有自己的设备节点和上下文（服务）管理器。您只能通过上下文管理器所属的设备节点对其进行访问，并且在通过特定上下文传递 Binder 节点时，只能由另一个进程从相同的上下文访问上下文管理器，从而确保这些域完全互相隔离。为了显示 /dev/vndbinder，请确保内核配置项 CONFIG_ANDROID_BINDER_DEVICES 设为"binder,hwbinder,vndbinder"

``` cpp
[->kernel/android/configs/android-base.cfg]
CONFIG_ANDROID_BINDER_DEVICES=binder,hwbinder,vndbinder


[->kernel/drivers/android/binder.c]
static char *binder_devices_param = CONFIG_ANDROID_BINDER_DEVICES;
module_param_named(devices, binder_devices_param, charp, S_IRUGO);

static int __init binder_init(void)
{
	int ret;
	......
	/*
	 * Copy the module_parameter string, because we don't want to
	 * tokenize it in-place.
	 */
	device_names = kzalloc(strlen(binder_devices_param) + 1, GFP_KERNEL);
	......
	strcpy(device_names, binder_devices_param);

	while ((device_name = strsep(&device_names, ","))) {
		ret = init_binder_device(device_name);
		......
	}

	return ret;
	......
}
static int __init init_binder_device(const char *name)
{
	int ret;
	struct binder_device *binder_device;

	binder_device = kzalloc(sizeof(*binder_device), GFP_KERNEL);
	......
	ret = misc_register(&binder_device->miscdev);
	......
	hlist_add_head(&binder_device->hlist, &binder_devices);

	return ret;
}
```
以kernel参数形式得到配置的binder设备节点名称，然后在binder驱动中创建不同的binder设备：
这样在驱动中就创建了binder、vndbinder、hwbinder三个驱动设备，并保存在binder设备列表binder_devices中。/dev/binder 设备节点成为了框架进程的专属节点，这意味着oem进程将无法再访问该节点。oem进程可以访问 /dev/hwbinder，但必须将其 AIDL 接口转为使用 HIDL。

###### 1.2、vndbinder && vndservicemanager  
一直以来，供应商进程都使用 Binder 进程间通信 (IPC) 技术进行通信。在 Android O 中，/dev/binder 设备节点成为了框架进程的专属节点，这意味着供应商进程将无法再访问该节点。供应商进程可以访问 /dev/hwbinder，但必须将其 AIDL 接口转为使用 HIDL。对于想要继续在供应商进程之间使用 AIDL 接口的供应商，Android 会按以下方式支持 Binder IPC。

###### 1.2.1、vndbinder
Android O 支持供应商服务使用新的 Binder 域，这可通过使用 /dev/vndbinder（而非 /dev/binder）进行访问。添加 /dev/vndbinder 后，Android 现在拥有以下 3 个 IPC 域：

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-04-HIDL-hwbinder-vndbinder.png)


通常，供应商进程不直接打开 Binder 驱动程序，而是链接到打开 Binder 驱动程序的 libbinder 用户空间库。为 ::android::ProcessState() 添加方法可为 libbinder 选择 Binder 驱动程序。供应商进程应该在调用 ProcessState,、IPCThreadState 或发出任何普通 Binder 调用之前调用此方法。要使用该方法，请在供应商进程（客户端和服务器）的 main() 后放置以下调用：

ProcessState::initWithDriver("/dev/vndbinder");
###### 1.2.2、vndservicemanager

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-05-HIDL-hwservicemanager.png)


以前，Binder 服务通过 servicemanager 注册，其他进程可从中检索这些服务。在 Android O 中，servicemanager 现在专用于框架和应用进程，供应商进程无法再对其进行访问。

不过，供应商服务现在可以使用 vndservicemanager，这是一个使用 /dev/vndbinder（**作为构建基础的源代码与框架 servicemanager 相同/frameworks/native/cmds/servicemanager/**）而非 /dev/binder 的 servicemanager 的新实例。供应商进程无需更改即可与 vndservicemanager 通信；当供应商进程打开 /dev/vndbinder 时，服务查询会自动转至 vndservicemanager。

vndservicemanager 二进制文件包含在 Android 的默认设备 Makefile 中。
servicemanager和vndservicemanager使用的是同一份代码，都是由service_manager.c编译而来。
``` cpp
[->/frameworks/native/cmds/servicemanager/]
Android.bp
bctest.c
binder.c
binder.h
service_manager.c
servicemanager.rc
vndservicemanager.rc

[->/frameworks/native/cmds/servicemanager/service_manager.c]
int main(int argc, char** argv)
{
    struct binder_state *bs;
    union selinux_callback cb;
    char *driver;

    if (argc > 1) {
        driver = argv[1]; /* /dev/vndbinder */
    } else {
        driver = "/dev/binder";
    }

    bs = binder_open(driver, 128*1024);
    ......
    if (binder_become_context_manager(bs)) {
        ALOGE("cannot become context manager (%s)\n", strerror(errno));
        return -1;
    }

    cb.func_audit = audit_callback;
    selinux_set_callback(SELINUX_CB_AUDIT, cb);
    cb.func_log = selinux_log_callback;
    selinux_set_callback(SELINUX_CB_LOG, cb);

#ifdef VENDORSERVICEMANAGER
    sehandle = selinux_android_vendor_service_context_handle();
#else
    sehandle = selinux_android_service_context_handle();
#endif
    selinux_status_open(true);
    ......
    binder_loop(bs, svcmgr_handler);
    return 0;
}

```

在启动servicemanager 时，并没有传参，而启动vndservicemanager时，传递了binder设备节点。
> service servicemanager /system/bin/servicemanager
> service vndservicemanager /vendor/bin/vndservicemanager ==**/dev/vndbinder**==
``` cpp
[->/frameworks/native/cmds/servicemanager/vndservicemanager.rc]
service vndservicemanager /vendor/bin/vndservicemanager /dev/vndbinder
    class core
    user system
    group system readproc
    writepid /dev/cpuset/system-background/tasks
    shutdown critical
```

``` cpp
[->/frameworks/native/cmds/servicemanager/servicemanager.rc]
service servicemanager /system/bin/servicemanager
    class core animation
    user system
    group system readproc
    critical
    onrestart restart healthd
    onrestart restart zygote
    onrestart restart audioserver
    onrestart restart media
    onrestart restart surfaceflinger
    onrestart restart inputflinger
    onrestart restart drm
    onrestart restart cameraserver
    onrestart restart keystore
    onrestart restart gatekeeperd
    writepid /dev/cpuset/system-background/tasks
    shutdown critical

```
###### 1.2.3、servicemanager

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-05-HIDL-servicemanager.png.png)


###### 1.3、hwservicemanager
hwservicemanager用于管理hidl服务，因此其实现和servicemanager完全不同，使用的binder库也完全不同。

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-05-HIDL-vndservicemanager.png.png)

###### 1.4、Binder库变化
servicemanager和vndservicemanager都使用libbinder库，只是他们使用的binder驱动不同而已，而hwservicemanager使用libhwbinder库，binder驱动也不同。

libbinder库源码(\frameworks\native\libs\binder)
![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-06-libbinder-sourcecode.png)

libhwbinder库源码：
![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-07-libhwbinder-sourcecode.png)

文件对比：
![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-08-HIDL-binder-hwbinder-file-compare.png)


###### 1.5、Binder通信框架变化

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-09-HIDL-binder-binder_ipc_process.jpg)


1.在普通Java Binder框架中，Client端Proxy类完成数据打包，然后交给mRemotebinder代理来完成数据传输。Server端Stub类完成数据解析，然后交给其子类实现。

2.在普通Native Binder框架中，Client端BpXXX类完成数据打包，然后交给mRemoteBpBinder来完成数据传输。Server端BnXXX类完成数据解析，然后交个其子类实现。

3.  在HwBinder框架中，Client端的BpHwXXX类完成数据打包，然后交给mRemoteBpHwBinder来完成数据传输。Server端的BnHwXXX类完成数据解析，然后交给_hidl_mImpl来实现。

###### 1.6、框架层Binder对象变化
参考：[【AndroidO Treble架构下的变化】](https://blog.csdn.net/yangwen123/article/details/79836109)
![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-10-HIDL-binder-hwbinder-fw-compare.png)
![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-11-HIDL-hwbinder-object.png)

#### （二）、Android O Treble 之 HIDL文件（.hal文件）接口文件编译

HIDL是一种接口定义语言，描述了HAL和它的用户之间的接口。同aidi类似，我们只需要为hal定义相关接口，然后通过hidl-gen工具即可自动编译生成对应的C++或者java源文件，定义hal接口的文件命名为xxx.hal，为了编译这些.hal文件，需要编写相应的Android.bp或者Android.mk文件:

1. Android.bp文件用于编译C++；

2. Android.mk文件用于编译Java；

##### 2.1、生成子Android.mk和Android.bp文件
所有的HIDL Interface 都是通过一个.hal文件来描述，为了方便编译生成每一个子hal。Google在系统默认提供了一个脚本update-makefiles.sh，位于hardware/interfaces/、frameworks/hardware/interfaces/、system/hardware/interfaces/、system/libhidl/。以hardware/interfaces/里面的代码为实例做介绍：

``` cpp
#!/bin/bash
 
source system/tools/hidl/update-makefiles-helper.sh
 
do_makefiles_update \
  "android.hardware:hardware/interfaces" \

```
这个脚本的主要作用：根据hal文件生成Android.mk(makefile)和Android.bp(blueprint)文件。在hardware/interfaces的子目录里面，存在.hal文件的目录，都会产生Android.bp和Android.mk文件。详细分析如下：
1. source system/tools下面的update-makefiles-helper.sh，然后执行do_makefiles_update
2. 解析传入进去的参数。参数android.hardware:hardware/interfaces:
    android.hardware: android.hardware表示包名。
    hardware/interfaces：表示相对于根目录的文件路径。
    会输出如下LOG：

``` cpp
Updating makefiles for android.hardware in hardware/interfaces.
Updating ….
```
3.获取所有的包名。通过function get_packages()函数，获取hardware/interfaces路径下面的所有hal文件所在的目录路径，比如子目录Light里面的hal文件的路径是\hardware\interfaces\light\2.0，加上当前的参数包名hardware/interfaces，通过点的方式连接，将light/2.0+hardware/interfaces里面的斜线转换成点,最终获取的包名就是 android.hardware.light@2.0，依次类推获取所有的包名。
4.执行hidl-gen命令.将c步骤里面获取的参数和包名还有类名传入hidl-gen命令，在\hardware\interfaces\light\2.0目录下产生Android.mk和Android.bp文件。
    Android.mk: hidl-gen -Lmakefile -r android.hardware:hardware/interfaces -r android.hidl:system/libhidl/transport android.hardware.light@2.0
    
![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-12-HIDL-android-hal-make.png)


编译最终在./out/target/common/gen/JAVA_LIBRARIES目录下生成Java源文件。

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-13-HIDL-android-make-java.png)

    Android.bp: hidl-gen -Landroidbp -r android.hardware:hardware/interfaces -r android.hidl:system/libhidl/transport android.hidl.manager@1.0

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-14-HIDL-android-hal-manager.png)




编译最终在./out/soong/.intermediates/hardware/interfaces目录下生成C++源文件。
![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-15-HIDL-android-hal-manager-gen-c++.png)



5.在hardware/interfaces的每个子目录下面产生Android.bp文件，文件内容主要是subdirs的初始化，存放当前目录需要包含的子目录。比如hardware/interfaces/light/下面的Android.bp文件。

经过以上步骤，就会在对应的子目录产生Android.mk和Android.bp文件。这样以后我们就可以执行正常的编译命令进行编译了。比如mmm hardware/interfaces/light/,默认情况下，在源码中，Android.mk和Android.bp文件已经存在。


##### 2.2、hidl-gen工具
在Treble架构中，系统定义的所有的.hal接口，都是通过hidl-gen工具转换成对应的代码。比如\hardware\interfaces\light\2.0\ILight.hal，会通过hidl-gen转换成\out\soong\.intermediates\hardware\interfaces\light\2.0\android.hardware.light@2.0_genc++\gen\android\hardware\light\2.0\LightAll.cpp文件。
hidl-gen源码路径：system/tools/hidl，是在ubuntu上可执行的二进制文件。
使用方法：hidl-gen -o output-path -L language (-r interface-root) fqname
参数含义：
    -L： 语言类型，包括c++, c++-headers, c++-sources, export-header, c++-impl, java, java-constants, vts, makefile, androidbp, androidbp-impl, hash等。hidl-gen可根据传入的语言类型产生不同的文件。fqname：完全限定名称的输入文件。比如本例中android.hardware.light@2.0，要求在源码目录下必须有hardware/interfaces/light/2.0/目录。
        对于单个文件来说，格式如下：package@version::fileName，比如android.hardware.light@1.0::types.Feature。
        对于目录来说。格式如下package@version，比如android.hardware.light@2.0。
    -r： 格式package:path，可选，对fqname对应的文件来说，用来指定包名和文件所在的目录到Android系统源码根目录的路径。如果没有制定，前缀默认是：android.hardware，目录是Android源码的根目录。
    -o ： 存放hidl-gen产生的中间文件的路径。我们查看\hardware\interfaces\light\2.0\Android.bp，可以看到，-o参数都是写的$(genDir),一般都是在\out\soong\.intermediates\hardware\interfaces\light\2.0\下面，根据-L的不同，后面产生的路径可能不太一样，比如c++，那么就会就是\out\soong\.intermediates\hardware\interfaces\light\2.0\gen，如果是c++-headers，那么就是\out\soong\.intermediates\hardware\interfaces\light\2.0\android.hardware.light@2.0_genc++_headers\gen。


对于实例来说，fqname是：android.hardware.light@2.0，包名是android.hardware，文件所在的目录是hardware/interfaces。例子中的命令会在\out\soong\.intermediates\hardware\interfaces\light\2.0\下面产生对应的c++文件。

在\hardware\interfaces\light\2.0\目录下mm编译将生成：

\system\lib64\android.hardware.light@2.0.so
\symbols\vendor\lib64\hw\android.hardware.light@2.0-impl.so
\vendor\etc\init\android.hardware.light@2.0-service.rc
\vendor\bin\hw\android.hardware.light@2.0-service

android.hardware.light@2.0-service为hal进程的可执行文件，在android.hardware.light@2.0-service.rc是hal进程启动的配置脚本文件：

``` cpp
[->\hardware\interfaces\light\2.0\default\android.hardware.light@2.0-service.rc]

service light-hal-2-0 /vendor/bin/hw/android.hardware.light@2.0-service
    class hal
    user system
    group system
```

也就是说AndroidO的Treble架构下，所有hal都运行在独立的进程空间：

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-16-HIDL-hal-process-.png)



#### （三）、Android O Treble 之 hwservicemanager 启动过程

hwservicemanager是hidl服务管理中心，负责管理系统中的所有hidl服务，由init进程启动。

``` cpp
[->/system/core/rootdir/init.rc]
on post-fs
    # Load properties from
    #     /system/build.prop,
    #     /odm/build.prop,
    #     /vendor/build.prop and
    #     /factory/factory.prop
    load_system_props
    # start essential services
    start logd
    start servicemanager
    start hwservicemanager
    start vndservicemanager

```
可以看到在文件系统刚初始化没多久，就启动了系统非常重要的三个管理服务，接下来分析hwservicemanager的启动流程。

``` cpp
[->\system\hwservicemanager\hwservicemanager.rc]
service hwservicemanager /system/bin/hwservicemanager
    user system
    disabled
    group system readproc
    critical
    onrestart setprop hwservicemanager.ready false
    onrestart class_restart hal
    onrestart class_restart early_hal
    writepid /dev/cpuset/system-background/tasks
    class animation
```
hwservicemanager的源码位于system\hwservicemanager\s。
我们从system\hwservicemanager\service.cpp 的main()入口函数开始：

``` cpp
[->system\hwservicemanager\service.cpp]
int main() {
    configureRpcThreadpool(1, true /* callerWillJoin */);
	//创建ServiceManager对象
    ServiceManager *manager = new ServiceManager();
	//将ServiceManager对象自身注册到mServiceMap表中
    if (!manager->add(serviceName, manager)) {
        ALOGE("Failed to register hwservicemanager with itself.");
    }
	//创建TokenManager对象
    TokenManager *tokenManager = new TokenManager();
	//将TokenManager对象自身注册到mServiceMap表中
    if (!manager->add(serviceName, tokenManager)) {
        ALOGE("Failed to register ITokenManager with hwservicemanager.");
    }
	//建立消息循环
    sp<Looper> looper(Looper::prepare(0 /* opts */));
 
    int binder_fd = -1;
	//将主线程加入binder线程池，并得到/dev/hwbinder句柄
    IPCThreadState::self()->setupPolling(&binder_fd);
    if (binder_fd < 0) {
        ALOGE("Failed to aquire binder FD. Aborting...");
        return -1;
    }
    // Flush after setupPolling(), to make sure the binder driver
    // knows about this thread handling commands.
    IPCThreadState::self()->flushCommands();
	//主线程监听EVENT_INPUT，通过回调BinderCallback处理
    sp<BinderCallback> cb(new BinderCallback);
    if (looper->addFd(binder_fd, Looper::POLL_CALLBACK, Looper::EVENT_INPUT, cb,
            nullptr) != 1) {
        ALOGE("Failed to add hwbinder FD to Looper. Aborting...");
        return -1;
    }
	//创建BnHwServiceManager对象
    // Tell IPCThreadState we're the service manager
    sp<BnHwServiceManager> service = new BnHwServiceManager(manager);
    IPCThreadState::self()->setTheContextObject(service);
    // Then tell binder kernel
    ioctl(binder_fd, BINDER_SET_CONTEXT_MGR, 0);
    // Only enable FIFO inheritance for hwbinder
    // FIXME: remove define when in the kernel
#define BINDER_SET_INHERIT_FIFO_PRIO    _IO('b', 10)
 
    int rc = ioctl(binder_fd, BINDER_SET_INHERIT_FIFO_PRIO);
    if (rc) {
        ALOGE("BINDER_SET_INHERIT_FIFO_PRIO failed with error %d\n", rc);
    }
	//通过属性方式告知其他进程，hwservicemanager已经就绪
    rc = property_set("hwservicemanager.ready", "true");
    if (rc) {
        ALOGE("Failed to set \"hwservicemanager.ready\" (error %d). "\
              "HAL services will not start!\n", rc);
    }
	//进入消息循环
    while (true) {
        looper->pollAll(-1 /* timeoutMillis */);
    }
 
    return 0;
}

```
hwservicemanager启动过程比较简单，最重要的就是以下几行：
##### 3.1、创建BnHwServiceManager
``` cpp
sp<BnHwServiceManager> service = new BnHwServiceManager(manager);
IPCThreadState::self()->setTheContextObject(service);
// Then tell binder kernel
ioctl(binder_fd, BINDER_SET_CONTEXT_MGR, 0);

//监听请求
sp<BinderCallback> cb(new BinderCallback);
looper->addFd(binder_fd, Looper::POLL_CALLBACK, Looper::EVENT_INPUT, cb,nullptr)
```
这里创建一个binder本地对象BnHwServiceManager，然后注册到binder驱动中，让其他client进程都可以找到这个binder本地对象，然后为其创建binder代理对象。需要注意的是BnHwServiceManager的成员变量_hidl_mImpl保存的是ServiceManager实例，ServiceManager类实现了IServiceManager接口。
``` cpp
[->\out\soong\.intermediates\system\libhidl\transport\manager\1.1\android.hidl.manager@1.1_genc++\gen\android\hidl\manager\1.1\ServiceManagerAll.cpp]

BnHwServiceManager::BnHwServiceManager(const ::android::sp<IServiceManager> &_hidl_impl)
        : ::android::hidl::base::V1_0::BnHwBase(_hidl_impl, "android.hidl.manager@1.1", "IServiceManager") { 
            _hidl_mImpl = _hidl_impl;
            auto prio = ::android::hardware::details::gServicePrioMap.get(_hidl_impl, {SCHED_NORMAL, 0});
            mSchedPolicy = prio.sched_policy;
            mSchedPriority = prio.prio;
}
```
##### 3.2、IPCThreadState->setTheContextObject(service)

``` cpp
//将BnHwServiceManager设置到IPCThreadState内部对象当中：
sp<BHwBinder>         mContextObject;
void IPCThreadState::setTheContextObject(sp<BHwBinder> obj)
{
    mContextObject = obj;
}
```
##### 3.3、BinderCallback

通过循环监听binder_fd，当有请求时会回调BinderCallback的handleEvent()函数，这部分的知识请参考
[【Android 7.1.2 (Android N) Android消息机制–Handler、Looper、Message 分析】]()

``` cpp
[->system\core\libutils\Looper.cpp]
int Looper::pollInner(int timeoutMillis) {
......
    int callbackResult = response.request.callback->handleEvent(fd, events, data);
......
}
```
处理请求 handlePolledCommands
``` cpp
class BinderCallback : public LooperCallback {
public:
    BinderCallback() {}
    ~BinderCallback() override {}

    int handleEvent(int /* fd */, int /* events */, void* /* data */) override {
        IPCThreadState::self()->handlePolledCommands();
        return 1;  // Continue receiving callbacks.
    }
};
```
进一步通过talkWithDriver() 和 executeCommand(cmd) 处理请求，这一部分跟普通的binder通信就没有区别了，这里就不做分析了。
``` cpp
[->\system\libhwbinder\IPCThreadState.cpp]
status_t IPCThreadState::handlePolledCommands()
{
    status_t result;

    do {
        result = getAndExecuteCommand();
    } while (mIn.dataPosition() < mIn.dataSize());

    processPendingDerefs();
    flushCommands();
    return result;
}
status_t IPCThreadState::getAndExecuteCommand()
{
    status_t result;
    int32_t cmd;

    result = talkWithDriver();
    if (result >= NO_ERROR) {
        size_t IN = mIn.dataAvail();
        if (IN < sizeof(int32_t)) return result;
        cmd = mIn.readInt32();
        .......
        result = executeCommand(cmd);
        ......
    }

    return result;
}
```

##### 3.4、hwservicemanager 继续关系
hwservicemanager进程中的servicemanager作为hidl服务，同样适用了hwbinder框架，其类继承关系图如下：


![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-17-HIDL-binder-bnservicemanager.png)


#### （四）、Android O Treble 之 hwservicemanager 添加服务（add）过程
这里以light模块为例子（其他模块类似）:

> [Android 7.0 init.rc的一点改变](https://blog.csdn.net/ly890700/article/details/62424821)
> 在Android 7之前的版本中，系统Native服务，不管它们的可执行文件位于系统什么位置都定义在根分区的init.*.rc文件中。这造成init＊.rc文件臃肿庞大，给维护带来了一些不便，而且其中定义的一些服务的二进制文件根本不存在。
> 
> 但在Android 7.0中，对该机制做了一些改变 。
> 
> 单一的init＊.rc，被拆分，服务根据其二进制文件的位置（/system，/vendor，/odm）定义到对应分区的etc/init目录中，每个服务一个rc文件。与该服务相关的触发器、操作等也定义在同一rc文件中。
> 
> - /system/etc/init，包含系统核心服务的定义，如SurfaceFlinger、MediaServer、Logcatd等。
> - /vendor/etc/init， SOC厂商针对SOC核心功能定义的一些服务。比如高通、MTK某一款SOC的相关的服务。
> - /odm/etc/init，OEM/ODM厂商如小米、华为、OPP其产品所使用的外设以及差异化功能相关的服务。
> 
> 这样的目录结构拆分，也与Android产品的开发流程相吻合，减轻了维护的负担。下图为Android7.0
> 模拟器/system/etc/init中定义的服务。

``` cpp
[源代码路径]
[->\hardware\interfaces\light\2.0\default\android.hardware.light@2.0-service.rc]
service light-hal-2-0 /vendor/bin/hw/android.hardware.light@2.0-service
    class hal
    user system
    group system
    
[编译生成路径（真机 /vendor/etc/init）]
[->\out\target\product\msm8909go\vendor\etc\init\android.hardware.light@2.0-service.rc]
```
开机会注册android.hardware.light@2.0-service， 直接看main()函数
``` cpp
[->\hardware\interfaces\light\2.0\default\service.cpp]
int main() {
    return defaultPassthroughServiceImplementation<ILight>();
}
```
接着调用defaultPassthroughServiceImplementation<ILight>()模板函数

``` cpp
[->\system\libhidl\transport\include\hidl\LegacySupport.h]
template<class Interface>
__attribute__((warn_unused_result))
status_t registerPassthroughServiceImplementation(
        std::string name = "default") {
    sp<Interface> service = Interface::getService(name, true /* getStub */);
    ......
    status_t status = service->registerAsService(name);
    ......
    return status;
}
template<class Interface>
__attribute__((warn_unused_result))
status_t defaultPassthroughServiceImplementation(std::string name,
                                            size_t maxThreads = 1) {
    configureRpcThreadpool(maxThreads, true);
    status_t result = registerPassthroughServiceImplementation<Interface>(name);
    ......
    joinRpcThreadpool();
    return 0;
}
template<class Interface>
__attribute__((warn_unused_result))
status_t defaultPassthroughServiceImplementation(size_t maxThreads = 1) {
    return defaultPassthroughServiceImplementation<Interface>("default", maxThreads);
}
```

```
[->\out\soong\.intermediates\hardware\interfaces\light\2.0\android.hardware.light@2.0_genc++\gen\android\hardware\light\2.0\LightAll.cpp]
// static
::android::sp<ILight> ILight::getService(const std::string &serviceName, const bool getStub) {
    using ::android::hardware::defaultServiceManager;
    using ::android::hardware::details::waitForHwService;
    using ::android::hardware::getPassthroughServiceManager;
    using ::android::hardware::Return;
    using ::android::sp;
    using Transport = ::android::hidl::manager::V1_0::IServiceManager::Transport;

    sp<ILight> iface = nullptr;

    const sp<::android::hidl::manager::V1_0::IServiceManager> sm = defaultServiceManager();

    Return<Transport> transportRet = sm->getTransport(ILight::descriptor, serviceName);
    
    Transport transport = transportRet;
    const bool vintfHwbinder = (transport == Transport::HWBINDER);
    const bool vintfPassthru = (transport == Transport::PASSTHROUGH);
    ......
    if (getStub || vintfPassthru || vintfLegacy) {
        const sp<::android::hidl::manager::V1_0::IServiceManager> pm = getPassthroughServiceManager();
        if (pm != nullptr) {
            Return<sp<::android::hidl::base::V1_0::IBase>> ret = 
                    pm->get(ILight::descriptor, serviceName);
            if (ret.isOk()) {
                sp<::android::hidl::base::V1_0::IBase> baseInterface = ret;
                if (baseInterface != nullptr) {
                    iface = ILight::castFrom(baseInterface);
                    if (!getStub || trebleTestingOverride) {
                        iface = new BsLight(iface);
                    }
                }
            }
        }
    }
    return iface;
}

```

sp<Interface> service = Interface::getService(name, true /* getStub */)所以getStub=true. 这里通过PassthroughServiceManager来获取ILight对象。其实所有的Hal 进程都是通过PassthroughServiceManager来得到hidl服务对象的，而作为Hal进程的Client端Framework进程在获取hidl服务对象时，需要通过hal的Transport类型来选择获取方式。

``` cpp
[\device\qcom\xxx\manifest.xml]
    <hal format="hidl">
        <name>android.hardware.light</name>
        <transport>hwbinder</transport>
        <version>2.0</version>
        <interface>
            <name>ILight</name>
            <instance>default</instance>
        </interface>
    </hal>
```

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-18-HIDL-treble_cpp_legacy_hal_progression.png)


``` cpp
[->system\libhidl\transport\ServiceManagement.cpp]

sp<IServiceManager1_0> getPassthroughServiceManager() {
    return getPassthroughServiceManager1_1();
}

sp<IServiceManager1_1> getPassthroughServiceManager1_1() {
    static sp<PassthroughServiceManager> manager(new PassthroughServiceManager());
    return manager;
}

struct PassthroughServiceManager : IServiceManager1_1 {
    static void openLibs(const std::string& fqName,
            std::function<bool /* continue */(void* /* handle */,
                const std::string& /* lib */, const std::string& /* sym */)> eachLib) {
        //fqName looks like android.hardware.foo@1.0::IFoo
        size_t idx = fqName.find("::");

        std::string packageAndVersion = fqName.substr(0, idx);
        std::string ifaceName = fqName.substr(idx + strlen("::"));

        const std::string prefix = packageAndVersion + "-impl";
        const std::string sym = "HIDL_FETCH_" + ifaceName;

        const int dlMode = RTLD_LAZY;
        void *handle = nullptr;

        std::vector<std::string> paths = {HAL_LIBRARY_PATH_ODM, HAL_LIBRARY_PATH_VENDOR,
                                          HAL_LIBRARY_PATH_VNDK_SP, HAL_LIBRARY_PATH_SYSTEM};
                                          
        for (const std::string& path : paths) {
            std::vector<std::string> libs = search(path, prefix, ".so");

            for (const std::string &lib : libs) {
                const std::string fullPath = path + lib;

                if (path != HAL_LIBRARY_PATH_SYSTEM) {
                    handle = android_load_sphal_library(fullPath.c_str(), dlMode);
                } else {
                    handle = dlopen(fullPath.c_str(), dlMode);
                }
                ......
                if (!eachLib(handle, lib, sym)) {
                    return;
                }
            }
        }
    }
```
这里只是简单的创建了一个PassthroughServiceManager对象。PassthroughServiceManager也实现了IServiceManager接口。然后通过PassthroughServiceManager询服务：

根据传入的fqName获取当前的接口名ILight，拼接出后面需要查找的函数名HIDL_FETCH_ILight和库名字android.hardware.light@2.0-impl.so,然后查找"/system/lib64/hw/"、"/vendor/lib64/hw/"、"/odm/lib64/hw/"下是否有对应的so库。接着通过dlopen载入/vendor/lib/hw/android.hardware.light@2.0-impl.so，然后通过dlsym查找并调用HIDL_FETCH_ILight函数。

``` cpp

[->\hardware\interfaces\light\2.0\default\Light.cpp]
ILight* HIDL_FETCH_ILight(const char* /* name */) {
    std::map<Type, light_device_t*> lights;

    for(auto const &pair : kLogicalLights) {
        Type type = pair.first;
        const char* name = pair.second;

        light_device_t* light = getLightDevice(name);

        if (light != nullptr) {
            lights[type] = light;
        }
    }

    if (lights.size() == 0) {
        // Log information, but still return new Light.
        // Some devices may not have any lights.
        ALOGI("Could not open any lights.");
    }

    return new Light(std::move(lights));
}
```

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-19-HIDL-binder-ILight-so-dlopen.png)




###### 4.1、ILight::registerAsService()
首先会调用registerAsService()注册服务，然后joinRpcThreadpool()加入线程池
``` cpp
[->\out\soong\.intermediates\hardware\interfaces\light\2.0\android.hardware.light@2.0_genc++\gen\android\hardware\light\2.0\LightAll.cpp]

::android::status_t ILight::registerAsService(const std::string &serviceName) {
    ::android::hardware::details::onRegistration("android.hardware.light@2.0", "ILight", serviceName);

    const ::android::sp<::android::hidl::manager::V1_0::IServiceManager> sm
            = ::android::hardware::defaultServiceManager();
    if (sm == nullptr) {
        return ::android::INVALID_OPERATION;
    }
    ::android::hardware::Return<bool> ret = sm->add(serviceName.c_str(), this);
    return ret.isOk() && ret ? ::android::OK : ::android::UNKNOWN_ERROR;
}

```
通过defaultServiceManager()获取IServiceManager，接着add()添加服务

首先根据属性"hwservicemanager.ready"值判断hwservicemanager进程是否启动就绪，如果hwservicemanager已经启动，那么通过fromBinder<IServiceManager, BpHwServiceManager, BnHwServiceManager>( ProcessState::self()->getContextObject(NULL))来获取hwservicemanager的代理。
``` cpp
[->\system\libhidl\transport\ServiceManagement.cpp]
sp<IServiceManager1_0> defaultServiceManager() {
    return defaultServiceManager1_1();
}
sp<IServiceManager1_1> defaultServiceManager1_1() {
    {
        AutoMutex _l(details::gDefaultServiceManagerLock);
        if (details::gDefaultServiceManager != NULL) {
            return details::gDefaultServiceManager;
        }

        if (access("/dev/hwbinder", F_OK|R_OK|W_OK) != 0) {
            return nullptr;
        }

        waitForHwServiceManager();

        while (details::gDefaultServiceManager == NULL) {
            details::gDefaultServiceManager =
                    fromBinder<IServiceManager1_1, BpHwServiceManager, BnHwServiceManager>(
                        ProcessState::self()->getContextObject(NULL));
            ......
        }
    }

    return details::gDefaultServiceManager;
}

```

首先看看ProcessState::getContextObject()

因此通过ProcessState::self()->getContextObject(NULL)将得到一个BpHwBinder对象，然后通过fromBinder<IServiceManager, BpHwServiceManager, BnHwServiceManager>进行转换。
``` cpp
[->\system\libhwbinder\ProcessState.cpp]
sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/)
{
    return getStrongProxyForHandle(0);
}
sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{
    sp<IBinder> result;

    AutoMutex _l(mLock);

    handle_entry* e = lookupHandleLocked(handle);

    if (e != NULL) {
        IBinder* b = e->binder;
        if (b == NULL || !e->refs->attemptIncWeak(this)) {
            b = new BpHwBinder(handle);
            e->binder = b;
            if (b) e->refs = b->getWeakRefs();
            result = b;
        } else {
            result.force_set(b);
            e->refs->decWeak(this);
        }
    }

    return result;
}
```

第一次调用会调用fromBinder()函数，在这里将创建一个BpHwServiceManager对象，ProcessState::self()->getContextObject(NULL)如果返回的是远程binder对象，那么基于BpHwBinder创建BpHwServiceManager对象，BpHwBinder负责数据传输，而BpHwServiceManager服务数据业务，业务数据在BpHwServiceManager层打包好后，转交给BpHwBinder发送。如果getContextObject(NULL)返回的是本地binder对象，那么将这个本地binder对象强制转换为BnHwBase类型，从上图可知BnHwBase继承BHwBinder类，BHwBinder即是本地binder对象。
static_cast<BnHwBase*>(binderIface.get()) 然后通过BnHwBase的getImpl()函数得到其业务实现对象IBase。
sp<IBase> base =static_cast<BnHwBase*>(binderIface.get())->getImpl();

然后检查业务接口是否相同：
if (details::canCastInterface(base.get(),IType::descriptor))
如果业务接口类型相同，那么再次将这个本地binder对象转换为孙类BnHwServiceManager类型：
BnHwServiceManager* stub = static_cast<BnHwServiceManager*>(binderIface.get());
然后返回业务实现类对象ServiceManager对象。

``` cpp
[->\system\libhidl\transport\include\hidl\HidlBinderSupport.h]
template <typename IType, typename ProxyType, typename StubType>
sp<IType> fromBinder(const sp<IBinder>& binderIface) {
    using ::android::hidl::base::V1_0::IBase;
    using ::android::hidl::base::V1_0::BnHwBase;

    if (binderIface.get() == nullptr) {
        return nullptr;
    }
    if (binderIface->localBinder() == nullptr) {
        return new ProxyType(binderIface);
    }
    sp<IBase> base = static_cast<BnHwBase*>(binderIface.get())->getImpl();
    if (details::canCastInterface(base.get(), IType::descriptor)) {
        StubType* stub = static_cast<StubType*>(binderIface.get());
        return stub->getImpl();
    } else {
        return nullptr;
    }
}
```
调用 getImpl()函数实现，返回的是
``` cpp
[->\out\soong\.intermediates\system\libhidl\transport\manager\1.0\android.hidl.manager@1.0_genc++_headers\gen\android\hidl\manager\1.0\BnHwServiceManager.h]

    ::android::sp<IServiceManager> _hidl_mImpl;
    
    ::android::sp<IServiceManager> getImpl() { return _hidl_mImpl; };
```


###### 4.2、sm->add(serviceName.c_str(), this) 添加light service

``` cpp
[->\out\soong\.intermediates\system\libhidl\transport\manager\1.0\android.hidl.manager@1.0_genc++\gen\android\hidl\manager\1.0\ServiceManagerAll.cpp]

::android::hardware::Return<bool> BpHwServiceManager::add(const ::android::hardware::hidl_string& name, const ::android::sp<::android::hidl::base::V1_0::IBase>& service){
    ::android::hardware::Return<bool>  _hidl_out = ::android::hidl::manager::V1_0::BpHwServiceManager::_hidl_add(this, this, name, service);

    return _hidl_out;
}

::android::hardware::Return<bool> BpHwServiceManager::_hidl_add(::android::hardware::IInterface *_hidl_this, ::android::hardware::details::HidlInstrumentor *_hidl_this_instrumentor, const ::android::hardware::hidl_string& name, const ::android::sp<::android::hidl::base::V1_0::IBase>& service) {
    //......

    ::android::hardware::Parcel _hidl_data;
    ::android::hardware::Parcel _hidl_reply;
    ::android::status_t _hidl_err;
    ::android::hardware::Status _hidl_status;

    //......
    ::android::hardware::ProcessState::self()->startThreadPool();
    _hidl_err = ::android::hardware::IInterface::asBinder(_hidl_this)->transact(2 /* add */, _hidl_data, &_hidl_reply);
   .......
}
```

``` cpp
[->\system\libhwbinder\BpHwBinder.cpp]
status_t BpHwBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags, TransactCallback /*callback*/)
{
    // Once a binder has died, it will never come back to life.
    if (mAlive) {
        status_t status = IPCThreadState::self()->transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;
        return status;
    }

    return DEAD_OBJECT;
}

```

``` cpp
[->/system/libhwbinder/IPCThreadState.cpp]
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    status_t err = data.errorCheck();

    flags |= TF_ACCEPT_FDS;
    ......
    if (err == NO_ERROR) {
        err = writeTransactionData(BC_TRANSACTION_SG, flags, handle, code, data, NULL);
    }

    if ((flags & TF_ONE_WAY) == 0) {
        ......
        if (reply) {
            err = waitForResponse(reply);
        } else {
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }
    } else {
        err = waitForResponse(NULL, NULL);
    }
    ......
    return err;
}


```

waitForResponse()会通过底层binder 驱动进入到服务端，服务端监听消息来到处理过后会调用onTransact()函数

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-20-HIDL-ioctl-kernel.png.png)

``` cpp
[->/system/libhwbinder/Binder.cpp]
status_t BHwBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags, TransactCallback callback)
{
    data.setDataPosition(0);

    status_t err = NO_ERROR;
    switch (code) {
        default:
            err = onTransact(code, data, reply, flags,
                    [&](auto &replyParcel) {
                        replyParcel.setDataPosition(0);
                        if (callback != NULL) {
                            callback(replyParcel);
                        }
                    });
            break;
    }

    return err;
}


```


``` cpp
[->\out\soong\.intermediates\system\libhidl\transport\manager\1.0\android.hidl.manager@1.0_genc++\gen\android\hidl\manager\1.0\ServiceManagerAll.cpp]
::android::status_t BnHwServiceManager::onTransact(
        uint32_t _hidl_code,
        const ::android::hardware::Parcel &_hidl_data,
        ::android::hardware::Parcel *_hidl_reply,
        uint32_t _hidl_flags,
        TransactCallback _hidl_cb) {
    ::android::status_t _hidl_err = ::android::OK;

    switch (_hidl_code) {
        ......
        case 2 /* add */:
        {
            bool _hidl_is_oneway = _hidl_flags & ::android::hardware::IBinder::FLAG_ONEWAY;
            if (_hidl_is_oneway != false) {
                return ::android::UNKNOWN_ERROR;
            }

            _hidl_err = ::android::hidl::manager::V1_0::BnHwServiceManager::_hidl_add(this, _hidl_data, _hidl_reply, _hidl_cb);
            break;
        }
        ......
}


::android::status_t BnHwServiceManager::_hidl_add(
        ::android::hidl::base::V1_0::BnHwBase* _hidl_this,
        const ::android::hardware::Parcel &_hidl_data,
        ::android::hardware::Parcel *_hidl_reply,
        TransactCallback _hidl_cb) {
    ......

    const ::android::hardware::hidl_string* name;
    ::android::sp<::android::hidl::base::V1_0::IBase> service;

    size_t _hidl_name_parent;

    _hidl_err = _hidl_data.readBuffer(sizeof(*name), &_hidl_name_parent,  reinterpret_cast<const void **>(&name));

    if (_hidl_err != ::android::OK) { return _hidl_err; }

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

    atrace_begin(ATRACE_TAG_HAL, "HIDL::IServiceManager::add::server");
    ......

    bool _hidl_out_success = static_cast<BnHwServiceManager*>(_hidl_this)->_hidl_mImpl->add(*name, service);

    ......
    return _hidl_err;
}

```
此处的_hidl_mImpl即之前得到的::android::sp<IServiceManager> ==_hidl_mImpl== 对象，所以会调用

``` cpp
[->\system\hwservicemanager\ServiceManager.cpp]
Return<bool> ServiceManager::add(const hidl_string& name, const sp<IBase>& service) {
    ......
    pid_t pid = IPCThreadState::self()->getCallingPid();
    auto context = mAcl.getContext(pid);

    auto ret = service->interfaceChain([&](const auto &interfaceChain) {
    
        ......
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
    ......
    return isValidService;
}

```
完成插入到PackageInterfaceMap &ifaceMap 对象中，并添加一些监听。之后就可以查询到light service了。

###### 4.3、joinRpcThreadpool()

加入线程池，至此Light service 就启动起来了，可以等待Client的请求了
``` cpp
[->\system\libhidl\transport\HidlTransportSupport.cpp]
void joinRpcThreadpool() {
    // TODO(b/32756130) this should be transport-dependent
    joinBinderRpcThreadpool();
}

[->\system\libhidl\transport\HidlBinderSupport.cpp]
void joinBinderRpcThreadpool() {
    IPCThreadState::self()->joinThreadPool();
}

[->\system\libhwbinder\IPCThreadState.cpp]
void IPCThreadState::joinThreadPool(bool isMain)
{
    ......
    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);

    status_t result;
    do {
        processPendingDerefs();
        // now get the next command to be processed, waiting if necessary
        result = getAndExecuteCommand();

        ......
    } while (result != -ECONNREFUSED && result != -EBADF);
    
    mOut.writeInt32(BC_EXIT_LOOPER);
    talkWithDriver(false);
}
```
到此就完成了hidl服务注册。


![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-21-HIDL-hwbinder-ILight-add.png)


#### （五）、Android O Treble 之 hwservicemanager 查询服务（get）过程
这里以light模块为例子（其他模块类似）：
通过前面的分析我们知道，Hal进程启动时，会向hwservicemanager进程注册hidl服务，那么当Framework Server需要通过hal访问硬件设备时，首先需要查询对应的hidl服务，那么Client进程是如何查询hidl服务的呢？这篇文章将展开分析，这里再次以ILight为例进行展开。


![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-22-android-hwservicemanager-get.png)



``` java 
[->/frameworks/base/services/core/java/com/android/server/lights/LightsService.java]
static native void setLight_native(int light, int color, int mode,
            int onMS, int offMS, int brightnessMode);
```

``` cpp
[->/frameworks/base/services/core/jni/com_android_server_lights_LightsService.cpp]
static void setLight_native(
        JNIEnv* /* env */,
        jobject /* clazz */,
        jint light,
        jint colorARGB,
        jint flashMode,
        jint onMS,
        jint offMS,
        jint brightnessMode) {
    ......
    sp<ILight> hal = LightHal::associate();
    .......
    Type type = static_cast<Type>(light);
    LightState state = constructState(
        colorARGB, flashMode, onMS, offMS, brightnessMode);

    {
        android::base::Timer t;
        Return<Status> ret = hal->setLight(type, state);
        processReturn(ret, type, state);
        if (t.duration() > 50ms) ALOGD("Excessive delay setting light");
    }
}

```
associate() 会获取ILight::getService();这里通过ILight::getService()函数来查询ILight这个HIDL服务，由于这里没有传递任何参数，因此函数最终会调用：

``` cpp
[->\out\soong\.intermediates\hardware\interfaces\light\2.0\android.hardware.light@2.0_genc++_headers\gen\android\hardware\light\2.0\ILight.h]
    static ::android::sp<ILight> getService(const std::string &serviceName="default", bool getStub=false);
```
注意，这里的getStub为false，说明加载hidl服务方式是由当前hidl服务的transport类型决定。

``` cpp
[\device\qcom\xxx\manifest.xml]
    <hal format="hidl">
        <name>android.hardware.light</name>
        <transport>hwbinder</transport>
        <version>2.0</version>
        <interface>
            <name>ILight</name>
            <instance>default</instance>
        </interface>
    </hal>
```
由于ILight的transport是hwbinder类型，那么将从hwservicemanager中查询hidl服务。

``` cpp
[\out\soong\.intermediates\hardware\interfaces\light\2.0\android.hardware.light@2.0_genc++\gen\android\hardware\light\2.0\LightAll.cpp]

// static
::android::sp<ILight> ILight::getService(const std::string &serviceName, const bool getStub) {
    using ::android::hardware::defaultServiceManager;
    using ::android::hardware::details::waitForHwService;
    using ::android::hardware::getPassthroughServiceManager;
    using ::android::hardware::Return;
    using ::android::sp;
    using Transport = ::android::hidl::manager::V1_0::IServiceManager::Transport;

    sp<ILight> iface = nullptr;

    const sp<::android::hidl::manager::V1_0::IServiceManager> sm = defaultServiceManager();

    Return<Transport> transportRet = sm->getTransport(ILight::descriptor, serviceName);
    
    Transport transport = transportRet;
    const bool vintfHwbinder = (transport == Transport::HWBINDER);
    const bool vintfPassthru = (transport == Transport::PASSTHROUGH);

    for (int tries = 0; !getStub && (vintfHwbinder || (vintfLegacy && tries == 0)); tries++) {
    
        if (vintfHwbinder && tries > 0) {
            waitForHwService(ILight::descriptor, serviceName);
        }
        Return<sp<::android::hidl::base::V1_0::IBase>> ret = 
                sm->get(ILight::descriptor, serviceName);

        sp<::android::hidl::base::V1_0::IBase> base = ret;

        Return<sp<ILight>> castRet = ILight::castFrom(base, true /* emitError */);

        iface = castRet;

        return iface;
    }
    if (getStub || vintfPassthru || vintfLegacy) {
        const sp<::android::hidl::manager::V1_0::IServiceManager> pm = getPassthroughServiceManager();
        if (pm != nullptr) {
            Return<sp<::android::hidl::base::V1_0::IBase>> ret = 
                    pm->get(ILight::descriptor, serviceName);
            if (ret.isOk()) {
                sp<::android::hidl::base::V1_0::IBase> baseInterface = ret;
                if (baseInterface != nullptr) {
                    iface = ILight::castFrom(baseInterface);
                    if (!getStub || trebleTestingOverride) {
                        iface = new BsLight(iface);
                    }
                }
            }
        }
    }
    return iface;
}

```

这里通过sm->get(ILight::descriptor, serviceName)查询ILight这个hidl服务，得到IBase对象后，在通过ILight::castFrom转换为ILight对象。

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-23-HIDL-castFrom.png)


##### 5.1、服务查询_hidl_get()

``` cpp
[->\out\soong\.intermediates\system\libhidl\transport\manager\1.1\android.hidl.manager@1.0_genc++\gen\android\hidl\manager\1.1\ServiceManagerAll.cpp]
::android::hardware::Return<::android::sp<::android::hidl::base::V1_0::IBase>> BpHwServiceManager::get(const ::android::hardware::hidl_string& fqName, const ::android::hardware::hidl_string& name){
    ::android::hardware::Return<::android::sp<::android::hidl::base::V1_0::IBase>>  _hidl_out = ::android::hidl::manager::V1_0::BpHwServiceManager::_hidl_get(this, this, fqName, name);

    return _hidl_out;
}
```

``` cpp
[->\out\soong\.intermediates\system\libhidl\transport\manager\1.0\android.hidl.manager@1.0_genc++\gen\android\hidl\manager\1.0\ServiceManagerAll.cpp]

::android::hardware::Return<::android::sp<::android::hidl::base::V1_0::IBase>> BpHwServiceManager::_hidl_get(::android::hardware::IInterface *_hidl_this, ::android::hardware::details::HidlInstrumentor *_hidl_this_instrumentor, const ::android::hardware::hidl_string& fqName, const ::android::hardware::hidl_string& name) {
    
    ::android::hardware::Parcel _hidl_data;
    ::android::hardware::Parcel _hidl_reply;
    ::android::status_t _hidl_err;
    ::android::hardware::Status _hidl_status;

    ::android::sp<::android::hidl::base::V1_0::IBase> _hidl_out_service;

    _hidl_err = _hidl_data.writeInterfaceToken(BpHwServiceManager::descriptor);

    size_t _hidl_fqName_parent;

    _hidl_err = _hidl_data.writeBuffer(&fqName, sizeof(fqName), &_hidl_fqName_parent);

    _hidl_err = ::android::hardware::writeEmbeddedToParcel(
            fqName,
            &_hidl_data,
            _hidl_fqName_parent,
            0 /* parentOffset */);


    size_t _hidl_name_parent;

    _hidl_err = _hidl_data.writeBuffer(&name, sizeof(name), &_hidl_name_parent);

    _hidl_err = ::android::hardware::writeEmbeddedToParcel(
            name,
            &_hidl_data,
            _hidl_name_parent,
            0 /* parentOffset */);


    _hidl_err = ::android::hardware::IInterface::asBinder(_hidl_this)->transact(1 /* get */, _hidl_data, &_hidl_reply);

    _hidl_err = ::android::hardware::readFromParcel(&_hidl_status, _hidl_reply);


    {
        ::android::sp<::android::hardware::IBinder> _hidl__hidl_out_service_binder;
        _hidl_err = _hidl_reply.readNullableStrongBinder(&_hidl__hidl_out_service_binder);

        _hidl_out_service = ::android::hardware::fromBinder<::android::hidl::base::V1_0::IBase,::android::hidl::base::V1_0::BpHwBase,::android::hidl::base::V1_0::BnHwBase>(_hidl__hidl_out_service_binder);
    }

    _hidl_status.setFromStatusT(_hidl_err);
    return ::android::hardware::Return<::android::sp<::android::hidl::base::V1_0::IBase>>(_hidl_out_service);
}

```

整个调用过程和hidl 添加服务过程完全一致，就是一个从BpHwServiceManager --> BnHwServiceManager --> ServiceManager的过程。但需要注意，BpHwServiceManager得到BnHwServiceManager返回过来的binder代理后，会通过fromBinder函数进行对象转换：


``` cpp
::android::hardware::fromBinder<::android::hidl::base::V1_0::IBase,::android::hidl::base::V1_0::BpHwBase,::android::hidl::base::V1_0::BnHwBase>(_hidl__hidl_out_service_binder)
```

hwservicemanager将ILight的binder代理BpHwBinder发给Framework Server进程，Framework Server进程拿到的依然是ILight的binder代理BpHwBinder对象，因此在fromBinder函数中将创建BpHwBase对象来封装BpHwBinder。

``` cpp
[->\out\soong\.intermediates\system\libhidl\transport\manager\1.0\android.hidl.manager@1.0_genc++\gen\android\hidl\manager\1.0\ServiceManagerAll.cpp]

::android::status_t BnHwServiceManager::_hidl_get(
        ::android::hidl::base::V1_0::BnHwBase* _hidl_this,
        const ::android::hardware::Parcel &_hidl_data,
        ::android::hardware::Parcel *_hidl_reply,
        TransactCallback _hidl_cb) {


    const ::android::hardware::hidl_string* fqName;
    const ::android::hardware::hidl_string* name;

    size_t _hidl_fqName_parent;

    _hidl_err = _hidl_data.readBuffer(sizeof(*fqName), &_hidl_fqName_parent,  reinterpret_cast<const void **>(&fqName));


    _hidl_err = ::android::hardware::readEmbeddedFromParcel(
            const_cast<::android::hardware::hidl_string &>(*fqName),
            _hidl_data,
            _hidl_fqName_parent,
            0 /* parentOffset */);

    size_t _hidl_name_parent;

    _hidl_err = _hidl_data.readBuffer(sizeof(*name), &_hidl_name_parent,  reinterpret_cast<const void **>(&name));

    _hidl_err = ::android::hardware::readEmbeddedFromParcel(
            const_cast<::android::hardware::hidl_string &>(*name),
            _hidl_data,
            _hidl_name_parent,
            0 /* parentOffset */);

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

    _hidl_cb(*_hidl_reply);
    return _hidl_err;
}

```
BnHwServiceManager通过ServiceManager对象查询到对应的hidl服务，返回IBase对象后，会调用toBinder函数转换为IBinder类型对象：

``` cpp
::android::hardware::toBinder< ::android::hidl::base::V1_0::IBase>(_hidl_out_service)
```

由于在hwservicemanager这边，保存的是ILight的BpHwBase对象，因此在toBinder函数中将调用IInterface::asBinder来得到BpHwBase的成员变量中的BpHwBinder对象。

``` cpp
if (ifacePtr->isRemote()) {
	return ::android::hardware::IInterface::asBinder(static_cast<ProxyType *>(ifacePtr));
}
```

服务查询过程其实就是根据接口包名及服务名称，从hwservicemanager管理的表中查询对应的IBase服务对象，然后在Client进程空间分别创建BpHwBinder和BpHwBase对象。

##### 5.2、接口转换IXXX::castFrom()
Framework Server进程通过上述hidl服务查询，得到了BpHwBase对象后，需要将其转换为与业务相关的代理对象，这就是通过：Return<sp<ILight>> castRet = ILight::castFrom(base, true /* emitError */)

``` cpp
[->\out\soong\.intermediates\hardware\interfaces\light\2.0\android.hardware.light@2.0_genc++\gen\android\hardware\light\2.0\LightAll.cpp]

::android::hardware::Return<::android::sp<ILight>> ILight::castFrom(const ::android::sp<::android::hidl::base::V1_0::IBase>& parent, bool emitError) {
    return ::android::hardware::details::castInterface<ILight, ::android::hidl::base::V1_0::IBase, BpHwLight>(
            parent, "android.hardware.light@2.0::ILight", emitError);
}
```

system\libhidl\transport\include\hidl\HidlTransportSupport.h

``` cpp
// cast the interface IParent to IChild.
// Return nonnull if cast successful.
// Return nullptr if:
// 1. parent is null
// 2. cast failed because IChild is not a child type of IParent.
// 3. !emitError, calling into parent fails.
// Return an error Return object if:
// 1. emitError, calling into parent fails.
template<typename IChild, typename IParent, typename BpChild, typename BpParent>
Return<sp<IChild>> castInterface(sp<IParent> parent, const char *childIndicator, bool emitError) {
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
        return sp<IChild>(new BpChild(toBinder<IParent, BpParent>(parent)));
    }
    // Passthrough mode. Got BnChild and BsChild.
    return sp<IChild>(static_cast<IChild *>(parent.get()));
}
 
}  // namespace details
```
这个模板函数展开后如下：

``` cpp
Return<sp<ILight>> castInterface(sp<IBase> parent, const char *childIndicator, bool emitError) {
    if (parent.get() == nullptr) {
        // casts always succeed with nullptrs.
        return nullptr;
    }
    Return<bool> canCastRet = details::canCastInterface(parent.get(), childIndicator, emitError);
    if (!canCastRet.isOk()) {
        // call fails, propagate the error if emitError
        return emitError
                ? details::StatusOf<bool, sp<ILight>>(canCastRet)
                : Return<sp<ILight>>(sp<ILight>(nullptr));
    }
 
    if (!canCastRet) {
        return sp<ILight>(nullptr); // cast failed.
    }
    // TODO b/32001926 Needs to be fixed for socket mode.
    if (parent->isRemote()) {
        // binderized mode. Got BpChild. grab the remote and wrap it.
        return sp<ILight>(new BpHwLight(toBinder<IBase, BpParent>(parent)));
    }
    // Passthrough mode. Got BnChild and BsChild.
    return sp<ILight>(static_cast<ILight*>(parent.get()));
}
```

因此最终会创建一个BpHwLight对象。new BpHwLight(toBinder<IBase, BpParent>(parent))


![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-24-HIDL-binder-bphwxxxx.png)



#### （六）、Android O Treble 之 HIDL服务Java框架实现

前面介绍了HIDL服务在native层的实现过程，包括HIDL服务加载创建、服务注册、服务查询过程等，那么Java层是否也实现了相关的服务框架呢？ 通常情况下，所有的Hal都实现在native层面，每个hal进程都是一个native进程，由init进程启动，在hal进程启动时会完成HIDL服务注册，Framework Server进程不一定完全是native进程，比如system_server进程，它运行在虚拟机环境中，由zygote进程fork而来，这时，Java层也需要请求HIDL服务，因此Android不仅在native层HIDL化了hal，在Java层同样也定义了相关的服务框架。

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-25-Java-binder.png)


上图是Java层binder和hwbinder之间的类基础图对比。当我们定义一个.hal接口文件时，通过hidl-gen编译为Java文件后，将按上图中的类继承关系自动生成代码。

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-26-Java-binder-class-inherit.png)


如上图所示，当我们定义IXXX.hal文件后，通过编译将在out/target/common/gen/JAVA_LIBRARIES目录下生成对应的IXXX.java，该文件按上述类继承关系自动生成相关代码，我们只需要定义一个XXXImp类，继承Stub并实现所有方法，然后在某个服务进程中创建一个XXXImp对象，并调用registerService（）函数进行hidl服务注册，如下所示：

``` cpp
XXXImp mXXXImp = new XXXImp();
mXXXImp.registerAsService("XXXImp");
```

这样就完成了一个Java层的hidl服务注册，当然在当前Android系统中，大部分还是native层的hidl服务，Java层的hidl服务还是比较少的。从上述可知，Java层的hidl服务包括2个步骤：

1. hidl服务对象创建；

2.hidl服务注册；

##### 6.1、Java hidl服务创建过程
从上面的类继承图可知，hidl服务实现类继承于Stub，Stub又继承于HwBinder，因此创建一个XXXImp对象时，会调用HwBinder的构造函数。

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-27-HwBinder.java.setup.png)

frameworks\base\core\java\android\os\HwBinder.java

``` java
public HwBinder() {
	native_setup();
 
	sNativeRegistry.registerNativeAllocation(
			this,
			mNativeContext);
}
static {
	long freeFunction = native_init();
 
	sNativeRegistry = new NativeAllocationRegistry(
			HwBinder.class.getClassLoader(),
			freeFunction,
			128 /* size */);
}
```

创建HwBinder对象会首先执行native_init()函数，然后调用native_setup()函数。
frameworks\base\core\jni\android_os_HwBinder.cpp

``` cpp
static jlong JHwBinder_native_init(JNIEnv *env) {
    JHwBinder::InitClass(env);
 
    return reinterpret_cast<jlong>(&releaseNativeContext);
}
 
static void JHwBinder_native_setup(JNIEnv *env, jobject thiz) {
    sp<JHwBinderHolder> context = new JHwBinderHolder;
    JHwBinder::SetNativeContext(env, thiz, context);
}
```

这里创建一个JHwBinderHolder 对象，并保存在HwBinder类的mNativeContext变量中。

``` cpp
sp<JHwBinderHolder> JHwBinder::SetNativeContext(
        JNIEnv *env, jobject thiz, const sp<JHwBinderHolder> &context) {
    sp<JHwBinderHolder> old =
        (JHwBinderHolder *)env->GetLongField(thiz, gFields.contextID);
 
    if (context != NULL) {
        context->incStrong(NULL /* id */);
    }
 
    if (old != NULL) {
        old->decStrong(NULL /* id */);
    }
 
    env->SetLongField(thiz, gFields.contextID, (long)context.get());
 
    return old;
}
```

这里出现了多个binder类型：HwBinder、JHwBinderHolder、JHwBinder他们的类继承图如下：

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-28-Java-binder---.png)


红线标识了这3个类对象之间的关系，为了更加清晰地描述他们之间的关联关系，如下图所示：

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-29-HIDL-java-object.png)


##### 6.2、Java层服务注册（add）查询（get）过程
服务注册查询过程本质也是通过Native层的hwservicemanager来进行的。
（略）请参考大牛博客：[Android O Treble架构下HIDL服务Java框架实现](https://blog.csdn.net/yangwen123/article/details/79876534)
到此Treble架构下的hwBinder实现过程就基本介绍完成。
总体架构：

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/o.hidl.system/Treble-09-HIDL-binder-binder_ipc_process.jpg)


#### （七）、参考资料(特别感谢各位前辈的分析和图示)：
[Android O Treble架构（系列分析文章） -  CSDN博客](https://blog.csdn.net/yangwen123)
[Android 7.1.2 (Android N) Android Binder 系统分析]()





