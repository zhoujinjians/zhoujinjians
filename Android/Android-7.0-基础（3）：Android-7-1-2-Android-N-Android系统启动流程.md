---
title: Android N 基础（3）：Android 7.1.2 Android 系统启动流程分析
cover: https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/hexo.themes/bing-wallpaper-2018.04.02.jpg
categories: 
  - Android
tags:
  - Android
toc: true
abbrlink: 20171008
date: 2017-10-08 09:25:00
---


## 源码：

**system/core/rootdir/**

- init.rc
- init.zygote64.rc

**system/core/init/**

- init.cpp
- init_parser.cpp
- signal_handler.cpp

**frameworks/base/cmds/app_process/**

- App_main.cpp

**frameworks/base/core/jni/**

- AndroidRuntime.cpp

**frameworks/base/core/java/com/android/internal/os/**

- ZygoteInit.java
- Zygote.java
- ZygoteConnection.java

**frameworks/base/core/java/com/android/internal/os/**

- ZygoteInit.java
- RuntimeInit.java
- Zygote.java

**frameworks/base/core/services/java/com/android/server/**

- SystemServer.java

**frameworks/base/core/jni/**

- com_android_internal_os_Zygote.cpp
- AndroidRuntime.cpp

**frameworks/base/services/java/com/android/server/**

- SystemServer.java

**frameworks/base/services/core/java/com/android/server/**

- SystemServiceManager.java
- ServiceThread.java
- am/ActivityManagerService.java

**frameworks/base/core/java/android/app/**

- ActivityThread.java
- LoadedApk.java
- ContextImpl.java

**frameworks/base/core/java/android/app/**

- ActivityThread.java
- LoadedApk.java
- ContextImpl.java

**frameworks/base/services/java/com/android/server/**

- SystemServer.java

**frameworks/base/services/core/java/com/android/server/**

- SystemServiceManager.java
- ServiceThread.java
- pm/Installer.java
- am/ActivityManagerService.java

### 一、Android概述

Android系统非常庞大，底层是采用Linux作为基底，上层采用带有虚拟机的Java层，通过通过JNI技术，将上下打通，融为一体。下图是Google提供的一张经典的4层架构图，从下往上，依次分为Linux内核，系统库和Android Runtime，应用框架层，应用程序层这4层架构，每一层都包含大量的子模块或子系统。 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/android.boot/N01-android-system-start-arch.png)

### 二、系统启动

Google提供的4层架构图，是非常经典，但只是如垒砖般的方式，简单地分层，而不足表达Android整个系统的启动过程，环环相扣的连接关系，本文更多的是以进程的视角，以分层的架构来诠释Android系统的全貌。 系统启动架构图 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/android.boot/N02-android-system-start-arch.png)
![Markdown](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/android.boot/N03-android-system-start-arch.png)
![Markdown](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/android.boot/N04-android-system-start-arch.png)

### 三、设备启动过程

#### 3.1、Bootloader引导

Boot ROM: 当手机处于关机状态时，长按Power键开机，引导芯片开始从固化在ROM里的预设出代码开始执行，然后加载引导程序到RAM；

Boot Loader：这是启动Android系统之前的引导程序，主要是检查RAM，初始化硬件设备（如CPU、内存、Flash等）并且通过建立内存空间映射，为装载Linux内核准备合适的环境。一旦Linux内核装载完毕，Bootloader将会从内存中清除掉。 如果用户在Bootloader运行期间，按下预定义的组合健，可以进入系统的更新模块。Android的下载更新可以选择进入Fastboot模式或者Recovery模式。

Fastboot是Android设计的一套通过USB来更新手机分区映像的协议，方便开发人员能快速更新指定的手机分区。但是一般的零售机上往往去掉了Fastboot，Google销售的开发机则带有Fastboot模块。 Recovery模式是Android特有的升级系统。利用Recovery模式，手机可以进行恢复出厂设置或进行OTA、补丁和固件升级。进入Recovery模式实际上是启动了一个文本模式的Linux。

#### 3.2、装载和启动Linux内核

到这里才刚刚开始进入Android系统.

启动Kernel的0号进程：初始化进程管理、内存管理，加载Display,Camera Driver，Binder Driver等相关工作； 启动kthreadd进程（pid=2）：是Linux系统的内核进程，会创建内核工作线程kworkder，软中断线程ksoftirqd，thermal等内核守护进程。kthreadd进程是所有内核进程的鼻祖。

Android的boot.img存放的就是Linux内核和一个根文件系统。Bootloader会把boot.img映像装载进内存。然后Linux内核会执行整个系统的初始化，完成后装载根文件系统，最后启动Init进程。

#### 3.3、启动Init进程

Linux内核加载完毕后，会首先启动Init进程，Init进程是系统的第一个进程。在Init进程的启动过程中，会解析Linux的配置脚本init.rc文件，根据init.rc文件的内容，Init进程会装载Android的文件系统、创建系统目录。初始 化属性系统、启动Android系统重要的守护进程，这些进程包括USB守护进程、adb守护进程、vold守护进程、rild守护进程。

启动init进程(pid=1),是Linux系统的用户进程，init进程是所有用户进程的鼻祖。

init进程启动Media Server(多媒体服务)、servicemanager(binder服务管家)、bootanim(开机动画)等重要服务 init进程还会孵化出installd、ueventd、adbd、等用户守护进程； init进程孵化出Zygote进程，Zygote进程是Android系统的首个Java进程，Zygote是所有Java进程的父进程，Zygote进程本身是由init进程孵化而来的。 [Android 7.0 init.rc的一点改变 - 哈哈的个人专栏 - CSDN博客](http://blog.csdn.net/sunao2002002/article/details/52454878)

```java
/system/core/rootdir/init.rc
.....
service ueventd /sbin/ueventd
class core
critical
seclabel u:r:ueventd:s0

service healthd /sbin/healthd
class core
critical
seclabel u:r:healthd:s0
group root system wakelock
......
```

#### 3.4、启动Zygote进程

init进程初始化结束时，会启动Zygote进程。Zygote进程负责fork出应用进程，是所有应用进程的父进程。Zygote进程初始化时会创建Dalivik虚拟机、预装系统的资源文件和Java类。所有从Zygote进程fork出的用户进程将继承和共享这些预加载的资源，不用浪费时间重新加载，加快了应用程序的启动过程。启动结束后，Zygote进程也将变成守护进程，负责响应和启动APK应用程序的请求：

```java
/system/core/rootdir/init.zygote64.rc
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server
class main
socket zygote stream 660 root system
onrestart write /sys/android_power/request_state wake
onrestart write /sys/power/state on
onrestart restart audioserver
onrestart restart cameraserver
onrestart restart media
onrestart restart netd
writepid /dev/cpuset/foreground/tasks
```

#### 3.5、启动SystemServer

SystemServer是Zygote进程fork出的第一个进程，也是整个Android系统的核心进程。在SystemServer中运行着系统大部分的Binder服务，SystemServer首先启动本地服务SensorService；接着启动ActivityManagerService、WindowManagerService、PackageManagerService在内的所有Java服务。

Zygote进程fork出System Server进程，System Server是Zygote孵化的第一个进程，地位非常重要； System Server进程：负责启动和管理整个Java framework，包含ActivityManager，PowerManager等服务。 Media Server进程：负责启动和管理整个C++ framework，包含AudioFlinger，Camera Service等服务。

```java
/frameworks/base/services/java/com/android/server/SystemServer.java
SystemServer().run()
```

#### 3.6、启动ActivityManagerService

#### 3.7、启动Launcher(Activity)

SystemServer加载完所有的Java服务后，最后会调用ActivityManagerService的SystemReady()方法，在这个方法的执行中，会发出Intent"android.intent.category.HOME"。凡是响应这个Intent的APK都会运行起来，Launcher应用就是Android系统默认的桌面应用，一般只有它会响应这个Intent，因此，系统开机后，第一个运行的应用就是Launcher。 Zygote进程孵化出的第一个App进程是Launcher；

```java
/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
startHomeActivityLocked(mCurrentUserId, "systemReady");
```

### 四、设备启动过程详细分析

### （1）、启动Init进程

概述： init是Linux系统中用户空间的第一个进程，进程号为1。Kernel启动后，在用户空间，启动init进程，并调用init中的main()方法执行init进程的职责。对于init进程的功能分为4部分：

分析和运行所有的init.rc文件; 生成设备驱动节点; （通过rc文件创建） 处理子进程的终止(signal方式); 提供属性服务。 接下来从main()方法说起。

#### 4.1.1、main()

[-> init.cpp]

```c
int main(int argc, char** argv) {
if (!strcmp(basename(argv[0]), "ueventd")) {
    return ueventd_main(argc, argv);
}

if (!strcmp(basename(argv[0]), "watchdogd")) {
    return watchdogd_main(argc, argv);
}

// Clear the umask.
//设置文件属性0777
umask(0);

add_environment("PATH", _PATH_DEFPATH);

bool is_first_stage = (argc == 1) || (strcmp(argv[1], "--second-stage") != 0);

// Get the basic filesystem setup we need put together in the initramdisk
// on / and then we'll let the rc file figure out the rest.
//创建文件系统目录并挂载相关的文件系统
if (is_first_stage) {
    mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755");
    mkdir("/dev/pts", 0755);
    mkdir("/dev/socket", 0755);
    mount("devpts", "/dev/pts", "devpts", 0, NULL);
    #define MAKE_STR(x) __STRING(x)
    mount("proc", "/proc", "proc", 0, "hidepid=2,gid=" MAKE_STR(AID_READPROC));
    mount("sysfs", "/sys", "sysfs", 0, NULL);
}

// We must have some place other than / to create the device nodes for
// kmsg and null, otherwise we won't be able to remount / read-only
// later on. Now that tmpfs is mounted on /dev, we can actually talk
// to the outside world.
//屏蔽标准的输入输出
open_devnull_stdio();
 //初始化kernel log，位于设备节点/dev/kmsg
klog_init();
 //设置输出的log级别
klog_set_level(KLOG_NOTICE_LEVEL);
 // 输出init启动阶段的log
NOTICE("init %s started!\n", is_first_stage ? "first stage" : "second stage");

if (!is_first_stage) {
    // Indicate that booting is in progress to background fw loaders, etc.
    close(open("/dev/.booting", O_WRONLY | O_CREAT | O_CLOEXEC, 0000));
    //创建一块共享的内存空间，用于属性服务
    property_init();

    // If arguments are passed both on the command line and in DT,
    // properties set in DT always have priority over the command-line ones.
    process_kernel_dt();
    process_kernel_cmdline();

    // Propagate the kernel variables to internal variables
    // used by init as well as the current required properties.
    export_kernel_boot_props();
}

// Set up SELinux, including loading the SELinux policy if we're in the kernel domain.
selinux_initialize(is_first_stage);

// If we're in the kernel domain, re-exec init to transition to the init domain now
// that the SELinux policy has been loaded.
if (is_first_stage) {
    if (restorecon("/init") == -1) {
        ERROR("restorecon failed: %s\n", strerror(errno));
        security_failure();
    }
    char* path = argv[0];
    char* args[] = { path, const_cast<char*>("--second-stage"), nullptr };
    if (execv(path, args) == -1) {
        ERROR("execv(\"%s\") failed: %s\n", path, strerror(errno));
        security_failure();
    }
}

// These directories were necessarily created before initial policy load
// and therefore need their security context restored to the proper value.
// This must happen before /dev is populated by ueventd.
NOTICE("Running restorecon...\n");
restorecon("/dev");
restorecon("/dev/socket");
restorecon("/dev/__properties__");
restorecon("/property_contexts");
restorecon_recursive("/sys");

epoll_fd = epoll_create1(EPOLL_CLOEXEC);
if (epoll_fd == -1) {
    ERROR("epoll_create1 failed: %s\n", strerror(errno));
    exit(1);
}
//初始化子进程退出的信号处理过程
signal_handler_init();

property_load_boot_defaults();
export_oem_lock_status();
start_property_service();

const BuiltinFunctionMap function_map;
Action::set_function_map(&function_map);

Parser& parser = Parser::GetInstance();
parser.AddSectionParser("service",std::make_unique<ServiceParser>());
parser.AddSectionParser("on", std::make_unique<ActionParser>());
parser.AddSectionParser("import", std::make_unique<ImportParser>());
parser.ParseConfig("/init.rc");

ActionManager& am = ActionManager::GetInstance();

am.QueueEventTrigger("early-init");

// Queue an action that waits for coldboot done so we know ueventd has set up all of /dev...
am.QueueBuiltinAction(wait_for_coldboot_done_action, "wait_for_coldboot_done");
// ... so that we can start queuing up actions that require stuff from /dev.
am.QueueBuiltinAction(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");
am.QueueBuiltinAction(set_mmap_rnd_bits_action, "set_mmap_rnd_bits");
am.QueueBuiltinAction(keychord_init_action, "keychord_init");
am.QueueBuiltinAction(console_init_action, "console_init");

// Trigger all the boot actions to get us started.
am.QueueEventTrigger("init");

// Repeat mix_hwrng_into_linux_rng in case /dev/hw_random or /dev/random
// wasn't ready immediately after wait_for_coldboot_done
am.QueueBuiltinAction(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");

// Don't mount filesystems or start core system services in charger mode.
std::string bootmode = property_get("ro.bootmode");
if (bootmode == "charger") {
    am.QueueEventTrigger("charger");
} else {
    am.QueueEventTrigger("late-init");
}

// Run all property triggers based on current state of the properties.
am.QueueBuiltinAction(queue_property_triggers_action, "queue_property_triggers");

while (true) {
    if (!waiting_for_exec) {
        am.ExecuteOneCommand();
        restart_processes();
    }

    int timeout = -1;
    if (process_needs_restart) {
        timeout = (process_needs_restart - gettime()) * 1000;
        if (timeout < 0)
            timeout = 0;
    }

    if (am.HasMoreCommands()) {
        timeout = 0;
    }

    bootchart_sample(&timeout);

    epoll_event ev;
    int nr = TEMP_FAILURE_RETRY(epoll_wait(epoll_fd, &ev, 1, timeout));
    if (nr == -1) {
        ERROR("epoll_wait failed: %s\n", strerror(errno));
    } else if (nr == 1) {
        ((void (*)()) ev.data.ptr)();
    }
}

return 0;
}
```

#### 4.1.2、创建文件系统目录并挂载相关的文件系统

此时android的log系统还没有启动，采用kernel的log系统，打开的设备节点/dev/kmsg， 那么可通过cat /dev/kmsg来获取内核log。

接下来，设置log的输出级别为KLOG_NOTICE_LEVEL(5)，当log级别小于5时则会输出到kernel log， 默认值为3.

```c
add_environment("PATH", _PATH_DEFPATH);

bool is_first_stage = (argc == 1) || (strcmp(argv[1], "--second-stage") != 0);

// Get the basic filesystem setup we need put together in the initramdisk
// on / and then we'll let the rc file figure out the rest.
if (is_first_stage) {
    mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755");
    mkdir("/dev/pts", 0755);
    mkdir("/dev/socket", 0755);
    mount("devpts", "/dev/pts", "devpts", 0, NULL);
    #define MAKE_STR(x) __STRING(x)
    mount("proc", "/proc", "proc", 0, "hidepid=2,gid=" MAKE_STR(AID_READPROC));
    mount("sysfs", "/sys", "sysfs", 0, NULL);
}
```

该部分主要用于创建和挂载启动所需的文件目录。 需要注意的是，在编译Android系统源码时，在生成的根文件系统中，并不存在这些目录，它们是系统运行时的目录，即当系统终止时，就会消失。

在init初始化过程中，Android分别挂载了tmpfs，devpts，proc，sysfs这4类文件系统。

tmpfs是一种虚拟内存文件系统，它会将所有的文件存储在虚拟内存中，如果你将tmpfs文件系统卸载后，那么其下的所有的内容将不复存在。 tmpfs既可以使用RAM，也可以使用交换分区，会根据你的实际需要而改变大小。tmpfs的速度非常惊人，毕竟它是驻留在RAM中的，即使用了交换分区，性能仍然非常卓越。 由于tmpfs是驻留在RAM的，因此它的内容是不持久的。断电后，tmpfs的内容就消失了，这也是被称作tmpfs的根本原因。

devpts文件系统为伪终端提供了一个标准接口，它的标准挂接点是/dev/ pts。只要pty的主复合设备/dev/ptmx被打开，就会在/dev/pts下动态的创建一个新的pty设备文件。

proc文件系统是一个非常重要的虚拟文件系统，它可以看作是内核内部数据结构的接口，通过它我们可以获得系统的信息，同时也能够在运行时修改特定的内核参数。

与proc文件系统类似，sysfs文件系统也是一个不占有任何磁盘空间的虚拟文件系统。它通常被挂接在/sys目录下。sysfs文件系统是Linux2.6内核引入的，它把连接在系统上的设备和总线组织成为一个分级的文件，使得它们可以在用户空间存取。

#### 4.1.3、屏蔽标准的输入输出

[-> init.cpp]

```c
void open_devnull_stdio(void)
{
// Try to avoid the mknod() call if we can. Since SELinux makes
// a /dev/null replacement available for free, let's use it.
int fd = open("/sys/fs/selinux/null", O_RDWR);
if (fd == -1) {
    // OOPS, /sys/fs/selinux/null isn't available, likely because
    // /sys/fs/selinux isn't mounted. Fall back to mknod.
    static const char *name = "/dev/__null__";
    if (mknod(name, S_IFCHR | 0600, (1 << 8) | 3) == 0) {
        fd = open(name, O_RDWR);
        unlink(name);
    }
    if (fd == -1) {
        exit(1);
    }
}

dup2(fd, 0);
dup2(fd, 1);
dup2(fd, 2);
if (fd > 2) {
    close(fd);
}
}
```

前文生成/dev目录后，init进程将调用open_devnull_stdio函数，屏蔽标准的输入输出。 open_devnull_stdio函数会在/dev目录下生成__null__设备节点文件，并将标准输入、标准输出、标准错误输出全部重定向到__null__设备中。 open_devnull_stdio函数定义于system/core/init/util.cpp中。

这里需要说明的是，dup2函数的作用是用来复制一个文件的描述符，通常用来重定向进程的stdin、stdout和stderr。它的函数原形是：

int dup2(int oldfd, int targetfd)

该函数执行后，targetfd将变成oldfd的复制品。

因此上述过程其实就是：创建出**null**设备后，将0、1、2绑定到**null**设备上。因此init进程调用open_devnull_stdio函数后，通过标准的输入输出无法输出信息。

#### 4.1.4、初始化内核log系统

我们继续回到init进程的main函数，init进程通过klog_init函数，提供输出log信息的设备。 [-> init.cpp]

```c
klog_init();
klog_set_level(KLOG_NOTICE_LEVEL);

void klog_init(void) {
if (klog_fd >= 0) return; /* Already initialized */

klog_fd = open("/dev/kmsg", O_WRONLY | O_CLOEXEC);
if (klog_fd >= 0) {
    return;
}

static const char* name = "/dev/__kmsg__";
if (mknod(name, S_IFCHR | 0600, (1 << 8) | 11) == 0) {
    klog_fd = open(name, O_WRONLY | O_CLOEXEC);
    unlink(name);
}
}
```

klog_init函数定义于system/core/libcutils/klog.c中。通过klog_init函数，init进程生成kmsg设备节点文件。该设备可以调用内核信息输出函数printk，以输出log信息。

#### 4.1.5、初始化属性域

```c
if (!is_first_stage) {
.......
property_init();
.......
}
```

调用property_init初始化属性域。在Android平台中，为了让运行中的所有进程共享系统运行时所需要的各种设置值，系统开辟了属性存储区域，并提供了访问该区域的API。

这里存在一个问题是，在init进程中有部分代码块以is_first_stage标志进行区分，决定是否需要进行初始化。 is_first_stage的值，由init进程main函数的入口参数决定，之前不太明白具体的含义。 后来写博客后，有朋友留言，在引入selinux机制后，有些操作必须要在内核态才能完成； 但init进程作为android的第一个进程，又是运行在用户态的。 于是，最终设计为用is_first_stage进行区分init进程的运行状态。init进程在运行的过程中，会完成从内核态到用户态的切换。

```c
void property_init() {
if (__system_property_area_init()) {
    ERROR("Failed to initialize property area\n");
    exit(1);
}
}
```

property_init函数定义于system/core/init/property_service.cpp中，如上面代码所示，最终调用_system_property_area_init函数初始化属性域。

#### 4.1.6、完成SELinux相关工作

```c
// Set up SELinux, including loading the SELinux policy if we're in the kernel domain.
selinux_initialize(is_first_stage);
static void selinux_initialize(bool in_kernel_domain) {
Timer t;

selinux_callback cb;
//用于打印log的回调函数
cb.func_log = selinux_klog_callback;
selinux_set_callback(SELINUX_CB_LOG, cb);
//用于检查权限的回调函数
cb.func_audit = audit_callback;
selinux_set_callback(SELINUX_CB_AUDIT, cb);

if (in_kernel_domain) {
   //内核态处理流程
    INFO("Loading SELinux policy...\n");
    //用于加载sepolicy文件。该函数最终将sepolicy文件传递给kernel，这样kernel就有了安全策略配置文件，后续的MAC才能开展起来。
    if (selinux_android_load_policy() < 0) {
        ERROR("failed to load policy: %s\n", strerror(errno));
        security_failure();
    }
    //内核中读取的信息
    bool kernel_enforcing = (security_getenforce() == 1);
    //命令行中得到的数据
    bool is_enforcing = selinux_is_enforcing();
    if (kernel_enforcing != is_enforcing) {
        //用于设置selinux的工作模式。selinux有两种工作模式：
        //1、”permissive”，所有的操作都被允许（即没有MAC），但是如果违反权限的话，会记录日志
        //2、”enforcing”，所有操作都会进行权限检查。在一般的终端中，应该工作于enforing模式
        if (security_setenforce(is_enforcing)) {
            ERROR("security_setenforce(%s) failed: %s\n",
                  is_enforcing ? "true" : "false", strerror(errno));
            //将重启进入recovery mode
            security_failure();
        }
    }

    if (write_file("/sys/fs/selinux/checkreqprot", "0") == -1) {
        security_failure();
    }

    NOTICE("(Initializing SELinux %s took %.2fs.)\n",
           is_enforcing ? "enforcing" : "non-enforcing", t.duration());
} else {
    selinux_init_all_handles();
}
}
```

init进程进程调用selinux_initialize启动SELinux。从注释来看，init进程的运行确实是区分用户态和内核态的。

#### 4.1.7、重新设置属性

```c
// If we're in the kernel domain, re-exec init to transition to the init domain now
// that the SELinux policy has been loaded.
if (is_first_stage) {
//按selinux policy要求，重新设置init文件属性
    if (restorecon("/init") == -1) {
        ERROR("restorecon failed: %s\n", strerror(errno));
        security_failure();
    }
    char* path = argv[0];
    char* args[] = { path, const_cast<char*>("--second-stage"), nullptr };
    //这里就是前面所说的，启动用户态的init进程，即second-stage
    if (execv(path, args) == -1) {
        ERROR("execv(\"%s\") failed: %s\n", path, strerror(errno));
        security_failure();
    }
}

// These directories were necessarily created before initial policy load
// and therefore need their security context restored to the proper value.
// This must happen before /dev is populated by ueventd.
NOTICE("Running restorecon...\n");
restorecon("/dev");
restorecon("/dev/socket");
restorecon("/dev/__properties__");
restorecon("/property_contexts");
restorecon_recursive("/sys");
```

上述文件节点在加载Sepolicy之前已经被创建了，因此在加载完Sepolicy后，需要重新设置相关的属性。

#### 4.1.8、创建epoll句柄

如下面代码所示，init进程调用epoll_create1创建epoll句柄。

```c
epoll_fd = epoll_create1(EPOLL_CLOEXEC);
if (epoll_fd == -1) {
    ERROR("epoll_create1 failed: %s\n", strerror(errno));
    exit(1);
}
```

在linux的网络编程中，很长的时间都在使用select来做事件触发。在linux新的内核中，有了一种替换它的机制，就是epoll。相比于select，epoll最大的好处在于它不会随着监听fd数目的增长而降低效率。因为在内核中的select实现中，它是采用轮询来处理的，轮询的fd数目越多，自然耗时越多。

epoll机制一般使用epoll_create(int size)函数创建epoll句柄，size用来告诉内核这个句柄可监听的fd的数目。注意这个参数不同于select()中的第一个参数，在select中需给出最大监听数加1的值。 此外，当创建好epoll句柄后，它就会占用一个fd值，在linux下如果查看/proc/进程id/fd/，能够看到创建出的fd，因此在使用完epoll后，必须调用close()关闭，否则可能导致fd被耗尽。 上述代码使用的epoll_create1(EPOLLCLOEXEC)来创建epoll句柄，该标志位表示生成的epoll fd具有"执行后关闭"特性。

#### 4.1.9、装载子进程信号处理器

```c
void signal_handler_init() {
// Create a signalling mechanism for SIGCHLD.
int s[2];
//利用socketpair创建出已经连接的两个socket，分别作为信号的读、写端
if (socketpair(AF_UNIX, SOCK_STREAM | SOCK_NONBLOCK | SOCK_CLOEXEC, 0, s) == -1) {
    ERROR("socketpair failed: %s\n", strerror(errno));
    exit(1);
}

signal_write_fd = s[0];
signal_read_fd = s[1];

// Write to signal_write_fd if we catch SIGCHLD.
struct sigaction act;
memset(&act, 0, sizeof(act));
//信号处理器为SIGCHLD_handler，其被存在sigaction结构体中，负责处理SIGCHLD消息
act.sa_handler = SIGCHLD_handler;
act.sa_flags = SA_NOCLDSTOP;
//调用信号安装函数sigaction，将监听的信号及对应的信号处理器注册到内核中
sigaction(SIGCHLD, &act, 0);
//相对于6.0的代码，进一步作了封装，用于终止出现问题的子进程，详细代码于后文分析。
ServiceManager::GetInstance().ReapAnyOutstandingChildren();

register_epoll_handler(signal_read_fd, handle_signal);
}
```

Linux进程通过互相发送接收消息来实现进程间的通信，这些消息被称为"信号"。每个进程在处理其它进程发送的信号时都要注册处理者，处理者被称为信号处理器。

注意到sigaction结构体的sa_flags为SA_NOCLDSTOP。由于系统默认在子进程暂停时也会发送信号SIGCHLD，init需要忽略子进程在暂停时发出的SIGCHLD信号，因此将act.sa_flags 置为SA_NOCLDSTOP，该标志位表示仅当进程终止时才接受SIGCHLD信号。

我们来看看SIGCHLD_handler的具体工作。

```c
static void SIGCHLD_handler(int) {
if (TEMP_FAILURE_RETRY(write(signal_write_fd, "1", 1)) == -1) {
    ERROR("write(signal_write_fd) failed: %s\n", strerror(errno));
}
}
```

从上面代码我们知道，init进程是所有进程的父进程，当其子进程终止产生SIGCHLD信号时，SIGCHLD_handler对signal_write_fd执行写操作。由于socketpair的绑定关系，这将触发信号对应的signal_read_fd收到数据。

在装载信号监听器的最后，signal_handler_init调用register_epoll_handler，其代码如下所示，传入参数分别为signal_read_fd和handle_signal。

```c
void register_epoll_handler(int fd, void (*fn)()) {
epoll_event ev;
ev.events = EPOLLIN;
ev.data.ptr = reinterpret_cast<void*>(fn);
//epoll_fd增加一个监听对象fd,fd上有数据到来时，调用fn处理
if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, fd, &ev) == -1) {
    ERROR("epoll_ctl failed: %s\n", strerror(errno));
}
}
```

根据代码，我们知道：当epoll句柄监听到signal_read_fd中有数据可读时，将调用handle_signal进行处理。

至此，结合上文我们知道：当init进程调用signal_handler_init后，一旦收到子进程终止带来的SIGCHLD消息后，将利用信号处理者SIGCHLD_handler向signal_write_fd写入信息； epoll句柄监听到signal_read_fd收消息后，将调用handle_signal进行处理。整个过程如下图所示。 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/android.boot/N05-android-system-start-signal_handler.jpg)

```c
static void handle_signal() {
// Clear outstanding requests.
char buf[32];
read(signal_read_fd, buf, sizeof(buf));

ServiceManager::GetInstance().ReapAnyOutstandingChildren();
}
```

从代码中可以看出，handle_signal只是清空signal_read_fd中的数据，然后调用ServiceManager::GetInstance().ReapAnyOutstandingChildren()。

ServiceManager定义于system/core/init/service.cpp中，是一个单例对象：

```c
............
//c中默认是private属性
ServiceManager::ServiceManager() {
}

ServiceManager& ServiceManager::GetInstance() {
static ServiceManager instance;
return instance;
}
void ServiceManager::ReapAnyOutstandingChildren() {
while (ReapOneProcess()) {
}
}
............
```

ReapAnyOutstandingChildren函数实际上调用了ReapOneProcess。 我们结合代码，看看ReapOneProcess的具体工作。

```c
bool ServiceManager::ReapOneProcess() {
int status;
//用waitpid函数获取状态发生变化的子进程pid
//waitpid的标记为WNOHANG，即非阻塞，返回为正值就说明有进程挂掉了
pid_t pid = TEMP_FAILURE_RETRY(waitpid(-1, &status, WNOHANG));
if (pid == 0) {
    return false;
} else if (pid == -1) {
    ERROR("waitpid failed: %s\n", strerror(errno));
    return false;
}

//利用FindServiceByPid函数，找到pid对应的服务。
//FindServiceByPid主要通过轮询解析init.rc生成的service_list，找到pid与参数一致的srvc。
Service* svc = FindServiceByPid(pid);
//输出服务结束的原因
.........
if (!svc) {
    return true;
}

//结束服务，相对于6.0作了进一步的封装
if (svc->Reap()) {
    waiting_for_exec = false;
    //移除服务对应的信息
    RemoveService(*svc);
}

return true;
}
bool Service::Reap() {
//清理未携带SVC_ONESHOT 或 携带了SVC_RESTART标志的srvc的子进程
if (!(flags_ & SVC_ONESHOT) || (flags_ & SVC_RESTART)) {
    NOTICE("Service '%s' (pid %d) killing any children in process group\n", name_.c_str(), pid_);
    kill(-pid_, SIGKILL);
}
//清除srvc中创建出的socket
for (const auto& si : sockets_) {
    std::string tmp = StringPrintf(ANDROID_SOCKET_DIR "/%s", si.name.c_str());
    unlink(tmp.c_str());
}

if (flags_ & SVC_EXEC) {
    INFO("SVC_EXEC pid %d finished...\n", pid_);
    return true;
}

pid_ = 0;
flags_ &= (~SVC_RUNNING);

//对于携带了SVC_ONESHOT并且未携带SVC_RESTART的srvc，将这类服务的标志置为SVC_DISABLED，不再启动
if ((flags_ & SVC_ONESHOT) && !(flags_ & SVC_RESTART)) {
    flags_ |= SVC_DISABLED;
}

// Disabled and reset processes do not get restarted automatically.
if (flags_ & (SVC_DISABLED | SVC_RESET))  {
    svc->NotifyStateChange("stopped");
    return true;
}

time_t now = gettime();
//未携带SVC_RESTART的关键服务，在规定的间隔内，crash字数过多时，会导致整机重启；
if ((flags_ & SVC_CRITICAL) && !(flags_ & SVC_RESTART)) {
    if (time_crashed_ + CRITICAL_CRASH_WINDOW >= now) {
        if (++nr_crashed_ > CRITICAL_CRASH_THRESHOLD) {
            ..........
            android_reboot(ANDROID_RB_RESTART2, 0, "recovery");
            return true;
        }
    } else {
        time_crashed_ = now;
        nr_crashed_ = 1;
    }
}
//将待重启srvc的标志位置为SVC_RESTARTING（init进程将根据该标志位，重启服务）
flags_ &= (~SVC_RESTART);
flags_ |= SVC_RESTARTING;

// Execute all onrestart commands for this service.
//重启在init.rc文件中带有onrestart选项的服务，相对于6.0，此处也增加了封装性
onrestart_.ExecuteAllCommands();

svc->NotifyStateChange("restarting");
return true;
}
void Action::ExecuteAllCommands() const {
for (const auto& c : commands_) {
    ExecuteCommand(c);
}
}
void Action::ExecuteCommand(const Command& command) const {
Timer t;
//服务重启时，将执行对应的选项
int result = command.InvokeFunc();
//打印log
........
}
```

waitpid的函数原型为:

```c
pid_t waitpid(pid_t pid, int *status, int options)
```

其中，第一个参数pid为预等待的子进程的识别码，pid=-1表示等待任何子进程是否发出SIGCHLD。第二个参数status，用于返回子进程的结束状态。第三个参数决定waitpid函数是否处于阻塞处理方式，WNOHANG表示若pid指定的子进程没有结束，则waitpid()函数返回0，不予等待；若子进程结束，则返回子进程的pid。waitpid如果出错，则返回-1。

总结一下：整个signal_handler_init其实就是为了重启子进程用的，上述过程其实最终可以简化为下图： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/android.boot/N06-android-system-start-signal.jpg)

#### 4.1.10、设置默认系统属性

```c
property_load_boot_defaults();
```

接下来，进程调用property_load_boot_defaults进行默认属性配置相关的工作。

```c
void property_load_boot_defaults() {
load_properties_from_file(PROP_PATH_RAMDISK_DEFAULT, NULL);
```

如代码所示，property_load_boot_defaults实际上就是调用load_properties_from_file解析配置文件；然后根据解析的结果，设置系统属性。该部分功能较为单一，不再深入分析。

#### 4.1.11、配置属性的服务端

```c
start_property_service();
void start_property_service() {
//创建了一个非阻塞socket
property_set_fd = create_socket(PROP_SERVICE_NAME, SOCK_STREAM | SOCK_CLOEXEC | SOCK_NONBLOCK, 0666, 0, 0, NULL);
if (property_set_fd == -1) {
    ERROR("start_property_service socket creation failed: %s\n", strerror(errno));
    exit(1);
}
//调用listen函数监听property_set_fd， 于是该socket变成一个server
listen(property_set_fd, 8);
//监听server socket上是否有数据到来
register_epoll_handler(property_set_fd,  handle_property_set_fd);
}
```

我们知道，在create_socket函数返回套接字property_set_fd时，property_set_fd是一个主动连接的套接字。此时，系统假设用户会对这个套接字调用connect函数，期待它主动与其它进程连接。

由于在服务器编程中，用户希望这个套接字可以接受外来的连接请求，也就是被动等待用户来连接，于是需要调用listen函数使用主动连接套接字变为被连接套接字，使得一个进程可以接受其它进程的请求，从而成为一个服务器进程。

因此，调用listen后，init进程成为一个服务进程，其它进程可以通过property_set_fd连接init进程，提交设置系统属性的申请。

listen函数的第二个参数，涉及到一些网络的细节。

在进程处理一个连接请求的时候，可能还存在其它的连接请求。因为TCP连接是一个过程，所以可能存在一种半连接的状态。有时由于同时尝试连接的用户过多，使得服务器进程无法快速地完成连接请求。

因此，内核会在自己的进程空间里维护一个队列，以跟踪那些已完成连接但服务器进程还没有接手处理的用户，或正在进行的连接的用户。这样的一个队列不可能任意大，所以必须有一个上限。listen的第二个参数就是告诉内核使用这个数值作为上限。因此，init进程作为系统属性设置的服务器，最多可以同时为8个试图设置属性的用户提供服务。

在启动配置属性服务的最后，调用函数register_epoll_handler。根据上文所述，我们知道该函数将利用之前创建出的epoll句柄监听property_set_fd。当property_set_fd中有数据到来时，init进程将利用handle_property_set_fd函数进行处理。

```c
static void handle_property_set_fd() {
    ..........
    if ((s = accept(property_set_fd, (struct sockaddr *) &addr, &addr_size)) < 0) {
        return;
    }
    ........
    r = TEMP_FAILURE_RETRY(recv(s, &msg, sizeof(msg), MSG_DONTWAIT));
    .........
    switch(msg.cmd) {
    .........
    }
    .........
}
```

handle_propery_set_fd函数实际上是调用accept函数监听连接请求，接收property_set_fd中到来的数据，然后利用recv函数接受到来的数据，最后根据到来数据的类型，进行设置系统属性等相关操作，在此不做深入分析。

在这一部分的最后，我们简单举例介绍一下，系统属性改变的一些用途。 在init.rc中定义了一些与属性相关的触发器。当某个条件相关的属性被改变时，与该条件相关的触发器就会被触发。举例来说，如下面代码所示，debuggable属性变为1时，将执行启动console进程等操作。

```c
on property:ro.debuggable=1
# Give writes to anyone for the trace folder on debug builds.
# The folder is used to store method traces.
chmod 0773 /data/misc/trace
start console
```

总结一下，其它进程修改系统属性时，大致的流程如下图所示：其它的进程像init进程发送请求后，由init进程检查权限后，修改共享内存区。

#### 4.1.12、解析init.rc文件

关于解析init.rc的代码，Android 7.0相对于6.0，作了巨大的修改。

```c
//这里将Action的function_map_替换为BuiltinFunctionMap
//下文将通过BuiltinFuntionMap的map方法，获取keyword对应的处理函数
const BuiltinFunctionMap function_map;
Action::set_function_map(&function_map);

//构造出解析文件用的parser对象
Parser& parser = Parser::GetInstance();
//为一些类型的关键字，创建特定的parser
parser.AddSectionParser("service",std::make_unique<ServiceParser>());
parser.AddSectionParser("on", std::make_unique<ActionParser>());
parser.AddSectionParser("import", std::make_unique<ImportParser>());
//开始实际的解析过程
parser.ParseConfig("/init.rc");
........
```

在解析init.rc文件的过程前，我们先来简单介绍一下init.rc文件。 init.rc文件是在init进程启动后执行的启动脚本，文件中记录着init进程需执行的操作。在Android系统中，使用init.rc和init.{ hardware }.rc两个文件。

其中init.rc文件在Android系统运行过程中用于通用的环境设置与进程相关的定义，init.{hardware}.rc（例如，高通有init.qcom.rc，MTK有init.mediatek.rc）用于定义Android在不同平台下的特定进程和环境设置等。

此处解析函数传入的参数为"/init.rc"，解析的是运行时与init进程同在根目录下的init.rc文件。该文件在编译前，定义于system/core/rootdir/init.rc中（与平台相关的rc文件不在这里加载）。

init.rc文件大致分为两大部分，一部分是以"on"关键字开头的动作列表（action list）：

```c
on early-init
    # Set init and its forked children's oom_adj.
    write /proc/1/oom_score_adj -1000
    .........
    start ueventd
```

另一部分是以"service"关键字开头的服务列表（service list）：

```c
service ueventd /sbin/ueventd
    class core
    critical
    seclabel u:r:ueventd:s0
```

借助系统环境变量或Linux命令，动作列表用于创建所需目录，以及为某些特定文件指定权限，而服务列表用来记录init进程需要启动的一些子进程。如上面代码所示，service关键字后的第一个字符串表示服务（子进程）的名称，第二个字符串表示服务的执行路径。

接下来，我们从ParseConfig函数入手，逐步分析整个解析过程(函数定义于system/core/init/ init_parser.cpp中)：

```c
bool Parser::ParseConfig(const std::string& path) {
    if (is_dir(path.c_str())) {
        //传入参数为目录地址
        return ParseConfigDir(path);
    }
    //传入参数为文件地址
    return ParseConfigFile(path);
}

bool Parser::ParseConfigDir(const std::string& path) {
    ...........
    std::unique_ptr<DIR, int(*)(DIR*)> config_dir(opendir(path.c_str()), closedir);
    ..........
    //看起来很复杂，其实就是递归目录
    while ((current_file = readdir(config_dir.get()))) {
        std::string current_path = android::base::StringPrintf("%s/%s", path.c_str(), current_file->d_name);
        if (current_file->d_type == DT_REG) {
            //最终还是靠ParseConfigFile来解析实际的文件
            if (!ParseConfigFile(current_path)) {
                .............
            }
        }
    }
}
```

从上面的代码可以看出，解析init.rc文件的函数是ParseConfigFile：

```c
bool Parser::ParseConfigFile(const std::string& path) {
    ........
    std::string data;
    //读取路径指定文件中的内容，保存为字符串形式
    if (!read_file(path.c_str(), &data)) {
        return false;
    }
    .........
    //解析获取的字符串
    ParseData(path, data);
    .........
}
```

ParseData函数定义于system/core/init/init_parser.cpp中，根据关键字解析出服务和动作。动作与服务会以链表节点的形式注册到service_list与action_list中，service_list与action_list是init进程中声明的全局结构体，其中的关键代码下所示。

```c
void Parser::ParseData(const std::string& filename, const std::string& data) {
.......
parse_state state;
.......
std::vector<std::string> args;

for (;;) {
    //next_token以行为单位分割参数传递过来的字符串
    //最先走到T_TEXT分支
    switch (next_token(&state)) {
    case T_EOF:
        if (section_parser) {
            //EOF,解析结束
            section_parser->EndSection();
        }
        return;
    case T_NEWLINE:
        state.line++;
        if (args.empty()) {
            break;
        }
        //在前文创建parser时，我们为service，on，import定义了对应的parser
        //这里就是根据第一个参数，判断是否有对应的parser
        if (section_parsers_.count(args[0])) {
            if (section_parser) {
                //结束上一个parser的工作，将构造出的对象加入到对应的service_list与action_list中
                section_parser->EndSection();
            }
            //获取参数对应的parser
            section_parser = section_parsers_[args[0]].get();
            std::string ret_err;
            //调用实际parser的ParseSection函数
            if (!section_parser->ParseSection(args, &ret_err)) {
                parse_error(&state, "%s\n", ret_err.c_str());
                section_parser = nullptr;
            }
        } else if (section_parser) {
            std::string ret_err;
            //如果第一个参数不是service，on，import
            //则调用前一个parser的ParseLineSection函数
            //这里相当于解析一个参数块的子项
            if (!section_parser->ParseLineSection(args, state.filename, state.line, &ret_err)) {
                parse_error(&state, "%s\n", ret_err.c_str());
            }
        }
        //清空本次解析的数据
        args.clear();
        break;
    case T_TEXT:
        //将本次解析的内容写入到args中
        args.emplace_back(state.text);
        break;
    }
}
}
```

这里的解析看起来比较复杂，在6.0以前的版本中，整个解析是面向过程的。init进程统一调用一个函数来进行解析，然后在该函数中利用switch-case的形式，根据解析的内容进行相应的处理。 在Android 7.0中，为了更好地封装及面向对象，对于不同的关键字定义了不同的parser对象，每个对象通过多态实现自己的解析操作。

我们现在回忆一下init进程main函数中，创建parser的代码：

```c
...........
Parser& parser = Parser::GetInstance();
parser.AddSectionParser("service",std::make_unique<ServiceParser>());
parser.AddSectionParser("on", std::make_unique<ActionParser>());
parser.AddSectionParser("import", std::make_unique<ImportParser>());
...........
```

看看三个Parser的定义：

```c
class ServiceParser : public SectionParser {......}
class ActionParser : public SectionParser {......}
class ImportParser : public SectionParser {.......}
```

可以看到三个Parser均是继承SectionParser，具体的实现各有不同，我们以比较常用的ServiceParser和ActionParser为例，看看解析的结果如何处理。

##### 4.1.12.1 ServiceParser

ServiceParser定义于system/core/init/service.cpp中。从前面的代码，我们知道，解析一个service块，首先需要调用ParseSection函数，接着利用ParseLineSection处理子块，解析完所有数据后，调用EndSection。 因此，我们着重看看ServiceParser的这三个函数：

```c
    bool ServiceParser::ParseSection(.....) {
    .......
    const std::string& name = args[1];
    .......
    std::vector<std::string> str_args(args.begin() + 2, args.end());
    //主要根据参数，构造出一个service对象
    service_ = std::make_unique<Service>(name, "default", str_args);
    return true;
}

//注意这里已经在解析子项了
bool ServiceParser::ParseLineSection(......) const {
    //调用service对象的HandleLine
    return service_ ? service_->HandleLine(args, err) : false;
}

bool Service::HandleLine(.....) {
    ........
    //OptionHandlerMap继承自keywordMap<OptionHandler>
    static const OptionHandlerMap handler_map;
    //根据子项的内容，找到对应的handler函数
    //FindFunction定义与keyword模块中,FindFunction方法利用子类生成对应的map中，然后通过通用的查找方法，即比较键值找到对应的处理函数
    auto handler = handler_map.FindFunction(args[0], args.size() - 1, err);

    if (!handler) {
        return false;
    }
    //调用handler函数
    return (this->*handler)(args, err);
}

class Service::OptionHandlerMap : public KeywordMap<OptionHandler> {
    ...........
    Service::OptionHandlerMap::Map& Service::OptionHandlerMap::map() const {
    constexpr std::size_t kMax = std::numeric_limits<std::size_t>::max();
    static const Map option_handlers = {
        {"class",       {1,     1,    &Service::HandleClass}},
        {"console",     {0,     0,    &Service::HandleConsole}},
        {"critical",    {0,     0,    &Service::HandleCritical}},
        {"disabled",    {0,     0,    &Service::HandleDisabled}},
        {"group",       {1,     NR_SVC_SUPP_GIDS + 1, &Service::HandleGroup}},
        {"ioprio",      {2,     2,    &Service::HandleIoprio}},
        {"keycodes",    {1,     kMax, &Service::HandleKeycodes}},
        {"oneshot",     {0,     0,    &Service::HandleOneshot}},
        {"onrestart",   {1,     kMax, &Service::HandleOnrestart}},
        {"seclabel",    {1,     1,    &Service::HandleSeclabel}},
        {"setenv",      {2,     2,    &Service::HandleSetenv}},
        {"socket",      {3,     6,    &Service::HandleSocket}},
        {"user",        {1,     1,    &Service::HandleUser}},
        {"writepid",    {1,     kMax, &Service::HandleWritepid}},
    };
    return option_handlers;
}

//以class对应的处理函数为例，可以看出其实就是填充service对象对应的域
bool Service::HandleClass(const std::vector<std::string>& args, std::string* err) {
    classname_ = args[1];
    return true;
}

//注意此时service对象已经处理完毕
void ServiceParser::EndSection() {
    if (service_) {
        ServiceManager::GetInstance().AddService(std::move(service_));
    }
}

void ServiceManager::AddService(std::unique_ptr<Service> service) {
    Service* old_service = FindServiceByName(service->name());
    if (old_service) {
        ERROR("ignored duplicate definition of service '%s'",
              service->name().c_str());
        return;
    }
    //将service对象加入到services_里
    //7.0里，services_已经是个vector了
    services_.emplace_back(std::move(service));
}
```

从上面的一系列代码，我们可以看出ServiceParser其实就是：首先根据第一行的名字和参数创建出service对象，然后根据选项域的内容填充service对象，最后将创建出的service对象加入到vector类型的service链表中。

#### 4.1.12.2 ActionParser

ActionParser定义于system/core/init/action.cpp中。Action的解析过程，其实与Service一样，也是先后调用ParseSection， ParseLineSection和EndSection。

```c
bool ActionParser::ParseSection(....) {
    ........
    //创建出新的action对象
    auto action = std::make_unique<Action>(false);
    //根据参数，填充action的trigger域，不详细分析了
    if (!action->InitTriggers(triggers, err)) {
        return false;
    }
    .........
}

bool ActionParser::ParseLineSection(.......) const {
    //构造Action对象的command域
    return action_ ? action_->AddCommand(args, filename, line, err) : false;
}

bool Action::AddCommand(.....) {
    ........
    //找出action对应的执行函数
    auto function = function_map_->FindFunction(args[0], args.size() - 1, err);
    ........
    //利用所有信息构造出command，加入到action对象中
    AddCommand(function, args, filename, line);
    return true;
}

void Action::AddCommand(......) {
    commands_.emplace_back(f, args, filename, line);
}

void ActionParser::EndSection() {
    if (action_ && action_->NumCommands() > 0) {
        ActionManager::GetInstance().AddAction(std::move(action_));
    }
}

void ActionManager::AddAction(.....) {
    ........
    auto old_action_it = std::find_if(actions_.begin(),
                     actions_.end(),
                     [&action] (std::unique_ptr<Action>& a) {
                         return action->TriggersEqual(*a);
                     });

    if (old_action_it != actions_.end()) {
        (*old_action_it)->CombineAction(*action);
    } else {
        //加入到action链表中，类型也是vector，其中装的是指针
        actions_.emplace_back(std::move(action));
    }
}
```

从上面的代码可以看出，加载action块的逻辑和service一样，不同的是需要填充trigger和command域。当然，最后解析出的action也需要加入到action链表中。

这里最后还剩下一个问题，那就是哪里定义了Action中command对应处理函数？ 实际上，前文已经出现了过了，在init.cpp的main函数中：

```c
.......
const BuiltinFunctionMap function_map;
Action::set_function_map(&function_map);
.......
```

因此，Action中调用function_map_->FindFunction时，实际上调用的是BuiltinFunctionMap的FindFunction函数。我们已经知道FindFunction是keyword定义的通用函数，重点是重构的map函数。我们看看system/core/init/builtins.cpp：

```c
BuiltinFunctionMap::Map& BuiltinFunctionMap::map() const {
    constexpr std::size_t kMax = std::numeric_limits<std::size_t>::max();
    static const Map builtin_functions = {
        {"bootchart_init",          {0,     0,    do_bootchart_init}},
        {"chmod",                   {2,     2,    do_chmod}},
        {"chown",                   {2,     3,    do_chown}},
        {"class_reset",             {1,     1,    do_class_reset}},
        {"class_start",             {1,     1,    do_class_start}},
        {"class_stop",              {1,     1,    do_class_stop}},
        {"copy",                    {2,     2,    do_copy}},
        {"domainname",              {1,     1,    do_domainname}},
        {"enable",                  {1,     1,    do_enable}},
        {"exec",                    {1,     kMax, do_exec}},
        {"export",                  {2,     2,    do_export}},
        {"hostname",                {1,     1,    do_hostname}},
        {"ifup",                    {1,     1,    do_ifup}},
        {"init_user0",              {0,     0,    do_init_user0}},
        {"insmod",                  {1,     kMax, do_insmod}},
        {"installkey",              {1,     1,    do_installkey}},
        {"load_persist_props",      {0,     0,    do_load_persist_props}},
        {"load_system_props",       {0,     0,    do_load_system_props}},
        {"loglevel",                {1,     1,    do_loglevel}},
        {"mkdir",                   {1,     4,    do_mkdir}},
        {"mount_all",               {1,     kMax, do_mount_all}},
        {"mount",                   {3,     kMax, do_mount}},
        {"powerctl",                {1,     1,    do_powerctl}},
        {"restart",                 {1,     1,    do_restart}},
        {"restorecon",              {1,     kMax, do_restorecon}},
        {"restorecon_recursive",    {1,     kMax, do_restorecon_recursive}},
        {"rm",                      {1,     1,    do_rm}},
        {"rmdir",                   {1,     1,    do_rmdir}},
        {"setprop",                 {2,     2,    do_setprop}},
        {"setrlimit",               {3,     3,    do_setrlimit}},
        {"start",                   {1,     1,    do_start}},
        {"stop",                    {1,     1,    do_stop}},
        {"swapon_all",              {1,     1,    do_swapon_all}},
        {"symlink",                 {2,     2,    do_symlink}},
        {"sysclktz",                {1,     1,    do_sysclktz}},
        {"trigger",                 {1,     1,    do_trigger}},
        {"verity_load_state",       {0,     0,    do_verity_load_state}},
        {"verity_update_state",     {0,     0,    do_verity_update_state}},
        {"wait",                    {1,     2,    do_wait}},
        {"write",                   {2,     2,    do_write}},
    };
    return builtin_functions;
}
```

上述代码的第四项就是Action每个command对应的执行函数。

#### 4.1.13、向执行队列中添加其它action

介绍完init进程解析init.rc文件的过程后，我们继续将视角拉回到init进程的main函数：

```c
ActionManager& am = ActionManager::GetInstance();

am.QueueEventTrigger("early-init");

// Queue an action that waits for coldboot done so we know ueventd has set up all of /dev...
m.QueueBuiltinAction(wait_for_coldboot_done_action, "wait_for_coldboot_done");
// ... so that we can start queuing up actions that require stuff from /dev.
am.QueueBuiltinAction(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");
am.QueueBuiltinAction(set_mmap_rnd_bits_action, "set_mmap_rnd_bits");
am.QueueBuiltinAction(keychord_init_action, "keychord_init");
am.QueueBuiltinAction(console_init_action, "console_init");

// Trigger all the boot actions to get us started.
am.QueueEventTrigger("init");

// Repeat mix_hwrng_into_linux_rng in case /dev/hw_random or /dev/random
// wasn't ready immediately after wait_for_coldboot_done
am.QueueBuiltinAction(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");

// Don't mount filesystems or start core system services in charger mode.
std::string bootmode = property_get("ro.bootmode");
if (bootmode == "charger") {
    am.QueueEventTrigger("charger");
} else {
    am.QueueEventTrigger("late-init");
}

// Run all property triggers based on current state of the properties.
    am.QueueBuiltinAction(queue_property_triggers_action, "queue_property_triggers");
```

从上面的代码可以看出，接下来init进程中调用了大量的QueueEventTrigger和QueueBuiltinAction函数。

```c
void ActionManager::QueueEventTrigger(const std::string& trigger) {
    trigger_queue_.push(std::make_unique<EventTrigger>(trigger));
}
```

处QueueEventTrigger函数就是利用参数构造EventTrigger，然后加入到trigger_queue_中。后续init进程处理trigger事件时，将会触发相应的操作。根据前文的分析，我们知道实际上就是将action_list中，对应trigger与第一个参数匹配的action，加入到运行队列action_queue中。

```c
void ActionManager::QueueBuiltinAction(BuiltinFunction func, const std::string& name) {
    //创建action
    auto action = std::make_unique<Action>(true);
    std::vector<std::string> name_vector{name};

    //保证唯一性
    if (!action->InitSingleTrigger(name)) {
        return;
    }

    //创建action的cmd，指定执行函数和参数
    action->AddCommand(func, name_vector);

    trigger_queue_.push(std::make_unique<BuiltinTrigger>(action.get()));
    actions_.emplace_back(std::move(action));
}
```

QueueBuiltinAction函数中构造新的action加入到actions_中，第一个参数作为新建action携带cmd的执行函数；第二个参数既作为action的trigger name，也作为action携带cmd的参数。

#### 4.1.14、处理添加到运行队列的事件

```c
while (true) {
    //判断是否有事件需要处理
    if (!waiting_for_exec) {
        //依次执行每个action中携带command对应的执行函数
        am.ExecuteOneCommand();
        //重启一些挂掉的进程
        restart_processes();
    }

    //以下决定timeout的时间，将影响while循环的间隔
    int timeout = -1;
    //有进程需要重启时，等待该进程重启
    if (process_needs_restart) {
        timeout = (process_needs_restart - gettime()) * 1000;
        if (timeout < 0)
            timeout = 0;
    }

    //有action待处理，不等待
    if (am.HasMoreCommands()) {
        timeout = 0;
    }

    //bootchart_sample应该是进行性能数据采样
    bootchart_sample(&timeout);

    epoll_event ev;
    //没有事件到来的话，最多阻塞timeout时间
    int nr = TEMP_FAILURE_RETRY(epoll_wait(epoll_fd, &ev, 1, timeout));
    if (nr == -1) {
        ERROR("epoll_wait failed: %s\n", strerror(errno));
    } else if (nr == 1) {
        //有事件到来，执行对应处理函数
        //根据上文知道，epoll句柄（即epoll_fd）主要监听子进程结束，及其它进程设置系统属性的请求。
        ((void (*)()) ev.data.ptr)();
    }
}
```

从上面代码可以看出，init进程将所有需要操作的action加入运行队列后， 进入无限循环过程，不断处理运行队列中的事件，同时进行重启service等操作。

ExecuteOneCommand中的主要部分如下图所示。

```c
void ActionManager::ExecuteOneCommand() {
    // Loop through the trigger queue until we have an action to execute
    //当有可执行action或trigger queue为空时结束
    while (current_executing_actions_.empty() && !trigger_queue_.empty()) {
        //轮询actions链表
        for (const auto& action : actions_) {
            //依次查找trigger表
            if (trigger_queue_.front()->CheckTriggers(*action)) {
                //当action与trigger对应时，就可以执行当前action
                //一个trigger可以对应多个action，均加入current_executing_actions_
                current_executing_actions_.emplace(action.get());
            }
        }
        //trigger event出队
        trigger_queue_.pop();
    }

    if (current_executing_actions_.empty()) {
        return;
    }

    //每次只执行一个action，下次init进程while循环时，跳过上面的while循环，接着执行
    auto action = current_executing_actions_.front();

    if (current_command_ == 0) {
        std::string trigger_name = action->BuildTriggersString();
        INFO("processing action (%s)\n", trigger_name.c_str());
    }

    //实际的执行过程，此处仅处理当前action中的一个cmd
    action->ExecuteOneCommand(current_command_);

    //适当地清理工作，注意只有当前action中所有的command均执行完毕后，才会将该action从current_executing_actions_移除
    // If this was the last command in the current action, then remove
    // the action from the executing list.
    // If this action was oneshot, then also remove it from actions_.
    ++current_command_;
    if (current_command_ == action->NumCommands()) {
        current_executing_actions_.pop();
        current_command_ = 0;
        if (action->oneshot()) {
            auto eraser = [&action] (std::unique_ptr<Action>& a) {
                return a.get() == action;
            };
            actions_.erase(std::remove_if(actions_.begin(), actions_.end(), eraser));
        }
    }
}
```

```c
void Action::ExecuteCommand(const Command& command) const {
    Timer t;
    //执行该command对应的处理函数
    int result = command.InvokeFunc();
    ........
}
```

从代码可以看出，当while循环不断调用ExecuteOneCommand函数时，将按照trigger表的顺序，依次取出action链表中与trigger匹配的action。 每次均执行一个action中的一个command对应函数（一个action可能携带多个command）。 当一个action所有的command均执行完毕后，再执行下一个action。 当一个trigger对应的action均执行完毕后，再执行下一个trigger对应action。

restart_processes函数负责按需重启service，代码如下图所示。

```c
static void restart_processes() {
    process_needs_restart = 0;
    ServiceManager::GetInstance().ForEachServiceWithFlags(
        SVC_RESTARTING,
        [] (Service* s) {
            s->RestartIfNeeded(process_needs_restart);
        });
}
```

从上面可以看出，该函数轮询service对应的链表，对于有SVC_RESTARING标志的service执行RestartIfNeeded（如上文所述，当子进程终止时，init进程会将可被重启进程的服务标志位置为SVC_RESTARTING）。

如下面代码所示，restart_service_if_needed可以重新启动服务。

```c
void Service::RestartIfNeeded(time_t& process_needs_restart)(struct service *svc)
{
    time_t next_start_time = svc->time_started + 5;

    //两次服务启动时间的间隔要大于5s
    if (next_start_time <= gettime()) {
        svc->flags &= (~SVC_RESTARTING);
        //满足时间间隔的要求后，重启服务
        //Start将会重新fork服务进程，并做相应的配置
        Start(svc, NULL);
        return;
    }

    //更新main函数中，while循环需要等待的时间
    if ((next_start_time < process_needs_restart) ||
        (process_needs_restart == 0)) {
        process_needs_restart = next_start_time;
    }
}
```

查阅资料发现：Bootchart 是一个能对 GNU/Linux boot 过程进行性能分析并把结果直观化的工具。它在 boot 过程中搜集资源利用情况及进程信息然后以PNG, SVG或EPS格式来显示结果。BootChart 包含数据收集工具和图像产生工具。数据收集工具在原始的BootChart中是独立的shell程序，但在Android中，数据收集工具被集成到了init 程序中。资料与代码基本吻合。

### （2）、启动Zygote进程

#### 4.2.1、概述

Zygote是由init进程通过解析init.zygote.rc文件而创建的，zygote所对应的可执行程序app_process，所对应的源文件是App_main.cpp，进程名为zygote。

```c
service zygote /system/bin/app_process32 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote
    class main
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart netd
    writepid /dev/cpuset/foreground/tasks

service zygote_secondary /system/bin/app_process64 -Xzygote /system/bin --zygote --socket-name=zygote_secondary
    class main
    socket zygote_secondary stream 660 root system
    onrestart restart zygote
    writepid /dev/cpuset/foreground/tasks
```

Zygote进程能够重启的地方:

servicemanager进程被杀; (onresart) surfaceflinger进程被杀; (onresart) Zygote进程自己被杀; (oneshot=false) system_server进程被杀; (waitpid) 从App_main()开始，Zygote启动过程的函数调用类大致流程如下： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/android.boot/N07-android-system-start-zygote_process.jpg)
#### 4.2.2、Zygote启动过程

##### 4.2.2.1、App_main.main()

[-> App_main.cpp]

```c
int main(int argc, char* const argv[])
{
    if (prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0) < 0) {
        // Older kernels don't understand PR_SET_NO_NEW_PRIVS and return
        // EINVAL. Don't die on such kernels.
        if (errno != EINVAL) {
            LOG_ALWAYS_FATAL("PR_SET_NO_NEW_PRIVS failed: %s", strerror(errno));
            return 12;
        }
    }
    //传到的参数argv为“-Xzygote /system/bin --zygote --start-system-server”
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
    // Process command line arguments
    // ignore argv[0] //忽略第一个参数
    argc--;
    argv++;

    int i;
    for (i = 0; i < argc; i++) {
        if (argv[i][0] != '-') {
            break;
        }
        if (argv[i][1] == '-' && argv[i][2] == 0) {
            ++i; // Skip --.
            break;
        }
        runtime.addOption(strdup(argv[i]));
    }

    // Parse runtime arguments.  Stop at first unrecognized option.
    //参数解析
    bool zygote = false;
    bool startSystemServer = false;
    bool application = false;
    String8 niceName;
    String8 className;

    ++i;  // Skip unused "parent dir" argument.
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = ZYGOTE_NICE_NAME;
        } else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;
        } else if (strcmp(arg, "--application") == 0) {
            application = true;
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
            niceName.setTo(arg + 12);
        } else if (strncmp(arg, "--", 2) != 0) {
            className.setTo(arg);
            break;
        } else {
            --i;
            break;
        }
    }

    Vector<String8> args;
    if (!className.isEmpty()) {
        // We're not in zygote mode, the only argument we need to pass
        // to RuntimeInit is the application argument.
        //
        // The Remainder of args get passed to startup class main(). Make
        // copies of them before we overwrite them with the process name.
        // 运行application或tool程序
        args.add(application ? String8("application") : String8("tool"));
        runtime.setClassNameAndArgs(className, argc - i, argv + i);
    } else {
        // We're in zygote mode.
         //进入zygote模式，创建 /data/dalvik-cache路径
        maybeCreateDalvikCache();

        if (startSystemServer) {
            args.add(String8("start-system-server"));
        }

        char prop[PROP_VALUE_MAX];
        if (property_get(ABI_LIST_PROPERTY, prop, NULL) == 0) {
            LOG_ALWAYS_FATAL("app_process: Unable to determine ABI list from property %s.",
                ABI_LIST_PROPERTY);
            return 11;
        }

        String8 abiFlag("--abi-list=");
        abiFlag.append(prop);
        args.add(abiFlag);

        // In zygote mode, pass all remaining arguments to the zygote
        // main() method.
        for (; i < argc; ++i) {
            args.add(String8(argv[i]));
        }
    }
    //设置进程名
    if (!niceName.isEmpty()) {
        runtime.setArgv0(niceName.string());
        set_process_name(niceName.string());
    }

    if (zygote) {
       // 启动AppRuntime
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
       //没有指定类名或zygote，参数错误
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
        return 10;
    }
}
```

#### 4.2.2.2、AndroidRuntime.start()

[-> AndroidRuntime.cpp]

```c
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
ALOGD(">>>>>> START %s uid %d <<<<<<\n",
        className != NULL ? className : "(unknown)", getuid());

static const String8 startSystemServer("start-system-server");

/*
 * 'startSystemServer == true' means runtime is obsolete and not run from
 * init.rc anymore, so we print out the boot start event here.
 */
for (size_t i = 0; i < options.size(); ++i) {
    if (options[i] == startSystemServer) {
       /* track our progress through the boot sequence */
       const int LOG_BOOT_PROGRESS_START = 3000;
       LOG_EVENT_LONG(LOG_BOOT_PROGRESS_START,  ns2ms(systemTime(SYSTEM_TIME_MONOTONIC)));
    }
}

const char* rootDir = getenv("ANDROID_ROOT");
if (rootDir == NULL) {
    rootDir = "/system";
    if (!hasDir("/system")) {
        LOG_FATAL("No root directory specified, and /android does not exist.");
        return;
    }
    setenv("ANDROID_ROOT", rootDir, 1);
}

//const char* kernelHack = getenv("LD_ASSUME_KERNEL");
//ALOGD("Found LD_ASSUME_KERNEL='%s'\n", kernelHack);

/* start the virtual machine */
JniInvocation jni_invocation;
jni_invocation.Init(NULL);
JNIEnv* env;
// 虚拟机创建
if (startVm(&mJavaVM, &env, zygote) != 0) {
    return;
}
onVmCreated(env);

/*
 * Register android functions.
 */
 // JNI方法注册
if (startReg(env) < 0) {
    ALOGE("Unable to register all android natives\n");
    return;
}

/*
 * We want to call main() with a String array with arguments in it.
 * At present we have two arguments, the class name and an option string.
 * Create an array to hold them.
 */
jclass stringClass;
jobjectArray strArray;
jstring classNameStr;
//等价 strArray= new String[options.size() + 1];
stringClass = env->FindClass("java/lang/String");
assert(stringClass != NULL);
strArray = env->NewObjectArray(options.size() + 1, stringClass, NULL);
assert(strArray != NULL);
//等价 strArray[0] = "com.android.internal.os.ZygoteInit"
classNameStr = env->NewStringUTF(className);
assert(classNameStr != NULL);
env->SetObjectArrayElement(strArray, 0, classNameStr);
//等价 strArray[1] = "start-system-server"；
//    strArray[2] = "--abi-list=xxx"；
//其中xxx为系统响应的cpu架构类型，比如arm64-v8a.
for (size_t i = 0; i < options.size(); ++i) {
    jstring optionsStr = env->NewStringUTF(options.itemAt(i).string());
    assert(optionsStr != NULL);
    env->SetObjectArrayElement(strArray, i + 1, optionsStr);
}

/*
 * Start VM.  This thread becomes the main thread of the VM, and will
 * not return until the VM exits.
 */
 //将"com.android.internal.os.ZygoteInit"转换为"com/android/internal/os/ZygoteInit"
char* slashClassName = toSlashClassName(className);
jclass startClass = env->FindClass(slashClassName);
if (startClass == NULL) {
    ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
    /* keep going */
} else {
    jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
        "([Ljava/lang/String;)V");
    if (startMeth == NULL) {
        ALOGE("JavaVM unable to find main() in '%s'\n", className);
        /* keep going */
    } else {
        // 调用ZygoteInit.main()方法
        env->CallStaticVoidMethod(startClass, startMeth, strArray);
        #if 0
        if (env->ExceptionCheck())
            threadExitUncaughtException(env);
            #endif
    }
}
free(slashClassName);

ALOGD("Shutting down VM\n");
if (mJavaVM->DetachCurrentThread() != JNI_OK)
    ALOGW("Warning: unable to detach main thread\n");
if (mJavaVM->DestroyJavaVM() != 0)
    ALOGW("Warning: VM did not shut down cleanly\n");
    }
```

#### 4.2.2.3、AndroidRuntime.startVm()

[–> AndroidRuntime.cpp]

```c
int AndroidRuntime::startVm(JavaVM** pJavaVM, JNIEnv** pEnv, bool zygote)
{
// JNI检测功能，用于native层调用jni函数时进行常规检测，比较弱字符串格式是否符合要求，资源是否正确释放。该功能一般用于早期系统调试或手机Eng版，对于User版往往不会开启，引用该功能比较消耗系统CPU资源，降低系统性能。
   bool checkJni = false;
property_get("dalvik.vm.checkjni", propBuf, "");
if (strcmp(propBuf, "true") == 0) {
    checkJni = true;
} else if (strcmp(propBuf, "false") != 0) {
    /* property is neither true nor false; fall back on kernel parameter */
    property_get("ro.kernel.android.checkjni", propBuf, "");
    if (propBuf[0] == '1') {
        checkJni = true;
    }
}
ALOGD("CheckJNI is %s\n", checkJni ? "ON" : "OFF");
if (checkJni) {
    addOption("-Xcheck:jni")
}
......
 //虚拟机产生的trace文件，主要用于分析系统问题，路径默认为/data/anr/traces.txt
parseRuntimeOption("dalvik.vm.stack-trace-file", stackTraceFileBuf, "-Xstacktracefile:");
//对于不同的软硬件环境，这些参数往往需要调整、优化，从而使系统达到最佳性能
parseRuntimeOption("dalvik.vm.heapstartsize", heapstartsizeOptsBuf, "-Xms", "4m");
parseRuntimeOption("dalvik.vm.heapsize", heapsizeOptsBuf, "-Xmx", "16m");
parseRuntimeOption("dalvik.vm.heapgrowthlimit", heapgrowthlimitOptsBuf, "-XX:HeapGrowthLimit=");
parseRuntimeOption("dalvik.vm.heapminfree", heapminfreeOptsBuf, "-XX:HeapMinFree=");
parseRuntimeOption("dalvik.vm.heapmaxfree", heapmaxfreeOptsBuf, "-XX:HeapMaxFree=");
parseRuntimeOption("dalvik.vm.heaptargetutilization",
                   heaptargetutilizationOptsBuf, "-XX:HeapTargetUtilization=");
...

//preloaded-classes文件内容是由WritePreloadedClassFile.java生成的，
//在ZygoteInit类中会预加载工作将其中的classes提前加载到内存，以提高系统性能
if (!hasFile("/system/etc/preloaded-classes")) {
    return -1;
}

//初始化虚拟机
if (JNI_CreateJavaVM(pJavaVM, pEnv, &initArgs) < 0) {
    ALOGE("JNI_CreateJavaVM failed\n");
    return -1;
}
}
```

创建Java虚拟机方法的主要篇幅是关于虚拟机参数的设置，下面只列举部分在调试优化过程中常用参数。

#### 4.2.2.4、AndroidRuntime.startReg()

```c
int AndroidRuntime::startReg(JNIEnv* env)
{
    //设置线程创建方法为javaCreateThreadEtc
    androidSetCreateThreadFunc((android_create_thread_fn) javaCreateThreadEtc);

    env->PushLocalFrame(200);
    //进程NI方法的注册
    if (register_jni_procs(gRegJNI, NELEM(gRegJNI), env) < 0) {
        env->PopLocalFrame(NULL);
        return -1;
    }
    env->PopLocalFrame(NULL);
    return 0;
}
```

#### 4.2.2.4.1、Threads.androidSetCreateThreadFunc()

[-> Threads.cpp]

```c
void androidSetCreateThreadFunc(android_create_thread_fn func)
{
    gCreateThreadFn = func;
}
```

虚拟机启动后startReg()过程，会设置线程创建函数指针gCreateThreadFn指向javaCreateThreadEtc.

#### 4.2.2.4.2、register_jni_procs()

```c
static int register_jni_procs(const RegJNIRec array[], size_t count, JNIEnv* env)
{
    for (size_t i = 0; i < count; i++) {
        if (array[i].mProc(env) < 0) {
            return -1;
        }
    }
    return 0;
}
```

#### 4.2.2.4.3、gRegJNI.mProc

```c
static const RegJNIRec gRegJNI[] = {
    REG_JNI(register_com_android_internal_os_RuntimeInit),
    REG_JNI(register_android_os_Binder)，
    ...
}；*
```

array[i]是指gRegJNI数组, 该数组有100多个成员。其中每一项成员都是通过REG_JNI宏定义的：

```c
#define REG_JNI(name)      { name }
struct RegJNIRec {
    int (*mProc)(JNIEnv*);
};
```

可见，调用mProc，就等价于调用其参数名所指向的函数。 例如REG_JNI(register_com_android_internal_os_RuntimeInit).mProc也就是指进入register_com_android_internal_os_RuntimeInit方法，接下来就继续以此为例来说明：

```c
int register_com_android_internal_os_RuntimeInit(JNIEnv* env)
{
    return jniRegisterNativeMethods(env, "com/android/internal/os/RuntimeInit",
        gMethods, NELEM(gMethods));
}
```

//gMethods：java层方法名与jni层的方法的一一映射关系

```c
static JNINativeMethod gMethods[] = {
    { "nativeFinishInit", "()V",
        (void*) com_android_internal_os_RuntimeInit_nativeFinishInit },
    { "nativeZygoteInit", "()V",
        (void*) com_android_internal_os_RuntimeInit_nativeZygoteInit },
    { "nativeSetExitWithoutCleanup", "(Z)V",
        (void*) com_android_internal_os_RuntimeInit_nativeSetExitWithoutCleanup },
};
```

![Markdown](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/android.boot/N08-android-system-start-ZygoteInit.main.png)

#### 4.2.2.5、进入Java层

AndroidRuntime.start()执行到最后通过反射调用到ZygoteInit.main(),见下文:

##### 4.2.2.5.1、ZygoteInit.main()

```java
public static void main(String argv[]) {
try {Init");
    //开启DDMS功能
    RuntimeInit.enableDdms();
    SamplingProfilerIntegration.start();
    boolean startSystemServer = false;
    String socketName = "zygote";
    String abiList = null;
    for (int i = 1; i < argv.length; i++) {
        if ("start-system-server".equals(argv[i])) {
            startSystemServer = true;
        } else if (argv[i].startsWith(ABI_LIST_ARG)) {
            abiList = argv[i].substring(ABI_LIST_ARG.length());
        } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
            socketName = argv[i].substring(SOCKET_NAME_ARG.length());
        } else {
            throw new RuntimeException("Unknown command line argument: " + argv[i]);
        }
    }
    ......
    //为Zygote注册socket
    registerZygoteSocket(socketName);
    preload();// 预加载类和资源
    SamplingProfilerIntegration.writeZygoteSnapshot();
    gcAndFinalize();//GC操作
    Zygote.nativeUnmountStorageOnInit();
    ZygoteHooks.stopZygoteNoThreadCreation();
    if (startSystemServer) {//启动system_server
        startSystemServer(abiList, socketName);
    }
    runSelectLoop(abiList);//进入循环模式

    closeServerSocket();
} catch (MethodAndArgsCaller caller) {
    caller.run();
} catch (Throwable ex) {
    closeServerSocket();
    throw ex;
}
}
```

在异常捕获后调用的方法caller.run()，会在后续的system_server文章会讲到。

##### 4.2.2.5.2、ZygoteInit.registerZygoteSocket()

```java
private static void registerZygoteSocket(String socketName) {
if (sServerSocket == null) {
    int fileDesc;
    final String fullSocketName = ANDROID_SOCKET_PREFIX + socketName;
    try {
        String env = System.getenv(fullSocketName);
        fileDesc = Integer.parseInt(env);
    } catch (RuntimeException ex) {
        ...
    }

    try {
        FileDescriptor fd = new FileDescriptor();
        fd.setInt$(fileDesc); //设置文件描述符
        sServerSocket = new LocalServerSocket(fd); //创建Socket的本地服务端
    } catch (IOException ex) {
        ...
    }
}
}
```

##### 4.2.2.5.2、ZygoteInit.preload()

```java
static void preload() {
    //预加载位于/system/etc/preloaded-classes文件中的类
    preloadClasses();

    //预加载资源，包含drawable和color资源
    preloadResources();

    //预加载OpenGL
    preloadOpenGL();

    //通过System.loadLibrary()方法，
    //预加载"android","compiler_rt","jnigraphics"这3个共享库
    preloadSharedLibraries();

    //预加载  文本连接符资源
    preloadTextResources();

    //仅用于zygote进程，用于内存共享的进程
    WebViewFactory.prepareWebViewInZygote();
}
```

执行Zygote进程的初始化,对于类加载，采用反射机制Class.forName()方法来加载。对于资源加载，主要是 com.android.internal.R.array.preloaded_drawables和com.android.internal.R.array.preloaded_color_state_lists，在应用程序中以com.android.internal.R.xxx开头的资源，便是此时由Zygote加载到内存的。

zygote进程内加载了preload()方法中的所有资源，当需要fork新进程时，采用copy on write技术，如下：

![Markdown](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/android.boot/N09-android-system-start-zygote_fork.jpg)

##### 4.2.2.5.3、ZygoteInit.startSystemServer()

```java
private static boolean startSystemServer(String abiList, String socketName)
        throws MethodAndArgsCaller, RuntimeException {
    long capabilities = posixCapabilitiesAsBits(
        OsConstants.CAP_BLOCK_SUSPEND,
        OsConstants.CAP_KILL,
        OsConstants.CAP_NET_ADMIN,
        OsConstants.CAP_NET_BIND_SERVICE,
        OsConstants.CAP_NET_BROADCAST,
        OsConstants.CAP_NET_RAW,
        OsConstants.CAP_SYS_MODULE,
        OsConstants.CAP_SYS_NICE,
        OsConstants.CAP_SYS_RESOURCE,
        OsConstants.CAP_SYS_TIME,
        OsConstants.CAP_SYS_TTY_CONFIG
    );
    //参数准备
    String args[] = {
        "--setuid=1000",
        "--setgid=1000",
        "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1032,3001,3002,3003,3006,3007",
        "--capabilities=" + capabilities + "," + capabilities,
        "--nice-name=system_server",
        "--runtime-args",
        "com.android.server.SystemServer",
    };

    ZygoteConnection.Arguments parsedArgs = null;
    int pid;
    try {
        //用于解析参数，生成目标格式
        parsedArgs = new ZygoteConnection.Arguments(args);
        ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
        ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);

        // fork子进程，用于运行system_server
        pid = Zygote.forkSystemServer(
                parsedArgs.uid, parsedArgs.gid,
                parsedArgs.gids,
                parsedArgs.debugFlags,
                null,
                parsedArgs.permittedCapabilities,
                parsedArgs.effectiveCapabilities);
    } catch (IllegalArgumentException ex) {
        throw new RuntimeException(ex);
    }

    //进入子进程system_server
    if (pid == 0) {
        if (hasSecondZygote(abiList)) {
            waitForSecondaryZygote(socketName);
        }
        // 完成system_server进程剩余的工作
        handleSystemServerProcess(parsedArgs);
    }
    return true;
}
```

准备参数并fork新进程，从上面可以看出system server进程参数信息为uid=1000,gid=1000,进程名为sytem_server，从zygote进程fork新进程后，需要关闭zygote原有的socket。另外，对于有两个zygote进程情况，需等待第2个zygote创建完成。

##### 4.2.2.5.4、ZygoteInit.runSelectLoop()

```java
private static void runSelectLoop(String abiList) throws MethodAndArgsCaller {
    ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
    ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();
    //sServerSocket是socket通信中的服务端，即zygote进程。保存到fds[0]
    fds.add(sServerSocket.getFileDescriptor());
    peers.add(null);

    while (true) {
        StructPollfd[] pollFds = new StructPollfd[fds.size()];
        for (int i = 0; i < pollFds.length; ++i) {
            pollFds[i] = new StructPollfd();
            pollFds[i].fd = fds.get(i);
            pollFds[i].events = (short) POLLIN;
        }
        try {
             //处理轮询状态，当pollFds有事件到来则往下执行，否则阻塞在这里
            Os.poll(pollFds, -1);
        } catch (ErrnoException ex) {
            ...
        }

        for (int i = pollFds.length - 1; i >= 0; --i) {
            //采用I/O多路复用机制，当接收到客户端发出连接请求 或者数据处理请求到来，则往下执行；
            // 否则进入continue，跳出本次循环。
            if ((pollFds[i].revents & POLLIN) == 0) {
                continue;
            }
            if (i == 0) {
                //即fds[0]，代表的是sServerSocket，则意味着有客户端连接请求；
                // 则创建ZygoteConnection对象,并添加到fds。
                ZygoteConnection newPeer = acceptCommandPeer(abiList);
                peers.add(newPeer);
                fds.add(newPeer.getFileDesciptor()); //添加到fds.
            } else {
                //i>0，则代表通过socket接收来自对端的数据，并执行相应操作
                boolean done = peers.get(i).runOnce();
                if (done) {
                    peers.remove(i);
                    fds.remove(i); //处理完则从fds中移除该文件描述符
                }
            }
        }
    }
}
```

Zygote采用高效的I/O多路复用机制，保证在没有客户端连接请求或数据处理时休眠，否则响应客户端的请求。

##### 4.2.2.5.4、ZygoteConnection.runOnce()

```java
boolean runOnce() throws ZygoteInit.MethodAndArgsCaller {
String args[];
Arguments parsedArgs = null;
FileDescriptor[] descriptors;

try {
    //读取socket客户端发送过来的参数列表
    args = readArgumentList();
    descriptors = mSocket.getAncillaryFileDescriptors();
} catch (IOException ex) {
    ...
    return true;
}
...

try {
    //将binder客户端传递过来的参数，解析成Arguments对象格式
    parsedArgs = new Arguments(args);
    ...
    pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
            parsedArgs.debugFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,
            parsedArgs.niceName, fdsToClose, parsedArgs.instructionSet,
            parsedArgs.appDataDir);
} catch (Exception e) {
    ...
}

try {
    if (pid == 0) {
        //子进程执行
        IoUtils.closeQuietly(serverPipeFd);
        serverPipeFd = null;
        //进入子进程流程
        handleChildProc(parsedArgs, descriptors, childPipeFd, newStderr);
        return true;
    } else {
        //父进程执行
        IoUtils.closeQuietly(childPipeFd);
        childPipeFd = null;
        return handleParentProc(pid, descriptors, serverPipeFd, parsedArgs);
    }
} finally {
    IoUtils.closeQuietly(childPipeFd);
    IoUtils.closeQuietly(serverPipeFd);
}
```

##### 4.2.2.6、总结

Zygote启动过程的调用流程图：

![Markdown](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/android.boot/N10-android-system-start-zygote_start.jpg)

> 1、解析init.zygote.rc中的参数，创建AppRuntime并调用AppRuntime.start()方法；
2、 调用AndroidRuntime的startVM()方法创建虚拟机，再调用startReg()注册JNI函数；
3、通过JNI方式调用ZygoteInit.main()，第一次进入Java世界；
4、registerZygoteSocket()建立socket通道，zygote作为通信的服务端，用于响应客户端请求；
5、preload()预加载通用类、drawable和color资源、openGL以及共享库以及WebView，用于提高app启动效率；
6、zygote完毕大部分工作，接下来再通过startSystemServer()，fork得力帮手system_server进程，也是上层framework的运行载体。
7、 zygote功成身退，调用runSelectLoop()，随时待命，当接收到请求创建新进程请求时立即唤醒并执行相应工作。 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/android.boot/N11-android-system-start-System_Server.png)

### （3）、启动SystemServer上篇

#### 4.3.1、启动流程

SystemServer的在Android体系中所处的地位，SystemServer由Zygote fork生成的，进程名为system_server，该进程承载着framework的核心服务。 Android系统启动-zygote篇中讲到Zygote启动过程中会调用startSystemServer()，可知startSystemServer()函数是system_server启动流程的起点， 启动流程图如下： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/android.boot/N12-android-system-start-system_server_.jpg)

#### 4.3.2、ZygoteInit.startSystemServer()

```java
private static boolean startSystemServer(String abiList, String socketName)
        throws MethodAndArgsCaller, RuntimeException {
    ...
    //参数准备
    String args[] = {
        "--setuid=1000",
        "--setgid=1000",
        "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1032,3001,3002,3003,3006,3007",
        "--capabilities=" + capabilities + "," + capabilities,
        "--nice-name=system_server",
        "--runtime-args",
        "com.android.server.SystemServer",
    };

    ZygoteConnection.Arguments parsedArgs = null;
    int pid;
    try {
        //用于解析参数，生成目标格式
        parsedArgs = new ZygoteConnection.Arguments(args);
        ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
        ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);

        // fork子进程，该进程是system_server进程
        pid = Zygote.forkSystemServer(
                parsedArgs.uid, parsedArgs.gid,
                parsedArgs.gids,
                parsedArgs.debugFlags,
                null,
                parsedArgs.permittedCapabilities,
                parsedArgs.effectiveCapabilities);
    } catch (IllegalArgumentException ex) {
        throw new RuntimeException(ex);
    }

    //进入子进程system_server
    if (pid == 0) {
        if (hasSecondZygote(abiList)) {
            waitForSecondaryZygote(socketName);
        }
        // 完成system_server进程剩余的工作
        handleSystemServerProcess(parsedArgs);
    }
    return true;
}
```

准备参数并fork新进程，从上面可以看出system server进程参数信息为uid=1000,gid=1000,进程名为sytem_server，从zygote进程fork新进程后，需要关闭zygote原有的socket。另外，对于有两个zygote进程情况，需等待第2个zygote创建完成。

#### 4.3.3、Zygote. forkSystemServer()

```java
public static int forkSystemServer(int uid, int gid, int[] gids, int debugFlags,
        int[][] rlimits, long permittedCapabilities, long effectiveCapabilities) {
    VM_HOOKS.preFork();
    // 调用native方法fork system_server进程
    int pid = nativeForkSystemServer(
            uid, gid, gids, debugFlags, rlimits, permittedCapabilities, effectiveCapabilities);
    if (pid == 0) {
        Trace.setTracingEnabled(true);
    }
    VM_HOOKS.postForkCommon();
    return pid;
}
```

nativeForkSystemServer()方法在AndroidRuntime.cpp中注册的，调用com_android_internal_os_Zygote.cpp中的register_com_android_internal_os_Zygote()方法建立native方法的映射关系，所以接下来进入如下方法。

#### 4.3.4、com_android_internal_os_Zygote.nativeForkSystemServer()

```java
static jint com_android_internal_os_Zygote_nativeForkSystemServer(
        JNIEnv* env, jclass, uid_t uid, gid_t gid, jintArray gids,
        jint debug_flags, jobjectArray rlimits, jlong permittedCapabilities,
        jlong effectiveCapabilities) {
  //fork子进程，
  pid_t pid = ForkAndSpecializeCommon(env, uid, gid, gids,
                                      debug_flags, rlimits,
                                      permittedCapabilities, effectiveCapabilities,
                                      MOUNT_EXTERNAL_DEFAULT, NULL, NULL, true, NULL,
                                      NULL, NULL);
  if (pid > 0) {
      // zygote进程，检测system_server进程是否创建
      gSystemServerPid = pid;
      int status;
      if (waitpid(pid, &status, WNOHANG) == pid) {
          //当system_server进程死亡后，重启zygote进程
          RuntimeAbort(env);
      }
  }
  return pid;
}
```

当system_server进程创建失败时，将会重启zygote进程。这里需要注意，对于Android 5.0以上系统，有两个zygote进程，分别是zygote、zygote64两个进程，system_server的父进程，一般来说64位系统其父进程是zygote64进程

当kill system_server进程后，只重启zygote64和system_server，不重启zygote; 当kill zygote64进程后，只重启zygote64和system_server，也不重启zygote； 当kill zygote进程，则重启zygote、zygote64以及system_server。

#### 4.3.5、com_android_internal_os_Zygote.ForkAndSpecializeCommon()

```java
static pid_t ForkAndSpecializeCommon(JNIEnv* env, uid_t uid, gid_t gid, jintArray javaGids,
                                     jint debug_flags, jobjectArray javaRlimits,
                                     jlong permittedCapabilities, jlong effectiveCapabilities,
                                     jint mount_external,
                                     jstring java_se_info, jstring java_se_name,
                                     bool is_system_server, jintArray fdsToClose,
                                     jstring instructionSet, jstring dataDir) {
  SetSigChldHandler(); //设置子进程的signal信号处理函数
  pid_t pid = fork(); //fork子进程
  if (pid == 0) {
    //进入子进程
    DetachDescriptors(env, fdsToClose); //关闭并清除文件描述符

    if (!is_system_server) {
        //对于非system_server子进程，则创建进程组
        int rc = createProcessGroup(uid, getpid());
    }
    SetGids(env, javaGids); //设置设置group
    SetRLimits(env, javaRlimits); //设置资源limit

    int rc = setresgid(gid, gid, gid);
    rc = setresuid(uid, uid, uid);

    SetCapabilities(env, permittedCapabilities, effectiveCapabilities);
    SetSchedulerPolicy(env); //设置调度策略

     //selinux上下文
    rc = selinux_android_setcontext(uid, is_system_server, se_info_c_str, se_name_c_str);

    if (se_info_c_str == NULL && is_system_server) {
      se_name_c_str = "system_server";
    }
    if (se_info_c_str != NULL) {
      SetThreadName(se_name_c_str); //设置线程名为system_server，方便调试
    }
    UnsetSigChldHandler(); //设置子进程的signal信号处理函数为默认函数
    //等价于调用zygote.callPostForkChildHooks()
    env->CallStaticVoidMethod(gZygoteClass, gCallPostForkChildHooks, debug_flags,
                              is_system_server ? NULL : instructionSet);
    ...

  } else if (pid > 0) {
    //进入父进程，即zygote进程
  }
  return pid;
}
```

fork()创建新进程，采用copy on write方式，这是linux创建进程的标准方法，会有两次return,对于pid==0为子进程的返回，对于pid>0为父进程的返回。 到此system_server进程已完成了创建的所有工作，接下来开始了system_server进程的真正工作。在前面startSystemServer()方法中，zygote进程执行完forkSystemServer()后，新创建出来的system_server进程便进入handleSystemServerProcess()方法。

#### 4.3.5、ZygoteInit.handleSystemServerProcess()

```java
private static void handleSystemServerProcess(
        ZygoteConnection.Arguments parsedArgs)
        throws ZygoteInit.MethodAndArgsCaller {

    closeServerSocket(); //关闭父进程zygote复制而来的Socket

    Os.umask(S_IRWXG | S_IRWXO);

    if (parsedArgs.niceName != null) {
        Process.setArgV0(parsedArgs.niceName); //设置当前进程名为"system_server"
    }

    final String systemServerClasspath = Os.getenv("SYSTEMSERVERCLASSPATH");
    if (systemServerClasspath != null) {
        //执行dex优化操作
        performSystemServerDexOpt(systemServerClasspath);
    }

    if (parsedArgs.invokeWith != null) {
        String[] args = parsedArgs.remainingArgs;

        if (systemServerClasspath != null) {
            String[] amendedArgs = new String[args.length + 2];
            amendedArgs[0] = "-cp";
            amendedArgs[1] = systemServerClasspath;
            System.arraycopy(parsedArgs.remainingArgs, 0, amendedArgs, 2, parsedArgs.remainingArgs.length);
        }
        //启动应用进程
        WrapperInit.execApplication(parsedArgs.invokeWith,
                parsedArgs.niceName, parsedArgs.targetSdkVersion,
                VMRuntime.getCurrentInstructionSet(), null, args);
    } else {
        ClassLoader cl = null;
        if (systemServerClasspath != null) {
            创建类加载器，并赋予当前线程
            cl = new PathClassLoader(systemServerClasspath, ClassLoader.getSystemClassLoader());
            Thread.currentThread().setContextClassLoader(cl);
        }

        //system_server故进入此分支
        RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl);
    }

    /* should never reach here */
}
```

此处systemServerClasspath环境变量主要有/system/framework/目录下的services.jar，ethernet-service.jar, wifi-service.jar这3个文件

#### 4.3.6、ZygoteInit.performSystemServerDexOpt()

```java
private static void performSystemServerDexOpt(String classPath) {
    final String[] classPathElements = classPath.split(":");
    //创建一个与installd的建立socket连接
    final InstallerConnection installer = new InstallerConnection();
    //执行ping操作，直到与installd服务端连通为止
    installer.waitForConnection();
    final String instructionSet = VMRuntime.getRuntime().vmInstructionSet();

    try {
        for (String classPathElement : classPathElements) {
            final int dexoptNeeded = DexFile.getDexOptNeeded(
                    classPathElement, "*", instructionSet, false /* defer */);
            if (dexoptNeeded != DexFile.NO_DEXOPT_NEEDED) {
                //以system权限，执行dex文件优化
                installer.dexopt(classPathElement, Process.SYSTEM_UID, false,
                        instructionSet, dexoptNeeded);
            }
        }
    } catch (IOException ioe) {
        throw new RuntimeException("Error starting system_server", ioe);
    } finally {
        installer.disconnect(); //断开与installd的socket连接
    }
}
```

将classPath字符串中的apk，分别进行dex优化操作。真正执行优化工作通过socket通信将相应的命令参数，发送给installd来完成。

#### 4.3.7、RuntimeInit.zygoteInit()

```java
public static final void zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
        throws ZygoteInit.MethodAndArgsCaller {

    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "RuntimeInit");
    redirectLogStreams(); //重定向log输出

    commonInit(); // 通用的一些初始化
    nativeZygoteInit(); // zygote初始化
    applicationInit(targetSdkVersion, argv, classLoader); // 应用初始化
}
```

#### 4.3.8、RuntimeInit.commonInit()

```java
private static final void commonInit() {
    // 设置默认的未捕捉异常处理方法
    Thread.setDefaultUncaughtExceptionHandler(new UncaughtHandler());

    // 设置市区，中国时区为"Asia/Shanghai"
    TimezoneGetter.setInstance(new TimezoneGetter() {
        @Override
        public String getId() {
            return SystemProperties.get("persist.sys.timezone");
        }
    });
    TimeZone.setDefault(null);

    //重置log配置
    LogManager.getLogManager().reset();
    new AndroidConfig();

    // 设置默认的HTTP User-agent格式,用于 HttpURLConnection。
    String userAgent = getDefaultUserAgent();
    System.setProperty("http.agent", userAgent);

    // 设置socket的tag，用于网络流量统计
    NetworkManagementSocketTagger.install();
}
```

默认的HTTP User-agent格式，例如：

"Dalvik/1.1.0 (Linux; U; Android 6.0.1；LenovoX3c70 Build/LMY47V)".

#### 4.3.9、AndroidRuntime.nativeZygoteInit()

nativeZygoteInit()方法在AndroidRuntime.cpp中，进行了jni映射，对应下面的方法。

```java
static void com_android_internal_os_RuntimeInit_nativeZygoteInit(JNIEnv* env, jobject clazz)
{
    //此处的gCurRuntime为AppRuntime，是在AndroidRuntime.cpp中定义的
    gCurRuntime->onZygoteInit();
}
```

```c
[–>app_main.cpp]

virtual void onZygoteInit()
{
    sp<ProcessState> proc = ProcessState::self();
    proc->startThreadPool(); //启动新binder线程
}
```

ProcessState::self()是单例模式，主要工作是调用open()打开/dev/binder驱动设备，再利用mmap()映射内核的地址空间，将Binder驱动的fd赋值ProcessState对象中的变量mDriverFD，用于交互操作。startThreadPool()是创建一个新的binder线程，不断进行talkWithDriver()，在binder系列文章中的[注册服务(addService)](http://gityuan.com/2015/11/14/binder-add-service/)详细这两个方法的执行原理。

#### 4.3.10、RuntimeInit.applicationInit()

```java
private static void applicationInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
        throws ZygoteInit.MethodAndArgsCaller {
    //true代表应用程序退出时不调用AppRuntime.onExit()，否则会在退出前调用
    nativeSetExitWithoutCleanup(true);

    //设置虚拟机的内存利用率参数值为0.75
    VMRuntime.getRuntime().setTargetHeapUtilization(0.75f);
    VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);

    final Arguments args;
    try {
        args = new Arguments(argv); //解析参数
    } catch (IllegalArgumentException ex) {
        return;
    }

    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);

    //调用startClass的static方法 main()
    invokeStaticMain(args.startClass, args.startArgs, classLoader);
}
```

在startSystemServer()方法中通过硬编码初始化参数，可知此处args.startClass为"com.android.server.SystemServer"。

#### 4.3.11、RuntimeInit.invokeStaticMain()

```java
private static void invokeStaticMain(String className, String[] argv, ClassLoader classLoader)
        throws ZygoteInit.MethodAndArgsCaller {
    Class<?> cl = Class.forName(className, true, classLoader);
    ...

    Method m;
    try {
        m = cl.getMethod("main", new Class[] { String[].class });
    } catch (NoSuchMethodException ex) {
        ...
    } catch (SecurityException ex) {
        ...
    }

    int modifiers = m.getModifiers();
    if (! (Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {
        ...
    }

    //通过抛出异常，回到ZygoteInit.main()。这样做好处是能清空栈帧，提高栈帧利用率。
    throw new ZygoteInit.MethodAndArgsCaller(m, argv);
}
```

#### 4.3.12、MethodAndArgsCaller.run()

在Zygote中遗留了一个问题没有讲解，如下：

[–>ZygoteInit.java]

```java
public static void main(String argv[]) {
    try {
        startSystemServer(abiList, socketName);//启动system_server
        ....
    } catch (MethodAndArgsCaller caller) {
        caller.run();
    } catch (RuntimeException ex) {
        closeServerSocket();
        throw ex;
    }
}
```

现在已经很明显了，是invokeStaticMain()方法中抛出的异常MethodAndArgsCaller，从而进入caller.run()方法。

[–>ZygoteInit.java]

```java
public static class MethodAndArgsCaller extends Exception
        implements Runnable {

    public void run() {
        try {
            //根据传递过来的参数，可知此处通过反射机制调用的是SystemServer.main()方法
            mMethod.invoke(null, new Object[] { mArgs });
        } catch (IllegalAccessException ex) {
            throw new RuntimeException(ex);
        } catch (InvocationTargetException ex) {
            Throwable cause = ex.getCause();
            if (cause instanceof RuntimeException) {
                throw (RuntimeException) cause;
            } else if (cause instanceof Error) {
                throw (Error) cause;
            }
            throw new RuntimeException(ex);
        }
    }
}
```

到此，总算是进入到了SystemServer类的main()方法， 在文章Android系统启动-SystemServer下篇中会紧接着这里开始讲述。

### （4）、启动SystemServer下篇

上篇文章Android系统启动-systemServer上篇 从Zygote一路启动到SystemServer的过程。 简单回顾下，在RuntimeInit.java中invokeStaticMain方法通过创建并抛出异常ZygoteInit.MethodAndArgsCaller，在ZygoteInit.java中的main()方法会捕捉该异常，并调用caller.run()，再通过反射便会调用到SystemServer.main()方法，该方法主要执行流程：

```java
SystemServer.main
    SystemServer.run
        createSystemContext
        startBootstrapServices();
        startCoreServices();
        startOtherServices();
        Looper.loop();
```

接下来，从其main方法说起。

#### 4.4.1、SystemServer.main()

```java
public final class SystemServer {
    ...
    public static void main(String[] args) {
        //先初始化SystemServer对象，再调用对象的run()方法
        new SystemServer().run();
    }
}
```

#### 4.4.2、SystemServer.run()

```java
private void run() {
    //当系统时间比1970年更早，就设置当前系统时间为1970年
    if (System.currentTimeMillis() < EARLIEST_SUPPORTED_TIME) {
        SystemClock.setCurrentTimeMillis(EARLIEST_SUPPORTED_TIME);
    }

    //变更虚拟机的库文件，对于Android 6.0默认采用的是libart.so
    SystemProperties.set("persist.sys.dalvik.vm.lib.2", VMRuntime.getRuntime().vmLibrary());

    if (SamplingProfilerIntegration.isEnabled()) {
        ...
    }

    //清除vm内存增长上限，由于启动过程需要较多的虚拟机内存空间
    VMRuntime.getRuntime().clearGrowthLimit();

    //设置内存的可能有效使用率为0.8
    VMRuntime.getRuntime().setTargetHeapUtilization(0.8f);
    // 针对部分设备依赖于运行时就产生指纹信息，因此需要在开机完成前已经定义
    Build.ensureFingerprintProperty();

    //访问环境变量前，需要明确地指定用户
    Environment.setUserRequired(true);

    //确保当前系统进程的binder调用，总是运行在前台优先级(foreground priority)
    BinderInternal.disableBackgroundScheduling(true);
    android.os.Process.setThreadPriority(android.os.Process.THREAD_PRIORITY_FOREGROUND);
    android.os.Process.setCanSelfBackground(false);

    // 主线程looper就在当前线程运行
    Looper.prepareMainLooper();

    //加载android_servers.so库，该库包含的源码在frameworks/base/services/目录下
    System.loadLibrary("android_servers");

    //检测上次关机过程是否失败，该方法可能不会返回
    performPendingShutdown();

    //初始化系统上下文
    createSystemContext();

    //创建系统服务管理
    mSystemServiceManager = new SystemServiceManager(mSystemContext);
    //将mSystemServiceManager添加到本地服务的成员sLocalServiceObjects
    LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);


    //启动各种系统服务
    try {
        startBootstrapServices(); // 启动引导服务
        startCoreServices();      // 启动核心服务
        startOtherServices();     // 启动其他服务
    } catch (Throwable ex) {
        Slog.e("System", "************ Failure starting system services", ex);
        throw ex;
    }

    //用于debug版本，将log事件不断循环地输出到dropbox（用于分析）
    if (StrictMode.conditionallyEnableDebugLogging()) {
        Slog.i(TAG, "Enabled StrictMode for system server main thread.");
    }
    //一直循环执行
    Looper.loop();
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

LocalServices通过用静态Map变量sLocalServiceObjects，来保存以服务类名为key，以具体服务对象为value的Map结构。

#### 4.4.3、SystemServer.performPendingShutdown()

```java
private void performPendingShutdown() {
    final String shutdownAction = SystemProperties.get(
            ShutdownThread.SHUTDOWN_ACTION_PROPERTY, "");
    if (shutdownAction != null && shutdownAction.length() > 0) {
        boolean reboot = (shutdownAction.charAt(0) == '1');

        final String reason;
        if (shutdownAction.length() > 1) {
            reason = shutdownAction.substring(1, shutdownAction.length());
        } else {
            reason = null;
        }
        // 当"sys.shutdown.requested"值不为空,则会重启或者关机
        ShutdownThread.rebootOrShutdown(null, reboot, reason);
    }
}
```

#### 4.4.4、SystemServer.createSystemContext()

```java
private void createSystemContext() {
    //创建system_server进程的上下文信息
    ActivityThread activityThread = ActivityThread.systemMain();
    mSystemContext = activityThread.getSystemContext();
    //设置主题
    mSystemContext.setTheme(android.R.style.Theme_DeviceDefault_Light_DarkActionBar);
}
```

[理解Application创建过程](http://gityuan.com/2017/04/02/android-application/)已介绍过createSystemContext()过程， 该过程会创建对象有ActivityThread，Instrumentation, ContextImpl，LoadedApk，Application。

#### 4.4.5、SystemServer.startBootstrapServices()

```java
private void startBootstrapServices() {
    //阻塞等待与installd建立socket通道
    Installer installer = mSystemServiceManager.startService(Installer.class);

    //启动服务ActivityManagerService
    mActivityManagerService = mSystemServiceManager.startService(
            ActivityManagerService.Lifecycle.class).getService();
    mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
    mActivityManagerService.setInstaller(installer);

    //启动服务PowerManagerService
    mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);

    //初始化power management
    mActivityManagerService.initPowerManagement();

    //启动服务LightsService
    mSystemServiceManager.startService(LightsService.class);

    //启动服务DisplayManagerService
    mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);

    //Phase100: 在初始化package manager之前，需要默认的显示.
    mSystemServiceManager.startBootPhase(SystemService.PHASE_WAIT_FOR_DEFAULT_DISPLAY);

    //当设备正在加密时，仅运行核心
    String cryptState = SystemProperties.get("vold.decrypt");
    if (ENCRYPTING_STATE.equals(cryptState)) {
        mOnlyCore = true;
    } else if (ENCRYPTED_STATE.equals(cryptState)) {
        mOnlyCore = true;
    }

    //启动服务PackageManagerService
    mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
            mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
    mFirstBoot = mPackageManagerService.isFirstBoot();
    mPackageManager = mSystemContext.getPackageManager();

    //启动服务UserManagerService，新建目录/data/user/
    ServiceManager.addService(Context.USER_SERVICE, UserManagerService.getInstance());

    AttributeCache.init(mSystemContext);

    //设置AMS
    mActivityManagerService.setSystemProcess();

    //启动传感器服务
    startSensorService();
}
```

该方法所创建的服务：ActivityManagerService, PowerManagerService, LightsService, DisplayManagerService， PackageManagerService， UserManagerService， sensor服务.

#### 4.4.5、SystemServer.startCoreServices()

```
private void startCoreServices() {
    //启动服务BatteryService，用于统计电池电量，需要LightService.
    mSystemServiceManager.startService(BatteryService.class);

    //启动服务UsageStatsService，用于统计应用使用情况
    mSystemServiceManager.startService(UsageStatsService.class);
    mActivityManagerService.setUsageStatsManager(
            LocalServices.getService(UsageStatsManagerInternal.class));

    mPackageManagerService.getUsageStatsIfNoPackageUsageInfo();

    //启动服务WebViewUpdateService
    mSystemServiceManager.startService(WebViewUpdateService.class);
}
```

启动服务BatteryService，UsageStatsService，WebViewUpdateService。

#### 4.4.6、SystemServer.startOtherServices()

该方法比较长，有近千行代码，逻辑很简单，主要是启动一系列的服务，这里就不具体列举源码了，在第四节直接对其中的服务进行一个简单分类。

```java
 private void startOtherServices() {
        ...
        SystemConfig.getInstance();
        mContentResolver = context.getContentResolver(); // resolver
        ...
        mActivityManagerService.installSystemProviders(); //provider
        mSystemServiceManager.startService(AlarmManagerService.class); // alarm
        // watchdog
        watchdog.init(context, mActivityManagerService);
        inputManager = new InputManagerService(context); // input
        wm = WindowManagerService.main(...); // window
        inputManager.start();  //启动input
        mDisplayManagerService.windowManagerAndInputReady();
        ...
        mSystemServiceManager.startService(MOUNT_SERVICE_CLASS); // mount
        mPackageManagerService.performBootDexOpt();  // dexopt操作
        ActivityManagerNative.getDefault().showBootMessage(...); //显示启动界面
        ...
        statusBar = new StatusBarManagerService(context, wm); //statusBar
        //dropbox
        ServiceManager.addService(Context.DROPBOX_SERVICE,
                    new DropBoxManagerService(context, new File("/data/system/dropbox")));
         mSystemServiceManager.startService(JobSchedulerService.class); //JobScheduler
         lockSettings.systemReady(); //lockSettings

        //phase480 和phase500
        mSystemServiceManager.startBootPhase(SystemService.PHASE_LOCK_SETTINGS_READY);
        mSystemServiceManager.startBootPhase(SystemService.PHASE_SYSTEM_SERVICES_READY);
        ...
        // 准备好window, power, package, display服务
        wm.systemReady();
        mPowerManagerService.systemReady(...);
        mPackageManagerService.systemReady();
        mDisplayManagerService.systemReady(...);

        //重头戏
        mActivityManagerService.systemReady(new Runnable() {
            public void run() {
              ...
            }
        });
    }
```

#### 4.4.7、服务启动阶段

SystemServiceManager的startBootPhase()贯穿system_server进程的整个启动过程： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/android.boot/N13-android-system-start-system_server_boot_process.jpg)

其中PHASE_BOOT_COMPLETED=1000，该阶段是发生在Boot完成和home应用启动完毕。系统服务更倾向于监听该阶段，而不是注册广播ACTION_BOOT_COMPLETED，从而降低系统延迟。

各个启动阶段所在源码的大致位置：

```java
public final class SystemServer {

private void startBootstrapServices() {
  ...
  //phase100
  mSystemServiceManager.startBootPhase(SystemService.PHASE_WAIT_FOR_DEFAULT_DISPLAY);
  ...
}

private void startCoreServices() {
  ...
}

private void startOtherServices() {
  ...
  //phase480 && 500
  mSystemServiceManager.startBootPhase(SystemService.PHASE_LOCK_SETTINGS_READY);
  mSystemServiceManager.startBootPhase(SystemService.PHASE_SYSTEM_SERVICES_READY);

  ...
  mActivityManagerService.systemReady(new Runnable() {
     public void run() {
         //phase550
         mSystemServiceManager.startBootPhase(
                 SystemService.PHASE_ACTIVITY_MANAGER_READY);
         ...
         //phase600
         mSystemServiceManager.startBootPhase(
                 SystemService.PHASE_THIRD_PARTY_APPS_CAN_START);
      }
  }
}
}
```

接下来再说说简单每个阶段的大概完成的工作：

##### 4.4.7.1、Phase0

创建四大引导服务:

```java
ActivityManagerService
PowerManagerService
LightsService
DisplayManagerService
```

##### 4.4.7.1.2、Phase100

进入阶段PHASE_WAIT_FOR_DEFAULT_DISPLAY=100回调服务

```java
onBootPhase(100)

DisplayManagerService
```

然后创建大量服务下面列举部分:

```java
PackageManagerService
WindowManagerService
InputManagerService
NetworkManagerService
DropBoxManagerService
FingerprintService
LauncherAppsService
…
```

##### 4.4.7.1.3、Phase480

进入阶段PHASE_LOCK_SETTINGS_READY=480回调服务

```java
onBootPhase(480)

DevicePolicyManagerService
```

阶段480后马上就进入阶段500.

##### 4.4.7.1.4、Phase500

PHASE_SYSTEM_SERVICES_READY=500，进入该阶段服务能安全地调用核心系统服务.

```java
onBootPhase(500)

AlarmManagerService
JobSchedulerService
NotificationManagerService
BackupManagerService
UsageStatsService
DeviceIdleController
TrustManagerService
UiModeManagerService
BluetoothService
BluetoothManagerService
EthernetService
WifiP2pService
WifiScanningService
WifiService
RttService
```

各大服务执行systemReady():

```java
WindowManagerService.systemReady():
PowerManagerService.systemReady():
PackageManagerService.systemReady():
DisplayManagerService.systemReady():
```

接下来就绪AMS.systemReady方法.

##### 4.4.7.1.5、Phase550

PHASE_ACTIVITY_MANAGER_READY=550， AMS.mSystemReady=true, 已准备就绪,进入该阶段服务能广播Intent;但是system_server主线程并没有就绪.

```java
onBootPhase(550)

MountService
TelecomLoaderService
UsbService
WebViewUpdateService
DockObserver
BatteryService
```

接下来执行: (AMS启动native crash监控, 加载WebView，启动SystemUI等),如下

```java
mActivityManagerService.startObservingNativeCrashes();
WebViewFactory.prepareWebViewInSystemServer();
startSystemUi(context);
networkScoreF.systemReady();
networkManagementF.systemReady();
networkStatsF.systemReady();
networkPolicyF.systemReady();
connectivityF.systemReady();
audioServiceF.systemReady();
Watchdog.getInstance().start();
```

##### 4.4.7.1.6、Phase600

PHASE_THIRD_PARTY_APPS_CAN_START=600

onBootPhase(600)

> JobSchedulerService
NotificationManagerService
BackupManagerService
AppWidgetService
GestureLauncherService
DreamManagerService
TrustManagerService
VoiceInteractionManagerService

接下来,各种服务的systemRunning过程:

WallpaperManagerService、InputMethodManagerService、LocationManagerService、CountryDetectorService、NetworkTimeUpdateService、CommonTimeManagementService、TextServicesManagerService、AssetAtlasService、InputManagerService、TelephonyRegistry、MediaRouterService、MmsServiceBroker这些服务依次执行其systemRunning()方法。

##### 4.4.7.1.7、Phase1000

在经过一系列流程，再调用AMS.finishBooting()时，则进入阶段Phase1000。

到此，系统服务启动阶段完成就绪，system_server进程启动完成则进入Looper.loop()状态，随时待命，等待消息队列MessageQueue中的消息到来，则马上进入执行状态。

#### 4.4.8、服务类别

system_server进程，从源码角度划分为引导服务、核心服务、其他服务3类。 以下这些系统服务的注册过程, 见Android系统服务的注册方式

引导服务(7个)：ActivityManagerService、PowerManagerService、LightsService、DisplayManagerService、PackageManagerService、UserManagerService、SensorService； 核心服务(3个)：BatteryService、UsageStatsService、WebViewUpdateService； 其他服务(70个+)：AlarmManagerService、VibratorService等。 合计总大约80个系统服务：

> ActivityManagerService PackageManagerService WindowManagerService PowerManagerService BatteryService BatteryStatsService DreamManagerService DropBoxManagerService SamplingProfilerService UsageStatsService DiskStatsService DeviceStorageMonitorService SchedulingPolicyService AlarmManagerService DeviceIdleController ThermalObserver JobSchedulerService AccessibilityManagerService DisplayManagerService LightsService GraphicsStatsService StatusBarManagerService NotificationManagerService WallpaperManagerService UiModeManagerService AppWidgetService LauncherAppsService TextServicesManagerService ContentService LockSettingsService InputMethodManagerService InputManagerService MountService FingerprintService TvInputManagerService DockObserver NetworkManagementService NetworkScoreService NetworkStatsService NetworkPolicyManagerService ConnectivityService BluetoothService WifiP2pService WifiService WifiScanningService AudioService MediaRouterService VoiceInteractionManagerService MediaProjectionManagerService MediaSessionService<br>
> DevicePolicyManagerService PrintManagerService BackupManagerService UserManagerService AccountManagerService TrustManagerService SensorService LocationManagerService VibratorService CountryDetectorService GestureLauncherService PersistentDataBlockService EthernetService WebViewUpdateService ClipboardService TelephonyRegistry TelecomLoaderService NsdService UpdateLockService SerialService SearchManagerService CommonTimeManagementService AssetAtlasService ConsumerIrService MidiServiceCameraService TwilightService RestrictionsManagerService MmsServiceBroker RttService UsbService

Service类别众多，其中表中加粗项是指博主挑选的较重要或者较常见的Service，并且在本博客中已经展开或者计划展开讲解的Service，当然如果有精力会讲解更多service，后续再更新。 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/android.boot/N15-android-system-start-AMS.jpg)

### （5）、启动ActivityManagerService

#### 4.5.1、概述

ActivityManagerService(AMS)是Android中最核心的服务，主要负责系统中四大组件的启动、切换、调度及应用程序的管理和调度等工作。

AMS通信结构如下图所示：
![Markdown](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/android.boot/N14-android-system-start-.png)

#### 4.5.2、SystemServer.startBootstrapServices()

```java
private void startBootstrapServices() {
...
//启动AMS服务
mActivityManagerService = mSystemServiceManager.startService(
        ActivityManagerService.Lifecycle.class).getService();

//设置AMS的系统服务管理器
mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
//设置AMS的APP安装器
mActivityManagerService.setInstaller(installer);
//初始化AMS相关的PMS
mActivityManagerService.initPowerManagement();
...

//设置SystemServer
mActivityManagerService.setSystemProcess();
}
```

#### 4.5.3、启动AMS服务

SystemServiceManager.startService(ActivityManagerService.Lifecycle.class) 功能主要：

创建ActivityManagerService.Lifecycle对象； 调用Lifecycle.onStart()方法。

#### 4.5.4、启动AMS服务

**4.5.4.1 AMS.Lifecycle** [-> ActivityManagerService.java]

```java
public static final class Lifecycle extends SystemService {
private final ActivityManagerService mService;

public Lifecycle(Context context) {
    super(context);
    //创建ActivityManagerService
    mService = new ActivityManagerService(context);
}

@Override
public void onStart() {
    mService.start();  
}

public ActivityManagerService getService() {
    return mService;
}
}
```

该过程：创建AMS内部类的Lifecycle，已经创建AMS对象，并调用AMS.start();

**4.5.4.2 AMS创建**

```java
public ActivityManagerService(Context systemContext) {
mContext = systemContext;
mFactoryTest = FactoryTest.getMode();//默认为FACTORY_TEST_OFF
mSystemThread = ActivityThread.currentActivityThread();

//创建名为"ActivityManager"的前台线程，并获取mHandler
mHandlerThread = new ServiceThread(TAG, android.os.Process.THREAD_PRIORITY_FOREGROUND, false);
mHandlerThread.start();

mHandler = new MainHandler(mHandlerThread.getLooper());

//通过UiThread类，创建名为"android.ui"的线程
mUiHandler = new UiHandler();

//前台广播接收器，在运行超过10s将放弃执行
mFgBroadcastQueue = new BroadcastQueue(this, mHandler,
        "foreground", BROADCAST_FG_TIMEOUT, false);
//后台广播接收器，在运行超过60s将放弃执行
mBgBroadcastQueue = new BroadcastQueue(this, mHandler,
        "background", BROADCAST_BG_TIMEOUT, true);
mBroadcastQueues[0] = mFgBroadcastQueue;
mBroadcastQueues[1] = mBgBroadcastQueue;

//创建ActiveServices，其中非低内存手机mMaxStartingBackground为8
mServices = new ActiveServices(this);
mProviderMap = new ProviderMap(this);

//创建目录/data/system
File dataDir = Environment.getDataDirectory();
File systemDir = new File(dataDir, "system");
systemDir.mkdirs();

//创建服务BatteryStatsService
mBatteryStatsService = new BatteryStatsService(systemDir, mHandler);
mBatteryStatsService.getActiveStatistics().readLocked();
...

//创建进程统计服务，信息保存在目录/data/system/procstats，
mProcessStats = new ProcessStatsService(this, new File(systemDir, "procstats"));

mAppOpsService = new AppOpsService(new File(systemDir, "appops.xml"), mHandler);
mGrantFile = new AtomicFile(new File(systemDir, "urigrants.xml"));

// User 0是第一个，也是唯一的一个开机过程中运行的用户
mStartedUsers.put(UserHandle.USER_OWNER, new UserState(UserHandle.OWNER, true));
mUserLru.add(UserHandle.USER_OWNER);
updateStartedUserArrayLocked();
...

//CPU使用情况的追踪器执行初始化
mProcessCpuTracker.init();
...
mRecentTasks = new RecentTasks(this);
// 创建ActivityStackSupervisor对象
mStackSupervisor = new ActivityStackSupervisor(this, mRecentTasks);
mTaskPersister = new TaskPersister(systemDir, mStackSupervisor, mRecentTasks);

//创建名为"CpuTracker"的线程
mProcessCpuThread = new Thread("CpuTracker") {
    public void run() {
      while (true) {
        try {
          try {
            synchronized(this) {
              final long now = SystemClock.uptimeMillis();
              long nextCpuDelay = (mLastCpuTime.get()+MONITOR_CPU_MAX_TIME)-now;
              long nextWriteDelay = (mLastWriteTime+BATTERY_STATS_TIME)-now;
              if (nextWriteDelay < nextCpuDelay) {
                  nextCpuDelay = nextWriteDelay;
              }
              if (nextCpuDelay > 0) {
                  mProcessCpuMutexFree.set(true);
                  this.wait(nextCpuDelay);
              }
            }
          } catch (InterruptedException e) {
          }
          updateCpuStatsNow(); //更新CPU状态
        } catch (Exception e) {
        }
      }
    }
};
...
}
```

**4.5.4.3、AMS的start函数**

```java
private void start() {
    //完成统计前的复位工作
    Process.removeAllProcessGroups();

    //开始监控进程的CPU使用情况
    mProcessCpuThread.start();

    //注册服务
    mBatteryStatsService.publish(mContext);
    mAppOpsService.publish(mContext);
    Slog.d("AppOps", "AppOpsService published");
    LocalServices.addService(ActivityManagerInternal.class, new LocalService());
```

AMS的start函数比较简单，主要是： 1、启动CPU监控线程。该线程将会开始统计不同进程使用CPU的情况。 2、发布一些服务，如BatteryStatsService、AppOpsService(权限管理相关)和本地实现的继承ActivityManagerInternal的服务。

#### **4.5.5 AMS.setSystemProcess()**

```java
public void setSystemProcess() {
    try {
        ServiceManager.addService(Context.ACTIVITY_SERVICE, this, true);
        ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats);
        ServiceManager.addService("meminfo", new MemBinder(this));
        ServiceManager.addService("gfxinfo", new GraphicsBinder(this));
        ServiceManager.addService("dbinfo", new DbBinder(this));
        if (MONITOR_CPU_USAGE) {
            ServiceManager.addService("cpuinfo", new CpuBinder(this));
        }
        ServiceManager.addService("permission", new PermissionController(this));
        ServiceManager.addService("processinfo", new ProcessInfoService(this));
        ApplicationInfo info = mContext.getPackageManager().getApplicationInfo(
                "android", STOCK_PM_FLAGS);

        mSystemThread.installSystemApplicationInfo(info, getClass().getClassLoader());
        synchronized (this) {
            //创建ProcessRecord对象
            ProcessRecord app = newProcessRecordLocked(info, info.processName, false, 0);
            app.persistent = true; //设置为persistent进程
            app.pid = MY_PID;
            app.maxAdj = ProcessList.SYSTEM_ADJ;
            app.makeActive(mSystemThread.getApplicationThread(), mProcessStats);
            synchronized (mPidsSelfLocked) {
                mPidsSelfLocked.put(app.pid, app);
            }
            updateLruProcessLocked(app, false, null);//维护进程lru
            updateOomAdjLocked(); //更新adj
        }
    } catch (PackageManager.NameNotFoundException e) {
        throw new RuntimeException("", e);
    }
}
```

该方法主要工作是注册各种服务。

**4.5.5.1 AT.installSystemApplicationInfo()**

```java
public void installSystemApplicationInfo(ApplicationInfo info, ClassLoader classLoader) {
    synchronized (this) {
        //
        getSystemContext().installSystemApplicationInfo(info, classLoader);
        //创建用于性能统计的Profiler对象
        mProfiler = new Profiler();
    }
}
```

该方法调用ContextImpl的nstallSystemApplicationInfo()方法，最终调用LoadedApk的installSystemApplicationInfo，加载名为"android"的package

**4.5.5.2 installSystemApplicationInfo()** [-> LoadedApk.java]

```java
void installSystemApplicationInfo(ApplicationInfo info, ClassLoader classLoader) {
    assert info.packageName.equals("android");
    mApplicationInfo = info; //将包名为"android"的应用信息保存到mApplicationInfo
    mClassLoader = classLoader;
}
```

#### **4.5.6 startOtherServices()**

```java
private void startOtherServices() {
  ...
  //安装系统Provider
  mActivityManagerService.installSystemProviders();
  ...

  //phase480 && 500
  mSystemServiceManager.startBootPhase(SystemService.PHASE_LOCK_SETTINGS_READY);
  mSystemServiceManager.startBootPhase(SystemService.PHASE_SYSTEM_SERVICES_READY);
  ...

  mActivityManagerService.systemReady(new Runnable() {
     public void run() {
         //phase550
         mSystemServiceManager.startBootPhase(
                 SystemService.PHASE_ACTIVITY_MANAGER_READY);
         ...
         //phase600
         mSystemServiceManager.startBootPhase(
                 SystemService.PHASE_THIRD_PARTY_APPS_CAN_START);
         ...
      }
  }
}
```

**4.5.6.1 AMS.installSystemProviders()**

```java
public final void installSystemProviders() {
    List<ProviderInfo> providers;
    synchronized (this) {
        ProcessRecord app = mProcessNames.get("system", Process.SYSTEM_UID);
        providers = generateApplicationProvidersLocked(app);
        if (providers != null) {
            for (int i=providers.size()-1; i>=0; i--) {
                ProviderInfo pi = (ProviderInfo)providers.get(i);
                //移除非系统的provider
                if ((pi.applicationInfo.flags&ApplicationInfo.FLAG_SYSTEM) == 0) {
                    providers.remove(i);
                }
            }
        }
    }
    if (providers != null) {
        //安装所有的系统provider
        mSystemThread.installSystemProviders(providers);
    }

    // 创建核心Settings Observer，用于监控Settings的改变。
    mCoreSettingsObserver = new CoreSettingsObserver(this);
}
```

#### 4.5.7、AMS.systemReady()

#### 4.5.7.1、阶段一

```java
public void systemReady(final Runnable goingCallback) {
    synchronized(this) {
        ..........
        //这一部分主要是调用一些关键服务SystemReady相关的函数，
        //进行一些等待AMS初始完，才能进行的工作

        // Make sure we have the current profile info, since it is needed for security checks.
        mUserController.onSystemReady();

        mRecentTasks.onSystemReadyLocked();
        mAppOpsService.systemReady();
        mSystemReady = true;
    }

    ArrayList<ProcessRecord> procsToKill = null;
    synchronized(mPidsSelfLocked) {
        //mPidsSelfLocked中保存当前正在运行的所有进程的信息
        for (int i=mPidsSelfLocked.size()-1; i>=0; i--) {
            ProcessRecord proc = mPidsSelfLocked.valueAt(i);

            //在AMS启动完成前，如果没有FLAG_PERSISTENT标志的进程已经启动了，
            //就将这个进程加入到procsToKill中
            if (!isAllowedWhileBooting(proc.info)){
                if (procsToKill == null) {
                    procsToKill = new ArrayList<ProcessRecord>();
                }
                procsToKill.add(proc);
            }
        }
    }

    synchronized(this) {
        //利用removeProcessLocked关闭procsToKill中的进程
        if (procsToKill != null) {
            for (int i=procsToKill.size()-1; i>=0; i--) {
                ProcessRecord proc = procsToKill.get(i);
                Slog.i(TAG, "Removing system update proc: " + proc);
                removeProcessLocked(proc, true, false, "system update done");
            }
        }

        // Now that we have cleaned up any update processes, we
        // are ready to start launching real processes and know that
        // we won't trample on them any more.

        //至此系统准备完毕
        mProcessesReady = true;
    }
    ............
    //根据数据库和资源文件，获取一些配置参数
    retrieveSettings();

    final int currentUserId;
    synchronized (this) {
        //得到当前的用户ID
        currentUserId = mUserController.getCurrentUserIdLocked();

        //读取urigrants.xml，为其中定义的ContentProvider配置对指定Uri数据的访问/修改权限
        //原生代码中，似乎没有urigrants.xml文件
        //实际使用的grant-uri-permission是分布式定义的
        readGrantedUriPermissionsLocked();
    }
    ..........
```

这一部分的工作主要是调用一些关键服务的初始化函数， 然后杀死那些没有FLAG_PERSISTENT却在AMS启动完成前已经存在的进程， 同时获取一些配置参数。 需要注意的是，由于只有Java进程才会向AMS注册，而一般的Native进程不会向AMS注册，因此此处杀死的进程是Java进程。

#### 4.5.7.2、阶段二

```java
//1、调用参数传入的runnable对象，SystemServer中有具体的定义
if (goingCallback != null) goingCallback.run();
..............
//调用所有系统服务的onStartUser接口
mSystemServiceManager.startUser(currentUserId);
.............
synchronized (this) {
    // Only start up encryption-aware persistent apps; once user is
    // unlocked we'll come back around and start unaware apps
    2、启动persistent为1的application所在的进程
    startPersistentApps(PackageManager.MATCH_DIRECT_BOOT_AWARE);

    // Start up initial activity.
    mBooting = true;

    // Enable home activity for system user, so that the system can always boot
    //当isSplitSystemUser返回true时，意味者system user和primary user是分离的
    //这里应该是让system user也有启动home activity的权限吧
    if (UserManager.isSplitSystemUser()) {
        ComponentName cName = new ComponentName(mContext, SystemUserHomeActivity.class);
        try {
            AppGlobals.getPackageManager().setComponentEnabledSetting(cName,
                    PackageManager.COMPONENT_ENABLED_STATE_ENABLED, 0,
                    UserHandle.USER_SYSTEM);
        } catch (RemoteException e) {
            throw e.rethrowAsRuntimeException();
        }
    }

    //3、启动Home
    startHomeActivityLocked(currentUserId, "systemReady");

    try {
        //发送消息，触发处理Uid错误的Application
        if (AppGlobals.getPackageManager().hasSystemUidErrors()) {
            ..........
            mUiHandler.obtainMessage(SHOW_UID_ERROR_UI_MSG).sendToTarget();
        }
    } catch (RemoteException e) {
    }
    //发送一些广播信息
    ............
    //这里暂时先不深入，等进一步了解Activity的启动过程后，再做了解
    mStackSupervisor.resumeFocusedStackTopActivityLocked();
    ............
}
.............
```

从部分代码来看，主要的工作就是通知一些服务可以进行systemReady相关的工作，并进行启动服务或应用进程的工作。

#### 2.1、调用回调接口

回调接口的具体内容定义与SystemServer.java中，其中会调用大量服务的onBootPhase函数、一些对象的systemReady函数或systemRunning函数。 此处，我们仅截取一些比较特别的内容：

```java
public void run() {
    ............
    try {
        //启动NativeCrashListener监听"/data/system/ndebugsocket"中的信息
        //实际上是监听debuggerd传入的信息
        mActivityManagerService.startObservingNativeCrashes();
    } catch (Throwable e) {
        reportWtf("observing native crashes", e);
    }
    ............
    try {
        //启动SystemUi
        startSystemUi(context);
    } catch (Throwable e) {
        reportWtf("starting System UI", e);
    }
    ............
    //这个以前分析过，启动Watchdog
    Watchdog.getInstance().start();
    ....................
}
```

回调接口中的内容较多，不做一一分析。

#### 2.2、启动persistent标志的进程

我们看看startPersistentApps对应的内容：

```java
private void startPersistentApps(int matchFlags) {
    .............

    synchronized (this) {
        try {
            //从PKMS中得到persistent为1的ApplicationInfo
            final List<ApplicationInfo> apps = AppGlobals.getPackageManager()
                    .getPersistentApplications(STOCK_PM_FLAGS | matchFlags).getList();
            for (ApplicationInfo app : apps) {
                //由于framework-res.apk已经由系统启动，所以此处不再启动它
                if (!"android".equals(app.packageName)) {
                    //addAppLocked中将启动application所在进程
                    addAppLocked(app, false, null /* ABI override */);
                }
            }
        } catch (RemoteException ex) {
        }
    }
}
```

跟进一下addAppLocked函数：

```java
final ProcessRecord addAppLocked(ApplicationInfo info, boolean isolated,
        String abiOverride) {
    //以下是取出或构造出ApplicationInfo对应的ProcessRecord
    ProcessRecord app;
    if (!isolated) {
        app = getProcessRecordLocked(info.processName, info.uid, true);
    } else {
        app = null;
    }

    if (app == null) {
        app = newProcessRecordLocked(info, null, isolated, 0);
        updateLruProcessLocked(app, false, null);
        updateOomAdjLocked();
    }
    ...........
    // This package really, really can not be stopped.
    try {
        //通过PKMS将package对应数据结构的StoppedState置为fasle
        AppGlobals.getPackageManager().setPackageStoppedState(
                info.packageName, false, UserHandle.getUserId(app.uid));
    } catch (RemoteException e) {
    } catch (IllegalArgumentException e) {
        Slog.w(TAG, "Failed trying to unstop package "
                + info.packageName + ": " + e);
    }

    if ((info.flags & PERSISTENT_MASK) == PERSISTENT_MASK) {
        app.persistent = true;
        app.maxAdj = ProcessList.PERSISTENT_PROC_ADJ;
    }

    if (app.thread == null && mPersistentStartingProcesses.indexOf(app) < 0) {
        mPersistentStartingProcesses.add(app);
        //启动应用所在进程，将发送消息给zygote，后者fork出进程
        startProcessLocked(app, "added application", app.processName, abiOverride,
                null /* entryPoint */, null /* entryPointArgs */);
    }

    return app;
}
```

这里最终将通过startProcessLocked函数，启动实际的应用进程。 正如之前分析zygote进程时，提过的一样，zygote中的server socket将接收消息，然后为应用fork出进程。

#### 总结

对于整个AMS启动过程而言，博客中涉及的内容可能只是极小的一部分。 但即使我们尽可能的简化，整个过程的内容还是非常多。

![Markdown](https://raw.githubusercontent.com/zhoujinjianmax/zhoujinjian.com.images/master/android.boot/N16-android-system-start-flow.jpg)
不过我们回头看看整个过程，还是能比较清晰地将AMS的启动过程分为四步，如上图所示： 1、创建出SystemServer进程的Android运行环境。 在这一部分，SystemServer进程主要创建出对应的ActivityThread和ContextImpl，构成Android运行环境。 AMS的后续工作依赖于SystemServer在此创建出的运行环境。

2、完成AMS的初始化和启动。 在这一部分，单纯地调用AMS的构造函数和start函数，完成AMS的一些初始化工作。

3、将SystemServer进程纳入到AMS的管理体系中。 AMS作为Java世界的进程管理和调度中心，要对所有Java进程一视同仁，因此SystemServer进程也必须被AMS管理。 在这个过程中，AMS加载了SystemServer中framework-res.apk的信息，并启动和注册了SettingsProvider.apk。

4、开始执行AMS启动完毕后才能进行的工作。 系统中的一些服务和进程，必须等待AMS完成启动后，才能展开后续工作。 在这一部分，AMS通过调用systemReady函数，通知系统中的其它服务和进程，可以进行对应工作了。 在这个过程中，值得我们关注的是：Home Activity被启动了。当该Activity被加载完成后，最终会触发ACTION_BOOT_COMPLETED广播。

#### （6）、启动Launcher(Activity)

看看启动Home Activity对应的startHomeActivityLocked函数：

```java
boolean startHomeActivityLocked(int userId, String reason) {
    ..............
    Intent intent = getHomeIntent();
    //根据intent中携带的ComponentName，利用PKMS得到ActivityInfo
    ActivityInfo aInfo = resolveActivityInfo(intent, STOCK_PM_FLAGS, userId);
    if (aInfo != null) {
        intent.setComponent(new ComponentName(aInfo.applicationInfo.packageName, aInfo.name));
        aInfo = new ActivityInfo(aInfo);
        aInfo.applicationInfo = getAppInfoForUser(aInfo.applicationInfo, userId);

        //此时home对应进程应该还没启动，app为null
        ProcessRecord app = getProcessRecordLocked(aInfo.processName,
                aInfo.applicationInfo.uid, true);
        if (app == null || app.instrumentationClass == null) {
            intent.setFlags(intent.getFlags() | Intent.FLAG_ACTIVITY_NEW_TASK);
            //启动home
            mActivityStarter.startHomeActivityLocked(intent, aInfo, reason);
        }
    } else {
        ..........
    }
    return true;
}
```

这里暂时先不深究Home Activity启动的具体过程。 从手头的资料来看，当Home Activity启动后， ActivityStackSupervisor中的activityIdleInternalLocked函数将被调用：

```java
final ActivityRecord activityIdleInternalLocked(final IBinder token, boolean fromTimeout,
        Configuration config) {
    ...........
    if (isFocusedStack(r.task.stack) || fromTimeout) {
        booting = checkFinishBootingLocked();
    }
    ............
}
```

在checkFinishBootingLocked函数中：

```java
private boolean checkFinishBootingLocked() {
    //mService为AMS，mBooting变量在AMS回调SystemServer中定义的Runnable时，置为了true
    final boolean booting = mService.mBooting;
    boolean enableScreen = false;
    mService.mBooting = false;
    if (!mService.mBooted) {
        mService.mBooted = true;
        enableScreen = true;
    }
    if (booting || enableScreen) {、
        //调用AMS的接口，发送消息
        mService.postFinishBooting(booting, enableScreen);
    }
    return booting;
}
```

最终，AMS的finishBooting函数将被调用：

```java
final void finishBooting() {
    .........
    //以下是注册广播接收器，用于处理需要重启的package
    IntentFilter pkgFilter = new IntentFilter();
    pkgFilter.addAction(Intent.ACTION_QUERY_PACKAGE_RESTART);
    pkgFilter.addDataScheme("package");
    mContext.registerReceiver(new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            String[] pkgs = intent.getStringArrayExtra(Intent.EXTRA_PACKAGES);
            if (pkgs != null) {
                for (String pkg : pkgs) {
                    synchronized (ActivityManagerService.this) {
                        if (forceStopPackageLocked(pkg, -1, false, false, false, false, false,
                                0, "query restart")) {
                            setResultCode(Activity.RESULT_OK);
                            return;
                        }
                    }
                }
            }
       }
    }, pkgFilter);
    ...........
    // Let system services know.
    mSystemServiceManager.startBootPhase(SystemService.PHASE_BOOT_COMPLETED);

    //以下是启动那些等待启动的进程
    synchronized (this) {
        // Ensure that any processes we had put on hold are now started
        // up.
        final int NP = mProcessesOnHold.size();
            if (NP > 0) {
                ArrayList<ProcessRecord> procs =
                        new ArrayList<ProcessRecord>(mProcessesOnHold);
                for (int ip=0; ip<NP; ip++) {
                    .................
                    startProcessLocked(procs.get(ip), "on-hold", null);
                }
            }
        }
    }
    ..............
    if (mFactoryTest != FactoryTest.FACTORY_TEST_LOW_LEVEL) {
        // Start looking for apps that are abusing wake locks.
        //每15min检查一次系统各应用进程使用电量的情况，如果某个进程使用WakeLock的时间过长
        //AMS将关闭该进程
        Message nmsg = mHandler.obtainMessage(CHECK_EXCESSIVE_WAKE_LOCKS_MSG);
        mHandler.sendMessageDelayed(nmsg, POWER_CHECK_DELAY);

        // Tell anyone interested that we are done booting!
        SystemProperties.set("sys.boot_completed", "1");
        .................
        //此处从代码来看发送的是ACTION_LOCKED_BOOT_COMPLETED广播
        //在进行unlock相关的工作后，mUserController将调用finishUserUnlocking，发送SYSTEM_USER_UNLOCK_MSG消息给AMS
        //AMS收到消息后，调用mUserController的finishUserUnlocked函数，经过相应的处理后，
        //在mUserController的finishUserUnlockedCompleted中，最终将会发送ACTION_BOOT_COMPLETED广播
        mUserController.sendBootCompletedLocked(.........);
        .................
    }
}
```

最终，当AMS启动Home Activity结束，并发送ACTION_BOOT_COMPLETED广播时，AMS的启动过程告一段落。

具体启动流程请参考：【Android 7.1.2 (Android N) Activity启动流程分析】

### 参考文档：

[Android 7.0 ActivityManagerService 1 - 10](http://blog.csdn.net/Gaugamela/article/category/6383486)
[Android Init进程源码分析(1) - jay_richard](http://blog.leanote.com/post/jay_richard/Android-Init%E8%BF%9B%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90)
[Android Init进程源码分析(2) - jay_richard](http://blog.leanote.com/post/jay_richard/Android-Init%E8%BF%9B%E7%A8%8B%E4%B9%8B%E8%A7%A3%E6%9E%90init.rc)
[Android Zygote进程分析 - jay_richard](http://blog.leanote.com/post/jay_richard/Android-Zygote%E8%BF%9B%E7%A8%8B%E5%88%86%E6%9E%90)
[图解Android - Zygote, System Server 启动分析 - 漫天尘沙 - 博客园](http://www.cnblogs.com/samchen2009/p/3294713.html)
[Android7.0 init进程源码分析 - ZhangJian的博客 - CSDN博客](http://blog.csdn.net/gaugamela/article/details/52133186)
[Android系统启动-SystemServer上篇 - Gityuan博客 | 袁辉辉博客](http://gityuan.com/2016/02/20/android-system-server/)
[Android系统启动-Init篇 - Gityuan博客 | 袁辉辉博客](http://gityuan.com/2016/02/05/android-init/)
[Android系统启动-zygote篇 - Gityuan博客 | 袁辉辉博客](http://gityuan.com/2016/02/13/android-zygote/)
[Android系统启动-SystemServer下篇 - Gityuan博客 | 袁辉辉博客](http://gityuan.com/2016/02/20/android-system-server-2/)
[ActivityManagerService启动过程 - Gityuan博客 | 袁辉辉博客](http://gityuan.com/2016/02/21/activity-manager-service/)
[Android Init进程源码分析 - 深入剖析Android系统 - CSDN博客](http://blog.csdn.net/yangwen123/article/details/9029959)
