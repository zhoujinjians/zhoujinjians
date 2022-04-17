---
title: Android N 基础（2）：Android 7.1.2 Android Binder 系统分析
cover: https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/hexo.themes/bing-wallpaper-2018.04.06.jpg
categories: 
  - Android
tags:
  - Android
toc: true
abbrlink: 20170908
date: 2017-09-08 09:25:00
---


Android Binder系统概述： Binder是Android系统中大量使用的IPC（Inter-process communication，进程间通讯）机制。无论是应用程序对系统服务的请求，还是应用程序自身提供对外服务，都需要使用到Binder。因此，Binder机制在Android系统中的地位非常重要，可以说，理解Binder是理解Android系统的绝对必要前提。

<!-- more -->

 > **framework/base/core/java/ (Java)**
 **framework/base/core/jni/ (JNI)**
 **framework/native/libs/binder (Native)**
 **framework/native/cmds/servicemanager/ (Native)**
 **kernel/drivers/staging/android (Driver)**

--------------------------------------------------------------------------------

> **Java framework**

**framework/base/core/java/android/os/**
● IInterface.java
● IBinder.java
● Parcel.java
● IServiceManager.java
● ServiceManager.java
● ServiceManagerNative.java
● Binder.java

**framework/base/core/jni/**
● android_os_Parcel.cpp
● AndroidRuntime.cpp
● android_util_Binder.cpp (核心类)

--------------------------------------------------------------------------------

> **Native framework**

**framework/native/libs/binder**<br>
● IServiceManager.cpp
● BpBinder.cpp
● Binder.cpp
● IPCThreadState.cpp (核心类)
● ProcessState.cpp (核心类)

**framework/native/include/binder/**
● IServiceManager.h
● IInterface.h

**framework/native/cmds/servicemanager/**
● bctest.c
● binder.h
● binder.c
● service_manager.c
● servicemanager.rc

--------------------------------------------------------------------------------

> **Kernel**

**kernel/drivers/staging/android/**

● binder.c
● binder.h


## 一、Android Binder系统C程序示例

### （1）、简述Binder跨进程机制

Android系统中，每个应用程序是由Android的Activity，Service，Broadcast，ContentProvider这四组件的中一个或多个组合而成，这四组件所涉及的多进程间的通信底层都是依赖于Binder IPC机制。

从进程角度来看IPC机制 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/01-Android-binder-binder_interprocess_communication.png)


![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/02-Android-binder-IPC-Binder.jpg)

现在Client进程需要访问Server进程中的服务，会经过以下步骤：
1、Server进程首先向ServiceManager注册服务（ServiceManager先于Server启动）
2、Client进程向ServiceManager查询服务得到一个句柄Handle（Server进程可能不止一个服务，用Handle区分是哪一个服务）
3、Client进程 封装数据Buffer通过Binder驱动发送给Server进程，Server进程取得数据后解析数据，使用Server进程的Handle服务对应的函数处理数据，处理完成后通过Binder驱动传输给Client进程

#### 1.1、Server进程向ServiceManager注册服务

ServiceManager是一个守护进程。它的main()函数源码如下：

> ServiceManager是如何启动的？ 这里简要介绍一下ServiceManager的启动方式。当Kernel启动加载完驱动之后，会启动Android的init进程，init进程会解析servicemanager.rc，进而启动servicemanager.rc中定义的守护进程。

```c
[->ServiceManager.c]

int main(int argc, char **argv)
{
    struct binder_state *bs;
    void *svcmgr = BINDER_SERVICE_MANAGER;

    bs = binder_open(128*1024);

    if (binder_become_context_manager(bs)) {
        ...
    }   

    svcmgr_handle = svcmgr;
    binder_loop(bs, svcmgr_handler);
    return 0;
}
```

```c
void binder_loop(struct binder_state *bs, binder_handler func)
{
    int res;
    struct binder_write_read bwr;
    unsigned readbuf[32];

    bwr.write_size = 0;
    bwr.write_consumed = 0;
    bwr.write_buffer = 0;

    // 告诉Kernel，ServiceManager进程进入了消息循环状态。
    readbuf[0] = BC_ENTER_LOOPER;
    binder_write(bs, readbuf, sizeof(unsigned));

    for (;;) {
        bwr.read_size = sizeof(readbuf);
        bwr.read_consumed = 0;
        bwr.read_buffer = (unsigned) readbuf;

        // 向Kernel中发送消息(先写后读)。
        // 先将消息传递给Kernel，然后再从Kernel读取消息反馈
        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);

        ...

        // 解析读取的消息反馈
        res = binder_parse(bs, 0, readbuf, bwr.read_consumed, func);
        ...
    }
}
```

binder_loop()主要工作： (1)、通过ioctl(,BINDER_WRITE_READ,)进入消息循环，休眠等待Client请求 (2)、当Client通过驱动请求服务时，binder驱动会唤醒ServiceManager，通过binder_parse()解析处理数据，回复信息

代码调用关系图： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/03-Android-binder-ServiceManager-main.jpg)

时序流程图： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/04-Android-binder-ServiceManager-main-flow.jpg)

main()主要进行了三项工作： (1) 、通过binder_open()打开"/dev/binder"文件，即打开Binder设备文件。 (2) 、调用binder_become_context_manager()，通过ioctl()告诉Binder驱动程序自己是Binder上下文管理者。 (3) 、调用binder_loop()进入消息循环，等待Client的请求。如果没有Client请求，则进入睡眠等待状态；当有Client请求时，就被唤醒，然后读取并处理Client请求。

#### 1.2、分析Android binder原生示例程序bctest.c：

```c
int main(int argc, char **argv)
{
    struct binder_state *bs;
    uint32_t svcmgr = BINDER_SERVICE_MANAGER;
    uint32_t handle;

    bs = binder_open(128*1024);
    ...
    while (argc > 0) {
            //svcmgr_lookup方法调用binder_call(bs, &msg, &reply, 0, SVC_MGR_ADD_SERVICE)
            handle = svcmgr_lookup(bs, svcmgr, "alt_svc_mgr");
            //svcmgr_publish方法调用binder_call(bs, &msg, &reply, 0, SVC_MGR_CHECK_SERVICE)
            svcmgr_publish(bs, svcmgr, argv[1], &token);
    }
    return 0;
}
```

#### 1.3、示例程序（bctest.c）注册服务、获取服务过程

注册服务的过程（bctest.c）: 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/06-Android-binder-ServiceManager-main-SM-Publish.png)

(1) 、bs = binder_open(128*1024) (2) 、binder_call(bs, &msg, &reply, 0, SVC_MGR_ADD_SERVICE) 参数说明： // msg含有服务的名字 // reply它会含有servicemanager回复的数据 // target为0表示servicemanager // code: 表示要调用servicemanager中的"addservice函数"

获取服务的过程（bctest.c）: 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/05-Android-binder-ServiceManager-LookUp.png)

(1) 、bs = binder_open(128*1024) (2) 、binder_call(bs, &msg, &reply, target, SVC_MGR_CHECK_SERVICE) 参数说明： // msg含有服务的名字 // reply它会含有servicemanager回复的数据, 表示提供服务的进程 // target为0表示servicemanager // code: 表示要调用servicemanager中的"getservice函数"

binder_call远程实现： 根据msg、target、code就知道需要调用哪个服务的哪一个函数。 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/07-Android-binder-Binder_call.png)

```c
int binder_call(struct binder_state *bs,
                struct binder_io *msg, struct binder_io *reply,
                uint32_t target, uint32_t code)
{
    int res;
    struct binder_write_read bwr;
    struct {
        uint32_t cmd;
        struct binder_transaction_data txn;
    } __attribute__((packed)) writebuf;
    unsigned readbuf[32];

    writebuf.cmd = BC_TRANSACTION;
    writebuf.txn.target.handle = target;
    writebuf.txn.code = code;
    writebuf.txn.flags = 0;
    writebuf.txn.data_size = msg->data - msg->data0;
    writebuf.txn.offsets_size = ((char*) msg->offs) - ((char*) msg->offs0);
    writebuf.txn.data.ptr.buffer = (uintptr_t)msg->data0;
    writebuf.txn.data.ptr.offsets = (uintptr_t)msg->offs0;

    bwr.write_size = sizeof(writebuf);
    bwr.write_consumed = 0;
    bwr.write_buffer = (uintptr_t) &writebuf;

    hexdump(msg->data0, msg->data - msg->data0);
    for (;;) {
        bwr.read_size = sizeof(readbuf);
        bwr.read_consumed = 0;
        bwr.read_buffer = (uintptr_t) readbuf;

        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);

        res = binder_parse(bs, reply, (uintptr_t) readbuf, bwr.read_consumed, 0);

    }
}
```

> 注： 结构体简介 binder_io 封装一次发送的数据 binder_write_read 存储一次读写操作的数据 binder_transaction_data 存储一次事务的数据 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/08-Android-binder-binder_io_struct.png)
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/09-Android-binder-binder_write_read.png)
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/10-Android-binder-binder_transaction_data.jpg)

（1）构造参数，使用binder_io 描述
（2）数据转换binder_io -> binder_write_read；首先根据binder_io 、target、code三者构造binder_transaction_data，然后将binder_write_read.write_buffer指向binder_transaction_data
（3）调用ioctl(bs->fd, BINDER_WRITE_READ, &bwr);发送数据 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/11-Android-Binder-binder_call.png)

### （2）、Android Binder系统_ServiceManager

我们先跳过ioctl(bs->fd, BINDER_WRITE_READ, &bwr)所涉及的内核知识和流程，稍后再Android Binder系统-Driver层详细介绍。

#### 2.1、ServiceManager中service句柄如何管理

前面分析过，ServiceManager开机初始会启动成为一个守护进程， ServiceManager是如何管理service句柄的？ 进程里有一个全局性的svclist变量：

```c
struct svcinfo *svclist = 0;
```

它记录着所有添加进系统的"Service"信息，这些信息被组织成一条单向链表，我们不妨称这条链表为"Service向量表"。示意图如下： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/12-Android-Binder-SM-svclist.jpg)

链表节点类型为svcinfo

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/13-Android-Binder-SM-svcinfo.png)

添加服务简单理解就是 新建svcinfo节点插入到单链表中，查询服务就是看单链表是否有此服务。

#### 2.2、解析Binder上传数据-(binder_parse函数)

回到ServiceManager的main()函数。binder_loop()会先向binder驱动发出了BC_ENTER_LOOPER命令，接着进入一个for循环不断调用ioctl()读取发来的数据，接着解析这些数据。假设现在Client有请求，Binder驱动就通过会上传数据。读取数据后会交由binder_parse()解析。

```c
binder_loop(bs, svcmgr_handler);
```

注意binder_loop()的参数svcmgr_handler()函数指针。而且这个参数会进一步传递给binder_parse()。binder_parse()负责解析从binder驱动读来的数据，其代码截选如下：

```c
int binder_parse(struct binder_state *bs, struct binder_io *bio,
                 uintptr_t ptr, size_t size, binder_handler func)
{
    int r = 1;
    uintptr_t end = ptr + (uintptr_t) size;

    while (ptr < end) {
        uint32_t cmd = *(uint32_t *) ptr;
        ptr += sizeof(uint32_t);
        switch(cmd) {
        ...
        //驱动有数据后会返回次cmd
        case BR_TRANSACTION: {
            struct binder_transaction_data *txn = (struct binder_transaction_data *) ptr;
            binder_dump_txn(txn);
            if (func) {
                unsigned rdata[256/4];
                struct binder_io msg;
                struct binder_io reply;
                int res;

                bio_init(&reply, rdata, sizeof(rdata), 4);
                bio_init_from_txn(&msg, txn);
                res = func(bs, txn, &msg, &reply);
                if (txn->flags & TF_ONE_WAY) {
                    binder_free_buffer(bs, txn->data.ptr.buffer);
                } else {
                    binder_send_reply(bs, &reply, txn->data.ptr.buffer, res);
                }              
                ...
            }
            ptr += sizeof(*txn);
            break;
        }
        ...
    }
    return r;
}
```

从前文的代码我们可以看到，binder_loop()声明了一个128节的buffer（即uint32_t readbuf[32]），每次用BINDER_WRITE_READ命令从驱动读取一些内容，并传入binder_parse()。 binder_parse()在合适的时机，会回调其func参数（binder_handler func）指代的回调函数，即前文说到的svcmgr_handler()函数。

binder_loop()就这样一直循环下去，完成了整个ServiceManager的工作。

#### 2.3、数据转换binder_transaction_data->binder_io

初始化reply；根据txt(Binder驱动反馈的信息)初始化msg

```c
bio_init(&reply, rdata, sizeof(rdata), 4);
bio_init_from_txn(&msg, txn);
```

#### 2.4、如何添加服务SVC_MGR_ADD_SERVICE

前面讲过 binder_parse()在合适的时机，会回调其func参数（binder_handler func）指代的回调函数，即前文说到的svcmgr_handler()函数。并且会根据binder_transaction_data的code判断具体调用哪一个函数。

```c
int svcmgr_handler(struct binder_state *bs,
                   struct binder_transaction_data *txn,
                   struct binder_io *msg,
                   struct binder_io *reply)
{
    struct svcinfo *si;
    uint16_t *s;
    size_t len;
    uint32_t handle;
    uint32_t strict_policy;
    int allow_isolated;

    ......
    switch(txn->code) {
    case SVC_MGR_GET_SERVICE:
    case SVC_MGR_CHECK_SERVICE:
        s = bio_get_string16(msg, &len);
        handle = do_find_service(s, len, txn->sender_euid, txn->sender_pid);
        bio_put_ref(reply, handle);
        return 0;

    case SVC_MGR_ADD_SERVICE:
        s = bio_get_string16(msg, &len);
        handle = bio_get_ref(msg);
        allow_isolated = bio_get_uint32(msg) ? 1 : 0;
        if (do_add_service(bs, s, len, handle, txn->sender_euid,
            allow_isolated, txn->sender_pid))
            return -1;
        break;
...
    }
    bio_put_uint32(reply, 0);
    return 0;
}
```

由代码可知code = SVC_MGR_ADD_SERVICE 会调用do_add_service()函数

```c
int do_add_service(struct binder_state *bs,
                   const uint16_t *s, size_t len,
                   uint32_t handle, uid_t uid, int allow_isolated,
                   pid_t spid)
{
    struct svcinfo *si;
    ...
    si = find_svc(s, len);
    if (si) {
        ...
    } else {
        si = malloc(sizeof(*si) + (len + 1) * sizeof(uint16_t));
        ...
        si->handle = handle;
        si->len = len;
        memcpy(si->name, s, (len + 1) * sizeof(uint16_t));
        si->name[len] = '\0';
        si->death.func = (void*) svcinfo_death;
        si->death.ptr = si;
        si->allow_isolated = allow_isolated;
        si->next = svclist;
        svclist = si;
    }
    binder_acquire(bs, handle);
    binder_link_to_death(bs, handle, &si->death);
    return 0;
}
```

可见添加Service只是新建了一个svcinfo然后插入到前面所说的"Service向量表"中。

#### 2.5、如何获取服务SVC_MGR_CHECK_SERVICE

```c
uint32_t do_find_service(const uint16_t *s, size_t len, uid_t uid, pid_t spid)
{
    struct svcinfo *si = find_svc(s, len);
    ...
    return si->handle;
}
```

获取服务会查询"Service向量表"是否有此服务，然后返回Service的句柄handle。

#### 2.6、ServiceManager回复数据

前面分析回调svcmgr_handler()函数处理数据后，会调用binder_send_reply()函数 回复消息给驱动。

```c
binder_send_reply(bs, &reply, txn->data.ptr.buffer, res)
```

#### 2.7、总结：

示例程序（bctest.c）注册、获取服务一般分以下步骤： （1）源进程通过binder_open()打开"/dev/binder"文件，即打开Binder设备文件。 （2）源进程构造数据：[a].构造binder_io [b].转为binder_transaction_data [c].放入binder_write_read （3）源进程调用ioctl(bs->fd, BINDER_WRITE_READ, &bwr);发送数据给驱动 （4）驱动上报数据到目的进程ServiceManager （5）目的进程ServiceManager处理完数据，重新构造数据，通过调用ioctl(bs->fd, BINDER_WRITE_READ, &bwr);发送数据给驱动 （6）驱动然后将数据反馈到源进程

### （3）、Android Binder系统C程序

#### 3.1、Android Binder系统C程序_框架

总结bctest.c注册服务获取服务的一般流程框架： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/14-Android-Binder-binder_C_app_client_server_Arc.png)

#### 3.2、Android Binder系统C程序_编码

参考bctest.c编码： test_server：向ServiceManager添加服务"hello" && "goodbye" Service test_client ：查询获取服务(ServiceManager) [链接：Binder_C_App](https://github.com/weidongshan/APP_0003_Binder_C_App) 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/15-Android-binder-C-Test-App.png)

#### 3.3、Android Binder系统C程序_测试

./test_server & ./test_client hello ./test_client hello 100ask.taobao.com ./test_client goodbye ./test_client goodbye 100ask.taobao.com

## 二、Android Binder系统-Driver层

前面打开驱动binder_open(128*1024)、ServiceManager启动是如何与驱动交互成为管理者的，以及添加服务获取服务 驱动部分都没有详细讲解，现在一起来看下。

### （1）、Binder驱动概述

#### 1.1 概述

Binder驱动是Android专用的，但底层的驱动架构与Linux驱动一样。binder驱动在以misc设备进行注册，作为虚拟字符设备，没有直接操作硬件，只是对设备内存的处理。主要是驱动设备的初始化(binder_init)，打开 (binder_open)，映射(binder_mmap)，数据操作(binder_ioctl)。如启动ServiceManager调用: 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/16-Android-Binder-start_service_manager.jpg)

#### 1.2 系统调用

用户态的程序调用Kernel层驱动是需要陷入内核态，进行系统调用(syscall)，比如打开Binder驱动方法的调用链为： open-> **open() -> binder_open()。 open()为用户空间的方法，**open()便是系统调用中相应的处理方法，通过查找，对应调用到内核binder驱动的binder_open()方法，至于其他的从用户态陷入内核态的流程也基本一致。 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/17-Android-binder_driver_interface.png)

### （2）、Binder核心方法

#### 2.1、binder_init()

主要工作是为了注册misc设备 binder_init函数中最主要的工作其实下面这行：

```c
ret = misc_register(&binder_miscdev);
```

该行代码真正向内核中注册了Binder设备。binder_miscdev的定义如下：

```c
static struct miscdevice binder_miscdev = {
    .minor = MISC_DYNAMIC_MINOR,
    .name = "binder",
    .fops = &binder_fops
};
```

这里指定了Binder设备的名称是"binder"。这样，在用户空间便可以通过对/dev/binder文件进行操作来使用Binder。 binder_miscdev同时也指定了该设备的fops。fops是另外一个结构体，这个结构中包含了一系列的函数指针，其定义如下：

```c
static const struct file_operations binder_fops = {
    .owner = THIS_MODULE,
    .poll = binder_poll,
    .unlocked_ioctl = binder_ioctl,
    .compat_ioctl = binder_ioctl,
    .mmap = binder_mmap,
    .open = binder_open,
    .flush = binder_flush,
    .release = binder_release,
};
```

#### 2.2、**主要结构**

Binder驱动中包含了很多的结构体。为了便于下文讲解，这里我们先对这些结构体做一些介绍。

驱动中的结构体可以分为两类：

一类是与用户空间共用的，这些结构体在Binder通信协议过程中会用到。因此，这些结构体定义在binder.h中，包括：

结构体名称                       | 说明
:-------------------------- | --------------------------
flat_binder_object          | 描述在Binder IPC中传递的对象，见下文
**binder_write_read**       | 存储一次读写操作的数据
binder_version              | 存储Binder的版本号
transaction_flags           | 描述事务的flag，例如是否是异步请求，是否支持fd
**binder_transaction_data** | 存储一次事务的数据
binder_ptr_cookie           | 包含了一个指针和一个cookie
binder_handle_cookie        | 包含了一个句柄和一个cookie
binder_pri_desc             | 暂未用到
binder_pri_ptr_cookie       | 暂未用到

从前面Binder系统C程序框架分析，这其中，**binder_write_read**和**binder_transaction_data**这两个结构体最为重要，它们存储了IPC调用过程中的数据。关于这一点，我们在下文中会讲解。

Binder驱动中，还有一类结构体是仅仅Binder驱动内部实现过程中需要的，它们定义在binder.c中，包括：

结构体名称                        | 说明
:--------------------------- | --------------------------
**binder_node**              | 描述Binder实体节点，即：对应了一个Server
**binder_ref**               | 描述对于Binder实体的引用
**binder_buffer**            | 描述Binder通信过程中存储数据的Buffer
**binder_proc**              | 描述使用Binder的进程
**binder_thread**            | 描述使用Binder的线程
binder_work                  | 描述通信过程中的一项任务
binder_transaction           | 描述一次事务的相关信息
binder_deferred_state        | 描述延迟任务
binder_ref_death             | 描述Binder实体死亡的信息
binder_transaction_log       | debugfs日志
binder_transaction_log_entry | debugfs日志条目

这里需要读者关注的结构体已经用加粗做了标注。

#### 2.3、**Binder协议**

Binder协议可以分为控制协议和驱动协议两类。

控制协议是进程通过ioctl("/dev/binder") 与Binder设备进行通讯的协议，该协议包含以下几种命令：

结构体名称                    | 说明                              |       参数类型
:----------------------- | ------------------------------- | :---------------:
**BINDER_WRITE_READ**    | 读写操作，最常用的命令。IPC过程就是通过这个命令进行数据传递 | binder_write_read
BINDER_SET_MAX_THREADS   | 设置进程支持的最大线程数量                   |      size_t
BINDER_SET_CONTEXT_MGR   | 设置自身为ServiceManager             |         无
BINDER_THREAD_EXIT       | 通知驱动Binder线程退出                  |         无
BINDER_VERSION           | 获取Binder驱动的版本号                  |  binder_version
BINDER_SET_IDLE_PRIORITY | 暂未用到                            |         -
BINDER_SET_IDLE_TIMEOUT  | 暂未用到                            |         -

Binder的驱动协议描述了对于Binder驱动的具体使用过程。驱动协议又可以分为两类：

一类是binder_driver_command_protocol，描述了进程发送给Binder驱动的命令 一类是binder_driver_return_protocol，描述了Binder驱动发送给进程的命令 binder_driver_command_protocol共包含17个命令，分别是：

结构体名称                         | 说明                           |          参数类型
:---------------------------- | ---------------------------- | :---------------------:
BC_TRANSACTION                | Binder事务，即：Client对于Server的请求 | binder_transaction_data
BC_REPLY                      | 事务的应答，即：Server对于Client的回复    | binder_transaction_data
BC_FREE_BUFFER                | 通知驱动释放Buffer                 |    binder_uintptr_t
BC_ACQUIRE                    | 强引用计数+1                      |          __u32
BC_RELEASE                    | 强引用计数-1                      |          __u32
BC_INCREFS                    | 弱引用计数+1                      |          __u32
BC_DECREFS                    | 弱引用计数-1 __u32
BC_ACQUIRE_DONE               | BR_ACQUIRE的回复                |    binder_ptr_cookie
BC_INCREFS_DONE               | BR_INCREFS的回复                |    binder_ptr_cookie
BC_ENTER_LOOPER               | 通知驱动主线程ready                 |          void
BC_REGISTER_LOOPER            | 通知驱动子线程ready                 |          void
BC_EXIT_LOOPER                | 通知驱动线程已经退出                   |          void
BC_REQUEST_DEATH_NOTIFICATION | 请求接收死亡通知                     |  binder_handle_cookie
BC_CLEAR_DEATH_NOTIFICATION   | 去除接收死亡通知                     |  binder_handle_cookie
BC_DEAD_BINDER_DONE           | 已经处理完死亡通知                    |    binder_uintptr_t
BC_ATTEMPT_ACQUIRE            | 暂未实现                         |            -
BC_ACQUIRE_RESULT             | 暂未实现                         |            -

binder_driver_return_protocol共包含18个命令，分别是：

结构体名称                            | 说明                        |          参数类型
:------------------------------- | ------------------------- | :---------------------:
BR_OK                            | 操作完成                      |          void
BR_NOOP                          | 操作完成                      |          void
BR_ERROR                         | 发生错误                      |          __s32
BR_TRANSACTION                   | 通知进程收到一次Binder请求（Server端） | binder_transaction_data
BR_REPLY                         | 通知进程收到Binder请求的回复（Client） | binder_transaction_data
BR_TRANSACTION_COMPLETE          | 驱动对于接受请求的确认回复             |          void
BR_FAILED_REPLY                  | 告知发送方通信目标不存在              |          void
BR_SPAWN_LOOPER                  | 通知Binder进程创建一个新的线程        |          void
BR_ACQUIRE                       | 强引用计数+1请求                 |    binder_ptr_cookie
BR_RELEASE                       | 强引用计数-1请求                 |    binder_ptr_cookie
BR_INCREFS                       | 弱引用计数+1请求                 |    binder_ptr_cookie
BR_DECREFS                       | 若引用计数-1请求                 |    binder_ptr_cookie
BR_DEAD_BINDER                   | 发送死亡通知                    |    binder_uintptr_t
BR_CLEAR_DEATH_NOTIFICATION_DONE | 清理死亡通知完成                  |    binder_uintptr_t
BR_DEAD_REPLY                    | 告知发送方对方已经死亡               |          void
BR_ACQUIRE_RESULT                | 暂未实现                      |            -
BR_ATTEMPT_ACQUIRE               | 暂未实现                      |            -
BR_FINISHED                      | 暂未实现                      |            -

单独看上面的协议可能很难理解，这里我们以一次Binder请求过程来详细看一下Binder协议是如何通信的，就比较好理解了。

这幅图的说明如下： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/18-Android-binder_transaction_ipc.jpg)

Binder是C/S架构的，通信过程牵涉到：Client，Server以及Binder驱动三个角色 Client对于Server的请求以及Server对于Client回复都需要通过Binder驱动来中转数据 BC_XXX命令是进程发送给驱动的命令 BR_XXX命令是驱动发送给进程的命令 整个通信过程由Binder驱动控制

#### 2.4、binder_open()

任何进程在使用Binder之前，都需要先通过open("/dev/binder")打开Binder设备。上文已经提到，用户空间的open系统调用对应了驱动中的binder_open函数。在这个函数，Binder驱动会为调用的进程做一些初始化工作。binder_open函数代码如下所示：

```c
static int binder_open(struct inode *nodp, struct file *filp)
{
    struct binder_proc *proc;

   // 创建进程对应的binder_proc对象
    proc = kzalloc(sizeof(*proc), GFP_KERNEL);
    if (proc == NULL)
        return -ENOMEM;
    get_task_struct(current);
    proc->tsk = current;
    // 初始化binder_proc
    INIT_LIST_HEAD(&proc->todo);
    init_waitqueue_head(&proc->wait);
    proc->default_priority = task_nice(current);

    // 锁保护
    binder_lock(__func__);

    binder_stats_created(BINDER_STAT_PROC);
    // 添加到全局列表binder_procs中
    hlist_add_head(&proc->proc_node, &binder_procs);
    proc->pid = current->group_leader->pid;
    INIT_LIST_HEAD(&proc->delivered_death);
    filp->private_data = proc;

    binder_unlock(__func__);

    return 0;
}
```

在Binder驱动中，通过binder_procs记录了所有使用Binder的进程。每个初次打开Binder设备的进程都会被添加到这个列表中的。

另外，请读者回顾一下上文介绍的Binder驱动中的几个关键结构体：

> binder_proc binder_node binder_thread binder_ref binder_buffer

在实现过程中，为了便于查找，这些结构体互相之间都留有字段存储关联的结构。

下面这幅图描述了这里说到的这些内容： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/19-Android-binder_main_struct.png)

#### 2.5、binder_mmap()

在打开Binder设备之后，进程还会通过mmap进行内存映射。mmap的作用有如下两个：

申请一块内存空间，用来接收Binder通信过程中的数据 对这块内存进行地址映射，以便将来访问 binder_mmap函数对应了mmap系统调用的处理，这个函数也是Binder驱动的精华所在（这里说的binder_mmap函数也包括其内部调用的binder_update_page_range函数，见下文）。

前文我们说到，使用Binder机制，数据只需要经历一次拷贝就可以了，其原理就在这个函数中。

binder_mmap这个函数中，会申请一块物理内存，然后在用户空间和内核空间同时对应到这块内存上。在这之后，当有Client要发送数据给Server的时候，只需一次，将Client发送过来的数据拷贝到Server端的内核空间指定的内存地址即可，由于这个内存地址在服务端已经同时映射到用户空间，因此无需再做一次复制，Server即可直接访问，整个过程如下图所示： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/20-Android-mmap_and_transaction.png)

这幅图的说明如下：

Server在启动之后，调用对/dev/binder设备调用mmap 内核中的binder_mmap函数进行对应的处理：申请一块物理内存，然后在用户空间和内核空间同时进行映射 Client通过BINDER_WRITE_READ命令发送请求，这个请求将先到驱动中，同时需要将数据从Client进程的用户空间拷贝到内核空间 驱动通过BR_TRANSACTION通知Server有人发出请求，Server进行处理。由于这块内存也在用户空间进行了映射，因此Server进程的代码可以直接访问 了解原理之后，我们再来看一下Binder驱动的相关源码。这段代码有两个函数：

binder_mmap函数对应了mmap的系统调用的处理 binder_update_page_range函数真正实现了内存分配和地址映射

```c
static int binder_mmap(struct file *filp, struct vm_area_struct *vma)
{
    int ret;

    struct vm_struct *area;
    struct binder_proc *proc = filp->private_data;
    const char *failure_string;
    struct binder_buffer *buffer;

    ...
   // 在内核空间获取一块地址范围
    area = get_vm_area(vma->vm_end - vma->vm_start, VM_IOREMAP);
    if (area == NULL) {
        ret = -ENOMEM;
        failure_string = "get_vm_area";
        goto err_get_vm_area_failed;
    }
    proc->buffer = area->addr;
    // 记录内核空间与用户空间的地址偏移
    proc->user_buffer_offset = vma->vm_start - (uintptr_t)proc->buffer;
    mutex_unlock(&binder_mmap_lock);

  ...
    proc->pages = kzalloc(sizeof(proc->pages[0]) * ((vma->vm_end - vma->vm_start) / PAGE_SIZE), GFP_KERNEL);
    if (proc->pages == NULL) {
        ret = -ENOMEM;
        failure_string = "alloc page array";
        goto err_alloc_pages_failed;
    }
    proc->buffer_size = vma->vm_end - vma->vm_start;

    vma->vm_ops = &binder_vm_ops;
    vma->vm_private_data = proc;

    /* binder_update_page_range assumes preemption is disabled */
    preempt_disable();
    // 通过下面这个函数真正完成内存的申请和地址的映射
    // 初次使用，先申请一个PAGE_SIZE大小的内存
    ret = binder_update_page_range(proc, 1, proc->buffer, proc->buffer + PAGE_SIZE, vma);
    ...
}

static int binder_update_page_range(struct binder_proc *proc, int allocate,
                    void *start, void *end,
                    struct vm_area_struct *vma)
{
    void *page_addr;
    unsigned long user_page_addr;
    struct vm_struct tmp_area;
    struct page **page;
    struct mm_struct *mm;

    ...

    for (page_addr = start; page_addr < end; page_addr += PAGE_SIZE) {
        int ret;
        struct page **page_array_ptr;
        page = &proc->pages[(page_addr - proc->buffer) / PAGE_SIZE];

        BUG_ON(*page);
        // 真正进行内存的分配
        *page = alloc_page(GFP_KERNEL | __GFP_HIGHMEM | __GFP_ZERO);
        if (*page == NULL) {
            pr_err("%d: binder_alloc_buf failed for page at %p\n",
                proc->pid, page_addr);
            goto err_alloc_page_failed;
        }
        tmp_area.addr = page_addr;
        tmp_area.size = PAGE_SIZE + PAGE_SIZE /* guard page? */;
        page_array_ptr = page;
        // 在内核空间进行内存映射
        ret = map_vm_area(&tmp_area, PAGE_KERNEL, &page_array_ptr);
        if (ret) {
            pr_err("%d: binder_alloc_buf failed to map page at %p in kernel\n",
                   proc->pid, page_addr);
            goto err_map_kernel_failed;
        }
        user_page_addr =
            (uintptr_t)page_addr + proc->user_buffer_offset;
        // 在用户空间进行内存映射
        ret = vm_insert_page(vma, user_page_addr, page[0]);
        if (ret) {
            pr_err("%d: binder_alloc_buf failed to map page at %lx in userspace\n",
                   proc->pid, user_page_addr);
            goto err_vm_insert_page_failed;
        }
        /* vm_insert_page does not seem to increment the refcount */
    }
    if (mm) {
        up_write(&mm->mmap_sem);
        mmput(mm);
    }

    preempt_disable();

    return 0;
...
```

binder_update_page_range主要完成工作：分配物理空间，将物理空间映射到内核空间，将物理空间映射到进程空间. 另外，不同参数下该方法也可以释放物理页面。 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/21-Android-binder_mmap.png)

#### 2.6、binder_ioctl()内存管理

上文中，我们看到binder_mmap的时候，会申请一个PAGE_SIZE(通常是4K)的内存。而实际使用过程中，一个PAGE_SIZE的大小通常是不够的。

在驱动中，会根据实际的使用情况进行内存的分配。有内存的分配，当然也需要内存的释放。这里我们就来看看Binder驱动中是如何进行内存的管理的。

首先，我们还是从一次IPC请求说起。

当一个Client想要对Server发出请求时，它首先将请求发送到Binder设备上，由Binder驱动根据请求的信息找到对应的目标节点，然后将请求数据传递过去。

进程通过ioctl系统调用来发出请求：ioctl(bs->fd, BINDER_WRITE_READ, &bwr)

这里的bs->fd对应了打开Binder设备时的fd。BINDER_WRITE_READ对应了具体要做的操作码，这个操作码将由Binder驱动解析。bwr存储了请求数据，其类型是binder_write_read。

binder_write_read其实是一个相对外层的数据结构，其内部会包含一个binder_transaction_data结构的数据。binder_transaction_data包含了发出请求者的标识，请求的目标对象以及请求所需要的参数。它们的关系如下图所示： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/22-Android-binder_write_read.png)

binder_ioctl函数对应了ioctl系统调用的处理。这个函数的逻辑比较简单，就是根据ioctl的命令来确定进一步处理的逻辑，具体如下:

● 如果命令是BINDER_WRITE_READ，并且
● 如果 bwr.write_size > 0，则调用binder_thread_write
● 如果 bwr.read_size > 0，则调用binder_thread_read
● 如果命令是BINDER_SET_MAX_THREADS，则设置进程的max_threads，即进程支持的最大线程数
● 如果命令是BINDER_SET_CONTEXT_MGR，则设置当前进程为ServiceManager，见下文
● 如果命令是BINDER_THREAD_EXIT，则调用binder_free_thread，释放binder_thread
● 如果命令是BINDER_VERSION，则返回当前的Binder版本号 这其中，最关键的就是binder_thread_write方法。当Client请求Server的时候，便会发送一个BINDER_WRITE_READ命令，同时框架会将将实际的数据包装好。此时，binder_transaction_data中的code将是BC_TRANSACTION，由此便会调用到binder_transaction方法，这个方法是对一次Binder事务的处理，这其中会调用binder_alloc_buf函数为此次事务申请一个缓存。

```c
struct binder_buffer {
    struct list_head entry;
    struct rb_node rb_node;

    unsigned free:1;
    unsigned allow_user_free:1;
    unsigned async_transaction:1;
    unsigned debug_id:29;

    struct binder_transaction *transaction;

    struct binder_node *target_node;
    size_t data_size;
    size_t offsets_size;
    uint8_t data[0];
};
```

而在binder_proc（描述了使用Binder的进程）中，包含了几个字段用来管理进程在Binder IPC过程中缓存，如下：

```c
struct binder_proc {
    ...
    struct list_head buffers; // 进程拥有的buffer列表
    struct rb_root free_buffers; // 空闲buffer列表
    struct rb_root allocated_buffers; // 已使用的buffer列表
    size_t free_async_space; // 剩余的异步调用的空间

    size_t buffer_size; // 缓存的上限
  ...
};
```

进程在mmap时，会设定支持的总缓存大小的上限（下文会讲到）。而进程每当收到BC_TRANSACTION，就会判断已使用缓存加本次申请的和有没有超过上限。如果没有，就考虑进行内存的分配。

进程的空闲缓存记录在binder_proc的free_buffers中，这是一个以红黑树形式存储的结构。每次尝试分配缓存的时候，会从这里面按大小顺序进行查找，找到最接近需要的一块缓存。查找的逻辑如下：

```c
while (n) {
    buffer = rb_entry(n, struct binder_buffer, rb_node);
    BUG_ON(!buffer->free);
    buffer_size = binder_buffer_size(proc, buffer);

    if (size < buffer_size) {
        best_fit = n;
        n = n->rb_left;
    } else if (size > buffer_size)
        n = n->rb_right;
    else {
        best_fit = n;
        break;
    }
}
```

找到之后，还需要对binder_proc中的字段进行相应的更新：

```c
rb_erase(best_fit, &proc->free_buffers);
buffer->free = 0;
binder_insert_allocated_buffer(proc, buffer);
if (buffer_size != size) {
    struct binder_buffer *new_buffer = (void *)buffer->data + size;
    list_add(&new_buffer->entry, &buffer->entry);
    new_buffer->free = 1;
    binder_insert_free_buffer(proc, new_buffer);
}
binder_debug(BINDER_DEBUG_BUFFER_ALLOC,
         "%d: binder_alloc_buf size %zd got %p\n",
          proc->pid, size, buffer);
buffer->data_size = data_size;
buffer->offsets_size = offsets_size;
buffer->async_transaction = is_async;
if (is_async) {
    proc->free_async_space -= size + sizeof(struct binder_buffer);
    binder_debug(BINDER_DEBUG_BUFFER_ALLOC_ASYNC,
             "%d: binder_alloc_buf size %zd async free %zd\n",
              proc->pid, size, proc->free_async_space);
}
```

下面我们再来看看内存的释放。

BC_FREE_BUFFER命令是通知驱动进行内存的释放，binder_free_buf函数是真正实现的逻辑，这个函数与binder_alloc_buf是刚好对应的。在这个函数中，所做的事情包括：

重新计算进程的空闲缓存大小 通过binder_update_page_range释放内存 更新binder_proc的buffers，free_buffers，allocated_buffers字段

#### 2.7、**Binder中的"面向对象"**

Binder机制淡化了进程的边界，使得跨越进程也能够调用到指定服务的方法，其原因是因为Binder机制在底层处理了在进程间的"对象"传递。

在Binder驱动中，并不是真的将对象在进程间来回序列化，而是通过特定的标识来进行对象的传递。Binder驱动中，通过flat_binder_object来描述需要跨越进程传递的对象。其定义如下：

```c
struct flat_binder_object {
    __u32        type;
    __u32        flags;

    union {
        binder_uintptr_t    binder; /* local object */
        __u32            handle;    /* remote object */
    };
    binder_uintptr_t    cookie;
};
```

这其中，type有如下5种类型。

```c
enum {
    BINDER_TYPE_BINDER    = B_PACK_CHARS('s', 'b', '*', B_TYPE_LARGE),
    BINDER_TYPE_WEAK_BINDER    = B_PACK_CHARS('w', 'b', '*', B_TYPE_LARGE),
    BINDER_TYPE_HANDLE    = B_PACK_CHARS('s', 'h', '*', B_TYPE_LARGE),
    BINDER_TYPE_WEAK_HANDLE    = B_PACK_CHARS('w', 'h', '*', B_TYPE_LARGE),
    BINDER_TYPE_FD        = B_PACK_CHARS('f', 'd', '*', B_TYPE_LARGE),
};
```

当对象传递到Binder驱动中的时候，由驱动来进行翻译和解释，然后传递到接收的进程。

例如当Server把Binder实体传递给Client时，在发送数据流中，flat_binder_object中的type是BINDER_TYPE_BINDER，同时binder字段指向Server进程用户空间地址。但这个地址对于Client进程是没有意义的（Linux中，每个进程的地址空间是互相隔离的），驱动必须对数据流中的flat_binder_object做相应的翻译：将type该成BINDER_TYPE_HANDLE；为这个Binder在接收进程中创建位于内核中的引用并将引用号填入handle中。对于发生数据流中引用类型的Binder也要做同样转换。经过处理后接收进程从数据流中取得的Binder引用才是有效的，才可以将其填入数据包binder_transaction_data的target.handle域，向Binder实体发送请求。

由于每个请求和请求的返回都会经历内核的翻译，因此这个过程从进程的角度来看是完全透明的。进程完全不用感知这个过程，就好像对象真的在进程间来回传递一样。

#### 2.8、**驱动层的线程管理**

上文多次提到，Binder本身是C/S架构。由Server提供服务，被Client使用。既然是C/S架构，就可能存在多个Client会同时访问Server的情况。 在这种情况下，如果Server只有一个线程处理响应，就会导致客户端的请求可能需要排队而导致响应过慢的现象发生。解决这个问题的方法就是引入多线程。

Binder机制的设计从最底层–驱动层，就考虑到了对于多线程的支持。具体内容如下：

使用Binder的进程在启动之后，通过BINDER_SET_MAX_THREADS告知驱动其支持的最大线程数量 驱动会对线程进行管理。在binder_proc结构中，这些字段记录了进程中线程的信息：max_threads，requested_threads，requested_threads_started，ready_threads binder_thread结构对应了Binder进程中的线程 驱动通过BR_SPAWN_LOOPER命令告知进程需要创建一个新的线程 进程通过BC_ENTER_LOOPER命令告知驱动其主线程已经ready 进程通过BC_REGISTER_LOOPER命令告知驱动其子线程（非主线程）已经ready 进程通过BC_EXIT_LOOPER命令告知驱动其线程将要退出 在线程退出之后，通过BINDER_THREAD_EXIT告知Binder驱动。驱动将对应的binder_thread对象销毁

#### 2.9、再聊ServiceManager

上文已经说过，每一个Binder Server在驱动中会有一个binder_node进行对应。同时，Binder驱动会负责在进程间传递服务对象，并负责底层的转换。另外，我们也提到，每一个Binder服务都需要有一个唯一的名称。由ServiceManager来管理这些服务的注册和查找。

而实际上，为了便于使用，ServiceManager本身也实现为一个Server对象。任何进程在使用ServiceManager的时候，都需要先拿到指向它的标识。然后通过这个标识来使用ServiceManager。

这似乎形成了一个互相矛盾的现象：

通过ServiceManager我们才能拿到Server的标识 ServiceManager本身也是一个Server 解决这个矛盾的办法其实也很简单：Binder机制为ServiceManager预留了一个特殊的位置。这个位置是预先定好的，任何想要使用ServiceManager的进程只要通过这个特定的位置就可以访问到ServiceManager了（而不用再通过ServiceManager的接口）。

在Binder驱动中，有一个全局的binder_node 变量：

> **一般情况下，对于每一个Server驱动层会对应一个binder_node节点，然而binder_context_mgr_node比较特殊，它没有对应的应用层binder实体。在整个系统里，它是如此特殊，以至于系统规定，任何应用都必须使用句柄0来跨进程地访问它。**

```c
static struct binder_node *binder_context_mgr_node;
```

这个变量指向的就是ServiceManager。

当有进程通过ioctl并指定命令为BINDER_SET_CONTEXT_MGR的时候，驱动被认定这个进程是ServiceManager，binder_ioctl()函数中对应的处理如下：

```c
case BINDER_SET_CONTEXT_MGR:
    if (binder_context_mgr_node != NULL) {
    }
    ret = security_binder_set_context_mgr(proc->tsk);
    else {
    binder_context_mgr_uid = current->cred->euid;
    binder_context_mgr_node = binder_new_node(proc, 0, 0);//在Binder驱动层创建binder_node结构体对象
    binder_context_mgr_node->local_weak_refs++;
    binder_context_mgr_node->local_strong_refs++;
    binder_context_mgr_node->has_strong_ref = 1;
    binder_context_mgr_node->has_weak_ref = 1;
    }
    break;
```

ServiceManager应当要先于所有Binder Server之前启动。在它启动完成并告知Binder驱动之后，驱动便设定好了这个特定的节点。

在这之后，当有其他模块想要使用ServerManager的时候，只要将请求指向ServiceManager所在的位置即可。

在Binder驱动中，通过handle = 0这个位置来访问ServiceManager。例如，binder_transaction中，判断如果target.handler为0，则认为这个请求是发送给ServiceManager的，相关代码如下：

```c

if (tr->target.handle) {
    struct binder_ref *ref;
    ref = binder_get_ref(proc, tr->target.handle, true);
    if (ref == NULL) {
        binder_user_error("%d:%d got transaction to invalid handle\n",
            proc->pid, thread->pid);
        return_error = BR_FAILED_REPLY;
        goto err_invalid_target_handle;
    }
    target_node = ref->node;
} else {
    target_node = binder_context_mgr_node;
    if (target_node == NULL) {
        return_error = BR_DEAD_REPLY;
        goto err_no_context_mgr_node;
    }
}
```

#### 2.10、binder_node等重要结构体

> binder_proc binder_node binder_thread binder_ref binder_buffer

**1\. Binder实体binder_node**

Binder实体，是各个Server以及ServiceManager在内核中的存在形式。 Binder实体实际上是内核中binder_node结构体的对象，它的作用是在内核中保存Server和ServiceManager的信息(例如，Binder实体中保存了Server对象在用户空间的地址)。简言之，Binder实体是Server在Binder驱动中的存在形式，内核通过Binder实体可以找到用户空间的Server对象。 在上图中，Server和ServiceManager在Binder驱动中都对应的存在一个Binder实体。

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/26-Android-Binder_node_struct.png)

**2\. Binder引用binder_ref**

说到Binder实体，就不得不说"Binder引用"。所谓Binder引用，实际上是内核中binder_ref结构体的对象，它的作用是在表示"Binder实体"的引用。换句话说，每一个Binder引用都是某一个Binder实体的引用，通过Binder引用可以在内核中找到它对应的Binder实体。 如果将Server看作是Binder实体的话，那么Client就好比Binder引用。Client要和Server通信，它就是通过保存一个Server对象的Binder引用，再通过该Binder引用在内核中找到对应的Binder实体，进而找到Server对象，然后将通信内容发送给Server对象。

Binder实体和Binder引用都是内核(即，Binder驱动)中的数据结构。每一个Server在内核中就表现为一个Binder实体，而每一个Client则表现为一个Binder引用。这样，每个Binder引用都对应一个Binder实体，而每个Binder实体则可以多个Binder引用。 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/24-Android-binder_ref.png)

**3、Binder buffer：binder_buffer** 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/99-Android-Binder-IPCall.png)

**4、Binder进程binder_proc**

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/25-Android-binder_proc.png)

**5、Binder线程binder_thread** 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/27-Android-binder_thread.png)

binder机制到底是如何从Binder对象找到其对应的Binder实体呢？ 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/28-Android-Bp-Bbinder.png)

注意其中的那4个rb_root域，"rb"的意思是"red black"，可见binder_proc里搞出了4个红黑树。 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/29-Android-binder_proc_red_root.png)

其中，nodes树用于记录binder实体，refs_by_desc树和refs_by_node树则用于记录binder代理。之所以会有两个代理树，是为了便于快速查找，我们暂时只关心其中之一就可以了。threads树用于记录执行传输动作的线程信息。

在一个进程中，有多少"被其他进程进行跨进程调用的"binder实体，就会在该进程对应的nodes树中生成多少个红黑树节点。另一方面，一个进程要访问多少其他进程的binder实体，则必须在其refs_by_desc树中拥有对应的引用节点。

这4棵树的节点类型是不同的，threads树的节点类型为binder_thread，nodes树的节点类型为binder_node，refs_by_desc树和refs_by_node树的节点类型相同，为binder_ref。这些节点内部都会包含rb_node子结构，该结构专门负责连接节点的工作，和前文的hlist_node有点儿异曲同工，这也是linux上一个常用的小技巧。我们以nodes树为例 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/30-Android-binder_proc_hlist_node.png)

nodes树是用于记录binder实体的，所以nodes树中的每个binder_node节点，必须能够记录下相应binder实体的信息。因此请大家注意binder_node的ptr域和cookie域。

另一方面，refs_by_desc树和refs_by_node树的每个binder_ref节点则和上层的一个BpBinder对应，而且更重要的是，它必须具有和"目标binder实体的binder_node"进行关联的信息。

请注意binder_ref的那个node域，它负责和binder_node关联。另外，binder_ref中有两个类型为rb_node的域：rb_node_desc域和rb_node_node域，它们分别用于连接refs_by_desc树和refs_by_node。也就是说虽然binder_proc中有两棵引用树，但这两棵树用到的具体binder_ref节点其实是复用的。

> binder_node.ptr对应于flat_binder_object.binder； binder_node.cookie对应于flat_binder_object.cookie。

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/31-Android-binder_ref-find-binder_node.png)
OK，现在我们可以更深入地说明binder句柄的作用了，比如进程1的BpBinder在发起跨进程调用时，向binder驱动传入了自己记录的句柄值，binder驱动就会在"进程1对应的binder_proc结构"的引用树中查找和句柄值相符的binder_ref节点，一旦找到binder_ref节点，就可以通过该节点的node域找到对应的binder_node节点，这个目标binder_node当然是从属于进程2的binder_proc啦，不过不要紧，因为binder_ref和binder_node都处于binder驱动的地址空间中，所以是可以用指针直接指向的。目标binder_node节点的cookie域，记录的其实是进程2中BBinder的地址，binder驱动只需把这个值反映给应用层，应用层就可以直接拿到BBinder了。这就是Binder完成精确打击的大体过程。

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/32-Android-binder_relationship.jpg)

## 三、Android Binder系统驱动情景分析

为了更深刻的了解Binder系统 注册服务、获取服务、使用服务的过程，在Driver层(kernel/drivers/staging/android/binder.c)的binder_thread_read()函数、binder_transaction()函数入打印log，让前面编写的C程序示例与binder驱动交互打印更详细的过程。

```c
static void binder_transaction(struct binder_proc *proc,
                   struct binder_thread *thread,
                   struct binder_transaction_data *tr, int reply)
int binder_thread_write(struct binder_proc *proc, struct binder_thread *thread,
            void __user *buffer, int size, signed long *consumed)
```

[已添加好打印log的binder.c文件见GitHub（注：搜索[/* print] 关键字）](https://github.com/weidongshan/DRV_0003_Binder/) 事先已经准备好打印log，现在结合log和Binder事务处理开始详细分析。注：log稍后分析再贴出。 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/33-Android-binder_transaction_ipc.jpg)

### （1）、Binder系统驱动情景分析--服务"Hello"注册过程

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/35-Android-binder-add_service.png)

#### 1.1、ServiceManager休眠等待

回顾一下ServiceManager启动流程，ServiceManager进入binder_loop()后 会休眠等待响应client请求。

```c
binder_loop(){
    // 告诉Kernel，ServiceManager进程进入了消息循环状态。
    readbuf[0] = BC_ENTER_LOOPER;
    binder_write(bs, readbuf, sizeof(unsigned));

    bwr.write_size = 0;
    bwr.write_consumed = 0;
    bwr.write_buffer = 0;

    for (;;) {
        bwr.read_size = sizeof(readbuf);
        bwr.read_consumed = 0;
        bwr.read_buffer = (unsigned) readbuf;

        // 向Kernel中发送消息(先写后读)。
        // 先将消息传递给Kernel，然后再从Kernel读取消息反馈
        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
        // 解析读取的消息反馈
        res = binder_parse(bs, 0, readbuf, bwr.read_consumed, func);
        ...
    }
}
```

binder_write(bs, readbuf, sizeof(unsigned));会调用ioctl向内核发送数据。

```c
 int binder_write(struct binder_state *bs, void *data, size_t len)
{
    struct binder_write_read bwr;
    int res;

    bwr.write_size = len;
    bwr.write_consumed = 0;
    bwr.write_buffer = (uintptr_t) data;
    bwr.read_size = 0;
    bwr.read_consumed = 0;
    bwr.read_buffer = 0;
    res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
    return res;
}
```

> 如果 bwr.write_size > 0，则调用binder_thread_write 如果 bwr.read_size >0，则调用binder_thread_read

```c
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
  int ret;
  struct binder_proc *proc = filp->private_data;
  struct binder_thread *thread;
  unsigned int size = _IOC_SIZE(cmd);
  void __user *ubuf = (void __user *)arg;

  // 中断等待函数。
  ret = wait_event_interruptible(...);
  // 在proc进程中查找该线程对应的binder_thread；若查找失败，则新建一个binder_thread，并添加到proc->threads中。
  thread = binder_get_thread(proc);
  ...
  switch (cmd) {
  case BINDER_WRITE_READ: {
      struct binder_write_read bwr;
      ...
      // 将binder_write_read从"用户空间" 拷贝到 "内核空间"
      if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
          ...
      }
      // 如果write_size>0，则进行写操作
      if (bwr.write_size > 0) {
          ret = binder_thread_write(proc, thread, (void __user *)bwr.write_buffer, bwr.write_size, &bwr.write_consumed);
          ...
      }
      // 如果read_size>0，则进行读操作
      if (bwr.read_size > 0) {
          ret = binder_thread_read(proc, thread, (void __user *)bwr.read_buffer, bwr.read_size, &bwr.read_consumed, filp->f_flags   & O_NONBLOCK);
          ...
      }
      ...
      if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
      }
      break;
  }
  }
  return ret;
}
```

bwr.write_size > 0; 继续查看binder_thread_write()

> 注：只有BR_TRANSACTION、BR_REPLY、BC_TRANSACTION、BC_REPLY涉及两进程 其他所有BC_XXX、BR_XXX都只是App和驱动交互用于改变报告状态。

```c
int binder_thread_write(struct binder_proc *proc, struct binder_thread *thread,
          void __user *buffer, int size, signed long *consumed)
{
  uint32_t cmd;
  void __user *ptr = buffer + *consumed;
  void __user *end = buffer + size;

  // 读取binder_write_read.write_buffer中的内容。
  // 每次读取32bit(即4个字节)
  while (ptr < end && thread->return_error == BR_OK) {
      // 从用户空间读取32bit到内核中，并赋值给cmd。
      if (get_user(cmd, (uint32_t __user *)ptr))
          return -EFAULT;

      ptr += sizeof(uint32_t);
      switch (cmd) {
        case BC_ENTER_LOOPER:
            thread->looper |= BINDER_LOOPER_STATE_ENTERED;
            break;
      ...
      }
      // 更新bwr.write_consumed的值
      *consumed = ptr - buffer;
  }
  return 0;
}
```

当前线程进入BC_ENTER_LOOPER状态，等待请求。 继续binder_loop()中的for(;;;)循环，bwr.read_size >0;会通过binder_thread_read()读操作。

```c
static int binder_thread_read(struct binder_proc *proc,
                struct binder_thread *thread,
                void  __user *buffer, int size,
                signed long *consumed, int non_block)
{
  void __user *ptr = buffer + *consumed;
  void __user *end = buffer + size;
  int ret = 0;
  int wait_for_proc_work;
  // 如果*consumed=0，则写入BR_NOOP到用户传进来的bwr.read_buffer缓存区
  if (*consumed == 0) {
      if (put_user(BR_NOOP, (uint32_t __user *)ptr))
          return -EFAULT;
      // 修改指针位置
      ptr += sizeof(uint32_t);
  }
  ...
}
```

可以看到驱动put_user(BR_NOOP, (uint32_t __user *)ptr)发送BR_NOOP到ServiceManager

> 对于所有的读操作，数据头都是BR_NOOP，如BR_REPLY 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/36-Android-binder-BWR-read-BR_NOOP.png)

> ```c
>  ./service_manager &
> [   32.566620] service_manager (1362, 1362), binder_thread_write : BC_ENTER_LOOPER
> [   32.566712] service_manager (1362, 1362), binder_thread_read : BR_NOOP
> ```

#### 1.2、Clent（此处为Test_server）请求SM添加服务

**构造数据发送给驱动** 我们执行Test_server时，打印了很多数据，我们首先看一下数据的构造过程 和 组织格式，这有助于加深我们对binder系统的理解。

```c
int svcmgr_publish(struct binder_state *bs, uint32_t target, const char *name, void *ptr)
{
    int status;
    unsigned iodata[512/4];
    struct binder_io msg, reply;

    bio_init(&msg, iodata, sizeof(iodata), 4);
    bio_put_uint32(&msg, 0);  // strict mode header
    bio_put_string16_x(&msg, SVC_MGR_NAME);
    bio_put_string16_x(&msg, name);
    bio_put_obj(&msg, ptr);

    if (binder_call(bs, &msg, &reply, target, SVC_MGR_ADD_SERVICE))
        return -1;
    status = bio_get_uint32(&reply);
    binder_done(bs, &msg, &reply);
    return status;
}
```

bio_init()、bio_put_uint32()、bio_put_string16_x()函数比较简洁。我们看下bio_put_obj()函数。 构建初始化flat_binder_object结构体：

```c
void bio_put_obj(struct binder_io *bio, void *ptr)
{
    struct flat_binder_object *obj;

    obj = bio_alloc_obj(bio);
    if (!obj)
        return;

    obj->flags = 0x7f | FLAT_BINDER_FLAG_ACCEPTS_FDS;//
    obj->type = BINDER_TYPE_BINDER;//
    obj->binder = (uintptr_t)ptr;//
    obj->cookie = 0;//0
}
```

**数据结构示意图：**

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/37-Android-binder-Binder-io-transaction-data.jpg)

Clent（此处为Test_server），test_server.c调用流程： ->svcmgr_publish() ->binder_call() ->ioctl(bs->fd, BINDER_WRITE_READ, &bwr) ->binder_thread_write() ->binder_transaction()

现在数据构造好了，binder_call()会调用ioctl(bs->fd, BINDER_WRITE_READ, &bwr)

```c
int binder_call(struct binder_state *bs,
                struct binder_io *msg, struct binder_io *reply,
                uint32_t target, uint32_t code)
{
    int res;
    struct binder_write_read bwr;
    struct {
        uint32_t cmd;
        struct binder_transaction_data txn;
    } __attribute__((packed)) writebuf;
    unsigned readbuf[32];

    writebuf.cmd = BC_TRANSACTION;
    writebuf.txn.target.handle = target;
    writebuf.txn.code = code;
    writebuf.txn.flags = 0;
    writebuf.txn.data_size = msg->data - msg->data0;
    writebuf.txn.offsets_size = ((char*) msg->offs) - ((char*) msg->offs0);
    writebuf.txn.data.ptr.buffer = (uintptr_t)msg->data0;
    writebuf.txn.data.ptr.offsets = (uintptr_t)msg->offs0;

    bwr.write_size = sizeof(writebuf);
    bwr.write_consumed = 0;
    bwr.write_buffer = (uintptr_t) &writebuf;

    hexdump(msg->data0, msg->data - msg->data0);
    for (;;) {
        bwr.read_size = sizeof(readbuf);
        bwr.read_consumed = 0;
        bwr.read_buffer = (uintptr_t) readbuf;

        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
        ...
        res = binder_parse(bs, reply, (uintptr_t) readbuf, bwr.read_consumed, 0);
    }
}
```

[ 38.320197] test_server (1363, 1363), binder_thread_write : BC_TRANSACTION 发送数据，进而会调用binder_thread_write()处理数据。

```c
binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
  int ret;
  struct binder_proc *proc = filp->private_data;
  struct binder_thread *thread;
  unsigned int size = _IOC_SIZE(cmd);
  void __user *ubuf = (void __user *)arg;

  // 中断等待函数。
  ret = wait_event_interruptible(...);
  // 在proc进程中查找该线程对应的binder_thread；若查找失败，则新建一个binder_thread，并添加到proc->threads中。
  thread = binder_get_thread(proc);
  ...
  switch (cmd) {
  case BINDER_WRITE_READ: {
      struct binder_write_read bwr;
      ...
      // 将binder_write_read从"用户空间" 拷贝到 "内核空间"
      if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
          ...
      }
      // 如果write_size>0，则进行写操作
      if (bwr.write_size > 0) {
          ret = binder_thread_write(proc, thread, (void __user *)bwr.write_buffer, bwr.write_size, &bwr.write_consumed);
          ...
      }
      // 如果read_size>0，则进行读操作
      if (bwr.read_size > 0) {
          ret = binder_thread_read(proc, thread, (void __user *)bwr.read_buffer, bwr.read_size, &bwr.read_consumed, filp->f_flags   & O_NONBLOCK);
          ...
      }
      ...
      if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
      }
      break;
  }
  }
  return ret;
}
```

由于write_size>0，调用binder_thread_write()处理数据：

```c
int binder_thread_write(struct binder_proc *proc, struct binder_thread *thread,
          void __user *buffer, int size, signed long *consumed)
{
  uint32_t cmd;
  void __user *ptr = buffer + *consumed;
  void __user *end = buffer + size;

  // 读取binder_write_read.write_buffer中的内容。
  // 每次读取32bit(即4个字节)
  while (ptr < end && thread->return_error == BR_OK) {
      // 从用户空间读取32bit到内核中，并赋值给cmd。
      if (get_user(cmd, (uint32_t __user *)ptr))
          return -EFAULT;

      ptr += sizeof(uint32_t);
      ...
      switch (cmd) {
      ...
        case BC_TRANSACTION:
        case BC_REPLY: {
            struct binder_transaction_data tr;

            if (copy_from_user(&tr, ptr, sizeof(tr)))
                return -EFAULT;
            ptr += sizeof(tr);
            binder_transaction(proc, thread, &tr, cmd == BC_REPLY);
            break;
        }
      ...
      }
      // 更新bwr.write_consumed的值
      *consumed = ptr - buffer;
  }
  return 0;
}
```

由之前binder_call()分析，writebuf.cmd = BC_TRANSACTION;会执行binder_transaction()函数

```c
static void binder_transaction(struct binder_proc *proc,
                   struct binder_thread *thread,
                   struct binder_transaction_data *tr, int reply)
{
    struct binder_transaction *t;
    struct binder_work *tcomplete;
    size_t *offp, *off_end;
    struct binder_proc *target_proc;
    struct binder_thread *target_thread = NULL;
    struct binder_node *target_node = NULL;
    struct list_head *target_list;
    wait_queue_head_t *target_wait;
    struct binder_transaction *in_reply_to = NULL;
    struct binder_transaction_log_entry *e;
    uint32_t return_error;

    ...
    if (reply) {
    } else {
        if (tr->target.handle) {
            target_node = ref->node;
        } else {
            // 事务目标对象是ServiceManager的binder实体
            // 即，该事务是交给Service Manager来处理的。
            target_node = binder_context_mgr_node;
        }
        // 设置处理事务的目标进程
        target_proc = target_node->proc;
        ...
    }

    if (target_thread) {
        ...
    } else {
        target_list = &target_proc->todo;
        target_wait = &target_proc->wait;
    }
    ...

    // 分配一个待处理的事务t，t是binder事务(binder_transaction对象)
    t = kzalloc(sizeof(*t), GFP_KERNEL);
    // 分配一个待完成的工作tcomplete，tcomplete是binder_work对象。
    tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);
    ...

    // 事务将交给target_proc进程进行处理
    t->to_proc = target_proc;
    // 事务将交给target_thread线程进行处理
    t->to_thread = target_thread;
    ...

    // 分配空间,从目的进程映射的空间分配buf
    t->buffer = binder_alloc_buf(target_proc, tr->data_size,
        tr->offsets_size, !reply && (t->flags & TF_ONE_WAY));
    ...
    // 保存事务
    t->buffer->transaction = t;
    // 保存事务的目标对象(即处理该事务的binder对象)
    t->buffer->target_node = target_node;

    offp = (size_t *)(t->buffer->data + ALIGN(tr->data_size, sizeof(void *)));


    // 将"用户空间的数据"拷贝到内核中
    // tr->data.ptr.buffer就是用户空间数据的起始地址，tr->data_size就是数据大小
    if (copy_from_user(t->buffer->data, tr->data.ptr.buffer, tr->data_size)) {
        ...
    }
    // 将"用户空间的数据中所含对象的偏移地址"拷贝到内核中
    // tr->data.ptr.offsets就是数据中的对象偏移地址数组，tr->offsets_size就数据中的对象个数
    // 拷贝之后，offp就是flat_binder_object对象数组在内核空间的偏移数组的起始地址
    if (copy_from_user(offp, tr->data.ptr.offsets, tr->offsets_size)) {
        ...
    }
    ...
    // off_end就是flat_binder_object对象数组在内核空间的偏移地址的结束地址
    off_end = (void *)offp + tr->offsets_size;
    // 将所有的flat_binder_object对象读取出来
    // 对TestServer而言，只有一个flat_binder_object对象。
    for (; offp < off_end; offp++) {
        struct flat_binder_object *fp;
        ...
        fp = (struct flat_binder_object *)(t->buffer->data + *offp);

        switch (fp->type) {
        case BINDER_TYPE_BINDER:
        case BINDER_TYPE_WEAK_BINDER: {
            struct binder_ref *ref;
            // 在proc中查找binder实体对应的binder_node
            struct binder_node *node = binder_get_node(proc, fp->binder);
            // 若找不到，则新建一个binder_node；下次就可以直接使用了。
            if (node == NULL) {
                node = binder_new_node(proc, fp->binder, fp->cookie);
            }
            ...
            // 在target_proc(即，ServiceManager的进程上下文)中查找是否包行"该Binder实体的引用"，
            // 如果没有找到的话，则将"该binder实体的引用"添加到target_proc->refs_by_node红黑树中。这样，就可以通过Service Manager对该
Binder实体进行管理了。
            ref = binder_get_ref_for_node(target_proc, node);

            // 现在修改目的进程type，表示ServiceManager持有TestServer引用，TestServer进程才能拥有实体。
            if (fp->type == BINDER_TYPE_BINDER)
                fp->type = BINDER_TYPE_HANDLE;
            else
                fp->type = BINDER_TYPE_WEAK_HANDLE;
            // 修改handle。handle和binder是联合体，这里将handle设为引用的描述。
            // 根据该handle可以找到"该binder实体在target_proc中的binder引用"；
            // 即，可以根据该handle，可以从Service Manager找到对应的Binder实体的引用，从而获取Binder实体。
            fp->handle = ref->desc;
            // 增加引用计数，防止"该binder实体"在使用过程中被销毁。
            binder_inc_ref(ref, fp->type == BINDER_TYPE_HANDLE,
                       &thread->todo);
            ...
        } break;
        ...
        }
    }
    if (reply) {
        ..
    } else if (!(t->flags & TF_ONE_WAY)) {
        BUG_ON(t->buffer->async_transaction != 0);
        t->need_reply = 1;
        t->from_parent = thread->transaction_stack;
        // 将当前事务添加到当前线程的事务栈中
        thread->transaction_stack = t;
    } else {
        ...
    }
    // 设置事务的类型为BINDER_WORK_TRANSACTION
    t->work.type = BINDER_WORK_TRANSACTION;
    // 将事务添加到target_list队列中，即target_list的待处理事务中
    list_add_tail(&t->work.entry, target_list);
    // 设置待完成工作的类型为BINDER_WORK_TRANSACTION_COMPLETE
    tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;
    // 将待完成工作添加到thread->todo队列中，即当前线程的待完成工作中。
    list_add_tail(&tcomplete->entry, &thread->todo);
    // 唤醒目标进程
    if (target_wait)
        wake_up_interruptible(target_wait);
    return;

    ...
}
```

说明：这里的tr->target.handle=0，因此，会设置target_node为ServiceManager对应的Binder实体。下面是target_node,target_proc等值初始化之后的值。

```c
target_node = binder_context_mgr_node; // 目标节点为Service Manager对应的Binder实体
target_proc = target_node->proc;       // 目标进程为Service Manager对应的binder_proc进程上下文信息
target_list = &target_thread->todo;    // 待处理事务队列
target_wait = &target_thread->wait;    // 等待队列
```

小结： 驱动接收到TestServer发送的数据后，驱动主要工作： （1）根据Handle = 0 找到目的进程ServiceManager （2）把数据通过copy_from_user()放到目的进程ServiceManager的空间（mmap） （3）处理offs数据，即解析flat_binder_object结构体 a. 为TestServer构造binder_node node = binder_new_node(proc, fp->binder, fp->cookie); b.构造binder_ref给目的进程ServiceManager ref = binder_get_ref_for_node(target_proc, node); c.增加引用计数TestServer binder_inc_ref(ref, fp->type == BINDER_TYPE_HANDLE, &thread->todo); 增加引用计数会添加work.entry（BR_INCREFS、BR_ACQUIR）到TestServer todod队列 list_add_tail(&node->work.entry, target_list)

说明：就新建Binder实体的引用，并将其添加到target_proc->refs_by_node红黑树 和 target_proc->refs_by_desc红黑树中。 这样，ServiceManager的进程上下文中就存在Hello Service的Binder引用，ServiceManager也就可以对Hello Service进行管理了！然后，修改fp->type=BINDER_TYPE_HANDLE，并使fp->handle = ref->desc。

（4)新建一个待处理事务t和待完成的工作tcomplete，并对它们进行初始化。待处理事务t会被提交给目标(即ServiceManager对应的Binder实体)进行处理；而待完成的工作tcomplete则是为了反馈给TestServer服务，告诉TestServer它的请求Binder驱动已经收到了。注意，这里仅仅是告诉TestServer该请求已经被收到，而不是处理完毕！待ServiceManager处理完毕该请求之后，Binder驱动会再次反馈相应的消息给TestServer。 （5）binder_thread_write()中执行binder_transaction()后，会更新*consumed的值，即bwr.write_consumed的值 （6）此时，TestServer进程还会继续运行，而且它也通过wake_up_interruptible()唤醒了ServiceManager进程。

```c
  // 更新bwr.write_consumed的值
  *consumed = ptr - buffer;
```

接下来，ioctl()会执行binder_thread_read()来设置反馈数据给TestServer进程

```c
    static int binder_thread_read(struct binder_proc *proc,
                  struct binder_thread *thread,
                  void  __user *buffer, int size,
                  signed long *consumed, int non_block)
{
    // 如果*consumed=0，则写入BR_NOOP到用户传进来的bwr.read_buffer缓存区
    if (*consumed == 0) {
        if (put_user(BR_NOOP, (uint32_t __user *)ptr))
            return -EFAULT;
        ptr += sizeof(uint32_t);
    }
    ...
    while (1) {
        uint32_t cmd;
        struct binder_transaction_data tr;
        struct binder_work *w;
        struct binder_transaction *t = NULL;

        // 如果当前线程的"待完成工作"不为空，则取出待完成工作。
        if (!list_empty(&thread->todo))
            w = list_first_entry(&thread->todo, struct binder_work, entry);
        }
        ...
        switch (w->type) {
        ...
        case BINDER_WORK_TRANSACTION_COMPLETE: {
            cmd = BR_TRANSACTION_COMPLETE;
            // 将BR_TRANSACTION_COMPLETE写入到用户缓冲空间中
            if (put_user(cmd, (uint32_t __user *)ptr))
                return -EFAULT;
        } break;

    ...
    // 更新bwr.read_consumed的值
    *consumed = ptr - buffer;
}
```

首先发送BR_NOOP给TestServer，然后处理todo队列，处理完成后会发送BR_TRANSACTION_COMPLETE。

现在内核已经处理完数据，我们从log看看数据发生了哪些变化： 我们发现flat_binder_object结构体的type值发生了变化，binder变成了Handle，看一下结构体，handler 和 binder是一个union，占用同一个位置；Handle为1代表第一个引用，意思是在ServiceManager进程里面根据1能找到第一个binder_ref，根据binder_ref能找到服务hello的binder_node实体。 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/38-Android-binder-flat_binder_object.png)

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/39-Android-binder-Binder-io-transaction-data.jpg)

接下来就等待ServiceManager处理完成后，回复消息。

#### 1.3、唤醒ServiceManager执行添加"hello"服务

前面驱动已经创建好TestServer的binder_node，现在唤醒ServiceManager添加svcinfo 看看ServiceManager被唤醒后，会干些什么。

```c
static int binder_thread_read(struct binder_proc *proc,
                struct binder_thread *thread,
                void  __user *buffer, int size,
                signed long *consumed, int non_block)
{
    ...
    wait_for_proc_work = thread->transaction_stack == NULL &&
                list_empty(&thread->todo);


    while (1) {
        struct binder_transaction_data tr;
        struct binder_work *w;
        struct binder_transaction *t = NULL;

        // 如果当前线程的"待完成工作"不为空，则取出待完成工作。
        if (!list_empty(&thread->todo))
            w = list_first_entry(&thread->todo, struct binder_work, entry);
        else if (!list_empty(&proc->todo) && wait_for_proc_work)
        ...
        switch (w->type) {
            case BINDER_WORK_TRANSACTION: {
                t = container_of(w, struct binder_transaction, work);
            } break;
            ...
        }

        // t->buffer->target_node是目标节点。
        // 这里，addService请求的目标是ServiceManager，因此target_node是ServiceManager对应的节点；
        // 它的值在事务交互时(binder_transaction中)，被赋值为ServiceManager对应的Binder实体。  
        if (t->buffer->target_node) {
            // 事务目标对应的Binder实体(即，ServiceManager对应的Binder实体)
            struct binder_node *target_node = t->buffer->target_node;
            // Binder实体在用户空间的地址(ServiceManager的ptr为NULL)
            tr.target.ptr = target_node->ptr;
            // Binder实体在用户空间的其它数据(ServiceManager的cookie为NULL)
            tr.cookie =  target_node->cookie;
            t->saved_priority = task_nice(current);
            if (t->priority < target_node->min_priority &&
                !(t->flags & TF_ONE_WAY))
                binder_set_nice(t->priority);
            else if (!(t->flags & TF_ONE_WAY) ||
                 t->saved_priority > target_node->min_priority)
                binder_set_nice(target_node->min_priority);
            **cmd = BR_TRANSACTION;//将命令改为BR_TRANSACTION
        } else {
            tr.target.ptr = NULL;
            tr.cookie = NULL;
            cmd = BR_REPLY;
        }
        // 交易码
        tr.code = t->code;
        tr.flags = t->flags;
        tr.sender_euid = t->sender_euid;

        if (t->from) {
            struct task_struct *sender = t->from->proc->tsk;
            tr.sender_pid = task_tgid_nr_ns(sender,
                            current->nsproxy->pid_ns);
        } else {
            tr.sender_pid = 0;
        }

        // 数据大小
        tr.data_size = t->buffer->data_size;
        // 数据中对象的偏移数组的大小(即对象的个数)
        tr.offsets_size = t->buffer->offsets_size;
        // 数据
        tr.data.ptr.buffer = (void *)t->buffer->data +
                    proc->user_buffer_offset;
        // 数据中对象的偏移数组
        tr.data.ptr.offsets = tr.data.ptr.buffer +
                    ALIGN(t->buffer->data_size,
                        sizeof(void *));

        // 将cmd指令写入到ptr，即传递到用户空间
        if (put_user(cmd, (uint32_t __user *)ptr))
            return -EFAULT;
        // 将tr数据拷贝到用户空间
        ptr += sizeof(uint32_t);
        if (copy_to_user(ptr, &tr, sizeof(tr)))
            return -EFAULT;
        ptr += sizeof(tr);

        ...
        // 删除已处理的事务
        list_del(&t->work.entry);
        t->buffer->allow_user_free = 1;
        // 设置回复信息
        if (cmd == BR_TRANSACTION && !(t->flags & TF_ONE_WAY)) {
            // 该事务会发送给Service Manager守护进程进行处理。
            // Service Manager处理之后，还需要给Binder驱动回复处理结果。
            // 这里设置Binder驱动回复信息。
            t->to_parent = thread->transaction_stack;
            // to_thread表示Service Manager反馈后，将反馈结果交给当前thread进行处理
            t->to_thread = thread;
            // transaction_stack交易栈保存当前事务。用于之处反馈是针对哪个事务的。
            thread->transaction_stack = t;
        } else {
            ...
        }
        break;
    }

done:

    // 更新bwr.read_consumed的值
    *consumed = ptr - buffer;

    ...
    return 0;
}
```

说明：ServiceManager进程在调用wait_event_interruptible_exclusive(proc->wait, binder_has_proc_work(proc, thread))进入等待之后，被TestServer进程唤醒。唤醒之后，binder_has_thread_work()为true，因为ServiceManager的待处理事务队列中有个待处理事务(即，TestServer添加服务的请求)。 (01) 进入while循环后，首先取出待处理事务。 (02) 事务的类型是BINDER_WORK_TRANSACTION，得到对应的binder_transaction*类型指针t之后，跳出switch语句。很显然，此时t不为NULL，因此继续往下执行。下面的工作的目的，是将t中的数据转移到tr中(tr是事务交互数据包结构体binder_transaction_data对应的指针)，然后将指令和tr数据都拷贝到用户空间，让ServiceManager读取后进行处理。

Service Manager守护进程在处理完事务之后，需要反馈结果给Binder驱动。因此，接下来会设置t->to_thread和t->transaction_stack等成员。最后，修改*consumed的值，即bwr.read_consumed的值，表示待读取内容的大小。 执行完binder_thread_read()之后，回到binder_ioctl()中，执行copy_to_user()将数据拷贝到用户空间。接下来，就回到了Service Manager的守护进程当中，即回到binder_loop()中。 binder_loop()会将ioctl()反馈的数据发送给binder_parse()进行解析。

```c
int binder_parse(struct binder_state *bs, struct binder_io *bio,
                 uintptr_t ptr, size_t size, binder_handler func)
{
        switch(cmd) {
        case BR_NOOP:
        case BR_TRANSACTION: {
            struct binder_transaction_data *txn = (struct binder_transaction_data *) ptr;
            if ((end - ptr) < sizeof(*txn)) {
                ALOGE("parse: txn too small!\n");
                return -1;
            }
            binder_dump_txn(txn);
            if (func) {
                unsigned rdata[256/4];
                struct binder_io msg;
                struct binder_io reply;
                int res;

                bio_init(&reply, rdata, sizeof(rdata), 4);
                bio_init_from_txn(&msg, txn);
                res = func(bs, txn, &msg, &reply);
                binder_send_reply(bs, &reply, txn->data.ptr.buffer, res);
            }
            ptr += sizeof(*txn);
            break;
        }
}
```

首先会调用svcmgr_handler()->do_add_service()

```c
int do_add_service(struct binder_state *bs,
                   const uint16_t *s, size_t len,
                   uint32_t handle, uid_t uid, int allow_isolated,
                   pid_t spid)
{
    struct svcinfo *si;
    si = find_svc(s, len);
    if (si) {
        if (si->handle) {
                    svcinfo_death(bs, si);
        }
        si->handle = handle;
    } else {
        si = malloc(sizeof(*si) + (len + 1) * sizeof(uint16_t));
        si->handle = handle;
        si->len = len;
        memcpy(si->name, s, (len + 1) * sizeof(uint16_t));
        si->name[len] = '\0';
        si->death.func = (void*) svcinfo_death;
        si->death.ptr = si;
        si->allow_isolated = allow_isolated;
        si->next = svclist;
        svclist = si;
    }

    binder_acquire(bs, handle);
    binder_link_to_death(bs, handle, &si->death);
    return 0;
}
```

可以看到首先为hello服务新分配一个结构体svcinfo，然后将handle赋值给svcinfo，这也是以后我们查找服务所得到的handle。 然后调动了binder_acquire、binder_link_to_death发送信息给驱动。 [ 38.467270] service_manager (1362, 1362), binder_thread_write : BC_ACQUIRE [ 38.480122] service_manager (1362, 1362), binder_thread_write : BC_REQUEST_DEATH_NOTIFICATION 接着看binder_send_reply(bs, &reply, txn->data.ptr.buffer, res);

```c
void binder_send_reply(struct binder_state *bs,
                       struct binder_io *reply,
                       binder_uintptr_t buffer_to_free,
                       int status)
{
    data.cmd_free = BC_FREE_BUFFER;
    data.buffer = buffer_to_free;
    data.cmd_reply = BC_REPLY;
    data.txn.target.ptr = 0;
    data.txn.cookie = 0;
    data.txn.code = 0;
    ...
    binder_write(bs, &data, sizeof(data));
}
```

可以看到有BC_FREE_BUFFER、BC_REPLY，通过binder_write(bs, &data, sizeof(data))回复BC_REPLY到驱动。

驱动处理消息跟之前流程类似，这里不再分析。简单总结： 1、驱动接收到BC_REPLY请求，会新建一个待处理事务t（TestServer处理）和待完成的工作tcomplete（service_manager处理） 2、然后唤醒TestServer处理BC_REPLY请求 至此，已经成功添加Hello Service svcmgr: add_service('hello'), handle = 1

### （2）、Binder系统驱动情景分析--TestClent获取"Hello"服务过程

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/40-Android-binder-binder_get_service.png)

#### 2.0、构造数据

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/41-Android-binder-getSvr-Binder-io-transaction-data.jpg)

#### 2.1、发送数据给ServiceManager

bwr初始化完成之后，调用ioctl(,BINDER_WRITE_READ,)和Binder驱动进行交互。

```c
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
  int ret;
  struct binder_proc *proc = filp->private_data;
  struct binder_thread *thread;
  unsigned int size = _IOC_SIZE(cmd);
  void __user *ubuf = (void __user *)arg;
  switch (cmd) {
  case BINDER_WRITE_READ: {
      struct binder_write_read bwr;
      // 将binder_write_read从"用户空间" 拷贝到 "内核空间"
      if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
      }

      // 如果write_size>0，则进行写操作
      if (bwr.write_size > 0) {
          ret = binder_thread_write(proc, thread, (void __user *)bwr.write_buffer, bwr.write_size, &bwr.write_consumed);
      }

      // 如果read_size>0，则进行读操作
      if (bwr.read_size > 0) {
          ret = binder_thread_read(proc, thread, (void __user *)bwr.read_buffer, bwr.read_size, &bwr.read_consumed, filp->f_flags   & O_NONBLOCK);

      }
      if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
          ret = -EFAULT;
          goto err;
      }
      break;
  }
}
```

首先，会将binder_write_read从用户空间拷贝到内核空间之后。拷贝之后，读取出来的bwr.write_size和bwr.read_size都>0，因此先写后读。即，先执行binder_thread_write()，然后执行binder_thread_read()。

#### 2.2、binder_thread_write()处理数据

```c
int binder_thread_write(struct binder_proc *proc, struct binder_thread *thread,
          void __user *buffer, int size, signed long *consumed)
{
  uint32_t cmd;
  void __user *ptr = buffer + *consumed;
  void __user *end = buffer + size;

  // 读取binder_write_read.write_buffer中的内容。
  // 每次读取32bit(即4个字节)
  while (ptr < end && thread->return_error == BR_OK) {
      // 从用户空间读取32bit到内核中，并赋值给cmd。
      if (get_user(cmd, (uint32_t __user *)ptr))
          return -EFAULT;

      ptr += sizeof(uint32_t);
      ...
      switch (cmd) {
      ...
        case BC_TRANSACTION:
        case BC_REPLY: {
            struct binder_transaction_data tr;

            if (copy_from_user(&tr, ptr, sizeof(tr)))
                return -EFAULT;
            ptr += sizeof(tr);
            binder_transaction(proc, thread, &tr, cmd == BC_REPLY);
            break;
        }
      ...
      }
      // 更新bwr.write_consumed的值
      *consumed = ptr - buffer;
  }
  return 0;
}
```

说明：MediaPlayer发送的指令是BC_TRANSACTION，这里只关心与BC_TRANSACTION相关的部分。在通过copy_from_user()将数据拷贝从用户空间拷贝到内核空间之后，就调用binder_transaction()进行处理。

#### 2.3、Binder驱动中binder_transaction()的源码

```c
static void binder_transaction(struct binder_proc *proc,
                   struct binder_thread *thread,
                   struct binder_transaction_data *tr, int reply)
{
    struct binder_transaction *t;
    struct binder_work *tcomplete;
    size_t *offp, *off_end;
    struct binder_proc *target_proc;
    struct binder_thread *target_thread = NULL;
    struct binder_node *target_node = NULL;
    struct list_head *target_list;
    wait_queue_head_t *target_wait;
    struct binder_transaction *in_reply_to = NULL;
    struct binder_transaction_log_entry *e;
    uint32_t return_error;

    ...

    if (reply) {
        ...
    } else {
        if (tr->target.handle) {
            ...
        } else {
            // 该getService是从ServiceManager中获取MediaPlayer；
            // 因此事务目标对象是ServiceManager的binder实体。
            target_node = binder_context_mgr_node;
            ...
        }
        ...
        // 设置处理事务的目标进程
        target_proc = target_node->proc;
        ...
    }

    if (target_thread) {
        ...
    } else {
        target_list = &target_proc->todo;
        target_wait = &target_proc->wait;
    }
    ...

    // 分配一个待处理的事务t，t是binder事务(binder_transaction对象)
    t = kzalloc(sizeof(*t), GFP_KERNEL);
    ...

    // 分配一个待完成的工作tcomplete，tcomplete是binder_work对象。
    tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);
    ...

    t->debug_id = ++binder_last_id;
    ...

    // 设置from，表示该事务是MediaPlayer线程发起的
    if (!reply && !(tr->flags & TF_ONE_WAY))
        t->from = thread;
    else
        t->from = NULL;
    // 下面的一些赋值是初始化事务t
    t->sender_euid = proc->tsk->cred->euid;
    // 事务将交给target_proc进程进行处理
    t->to_proc = target_proc;
    // 事务将交给target_thread线程进行处理
    t->to_thread = target_thread;
    // 事务编码
    t->code = tr->code;
    // 事务标志
    t->flags = tr->flags;
    // 事务优先级
    t->priority = task_nice(current);

    ...

    // 分配空间
    t->buffer = binder_alloc_buf(target_proc, tr->data_size,
        tr->offsets_size, !reply && (t->flags & TF_ONE_WAY));
    ...
    t->buffer->allow_user_free = 0;
    t->buffer->debug_id = t->debug_id;
    // 保存事务
    t->buffer->transaction = t;
    // 保存事务的目标对象(即处理该事务的binder对象)
    t->buffer->target_node = target_node;
    trace_binder_transaction_alloc_buf(t->buffer);
    if (target_node)
        binder_inc_node(target_node, 1, 0, NULL);

    offp = (size_t *)(t->buffer->data + ALIGN(tr->data_size, sizeof(void *)));

    // 将"用户空间的数据"拷贝到内核中
    // tr->data.ptr.buffer就是用户空间数据的起始地址，tr->data_size就是数据大小
    if (copy_from_user(t->buffer->data, tr->data.ptr.buffer, tr->data_size)) {
        ...
    }
    // 将"用户空间的数据中所含对象的偏移地址"拷贝到内核中
    // MediaPlayer中不包含对象, offp=null
    if (copy_from_user(offp, tr->data.ptr.offsets, tr->offsets_size)) {
        ...
    }
    ...
    // MediaPlayer中不包含对象, off_end为null
    off_end = (void *)offp + tr->offsets_size;
    // MediaPlayer中不包含对象, offp=off_end
    for (; offp < off_end; offp++) {
        ...
    }
    if (reply) {
        ..
    } else if (!(t->flags & TF_ONE_WAY)) {
        BUG_ON(t->buffer->async_transaction != 0);
        t->need_reply = 1;
        t->from_parent = thread->transaction_stack;
        // 将当前事务添加到当前线程的事务栈中
        thread->transaction_stack = t;
    } else {
        ...
    }
    // 设置事务的类型为BINDER_WORK_TRANSACTION
    t->work.type = BINDER_WORK_TRANSACTION;
    // 将事务添加到target_list队列中，即target_list的待处理事务中
    list_add_tail(&t->work.entry, target_list);
    // 设置待完成工作的类型为BINDER_WORK_TRANSACTION_COMPLETE
    tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;
    // 将待完成工作添加到thread->todo队列中，即当前线程的待完成工作中。
    list_add_tail(&tcomplete->entry, &thread->todo);
    // 唤醒目标进程
    if (target_wait)
        wake_up_interruptible(target_wait);
    return;

    ...
}
```

说明：参数reply=0，表明这是个请求事务，而不是反馈。binder_transaction新建会新建"一个待处理事务t"和"待完成的工作tcomplete"，并根据请求的数据对它们进行初始化。 (01) TestClent的getService请求是提交给ServiceManager进行处理的，因此，"待处理事务t"会被添加到ServiceManager的待处理事务队列中。此时的target_thread是ServiceManager对应的线程，而target_proc则是ServiceManager对应的进程上下文环境。 (02) 此时，Binder驱动已经收到了TestClent的getService请求；于是，将一个BINDER_WORK_TRANSACTION_COMPLETE类型的"待完成工作tcomplete"添加到当前线程(即，TestClent线程)的待处理事务队列中。目的是告诉TestClent，Binder驱动已经收到它的getService请求了。 (03) 最后，调用wake_up_interruptible(target_wait)将ServiceManager唤醒。

接下来，还是先分析完TestClent线程，再看ServiceManager被唤醒后做了些什么。

binder_transaction()执行完毕之后，就会返回到binder_thread_write()中。binder_thread_write()更新bwr.write_consumed的值后，就返回到binder_ioctl()继续执行"读"动作。即执行binder_thread_read()。

#### 2.4、Binder驱动中binder_thread_read()的源码

```c
static int binder_thread_read(struct binder_proc *proc,
                  struct binder_thread *thread,
                  void  __user *buffer, int size,
                  signed long *consumed, int non_block)
{
    void __user *ptr = buffer + *consumed;
    void __user *end = buffer + size;

    int ret = 0;
    int wait_for_proc_work;

    // 如果*consumed=0，则写入BR_NOOP到用户传进来的bwr.read_buffer缓存区
    if (*consumed == 0) {
        if (put_user(BR_NOOP, (uint32_t __user *)ptr))
            return -EFAULT;
        ptr += sizeof(uint32_t);
    }

retry:
    // 等待proc进程的事务标记。
    // 当线程的事务栈为空 并且 待处理事务队列为空时，该标记位true。
    wait_for_proc_work = thread->transaction_stack == NULL &&
                list_empty(&thread->todo);

    ...

    if (wait_for_proc_work) {
        ...
    } else {
        if (non_block) {
            ...
        } else
            ret = wait_event_interruptible(thread->wait, binder_has_thread_work(thread));
    }

    ...

    while (1) {
        uint32_t cmd;
        struct binder_transaction_data tr;
        struct binder_work *w;
        struct binder_transaction *t = NULL;

        // 如果当前线程的"待完成工作"不为空，则取出待完成工作。
        if (!list_empty(&thread->todo))
            w = list_first_entry(&thread->todo, struct binder_work, entry);
        else if (!list_empty(&proc->todo) && wait_for_proc_work)
            ...
        else {
            if (ptr - buffer == 4 && !(thread->looper & BINDER_LOOPER_STATE_NEED_RETURN)) /* no data added */
                goto retry;
            break;
        }

        ...

        switch (w->type) {
        ...
        case BINDER_WORK_TRANSACTION_COMPLETE: {
            cmd = BR_TRANSACTION_COMPLETE;
            // 将BR_TRANSACTION_COMPLETE写入到用户缓冲空间中
            if (put_user(cmd, (uint32_t __user *)ptr))
                return -EFAULT;
            ptr += sizeof(uint32_t);

            ...
            // 待完成事务已经处理完毕，将其从待完成事务队列中删除。
            list_del(&w->entry);
            kfree(w);
            binder_stats_deleted(BINDER_STAT_TRANSACTION_COMPLETE);
        } break;
        ...
        }

        if (!t)
            continue;

        ...
    }

    ...
    // 更新bwr.read_consumed的值
    *consumed = ptr - buffer;

    ...
    return 0;
}
```

说明： (01) bwr.read_consumed=0，即if (*consumed == 0)为true。因此，会将BR_NOOP写入到bwr.read_buffer中。 (02) thread->transaction_stack不为空，thread->todo也不为空。因为，前面在binder_transaction()中有将一个BINDER_WORK_TRANSACTION_COMPLETE类型的待完成工作添加到thread的待完成工作队列中。因此，wait_for_proc_work为false。 (03) binder_has_thread_work(thread)为true。因此，在调用wait_event_interruptible()时，不会进入等待状态，而是继续运行。 (04) 进入while循环后，通过list_first_entry()取出待完成工作w。w的类型w->type=BINDER_WORK_TRANSACTION_COMPLETE，进入到对应的switch分支。随后，将BR_TRANSACTION_COMPLETE写入到bwr.read_buffer中。此时，待处理工作已经完成，将其从当前线程的待处理工作队列中删除。 (05) 最后，更新bwr.read_consumed的值。

经过binder_thread_read()处理之后，bwr.read_buffer中包含了两个指令：BR_NOOP和BR_TRANSACTION_COMPLETE。

#### 2.5、ServiceManager处理getService请求

下面看看ServiceManager被唤醒之后，是如何处理getService请求的

```c
int binder_parse(struct binder_state *bs, struct binder_io *bio,
                 uint32_t *ptr, uint32_t size, binder_handler func)
{
    while (ptr < end) {
        case BR_TRANSACTION: {
            struct binder_txn *txn = (void *) ptr;
            ...
            if (func) {
                unsigned rdata[256/4];
                struct binder_io msg;   // 用于保存"Binder驱动反馈的信息"
                struct binder_io reply; // 用来保存"回复给Binder驱动的信息"
                int res;

                // 初始化reply
                bio_init(&reply, rdata, sizeof(rdata), 4);
                // 根据txt(Binder驱动反馈的信息)初始化msg
                bio_init_from_txn(&msg, txn);
                // 消息处理
                res = func(bs, txn, &msg, &reply);
                // 反馈消息给Binder驱动。
                binder_send_reply(bs, &reply, txn->data, res);
}
```

binder_send_reply(bs, &reply, txn->data, res);->binder_write()

```c
binder_write()
int binder_write(struct binder_state *bs, void *data, unsigned len)
{
    struct binder_write_read bwr;
    int res;
    bwr.write_size = len;
    bwr.write_consumed = 0;
    bwr.write_buffer = (unsigned) data;
    bwr.read_size = 0;
    bwr.read_consumed = 0;
    bwr.read_buffer = 0;
    res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
    if (res < 0) {
        fprintf(stderr,"binder_write: ioctl failed (%s)\n",
                strerror(errno));
    }
    return res;
}
```

说明：binder_write()会将数据封装到binder_write_read的变量bwr中；其中，bwr.read_size=0，而bwr.write_size>0。接着，便通过ioctl(,BINDER_WRITE_READ,)和Binder驱动交互，将数据反馈给Binder驱动。

再次回到Binder驱动的binder_ioctl()对应的BINDER_WRITE_READ分支中。此时，由于bwr.read_size=0，而bwr.write_size>0；因此，Binder驱动只调用binder_thread_write进行写操作，而不会进行读。

返回数据：

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/42-Android-binder-Binder-io-transaction-data.jpg)

handle = 1 代表第一个

#### 2.6、Binder驱动中处理ServiceManager返回数据

```c

int binder_thread_write(struct binder_proc *proc, struct binder_thread *thread,
          void __user *buffer, int size, signed long *consumed)
{
  uint32_t cmd;
  void __user *ptr = buffer + *consumed;
  void __user *end = buffer + size;

  // 读取binder_write_read.write_buffer中的内容。
  // 每次读取32bit(即4个字节)
  while (ptr < end && thread->return_error == BR_OK) {
      // 从用户空间读取32bit到内核中，并赋值给cmd。
      if (get_user(cmd, (uint32_t __user *)ptr))
          return -EFAULT;

      ptr += sizeof(uint32_t);
      ...
      switch (cmd) {
        case BC_FREE_BUFFER:
        ...
        case BC_TRANSACTION:
        case BC_REPLY: {
            struct binder_transaction_data tr;

            if (copy_from_user(&tr, ptr, sizeof(tr)))
                return -EFAULT;
            ptr += sizeof(tr);
            binder_transaction(proc, thread, &tr, cmd == BC_REPLY);
            break;
        }
      ...
      }
      // 更新bwr.write_consumed的值
      *consumed = ptr - buffer;
  }
  return 0;
}
```

binder_thread_write()进入BC_REPLY之后，会将数据拷贝到内核空间，然后调用binder_transaction()进行处理

```c

static void binder_transaction(struct binder_proc *proc,
                   struct binder_thread *thread,
                   struct binder_transaction_data *tr, int reply)
{
    struct binder_transaction *t;
    struct binder_work *tcomplete;
    size_t *offp, *off_end;
    struct binder_proc *target_proc;
    struct binder_thread *target_thread = NULL;
    struct binder_node *target_node = NULL;
    struct list_head *target_list;
    wait_queue_head_t *target_wait;
    struct binder_transaction *in_reply_to = NULL;
    struct binder_transaction_log_entry *e;
    uint32_t return_error;

    ...

    if (reply) {
        // 事务栈
        in_reply_to = thread->transaction_stack;
        ...
        // 设置优先级
        binder_set_nice(in_reply_to->saved_priority);
        ...
        thread->transaction_stack = in_reply_to->to_parent;
        // 发起请求的线程，即MediaPlayer所在线程。
        // from的值，是MediaPlayer发起请求时在binder_transaction()中赋值的。
        target_thread = in_reply_to->from;
        ...
        // MediaPlayer对应的进程
        target_proc = target_thread->proc;
    } else {
        ...
    }
    if (target_thread) {
        e->to_thread = target_thread->pid;
        target_list = &target_thread->todo;
        target_wait = &target_thread->wait;
    } else {
        ...
    }
    e->to_proc = target_proc->pid;

    // 分配一个待处理的事务t，t是binder事务(binder_transaction对象)
    t = kzalloc(sizeof(*t), GFP_KERNEL);
    if (t == NULL) {
        return_error = BR_FAILED_REPLY;
        goto err_alloc_t_failed;
    }

    // 分配一个待完成的工作tcomplete，tcomplete是binder_work对象。
    tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);
    if (tcomplete == NULL) {
        return_error = BR_FAILED_REPLY;
        goto err_alloc_tcomplete_failed;
    }
    binder_stats_created(BINDER_STAT_TRANSACTION_COMPLETE);

    t->debug_id = ++binder_last_id;
    e->debug_id = t->debug_id;

    if (!reply && !(tr->flags & TF_ONE_WAY))
        t->from = thread;
    else
        t->from = NULL;
    // 下面的一些赋值是初始化事务t
    t->sender_euid = proc->tsk->cred->euid;
    // 事务将交给target_proc进程进行处理
    t->to_proc = target_proc;
    // 事务将交给target_thread线程进行处理
    t->to_thread = target_thread;
    // 事务编码
    t->code = tr->code;
    // 事务标志
    t->flags = tr->flags;
    // 事务优先级
    t->priority = task_nice(current);

    // 分配空间
    t->buffer = binder_alloc_buf(target_proc, tr->data_size,
        tr->offsets_size, !reply && (t->flags & TF_ONE_WAY));
    if (t->buffer == NULL) {
        return_error = BR_FAILED_REPLY;
        goto err_binder_alloc_buf_failed;
    }
    t->buffer->allow_user_free = 0;
    t->buffer->debug_id = t->debug_id;
    // 保存事务
    t->buffer->transaction = t;
    // target_node为NULL
    t->buffer->target_node = target_node;
    trace_binder_transaction_alloc_buf(t->buffer);
    if (target_node)
        binder_inc_node(target_node, 1, 0, NULL);

    offp = (size_t *)(t->buffer->data + ALIGN(tr->data_size, sizeof(void *)));

    // 将"用户传入的数据"保存到事务中
    if (copy_from_user(t->buffer->data, tr->data.ptr.buffer, tr->data_size)) {
        ...
    }
    // 将"用户传入的数据偏移地址"保存到事务中
    if (copy_from_user(offp, tr->data.ptr.offsets, tr->offsets_size)) {
        ...
    }

    ...
    off_end = (void *)offp + tr->offsets_size;
    // 将flat_binder_object对象读取出来，
    // 这里就是Service Manager中反馈的MediaPlayerService对象。
    for (; offp < off_end; offp++) {
        struct flat_binder_object *fp;
        ...
        fp = (struct flat_binder_object *)(t->buffer->data + *offp);

        switch (fp->type) {
            ...
            case BINDER_TYPE_HANDLE:
            case BINDER_TYPE_WEAK_HANDLE: {
                // 根据handle获取对应的Binder引用，即得到MediaPlayerService的Binder引用
                struct binder_ref *ref = binder_get_ref(proc, fp->handle);
                if (ref == NULL) {
                    ...
                }
                // ref->node->proc是MediaPlayerService的进程上下文环境，
                // 而target_proc是MediaPlayer的进程上下文环境
                if (ref->node->proc == target_proc) {
                    ...
                } else {
                    struct binder_ref *new_ref;
                    // 在MediaPlayer进程中引用"MediaPlayerService"。
                    // 表现为，执行binder_get_ref_for_node()会，会先在MediaPlayer进程中查找是否存在MediaPlayerService对应的Binder引用；
                    // 很显然是不存在的。于是，并新建MediaPlayerService对应的Binder引用，并将其添加到MediaPlayer的Binder引用红黑树中。
                    new_ref = binder_get_ref_for_node(target_proc, ref->node);
                    if (new_ref == NULL) {
                        ...
                    }
                    // 将new_ref的引用描述复制给fp->handle。
                    fp->handle = new_ref->desc;
                    binder_inc_ref(new_ref, fp->type == BINDER_TYPE_HANDLE, NULL);
                    ...
                }
            } break;

        }
    }

    if (reply) {
        binder_pop_transaction(target_thread, in_reply_to);
    } else if (!(t->flags & TF_ONE_WAY)) {
        ...
    } else {
        ...
    }
    // 设置事务的类型为BINDER_WORK_TRANSACTION
    t->work.type = BINDER_WORK_TRANSACTION;
    // 将事务添加到target_list队列中，即target_list的待处理事务中
    list_add_tail(&t->work.entry, target_list);
    // 设置待完成工作的类型为BINDER_WORK_TRANSACTION_COMPLETE
    tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;
    // 将待完成工作添加到thread->todo队列中，即当前线程的待完成工作中。
    list_add_tail(&tcomplete->entry, &thread->todo);
    // 唤醒目标进程
    if (target_wait)
        wake_up_interruptible(target_wait);
    return;
    ...
}
```

说明：reply=1，这里只关注reply部分。 (01) 此反馈最终是要回复给TestClient的。因此，target_thread被赋值为TestServer所在的线程，target_proc则是TestClient对应的进程，target_node为null。 (02) 这里，先看看for循环里面的内容，取出BR_REPLY指令所发送的数据，然后获取数据中的flat_binder_object变量fp。因为fp->type为BINDER_TYPE_HANDLE，因此进入BINDER_TYPE_HANDLE对应的分支。接着，通过binder_get_ref()获取Hello Service对应的Binder引用；很明显，能够正常获取到Hello Service的Binder引用。因为在Hello Service调用addService请求时，已经创建了它的Binder引用。 binder_get_ref_for_node()的作用是在TestClent进程上下文中添加"TestServer对应的Binder引用"。这样，后面就可以根据该Binder引用一步步的获取TestServer对象。 最后，将Binder引用的描述赋值给fp->handle。 (03) 此时，Service Manager已经处理了getService请求。便调用binder_pop_transaction(target_thread, in_reply_to)将事务从"target_thread的事务栈"中删除，即从MediaPlayer线程的事务栈中删除该事务。 (04) 新建的"待处理事务t"的type为设为BINDER_WORK_TRANSACTION后，会被添加到MediaPlayer的待处理事务队列中。 (05) 此时，Service Manager已经处理了getService请求，而Binder驱动在等待它的回复。于是，将一个BINDER_WORK_TRANSACTION_COMPLETE类型的"待完成工作tcomplete"(作为回复)添加到当前线程(即，Service Manager线程)的待处理事务队列中。 (06) 最后，调用wake_up_interruptible()唤醒TestServer。TestServer被唤醒后，会对事务BINDER_WORK_TRANSACTION进行处理。

OK，到现在为止，还有两个待处理事务：(01) ServiceManager待处理事务列表中有个BINDER_WORK_TRANSACTION_COMPLETE类型的事务 (02) TestServer待处理事务列表中有个BINDER_WORK_TRANSACTION事务。

#### 2.7\. Testclient获取handle

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/43-Android-binder-binder_use_service.png)

### （3）、Binder系统驱动情景分析--TestClent使用"Hello"服务过程

构造数据发送数据"weidongshan" 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/44-Android-binder-Binder-io-transaction-data.jpg)

## 四、Android Binder系统-Native层

前面我们分析内核驱动Binder使用过程，可以看到，binder系统在内核能正常完成IPC通信，接下来分析Android framwork层，最后是App层。

Framework是一个中间层，它对接了底层实现，封装了复杂的内部逻辑，并提供供外部使用的接口。Framework层是应用程序开发的基础。

Binder Framework层分为C++和Java两个部分，为了达到功能的复用，中间通过JNI进行衔接。

Binder Framework的C++部分，头文件位于这个路径：/frameworks/native/include/binder/，实现位于这个路径：/frameworks/native/libs/binder/ 。Binder库最终会编译成一个动态链接库：libbinder.so，供其他进程链接使用。

为了便于说明，下文中我们将Binder Framework 的C++部分称之为libbinder。首先说一下ServiceManager，然后详细介绍。

### (1)、ServiceManager类图(Native层)

IServiceManager相关类如下图所示： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/45-Android-native_binder_framework_servicemananger.png)

IServiceManager是表示servicemanager的接口，有如下方法：

1) getService获得binder service引用，

2) checkService获得binder service引用，

3) addService添加binder service，

4) listServices 列举所有binder service。

servicemanager的binder service服务端其实是在frameworks/base/cmds/servicemanager 里实现，BnServiceMananger实际上并未使用。BpServiceMananger就是利用获得的IBinder指针建立的IServiceMananger对象的实际类型。

### (2)、Binder框架Native层

libbinder中，将实现分为Proxy和Native两端。Proxy对应了上文提到的Client端，是服务对外提供的接口。而Native是服务实现的一端，对应了上文提到的Server端。类名中带有小写字母p的（例如BpInterface），就是指Proxy端。类名带有小写字母n的（例如BnInterface），就是指Native端。

Proxy代表了调用方，通常与服务的实现不在同一个进程，因此下文中，我们也称Proxy端为"远程"端。Native端是服务实现的自身，因此下文中，我们也称Native端为"本地"端。

这里，我们先对libbinder中的主要类做一个简要说明，了解一下它们的关系，然后再详细的讲解。

类名             | 说明
:------------- | :----------------------------------------
BpRefBase      | RefBase的子类，提供remote()方法获取远程Binder
IInterface     | Binder服务接口的基类，Binder服务通常需要同时提供本地接口和远程接口
BpInterface    | 远程接口的基类，远程接口是供客户端调用的接口集
BnInterface    | 本地接口的基类，本地接口是需要服务中真正实现的接口集
IBiner         | Binder对象的基类，BBinder和BpBinder都是这个类的子类
BpBinder       | 远程Binder，这个类提供transact方法来发送请求，BpXXX实现中会用到
BBinder        | 本地Binder，服务实现方的基类，提供了onTransact接口来接收请求
ProcessState   | 代表了使用Binder的进程
IPCThreadState | 代表了使用Binder的线程，这个类中封装了与Binder驱动通信的逻辑
Parcel         | 在Binder上传递的数据的包装器

下图描述了这些类之间的关系： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/46-Androi-binder_middleware.png)

另外说明一下，Binder服务的实现类（图中紫色部分）通常都会遵守下面的命名规则：

☯ 服务的接口使用I字母作为前缀 ☯ 远程接口使用Bp作为前缀 ☯ 本地接口使用Bn作为前缀

看了上面这些介绍，你可能还是不太容易理解。不过不要紧，下面我们会逐步拆分讲解这些内容。

在这幅图中，浅黄色部分的结构是最难理解的，因此我们先从它们着手。

我们先来看看IBinder这个类。这个类描述了所有在Binder上传递的对象，它既是Binder本地对象BBinder的父类，也是Binder远程对象BpBinder的父类。这个类中的主要方法说明如下：

方法名                    | 说明
:--------------------- | :--------------------------------
localBinder            | 获取本地Binder对象
remoteBinder           | 获取远程Binder对象
transact               | 进行一次Binder操作
queryLocalInterface    | 尝试获取本地Binder，如何失败返回NULL
getInterfaceDescriptor | 获取Binder的服务接口描述，其实就是Binder服务的唯一标识
isBinderAlive          | 查询Binder服务是否还活着
pingBinder             | 发送PING_TRANSACTION给Binder服务

BpBinder的实例代表了远程Binder，这个类的对象将被客户端调用。其中handle方法会返回指向Binder服务实现者的句柄，这个类最重要就是提供了transact方法，这个方法会将远程调用的参数封装好发送的Binder驱动。

由于每个Binder服务通常都会提供多个服务接口，而这个方法中的uint32_t code参数就是用来对服务接口进行编号区分的。Binder服务的每个接口都需要指定一个唯一的code，这个code要在Proxy和Native端配对好。当客户端将请求发送到服务端的时候，服务端根据这个code（onTransact方法中）来区分调用哪个接口方法。

BBinder的实例代表了本地Binder，它描述了服务的提供方，所有Binder服务的实现者都要继承这个类（的子类），在继承类中，最重要的就是实现onTransact方法，因为这个方法是所有请求的入口。因此，这个方法是和BpBinder中的transact方法对应的，这个方法同样也有一个uint32_t code参数，在这个方法的实现中，由服务提供者通过code对请求的接口进行区分，然后调用具体实现服务的方法。

IBinder中定义了uint32_t code允许的范围：

```c
FIRST_CALL_TRANSACTION  = 0x00000001,
LAST_CALL_TRANSACTION   = 0x00ffffff,
```

Binder服务要保证自己提供的每个服务接口有一个唯一的code，例如hello服务:

```c
#define HELLO_SVR_CMD_SAYHELLO     1
#define HELLO_SVR_CMD_SAYHELLO_TO  2
#define HELLO_SVR_CMD_GET_FD       3
```

讲完了IBinder，BpBinder和BBinder三个类，我们再来看看BpReBase，IInterface，BpInterface和BnInterface。

每个Binder服务都是为了某个功能而实现的，因此其本身会定义一套接口集（通常是C++的一个类）来描述自己提供的所有功能。而Binder服务既有自身实现服务的类，也要有给客户端进程调用的类。为了便于开发，这两中类里面的服务接口应当是一致的，例如：假设服务实现方提供了一个接口为sayhello(void)的服务方法，那么其远程接口中也应当有一个sayhello(void)方法。因此为了实现方便，本地实现类和远程接口类需要有一个公共的描述服务接口的基类（即上图中的IXXXService）来继承。而这个基类通常是IInterface的子类，IInterface的定义如下：

```c
class IInterface : public virtual RefBase
{
public:
            IInterface();
            static sp<IBinder>  asBinder(const IInterface*);
            static sp<IBinder>  asBinder(const sp<IInterface>&);

protected:
    virtual                     ~IInterface();
    virtual IBinder*            onAsBinder() = 0;
};
```

之所以要继承自IInterface类是因为这个类中定义了onAsBinder让子类实现。onAsBinder在本地对象的实现类中返回的是本地对象，在远程对象的实现类中返回的是远程对象。onAsBinder方法被两个静态方法asBinder方法调用。有了这些接口之后，在代码中便可以直接通过IXXX::asBinder方法获取到不用区分本地还是远程的IBinder对象。这个在跨进程传递Binder对象的时候有很大的作用（因为不用区分具体细节，只要直接调用和传递就好）。

下面，我们来看一下BpInterface和BnInterface的定义：

```c
template<typename INTERFACE>
class BnInterface : public INTERFACE, public BBinder
{
public:
    virtual sp<IInterface>      queryLocalInterface(const String16& _descriptor);
    virtual const String16&     getInterfaceDescriptor() const;

protected:
    virtual IBinder*            onAsBinder();
};

// ----------------------------------------------------------------------

template<typename INTERFACE>
class BpInterface : public INTERFACE, public BpRefBase
{
public:
                                BpInterface(const sp<IBinder>& remote);

protected:
    virtual IBinder*            onAsBinder();
};
```

这两个类都是模板类，它们在继承自INTERFACE的基础上各自继承了另外一个类。这里的INTERFACE便是我们Binder服务接口的基类。另外，BnInterface继承了BBinder类，由此可以通过复写onTransact方法来提供实现。BpInterface继承了BpRefBase，通过这个类的remote方法可以获取到指向服务实现方的句柄。在客户端接口的实现类中，每个接口在组装好参数之后，都会调用remote()->transact来发送请求，而这里其实就是调用的BpBinder的transact方法，这样请求便通过Binder到达了服务实现方的onTransact中。这个过程如下图所示： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/47-Android-Binder-IPCall.png)

基于Binder框架开发的服务，除了满足上文提到的类名规则之外，还需要遵守其他一些共同的规约：

☯为了进行服务的区分，每个Binder服务需要指定一个唯一的标识，这个标识通过getInterfaceDescriptor返回，类型是一个字符串。通常，Binder服务会在类中定义static const android::String16 descriptor;这样一个常量来描述这个标识符，然后在getInterfaceDescriptor方法中返回这个常量。 ☯为了便于调用者获取到调用接口，服务接口的公共基类需要提供一个android::sp

<ixxx> asInterface方法来返回基类对象指针。
由于上面提到的这两点对于所有Binder服务的实现逻辑都是类似的。为了简化开发者的重复工作，在libbinder中，定义了两个宏来简化这些重复工作，它们是：</ixxx>

```c
#define DECLARE_META_INTERFACE(INTERFACE)                            \
    static const android::String16 descriptor;                       \
    static android::sp<I##INTERFACE> asInterface(                    \
            const android::sp<android::IBinder>& obj);               \
    virtual const android::String16& getInterfaceDescriptor() const; \
    I##INTERFACE();                                                  \
    virtual ~I##INTERFACE();                                         \


#define IMPLEMENT_META_INTERFACE(INTERFACE, NAME)                    \
    const android::String16 I##INTERFACE::descriptor(NAME);          \
    const android::String16&                                         \
            I##INTERFACE::getInterfaceDescriptor() const {           \
        return I##INTERFACE::descriptor;                             \
    }                                                                \
    android::sp<I##INTERFACE> I##INTERFACE::asInterface(             \
            const android::sp<android::IBinder>& obj)                \
    {                                                                \
        android::sp<I##INTERFACE> intr;                              \
        if (obj != NULL) {                                           \
            intr = static_cast<I##INTERFACE*>(                       \
                obj->queryLocalInterface(                            \
                        I##INTERFACE::descriptor).get());            \
            if (intr == NULL) {                                      \
                intr = new Bp##INTERFACE(obj);                       \
            }                                                        \
        }                                                            \
        return intr;                                                 \
    }                                                                \
    I##INTERFACE::I##INTERFACE() { }                                 \
    I##INTERFACE::~I##INTERFACE() { }                                \
```

有了这两个宏之后，开发者只要在接口基类（IXXX）头文件中，使用DECLARE_META_INTERFACE宏便完成了需要的组件的声明。然后在cpp文件中使用IMPLEMENT_META_INTERFACE便完成了这些组件的实现。

#### 2.1、**Binder的初始化ProcessState**

在讲解Binder驱动的时候我们就提到：任何使用Binder机制的进程都必须要对/dev/binder设备进行open以及mmap之后才能使用，这部分逻辑是所有使用Binder机制进程共同的。对于这种共同逻辑的封装便是Framework层的职责之一。libbinder中，ProcessState类封装了这个逻辑，相关代码见下文。

这里是ProcessState构造函数，在这个函数中，初始化mDriverFD的时候调用了open_driver方法打开binder设备，然后又在函数体中，通过mmap进行内存映射。

```c
ProcessState::ProcessState()
    : mDriverFD(open_driver())
    , mVMStart(MAP_FAILED)
    , mThreadCountLock(PTHREAD_MUTEX_INITIALIZER)
    , mThreadCountDecrement(PTHREAD_COND_INITIALIZER)
    , mExecutingThreadsCount(0)
    , mMaxThreads(DEFAULT_MAX_BINDER_THREADS)
    , mStarvationStartTimeMs(0)
    , mManagesContexts(false)
    , mBinderContextCheckFunc(NULL)
    , mBinderContextUserData(NULL)
    , mThreadPoolStarted(false)
    , mThreadPoolSeq(1)
{
    if (mDriverFD >= 0) {
        mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
        ...
    }
}
```

open_driver的函数实现如下所示。在这个函数中完成了三个工作：

☯首先通过open系统调用打开了dev/binder设备 ☯然后通过ioctl获取Binder实现的版本号，并检查是否匹配 ☯最后通过ioctl设置进程支持的最大线程数量 关于这部分逻辑背后的处理，在讲解Binder驱动的时候，我们已经讲解过了。

```c
static int open_driver()
{
    int fd = open("/dev/binder", O_RDWR | O_CLOEXEC);
    if (fd >= 0) {
        int vers = 0;
        status_t result = ioctl(fd, BINDER_VERSION, &vers);
        ...
        size_t maxThreads = DEFAULT_MAX_BINDER_THREADS;
        result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads);
    } else {
        ...
    }
    return fd;
}
```

ProcessState是一个Singleton（单例）类型的类，在一个进程中，只会存在一个实例。通过ProcessState::self()接口获取这个实例。一旦获取这个实例，便会执行其构造函数，由此完成了对于Binder设备的初始化工作。

#### 2.2、**关于Binder传递数据的大小限制**

由于Binder的数据需要跨进程传递，并且还需要在内核上开辟空间，因此允许在Binder上传递的数据并不是无无限大的。mmap中指定的大小便是对数据传递的大小限制：

```c
#define BINDER_VM_SIZE ((1*1024*1024) - (4096 *2)) // 1M - 8k
mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
```

这里我们看到，在进行mmap的时候，指定了最大size为BINDER_VM_SIZE，即 1M - 8k的大小。 因此我们在开发过程中，一次Binder调用的数据总和不能超过这个大小。

对于这个区域的大小，我们也可以在设备上进行确认。这里我们还之前提到的system_server为例。上面我们讲解了通过procfs来获取映射的内存地址，除此之外，我们也可以通过showmap命令，来确定这块区域的大小，相关命令如下：

```c
angler:/ # ps  | grep system_server                                            
system    1889  526   2353404 135968 SyS_epoll_ 72972eeaf4 S system_server
angler:/ # showmap 1889 | grep "/dev/binder"                                   
    1016        4        4        0        0        4        0        0    1 /dev/binder
```

这里可以看到，这块区域的大小正是 1M - 8K = 1016k。

Tips: 通过showmap命令可以看到进程的详细内存占用情况。在实际的开发过程中，当我们要对某个进程做内存占用分析的时候，这个命令是相当有用的。建议读者尝试通过showmap命令查看system_server或其他感兴趣进程的完整map，看看这些进程都依赖了哪些库或者模块，以及内存占用情况是怎样的。

#### 2.3、**与驱动的通信IPCThreadState**

上文提到ProcessState是一个单例类，一个进程只有一个实例。而负责与Binder驱动通信的IPCThreadState也是一个单例类。但这个类不是一个进程只有一个实例，而是一个线程有一个实例。

IPCThreadState负责了与驱动通信的细节处理。这个类中的关键几个方法说明如下：

方法                   | 说明
:------------------- | :----------------------------------
transact             | 公开接口。供Proxy发送数据到驱动，并读取返回结果
sendReply            | 供Server端写回请求的返回结果
waitForResponse      | 发送请求后等待响应结果
talkWithDriver       | 通过ioctl BINDER_WRITE_READ来与驱动通信
writeTransactionData | 写入一次事务的数据
executeCommand       | 处理binder_driver_return_protocol协议命令
freeBuffer           | 通过BC_FREE_BUFFER命令释放Buffer

BpBinder::transact方法在发送请求的时候，其实就是直接调用了IPCThreadState对应的方法来发送请求到Binder驱动的，相关代码如下：

```c
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    if (mAlive) {
        status_t status = IPCThreadState::self()->transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;
        return status;
    }

    return DEAD_OBJECT;
}
```

而IPCThreadState::transact方法主要逻辑如下：

```c
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    status_t err = data.errorCheck();

    flags |= TF_ACCEPT_FDS;

    if (err == NO_ERROR) {
        err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
    }

    if (err != NO_ERROR) {
        if (reply) reply->setError(err);
        return (mLastError = err);
    }

    if ((flags & TF_ONE_WAY) == 0) {
        if (reply) {
            err = waitForResponse(reply);
        } else {
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }
    } else {
        err = waitForResponse(NULL, NULL);
    }

    return err;
}
```

这段代码应该还是比较好理解的：首先通过writeTransactionData写入数据，然后通过waitForResponse等待返回结果。TF_ONE_WAY表示此次请求是单向的，即：不用真正等待结果即可返回。

而writeTransactionData方法其实就是在组装binder_transaction_data数据：

```c
status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
{
    binder_transaction_data tr;

    tr.target.ptr = 0; /* Don't pass uninitialized stack data to a remote process */
    tr.target.handle = handle;
    tr.code = code;
    tr.flags = binderFlags;
    tr.cookie = 0;
    tr.sender_pid = 0;
    tr.sender_euid = 0;

    const status_t err = data.errorCheck();
    if (err == NO_ERROR) {
        tr.data_size = data.ipcDataSize();
        tr.data.ptr.buffer = data.ipcData();
        tr.offsets_size = data.ipcObjectsCount()*sizeof(binder_size_t);
        tr.data.ptr.offsets = data.ipcObjects();
    } else if (statusBuffer) {
        tr.flags |= TF_STATUS_CODE;
        *statusBuffer = err;
        tr.data_size = sizeof(status_t);
        tr.data.ptr.buffer = reinterpret_cast<uintptr_t>(statusBuffer);
        tr.offsets_size = 0;
        tr.data.ptr.offsets = 0;
    } else {
        return (mLastError = err);
    }

    mOut.writeInt32(cmd);
    mOut.write(&tr, sizeof(tr));

    return NO_ERROR;
}
```

对于binder_transaction_data在讲解Binder驱动的时候我们已经详细讲解过了。而这里的Parcel我们还不了解，那么接下来我们马上就来看一下这个类。

数据包装器：Parcel Binder上提供的是跨进程的服务，每个服务包含了不同的接口，每个接口的参数数量和类型都不一样。那么当客户端想要调用服务端的接口，参数是如何跨进程传递给服务端的呢？除此之外，服务端想要给客户端返回结果，结果又是如何传递回来的呢？

这些问题的答案就是：Parcel。Parcel就像一个包装器，调用者可以以任意顺序往里面放入需要的数据，所有写入的数据就像是被打成一个整体的包，然后可以直接在Binde上传输。

Parcel提供了所有基本类型的写入和读出接口，下面是其中的一部分：

```c
...
status_t            writeInt32(int32_t val);
status_t            writeUint32(uint32_t val);
......
status_t            readUtf8FromUtf16(std::string* str) const;
status_t            readUtf8FromUtf16(std::unique_ptr<std::string>* str) const;

const char*         readCString() const;
...
```

因此对于基本类型，开发者可以直接调用接口写入和读出。而对于非基本类型，需要由开发者将其拆分成基本类型然后写入到Parcel中（读出的时候也是一样）。 Parcel会将所有写入的数据进行打包，Parcel本身可以作为一个整体在进程间传递。接收方在收到Parcel之后，只要按写入同样的顺序读出即可。

这个过程，和我们现实生活中寄送包裹做法是一样的：我们将需要寄送的包裹放到硬纸盒中交给快递公司。快递公司将所有的包裹进行打包，然后集中放到运输车中送到目的地，到了目的地之后然后再进行拆分。

Parcel既包含C++部分的实现，也同时提供了Java的接口，中间通过JNI衔接。Java层的接口其实仅仅是一层包装，真正的实现都是位于C++部分中。 特别需要说明一下的是，Parcel类除了可以传递基本数据类型，还可以传递Binder对象：

```c
status_t Parcel::writeStrongBinder(const sp<IBinder>& val)
{
    return flatten_binder(ProcessState::self(), val, this);
}
```

这个方法写入的是sp

<ibinder>类型的对象，而IBinder既可能是本地Binder，也可能是远程Binder，这样我们就不可以不用关心具体细节直接进行Binder对象的传递。</ibinder>

这也是为什么IInterface中定义了两个asBinder的static方法，如果你不记得了，请回忆一下这两个方法：

```c
static sp<IBinder>  asBinder(const IInterface*);
static sp<IBinder>  asBinder(const sp<IInterface>&);
```

而对于Binder驱动，我们前面已经讲解过：Binder驱动并不是真的将对象在进程间序列化传递，而是由Binder驱动完成了对于Binder对象指针的解释和翻译，使调用者看起来就像在进程间传递对象一样。

#### 2.4、**Framework层的线程管理**

在讲解Binder驱动的时候，我们就讲解过驱动中对应线程的管理。这里我们再来看看，Framework层是如何与驱动层对接进行线程管理的。

ProcessState::setThreadPoolMaxThreadCount 方法中，会通过BINDER_SET_MAX_THREADS命令设置进程支持的最大线程数量：

```c
#define DEFAULT_MAX_BINDER_THREADS 15

status_t ProcessState::setThreadPoolMaxThreadCount(size_t maxThreads) {
    status_t result = NO_ERROR;
    if (ioctl(mDriverFD, BINDER_SET_MAX_THREADS, &maxThreads) != -1) {
        mMaxThreads = maxThreads;
    } else {
        result = -errno;
        ALOGE("Binder ioctl to set max threads failed: %s", strerror(-result));
    }
    return result;
}
```

由此驱动便知道了该Binder服务支持的最大线程数。驱动在运行过程中，会根据需要，并在没有超过上限的情况下，通过BR_SPAWN_LOOPER命令通知进程创建线程：

IPCThreadState在收到BR_SPAWN_LOOPER请求之后，便会调用ProcessState::spawnPooledThread来创建线程：

```c
status_t IPCThreadState::executeCommand(int32_t cmd)
{
    ...
    case BR_SPAWN_LOOPER:
        mProcess->spawnPooledThread(false);
        break;
    ...
}
```

ProcessState::spawnPooledThread方法负责为线程设定名称并创建线程：

```c
void ProcessState::spawnPooledThread(bool isMain)
{
    if (mThreadPoolStarted) {
        String8 name = makeBinderThreadName();
        ALOGV("Spawning new pooled thread, name=%s\n", name.string());
        sp<Thread> t = new PoolThread(isMain);
        t->run(name.string());
    }
}
```

线程在run之后，会调用threadLoop将自身添加的线程池中：

```c
virtual bool threadLoop()
{
   IPCThreadState::self()->joinThreadPool(mIsMain);
   return false;
}
```

而IPCThreadState::joinThreadPool方法中，会根据当前线程是否是主线程发送BC_ENTER_LOOPER或者BC_REGISTER_LOOPER命令告知驱动线程已经创建完毕。整个调用流程如下图所示：

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/48-Android-binder_thread_create.jpg)

### （3）、Android Binder系统-Native层添加hello服务

#### 3.1、**Client构造数据，发送数据给驱动**

首先看一下Native ServiceManager架构图

只讲数据构造过程。。

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/49-Android-addService.jpg)

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/50-Android-binder-BpBinder.png)

构造： [-> IServiceManager.cpp ::BpServiceManager]

```c
virtual status_t addService(const String16& name, const sp<IBinder>& service, bool allowIsolated) {
    Parcel data, reply; //Parcel是数据通信包
    //写入头信息"android.os.IServiceManager"
    data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());   
    data.writeString16(name);        // name为 "hello"
    data.writeStrongBinder(service); // HelloService对象，把一个binder实体“打扁”并写入parcel
    data.writeInt32(allowIsolated ? 1 : 0); // allowIsolated= false
    //remote()指向的是BpBinder对象
    status_t err = remote()->transact(ADD_SERVICE_TRANSACTION, data, &reply);
    return err == NO_ERROR ? reply.readExceptionCode() : err;
}
```

服务注册过程：向ServiceManager注册服务hello Service，服务名为"hello"； 请大家注意上面data.writeStrongBinder()一句，它专门负责把一个binder实体"打扁"并写入parcel。其代码如下：

#### 3.2.1、_* writeStrongBinder()_

[-> parcel.cpp]

```c
status_t Parcel::writeStrongBinder(const sp<IBinder>& val)
{
    return flatten_binder(ProcessState::self(), val, this);
}
```

#### 3.2.2、**flatten_binder()**

[-> parcel.cpp]

```c
status_t flatten_binder(const sp<ProcessState>& /*proc*/,
    const sp<IBinder>& binder, Parcel* out)
{
    flat_binder_object obj;

    obj.flags = 0x7f | FLAT_BINDER_FLAG_ACCEPTS_FDS;
    if (binder != NULL) {
        IBinder *local = binder->localBinder(); //本地Binder不为空
        if (!local) {
            BpBinder *proxy = binder->remoteBinder();
            const int32_t handle = proxy ? proxy->handle() : 0;
            obj.type = BINDER_TYPE_HANDLE;
            obj.binder = 0;
            obj.handle = handle;
            obj.cookie = 0;
        } else { //进入该分支
            obj.type = BINDER_TYPE_BINDER;
            obj.binder = reinterpret_cast<uintptr_t>(local->getWeakRefs());
            obj.cookie = reinterpret_cast<uintptr_t>(local);
        }
    } else {
        ...
    }
    return finish_flatten_binder(binder, obj, out);
}
```

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/51-Android-Binder-flatten_binder.png)

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/52-Android-Binder-flatten_binder.png)

将Binder对象扁平化，转换成flat_binder_object对象。 看到了吗？"打扁"的意思就是把binder对象整理成flat_binder_object变量，如果打扁的是binder实体，那么flat_binder_object用cookie域记录binder实体的指针，即BBinder指针，而如果打扁的是binder代理，那么flat_binder_object用handle域记录的binder代理的句柄值。

> 总结：Parcel的数据区域分两个部分：mData和mObjects，所有的数据不管是基础数据类型还是对象实体，全都追加到mData里，mObjects是一个偏移量数组，记录所有存放在mData中的flat_binder_object实体的偏移量。

#### 3.2.3、**finish_flatten_binder()**

将flat_binder_object写入out。

```c
inline static status_t finish_flatten_binder(
    const sp<IBinder>& , const flat_binder_object& flat, Parcel* out)
{
    return out->writeObject(flat, false);
}
```

然后flatten_binder()调用了一个关键的finish_flatten_binder()函数。这个函数内部会记录下刚刚被扁平化的flat_binder_object在parcel中的位置。说得更详细点儿就是，parcel对象内部会有一个buffer，记录着parcel中所有扁平化的数据，有些扁平数据是普通数据，而另一些扁平数据则记录着binder对象。所以parcel中会构造另一个mObjects数组，专门记录那些binder扁平数据所在的位置，示意图如下：

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/53-Android-Binder-parcel.png)

一旦到了向驱动层传递数据的时候，IPCThreadState::writeTransactionData()会先把Parcel数据整理成一个binder_transaction_data数据

```c
status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
{
    binder_transaction_data tr;

    tr.target.ptr = 0; /* Don't pass uninitialized stack data to a remote process */
    tr.target.handle = handle;
    tr.code = code;
    tr.flags = binderFlags;
    tr.cookie = 0;
    tr.sender_pid = 0;
    tr.sender_euid = 0;

    const status_t err = data.errorCheck();
    if (err == NO_ERROR) {
        tr.data_size = data.ipcDataSize();
        tr.data.ptr.buffer = data.ipcData();
        tr.offsets_size = data.ipcObjectsCount()*sizeof(binder_size_t);
        tr.data.ptr.offsets = data.ipcObjects();
    } else if (statusBuffer) {
        tr.flags |= TF_STATUS_CODE;
        *statusBuffer = err;
        tr.data_size = sizeof(status_t);
        tr.data.ptr.buffer = reinterpret_cast<uintptr_t>(statusBuffer);
        tr.offsets_size = 0;
        tr.data.ptr.offsets = 0;
    } else {
        return (mLastError = err);
    }

    mOut.writeInt32(cmd);
    mOut.write(&tr, sizeof(tr));

    return NO_ERROR;
}
```

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/54-Android-binder-writeTransactionData.png)

#### 3.2.4 、**waitForResponse()**

```c
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    int32_t cmd;
    int32_t err;

    while (1) {
        if ((err=talkWithDriver()) < NO_ERROR) break;//目的就是把上面打包的mOut数据给kernel,接着看taklWithDriver();
        cmd = mIn.readInt32();
        }
        switch (cmd) {
        case BR_REPLY:
            ......
            goto finish;

        default:
            err = executeCommand(cmd);
            break;
        }
    }
finish:
    .....
    return err;
}
```

该函数是与serviceManager通信的主要函数，首先会调用talkWithDriver()方法，将之前的打包在mOut中的数据打包成struct binder_write_read 对象，并通过ioctrl发送给kernel。

#### 3.2.5、 **IPCThreadState::talkWithDriver**

```c
status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    binder_write_read bwr;
    const bool needRead = mIn.dataPosition() >= mIn.dataSize();

    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;
    //doReceive参数，默认是为true,上面我们看到没有传参数，那么doReceive = 1；
    bwr.write_size = outAvail;
    bwr.write_buffer = (uintptr_t)mOut.data(); //将mOut数据指针存放到这里,这就是我们上面打包的数据。

    // This is what we'll read.
    if (doReceive && needRead) {
        bwr.read_size = mIn.dataCapacity(); //注意这里数据的大小，在我们new IPCThreadState对象时，已经初始化为256.
        bwr.read_buffer = (uintptr_t)mIn.data(); //mIn数据指针，放到这里
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }
    //......
    bwr.write_consumed = 0;
    bwr.read_consumed = 0;
    status_t err;
    do {
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0) //这里通过ioctl将数据写给kernel
      .....
    } while (err == -EINTR);
    return err;
}
```

该函数的作用就是将之前打包的数据通过系统调用ioctl发送给kernel，最终发送给kernel的数据是struct binder_write_read对象。该对象已经被打包了3次，它们的包含关系如下所示。 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/55-Android-binder-Transaction_data.png)

#### 3.2.6、**Client获取服务、处理回复数据过程**

内核会唤醒Client进程处理回复消息。

```c
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    int32_t cmd;
    int32_t err;
    while (1) {
        if ((err=talkWithDriver()) < NO_ERROR) break;
        ...
        cmd = mIn.readInt32();
        switch (cmd) {
          case BR_REPLY:
          {
            binder_transaction_data tr;
            err = mIn.read(&tr, sizeof(tr));
            if (reply) {
                if ((tr.flags & TF_STATUS_CODE) == 0) {
                    //当reply对象回收时，则会调用freeBuffer来回收内存
                    reply->ipcSetDataReference(
                        reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                        tr.data_size,
                        reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                        tr.offsets_size/sizeof(binder_size_t),
                        freeBuffer, this);
                }
          ...     
    }
    ...
    return err;
}
```

#### 3.2.7、**Parcel::ipcSetDataReference**

```c
void Parcel::ipcSetDataReference(const uint8_t* data, size_t dataSize,
    const binder_size_t* objects, size_t objectsCount, release_func relFunc, void* relCookie)
{
    binder_size_t minOffset = 0;
    freeDataNoInit();
    mError = NO_ERROR;
    mData = const_cast<uint8_t*>(data); //这是有4个字节的buffer。且存放的数据是0
    mDataSize = mDataCapacity = dataSize; //之前申请的大小就是4个字节。
    //ALOGI("setDataReference Setting data size of %p to %lu (pid=%d)", this, mDataSize, getpid());
    mDataPos = 0;
    ALOGV("setDataReference Setting data pos of %p to %zu", this, mDataPos);
    mObjects = const_cast<binder_size_t*>(objects); //binder对象其实地址
    mObjectsSize = mObjectsCapacity = objectsCount; //binder对象的个数。
    mNextObjectHint = 0;
    mOwner = relFunc; //释放内存的函数，后面我们就不进行了。
    mOwnerCookie = relCookie;
    for (size_t i = 0; i < mObjectsSize; i++) {
        binder_size_t offset = mObjects[i];
        if (offset < minOffset) {
            ALOGE("%s: bad object offset %"PRIu64" < %"PRIu64"\n",
                  __func__, (uint64_t)offset, (uint64_t)minOffset);
            mObjectsSize = 0;
            break;
        }
        minOffset = offset + sizeof(flat_binder_object);
    }
    scanForFds();
}
```

上面做的工作只是将事务数据分别安放到当前Parcel对象的相应位置。其中scanForFds（）是为了查找返回来的数据中是否有binder对象，这个在获取代理对象时有用。

#### 3.2.8、**readStrongBinder()**

[-> Parcel.java]

readStrongBinder的过程基本是writeStrongBinder逆过程。

```java
static jobject android_os_Parcel_readStrongBinder(JNIEnv* env, jclass clazz, jlong nativePtr) {
    Parcel* parcel = reinterpret_cast<Parcel*>(nativePtr);
    if (parcel != NULL) {
        return javaObjectForIBinder(env, parcel->readStrongBinder());
    }
    return NULL;
}
```

javaObjectForIBinder 将native层BpBinder对象转换为Java层BinderProxy对象。

#### 3.2.9、**readStrongBinder(C++)**

[-> Parcel.cpp]

```c
sp<IBinder> Parcel::readStrongBinder() const
{
    sp<IBinder> val;
    unflatten_binder(ProcessState::self(), *this, &val);
    return val;
}
```

#### 3.2.10、**unflatten_binder()**

[-> Parcel.cpp]

```c
status_t unflatten_binder(const sp<ProcessState>& proc, const Parcel& in, sp<IBinder>* out) {
    const flat_binder_object* flat = in.readObject(false);
    if (flat) {
        switch (flat->type) {
            case BINDER_TYPE_BINDER:
                *out = reinterpret_cast<IBinder*>(flat->cookie);
                return finish_unflatten_binder(NULL, *flat, in);
            case BINDER_TYPE_HANDLE:
                *out = proc->getStrongProxyForHandle(flat->handle);
                //创建BpBinder对象
                return finish_unflatten_binder(
                    static_cast<BpBinder*>(out->get()), *flat, in);
        }
    }
    return BAD_TYPE;
}
```

说明：readObject()的作用是从Parcel中读取出它所保存的flat_binder_object类型的对象。该对象的类型是BINDER_TYPE_HANDLE，因此会指向BINDER_TYPE_HANDLE对应的switch分支。 (01) 这里的proc是ProcessState对象，执行proc->getStrongProxyForHandle()会将句柄(MediaPlayerService的Binder引用描述)保存到ProcessState的链表中，然后再创建并返回该句柄的BpBinder对象(即Binder的代理)。在Android Binder机制(四) defaultServiceManager()的实现中有getStrongProxyForHandle()的详细说明，下面只给出getStrongProxyForHandle()代码。 (02) finish_unflatten_binder()中只有return NO_ERROR。

#### 3.2.11、**getStrongProxyForHandle()**

[-> ProcessState.cpp]

```c
sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{
    sp<IBinder> result;

    AutoMutex _l(mLock);
    //查找handle对应的资源项
    handle_entry* e = lookupHandleLocked(handle);

    if (e != NULL) {
        IBinder* b = e->binder;
        if (b == NULL || !e->refs->attemptIncWeak(this)) {
            ...
            //当handle值所对应的IBinder不存在或弱引用无效时，则创建BpBinder对象
            b = new BpBinder(handle);
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

经过该方法，最终创建了指向Binder服务端的BpBinder代理对象。

### （4）、Android Binder系统-Native层获取hello服务

经过前面的分析，知道流程基本类似，这里不再继续分析获取hello服务

## 五、Android Binder系统-Framwork-Java层

### （1）、Android Binder系统Java层

主要结构 Android应用程序使用Java语言开发，Binder框架自然也少不了在Java层提供接口。

前文中我们看到，Binder机制在C++层已经有了完整的实现。因此Java层完全不用重复实现，而是通过JNI衔接了C++层以复用其实现。

下图描述了Binder Framework Java层到C++层的衔接关系。 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/56-Android-Binder_JNI.png)

这里对图中Java层和JNI层的几个类做一下说明( 关于C++层的讲解请看这里 )：

这里的IInterface，IBinder和C++层的两个类是同名的。这个同名并不是巧合：它们不仅仅同名，它们所起的作用，以及其中包含的接口都是几乎一样的，区别仅仅在于一个是C++层，一个是Java层而已。

除了IInterface，IBinder之外，这里Binder与BinderProxy类也是与C++的类对应的，下面列出了Java层和C++层类的对应关系：

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/57-Android-binder-Java-class.png)

除了IInterface，IBinder之外，这里Binder与BinderProxy类也是与C++的类对应的，下面列出了Java层和C++层类的对应关系：

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/58-Android-binder-Java-c-class.png)

### （2）、JNI的衔接

JNI全称是Java Native Interface，这个是由Java虚拟机提供的机制。这个机制使得native代码可以和Java代码互相通讯。简单来说就是：我们可以在C/c端调用Java代码，也可以在Java端调用C/c代码。

关于JNI的详细说明，可以参见Oracle的官方文档：Java Native Interface ，这里不多说明。

实际上，在Android中很多的服务或者机制都是在C/c层实现的，想要将这些实现复用到Java层，就必须通过JNI进行衔接。AOSP源码中，/frameworks/base/core/jni/ 目录下的源码就是专门用来对接Framework层的JNI实现的。

看一下Binder.java的实现就会发现，这里面有不少的方法都是用native关键字修饰的，并且没有方法实现体，这些方法其实都是在C++中android_util_Binder.cpp实现的： 那么，那么，c是如何调用Java的呢？最关键的，libbinder中的BBinder::onTransact是如何能够调用到Java中的Binder::onTransact的呢？ 
![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/59-Android-binder-JavaBBinder.png)

这段逻辑就是android_util_Binder.cpp中JavaBBinder::onTransact中处理的了。JavaBBinder是BBinder子类，其类结构如下：libbinder中的BBinder::onTransact是如何能够调用到Java中的Binder::onTransact的呢？ JavaBBinder::onTransact关键代码如下：

```c
virtual status_t onTransact(
   uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags = 0)
{
   JNIEnv* env = javavm_to_jnienv(mVM);

   IPCThreadState* thread_state = IPCThreadState::self();
   const int32_t strict_policy_before = thread_state->getStrictModePolicy();

   jboolean res = env->CallBooleanMethod(mObject, gBinderOffsets.mExecTransact,
       code, reinterpret_cast<jlong>(&data), reinterpret_cast<jlong>(reply), flags);
   ...
}
```

请注意这段代码中的这一行：

```c
jboolean res = env->CallBooleanMethod(mObject, gBinderOffsets.mExecTransact,
  code, reinterpret_cast<jlong>(&data), reinterpret_cast<jlong>(reply), flags);
```

这一行代码其实是在调用mObject上offset为mExecTransact的方法。这里的几个参数说明如下：

mObject 指向了Java端的Binder对象 gBinderOffsets.mExecTransact 指向了Binder类的execTransact方法 data 调用execTransact方法的参数 code, data, reply, flags都是传递给调用方法execTransact的参数 而JNIEnv.CallBooleanMethod这个方法是由虚拟机实现的。即：虚拟机会提供native方法来调用一个Java Object上的方法（关于Android上的Java虚拟机，今后我们会专门讲解）。

这样，就在C++层的JavaBBinder::onTransact中调用了Java层Binder::execTransact方法。而在Binder::execTransact方法中，又调用了自身的onTransact方法，由此保证整个过程串联了起来：

```c
private boolean execTransact(int code, long dataObj, long replyObj,
       int flags) {
   Parcel data = Parcel.obtain(dataObj);
   Parcel reply = Parcel.obtain(replyObj);
   boolean res;
   try {
       res = onTransact(code, data, reply, flags);
   } catch (RemoteException|RuntimeException e) {
       if (LOG_RUNTIME_EXCEPTION) {
           Log.w(TAG, "Caught a RuntimeException from the binder stub implementation.", e);
       }
       if ((flags & FLAG_ONEWAY) != 0) {
           if (e instanceof RemoteException) {
               Log.w(TAG, "Binder call failed.", e);
           } else {
               Log.w(TAG, "Caught a RuntimeException from the binder stub implementation.", e);
           }
       } else {
           reply.setDataPosition(0);
           reply.writeException(e);
       }
       res = true;
   } catch (OutOfMemoryError e) {
       RuntimeException re = new RuntimeException("Out of memory", e);
       reply.setDataPosition(0);
       reply.writeException(re);
       res = true;
   }
   checkParcel(this, code, reply, "Unreasonably large binder reply buffer");
   reply.recycle();
   data.recycle();

   StrictMode.clearGatheredViolations();

   return res;
}
```

### （3）、Java层的ServiceManager

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/60-Android-binder-class_ServiceManager_java.jpg)

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/61-Android-binder-ServiceManager_Java.png)

通过这个类图我们看到，Java层的ServiceManager和C++层的接口是一样的。

通过这个类图我们看到，Java层的ServiceManager和C++层的接口是一样的。

然后我们再选取addService方法看一下实现：

```java
public static void addService(String name, IBinder service, boolean allowIsolated) {
   try {
       getIServiceManager().addService(name, service, allowIsolated);
   } catch (RemoteException e) {
       Log.e(TAG, "error in addService", e);
   }
}

   private static IServiceManager getIServiceManager() {
   if (sServiceManager != null) {
       return sServiceManager;
   }

   // Find the service manager
   sServiceManager = ServiceManagerNative.asInterface(BinderInternal.getContextObject());
   return sServiceManager;
}
```

很显然，这段代码中，最关键就是下面这个调用：

```java
ServiceManagerNative.asInterface(BinderInternal.getContextObject());
```

然后我们需要再看一下BinderInternal.getContextObject()和ServiceManagerNative.asInterface两个方法。

BinderInternal.getContextObject()是一个JNI方法，其实现代码在android_util_Binder.cpp中：

```java
static jobject android_os_BinderInternal_getContextObject(JNIEnv* env, jobject clazz)
{
    sp<IBinder> b = ProcessState::self()->getContextObject(NULL);
    return javaObjectForIBinder(env, b);
}
```

而ServiceManagerNative.asInterface的实现和其他的Binder服务是一样的套路：

```java
static public IServiceManager asInterface(IBinder obj)
{
   if (obj == null) {
       return null;
   }
   IServiceManager in =
       (IServiceManager)obj.queryLocalInterface(descriptor);
   if (in != null) {
       return in;
   }

   return new ServiceManagerProxy(obj);
}
```

先通过queryLocalInterface查看能不能获得本地Binder，如果无法获取，则创建并返回ServiceManagerProxy对象。

而ServiceManagerProxy自然也是和其他Binder Proxy一样的实现套路：

```java
public void addService(String name, IBinder service, boolean allowIsolated)
       throws RemoteException {
   Parcel data = Parcel.obtain();
   Parcel reply = Parcel.obtain();
   data.writeInterfaceToken(IServiceManager.descriptor);
   data.writeString(name);
   data.writeStrongBinder(service);
   data.writeInt(allowIsolated ? 1 : 0);
   mRemote.transact(ADD_SERVICE_TRANSACTION, data, reply, 0);
   reply.recycle();
   data.recycle();
}
```

接下来的调用流程前面已经分析过了，在此就不再分析了。 

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/62-Android-binder-binder_ipc_process.jpg)

## 六、Android Binder系统-AIDL

作为Binder机制的最后一个部分内容，我们来讲解一下开发者经常使用的AIDL机制是怎么回事。

AIDL全称是Android Interface Definition Language，它是Android SDK提供的一种机制。借助这个机制，应用可以提供跨进程的服务供其他应用使用。AIDL的详细说明可以参见官方开发文档：<https://developer.android.com/guide/components/aidl.html> 。

这里，我们就以官方文档上的例子看来一下AIDL与Binder框架的关系。

开发一个基于AIDL的Service需要三个步骤：

定义一个.aidl文件 实现接口 暴露接口给客户端使用 aidl文件使用Java语言的语法来定义，每个.aidl文件只能包含一个interface，并且要包含interface的所有方法声明。

默认情况下，AIDL支持的数据类型包括：

基本数据类型（即int，long，char，boolean等） String CharSequence List（List的元素类型必须是AIDL支持的） Map（Map中的元素必须是AIDL支持的） 对于AIDL中的接口，可以包含0个或多个参数，可以返回void或一个值。所有非基本类型的参数必须包含一个描述是数据流向的标签，可能的取值是：in，out或者inout。

下面是一个aidl文件的示例：

```java
// IRemoteService.aidl
package com.example.android;

// Declare any non-default types here with import statements

/** Example service interface */
interface IRemoteService {
    /** Request the process ID of this service, to do evil things with it. */
    int getPid();

    /** Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
            double aDouble, String aString);
}
```

这个文件中包含了两个接口 ：

getPid 一个无参的接口，返回值类型为int basicTypes，包含了几个基本类型作为参数的接口，无返回值 对于包含.aidl文件的工程，Android IDE（以前是Eclipse，现在是Android Studio）在编译项目的时候，会为aidl文件生成对应的Java文件。

针对上面这个aidl文件生成的java文件中包含的结构如下图所示：

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/63-Android-binder-aidl_java.png)

在这个生成的Java文件中，包括了：

一个名称为IRemoteService的interface，该interface继承自android.os.IInterface并且包含了我们在aidl文件中声明的接口方法 IRemoteService中包含了一个名称为Stub的静态内部类，这个类是一个抽象类，它继承自android.os.Binder并且实现了IRemoteService接口。这个类中包含了一个onTransact方法 Stub内部又包含了一个名称为Proxy的静态内部类，Proxy类同样实现了IRemoteService接口 仔细看一下Stub类和Proxy两个中包含的方法，是不是觉得很熟悉？是的，这里和前面介绍的服务实现是一样的模式。这里我们列一下各层类的对应关系：

c层  |   Java层   |      AIDL
:---: | :-------: | :-------------:
BpXXX | XXXProxy  | IXXX.Stub.Proxy
BnXXX | XXXNative |    IXXX.Stub

为了整个结构的完整性，最后我们还是来看一下生成的Stub和Proxy类中的实现逻辑。

Stub是提供给开发者实现业务的父类，而Proxy的实现了对外提供的接口。Stub和Proxy两个类都有一个asBinder的方法。

Stub类中的asBinder实现就是返回自身对象：

```java
Override
public android.os.IBinder asBinder() {
    return this;
}
```

而Proxy中asBinder的实现是返回构造函数中获取的mRemote对象，相关代码如下：

```java
private android.os.IBinder mRemote;

Proxy(android.os.IBinder remote) {
    mRemote = remote;
}

Override
public android.os.IBinder asBinder() {
    return mRemote;
}
```

而这里的mRemote对象其实就是远程服务在当前进程的标识。

上文我们说了，Stub类是用来提供给开发者实现业务逻辑的父类，开发者者继承自Stub然后完成自己的业务逻辑实现，例如这样：

```java
private final IRemoteService.Stub mBinder = new IRemoteService.Stub() {
   public int getPid(){
       return Process.myPid();
   }
   public void basicTypes(int anInt, long aLong, boolean aBoolean,
       float aFloat, double aDouble, String aString) {
       // Does something
   }
};
```

而这个Proxy类，就是用来给调用者使用的对外接口。我们可以看一下Proxy中的接口到底是如何实现的：

Proxy中getPid方法实现如下所示：

```java
Override
public int getPid() throws android.os.RemoteException {
    android.os.Parcel _data = android.os.Parcel.obtain();
    android.os.Parcel _reply = android.os.Parcel.obtain();
    int _result;
    try {
        _data.writeInterfaceToken(DESCRIPTOR);
        mRemote.transact(Stub.TRANSACTION_getPid, _data, _reply, 0);
        _reply.readException();
        _result = _reply.readInt();
    } finally {
        _reply.recycle();
        _data.recycle();
    }
    return _result;
}
```

这里就是通过Parcel对象以及transact调用对应远程服务的接口。而在Stub类中，生成的onTransact方法对应的处理了这里的请求：

```java
Override
public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags)
        throws android.os.RemoteException {
    switch (code) {
    case INTERFACE_TRANSACTION: {
        reply.writeString(DESCRIPTOR);
        return true;
    }
    case TRANSACTION_getPid: {
        data.enforceInterface(DESCRIPTOR);
        int _result = this.getPid();
        reply.writeNoException();
        reply.writeInt(_result);
        return true;
    }
    case TRANSACTION_basicTypes: {
        data.enforceInterface(DESCRIPTOR);
        int _arg0;
        _arg0 = data.readInt();
        long _arg1;
        _arg1 = data.readLong();
        boolean _arg2;
        _arg2 = (0 != data.readInt());
        float _arg3;
        _arg3 = data.readFloat();
        double _arg4;
        _arg4 = data.readDouble();
        java.lang.String _arg5;
        _arg5 = data.readString();
        this.basicTypes(_arg0, _arg1, _arg2, _arg3, _arg4, _arg5);
        reply.writeNoException();
        return true;
    }
    }
    return super.onTransact(code, data, reply, flags);
}
```

onTransact()所要做的就是：

根据code区分请求的是哪个接口 通过data来获取请求的参数 调用由子类实现的抽象方法 有了前文的讲解，对于这部分内容应当不难理解了。

到这里，我们终于讲解完Binder了。

完整框架： 

![Markdown](https://raw.githubusercontent.com/zhoujinjianmsn/PicGo/master/android.binder/64-Android-Binder-IPCall.png)


## 七、参考文档(特别感谢各位前辈的分析和图示)：

[Binder源码分析](https://github.com/xdtianyu/SourceAnalysis/blob/master/Binder%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md)
[深入分析Android Binder](http://blog.csdn.net/yangwen123/article/category/1609389)
[Binder系列 - Gityuan博客 | 袁辉辉博客](http://gityuan.com/tags/#binder)
[Android Binder机制(1) ~ (12) - Wangkuiwu.github.io](https://wangkuiwu.github.io/page2/)
[Binder机制-关于Binder的文章 - 泡在网上的日子](http://www.jcodecraeer.com/tags.php?/Binder/)
[理解Android Binder机制 - Qiangbo.space博客](http://qiangbo.space/tags/#Android)
[红茶一杯话Binder - 悠然红茶](https://my.oschina.net/youranhongcha/blog?catalog=373547&temp=1505099522160)    
[Binder框架解析](https://github.com/hehonghui/android-tech-frontier/blob/master/issue-22/Binder%E6%A1%86%E6%9E%B6%E8%A7%A3%E6%9E%90.md) 
[Android Binder详解](https://mr-cao.gitbooks.io/android/content/android-binder.html)
[图文详解 Android Binder跨进程通信机制 原理](http://blog.csdn.net/carson_ho/article/details/73560642)
[理解Android Binder机制(1/3)：驱动篇-qiangbo.space](http://qiangbo.space/2017-01-15/AndroidAnatomy_Binder_Driver/)
[理解Android Binder机制(2/3)：c层-qiangbo.space](http://qiangbo.space/2017-02-12/AndroidAnatomy_Binder_CPP/) 
[理解Android Binder机制(3/3)：Java层-qiangbo.space](http://qiangbo.space/2017-03-15/AndroidAnatomy_Binder_Java/)
[Android Binder 分析--系列-light3moon](http://light3moon.com/1986/12/20/%E6%96%87%E7%AB%A0%E7%B4%A2%E5%BC%95/#)
[Android学习笔记-Binder | Palance's Blog](http://palanceli.com/categories/Android%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/page/2/)
[android系统 -Binder - armwind的专栏 - CSDN博客](http://blog.csdn.net/armwind/article/category/6282972) 
[Bettarwang的专栏 -Android Binder机制](http://blog.csdn.net/Bettarwang/article/category/2276043)
[深入剖析Android系统 - binder - CSDN博客](http://blog.csdn.net/yangwen123/article/category/1609389)
